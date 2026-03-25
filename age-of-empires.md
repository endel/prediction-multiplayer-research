# 1500 Archers on a 28.8: Age of Empires Networking Architecture

## Context

At the 2001 Game Developers Conference (GDC), **Paul Bettner** and **Mark Terrano** of
Ensemble Studios presented "1500 Archers on a 28.8: Network Programming in Age of
Empires and Beyond." The talk described the multiplayer networking architecture behind
*Age of Empires* (1997) and *Age of Empires II: The Age of Kings* (1999), and was later
published as an article in Game Developer magazine (March 2001).

The presentation became one of the most influential talks in the history of real-time
strategy (RTS) game networking, establishing patterns that the genre would follow for
over two decades.

---

## The Problem

Ensemble Studios faced a seemingly impossible constraint: synchronize **1500+ game
units** across **up to 8 players** connected via **28.8 kbps modems** -- the typical
consumer internet connection of the late 1990s.

### Why naive approaches fail

A quick calculation reveals the problem. Even a minimal per-unit state update
containing only X position, Y position, status, action, facing direction, and damage
would require roughly 16-20 bytes per unit. With 1500 units updating at even a modest
rate:

```
1500 units x 20 bytes x 10 updates/sec = 300,000 bytes/sec = 2.4 Mbps
```

A 28.8 kbps modem can transmit roughly 3,600 bytes per second. This means a
traditional state-replication approach would limit the game to approximately **250
moving units at most** -- far too few for the sweeping, epic historical battles that
defined Age of Empires.

The designers wanted players to be able to devastate a Greek city with catapults,
archers, and warriors on one side while it was being besieged from the sea with
triremes. Clearly, another approach was needed.

---

## Deterministic Lockstep Architecture

### The core insight

Instead of transmitting the state of every unit every frame, Ensemble Studios
transmitted only **player commands** (inputs). Each machine ran an identical copy of
the game simulation. As long as every machine processed the same commands in the same
order at the same simulation tick, all machines would produce identical game states --
without ever sending a single unit position across the network.

This is the **deterministic lockstep** model: every peer executes the same
deterministic simulation in lockstep, synchronized by exchanging only the inputs.

### What "deterministic" means in practice

For lockstep to work, the simulation must be **perfectly deterministic**: given the
same initial state and the same sequence of inputs, every machine must produce
bit-identical results. This requirement is deceptively difficult. Sources of
non-determinism include:

- **Floating-point arithmetic**: Different CPU architectures (or even different
  compiler optimization levels) can produce slightly different floating-point results.
  Age of Empires used **fixed-point math** for all gameplay-critical calculations.
- **Random number generators**: All machines must use the same RNG with the same seed,
  advanced the same number of times in the same order.
- **Iteration order**: Hash maps, sets, and other unordered data structures may iterate
  in different orders on different machines. All iteration over game objects must use a
  deterministic ordering.
- **Uninitialized memory**: Any uninitialized variable can diverge across machines.
- **System-dependent behavior**: Pathfinding, AI decisions, and targeting must all
  produce identical results. Even a deer slightly out of alignment on one machine would
  forage slightly differently, leading to a different map state, causing cascading
  divergence.

### What gets transmitted

The only data sent across the network are **player commands** -- high-level actions
such as:

- "Move selected units to position (X, Y)"
- "Attack target unit ID"
- "Build a barracks at position (X, Y)"
- "Research technology T"
- "Set rally point"

A typical command might be 10-20 bytes. Even in a hectic battle, a human player rarely
issues more than a few commands per second. This reduces bandwidth requirements by
orders of magnitude compared to state replication.

---

## The Turn-Based Command System

### Turn structure

The simulation is divided into discrete **turns** (also called communication turns or
network turns), each approximately **200 milliseconds** long. Within each turn:

1. The local player's commands are collected.
2. Those commands are broadcast to all other peers.
3. Commands received from remote peers are buffered for future execution.

### The two-turn execution delay

Commands are **not executed immediately**. A command issued during turn `N` is
scheduled for execution during turn `N + 2`. This two-turn delay (approximately 400ms)
exists for critical reasons:

```
Turn N:     Player issues command, command is sent to all peers
Turn N+1:   Command is in transit / being received and acknowledged
Turn N+2:   All peers execute the command simultaneously
```

**Why the delay is necessary:**

1. **Network transit time**: On a 28.8 kbps modem with typical internet latency of the
   era (100-300ms), a command needs time to physically reach all peers.
2. **Acknowledgment window**: The system needs to confirm that all peers have received
   the commands for a given turn before that turn can be executed. The extra turn
   provides a buffer for retransmission if a packet is lost.
3. **Synchronization guarantee**: Without the delay, fast-connection players would
   execute commands before slow-connection players even received them, breaking
   lockstep.

### Latency perception in RTS games

Ensemble Studios discovered that RTS games are remarkably tolerant of input latency:

- **Under 250ms**: Players did not even notice the delay.
- **250-500ms**: Gameplay remained very playable.
- **Beyond 500ms**: Delay started to become noticeable.

This is in stark contrast to first-person shooters, where even 50ms of input lag is
perceptible. The inherent latency tolerance of the RTS genre made the two-turn delay
acceptable.

---

## Command Buffering and Transmission

### Packet structure

Each network packet contained:

- **Turn identifier**: Which simulation turn these commands belong to.
- **Sequence number**: For ordering and drop detection.
- **Command data**: The serialized player commands for that turn.
- **Speed control data**: A mere **2 bytes** of overhead for the dynamic speed control
  system.
- **Acknowledgments**: Confirmation of received turns from other peers.

### Transport protocol

Age of Empires used **UDP** (via DirectPlay) rather than TCP, implementing its own
reliability layer on top:

- **Client-side ordering**: Packets were tagged with sequence numbers and reordered on
  receipt.
- **Drop detection**: Missing sequence numbers triggered immediate resend requests.
- **Philosophy**: "When in doubt, assume it dropped." The system was aggressive about
  requesting retransmissions rather than waiting.
- **No Nagle's algorithm**: Commands were sent immediately, not buffered at the
  transport level.

### Peer-to-peer topology

The game used a **peer-to-peer** architecture rather than client-server. Every player
sent their commands directly to every other player. With 8 players, each peer
maintained 7 connections. This eliminated the single-point-of-failure problem of a
dedicated server and halved the round-trip time compared to relaying through a host.

---

## Synchronization Verification via Checksums

### The out-of-sync problem

Because the entire architecture depends on determinism, even the slightest divergence
between machines is catastrophic. A single bit difference in one variable will cascade
through the simulation, producing wildly different game states within seconds.

### Checksum system

To detect desynchronization, each machine periodically computed **checksums** over
critical game state:

- **World state checksum**: Covering unit positions, health, status, and other core
  simulation data.
- **Object checksums**: Verifying the state of all game objects.
- **Pathfinding checksums**: Ensuring pathfinding results were identical.
- **Targeting checksums**: Verifying that combat targeting decisions matched.

These checksums were exchanged between peers and compared. If any checksum failed to
match, the game detected an **"Out of Sync"** error.

### Consequences of desync

When a desync was detected, the game would **stop** and notify all players. There was
no graceful recovery -- in a deterministic lockstep system, once machines diverge, it
is impossible to know which machine has the "correct" state (since there is no
authoritative server). The game session would effectively end.

This was a deliberate trade-off: the severity of the consequence ensured that the
development team invested heavily in eliminating all sources of non-determinism.

### Security benefit

The checksum system had an unexpected benefit: it made the game **extremely difficult
to hack**. Any client that modified its simulation state (e.g., giving itself extra
resources or revealing fog of war) would immediately produce a different checksum,
triggering an out-of-sync error. The cheater's game would stop, effectively
self-policing.

---

## Handling Disconnects and Resynchronization

### Waiting for stragglers

If a peer's commands for a given turn had not arrived by the time that turn needed to
execute, the simulation **stalled** -- all machines would pause and wait. This is the
fundamental trade-off of lockstep: the game can only run as fast as the slowest machine
can process the simulation, render the frame, and send out new commands.

### Timeout and dropout

If a peer remained unresponsive beyond a configurable timeout:

1. The game would notify remaining players that a peer had dropped.
2. The missing player's slot would be handled (AI takeover or elimination).
3. The remaining peers would resynchronize and continue.

### No mid-game rejoin

True lockstep architectures make **reconnection extremely difficult**. Since there is
no authoritative state snapshot to send to a reconnecting player, the reconnecting
client would need to either:

- Replay the entire game from the beginning (impractical), or
- Receive a full state dump from another peer (which the architecture was specifically
  designed to avoid).

Age of Empires did not support mid-game reconnection for dropped players.

### Speed control and adaptive turn length

To minimize stalling, Age of Empires implemented a **dynamic speed control** system:

- Each client continuously measured its own **frame rate** and **round-trip ping
  times** to all peers.
- This data was encoded in just **2 bytes** and included in every packet.
- The turn length was dynamically adjusted: if the slowest machine or worst connection
  was struggling, turns would be lengthened to give more time for processing and
  transmission.
- The adjustment was **asymmetric**: the system would "quickly rise to handle Internet
  latency changes, and slowly settle back down." This prevented oscillation -- the
  system was conservative about speeding up but aggressive about slowing down.

---

## Why This Approach Was Chosen Over Alternatives

### Client-server state replication

The traditional approach of having an authoritative server that broadcasts full game
state was ruled out because:

1. **Bandwidth**: As calculated above, replicating even minimal per-unit state for
   1500 units would require ~2.4 Mbps -- nearly 100x the available bandwidth.
2. **Server cost**: In the late 1990s, dedicated game servers were not common for RTS
   games. Hosting would fall on one player's machine, creating an unfair advantage.
3. **Latency**: All commands would need to round-trip through the server, doubling
   effective latency.

### Delta compression / state diffing

Sending only changes (deltas) to game state could reduce bandwidth, but:

1. With 1500 units potentially moving every frame, the delta would still be enormous.
2. Lost packets would require full state retransmission.
3. The complexity of diffing a full simulation state is substantial.

### Why lockstep won

Deterministic lockstep provided:

- **Minimal bandwidth**: Only player commands (a few bytes per turn) needed
  transmission.
- **Perfect consistency**: All machines were guaranteed identical state by construction.
- **Scalable unit count**: Adding more units cost zero additional network bandwidth.
- **Built-in cheat prevention**: Any state modification was immediately detected.

The trade-off was increased input latency and the requirement for perfect determinism,
both of which were acceptable for the RTS genre.

---

## Implementation Pseudo-Code: The Turn System

```
// Constants
TURN_DURATION_MS = 200
COMMAND_DELAY_TURNS = 2

// State
current_turn = 0
local_command_buffer = {}      // commands indexed by scheduled turn
remote_command_buffers = {}    // per-peer command buffers indexed by scheduled turn

function game_loop():
    while game_is_running:
        // 1. Collect local input
        commands = collect_player_input()
        scheduled_turn = current_turn + COMMAND_DELAY_TURNS

        // Store locally
        local_command_buffer[scheduled_turn] = commands

        // Broadcast to all peers
        send_to_all_peers(Packet{
            turn: scheduled_turn,
            sequence: next_sequence_number(),
            commands: commands,
            speed_control: encode_speed_data()   // 2 bytes
        })

        // 2. Check if we can advance the simulation
        if can_advance(current_turn):
            execute_turn(current_turn)
            current_turn += 1
        else:
            // Stall: waiting for commands from a peer
            display_waiting_indicator()
            request_resend_from_missing_peers(current_turn)

        // 3. Render the current state (rendering is decoupled from sim)
        render()

function can_advance(turn):
    // We need commands from ALL peers for this turn
    for each peer in connected_peers:
        if turn not in remote_command_buffers[peer]:
            return false
    return true

function execute_turn(turn):
    // Gather all commands for this turn, in deterministic order
    all_commands = []
    all_commands.append(local_command_buffer[turn])

    // Process peers in deterministic order (e.g., by player ID)
    for each peer in sorted(connected_peers, by=player_id):
        all_commands.append(remote_command_buffers[peer][turn])

    // Execute all commands in the simulation
    for each command in flatten(all_commands):
        simulation.execute(command)

    // Advance the simulation by one turn's worth of game time
    simulation.step(TURN_DURATION_MS)

    // Periodic sync verification
    if turn % CHECKSUM_INTERVAL == 0:
        checksum = compute_checksum(simulation.state)
        send_checksum_to_all_peers(turn, checksum)

    // Clean up old buffers
    cleanup_buffers(turn)

function on_receive_packet(packet, from_peer):
    // Handle out-of-order delivery
    if packet.sequence <= last_received_sequence[from_peer]:
        return  // duplicate, ignore

    if packet.sequence > last_received_sequence[from_peer] + 1:
        // Gap detected -- request resend of missing packets
        request_resend(from_peer, last_received_sequence[from_peer] + 1)

    last_received_sequence[from_peer] = packet.sequence
    remote_command_buffers[from_peer][packet.turn] = packet.commands

    // Process speed control data
    update_speed_control(from_peer, packet.speed_control)

function on_receive_checksum(turn, remote_checksum, from_peer):
    local_checksum = stored_checksums[turn]
    if local_checksum != remote_checksum:
        trigger_out_of_sync_error(turn, from_peer)
```

### Timing diagram

```
Time -------->

Player A:  [Turn 0: Cmd]--->[Turn 1: Cmd]--->[Turn 2: Cmd]--->[Turn 3: Cmd]--->
                |                 |                |                |
           Send Cmd(T=2)    Send Cmd(T=3)    Send Cmd(T=4)    Send Cmd(T=5)
                |                 |                |                |
Player B:  [Turn 0: Cmd]--->[Turn 1: Cmd]--->[Turn 2: Cmd]--->[Turn 3: Cmd]--->
                |                 |                |                |
           Send Cmd(T=2)    Send Cmd(T=3)    Send Cmd(T=4)    Send Cmd(T=5)

At Turn 2:
  - Both players execute commands that were issued at Turn 0
  - Both players have had 2 full turns (~400ms) to receive those commands

At Turn 3:
  - Both players execute commands from Turn 1
  - And so on...
```

---

## Influence on Other RTS Games

The deterministic lockstep model pioneered (and popularized) by Age of Empires became
the **dominant networking architecture for RTS games** for over two decades.

### StarCraft (1998) and StarCraft II (2010)

Blizzard's *StarCraft* used an almost identical deterministic lockstep approach. The
game transmitted only player commands and relied on identical simulations across all
peers. *StarCraft II* continued this tradition, using lockstep with a similar turn
structure. The replay system in both games -- which stores only the sequence of player
inputs and can perfectly reproduce an entire match -- is a direct consequence of the
deterministic lockstep design.

### Command & Conquer series

Westwood Studios' *Command & Conquer* series (including *Red Alert* and *Generals*)
also used deterministic lockstep. The architecture was so similar that it likely
represented independent convergent design driven by the same constraints: many units,
low bandwidth, RTS input latency tolerance.

### Supreme Commander (2007)

Gas Powered Games' *Supreme Commander* pushed lockstep to its limits with thousands of
units and projectiles. The game's networking was explicitly inspired by the Age of
Empires model and maintained deterministic lockstep even at an extreme scale.

### Factorio (2020)

The multiplayer architecture of *Factorio* uses deterministic lockstep to synchronize
factories with millions of entities -- a modern descendant of the same approach,
proving its viability even decades later.

### DOTA / MOBA games

The lockstep model also influenced early MOBA games. The original *Defense of the
Ancients* (DotA) inherited its networking from the *Warcraft III* engine, which itself
used deterministic lockstep. Later standalone MOBAs like *League of Legends* moved to
client-server architectures to support features like reconnection and spectating, but
the lineage is clear.

### Replay systems

One of the most lasting legacies of the lockstep model is the **replay file format**.
Because only player inputs are needed to reproduce an entire game, replay files are
extraordinarily small (often under 100 KB for a full match). This format was pioneered
by these early RTS games and remains standard in the genre.

---

## Summary of Trade-Offs

| Aspect | Deterministic Lockstep | Client-Server State Replication |
|---|---|---|
| Bandwidth | Minimal (commands only) | High (state updates for all entities) |
| Unit count scaling | Free (no per-unit network cost) | Linear cost per unit |
| Input latency | Higher (turn delay) | Lower (immediate local prediction) |
| Cheat resistance | Very high (checksum verification) | Moderate (server authoritative) |
| Reconnection | Very difficult / unsupported | Straightforward (send current state) |
| Spectating | Difficult in real-time | Straightforward |
| Determinism requirement | Absolute (bugs cause desync) | None |
| Implementation complexity | High (determinism discipline) | Moderate |
| Suitable genres | RTS, turn-based | FPS, action, RPG |

---

## References

1. **Bettner, Paul and Terrano, Mark.** "1500 Archers on a 28.8: Network Programming
   in Age of Empires and Beyond." GDC 2001 / Game Developer Magazine, March 2001.
   - Article: [https://www.gamedeveloper.com/programming/1500-archers-on-a-28-8-network-programming-in-age-of-empires-and-beyond](https://www.gamedeveloper.com/programming/1500-archers-on-a-28-8-network-programming-in-age-of-empires-and-beyond)
   - PDF (Yale CS538 mirror): [https://zoo.cs.yale.edu/classes/cs538/readings/papers/terrano_1500arch.pdf](https://zoo.cs.yale.edu/classes/cs538/readings/papers/terrano_1500arch.pdf)

2. **Terrano, Mark and Bettner, Paul.** "1500 Archers on a 28.8: Network Programming
   in Age of Empires and Beyond." Proceedings of GDC 2001.
   - GDC Vault: [https://www.gdcvault.com/](https://www.gdcvault.com/)

3. **Fiedler, Glenn.** "Deterministic Lockstep." Gaffer On Games, 2014.
   - [https://gafferongames.com/post/deterministic_lockstep/](https://gafferongames.com/post/deterministic_lockstep/)

4. **Fiedler, Glenn.** "Networked Physics." Gaffer On Games.
   - [https://gafferongames.com/categories/networked-physics/](https://gafferongames.com/categories/networked-physics/)

5. **Peeters, Christophe.** "Lockstep Multiplayer in RTS Games." 2018.
   - [https://www.codeofhonor.com/blog/the-starcraft-path-finding-hack](https://www.codeofhonor.com/blog/the-starcraft-path-finding-hack)

6. **Browning, Patrick.** "Lessons from the StarCraft Engine." Code of Honor blog.
   - [https://www.codeofhonor.com/blog/](https://www.codeofhonor.com/blog/)

7. **Wikipedia.** "Lockstep protocol."
   - [https://en.wikipedia.org/wiki/Lockstep_protocol](https://en.wikipedia.org/wiki/Lockstep_protocol)
