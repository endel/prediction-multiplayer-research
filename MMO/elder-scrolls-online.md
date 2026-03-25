# Elder Scrolls Online -- Netcode and Megaserver Architecture

> **Disclaimer:** ZeniMax Online Studios has been notably secretive about the
> internal technical details of ESO's networking and server architecture. Much of
> the information below is assembled from official blog posts, patch notes,
> developer forum comments, community network analysis, and informed inference by
> experienced network engineers in the player community. Where a claim is
> unconfirmed, it is marked as such.

---

## 1. Overview

The Elder Scrolls Online (ESO), launched in 2014, is a massively multiplayer
online role-playing game developed by ZeniMax Online Studios. Its networking
architecture is built around a proprietary **megaserver** system -- a logical
abstraction that presents a single server endpoint per region (North America and
Europe) while internally orchestrating a fleet of physical servers, databases,
and zone instance processes.

ESO is unusual among modern online games in its reliance on **TCP** as the
primary game transport protocol. While most real-time multiplayer games favor UDP
(or custom reliable-UDP layers) for low-latency state replication, ESO routes
gameplay traffic over TCP connections on port ranges 24100-24131 and 24300-24331
(both TCP and UDP are opened, but community packet analysis and the game's
behavior under packet loss strongly suggest TCP carries the critical gameplay
stream).

The game uses a **server-authoritative** model: the server maintains ground truth
for all game state, and client-side visuals are largely presentation-layer
predictions that the server can override. ESO's server tick rate is
**community-estimated at approximately 50-64 Hz** (roughly 15-20 ms per tick),
based on in-game timing analysis of the 1-second Global Cooldown (GCD) and
observed update granularity. ZeniMax has never officially confirmed the tick rate.

### Key characteristics

| Property                   | Detail                                                             |
|----------------------------|--------------------------------------------------------------------|
| **Transport protocol**     | TCP (primary), with some UDP for auxiliary traffic                  |
| **Server authority**       | Fully server-authoritative                                         |
| **Server tick rate**       | ~50-64 Hz (community estimate, unconfirmed)                        |
| **Global Cooldown**        | 1.0 second (server-enforced)                                       |
| **Megaservers**            | 2 regional (NA in Texas, EU in Germany)                            |
| **Cyrodiil pop cap**       | Originally ~200 per faction (600 total); reduced over time         |
| **Zone instancing**        | Dynamic shard/channel system per zone                              |

---

## 2. Architecture

### The megaserver concept

In a traditional MMO, players choose a named server (or "realm") at character
creation. Each realm runs its own isolated world with a fixed population cap.
ESO's megaserver eliminates this choice entirely: players select only a region
(NA or EU), and the megaserver handles all routing, instancing, and load
balancing transparently.

As ESO Game Director **Matt Firor** explained in a [Game Developer
interview](https://www.gamedeveloper.com/production/-i-elder-scrolls-online-i-dev-speaks-to-the-power-of-megaservers-in-mmo-game-design):

> "Once you're on the North American server, you never pick another server. The
> game figures out how many instances of each zone to spin up, and which one to
> put you in."

Internally, the megaserver is a **cluster of physical servers** working
together as a single logical unit. Each zone in the game world runs as one or
more **shard instances** (also called "channels"), with the megaserver
orchestrating:

- How many instances of each zone are active
- Which instance a player is assigned to upon entering a zone
- Friend/guild affinity routing (placing social connections together)
- Quest phasing layers within each instance

### Physical infrastructure

- **NA megaserver:** Hosted in a datacenter in Texas, USA
- **EU megaserver:** Hosted in a datacenter in Germany
- **Hardware refresh (2022):** ZeniMax replaced the original 2012-era server
  hardware for the NA megaserver in May 2022. The EU datacenter received a
  similar refresh later. The upgrade was delayed from 2021 due to the
  COVID-19-era hardware shortage.
- **PTS (Public Test Server):** Runs on separate, upgraded hardware and is often
  used to validate infrastructure changes before they reach live servers.

### High-level architecture diagram

```
                        ┌─────────────────────────────────────┐
                        │         Megaserver Cluster           │
                        │                                     │
   Player A ──TCP──►    │  ┌──────────┐   ┌───────────────┐   │
   Player B ──TCP──►    │  │  Gateway  │──►│ Zone Router   │   │
   Player C ──TCP──►    │  │  Servers  │   │ (Affinity +   │   │
                        │  └──────────┘   │  Load Balance) │   │
                        │                  └───────┬───────┘   │
                        │          ┌───────────────┼────────┐  │
                        │          ▼               ▼        ▼  │
                        │   ┌───────────┐  ┌──────────┐ ┌─────┐│
                        │   │ Zone Inst │  │Zone Inst │ │ ... ││
                        │   │ Grahtwood │  │Grahtwood │ │     ││
                        │   │ Shard #1  │  │Shard #2  │ │     ││
                        │   └─────┬─────┘  └────┬─────┘ └─────┘│
                        │         └──────┬───────┘              │
                        │                ▼                      │
                        │   ┌──────────────────────────┐       │
                        │   │  Shared Database Cluster  │       │
                        │   │  (Character, Inventory,   │       │
                        │   │   Guild, Achievements)    │       │
                        │   └──────────────────────────┘       │
                        └─────────────────────────────────────┘
```

### TCP: an unusual choice for an MMO

Most modern MMOs and real-time games use UDP or reliable-UDP because TCP's
guarantees come with costs that are punishing in real-time contexts:

| TCP Property                  | Benefit                        | Cost in a real-time game                    |
|-------------------------------|--------------------------------|---------------------------------------------|
| Guaranteed delivery           | No lost messages               | Retransmission stalls on packet loss        |
| In-order delivery             | Simple application logic       | Head-of-line blocking                       |
| Congestion control            | Fair bandwidth sharing         | Throughput reduction spikes under loss       |
| Connection-oriented           | Stateful sessions              | Higher overhead per player                  |

ESO's use of TCP means that a single dropped packet can stall the delivery of
all subsequent packets on that connection until the retransmission completes.
This is the **head-of-line (HOL) blocking** problem, and it becomes the defining
networking challenge in ESO's large-scale PvP (see Section 4).

---

## 3. Movement and Combat

### Action combat over TCP

ESO's combat system is described as "action combat" -- a hybrid between
traditional MMO tab-targeting and real-time action game mechanics. Players aim
abilities, dodge roll, block, interrupt, and weave light attacks between ability
casts. This system demands responsive input handling, yet it runs over TCP.

### Movement prediction

ESO uses **client-side movement prediction** with server reconciliation. The
client immediately applies player movement inputs locally (WASD, sprinting,
dodge rolling) and sends movement commands to the server. The server validates
these against its authoritative world state and either confirms or corrects them.

Other players' positions are received from the server and **interpolated** on
the client to smooth out the discrete updates received each tick. Under normal
conditions (~50-100 ms ping), this produces acceptably smooth movement. Under
high latency or packet loss, TCP retransmissions cause visible rubber-banding as
the client receives delayed corrections.

```cpp
// Pseudo-code: ESO-style client movement prediction over TCP

struct MovementInput {
    uint32_t sequence_number;
    Vec3     direction;
    float    speed;
    bool     sprinting;
    bool     dodge_roll;
    double   client_timestamp;
};

// CLIENT SIDE
void Client::ProcessLocalMovement(const Input& input) {
    MovementInput move;
    move.sequence_number = next_sequence_++;
    move.direction       = input.GetMoveDirection();
    move.speed           = CalculateSpeed(input);
    move.sprinting       = input.IsSprinting();
    move.dodge_roll      = input.IsDodgeRoll();
    move.client_timestamp = GetCurrentTime();

    // Apply prediction immediately for responsive feel
    local_player_.position += move.direction * move.speed * delta_time_;

    // Buffer the input for reconciliation
    pending_inputs_.push_back(move);

    // Send to server over TCP (guaranteed delivery, ordered)
    tcp_connection_.Send(Serialize(move));
}

// When server confirms position:
void Client::OnServerPositionUpdate(const ServerState& state) {
    local_player_.position = state.authoritative_position;

    // Remove acknowledged inputs
    pending_inputs_.erase_up_to(state.last_processed_sequence);

    // Re-apply any inputs the server hasn't processed yet
    for (auto& input : pending_inputs_) {
        local_player_.position += input.direction * input.speed * TICK_DURATION;
    }
}
```

### The Global Cooldown (GCD) system

ESO's combat revolves around a server-enforced **1-second Global Cooldown**. The
GCD governs how frequently a player can activate abilities. There are multiple
GCD tracks that operate semi-independently:

1. **Active abilities:** 1.0 second GCD per cast
2. **Light/heavy attacks:** Separate 1.0 second GCD track
3. **Bash:** Up to 3 per GCD window
4. **Synergies:** 1.0 second GCD, plus individual cooldowns

Because the GCD is server-enforced, the effective ability activation rate is
sensitive to round-trip latency. A player with 50 ms ping can reliably fire an
ability every 1.0 seconds. A player with 200 ms ping may effectively experience
~1.2 second intervals because the server must receive, validate, and acknowledge
each activation before the next can begin.

### Ability weaving (light attack canceling)

"Weaving" is the practice of firing a light attack and immediately canceling its
animation with an ability cast. Because light attacks and abilities occupy
**separate GCD tracks**, both can register within the same 1-second window --
essentially doubling the player's output.

The timing is tight and **highly sensitive to latency**:

```cpp
// Pseudo-code: Ability weaving timing sensitivity

struct AbilityInput {
    uint32_t ability_id;
    uint32_t sequence;
    double   client_timestamp;
    bool     is_light_attack;
};

// SERVER SIDE: Validate weave timing
struct GCDState {
    double light_attack_gcd_expires;  // When the LA track becomes available
    double ability_gcd_expires;       // When the ability track becomes available
};

ValidationResult Server::ValidateWeaveInput(
    const Player& player,
    const AbilityInput& input,
    double server_time
) {
    GCDState& gcd = player.gcd_state;

    if (input.is_light_attack) {
        // Light attack must wait for its own GCD track
        if (server_time < gcd.light_attack_gcd_expires) {
            return ValidationResult::REJECTED_GCD_ACTIVE;
        }
        gcd.light_attack_gcd_expires = server_time + LIGHT_ATTACK_GCD; // ~1.0s
        return ValidationResult::ACCEPTED;
    } else {
        // Ability must wait for the ability GCD track
        if (server_time < gcd.ability_gcd_expires) {
            return ValidationResult::REJECTED_GCD_ACTIVE;
        }
        gcd.ability_gcd_expires = server_time + ABILITY_GCD; // ~1.0s
        return ValidationResult::ACCEPTED;
    }
}

// KEY INSIGHT: The client sends a light attack, then immediately sends an
// ability. Both arrive at the server within the same tick window.
// - At 50ms ping: both inputs arrive ~25ms apart server-side -> both accepted
//   on adjacent ticks, animation cancel works perfectly.
// - At 200ms ping: inputs arrive ~100ms apart server-side, but the client's
//   timing feel is sluggish because the server acknowledgment of the LA
//   arrives 200ms later, making the player "guess" the right moment to cast.
// - Under server stress: variable tick processing time means inputs that
//   should land on the same tick may straddle a tick boundary -> missed weave.
```

Weaving is not an intended mechanic but an emergent consequence of the dual-GCD
system and animation canceling. Its sensitivity to latency and server tick timing
is one of the most discussed aspects of ESO's combat feel.

---

## 4. Cyrodiil PvP: The Networking Bottleneck

Cyrodiil is ESO's large-scale PvP zone where three alliances compete for control
of keeps, resources, and Elder Scrolls across a massive open-world map. It is
the single most demanding networking scenario in the game and the source of ESO's
most persistent performance complaints.

### Scale of the problem

- **Original population cap:** ~200 players per faction, ~600 total per campaign
  (community-estimated; ZeniMax has never confirmed exact numbers)
- **Current population cap:** Significantly reduced over the years --
  community estimates suggest ~60-80 per faction in recent years
- **Engagement density:** During keep sieges, 100+ players can converge on a
  single small area, each casting AoE abilities, siege weapons, and crowd control

### Why TCP fails at scale

The core problem is the interaction between TCP's head-of-line blocking and the
**message flood** created by large-scale AoE combat:

```cpp
// Pseudo-code: TCP head-of-line blocking in Cyrodiil

// The problem: when 200 players are near a keep, each player's TCP stream
// must carry state updates for every nearby player's abilities, movements,
// health changes, and buff/debuff applications.

struct PlayerConnectionState {
    TCPSocket   socket;
    ByteBuffer  send_queue;          // Outbound messages waiting to be sent
    uint32_t    unacked_bytes;       // Bytes sent but not yet acknowledged
    uint32_t    congestion_window;   // TCP congestion control limit
};

// SERVER SIDE: Broadcasting combat events in a large battle
void Server::BroadcastCombatEvent(
    const CombatEvent& event,
    const std::vector<Player*>& nearby_players
) {
    auto serialized = Serialize(event);

    for (Player* player : nearby_players) {
        auto& conn = player->connection;

        // TCP guarantees: all bytes must arrive, in order.
        // If this player's connection has unacknowledged data...
        if (conn.unacked_bytes >= conn.congestion_window) {
            // TCP congestion control kicks in: we CANNOT send more data
            // until the client acknowledges previous packets.
            // All new messages queue up behind the blocked data.
            conn.send_queue.Append(serialized);  // growing backlog
            continue;
        }

        conn.socket.Send(serialized);
        conn.unacked_bytes += serialized.size();
    }
}

// THE CASCADE EFFECT:
//
// 1. 100 players each cast an AoE ability in a keep siege
// 2. Each AoE generates hit results for every target in range (10-30 players)
// 3. Server must send ~100 * 20 = 2000 combat event messages per ability wave
// 4. These messages go to every nearby player's TCP stream
// 5. Player X experiences 2% packet loss -> TCP retransmits, stalling the stream
// 6. While waiting for retransmission ACK (~RTT * 2), ALL new messages queue
// 7. Player X's send queue grows: 50ms of queued data... 200ms... 500ms...
// 8. When retransmission succeeds, a burst of stale data arrives at once
// 9. Client processes this burst -> frame hitch, ability spam, rubber-banding
// 10. Meanwhile, player X's outbound inputs are also delayed -> the server
//     receives X's actions late, potentially affecting other players' experience
```

### The "death spiral" effect

The community has identified a **cascading failure pattern** in large Cyrodiil
battles:

1. Server load increases as players converge on a keep
2. Per-tick processing time increases, server falls behind on ticks
3. TCP send queues grow because the server produces data faster than connections
   can drain
4. Packet loss rates increase due to congestion
5. TCP retransmission timeouts stall individual connections
6. The server must still track pending data for stalled connections, consuming
   memory and CPU
7. A player with poor connectivity can effectively slow down server processing
   for other nearby players because the server must maintain ordered delivery
   guarantees for that connection
8. Reported in-game latency spikes to 999+ ms, abilities fail to fire, the
   "skill lag" phenomenon

Community network analysis has shown that in-game ping during Cyrodiil battles
(120-150+ ms) substantially exceeds the raw network round-trip time to the
server (45-50 ms), indicating significant **server-side processing overhead** on
top of the network transit time.

### Why ZeniMax hasn't switched to UDP

This is speculation, but the most common community theories are:

- The game was architected around TCP from the ground up during development
  (2007-2014), and the networking layer is deeply coupled to TCP semantics
- Switching to UDP would require implementing a custom reliability layer,
  reordering logic, and congestion control -- essentially rewriting the entire
  transport layer
- ESO's game logic may depend on TCP's ordering guarantees in ways that would
  require significant refactoring
- The cost-benefit analysis for a live game with millions of players may not
  justify the risk of such a fundamental change

---

## 5. Update 27 (2020): Major Networking Overhaul

In late 2019, ZeniMax announced a multi-update performance improvement plan
spanning Updates 24 through 27. Update 27 (Stonethorn, August 2020) was the
culmination of this effort and contained the most significant server-side
networking improvements in ESO's history.

### 5.1 Server-side AoE message batching

**The problem:** Persistent AoE abilities (ground-targeted damage/healing fields
that last for several seconds) generated individual hit-result messages for every
target on every tick. A single Dragonknight Standard (a commonly used ultimate
ability) hitting 12 players over its 15-second duration generated hundreds of
individual messages.

**The solution:** AoE abilities were refactored to use **batched message
delivery**. Instead of sending individual hit results per tick, the server
aggregates hits over a short window and sends a single consolidated message.

```cpp
// Pseudo-code: AoE message batching (Update 27 optimization)

// BEFORE Update 27: One message per target per tick
struct AoEHitMessage_Old {
    uint32_t ability_id;
    uint32_t caster_id;
    uint32_t target_id;
    float    damage;
    double   timestamp;
    // Status effects, crit info, etc.
};

// Each tick, for a 12-target AoE ticking every 0.5s:
// Messages per second = 12 targets * 2 ticks/sec = 24 messages/sec PER ABILITY
// In a Cyrodiil siege: 30 AoEs active * 24 messages = 720 messages/sec
// Broadcast to 100 nearby players = 72,000 message deliveries/sec

// AFTER Update 27: Batched delivery
struct AoEBatchMessage_New {
    uint32_t ability_id;
    uint32_t caster_id;
    uint32_t batch_tick_start;
    uint32_t batch_tick_end;

    struct TargetHit {
        uint32_t target_id;
        float    total_damage;
        uint8_t  hit_count;
        uint16_t status_effect_flags;
    };
    std::vector<TargetHit> targets;  // All targets in this batch window
};

void Server::ProcessPersistentAoE(AoEAbility& aoe, double current_time) {
    // Accumulate hits over a batch window (~250ms, community-estimated)
    for (Player* target : GetPlayersInRadius(aoe.position, aoe.radius)) {
        if (current_time >= target->next_aoe_tick_time[aoe.id]) {
            aoe.batch_buffer.AddHit(target->id, CalculateDamage(aoe, target));
            target->next_aoe_tick_time[aoe.id] = current_time + aoe.tick_interval;
        }
    }

    // Flush batch at regular intervals instead of per-tick
    if (current_time >= aoe.next_batch_flush_time) {
        AoEBatchMessage_New batch_msg;
        batch_msg.ability_id = aoe.id;
        batch_msg.caster_id  = aoe.caster_id;
        batch_msg.targets    = aoe.batch_buffer.Flush();

        // Single message replaces dozens of individual messages
        BroadcastToNearby(aoe.position, batch_msg);

        aoe.next_batch_flush_time = current_time + AOE_BATCH_INTERVAL;
    }
}

// RESULT: 30 AoEs * 4 batches/sec * 1 message each = 120 messages/sec
// Broadcast to 100 players = 12,000 deliveries/sec
// ~83% reduction in message volume for AoE-heavy scenarios
```

The initial rollout in Update 27 applied this optimization to **Dragonknight
Standard and its morphs** as a test case, with plans to extend it to all
persistent AoE abilities over subsequent updates.

### 5.2 In-memory database caching layer

**The problem:** ESO's shared database cluster handles character data, inventory,
guild information, achievements, and more. Under high load (particularly during
Cyrodiil battles where many item set procs and equipment checks occur), database
queries became a bottleneck.

**The solution:** Update 27 introduced an **in-memory cache layer** between the
game servers and the production database:

```cpp
// Pseudo-code: In-memory database caching (Update 27)

class DatabaseCacheLayer {
    std::unordered_map<CacheKey, CachedEntry> hot_cache_;
    DatabaseConnection                        backing_store_;
    std::priority_queue<CacheKey, TTLCompare>  eviction_queue_;

public:
    // High-frequency reads (item set checks, character stats) hit cache first
    QueryResult Read(const CacheKey& key) {
        auto it = hot_cache_.find(key);
        if (it != hot_cache_.end() && !it->second.IsExpired()) {
            stats_.cache_hits++;
            return it->second.data;
        }

        // Cache miss -> query backing database
        stats_.cache_misses++;
        QueryResult result = backing_store_.Query(key);

        // Populate cache for subsequent reads
        hot_cache_[key] = CachedEntry{result, GetCurrentTime() + TTL};
        eviction_queue_.push(key);
        return result;
    }

    // Writes go to both cache and database
    void Write(const CacheKey& key, const WriteData& data) {
        backing_store_.Write(key, data);  // Authoritative write
        hot_cache_[key] = CachedEntry{data.AsQueryResult(), GetCurrentTime() + TTL};
    }
};

// Initially deployed for Activity Finder optimization, with plans to expand
// to character loading, inventory queries, and guild operations.
```

### 5.3 Database worker prioritization

**The problem:** Under heavy load, database worker threads could become saturated
by low-priority operations (achievement tracking, logging), starving
high-priority operations (combat calculations, inventory changes).

**The solution:** A priority-based worker allocation system ensuring critical
game functions always have dedicated database workers:

```cpp
// Pseudo-code: Database worker priority queue (Update 27)

enum class QueryPriority : uint8_t {
    CRITICAL  = 0,  // Combat, ability validation, health/resource changes
    HIGH      = 1,  // Inventory, equipment, item set procs
    MEDIUM    = 2,  // Guild operations, mail, Activity Finder
    LOW       = 3,  // Achievements, statistics, logging
};

class DatabaseWorkerPool {
    std::array<WorkerQueue, 4> priority_queues_;  // One per priority level
    std::vector<WorkerThread>  workers_;

    // Guaranteed minimum workers per priority level
    static constexpr int MIN_CRITICAL_WORKERS = 4;
    static constexpr int MIN_HIGH_WORKERS     = 2;

public:
    void SubmitQuery(QueryPriority priority, DatabaseQuery query) {
        priority_queues_[static_cast<int>(priority)].Enqueue(std::move(query));
    }

    // Worker threads pull from highest-priority non-empty queue first
    void WorkerLoop(int worker_id) {
        while (running_) {
            for (int p = 0; p < 4; ++p) {
                if (auto query = priority_queues_[p].TryDequeue()) {
                    Execute(*query);
                    break;  // Re-check from highest priority
                }
            }
        }
    }
};
```

### 5.4 Item set calculation streamlining

**The problem:** ESO has hundreds of item sets, each with unique proc
conditions. Every combat event required checking all equipped set bonuses against
the event's parameters, generating unnecessary server messages even when
conditions were not met.

**The solution:** A comprehensive audit of all item sets to optimize their
conditional logic:

```cpp
// Pseudo-code: Item set proc optimization (Update 27)

// BEFORE: Every combat event checks every equipped set bonus
void Server::OnCombatEvent_Old(const Player& player, const CombatEvent& event) {
    for (const ItemSet& set : player.equipped_sets) {
        // This check was overly broad -- evaluated full proc logic
        // and sent "no proc" messages to the client even on failure
        ProcResult result = set.EvaluateProc(event);
        SendProcResult(player, set, result);  // Sent even on NO_PROC!
    }
}

// AFTER: Early-out filtering prevents unnecessary evaluation and messaging
void Server::OnCombatEvent_New(const Player& player, const CombatEvent& event) {
    for (const ItemSet& set : player.equipped_sets) {
        // Quick bitflag check: does this event type even match
        // the set's trigger category?
        if (!(event.category_flags & set.trigger_mask)) {
            continue;  // Skip entirely -- no evaluation, no message
        }

        // Check internal cooldown before full evaluation
        if (player.set_cooldowns[set.id] > current_time_) {
            continue;  // On cooldown, skip
        }

        ProcResult result = set.EvaluateProc(event);
        if (result.did_proc) {
            ApplySetBonus(player, set, result);
            SendProcResult(player, set, result);  // Only send on actual proc
        }
    }
}
```

### 5.5 Trial instance distribution

**The problem:** ESO's 12-player Trials (raid content) were being assigned to
instance launchers unevenly, causing some physical servers to host multiple
simultaneous Trials and suffer from resource contention.

**The solution:** A revised distribution algorithm that spreads Trial instances
more evenly across the server cluster, reducing the likelihood of "bad instances"
where enemy AI behaves erratically and latency is abnormally high.

### Update 27 improvement timeline context

| Update | Release    | Key Performance Changes                                    |
|--------|------------|------------------------------------------------------------|
| 24     | Q4 2019    | Memory management overhaul, initial combat optimizations   |
| 25     | Q1 2020    | Patching system rewrite (16+ GB client savings)            |
| 26     | Q2 2020    | Non-combat pet system rewrite, character loading threading  |
| 27     | Q3 2020    | AoE batching, database caching, worker prioritization      |

---

## 6. Megaserver Dynamic Instancing

### Zone instance lifecycle

ESO's overworld zones are dynamically instanced. Unlike Cyrodiil (which uses
fixed campaign instances), PvE zones create and destroy shard instances based on
population:

```cpp
// Pseudo-code: Dynamic zone instancing in the megaserver

struct ZoneInstance {
    uint32_t            instance_id;
    uint32_t            zone_id;
    uint32_t            player_count;
    uint32_t            soft_cap;       // Target max players before new instance
    uint32_t            hard_cap;       // Absolute maximum
    PhysicalServer*     host_server;
    InstanceState       state;          // STARTING, RUNNING, DRAINING, SHUTDOWN
};

class ZoneInstanceManager {
    std::unordered_map<uint32_t, std::vector<ZoneInstance*>> zone_instances_;

    // Configured limits (community-estimated)
    static constexpr uint32_t DEFAULT_SOFT_CAP = 150;  // Approximate
    static constexpr uint32_t DEFAULT_HARD_CAP = 200;
    static constexpr uint32_t MAX_INSTANCES_PER_ZONE = 3;

public:
    // Called when a player needs to enter a zone
    ZoneInstance* AssignPlayerToInstance(
        uint32_t zone_id,
        Player* player
    ) {
        auto& instances = zone_instances_[zone_id];

        // STEP 1: Try to place with friends or guild members
        ZoneInstance* social_instance = FindSocialInstance(instances, player);
        if (social_instance && social_instance->player_count < social_instance->soft_cap) {
            return social_instance;
        }

        // STEP 2: Find an instance below soft cap
        for (auto* inst : instances) {
            if (inst->state == InstanceState::RUNNING
                && inst->player_count < inst->soft_cap) {
                return inst;
            }
        }

        // STEP 3: All instances are above soft cap -- create a new one
        //         if we haven't hit the per-zone instance limit
        if (instances.size() < MAX_INSTANCES_PER_ZONE) {
            auto* server = FindLeastLoadedServer();
            auto* new_inst = LaunchZoneInstance(zone_id, server);
            instances.push_back(new_inst);
            return new_inst;
        }

        // STEP 4: All instances exist and are full -- place in least-loaded
        return FindLeastPopulatedInstance(instances);
    }
};
```

### Zone splitting under load

When a zone instance reaches its population threshold, the megaserver can spin up
additional instances. Players already in the zone remain on their current
instance; new arrivals are routed to the new one:

```cpp
// Pseudo-code: Dynamic zone splitting under load

class ZoneLoadMonitor {
    // Runs periodically (e.g., every 30 seconds)
    void EvaluateZoneHealth() {
        for (auto& [zone_id, instances] : zone_instances_) {
            // Check if ANY instance is overloaded
            for (auto* inst : instances) {
                float load_ratio = static_cast<float>(inst->player_count)
                                 / inst->soft_cap;

                if (load_ratio >= SPLIT_THRESHOLD  // e.g., 0.9
                    && instances.size() < MAX_INSTANCES_PER_ZONE
                    && inst->state == InstanceState::RUNNING)
                {
                    // Spin up a new instance on a different physical server
                    auto* server = FindLeastLoadedServer();
                    auto* new_inst = LaunchZoneInstance(zone_id, server);
                    new_inst->state = InstanceState::STARTING;
                    instances.push_back(new_inst);

                    Log("Zone %u instance %u at %.0f%% capacity. "
                        "Launched new instance %u on server %s.",
                        zone_id, inst->instance_id,
                        load_ratio * 100,
                        new_inst->instance_id,
                        server->hostname.c_str());
                }
            }

            // MERGE CHECK: if multiple instances are near-empty,
            // drain one and redirect new players to the other
            MergeUnderPopulatedInstances(instances);
        }
    }

    void MergeUnderPopulatedInstances(std::vector<ZoneInstance*>& instances) {
        if (instances.size() <= 1) return;

        // Find instances below the merge threshold
        std::vector<ZoneInstance*> low_pop;
        for (auto* inst : instances) {
            float load_ratio = static_cast<float>(inst->player_count)
                             / inst->soft_cap;
            if (load_ratio < MERGE_THRESHOLD) {  // e.g., 0.25
                low_pop.push_back(inst);
            }
        }

        // If two or more instances are under-populated, drain the emptiest
        if (low_pop.size() >= 2) {
            std::sort(low_pop.begin(), low_pop.end(),
                [](auto* a, auto* b) {
                    return a->player_count < b->player_count;
                });

            auto* drain_target = low_pop.front();
            drain_target->state = InstanceState::DRAINING;
            // No new players will be assigned here.
            // Existing players remain until they leave naturally
            // or are migrated at the next zone transition.
        }
    }
};
```

### Cyrodiil: a special case

Cyrodiil does **not** use the dynamic instancing system. Instead, it uses
**campaigns** -- fixed, named PvP instances that persist for the campaign
duration (30 days). Each campaign has a per-faction population cap. Players
choose a home campaign and a guest campaign. If a campaign is full, players
enter a queue.

This is intentional: Cyrodiil's PvP design requires persistent world state
(keep ownership, resource control, Elder Scroll positions) that would not
survive dynamic instancing or merging.

---

## 7. Combat Ability Validation Pipeline

ESO's server-authoritative combat system must validate every ability activation
before applying it to the game world. The validation pipeline runs on each
server tick:

```cpp
// Pseudo-code: Server-side combat ability validation pipeline

enum class AbilityResult {
    SUCCESS,
    FAILED_ON_COOLDOWN,
    FAILED_GCD_ACTIVE,
    FAILED_INSUFFICIENT_RESOURCE,
    FAILED_OUT_OF_RANGE,
    FAILED_LINE_OF_SIGHT,
    FAILED_CROWD_CONTROLLED,
    FAILED_SILENCED,
    FAILED_DEAD,
    FAILED_INVALID_TARGET,
    FAILED_WEAPON_MISMATCH,     // Ability requires weapon not currently equipped
    FAILED_BAR_SWAP_IN_PROGRESS // Bar swap desync -- see Known Issues
};

AbilityResult Server::ValidateAbilityActivation(
    Player& player,
    uint32_t ability_id,
    uint32_t target_id,
    double server_time
) {
    // 1. Basic state checks
    if (player.IsDead())
        return AbilityResult::FAILED_DEAD;

    if (player.HasCrowdControl(CC_STUN | CC_KNOCKDOWN | CC_FEAR))
        return AbilityResult::FAILED_CROWD_CONTROLLED;

    if (player.HasCrowdControl(CC_SILENCE) && IsAbilityMagicka(ability_id))
        return AbilityResult::FAILED_SILENCED;

    // 2. Verify the ability is on the player's CURRENT active bar
    //    (This is where bar swap desync manifests)
    if (!player.GetActiveBar().HasAbility(ability_id))
        return AbilityResult::FAILED_WEAPON_MISMATCH;

    // 3. Global Cooldown check
    AbilityType type = GetAbilityType(ability_id);
    if (server_time < player.gcd_state.GetExpiry(type))
        return AbilityResult::FAILED_GCD_ACTIVE;

    // 4. Individual ability cooldown
    if (server_time < player.GetAbilityCooldown(ability_id))
        return AbilityResult::FAILED_ON_COOLDOWN;

    // 5. Resource check (magicka, stamina, or ultimate)
    ResourceType cost_type = GetAbilityCostType(ability_id);
    float cost = CalculateAbilityCost(player, ability_id);
    if (player.GetResource(cost_type) < cost)
        return AbilityResult::FAILED_INSUFFICIENT_RESOURCE;

    // 6. Range and line-of-sight (for targeted abilities)
    if (target_id != SELF_TARGET) {
        Entity* target = world_.FindEntity(target_id);
        if (!target)
            return AbilityResult::FAILED_INVALID_TARGET;

        float distance = Distance(player.position, target->position);
        if (distance > GetAbilityRange(ability_id))
            return AbilityResult::FAILED_OUT_OF_RANGE;

        if (!HasLineOfSight(player.position, target->position))
            return AbilityResult::FAILED_LINE_OF_SIGHT;
    }

    // 7. All checks passed -- apply the ability
    player.ConsumeResource(cost_type, cost);
    player.gcd_state.SetExpiry(type, server_time + GCD_DURATION);

    // Apply effects (damage, healing, buffs, debuffs)
    ApplyAbilityEffects(player, ability_id, target_id, server_time);

    return AbilityResult::SUCCESS;
}

// The pipeline runs for EVERY ability input received EVERY tick.
// In a Cyrodiil siege with 200 players each casting ~1 ability/sec:
// ~200 validations/sec * 6-7 checks each = ~1200-1400 checks/sec
// Add AoE hit resolution (each AoE checks all targets in radius):
// 30 active AoEs * 20 potential targets * 2 ticks/sec = 1200 hit checks/sec
// TOTAL: ~2400+ validation operations per second in a large battle
```

---

## 8. Known Issues

### 8.1 TCP head-of-line blocking in large PvP

The fundamental issue described in Section 4. TCP's guarantee of in-order
delivery means that a single dropped packet stalls all subsequent data on that
connection. In Cyrodiil's high-bandwidth scenario, this creates cascading lag
that can render the game unplayable during large fights. Community members have
documented in-game latency exceeding 999 ms during peak Cyrodiil activity, with
abilities failing to fire for 3+ seconds.

### 8.2 Ability queuing failures under load

When the server is under heavy load, ability inputs can be dropped or arrive
outside their valid timing window. Players report pressing abilities that never
activate -- the input is silently consumed with no effect. This is particularly
punishing for defensive abilities like Break Free (crowd control escape) and
Dodge Roll, where a missed activation can mean death.

### 8.3 "Bar swap" desync

ESO players use two weapon bars and frequently swap between them during combat
rotations. The bar swap bug causes the client and server to disagree about which
bar is currently active. Symptoms include:

- Wrong skills appearing on the ability bar
- Skills failing to cast ("locked out" for ~1.5 seconds)
- Light attacks firing but abilities not registering
- Wrong weapon displaying visually

The bug is triggered by rapid bar swap inputs, especially during animation
canceling, and is exacerbated by latency. It was first reported years ago and a
ZeniMax community manager confirmed it is "not a simple one to fix." The root
cause appears to be a race condition between client-side bar swap state and
server acknowledgment, where rapid inputs can cause the client's bar swap counter
to advance ahead of the server's confirmed state.

### 8.4 Light attack weaving sensitivity to latency

Because weaving relies on precise timing between two independent GCD tracks, its
effectiveness varies significantly with ping. Players with sub-60 ms connections
can reliably weave every rotation, while players with 150+ ms connections
experience inconsistent weave registration. This creates a measurable DPS gap
based on geographic proximity to the megaserver. The community has documented
5-10% DPS differences between low and high latency players executing identical
rotations.

### 8.5 Cyrodiil population cap reductions

ZeniMax has repeatedly lowered Cyrodiil's population caps over the game's
lifetime as a pragmatic (but unpopular) mitigation for server performance. The
original ~200-per-faction cap has been reduced to what the community estimates is
60-80 per faction. These reductions acknowledge the fundamental TCP throughput
limitation without addressing its root cause.

---

## 9. Key Sources

### Official ZeniMax communications

- [ESO's Performance Improvements Plan](https://www.elderscrollsonline.com/en-us/news/post/56681)
  -- Official blog post detailing the Update 24-27 improvement roadmap
- [Game Performance Improvements Preview -- Update 27](https://forums.elderscrollsonline.com/en/discussion/536737/game-performance-improvements-preview-update-27)
  -- Developer forum post with AoE batching and database caching details
- [PC/Mac Patch Notes v6.1.5 -- Stonethorn & Update 27](https://forums.elderscrollsonline.com/en/discussion/542612/pc-mac-patch-notes-v6-1-5-stonethorn-update-27)
  -- Full patch notes for Update 27
- [ESO Port Requirements](https://help.elderscrollsonline.com/app/answers/detail/a_id/1133/~/what-ports-do-i-need-to-open-for-the-elder-scrolls-online)
  -- Official TCP/UDP port ranges

### Developer interviews and articles

- [ESO dev speaks to the power of megaservers in MMO game design](https://www.gamedeveloper.com/production/-i-elder-scrolls-online-i-dev-speaks-to-the-power-of-megaservers-in-mmo-game-design)
  -- Game Developer article with Matt Firor on megaserver architecture
- [Megaservers -- UESP Wiki](https://en.uesp.net/wiki/Online:Megaservers)
  -- Community wiki documentation of the megaserver system

### Community technical analysis

- [Server Lag Explained -- Forum Analysis](https://forums.elderscrollsonline.com/en/discussion/495264/server-lag-explained-server-calculations-skill-actions-are-not-the-main-reason-for-lag)
  -- Community engineer analysis of 60-64 Hz tick rate, bandwidth scaling, and TCP issues
- [Lag is a bandwidth issue, not a server issue](https://forums.elderscrollsonline.com/en/discussion/594604/lag-is-a-bandwidth-issue-not-a-server-issue)
  -- Technical discussion of bandwidth vs. server CPU as bottleneck, with ping measurements
- [To reduce the lag, reduce the data transferred](https://forums.elderscrollsonline.com/en/discussion/450244/to-reduce-the-lag-reduce-the-data-transferred)
  -- Community proposals for reducing network message volume
- [ESO Performance and Lag -- Technical Discussion](https://forums.elderscrollsonline.com/en/discussion/256376/eso-performance-and-lag-technical-discussion)
  -- Long-running community thread on TCP networking issues

### Bar swap and ability bugs

- [Video: Bar swap glitch, and how to 100% reproduce it](https://forums.elderscrollsonline.com/en/discussion/451593/video-bar-swap-glitch-and-how-to-100-reproduce-it)
  -- Detailed reproduction steps and developer acknowledgment
- [Bar Swap Bug -- Forum Thread](https://forums.elderscrollsonline.com/en/discussion/395061/bar-swap-bug)
  -- Community documentation of the desync issue

### Hardware and infrastructure

- [ESO NA Datacenters Getting Upgrade](https://mmos.com/news/elder-scrolls-online-na-datacenters-are-getting-a-much-needed-upgrade-today)
  -- 2022 hardware refresh replacing 2012-era servers
- [PS EU Server Upgrade Confirmation](https://forums.elderscrollsonline.com/en/discussion/631839/we-finally-recieved-the-new-serverupgrade-on-ps-eu-big-improvement)
  -- Community confirmation of console hardware refresh

### Combat mechanics

- [Animation Canceling and Global Cooldowns (Steam Guide)](https://steamcommunity.com/sharedfiles/filedetails/?id=2126976552)
  -- Detailed breakdown of GCD tracks and weaving mechanics
- [Animation Canceling -- UESP Wiki](https://en.uesp.net/wiki/Online:Animation_Canceling)
  -- Community documentation of animation cancel mechanics
- [Cyrodiil Performance Improvements](https://massivelyop.com/2020/07/28/elder-scrolls-online-is-promising-live-tests-and-major-cyrodiil-performance-improvements/)
  -- Coverage of ZeniMax's Cyrodiil improvement plans
