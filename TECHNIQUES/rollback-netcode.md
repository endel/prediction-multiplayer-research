# Rollback Netcode (GGPO-Style): A General Technique Deep-Dive

> **Disclaimer:** This content has been "vibe-researched" -- it was generated with the assistance of AI agents performing web research and synthesis. While efforts were made to fetch and cross-reference primary sources, details may be inaccurate, outdated, or misrepresented. Always verify against the original references linked throughout before relying on any technical claims.

> **See also:** For GGPO-specific API details, fighting game history, and the GGPO library in particular, see [GGPO & Rollback in Fighting Games](../ggpo-rollback.md).

## Table of Contents

1. [Overview](#overview)
2. [Rollback vs CSP+Reconciliation](#rollback-vs-cspreconciliation)
3. [How It Works Step by Step](#how-it-works-step-by-step)
4. [State Management](#state-management)
5. [Input Prediction Strategies](#input-prediction-strategies)
6. [Resimulation](#resimulation)
7. [Determinism Requirements](#determinism-requirements)
8. [Visual Artifacts & Mitigation](#visual-artifacts--mitigation)
9. [Audio Handling During Rollback](#audio-handling-during-rollback)
10. [Beyond Fighting Games](#beyond-fighting-games)
11. [Pseudo-Code / Code Examples](#pseudo-code--code-examples)
12. [Performance Considerations](#performance-considerations)
13. [Framework Implementations](#framework-implementations)
14. [When to Use / When Not to Use](#when-to-use--when-not-to-use)
15. [References](#references)

---

## Overview

Rollback netcode is a networking technique for real-time multiplayer games that eliminates perceived input latency by speculatively executing game simulation before remote player inputs have arrived. When those inputs do arrive and differ from what was predicted, the game "rolls back" to the last known-correct state and resimulates forward to the present using the corrected inputs -- all within the span of a single rendered frame.

The technique was popularized by Tony Cannon's [GGPO library](https://github.com/pond3r/ggpo) (2006/2009), originally designed for fighting games, but the underlying principles apply to any genre that can meet the architectural requirements: deterministic simulation, fast state serialization, and decoupled game logic from rendering.

### The Core Trade-Off

Every netcode architecture must choose how to distribute the pain of latency:

| Architecture | Where Latency is Felt | Failure Mode |
|---|---|---|
| **Delay-based (lockstep)** | Every frame feels sluggish | Freezes on packet loss |
| **Rollback** | Occasional visual corrections | Teleporting / animation pops on misprediction |
| **CSP + Reconciliation** | Remote entities may snap | Local player may snap on correction |

Rollback's fundamental insight is that **making some frames wrong is better than making every frame late**. Players overwhelmingly prefer occasional visual corrections over constant input delay, especially in competitive contexts.

### When People Say "Rollback"

The term "rollback netcode" is used to describe several related but distinct patterns:

- **Peer-to-peer rollback (GGPO-style):** Each peer runs a local simulation, predicts remote inputs, and rolls back on misprediction. No server authority -- both peers are equal. This is the classic model for fighting games.
- **Client-server rollback with prediction:** A server is authoritative but clients predict locally and roll back when the server's authoritative state diverges. Rocket League and Photon Fusion use this model.
- **Unconditional rollback:** Every time a server snapshot arrives, the client rolls back to that snapshot's timestamp and resimulates forward, regardless of whether there was a misprediction. CrystalOrb uses this approach.

This document covers rollback as a general technique across all these variants.

---

## Rollback vs CSP+Reconciliation

Rollback netcode and Client-Side Prediction with Server Reconciliation (CSP+R) are often conflated because they share the same core mechanic: predict locally, correct when authoritative data arrives, replay to catch up. But they differ in important architectural ways.

### Structural Differences

| Aspect | Rollback (GGPO-style) | CSP + Reconciliation (Valve/Overwatch-style) |
|---|---|---|
| **Topology** | Typically peer-to-peer | Client-server (server authoritative) |
| **What is predicted** | Remote player inputs | Local player's own state (before server confirms) |
| **What triggers correction** | Remote input differs from prediction | Server state snapshot differs from local prediction |
| **Authority** | All peers are equal (or virtual server) | Server is authoritative |
| **What is synced over network** | Inputs only | State snapshots (full or delta) |
| **Bandwidth** | Very low (inputs are tiny) | Medium-high (state snapshots) |
| **Determinism required** | Yes (strict) | No (server is the source of truth) |
| **Remote entities** | Predicted (in the present) | Interpolated (in the past) |
| **Typical genres** | Fighting games, competitive action | FPS, third-person shooters, action games |

### The Key Conceptual Difference

In **CSP+Reconciliation**, the local player is predicting *their own* future state and correcting when the server disagrees. Remote entities are rendered in the past via interpolation and are never predicted.

In **Rollback**, the local player is predicting *remote players' inputs* and running the entire shared simulation forward speculatively. When remote inputs arrive and differ, the *entire game state* must be rolled back and resimulated -- not just the local player.

This has profound architectural consequences:

1. **Rollback requires determinism.** Because there is no authoritative server snapshot to snap to, both peers must arrive at the exact same state given the same inputs. CSP+R does not require determinism because the server provides authoritative state.

2. **Rollback requires complete state serialization.** The game must be able to save and restore its *entire* state, not just the local player's position and velocity. CSP+R typically only needs to reconcile the predicted entity.

3. **Rollback produces different visual artifacts.** In CSP+R, the local player almost never sees corrections (because the server usually agrees with the client). In rollback, the local player sees *remote entities* pop or teleport when predictions are wrong.

### Can They Be Combined?

Yes. Some architectures combine elements of both:

- **Photon Fusion** uses a server-authoritative model with tick-based snapshots, but clients predict forward and resimulate from the server snapshot (rollback-style reconciliation).
- **Rocket League** is server-authoritative but predicts *all* entities (not just the local player), using rollback to correct the entire physics world when the server disagrees.
- **CrystalOrb** calls itself "client-side prediction and server reconciliation" but uses unconditional rollback -- every server snapshot triggers a full re-simulation from the snapshot timestamp to the present.

---

## How It Works Step by Step

The rollback loop is the same regardless of whether the architecture is peer-to-peer or client-server. The following describes a single frame of the rollback cycle.

### 1. Sample Local Input

The local player's controller is read. The input is timestamped with the current simulation frame number and immediately available for use in the local simulation. There is zero added delay (or a small fixed delay in hybrid configurations).

### 2. Send Local Input to Remote Peers / Server

The local input (plus a trailing window of recent inputs for packet loss resilience) is transmitted over the network. Including redundant input history means that even if some packets are dropped, the remote side can still reconstruct the input timeline.

```
Packet contents:
  frame: 142
  input: { right: true, jump: false, action: false }
  history: [
    { frame: 141, input: { right: true, jump: false, action: false } },
    { frame: 140, input: { right: true, jump: true, action: false } },
    { frame: 139, input: { right: false, jump: false, action: false } },
  ]
```

### 3. Receive Remote Inputs

Process any incoming network packets. For each received remote input, store it in the input history buffer. Compare received inputs against any predictions that were made for those frames.

### 4. Detect Misprediction

If a received remote input for frame N differs from what was predicted for frame N, a rollback is required. The rollback target is the *earliest* mispredicted frame.

### 5. Rollback (if needed)

If a misprediction was detected:

1. **Load state** from the rollback target frame (the last frame before the misprediction).
2. **Resimulate** forward from that frame to the current frame, using:
   - Real local inputs (always available, since they were recorded)
   - Real remote inputs for any frame where they have now been received
   - Predicted remote inputs for any frame where real inputs have still not arrived
3. **Save state** after each resimulated frame (in case a future rollback needs them).

This entire sequence -- load, resimulate N frames, save N states -- must complete within the CPU budget of a single rendered frame.

### 6. Advance the Current Frame

With the corrected (or already-correct) state, advance the simulation by one frame using the local input and the remote input (real or predicted) for this frame. Save the resulting state.

### 7. Render

Render the current game state to the screen. Only this final state is rendered -- the intermediate resimulated frames during rollback are **never rendered**.

### 8. Time Synchronization

Both peers must be running approximately the same simulation frame at the same time. If one peer gets ahead, the system briefly pauses the leading peer (typically 1-2 frames) to allow the lagging peer to catch up. GGPO signals this via the `GGPO_EVENTCODE_TIMESYNC` event.

```
Frame timeline showing a 3-frame rollback:

  Confirmed state   Mispredicted     Current frame
  (all inputs known)  frames          (being rendered)
        v              v                    v
  ... [50] [51] [52] [53] [54] [55] [56] [57]
              |                              |
              +-- Rollback loads state 51 ---+
                  Resimulates 52..57 with
                  corrected inputs
```

---

## State Management

State management is the make-or-break factor in a rollback implementation. The game must be able to save its complete state extremely quickly (ideally < 0.2ms) and restore it equally fast, potentially many times per rendered frame.

### What Must Be Saved

Everything that affects the deterministic simulation must be captured:

- Player positions, velocities, accelerations
- Health, meter, ammo, cooldown timers
- Active hitboxes, hurtboxes, collision shapes
- Projectile / entity positions and states
- Animation frame indices (if they affect gameplay logic)
- Physics state (positions, velocities, contacts)
- Random number generator state
- Round/match timer
- Game mode state (score, round number, etc.)

**What should NOT be saved:** Rendering state (particle positions, camera shake), audio playback state, UI state, analytics, or anything that does not feed back into the simulation.

### Ring Buffer Architecture

The most common storage pattern is a **ring buffer** (circular buffer) of state snapshots:

```
Ring buffer of size MAX_ROLLBACK + INPUT_DELAY + 2:

  Index: [0] [1] [2] [3] [4] [5] [6] [7] [8] [9]
  Frame:  48  49  50  51  52  53  54  55  56  57
                                              ^
                                        current frame

  When frame 58 is saved, it overwrites index 0 (frame 48):
  Frame:  58  49  50  51  52  53  54  55  56  57
```

The buffer must be large enough to hold states for the maximum rollback depth. For a system with 2 frames of input delay and a maximum of 8 rollback frames, a buffer of at least 12 entries is typical.

### Strategies for Fast Serialization

#### 1. Contiguous Struct (memcpy)

The ideal approach: keep all game state in a single, flat, contiguous memory region. Save and load become a single `memcpy`.

```c
// C/C++ - the fastest possible approach
struct GameState {
    Player players[2];
    Projectile projectiles[MAX_PROJECTILES];
    int round_timer;
    uint64_t rng_state;
    // ... everything in one flat struct
};

// Save: ~0.01ms for a small game state
memcpy(snapshot_buffer[frame % BUFFER_SIZE], &game_state, sizeof(GameState));

// Load:
memcpy(&game_state, snapshot_buffer[frame % BUFFER_SIZE], sizeof(GameState));
```

This is what GGPO's documentation recommends and what most competitive fighting game implementations use.

#### 2. ECS Column Snapshot

In an Entity-Component-System architecture, snapshot each component array (column) separately. This can be faster than serializing the entire world if only some component types participate in the simulation.

#### 3. Delta / Incremental Snapshots

Instead of saving the full state every frame, save only what changed since the previous frame.

**Advantages:**
- Reduced memory usage (deltas are typically much smaller than full snapshots)
- Faster save operations when few things change per frame

**Disadvantages:**
- More complex implementation (must instrument all state mutations)
- Loading may be slower (must apply chain of deltas)
- Harder to debug desync issues

**Hybrid approach:** Save a full snapshot every N frames (e.g., every 8 frames) and deltas between them. This bounds the maximum number of deltas that need to be applied during a rollback.

#### 4. Copy-on-Write

Track which parts of the state have been written to since the last snapshot. Only copy the dirty regions. This is conceptually similar to delta snapshots but operates at the memory level rather than the logical level.

### Checksum-Based Desync Detection

Many rollback implementations compute a checksum (e.g., Fletcher-32, CRC32, or xxHash) of the saved state each frame and transmit it to the remote peer. If checksums diverge, a desynchronization has occurred -- typically caused by a determinism bug. GGPO provides a dedicated sync test mode that catches these issues during development.

---

## Input Prediction Strategies

When the remote player's input has not yet arrived for a frame that needs to be simulated, a prediction must be made. The quality of this prediction directly affects the frequency and severity of rollbacks.

### 1. Last-Known-Input Repeat (Standard)

The simplest and most widely used strategy: assume the remote player is still doing whatever they were doing on the last frame for which real input was received.

```
Frame 50: Remote input received = { right: true, jump: false }
Frame 51: No input received → predict { right: true, jump: false }
Frame 52: No input received → predict { right: true, jump: false }
Frame 53: Real input arrives = { right: true, jump: false } → correct!
```

This works remarkably well because at 60fps, players rarely change their input from one frame to the next. The prediction accuracy is typically **90%+** under normal network conditions.

**Why this is preferred over smarter prediction:** Incorrectly predicting that a player *did* something (e.g., threw a punch) creates far worse visual artifacts when rolled back than predicting they did *nothing new*. A predicted action that suddenly vanishes is jarring; a slightly delayed appearance of a new action is natural.

### 2. Input Decay (Graduated Prediction)

For games with continuous movement (analog sticks, vehicle physics), simply repeating the last input can cause overshooting. Input decay reduces the predicted input magnitude over successive prediction frames:

| Prediction Depth | Applied Input Magnitude |
|---|---|
| Frame +1 | 100% of last known input |
| Frame +2 | 66% (2/3) |
| Frame +3 | 33% (1/3) |
| Frame +4+ | 0% (assume idle) |

The intuition: **undershooting then correcting forward looks much better than overshooting then snapping back**. If a player was moving right and stops, decayed prediction shows them slowing to a stop and then nudging forward to the correct position. Without decay, the player overshoots and rubber-bands backward.

This technique is used by **Rocket League** for vehicle physics prediction, as described in Jared Cone's GDC 2018 talk *"It IS Rocket Science!"*

### 3. Predict Neutral (Assume Idle)

Always predict that the remote player is doing nothing. This minimizes the worst-case visual artifact (no false action to undo) at the cost of making remote players appear to react late. Useful for games where actions have dramatic visual consequences.

### 4. Hybrid Delay + Rollback

Rather than predicting from frame 0, add a small fixed input delay (typically 2-3 frames / 33-50ms) so that remote inputs have time to arrive before they are needed. Any remaining latency beyond the delay window is handled by prediction and rollback.

```
Total latency coverage = input_delay_frames + max_rollback_frames
```

| Network RTT | Input Delay | Max Rollback | Player Experience |
|---|---|---|---|
| 0-50ms | 2-3 frames | 0 frames | Feels offline |
| 50-100ms | 2-3 frames | 1-3 frames | Excellent |
| 100-150ms | 3 frames | 3-6 frames | Good; occasional pops |
| 150-200ms | 3-4 frames | 6-8 frames | Noticeable artifacts |
| 200ms+ | 4+ frames | 8+ frames | Degraded experience |

Most players cannot perceive 2-3 frames (33-50ms) of input delay. Console TVs and monitors often add 2-4 frames of display lag on their own. Fighting games have historically shipped with 3-4 frames of built-in input delay even offline. By absorbing baseline latency with a small fixed delay, the prediction window shrinks and rollbacks become rarer and less severe.

### 5. ML-Based Prediction (Experimental)

Some researchers have explored using machine learning to predict remote inputs based on player behavior patterns. In practice this has not been adopted in production games because:

- Incorrect predictions of *action* are worse than incorrect predictions of *inaction*
- Training data is player-specific and session-specific
- The marginal improvement over last-input-repeat is small
- The computational cost conflicts with the tight CPU budget

---

## Resimulation

Resimulation is the process of re-running the game simulation from a past state to the present after a rollback. It is the most computationally expensive part of the rollback loop and the primary constraint on how many frames of rollback a game can support.

### What Resimulation Requires

For each resimulated frame, the game must:

1. Read the local and remote inputs for that frame from the input history
2. Run one complete tick of game logic (physics, collision, gameplay, animation state machines)
3. Save the resulting state to the ring buffer

Critically, resimulation must **not** produce any side effects:
- No rendering
- No audio playback
- No particle spawning
- No analytics events
- No network I/O

This is why rollback architectures mandate a strict separation between simulation and presentation.

### Performance Budget

For a 60fps game (16.67ms per rendered frame):

```
Frame budget breakdown (worst case, 8-frame rollback):

  Render:                          ~6ms
  Normal frame simulation:         ~1ms
  State load:                      ~0.2ms
  Resimulate 8 frames:        8 x ~1ms = ~8ms
  State save (8 frames):      8 x ~0.2ms = ~1.6ms
  Network I/O:                     ~0.5ms
  ─────────────────────────────────────────
  Total:                           ~17.3ms  (TIGHT)
```

This means game logic must be **extremely fast** -- ideally under 1ms per tick. NetherRealm Studios' Michael Stallone reported in his GDC 2018 talk ["8 Frames in 16ms"](https://www.gdcvault.com/play/1025471/8-Frames-in-16ms-Rollback) that achieving this for Mortal Kombat required approximately **2 man-years** of optimization across nearly every engine system.

### The Spiral of Death

If resimulation takes longer than one frame, the game falls behind. On the next frame, the rollback window is now larger (one more frame to resimulate), which takes even longer, which causes the game to fall further behind. This positive feedback loop is called the **spiral of death** and is the catastrophic failure mode of rollback netcode.

Prevention strategies:
- Cap the maximum rollback depth (GGPO defaults to 8 frames)
- Accept desyncs rather than attempting unbounded resimulation
- Dynamically increase input delay when latency spikes
- Monitor resimulation time and alert if approaching budget

### Optimization Techniques

1. **Skip confirmed frames.** If frames 50-53 are now fully confirmed and were already simulated correctly, only resimulate frames 54-57.

2. **Early exit on matching state.** If after resimulating frame N, the new state matches the previously predicted state (via checksum), skip resimulating frames N+1 onward -- the rest of the prediction was correct.

3. **Reduce simulation fidelity during rollback.** Some implementations run a simplified physics pass during resimulation (e.g., skip broad-phase, use bounding boxes instead of precise collision). This is risky -- any difference in logic between normal simulation and resimulation will cause desyncs.

4. **Amortize across frames.** If a rollback requires resimulating more frames than the budget allows, spread the correction across 2-3 rendered frames. This introduces brief visual inconsistency but prevents frame drops.

---

## Determinism Requirements

Determinism is the most demanding architectural requirement of rollback netcode. The simulation must produce **bit-identical** results on all peers given the same inputs and starting state. Any divergence, no matter how small, will compound over frames and result in a desync.

### Why Rollback Requires Determinism

In peer-to-peer rollback, there is no authoritative server to provide a canonical state. Both peers run the same simulation independently. If peer A computes a slightly different position for a character than peer B (even by 0.001 units), that difference will grow over time until the two simulations are completely out of sync.

Note: Server-authoritative rollback systems (like Rocket League or Photon Fusion) are more forgiving because the server periodically corrects client state. But even these benefit from determinism, as it reduces the frequency and magnitude of corrections.

### Sources of Non-Determinism

#### Floating-Point Arithmetic

The single largest source of non-determinism. As Glenn Fiedler describes in [Floating Point Determinism](https://gafferongames.com/post/floating_point_determinism/):

- **Compiler optimizations** may reorder floating-point operations, changing results due to rounding
- **x87 vs. SSE instructions** compute at different internal precisions (80-bit vs. 64-bit)
- **Fused multiply-add (FMA)** on some architectures produces different results than separate multiply + add
- **Transcendental functions** (sin, cos, tan, sqrt) are implemented differently across platforms and even across CPU models from the same vendor
- **Debug vs. Release builds** may produce different results due to optimization level differences

#### Random Number Generation

Any use of randomness in the simulation must use a **seeded, deterministic PRNG** whose state is part of the saved game state. System-level random functions or time-based seeds will diverge.

#### Execution Order

If entities are processed in a non-deterministic order (e.g., iterating an unordered hash map, or processing based on memory address), results will differ across machines.

#### Uninitialized Memory

Reading from uninitialized variables can produce different values on different machines or runs.

#### Third-Party Libraries

Physics engines, math libraries, and middleware may not guarantee determinism. Unity's built-in physics, for example, is explicitly non-deterministic -- the coherence documentation [warns against using it](https://docs.coherence.io/manual/advanced-topics/competitive-games/determinism-prediction-rollback) in GGPO-mode games.

### Solutions

#### Fixed-Point Math

Replace all floating-point arithmetic in the simulation with fixed-point integers. A common representation is Q16.16 (16 bits integer, 16 bits fractional):

```typescript
// Fixed-point representation (Q16.16)
type Fixed = number; // In TypeScript, we use integer arithmetic

const FIXED_SHIFT = 16;
const FIXED_ONE = 1 << FIXED_SHIFT; // 65536 = 1.0

function fixedMul(a: Fixed, b: Fixed): Fixed {
  // Use BigInt or careful integer math to avoid overflow
  return Math.trunc((a * b) / FIXED_ONE);
}

function fixedDiv(a: Fixed, b: Fixed): Fixed {
  return Math.trunc((a * FIXED_ONE) / b);
}

function toFixed(f: number): Fixed {
  return Math.trunc(f * FIXED_ONE);
}

function fromFixed(f: Fixed): number {
  return f / FIXED_ONE;
}
```

Fixed-point math is completely deterministic but sacrifices precision and requires rewriting all math operations. It is used by games like **Age of Empires** (deterministic lockstep) and is recommended by the **coherence** documentation for GGPO-mode games.

#### Strict Floating-Point Compiler Settings

If cross-platform determinism is not required (e.g., same-platform peer-to-peer), floating-point determinism can often be achieved with strict compiler settings:

- **MSVC:** `/fp:strict`
- **GCC/Clang:** `-ffp-contract=off -fno-fast-math`
- **FPU configuration:** Force consistent precision mode (64-bit, not 80-bit x87) and rounding mode via `_controlfp()` or equivalent

This approach is fragile -- any third-party library or driver that changes FPU settings can break determinism.

#### Softfloat Libraries

Software-emulated floating-point libraries (e.g., Berkeley SoftFloat) guarantee bit-identical results across all platforms at significant performance cost. Some games use softfloat selectively for critical calculations (collision detection, physics) while using hardware floats for non-critical visuals.

#### The Pragmatic Tier System

[Yal's guide to deterministic netcode](https://yal.cc/preparing-your-game-for-deterministic-netcode/) recommends a progressive approach:

| Tier | Scope | Approach |
|---|---|---|
| **Tier 0** | Foundation | Build local multiplayer first; document all simulation state |
| **Tier 1** | Desktop lockstep | Abstract input polling; implement replay system as determinism test |
| **Tier 2** | Cross-platform | Decouple simulation from rendering; support variable simulation steps per frame |
| **Tier 3** | Rollback-ready | Fast state serialization/deserialization (< 10% of frame time); test via replay save/load |

The key practical test: **if your replay system works without desyncing, your simulation is deterministic enough for rollback.**

---

## Visual Artifacts & Mitigation

Rollback netcode trades input latency for visual consistency. When predictions are wrong, corrections manifest as visible artifacts. Understanding and mitigating these artifacts is critical for a polished rollback implementation.

### Types of Visual Artifacts

#### 1. Teleporting / Position Snapping

The most common artifact. When a remote player's predicted position differs from their actual position after rollback, they visually snap to the corrected location. The severity depends on:
- **Prediction depth:** More frames of prediction = larger potential position error
- **Movement speed:** Fast-moving entities produce larger positional errors
- **Input change frequency:** Players who change direction frequently trigger more mispredictions

#### 2. Animation Pops

An animation that was playing during prediction may be replaced by a different animation after correction. A character that appeared to be walking may suddenly be in a jump animation, or a punch startup may appear to skip frames.

#### 3. Action Reversals

The most jarring artifact: an action that appeared to happen (a hit connecting, a projectile spawning) is undone by rollback. The player sees a hit, then the opponent is suddenly not hit. A projectile appears and then vanishes.

#### 4. Phantom Collisions

During prediction, a collision may be detected that does not exist after correction (or vice versa). Objects may briefly clip through each other or bounce off invisible barriers.

### Mitigation Strategies

#### Visual Smoothing / Interpolation

Rather than snapping to the corrected position immediately, interpolate over 2-4 frames:

```typescript
// After rollback correction, calculate visual offset
const correctionOffset = {
  x: predictedPosition.x - correctedPosition.x,
  y: predictedPosition.y - correctedPosition.y,
};

// Each rendered frame, decay the offset
const SMOOTHING_RATE = 0.15; // Blend 15% per frame
visualOffset.x *= (1 - SMOOTHING_RATE);
visualOffset.y *= (1 - SMOOTHING_RATE);

// Render at corrected position + decaying offset
renderPosition.x = correctedPosition.x + visualOffset.x;
renderPosition.y = correctedPosition.y + visualOffset.y;
```

This makes corrections appear as smooth adjustments rather than hard snaps.

#### Startup Frames (Design for Rollback)

Game designers can add **anticipatory animation frames** before impactful actions. If a punch takes 3 startup frames before the active hitbox appears, even a 3-frame rollback will show the startup animation on remote clients before the hit connects. Without startup frames, the hit appears to come from nowhere.

This is standard practice in fighting games (where moves have natural startup, active, and recovery phases) and should be adopted by any genre using rollback. As the SnapNet documentation notes: "adding just 2 or 3 frames to the startup of moves can make the game look vastly better online while playing almost identically."

#### Effect Confirmation Delay

Delay visual effects (particles, screen shake, flash) until the frame they occurred on has been confirmed by the remote peer. This prevents the worst visual artifact -- seeing an effect and then having it disappear.

```typescript
// Instead of:
if (hit.detected) {
  spawnHitParticles(hit.position);
}

// Use:
if (hit.detected && hit.frame <= lastConfirmedFrame) {
  spawnHitParticles(hit.position);
}
```

The trade-off is that effects appear with a slight delay proportional to the network latency.

#### Cosmetic Divergence Tolerance

For non-gameplay-affecting visuals (particles, debris, camera shake), allow the local simulation to diverge from the "true" state. Do not include these in the rollback state. They will be slightly different on each peer, but this is invisible to players.

#### Clamped Correction Distance

Set a maximum correction distance per frame. If the correction would move an entity further than the threshold, spread it across multiple frames:

```typescript
const MAX_CORRECTION_PER_FRAME = 5; // units
const correction = correctedPos - displayPos;
const correctionMagnitude = Math.sqrt(correction.x ** 2 + correction.y ** 2);

if (correctionMagnitude > MAX_CORRECTION_PER_FRAME) {
  const scale = MAX_CORRECTION_PER_FRAME / correctionMagnitude;
  displayPos.x += correction.x * scale;
  displayPos.y += correction.y * scale;
} else {
  displayPos = correctedPos;
}
```

---

## Audio Handling During Rollback

Audio is one of the most challenging aspects of rollback netcode. During resimulation, the game logic runs through multiple frames that may trigger sound effects -- but these frames are not being rendered, so sounds should not play. And when a rollback invalidates a previously triggered sound, the already-playing audio must be handled gracefully.

### The Problem

Consider this scenario:
1. Frame 50: Game logic triggers "punch impact" sound. Sound begins playing.
2. Frame 52: Remote input for frame 48 arrives, causing a rollback to frame 48.
3. During resimulation of frames 48-52, frame 50's logic runs again. If not handled, the "punch impact" sound would play *again*, overlapping with the already-playing instance.
4. Worse: the corrected simulation might show the punch *missed*, so the sound should not have played at all.

### Solution Architecture: Desired vs. Actual State

The recommended approach, described in detail in the [Cargo Space devlog](https://johanhelsing.studio/posts/cargo-space-devlog-4/), separates sound into two layers:

1. **Desired state** (inside the rollback simulation): Sound effects are represented as data entities -- "this sound should be playing starting at frame N" -- not as actual audio playback calls.

2. **Actual state** (outside the rollback simulation): A post-rollback system compares the desired sound state against currently playing audio and reconciles:

| Scenario | Action |
|---|---|
| Sound exists in desired state but not playing | Start playing (possibly seek to correct position) |
| Sound playing but not in desired state | Fade out over ~100ms (mispredicted sound) |
| Sound exists in both | Continue playing (adjust seek position if needed) |

### Practical Rules

1. **Never play audio directly from simulation code.** Instead, set flags or spawn data entities that a post-simulation system reads.

2. **During resimulation, skip all audio processing.** Use a boolean flag (`isResimulating`) that the audio system checks.

3. **Use fade-outs, not hard stops.** When a mispredicted sound must be cancelled, fade it out over 50-100ms rather than cutting it instantly. Abrupt silence is more noticeable than a brief phantom sound.

4. **Accept short phantom sounds.** Very short sound effects (< 100ms) that were mispredicted are often not worth cancelling -- the cancellation itself draws more attention.

5. **Distinguish sound identity.** Use a composite key (sound clip + frame number + entity ID) to match desired sounds to playing instances. This prevents different entities' sounds from interfering with each other.

---

## Beyond Fighting Games

While rollback netcode was popularized by fighting games, its principles apply broadly. Here is how different genres adapt the technique.

### Physics Games: Rocket League

Rocket League (Psyonix, 2015) is the most prominent non-fighting-game using rollback. As described in Jared Cone's GDC 2018 talk ["It IS Rocket Science!"](https://www.gdcvault.com/play/1024972/It-IS-Rocket-Science-The):

- **Server-authoritative:** Unlike GGPO's peer-to-peer model, Rocket League uses dedicated servers running the Bullet physics engine at 60Hz.
- **Full world prediction:** Clients predict *all* physics objects (cars, ball, boost pads), not just the local player. Remote cars are rendered in the present (predicted), not in the past (interpolated).
- **Input decay:** Remote car inputs are decayed over prediction frames (100% → 66% → 33% → 0%) to reduce overshooting artifacts.
- **Physics rollback:** When the server snapshot arrives and differs from the client's predicted state, the client rolls back the *entire physics world* and resimulates forward. This requires the physics engine itself to support fast save/restore.
- **Visual smoothing:** Corrections are blended over time using lerp (position) and slerp (rotation) to avoid hard snaps.

The key challenge for physics games is that physics engines are typically not designed for fast save/restore. Psyonix had to modify Bullet to support rapid state serialization.

### Platform Fighters / Brawlers

Games like **Brawlhalla** (Blue Mammoth, 2017), **Nickelodeon All-Star Brawl** (Ludosity, 2021), and **MultiVersus** (Player First Games, 2022) use rollback for platform-fighter gameplay. Challenges specific to this genre:

- **More entities than a 1v1 fighter:** Items, projectiles, stage hazards all need to be part of the rollback state
- **Larger stages with scrolling cameras:** Camera state may need rollback to avoid visual jumps
- **4+ player support:** More players = more mispredictions and larger state to serialize

### Action Games / Beat-'em-ups

**Dungeons & Dragons: Chronicles of Mystara** (Iron Galaxy, 2013) used GGPO for 4-player co-op beat-'em-up action. The challenges here include:

- **Many AI enemies:** All enemy AI state must be deterministic and serializable
- **Complex game state:** Items, environmental objects, NPC states
- **Cooperative tolerance:** Co-op players are more tolerant of visual artifacts than competitive players

### Platformers

**DelayNoMore** is an open-source multiplayer platformer using rollback netcode with websockets, demonstrating that rollback works for web-based platformers. Key adaptations:

- **Simpler state:** 2D platformer state is typically small and fast to serialize
- **Deterministic collision:** Tile-based collision is naturally deterministic
- **Lower stakes for misprediction:** A platform game character teleporting a few pixels is less disruptive than in a fighting game

### Where Rollback Struggles

- **Large-scale games (MMOs, battle royale):** Too many entities to serialize and resimulate efficiently.
- **FPS games:** Precise aim makes even small corrections feel wrong. CSP+Reconciliation with entity interpolation is preferred.
- **RTS games:** Hundreds of units make state serialization impractical. Deterministic lockstep with input-only sync is preferred.
- **Physics-heavy sandbox games:** Complex physics interactions are expensive to resimulate and hard to make deterministic.

---

## Pseudo-Code / Code Examples

### Complete Rollback Loop (Pseudo-Code)

```
const TICK_RATE = 60
const MAX_ROLLBACK = 8
const INPUT_DELAY = 2
const INPUT_HISTORY_SIZE = 128
const STATE_BUFFER_SIZE = MAX_ROLLBACK + INPUT_DELAY + 4

// ─── Data Structures ───

stateBuffer    = RingBuffer<GameState>(STATE_BUFFER_SIZE)
localInputs    = RingBuffer<Input>(INPUT_HISTORY_SIZE)
remoteInputs   = RingBuffer<Input?>(INPUT_HISTORY_SIZE)  // nullable (may not have arrived)
predictions    = RingBuffer<Input>(INPUT_HISTORY_SIZE)
currentFrame   = 0
lastConfirmed  = -1

// ─── Main Loop ───

function mainLoop():
    while running:
        frameStart = clock()

        // 1. Poll network
        for (frame, input) in network.receive():
            remoteInputs[frame] = input

        // 2. Detect misprediction
        rollbackTo = -1
        for frame in range(lastConfirmed + 1, currentFrame):
            if remoteInputs[frame] != null
               and remoteInputs[frame] != predictions[frame]:
                rollbackTo = frame
                break

        // 3. Update lastConfirmed
        while remoteInputs[lastConfirmed + 1] != null:
            lastConfirmed += 1

        // 4. Rollback if needed
        if rollbackTo >= 0:
            stateBuffer.load(rollbackTo, gameState)

            for f in range(rollbackTo, currentFrame):
                local  = localInputs[f]
                remote = remoteInputs[f] ?? predictions[f]  // real or re-predict
                simulate(gameState, local, remote)
                stateBuffer.save(f + 1, gameState)

        // 5. Read local input and schedule it with delay
        rawInput = readController()
        scheduledFrame = currentFrame + INPUT_DELAY
        localInputs[scheduledFrame] = rawInput

        // 6. Send input to peer (with redundant history)
        network.send({
            frame: scheduledFrame,
            input: rawInput,
            history: localInputs.range(scheduledFrame - 5, scheduledFrame)
        })

        // 7. Predict remote input for current frame (if not yet received)
        if remoteInputs[currentFrame] == null:
            predictions[currentFrame] = predictInput(currentFrame)

        // 8. Advance simulation
        local  = localInputs[currentFrame]
        remote = remoteInputs[currentFrame] ?? predictions[currentFrame]
        simulate(gameState, local, remote)
        currentFrame += 1
        stateBuffer.save(currentFrame, gameState)

        // 9. Render
        render(gameState)

        // 10. Time sync (pause if ahead of peer)
        if framesAheadOfPeer() > TIME_SYNC_THRESHOLD:
            sleep(frameDuration)

        // 11. Frame pacing
        elapsed = clock() - frameStart
        if elapsed < frameDuration:
            sleep(frameDuration - elapsed)


function predictInput(frame):
    // Find the most recent confirmed remote input
    for f in range(frame - 1, frame - MAX_ROLLBACK, -1):
        if remoteInputs[f] != null:
            return remoteInputs[f]  // Carry forward last known
    return INPUT_NEUTRAL            // No data: assume idle


function simulate(state, localInput, remoteInput):
    applyInput(state.players[0], localInput)
    applyInput(state.players[1], remoteInput)
    updatePhysics(state)
    resolveCollisions(state)
    updateGameLogic(state)
    state.frameNumber += 1
```

### TypeScript Implementation

The following is a practical TypeScript implementation of a rollback networking system. It is intentionally self-contained and demonstrates the full rollback loop with state management, input prediction, and resimulation.

```typescript
// ============================================================
// rollback.ts — Practical Rollback Netcode Implementation
// ============================================================

// ─── Configuration ───

interface RollbackConfig {
  tickRate: number;         // Simulation ticks per second (e.g., 60)
  maxRollbackFrames: number; // Maximum frames to roll back (e.g., 8)
  inputDelay: number;       // Frames of input delay (e.g., 2)
  inputHistorySize: number;  // Ring buffer capacity for inputs
  stateBufferSize: number;   // Ring buffer capacity for snapshots
}

const DEFAULT_CONFIG: RollbackConfig = {
  tickRate: 60,
  maxRollbackFrames: 8,
  inputDelay: 2,
  inputHistorySize: 128,
  stateBufferSize: 14, // maxRollback + inputDelay + 4
};

// ─── Input ───

interface GameInput {
  up: boolean;
  down: boolean;
  left: boolean;
  right: boolean;
  action1: boolean;
  action2: boolean;
}

const NEUTRAL_INPUT: GameInput = {
  up: false, down: false, left: false, right: false,
  action1: false, action2: false,
};

function inputsEqual(a: GameInput, b: GameInput): boolean {
  return (
    a.up === b.up && a.down === b.down &&
    a.left === b.left && a.right === b.right &&
    a.action1 === b.action1 && a.action2 === b.action2
  );
}

// ─── Ring Buffer ───

class RingBuffer<T> {
  private buffer: (T | null)[];
  private capacity: number;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.buffer = new Array(capacity).fill(null);
  }

  set(frame: number, value: T): void {
    this.buffer[frame % this.capacity] = value;
  }

  get(frame: number): T | null {
    return this.buffer[frame % this.capacity];
  }

  has(frame: number): boolean {
    return this.buffer[frame % this.capacity] !== null;
  }
}

// ─── Game State Interface ───
// Your game must implement this interface.

interface GameState {
  /** Deep clone the state. Must produce an independent copy. */
  clone(): GameState;

  /**
   * Compute a checksum for desync detection.
   * Must be deterministic — same state always produces same checksum.
   */
  checksum(): number;

  /** The frame number this state represents. */
  frameNumber: number;
}

/** Function that advances the game state by one tick given two players' inputs. */
type SimulateFn = (state: GameState, localInput: GameInput, remoteInput: GameInput) => void;

/** Function that reads local controller input. */
type ReadInputFn = () => GameInput;

// ─── Network Interface ───

interface InputMessage {
  frame: number;
  input: GameInput;
  /** Recent input history for packet loss recovery */
  history: { frame: number; input: GameInput }[];
}

interface NetworkAdapter {
  send(msg: InputMessage): void;
  receive(): InputMessage[];
}

// ─── Rollback Stats ───

interface RollbackStats {
  rollbacksThisSecond: number;
  maxRollbackDepth: number;
  averagePredictionAccuracy: number; // 0..1
  confirmedFrame: number;
  currentFrame: number;
}

// ─── Rollback Session ───

class RollbackSession {
  private config: RollbackConfig;
  private simulate: SimulateFn;
  private network: NetworkAdapter;

  // State management
  private stateBuffer: RingBuffer<GameState>;
  private gameState: GameState;

  // Input management
  private localInputs: RingBuffer<GameInput>;
  private remoteInputs: RingBuffer<GameInput | null>;
  private predictions: RingBuffer<GameInput>;

  // Frame tracking
  private currentFrame: number = 0;
  private lastConfirmedFrame: number = -1;

  // Stats
  private rollbackCount: number = 0;
  private predictionHits: number = 0;
  private predictionTotal: number = 0;

  constructor(
    config: RollbackConfig,
    initialState: GameState,
    simulateFn: SimulateFn,
    network: NetworkAdapter,
  ) {
    this.config = config;
    this.simulate = simulateFn;
    this.network = network;
    this.gameState = initialState.clone();

    this.stateBuffer = new RingBuffer(config.stateBufferSize);
    this.localInputs = new RingBuffer(config.inputHistorySize);
    this.remoteInputs = new RingBuffer(config.inputHistorySize);
    this.predictions = new RingBuffer(config.inputHistorySize);

    // Save initial state
    this.stateBuffer.set(0, initialState.clone());
  }

  /**
   * Call this once per rendered frame. Returns the game state to render.
   */
  advanceFrame(localRawInput: GameInput): GameState {
    // ── Step 1: Receive remote inputs ──
    const received = this.network.receive();
    for (const msg of received) {
      // Store the primary input
      if (!this.remoteInputs.has(msg.frame)) {
        this.remoteInputs.set(msg.frame, msg.input);
      }
      // Also store any history entries (packet loss recovery)
      for (const h of msg.history) {
        if (!this.remoteInputs.has(h.frame)) {
          this.remoteInputs.set(h.frame, h.input);
        }
      }
    }

    // ── Step 2: Detect misprediction ──
    let rollbackTarget = -1;
    for (let f = this.lastConfirmedFrame + 1; f < this.currentFrame; f++) {
      const realInput = this.remoteInputs.get(f);
      const predicted = this.predictions.get(f);
      if (realInput !== null && predicted !== null) {
        // Track prediction accuracy
        this.predictionTotal++;
        if (inputsEqual(realInput, predicted)) {
          this.predictionHits++;
        } else if (rollbackTarget < 0) {
          rollbackTarget = f;
        }
      }
    }

    // ── Step 3: Update confirmed frame ──
    this.updateConfirmedFrame();

    // ── Step 4: Perform rollback if needed ──
    if (rollbackTarget >= 0) {
      const framesToResim = this.currentFrame - rollbackTarget;
      const clampedTarget = (framesToResim > this.config.maxRollbackFrames)
        ? this.currentFrame - this.config.maxRollbackFrames
        : rollbackTarget;

      // Load saved state
      const savedState = this.stateBuffer.get(clampedTarget);
      if (savedState === null) {
        console.error(`Missing state for frame ${clampedTarget}`);
      } else {
        // Restore state
        this.gameState = savedState.clone();

        // Resimulate forward to currentFrame
        for (let f = clampedTarget; f < this.currentFrame; f++) {
          const local = this.localInputs.get(f) ?? NEUTRAL_INPUT;
          const remote = this.remoteInputs.get(f) ?? this.predictions.get(f) ?? NEUTRAL_INPUT;

          this.simulate(this.gameState, local, remote);
          this.stateBuffer.set(f + 1, this.gameState.clone());
        }

        this.rollbackCount++;
      }
    }

    // ── Step 5: Schedule local input with delay ──
    const scheduledFrame = this.currentFrame + this.config.inputDelay;
    this.localInputs.set(scheduledFrame, localRawInput);

    // ── Step 6: Send to peer ──
    const history: { frame: number; input: GameInput }[] = [];
    for (let f = scheduledFrame - 5; f < scheduledFrame; f++) {
      const inp = this.localInputs.get(f);
      if (inp) history.push({ frame: f, input: inp });
    }
    this.network.send({ frame: scheduledFrame, input: localRawInput, history });

    // ── Step 7: Predict remote input if needed ──
    const remoteForThisFrame = this.remoteInputs.get(this.currentFrame);
    if (remoteForThisFrame === null || remoteForThisFrame === undefined) {
      const predicted = this.predictRemoteInput(this.currentFrame);
      this.predictions.set(this.currentFrame, predicted);
    }

    // ── Step 8: Advance simulation ──
    const localInput = this.localInputs.get(this.currentFrame) ?? NEUTRAL_INPUT;
    const remoteInput = this.remoteInputs.get(this.currentFrame)
      ?? this.predictions.get(this.currentFrame)
      ?? NEUTRAL_INPUT;

    this.simulate(this.gameState, localInput, remoteInput);
    this.currentFrame++;
    this.stateBuffer.set(this.currentFrame, this.gameState.clone());

    return this.gameState;
  }

  /**
   * Predict what the remote player is doing on the given frame.
   * Default: carry forward the last known input.
   */
  private predictRemoteInput(frame: number): GameInput {
    // Walk backward to find the most recent confirmed remote input
    for (let f = frame - 1; f >= Math.max(0, frame - this.config.maxRollbackFrames); f--) {
      const input = this.remoteInputs.get(f);
      if (input !== null && input !== undefined) {
        return input;
      }
    }
    return NEUTRAL_INPUT;
  }

  /**
   * Walk forward from lastConfirmedFrame to find the latest frame
   * where both local and remote inputs are known.
   */
  private updateConfirmedFrame(): void {
    let f = this.lastConfirmedFrame + 1;
    while (
      this.localInputs.has(f) &&
      this.remoteInputs.get(f) !== null &&
      this.remoteInputs.get(f) !== undefined
    ) {
      this.lastConfirmedFrame = f;
      f++;
    }
  }

  /** Get diagnostic statistics. */
  getStats(): RollbackStats {
    return {
      rollbacksThisSecond: this.rollbackCount,
      maxRollbackDepth: this.config.maxRollbackFrames,
      averagePredictionAccuracy:
        this.predictionTotal > 0 ? this.predictionHits / this.predictionTotal : 1,
      confirmedFrame: this.lastConfirmedFrame,
      currentFrame: this.currentFrame,
    };
  }
}
```

### Usage Example

```typescript
// ─── Example: Simple 2D Game ───

interface MyGameState extends GameState {
  players: {
    x: number;
    y: number;
    velX: number;
    velY: number;
    hp: number;
  }[];
  projectiles: {
    x: number;
    y: number;
    velX: number;
    velY: number;
    owner: number;
    alive: boolean;
  }[];
  rngState: number;
  frameNumber: number;
}

function createInitialState(): MyGameState {
  return {
    players: [
      { x: 100, y: 300, velX: 0, velY: 0, hp: 100 },
      { x: 700, y: 300, velX: 0, velY: 0, hp: 100 },
    ],
    projectiles: [],
    rngState: 12345,
    frameNumber: 0,

    clone(): MyGameState {
      return JSON.parse(JSON.stringify(this));
    },
    checksum(): number {
      // Simple hash for desync detection
      let hash = 0;
      for (const p of this.players) {
        hash = ((hash << 5) - hash + (p.x * 1000 | 0)) | 0;
        hash = ((hash << 5) - hash + (p.y * 1000 | 0)) | 0;
        hash = ((hash << 5) - hash + p.hp) | 0;
      }
      return hash;
    },
  };
}

const MOVE_SPEED = 3;
const GRAVITY = 0.5;

function simulateMyGame(state: GameState, localInput: GameInput, remoteInput: GameInput): void {
  const s = state as MyGameState;
  const inputs = [localInput, remoteInput];

  for (let i = 0; i < 2; i++) {
    const player = s.players[i];
    const input = inputs[i];

    // Apply movement
    if (input.left) player.velX = -MOVE_SPEED;
    else if (input.right) player.velX = MOVE_SPEED;
    else player.velX = 0;

    if (input.up && player.y >= 300) {
      player.velY = -10; // Jump
    }

    // Apply gravity
    player.velY += GRAVITY;

    // Update position
    player.x += player.velX;
    player.y += player.velY;

    // Floor collision
    if (player.y > 300) {
      player.y = 300;
      player.velY = 0;
    }

    // Clamp to screen
    player.x = Math.max(0, Math.min(800, player.x));
  }

  // Update projectiles
  for (const proj of s.projectiles) {
    if (!proj.alive) continue;
    proj.x += proj.velX;
    proj.y += proj.velY;
    if (proj.x < 0 || proj.x > 800 || proj.y < 0 || proj.y > 600) {
      proj.alive = false;
    }
  }

  s.frameNumber++;
}

// ─── Initialize and run ───

const session = new RollbackSession(
  DEFAULT_CONFIG,
  createInitialState(),
  simulateMyGame,
  myNetworkAdapter, // Your WebRTC / WebSocket adapter
);

function gameLoop(): void {
  const localInput = readController();
  const stateToRender = session.advanceFrame(localInput);
  render(stateToRender);
  requestAnimationFrame(gameLoop);
}
```

---

## Performance Considerations

### CPU Budget

The central performance constraint of rollback is the **resimulation budget**. For a 60fps game:

```
Available per frame:                     16.67ms
  - Rendering:                           ~6-8ms
  - Audio / UI / misc:                   ~1-2ms
  ─────────────────────────────────────────────
  Available for simulation + rollback:   ~6-9ms

With 8-frame max rollback:
  Per-tick simulation budget:            ~0.7-1.1ms
  Per-tick state save:                   ~0.1-0.2ms
```

If your game logic takes 2ms per tick, you can afford at most ~3-4 frames of rollback before exceeding budget. Optimization targets for competitive implementations (per NetherRealm's GDC talk):

- Game logic per tick: **< 1ms**
- State save/load: **< 0.2ms**
- Total state size: **< 64KB** (smaller = faster memcpy)

### State Size Optimization

| Strategy | Description | Impact |
|---|---|---|
| **Flat struct layout** | Keep all state in contiguous memory | Enables memcpy-based save/load |
| **Separate hot/cold state** | Only rollback simulation-critical state | Reduces serialization size |
| **Bit-pack inputs** | Pack boolean inputs into bitfields | Minimal input packet size |
| **Pool allocations** | Pre-allocate all entity slots | Avoids dynamic allocation during save |
| **Fixed-size arrays** | Use `Projectile[MAX_PROJECTILES]` not `ArrayList<Projectile>` | Predictable state size |

### Memory Budget

```
State size: 32KB
Buffer entries: 14 (10 rollback + 2 delay + 2 padding)
Total state memory: 32KB x 14 = 448KB

Input size: 4 bytes per player
Input buffer: 128 entries x 2 players x 4 bytes = 1KB

Total rollback memory overhead: ~450KB
```

This is negligible for modern hardware, but state sizes can grow significantly for complex games. Rocket League's physics state, for instance, must include positions, velocities, and angular velocities for all cars and the ball, plus boost pad states.

### Tick Rate Considerations

Lower tick rates reduce the per-frame resimulation count but increase the time window each tick represents, making corrections more visible:

| Tick Rate | Frame Duration | 8-tick Rollback Window | Effect |
|---|---|---|---|
| 60 Hz | 16.67ms | 133ms | Standard for fighting games |
| 30 Hz | 33.33ms | 267ms | More time per tick but larger corrections |
| 20 Hz | 50ms | 400ms | Feasible for complex simulations |

### Profiling Checklist

When optimizing a rollback implementation, measure:

1. **Time per simulation tick** (the number you must minimize)
2. **Time per state save** (memcpy vs. serialization)
3. **Time per state load**
4. **Peak rollback depth** (number of frames resimulated)
5. **Rollback frequency** (how often rollbacks occur)
6. **Prediction accuracy** (percentage of frames where prediction was correct)

---

## Framework Implementations

### GGPO (C/C++)

The original rollback networking library by Tony Cannon. Open source under MIT license since 2019.

- **Repository:** [github.com/pond3r/ggpo](https://github.com/pond3r/ggpo)
- **Architecture:** Peer-to-peer, callback-based API
- **Max players:** 4 (with spectator support for up to 32)
- **Max prediction frames:** 8
- **Key features:** Sync test mode, spectator mode, network stats, time synchronization
- **Limitations:** C/C++ only; callback-based API requires careful integration; no built-in server authority
- **Used by:** Skullgirls, Street Fighter III: 3rd Strike Online Edition, Guilty Gear XX Accent Core +R

See [GGPO & Rollback in Fighting Games](../ggpo-rollback.md) for a detailed API walkthrough and integration guide.

### GGRS (Rust)

A Rust reimplementation of GGPO with an idiomatic Rust API. Replaces GGPO's callback-style API with a request-based control flow.

- **Repository:** [github.com/gschup/ggrs](https://github.com/gschup/ggrs)
- **Bevy integration:** [github.com/gschup/bevy_ggrs](https://github.com/gschup/bevy_ggrs) -- handles state snapshotting and rollback via Bevy's ECS schedules
- **Key features:** 100% safe Rust, WASM-compatible, P2P and spectator support
- **Architecture:** Request-based (returns a list of actions for the game to fulfill rather than using callbacks)
- **Used by:** Indie Rust/Bevy games; demonstrated in [Extreme Bevy](https://johanhelsing.studio/posts/extreme-bevy) web game tutorial

### CrystalOrb (Rust)

A network-agnostic rollback library focused on client-server architecture with unconditional rollback.

- **Repository:** [github.com/ErnWong/crystalorb](https://github.com/ErnWong/crystalorb)
- **Architecture:** Client-server (not P2P); unconditional rollback on every server snapshot
- **Key differentiator:** Simulates all entities in the present (not interpolating remote entities in the past). Collisions are consistent across clients provided inputs don't dramatically change trajectories.
- **Design philosophy:** Physics-engine and networking agnostic. Functions as a "sequencer" orchestrating state synchronization.
- **Features:** Display state interpolation (simulation timestep independent of render framerate), smooth blending from predicted to authoritative state
- **Limitations:** Requires Rust nightly; no built-in lag compensation; considered experimental

### Photon Fusion (Unity, C#)

A commercial networking framework for Unity with built-in tick-based prediction and rollback.

- **Documentation:** [doc.photonengine.com/fusion](https://doc.photonengine.com/fusion/current/fusion-2-intro)
- **Architecture:** Server-authoritative (Server Mode) or host-authoritative (Host Mode)
- **Prediction model:** Clients predict forward from server snapshots by resimulating `FixedUpdateNetwork()` through all ticks between the validated snapshot and the client's current tick
- **Input system:** `NetworkInput` system shares inputs with server for prediction and resimulation
- **Reconciliation:** When server snapshot arrives, client resets to snapshot tick and resimulates forward with stored inputs
- **Interpolation:** Remote entities (proxies) are interpolated between snapshots for smooth visuals
- **Key features:** Sub-tick lag compensation, automatic snapshot management, Shared Mode (eventual consistency, no rollback)
- **Limitations:** Unity-only; Shared Mode does not support prediction/rollback; commercial licensing

### Coherence GGPO Path (Unity, C#)

Coherence offers two prediction models; the GGPO path provides deterministic rollback.

- **Documentation:** [docs.coherence.io](https://docs.coherence.io/manual/advanced-topics/competitive-games/determinism-prediction-rollback)
- **Architecture:** Rolling buffer of inputs transmitted between clients
- **Mechanisms:** Input delay + prediction + rollback
- **Determinism:** Required -- developers must implement `SimulationState` struct and override `SetInputs()`, `Simulate()`, `Rollback()`, `CreateState()`
- **Constraints:** No Unity built-in physics; no async code in simulation; seeded RNG required
- **Explicitly warned against:** Using for FPS-style games; floating-point reliance across platforms

### Godot Rollback Netcode (GDScript / C#)

A community addon providing GGPO-style rollback for Godot Engine.

- **Repository:** [github.com/dsnopek/godot-rollback-netcode](https://github.com/dsnopek/godot-rollback-netcode)
- **Engine:** Godot 3.x / 4.x
- **Key features:** Integration with Godot's scene tree, state serialization helpers, input manager, SyncTest mode

### Backroll (Rust)

An async-I/O-focused Rust rollback library.

- **Repository:** [github.com/HouraiTeahouse/backroll-rs](https://github.com/HouraiTeahouse/backroll-rs)
- **Key features:** Async transport layer (tokio-compatible), session-based API similar to GGPO

### Easel (JavaScript/TypeScript)

A managed rollback platform for web games with built-in Cloudflare relay infrastructure.

- **Documentation:** [easel.games/docs](https://easel.games/docs/learn/multiplayer/rollback-netcode)
- **Architecture:** True P2P with Cloudflare fan-out and virtual server authority
- **Key features:** Incremental snapshots (only changes saved), automatic rubberbanding, fair latency distribution, server-authoritative input sequencing despite P2P messaging

---

## When to Use / When Not to Use

### Rollback is a Strong Fit When:

- **The game requires instant input responsiveness.** Fighting games, competitive action, rhythm games -- any genre where input delay breaks the experience.
- **The game has a small, fast-to-serialize state.** 2D fighters, platformers, simple physics games. If your complete game state fits in < 64KB and can be saved/loaded in < 0.2ms, rollback is practical.
- **The game has few players per session.** 2-4 players is the sweet spot. Each additional player increases misprediction frequency and state complexity.
- **Network bandwidth is constrained.** Rollback syncs only inputs (a few bytes per frame), not state snapshots (potentially kilobytes). This makes it ideal for P2P connections and mobile networks.
- **Determinism is achievable.** If you control the simulation code and can avoid floating-point non-determinism, rollback's requirements are met.

### Rollback is a Poor Fit When:

- **The game has large, expensive-to-simulate state.** MMOs, battle royale, large-scale RTS -- resimulating 8 frames of hundreds of entities is impractical.
- **The game uses non-deterministic systems you cannot modify.** If you depend on a third-party physics engine that is not deterministic (e.g., Unity's built-in physics), peer-to-peer rollback will desync.
- **Visual precision matters more than input responsiveness.** FPS games where even 1-pixel corrections are noticed by players. CSP+Reconciliation with entity interpolation is preferred.
- **Many players per session.** 8+ players means many independent input streams to predict, exponentially increasing misprediction probability.
- **Development resources are limited.** Rollback requires significant architectural investment: deterministic simulation, fast serialization, logic/rendering separation, audio handling, visual smoothing. CSP+Reconciliation on a server-authoritative model is simpler to implement correctly.

### Decision Quick-Reference

| Game Type | Recommended Approach | Why |
|---|---|---|
| 2D fighting game | **Rollback (P2P)** | Small state, input-critical, 2 players |
| Platform fighter (2-4 players) | **Rollback (P2P or server)** | Small state, tolerant of corrections |
| Physics competitive (Rocket League) | **Server-authoritative + rollback** | Needs authority; physics state manageable |
| FPS (Overwatch, Valorant) | **CSP + Reconciliation** | Visual precision for aiming; interpolate remotes |
| RTS (StarCraft, Age of Empires) | **Deterministic lockstep** | Many units; state too large for rollback |
| Co-op action RPG | **CSP + Reconciliation or lockstep** | Players tolerant of delay; large game state |
| Web-based casual multiplayer | **Rollback or CSP** | Depends on state size and input sensitivity |
| MMO | **Server authority + interpolation** | Massive state; input delay acceptable |

---

## References

1. **GGPO Official Website** -- https://www.ggpo.net

2. **GGPO Source Code (GitHub)** -- https://github.com/pond3r/ggpo

3. **GGPO & Rollback in Fighting Games** (companion deep-dive) -- [../ggpo-rollback.md](../ggpo-rollback.md)

4. **SnapNet: Netcode Architectures Part 2 - Rollback** -- https://www.snapnet.dev/blog/netcode-architectures-part-2-rollback/

5. **Michael Stallone (NetherRealm Studios) - "8 Frames in 16ms: Rollback Networking in Mortal Kombat and Injustice 2"** (GDC 2018) -- https://www.gdcvault.com/play/1025471/8-Frames-in-16ms-Rollback

6. **Jared Cone (Psyonix) - "It IS Rocket Science! The Physics of Rocket League Detailed"** (GDC 2018) -- https://www.gdcvault.com/play/1024972/It-IS-Rocket-Science-The

7. **Glenn Fiedler - "Floating Point Determinism"** (Gaffer On Games) -- https://gafferongames.com/post/floating_point_determinism/

8. **Yal - "Preparing your game for deterministic netcode"** -- https://yal.cc/preparing-your-game-for-deterministic-netcode/

9. **Bruce Dawson - "Floating-Point Determinism"** (Random ASCII) -- https://randomascii.wordpress.com/2013/07/16/floating-point-determinism/

10. **Coherence Documentation: Determinism, Prediction and Rollback** -- https://docs.coherence.io/manual/advanced-topics/competitive-games/determinism-prediction-rollback

11. **Photon Fusion Documentation: Network Simulation Loop** -- https://doc.photonengine.com/fusion/current/concepts-and-patterns/network-simulation-loop

12. **CrystalOrb (Rust Rollback Library)** -- https://github.com/ErnWong/crystalorb

13. **GGRS (Good Game Rollback System, Rust)** -- https://github.com/gschup/ggrs

14. **bevy_ggrs (Bevy ECS Rollback Plugin)** -- https://github.com/gschup/bevy_ggrs

15. **Backroll (Rust Rollback Library)** -- https://github.com/HouraiTeahouse/backroll-rs

16. **Godot Rollback Netcode Addon** -- https://github.com/dsnopek/godot-rollback-netcode

17. **Johan Helsing - "Cargo Space devlog #4: Playing sound effects in a rollback world"** -- https://johanhelsing.studio/posts/cargo-space-devlog-4/

18. **Johan Helsing - "Extreme Bevy: Making a P2P web game with Rust and rollback netcode"** -- https://johanhelsing.studio/posts/extreme-bevy

19. **Ruoyu Sun - "Game Networking Demystified Part V: Interpolation and Rollback"** -- https://ruoyusun.com/2019/09/21/game-networking-5.html

20. **Gabriel Gambetta - "Client-Side Prediction and Server Reconciliation"** -- https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html

21. **Infil - "Fightin' Words: Netcode"** -- https://words.infil.net/w02-netcode-p5.html

22. **Easel Games - "Rollback Netcode, Explained"** -- https://easel.games/docs/learn/multiplayer/rollback-netcode

23. **Delta Rollback: New Optimizations for Rollback Netcode** -- https://medium.com/@david.dehaene/delta-rollback-new-optimizations-for-rollback-netcode-7d283d56e54b

24. **DelayNoMore (Multiplayer Platformer with Rollback)** -- https://github.com/genxium/DelayNoMore
