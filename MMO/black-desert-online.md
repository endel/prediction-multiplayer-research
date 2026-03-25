# Black Desert Online: Aggressive Client Prediction and Desync as a Design Tradeoff

## Overview

**Developer:** Pearl Abyss (founded 2010, Anyang, South Korea)
**Engine:** Custom in-house engine (precursor to the modern "BlackSpace Engine" used in Crimson Desert)
**Launch:** KR 2014, NA/EU 2016, Console 2019
**Genre:** Action-combat MMORPG with seamless open world

Black Desert Online (BDO) represents a fundamentally different philosophy from most MMOs.
Where games like World of Warcraft or Final Fantasy XIV design gameplay *around* their
network limitations (tab-targeting, GCD windows, snapshot AoE), BDO goes the opposite
direction: it implements aggressive client-side prediction for *all* combat actions to
deliver a responsive, fighting-game-feel action combat system -- and then **explicitly
accepts desynchronization between players' views of the world** as the price of that
responsiveness.

This makes BDO one of the most instructive case studies in the MMO space. It demonstrates
both the power and the limits of client-side prediction at MMO scale, and it shows what
happens when a developer chooses "feels good to play" over "everyone sees the same thing."

> **Transparency note:** Pearl Abyss has disclosed very little about their internal
> architecture. The information below is assembled from official forum posts, developer
> announcements, community reverse-engineering, player network analysis, and inferences
> from observable behavior. Where something is confirmed by Pearl Abyss, it is noted.
> Everything else should be understood as community-estimated or inferred.

---

## Architecture

### Engine

BDO runs on Pearl Abyss's custom proprietary engine, originally built specifically for
the game's requirements: fast rendering of a seamless open world, large-scale sieges with
hundreds of characters, and fluid action combat animations. This engine later evolved into
the **BlackSpace Engine** showcased at GDC 2025, which powers Crimson Desert, DokeV, and
future Pearl Abyss titles.

The in-house engine gives Pearl Abyss full control over the networking layer -- but also
means the networking decisions made early in development (circa 2012-2013, targeting
Korean LAN-like internet) are deeply embedded in the architecture.

### Server Authority Model

BDO uses a **server-authoritative model with heavy client prediction**. The server is
the final arbiter of game state -- items, currency, experience, character stats, and
combat outcomes are all resolved server-side. However, the client is trusted to a
significant degree for:

- **Movement execution** -- the client moves immediately and the server validates
- **Combat action initiation** -- skills activate instantly on the client
- **Hit detection hints** -- the client reports what it believes it hit
- **Animation state** -- the client drives its own animation state machine locally

This hybrid model means the server has authority over *outcomes* but the client has
authority over *presentation*, creating inherent disagreements between what different
players see at any given moment.

### Estimated Tick Rate

Community analysis estimates BDO's server tick rate at approximately **20-30 Hz**
(~33-50ms per tick). This is notably lower than competitive shooters (Overwatch at
62.5 Hz, Valorant at 128 Hz) but comparable to many MMOs. The low tick rate is a
deliberate tradeoff for server scalability -- BDO servers must handle:

- Hundreds of concurrent players per channel in a seamless world
- 20-40% of connected players AFK (lifeskilling, autopathing) who still consume
  server resources
- Persistent world simulation (NPC schedules, node systems, trade routes)
- Large-scale PvP events (Node Wars, Siege) with 100-400+ combatants

| Parameter | Estimated Value | Source |
|-----------|----------------|--------|
| Server tick rate | ~20-30 Hz | Community analysis |
| Position update rate | ~20-30 Hz | Inferred from tick rate |
| Client input send rate | ~30-60 Hz | Community packet analysis |
| Protocol | UDP-based (with reliability layer) | Inferred from packet loss behavior |
| Seamless world | Yes (no loading screens between zones) | Confirmed |
| Channel system | Multiple channels per region, 5-min swap cooldown | Confirmed |

### Seamless Open World

BDO features a fully seamless open world with no loading screens during overworld travel.
The server architecture uses a **one-world-per-region system**: all of North America is
one world, all of Europe is another. Within each world, multiple **channels** (server
instances) run in parallel. Characters can swap channels with a 5-minute cooldown.

The seamless world likely uses spatial partitioning internally -- entities are managed by
the server zone responsible for their current position, with handoff occurring
transparently as players cross boundaries. This is invisible to the player but adds
complexity to the networking layer.

---

## Movement and Combat: Aggressive Client Prediction

### The Core Philosophy

Most MMOs treat combat networking conservatively: the client sends an intent ("I want to
cast Fireball"), the server validates and executes, and the client waits for confirmation
before showing the result. This introduces a round-trip delay visible to the player.

BDO takes the opposite approach: **the client predicts everything immediately**. When you
press a skill key, your character begins the animation, moves, applies super armor or
iframe protection, and even shows hit effects -- all before the server has processed the
action. The server then resolves what "actually" happened, and discrepancies are smoothed
over or silently resolved.

This aggressive prediction is what gives BDO its signature combat feel -- fluid,
responsive, and immediate -- but it is also the root cause of the desync that defines
PvP.

### How Client Prediction Works in BDO

```cpp
// PSEUDO-CODE: BDO-style aggressive client prediction for action combat
// NOTE: Speculative reconstruction based on observable behavior.
// Pearl Abyss has not published their actual implementation.

struct ActionInput {
    uint32_t sequence_id;        // Monotonically increasing input sequence
    uint32_t skill_id;           // Which skill was activated
    float    timestamp;          // Client-local time of activation
    Vector3  position;           // Client position at activation
    Vector3  direction;          // Facing direction
    uint32_t target_entity_id;   // Nearest valid target (if applicable)
};

struct PredictedState {
    Vector3  position;
    float    health;
    uint32_t active_skill_id;
    float    skill_progress;     // 0.0 to 1.0 through animation
    uint32_t protection_flags;   // SUPER_ARMOR, FRONTAL_GUARD, IFRAME, NONE
    uint32_t cc_state;           // STANDING, KNOCKED_DOWN, STUNNED, GRABBED, etc.
    bool     is_predicted;       // True until server confirms
};

class BDOClientPrediction {
    // Ring buffer of unacknowledged predicted actions
    CircularBuffer<ActionInput> pending_inputs;
    PredictedState              predicted_state;
    PredictedState              last_server_state;
    uint32_t                    last_ack_sequence;

    void OnPlayerInput(ActionInput input) {
        // === STEP 1: Immediate local execution ===
        // Do NOT wait for server. Apply instantly for responsiveness.
        predicted_state = SimulateAction(predicted_state, input);

        // Begin animation immediately
        PlaySkillAnimation(input.skill_id);

        // Apply protection frames immediately (super armor, iframe, etc.)
        ApplyProtectionState(input.skill_id, predicted_state);

        // Show hit effects if we predict a hit on a nearby target
        if (PredictHitOnTarget(input, predicted_state)) {
            ShowClientSideHitEffect(input.target_entity_id);
        }

        // === STEP 2: Send to server for authoritative resolution ===
        pending_inputs.Push(input);
        SendToServer(input);
    }

    void OnServerStateReceived(ServerSnapshot snapshot) {
        last_server_state = snapshot.state;
        last_ack_sequence = snapshot.last_processed_sequence;

        // Discard acknowledged inputs
        pending_inputs.DiscardUpTo(last_ack_sequence);

        // === STEP 3: Reconciliation ===
        // Unlike FPS games, BDO does NOT do full rewind-and-replay.
        // Instead, it uses a softer correction model:

        float position_error = Distance(predicted_state.position,
                                        snapshot.state.position);

        if (position_error > HARD_CORRECTION_THRESHOLD) {
            // Large desync: snap to server position (rubberband)
            predicted_state.position = snapshot.state.position;
        } else if (position_error > SOFT_CORRECTION_THRESHOLD) {
            // Moderate desync: blend toward server over several frames
            predicted_state.position = Lerp(predicted_state.position,
                                            snapshot.state.position,
                                            CORRECTION_BLEND_RATE);
        }
        // Small errors: ignore. The server state will converge naturally.

        // Re-apply unacknowledged inputs on top of corrected state
        PredictedState corrected = snapshot.state;
        for (auto& input : pending_inputs) {
            corrected = SimulateAction(corrected, input);
        }
        predicted_state = corrected;
    }
};
```

### What Gets Predicted (and What Doesn't)

| System | Predicted Client-Side? | Server-Authoritative? | Notes |
|--------|----------------------|----------------------|-------|
| Movement / position | Yes (instant) | Yes (validates) | Client moves first, server corrects |
| Skill activation | Yes (instant animation) | Yes (resolves outcome) | Client shows animation before server confirms |
| Protection states (SA/FG/iframe) | Yes (instant) | Yes (final arbiter) | Creates grab-vs-iframe conflicts |
| Hit detection | Yes (shows effects) | Yes (confirms damage) | Client sees hits that server may reject |
| Crowd control application | Partially | Yes | CC state transitions resolved server-side |
| Damage numbers | No (waits for server) | Yes | Actual damage shown after server confirms |
| Item/inventory changes | No | Yes | Fully server-authoritative |
| Currency/marketplace | No | Yes | Fully server-authoritative |

### The Afterimage Problem

A critical consequence of aggressive prediction is what the community calls the
**"afterimage" problem**: when you see another player on your screen, you are seeing where
they *were* approximately `your_latency + their_latency + server_processing_time` ago --
not where they are now. In a game where characters dash, teleport, and iframe constantly,
this means:

- The enemy you see standing still may have already started dashing on their client
- The enemy you see mid-animation may have already finished it and activated a protection
- The enemy you try to grab may have already iframed away on their client

```
Timeline: Two players with different latencies

Player A (30ms to server):
  T=0ms    T=30ms    T=60ms
  [Dash]--->[Server receives]--->[Other players see dash]

Player B (100ms to server):
  T=0ms    T=100ms   T=200ms
  [Dash]--->[Server receives]--->[Other players see dash]

What Player A sees of Player B:
  At T=0ms on A's screen, B appears to be where B was at T=-130ms
  (100ms B→server + 30ms server→A)

What Player B sees of Player A:
  At T=0ms on B's screen, A appears to be where A was at T=-130ms
  (30ms A→server + 100ms server→B)

Both players are fighting an afterimage of the other.
```

---

## Desync as a Design Tradeoff

### Why BDO Accepts Desync

Desync in BDO is not a bug -- it is a **deliberate architectural tradeoff**. Pearl Abyss
chose to prioritize combat responsiveness over visual consistency between players. The
reasoning:

1. **Action combat demands instant feedback.** If BDO waited for server confirmation
   before playing skill animations, every action would feel delayed by the player's
   round-trip time. At 60-100ms RTT, this makes combat feel sluggish. At 200ms+ (common
   for players far from servers), it would be unplayable as an action game.

2. **MMO scale makes alternatives infeasible.** Rollback netcode (as used in fighting
   games) requires resimulating the entire game state on correction. With hundreds of
   players and NPCs in a seamless world, this is computationally impossible. Server-side
   lag compensation (as used in Overwatch/Valorant) requires rewinding the world state
   per-shot, which is impractical at MMO entity counts.

3. **The Korean internet assumption.** BDO was designed for Korea, where average latency
   to game servers is under 10ms and packet loss is negligible. At Korean latencies,
   desync is barely noticeable. The architecture assumed this level of connectivity, which
   became problematic when the game launched globally.

4. **Packet handling design.** Community analysis has identified that BDO's netcode
   appears to assume reliable packet delivery without implementing robust retransmission.
   Lost packets are not always detected or resent, leading to state gaps. This "rookie
   mistake" (as community members describe it) would be invisible on Korean fiber
   connections but causes cascading failures on lossy connections.

### What Desync Looks Like in Practice

```
SCENARIO: Grab Desync (the most common PvP complaint)

Player A (Warrior, 40ms ping) attempts to grab Player B (Ninja, 80ms ping)

=== Player A's screen ===
T=0ms:   A sees B standing still after a skill
T=50ms:  A presses Grab, animation plays immediately
T=80ms:  Grab animation connects with B's character model
         A sees: "Grab successful!" -- B lifted into the air
         A begins followup damage combo

=== Player B's screen ===
T=0ms:   B finishes their skill, immediately presses iframe dash
T=10ms:  B sees themselves dashing away (iframe active)
T=120ms: B suddenly teleported back to A's position, in grabbed state
         B sees: "What?! I was already in iframe!"

=== Server's perspective ===
T=0ms:   B's skill ends (server-side)
T=40ms:  A's grab input arrives (40ms latency)
T=45ms:  Server processes grab -- B is still in post-skill recovery
         Grab connects. B is in grabbed state.
T=80ms:  B's iframe input arrives (80ms latency)
         Too late -- B is already grabbed. iframe input rejected.
T=85ms:  Server broadcasts: B is grabbed by A.

Result: A sees a clean grab. B sees desync -- they "reacted in time"
on their screen but the server had already resolved the grab.
```

```
SCENARIO: Mutual Kill / Ghost Damage

Player A and Player B attack each other simultaneously.

=== Server Timeline ===
T=0ms:   Server tick begins
T=2ms:   A's attack arrives -- hits B for lethal damage
T=3ms:   B's attack arrives -- hits A for lethal damage
         Both arrived in the same tick window.

Server resolves: Both attacks are valid.
                 Both players die.

=== Player A's screen ===
A sees their attack connect and B die.
Then A dies "from nowhere" because B's attack was already
in flight when A's kill resolved.

=== Player B's screen ===
Same experience in reverse. Both players feel cheated.
This is colloquially called "trading" and is unique to
BDO's aggressive prediction model at MMO scale.
```

### Factors That Amplify Desync

| Factor | Impact | Why |
|--------|--------|-----|
| High latency (>100ms) | Severe | Larger time window for conflicting actions |
| Packet loss (>1%) | Severe | BDO netcode does not robustly retransmit lost packets |
| WiFi connections | High | Introduces jitter and intermittent packet loss |
| Server load (Node Wars) | Extreme | Tick rate degrades under load, widening the desync window |
| Fast movement skills | High | Position changes faster than server can track |
| AFK players on channel | Moderate | Consume server resources, potentially affecting tick rate |

### Connection Stability vs. Raw Latency

A key insight from the BDO community: **consistent latency matters more than low
latency.** A player with stable 80ms ping will experience less noticeable desync than a
player with ping fluctuating between 20ms and 120ms. When latency is consistent, both the
human brain and BDO's prediction model can compensate for the delay. When it fluctuates,
predictions become unreliable and corrections become jarring.

---

## Grab vs. iFrame Conflicts: Server Resolution Logic

The most contentious desync scenario in BDO occurs when a **grab** (which pierces super
armor and frontal guard) collides with an **iframe** (which grants full invincibility).
These are the two "trump card" mechanics in BDO combat, and when they conflict, the
server must decide who wins.

### Protection Hierarchy

```
BDO Protection Priority (from strongest to weakest):

1. Invincibility (iframe)  -- immune to ALL damage and CC, including grabs
2. Grab                    -- pierces Super Armor and Frontal Guard
3. Super Armor             -- immune to all CC except grabs; reduced damage
4. Frontal Guard           -- immune to CC/damage from the front; vulnerable to grabs
5. No protection           -- fully vulnerable
```

### Server-Side Resolution

```cpp
// PSEUDO-CODE: Server resolution of conflicting combat actions
// Speculative reconstruction based on observed behavior and community analysis.

enum Protection {
    NONE          = 0,
    FRONTAL_GUARD = 1,
    SUPER_ARMOR   = 2,
    IFRAME        = 3,
    INVINCIBLE    = 4   // absolute invincibility (e.g., resurrection)
};

struct CombatAction {
    uint32_t   source_id;
    uint32_t   target_id;
    uint32_t   skill_id;
    float      server_arrival_time;
    bool       is_grab;
    bool       is_cc;
    Protection source_protection;  // attacker's protection during this skill
};

struct PlayerState {
    Protection   active_protection;
    uint32_t     active_skill_id;
    float        skill_start_time;
    float        skill_end_time;
    uint32_t     cc_state;          // STANDING, GRABBED, KNOCKED_DOWN, etc.
    Vector3      position;
};

// Called each server tick to resolve pending combat actions
void ResolveActions(vector<CombatAction>& actions, GameState& state) {
    // Sort by server arrival time (FIFO within a tick)
    SortByArrivalTime(actions);

    for (auto& action : actions) {
        PlayerState& target = state.GetPlayer(action.target_id);
        PlayerState& source = state.GetPlayer(action.source_id);

        // Skip if source is already CC'd (can't act while stunned/grabbed)
        if (source.cc_state != CC_STANDING && source.cc_state != CC_NONE) {
            continue;
        }

        // === GRAB VS PROTECTION RESOLUTION ===
        if (action.is_grab) {
            switch (target.active_protection) {
                case IFRAME:
                case INVINCIBLE:
                    // iframe wins -- grab fails entirely
                    // This is the "protection outcome" preference
                    RejectAction(action, "target_in_iframe");
                    NotifySource(action.source_id, GRAB_FAILED);
                    break;

                case SUPER_ARMOR:
                case FRONTAL_GUARD:
                    // Grab pierces SA and FG -- grab succeeds
                    ApplyGrab(source, target);
                    // Target's protection is overridden
                    target.active_protection = NONE;
                    target.cc_state = CC_GRABBED;
                    break;

                case NONE:
                    // No protection -- grab succeeds
                    ApplyGrab(source, target);
                    target.cc_state = CC_GRABBED;
                    break;
            }
        }
        // === STANDARD CC RESOLUTION ===
        else if (action.is_cc) {
            switch (target.active_protection) {
                case IFRAME:
                case INVINCIBLE:
                    RejectAction(action, "target_in_iframe");
                    break;

                case SUPER_ARMOR:
                    // CC blocked, but damage still applies (reduced)
                    ApplyDamage(action, target, DAMAGE_REDUCED);
                    break;

                case FRONTAL_GUARD:
                    if (IsFromFront(action, target)) {
                        // Blocked from front
                        ApplyBlockEffect(action, target);
                    } else {
                        // Side/rear attack bypasses frontal guard
                        ApplyCC(action, target);
                        ApplyDamage(action, target, DAMAGE_FULL);
                    }
                    break;

                case NONE:
                    ApplyCC(action, target);
                    ApplyDamage(action, target, DAMAGE_FULL);
                    break;
            }
        }
    }
}
```

### The Tick Boundary Problem

The most frustrating desync occurs when a grab and an iframe arrive in the **same server
tick**. Because the server processes actions in arrival order within a tick, the outcome
depends on which packet arrived first -- often a matter of single milliseconds and
network routing, not player skill.

```cpp
// PSEUDO-CODE: Same-tick conflict resolution

void ResolveSameTickConflict(CombatAction& grab_action,
                              CombatAction& iframe_action,
                              PlayerState& grabber,
                              PlayerState& target) {
    // Both actions arrived within the same server tick (~33-50ms window)
    // Server must decide: did the iframe activate before the grab landed?

    float time_delta = iframe_action.server_arrival_time
                     - grab_action.server_arrival_time;

    if (time_delta < 0) {
        // iframe arrived first -- protection was active when grab arrived
        // Result: grab FAILS, target is safe
        RejectAction(grab_action, "iframe_active");

    } else if (time_delta > 0) {
        // Grab arrived first -- target was not yet in iframe
        // Result: grab SUCCEEDS, iframe is overridden
        ApplyGrab(grabber, target);

    } else {
        // Exact same arrival time (extremely rare)
        // Historical behavior: favor the protective outcome (iframe wins)
        // NOTE: Community reports suggest this priority may have shifted
        // in some patches, with grabs being favored instead.
        RejectAction(grab_action, "simultaneous_protection_favored");
    }
}
```

> **Community note:** Players have reported that the grab-vs-iframe priority has changed
> across patches. In earlier versions of BDO, iframes were consistently favored in
> simultaneous conflicts ("protection outcome" preference). More recent reports suggest
> grabs may now take priority in ties, meaning a player can react to a grab with an iframe
> and still be grabbed because both inputs fell within the same server tick. This change,
> if intentional, makes grabbing stronger at the cost of increased perceived desync for
> the defending player.

---

## Server Relocation: NA Servers Move to Central US

### Background

From NA launch in 2016 until May 2024, BDO's North American servers were located on the
**US West Coast**. This was a legacy of the original publishing arrangement and Korean
development norms (proximity to trans-Pacific cables).

### The Data

Pearl Abyss published player distribution data that revealed a significant mismatch
between server location and player population:

| Region | % of NA Players |
|--------|----------------|
| Central & Eastern US/Canada | **70%** |
| Western US/Canada | 20% |
| Australia/NZ/Pacific Islands | 4% |
| Mexico, Europe, Other | 6% |

**70% of the player base** was playing from the Central and Eastern US, suffering
unnecessarily high latency to West Coast servers.

### The Move

On **May 22, 2024**, Pearl Abyss relocated NA servers to **North Central US**. The
expected impact on average latency:

| Server Location | Average Ping (all NA players) |
|----------------|------------------------------|
| US West Coast (before) | 62.44 ms |
| North Central US (after) | **44.08 ms** |
| South Central US (evaluated) | 46.81 ms |
| US East Coast (evaluated) | 48.01 ms |

The North Central location was chosen because it minimized average latency across the
entire player base, reducing it by approximately **18ms** (~29% improvement). For East
Coast players specifically, the improvement was even more dramatic -- potentially 30-50ms
reduction.

### Impact on Desync

The ~18ms average improvement directly reduces the desync window. In a game where grab
vs. iframe conflicts are resolved within a single tick (~33-50ms), an 18ms improvement
represents a significant reduction in the "gray zone" where both players' actions appear
valid on their respective screens.

West Coast players experienced a latency increase (roughly symmetric to East Coast
improvement), but since they represented only 20% of the player base, the tradeoff was
considered favorable.

---

## PvP Implications

### Small-Scale PvP (1v1, 3v3 Arena)

In small-scale PvP, desync is the defining skill differentiator beyond mechanical
execution. High-level BDO PvP players learn to:

- **Fight the afterimage**: Attack where the enemy *will be* based on server timing, not
  where they appear on screen
- **Activate protection early**: Use iframes and super armor preemptively, before you see
  the opponent's attack on your screen, because by the time you see it, the server may
  have already resolved it
- **Exploit skill startup**: CC opponents at the *start* of their skill animations, when
  their position is most likely to match the server state, rather than at the end when
  they may have already moved server-side
- **Minimize protection gaps**: Chain skills to eliminate frames without protection, since
  those gaps are where desync grabs connect

```
// PSEUDO-CODE: Experienced BDO player's desync-aware combat decision model

struct CombatDecision {
    float    estimated_opponent_latency;  // inferred from behavior patterns
    float    my_latency;
    float    desync_window;              // sum of both latencies + tick time

    Action ChooseAction(EnemyState observed_enemy, float server_tick_ms) {
        desync_window = my_latency + estimated_opponent_latency + server_tick_ms;

        // The enemy's position on my screen is outdated by desync_window ms
        Vector3 predicted_server_position = ExtrapolatePosition(
            observed_enemy.position,
            observed_enemy.velocity,
            desync_window * 0.001f  // convert ms to seconds
        );

        // If enemy appears to be in recovery (no protection), they may
        // ALREADY have activated a protection skill server-side
        if (observed_enemy.apparent_state == RECOVERY) {
            float time_since_skill_end = CurrentTime() - observed_enemy.skill_end;

            if (time_since_skill_end < desync_window) {
                // DANGER: They might already be protected server-side.
                // Use a protected engage (super armor skill) instead of grab.
                return Action::PROTECTED_ENGAGE;
            } else {
                // Safe window: they've been in recovery longer than
                // the desync window. Grab is likely to succeed.
                return Action::GRAB;
            }
        }

        // If enemy is in iframe on my screen, they may have already
        // exited it server-side
        if (observed_enemy.apparent_state == IFRAME) {
            // Wait for iframe to end on-screen, then add desync_window
            // before attempting CC
            return Action::WAIT_AND_PUNISH;
        }

        return Action::PROTECTED_ENGAGE;
    }
};
```

### Large-Scale PvP: Node Wars and Siege

Node Wars (40-100 players per side) and Siege Warfare (potentially hundreds of players)
represent the extreme stress test of BDO's networking model. Under high player counts, the
system degrades significantly.

#### The Valencia Siege Incident

The most dramatic example of network failure occurred during what was reportedly the
largest siege in BDO history across all regions: **27 guilds placed forts in Valencia**,
with over **1,400 players** on a single battlefield. When the siege began:

- **95% of participants were disconnected** from the game
- For the next **90 minutes**, gameplay consisted of logging in, walking a few meters, and
  being disconnected instantly
- The server could not load player names or family names, displaying only class names
- Players experienced teleportation, rubberbanding, and complete ability failure
- Potions did not work for 90 minutes
- Skills did not register or took seconds to activate

#### Why Large-Scale PvP Breaks

```cpp
// PSEUDO-CODE: Server tick degradation under load

struct ServerTick {
    float    target_tick_rate = 30.0f;  // target: 30 Hz
    float    target_tick_ms   = 33.3f;  // ~33ms per tick
    float    actual_tick_ms;            // what actually happens

    void SimulateTick(int player_count) {
        // Cost scales with player interactions, not just player count
        // N players can generate up to N*(N-1)/2 interaction pairs
        int max_interactions = player_count * (player_count - 1) / 2;

        // Each interaction requires:
        // - Position validation for both players
        // - Protection state checks
        // - Hit detection (spatial queries)
        // - CC resolution
        // - Damage calculation
        // - State broadcast to relevant players
        float processing_time_ms = EstimateProcessingTime(max_interactions);

        // Normal gameplay (50 players in an area): ~20ms, fits in tick budget
        // Node War (200 players):  ~150ms, tick rate drops to ~6 Hz
        // Massive siege (400+):    ~500ms+, tick rate drops to ~2 Hz
        // Valencia incident (1400): tick budget exceeded by 10x+, cascading failure

        actual_tick_ms = max(target_tick_ms, processing_time_ms);
        float effective_tick_rate = 1000.0f / actual_tick_ms;

        // When tick rate drops below ~5 Hz, the system enters a death spiral:
        // - Actions queue faster than they can be processed
        // - Memory pressure increases
        // - Connection timeouts trigger disconnections
        // - Disconnection handling adds more server work
        if (effective_tick_rate < 5.0f) {
            TriggerDegradedMode();
            // Degraded mode: drop non-essential packets, reduce broadcast radius
            // But in extreme cases, even this is insufficient
        }
    }
};
```

#### Network Optimizations for Large-Scale PvP

Pearl Abyss has implemented several optimizations over the years, though specifics are
limited:

```cpp
// PSEUDO-CODE: Siege/Node War network optimizations (speculative)

struct LargeScalePvPOptimizations {
    // 1. Area-of-Interest (AoI) culling
    // Only send updates about players within a radius
    float aoi_radius = 100.0f;  // meters; reduced under load
    float aoi_radius_degraded = 50.0f;

    // 2. Update priority system
    // Players near you get higher update priority
    float GetUpdatePriority(Player& self, Player& other) {
        float distance = Distance(self.position, other.position);
        float priority = 1.0f / (distance + 1.0f);

        // Enemies in combat with you get highest priority
        if (other.IsInCombatWith(self))
            priority *= 10.0f;

        // Guild members get elevated priority
        if (other.guild_id == self.guild_id)
            priority *= 2.0f;

        return priority;
    }

    // 3. Packet batching and compression
    // Combine multiple state updates into single packets
    void BatchUpdates(vector<StateUpdate>& updates) {
        // Sort by priority, send top N per tick
        SortByPriority(updates);
        int budget = MAX_UPDATES_PER_TICK;  // reduced under load

        for (int i = 0; i < min(budget, updates.size()); i++) {
            CompressAndSend(updates[i]);
        }
        // Lower priority updates deferred to next tick
    }

    // 4. Visual simplification under load
    // Reduce visual effects, animation detail for distant players
    void ApplyLODNetworking(Player& viewer, Player& target, float distance) {
        if (distance > 30.0f) {
            // Reduce animation update rate
            target.SetNetAnimRate(viewer, 10.0f);  // 10 Hz instead of 30
        }
        if (distance > 60.0f) {
            // Only send position, no animation data
            target.SetNetMode(viewer, POSITION_ONLY);
        }
    }
};
```

#### 2024 Node War Revamp

In May 2024, Pearl Abyss launched a **Node War reorganization** alongside the NA server
relocation. Key changes included:

- **Streamlined participation** -- reduced preparation overhead to encourage more guilds
- **Better packet optimization** -- removal of unnecessary data packets to reduce server
  load (partially implemented, with further improvements planned)
- **New UI systems** -- better battlefield awareness to compensate for networking
  limitations
- The goal was lighter preparations and fiercer battles, acknowledging that raw server
  performance in massive engagements remains a fundamental challenge

---

## Action Combat Input Pipeline: From Keypress to Server Validation

The full lifecycle of a combat action in BDO, from input to resolution:

```cpp
// PSEUDO-CODE: Full action combat pipeline

// ============================================================
// STAGE 1: CLIENT - Input Processing (~0ms, instant)
// ============================================================

void Client::OnKeyPress(KeyEvent event) {
    Skill* skill = GetSkillForInput(event);
    if (!skill) return;

    // Validate locally: is the skill off cooldown? Do we have resources?
    if (!CanActivateSkill(skill, local_player)) return;

    // Create the action packet
    ActionPacket packet;
    packet.sequence      = next_sequence_id++;
    packet.skill_id      = skill->id;
    packet.timestamp     = GetClientTime();
    packet.position      = local_player.position;
    packet.rotation      = local_player.rotation;
    packet.target_id     = GetSoftTarget();

    // PREDICT IMMEDIATELY: don't wait for server
    PredictAction(packet);

    // Send to server
    network.Send(packet);
}

void Client::PredictAction(ActionPacket& packet) {
    Skill* skill = GetSkill(packet.skill_id);

    // Start animation
    animator.PlaySkill(skill, local_player);

    // Apply protection state
    local_player.SetProtection(skill->protection_type);
    // e.g., SUPER_ARMOR for Warrior's Grave Digging
    //        IFRAME for Ninja's Shadow Stomp
    //        FRONTAL_GUARD for Valkyrie's Shield Chase

    // Apply movement (many skills move the character)
    local_player.position += skill->GetMovementVector(local_player.rotation);

    // Predict hits on nearby enemies
    auto targets = SpatialQuery(local_player.position, skill->range, skill->arc);
    for (auto* target : targets) {
        // Show client-side hit effect for responsiveness
        PlayHitEffect(target, skill);
        // NOTE: Actual damage will be applied when server confirms
    }

    // Store prediction for later reconciliation
    prediction_history.Store(packet.sequence, local_player.GetState());
}

// ============================================================
// STAGE 2: NETWORK TRANSIT (~20-100ms depending on latency)
// ============================================================
// Packet traverses: Client -> ISP -> Internet -> Server datacenter
// BDO appears to use UDP with a custom reliability layer
// Lost packets may not be retransmitted in all cases

// ============================================================
// STAGE 3: SERVER - Validation and Resolution (~1-5ms processing)
// ============================================================

void Server::OnActionPacketReceived(ActionPacket& packet, Connection& conn) {
    Player* player = GetPlayer(conn);

    // --- Validation Phase ---

    // 1. Anti-cheat: is this action physically possible?
    if (!ValidateSkillAvailability(player, packet.skill_id)) {
        RejectAction(conn, "skill_unavailable");
        return;
    }

    // 2. Position validation: is the client's reported position
    //    within acceptable range of server's known position?
    float pos_delta = Distance(player->server_position, packet.position);
    if (pos_delta > MAX_POSITION_TOLERANCE) {
        // Possible speed hack or extreme desync
        CorrectPosition(player, player->server_position);
        LogSuspiciousActivity(conn, "position_mismatch", pos_delta);
        return;
    }

    // 3. Cooldown validation
    if (!IsSkillOffCooldown(player, packet.skill_id)) {
        RejectAction(conn, "skill_on_cooldown");
        return;
    }

    // 4. Resource validation (mana/stamina/rage)
    if (!HasSufficientResources(player, packet.skill_id)) {
        RejectAction(conn, "insufficient_resources");
        return;
    }

    // --- Execution Phase ---

    Skill* skill = GetSkill(packet.skill_id);

    // Update server-side player state
    player->active_skill = skill;
    player->protection = skill->protection_type;
    player->skill_start_time = GetServerTime();
    player->server_position += skill->GetMovementVector(player->rotation);

    // Perform server-side hit detection
    auto targets = SpatialQuery(player->server_position, skill->range, skill->arc);
    for (auto* target : targets) {
        ResolveCombatInteraction(player, target, skill);
    }

    // Queue state update for broadcast
    BroadcastStateUpdate(player);
}

// ============================================================
// STAGE 4: SERVER - Combat Interaction Resolution
// ============================================================

void Server::ResolveCombatInteraction(Player* attacker, Player* target, Skill* skill) {
    // Check target's current protection state (server-side truth)
    Protection target_prot = target->protection;

    if (target_prot == IFRAME || target_prot == INVINCIBLE) {
        // Target is invulnerable -- attack has no effect
        return;
    }

    if (skill->is_grab) {
        if (target_prot == IFRAME || target_prot == INVINCIBLE) {
            return;  // Grab fails against iframe
        }
        // Grab pierces Super Armor and Frontal Guard
        ApplyGrab(attacker, target);
        return;
    }

    if (target_prot == SUPER_ARMOR) {
        // CC blocked, reduced damage applied
        ApplyDamage(attacker, target, skill, REDUCED_MODIFIER);
        return;
    }

    if (target_prot == FRONTAL_GUARD) {
        if (IsAttackFromFront(attacker, target)) {
            ApplyBlockDamage(attacker, target, skill);
            return;
        }
        // Side/rear attack bypasses frontal guard
    }

    // No applicable protection -- full effect
    if (skill->cc_type != CC_NONE) {
        ApplyCC(attacker, target, skill->cc_type);
    }
    ApplyDamage(attacker, target, skill, FULL_MODIFIER);
}

// ============================================================
// STAGE 5: BROADCAST - State Update to All Relevant Clients
// ============================================================

void Server::BroadcastStateUpdate(Player* player) {
    StateUpdate update;
    update.entity_id    = player->id;
    update.position     = player->server_position;
    update.rotation     = player->rotation;
    update.active_skill = player->active_skill ? player->active_skill->id : 0;
    update.protection   = player->protection;
    update.cc_state     = player->cc_state;
    update.health_pct   = player->health / player->max_health;

    // Send to all players within Area of Interest
    for (auto* viewer : GetPlayersInAoI(player->server_position, AOI_RADIUS)) {
        viewer->connection->Send(update);
    }
}

// ============================================================
// STAGE 6: CLIENT - Reconciliation (receiving other players' states)
// ============================================================

void Client::OnRemotePlayerUpdate(StateUpdate& update) {
    RemotePlayer* remote = GetRemotePlayer(update.entity_id);

    // Interpolate remote player's position for smooth visual
    remote->interp_start_pos = remote->display_position;
    remote->interp_end_pos   = update.position;
    remote->interp_timer     = 0.0f;
    remote->interp_duration  = GetInterpolationDuration();

    // Update remote player's apparent state
    remote->apparent_protection = update.protection;
    remote->apparent_cc_state   = update.cc_state;
    remote->apparent_skill      = update.active_skill;

    // NOTE: What you SEE on screen for other players is always
    // in the past. This is the source of the "afterimage" problem.
}
```

---

## Position Reconciliation with Visual Desync

When the server corrects a player's position, the client must reconcile the predicted
position with the authoritative one. BDO uses a tiered correction system:

```cpp
// PSEUDO-CODE: Position reconciliation with visual smoothing

class PositionReconciliation {
    // Thresholds (estimated from observed behavior)
    static constexpr float IGNORE_THRESHOLD     = 0.5f;   // meters
    static constexpr float BLEND_THRESHOLD      = 3.0f;   // meters
    static constexpr float SNAP_THRESHOLD       = 10.0f;  // meters
    static constexpr float TELEPORT_THRESHOLD   = 50.0f;  // meters
    static constexpr float BLEND_SPEED          = 8.0f;   // meters per second

    void ReconcilePosition(Player& player, Vector3 server_position) {
        float error = Distance(player.predicted_position, server_position);

        if (error < IGNORE_THRESHOLD) {
            // Negligible difference. Do nothing.
            // The natural convergence of prediction will resolve this.
            return;
        }

        if (error < BLEND_THRESHOLD) {
            // Moderate error: smoothly blend toward server position.
            // This is invisible to the player if done quickly.
            player.correction_target = server_position;
            player.correction_active = true;
            player.correction_speed  = BLEND_SPEED;
            // Visual position will lerp over ~2-4 frames
            return;
        }

        if (error < SNAP_THRESHOLD) {
            // Large error: noticeable "rubberband" snap.
            // Player will see their character jump back.
            // This is the classic desync rubberband.
            player.predicted_position = server_position;
            player.correction_active = false;
            PlayRubberbandEffect(player);
            return;
        }

        if (error < TELEPORT_THRESHOLD) {
            // Very large error: teleport with brief screen fade.
            player.predicted_position = server_position;
            PlayTeleportTransition(player);
            return;
        }

        // Extreme error (>50m): likely a hack or catastrophic desync.
        // Disconnect or force full state reload.
        ForceFullStateSync(player);
    }

    // Called every frame during active correction
    void UpdateCorrection(Player& player, float dt) {
        if (!player.correction_active) return;

        float remaining = Distance(player.display_position,
                                    player.correction_target);

        if (remaining < 0.01f) {
            player.display_position = player.correction_target;
            player.correction_active = false;
            return;
        }

        // Smooth interpolation toward corrected position
        float step = player.correction_speed * dt;
        player.display_position = MoveToward(player.display_position,
                                              player.correction_target,
                                              step);
    }
};
```

---

## Desync Scenario Walkthrough: Two Players See Different Outcomes

This walkthrough demonstrates the complete lifecycle of a desync event, showing exactly
how two players can see fundamentally different outcomes for the same combat exchange.

```
FULL DESYNC SCENARIO WALKTHROUGH
=================================

Setup:
  Player A: Warrior (40ms ping), using Grave Digging (Super Armor)
  Player B: Ninja (90ms ping), using Shadow Stomp (iFrame) into Grab

=== T=0ms: Player B's Client ===
  B presses Shadow Stomp (iframe dash)
  B's client: iframe activates immediately, B dashes forward
  B's client sends packet to server

=== T=10ms: Player B's Client ===
  B chains into Grab (during iframe transition)
  B sees A standing in Grave Digging animation
  B's client: Grab animation plays, appears to connect with A
  B sees: "I grabbed A during their super armor skill!"

=== T=40ms: Player A's Client ===
  A is mid-Grave Digging (Super Armor active)
  A sees B standing still (B's iframe hasn't rendered yet for A)
  A continues attacking -- sees damage numbers on nearby mobs

=== T=40ms: Server ===
  A's latest state: mid-Grave Digging, Super Armor active
  (A's inputs arrive at 40ms latency -- server has current state)

=== T=90ms: Server ===
  B's Shadow Stomp packet arrives (90ms latency)
  Server processes: B enters iframe
  But server also checks: where was A when B cast Shadow Stomp?
  A was in Super Armor -- B's iframe doesn't affect A's state

=== T=100ms: Server ===
  B's Grab packet arrives
  Server checks A's state: A is in Super Armor (Grave Digging)
  Grab pierces Super Armor? YES -- Grab succeeds!
  Wait -- but A has been continuously attacking, and by now A's
  Grave Digging may have ended. Exact timing depends on animation frames.

  SCENARIO BRANCH A: Grave Digging still active on server
    Grab connects! A is grabbed.
    Server broadcasts: A is in grabbed state.

  SCENARIO BRANCH B: Grave Digging ended 20ms ago on server
    A has entered next skill (with frontal guard).
    Grab still pierces frontal guard. Grab connects!

  SCENARIO BRANCH C: Grave Digging ended and A is in iframe
    A used an iframe skill in the gap. Grab FAILS.
    Server broadcasts: Grab failed, B recovers.

=== T=130ms: Player A's Client (Branch A) ===
  A receives server update: "You are grabbed by B"
  A was mid-attack animation. Suddenly snapped into grab animation.
  A sees: "B teleported onto me and grabbed me through my Super Armor!"
  (A never saw B's iframe approach because B's packets hadn't rendered)

=== T=180ms: Player B's Client (Branch C) ===
  B receives server update: "Grab failed, target in iframe"
  B was mid-grab-followup animation. Suddenly, A escapes.
  B sees: "My grab connected on my screen! This is desync!"

Both players are correct from their perspective. Neither is cheating.
The server resolved based on the timing of packet arrival.
This is desync as a design tradeoff.
```

---

## Key Sources

> Pearl Abyss has disclosed very little about BDO's internal networking architecture. The
> following sources represent the best available public information, combining official
> statements, community analysis, and observable behavior.

### Official / Pearl Abyss

- [NA Server Relocation Announcement (May 2024)](https://www.naeu.playblackdesert.com/en-US/News/Detail?groupContentNo=6926&countryType=en-US) -- Confirmed player distribution statistics (70% East/Central), ping data, and relocation rationale
- [BDO Node Wars Revamp (May 2024)](https://pressreleases.triplepointpr.com/2024/05/22/black-desert-online-revamps-large-scale-pvp-battles-with-streamlined-node-wars/) -- Official press release on large-scale PvP improvements
- [BlackSpace Engine Dev Archives](https://crimsondesert.pearlabyss.com/en-us/News/Notice/Detail?_boardNo=40) -- Technical showcase of Pearl Abyss's engine (successor to BDO's engine)
- [BDO Combat Overhaul (2025)](https://www.mmorpg.com/news/black-desert-onlines-huge-combat-overhaul-with-every-class-affected-is-live-2000135594) -- Major combat system changes affecting CC, protection, and class balance

### Community Analysis and Forums

- [Desync Guide -- BDO NA/EU Forums](https://www.naeu.playblackdesert.com/en-us/Forum/ForumTopic/Detail?_topicNo=58537&_opinionNo=0) -- Comprehensive community-written guide explaining desync mechanics, tick behavior, grab-vs-iframe resolution, and mitigation strategies
- [Why Desync is Still an Issue -- BDO NA/EU Forums](https://www.naeu.playblackdesert.com/en-US/Forum/ForumTopic/Detail?_topicNo=7150) -- Technical critique of BDO's packet handling, Korean internet assumptions, and retransmission deficiencies
- [Disappointed With PVP State -- BDO Console Forums](https://www.console.playblackdesert.com/Community/Detail?topicNo=5490&topicType=6) -- Console tick rate analysis and PvP impact discussion
- [Combat Framework Reboot Discussion -- BDO NA/EU Forums](https://www.naeu.playblackdesert.com/en-US/Forum/ForumTopic/Detail?_topicNo=19132) -- Community analysis of engine limitations, FPS exploits, and architectural constraints
- [BDO Crowd Control Guide -- mmosumo](https://mmosumo.com/black-desert-online-crowd-control-guide/) -- Detailed CC mechanics, protection states, and interaction rules

### Press and Industry Coverage

- [Sieges Besieged by Persistent Lag -- MMORPG.com](https://www.mmorpg.com/news/sieges-besieged-by-persistent-lag-and-connection-issues-2000098094) -- Coverage of the Valencia siege incident and large-scale PvP server failures
- [BDO NA Server Move -- MMORPG.com](https://www.mmorpg.com/news/black-desert-online-will-move-na-servers-for-better-performance-adds-north-kamasylvia-to-mobile-2000131467) -- Press coverage of the server relocation decision
- [Pearl Abyss BlackSpace Engine -- GDC 2025](https://www.mmorpg.com/features/gdc-2025-we-saw-under-the-hood-of-crimson-deserts-engine-here-are-some-of-our-takeaways-2000134455) -- Under-the-hood look at Pearl Abyss's engine technology
- [BDO Anti-Cheat Architecture](https://foxtailinsights.com/2023/04/20/how-does-black-desert-online-s-anti-cheat-software-work/) -- Analysis of server-side validation and client monitoring

### Technical Context

- [BDO Wikipedia](https://en.wikipedia.org/wiki/Black_Desert_Online) -- General reference for engine, launch history, and game architecture
- [Pearl Abyss Engine Interview -- Inven Global](https://www.invenglobal.com/articles/6046/interview-with-the-lead-engine-programmer-in-pearl-abyss-on-black-desert-online-graphic-remaster) -- Lead engine programmer discusses in-house engine philosophy
