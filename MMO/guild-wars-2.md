# Guild Wars 2: Netcode Deep-Dive

## Overview

Guild Wars 2 (2012, ArenaNet) employs a **hybrid authority model** with a distinctive
approach to latency compensation. The server is authoritative for all critical game
state -- combat resolution, item ownership, skill validation -- but grants the client
significant trust over movement and position. This design, inherited from the original
Guild Wars (2005), prioritizes fluid moment-to-moment gameplay feel over strict
anti-cheat enforcement.

The most technically novel aspect of GW2's netcode is its **time-based latency
compensation system**, described by Lead Gameplay Programmer MikeLewis. Rather than
using traditional lag compensation techniques like rewinding server state (as in
Source Engine), GW2 **fast-forwards animations** on both server and client to mask
input delay. This two-stage approach is unusual among MMOs and creates a distinctive
feel where skills appear responsive regardless of latency -- at the cost of visual
artifacts during high-ping scenarios.

The game's architecture faces its greatest challenge in **World vs. World (WvW)**,
where 50-100+ players converge in a single map instance. The server's processing
model, inherited from Guild Wars 1, struggles with the quadratic scaling of
player-vs-player AoE interactions, resulting in the infamous "skill lag" where
server game time dilates relative to real time.

| Aspect | Approach |
|---|---|
| **Protocol** | TCP (confirmed by Patrick Wyatt, HandmadeCon 2015) |
| **Movement** | Client-trusted with server validation |
| **Combat** | Server-authoritative with time-based latency compensation |
| **Remote Players** | Dead reckoning / extrapolation from last known state |
| **AoE Target Cap** | 5 targets (balance + server performance constraint) |
| **Networking Codebase** | ~50,000 lines custom C++ (Patrick Wyatt) |
| **Infrastructure** | Microservices (since 2001), migrated to AWS (2017) |

---

## Architecture

### TCP-Based Custom Networking Stack

Guild Wars and Guild Wars 2 run entirely on **TCP**, a decision Patrick Wyatt
(co-founder of ArenaNet, formerly Blizzard's Battle.net lead) made deliberately.
His rationale, explained at HandmadeCon 2015 and on his Code of Honor blog:

> "For Guild Wars, everything is TCP."

> "Almost every single message that you would send is something that you would have
> wanted to know correctly... I picked up this [item] right, that is not a message
> that you can lose because if you did then multiple people could pick up the same item."

The core networking libraries Wyatt wrote comprise approximately **50,000 lines of
custom C++ code** and took several years to fully stabilize due to esoteric edge
cases in TCP behavior -- window sizing, message buffering, connection teardown
semantics, and rate-limiting for DDoS protection.

TCP's primary risk in game networking is **route convergence** during ISP maintenance
windows, where connections could drop for 30+ seconds, affecting tens of thousands
of players simultaneously. ArenaNet mitigated this through connection resilience
and reconnection logic rather than switching to UDP.

### Microservices Architecture (2001-Present)

ArenaNet has been writing microservices since 2001. At launch, Guild Wars featured
**18 distinct server types**, each handling a specific concern:

- **Lobby server** -- authentication, friends lists, guild info, persistent connections
- **Game servers** -- authoritative simulation of combat, movement, skills
- **File servers** -- distributed across 6 global data centers with delta compression
- **Match-making servers** (SiteSrv)
- **Database cache servers**
- **Database servers**
- **Chat, tournament, trading post**, and other specialized services

All servers shared a common core framework called **ArenaSrv** (the loader) with
dynamically hot-loadable DLL modules. A critical design achievement: developers
could run all 18 server types simultaneously in a single process on one machine for
local testing. This same code scaled to support millions of concurrent players in
production (Stephen Clarke-Willson, GDC 2013, "Guild Wars 2: Scaling from One to
Millions").

Inter-server communication used a custom **RPC protocol** (SrvSocket) with built-in
versioning, enabling zero-downtime rolling upgrades -- servers could be taken offline
individually for updates while maintaining service continuity.

### AWS Migration (2017)

Originally hosted in internal data centers in Dallas and Frankfurt, Guild Wars 2
migrated to AWS during the Path of Fire expansion launch (2017). Stephen
Clarke-Willson (VP Technology) described the migration at GDC 2017:

- Hardware purchased in 2012 was failing frequently, limiting ability to experiment
  with faster hardware or better networking
- The Path of Fire launch served as the proving ground -- scaling up for double the
  normal active users, then stabilizing maps in AWS
- After successful deployment, all remaining services migrated without downtime
- ArenaNet became the **first Windows-based "all-in" MMO deployment on AWS**
- Infrastructure: AWS CloudFormation for deployment, Amazon CloudWatch for monitoring,
  EC2 instance types T3, C5, M5, and R5
- Operational costs were cut in half within the first year through aggressive analysis

### Megaserver System (2014)

In 2014, ArenaNet introduced the **megaserver system** -- a weighted load balancer
that dynamically creates and destroys map instances based on population. The system
scores each map copy by affinity factors (party, guild, language, home world) and
places players in the highest-scoring instance. This replaced the earlier overflow
system where players were dumped into secondary instances.

---

## Movement: Client-Trusted

Guild Wars 2 grants the client **significant authority over player position**. The
client performs local pathfinding against the navmesh, handles collision detection,
and reports position updates to the server. The server validates but is permissive --
it trusts client-reported positions unless they violate physical constraints.

Patrick Wyatt described this model at HandmadeCon 2015:

> "If I start walking, I don't have to go ask the server 'is it okay if I walk over
> here?' I already know all the collision information that's local to my system...
> I'm telling the server the whole time while I'm doing this and the server telling
> me what is actually true."

> "The server just says nope. I [the server] am telling the truth. You have to move
> back."

### How It Works

1. **Client moves immediately** using local navmesh and collision data
2. **Client streams position updates** to the server continuously
3. **Server validates** against its own state but is lenient -- small discrepancies
   are accepted to avoid constant rubber-banding
4. **Server rejects** only when positions are clearly invalid (inside walls,
   impossible distances)
5. **Rubber-banding** occurs only on explicit server rejection, not as a continuous
   correction

### Historical Exploits

This permissive model has historically allowed movement-based exploits. Speed hacks,
teleport hacks, fly hacks, and noclip exploits have all functioned in Guild Wars 2
because the server lacks aggressive internal tracking of expected player velocity
and position. Community analysis has noted that the server "should be able to
determine a player's speed and location based on their world position" but the
validation thresholds are generous enough that manipulation is possible.

### Shadowstep Validation

Skills that teleport or shadowstep have **additional server-side validation**. When
the client requests activation, the server performs pathfinding to the target point
before any activation costs are paid. A second validation occurs when the skill
actually executes, and this check can fail if the player moved to an invalid position
between the two checks.

```cpp
// Pseudo-code: Client-trusted movement with server validation

// === CLIENT SIDE ===
struct ClientMovement {
    Vector3 position;
    Vector3 velocity;
    float   move_speed;
    NavMesh* local_navmesh;

    void Update(float dt, InputState input) {
        // Client computes movement locally -- no server round-trip needed
        Vector3 desired_direction = InputToWorldDirection(input);
        Vector3 desired_position  = position + desired_direction * move_speed * dt;

        // Client-side pathfinding and collision against local navmesh
        NavMeshQueryResult result = local_navmesh->FindNearestValidPoint(desired_position);
        if (result.valid) {
            position = result.snapped_position;
            velocity = desired_direction * move_speed;
        } else {
            // Blocked by collision -- slide along wall
            position = local_navmesh->SlideAlongEdge(position, desired_position);
            velocity = (position - old_position) / dt;
        }

        // Stream position to server at regular intervals (~10-20 Hz)
        if (ShouldSendUpdate()) {
            SendToServer(PacketType::POSITION_UPDATE, {
                .position  = position,
                .velocity  = velocity,
                .timestamp = GetClientTime()
            });
        }
    }

    void OnServerCorrection(Vector3 corrected_position) {
        // Rubber-band: snap to server-authoritative position
        // Only happens on explicit rejection, not continuously
        position = corrected_position;
        velocity = Vector3::Zero();
    }
};

// === SERVER SIDE ===
struct ServerMovementValidator {
    Vector3 last_known_position;
    float   last_update_time;
    float   max_speed;                  // Character's maximum allowed speed
    float   tolerance_multiplier = 1.5; // Permissive threshold

    ValidationResult ValidatePosition(Vector3 reported_pos, float timestamp) {
        float dt = timestamp - last_update_time;
        float max_distance = max_speed * dt * tolerance_multiplier;
        float actual_distance = Distance(last_known_position, reported_pos);

        if (actual_distance > max_distance) {
            // Position is impossibly far -- reject and rubber-band
            return { .accepted = false, .corrected_position = last_known_position };
        }

        // Check navmesh validity (is the position walkable?)
        if (!server_navmesh->IsValidPosition(reported_pos)) {
            return { .accepted = false, .corrected_position = last_known_position };
        }

        // Accept the position -- server trusts the client
        last_known_position = reported_pos;
        last_update_time    = timestamp;
        return { .accepted = true };
    }
};
```

---

## Remote Players: Dead Reckoning

Other players' characters are rendered using **dead reckoning** (extrapolation from
the last known state). When the server relays position updates for remote players,
the client extrapolates their movement along their last known trajectory. During
server hiccups or packet delays, remote characters continue moving in their previous
direction rather than freezing in place.

GW2 simulates gravity independently and interpolates horizontal movement for remote
characters, with periodic corrections when new authoritative updates arrive.

### Culling System (2012-2013)

At launch, GW2 employed aggressive **player culling** -- a hard limit on the number
of characters reported to each client. Only the closest characters were transmitted,
meaning distant enemies in WvW were literally invisible. This reduced bandwidth and
server load but created severe gameplay problems (invisible zergs). Culling was
removed in March 2013 and replaced with client-side LOD controls (generic character
models for distant players, configurable "Character Model Limit" settings).

The removal of culling increased the networking burden: every client now receives
updates for every character in range, with the client controlling rendering fidelity.

```cpp
// Pseudo-code: Dead reckoning for remote player characters

struct RemotePlayerState {
    Vector3 position;
    Vector3 velocity;
    float   last_server_update_time;
    float   interpolation_factor;   // Blending weight for corrections
};

struct DeadReckoningSystem {
    // Called every client frame to update remote player positions
    void UpdateRemotePlayers(float current_time,
                             std::vector<RemotePlayerState>& players) {
        for (auto& player : players) {
            float dt = current_time - player.last_server_update_time;

            // Extrapolate: continue along last known velocity
            Vector3 predicted_position = player.position + player.velocity * dt;

            // Apply gravity for vertical axis (simulated independently)
            predicted_position.y -= 0.5f * GRAVITY * dt * dt;

            // Snap to navmesh to prevent floating/underground
            predicted_position = SnapToNavMesh(predicted_position);

            player.render_position = predicted_position;
        }
    }

    // Called when a new server update arrives for a remote player
    void OnServerUpdate(RemotePlayerState& player,
                        Vector3 server_position,
                        Vector3 server_velocity,
                        float server_timestamp) {
        // Calculate prediction error
        Vector3 predicted = player.position
                          + player.velocity
                          * (server_timestamp - player.last_server_update_time);
        float error = Distance(predicted, server_position);

        if (error < SNAP_THRESHOLD) {
            // Small error: smoothly blend toward corrected position
            // Avoids jarring pops for minor discrepancies
            player.position = Lerp(player.render_position,
                                   server_position,
                                   SMOOTH_CORRECTION_RATE);
        } else {
            // Large error: snap immediately (player changed direction,
            // used a movement skill, or rubber-banded)
            player.position = server_position;
        }

        player.velocity                = server_velocity;
        player.last_server_update_time = server_timestamp;
    }
};
```

---

## Combat System: Server-Authoritative with Time-Based Latency Compensation

Combat in Guild Wars 2 is fully **server-authoritative**. The server resolves all
hit detection, damage calculations, condition applications, and buff/debuff
interactions. However, ArenaNet implemented a unique **time-based latency
compensation** system to make combat feel responsive despite the server authority.

### Hit Resolution

Patrick Wyatt described the fundamental model: when a projectile is fired, the
hit/miss decision is made **at the point of firing** on the server, not based on
visual trajectory:

> "At the point the arrow is fired we decide whether it is going to hit or not.
> Has nothing to do with motion."

This means projectile animations are cosmetic -- the outcome is already determined.
Animation timing for item pickup and skill activation is deliberately designed to
mask round-trip latency:

> "If you want to pick up an item... you have an animation sequence that takes a
> while to get done so that the likelihood is that the round trip to the server has
> already elapsed."

### The Two-Stage Latency Compensation System

MikeLewis (Lead Gameplay Programmer) described a novel two-stage fast-forward
mechanism for hiding input delay:

**Stage 1 -- Server-Side Compensation:**
The server estimates how long the player's input took to arrive. It then fast-forwards
the skill action by that amount.

> "If the compensator on the server guesses that you actually started that action
> 100 milliseconds ago, it fast-forwards the action and says 'start the action at
> 100ms in and play for 900ms' instead of the usual 'start at 0ms and play for
> 1000ms.'"

**Stage 2 -- Client-Side Secondary Compensation:**
When the server relays the compensated action to other players' clients, additional
latency is incurred. The receiving client applies a second stage of fast-forwarding.

> "If the relay takes 50ms, the final action as seen by the second player will look
> like 'start at 150ms and play for 850ms.' This can lead to significant popping
> and other visual artifacts."

The GW2 team ran **latency tuning experiments** (documented in a public forum thread)
to adjust these compensation parameters, particularly the secondary client-side
compensation, to find the best balance between responsiveness and visual quality.

```cpp
// Pseudo-code: Two-stage animation fast-forward latency compensation

// === STAGE 1: Server-Side Compensation ===
struct ServerLatencyCompensator {
    // Estimates one-way input delay from client to server.
    // GW2 combines actual network latency measurement with a guess at
    // how long the server will take to process the message.
    float EstimateInputDelay(PlayerConnection& conn) {
        float network_latency = conn.GetSmoothedOneWayLatency();
        float processing_estimate = conn.GetServerProcessingEstimate();
        return network_latency + processing_estimate;
    }

    SkillAction CompensateSkillActivation(SkillRequest request,
                                          PlayerConnection& conn) {
        float input_delay = EstimateInputDelay(conn);

        // Clamp to prevent excessive fast-forwarding
        input_delay = Clamp(input_delay, 0.0f, MAX_COMPENSATION_MS);

        SkillAction action;
        action.skill_id       = request.skill_id;
        action.caster         = request.player_id;
        action.total_duration = GetSkillDuration(request.skill_id);

        // Fast-forward: start the action partway through
        action.start_offset   = input_delay;
        action.remaining_time = action.total_duration - input_delay;

        // Server resolves the skill at the compensated time
        // (hit detection, damage, conditions happen at server time)
        ResolveSkillAtTime(action, input_delay);

        return action;
    }
};

// === STAGE 2: Client-Side Secondary Compensation ===
struct ClientSecondaryCompensator {
    // When the client receives a skill action from the server (for a remote
    // player), it applies a second fast-forward to account for relay latency.
    void PlayRemoteSkillAnimation(SkillAction action, float relay_latency) {
        float total_offset = action.start_offset + relay_latency;

        // The animation starts partway through -- the further the total
        // compensation, the more of the wind-up is skipped
        float animation_start = total_offset;
        float animation_remaining = action.total_duration - total_offset;

        if (animation_remaining <= 0) {
            // Entire animation was consumed by latency -- show only the
            // impact effect (pop)
            PlayImpactEffectOnly(action);
            return;
        }

        // Play the animation starting from the compensated point
        PlaySkillAnimation(action.skill_id, animation_start, animation_remaining);

        // NOTE: MikeLewis acknowledged this can cause "significant popping
        // and other visual artifacts" when total compensation is high.
    }
};
```

### Ping Measurement in GW2

GW2's displayed ping combines actual network round-trip time with an estimate of
server processing delay:

> "We actually combine this data with a second statistic, which is a _guess_ at
> how long it will take the server to actually process and respond to your message."

This means displayed ping is always higher than raw network latency, and the latency
compensation system operates independently of the displayed ping value.

---

## The 5-Target AoE Cap

Guild Wars 2 limits most Area of Effect skills to hitting a **maximum of 5 targets**.
This is both a balance decision and a **server performance constraint**, confirmed
by BillFreist (Gameplay Programmer) in official forum posts.

BillFreist stated:

> "The AoE hit limits exist for more than just performance reasons. We already have
> a lot of places in the game where AoE limits are extremely high (WvW siege) or
> uncapped (monster skills)."

### Why It Matters for Networking

The core issue is **quadratic scaling** of AoE interactions. In a battle with N
players, if each player can hit all others with AoE:

- Each player casts AoE skills that must check against all other players in range
- Each hit generates damage events, condition applications, buff interactions
- Each condition must be calculated server-side (GW2 does all condition calculations
  on the server)
- Each condition can modify existing conditions (overwriting, stacking)
- Each result must be broadcast back to all affected clients

Without a target cap, the computational cost scales with O(N^2) for the interaction
check alone, and worse when factoring in condition recalculations and state
broadcasts.

With a 5-target cap, the cost is bounded at O(N * 5) = O(N) for hit resolution,
transforming an exponential problem into a linear one.

```cpp
// Pseudo-code: AoE target cap and server performance implications

struct AoESkillProcessor {
    static const int TARGET_CAP = 5;

    // Without target cap: O(N^2) interaction checks in a zerg fight
    // With target cap: O(N * TARGET_CAP) = O(N) -- linear scaling
    void ProcessAoESkill(SkillInstance& skill,
                         const std::vector<Player*>& players_in_range) {

        // Step 1: Find all valid targets in the AoE radius
        std::vector<Player*> candidates;
        for (auto* player : players_in_range) {
            if (player == skill.caster) continue;
            if (!IsEnemy(skill.caster, player)) continue;
            if (Distance(skill.center, player->position) > skill.radius) continue;
            candidates.push_back(player);
        }

        // Step 2: Apply target cap -- select closest N targets
        // This bounds the per-skill computation cost
        if (candidates.size() > TARGET_CAP) {
            std::partial_sort(candidates.begin(),
                              candidates.begin() + TARGET_CAP,
                              candidates.end(),
                              [&](auto* a, auto* b) {
                return Distance(skill.center, a->position)
                     < Distance(skill.center, b->position);
            });
            candidates.resize(TARGET_CAP);
        }

        // Step 3: For each hit target, resolve damage and conditions
        // WITHOUT the cap, a 50v50 fight would be:
        //   50 casters * 50 targets * ~10 conditions each = 25,000 condition events
        // WITH the cap:
        //   50 casters * 5 targets * ~10 conditions each  = 2,500 condition events
        //   (10x reduction)
        for (auto* target : candidates) {
            DamageResult result = CalculateDamage(skill, target);
            ApplyDamage(target, result);

            // Condition application is expensive:
            // - Server calculates all conditions
            // - Checks for existing stacks (25 stack cap per condition)
            // - Recalculates based on caster's current stats
            // - Updates condition if new application is stronger
            for (auto& condition : skill.conditions) {
                ApplyCondition(target, condition, skill.caster);
            }

            // Broadcast damage event to all nearby clients
            BroadcastDamageEvent(target, result);
        }
    }
};

// In a 100-player WvW fight WITHOUT caps:
//   100 players * 100 targets * 10 conditions = 100,000 events per AoE volley
// WITH 5-target cap:
//   100 players * 5 targets * 10 conditions   = 5,000 events per AoE volley
//   (20x reduction -- the difference between playable and unplayable)
```

### Condition System Complexity

GW2's condition damage system is unusually expensive because:

1. **All calculations are server-side** -- the client does not predict condition ticks
2. **Conditions can overwrite each other** -- when a new condition is applied, the
   server compares it against existing stacks and may replace weaker ones
3. **Stat changes retroactively affect conditions** -- gaining a buff can recalculate
   already-applied conditions
4. **Per-tick recalculation** -- each condition ticks once per second, and each tick
   involves recalculating against current stats

A single attack may apply 5-15 conditions (including sigil and trait procs). In a
large fight, the server must process thousands of condition recalculations per second.

---

## WvW Scaling Challenges

World vs. World represents Guild Wars 2's most severe netcode challenge. The game
mode puts potentially hundreds of players into a single map instance, creating
server load that the engine architecture struggles to handle.

### The Core Problem: Server Time Dilation

When the game server becomes CPU-bound during large battles, **game time dilates
relative to real time**. Community analysis has documented this behavior extensively:

If the server takes 2 real seconds to process 1 second of game time, all server-side
timers -- including skill cooldowns -- run at half speed. However, the client has no
awareness of this dilation. Client-side cooldown timers count down in real time,
creating a mismatch:

- A skill with a 20-second cooldown appears ready on the client after 20 real seconds
- But the server has only processed 10 seconds of game time
- The player presses the skill and nothing happens -- the skill is still on cooldown
  server-side

This is the fundamental mechanism behind "skill lag" in WvW. It is not network
latency -- it is the server falling behind real time.

### Architecture Limitations

BillFreist confirmed the server-side bottleneck:

> "Most of what goes on in the game is processed on the server, including combat,
> skills and movement. When this bottleneck is relieved, the changes will pretty
> much speak for themselves."

> "I'm working primarily on server-side optimizations... server CPU usage is the
> biggest factor here."

The engine inherits architectural constraints from Guild Wars 1:

- **Threading model**: While not strictly single-threaded, the game simulation
  thread is the bottleneck. BillFreist clarified: "The engine isn't as single-threaded
  as you might think. It will only use as many threads as your machine can support"
  -- but the critical game logic (combat resolution, skill processing, movement
  validation) runs on a primary thread that becomes the bottleneck
- **Passive effects vs. active skills**: Passive effects (like regeneration) run on
  separate processing paths that do not get bogged down, which is why passive effects
  always work even during severe skill lag
- **Quadratic scaling**: Doubling CPU speed yields less than proportional improvement
  because the underlying algorithms have super-linear complexity

```cpp
// Pseudo-code: Server tick rate degradation under WvW load

struct GameServerInstance {
    float game_time         = 0.0f;    // Server's simulation time
    float target_tick_rate  = 30.0f;   // Desired ticks per second (33ms per tick)
    float target_tick_delta = 1.0f / 30.0f;  // ~33ms

    // Under normal load (< ~40 players), game time tracks real time
    // Under heavy load (50-100+ players), game time falls behind
    void ServerMainLoop() {
        while (running) {
            float tick_start = GetRealTime();

            // === Process all game simulation for this tick ===
            ProcessPlayerMovement();     // Validate all position updates
            ProcessSkillActivations();   // Handle queued skill requests
            ProcessCombatResolution();   // Resolve hits, damage, conditions
            ProcessConditionTicks();     // Tick all active conditions
            ProcessBuffUpdates();        // Recalculate buffs/debuffs
            BroadcastStateUpdates();     // Send results to all clients

            float tick_duration = GetRealTime() - tick_start;

            // === THE CRITICAL BOTTLENECK ===
            if (tick_duration < target_tick_delta) {
                // Healthy: tick completed in time, sleep until next tick
                Sleep(target_tick_delta - tick_duration);
                game_time += target_tick_delta;
                // Game time == Real time (1:1)
            } else {
                // OVERLOADED: tick took longer than budget
                // Game time advances by one tick, but real time advanced
                // by tick_duration (> target_tick_delta)
                game_time += target_tick_delta;
                // Game time < Real time (dilation!)

                // Example: 50v50 battle
                // tick_duration = 66ms (2x over budget)
                // Server processes 33ms of game time every 66ms of real time
                // Effective ratio: 2 seconds real = 1 second game time
                // Player's 20s cooldown takes 40 real seconds to expire
            }
        }
    }

    // The cost of ProcessCombatResolution scales super-linearly:
    //
    //   20 players: ~5ms per tick   (well within 33ms budget)
    //   50 players: ~30ms per tick  (approaching budget limit)
    //   80 players: ~80ms per tick  (2.4x over budget, game runs at 41% speed)
    //  100 players: ~150ms per tick (4.5x over budget, game runs at 22% speed)
    //
    // The nonlinear scaling comes from:
    //   - AoE hit detection:    O(players * targets_in_range)
    //   - Condition processing:  O(players * conditions_per_player)
    //   - State broadcasting:    O(players * nearby_players)
    //   - Buff recalculation:    O(players * buffs * affected_conditions)
};
```

### Skill Lag vs. Passive Effects

A telling detail about the architecture: **passive effects never lag**, even during
the worst skill lag. BillFreist's team identified that passive processing runs on
a separate code path that is not affected by the combat resolution bottleneck.
This means the skill activation and cooldown pipeline is specifically the bottleneck,
not the entire game simulation.

### Optimization Approach

Rather than rewriting the architecture, ArenaNet's approach was to optimize
individual skills:

> "We plan on going through almost every skill in the game over the coming months
> and optimizing them one-by-one... when that patch note comes through, you can
> expect a very different WvW experience."

Feature teams were made subject to **performance reviews for both client and server**
before shipping new content, preventing the regression of adding new unoptimized
skills.

### Population Cap Adjustments

ArenaNet tested reducing WvW map population caps as a short-term mitigation:

> "We still want WvW to feel like WvW, and we realize that any reduction to player
> counts will have a negative impact on the overall gameplay experience."

This acknowledged that the fundamental issue was architectural -- the server could
not handle the player density that the game mode's design demanded.

---

## Latency Compensation Deep-Dive

### How It Differs from Traditional Lag Compensation

Most FPS games (e.g., Source Engine, Overwatch) use **server-side rewind**: the server
stores a history of world states and, when processing a player's shot, rewinds the
world to where it was when the player actually fired. This gives the shooter an
accurate "what you see is what you hit" experience.

GW2 takes a fundamentally different approach because:

1. **MMO scale** -- rewinding world state for hundreds of simultaneous players is
   computationally prohibitive
2. **TCP protocol** -- no out-of-order delivery means commands arrive in sequence,
   reducing the need for timestamp-based rewind
3. **Skill-based combat** -- most skills are targeted (tab-target or ground-target),
   not free-aim, so pixel-perfect hit detection is less critical

Instead of rewinding the world, GW2 **fast-forwards the action**. The server and
client collaborate to make it appear as though the skill started earlier than it
actually did on the server.

### The Compensation Pipeline

```
Player presses skill key
    |
    v
[Client sends skill request] ----network----> [Server receives request]
    |                                              |
    v                                              v
Client plays anticipatory                   Server estimates input delay (D_input)
animation (optional cast bar)               Server fast-forwards skill by D_input
    |                                       Server resolves skill (hit/miss/damage)
    |                                       Server sends result to all clients
    |                                              |
    |                    <----network----           |
    v                                              v
[Client receives result]                    [Other clients receive relayed action]
Client adjusts local animation                     |
to match server state                              v
                                            Other clients fast-forward animation
                                            by D_input + D_relay
                                            (Stage 2 compensation)
```

### Detailed Pseudo-Code: Full Skill Validation Pipeline

```cpp
// Pseudo-code: Complete skill activation pipeline from client to resolution

// === CLIENT: Skill Request ===
struct ClientSkillSystem {
    void OnSkillKeyPressed(SkillId skill_id) {
        // 1. Local validation (fast reject)
        if (!CanCastLocally(skill_id)) return;  // cooldown, resources, etc.

        // 2. Play anticipatory animation (cast bar begins)
        //    This masks the round-trip latency to the server
        PlayCastBarAnimation(skill_id);

        // 3. Send request to server
        SendToServer(PacketType::SKILL_REQUEST, {
            .skill_id       = skill_id,
            .target_id      = GetCurrentTarget(),
            .position       = GetPlayerPosition(),
            .client_time    = GetClientTime()
        });
    }

    void OnSkillResult(SkillResultPacket result) {
        if (result.accepted) {
            // Server accepted -- adjust local animation timing to match
            // the server's compensated start time
            AdjustAnimationTiming(result.skill_id, result.start_offset);
        } else {
            // Server rejected (on cooldown server-side, out of range, etc.)
            CancelCastBarAnimation(result.skill_id);
            ShowErrorMessage(result.rejection_reason);
        }
    }
};

// === SERVER: Skill Resolution Pipeline ===
struct ServerSkillPipeline {

    // Step 1: Receive and validate
    SkillResult ProcessSkillRequest(SkillRequest request,
                                    PlayerConnection& conn) {

        Player& caster = GetPlayer(request.player_id);

        // Validate prerequisites (server-authoritative)
        if (!caster.HasSkillUnlocked(request.skill_id))
            return Reject("Skill not available");
        if (caster.IsOnCooldown(request.skill_id))
            return Reject("Skill on cooldown");         // Common in WvW dilation
        if (!caster.HasRequiredResources(request.skill_id))
            return Reject("Insufficient resources");
        if (!IsTargetInRange(caster, request.target_id, request.skill_id))
            return Reject("Out of range");
        if (!HasLineOfSight(caster, request.target_id))
            return Reject("No line of sight");

        // Step 2: Latency compensation -- estimate input delay
        float input_delay = EstimateInputDelay(conn);

        // Step 3: Consume resources and start cooldown (server time)
        caster.ConsumeResources(request.skill_id);
        caster.StartCooldown(request.skill_id);  // Cooldown in game_time, not real_time

        // Step 4: Resolve the skill with fast-forwarding
        SkillAction action = CreateAction(request, input_delay);
        ResolveSkillEffect(action);

        // Step 5: Broadcast to all nearby clients
        BroadcastSkillAction(action, conn);

        return Accept(action);
    }

    // Step 6: Broadcast with per-recipient relay compensation
    void BroadcastSkillAction(SkillAction& action, PlayerConnection& caster_conn) {
        auto nearby_players = GetPlayersInRange(action.position, BROADCAST_RANGE);

        for (auto& recipient : nearby_players) {
            float relay_latency = EstimateOneWayLatency(recipient.connection);

            SkillActionPacket packet;
            packet.action       = action;
            packet.relay_offset = relay_latency;  // Client applies Stage 2

            recipient.connection.Send(packet);
        }
    }
};
```

### Steady vs. Unsteady Connections

MikeLewis noted that the compensation system works best with stable latency:

> "If ping time is steady, the LC experiment should make the connection feel a bit
> smoother. However, if ping time is unsteady or very high, there will be no
> noticeable improvement."

He used an analogy for the compensation tuning experiments:

> "If you add the sugar (latency experiments) to a small glass of water (good pings)
> you get sweet water. If you add the sugar to a swimming pool (bad pings) you won't
> even notice the sugar."

This confirms the system is designed as a polish layer for moderate-latency
connections, not a solution for high or unstable latency.

---

## Key Sources

### Primary Developer Sources

| Source | Author | Context |
|---|---|---|
| [HandmadeCon 2015 Interview Transcript](https://www.codeofhonor.com/blog/handmadecon-2015-interview-transcript) | Patrick Wyatt | TCP choice, 50K lines networking code, client-server authority model, movement, combat resolution |
| [Code of Honor Blog](https://www.codeofhonor.com/blog/) | Patrick Wyatt | Choosing game network libraries, scaling Guild Wars for massive concurrency |
| [Latency Tuning Experiments (Forum)](https://forum-en.gw2archive.eu/forum/support/support/Latency-Tuning-Experiments/) | MikeLewis (Lead Gameplay Programmer) | Two-stage fast-forward latency compensation, ping measurement, compensation tuning |
| [Official State of Skill Lag and Server Optimizations (Forum)](https://forum-en.gw2archive.eu/forum/game/gw2/Official-state-of-skill-lag-and-server-optimizations) | BillFreist (Gameplay Programmer) | Server bottlenecks, AoE target caps, threading model, optimization strategy |
| [GDC 2013: Guild Wars 2 -- Scaling from One to Millions](https://archive.org/details/GDC2013Willson) | Stephen Clarke-Willson | Server architecture, scaling from single machine to millions of players |
| [GDC 2017: Guild Wars Microservices and 24/7 Uptime](https://www.gdcvault.com/play/1024018/-Guild-Wars-Microservices-and) | Stephen Clarke-Willson | Microservices architecture since 2001, zero-downtime deployments |
| [GDC Online 2012: Programming the Next Generation Online World](https://gdcvault.com/play/1016640/Guild-Wars-2-Programming-the) | Cameron Dunn | Patching system, EC2 bot testing, memory optimization |
| [AWS Migration Blog Post](https://aws.amazon.com/blogs/gametech/arenanet-guild-wars-mmorpg-migration/) | AWS / ArenaNet | Migration from Dallas/Frankfurt data centers to AWS, first all-in Windows MMO on AWS |

### Community Analysis and Documentation

- [AoE Target Cap Discussion (Forum)](https://forum-en.gw2archive.eu/forum/game/gw2/Please-Explain-the-Logic-of-the-AoE-Limit/first) -- Community analysis of why the 5-target cap exists
- [Conditions Cause More Lag (WvW Forum)](https://forum-en.gw2archive.eu/forum/game/wuv/Conditions-cause-more-lag) -- Technical analysis of condition calculation cost
- [Culling Removal (2013)](https://wiki.guildwars2.com/wiki/Culling) -- Documentation of the transition from player culling to client-side LOD
- [Megaserver System (2014)](https://www.guildwars2.com/en/news/introducing-the-megaserver-system/) -- Weighted load-balancing map instance system
- [WvW Performance Improvements (MassivelyOP, 2020)](https://massivelyop.com/2020/07/11/guild-wars-2-looks-for-short-term-wvw-performance-improvements/) -- Population cap discussions
- [Server Tick Rate Discussion (Forum)](https://en-forum.guildwars2.com/topic/115452-unstable-server-tickrate/) -- Community analysis of tick rate instability
