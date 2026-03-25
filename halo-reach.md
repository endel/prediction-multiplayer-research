# Halo: Reach -- Networking Architecture

**Speaker:** David Aldridge, Lead Networking Engineer at Bungie
**Talk:** "I Shot You First: Networking the Gameplay of HALO: REACH"
**Event:** GDC 2011

---

## 1. Overview

David Aldridge spent three years as Lead Networking Engineer at Bungie working on the
networking systems for Halo: Reach. His GDC 2011 talk, titled "I Shot You First:
Networking the Gameplay of HALO: REACH," is widely regarded as one of the most
important GDC presentations on multiplayer game networking.

The talk covers the evolution from Halo 3's networking to Halo: Reach, focusing on
how Bungie achieved a dramatic reduction in bandwidth usage (down to roughly 20% of
Halo 3's requirements), built robust tools for network debugging, and designed
systems to resolve the fundamental tension in online shooters: every player is
playing a slightly different game due to latency, but the experience must feel fair.

The networking design is rooted in **"The TRIBES Engine Networking Model"** presented
by Mark Frohnmayer and Tim Gift at GDC 1999, which established:

- A host/client model resilient to cheating
- Protocols for semi-reliable data delivery
- Support for both persistent state and transient events
- Scalability to match available bandwidth

Halo: Reach supports up to **16 players** in a single match, using a **peer-hosted
architecture** where one player's console acts as the authoritative server.

---

## 2. Core Networking Concepts

Aldridge defined three foundational concepts that underpin the entire architecture:

### Replication

> "The communication of state or events to a remote peer."

Replicating an object means causing it to be created and kept approximately in sync
between peers. Replication is broken into three distinct data channels:

| Channel        | Delivery       | Direction         | Purpose                                              |
|----------------|----------------|-------------------|------------------------------------------------------|
| **State Data** | Eventual (reliable) | Host → Client | Object positions, health, territory timers (~150 properties) |
| **Events**     | Unreliable     | Bidirectional     | Weapon firing, detonations, hit effects (~50 event types) |
| **Control Data** | Best-effort (high frequency) | Client → Host | Player input analog values and position updates |

### Authority

> "Permission to update the persistent state of an object."

The host maintains authority over critical decisions:

- Damage calculation and application
- Object creation and destruction
- Grenade physics
- Equipment ability activation (e.g., invulnerability timing)

When clients perform actions, they send that information to the host, which accepts
or rejects it according to various criteria.

### Prediction

> "Extrapolating the current properties of an entity based on historical
> authoritative data and local guesses about the future."

Prediction is the technique of changing client state *before* receiving
authentication from the server, making the game feel responsive despite network
latency.

---

## 3. Deterministic Simulation

### What Is Deterministic

Halo: Reach separates its simulation into **deterministic** and **non-deterministic**
layers:

**Deterministic (game simulation):**

- All game objects and their movement
- Physics simulation
- Game state transitions
- Damage calculations
- Collision detection

The game state is updated in **fixed time steps called ticks** through a
deterministic process. Given any initial game state, advancing it by some number of
ticks and replaying the same player inputs will always produce the identical result.
This property is essential for:

- **Co-op (synchronous) networking** -- where all peers must agree on game state
- **Theater Mode (Films)** -- deterministic replay of recorded sessions

```
// Conceptual deterministic simulation loop
function simulateTick(gameState, playerInputs, tickNumber):
    // Fixed timestep ensures determinism
    dt = FIXED_TICK_DURATION  // e.g., 1/30 second

    for each entity in gameState.entities:
        applyInput(entity, playerInputs[entity.owner])
        updatePhysics(entity, dt)
        resolveCollisions(entity, gameState)
        applyDamage(entity, gameState.pendingDamage)

    // Same inputs + same initial state = same output state
    return gameState
```

### What Is Non-Deterministic

**Non-deterministic (presentation layer):**

- Rendering and visual effects
- Sound effects and audio mixing
- Particle systems
- Camera effects and screen shake
- UI elements

These elements are computed locally on each machine and do not need to be
synchronized across the network. They respond to the deterministic game state but
their exact output can vary per machine without affecting gameplay correctness.

```
// Non-deterministic presentation runs independently
function renderFrame(gameState, localCamera):
    // These vary per-machine -- that's acceptable
    renderEntities(gameState, localCamera)
    updateParticles(localDeltaTime)   // local frame timing
    mixAudio(localListenerPosition)   // local audio
    applyPostProcessing(localCamera)  // local visual effects
```

This split is what makes Films (Theater Mode) work: only the deterministic
simulation inputs need to be recorded. On replay, the simulation reproduces the
exact same game state, and the presentation layer generates visuals and audio fresh
from that state.

### Lockstep vs. Host/Client

Aldridge explicitly distinguished between two networking models:

- **Lockstep (deterministic/input-passing):** Common in RTS games. All peers
  simulate identically, exchanging only inputs. Requires strict determinism and has
  input latency proportional to round-trip time.

- **Host/Client (Tribes-style):** What Halo uses. One peer is authoritative; clients
  predict locally and correct based on host state. Allows responsive input at the
  cost of occasional corrections.

Halo: Reach uses the **host/client model** for competitive multiplayer, leveraging
deterministic simulation properties primarily for replay/debugging rather than for
network synchronization.

---

## 4. Target-Relative Gunshots vs. World-Relative Hit Detection

One of the most impactful design decisions in Halo: Reach's networking is how
gunshots are communicated across the network.

### The Problem

In a traditional (world-relative) approach, the client sends the exact world-space
coordinates of where the bullet was fired and its direction vector. The server then
performs its own hit detection. Due to latency, the target may have moved between
when the shooter fired and when the server processes the shot, causing shots that
looked perfect on the shooter's screen to miss on the server.

### The Solution: Target-Relative Shots

Instead of sending world-space ray directions, Halo sends **target-relative gunshot
information**:

> "I shot directly at the head of Player 7."

The server then validates whether a clean line-of-sight exists from the shooter's
weapon to the target's head. If the geometry check passes, the shot connects.

```
// Client-side: when the player fires
function onPlayerFire(shooter, crosshairTarget):
    if crosshairTarget.isPlayer:
        // Send target-relative shot
        sendToHost({
            type: "SHOT_FIRED",
            shooter: shooter.id,
            target: crosshairTarget.player.id,
            bodyPart: crosshairTarget.hitZone,  // HEAD, TORSO, etc.
            weapon: shooter.currentWeapon
        })
    else:
        // Environment shot -- send world-relative
        sendToHost({
            type: "SHOT_FIRED",
            shooter: shooter.id,
            target: null,
            worldDirection: crosshairTarget.direction
        })

// Host-side: validate the shot
function validateShot(shotEvent):
    shooter = getEntity(shotEvent.shooter)
    target = getEntity(shotEvent.target)

    if target == null:
        return false

    // Check line of sight from shooter's weapon to target's body part
    hitPoint = target.getBodyPartPosition(shotEvent.bodyPart)
    origin = shooter.getWeaponMuzzlePosition()

    if hasLineOfSight(origin, hitPoint):
        applyDamage(target, shotEvent.weapon, shotEvent.bodyPart)
        return true
    else:
        return false  // Geometry blocked the shot
```

### Why This Works

This approach **favors the shooter** -- what appeared on the shooter's screen is
honored by the server, as long as the geometric plausibility check passes. The key
insight is that a latency window (approximately 200-300ms) is acceptable: if the
client saw a valid shot within that window, the server will likely confirm it
because the target was recently in that position.

This is particularly important for Halo's combat, where precision aiming at specific
body parts (headshots) is a core skill. Missing shots that looked clean on the
shooter's screen would feel fundamentally unfair.

---

## 5. "I Shot You First" -- Resolving Conflicting Perceptions

The title of the talk addresses the most challenging problem in networked shooters:
two players shoot each other simultaneously, and both believe they fired first.

### The Core Problem

Due to network latency, every player is actually playing a **slightly different
game**. Each client has a local approximation of the game state that is some number
of milliseconds behind the host's authoritative state. When two players engage in
a firefight:

- Player A sees themselves shoot Player B first
- Player B sees themselves shoot Player A first
- The host sees a slightly different sequence from both

### Halo's Resolution Strategy

1. **The host is authoritative for damage.** All damage decisions flow through the
   host. The host's timeline is the canonical reality.

2. **Target-relative shots favor the shooter.** When a shot arrives at the host, if
   the geometric plausibility check passes, the shot counts. This means that what
   happened on the shooter's screen is largely honored.

3. **Latency tolerance windows.** The system has defined tolerances:
   - **Tournament (close-quarters):** 100-133ms
   - **Tournament (ranged):** 133-300ms
   - **Casual (close-quarters):** 200-300ms
   - **Casual (ranged):** 300ms+

4. **Hiding lag through event sequencing.** Designers sometimes changed the event
   sequencing for particular game actions so that the necessary plausibility is
   preserved, but the lag is hidden somewhere in the message sequence. The key is
   identifying *where* in the action sequence latency can be absorbed without the
   player noticing.

### The Melee Problem

Melee combat is particularly sensitive because it involves close-range, high-stakes
exchanges. The "melee dance" -- two players trading melee blows -- is a fundamental
part of Halo combat. The system handles melee edge cases through latency
compensation similar to ranged hit detection, acknowledging that two players may
both perceive they landed the killing blow.

### The "Lag Button"

During Halo: Reach playtests, testers had a dedicated **"lag" button** they could
press whenever anything felt laggy. This injected a timestamp into the Film
recording so developers could jump directly to the reported moment and analyze what
went wrong using the network profiler data embedded in the replay. Whenever a
particular event received many lag reports, network engineers experimented with
different approaches to hide the lag until reports stopped.

---

## 6. Bandwidth Optimization

### Halo 3 Baseline

Inspection of Halo 3's bandwidth profile revealed the following breakdown:

| Category                          | % of Bandwidth |
|-----------------------------------|---------------|
| Positions / Velocities / Orientations | 50%       |
| Player control data               | 20%           |
| Weapon firing / Bullets / Damage  | 20%           |
| Other                             | 10%           |

### Reach Targets and Results

Halo: Reach achieved a dramatic reduction -- **bandwidth usage was reduced to
approximately 20% of Halo 3's usage** -- while supporting the same 16-player matches.

| Metric                                 | Value        |
|----------------------------------------|-------------|
| Minimum total upstream (host, 16p)     | 250 kbits/s |
| Maximum total upstream (single peer)   | 675 kbits/s |
| Maximum bandwidth to individual client | 45 kbits/s  |
| Per-character replication (combat rate) | ~1 kbit/s   |
| Minimum packet rate for solid gameplay | 10 Hz       |

### Key Optimization Techniques

**1. Aggressive Prioritization**

Rather than networking everything equally, the system calculates per-object,
per-client priorities:

```
function calculatePriority(object, client):
    priority = BASE_PRIORITY

    // Distance and direction (primary factors)
    distance = length(object.position - client.viewPosition)
    priority *= distanceFalloff(distance)

    dot = dot(client.viewDirection, normalize(object.position - client.viewPosition))
    priority *= viewDirectionBoost(dot)

    // Modifiers
    priority *= object.size        // larger objects are more important
    priority *= object.speed       // fast-moving objects need more updates

    // Combat relevance boosts
    if object.isDamagingClient or client.isDamagingObject:
        priority *= COMBAT_BOOST   // entities in combat are critical

    if object.isThrowingGrenade:
        priority *= GRENADE_BOOST  // thrown grenades get high priority

    return priority
```

> "Unreliability enables aggressive prioritization, which lets us handle the
> richness of the simulation."

Because events are unreliable, the system can freely drop low-priority updates when
bandwidth is constrained. The most important objects (nearby enemies, active
combatants) always get bandwidth first.

**2. Removing Redundant Control Replication**

Host-to-client control replication consumed 22% of bandwidth in Halo 3. By removing
duplicated data and information that clients didn't actually need, they achieved a
**60% reduction in control data** (14% overall bandwidth reduction).

**3. Eliminating Ragdoll Networking**

In Halo 3, ragdoll physics were continuously networked between peers. Reach changed
this to only synchronize the **initial state** of ragdolls. Since ragdolls are
cosmetic (a dead player doesn't need identical ragdoll positions on all screens),
each client simulates ragdoll physics locally from the shared initial conditions.

**4. Fixing Prioritization Bugs**

Profiling revealed that idle items rolling slowly on terrain were consuming
disproportionate bandwidth because the prioritization system kept trying to update
their slowly-changing positions. Fixing these edge cases freed significant bandwidth.

**5. Game Design Changes for Networking**

The team reduced artificial friction on certain surfaces, which decreased the
frequency of micro-updates for objects barely moving on terrain. This is a notable
example of **game design being influenced by networking constraints** -- a theme
Aldridge emphasized throughout the talk.

**6. Smoothing High-Rate-of-Fire Weapon Bursts**

Weapons with high fire rates generated bursts of bullet packets. The system smoothed
these by batching and spreading updates across multiple network ticks rather than
sending them all at once.

---

## 7. Network Profiling Embedded in Replays

### The Films System

Halo's **Films** (Theater Mode) provide deterministic replay of gameplay sessions.
Because the simulation is deterministic, only player inputs need to be recorded;
on playback, the simulation reproduces the exact same game state.

### The Innovation: Network Data in Films

Traditionally, Films were useful for debugging gameplay but not networking -- network
systems sit idle during offline playback. Halo: Reach's key innovation was
**embedding network profiling data directly into Film recordings**.

This meant that any replay could be examined to see:

- Packets sent to every player at every frame
- Bandwidth usage per object per client
- Priority calculation results
- Replication breakdown by type (state, events, control)
- Specific event and state replications per object

```
// Conceptual: recording network debug data into Films
function recordNetworkFrame(filmRecorder, tick):
    for each client in connectedClients:
        filmRecorder.addNetworkSnapshot(tick, {
            clientId: client.id,
            packetsSent: client.outgoingPackets,
            bytesSent: client.outgoingBytes,
            objectPriorities: client.priorityTable,
            replicationBreakdown: {
                stateBytes: client.stateDataBytes,
                eventBytes: client.eventDataBytes,
                controlBytes: client.controlDataBytes
            }
        })
```

### The Profiler Tools

The profiling system provided:

- **Per-session bandwidth graphs** showing replication by type (state, events,
  control) over time
- **Per-object inspection** allowing engineers to drill into any specific entity and
  see exactly what data was being sent about it
- **Priority visualization** showing why certain objects were receiving bandwidth
- **Lag-button integration** where tester-reported lag moments were bookmarked
  in the Film for direct navigation

### Monthly Network Playtests

Bungie ran dedicated network performance playtests **once a month** during
production. These playtests used **traffic shaping tools** to simulate adverse
network conditions (high latency, packet loss, low bandwidth). Combined with the
Films system, this gave engineers a continuous stream of real-world network
performance data to analyze offline.

---

## 8. Authority Model and Host Migration

### The Host/Client Architecture

One of the 16 players in a match acts as the **host** (authoritative server). The
host console:

- Authorizes essentially every action that takes place in the game
- Makes final decisions on damage, object creation, and object destruction
- Runs the canonical simulation that all clients approximate
- Sends state updates to all connected clients

Client machines run a **simulated approximation** of the world based on the host's
version of events, constantly being updated with corrections.

```
// Host authority flow
function hostProcessTick():
    // 1. Receive control data from all clients
    for each client in clients:
        clientInput = receiveControlData(client)
        applyClientInput(client.playerEntity, clientInput)

    // 2. Run authoritative simulation
    simulateTick(gameState)

    // 3. Process shot events (target-relative)
    for each shot in pendingShots:
        if validateShot(shot):
            applyDamage(shot.target, shot.weapon, shot.bodyPart)

    // 4. Send state updates to all clients (prioritized)
    for each client in clients:
        priorities = calculateAllPriorities(gameState, client)
        statePacket = buildPrioritizedPacket(gameState, priorities, client.bandwidth)
        send(client, statePacket)
```

### Client Prediction and Correction

Clients predict their own movement and actions locally for responsiveness, then
correct when host state arrives:

```
// Client prediction loop
function clientProcessTick():
    // 1. Sample local input
    input = sampleInput()

    // 2. Send control data to host
    sendControlData(host, input)

    // 3. Apply input locally (prediction)
    predictMovement(localPlayer, input)

    // 4. Receive and apply host state
    hostState = receiveStateData()
    if hostState:
        reconcileState(localGameState, hostState)
        // Smoothly correct any prediction errors
        interpolateCorrections()
```

### Host Advantage

The host player experiences inherent advantages:

- **Instant action resolution** -- shots and actions take effect immediately with
  no round-trip delay
- **No prediction errors** -- the host's local state is the authoritative state
- **Immediate grenade/rocket spawning** -- other players must wait for host
  confirmation

### Host Migration

When the host disconnects, the game performs **host migration** to transfer
authority to another player. The system tracks per-console connection performance
using a **host record** that measures:

- Maximum and average throughput
- Connection interruptions
- Historical reliability

The host selection algorithm uses this data to choose the best candidate, aiming to
minimize the latency and disruption for remaining players.

---

## 9. Case Studies: Networking Specific Mechanics

### Grenade Throws

The grenade throw implementation demonstrates the prediction/authority split:

```
// Evolution of grenade throw networking

// V1 (Halo 3 style - unresponsive):
// Client presses throw → waits for host permission → plays animation
// Problem: feels sluggish due to round-trip delay

// V2 (Reach - responsive):
function onGrenadeButtonPress():
    // Client PREDICTS the throw animation immediately
    playThrowAnimation()  // instant feedback

    // Client does NOT predict grenade release
    // Waits for host confirmation before spawning the grenade object
    sendToHost({ type: "GRENADE_THROW", direction: aimDirection })

function onHostGrenadeConfirm(grenadeData):
    // Host spawns the grenade authoritatively
    // Host maintains authority over grenade in flight
    spawnGrenade(grenadeData)
```

The lag becomes imperceptible because the throw animation takes several frames to
play; by the time the arm swing reaches the release point, the host has confirmed
the throw and the grenade appears on time.

### Armor Lock

Armor Lock (an equipment ability granting temporary invulnerability) evolved through
three networking iterations:

**V1:** All animations predicted by clients. Responsive but exploitable -- clients
could activate invulnerability before the host agreed.

**V2:** Client controls animation but waits for host authority to enable
invulnerability. Safer but could feel unresponsive.

**V3 (Shipped):** Client predicts the intro animation immediately on button press.
The host introduces a deliberate **3-frame delay** before activating the
invulnerability shield. This balances responsiveness (the player sees immediate
feedback) with authority (the host controls when damage immunity actually begins).

```
// Armor Lock V3 (shipped)
// Client side:
function onArmorLockActivate():
    playIntroAnimation()  // immediate visual feedback
    sendToHost({ type: "ARMOR_LOCK_ACTIVATE" })
    // Invulnerability NOT active yet on client

// Host side:
function onArmorLockRequest(client):
    // Intentional delay calibrated to measured round-trip time
    scheduleAfterFrames(3, function():
        client.entity.setInvulnerable(true)
        broadcastState(client.entity)  // all clients now see shield
    )
```

### Assassinations

Assassination animations (triggered by a melee from behind) required synchronized
positioning of two players through a multi-second animation sequence.

**V1 (Beta):** Used local prediction of participant positions and orientations.
Failed because animations didn't fit the predicted locations -- the attacker and
victim would end up in different positions on different clients.

**V2 (Shipped):** All peers (including the participants themselves) obey the host
strictly. A camera transition is used to hide the initial latency, and synchronized
state snapshots ensure all clients see the same animation play out identically.

---

## 10. Design Philosophy

Aldridge distilled four core rules for networking gameplay:

1. **Determine which gameplay aspects require consistency.** Not everything needs to
   be perfectly synchronized. Ragdolls, particles, and bullet decals can differ
   between clients. Damage, positions, and game state cannot.

2. **Separate prediction from authority appropriately.** Predict what you can
   (animations, local movement) but let the host be authoritative for what matters
   (damage, game-state-changing events).

3. **Measure before optimizing.** "Measure twice, cut once -- use tools to understand
   the system." The profiling infrastructure was essential to achieving the 80%
   bandwidth reduction.

4. **Design mechanics with networking as a primary constraint.** Networking should
   not be an afterthought. Mechanics should be designed from the start with
   consideration for how they will be communicated over the network.

---

## References

- [GDC Vault -- "I Shot You First: Networking the Gameplay of HALO: REACH"](https://www.gdcvault.com/play/1014345/I-Shot-You-First-Networking) -- The original GDC 2011 talk recording by David Aldridge
- [Wolfire Games Blog -- GDC Session Summary: Halo Networking](http://blog.wolfire.com/2011/03/GDC-Session-Summary-Halo-networking) -- Detailed third-party summary of the talk
- [Bungie Presentation Slides (PPTX)](http://downloads.bungie.net/presentations/David_Aldridge_Programming_Gameplay_Networking_Halo_final_pub_without_video.pptx) -- Original slide deck from Bungie
- [Networked Graphics (UCL) -- Halo: Reach](https://ng.cs.ucl.ac.uk/2011/05/13/halo-reach/) -- Academic summary with bandwidth specifications
- [Halo Networking Guide (Bungie.org)](https://halo.bungie.org/misc/networkguide.html) -- Community guide on Halo networking, matchmaking, and host mechanics
- [Internet Archive -- Full Talk Recording](https://archive.org/details/youtube-h47zZrqjgLc) -- Archived video of the complete GDC presentation
- [SlideToDoc -- Presentation Slides](https://slidetodoc.com/i-shot-you-first-gameplay-networking-in-halo/) -- Slide-by-slide breakdown of the presentation
- [Gamasutra/Game Developer -- Video: Networking the online gameplay of Halo: Reach](https://www.gamedeveloper.com/programming/video-networking-the-online-gameplay-of-i-halo-reach-i-) -- Coverage of the talk on Game Developer
