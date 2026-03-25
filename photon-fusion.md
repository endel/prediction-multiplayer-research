# Photon Fusion: Client-Side Prediction, Lag Compensation & Snapshot Interpolation

## Table of Contents

1. [Overview: Fusion vs PUN vs Quantum](#1-overview-fusion-vs-pun-vs-quantum)
2. [Fusion's Networking Modes](#2-fusions-networking-modes)
3. [Client-Side Prediction](#3-client-side-prediction)
4. [Lag Compensation](#4-lag-compensation)
5. [Network Simulation Loop](#5-network-simulation-loop)
6. [Code Examples](#6-code-examples)
7. [Scalability](#7-scalability)
8. [Trade-offs and Comparison to Alternatives](#8-trade-offs-and-comparison-to-alternatives)
9. [References](#9-references)

---

## 1. Overview: Fusion vs PUN vs Quantum

Photon offers three distinct networking products for Unity, each designed for different use cases:

| Feature | PUN (Photon Unity Networking) | Fusion | Quantum |
|---------|-------------------------------|--------|---------|
| **Architecture** | Event/RPC-driven | Tick-based state synchronization | Deterministic lockstep/rollback |
| **Prediction** | None built-in | Full client-side prediction with resimulation | Deterministic rollback |
| **Authority Model** | Client-authoritative (mostly) | Server-authoritative or shared | Server-authoritative (deterministic) |
| **Simulation** | Frame-based | Fixed-tick simulation (`FixedUpdateNetwork`) | Fixed-tick deterministic ECS |
| **State Sync** | Manual via RPCs/serialization | Automatic via `[Networked]` properties | Automatic (deterministic state) |
| **Lag Compensation** | None | Built-in hitbox history + sub-tick accuracy | Rollback-based |
| **Target Games** | Casual, turn-based, simple real-time | FPS, action, competitive real-time | Fighting games, sports, competitive |
| **Scalability** | ~20 players typical | Up to 200 players (with interest management) | 2-8 players typical |
| **Complexity** | Low | Medium-High | High (requires ECS, deterministic math) |

**PUN** is the older, simpler product. It synchronizes state through RPCs and `OnPhotonSerializeView`. There is no tick-based simulation, no built-in prediction, and no lag compensation. It works well for casual multiplayer but struggles with fast-paced competitive games.

**Fusion** is a state synchronization library built around a fixed-tick simulation loop. It provides built-in client-side prediction, resimulation-based reconciliation, lag-compensated hit detection, and snapshot interpolation -- all within Unity's familiar MonoBehaviour/prefab workflow. Data is transferred as partial compressed chunks with eventual consistency.

**Quantum** is a fully deterministic simulation engine using a custom ECS (not Unity's ECS). It requires all game logic to use deterministic fixed-point math. Quantum uses rollback netcode where the entire game state can be rewound and replayed. It excels at precision-critical genres (fighting games, sports) but demands a fundamentally different programming model.

Fusion occupies the sweet spot for most real-time action games: it provides server-authoritative prediction without requiring deterministic simulation, and it integrates naturally with Unity's existing component model.

---

## 2. Fusion's Networking Modes

Fusion supports three network topologies through the same unified API. Game code written for one mode largely works across all three.

### Host Mode (Client-Host)

One player acts as both client and authoritative server. The host runs the simulation authoritatively and replicates state to all other clients.

- **State Authority**: The host machine
- **Prediction**: Clients predict ahead and resimulate on snapshot receipt
- **Lag Compensation**: Full support (host validates hits against historical hitbox positions)
- **Use Case**: Small to medium sessions where a dedicated server is unnecessary
- **Trade-off**: The host player has zero latency advantage; other players experience round-trip latency

### Server Mode (Dedicated Server)

A dedicated server (headless Unity instance or Photon Cloud) holds state authority. No player has a latency advantage.

- **State Authority**: The dedicated server
- **Prediction**: Clients predict ahead and resimulate on snapshot receipt
- **Lag Compensation**: Full support
- **Use Case**: Competitive games, larger sessions, anti-cheat critical scenarios
- **Trade-off**: Requires server infrastructure; higher operational cost

### Shared Mode

Authority is distributed: each client has state authority over objects it owns. There is no single authoritative simulation.

- **State Authority**: Distributed per-object (creator has authority)
- **Prediction**: Not applicable in the traditional sense (each client is authoritative over its own objects)
- **Lag Compensation**: Not available (each client decides locally what they hit)
- **Use Case**: Cooperative games, social experiences, prototyping
- **Trade-off**: Simpler model but vulnerable to cheating; no server-side hit validation

### Mode Comparison Summary

```
Host Mode:      [Client A (Host)] ←→ [Client B] ←→ [Client C]
                     ↑ State Authority

Server Mode:    [Dedicated Server] ←→ [Client A] ←→ [Client B] ←→ [Client C]
                     ↑ State Authority

Shared Mode:    [Client A] ←→ [Client B] ←→ [Client C]
                 ↑ Auth(A)     ↑ Auth(B)     ↑ Auth(C)
```

---

## 3. Client-Side Prediction

### 3.1 Fixed-Tick Simulation (`FixedUpdateNetwork`)

Fusion's simulation runs at a configurable fixed tick rate (e.g., 60 Hz), completely independent of the rendering frame rate. All networked game logic executes inside `FixedUpdateNetwork()` on `NetworkBehaviour` subclasses:

```csharp
public class PlayerMovement : NetworkBehaviour
{
    public override void FixedUpdateNetwork()
    {
        // This runs once per simulation tick, not once per frame
        transform.position += speed * transform.forward * Runner.DeltaTime;
    }
}
```

Key distinctions from Unity's standard update methods:

| Aspect | `FixedUpdateNetwork()` | `FixedUpdate()` / `Update()` |
|--------|----------------------|------------------------------|
| **Time step** | `Runner.DeltaTime` (fixed, tick-based) | `Time.fixedDeltaTime` / `Time.deltaTime` |
| **Execution frequency** | May run multiple times per frame (resimulation) | Once per physics step / once per frame |
| **State context** | Networked state is set to the correct tick before each call | Always operates on current state |
| **Use for** | All gameplay logic affecting networked state | Rendering, UI, visual effects |

**Critical**: `FixedUpdateNetwork()` may be called many times per rendered frame. On a client, a single render frame might trigger:
- Multiple **resimulation** ticks (replaying from a received server snapshot)
- One or more **forward** ticks (predicting new state)

This means any code in `FixedUpdateNetwork()` must be purely deterministic given its inputs -- it must derive state from networked data rather than accumulating progressive deltas from non-networked sources.

### 3.2 Snapshot Receive, State Reset, and Resimulation Loop

The prediction and reconciliation cycle works as follows:

**On the Server (linear, never rewinds):**
1. Collect inputs from all clients for tick N
2. Call `FixedUpdateNetwork()` on all `NetworkBehaviour` components to advance from tick N to N+1
3. Broadcast compressed state delta (snapshot) to all clients

**On the Client (rewinds and resimulates):**
1. **Receive** a server snapshot for tick S (an authoritative, validated state)
2. **Reset** all `[Networked]` state back to tick S (replacing the client's predicted state for that tick)
3. **Resimulate** by calling `FixedUpdateNetwork()` sequentially for every tick from S+1 up to the client's current predicted tick C
4. **Forward-simulate** one new tick (C+1) using the latest local input

```
Server tick:    ... 98  99  100 101 102 ...
                              ↑ snapshot sent

Client receives snapshot for tick 100 while at predicted tick 104:

  [Reset to tick 100 state]
  [Resimulate tick 101 with stored input]
  [Resimulate tick 102 with stored input]
  [Resimulate tick 103 with stored input]
  [Resimulate tick 104 with stored input]   ← arrives at corrected current state
  [Forward-simulate tick 105 with new input]
```

During resimulation, `[Networked]` properties are automatically restored to their server-validated values at tick S. Local (non-networked) variables are **not** reset -- this is why all gameplay-affecting state must use `[Networked]` properties.

If the client's prediction was correct, resimulation produces the same result and nothing visually changes. If there was a misprediction (e.g., another player's input differed from what was assumed), the state snaps to the corrected value. Smooth visual correction is handled by the interpolation/rendering layer.

### 3.3 How Clients Predict Ahead

The server dynamically tells each client how many ticks ahead to simulate, based on measured round-trip time (RTT):

```
RTT = 4 ticks (2 ticks server→client + 2 ticks client→server)

Server is at tick:  100
Client predicts at: 102  (half RTT ahead)

This ensures the client's input for tick 102 arrives at the server
before the server needs to simulate tick 102.
```

The prediction lead is calculated so that:
- Input collected by the client at tick C arrives at the server before the server reaches tick C
- The client's "current" tick is always ahead of the last confirmed server tick by approximately half the RTT

This prediction offset is adjusted dynamically as network conditions change. If latency increases, the client is instructed to predict further ahead. If latency decreases, the prediction window shrinks.

### 3.4 The `[Networked]` Attribute System

`[Networked]` properties are the backbone of Fusion's state synchronization and prediction system. They define what constitutes the "game state" that gets:
- Replicated from state authority to all clients
- Snapshotted for history (enabling resimulation)
- Automatically reset during resimulation
- Delta-compressed for bandwidth efficiency

```csharp
public class Player : NetworkBehaviour
{
    // Basic types
    [Networked] public float Health { get; set; }
    [Networked] public int Score { get; set; }
    [Networked] public NetworkBool IsAlive { get; set; }

    // Timers (store end-tick, not remaining time -- no per-tick sync needed)
    [Networked] public TickTimer RespawnTimer { get; set; }

    // Strings (fixed capacity)
    [Networked, Capacity(32)]
    public NetworkString<_32> PlayerName { get; set; }

    // Collections
    [Networked, Capacity(8)]
    public NetworkArray<int> Inventory { get; }

    [Networked, Capacity(16)]
    public NetworkDictionary<int, float> Cooldowns { get; }
}
```

**Supported types**: All blittable primitives (`int`, `float`, `byte`, etc.), Unity structs (`Vector3`, `Quaternion`, `Color`, etc.), Fusion types (`NetworkBool`, `TickTimer`, `PlayerRef`, `NetworkButtons`, etc.), enums, user-defined blittable structs, and fixed-size collections.

**Implementation detail**: Properties must be declared as auto-properties (`{ get; set; }`). Fusion's IL weaving generates code at compile time to connect getters/setters directly to the object's networked state memory buffer. This is not reflection-based serialization -- it is direct memory access with zero-copy semantics.

**Modification rule**: `[Networked]` properties should only be modified inside `FixedUpdateNetwork()` on the machine that has `StateAuthority` over the object. Modifying them elsewhere breaks the prediction/resimulation contract.

---

## 4. Lag Compensation

### 4.1 The Problem

In a server-authoritative game, the client renders an interpolated view of the world that is inherently behind the server. When a player fires at a target they can see on screen, that target may have already moved to a different position on the server. Without lag compensation, players would need to "lead" their shots, producing unnatural gameplay.

### 4.2 Hitbox History Storage

Fusion maintains a rolling buffer of historical hitbox positions and rotations. This buffer is configured in `NetworkProjectConfig`:

- **Hitbox Buffer Length (ms)**: How far back in time snapshots are kept (e.g., 200ms covers most reasonable latencies)
- **Hitbox Default Capacity**: Number of hitbox values stored per snapshot (minimum 16)

The system records the position and rotation of every `Hitbox` component at every tick, creating a timeline of where each hitbox was in the past.

**Component hierarchy:**
- **`HitboxRoot`**: Attached to the topmost node of a networked object. Groups all `Hitbox` components (including children). Provides a bounding sphere for broad-phase culling. Maximum of 31 `Hitbox` nodes per root.
- **`Hitbox`**: Individual queryable volumes (sphere or box). Represent body parts or hit regions. Attached to child GameObjects (e.g., head, torso, limbs).

```
PlayerPrefab (NetworkObject + HitboxRoot)
├── Head   (Hitbox - sphere)
├── Torso  (Hitbox - box)
├── LeftArm  (Hitbox - sphere)
└── RightArm (Hitbox - sphere)
```

**Important**: Dynamic Unity `Collider` components (used for physics) must be on **separate layers** from `Hitbox` components. Lag-compensated queries operate on the Hitbox system, not on Unity's physics colliders.

### 4.3 Sub-Tick Accuracy

By default, lag-compensated queries resolve against the closest full-tick snapshot. But player input (e.g., firing a weapon) typically occurs between ticks -- at some fractional point during the render frame.

Fusion captures the exact interpolation factor (alpha) at the moment input was polled. When the `SubtickAccuracy` flag is used, the query interpolates hitbox positions between two adjacent tick snapshots at precisely the alpha value the player was seeing:

```
Tick 100                    Tick 101
  |--------●--------------------|
           ↑
    Player fired here (alpha = 0.3)
    Hitbox position = lerp(pos@100, pos@101, 0.3)
```

This means the server reconstructs exactly what the player saw on their screen at the sub-tick moment they pulled the trigger.

### 4.4 How Lag-Compensated Raycasts Work

When a lag-compensated query executes on the server/host:

1. **Determine the client's perceived time**: Using the `player` parameter (typically `Object.InputAuthority`), Fusion calculates which historical tick corresponds to what that client was seeing when they fired
2. **Broad-phase**: Check each `HitboxRoot`'s bounding sphere against the query volume to quickly eliminate distant objects
3. **Rewind hitboxes**: Retrieve the stored positions/rotations for all candidate hitboxes at the target historical tick
4. **Sub-tick interpolation** (if `SubtickAccuracy` is enabled): Interpolate between the two surrounding tick snapshots using the exact alpha value
5. **Narrow-phase**: Perform the actual raycast/overlap test against the rewound hitbox volumes
6. **Return results**: Hits are returned with references to the `Hitbox`, `HitboxRoot`, hit point, normal, and distance

The available query methods mirror Unity's physics API:

| Method | Description |
|--------|-------------|
| `Runner.LagCompensation.Raycast()` | Single-hit ray query |
| `Runner.LagCompensation.RaycastAll()` | Multi-hit ray query |
| `Runner.LagCompensation.OverlapSphere()` | Spherical volume overlap |
| `Runner.LagCompensation.OverlapBox()` | Box volume overlap |

### 4.5 Mode Differences

- **Host Mode / Server Mode**: Full lag compensation. The host/server validates all hits by rewinding hitboxes to the shooting client's perceived time. Authoritative and cheat-resistant.
- **Shared Mode**: No lag compensation. Each client has authority over its own objects and decides locally what it hits. Precise from the shooter's perspective but vulnerable to cheating.

---

## 5. Network Simulation Loop

The Network Simulation Loop is the core execution model that ties prediction, resimulation, and rendering together.

### 5.1 Loop Phases

Each render frame, Fusion's `NetworkRunner` executes the following:

```
┌─────────────────────────────────────────────────────┐
│                  RENDER FRAME                       │
│                                                     │
│  1. RECEIVE SNAPSHOT (if available)                  │
│     └─ Server state for tick S arrives              │
│                                                     │
│  2. RESIMULATION PHASE                              │
│     └─ Reset [Networked] state to tick S            │
│     └─ For each tick from S+1 to C:                 │
│         └─ Restore stored input for that tick       │
│         └─ Call FixedUpdateNetwork()                │
│         └─ (IsResimulation = true on Runner)        │
│                                                     │
│  3. FORWARD PHASE                                   │
│     └─ Collect new local input                      │
│     └─ Call FixedUpdateNetwork() for tick C+1       │
│     └─ Store input for later resimulation           │
│     └─ Send input to server                         │
│     └─ (IsForward = true on Runner)                 │
│                                                     │
│  4. RENDER PHASE                                    │
│     └─ Interpolate visual state for remote objects  │
│     └─ Call Render() on NetworkBehaviours           │
│     └─ Unity renders the frame                      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 5.2 Tick Types

Within `FixedUpdateNetwork()`, the code can distinguish between tick types:

```csharp
public override void FixedUpdateNetwork()
{
    if (Runner.IsResimulation)
    {
        // This tick is being replayed from history after a server snapshot
        // Avoid spawning particles, playing sounds, etc.
    }

    if (Runner.IsForward)
    {
        // This is a new tick being predicted for the first time
        // Safe to trigger visual/audio effects
    }

    // Gameplay logic runs in both cases
    if (GetInput(out NetworkInputData input))
    {
        transform.position += input.direction * speed * Runner.DeltaTime;
    }
}
```

### 5.3 Rendering and Interpolation

Remote objects (proxies) are not predicted -- they are interpolated between two received snapshots. Fusion maintains a small buffer of recent snapshots and interpolates visual state between them in the `Render()` callback:

```
Snapshot A (tick 98)          Snapshot B (tick 100)
      |__________________________|
              ↑
        Render at interpolated position (alpha = 0.6)
```

This provides smooth visuals regardless of:
- Variable packet arrival times (jitter)
- Tick rate vs frame rate mismatch
- Temporary packet loss (the buffer absorbs gaps)

The interpolation delay is dynamically adjusted based on network conditions -- minimizing visual latency while preventing stutter.

### 5.4 Runner.DeltaTime vs Time.deltaTime

| Property | Value | Use In |
|----------|-------|--------|
| `Runner.DeltaTime` | Fixed: `1.0 / TickRate` | `FixedUpdateNetwork()` |
| `Time.deltaTime` | Variable: actual frame time | `Update()`, `Render()` |

Using `Time.deltaTime` inside `FixedUpdateNetwork()` would break prediction because resimulated ticks would use different time steps than the original simulation.

---

## 6. Code Examples

### 6.1 Basic Networked Movement with Prediction

```csharp
using Fusion;
using UnityEngine;

// Input data structure sent from client to server each tick
public struct PlayerInput : INetworkInput
{
    public Vector3 direction;
    public NetworkButtons buttons;
    public const byte BUTTON_JUMP = 0;
    public const byte BUTTON_FIRE = 1;
}

public class PlayerController : NetworkBehaviour
{
    [SerializeField] private float speed = 6f;

    // Networked state -- automatically synced from StateAuthority to all clients.
    // Reset to server-validated values during resimulation.
    [Networked] public Vector3 Velocity { get; set; }
    [Networked] public NetworkBool IsGrounded { get; set; }

    private CharacterController _cc;

    public override void Spawned()
    {
        _cc = GetComponent<CharacterController>();
    }

    public override void FixedUpdateNetwork()
    {
        // GetInput returns true only for the InputAuthority client and the StateAuthority.
        // For other clients (proxies), this returns false -- they see interpolated state instead.
        if (GetInput(out PlayerInput input))
        {
            Vector3 move = input.direction.normalized * speed;

            if (IsGrounded && input.buttons.IsSet(PlayerInput.BUTTON_JUMP))
            {
                move.y = 8f;
            }

            move.y += Physics.gravity.y * Runner.DeltaTime;
            Velocity = move;

            // Runner.DeltaTime is the fixed tick timestep, NOT Time.deltaTime
            _cc.Move(Velocity * Runner.DeltaTime);
            IsGrounded = _cc.isGrounded;
        }
    }
}
```

### 6.2 Input Collection (MonoBehaviour on the Client)

```csharp
using Fusion;
using Fusion.Sockets;
using UnityEngine;
using System;
using System.Collections.Generic;

public class InputProvider : MonoBehaviour, INetworkRunnerCallbacks
{
    // Called by Fusion each tick to collect input before FixedUpdateNetwork runs
    public void OnInput(NetworkRunner runner, NetworkInput input)
    {
        var data = new PlayerInput();

        // Sample Unity's input system
        data.direction = new Vector3(
            Input.GetAxisRaw("Horizontal"),
            0f,
            Input.GetAxisRaw("Vertical")
        );

        data.buttons.Set(PlayerInput.BUTTON_JUMP, Input.GetKey(KeyCode.Space));
        data.buttons.Set(PlayerInput.BUTTON_FIRE, Input.GetMouseButton(0));

        // Pass the input struct to Fusion
        input.Set(data);
    }

    // ... other INetworkRunnerCallbacks methods ...
    public void OnPlayerJoined(NetworkRunner runner, PlayerRef player) { }
    public void OnPlayerLeft(NetworkRunner runner, PlayerRef player) { }
    public void OnInputMissing(NetworkRunner runner, PlayerRef player, NetworkInput input) { }
    public void OnConnectedToServer(NetworkRunner runner) { }
    public void OnDisconnectedFromServer(NetworkRunner runner, NetDisconnectReason reason) { }
    public void OnShutdown(NetworkRunner runner, ShutdownReason shutdownReason) { }
    public void OnObjectExitAOI(NetworkRunner runner, NetworkObject obj, PlayerRef player) { }
    public void OnObjectEnterAOI(NetworkRunner runner, NetworkObject obj, PlayerRef player) { }
    public void OnReliableDataReceived(NetworkRunner runner, PlayerRef player, ReliableKey key, ArraySegment<byte> data) { }
    public void OnReliableDataProgress(NetworkRunner runner, PlayerRef player, ReliableKey key, float progress) { }
    public void OnSceneLoadDone(NetworkRunner runner) { }
    public void OnSceneLoadStart(NetworkRunner runner) { }
    public void OnSessionListUpdated(NetworkRunner runner, List<SessionInfo> sessionList) { }
    public void OnCustomAuthenticationResponse(NetworkRunner runner, Dictionary<string, object> data) { }
    public void OnHostMigration(NetworkRunner runner, HostMigrationToken hostMigrationToken) { }
    public void OnConnectRequest(NetworkRunner runner, NetworkRunnerCallbackArgs.ConnectRequest request, byte[] token) { }
    public void OnConnectFailed(NetworkRunner runner, NetAddress remoteAddress, NetConnectFailedReason reason) { }
}
```

### 6.3 Lag-Compensated Raycast

```csharp
using System.Collections.Generic;
using Fusion;
using UnityEngine;

public class WeaponController : NetworkBehaviour
{
    [SerializeField] private float rayLength = 100f;
    [SerializeField] private float damage = 25f;
    [SerializeField] private float cooldownSeconds = 0.2f;
    [SerializeField] private LayerMask hitLayers = -1;

    [Networked] public TickTimer FireCooldown { get; set; }

    // Pre-allocated hit list to avoid GC
    private readonly List<LagCompensatedHit> _hits = new List<LagCompensatedHit>();

    public override void FixedUpdateNetwork()
    {
        if (GetInput(out PlayerInput input))
        {
            if (FireCooldown.ExpiredOrNotRunning(Runner) &&
                input.buttons.IsSet(PlayerInput.BUTTON_FIRE))
            {
                FireCooldown = TickTimer.CreateFromSeconds(Runner, cooldownSeconds);

                // Lag-compensated raycast:
                // - Rewinds hitboxes to what Object.InputAuthority was seeing
                // - Uses sub-tick interpolation for precision
                // - IgnoreInputAuthority prevents shooting yourself
                Runner.LagCompensation.RaycastAll(
                    transform.position,
                    transform.forward,
                    rayLength,
                    player: Object.InputAuthority,  // whose perspective to use
                    _hits,
                    hitLayers,
                    clearHits: true,
                    options: HitOptions.SubtickAccuracy | HitOptions.IgnoreInputAuthority
                );

                for (var i = 0; i < _hits.Count; i++)
                {
                    var hit = _hits[i];

                    // hit.Hitbox gives us the specific hitbox struck
                    // hit.Hitbox.Root gives us the HitboxRoot (the target entity)
                    if (hit.Hitbox.Root.TryGetComponent(out Health health))
                    {
                        health.TakeDamage(damage);
                    }
                }
            }
        }
    }
}
```

### 6.4 Networked State with `[Networked]` Attributes

```csharp
using Fusion;
using UnityEngine;

public class Health : NetworkBehaviour
{
    [Networked] public float CurrentHealth { get; set; } = 100f;
    [Networked] public NetworkBool IsDead { get; set; }
    [Networked] public TickTimer InvulnerabilityTimer { get; set; }

    // Networked collection with fixed capacity
    [Networked, Capacity(8)]
    public NetworkArray<float> DamageLog { get; }

    [Networked] private int _damageLogIndex { get; set; }

    // Change detection for visual feedback
    private ChangeDetector _changeDetector;

    public override void Spawned()
    {
        _changeDetector = GetChangeDetector(ChangeDetector.Source.SimulationState);
    }

    public override void FixedUpdateNetwork()
    {
        // Only state authority modifies networked state
        if (!HasStateAuthority) return;

        if (IsDead && InvulnerabilityTimer.Expired(Runner))
        {
            Respawn();
        }
    }

    // Called from weapon's FixedUpdateNetwork on the state authority
    public void TakeDamage(float amount)
    {
        if (!HasStateAuthority) return;
        if (!InvulnerabilityTimer.ExpiredOrNotRunning(Runner)) return;

        CurrentHealth -= amount;

        // Log damage in networked array
        DamageLog.Set(_damageLogIndex % 8, amount);
        _damageLogIndex++;

        if (CurrentHealth <= 0f)
        {
            CurrentHealth = 0f;
            IsDead = true;
        }
    }

    private void Respawn()
    {
        CurrentHealth = 100f;
        IsDead = false;
        InvulnerabilityTimer = TickTimer.CreateFromSeconds(Runner, 3f);
    }

    public override void Render()
    {
        // Render() runs once per frame (not per tick) -- safe for visuals
        foreach (var change in _changeDetector.DetectChanges(this))
        {
            switch (change)
            {
                case nameof(CurrentHealth):
                    UpdateHealthBar();
                    break;
                case nameof(IsDead):
                    PlayDeathEffect();
                    break;
            }
        }
    }

    private void UpdateHealthBar() { /* update UI */ }
    private void PlayDeathEffect() { /* particles, sound */ }
}
```

### 6.5 Spawning Networked Objects (Server-Authority Only)

```csharp
using Fusion;
using UnityEngine;

public class ProjectileSpawner : NetworkBehaviour
{
    [SerializeField] private NetworkPrefabRef _projectilePrefab;
    [SerializeField] private float fireCooldown = 0.5f;

    [Networked] private TickTimer _cooldown { get; set; }

    public override void FixedUpdateNetwork()
    {
        if (GetInput(out PlayerInput input))
        {
            if (_cooldown.ExpiredOrNotRunning(Runner) &&
                input.buttons.IsSet(PlayerInput.BUTTON_FIRE))
            {
                _cooldown = TickTimer.CreateFromSeconds(Runner, fireCooldown);

                // Only StateAuthority can spawn network objects.
                // In Host/Server mode, this runs on the host/server.
                // Clients predict locally but spawning only executes authoritatively.
                if (HasStateAuthority)
                {
                    Vector3 forward = transform.forward;
                    Runner.Spawn(
                        _projectilePrefab,
                        transform.position + forward * 1.5f,
                        Quaternion.LookRotation(forward),
                        Object.InputAuthority,
                        // Initialization callback runs before the first FixedUpdateNetwork
                        (runner, obj) =>
                        {
                            obj.GetComponent<Projectile>().Init(forward * 50f);
                        }
                    );
                }
            }
        }
    }
}

public class Projectile : NetworkBehaviour
{
    [Networked] private TickTimer _lifetime { get; set; }
    [Networked] private Vector3 _velocity { get; set; }

    public void Init(Vector3 velocity)
    {
        _velocity = velocity;
        _lifetime = TickTimer.CreateFromSeconds(Runner, 5f);
    }

    public override void FixedUpdateNetwork()
    {
        if (_lifetime.Expired(Runner))
        {
            Runner.Despawn(Object);
            return;
        }

        transform.position += _velocity * Runner.DeltaTime;
    }
}
```

---

## 7. Scalability

Fusion claims support for up to **200 concurrent players** in a single session through its Area of Interest (AOI) system.

### Area of Interest

AOI is a fully configurable interest management system that limits which networked objects are replicated to each client based on spatial proximity and game-defined relevance rules.

Without AOI, every client receives state updates for every networked object in the session -- bandwidth scales as O(N^2). With AOI, each client only receives updates for objects within their "area of interest," reducing bandwidth to O(N * K) where K is the number of relevant objects per player.

### Compression

Fusion uses a "state-of-the-art compression algorithm" for state synchronization. Key techniques:
- **Delta compression**: Only changed properties are transmitted, not full snapshots
- **Partial chunks with eventual consistency**: Data is transferred in small pieces; the full state converges over time
- **Quantization**: Floats and vectors can be quantized to reduce precision (and bandwidth) where appropriate

### Bandwidth Considerations at Scale

| Player Count | Without AOI | With AOI |
|-------------|-------------|----------|
| 10 | Manageable | Not needed |
| 50 | Strained | Recommended |
| 100 | Infeasible | Required |
| 200 | Impossible | Feasible |

The actual limit depends on:
- Number of `[Networked]` properties per object
- Rate of state change
- AOI radius and density
- Available server bandwidth
- Tick rate (higher tick rate = more bandwidth)

---

## 8. Trade-offs and Comparison to Alternatives

### Fusion's Strengths

1. **Integrated prediction and reconciliation**: No manual implementation required. The `[Networked]` attribute system and resimulation loop handle prediction automatically.
2. **Lag compensation built in**: Hitbox history, sub-tick accuracy, and temporal queries are first-party features, not third-party add-ons.
3. **Unity-native workflow**: Uses MonoBehaviour, prefabs, and Unity's component model. No custom ECS or separate simulation world required.
4. **Single API across modes**: Code works across Host, Server, and Shared modes with minimal changes.
5. **Snapshot interpolation for proxies**: Remote objects render smoothly without prediction artifacts.

### Fusion's Weaknesses

1. **Unity-only**: Tightly coupled to Unity. Cannot be used with other engines or custom engines.
2. **Closed source**: The core networking layer is a black box. Debugging deep networking issues requires relying on Photon's tools and support.
3. **Vendor lock-in**: Depends on Photon Cloud infrastructure (or self-hosted Photon Server). Switching away requires rewriting the networking layer.
4. **IL weaving complexity**: The `[Networked]` property system uses IL post-processing, which can cause issues with certain build pipelines, code stripping, and debugging.
5. **Prediction limitations**: Cannot predict other players' input. Mispredictions for interactions involving multiple players (e.g., collisions) are inherent and can cause visible corrections.
6. **Cost at scale**: Photon Cloud pricing is usage-based. High-CCU games can incur significant operational costs.
7. **Shared Mode limitations**: No lag compensation, no server-authoritative hit validation. Essentially a different networking model that happens to share the same API surface.

### Comparison with Other Solutions

| Feature | Photon Fusion | Netcode for GameObjects (Unity) | Mirror | Custom (e.g., Colyseus + custom prediction) |
|---------|--------------|-------------------------------|--------|---------------------------------------------|
| **Prediction** | Built-in resimulation | State sync only (no built-in prediction) | RPCs / manual | Full control, manual implementation |
| **Lag Compensation** | Built-in hitbox history | None | None | Full control, manual implementation |
| **Tick Simulation** | Fixed tick with `FixedUpdateNetwork` | `NetworkFixedUpdate` (basic) | Server-tick optional | Fully custom |
| **State Sync** | `[Networked]` attributes, delta compressed | `NetworkVariable`, owner-write | `[SyncVar]`, RPCs | Custom serialization |
| **Interpolation** | Automatic snapshot interpolation | Basic `NetworkTransform` | `NetworkTransform` | Custom interpolation |
| **Platform** | Unity only | Unity only | Unity only | Engine-agnostic (server) |
| **Server Model** | Photon Cloud or self-hosted | Unity relay/dedicated | Self-hosted | Self-hosted |
| **Open Source** | No | Partial | Yes | Yes (typically) |
| **Max Players** | ~200 (with AOI) | ~100 (theoretical) | ~100+ | Depends on implementation |

### When to Choose Fusion

- Fast-paced shooters or action games requiring lag compensation
- Projects that need client-side prediction without writing it from scratch
- Teams comfortable with vendor lock-in for faster time-to-market
- Medium-scale sessions (10-200 players)

### When to Avoid Fusion

- Projects requiring engine portability or open-source networking
- Games where you need full control over the networking layer
- Very large-scale (1000+ player) simulations (consider spatial partitioning with custom solutions)
- Deterministic simulation requirements (consider Photon Quantum instead)
- Budget-constrained projects sensitive to per-CCU cloud costs

---

## 9. References

### Official Documentation

1. **Fusion 2 Introduction**
   https://doc.photonengine.com/fusion/current/fusion-2-intro

2. **Host Mode Basics - Prediction Tutorial**
   https://doc.photonengine.com/fusion/current/tutorials/host-mode-basics/3-prediction

3. **Lag Compensation (Advanced)**
   https://doc.photonengine.com/fusion/current/manual/advanced/lag-compensation

4. **Network Simulation Loop**
   https://doc.photonengine.com/fusion/current/concepts-and-patterns/network-simulation-loop

5. **Networked Properties**
   https://doc.photonengine.com/fusion/current/manual/data-transfer/networked-properties

6. **Network Input**
   https://doc.photonengine.com/fusion/current/manual/data-transfer/network-input

7. **Snapshots and Interpolation**
   https://doc.photonengine.com/fusion/current/manual/data-transfer/snapshots-and-interpolation

8. **Area of Interest**
   https://doc.photonengine.com/fusion/current/manual/data-transfer/area-of-interest

### Related Resources

9. **Photon Fusion SDK (Unity Asset Store / Photon Dashboard)**
   https://www.photonengine.com/fusion

10. **Photon Quantum (Deterministic Alternative)**
    https://www.photonengine.com/quantum

11. **Gabriel Gambetta - Fast-Paced Multiplayer (foundational concepts)**
    https://www.gabrielgambetta.com/client-server-game-architecture.html

12. **Valve Developer Community - Source Multiplayer Networking**
    https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking

13. **Glenn Fiedler - Networked Physics**
    https://gafferongames.com/post/networked_physics_in_virtual_reality/
