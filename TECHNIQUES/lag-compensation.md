# Lag Compensation (Server-Side Rewinding)


---

## Table of Contents

1. [Overview](#overview)
2. [History and Origins](#history-and-origins)
3. [The Problem: Why Naive Hit Detection Fails](#the-problem-why-naive-hit-detection-fails)
4. [How It Works Step by Step](#how-it-works-step-by-step)
5. [Hitbox History Buffer](#hitbox-history-buffer)
6. [The Rewinding Algorithm](#the-rewinding-algorithm)
7. [Favor-the-Shooter vs Favor-the-Target](#favor-the-shooter-vs-favor-the-target)
8. [Peeker's Advantage](#peekers-advantage)
9. [Shot Behind Walls Problem](#shot-behind-walls-problem)
10. [Sub-Tick Lag Compensation](#sub-tick-lag-compensation)
11. [Integration with Other Techniques](#integration-with-other-techniques)
12. [Pseudo-Code and Code Examples](#pseudo-code-and-code-examples)
13. [Anti-Cheat Considerations](#anti-cheat-considerations)
14. [Game Implementations](#game-implementations)
15. [When to Use / When Not to Use](#when-to-use--when-not-to-use)
16. [References](#references)

---

## Overview

**Lag compensation** (also called **server-side rewinding** or **server rewind**) is a server-authoritative technique that allows the game server to evaluate combat actions -- particularly hitscan weapon fire -- from the perspective of what the attacking player *actually saw* at the moment they pulled the trigger, rather than where targets currently are on the server at the time the shot message arrives.

In any networked game, information takes time to travel between the client and the server. When a player fires at an enemy, the server does not receive that input until some tens of milliseconds later. By that time, the target has moved. Without lag compensation, even perfectly aimed shots would routinely miss because the server evaluates them against the *current* world state rather than the state the shooter perceived.

Lag compensation solves this by having the server maintain a **history buffer** of past entity states (positions, rotations, hitboxes, animations). When a shot arrives, the server calculates what the shooter's view of the world looked like at the time the shot was fired, temporarily **rewinds** all relevant entities to that historical state, performs the hit test, and then **restores** the current state. The entire operation completes in microseconds and is invisible to other players' simulations.

**Key properties:**
- Server-authoritative: the server performs all hit validation
- Backward-looking: rewinding time rather than predicting the future
- Transparent to the shooter: players aim at what they see with no need to "lead" targets
- Complementary: works alongside client-side prediction and entity interpolation

---

## History and Origins

### Half-Life and GoldSrc (1998-2000)

Lag compensation emerged from Valve Software's work on Half-Life's multiplayer networking. When Half-Life shipped in 1998, most FPS games either used pure client-side hit detection (trusting the client, easily exploitable) or pure server-side hit detection with no compensation (accurate shots missed due to latency).

Yahn Bernier, a Valve engineer, authored the foundational paper **"Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization"** (2001), which formalized the technique alongside client-side prediction and entity interpolation. The paper described these as the three pillars of responsive networked gameplay.

### Counter-Strike 1.6 and Source Engine (2000-2004)

Counter-Strike, originally a Half-Life mod, became the proving ground for lag compensation. The competitive nature of the game -- where a single headshot could end a round -- made precise hit registration critical. Valve's Source Engine (2004) shipped with a mature lag compensation system that became the industry reference implementation.

The Source Engine implementation introduced key concepts still used today:
- **`sv_maxunlag`**: a server ConVar capping the maximum rewind window (default: 1.0 second)
- **`cl_interp`**: client-side interpolation period, factored into the rewind calculation
- **`StartLagCompensation()` / `FinishLagCompensation()`**: the API brackets for rewinding and restoring entity state

### Industry Adoption (2005-present)

Following Valve's work, lag compensation became standard in competitive FPS games:
- **Halo 2/3** (Bungie, 2004-2007): early console implementations
- **Battlefield series** (DICE): server-side rewind with configurable limits
- **Overwatch** (Blizzard, 2016): "favor-the-shooter" with a 250ms cap
- **Valorant** (Riot Games, 2020): 128-tick servers with sub-frame precision
- **Counter-Strike 2** (Valve, 2023): sub-tick system replacing traditional tick-aligned compensation

---

## The Problem: Why Naive Hit Detection Fails

Consider a scenario with two players -- the **Shooter** (Player A) and the **Target** (Player B) -- connected to a server. Player A has 60ms round-trip latency; Player B has 40ms round-trip latency.

### Timeline Without Lag Compensation

```
Time (ms)    Shooter (A)              Server                   Target (B)
─────────────────────────────────────────────────────────────────────────────
   0         Sees B at position X     B is at position X       At position X
  30         (A sees B at X due       B starts moving right    Starts moving
             to 30ms one-way delay)                            right
  60         A fires at position X    B is now at X+60         B is at X+60
             (B appears at X on       (B moved during A's      (moved for 30ms
             A's screen)              latency window)          of server time)
  90         Shot arrives at server   B is at X+90             B is at X+70
             Server checks: is B
             at position X? NO!
             → MISS
─────────────────────────────────────────────────────────────────────────────
Result: Shot misses despite perfect aim on the shooter's screen.
```

The shooter aimed precisely at where the target appeared on their screen. The shot was pixel-perfect. But by the time the server received the shot, the target had moved. Without lag compensation, **the faster the target moves and the higher the shooter's latency, the more shots will "miss" despite perfect aim.**

### Timeline With Lag Compensation

```
Time (ms)    Shooter (A)              Server                   Target (B)
─────────────────────────────────────────────────────────────────────────────
   0         Sees B at position X     B is at position X       At position X
  30         (A sees B at X)          B moving right           Moving right
  60         A fires at position X    B is at X+60             B is at X+60
  90         Shot arrives at server   Server calculates:       B is at X+70
             with timestamp T=60      "A fired at T=60"
                                      Rewinds B to position
                                      at T=60 → B was at X
                                      Raycast: hit at X? YES!
                                      → HIT registered
                                      Restores B to X+90
─────────────────────────────────────────────────────────────────────────────
Result: Hit registers -- what the shooter saw is what the shooter gets.
```

---

## How It Works Step by Step

The lag compensation process follows a precise sequence on the server for every combat action:

### Step 1: Client Fires and Timestamps the Action

When the player presses the fire button, the client records:
- The **current client time** (derived from the server simulation tick the client is currently rendering)
- The **shot origin** (weapon muzzle position)
- The **shot direction** (aim vector)
- Any relevant weapon state (spread, recoil pattern index)

This information is bundled with the player's regular input packet (user command) and sent to the server.

### Step 2: Server Receives the Command

The server receives the player's input after a one-way network delay. The server now has:
- The player's firing command
- The client's timestamp (or the server can compute it)

### Step 3: Server Calculates the Client's Perceived Time

The server computes what time the world looked like from the shooter's perspective:

```
command_execution_time = current_server_time - packet_latency - client_interpolation_delay
```

Where:
- **`current_server_time`**: the authoritative server clock
- **`packet_latency`**: one-way latency from client to server (typically RTT / 2)
- **`client_interpolation_delay`**: the client's entity interpolation buffer (e.g., `cl_interp` in Source Engine, typically 2 ticks worth = ~31ms at 64 tick)

This formula accounts for the fact that the client renders other entities in the *past* due to entity interpolation. The shooter was not seeing the target at the server's "current time minus latency" but at an even earlier point because of the interpolation buffer.

### Step 4: Server Rewinds All Relevant Entities

Using `StartLagCompensation()` (or equivalent), the server:

1. Iterates through all entities that could be hit (typically all other players)
2. Searches the history buffer for the snapshot closest to `command_execution_time`
3. If the exact timestamp is not found, **interpolates** between the two nearest snapshots
4. Temporarily moves each entity's hitboxes to the computed historical position

This is a very fast operation -- no physics simulation is re-run. Only positions and collision geometry are moved.

### Step 5: Server Performs Hit Detection

With all entities rewound to the shooter's perceived time, the server performs the actual hit test:
- **Hitscan weapons**: a raycast from the shooter's position along their aim direction
- **Melee attacks**: an overlap test in front of the attacker
- **Projectile origins**: spawn the projectile at the correct historical position (the projectile itself then simulates forward normally)

If the raycast intersects an enemy hitbox, the hit is registered.

### Step 6: Server Restores Current State

Using `FinishLagCompensation()`, the server restores all entities to their actual current positions. The rewind was purely ephemeral -- it never affected the authoritative simulation or other players' views.

### Step 7: Server Applies Results

If a hit was detected, the server applies damage, triggers kill events, and sends the result to all clients. The shooter gets hit confirmation; the target gets a damage notification.

---

## Hitbox History Buffer

The history buffer is the core data structure that makes lag compensation possible. It stores a rolling window of past entity states that the server can query when rewinding.

### What Gets Stored Per Snapshot

Each entry in the history buffer (one per entity per tick) typically contains:

| Field | Description |
|-------|-------------|
| `timestamp` | Server tick number or time when this state was captured |
| `position` | Entity world position (Vector3) |
| `rotation` | Entity orientation (Quaternion or Euler) |
| `hitboxPositions[]` | Array of per-bone hitbox transforms (position + rotation for each bone/hitbox) |
| `hitboxExtents[]` | Bounding box half-extents or sphere radii for each hitbox |
| `animationState` | Crouch height, lean, animation frame (affects hitbox positions) |
| `velocity` | Used for interpolation and dead reckoning between snapshots |

### Ring Buffer Design

The history buffer is typically implemented as a **ring buffer** (circular buffer) for each entity. This provides O(1) insertion and bounded memory usage.

```
Ring Buffer (capacity = N snapshots):

  write index
       ↓
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ T7 │ T8 │ T9 │    │ T3 │ T4 │ T5 │ T6 │
└────┴────┴────┴────┴────┴────┴────┴────┘
  0    1    2    3    4    5    6    7

oldest ──────────────────────────→ newest
(T3 is oldest, T9 is newest; slot 3 is next to be overwritten)
```

**Sizing the buffer:**
- At **64 tick**, each tick is ~15.6ms. To store 1 second of history: 64 entries per entity.
- At **128 tick**, each tick is ~7.8ms. To store 1 second of history: 128 entries per entity.
- Typical buffer covers 0.5 to 1.0 second (controlled by `sv_maxunlag` in Source Engine).

**Memory considerations:**
- Per-snapshot size depends on hitbox count. A player model with 19 hitboxes, each needing a transform (position + rotation = 28 bytes), plus entity-level data, might total ~600-800 bytes per snapshot.
- At 64 tick with 64 entries and 10 players: ~500 KB total. This is very manageable.

### Timestamp Management

Each snapshot is tagged with the server tick number at which it was captured. When the server needs to rewind to a specific time:

1. Convert the requested time to the nearest tick number
2. Binary search (or linear scan for small buffers) the ring buffer for the two entries bracketing that tick
3. Compute an interpolation factor between the two snapshots

Because the ring buffer is ordered by time (oldest entries wrap around), the search is straightforward.

---

## The Rewinding Algorithm

### Finding the Right Snapshot

Given a `targetTime`, the server needs to find the two snapshots `S_before` and `S_after` such that:

```
S_before.timestamp <= targetTime <= S_after.timestamp
```

If `targetTime` exactly matches a snapshot, no interpolation is needed. Otherwise, the server interpolates.

### Interpolating Between Snapshots

For each entity, the interpolation factor `alpha` is:

```
alpha = (targetTime - S_before.timestamp) / (S_after.timestamp - S_before.timestamp)
```

Then each hitbox position is computed as:

```
rewindPosition = lerp(S_before.position, S_after.position, alpha)
rewindRotation = slerp(S_before.rotation, S_after.rotation, alpha)
```

For non-rotational hitbox data (like bounding box extents), linear interpolation suffices. For rotations, spherical linear interpolation (slerp) is more correct but often unnecessary when hitboxes are axis-aligned bounding boxes.

### Performing the Hit Test

With all entities at their rewound positions, the server performs a standard raycast or overlap test. This uses the same physics/collision code as real-time hit detection -- the only difference is that the world has been temporarily moved backward.

### Edge Cases

- **Target time older than buffer**: The server cannot rewind further than the oldest snapshot. The shot is evaluated at the oldest available state, or rejected entirely. This is where `sv_maxunlag` acts as a safety net.
- **Target time in the future**: Invalid -- clamped to the current time. Indicates a misbehaving client.
- **Shooter and target are the same entity**: Skip self -- the shooter is never rewound against themselves.

### Selective Rewinding

In practice, not every entity needs to be fully rewound:

1. **Broad-phase culling**: Only rewind entities within a reasonable distance of the shot (e.g., within weapon range)
2. **Bounding sphere pre-test**: Check if the ray passes near the entity's broad-phase bounding sphere before doing fine-grained hitbox interpolation
3. **Animation reconstruction**: Valorant optimizes by performing a sphere cast against bounding boxes first, then only reconstructing full skeletal animation for entities the ray might actually hit. Riot reported this reduced per-frame animation processing from ~0.1ms per player to ~0.3ms total.

---

## Favor-the-Shooter vs Favor-the-Target

Lag compensation inherently involves a design decision: when the shooter's view of the world disagrees with the target's view, whose perspective does the server honor?

### Favor-the-Shooter (Most Common)

**Philosophy:** "If you aimed at the target on your screen and your aim was true, you should get the hit."

The server rewinds to the shooter's perceived time and validates the shot from the shooter's perspective. This means:

- **For the shooter**: The game feels responsive and fair. Aim at what you see, hit what you aim at.
- **For the target**: Occasionally die after reaching cover, because the shooter saw you exposed and fired before you moved.

**Games using this approach:**
- Counter-Strike (all versions)
- Overwatch (with a 250ms cap)
- Valorant
- Halo Infinite
- Battlefield series
- Hunt: Showdown

### Favor-the-Target (Rare)

**Philosophy:** "If you are behind cover on your screen, you should not be hit."

The server evaluates shots against the *current* (or the target's perceived) entity positions, requiring the shooter to lead their targets based on latency.

- **For the shooter**: Must predict where the target will be, not where they are. Extremely frustrating at higher latencies.
- **For the target**: Feels safe behind cover. Defensive play is reliable.

Almost no modern competitive game uses pure favor-the-target, because:
- Hitscan weapons (bullets arrive instantly) are not reactably dodgeable -- the defender cannot meaningfully use the advantage
- It punishes the shooter far more than it benefits the target
- Leading targets by an unknown, fluctuating latency value is an unreasonable ask

### Hybrid Approaches

Some games use conditional systems:

| Approach | Description | Example |
|----------|-------------|---------|
| **Capped rewind** | Favor the shooter up to a maximum rewind limit (e.g., 250ms); beyond that, favor the target | Overwatch, Battlefield 4 |
| **Weighted blend** | Blend between shooter and target perspectives based on relative latency | Research prototypes |
| **Action-dependent** | Favor the shooter for hitscan; use server-current for projectiles and abilities | Overwatch (projectiles simulate forward from spawn, not rewound) |
| **Advanced Lag Compensation (ALC)** | Academic research combining perspectives to reduce "shot behind cover" incidents by up to 94% while preserving shooter accuracy | [Enhancing the experience of multiplayer shooter games via advanced lag compensation](https://dl.acm.org/doi/10.1145/3204949.3204971) |

### The Psychology

From Halo Infinite's engineering team (Richard Watson, Lead Engineer): *"Whatever happened on the shooter's screen the server endeavors to honor."*

The industry consensus favors the shooter because:
1. Missing a clearly aimed shot is more frustrating than occasionally dying behind cover
2. The shooter actively performed a skill-based action (aiming); the target's perception of safety is passive
3. At typical competitive latencies (20-60ms), the "shot behind cover" effect is barely perceptible

---

## Peeker's Advantage

### What It Is

**Peeker's advantage** is an inherent asymmetry in networked games where the player who initiates movement around a corner (the "peeker") has a timing advantage over the player holding an angle (the "holder"). The peeker sees the holder before the holder sees the peeker, due to the accumulated latency in the network pipeline.

This is not a bug -- it is an unavoidable consequence of the speed of light (and network routing) being finite. Lag compensation makes it *fair* for the shooter but does not eliminate the underlying timing asymmetry.

### Why It Exists

The peeker's movement is processed through this pipeline:

```
Peeker presses move → Client predicts instantly (0ms) →
  Packet travels to server (peeker's latency / 2) →
    Server processes and broadcasts →
      Packet travels to holder (holder's latency / 2) →
        Holder's client interpolation buffer adds delay →
          Holder finally SEES the peeker
```

Meanwhile, the peeker already saw the holder on their screen (the holder was standing still, so their position is stable and well-known). The peeker can aim and fire before the holder even knows the peeker has appeared.

### Valorant's Peeker's Advantage Formula

Riot Games published a detailed analysis of peeker's advantage in Valorant. The core equation:

```
EffectiveReactionTime(Holder) = ReactionTime(Peeker) - RTT(Holder) - Buffering
```

With realistic parameters at Valorant's launch:
- Server tick rate: 128 Hz (7.8ms per frame)
- Server buffering: ~2 server frames (~15.6ms)
- Client buffering: ~3 client frames at 60 FPS (~50ms)
- Network round-trip: 35ms (target for 70% of players)
- Client rendering delay: ~0.5 frames

**Result: peekers have approximately 141ms longer to react than holders.**

### Valorant's Mitigation

Through infrastructure and optimization investments, Riot reduced this to approximately **98ms** (a 28% improvement):

| Optimization | Savings |
|-------------|---------|
| 128-tick servers (vs 64-tick) | ~7.8ms |
| Riot Direct backbone (lower routing latency) | ~10-15ms |
| Minimal client/server buffering | ~15-20ms |
| Network interpolation delay of 7.8ms | Significant vs typical 30-50ms |

**Additional game design mitigations:**
- Map layouts structurally favor defenders (multiple angles, elevation advantages)
- Character shoulders appear before the head when peeking, giving the holder a visual cue
- Movement inaccuracy while shooting penalizes run-and-gun peeking
- Hit tagging (being shot briefly slows you) punishes aggressive peeks

### Peeker's Advantage at Different Latencies

```
Peeker RTT    Holder RTT    Approximate Advantage
──────────    ──────────    ─────────────────────
  20ms          20ms          ~60-80ms
  40ms          40ms          ~80-120ms
  80ms          40ms          ~120-160ms
  80ms          80ms          ~140-200ms
 150ms         150ms          ~250-350ms
```

The advantage scales with combined latency. This is why competitive games invest heavily in minimizing ping.

---

## Shot Behind Walls Problem

### What the Target Experiences

The "shot behind walls" (or "shot behind cover" / "dying around corners") problem is the most visible side effect of lag compensation. From the target's perspective:

1. They are engaged in a firefight
2. They break line of sight by moving behind a wall or ducking behind cover
3. They believe they are safe
4. They receive damage and die, seemingly through solid geometry

### Why It Happens

```
Time    Shooter's Screen         Server                    Target's Screen
────    ────────────────         ──────                    ───────────────
T=0     Target visible           Target exposed            Standing in open
T=30    Target visible           Target starts moving      Moving to cover
T=60    Target visible           Target behind wall        Behind wall (safe!)
        Shooter FIRES
T=90    (shot in transit)        Receives shot at T=90     Relaxing behind wall
                                 Rewinds to T=0 (60ms
                                 latency + 30ms interp)
                                 At T=0, target WAS
                                 exposed → HIT!
T=110                            Sends damage to target
T=130                                                      Receives damage
                                                           → DIES BEHIND WALL
```

The total "behind cover" window equals the shooter's full round-trip time plus the target's one-way latency plus interpolation delays. At typical latencies, this might be 60-150ms of "unfair" death perception.

### Mitigation Strategies

#### 1. Maximum Rewind Limit

Cap the maximum time the server will rewind. Beyond this limit, the shooter's perspective is no longer honored.

| Game | Maximum Rewind |
|------|---------------|
| Source Engine (default) | 1000ms (`sv_maxunlag`) |
| Overwatch | 250ms |
| Battlefield 4 | 250ms |
| Call of Duty: Infinite Warfare | 500ms |

**Trade-off**: Lower limits protect the target but mean high-ping shooters must lead their targets.

#### 2. Partial Compensation

Instead of fully rewinding to the shooter's time, rewind only *partially* -- splitting the difference between shooter and target perspectives.

```
rewind_time = shooter_perceived_time * compensation_factor  // e.g., 0.8 = 80% compensation
```

This reduces the "behind cover" window at the cost of slightly less accurate hit registration for the shooter.

#### 3. Rollback Distance Awareness

Some implementations scale compensation based on target velocity. A fast-moving target that is actively taking cover gets less rewind than a stationary target:

```
effective_rewind = base_rewind - (target_velocity * velocity_penalty_factor)
```

#### 4. Client-Side Kill Confirmation Delay

Overwatch uses a system where mutual kills are possible -- if both players fire shots that would have killed the other within a certain time window, both die. This feels more fair than one player's shot being silently dropped.

#### 5. Visual Communication

Some games show a "killed by" indicator from the shooter's perspective, helping the target understand *why* they died behind cover. Halo Infinite plans to add round-trip time displays on scoreboards so players can contextualize laggy interactions.

#### 6. Map and Game Design

- Thick walls and extended cover areas mean a few frames of "behind cover" death is less common because players must move farther to be truly safe
- Movement speed limitations reduce how far a target can travel during the rewind window

---

## Sub-Tick Lag Compensation

Traditional lag compensation operates at the server's **tick rate** -- the granularity of the simulation. At 64 tick, actions are quantized to 15.6ms windows; at 128 tick, to 7.8ms windows. Sub-tick systems aim to remove this quantization, evaluating actions at the precise moment they occurred, even between ticks.

### Counter-Strike 2's Sub-Tick System

CS2 replaced CS:GO's traditional 64/128 tick model with a **sub-tick architecture**:

**How it works:**
1. Every client input (movement, shooting, grenade throws) is tagged with a **precise timestamp** (sub-millisecond accuracy)
2. These timestamped inputs are sent to the server alongside the regular tick-based updates
3. The server collects inputs from all clients and orders them **chronologically**, not by tick boundary
4. When evaluating a shot, the server uses the exact timestamp to rewind entities to the precise sub-tick position, interpolating between the tick-boundary snapshots

**Example:**
```
64 Hz server, tick duration = 15.625ms

Tick 100 (T=0ms)     Player fires at T=7.2ms     Tick 101 (T=15.625ms)
      │                        │                          │
      ▼                        ▼                          ▼
   ┌──────────────────────────────────────────────────────┐
   │  Snapshot A               Shot                Snapshot B │
   └──────────────────────────────────────────────────────┘

   Rewind target to: lerp(SnapshotA, SnapshotB, 7.2 / 15.625)
                    = lerp(SnapshotA, SnapshotB, 0.461)
```

**Claimed benefit:** The sub-tick system should deliver **128-tick equivalent precision on 64 Hz servers**, since the actual evaluation happens at sub-tick granularity regardless of the server's update rate.

**Community reception:** Mixed. While the system is theoretically superior, players have reported inconsistencies, particularly around:
- Mid-spray hitbox rewinding bugs (one was fixed in a 2024 update where "lag compensation would rewind target hitboxes further into the past than what was on screen")
- Movement registration feeling less crisp than CS:GO's 128-tick servers
- Grenade lineup precision being affected by sub-tick timestamp differences

### Photon Fusion's Sub-Tick Accuracy

Photon Fusion (a Unity networking framework) implements sub-tick lag compensation as an opt-in feature:

**Architecture:**
- The server maintains a **hitbox history buffer** configurable via `Hitbox Buffer Length In Ms`
- Each snapshot stores hitbox transforms at discrete tick boundaries
- When a lag-compensated query is performed, Fusion identifies the two bracketing ticks

**Two accuracy modes:**

| Mode | Behavior |
|------|----------|
| **Tick-Aligned** (default) | Snaps to the nearest discrete tick. Queries resolve against either the "from" or "to" interpolation snapshot based on proximity. |
| **Sub-Tick Accuracy** | Uses the `SubtickAccuracy` flag in `HitOptions`. Interpolates between the two bracketing tick snapshots using the **exact interpolation factor** that was active on the player's screen when they pressed fire. |

**API example** (C# / Photon Fusion):
```csharp
var hitOptions = HitOptions.SubtickAccuracy | HitOptions.IgnoreInputAuthority;

Runner.LagCompensation.RaycastAll(
    origin:     firePointPosition,
    direction:  firePointDirection,
    length:     100f,
    player:     Object.InputAuthority,   // whose perspective to rewind to
    hits:       _hitBuffer,
    layerMask:  hitboxLayerMask,
    options:    hitOptions
);
```

Fusion handles all the complexity internally: determining the player's view time, finding the right snapshots, interpolating hitboxes, and performing the query.

**Constraints:** Lag compensation in Fusion is only available in **Server Mode** and **Host Mode** (not in Shared Mode), because a central authority is required to maintain the canonical history buffer.

---

## Integration with Other Techniques

Lag compensation does not exist in isolation. It is one component of a larger netcode architecture:

### Lag Compensation + Client-Side Prediction (CSP)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENT                                       │
│                                                                     │
│   Input → CSP predicts shooter movement locally (instant response)  │
│        → Entity Interpolation renders OTHER players in the past     │
│        → Client sends input + timestamp to server                   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                        SERVER                                       │
│                                                                     │
│   Receives input → Applies shooter movement (authoritative)         │
│                  → If shot fired:                                    │
│                       Lag Compensation rewinds OTHER players         │
│                       to shooter's perceived time                    │
│                       Performs hit test                              │
│                       Restores current positions                    │
│                  → Sends state update to all clients                │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                        CLIENT (correction)                          │
│                                                                     │
│   Receives server state → Reconciles own position                   │
│                         → Updates interpolation targets for others  │
│                         → Displays hit markers / damage events      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key insight:** The shooter's own position is NOT rewound -- it is processed through client-side prediction and server reconciliation. Only OTHER entities are rewound to the shooter's perceived time.

### Lag Compensation + Entity Interpolation

Entity interpolation causes remote entities to be rendered slightly in the past (typically 2 ticks behind the latest received snapshot). This delay is **factored into the rewind calculation**:

```
rewind_time = current_server_time - one_way_latency - interpolation_delay
```

If interpolation delay is not accounted for, the server would rewind to a time that is slightly too recent, causing shots to miss even with compensation enabled.

### Lag Compensation + Projectile Prediction

Projectile weapons interact differently with lag compensation:

- **Overwatch approach**: Projectiles are predicted on the shooter's client. The server spawns the authoritative projectile at the lag-compensated origin point (where the shooter was when they fired) but simulates it **forward** in current time. The projectile is NOT rewound -- only its spawn point is.
- **Valorant approach**: Since Valorant is primarily hitscan, most weapons use standard lag-compensated raycasts. Abilities with travel time use server-authoritative simulation.

### Lag Compensation + Rollback Netcode

In fighting games and some action games using GGPO-style rollback, lag compensation is handled differently. Rather than the server rewinding, each client predicts remote player inputs and resimulates from the last confirmed state on misprediction. This is a fundamentally different model -- there is no server-side history buffer or rewinding. Instead, the *entire simulation* can be rolled back and replayed.

---

## Pseudo-Code and Code Examples

### Pseudo-Code: Core Algorithm

```
// Called on the server when processing a player's fire command

function processFireCommand(shooter, command):
    // Step 1: Calculate the time the shooter perceived the world
    clientPerceivedTime = command.serverTimestamp
                        - shooter.latency
                        - shooter.interpolationDelay

    // Step 2: Clamp to maximum rewind limit
    maxRewindTime = currentServerTime - MAX_REWIND_SECONDS
    clientPerceivedTime = max(clientPerceivedTime, maxRewindTime)

    // Step 3: Rewind all potential targets
    rewindSnapshots = {}
    for each entity in allEntities:
        if entity == shooter: continue
        if not entity.isAlive: continue

        // Save current state for restoration
        rewindSnapshots[entity.id] = entity.getCurrentState()

        // Find bracketing history snapshots
        [before, after] = entity.historyBuffer.findBracketing(clientPerceivedTime)

        if before == null:
            continue  // No history available that far back

        if after == null:
            // Exact match or only one snapshot
            entity.setHitboxState(before)
        else:
            // Interpolate between snapshots
            alpha = (clientPerceivedTime - before.timestamp)
                  / (after.timestamp - before.timestamp)
            interpolatedState = lerpState(before, after, alpha)
            entity.setHitboxState(interpolatedState)

    // Step 4: Perform hit detection at rewound state
    hitResult = performRaycast(
        origin:    command.shotOrigin,
        direction: command.shotDirection,
        maxRange:  weapon.maxRange
    )

    // Step 5: Restore all entities to current state
    for each [entityId, savedState] in rewindSnapshots:
        getEntity(entityId).setHitboxState(savedState)

    // Step 6: Apply results
    if hitResult.hit:
        applyDamage(hitResult.entity, weapon.damage, hitResult.hitbox)
        notifyShooter(shooter, hitResult)
        notifyTarget(hitResult.entity, shooter)
```

### Pseudo-Code: History Buffer

```
class HistoryBuffer:
    entries: HistoryEntry[BUFFER_SIZE]  // Ring buffer
    writeIndex: int = 0
    count: int = 0

    function record(state, timestamp):
        entries[writeIndex] = { state: state, timestamp: timestamp }
        writeIndex = (writeIndex + 1) % BUFFER_SIZE
        count = min(count + 1, BUFFER_SIZE)

    function findBracketing(targetTime):
        // Find the two entries that bracket targetTime
        // Entries are in chronological order (with wrap-around)

        bestBefore = null
        bestAfter = null

        for i in 0..count-1:
            idx = (writeIndex - 1 - i + BUFFER_SIZE) % BUFFER_SIZE
            entry = entries[idx]

            if entry.timestamp <= targetTime:
                bestBefore = entry
                // The entry just after this one (if it exists) is bestAfter
                if i > 0:
                    nextIdx = (idx + 1) % BUFFER_SIZE
                    bestAfter = entries[nextIdx]
                break

        return [bestBefore, bestAfter]
```

### TypeScript Implementation

Below is a practical, detailed TypeScript implementation of a server-side lag compensation system suitable for a 2D or 3D hitscan game:

```typescript
// ─── Types ──────────────────────────────────────────────────────────

interface Vector3 {
  x: number;
  y: number;
  z: number;
}

interface HitboxSnapshot {
  position: Vector3;
  halfExtents: Vector3;  // AABB half-sizes
  rotation: number;       // Y-axis rotation (simplified)
}

interface EntitySnapshot {
  timestamp: number;              // Server tick time in ms
  tick: number;                   // Server tick number
  position: Vector3;
  hitboxes: HitboxSnapshot[];     // One per bone/region (head, torso, limbs)
  velocity: Vector3;
  isCrouching: boolean;
}

interface FireCommand {
  shooterId: string;
  origin: Vector3;
  direction: Vector3;
  clientTimestamp: number;        // Server time the client was rendering
  weaponRange: number;
  weaponDamage: number;
}

interface HitResult {
  hit: boolean;
  entityId?: string;
  hitboxIndex?: number;           // 0 = head, 1 = torso, etc.
  hitPoint?: Vector3;
  distance?: number;
}

// ─── History Buffer (Ring Buffer) ───────────────────────────────────

class HistoryBuffer {
  private entries: (EntitySnapshot | null)[];
  private writeIndex: number = 0;
  private count: number = 0;
  private capacity: number;

  constructor(
    /** Duration of history to keep, in milliseconds */
    private maxDurationMs: number,
    /** Server tick rate in Hz */
    private tickRate: number
  ) {
    // Allocate enough slots for maxDuration at the given tick rate
    this.capacity = Math.ceil((maxDurationMs / 1000) * tickRate) + 1;
    this.entries = new Array(this.capacity).fill(null);
  }

  /** Record a new snapshot at the current tick */
  record(snapshot: EntitySnapshot): void {
    this.entries[this.writeIndex] = snapshot;
    this.writeIndex = (this.writeIndex + 1) % this.capacity;
    this.count = Math.min(this.count + 1, this.capacity);
  }

  /** Find the two snapshots bracketing the target timestamp */
  findBracketing(targetTime: number): {
    before: EntitySnapshot | null;
    after: EntitySnapshot | null;
  } {
    let before: EntitySnapshot | null = null;
    let after: EntitySnapshot | null = null;

    // Iterate from newest to oldest
    for (let i = 0; i < this.count; i++) {
      const idx =
        (this.writeIndex - 1 - i + this.capacity) % this.capacity;
      const entry = this.entries[idx];
      if (!entry) continue;

      if (entry.timestamp <= targetTime) {
        before = entry;
        // The next newer entry is 'after'
        if (i > 0) {
          const afterIdx =
            (this.writeIndex - i + this.capacity) % this.capacity;
          after = this.entries[afterIdx];
        }
        break;
      }
    }

    return { before, after };
  }

  /** Get the oldest timestamp in the buffer */
  getOldestTimestamp(): number {
    if (this.count === 0) return Infinity;
    const oldestIdx =
      this.count < this.capacity
        ? 0
        : this.writeIndex; // writeIndex points to the oldest when full
    return this.entries[oldestIdx]?.timestamp ?? Infinity;
  }
}

// ─── Vector Math Utilities ──────────────────────────────────────────

function lerp(a: number, b: number, t: number): number {
  return a + (b - a) * t;
}

function lerpVector3(a: Vector3, b: Vector3, t: number): Vector3 {
  return {
    x: lerp(a.x, b.x, t),
    y: lerp(a.y, b.y, t),
    z: lerp(a.z, b.z, t),
  };
}

function subtractVector3(a: Vector3, b: Vector3): Vector3 {
  return { x: a.x - b.x, y: a.y - b.y, z: a.z - b.z };
}

function dotVector3(a: Vector3, b: Vector3): number {
  return a.x * b.x + a.y * b.y + a.z * b.z;
}

function scaleVector3(v: Vector3, s: number): Vector3 {
  return { x: v.x * s, y: v.y * s, z: v.z * s };
}

function lengthVector3(v: Vector3): number {
  return Math.sqrt(v.x * v.x + v.y * v.y + v.z * v.z);
}

// ─── AABB-Ray Intersection (Slab Method) ────────────────────────────

function rayIntersectsAABB(
  rayOrigin: Vector3,
  rayDir: Vector3,
  boxCenter: Vector3,
  boxHalfExtents: Vector3,
  maxDistance: number
): { hit: boolean; distance: number; point: Vector3 } {
  const min = subtractVector3(boxCenter, boxHalfExtents);
  const max = {
    x: boxCenter.x + boxHalfExtents.x,
    y: boxCenter.y + boxHalfExtents.y,
    z: boxCenter.z + boxHalfExtents.z,
  };

  let tMin = 0;
  let tMax = maxDistance;

  for (const axis of ["x", "y", "z"] as const) {
    const invD = 1.0 / rayDir[axis];
    let t0 = (min[axis] - rayOrigin[axis]) * invD;
    let t1 = (max[axis] - rayOrigin[axis]) * invD;

    if (invD < 0) {
      [t0, t1] = [t1, t0];
    }

    tMin = Math.max(tMin, t0);
    tMax = Math.min(tMax, t1);

    if (tMax < tMin) {
      return { hit: false, distance: Infinity, point: { x: 0, y: 0, z: 0 } };
    }
  }

  const hitPoint = {
    x: rayOrigin.x + rayDir.x * tMin,
    y: rayOrigin.y + rayDir.y * tMin,
    z: rayOrigin.z + rayDir.z * tMin,
  };

  return { hit: true, distance: tMin, point: hitPoint };
}

// ─── Lag Compensation System ────────────────────────────────────────

interface PlayerConnection {
  id: string;
  latencyMs: number;          // One-way latency estimate (RTT / 2)
  interpolationDelayMs: number; // Client's entity interpolation buffer
  isAlive: boolean;
}

class LagCompensationSystem {
  /** History buffers keyed by entity ID */
  private historyBuffers: Map<string, HistoryBuffer> = new Map();

  /** Current server time in ms */
  private currentServerTime: number = 0;

  constructor(
    /** Server tick rate in Hz */
    private tickRate: number = 64,
    /** Maximum rewind window in ms */
    private maxRewindMs: number = 250,
    /** Duration of history to keep in ms (should be >= maxRewindMs) */
    private historyDurationMs: number = 1000
  ) {}

  /** Register an entity for lag compensation tracking */
  registerEntity(entityId: string): void {
    this.historyBuffers.set(
      entityId,
      new HistoryBuffer(this.historyDurationMs, this.tickRate)
    );
  }

  /** Remove an entity (e.g., on disconnect or death) */
  unregisterEntity(entityId: string): void {
    this.historyBuffers.delete(entityId);
  }

  /**
   * Called every server tick to record current entity states.
   * This should happen AFTER the simulation step but BEFORE
   * processing any fire commands for this tick.
   */
  recordSnapshot(
    entityId: string,
    snapshot: EntitySnapshot,
    serverTimeMs: number
  ): void {
    this.currentServerTime = serverTimeMs;
    const buffer = this.historyBuffers.get(entityId);
    if (buffer) {
      buffer.record({ ...snapshot, timestamp: serverTimeMs });
    }
  }

  /**
   * Process a fire command with full lag compensation.
   *
   * @param command       The fire command from the client
   * @param shooter       The shooting player's connection info
   * @param allPlayers    Map of entityId → current hitbox state
   * @param connections   Map of entityId → connection info
   * @returns             Hit result
   */
  processFireCommand(
    command: FireCommand,
    shooter: PlayerConnection,
    allPlayers: Map<string, EntitySnapshot>,
    connections: Map<string, PlayerConnection>
  ): HitResult {
    // ── Step 1: Calculate the shooter's perceived time ──
    const perceivedTime =
      this.currentServerTime -
      shooter.latencyMs -
      shooter.interpolationDelayMs;

    // ── Step 2: Clamp to max rewind window ──
    const earliestAllowed = this.currentServerTime - this.maxRewindMs;
    const rewindTime = Math.max(perceivedTime, earliestAllowed);

    // ── Step 3: Rewind entities and test hits ──
    let closestHit: HitResult = { hit: false };
    let closestDistance = Infinity;

    for (const [entityId, currentSnapshot] of allPlayers) {
      // Skip the shooter themselves
      if (entityId === command.shooterId) continue;

      // Skip dead players
      const conn = connections.get(entityId);
      if (!conn || !conn.isAlive) continue;

      // Get history buffer
      const buffer = this.historyBuffers.get(entityId);
      if (!buffer) continue;

      // Find bracketing snapshots
      const { before, after } = buffer.findBracketing(rewindTime);
      if (!before) continue;

      // Compute interpolated hitbox positions
      let rewoundHitboxes: HitboxSnapshot[];

      if (!after || before.timestamp === after.timestamp) {
        // Exact match or edge case -- use the 'before' snapshot directly
        rewoundHitboxes = before.hitboxes;
      } else {
        // Interpolate between the two snapshots
        const alpha =
          (rewindTime - before.timestamp) /
          (after.timestamp - before.timestamp);

        rewoundHitboxes = before.hitboxes.map((hb, i) => ({
          position: lerpVector3(hb.position, after.hitboxes[i].position, alpha),
          halfExtents: hb.halfExtents, // Extents don't change between ticks
          rotation: lerp(hb.rotation, after.hitboxes[i].rotation, alpha),
        }));
      }

      // Test ray against each hitbox (in priority order: head first)
      for (let hbIdx = 0; hbIdx < rewoundHitboxes.length; hbIdx++) {
        const hb = rewoundHitboxes[hbIdx];
        const result = rayIntersectsAABB(
          command.origin,
          command.direction,
          hb.position,
          hb.halfExtents,
          command.weaponRange
        );

        if (result.hit && result.distance < closestDistance) {
          closestDistance = result.distance;
          closestHit = {
            hit: true,
            entityId,
            hitboxIndex: hbIdx,
            hitPoint: result.point,
            distance: result.distance,
          };
        }
      }
    }

    return closestHit;
  }
}

// ─── Usage Example ──────────────────────────────────────────────────

/*
const lagComp = new LagCompensationSystem(
  64,    // 64 tick server
  250,   // Max 250ms rewind
  1000   // Keep 1s of history
);

// Register players
lagComp.registerEntity("player-1");
lagComp.registerEntity("player-2");

// Every tick, record snapshots
lagComp.recordSnapshot("player-1", player1Snapshot, serverTime);
lagComp.recordSnapshot("player-2", player2Snapshot, serverTime);

// When a player fires
const hitResult = lagComp.processFireCommand(
  fireCommand,
  shooterConnection,
  allPlayerSnapshots,
  allConnections
);

if (hitResult.hit) {
  const damageMultiplier = hitResult.hitboxIndex === 0 ? 2.5 : 1.0; // Headshot
  applyDamage(hitResult.entityId!, fireCommand.weaponDamage * damageMultiplier);
}
*/
```

### Key Design Decisions in the Implementation

1. **Separate history buffers per entity**: Each player has their own ring buffer, allowing independent memory management and easy cleanup on disconnect.

2. **Interpolation between snapshots**: When the rewind time falls between two recorded ticks, linear interpolation produces sub-tick accuracy without requiring the CS2-style sub-tick system.

3. **Priority-ordered hitbox testing**: Hitboxes are tested in order (head first), and the closest hit wins. This mirrors how real engines prioritize headshots.

4. **Max rewind clamping**: The `maxRewindMs` parameter prevents high-latency players from rewinding excessively, protecting low-latency targets.

---

## Anti-Cheat Considerations

Lag compensation introduces attack surfaces that anti-cheat systems must address:

### Timestamp Manipulation

**Threat:** A cheating client sends artificially old timestamps, causing the server to rewind further than the client's actual latency warrants. This makes targets easier to hit because they are effectively "frozen" further in the past.

**Mitigation:**
- The server should **compute the rewind time itself** based on measured RTT, not trust client-supplied timestamps
- Maintain a running average of each client's latency and reject commands with timestamps deviating significantly from the expected range
- Source Engine uses: `command_execution_time = current_server_time - packet_latency - cl_interp`, where `packet_latency` is server-measured, not client-reported

### Maximum Rewind Limits

**Configuration:** The `sv_maxunlag` ConVar in Source Engine caps rewind at 1.0 second by default. Competitive servers typically set this much lower (200-500ms).

**Effect:** Even if a client claims extremely high latency, the server will not rewind beyond the cap. Players with genuinely high latency beyond the cap must lead their targets, similar to playing without lag compensation.

### Speed Hacks and Time Manipulation

**Threat:** A client runs the game at an accelerated rate, sending commands faster than real-time. This can cause the server to process more inputs per second, potentially allowing faster movement or firing.

**Mitigation:**
- Server-side **command rate limiting**: reject commands arriving faster than the tick rate allows
- **Clock drift detection**: monitor the client's reported timestamps against server time. Consistent drift indicates time manipulation
- **Movement validation**: verify that the distance traveled between commands is physically possible given the time delta

### Artificial Latency (Lag Switching)

**Threat:** A player deliberately introduces latency spikes to abuse lag compensation. During the spike, they continue to see and shoot at enemy positions. When the spike resolves, all their shots are processed with large rewind windows.

**Mitigation:**
- **Adaptive rewind limits**: reduce the maximum rewind window for players with unstable connections or frequent latency spikes
- **Latency spike detection**: if RTT suddenly increases dramatically, cap the rewind to the player's *average* latency rather than the current spike value
- **Command queue limits**: limit how many commands can be buffered and processed in a single tick to prevent "burst" attacks

### Hitbox Desync Exploitation

**Threat:** Cheats that manipulate client-side animations or hitbox positions to create desyncs between what the server thinks the player's hitboxes are and where they visually appear. This can make a player harder to hit while their own lag compensation works normally.

**Mitigation:**
- Server-authoritative animation state
- Validate that client-reported positions are consistent with server simulation
- Use server-side hitbox positions for *all* lag-compensated queries, never client-reported ones

---

## Game Implementations

### Counter-Strike (CS:GO / CS2)

| Aspect | CS:GO | CS2 |
|--------|-------|-----|
| **Tick Rate** | 64 Hz (Matchmaking) / 128 Hz (FACEIT/ESEA) | 64 Hz (sub-tick) |
| **Lag Comp Model** | Traditional tick-aligned rewind | Sub-tick timestamped rewind |
| **Max Rewind** | `sv_maxunlag` = 1.0s (default) | Similar, with sub-tick precision |
| **Key Innovation** | Mature, well-understood model | Sub-tick: actions timestamped between ticks, evaluated at precise sub-tick positions |
| **Known Issues** | 64-tick "hitreg" complaints | Sub-tick inconsistencies, mid-spray rewind bugs (patched) |

CS2's sub-tick system timestamps every input and evaluates it at the precise sub-frame moment, rather than snapping to tick boundaries. Valve has stated this should provide 128-tick equivalent accuracy on 64 Hz servers, though community reception has been mixed.

### Overwatch / Overwatch 2

| Aspect | Detail |
|--------|--------|
| **Tick Rate** | 63 Hz (server); clients can receive at 63 Hz with high-bandwidth option |
| **Lag Comp Model** | Favor-the-shooter with 250ms rewind cap |
| **Mutual Kills** | Allowed -- both players can kill each other within the same compensation window |
| **Hitscan vs Projectile** | Hitscan: fully lag-compensated. Projectiles: spawn at compensated origin but simulate forward in real-time |
| **Abilities** | Movement abilities (e.g., Tracer Recall) use server-authoritative state; are NOT rewound |
| **Key Innovation** | Predicted projectiles (e.g., rockets) -- the client predicts slow-moving projectiles locally for instant feedback |

Overwatch's GDC talk described their approach to "favor the shooter" with limits: high-latency players (>250ms one-way) lose lag compensation benefits and must lead targets.

### Valorant

| Aspect | Detail |
|--------|--------|
| **Tick Rate** | 128 Hz |
| **Lag Comp Model** | Favor-the-shooter with optimized rewind |
| **Infrastructure** | Riot Direct backbone; targeting 35ms ping for 70% of players |
| **Peeker's Advantage** | ~98ms (optimized from ~141ms baseline) |
| **Key Innovation** | Animation rollback optimization -- sphere cast pre-test, then reconstruct full animation only for potential hits (~0.3ms total vs ~0.1ms per player) |
| **Buffering** | Minimal: 1 buffered frame of client movement, average 0.5 frame on server |
| **Interpolation Delay** | 7.8ms (one tick at 128 Hz) |

Valorant's engineering blog provides the most detailed public analysis of peeker's advantage mathematics in any commercial game.

### Halo Infinite

| Aspect | Detail |
|--------|--------|
| **Tick Rate** | 60 Hz (Arena 4v4), 30 Hz (Big Team Battle) |
| **Lag Comp Model** | Favor-the-shooter |
| **Notable Issues** | "Desync" -- high-profile community complaints about shots not registering due to non-deterministic simulation divergence |
| **Melee** | Particularly problematic; compounded latency creates "phasing" where both players lunge through each other |
| **Posthumous Shots** | Shots fired after server-side death (but before client notification) are dropped; causes frustration |
| **Key Innovation** | Detailed public communication about netcode challenges and the inherent trade-offs |

343 Industries published a comprehensive blog post acknowledging the fundamental tension between favor-the-shooter (shots feel good) and the resulting desync artifacts (getting hit behind cover, melee phasing).

---

## When to Use / When Not to Use

### Essential For

| Genre/Scenario | Why |
|---------------|-----|
| **Competitive FPS (hitscan)** | Players must be able to aim at what they see and get hits. Without lag compensation, the game is unplayable at any non-trivial latency. |
| **Battle Royale** | Large player counts with varied latencies; hitscan and projectile weapons both need compensation. |
| **Third-person shooters** | Same hit detection requirements as FPS, with the added complexity of camera offset. |
| **Melee action games with precise hit detection** | Fast combat with tight hit windows (e.g., sword fighting) benefits from rewinding to the attacker's perceived time. |

### Beneficial But Not Critical

| Genre/Scenario | Why |
|---------------|-----|
| **Projectile-heavy games** | Projectiles already have travel time, so latency is partially masked. Lag compensation is still useful for spawn position accuracy. |
| **Vehicle combat** | Vehicles move predictably (momentum-based), making extrapolation more viable. Lag compensation helps for turret gunners. |
| **MMORPGs** | Tab-target combat does not need spatial hit detection. Action combat MMOs benefit from it. |

### Not Needed

| Genre/Scenario | Why |
|---------------|-----|
| **Turn-based games** | No real-time hit detection; latency is irrelevant to combat resolution. |
| **RTS games** | Unit selection and commands are not spatially precise enough to need frame-level rewinding. Deterministic lockstep handles synchronization. |
| **Cooperative PvE** | Friendly fire is typically off; precision against AI can be handled with generous hitboxes and client-side hit detection. |
| **Fighting games (rollback)** | Use GGPO-style rollback netcode instead, which resimulates the entire game state rather than rewinding individual hitboxes. |
| **Puzzle / casual games** | No combat hit detection. |

### Decision Checklist

Before implementing lag compensation, ask:

1. **Does the game have real-time ranged or melee combat?** If no, you probably do not need it.
2. **Is the combat server-authoritative?** If client-authoritative, lag compensation is moot (but you have cheating problems).
3. **Are there hitscan weapons?** If yes, lag compensation is almost mandatory for a good experience.
4. **What latency range must be supported?** Higher supported latency = larger history buffer = more "shot behind walls" artifacts.
5. **Is there competitive PvP?** If yes, the perceived fairness of hit detection is paramount.
6. **Can the server afford the CPU and memory cost?** Ring buffers are cheap, but rewinding and raycasting across many entities adds per-shot cost.

---

## References

### Primary Sources

- **Yahn Bernier (Valve)** -- "Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization" (2001) -- [Valve Developer Community](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization)
- **Valve Developer Community** -- "Source Multiplayer Networking" -- [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- **Valve Developer Community** -- "Lag Compensation" -- [Lag Compensation Wiki](https://developer.valvesoftware.com/wiki/Lag_Compensation)
- **Source Engine 2007 Source Code** -- `player_lagcompensation.cpp` -- [GitHub (VSES/SourceEngine2007)](https://github.com/VSES/SourceEngine2007/blob/master/se2007/game/server/player_lagcompensation.cpp)

### Game Developer Talks and Articles

- **Tim Ford & Philip Orwig (Blizzard)** -- "Overwatch Gameplay Architecture and Netcode" (GDC 2017) -- [GDC Vault](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)
- **Riot Games Technology** -- "Peeking into Valorant's Netcode" -- [Riot Technology Blog](https://technology.riotgames.com/news/peeking-valorants-netcode)
- **Riot Games Technology** -- "Valorant's 128-Tick Servers" -- [Riot Technology Blog](https://technology.riotgames.com/news/valorants-128-tick-servers)
- **343 Industries (Richard Watson)** -- "Closer Look: Halo Infinite's Online Experience" -- [Halo Waypoint](https://www.halowaypoint.com/news/closer-look-halo-infinite-online-experience)
- **HLTV** -- "Lag compensation change headlines CS2 update" -- [HLTV](https://www.hltv.org/news/40200/lag-compensation-change-headlines-cs2-update)

### Frameworks and Engines

- **Photon Fusion 2** -- "Lag Compensation" documentation -- [Photon Engine Docs](https://doc.photonengine.com/fusion/current/manual/advanced/lag-compensation)
- **SnapNet** -- "Performing Lag Compensation in Unreal Engine 5" -- [SnapNet Blog](https://snapnet.dev/blog/performing-lag-compensation-in-unreal-engine-5/)
- **Mirror Networking** -- "Lag Compensation" -- [Mirror Docs](https://mirror-networking.gitbook.io/docs/manual/general/lag-compensation)
- **Unity Netcode for GameObjects** -- "Dealing with Latency" -- [Unity Docs](https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.7/manual/learn/dealing-with-latency.html)

### Community and Educational Resources

- **Glenn Fiedler** -- "Networked Physics" series -- [Gaffer on Games](https://gafferongames.com/categories/networked-physics/)
- **Daniel Jimenez Morales** -- "The Art of Hit Registration" -- [Blog Post](https://danieljimenezmorales.github.io/2023-10-29-the-art-of-hit-registration/)
- **Nicola Geretti** -- "Netcode Series Part 4: Projectiles (Lag Compensation and Prediction)" -- [Medium](https://medium.com/@geretti/netcode-series-part-4-projectiles-96427ac53633)
- **Outscal** -- "Lag Compensation in FPS Games" -- [Outscal Blog](https://outscal.com/blog/lag-compensation-in-fps-games-the-hidden-systems-making-your-shots-count)
- **TimeToCode** -- "The Peeker's Compromise: A Fair(er) Netcode Model" -- [Tumblr](https://timetocode.tumblr.com/post/635717298958794752/the-peekers-compromise-a-fairer-netcode-model)

### Academic

- **Enhancing the experience of multiplayer shooter games via advanced lag compensation** -- ACM Multimedia Systems 2018 -- [ACM Digital Library](https://dl.acm.org/doi/10.1145/3204949.3204971)
- **Master of Science Thesis: Implementation and Evaluation of Hit Registration** -- Linkoping University -- [Full Text (PDF)](https://liu.diva-portal.org/smash/get/diva2:1605200/FULLTEXT01.pdf)
