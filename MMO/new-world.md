# New World -- Netcode and Server Architecture Deep Dive

## 1. Overview

**New World** is a massively multiplayer online game developed by **Amazon Games**
(formerly Amazon Game Studios) and released in September 2021. Set on the
fictional island of Aeternum, it supports up to **2,500 concurrent players per
world** alongside **7,000+ AI entities** and hundreds of thousands of interactive
objects -- all simulated at **30 ticks per second** on a fully
**server-authoritative** architecture.

The game is built on **Amazon Lumberyard** (now **Open 3D Engine / O3DE**), which
ships with the **GridMate** networking framework. GridMate provides a
replica-based replication system using **DataSets** for state synchronization and
**RPCs** for client-to-server requests -- both designed around server authority
from the ground up.

New World is notable in the netcode world for three reasons:

1. **Cloud-native MMO architecture** -- purpose-built for AWS with a hub-based
   spatial partitioning scheme spread across multiple EC2 instances per world.
2. **The "client authoritative" controversy** -- a widely misunderstood
   invulnerability bug that became a case study in how server-blocking code
   paths can mimic client authority without actually being client authoritative.
3. **A brutal launch gauntlet** -- gold duplication via race conditions, HTML
   injection through unsanitized chat, and persistent desync issues that exposed
   the realities of shipping a large-scale MMO on new infrastructure.

---

## 2. Architecture

### 2.1 GridMate Networking Foundation

Lumberyard's **GridMate** is a replica-based networking framework where all
networked state lives in **ReplicaChunks**. Each chunk exists as either a
**Master** (authoritative, lives on the server) or a **Proxy** (read-only mirror,
lives on clients). When a master chunk is created on the server, GridMate
automatically creates proxy chunks on every connected client and keeps them
synchronized.

The two primary primitives:

- **DataSets** -- Typed state containers that automatically replicate from master
  to proxy when their value changes. They guarantee **eventual consistency**.
  The server writes; clients observe.
- **RPCs (Remote Procedure Calls)** -- Messages sent from proxy to master to
  request state changes. The server validates and either accepts or rejects the
  request. The result flows back through DataSet updates.

```cpp
// -----------------------------------------------------------------------
// GridMate ReplicaChunk: DataSet + RPC replication model
// -----------------------------------------------------------------------
// In Lumberyard/O3DE, all networked state is organized into ReplicaChunks.
// The server owns "Master" chunks; clients receive "Proxy" copies.

class PlayerStateChunk : public GridMate::ReplicaChunk
{
public:
    GM_CLASS_ALLOCATOR(PlayerStateChunk);

    // Unique chunk type name for the replication registry
    static const char* GetChunkName() { return "PlayerStateChunk"; }

    PlayerStateChunk()
        : m_position("Position")        // DataSet: server-authoritative position
        , m_health("Health")            // DataSet: server-authoritative health
        , m_currentAnim("Animation")    // DataSet: current animation state
        , RequestMoveRpc("RequestMove") // RPC: client requests movement
    {
    }

    // ---- DataSets (Server -> All Clients) ----
    // These are write-once-on-master, read-only-on-proxy.
    // GridMate automatically replicates changes to all proxies.
    GridMate::DataSet<AZ::Vector3>    m_position;
    GridMate::DataSet<float>          m_health;
    GridMate::DataSet<AZ::u32>        m_currentAnim;

    // ---- RPCs (Client -> Server) ----
    // Clients call RPCs to request state changes.
    // The server validates and applies (or rejects) the request.
    GridMate::Rpc<GridMate::RpcArg<AZ::Vector3>>  RequestMoveRpc;

    // Called on the SERVER when a client sends RequestMoveRpc
    bool OnRequestMove(const AZ::Vector3& desiredVelocity,
                       const GridMate::RpcContext& ctx)
    {
        // Server-side validation: check speed limits, collision, cooldowns
        if (!ValidateMovement(desiredVelocity, ctx.m_sourcePeer))
            return false;  // Reject: input exceeds allowed parameters

        // Server runs the full physics simulation
        AZ::Vector3 newPos = RunPhysicsStep(m_position.Get(), desiredVelocity);

        // Authoritative update -- DataSet automatically replicates to all proxies
        m_position.Set(newPos);
        return true;
    }
};
```

### 2.2 World Topology: REPs and Hubs

Each New World **world** (a single named server like "Valhalla" or "Olympus") is
composed of **11 EC2 instances** working together:

| Component | Count | Instance Type | Role |
|-----------|-------|---------------|------|
| **REP** (Remote Entry Point) | 4 | EC2 C5.2xlarge | Public-facing entry points. Only instances with public IPs. Handle connection routing, resilience, and security. Function as application-layer routers. |
| **Hub** | 7 | EC2 C5.9xlarge | Game simulation. Each hub processes a portion of the world grid. No public IPs -- only reachable through REPs. |

**REPs** act as the gateway layer. When a player connects, they reach one of the
4 REPs, which routes their traffic to whichever hub currently owns the grid cell
the player occupies. REPs handle reconnection resilience and serve as a security
boundary -- hubs never expose public endpoints.

**Hubs** are the simulation workhorses. The game world is overlaid with a
**spatial grid**, and each hub owns **two non-contiguous grid sections**. The
non-contiguous assignment is a deliberate load-balancing strategy: if a popular
area (like a town or world boss) creates a hotspot in one grid cell, the other
cell owned by that hub is likely in a different, quieter region -- spreading load
more evenly across the 7 hubs.

```
// -----------------------------------------------------------------------
// Hub-based zone grid layout (conceptual)
// -----------------------------------------------------------------------
// The world map of Aeternum is divided into a grid. Each cell is assigned
// to one of 7 hubs. Hubs own TWO non-contiguous cells to spread load.
//
//   +-------+-------+-------+-------+
//   | Hub 1 | Hub 2 | Hub 3 | Hub 4 |    Row 0 (northern regions)
//   +-------+-------+-------+-------+
//   | Hub 5 | Hub 6 | Hub 7 | Hub 1 |    Row 1 (central regions)
//   +-------+-------+-------+-------+
//   | Hub 2 | Hub 3 | Hub 4 | Hub 5 |    Row 2 (southern regions)
//   +-------+-------+-------+-------+
//   | Hub 6 | Hub 7 |               |    Row 3 (overflow / instanced)
//   +-------+-------+               +
//
// Notice: Hub 1 owns cells at (0,0) and (1,3) -- non-contiguous.
// If a zerg rush hits Windsward (0,0), the other cell (1,3) in a
// quieter region keeps Hub 1's total load balanced.
```

### 2.3 Player Handoff Between Hubs

When a player crosses a grid cell boundary, their simulation must be handed off
from one hub to another. This happens seamlessly -- no loading screens, no
disconnection. The handoff is orchestrated by the REP layer:

```cpp
// -----------------------------------------------------------------------
// Zone boundary handoff between EC2 hub instances
// -----------------------------------------------------------------------
// When a player crosses from one grid cell to another, the REP layer
// orchestrates the handoff between the source and destination hubs.

struct ZoneHandoffManager
{
    // Called on the REP when a hub reports a player near a cell boundary
    void BeginHandoff(PlayerId player, HubId sourceHub, HubId destHub)
    {
        // 1. Notify the destination hub to prepare for this player.
        //    The dest hub allocates a proxy entity and begins receiving
        //    state snapshots from the source hub.
        SendToHub(destHub, PreparePlayerMsg{
            .playerId   = player,
            .snapshot   = GetPlayerSnapshot(sourceHub, player),
            .gridCell   = GetDestinationCell(player)
        });

        // 2. Enter the "overlap" phase. Both hubs briefly simulate
        //    the player. The source hub remains authoritative during
        //    this window while the dest hub warms up.
        SetHandoffState(player, HandoffState::Overlap);

        // 3. Once the dest hub confirms it has a warm entity with
        //    current state, transfer authority atomically.
        //    The REP re-routes all future client packets to the dest hub.
        OnHubReady(destHub, [this, player, sourceHub, destHub]()
        {
            TransferAuthority(player, sourceHub, destHub);
            RerouteClientTraffic(player, destHub);

            // 4. Source hub tears down its copy of the player entity.
            SendToHub(sourceHub, ReleasePlayerMsg{ .playerId = player });
            SetHandoffState(player, HandoffState::Complete);
        });
    }
};
```

Key properties of this design:

- **Stateless hubs** -- Hubs do not store persistent player data. All durable
  state is written to DynamoDB. If a hub crashes, a replacement EC2 instance
  can spin up, load state from DynamoDB, and resume simulation.
- **Seamless transitions** -- The overlap phase ensures the player never
  experiences a gap. Both hubs briefly simulate the player, then authority
  transfers atomically.
- **REP as orchestrator** -- The REP layer acts as the control plane for
  handoffs, while hubs are the data plane.

---

## 3. The "Client Authoritative" Controversy

### 3.1 What the Community Believed

In October 2021, shortly after launch, players discovered that they could become
**invincible in PvP** by a comically simple method: switch to windowed mode, grab
the title bar, and **drag the window around the screen**. While the window was
being repositioned, the player's character entered a state of suspended animation
and could not take damage. Players could stand on War capture points indefinitely,
winning territory battles without risk.

The gaming press and community immediately concluded that New World was **"client
authoritative"** -- that the server trusted the client for damage calculations
and the window drag was preventing the client from reporting "I took damage."

### 3.2 What Actually Happened

Amazon community manager **Luxendra** posted a detailed explanation on the
official forums. The key quotes:

> "New World is not client authoritative. Clients dispatch controller inputs to
> the server, and the server then checks that input for limits that might
> invalidate it, then if accepted uses it as an input to a character within
> server memory. Physics and game rules are then run (entirely server side), and
> the outcome is sent back to the original client."

> "We do full physics detail for all such actions. Upon receiving the outcome,
> either hit or miss, the client will adjust its visual display to match what
> the server has determined."

The bug was a confluence of **two separate issues**:

1. **A server-side code path that blocked waiting for client input.** In certain
   weapon ability sequences, the server's state machine would enter a state
   where it was waiting for the client to send a "next input" message before
   advancing the simulation for that player. If the client stopped sending
   (because the window was being dragged and the game loop was frozen), the
   server stalled on that player's state -- it neither applied damage nor
   advanced the character.

2. **An intentional brief invulnerability weapon effect.** Certain weapon
   abilities (such as the Hatchet's **Berserk** tree, and dodge-roll i-frames)
   granted a short window of invulnerability. This is a normal action-combat
   mechanic -- invulnerability frames during ability activation.

The exploit combined these two issues: a player could activate a weapon ability
that granted brief invulnerability, then immediately freeze the client (by
dragging the window). The server was waiting for the client's next input to
advance past the invulnerability phase, and since the client was frozen, the
server never received that input. The player remained stuck in the invulnerable
state indefinitely.

### 3.3 Pseudo-Code: The Server Blocking Bug

```cpp
// -----------------------------------------------------------------------
// The actual bug: server code path blocked on client input
// -----------------------------------------------------------------------
// This is NOT "client authoritative." The server still owned all damage
// calculations. The problem was a BLOCKING WAIT in the server's ability
// state machine that prevented the server from advancing past the
// invulnerability phase without client input.

class AbilityStateMachine
{
    enum class Phase { WindUp, Active, Invulnerable, Recovery, Done };
    Phase m_phase = Phase::WindUp;

    void ServerTick(PlayerEntity& player, float dt)
    {
        switch (m_phase)
        {
        case Phase::WindUp:
            // Server plays the wind-up animation on the server skeleton
            if (AdvanceAnimation(player, dt))
                m_phase = Phase::Active;
            break;

        case Phase::Active:
            // Ability hits are calculated server-side using full physics
            PerformHitDetection(player);
            m_phase = Phase::Invulnerable;
            // Grant brief invulnerability (intentional game mechanic)
            player.SetInvulnerable(true);
            break;

        case Phase::Invulnerable:
            // >>> THE BUG <<<
            // The server WAITS for the client to send the next input
            // (e.g., dodge direction, follow-up attack, or cancel)
            // before transitioning out of the invulnerable phase.
            //
            // If the client stops sending inputs (window drag, alt-tab,
            // network stall), the server NEVER exits this state.
            // The player remains invulnerable indefinitely.
            if (player.HasPendingClientInput())
            {
                player.SetInvulnerable(false);
                m_phase = Phase::Recovery;
            }
            // else: keep waiting... player stays invulnerable
            //
            // THE FIX: Add a server-side timeout.
            // if (m_invulnerabilityTimer > MAX_INVULN_DURATION)
            // {
            //     player.SetInvulnerable(false);
            //     m_phase = Phase::Recovery;
            // }
            break;

        case Phase::Recovery:
            if (AdvanceAnimation(player, dt))
                m_phase = Phase::Done;
            break;
        }
    }
};
```

The fix was straightforward: Amazon added **server-side timeouts** to all code
paths that waited on client input. If the client does not respond within a
bounded time window, the server forcibly advances the state machine regardless.

This was **not** client authority. The server never delegated damage decisions to
the client. It was a server-side blocking wait that happened to preserve a
favorable state for an unresponsive client. The distinction matters: client
authority means the server **trusts** client-reported outcomes. This bug meant
the server **stalled** waiting for client input to proceed.

---

## 4. Movement and Combat

### 4.1 Server-Authoritative Movement with Client Prediction

New World uses standard server-authoritative movement with client-side prediction.
From Luxendra's forum post:

> "Clients dispatch controller inputs to the server... Physics and game rules
> are then run (entirely server side), and the outcome is sent back to the
> original client. There are some client side tricks we use here to 'stretch'
> the animation while the client is waiting for the server answer, but the
> outcome is always based only on the server answer."

The client predicts movement locally for responsiveness, but the server is the
final arbiter. The "animation stretching" Luxendra describes is a form of
**client prediction smoothing** -- the client plays movement animations
optimistically while waiting for server confirmation, then blends toward the
authoritative result when it arrives.

```cpp
// -----------------------------------------------------------------------
// Movement prediction with server validation
// -----------------------------------------------------------------------
// Client-side: predict movement locally for responsiveness.
// Server-side: validate and simulate authoritatively.
// Client reconciles when the server result arrives.

// === CLIENT SIDE ===
class ClientMovementPredictor
{
    struct PendingInput
    {
        uint32_t    sequenceNum;
        AZ::Vector3 inputVelocity;
        float       deltaTime;
    };

    AZ::Vector3              m_predictedPosition;
    std::deque<PendingInput> m_pendingInputs;  // Inputs sent but not yet ack'd
    uint32_t                 m_nextSequence = 0;

    void OnLocalInput(const AZ::Vector3& moveDir, float dt)
    {
        // 1. Record and send input to server
        PendingInput input{ m_nextSequence++, moveDir, dt };
        m_pendingInputs.push_back(input);
        SendToServer(MoveInputMsg{ input.sequenceNum, moveDir, dt });

        // 2. Predict locally -- apply movement immediately for responsiveness
        m_predictedPosition += RunLocalPhysics(m_predictedPosition, moveDir, dt);

        // 3. Play movement animation ("stretching" while awaiting server answer)
        PlayMoveAnimation(moveDir);
    }

    void OnServerStateReceived(uint32_t ackedSequence,
                               const AZ::Vector3& serverPosition)
    {
        // 4. Discard all inputs the server has already processed
        while (!m_pendingInputs.empty()
               && m_pendingInputs.front().sequenceNum <= ackedSequence)
        {
            m_pendingInputs.pop_front();
        }

        // 5. Re-predict from server-authoritative position
        //    using only the un-acknowledged inputs
        AZ::Vector3 reconciled = serverPosition;
        for (const auto& pending : m_pendingInputs)
        {
            reconciled += RunLocalPhysics(reconciled,
                                          pending.inputVelocity,
                                          pending.deltaTime);
        }

        // 6. Blend toward reconciled position (avoid rubber-banding snap)
        float error = (m_predictedPosition - reconciled).GetLength();
        if (error > SNAP_THRESHOLD)
            m_predictedPosition = reconciled;  // Too far off -- hard snap
        else
            m_predictedPosition = Lerp(m_predictedPosition, reconciled, 0.3f);
    }
};

// === SERVER SIDE ===
class ServerMovementValidator
{
    void OnClientMoveInput(PlayerId player, uint32_t seq,
                           const AZ::Vector3& inputVel, float dt)
    {
        PlayerEntity& entity = GetEntity(player);

        // Validate: speed within allowed range? Cooldowns respected?
        AZ::Vector3 clampedVel = ClampToMaxSpeed(inputVel, entity.GetMaxSpeed());

        // Full server-side physics simulation
        AZ::Vector3 newPos = RunServerPhysics(entity.GetPosition(), clampedVel, dt);

        // Check collision, terrain, anti-cheat boundaries
        newPos = ResolveCollisions(newPos, entity);

        // Commit authoritative position
        entity.SetPosition(newPos);

        // Send authoritative result back to client
        SendToClient(player, MoveResultMsg{ seq, newPos });
    }
};
```

### 4.2 Action Combat Networking

New World's combat is real-time action combat -- not tab-target. Melee hits are
resolved through **server-side hitbox collision** using a full character skeleton
simulated on the server. Ranged attacks use server-side projectile simulation.

From the developer post:

> "It's important to note that only after the server has performed the animation,
> and that results in the axe intersecting a tree, is this considered a success."

This means:

1. Client sends an attack input via RPC
2. Server plays the full attack animation on a server-side skeleton
3. Server runs physics intersection tests against the server-side positions of
   other players/entities
4. Hit or miss result is determined entirely server-side
5. Client receives the outcome and adjusts its visual display to match

The server simulates at **30 ticks per second** -- substantially higher than the
5 ticks per second typical of traditional MMOs. This higher tick rate is critical
for the action combat system, where hit windows can be as short as a few hundred
milliseconds.

### 4.3 Desync Issues

Despite the server-authoritative model, New World suffered significant **desync
problems** throughout 2021-2022. Players reported:

- Melee hits registering on the client but not on the server (or vice versa)
- Projectiles visually passing through targets without dealing damage
- Enemies teleporting or snapping to new positions mid-animation
- Skills "phasing through" opponents with delayed or absent damage registration

These issues were exacerbated during large-scale PvP (50v50 Wars) where server
load peaked. The root causes were a combination of:

- **Server overload** -- 30-tick simulation across 2,500 players and 7,000 AI
  entities is computationally intensive. Under War conditions, tick rate could
  degrade.
- **Animation-dependent hit detection** -- Because hits are resolved through
  server-side animation, any desynchronization between client and server
  animation timelines creates visible discrepancies.
- **Prediction mismatch** -- The client predicts player positions locally, but
  the server resolves hits against server-side positions. Under latency, the
  client may show a hit that the server does not confirm.

---

## 5. AWS Infrastructure

### 5.1 Compute Layer

New World is a showcase for AWS gaming infrastructure. Each world runs on:

- **4x EC2 C5.2xlarge** instances as REPs (Remote Entry Points)
- **7x EC2 C5.9xlarge** instances as Hubs (game simulation)

The C5 instance family is compute-optimized, powered by Intel Xeon Scalable
processors. The C5.9xlarge provides 36 vCPUs and 72 GiB of memory -- sized for
simulating thousands of entities at 30 Hz.

### 5.2 Persistence with DynamoDB

All persistent game state -- player inventories, character progression, company
treasuries, territory ownership, housing -- is stored in **Amazon DynamoDB**.
The write throughput is staggering: approximately **800,000 writes every 30
seconds** (roughly 26,700 writes per second sustained).

DynamoDB's key properties that make it suitable for this workload:

- **Single-digit millisecond latency** at any scale
- **On-demand capacity** that auto-scales with load
- **Strongly consistent reads** when needed (e.g., trade transactions)
- **Conditional writes** for atomicity (critical for preventing race conditions
  -- though as we will see, this was not always correctly implemented)

```cpp
// -----------------------------------------------------------------------
// DynamoDB persistence pattern for New World
// -----------------------------------------------------------------------
// Hubs are stateless -- all durable player state is periodically flushed
// to DynamoDB. ~800K writes per 30 seconds across the entire world.

class DynamoPersistenceManager
{
    // Periodic flush: every hub writes its dirty player states to DynamoDB
    // on a regular interval. Writes are batched for efficiency.

    void FlushDirtyEntities(const std::vector<PlayerEntity*>& dirtyPlayers)
    {
        Aws::DynamoDB::Model::BatchWriteItemRequest batchRequest;
        std::vector<Aws::DynamoDB::Model::WriteRequest> writeRequests;

        for (const auto* player : dirtyPlayers)
        {
            Aws::DynamoDB::Model::PutRequest putReq;

            // Primary key: player's unique account-bound ID
            putReq.AddItem("PlayerId",
                Aws::DynamoDB::Model::AttributeValue().SetS(
                    player->GetPlayerId()));

            // Partition key includes the world for cross-world queries
            putReq.AddItem("WorldId",
                Aws::DynamoDB::Model::AttributeValue().SetS(
                    player->GetWorldId()));

            // Serialize full player state: position, inventory, skills, etc.
            putReq.AddItem("State",
                Aws::DynamoDB::Model::AttributeValue().SetB(
                    SerializePlayerState(player)));

            // Version counter for optimistic concurrency control
            putReq.AddItem("Version",
                Aws::DynamoDB::Model::AttributeValue().SetN(
                    std::to_string(player->GetStateVersion())));

            Aws::DynamoDB::Model::WriteRequest wr;
            wr.SetPutRequest(putReq);
            writeRequests.push_back(wr);
        }

        // DynamoDB BatchWriteItem supports up to 25 items per call.
        // For ~800K writes/30s, this requires aggressive batching
        // and parallel requests across multiple threads.
        batchRequest.AddRequestItems("PlayerStates", writeRequests);
        m_dynamoClient->BatchWriteItemAsync(batchRequest, OnWriteComplete);
    }

    // Conditional write for critical operations (e.g., gold transfer)
    // to prevent race conditions -- when implemented correctly.
    bool TransferGold(const std::string& fromId, const std::string& toId,
                      int64_t amount, int64_t fromVersion)
    {
        // Use DynamoDB conditional expression to ensure atomicity:
        // Only debit if the version matches (optimistic locking)
        Aws::DynamoDB::Model::UpdateItemRequest debitReq;
        debitReq.SetTableName("PlayerStates");
        debitReq.AddKey("PlayerId",
            Aws::DynamoDB::Model::AttributeValue().SetS(fromId));

        debitReq.SetUpdateExpression("SET Gold = Gold - :amt, Version = Version + :one");
        debitReq.SetConditionExpression(
            "Gold >= :amt AND Version = :expectedVer");

        debitReq.AddExpressionAttributeValues(":amt",
            Aws::DynamoDB::Model::AttributeValue().SetN(std::to_string(amount)));
        debitReq.AddExpressionAttributeValues(":expectedVer",
            Aws::DynamoDB::Model::AttributeValue().SetN(
                std::to_string(fromVersion)));
        debitReq.AddExpressionAttributeValues(":one",
            Aws::DynamoDB::Model::AttributeValue().SetN("1"));

        auto result = m_dynamoClient->UpdateItem(debitReq);
        if (!result.IsSuccess())
            return false;  // Condition failed: insufficient gold or stale version

        // Credit the recipient (separate write -- not truly atomic across items)
        // This two-phase approach is where the gold dupe race condition lived.
        CreditGold(toId, amount);
        return true;
    }
};
```

### 5.3 Analytics Pipeline

New World generates approximately **23 million events per minute** for game
analytics. The pipeline:

1. **Amazon Kinesis** -- Ingests real-time event streams from all hubs
2. **Amazon S3** -- Long-term storage for event data
3. **Amazon Athena** -- Ad-hoc SQL queries against the S3 data lake
4. Enables real-time game design adjustments based on player behavior patterns

### 5.4 Microservices

Non-core gameplay systems run as **serverless microservices** using:

- **AWS Lambda** -- Character creation, company management, trading post logic
- **Amazon API Gateway** -- RESTful interfaces for these services
- This separation keeps the hub simulation focused on real-time gameplay while
  offloading asynchronous operations to serverless compute

### 5.5 Session-Based Content

Expeditions (5-player dungeons) and other instanced content use a **shared EC2
instance pool**. When a group enters an expedition, an instance is provisioned
from the pool. Upon completion, the instance is returned. This elasticity avoids
dedicating permanent compute to content that is only occasionally active.

---

## 6. Launch Issues and Fixes

### 6.1 Gold Duplication (Race Conditions)

**What happened:** In October 2021, following the release of patch 1.0.3 which
introduced server transfers, players discovered they could duplicate gold.
Approximately 150,000 players had transferred characters when the exploit was
discovered.

**Root cause:** When a player transferred to a new server, the transfer process
needed to serialize their character state (including gold) on the source server,
transmit it, and deserialize on the destination server. Some transfers failed
with a `Character_Persist_Failure` error, leaving the player's data in an
**inconsistent state** across the source and destination databases.

Players in this invalid state discovered that sending gold to another player and
then relogging would **restore their gold** -- the transfer rollback logic
returned the gold to the sender, but the recipient kept their copy. This is a
classic **check-then-act race condition**: the debit and credit were not executed
as an atomic transaction.

**Amazon's response:** All forms of wealth transfer were temporarily disabled --
player-to-player trading, the trading post, guild treasury operations, and
currency transfers. Server transfers were halted.

**The fix created a second exploit:** With direct trading disabled, the company
treasury system remained operational for town upgrades. Players discovered that
initiating a town upgrade would fail to deduct gold from the treasury but, upon
reconnection, would **add** the upgrade cost to the treasury instead. The same
non-atomic transaction pattern was at fault -- the debit failed silently while
the system recorded the intent to credit.

**What this reveals about the architecture:**

- Gold transfers were implemented as **two separate DynamoDB operations** (debit
  sender, credit recipient) rather than a single atomic transaction.
- DynamoDB does not natively support multi-item transactions in the same way a
  relational database does (though it does support `TransactWriteItems` for up
  to 100 items). The gold transfer logic likely predated proper use of this API,
  or the server transfer path bypassed it.
- The server transfer system introduced a new code path for state serialization
  that did not share the same consistency guarantees as the in-game trading path.

### 6.2 Chat HTML Injection

**What happened:** In October 2021, players discovered that New World's chat
system rendered user input as **unsanitized HTML**. The game's UI layer used an
HTML-based text rendering system (common in Lumberyard/CryEngine UIs), and chat
messages were injected directly into the DOM without escaping special characters.

**What players could do:**

- Inject `<img>` tags to display arbitrary images from game asset files, scaled
  to any size -- covering other players' entire screens
- Inject markup that crashed the game client when hovered over
- Inject scripts that interfered with the game's UI state
- Reports (likely exaggerated) of injecting quest-completion triggers for gold
  rewards

**The technical failure:** This is a textbook **Cross-Site Scripting (XSS)**
vulnerability, transplanted from web security into a game client. The chat input
did not escape HTML-significant characters (`<`, `>`, `&`, `"`, `'`). An IT
security consultant publicly stated that the developers "should be ashamed"
because this class of vulnerability was solved in web development decades ago.

**Amazon's initial fix:** They deployed a regex-based filter to block specific
known exploit strings -- a blocklist approach. This was quickly circumvented by
players using alternative encodings and tag variations. The proper fix (HTML
entity escaping or switching to a non-HTML text renderer) was deployed in a
subsequent patch.

**What this reveals about the architecture:**

- The Lumberyard UI layer (inherited from CryEngine's Scaleform / HTML-based UI
  system) renders text using an embedded HTML engine.
- The chat pipeline sent messages from client to server to other clients without
  sanitization at any stage -- neither the client, the server, nor the receiving
  client filtered the content.
- This was a client-side rendering vulnerability, not a server authority issue.
  The server was correctly relaying messages -- it just should not have relayed
  HTML markup.

### 6.3 Persistent Desync

**What happened:** Throughout 2021 and 2022, players reported chronic desync in
combat -- attacks registering visually but not dealing damage, enemies
teleporting, and skills passing through targets. The issue was worst during
**50v50 Wars**, the game's flagship large-scale PvP mode.

**Root causes:**

- **Server tick degradation under load.** The 30 Hz simulation target could not
  be maintained when 100 players converged in a single grid cell during Wars.
  Tick rate would drop, causing increasingly stale state on clients.
- **Animation-dependent hit resolution.** Because the server runs full skeletal
  animation for hit detection, any tick rate degradation directly impacts hit
  registration accuracy. If the server cannot advance animations at full rate,
  hit windows shift or collapse.
- **Grid cell hotspots.** Wars concentrate all participants in a small area,
  often within a single hub's grid cell. The non-contiguous hub design helps for
  normal play but cannot prevent a single cell from being overwhelmed by a
  designed-in 100-player battle.

**Fixes over 2022-2023:**

- Server-side optimizations to reduce per-entity simulation cost
- Adjustments to War participant rendering and simulation LOD
- Improved client-side interpolation to smooth over tick rate drops
- The 2023 Aeternum expansion and subsequent server merges consolidated
  populations onto fewer, higher-capacity worlds, reducing per-world overhead

### 6.4 Timeline of Major Issues

| Date | Issue | Root Cause | Fix |
|------|-------|------------|-----|
| Oct 2021 | Window-drag invulnerability | Server blocking on client input during i-frame phase | Server-side timeout on all client-input-dependent state transitions |
| Oct 2021 | Chat HTML injection | Unsanitized HTML in chat rendering pipeline | HTML entity escaping; moved to non-HTML text renderer |
| Oct 2021 | Server transfer gold dupe | Non-atomic debit/credit in transfer serialization | Halted transfers; rewrote transfer pipeline with transactional writes |
| Nov 2021 | Company treasury gold dupe | Same non-atomic pattern in town upgrade payments | Disabled treasury operations; unified transaction logic |
| 2021-2023 | Persistent combat desync | Server overload during mass PvP; animation-dependent hit detection | Simulation LOD, tick rate optimization, interpolation improvements |

---

## 7. Key Sources

- **AWS for Games Blog** -- ["The Unique Architecture behind Amazon Games'
  Seamless MMO New World"](https://aws.amazon.com/blogs/gametech/the-unique-architecture-behind-amazon-games-seamless-mmo-new-world/)
  -- Primary source on REPs, hubs, DynamoDB throughput, and analytics pipeline.

- **Amazon Developer Forums (Luxendra)** -- ["How Server/Client Authority is
  handled in New World"](https://devtrackers.gg/new-world/p/f2ab12db-notice-how-server-client-authority-is-handled-in-new-world)
  -- Full developer explanation of the server-authoritative model and the
  invulnerability bug.

- **PC Gamer** -- ["New World is not 'client authoritative,' says Amazon"](https://www.pcgamer.com/new-world-is-not-client-authoritative-amazon-says/)
  -- Reporting on the controversy and Amazon's response.

- **PC Gamer** -- ["New World invulnerability trick is hilariously easy to pull
  off"](https://www.pcgamer.com/new-world-invincibility-exploit/)
  -- Initial reporting on the window-drag exploit.

- **PC Gamer** -- ["Players are accidentally triggering New World's latest gold
  duplication exploit"](https://www.pcgamer.com/players-are-accidentally-triggering-new-worlds-latest-gold-duplication-exploit/)
  -- Coverage of the gold dupe cascade.

- **PC Gamer** -- ["Amazon deploys fix for New World exploit that let players
  crash the game using chat"](https://www.pcgamer.com/new-world-players-are-crashing-the-game-with-malicious-chat-exploit/)
  -- Coverage of the HTML injection vulnerability.

- **Inven Global** -- ["IT risk consultant says New World devs 'should be ashamed
  of themselves' for code injection vulnerability"](https://www.invenglobal.com/articles/15527/it-consultant-responds-new-world-code-injection-exploit)
  -- Security analysis of the chat injection.

- **DEV Community** -- ["New World Game Architecture"](https://dev.to/awscommunity-asean/new-world-game-architecture-46pb)
  -- Community analysis of the AWS architecture blog post.

- **Third Kind Games** -- ["Lumberyard and NetCode"](https://blog.thirdkindgames.com/amazon-lumberyard-tutorials/lumberyard-netcode/)
  -- GridMate DataSet/RPC documentation and server-authoritative patterns.

- **GitHub (aws/lumberyard)** -- [GridMate ReplicaMgr.cpp source](https://github.com/aws/lumberyard/blob/master/dev/Code/Framework/GridMate/GridMate/Replica/ReplicaMgr.cpp)
  -- Open-source GridMate replica management implementation.

- **MMORPG.com** -- ["New World Apologizes for Invulnerability Bug"](https://www.mmorpg.com/news/new-world-apologizes-for-invulnerability-bug-explains-how-serverclient-authority-works-updated-2000123500)
  -- Coverage of Amazon's apology and technical explanation.

- **Massively Overpowered** -- ["New World's gold dupe exploit stopgap fix
  revealed... another gold dupe"](https://massivelyop.com/2021/11/02/new-worlds-gold-dupe-exploit-stopgap-fix-revealed-another-gold-dupe/)
  -- The cascade of gold dupe exploits.
