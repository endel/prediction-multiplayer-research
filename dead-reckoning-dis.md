# Dead Reckoning and the IEEE 1278 DIS Standard

## Table of Contents

1. [History of DIS](#1-history-of-dis)
2. [What Dead Reckoning Is and Why It Matters](#2-what-dead-reckoning-is-and-why-it-matters)
3. [The Three Models](#3-the-three-models-truth-dead-reckoned-and-smoothing)
4. [The Dead Reckoning Algorithms](#4-the-dead-reckoning-algorithms)
5. [Threshold-Based Updates](#5-threshold-based-updates)
6. [Heartbeat Updates](#6-heartbeat-updates)
7. [Entity State PDU Structure](#7-entity-state-pdu-structure)
8. [Smoothing Techniques for Corrections](#8-smoothing-techniques-for-corrections)
9. [Application to Games](#9-how-dead-reckoning-applies-to-games)
10. [Implementation Pseudo-Code](#10-implementation-pseudo-code)
11. [Comparison to Game-Specific Prediction Approaches](#11-comparison-to-game-specific-prediction-approaches)
12. [Modern Relevance and Applications](#12-modern-relevance-and-applications)
13. [References](#13-references)

---

## 1. History of DIS

### Origins in Military Simulation

The concept of networked, distributed simulation for military training originated in the late 1970s and early 1980s. In 1978, Captain Jack Thorpe of the U.S. Air Force articulated a vision for real-time, networked combat simulation that would allow geographically dispersed participants to engage in shared virtual battlefields. The Defense Advanced Research Projects Agency (DARPA) took interest and funded research to determine whether such a system was feasible.

### SIMNET (1983-1990s)

SIMNET (SIMulator NETworking) was the groundbreaking prototype that proved the concept. Development began in the mid-1980s, led by three companies:

- **Bolt, Beranek and Newman (BBN)** -- responsible for the dynamic simulation software and the distributed networking communication layer
- **Delta Graphics, Inc.** -- visual display systems
- **Perceptronics, Inc.** -- simulator hardware

SIMNET was fielded starting in 1987 and used for training well into the 1990s. It connected tank simulators, aircraft simulators, and command post stations across the United States and Europe into a shared virtual environment.

**BBN's critical contribution was the introduction of dead reckoning to networked simulation.** Duncan "Duke" Miller, the BBN SIMNET program manager, coined the term's usage in this context to describe how simulators communicated entity state changes while minimizing network traffic. Rather than each simulator broadcasting its position every frame, SIMNET used dead reckoning to allow receiving simulators to predict entity positions, only sending corrections when predictions diverged too far from reality.

### From SIMNET to IEEE 1278

The success of SIMNET led to formalization efforts through a series of "DIS Workshops" held at the Interactive Networked Simulation for Training symposium, organized by the University of Central Florida's Institute for Simulation and Training (IST). The resulting standard was closely patterned after BBN's original SIMNET protocol.

The IEEE 1278 family of standards emerged:

| Standard | Year | Description |
|----------|------|-------------|
| IEEE 1278-1993 | 1993 | Original DIS application protocols |
| IEEE 1278.1-1995 | 1995 | Revised application protocols |
| IEEE 1278.1A-1998 | 1998 | Errata and amendments |
| IEEE 1278.2-1995 | 1995 | Communication services and profiles |
| IEEE 1278.3-1996 | 1996 | Exercise management and feedback |
| IEEE 1278.4-1997 | 1997 | Verification, validation, and accreditation |
| IEEE 1278.1-2012 | 2012 | Major update (DIS 7): 72 PDU types, improved extensibility |

NATO adopted DIS via STANAG 4482 in 1995, though this was later retired in favor of the High Level Architecture (HLA), which merged DIS concepts with the Aggregate Level Simulation Protocol (ALSP) designed by MITRE.

Despite HLA's emergence, DIS remains in active use worldwide, particularly for platform-level real-time simulation. The Simulation Interoperability Standards Organization (SISO) continues to maintain and publish updates.

---

## 2. What Dead Reckoning Is and Why It Matters

### The Fundamental Problem

In a distributed simulation with hundreds or thousands of entities, naively transmitting every entity's position every frame creates enormous network and processing overhead:

- At 30 updates/second, a single Entity State PDU (~144 bytes on the wire) generates **~4,320 bytes/second** per entity
- With 1,000 entities, that becomes **~4.3 MB/second** of raw position data
- Each UDP packet triggers a network interrupt; at high rates, systems approach the practical limit of ~50,000 packets/CPU core, leading to dropped packets
- Every received PDU must be decoded, consuming CPU that could be used for rendering or physics

### The Dead Reckoning Solution

Dead reckoning is a **replicated computing technique** where all participating nodes agree in advance on a set of algorithms to extrapolate entity behavior. Instead of receiving continuous updates, each node:

1. Receives an occasional state update (position, velocity, acceleration, orientation)
2. Uses the agreed-upon algorithm to predict the entity's position between updates
3. Continues predicting until a correction arrives

The term originates from maritime navigation ("deduced reckoning"), where sailors estimated their current position based on a previously known position, speed, heading, and elapsed time.

### Why It Matters

- **Bandwidth reduction**: Updates are only sent when predictions diverge from reality, reducing traffic by 50-90% depending on entity behavior
- **Latency hiding**: Remote nodes see smooth, continuous motion even when updates arrive infrequently
- **Scalability**: Enables simulations with thousands of entities on limited-bandwidth networks
- **Graceful degradation**: If packets are lost, the simulation continues with degraded but reasonable accuracy rather than freezing

---

## 3. The Three Models: Truth, Dead Reckoned, and Smoothing

A complete dead reckoning system maintains three conceptual models for each entity:

### 3.1 The Truth Model (Sender Side)

The **truth model** represents the actual, authoritative state of an entity as computed by its owning simulation. This is the "ground truth" -- the real position, velocity, orientation, and acceleration as determined by the local physics engine, player input, or AI.

Only the owning simulation maintains the truth model. It is never directly visible to remote nodes.

### 3.2 The Dead Reckoned Model (Both Sides)

The **dead reckoned model** is the predicted state computed using the agreed-upon algorithm and the last received state update. Both the sender and all receivers maintain this model:

- **On the sender**: The sender runs dead reckoning on its own entity using the last *transmitted* state. It compares the dead reckoned position against the truth model to decide when to send updates (threshold comparison).
- **On the receiver**: The receiver runs dead reckoning using the last *received* state to predict where the entity is right now, filling in between infrequent updates.

### 3.3 The Smoothing Model (Receiver Side)

When a correction arrives, the receiver's dead reckoned position may differ from the newly reported position. Jumping the entity instantly to the corrected position creates visual artifacts ("warping"). The **smoothing model** provides a convergence path from the current rendered position to the corrected trajectory.

The three models work together as a pipeline:

```
Sender:                          Receiver:

[Truth Model] ---compare---> [DR Model (sender)]
     |                           |
     | (exceeds threshold?)      |
     v                           |
  Send ESPDU ----network----> Receive ESPDU
                                 |
                                 v
                            [DR Model (receiver)] ---blend---> [Smoothing Model]
                                                                    |
                                                                    v
                                                              Rendered Position
```

---

## 4. The Dead Reckoning Algorithms

### Naming Convention

IEEE 1278.1 defines a three-element naming notation: **DRM(X, Y, Z)**

| Element | Values | Meaning |
|---------|--------|---------|
| **X** (Rotation) | F = Fixed | Orientation does not change (no angular velocity applied) |
| | R = Rotating | Orientation is extrapolated using angular velocity |
| **Y** (Order) | P = Position rate | First-order: constant velocity (linear extrapolation) |
| | V = Velocity rate | Second-order: velocity + acceleration (quadratic extrapolation) |
| **Z** (Coordinates) | W = World | Calculations in geocentric world coordinates (ECEF) |
| | B = Body | Calculations in entity body-axis coordinates, then transformed to world |

### The Standard Algorithms

The dead reckoning algorithm field is an 8-bit enumeration in the Entity State PDU. IEEE 1278.1 defines the following values:

#### Algorithm 0: Other
Reserved for non-standard or custom algorithms. Not used in interoperable simulations.

#### Algorithm 1: Static
The entity does not move. No extrapolation is performed. The entity remains at the last reported position and orientation indefinitely. Used for buildings, terrain features, parked vehicles, and other stationary entities.

```
P(t) = P(t0)
O(t) = O(t0)
```

#### Algorithm 2: DRM(F, P, W) -- Fixed, Position, World
**Constant velocity, no rotation, world coordinates.**

The simplest moving algorithm. The entity's position is extrapolated linearly based on velocity. Orientation remains fixed at the last reported value.

```
P(t) = P(t0) + V * dt
O(t) = O(t0)
```

Where `dt = t - t0` (elapsed time since last update).

Best for: Entities moving in a straight line at constant speed -- projectiles in initial flight, ships at cruising speed, vehicles on a straight road.

#### Algorithm 3: DRM(R, P, W) -- Rotating, Position, World
**Constant velocity with rotation, world coordinates.**

Same position extrapolation as Algorithm 2, but orientation is also extrapolated using angular velocity. This allows receivers to predict turning motion visually even though the position path remains linear.

```
P(t) = P(t0) + V * dt
O(t) = O(t0) + W * dt       (simplified; actual uses rotation matrix integration)
```

Best for: Entities where visual orientation matters and changes predictably -- aircraft in gentle banks, vehicles beginning turns.

#### Algorithm 4: DRM(R, V, W) -- Rotating, Velocity, World
**Acceleration with rotation, world coordinates.**

Position is extrapolated using both velocity and acceleration (second-order). Orientation is extrapolated using angular velocity. This captures curved trajectories.

```
P(t) = P(t0) + V * dt + 0.5 * A * dt^2
O(t) = O(t0) + W * dt       (simplified)
```

Best for: Maneuvering entities where both position accuracy and orientation matter -- aircraft in combat maneuvers, vehicles turning and accelerating.

#### Algorithm 5: DRM(F, V, W) -- Fixed, Velocity, World
**Acceleration without rotation, world coordinates.**

Position uses velocity and acceleration (second-order), but orientation remains fixed. Used for high-speed entities where orientation is less critical or doesn't change.

```
P(t) = P(t0) + V * dt + 0.5 * A * dt^2
O(t) = O(t0)
```

Best for: Missiles, high-speed projectiles, entities maneuvering but whose visual orientation is less important.

#### Algorithm 6: DRM(F, P, B) -- Fixed, Position, Body
**Constant velocity, no rotation, body coordinates.**

Same as Algorithm 2 but velocity is specified in the entity's body-axis coordinate system rather than world coordinates. The body-axis velocity is transformed to world coordinates using the entity's current orientation matrix before extrapolation.

```
V_world = R(O) * V_body
P(t) = P(t0) + V_world * dt
O(t) = O(t0)
```

Best for: When it is more natural to express velocity relative to the entity's own axes (forward, right, up).

#### Algorithm 7: DRM(R, P, B) -- Rotating, Position, Body
**Constant velocity with rotation, body coordinates.**

Position extrapolation in body coordinates with orientation extrapolation. Because the orientation changes over time (via angular velocity) and velocity is in body coordinates, the world-space velocity vector rotates as the entity turns, **enabling simulation of curved (circular) paths even without explicit acceleration**.

```
O(t) = O(t0) + W * dt
V_world(t) = R(O(t)) * V_body
P(t) = P(t0) + integrate(V_world, dt)
```

Best for: Entities in steady circular turns -- aircraft in holding patterns, vehicles on curved roads. This algorithm can produce perfect circular trajectories with zero body-axis acceleration.

#### Algorithm 8: DRM(R, V, B) -- Rotating, Velocity, Body
**Acceleration with rotation, body coordinates.**

The most complete algorithm. Position uses velocity and acceleration in body coordinates, with orientation extrapolation. The combination of body-axis acceleration and rotating reference frame allows representation of complex maneuvering.

```
O(t) = O(t0) + W * dt
A_world(t) = R(O(t)) * A_body
V_world(t) = V_world(t0) + integrate(A_world, dt)
P(t) = P(t0) + integrate(V_world, dt)
```

Best for: Highly maneuvering entities with changing acceleration vectors -- fighter aircraft in dogfights, submarines performing evasive maneuvers.

#### Algorithm 9: DRM(F, V, B) -- Fixed, Velocity, Body
**Acceleration without rotation, body coordinates.**

Acceleration-based extrapolation in body coordinates with fixed orientation. Similar to Algorithm 5 but with body-axis quantities.

```
V_world = R(O) * V_body
A_world = R(O) * A_body
P(t) = P(t0) + V_world * dt + 0.5 * A_world * dt^2
O(t) = O(t0)
```

Best for: High-speed entities where body-axis acceleration is more natural but orientation prediction is unnecessary.

### Summary Table

| # | Name | Position Order | Rotation | Coords | Typical Use |
|---|------|---------------|----------|--------|-------------|
| 0 | Other | -- | -- | -- | Non-standard |
| 1 | Static | None | None | -- | Stationary entities |
| 2 | FPW | 1st (V) | Fixed | World | Straight-line movement |
| 3 | RPW | 1st (V) | Rotating | World | Straight-line with turning |
| 4 | RVW | 2nd (V+A) | Rotating | World | Maneuvering with orientation |
| 5 | FVW | 2nd (V+A) | Fixed | World | High-speed, no rotation |
| 6 | FPB | 1st (V) | Fixed | Body | Straight-line, body coords |
| 7 | RPB | 1st (V) | Rotating | Body | Circular/curved motion |
| 8 | RVB | 2nd (V+A) | Rotating | Body | Complex maneuvering |
| 9 | FVB | 2nd (V+A) | Fixed | Body | High-speed, body coords |

> **Note:** Earlier versions of IEEE 1278.1 reserved values 10 and 11 for alternate algorithms. These were removed in later revisions.

### Choosing an Algorithm

The choice depends on the entity's behavior and the simulation's fidelity requirements:

- **Static entities** (buildings, terrain): Algorithm 1
- **Constant velocity, no turns**: Algorithm 2 (FPW) -- simplest and cheapest
- **Entities that turn**: Algorithm 3 (RPW) or 7 (RPB) -- RPB enables curved paths
- **Accelerating entities**: Algorithm 4 (RVW) or 5 (FVW)
- **Complex maneuvering**: Algorithm 4 (RVW) or 8 (RVB) -- highest fidelity

---

## 5. Threshold-Based Updates

### The Core Mechanism

The owning simulation does not send updates on a fixed schedule. Instead, it runs the dead reckoning algorithm locally on its own entity, using the **last transmitted state** as the starting point. It continuously compares the dead reckoned position against the truth model.

When the difference exceeds a predefined threshold, the simulation sends a new Entity State PDU with the current truth state.

### Position Threshold

The most common threshold is a **distance threshold** -- typically measured as the Euclidean distance between the dead reckoned position and the true position:

```
error = ||P_truth - P_deadreckoned||
if error > threshold_position:
    send_update()
```

Typical values range from 1 meter (high fidelity, e.g., close combat) to 100 meters (low fidelity, e.g., distant radar tracks).

### Orientation Threshold

Similarly, orientation error can trigger updates:

```
orientation_error = angular_difference(O_truth, O_deadreckoned)
if orientation_error > threshold_orientation:
    send_update()
```

### Adaptive Thresholds

Research has explored adaptive thresholds that change based on:

- **Proximity to other entities**: Tighter thresholds when entities are close together (where errors are more noticeable)
- **Entity importance**: Lower thresholds for the player's own vehicle or entities in combat
- **Network congestion**: Larger thresholds when bandwidth is constrained
- **Viewer distance**: Screen-space error metrics that account for how large the position error appears to the observer

### Trade-offs

| Smaller Threshold | Larger Threshold |
|-------------------|-----------------|
| More accurate remote representation | Less accurate remote representation |
| More network traffic | Less network traffic |
| Smoother perceived motion | More correction jumps |
| Higher CPU cost (more PDU encoding/decoding) | Lower CPU cost |

---

## 6. Heartbeat Updates

### Purpose

Even if an entity's dead reckoned position never exceeds the threshold (e.g., a ship traveling in a perfectly straight line at constant speed), periodic updates are still necessary to:

1. **Confirm entity existence**: Without updates, remote simulations cannot distinguish between "entity is still there, moving predictably" and "entity has been destroyed / simulation has crashed / network has failed"
2. **Correct accumulated floating-point drift**: Small numerical errors accumulate over time
3. **Handle missed packets**: If a threshold-triggered update is lost, the heartbeat ensures eventual correction

### IEEE 1278.1 Requirements

The standard specifies default heartbeat intervals:

- **Entity State PDU**: 5 seconds (default). If no threshold-triggered update has been sent within 5 seconds, a heartbeat ESPDU must be transmitted.
- **Timeout**: If a receiving simulation has not received any ESPDU (threshold or heartbeat) for an entity within a timeout period (typically 12 seconds, or 2.4x the heartbeat interval), the entity is considered to have departed the simulation and should be removed.

### Interaction with Threshold Updates

The heartbeat timer resets whenever any update is sent (threshold-triggered or otherwise). The actual update rate for an entity is therefore:

```
effective_rate = max(threshold_triggered_rate, heartbeat_rate)
```

For a stationary entity: updates every 5 seconds (heartbeat only).
For a wildly maneuvering fighter jet: updates several times per second (threshold-triggered), no heartbeats needed.

---

## 7. Entity State PDU Structure

The Entity State PDU (ESPDU) is the most commonly used PDU in DIS. It conveys all the information a receiving simulation needs to represent a remote entity.

### PDU Layout

```
+--------------------------------------------------+
|                   PDU Header (12 bytes)           |
+--------------------------------------------------+
| Protocol Version      | 1 byte  (UnsignedByte)   |
| Exercise ID           | 1 byte  (UnsignedByte)   |
| PDU Type (= 1)        | 1 byte  (UnsignedByte)   |
| Protocol Family (= 1) | 1 byte  (UnsignedByte)   |
| Timestamp             | 4 bytes (UnsignedInt)    |
| PDU Length             | 2 bytes (UnsignedShort)  |
| Padding               | 2 bytes                  |
+--------------------------------------------------+
|                   PDU Body (132 bytes)            |
+--------------------------------------------------+
| Entity ID             | 6 bytes                  |
|   Site ID             |   2 bytes (UnsignedShort)|
|   Application ID      |   2 bytes (UnsignedShort)|
|   Entity Number       |   2 bytes (UnsignedShort)|
| Force ID              | 1 byte  (UnsignedByte)   |
| # Articulation Params | 1 byte  (UnsignedByte)   |
| Entity Type           | 8 bytes                  |
|   Kind                |   1 byte                 |
|   Domain              |   1 byte                 |
|   Country             |   2 bytes                |
|   Category            |   1 byte                 |
|   Subcategory         |   1 byte                 |
|   Specific            |   1 byte                 |
|   Extra               |   1 byte                 |
| Alternative Entity Type | 8 bytes                |
| Entity Linear Velocity| 12 bytes (3x Float32)    |
|   X velocity          |   4 bytes                |
|   Y velocity          |   4 bytes                |
|   Z velocity          |   4 bytes                |
| Entity Location       | 24 bytes (3x Float64)    |
|   X (geocentric)      |   8 bytes                |
|   Y (geocentric)      |   8 bytes                |
|   Z (geocentric)      |   8 bytes                |
| Entity Orientation    | 12 bytes (3x Float32)    |
|   Psi (heading)       |   4 bytes                |
|   Theta (pitch)       |   4 bytes                |
|   Phi (roll)          |   4 bytes                |
| Entity Appearance     | 4 bytes  (UnsignedInt)   |
| Dead Reckoning Params | 40 bytes                 |
|   DR Algorithm        |   1 byte  (enum)         |
|   Other Parameters    |   15 bytes               |
|   Linear Acceleration |   12 bytes (3x Float32)  |
|   Angular Velocity    |   12 bytes (3x Float32)  |
| Entity Marking        | 12 bytes                 |
|   Character Set       |   1 byte                 |
|   Characters          |   11 bytes (ASCII)       |
| Entity Capabilities   | 4 bytes  (UnsignedInt)   |
+--------------------------------------------------+
| Articulation Parameters| Variable length          |
|   (each record: 16 bytes)                        |
+--------------------------------------------------+
```

**Minimum PDU size**: 144 bytes (12 header + 132 body, no articulation parameters)

### Key Fields for Dead Reckoning

The fields that directly support dead reckoning are:

- **Entity Linear Velocity** (12 bytes): Current velocity vector in m/s. Used by all algorithms except Static.
- **Entity Location** (24 bytes): Current position in geocentric (ECEF) coordinates using 64-bit doubles for precision.
- **Entity Orientation** (12 bytes): Euler angles (psi, theta, phi) in radians.
- **Dead Reckoning Parameters** (40 bytes):
  - **Algorithm** (1 byte): Which of the 9 algorithms the receiver should use.
  - **Other Parameters** (15 bytes): Algorithm-specific parameters (reserved for future use in most algorithms).
  - **Linear Acceleration** (12 bytes): Acceleration vector for second-order algorithms.
  - **Angular Velocity** (12 bytes): Rotational velocity for rotating algorithms.

### Coordinate System

DIS uses a geocentric Cartesian coordinate system (Earth-Centered, Earth-Fixed / ECEF):
- **X-axis**: Points from Earth's center through the intersection of the Prime Meridian and the Equator
- **Y-axis**: Points from Earth's center through 90 degrees East longitude on the Equator
- **Z-axis**: Points from Earth's center through the North Pole

Positions use 64-bit doubles to maintain centimeter-level precision anywhere on Earth.

---

## 8. Smoothing Techniques for Corrections

When a new Entity State PDU arrives, the receiver's dead reckoned position typically does not match the newly reported position. The entity must be moved to the correct trajectory, but an instantaneous jump ("warp") is visually jarring. Smoothing algorithms provide a gradual transition.

### The Correction Problem

At time `t_update`:
- **Current rendered position**: Where the entity is being displayed (from previous dead reckoning)
- **Corrected position**: Where the new PDU says the entity is
- **Corrected trajectory**: The new dead reckoned path starting from the corrected state

The smoothing algorithm must transition from the current rendered position to a point on the corrected trajectory over a convergence period.

### Convergence Period

The **convergence period** (also called the smoothing interval) is the time over which the correction is applied. Typical values are 1-3 seconds or a fixed number of frames. Too short causes visible snapping; too long causes the entity to appear sluggish.

### Linear Smoothing (Straight-Line Convergence)

The simplest approach: linearly interpolate between the current rendered position and the target position on the corrected dead reckoned trajectory.

```
t_blend = (t - t_update) / convergence_period    // 0.0 to 1.0

P_target(t) = dead_reckon(corrected_state, t - t_update)
P_smooth(t) = lerp(P_rendered_at_update, P_target(t), t_blend)
```

**Pros**: Simple to implement, computationally cheap.
**Cons**: Can produce noticeable direction changes at the start and end of the convergence period. The blended path may not be physically plausible.

### Cubic Spline Smoothing (Hermite Convergence)

Uses a cubic Hermite spline to create a smooth curve from the current position/velocity to the corrected position/velocity. This ensures continuity of both position and velocity (C1 continuity), eliminating abrupt direction changes.

The Hermite curve is defined by two endpoints and two tangent vectors:

```
P0 = current rendered position
V0 = current rendered velocity
P1 = target position on corrected DR trajectory at end of convergence period
V1 = target velocity at end of convergence period

t_blend = (t - t_update) / convergence_period

h00 = 2t^3 - 3t^2 + 1
h10 = t^3 - 2t^2 + t
h01 = -2t^3 + 3t^2
h11 = t^3 - t^2

P_smooth = h00 * P0 + h10 * (convergence_period * V0) + h01 * P1 + h11 * (convergence_period * V1)
```

**Pros**: Smooth transitions with no sudden velocity changes. Physically more plausible paths.
**Cons**: More computationally expensive. Can overshoot in extreme cases.

### Catmull-Rom Spline Smoothing

A variant of cubic smoothing using Catmull-Rom splines, which automatically derive tangent vectors from surrounding control points. Useful when multiple correction points are available.

### Blended Dead Reckoning

Rather than smoothing position directly, some implementations blend the *inputs* to the dead reckoning algorithm:

```
V_blended = lerp(V_old, V_new, t_blend)
A_blended = lerp(A_old, A_new, t_blend)
// Run dead reckoning with blended inputs
```

This can produce more natural-looking motion since the extrapolation itself generates the smooth path.

### Recommendations

Research comparing smoothing methods in DIS simulations has found:

- **Linear smoothing** is adequate for most applications and is recommended as the default
- **Cubic spline smoothing** is preferred when visual quality is critical (e.g., cockpit-view flight simulators)
- **More smoothing steps** (longer convergence periods) improve visual smoothness but increase the time the entity spends at an incorrect position
- The convergence period should be shorter than the expected update interval to avoid overlapping corrections

---

## 9. How Dead Reckoning Applies to Games

### Vehicle Simulations and Racing Games

Vehicle simulations are the most natural fit for DIS-style dead reckoning because vehicles have **high inertia** -- they cannot change direction or speed instantly. A car traveling at 100 km/h will be very close to where velocity-based extrapolation predicts it will be in 100ms. Even during turns, body-axis algorithms (DRM 7, RPB) can accurately predict curved paths.

**Typical approach**:
- Use DRM(R, P, B) or DRM(R, V, W) depending on maneuvering intensity
- Position threshold: 0.5-2.0 meters
- Orientation threshold: 2-5 degrees
- Convergence period: 200-500ms

### Flight Simulators

Flight simulators were the original use case (SIMNET included helicopter and fixed-wing simulators). Aircraft have even higher inertia than ground vehicles, making dead reckoning highly effective.

**Specific considerations**:
- DRM(R, V, W) or DRM(R, V, B) for maneuvering aircraft
- DRM(F, V, W) for missiles and munitions
- Longer convergence periods are acceptable because aircraft motion is smoother
- Angular velocity prediction is critical for visual orientation (banking, pitching)

### MMOs and RPGs

Dead reckoning is **less directly applicable** to character-based games where players can start, stop, and change direction nearly instantaneously. A character controlled by keyboard input (WASD) has very low inertia and high unpredictability.

However, adapted forms are widely used:

- **Input prediction**: Instead of extrapolating physics, predict that the player continues their current movement input (if pressing "forward", assume they keep pressing "forward")
- **Path-based prediction**: If the entity is following a known path (e.g., an NPC on a patrol route, a player on a road), predict along the path
- **Hybrid approaches**: Use dead reckoning for vehicles and physics objects in the MMO, but input-based prediction for player characters

### Real-Time Strategy Games

RTS games often have large numbers of entities (hundreds or thousands of units) moving in formation or along computed paths. Dead reckoning can reduce update rates for units in predictable motion (marching, patrolling) while sending frequent updates during combat.

### The Predictability Spectrum

The effectiveness of dead reckoning correlates directly with entity **inertia** and **predictability**:

```
More Predictable (DR works well)          Less Predictable (DR struggles)
<----------------------------------------------------------------------->
Ships    Aircraft    Vehicles    NPCs    FPS Characters    Teleporting
Trains   Missiles    Balls       Mobs    Fighting Games    Abilities
```

---

## 10. Implementation Pseudo-Code

### 10.1 Basic Dead Reckoning Extrapolation

```python
class DeadReckonedEntity:
    def __init__(self):
        # Last known state (from most recent ESPDU)
        self.position = Vector3(0, 0, 0)      # meters, world coords
        self.velocity = Vector3(0, 0, 0)      # m/s
        self.acceleration = Vector3(0, 0, 0)  # m/s^2
        self.orientation = Euler(0, 0, 0)     # radians (psi, theta, phi)
        self.angular_velocity = Vector3(0, 0, 0)  # rad/s
        self.timestamp = 0.0                  # seconds
        self.algorithm = 2                    # DRM(F, P, W) default

    def extrapolate(self, current_time):
        """Compute dead reckoned position at current_time."""
        dt = current_time - self.timestamp

        if self.algorithm == 1:
            # Static: no movement
            return self.position, self.orientation

        elif self.algorithm == 2:
            # DRM(F, P, W): constant velocity, no rotation
            pos = self.position + self.velocity * dt
            ori = self.orientation
            return pos, ori

        elif self.algorithm == 3:
            # DRM(R, P, W): constant velocity + rotation
            pos = self.position + self.velocity * dt
            ori = self.orientation + self.angular_velocity * dt
            return pos, ori

        elif self.algorithm == 4:
            # DRM(R, V, W): velocity + acceleration + rotation
            pos = (self.position
                   + self.velocity * dt
                   + 0.5 * self.acceleration * dt * dt)
            ori = self.orientation + self.angular_velocity * dt
            return pos, ori

        elif self.algorithm == 5:
            # DRM(F, V, W): velocity + acceleration, no rotation
            pos = (self.position
                   + self.velocity * dt
                   + 0.5 * self.acceleration * dt * dt)
            ori = self.orientation
            return pos, ori

        elif self.algorithm in (6, 7, 8, 9):
            # Body-axis algorithms: transform to world first
            return self._extrapolate_body_axis(dt)

    def _extrapolate_body_axis(self, dt):
        """Body-axis algorithms (6-9): velocity/acceleration are in body coords."""
        # For rotating algorithms, update orientation first
        if self.algorithm in (7, 8):
            ori = self.orientation + self.angular_velocity * dt
        else:
            ori = self.orientation

        # Build rotation matrix from orientation
        R = rotation_matrix_from_euler(ori)

        # Transform body-axis quantities to world
        vel_world = R * self.velocity
        acc_world = R * self.acceleration

        if self.algorithm in (6, 7):
            # First-order: velocity only
            pos = self.position + vel_world * dt
        else:
            # Second-order: velocity + acceleration
            pos = (self.position
                   + vel_world * dt
                   + 0.5 * acc_world * dt * dt)

        return pos, ori
```

### 10.2 Threshold Comparison (Sender Side)

```python
class EntityOwner:
    def __init__(self):
        self.truth = EntityState()          # actual state from physics/input
        self.last_transmitted = EntityState()  # state in most recent sent ESPDU
        self.dr_model = DeadReckonedEntity()   # DR model using last_transmitted
        self.position_threshold = 1.0       # meters
        self.orientation_threshold = 0.05   # radians (~3 degrees)
        self.heartbeat_interval = 5.0       # seconds
        self.last_transmit_time = 0.0

    def update(self, current_time):
        """Called every simulation frame."""

        # Step 1: Update truth model from physics/input
        self.truth.update_from_physics()

        # Step 2: Compute what remote nodes think our position is
        dr_position, dr_orientation = self.dr_model.extrapolate(current_time)

        # Step 3: Check position threshold
        position_error = distance(self.truth.position, dr_position)
        position_exceeded = position_error > self.position_threshold

        # Step 4: Check orientation threshold
        orientation_error = angular_distance(self.truth.orientation, dr_orientation)
        orientation_exceeded = orientation_error > self.orientation_threshold

        # Step 5: Check heartbeat
        time_since_last = current_time - self.last_transmit_time
        heartbeat_due = time_since_last >= self.heartbeat_interval

        # Step 6: Send update if any condition is met
        if position_exceeded or orientation_exceeded or heartbeat_due:
            self.send_entity_state_pdu(current_time)

    def send_entity_state_pdu(self, current_time):
        """Construct and transmit an ESPDU with current truth state."""
        pdu = EntityStatePDU()
        pdu.entity_id = self.entity_id
        pdu.position = self.truth.position
        pdu.velocity = self.truth.velocity
        pdu.acceleration = self.truth.acceleration
        pdu.orientation = self.truth.orientation
        pdu.angular_velocity = self.truth.angular_velocity
        pdu.dead_reckoning_algorithm = self.dr_model.algorithm
        pdu.timestamp = current_time

        network.send(pdu)

        # Reset DR model to current truth state
        self.dr_model.position = self.truth.position
        self.dr_model.velocity = self.truth.velocity
        self.dr_model.acceleration = self.truth.acceleration
        self.dr_model.orientation = self.truth.orientation
        self.dr_model.angular_velocity = self.truth.angular_velocity
        self.dr_model.timestamp = current_time

        self.last_transmit_time = current_time
```

### 10.3 Smoothing on Correction (Receiver Side)

```python
class SmoothedEntity:
    def __init__(self):
        self.dr_model = DeadReckonedEntity()
        self.is_converging = False
        self.convergence_duration = 1.0     # seconds
        self.convergence_start_time = 0.0

        # Smoothing state
        self.smooth_start_pos = Vector3(0, 0, 0)
        self.smooth_start_vel = Vector3(0, 0, 0)
        self.smooth_start_ori = Euler(0, 0, 0)

    def receive_update(self, pdu, current_time):
        """Called when a new ESPDU arrives from the network."""

        # Save current rendered state as smoothing start point
        if self.is_converging:
            # Already converging -- use current blended position
            self.smooth_start_pos, self.smooth_start_ori = self.get_rendered_state(current_time)
        else:
            # Use current DR position
            self.smooth_start_pos, self.smooth_start_ori = self.dr_model.extrapolate(current_time)

        self.smooth_start_vel = self.dr_model.velocity  # velocity at moment of correction

        # Update DR model with new state from PDU
        self.dr_model.position = pdu.position
        self.dr_model.velocity = pdu.velocity
        self.dr_model.acceleration = pdu.acceleration
        self.dr_model.orientation = pdu.orientation
        self.dr_model.angular_velocity = pdu.angular_velocity
        self.dr_model.algorithm = pdu.dead_reckoning_algorithm
        self.dr_model.timestamp = pdu.timestamp

        # Begin convergence
        self.is_converging = True
        self.convergence_start_time = current_time

    def get_rendered_state(self, current_time):
        """Get the position/orientation to actually render."""

        # Always compute the DR target (where entity should be on corrected trajectory)
        dr_pos, dr_ori = self.dr_model.extrapolate(current_time)

        if not self.is_converging:
            return dr_pos, dr_ori

        # Compute blend factor
        elapsed = current_time - self.convergence_start_time
        t = clamp(elapsed / self.convergence_duration, 0.0, 1.0)

        if t >= 1.0:
            # Convergence complete
            self.is_converging = False
            return dr_pos, dr_ori

        # --- Linear smoothing ---
        # pos = lerp(smooth_start_pos, dr_pos, t)

        # --- Cubic Hermite smoothing (preferred) ---
        # Hermite basis functions
        h00 = 2*t*t*t - 3*t*t + 1
        h10 = t*t*t - 2*t*t + t
        h01 = -2*t*t*t + 3*t*t
        h11 = t*t*t - t*t

        T = self.convergence_duration
        pos = (h00 * self.smooth_start_pos
               + h10 * T * self.smooth_start_vel
               + h01 * dr_pos
               + h11 * T * self.dr_model.velocity)

        # Spherical interpolation for orientation
        ori = slerp(self.smooth_start_ori, dr_ori, t)

        return pos, ori
```

---

## 11. Comparison to Game-Specific Prediction Approaches

| Aspect | DIS Dead Reckoning | Client-Side Prediction (FPS) | Entity Interpolation (Valve) | Rollback Netcode (Fighting Games) |
|--------|-------------------|------------------------------|-----------------------------|------------------------------------|
| **Architecture** | Peer-to-peer / multicast | Client-server | Client-server | Peer-to-peer |
| **Authority** | Each node owns its entities | Server authoritative | Server authoritative | Each peer authoritative over its inputs |
| **Prediction basis** | Physics (velocity, acceleration) | Player input replay | None (renders the past) | Full game state simulation |
| **Update trigger** | Threshold exceeded | Every tick (server) | Every tick (server) | Every tick (both peers) |
| **Correction method** | Smoothed convergence | Replay from corrected state | Display delay (interpolation buffer) | Rollback, re-simulate, fast-forward |
| **Latency handling** | Extrapolation forward | Extrapolation forward + input buffer | Interpolation backward (adds display delay) | Rollback to past state |
| **Best for** | High-inertia entities (vehicles) | Player-controlled characters | All entity types | Precise frame-by-frame gameplay |
| **Bandwidth** | Very low (threshold-based) | Moderate (input + state) | Moderate (state snapshots) | Low (input only) |
| **Entity count** | Thousands | Tens to low hundreds | Tens to low hundreds | 2-8 players |

### Key Differences

**DIS vs. Client-Side Prediction (Quake/Source Engine)**:
DIS dead reckoning is primarily *physics-based extrapolation* -- it assumes entities obey Newtonian mechanics. Client-side prediction in FPS games replays *player inputs* against game logic, which handles discontinuous movement (jumping, stopping) far better. However, client-side prediction requires a deterministic simulation and server authority, while DIS works in a peer-to-peer model with no central server.

**DIS vs. Entity Interpolation (Valve/Source Engine)**:
Entity interpolation deliberately renders entities in the *past* (typically 100ms behind real-time) so it always has two known states to interpolate between. This guarantees smooth motion but adds display latency. DIS dead reckoning extrapolates *forward* into the future, avoiding added display latency but accepting the risk of incorrect predictions. Many modern games combine both: interpolation when data is available, extrapolation when it runs out.

**DIS vs. Rollback Netcode (GGPO)**:
Rollback netcode is designed for games where a single frame of error is unacceptable (fighting games). It re-simulates past frames when corrections arrive, which requires the full game state to be deterministically reproducible. DIS dead reckoning is designed for continuous physical motion where small positional errors are acceptable and smoothed out. The approaches solve fundamentally different problems.

### What Games Have Borrowed from DIS

Many game networking concepts trace back to DIS:

- **Threshold-based updates** (only send when state diverges) are used in virtually every networked game
- **Interest management** (only process entities in range) evolved from DIS's multicast group management
- **Heartbeat messages** for entity timeout/removal
- **Extrapolation** as a general technique for hiding latency
- **The concept of agreed-upon prediction algorithms** between sender and receiver

---

## 12. Modern Relevance and Applications

### Military and Defense

DIS remains the standard for platform-level military simulation:

- **Live, Virtual, and Constructive (LVC) training**: Connecting real vehicles, human-in-the-loop simulators, and computer-generated forces in a shared environment
- **Joint and coalition exercises**: NATO and allied nations use DIS for interoperable multi-national exercises
- **After-action review**: DIS data streams (PDU logs) provide complete records of exercise events

The 2012 update (DIS 7) added 72 PDU types, support for directed energy weapons, information operations, and improved extensibility, ensuring continued relevance.

### Space and Aerospace

NASA and aerospace organizations use DIS for:
- Launch simulation and mission rehearsal
- Satellite constellation simulation
- Space situational awareness

### Autonomous Vehicles and Robotics

Dead reckoning principles from DIS apply directly to:
- **Autonomous vehicle simulation**: Testing self-driving cars in virtual environments with thousands of simulated traffic participants
- **Robot swarm coordination**: Distributed prediction of peer robot positions to reduce communication requirements
- **Digital twins**: Maintaining synchronized representations of physical assets

### Game Engines and Middleware

Several modern game engines and networking libraries incorporate DIS-derived concepts:

- **Open-DIS**: Free, open-source DIS implementation in Java, C++, Python, JavaScript, C#, and Objective-C, maintained by the MOVES Institute at the Naval Postgraduate School
- **VR-Forces / VR-Link (MAK Technologies)**: Commercial DIS middleware used in both defense and entertainment
- **Unreal Engine / Unity plugins**: DIS integration plugins for commercial game engines used in training applications

### Lessons for Modern Game Networking

The DIS standard, despite being designed for military simulation in the 1980s-1990s, established principles that remain fundamental:

1. **Agree on prediction algorithms**: Sender and receiver must use the same extrapolation method
2. **Only send what's needed**: Threshold-based updates are more efficient than fixed-rate updates
3. **Smooth corrections**: Never teleport entities; always converge gracefully
4. **Plan for failure**: Heartbeat timeouts handle network failures and simulation crashes
5. **Match the algorithm to the entity**: Different entity types need different prediction models
6. **Body-axis vs. world-axis**: The right coordinate system simplifies prediction for specific motion types

---

## 13. References

### Standards and Specifications

1. IEEE 1278.1-2012 -- *IEEE Standard for Distributed Interactive Simulation (DIS) -- Application Protocols*
   - https://standards.ieee.org/ieee/1278.1/4949/

2. SISO -- Simulation Interoperability Standards Organization
   - https://www.sisostds.org/

### Open Source Implementations

3. Open-DIS Project -- Free, open-source DIS protocol implementation
   - http://open-dis.org/
   - https://github.com/open-dis

4. Open-DIS Tutorial -- Dead Reckoning
   - https://github.com/open-dis/dis-tutorial/wiki/Dead-Reckoning

5. Open-DIS Tutorial -- Entity State PDUs
   - https://open-dis.github.io/dis-tutorial/EntityStatePDUs.html

6. Open-DIS Tutorial -- Dead Reckoning State Updates
   - https://github.com/open-dis/dis-tutorial/blob/master/DeadReckoningStateUpdate.md

7. Dead Reckoning Algorithm Enumerations (Open-DIS 7 Javadoc)
   - https://savage.nps.edu/open-dis7-java/javadoc/edu/nps/moves/dis7/enumerations/DeadReckoningAlgorithm.html

8. DIS Data Dictionary -- Dead Reckoning Algorithm Field
   - https://faculty.nps.edu/brutzman/vrtp/mil/navy/nps/disEnumerations/JdbeHtmlFiles/pdu/38.htm

### Historical and Technical References

9. Wikipedia -- *Distributed Interactive Simulation*
   - https://en.wikipedia.org/wiki/Distributed_Interactive_Simulation

10. Wikipedia -- *SIMNET*
    - https://en.wikipedia.org/wiki/SIMNET

11. Pantel, L. and Wolf, L. -- "On the Suitability of Dead Reckoning Schemes for Games" (2002)
    - https://dl.acm.org/doi/10.1145/566500.566511

12. Ryan, M. -- "Modelling of Dead Reckoning and Heartbeat Update Mechanisms in Distributed Interactive Simulation" (PDF)
    - https://open-dis.github.io/dis-tutorial/documents/DeadReckoning_Ryan.pdf

13. Lee, B.S. et al. -- "Adaptive Dead Reckoning Algorithms for Distributed Interactive Simulation"
    - https://ijssst.info/Vol-01/No-1+2/paper3.pdf

### Game Networking

14. Aronson, J. -- "Dead Reckoning: Latency Hiding for Networked Games" (Gamasutra/Game Developer)
    - https://www.gamedeveloper.com/programming/dead-reckoning-latency-hiding-for-networked-games

15. Gambetta, G. -- "Fast-Paced Multiplayer" (Entity Interpolation and Client-Side Prediction)
    - https://www.gabrielgambetta.com/entity-interpolation.html

16. Bernier, Y. -- "Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization" (Valve)
    - https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization

17. Smith, S.L. et al. -- "Predictive Dead Reckoning for Online Peer-to-Peer Games" (2023)
    - https://ece.uwaterloo.ca/~sl2smith/papers/2023TOG-Predictive_Dead_Reckoning.pdf
