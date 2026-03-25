# Valve Source Engine: Multiplayer Networking Deep Dive

Client-Side Prediction, Server Reconciliation, Entity Interpolation, and Lag Compensation

---

## Table of Contents

1. [History and Context](#1-history-and-context)
2. [Architecture Overview](#2-architecture-overview)
3. [Client-Side Prediction](#3-client-side-prediction)
4. [Server Reconciliation](#4-server-reconciliation)
5. [Entity Interpolation](#5-entity-interpolation)
6. [Lag Compensation](#6-lag-compensation)
7. [Key ConVars Reference](#7-key-convars-reference)
8. [Timeline Diagrams](#8-timeline-diagrams)
9. [Trade-offs and Known Issues](#9-trade-offs-and-known-issues)
10. [References](#10-references)

---

## 1. History and Context

Valve's approach to multiplayer networking evolved through decades of competitive online games:

- **QuakeWorld (1996)** -- John Carmack pioneered client-side prediction in QuakeWorld, introducing the idea that clients could run movement code locally rather than waiting for server confirmation. This was a foundational breakthrough that Valve built upon.

- **Half-Life 1 / GoldSrc Engine (1998)** -- Valve licensed the Quake engine and built GoldSrc. The original Half-Life and Counter-Strike (1999) introduced Valve's first implementation of client-side prediction and lag compensation. Yahn Bernier presented the technical foundations at GDC 2001 in his paper *"Latency Compensating Methods in Client/Server In-Game Protocol Design and Optimization."*

- **Source Engine (2004)** -- Released with Half-Life 2, the Source Engine formalized the four-pillar networking model: client-side prediction, server reconciliation, entity interpolation, and lag compensation. This architecture powered Counter-Strike: Source (2004), Day of Defeat: Source (2005), and Team Fortress 2 (2007).

- **Left 4 Dead (2008) / CS:GO (2012)** -- These titles refined the system, with CS:GO increasing the competitive tick rate to 64 (matchmaking) and 128 (third-party services like FACEIT/ESEA). The networking fundamentals remained the same.

- **Source 2 / CS2 (2023)** -- Counter-Strike 2 introduced sub-tick networking, moving beyond the fixed tick rate model to process inputs at the exact moment they occur within a tick. This represents the evolution beyond the classic Source model documented here.

The Source Engine's networking model became the canonical reference architecture for multiplayer FPS games, and its terminology (`cl_interp`, `tickrate`, lag compensation) became industry-standard vocabulary.

---

## 2. Architecture Overview

### Authoritative Server Model

The Source Engine uses a strict **client-server architecture** where the server is the single authority on game state. The core principle: **never trust the client.**

```
 Client A          Server           Client B
    |                 |                 |
    |--- UserCmd ---->|                 |
    |    (inputs)     |                 |
    |                 |-- simulate --   |
    |                 |   tick          |
    |<-- Snapshot ----|---- Snapshot -->|
    |   (world state) |  (world state) |
```

- **Clients** send only their input commands (key presses, mouse movements) to the server
- The **server** processes all inputs, simulates the game world, and sends authoritative state snapshots back to all clients
- Clients never send position, health, or any derived game state -- only raw inputs

### Tick Rate

The server simulates the game world in discrete time steps called **ticks**. The tick rate (`sv_tickrate` or `tickrate`) defines how many times per second the server updates:

| Game                | Default Tick Rate |
|---------------------|-------------------|
| Half-Life 2 DM      | 66 ticks/sec      |
| Counter-Strike: Source | 66 ticks/sec   |
| Team Fortress 2     | 66 ticks/sec      |
| Left 4 Dead 2       | 30 ticks/sec      |
| CS:GO (Matchmaking) | 64 ticks/sec      |
| CS:GO (FACEIT/ESEA) | 128 ticks/sec     |

At 66 ticks/sec, each tick represents approximately **15.15 ms** of simulation time. At 128 ticks/sec, each tick is approximately **7.8 ms**.

### Update Rate and Command Rate

Two additional rates govern the network communication:

- **`cl_updaterate`** -- How many state snapshots per second the client requests from the server (default: 20-66, capped by tick rate). This determines how often the client receives authoritative world state.

- **`cl_cmdrate`** -- How many command packets per second the client sends to the server (default: 30-66, capped by tick rate). Multiple commands within a single packet are batched.

- **`rate`** -- Maximum bytes per second the server will send to a client (bandwidth throttle).

The relationship:

```
cl_cmdrate  <= tickrate     (can't send inputs faster than ticks)
cl_updaterate <= tickrate   (can't receive updates faster than ticks)
```

### Packet Structure

**Client to Server (CUserCmd):**
Each input command (`CUserCmd`) contains:
- `command_number` -- monotonically increasing sequence number
- `tick_count` -- the client's estimation of the current server tick
- `viewangles` -- pitch, yaw, roll of the player's view
- `forwardmove`, `sidemove`, `upmove` -- movement input values
- `buttons` -- bitmask of pressed buttons (fire, jump, duck, etc.)
- `weaponselect`, `weaponsubtype` -- weapon switch commands

**Server to Client (Snapshot/Delta):**
The server sends compressed delta-encoded snapshots containing:
- The full world state (entity positions, animations, effects)
- The `last_command_acknowledged` -- the sequence number of the last `CUserCmd` the server has processed for this client
- Tick number of this snapshot

---

## 3. Client-Side Prediction

### The Problem

In a naive client-server model, every player action suffers a full round-trip delay:

```
Time 0ms:    Player presses "move forward"
Time 50ms:   Input arrives at server (one-way latency)
Time 50ms:   Server processes input, updates position
Time 100ms:  New position arrives back at client
Time 100ms:  Player sees themselves move (100ms after pressing the key)
```

At 100ms RTT (round-trip time), this delay makes the game feel sluggish and unresponsive.

### The Solution

Client-side prediction lets the client **immediately execute the same movement code** that the server will run, so the player sees instant feedback:

```
Time 0ms:    Player presses "move forward"
Time 0ms:    Client runs movement code locally -> player sees movement immediately
Time 0ms:    Client sends input to server
Time 50ms:   Server receives input, processes it (should produce same result)
Time 100ms:  Server confirmation arrives at client (used for reconciliation)
```

The player perceives **zero latency** on their own movement.

### How It Works

The key insight is **determinism**: if both client and server run the same movement code with the same inputs and the same starting state, they will produce the same result.

The Source Engine shares movement code between client and server through its **shared prediction code** architecture. The same C++ functions exist in both the client DLL and the server DLL:

```cpp
// Simplified version of the shared movement code
// This exact code runs on BOTH client and server
void CGameMovement::PlayerMove() {
    CheckParameters();    // validate inputs
    ReduceTimers();       // update status timers

    if (player->GetMoveType() == MOVETYPE_WALK) {
        FullWalkMove();   // ground/air movement + collision
    }
    // ... other move types (ladder, noclip, observer, etc.)
}

void CGameMovement::FullWalkMove() {
    if (player->GetGroundEntity() != NULL) {
        WalkMove();       // apply friction, accelerate on ground
    } else {
        AirMove();        // air strafing, gravity
    }
    CategorizePosition(); // determine if on ground, in water, etc.
}
```

### The Prediction Loop

On each client frame, the prediction system:

1. **Stores** the current input as a `CUserCmd` with a unique sequence number
2. **Sends** the command to the server
3. **Saves** the command in a circular buffer of pending (unacknowledged) inputs
4. **Runs** the shared movement code locally using that input
5. **Renders** the predicted result immediately

```
// Pseudo-code: Client prediction loop (per frame)
function clientFrame(dt):
    // 1. Sample input
    cmd = new CUserCmd()
    cmd.sequence = ++sequenceCounter
    cmd.tick_count = estimatedServerTick
    cmd.buttons = getButtonState()
    cmd.viewangles = getMouseLook()
    cmd.forwardmove = getForwardInput()
    cmd.sidemove = getStrafeInput()

    // 2. Send to server
    networkSend(cmd)

    // 3. Buffer the command locally
    pendingCommands.push(cmd)

    // 4. Run prediction (same code the server will run)
    runMovementSimulation(localPlayer, cmd)

    // 5. Save predicted state for this sequence number
    predictionHistory[cmd.sequence] = {
        position: localPlayer.position,
        velocity: localPlayer.velocity,
        onGround: localPlayer.onGround
    }
```

### What Gets Predicted

The Source Engine predicts:
- **Player movement** (position, velocity, ground state)
- **Weapon firing** (muzzle flash, tracer, ammo count, fire rate timing)
- **Weapon switching and reloading** (animation and timing)
- **Use key interactions** (doors, buttons)
- **View effects** (view punch from recoil, fall damage screen shake)

What is NOT predicted:
- **Other players' movements** (handled by interpolation instead)
- **Projectile trajectories from other players**
- **World physics objects** (server-authoritative)
- **Health and damage** (server-authoritative, though predicted in some cases)

### Source Engine Implementation Classes

```
CPrediction                  -- Main prediction system manager
C_BasePlayer::PhysicsSimulate() -- Entry point for local player prediction
CGameMovement               -- Shared movement code (client & server)
CMoveData                    -- Input/output data for movement
CUserCmd                     -- Player input command structure
```

The `CPrediction` class manages a buffer of `CUserCmd` commands. When the client needs to predict, it calls `CPrediction::RunCommand()`, which sets up the player entity state and calls into the shared `CGameMovement` code.

---

## 4. Server Reconciliation

### The Problem with Prediction

Client-side prediction is not always correct. The client predicts based on its local state, but the server may have additional information (another player blocking the path, a door closing, a teleporter activating). When the server's authoritative result differs from the client's prediction, the client must **reconcile**.

### How Reconciliation Works

Every server snapshot includes `last_command_acknowledged` -- the sequence number of the most recent `CUserCmd` the server has processed for this client.

When the client receives a server update:

```
// Pseudo-code: Server reconciliation
function onServerSnapshot(snapshot):
    // 1. Accept the server's authoritative state
    serverState = snapshot.playerState        // position, velocity, etc.
    acked = snapshot.lastCommandAcknowledged   // e.g., sequence #47

    // 2. Compare server state with our prediction for that same tick
    predictedState = predictionHistory[acked]

    // 3. Check for misprediction
    error = distance(serverState.position, predictedState.position)

    if error > PREDICTION_ERROR_TOLERANCE:
        // MISPREDICTION DETECTED!

        // 4. Snap to the server's authoritative state
        localPlayer.position = serverState.position
        localPlayer.velocity = serverState.velocity
        localPlayer.onGround = serverState.onGround

        // 5. Discard all acknowledged commands
        pendingCommands.removeWhere(cmd => cmd.sequence <= acked)

        // 6. RE-PREDICT: replay all unacknowledged commands
        //    from the corrected server state
        for cmd in pendingCommands:   // these are commands #48, #49, #50...
            runMovementSimulation(localPlayer, cmd)

        // Now localPlayer has the corrected predicted state
    else:
        // Prediction was correct! Just discard acknowledged commands.
        pendingCommands.removeWhere(cmd => cmd.sequence <= acked)

    // 7. Clean up old prediction history
    predictionHistory.removeWhere(seq => seq <= acked)
```

### The Re-Prediction Step (Key Insight)

The critical step is #6: **replaying unacknowledged inputs**. At any given moment, the client may have 5-10 commands "in flight" (sent to server but not yet acknowledged). When a misprediction occurs, the client cannot simply snap to the server state -- that would be the state from 50-100ms ago. Instead, it must:

1. Accept the server state (which is in the past)
2. Replay all commands the server hasn't processed yet
3. Arrive at a corrected present-time state

This is why the client buffers all unacknowledged commands.

### Visualizing the Buffer

```
Commands sent by client over time:
  [#40] [#41] [#42] [#43] [#44] [#45] [#46] [#47] [#48] [#49] [#50]
                                                     ^               ^
                                                     |               |
                                        server has processed    client is
                                        up to here (#47)        currently here

After receiving server snapshot (ack=#47):
  - Discard #40-#47 (server has confirmed these)
  - Keep #48, #49, #50 (still in flight)
  - If misprediction: reset to server state, replay #48, #49, #50
```

### Smoothing Misprediction Corrections

When a misprediction occurs, snapping instantly to the corrected position can be visually jarring. The Source Engine applies **error smoothing** to spread the correction over several frames:

```
// Pseudo-code: Error smoothing
predictionErrorPosition = correctedPosition - previousPredictedPosition
predictionErrorTime = currentTime

// During rendering:
elapsed = currentTime - predictionErrorTime
fraction = elapsed / SMOOTH_DURATION    // typically ~100ms

if fraction < 1.0:
    renderPosition = actualPosition + predictionErrorPosition * (1.0 - fraction)
else:
    renderPosition = actualPosition     // smoothing complete
```

The ConVar `cl_smoothtime` controls this smoothing window (default: 0.1 seconds).

---

## 5. Entity Interpolation

### The Problem

Other players (remote entities) cannot be predicted because their inputs are unknown. The client receives their positions only via server snapshots, which arrive at discrete intervals (e.g., every 15ms at 66 tick). Rendering them only when snapshots arrive would produce extremely choppy movement.

### The Solution: Render in the Past

Entity interpolation renders other players **slightly in the past**, smoothly blending between the two most recent known positions. Rather than showing where other players *might* be (extrapolation/prediction), the client shows where they *actually were* according to the server.

### How Interpolation Works

The client maintains a buffer of recent snapshots for each entity. At render time, it finds two snapshots that bracket the current **interpolation time** (which is behind real time):

```
interpolation_time = current_client_time - cl_interp

Example with cl_interp = 100ms:
  Current client time: 1000ms
  Interpolation time:  900ms

  Find two snapshots:
    Snapshot A: time=850ms, position=(10, 0, 0)
    Snapshot B: time=915ms, position=(13, 0, 0)

  Fraction = (900 - 850) / (915 - 850) = 50/65 = 0.769

  Rendered position = lerp(A.position, B.position, 0.769)
                    = (10 + (13-10) * 0.769, 0, 0)
                    = (12.31, 0, 0)
```

### The Interpolation Period (`cl_interp`)

The interpolation delay is calculated as:

```
cl_interp = max(cl_interp, cl_interp_ratio / cl_updaterate)
```

- **`cl_interp`** -- Minimum interpolation delay in seconds (default: 0.1)
- **`cl_interp_ratio`** -- Multiplier for the update interval (default: 2)
- **`cl_updaterate`** -- Snapshots per second from the server

Example at 66 tick with `cl_interp_ratio = 2`:
```
cl_interp = max(0.1, 2 / 66) = max(0.1, 0.0303) = 0.1 seconds
```

Competitive CS:GO players often set `cl_interp_ratio 1` and `cl_interp 0` to minimize the interpolation delay:
```
cl_interp = max(0, 1 / 128) = 0.0078 seconds (7.8ms at 128 tick)
```

### Why `cl_interp_ratio = 2`?

A ratio of 2 means the interpolation buffer covers **two update intervals**. This provides tolerance for one dropped or late packet -- the client always has a valid pair of snapshots to interpolate between. Setting it to 1 means any single dropped packet can cause interpolation failure (extrapolation or choppy visuals).

### Extrapolation (Fallback)

When the client runs out of valid snapshots (too much packet loss or jitter), it falls back to **extrapolation** -- projecting the last known velocity forward. This is unreliable (entities might turn, stop, or collide) and is used only as a last resort. The ConVar `cl_extrapolate` (default: 1) enables this, and `cl_extrapolate_amount` controls the maximum extrapolation duration.

### Interpolation Timeline

```
Real server time:  ----[T1]--------[T2]--------[T3]--------[T4]---->
                        |           |           |           |
                        v           v           v           v
Snapshots received      S1          S2          S3          S4
by client:

Client renders                  [interp between S1-S2]
at time T3:                      <-- cl_interp delay -->

What the client sees at time T3:
  - LOCAL player:  shown at T3 (predicted, present time)
  - OTHER players: shown at T3 - cl_interp (interpolated, in the past)
```

---

## 6. Lag Compensation

### The Problem

Because of client-side prediction and entity interpolation, the local player exists in the **present** but sees other players in the **past**. When the player shoots at an enemy, they're aiming at where the enemy was `cl_interp + half_RTT` milliseconds ago. If the server performs hit detection using the current (real-time) positions of entities, shots that visually appear on-target will miss.

### The Solution: Server-Side Time Rewind

When the server processes a player's attack command, it **rewinds** all other entities to where they were at the time the shooting player saw them. The server maintains a history of entity positions (a circular buffer of past states) and can reconstruct the game world at any recent point in time.

### How It Works Step by Step

```
1. Client renders frame at time T_client
   - Sees enemy at position P_interpolated (which is from time T_client - cl_interp)
   - Client fires at this visible position

2. Client sends fire command to server
   - Command includes: tick_count (client's estimation of server tick)

3. Server receives the command at time T_server (after network delay)
   - Server calculates the "command execution time":

     command_execution_time = T_server - client_latency

     This estimates when the client actually pressed the fire button.

4. Server calculates the lag-compensated time:

     lag_compensated_time = command_execution_time - cl_interp

     This accounts for the interpolation delay on the client.

5. Server rewinds all lag-compensated entities to their positions
   at lag_compensated_time

6. Server performs the hit trace (ray cast) against the rewound positions

7. Server restores all entities to their current positions

8. If hit was detected: apply damage
```

### Server-Side Implementation

```cpp
// Simplified pseudo-code of lag compensation in Source Engine

class CLagCompensationManager {
    // Circular buffer of entity states, one entry per tick
    struct LagRecord {
        float           simulationTime;
        Vector          origin;
        QAngle          angles;
        matrix3x4_t     boneMatrix[MAXSTUDIOBONES]; // hitbox transforms
    };

    // Each entity has a history of positions
    CUtlFixedLinkedList<LagRecord> entityHistory[MAX_ENTITIES];

    void StartLagCompensation(CBasePlayer* player, CUserCmd* cmd) {
        // Calculate how far back to rewind
        float targetTime = cmd->tick_count * tickInterval;

        // Clamp: don't rewind more than sv_maxunlag (default 1.0s)
        float maxUnlag = sv_maxunlag.GetFloat();
        float serverTime = gpGlobals->curtime;

        if (serverTime - targetTime > maxUnlag) {
            targetTime = serverTime - maxUnlag;
        }

        // For each entity that can be lag-compensated:
        for (int i = 1; i <= gpGlobals->maxClients; i++) {
            CBasePlayer* target = UTIL_PlayerByIndex(i);
            if (target == player) continue;  // don't rewind self

            // Save current state (for restoration later)
            SaveCurrentState(target);

            // Find the two history records bracketing targetTime
            LagRecord* prev = NULL;
            LagRecord* next = NULL;
            FindRecordsForTime(target, targetTime, &prev, &next);

            if (prev && next) {
                // Interpolate between the two records
                float frac = (targetTime - prev->simulationTime) /
                             (next->simulationTime - prev->simulationTime);

                Vector rewindPos = Lerp(frac, prev->origin, next->origin);
                QAngle rewindAng = Lerp(frac, prev->angles, next->angles);

                // Move the entity to the rewound position
                target->SetAbsOrigin(rewindPos);
                target->SetAbsAngles(rewindAng);
                // Also rewind hitboxes (bone matrices)
                RestoreHitboxes(target, prev, next, frac);
            }
        }
    }

    void FinishLagCompensation(CBasePlayer* player) {
        // Restore all entities to their current positions
        for (int i = 1; i <= gpGlobals->maxClients; i++) {
            CBasePlayer* target = UTIL_PlayerByIndex(i);
            if (target == player) continue;
            RestoreCurrentState(target);
        }
    }
};

// Usage in weapon fire:
void CWeaponCSBase::PrimaryAttack() {
    CBasePlayer* owner = GetOwner();

    lagcompensation->StartLagCompensation(owner, owner->GetCurrentCommand());

    // Perform ray trace -- hits are tested against REWOUND positions
    FireBullets(owner);

    lagcompensation->FinishLagCompensation(owner);
}
```

### The History Buffer

The server stores entity positions for up to `sv_maxunlag` seconds (default: 1.0 second). At 66 ticks/sec, that's approximately 66 history records per entity. Each record includes:

- Position (origin)
- Angles (orientation)
- Bone transforms (hitbox positions for skeletal models)
- Simulation time (the tick time this state was valid)
- Animation data (sequence, cycle)

### Maximum Lag Compensation Window

The `sv_maxunlag` ConVar (default: 1.0 second) caps how far back the server will rewind. If a player has more than 1 second of latency, their shots will start to miss even when they appear on-target. This prevents abuse from artificially inflated ping.

---

## 7. Key ConVars Reference

### Client ConVars

| ConVar | Default | Description |
|--------|---------|-------------|
| `cl_predict` | `1` | Enable/disable client-side prediction. When 0, the player experiences full round-trip latency on all actions. |
| `cl_interp` | `0.1` | Entity interpolation period in seconds. The minimum time other entities are rendered "in the past." |
| `cl_interp_ratio` | `2` | Interpolation safety ratio. `cl_interp` is calculated as `max(cl_interp, cl_interp_ratio / cl_updaterate)`. Value of 2 tolerates one dropped packet. |
| `cl_updaterate` | `20`* | Number of snapshot updates per second requested from server. *Default varies by game; competitive CS:GO uses 128. |
| `cl_cmdrate` | `30`* | Number of input command packets per second sent to server. *Default varies by game. |
| `cl_smoothtime` | `0.1` | Duration (seconds) over which prediction errors are visually smoothed. |
| `cl_smooth` | `1` | Enable/disable smoothing of prediction corrections. |
| `cl_lagcompensation` | `1` | Enable client-side markers for lag compensation. |
| `cl_extrapolate` | `1` | Allow extrapolation when interpolation data is unavailable. |
| `rate` | `196608`* | Maximum bytes/sec from server to client. *Varies; CS:GO competitive uses 786432. |

### Server ConVars

| ConVar | Default | Description |
|--------|---------|-------------|
| `sv_maxunlag` | `1.0` | Maximum lag compensation rewind in seconds. |
| `sv_unlag` | `1` | Enable/disable server-side lag compensation entirely. |
| `sv_maxupdaterate` | `66`* | Maximum update rate the server will send to clients. |
| `sv_minupdaterate` | `10` | Minimum update rate. |
| `sv_maxcmdrate` | `66`* | Maximum command rate accepted from clients. |
| `sv_mincmdrate` | `10` | Minimum command rate. |
| `sv_client_predict` | `-1` | Force client prediction on/off for all clients (-1 = use client setting). |
| `sv_clockcorrection_msecs` | `60` | Maximum clock drift correction per snapshot. |
| `sv_lagcompensation_teleport_dist` | `64` | If an entity moved more than this distance in one tick, skip lag compensation for it (likely teleported). |

### Tick Rate

| ConVar | Typical | Description |
|--------|---------|-------------|
| `tickrate` | `66` | Server tick rate (set at launch, not changeable at runtime in most Source games). CS:GO uses `-tickrate 128` launch option for 128-tick servers. |

### Competitive CS:GO Recommended Settings

```
cl_interp "0"
cl_interp_ratio "1"
cl_cmdrate "128"
cl_updaterate "128"
rate "786432"
```

This minimizes the interpolation delay to `max(0, 1/128) = 7.8ms` and ensures the client communicates at the full 128-tick rate.

---

## 8. Timeline Diagrams

### Full Prediction and Reconciliation Timeline

```
CLIENT TIMELINE (time flows left to right -->)
===========================================================================

Client
Ticks:    T1      T2      T3      T4      T5      T6      T7      T8
          |       |       |       |       |       |       |       |
Input:    Cmd#1   Cmd#2   Cmd#3   Cmd#4   Cmd#5   Cmd#6   Cmd#7   Cmd#8
          |       |       |       |       |       |       |       |
Predict:  P1      P2      P3      P4      P5      P6      P7      P8
          |       |       |       |       |       |       |       |
Send:     >------>|       |       |       |       |       |       |
          |  Cmd#1 in flight      |       |       |       |       |
          |       >------>|       |       |       |       |       |
          |       | Cmd#2 in flght|       |       |       |       |

SERVER TIMELINE
===========================================================================

Server
Ticks:    T1      T2      T3      T4      T5      T6      T7      T8
          |       |       |       |       |       |       |       |
                          |       |
Receive:          Cmd#1-->|       |
                  arrives |Cmd#2->|
                          |       |
Process:                  Exec#1  Exec#2
                          |       |
Snapshot:                 S3------>|
                          (ack=1) |S4---->|
                                  (ack=2)

RECONCILIATION (at client T5, receiving snapshot S3 with ack=1)
===========================================================================

Client has: [Cmd#2, Cmd#3, Cmd#4, Cmd#5] still pending (unacknowledged)

    Server says: "After processing Cmd#1, player is at position X"
    Client predicted: position was P1 after Cmd#1

    If X != P1 (misprediction):
      1. Snap state to X (server's authoritative position after Cmd#1)
      2. Re-apply Cmd#2 -> new P2'
      3. Re-apply Cmd#3 -> new P3'
      4. Re-apply Cmd#4 -> new P4'
      5. Re-apply Cmd#5 -> new P5' (corrected current position)
```

### Entity Interpolation Timeline

```
SERVER                CLIENT RECEIVE              CLIENT RENDER
(generates)           (arrives)                   (interpolates)
==========================================================================

Tick 100              .                           .
pos=(10,0,0)          .                           .
  |                   .                           .
  |----network------->Tick 100 snapshot            .
  |   delay (~50ms)   received at T=150ms         .
  |                   |                           .
Tick 101              |                           .
pos=(11,0,0)          |                           .
  |                   |                           .
  |----network------->Tick 101 snapshot           Rendering at T=200ms:
  |   delay (~50ms)   received at T=200ms        interp_time = 200 - 100 = 100ms
  |                   |
  |                   |                          Need snapshots around T=100ms
Tick 102              |                          Use Tick 100 (T=100) and
pos=(12,0,0)          |                               Tick 101 (T=115)
  |                   |
  |                   |                          fraction = (100-100)/(115-100) = 0
  |                   |                          render at (10,0,0) <-- Tick 100 pos
  |                   |
  |----network------->Tick 102 snapshot           Rendering at T=215ms:
                      received at T=250ms        interp_time = 215 - 100 = 115ms

                                                 fraction = (115-100)/(115-100) = 1.0
                                                 render at (11,0,0) <-- Tick 101 pos
```

### Lag Compensation Timeline

```
TIME:  0ms    25ms    50ms    75ms    100ms   125ms   150ms
       |       |       |       |       |       |       |

PLAYER A (Shooter, 80ms RTT = 40ms one-way):
       |                                       |
       A sees enemy at                         A's fire command
       position P_old                          arrives at server
       (interpolated from
       ~140ms ago total)
       |                                       |
       A fires! -------- network (40ms) -----> |
       |                                       |

ENEMY B (Target):
       [at P_old]  ...moving...  ...moving... [at P_new, behind wall]

SERVER at T=125ms:
       |
       Receives A's fire command
       |
       Calculates: A pressed fire at ~T=85ms (125 - 40ms latency)
       Subtracts cl_interp: target time = ~T=-15ms... hmm let's redo:

       Actually:
       - A saw the frame rendered at T=0ms on client
       - That frame showed enemy interpolated ~100ms in the past
       - A pressed fire at T=0ms (client time)
       - Command arrives at server at T=40ms (server time)
       - Server estimates A pressed fire at T=0ms (server_time - latency)
       - Server rewinds entities to T=0ms - cl_interp = T=-100ms
       - At T=-100ms, enemy WAS at P_old (not yet behind wall)
       - Hit trace SUCCEEDS against P_old
       |
       Enemy B takes damage even though B is now behind the wall!
       |
       Total "behind the wall" window: ~140ms (RTT + interp)
```

### Combined System Overview

```
==========================================================================
                    COMPLETE SOURCE ENGINE NETWORKING
==========================================================================

                     CLIENT A                    SERVER                    CLIENT B
                     ========                    ======                    ========

Frame N:
  Sample input -----> CUserCmd #N
  Predict locally     |
  (movement,          |---- network (40ms) --->  Receive Cmd#N
   weapons)           |                          Process Cmd#N
  Render:             |                          Update A's state
   Self: predicted    |                          Store A's pos in
    (present time)    |                           lag comp history
   Others: interp     |
    (in the past)     |                          Build snapshot
                      |                           (all entity states,
                      |                            ack = N)
                      |                          |
                      |<-- network (40ms) -----  |---- network (40ms) -->
                      |                          |
Frame N+5:            |                          |
  Receive snapshot    |                                                   Receive snapshot
  (ack = N)           |
  Reconcile:          |                                                   Interpolate A's
   Compare server     |                                                    position from
   state vs predicted |                                                    snapshot history
   If mismatch:       |
    Reset + replay    |                                                   Render A in
    unacked cmds      |                                                    the past

==========================================================================
```

---

## 9. Trade-offs and Known Issues

### "Getting Shot Behind Walls" (Peeker's Disadvantage for the Target)

The most infamous artifact of lag compensation. Because the server rewinds time to validate hits, a player who ducks behind cover can still be killed **after** they believe they are safe. The total vulnerability window is approximately:

```
vulnerability_window = shooter_latency + target_latency + cl_interp
```

For two players with 50ms ping each and 100ms interpolation:
```
vulnerability_window = 50 + 50 + 100 = 200ms
```

This is a **deliberate design trade-off**. Valve chose to favor the shooter's experience ("if your crosshair was on the enemy and you clicked, you should get the hit") over the target's experience ("I was already behind the wall"). The alternative -- no lag compensation -- would mean every shot requires the shooter to lead their target by an unpredictable, latency-dependent amount, which is far worse for gameplay.

### Peeker's Advantage

A related but distinct issue. The player who peeks around a corner has an inherent advantage because:

1. The peeker sees the stationary target at their interpolated (past) position -- which is approximately correct since the target isn't moving
2. The target sees the peeker at their interpolated (past) position -- which is wrong because the peeker is actively moving

The peeker effectively gets `latency` milliseconds of free visibility before the target can react. This is intrinsic to any system with network latency and is not caused by lag compensation.

### Prediction Errors and "Rubber-Banding"

When predictions fail frequently (e.g., two players colliding, complex physics interactions, or server-side game logic the client doesn't replicate), players experience correction snaps. The `cl_smooth` and `cl_smoothtime` ConVars mitigate this visually, but large corrections are still perceptible. Common causes:

- **Player-player collisions** -- The client doesn't know other players' exact positions
- **Moving platforms** -- Timing differences between client and server
- **Server-side anti-cheat corrections** -- Server overrides client position
- **Packet loss** -- Missing input commands cause the server to simulate different inputs

### Clock Drift and Synchronization

The client and server have independent clocks that drift apart. The Source Engine uses a clock correction algorithm (`sv_clockcorrection_msecs`) that gradually nudges the client's tick estimation to match the server's. Sudden corrections would cause visible hitches.

### Bandwidth vs. Update Rate Trade-off

Higher `cl_updaterate` means smoother interpolation and faster reconciliation, but requires more bandwidth. The `rate` ConVar caps bandwidth, and if the update rate exceeds available bandwidth, the server will reduce the effective update rate, increasing interpolation delay and prediction error feedback time.

### Interpolation vs. Responsiveness Trade-off

- **Higher `cl_interp`**: Smoother other-player movement, more tolerance for packet loss, but increased temporal inaccuracy when aiming at moving targets.
- **Lower `cl_interp`**: More responsive/accurate other-player positions, but more susceptible to packet loss causing visual artifacts (extrapolation jitter).

### Entity Teleportation

If an entity moves too far between two ticks (e.g., teleportation, respawn), interpolation would produce a nonsensical sliding effect. The `sv_lagcompensation_teleport_dist` ConVar (default: 64 units) tells the system to skip interpolation and snap the entity to the new position when the delta exceeds this threshold.

### The "100% Accurate" Myth

No networked game can be 100% accurate for all players simultaneously. The Source Engine's system is a set of compromises:

| Technique | Benefits | Costs |
|-----------|----------|-------|
| Client-side prediction | Responsive local movement | Mispredictions require correction |
| Server reconciliation | Corrects prediction errors | Replay cost; visual snapping on errors |
| Entity interpolation | Smooth remote entity movement | Other players shown in the past |
| Lag compensation | Shots hit where aimed | Targets can be hit after reaching cover |

---

## 10. References

### Primary Sources (Valve)

1. **"Source Multiplayer Networking"** -- Valve Developer Community Wiki
   https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking
   The canonical reference for Source Engine networking architecture. Covers tick rate, interpolation, prediction, and lag compensation with diagrams.

2. **"Prediction"** -- Valve Developer Community Wiki
   https://developer.valvesoftware.com/wiki/Prediction
   Detailed documentation of the client-side prediction system, including the `CPrediction` class, `CUserCmd`, and the prediction/reconciliation loop.

3. **"Lag Compensation"** -- Valve Developer Community Wiki
   https://developer.valvesoftware.com/wiki/Lag_Compensation
   Documentation of the server-side lag compensation system, `CLagCompensationManager`, and the time-rewind hit detection process.

4. **Bernier, Yahn W. "Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization."** Game Developers Conference, 2001.
   https://www.gamedevs.org/uploads/latency-compensation-in-client-server-protocols.pdf
   The foundational paper from Valve's lead network programmer. Describes the original GoldSrc implementation that evolved into the Source Engine system. Covers input prediction, lag compensation, and entity interpolation in detail.

### Secondary Sources

5. **Gambetta, Gabriel. "Fast-Paced Multiplayer"** (4-part series)
   - Part I: Client-Server Game Architecture -- https://gabrielgambetta.com/client-server-game-architecture.html
   - Part II: Client-Side Prediction and Server Reconciliation -- https://gabrielgambetta.com/client-side-prediction-server-reconciliation.html
   - Part III: Entity Interpolation -- https://gabrielgambetta.com/entity-interpolation.html
   - Part IV: Lag Compensation -- https://gabrielgambetta.com/lag-compensation.html
   Excellent visual explanations of the same concepts with interactive demos. Heavily inspired by the Valve model.

6. **Fiedler, Glenn. "Networked Physics"** and related articles
   https://gafferongames.com/
   In-depth coverage of deterministic lockstep, state synchronization, and snapshot interpolation from a physics simulation perspective.

7. **Carmack, John. Plan files and QuakeWorld source code** (1996-1998)
   The original client-side prediction implementation that inspired Valve's approach.

8. **Valve Source SDK** (publicly available)
   https://github.com/ValveSoftware/source-sdk-2013
   The actual Source Engine SDK code, including `CPrediction`, `CGameMovement`, `CLagCompensationManager`, and `CUserCmd` implementations.

---

*This document describes the networking model used in Source Engine 1 games (2004-2023). Counter-Strike 2 (Source 2, 2023) introduced sub-tick networking which processes inputs at their exact timestamp within a tick, reducing the quantization error of fixed tick rates. The fundamental principles of prediction, reconciliation, interpolation, and lag compensation remain the same.*
