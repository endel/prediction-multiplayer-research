# FishNet -- Client-Side Prediction in Unity

## 1. Overview of FishNet

**Fish-Networking (FishNet)** is a free, open-source networking solution for Unity, built from the ground up by FirstGearGames. It positions itself as offering "more features than any other free solution" for Unity multiplayer development.

Key characteristics:

- **Free with no CCU caps or paywalls.** Available on GitHub and the Unity Asset Store.
- **Server-authoritative architecture.** Supports dedicated servers, player-hosted servers, and a host mode where one process acts as both server and client.
- **Transport-agnostic.** Works with any transport layer through a modular Transport system.
- **Built-in prediction.** First-class support for client-side prediction and server reconciliation via the `[Replicate]` and `[Reconcile]` attribute system.
- **High-level and low-level APIs.** Provides rapid state synchronization alongside low-level access for advanced use cases.

---

## 2. FishNet's Networking Model

FishNet uses a **server-authoritative** model, but it draws a distinction between "server-authoritative" and "client-side prediction." In a naive server-authoritative design, all movement is processed solely on the server and the results are transmitted to clients, introducing latency-based delays that penalize players with higher ping. Client-side prediction solves this by letting the client run movement logic locally for immediate responsiveness, while the server retains final authority.

FishNet's prediction system is tick-based. The `TimeManager` drives a fixed-rate tick loop that is synchronized between server and clients. All prediction logic runs within this tick system, not in Unity's `Update()` or `FixedUpdate()`.

### Tick-Based Architecture

| Component | Role |
|-----------|------|
| **TimeManager** | Drives the fixed tick rate; provides `OnTick` and `OnPostTick` events |
| **PredictionManager** | Handles global prediction settings and prediction-related metadata |
| **NetworkObject** | Per-object inspector settings for prediction (must be configured for any object using `[Replicate]`/`[Reconcile]`) |

### Authority Model

- **Owner**: The client that owns a NetworkObject. Generates input and runs local prediction.
- **Server**: Receives owner input, processes it authoritatively, and sends reconciliation data back.
- **Spectators/Other Clients**: Receive state via **state forwarding** from the server and can optionally run prediction locally.

---

## 3. Prediction System in Detail

FishNet's prediction is built around two attributed methods on a `NetworkBehaviour`:

- **`[Replicate]`** -- Processes input. Runs on the owner (locally predicted), server (authoritative), and optionally other clients (state forwarding).
- **`[Reconcile]`** -- Sends authoritative state from server to owner. The owner compares it to its predicted state and, on mismatch, snaps to the server state and replays all unacknowledged inputs.

### 3.1 Replicate Method (Input Handling)

The `[Replicate]` method takes an input struct and applies it to the object's state. It is called every tick.

```csharp
[Replicate]
private void RunInputs(ReplicateData data, ReplicateState state = ReplicateState.Invalid,
    Channel channel = Channel.Unreliable)
{
    Vector3 forces = new Vector3(data.Horizontal, 0f, data.Vertical) * _moveRate;
    PredictionRigidbody.AddForce(forces);

    if (data.Jump)
    {
        Vector3 jumpForce = new Vector3(0f, _jumpForce, 0f);
        PredictionRigidbody.AddForce(jumpForce, ForceMode.Impulse);
    }

    PredictionRigidbody.AddForce(Physics.gravity * 3f);
    PredictionRigidbody.Simulate();
}
```

**Key rules:**

- Always apply forces through `PredictionRigidbody`, never directly on the Unity `Rigidbody`.
- Call `PredictionRigidbody.Simulate()` at the end of the replicate method to process all pending forces.
- The method signature must include `ReplicateState state` and `Channel channel` parameters (even if unused).

### 3.2 Reconcile Method (State Correction)

The `[Reconcile]` method receives authoritative state from the server. On the client, it restores the object to the server-verified state and replays all inputs that occurred after that tick.

```csharp
[Reconcile]
private void ReconcileState(ReconcileData data, Channel channel = Channel.Unreliable)
{
    PredictionRigidbody.Reconcile(data.PredictionRigidbody);
}
```

**Reconciliation is triggered via `CreateReconcile()`:**

```csharp
public override void CreateReconcile()
{
    ReconcileData rd = new ReconcileData(PredictionRigidbody);
    ReconcileState(rd);
}
```

> **Important:** Only the server actually sends the reconcile data, but `CreateReconcile()` must be called on all peers (client, server, owner, non-owner). FishNet handles the routing internally.

### 3.3 ReplicateState Enum and Its Values

`ReplicateState` is a **flags enum** -- a single state value may contain multiple flags combined. It tells you the context in which the `[Replicate]` method is being called.

| Flag | Description |
|------|-------------|
| **Invalid** | Should never occur. Indicates an internal FishNet error in setting the state. |
| **Ticked** | The data tick is running outside a reconcile, such as from user code within `OnTick`. Used by both server and clients during normal forward simulation. |
| **Replayed** | The data is being re-run during a reconcile (input replay). **Only clients use this flag.** The server never reconciles or replays because it is always authoritative. |
| **Created** | The data was created by the controller (the owner, or the server if there is no owner). Indicates real, user-generated input as opposed to predicted/forwarded input. |

**Flag combinations in practice:**

| Combined State | Meaning |
|---------------|---------|
| `Ticked + Created` | Normal forward tick with real input (owner running their own input, or server processing owner's input) |
| `Ticked` (alone) | Normal forward tick with **predicted** or **forwarded** input (non-owner client running state-forwarded data) |
| `Replayed + Ticked + Created` | Replaying real input during reconciliation |
| `Replayed + Ticked` | Replaying predicted/future input during reconciliation |
| `Replayed` (alone) | "Future state" -- replaying beyond the last known real input, using predicted data |

**Checking flags in code:**

```csharp
[Replicate]
private void RunInputs(ReplicateData data, ReplicateState state = ReplicateState.Invalid,
    Channel channel = Channel.Unreliable)
{
    // Check if this is a replay (reconciliation re-simulation)
    if (state.IsReplayed())
    {
        // Skip side effects like sound or particles during replay
        return;
    }

    // Check if the data was created by the actual controller
    if (state.IsCreated())
    {
        // This is real user input
    }
}
```

### 3.4 State Forwarding to Other Clients

State forwarding allows non-owner clients (spectators) to see predicted movement of other players. Without it, spectators would only see interpolated server updates.

With state forwarding enabled:

1. The owner sends input to the server.
2. The server processes the input and forwards the resulting state to all other clients.
3. Other clients run their own `[Replicate]` method with the forwarded data, allowing them to predict the other player's movement locally.

State forwarding is configured on the **NetworkObject** inspector and the **PredictionManager**. When enabled, the `[Replicate]` method fires on all clients for all predicted objects, with the `ReplicateState` flags indicating whether the data is real (`Created`) or forwarded/predicted.

### 3.5 OnTick vs. OnPostTick Timing

FishNet's tick system provides two callback points:

| Callback | When It Fires | Use For |
|----------|--------------|---------|
| `OnTick` | Before physics simulation | Gathering input, calling `[Replicate]` methods, non-physics controllers |
| `OnPostTick` | After physics simulation | Sending reconcile data for physics-based objects |

**Critical rule for physics:** When using `Rigidbody` or `PredictionRigidbody`, send the reconcile during `OnPostTick` because you need the state *after* the physics engine has simulated the forces applied in the replicate method.

For non-physics controllers (e.g., `CharacterController` or transform-based movement), reconcile data can be sent during `OnTick`.

```csharp
public override void OnStartNetwork()
{
    base.TimeManager.OnTick += TimeManager_OnTick;
    base.TimeManager.OnPostTick += TimeManager_OnPostTick;
}

private void TimeManager_OnTick()
{
    RunInputs(CreateReplicateData());   // Apply input (Replicate)
}

private void TimeManager_OnPostTick()
{
    CreateReconcile();                  // Send state after physics (Reconcile)
}
```

---

## 4. PredictionRigidbody Helper Class

`PredictionRigidbody` is a wrapper around Unity's `Rigidbody` designed specifically for networked prediction. It manages velocities, kinematic state, transform properties, and pending forces -- all of which must be tracked for accurate reconciliation and replay.

### Why It Exists

During reconciliation, FishNet must restore a Rigidbody to a previous state and replay inputs. Unity's `Rigidbody` does not natively support snapshotting and restoring its full internal state. `PredictionRigidbody` fills this gap by:

- Tracking all applied forces in a pending queue
- Serializing velocity, angular velocity, position, rotation, and kinematic state
- Providing `Simulate()` to flush pending forces into the Rigidbody
- Providing `Reconcile()` to restore a Rigidbody to a previous snapshot

### Key API

| Method / Property | Purpose |
|-------------------|---------|
| `Initialize(Rigidbody)` | Binds to a Unity Rigidbody |
| `AddForce(Vector3, ForceMode)` | Queues a force (do **not** call `Rigidbody.AddForce` directly) |
| `Velocity(Vector3)` | Sets velocity directly |
| `Simulate()` | Flushes all pending forces into the Rigidbody. **Only call inside `[Replicate]`.** |
| `Reconcile(PredictionRigidbody)` | Restores Rigidbody state from a reconciliation snapshot |

### Initialization and Cleanup

`PredictionRigidbody` instances are retrieved from and returned to an object cache:

```csharp
public PredictionRigidbody PredictionRigidbody;

private void Awake()
{
    PredictionRigidbody = ObjectCaches<PredictionRigidbody>.Retrieve();
    PredictionRigidbody.Initialize(GetComponent<Rigidbody>());
}

private void OnDestroy()
{
    ObjectCaches<PredictionRigidbody>.StoreAndDefault(ref PredictionRigidbody);
}
```

### Outside Forces (Triggers, Collisions)

When external objects (bumpers, boost pads, etc.) apply forces to a predicted object, they must target the `PredictionRigidbody`, not the raw `Rigidbody`. FishNet provides `NetworkTrigger` and `NetworkCollision` components for prediction-aware Enter/Exit events.

```csharp
// External script applying a force to a predicted player
rbPlayer.PredictionRigidbody.AddForce(Vector3.up * 20f, ForceMode.Impulse);
// Do NOT call Simulate() here -- that only happens inside [Replicate]
```

---

## 5. C# Code Examples

### 5.1 Input Struct Definition

```csharp
public struct ReplicateData : IReplicateData
{
    public bool Jump;
    public bool Sprint;
    public float Horizontal;
    public float Vertical;

    public ReplicateData(bool jump, bool sprint, float horizontal, float vertical) : this()
    {
        Jump = jump;
        Sprint = sprint;
        Horizontal = horizontal;
        Vertical = vertical;
    }

    private uint _tick;
    public void Dispose() { }
    public uint GetTick() => _tick;
    public void SetTick(uint value) => _tick = value;
}
```

**Notes:**
- Must implement `IReplicateData`.
- The `_tick` field and its accessors are required by FishNet for internal tick tracking.
- Only include data that affects prediction (raw input values, not derived state).

### 5.2 State Struct Definition

```csharp
public struct ReconcileData : IReconcileData
{
    public PredictionRigidbody PredictionRigidbody;
    public float Stamina;

    public ReconcileData(PredictionRigidbody pr, float stamina) : this()
    {
        PredictionRigidbody = pr;
        Stamina = stamina;
    }

    private uint _tick;
    public void Dispose() { }
    public uint GetTick() => _tick;
    public void SetTick(uint value) => _tick = value;
}
```

**Notes:**
- Must implement `IReconcileData`.
- Must include **all mutable state** that affects prediction. If a value (like stamina) can change the outcome of replayed inputs, it must be reconciled. Otherwise, replayed inputs after a correction produce different results, causing visible desynchronization.
- When using multiple rigidbodies, each must have its own `PredictionRigidbody` included in the reconcile data.

### 5.3 Complete Predicted Player Controller

```csharp
using FishNet.Object;
using FishNet.Object.Prediction;
using FishNet.Transporting;
using UnityEngine;

public class PredictedPlayerController : NetworkBehaviour
{
    [SerializeField] private float _moveRate = 15f;
    [SerializeField] private float _jumpForce = 8f;

    public PredictionRigidbody PredictionRigidbody;
    private bool _jump;
    private float _stamina = 100f;

    // ── Structs ──────────────────────────────────────────────

    public struct ReplicateData : IReplicateData
    {
        public bool Jump;
        public bool Sprint;
        public float Horizontal;
        public float Vertical;

        public ReplicateData(bool jump, bool sprint, float horizontal, float vertical) : this()
        {
            Jump = jump;
            Sprint = sprint;
            Horizontal = horizontal;
            Vertical = vertical;
        }

        private uint _tick;
        public void Dispose() { }
        public uint GetTick() => _tick;
        public void SetTick(uint value) => _tick = value;
    }

    public struct ReconcileData : IReconcileData
    {
        public PredictionRigidbody PredictionRigidbody;
        public float Stamina;

        public ReconcileData(PredictionRigidbody pr, float stamina) : this()
        {
            PredictionRigidbody = pr;
            Stamina = stamina;
        }

        private uint _tick;
        public void Dispose() { }
        public uint GetTick() => _tick;
        public void SetTick(uint value) => _tick = value;
    }

    // ── Lifecycle ────────────────────────────────────────────

    private void Awake()
    {
        PredictionRigidbody = ObjectCaches<PredictionRigidbody>.Retrieve();
        PredictionRigidbody.Initialize(GetComponent<Rigidbody>());
    }

    private void OnDestroy()
    {
        ObjectCaches<PredictionRigidbody>.StoreAndDefault(ref PredictionRigidbody);
    }

    public override void OnStartNetwork()
    {
        base.TimeManager.OnTick += TimeManager_OnTick;
        base.TimeManager.OnPostTick += TimeManager_OnPostTick;
    }

    public override void OnStopNetwork()
    {
        base.TimeManager.OnTick -= TimeManager_OnTick;
        base.TimeManager.OnPostTick -= TimeManager_OnPostTick;
    }

    // ── Input Gathering (runs in Update, not tick) ───────────

    private void Update()
    {
        if (base.IsOwner)
        {
            if (Input.GetKeyDown(KeyCode.Space))
                _jump = true;
        }
    }

    // ── Tick Callbacks ───────────────────────────────────────

    private void TimeManager_OnTick()
    {
        RunInputs(CreateReplicateData());
    }

    private void TimeManager_OnPostTick()
    {
        CreateReconcile();
    }

    // ── Input Creation ───────────────────────────────────────

    private ReplicateData CreateReplicateData()
    {
        // Non-owners return default (no input).
        if (!base.IsOwner)
            return default;

        float horizontal = Input.GetAxisRaw("Horizontal");
        float vertical = Input.GetAxisRaw("Vertical");
        bool sprint = Input.GetKey(KeyCode.LeftShift);

        ReplicateData md = new ReplicateData(_jump, sprint, horizontal, vertical);
        _jump = false;

        return md;
    }

    // ── Replicate (runs on owner, server, and other clients) ─

    [Replicate]
    private void RunInputs(ReplicateData data, ReplicateState state = ReplicateState.Invalid,
        Channel channel = Channel.Unreliable)
    {
        float delta = (float)base.TimeManager.TickDelta;

        // Stamina regeneration
        _stamina += 3f * delta;

        // Movement forces
        Vector3 forces = new Vector3(data.Horizontal, 0f, data.Vertical) * _moveRate;

        // Sprint modifier (costs stamina)
        float sprintCost = 6f * delta;
        if (data.Sprint && _stamina >= sprintCost)
        {
            _stamina -= sprintCost;
            forces *= 1.3f;
        }

        PredictionRigidbody.AddForce(forces);

        // Jump
        if (data.Jump)
        {
            Vector3 jumpForce = new Vector3(0f, _jumpForce, 0f);
            PredictionRigidbody.AddForce(jumpForce, ForceMode.Impulse);
        }

        // Extra gravity
        PredictionRigidbody.AddForce(Physics.gravity * 3f);

        // Flush all pending forces into the Rigidbody
        PredictionRigidbody.Simulate();
    }

    // ── Reconcile (server sends authoritative state) ─────────

    public override void CreateReconcile()
    {
        ReconcileData rd = new ReconcileData(PredictionRigidbody, _stamina);
        ReconcileState(rd);
    }

    [Reconcile]
    private void ReconcileState(ReconcileData data, Channel channel = Channel.Unreliable)
    {
        PredictionRigidbody.Reconcile(data.PredictionRigidbody);
        _stamina = data.Stamina;
    }
}
```

### 5.4 Non-Controlled Object (Reactive Physics Body)

For physics objects that are not player-controlled (crates, balls, etc.) but still need to stay in sync with the prediction system:

```csharp
using FishNet.Object;
using FishNet.Object.Prediction;
using FishNet.Transporting;
using UnityEngine;

public class PredictedPhysicsObject : NetworkBehaviour
{
    public PredictionRigidbody PredictionRigidbody;

    public struct ReplicateData : IReplicateData
    {
        private uint _tick;
        public void Dispose() { }
        public uint GetTick() => _tick;
        public void SetTick(uint value) => _tick = value;
    }

    public struct ReconcileData : IReconcileData
    {
        public PredictionRigidbody PredictionRigidbody;

        public ReconcileData(PredictionRigidbody pr) : this()
        {
            PredictionRigidbody = pr;
        }

        private uint _tick;
        public void Dispose() { }
        public uint GetTick() => _tick;
        public void SetTick(uint value) => _tick = value;
    }

    private void Awake()
    {
        PredictionRigidbody = ObjectCaches<PredictionRigidbody>.Retrieve();
        PredictionRigidbody.Initialize(GetComponent<Rigidbody>());
    }

    private void OnDestroy()
    {
        ObjectCaches<PredictionRigidbody>.StoreAndDefault(ref PredictionRigidbody);
    }

    public override void OnStartNetwork()
    {
        base.TimeManager.OnPostTick += TimeManager_OnPostTick;
    }

    public override void OnStopNetwork()
    {
        base.TimeManager.OnPostTick -= TimeManager_OnPostTick;
    }

    private void TimeManager_OnPostTick()
    {
        // Run with default (empty) input -- physics drives movement
        RunInputs(default);
        CreateReconcile();
    }

    [Replicate]
    private void RunInputs(ReplicateData data, ReplicateState state = ReplicateState.Invalid,
        Channel channel = Channel.Unreliable)
    {
        // No logic needed -- physics simulation handles everything
    }

    public override void CreateReconcile()
    {
        ReconcileData rd = new ReconcileData(PredictionRigidbody);
        ReconcileState(rd);
    }

    [Reconcile]
    private void ReconcileState(ReconcileData data, Channel channel = Channel.Unreliable)
    {
        PredictionRigidbody.Reconcile(data.PredictionRigidbody);
    }
}
```

### 5.5 CharacterController Special Handling

When using Unity's `CharacterController` instead of a `Rigidbody`, disable the component before position changes and re-enable it afterward. The `CharacterController` maintains internal physics state that conflicts with direct transform manipulation during reconciliation:

```csharp
[Reconcile]
private void ReconcileState(ReconcileData data, Channel channel = Channel.Unreliable)
{
    CharacterController cc = GetComponent<CharacterController>();
    cc.enabled = false;
    transform.position = data.Position;
    transform.rotation = data.Rotation;
    cc.enabled = true;
}
```

---

## 6. Known Issues and Pitfalls

### Jitter During Reconciliation

When the server's authoritative state diverges from the client's predicted state, the client snaps to the corrected position and replays inputs. If corrections happen frequently (due to non-determinism in physics, floating-point differences, or external forces), this manifests as visible jitter. FishNet provides interpolation settings on both `PredictionManager` and `NetworkObject` to smooth corrections, but the two serve distinct purposes and must be configured separately.

### Force-Doubling

A common bug when applying forces directly to the `Rigidbody` instead of through `PredictionRigidbody`. During reconciliation, forces that were applied directly to the Rigidbody get baked into the reconcile snapshot's velocity, and then the replayed `[Replicate]` call applies the force again on top, effectively doubling it. The fix is to **always** use `PredictionRigidbody.AddForce()` and never `Rigidbody.AddForce()`.

### Unreconciled Mutable State

If a value stored outside the `[Replicate]` method can affect prediction outcomes (stamina, ammo, cooldowns), it **must** be included in the `ReconcileData` struct and restored in the `[Reconcile]` method. Otherwise, after a reconciliation the replayed inputs will produce different results because the stale value was not reset, causing progressive drift.

From the FishNet documentation:

> "If a value can affect your prediction do not store it outside the replicate method, unless you are also reconciling the value."

### Simulate() Called Outside Replicate

`PredictionRigidbody.Simulate()` must only be called inside the `[Replicate]` method. Calling it elsewhere (e.g., in a trigger callback that adds an outside force) will cause incorrect physics behavior during replay.

### Multiple Rigidbodies

If an object uses multiple rigidbodies (e.g., a vehicle with wheel colliders), each rigidbody needs its own `PredictionRigidbody` instance, and all of them must be included in the `ReconcileData` and restored during reconciliation.

---

## 7. Comparison to Mirror and NGO

| Feature | FishNet | Mirror | Unity NGO |
|---------|---------|--------|-----------|
| **Price** | Free (open-source) | Free (open-source) | Free (Unity package) |
| **Prediction approach** | `[Replicate]` / `[Reconcile]` attributes with automatic input replay | `PredictedRigidbody` component with triple-object technique | No built-in prediction; offers `AnticipatedNetworkVariable` / `AnticipatedNetworkTransform` |
| **Physics handling** | `PredictionRigidbody` wrapper; applies forces through a pending queue; full Simulate/Reconcile cycle | Does **not** call `Physics.Simulate()` for corrections; manually re-applies Rigidbody properties in C# outside PhysX | No physics prediction support |
| **Reconciliation** | Server sends authoritative state; client snaps and replays all unacknowledged inputs | Smoothing between predicted object and server ghost; no input replay | `StaleDataHandling.Reanticipate` fires `OnReanticipate` for manual replay logic |
| **State forwarding** | Built-in; other clients can run `[Replicate]` with forwarded data | Not built-in | Not applicable |
| **Tick system** | Built-in `TimeManager` with `OnTick`/`OnPostTick` | Server tick available but prediction is not tick-driven | `NetworkTickSystem` available but not integrated with prediction |
| **Scalability** | One `PredictionRigidbody` per predicted object | Designed to handle thousands of predicted objects by avoiding `Physics.Simulate()` | N/A |
| **Reconcile accuracy** | High -- uses actual physics simulation during replay via `PredictionRigidbody.Simulate()` | Lower -- manual C# resimulation outside PhysX sacrifices accuracy for performance | N/A |
| **Maturity** | Production-ready; actively maintained | Experimental prediction; core networking is mature | Anticipation API is relatively new |

### Key Architectural Differences

**FishNet vs. Mirror:** FishNet's approach is closer to the classic Valve/Source model -- it replays actual inputs through the same simulation code. Mirror takes a different path by maintaining three simultaneous representations of each object (predicted, rendered, remote) and smoothing between them without full input replay. Mirror's approach scales better with many predicted objects but is less accurate for complex physics interactions.

**FishNet vs. NGO:** Unity's Netcode for GameObjects does not attempt full client-side prediction. Its "Client Anticipation" system is a simpler model where clients can optimistically set values and receive corrections, but there is no automatic input buffering, replay, or physics resimulation. Developers must build rollback logic manually on top of the anticipation primitives.

---

## 8. References

### Official FishNet Documentation
- [FishNet Documentation Home](https://fish-networking.gitbook.io/docs)
- [What Is Client-Side Prediction](https://fish-networking.gitbook.io/docs/guides/features/prediction/what-is-client-side-prediction)
- [Prediction Overview](https://fish-networking.gitbook.io/docs/guides/features/prediction)
- [Controlling an Object](https://fish-networking.gitbook.io/docs/guides/features/prediction/creating-code/controlling-an-object)
- [Non-Controlled Object](https://fish-networking.gitbook.io/docs/guides/features/prediction/creating-code/non-controlled-object)
- [Understanding ReplicateState](https://fish-networking.gitbook.io/docs/guides/features/prediction/creating-code/understanding-replicatestate)
- [Advanced Controls](https://fish-networking.gitbook.io/docs/guides/features/prediction/creating-code/advanced-controls)
- [PredictionRigidbody](https://fish-networking.gitbook.io/docs/guides/features/prediction/predictionrigidbody)
- [Creating Code (Overview)](https://fish-networking.gitbook.io/docs/guides/features/prediction/creating-code)

### FishNet Source and Community
- [FishNet GitHub Repository](https://github.com/FirstGearGames/FishNet)
- [FishNet Unity Asset Store](https://assetstore.unity.com/packages/tools/network/fish-net-networking-evolved-207815)
- [FirstGearGames Discord](https://discord.gg/Ta9HgDh4Hj)

### Comparison References
- [Mirror Client-Side Prediction Docs](https://mirror-networking.gitbook.io/docs/manual/general/client-side-prediction)
- [Unity NGO Client Anticipation](https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.7/manual/advanced-topics/client-anticipation.html)
- [Gabriel Gambetta -- Client-Side Prediction and Server Reconciliation](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
- [Valve Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
