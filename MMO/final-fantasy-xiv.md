# Final Fantasy XIV: A Realm Reborn -- Netcode Deep-Dive

## Overview

Final Fantasy XIV (FFXIV) takes an unusual stance in the real-time multiplayer
landscape: rather than engineering sophisticated prediction and reconciliation
systems to hide latency, the game **designs its mechanics around the
limitations of its netcode**. The 2.5-second Global Cooldown (GCD), generous
AoE telegraph windows, and the slidecast buffer all exist in part because the
underlying network architecture cannot support the tight feedback loops found in
shooters or fighting games.

Director/Producer Naoki Yoshida and the development team at Square Enix, based
in Tokyo, primarily play and test on the Japanese datacenters at 10-30 ms ping.
Many design decisions -- animation lock timing, snapshot leniency, the pace of
combat -- feel natural at that latency but degrade noticeably for players on
North American (~80-150 ms), European (~120-200 ms), or Oceanian (~180-300 ms)
datacenters. The Fall Guys crossover event in Patch 6.51 made this painfully
visible when platforming precision collided with a system built for tab-target
cooldown management.

Despite these constraints, FFXIV supports millions of concurrent players across
dozens of World servers, and its netcode -- while far from state-of-the-art --
is a fascinating case study in pragmatic trade-offs.

---

## Architecture

### Protocol: TCP-Only

FFXIV uses **TCP exclusively** for all client-server communication -- lobby,
chat, zone, and combat data all travel over a single reliable, ordered stream.
This is uncommon for a game with real-time positional gameplay. Most modern
action games use UDP (or a custom reliable-UDP layer) for latency-sensitive
state, but FFXIV shares its TCP-only approach with other MMOs of its era (World
of Warcraft also uses TCP). The choice guarantees packet ordering and delivery
at the cost of head-of-line blocking and retransmission delays.

### Client-Server Model

FFXIV is a traditional client-server architecture:

- **One authoritative server per World** handles game logic, combat resolution,
  loot, and persistence.
- **Clients** render the world, handle input, perform local animation, and
  report their position.
- **Movement is client-authoritative** -- the client tells the server where the
  player is.
- **Combat outcomes are server-authoritative** -- every ability result requires a
  full round-trip to the server.

### Tick Rates and Update Frequencies

| System                     | Frequency         | Interval  |
|:---------------------------|:------------------|:----------|
| Client position reports    | ~3.3 Hz           | ~300 ms   |
| Position relay to others   | ~1 Hz             | ~1000 ms  |
| DoT / HoT server ticks    | 0.33 Hz           | 3000 ms   |
| GCD base                   | 0.4 Hz            | 2500 ms   |
| Animation lock (oGCD)      | --                | ~600 ms   |

The ~3.3 Hz client position update rate is notably slow. For comparison, World
of Warcraft updates positions at ~20 Hz, and Overwatch runs at 63 Hz. This low
frequency is the root cause of many of FFXIV's distinctive quirks: ghost AoE
hits, slidecast windows, and the sluggish feel of other players' movement.

---

## Movement: Client-Authoritative

### How It Works

The FFXIV client determines the player's position locally using standard
movement input and collision detection. Approximately every 300 ms, the client
sends an `UpdatePosition` packet to the server containing the player's current
coordinates, rotation, and animation state.

The server **trusts** the reported position with minimal validation. Server-side
checks are limited to gross violations:

- Teleportation (position delta exceeds plausible movement speed)
- Out-of-bounds coordinates
- Movement during hard crowd-control states the server tracks

The server does **not** simulate player physics, pathfinding, or collision. It
simply records the last reported position and uses it for all game-logic checks.

### Packet Structure (from Sapphire emulator)

```cpp
// Client -> Server: Position update (~3.3 Hz)
// Based on Sapphire's FFXIVIpcUpdatePosition
struct UpdatePosition
{
    float    rotation;              // Character facing direction (radians)
    uint8_t  animationType;         // Current movement animation ID
    uint8_t  animationState;        // Animation blend state
    uint8_t  clientAnimationType;   // Client-side animation override
    uint8_t  headPosition;          // Head look direction
    float    x, y, z;              // World-space coordinates
    uint8_t  unknown[4];           // Padding / reserved
};

// Client -> Server: Instance-specific position (dungeons, raids)
// Includes interpolation hints for smoother relay to other clients
struct UpdatePositionInstance
{
    float    rotation;
    float    interpolateRotation;   // Target rotation for smoothing
    uint32_t flags;                 // Movement state flags
    float    x, y, z;              // Current position
    float    interpX, interpY, interpZ;  // Interpolation target
    uint32_t unknown;
};
```

### Pseudo-Code: Client Position Reporting

```cpp
// Runs on the client, called every frame
void Client::update(float deltaTime) {
    // 1. Process input and move locally (client-authoritative)
    Vector3 movement = getMovementInput() * moveSpeed * deltaTime;
    localPlayer.position += movement;
    localPlayer.position = applyCollision(localPlayer.position);
    localPlayer.rotation = calculateFacing(inputDirection);

    // 2. Render immediately -- no waiting for server confirmation
    renderer.drawCharacter(localPlayer.position, localPlayer.rotation);

    // 3. Throttle network updates to ~3.3 Hz
    positionTimer += deltaTime;
    if (positionTimer >= 0.300f) {  // ~300ms interval
        positionTimer = 0.0f;

        UpdatePosition packet;
        packet.rotation       = localPlayer.rotation;
        packet.animationType  = localPlayer.currentMoveAnim;
        packet.animationState = localPlayer.animBlendState;
        packet.x = localPlayer.position.x;
        packet.y = localPlayer.position.y;
        packet.z = localPlayer.position.z;

        sendToServer(OPCODE_UPDATE_POSITION, &packet);
    }
}
```

### Consequences: Cheating

Because the client is the authority on position, manipulation is
straightforward:

- **Speed hacks**: modify the movement speed multiplier; the server rarely
  rejects slightly elevated speeds (~20% faster passes most server checks in
  many zones).
- **Teleport hacks**: set position directly; more aggressive checks exist for
  large jumps but can be circumvented by "warping" in small increments.
- **Fly hacks**: disable gravity and collision client-side; report arbitrary Y
  coordinates.

FFXIV does **not** ship a kernel-level anti-cheat. Detection relies on
server-side heuristics and player reports, both of which are limited by the
300 ms position sample rate.

---

## Remote Players

### The Problem

Other players' positions are relayed at an even lower rate than the client's own
reporting. The server receives position updates at ~3.3 Hz from each client, but
**distributes them to other clients at roughly ~1 Hz or slower** -- likely
batched and prioritized by proximity. Combined with the sender's 300 ms update
interval and the receiver's own rendering pipeline, remote players appear
**500-1000 ms behind their actual position**.

### What You See

- Characters "slide" to new positions rather than moving fluidly.
- Quick direction changes produce **"moonwalking"** -- the movement animation
  plays in one direction while the character's position interpolates in another.
- In crowded areas (Limsa Lominsa, hunt trains), players appear to teleport
  short distances as position updates arrive in bursts.

### No Dead Reckoning

FFXIV performs **no extrapolation or dead reckoning** for other players. The
client simply smooths (lerps) between the last known position and the newly
received position. If a new update does not arrive, the remote character holds
its last position rather than continuing along a predicted path.

### Pseudo-Code: Remote Player Smoothing

```cpp
// For each remote player visible on screen
struct RemotePlayer {
    Vector3 displayPosition;     // What we render
    Vector3 lastServerPosition;  // Previous update from server
    Vector3 targetPosition;      // Most recent update from server
    float   interpolationAlpha;  // 0.0 = lastPos, 1.0 = targetPos
    float   interpDuration;      // Time to lerp (roughly matches update interval)
};

// Called when a position packet arrives for this remote player (~1 Hz)
void RemotePlayer::onPositionUpdate(Vector3 newPos, float newRotation) {
    lastServerPosition = displayPosition;  // Start from where we are now
    targetPosition     = newPos;
    interpolationAlpha = 0.0f;

    // Duration is roughly the expected interval between updates
    // Slightly padded to avoid snapping if the next packet is late
    interpDuration = 1.0f;  // ~1 second between relay updates
}

// Called every render frame
void RemotePlayer::update(float deltaTime) {
    interpolationAlpha += deltaTime / interpDuration;
    interpolationAlpha = clamp(interpolationAlpha, 0.0f, 1.0f);

    // Simple linear interpolation -- no extrapolation beyond target
    displayPosition = lerp(lastServerPosition, targetPosition, interpolationAlpha);

    // If alpha reaches 1.0 and no new packet has arrived, the character
    // simply stops at targetPosition. No prediction, no dead reckoning.
}
```

### Why This Matters

In casual gameplay (questing, city socializing), the lag is barely noticeable.
In group content -- especially PvP and the Fall Guys crossover event -- the
disconnect between where a player appears on your screen and where the server
thinks they are leads to frustrating inconsistencies. You cannot reliably chase
or follow another player's exact movements.

---

## Action System: Server-Authoritative

### The Round-Trip

Every offensive ability, healing spell, and buff in FFXIV requires a full
round-trip to the server:

1. **Client presses ability** -> client sends `SkillHandler` packet to server.
2. **Client applies local animation lock** (500 ms for oGCDs, 100 ms for GCD
   casts).
3. **Server validates** the action (cooldown ready, resources available, in
   range, line of sight).
4. **Server resolves** damage/healing, applies status effects, generates threat.
5. **Server sends `Effect` packet** back to client with results and a new
   animation lock value (typically 600 ms for oGCDs).
6. **Client receives response**, plays hit/damage effects, updates UI, and
   **overwrites** the local animation lock with the server's value.

There is **no client-side prediction of outcomes**. The client does not
speculatively apply damage numbers, HP changes, or status effects. Everything
waits for the server's `Effect` packet.

### Packet Structures

```cpp
// Client -> Server: Ability request
struct SkillHandler
{
    uint8_t  type;                // Action category (spell, weaponskill, ability)
    uint32_t actionId;            // Which ability (e.g., 0x1D4F = Fire IV)
    uint16_t sequence;            // Monotonic counter for request tracking
    uint64_t targetId;            // Entity ID of the target
    uint16_t itemSourceSlot;      // For item usage
    uint16_t itemSourceContainer;
};

// Server -> Client: Ability result
// (Simplified from FFXIVIpcEffect)
struct EffectResult
{
    uint32_t actionId;
    uint32_t sourceId;
    uint16_t sequence;            // Matches the request's sequence number
    float    animationLock;       // Server-dictated lock duration (usually 0.6s)
    // ... target IDs, damage values, status effects, etc.
};
```

### Pseudo-Code: Ability Request Flow

```cpp
// === CLIENT SIDE ===
void Client::useAbility(uint32_t actionId, uint64_t targetId) {
    if (animationLockTimer > 0.0f) return;  // Blocked by animation lock
    if (gcdTimer > 0.0f && isGCD(actionId)) return;  // On GCD

    // Send request to server
    SkillHandler packet;
    packet.actionId = actionId;
    packet.targetId = targetId;
    packet.sequence = nextSequence++;
    sendToServer(OPCODE_SKILL_HANDLER, &packet);

    // Apply LOCAL animation lock immediately (500ms for oGCDs)
    animationLockTimer = 0.5f;

    // Start ability animation locally (visual only, no gameplay effect yet)
    playAnimation(actionId);

    // Record when we sent the request (for latency tracking)
    pendingRequests[packet.sequence] = getCurrentTime();
}

// Called when server's Effect packet arrives
void Client::onEffectResult(EffectResult& result) {
    // Apply damage numbers, HP changes, status effects from server data
    applyServerEffects(result);

    // CRITICAL: Server OVERWRITES the animation lock timer
    // Does NOT account for time already elapsed during round-trip
    animationLockTimer = result.animationLock;  // Typically 0.6s

    // At 0ms ping: total lock = 0.5s local + 0.0s wait + 0.6s server = ~0.6s
    //   (server response arrives almost instantly, overwrites with 0.6s)
    // At 100ms ping: total lock = 0.5s local + 0.1s wait + 0.6s server = ~0.7s
    // At 200ms ping: total lock = 0.5s local + 0.2s wait + 0.6s server = ~0.8s
    //   (double weaving becomes unreliable)
}
```

---

### The Double Animation Lock Problem

The core issue: when the server responds with a 600 ms animation lock, the
client sets `animationLockTimer = 0.6` **at the moment the response arrives**,
not at the moment the ability was used. The round-trip time (RTT) is effectively
added on top of the animation lock.

```
Timeline at 0ms ping (ideal, as played by devs in Tokyo):
  t=0.000  Use oGCD A -> local lock = 0.5s
  t=0.000  Server receives, processes
  t=0.000  Client receives Effect -> lock overwritten to 0.6s
  t=0.600  Lock expires, can use oGCD B
  Total lockout: 0.6s  (exactly the server's intended duration)

Timeline at 150ms ping (typical NA player):
  t=0.000  Use oGCD A -> local lock = 0.5s
  t=0.075  Server receives request (75ms one-way)
  t=0.075  Server processes, sends Effect
  t=0.150  Client receives Effect -> lock overwritten to 0.6s
  t=0.750  Lock expires, can use oGCD B
  Total lockout: 0.75s  (0.6s + 0.15s RTT -- 25% penalty)

Timeline at 300ms ping (OCE player before OCE datacenter):
  t=0.000  Use oGCD A -> local lock = 0.5s
  t=0.150  Server receives request
  t=0.150  Server processes, sends Effect
  t=0.300  Client receives Effect -> lock overwritten to 0.6s
  t=0.900  Lock expires, can use oGCD B
  Total lockout: 0.9s  (0.6s + 0.3s RTT -- 50% penalty)
```

With a 2.5-second GCD and two oGCDs to weave between GCDs ("double weaving"),
each oGCD gets roughly a 0.7s window. At 150 ms ping, the effective lock of
0.75s makes double weaving tight. At 200 ms+, it becomes physically impossible
without clipping the GCD.

### XivAlexander: The Community Fix

[XivAlexander](https://github.com/Soreepeong/XivAlexander) is a third-party
add-on that intercepts the server's `Effect` packet and subtracts the measured
round-trip time from the animation lock value before the client applies it.
This makes the effective lockout duration closer to the server's intended 600 ms
regardless of the player's ping.

```cpp
// === XivAlexander's latency mitigation (simplified) ===

// Mode 1: "Subtract Latency"
void XivAlexander::onEffectPacket(EffectResult& result) {
    float rtt = measureRoundTripTime();  // via ICMP ping or TCP_INFO

    // Subtract the round-trip time from the server's animation lock
    float adjustedLock = result.animationLock - rtt;

    // Safety: never allow negative lock (would let actions fire instantly)
    // Also prevent extremely low values that might trigger server-side
    // sanity checks (the server knows roughly when you sent the request)
    adjustedLock = max(adjustedLock, 0.05f);

    result.animationLock = adjustedLock;

    // Now the client will unlock at approximately:
    //   t_ability_used + rtt + (0.6 - rtt) = t_ability_used + 0.6
    // Which is the intended behavior regardless of ping.
}

// Mode 2: "Simulate RTT" -- pretend ping is always ~75ms
// Used when actual RTT measurement is unreliable
void XivAlexander::onEffectPacket_SimMode(EffectResult& result) {
    const float SIMULATED_RTT = 0.075f;
    result.animationLock = max(result.animationLock - SIMULATED_RTT, 0.05f);
}

// Edge case: if RTT > 500ms, the local 500ms lock has already expired
// before the server response arrives. Without intervention, the client
// would briefly allow action input, then re-lock on response.
// XivAlexander tracks pending actions and forces a combined lock to
// prevent triggering the server's anti-cheat.
```

XivAlexander effectively normalizes the gameplay experience across all
latencies, making double weaving reliable at any ping. The add-on's README
acknowledges this is "only a step away from flat out cheating" since it
effectively claims sub-zero latency to the server, but it compensates for a
design deficiency rather than providing an unfair advantage.

---

## Slidecasting

### What It Is

"Slidecasting" is the technique of moving during the final ~500 ms of a spell's
cast bar without interrupting the cast. Every caster job relies on this for
mobility.

### Why It Works

The cast time displayed in the UI tooltip is **~500 ms longer** than the
server's actual completion threshold. This buffer was intentionally added as
compensation for client-server synchronization latency -- preventing cases where
the cast bar appears to complete on screen but the server has not yet registered
it.

```
Example: Fire IV tooltip says "2.80s cast time"

  Server's actual threshold:  ~2.30s
  Displayed cast bar:          2.80s
  Slidecast window:           ~0.50s (the difference)
```

When the player starts moving at 2.35s into a 2.80s displayed cast, the
client sends a position update to the server. But the server has already
registered the cast as complete at ~2.30s, so the movement does not interrupt
it. The cast succeeds, and the player gets a head start on repositioning.

### Pseudo-Code: Slidecast Window

```cpp
// === SERVER SIDE ===
struct ActiveCast {
    uint32_t actionId;
    float    serverCastTime;     // The real threshold (e.g., 2.30s)
    float    displayCastTime;    // What the client shows (e.g., 2.80s)
    float    elapsed;
    bool     completed;
};

void Server::updateCast(ActiveCast& cast, float deltaTime) {
    if (cast.completed) return;

    cast.elapsed += deltaTime;

    // Server considers the cast complete at the REAL threshold,
    // which is ~500ms BEFORE the client's cast bar finishes
    if (cast.elapsed >= cast.serverCastTime) {
        cast.completed = true;
        resolveSpell(cast.actionId);

        // Note: the client's cast bar is still filling up for another ~500ms!
        // Any movement during this tail period is harmless because
        // the server has already accepted the cast.
    }
}

// === CLIENT SIDE ===
void Client::onMoveDuringCast() {
    // Client checks against the DISPLAY cast time
    if (currentCast.elapsed < currentCast.displayCastTime) {
        // The client thinks the cast is still in progress.
        // It sends a position update to the server.
        // If elapsed > serverCastTime (the slidecast window), the server
        // has already completed the cast -- movement is safe.
        //
        // If elapsed < serverCastTime, the server will interrupt the cast.
        //
        // The ~500ms buffer means the player can move during the final
        // portion of the displayed bar:
        //
        //   |=================|===SAFE===|
        //   0            serverTime    displayTime
        //                (~2.30s)      (~2.80s)
    }
}

// Slidecast window calculation:
//   window = displayCastTime - serverCastTime
//   window ≈ 0.50s (fixed) + additional ping-dependent slush
//
// A player can begin moving when:
//   elapsed >= serverCastTime
//   elapsed >= displayCastTime - 0.50s  (approximately)
```

### Practical Notes

- The 500 ms is a **fixed baseline** -- even at 0 ms ping (e.g., inside the
  server room), you can slidecast for the last 500 ms.
- Additional ping-based leniency exists on top of this, giving high-ping players
  a slightly wider effective slidecast window (their position update takes longer
  to reach the server, during which the server may complete the cast).
- Shorter casts have proportionally larger slidecast windows relative to their
  total duration, making them easier to slidecast.
- The Dalamud plugin
  [SlideCast](https://github.com/Haplo064/SlideCast) adds a visual indicator on
  the cast bar showing exactly when the safe window begins.

---

## AoE / Snapshot System

### How Snapshotting Works

Boss abilities in FFXIV use a **snapshot** model: the server checks positions at
a specific moment in time and determines who is hit. This snapshot occurs at
**cast bar completion** for cast-based abilities, or during the animation for
instant-cast mechanics.

The critical detail: the server's knowledge of a player's position is only as
fresh as the **last position update it received** -- which can be up to 300 ms
old (one position tick interval) plus the player's one-way network latency.

### The Ghost Hit Problem

"Ghost hits" occur when a player moves out of an AoE on their screen but the
server's last-known position still places them inside the danger zone at
snapshot time.

```
Scenario: Boss casts a circular AoE, cast bar finishes at t=5.000s

  Player's actual movement:
    t=4.500  Player starts running out of AoE
    t=4.700  Player is visually outside the AoE on their screen
    t=5.000  Cast bar completes, player is clearly out on their screen

  Server's view (100ms ping, worst-case 300ms position staleness):
    t=4.500  Player starts moving
    t=4.600  Server receives position from t=4.300 (still inside AoE)
    t=4.900  Server receives position from t=4.600 (barely inside AoE)
    t=5.000  SNAPSHOT -- server uses last known position from t=4.600
             Server says: PLAYER IS HIT

  The player was out on their screen at t=4.700, but the server's view
  was 400ms behind (300ms tick + 100ms ping). Ghost hit.
```

### Pseudo-Code: Server-Side AoE Resolution

```cpp
// === SERVER SIDE ===

// Position storage: updated whenever an UpdatePosition packet arrives
struct PlayerState {
    uint64_t playerId;
    Vector3  lastKnownPosition;   // From most recent UpdatePosition packet
    float    lastUpdateTime;      // Server time when the update was processed
};

// Called when a boss's cast bar completes
void Server::resolveAoE(AoEAbility& ability, float snapshotTime) {
    Vector3 aoeCenter  = ability.position;
    float   aoeRadius  = ability.radius;

    for (auto& player : playersInZone) {
        // Use the LAST KNOWN POSITION -- which may be stale by up to:
        //   positionTickInterval (300ms) + playerPing (one-way)
        //
        // The server does NOT extrapolate or predict where the player
        // might be "right now" -- it uses whatever was last reported.
        float staleness = snapshotTime - player.lastUpdateTime;
        // staleness could be 0-400+ ms depending on timing and ping

        float distance = length(player.lastKnownPosition - aoeCenter);

        if (distance <= aoeRadius) {
            // HIT -- even if the player moved out on their screen
            applyDamage(player.playerId, ability.damage);
            applyDebuff(player.playerId, ability.debuffId);
        }
    }
}

// AoE shapes: circular (distance check), conal (angle + distance),
// rectangular (AABB or OBB), donut (min + max radius), line (capsule)
// All use the same stale-position principle.

// Friendly AoE snapshotting works similarly for buff resolution:
// If a damage buff is active on the caster at snapshot time, it amplifies
// the entire attack. DoTs/HoTs snapshot their buff state at cast time
// and retain that potency for their full duration.
```

### Position Forcing

Using an ability (even an instant oGCD) forces the client to send a position
update immediately, outside the normal 300 ms cadence. Experienced players
exploit this: pressing an instant ability just before a snapshot effectively
"refreshes" the server's knowledge of their position, reducing the chance of a
ghost hit. This is sometimes called a **"position lock"** and is part of
high-level raid optimization.

---

## Protocol Details

### TCP-Only Transport

All FFXIV traffic (lobby, zone, chat, combat) travels over TCP on a set of
well-known ports (54992-54994 for game data, 55006-55007 for lobby, among
others). The game does not use UDP for any gameplay communication.

TCP provides:
- **Guaranteed delivery** -- no lost packets.
- **Ordered delivery** -- actions always arrive in sequence.
- **Built-in flow and congestion control**.

The trade-offs are:
- **Head-of-line blocking** -- one lost packet stalls all subsequent data.
- **Retransmission delays** -- can cause periodic hitches (the "900X"
  disconnections).
- **Higher minimum latency** -- TCP handshake and ACK overhead.

For a 2.5-second GCD MMO, TCP's reliability is more valuable than UDP's speed.
The game never needs to tolerate lost position updates the way a 128-tick
shooter would.

### Compression: zlib to Oodle

Prior to Patch 6.3 (January 2023), FFXIV compressed packets with **zlib**, a
stateless compression algorithm. Each packet could be decompressed
independently, making it straightforward for third-party tools (ACT, packet
dissectors) to read network traffic via packet capture.

In Patch 6.3, Square Enix migrated to **Oodle Network Compression** by RAD Game
Tools. Oodle uses **stateful compression** -- the compressor and decompressor
maintain a shared dictionary that evolves over the session. This means:

- A packet cannot be decompressed without the full preceding session context.
- If any packet is missed (e.g., capture started mid-session), all subsequent
  packets are unreadable.
- Tools must capture from the very start of a login session, or inject into the
  game process to read decompressed data directly.

The community responded with **Deucalion**, an injectable library that hooks
the game's network layer and reads packets after the game itself has
decompressed them, bypassing the Oodle state requirement.

### Opcode Randomization

FFXIV's IPC (Inter-Process Communication) opcodes -- the identifiers that tell
the client what type of data a packet contains -- are **randomized every patch**.
An `Effect` packet might be opcode `0x0234` in Patch 7.0 and `0x01A7` in Patch
7.01.

This is widely believed to be an anti-modding measure, but the community
maintains shared opcode repositories
([SapphireServer/FFXIVOpcodes](https://github.com/SapphireServer/FFXIVOpcodes),
[karashiiro/FFXIVOpcodes](https://github.com/karashiiro/FFXIVOpcodes)) that are
updated within hours of each patch by reverse-engineering the client binary.

---

## Known Issues

### 1. Fall Guys Crossover (Patch 6.51)

The "Blunderville" event added competitive platforming mini-games requiring
precise spatial awareness and tight movement timing -- exactly the kind of
gameplay FFXIV's netcode handles worst. Players experienced:

- **Position desync**: appearing to reach the finish line first but losing to
  another player whose server-side position was ahead.
- **Phantom collisions**: being knocked by obstacles they had visually dodged
  1+ seconds prior.
- **Inconsistent pickups**: items going to a player who appeared farther away.

The event was widely nicknamed "Latency Guys" and reignited long-standing
complaints about the ~3.3 Hz position update rate.

### 2. Ghost AoE Hits

As detailed in the AoE/Snapshot section, players routinely take damage from
attacks they dodged on their screen. This is an inherent consequence of the
server using stale position data. Higher-ping players must dodge **earlier**
than the visual telegraph suggests, leading to the community mantra: "everything
on your screen is a lie."

### 3. JP-Centric Design Bias

The development team in Tokyo plays and tests on JP datacenters with 10-30 ms
ping. The animation lock system, slidecast windows, and snapshot leniency all
feel correct at that latency. For NA (~100 ms), EU (~150 ms), and pre-Oceanian
datacenter players (~250 ms), the experience degrades:

- Double weaving becomes inconsistent or impossible without XivAlexander.
- AoE dodges require larger safety margins.
- Some jobs with tight oGCD windows (e.g., Ninja, Monk) lose DPS purely due to
  latency.

When JP ISP routing issues caused increased latency within Japan, Square Enix
issued official communications and investigated promptly -- a response speed
that contrasted with years of community complaints from other regions going
largely unaddressed.

### 4. Cheating from Client-Authoritative Movement

The combination of client-authoritative positioning, no kernel-level anti-cheat,
and a 300 ms server polling rate means:

- **Speed hacks** are trivial and difficult to detect at moderate levels.
- **Teleport bots** farm gathering nodes and FATEs across zones.
- **Fly hacks** allow out-of-bounds movement.

Detection relies primarily on player reports and basic server-side velocity
checks, both of which have significant blind spots.

---

## Key Sources

### Official

- **GDC 2014 -- "Behind the Realm Reborn"** (Naoki Yoshida): Covered the
  rebuild from 1.0 to 2.0, including architecture decisions. Available on
  [GDC Vault](https://gdcvault.com/play/1020796/Behind-the-Realm) and
  [Internet Archive](https://archive.org/details/201403828297JPUSHLV02dee1300).
- **Lodestone Technical Posts**: Square Enix's official network troubleshooting
  guidance, particularly the
  [network delay and packet loss notice](https://eu.finalfantasyxiv.com/lodestone/topics/detail/6c7c327f44bee8582950f88264a451d5a8daa66f).

### Community Reverse Engineering

- **[xiv.dev](https://xiv.dev/game-internals/actions)**: Community wiki
  documenting game internals, including
  [animation lock](https://xiv.dev/game-internals/actions/animation-lock),
  action processing, and packet structures.
- **[Sapphire](https://github.com/SapphireServer/Sapphire)**: Open-source FFXIV
  server emulator written in C++. Its packet definitions
  ([ClientZoneDef.h](https://github.com/SapphireServer/Sapphire/blob/develop5.58/src/common/Network/PacketDef/Zone/ClientZoneDef.h),
  [ServerZoneDef.h](https://github.com/SapphireServer/Sapphire/blob/develop5.58/src/common/Network/PacketDef/Zone/ServerZoneDef.h))
  are the most detailed public documentation of FFXIV's network protocol.
- **[XivAlexander](https://github.com/Soreepeong/XivAlexander)**: The
  double-weave latency fix. Source code and
  [wiki](https://github.com/Soreepeong/XivAlexander/wiki/Interface:-Main-Menu)
  contain the most thorough analysis of the animation lock problem.
- **[FFXIVClientStructs](https://github.com/aers/FFXIVClientStructs)**: Reverse-
  engineered client data structures used by the Dalamud plugin framework.
- **[Machina](https://github.com/ravahn/machina)**: Network capture library for
  FFXIV, including Oodle decompression support.
- **[Deucalion](https://github.com/perchbird/Deucalion)**: Injectable packet
  reader that bypasses Oodle's stateful compression.
- **[FFXIVOpcodes](https://github.com/karashiiro/FFXIVOpcodes)**: Community-
  maintained opcode mappings updated each patch.

### Community Analysis

- **[AkhMorning -- Raiding Fundamentals](https://www.akhmorning.com/resources/raiding-fundamentals/)**:
  Comprehensive guide to snapshotting, server ticks, DoT/HoT mechanics, and
  other "unconveyed information" about FFXIV's engine.
- **[SlideCast Plugin](https://github.com/Haplo064/SlideCast)**: Dalamud plugin
  that visualizes the slidecast window; its source confirms the 500 ms
  (50 centisecond) threshold.
- **[Forum: "Why the netcode issue exists, and why it cannot be fixed"](https://gamefaqs.gamespot.com/boards/678050-final-fantasy-xiv-online-a-realm-reborn/67840591)**:
  Long-running community discussion of FFXIV's fundamental networking
  limitations.
