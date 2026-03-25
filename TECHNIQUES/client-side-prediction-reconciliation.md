# Client-Side Prediction & Server Reconciliation


---

## Table of Contents

1. [Overview](#1-overview)
2. [History & Origins](#2-history--origins)
3. [How It Works Step by Step](#3-how-it-works-step-by-step)
4. [The Prediction Loop](#4-the-prediction-loop)
5. [Server Processing](#5-server-processing)
6. [Reconciliation (Input Replay)](#6-reconciliation-input-replay)
7. [Smoothing & Corrections](#7-smoothing--corrections)
8. [Common Pitfalls](#8-common-pitfalls)
9. [Pseudo-Code / Code Examples](#9-pseudo-code--code-examples)
10. [Variations & Optimizations](#10-variations--optimizations)
11. [When to Use / When Not to Use](#11-when-to-use--when-not-to-use)
12. [References](#12-references)

---

## 1. Overview

### What It Is

Client-Side Prediction with Server Reconciliation (CSP+R) is a netcode technique that allows players to see the results of their own actions immediately, even though the game server -- which has final authority over all game state -- runs remotely across a network with non-trivial latency.

### The Problem It Solves

In a pure authoritative server model, the game loop works like this:

1. Client captures player input (e.g., "move forward")
2. Client sends the input to the server
3. Server receives and processes the input
4. Server sends the updated authoritative state back to the client
5. Client renders the new state

With a round-trip time (RTT) of 100 ms, every action the player takes appears to respond with at least 100 ms of delay. At 200 ms RTT (common for cross-continental connections), the game becomes sluggish and unplayable for any action-oriented genre.

Client-Side Prediction eliminates this perceived latency by letting the client simulate the effect of its own inputs immediately, without waiting for the server's response. Server Reconciliation is the complementary mechanism that corrects the client when its prediction diverges from the server's authoritative state -- without causing jarring visual snaps.

### The Core Insight

> "Since most inputs are valid and the game world is sufficiently deterministic, clients can immediately simulate input effects locally rather than waiting for server confirmation."
>
> -- Gabriel Gambetta, [Client-Side Prediction and Server Reconciliation](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)

The technique rests on two assumptions:

1. **The game simulation is sufficiently deterministic.** Given the same starting state and the same input, the client and server will produce the same (or very similar) resulting state.
2. **The server remains authoritative.** The client's prediction is always provisional. When the server disagrees, the server wins.

---

## 2. History & Origins

### Duke Nukem 3D (January 1996) -- The First Known Implementation

The earliest known first-person shooter to implement client-side prediction is **Duke Nukem 3D**, created by Ken Silverman using the Build engine. The technique was present in the January 29, 1996 shareware release.

Silverman confirmed the innovation in a 2018 Retro Gamer interview:

> "That's something I had come up with first and implemented in the January 1996 release of Duke 3D shareware."

The prediction functions are visible in the released source code: `domovethings()`, `fakedomovethings()`, and `fakedomovethingscorrect()`.

### QuakeWorld (August 1996) -- Popularization

John Carmack implemented client-side prediction in **QuakeWorld**, an update to Quake designed for internet play. In his `.plan` file on August 2, 1996, Carmack wrote:

> "I am now allowing the client to guess at the results of the user's movement until the authoritative response from the server comes through. This is a biiiig architectural change."

The original Quake was optimized for LAN play with low latency. QuakeWorld's prediction allowed players with 200+ ms ping to play competitively against players with 50 ms ping -- a transformative moment for online gaming.

QuakeWorld's implementation stored commands in a `frames` array indexed by `netchannel.outgoingsequence`. When server acknowledgments arrived, the client would replay unacknowledged commands forward from the last confirmed state, with full collision detection via `CL_PredictUsercmd`.

### Key Milestones

| Year | Milestone |
|------|-----------|
| 1996 | Duke Nukem 3D ships with client-side prediction (Build engine) |
| 1996 | QuakeWorld introduces CSP to the Quake engine, popularizing the technique |
| 2001 | Yahn Bernier (Valve) publishes ["Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization"](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization), formalizing CSP + lag compensation for Half-Life/Source Engine |
| 2004 | Glenn Fiedler publishes ["Networked Physics"](https://gafferongames.com/post/networked_physics_2004/), describing CSP with physics simulations and smoothing techniques |
| ~2012 | Unreal Engine's CharacterMovementComponent ships with built-in CSP+R via `FSavedMove_Character` and `FClientAdjustment` |
| 2017 | Timothy Ford (Blizzard) presents ["Overwatch Gameplay Architecture and Netcode"](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and) at GDC, detailing ECS-based prediction including predicted projectiles |
| 2017 | Jared Cone (Psyonix) presents "It IS Rocket Science!" at GDC, describing CSP with full physics scene replay at 120 Hz |

---

## 3. How It Works Step by Step

The full CSP+R loop involves the client, the network, and the server working together. Here is the complete cycle:

### The Full Loop

```
   CLIENT                          NETWORK                         SERVER
   ──────                          ───────                         ──────

1. Capture input (tick N)
2. Assign sequence number
3. Store input in local buffer
4. Simulate input locally           ──── send input ────►
   (predict new state)                                        5. Receive input
5. Render predicted state                                     6. Validate input
                                                              7. Simulate input
                                                                 (authoritative)
                                                              8. Record last processed
                                                                 sequence number
                                    ◄── send state + seq ──  9. Broadcast state

10. Receive server state
11. Set local state = server state
12. Discard inputs with seq <= acked seq
13. Re-simulate all remaining
    unacknowledged inputs
14. Render reconciled state
```

### Walkthrough

**Step 1-4 (Client Predicts):** The player presses a key. The client assigns a monotonically increasing sequence number, saves the input + resulting state into a buffer, sends the input to the server over UDP, and immediately applies the input to the local simulation. The player sees instant feedback.

**Step 5-9 (Server Processes):** The server receives the input (possibly out of order or with delay), validates it (anti-cheat, bounds checking), applies it to its authoritative copy of the game state, and broadcasts the resulting state to all clients along with the sequence number of the last input it processed for that client.

**Step 10-14 (Client Reconciles):** The client receives the server's authoritative state. It snaps its simulation to this state, discards all inputs that the server has already processed (sequence number <= acknowledged sequence), and then re-applies all inputs that the server hasn't seen yet. The result is a corrected state that accounts for both the server's authority and the client's recent inputs.

---

## 4. The Prediction Loop

### Input Collection

Each game tick, the client:

1. Samples raw input devices (keyboard, mouse, gamepad)
2. Constructs an **input command** (often called a "usercmd" in Valve/id terminology)
3. Assigns a **sequence number** (monotonically increasing integer)
4. Timestamps the command (local tick or wall-clock time)

A typical input command structure:

```typescript
interface InputCommand {
  sequenceNumber: number;       // Monotonically increasing ID
  tick: number;                 // The simulation tick this was generated on
  deltaTime: number;            // Time step for this tick

  // Movement
  moveForward: number;          // -1 to 1
  moveRight: number;            // -1 to 1
  jump: boolean;

  // Aiming
  yaw: number;                  // Horizontal look angle
  pitch: number;                // Vertical look angle

  // Actions
  fire: boolean;
  use: boolean;
}
```

### Local Simulation

After constructing the input command, the client runs the same simulation function the server uses:

```
newState = simulate(currentState, inputCommand, deltaTime)
```

This must be the **exact same function** that the server calls. Any divergence between the client and server simulation logic guarantees prediction errors.

### Input Buffer Management

The client maintains a circular buffer of recent input commands alongside their resulting predicted states:

```typescript
interface PendingInput {
  input: InputCommand;
  predictedState: PlayerState;  // State AFTER applying this input
}

// Ring buffer, typically sized for ~1-2 seconds of inputs
const pendingInputs: PendingInput[] = [];
```

The buffer serves two purposes:
1. **Retransmission:** Inputs may be re-sent in case of packet loss (many implementations pack the last N inputs into each packet)
2. **Reconciliation replay:** When the server confirms a state, unacknowledged inputs are replayed from the buffer

**Buffer sizing:** At 60 ticks/second with a maximum expected RTT of 500 ms, you need at minimum 30 entries. In practice, 64-128 entries (a power of 2 for efficient ring buffer indexing) is standard.

### Redundant Input Transmission

Because CSP systems typically use UDP (unreliable), individual input packets can be lost. A common mitigation is to include multiple inputs per packet:

```
Packet contents:
  - Input for tick 105 (latest)
  - Input for tick 104
  - Input for tick 103
  - Last acknowledged server tick: 98
```

The server ignores inputs it has already processed (by checking sequence numbers) and processes any it missed. This approach tolerates sporadic packet loss without requiring reliable delivery.

---

## 5. Server Processing

### Receiving and Validating Inputs

The server receives input commands from each client. For each command:

1. **Sequence check:** Discard if the sequence number is older than the last processed input for this client (duplicate or out-of-order late arrival).
2. **Timestamp validation:** Verify the command's timing is plausible. The Source Engine notably uses server-side timestamps rather than trusting client timestamps, preventing speed hacks.
3. **Input validation:** Bounds-check input values. Ensure movement values are within legal ranges. Check for impossible input combinations.
4. **Rate limiting:** Ensure the client isn't sending commands faster than the server's tick rate allows.

### Authoritative Simulation

The server runs the simulation step for each client's input:

```
serverState[clientId] = simulate(serverState[clientId], validatedInput, serverDeltaTime)
```

Key considerations:

- The server processes inputs **in order of their sequence numbers**, not in order of arrival.
- If inputs arrive out of order, the server may buffer briefly or skip gaps.
- The server must handle **missing inputs** gracefully -- typically by assuming the player's previous input continues (input extrapolation) or by applying no input.

### Delta Time Handling

A critical subtlety: the server and client must agree on the delta time used for each simulation step. Two approaches:

1. **Fixed tick rate (preferred):** Both client and server simulate at a fixed rate (e.g., 60 Hz = 16.67 ms per tick). This eliminates delta-time mismatches as a source of divergence.
2. **Client-reported delta time:** The client includes its delta time in each command. The server may use this value (Unreal Engine's approach) but must clamp it to prevent manipulation.

### State Broadcasting

After processing a tick, the server broadcasts to each client:

```typescript
interface ServerStateMessage {
  serverTick: number;                    // Current server tick
  lastProcessedInputSequence: number;    // Per-client: the last input seq# processed
  playerState: PlayerState;              // Authoritative state for this player
  // ... other player states, entity states, etc.
}
```

The `lastProcessedInputSequence` is the critical field that enables reconciliation -- it tells the client exactly which of its inputs the server has "seen."

### Tick Rate and Send Rate

These are often different:

| Parameter | Typical Values | Notes |
|-----------|---------------|-------|
| Server tick rate | 30-128 Hz | How often the server simulates |
| Server send rate | 20-64 Hz | How often the server sends state updates |
| Client tick rate | 60-128 Hz | How often the client simulates/predicts |
| Client send rate | 30-128 Hz | How often the client sends inputs |

Valve's Source Engine traditionally runs at 64 tick (competitive CS:GO at 128 tick). Overwatch runs at 63 tick. Rocket League simulates at 120 Hz.

---

## 6. Reconciliation (Input Replay)

Reconciliation is the mechanism that corrects the client's prediction when it diverges from the server's authoritative state, while preserving the responsiveness of inputs the server hasn't processed yet.

### The Core Algorithm

When the client receives an authoritative state update from the server:

```
function onServerStateReceived(serverState, lastProcessedSequence):
    // 1. SNAP to server state
    //    The server is always right. Accept its state unconditionally.
    localState = serverState

    // 2. DISCARD acknowledged inputs
    //    Remove all inputs from the pending buffer that the server
    //    has already processed.
    remove all entries from pendingInputs where
        entry.input.sequenceNumber <= lastProcessedSequence

    // 3. REPLAY unacknowledged inputs
    //    Re-simulate all remaining inputs on top of the server state.
    //    These are inputs the client has predicted locally but the
    //    server hasn't confirmed yet.
    for each entry in pendingInputs (in order):
        localState = simulate(localState, entry.input, entry.input.deltaTime)
        entry.predictedState = localState  // Update stored prediction

    // 4. The localState now represents the client's best estimate
    //    of the current state, incorporating server authority +
    //    pending local inputs.
```

### Why Replay Is Necessary

Without replay, reconciliation would "teleport" the player back to a past state. Consider:

- Client has predicted inputs 10, 11, 12, 13, 14 (is at position X=50)
- Server confirms state after input 10 (position X=10)
- Without replay: client snaps to X=10 (4 ticks of movement lost)
- With replay: client snaps to X=10, replays inputs 11-14, arrives at X=50 (or close to it)

If the simulation is deterministic and no other players interfered, the replayed state matches the original prediction exactly -- the player notices nothing. If another player or game event caused divergence, the replay produces a slightly different result, which is then smoothed visually (see Section 7).

### Detecting Mismatches

After reconciliation, you can measure the prediction error:

```typescript
const error = vectorDistance(reconciledState.position, previousPredictedPosition);
if (error > SNAP_THRESHOLD) {
  // Large error: snap immediately (e.g., teleport, death, knockback)
} else if (error > SMOOTH_THRESHOLD) {
  // Small error: apply visual smoothing (see Section 7)
} else {
  // Negligible error: no correction needed
}
```

Common threshold values:
- **SMOOTH_THRESHOLD:** 0.01 - 0.1 units (ignore errors below this)
- **SNAP_THRESHOLD:** 2.0 - 5.0 units (too large to smooth, just snap)

### How Different Engines Implement This

**Valve Source Engine:**
- Inputs are `CUserCmd` structures with sequence numbers
- Client stores predicted states per tick in a circular buffer
- On server update, client finds the matching tick, compares state, and replays forward
- Uses `cl_predict` and `cl_smooth` console variables for tuning
- Prediction errors are smoothed over `cl_smoothtime` seconds (default 0.1s)

**Unreal Engine (CharacterMovementComponent):**
- Client stores `FSavedMove_Character` structs in a `SavedMoves` buffer
- Moves are ordered oldest-to-newest; similar moves can be combined to reduce bandwidth
- Server processes `ServerMove` calls and computes the authoritative result
- If the server disagrees, it fills an `FClientAdjustment` (aka `PendingAdjustment`) with corrected position, rotation, velocity, and movement base
- If the move was correct, `PendingAdjustment.bAckGoodMove = true`
- Client replays all saved moves after the corrected timestamp

**Overwatch (ECS Architecture):**
- Uses an Entity Component System where prediction operates on components
- Clients predict not just movement but also abilities and slow-moving projectiles
- Timothy Ford described how they achieved "predicted rockets" -- something other studios considered impractical
- If client prediction conflicts with the server, the entire entity state rolls back to the server's version and replays forward
- The ECS architecture makes state serialization and rollback efficient

**Rocket League:**
- Predicts the entire physics scene, not just the local player
- Replays the full Bullet Physics simulation at 120 Hz
- Uses input decay for remote player prediction: 100% of last known input on frame 1, 67% on frame 2, 33% on frame 3, 0% on frame 4
- This undershooting approach reduces rubber-banding vs. overshooting

---

## 7. Smoothing & Corrections

Even with reconciliation, small prediction errors are inevitable in multiplayer environments. Raw snapping to corrected positions creates jarring visual artifacts. Smoothing techniques hide these corrections.

### The Visual Entity vs. Simulation Entity Pattern

A common architectural approach separates the **simulation state** (used for game logic) from the **visual state** (used for rendering):

```
Simulation entity: snaps immediately to reconciled position
Visual entity: smoothly interpolates toward simulation entity
```

The player "controls" the simulation entity, but the camera and rendered model follow the visual entity. This decouples the accuracy of gameplay from the smoothness of rendering.

### Technique 1: Error Offset Decay

When a correction occurs:

1. Calculate the error vector: `errorOffset = previousVisualPosition - reconciledPosition`
2. Each frame, decay the offset: `errorOffset *= (1 - decayRate * deltaTime)` or `errorOffset = lerp(errorOffset, ZERO, decayFactor)`
3. Render at: `visualPosition = simulationPosition + errorOffset`

```typescript
// On reconciliation:
const correctionError = vec3Subtract(oldPredictedPos, newReconciledPos);
visualOffset = vec3Add(visualOffset, correctionError);

// Every render frame:
const DECAY_RATE = 10.0; // Higher = faster correction
visualOffset = vec3Scale(visualOffset, Math.exp(-DECAY_RATE * deltaTime));
const renderPosition = vec3Add(simulationPosition, visualOffset);
```

This approach, described by Glenn Fiedler, produces exponentially decaying corrections that are nearly invisible for small errors.

### Technique 2: Distance-Based Correction (Gaffer on Games)

Glenn Fiedler's networked physics implementation uses distance-based thresholds:

| Distance | Behavior |
|----------|----------|
| < 0.1 units | Ignore (within tolerance) |
| 0.1 - 2.0 units | Move 10% toward correct position per update |
| > 2.0 units | Snap immediately |

For derivative quantities (velocity, angular velocity), snap directly rather than smoothing -- abrupt velocity changes are less visually noticeable than position jumps.

### Technique 3: Interpolation Over Time

Instead of exponential decay, interpolate from the error position to the correct position over a fixed duration:

```typescript
const SMOOTH_DURATION = 0.1; // 100ms (Valve's cl_smoothtime default)
let smoothTimer = 0;
let smoothStartOffset = vec3Zero();

function onCorrectionDetected(error: Vec3) {
  smoothStartOffset = error;
  smoothTimer = SMOOTH_DURATION;
}

function updateVisual(dt: number): Vec3 {
  if (smoothTimer > 0) {
    smoothTimer = Math.max(0, smoothTimer - dt);
    const t = 1 - (smoothTimer / SMOOTH_DURATION);
    const offset = vec3Lerp(smoothStartOffset, vec3Zero(), t);
    return vec3Add(simulationPosition, offset);
  }
  return simulationPosition;
}
```

### Technique 4: Spherical Interpolation for Rotations

Position corrections use linear interpolation (lerp), but rotation corrections should use **spherical linear interpolation (slerp)** to avoid distortion:

```typescript
visualRotation = quaternionSlerp(visualRotation, simulationRotation, correctionFactor);
```

### Combining Techniques

In practice, production games layer multiple approaches:

1. **Immediate snap** for large corrections (teleport, respawn, knockback)
2. **Error offset decay** for small position corrections during normal play
3. **Slerp** for rotation corrections
4. **Velocity snap** (no smoothing on velocity -- only position is smoothed visually)
5. **Camera smoothing** as an additional layer on top of entity smoothing

---

## 8. Common Pitfalls

### 8.1 Floating-Point Non-Determinism

The most insidious source of prediction errors. Even with identical code, floating-point arithmetic can produce different results due to:

- **Compiler differences:** Debug vs. release builds may optimize floating-point operations differently
- **CPU architecture:** AMD and Intel processors compute transcendental functions (sin, cos, tan) with slightly different precision
- **FPU precision modes:** x87 uses 80-bit internal precision; SSE/SSE2 uses 32/64-bit. Results differ.
- **Instruction reordering:** `(a + b) + c != a + (b + c)` in floating point; compilers may reorder

**Mitigation strategies:**
- Use fixed-point arithmetic for critical gameplay calculations
- Enforce FPU control settings at startup (Gas Powered Games' approach for Supreme Commander):
  ```c
  _controlfp(_PC_24, _MCW_PC);  // Single precision
  _controlfp(_RC_NEAR, _MCW_RC); // Round to nearest
  ```
- Use `/fp:strict` (MSVC) or `-ffp-model=strict` (GCC/Clang) to disable dangerous optimizations
- Ensure client and server use the same compiler, platform, and build configuration when possible
- Wrap transcendental functions in non-optimized calls to force consistency (Pandemic Studios' approach)

For CSP+R specifically, perfect determinism is not strictly required (unlike lockstep) because the server sends corrections. However, lower divergence means fewer visible corrections, so minimizing floating-point drift is still important.

### 8.2 Physics Engine Divergence

Most physics engines (PhysX, Bullet, Box2D) are **not deterministic** across platforms or even between runs. Applying force on a client produces slightly different results than on the server, and differences compound rapidly.

**Mitigation:**
- Use simple, custom physics for predicted movement (AABB collision, no broadphase randomness)
- Reserve the full physics engine for server-side simulation only
- If predicting physics (like Rocket League), accept frequent corrections and use aggressive smoothing
- Rocket League runs physics at 120 Hz and replays the entire scene -- an expensive but effective approach

### 8.3 Delta Time Mismatches

If the client and server use different delta times for simulation, the results will diverge even with identical code.

**Mitigation:**
- Use a **fixed timestep** on both client and server (e.g., exactly 16.667 ms at 60 Hz)
- If variable timestep is necessary, include the delta time in the input command and have the server use the client's reported value (with validation/clamping)
- Never accumulate time differently on client vs. server

### 8.4 Input Buffer Overflow

If the client sends inputs faster than the server processes them (or network latency spikes), the server's input buffer can overflow, causing dropped inputs and severe desync.

**Mitigation:**
- Size the input buffer generously (2-3x the expected maximum RTT in ticks)
- Implement input throttling on the client when the server signals it's falling behind
- Use adaptive send rates based on network conditions

### 8.5 Jitter and Packet Reordering

Network jitter causes server state updates to arrive at irregular intervals, making reconciliation timing unpredictable.

**Mitigation:**
- Use a jitter buffer on the client to smooth out arrival times
- Timestamp all messages and process them based on tick numbers, not arrival order
- Discard state updates that are older than the most recent one processed

### 8.6 Non-Predicted Game Events

Prediction only works for events the client can locally simulate. Unpredictable events cause forced corrections:

- Another player blocking your path
- Server-side random number generation (damage rolls, spread patterns)
- Environmental changes triggered by other players
- Abilities that depend on other players' states

**Mitigation:**
- Minimize server-side randomness in predicted systems (use seed-based RNG synchronized between client and server)
- Accept that some corrections are unavoidable in multiplayer scenarios
- Use generous smoothing for corrections caused by other players' actions

### 8.7 State Divergence Spiral

If the client and server diverge significantly, each reconciliation produces a large correction, which can compound if the replayed inputs also diverge.

**Mitigation:**
- Set a maximum correction distance; beyond it, hard-snap and clear the input buffer
- Monitor prediction error magnitude over time; if it exceeds thresholds, reset the prediction system
- Log prediction errors during development to identify systematic divergence sources

---

## 9. Pseudo-Code / Code Examples

### 9.1 Pseudo-Code: The Complete Loop

```
=== CLIENT TICK ===

function clientTick(dt):
    // 1. Collect input
    input = sampleInput()
    input.sequenceNumber = nextSequenceNumber++
    input.deltaTime = dt

    // 2. Send to server
    network.send({
        type: "INPUT",
        inputs: [input],           // Could include redundant older inputs
        lastReceivedServerTick: lastServerTick
    })

    // 3. Predict locally
    playerState = simulate(playerState, input, dt)

    // 4. Store in pending buffer
    pendingInputs.push({
        input: input,
        predictedState: clone(playerState)
    })

    // 5. Render
    render(applyVisualSmoothing(playerState))


function onServerStateReceived(message):
    serverState = message.playerState
    ackedSequence = message.lastProcessedInputSequence

    // Store old predicted position for smoothing
    oldPosition = playerState.position

    // 1. Apply authoritative state
    playerState = serverState

    // 2. Remove acknowledged inputs
    while pendingInputs.length > 0 AND
          pendingInputs[0].input.sequenceNumber <= ackedSequence:
        pendingInputs.removeFirst()

    // 3. Replay unacknowledged inputs
    for each entry in pendingInputs:
        playerState = simulate(playerState, entry.input, entry.input.deltaTime)
        entry.predictedState = clone(playerState)

    // 4. Calculate visual correction offset
    correctionError = oldPosition - playerState.position
    addVisualSmoothing(correctionError)


=== SERVER TICK ===

function serverTick(dt):
    // Process inputs from all clients
    for each client in connectedClients:
        while client.inputQueue has unprocessed inputs:
            input = client.inputQueue.dequeue()

            // Validate
            if not isValidInput(input): continue
            if input.sequenceNumber <= client.lastProcessedSequence: continue

            // Simulate authoritatively
            client.playerState = simulate(client.playerState, input, dt)
            client.lastProcessedSequence = input.sequenceNumber

    // Resolve interactions (collisions between players, etc.)
    resolveInteractions()

    // Broadcast state to all clients
    for each client in connectedClients:
        network.sendTo(client, {
            type: "STATE",
            serverTick: currentTick,
            lastProcessedInputSequence: client.lastProcessedSequence,
            playerState: client.playerState,
            otherPlayers: getVisiblePlayersFor(client)
        })
```

### 9.2 TypeScript Implementation: Full Prediction + Reconciliation Loop

The following is a practical, runnable TypeScript implementation demonstrating the full CSP+R system. It models a simple 2D top-down movement game.

```typescript
// ============================================================
// Types
// ============================================================

interface Vec2 {
  x: number;
  y: number;
}

interface InputCommand {
  sequenceNumber: number;
  tick: number;
  deltaTime: number;
  moveX: number;   // -1, 0, or 1
  moveY: number;   // -1, 0, or 1
}

interface PlayerState {
  position: Vec2;
  velocity: Vec2;
}

interface ServerStateMessage {
  serverTick: number;
  lastProcessedInputSequence: number;
  playerState: PlayerState;
}

interface PendingInput {
  input: InputCommand;
  predictedState: PlayerState;
}

// ============================================================
// Shared Simulation (identical on client and server)
// ============================================================

const MOVE_SPEED = 200;       // units per second
const FRICTION = 10;           // deceleration factor

function simulate(state: PlayerState, input: InputCommand, dt: number): PlayerState {
  // Calculate desired velocity from input
  const desiredVelX = input.moveX * MOVE_SPEED;
  const desiredVelY = input.moveY * MOVE_SPEED;

  // Apply acceleration toward desired velocity with friction
  const frictionFactor = Math.min(1, FRICTION * dt);
  const newVelX = state.velocity.x + (desiredVelX - state.velocity.x) * frictionFactor;
  const newVelY = state.velocity.y + (desiredVelY - state.velocity.y) * frictionFactor;

  // Integrate position
  const newPosX = state.position.x + newVelX * dt;
  const newPosY = state.position.y + newVelY * dt;

  // Clamp to world bounds [0, 1000]
  const clampedPosX = Math.max(0, Math.min(1000, newPosX));
  const clampedPosY = Math.max(0, Math.min(1000, newPosY));

  return {
    position: { x: clampedPosX, y: clampedPosY },
    velocity: { x: newVelX, y: newVelY },
  };
}

function cloneState(s: PlayerState): PlayerState {
  return {
    position: { x: s.position.x, y: s.position.y },
    velocity: { x: s.velocity.x, y: s.velocity.y },
  };
}

function vec2Distance(a: Vec2, b: Vec2): number {
  const dx = a.x - b.x;
  const dy = a.y - b.y;
  return Math.sqrt(dx * dx + dy * dy);
}

// ============================================================
// Client
// ============================================================

class Client {
  // Simulation state
  private playerState: PlayerState = {
    position: { x: 500, y: 500 },
    velocity: { x: 0, y: 0 },
  };

  // Input tracking
  private nextSequenceNumber = 1;
  private currentTick = 0;
  private pendingInputs: PendingInput[] = [];

  // Visual smoothing
  private visualOffset: Vec2 = { x: 0, y: 0 };
  private readonly VISUAL_DECAY_RATE = 10; // per second

  // Thresholds
  private readonly SMOOTH_THRESHOLD = 0.01;
  private readonly SNAP_THRESHOLD = 50;

  // Network interface (to be wired to server)
  public onSendInput: ((input: InputCommand) => void) | null = null;

  /**
   * Called every client tick. Captures input, predicts, and stores.
   */
  tick(dt: number, moveX: number, moveY: number): void {
    this.currentTick++;

    // 1. Build the input command
    const input: InputCommand = {
      sequenceNumber: this.nextSequenceNumber++,
      tick: this.currentTick,
      deltaTime: dt,
      moveX,
      moveY,
    };

    // 2. Send input to server
    if (this.onSendInput) {
      this.onSendInput(input);
    }

    // 3. Predict locally using the shared simulation
    this.playerState = simulate(this.playerState, input, dt);

    // 4. Store input and predicted state for reconciliation
    this.pendingInputs.push({
      input,
      predictedState: cloneState(this.playerState),
    });
  }

  /**
   * Called when an authoritative state update arrives from the server.
   */
  onServerState(message: ServerStateMessage): void {
    const previousPosition = {
      x: this.playerState.position.x,
      y: this.playerState.position.y,
    };

    // Step 1: Accept the server's authoritative state
    this.playerState = cloneState(message.playerState);

    // Step 2: Discard inputs the server has already processed
    this.pendingInputs = this.pendingInputs.filter(
      (entry) => entry.input.sequenceNumber > message.lastProcessedInputSequence
    );

    // Step 3: Re-apply all unacknowledged inputs
    for (const entry of this.pendingInputs) {
      this.playerState = simulate(
        this.playerState,
        entry.input,
        entry.input.deltaTime
      );
      entry.predictedState = cloneState(this.playerState);
    }

    // Step 4: Calculate prediction error for visual smoothing
    const error = vec2Distance(previousPosition, this.playerState.position);

    if (error > this.SNAP_THRESHOLD) {
      // Large error: snap immediately, clear visual offset
      this.visualOffset = { x: 0, y: 0 };
    } else if (error > this.SMOOTH_THRESHOLD) {
      // Small error: add to visual offset for smooth correction
      this.visualOffset.x += previousPosition.x - this.playerState.position.x;
      this.visualOffset.y += previousPosition.y - this.playerState.position.y;
    }
    // Below SMOOTH_THRESHOLD: no visible correction needed
  }

  /**
   * Called every render frame. Returns the position to draw at.
   * Decays the visual offset for smooth corrections.
   */
  getRenderPosition(renderDt: number): Vec2 {
    // Exponential decay of visual offset
    const decayFactor = Math.exp(-this.VISUAL_DECAY_RATE * renderDt);
    this.visualOffset.x *= decayFactor;
    this.visualOffset.y *= decayFactor;

    // Snap to zero if negligible
    if (Math.abs(this.visualOffset.x) < 0.001) this.visualOffset.x = 0;
    if (Math.abs(this.visualOffset.y) < 0.001) this.visualOffset.y = 0;

    return {
      x: this.playerState.position.x + this.visualOffset.x,
      y: this.playerState.position.y + this.visualOffset.y,
    };
  }

  /** Get the current simulation state (for debugging). */
  getState(): PlayerState {
    return cloneState(this.playerState);
  }

  /** Get number of pending (unacknowledged) inputs. */
  getPendingInputCount(): number {
    return this.pendingInputs.length;
  }
}

// ============================================================
// Server
// ============================================================

interface ServerClient {
  id: string;
  playerState: PlayerState;
  lastProcessedSequence: number;
  inputQueue: InputCommand[];
}

class Server {
  private clients: Map<string, ServerClient> = new Map();
  private currentTick = 0;

  // Network interface (to be wired to clients)
  public onSendState: ((clientId: string, message: ServerStateMessage) => void) | null = null;

  /**
   * Register a new client.
   */
  addClient(id: string, initialPosition: Vec2): void {
    this.clients.set(id, {
      id,
      playerState: {
        position: { ...initialPosition },
        velocity: { x: 0, y: 0 },
      },
      lastProcessedSequence: 0,
      inputQueue: [],
    });
  }

  /**
   * Called when an input arrives from a client.
   */
  onClientInput(clientId: string, input: InputCommand): void {
    const client = this.clients.get(clientId);
    if (!client) return;

    // Discard old/duplicate inputs
    if (input.sequenceNumber <= client.lastProcessedSequence) return;

    // Insert into queue (maintain order by sequence number)
    client.inputQueue.push(input);
    client.inputQueue.sort((a, b) => a.sequenceNumber - b.sequenceNumber);
  }

  /**
   * Called every server tick. Processes all queued inputs and broadcasts state.
   */
  tick(dt: number): void {
    this.currentTick++;

    // Process inputs for each client
    for (const [clientId, client] of this.clients) {
      while (client.inputQueue.length > 0) {
        const input = client.inputQueue[0];

        // Skip if out of order (shouldn't happen after sort, but safety check)
        if (input.sequenceNumber <= client.lastProcessedSequence) {
          client.inputQueue.shift();
          continue;
        }

        // Process this input
        client.playerState = simulate(client.playerState, input, dt);
        client.lastProcessedSequence = input.sequenceNumber;
        client.inputQueue.shift();
      }
    }

    // Broadcast authoritative state to all clients
    for (const [clientId, client] of this.clients) {
      if (this.onSendState) {
        this.onSendState(clientId, {
          serverTick: this.currentTick,
          lastProcessedInputSequence: client.lastProcessedSequence,
          playerState: cloneState(client.playerState),
        });
      }
    }
  }
}

// ============================================================
// Example Usage: Simulating a Network Loop
// ============================================================

function runSimulation(): void {
  const TICK_RATE = 60;
  const DT = 1 / TICK_RATE;
  const SIMULATED_LATENCY_MS = 100;  // One-way latency
  const SIMULATED_LATENCY_TICKS = Math.ceil((SIMULATED_LATENCY_MS / 1000) * TICK_RATE);

  const server = new Server();
  const client = new Client();

  server.addClient("player1", { x: 500, y: 500 });

  // Simulated network: delayed message queues
  const clientToServerQueue: { deliverAtTick: number; input: InputCommand }[] = [];
  const serverToClientQueue: { deliverAtTick: number; message: ServerStateMessage }[] = [];

  let globalTick = 0;

  // Wire up network simulation
  client.onSendInput = (input) => {
    clientToServerQueue.push({
      deliverAtTick: globalTick + SIMULATED_LATENCY_TICKS,
      input,
    });
  };

  server.onSendState = (clientId, message) => {
    serverToClientQueue.push({
      deliverAtTick: globalTick + SIMULATED_LATENCY_TICKS,
      message,
    });
  };

  // Simulate 3 seconds of gameplay: move right for 1s, then up for 1s, then stop
  const TOTAL_TICKS = TICK_RATE * 3;

  for (globalTick = 0; globalTick < TOTAL_TICKS; globalTick++) {
    // Determine input based on time
    let moveX = 0;
    let moveY = 0;
    if (globalTick < TICK_RATE) {
      moveX = 1; // Move right for first second
    } else if (globalTick < TICK_RATE * 2) {
      moveY = -1; // Move up for second second
    }
    // Third second: no input (stop)

    // Deliver delayed messages from server to client
    while (serverToClientQueue.length > 0 &&
           serverToClientQueue[0].deliverAtTick <= globalTick) {
      client.onServerState(serverToClientQueue.shift()!.message);
    }

    // Deliver delayed messages from client to server
    while (clientToServerQueue.length > 0 &&
           clientToServerQueue[0].deliverAtTick <= globalTick) {
      server.onClientInput("player1", clientToServerQueue.shift()!.input);
    }

    // Run ticks
    client.tick(DT, moveX, moveY);
    server.tick(DT);

    // Log periodically
    if (globalTick % 30 === 0) {
      const renderPos = client.getRenderPosition(DT);
      console.log(
        `Tick ${globalTick}: ` +
        `render=(${renderPos.x.toFixed(1)}, ${renderPos.y.toFixed(1)}) ` +
        `pending=${client.getPendingInputCount()}`
      );
    }
  }
}

runSimulation();
```

### 9.3 Key Implementation Notes

**On the `simulate` function:** This function MUST be identical on client and server. Any difference -- even a different clamping order or a different constant -- will cause persistent prediction errors. In production, this function is typically shared code compiled into both the client and server binaries.

**On sequence numbers:** Use a monotonically increasing integer (not a timestamp). This avoids issues with clock synchronization and makes comparison trivial.

**On the input buffer:** The `filter()` call in `onServerState` is O(n) but n is small (typically < 30). For performance-critical implementations, use a ring buffer with a head pointer that advances as inputs are acknowledged.

**On visual smoothing:** The `getRenderPosition` function should be called at the rendering frame rate (potentially higher than the simulation tick rate), not at the tick rate. This ensures smooth visual interpolation independent of simulation frequency.

---

## 10. Variations & Optimizations

### 10.1 Delta Compression

Instead of sending the full player state every tick, the server sends only what changed since the last acknowledged state:

```
Full state:  position(12 bytes) + velocity(12 bytes) + rotation(16 bytes) = 40 bytes
Delta state: "position.x changed by +0.5" = 5 bytes
```

The Source Engine implements this by maintaining per-client baselines. Each state update is a delta from the last acknowledged snapshot. If a packet is lost, the server falls back to the last acknowledged baseline.

Glenn Fiedler's snapshot compression techniques achieve ~98.5% bandwidth reduction through layered compression:
1. Quaternion "smallest three" encoding (128 bits -> 29 bits)
2. Position quantization within world bounds
3. Delta encoding from acknowledged baselines
4. Relative index encoding for changed entities
5. Per-component small/large delta ranges

### 10.2 Input Redundancy

Pack the last N inputs into each outgoing packet:

```typescript
interface ClientPacket {
  inputs: InputCommand[];      // Most recent N inputs (newest first)
  lastReceivedServerTick: number;
}
```

This tolerates sporadic packet loss without requiring reliable delivery or retransmission. The server discards any inputs it has already processed (by checking sequence numbers).

Typical redundancy: 3-5 inputs per packet. At 60 Hz with 8-byte inputs, this adds only 24-40 bytes of overhead per packet.

### 10.3 Priority Systems

Not all entities need the same update frequency. Priority-based bandwidth allocation:

| Priority | Update Rate | Examples |
|----------|------------|---------|
| Critical | Every tick | Local player, nearby enemies |
| High | Every 2-3 ticks | Nearby projectiles, interactive objects |
| Medium | Every 5-10 ticks | Distant players, vehicles |
| Low | Every 20+ ticks | Distant scenery, ambient entities |

The server maintains a priority accumulator per entity per client. Each tick, priority accrues based on distance, relevance, and time since last update. Entities are packed into outgoing packets in priority order until the bandwidth budget is exhausted.

### 10.4 Partial Prediction

Not everything needs to be predicted. A hybrid approach:

- **Predicted:** Player movement, weapon firing, ability activation, jump
- **Interpolated:** Remote player positions, NPC movement
- **Server-authoritative only:** Health changes, inventory, score, matchmaking state

This reduces prediction complexity and limits the surface area for visible corrections.

### 10.5 Move Combining (Unreal Engine)

Unreal's CharacterMovementComponent combines "similar" saved moves to reduce bandwidth. If two consecutive moves have the same input direction and similar velocity, they're merged into a single move with a combined delta time. This is particularly effective for sustained movement in a single direction.

### 10.6 Adaptive Tick Rates

Some implementations dynamically adjust the client's send rate based on network conditions:

- **Good connection (< 50 ms RTT):** Send at full tick rate (60 Hz)
- **Moderate connection (50-150 ms RTT):** Reduce to 30 Hz with input redundancy
- **Poor connection (> 150 ms RTT):** Reduce to 20 Hz with higher redundancy (5-8 inputs per packet)

### 10.7 Prediction of Non-Local Entities

While classic CSP only predicts the local player, some games extend prediction:

- **Rocket League:** Predicts all cars and the ball using full physics simulation
- **Overwatch:** Predicts slow-moving projectiles (rockets, orbs) on the firing client
- **Fighting games (GGPO/rollback):** Predicts all players, assuming they repeat their last input

Predicting non-local entities increases CPU cost (more to simulate) but reduces perceived latency for fast-paced interactions.

---

## 11. When to Use / When Not to Use

### Strong Fit

| Genre / Game Type | Why CSP+R Works Well |
|-------------------|---------------------|
| **First-Person Shooters** | Player movement must feel instant; small corrections are invisible from first-person view |
| **Third-Person Action** | Movement responsiveness is critical; camera perspective hides small corrections |
| **Battle Royale** | Large player counts require server authority; CSP masks the resulting latency |
| **Sports Games** | Ball and player control must feel responsive; physics prediction handles ball movement |
| **VR Multiplayer** | Even 50 ms of input latency causes motion sickness; CSP is essential |
| **Fighting Games** (variant) | Rollback netcode is a specialized form of CSP; GGPO-style prediction is the gold standard |

### Moderate Fit

| Genre / Game Type | Considerations |
|-------------------|---------------|
| **MMORPGs** | CSP for movement is standard (World of Warcraft uses it); ability prediction is game-dependent |
| **Platformers** | Works well for movement; tricky for tight collision-dependent mechanics (pixel-perfect jumps) |
| **Racing Games** | Good for local car; challenging for remote cars and collision interactions |

### Poor Fit / Unnecessary

| Genre / Game Type | Why CSP+R Is Overkill or Inappropriate |
|-------------------|---------------------------------------|
| **Turn-Based Games** | No real-time input to predict; server authority with confirmations is sufficient |
| **Card Games** | Actions are discrete, not continuous; 100 ms delay is imperceptible |
| **Real-Time Strategy** | Deterministic lockstep is preferred; commanding many units is better served by synchronized simulation than per-unit prediction |
| **Puzzle Games** | Typically cooperative or single-player; network model is simpler |

### Trade-Offs Summary

| Advantage | Disadvantage |
|-----------|-------------|
| Eliminates perceived input latency | Adds implementation complexity |
| Server remains authoritative (anti-cheat) | Corrections can cause visible "rubber-banding" |
| Works over high-latency connections | Requires identical simulation on client and server |
| Bandwidth-efficient (sends inputs, not states) | CPU cost of input replay during reconciliation |
| Battle-tested in production (20+ years) | Floating-point determinism issues across platforms |
| Compatible with lag compensation | Not suitable for all game genres |

### Decision Framework

Ask these questions:

1. **Is the game real-time with continuous player input?** If no, CSP is probably unnecessary.
2. **Does input latency of 100+ ms significantly harm the experience?** If no, a simpler model (pure server authority with interpolation) may suffice.
3. **Is server authority required (competitive, anti-cheat)?** If yes, CSP+R is the standard approach. If not, consider distributed authority or client authority.
4. **Can the core simulation be made deterministic enough?** If the game relies heavily on complex physics that can't be reliably reproduced on the client, CSP corrections will be frequent and visible.
5. **Is the CPU budget available for input replay?** At high tick rates with many predicted entities, reconciliation replay can be expensive. Budget 1-2 ms per reconciliation pass.

---

## 12. References

### Primary Sources

- **Gabriel Gambetta** -- [Fast-Paced Multiplayer (Part II): Client-Side Prediction and Server Reconciliation](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html) -- The most widely-cited tutorial on CSP+R. Clear walkthrough with diagrams.
- **Gabriel Gambetta** -- [Fast-Paced Multiplayer (Part I): Client-Server Game Architecture](https://www.gabrielgambetta.com/client-server-game-architecture.html) -- Foundational overview of the authoritative server model.
- **Gabriel Gambetta** -- [Fast-Paced Multiplayer (Part III): Entity Interpolation](https://www.gabrielgambetta.com/entity-interpolation.html) -- How remote entities are rendered alongside predicted local entities.
- **Gabriel Gambetta** -- [Fast-Paced Multiplayer (Part IV): Lag Compensation](https://www.gabrielgambetta.com/lag-compensation.html) -- Server-side rewinding for hit detection.
- **Glenn Fiedler (Gaffer on Games)** -- [Networked Physics (2004)](https://gafferongames.com/post/networked_physics_2004/) -- Detailed treatment of CSP with physics, including circular buffer architecture and distance-based correction smoothing.
- **Glenn Fiedler** -- [Floating Point Determinism](https://gafferongames.com/post/floating_point_determinism/) -- Deep dive into why floating-point math breaks determinism and practical workarounds.
- **Glenn Fiedler** -- [Snapshot Interpolation](https://gafferongames.com/post/snapshot_interpolation/) -- Alternative to CSP for non-player entities; interpolation buffers and jitter handling.
- **Glenn Fiedler** -- [Snapshot Compression](https://gafferongames.com/post/snapshot_compression/) -- Delta encoding, quantization, and bandwidth optimization techniques.
- **Glenn Fiedler** -- [Choosing the Right Network Model for Your Multiplayer Game](https://mas-bandwidth.com/choosing-the-right-network-model-for-your-multiplayer-game/) -- Comprehensive comparison of network architectures with genre recommendations.
- **Yahn Bernier (Valve)** -- [Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization) -- The foundational Valve paper on Source Engine networking, covering prediction, lag compensation, and entity interpolation.
- **Valve Developer Community** -- [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking) -- Technical documentation for Source Engine's networking layer.
- **Valve Developer Community** -- [Prediction](https://developer.valvesoftware.com/wiki/Prediction) -- Source Engine prediction system specifics.

### GDC Talks

- **Timothy Ford (Blizzard)** -- ["Overwatch Gameplay Architecture and Netcode" (GDC 2017)](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and) -- ECS-based prediction, predicted projectiles, and reconciliation in Overwatch.
- **Jared Cone (Psyonix)** -- "It IS Rocket Science!" (GDC) -- Full physics scene prediction and replay at 120 Hz in Rocket League, including input decay for remote player prediction.

### Historical Sources

- **Nition** -- [The Origin of Client-Side Prediction in Games](https://nition.momentstudio.co.nz/2020/01/the-origin-of-client-side-prediction/) -- Research into Duke Nukem 3D as the first known implementation, with Ken Silverman's confirmation.
- **John Carmack** -- [QuakeWorld Development Log (August 1996)](https://fabiensanglard.net/quakeSource/johnc-log.aug.htm) -- Carmack's .plan file announcing client-side prediction in QuakeWorld.
- **Fabien Sanglard** -- [Quake Engine Code Review: Prediction](https://fabiensanglard.net/quakeSource/quakeSourcePrediction.php) -- Technical analysis of QuakeWorld's prediction implementation in the source code.

### Engine Documentation

- **Epic Games** -- [Understanding Networked Movement in the Character Movement Component](https://dev.epicgames.com/documentation/en-us/unreal-engine/understanding-networked-movement-in-the-character-movement-component-for-unreal-engine) -- Unreal Engine's built-in CSP+R via FSavedMove_Character and FClientAdjustment.

### Community Resources

- **SnapNet** -- [Netcode Architectures Part 2: Rollback](https://www.snapnet.dev/blog/netcode-architectures-part-2-rollback/) -- Comparison of rollback netcode (a CSP variant) with implementation details and performance analysis.
- **GameDev.net Forums** -- [Smoothing Corrections to Client-Side Prediction](https://www.gamedev.net/forums/topic/658931-smoothing-corrections-to-client-side-prediction/) -- Community discussion of visual smoothing approaches.
- **CrystalOrb** -- [GitHub Repository](https://github.com/ErnWong/crystalorb) -- Network-agnostic Rust library for CSP+R with unconditional rollback, useful as a reference implementation.
