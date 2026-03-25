# World of Warcraft: Netcode Deep-Dive

## Overview

World of Warcraft (2004--present) is the most commercially successful MMO ever shipped, peaking at 12 million subscribers and still actively developed two decades later. Its networking model is a pragmatic hybrid that prioritizes responsiveness at massive scale over cheat-proof authority:

- **Movement is client-authoritative.** The client runs the full physics simulation and tells the server where the player is. The server performs basic sanity checks (speed, teleport) but otherwise trusts the client.
- **Combat is fully server-authoritative.** All damage, healing, hit/miss/dodge/parry rolls, and spell effects are computed on the server. The client never determines combat outcomes.
- **Remote players are rendered via dead reckoning** (extrapolation from last known position, direction, and speed). NPCs follow server-sent splines.
- **TCP is the transport layer**, which is unusual for a real-time game but appropriate given WoW's tolerance for occasional latency spikes and the reliability guarantees TCP provides for combat state.

This hybrid split -- trust the client for movement, trust the server for combat -- is WoW's defining networking decision. It makes the game feel responsive even at 200ms+ latency, but it opens the door to movement cheats (speed hacks, teleport hacks, fly hacks) that have plagued the game since launch.

---

## Architecture

### Zone-Partitioned Server Cluster

Each WoW realm is not a single server but a cluster of cooperating processes:

| Component | Role |
|-----------|------|
| **Realm Server** | Authentication gateway; routes players to the correct world server |
| **World Server(s)** | Authoritative simulation for a set of zones (maps); runs NPC AI, combat, loot, quests |
| **Instance Server(s)** | Spun up on demand for dungeons/raids/battlegrounds; recycled when empty |
| **Database Server** | Persistent character and world state (MySQL in open-source emulators; proprietary at Blizzard) |
| **Chat / Social** | Separate service handling guild, friends, whispers across zones |

The world is partitioned by **MapID** -- Kalimdor and Eastern Kingdoms are each a single contiguous map, while each dungeon/raid is its own map. A player transferring between maps (e.g., entering a dungeon) is handed off between world/instance servers. Modern WoW adds **phasing**, **sharding**, and **cross-realm zones** as additional virtualization layers on top of this.

### JAM: Serialization and Routing Layer

WoW's inter-server communication is built on **JAM** ("Joe's Automated Messages"), described by Joe Rumsey (Blizzard) in his [GDC 2013 talk](https://www.gdcvault.com/play/1017733/Network-Serialization-and-Routing-in).

Key properties of JAM:

- **Code generation from `.jam` definition files.** Developers describe message schemas in a custom IDL; tools (using Flex and Bison) generate C++ serializer/deserializer classes automatically.
- **Dynamic protocol negotiation.** When two services connect, they exchange CRC values for the protocols they support. This allows rolling upgrades -- old and new server versions can coexist during deployment.
- **Entity-addressed routing.** You send a message *to an entity* (e.g., a player GUID), and JAM automatically routes it to whichever server currently owns that entity. This is critical for cross-zone operations like mail, auction house, and guild chat.
- **Binary serialization.** When the wire format matches the struct layout, JAM uses direct memory copies for maximum throughput. Otherwise it falls back to field-by-field encoding.

At WoW's 2004 launch, JAM was only used for server-to-server communication. Client-to-server used a hand-written binary protocol (the opcodes documented on WoWDev wiki). In later Blizzard projects, JAM was extended to client-server communication as well.

### TCP Transport

WoW uses **TCP port 3724** for all game traffic. This is unusual -- most real-time games use UDP to avoid head-of-line blocking. WoW gets away with TCP because:

1. **Combat is turn-based at the network level.** Tab-target, cast-time abilities, and the GCD mean the game never needs sub-frame input precision.
2. **Movement is client-authoritative.** The client does not wait for server acknowledgment before moving, so TCP retransmission delays only affect *other players seeing you*, not your own responsiveness.
3. **Reliability matters more than latency.** A dropped spell cast or loot roll is far worse than a 50ms delay. TCP guarantees delivery.
4. **Firewall/NAT friendliness.** TCP works through virtually all consumer networking equipment without configuration.

The trade-off is that during packet loss, TCP's retransmission and congestion control can cause multi-second freezes. WoW players in lossy network environments (Wi-Fi, mobile tethering) experience this as "rubber-banding" or disconnects.

---

## Movement: Client-Authoritative

### The Unusual Decision

In most competitive multiplayer games, the server is the authority on player position. WoW does the opposite: **the client runs the full movement physics simulation and tells the server where it is.** The server does not simulate player movement at all -- it receives position updates and redistributes them to nearby players.

This was a deliberate decision driven by scale. With thousands of players per realm and hundreds per zone, running server-side physics for every player would be prohibitively expensive, especially in 2004. Client-authoritative movement also provides perfect responsiveness: your character reacts to your input with zero network latency.

### How It Works

```cpp
// === CLIENT-SIDE: Movement Authority Loop ===
// The client runs the full physics simulation locally.
// The server does NOT simulate player movement.

void ClientMovementUpdate(Player* player, float dt) {
    // 1. Read local input (WASD, mouse, jump, etc.)
    MovementInput input = ReadInput();

    // 2. Run full physics simulation locally
    //    - Gravity, collision, swimming, flying, terrain following
    //    - The client is the AUTHORITY -- this is the real position
    Vector3 old_position = player->position;
    uint32 old_flags = player->movement_flags;

    ApplyMovementPhysics(player, input, dt);  // gravity, collision, etc.
    player->position += player->velocity * dt;
    player->orientation = input.facing;

    // 3. Determine if we need to send a packet to the server
    //    Movement packets are EVENT-DRIVEN, not polled:
    //    - Send on input state CHANGE (start/stop moving, turn, jump, land)
    //    - Send heartbeat every ~500ms during continuous movement
    bool state_changed = (player->movement_flags != old_flags);
    bool heartbeat_due = (time_since_last_packet > HEARTBEAT_INTERVAL);  // ~500ms

    if (state_changed) {
        // Specific opcode for the state change
        SendMovementPacket(GetOpcodeForStateChange(old_flags, player->movement_flags),
                           player->BuildMovementInfo());
    } else if (heartbeat_due && player->IsMoving()) {
        // Periodic position sync during straight-line movement
        SendMovementPacket(MSG_MOVE_HEARTBEAT, player->BuildMovementInfo());
    }
    // If standing still: no packets sent at all (zero bandwidth)

    // 4. Render immediately -- no waiting for server confirmation
    RenderPlayerAt(player->position, player->orientation);
}
```

### Movement Packet Structure

Movement packets are **event-driven**, not polled. The client sends a packet when the player's movement *state* changes, and a heartbeat during continuous movement. Based on reverse-engineering documented on the [WoWDev wiki](https://wowdev.wiki/Opcodes):

```cpp
// === Movement Opcodes (event-driven) ===
// Each state change has its own opcode:
//   MSG_MOVE_START_FORWARD     // player presses W
//   MSG_MOVE_STOP              // player releases W
//   MSG_MOVE_START_STRAFE_LEFT // player presses A (strafe mode)
//   MSG_MOVE_STOP_STRAFE       // player releases A
//   MSG_MOVE_JUMP              // player presses space
//   MSG_MOVE_FALL_LAND         // player hits the ground
//   MSG_MOVE_START_TURN_LEFT   // player begins turning
//   MSG_MOVE_STOP_TURN         // player stops turning
//   MSG_MOVE_START_SWIM        // player enters water
//   MSG_MOVE_STOP_SWIM         // player exits water
//   MSG_MOVE_HEARTBEAT         // periodic sync (~500ms)
//   MSG_MOVE_SET_FACING        // orientation-only change

// === MovementInfo Structure (payload of every movement packet) ===
struct MovementInfo {
    uint32  movement_flags;  // bitfield: forward, backward, strafe, falling,
                             //   swimming, flying, on_transport, etc.
    uint16  extra_flags;     // pitch/spline flags (later expansions)
    uint32  timestamp;       // client-side millisecond timestamp
    float   x, y, z;        // world position
    float   orientation;     // facing direction (radians)

    // Conditional fields based on movement_flags:
    // --- If ON_TRANSPORT ---
    uint64  transport_guid;  // GUID of ship/elevator/vehicle
    float   transport_x, transport_y, transport_z;  // offset on transport
    float   transport_orientation;
    uint32  transport_time;

    // --- If SWIMMING or FLYING ---
    float   pitch;           // vertical look angle

    // --- If FALLING ---
    uint32  fall_time;       // milliseconds spent in air
    float   jump_velocity;   // initial vertical velocity
    float   jump_sin_angle;  // horizontal jump direction (sin)
    float   jump_cos_angle;  // horizontal jump direction (cos)
    float   jump_xy_speed;   // horizontal speed at jump start

    // --- If SPLINE_ELEVATION ---
    float   spline_elevation;
};
```

### Server-Side Validation

The server does not simulate movement, but it performs **basic sanity checks**:

```cpp
// === SERVER-SIDE: Movement Validation (simplified) ===
void ServerHandleMovement(Player* player, MovementInfo& info) {
    float distance = Distance(player->last_known_position, info.position);
    float elapsed = (info.timestamp - player->last_move_time) / 1000.0f;
    float max_speed = player->GetMaxMoveSpeed();  // includes buffs, mounts

    // Speed check: did the player move faster than physically possible?
    float max_distance = max_speed * elapsed * SPEED_TOLERANCE;  // ~1.1x tolerance
    if (distance > max_distance && distance > TELEPORT_THRESHOLD) {
        // Flag for anti-cheat review; may disconnect or rubber-band
        LogCheatDetection(player, "SPEED_HACK", distance, max_distance);
        // Optionally: force player back to last valid position
        // player->TeleportTo(player->last_known_position);
        return;
    }

    // Z-axis check: is the player underground or in the air without flying?
    float ground_z = GetTerrainHeight(info.x, info.y);
    if (info.z < ground_z - UNDERGROUND_TOLERANCE && !player->CanFly()) {
        LogCheatDetection(player, "TELEPORT_PLANE", info.z, ground_z);
        return;
    }

    // Accept the position -- the client is authoritative
    player->last_known_position = info.position;
    player->last_move_time = info.timestamp;
    player->movement_flags = info.movement_flags;
    player->orientation = info.orientation;

    // Broadcast to nearby players (they will dead-reckon from this)
    BroadcastToNearbyPlayers(player, BuildMovementBroadcast(info));
}
```

### Trade-Offs

| Benefit | Cost |
|---------|------|
| Zero-latency movement response | Speed hacks, teleport hacks, fly hacks are possible |
| No server CPU spent on player physics | Server cannot detect subtle movement exploits |
| Scales to thousands of players per zone | Wall-climbing and terrain exploits are client-side |
| Simple server implementation | Requires a separate anti-cheat system (Warden) |

---

## Remote Players: Dead Reckoning

### Extrapolation Model

When the server relays Player A's movement packet to Player B's client, Player B does not see Player A's *current* position -- they see the *last reported* position plus extrapolation. Since movement packets are event-driven (sent on state changes) with ~500ms heartbeats during straight-line movement, remote players can be up to half a second out of date, plus network latency.

```cpp
// === CLIENT-SIDE: Dead Reckoning for Remote Players ===
// When we receive a movement packet for another player, we store it and
// extrapolate their position forward until the next packet arrives.

struct RemotePlayerState {
    Vector3     last_known_position;
    float       last_known_orientation;
    Vector3     velocity;           // derived from movement_flags + speed
    uint32      last_update_time;   // server timestamp from the packet
    uint32      movement_flags;     // forward, backward, strafing, etc.
    float       move_speed;         // current speed (walk/run/mount)
};

Vector3 DeadReckonRemotePlayer(RemotePlayerState& state, uint32 current_time) {
    float elapsed = (current_time - state.last_update_time) / 1000.0f;

    if (!IsMoving(state.movement_flags)) {
        // Player is stationary -- no extrapolation needed
        return state.last_known_position;
    }

    // Reconstruct velocity from orientation and speed
    // (the client sends orientation + movement_flags, not a velocity vector)
    Vector3 direction;
    if (state.movement_flags & MOVEFLAG_FORWARD) {
        direction.x = cos(state.last_known_orientation);
        direction.y = sin(state.last_known_orientation);
    }
    if (state.movement_flags & MOVEFLAG_STRAFE_LEFT) {
        direction.x += cos(state.last_known_orientation + PI/2);
        direction.y += sin(state.last_known_orientation + PI/2);
    }
    // ... similar for backward, strafe_right

    direction = Normalize(direction);
    Vector3 extrapolated = state.last_known_position
                         + direction * state.move_speed * elapsed;

    // Clamp to terrain to prevent floating/underground
    extrapolated.z = GetTerrainHeight(extrapolated.x, extrapolated.y);

    return extrapolated;
}

// When a new movement packet arrives, snap or blend to the corrected position
void OnRemoteMovementPacket(RemotePlayer* remote, MovementInfo& info) {
    Vector3 predicted = DeadReckonRemotePlayer(remote->state, GetCurrentTime());
    Vector3 actual = Vector3(info.x, info.y, info.z);
    float error = Distance(predicted, actual);

    if (error < SNAP_THRESHOLD) {
        // Small error: blend smoothly to correct position
        remote->visual_position = Lerp(predicted, actual, BLEND_FACTOR);
    } else {
        // Large error (direction change, lag spike): snap immediately
        remote->visual_position = actual;
    }

    // Update state for future extrapolation
    remote->state.last_known_position = actual;
    remote->state.last_known_orientation = info.orientation;
    remote->state.movement_flags = info.movement_flags;
    remote->state.last_update_time = info.timestamp;
    remote->state.move_speed = GetSpeedForFlags(info.movement_flags);
}
```

### NPC Movement: Spline Interpolation

NPCs do not use dead reckoning. Instead, the server sends **SMSG_ON_MONSTER_MOVE** packets containing a complete spline (series of waypoints + timing). The client interpolates the NPC along this spline deterministically:

```cpp
// === SERVER -> CLIENT: NPC Spline Movement ===
// The server sends a full movement spline for the NPC.
// The client follows it locally without further packets.

struct MonsterMovePacket {            // SMSG_ON_MONSTER_MOVE
    uint64  guid;                     // NPC's GUID
    Vector3 start_position;           // current position
    uint32  spline_id;               // unique ID for this movement
    uint8   move_type;                // NORMAL, STOP, FACING_TARGET, etc.
    uint32  move_time;                // total duration in milliseconds
    uint32  num_waypoints;
    Vector3 waypoints[];              // series of positions
};

// Client-side: interpolate NPC along the spline
Vector3 InterpolateNPCSpline(NPCSpline& spline, uint32 current_time) {
    float t = (current_time - spline.start_time) / (float)spline.move_time;
    t = Clamp(t, 0.0f, 1.0f);

    // Find which segment we're on
    int segment = (int)(t * (spline.num_waypoints - 1));
    float local_t = (t * (spline.num_waypoints - 1)) - segment;

    // Catmull-Rom interpolation between waypoints for smooth curves
    Vector3 p0 = spline.waypoints[max(0, segment - 1)];
    Vector3 p1 = spline.waypoints[segment];
    Vector3 p2 = spline.waypoints[min(segment + 1, spline.num_waypoints - 1)];
    Vector3 p3 = spline.waypoints[min(segment + 2, spline.num_waypoints - 1)];

    return CatmullRomInterpolate(p0, p1, p2, p3, local_t);
}
```

This means NPC movement is perfectly smooth and bandwidth-efficient -- the server only sends a packet when the NPC starts, stops, or changes its path.

---

## Combat System: Server-Authoritative

### Target-Locked Design

WoW's combat is fundamentally different from FPS combat. Players do not aim freely; instead they:

1. Select a target (click or Tab)
2. Cast an ability: `CastSpell(target_guid, spell_id)`
3. The server resolves everything: range check, line-of-sight, hit/miss/dodge/parry/resist, damage/healing calculation

This means **traditional lag compensation (server-side hitbox rewinding) does not apply to WoW.** There are no projectile trajectories to rewind, no hitboxes to reconstruct. The server simply checks: "Is the target within range of the caster at the time the server processes the cast?"

### Server-Side Combat Resolution

```cpp
// === SERVER-SIDE: Spell Cast Resolution ===
// The server is the SOLE AUTHORITY on all combat outcomes.

void ServerProcessSpellCast(Player* caster, uint64 target_guid, uint32 spell_id) {
    Unit* target = GetUnit(target_guid);
    SpellInfo* spell = GetSpellInfo(spell_id);

    // 1. Validate the cast
    if (!target || !target->IsAlive())               return SendError(SPELL_FAILED_BAD_TARGETS);
    if (Distance(caster, target) > spell->range)     return SendError(SPELL_FAILED_OUT_OF_RANGE);
    if (!HasLineOfSight(caster, target))             return SendError(SPELL_FAILED_LINE_OF_SIGHT);
    if (caster->HasCooldown(spell_id))               return SendError(SPELL_FAILED_NOT_READY);
    if (caster->GetPower(spell->power_type) < spell->cost)
                                                     return SendError(SPELL_FAILED_NO_POWER);

    // 2. Begin cast (if cast time > 0)
    if (spell->cast_time > 0) {
        caster->StartCasting(spell, target);
        BroadcastCastBar(caster, spell);  // other players see the cast bar
        return;  // completion handled by timer
    }

    // 3. Resolve instant cast (or cast completion callback)
    ResolveSpellHit(caster, target, spell);
}

void ResolveSpellHit(Unit* caster, Unit* target, SpellInfo* spell) {
    // Single-roll attack table (all outcomes are mutually exclusive)
    // Roll order: Miss -> Dodge -> Parry -> Block -> Crit -> Hit
    float roll = RandomFloat(0.0f, 100.0f);
    float miss_chance    = CalculateMissChance(caster, target, spell);
    float dodge_chance   = CalculateDodgeChance(target, caster);
    float parry_chance   = CalculateParryChance(target, caster);

    if (roll < miss_chance) {
        SendCombatResult(caster, target, RESULT_MISS);
        return;
    }
    roll -= miss_chance;

    if (spell->IsPhysical() && roll < dodge_chance) {
        SendCombatResult(caster, target, RESULT_DODGE);
        return;
    }
    roll -= dodge_chance;

    if (spell->IsPhysical() && target->IsFacing(caster) && roll < parry_chance) {
        SendCombatResult(caster, target, RESULT_PARRY);
        return;
    }

    // Calculate damage/healing
    int32 amount = CalculateSpellEffect(caster, target, spell);
    ApplyDamageOrHealing(target, amount, spell);
    SendCombatResult(caster, target, RESULT_HIT, amount);
}
```

Because combat is entirely server-side, **the client cannot determine combat outcomes.** Damage numbers, health changes, buff/debuff applications, and death are all communicated from server to client. The client's role in combat is purely input (ability selection) and display (health bars, combat log, animations).

---

## Spell Batching

### The Defining Netcode Mechanic

Spell batching is the single most discussed WoW netcode mechanic. From 2004 to 2014, the server processed player actions in **400ms batch windows** rather than immediately. This created a distinctive feel where two players could seemingly hit each other "at the same time" -- both Mages could Polymorph each other, both Rogues could Gouge each other, a Mage could Ice Block a Pyroblast that was already in flight.

### How It Works

```cpp
// === SERVER-SIDE: Spell Batching Algorithm ===
// Actions are not processed immediately. Instead, they accumulate in a queue
// and are resolved together at fixed intervals.

class SpellBatchProcessor {
    struct PendingAction {
        Unit*       caster;
        Unit*       target;
        uint32      spell_id;
        uint32      arrival_time;  // when the server received this
    };

    std::vector<PendingAction> pending_actions;
    uint32 batch_interval;   // 400ms (2004-2014), ~10-20ms (2014+)
    uint32 last_batch_time;

    // Called when a player casts a spell
    void QueueAction(Unit* caster, Unit* target, uint32 spell_id) {
        pending_actions.push_back({caster, target, spell_id, GetServerTime()});
    }

    // Called every server tick
    void ProcessBatches(uint32 current_time) {
        if (current_time - last_batch_time < batch_interval)
            return;  // not time for a batch yet

        last_batch_time = current_time;

        // ALL actions in the queue are resolved "simultaneously"
        // This means both sides of a simultaneous interaction can succeed
        for (auto& action : pending_actions) {
            // Resolve using the world state as it was at batch START
            // (not after previous actions in this batch have applied)
            ResolveSpellHit(action.caster, action.target,
                            GetSpellInfo(action.spell_id));
        }

        // Now apply all results at once
        ApplyAllPendingResults();
        pending_actions.clear();
    }
};

// Example: Simultaneous Polymorph scenario (400ms batch window)
//
// Time 0ms:    Mage A casts Polymorph on Mage B (queued)
// Time 150ms:  Mage B casts Polymorph on Mage A (queued)
// Time 400ms:  BATCH PROCESSES -- both Polymorphs resolve against
//              pre-batch state. Both succeed! Both mages are Polymorphed.
//
// With 10ms batching (modern WoW):
// Time 0ms:    Mage A casts Polymorph on Mage B (queued)
// Time 10ms:   BATCH PROCESSES -- Mage B is Polymorphed
// Time 150ms:  Mage B tries to cast Polymorph, but is already Polymorphed -> fails
```

### Timeline of Batching Changes

| Period | Batch Window | Context |
|--------|-------------|---------|
| **2004--2014** (Vanilla through MoP) | ~400ms | Original design. The server processed unit-to-unit interactions in 400ms batches. Confirmed by Celestalon (Blizzard, June 2014). |
| **2014** (Warlords of Draenor 6.0) | ~10--20ms | Celestalon: "We no longer batch them up like that. We just do it as fast as we can, which usually amounts to between 1ms and 10ms later." |
| **2019** (WoW Classic 1.13) | 400ms (artificial) | Blizzard deliberately re-implemented 400ms batching for the "authentic" Classic experience. |
| **2021** (Classic patch 1.13.7) | ~10ms | Community backlash led Blizzard to reduce Classic's batching to match modern retail. Went live April 20, 2021. |

### Celestalon's Explanation (June 2014)

The key blue post: any action that one unit takes on another different unit used to be processed in batches every 400ms. Self-targeted actions (like healing yourself) were processed immediately. The asymmetry -- self-heals instant, cross-target heals batched -- created quirks that became embedded in high-level PvP strategy.

---

## Melee Leeway

### WoW's Lag Compensation for Melee

Since WoW has no server-side hitbox rewinding (the technique FPS games use for lag compensation), it instead uses a simpler system for melee combat: **melee leeway**. When both the attacker and the target are moving, the effective melee range is extended by approximately 2.66 yards. This compensates for the fact that both characters' positions are slightly out of date due to network latency and dead reckoning.

### The Formula

```cpp
// === SERVER-SIDE: Melee Leeway Range Calculation ===
// Documented formula (confirmed by Blizzard as "not a bug" for Classic):
//
// Melee Range = MAX(5, SourceCombatReach + TargetCombatReach + 4/3)
//             + BothMovingBonus
//
// BothMovingBonus = 8/3 yards (~2.66) if:
//   - Source is moving AND
//   - Target is moving AND
//   - Neither is snared below 70% movement speed

float CalculateMeleeRange(Unit* attacker, Unit* target) {
    // Base melee range: combat reach of both units + constant
    // Most humanoid players have CombatReach = 1.5 yards
    float base_range = attacker->GetCombatReach()   // typically 1.5
                     + target->GetCombatReach()      // typically 1.5
                     + (4.0f / 3.0f);                // +1.33 yards

    // Floor of 5 yards (ensures minimum melee range)
    base_range = std::max(5.0f, base_range);

    // Leeway bonus when both combatants are moving
    if (IsMoving(attacker) && IsMoving(target)) {
        // Check neither is significantly snared (threshold: 70% move speed)
        bool attacker_snared = attacker->GetSpeedRate(MOVE_RUN) < 0.70f;
        bool target_snared   = target->GetSpeedRate(MOVE_RUN) < 0.70f;

        if (!attacker_snared && !target_snared) {
            base_range += (8.0f / 3.0f);  // +2.66 yards leeway
        }
    }

    return base_range;
}

// Practical effect:
// - Both standing still: MAX(5, 1.5 + 1.5 + 1.33) = 5.0 yards (floor applies)
// - Both moving:         5.0 + 2.66 = ~7.66 yards
//
// This is why melee classes can seemingly hit from very far away when
// both players are running. It's not a bug -- it's intentional lag
// compensation for the client-authoritative movement model.
```

### Ranged Leeway

Leeway also applies to ranged attacks and AoE, but with different formulas:

```
Ranged: SourceCombatReach + TargetCombatReach + SpellRange + BothMovingBonus(2.0)
AoE:    TargetCombatReach + SpellRadius + SourceMovingBonus(2.0)
```

### Impact on Gameplay

Melee leeway was one of the most debated mechanics in WoW Classic. Players with low latency experienced melee range as noticeably larger than expected, because the leeway bonus was designed for the higher-latency connections common in 2004--2006. Blizzard confirmed it was working as originally intended and declined to change it, stating: "After careful study and testing, we found that for players with low latency, the state of melee leeway is how they would have experienced the game in 2006."

---

## Spell Queue Window

### The Primary Combat Latency Compensation

The **Spell Queue Window** (CVar: `SpellQueueWindow`) is WoW's most important combat latency compensation mechanism. It allows players to press the next ability **before the current one finishes**, and the client will queue it for execution the instant the GCD or cast time expires.

```cpp
// === CLIENT-SIDE: Spell Queue Window ===
// Default: 400ms. Configurable from 0-400ms via /console SpellQueueWindow <ms>

class SpellQueueSystem {
    uint32 queue_window_ms = 400;    // CVar SpellQueueWindow
    uint32 queued_spell_id = 0;
    uint64 queued_target_guid = 0;
    uint32 queue_time = 0;

    // Called when the player presses an ability key
    void OnAbilityPress(uint32 spell_id, uint64 target_guid) {
        float gcd_remaining = GetGCDRemaining();
        float cast_remaining = GetCurrentCastRemaining();
        float time_remaining = std::max(gcd_remaining, cast_remaining);

        if (time_remaining <= 0) {
            // GCD/cast is already finished -- send immediately
            SendCastRequest(spell_id, target_guid);
        }
        else if (time_remaining <= queue_window_ms / 1000.0f) {
            // Within the queue window -- queue for execution at GCD/cast end
            queued_spell_id = spell_id;
            queued_target_guid = target_guid;
            queue_time = GetClientTime();
        }
        else {
            // Too early to queue -- reject or replace existing queue
            // (some players prefer low SpellQueueWindow to avoid "locking in"
            //  an ability they don't want)
        }
    }

    // Called every frame
    void Update() {
        if (queued_spell_id != 0 && GetGCDRemaining() <= 0 && GetCurrentCastRemaining() <= 0) {
            // GCD/cast just expired -- fire the queued spell immediately
            SendCastRequest(queued_spell_id, queued_target_guid);
            queued_spell_id = 0;
        }
    }
};

// Without the spell queue window, ability chaining would require:
//   1. Wait for GCD to expire (1500ms)
//   2. Player reaction time to notice GCD is done (~100-200ms)
//   3. Network round trip to send the cast (~50-200ms)
//   Total gap between abilities: 150-400ms of dead time
//
// With the queue window:
//   1. Player presses next ability during the last 400ms of GCD
//   2. Client sends cast request the INSTANT the GCD expires
//   3. Only network one-way latency before server processes it
//   Total gap: effectively zero from the player's perspective
```

### Tuning Considerations

| SpellQueueWindow Value | Effect |
|------------------------|--------|
| **400ms** (default) | Most forgiving. Abilities chain smoothly even with poor timing. Risk: may "lock in" an ability you want to cancel. |
| **100-200ms** | Common for competitive PvP players. Less chance of accidentally queuing the wrong ability. |
| **0ms** | No queuing. Must press ability *after* GCD ends. Results in significant DPS/HPS loss from gaps between casts. |

The spell queue window interacts with the server's `SpellQueueWindow` CVar (also known as "Custom Lag Tolerance" in the UI). Competitive players often tune this value based on their latency. Some addons like [LatencyGuard](https://github.com/Kkthnx-Wow/LatencyGuard) dynamically adjust it based on current network conditions.

---

## Evolution Over 20 Years

### Timeline

| Year | Expansion | Effective Tick Rate | Batch Window | Key Networking Changes |
|------|-----------|-------------------|--------------|----------------------|
| **2004** | Vanilla | ~2.5 Hz (estimated from traffic analysis) | ~400ms | TCP-only; client-auth movement; JAM for inter-server; hand-written client protocol |
| **2007** | The Burning Crusade | ~5-10 Hz (estimated) | ~400ms | Arena system (small-scale PvP demands tighter networking) |
| **2008** | Wrath of the Lich King | ~10 Hz (estimated) | ~400ms | 25-man raids push server tick improvements; Wintergrasp world PvP zone |
| **2010** | Cataclysm | ~10-15 Hz | ~400ms | Rated Battlegrounds; incremental server infrastructure upgrades |
| **2012** | Mists of Pandaria | ~15-20 Hz | ~400ms | Large-scale world PvP (Timeless Isle); spell batching quirks well-known in PvP community |
| **2014** | Warlords of Draenor | ~20-30 Hz | **~10-20ms** | **Spell batching effectively eliminated.** Celestalon confirms the change. "The game feels noticeably more responsive." |
| **2016** | Legion | ~20-30 Hz | ~10-20ms | Mythic+ dungeons demand consistent server performance; Demon Hunter double-jump tests movement system |
| **2018** | Battle for Azeroth | ~30 Hz | ~10-20ms | War Mode (world PvP sharding); server infrastructure improvements |
| **2019** | WoW Classic | N/A | **400ms (artificial)** | Blizzard deliberately re-implements 400ms batching on the modern server for "authenticity" |
| **2020** | Shadowlands | ~30-50 Hz | ~10-20ms | Continued infrastructure modernization |
| **2021** | Classic patch 1.13.7 | N/A | **~10ms** | Community rejects artificial batching; Blizzard reduces to modern values. Goes live April 20. |
| **2022** | Dragonflight | ~30-50 Hz | ~10-20ms | Dragonriding system pushes movement system to new speeds (>900% mount speed) |
| **2024** | The War Within | ~30-50 Hz | ~10-20ms | 20th anniversary; continued refinements |

### Key Observations

- **The network tick rate is "over 10 times what it was back in 2004"** (Blizzard, 2019 forum post). If vanilla was ~2.5 Hz, modern WoW is ~25-50 Hz.
- **WoW is not bandwidth-hungry.** Academic traffic analysis (TU Wien, 2007) measured peak bandwidth at ~80 kbps upstream and ~420 kbps downstream. The event-driven movement model and TCP's Nagle algorithm keep packet rates low.
- **The biggest responsiveness improvement was the WoD batch window reduction** (2014). Reducing 400ms to ~10ms had a more noticeable impact than any tick rate change.

---

## Anti-Cheat Challenges

### The Fundamental Problem

Client-authoritative movement means **the client can lie about its position**. This creates cheat vectors that are fundamentally impossible with server-authoritative movement:

| Cheat Type | How It Works | Detection Difficulty |
|------------|-------------|---------------------|
| **Speed hack** | Client sends positions that imply faster-than-possible movement | Medium -- server checks distance/time |
| **Teleport hack** | Client sends a position far from the previous one | Easy to detect, harder to distinguish from lag |
| **Fly hack** | Client sends airborne positions without a flying mount | Medium -- requires Z-axis validation against terrain |
| **No-clip / Wall hack** | Client ignores collision and moves through terrain | Hard -- server does not have full collision mesh |
| **No-fall damage** | Client suppresses FALL_LAND packets or manipulates fall time | Medium |

### Warden

**Warden** is Blizzard's anti-cheat system, integrated into all Blizzard games since the early 2000s. It operates entirely in **user mode** (no kernel driver) and functions as a memory scanner:

1. **Module scanning.** Warden enumerates loaded DLLs in the WoW process, looking for known cheat tool signatures.
2. **Memory hashing.** The server sends a list of memory addresses to scan. The client reads those addresses, computes hashes, and sends them back. The server compares against known signatures of popular cheat programs.
3. **Code integrity.** Warden can verify that WoW's own code sections have not been modified in memory (e.g., NOP-ing out anti-cheat checks).
4. **Periodic scanning.** Scans run several times per minute at irregular intervals to make timing-based evasion difficult.

```
Warden Operating Cycle:
  Server -> Client:  "Scan these memory addresses: [addr1, addr2, ...]"
  Client:            Reads bytes at each address, computes SHA1 hashes
  Client -> Server:  "Here are the hashes: [hash1, hash2, ...]"
  Server:            Compares against blacklist of known cheat signatures
                     If match found -> flag account for review/ban
```

### Limitations

Warden is a **reactive** system. It detects known cheats by signature, but novel or private cheats can evade it until Blizzard adds their signatures. The user-mode design means a kernel-level cheat (rootkit) can hide from Warden entirely, though Blizzard has historically chosen not to use kernel-level anti-cheat to avoid the controversy and compatibility issues that affect systems like Vanguard (Valorant) or EasyAntiCheat.

Server-side movement validation catches gross violations (speed hacks, teleports) but cannot detect subtle exploits (slightly-too-fast movement, wall-climbing using geometry seams) because the server does not maintain a physics simulation or full collision mesh.

---

## Key Sources

### Primary (Official / First-Party)

| Source | Author | Year | Content |
|--------|--------|------|---------|
| [GDC 2013: Network Serialization and Routing in World of Warcraft](https://www.gdcvault.com/play/1017733/Network-Serialization-and-Routing-in) | Joe Rumsey (Blizzard) | 2013 | JAM serialization, code generation, routing, protocol negotiation |
| [GDC 2013 Slides (PDF)](http://rumsey.org/GDC2013/JAM%20Presentation%20GDC%20Final.pdf) | Joe Rumsey | 2013 | Slide deck from the talk |
| Celestalon Blue Post on Spell Batching | Chadd "Celestalon" Nervig (Blizzard) | June 2014 | Confirmed 400ms batch window; announced reduction for WoD |
| [Classic Spell Batching Announcement](https://us.forums.blizzard.com/en/wow/t/spell-batching-in-classic/137118) | Blizzard (Blue Post) | 2019 | Re-implementation of 400ms batching for Classic |
| [CVar SpellQueueWindow](https://wowpedia.fandom.com/wiki/CVar_SpellQueueWindow) | Wowpedia | -- | Documentation of the spell queue window mechanic |

### Reverse Engineering / Community

| Source | Content |
|--------|---------|
| [WoWDev Wiki](https://wowdev.wiki/) | Packet structures, opcodes, file formats |
| [WoWDev Wiki: Opcodes](https://wowdev.wiki/Opcodes) | Movement packet opcodes (MSG_MOVE_*) |
| [TrinityCore: Movement Documentation](https://trinitycore.atlassian.net/wiki/spaces/tc/pages/721256449/Movement) | Event-driven packets, dead reckoning, heartbeat, spline movement |
| [TrinityCore AntiCheat PR](https://github.com/TrinityCore/TrinityCore/pull/23509) | Server-side speed/teleport/fly detection implementation |
| [MaNGOS / TrinityCore Source](https://github.com/TrinityCore/TrinityCore) | Open-source WoW server emulator; reference for server-side combat resolution |
| [Warden (Wowpedia)](https://wowpedia.fandom.com/wiki/Warden_(software)) | Anti-cheat system documentation |
| [FreecraftCore Serializer](https://github.com/FreecraftCore/FreecraftCore.Serializer) | .NET implementation inspired by JAM |
| [Melee Leeway Testing (Barrens Chat)](https://barrens.chat/viewtopic.php?t=1659) | Community-verified leeway formula |

### Academic

| Paper | Authors | Year | Content |
|-------|---------|------|---------|
| [Traffic Analysis and Modeling for World of Warcraft](https://publik.tuwien.ac.at/files/pub-et_12119.pdf) | Suznjevic et al. (TU Wien) | 2007 | Packet rates, bandwidth, session analysis |
| [Characterizing the Gaming Traffic of World of Warcraft](https://www.researchgate.net/publication/220553999) | Various | 2012 | Traffic patterns across game scenarios and network technologies |

### Community Analysis

| Source | Content |
|--------|---------|
| [Spell Batching Impact (wowisclassic.com)](https://www.wowisclassic.com/en/news/spell-batching-impact-ptr/) | Detailed analysis of 1.13.7 batching changes |
| [Spell Batching Guide (Wowhead)](https://www.wowhead.com/classic/news/how-the-spell-batching-change-in-1-13-7-impacts-everything-in-wow-classic-320587) | Community guide to spell batching mechanics |
| [Classic Warrior Wiki: Spell Batching](https://github.com/magey/classic-warrior/wiki/Spell-batching) | Technical breakdown of batching for warriors |
| [LatencyGuard Addon](https://github.com/Kkthnx-Wow/LatencyGuard) | Dynamic SpellQueueWindow adjustment based on latency |
| [Maxroll: Spell Queue Window Guide](https://maxroll.gg/wow/resources/spell-queue-window) | Practical guide to tuning SpellQueueWindow |
