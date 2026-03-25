# Rocket League: Physics Networking & Prediction

## Overview

**Speaker:** Jared Cone, Lead Gameplay Engineer at Psyonix
**Talk:** "It IS Rocket Science! The Physics of Rocket League Detailed"
**Event:** GDC 2018 (Wednesday, March 21, 2018)

Jared Cone's GDC 2018 presentation is one of the most detailed public accounts of networking a real-time physics-driven competitive game. Rocket League is a uniquely demanding networked game: cars, the ball, and the arena are all fully physically simulated at 120 Hz, collisions are core gameplay (not edge cases), and competitive integrity demands that what players see closely matches the authoritative server state -- even at 100+ ms round-trip latency.

The predecessor game, **Supersonic Acrobatic Rocket-Powered Battle Cars** (SARPBC, 2008), used **snapshot interpolation** for its netcode. While functional, the fundamental tension of predicted local cars interacting with an interpolated ball created unsatisfying gameplay. For Rocket League, Psyonix redesigned the networking architecture around **client-side prediction with rollback and resimulation**, resulting in a dramatically improved player experience.

---

## 1. Why Bullet Physics Instead of PhysX

Rocket League is built on **Unreal Engine 3**, which ships with NVIDIA PhysX as its integrated physics engine. Despite this, Psyonix chose to replace PhysX with the open-source **Bullet Physics** engine. The key reasons were:

### Networking Requirements

- **Save/Load State:** Rollback netcode requires the ability to snapshot the entire physics world state and restore it to a previous frame. PhysX (as integrated in UE3) did not expose the internal state serialization needed for this. Bullet's open-source codebase allowed Psyonix to implement complete state save/restore.
- **Deterministic Replay:** While Bullet is not perfectly deterministic across platforms (floating-point differences still occur), its open source nature allowed Psyonix to control and understand exactly what was happening inside the physics engine. With PhysX as a closed black box, debugging desynchronization issues would have been far harder.
- **Full Control:** Being able to modify, extend, and deeply integrate the physics engine with their networking layer was essential. Bullet gave them the source-level control that PhysX's UE3 integration did not.

### Vehicle Simulation

Psyonix based their car simulation on Bullet's **`btRaycastVehicle`** class, which uses downward-facing raycasts from each wheel position to detect ground contact and apply suspension forces, rather than simulating physical wheel geometry. This approach is computationally efficient and well-suited for the arcade-style vehicle handling Rocket League requires.

### Scale Factor

The simulation space in Bullet Physics is **50x smaller** than in Unreal Engine. All positions, velocities, and forces are scaled down by 50x when passed from Unreal to Bullet, and scaled back up when reading results. This keeps the physics values in a range where Bullet's solver is most numerically stable.

### Performance

Bullet is slightly slower than PhysX in raw throughput, but Rocket League's physics world is relatively simple -- a handful of cars, one ball, and an arena with simple collision geometry. At 120 Hz tick rate, the per-frame cost is small enough that Bullet's performance is more than adequate, and the extra overhead is far outweighed by the networking benefits.

---

## 2. Architecture Overview

Rocket League uses a **server-authoritative** architecture with **client-side prediction** and **rollback resimulation**:

```
┌─────────────────────────────────────────────────────┐
│                   DEDICATED SERVER                   │
│  - Authoritative physics simulation at 120 Hz       │
│  - Processes client inputs from input buffer         │
│  - Sends state snapshots to clients at ~60 Hz        │
│  - Never resimulates / never rolls back              │
└─────────────────────────────────────────────────────┘
         ▲ inputs (60 pkt/s)    │ state snapshots (60 pkt/s)
         │                      ▼
┌─────────────────────────────────────────────────────┐
│                      CLIENT                          │
│  - Runs full physics simulation at 120 Hz            │
│  - Predicts local car, remote cars, AND ball         │
│  - Runs "ahead" of server by ~½ RTT + buffer        │
│  - On correction: rollback → restore → resimulate   │
│  - Visual smoothing decoupled from physics state     │
└─────────────────────────────────────────────────────┘
```

### Key Properties

- **Server tick rate:** 120 Hz physics simulation
- **Server send rate:** ~60 packets/second (every other physics frame)
- **Client send rate:** ~60 packets/second (bundling 2 input frames per packet)
- **Client input history:** Up to 10 unacknowledged input frames sent per packet
- **Clients send only inputs** (steering, throttle, boost, jump) -- never game state
- **Server sends authoritative state** (positions, velocities, rotations, angular velocities, boost amounts)

---

## 3. Client-Side Prediction for the Local Car

The local player's car is **fully predicted** on the client. The client applies inputs immediately to its local physics simulation without waiting for server confirmation, providing zero-latency input response.

### How it works

1. The client runs the same physics simulation code as the server.
2. When the player presses a button, the input is applied immediately to the local simulation.
3. The input is also sent to the server (with a frame number).
4. The client stores a history of its inputs and resulting physics states.
5. The client runs its simulation **ahead** of the server by approximately `½ RTT + input_buffer_time`, so that by the time the server processes a frame, the client's inputs for that frame have already arrived.

```
Timeline (50ms ping, 4-frame input buffer at 120Hz ≈ 33ms):

Client frame 100 ──input──► Server receives at ~frame 97
                              (input arrives ~3 frames early,
                               sits in buffer until server
                               reaches frame 100)
```

### Pseudocode: Local Car Prediction

```python
class ClientPrediction:
    def __init__(self):
        self.input_history = RingBuffer(capacity=128)  # stores (frame, input, state)
        self.current_frame = 0

    def tick(self, local_input, dt):
        # 1. Record input for this frame
        self.input_history.push(self.current_frame, local_input, self.get_physics_state())

        # 2. Apply input to local physics immediately
        self.apply_input(local_input)
        self.step_physics(dt)  # 1/120 second

        # 3. Send input to server
        self.send_input_to_server(self.current_frame, local_input)

        self.current_frame += 1
```

Because the client and server run the same simulation code with the same inputs, the local car's prediction is usually very accurate. Corrections are only needed when:
- The local car collides with another car (whose inputs were predicted incorrectly)
- The local car hits the ball (whose trajectory was predicted incorrectly)
- Network jitter causes input timing mismatches

---

## 4. Server Reconciliation and Physics Rollback

When the client receives an authoritative state snapshot from the server, it must reconcile its predicted state with the server's truth.

### The Reconciliation Process

1. **Receive server snapshot** for frame `N` (which is in the client's past, due to latency).
2. **Compare** server state against the client's stored state for frame `N`.
3. If the difference exceeds a threshold, **roll back**:
   a. **Save** the current physics world state.
   b. **Restore** the physics world to the server's authoritative state at frame `N`.
   c. **Resimulate** all frames from `N` to the client's current frame, replaying stored inputs.
   d. The result is the new "corrected present."
4. Calculate the **visual offset** (difference between old predicted present and new corrected present) for smoothing.

### Pseudocode: Server Reconciliation

```python
def on_server_state_received(self, server_frame, server_state):
    # Find our stored state for the server's frame
    stored = self.input_history.get(server_frame)
    if stored is None:
        return  # frame too old, discard

    # Compare predicted vs authoritative state
    error = compute_state_error(stored.state, server_state)

    if error < CORRECTION_THRESHOLD:
        return  # prediction was close enough

    # --- ROLLBACK AND RESIMULATE ---

    # 1. Record current visual position (for smoothing later)
    old_visual_position = self.car.visual_position
    old_visual_rotation = self.car.visual_rotation

    # 2. Restore physics world to server's authoritative state
    self.restore_physics_state(server_frame, server_state)

    # 3. Resimulate from server_frame to current_frame
    for frame in range(server_frame, self.current_frame):
        input_for_frame = self.input_history.get(frame).input
        self.apply_input(input_for_frame)
        self.step_physics(DT)

    # 4. Compute visual offset for smooth correction
    new_visual_position = self.car.physics_position
    new_visual_rotation = self.car.physics_rotation

    self.position_offset = old_visual_position - new_visual_position
    self.rotation_offset = old_visual_rotation * inverse(new_visual_rotation)
```

### Important: The Server Never Resimulates

The server is always authoritative and always moves forward. If a client's inputs arrive late (input buffer runs empty), the server **repeats the last known inputs** rather than waiting or resimulating. This means:
- The server simulation never pauses or rolls back.
- Late inputs are simply queued normally when they arrive -- there is no retroactive correction on the server side.
- This approach is simple and robust, though it can cause minor desyncs on unstable connections.

---

## 5. Ball and Remote Car Prediction

Unlike many networked games that only predict the local player, Rocket League **predicts everything** -- remote cars and the ball are all part of the client's forward-simulated physics world. This was a deliberate departure from the SARPBC approach (snapshot interpolation for remote objects).

### Why Predict Everything?

In SARPBC, local cars were predicted (running in "present" time) while the ball was interpolated (running in the "past"). This created a fundamental problem: a predicted car would contact the interpolated ball, but the ball wouldn't respond correctly because it was being driven by old server data rather than live physics. The interaction felt wrong.

By predicting everything, all objects exist in the same time frame and interact through the same physics simulation, producing consistent and believable collisions.

### Remote Car Prediction with Input Decay

The client cannot know what inputs remote players are pressing. The simplest approach is to assume they continue their last known inputs, but this leads to significant overshoot and rubber-banding when players change direction.

Rocket League uses an **input decay** technique: predicted remote inputs are gradually faded out over several frames:

| Prediction Frame | Input Applied |
|-----------------|---------------|
| Frame 1 (most recent known input) | 100% of last known input |
| Frame 2 | 66% (two-thirds) |
| Frame 3 | 33% (one-third) |
| Frame 4+ | 0% (no input -- car coasts) |

### Pseudocode: Remote Car Input Decay

```python
def predict_remote_car_input(last_known_input, frames_since_last_known):
    """
    Decay remote player's predicted input over time.
    Undershooting (car coasts to a stop) looks much better than
    overshooting (car flies past, then rubber-bands back).
    """
    if frames_since_last_known <= 0:
        return last_known_input  # we have the actual input

    decay_factors = [1.0, 0.66, 0.33, 0.0]
    factor = decay_factors[min(frames_since_last_known, len(decay_factors) - 1)]

    predicted_input = Input()
    predicted_input.throttle = last_known_input.throttle * factor
    predicted_input.steer = last_known_input.steer * factor
    # Jump/boost are binary -- not decayed, just assumed "not pressed"
    predicted_input.jump = False
    predicted_input.boost = last_known_input.boost if factor > 0 else False

    return predicted_input
```

### Ball Prediction

The ball is fully simulated by the client's physics engine, bouncing off walls, responding to gravity, and interacting with cars. Since the ball has no "inputs" (only physics), its prediction is purely a physics forward-simulation from its last known state. Prediction errors for the ball arise when:
- A remote car hits the ball and the client mispredicted that car's position/velocity
- The ball's trajectory diverges due to floating-point differences between client and server

### Forced Updates for Critical Events

When a player performs a jump, dodge, or double-jump **near the ball**, the server forces an immediate network packet via `ForceNetPacketIfNearBall()`. This ensures that critical ball-contact events propagate to all clients as quickly as possible, rather than waiting for the next scheduled replication cycle.

---

## 6. Physics Determinism Challenges

Rocket League's physics are **not perfectly deterministic**. This is a pragmatic design decision with important consequences.

### Sources of Non-Determinism

- **Floating-point variance:** Different CPUs (and even different compiler optimizations) can produce slightly different floating-point results for the same calculations. Bullet Physics, like most physics engines, accumulates these tiny differences over time.
- **Constraint solver iteration order:** Bullet's constraint solver may process contacts in different orders depending on memory layout, which can produce slightly different results.
- **Platform differences:** The game runs on PC, PlayStation, Xbox, and Switch, each with different floating-point hardware characteristics.

### How They Handle It

Rather than pursuing perfect determinism (which would require fixed-point math, identical hardware, and extreme care -- as in RTS lockstep games), Rocket League accepts non-determinism and compensates through **frequent state replication**:

- The server sends authoritative physics state at ~60 Hz.
- Clients correct any drift through the rollback/resimulation process.
- Small errors are caught before they compound into visible problems.

### Patch 2.53 Determinism Fix

Psyonix specifically addressed physics determinism issues between different processor types in Patch 2.53, though they noted that the majority of desync cases were caused by other factors (network jitter, input buffer issues) rather than CPU-level floating-point differences.

---

## 7. Collisions Between Predicted and Interpolated Objects

One of Rocket League's most important design decisions was to **avoid mixing predicted and interpolated objects** in the same physics simulation. In the predecessor game SARPBC (which used snapshot interpolation), this mixing created a fundamental problem:

### The SARPBC Problem

```
LOCAL CAR: Predicted (present time T)
BALL:      Interpolated (past time T - latency)
REMOTE CARS: Interpolated (past time T - latency)
```

When the predicted local car contacts the interpolated ball, the ball doesn't respond physically because it's being driven by interpolation, not simulation. The client must wait for the server to report the ball's new trajectory, creating a visible delay between "I hit the ball" and "the ball moves."

### The Rocket League Solution

```
LOCAL CAR:   Predicted (present time T)
BALL:        Predicted (present time T)
REMOTE CARS: Predicted (present time T)
ALL OBJECTS: Same physics world, same time frame
```

By predicting everything in the same physics world, all collisions happen naturally. When the local car hits the ball, the ball responds immediately through the physics simulation. When a predicted remote car hits the predicted ball, that interaction also happens in real-time.

### The Tradeoff

Predicting everything means **more mispredictions** to correct. Remote car predictions will inevitably be wrong (the client is guessing at their inputs), and those wrong predictions cascade into wrong ball predictions. But the tradeoff is worth it: the game **feels** responsive and physically consistent, even if corrections are needed more often.

### Resimulation Corrects Everything

When server state arrives and reveals a misprediction:
1. **All** objects (local car, remote cars, ball) are snapped to their server-authoritative positions in the physics world.
2. The physics world is resimulated forward from the correction frame to the present.
3. During resimulation, the local car uses stored actual inputs, while remote cars use the best available predicted inputs.
4. The result is a corrected present state where all objects are physically consistent with each other.

```python
def resimulate_world(self, server_frame, server_state):
    """
    Restore ENTIRE physics world to server state, then resimulate.
    All objects -- local car, remote cars, ball -- are corrected together.
    """
    # Restore every physics body to server-authoritative state
    for obj_id, obj_state in server_state.items():
        self.physics_world.set_body_state(obj_id, obj_state)

    # Resimulate from server_frame to current_frame
    for frame in range(server_frame, self.current_frame):
        # Local car: use recorded actual inputs
        local_input = self.input_history.get(frame).input
        self.apply_car_input(self.local_car, local_input)

        # Remote cars: use best predicted inputs (with decay)
        for remote_car in self.remote_cars:
            predicted_input = self.predict_remote_car_input(
                remote_car.last_known_input,
                frame - remote_car.last_known_input_frame
            )
            self.apply_car_input(remote_car, predicted_input)

        # Step the entire physics world
        self.physics_world.step(DT)
```

---

## 8. Correction Smoothing (Visual vs Physics State)

When a correction occurs after resimulation, the physics state snaps immediately to the corrected position, but the **visual representation** is smoothed over time to avoid jarring teleportation.

### The Core Principle

**Physics snaps. Visuals smooth.**

This separation is critical: smoothing the physics state would break determinism completely, because the physics engine would be operating on positions that don't match the authoritative state. Instead:

- The **physics body** is teleported instantly to the corrected position.
- A **visual offset** is calculated (difference between where the object was rendering and where it should now render).
- The visual offset is **decayed over time**, causing the rendered mesh to smoothly converge on the physics position.

### Pseudocode: Visual Correction Smoothing

```python
class CorrectionSmoothing:
    def __init__(self):
        self.position_offset = Vector3(0, 0, 0)  # visual offset from physics
        self.rotation_offset = Quaternion.identity()
        self.smoothing_rate = 0.1  # fraction to decay per frame

    def apply_correction(self, old_visual_pos, new_physics_pos,
                         old_visual_rot, new_physics_rot):
        """
        Called after resimulation. Physics has already been snapped
        to the corrected state. We just need to track the visual offset.
        """
        self.position_offset = old_visual_pos - new_physics_pos
        self.rotation_offset = old_visual_rot * inverse(new_physics_rot)

    def get_render_transform(self, physics_position, physics_rotation):
        """
        Called every render frame. Returns the smoothed visual position
        by blending the offset toward zero.
        """
        # Decay the offset over time
        self.position_offset = lerp(self.position_offset, Vector3.zero,
                                     self.smoothing_rate)
        self.rotation_offset = slerp(self.rotation_offset,
                                      Quaternion.identity,
                                      self.smoothing_rate)

        # Render position = physics position + remaining offset
        visual_pos = physics_position + self.position_offset
        visual_rot = self.rotation_offset * physics_rotation

        return visual_pos, visual_rot
```

### What This Looks Like in Practice

- **Small corrections** (a few units of position): The visual mesh slides smoothly to the corrected position over a few frames. Players rarely notice.
- **Large corrections** (significant desync due to network issues): There may be a visible "rubber-band" effect, but it's still smoother than an instant teleport.
- **The physics simulation is always authoritative** -- gameplay collision checks, boost pickups, goal detection all use the snapped physics position, not the smoothed visual position.

### Debug Visualization

Rocket League includes a debug mode where a red box shows the **server-authoritative car position**. The distance between the red box and the player's visible car represents the combined effect of prediction error, latency, and input buffer size.

---

## 9. The Input Buffer

The input buffer is a critical component that sits on the server side, absorbing network jitter to ensure the server always has inputs available when it needs to simulate a frame.

### How It Works

1. The client sends inputs tagged with frame numbers. Each outgoing packet contains the **last 10 unacknowledged input frames** for redundancy against packet loss.
2. The server maintains a small buffer of future inputs for each player.
3. When the server is ready to simulate frame `N`, it pulls the input for frame `N` from the buffer.
4. If the buffer is empty (inputs haven't arrived yet), the server **repeats the last known input** rather than waiting.

### Dynamic Buffer Sizing

The input buffer size is **not fixed** -- it grows and shrinks based on observed network jitter:

- A player with 25ms jitter gets a ~25ms input buffer (~3 frames at 120 Hz).
- A player with 50ms jitter gets a ~50ms input buffer (~6 frames at 120 Hz).
- At 4 frames of buffer at 120 Hz, the added latency is approximately **33ms**.

### STS vs Legacy Buffering

Rocket League later introduced two input buffering modes:

- **Legacy:** If the server's input buffer runs empty, it repeats the last known input. This causes minor desyncs because the client and server are no longer processing the same inputs for the same frames. On unstable connections, this feels poor.
- **STS (Speed To Server):** An improved algorithm that better handles buffer management to reduce desyncs on jittery connections.

### Frame-Based Communication

The system communicates in **frames** (physics ticks) rather than continuous timestamps. Both client and server index data by frame number, ensuring unambiguous correspondence between inputs and simulation steps.

---

## 10. Unique Challenges of a Physics-Based Competitive Game

Rocket League faces several challenges that most networked games do not:

### Everything Is Physics

In a typical FPS, only player movement is predicted; projectiles are often cosmetic, and hit detection uses raycasts. In Rocket League, **every gameplay-relevant interaction** is a physics collision:
- Car hitting ball
- Car hitting car (demos, bumps)
- Ball hitting walls/ceiling/floor
- Car landing, jumping, dodging
- Ball bouncing

Every one of these interactions must feel correct on the client, even though the true outcome can only be determined on the server.

### Cascading Mispredictions

Because everything is interconnected through physics, a single misprediction cascades:
1. Remote car's input is mispredicted.
2. Remote car's position diverges from server truth.
3. Remote car hits the ball at a slightly different angle than on the server.
4. Ball trajectory diverges.
5. Local car, predicting based on the wrong ball position, makes incorrect gameplay decisions.

The input decay technique helps limit the magnitude of these cascades by ensuring remote car predictions err on the side of "less movement" rather than "wrong movement."

### Resimulation Performance

The client must potentially resimulate **many frames** within a single render frame. At 120 Hz physics with 100ms latency, that's ~12 frames of resimulation that must complete within ~8ms (half a render frame at 60 FPS). Rocket League's physics world is intentionally simple:
- Few rigid bodies (~6-8 cars + 1 ball)
- Simple collision geometry (box/cylinder hitboxes, arena is a static mesh)
- No complex joints or soft bodies

This simplicity makes resimulation fast enough to be practical.

### Competitive Integrity

In a competitive esports title, players are acutely sensitive to:
- **"Ghost hits"** where the ball appears to pass through the car (the client predicted a hit, but the server disagrees)
- **Rubberbanding** of cars or the ball after corrections
- **Demo/bump inconsistencies** where a collision appears to occur on one client but not on the server

Psyonix addresses these through the combination of high tick rate (120 Hz), high update rate (60 Hz), forced packets on critical events, and the smoothing system described above.

### One-Frame Input Delay

Due to the order of operations in Rocket League's physics loop, steering and acceleration forces are computed **after** the physics engine advances the car each frame. This means there is an inherent one-frame input delay (approximately 8.3ms at 120 Hz) that has existed since launch. At 120 Hz, this is generally imperceptible.

---

## 11. Implementation Concepts Summary

### Complete Client Frame Loop

```python
class RocketLeagueClient:
    def __init__(self):
        self.physics_world = BulletPhysicsWorld()
        self.current_frame = 0
        self.input_history = RingBuffer(256)
        self.state_history = RingBuffer(256)
        self.smoothing = {}  # object_id -> CorrectionSmoothing

    def game_loop(self, dt):
        # Accumulate time, run fixed-step physics
        self.time_accumulator += dt
        while self.time_accumulator >= PHYSICS_DT:  # PHYSICS_DT = 1/120
            self.physics_tick()
            self.time_accumulator -= PHYSICS_DT

        # Render with visual smoothing
        self.render()

    def physics_tick(self):
        # 1. Gather local input
        local_input = self.poll_input()
        self.input_history.store(self.current_frame, local_input)

        # 2. Apply local input to local car
        self.apply_input(self.local_car, local_input)

        # 3. Predict remote car inputs (with decay)
        for car in self.remote_cars:
            predicted = predict_remote_input(
                car.last_known_input,
                self.current_frame - car.last_input_frame
            )
            self.apply_input(car, predicted)

        # 4. Step physics (all objects -- cars + ball)
        self.physics_world.step(PHYSICS_DT)

        # 5. Save state snapshot
        self.state_history.store(self.current_frame, self.save_world_state())

        # 6. Send inputs to server (batched, with redundancy)
        self.network.send_inputs(self.get_unacked_inputs())

        self.current_frame += 1

    def on_server_snapshot(self, server_frame, server_state):
        # Compare server state with our prediction for that frame
        our_state = self.state_history.get(server_frame)
        if not needs_correction(our_state, server_state):
            return

        # Save current visual positions for smoothing
        old_visuals = self.save_visual_positions()

        # ROLLBACK: Restore entire physics world to server state
        self.restore_world_state(server_frame, server_state)

        # RESIMULATE: Replay from server_frame to current_frame
        for frame in range(server_frame + 1, self.current_frame):
            # Local car gets actual recorded inputs
            self.apply_input(self.local_car, self.input_history.get(frame))

            # Remote cars get predicted inputs
            for car in self.remote_cars:
                predicted = predict_remote_input(
                    car.last_known_input,
                    frame - car.last_input_frame
                )
                self.apply_input(car, predicted)

            self.physics_world.step(PHYSICS_DT)
            # Update saved state history with corrected states
            self.state_history.store(frame, self.save_world_state())

        # Calculate visual offsets for smooth correction
        new_visuals = self.save_visual_positions()
        for obj_id in old_visuals:
            self.smoothing[obj_id].apply_correction(
                old_visuals[obj_id], new_visuals[obj_id]
            )

    def render(self):
        for obj in self.all_objects:
            physics_pos = obj.physics_position
            physics_rot = obj.physics_rotation
            # Apply visual smoothing offset
            visual_pos, visual_rot = self.smoothing[obj.id].get_render_transform(
                physics_pos, physics_rot
            )
            obj.mesh.render_at(visual_pos, visual_rot)
```

### Server Frame Loop

```python
class RocketLeagueServer:
    def __init__(self):
        self.physics_world = BulletPhysicsWorld()
        self.current_frame = 0
        self.input_buffers = {}  # player_id -> InputBuffer
        self.send_timer = 0

    def physics_tick(self):
        # 1. Pull inputs from each player's buffer
        for player_id, car in self.cars.items():
            buffer = self.input_buffers[player_id]
            if buffer.has_input(self.current_frame):
                input = buffer.pop(self.current_frame)
                car.last_input = input
            else:
                # Buffer empty -- repeat last known input
                input = car.last_input

            self.apply_input(car, input)

        # 2. Step physics (authoritative)
        self.physics_world.step(PHYSICS_DT)

        # 3. Send state to clients at ~60 Hz (every other frame)
        self.send_timer += 1
        if self.send_timer >= 2:
            self.broadcast_state_snapshot(self.current_frame)
            self.send_timer = 0

        self.current_frame += 1

    def on_client_inputs(self, player_id, input_frames):
        """
        Client sends up to 10 unacknowledged input frames per packet.
        Redundancy handles packet loss -- if earlier packet was lost,
        this one carries the missing inputs.
        """
        buffer = self.input_buffers[player_id]
        for frame_num, input in input_frames:
            if frame_num >= self.current_frame:  # only future frames
                buffer.store(frame_num, input)

    def force_net_packet_if_near_ball(self, player):
        """
        Called when a player jumps/dodges near the ball.
        Forces immediate state replication rather than waiting
        for the next scheduled send.
        """
        if distance(player.car.position, self.ball.position) < THRESHOLD:
            self.send_state_snapshot_now()
```

---

## 12. Key Takeaways

1. **Predict everything in the same time frame.** Mixing predicted and interpolated objects in a physics game creates fundamentally broken interactions. Rocket League's biggest networking improvement over its predecessor was moving everything to prediction.

2. **Physics snaps, visuals smooth.** Never smooth the physics state -- it destroys consistency. Instead, track a visual offset and decay it over time. The physics engine always operates on the authoritative corrected state.

3. **Input decay for remote prediction.** When predicting remote players, decay their inputs toward zero over a few frames. Undershooting (car slows down) is far less noticeable than overshooting (car flies past and rubber-bands).

4. **Use an open-source physics engine you can control.** The ability to save/restore complete physics state is non-negotiable for rollback. Bullet's open source nature made this possible where PhysX's UE3 integration did not.

5. **Keep the physics world simple.** Resimulation must be fast enough to run 10-15 frames within a single render frame. Simple collision geometry and few rigid bodies make this feasible.

6. **Accept non-determinism, compensate with replication.** Perfect determinism is extremely difficult across platforms. Instead, replicate authoritative state frequently enough to catch and correct drift before it compounds.

7. **Buffer inputs on the server to absorb jitter.** A dynamic input buffer sized to each player's connection quality prevents the server from running out of inputs, at the cost of a small amount of additional latency.

8. **Force-send critical events.** Don't let replication scheduling delay critical gameplay moments like ball contacts. Immediate forced packets ensure these events propagate with minimum latency.

---

## References

- [GDC Vault: "It IS Rocket Science! The Physics of Rocket League Detailed" (video)](https://www.gdcvault.com/play/1024972/It-IS-Rocket-Science-The)
- [GDC 2018 Presentation Slides (PDF) -- Jared Cone](https://media.gdcvault.com/gdc2018/presentations/Cone_Jared_It_Is_Rocket.pdf)
- [Alternate PDF mirror (UBM/S3)](https://ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc2018/presentations/Cone_Jared_It_Is_Rocket.pdf)
- [Rocket League at GDC 2018 -- Official Announcement](https://www.rocketleague.com/en/news/rocket-league-at-gdc-2018/)
- [Psyonix_Cone on netcode & extrapolation (DevTrackers)](https://devtrackers.gg/rocket-league/p/bb42f194-rocket-league-netcode-question-is-your-local-client-constantly-extrapolating-the-ball-other-players)
- [Psyonix_Cone on physics sync & lag compensation (DevTrackers)](https://devtrackers.gg/rocket-league/p/deb920a1-how-does-physics-synchronization-and-lag-compensation-work-in-rocket-league)
- [Psyonix_Cone on netcode & lag (DevTrackers)](https://devtrackers.gg/rocket-league/p/c2e0e1bc-netcode-lag-explained-with-lots-of-examples-rocket-science-16)
- [Bullet Physics btRaycastVehicle (source)](https://github.com/bulletphysics/bullet3/blob/master/src/BulletDynamics/Vehicle/btRaycastVehicle.h)
- [Bullet Physics: Rocket League integration announcement](https://pybullet.org/wordpress/index.php/2018/03/15/rocket-league-using-bullet-physics-in-unreal-engine-4/)
- [SnapNet: Netcode Architectures Part 2 -- Rollback (input decay reference)](https://www.snapnet.dev/blog/netcode-architectures-part-2-rollback/)
- [HeavyCar Research: Known facts about Rocket League internals](https://gitlab.com/heavycar/research/-/issues/2)
- [RocketScience.fyi: Known Issues in Rocket League](https://rocketscience.fyi/know/rl/issues)
- [Game Developer: How Rocket League's netcode stacks up](https://www.gamedeveloper.com/game-platforms/how-i-rocket-league-i-s-netcode-stacks-up-against-other-online-games)
- [Game Developer: Video summary of GDC talk](https://www.gamedeveloper.com/design/video-the-rocket-science-behind-i-rocket-league-s-i-physics)
