# SnapNet -- Multiplayer Networking SDK

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Key Concepts and Terminology](#3-key-concepts-and-terminology)
4. [Client-Side Prediction](#4-client-side-prediction)
5. [Reconciliation and Rollback](#5-reconciliation-and-rollback)
6. [State Synchronization](#6-state-synchronization)
7. [Lag Compensation (Server Rewind)](#7-lag-compensation-server-rewind)
8. [Input Handling](#8-input-handling)
9. [Snapshot System and Replays](#9-snapshot-system-and-replays)
10. [Code Examples](#10-code-examples)
11. [Comparison to Other Frameworks](#11-comparison-to-other-frameworks)
12. [Blog Series: Netcode Architectures](#12-blog-series-netcode-architectures)
13. [Trade-offs, Strengths, and Limitations](#13-trade-offs-strengths-and-limitations)
14. [References](#14-references)

---

## 1. Overview

**SnapNet** is a commercial, complete netcode solution developed by **High Horse Entertainment** (founded by Jay Mattis). It is positioned as "AAA netcode that handles all the tricky bits" -- a software library for synchronizing real-time gameplay over the Internet with a focus on **competitive integrity** and **responsiveness**.

### Design Philosophy

SnapNet's core philosophy is that *implementing a game mechanic and networking it should be the same thing* -- all features work online from day one. Traditionally, gameplay developers handle their own networking code, which often leads to bugs, exploits, and inconsistent responsiveness. SnapNet aims to eliminate this class of problems by providing a unified simulation layer where networked behavior is baked in from the start.

### Platform and Engine Support

| Platform | Integration Method |
|----------|-------------------|
| **Unreal Engine** | Standalone plugin (no engine modifications required) |
| **Unity** | UPM (Unity Package Manager) |
| **Custom Engines** | Available upon request (C API) |

### Genre Support

SnapNet provides genre-specific configurations:

- **Shooters**: Client-side prediction for aiming/movement; remote players interpolated between server updates
- **Fighting Games**: Full state rollback without strict determinism requirements (unlike GGPO)
- **Sports**: Handles multiple players interacting with shared physics objects (e.g., a ball)

### Licensing

SnapNet is a commercial product. Evaluation licenses and demos are available by contacting `snapnet@highhorse.dev`. No public pricing is listed.

---

## 2. Architecture

### Client-Server, Server-Authoritative Model

SnapNet uses a **server-authoritative architecture** with **dedicated server support**. The server is "a process running on a physical machine that accepts incoming connections from clients, simulates the game, and is the ground source of truth." Clients never have final authority over game state -- the server always adjudicates.

### Multiple Worlds / Simulation Isolation

A distinguishing architectural feature of SnapNet is its **multiple UWorlds** design (in Unreal Engine). Rather than a single world for both gameplay logic and rendering, SnapNet creates three separate worlds:

| World | Purpose |
|-------|---------|
| **Main / Presentation World** | Contains rendered actors, collision data, physics, navigation. Runs exactly once per rendered frame. |
| **Client Simulation World** | Maintains separate client actors, collision, physics for prediction/rollback. May run multiple times per rendered frame. |
| **Server Simulation World** | Maintains separate server actors, collision, physics. Authoritative simulation. |

This separation is managed by the **SnapNet Subsystem** (a subclass of `UGameInstanceSubsystem`), which initializes both a SnapNet Server and a SnapNet Client, each with its own `SnapNet Simulation`.

### Simulation vs. Presentation

SnapNet fundamentally decouples simulation from presentation:

- **Simulation**: Any logic or code that advances the networked state (gameplay code). Must be lightweight because it may execute multiple times per rendered frame during rollback/reconciliation.
- **Presentation**: Visual and cosmetic elements (spawning meshes, shaders, VFX, audio). Guaranteed to run exactly once per rendered frame.

**Practical example from the docs**: A character cape with cloth physics should *not* be part of simulation (saving, restoring, and resimulating cloth physics is infeasible). Instead, cloth runs only during the presentation phase.

### Fixed Tick Rate with Variable Frame Rate Rendering

SnapNet runs gameplay simulation at a **fixed tick rate** (e.g., 60 Hz, configurable in project settings). Rendering is fully decoupled -- SnapNet automatically interpolates between simulation frames, enabling "a predictable fixed tick rate simulation that can render smoothly at any frame rate."

### Security and Competitive Integrity

- **DDoS Protection**: Stateless cookie verification prevents amplification attacks
- **Stateless Authentication**: Blocks connection flooding exploits
- **X.509 Certificate Validation**: Prevents man-in-the-middle attacks
- **DTLS Encryption**: End-to-end encryption for all communications
- **Server Authority**: Limits client-side cheating possibilities
- **Connection Fairness**: Maintains simulation quality for well-connected players despite poor connections elsewhere

### Transport Layer

SnapNet supports multiple transport options:

- **DTLS Transport**: Production transport with server certificates and authentication tokens
- **Steam Transport**: For development (not intended for production)
- **Offline Sessions**: For single-player or testing

---

## 3. Key Concepts and Terminology

### Glossary (from official documentation)

| Term | Definition |
|------|-----------|
| **Entity** | The core gameplay object in SnapNet (analogous to Unreal Actors or Unity GameObjects). Contains networked fields that are synchronized each frame. |
| **Entity Index** | An integer that uniquely identifies a specific entity in the simulation, globally agreed upon between server and clients. Reused after entity destruction. |
| **Event** | Constructs for networking visual/sound effects and transmitting instantaneous state. Callbacks execute during prediction, then again when confirmed or canceled by the server. |
| **Client** | Represents a physical machine connected to a server. Distinct from players -- a single client may support multiple humans (splitscreen). |
| **Client Index** | An integer that uniquely identifies a specific client, globally synchronized. Reused on disconnect. -1 indicates invalid. |
| **Player** | A human or bot that is playing the game, providing input and potentially owning entities and events. |
| **Player Index** | An integer that uniquely identifies a specific player, globally synchronized and reused after departure. |
| **Local Player** | A human playing the game on a particular client. Multiple local players possible in splitscreen. |
| **Local Player Index** | An integer identifying a specific local player per client (0 = first player on that client). |
| **Server** | The process that accepts connections, simulates the game, and is the ground source of truth. |
| **Backwards Reconciliation** | The process of rewinding the state of interpolated entities on the server to match what a given player was seeing on that frame. Also called **Server Rewind**. |
| **Reliable Message** | Packets of data sent reliably outside the normal game traffic flow (similar to RPCs). Not stored in replays. Used for UI commands, chat, etc. |
| **Instant Replay** | Earlier gameplay shown to clients while the match is running, for killcams or highlight reels. |
| **Full Match Replay** | Server-emitted gameplay chunks that clients can later replay from any player's perspective with lag compensation. |

### Interpolated vs. Predicted Entities

Every entity must be designated as either **interpolated** or **predicted**:

| Aspect | Interpolated | Predicted |
|--------|-------------|-----------|
| **Behavior** | Clients smoothly interpolate between server updates; entities are slightly behind real-time | Clients simulate forward using server logic, predicting future state |
| **CPU Cost** | Minimal -- only simple interpolation | Higher -- rollback, reconciliation, and prediction cycles |
| **Responsiveness** | Requires full round-trip for local changes | Instant response to local input |
| **Artifacts** | None -- only travels through actual server states | Possible misprediction artifacts (popping, rubber-banding) |
| **Server Rewind** | Supports backwards reconciliation for lag compensation | Cannot perform server rewind |
| **Timeline** | Behind real-time | Ahead of server (aligned with other predicted entities) |

**Genre recommendations**:
- **Fighting / Sports**: Predict all entities
- **Shooters**: Predict only locally-owned entities; interpolate remote players (except deterministic projectiles like rockets)

### Input Delay vs. Rollback

SnapNet supports a **hybrid model** combining input delay and rollback:

- **Lockstep / Input Delay mode**: Clients send inputs, wait for server response, then show results. Zero visual disagreements but response delay equals RTT. Acceptable with low ping (~50ms or less).
- **Rollback / Client-Side Prediction mode**: Clients simulate immediately without waiting for the server. Instant visual feedback at the cost of possible misprediction artifacts and CPU overhead from resimulation.

SnapNet allows **dynamic blending** of both approaches based on network conditions.

---

## 4. Client-Side Prediction

### How It Works

Client-side prediction (also called rollback) allows clients to simulate the results of their input and show those results immediately, without waiting for the server's response. Because the simulation runs without knowing the server's response or other players' inputs, it is called **prediction**.

For predicted entities, the process is:

1. The client samples local input
2. The input is sent to the server
3. Simultaneously, the client applies the input to its local simulation and renders the predicted result
4. For remote player inputs, the system carries the **last known input forward** -- assuming the remote player continues their previous action

### Input Decay for Smoother Predictions

To reduce teleporting/rubber-banding during major direction changes, SnapNet supports **input decay**. Movement input diminishes over prediction frames:

```
Frame 0: 100% of last known input
Frame 1: 66%
Frame 2: 33%
Frame 3: 0% (neutral)
```

This causes motion to undershoot rather than overshoot, so corrections appear more natural. This technique is notably used by Rocket League.

The input age can be retrieved per-entity and used to decay analog inputs programmatically, preventing extreme mispredictions.

### Prediction Limits

The longer the prediction horizon, the more likely mispredictions become. The SnapNet documentation notes that prediction beyond **100ms to 150ms** tends to become unplayable due to accumulated inconsistency. However, for shooters (which only predict the local player), prediction can be extended much further (up to **1000ms**) because local-player misprediction is rare.

### Works with Non-Deterministic Physics

Unlike GGPO-style rollback, SnapNet does **not require strict bitwise determinism**. It works with engine-built physics (rigid bodies, navmeshes, etc.) without synchronization issues. This is a significant differentiator -- traditional rollback architectures demand deterministic simulation, which is extremely difficult to achieve with floating-point math, third-party physics engines, and cross-platform builds.

---

## 5. Reconciliation and Rollback

### The Reconciliation Process

When authoritative server state arrives, the client must integrate it into its current predicted state. SnapNet performs **reconciliation** as follows:

1. **Rewind**: The client restores the entire simulation state to the last frame where all inputs were known (the last non-predicted, authoritative frame)
2. **Resimulate**: Re-advance the simulation frame-by-frame using the original local inputs plus the newly received authoritative remote inputs
3. **Re-predict**: Continue predicting forward from the corrected state for any frames that still lack authoritative data
4. **Render**: Display the final corrected frame

### Example Timeline

Suppose a client is rendering frame 7 when it receives the authoritative remote input for frame 5:

1. Load the complete game state from frame 4 (last frame with all known inputs)
2. Re-advance using correct inputs through frame 5
3. Re-predict frames 6 and 7 with the original local inputs
4. Render the corrected frame 7

### Rollback-Aware Events and Effects

SnapNet's event system handles mispredictions intelligently in the Shooter Game Sample:

- **Impact effects** (decals, blood splatters): Triggered immediately during prediction for instant visual feedback
- **Confirmation effects** (crosshair hit markers): Play only when the server confirms the hit
- **Mispredicted effects**: Terminated early if the server disagrees (e.g., a shot that missed due to excessive latency)

### Performance Considerations

Rollback demands significant CPU efficiency. Advancing the simulation multiple times within one render frame creates tight budgets. For a 60 Hz game supporting 300ms latency with 3-frame input delay, the CPU must resimulate ~15 frames in 16.66ms -- roughly **1.1ms per simulation frame** versus the full 16.66ms available under lockstep.

The **"spiral of death"** can occur when resimulation exceeds the frame budget, causing the next frame to require even more rollback, cascading into a performance collapse.

---

## 6. State Synchronization

### Networked Properties

SnapNet synchronizes state through **networked properties**. Adding a variable that derives from `FSnapNetProperty` (in C++ or Blueprint) automatically synchronizes it. The same system works consistently across entities, events, reliable messages, and player join requests.

#### Supported Data Types

| Category | Types |
|----------|-------|
| **Primitives** | Boolean, Double, Float, Int32 |
| **Index Types** | Client Index, Entity Index, Player Index |
| **Geometric** | Vector, Vector 2D, Rotator, Quaternion |
| **Specialized Vectors** | Angular Velocity, Position, Velocity |
| **Complex** | String, Soft Object Path, Enum, Primary Asset |
| **Collections** | Dynamic arrays, fixed-length arrays, nested structs |
| **Custom** | Derive from `FSnapNetProperty` for full control over serialization, delta compression, and interpolation |

#### Bandwidth Optimization

SnapNet employs aggressive bandwidth optimization techniques drawn from the snapshot interpolation model:

- **Bit-packing**: Data encoded bit-by-bit rather than byte-by-byte (e.g., health 0-100 uses 7 bits instead of 32)
- **Quantization**: Reducing floating-point precision (e.g., positions in a <1km world using 17 bits per axis)
- **Delta encoding**: Only data changed since the most recently acknowledged state is transmitted
- **String pools**: Replace full strings with compact integer indices for finite string sets
- **Compression**: Entropy encoding and other algorithms further reduce payload size

#### Relevance Settings

Each property has relevance settings controlling which clients receive it:

- Always / Never
- Simulated / Not Simulated
- Spectated / Not Spectated
- Simulated or Spectated / Neither

#### Interpolation Settings

Automatic interpolation for float and vector types enables fixed tick-rate simulation with variable frame-rate rendering. Features include:

- **Looping**: Handles wraparound values (e.g., animation timers resetting from 0.98 to 0.01) via `SetLoop(true)`
- **Discontinuities**: `ESnapNetInterpolation::SnapToValue` prevents unwanted interpolation transitions when values reset

#### Real Number Encodings

- **Fixed Range**: Transmits floats as integers based on min/max/precision (e.g., health)
- **Signed Range**: Handles zero-crossing ranges by encoding magnitude + sign separately
- **Floating Point**: Uses specified exponent/significand bit counts for values spanning multiple orders of magnitude (e.g., physics velocities)

---

## 7. Lag Compensation (Server Rewind)

### The Problem

When using entity interpolation for remote characters alongside prediction for the local player, a timing mismatch occurs:

- Remote players appear in **interpolated past positions** (behind real-time)
- Local actions are **predicted ahead** of server confirmation
- By the time client actions reach the server, remote entities have moved to new states

For hitscan weapons, players would need to "lead their targets" for the server to register hits -- which is unacceptable.

### The Solution: Backwards Reconciliation

SnapNet's backwards reconciliation rewinds the **entire simulation** to match what a given player was seeing when they performed an action. The server reconstructs the exact client-perceived state at the moment of input.

For shooters, this means: "the server can determine precisely who or what the player had in their crosshairs when they fired their weapon."

### Implementation: `FSnapNetScopedRewind`

```cpp
// Rewind the simulation to what the client saw when they fired
FSnapNetScopedRewind Rewind( EntityComponent );

// Perform hit detection queries against the rewound state
// (line traces, overlap checks, etc.)
FHitResult HitResult;
// ... trace logic here ...

// When Rewind goes out of scope, simulation state is automatically restored
```

### Key Characteristics

- Rewinds **entire entities**, not just individual properties or hitboxes
- Allows checking historical ability states (e.g., was the target invulnerable when shot?)
- Configurable **maximum rewind duration** (default: 200ms, comparable to competitive shooters)
- Longer rewind durations accommodate higher-latency players but extend "peeker's advantage"
- Performance optimization: optionally specify an array of entity indices to rewind instead of the entire simulation

### Why Not Client-Authoritative Hit Detection?

SnapNet's blog post on lag compensation in UE5 explains why client-authoritative approaches (like those in Epic's ShooterGame and Lyra samples) are problematic:

- **Security**: "I shot this guy, trust me" creates exploit vectors (shooting from impossible locations, manipulating fire rates)
- **No timing guarantees**: "There is no guarantee when [the server] receives an RPC how long ago it was sent"
- **Fairness**: Without server-side validation, high-latency players can cause unfair deaths

### Lag-Compensated Replays

Both instant and full match replays automatically apply lag compensation. When spectating any player's perspective, viewers see "the state of the world as they saw it rather than simply the state of the server."

---

## 8. Input Handling

### Input Configuration

SnapNet supports three input systems:

1. **Legacy Unreal Input**: Configured via Project Settings matching Engine Input axis/action names
2. **Enhanced Input** (Epic's plugin): Register `UInputAction` assets in SnapNet settings. Input Mapping Contexts are not networked -- clients apply them locally.
3. **Custom Input**: Subclass `USnapNetCustomInput` and override `Populate()` for arbitrary client-side data.

An optional **"Include Control Rotation"** checkbox makes each player's control rotation available in the simulation (used for first-person cameras, aim direction, etc.).

### Accessing Input in Simulation Code

```cpp
// Legacy Input
const float MoveForward = Simulation->GetInputAxis(PlayerIndex, "MoveForward");
const bool bJumpHeld = Simulation->IsInputActionDown(PlayerIndex, "Jump");
const bool bJumpPressed = Simulation->WasInputActionPressed(PlayerIndex, "Jump");
const bool bJumpReleased = Simulation->WasInputActionReleased(PlayerIndex, "Jump");

// Enhanced Input
const FVector2D MoveValue = Simulation->GetEnhancedInputActionValue<FVector2D>(
    PlayerIndex, MoveAction);
const bool bJumpHeld = Simulation->GetEnhancedInputActionValue<bool>(
    PlayerIndex, JumpAction);
const bool bJumpPressed = Simulation->WasEnhancedInputActionPressed(
    PlayerIndex, JumpAction);

// Custom Input
if (const UMyCustomInput* MyInput = Simulation->GetCustomInput<UMyCustomInput>(PlayerIndex))
{
    const FVector CursorPos = MyInput->GetCursorPosition();
}
```

SnapNet automatically tracks both current and previous inputs, enabling direct state queries (pressed, released, held) without manual tracking.

### Security Considerations

The documentation explicitly warns against two patterns:

- **Input Triggers**: Client-side trigger logic creates "attack vectors for cheaters." Delays and timing should be implemented in simulation code using networked properties.
- **Unsanitized Custom Input**: Carefully distinguish safe inputs (button presses) from unsafe inputs (positions). Unsanitized position data enables teleportation exploits.

---

## 9. Snapshot System and Replays

### Snapshot-Based Architecture

While the name "SnapNet" may suggest a snapshot-based system, SnapNet actually implements a **hybrid architecture** that combines elements of snapshot interpolation with rollback. The server generates authoritative state that drives both interpolated and predicted entities, using snapshot interpolation principles for bandwidth optimization (delta encoding, bit-packing, quantization) while supporting full rollback for predicted entities.

### Instant Replays

SnapNet provides built-in instant replay functionality:

- **Configuration**: Set the number of saved replay slots and maximum seconds of history the server retains
- **Saving**: `SaveInstantReplay()` stores a time window (start/end in milliseconds) to a named slot
- **Playback (saved)**: `PlaySavedInstantReplay()` retrieves stored clips with player perspective and context entity parameters
- **Playback (live)**: `StartInstantReplay()` plays directly from the server buffer without prior saving -- supports end times extending into the future (e.g., killcams showing "3 seconds before death and 3 seconds after")
- **Monitoring**: `IsPlayingInstantReplay()`, `GetInstantReplaySlotIndex()`, `GetInstantReplayContextEntityIndex()`

Instant replays operate **per-client** (all splitscreen players see the same replay).

### Full Match Replays

Complete match recordings controllable via console commands:

```
SaveSnapNetReplay myreplay      // Save the match
PlaySnapNetReplay myreplay      // Load and play
SpectateSnapNetReplay 0         // Spectate from player 0's perspective
```

Full match replays are server-emitted in chunks and can be:
- Saved to disk for later viewing
- Streamed live with minimal delay during active matches
- Viewed from any player's perspective with full lag compensation

### Reliable Messages

For communication outside the normal simulation flow (UI commands, chat), SnapNet provides **reliable messages** -- similar to RPCs but integrated into the SnapNet system:

```cpp
// Create and send a chat message
UExampleChatMessage* ChatMessage = NewObject<UExampleChatMessage>();
ChatMessage->Message.SetValue(message);
Client->SendReliableMessage(LocalPlayerIndex, ChatMessage);
```

Reliable messages are **not stored in replays**. Data that should appear in replays should use entities or events instead.

---

## 10. Code Examples

### Configuring Prediction Parameters (C API)

```c
struct snapnet_client_configuration client_configuration;
snapnet_client_configuration_get_default(&client_configuration);

// First 50ms of latency handled by input delay (artifact-free)
client_configuration.min_input_delay_before_prediction_milliseconds = 0;
client_configuration.max_input_delay_before_prediction_milliseconds = 50;

// Next 100ms handled by prediction (responsive with minor artifacts)
// Beyond 150ms total: additional input delay applied gracefully
client_configuration.max_predicted_milliseconds = 100;

struct snapnet_client* client = snapnet_client_create(
    &shared_configuration, &client_configuration
);
```

### Genre-Specific Configuration Recommendations

```
// SHOOTER (maximum responsiveness, extended prediction tolerance)
min_input_delay_before_prediction_milliseconds = 0
max_input_delay_before_prediction_milliseconds = 0
max_predicted_milliseconds = 1000  // 1 second -- misprediction is rare for local player

// FIGHTING GAME (consistent input feel, moderate prediction)
min_input_delay_before_prediction_milliseconds = 50  // Match max for consistency
max_input_delay_before_prediction_milliseconds = 50
max_predicted_milliseconds = 100

// SPORTS (balanced defaults)
min_input_delay_before_prediction_milliseconds = 0
max_input_delay_before_prediction_milliseconds = 50
max_predicted_milliseconds = 100  // Or 150 for higher latency tolerance
```

### Server Rewind for Hit Detection

```cpp
void AMyWeapon::PerformHitscan(USnapNetEntityComponent* EntityComponent)
{
    // Rewind simulation to match what the firing client saw
    FSnapNetScopedRewind Rewind(EntityComponent);

    // Perform line trace against the rewound world state
    FHitResult HitResult;
    FVector Start = GetMuzzleLocation();
    FVector End = Start + GetAimDirection() * MaxRange;

    if (GetWorld()->LineTraceSingleByChannel(HitResult, Start, End, ECC_Visibility))
    {
        // Hit detected against the rewound state
        ApplyDamage(HitResult);
    }
    // Rewind automatically restored when scope exits
}
```

### Optimized Server Rewind (Selective Entities)

```cpp
// Only rewind specific entities for performance
TArray<int32> EntitiesToRewind;
EntitiesToRewind.Add(TargetEntityIndex);
EntitiesToRewind.Add(OtherRelevantEntityIndex);

FSnapNetScopedRewind Rewind(EntityComponent, EntitiesToRewind);
// ... perform queries against rewound state ...
```

### Reading Player Input in a Simulation Tick

```cpp
void UMyEntity::SimulationTick(USnapNetSimulation* Simulation, int32 PlayerIndex)
{
    // Movement input
    const float MoveForward = Simulation->GetInputAxis(PlayerIndex, "MoveForward");
    const float MoveRight = Simulation->GetInputAxis(PlayerIndex, "MoveRight");

    // Action input with press/release detection
    if (Simulation->WasInputActionPressed(PlayerIndex, "Fire"))
    {
        BeginFiring();
    }
    if (Simulation->WasInputActionReleased(PlayerIndex, "Fire"))
    {
        StopFiring();
    }

    // Enhanced Input alternative
    const FVector2D MoveValue = Simulation->GetEnhancedInputActionValue<FVector2D>(
        PlayerIndex, MoveAction);
}
```

### Networked Property Declaration

```cpp
UCLASS()
class UExampleEntity : public USnapNetEntity
{
    GENERATED_BODY()

public:
    // Automatically synchronized across the network
    UPROPERTY()
    FSnapNetPropertyFloat Health;

    UPROPERTY()
    FSnapNetPropertyVector Position;

    UPROPERTY()
    FSnapNetPropertyRotator Rotation;

    UPROPERTY()
    FSnapNetPropertyInt32 Score;

    UExampleEntity()
        : Health(0.0f, 100.0f, 0.5f)  // min, max, precision
        , Position(/*...encoding config...*/)
        , Rotation(/*...encoding config...*/)
        , Score(0, 999, 1)
    {
    }
};
```

### Instant Replay (Single Line)

```cpp
// Trigger a killcam: replay 3 seconds before death to 3 seconds after
Server->StartInstantReplay(
    VictimClientIndex,
    AttackerPlayerIndex,      // Perspective
    CurrentTimeMs - 3000,     // Start time
    CurrentTimeMs + 3000,     // End time
    ContextEntityIndex        // Optional context
);
```

---

## 11. Comparison to Other Frameworks

### SnapNet vs. the Field

| Feature | SnapNet | Photon Fusion (Unity) | FishNet (Unity) | Mirror (Unity) | UE5 Replication | Coherence (Unity) |
|---------|---------|----------------------|-----------------|----------------|-----------------|-------------------|
| **Engine Support** | UE, Unity, Custom | Unity only | Unity only | Unity only | UE only | Unity only |
| **Authority Model** | Server-authoritative | Server or client auth | Server-authoritative | Server-authoritative | Server-authoritative | Server-authoritative |
| **Client-Side Prediction** | Built-in, automatic | Built-in (Shared mode) | Manual implementation | Manual implementation | Manual (complex) | Built-in |
| **Rollback** | Full state rollback | Snapshot-based | Not built-in | Not built-in | Not built-in | Not built-in |
| **Reconciliation** | Automatic | Automatic (Shared mode) | Manual | Manual | Manual | Automatic |
| **Server Rewind / Lag Comp** | Built-in, entity-level | Lag Compensation module | Not built-in | Not built-in | Not built-in (samples only) | Not built-in |
| **Determinism Required** | No | No | N/A | N/A | N/A | No |
| **Physics Integration** | Engine physics, non-deterministic | Fusion physics addon | Engine physics | Engine physics | Engine physics | Engine physics |
| **Replay System** | Instant + full match, lag-compensated | Not built-in | Not built-in | Not built-in | Basic demo recording | Not built-in |
| **Transport Encryption** | DTLS (built-in) | Photon Cloud | Varies | Varies | Varies | Coherence Cloud |
| **Fixed Tick + Variable Render** | Built-in | Built-in | Manual | Manual | Partial | Built-in |
| **Pricing** | Commercial license | Per-CCU or subscription | Free (open source) | Free (open source) | Included with UE | Free tier + paid |

### Key Differentiators

**vs. Photon Fusion**: Photon Fusion is the closest competitor in feature set, offering tick-aligned state synchronization with prediction and reconciliation. However, Fusion is Unity-only, requires the Photon Cloud infrastructure, and charges per concurrent user. SnapNet is engine-agnostic, supports dedicated servers directly, and includes features like lag-compensated replays that Fusion lacks.

**vs. UE5 Native Replication**: Unreal's built-in replication is a general-purpose framework but, as SnapNet's blog argues, "doesn't provide many guarantees to gameplay code" and "it can be extremely challenging to align things accurately in time." Standard UE5 replication uses property-level replication with RPCs -- a Tribes-style model that suffers from inter-object consistency issues, jitter from uncoordinated property updates, and no built-in prediction/rollback pipeline. SnapNet replaces this with a snapshot-based system that guarantees temporal alignment.

**vs. FishNet / Mirror**: These are open-source Unity networking libraries focused on ease of use for indie developers. They provide basic state synchronization and RPCs but lack built-in prediction, rollback, reconciliation, and lag compensation. Developers must implement these features manually, which is where most netcode complexity lies.

**vs. Coherence**: Coherence provides a cloud-based networking solution for Unity with some built-in prediction. However, it does not offer the depth of rollback, server rewind, or replay features that SnapNet provides. Coherence is more focused on ease of onboarding than competitive-grade netcode.

**vs. GGPO (Rollback library)**: GGPO pioneered rollback for fighting games but requires **strict bitwise determinism** across all platforms -- a requirement SnapNet explicitly eliminates. SnapNet supports rollback with non-deterministic physics engines, making it applicable to a broader range of game genres.

---

## 12. Blog Series: Netcode Architectures

SnapNet's blog (authored by Jay Mattis) contains an excellent four-part series on netcode architectures. These are educational deep-dives, not SnapNet-specific marketing.

### Part 1: Lockstep

**Core principle**: Each player sends their input to everyone else and advances the simulation only after receiving inputs from all players.

**Key insights**:
- Minimal bandwidth (only inputs transmitted, not game state)
- Requires **strict bitwise determinism** -- identical simulation across all machines given matching inputs
- Floating-point arithmetic creates platform-specific variations; many developers use fixed-point math
- Input delay is proportional to the highest-latency player
- Scales poorly: in 8-player matches with 10% bad connections, ~57% of matches have significant delay
- **Client-server variant** eliminates determinism requirements but dramatically increases bandwidth (full state must be transmitted)
- Join-in-progress is problematic (must replay all inputs or serialize full state)
- Best suited for: RTS games, older fighting games

### Part 2: Rollback

**Core principle**: Players send input each frame and advance simulation immediately using predicted remote inputs, then reconcile when actual inputs arrive.

**Key insights**:
- Extends lockstep by removing the wait for remote inputs
- Remote inputs predicted by carrying the last known input forward
- **Input decay** variation: diminish predicted input over frames (100% -> 66% -> 33% -> 0%) to reduce teleporting on misprediction. Used by Rocket League.
- Requires game state serialization (save/load on demand)
- Misprediction becomes unplayable beyond ~100-150ms
- **Anticipatory frames** (fighting game "startup frames") help hide mispredictions in animation
- **Hybrid approach**: SnapNet configures input delay + prediction progressively:
  - First 50ms: handled via input delay (~3 frames at 60 Hz)
  - Next 100ms: handled via prediction
  - Beyond 150ms: additional input delay applied
- **Performance constraint**: Must resimulate many frames within a single render frame budget
- Best suited for: fighting games, some sports titles

### Part 3: Snapshot Interpolation

**Core principle**: Clients send inputs to the server; the server advances the authoritative simulation and sends back snapshots of game state.

**Key insights**:
- Popularized by Quake, used by Apex Legends, Call of Duty, Counter-Strike
- Splits world into **two time streams**: interpolated objects (in the past) and predicted objects (in the future)
- Instant input responsiveness for the local player
- Reduced CPU demands vs. full rollback (most objects are interpolated, not simulated)
- No prediction artifacts on interpolated objects
- Enables **backwards reconciliation** for accurate hit detection
- Bandwidth optimized via bit-packing, quantization, delta encoding, compression
- **Scalability limitation**: Packet generation scales with player count squared (unique deltas per client per object)
- Best suited for: small-to-medium scale shooters

### Part 4: Tribes (Partial State Updates)

**Core principle**: Client-server model with eventual consistency via partial state updates (the "Ghost Manager" pattern).

**Key insights**:
- Developed for the Tribes franchise, adopted by Fortnite, Halo, and is the basis of UE's native replication
- Uses **eventual consistency**: client "ghosts" will eventually match server state but may receive updates out of order
- **Control Object**: Each player gets one object guaranteed the latest state each update
- Only the control object gets deterministic prediction; everything else is extrapolated
- **Intra-object inconsistency**: Properties on a single object can arrive out of order (e.g., "in passenger seat with no vehicle")
- **Inter-object inconsistency**: Related changes across objects have no synchronization guarantees
- Tuning is an inexact science (priority assignments, bandwidth caps)
- Backwards reconciliation is extremely difficult because each object has its own time stream
- Often necessitates client-authoritative hit detection (security vulnerability)
- Best suited for: large-scale games (MMOs), bandwidth-constrained environments
- **Notable**: Overwatch succeeds with a Tribes-like model because its state fits in single packets, effectively becoming snapshot interpolation

---

## 13. Trade-offs, Strengths, and Limitations

### Strengths

1. **Comprehensive out-of-the-box solution**: Prediction, rollback, reconciliation, lag compensation, replays, and encryption are all built-in -- not piecemeal additions.

2. **No determinism requirement**: Unlike GGPO or traditional lockstep/rollback, SnapNet works with non-deterministic engine physics. This is a major practical advantage, as achieving bitwise determinism across platforms is extremely difficult.

3. **Genre flexibility**: Configurable from full rollback (fighting games) to local-only prediction (shooters) with dynamic input delay / prediction blending.

4. **Simulation-presentation separation**: Clean architectural boundary prevents expensive visual code from running during rollback resimulation. Forces developers into good practices.

5. **Server-authoritative security**: Proper hit detection via server rewind rather than trusting clients. DTLS encryption, DDoS protection, and stateless authentication.

6. **Multi-engine support**: Unlike most competitors that are locked to a single engine, SnapNet supports Unreal, Unity, and custom engines.

7. **Replay infrastructure**: Lag-compensated instant replays and full match replays with minimal implementation effort (single line of code for killcams).

8. **Educational resources**: The blog series on netcode architectures is among the best publicly available writing on the topic.

### Limitations and Trade-offs

1. **Commercial / closed-source**: SnapNet requires a license. There is no free tier, no public pricing, and no open-source access. This is a significant barrier for indie developers and hobbyists.

2. **Limited public documentation**: Some documentation pages return 404 errors; certain quick-start guides are inaccessible without a license. The community around SnapNet is small compared to Photon, Mirror, or FishNet.

3. **CPU overhead of rollback**: Like all rollback systems, SnapNet must resimulate multiple frames per render tick, imposing tight per-frame CPU budgets. The "spiral of death" is a real risk for complex simulations.

4. **Prediction artifacts are inherent**: Even with input decay and hybrid input delay, mispredictions will produce visual artifacts (teleporting, rubber-banding) at moderate-to-high latencies. This is a fundamental limitation of any prediction system.

5. **Interaction across time streams**: Predicted entities interacting with interpolated entities creates inherent temporal misalignment. This limits genre applicability -- the docs note that sports games with predicted players hitting an interpolated ball was problematic enough that Psyonix moved Rocket League to full rollback.

6. **Scalability of snapshot model**: Full snapshot synchronization scales poorly with large player counts (packet generation is O(n^2) in the worst case). Large-scale games (64+ players) may face bandwidth constraints.

7. **Learning curve**: The multiple-worlds architecture, simulation vs. presentation separation, and entity designation (interpolated vs. predicted) require a different mental model from traditional networking. Developers must understand which code runs in simulation (lightweight, deterministic-ish) vs. presentation (heavy, once-per-frame).

8. **Ecosystem maturity**: Compared to Photon (established since 2011) or UE's native replication, SnapNet has a smaller user base, fewer tutorials, fewer community-created resources, and less battle-testing at scale.

---

## 14. References

### Official SnapNet Resources

- [SnapNet Homepage](https://www.snapnet.dev/) -- Product overview, feature list, contact information
- [SnapNet Documentation](https://www.snapnet.dev/docs/) -- Full SDK documentation index
- [SnapNet Overview](https://www.snapnet.dev/docs/overview/) -- Design philosophy and feature set
- [Input Delay vs. Rollback](https://www.snapnet.dev/docs/core-concepts/input-delay-vs-rollback/) -- Core concept: hybrid input delay + rollback configuration
- [Interpolation vs. Prediction](https://www.snapnet.dev/docs/core-concepts/interpolation-vs-prediction/) -- Core concept: entity designation
- [Simulation vs. Presentation](https://www.snapnet.dev/docs/core-concepts/simulation-vs-presentation/) -- Core concept: architectural separation
- [Glossary](https://www.snapnet.dev/docs/core-concepts/glossary/) -- Official terminology definitions
- [Server Rewind (Lag Compensation)](https://snapnet.dev/docs/unreal-engine-sdk/manual/server-rewind/) -- Backwards reconciliation documentation
- [Shooter Game Sample](https://snapnet.dev/docs/unreal-engine-sdk/samples/shooter-game-sample/) -- Reference implementation for FPS networking

### SnapNet Blog Series: Netcode Architectures (by Jay Mattis)

- [Part 1: Lockstep](https://www.snapnet.dev/blog/netcode-architectures-part-1-lockstep/)
- [Part 2: Rollback](https://www.snapnet.dev/blog/netcode-architectures-part-2-rollback/)
- [Part 3: Snapshot Interpolation](https://www.snapnet.dev/blog/netcode-architectures-part-3-snapshot-interpolation/)
- [Part 4: Tribes](https://www.snapnet.dev/blog/netcode-architectures-part-4-tribes/)

### SnapNet Blog: Technical Deep-Dives

- [Performing Lag Compensation in Unreal Engine 5](https://snapnet.dev/blog/performing-lag-compensation-in-unreal-engine-5/) -- Analysis of UE5's replication limitations and the case for snapshot interpolation

### Creator

- **High Horse Entertainment** -- [highhorseentertainment.com](https://highhorseentertainment.com)
- Contact: `snapnet@highhorse.dev`
- Copyright: 2021-2025 High Horse Entertainment

### Community and Ecosystem

- [Multiplayer Networking Resources](https://multiplayernetworking.com/) -- Curated list that includes SnapNet
- [GitHub: MultiplayerNetworkingResources](https://github.com/0xFA11/MultiplayerNetworkingResources) -- Community resource list
