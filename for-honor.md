# Back to the Future: Deterministic Simulation and "Time Travel" in For Honor

## Context

At GDC 2019, **Jennifer Henry**, Gameplay Programmer at Ubisoft Montreal, presented
"Back to the Future! Working with Deterministic Simulation in *For Honor*." The talk
provided a deep dive into the non-authoritative, peer-to-peer architecture behind the
game -- a variation on deterministic lockstep simulation that uses a mechanism the team
called **"time travel"** to maintain responsiveness without sacrificing correctness.

Two years earlier, at GDC 2017, **Frederic Doll** and **Xavier Guilbeault** (also
Ubisoft Montreal) presented "Deterministic vs. Replicated AI: Building the Battlefield
of *For Honor*," which focused on how the team created a believable large-scale
battlefield with dozens of AI-driven NPCs on the backbone of the same distributed
deterministic simulation -- all perfectly synchronized between peers with **zero
network traffic for state**.

Together, these two talks document one of the most ambitious uses of deterministic
lockstep in a modern action game: 8 human players, up to 100 NPCs, and a precision
fighting system that demands frame-perfect fairness, all running on a peer-to-peer
network where only player inputs are transmitted.

---

## The Problem

*For Honor* is a melee combat game where timing, positioning, and reaction speed are
everything. A single guard-break or parry can decide a fight. The game gathers **8
players** on a battlefield alongside **up to 100 NPCs** and numerous interactive
objects (ballistas, catapults, environmental hazards, capture zones).

### Why traditional approaches fall short

A **client-server state replication** model would require transmitting positions,
animations, health, stamina, guard stances, and status effects for every actor on the
battlefield -- potentially hundreds of entities -- at a high enough tick rate to
preserve the precision of the fight system. The bandwidth cost would be enormous, and
any authority delay would undermine the game's responsiveness.

A naive **peer-to-peer host** model would give one player a latency advantage in a
game where a single frame of advantage can mean landing or missing a 400ms light
attack. This is unacceptable in a competitive fighting game.

The team needed an architecture that:

- Keeps bandwidth consumption extremely low regardless of actor count
- Preserves **frame-perfect fairness** (no host advantage)
- Maintains **fluid, responsive** controls with no perceptible input lag
- Scales to 100+ interactive entities without proportional network cost

---

## Deterministic Lockstep Architecture

### The core approach

*For Honor* uses **deterministic lockstep simulation**: every peer runs an identical
copy of the full game simulation locally. Only **player inputs** are transmitted over
the network. Since all peers process the same inputs in the same order at the same
simulation tick, they all arrive at the same game state -- without ever sending a
single position, health value, or animation state across the wire.

```
Traditional Model:
  Server ---> [position, health, animation, status...] ---> Clients
  Bandwidth ~ O(number_of_entities)

For Honor Model:
  Peer A ---> [input: guard_left, heavy_attack] ---> Peers B, C, D, E, F, G, H
  Bandwidth ~ O(number_of_players)   (independent of entity count!)
```

### What gets transmitted

Only player inputs are sent: button presses, guard stance changes, movement direction,
and action commands. The entire simulation state -- every NPC behavior, every physics
interaction, every health change -- is computed locally on each peer from those inputs.

This means that **100 NPCs fighting on the battlefield cost exactly zero network
bandwidth**. Their behavior is fully derived from the deterministic simulation running
on each peer.

### No central authority

Unlike a traditional client-server model, there is **no authoritative server** running
the simulation. Every peer is equal; every peer computes the full simulation. At
launch, *For Honor* used a "session host" that handled matchmaking, invites, and
handshakes, but during gameplay every player ran a fully synchronized simulation with
no game host -- all players sent to all players what they were doing, and no answers
from other players were needed, thanks to the deterministic simulation.

```
Peer Topology (mesh):

    A <---> B
    ^       ^
    |       |
    v       v
    C <---> D

  Every peer sends inputs to every other peer.
  Every peer runs the full simulation independently.
  All peers arrive at the same state.
```

---

## The "Time Travel" Mechanism

### The responsiveness problem with lockstep

In classic lockstep, the simulation cannot advance until inputs from **all** peers have
been received. With 8 players connected over the internet, this means waiting for the
slowest connection before any peer can step forward. The result is perceptible input
lag -- unacceptable in a fighting game where attacks can be as fast as 400ms.

### How For Honor solves it

*For Honor* uses a technique the team calls **"time travel"** -- conceptually similar
to rollback netcode, but applied within a lockstep framework.

Each player maintains a **rolling history buffer of 5 seconds** of input data. The
local simulation runs ahead immediately using locally available inputs, providing
instant responsiveness. When a delayed input arrives from a remote peer, the simulation
**rolls back** to the tick where that input should have been applied and
**resimulates forward** from that point to the present, producing the corrected state.

```
Time Travel Conceptual Flow:

  Frame 100: Local peer is simulating normally
  Frame 105: Remote input arrives for frame 100 (5 frames late)

  1. Save current state
  2. Roll back simulation to frame 100
  3. Insert the newly received remote input at frame 100
  4. Resimulate frames 100 -> 101 -> 102 -> 103 -> 104 -> 105
  5. The corrected state is now the authoritative state
  6. Render the corrected frame 105
```

### The performance cost: 8x gameplay in a single frame

Because the game may need to resimulate multiple frames of gameplay when late inputs
arrive, and because there are up to 8 players whose inputs may arrive at different
times, the simulation must be fast enough to run **up to 8 times the gameplay within a
single render frame**. Jennifer Henry described the "drastic optimizations" needed to
achieve this as one of the team's biggest challenges.

Input is processed at a rate of **8x per frame**, with physics stepping at intervals as
small as **0.5ms**.

```
Performance Budget (conceptual):

  Target frame time: 16.67ms (60 FPS)
  Max resimulation depth: ~5 seconds of history
  Simulation must complete 8 gameplay ticks per render frame

  Per-tick budget: 16.67ms / 8 = ~2.08ms per simulation tick
  Physics step: 0.5ms
  Remaining for game logic, AI, etc.: ~1.5ms per tick
```

### Why "time travel" and not "rollback"

The team chose the metaphor "time travel" rather than "rollback" because the mechanism
is not limited to correcting mispredictions. The system literally travels back in time,
replays the simulation with new information, and fast-forwards to the present. The
entire 5-second window is available for this purpose, and any input arriving within
that window can be incorporated without stalling the game.

---

## Pseudo-code: Deterministic Lockstep with Time Travel

```python
class DeterministicSimulation:
    HISTORY_SECONDS = 5
    TICKS_PER_SECOND = 30   # example tick rate
    MAX_HISTORY = HISTORY_SECONDS * TICKS_PER_SECOND  # 150 ticks

    def __init__(self, num_peers):
        self.current_tick = 0
        self.num_peers = num_peers

        # Ring buffer of simulation snapshots (for rollback)
        self.state_history = {}        # tick -> serialized state
        self.input_history = {}        # tick -> { peer_id: input }
        self.confirmed_tick = 0        # latest tick with all inputs confirmed

    def local_update(self, local_input):
        """Called each frame by the local peer."""
        # 1. Record local input
        self.record_input(self.current_tick, local_peer_id, local_input)

        # 2. Broadcast local input to all peers
        network.broadcast(InputMessage(
            peer_id=local_peer_id,
            tick=self.current_tick,
            input=local_input
        ))

        # 3. Advance simulation (may use predicted inputs for remote peers)
        self.step_simulation(self.current_tick)
        self.current_tick += 1

    def on_remote_input_received(self, peer_id, tick, remote_input):
        """Called when a remote peer's input arrives (possibly late)."""
        if tick < self.current_tick - self.MAX_HISTORY:
            # Too old to resimulate -- drop or handle error
            return

        # Record the remote input
        self.record_input(tick, peer_id, remote_input)

        if tick < self.current_tick:
            # ---- TIME TRAVEL ----
            # This input arrived late. We must resimulate from that tick.
            self.rollback_to(tick)
            for t in range(tick, self.current_tick):
                self.step_simulation(t)

    def step_simulation(self, tick):
        """Run one tick of the deterministic simulation."""
        inputs = self.gather_inputs(tick)
        # Deterministic update: same inputs -> same result on all peers
        game_state.deterministic_update(inputs)
        # Save snapshot for potential future rollback
        self.state_history[tick] = game_state.serialize()

    def rollback_to(self, tick):
        """Restore simulation state to a previous tick."""
        game_state.deserialize(self.state_history[tick])

    def record_input(self, tick, peer_id, input_data):
        if tick not in self.input_history:
            self.input_history[tick] = {}
        self.input_history[tick][peer_id] = input_data

    def gather_inputs(self, tick):
        """
        Gather inputs for all peers at this tick.
        For missing remote inputs, use last known input (prediction).
        """
        inputs = {}
        for peer in range(self.num_peers):
            if tick in self.input_history and peer in self.input_history[tick]:
                inputs[peer] = self.input_history[tick][peer]
            else:
                # Predict: carry forward last known input
                inputs[peer] = self.get_last_known_input(peer, tick)
        return inputs
```

---

## Determinism Requirements

### Why strict determinism is essential

In a system where no state is transmitted -- only inputs -- every peer must produce
**bit-identical results** from the same inputs. Even a single bit of divergence will
compound over time, causing the peers to see entirely different game states (a
"desync"). Jennifer Henry emphasized: **"Don't underestimate desync."**

### Sources of non-determinism in For Honor

The team identified several treacherous sources of non-determinism:

| Source | Problem | Mitigation |
|--------|---------|------------|
| **Floating-point arithmetic** | AMD and Intel CPUs produce different results for the same operations | Controlled floating-point operations; careful compiler settings; avoiding platform-specific optimizations |
| **Multithreading** | Thread scheduling varies between machines, causing different execution orders | Simulation logic runs on a single deterministic thread; only rendering/audio are multithreaded |
| **Random number generators** | Different RNG sequences break determinism | Shared seeded RNG; all peers use same seed, advance same number of times in same order |
| **Uninitialized memory** | Random values from uninitialized variables | Strict initialization policies; static analysis tools |
| **Iteration order** | Hash maps, sets may iterate differently per platform | Deterministic data structures with guaranteed iteration order |
| **Third-party libraries** | Physics engines, animation systems may not be deterministic | Custom or heavily modified libraries; careful integration testing |

### Desync detection and recovery

The team implemented a multi-layered approach to handling desyncs:

1. **Checksum verification**: Peers periodically compute and exchange checksums of
   their simulation state to detect divergence early.
2. **Debug mode**: Desyncs are automatically reported to Jira with full state snapshots
   for developer investigation.
3. **Production recovery**: When a desync is detected in a live game, the system
   attempts one of three responses:
   - **Recovery**: Attempt to resynchronize the diverging peer
   - **Peer removal**: Kick the desynchronized peer from the session
   - **Session disband**: If recovery is impossible, end the entire session

```
Desync Detection (conceptual):

  Every N ticks:
    local_checksum = hash(game_state.serialize())
    broadcast(ChecksumMessage(tick=current_tick, checksum=local_checksum))

  On receiving remote checksum:
    if remote_checksum != local_checksum:
      trigger_desync_handling(remote_peer, tick)
```

---

## Deterministic AI: Building the Battlefield (GDC 2017)

### The challenge of 100 NPCs

*For Honor*'s Dominion mode features large-scale battles with up to 100 AI-controlled
"minions" fighting alongside the 8 human players. In a traditional networked game, each
of these NPCs would need its position, health, animation, and AI state replicated
across the network. With deterministic lockstep, **none of this data is transmitted**.

Every NPC's behavior is fully computed from the deterministic simulation. If all peers
run the same AI logic with the same inputs, all NPCs behave identically on all peers.

### Deterministic vs. replicated AI

The 2017 GDC talk by Doll and Guilbeault drew a clear distinction:

- **Replicated AI** (traditional): AI decisions are made on one machine (the server
  or host) and the results are sent to all clients. This requires bandwidth
  proportional to the number of AI actors and introduces authority-dependent latency.

- **Deterministic AI** (*For Honor*'s approach): AI decisions are made independently
  on every peer using the same deterministic logic. No AI state is transmitted. The
  only requirement is that the AI logic itself is deterministic.

```
Replicated AI:
  Host computes: NPC_42.decide() -> "attack player_3"
  Host sends: [NPC_42, action=attack, target=player_3] -> all clients
  Cost: O(num_NPCs) network messages

Deterministic AI:
  Every peer computes: NPC_42.decide() -> "attack player_3"  (same result everywhere)
  Network sends: nothing
  Cost: O(1) -- zero network cost for AI
```

### How the AI system works

The AI system was built on several key principles:

1. **Metadata-rich world**: The battlefield is annotated with spatial metadata --
   navigation data, zone ownership, strategic points, threat levels -- that AI agents
   query deterministically.

2. **Simple rules, emergent behavior**: Rather than complex individual behavior trees,
   designers were given "levers" to control many actors at once. Complex and interesting
   battlefield behaviors emerged from simple, elegant rules applied across groups.

3. **Layered decision-making**:
   - **High-level objectives**: Capture zones, push lanes, defend positions
   - **Reactive decisions**: Respond to nearby threats, support allies, retreat when
     outnumbered
   - **Environmental awareness**: Use choke points, avoid hazards, coordinate with
     other NPCs

4. **Extracted fight logic**: The AI's combat decisions used data extracted from
   Ubisoft's fight logic system, ensuring NPC combat behavior matched the rules of
   the fight system precisely and deterministically.

5. **Designer tools and automated testing**: AI designers had dedicated tools to tune
   NPC behavior, and automated testing validated that design intentions were met --
   critically, that behavior remained deterministic across platforms and builds.

---

## Fairness and Precision in the Fight System

### Why fairness demands deterministic lockstep

*For Honor*'s fight system is built around **reactable attacks with tight timing
windows**. A light attack might have a 400-500ms startup, and the defender has a narrow
window to block, parry, or dodge. In this context:

- **Host advantage** (even 30-50ms) would make attacks unreactable for non-host players
- **State replication delay** would mean defenders see attacks later than attackers
  intend them, breaking the fairness contract

Deterministic lockstep eliminates host advantage entirely: there is no host. Every peer
computes the same simulation independently. When a conflict arises (e.g., two players
attack at the exact same frame), the deterministic simulation resolves it identically
on all peers.

### Frame-perfect synchronization

Because the simulation is deterministic and all peers process the same inputs at the
same tick, combat interactions are resolved with **frame-perfect precision**:

- If Player A's heavy attack lands on tick 450 and Player B's parry input arrives on
  tick 449, the parry succeeds on all peers identically.
- There is no "I parried on my screen but it didn't register" -- the inputs determine
  the outcome, and all peers see the same inputs at the same ticks.

---

## Challenges of Peer-to-Peer

### Cheating

P2P architectures are inherently more vulnerable to cheating than client-server models:

- **Input manipulation**: A malicious peer could inject false inputs or modify their
  local simulation to gain information (e.g., seeing through fog of war).
- **Traffic manipulation**: Players could throttle or manipulate their network traffic
  to gain timing advantages ("lag switching").
- **No server-side validation**: Without an authoritative server, there is no single
  point that can detect and reject invalid inputs.

The deterministic simulation provides *some* protection: if a peer's state diverges
from the others (due to cheating), the checksum verification system will detect the
desync. However, this is reactive rather than preventive.

### Disconnects and session stability

P2P was *For Honor*'s most criticized aspect at launch. Key problems included:

- **Host migration**: When the session host left, the game had to migrate the session
  to another peer. This process frequently failed, disconnecting all remaining players.
- **NAT traversal**: Players behind restrictive NATs could not connect to each other,
  limiting the matchmaking pool.
- **Cascade disconnects**: One player's disconnection could destabilize the session,
  causing additional disconnects as the mesh topology reconfigured.
- **Resynchronization pauses**: When peers fell out of sync, the game would pause for
  all players while attempting to resynchronize -- a jarring experience in a fast-paced
  combat game.

### Latency dependency

In a mesh P2P topology, the overall experience is bounded by the **worst connection**
in the session. If one of 8 players has a poor connection, all players experience
degraded performance (either through increased resimulation or more frequent rollbacks).

---

## Migration to Dedicated Servers

### Timeline

- **February 2017**: *For Honor* launches with P2P networking
- **August 2017**: Ubisoft announces dedicated servers are in development following
  sustained community criticism
- **February 19, 2018**: Dedicated servers launch for PC (Season 5: "Age of Wolves")
- **March 6, 2018**: Dedicated servers launch for PS4 and Xbox One

### Architecture change

The dedicated servers were hosted on **Amazon GameLift** and designed to be
**lightweight**. Critically, the move to dedicated servers did **not** abandon the
deterministic simulation model. Instead, the dedicated server replaced the P2P mesh
topology and session host:

```
Before (P2P Mesh):
  All peers send inputs to all peers directly
  Session host manages handshakes/invites only

After (Dedicated Server):
  All peers send inputs to the dedicated server
  Server relays inputs to all peers
  Server runs the simulation for validation
  Server handles session management, join/leave
```

The server was designed to be lightweight enough to run at **150-200 FPS**, handling
input relay and session management. The server also runs its own copy of the simulation
for state awareness and validation purposes, while each client continues to run the
full deterministic simulation locally.

### What improved

| Problem | P2P | Dedicated Server |
|---------|-----|-----------------|
| Host migration | Frequent crashes and disconnects | Eliminated entirely |
| NAT traversal | Blocked many players | No longer required |
| Session stability | Fragile mesh topology | Centralized, stable |
| Matchmaking | Limited by NAT compatibility | Region-based, lower latency |
| Resync pauses | Common | Largely eliminated |
| Match completion rate | Low in 4v4 modes | Significantly improved |
| Cheating surface | Broad (no authority) | Reduced (server can validate) |

### What stayed the same

The fundamental architecture -- deterministic lockstep with time travel -- remained
intact. Clients still run the full simulation locally, only inputs are transmitted, and
the time travel mechanism still handles late-arriving inputs. The dedicated server acts
primarily as a **relay and validator** rather than a traditional authoritative server.

---

## Implementation Concepts

### State serialization for rollback

Time travel requires the ability to **save and restore** the complete simulation state
efficiently. This is one of the most technically demanding aspects of the architecture:

```python
class GameState:
    def serialize(self):
        """
        Serialize the ENTIRE simulation state to a byte buffer.
        Must capture every piece of mutable state:
        - All entity positions, velocities, rotations
        - All health, stamina, status effects
        - All AI state machines and their current states
        - All animation states and blend weights
        - All physics body states
        - RNG state
        - Game mode state (capture points, scores, timers)
        """
        buffer = ByteBuffer()
        for entity in self.entities:
            entity.serialize(buffer)
        buffer.write(self.rng.get_state())
        buffer.write(self.game_mode.serialize())
        return buffer

    def deserialize(self, buffer):
        """Restore simulation to a previous state for resimulation."""
        for entity in self.entities:
            entity.deserialize(buffer)
        self.rng.set_state(buffer.read_rng_state())
        self.game_mode.deserialize(buffer)
```

### Deterministic update loop

```python
def deterministic_update(game_state, inputs_for_tick):
    """
    A single tick of the deterministic simulation.
    MUST produce identical results on all peers given identical inputs.
    """
    # 1. Process player inputs (deterministic order: by peer ID)
    for peer_id in sorted(inputs_for_tick.keys()):
        player = game_state.get_player(peer_id)
        player.apply_input(inputs_for_tick[peer_id])

    # 2. Update fight system (attack resolution, hit detection)
    game_state.fight_system.update()

    # 3. Update AI (deterministic decisions for all NPCs)
    for npc in game_state.npcs:  # deterministic iteration order
        npc.ai.update(game_state)

    # 4. Step physics (deterministic physics engine)
    game_state.physics.step(FIXED_TIMESTEP)

    # 5. Update game mode (capture points, scoring)
    game_state.game_mode.update()

    # 6. Periodic checksum for desync detection
    if game_state.tick % CHECKSUM_INTERVAL == 0:
        checksum = hash(game_state.serialize())
        network.broadcast(ChecksumMessage(game_state.tick, checksum))

    game_state.tick += 1
```

### Input prediction for time travel

When a remote peer's input has not yet arrived, the simulation must still advance. The
simplest and most common prediction strategy is to **carry forward the last known
input**:

```python
def predict_input(peer_id, tick):
    """
    Predict a remote peer's input when it hasn't arrived yet.
    Carrying forward the last known input minimizes visual disruption:
    - If a player was moving left, they probably still are
    - If a player was idle, they probably still are
    """
    for t in range(tick - 1, tick - MAX_HISTORY, -1):
        if t in input_history and peer_id in input_history[t]:
            return input_history[t][peer_id]
    return Input.NEUTRAL  # default: no input
```

### Managing the resimulation budget

```python
class ResimulationManager:
    MAX_RESIM_TICKS = 150  # 5 seconds at 30 ticks/sec
    FRAME_BUDGET_MS = 16.67  # 60 FPS target

    def process_late_input(self, peer_id, tick, input_data):
        resim_depth = self.current_tick - tick

        if resim_depth > self.MAX_RESIM_TICKS:
            # Input too old -- cannot resimulate that far back
            self.handle_desync(peer_id)
            return

        if resim_depth * self.per_tick_cost_ms > self.FRAME_BUDGET_MS:
            # Resimulation would blow the frame budget
            # Spread across multiple frames or accept a hitch
            self.schedule_distributed_resim(tick, self.current_tick)
            return

        # Perform resimulation
        self.rollback_to(tick)
        self.input_history.insert(tick, peer_id, input_data)
        for t in range(tick, self.current_tick):
            self.step_simulation(t)
```

---

## Key Takeaways

1. **Bandwidth scales with players, not entities.** By transmitting only inputs and
   computing everything else deterministically, *For Honor* supports 100+ battlefield
   actors with the same bandwidth as a 1v1 duel.

2. **Time travel enables responsiveness in lockstep.** Classic lockstep forces the
   simulation to wait for all inputs. *For Honor*'s time travel approach lets the
   simulation run ahead and correct retroactively, preserving the feel of instant
   input response.

3. **Determinism is brutally hard.** Floating-point differences between AMD and Intel,
   multithreading, uninitialized memory, and iteration order are all potential sources
   of desync. The team spent 6+ years learning to manage this, and desync remained a
   constant battle.

4. **Performance is non-negotiable.** The ability to resimulate up to 8x the gameplay
   in a single frame requires extreme optimization of every system -- physics, AI,
   animation, game logic.

5. **P2P has real costs.** Despite the elegance of the deterministic model, the P2P
   topology caused severe player-facing problems (disconnects, NAT issues, cheating
   vulnerability) that ultimately drove the migration to dedicated servers.

6. **Dedicated servers can augment lockstep.** *For Honor* proved that moving to
   dedicated servers does not require abandoning deterministic lockstep. The server
   can act as a relay and validator while clients continue to run the full simulation.

---

## References

- [GDC Vault -- "Back to the Future! Working with Deterministic Simulation in *For Honor*" (Jennifer Henry, GDC 2019)](https://www.gdcvault.com/play/1026322/Back-to-the-Future-Working)
- [GDC 2019 Presentation Slides (PDF)](https://media.gdcvault.com/gdc2019/presentations/Henry_Jennifer_BackToTheFuture.pdf)
- [GDC Vault -- "Deterministic vs. Replicated AI: Building the Battlefield of *For Honor*" (Frederic Doll & Xavier Guilbeault, GDC 2017)](https://www.gdcvault.com/play/1024454/Deterministic-vs-Replicated-AI-Building)
- [Game Developer -- "Come to GDC and see how *For Honor*'s multiplayer is built on time travel"](https://www.gamedeveloper.com/programming/come-to-gdc-and-see-how-i-for-honor-i-s-multiplayer-is-built-on-time-travel)
- [Game Developer -- "Video: Deterministic vs. replicated AI: Building *For Honor*'s battlefield"](https://www.gamedeveloper.com/design/video-deterministic-vs-replicated-ai-building-i-for-honor-i-s-battlefield)
- [Ubisoft -- "For Honor Now on Dedicated Servers on All Platforms"](https://www.ubisoft.com/en-us/game/for-honor/news-updates/2HayRoZjbJzSEJAhJMpeF7/for-honor-now-on-dedicated-servers-on-all-platforms)
- [AWS GameTech Blog -- "Moving from P2P to Cloud: How *For Honor* & *Friday the 13th* Move from P2P to the Cloud"](https://aws.amazon.com/blogs/gametech/for-honor-friday-the-13th-the-game-move-from-p2p-to-the-cloud-to-improve-player-experience/)
- [SlideShare -- "MIGS18: Transforming from Peer-to-Peer to Dedicated Servers on a Live Game"](https://www.slideshare.net/LaurentPointCa/migs18-transforming-from-peertopeer-to-dedicated-servers-on-a-live-game)
- [PCGamesN -- "For Honor uses peer-to-peer networking, but Ubisoft say 'no player will have an advantage'"](https://www.pcgamesn.com/for-honor/for-honor-multiplayer-connection-peer-to-peer)
- [Kotaku -- "For Honor's Dedicated Servers Make A Big Difference"](https://kotaku.com/for-honors-dedicated-servers-make-a-big-difference-1823559981)
- [AVA Blog -- GDC 2019 Part Three (session notes)](https://grrava.blogspot.com/2019/04/gdc-2019-part-three.html)
- [SnapNet -- "Netcode Architectures Part 1: Lockstep"](https://www.snapnet.dev/blog/netcode-architectures-part-1-lockstep/)
- [SnapNet -- "Netcode Architectures Part 2: Rollback"](https://www.snapnet.dev/blog/netcode-architectures-part-2-rollback/)
