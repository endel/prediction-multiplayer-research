# Snapshot Interpolation


---

## Table of Contents

1. [Overview](#overview)
2. [History & Origins](#history--origins)
3. [How It Works Step by Step](#how-it-works-step-by-step)
4. [Snapshot Design](#snapshot-design)
5. [Delta Compression](#delta-compression)
6. [The Interpolation Buffer](#the-interpolation-buffer)
7. [Latency-Bandwidth Trade-off](#latency-bandwidth-trade-off)
8. [Clock Synchronization](#clock-synchronization)
9. [Handling Edge Cases](#handling-edge-cases)
10. [Pseudo-Code / Code Examples](#pseudo-code--code-examples)
11. [Comparison with Entity Interpolation](#comparison-with-entity-interpolation)
12. [Framework Implementations](#framework-implementations)
13. [When to Use / When Not to Use](#when-to-use--when-not-to-use)
14. [References](#references)

---

## Overview

**Snapshot interpolation** is a netcode architecture in which the server periodically captures the complete authoritative game state (a "snapshot"), transmits it to all connected clients, and clients visually reconstruct smooth motion by interpolating between two buffered snapshots from the recent past. The client never runs the game simulation itself for remote entities — it simply renders a time-delayed, smoothly blended view of what the server computed.

### The Problem It Solves

In a client-server multiplayer game, the server is the single source of truth. Clients need to display a smooth, visually coherent game world despite:

- **Discrete updates**: The server sends state at a fixed tick rate (e.g., 20-60 times per second), not continuously.
- **Network jitter**: Packets arrive at irregular intervals, not perfectly spaced.
- **Packet loss**: Some updates never arrive at all.
- **Latency**: Updates are always stale by the time they arrive.

Snapshot interpolation addresses all of these by buffering incoming snapshots and rendering the world at a point in time *between two known states*, guaranteeing that displayed positions are always based on real server-computed data rather than client-side guesses.

### Key Properties

| Property | Description |
|---|---|
| **Server authority** | Server computes all game logic; clients are "privileged spectators" |
| **Visual fidelity** | Objects only travel through states they actually occupied on the server |
| **Temporal coherence** | All objects in a single snapshot share the same server timestamp, so interactions appear synchronized |
| **No simulation on client** | Remote entities require zero game logic execution on the client (only interpolation math) |
| **Inherent latency** | The client always renders the past, introducing visual delay equal to the interpolation buffer depth |

---

## History & Origins

### Quake (1996) and QuakeWorld (1996-1997)

The roots of snapshot interpolation trace back to id Software. The original Quake (1996) used a naive client-server model where the client waited for server updates before rendering, resulting in poor responsiveness. QuakeWorld introduced client-side prediction for the local player, but remote entities still needed smooth rendering from discrete server updates.

### Quake III Arena (1999) — The Canonical Implementation

Quake III Arena, designed by John Carmack and the id Software team, established the snapshot interpolation model that became the industry standard for over two decades. Its key innovations:

- **Full world snapshots**: The server captures the entire game state (all entity positions, orientations, animations, events) into a single snapshot after each simulation tick.
- **Delta compression against acknowledged baselines**: Rather than sending full snapshots every frame, the server compresses each snapshot as a delta against the last snapshot the client *acknowledged receiving*. This is not necessarily the previous snapshot — it may be several frames old, depending on RTT.
- **Per-field delta encoding**: The engine uses C struct metadata to compare individual fields (position X, position Y, health, etc.) and only transmits changed fields, prefixed by single-bit change flags.
- **Huffman compression**: An additional layer of entropy coding applied to the entire packet payload using a pre-computed frequency table.
- **Unreliable delivery**: Snapshots are sent via UDP with no retransmission. If a packet is lost, the server simply sends the next snapshot (delta-compressed against the last *acknowledged* baseline, not the lost packet), and the client interpolates or extrapolates to cover the gap.

The Quake III server maintained a cycling array of 32 snapshots per client, allowing it to delta-compress against baselines up to ~1.6 seconds in the past (at 20 snapshots/second). This required approximately 8 MB of memory for four players.

### Half-Life / Source Engine (2004)

Valve's Source Engine refined the model with configurable tick rates (`sv_tickrate`), client-side interpolation controls (`cl_interp`, `cl_interp_ratio`), and server-side lag compensation (rewinding entity positions for hit detection). The Source Engine's approach became the standard for competitive FPS games.

### Overwatch (2016)

Blizzard's Overwatch, presented at GDC 2017 by Timothy Ford, demonstrated a modern take on snapshot interpolation built atop an Entity Component System (ECS) architecture. Key characteristics:

- **Fixed tick rate** (originally 20.8 Hz, later increased to 62.5 Hz for high-priority updates) with deterministic `UpdateFixed` command frames.
- **Client clock runs ahead** of the server by `RTT/2 + one command frame`, enabling responsive client-side prediction for the local player while remote entities are interpolated from snapshots.
- **ECS-driven delta compression**: The component-based architecture allows efficient per-component delta encoding.

### Gaffer on Games (2014-2015)

Glenn Fiedler's landmark article series "Networked Physics" provided the most detailed public analysis of snapshot interpolation, including bandwidth calculations, interpolation buffer design, compression techniques, and working reference implementations. These articles remain the primary educational resource on the topic.

---

## How It Works Step by Step

### The Core Loop

```
┌─────────────────────────────────────────────────────────────────┐
│                         SERVER                                  │
│                                                                 │
│  1. Receive client inputs                                       │
│  2. Advance simulation by one tick (apply physics, game logic)  │
│  3. Capture snapshot of entire world state                      │
│  4. For each client:                                            │
│     a. Delta-compress snapshot against client's last acked      │
│     b. Serialize and send via UDP                               │
│  5. Repeat at fixed tick rate                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │ UDP packets
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT                                  │
│                                                                 │
│  1. Receive snapshot packet                                     │
│  2. Decompress / reconstruct full snapshot                      │
│  3. Insert into interpolation buffer (sorted by server time)    │
│  4. Send acknowledgment (piggybacked on next input packet)      │
│  5. Each render frame:                                          │
│     a. Compute render_time = current_server_time - buffer_delay │
│     b. Find two snapshots bracketing render_time                │
│     c. Compute interpolation factor: t = (render_time - t0)    │
│        / (t1 - t0)                                              │
│     d. For each entity: lerp(state_0, state_1, t)              │
│     e. Render the interpolated state                            │
└─────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Breakdown

#### Step 1: Server Captures Snapshot

After processing all client inputs and advancing the simulation by one tick, the server serializes the state of every relevant entity into a snapshot structure. A snapshot includes a **server timestamp** (or tick number) so clients can place it on the timeline.

#### Step 2: Delta Compression and Transmission

The server maintains, per client, a record of which snapshot that client last acknowledged. It compresses the current snapshot as a delta against that baseline. If no acknowledgment has been received yet (initial connection), the snapshot is compressed against a zeroed-out "dummy" baseline.

#### Step 3: Client Receives and Buffers

When a snapshot packet arrives, the client:
1. Checks the **sequence number** — if it is older than the most recent received snapshot, it is discarded.
2. Reconstructs the full snapshot by applying the delta to the stored baseline.
3. Inserts the snapshot into the **interpolation buffer**, ordered by server timestamp.
4. Sends an acknowledgment to the server (typically piggybacked on the next input packet).

#### Step 4: Client Renders by Interpolating

The client does **not** render the latest snapshot. Instead, it computes a `render_time` that is deliberately in the past:

```
render_time = estimated_server_time - interpolation_delay
```

It then finds the two snapshots in the buffer that bracket `render_time` — one before (`snapshot_a` at time `t_a`) and one after (`snapshot_b` at time `t_b`). The interpolation factor is:

```
alpha = (render_time - t_a) / (t_b - t_a)
```

Each entity's rendered state is computed as:

```
rendered_position = lerp(snapshot_a.position, snapshot_b.position, alpha)
rendered_rotation = slerp(snapshot_a.rotation, snapshot_b.rotation, alpha)
```

This produces smooth, continuous motion even though the server only sends updates at a discrete tick rate.

---

## Snapshot Design

### What Goes in a Snapshot

A snapshot is a serialized representation of the entire game world at a single point in time. The contents vary by game, but a typical snapshot includes:

| Category | Examples |
|---|---|
| **Entity transforms** | Position (vec3), rotation (quaternion or Euler angles), scale |
| **Entity velocities** | Linear velocity, angular velocity (optional; useful for extrapolation or Hermite interpolation) |
| **Player state** | Health, armor, ammo, current weapon, animation state |
| **Game state** | Score, timer, round phase, flags |
| **Events** | Sounds to play, effects to spawn, hit markers (sometimes sent as separate reliable messages) |
| **Visibility / relevance** | Per-client filtering — only entities the client can perceive |

#### Quake III's Entity State Structure

Quake III defined its entity state as a C struct with a companion metadata table describing each field's offset and bit width:

```c
typedef struct {
    int     number;         // entity index
    int     eType;          // entity type
    int     eFlags;         // entity flags
    trajectory_t pos;       // position trajectory (type, time, duration, base, delta)
    trajectory_t apos;      // angular position trajectory
    int     time;
    int     time2;
    vec3_t  origin;
    vec3_t  origin2;
    vec3_t  angles;
    vec3_t  angles2;
    int     otherEntityNum;
    int     groundEntityNum;
    int     constantLight;
    int     loopSound;
    int     modelindex;
    int     frame;
    int     solid;
    int     event;
    int     eventParm;
    int     powerups;
    int     weapon;
    int     legsAnim;
    int     torsoAnim;
    int     generic1;
    // ... additional fields
} entityState_t;
```

The companion metadata table enabled generic delta comparison without type-specific code:

```c
typedef struct {
    char    *name;
    int     offset;
    int     bits;       // 0 = float, >0 = integer with N bits
} netField_t;

netField_t entityStateFields[] = {
    { NETF(pos.trTime),       32 },
    { NETF(pos.trBase[0]),     0 },  // 0 means float
    { NETF(pos.trBase[1]),     0 },
    { NETF(pos.trBase[2]),     0 },
    { NETF(pos.trDelta[0]),    0 },
    { NETF(pos.trDelta[1]),    0 },
    { NETF(pos.trDelta[2]),    0 },
    // ... ordered by change frequency for optimization
};
```

Fields were ordered by change frequency so that the "last changed field index" optimization (transmitting only up to the highest-index changed field) saved the most bits.

### Serialization Strategies

#### Bit Packing

Rather than writing data byte-by-byte, snapshot serialization operates at the bit level. This allows tight encoding:

- A boolean value: 1 bit
- Player health (0-100): 7 bits instead of 32
- Entity index (0-1023): 10 bits
- Animation index (0-255): 8 bits

#### Quantization

Floating-point values are converted to fixed-point integers within known bounds:

```
quantized = round((value - min) / (max - min) * (2^bits - 1))
```

Example from Gaffer on Games:
- Position bounded to [-256, 255] meters horizontally, [0, 32] meters vertically
- 512 values per meter (~2mm resolution)
- X, Y: 18 bits each; Z: 14 bits
- Total position: 50 bits (vs. 96 bits for three 32-bit floats)

#### Smallest-Three Quaternion Compression

Since unit quaternions satisfy `x² + y² + z² + w² = 1`, one component can be reconstructed from the other three. The "smallest three" method:

1. Identify the component with the largest absolute value (2 bits for the index).
2. Transmit the three smallest components (9 bits each, since they are bounded to ±0.707107).
3. Reconstruct the largest component: `w = sqrt(1 - x² - y² - z²)`.

**Result**: 29 bits per quaternion (vs. 128 bits for four 32-bit floats).

### Full vs. Delta Snapshots

| Aspect | Full Snapshot | Delta Snapshot |
|---|---|---|
| **Content** | Complete world state | Only fields that changed since a baseline |
| **Size** | Large (kilobytes to tens of KB) | Small (typically 10-20% of full) |
| **When used** | Initial connection, after long packet loss | Steady-state gameplay |
| **Baseline dependency** | None | Requires both sides to agree on baseline |
| **Failure mode** | None (self-contained) | If baseline is wrong, entire state corrupts |

---

## Delta Compression

Delta compression is the single most important bandwidth optimization for snapshot interpolation. Without it, the technique is impractical for all but the simplest games.

### How Delta Compression Works

The core idea: instead of sending the current snapshot `S_n`, send only the difference between `S_n` and a previous snapshot `S_base` that the client is known to have received.

```
delta = S_current - S_baseline
```

The baseline is selected based on acknowledgment: the client continuously reports the sequence number of the most recent snapshot it received, and the server uses that acknowledged snapshot as the delta baseline.

**Critical insight**: The baseline is *not* necessarily the previous snapshot. If RTT is 100ms and the server sends at 60 Hz (~16.6ms intervals), the baseline might be 6-7 snapshots old. This is why the server must buffer recent snapshots per client.

### Quake III's Approach: Field-Level Delta Encoding

Quake III's delta compression operates at the individual field level within each entity:

1. **Entity-level comparison**: Compare the entity list in the current snapshot against the baseline.
   - **Unchanged entities**: Skip entirely (zero bits).
   - **New entities**: Send full field data.
   - **Removed entities**: Send a deletion marker.
   - **Changed entities**: Proceed to field-level comparison.

2. **Field-level comparison**: For each field in a changed entity:
   - Send a 1-bit flag: `0` = unchanged, `1` = changed.
   - If changed, send the new value in its entirety (at the field's defined bit width).
   - Optimization: only transmit up to the index of the last changed field (8-bit "field count" header).

3. **Float field encoding** (Quake III specific):
   - 1 bit: zero or non-zero
   - If non-zero: 1 bit selecting compressed (13-bit integer) or full (32-bit IEEE float)
   - Many game values (positions near origin, small angles) fit in 13 bits

4. **Integer field encoding**:
   - 1 bit: zero or non-zero
   - If non-zero: value at the field's defined bit width

**Example**: If only `position.y` changed in an entity with 20 fields:

```
Field 0 (pos.x):  [0]              — 1 bit (unchanged)
Field 1 (pos.y):  [1][32-bit value] — 33 bits (changed)
[field count = 1]                    — stop here, remaining fields implied unchanged
Total: 34 bits + overhead (vs. hundreds of bits for full entity)
```

### Gaffer on Games Approach: Multi-Level Delta Encoding

Glenn Fiedler's snapshot compression article describes a more sophisticated delta encoding for a physics simulation with 900 cubes:

#### Changed-Entity Identification

Two strategies, selected per-packet based on efficiency:
- **Index mode**: Send a 10-bit cube index per changed cube (efficient when few cubes change).
- **Bitmask mode**: Send one bit per cube (efficient when many cubes change).

#### Relative Index Encoding

When using index mode, indices are encoded relative to the previous index, exploiting spatial clustering:
- Offset [1, 8]: 4 bits
- Offset [9, 40]: 7 bits
- Offset [41, 900]: 12 bits
- Average: ~5.5 bits per index

#### Position Delta Encoding

Positions are encoded as deltas from the baseline position:
- **Small delta** (±16 quantized units): 5 bits per component
- **Large delta** (±256 quantized units): 9 bits per component
- **Fallback**: 50 bits absolute (if delta exceeds large range)
- Average result: 26.1 bits per changed position

#### Quaternion Delta Encoding

- 90% of orientations maintain the same largest-component index across 100ms intervals.
- Per-component delta encoding with multi-level ranges (5 or 8 bits per component).
- Average: 23.3 bits (80.3% of the absolute "smallest three" encoding).
- Zero additional precision loss vs. baseline reconstruction.

#### Unchanged Component Optimization

- ~5% of positions and ~5% of orientations remain unchanged after quantization.
- 2-bit flags (position changed / orientation changed) save ~2 bits per cube overall.

#### Results

| Stage | Per-Cube Size | Total Bandwidth (60 Hz, 900 cubes) |
|---|---|---|
| Uncompressed | 40.1 bytes | 17.38 Mbit/s |
| Quantized + smallest-three | ~14 bytes | ~6 Mbit/s |
| + Delta compression | ~10 bytes (steady state) | ~256 Kbit/s |

Delta compression alone provided approximately **90% bandwidth reduction**.

### Overwatch's Approach

Overwatch uses ECS-driven delta compression where:

- Each ECS component type has its own serialization/delta logic.
- Clients acknowledge received snapshots, enabling per-client baseline tracking.
- The fixed tick rate (deterministic `UpdateFixed`) ensures consistent state transitions, which improves delta compression ratios since fewer fields change unpredictably.
- Additional entropy coding (Huffman or arithmetic) is applied as a final compression pass.

### Baseline Management

Proper baseline management is critical to delta compression correctness:

```
Server state per client:
  - last_acked_snapshot: sequence number of most recent snapshot the client confirmed
  - snapshot_buffer[32]: ring buffer of recent snapshots (Quake III used 32 slots)
  - If last_acked_snapshot is too old (fallen out of buffer), send a full snapshot

Client state:
  - Piggyback ack of most recent received snapshot sequence in every input packet
  - Must store the baseline snapshot to apply incoming deltas against
```

**Initial connection**: Before the client has acknowledged any snapshot, the server delta-encodes against a zeroed-out "dummy" baseline (equivalent to sending a full snapshot, but using the same code path).

**Packet loss recovery**: If several consecutive packets are lost and the acknowledged baseline becomes too stale, the server may need to fall back to a full (non-delta) snapshot or a very large delta, temporarily spiking bandwidth.

---

## The Interpolation Buffer

The interpolation buffer (also called a "jitter buffer," borrowed from VoIP terminology) is the mechanism that enables smooth rendering despite irregular packet arrival.

### Why a Buffer Is Necessary

Without buffering, the client would need to display each snapshot the instant it arrives. But packets do not arrive at uniform intervals:

```
Server sends at t=0, 50, 100, 150, 200 ms  (20 Hz)

Client receives at t=62, 108, 170, 215, 278 ms  (jitter + latency)
         Gaps:       62   46   62   45   63  ms  (irregular!)
```

If the client renders immediately on receipt, motion would stutter, speed up, and slow down with every jitter spike.

### How the Buffer Works

The buffer introduces a deliberate delay so that the client always has at least two snapshots available to interpolate between:

```
Time ──────────────────────────────────────────────────►

Snapshots received: ──S1──S2──S3──S4──S5──S6──────────
                          ▲
Buffer delay: ────────────┤◄── interpolation_delay ──►│
                          │                            │
                    Rendering S1↔S2              Rendering S3↔S4
```

The client computes:

```
render_time = estimated_current_server_time - interpolation_delay
```

It then finds the two snapshots bracketing `render_time` and interpolates between them.

### Buffer Sizing: The Core Trade-off

The buffer must be large enough to absorb jitter and packet loss, but small enough to minimize visual latency.

**Glenn Fiedler's rule of thumb**: The interpolation delay should be large enough to survive **two consecutive lost packets** and still have a snapshot to interpolate toward:

```
interpolation_delay = 3 × packet_interval + jitter_margin
```

Where:
- `packet_interval = 1000 / send_rate` (ms)
- `jitter_margin` accounts for variance in delivery timing (~50ms typical)
- The factor of 3 provides tolerance for two lost packets (you need the *third* packet to arrive)

### Buffer Sizing Math by Send Rate

| Send Rate | Packet Interval | 3× Interval | + Jitter (50ms) | Total Visual Delay |
|---|---|---|---|---|
| 10 pps | 100 ms | 300 ms | 350 ms | ~350 ms |
| 20 pps | 50 ms | 150 ms | 200 ms | ~200 ms |
| 30 pps | 33 ms | 100 ms | 150 ms | ~150 ms |
| 60 pps | 16.7 ms | 50 ms | 100 ms | ~85-100 ms |

At 10 packets per second, the client renders the world **350ms behind real-time** — clearly visible in fast-paced gameplay. At 60 packets per second, the delay drops to **~85ms**, which is generally acceptable.

### Adaptive Buffering

Rather than using a fixed buffer size, production implementations often adapt to current network conditions:

```
measured_jitter = exponential_moving_average(|arrival_interval - expected_interval|)
adaptive_delay  = base_delay + k × measured_jitter
```

Where `k` is a tuning parameter (typically 2-3 standard deviations of jitter).

**Photon Fusion** implements adaptive interpolation that "adapts the offset and buffering required based on current network conditions," giving clients "as little latency as possible while still providing smooth visuals."

### Buffer Underrun and Overrun

**Underrun** (buffer empty — no future snapshot to interpolate toward):
- Can happen during severe packet loss or sudden latency spikes.
- Options: freeze the last known state, or extrapolate (risky — see [Handling Edge Cases](#handling-edge-cases)).
- After recovery, gradually speed up playback to refill the buffer rather than snapping forward.

**Overrun** (buffer too full — latency growing):
- Can happen when network conditions improve but the buffer was sized for worse conditions.
- Options: slightly accelerate playback speed (e.g., 1.05x) to drain excess, or drop the oldest buffered snapshot.

### Interpolation Modes

#### Linear Interpolation (Lerp)

The simplest and most common approach:

```
position = snapshot_a.position + alpha * (snapshot_b.position - snapshot_a.position)
rotation = slerp(snapshot_a.rotation, snapshot_b.rotation, alpha)
```

Produces smooth motion for most cases but can exhibit subtle artifacts:
- **Position jitter**: Slight pulsing when objects interact (e.g., stacked physics objects).
- **First-order discontinuity**: Velocity changes abruptly at snapshot boundaries.

#### Hermite Spline Interpolation

Uses velocity data from snapshots to match both position and velocity at sample points, eliminating first-order discontinuities:

```
// Hermite basis functions
h00(t) = 2t³ - 3t² + 1
h10(t) = t³ - 2t² + t
h01(t) = -2t³ + 3t²
h11(t) = t³ - t²

position = h00(t) * p0 + h10(t) * dt * v0 + h01(t) * p1 + h11(t) * dt * v1
```

Where `p0`, `v0` are position/velocity at snapshot A, `p1`, `v1` at snapshot B, and `dt` is the time between snapshots.

**Trade-off**: Requires transmitting velocity per entity (extra bandwidth) but produces noticeably smoother results at lower send rates. Glenn Fiedler notes this "eliminates first-order discontinuities" and improves quality without increasing the send rate.

---

## Latency-Bandwidth Trade-off

Snapshot interpolation presents a fundamental and unavoidable trade-off between **visual latency** and **bandwidth consumption**. Understanding this trade-off is essential for choosing appropriate parameters.

### The Mathematics

**Bandwidth** scales linearly with send rate:

```
bandwidth = snapshot_size × send_rate
```

**Visual delay** scales inversely with send rate (approximately):

```
visual_delay ≈ 3 / send_rate + jitter_margin + network_latency
```

### Concrete Examples

Assume a game with 50 entities, each requiring ~60 bytes after quantization and delta compression:

| Send Rate | Snapshot Size (delta) | Bandwidth | Interpolation Delay | Total Visual Delay (+ 40ms latency) |
|---|---|---|---|---|
| 10 Hz | ~3 KB | 30 KB/s (240 Kbit/s) | 350 ms | ~390 ms |
| 20 Hz | ~2 KB | 40 KB/s (320 Kbit/s) | 200 ms | ~240 ms |
| 30 Hz | ~1.5 KB | 45 KB/s (360 Kbit/s) | 150 ms | ~190 ms |
| 60 Hz | ~1 KB | 60 KB/s (480 Kbit/s) | 85 ms | ~125 ms |
| 128 Hz | ~0.7 KB | 90 KB/s (720 Kbit/s) | 50 ms | ~90 ms |

Note: Delta snapshot sizes *decrease* at higher send rates because less changes between consecutive ticks.

### Why Higher Tick Rates Help

Higher send rates reduce interpolation delay, but they also improve delta compression because less state changes between adjacent snapshots. The relationship is sublinear — doubling the send rate does not double bandwidth because deltas shrink.

### MTU Constraints

A critical practical limit: **snapshots should fit in a single UDP packet** (ideally under 1200-1400 bytes to avoid IP fragmentation). If a snapshot must be fragmented across multiple UDP packets, *all fragments must arrive* for the snapshot to be usable. The probability of losing at least one fragment grows exponentially with fragment count:

```
P(loss) = 1 - (1 - p)^n

Where p = per-packet loss rate, n = number of fragments

Example: 2% loss rate, 3 fragments:
P(loss) = 1 - (0.98)^3 = 5.88%  (vs. 2% for single packet)
```

This makes MTU-awareness essential for snapshot design. Gaffer on Games pre-fragments messages into 1400-byte chunks; Quake III used the same approach.

---

## Clock Synchronization

For snapshot interpolation to work, the client must know the current server time so it can compute `render_time`. Since clients and servers have independent clocks that drift and differ in absolute value, clock synchronization is required.

### The NTP-Like Approach

Most game implementations use a simplified version of the Network Time Protocol:

1. **Client sends a ping** with its local timestamp `t1`.
2. **Server receives** the ping at server time `t2`, immediately responds with `t2` included.
3. **Client receives** the response at local time `t3`.
4. **RTT estimation**: `rtt = t3 - t1`
5. **One-way latency estimate**: `owd = rtt / 2` (assumes symmetric path).
6. **Server time estimate**: `server_time = t2 + owd = t2 + (t3 - t1) / 2`
7. **Clock offset**: `offset = server_time - t3`

```
Client:  ──t1──────────────────────────────t3──
              \                           /
               \    network (owd)        /
                \                       /
Server:  ────────t2──────────────────────────
```

### Smoothed Estimation

A single measurement is noisy. Production implementations take multiple samples and apply filtering:

```
// Exponentially weighted moving average
estimated_offset = estimated_offset * (1 - alpha) + measured_offset * alpha

// Or use median filtering over a window of recent samples
// to reject outliers from route changes or congestion spikes
```

Glenn Fiedler recommends maintaining an RTT estimate using "an exponentially smoothed moving average" updated whenever packets are acknowledged.

### Clock Drift

Even after synchronization, client and server clocks drift apart at approximately **1 millisecond per 10 seconds** of real time. To compensate:

- Re-synchronize periodically (every 5-10 seconds is typical for game protocols).
- Apply corrections gradually (adjust the clock speed slightly rather than jumping) to avoid visual pops.

### Practical Implementation

In many implementations, the server simply includes its current time (or tick number) in every snapshot packet, and the client:

1. Records the local time when each snapshot arrives.
2. Maintains a running estimate of the offset between local time and server time.
3. Uses this offset to compute `estimated_server_time = local_time + offset`.
4. Computes `render_time = estimated_server_time - interpolation_delay`.

Since snapshots arrive continuously during gameplay, clock sync "comes for free" — each snapshot is a sync sample.

### Tick-Based vs. Wall-Clock Time

Some engines (Quake III, Overwatch) work in **tick numbers** rather than wall-clock milliseconds. The server sends the current tick number, and the client converts to time:

```
server_time = tick_number × tick_interval
```

This avoids floating-point drift and ensures both sides agree on the time grid.

---

## Handling Edge Cases

### Teleportation

When an entity teleports (instantly moves a large distance), linear interpolation would show the entity sliding across the map over the interpolation interval. Solutions:

- **Flag the teleport** in the snapshot (a single bit per entity). The client skips interpolation for flagged entities and snaps them to the new position.
- **Threshold detection**: If the position delta exceeds a maximum plausible velocity, treat it as a teleport.
- **Entity recreation**: Destroy the old entity and create a new one at the destination (some engines use this approach for portals).

### Entity Creation and Destruction

**Creation**: A new entity appears in snapshot `S_n` that was not in `S_n-1`. The client cannot interpolate from a previous state because none exists.
- Snap to the first known position immediately.
- Optionally play a spawn effect to mask the sudden appearance.

**Destruction**: An entity present in `S_n-1` is absent in `S_n`. The client is currently interpolating toward a state that no longer exists.
- If the client is interpolating between `S_n-1` and `S_n`, it can finish the interpolation to the last known position, then remove the entity.
- Optionally play a death/despawn effect.
- Keep the entity in a "pending removal" state until the interpolation buffer passes the removal timestamp.

### Buffer Underrun (Starvation)

When the client's `render_time` advances past the newest buffered snapshot, it has nothing to interpolate toward. Strategies:

1. **Freeze**: Hold the last known state. Simple, but entities visibly stop moving.
2. **Extrapolate**: Project forward using the last known velocity:
   ```
   extrapolated_pos = last_pos + last_velocity * time_since_last_snapshot
   ```
   Glenn Fiedler explicitly warns against extrapolation for physics simulations: "you simply can't accurately match a physics simulation with an approximation" — objects clip through walls, miss collisions, and diverge from the real state. However, simple linear extrapolation can be acceptable for brief gaps (< 100ms) in simpler games.
3. **Time dilation**: Slow down the client's effective playback rate to stretch the available snapshots, then gradually speed back up when new data arrives.

### Buffer Overrun

When the buffer accumulates too many snapshots (network conditions improved, or initial estimate was too conservative):

- **Time acceleration**: Speed up playback (e.g., 1.05-1.1x) to consume excess snapshots.
- **Drop oldest**: Discard stale snapshots that are too far in the past to be useful.
- **Adaptive reduction**: Lower the interpolation delay parameter gradually.

### Packet Reordering

UDP does not guarantee packet ordering. A snapshot with sequence number 105 might arrive before 104. The interpolation buffer naturally handles this — snapshots are inserted in order by server timestamp, regardless of arrival order. The sequence number is used only to discard duplicates and identify the newest snapshot for acknowledgment purposes.

### Variable Tick Rate

If the server's tick rate varies (e.g., due to load), the interpolation interval between consecutive snapshots changes. The interpolation math handles this naturally since it uses actual timestamps:

```
alpha = (render_time - t_a) / (t_b - t_a)  // works regardless of interval
```

---

## Pseudo-Code / Code Examples

### Pseudo-Code: Core Interpolation Loop

```
// === SERVER SIDE ===

function serverTick():
    processClientInputs()
    advanceSimulation(TICK_INTERVAL)

    snapshot = captureWorldState()
    snapshot.tick = currentTick
    snapshot.serverTime = currentTick * TICK_INTERVAL

    for each client in connectedClients:
        baseline = snapshotBuffer[client.lastAckedSequence]
        delta = computeDelta(snapshot, baseline)
        packet = serialize(delta)
        packet.sequence = nextSequence++
        send(client, packet)

        // Store for future delta compression
        snapshotBuffer.store(packet.sequence, snapshot)

// === CLIENT SIDE ===

class InterpolationBuffer:
    snapshots: SortedList<Snapshot>  // sorted by serverTime
    interpolationDelay: float       // milliseconds

    function insertSnapshot(snapshot):
        if snapshot.sequence <= mostRecentSequence:
            return  // discard old/duplicate
        mostRecentSequence = snapshot.sequence
        snapshots.insertSorted(snapshot, by: snapshot.serverTime)
        sendAck(snapshot.sequence)

    function sample(estimatedServerTime):
        renderTime = estimatedServerTime - interpolationDelay

        // Find bracketing snapshots
        snapshotA = null
        snapshotB = null
        for i in range(snapshots.length - 1):
            if snapshots[i].serverTime <= renderTime <= snapshots[i+1].serverTime:
                snapshotA = snapshots[i]
                snapshotB = snapshots[i+1]
                break

        if snapshotA == null or snapshotB == null:
            return lastRenderedState  // buffer underrun: freeze

        // Compute interpolation factor
        alpha = (renderTime - snapshotA.serverTime)
              / (snapshotB.serverTime - snapshotA.serverTime)
        alpha = clamp(alpha, 0, 1)

        // Interpolate each entity
        result = {}
        for entityId in union(snapshotA.entities, snapshotB.entities):
            if entityId not in snapshotA:
                result[entityId] = snapshotB.entities[entityId]  // just spawned
            else if entityId not in snapshotB:
                result[entityId] = snapshotA.entities[entityId]  // about to despawn
            else:
                stateA = snapshotA.entities[entityId]
                stateB = snapshotB.entities[entityId]

                if stateB.teleported:
                    result[entityId] = stateB
                else:
                    result[entityId] = interpolateEntity(stateA, stateB, alpha)

        // Prune old snapshots no longer needed
        pruneOlderThan(renderTime - TICK_INTERVAL)

        return result

function interpolateEntity(a, b, alpha):
    return {
        position: lerp(a.position, b.position, alpha),
        rotation: slerp(a.rotation, b.rotation, alpha),
        health: (alpha < 1.0) ? a.health : b.health,  // don't lerp discrete values
        animationFrame: lerpAnimationFrame(a, b, alpha),
    }
```

### TypeScript Implementation

Below is a practical, annotated TypeScript implementation of a client-side snapshot interpolation system:

```typescript
// ============================================================
// snapshot-interpolation.ts
// A complete client-side snapshot interpolation implementation
// ============================================================

/** Represents a 3D vector */
interface Vec3 {
  x: number;
  y: number;
  z: number;
}

/** Represents a quaternion rotation */
interface Quat {
  x: number;
  y: number;
  z: number;
  w: number;
}

/** State of a single networked entity within a snapshot */
interface EntityState {
  id: number;
  position: Vec3;
  rotation: Quat;
  velocity: Vec3;
  health: number;
  animationId: number;
  animationTime: number;
  teleported: boolean;
}

/** A complete world snapshot received from the server */
interface Snapshot {
  sequence: number;       // Monotonically increasing packet sequence
  serverTime: number;     // Server timestamp in milliseconds
  entities: Map<number, EntityState>;
}

/** Interpolated state ready for rendering */
interface InterpolatedState {
  entities: Map<number, EntityState>;
  renderTime: number;
  bufferHealth: number;   // 0 = empty, 1 = healthy
}

// ──────────────────────────────────────────────
// Vector / Quaternion math utilities
// ──────────────────────────────────────────────

function lerpVec3(a: Vec3, b: Vec3, t: number): Vec3 {
  return {
    x: a.x + (b.x - a.x) * t,
    y: a.y + (b.y - a.y) * t,
    z: a.z + (b.z - a.z) * t,
  };
}

function slerpQuat(a: Quat, b: Quat, t: number): Quat {
  // Compute cosine of angle between quaternions
  let dot = a.x * b.x + a.y * b.y + a.z * b.z + a.w * b.w;

  // If negative dot, negate one quaternion to take shorter arc
  const bSign = dot < 0 ? -1 : 1;
  dot = Math.abs(dot);

  let s0: number, s1: number;
  if (dot > 0.9995) {
    // Quaternions are very close — use linear interpolation to avoid division by zero
    s0 = 1 - t;
    s1 = t * bSign;
  } else {
    const theta = Math.acos(dot);
    const sinTheta = Math.sin(theta);
    s0 = Math.sin((1 - t) * theta) / sinTheta;
    s1 = (Math.sin(t * theta) / sinTheta) * bSign;
  }

  return {
    x: s0 * a.x + s1 * b.x,
    y: s0 * a.y + s1 * b.y,
    z: s0 * a.z + s1 * b.z,
    w: s0 * a.w + s1 * b.w,
  };
}

function vec3DistanceSq(a: Vec3, b: Vec3): number {
  const dx = a.x - b.x;
  const dy = a.y - b.y;
  const dz = a.z - b.z;
  return dx * dx + dy * dy + dz * dz;
}

// ──────────────────────────────────────────────
// Clock Synchronization
// ──────────────────────────────────────────────

class ServerClockEstimator {
  private offset: number = 0;          // estimated offset: serverTime = localTime + offset
  private rtt: number = 100;           // smoothed round-trip time estimate
  private initialized: boolean = false;
  private readonly alpha: number = 0.1; // EMA smoothing factor

  /**
   * Process a time sync sample.
   * @param localSendTime    Local time when ping was sent
   * @param serverTime       Server timestamp from the pong
   * @param localReceiveTime Local time when pong was received
   */
  processSyncSample(localSendTime: number, serverTime: number, localReceiveTime: number): void {
    const measuredRtt = localReceiveTime - localSendTime;
    const measuredOffset = serverTime - (localSendTime + measuredRtt / 2);

    if (!this.initialized) {
      this.rtt = measuredRtt;
      this.offset = measuredOffset;
      this.initialized = true;
    } else {
      // Exponentially weighted moving average
      this.rtt = this.rtt * (1 - this.alpha) + measuredRtt * this.alpha;
      this.offset = this.offset * (1 - this.alpha) + measuredOffset * this.alpha;
    }
  }

  /**
   * Convenience: process a snapshot arrival as an implicit time sync sample.
   * The snapshot's serverTime acts as the "server timestamp."
   * Assumes the snapshot was generated RTT/2 ago.
   */
  processSnapshotArrival(snapshotServerTime: number, localArrivalTime: number): void {
    // We don't have a send time, so we estimate using current RTT
    // server_time ≈ local_arrival_time + offset
    // measured_offset = snapshotServerTime - localArrivalTime
    // But this ignores one-way delay, so we add rtt/2 correction
    const measuredOffset = snapshotServerTime - localArrivalTime + this.rtt / 2;

    if (!this.initialized) {
      this.offset = measuredOffset;
      this.initialized = true;
    } else {
      // Use smaller alpha for snapshot-based sync (noisier)
      const snapshotAlpha = 0.05;
      this.offset = this.offset * (1 - snapshotAlpha) + measuredOffset * snapshotAlpha;
    }
  }

  /** Estimate the current server time given the current local time */
  estimateServerTime(localTime: number): number {
    return localTime + this.offset;
  }

  /** Current RTT estimate in milliseconds */
  getRtt(): number {
    return this.rtt;
  }
}

// ──────────────────────────────────────────────
// Interpolation Buffer
// ──────────────────────────────────────────────

interface InterpolationBufferConfig {
  /** Base interpolation delay in milliseconds (before jitter adaptation) */
  baseDelay: number;
  /** Maximum number of snapshots to buffer */
  maxBufferSize: number;
  /** Teleport distance threshold (squared) — skip interpolation if exceeded */
  teleportDistanceSq: number;
  /** Enable adaptive delay based on measured jitter */
  adaptiveDelay: boolean;
  /** Multiplier for jitter when computing adaptive delay */
  jitterMultiplier: number;
}

const DEFAULT_CONFIG: InterpolationBufferConfig = {
  baseDelay: 100,
  maxBufferSize: 30,
  teleportDistanceSq: 100 * 100,  // 100 units
  adaptiveDelay: true,
  jitterMultiplier: 3,
};

class SnapshotInterpolationBuffer {
  private buffer: Snapshot[] = [];
  private config: InterpolationBufferConfig;
  private clock: ServerClockEstimator;

  // Jitter measurement
  private lastArrivalTime: number = -1;
  private expectedInterval: number = 50;  // Will be updated based on observed rate
  private jitter: number = 0;             // Smoothed jitter estimate (ms)

  // Sequence tracking
  private highestSequence: number = -1;

  // Stats
  private underrunCount: number = 0;
  private overrunCount: number = 0;

  constructor(
    clock: ServerClockEstimator,
    config: Partial<InterpolationBufferConfig> = {},
  ) {
    this.config = { ...DEFAULT_CONFIG, ...config };
    this.clock = clock;
  }

  /** Get the effective interpolation delay (base + adaptive jitter) */
  get interpolationDelay(): number {
    if (this.config.adaptiveDelay) {
      return this.config.baseDelay + this.jitter * this.config.jitterMultiplier;
    }
    return this.config.baseDelay;
  }

  /** Get diagnostic info */
  get stats() {
    return {
      bufferSize: this.buffer.length,
      interpolationDelay: this.interpolationDelay,
      jitter: this.jitter,
      rtt: this.clock.getRtt(),
      underrunCount: this.underrunCount,
      overrunCount: this.overrunCount,
    };
  }

  /**
   * Insert a received snapshot into the buffer.
   * Call this when a snapshot packet is received from the server.
   */
  receiveSnapshot(snapshot: Snapshot, localArrivalTime: number): void {
    // Discard out-of-order / duplicate snapshots
    if (snapshot.sequence <= this.highestSequence) {
      return;
    }
    this.highestSequence = snapshot.sequence;

    // Update clock estimate
    this.clock.processSnapshotArrival(snapshot.serverTime, localArrivalTime);

    // Measure jitter
    if (this.lastArrivalTime >= 0) {
      const arrivalInterval = localArrivalTime - this.lastArrivalTime;
      const deviation = Math.abs(arrivalInterval - this.expectedInterval);
      this.jitter = this.jitter * 0.9 + deviation * 0.1;  // EMA

      // Adapt expected interval based on observed snapshot spacing
      this.expectedInterval = this.expectedInterval * 0.95 + arrivalInterval * 0.05;
    }
    this.lastArrivalTime = localArrivalTime;

    // Insert into buffer, maintaining sort order by serverTime
    let insertIdx = this.buffer.length;
    for (let i = this.buffer.length - 1; i >= 0; i--) {
      if (this.buffer[i].serverTime <= snapshot.serverTime) {
        insertIdx = i + 1;
        break;
      }
      if (i === 0) insertIdx = 0;
    }
    this.buffer.splice(insertIdx, 0, snapshot);

    // Evict oldest if buffer exceeds max size
    while (this.buffer.length > this.config.maxBufferSize) {
      this.buffer.shift();
      this.overrunCount++;
    }
  }

  /**
   * Sample the interpolated world state for rendering.
   * Call this every render frame.
   *
   * @param localTime Current local time (e.g., performance.now())
   * @returns Interpolated state, or null if buffer is empty
   */
  sample(localTime: number): InterpolatedState | null {
    if (this.buffer.length < 2) {
      return null;  // Need at least 2 snapshots to interpolate
    }

    const estimatedServerTime = this.clock.estimateServerTime(localTime);
    const renderTime = estimatedServerTime - this.interpolationDelay;

    // Find the two snapshots bracketing renderTime
    let snapA: Snapshot | null = null;
    let snapB: Snapshot | null = null;

    for (let i = 0; i < this.buffer.length - 1; i++) {
      if (
        this.buffer[i].serverTime <= renderTime &&
        this.buffer[i + 1].serverTime >= renderTime
      ) {
        snapA = this.buffer[i];
        snapB = this.buffer[i + 1];
        break;
      }
    }

    // Handle edge cases
    if (!snapA || !snapB) {
      if (renderTime < this.buffer[0].serverTime) {
        // renderTime is before all buffered snapshots — we're too far ahead
        // This shouldn't normally happen; return the oldest snapshot
        this.underrunCount++;
        const oldest = this.buffer[0];
        return {
          entities: oldest.entities,
          renderTime,
          bufferHealth: 0,
        };
      } else {
        // renderTime is past all buffered snapshots — buffer underrun
        this.underrunCount++;
        const newest = this.buffer[this.buffer.length - 1];
        return {
          entities: newest.entities,
          renderTime,
          bufferHealth: 0,
        };
      }
    }

    // Compute interpolation factor
    const duration = snapB.serverTime - snapA.serverTime;
    const alpha = duration > 0 ? (renderTime - snapA.serverTime) / duration : 0;
    const clampedAlpha = Math.max(0, Math.min(1, alpha));

    // Interpolate all entities
    const interpolatedEntities = new Map<number, EntityState>();
    const allEntityIds = new Set<number>([
      ...snapA.entities.keys(),
      ...snapB.entities.keys(),
    ]);

    for (const entityId of allEntityIds) {
      const stateA = snapA.entities.get(entityId);
      const stateB = snapB.entities.get(entityId);

      if (!stateA && stateB) {
        // Entity just spawned — use its first known state
        interpolatedEntities.set(entityId, { ...stateB });
      } else if (stateA && !stateB) {
        // Entity about to be destroyed — hold last known state
        interpolatedEntities.set(entityId, { ...stateA });
      } else if (stateA && stateB) {
        // Entity exists in both snapshots — interpolate
        interpolatedEntities.set(
          entityId,
          this.interpolateEntity(stateA, stateB, clampedAlpha),
        );
      }
    }

    // Prune snapshots that are too old to ever be needed
    this.pruneOldSnapshots(renderTime);

    // Compute buffer health (1.0 = healthy, 0.0 = starving)
    const bufferedDuration =
      this.buffer[this.buffer.length - 1].serverTime - renderTime;
    const bufferHealth = Math.max(
      0,
      Math.min(1, bufferedDuration / this.interpolationDelay),
    );

    return {
      entities: interpolatedEntities,
      renderTime,
      bufferHealth,
    };
  }

  /**
   * Interpolate between two entity states.
   */
  private interpolateEntity(
    a: EntityState,
    b: EntityState,
    alpha: number,
  ): EntityState {
    // Check for teleportation
    if (b.teleported || vec3DistanceSq(a.position, b.position) > this.config.teleportDistanceSq) {
      // Snap to new position — don't interpolate
      return { ...b };
    }

    return {
      id: a.id,
      position: lerpVec3(a.position, b.position, alpha),
      rotation: slerpQuat(a.rotation, b.rotation, alpha),
      velocity: lerpVec3(a.velocity, b.velocity, alpha),
      // Discrete values: use snapshot A's value until alpha >= 1, then switch to B
      health: alpha < 1.0 ? a.health : b.health,
      animationId: alpha < 0.5 ? a.animationId : b.animationId,
      animationTime: a.animationTime + (b.animationTime - a.animationTime) * alpha,
      teleported: false,
    };
  }

  /**
   * Remove snapshots older than the given time, keeping at least one
   * snapshot before renderTime for interpolation.
   */
  private pruneOldSnapshots(renderTime: number): void {
    // Keep one snapshot before renderTime as the interpolation source
    while (
      this.buffer.length > 2 &&
      this.buffer[1].serverTime < renderTime
    ) {
      this.buffer.shift();
    }
  }
}

// ──────────────────────────────────────────────
// Delta Compression (Server-Side Helper)
// ──────────────────────────────────────────────

interface DeltaField {
  entityId: number;
  field: string;
  value: number | boolean | Vec3 | Quat;
}

interface DeltaSnapshot {
  sequence: number;
  serverTime: number;
  baselineSequence: number;  // -1 = full snapshot (no baseline)
  changedEntities: Map<number, Partial<EntityState>>;
  removedEntities: number[];
  newEntities: EntityState[];
}

/**
 * Compute a delta snapshot between two full snapshots.
 * This would run on the server.
 */
function computeDelta(
  current: Snapshot,
  baseline: Snapshot | null,
): DeltaSnapshot {
  const delta: DeltaSnapshot = {
    sequence: current.sequence,
    serverTime: current.serverTime,
    baselineSequence: baseline ? baseline.sequence : -1,
    changedEntities: new Map(),
    removedEntities: [],
    newEntities: [],
  };

  if (!baseline) {
    // No baseline — send everything as "new"
    for (const [id, state] of current.entities) {
      delta.newEntities.push(state);
    }
    return delta;
  }

  // Find new and changed entities
  for (const [id, currentState] of current.entities) {
    const baselineState = baseline.entities.get(id);

    if (!baselineState) {
      // New entity — not in baseline
      delta.newEntities.push(currentState);
      continue;
    }

    // Compare fields and collect changes
    const changes: Partial<EntityState> = {};
    let hasChanges = false;

    if (
      currentState.position.x !== baselineState.position.x ||
      currentState.position.y !== baselineState.position.y ||
      currentState.position.z !== baselineState.position.z
    ) {
      changes.position = currentState.position;
      hasChanges = true;
    }

    if (
      currentState.rotation.x !== baselineState.rotation.x ||
      currentState.rotation.y !== baselineState.rotation.y ||
      currentState.rotation.z !== baselineState.rotation.z ||
      currentState.rotation.w !== baselineState.rotation.w
    ) {
      changes.rotation = currentState.rotation;
      hasChanges = true;
    }

    if (currentState.health !== baselineState.health) {
      changes.health = currentState.health;
      hasChanges = true;
    }

    if (currentState.animationId !== baselineState.animationId) {
      changes.animationId = currentState.animationId;
      hasChanges = true;
    }

    if (hasChanges) {
      delta.changedEntities.set(id, changes);
    }
  }

  // Find removed entities (in baseline but not in current)
  for (const id of baseline.entities.keys()) {
    if (!current.entities.has(id)) {
      delta.removedEntities.push(id);
    }
  }

  return delta;
}

/**
 * Apply a delta snapshot to reconstruct the full snapshot.
 * This would run on the client.
 */
function applyDelta(
  delta: DeltaSnapshot,
  baseline: Snapshot | null,
): Snapshot {
  const entities = new Map<number, EntityState>();

  // Start with baseline entities (if any)
  if (baseline) {
    for (const [id, state] of baseline.entities) {
      entities.set(id, { ...state });
    }
  }

  // Apply changes
  for (const [id, changes] of delta.changedEntities) {
    const existing = entities.get(id);
    if (existing) {
      entities.set(id, { ...existing, ...changes });
    }
  }

  // Add new entities
  for (const newEntity of delta.newEntities) {
    entities.set(newEntity.id, { ...newEntity });
  }

  // Remove deleted entities
  for (const removedId of delta.removedEntities) {
    entities.delete(removedId);
  }

  return {
    sequence: delta.sequence,
    serverTime: delta.serverTime,
    entities,
  };
}

// ──────────────────────────────────────────────
// Usage Example
// ──────────────────────────────────────────────

/*
// === Setup ===
const clock = new ServerClockEstimator();
const buffer = new SnapshotInterpolationBuffer(clock, {
  baseDelay: 100,        // 100ms base delay
  adaptiveDelay: true,   // adapt to network jitter
  jitterMultiplier: 3,
  teleportDistanceSq: 50 * 50,
});

// === When a snapshot packet arrives from the network ===
function onSnapshotReceived(packet: NetworkPacket): void {
  const snapshot = deserializeSnapshot(packet);
  const now = performance.now();
  buffer.receiveSnapshot(snapshot, now);
}

// === Every render frame ===
function renderFrame(): void {
  const now = performance.now();
  const state = buffer.sample(now);

  if (state) {
    for (const [entityId, entityState] of state.entities) {
      const renderObj = getRenderObject(entityId);
      if (renderObj) {
        renderObj.position.set(
          entityState.position.x,
          entityState.position.y,
          entityState.position.z,
        );
        renderObj.quaternion.set(
          entityState.rotation.x,
          entityState.rotation.y,
          entityState.rotation.z,
          entityState.rotation.w,
        );
      }
    }

    // Display buffer health indicator (for debugging / netgraph)
    debugUI.setBufferHealth(state.bufferHealth);
    debugUI.setRenderDelay(now - state.renderTime);
  }

  requestAnimationFrame(renderFrame);
}
*/
```

---

## Comparison with Entity Interpolation

The terms "snapshot interpolation" and "entity interpolation" are often used interchangeably, which causes confusion. They are closely related but describe different scopes:

### Definitions

| Term | Scope | Description |
|---|---|---|
| **Snapshot Interpolation** | **Architecture** | The entire netcode pattern: server sends world snapshots, clients buffer and interpolate between them. Encompasses the full pipeline from capture to rendering. |
| **Entity Interpolation** | **Technique** | The per-entity math of smoothly blending between two known states. A component *within* snapshot interpolation (and other architectures). |

Entity interpolation is *part of* snapshot interpolation. But entity interpolation can also be used in other architectures — for example, a state synchronization system that sends per-entity updates (not full snapshots) still uses entity interpolation for rendering.

### Key Differences

| Aspect | Snapshot Interpolation | Entity Interpolation (standalone) |
|---|---|---|
| **What is sent** | Full world state per tick (all entities at same timestamp) | Per-entity state updates (potentially at different times) |
| **Temporal coherence** | All entities in a snapshot share the same server timestamp | Entities may have updates at different times, causing temporal misalignment |
| **Bandwidth** | Higher baseline (full world per tick) but excellent delta compression | Lower baseline (only changed entities) but harder to delta-compress |
| **Consistency** | Interactions between entities appear synchronized | Interactions may appear slightly desynced due to different update times |
| **Interpolation target** | Always have exactly two world states to blend between | May have different numbers of samples per entity |
| **CPU on client** | Minimal — just interpolation math | Minimal — just interpolation math |

### Gabriel Gambetta's Perspective

Gabriel Gambetta describes entity interpolation as follows: when a position update arrives at time `t=1000`, the client already has the prior update from `t=900`. Between `t=1000` and `t=1100`, the client renders the movement from `t=900` to `t=1000`. The result is that "you're always showing the user *actual* movement data, except you're showing it 100ms 'late'."

This is functionally identical to snapshot interpolation's approach, but described at the per-entity level. In practice, the distinction matters when:

- **Snapshot interpolation**: You can guarantee all entities are temporally aligned in every frame.
- **Per-entity updates**: Different entities may receive updates at different frequencies (via priority systems), so their "lag behind real-time" may differ slightly.

---

## Framework Implementations

### Gaffer on Games Reference Implementation

Glenn Fiedler published a complete reference implementation accompanying his "Networked Physics" article series. It demonstrates:

- 900-cube physics simulation
- Snapshot capture and serialization
- Quantization (position, quaternion "smallest three")
- Delta compression with multi-level encoding
- Jitter buffer with configurable delay
- Linear and Hermite spline interpolation

The source code is available on [GitHub](https://github.com/mas-bandwidth/yojimbo) as part of the Yojimbo networking library and the [netcode.io](https://netcode.io) project.

### Photon Fusion (Unity)

Photon Fusion is a high-end Unity networking SDK that uses snapshot interpolation as its core architecture:

- **Tick-based simulation** with consistent time steps.
- **Adaptive interpolation** that adjusts buffering based on current network conditions.
- **Built-in components** (`NetworkTransform`, `NetworkRigidbody`) with automatic interpolation.
- **Two modes**: Server Mode (dedicated server authority) and Shared Mode (both use snapshot interpolation since Fusion 2).
- Supports both interpolated and predicted objects in the same scene.

### Unity Netcode for Entities

Unity's DOTS-based networking package (Netcode for Entities) implements snapshot interpolation natively:

- **Ghost snapshots**: Entities are replicated as "ghosts" with automatic serialization.
- **Interpolated vs. predicted ghosts**: Each ghost type can be configured for interpolation (rendered behind server time) or prediction (rendered ahead).
- **Delta compression**: Built-in per-component delta encoding.
- **Configurable interpolation delay**: Uses `ClientTickRate.InterpolationTimeMS` and `InterpolationTimeNetTicks`.

### SnapNet

SnapNet is a networking framework specifically designed around the snapshot interpolation architecture, providing:

- Structured snapshot serialization with built-in delta compression.
- Priority-based entity relevance filtering.
- Adaptive interpolation buffering.

### Quake III (id Tech 3)

The original and most influential implementation. Key characteristics documented from the source code:

- 32-snapshot ring buffer per client (binary mask cycling).
- Per-field delta using `netField_t` metadata tables.
- Huffman compression with pre-computed static frequency tables.
- XOR obfuscation layer.
- 1400-byte pre-fragmentation to avoid MTU issues.
- Default 20 snapshots/second (`sv_fps 20`), configurable up to 40.
- Client `snaps` and `rate` cvars for throttling.

### Source Engine (Valve)

The Source Engine's implementation is notable for:

- Configurable tick rates (`sv_tickrate`, commonly 64 or 128 for competitive CS).
- Client interpolation controls: `cl_interp` (interpolation period), `cl_interp_ratio` (multiplier on tick interval).
- Server-side lag compensation with entity position history rewinding.
- Per-entity PVS (Potentially Visible Set) filtering to reduce snapshot size.
- StringTable delta compression for repeated data.

---

## When to Use / When Not to Use

### Ideal Use Cases

| Use Case | Why Snapshot Interpolation Works Well |
|---|---|
| **Spectator / replay systems** | No prediction needed; pure interpolation produces smooth, accurate playback |
| **FPS games (server-authoritative)** | Combined with client-side prediction for the local player and lag compensation for hit detection; proven architecture (Quake III, CS, Apex Legends, Valorant, Call of Duty) |
| **Strategy games with limited entities** | Low entity count keeps snapshots small; visual latency is less noticeable |
| **Games with mostly passive remote entities** | NPCs, environmental objects, and other non-player entities are cheap to interpolate |
| **Games requiring anti-cheat guarantees** | Server authority is absolute; clients cannot fabricate entity states |

### Poor Fit / Limitations

| Scenario | Problem |
|---|---|
| **Player-to-player physics** (e.g., pushing, riding) | Predicted local player interacts with interpolated remote player in different time streams, causing desync artifacts |
| **Competitive games at very low tick rates** (< 20 Hz) | Interpolation delay exceeds 200ms, making combat feel sluggish |
| **Very high player counts** (100+ in a single scene) | Snapshot size grows linearly with entity count; per-client delta encoding CPU cost grows with player count squared |
| **Deterministic lockstep games** (RTS with thousands of units) | Sending the state of every unit is impractical; input-based synchronization is more efficient |
| **Peer-to-peer architectures** | Snapshot interpolation assumes a single authoritative server; P2P requires different approaches (rollback, lockstep) |
| **Physics-heavy cooperative games** (e.g., Rocket League) | The dual-time-stream problem (predicted car + interpolated ball) creates significant interaction artifacts; Rocket League's predecessor used snapshot interpolation but Psyonix switched to rollback |

### Decision Framework

```
Should you use Snapshot Interpolation?

  ┌─ Is there a dedicated server?
  │   ├─ NO  → Consider rollback or lockstep
  │   └─ YES
  │       ├─ Are there < ~200 networked entities visible at once?
  │       │   ├─ NO  → Consider state synchronization (priority-based) or area-of-interest filtering
  │       │   └─ YES
  │       │       ├─ Do players need to physically interact with each other? (pushing, riding, melee)
  │       │       │   ├─ YES (heavily) → Consider rollback
  │       │       │   └─ NO or minimal
  │       │       │       └─ ✅ Snapshot Interpolation is likely a good fit
  │       │       │           ├─ Add client-side prediction for local player responsiveness
  │       │       │           └─ Add lag compensation for server-side hit detection
```

### Hybrid Approaches

Most modern competitive games do not use "pure" snapshot interpolation. They combine it with:

- **Client-side prediction**: The local player is predicted ahead of the server using local input replay, while all other entities are interpolated from snapshots.
- **Lag compensation / backward reconciliation**: The server rewinds entity positions when processing shots, matching what the shooting client saw at the time they fired.
- **Interest management / relevance filtering**: Only entities within the client's perceptual range are included in their snapshots.
- **Priority-based updates**: When snapshots approach MTU limits, lower-priority entities are sent less frequently (used in Overwatch's bandwidth management).

This hybrid is the architecture used by Counter-Strike 2, Apex Legends, Valorant, Call of Duty, Overwatch 2, and most modern competitive shooters.

---

## References

### Primary Sources

1. **Glenn Fiedler (Gaffer on Games) — "Snapshot Interpolation"**
   https://gafferongames.com/post/snapshot_interpolation/
   Comprehensive article covering the interpolation buffer, jitter handling, Hermite spline interpolation, and bandwidth calculations.

2. **Glenn Fiedler (Gaffer on Games) — "Snapshot Compression"**
   https://gafferongames.com/post/snapshot_compression/
   Detailed analysis of quantization, smallest-three quaternion encoding, delta compression with multi-level encoding, achieving 90% bandwidth reduction.

3. **Glenn Fiedler (Gaffer on Games) — "State Synchronization"**
   https://gafferongames.com/post/state_synchronization/
   Comparison architecture: priority accumulators, jitter buffers, visual smoothing, delta compression for per-entity updates.

4. **Gabriel Gambetta — "Fast-Paced Multiplayer" Series**
   - [Client-Server Game Architecture](https://www.gabrielgambetta.com/client-server-game-architecture.html)
   - [Entity Interpolation](https://www.gabrielgambetta.com/entity-interpolation.html)
   - [Lag Compensation](https://www.gabrielgambetta.com/lag-compensation.html)

5. **Quake III Source Code Review: Network Model (Fabien Sanglard)**
   https://fabiensanglard.net/quake3/network.php
   Deep analysis of Quake III's networking code including delta compression, field encoding, memory layout, and Huffman compression.

6. **Quake 3 Network Protocol (Jacek Fedorynski)**
   https://www.jfedor.org/quake3/
   Complete reverse-engineered protocol specification: packet structure, entity encoding, delta compression, Huffman tables, XOR obfuscation, user command format.

7. **Quake 3 Networking Primer**
   https://www.ra.is/unlagged/network.html
   Practical guide to Quake III's snapshot system, interpolation/extrapolation, command processing, and timing.

### GDC Presentations

8. **Timothy Ford — "Overwatch Gameplay Architecture and Netcode" (GDC 2017)**
   https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and
   ECS architecture, deterministic fixed-tick simulation, client-ahead clock model, delta compression.

### Framework Documentation

9. **Photon Fusion — Network Simulation Loop**
   https://doc.photonengine.com/fusion/current/concepts-and-patterns/network-simulation-loop

10. **Unity Netcode for Entities — Interpolation and Extrapolation**
    https://docs.unity3d.com/Packages/com.unity.netcode@1.7/manual/interpolation.html

11. **SnapNet — Netcode Architectures Part 3: Snapshot Interpolation**
    https://www.snapnet.dev/blog/netcode-architectures-part-3-snapshot-interpolation/

### Additional Resources

12. **The Quake3 Networking Model (Book of Hook)**
    http://trac.bookofhook.com/bookofhook/trac.cgi/wiki/Quake3Networking

13. **Yojimbo Networking Library (Glenn Fiedler)**
    https://github.com/mas-bandwidth/yojimbo

14. **netcode.io**
    https://netcode.io
