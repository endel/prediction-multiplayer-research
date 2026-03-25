# VALORANT Netcode: Deep Dive

An analysis of Riot Games' networking architecture for VALORANT, covering server
simulation, peeker's advantage, lag compensation, and the Riot Direct
infrastructure that underpins it all.

---

## 1. Overview: Riot Games' Approach to Competitive FPS Networking

VALORANT is a tactical shooter where time-to-kill is extremely low -- most
weapons can kill with a single well-placed headshot. This design constraint makes
networking quality the difference between a fair competitive experience and an
unplayable one.

Riot's networking philosophy centres on three design goals:

1. **Game Fairness** -- minimise the reaction-time advantage that a peeking
   player has over a player holding an angle.
2. **Smooth Movement** -- characters should never pop or slide; the world
   presented to each player should be as close to ground truth as possible.
3. **Holder's Advantage** -- in a tactical shooter the defender should have the
   edge, not the aggressor.

To achieve these goals Riot invested across the entire stack: custom internet
backbone (Riot Direct), 128-tick dedicated servers, a server-authoritative
networking model, fixed-timestep simulation decoupled from render framerate, and
aggressive buffering minimisation.

---

## 2. 128-Tick Servers and Fixed Timestep Simulation

### Why 128 Tick?

VALORANT servers run at **128 ticks per second**, yielding a frame budget of
**7.8125 ms** per tick. The choice of 128 tick (vs. the 64 tick that was
standard in CS:GO) halves the per-frame duration and directly reduces peeker's
advantage by shrinking buffering windows.

### Server Performance Budget

Riot's goal was to run **3 game instances per CPU core** on 36-core hosts (108
concurrent matches / 1,080 players per physical server). After reserving 10% for
OS overhead, the per-game frame budget is:

```
7.8125 ms / 3 games = 2.6 ms per game
2.6 ms * 0.90 (OS headroom) ≈ 2.34 ms target per frame per game
```

Initial measurements showed **50 ms per frame** -- requiring a roughly **20x
optimisation** effort before launch.

### Key Server-Side Optimisations

| Area | Technique | Impact |
|------|-----------|--------|
| Replication | Replaced UE4 property-polling replication with event-driven RPCs | 100x -- 10,000x improvement |
| Animation | Frame-skipping (compute every 4th frame, interpolate between) | ~75% cost reduction |
| Animation (phases) | Disable server-side animation during buy phase | Additional ~33% savings |
| CPU architecture | Migrated to Intel Xeon Scalable (non-inclusive L3 cache) | ~30% improvement |
| NUMA binding | `numactl` to pin processes to local memory nodes (97-99% local access) | ~5% improvement |
| Linux scheduler | CFS migration cost reduced from 0.5 ms to 0 | ~4% improvement |
| C-States | Restricted to C0/C1/C1E to avoid deep-sleep transition costs | 1-3% improvement |
| Hyperthreading | Re-enabled after other optimisations | ~25% improvement |
| Clocksource | Switched from Xen to CPU `tsc` instruction | 1-3% improvement |

---

## 3. Decoupling Simulation from Render Framerate

Running two 5 ms physics updates back-to-back yields a **different result** than
running a single 10 ms update, because forces are numerically integrated
(accumulated and applied) each step. A client at 30 Hz would drift from a server
at 128 Hz and require constant correction.

VALORANT solves this by **decoupling simulation updates from the render
framerate**. Regardless of how many FPS the client renders, the movement /
physics / gameplay simulation always runs at a **fixed timestep of exactly 128
updates per second** (7.8125 ms per step).

### How It Works at Different Framerates

```
Client @ 60 FPS (~16.67 ms per render frame):
  - Each render frame simulates ~2 movement ticks
  - Ticks are interpolated to the exact render time

Client @ 144 FPS (~6.94 ms per render frame):
  - Some render frames have 0 new simulation ticks
  - The client interpolates the single tick across multiple render frames

Client @ 240 FPS (~4.17 ms per render frame):
  - Similar to 144 FPS -- blend a single simulation update across frames
```

The client simulates slightly into the future (e.g. move #408) as needed to know
exactly where the player will be at frame boundaries, then linearly interpolates
the world state within a move update to draw things at the correct position when
the frame is rendered.

### Pseudo-code: Fixed Timestep with Render Interpolation

```python
TICK_RATE = 128
TICK_DURATION = 1.0 / TICK_RATE  # 7.8125 ms

accumulator = 0.0
previous_state = current_state.copy()

def game_loop():
    while running:
        dt = time_since_last_frame()
        accumulator += dt

        # Run as many fixed-step simulation ticks as needed
        while accumulator >= TICK_DURATION:
            previous_state = current_state.copy()
            gather_input()
            simulate_tick(current_state, TICK_DURATION)   # physics, movement
            send_input_and_result_to_server(current_state)
            accumulator -= TICK_DURATION

        # Interpolate for rendering (sub-tick precision)
        alpha = accumulator / TICK_DURATION
        render_state = lerp(previous_state, current_state, alpha)
        render(render_state)
```

Because both client and server use the identical 7.8125 ms timestep, when a
client reports "I just simulated move #408" the server knows exactly which move
is being referenced and can run an apples-to-apples comparison.

---

## 4. Client and Server Buffering

### The Problem

The internet is unreliable. Packets arrive late or not at all. If the simulation
consumed data the instant it arrived, any late packet would force a prediction,
causing visual pops and slides when the prediction is corrected. Buffering
smooths out the incoming data stream -- but every buffered frame adds latency and
increases peeker's advantage.

### Server Buffering (simplified model: ~2 frames)

| Component | Duration |
|-----------|----------|
| Average wait until the queue-insertion point in the frame | 0.5 frames |
| Target network buffer (data sits in queue to smooth jitter) | 0.5 frames |
| Application + broadcast (move is applied and sent to clients) | 1.0 frame |
| **Total** | **~2 frames** |

At 128 tick, 1 frame = 7.8125 ms, so 2 frames ≈ **15.625 ms**.

### Client Buffering (simplified model: ~3 frames)

| Component | Duration |
|-----------|----------|
| Average wait until queue-insertion point | 0.5 frames |
| Target network buffer (smoothing) | 1.0 frame |
| GPU / swap-chain delay (variable, 0-2 frames; ~0.5 typical) | 0.5 frames |
| Render output | 1.0 frame |
| **Total** | **~3 frames** |

At 60 FPS, 1 frame ≈ 16.67 ms, so 3 frames ≈ **50 ms**.
At 144 FPS, 1 frame ≈ 6.94 ms, so 3 frames ≈ **20.8 ms**.

### Buffering Targets

- **Server**: half a frame of movement data in the buffer (on average).
- **Client**: one full buffered frame of movement data.
- The client exposes a **"Network Buffering"** setting in the options menu to
  increase the buffer for players with poor internet or bandwidth (the default
  "Minimum" uses the values above).

### Network Interpolation Delay

Separately from the buffering pipeline, VALORANT applies a fixed
**7.8125 ms network interpolation delay** (one server tick) to smooth enemy
player positions on the client. This is the "interp" value visible in the
netcode HUD.

---

## 5. Peeker's Advantage Formula

### The Core Inequality

For the holder's shot to be processed on the server before they die, the
following must be true:

```
Time(PeekerVisible) + ReactionTime(Holder) + RTT_Holder/2 + Buffering(Server)
  < Time(PeekerVisible) + ReactionTime(Peeker) + RTT_Peeker/2 + Buffering(Server)
    + RTT_Holder/2 + Buffering(Holder)
```

Simplifying (cancelling common terms and rearranging):

```
ReactionTime(Holder) < ReactionTime(Peeker)
                       - RTT(Holder_to_Server)
                       - Buffering(Server)
                       - Buffering(Holder)
```

Or equivalently, the **peeker's raw advantage** is:

```
Peeker's Advantage = Buffering(Server) + Buffering(Holder)
                   + RTT(Holder_to_Server)
```

### The Operational Formula (from the dev blog)

From the official Riot post, peeker's advantage is expressed as the sum:

```
Peeker's Advantage =
    <ENEMY'S CLIENT FRAMERATE DELAY>      (1 / FPS of the holder)
  + <ENEMY'S 1-WAY NETWORK LATENCY>       (holder's half-RTT)
  + <SERVER FRAMERATE>                     (7.8125 ms at 128-tick)
  + <YOUR 1-WAY NETWORK LATENCY>          (peeker's half-RTT)
  + <NETWORK INTERP DELAY>                (7.8125 ms at 128-tick)
```

**Constants** (at 128 tick):
- Server frame time: **7.8125 ms**
- Network interp delay: **7.8125 ms**

**Variables**:
- 1-way network latency: Riot targets **< 17.5 ms** (35 ms RTT / 2) for 70% of
  the player population.
- Client framerate delay: depends on the holder's hardware.

### Worked Example 1: Baseline (64-tick, 60 ms ping, 60 FPS)

This represents a "typical older game" scenario:

```
Server buffering:  2 frames @ 64 tick  = 2 * 15.625 ms  = 31.25 ms
Client buffering:  3 frames @ 60 FPS   = 3 * 16.67 ms   = 50.0 ms
Network RTT:                                             = 60.0 ms
                                                         --------
Total peeker's advantage:                                ≈ 141 ms
```

At ~300 ms average human reaction time, 141 ms is **enormous** -- nearly half
your reaction budget is consumed by network disadvantage.

### Worked Example 2: VALORANT Target (128-tick, 35 ms ping, 60 FPS)

```
Server buffering:  2 frames @ 128 tick = 2 * 7.8125 ms  = 15.625 ms
Client buffering:  3 frames @ 60 FPS   = 3 * 16.67 ms   = 50.0 ms
Network RTT:                                             = 35.0 ms
                                                         ---------
Total peeker's advantage:                                ≈ 101 ms
```

This is a **28% reduction** compared to the 64-tick baseline.

### Worked Example 3: High Refresh Rate (128-tick, 35 ms ping, 144 FPS)

```
Server buffering:  2 frames @ 128 tick = 2 * 7.8125 ms  =  15.625 ms
Client buffering:  3 frames @ 144 FPS  = 3 * 6.94 ms    =  20.83 ms
Network RTT:                                             =  35.0 ms
                                                         ----------
Total peeker's advantage:                                ≈  71 ms
```

This is a **49% reduction** from baseline -- demonstrating why higher FPS
matters even beyond visual smoothness.

### Worked Example 4: Using the Operational Formula (128-tick, 35 ms RTT)

```
Holder's client frame time (60 FPS):     16.67 ms
Holder's 1-way latency (35 ms RTT):      17.5 ms
Server frame time (128-tick):              7.8125 ms
Peeker's 1-way latency (35 ms RTT):      17.5 ms
Network interp delay (128-tick):           7.8125 ms
                                         ----------
Total:                                   ≈ 67.3 ms
```

(The slight difference from Example 2 is due to the simplified buffering model
used in the earlier calculation vs. the component-based formula here.)

### Sensitivity at the Pro Level

Riot's internal experiments with top-tier players revealed:

- At the highest skill level, the difference between winning and losing often
  comes down to **20-50 ms** of reaction time.
- Skilled players could **accurately identify ~10 ms changes** to peeker's
  advantage in blind tests.
- **A 10 ms shift** in peeker's advantage swung winrates from **90% holder win
  (with an Operator)** to **90% peeker win (with a rifle)** for evenly matched
  players.

---

## 6. Lag Compensation

### Client Timestamp Transmission

When a player fires, the client sends the **exact simulation time** it was
looking at when the trigger was pulled. Because simulation uses fixed timesteps,
this timestamp maps unambiguously to a specific move number (e.g. "I fired
during move #408").

### Server Rewind (Lag Compensation)

Upon receiving a shot, the server **rewinds the world state** to what the
shooting player was looking at when they pulled the trigger. It then performs
hit detection against that historical state.

```python
def process_shot(shooter, shot_timestamp, aim_direction):
    # 1. Rewind world to shooter's perceived time
    historical_state = world_history.get_state_at(shot_timestamp)

    # 2. Perform hit detection against rewound state
    hit_result = raycast(
        origin=historical_state.get_position(shooter),
        direction=aim_direction,
        targets=historical_state.get_all_player_hitboxes()
    )

    # 3. Apply damage in present time if hit confirmed
    if hit_result.hit:
        current_target = world.get_player(hit_result.target_id)
        current_target.apply_damage(hit_result.damage, hit_result.bodypart)

        # Mark victim as dead -- reject any future shots from them
        if current_target.health <= 0:
            current_target.mark_dead(server_time=now())
```

### Limits on Rewind Distance

The server imposes **limits on how far it will rewind** for hit registration.
Without limits, a player with 500 ms latency could kill an opponent half a
second after they had moved behind cover. The exact cap is not publicly
documented, but the system is designed to reject shots from clients whose
reported timestamps are too far in the past.

### Death Processing

When the server processes a lethal shot, it **marks the victim as dead and
rejects any future shots** from that victim. There is a short window where the
victim does not yet know they are dead and may fire shots locally -- those shots
are discarded by the server.

This creates the familiar "I shot but it didn't count" experience, which is an
inherent trade-off of lag compensation in any server-authoritative system.

---

## 7. Corrections Only Affect the Mispredicting Player

When a client's predicted simulation diverges from the server's authoritative
state, the server sends a correction. The critical design decision in VALORANT
is that **corrections are rare, small in magnitude, and only visible to the
player who encountered the underlying network issue**.

The other nine players in the match see smooth, uninterrupted movement
regardless of any individual player's network problems.

### How This Works

```python
def handle_server_correction(correction):
    """
    Called on the client when the server disagrees with our predicted state.
    Only this client sees the correction -- other players are unaffected.
    """
    # Find how far we've diverged
    delta = correction.server_position - predicted_position_at(correction.move_id)

    if delta.magnitude < SNAP_THRESHOLD:
        # Smoothly blend toward the corrected position over several frames
        apply_smooth_correction(delta, blend_frames=3)
    else:
        # Large divergence: snap to server state (rare)
        snap_to(correction.server_position)

    # Re-simulate all moves from the corrected move forward
    resimulate_from(correction.move_id, correction.server_state)
```

### Why Corrections Are Rare

1. **Fixed timesteps** prevent framerate-induced drift between client and server.
2. **Minimal buffering** keeps the client's predicted state close to the
   server's actual state.
3. **Move queueing** on the server handles minor packet timing variations
   without needing to predict.

When a server tick arrives and no input has been received from a client, the
server predicts that the player "continued to hold down whatever keys were being
held in the last received update, since only a few milliseconds have passed."
This simple heuristic is correct the vast majority of the time.

---

## 8. Riot Direct: Custom Internet Backbone

### What Is Riot Direct?

Riot Direct is Riot Games' **privately operated internet backbone** -- a
dedicated network infrastructure that carries game traffic between players and
game servers, bypassing the unpredictable routing of the public internet.

### The 35 ms Target

Riot's goal is to deliver **35 ms round-trip ping** (17.5 ms one-way) to
**70% of the player population**. This target was set specifically because of
the peeker's advantage equation: every millisecond of RTT directly adds to the
advantage a peeking player has.

### How It Reduces Latency

Standard ISP routing optimises for cost and throughput, not latency. A packet
from a player in Chicago to a server in Dallas might traverse multiple ISP
networks, each adding routing hops and queuing delays. Riot Direct:

1. **Peers directly with ISPs** at internet exchange points, taking custody of
   game traffic as close to the player as possible.
2. **Routes traffic over Riot's own fibre links** between points of presence
   (PoPs), using latency-optimised paths rather than cost-optimised ones.
3. **Eliminates unnecessary hops** -- traffic stays on Riot's network from
   ingress to the game server.

### Architecture

```
Player → ISP → [Internet Exchange] → Riot Direct PoP → Riot backbone
    → Riot Direct PoP (near data centre) → Game Server

vs. standard routing:

Player → ISP → ISP peer → Transit provider → Transit provider
    → ISP peer → Data centre network → Game Server
```

### Impact on VALORANT

The combination of Riot Direct and strategically placed server locations is what
makes the 35 ms target achievable. Without it, average RTTs in many regions
would be 60-100+ ms, roughly doubling peeker's advantage.

Riot has stated that the closed beta was a period for testing and improving
toward this target, and they continue to expand Riot Direct infrastructure
globally.

---

## 9. Anti-Cheat Considerations in the Netcode

### Server-Authoritative Model

VALORANT uses a **server-authoritative networking model** specifically to limit
the types of cheats that are possible. The fundamental principle: **the server
must never trust a client's view of the world**.

| What the client controls | What the server controls |
|--------------------------|--------------------------|
| Input (keys, mouse) | Movement simulation (authoritative) |
| Local prediction (visual only) | Hit registration (authoritative) |
| Camera / rendering | Game state (health, ammo, abilities) |
| Reported timestamps | Rewind limits and validation |

### Specific Anti-Cheat Netcode Measures

1. **Movement validation**: The client sends both its input and its predicted
   movement result. The server re-simulates and corrects if they disagree. A
   speed hack or teleport hack would be immediately corrected.

2. **Server-side hit registration with rewind limits**: Shots are validated on
   the server. The rewind window is capped, so aimbots cannot exploit extreme
   artificial latency to hit targets that have long since moved.

3. **Dead player shot rejection**: Once the server marks a player as dead, all
   subsequent shots from that player are discarded, preventing "trading" exploits
   where a dead player continues to deal damage.

4. **Fog of war / visibility**: The server can restrict what positional data is
   sent to each client based on line-of-sight and proximity, limiting the
   effectiveness of wallhacks. (The server has a field-of-view subsystem
   specifically for this purpose.)

5. **Vanguard (kernel-level anti-cheat)**: While not strictly part of the
   netcode, Vanguard runs at the kernel level to prevent tampering with the game
   client, memory reading, and injection -- complementing the server-side
   protections.

### Pseudo-code: Server-Side Input Validation

```python
def process_client_update(client, update):
    # Client sends: input + predicted movement result
    client_input = update.input          # keys, mouse delta
    client_result = update.predicted_pos # where the client thinks it ended up

    # Server re-simulates from authoritative state
    server_result = simulate_movement(
        start_pos=server_state[client.id].position,
        input=client_input,
        dt=TICK_DURATION
    )

    # Compare
    if distance(server_result, client_result) > CORRECTION_THRESHOLD:
        # Client is wrong (or cheating) -- send correction
        send_correction(client, server_result)
        server_state[client.id].position = server_result
    else:
        # Accept client result (within tolerance)
        server_state[client.id].position = server_result
```

---

## 10. How VALORANT Differs from CS:GO / CS2

| Aspect | CS:GO | CS2 | VALORANT |
|--------|-------|-----|----------|
| **Tick rate** | 64 tick (matchmaking) / 128 tick (FACEIT/ESEA) | Sub-tick (tick-independent) | 128 tick for all players |
| **Cost** | 128 tick required third-party services | Built-in | Free for all players |
| **Network infrastructure** | Standard ISP routing / Valve relay | Steam Datagram Relay | Riot Direct (private backbone) |
| **Latency target** | No published target | No published target | 35 ms RTT for 70% of players |
| **Simulation model** | Tick-locked (sim = tick rate) | Sub-tick interpolation | Fixed 128-step sim, decoupled from render FPS |
| **Client prediction** | Predict + correct | Predict + correct | Predict + correct (corrections scoped to mispredicting player) |
| **Hit registration** | Server-side with rewind | Server-side with sub-tick precision | Server-side with rewind + rewind limits |
| **Movement accuracy** | Movement inaccuracy curve | Movement inaccuracy curve | Aggressive movement inaccuracy + hit tagging (slow on hit) |
| **Anti-cheat** | VAC (user-mode) | VAC + VAC Live | Vanguard (kernel-mode) + server authority |
| **Interp** | Configurable (`cl_interp`, `cl_interp_ratio`) | Engine-managed | Fixed 7.8125 ms (1 server tick); "Network Buffering" toggle |

### Key Philosophical Differences

**CS:GO's 64-tick matchmaking** was long criticised because the 15.625 ms frame
time added significant buffering delay. Valve's answer with CS2 was the
**sub-tick system**, which timestamps player actions between ticks so the server
can reconstruct what happened at sub-tick precision, reducing the impact of tick
rate on gameplay without increasing computational cost.

**VALORANT's answer** was to invest in raw server power to run 128-tick for
everyone, and to invest in network infrastructure (Riot Direct) to reduce RTT --
attacking the problem from both the server-performance and network-latency sides
simultaneously.

**CS2's sub-tick** effectively makes the tick rate less relevant for shot
registration but does not reduce the buffering windows for movement visibility.
VALORANT's 128-tick approach reduces both.

---

## 11. Implementation Concepts

### Move Queueing and Scheduling

The server maintains a queue of upcoming moves for each client, keyed by move
number. When data arrives, the server slots it into the corresponding position:

```python
class MoveQueue:
    def __init__(self):
        self.queue = {}           # move_number -> input_data
        self.basis = None         # (client_move, server_move) tuple
        self.last_input = None    # for prediction when data is missing

    def establish_basis(self, client_move_num, server_move_num):
        """Map client timeline to server timeline."""
        self.basis = (client_move_num, server_move_num)
        # e.g. client move #239 = server move #400

    def client_to_server_move(self, client_move):
        """Convert client move number to server move number."""
        offset = client_move - self.basis[0]
        return self.basis[1] + offset

    def enqueue(self, client_move_num, input_data):
        server_move = self.client_to_server_move(client_move_num)
        self.queue[server_move] = input_data
        self.last_input = input_data

    def get_input_for_tick(self, server_move_num):
        if server_move_num in self.queue:
            return self.queue.pop(server_move_num)
        else:
            # Predict: assume player continues holding last known keys
            return self.last_input
```

### World State History for Lag Compensation

```python
class WorldHistory:
    def __init__(self, max_rewind_ticks=128):
        self.history = collections.deque(maxlen=max_rewind_ticks)

    def record(self, tick_number, state_snapshot):
        """Save a snapshot of all player positions/hitboxes at this tick."""
        self.history.append((tick_number, state_snapshot))

    def get_state_at(self, target_tick):
        """Retrieve the world state at a given tick for lag compensation."""
        # Reject rewind requests that are too far in the past
        oldest = self.history[0][0] if self.history else 0
        if target_tick < oldest:
            raise RewindLimitExceeded(
                f"Cannot rewind to tick {target_tick}, "
                f"oldest available is {oldest}"
            )

        for tick, state in self.history:
            if tick == target_tick:
                return state

        # Interpolate between two nearest snapshots if exact tick not found
        return self._interpolate(target_tick)
```

### Client-Side Prediction and Reconciliation

```python
class ClientPrediction:
    def __init__(self):
        self.pending_inputs = []   # inputs sent but not yet confirmed
        self.last_confirmed_move = 0

    def predict(self, input_data, move_number):
        """Apply input locally for immediate feedback."""
        self.pending_inputs.append((move_number, input_data))
        apply_input_locally(input_data)

    def reconcile(self, server_correction):
        """
        Server disagreed with our prediction.
        Re-simulate from the corrected state forward.
        Only THIS client sees any visual correction.
        """
        self.last_confirmed_move = server_correction.move_id

        # Discard inputs older than the corrected move
        self.pending_inputs = [
            (move_num, inp) for move_num, inp in self.pending_inputs
            if move_num > server_correction.move_id
        ]

        # Snap to server's authoritative state
        set_state(server_correction.state)

        # Re-apply all unconfirmed inputs on top of corrected state
        for move_num, inp in self.pending_inputs:
            apply_input_locally(inp)
```

### Peeker's Advantage Calculator

```python
def peekers_advantage_ms(
    server_tick_rate: int,
    holder_fps: float,
    holder_rtt_ms: float,
    peeker_rtt_ms: float,
    interp_frames: float = 1.0,   # network interp (in server frames)
    server_buffer_frames: float = 2.0,
    client_buffer_frames: float = 3.0,
):
    """
    Estimate peeker's advantage in milliseconds.

    Uses the operational formula:
      PA = holder_frame_time + holder_one_way + server_frame_time
         + peeker_one_way + interp_delay
    """
    server_frame_ms = 1000.0 / server_tick_rate
    holder_frame_ms = 1000.0 / holder_fps

    holder_one_way = holder_rtt_ms / 2.0
    peeker_one_way = peeker_rtt_ms / 2.0
    interp_delay = interp_frames * server_frame_ms

    advantage = (
        holder_frame_ms
        + holder_one_way
        + server_frame_ms
        + peeker_one_way
        + interp_delay
    )
    return advantage

# --- Worked examples ---
# Baseline: 64 tick, 60ms ping, 60 FPS
print(peekers_advantage_ms(64, 60, 60, 60))
# => ~67.3 ms (operational formula)
# Full buffering model: ~141 ms

# VALORANT target: 128 tick, 35ms ping, 60 FPS
print(peekers_advantage_ms(128, 60, 35, 35))
# => ~67.3 ms (operational formula)
# Full buffering model: ~101 ms

# High refresh: 128 tick, 35ms ping, 144 FPS
print(peekers_advantage_ms(128, 144, 35, 35))
# => ~57.7 ms (operational formula)
# Full buffering model: ~71 ms

# Pro setup: 128 tick, 20ms ping, 240 FPS
print(peekers_advantage_ms(128, 240, 20, 20))
# => ~39.8 ms
```

---

## 12. Summary: The Full Picture

```
┌──────────┐                                      ┌──────────┐
│  PEEKER  │                                      │  HOLDER  │
│ (Client) │                                      │ (Client) │
│          │                                      │          │
│ Fixed 128│        ┌──────────────────┐          │ Fixed 128│
│ tick sim │───────►│   GAME SERVER    │─────────►│ tick sim │
│          │  input │  128 tick auth.  │ state    │          │
│ Predict  │  move  │  Lag compensate  │ updates  │ Buffer   │
│ locally  │  result│  Rewind + verify │          │ + interp │
│          │        │  Move queue/sched│          │          │
│ Decouple │        │  Riot Direct     │          │ Decouple │
│ sim from │        │  backbone        │          │ sim from │
│ render   │        └──────────────────┘          │ render   │
└──────────┘                                      └──────────┘

Peeker's Advantage (time holder cannot see peeker):
= ServerBuffering + HolderClientBuffering + HolderRTT
= ~15.6ms (128t) + ~50ms (60fps) + 35ms = ~101ms
= ~15.6ms (128t) + ~20.8ms (144fps) + 35ms = ~71ms
```

---

## References

1. **"Peeking at VALORANT's Netcode"** -- Riot Games Technology Blog
   - https://technology.riotgames.com/news/peeking-valorants-netcode
   - Deep technical breakdown of peeker's advantage formula, buffering, fixed
     timesteps, lag compensation, and correction scoping.

2. **"04: On Peeker's Advantage & Ranked"** -- VALORANT Dev Blog
   - https://playvalorant.com/en-us/news/game-updates/04-on-peeker-s-advantage-ranked/
   - David Straily's operational formula for peeker's advantage, the 35 ms / 70%
     target, network interp delay of 7.8125 ms, and discussion of strafe
     shooting desync.

3. **"VALORANT's 128-Tick Servers"** -- Riot Games Technology Blog
   - https://technology.riotgames.com/news/valorants-128-tick-servers
   - Server performance budget, optimisation techniques (RPC vs. replication,
     animation frame-skipping, NUMA binding, CFS tuning), and hardware
     selection.

4. **Riot Direct** -- Riot Games Technology Blog
   - https://technology.riotgames.com/news/riot-direct
   - Architecture of Riot's private internet backbone, ISP peering strategy,
     and latency reduction methodology.

5. **"Fixing Peeker's Advantage with Client-Side Prediction"** -- Gabriel Gambetta
   - https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html
   - General reference on client-side prediction and server reconciliation
     techniques used across FPS games.

6. **Human Reaction Time Study** (referenced by Riot)
   - https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4456887/
   - Average human reaction time ~247 ms, contextualising the impact of
     peeker's advantage on competitive play.
