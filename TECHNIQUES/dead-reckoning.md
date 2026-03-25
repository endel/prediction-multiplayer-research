# Dead Reckoning for Game Developers

> **Disclaimer:** This content has been "vibe-researched" -- it was generated with the assistance of AI agents performing web research and synthesis. While efforts were made to fetch and cross-reference primary sources, details may be inaccurate, outdated, or misrepresented. Always verify against the original references linked throughout before relying on any technical claims.

> For the military simulation and IEEE 1278 DIS Standard perspective on dead reckoning, see the companion document: [Dead Reckoning & DIS Standard](../dead-reckoning-dis.md).

## Table of Contents

1. [Overview](#1-overview)
2. [Dead Reckoning vs Client-Side Prediction](#2-dead-reckoning-vs-client-side-prediction)
3. [How It Works Step by Step](#3-how-it-works-step-by-step)
4. [Extrapolation Models](#4-extrapolation-models)
5. [Threshold-Based Updates](#5-threshold-based-updates)
6. [Smoothing and Convergence](#6-smoothing-and-convergence)
7. [Application in Different Genres](#7-application-in-different-genres)
8. [Dead Reckoning for Different Entity Types](#8-dead-reckoning-for-different-entity-types)
9. [Common Pitfalls](#9-common-pitfalls)
10. [Pseudo-Code and Code Examples](#10-pseudo-code-and-code-examples)
11. [Comparison with Other Techniques](#11-comparison-with-other-techniques)
12. [References](#12-references)

---

## 1. Overview

### What Is Dead Reckoning?

Dead reckoning is a **replicated computing technique** for networked multiplayer games where all participating nodes agree in advance on algorithms to extrapolate (predict forward) the behavior of remote entities. Rather than requiring constant position updates from the server, each client uses the last known state of an entity -- its position, velocity, and optionally acceleration -- to predict where that entity is *right now*, filling in the gaps between infrequent network updates.

The term originates from maritime navigation ("deduced reckoning"), where sailors estimated their current position based on a previously known position, compass heading, speed, and elapsed time. In the networking context, it was first formalized in the 1980s as part of DARPA's SIMNET project for military distributed simulation, and later standardized in IEEE 1278 (DIS). The game development community adopted and adapted these techniques starting in the late 1990s.

### The Problem It Solves

In any networked game, there is an inherent tension between three constraints:

1. **Bandwidth is limited.** Sending every entity's full state at 60 Hz to every client creates enormous traffic. With 100 entities and 64-byte state packets, that is ~375 KB/s per client -- unsustainable for many games.
2. **Latency is unavoidable.** Even at optimal network conditions, packets take 20-150+ ms to arrive. During that time, the game world continues to evolve.
3. **Packet loss happens.** UDP packets can be dropped, reordered, or delayed. The game must continue to function even when updates are missing.

Dead reckoning addresses all three:

| Problem | How Dead Reckoning Helps |
|---------|--------------------------|
| **Bandwidth** | Updates are sent only when the prediction diverges from reality beyond a threshold, reducing traffic by 50-90% |
| **Latency** | Remote entities continue moving smoothly between updates via local extrapolation |
| **Packet loss** | If a packet is lost, the simulation continues with degraded but reasonable accuracy rather than freezing or teleporting |

### Dead Reckoning in One Sentence

> The sender extrapolates its own entity using the last *transmitted* state; when the extrapolation diverges too far from reality, it sends a correction. The receiver extrapolates the entity using the last *received* state, and smoothly blends toward corrections when they arrive.

---

## 2. Dead Reckoning vs Client-Side Prediction

These two terms are frequently confused. They solve *related* but *different* problems, and in a complete game networking stack, they are often used simultaneously.

### Client-Side Prediction (CSP)

CSP predicts the **local player's own entity**. The client applies the player's inputs immediately to a local simulation, producing instant visual feedback. When the server sends back the authoritative result, the client reconciles by comparing its predicted state against the server's state and replaying any unacknowledged inputs.

- **Who uses it:** The local player only.
- **What drives it:** Player input (keystrokes, mouse movement, controller).
- **How it corrects:** Input replay / reconciliation against server authority.
- **Example:** In Overwatch, your character moves instantly when you press W -- that is CSP.

### Dead Reckoning (DR)

DR predicts **other (remote) entities**. The client extrapolates remote entities forward from their last known state using velocity and acceleration. When a new state update arrives from the server, the client smoothly corrects toward the new trajectory.

- **Who uses it:** All remote entities (other players, NPCs, vehicles, projectiles).
- **What drives it:** Physics state (position, velocity, acceleration).
- **How it corrects:** Smoothing / blending toward newly received state.
- **Example:** In an MMO, the other player's character continues running forward between server ticks -- that is dead reckoning.

### Side-by-Side Comparison

| Aspect | Client-Side Prediction | Dead Reckoning |
|--------|----------------------|----------------|
| **Predicts** | Local player | Remote entities |
| **Input** | Player commands | Last known physics state |
| **Authority** | Server (client predicts, server validates) | Server (or owning client in P2P) |
| **Correction method** | Reconciliation + input replay | Smooth blending / convergence |
| **Latency feel** | Instant (zero perceived input lag) | Smooth (fills gaps between updates) |
| **Failure mode** | Snap-back on misprediction | Overshoot / warp on direction change |
| **Typical use** | FPS, action games | MMOs, vehicles, sims, large-scale games |

### How They Work Together

In a typical FPS or action game:

```
Local Player:   CSP (predict locally, reconcile with server)
Remote Players: Entity Interpolation or Dead Reckoning
NPCs:           Dead Reckoning or server-driven interpolation
Projectiles:    CSP (local) or Dead Reckoning (remote)
Vehicles:       Dead Reckoning (high inertia, predictable)
```

Gabriel Gambetta's formulation is helpful: dead reckoning is suitable where "the car's heading and acceleration will remain constant" during the prediction window, while CSP is needed when "the player's direction and speed can change instantly."

---

## 3. How It Works Step by Step

Dead reckoning operates as a feedback loop between the **sender** (the entity's owner) and the **receiver** (every other node). Both sides run the same extrapolation algorithm, but they serve different purposes.

### 3.1 The Three Models

A complete dead reckoning system maintains three conceptual models per entity:

```
Sender Side:                         Receiver Side:

[Truth Model]                        [DR Model (receiver)]
  The real state from                   Extrapolation from last
  local physics/input                   received update
       |                                     |
       |  compare                            |  on correction
       v                                     v
[DR Model (sender)]                  [Smoothing Model]
  Extrapolation from last                Blends from current visual
  TRANSMITTED state                      position to corrected trajectory
       |                                     |
       |  exceeds threshold?                 |
       v                                     v
  SEND UPDATE -------network-------> RECEIVE UPDATE
                                          |
                                          v
                                     Rendered Position
```

### 3.2 Sender-Side Logic (The Entity Owner)

1. **Maintain the truth model.** Every simulation tick, update the entity's true position from physics, player input, or AI.

2. **Run dead reckoning on yourself.** Using the *last transmitted* state (position, velocity, acceleration at the time of the last sent update), extrapolate where receivers *think* you are right now.

3. **Compare truth vs. DR.** Compute the distance between where you actually are and where receivers think you are.

4. **Threshold check.** If the distance exceeds the position threshold (or the orientation error exceeds the orientation threshold), send a new state update containing the current truth state.

5. **Heartbeat.** If no threshold-triggered update has been sent within a heartbeat interval (e.g., 1-5 seconds), send one anyway to confirm the entity still exists and to correct accumulated drift.

6. **Record the transmitted state.** Store the state that was just sent as the new basis for the sender's DR model.

### 3.3 Receiver-Side Logic (All Other Nodes)

1. **Receive state update.** Parse the incoming packet: position, velocity, acceleration, orientation, timestamp.

2. **Calculate correction.** Compare where you were rendering the entity (current DR position) against where the update says the entity is.

3. **Start smoothing.** Begin a convergence routine that blends from the current visual position toward the new trajectory over a short period (100-500 ms).

4. **Update the DR basis.** Store the received state as the new basis for extrapolation.

5. **Every frame: extrapolate + smooth.** Each render frame, compute the DR position from the basis state, then blend with the smoothing model to produce the final rendered position.

### 3.4 Timeline Visualization

```
Time ──────────────────────────────────────────────────────────►

Server sends update at t=0:
  pos=(0,0), vel=(10,0)    "Entity moving right at 10 units/s"
       │
       ▼
Client receives at t=0.05 (50ms latency):
  DR extrapolates: pos = (0,0) + (10,0) * elapsed_time

  t=0.05: rendered at (0.5, 0)    ← extrapolation starts
  t=0.10: rendered at (1.0, 0)
  t=0.15: rendered at (1.5, 0)
  t=0.20: rendered at (2.0, 0)

Server sends correction at t=0.20:
  pos=(1.8, 0.5), vel=(8, 3)    "Entity turned slightly upward"
       │
       ▼
Client receives at t=0.25:
  Current DR position: (2.5, 0)
  Corrected position:  (1.8, 0.5) + extrapolate 0.05s = (2.2, 0.65)
  Error: 0.72 units

  Begin smoothing over 200ms:
  t=0.25: blend 0%   → still near (2.5, 0)
  t=0.30: blend 25%  → moving toward corrected trajectory
  t=0.35: blend 50%
  t=0.40: blend 75%
  t=0.45: blend 100% → now fully on corrected trajectory
```

---

## 4. Extrapolation Models

The choice of extrapolation model determines how accurately the prediction tracks the entity's actual motion. More complex models capture more movement patterns but require more data in each update packet.

### 4.1 Zero-Order (Static)

```
P(t) = P₀
```

No extrapolation. The entity stays at its last known position. Used for stationary entities (buildings, idle NPCs, dropped items). Costs nothing to compute and requires no velocity data.

### 4.2 First-Order: Constant Velocity (Linear Extrapolation)

```
P(t) = P₀ + V₀ * dt
```

Where `dt = t - t₀` (time since last update). This is the most commonly used model in games. It assumes the entity continues at constant velocity in a straight line. Works well for:

- Entities in steady-state motion (cruising vehicles, running characters)
- Short prediction windows (< 200 ms)
- Games with low update rates where simplicity is paramount

**Per-update data needed:** position (12 bytes) + velocity (12 bytes) = 24 bytes for 3D.

### 4.3 Second-Order: Linear Acceleration (Quadratic Extrapolation)

```
P(t) = P₀ + V₀ * dt + 0.5 * A₀ * dt²
```

Incorporates acceleration, producing curved trajectories. Better for:

- Accelerating / decelerating vehicles
- Entities affected by gravity (projectiles in ballistic flight)
- Turning vehicles where centripetal acceleration is significant

**Per-update data needed:** position (12) + velocity (12) + acceleration (12) = 36 bytes for 3D.

**Caution:** Second-order extrapolation can *amplify* errors over time. If the acceleration value becomes stale (the entity stopped accelerating but the old acceleration is still being applied), the prediction diverges faster than first-order would. Glenn Fiedler notes in his state synchronization article that extrapolation with incorrect velocities leads to "pops when objects are updated."

### 4.4 Higher-Order and Polynomial Models

Some advanced systems use third-order (jerk) or polynomial curve fitting:

```
P(t) = P₀ + V₀ * dt + 0.5 * A₀ * dt² + (1/6) * J₀ * dt³
```

These are rarely used in games because:
- They require more state data per update
- Numerical stability decreases with higher orders
- The prediction window in games is typically short enough (50-200 ms) that first or second order suffices
- Error amplification grows with polynomial degree

### 4.5 Body-Axis Extrapolation (Curved Paths)

For vehicles that maintain a constant local velocity while turning, body-axis extrapolation rotates the velocity vector as the entity's orientation changes:

```
O(t) = O₀ + AngularVelocity * dt
V_world(t) = Rotate(V_body, O(t))
P(t) = P₀ + Integrate(V_world, 0, dt)
```

This produces **circular arc trajectories** from linear data -- a car turning at constant speed traces a smooth curve rather than a straight line. The DIS standard formalizes this as algorithms DRM(R,P,B) and DRM(R,V,B). For game development, body-axis extrapolation is particularly effective for:

- Racing games (cars cornering)
- Flight games (banking aircraft)
- Naval games (ships turning)

### 4.6 Model Selection Guide

| Model | Order | Captures | Best For | Update Size (3D) |
|-------|-------|----------|----------|-------------------|
| Static | 0 | Nothing | Stationary entities | 12 bytes (pos only) |
| Constant Velocity | 1st | Straight-line motion | Most game entities | 24 bytes |
| Linear Acceleration | 2nd | Curved paths, accel/decel | Vehicles, projectiles | 36 bytes |
| Body-Axis Rotation | 1st+ | Turning arcs | Vehicles, aircraft | 36+ bytes |
| Polynomial (3rd+) | 3rd+ | Complex maneuvers | Military sims (rarely games) | 48+ bytes |

**Practical recommendation:** Start with first-order (constant velocity). It handles the vast majority of game scenarios. Move to second-order only if you have entities with sustained acceleration where the prediction error is noticeably worse.

---

## 5. Threshold-Based Updates

The key insight of dead reckoning is that updates are not sent on a fixed schedule. Instead, the sender monitors the divergence between its truth model and its own dead reckoned model, and sends an update only when the divergence exceeds a threshold.

### 5.1 Position Threshold

The most common threshold is Euclidean distance:

```
error = |P_truth - P_dead_reckoned|
if (error > POSITION_THRESHOLD) {
    sendUpdate();
}
```

Typical values depend on the game:

| Game Type | Typical Position Threshold | Rationale |
|-----------|---------------------------|-----------|
| FPS / competitive | 0.1 - 0.5 units | Precision matters; small threshold, more updates |
| Racing game | 0.5 - 2.0 meters | Vehicles are large; moderate tolerance |
| MMO (near) | 1.0 - 3.0 meters | Characters are small on screen; budget for many entities |
| MMO (far) | 5.0 - 20.0 meters | Distant entities tolerate more error |
| Space game | 10 - 100+ meters | Vast distances; entities are tiny on screen |

### 5.2 Orientation Threshold

Angular divergence can also trigger updates, particularly for entities where facing direction matters (turrets, characters with directional attacks):

```
angle_error = angularDistance(O_truth, O_dead_reckoned)
if (angle_error > ORIENTATION_THRESHOLD) {
    sendUpdate();
}
```

Typical values: 2-10 degrees for vehicles, 5-15 degrees for characters.

### 5.3 Heartbeat Interval

Even if the prediction never diverges (e.g., an entity moving in a perfectly straight line), periodic heartbeat updates are necessary to:

1. **Confirm entity existence.** Without updates, receivers cannot distinguish "entity moving predictably" from "entity destroyed / simulation crashed."
2. **Correct floating-point drift.** Small numerical errors accumulate over time.
3. **Handle lost packets.** If a threshold-triggered update was dropped, the heartbeat provides eventual correction.

The DIS standard uses a 5-second heartbeat with a 12-second timeout. Games typically use shorter intervals:

| Game Type | Heartbeat Interval | Entity Timeout |
|-----------|-------------------|----------------|
| Fast-paced (FPS) | 0.5 - 1.0 s | 2 - 3 s |
| Moderate (MMO) | 1.0 - 3.0 s | 5 - 10 s |
| Slow (strategy) | 3.0 - 5.0 s | 10 - 15 s |

### 5.4 Adaptive Thresholds

Static thresholds waste bandwidth on faraway entities while being too loose for nearby ones. Adaptive approaches adjust the threshold based on context:

**Distance-based:** Tighter threshold for entities close to the local player, looser for distant ones. This mirrors human perception -- a 2-meter error is noticeable at 10 meters but invisible at 500 meters.

```
threshold = BASE_THRESHOLD * (1 + distanceToViewer / REFERENCE_DISTANCE)
```

**Screen-space error:** Convert the position error to pixel error on screen. Send an update when the entity would be more than N pixels off from where it should be.

```
pixel_error = worldToScreenDistance(position_error, distanceToCamera, fov)
if (pixel_error > MAX_PIXEL_ERROR) {
    sendUpdate();
}
```

**Priority-based (Fiedler's priority accumulator):** Assign each entity a priority that accumulates over time. Higher-priority entities (nearby, in combat, player-controlled) accumulate faster and get bandwidth sooner. Glenn Fiedler describes this in his state synchronization article: "each frame we add the current priority for each object to its priority accumulator value, then sort objects in order from largest to smallest."

**Network-aware:** Increase thresholds when bandwidth is constrained or packet loss is high, trading accuracy for reliability.

### 5.5 Bandwidth Impact

The bandwidth savings from threshold-based updates are substantial. Consider 100 entities at 60 Hz with 36-byte state updates:

| Approach | Updates/sec/entity | Bandwidth (100 entities) |
|----------|-------------------|------------------------|
| Fixed 60 Hz | 60 | 216 KB/s |
| Fixed 20 Hz | 20 | 72 KB/s |
| Fixed 10 Hz | 10 | 36 KB/s |
| DR threshold (typical) | 2-8 (depends on motion) | 7-29 KB/s |
| DR threshold (mostly idle) | 0.2-1 | 0.7-3.6 KB/s |

Dead reckoning achieves its greatest savings when entities move predictably. A ship cruising in a straight line might go seconds between updates, while a fighter jet in a dogfight might update several times per second -- but still less than a fixed rate.

---

## 6. Smoothing and Convergence

When a new state update arrives, the receiver's dead reckoned position typically differs from the corrected position. The entity must be moved to the correct trajectory, but an instantaneous jump ("snap" or "warp") is visually jarring. Smoothing algorithms provide a gradual transition.

### 6.1 The Correction Problem

At the moment a correction arrives:

```
Current visual position:  Where the entity is being rendered (from previous DR)
Corrected position:       Where the new update says the entity is
Corrected trajectory:     The new DR path starting from the corrected state
Position error:           Distance between current visual and corrected position
```

The smoothing algorithm must transition from the current visual position to a point on the corrected trajectory over a **convergence period** without introducing worse artifacts than the error it is correcting.

### 6.2 Hard Snap (No Smoothing)

The simplest approach: teleport the entity to the corrected position immediately.

```typescript
entity.position = correctedPosition;
entity.velocity = correctedVelocity;
```

**When to use:** When the error is very large (entity is wildly off -- e.g., > 10 meters), when the entity is off-screen, or for non-visual entities (server-side collision proxies).

**Problem:** Visible teleportation. Characters appear to "rubber band." Completely unacceptable for entities the player is looking at during normal play.

### 6.3 Linear Interpolation (Lerp Smoothing)

Linearly interpolate between the current visual position and the target position on the corrected DR trajectory.

```
t_blend = (t - t_correction) / convergence_period    // 0.0 → 1.0
P_target(t) = deadReckon(corrected_state, t - t_correction)
P_rendered(t) = lerp(P_at_correction, P_target(t), t_blend)
```

**Pros:** Simple to implement, computationally cheap, predictable behavior.

**Cons:** Introduces a visible "kink" at the start and end of the convergence. The entity appears to momentarily change direction, then resume its course. Velocity is discontinuous at the transition points.

**Convergence period:** Typically 100-500 ms. Shorter = faster correction but more visible. Longer = smoother but entity stays wrong longer. Should be shorter than the expected update interval to avoid overlapping corrections.

### 6.4 Exponential Smoothing (Spring-Like)

Instead of blending over a fixed period, reduce the error by a constant fraction each frame. This creates natural-feeling "elastic" corrections -- large errors correct quickly, small errors correct slowly.

```
error = targetPosition - currentVisualPosition
currentVisualPosition += error * smoothingFactor * deltaTime

// Or multiplicative per-frame:
currentVisualPosition = lerp(currentVisualPosition, targetPosition, 1 - Math.pow(smoothingFactor, deltaTime))
```

Glenn Fiedler recommends this approach in his state synchronization article: maintain a position error offset that is "multiplied by some factor each frame (e.g., 0.9)" and added to the rendered position. The simulation state itself is set directly to the corrected value (no smoothing on the simulation), and only the *visual* offset is smoothed away.

**Adaptive smoothing rates:** Fiedler suggests using different rates for different error magnitudes -- e.g., a factor of 0.95 for small errors (under 25 cm) and 0.85 for large errors (over 1 meter), blending linearly between them. This prevents small corrections from causing high-frequency jitter while ensuring large corrections resolve quickly.

**Pros:** No fixed convergence period to tune. Naturally handles overlapping corrections. Frame-rate independent with the `pow` formulation.

**Cons:** Technically never reaches zero error (asymptotic). Need a "close enough" snap threshold.

### 6.5 Cubic Hermite Spline Smoothing

Uses a cubic Hermite curve to create a C1-continuous path (smooth position AND velocity) from the current state to the corrected state. This eliminates the velocity discontinuities of linear smoothing.

```
Given:
  P0 = current visual position
  V0 = current visual velocity
  P1 = target position at end of convergence period (extrapolated from corrected state)
  V1 = target velocity at end of convergence period
  t  = blend factor (0.0 → 1.0)

Hermite basis functions:
  h00 = 2t³ - 3t² + 1
  h10 = t³ - 2t² + t
  h01 = -2t³ + 3t²
  h11 = t³ - t²

P(t) = h00 * P0 + h10 * (T * V0) + h01 * P1 + h11 * (T * V1)

Where T = convergence_period (scales velocities to match the time domain)
```

**Pros:** Smooth transitions with no sudden direction changes. Physically more plausible paths. The entity appears to naturally curve toward its corrected trajectory.

**Cons:** More computationally expensive. Can overshoot in extreme cases (large velocity differences). More complex to implement.

### 6.6 Projective Velocity Blending

Rather than smoothing position directly, this technique (described in Eric Lengyel's "Believable Dead Reckoning for Networked Games") creates two projections and blends between them:

1. **Old projection:** Continue extrapolating from the old state (what the entity was doing before the correction).
2. **New projection:** Extrapolate from the new state (what the entity should be doing now).
3. **Blend:** Linearly interpolate between the two projections over the convergence period.

```
P_old(t) = deadReckon(old_state, t)
P_new(t) = deadReckon(new_state, t)
P_rendered(t) = lerp(P_old(t), P_new(t), t_blend)
```

**Pros:** Both projections are physically plausible paths, so the blend tends to produce natural-looking motion. The entity doesn't need to "correct" to a target -- it transitions between two continuous trajectories.

**Cons:** Minor oscillation artifacts when the entity makes frequent direction changes (e.g., moving in circles). Using the last known acceleration for both projections converges to the true path faster in practice.

### 6.7 The Visual Offset Pattern (Recommended)

Glenn Fiedler's recommended approach separates simulation from rendering:

1. **Simulation state:** Set directly to the corrected value. No smoothing. The physics/game logic always works with the best known state.
2. **Visual offset:** Compute the difference between the old visual position and the new simulation position. This is the "error offset."
3. **Decay the offset:** Each frame, multiply the offset by a decay factor (e.g., 0.9 per frame at 60 Hz, or use framerate-independent exponential decay).
4. **Render at:** simulation position + decaying offset.

```typescript
// On receiving correction:
const visualOffset = currentVisualPosition.sub(correctedPosition);
entity.simulationPosition = correctedPosition;
entity.simulationVelocity = correctedVelocity;
entity.visualOffset = visualOffset;

// Each frame:
entity.simulationPosition = deadReckon(entity.basisState, elapsed);
entity.visualOffset.scale(Math.pow(DECAY_RATE, deltaTime * 60));
entity.renderPosition = entity.simulationPosition.add(entity.visualOffset);
```

**Why this is recommended:** It avoids "ruining the extrapolation" (Fiedler's words). If you smooth the simulation state itself, the extrapolation runs from a wrong position with wrong velocity, creating compounding errors. By keeping simulation state crisp and only smoothing the visual, you get correct physics with smooth visuals.

### 6.8 Snap Threshold

When the error exceeds a certain magnitude, smoothing becomes counterproductive -- the entity would visibly "slide" across the screen for too long. Define a snap threshold beyond which the entity simply teleports:

```typescript
const error = correctedPosition.distanceTo(currentVisualPosition);
if (error > SNAP_THRESHOLD) {
    // Teleport immediately
    entity.renderPosition = correctedPosition;
    entity.visualOffset = Vector3.ZERO;
} else {
    // Apply smoothing
    entity.visualOffset = currentVisualPosition.sub(correctedPosition);
}
```

Typical snap thresholds: 5-20 meters (game-dependent). Some games also snap when the entity has been off-screen or when the error has persisted for too long.

### 6.9 Orientation Smoothing

Orientation corrections use the same principles but with quaternion operations:

```typescript
// Spherical linear interpolation for orientation
entity.renderOrientation = Quaternion.slerp(
    currentVisualOrientation,
    correctedOrientation,
    blendFactor
);
```

Or with the visual offset pattern:
```typescript
const orientationOffset = correctedOrientation.inverse().mul(currentVisualOrientation);
// Decay toward identity quaternion each frame
entity.orientationOffset = Quaternion.slerp(
    orientationOffset,
    Quaternion.IDENTITY,
    1 - Math.pow(DECAY_RATE, deltaTime * 60)
);
entity.renderOrientation = entity.simulationOrientation.mul(entity.orientationOffset);
```

---

## 7. Application in Different Genres

Dead reckoning's effectiveness depends heavily on entity **inertia** and motion **predictability**. The spectrum runs from highly predictable (ships, aircraft) to highly unpredictable (twitch FPS characters):

```
More Predictable (DR excels)               Less Predictable (DR struggles)
◄─────────────────────────────────────────────────────────────────────────►
Trains  Ships  Aircraft  Cars  NPCs  MMO chars  FPS chars  Fighting games
```

### 7.1 MMOs (World of Warcraft, Guild Wars 2)

MMOs are one of the most common applications of dead reckoning in commercial games. The key challenges are **scale** (hundreds of visible entities) and **low update rates** (typically 5-20 Hz).

**WoW's approach** (based on TrinityCore reverse engineering and community analysis):

- **Player movement:** Clients send movement start/stop/change events rather than continuous position. Between events, other clients extrapolate position using position + direction + speed. The server validates movement but doesn't send position every tick.
- **NPC/monster movement:** The server sends `SMSG_MONSTER_MOVE` packets containing **spline data** -- a set of waypoints that the client interpolates along. The client animates the NPC moving along the spline at the specified speed, with a configurable spline flag that predicts "500ms into the future for smoothing."
- **Threshold:** Players send updates at movement start/stop and every ~500 ms during movement.
- **Correction:** When a new position update arrives, the client blends toward the corrected position over a short window.

**Why DR works for MMOs:** Characters run at fixed speeds (walking, running, mounted), change direction relatively infrequently, and the camera is typically zoomed out enough that small errors are not noticeable. The server can also use "area of interest" (AOI) filtering to only send updates for nearby entities.

**Where DR fails in MMOs:** Instant abilities (blink, teleport, charge), PvP combat at close range, and any ability that requires precise position (AoE placement). These typically trigger forced position updates.

### 7.2 Vehicle Simulations and Racing Games

Vehicles are the ideal application for dead reckoning because of their **high inertia** -- a car at 200 km/h cannot change direction instantaneously. Even during turns, the vehicle's future position is highly predictable from its current velocity and steering angle.

**Typical approach:**

- **Model:** First-order (constant velocity) for straight sections, body-axis rotation for cornering. Some racing games use second-order with centripetal acceleration.
- **Position threshold:** 0.5 - 2.0 meters.
- **Orientation threshold:** 2 - 5 degrees.
- **Convergence period:** 200 - 500 ms.
- **Special consideration:** Collisions. When two cars collide, dead reckoning fails completely (it cannot predict the collision response). Collision events should trigger immediate forced updates.

Academic research on dead reckoning for racing games (Pantel & Wolf, 2002) found that player movement is "rarely linear in nature," motivating the use of body-axis algorithms or curvature-based prediction rather than simple linear extrapolation.

**Rocket League note:** Interestingly, Rocket League does *not* use traditional dead reckoning. As described by Jared Cone at GDC 2018, Rocket League predicts all entities (not just the local player) by **replaying the entire physics scene** at 120 TPS. This is closer to rollback netcode than dead reckoning, but was necessary because the game's physics-driven interactions (ball bounces, car-to-car collisions) make velocity-based extrapolation unreliable.

### 7.3 Flight Simulators

Flight sims were the original use case (SIMNET was built for helicopter and fixed-wing simulators). Aircraft have even higher inertia than ground vehicles and tend to make smooth, gradual maneuvers.

**Typical approach:**

- **Model:** DRM(R,V,W) or DRM(R,V,B) for maneuvering aircraft. DRM(F,V,W) for missiles.
- **Angular velocity is critical:** Banking, pitching, and yawing must be extrapolated to look correct.
- **Longer convergence periods** are acceptable because flight motion is smooth.
- **Terrain avoidance:** Dead reckoned positions must be checked against terrain to avoid entities clipping through mountains.

### 7.4 Space Games (EVE Online)

Space games present a unique environment for dead reckoning: vast distances, few collisions, and predictable orbital/thrust-based movement.

**EVE Online's approach:**

- The Tranquility server runs at **1 Hz** (one tick per second) under normal conditions.
- With Time Dilation (TiDi) during large fleet battles (1000+ players), the tick rate can drop to **0.1 Hz**.
- Ships in space move along predictable trajectories (orbit, approach, warp). The client extrapolates between ticks.
- The enormous distances mean position errors of tens of meters are invisible to the player.
- TiDi effectively solves the dead reckoning accuracy problem by slowing the simulation to match server capacity -- entities barely move between ticks, so extrapolation errors are minimal.

### 7.5 First-Person Shooters

Traditional dead reckoning is generally **not suitable** for FPS characters. Players change direction instantaneously via keyboard input, making velocity-based extrapolation unreliable.

Most modern FPS games use **entity interpolation** for remote players (rendering them slightly in the past between two known server snapshots) rather than dead reckoning (extrapolating them into the future). Gabriel Gambetta explains: in a 3D shooter, "players run, stop, and turn corners at very high speeds," making dead reckoning "essentially useless."

However, dead reckoning can still be useful for *specific entities* within an FPS:
- **Vehicles** (Battlefield, Halo) -- high inertia
- **Slow projectiles** (rocket launchers, grenades) -- ballistic extrapolation
- **Physics objects** (barrels, debris) -- short prediction windows

---

## 8. Dead Reckoning for Different Entity Types

Not all entities benefit equally from dead reckoning. The appropriate strategy depends on the entity's movement characteristics.

### 8.1 Player Characters

| Movement Type | DR Suitability | Recommended Approach |
|--------------|---------------|---------------------|
| WASD keyboard (FPS) | Poor | Entity interpolation instead |
| Click-to-move (MMO/RTS) | Good | DR during movement, snap at destination |
| Analog stick (3rd person) | Moderate | First-order DR with short convergence |
| Vehicle-mounted | Excellent | Second-order or body-axis DR |

**For WASD characters:** Consider input-based prediction rather than physics-based dead reckoning. If the player is pressing "forward," predict they continue pressing "forward" -- extrapolate along the movement direction at the character's movement speed.

### 8.2 NPCs and AI-Controlled Entities

NPCs are often *more* predictable than players:

- **Patrol routes:** If the NPC follows a known path, predict along the path (spline-based DR). This is what WoW does with `SMSG_MONSTER_MOVE`.
- **Chase behavior:** If the NPC is chasing a target, predict it continues toward the target.
- **Idle/standing:** Use zero-order (static) prediction.
- **Combat movement:** Less predictable; use standard first-order DR with tighter thresholds.

**Optimization:** Since NPC behavior is deterministic (driven by server-side AI), the server can pre-compute the NPC's path and send it as a spline. The client interpolates along the spline rather than extrapolating from velocity. This is technically not dead reckoning but achieves the same bandwidth reduction.

### 8.3 Vehicles

Vehicles are the sweet spot for dead reckoning:

| Vehicle Type | Recommended Model | Notes |
|-------------|-------------------|-------|
| Car/truck | Body-axis 1st order | Constant speed + turning = arcs |
| Aircraft | 2nd order + rotation | Acceleration changes during maneuvers |
| Ship/boat | 1st order | Very high inertia; extremely predictable |
| Motorcycle | Body-axis 1st order | Like cars but leaning is important |
| Tank | 1st order | Slow speed changes; turret rotation separate |

**Key insight:** Separate the vehicle's hull/body from its turret/weapon. The hull moves with high inertia (DR works well), but the turret can be aimed independently and changes direction unpredictably (use interpolation or tighter DR thresholds for the turret).

### 8.4 Projectiles

Projectiles are excellent candidates for dead reckoning because they follow predictable physical trajectories:

| Projectile Type | Model | Notes |
|----------------|-------|-------|
| Hitscan (bullet) | N/A | Instant; no prediction needed |
| Slow bullet | 1st order (constant velocity) | Straight line, no gravity |
| Rocket/missile | 1st order + guidance | Predict toward target |
| Grenade | 2nd order (ballistic) | Gravity provides constant acceleration |
| Arrow | 2nd order (ballistic) | Same as grenade |
| Ball (sports) | 2nd order + drag | Requires air resistance model |

**Grenade/arrow example:** Once launched, the trajectory is determined by initial velocity and gravity. The client can predict the entire flight path from a single update:

```
P(t) = P₀ + V₀ * dt + 0.5 * gravity * dt²
```

No further updates are needed until the projectile hits something or the server corrects its trajectory (e.g., due to wind or collision). This is an extremely bandwidth-efficient use of dead reckoning.

---

## 9. Common Pitfalls

### 9.1 Overshoot on Direction Changes

**The problem:** The entity is moving right at 10 m/s. The player suddenly turns left. Until the correction arrives (50-200 ms later), dead reckoning continues moving the entity *right*. When the correction arrives, the entity has to be pulled back to the left -- a visible rubber-band or snap.

**Why it happens:** Dead reckoning fundamentally assumes motion continuity. Sudden direction changes violate this assumption.

**Mitigation strategies:**
- Use tighter thresholds so corrections arrive sooner on direction changes.
- Send an immediate "direction change" event rather than waiting for the threshold to be exceeded.
- Use input-based prediction for entities with instant direction changes (characters), reserving dead reckoning for high-inertia entities.
- Apply a velocity damping/decay so extrapolated velocity gradually reduces over time, limiting overshoot.

### 9.2 Oscillation

**The problem:** The correction overshoots. The next correction corrects back. This creates a visible oscillation or wobble, especially with cubic spline smoothing.

**Why it happens:** When the convergence algorithm overshoots, the next correction has to fix both the original error AND the overshoot, creating a feedback loop.

**Mitigation strategies:**
- Use shorter convergence periods.
- Use exponential decay smoothing instead of fixed-time blending (it naturally dampens).
- Add velocity-based damping to the smoothing.
- Use the visual offset pattern (separate simulation from rendering) to prevent smoothing errors from compounding.

### 9.3 Snap-Back Artifacts (Rubber Banding)

**The problem:** The entity visibly teleports backward when a correction arrives, especially when the entity reverses direction or stops suddenly.

**Why it happens:** The dead reckoned position has moved forward of the true position. The correction has to move the entity backward in the direction of travel.

**Mitigation strategies:**
- Use the visual offset pattern with fast decay for large errors.
- Define a maximum correction distance; if exceeded, snap the entity (a big teleport is less noticeable than a long, slow slide backward).
- In third-person games, briefly speed up the camera to "catch up" rather than teleporting the character.

### 9.4 Entities Clipping Through Walls / Floors

**The problem:** Dead reckoning extrapolates position without awareness of the game world's geometry. A character running toward a wall will be extrapolated through the wall. A vehicle cresting a hill will be extrapolated into the air.

**Why it happens:** The extrapolation is a pure mathematical projection; it knows nothing about collisions, terrain, or world boundaries.

**Mitigation strategies:**
- Run lightweight collision checks on the dead reckoned position and clamp to valid positions.
- Cap extrapolation distance to prevent entities from traveling too far between updates.
- Use terrain data to project entities onto the ground (for ground-based entities).
- Glenn Fiedler notes that extrapolation "doesn't know anything about the physics simulation" and will extrapolate "cubes through floors before snapping back."

### 9.5 Stale Acceleration

**The problem:** With second-order extrapolation, if an entity was accelerating when the last update was sent but has since stopped accelerating, the stale acceleration continues to be applied, causing the entity to speed up indefinitely.

**Why it happens:** The acceleration value in the last update doesn't have an expiration date.

**Mitigation strategies:**
- Decay acceleration toward zero over time: `A_effective = A₀ * (1 - dt / MAX_ACCEL_LIFETIME)`.
- Send acceleration-stop events separately from position updates.
- Use first-order extrapolation for entities where acceleration changes frequently.
- Set a maximum extrapolation time; beyond it, freeze the entity's extrapolation at constant velocity.

### 9.6 Time Synchronization Issues

**The problem:** Dead reckoning depends on accurate timestamps to compute `dt`. If clocks are misaligned between sender and receiver, the extrapolation is off.

**Why it happens:** Client and server clocks drift. Network jitter causes variable-latency delivery.

**Mitigation strategies:**
- Implement clock synchronization (NTP-lite or custom protocol).
- Use a jitter buffer to smooth out packet delivery timing. Fiedler recommends holding packets for "4-5 frames at 60 Hz so that they come out of the buffer properly spaced apart."
- Timestamp packets with the sender's simulation tick, not wall-clock time.
- Compute effective latency as a rolling average rather than per-packet.

### 9.7 Overlapping Corrections

**The problem:** A new correction arrives before the previous correction's smoothing has completed, causing jerky motion or the entity oscillating between two targets.

**Why it happens:** High update rates combined with long convergence periods; or a burst of queued packets arriving simultaneously.

**Mitigation strategies:**
- Keep the convergence period shorter than the expected update interval.
- When a new correction arrives, start from the current visual position (not the target of the previous correction).
- The exponential smoothing / visual offset pattern handles this naturally -- each new correction simply sets a new offset; there's no "convergence period" that can overlap.

---

## 10. Pseudo-Code and Code Examples

### 10.1 Pseudo-Code: Complete Dead Reckoning System

```
=== SENDER SIDE (Entity Owner) ===

every simulation tick:
    // 1. Update truth model from physics/input
    truth_state = simulate(truth_state, input, delta_time)

    // 2. Extrapolate from last TRANSMITTED state
    dr_state = extrapolate(last_sent_state, current_time)

    // 3. Check thresholds
    position_error = distance(truth_state.position, dr_state.position)
    orientation_error = angular_diff(truth_state.orientation, dr_state.orientation)
    time_since_last_send = current_time - last_sent_time

    should_send = false
    if position_error > POSITION_THRESHOLD:
        should_send = true
    if orientation_error > ORIENTATION_THRESHOLD:
        should_send = true
    if time_since_last_send > HEARTBEAT_INTERVAL:
        should_send = true

    // 4. Send update if needed
    if should_send:
        send_state_update(truth_state, current_time)
        last_sent_state = truth_state
        last_sent_time = current_time


=== RECEIVER SIDE ===

on_state_update_received(new_state, timestamp):
    // 1. Calculate where we WERE rendering this entity
    current_visual = get_current_visual_position(entity)

    // 2. Extrapolate the new state to "now" (accounting for network latency)
    time_since_sent = estimated_current_time - timestamp
    corrected_now = extrapolate(new_state, time_since_sent)

    // 3. Compute visual offset for smoothing
    visual_offset = current_visual - corrected_now.position

    // 4. Snap if error is too large
    if length(visual_offset) > SNAP_THRESHOLD:
        visual_offset = Vector3(0, 0, 0)

    // 5. Update entity basis state
    entity.basis_state = new_state
    entity.basis_timestamp = timestamp
    entity.visual_offset = visual_offset


every render frame:
    for each dead_reckoned_entity:
        // 1. Extrapolate from basis state
        elapsed = estimated_current_time - entity.basis_timestamp
        dr_position = extrapolate(entity.basis_state, elapsed)

        // 2. Decay visual offset
        entity.visual_offset *= pow(SMOOTHING_RATE, delta_time * 60)

        // 3. Render
        render_position = dr_position + entity.visual_offset
        draw(entity, render_position)
```

### 10.2 TypeScript Implementation: Full Dead Reckoning System

```typescript
// ============================================================
// dead-reckoning.ts — Complete Dead Reckoning Implementation
// ============================================================

/** Simple 2D vector (extend to 3D as needed). */
class Vec2 {
    constructor(public x: number = 0, public y: number = 0) {}

    add(other: Vec2): Vec2 {
        return new Vec2(this.x + other.x, this.y + other.y);
    }

    sub(other: Vec2): Vec2 {
        return new Vec2(this.x - other.x, this.y - other.y);
    }

    scale(factor: number): Vec2 {
        return new Vec2(this.x * factor, this.y * factor);
    }

    length(): number {
        return Math.sqrt(this.x * this.x + this.y * this.y);
    }

    static lerp(a: Vec2, b: Vec2, t: number): Vec2 {
        return new Vec2(
            a.x + (b.x - a.x) * t,
            a.y + (b.y - a.y) * t
        );
    }

    static ZERO = new Vec2(0, 0);
}

/** Kinematic state of an entity at a point in time. */
interface EntityState {
    position: Vec2;
    velocity: Vec2;
    acceleration: Vec2;
    rotation: number;         // radians
    angularVelocity: number;  // radians/sec
    timestamp: number;        // seconds (simulation time)
}

/** Configuration for the dead reckoning system. */
interface DeadReckoningConfig {
    /** Position error threshold before an update is sent (units). */
    positionThreshold: number;
    /** Orientation error threshold before an update is sent (radians). */
    orientationThreshold: number;
    /** Maximum time between updates even if threshold is not exceeded (seconds). */
    heartbeatInterval: number;
    /** Distance beyond which the entity snaps rather than smoothing (units). */
    snapThreshold: number;
    /** Visual offset decay rate per frame at 60fps (0-1, e.g. 0.9). */
    smoothingDecayRate: number;
    /** Maximum extrapolation time before freezing (seconds). */
    maxExtrapolationTime: number;
    /** Use acceleration in extrapolation (second-order). */
    useAcceleration: boolean;
}

const DEFAULT_CONFIG: DeadReckoningConfig = {
    positionThreshold: 1.0,
    orientationThreshold: 0.1,  // ~5.7 degrees
    heartbeatInterval: 3.0,
    snapThreshold: 10.0,
    smoothingDecayRate: 0.85,
    maxExtrapolationTime: 1.0,
    useAcceleration: false,
};

// --------------------------------------------------
//  Extrapolation Functions
// --------------------------------------------------

/**
 * First-order extrapolation: constant velocity.
 * P(t) = P0 + V0 * dt
 */
function extrapolateFirstOrder(state: EntityState, dt: number): Vec2 {
    return state.position.add(state.velocity.scale(dt));
}

/**
 * Second-order extrapolation: constant acceleration.
 * P(t) = P0 + V0 * dt + 0.5 * A0 * dt^2
 */
function extrapolateSecondOrder(state: EntityState, dt: number): Vec2 {
    return state.position
        .add(state.velocity.scale(dt))
        .add(state.acceleration.scale(0.5 * dt * dt));
}

/**
 * Extrapolate rotation: linear angular velocity.
 */
function extrapolateRotation(state: EntityState, dt: number): number {
    return state.rotation + state.angularVelocity * dt;
}

/**
 * Extrapolate the full state using the configured model.
 */
function extrapolate(
    state: EntityState,
    dt: number,
    config: DeadReckoningConfig
): { position: Vec2; rotation: number } {
    // Clamp extrapolation time to prevent runaway prediction
    const clampedDt = Math.min(dt, config.maxExtrapolationTime);

    const position = config.useAcceleration
        ? extrapolateSecondOrder(state, clampedDt)
        : extrapolateFirstOrder(state, clampedDt);

    const rotation = extrapolateRotation(state, clampedDt);

    return { position, rotation };
}

// --------------------------------------------------
//  Sender: Decides When to Send Updates
// --------------------------------------------------

class DeadReckoningSender {
    private config: DeadReckoningConfig;
    private lastSentState: EntityState | null = null;
    private lastSentTime: number = 0;

    constructor(config: Partial<DeadReckoningConfig> = {}) {
        this.config = { ...DEFAULT_CONFIG, ...config };
    }

    /**
     * Called every simulation tick with the entity's true (authoritative) state.
     * Returns true if an update should be sent to remote nodes.
     */
    shouldSendUpdate(truthState: EntityState, currentTime: number): boolean {
        // First update: always send
        if (this.lastSentState === null) {
            return true;
        }

        // Heartbeat check
        const timeSinceLastSend = currentTime - this.lastSentTime;
        if (timeSinceLastSend >= this.config.heartbeatInterval) {
            return true;
        }

        // Extrapolate from last sent state
        const dt = currentTime - this.lastSentState.timestamp;
        const predicted = extrapolate(this.lastSentState, dt, this.config);

        // Position threshold
        const positionError = truthState.position.sub(predicted.position).length();
        if (positionError > this.config.positionThreshold) {
            return true;
        }

        // Orientation threshold
        const orientationError = Math.abs(
            normalizeAngle(truthState.rotation - predicted.rotation)
        );
        if (orientationError > this.config.orientationThreshold) {
            return true;
        }

        return false;
    }

    /**
     * Call this after sending the update to record the transmitted state.
     */
    onUpdateSent(sentState: EntityState, currentTime: number): void {
        this.lastSentState = { ...sentState };
        this.lastSentTime = currentTime;
    }
}

// --------------------------------------------------
//  Receiver: Extrapolates + Smooths Remote Entities
// --------------------------------------------------

class DeadReckoningReceiver {
    private config: DeadReckoningConfig;

    /** The basis state from the last received update. */
    private basisState: EntityState | null = null;

    /** Visual offset for smoothing (decays toward zero). */
    private positionOffset: Vec2 = Vec2.ZERO;
    private rotationOffset: number = 0;

    /** The last rendered position (for computing offset on correction). */
    private lastRenderedPosition: Vec2 = Vec2.ZERO;
    private lastRenderedRotation: number = 0;

    constructor(config: Partial<DeadReckoningConfig> = {}) {
        this.config = { ...DEFAULT_CONFIG, ...config };
    }

    /**
     * Called when a new state update is received from the network.
     *
     * @param newState The state from the update packet.
     * @param networkLatency Estimated one-way latency in seconds (optional,
     *        used to extrapolate the received state to "now").
     */
    onStateReceived(newState: EntityState, networkLatency: number = 0): void {
        if (this.basisState === null) {
            // First update: snap to position, no smoothing
            this.basisState = { ...newState };
            this.lastRenderedPosition = new Vec2(newState.position.x, newState.position.y);
            this.lastRenderedRotation = newState.rotation;
            this.positionOffset = Vec2.ZERO;
            this.rotationOffset = 0;
            return;
        }

        // Extrapolate the received state forward by the estimated latency
        // to approximate where the entity is "right now"
        const corrected = extrapolate(newState, networkLatency, this.config);

        // Compute the visual offset (difference between where we're rendering
        // and where the entity should be)
        this.positionOffset = this.lastRenderedPosition.sub(corrected.position);
        this.rotationOffset = normalizeAngle(
            this.lastRenderedRotation - corrected.rotation
        );

        // Snap if error is too large
        if (this.positionOffset.length() > this.config.snapThreshold) {
            this.positionOffset = Vec2.ZERO;
            this.rotationOffset = 0;
        }

        // Update basis state
        this.basisState = { ...newState };
    }

    /**
     * Called every render frame to get the position to display.
     *
     * @param currentTime The current simulation/render time in seconds.
     * @param deltaTime Time since last frame in seconds.
     * @returns The position and rotation to render the entity at.
     */
    update(
        currentTime: number,
        deltaTime: number
    ): { position: Vec2; rotation: number } | null {
        if (this.basisState === null) {
            return null; // No data received yet
        }

        // 1. Extrapolate from basis state
        const dt = currentTime - this.basisState.timestamp;
        const predicted = extrapolate(this.basisState, dt, this.config);

        // 2. Decay the visual offset (frame-rate independent)
        //    Using pow() ensures consistent decay regardless of frame rate
        const decayFactor = Math.pow(
            this.config.smoothingDecayRate,
            deltaTime * 60  // normalized to 60fps
        );
        this.positionOffset = this.positionOffset.scale(decayFactor);
        this.rotationOffset *= decayFactor;

        // 3. Snap offset to zero if negligibly small (avoid permanent micro-drift)
        if (this.positionOffset.length() < 0.001) {
            this.positionOffset = Vec2.ZERO;
        }
        if (Math.abs(this.rotationOffset) < 0.001) {
            this.rotationOffset = 0;
        }

        // 4. Compute final rendered state
        const renderPosition = predicted.position.add(this.positionOffset);
        const renderRotation = predicted.rotation + this.rotationOffset;

        // 5. Store for next correction computation
        this.lastRenderedPosition = renderPosition;
        this.lastRenderedRotation = renderRotation;

        return { position: renderPosition, rotation: renderRotation };
    }

    /**
     * Returns the raw extrapolated position (no smoothing offset).
     * Useful for gameplay logic that needs the "true" predicted position.
     */
    getSimulationPosition(currentTime: number): Vec2 | null {
        if (this.basisState === null) return null;
        const dt = currentTime - this.basisState.timestamp;
        return extrapolate(this.basisState, dt, this.config).position;
    }
}

// --------------------------------------------------
//  Utility Functions
// --------------------------------------------------

/** Normalize an angle to [-PI, PI]. */
function normalizeAngle(angle: number): number {
    while (angle > Math.PI) angle -= 2 * Math.PI;
    while (angle < -Math.PI) angle += 2 * Math.PI;
    return angle;
}

// --------------------------------------------------
//  Usage Example: Game Loop Integration
// --------------------------------------------------

/*
// === SERVER / ENTITY OWNER ===

const sender = new DeadReckoningSender({
    positionThreshold: 1.0,
    orientationThreshold: 0.1,
    heartbeatInterval: 3.0,
    useAcceleration: false,
});

function serverTick(entity: GameEntity, currentTime: number) {
    // 1. Simulate the entity (physics, input, AI)
    entity.simulate(deltaTime);

    // 2. Build the truth state
    const truthState: EntityState = {
        position: entity.position,
        velocity: entity.velocity,
        acceleration: entity.acceleration,
        rotation: entity.rotation,
        angularVelocity: entity.angularVelocity,
        timestamp: currentTime,
    };

    // 3. Check if we should send an update
    if (sender.shouldSendUpdate(truthState, currentTime)) {
        broadcastToClients({
            type: 'entity_state',
            entityId: entity.id,
            state: truthState,
        });
        sender.onUpdateSent(truthState, currentTime);
    }
}


// === CLIENT / RECEIVER ===

const receivers = new Map<string, DeadReckoningReceiver>();

function onNetworkMessage(msg: any) {
    if (msg.type === 'entity_state') {
        let receiver = receivers.get(msg.entityId);
        if (!receiver) {
            receiver = new DeadReckoningReceiver({
                smoothingDecayRate: 0.85,
                snapThreshold: 10.0,
                maxExtrapolationTime: 1.0,
                useAcceleration: false,
            });
            receivers.set(msg.entityId, receiver);
        }
        // Pass estimated one-way latency for time compensation
        receiver.onStateReceived(msg.state, estimatedLatency / 1000);
    }
}

function clientRenderFrame(currentTime: number, deltaTime: number) {
    for (const [entityId, receiver] of receivers) {
        const result = receiver.update(currentTime, deltaTime);
        if (result) {
            const sprite = getEntitySprite(entityId);
            sprite.x = result.position.x;
            sprite.y = result.position.y;
            sprite.rotation = result.rotation;
        }
    }
}
*/
```

### 10.3 TypeScript: Hermite Spline Smoothing Variant

For games that need smoother corrections than exponential decay provides:

```typescript
/**
 * Hermite spline smoothing for dead reckoning corrections.
 * Provides C1-continuous (smooth position AND velocity) transitions.
 */
class HermiteSmoothing {
    private startPos: Vec2 = Vec2.ZERO;
    private startVel: Vec2 = Vec2.ZERO;
    private endPos: Vec2 = Vec2.ZERO;
    private endVel: Vec2 = Vec2.ZERO;
    private duration: number = 0;
    private elapsed: number = 0;
    private active: boolean = false;

    /**
     * Begin a smoothing transition.
     *
     * @param currentPos   Where the entity is visually right now.
     * @param currentVel   Current visual velocity.
     * @param targetPos    Where the entity should be at the end of the blend.
     * @param targetVel    What velocity the entity should have at the end.
     * @param duration     How long the blend should take (seconds).
     */
    startBlend(
        currentPos: Vec2,
        currentVel: Vec2,
        targetPos: Vec2,
        targetVel: Vec2,
        duration: number
    ): void {
        this.startPos = currentPos;
        this.startVel = currentVel;
        this.endPos = targetPos;
        this.endVel = targetVel;
        this.duration = duration;
        this.elapsed = 0;
        this.active = true;
    }

    /**
     * Update the smoothing each frame.
     * Returns the blended position, or null if no blend is active.
     */
    update(deltaTime: number): { position: Vec2; velocity: Vec2 } | null {
        if (!this.active) return null;

        this.elapsed += deltaTime;
        let t = Math.min(this.elapsed / this.duration, 1.0);

        if (t >= 1.0) {
            this.active = false;
            return { position: this.endPos, velocity: this.endVel };
        }

        // Hermite basis functions
        const t2 = t * t;
        const t3 = t2 * t;
        const h00 = 2 * t3 - 3 * t2 + 1;
        const h10 = t3 - 2 * t2 + t;
        const h01 = -2 * t3 + 3 * t2;
        const h11 = t3 - t2;

        // Position on the Hermite curve
        // P(t) = h00*P0 + h10*(T*V0) + h01*P1 + h11*(T*V1)
        const T = this.duration;
        const position = new Vec2(
            h00 * this.startPos.x + h10 * T * this.startVel.x +
            h01 * this.endPos.x + h11 * T * this.endVel.x,
            h00 * this.startPos.y + h10 * T * this.startVel.y +
            h01 * this.endPos.y + h11 * T * this.endVel.y
        );

        // Velocity (derivative of Hermite curve)
        const dh00 = 6 * t2 - 6 * t;
        const dh10 = 3 * t2 - 4 * t + 1;
        const dh01 = -6 * t2 + 6 * t;
        const dh11 = 3 * t2 - 2 * t;

        const velocity = new Vec2(
            (dh00 * this.startPos.x + dh10 * T * this.startVel.x +
             dh01 * this.endPos.x + dh11 * T * this.endVel.x) / T,
            (dh00 * this.startPos.y + dh10 * T * this.startVel.y +
             dh01 * this.endPos.y + dh11 * T * this.endVel.y) / T
        );

        return { position, velocity };
    }

    isActive(): boolean {
        return this.active;
    }
}
```

### 10.4 TypeScript: Adaptive Threshold Sender

```typescript
/**
 * Adaptive threshold dead reckoning sender.
 * Adjusts thresholds based on distance to the nearest observer.
 */
class AdaptiveDeadReckoningSender extends DeadReckoningSender {
    private basePositionThreshold: number;
    private baseOrientationThreshold: number;
    private referenceDistance: number;

    constructor(
        config: Partial<DeadReckoningConfig> & {
            /** Distance at which the base threshold applies. */
            referenceDistance?: number;
        } = {}
    ) {
        super(config);
        this.basePositionThreshold = config.positionThreshold ?? 1.0;
        this.baseOrientationThreshold = config.orientationThreshold ?? 0.1;
        this.referenceDistance = config.referenceDistance ?? 50.0;
    }

    /**
     * Compute adaptive thresholds based on the distance to the closest
     * observer (i.e., the nearest other player).
     *
     * Closer observers get tighter thresholds (more updates).
     * Farther observers get looser thresholds (fewer updates).
     */
    getAdaptiveThresholds(distanceToNearestObserver: number): {
        positionThreshold: number;
        orientationThreshold: number;
    } {
        // Scale factor: 1.0 at reference distance, increases linearly with distance
        const scale = Math.max(
            0.25,  // minimum scale (never go below 25% of base threshold)
            distanceToNearestObserver / this.referenceDistance
        );

        return {
            positionThreshold: this.basePositionThreshold * scale,
            orientationThreshold: this.baseOrientationThreshold * scale,
        };
    }
}
```

---

## 11. Comparison with Other Techniques

### 11.1 Dead Reckoning vs Entity Interpolation

Entity interpolation renders remote entities *in the past*, between two known server snapshots. Dead reckoning renders them *in the present* (or even slightly in the future) via extrapolation.

| Aspect | Dead Reckoning (Extrapolation) | Entity Interpolation |
|--------|-------------------------------|---------------------|
| **Time domain** | Predicts the present/future | Renders the past |
| **Accuracy** | Approximate (prediction can be wrong) | Exact (uses real server data) |
| **Visual delay** | None (renders "now") | ~100 ms (one snapshot interval behind) |
| **Direction changes** | Overshoot, rubber-banding | Handled perfectly (data is historical) |
| **Bandwidth** | Low (threshold-based) | Medium-high (regular snapshots needed) |
| **CPU cost** | Low (per-entity extrapolation) | Low (per-entity lerp) |
| **Packet loss** | Graceful degradation (continues extrapolating) | Stuttering (no data to interpolate to) |
| **Best for** | Vehicles, predictable motion, large entity counts | FPS, unpredictable player movement |

**Glenn Fiedler's perspective:** In his snapshot interpolation article, Fiedler demonstrates that extrapolation "doesn't know anything about the physics simulation" and will extrapolate objects through floors before snapping back. He recommends interpolation for physics-heavy scenarios, noting that the solution to interpolation's latency penalty is to "increase the send rate" rather than switching to extrapolation.

**Gabriel Gambetta's perspective:** Gambetta presents dead reckoning as suitable for entities where "the car's heading and acceleration will remain constant" during the prediction window, but recommends entity interpolation for FPS characters because "positions and speeds can no longer be predicted from previous data."

**When to choose dead reckoning over interpolation:**
- Many entities (hundreds+) where bandwidth for frequent snapshots is prohibitive
- High-inertia entities with predictable motion (vehicles, ships)
- Games where the ~100 ms visual delay of interpolation is unacceptable for gameplay
- Peer-to-peer architectures where there is no central server sending regular snapshots

**When to choose interpolation over dead reckoning:**
- FPS or action games with twitch character movement
- Physics-heavy scenarios with collisions and complex interactions
- When visual accuracy matters more than visual timeliness
- When the server can send snapshots at a sufficient rate (20-60 Hz)

### 11.2 Dead Reckoning vs Client-Side Prediction + Reconciliation

| Aspect | Dead Reckoning | CSP + Reconciliation |
|--------|---------------|---------------------|
| **Applies to** | Remote entities | Local player |
| **Input source** | Physics state | Player commands |
| **Correction** | Smooth blend | Re-simulate from acknowledged state |
| **Complexity** | Medium | High |
| **Server trust** | Optional | Required (server is authority) |
| **Determinism** | Not required | Helpful but not required |

These are complementary, not competing techniques. A well-architected game uses CSP for the local player AND dead reckoning (or interpolation) for remote entities.

### 11.3 Dead Reckoning vs Rollback Netcode

| Aspect | Dead Reckoning | Rollback (GGPO-style) |
|--------|---------------|----------------------|
| **Scope** | Per-entity extrapolation | Full game state rollback |
| **Determinism** | Not required | Required |
| **CPU cost** | Low (simple math) | High (re-simulate N frames) |
| **Accuracy** | Approximate | Can be exact (deterministic) |
| **Correction** | Visual smoothing | Re-simulation + visual masking |
| **Best for** | Large-scale, vehicles, MMOs | Fighting games, competitive action |

### 11.4 Decision Matrix

| Scenario | Recommended Technique |
|----------|----------------------|
| Local player in FPS/action | CSP + Reconciliation |
| Remote players in FPS | Entity Interpolation |
| Vehicles (any game) | Dead Reckoning |
| NPCs with set paths | Spline-based DR or server-driven interpolation |
| Projectiles (slow) | Dead Reckoning (ballistic extrapolation) |
| MMO with 500+ visible entities | Dead Reckoning with adaptive thresholds |
| Fighting game | Rollback Netcode |
| Spectator camera | Snapshot Interpolation |
| Physics objects (barrels, crates) | Entity Interpolation or State Sync |
| Space ships in open space | Dead Reckoning (high inertia, vast distances) |

---

## 12. References

### Primary Sources

- **IEEE 1278.1** -- Standard for Distributed Interactive Simulation (DIS) Application Protocols. The formal definition of dead reckoning algorithms for distributed simulation. [IEEE Standards](https://standards.ieee.org/standard/1278_1-2012.html)

- **"Dead Reckoning: Latency Hiding for Networked Games"** -- Jesse Aronson, Gamasutra (now Game Developer), 1997. The seminal article bridging DIS dead reckoning to game development. [Game Developer](https://www.gamedeveloper.com/programming/dead-reckoning-latency-hiding-for-networked-games)

- **"Believable Dead Reckoning for Networked Games"** -- Eric Lengyel, Game Engine Gems 2 (Chapter 22), 2011. Describes projective velocity blending and improved smoothing techniques. [Taylor & Francis](https://www.taylorfrancis.com/chapters/believable-dead-reckoning-networked-games-eric-lengyel/10.1201/b11333-22)

### Gaffer on Games (Glenn Fiedler)

- **"Snapshot Interpolation"** -- Detailed comparison of interpolation vs. extrapolation for networked physics. [Gaffer on Games](https://gafferongames.com/post/snapshot_interpolation/)
- **"State Synchronization"** -- Priority accumulators, jitter buffers, visual offset smoothing, and quantization for state sync. [Gaffer on Games](https://gafferongames.com/post/state_synchronization/)
- **"Introduction to Networked Physics"** -- Overview of deterministic lockstep, snapshot interpolation, and state synchronization. [Gaffer on Games](https://gafferongames.com/post/introduction_to_networked_physics/)

### Gabriel Gambetta

- **"Fast-Paced Multiplayer Part III: Entity Interpolation"** -- Compares dead reckoning and entity interpolation; explains when each is appropriate. [Gabriel Gambetta](https://www.gabrielgambetta.com/entity-interpolation.html)

### GDC Talks

- **"It IS Rocket Science! The Physics of Rocket League"** -- Jared Cone, GDC 2018. Explains why Rocket League uses full physics rollback rather than dead reckoning. [GDC Vault](https://www.gdcvault.com/play/1024972/It-IS-Rocket-Science-The)
- **"Overwatch Gameplay Architecture and Netcode"** -- Tim Ford, Blizzard, GDC 2017. ECS architecture enabling prediction and replay. [GDC Vault](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)

### Academic Papers

- **"Accuracy in Dead-Reckoning Based Distributed Multi-Player Games"** -- Proceedings of 3rd ACM SIGCOMM Workshop on Network and System Support for Games, 2004. [ACM Digital Library](https://dl.acm.org/doi/10.1145/1016540.1016559)
- **"On the Suitability of Dead Reckoning Schemes for Games"** -- Proceedings of 1st Workshop on Network and System Support for Games, 2002. [ACM Digital Library](https://dl.acm.org/doi/10.1145/566500.566512)
- **"Dead Reckoning Using Play Patterns in a Simple 2D Multiplayer Online Game"** -- Shi, 2014. Machine learning approaches to improved dead reckoning. [Wiley / Hindawi](https://www.hindawi.com/journals/ijcgt/2014/138596/)
- **"Predictive Dead Reckoning for Online Peer-to-Peer Games"** -- Walker et al., 2023. [University of Waterloo](https://ece.uwaterloo.ca/~sl2smith/papers/2023TOG-Predictive_Dead_Reckoning.pdf)
- **"Smart Reckoning: Reducing the Traffic of Online Multiplayer Games Using Machine Learning for Movement Prediction"** -- Machine learning applied to WoW movement prediction. [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S1875952119300552)
- **"Dead Reckoning for Distributed Network Online Games"** -- Tristan Walker, MSc thesis, University of Waterloo. [UWSpace](https://uwspace.uwaterloo.ca/bitstream/handle/10012/16960/Walker_Tristan.pdf)

### Engine and Framework Documentation

- **Photon Bolt: Interpolation vs. Extrapolation** -- Practical comparison with configuration guidance. [Photon Engine](https://doc.photonengine.com/bolt/current/in-depth/interpolation-vs-extrapolation)
- **Unreal Engine: Character Movement Component Networking** -- How UE handles remote character extrapolation and smoothing. [Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/understanding-networked-movement-in-the-character-movement-component-for-unreal-engine)
- **TrinityCore: Movement** -- Reverse-engineered WoW movement system including spline-based prediction. [GitHub PR #18128](https://github.com/TrinityCore/TrinityCore/pull/18128)

### Community Resources

- **"Is Dead Reckoning Applied in Commercial Games?"** -- GameDev.net discussion. [GameDev.net](https://www.gamedev.net/forums/topic/708804-is-dead-reckoning-applied-in-commercial-games-if-so-what-are-the-variations/)
- **"Entity Movement Extrapolation"** -- Early proposal for Quake 3-era entity prediction. [gamers.org](https://www.gamers.org/dEngine/q3suggest/final/genericpred.html)

### Related Document

- **[Dead Reckoning & DIS Standard](../dead-reckoning-dis.md)** -- Companion deep-dive covering the military simulation / IEEE 1278 perspective, including the nine DIS dead reckoning algorithms, Entity State PDU structure, and DIS-specific implementation details.
