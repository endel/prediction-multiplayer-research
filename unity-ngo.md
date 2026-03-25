# Unity Netcode for GameObjects (NGO) -- Client Anticipation System

## Table of Contents

1. [Overview of Unity NGO's Networking Model](#overview-of-unity-ngos-networking-model)
2. [Why NGO Does NOT Provide Full Rollback-and-Replay](#why-ngo-does-not-provide-full-rollback-and-replay)
3. [Client Anticipation System in Detail](#client-anticipation-system-in-detail)
   - [AnticipatedNetworkVariable\<T\>](#anticipatednetworkvariablet)
   - [AnticipatedNetworkTransform](#anticipatednetworktransform)
   - [StaleDataHandling Modes](#staledatahandling-modes)
   - [Smooth() Method and Interpolation](#smooth-method-and-interpolation)
4. [C# Code Examples](#c-code-examples)
5. [Comparison with Photon Fusion and FishNet](#comparison-with-photon-fusion-and-fishnet)
6. [Limitations and Workarounds](#limitations-and-workarounds)
7. [References](#references)

---

## Overview of Unity NGO's Networking Model

Unity Netcode for GameObjects (NGO) is Unity's first-party networking framework for multiplayer games. It operates on a **server-authoritative** model by default:

- **Server authority**: The server (or host, in host mode) makes all final gameplay decisions. Clients send inputs or requests via RPCs, and the server processes them and replicates the results back through `NetworkVariable` synchronization.
- **NetworkVariable replication**: State is synchronized from server to clients through `NetworkVariable<T>`, which only allows the server (or the authority, depending on write permissions) to write values. Clients receive updates automatically.
- **NetworkTransform**: Built-in component that synchronizes an object's transform (position, rotation, scale) from the authority to all other peers.
- **RPCs (Remote Procedure Calls)**: Clients communicate intent to the server via `ServerRpc`, and the server can push events to clients via `ClientRpc`.
- **Tick-based**: NGO uses a tick-based system with configurable tick rates. The `NetworkTickSystem` provides timing information, and `NetworkTime` exposes both server time and local time.

The fundamental flow is:

```
Client Input --> ServerRpc --> Server processes --> NetworkVariable updates --> Clients receive
```

This creates a round-trip delay between a player's action and seeing the result on screen, which is the core problem that the Client Anticipation system addresses.

---

## Why NGO Does NOT Provide Full Rollback-and-Replay

Unity NGO's documentation is explicit: **NGO does not support full client-side prediction and reconciliation.** This is a deliberate design decision, not an oversight. The reasons include:

### Complexity for game developers

Full rollback-and-replay (as seen in Photon Fusion or custom implementations) requires:

- Deterministic simulation or per-tick state snapshots
- Input buffers with tick-stamped commands
- The ability to re-simulate N ticks of game logic when a correction arrives
- Careful management of physics, animation, and all side effects during replay

This is extremely difficult to implement correctly in a general-purpose framework, especially one built on Unity's non-deterministic physics engine.

### Not all games need it

Many game genres -- turn-based games, RPGs, cooperative PvE, strategy games -- do not require frame-precise prediction. A simpler "anticipate and correct" model covers a large percentage of use cases without imposing the complexity overhead.

### NGO's design philosophy

NGO targets a broad audience of Unity developers, many of whom are building their first multiplayer game. Rather than providing a complex prediction system that is easy to misuse, NGO provides **Client Anticipation** -- a simpler primitive that:

- Separates visual (anticipated) state from authoritative state
- Lets developers anticipate the result of an action immediately
- Provides hooks (`OnReanticipate`) where advanced users can layer full prediction on top
- Handles stale data gracefully via configurable policies

As the documentation states:

> "If you want to implement a full client-side prediction model in your game, the global `OnReanticipate` callback is likely the ideal place to incorporate your rollback and replay logic. The details of implementing this, however, are left up to users. Implementing a full, production-ready prediction loop is a complex topic and recommended for advanced users only."

---

## Client Anticipation System in Detail

Client Anticipation is NGO's answer to latency hiding. Rather than predicting and replaying, it lets the client **anticipate** the outcome of an action, show that outcome immediately, and then reconcile when the server responds.

### Core Concept: Two Values

Every anticipated variable or transform maintains two separate values:

| Value | Description |
|---|---|
| **Anticipated (Visual) Value** | What the client displays. Can be set locally via `Anticipate()` methods. |
| **Authoritative Value** | The ground-truth value from the server. Updated when server data arrives. |

When no anticipation is active, both values are identical. When a client anticipates, the visual value diverges from the authoritative value until the server catches up.

---

### AnticipatedNetworkVariable\<T\>

`AnticipatedNetworkVariable<T>` is a drop-in replacement for `NetworkVariable<T>` that supports client anticipation. It is a generic, serializable class that extends `NetworkVariableBase`.

**Key API surface:**

| Member | Description |
|---|---|
| `Value` | The current display/anticipated value. Affected by `Anticipate()`, `Smooth()`, and server updates. |
| `AuthoritativeValue` | The most recent authoritative value from the server. Read-only on clients; read-write on server. |
| `PreviousAnticipatedValue` | The value most recently passed to `Anticipate()`. Not overwritten by server updates. |
| `ShouldReanticipate` | `true` if the anticipated value was overwritten by a server update and needs reanticipation. |
| `Anticipate(T newValue)` | Sets the anticipated value on the client. On the server, sets both anticipated and authoritative. |
| `Smooth(in T from, in T to, float duration, SmoothDelegate)` | Interpolates the visual value from `from` to `to` over `duration` seconds using a provided lerp function. |
| `StaleDataHandling` | Field controlling how stale server updates are handled (`Ignore` or `Reanticipate`). |
| `OnAuthoritativeValueChanged` | Callback invoked whenever the authoritative value changes, regardless of stale data handling. |

**Constructor:**

```csharp
public AnticipatedNetworkVariable(T value = default, StaleDataHandling staleDataHandling = StaleDataHandling.Ignore)
```

**Three operating modes** (which can be combined):

1. **Snap**: `StaleDataHandling.Ignore`, no `OnReanticipate` callback. Server value immediately replaces the anticipated value.
2. **Smooth**: `StaleDataHandling.Ignore`, with an `OnReanticipate` callback that calls `Smooth()`. Interpolates from incorrect anticipated value to the correct authoritative value.
3. **Constant Reanticipation**: `StaleDataHandling.Reanticipate`, with an `OnReanticipate` callback that recalculates a new anticipated value from the current authoritative state.

---

### AnticipatedNetworkTransform

`AnticipatedNetworkTransform` is a subclass of `NetworkTransform` that adds anticipation support for position, rotation, and scale.

**Key API surface:**

| Member | Description |
|---|---|
| `AnticipatedState` | The current anticipated transform state (position, rotation, scale). Also reflected in `gameObject.transform`. |
| `AuthoritativeState` | The most recent authoritative transform state from the server. |
| `PreviousAnticipatedState` | The state most recently set via any `Anticipate` method. |
| `ShouldReanticipate` | `true` if the state was overwritten by a server update. |
| `AnticipateMove(Vector3 position)` | Anticipate a new position. |
| `AnticipateRotate(Quaternion rotation)` | Anticipate a new rotation. |
| `AnticipateScale(Vector3 scale)` | Anticipate a new scale. |
| `AnticipateState(TransformState state)` | Anticipate position, rotation, and scale all at once. |
| `Smooth(TransformState from, TransformState to, float duration)` | Interpolate the visual transform state over time. |
| `StaleDataHandling` | Same as `AnticipatedNetworkVariable` -- controls stale data policy. |

**Important**: On the client, directly modifying `gameObject.transform` values will be overwritten on the next server update. You **must** use the `Anticipate*` methods for changes to persist.

On the server, calling any `Anticipate*` method updates both the anticipated and authoritative states simultaneously.

---

### StaleDataHandling Modes

The stale data problem arises when the server sends an update that reflects a world state from *before* the client's most recent anticipation. NGO tracks this using an internal `AnticipationCounter` that increments with each network batch.

#### StaleDataHandling.Ignore (default)

- Stale server updates do **not** overwrite the anticipated value.
- `ShouldReanticipate` remains `false`.
- The `OnReanticipate` callback is **not** triggered for stale data.
- `AuthoritativeValue` is still updated (so you can inspect it).
- `OnAuthoritativeValueChanged` is still fired.

**Best for**: Discrete, player-initiated actions (e.g., changing a color, toggling a switch) where the client is confident its anticipation will be confirmed.

**Example scenario**: Client A changes color to blue, Client B changes color to red. Client B anticipated red, and the stale "blue" update from Client A's earlier action is ignored -- no flicker.

#### StaleDataHandling.Reanticipate

- Stale server updates **do** overwrite the anticipated value (rolling it back to the server state).
- `ShouldReanticipate` is set to `true`.
- The `OnReanticipate` callback fires, giving the developer a chance to recalculate the anticipated value.
- The callback receives a `double lastRoundTripTime` parameter (in seconds) representing how far back the rollback goes.

**Best for**: Continuously changing values (e.g., player position, velocity) where the client needs to constantly re-predict based on fresh server state.

---

### Smooth() Method and Interpolation

When the server sends a correction that differs from the client's anticipation, a raw snap to the new value can be visually jarring. The `Smooth()` method provides interpolation to soften corrections.

**For `AnticipatedNetworkVariable<T>`:**

```csharp
void Smooth(in T from, in T to, float duration, SmoothDelegate smoothDelegate)
```

Where `SmoothDelegate` is:

```csharp
delegate T SmoothDelegate(T from, T to, float percent);
```

For simple floats, this is often just `Mathf.Lerp`. For vectors, `Vector3.Lerp`. For quaternions, `Quaternion.Slerp`.

**For `AnticipatedNetworkTransform`:**

```csharp
void Smooth(TransformState from, TransformState to, float duration)
```

The transform version handles position/rotation/scale interpolation internally.

**Server-side smoothing** is also supported. In host mode, remote client movement can appear choppy due to jitter. Calling `Smooth()` on the server after receiving client input smooths the visual motion. An important distinction: on the server, smoothing only affects `AnticipatedState` (the visual), not `transform` directly, so game logic and physics use the actual position.

---

## C# Code Examples

### Setting Up an Anticipated Variable

```csharp
using Unity.Netcode;
using UnityEngine;

public class ColorChanger : NetworkBehaviour
{
    // Anticipated variable with default StaleDataHandling.Ignore
    private AnticipatedNetworkVariable<Color> _color =
        new AnticipatedNetworkVariable<Color>(Color.white);

    private Renderer _renderer;

    private void Awake()
    {
        _renderer = GetComponent<Renderer>();
    }

    public override void OnNetworkSpawn()
    {
        // React to any value change (anticipated or authoritative)
        _color.OnAuthoritativeValueChanged += OnColorAuthoritativeChanged;
    }

    public override void OnNetworkDespawn()
    {
        _color.OnAuthoritativeValueChanged -= OnColorAuthoritativeChanged;
    }

    private void OnColorAuthoritativeChanged(
        AnticipatedNetworkVariable<Color> variable,
        in Color previousValue,
        in Color newValue)
    {
        // Called whenever the authoritative value changes, even for stale data.
        // You can use this for logging, validation, etc.
    }

    private void Update()
    {
        // Always use the anticipated Value for rendering
        _renderer.material.color = _color.Value;
    }

    // Called by UI button
    public void SetColorBlue()
    {
        // Anticipate the result immediately on the client
        _color.Anticipate(Color.blue);

        // Send the request to the server
        SetColorServerRpc(Color.blue);
    }

    [ServerRpc]
    private void SetColorServerRpc(Color newColor)
    {
        // Server validates and applies the change
        _color.AuthoritativeValue = newColor;
    }
}
```

### Using AnticipateMove / AnticipateState

```csharp
using Unity.Netcode;
using Unity.Netcode.Components;
using UnityEngine;

[RequireComponent(typeof(AnticipatedNetworkTransform))]
public class PlayerMovement : NetworkBehaviour
{
    [SerializeField] private float _moveSpeed = 5f;

    private AnticipatedNetworkTransform _anticipatedTransform;

    private void Awake()
    {
        _anticipatedTransform = GetComponent<AnticipatedNetworkTransform>();
        // Use Reanticipate for continuous movement -- we want to re-predict
        // whenever a server correction arrives
        _anticipatedTransform.StaleDataHandling = StaleDataHandling.Reanticipate;
    }

    private void Update()
    {
        if (!IsOwner) return;

        Vector3 input = new Vector3(
            Input.GetAxis("Horizontal"),
            0,
            Input.GetAxis("Vertical")
        );

        if (input.sqrMagnitude > 0.01f)
        {
            Vector3 movement = input.normalized * _moveSpeed * Time.deltaTime;
            Vector3 newPosition = transform.position + movement;

            // Anticipate the new position locally
            _anticipatedTransform.AnticipateMove(newPosition);

            // Send input to server for authoritative processing
            MoveServerRpc(input.normalized);
        }
    }

    [ServerRpc]
    private void MoveServerRpc(Vector3 inputDirection)
    {
        // Server applies movement authoritatively
        Vector3 movement = inputDirection * _moveSpeed * NetworkManager.ServerTime.FixedDeltaTime;
        Vector3 newPosition = transform.position + movement;

        // On the server, AnticipateMove updates both anticipated and authoritative
        _anticipatedTransform.AnticipateMove(newPosition);
    }
}
```

**Using `AnticipateState` for combined updates:**

```csharp
// Anticipate position, rotation, and scale simultaneously
var newState = new TransformState
{
    Position = newPosition,
    Rotation = Quaternion.LookRotation(moveDirection),
    Scale = Vector3.one
};
_anticipatedTransform.AnticipateState(newState);
```

### Handling the OnReanticipate Callback

```csharp
using Unity.Netcode;
using Unity.Netcode.Components;
using UnityEngine;

public class PredictedCharacter : NetworkBehaviour
{
    private AnticipatedNetworkVariable<float> _health =
        new AnticipatedNetworkVariable<float>(100f, StaleDataHandling.Reanticipate);

    private AnticipatedNetworkTransform _anticipatedTransform;

    private void Awake()
    {
        _anticipatedTransform = GetComponent<AnticipatedNetworkTransform>();
        _anticipatedTransform.StaleDataHandling = StaleDataHandling.Reanticipate;
    }

    /// <summary>
    /// Called when server data arrives and variables/transforms are rolled back.
    /// The parameter is the round-trip time in seconds, representing how far
    /// back the rollback goes.
    /// </summary>
    public override void OnReanticipate(double lastRoundTripTime)
    {
        // Check each anticipated variable/transform individually
        if (_anticipatedTransform.ShouldReanticipate)
        {
            // Store old anticipated state before reanticipation
            var previousState = _anticipatedTransform.PreviousAnticipatedState;

            // Re-predict: extrapolate from the server-corrected position
            // using the round-trip time to estimate where we should be now
            Vector3 serverPosition = _anticipatedTransform.AuthoritativeState.Position;
            Vector3 velocity = (previousState.Position - serverPosition) /
                               (float)lastRoundTripTime;
            Vector3 reanticipatedPosition = serverPosition + velocity *
                                            (float)lastRoundTripTime;

            _anticipatedTransform.AnticipateMove(reanticipatedPosition);

            // Smooth from the old anticipated position to the new one
            // over 100ms to avoid a visual pop
            _anticipatedTransform.Smooth(
                previousState,
                _anticipatedTransform.AnticipatedState,
                0.1f
            );
        }

        if (_health.ShouldReanticipate)
        {
            // For health, just smooth to the authoritative value
            float previousHealth = _health.PreviousAnticipatedValue;
            float authoritativeHealth = _health.AuthoritativeValue;

            _health.Smooth(
                previousHealth,
                authoritativeHealth,
                0.2f,
                Mathf.Lerp   // SmoothDelegate for float interpolation
            );
        }
    }
}
```

### Building Full Prediction on Top of Anticipation Primitives

NGO's anticipation system provides the **building blocks** for a full prediction loop. Advanced users can implement rollback-and-replay using the global `OnReanticipate` callback on `NetworkManager`.

```csharp
using Unity.Netcode;
using Unity.Netcode.Components;
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// Example: building a basic prediction loop on top of NGO's anticipation system.
/// This stores an input history and replays inputs during reanticipation.
/// </summary>
public class PredictionManager : MonoBehaviour
{
    // Circular buffer of past inputs, indexed by tick
    private struct InputFrame
    {
        public int Tick;
        public Vector3 MoveDirection;
        public bool Jump;
    }

    private const int InputBufferSize = 128;
    private InputFrame[] _inputBuffer = new InputFrame[InputBufferSize];
    private NetworkManager _networkManager;

    private void Start()
    {
        _networkManager = NetworkManager.Singleton;

        // Subscribe to the global OnReanticipate callback.
        // This fires AFTER all per-NetworkBehaviour OnReanticipate calls,
        // making it ideal for coordinated, step-wise replay.
        _networkManager.OnReanticipate += OnGlobalReanticipate;
    }

    private void OnDestroy()
    {
        if (_networkManager != null)
        {
            _networkManager.OnReanticipate -= OnGlobalReanticipate;
        }
    }

    /// <summary>
    /// Called each frame by your input system. Stores input and sends to server.
    /// </summary>
    public void RecordInput(int tick, Vector3 moveDir, bool jump)
    {
        int index = tick % InputBufferSize;
        _inputBuffer[index] = new InputFrame
        {
            Tick = tick,
            MoveDirection = moveDir,
            Jump = jump
        };
    }

    /// <summary>
    /// Global reanticipation: replay all stored inputs from the server's
    /// corrected state forward to the current tick.
    /// </summary>
    private void OnGlobalReanticipate(double lastRoundTripTime)
    {
        // Calculate how many ticks we need to replay
        int tickRate = (int)(1.0 / _networkManager.ServerTime.FixedDeltaTime);
        int ticksToReplay = Mathf.CeilToInt((float)(lastRoundTripTime * tickRate));
        int currentTick = _networkManager.ServerTime.Tick;
        int replayStartTick = currentTick - ticksToReplay;

        // Replay each tick's input from the corrected server state
        for (int tick = replayStartTick; tick <= currentTick; tick++)
        {
            int index = tick % InputBufferSize;
            if (_inputBuffer[index].Tick == tick)
            {
                // Apply this tick's input to all predicted objects
                SimulateTick(_inputBuffer[index]);
            }
        }
    }

    private void SimulateTick(InputFrame input)
    {
        // Apply movement logic to all predicted objects.
        // Each object's AnticipatedNetworkTransform has already been
        // rolled back to the server state by NGO before this callback.
        //
        // Your game-specific simulation logic goes here:
        // - Apply input as movement
        // - Run simplified physics
        // - Update anticipated variables
        //
        // Example for a single player object:
        var player = FindPlayerPredictedObject();
        if (player != null && player.ShouldReanticipate)
        {
            float dt = (float)_networkManager.ServerTime.FixedDeltaTime;
            Vector3 pos = player.AnticipatedState.Position;
            pos += input.MoveDirection * 5f * dt;
            player.AnticipateMove(pos);
        }
    }

    private AnticipatedNetworkTransform FindPlayerPredictedObject()
    {
        // In practice, cache this reference
        var localClient = _networkManager.LocalClient;
        if (localClient?.PlayerObject != null)
        {
            return localClient.PlayerObject.GetComponent<AnticipatedNetworkTransform>();
        }
        return null;
    }
}
```

### Smoothing Corrections

```csharp
using Unity.Netcode;
using UnityEngine;

public class SmoothedPosition : NetworkBehaviour
{
    private AnticipatedNetworkVariable<Vector3> _position =
        new AnticipatedNetworkVariable<Vector3>(Vector3.zero, StaleDataHandling.Ignore);

    public override void OnReanticipate(double lastRoundTripTime)
    {
        if (_position.ShouldReanticipate)
        {
            // Grab previous value BEFORE any new anticipation
            Vector3 previousValue = _position.PreviousAnticipatedValue;
            Vector3 serverValue = _position.AuthoritativeValue;

            // Check if the error is small enough to smooth (vs. snap for large errors)
            float error = Vector3.Distance(previousValue, serverValue);

            if (error < 5f)
            {
                // Small correction: smooth over 150ms
                _position.Smooth(
                    previousValue,      // from: where we thought we were
                    serverValue,        // to: where the server says we are
                    0.15f,              // duration in seconds
                    Vector3.Lerp        // lerp function
                );
            }
            // else: large error, just snap (don't call Smooth)
        }
    }
}
```

**Server-side smoothing for host mode (smoothing remote player movement on the host):**

```csharp
[ServerRpc]
private void SubmitPositionServerRpc(Vector3 newPosition)
{
    var ant = GetComponent<AnticipatedNetworkTransform>();
    var previousState = ant.AnticipatedState;

    // Apply the authoritative position
    ant.AnticipateMove(newPosition);

    // Smooth visual motion for the host player viewing this remote client
    ant.Smooth(previousState, ant.AnticipatedState, 0.05f);
}
```

---

## Comparison with Photon Fusion and FishNet

| Feature | Unity NGO (Anticipation) | Photon Fusion | FishNet (PredictionV2) |
|---|---|---|---|
| **Architecture** | Server-authoritative | Server-authoritative (Shared or Host/Server mode) | Server-authoritative |
| **Prediction model** | Client Anticipation (no built-in rollback/replay) | Full tick-based rollback-and-replay (built-in) | Full tick-based rollback-and-replay (built-in) |
| **State management** | `AnticipatedNetworkVariable<T>`, `AnticipatedNetworkTransform` | `[Networked]` properties with automatic snapshotting per tick | `ReplicateState` with automatic snapshots |
| **Input handling** | Manual (RPCs) | `NetworkInput` struct with automatic buffering and replay | `ReplicateInput` with automatic buffering |
| **Rollback** | Manual via `OnReanticipate` callback; framework rolls back values, user replays | Automatic: framework restores state to server tick, re-simulates all ticks to present | Automatic: framework handles rollback and re-simulation |
| **Physics prediction** | Not supported (user must implement) | Built-in physics rollback via `NetworkRigidbody` with `PhysicsScene.Simulate()` | Built-in physics rollback via `PredictedObject` |
| **Stale data handling** | `StaleDataHandling.Ignore` / `Reanticipate` enum | Automatic via tick comparison (built into snapshot system) | Automatic via tick-based reconciliation |
| **Smoothing** | `Smooth()` method with custom lerp delegates | Built-in interpolation with configurable error thresholds | `TransformSmoothing` / configurable correction speeds |
| **Complexity** | Low to moderate; developers choose how much prediction to implement | High (powerful but complex API surface) | Moderate to high |
| **Determinism requirement** | None | None (uses snapshot comparison, not deterministic lockstep) | None (snapshot-based) |
| **Ease of getting started** | Simple -- just swap `NetworkVariable` for `AnticipatedNetworkVariable` | Steeper learning curve; must understand tick lifecycle | Moderate; attribute-driven API |
| **Flexibility** | High -- you build what you need | Framework-driven; must conform to Fusion's patterns | Framework-driven with some customization |

### Key Takeaways

- **Photon Fusion** provides the most complete, production-ready prediction system. It automatically snapshots state per tick, buffers inputs, and replays when corrections arrive. The tradeoff is a more opinionated, complex framework.
- **FishNet** provides a similar automatic rollback system through its PredictionV2 API with `[Replicate]` and `[Reconcile]` attributes. It handles the replay loop internally.
- **Unity NGO** deliberately offers a simpler primitive. It gives you the tools to separate visual from authoritative state, handle stale data, and smooth corrections -- but it does not automate the replay loop. This makes it easier to understand but requires more work for competitive, latency-sensitive games that need frame-precise prediction.

---

## Limitations and Workarounds

### 1. No built-in input replay

**Limitation**: NGO does not buffer inputs or automatically replay them during reanticipation. When `OnReanticipate` fires, you know the round-trip time but must implement your own input history and replay logic.

**Workaround**: Maintain a circular buffer of inputs indexed by tick (as shown in the prediction example above). During `OnReanticipate`, replay stored inputs from the server-corrected state forward.

### 2. No physics rollback

**Limitation**: Unity's `PhysicsScene.Simulate()` is not integrated with NGO's anticipation system. You cannot roll back the physics world to re-simulate collisions.

**Workaround**: For simple cases, use your own collision logic (raycasts, overlap checks) during replay rather than relying on Unity's physics engine. For complex physics-driven games, consider Photon Fusion or a custom solution.

### 3. Single-variable granularity

**Limitation**: `OnReanticipate` fires per-NetworkBehaviour (all behaviours on the object are called), but you must check `ShouldReanticipate` on each variable individually. There is no automatic "replay all variables atomically" mechanism.

**Workaround**: Use the global `NetworkManager.OnReanticipate` callback for coordinated multi-object replay, ensuring all objects complete each step before proceeding to the next.

### 4. Host mode visual artifacts

**Limitation**: In host mode, the host player sees remote client movement with jitter because input updates don't arrive every frame.

**Workaround**: Use server-side `Smooth()` when receiving client inputs to interpolate between positions. Note that server-side smoothing only affects `AnticipatedState`, not the actual `transform`, so physics and game logic remain accurate.

### 5. No automatic reconciliation

**Limitation**: Unlike Fusion/FishNet, there is no built-in mechanism to detect prediction errors and automatically correct them. The developer must implement error detection and correction logic.

**Workaround**: In `OnReanticipate`, compare `PreviousAnticipatedValue` / `PreviousAnticipatedState` with the new authoritative value. Apply `Smooth()` for small corrections and snap for large ones (teleportation threshold).

### 6. Anticipated variables are not composable with standard NetworkVariable features

**Limitation**: `AnticipatedNetworkVariable<T>` replaces `NetworkVariable<T>` and does not support all the same features (e.g., custom serialization may require additional work).

**Workaround**: Use `AnticipatedNetworkVariable<T>` only for values that need anticipation. Keep standard `NetworkVariable<T>` for values that don't need client-side prediction.

### 7. No interpolation for other clients' objects

**Limitation**: When another client's object moves, the local client has no anticipation for it -- the position just arrives from the server with latency.

**Workaround**: Use `Smooth()` in the `OnReanticipate` callback even for non-local objects, or use standard `NetworkTransform` interpolation settings (which NGO provides separately from the anticipation system).

### 8. AnticipationCounter limitations

**Limitation**: The stale data detection relies on an `AnticipationCounter` that assumes anticipation and the corresponding RPC are sent in the same frame. If your RPC is delayed or batched differently, stale data detection may not work correctly.

**Workaround**: Ensure that `Anticipate()` calls and the corresponding `ServerRpc` calls happen in the same frame/tick.

---

## References

1. **Client Anticipation (NGO 2.7 Manual)**
   https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.7/manual/advanced-topics/client-anticipation.html

2. **Dealing with Latency (NGO 2.7 Manual)**
   https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.7/manual/learn/dealing-with-latency.html

3. **AnticipatedNetworkVariable\<T\> API Reference**
   https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.7/api/Unity.Netcode.AnticipatedNetworkVariable-1.html

4. **AnticipatedNetworkTransform API Reference**
   https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.7/api/Unity.Netcode.Components.AnticipatedNetworkTransform.html

5. **Latency and Packet Loss (NGO 2.7 Manual)**
   https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.7/manual/learn/lagandpacketloss.html

6. **NetworkVariable (NGO 2.7 Manual)**
   https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.7/manual/networkvariable.html

7. **NetworkTransform (NGO 2.7 Manual)**
   https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.7/manual/components/networktransform.html

8. **Photon Fusion -- Prediction and Rollback**
   https://doc.photonengine.com/fusion/current/manual/state/prediction

9. **FishNet -- Prediction (PredictionV2)**
   https://fish-networking.gitbook.io/docs/manual/guides/prediction
