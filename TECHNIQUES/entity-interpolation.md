# Entity Interpolation

> **Disclaimer:** This content has been "vibe-researched" -- it was generated with the assistance of AI agents performing web research and synthesis. While efforts were made to fetch and cross-reference primary sources, details may be inaccurate, outdated, or misrepresented. Always verify against the original references linked throughout before relying on any technical claims.

---

## Table of Contents

1. [Overview](#overview)
2. [History and Origins](#history-and-origins)
3. [How It Works Step by Step](#how-it-works-step-by-step)
4. [The Interpolation Buffer](#the-interpolation-buffer)
5. [Interpolation vs Extrapolation](#interpolation-vs-extrapolation)
6. [Interpolation Methods](#interpolation-methods)
7. [Integration with Client-Side Prediction](#integration-with-client-side-prediction)
8. [Visual Delay and Its Implications](#visual-delay-and-its-implications)
9. [Common Pitfalls](#common-pitfalls)
10. [Pseudo-Code and Code Examples](#pseudo-code-and-code-examples)
11. [Framework Implementations](#framework-implementations)
12. [When to Use / When Not to Use](#when-to-use--when-not-to-use)
13. [References](#references)

---

## Overview

Entity interpolation is the technique of rendering remote entities (other players, NPCs, projectiles) **between two known past server snapshots** rather than at their latest known position. It solves a fundamental problem in networked games: the server sends updates at a relatively low frequency (typically 10--128 times per second), but clients render frames at a much higher rate (60--240+ fps). Without interpolation, remote entities would visibly "jump" from one snapshot position to the next.

The core insight is deceptively simple: instead of displaying remote entities at their most recent server position, the client renders them **slightly in the past**, at a point in time for which it already has two bracketing snapshots. It then smoothly blends between those snapshots, producing fluid motion with zero prediction error -- because it only ever displays positions the server has already confirmed.

**Key properties:**

| Property | Value |
|----------|-------|
| Data source | Authoritative server snapshots only (no prediction) |
| Applies to | Remote entities (other players, AI, physics objects) |
| Visual delay introduced | Typically 100--350ms depending on configuration |
| Accuracy | Perfect -- only displays confirmed server state |
| Bandwidth cost | None (uses snapshots already being sent) |
| CPU cost | Negligible (simple math per entity per frame) |

Entity interpolation is one of the "four pillars" of modern netcode -- alongside client-side prediction, server reconciliation, and lag compensation -- described in Yahn Bernier's foundational 2001 paper and Gabriel Gambetta's widely referenced tutorial series.

---

## History and Origins

### The Problem: NetQuake and the Internet (1996)

Quake (1996) was designed for LAN play, where latencies are typically under 5ms. When players tried to play over the internet -- with latencies of 150--300ms on dial-up connections -- the game became nearly unplayable. Characters would "ice skate" across the map, appearing frozen and then teleporting to new positions as delayed packets arrived. Entities updated at the server's tick rate with no smoothing, creating a profoundly jittery experience.

### QuakeWorld: The First Client-Side Prediction (December 1996)

John Carmack, with John Cash and Christian Antkow, released QuakeWorld as a direct response to internet latency. QuakeWorld introduced **client-side prediction** for the local player and rudimentary smoothing for remote entities. The local player could move immediately based on predicted physics, while other players were displayed using the latest snapshot data. This was the origin point for the snapshot-based netcode architecture that dominates multiplayer gaming to this day.

### Quake III Arena: Snapshot Interpolation (1999)

Quake III Arena refined the approach significantly. The server maintained a cycling array of the 32 most recently sent game states per client. On the client side, the game code maintained two references -- the "current" snapshot (`cg.snap`) and the "next" snapshot (`cg.nextSnap`) -- and interpolated all remote entity positions between them based on the current client time. Projectiles used extrapolation (base origin + velocity * elapsed time), while players used interpolation between known positions. The network code operated at a fixed 20 snapshots per second by default (50ms intervals), with the client rendering smoothly between them.

Fabien Sanglard's source code review describes the process: "All entities in the old snapshot are processed. This involves interpolating or extrapolating states, firing events, and adding audio and visual representation."

### Valve Source Engine: The Gold Standard (2001--2004)

Yahn Bernier's paper, *"Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization"* (GDC 2001), codified the entity interpolation approach used in Half-Life, Counter-Strike, and later the Source engine. The Source engine made the technique configurable and measurable via console variables:

- **`cl_interp`** -- The interpolation period in seconds (default: `0.1`, i.e., 100ms)
- **`cl_interp_ratio`** -- Number of server updates to keep in reserve (default: `2`)
- **`cl_updaterate`** -- Requested snapshot rate from the server (default: `20`)

The interpolation period is calculated as:

```
interpolation_period = max(cl_interp, cl_interp_ratio / cl_updaterate)
```

This formula ensures a minimum interpolation delay even if `cl_interp` is set to zero, providing packet-loss resilience. With `cl_interp_ratio = 2` and `cl_updaterate = 64`, the interpolation period is approximately 31.25ms. With the default `cl_updaterate = 20`, it is 100ms.

The Source engine also introduced `net_graph`, a real-time visualization of interpolation state, packet timing, and lerp values that became an essential debugging tool for both developers and competitive players.

### Glenn Fiedler / Gaffer on Games (2014)

Glenn Fiedler's *"Snapshot Interpolation"* article provided a rigorous treatment of the technique applied to physics simulations. His key contributions to the understanding of entity interpolation include:

- Formalizing the **jitter buffer** concept from VoIP/audio streaming for game networking
- Demonstrating that **Hermite spline interpolation** eliminates first-order discontinuities (velocity jumps) at snapshot boundaries
- Showing that buffer delay should be approximately **3x the packet send interval** to tolerate two consecutive lost packets
- Quantifying the relationship between send rate and required delay: 10 pps requires ~350ms, 30 pps ~150ms, 60 pps ~85ms

### Overwatch (2017)

Blizzard's GDC 2017 talk on *"Overwatch Gameplay Architecture and Netcode"* demonstrated a modern ECS-based implementation where the entire world state can be efficiently snapshotted and rolled back. Remote entities are interpolated on the client while the local player is predicted. The ECS architecture (components as pure data) makes it straightforward to take snapshots for interpolation and rollback for lag compensation.

---

## How It Works Step by Step

### Step 1: Server Sends Snapshots

The server simulates the game world at a fixed tick rate (e.g., 64 ticks per second). After each tick (or at a configured update rate), it serializes the relevant state of each entity -- position, rotation, velocity, animation state -- into a **snapshot** and sends it to each client over UDP.

Each snapshot includes a **server timestamp** (or tick number) so the client knows exactly what point in simulated time the snapshot represents.

```
Snapshot {
    serverTick: 1042
    serverTime: 16.281s
    entities: [
        { id: 7, position: {x: 10.5, y: 0, z: 3.2}, rotation: {qx: 0, qy: 0.707, qz: 0, qw: 0.707}, ... },
        { id: 12, position: {x: -2.1, y: 0, z: 8.7}, rotation: {...}, ... },
        ...
    ]
}
```

### Step 2: Client Buffers Snapshots

When a snapshot arrives on the client, it is **not rendered immediately**. Instead, it is placed into a **snapshot buffer** (also called an interpolation buffer or jitter buffer). The buffer is an ordered collection of snapshots sorted by server timestamp.

```
Buffer: [ snap@15.981s, snap@16.031s, snap@16.081s, snap@16.131s, snap@16.181s, snap@16.231s, snap@16.281s ]
                                                        ^                          ^
                                                   "from" snapshot            "to" snapshot
                                                        |_________ render here _________|
```

### Step 3: Calculate the Render Timestamp

Each frame, the client calculates the **render timestamp** by subtracting the interpolation delay from the estimated current server time:

```
renderTimestamp = estimatedServerTime - interpolationDelay
```

Where:
- `estimatedServerTime` is the client's best estimate of the current server clock (maintained via clock synchronization)
- `interpolationDelay` is the configured interpolation period (e.g., 100ms)

This render timestamp is always **in the past** relative to the server's current time, ensuring the client has already received the data it needs.

### Step 4: Find Bracketing Snapshots

The client searches the buffer for two consecutive snapshots that **bracket** the render timestamp:

```
snapshotA.serverTime <= renderTimestamp < snapshotB.serverTime
```

`snapshotA` is the most recent snapshot at or before the render timestamp, and `snapshotB` is the first snapshot after it.

### Step 5: Compute the Interpolation Factor

The interpolation factor `t` (a value from 0.0 to 1.0) represents how far between the two bracketing snapshots the render timestamp falls:

```
t = (renderTimestamp - snapshotA.serverTime) / (snapshotB.serverTime - snapshotA.serverTime)
```

When `t = 0.0`, the entity is at `snapshotA`'s state. When `t = 1.0`, it is at `snapshotB`'s state. For a render timestamp exactly halfway between the two, `t = 0.5`.

### Step 6: Interpolate Entity State

For each remote entity, blend between the states in the two bracketing snapshots:

```
position = lerp(snapshotA.entities[id].position, snapshotB.entities[id].position, t)
rotation = slerp(snapshotA.entities[id].rotation, snapshotB.entities[id].rotation, t)
```

The interpolated values are used for rendering. The entity appears to move smoothly between its two known positions at a constant visual rate.

### Step 7: Discard Old Snapshots

Snapshots older than the current render timestamp minus some safety margin can be discarded to prevent unbounded memory growth. Typically, the client retains at least 2--3 snapshots beyond what is currently needed for interpolation.

---

## The Interpolation Buffer

The interpolation buffer is the critical data structure that makes entity interpolation reliable in the face of real-world network conditions. It serves the same role as a **jitter buffer** in VoIP and audio streaming: absorbing variability in packet arrival times so the consumer (the renderer) sees a steady stream of data.

### Buffer Design

A typical interpolation buffer is a **sorted ring buffer** or sorted array of snapshots keyed by server timestamp. Requirements:

1. **Fast insertion**: Snapshots arrive out of order occasionally; the buffer must handle this.
2. **Fast bracketing query**: Each frame needs to find the two snapshots surrounding the render timestamp.
3. **Bounded size**: Old snapshots must be evicted to prevent memory growth.
4. **Duplicate/stale rejection**: Snapshots with timestamps older than the current render timestamp (or duplicates) should be discarded on arrival.

```
InterpolationBuffer {
    snapshots: SortedArray<Snapshot>   // sorted by serverTime
    maxSize: number                     // e.g., 30 snapshots

    addSnapshot(snapshot):
        if snapshot.serverTime <= oldestNeededTime:
            discard  // too old
        insert into sorted position
        if snapshots.length > maxSize:
            evict oldest

    getSurroundingSnapshots(renderTime) -> (snapshotA, snapshotB):
        for i in 0..snapshots.length-1:
            if snapshots[i].time <= renderTime < snapshots[i+1].time:
                return (snapshots[i], snapshots[i+1])
        return null  // buffer starvation or ahead of buffer
}
```

### Jitter Buffer Sizing

Network packets do not arrive at perfectly regular intervals. A packet sent every 50ms might arrive at 48ms, 53ms, 47ms, 55ms, etc. Worse, packets can be lost entirely. The interpolation delay must be large enough to absorb this jitter.

**Glenn Fiedler's rule of thumb:** The interpolation delay should be approximately **3x the packet send interval**. At 10 packets per second (100ms interval), this means a 300ms delay. Adding a small safety margin for jitter yields ~350ms. The reasoning: a 3x delay tolerates **two consecutive lost packets** while still having a valid "to" snapshot to interpolate toward.

| Send Rate | Interval | 3x Delay | With Jitter Margin |
|-----------|----------|----------|-------------------|
| 10 pps | 100ms | 300ms | ~350ms |
| 20 pps | 50ms | 150ms | ~170ms |
| 30 pps | 33ms | 100ms | ~115ms |
| 60 pps | 16.7ms | 50ms | ~60ms |
| 128 pps | 7.8ms | 23ms | ~30ms |

**Valve Source engine approach:** Use `cl_interp_ratio` (default: 2) to express the delay as a multiple of the update interval. `cl_interp_ratio = 2` means "keep two update intervals of buffer," tolerating one lost packet. Players with poor connections can increase this to 3 or 4.

### Adaptive Buffering

Static buffer sizing works for stable connections but is suboptimal for variable network conditions. An adaptive buffer dynamically adjusts the interpolation delay based on observed jitter:

1. **Measure jitter**: Track the variance in packet arrival intervals over a sliding window.
2. **Estimate safe delay**: Set the interpolation delay to cover a target percentile of observed jitter (e.g., 95th or 99th percentile).
3. **Smooth transitions**: When adjusting the delay, change it gradually (e.g., over 1--2 seconds) to avoid visible speed changes in interpolated entities.
4. **Floor and ceiling**: Enforce minimum (one packet interval) and maximum (e.g., 500ms) delay bounds.

The concept is borrowed directly from adaptive jitter buffers in VoIP, where playout delay must balance latency against underrun risk. The same principles apply: too little buffer causes starvation (entities freeze or snap); too much buffer adds unnecessary visual delay.

```
// Simplified adaptive buffer delay
jitterEstimate = exponentialMovingAverage(abs(actualInterval - expectedInterval))
targetDelay = max(minDelay, packetInterval + 3.0 * jitterEstimate)
currentDelay = lerp(currentDelay, targetDelay, adaptationRate * deltaTime)
```

---

## Interpolation vs Extrapolation

These two techniques address the same problem (smooth entity motion between updates) but use fundamentally different strategies.

### Interpolation: Rendering the Past

Interpolation renders entities between **two known past states**. It introduces visual delay but guarantees accuracy -- the entity is always shown at a position the server has confirmed.

```
Time axis:   ... [A]--------[B]--------[C]--------[?]
                      ^                              ^
              render here (past)              server "now"
```

**Advantages:**
- Zero prediction error; always displays confirmed server state
- Smooth, consistent motion regardless of entity behavior
- Handles sudden direction changes, stops, and complex physics correctly
- Simple to implement

**Disadvantages:**
- Introduces visual delay (typically 50--350ms)
- Remote entities are always displayed "in the past"
- Requires lag compensation for time-sensitive interactions (e.g., shooting)

### Extrapolation: Predicting the Future

Extrapolation attempts to predict where an entity **will be** based on its last known state (position + velocity, sometimes acceleration). Also called **dead reckoning** in military simulation contexts.

```
Time axis:   ... [A]--------[B]--------[?]
                              ^          ^
                     last known     render here (predicted)
```

**Advantages:**
- No additional visual delay; entities shown at predicted "present" position
- Works well for entities with predictable, linear motion (vehicles, projectiles in flight)

**Disadvantages:**
- Predictions are often wrong, especially for:
  - Player-controlled characters (unpredictable direction changes)
  - Physics objects (collisions, bounces, forces)
  - Any entity with complex behavior
- Requires "snap-back" corrections when the next real update arrives, causing visible rubber-banding
- Glenn Fiedler demonstrated this conclusively: "Extrapolation doesn't know anything about the physics simulation. Cubes extrapolate through floors, miss spring forces, and fail to predict collision responses."

### When Each Is Appropriate

| Scenario | Recommended | Rationale |
|----------|-------------|-----------|
| Remote player characters | **Interpolation** | Player movement is unpredictable; prediction errors cause rubber-banding |
| Remote physics objects | **Interpolation** | Physics is chaotic; extrapolation fails at collisions |
| Projectiles (simple ballistic) | **Extrapolation** | Trajectory is mathematically predictable |
| Vehicles (cruising) | **Either** | Linear motion extrapolates well, but sudden stops need interpolation |
| Buffer starvation fallback | **Extrapolation** | When the interpolation buffer is empty, brief extrapolation is better than freezing |
| Spectator/replay systems | **Interpolation** | No real-time pressure; maximize visual accuracy |

### Hybrid Approaches

Many production games use a combination:
- **Interpolation** as the primary technique for all remote entities
- **Extrapolation** as a brief fallback when the buffer runs dry (typically limited to 250ms, as in Source's `cl_extrapolate_amount`)
- **Visual smoothing** to blend from an extrapolated position back to the correct interpolated position when new data arrives, avoiding hard snaps

Rocket League uses a particularly interesting approach for remote players: it applies **input decay** to predicted remote movement -- 100% of the last known input on the first prediction frame, two-thirds on the second, one-third on the third, and zero by the fourth. This undershoots motion intentionally, because catching up visually looks better than overshooting and rubber-banding backward.

---

## Interpolation Methods

### Linear Interpolation (Lerp)

The most common method. For each component (x, y, z), the interpolated value is:

```
lerp(a, b, t) = a + (b - a) * t = a * (1 - t) + b * t
```

Where `t` is the interpolation factor (0.0 to 1.0).

**Properties:**
- Simple and fast
- Produces correct positions at snapshot boundaries
- **First-order discontinuity** at snapshot boundaries: velocity changes instantaneously when transitioning from one snapshot pair to the next, causing a subtle "pulsing" or jitter effect

For many games, especially those with 20+ updates per second, the first-order discontinuity of linear interpolation is imperceptible and this is the correct choice.

### Hermite Spline Interpolation

Hermite splines use **both position and velocity** at each snapshot to construct a smooth cubic curve that passes through the known points while matching velocities. This eliminates the first-order discontinuity of linear interpolation.

The cubic Hermite basis functions are:

```
h00(t) = 2t^3 - 3t^2 + 1
h10(t) = t^3 - 2t^2 + t
h01(t) = -2t^3 + 3t^2
h11(t) = t^3 - t^2
```

The interpolated position is:

```
p(t) = h00(t) * p0 + h10(t) * m0 * duration + h01(t) * p1 + h11(t) * m1 * duration
```

Where:
- `p0`, `p1` are the positions at the start and end snapshots
- `m0`, `m1` are the velocities at the start and end snapshots
- `duration` is the time between the two snapshots
- `t` is the normalized interpolation factor (0.0 to 1.0)

**Properties:**
- Smooth velocity transitions across snapshot boundaries (C1 continuity)
- Requires velocity data in each snapshot (modest bandwidth increase)
- Eliminates the "pulsing" artifacts visible with linear interpolation at low update rates
- Glenn Fiedler demonstrated this as the superior choice for physics simulations at 10 pps

**Trade-off:** Requires transmitting velocity vectors per entity per snapshot (an additional 12 bytes for a 3D velocity vector at float32 precision, or less with quantization).

### Catmull-Rom Spline

A special case of the Hermite spline where velocities are automatically estimated from surrounding snapshots rather than transmitted explicitly:

```
m0 = (p1 - p_{-1}) / (t1 - t_{-1})   // velocity estimated from surrounding points
m1 = (p2 - p0) / (t2 - t0)
```

This requires **four** snapshots (two on each side of the interpolation interval) but avoids transmitting velocity data. It trades buffer depth for bandwidth savings.

### Quaternion Slerp for Rotations

Rotations cannot be linearly interpolated component-wise (this produces incorrect orientations and non-unit quaternions). **Spherical Linear Interpolation (Slerp)** is the standard approach:

```
slerp(q0, q1, t) = q0 * sin((1-t) * omega) / sin(omega) + q1 * sin(t * omega) / sin(omega)
```

Where `omega = acos(dot(q0, q1))` is the angle between the two quaternions.

**Properties:**
- Constant angular velocity throughout the interpolation (great circle arc on the 4D unit sphere)
- No gimbal lock (unlike Euler angle interpolation)
- Does not require angular velocity transmission
- For very small angles, `nlerp` (normalized linear interpolation) is a cheaper approximation: `normalize(lerp(q0, q1, t))`

**Practical note:** Always check that `dot(q0, q1) >= 0` before slerping. If negative, negate one quaternion to take the shorter arc. Otherwise, the interpolation takes the "long way around."

### Method Comparison

| Method | Continuity | Extra Data Needed | CPU Cost | Best For |
|--------|-----------|-------------------|----------|----------|
| Linear (lerp) | C0 (position only) | None | Minimal | Most games at 20+ pps |
| Hermite spline | C1 (position + velocity) | Velocity per entity | Low | Physics sims, low update rates |
| Catmull-Rom | C1 (estimated) | 4 snapshots in buffer | Low | When velocity is unavailable |
| Slerp | C0 (angular) | None | Low-medium | Rotation (always) |

---

## Integration with Client-Side Prediction

Entity interpolation does not operate in isolation. In a typical multiplayer game, it is one half of a dual-timeline system:

### The Two Timelines

1. **Predicted timeline (present)**: The local player's character is rendered using client-side prediction. The client applies inputs immediately, simulates locally, and shows the result with zero perceived latency. When the server's authoritative state arrives, unacknowledged inputs are replayed to reconcile.

2. **Interpolated timeline (past)**: All remote entities -- other players, AI characters, physics objects -- are rendered via entity interpolation, displayed at a point in time approximately `interpolationDelay` milliseconds in the past.

This means **every player sees a slightly different version of the game world**:

```
Player A's view:
  - Player A's character: present (predicted)
  - Player B's character: ~100ms in the past (interpolated)

Player B's view:
  - Player B's character: present (predicted)
  - Player A's character: ~100ms in the past (interpolated)
```

### Why Not Predict Remote Entities?

Predicting remote players would require knowing their inputs -- which the client does not have. As the Source engine documentation notes: predicting other players would require "predicting the future with no data." Any prediction would be speculative and frequently wrong, causing constant correction artifacts (rubber-banding). Interpolation avoids this entirely by only showing confirmed data.

### The Temporal Split

The integration creates a fundamental temporal asymmetry. From any client's perspective:

```
Server time:         ──────────────────────────────────[NOW]
Local player (predicted):                               [HERE] (present)
Remote players (interpolated):              [HERE]              (past, ~100-200ms behind)
```

This temporal split is invisible to players during normal gameplay -- 100ms of visual delay for remote entities is below the threshold of perception for most human players. However, it has critical implications for time-sensitive interactions, which is why **lag compensation** exists as a companion technique.

### The Handoff Problem

When switching between predicted and interpolated timelines (e.g., a player gains/loses ownership of an entity, or spectator mode transitions), a **smoothing transition** is needed to avoid visual discontinuity. Unity's Netcode for Entities provides explicit "prediction switching" with optional smoothing during the transition.

---

## Visual Delay and Its Implications

### Quantifying the Delay

The total visual delay for a remote entity, from that player's actual input to your screen, is:

```
totalVisualDelay = inputProcessingDelay     // ~0-16ms (one frame on their client)
                 + uploadLatency             // their half-RTT to server
                 + serverProcessingDelay     // ~one tick (7.8-50ms)
                 + downloadLatency           // your half-RTT from server
                 + interpolationDelay        // your configured interp delay
                 + renderingDelay            // ~one frame (4-16ms)
```

For two players with 50ms ping each and a 100ms interpolation delay:

```
0 + 25ms + 15ms + 25ms + 100ms + 8ms = ~173ms total visual delay
```

With a 64-tick server and optimized interp (`cl_interp_ratio=2`, `cl_updaterate=64`):

```
0 + 25ms + 7.8ms + 25ms + 31.25ms + 8ms = ~97ms total visual delay
```

### Implications for Gameplay

**Hitscan weapons and shooting**: When you aim at a remote player, you are aiming at where they were ~100-200ms ago. Without lag compensation, you would need to "lead" your shots even with hitscan weapons. This is why server-side lag compensation (time rewinding) is essential -- the server processes your shot against the historical world state you actually saw.

**Dodging and reaction**: As the SnapNet article describes, if a player character is predicted but an incoming projectile is interpolated, "the projectile is displayed where it was on the server at some point in the past rather than where it will be on the server when your dodge is applied." By the time your dodge reaches the server, the projectile has already hit you in server time. This is why some games predict projectiles rather than interpolating them.

**"Shot behind walls"**: Because lag compensation rewinds time for hit detection, a target player can perceive being hit after they have already moved behind cover from their own perspective. This is an inherent trade-off of the "favor the shooter" philosophy used by Valve, Blizzard, and most FPS developers. The alternative -- not using lag compensation -- would mean shooters must lead their targets by their ping, which feels far worse.

**Competitive play**: In competitive FPS titles like Counter-Strike, players obsess over minimizing interpolation delay. Setting `cl_interp 0` and maximizing `cl_updaterate` to match the server's tick rate (64 or 128) minimizes the interp period via the formula `max(0, cl_interp_ratio / cl_updaterate)`. At 128 tick with `cl_interp_ratio = 2`, the interpolation period is only ~15.6ms.

### The Delay Budget

Game designers must understand and manage the total delay budget:

| Component | Typical Range | Can Be Reduced By |
|-----------|--------------|-------------------|
| Network latency (RTT/2 each way) | 10-100ms | Server proximity, better infrastructure |
| Server tick interval | 7.8-50ms | Higher tick rate (more CPU) |
| Interpolation delay | 15-350ms | Higher update rate, lower interp ratio |
| Client frame time | 4-16ms | Higher frame rate |

---

## Common Pitfalls

### 1. Teleportation Handling

When an entity legitimately teleports (respawn, portal, ability), interpolation will smoothly slide the entity from the old position to the new one over the interpolation period -- which looks terrible. The entity appears to "zip" across the map.

**Solution**: Include a **teleport flag** (or sequence counter) in snapshot data. When the client detects a teleport, skip interpolation for that entity and snap directly to the new position:

```
if (snapshotB.entities[id].teleportSequence !== snapshotA.entities[id].teleportSequence) {
    // Snap to new position, do not interpolate
    entity.position = snapshotB.entities[id].position;
} else {
    // Normal interpolation
    entity.position = lerp(snapshotA.entities[id].position, snapshotB.entities[id].position, t);
}
```

Alternatively, use a **distance threshold**: if the distance between two consecutive snapshot positions exceeds a reasonable maximum (e.g., more than the entity could have moved at max speed in the interval), treat it as a teleport.

### 2. Entity Spawn and Despawn

When an entity appears in snapshot B but not in snapshot A, there is no "from" state to interpolate from. Similarly, when an entity disappears, there is no "to" state.

**Solutions:**
- **Spawn**: Snap the entity into existence at its snapshot B position. Optionally, apply a visual effect (fade-in, scale-up) to mask the instant appearance.
- **Despawn**: When the entity disappears from the "to" snapshot, either snap it out immediately or continue displaying it at its last known position until the render timestamp passes the despawn time, then remove it (optionally with a fade-out).
- **Grace period**: Keep despawned entities in the buffer for a brief period so they can still be the "from" snapshot for partial interpolation.

### 3. Buffer Starvation

If the interpolation buffer has no snapshot newer than the current render timestamp (all snapshots are in the past), interpolation cannot proceed. This happens during network congestion, packet bursts, or if the interpolation delay is set too low.

**Symptoms:** Entities freeze, then snap to new positions when data arrives.

**Solutions:**
- Fall back to **brief extrapolation** (linear, using last known velocity), capped at a short duration (e.g., 250ms as in Source)
- **Increase the interpolation delay** (larger buffer)
- Implement **adaptive buffering** that detects starvation and temporarily increases delay
- Freeze the entity in place and apply a smoothing correction when data resumes (less disruptive than extrapolation for non-linear motion)

### 4. Buffer Flooding

If the network suddenly delivers a burst of delayed packets, the buffer fills with snapshots that are all older than expected. The render timestamp may "jump" forward.

**Solution:** Evict snapshots older than the current render timestamp. When the burst clears, the buffer returns to normal operation. Consider smoothly adjusting the render timestamp rather than jumping to avoid visual artifacts.

### 5. Clock Synchronization

The entire system depends on the client having an accurate estimate of the server's current time. If the client's clock drifts relative to the server's, the render timestamp will be wrong:
- **Clock too fast**: Render timestamp catches up to the latest snapshot; buffer starvation
- **Clock too slow**: Render timestamp falls further behind; unnecessary extra delay

**Solutions:**
- Use a **clock synchronization protocol** (e.g., periodic RTT measurement with NTP-style offset estimation)
- Smooth clock corrections gradually to avoid visible speed-ups or slow-downs in interpolated entities
- Some implementations adjust the interpolation playback rate slightly (e.g., 99% or 101% of real time) to gently converge on the correct time offset

### 6. Variable Tick Rate and Uneven Snapshot Spacing

If the server doesn't send snapshots at perfectly regular intervals (due to load, or because it uses a variable tick rate), the interpolation factor calculation still works correctly because it uses actual timestamps:

```
t = (renderTime - snapA.time) / (snapB.time - snapA.time)
```

However, uneven spacing can cause **aliasing artifacts**: remote players appear to speed up and slow down rhythmically. This is especially noticeable when the player's input sampling rate differs from the server's snapshot generation rate.

**Solution:** Use Hermite or Catmull-Rom interpolation, which smooths velocity across unevenly spaced samples.

### 7. Animation and State Synchronization

Position interpolation alone is insufficient. If an entity's animation state (walk cycle phase, attack animation) is not also interpolated or timed correctly, you get:
- Feet sliding (position moves but walk animation doesn't match speed)
- Mismatched visual/physical state (entity appears to be standing but has moved)

**Solution:** Synchronize animation state in snapshots and interpolate animation time alongside position. Alternatively, derive animation state from interpolated velocity on the client.

---

## Pseudo-Code and Code Examples

### General Pseudo-Code

```
// -- Server side (each tick) --
function serverTick():
    processInputs()
    simulateWorld()

    for each client:
        snapshot = captureWorldState(client.relevantEntities)
        snapshot.serverTime = currentServerTime
        snapshot.tick = currentTick
        send(client, snapshot)

// -- Client side (each render frame) --
function clientUpdate(deltaTime):
    // 1. Receive and buffer snapshots
    while hasIncomingPacket():
        snapshot = receive()
        interpolationBuffer.add(snapshot)

    // 2. Update server time estimate
    estimatedServerTime += deltaTime
    // (Periodically corrected via clock sync)

    // 3. Calculate render timestamp
    renderTime = estimatedServerTime - interpolationDelay

    // 4. Find bracketing snapshots
    (snapA, snapB) = interpolationBuffer.getBracketingSnapshots(renderTime)

    if snapA == null or snapB == null:
        // Buffer starvation -- extrapolate or freeze
        handleBufferStarvation()
        return

    // 5. Calculate interpolation factor
    t = (renderTime - snapA.serverTime) / (snapB.serverTime - snapA.serverTime)
    t = clamp(t, 0.0, 1.0)

    // 6. Interpolate each remote entity
    for each entityId in snapB.entities:
        if entityId not in snapA.entities:
            // Entity just spawned -- snap to position
            spawnEntity(entityId, snapB.entities[entityId])
            continue

        stateA = snapA.entities[entityId]
        stateB = snapB.entities[entityId]

        // Check for teleport
        if stateB.teleportSeq != stateA.teleportSeq:
            entity.position = stateB.position
            entity.rotation = stateB.rotation
        else:
            entity.position = lerp(stateA.position, stateB.position, t)
            entity.rotation = slerp(stateA.rotation, stateB.rotation, t)

    // 7. Handle entities that despawned
    for each entityId in snapA.entities:
        if entityId not in snapB.entities and t >= 1.0:
            despawnEntity(entityId)

    // 8. Prune old snapshots
    interpolationBuffer.removeOlderThan(renderTime - safetyMargin)
```

### TypeScript Implementation

The following is a practical, self-contained TypeScript implementation suitable for a browser-based multiplayer game:

```typescript
// ============================================================
// Entity Interpolation System -- TypeScript Implementation
// ============================================================

interface Vec3 {
    x: number;
    y: number;
    z: number;
}

interface Quat {
    x: number;
    y: number;
    z: number;
    w: number;
}

interface EntityState {
    position: Vec3;
    rotation: Quat;
    velocity?: Vec3;          // Optional: needed for Hermite interpolation
    teleportSequence: number; // Increment on teleport to skip interpolation
}

interface Snapshot {
    serverTime: number;       // Server timestamp in seconds
    tick: number;             // Server tick number
    entities: Map<number, EntityState>;  // entityId -> state
}

// ---- Math Utilities ----

function lerpScalar(a: number, b: number, t: number): number {
    return a + (b - a) * t;
}

function lerpVec3(a: Vec3, b: Vec3, t: number): Vec3 {
    return {
        x: lerpScalar(a.x, b.x, t),
        y: lerpScalar(a.y, b.y, t),
        z: lerpScalar(a.z, b.z, t),
    };
}

function quatDot(a: Quat, b: Quat): number {
    return a.x * b.x + a.y * b.y + a.z * b.z + a.w * b.w;
}

function slerpQuat(a: Quat, b: Quat, t: number): Quat {
    // Ensure shortest path
    let dot = quatDot(a, b);
    let bAdj = b;
    if (dot < 0) {
        dot = -dot;
        bAdj = { x: -b.x, y: -b.y, z: -b.z, w: -b.w };
    }

    // If quaternions are very close, fall back to normalized lerp
    if (dot > 0.9995) {
        const result: Quat = {
            x: lerpScalar(a.x, bAdj.x, t),
            y: lerpScalar(a.y, bAdj.y, t),
            z: lerpScalar(a.z, bAdj.z, t),
            w: lerpScalar(a.w, bAdj.w, t),
        };
        const mag = Math.sqrt(result.x ** 2 + result.y ** 2 + result.z ** 2 + result.w ** 2);
        return { x: result.x / mag, y: result.y / mag, z: result.z / mag, w: result.w / mag };
    }

    const omega = Math.acos(dot);
    const sinOmega = Math.sin(omega);
    const factorA = Math.sin((1 - t) * omega) / sinOmega;
    const factorB = Math.sin(t * omega) / sinOmega;

    return {
        x: factorA * a.x + factorB * bAdj.x,
        y: factorA * a.y + factorB * bAdj.y,
        z: factorA * a.z + factorB * bAdj.z,
        w: factorA * a.w + factorB * bAdj.w,
    };
}

function vec3Distance(a: Vec3, b: Vec3): number {
    const dx = a.x - b.x;
    const dy = a.y - b.y;
    const dz = a.z - b.z;
    return Math.sqrt(dx * dx + dy * dy + dz * dz);
}

/**
 * Hermite spline interpolation for Vec3.
 * Requires velocity at both endpoints.
 */
function hermiteVec3(
    p0: Vec3, v0: Vec3,
    p1: Vec3, v1: Vec3,
    duration: number, t: number
): Vec3 {
    const t2 = t * t;
    const t3 = t2 * t;

    const h00 = 2 * t3 - 3 * t2 + 1;
    const h10 = t3 - 2 * t2 + t;
    const h01 = -2 * t3 + 3 * t2;
    const h11 = t3 - t2;

    return {
        x: h00 * p0.x + h10 * v0.x * duration + h01 * p1.x + h11 * v1.x * duration,
        y: h00 * p0.y + h10 * v0.y * duration + h01 * p1.y + h11 * v1.y * duration,
        z: h00 * p0.z + h10 * v0.z * duration + h01 * p1.z + h11 * v1.z * duration,
    };
}

// ---- Interpolation Buffer ----

class InterpolationBuffer {
    private snapshots: Snapshot[] = [];
    private maxSnapshots: number;

    constructor(maxSnapshots: number = 30) {
        this.maxSnapshots = maxSnapshots;
    }

    /** Insert a snapshot in sorted order by serverTime. */
    addSnapshot(snapshot: Snapshot): void {
        // Binary search for insertion point
        let lo = 0;
        let hi = this.snapshots.length;
        while (lo < hi) {
            const mid = (lo + hi) >> 1;
            if (this.snapshots[mid].serverTime < snapshot.serverTime) {
                lo = mid + 1;
            } else {
                hi = mid;
            }
        }

        // Reject exact duplicates
        if (lo < this.snapshots.length && this.snapshots[lo].serverTime === snapshot.serverTime) {
            return;
        }

        this.snapshots.splice(lo, 0, snapshot);

        // Evict oldest if over capacity
        while (this.snapshots.length > this.maxSnapshots) {
            this.snapshots.shift();
        }
    }

    /**
     * Find the two snapshots that bracket the given render time.
     * Returns null if not enough data is available.
     */
    getBracketingSnapshots(renderTime: number): [Snapshot, Snapshot] | null {
        for (let i = 0; i < this.snapshots.length - 1; i++) {
            if (this.snapshots[i].serverTime <= renderTime &&
                renderTime < this.snapshots[i + 1].serverTime) {
                return [this.snapshots[i], this.snapshots[i + 1]];
            }
        }
        return null;
    }

    /** Remove snapshots older than the given time, keeping at least `minKeep`. */
    pruneOlderThan(time: number, minKeep: number = 2): void {
        while (this.snapshots.length > minKeep && this.snapshots[0].serverTime < time) {
            this.snapshots.shift();
        }
    }

    /** Get the most recent snapshot (for extrapolation fallback). */
    getLatestSnapshot(): Snapshot | null {
        return this.snapshots.length > 0 ? this.snapshots[this.snapshots.length - 1] : null;
    }

    /** Get the number of buffered snapshots. */
    get count(): number {
        return this.snapshots.length;
    }
}

// ---- Clock Synchronization (simplified) ----

class ServerClock {
    private offset: number = 0;           // Local time + offset = estimated server time
    private smoothOffset: number = 0;     // Gradually approaches offset
    private smoothingRate: number = 0.1;  // How fast to converge (per second)

    /** Call when a clock sync round-trip completes. */
    onSyncResponse(localSendTime: number, serverTime: number, localReceiveTime: number): void {
        const rtt = localReceiveTime - localSendTime;
        const oneWay = rtt / 2;
        const estimatedServerNow = serverTime + oneWay;
        this.offset = estimatedServerNow - localReceiveTime;
    }

    /** Update each frame to smooth clock corrections. */
    update(deltaTime: number): void {
        // Gradually converge smoothOffset toward offset
        const diff = this.offset - this.smoothOffset;
        this.smoothOffset += diff * Math.min(1.0, this.smoothingRate * deltaTime);
    }

    /** Get the estimated current server time. */
    getServerTime(localTime: number): number {
        return localTime + this.smoothOffset;
    }
}

// ---- Entity Interpolation System ----

interface InterpolatedEntity {
    id: number;
    position: Vec3;
    rotation: Quat;
    isActive: boolean;
}

interface InterpolationConfig {
    /** Interpolation delay in seconds (e.g., 0.1 for 100ms). */
    interpolationDelay: number;

    /** Maximum distance between snapshots before treating as teleport. */
    teleportThreshold: number;

    /** Whether to use Hermite interpolation (requires velocity in snapshots). */
    useHermite: boolean;

    /** Maximum extrapolation duration in seconds when buffer is starved. */
    maxExtrapolationTime: number;
}

const DEFAULT_CONFIG: InterpolationConfig = {
    interpolationDelay: 0.1,
    teleportThreshold: 10.0,
    useHermite: false,
    maxExtrapolationTime: 0.25,
};

class EntityInterpolationSystem {
    private buffer: InterpolationBuffer;
    private clock: ServerClock;
    private config: InterpolationConfig;
    private entities: Map<number, InterpolatedEntity> = new Map();
    private lastRenderTime: number = 0;
    private extrapolationStart: number = 0;
    private isExtrapolating: boolean = false;

    constructor(clock: ServerClock, config: Partial<InterpolationConfig> = {}) {
        this.config = { ...DEFAULT_CONFIG, ...config };
        this.buffer = new InterpolationBuffer();
        this.clock = clock;
    }

    /** Called when a snapshot arrives from the server. */
    onSnapshotReceived(snapshot: Snapshot): void {
        this.buffer.addSnapshot(snapshot);
    }

    /** Called each render frame. Returns the map of interpolated entity states. */
    update(localTime: number, deltaTime: number): Map<number, InterpolatedEntity> {
        this.clock.update(deltaTime);

        const estimatedServerTime = this.clock.getServerTime(localTime);
        const renderTime = estimatedServerTime - this.config.interpolationDelay;
        this.lastRenderTime = renderTime;

        const brackets = this.buffer.getBracketingSnapshots(renderTime);

        if (brackets !== null) {
            this.isExtrapolating = false;
            const [snapA, snapB] = brackets;
            this.interpolateBetween(snapA, snapB, renderTime);
        } else {
            // Buffer starvation: attempt brief extrapolation
            this.handleBufferStarvation(renderTime);
        }

        // Prune old snapshots
        this.buffer.pruneOlderThan(renderTime - this.config.interpolationDelay);

        return this.entities;
    }

    private interpolateBetween(snapA: Snapshot, snapB: Snapshot, renderTime: number): void {
        const duration = snapB.serverTime - snapA.serverTime;
        if (duration <= 0) return;

        const t = Math.max(0, Math.min(1, (renderTime - snapA.serverTime) / duration));

        // Track which entities are present in snapB
        const activeIds = new Set<number>();

        for (const [entityId, stateB] of snapB.entities) {
            activeIds.add(entityId);
            const stateA = snapA.entities.get(entityId);

            let entity = this.entities.get(entityId);
            if (!entity) {
                entity = {
                    id: entityId,
                    position: { x: 0, y: 0, z: 0 },
                    rotation: { x: 0, y: 0, z: 0, w: 1 },
                    isActive: true,
                };
                this.entities.set(entityId, entity);
            }
            entity.isActive = true;

            if (!stateA) {
                // Entity just spawned -- snap to position
                entity.position = { ...stateB.position };
                entity.rotation = { ...stateB.rotation };
                continue;
            }

            // Check for teleport
            const isTeleport =
                stateA.teleportSequence !== stateB.teleportSequence ||
                vec3Distance(stateA.position, stateB.position) > this.config.teleportThreshold;

            if (isTeleport) {
                entity.position = { ...stateB.position };
                entity.rotation = { ...stateB.rotation };
                continue;
            }

            // Interpolate position
            if (this.config.useHermite && stateA.velocity && stateB.velocity) {
                entity.position = hermiteVec3(
                    stateA.position, stateA.velocity,
                    stateB.position, stateB.velocity,
                    duration, t
                );
            } else {
                entity.position = lerpVec3(stateA.position, stateB.position, t);
            }

            // Interpolate rotation (always slerp)
            entity.rotation = slerpQuat(stateA.rotation, stateB.rotation, t);
        }

        // Mark entities not in snapB as inactive
        for (const [entityId, entity] of this.entities) {
            if (!activeIds.has(entityId)) {
                entity.isActive = false;
            }
        }
    }

    private handleBufferStarvation(renderTime: number): void {
        const latest = this.buffer.getLatestSnapshot();
        if (!latest) return;

        if (!this.isExtrapolating) {
            this.isExtrapolating = true;
            this.extrapolationStart = renderTime;
        }

        const extrapolationDuration = renderTime - this.extrapolationStart;
        if (extrapolationDuration > this.config.maxExtrapolationTime) {
            // Exceeded max extrapolation -- freeze entities
            return;
        }

        const elapsed = renderTime - latest.serverTime;

        for (const [entityId, state] of latest.entities) {
            let entity = this.entities.get(entityId);
            if (!entity) continue;

            if (state.velocity) {
                // Linear extrapolation using last known velocity
                entity.position = {
                    x: state.position.x + state.velocity.x * elapsed,
                    y: state.position.y + state.velocity.y * elapsed,
                    z: state.position.z + state.velocity.z * elapsed,
                };
            }
            // Rotation: no extrapolation (hold last known rotation)
        }
    }

    /** Get the current interpolation delay being used. */
    getInterpolationDelay(): number {
        return this.config.interpolationDelay;
    }

    /** Get the current render timestamp. */
    getRenderTime(): number {
        return this.lastRenderTime;
    }

    /** Get the number of snapshots in the buffer. */
    getBufferSize(): number {
        return this.buffer.count;
    }
}

// ---- Usage Example ----

/*
// Setup
const clock = new ServerClock();
const interpSystem = new EntityInterpolationSystem(clock, {
    interpolationDelay: 0.1,     // 100ms delay
    teleportThreshold: 10.0,     // 10 units = teleport
    useHermite: false,            // Use linear interpolation
    maxExtrapolationTime: 0.25,  // 250ms max extrapolation
});

// When receiving a snapshot from the server:
socket.onmessage = (data) => {
    const snapshot = deserializeSnapshot(data);
    interpSystem.onSnapshotReceived(snapshot);
};

// When a clock sync response arrives:
socket.onClockSync = (localSendTime, serverTime, localReceiveTime) => {
    clock.onSyncResponse(localSendTime, serverTime, localReceiveTime);
};

// Each frame:
function gameLoop(localTime: number, deltaTime: number) {
    // Update local player via client-side prediction (not shown)
    updateLocalPlayerPrediction();

    // Update all remote entities via interpolation
    const entities = interpSystem.update(localTime, deltaTime);

    // Render
    for (const [id, entity] of entities) {
        if (entity.isActive) {
            renderEntity(id, entity.position, entity.rotation);
        }
    }
}
*/
```

### Adaptive Buffer Delay (TypeScript)

```typescript
/**
 * Adaptive interpolation delay that adjusts based on observed jitter.
 * Inspired by adaptive jitter buffers in VoIP systems.
 */
class AdaptiveInterpolationDelay {
    private packetInterval: number;       // Expected interval between snapshots (seconds)
    private jitterEstimate: number = 0;   // Exponential moving average of jitter
    private jitterAlpha: number = 0.05;   // EMA smoothing factor (lower = smoother)
    private targetDelay: number;
    private currentDelay: number;
    private minDelay: number;
    private maxDelay: number;
    private convergenceRate: number = 2.0; // How fast currentDelay approaches targetDelay

    private lastArrivalTime: number = -1;

    constructor(
        packetInterval: number,
        options: {
            minDelay?: number;
            maxDelay?: number;
            jitterMultiplier?: number;
        } = {}
    ) {
        this.packetInterval = packetInterval;
        this.minDelay = options.minDelay ?? packetInterval;
        this.maxDelay = options.maxDelay ?? packetInterval * 10;
        this.targetDelay = packetInterval * 3; // Start with 3x (Glenn Fiedler's rule)
        this.currentDelay = this.targetDelay;
    }

    /** Call when a snapshot arrives. */
    onSnapshotReceived(arrivalTime: number): void {
        if (this.lastArrivalTime >= 0) {
            const actualInterval = arrivalTime - this.lastArrivalTime;
            const jitter = Math.abs(actualInterval - this.packetInterval);

            // Update jitter estimate (exponential moving average)
            this.jitterEstimate =
                this.jitterAlpha * jitter + (1 - this.jitterAlpha) * this.jitterEstimate;

            // Target delay: one interval + 3 standard deviations of jitter
            this.targetDelay = Math.max(
                this.minDelay,
                Math.min(this.maxDelay, this.packetInterval + 3.0 * this.jitterEstimate)
            );
        }
        this.lastArrivalTime = arrivalTime;
    }

    /** Call each frame to smoothly converge toward the target delay. */
    update(deltaTime: number): number {
        const diff = this.targetDelay - this.currentDelay;
        this.currentDelay += diff * Math.min(1.0, this.convergenceRate * deltaTime);
        return this.currentDelay;
    }

    /** Get the current interpolation delay. */
    getDelay(): number {
        return this.currentDelay;
    }

    /** Get diagnostic info. */
    getDiagnostics(): { jitter: number; target: number; current: number } {
        return {
            jitter: this.jitterEstimate,
            target: this.targetDelay,
            current: this.currentDelay,
        };
    }
}
```

---

## Framework Implementations

### Valve Source Engine

The Source engine's entity interpolation is the most thoroughly documented implementation, thanks to publicly available console variables and Yahn Bernier's paper.

**Key details:**
- Tick rates: 66.67 (default) or 128 (competitive CS)
- Default update rate: 20 snapshots/sec (`cl_updaterate 20`)
- Default interpolation: 100ms (`cl_interp 0.1`)
- Formula: `interpolation_period = max(cl_interp, cl_interp_ratio / cl_updaterate)`
- Extrapolation fallback: linear, capped at 250ms (`cl_extrapolate_amount 0.25`)
- Lag compensation formula: `command_execution_time = server_time - packet_latency - client_interp`
- Server maintains 1-second position history for all players for lag compensation rewinding
- Prediction error smoothing: gradual visual correction over `cl_smoothtime` seconds
- Debug tools: `net_graph 1` shows real-time lerp, choke, loss, and timing

**Console variables:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `cl_interp` | 0.1 | Minimum interpolation period (seconds) |
| `cl_interp_ratio` | 2 | Minimum snapshots to buffer |
| `cl_updaterate` | 20 | Requested snapshots/second |
| `cl_interpolate` | 1 | Enable/disable interpolation |
| `cl_extrapolate` | 1 | Enable extrapolation fallback |
| `cl_extrapolate_amount` | 0.25 | Max extrapolation duration (seconds) |
| `cl_smooth` | 1 | Smooth prediction errors |
| `cl_smoothtime` | 0.1 | Prediction error smoothing duration |

### Unity Netcode for Entities

Unity's Netcode for Entities (the DOTS-based networking stack) implements interpolation as a core timeline concept.

**Key details:**
- Three concurrent timelines: server present, client predicted, client interpolated
- `NetworkTime.InterpolationTick` and `InterpolationTickFraction` provide the interpolated time reference
- Buffer depth configured via `ClientTickRate.InterpolationTimeMS`
- Entities are designated as "interpolated ghosts" or "predicted ghosts" via Ghost Mode
- Extrapolation uses "unclamped interpolation" -- continuing linearly beyond the last known snapshot
- Explicit "prediction switching" support with optional smoothing for entities that transition between predicted and interpolated timelines
- Interpolated ghosts spawn on appropriate interpolation ticks, not when snapshot data arrives

### Unity Netcode for GameObjects

The older, MonoBehaviour-based Netcode for GameObjects uses `BufferedLinearInterpolator<T>`:
- Buffers measurements via `AddMeasurement()` calls
- Interpolation time is configured through `NetworkTimeSystem`
- Provides both clamped and unclamped interpolation modes
- Separate handling for position (Vector3) and rotation (Quaternion) types

### Unreal Engine

Unreal uses **network smoothing** on `CharacterMovementComponent` for simulated proxies (remote players):

**Three smoothing modes** (`ENetworkSmoothingMode`):
- **Disabled**: No smoothing; positions snap on each replicated update
- **Linear**: Constant-rate interpolation from source position to target over `NetworkSimulatedSmoothLocationTime` seconds
- **Exponential**: Variable-rate smoothing that moves faster when far from the target and decelerates near it (the default for most projects)

**Implementation details:**
- Actors replicate movement via `FRepMovement` (not raw transforms)
- `bReplicateMovement` flag enables movement replication per actor
- `SmoothClientPosition` handles the interpolation in the movement component
- `NetworkSimulatedSmoothLocationTime` controls location smoothing duration
- `NetworkSimulatedSmoothRotationTime` controls rotation smoothing duration
- Local player uses full CSP with correction smoothing; remote players use the smoothing modes above

**Notable difference from snapshot interpolation:** Unreal's default approach smooths from the current visual position toward the latest replicated position, rather than interpolating between two past snapshots. This is conceptually closer to exponential decay toward a target than true buffered interpolation, which means it doesn't introduce a fixed delay but may overshoot or undershoot during direction changes.

### Quake III Arena

The original reference implementation:
- Server maintains a cycling array of 32 snapshots per client
- Client maintains `cg.snap` (current) and `cg.nextSnap` (next) references
- Interpolation factor derived from client time relative to snapshot timestamps
- Entity positions interpolated between `cg.snap` and `cg.nextSnap`
- Projectiles extrapolated using origin + velocity * time
- Delta compression: only changed fields are transmitted, using pre-constructed field metadata (`netField_t`) for blind field-by-field comparison
- Messages pre-fragmented to 1400 bytes to avoid router-level fragmentation

### Comparison

| Feature | Source Engine | Unity NGO | Unity Netcode (Entities) | Unreal Engine | Quake III |
|---------|-------------|-----------|------------------------|---------------|-----------|
| **Approach** | Buffered snapshot interpolation | Buffered linear interpolation | Buffered snapshot interpolation | Exponential/linear smoothing toward latest | Snapshot interpolation |
| **Configurable delay** | Yes (cl_interp) | Yes (NetworkTimeSystem) | Yes (InterpolationTimeMS) | Indirect (smoothing time) | Fixed (via tick rate) |
| **Extrapolation fallback** | Yes (cl_extrapolate) | Yes (unclamped) | Yes (unclamped) | Implicit (smoothing) | Yes (for projectiles) |
| **Hermite/spline** | No (linear only) | No (linear only) | No (linear only) | No | No |
| **Rotation method** | Quaternion slerp | Quaternion slerp | Quaternion slerp | Quaternion slerp | Euler lerp |
| **Debug tools** | net_graph | NetworkDebugger | GhostStats | Network Profiler | None built-in |

---

## When to Use / When Not to Use

### Always Use Entity Interpolation When:

- **Rendering remote player characters** in any real-time multiplayer game. This is the primary and most universal use case. Virtually every shipped multiplayer game with real-time movement uses some form of entity interpolation for remote players.

- **Rendering server-authoritative physics objects** (e.g., balls, crates, vehicles driven by other players). These objects have unpredictable motion that cannot be reliably extrapolated.

- **Spectator and replay systems**. There is no real-time pressure, so the delay is irrelevant, and you want maximum visual fidelity.

- **Any entity whose motion is non-deterministic or input-driven by a remote player**. The fundamental principle: if you cannot predict it accurately, interpolate it.

### Consider Alternatives When:

- **The local player's own character**: Use client-side prediction + server reconciliation instead. The local player should see immediate response to their inputs.

- **Simple ballistic projectiles** (bullets, grenades in flight): These follow predictable physics trajectories that can be accurately extrapolated or locally simulated. Predicting them reduces the temporal gap between the player's predicted position and the projectile's displayed position, making dodging feel responsive.

- **Turn-based or low-frequency games**: If the game only updates every second or more, interpolation would introduce unacceptable delay. Consider animating transitions client-side instead.

- **Deterministic lockstep games** (RTS, some fighting games): These run identical simulations on all clients from shared inputs. There is no "remote entity" to interpolate -- all entities are simulated locally. Rollback netcode handles mispredictions.

- **Entities with perfect client-side prediction**: If an entity's behavior is fully deterministic given known inputs (e.g., the local player's pet that follows deterministic AI), it can be predicted rather than interpolated.

### The Default Choice

When in doubt, use entity interpolation for remote entities. It is the safest, most widely proven technique. The ~100ms visual delay it introduces is a small price for guaranteed smooth, artifact-free motion. Every major multiplayer framework (Source, Unreal, Unity, Godot addons, Photon, FishNet, Mirror) either implements it directly or provides the primitives to build it.

---

## References

### Primary Sources

1. **Gabriel Gambetta** -- *Fast-Paced Multiplayer, Part III: Entity Interpolation*
   [https://www.gabrielgambetta.com/entity-interpolation.html](https://www.gabrielgambetta.com/entity-interpolation.html)

2. **Gabriel Gambetta** -- *Fast-Paced Multiplayer, Part IV: Lag Compensation*
   [https://www.gabrielgambetta.com/lag-compensation.html](https://www.gabrielgambetta.com/lag-compensation.html)

3. **Glenn Fiedler (Gaffer on Games)** -- *Snapshot Interpolation*
   [https://gafferongames.com/post/snapshot_interpolation/](https://gafferongames.com/post/snapshot_interpolation/)

4. **Yahn Bernier (Valve)** -- *Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization* (GDC 2001)
   [https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization)

5. **Valve Developer Community** -- *Source Multiplayer Networking*
   [https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)

6. **Valve Developer Community** -- *Interpolation*
   [https://developer.valvesoftware.com/wiki/Interpolation](https://developer.valvesoftware.com/wiki/Interpolation)

7. **SnapNet** -- *Netcode Architectures Part 3: Snapshot Interpolation*
   [https://www.snapnet.dev/blog/netcode-architectures-part-3-snapshot-interpolation/](https://www.snapnet.dev/blog/netcode-architectures-part-3-snapshot-interpolation/)

### Engine Documentation

8. **Unity** -- *Interpolation (Netcode for Entities)*
   [https://docs.unity3d.com/Packages/com.unity.netcode@1.3/manual/interpolation.html](https://docs.unity3d.com/Packages/com.unity.netcode@1.3/manual/interpolation.html)

9. **Unity** -- *BufferedLinearInterpolator (Netcode for GameObjects)*
   [https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@1.0/api/Unity.Netcode.BufferedLinearInterpolator-1.html](https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@1.0/api/Unity.Netcode.BufferedLinearInterpolator-1.html)

10. **Unreal Engine** -- *Understanding Networked Movement in the Character Movement Component*
    [https://docs.unrealengine.com/5.3/en-US/understanding-networked-movement-in-the-character-movement-component-for-unreal-engine/](https://docs.unrealengine.com/5.3/en-US/understanding-networked-movement-in-the-character-movement-component-for-unreal-engine/)

11. **Unreal Engine** -- *ENetworkSmoothingMode*
    [https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/ENetworkSmoothingMode](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/ENetworkSmoothingMode)

### Source Code Analysis

12. **Fabien Sanglard** -- *Quake 3 Source Code Review: Network Model*
    [https://fabiensanglard.net/quake3/network.php](https://fabiensanglard.net/quake3/network.php)

13. **Quake 3 Networking Primer** (Unlagged documentation)
    [https://www.ra.is/unlagged/network.html](https://www.ra.is/unlagged/network.html)

### Conference Talks

14. **Timothy Ford (Blizzard)** -- *Overwatch Gameplay Architecture and Netcode* (GDC 2017)
    [https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)

### Additional Resources

15. **GitHub Gist** -- *About cl_interp* (comprehensive Source engine interp reference)
    [https://gist.github.com/ribasco/046c333024db9e75b2e4f314baa11799](https://gist.github.com/ribasco/046c333024db9e75b2e4f314baa11799)

16. **Wikipedia** -- *Cubic Hermite Spline*
    [https://en.wikipedia.org/wiki/Cubic_Hermite_spline](https://en.wikipedia.org/wiki/Cubic_Hermite_spline)

17. **Wikipedia** -- *Spherical Linear Interpolation (Slerp)*
    [https://en.wikipedia.org/wiki/Slerp](https://en.wikipedia.org/wiki/Slerp)

18. **Ken Shoemake** -- *Animating Rotation with Quaternion Curves* (SIGGRAPH 1985)
    Referenced via: [http://number-none.com/product/Understanding%20Slerp,%20Then%20Not%20Using%20It/](http://number-none.com/product/Understanding%20Slerp,%20Then%20Not%20Using%20It/)
