# EVE Online: Netcode Deep-Dive

## Overview

EVE Online (CCP Games, 2003) is a spaceship MMO that defies nearly every convention of real-time multiplayer networking. Where most games shard their playerbase across dozens of identical servers and run simulations at 20-128 Hz, EVE puts every player in a single shared universe and ticks its physics at **1 Hz**. The result is a game where thousands of ships can clash in one solar system -- at the cost of real-time responsiveness -- using a unique set of technologies including Stackless Python, the Destiny physics engine, a formula-driven combat model, and the landmark **Time Dilation (TiDi)** system that slows the simulation rather than letting it desync or crash.

EVE's networking story is one of working around a single fundamental bottleneck -- Python's Global Interpreter Lock (GIL) -- across two decades of progressive re-architecture: from the original StacklessIO, through CarbonIO/BlueNet (2011), to Quasar/gRPC (2021+).

### Key Characteristics

| Property | Value |
|---|---|
| **Server tick rate** | 1 Hz (one simulation step per second) |
| **Architecture** | Single-shard cluster ("Tranquility"), all players in one universe |
| **Language** | Stackless Python (server and client game logic) |
| **Physics engine** | Destiny (deterministic, runs on both server and client) |
| **Combat model** | Formula-based (tracking, signature radius, range -- no spatial hitboxes) |
| **Lag mitigation** | Time Dilation (TiDi) -- slows simulation to 10% rather than desyncing |
| **Peak concurrent users** | ~65,000+ on one cluster |
| **Record battle** | 6,557 players in one system (M2-XFE, 2020-2021) |

---

## Architecture

### Single-Shard Philosophy

Most MMOs split their playerbase across separate "shards" or "worlds" -- identical copies of the game running in parallel. EVE Online rejected this from day one. CCP's design philosophy was that a single shared universe was not merely a technical preference but the foundation for meaningful emergent gameplay: a unified economy driven by real supply and demand, political alliances spanning thousands of players, and wars where every battle is part of one continuous history.

As CCP put it: "A single shard should be the natural choice of any MMO developer." The single-shard approach means that every player action can, in principle, ripple through the entire universe. It also means the server infrastructure must handle arbitrarily large concentrations of players in one location -- a problem no amount of load-balancing across shards can solve when the players themselves choose to congregate.

### The Tranquility Cluster

EVE's production server cluster, **Tranquility**, is located in London (originally; later migrated to TQ Datacenter). Its core components:

```
+------------------------------------------------------+
|                  TRANQUILITY CLUSTER                 |
|                                                      |
|  +------------+    +------------+    +------------+  |
|  | Proxy      |    | Proxy      |    | Proxy      |  |
|  | Blade      |    | Blade      |    | Blade      |  |
|  +-----+------+    +-----+------+    +-----+------+  |
|        |                  |                  |        |
|        +--------+---------+--------+---------+        |
|                 |                  |                   |
|  +--------------v--+  +-----------v-------+           |
|  |   SOL Blades    |  |   SOL Blades      |          |
|  |  (90-100 blades |  |  Each blade runs  |          |
|  |   2 nodes each) |  |  2 sol nodes      |          |
|  +---------+-------+  +---------+---------+          |
|            |                    |                     |
|            +--------+-----------+                     |
|                     |                                 |
|          +----------v----------+                      |
|          |   Database Layer    |                      |
|          | (SQL Server + SSDs) |                      |
|          | 250M txns/day       |                      |
|          +---------------------+                      |
+------------------------------------------------------+
```

- **Proxy Blades**: Public-facing servers that accept player TCP connections, handle authentication/encryption, and route traffic into the cluster.
- **SOL Blades**: The workhorses. ~90-100 blades, each running 2 "nodes" (server processes). Each node runs one CPU core worth of Stackless Python -- one node can host many low-population solar systems, or a single high-traffic system (like the Jita trade hub) can be pinned to a dedicated node.
- **Database Layer**: SQL Server on SSDs handling persistent state (items, market orders, skills, etc.). The 1 Hz tick rate was partly chosen to keep database load manageable -- 250 million transactions per day.

### Stackless Python

EVE's server (and client game logic) is written in **Stackless Python**, a variant of CPython that adds lightweight microthreads called **tasklets**. Tasklets are cooperatively scheduled -- they yield execution only at explicit switching points (channel send/receive, sleep, I/O operations). This eliminates the need for locks and mutexes that plague conventional multithreaded code.

```python
# Pseudo-code: Stackless Python tasklet model in EVE

import stackless

def ship_behavior(ship_id):
    """Each ship's AI/behavior runs as a tasklet."""
    while ship.is_alive():
        target = find_nearest_enemy(ship_id)
        if target:
            fire_weapons(ship_id, target)
        # Cooperatively yield -- other tasklets run now
        stackless.schedule()

# Thousands of tasklets, all on one OS thread
for ship_id in all_ships_on_grid:
    stackless.tasklet(ship_behavior)(ship_id)

# The scheduler round-robins through tasklets
stackless.run()
```

The critical limitation: **Stackless Python still has the GIL.** Despite thousands of cooperative tasklets, only one can execute at any instant. A multi-core server burns one CPU core and leaves the rest idle (unless multiple sol nodes are deployed). This single-threaded bottleneck has driven every major networking overhaul in EVE's history.

---

## The 1 Hz Tick

### Why 1 Hz?

EVE's server simulation runs at **1 tick per second** (1 Hz). This is extraordinarily slow compared to most multiplayer games (Overwatch: 63 Hz, Counter-Strike: 128 Hz, Rocket League: 120 Hz). The choice was deliberate:

1. **Bandwidth**: With thousands of entities on a single grid, update packets scale with entity count. At 1 Hz, the bandwidth stays manageable even with massive fleets.
2. **CPU budget**: Each tick must process physics, combat, effects, and networking for every entity. At 1 Hz, the server has a full second to complete this work.
3. **Genre fit**: EVE's combat is strategic, not twitch-based. Ships take seconds to lock targets, minutes to align and warp. A 1-second resolution is acceptable for the pace of gameplay.

### The Tick Loop

The Destiny engine processes each tick in three sequential phases:

```python
# Pseudo-code: EVE Online 1 Hz server tick loop (Destiny engine)

TICK_INTERVAL = 1.0  # seconds (real-time, before TiDi scaling)

class DestinyBallpark:
    """Manages all entities ('balls') in a solar system."""

    def run_tick_loop(self):
        while self.is_running:
            tick_start = time.monotonic()

            # === PHASE 1: PRE-TICK ===
            # Process queued client commands (warp, orbit, activate module)
            self.process_pending_commands()

            # Update bubble-ball membership (spatial partitioning)
            self.update_bubble_assignments()

            # Determine which clients need updates about added/removed entities
            # (This was the original O(n*m) bottleneck fixed by Team Gridlock)
            updates = self.gather_client_updates()

            # Serialize and dispatch state updates to clients
            #   - Pre-serialize once per bubble, share blob across all observers
            self.send_updates_to_clients(updates)

            # === PHASE 2: EVOLVE (Physics) ===
            # Advance all ball positions by one tick
            for ball in self.all_balls:
                ball.position += ball.velocity * TICK_INTERVAL
                ball.velocity = apply_flight_model(ball)

            # Resolve collisions between balls
            self.resolve_collisions()

            # === PHASE 3: POST-TICK ===
            # Handle newly connected clients, cleanup
            self.process_new_connections()
            self.cleanup_destroyed_entities()

            # Sleep for the remainder of the tick
            elapsed = time.monotonic() - tick_start
            sleep_time = max(0, TICK_INTERVAL - elapsed)
            time.sleep(sleep_time)

            # If elapsed > TICK_INTERVAL, the tick is "late"
            # This triggers Time Dilation (see below)
```

### Timing Implications

The 1 Hz tick creates distinct gameplay behaviors:

- **Action rounding**: All timed actions round up to the next whole tick. A 1.2s lock time becomes 2 ticks (2 seconds). A 3.6s align time becomes 4 ticks.
- **Delayed visibility**: When you activate a module, your client gets immediate acknowledgment. But every other player (and the target) learns the result only at the next tick boundary.
- **Multiple shots per tick**: Weapons with sub-second cycle times fire multiple times within a tick. Both shots appear simultaneously in the next tick's update.
- **"Dead for up to 1 second"**: A player can be destroyed between ticks and not know it for up to a full second (or 10 seconds under heavy TiDi).

---

## Time Dilation (TiDi)

### The Defining Innovation

Introduced by CCP Veritas in late 2011 (deployed January 2012), Time Dilation is arguably EVE's most important networking innovation and a concept rarely seen in other games. The core insight:

> When the server cannot complete a tick's work within one real-time second, **slow down the simulation clock** rather than dropping frames, desyncing clients, or crashing.

Before TiDi, large fleet fights simply broke. Commands would queue for minutes, modules would not activate, ships would warp into invisible enemies. The server was running at 1 Hz but trying to process 5 seconds of work per tick -- the excess was pure lag.

TiDi changes the contract: the simulation always stays consistent, but time itself stretches.

### How TiDi Works

```python
# Pseudo-code: Time Dilation calculation and application

class TimeDilationController:
    MIN_TIDI_FACTOR = 0.10   # Floor: 10% speed (1 sim-second = 10 real-seconds)
    MAX_TIDI_FACTOR = 1.00   # Normal: 100% speed (1 sim-second = 1 real-second)

    # Smoothing to avoid jarring speed changes
    RAMP_UP_RATE   = 0.05    # Increase by 5% per tick toward 1.0
    RAMP_DOWN_RATE = 0.20    # Decrease by 20% per tick when overloaded

    def __init__(self):
        self.tidi_factor = 1.0  # Current dilation factor

    def calculate_tidi(self, tick_elapsed_time, tick_budget):
        """
        After each tick, measure how long it took vs. the budget.
        Adjust the dilation factor so the next tick has enough time.
        """
        load_ratio = tick_elapsed_time / tick_budget

        if load_ratio > 1.0:
            # Tick took longer than budget -- slow down
            target_factor = self.tidi_factor / load_ratio
            self.tidi_factor -= self.RAMP_DOWN_RATE * (self.tidi_factor - target_factor)
        else:
            # Tick completed within budget -- ramp back toward 1.0
            self.tidi_factor += self.RAMP_UP_RATE * (self.MAX_TIDI_FACTOR - self.tidi_factor)

        # Clamp to allowed range
        self.tidi_factor = max(self.MIN_TIDI_FACTOR,
                               min(self.MAX_TIDI_FACTOR, self.tidi_factor))

        return self.tidi_factor

    def get_dilated_tick_interval(self):
        """
        The real-time duration of one simulation tick.
        At 100% TiDi: 1 second. At 10% TiDi: 10 seconds.
        """
        return 1.0 / self.tidi_factor


# === Applying TiDi to game mechanics ===

class ModuleCycle:
    """A ship module with a cycle time affected by TiDi."""

    def __init__(self, base_cycle_time):
        self.base_cycle_time = base_cycle_time  # e.g. 5.0 seconds

    def get_dilated_cycle_time(self, tidi_factor):
        """
        At 100% TiDi: 5s cycle = 5 real seconds.
        At  10% TiDi: 5s cycle = 50 real seconds.
        """
        return self.base_cycle_time / tidi_factor

    def get_remaining_ticks(self, elapsed_ticks, tidi_factor):
        """Module cycles are measured in simulation ticks, not real time."""
        total_ticks = self.base_cycle_time  # 1 tick = 1 sim-second
        return max(0, total_ticks - elapsed_ticks)
```

### What TiDi Affects (and Doesn't)

**Dilated (scales with the simulation clock):**
- Module cycle times (weapons, repairers, etc.)
- Ship movement and physics (velocity, alignment, warp)
- Drone behavior cycles
- Shield/armor/capacitor recharge
- Target locking time
- Bomb detonation timers

**Not dilated (tied to real-world time):**
- Reinforcement timers on structures (prevents manipulation)
- Skill training queue
- Industry job timers
- EVE Server Time (the universal clock displayed in-game)

### O(n^2) Scaling: The Fundamental Limit

The reason TiDi exists at all is that fleet fight load scales roughly as **O(n^2)** with the number of players on grid. When n players act, those actions must be communicated to n observers: n * n interactions. Drones compound this further -- each drone is an independent entity with its own decision-making, approach, orbit, and attack cycle.

In the infamous HED-GP battle (January 2014), with TiDi pegged at 10%, the Dogma effects system fell **193 simulation-seconds behind** -- meaning players waited **32 real-time minutes** for their module activations to resolve. The earlier 6VDT battle peaked at 42 simulation-seconds of Dogma lateness (7 real-time minutes at 10% TiDi). The difference: HED-GP had 38,852 unique drones deployed vs. 21,123 in 6VDT -- an 84% increase in entities that each require O(n) processing.

### TiDi in Practice: A Fleet Fight Timeline

A typical large engagement might look like:

| Phase | Players on Grid | TiDi Factor | Real-Time per Sim-Second |
|---|---|---|---|
| Pre-engagement | 200 | 100% | 1 second |
| Fleet warp-in | 1,600 | ~5% | ~20 seconds |
| Fight stabilizes | 1,600 | ~30% | ~3.3 seconds |
| Mid-battle (attrition) | 1,200 | ~60% | ~1.7 seconds |
| Retreat/extraction | 1,400 | ~10% | 10 seconds |
| Post-fight cleanup | 300 | 100% | 1 second |

---

## Movement and Combat

### Formula-Based Combat

EVE does not use spatial hitboxes, raycasts, or projectile physics for combat resolution. Instead, all combat is resolved through **mathematical formulas** evaluated on the server. This is why a 1 Hz tick rate is viable -- there is no need to detect fast-moving projectiles intersecting small collision volumes between frames.

### Turret Hit Chance Formula

The core turret tracking formula:

```
Chance to Hit = 0.5 ^ ( (Tracking Term)^2 + (Range Term)^2 )
```

Where:

```
Tracking Term = (Angular Velocity * 40000) / (Turret Tracking * Target Signature Radius)
Range Term    = max(0, Distance - Optimal Range) / Falloff
```

```python
# Pseudo-code: EVE Online turret hit chance calculation

import math
import random

def calculate_hit_chance(attacker, target, turret):
    """
    Compute chance to hit based on tracking, signature, range.
    All combat is resolved server-side with this formula.

    Parameters:
        attacker: attacking ship state
        target: target ship state
        turret: turret module attributes
    """
    # --- Tracking component ---
    # Angular velocity: how fast the target moves across the turret's FOV
    relative_velocity = target.velocity - attacker.velocity
    displacement = target.position - attacker.position
    distance = displacement.magnitude()

    # Transversal velocity (perpendicular to line of sight)
    radial_component = displacement.normalized().dot(relative_velocity)
    transversal_speed = math.sqrt(
        relative_velocity.magnitude_squared() - radial_component ** 2
    )

    # Angular velocity in radians/second
    angular_velocity = transversal_speed / max(distance, 1.0)

    tracking_term = (angular_velocity * 40000.0) / (
        turret.tracking_speed * target.signature_radius
    )

    # --- Range component ---
    range_term = max(0.0, distance - turret.optimal_range) / turret.falloff

    # --- Combined hit chance ---
    exponent = tracking_term ** 2 + range_term ** 2
    chance_to_hit = 0.5 ** exponent

    return chance_to_hit

def resolve_turret_volley(attacker, target, turret):
    """Resolve a turret shot. Called once per weapon cycle (server-side)."""
    hit_chance = calculate_hit_chance(attacker, target, turret)
    roll = random.random()

    if roll < hit_chance:
        # Hit quality determines damage multiplier
        # Perfect hit (roll = 0) does full damage; grazing hits do less
        quality = min(1.0, (hit_chance - roll) / hit_chance)

        # Damage also scales with signature: oversized guns vs small targets
        sig_factor = min(1.0, target.signature_radius / turret.signature_resolution)

        damage = turret.base_damage * quality * sig_factor
        apply_damage(target, damage, turret.damage_type)
    else:
        # Miss
        pass
```

Key insight: **doubling tracking speed, doubling target signature radius, and halving angular velocity all have exactly the same effect on hit chance.** This multiplicative relationship is what makes EVE's combat so analytically deep.

### Dead Reckoning Between 1 Hz Ticks

Since the server only sends position updates at 1 Hz, the client must predict entity movement between ticks. EVE uses **dead reckoning** on the client side -- the Destiny engine runs identically on both server and client, advancing entity positions using the same differential equations.

```python
# Pseudo-code: Dead reckoning on the EVE client between 1 Hz server ticks

class ClientDeadReckoning:
    """
    The client runs its own copy of the Destiny physics engine.
    Between server ticks, it extrapolates entity positions.
    On each server tick, it reconciles with authoritative state.
    """

    def __init__(self):
        self.entities = {}  # entity_id -> EntityState

    def on_server_tick(self, server_updates):
        """Called once per second (or per dilated tick) with authoritative state."""
        for entity_id, server_state in server_updates.items():
            local = self.entities.get(entity_id)
            if local is None:
                # New entity
                self.entities[entity_id] = EntityState(server_state)
                continue

            # Compare predicted position with server authority
            position_error = (server_state.position - local.position).magnitude()

            if position_error < SNAP_THRESHOLD:
                # Small error: blend toward server position over several frames
                local.correction_target = server_state.position
                local.correction_blend = 0.0
            else:
                # Large desync: snap immediately
                local.position = server_state.position

            # Always accept authoritative velocity and flight state
            local.velocity = server_state.velocity
            local.flight_mode = server_state.flight_mode  # orbit, approach, warp, etc.

    def update_frame(self, dt):
        """Called every render frame (e.g., 60 Hz) for smooth visual movement."""
        for entity in self.entities.values():
            # Dead-reckon: advance position using current velocity
            entity.position += entity.velocity * dt

            # Apply flight model (orbit curvature, approach deceleration, etc.)
            entity.velocity = apply_client_flight_model(entity, dt)

            # Blend correction from last server tick
            if entity.correction_target is not None:
                entity.correction_blend += dt * CORRECTION_RATE
                if entity.correction_blend >= 1.0:
                    entity.correction_target = None
                else:
                    entity.position = lerp(
                        entity.position,
                        entity.correction_target,
                        entity.correction_blend
                    )
```

The Destiny engine is designed as a deterministic differential equation solver: "it only knows the current state of objects and only needs this to calculate what their state will be in the next timestep." Identical code with identical data produces identical results on client and server. Historically, desync bugs arose from subtle differences -- such as collision detection iteration order differing between the server's full entity set and the client's visible subset (a bug identified by CCP and fixed in the "Facing Destiny" update).

### TiDi-Aware Client Interpolation

When TiDi is active, the client must adjust its interpolation to account for the dilated tick interval:

```python
# Pseudo-code: TiDi-aware client interpolation

class TiDiAwareInterpolation:
    """
    Under TiDi, server ticks arrive less frequently in real time.
    The client must stretch its interpolation accordingly.
    """

    def __init__(self):
        self.current_tidi = 1.0
        self.last_tick_time = time.monotonic()
        self.expected_tick_interval = 1.0  # seconds (real-time)

    def on_tidi_update(self, new_tidi_factor):
        """Server broadcasts current TiDi factor to all clients."""
        self.current_tidi = new_tidi_factor
        # At 10% TiDi, expect a tick every 10 real seconds
        self.expected_tick_interval = 1.0 / self.current_tidi

    def on_server_tick(self, tick_data):
        """Received authoritative tick update from server."""
        now = time.monotonic()
        actual_interval = now - self.last_tick_time
        self.last_tick_time = now

        # Use actual interval for more accurate interpolation
        self.expected_tick_interval = actual_interval

        # Apply authoritative state (see dead reckoning above)
        self.apply_server_state(tick_data)

    def get_interpolation_alpha(self):
        """
        How far through the current tick are we? (0.0 to 1.0)
        Used to interpolate between last known state and predicted state.
        """
        elapsed = time.monotonic() - self.last_tick_time
        return min(1.0, elapsed / self.expected_tick_interval)

    def render_entity(self, entity):
        """
        Render an entity using interpolation between tick states.
        Smooth even when ticks are 10 seconds apart under TiDi.
        """
        alpha = self.get_interpolation_alpha()

        # Interpolate position between last authoritative and dead-reckoned
        render_position = lerp(
            entity.last_server_position,
            entity.dead_reckoned_position,
            alpha
        )

        # Scale visual effects (engine trails, turret rotation) to TiDi
        visual_speed = entity.velocity.magnitude() * self.current_tidi
        render_engine_trail(entity, visual_speed)
        render_turret_tracking(entity, self.current_tidi)
```

---

## Stackless Python and the GIL

### The Core Bottleneck

EVE's entire game simulation -- Destiny physics, Dogma effects, market transactions, chat, session management -- runs as Stackless Python tasklets under a single GIL. Despite Stackless providing thousands of lightweight microthreads, they are cooperatively scheduled on **one OS thread**. On a 16-core server, 15 cores sit idle per sol node.

This is the architectural constraint that has driven every major networking evolution:

```
+------------------------------------------------------------------+
|                    ORIGINAL ARCHITECTURE (~2003-2010)             |
|                                                                  |
|  +------------------------------------------------------------+  |
|  |                    GIL (Global Interpreter Lock)            |  |
|  |                                                            |  |
|  |  +----------+  +---------+  +----------+  +------------+  |  |
|  |  | Destiny  |  | Dogma   |  | MachoNet |  | StacklessIO|  |  |
|  |  | (Physics)|  | (Effects)|  | (Routing)|  | (Network)  |  |  |
|  |  +----------+  +---------+  +----------+  +------------+  |  |
|  |                                                            |  |
|  |  All systems share the GIL. Only one tasklet runs at a    |  |
|  |  time. Networking I/O blocks the physics simulation.       |  |
|  +------------------------------------------------------------+  |
|                                                                  |
|  CPU cores:  [BUSY] [idle] [idle] [idle] [idle] [idle] [idle]    |
+------------------------------------------------------------------+
```

### CarbonIO / BlueNet (2011)

CarbonIO was a complete rewrite of the StacklessIO networking layer, with one goal: **move network I/O off the GIL**.

```
+------------------------------------------------------------------+
|                    CARBONIO / BLUENET (2011+)                     |
|                                                                  |
|  +----------------------------+  +-----------------------------+  |
|  |         GIL Thread         |  |    Off-GIL Worker Threads   |  |
|  |                            |  |                             |  |
|  | +----------+ +-----------+ |  | +-------------------------+ |  |
|  | | Destiny  | | Dogma     | |  | | CarbonIO               | |  |
|  | | (Physics)| | (Effects) | |  | |  - WSARecv() on connect| |  |
|  | +----------+ +-----------+ |  | |  - Decrypt/decompress  | |  |
|  | | MachoNet |               |  | |  - Parse packets        | |  |
|  | | (Routing)|               |  | |  - Queue for GIL or..  | |  |
|  | +----------+               |  | +-------------------------+ |  |
|  +----------------------------+  | +-------------------------+ |  |
|                                  | | BlueNet                 | |  |
|    GIL still serializes game     | |  - Route by 8-10 byte  | |  |
|    logic, but networking I/O     | |    out-of-band header   | |  |
|    no longer competes for it.    | |  - Deliver to C++ apps  | |  |
|                                  | |    WITHOUT acquiring GIL| |  |
|                                  | |  - Physics data stays   | |  |
|                                  | |    in native C++ form   | |  |
|                                  | +-------------------------+ |  |
|                                  +-----------------------------+  |
|                                                                  |
|  CPU cores:  [GIL-Game] [CarbonIO] [BlueNet] [idle] [idle]      |
+------------------------------------------------------------------+
```

Key innovations of CarbonIO:
- **Parallel recv**: Initiates `WSARecv()` immediately on connection, processes decryption and decompression on worker threads without the GIL.
- **Read-ahead**: Can receive and parse packets without Python asking it to, concurrently.
- **Work grouping**: Batches multiple GIL acquisition requests so the overhead of lock acquire/release is amortized.
- **SSL/compression off-GIL**: OpenSSL and zlib/snappy run on completion port threads.

BlueNet extended this further:
- Maintains **read-only copies** of MachoNet routing structures.
- Prepends small 8-10 byte out-of-band headers to packets containing routing information.
- Routes packets by inspecting these headers **without ever acquiring the GIL or invoking Python**.
- C++ systems (like Destiny physics) receive their data directly in native form -- no Python pickling/unpickling.

In a 24-hour field test with ~1,600 peak users, CarbonIO showed significant CPU reduction both overall and per-user. Sol nodes showed 8-10% improvement even though they do comparatively little network I/O. CCP described the results as "somewhere between unbelievably stunning and impossibly amazing."

### Quasar / gRPC (2021+)

```
+------------------------------------------------------------------+
|                    QUASAR ARCHITECTURE (2021+)                   |
|                                                                  |
|  +---------------------------+   +-----------------------------+  |
|  |     TRANQUILITY CORE      |   |    QUASAR ECOSYSTEM        |  |
|  |                           |   |    (Cloud-Native)           |  |
|  |  +--------+ +----------+  |   |                             |  |
|  |  |Destiny | | Dogma    |  |   |  +--------+  +-----------+ |  |
|  |  |        | |          |  |   |  | Skill  |  | Activity  | |  |
|  |  +--------+ +----------+  |   |  | Plans  |  | Tracker   | |  |
|  |  | MachoNet / BlueNet  |  |   |  +---+----+  +-----+-----+ |  |
|  |  +----------+----------+  |   |      |              |       |  |
|  +-------------|-------------+   |  +---v--------------v---+   |  |
|                |                 |  |   gRPC + ProtoBuf     |   |  |
|                |                 |  |   (fast serialization)|   |  |
|       +--------v--------+       |  +-----------+-----------+   |  |
|       |  EVE Client     |       |              |               |  |
|       | (Desktop)       +-------|---> RabbitMQ (message bus)   |  |
|       +-----------------+       |              |               |  |
|                                 |  +-----------v-----------+   |  |
|                                 |  | Kubernetes / Go       |   |  |
|                                 |  | Microservices         |   |  |
|                                 |  | (own databases, no    |   |  |
|                                 |  |  Tranquility DB dep.) |   |  |
|                                 |  +------------------------+   |  |
|                                 +-----------------------------+  |
+------------------------------------------------------------------+
```

Quasar, originating as "Project Sanguine" (2016), represents the latest evolution. It introduces a **message-bus paradigm** using:

- **gRPC**: For building APIs between services, providing fast serialization via protocol buffers.
- **RabbitMQ**: For asynchronous message routing between microservices.
- **Kubernetes + Go**: Cloud-native orchestration for services that handle the "message firehose" from the simulation.

The key architectural shift: features like Skill Plans, Activity Tracker, and Abyssal Proving Grounds leaderboards now bypass Tranquility's database entirely. The EVE client communicates directly via gRPC to dedicated microservices with their own databases. This means:

- **All serialization, transmission, and message routing happens outside the GIL** except for the initial memory copy.
- **No downtime required** to update Quasar-based services -- no Tranquility restart needed.
- **Fault isolation**: When Quasar services degrade, the core simulation is unaffected (and vice versa).

---

## Network Evolution

### Timeline

| Year | Technology | Impact |
|---|---|---|
| **2003** | **StacklessIO + MachoNet** | Original networking layer. All I/O under the GIL. MachoNet handles marshaling, routing, sessions, encryption. Good enough for small fleet sizes. |
| **2006** | **GDC: "Server Technology of EVE Online"** | CCP presents at GDC, describing the single-shard architecture, Stackless Python, and cluster technology to the industry. |
| **2009** | **Destiny rewrite ("Facing Destiny")** | Fixes long-standing desync bugs in the physics engine caused by collision detection iteration order differences between client and server. |
| **2010** | **"Drakes of Destiny" optimization** | Team Gridlock refactors the Destiny pre-tick from O(n*m) to O(n+m) by pre-serializing updates once per bubble instead of once per observer. Dramatically reduces CPU cost of fleet fights. |
| **2011** | **CarbonIO + BlueNet** | Networking I/O moves off the GIL. C++ systems receive packets directly without Python involvement. Enables the server headroom that makes TiDi practical. |
| **2012** | **Time Dilation (TiDi) deployed** | Activated January 18, 2012. Transforms fleet fights from broken lag-fests into slow but playable engagements. Enabled by the CarbonIO headroom. |
| **2014** | **HED-GP battle retrospective** | Exposes Dogma lateness as the remaining bottleneck. 193 sim-seconds behind = 32 real minutes of module lag at 10% TiDi. Drones identified as primary scaling problem. |
| **2016** | **Project Sanguine / ESI** | Message-bus paradigm introduced for out-of-game services (ESI, EVE Portal). Seeds the Quasar architecture. |
| **2019** | **Aether Wars tech demo (with Hadean)** | Experimental test of spatially-distributed compute. 3,852 human players + 10,422 AI in one battle. Not deployed to production EVE, but validates future directions. |
| **2021** | **Quasar + gRPC deployed** | New features built on gRPC microservices outside Tranquility. Skill Plans, Activity Tracker operate independently with own databases. No downtime deployments. |
| **2022** | **GDC: "Quasar, Brightest in the Galaxy"** | CCP presents the Quasar architecture and gRPC integration to the industry at GDC. |

---

## The Dogma Effects System

### The Server-Side Bottleneck

While Destiny (physics) was the primary CPU consumer in early EVE, optimizations like the Drakes of Destiny refactor shifted the bottleneck to **Dogma** -- the system that manages all module effects, skill bonuses, and attribute modifiers.

### How Dogma Works

Every item in EVE is defined by **attributes** (numbers like `maxVelocity`, `shieldCapacity`, `trackingSpeed`) and **effects** (operations that modify attributes on the item itself or on other items). When a player activates a module, Dogma must:

1. Evaluate the module's effects (which may reference other effects).
2. Apply modifiers to target attributes (using operations like PostPercent, PostDiv, etc.).
3. Chain through "spliced" effects that link multiple modifiers.
4. Recalculate all dependent attributes (a change to `maxVelocity` may cascade to navigation, orbit calculations, etc.).

```python
# Pseudo-code: Dogma effects chain processing

class DogmaEngine:
    """
    Server-side effects processor. Handles module activation,
    skill bonuses, and all attribute modifications.
    """

    def __init__(self):
        self.effect_queue = []       # Pending module activations/deactivations
        self.active_effects = {}     # Currently active effects per entity
        self.attribute_cache = {}    # Cached computed attributes

    def activate_module(self, ship, module, target=None):
        """Called when a player activates a module (e.g., a Stasis Webifier)."""
        for effect in module.effects:
            self.effect_queue.append(EffectEvent(
                type='activate',
                source_item=module,
                source_ship=ship,
                target=target,
                effect=effect
            ))

    def process_effect_queue(self):
        """
        Process all pending effects. This runs once per tick.
        THIS IS THE FLEET-FIGHT BOTTLENECK.

        With 2000 ships each running 6-8 modules, and each module
        having 1-11 effects with chained modifiers, this loop can
        process tens of thousands of effect applications per tick.
        """
        # Deduplicate queue (fix for the duplicate effects bug - June 2010)
        self.effect_queue = deduplicate(self.effect_queue)

        for event in self.effect_queue:
            try:
                self._apply_effect(event)
            except EffectAlreadyApplied:
                # Previously this caused the loop to yield early,
                # starving the entire module system.
                # Fix: log and continue, don't yield.
                log_warning(f"Duplicate effect skipped: {event}")
                continue

        self.effect_queue.clear()

    def _apply_effect(self, event):
        """Apply a single effect with all its modifiers."""
        effect = event.effect

        for modifier in effect.modifiers:
            # Resolve source attribute value (from the module/charge)
            source_value = self._get_attribute(
                event.source_item, modifier.source_attribute_id
            )

            # Resolve target (could be self, current target, gang, etc.)
            target_items = self._resolve_target(event, modifier)

            for target_item in target_items:
                # Apply the modifier operation
                current_value = self._get_attribute(
                    target_item, modifier.target_attribute_id
                )

                new_value = self._apply_operation(
                    current_value, source_value, modifier.operation
                )

                self._set_attribute(
                    target_item, modifier.target_attribute_id, new_value
                )

                # Invalidate dependent attribute caches
                self._invalidate_dependents(
                    target_item, modifier.target_attribute_id
                )

    def _apply_operation(self, current, source, operation):
        """Apply a Dogma modifier operation."""
        if operation == 'PreAssign':
            return source
        elif operation == 'PreMul':
            return current * source
        elif operation == 'PreDiv':
            return current / source
        elif operation == 'ModAdd':
            return current + source
        elif operation == 'ModSub':
            return current - source
        elif operation == 'PostMul':
            return current * source
        elif operation == 'PostDiv':
            return current / source
        elif operation == 'PostPercent':
            return current * (1.0 + source / 100.0)
        elif operation == 'PostAssign':
            return source
        else:
            raise ValueError(f"Unknown operation: {operation}")

    def _resolve_target(self, event, modifier):
        """
        Resolve which items the modifier applies to.
        Can be complex -- e.g., 'all items on my ship requiring
        skill X' or 'all fleet members within range'.
        """
        domain = modifier.domain
        if domain == 'self':
            return [event.source_item]
        elif domain == 'ship':
            return [event.source_ship]
        elif domain == 'target':
            return [event.target] if event.target else []
        elif domain == 'gang':
            return get_fleet_members_in_range(event.source_ship)
        elif domain == 'locationGroup':
            # All items on ship matching a specific group
            return [item for item in event.source_ship.fitted_items
                    if item.group_id == modifier.filter_group_id]
        elif domain == 'locationRequiredSkill':
            # All items on ship requiring a specific skill
            return [item for item in event.source_ship.fitted_items
                    if modifier.filter_skill_id in item.required_skills]
```

### Why Dogma is the Bottleneck

The core problem is **combinatorial explosion in large fights**:

- A fleet of 2,000 ships, each with ~8 active modules.
- Each module has 1-11 effects (a Damage Control module alone has 12 modifiers via spliced expressions).
- Each effect may target multiple items.
- Every attribute change invalidates cached values, triggering recalculations.
- Drones add entities with their own effects and decision-making (38,852 drones in HED-GP).

The Dogma processing loop runs under the GIL as Python code. It cannot be trivially parallelized because effects chain and depend on each other. The "Dogma lateness" metric measures how far behind the effect queue falls -- in extreme cases, minutes of real time behind the simulation clock.

A critical 2010 bug exacerbated this: duplicate effect entries in the stop/repeat manager caused error handling to yield the processing loop early (cooperative multitasking), starving the module system. The error occurred 1.5 million times in June 2010. CCP's fix was three-layered: deduplicate the queue, continue processing on errors instead of yielding, and fix the entity docility code that incorrectly called repeat instead of stop.

---

## Key Sources

### GDC Presentations
- [GDC 2006: "The Server Technology of EVE Online: How to Cope with 300,000 Players in One World"](https://gdcvault.com/play/109/The-Server-Technology-of-EVE) -- Kristjan Valur Jonsson (CCP). Foundational talk on single-shard architecture, Stackless Python, and cluster technology.
- [GDC 2008: "The Server Technology of EVE Online" (updated)](https://www.gdcvault.com/play/1014031/The-Server-Technology-of-EVE) -- Updated version covering growth to 300,000+ subscribers.
- [GDC 2013: "Multitasking with Coroutines"](https://www.gdcvault.com/play/1020578/Multitasking-with) -- Stackless Python coroutine patterns in game engines.
- [GDC 2022: "Quasar, Brightest in the Galaxy: Expanding EVE Online's Server Potential with gRPC"](https://gdcvault.com/play/1027755/Online-Game-Technology-Summit-Quasar) -- The Quasar/gRPC architecture and its impact on fleet fight scalability.

### CCP Dev Blogs
- ["Introducing Time Dilation (TiDi)"](https://www.eveonline.com/news/view/introducing-time-dilation-tidi) (2011) -- CCP Veritas's original TiDi announcement.
- ["Time Dilation -- How's That Going?"](https://www.eveonline.com/news/view/time-dilation-hows-that-going) (2012) -- Post-deployment TiDi analysis.
- ["CarbonIO and BlueNet: Next Level Network Technology"](https://www.eveonline.com/news/view/carbonio-and-bluenet-next-level-network-technology-1) (2011) -- Technical deep-dive on moving networking off the GIL.
- ["Fixing Lag: Drakes of Destiny (Part 1)"](https://www.eveonline.com/news/view/fixing-lag-drakes-of-destiny-part-1-1) (2010) -- Destiny engine profiling and the O(n*m) pre-tick bottleneck.
- ["Fixing Lag: Drakes of Destiny (Part 2)"](https://www.eveonline.com/news/view/fixing-lag-drakes-of-destiny-part-2-1) (2010) -- The O(n+m) refactor and serialization optimization.
- ["Fixing Lag: Module Lag"](https://www.eveonline.com/news/view/fixing-lag-module-lag-why-not-all-bugfixes-are-a-good-idea) (2010) -- Dogma duplicate effects bug and the stop/repeat manager.
- ["Facing Destiny"](https://www.eveonline.com/news/view/facing-destiny) (2014) -- Destiny desync investigation (collision iteration order).
- ["HED-GP Technical Retrospective"](https://www.eveonline.com/news/view/what-a-hed-ache) (2014) -- Post-mortem on HED-GP battle performance, Dogma lateness, drone overhead.
- ["Introducing Quasar"](https://www.eveonline.com/news/view/introducing-quasar) (2021) -- Quasar/gRPC architecture announcement.

### Community and Reference
- [EVE University Wiki: Server Tick](https://wiki.eveuniversity.org/Server_tick) -- Comprehensive reference on the 1 Hz tick and its gameplay implications.
- [EVE University Wiki: Time Dilation](https://wiki.eveuniversity.org/Time_dilation) -- TiDi mechanics reference including the O(n^2) scaling note.
- [EVE University Wiki: Turret Mechanics](https://wiki.eveuniversity.org/Turret_mechanics) -- Complete turret hit chance formula and tracking mechanics.
- [High Scalability: EVE Online Architecture](https://highscalability.com/eve-online-architecture/) -- External analysis of the cluster architecture.
- [Gamasutra: "Infinite Space: An Argument for Single-Sharded Architecture in MMOs"](https://www.gamedeveloper.com/design/infinite-space-an-argument-for-single-sharded-architecture-in-mmos) -- CCP's design philosophy on single-shard.
- [ESI Docs: Dogma](https://docs.esi.evetech.net/docs/dogma.html) -- Technical reference for the Dogma effects/modifier system.
- [Nosy Gamer: "Quasar: Updating EVE Online's Network Layer"](https://nosygamer.blogspot.com/2021/10/quasar-updating-eve-onlines-network.html) -- Timeline analysis from StacklessIO through Quasar.
