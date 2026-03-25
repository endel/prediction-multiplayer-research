# Godot Engine: Multiplayer Synchronization and Prediction

## Table of Contents

1. [MultiplayerSynchronizer Overview (Godot 4.0+)](#multiplayersynchronizer-overview-godot-40)
2. [What MultiplayerSynchronizer Provides](#what-multiplayersynchronizer-provides)
3. [What MultiplayerSynchronizer Lacks](#what-multiplayersynchronizer-lacks)
4. [Open Proposal #7280: Prediction and Interpolation](#open-proposal-7280-prediction-and-interpolation)
5. [Community Addons](#community-addons)
   - [Netfox](#netfox)
   - [Godot Rollback Netcode (Snopek Games)](#godot-rollback-netcode-snopek-games)
   - [MonkeNet](#monkenet)
6. [Code Examples](#code-examples)
7. [Trade-offs Comparison](#trade-offs-comparison)
8. [References](#references)

---

## MultiplayerSynchronizer Overview (Godot 4.0+)

Godot 4.0 introduced a completely rewritten multiplayer system centered around two "configuration" nodes that replaced the older RPC-heavy approach to state replication:

- **MultiplayerSpawner** -- configures where and which scenes can be remotely instantiated by which peer.
- **MultiplayerSynchronizer** -- configures which node properties are synchronized and by which peer.

`MultiplayerSynchronizer` inherits from `Node` and synchronizes configured properties from the multiplayer authority to remote peers. It uses Godot's `SceneReplicationConfig` resource to define which properties to replicate, when to replicate them, and to whom.

The design philosophy is declarative: rather than writing explicit RPC calls for every property, developers add a `MultiplayerSynchronizer` node, configure property paths in the inspector, and the engine handles serialization and transport automatically.

### Scene Tree Architecture

A typical multiplayer scene hierarchy looks like:

```
World (Node)
├── MultiplayerSpawner          # Handles spawning/despawning across peers
├── Level (Node)
│   ├── Environment
│   └── ...
└── Players (Node)
    └── Player (CharacterBody3D)
        ├── MultiplayerSynchronizer   # Syncs position, velocity, etc.
        ├── PlayerInput (Node)
        │   └── MultiplayerSynchronizer  # Syncs input from owning peer
        ├── CollisionShape3D
        └── MeshInstance3D
```

The recommended pattern (from Godot's official article) separates the **player input** synchronizer from the **character state** synchronizer. The input node is owned by the respective player peer, while the character node's authority remains with the server.

---

## What MultiplayerSynchronizer Provides

### Property Replication

Properties are configured through `SceneReplicationConfig`, which supports three synchronization modes:

| Mode | Behavior |
|------|----------|
| **Always** | Property is sent every replication interval, regardless of changes |
| **On Change** | Property is only sent when its value differs from the previously sent value |
| **Spawn** | Property is only sent once during the initial spawn/replication setup |

Properties are referenced by `NodePath` and can include any exported or non-exported property on any node in the subtree. The synchronizer serializes them using Godot's built-in `Variant` encoding.

### Authority Model

`MultiplayerSynchronizer` integrates with Godot's authority system:

- **`set_multiplayer_authority(peer_id)`** -- designates which peer is authoritative over a node subtree.
- Only the authority peer sends synchronization data; all other peers receive.
- The authority defaults to the server (peer ID 1).
- Authority can be transferred at runtime using `set_multiplayer_authority()`.
- A `visibility_for` system (per-peer filtering) controls which peers receive synchronization updates.

```gdscript
# Server-authoritative by default (peer_id = 1)
player_node.set_multiplayer_authority(1)

# Transfer input authority to the owning player
player_input_node.set_multiplayer_authority(player_peer_id)
```

### Replication Intervals

- **`replication_interval`** -- the minimum time (in seconds) between replication updates (default: `0.0`, meaning every physics frame).
- **`delta_interval`** -- the minimum time between delta-encoded updates, which send only changed properties.
- The `synchronized` signal is emitted on the receiving peer when new data arrives.
- The `delta_synchronized` signal is emitted when a delta update is received.
- Visibility filters can further restrict which peers receive data via `set_visibility_for(peer_id, visible)` and the `visibility_changed` signal.

### Built-in Features Summary

- Automatic property serialization and deserialization
- Configurable per-property sync modes (Always, On Change, Spawn)
- Server-authoritative ownership model with transferable authority
- Peer-level visibility filtering
- Public visibility toggle (`public_visibility`)
- Integration with `MultiplayerSpawner` for automatic spawn-time property transfer
- Delta compression for bandwidth savings

---

## What MultiplayerSynchronizer Lacks

Despite being a solid replication primitive, `MultiplayerSynchronizer` has significant gaps for fast-paced or competitive multiplayer games:

### No Built-in Client-Side Prediction

When a client sends input to the server and waits for the authoritative state to return, the round-trip time creates visible movement delay. `MultiplayerSynchronizer` provides no mechanism for the client to speculatively apply its own input and then reconcile with the server's response. The client either:

1. Waits for the server (high latency feel), or
2. Runs with local authority (no server validation, vulnerable to cheating).

There is no middle ground provided out of the box.

### No Built-in Interpolation

`MultiplayerSynchronizer` writes received values directly to the target properties. There is no buffering, no lerp between received states, and no way to smooth the transition between updates. At lower sync rates or under packet loss, this produces visibly choppy movement for remote entities.

The workaround described in proposal #7280 is to sync to a separate "shadow" variable and manually interpolate in `_process()`:

```gdscript
# Workaround: sync to a shadow variable, then interpolate manually
var synced_position: Vector3  # Updated by MultiplayerSynchronizer
var display_position: Vector3  # Used for rendering

func _process(delta):
    display_position = display_position.lerp(synced_position, 10.0 * delta)
    global_position = display_position
```

But this is fragile, adds boilerplate, and does not integrate with the synchronizer's property system.

### No Rollback / Server Reconciliation

There is no concept of a tick-based state buffer, no ability to rewind the game state to a previous tick and re-simulate forward with corrected data. When the server sends an authoritative correction that differs from the client's predicted state, the client simply snaps to the new value. There is no replay of intervening inputs.

### No Tick Synchronization

The engine does not provide a shared, drift-corrected tick clock across peers. Each peer runs its own `_physics_process` loop independently. Correlating "which server tick does this state correspond to" requires manual implementation.

### No Input Delay Management

There is no built-in mechanism for input delay (adding a few frames of buffer to ensure inputs arrive before the tick they are needed for), which is essential for peer-to-peer rollback architectures.

---

## Open Proposal #7280: Prediction and Interpolation

**GitHub Issue:** [godotengine/godot-proposals#7280](https://github.com/godotengine/godot-proposals/issues/7280)

Opened on **July 14, 2023** by user `vectorassembly`, this proposal is titled *"Add property prediction and interpolation, and a way to capture MultiplayerSynchronization events."*

### The Problem Statement

> "There's no easy way to interpolate multiplayer properties. As a workaround, I make a variable that contains current position and share it via MultiplayerSynchronizer node. The reason why I'm using a custom variable for position is that MultiplayerSynchronizer automatically replaces the position value with the one it just got from the server."

The core issue is that `MultiplayerSynchronizer` **directly overwrites** target properties with received values, leaving no hook for the developer to intercept, buffer, interpolate, or predict those values.

### Proposed Solution

The proposal suggests two changes:

1. **`SceneReplicationConfig` should have a property to disable automatic property updates** -- allowing the developer to receive the data without it being immediately applied.

2. **The `synchronized` signal should pass a dictionary of synchronized values** -- enabling custom logic like interpolation, prediction, or validation before the values are applied.

Pseudocode from the proposal:

```gdscript
# Proposed API (not yet implemented)
func _on_synchronized(values: Dictionary):
    # Interpolate position instead of snapping
    target_position = values["position"]

    # Or validate before applying
    if is_reasonable(values["velocity"]):
        velocity = values["velocity"]
```

### Current Status

As of this writing, the proposal remains **open** with community discussion. No implementation PR has been merged. The community consensus points to the addon ecosystem (Netfox, Godot Rollback Netcode) as the current practical solution, while the core engine team has not committed to a timeline for built-in prediction/interpolation support.

---

## Community Addons

### Netfox

**Repository:** [github.com/foxssake/netfox](https://github.com/foxssake/netfox)
**License:** MIT
**Language:** GDScript (with experimental C# support via [Netfox Sharp](https://github.com/CyFurStudios/NetfoxSharp))
**Godot Version:** 4.x
**Asset Library:** Available as `netfox`, `netfox.noray`, `netfox.extras`

#### Architecture

Netfox is a modular addon suite consisting of four packages:

| Package | Purpose |
|---------|---------|
| **netfox** | Core: timing, rollback, CSP, state interpolation |
| **netfox.noray** | NAT traversal via the [noray](https://github.com/foxssake/noray) relay server |
| **netfox.extras** | High-level components: `BaseNetInput`, `NetworkWeapon`, `RewindableStateMachine` |
| **netfox.internals** | Shared utilities (auto-included as dependency) |

The core architecture is built around three key singletons and two key nodes:

**Singletons:**

- **`NetworkTime`** -- Provides a synchronized tick clock across all peers. Runs game logic at a fixed, configurable tickrate. Emits `before_tick_loop`, `on_tick`, and `after_tick_loop` signals. Uses time synchronization to keep the host and clients in lockstep.

- **`NetworkRollback`** -- Manages the rollback loop. Calls `_rollback_tick(delta, tick, is_fresh)` on all rollback-aware nodes during re-simulation. The `is_fresh` parameter indicates whether a tick is being simulated for the first time (useful for triggering one-shot effects like sounds or particles without repeating them during rollback).

- **`NetworkTimeSynchronizer`** -- Synchronizes clocks between host and clients, measuring and compensating for drift and latency.

**Nodes:**

- **`RollbackSynchronizer`** -- The workhorse node. Manages state and input properties, records per-tick state history, sends authoritative state from the server, detects mismatches, and triggers rollback + re-simulation. Supports diff states for bandwidth optimization.

- **`TickInterpolator`** -- Provides visual interpolation between ticks. If the tickrate (e.g., 24 Hz) is lower than the rendering framerate (e.g., 60 FPS), this node interpolates state properties to produce smooth visuals.

#### Client-Side Prediction Flow

1. **Input Gathering:** Before each tick loop, the client's `PlayerInput` node reads local input and stores it in exported properties.
2. **Tick Simulation:** The `_rollback_tick()` method on the player's `CharacterBody3D` applies input to produce new state (position, velocity).
3. **State Sent to Server:** `RollbackSynchronizer` sends input properties to the server.
4. **Server Simulates:** The server runs the same `_rollback_tick()` with the received input and produces authoritative state.
5. **State Returned to Client:** The server's authoritative state properties are sent back to the client.
6. **Mismatch Detection:** If the client's predicted state differs from the server's authoritative state, a rollback is triggered.
7. **Rollback + Replay:** The client rewinds to the last known-good server state and re-simulates all subsequent ticks with the stored inputs, producing a corrected present state.

#### GDScript Code Example -- Netfox CSP

**PlayerInput node (input gathering):**

```gdscript
extends Node
class_name PlayerInput

var movement: Vector3 = Vector3.ZERO

func _ready():
    NetworkTime.before_tick_loop.connect(_gather)

func _gather():
    if not is_multiplayer_authority():
        return

    var mx = Input.get_axis("move_left", "move_right")
    var mz = Input.get_axis("move_forward", "move_back")
    movement = Vector3(mx, 0, mz)
```

**Player controller (state simulation):**

```gdscript
extends CharacterBody3D

@export var speed := 5.0
@export var input: PlayerInput

var gravity = ProjectSettings.get_setting(&"physics/3d/default_gravity")

func _ready():
    position = Vector3(0, 4, 0)
    if input == null:
        input = $Input

func _rollback_tick(delta, _tick, _is_fresh):
    # Apply gravity
    if not is_on_floor():
        velocity.y -= gravity * delta

    # Apply input-driven movement
    var input_dir = input.movement
    var direction = (transform.basis * Vector3(input_dir.x, 0, input_dir.z)).normalized()
    velocity.x = direction.x * speed
    velocity.z = direction.z * speed

    # Compensate for physics delta vs network tick delta
    velocity *= NetworkTime.physics_factor
    move_and_slide()
    velocity /= NetworkTime.physics_factor
```

**Scene tree setup:**

```
Player (CharacterBody3D)          -- authority: server (peer 1)
├── Input (PlayerInput)           -- authority: owning player
├── RollbackSynchronizer
│   ├── State properties:  :position, :velocity
│   └── Input properties:  Input:movement
├── TickInterpolator
│   └── Properties: :position
├── CollisionShape3D
└── MeshInstance3D
```

**Ownership setup at spawn time:**

```gdscript
@onready var rollback_synchronizer = $RollbackSynchronizer
var peer_id = 0

func _ready():
    await get_tree().process_frame
    set_multiplayer_authority(1)              # Server owns the character
    $Input.set_multiplayer_authority(peer_id) # Player owns their input
    rollback_synchronizer.process_settings()  # Re-sort properties by ownership
```

#### C# Code Example -- Netfox Sharp

```csharp
using Godot;
using NetfoxSharp;

public partial class PlayerInput : BaseNetInput
{
    [Networked] public Vector3 Movement { get; set; } = Vector3.Zero;

    public override void _Gather()
    {
        float mx = Input.GetAxis("move_left", "move_right");
        float mz = Input.GetAxis("move_forward", "move_back");
        Movement = new Vector3(mx, 0, mz);
    }
}

public partial class Player : CharacterBody3D
{
    [Export] public float Speed = 5.0f;
    [Export] public PlayerInput Input;

    private float _gravity = (float)ProjectSettings.GetSetting("physics/3d/default_gravity");

    public override void _RollbackTick(double delta, int tick, bool isFresh)
    {
        if (!IsOnFloor())
            Velocity = new Vector3(Velocity.X, Velocity.Y - _gravity * (float)delta, Velocity.Z);

        var dir = (Transform.Basis * new Vector3(Input.Movement.X, 0, Input.Movement.Z)).Normalized();
        Velocity = new Vector3(dir.X * Speed, Velocity.Y, dir.Z * Speed);

        Velocity *= NetworkTime.PhysicsFactor;
        MoveAndSlide();
        Velocity /= NetworkTime.PhysicsFactor;
    }
}
```

#### Additional Netfox Features

- **Diff states:** Only sends changed properties to reduce bandwidth. Uses a full-state/ack protocol to ensure consistency.
- **Input broadcast toggle:** Can restrict input sending to server-only (recommended) to reduce bandwidth and attack surface.
- **`PredictiveSynchronizer`:** An alternative to `RollbackSynchronizer` for simpler use cases.
- **`StateSynchronizer`:** For non-rollback state sync (e.g., NPC state).
- **`RewindableAction`:** For lag-compensated hit detection.
- **`NetworkWeapon`:** High-level weapon component with built-in lag compensation.
- **`RewindableStateMachine`:** State machine that participates in rollback.
- **Rollback Debugger:** Editor tool for visualizing rollback behavior.
- **Visibility filtering:** Control which peers see which entities.

---

### Godot Rollback Netcode (Snopek Games)

**Repository:** [gitlab.com/snopek-games/godot-rollback-netcode](https://gitlab.com/snopek-games/godot-rollback-netcode)
**License:** MIT
**Language:** GDScript (C# via [GodotRollbackNetcodeMono](https://github.com/Fractural/GodotRollbackNetcodeMono))
**Architecture:** GGPO-style peer-to-peer rollback
**Asset Library:** Available as "Godot Rollback Netcode"

#### GGPO-Style Implementation

Unlike Netfox's client-server model, Godot Rollback Netcode implements a **peer-to-peer lockstep with rollback** architecture inspired by GGPO (Good Game Peace Out), the industry-standard rollback library used in fighting games.

Key architectural differences from Netfox:

| Aspect | Netfox | Godot Rollback Netcode |
|--------|--------|----------------------|
| **Topology** | Client-server | Peer-to-peer |
| **Authority** | Server is authoritative | All peers simulate equally |
| **Determinism** | Not strictly required | **Required** -- all peers must produce identical simulation results |
| **Rollback trigger** | Server correction mismatch | Input arrival from remote peers |
| **Primary use case** | Action games, FPS, MMO | Fighting games, sports, real-time strategy |

#### Core Singleton: SyncManager

The entire addon is driven by the `SyncManager` singleton, which:

1. Manages a tick-based simulation loop
2. Gathers local input and sends it to peers
3. Predicts remote input when it has not yet arrived
4. Detects when predicted input was wrong (real input arrives)
5. Rolls back to the tick where the misprediction occurred
6. Re-simulates all ticks from that point to the present
7. Detects state mismatches via hash comparison

#### Virtual Methods (Group-based)

Nodes participate in rollback by being added to the `"network_sync"` group. `SyncManager` then calls these virtual methods:

```gdscript
# Required: save and restore state for rollback
func _save_state() -> Dictionary:
    return {
        "position": position,
        "velocity": velocity,
        "health": health,
    }

func _load_state(state: Dictionary) -> void:
    position = state["position"]
    velocity = state["velocity"]
    health = state["health"]

# Required: game logic runs here instead of _physics_process
func _network_process(input: Dictionary) -> void:
    var direction = input.get("direction", Vector2.ZERO)
    velocity = direction * speed
    move_and_slide()

# Required for player-controlled nodes: gather local input
func _get_local_input() -> Dictionary:
    return {
        "direction": Input.get_vector("left", "right", "up", "down"),
        "shoot": Input.is_action_just_pressed("shoot"),
    }

# Optional: predict remote player input when it hasn't arrived yet
func _predict_remote_input(previous_input: Dictionary, ticks_since_real_input: int) -> Dictionary:
    # Default behavior: repeat last known input
    # Custom: could decay movement, stop shooting, etc.
    return {
        "direction": previous_input.get("direction", Vector2.ZERO),
        "shoot": false,  # Don't predict shooting
    }

# Optional: interpolate between states for smooth rendering
func _interpolate_state(old_state: Dictionary, new_state: Dictionary, weight: float) -> void:
    position = old_state["position"].lerp(new_state["position"], weight)
```

#### Rollback-Aware Node Types

The addon provides specialized nodes that integrate with the rollback system:

- **`NetworkTimer`** -- Timer that counts in ticks (not seconds) and properly rolls back.
- **`NetworkAnimationPlayer`** -- AnimationPlayer that advances per tick and supports rollback.
- **`NetworkRandomNumberGenerator`** -- Deterministic RNG that rolls back correctly. Uses a "mother seed" pattern for distributing seeds.

#### Setup and Usage

**1. Install and enable the plugin:**

```
addons/godot-rollback-netcode/
```

**2. Connect peers and start synchronization:**

```gdscript
# After establishing connections via Godot's High-Level Multiplayer API:
func _on_peer_connected(peer_id: int):
    SyncManager.add_peer(peer_id)

func _on_all_players_ready():
    # Only the host (peer 1) calls start
    if multiplayer.is_server():
        SyncManager.start()

func _on_sync_stopped():
    SyncManager.clear_peers()
```

**3. Implement a network-synced player:**

```gdscript
extends CharacterBody2D

const SPEED = 300.0

func _ready():
    add_to_group("network_sync")

func _save_state() -> Dictionary:
    return {
        "pos_x": position.x,
        "pos_y": position.y,
        "vel_x": velocity.x,
        "vel_y": velocity.y,
    }

func _load_state(state: Dictionary) -> void:
    position.x = state["pos_x"]
    position.y = state["pos_y"]
    velocity.x = state["vel_x"]
    velocity.y = state["vel_y"]

func _get_local_input() -> Dictionary:
    var input_vector = Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    # Only include input if there is any, to save bandwidth
    var input := {}
    if input_vector != Vector2.ZERO:
        input["direction"] = input_vector
    return input

func _predict_remote_input(prev: Dictionary, _ticks_since: int) -> Dictionary:
    var predicted := {}
    if prev.has("direction"):
        predicted["direction"] = prev["direction"]
    return predicted

func _network_process(input: Dictionary) -> void:
    var direction = input.get("direction", Vector2.ZERO)
    velocity = direction * SPEED
    move_and_slide()

func _interpolate_state(old_state: Dictionary, new_state: Dictionary, weight: float) -> void:
    position.x = lerp(old_state["pos_x"], new_state["pos_x"], weight)
    position.y = lerp(old_state["pos_y"], new_state["pos_y"], weight)
```

**4. Spawn network-synced objects:**

```gdscript
# Use SyncManager.spawn() instead of instantiate() for rollback-safe spawning
func fire_bullet(pos: Vector2, dir: Vector2):
    SyncManager.spawn(
        "Bullet",           # Name
        $Bullets,           # Parent node
        bullet_scene,       # PackedScene
        {                   # Data dictionary
            "position": pos,
            "direction": dir,
        }
    )
```

**5. Play rollback-safe sounds:**

```gdscript
# Sounds played during rollback ticks won't repeat
func _network_process(input: Dictionary) -> void:
    if input.get("shoot", false):
        SyncManager.play_sound(str(get_path()) + ":shoot", shoot_sound, {
            "position": position,
        })
```

#### Logging and Replay

The addon includes a **Log Inspector** editor tool:

```gdscript
# Start logging at match start
SyncManager.start_logging("user://detailed_logs/match_%d.log" % Time.get_unix_time_from_system())

# Stop logging at match end
SyncManager.stop_logging()
```

Logs can be replayed in the editor to debug desynchronization issues, showing frame-by-frame state, input, and rollback data.

#### Project Settings

Configured under **Network > Rollback** in Project Settings:

| Setting | Description |
|---------|-------------|
| **Max Buffer Size** | Maximum ticks the game can roll back |
| **Input Delay** | Frames of input delay (reduces rollback frequency) |
| **Interpolation** | Enable 1-tick interpolation for smooth rendering |
| **Ping Frequency** | Seconds between ping messages |
| **Rollback Ticks** (debug) | Force N ticks of rollback every frame for testing |
| **Random Rollback Ticks** (debug) | Random rollback depth for stress testing |

---

### MonkeNet

**Repository:** [github.com/grazianobolla/godot-monke-net](https://github.com/grazianobolla/godot-monke-net)
**License:** MIT
**Language:** C# (.NET 8 required)
**Architecture:** Client-authoritative server
**Dependency:** [ImGui Godot](https://github.com/pkdawson/imgui-godot) (for debug overlays)

#### Important Caveat: Requires Custom Godot Build

MonkeNet uses a [custom fork of Godot](https://github.com/grazianobolla/godot) that exposes `PhysicsServer2D/3D::space_step()` for manual physics stepping. This is based on an unmerged Godot PR ([#76462](https://github.com/godotengine/godot/pull/76462): *"Add PhysicsServer2/3D::space_step() to step physics simulation manually"*). Vanilla Godot does not support stepping the physics world manually, which is required for proper rollback of physics-based movement.

#### Client-Authoritative Approach

Unlike Netfox (server-authoritative) and Godot Rollback Netcode (peer-to-peer), MonkeNet implements a **client-authoritative server** pattern:

- Each client runs its own physics simulation and is authoritative over its own character.
- The server receives client states and distributes them to other clients.
- The server can validate client states but does not re-simulate.
- Other clients' entities are displayed using **snapshot interpolation**.

#### Component Architecture

MonkeNet uses a paired client/server component pattern:

| Functionality | Client Component | Server Component |
|--------------|-----------------|-----------------|
| Entity management | `ClientEntityManager` | `ServerEntityManager` |
| Network clock | `ClientNetworkClock` | `ServerNetworkClock` |
| Input handling | `InputManager` | `InputReceiver` |
| Interpolation | `SnapshotInterpolator` | -- |
| Rollback | `SnapshotRollbacker` | -- |

#### Features

- CharacterBody client-side prediction and reconciliation
- Snapshot interpolation for smooth remote entity rendering
- Clock synchronization between server and clients
- State replication between clients
- ImGui-based debug visualization

#### Planned / Missing Features

- Delta compression for inputs/entity states (not yet implemented)
- Lag compensation for client-to-client interactions (not yet implemented)

---

## Code Examples

### 1. Basic MultiplayerSynchronizer Setup (No Prediction)

This is the baseline Godot approach -- simple property replication with no prediction or interpolation.

```gdscript
# player.gd -- Attached to CharacterBody3D
extends CharacterBody3D

@export var speed = 5.0

var gravity = ProjectSettings.get_setting(&"physics/3d/default_gravity")

func _physics_process(delta):
    if not is_multiplayer_authority():
        # Not our player -- MultiplayerSynchronizer handles updates
        return

    if not is_on_floor():
        velocity.y -= gravity * delta

    var input_dir = Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    var direction = (transform.basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()
    if direction:
        velocity.x = direction.x * speed
        velocity.z = direction.z * speed
    else:
        velocity.x = move_toward(velocity.x, 0, speed)
        velocity.z = move_toward(velocity.z, 0, speed)

    move_and_slide()
```

```gdscript
# player_input.gd -- Attached to the input MultiplayerSynchronizer
extends MultiplayerSynchronizer

@export var jumping := false
@export var direction := Vector2()

func _ready():
    set_process(get_multiplayer_authority() == multiplayer.get_unique_id())

@rpc("call_local")
func jump():
    jumping = true

func _process(_delta):
    direction = Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    if Input.is_action_just_pressed("ui_accept"):
        jump.rpc()
```

**Scene tree:**
```
Player (CharacterBody3D)
├── MultiplayerSynchronizer          # Syncs: position, velocity
│   └── ReplicationConfig: position (Always), velocity (On Change)
├── PlayerInput (MultiplayerSynchronizer)  # Syncs: direction, jumping
├── CollisionShape3D
└── MeshInstance3D
```

**Limitations:** Remote players see snappy, jittery movement. The local player feels responsive, but only because they have full authority. No reconciliation occurs.

### 2. Manual Prediction Implementation (No Addon)

Adding basic client-side prediction on top of `MultiplayerSynchronizer`:

```gdscript
# manual_predicted_player.gd
extends CharacterBody3D

@export var speed := 5.0

# State buffer for reconciliation
var state_buffer: Array[Dictionary] = []
var input_buffer: Array[Dictionary] = []
var last_server_tick: int = 0
var current_tick: int = 0

# Synced by MultiplayerSynchronizer (from server)
var server_position: Vector3
var server_velocity: Vector3
var server_tick: int

func _physics_process(delta):
    current_tick += 1

    if is_multiplayer_authority():
        # --- Server: apply received input, simulate, broadcast state ---
        _apply_input_and_simulate(delta)
        server_position = position
        server_velocity = velocity
        server_tick = current_tick
    else:
        if multiplayer.get_unique_id() == _get_owning_peer():
            # --- Owning client: predict locally, reconcile with server ---
            _client_predict(delta)
        else:
            # --- Other clients: interpolate toward server state ---
            _interpolate_remote(delta)

func _client_predict(delta):
    # Gather and store input
    var input = _gather_input()
    input_buffer.append({"tick": current_tick, "input": input})

    # Apply input locally (prediction)
    _apply_movement(input, delta)

    # Store predicted state
    state_buffer.append({
        "tick": current_tick,
        "position": position,
        "velocity": velocity,
    })

    # Reconcile when server state arrives
    if server_tick > last_server_tick:
        last_server_tick = server_tick
        _reconcile()

func _reconcile():
    # Find the predicted state for the server's tick
    var server_state_idx = -1
    for i in range(state_buffer.size()):
        if state_buffer[i]["tick"] == server_tick:
            server_state_idx = i
            break

    if server_state_idx == -1:
        return

    # Check if prediction was wrong
    var predicted = state_buffer[server_state_idx]
    var error = predicted["position"].distance_to(server_position)

    if error > 0.01:  # Threshold
        # Snap to server state
        position = server_position
        velocity = server_velocity

        # Replay all inputs after the server tick
        var replay_inputs = []
        for entry in input_buffer:
            if entry["tick"] > server_tick:
                replay_inputs.append(entry)

        for entry in replay_inputs:
            _apply_movement(entry["input"], get_physics_process_delta_time())

    # Prune old buffer entries
    state_buffer = state_buffer.filter(func(s): return s["tick"] > server_tick)
    input_buffer = input_buffer.filter(func(i): return i["tick"] > server_tick)

func _interpolate_remote(delta):
    position = position.lerp(server_position, 10.0 * delta)

func _gather_input() -> Dictionary:
    return {
        "direction": Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down"),
    }

func _apply_movement(input: Dictionary, delta: float):
    var dir = input.get("direction", Vector2.ZERO)
    velocity.x = dir.x * speed
    velocity.z = dir.y * speed
    if not is_on_floor():
        velocity.y -= ProjectSettings.get_setting(&"physics/3d/default_gravity") * delta
    move_and_slide()

func _apply_input_and_simulate(delta):
    # Server would receive input via RPC and apply it
    _apply_movement(_gather_input(), delta)

func _get_owning_peer() -> int:
    return get_meta("peer_id", 1)
```

**Limitations:** This is complex, error-prone, and does not handle physics interactions between predicted entities. Netfox and Godot Rollback Netcode solve these problems systematically.

### 3. Using Netfox for Prediction and Reconciliation

The same gameplay with Netfox requires significantly less boilerplate:

```gdscript
# player_input.gd
extends Node
class_name PlayerInput

var movement: Vector3 = Vector3.ZERO

func _ready():
    NetworkTime.before_tick_loop.connect(_gather)

func _gather():
    if not is_multiplayer_authority():
        return

    var mx = Input.get_axis("move_left", "move_right")
    var mz = Input.get_axis("move_forward", "move_back")
    movement = Vector3(mx, 0, mz)
```

```gdscript
# player.gd
extends CharacterBody3D

@export var speed := 5.0
@export var input: PlayerInput

var gravity = ProjectSettings.get_setting(&"physics/3d/default_gravity")

func _rollback_tick(delta, _tick, _is_fresh):
    if not is_on_floor():
        velocity.y -= gravity * delta

    var dir = (transform.basis * Vector3(input.movement.x, 0, input.movement.z)).normalized()
    velocity.x = dir.x * speed
    velocity.z = dir.z * speed

    velocity *= NetworkTime.physics_factor
    move_and_slide()
    velocity /= NetworkTime.physics_factor
```

That is the complete gameplay code. The `RollbackSynchronizer` node (configured in the editor) handles:
- Recording state each tick
- Sending input to the server
- Receiving authoritative state from the server
- Detecting mismatches
- Rolling back and replaying ticks

**Scene tree:**
```
Player (CharacterBody3D)                # Script: player.gd
├── Input (Node)                        # Script: player_input.gd
├── RollbackSynchronizer
│   ├── State: :position, :velocity
│   └── Input: Input:movement
├── TickInterpolator
│   └── Properties: :position
├── CollisionShape3D
└── MeshInstance3D
```

### 4. Rollback Netcode Setup (Godot Rollback Netcode / Snopek Games)

```gdscript
# game_manager.gd -- Match lifecycle
extends Node

func _ready():
    multiplayer.peer_connected.connect(_on_peer_connected)
    multiplayer.peer_disconnected.connect(_on_peer_disconnected)
    SyncManager.sync_started.connect(_on_sync_started)
    SyncManager.sync_stopped.connect(_on_sync_stopped)
    SyncManager.sync_error.connect(_on_sync_error)

func _on_peer_connected(peer_id: int):
    SyncManager.add_peer(peer_id)

func _on_peer_disconnected(peer_id: int):
    # Handle disconnection
    SyncManager.stop()
    SyncManager.clear_peers()

func start_match():
    if multiplayer.is_server():
        SyncManager.start()

func _on_sync_started():
    print("Match started!")

func _on_sync_stopped():
    SyncManager.clear_peers()

func _on_sync_error(msg: String):
    printerr("Sync error: ", msg)
    SyncManager.stop()
    SyncManager.clear_peers()
```

```gdscript
# fighter.gd -- A fighting game character using Godot Rollback Netcode
extends CharacterBody2D

const SPEED = 400.0
const JUMP_FORCE = -600.0

func _ready():
    add_to_group("network_sync")

func _save_state() -> Dictionary:
    return {
        "px": position.x,
        "py": position.y,
        "vx": velocity.x,
        "vy": velocity.y,
    }

func _load_state(state: Dictionary) -> void:
    position = Vector2(state["px"], state["py"])
    velocity = Vector2(state["vx"], state["vy"])

func _get_local_input() -> Dictionary:
    var input := {}
    var dir = Input.get_axis("move_left", "move_right")
    if dir != 0:
        input["dir"] = dir
    if Input.is_action_just_pressed("jump") and is_on_floor():
        input["jump"] = true
    if Input.is_action_just_pressed("punch"):
        input["punch"] = true
    return input

func _predict_remote_input(prev: Dictionary, _ticks_since: int) -> Dictionary:
    # Keep directional movement, don't predict actions
    var predicted := {}
    if prev.has("dir"):
        predicted["dir"] = prev["dir"]
    return predicted

func _network_process(input: Dictionary) -> void:
    # Gravity
    velocity.y += 980.0 * (1.0 / Engine.physics_ticks_per_second)

    # Horizontal movement
    var dir = input.get("dir", 0.0)
    velocity.x = dir * SPEED

    # Jump
    if input.get("jump", false) and is_on_floor():
        velocity.y = JUMP_FORCE

    # Attack
    if input.get("punch", false):
        _do_punch()

    move_and_slide()

func _interpolate_state(old_state: Dictionary, new_state: Dictionary, weight: float) -> void:
    position.x = lerpf(old_state["px"], new_state["px"], weight)
    position.y = lerpf(old_state["py"], new_state["py"], weight)

func _do_punch():
    # Attack logic...
    pass
```

---

## Trade-offs Comparison

| Criteria | MultiplayerSynchronizer (Built-in) | Netfox | Godot Rollback Netcode | MonkeNet |
|----------|-----------------------------------|--------|----------------------|----------|
| **Architecture** | Client-server replication | Client-server with CSP | Peer-to-peer rollback (GGPO-style) | Client-authoritative server |
| **Language** | GDScript / C# | GDScript (C# via NetfoxSharp) | GDScript (C# via Mono addon) | C# only (.NET 8) |
| **Prediction** | None | Built-in CSP via `RollbackSynchronizer` | Built-in via `_network_process` + rollback | Built-in via `SnapshotRollbacker` |
| **Interpolation** | None | `TickInterpolator` node | `_interpolate_state()` virtual method | `SnapshotInterpolator` component |
| **Determinism required** | No | No (server is authority) | **Yes** (all peers must match) | No |
| **Physics rollback** | N/A | Supported (re-runs `move_and_slide`) | Supported (manual save/load) | Requires custom Godot build |
| **Ease of setup** | Very easy (inspector-only) | Easy (a few nodes + simple scripts) | Moderate (group membership + virtual methods) | Hard (custom Godot build + .NET 8 + ImGui) |
| **Boilerplate** | Minimal | Low | Moderate (save/load state, input methods) | Moderate |
| **Debug tooling** | None | Rollback Debugger | Log Inspector + replay system | ImGui overlays |
| **Spawning** | `MultiplayerSpawner` | Uses Godot's built-in spawner | `SyncManager.spawn()` (rollback-safe) | `EntityManager` components |
| **Sound handling** | N/A | One-shot via `is_fresh` flag | `SyncManager.play_sound()` (dedup during rollback) | Manual |
| **Animation** | N/A | Manual with `is_fresh` flag | `NetworkAnimationPlayer` (rollback-aware) | Manual |
| **RNG** | N/A | Manual | `NetworkRandomNumberGenerator` (deterministic) | Manual |
| **NAT traversal** | None | `netfox.noray` addon | None | None |
| **Maturity** | Engine built-in, stable | Active development, used in shipped games | Mature, used in shipped games | Early stage / experimental |
| **Best for** | Simple/casual multiplayer | Action games, FPS, general purpose | Fighting games, 1v1/small player count | Physics-heavy games with C# |
| **Max players** | Many (server-limited) | Many (server-limited) | 2-4 (P2P bandwidth) | Many (server-limited) |
| **Godot version** | 4.0+ (built-in) | 4.x | 3.x and 4.x | 4.x (custom fork only) |

### When to Use What

- **MultiplayerSynchronizer alone** -- Turn-based games, cooperative games, casual games where 50-100ms of latency is acceptable, or prototypes.

- **Netfox** -- The most versatile option. Best for action games, FPS, third-person shooters, platformers, and any client-server game needing responsive controls. Lowest barrier to entry for prediction/rollback. Actively maintained with a growing community.

- **Godot Rollback Netcode** -- The gold standard for fighting games and any genre where deterministic simulation and peer-to-peer play are required. More complex to set up (determinism is hard), but provides the most robust rollback implementation with excellent debugging tools.

- **MonkeNet** -- Interesting for physics-heavy games, but the requirement for a custom Godot build makes it impractical for most production use. Best considered as a reference implementation or for personal projects where building Godot from source is acceptable.

---

## References

### Official Godot Documentation
- [MultiplayerSynchronizer Class Reference](https://docs.godotengine.org/en/stable/classes/class_multiplayersynchronizer.html)
- [Multiplayer in Godot 4.0: Scene Replication (Blog Article)](https://godotengine.org/article/multiplayer-in-godot-4-0-scene-replication/)
- [High-Level Multiplayer Tutorial](https://docs.godotengine.org/en/stable/tutorials/networking/high_level_multiplayer.html)

### Proposals
- [Proposal #7280: Add property prediction and interpolation, and a way to capture MultiplayerSynchronization events](https://github.com/godotengine/godot-proposals/issues/7280)
- [PR #76462: Add PhysicsServer2/3D::space_step() for manual physics stepping](https://github.com/godotengine/godot/pull/76462)

### Netfox
- [GitHub Repository](https://github.com/foxssake/netfox)
- [Documentation](https://foxssake.github.io/netfox/)
- [Netfox Sharp (C# Support)](https://github.com/CyFurStudios/NetfoxSharp)
- [Godot Asset Library -- netfox](https://godotengine.org/asset-library/asset/2375)
- [Godot Asset Library -- netfox.noray](https://godotengine.org/asset-library/asset/2376)
- [Godot Asset Library -- netfox.extras](https://godotengine.org/asset-library/asset/2377)
- [Forest Brawl Example Game](https://github.com/foxssake/netfox/tree/main/examples/forest-brawl)
- [Godot Rocket League (Physics Rollback Example)](https://github.com/albertok/godot-rocket-league)
- [noray (NAT Traversal Relay)](https://github.com/foxssake/noray)

### Godot Rollback Netcode (Snopek Games)
- [GitLab Repository](https://gitlab.com/snopek-games/godot-rollback-netcode)
- [Godot Asset Library](https://godotengine.org/asset-library/asset/2450)
- [Video Tutorial Playlist (YouTube)](https://www.youtube.com/playlist?list=PLCBLMvLIundBXwTa6gwlOUNc29_9btoir)
- [GodotRollbackNetcodeMono (C# Support)](https://github.com/Fractural/GodotRollbackNetcodeMono)
- [Retro Tank Party (Game Using This Addon)](https://www.snopekgames.com/games/retro-tank-party)

### MonkeNet
- [GitHub Repository](https://github.com/grazianobolla/godot-monke-net)
- [Custom Godot Fork (Manual Physics Stepping)](https://github.com/grazianobolla/godot)
- [Discord Community](https://discord.gg/EmyhsVZCnZ)

### Community Discussions
- [Godot Forum: Netfox -- Addons for Online Multiplayer Games](https://forum.godotengine.org/t/netfox-addons-for-online-multiplayer-games/36066)
- [Godot Forum: Interpolation with MultiplayerSynchronizer](https://forum.godotengine.org/t/is-it-possible-to-achieve-network-interpolation-with-multiplayer-synchronizer/48514)
- [Godot Forum: Validating Client-Side Prediction with MultiplayerSynchronizer](https://forum.godotengine.org/t/validating-client-side-prediction-with-multiplayersynchronizer/113738)
