# Overwatch: Gameplay Architecture and Netcode

## Overview

**Speaker:** Timothy Ford, Lead Gameplay Engineer at Blizzard Entertainment
**Event:** GDC 2017 (Game Developers Conference)
**Talk Title:** "Overwatch Gameplay Architecture and Netcode"

In this talk, Tim Ford explains how Overwatch uses a cutting-edge **Entity Component System (ECS)** architecture to create a rich variety of layered gameplay, how this networked simulation works, and how it leverages **determinism** to achieve responsiveness and precision. The ECS architecture is central to nearly every networking feature: prediction, rollback, replay, and the killcam system all depend on the clean separation of data and behavior that ECS provides.

A companion talk, **"Networking Scripted Weapons and Abilities in Overwatch"** by Dan Reed (Senior Gameplay Engineer), covers the **Statescript** visual scripting language used for hero weapons and abilities, where prediction and replication of script behavior is automated for designers.

---

## 1. ECS (Entity Component System) Architecture

Overwatch's engine is built on a strict Entity Component System where data and behavior are fully separated.

### Core Elements

| Concept | Description |
|---------|-------------|
| **Entity** | A 32-bit unsigned integer ID (not a pointer). Functions as a lifetime management handle that corresponds to a collection of components. |
| **Component** | Pure data -- no methods, no behavior. Hundreds of component subclasses exist. Examples: `MovementState`, `Transform`, `ModifyHealthQueue`, `DynamicPhysicsComponent`, `InputState`, `ConnectionComponent`. |
| **System** | Pure behavior -- no internal state, no member variables. Systems iterate over "component tuples" (sets of components that match the system's requirements). |
| **World (EntityAdmin)** | The top-level container that manages all systems and entities via hash tables. |

### Component Tuples and Filtering

Systems declare which component types they require. Only entities possessing **all** required components are processed. This naturally decouples modules:

```
// Pseudo-code: A system declares its required component tuple
class PhysicsSystem : System {
    // Only processes entities that have ALL of these components
    using Tuple = ComponentTuple<DynamicPhysicsComponent, Transform, Contact>;

    void UpdateFixed(float dt) {
        for (auto& [physics, transform, contact] : EntityAdmin.GetTuples<Tuple>()) {
            // simulate physics for this entity
            SimulatePhysics(physics, transform, contact, dt);
        }
    }
};
```

The collision system ignores connection status; AFK detection ignores rendering data. Each system operates only on the components it cares about.

### Singleton Components

Approximately **40% of Overwatch's components are singletons** -- single-instance components that store global or shared state. Examples:

- **`InputSingleton`** -- stores the current player's keystroke/controller state
- **`ContactSingleton`** -- queues pending collision records for deferred processing

Singletons solved a key coupling problem: when multiple systems need access to shared data, singletons belong to a single anonymous entity and are accessed directly through `EntityAdmin`.

### Deferred Side Effects

Complex side effects are not executed immediately. Instead, they are queued into singleton components and processed at specific frame points:

```
// Pseudo-code: Deferred contact resolution
class ProjectileSystem : System {
    void UpdateFixed(float dt) {
        for (auto& projectile : GetProjectiles()) {
            if (CheckCollision(projectile)) {
                // Don't resolve immediately -- queue it
                ContactSingleton.Queue(projectile, hitInfo);
            }
        }
    }
};

// Later in the frame:
class ResolveContactSystem : System {
    void UpdateFixed(float dt) {
        for (auto& contact : ContactSingleton.Drain()) {
            ApplyDamage(contact);
            SpawnEffects(contact);
        }
    }
};
```

This pattern enables performance budgeting, multithreading (e.g., fork-join for particle effects), and deterministic ordering.

### Utility Functions

Shared logic that spans multiple systems is extracted into utility functions that operate on minimal, typically read-only component sets:

- **`CombatUtilityIsHostile(entityA, entityB)`** -- determines hostility via Filter bits (team index), PET Master components, and PET components
- **`CharacterMoveUtil(movementState, input, dt)`** -- shared movement logic

These have limited call points to minimize coupling and side effects.

---

## 2. Server-Authoritative Model

Overwatch uses a **server-authoritative** architecture. The two key constraints are:

1. **The client must react immediately to user input** (responsiveness)
2. **The server must have the authoritative game state** (security and consistency)

### How It Works

- The **server** runs the full authoritative simulation.
- **Clients only send their controller inputs** (button presses, mouse movements, stick positions) -- never game state.
- **Clients also run their own local simulation** (prediction) to provide immediate feedback.
- When the server's authoritative state arrives and disagrees with the client's predicted state, the client is corrected.

```
// Pseudo-code: Client frame loop
void ClientFrame() {
    Input input = SampleInput();

    // Send input to server
    NetworkSend(input, currentCommandFrame);

    // Also store input locally for prediction
    inputHistory.Store(currentCommandFrame, input);

    // Predict locally -- run the same simulation the server will run
    SimulateCommandFrame(localGameState, input);
}
```

The server never trusts the client's reported position, health, or ability state. Clients cannot cheat by reporting false game state because they only send raw inputs.

---

## 3. Client Timing: Running Ahead of the Server

### The Core Idea

Assuming a steady output stream, the **client's clock is always ahead of the server by half-RTT plus one buffered command frame**.

```
Client Time = Server Time + (RTT / 2) + BufferFrames
```

This means: if the server is currently on command frame 9, and the client has 60ms ping (30ms one-way), the client is locally simulating around command frame 11-12. The client sends its input early enough that it arrives at the server "just in time" for the server to use it on the correct command frame.

### Command Frames and Fixed Timestep

Both client and server run at a **fixed timestep** (called "command frames"):

- **Live servers:** 62.5 Hz (~16ms per command frame)
- **Esports/competitive:** up to ~143 Hz (~7ms per command frame)

The `UpdateFixed` function is called on every single fixed command frame. This deterministic time quantization is essential for synchronization.

### Dynamic Time Adjustment

Network latency fluctuates, so the client must dynamically adjust its simulation speed:

- The **server periodically tells the client** what the client's offset is relative to the server.
- If the client is running too far behind, it **speeds up** (e.g., running at ~65 FPS equivalent instead of 60 FPS), generating slightly more command frames without increasing traffic volume.
- If the client is too far ahead, it **slows down**.

This is referred to as **time compression/dilation**. The goal is to keep the server on the "bleeding edge" of client inputs -- arriving just in time, not too early and not too late.

### Input Sliding Window

To handle packet loss, the client sends a **sliding window** of inputs -- approximately the **last 10 command frames** of input. This way, even if a packet is lost, the server can reconstruct the missing inputs from a later packet that contains the overlapping window.

```
// Pseudo-code: Input packet structure
struct InputPacket {
    uint32 latestCommandFrame;
    // Sliding window: inputs from (latestCommandFrame - windowSize) to latestCommandFrame
    Input inputs[WINDOW_SIZE]; // ~10 frames
};
```

---

## 4. Prediction System

### What IS Predicted (on the owning client)

The local player's simulation is predicted immediately on the client without waiting for server confirmation:

- **Movement** -- WASD/stick input is applied locally; the player sees immediate movement
- **Abilities** -- cooldowns start, state transitions happen, effects play
- **Weapon fire** -- muzzle flash, animations, sounds
- **Projectile spawning** -- the client spawns a local "predicted" projectile entity (see section 5)
- **Hit detection** -- hitscan weapons resolve hits locally for instant feedback (visual/audio)

### What is NOT Predicted

- **Other players** -- remote players are **interpolated** between server snapshots, not predicted. The client receives authoritative positions from the server and smoothly interpolates between them.
- **Server-only events** -- kill confirmations, match state changes, and score updates come exclusively from the server.

Remote players are displayed at a position that is slightly in the past (by the interpolation delay, typically ~2 ticks). This is why "favor the shooter" sometimes means you can die after you think you've gone around a corner.

### Misprediction Detection

After each server snapshot arrives, the client compares its predicted state against the server's authoritative state:

```
// Pseudo-code: Misprediction check
void OnServerSnapshotReceived(ServerSnapshot snapshot) {
    uint32 serverFrame = snapshot.commandFrame;

    // Compare predicted state at that frame with server's authoritative state
    PredictedState predicted = predictionHistory.Get(serverFrame);

    if (predicted.movementState != snapshot.movementState) {
        // MISPREDICTION -- need to rollback and replay
        HandleMisprediction(snapshot, serverFrame);
    }
}
```

### Misprediction Handling: Rollback and Replay

When a misprediction is detected:

1. **Rollback** the relevant components to the server's authoritative state at the server's command frame.
2. **Replay** all stored inputs from that frame forward to the current predicted frame, re-simulating deterministically.

Because components are **pure data** in ECS, taking snapshots and rolling back is straightforward -- it is essentially a memcpy of component data.

```
// Pseudo-code: Rollback and replay
void HandleMisprediction(ServerSnapshot snapshot, uint32 serverFrame) {
    // Step 1: Rollback -- restore server state
    localMovementState = snapshot.movementState;

    // Step 2: Replay -- re-simulate from serverFrame to current frame
    for (uint32 frame = serverFrame + 1; frame <= currentCommandFrame; frame++) {
        Input historicInput = inputHistory.Get(frame);
        SimulateCommandFrame(localGameState, historicInput);
    }
}
```

**Partial rollback:** Not all components are rolled back. `MovementState` is rolled back because it needs correction, while some non-movement state (e.g., cosmetic state) may not need rollback. The ECS makes this selective rollback natural because components are independent data containers.

### High-Latency Threshold

When ping exceeds **~220ms**, the client **stops predicting hit events** in advance. Instead, it waits for the server to confirm hits before displaying damage effects. This prevents extreme cases where the victim perceives unfair "pulled-back" damage from hits that were predicted far in the past.

---

## 5. Predicted Projectiles

One of the most notable achievements discussed in the talk is that Overwatch predicts **slow-moving projectiles** (e.g., Pharah's rockets, Junkrat's grenades, Hanzo's arrows) on the client, something other major FPS studios had stated was not feasible.

### How Predicted Projectiles Work

1. **Client fires:** When the player presses fire, the client immediately spawns a **"fake" (predicted) projectile entity** locally. This projectile simulates deterministically -- given an origin point and direction vector, every projectile behaves identically.

2. **Input sent to server:** The fire input (with spawn location, rotation, and a unique projectile ID) is sent to the server via the normal input pipeline.

3. **Server spawns the authoritative projectile:** The server creates the real projectile. Because of latency, the server's projectile spawns "behind" where the client's predicted projectile already is. The server **fast-forwards** the authoritative projectile by approximately `(ClientBias * Ping / 2)` to bring it closer to where the client expects it to be.

4. **Synchronization:** When the authoritative projectile data arrives back at the client, the predicted (fake) projectile is **lerped** toward the authoritative position over time until they converge.

5. **Remote clients:** When other clients receive the replicated projectile, they can **rewind** it to its spawn location and **resimulate** its trajectory for smooth appearance.

```
// Pseudo-code: Predicted projectile spawning
void OnFireInput(Input input) {
    // CLIENT: Spawn predicted (fake) projectile immediately
    Entity fakeProjectile = SpawnProjectile(input.aimOrigin, input.aimDirection);
    fakeProjectile.SetPredicted(true);
    predictedProjectiles[nextProjectileId] = fakeProjectile;

    // Send fire command to server
    SendToServer(FireCommand {
        .origin = input.aimOrigin,
        .direction = input.aimDirection,
        .projectileId = nextProjectileId++,
        .commandFrame = currentCommandFrame
    });
}

// SERVER: On receiving fire command
void OnFireCommandReceived(FireCommand cmd, float clientPing) {
    Entity realProjectile = SpawnProjectile(cmd.origin, cmd.direction);

    // Fast-forward to compensate for latency
    float forwardTime = (clientPing / 2.0f) * clientBiasPct;
    realProjectile.TickMovement(forwardTime);
    realProjectile.lifespan -= forwardTime;
}
```

### Misprediction of Projectiles

If the server **rejects** the fire action (e.g., the player was stunned, out of ammo, or the ability was on cooldown from the server's perspective), the predicted projectile is simply **destroyed** on the client. This manifests as the projectile vanishing -- a visible but rare artifact.

Projectile hit registration also has prediction artifacts: a player might see impact visuals (e.g., Genji's shurikens hitting) but hear no hit sound and deal no damage, because the client incorrectly predicted the hit outcome and the server disagreed.

---

## 6. Deterministic Replay and Kill Cam System

Overwatch's replay and kill cam system leverages the ECS architecture and deterministic simulation.

### Kill Cam Architecture

The kill cam creates a **second parallel ECS World** (`replayAdmin`) on the client:

1. When a player dies, the server sends **8-12 seconds of game data** (the input stream and key state from the killer's perspective).
2. The client instantiates a second `EntityAdmin` (World) alongside the live game world.
3. The renderer switches to the replay world, while the live world continues receiving network updates in the background.
4. All systems run unchanged in the replay world -- they are **unaware** they are replaying rather than running live. This is possible because systems have no internal state; they only operate on component data.

```
// Pseudo-code: Kill cam architecture
class GameClient {
    EntityAdmin liveWorld;    // The real game
    EntityAdmin replayWorld;  // The kill cam / replay

    void OnPlayerDeath(ReplayData replayData) {
        // Create a parallel simulation
        replayWorld.Initialize(replayData.initialState);

        // Feed recorded inputs into the replay world
        for (auto& frame : replayData.frames) {
            replayWorld.InjectInput(frame.input);
            replayWorld.UpdateFixed(FIXED_DT);
        }

        // Switch rendering to replay world
        renderer.SetActiveWorld(replayWorld);
    }
};
```

### Why ECS Enables This

- **Systems have no state**, so the same system code can run against any `EntityAdmin` instance without modification.
- **Components are pure data**, so serialization, snapshot, and restoration are trivial.
- **Deterministic simulation** means that given the same inputs, replaying produces identical results.

This same architecture supports **Play of the Game** highlights and was invaluable internally for **bug reproduction** -- developers can replay exact sequences of events deterministically.

A companion GDC 2017 talk by **Philip Orwig**, **"Replay Technology in Overwatch: Kill Cam, Gameplay, and Highlights"**, goes deeper into how reels are generated and transferred within the hard respawn deadline, and how the client architecture was refactored to support an interruptible kill cam.

---

## 7. Input-to-Command-to-Simulation Decoupling

The ECS architecture creates a clean three-stage pipeline:

```
Input  -->  Commands  -->  Simulation
(raw)       (intent)       (state changes)
```

### Stage 1: Input

Raw controller/keyboard/mouse data is sampled and stored in the `InputSingleton` component. This is hardware-level data: which buttons are pressed, mouse delta, stick positions.

### Stage 2: Commands

Input systems consume the `InputSingleton` and produce **commands** -- high-level gameplay intents stored in components. For example:

- "Move forward" becomes a movement vector in `MovementInput`
- "Press fire" becomes a fire command
- "Press ability 1" triggers a state transition in the ability state machine

### Stage 3: Simulation

Simulation systems consume command components and update game state:

- `MovementSystem` reads `MovementInput` and updates `MovementState` and `Transform`
- `AbilitySystem` reads ability commands and updates ability state machines
- `WeaponSystem` reads fire commands and spawns projectiles or performs hitscan traces

### Why This Decoupling Matters for Networking

Because input and simulation are decoupled:

- **For networking:** The server receives raw inputs and runs the same command and simulation stages. No game logic needs to know whether it is running on the client or server.
- **For replay:** Recorded inputs can be fed into the command stage, and the simulation produces identical results (determinism).
- **For prediction:** The client can re-run the command and simulation stages during rollback/replay using stored historic inputs.
- **For AI:** Bot inputs can be substituted at the input stage without changing any simulation code.

```
// Pseudo-code: The pipeline is the same everywhere
void UpdateFixed(float dt) {
    // Stage 1: Input (from network, local hardware, AI, or replay)
    InputSystem.Update(dt);

    // Stage 2: Commands
    MovementCommandSystem.Update(dt);
    AbilityCommandSystem.Update(dt);
    WeaponCommandSystem.Update(dt);

    // Stage 3: Simulation
    MovementSystem.Update(dt);
    AbilitySystem.Update(dt);
    ProjectileSystem.Update(dt);
    ResolveContactSystem.Update(dt);
    // ... etc
}
```

---

## 8. Netcode Challenges Specific to a Hero Shooter

Overwatch's diverse roster of heroes creates unique networking challenges that a traditional military FPS does not face.

### Diversity of Ability Types

Each hero has fundamentally different mechanics that must all work within the same prediction and networking framework:

| Challenge | Examples |
|-----------|----------|
| **Instant teleportation** | Tracer's Blink, Reaper's Shadow Step -- discrete position changes that are hard to interpolate smoothly for remote players |
| **Channeled abilities** | Roadhog's Hook, Mercy's Beam -- persistent state that must be maintained across network boundaries |
| **Projectile variety** | Pharah's rockets (slow, arcing), Hanzo's arrows (fast, gravity-affected), Junkrat's grenades (bouncing) -- all need prediction |
| **Area denial** | Mei's Ice Wall, Symmetra's turrets -- server-authoritative objects that affect all players' movement |
| **Damage modification** | Damage boost (Mercy), damage resistance (Orisa), anti-heal (Ana) -- applied via `ModifyHealthQueue` component |
| **Movement abilities** | Lucio's wall-ride, Doomfist's Meteor Strike, Pharah's hover -- non-standard movement that must be predicted |
| **Transformation ultimates** | Bastion's tank form, Torbjorn's Overcharge -- hero state completely changes |

### Statescript: Automated Prediction for Abilities

To manage this complexity, Blizzard built **Statescript**, a proprietary visual scripting language for hero ability state machines. Key networking features:

- **Prediction and replication of script behavior is automated.** Common networking problems (rollback, state reconciliation) are handled by the framework, not by individual scripters.
- **New heroes can go from prototype to shippable with little to no new code.** Designers use Statescript to wire up ability logic, and the networking layer handles synchronization.
- This approach addresses **responsiveness, security, bandwidth, seamlessness, and ease of implementation** simultaneously.

### Hit Registration Across Ability Types

Overwatch uses **"favor the shooter"** as a core philosophy:

- **Hitscan weapons** (Widowmaker, McCree/Cassidy): The server performs **backwards reconciliation** -- rewinding `MovementState` of target entities to the attacker's reported command frame to validate the hit from the shooter's perspective.
- **Projectile weapons**: Hit detection is server-authoritative. Any entity with a `MovementState` component is rewound during hit validation.
- The `ModifyHealthQueue` component queues all damage and healing effects rather than applying them immediately, allowing the server to validate before effects manifest.

### Logical Hit Boundaries

Because of network latency, the server uses **logical boundaries** for hit detection -- representing the union of entity position snapshots over approximately the last half-second. This accounts for the spatial uncertainty inherent in networked play.

---

## 9. Implementation Concepts

### The UpdateFixed / UpdateDynamic Split

Overwatch separates its update loop into two categories:

```
// Pseudo-code: Frame structure
void Frame() {
    // Fixed-rate updates: deterministic simulation (60 Hz / 16ms)
    // Multiple UpdateFixed calls may occur per render frame
    while (accumulator >= FIXED_DT) {
        UpdateFixed(FIXED_DT);  // Networked simulation
        accumulator -= FIXED_DT;
    }

    // Dynamic-rate updates: rendering, VFX, UI (matches display refresh)
    UpdateDynamic(renderDt);
}
```

- **`UpdateFixed`**: Called at a fixed rate (every command frame). All networked, deterministic simulation happens here. Movement, abilities, projectiles, hit detection -- everything that must be synchronized.
- **`UpdateDynamic`**: Called at the display refresh rate. Handles rendering, visual effects, UI, and interpolation of remote entities for smooth display.

### Ring Buffer for Prediction History

The client maintains a **ring buffer** of recent states for each predicted component, indexed by command frame:

```
// Pseudo-code: Ring buffer for prediction history
template<typename T>
class PredictionHistory {
    T buffer[HISTORY_SIZE];  // ~256 frames
    uint32 headFrame;

    void Store(uint32 frame, const T& state) {
        buffer[frame % HISTORY_SIZE] = state;
    }

    T& Get(uint32 frame) {
        return buffer[frame % HISTORY_SIZE];
    }
};

// Usage:
PredictionHistory<MovementState> movementHistory;

// Each frame, store the predicted state
movementHistory.Store(currentFrame, entity.Get<MovementState>());

// On server snapshot, compare
if (movementHistory.Get(serverFrame) != serverSnapshot.movementState) {
    // Misprediction! Rollback and replay.
}
```

### Interpolation of Remote Entities

Remote players (not the local player) are displayed using **interpolation** between server snapshots:

```
// Pseudo-code: Interpolation of remote entities
void InterpolateRemoteEntity(Entity entity, float renderTime) {
    // Find the two server snapshots bracketing renderTime
    Snapshot prev = snapshotBuffer.GetBefore(renderTime);
    Snapshot next = snapshotBuffer.GetAfter(renderTime);

    float t = (renderTime - prev.time) / (next.time - prev.time);

    entity.transform.position = Lerp(prev.position, next.position, t);
    entity.transform.rotation = Slerp(prev.rotation, next.rotation, t);
}
```

The interpolation delay is typically **~2 ticks** behind the latest received server snapshot, meaning remote entities are shown at a position slightly in the past.

### Client Delay Formula

```
ClientDelay = (Latency / 2) + InterpolationDelay
```

Where interpolation delay is typically ~2 ticks (~32ms at 62.5 Hz).

---

## 10. References

### Primary GDC Talks (GDC 2017)

- **"Overwatch Gameplay Architecture and Netcode"** -- Timothy Ford
  - GDC Vault: [https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)
  - YouTube: [https://www.youtube.com/watch?v=W3aieHjyNvw](https://www.youtube.com/watch?v=W3aieHjyNvw)

- **"Networking Scripted Weapons and Abilities in Overwatch"** -- Dan Reed
  - GDC Vault: [https://www.gdcvault.com/play/1024653/Networking-Scripted-Weapons-and-Abilities](https://www.gdcvault.com/play/1024653/Networking-Scripted-Weapons-and-Abilities)
  - Slides (PDF): [https://ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc2017/Presentations/Reed_Dan_NetworkingScriptedWeapons.pdf](https://ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc2017/Presentations/Reed_Dan_NetworkingScriptedWeapons.pdf)

- **"Replay Technology in Overwatch: Kill Cam, Gameplay, and Highlights"** -- Philip Orwig
  - GDC Vault: [https://www.gdcvault.com/play/1024053/Replay-Technology-in-Overwatch-Kill](https://www.gdcvault.com/play/1024053/Replay-Technology-in-Overwatch-Kill)

### Articles and Community Analysis

- [Overwatch ECS Architecture Analysis (Moment for Technology)](https://www.mo4tech.com/overwatchs-architecture-is-designed-to-synchronize-with-the-web.html)
- [Overwatch ECS Architecture (Alibaba Cloud)](https://topic.alibabacloud.com/a/on-the-ecs-architecture-in-the-overwatch_8_8_31063753.html)
- [A Guide to Understanding Netcode (GameReplays)](https://www.gamereplays.org/overwatch/portals.php?show=page&name=overwatch-a-guide-to-understanding-netcode)
- [Overwatch Netcode Summary (Nelson's Log)](https://nelsonslog.wordpress.com/2017/01/10/overwatch-netcode/)
- [Overwatch Predicted Rockets Analysis (GameDev.net)](https://www.gamedev.net/forums/topic/701388-overwatch-predicted-rockets-analysis/)
- [Projectile Prediction Implementation (Sam Reitich)](https://sreitich.github.io/projectile-prediction-1/)
- [Gamedeveloper.com -- How Overwatch's Gameplay Architecture Creates Variety](https://www.gamedeveloper.com/design/video-how-i-overwatch-s-i-gameplay-architecture-creates-variety)
