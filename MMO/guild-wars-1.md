# Guild Wars 1: Netcode Deep-Dive

## Overview

Guild Wars 1 (2005, ArenaNet) is arguably the **best-documented MMO netcode** in the industry, thanks to co-founder **Pat Wyatt** (who also led networking on Warcraft, Diablo, and StarCraft at Blizzard). The architecture is a fully **server-authoritative**, **TCP-only**, **instanced** design that ran **24/7/365 with zero scheduled downtime** -- a claim almost no other MMO of its era could make.

The game was designed from the start with an economic constraint: no monthly subscription fee. This meant ArenaNet could not afford expensive server farms, so every architectural decision optimized for efficiency. The result was a system of **18 specialized server types** running as dynamically-loaded DLLs, a custom RPC protocol with versioned inter-server communication, and a combat system that deliberately used **animation timing to mask network latency** rather than client-side prediction.

Key architectural characteristics:

| Property | Value |
|----------|-------|
| **Transport** | TCP exclusively (no UDP) |
| **Authority** | Fully server-authoritative |
| **World Model** | Instanced (2-150 players per instance) |
| **Server Types** | 18 specialized microservices at launch |
| **Downtime** | Zero scheduled maintenance; individual services had 2+ years uptime |
| **Prediction Model** | No dead reckoning; client predicts local pathfinding, server corrects |
| **Latency Masking** | Two-stage commit: animation duration > RTT |
| **Language** | C++ |
| **Core Netcode Size** | ~50,000 lines of code |

**Primary sources:** Pat Wyatt's [HandmadeCon 2015 interview](https://www.codeofhonor.com/blog/handmadecon-2015-interview-transcript) (interviewed by Casey Muratori), the [Code of Honor](https://www.codeofhonor.com/blog/) blog series, and Stephen Clarke-Willson's [GDC 2017 talk](https://www.gdcvault.com/play/1024018/-Guild-Wars-Microservices-and) on Guild Wars microservices.

---

## Architecture

### Why TCP Exclusively

Guild Wars used TCP for all communication -- client-to-server and server-to-server. This was unusual for an online game even in 2005, when most action-oriented titles used UDP. Wyatt explained the rationale:

> "Everything is TCP... almost every single message that you would send is something that you would have wanted to know correctly."

The decision was driven by three factors:

1. **Message reliability requirements.** Nearly every game message in Guild Wars is state-critical. Wyatt gave the example of item pickup: "I picked up this [item] -- that is not a message that you can lose because if you did then multiple people could pick up the same item." Unlike an FPS where a dropped position update is superseded by the next one, Guild Wars messages represented discrete state transitions that could not be skipped.

2. **Firewall traversal.** TCP connections on ports 80 and 443 pass through virtually any firewall or NAT without configuration. UDP traffic is frequently blocked or requires explicit port forwarding. For a game targeting a broad audience (including players on corporate and university networks), TCP eliminated an entire category of connectivity support issues.

3. **Engineering pragmatism.** Building reliable ordered delivery over UDP is substantial work. The team chose to ship with TCP and evaluate whether UDP was needed later: "We're gonna use TCP and then decide later on if we needed to use UDP... it turned out that it just wasn't a big issue for the type of game that we were doing."

**Trade-offs accepted:**

- **Head-of-line blocking.** A single lost TCP segment stalls all subsequent data on that connection. For a 200ms RTT connection, a packet loss event adds ~200ms of additional latency to everything behind it. Guild Wars mitigated this by keeping instance sizes small (reducing total message volume per connection) and by designing gameplay that tolerated occasional latency spikes (animation-length actions rather than twitch reactions).

- **TCP window tuning.** The team invested significant effort in TCP window size management to enable high throughput over long-distance connections (US to Asia) without exposing servers to DDoS amplification attacks.

- **No unreliable channel for cosmetic data.** Effects like particle positions or non-critical animations could not be sent on a lossy "fire and forget" channel. Everything went through the same reliable, ordered stream.

### 18 Server Types (Microservices Before the Term Existed)

ArenaNet's philosophy was to decompose functionality into the smallest independently-scalable units:

> "If we can define a set of operations that clearly form something that can all be done within the same set of data, we're gonna make that into a separate server if we can."

This predated the "microservices" terminology by over a decade. Each server type was implemented as a **DLL** loaded by a common host executable called `ArenaSrv`. The known server types include:

| Server Type | Responsibility |
|-------------|---------------|
| **ArenaSrv** | Host executable / DLL loader |
| **LoginSrv** | Authentication |
| **LobbySrv** | Presence, friends lists, guild info, persistent client connection |
| **SiteSrv** | Matchmaking / instance assignment |
| **GameSrv** | Gameplay instance host (one per active instance) |
| **DbSrv** | Persistent character and world state |
| **DbCacheSrv** | Database caching layer |
| **FileSrv** | Asset distribution with delta compression |
| **ChatSrv** | Chat routing |
| **GuildSrv** | Guild management |
| **TournamentSrv** | PvP tournament orchestration |
| **PatchSrv** | Client update distribution |
| **LogSrv** | Centralized logging |
| **MonitorSrv** | Health monitoring and alerting |
| **CrashSrv** | Crash dump collection and analysis |
| (+ others) | Additional infrastructure services |

File servers were distributed globally (Japan, Taiwan, Korea, two US data centers, Europe), with each server capable of saturating "its uplink at about 400 Megabit" during peak load.

### Custom RPC Protocol

Servers communicated via a custom RPC protocol built on TCP:

> "The servers speak a different protocol, an RPC protocol and it happened to be a custom one... today you would use protobufs or thrift or MsgPack."

The protocol was embedded in a shared library (`SrvSocket`) linked by every server type, enabling consistent serialization and versioning across the entire cluster. Each message carried a protocol version, allowing old and new servers to coexist during rolling upgrades.

### Single-Process Development Mode

A distinctive feature of the architecture was that developers could load **all 18 server DLLs into a single process** for local development:

> "On a developer system you would actually load up every single DLL so that you could have the entire stack of 18 servers running in one process... test them just right there and see how they work."

This eliminated the need to deploy a full cluster for development, dramatically accelerating iteration. The team averaged **20 builds per day over four years** -- roughly one build every five minutes during business hours. At peak, the rate was 30-40 builds per day.

---

## Movement: Server-Authoritative

Guild Wars used a **server-authoritative movement model** where the client performed local pathfinding and collision prediction, but the server held the true position of every entity.

> "In Guild Wars the server is authoritative for any important decisions, like where players are actually located... But clients do their best to model that in a way that seems seamless to users."

### How Movement Worked

The client had full access to the map's collision mesh and navigation data. When a player clicked to move, the client computed a local path and began animating movement immediately. Simultaneously, it sent the movement intent (destination or direction) to the server. The server independently computed the authoritative path and position.

> "I don't have to go ask the server 'is it okay if I walk over here?' I already know all the collision information that's local to my system... I can effectively perform pathfinding around those things."

Because the client and server shared the same collision data and pathfinding algorithm, they almost always agreed. When they diverged (e.g., due to another player body-blocking a path that the client did not yet know about), the server corrected the client:

> "The server telling me what is actually true. So it can provide modifications... then in my system I might have to teleport back a little bit. But by-and-large the rate of communication that you have with the server is high enough that you don't see those artifacts."

This "teleport back" is the **rubber-banding** that Guild Wars players occasionally experienced -- the direct consequence of server authority without dead reckoning.

### Pseudo-Code: Server-Authoritative Movement

```cpp
// ============================================================
// CLIENT SIDE: Movement Intent
// ============================================================

void Client::OnClickToMove(Vec2 destination) {
    // Client has full collision mesh -- compute path locally
    Path localPath = pathfinder.ComputePath(
        myPosition, destination, collisionMesh
    );

    // Begin animating immediately (local prediction)
    localPlayer.StartWalkingAlongPath(localPath);

    // Send intent to server -- NOT the computed path,
    // just the destination. Server will compute its own path.
    MoveRequest msg;
    msg.playerId    = myId;
    msg.destination = destination;
    msg.timestamp   = currentTime;
    tcpSocket.Send(msg);
}

void Client::OnServerPositionUpdate(PositionUpdate update) {
    Vec2 serverPos = update.position;
    Vec2 clientPos = localPlayer.GetPosition();

    float error = Distance(serverPos, clientPos);

    if (error < SNAP_THRESHOLD) {  // e.g., < 0.5 units
        // Minor divergence -- nudge smoothly
        localPlayer.SetPosition(Lerp(clientPos, serverPos, 0.3f));
    } else {
        // Major divergence -- rubber-band snap
        // This is the "teleport back a little bit" Wyatt describes
        localPlayer.SetPosition(serverPos);
        localPlayer.RecalculatePath(update.destination);
    }
}

// ============================================================
// SERVER SIDE: Authoritative Resolution
// ============================================================

void GameServer::OnMoveRequest(PlayerID id, MoveRequest msg) {
    Player& player = GetPlayer(id);

    // Server computes path using the SAME collision mesh
    // but with authoritative knowledge of all entity positions
    // (including body-blocking players the client may not know about)
    Path serverPath = pathfinder.ComputePath(
        player.position, msg.destination, collisionMesh,
        GetAllEntityBounds()  // includes other players for body-blocking
    );

    player.activePath = serverPath;
    player.isMoving   = true;
}

void GameServer::Tick(float dt) {
    for (Player& p : allPlayers) {
        if (!p.isMoving) continue;

        // Advance along server-computed path
        Vec2 newPos = p.activePath.AdvanceBy(p.moveSpeed * dt);

        // Check for collisions with other entities (body-blocking)
        if (IsBlockedByEntity(p, newPos)) {
            newPos = ResolveCollision(p, newPos);
            p.activePath = pathfinder.Reroute(
                newPos, p.activePath.destination, collisionMesh,
                GetAllEntityBounds()
            );
        }

        p.position = newPos;

        // Broadcast authoritative position to all clients in instance
        BroadcastPositionUpdate(p);
    }
}
```

### Key Insight: Shared Collision Data

The reason this system worked well despite having no client-side prediction in the traditional FPS sense is that **both client and server ran identical pathfinding on identical collision data**. The only source of divergence was other entities (players, NPCs) whose positions the client might not yet know about. For a single player walking through an empty corridor, client and server agreed perfectly. Divergence only occurred in crowded areas or when body-blocking happened -- exactly the situations where the server's authoritative answer was most important.

---

## Two-Stage Commit for Combat

The single most important latency-masking technique in Guild Wars was what can be called a **two-stage commit**: the client plays a long animation locally while the server resolves the action, with the result arriving before the animation completes.

> "If you want to pick up an item then what happens is you initiate the pick-up request and now you have an animation sequence that takes a while to get done so that the likelihood is that the round trip to the server has already elapsed."

This was a deliberate design choice. Skills in Guild Wars had **activation times** ranging from 0.25 seconds to 3+ seconds, plus a standard **aftercast delay** of 0.75 seconds. The total visual commitment for a typical 1-second skill was 1.75 seconds -- far longer than the ~100-300ms round-trip time for most players. The server's response arrived during the animation, so the player never perceived the network delay.

### How the Timing Worked

```
Timeline for a skill with 1-second activation time:

Player                          Network                    Server
  |                                                          |
  |-- Click skill ------------------------------------------>|
  |   (start cast animation locally)                         |
  |   [0ms]                                                  |
  |                                                          |
  |   ~~~ cast animation playing ~~~          [~50ms] Receive request
  |                                                          |
  |                                           [~55ms] Validate:
  |                                                    - In range?
  |                                                    - Has energy?
  |                                                    - Line of sight?
  |                                                    - Not interrupted?
  |                                                          |
  |                                           [~60ms] Resolve effect:
  |                                                    - Calculate damage
  |                                                    - Apply conditions
  |                                                    - Check for kills
  |                                                          |
  |   ~~~ cast animation still playing ~~~                   |
  |                                                          |
  |<--- Result packet ---------------------- [~60ms] Send result
  |                                                          |
  |   [~110ms] Receive result                                |
  |   (store pending -- don't apply yet)                     |
  |                                                          |
  |   ~~~ cast animation STILL playing ~~~                   |
  |                                                          |
  |   [1000ms] Cast animation ends                           |
  |   [1000ms] Apply stored result visually:                 |
  |            - show damage number                          |
  |            - play hit effect                             |
  |            - update health bars                          |
  |                                                          |
  |   [1000-1750ms] Aftercast delay (0.75s)                  |
  |   (player locked out of next action)                     |
```

The critical insight: the **animation was deliberately designed to be longer than the expected RTT**. Typical skill activation times:

| Activation Time | + Aftercast | Total Lock | Covers RTT Up To |
|-----------------|-------------|------------|-------------------|
| 0.25s (1/4 sec) | + 0.75s | 1.0s | ~900ms |
| 0.75s (3/4 sec) | + 0.75s | 1.5s | ~1400ms |
| 1.0s | + 0.75s | 1.75s | ~1650ms |
| 2.0s | + 0.75s | 2.75s | ~2650ms |
| 3.0s | + 0.75s | 3.75s | ~3650ms |

Even the fastest skills (0.25s activation) provided a full second of animation to hide round-trip latency -- sufficient for connections up to ~900ms RTT.

### Pseudo-Code: Two-Stage Combat Commit

```cpp
// ============================================================
// CLIENT SIDE: Skill Activation
// ============================================================

void Client::OnSkillActivated(SkillID skillId, EntityID target) {
    const SkillDef& skill = GetSkillDef(skillId);

    // STAGE 1: Immediately begin cast animation (optimistic)
    localPlayer.PlayCastAnimation(skill.castAnimId, skill.activationTime);
    localPlayer.LockMovement();  // cannot move during cast

    // Send activation request to server
    SkillRequest msg;
    msg.skillId   = skillId;
    msg.caster    = myId;
    msg.target    = target;
    msg.timestamp = currentTime;
    tcpSocket.Send(msg);

    // Enter "waiting for resolution" state
    pendingSkill = { skillId, target, currentTime };
}

void Client::OnSkillResult(SkillResult result) {
    if (result.outcome == SKILL_INTERRUPTED ||
        result.outcome == SKILL_OUT_OF_RANGE ||
        result.outcome == SKILL_FAILED) {
        // Server rejected -- cancel animation early
        localPlayer.InterruptCastAnimation();
        localPlayer.UnlockMovement();
        ShowFailureMessage(result.reason);
        return;
    }

    // STAGE 2: Server confirmed the skill landed
    // Store the result -- we'll apply it visually when
    // the cast animation finishes (which is almost certainly
    // AFTER this packet arrives, since animation > RTT)
    pendingResult = result;
}

void Client::OnCastAnimationComplete() {
    if (pendingResult.valid) {
        // Apply stored result now that animation is done
        ApplyDamageNumbers(pendingResult.damage);
        PlayHitEffects(pendingResult.target, pendingResult.effects);
        UpdateHealthBars(pendingResult.healthChanges);
    }

    // Begin aftercast delay (0.75s standard)
    localPlayer.BeginAftercast(0.75f);
}

// ============================================================
// SERVER SIDE: Skill Resolution
// ============================================================

void GameServer::OnSkillRequest(PlayerID casterId, SkillRequest msg) {
    Player& caster = GetPlayer(casterId);
    Entity& target = GetEntity(msg.target);
    const SkillDef& skill = GetSkillDef(msg.skillId);

    // Validate preconditions using server-authoritative state
    if (!caster.HasEnoughEnergy(skill.energyCost)) {
        SendSkillResult(casterId, SKILL_FAILED, "Not enough energy");
        return;
    }

    float distance = Distance(caster.position, target.position);
    if (distance > skill.range) {
        SendSkillResult(casterId, SKILL_OUT_OF_RANGE, "Out of range");
        return;
    }

    if (!HasLineOfSight(caster.position, target.position)) {
        SendSkillResult(casterId, SKILL_FAILED, "Obstructed");
        return;
    }

    // Register the pending cast -- will resolve after activation time
    PendingCast cast;
    cast.caster         = casterId;
    cast.target         = msg.target;
    cast.skill          = msg.skillId;
    cast.resolveTime    = currentTime + skill.activationTime;
    cast.interruptible  = true;
    pendingCasts.push_back(cast);

    // Deduct energy immediately
    caster.energy -= skill.energyCost;
}

void GameServer::Tick(float dt) {
    // Process pending casts
    for (auto it = pendingCasts.begin(); it != pendingCasts.end(); ) {
        PendingCast& cast = *it;

        if (cast.interrupted) {
            SendSkillResult(cast.caster, SKILL_INTERRUPTED, "Interrupted");
            it = pendingCasts.erase(it);
            continue;
        }

        if (currentTime >= cast.resolveTime) {
            // Activation time elapsed -- resolve the skill
            ResolveCombatSkill(cast);
            it = pendingCasts.erase(it);
        } else {
            ++it;
        }
    }
}

void GameServer::ResolveCombatSkill(const PendingCast& cast) {
    Player& caster = GetPlayer(cast.caster);
    Entity& target = GetEntity(cast.target);
    const SkillDef& skill = GetSkillDef(cast.skill);

    // Re-check range at resolution time (target may have moved)
    float distance = Distance(caster.position, target.position);

    SkillResult result;
    result.outcome = SKILL_SUCCESS;

    // Calculate damage with all modifiers
    int damage = skill.baseDamage + caster.GetAttributeBonus(skill.attribute);
    damage = ApplyArmorReduction(damage, target.armorLevel);
    damage = ApplyProtectionEffects(damage, target);

    target.health -= damage;
    result.damage  = damage;
    result.target  = cast.target;

    // Apply conditions (bleeding, burning, etc.)
    for (const Condition& cond : skill.conditions) {
        target.ApplyCondition(cond);
        result.effects.push_back(cond);
    }

    if (target.health <= 0) {
        // "If two people say that they killed the monster it doesn't
        //  matter because the state just goes to dead." - Pat Wyatt
        target.health = 0;
        target.state  = DEAD;
    }

    // Send result to caster (arrives during their cast animation)
    SendSkillResult(cast.caster, result);

    // Broadcast effect to all other clients in instance
    BroadcastSkillEffect(cast, result);
}
```

### Arrow Resolution: Fire-Time, Not Impact-Time

Ranged attacks resolved at fire-time, not when the projectile visually reached the target. Wyatt explained:

> "When you fire an arrow, you're actually over there because you've been running this way and so I don't know that yet because of network latency so I fire the arrow... On my system you're standing over there and I fire and it hits you. But really you're standing over there... At the point the arrow is fired we decide whether it is going to hit or not. Has nothing to do with motion."

This meant:
- **On the shooter's screen:** The arrow flies to where the target appears to be and hits.
- **On the target's screen:** The arrow may appear to curve slightly or hit from an unexpected angle.
- **On the server:** Hit/miss is decided at the moment of firing, using server-authoritative positions. The visual flight of the arrow is cosmetic.

---

## No Dead Reckoning (By Design)

Guild Wars deliberately chose **not** to use dead reckoning for remote player positions. This was not an oversight -- it was a core design decision driven by the game's positional combat mechanics.

### Why Dead Reckoning Would Break GW1

Guild Wars 1 had several mechanics that required **exact server-confirmed positions**:

1. **Body-blocking.** Players and creatures had collision volumes. Positioning your character in a doorway or narrow path physically blocked enemies from passing. This was a core PvP and PvE tactic -- teams would body-block fleeing enemies or protect healers by forming walls. If the client predicted another player's position incorrectly via dead reckoning, body-blocks would fail or succeed erroneously.

2. **AoE placement.** Area-of-effect spells like Fire Storm resolved at the server's known position of the target at cast completion. Dead reckoning errors would cause AoE effects to land in visually incorrect locations.

3. **Skill interrupts.** Interrupting an enemy's spell required precise timing and range checks. The server needed to know the exact position of both the interrupter and the target. Dead reckoning could place a player within interrupt range on one client but out of range on the server.

4. **Aggro ranges.** Enemy aggro was determined by distance thresholds checked on the server. Dead reckoning-induced position errors could cause monsters to aggro (or fail to aggro) in ways inconsistent with what the player saw.

### What Would Go Wrong With Dead Reckoning

```cpp
// ============================================================
// HYPOTHETICAL: What dead reckoning would break in GW1
// ============================================================

// SCENARIO: Player A tries to body-block Player B in a narrow corridor
// Player B is running east at 300 units/sec
// Last server update was 150ms ago

// --- WITH DEAD RECKONING (hypothetical -- GW1 did NOT do this) ---

void Client_DR::UpdateRemotePlayer(RemotePlayer& remote, float dt) {
    // Extrapolate position based on last known velocity
    remote.displayPos += remote.lastVelocity * dt;
    // ^^^ This is the problem. We're GUESSING where they are.
}

void Client_DR::CheckBodyBlock(Player& local, RemotePlayer& remote) {
    // Client thinks remote player is at extrapolated position
    float dist = Distance(local.position, remote.displayPos);

    if (dist < BODY_BLOCK_RADIUS) {
        // Client shows: "I'm blocking them!"
        // But server may disagree...
        //
        // If remote player turned 100ms ago and the client
        // hasn't received that update yet, the extrapolated
        // position is WRONG. The remote player already turned
        // and walked around the body-block on the server.
        //
        // Result: Client shows a successful body-block,
        // then 100ms later the remote player teleports
        // past the blocker. The entire tactic is unreliable.
        ShowBodyBlockEffect();
    }
}

// Interrupt example with dead reckoning:
void Client_DR::AttemptInterrupt(RemotePlayer& target) {
    // Client thinks target is at dead-reckoned position
    float range = Distance(localPlayer.position, target.displayPos);

    if (range <= INTERRUPT_RANGE) {
        // Client shows interrupt animation
        // But server checks REAL position:
        //   server distance = 1.2 * INTERRUPT_RANGE  (target was further away)
        //   -> Server: "Out of range, interrupt fails"
        //
        // Player sees: interrupt animation plays, then nothing happens.
        // Extremely frustrating for competitive PvP.
        SendInterruptRequest(target.id);
    }
}

// --- GW1's ACTUAL APPROACH (no dead reckoning) ---

void Client_GW1::UpdateRemotePlayer(RemotePlayer& remote, PositionUpdate update) {
    // Only update position when server tells us
    // Remote player may appear to move in short bursts
    // rather than perfectly smooth motion
    remote.position = update.serverPosition;

    // Optional: smooth interpolation between server updates
    // but NEVER extrapolate beyond what server has confirmed
    remote.displayPos = SmoothToward(remote.displayPos, remote.position, dt);
}

void Client_GW1::CheckBodyBlock(Player& local, RemotePlayer& remote) {
    // Position is server-confirmed -- if it looks like a body-block,
    // it IS a body-block on the server too.
    float dist = Distance(local.position, remote.position);

    if (dist < BODY_BLOCK_RADIUS) {
        // This is reliable because remote.position came from the server.
        // The server sees the same thing.
        ShowBodyBlockEffect();
    }
}
```

### The Accepted Trade-Off

The cost of not using dead reckoning was visible: **remote players moved in slightly jerky increments** rather than perfectly smooth motion. On high-latency connections, other players appeared to "slide" between server-confirmed positions. This was a known and accepted trade-off:

> "By-and-large the rate of communication that you have with the server is high enough that you don't see those artifacts."

For most players on reasonable connections (< 200ms), the update rate was sufficient for smooth-looking motion. The rare visible jitter was deemed far less harmful than the gameplay-breaking consequences of dead reckoning errors in a positional combat system.

**Comparison to other MMOs:**

| Game | Remote Player Display | Positional Combat? | Trade-Off |
|------|----------------------|-------------------|-----------|
| Guild Wars 1 | Server-confirmed only | Yes (body-blocking, interrupts, AoE) | Slight jitter on remote players |
| World of Warcraft | Dead reckoning (extrapolation) | No (target-locked, no body-block) | Smooth but positions can be wrong |
| FFXIV | Low-freq updates + smoothing | Minimal (snapshot AoE, no body-block) | Ghost hits on AoE at 3.3 Hz |
| Guild Wars 2 | Dead reckoning | Limited (AoE target cap) | Smooth but position desyncs in WvW |

---

## Zero-Downtime Rolling Upgrades

Guild Wars achieved what was extremely rare for online games at the time: continuous operation without maintenance windows.

> "We tried never to do maintenance, we ran 24/7/365. No maintenance periods. Which is extremely unusual."

Individual game services maintained **over two years of continuous uptime**. After Wyatt's departure from ArenaNet, uptime records reportedly tripled beyond that.

### How It Worked

The key enabler was the **DLL-based architecture combined with versioned inter-server RPC**.

Every server type was a DLL loaded by the `ArenaSrv` host executable. The shared `SrvSocket` library embedded protocol version information in every message. When a server was upgraded to a new version, it could communicate with both old-version and new-version peers:

> "New protocol and old protocol had to continue to work... you've got a new server running, it can speak the new protocol and it can do additional functionality."

The upgrade process for a single server type:

1. Stop routing new players/requests to the target server instance.
2. Wait for existing work to drain ("okay this server is going to go out of service and so everybody get off it!").
3. Kill the old process.
4. Start the new version (new DLL loaded by `ArenaSrv`).
5. Resume routing.

Because each server type was independent and multiple instances of each type existed, upgrading one instance at a time had zero impact on players.

### Pseudo-Code: Version Negotiation

```cpp
// ============================================================
// VERSIONED RPC PROTOCOL
// ============================================================

// Every message carries a version header
struct MessageHeader {
    uint16_t protocolVersion;  // e.g., 42
    uint16_t messageType;      // e.g., MSG_SKILL_ACTIVATE
    uint32_t payloadSize;
};

// The SrvSocket library handles version compatibility
class SrvSocket {
    uint16_t localVersion;   // this server's protocol version
    uint16_t remoteVersion;  // negotiated peer version

    void OnConnect(Connection& conn) {
        // First message exchanged is a version handshake
        VersionHandshake hello;
        hello.version      = localVersion;
        hello.minSupported = localVersion - 2;  // support 2 versions back
        conn.Send(hello);
    }

    void OnVersionHandshake(Connection& conn, VersionHandshake peer) {
        // Negotiate to the LOWER of the two versions
        // so the newer server speaks the older protocol
        remoteVersion = Min(localVersion, peer.version);

        if (remoteVersion < peer.minSupported ||
            remoteVersion < (localVersion - 2)) {
            // Too far apart -- cannot communicate
            conn.Reject("Protocol version mismatch");
            return;
        }

        conn.negotiatedVersion = remoteVersion;
        conn.state = CONNECTED;
    }

    void SendMessage(Connection& conn, uint16_t type, const void* data) {
        MessageHeader header;
        header.protocolVersion = conn.negotiatedVersion;
        header.messageType     = type;

        // Serialize based on the negotiated version
        // Newer fields are simply omitted when speaking old protocol
        Buffer buf = Serialize(type, data, conn.negotiatedVersion);
        header.payloadSize = buf.size();

        conn.Send(header);
        conn.Send(buf);
    }

    void OnMessage(Connection& conn, MessageHeader header, Buffer data) {
        // Deserialize based on the version in the header
        // Missing fields (from older senders) get defaults
        auto msg = Deserialize(
            header.messageType, data, header.protocolVersion
        );

        // If the message has new fields we don't understand,
        // they were already stripped during negotiation
        DispatchToHandler(header.messageType, msg);
    }
};

// ============================================================
// ROLLING UPGRADE ORCHESTRATOR
// ============================================================

void UpgradeController::RollingUpgrade(ServerType type, DllVersion newVer) {
    auto instances = GetAllInstances(type);

    for (ServerInstance& inst : instances) {
        // Step 1: Drain -- stop sending new work
        inst.SetAcceptingWork(false);
        Log("Draining %s instance %d", type, inst.id);

        // Step 2: Wait for current work to complete
        while (inst.GetActiveConnections() > 0) {
            Wait(1000);  // check every second
            // Connected clients are migrated by the matchmaker
            // to other instances of the same type
        }

        // Step 3: Shutdown old version
        inst.Shutdown();
        Log("Stopped %s instance %d (version %d)", type, inst.id, inst.version);

        // Step 4: Start new version
        inst.LoadDll(newVer);
        inst.Start();
        Log("Started %s instance %d (version %d)", type, inst.id, newVer);

        // Step 5: Resume accepting work
        inst.SetAcceptingWork(true);

        // Step 6: Verify health before proceeding to next instance
        if (!inst.HealthCheck()) {
            Log("ALERT: New version unhealthy, halting rollout");
            return;  // stop rolling upgrade on failure
        }
    }

    Log("Rolling upgrade of %s to version %d complete", type, newVer);
}
```

### Rapid Patch Deployment

The build and deploy pipeline was exceptionally fast:

> "One of the great advantages of the Guild Wars development environment we created was the ability to checkin code, then build and deploy it to millions of end-users across the globe with a single command, all in the span of a couple of minutes."

When a critical bug was discovered in production (e.g., via the crash reporting system with stack dumps), the team could have a fix deployed in as little as **five minutes**.

---

## Instancing as a Networking Solution

Guild Wars solved the hardest problem in MMO networking -- scaling to thousands of simultaneous players -- by **not having thousands of players in the same simulation**. Every explorable area in Guild Wars was instanced.

### Instance Types and Sizes

| Zone Type | Max Players | Typical Party Size | Description |
|-----------|-------------|-------------------|-------------|
| Towns / Outposts | ~100 | N/A (social hubs) | Shared social spaces, created as "districts" when population exceeds 100 |
| Explorable Areas | 8 | 4-8 | Private PvE instances for one party |
| Missions | 8 | 4-8 | Story-driven PvE instances |
| PvP Arenas | 16 | 4v4 or 8v8 | Competitive matches |
| Alliance Battles | ~48 | 12v12v12 (approx.) | Larger PvP |
| Festival Events | ~150 | N/A | Special seasonal gatherings |

> "You'd join a particular game and you play there for a brief while, and then you join another game... you and I get in our own private copy of a two, four or eight player world."

### Why Instancing Works for Networking

Traditional MMOs like World of Warcraft face a fundamental scaling challenge: when 500 players gather in one area, the server must broadcast state updates for all 500 to all 500, creating O(n^2) message complexity. WoW mitigates this with culling, interest management, and reduced update rates, but large gatherings still cause server lag.

Guild Wars avoided this entirely. With a maximum of ~100 players in social areas and ~8 in gameplay areas, the networking problem was bounded:

- **8-player PvE instance:** 8 players * ~50 updates/sec = ~400 messages/sec total -- trivial for any server.
- **100-player town:** 100 * ~10 updates/sec (lower rate for social zones) = ~1000 messages/sec -- still well within a single server's capacity.

When a town reached capacity, the game created a new **district** -- an identical parallel instance:

> "If 3000 players want to join Ascalon City, which can hold 100 players per town, then the game will create (3000/100=) 30 town instances."

### Pseudo-Code: Instance Creation and Player Routing

```cpp
// ============================================================
// INSTANCE CREATION AND ROUTING
// ============================================================

// SiteSrv (Matchmaking Server): Routes players to game instances

struct GameInstance {
    InstanceID  id;
    MapID       mapId;
    ServerID    hostServer;   // which GameSrv hosts this
    int         playerCount;
    int         maxPlayers;
    bool        isPrivate;    // party instance vs. public district
    uint32_t    creationTime;
};

class SiteSrv {
    Map<MapID, Vector<GameInstance>> publicDistricts;
    Map<InstanceID, GameInstance>    allInstances;

    // Called when a party wants to enter an explorable area
    InstanceID AssignPartyToInstance(Party& party, MapID map) {
        if (IsExplorable(map) || IsMission(map) || IsPvPArena(map)) {
            // Private instance -- always create new
            return CreatePrivateInstance(map, party);
        } else {
            // Town/outpost -- find or create a public district
            return FindOrCreateDistrict(map);
        }
    }

    InstanceID CreatePrivateInstance(MapID map, Party& party) {
        // Find the GameSrv with the most available capacity
        ServerID server = FindLeastLoadedGameServer();

        // Request instance creation
        CreateInstanceRequest req;
        req.mapId      = map;
        req.maxPlayers = GetMaxPlayersForMap(map);  // 8 for PvE, 16 for PvP
        req.isPrivate  = true;

        InstanceID id = SendToGameServer(server, req);

        GameInstance inst;
        inst.id          = id;
        inst.mapId       = map;
        inst.hostServer  = server;
        inst.playerCount = 0;
        inst.maxPlayers  = req.maxPlayers;
        inst.isPrivate   = true;
        allInstances[id] = inst;

        return id;
    }

    InstanceID FindOrCreateDistrict(MapID map) {
        auto& districts = publicDistricts[map];

        // Find a district with room
        for (auto& dist : districts) {
            if (dist.playerCount < dist.maxPlayers - 5) {  // leave buffer
                return dist.id;
            }
        }

        // All districts full -- create new one
        ServerID server = FindLeastLoadedGameServer();

        CreateInstanceRequest req;
        req.mapId      = map;
        req.maxPlayers = 100;  // town capacity
        req.isPrivate  = false;

        InstanceID id = SendToGameServer(server, req);

        GameInstance inst;
        inst.id          = id;
        inst.mapId       = map;
        inst.hostServer  = server;
        inst.playerCount = 0;
        inst.maxPlayers  = 100;
        inst.isPrivate   = false;

        districts.push_back(inst);
        allInstances[id] = inst;

        return id;
    }

    // Called by LobbySrv when player is ready to enter game
    JoinToken RoutePlayerToInstance(PlayerID player, InstanceID instance) {
        GameInstance& inst = allInstances[instance];

        // Generate a one-time-use token for the client
        JoinToken token;
        token.instanceId = instance;
        token.serverId   = inst.hostServer;
        token.serverAddr = GetServerAddress(inst.hostServer);
        token.expiry     = currentTime + 30;  // 30 second TTL
        token.secret     = GenerateSecret();

        // Notify the GameSrv to expect this player
        ExpectPlayer expectMsg;
        expectMsg.playerId = player;
        expectMsg.token    = token.secret;
        SendToGameServer(inst.hostServer, expectMsg);

        return token;
    }
};

// ============================================================
// GameSrv: Hosts actual gameplay instances
// Each instance runs single-threaded for simplicity
// ============================================================

class GameSrv {
    Map<InstanceID, GameInstance*> hostedInstances;

    void OnCreateInstance(CreateInstanceRequest req) {
        // Each game instance runs single-threaded
        // "each game instance ran single-threaded" to simplify
        // content programming - Pat Wyatt
        GameInstance* inst = new GameInstance();
        inst->LoadMap(req.mapId);
        inst->maxPlayers = req.maxPlayers;

        // Async network library handles message dispatch
        // "a low-level, asynchronous library handled receiving
        //  network messages from players and other backend services
        //  and posting those messages up into the synchronous,
        //  single-threaded game logic"
        inst->StartTickLoop();

        hostedInstances[inst->id] = inst;
    }
};
```

### Comparison to Open-World MMOs

| Challenge | Guild Wars 1 (Instanced) | WoW (Open World) | GW2 (Hybrid) |
|-----------|--------------------------|-------------------|---------------|
| 500+ player gatherings | Impossible by design (districts cap at ~100) | Server-wide lag, reduced update rate | Overflow maps; AoE target cap of 5 |
| Interest management | Not needed (few players per instance) | Complex culling by distance | Priority-based culling |
| Zone transitions | Seamless (loading screen to new instance) | Seamless (cross-zone streaming) | Seamless within maps; loading between |
| Server cost per player | Low (instances are cheap, short-lived) | High (persistent world state) | Medium |
| Social presence | Limited (only see ~100 in towns) | Unlimited (world feels alive) | Mega-server merging |

---

## Key Sources

### Primary Sources (Pat Wyatt / ArenaNet)

- **[HandmadeCon 2015 Interview Transcript](https://www.codeofhonor.com/blog/handmadecon-2015-interview-transcript)** -- Pat Wyatt interviewed by Casey Muratori. The single most comprehensive source on GW1 netcode: covers TCP choice, 18 server types, DLL architecture, movement authority, combat resolution, rolling upgrades, file distribution, and development methodology. Transcribed on Code of Honor.

- **[Twenty Years of Guild Wars](https://www.codeofhonor.com/blog/twenty-years-of-guild-wars)** -- Pat Wyatt's retrospective on GW1's 20th anniversary. Covers uptime records, the C++ codebase, and the no-subscription business model's impact on server architecture.

- **[Scaling Guild Wars for Massive Concurrency](https://www.codeofhonor.com/blog/scaling-guild-wars-for-massive-concurrency)** -- Load testing methodology, 600+ skills system, AMD Opteron server selection, game recording and playback for stress testing.

- **[Debugging Running Server Applications](https://www.codeofhonor.com/blog/debugging-running-server-applications)** -- Instance-based architecture details, single-threaded instance design, async network library, stack-walk crash reporting, rapid deployment pipeline.

- **[Writing Server and Network Code for Your Online Game (GDC 2012)](https://www.codeofhonor.com/blog/writing-server-and-network-code-for-your-online-game)** -- GDC presentation on server architecture patterns drawn from GW1 experience.

- **[Code of Honor Blog -- Guild Wars Tag](https://www.codeofhonor.com/blog/author/patrick-wyatt)** -- Pat Wyatt's full blog with additional posts on networking, security, and game development.

### Secondary Sources

- **['Guild Wars' Microservices and 24/7 Uptime (GDC 2017)](https://www.gdcvault.com/play/1024018/-Guild-Wars-Microservices-and)** -- Stephen Clarke-Willson's talk on ArenaNet's microservices architecture and zero-downtime operations. Covers both GW1 and GW2.

- **[Guild Wars Wiki: Rubber-banding](https://wiki.guildwars.com/wiki/Rubber-banding)** -- Player-documented evidence of server-authoritative movement. Confirms desync correction behavior.

- **[Guild Wars Wiki: Activation Time](https://wiki.guildwars.com/wiki/Activation_time)** -- Detailed skill timing data: activation times from 0.25s to 3s+, server-side queue processing.

- **[Guild Wars Wiki: Aftercast Delay](https://wiki.guildwars.com/wiki/Aftercast_delay)** -- Standard 0.75s aftercast delay documentation.

- **[Guild Wars Wiki: Body Block](https://wiki.guildwars.com/wiki/Body_block)** -- Positional combat mechanics including collision-based blocking tactics.

- **[Guild Wars Wiki: Interrupt](https://wiki.guildwars.com/wiki/Interrupt)** -- Skill interrupt mechanics, timing windows, and server-side activation handling.

- **[How ArenaNet Moved Guild Wars to the Cloud (AWS Blog)](https://aws.amazon.com/blogs/gametech/arenanet-guild-wars-mmorpg-migration/)** -- ArenaNet's migration from bare-metal to AWS EC2, preserving the original microservices architecture.
