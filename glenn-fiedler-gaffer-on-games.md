# Glenn Fiedler's Networked Physics & Game Networking -- Gaffer on Games

## Overview

Glenn Fiedler (known online as "Gaffer on Games") is one of the most influential voices in game networking and networked physics. His articles on [gafferongames.com](https://gafferongames.com), written from 2004 through the late 2010s, have become essential reading for anyone implementing multiplayer game networking. He has worked as a network programmer in the game industry and later founded a networking middleware company (Network Next / mas-bandwidth.com).

His key contributions include:

- **Categorizing the three fundamental networking models** (peer-to-peer lockstep, client/server, client-side prediction) in a way that became the standard mental framework for game developers.
- **The "Networked Physics" article series**, which methodically demonstrates three approaches to synchronizing a physics simulation: deterministic lockstep, snapshot interpolation, and state synchronization.
- Practical techniques for **snapshot compression**, **delta encoding**, **priority accumulators**, **jitter buffers**, and **visual smoothing** that are widely adopted in the industry.
- **Networked Physics in VR** research (sponsored by Oculus), demonstrating a distributed authority model for cooperative VR physics experiences with Unity and PhysX.
- Numerous articles on building reliable protocols over UDP, fixed timestep game loops, and floating point determinism.

His writing style is notable for being deeply practical -- he builds a real physics simulation (cubes driven by ODE or PhysX), networks it three different ways, and shows video of the results at various latency/packet-loss conditions, making the tradeoffs viscerally clear.

---

## 1. What Every Programmer Needs to Know About Game Networking

**Source:** [gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/)

This foundational article traces the historical evolution of multiplayer networking through three models:

### Model 1: Peer-to-Peer Lockstep

The earliest model, used in RTS games (Age of Empires, StarCraft, Command & Conquer). The game is abstracted into a series of **turns**; only the **command messages** (move unit, attack, build) are exchanged between peers. All machines start from a common initial state and process identical commands, producing identical results.

**Limitations:**
- Requires **perfect determinism** -- the tiniest floating point difference cascades into total desynchronization.
- Every player must **wait for all other players' inputs** before advancing a turn, so latency equals that of the most lagged player.
- **Late join is very difficult** because it requires capturing and transmitting a fully deterministic starting point mid-game.

**Why it persists:** For RTS games with thousands of units, the game state is simply too large to transmit directly. Sending commands is the only feasible option.

### Model 2: Client/Server

Introduced by Quake (1996). Instead of peer-to-peer, all players connect as **clients** to a single **server**. The server runs the authoritative simulation; clients act as "dumb terminals" sending inputs and receiving state.

**Advantages over P2P lockstep:**
- No need for determinism.
- Game quality depends on each client's connection to the server, not on the worst peer.
- Players can join/leave mid-game easily.
- Reduced bandwidth per-player.

**Problem:** In the pure form, every action has noticeable latency (round-trip to server and back).

### Model 3: Client-Side Prediction

The breakthrough by John Carmack in QuakeWorld (1996). The client runs a **subset of the game code locally** to predict the player's own movement immediately, while the server remains fully authoritative.

Key mechanics:
- The client predicts movement from local input and renders it immediately -- no waiting for the server.
- The server sends corrections back to the client.
- The client maintains a **circular buffer of past states and inputs**. When a correction arrives (necessarily in the past due to RTT), the client **rewinds to the corrected state and replays** all buffered inputs from that point forward.
- Because client and server run the same simulation code, corrections are rare -- they happen only when external events (being hit by a rocket, colliding with another player) affect the character.

As Tim Sweeney (Unreal Engine) put it: **"The Server Is The Man."**

---

## 2. Deterministic Lockstep

**Source:** [gafferongames.com/post/deterministic_lockstep](https://gafferongames.com/post/deterministic_lockstep/)

### Concept

Deterministic lockstep networks a simulation by sending **only inputs**, not state. The same simulation runs on all machines. Given identical initial conditions and identical inputs applied at identical times, the simulation produces bit-identical results.

**Bandwidth is proportional to input size, not the number of objects.** You can theoretically network a million-object simulation with the same bandwidth as one object.

### Determinism Requirements

Determinism means **exactly the same result**, down to the bit level. You could checksum the entire physics state each frame and it would match across machines.

Fiedler demonstrates this with ODE (Open Dynamics Engine), which uses a random number generator inside its solver to randomize constraint processing order. By calling `dSetRandomSeed(frameNumber)` before each simulation step, ODE produces identical results on the same machine/binary/OS.

**Critical caveat:** Determinism on the same machine does NOT guarantee determinism across:
- Different compilers
- Different operating systems
- Different CPU architectures (PowerPC vs. Intel)
- Debug vs. release builds (due to floating point optimizations)

### Floating Point Determinism Challenges

- Different compilers may reorder floating point operations or use different precision (80-bit x87 vs. 64-bit SSE).
- Compiler optimizations like fused multiply-add (FMA) can change results.
- Different CPUs may implement transcendental functions (sin, cos, sqrt) differently.
- Cross-platform determinism is extremely hard and often impractical.

### Implementation: Input Networking

Inputs are represented as a compact struct:

```c
struct Input {
    bool left;
    bool right;
    bool up;
    bool down;
    bool space;
    bool z;
};
```

Each frame, the sender samples this struct and transmits it. The receiver can only simulate frame `n` when it has received input for frame `n`. If input hasn't arrived, the simulation **must wait**.

### The Playout Delay Buffer (Jitter Buffer)

Since packets don't arrive evenly spaced (network jitter), a **playout delay buffer** is needed. This is analogous to video streaming buffering:

- Buffer incoming input packets for a short time.
- Deliver them to the simulation at regular intervals (e.g., every 1/60th of a second).
- If the buffer runs dry, the simulation must pause (hitch).
- If packets bunch up, allow up to 4 simulation steps per render frame to catch up (avoiding the "spiral of death").

The buffer introduces latency but provides smoothness. The cost of increasing buffer size is additional latency.

### TCP vs. UDP for Deterministic Lockstep

Fiedler demonstrates that **TCP performs terribly** for deterministic lockstep under real-world conditions:
- At 100ms latency + 1% packet loss: visible hitches every few seconds (TCP must wait ~2x RTT for retransmission).
- At 250ms latency + 5% packet loss: nearly unplayable.

**The UDP alternative -- redundant inputs:**

Since inputs are tiny (6 bits in this example), include **all unacknowledged inputs** in every UDP packet. If a packet is lost, the next packet contains all the data that was in the lost packet plus new data.

- With 60fps and worst-case 2-second RTT: 120 inputs * 6 bits = 720 bits = 90 bytes. Trivial.
- Further optimized by encoding deltas: write 1 bit if input changed from previous frame (common: 0 = same), 7 bits if different. Average overhead: ~15 bytes worst case.
- The receiver sends acks back so the sender knows which inputs were received and can discard old ones.

**Result:** At 2 seconds latency + 25% packet loss, the UDP approach works flawlessly -- no hitches whatsoever.

### When to Use Deterministic Lockstep

- Best for **2-4 players** (all players wait for the most lagged player).
- Best when simulation state is very large (thousands of units, like RTS games).
- Requires the simulation to be **fully deterministic**.
- Not suitable for games where late join is important.

---

## 3. Snapshot Interpolation

**Source:** [gafferongames.com/post/snapshot_interpolation](https://gafferongames.com/post/snapshot_interpolation/)

### Concept

The polar opposite of deterministic lockstep. The simulation runs **only on the server** (or authority side). The server captures a **snapshot** of all relevant state and transmits it to clients. Clients **do not run the simulation at all** -- they reconstruct a visual approximation by interpolating between received snapshots.

### Snapshot Data

Per-object state for rendering:

```c
struct CubeState {
    bool interacting;
    vec3f position;
    quat4f orientation;
};
```

Each cube serializes to ~225 bits (28 bytes). With 900 cubes: ~25 KB per snapshot.

### The Jitter Problem

Sending snapshots at 60 pps and rendering the most recent one causes visible hitches because packets don't arrive evenly spaced over the internet. Some frames you receive two packets, other frames you receive none.

### The Interpolation Buffer

Instead of rendering the most recent snapshot immediately, **buffer snapshots** for a short time so that you always have the current snapshot AND the next snapshot available. Interpolate between the two delayed snapshots, trading a small amount of **added latency for smoothness**.

### Linear vs. Hermite Interpolation

**Linear interpolation at 10 pps** looks surprisingly good but has artifacts:
- Position jitter when hovering (brain detects 1st-order discontinuity at sample points).
- "Pulsing" in rotating groups of objects (objects interpolate through other objects taking the shortest linear path between two points on a circle).

**Hermite spline interpolation** eliminates these artifacts at the same send rate by incorporating linear velocity at each sample point. The hermite spline passes through start/end points while matching start/end velocities, producing smooth velocity across sample points.

For orientation, **slerp** (spherical linear interpolation) is sufficient -- no need for higher-order orientation interpolation. This avoids needing to send angular velocity.

### Handling Packet Loss

Snapshots are sent over **UDP** (never TCP) because they are time-critical and non-reliable. If a snapshot is lost, skip it and interpolate toward a more recent one.

**Rule of thumb:** The interpolation buffer should have enough delay to survive **two consecutive lost packets**.

At 10 pps with 2-5% packet loss: **300ms buffer + ~50ms jitter margin = 350ms total delay**.

### Extrapolation

Fiedler tried extrapolation (200ms) to reduce delay from 350ms to 150ms but found it **does not work well for rigid body physics** because:
- Extrapolation doesn't know about collisions (cubes go through the floor).
- It doesn't know about forces (spring forces, gravity interactions).
- Attached objects predict along tangent velocity instead of rotating.

### Bandwidth vs. Latency Tradeoff

Higher send rates reduce the required interpolation delay, but increase bandwidth:

| Send Rate | Interpolation Delay | Bandwidth (uncompressed) |
|-----------|-------------------|--------------------------|
| 10 pps    | ~350ms            | ~2 Mbps                  |
| 30 pps    | ~150ms            | ~6 Mbps                  |
| 60 pps    | ~85ms             | ~12 Mbps                 |

The solution is aggressive **snapshot compression** (see next section) to enable higher send rates within bandwidth budgets.

---

## 4. Snapshot Compression

**Source:** [gafferongames.com/post/snapshot_compression](https://gafferongames.com/post/snapshot_compression/)

Starting point: 60 pps uncompressed = **17.37 Mbps**. Target: **256 Kbps**.

### Orientation: Smallest Three

Instead of sending a full quaternion (128 bits), use the **smallest three** representation:
1. Find the component with the largest absolute value.
2. Send 2 bits for its index (x=0, y=1, z=2, w=3).
3. Send the other three components, quantized.
4. Negate the entire quaternion if the largest component is negative (so it's always reconstructed as positive).
5. The three "smallest" components are bounded in `[-1/sqrt(2), +1/sqrt(2)]` (not `[-1, +1]`), gaining extra precision.

At 9 bits per component: **2 + 9 + 9 + 9 = 29 bits** (down from 128). Saves over 5 Mbps.

### Linear Velocity: Bound, Quantize, At-Rest Flag

- Bound to [-32, +31] m/s with 32 values per m/s precision.
- 11 bits per component = **33 bits total** (down from 96).
- Add a 1-bit **"at rest" flag**: if set, velocity is implicitly zero and not sent.
- At 60 pps, linear velocity isn't even needed (linear interpolation is sufficient). **Don't send it at all.**

### Position: Bound and Quantize

- Bound to [-256, +255] meters (xy) and [0, 32] meters (z).
- 512 values per meter (~2mm precision).
- **18 + 18 + 14 = 50 bits** (down from 96).

After these optimizations: **80 bits (10 bytes) per cube**, approximately 1/4 original size.

### Delta Compression

The biggest remaining win. Encode each snapshot **relative to a baseline** (a previously acknowledged snapshot).

1. For unchanged cubes: send just **1 bit** ("not changed").
2. The receiver continuously acks received snapshots. The sender uses the most recent ack as the baseline.
3. For the first RTT (no ack yet), encode relative to the initial simulation state.

### Per-Cube Indexing Optimization

- With 901 cubes and few changes per frame, sending a **10-bit index** per changed cube is cheaper than 901 "changed/not-changed" bits.
- Encode cube indices **relative to the previous index** (delta index): most offsets are small.
- Variable-length encoding: [1-8] = 4 bits, [9-40] = 7 bits, [41-900] = 12 bits. Average: **5.5 bits per index**.

### Delta Position and Orientation

- **Position deltas:** Per-component, 1 bit "small" flag. Small = 5 bits [-16,+15]. Large = 9 bits [-256,+255]. Average: **26.1 bits** (down from 50).
- **Orientation deltas (smallest three):** 90% of orientations share the same largest component index as their baseline. Encode deltas directly in smallest-three space. Small = 5 bits, large = 8 bits. Average: **23.3 bits** (80% of absolute smallest three).

### End Result

From 17.37 Mbps to approximately **256 Kbps** at 60 snapshots per second -- a roughly **98.5% reduction**.

---

## 5. State Synchronization

**Source:** [gafferongames.com/post/state_synchronization](https://gafferongames.com/post/state_synchronization/)

### Concept

A hybrid of deterministic lockstep and snapshot interpolation. The simulation runs on **both sides**, and **both inputs and state** are sent. Because the simulation runs locally, objects continue to move between updates (extrapolation happens naturally). Because state is sent, perfect determinism is not required.

Only a **subset of objects** needs to be sent per packet, and a **priority accumulator** determines which objects are most important.

### Per-Object State

```c
struct StateUpdate {
    int index;
    vec3f position;
    quat4f orientation;
    vec3f linear_velocity;
    vec3f angular_velocity;
};
```

Unlike snapshot interpolation, velocities **must** be sent because the remote side is always extrapolating from the last received state. Incorrect velocities cause pops on correction.

At-rest optimization: if an object is at rest, don't send velocities (just 1 bit flag).

### Priority Accumulator System

Since only ~64 state updates fit per packet (out of 901 objects), a prioritization system selects the most important updates.

**How it works:**

```
for each object:
    current_priority = calculate_priority(object)
    // e.g., player cube = 1000000, interacting = 100, at rest = 1

    priority_accumulator[object] += current_priority

sort objects by priority_accumulator (descending)

for each object in sorted order:
    if state_update fits in packet:
        write state_update to packet
        priority_accumulator[object] = 0  // reset: give others a chance
    else:
        skip (accumulator value preserved for next packet)
```

**Key properties:**
- Important objects (high priority) rise faster in the accumulator and get sent more often.
- Objects that are skipped or don't fit keep accumulating, guaranteeing they eventually get sent.
- Bandwidth can be **dynamically adjusted** -- just change the maximum packet size. Quality scales automatically.
- Good for congestion avoidance: reduce bandwidth under bad conditions, increase when conditions improve.

### Jitter Buffer

On the receiving side, packets must be buffered to smooth out network jitter before applying state updates.

Without a jitter buffer, packets arriving in bursts (2 packets one frame, 0 the next) cause objects in different packets to be slightly out of phase with each other, creating pops and divergence in stacks.

Buffer delay: **4-5 frames at 60 Hz** (~67-83ms) -- much smaller than snapshot interpolation's delay because the simulation runs locally and provides extrapolation between updates.

### Applying State Updates: "Snap Physics Hard"

**Apply state updates directly to the simulation (hard snap).** Do NOT smooth or blend at the simulation level.

Rationale: The simulation extrapolates from the applied state. Extrapolating from a "smoothed, made-up" state produces incorrect physics and causes worse artifacts, especially in stacks of objects. Extrapolating from a valid physics state (the exact state from the authority) keeps physics correct.

### Quantize Both Sides

If state is quantized for bandwidth savings, the sender must also quantize its own simulation state before each step. Otherwise, the receiver extrapolates from quantized values while the sender extrapolates from full-precision values, causing systematic divergence and pops.

For state synchronization, **higher precision** is needed than for snapshot interpolation:
- **Position:** 4096 values per meter (up from 512 for snapshot interpolation)
- **Orientation:** 15 bits per smallest-three component (up from 9)

This is because quantized values are fed back into the physics solver, and insufficient precision causes penetration/depenetration fighting.

### Visual Smoothing

While the simulation receives hard-snapped state, the **rendering layer** uses error offset smoothing:

1. Maintain a **position error offset** and **orientation error quaternion** per object.
2. When a state update arrives, calculate the visual difference between the current smoothed visual position and the new simulation position. This becomes the new error offset.
3. Each frame, reduce the error offset toward zero:
   - Position: multiply by a decay factor (e.g., 0.9) each frame.
   - Orientation: slerp toward identity by a fraction (e.g., 0.1) each frame.
4. Render at: `simulation_position + position_error`, `simulation_orientation * orientation_error`.

**Adaptive smoothing:** A single decay factor doesn't work well for all error magnitudes:
- Small errors (< 25cm): use 0.95 (slow, smooth fade).
- Large errors (> 1m): use 0.85 (fast correction -- large pops are disorienting if dragged out).
- Blend linearly between the two factors based on error magnitude.

The same approach applies to orientation using the dot product between error quaternion and identity.

### Delta Compression for State Synchronization

More complex than snapshot delta compression because not all objects are in every packet.

- **Per-object baselines:** Each object's delta is encoded relative to the most recently acknowledged state for *that specific object* (not a global snapshot number).
- Requires a **bidirectional ack system** over UDP to know exactly which packets were received.
- Send 1 bit per update: relative or absolute. If relative, include 5 bits identifying the base packet sequence offset (relative to the most recent globally acked packet). Absolute fallback for objects with stale baselines.
- Objects that recently came to rest get **elevated priority** until their at-rest state is acked, preventing stale positions after packet loss.

---

## 6. Networked Physics in Virtual Reality

**Source:** [gafferongames.com/post/networked_physics_in_virtual_reality](https://gafferongames.com/post/networked_physics_in_virtual_reality/)

### Unique Challenges

Fiedler worked with Oculus to develop a networked physics sample for VR with Touch controllers. Players pick up, throw, stack, and interact with physically simulated cubes.

**VR-specific requirements:**
1. Interactions must be **latency-free** for the local player (picking up, throwing, catching).
2. Stacks of cubes must be **stable and jitter-free** in the remote view.
3. Interactions from remote players should also appear without latency wherever possible.

### Why Not Deterministic Lockstep or Client/Server?

- **Deterministic lockstep** is ruled out because PhysX is non-deterministic. Even if it were deterministic, hiding 250ms of latency at 90 Hz VR would require 25 physics steps per render frame (GGPO style) -- too expensive.
- **Client/server with prediction** would require expensive rollback/resimulation and dedicated servers. For a cooperative (non-competitive) experience, the security benefit doesn't justify the cost.

### Chosen Model: Distributed Simulation with Authority

Players take **authority** over cubes they interact with. The authority-holder sends state for those cubes to other players.

**Two key concepts:**
- **Authority:** Set to the player who last interacted with a cube. Can transfer between players (e.g., a thrown cube hitting someone else's stack transfers authority recursively to all affected cubes).
- **Ownership:** Exclusive possession (e.g., holding a cube in your hand). Once owned, no other player can take it until released. Ownership takes precedence over authority.

Both are encoded as **per-cube sequence numbers** (authority sequence, ownership sequence) included in state updates. Higher sequence numbers win conflicts. The host acts as arbiter.

### Quantize-Both-Sides in PhysX

Quantizing state caused several PhysX-specific problems:
- PhysX pushes penetrating objects apart with huge velocities. Fix: set `maxDepenetrationVelocity = 1 m/s` per rigid body.
- Quantized rotations cause cubes to slide across floors in feedback loops.
- Cubes in stacks jitter because quantization puts them just above surfaces, causing micro-falls. Fix: disable PhysX's built-in at-rest detection and replace it with a **ring buffer of positions/rotations per cube**. If a cube hasn't moved significantly in 16 frames, force it to rest.

### Avatar Synchronization

Avatar state (head + two hands from HMD/Touch controllers) is sampled at render framerate but applied at physics framerate, causing jitter. Fix: include the **time offset** between physics and render time in avatar state, and use a **100ms jitter buffer** with interpolation to reconstruct avatar samples at the correct time.

Cubes held by avatars: set priority factor to -1 (don't send in regular state updates). Instead, include the cube's id and relative transform as part of avatar state.

### Delta Compression with Prediction

Fiedler implemented a **ballistic predictor** (written in fixed point for determinism) that predicts cube positions under gravity. When prediction matches within quantization resolution (~90% of the time), the cube is encoded as just 1 bit: "perfect prediction." Remaining cases encode small error offsets relative to prediction.

**Target:** < 256 Kbps per player, < 1 Mbps total for 4 players.

---

## 7. Implementation Examples (Pseudo-code)

### Priority Accumulator System

```
class PriorityAccumulator:
    accumulators: float[NUM_OBJECTS]  // persists between frames, initialized to 0

    function get_updates_for_packet(objects, max_packet_bytes, header_bytes):
        // Step 1: Accumulate priorities
        for each object in objects:
            priority = calculate_priority(object)
            if priority < 0:
                accumulators[object.id] = -1.0
            else:
                accumulators[object.id] += priority

        // Step 2: Sort by accumulated priority (descending)
        sorted_objects = sort(objects, key=accumulators[obj.id], descending)

        // Step 3: Fill packet respecting bandwidth budget
        bytes_remaining = max_packet_bytes - header_bytes
        updates_to_send = []

        for each object in sorted_objects:
            if accumulators[object.id] < 0:
                continue
            update_size = estimate_update_size(object)
            if update_size <= bytes_remaining:
                updates_to_send.append(object)
                bytes_remaining -= update_size
                accumulators[object.id] = 0.0   // reset on send
            // else: accumulator value preserved -- this object will
            //        have even higher priority next frame

        return updates_to_send

    function calculate_priority(object):
        if object.is_player_controlled:
            return 1000000.0    // always include
        if object.is_interacting:
            return 100.0
        if object.is_at_rest:
            return 1.0
        return 10.0             // moving but not directly interacting
```

### Jitter Buffer

```
class JitterBuffer:
    buffer: map<int, Packet>    // sequence -> packet
    buffer_delay_frames: int    // e.g., 4-5 frames at 60 Hz
    first_sequence: int = -1
    first_receive_time: float = -1.0

    function receive_packet(packet):
        seq = packet.sequence
        if first_sequence < 0:
            first_sequence = seq
            first_receive_time = current_time()
        buffer[seq] = packet

    function get_packet_for_frame(frame_number):
        // Calculate which sequence should play at this time,
        // accounting for the buffer delay
        target_time = first_receive_time + buffer_delay_frames * dt
                      + (frame_number - first_sequence) * dt

        if current_time() < target_time:
            return null   // too early, hold the packet

        if buffer contains frame_number:
            packet = buffer[frame_number]
            remove frame_number from buffer
            return packet

        return null  // packet lost or hasn't arrived yet

    // Simplified approach for state synchronization:
    // Buffer all packets, deliver them spaced dt apart
    // starting buffer_delay_frames after the first packet arrives
```

### Snapshot Delta Encoding

```
class SnapshotDeltaEncoder:
    baseline_snapshot: Snapshot = initial_state
    baseline_sequence: int = -1

    function encode_snapshot(current_snapshot, current_sequence):
        packet = new Packet()
        packet.sequence = current_sequence
        packet.baseline_sequence = baseline_sequence

        if baseline_sequence < 0:
            packet.is_initial_baseline = true
            baseline = initial_state
        else:
            packet.is_initial_baseline = false
            baseline = baseline_snapshot

        // Determine changed vs. unchanged objects
        changed_objects = []
        for i in 0..NUM_OBJECTS:
            if current_snapshot[i] == baseline[i]:
                continue  // will encode as "not changed" (1 bit)
            changed_objects.append(i)

        // Choose encoding: relative indices vs. changed-bits
        if len(changed_objects) < 90:
            packet.use_indices = true
            encode_with_relative_indices(packet, changed_objects,
                                         current_snapshot, baseline)
        else:
            packet.use_indices = false
            encode_with_changed_bits(packet, current_snapshot, baseline)

        return packet

    function encode_with_relative_indices(packet, changed, current, baseline):
        prev_index = 0
        for index in changed:
            delta_index = index - prev_index
            write_variable_length_index(packet, delta_index)
            encode_object_delta(packet, current[index], baseline[index])
            prev_index = index

    function encode_object_delta(packet, current_state, baseline_state):
        // Position delta: per-component small/large encoding
        for each component (x, y, z):
            delta = current.position[c] - baseline.position[c]
            if abs(delta) <= 15:
                write_bit(packet, 1)        // small flag
                write_bits(packet, delta, 5)
            elif abs(delta) <= 255:
                write_bit(packet, 0)        // large flag
                write_bits(packet, delta, 9)
            else:
                // fallback to absolute
                write_absolute_position(packet, current.position)
                return

        // Orientation delta in smallest-three space
        encode_orientation_delta(packet, current.orientation,
                                 baseline.orientation)

    function on_ack_received(acked_sequence, acked_snapshot):
        if acked_sequence > baseline_sequence:
            baseline_sequence = acked_sequence
            baseline_snapshot = acked_snapshot
```

### State Synchronization Loop

```
// ============================================================
// AUTHORITY SIDE (sender)
// ============================================================
class AuthoritySide:
    priority_accumulator: PriorityAccumulator
    simulation: PhysicsSimulation
    per_object_baselines: map<int, PerObjectBaseline>

    function tick(dt):
        // 1. Quantize state (ensure both sides extrapolate identically)
        for each object in simulation:
            state = object.get_state()
            quantized = quantize(state)
            object.set_state(quantized)

        // 2. Step physics
        simulation.step(dt)

        // 3. Gather state updates
        updates = priority_accumulator.get_updates_for_packet(
            simulation.objects, MAX_PACKET_SIZE, HEADER_SIZE)

        // 4. Build and send packet
        packet = new Packet()
        packet.sequence = current_frame
        packet.inputs = get_redundant_inputs()   // last N inputs

        for each object in updates:
            state_update = object.get_state()
            baseline = per_object_baselines.get(object.id)
            if baseline exists and baseline.sequence is recently acked:
                // Delta encode relative to baseline
                packet.write_delta(object.id, state_update, baseline)
            else:
                // Absolute encode
                packet.write_absolute(object.id, state_update)

        send(packet)

        // 5. Track what was sent (for delta compression)
        for each object in updates:
            sent_states[current_frame][object.id] = object.get_state()

    function on_ack(acked_sequence):
        // Update per-object baselines
        if acked_sequence in sent_states:
            for (object_id, state) in sent_states[acked_sequence]:
                per_object_baselines[object_id] = {
                    sequence: acked_sequence,
                    state: state
                }

// ============================================================
// NON-AUTHORITY SIDE (receiver)
// ============================================================
class NonAuthoritySide:
    simulation: PhysicsSimulation
    jitter_buffer: JitterBuffer
    visual_errors: map<int, VisualError>   // per-object smoothing state

    function tick(dt):
        // 1. Process packets from jitter buffer
        packet = jitter_buffer.get_packet_for_frame(current_frame)
        if packet != null:
            apply_inputs(packet.inputs)
            for each state_update in packet.state_updates:
                apply_state_update(state_update)

        // 2. Quantize state (match authority side)
        for each object in simulation:
            state = object.get_state()
            quantized = quantize(state)
            object.set_state(quantized)

        // 3. Step physics (extrapolation happens naturally)
        simulation.step(dt)

        // 4. Reduce visual errors
        for each (object_id, error) in visual_errors:
            reduce_error(error)

    function apply_state_update(update):
        object = simulation.get_object(update.index)

        // Calculate new visual error offset
        old_visual_pos = object.position + visual_errors[update.index].pos_error
        old_visual_rot = object.rotation * visual_errors[update.index].rot_error

        // SNAP physics state hard
        object.position = update.position
        object.rotation = update.orientation
        object.linear_velocity = update.linear_velocity
        object.angular_velocity = update.angular_velocity

        // Recalculate visual error so visual position doesn't jump
        visual_errors[update.index].pos_error = old_visual_pos - update.position
        visual_errors[update.index].rot_error = old_visual_rot * conjugate(update.orientation)

    function reduce_error(error):
        // Adaptive smoothing: tight for large errors, gentle for small
        pos_magnitude = length(error.pos_error)

        if pos_magnitude < 0.25:
            factor = 0.95     // slow, smooth for small errors
        elif pos_magnitude > 1.0:
            factor = 0.85     // fast correction for large pops
        else:
            // Linear blend between thresholds
            t = (pos_magnitude - 0.25) / (1.0 - 0.25)
            factor = lerp(0.95, 0.85, t)

        error.pos_error *= factor
        if length(error.pos_error) < 0.001:
            error.pos_error = vec3(0, 0, 0)

        // Orientation: slerp toward identity
        rot_closeness = dot(error.rot_error, identity_quat)
        if rot_closeness < 0.1:
            rot_factor = 0.85
        elif rot_closeness > 0.5:
            rot_factor = 0.95
        else:
            t = (rot_closeness - 0.1) / (0.5 - 0.1)
            rot_factor = lerp(0.85, 0.95, t)

        error.rot_error = slerp(error.rot_error, identity_quat, 1.0 - rot_factor)

    function render(object):
        // Render at smoothed position, not raw simulation position
        render_pos = object.position + visual_errors[object.id].pos_error
        render_rot = object.rotation * visual_errors[object.id].rot_error
        draw(object.mesh, render_pos, render_rot)
```

---

## 8. References

### Core Articles

| Article | URL |
|---------|-----|
| What Every Programmer Needs to Know About Game Networking | [gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/) |
| Introduction to Networked Physics | [gafferongames.com/post/introduction_to_networked_physics](https://gafferongames.com/post/introduction_to_networked_physics/) |
| Deterministic Lockstep | [gafferongames.com/post/deterministic_lockstep](https://gafferongames.com/post/deterministic_lockstep/) |
| Snapshot Interpolation | [gafferongames.com/post/snapshot_interpolation](https://gafferongames.com/post/snapshot_interpolation/) |
| Snapshot Compression | [gafferongames.com/post/snapshot_compression](https://gafferongames.com/post/snapshot_compression/) |
| State Synchronization | [gafferongames.com/post/state_synchronization](https://gafferongames.com/post/state_synchronization/) |
| Networked Physics in Virtual Reality | [gafferongames.com/post/networked_physics_in_virtual_reality](https://gafferongames.com/post/networked_physics_in_virtual_reality/) |
| Floating Point Determinism | [gafferongames.com/post/floating_point_determinism](https://gafferongames.com/post/floating_point_determinism/) |

### Related Articles on Gaffer on Games

| Article | URL |
|---------|-----|
| Fix Your Timestep! | [gafferongames.com/post/fix_your_timestep](https://gafferongames.com/post/fix_your_timestep/) |
| UDP vs. TCP | [gafferongames.com/post/udp_vs_tcp](https://gafferongames.com/post/udp_vs_tcp/) |
| Reliability and Flow Control | [gafferongames.com/post/reliability_and_flow_control](https://gafferongames.com/post/reliability_and_flow_control/) |
| Serialization Strategies | [gafferongames.com/post/serialization_strategies](https://gafferongames.com/post/serialization_strategies/) |
| Networked Physics (2004) | [gafferongames.com/post/networked_physics_2004](https://gafferongames.com/post/networked_physics_2004/) |

### External References Cited by Fiedler

| Reference | URL |
|-----------|-----|
| 1500 Archers on a 28.8 (Age of Empires networking) | [gamasutra.com/view/feature/3094](http://www.gamasutra.com/view/feature/3094/1500_archers_on_a_288_network_.php) |
| The Unreal Networking Architecture (Tim Sweeney) | [docs.unrealengine.com/udk/Three/NetworkingOverview](https://docs.unrealengine.com/udk/Three/NetworkingOverview.html) |
| Snapshot compression source code (Fiedler) | [gist.github.com/gafferongames/bb7e593ba1b05da35ab6](https://gist.github.com/gafferongames/bb7e593ba1b05da35ab6) |
| Context-aware arithmetic encoding (Fabian Giesen) | [github.com/rygorous/gaffer_net](https://github.com/rygorous/gaffer_net/blob/master/main.cpp) |
| Oculus Networked Physics Sample (BSD licensed) | [github.com/OculusVR/oculus-networked-physics-sample](https://github.com/OculusVR/oculus-networked-physics-sample) |
| Hermite Interpolation (Wikipedia) | [en.wikipedia.org/wiki/Hermite_interpolation](https://en.wikipedia.org/wiki/Hermite_interpolation) |
| Glenn Fiedler's new blog | [mas-bandwidth.com](https://mas-bandwidth.com) |
