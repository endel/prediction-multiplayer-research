# Coherence: Client-Side Prediction and Rollback

## 1. Architecture Overview

Coherence is a Unity-focused networking SDK built around a central **Replication Server** that relays entity state between connected clients. The architecture separates concerns into three key components:

- **Replication Server**: A relay server that maintains the authoritative world state. It does not run game logic itself. It tracks entity ownership, replicates state changes between clients, and maintains a synchronised simulation frame clock at 60Hz (~16ms per tick).

- **Game Clients**: Unity instances that connect to the Replication Server. Each client has a `CoherenceBridge` component that manages the connection, synchronisation, and entity replication. Clients own the entities they spawn and can simulate them locally.

- **Simulators**: Headless Unity builds that run on the server side. They connect to the Replication Server just like clients but are granted State Authority over critical entities. Simulators handle authoritative game logic (AI, physics, scoring, player state). Two types exist:
  - **Room Simulators**: Spun up per-room/match, handling isolated sessions.
  - **World Simulators**: Persistent processes for open-world scenarios.

Authority in Coherence is per-entity (per `CoherenceSync` component), not per-match. Different authority models can be mixed within a single session, and authority can transfer between Clients and Simulators at any time.

---

## 2. Authority System: State Authority vs Input Authority

Coherence splits authority into two orthogonal types, each independently assignable per entity:

### State Authority

Grants the right to **modify** an entity's networked properties. Only the State Authority holder can change synced values; if a non-authoritative client attempts to modify a property, coherence immediately resets it. There is exactly one State Authority holder per entity at any time.

### Input Authority

Grants the right to **send inputs** to the entity's State Authority holder. The Input Authority holder does not directly mutate state; it sends structured inputs that the State Authority processes. When a client spawns an entity, it automatically becomes both State and Input Authority.

### Authority Combinations

| Configuration | State Authority | Input Authority | Use Case |
|---|---|---|---|
| Full Authority | Client | Client | Single-player-like, simple multiplayer |
| Distributed Client | Client (owner) | Client (owner) | Peer-to-peer style |
| Server-side | Simulator | None needed | Server-controlled entities (AI, environment) |
| Server-side with Client Input | Simulator | Client | Player characters with server validation |

### Authority Transfer

Authority can transfer at any time between Clients, or between Clients and Simulators. The API uses:

```csharp
coherenceSync.TransferAuthority(newOwner.ClientId, AuthorityType.Input);
coherenceSync.TransferAuthority(newOwner.ClientId, AuthorityType.State);
```

Events `OnInputAuthority` and `OnInputRemote` notify when Input Authority changes. `OnInputSimulatorConnected` fires when a Simulator or Host gains Input Authority.

### Orphaned Entities

Entities can be abandoned via `CoherenceSync.AbandonAuthority()`, leaving them with neither State nor Input Authority. Other clients can adopt them via `CoherenceSync.Adopt()` or through "Auto-Adopt Orphan" settings.

---

## 3. Model A: Server-Authoritative Prediction (State-Based)

This model uses "Server Side With Client Input" simulation. The Simulator holds State Authority and runs the authoritative simulation. The client holds Input Authority and can locally predict state to mask round-trip latency.

### How It Works

1. The client reads local input and sends it to the Simulator via `CoherenceInput`.
2. The client simultaneously applies the input locally (prediction).
3. The Simulator processes the input authoritatively and replicates the resulting state.
4. When the client receives the authoritative state, it compares it against its prediction and reconciles.

### Setting Up CoherenceInput

On the CoherenceSync inspector:
- Set simulation type to **"Server Side With Client Input"**
- Define input fields on the `CoherenceInput` component (Button, Axis, Axis2D, Axis3D, String, Rotation, Integer)
- Bake the netcode after any input field changes

Input types available:
| Type | Set Method | Get Method |
|---|---|---|
| Button | `SetButton(name, bool)` | `GetButton(name)` |
| Axis | `SetAxis(name, float)` | `GetAxis(name)` |
| Axis2D | `SetAxis2D(name, Vector2)` | `GetAxis2D(name)` |
| Axis3D | `SetAxis3D(name, Vector3)` | `GetAxis3D(name)` |
| Rotation | `SetRotation(name, Quaternion)` | `GetRotation(name)` |
| Integer | `SetInteger(name, int)` | `GetInteger(name)` |
| String | `SetString(name, string)` | `GetString(name)` |

#### Client-Side: Sending Inputs

```csharp
void SendInputs()
{
    var jump = Input.GetButton("Jump");
    coherenceSync.Input.SetButton("Jump", jump);

    var moveX = Input.GetAxis("Horizontal");
    var moveY = Input.GetAxis("Vertical");
    var move = new Vector2(moveX, moveY);
    coherenceSync.Input.SetAxis2D("Move", move);
}
```

#### Server-Side (Simulator): Processing Inputs

```csharp
void ProcessInputs()
{
    var jump = coherenceSync.Input.GetButton("Jump");
    var move = coherenceSync.Input.GetAxis2D("Move");
    // Apply movement and jump logic authoritatively
}
```

### Prediction-Enabled Bindings

When prediction is enabled on a binding, **incoming network data for that property is ignored** on the owning client. This allows the client to calculate values locally, eliminating the perception of round-trip input lag.

Configuration:
- Open CoherenceSync configuration and enable prediction on specific bindings.
- Set auto-sync to **"Always Client Predict"** for predicted properties.

### OnNetworkSampleReceived Callback

When new authoritative state arrives from the Simulator, the `OnNetworkSampleReceived` callback fires. Developers implement misprediction detection here by comparing the received server state against the local predicted state:

```csharp
private void DetectMisprediction(object sampleData, bool stopped, long simulationFrame)
{
    const float MispredictionThreshold = 3;
    var networkPosition = (Vector3)sampleData;
    var distance = (networkPosition - transform.position).magnitude;

    if (distance > MispredictionThreshold)
    {
        // Correct the misprediction
        transform.position = networkPosition;
    }
}
```

The `simulationFrame` parameter provides the exact frame the sample corresponds to. For best accuracy, incoming network samples should be compared to the predicted state at that specific simulation frame, which requires maintaining a history buffer of predicted states.

### Reconciliation Strategies

Two primary approaches for handling mispredictions:

**Snap Reconciliation**: Immediately teleport the client state to match the server state when a misprediction is detected. Simple and correct, but visually jarring.

```csharp
// Snap: jump directly to server position
if (distance > threshold)
{
    transform.position = networkPosition;
}
```

**Blend Reconciliation**: Continuously interpolate from the current client state toward the server state over time. Smoother visually, but temporarily inaccurate during the blend.

```csharp
// Blend: lerp toward server position
transform.position = Vector3.Lerp(transform.position, networkPosition, blendFactor * Time.deltaTime);
```

### LiveQuery Integration

When a `CoherenceLiveQuery` is on a "Server Side With Client Inputs" entity, query visibility is applied based on the **Input Authority** owner, while the component's state remains under the State Authority's control. This prevents clients from exploiting visibility by manipulating query extents.

---

## 4. Model B: GGPO-Style Deterministic Rollback (Input-Based)

Coherence provides a built-in GGPO (Good Game Peace Out) implementation for fighting games and other genres that demand deterministic lockstep with rollback. This system exchanges **inputs** rather than state, and every client runs the full simulation locally.

> **Production Caveat**: Coherence's documentation explicitly states: *"This feature is currently not production ready out-of-the-box. The user experience is not up to our standards yet."* Additionally, *"GGPO is not recommended for FPS-style games."*

### How GGPO Works in Coherence

1. **Input Delay**: When a player presses a button at frame N, the input is scheduled for frame N + delay. The input is sent immediately to all other clients, who will likely receive it before the target frame.

2. **Prediction**: If a remote player's input has not arrived for the current frame, the system assumes the input is unchanged from the previous frame. The documentation notes: *"this assumption is valid most of the time."*

3. **Rollback**: The system maintains a history of game states. When a mispredicted input arrives for a past frame, the system:
   - Restores the simulation to the last known-good state
   - Re-applies the corrected inputs
   - Re-simulates forward to the current frame

### CoherenceInputSimulation\<TState\> Base Class

The core abstraction for GGPO rollback. Developers subclass this with a custom state struct that captures the entire simulation state.

#### State Struct

The state struct must contain **all information** needed to restore the simulation to any past frame:

```csharp
using UnityEngine;

public struct SimulationState
{
    public Vector3[] PlayerPositions;
}
```

The framework assumes the same number and order of players in the simulation. Player order is guaranteed by `CoherenceInputSimulation`, but handling variable client counts is the developer's responsibility.

#### Required Method Overrides

**SetInputs** -- Called each frame for the local player to sample and submit input:

```csharp
protected override void SetInputs(CoherenceClientConnection client)
{
    var player = client.GameObject.GetComponent<Player>();
    player.SetMovement();
}
```

**Simulate** -- Executes one frame of game logic. Called during both initial simulation and during re-simulation after a misprediction. The rollback complexity is handled internally by the framework:

```csharp
protected override void Simulate(long simulationFrame)
{
    foreach (CoherenceClientConnection client in AllClients)
    {
        var player = client.GameObject.GetComponent<Player>();
        var movement = (Vector3)player.GetMovement(simulationFrame);
        player.transform.position += movement * player.Speed * FixedTimeStep;
    }
}
```

**Rollback** -- Restores the simulation to a previously saved state at a specific frame:

```csharp
protected override void Rollback(long toFrame, SimulationState state)
{
    for (var i = 0; i < AllClients.Count; i++)
    {
        Transform playerTransform = AllClients[i].GameObject.transform;
        playerTransform.position = state.PlayerPositions[i];
    }
}
```

**CreateState** -- Captures a snapshot of the current simulation state for rollback storage:

```csharp
protected override SimulationState CreateState()
{
    var simulationState = new SimulationState
    {
        PlayerPositions = new Vector3[AllClients.Count]
    };

    for (var i = 0; i < AllClients.Count; i++)
    {
        Transform playerTransform = AllClients[i].GameObject.transform;
        simulationState.PlayerPositions[i] = playerTransform.position;
    }

    return simulationState;
}
```

#### Optional Lifecycle Callbacks

```csharp
protected override void OnClientJoined(CoherenceClientConnection client)
{
    SimulationEnabled = AllClients.Count >= 2;
    if (SimulationEnabled)
    {
        StateStore.Clear();
    }
}

protected override void OnClientLeft(CoherenceClientConnection client)
{
    SimulationEnabled = AllClients.Count >= 2;
}
```

`SimulationEnabled` defaults to `false` and must be explicitly set to `true` to begin simulation. `StateStore.Clear()` resets the rollback history.

### Input Handling via CoherenceInput

The `Player` component reads and writes input through `CoherenceInput`:

```csharp
using Coherence.Toolkit;
using UnityEngine;

[RequireComponent(typeof(CoherenceSync))]
[RequireComponent(typeof(CoherenceInput))]
public class Player : MonoBehaviour
{
    public float Speed = 5f;

    private CoherenceInput input;

    private void Awake()
    {
        input = GetComponent<CoherenceInput>();
    }

    public Vector2 GetMovement(long frame)
    {
        return input.GetAxis2D("Mov", frame);
    }

    public void SetMovement()
    {
        Vector2 movement = new Vector2(
            Input.GetAxis("Horizontal"),
            Input.GetAxis("Vertical")
        ).normalized;
        input.SetAxis2D("Mov", movement);
    }
}
```

In GGPO mode, `GetAxis2D` / `GetButton` etc. take a `long frame` parameter to retrieve the input for a specific simulation frame (which may be a past frame during re-simulation).

### FixedUpdateInput: Preventing Lost Keypresses

Because `FixedNetworkUpdate` runs at a different rate than `Update`, single-frame keypresses can be lost. The `FixedUpdateInput` helper samples inputs during `Update` and extends their lifetime to `FixedNetworkUpdate`:

```csharp
using Coherence.Toolkit;
using UnityEngine;

[RequireComponent(typeof(CoherenceSync))]
[RequireComponent(typeof(CoherenceInput))]
public class Player : MonoBehaviour
{
    private FixedUpdateInput fixedUpdateInput;
    private CoherenceInput input;

    private void Awake()
    {
        var coherenceSync = GetComponent<CoherenceSync>();
        fixedUpdateInput = coherenceSync.Bridge.FixedUpdateInput;
        input = coherenceSync.Input;
    }

    public bool GetJump(long frame)
    {
        return input.GetButton("Jump", frame);
    }

    public void SetJump()
    {
        bool hasJumped = fixedUpdateInput.GetKey(KeyCode.Space);
        input.SetButton("Jump", hasJumped);
    }
}
```

> **Note**: `FixedUpdateInput` only works with the legacy `UnityEngine.Input` system, not the new Input System package.

### Input Delay Configuration

Input delay is the number of frames between when a player presses a button and when that input takes effect. Higher delay gives more time for inputs to propagate to remote clients (reducing rollbacks) but increases perceived input lag.

Example: With a delay of 3 frames, a button press at frame 10 is scheduled for frame 13 and sent immediately to remote clients, who will likely receive it before frame 13.

### Pause Mechanism and Time.timeScale Catch-Up

When prediction exceeds the input buffer capacity (i.e., the local client has advanced too far ahead without receiving remote inputs), the simulation **pauses**. This pause affects only the local client.

```csharp
protected override void OnPauseChange(bool isPaused)
{
    PauseScreen.SetActive(isPaused);
}
```

After the pause resolves (sufficient remote inputs arrive), the system automatically speeds up the simulation to catch up. The time scale change is **gradual** -- for small frame gaps, it can be imperceptible. The system adjusts `NetworkTime.NetworkTimeScale`, which is synced to Unity's `Time.timeScale` when `CoherenceBridge.controlTimeScale` is enabled (the default). Developers can disable automatic time scale control via the `CoherenceBridge.controlTimeScale` flag for manual handling.

---

## 5. InputBuffer Configuration

The InputBuffer settings are found on the `CoherenceBridge` component:

### InputBufferSize (Initial Buffer Size)

Determines how far ahead the local client can predict before pausing. A larger buffer:
- Allows more prediction frames before a pause is triggered
- Increases the window for potentially surprising rollbacks (further-back corrections)

### InitialBufferDelay (Initial Buffer Delay)

The number of frames to delay before applying input. A higher delay:
- Reduces the likelihood of misprediction (remote inputs have more time to arrive)
- Increases perceived input lag for the player

### Disconnect on Time Reset

An option to automatically disconnect the client when the frame drift between the client and the Replication Server exceeds a recovery threshold.

### Use Fixed Simulation Frames

Must be enabled for deterministic simulation. When active, the system uses `IClient.ClientFixedSimulationFrame` for frame tracking.

### Prefab Configuration Checklist (GGPO)

| Setting | Value |
|---|---|
| Simulate | Client Side |
| Auto-sync | Always Client Predict |
| Use Fixed Simulation Frames | Enabled |
| Client Connection Prefab | Registered in CoherenceBridge |
| CoherenceBridge.Adjust Simulation Frame | Enabled |

---

## 6. Frame Synchronization Across Clients

Coherence maintains a simulation frame clock that all connected clients synchronize against. The Replication Server runs an internal clock at 60Hz (~16ms per tick).

### Three Frame Types

1. **Simulation Frame**: The main networked clock, accessible via `CoherenceBridge.NetworkTime`. Tracks both `ServerSimulationFrame.Frame` (authoritative server clock) and `ClientSimulationFrame.Frame` (local client clock).

2. **Client Fixed Simulation Frame**: A locally-controlled frame counter that advances in consistent increments based on `Time.fixedDeltaTime`. Accessed via `CoherenceBridge.ClientFixedSimulationFrame`. Uses `NetworkTime.NetworkTimeScale` to correct drift. This is the frame used by `CoherenceInput` and the GGPO system.

3. **Connection Simulation Frame**: Captured at the moment a client connects, serving as the baseline reference.

### Synchronization Mechanism

Clients continuously monitor the distance between their local frame counter and the server's frame counter. `NetworkTime.NetworkTimeScale` is adjusted based on the distance, ping, delta time, and other factors. Under ideal conditions, all clients connected to a session should have exactly the same `ClientFixedSimulationFrame` value at any point in real-world time.

### Key Behaviors

- **Time.timeScale Integration**: Automatically synced to `NetworkTime.NetworkTimeScale` when `CoherenceBridge.controlTimeScale` is enabled (default).
- **Frame Skipping**: Client simulation frames may increment by more than 1 between engine frames during low framerates.
- **OnFixedNetworkUpdate**: Guarantees execution for every frame without skipping. This is the loop that drives `CoherenceInput` and the GGPO system.
- **Entity Timestamping**: Client simulation frames timestamp outgoing entity changes so remote clients can apply them at the correct point in time.

### Querying the Current Frame

```csharp
// Via CoherenceBridge
long serverFrame = bridge.NetworkTime.ServerSimulationFrame.Frame;
long clientFrame = bridge.ClientFixedSimulationFrame;

// Via CoherenceInputSimulation
long currentFrame = CurrentSimulationFrame;
```

---

## 7. Debug Infrastructure (COHERENCE_INPUT_DEBUG)

### Enabling Debug Mode

Add the scripting define symbol `COHERENCE_INPUT_DEBUG` to your Unity project's Player Settings. This enables automatic collection of per-frame debug data.

### Automatic JSON Dump

On client disconnect, frame data is dumped to a JSON file: `inputDbg_<ClientId>.json` (written to the project root when running in the editor).

Override the dump behavior via the `CoherenceInputDebugger.OnDump` delegate.

### Debug Data Per Frame

| Field | Description |
|---|---|
| `Frame` | Simulation frame number |
| `AckFrame` | Lowest frame with confirmed inputs from all clients |
| `ReceiveFrame` | Lowest frame with received (but not necessarily confirmed) inputs from all clients |
| `AckedAt` | Frame number when acknowledgment occurred |
| `MispredictionFrame` | The known mispredicted frame, or -1 if none |
| `Hash` | State hash (requires `IHashable` on the state struct) |
| `InitialState` | State snapshot before rollback/re-simulation |
| `InitialInputs` | Original inputs before correction |
| `UpdatedState` | State snapshot after rollback/re-simulation |
| `UpdatedInputs` | Corrected inputs after rollback |
| `InputBufferStates` | Per-client input buffer state dump |
| `Events` | Custom and system events for the frame |

### Custom Debug Events

Accessed via `CoherenceInputSimulation.Debugger`:

```csharp
protected override void SetInputs(CoherenceClientConnection client)
{
    Debugger.AddEvent("myCustomEvent", new
    {
        Data = "my custom data",
        UnityFrameCount = Time.frameCount
    });
}
```

### FramesToKeep

Controls how many frames of debug data are kept in memory. Unlimited by default.

---

## 8. Desync Detection via IHashable

To detect simulation desync between clients, the state struct should implement `IHashable`. The framework computes and compares hashes across clients to identify divergence.

```csharp
using Coherence.Toolkit;
using Newtonsoft.Json;
using UnityEngine;

public struct SimulationState : IHashable
{
    [JsonProperty(ItemConverterType = typeof(UnityVector3Converter))]
    public Vector3[] PlayerPositions { get; set; }

    public Hash128 ComputeHash()
    {
        return Hash128.Compute(PlayerPositions);
    }
}
```

The `[JsonProperty]` attribute ensures proper serialization of Unity types in the debug JSON dump. The computed hash appears in the debug output's `Hash` field, allowing developers to compare state hashes across clients to pinpoint exactly when and where a desync occurs.

---

## 9. Determinism Requirements

For the GGPO rollback model to function correctly, the simulation **must** be fully deterministic: given the same inputs and starting state, every client must produce the exact same output.

### What to Avoid

| Pitfall | Why It Breaks Determinism |
|---|---|
| Using `Update` instead of `FixedUpdate` | Variable delta time produces different results per frame rate |
| Coroutines, async code, or `System.DateTime` | Execution timing varies between clients |
| Unity's built-in physics (PhysX) | Non-deterministic across platforms and even between runs |
| Unseeded `Random` / `UnityEngine.Random` | Different random sequences per client |
| Non-symmetrical player processing | Processing players in different orders produces different results |
| Cross-platform floating-point arithmetic | IEEE 754 implementation varies between CPU architectures |

### What to Use Instead

- Fixed timestep simulation via `FixedNetworkUpdate`
- Deterministic math libraries (fixed-point arithmetic for cross-platform)
- Seeded random number generators shared across all clients
- Process all players in a guaranteed consistent order (provided by `AllClients` in `CoherenceInputSimulation`)

### Desync Risks

- Starting simulation on different frames across clients causes immediate desync
- Mid-session joining without full state synchronization causes desync
- Objects subject to `LiveQuery` visibility (appearing/disappearing based on distance) can cause simulation divergence

The system **requires the Client Connection system**, which is not subject to LiveQuery. The documentation warns: *"Objects that might disappear or change based on the client-to-client distance are likely to cause simulation divergence leading to a desync."*

---

## 10. Production Readiness Caveat

Coherence is transparent about the maturity of their GGPO implementation:

> *"This feature is currently not production ready out-of-the-box. The user experience is not up to our standards yet."*

Additionally, the GGPO model is **not recommended for FPS-style games**. Coherence states that a dedicated rollback networking solution for FPS games is planned for the future.

The state-based prediction model (Model A) is more mature and suitable for production use in typical server-authoritative game setups.

---

## 11. Complete GGPO Rollback Example

Bringing all the pieces together, here is a minimal but complete rollback setup:

### SimulationState.cs
```csharp
using Coherence.Toolkit;
using Newtonsoft.Json;
using UnityEngine;

public struct SimulationState : IHashable
{
    [JsonProperty(ItemConverterType = typeof(UnityVector3Converter))]
    public Vector3[] PlayerPositions { get; set; }

    public Hash128 ComputeHash()
    {
        return Hash128.Compute(PlayerPositions);
    }
}
```

### GameSimulation.cs (CoherenceInputSimulation subclass)
```csharp
using Coherence.Connection;
using Coherence.Toolkit;
using UnityEngine;

public class GameSimulation : CoherenceInputSimulation<SimulationState>
{
    protected override void SetInputs(CoherenceClientConnection client)
    {
        var player = client.GameObject.GetComponent<Player>();
        player.SetMovement();

        Debugger.AddEvent("inputSet", new { Frame = CurrentSimulationFrame });
    }

    protected override void Simulate(long simulationFrame)
    {
        foreach (CoherenceClientConnection client in AllClients)
        {
            var player = client.GameObject.GetComponent<Player>();
            var movement = (Vector3)player.GetMovement(simulationFrame);
            player.transform.position += movement * player.Speed * FixedTimeStep;
        }
    }

    protected override void Rollback(long toFrame, SimulationState state)
    {
        for (var i = 0; i < AllClients.Count; i++)
        {
            Transform playerTransform = AllClients[i].GameObject.transform;
            playerTransform.position = state.PlayerPositions[i];
        }
    }

    protected override SimulationState CreateState()
    {
        var simulationState = new SimulationState
        {
            PlayerPositions = new Vector3[AllClients.Count]
        };

        for (var i = 0; i < AllClients.Count; i++)
        {
            Transform playerTransform = AllClients[i].GameObject.transform;
            simulationState.PlayerPositions[i] = playerTransform.position;
        }

        return simulationState;
    }

    protected override void OnClientJoined(CoherenceClientConnection client)
    {
        SimulationEnabled = AllClients.Count >= 2;
        if (SimulationEnabled)
        {
            StateStore.Clear();
        }
    }

    protected override void OnClientLeft(CoherenceClientConnection client)
    {
        SimulationEnabled = AllClients.Count >= 2;
    }

    protected override void OnPauseChange(bool isPaused)
    {
        // Show/hide pause UI
        Debug.Log(isPaused ? "Simulation paused (waiting for inputs)" : "Simulation resumed");
    }
}
```

### Player.cs (Input component)
```csharp
using Coherence.Toolkit;
using UnityEngine;

[RequireComponent(typeof(CoherenceSync))]
[RequireComponent(typeof(CoherenceInput))]
public class Player : MonoBehaviour
{
    public float Speed = 5f;

    private CoherenceInput input;
    private FixedUpdateInput fixedUpdateInput;

    private void Awake()
    {
        input = GetComponent<CoherenceInput>();
        var coherenceSync = GetComponent<CoherenceSync>();
        fixedUpdateInput = coherenceSync.Bridge.FixedUpdateInput;
    }

    public Vector2 GetMovement(long frame)
    {
        return input.GetAxis2D("Mov", frame);
    }

    public void SetMovement()
    {
        Vector2 movement = new Vector2(
            Input.GetAxis("Horizontal"),
            Input.GetAxis("Vertical")
        ).normalized;
        input.SetAxis2D("Mov", movement);
    }

    public bool GetJump(long frame)
    {
        return input.GetButton("Jump", frame);
    }

    public void SetJump()
    {
        bool hasJumped = fixedUpdateInput.GetKey(KeyCode.Space);
        input.SetButton("Jump", hasJumped);
    }
}
```

---

## 12. References

- [Coherence Authority Overview](https://docs.coherence.io/manual/authority)
- [Server-Authoritative Setup](https://docs.coherence.io/manual/authority/server-authoritative-setup)
- [Determinism, Prediction and Rollback](https://docs.coherence.io/manual/advanced-topics/competitive-games/determinism-prediction-rollback)
- [Competitive Games Overview](https://docs.coherence.io/manual/advanced-topics/competitive-games)
- [Simulation Frame](https://docs.coherence.io/manual/advanced-topics/competitive-games/simulation-frame)
- [Coherence Documentation Home](https://docs.coherence.io/)
