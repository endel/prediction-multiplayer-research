# Deterministic Lockstep


---

## Table of Contents

1. [Overview](#1-overview)
2. [History & Origins](#2-history--origins)
3. [How It Works Step by Step](#3-how-it-works-step-by-step)
4. [The Determinism Requirement](#4-the-determinism-requirement)
5. [Achieving Determinism](#5-achieving-determinism)
6. [Turn-Based Input Collection](#6-turn-based-input-collection)
7. [Handling Slow Peers](#7-handling-slow-peers)
8. [Desync Detection](#8-desync-detection)
9. [Lockstep + Rollback Hybrid](#9-lockstep--rollback-hybrid)
10. [Bandwidth Efficiency](#10-bandwidth-efficiency)
11. [Common Pitfalls](#11-common-pitfalls)
12. [Pseudo-Code / Code Examples](#12-pseudo-code--code-examples)
13. [Modern Relevance](#13-modern-relevance)
14. [When to Use / When Not to Use](#14-when-to-use--when-not-to-use)
15. [References](#15-references)

---

## 1. Overview

### What It Is

Deterministic lockstep is a method of networking a multiplayer game by sending **only the player inputs** that control the simulation rather than the state of objects in the game world. Every peer runs an identical copy of the game simulation. Given the same initial state and the same sequence of inputs, every machine produces exactly the same result -- frame by frame, bit for bit.

As Glenn Fiedler describes it:

> "Deterministic lockstep is a method of networking a system from one computer to another by sending only the inputs that control that system, rather than the state of that system."
>
> -- [Deterministic Lockstep, Gaffer on Games](https://gafferongames.com/post/deterministic_lockstep/)

### The Problem It Solves

Consider an RTS game like Age of Empires with 1,500 units on screen. If you were to synchronize the game by sending each unit's position, velocity, health, and action state, even a minimal representation (X, Y, status, action, facing, damage) would limit the game to roughly 250 moving units over a 28.8 kbps modem. That is nowhere near enough for a large-scale RTS.

Deterministic lockstep solves this by inverting the approach: instead of replicating state, replicate the **inputs that produce the state**. A player might issue only a few commands per second. If every machine starts from the same initial state and processes the same inputs in the same order at the same time, every machine will arrive at the same resulting state -- no matter how many units, particles, or entities exist in the simulation.

The key insight is that **bandwidth is proportional to the number of players and the size of their inputs, not the number of objects in the simulation.**

### Core Properties

| Property | Description |
|---|---|
| **Synchronized** | All peers advance in lockstep; no peer is ahead or behind |
| **Deterministic** | Given identical inputs and initial state, every peer produces identical output |
| **Input-based** | Only player commands are sent over the network |
| **Bandwidth-efficient** | Network traffic scales with player count, not entity count |
| **Latency-visible** | Players experience input delay equal to the network round-trip plus buffering |

---

## 2. History & Origins

### SIMNET (1983--1990)

The earliest large-scale networked simulation was **SIMNET**, developed by DARPA (then ARPA) in partnership with the US Army between 1983 and 1990. Developed by Bolt, Beranek and Newman (BBN), Delta Graphics, and Perceptronics, SIMNET connected dozens of vehicle simulators across wide-area networks for military training.

Notably, SIMNET did **not** use deterministic lockstep. It relied on **dead reckoning** -- each simulator broadcast position updates, and remote nodes predicted entity positions between updates. When the true state deviated beyond a threshold, a correction was sent. This approach influenced later FPS networking (Quake, Unreal), but it was not practical for games with thousands of entities and limited bandwidth.

The distinction is important: SIMNET demonstrated that networked simulation was possible, but the deterministic lockstep technique emerged from a different need -- synchronizing massive simulations over minimal bandwidth.

### Command & Conquer and Early RTS (1995--1996)

The earliest commercial RTS games with multiplayer -- including **Command & Conquer** (1995) and **Warcraft II** (1995) -- used variations of lockstep synchronization. These games needed to synchronize hundreds of units with only dial-up bandwidth available. The natural solution was to send player commands rather than unit states.

### Age of Empires (1997) -- The Seminal Implementation

The most detailed public documentation of deterministic lockstep comes from **"1500 Archers on a 28.8: Network Programming in Age of Empires and Beyond"** by Paul Bettner and Mark Terrano, presented at GDC 2001. This paper described the networking architecture for Age of Empires I and II, which had to support:

- **8 players** in multiplayer
- Smooth simulation over LAN, modem, and internet
- A target platform of a **Pentium 90 with 16 MB RAM and a 28.8 kbps modem**

Their design goals were clear: rather than passing unit status, **run the exact same simulation on each machine, passing identical sets of commands issued by users at the same time**. Each PC synchronized their "game watches" and executed in exactly the same way.

Key implementation details from the paper:

- **Turns lasted 200 milliseconds** (the "communication turn")
- Commands issued during turn N were executed during turn **N+2** (a two-turn pipeline delay)
- This two-turn buffer allowed asynchronous message handling while gameplay continued
- Each client sent a **"Turn Done" message** (only 2 bytes for speed control data) when it finished processing
- The system checksummed **world state, pathfinding calculations, targeting data, and random number sequences** to detect desynchronization

Latency tolerance testing revealed:

| Latency | Player Experience |
|---------|-------------------|
| < 250 ms | Unnoticeable |
| 250--500 ms | Highly playable |
| > 500 ms | Noticeable degradation |

### StarCraft (1998) and StarCraft II (2010)

**StarCraft** used peer-to-peer deterministic lockstep. No unit positions, health, or stats were ever transmitted -- only player commands (click coordinates, unit orders). Everything else was derived from the deterministic simulation running on each machine.

**StarCraft II** shifted to a client-server topology but retained the lockstep model. A central server relayed inputs to all clients, but each client still ran the full deterministic simulation locally. This gave Blizzard better anti-cheat control while preserving the bandwidth efficiency of input-only synchronization.

### Supreme Commander (2007)

Gas Powered Games' **Supreme Commander** pushed deterministic lockstep to its limits with massive unit counts (thousands of units per player). The engine relied on IEEE 754 floating-point determinism, carefully controlling FPU settings at startup:

```c
_controlfp(_PC_24, _MCW_PC);   // Set 24-bit precision
_controlfp(_RC_NEAR, _MCW_RC);  // Set round-to-nearest mode
```

They asserted FPU register consistency every tick and achieved deterministic results across millions of users on Windows.

### Key Historical Timeline

| Year | Milestone |
|------|-----------|
| 1983--1990 | SIMNET uses dead reckoning for networked military simulation |
| 1995 | Command & Conquer and Warcraft II ship with lockstep multiplayer |
| 1997 | Age of Empires ships with 200 ms turn-based lockstep for 8 players |
| 1998 | StarCraft uses peer-to-peer deterministic lockstep |
| 2001 | Bettner & Terrano publish "1500 Archers on a 28.8" at GDC |
| 2007 | Supreme Commander demonstrates IEEE 754-based determinism at scale |
| 2010 | StarCraft II uses client-server lockstep |
| 2014 | Glenn Fiedler publishes "Deterministic Lockstep" on Gaffer on Games |
| 2017 | For Honor ships with lockstep + rollback hybrid |
| 2020 | Factorio 1.0 ships with deterministic lockstep for massive factory simulations |

---

## 3. How It Works Step by Step

### The Fundamental Loop

```
┌─────────────────────────────────────────────────────────────────┐
│                    DETERMINISTIC LOCKSTEP                        │
│                                                                 │
│   TICK N:                                                       │
│                                                                 │
│   1. Collect local player input                                 │
│   2. Broadcast input to all peers (or server)                   │
│   3. Wait until inputs from ALL peers for tick N are received   │
│   4. Apply all inputs to the simulation in a deterministic      │
│      order (e.g., sorted by player ID)                          │
│   5. Advance the simulation by one fixed timestep               │
│   6. (Optional) Compute and share a state checksum              │
│   7. Repeat for tick N+1                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Breakdown

**Step 1: Collect Local Input**

Each frame, the local player's input is captured and encoded into a compact structure. For an RTS, this might be a command like "select units 4,7,12 and move to (340, 560)." For a fighting game, it might be a bitmask of buttons pressed. The input must be fully self-contained -- no references to local state.

**Step 2: Broadcast Input**

The local input for tick N is sent to all other peers (in peer-to-peer) or to a relay server (in client-server lockstep). The input is tagged with the tick number it belongs to.

**Step 3: Wait for All Peers**

This is the "lock" in lockstep. The simulation **cannot advance** to tick N until inputs from every connected peer for tick N have arrived. If any peer's input is late, **everyone waits**. This is the fundamental trade-off: perfect synchronization at the cost of latency sensitivity.

**Step 4: Apply Inputs Deterministically**

Once all inputs for tick N are collected, they are applied to the simulation in a **canonical order** -- typically sorted by player ID. This ordering must be identical on every machine. If Player 2's input is applied before Player 1's on one machine but after on another, the simulations will diverge.

**Step 5: Advance the Simulation**

The game simulation advances by exactly one fixed timestep (e.g., 1/10th of a second, 1/30th of a second). The timestep **must be fixed** and identical on all machines. Variable timesteps are forbidden because they introduce non-determinism.

**Step 6: Checksum Verification (Optional but Recommended)**

After advancing, each peer computes a hash of the relevant game state and shares it. If hashes diverge, a desync has been detected. This is a development and debugging aid more than a runtime recovery mechanism.

### Peer-to-Peer vs. Client-Server Topology

**Peer-to-Peer (P2P):**

```
  Player A ◄──────► Player B
      ▲                 ▲
      │                 │
      └──── Player C ───┘

  Each peer sends input to every other peer.
  O(N²) connections for N players.
```

- Used by: Age of Empires, StarCraft 1, Supreme Commander
- Pros: No central server needed, lowest possible latency between any two peers
- Cons: O(N^2) connections, NAT traversal required, harder to prevent cheating

**Client-Server (Relay):**

```
  Player A ──► Server ──► Player A
  Player B ──► Server ──► Player B
  Player C ──► Server ──► Player C

  Server relays inputs. Does NOT simulate.
```

- Used by: StarCraft II, Factorio
- Pros: O(N) connections, easier NAT traversal, server can enforce input ordering and timing
- Cons: Slightly higher latency (client → server → client), requires server infrastructure

In both topologies, every client runs the full simulation. The server in client-server lockstep is a **relay**, not an authority -- it forwards inputs but does not compute game state.

---

## 4. The Determinism Requirement

### What Determinism Means

For lockstep to work, the simulation must be **deterministic**: given the same initial state and the same sequence of inputs, every machine must produce **bit-identical** results at every tick. Not "close enough" -- **exactly the same, down to the last bit.**

This is because errors compound. A unit positioned one pixel differently on tick 100 might path differently on tick 101, which changes combat outcomes on tick 150, which snowballs into a completely different game state by tick 300. There is no self-correction mechanism in pure lockstep -- any divergence is permanent and growing.

### Sources of Non-Determinism

Here is a comprehensive catalog of what can break determinism:

#### Floating-Point Arithmetic

This is the most notorious source of desyncs. IEEE 754 floating-point is **not inherently deterministic** across:

- **Different CPU architectures**: x87 (80-bit extended precision) vs. SSE (64-bit double) produce different intermediate results for the same operations
- **Different compilers**: MSVC, GCC, and Clang may reorder or optimize floating-point operations differently
- **Different optimization levels**: Debug builds may keep values in 80-bit registers while release builds use 64-bit
- **Fused multiply-add (FMA)**: Some CPUs (PowerPC, modern x86 with FMA3) perform `a * b + c` in a single instruction with only one rounding step, while others use two separate operations with two rounding steps
- **Transcendental functions**: `sin()`, `cos()`, `sqrt()` may use different algorithms on AMD vs. Intel CPUs

As Glenn Fiedler notes:

> "Even though the simulation above is deterministic on the same machine, that does not necessarily mean it would also be deterministic across different compilers, a different OS or different machine architectures."
>
> -- [Floating Point Determinism, Gaffer on Games](https://gafferongames.com/post/floating_point_determinism/)

#### Random Number Generators

Any randomness in the simulation (e.g., hit chance, AI decision-making, particle spawns that affect gameplay) must use a **seeded pseudorandom number generator** (PRNG) that is:

- Seeded identically on all peers
- Called in exactly the same order on all peers
- Never called from non-simulation code (rendering, audio, UI)

A single extra RNG call on one machine -- perhaps from rendering code that calls `random()` -- will shift the entire sequence and desync the game.

#### Iteration Order

Many programming languages do not guarantee iteration order for associative containers:

- **Hash maps / dictionaries** in most languages iterate in insertion order or hash-bucket order, which can vary by platform
- **Sets** may iterate differently across implementations
- **Object property enumeration** in JavaScript has a complex specification that most engines follow, but edge cases exist

If the simulation iterates over entities in a different order on different machines, the results will diverge.

#### Threading and Parallelism

Multithreaded code is inherently non-deterministic due to scheduling differences. If two threads write to shared state, the order of writes depends on OS scheduling, CPU load, and other unpredictable factors.

> "Reproducibility requires a programming methodology -- it's effectively incompatible with most parallelism forms."
>
> -- Expert consensus from [Floating Point Determinism, Gaffer on Games](https://gafferongames.com/post/floating_point_determinism/)

#### Uninitialized Memory

Reading uninitialized variables produces undefined behavior in C/C++ and unpredictable values that differ between machines and runs.

#### System-Dependent Behavior

- **Filesystem ordering** (e.g., `readdir()` returns files in different order on different OS)
- **Locale settings** affecting string comparison or number parsing
- **Timer resolution** differences affecting time-dependent calculations
- **Memory allocation addresses** that leak into hash computations

---

## 5. Achieving Determinism

### Fixed-Point Mathematics

The most reliable way to avoid floating-point non-determinism is to **not use floating-point at all** for simulation code. Fixed-point arithmetic uses integers scaled by a constant factor to represent fractional values.

A common format is **Q16.16** (16 integer bits, 16 fractional bits):

```
Value 10.5 in Q16.16:
  10.5 × 2^16 = 10.5 × 65536 = 688128

Stored as integer: 688128

To convert back: 688128 / 65536 = 10.5
```

**Advantages:**
- Integer arithmetic is identical on every platform
- Addition and subtraction work directly
- Multiplication requires a shift: `(a * b) >> 16`
- Completely deterministic across all hardware

**Disadvantages:**
- Limited range and precision compared to floating-point
- Requires reimplementing trigonometric functions (usually via lookup tables)
- Cannot use built-in physics engines or math libraries that use floats

The mobile RTS developers behind ["Minimizing the Pain of Lockstep Multiplayer"](https://www.gamedeveloper.com/programming/minimizing-the-pain-of-lockstep-multiplayer) used 32-bit fixed-point numbers with 12 fractional bits (4096 = 1.0). They also leveraged the x86 floating-point inexact exception flag to automatically detect accidental float usage in simulation code during development.

In TypeScript/JavaScript, a fixed-point library can be implemented using Q16.16 format:

```typescript
// Fixed-point Q16.16 arithmetic
const FRAC_BITS = 16;
const SCALE = 1 << FRAC_BITS; // 65536

function fpFromFloat(value: number): number {
  return Math.round(value * SCALE);
}

function fpToFloat(fp: number): number {
  return fp / SCALE;
}

function fpAdd(a: number, b: number): number {
  return a + b;
}

function fpSub(a: number, b: number): number {
  return a - b;
}

function fpMul(a: number, b: number): number {
  // Use intermediate 64-bit representation to avoid overflow
  // In JS, numbers are 64-bit floats, so this works for values < 2^31
  return (a * b) >> FRAC_BITS;
}

function fpDiv(a: number, b: number): number {
  return ((a << FRAC_BITS) / b) | 0;
}
```

For higher precision in TypeScript, `BigInt` can be used to avoid the 53-bit mantissa limitation of JavaScript `number`:

```typescript
const FRAC_BITS_64 = 32n;
const SCALE_64 = 1n << FRAC_BITS_64;

function fpMul64(a: bigint, b: bigint): bigint {
  return (a * b) >> FRAC_BITS_64;
}
```

### Controlled Floating-Point (When Fixed-Point Is Not Feasible)

If the game must use floating-point (e.g., because it relies on a physics engine), determinism can sometimes be achieved with strict controls:

1. **Restrict to a single compiler, OS, and CPU architecture** -- this is what Gas Powered Games did for Supreme Commander
2. **Set FPU control registers at startup** to force consistent precision and rounding:
   ```c
   _controlfp(_PC_24, _MCW_PC);    // 24-bit (single) precision
   _controlfp(_RC_NEAR, _MCW_RC);  // Round-to-nearest
   ```
3. **Assert FPU settings every tick** to catch external code (DLLs, drivers) that may change them
4. **Use `/fp:strict`** (MSVC) or `-ffp-model=strict` (GCC/Clang) to disable value-changing optimizations
5. **Avoid SSE for transcendental functions** -- wrap `sin()`, `cos()`, `sqrt()` in non-optimized function calls or use software implementations
6. **Never use FMA instructions** unless all target platforms support them identically

> **Warning:** Cross-platform floating-point determinism (e.g., Windows vs. macOS, x86 vs. ARM) is generally considered **not achievable** in practice. Even Battlezone 2 could not share replays between Xbox and PC versions due to floating-point differences.

### Seeded Pseudorandom Number Generators

All randomness in the simulation must come from a single, deterministic PRNG:

```typescript
/**
 * A simple xorshift32 PRNG — deterministic, fast, and identical across platforms.
 */
class DeterministicRNG {
  private state: number;

  constructor(seed: number) {
    this.state = seed || 1; // Must not be zero
  }

  /** Returns a pseudorandom integer in [0, 2^32) */
  next(): number {
    let x = this.state;
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    this.state = x >>> 0; // Ensure unsigned 32-bit
    return this.state;
  }

  /** Returns a pseudorandom float in [0, 1) */
  nextFloat(): number {
    return this.next() / 0x100000000;
  }

  /** Returns a pseudorandom integer in [min, max) */
  nextRange(min: number, max: number): number {
    return min + (this.next() % (max - min));
  }
}
```

Critical rules for simulation RNGs:

- **Seed from shared data** (e.g., match ID, combined player seeds, or a checksum of lobby settings) -- as done in Factorio, where "every client will always choose the same robot at the same time"
- **Never call the simulation RNG from rendering, audio, or UI code** -- maintain a separate RNG for visual effects
- **Call the RNG in the same order on every machine** -- if entity iteration order differs, RNG sequences diverge

### Deterministic Containers and Iteration

Replace non-deterministic data structures with deterministic alternatives:

| Non-Deterministic | Deterministic Alternative |
|---|---|
| `HashMap` / JS `Object` keys | Sorted array, `Map` with explicit sort before iteration, or integer-indexed array |
| `HashSet` | Sorted array or `Map` with deterministic iteration |
| Unordered entity lists | Arrays sorted by a stable entity ID |

In TypeScript, `Map` preserves insertion order (per the ES2015 spec), which is deterministic **if** entities are always inserted in the same order. However, after deletions and re-insertions, order can diverge. The safest approach is to always iterate over explicitly sorted arrays:

```typescript
// Safe: iterate entities in deterministic order
const sortedEntities = [...entities.values()].sort((a, b) => a.id - b.id);
for (const entity of sortedEntities) {
  entity.update(dt);
}
```

### Avoiding OS-Dependent Behavior

- **Never use wall-clock time** in simulation code -- use only the tick counter
- **Never use `Date.now()` or `performance.now()`** in simulation logic
- **Avoid system locale** in any string operations that affect gameplay
- **Be cautious with third-party libraries** -- many use floats, hash maps, or system calls internally

---

## 6. Turn-Based Input Collection

### The "Command Turn" System

The original Age of Empires implementation organized time into **communication turns** of approximately 200 ms. The system worked as a pipeline:

```
Timeline:
  Turn 1000    Turn 1001    Turn 1002    Turn 1003
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ Collect   │ │ Transmit │ │ Execute  │ │          │
  │ commands  │ │ commands │ │ commands │ │   ...    │
  │ from user │ │ to peers │ │ from     │ │          │
  │           │ │          │ │ turn 1000│ │          │
  └──────────┘ └──────────┘ └──────────┘ └──────────┘

  Commands issued in turn N are executed in turn N+2.
```

This two-turn pipeline gave the network a **full turn (200 ms) to deliver messages** before they were needed for execution. Commands from turn 1000 were transmitted during turn 1001 and executed during turn 1002.

### Why Input Delay Is Necessary

Without a pipeline delay, the game would need to wait for network delivery *within the same turn the input was issued*, causing constant hitching. The two-turn pipeline means:

- **Minimum input delay** = 2 × turn length (e.g., 2 × 200 ms = 400 ms in AoE)
- **Maximum input delay** = 2 × turn length + network round-trip time
- Turn length can be dynamically adjusted based on network conditions

### Adaptive Turn Length

Age of Empires dynamically adjusted the turn length based on two measurements:

1. **Local frame rate** -- averaged across multiple frames
2. **Network ping time** -- round-trip measurements to other players

The host received these measurements via "Turn Done" messages (each consuming only 2 bytes for speed control data) and adjusted the target frame rate and communication turn length. The adjustment algorithm rose quickly during latency spikes but descended slowly during stable conditions -- prioritizing smooth gameplay over responsiveness.

### Modern Tick-Based Systems

Modern implementations typically use fixed ticks (e.g., 10 Hz, 30 Hz, 60 Hz) rather than variable-length turns:

```typescript
const TICK_RATE = 10; // 10 ticks per second
const TICK_DURATION_MS = 1000 / TICK_RATE; // 100ms per tick
const INPUT_DELAY_TICKS = 3; // Commands execute 3 ticks after being issued

// Input issued at tick T is scheduled for execution at tick T + INPUT_DELAY_TICKS
function scheduleInput(input: PlayerInput, currentTick: number): void {
  const executionTick = currentTick + INPUT_DELAY_TICKS;
  inputBuffer.set(executionTick, input);
  broadcastToAllPeers({ tick: executionTick, input });
}
```

### Input Delay Windows

The input delay window is the number of ticks between when an input is issued and when it executes. Larger windows provide more time for network delivery but increase perceived latency:

| Tick Rate | Input Delay (ticks) | Input Delay (ms) | Use Case |
|---|---|---|---|
| 10 Hz | 2 | 200 ms | RTS games, city builders |
| 30 Hz | 3 | 100 ms | Strategy games, slower-paced action |
| 60 Hz | 4 | 67 ms | Fighting games (tight, but achievable on LAN) |
| 60 Hz | 6 | 100 ms | Fighting games (online) |

---

## 7. Handling Slow Peers

### The Fundamental Problem

In pure lockstep, the simulation on every machine advances at the speed of the **slowest peer**. If Player C has a 500 ms ping while Players A and B have 50 ms, everyone experiences delays dictated by Player C.

As noted by the SnapNet lockstep analysis:

> "Input delay will be dictated by whichever player has the highest latency."

The problem scales poorly: in a hypothetical scenario where 90% of players have acceptable connections, roughly 35% of 4-player matches and 57% of 8-player matches will be degraded by at least one poor connection.

### Strategy 1: Wait for Everyone (Classic Lockstep)

The simplest approach: if inputs for the current tick have not arrived from all peers, the simulation pauses. This is what Age of Empires, StarCraft, and Supreme Commander do.

**Pros:** Simple, guarantees synchronization
**Cons:** One slow peer freezes everyone; game stalls on packet loss

### Strategy 2: Adaptive Timeouts with Dropped Ticks

Instead of waiting forever, set a deadline. If a peer's input has not arrived by the deadline, treat it as "no input" for that tick:

```typescript
const TICK_TIMEOUT_MS = 150;

function tryAdvanceTick(currentTick: number): boolean {
  const deadline = tickStartTime + TICK_TIMEOUT_MS;

  for (const peer of peers) {
    if (!hasInputForTick(peer, currentTick)) {
      if (Date.now() < deadline) {
        return false; // Still waiting
      }
      // Timeout: use empty input for this peer
      setInputForTick(peer, currentTick, EMPTY_INPUT);
      peer.missedTicks++;
    }
  }

  return true; // All inputs ready (or timed out)
}
```

**Pros:** Non-lagging players are not frozen by a slow peer
**Cons:** The slow peer's commands are dropped, which may cause that player to experience desync or loss of control

### Strategy 3: Differential Input Delay

Apply higher input delay to peers with worse connections while keeping low delay for players with good connections:

```
Player A (20ms RTT):  input delay = 2 ticks (33ms at 60Hz)
Player B (50ms RTT):  input delay = 3 ticks (50ms at 60Hz)
Player C (200ms RTT): input delay = 6 ticks (100ms at 60Hz)
```

This shifts the latency burden to the player with the bad connection. All peers still synchronize on the same tick, but Player C's inputs are scheduled further into the future. Each player perceives a different amount of input lag.

### Strategy 4: Catch-Up Mechanism

When a peer falls behind (e.g., due to a CPU spike or reconnection), it can "catch up" by simulating multiple ticks per frame without rendering:

```typescript
function catchUp(currentTick: number, targetTick: number): void {
  while (currentTick < targetTick) {
    // Run simulation without rendering
    applyInputsForTick(currentTick);
    advanceSimulation(TICK_DURATION);
    currentTick++;
  }
  // Resume normal rendering
}
```

Factorio uses this approach: when a client reconnects after a brief disconnection, it must "simulate up to 400 ticks in order to catch up to the server again."

### Strategy 5: Speed Control (AoE Approach)

Rather than timeouts, Age of Empires dynamically adjusted the simulation speed for **all players** based on the worst-performing peer. If one player had high latency, the game slowed down slightly for everyone. The adjustment was:

- **Fast to increase** delay during spikes (aggressive ramp-up)
- **Slow to decrease** delay during stability (conservative ramp-down)

This prevented jarring stop/start behavior by making slowdowns smooth and gradual.

---

## 8. Desync Detection

### Why Desyncs Happen

Despite best efforts, desynchronization bugs are **inevitable** during development. They arise from:

- Accidental use of floating-point in simulation code
- Iteration over non-deterministic data structures
- Off-by-one errors in input scheduling
- Platform-specific behavior in library code
- Uninitialized memory
- Threading race conditions

A desync means that two or more peers' game states have diverged. Because lockstep has no authoritative server to correct the state, a desync is **permanent** and will only grow worse over time due to the butterfly effect.

### Checksum-Based Detection

The standard approach is to compute a hash or checksum of the game state at regular intervals and compare it across peers.

**What to hash:**

Age of Empires checksummed:
- World state (entity positions, health, etc.)
- Pathfinding calculation results
- Targeting data
- Random number generator state

A practical approach:

```typescript
function computeStateChecksum(state: GameState): number {
  let hash = 0;

  // Hash entity states in deterministic order
  const sortedEntities = [...state.entities.values()].sort((a, b) => a.id - b.id);
  for (const entity of sortedEntities) {
    hash = hashCombine(hash, entity.id);
    hash = hashCombine(hash, entity.x);
    hash = hashCombine(hash, entity.y);
    hash = hashCombine(hash, entity.hp);
    hash = hashCombine(hash, entity.actionState);
  }

  // Hash RNG state
  hash = hashCombine(hash, state.rng.state);

  // Hash global counters
  hash = hashCombine(hash, state.tickNumber);
  hash = hashCombine(hash, state.entityIdCounter);

  return hash >>> 0;
}

function hashCombine(seed: number, value: number): number {
  // FNV-1a inspired combine
  seed ^= value;
  seed = Math.imul(seed, 0x01000193);
  return seed >>> 0;
}
```

### When to Check

Computing a full state checksum every tick can be expensive. Common strategies:

- **Every tick** during development and testing
- **Every N ticks** (e.g., every 10 ticks) in production
- **Sampling** -- randomly select ticks to checksum
- **Hierarchical** -- hash subsystems separately (entities, RNG, resources) to narrow down the source

### What to Do When a Desync Is Detected

**During development:**
1. Dump the full game state on all peers at the desynced tick
2. Diff the state dumps to find the first divergence
3. Trace back to find the root cause

**In production:**
1. Display an "out of sync" error to all players
2. Optionally, attempt to resynchronize by having the "authority" (host or server) send a full state snapshot
3. Log the desync report for developers

### Factorio's Approach

Factorio's desync detection and debugging is one of the most documented in the industry:

- When a desync occurs, the game **automatically generates a desync report** (`desync-report-[timestamp].zip`)
- The report contains the **client's game state** and the **server's game state** at the time of the desync
- Developers **compare `script.dat` files** (containing mod/scenario storage tables) and level data files to spot divergences
- A **"Heavy Mode"** debug command saves and loads the game every tick, detecting state changes and recording dumps to identify problematic data modifications
- The key insight: by the time a desync is reported, the butterfly effect has caused millions of differences. To find the *original* cause, the server saves the current map, loads and re-saves a second instance, and compares whether the two stay identical -- isolating the initial divergence from accumulated drift

### Save/Load Determinism Testing

A powerful technique from Factorio: periodically save the game state, reload it, simulate forward, and compare against the never-saved simulation. If they diverge, the save/load process failed to capture some state. This catches:

- Uninitialized fields not included in serialization
- Transient state that leaks into simulation logic
- Platform-dependent behavior in serialization code

---

## 9. Lockstep + Rollback Hybrid

### The Latency Problem

Pure lockstep requires waiting for all inputs before advancing. For action games where responsiveness is critical, this is unacceptable. A 100 ms input delay in a fighting game or melee combat game makes the game feel unresponsive.

### For Honor's "Time Travel" Architecture

**For Honor** (Ubisoft Montreal, 2017) is the most prominent example of combining deterministic lockstep with rollback. The game features 8-player matches with hundreds of NPCs in a melee combat system that demands frame-precise responsiveness.

Jennifer Henry described the approach in her GDC 2019 talk ["Back to the Future! Working with Deterministic Simulation in 'For Honor'"](https://www.gdcvault.com/play/1026322/Back-to-the-Future-Working):

- The game uses a **non-authoritative, deterministic simulation** -- no server computes the game state
- To eliminate input delay, For Honor uses **"time travel"** (rollback): when a remote player's input arrives late, the game rolls back to the tick where that input should have been applied, re-simulates forward with the correct input, and presents the corrected state
- This required **running up to 8x the simulation in a single frame** (one resimulation per player in the worst case)
- The approach required six years of optimization and "drastic optimizations needed to run 8 times the gameplay in a single frame"

### How Lockstep + Rollback Works

```
Timeline (3-player game, Player C's input arrives late):

Tick 10:  A's input ✓  B's input ✓  C's input ✗ (not yet arrived)
          → Predict C's input (repeat last input or "no input")
          → Advance to tick 11 speculatively

Tick 11:  A's input ✓  B's input ✓  C's input for tick 10 arrives!
          → ROLLBACK to tick 10's confirmed state
          → Re-apply tick 10 with A + B + C's REAL inputs
          → Re-simulate tick 10 → tick 11
          → Continue forward

Tick 12:  All inputs arrive on time
          → Normal lockstep advance
```

### Requirements for Rollback

Adding rollback to lockstep requires:

1. **State snapshot/restore**: The full game state must be serializable and restorable quickly (ideally under 10% of the frame budget)
2. **Fast re-simulation**: The game must be able to simulate multiple ticks within a single frame, **without rendering**
3. **Decoupled rendering**: Visual output must be independent of simulation ticks -- interpolation between states is essential
4. **Misprediction handling**: When a rollback corrects a prediction, visual artifacts (teleporting entities) must be smoothed

### ECS Architecture Advantage

Entity-Component-System (ECS) architectures are particularly well-suited for lockstep + rollback because:

- **State snapshots** are cheap: memcpy entire component arrays
- **Restore** is equally cheap: memcpy back
- **Re-simulation** is fast: systems process dense component arrays efficiently
- **State isolation**: components clearly separate "simulation state" from "presentation state"

### Pseudo-Code: Lockstep with Rollback

```
function onTick(currentTick):
    // Save state snapshot for potential rollback
    snapshots[currentTick] = saveGameState()

    // Collect known inputs; predict missing ones
    for each player:
        if hasInput(player, currentTick):
            inputs[currentTick][player] = receivedInput(player, currentTick)
        else:
            inputs[currentTick][player] = predictInput(player)
            predictions[currentTick].add(player)

    // Apply all inputs and advance
    applyInputs(inputs[currentTick])
    advanceSimulation()

function onLateInputReceived(player, tick, input):
    if tick < currentTick AND player in predictions[tick]:
        // We predicted wrong -- rollback needed
        restoreGameState(snapshots[tick])
        inputs[tick][player] = input  // Replace prediction with real input

        // Re-simulate from the corrected tick to current
        for t in range(tick, currentTick):
            applyInputs(inputs[t])
            advanceSimulation()
            snapshots[t + 1] = saveGameState()
```

---

## 10. Bandwidth Efficiency

### Why Lockstep Is Bandwidth-Efficient

The core advantage of deterministic lockstep is that **only player inputs traverse the network**. Consider the comparison:

**State synchronization (server-authoritative):**
```
Per entity per tick:
  Position (x, y, z):     12 bytes
  Rotation (quat):        16 bytes
  Velocity:               12 bytes
  Health:                  4 bytes
  Action state:            4 bytes
  Total per entity:       48 bytes

1,000 entities × 48 bytes × 30 ticks/sec = 1,440,000 bytes/sec ≈ 1.4 MB/s
```

**Deterministic lockstep:**
```
Per player per tick:
  Command type:            1 byte
  Target position:         8 bytes (2 × 32-bit fixed-point)
  Selected units (bitmask): 4 bytes
  Total per input:        ~13 bytes

8 players × 13 bytes × 10 ticks/sec = 1,040 bytes/sec ≈ 1 KB/s
```

That is a **1,400x reduction** in bandwidth. As Factorio's developers noted, direct state transmission for their game could require "1500 MB per second of transfer in some cases," while their lockstep approach consumes almost no bandwidth.

### Scaling with Player Count

Bandwidth scales as O(P) in client-server lockstep or O(P^2) in peer-to-peer:

| Players | P2P Bandwidth (per player) | Client-Server Bandwidth (per player) |
|---|---|---|
| 2 | 1 × input_size | 2 × input_size |
| 4 | 3 × input_size | 4 × input_size |
| 8 | 7 × input_size | 8 × input_size |
| 16 | 15 × input_size | 16 × input_size |

Even with 16 players in P2P, bandwidth remains trivial compared to state synchronization.

### UDP with Redundant Inputs

Glenn Fiedler recommends using UDP with **redundant input transmission** to handle packet loss without retransmission delays:

> Include all unacknowledged inputs in every UDP packet. With 6-bit inputs at 60 Hz and a 120-frame worst-case buffer, overhead reaches only "90 bytes of input" maximum.

Additionally, **delta encoding** compresses inputs efficiently: write a single bit `0` if the input is the same as the previous tick, or `1` followed by the changed input. In many games, inputs are identical across consecutive ticks most of the time.

```typescript
function encodeInputPacket(
  unackedInputs: Map<number, PlayerInput>,
  lastAckedTick: number
): Uint8Array {
  const writer = new BitWriter();

  writer.writeUint32(lastAckedTick);
  writer.writeUint8(unackedInputs.size);

  let previousInput: PlayerInput | null = null;

  for (const [tick, input] of unackedInputs) {
    if (previousInput && inputsEqual(input, previousInput)) {
      writer.writeBit(0); // Same as previous
    } else {
      writer.writeBit(1); // Changed
      writeInput(writer, input);
    }
    previousInput = input;
  }

  return writer.toBytes();
}
```

---

## 11. Common Pitfalls

### Pitfall 1: Floating-Point in Simulation Code

**Symptom:** Desyncs that only appear on certain hardware, or between debug and release builds.

**Cause:** Floating-point arithmetic producing different results across platforms.

**Solution:** Use fixed-point math for all simulation logic. If floats are unavoidable, control compiler settings and FPU registers rigorously. Assert FPU state every tick.

### Pitfall 2: Non-Deterministic Iteration Order

**Symptom:** Desyncs that appear sporadically, especially after entity creation/destruction.

**Cause:** Iterating over hash maps, sets, or objects in platform-dependent order.

**Solution:** Always sort entities by a stable ID before iterating. Use arrays or ordered maps.

### Pitfall 3: RNG Contamination

**Symptom:** Desyncs that appear when one player has a different frame rate or rendering configuration.

**Cause:** Rendering code calls the simulation RNG, shifting the sequence differently on machines with different frame rates.

**Solution:** Maintain strictly separate RNGs for simulation and presentation. Never let rendering, audio, or UI code touch the simulation RNG.

### Pitfall 4: Input Delay Feel

**Symptom:** Players complain the game feels "sluggish" or "laggy" even on good connections.

**Cause:** The inherent input pipeline delay (typically 100--400 ms) is noticeable in action-oriented gameplay.

**Mitigation strategies:**
- **Immediate visual feedback**: Play animations, sound effects, and particles immediately on input, even though the simulation hasn't processed it yet. Age of Empires played unit acknowledgment sounds ("Yes, my lord!") immediately.
- **Dynamic delay adjustment**: Reduce input delay when network conditions are good, increase when they're bad.
- **Genre selection**: Lockstep works best in genres where 200+ ms delay is tolerable (RTS, turn-based, simulation).

### Pitfall 5: Late Joiners and Reconnection

**Symptom:** Cannot support players joining mid-match or reconnecting after disconnection.

**Cause:** A new peer has no game state and cannot derive it without either (a) the full input history or (b) a state snapshot.

**Solutions:**

| Approach | Pros | Cons |
|---|---|---|
| **Send full input history** | No serialization infrastructure needed | Memory-intensive; must simulate from tick 0; slow for long matches |
| **Send state snapshot** | Instant join; no history needed | Requires serialization/deserialization; must pause others briefly |
| **Hybrid: periodic checkpoints + recent inputs** | Fast join; bounded history | More complex; must maintain checkpoint system |

Factorio uses the state snapshot approach: the server sends the full map, then the client catches up by replaying recent inputs. During catch-up, the client "simulates up to 400 ticks" without rendering to reach the current state.

### Pitfall 6: Desync Bugs Are Extremely Hard to Debug

**Symptom:** The game desyncs, but by the time it's detected, the states have diverged so much that the original cause is buried under cascading differences.

**Cause:** The butterfly effect -- one small difference amplifies exponentially.

**Solutions:**
- Checksum **every tick** during development
- Checksum **subsystems independently** (entities, RNG, resources, pathfinding) to narrow down the source
- Use **save/load verification** (Factorio's technique): save state, reload, re-simulate, compare
- Use **record/replay**: record all inputs and replay them deterministically; if the replay diverges from the live game, the simulation is non-deterministic

### Pitfall 7: Save/Load Determinism

**Symptom:** Loading a saved game and continuing produces different results than never saving.

**Cause:** The save file does not capture all simulation-relevant state (e.g., internal data structure ordering, cache state, accumulated floating-point errors).

**Solution:** Treat save/load as a determinism test. Periodically save and reload during automated testing and verify that the simulation remains bit-identical.

### Pitfall 8: Third-Party Libraries

**Symptom:** Desyncs after integrating a physics engine, pathfinding library, or other middleware.

**Cause:** Most libraries use floating-point, hash maps, and may have internal non-deterministic behavior.

**Solution:** Audit all third-party code paths for determinism. Prefer libraries specifically designed for deterministic simulation. In many cases, you will need to **reimplement** physics, pathfinding, and collision detection from scratch using fixed-point math.

---

## 12. Pseudo-Code / Code Examples

### Generic Lockstep Manager (Pseudo-Code)

```
class LockstepManager:
    currentTick = 0
    inputBuffer = {}       // tick → {playerId → input}
    playerCount = 0
    tickRate = 10          // ticks per second
    inputDelayTicks = 3    // schedule inputs N ticks ahead
    accumulator = 0.0

    function update(deltaTime):
        accumulator += deltaTime

        while accumulator >= (1.0 / tickRate):
            if canAdvanceTick():
                executeTick()
                accumulator -= (1.0 / tickRate)
            else:
                break  // Waiting for inputs

    function canAdvanceTick():
        for each player in connectedPlayers:
            if inputBuffer[currentTick][player] is missing:
                return false
        return true

    function onLocalInput(input):
        scheduledTick = currentTick + inputDelayTicks
        inputBuffer[scheduledTick][localPlayerId] = input
        broadcast({type: "input", tick: scheduledTick, input: input})

    function onRemoteInput(playerId, tick, input):
        inputBuffer[tick][playerId] = input

    function executeTick():
        // Apply inputs in deterministic order
        sortedPlayers = sort(connectedPlayers, by: playerId)
        for each player in sortedPlayers:
            input = inputBuffer[currentTick][player]
            applyInput(player, input)

        // Advance simulation
        simulation.step(1.0 / tickRate)

        // Checksum verification
        checksum = simulation.computeChecksum()
        broadcast({type: "checksum", tick: currentTick, value: checksum})

        // Cleanup old inputs
        delete inputBuffer[currentTick - HISTORY_SIZE]

        currentTick++
```

### Complete TypeScript Implementation

The following is a practical, runnable TypeScript implementation of a deterministic lockstep system suitable for a simple 2D game:

```typescript
// ─────────────────────────────────────────────
// types.ts — Core types for the lockstep system
// ─────────────────────────────────────────────

/** A player input for a single tick */
interface PlayerInput {
  /** Bitmask: bit 0=up, 1=down, 2=left, 3=right, 4=action */
  buttons: number;
  /** Target X coordinate (fixed-point Q16.16) */
  targetX: number;
  /** Target Y coordinate (fixed-point Q16.16) */
  targetY: number;
}

/** An input message sent over the network */
interface InputMessage {
  type: "input";
  playerId: number;
  tick: number;
  input: PlayerInput;
}

/** A checksum message for desync detection */
interface ChecksumMessage {
  type: "checksum";
  playerId: number;
  tick: number;
  checksum: number;
}

type NetworkMessage = InputMessage | ChecksumMessage;

/** An entity in the game world */
interface Entity {
  id: number;
  ownerId: number;
  x: number; // Fixed-point Q16.16
  y: number; // Fixed-point Q16.16
  hp: number;
  state: number;
}

// ─────────────────────────────────────────────
// fixed-point.ts — Deterministic fixed-point math
// ─────────────────────────────────────────────

const FP_BITS = 16;
const FP_SCALE = 1 << FP_BITS; // 65536
const FP_HALF = FP_SCALE >> 1; // 32768 (for rounding)

const FP = {
  fromInt(n: number): number {
    return n << FP_BITS;
  },

  fromFloat(n: number): number {
    return Math.round(n * FP_SCALE);
  },

  toFloat(fp: number): number {
    return fp / FP_SCALE;
  },

  toInt(fp: number): number {
    return fp >> FP_BITS;
  },

  add(a: number, b: number): number {
    return (a + b) | 0;
  },

  sub(a: number, b: number): number {
    return (a - b) | 0;
  },

  mul(a: number, b: number): number {
    // For values that fit in 32 bits, this is safe in JS
    // For larger values, split into high/low 16-bit parts
    const aHi = a >> FP_BITS;
    const aLo = a & 0xFFFF;
    const bHi = b >> FP_BITS;
    const bLo = b & 0xFFFF;
    return (aHi * bHi << FP_BITS)
         + (aHi * bLo)
         + (aLo * bHi)
         + ((aLo * bLo) >> FP_BITS);
  },

  div(a: number, b: number): number {
    if (b === 0) throw new Error("Division by zero");
    // Use Number to avoid overflow, then truncate
    return ((a / b) * FP_SCALE) | 0;
  },

  /** Integer square root via Newton's method (deterministic) */
  sqrt(fp: number): number {
    if (fp <= 0) return 0;
    let guess = fp >> 1;
    if (guess === 0) guess = 1;
    for (let i = 0; i < 20; i++) {
      const next = (guess + FP.div(fp, guess)) >> 1;
      if (next >= guess) break;
      guess = next;
    }
    return guess;
  },
} as const;

// ─────────────────────────────────────────────
// rng.ts — Deterministic PRNG (xorshift32)
// ─────────────────────────────────────────────

class DeterministicRNG {
  private _state: number;

  constructor(seed: number) {
    // Seed must not be zero
    this._state = (seed === 0 ? 1 : seed) >>> 0;
  }

  get state(): number {
    return this._state;
  }

  next(): number {
    let x = this._state;
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    this._state = x >>> 0;
    return this._state;
  }

  /** Returns integer in [min, max) */
  range(min: number, max: number): number {
    return min + (this.next() % (max - min));
  }
}

// ─────────────────────────────────────────────
// simulation.ts — Deterministic game simulation
// ─────────────────────────────────────────────

const MOVE_SPEED = FP.fromFloat(2.0); // 2 units per tick
const BUTTON_UP = 1 << 0;
const BUTTON_DOWN = 1 << 1;
const BUTTON_LEFT = 1 << 2;
const BUTTON_RIGHT = 1 << 3;
const BUTTON_ACTION = 1 << 4;

class GameSimulation {
  entities: Map<number, Entity> = new Map();
  rng: DeterministicRNG;
  tick: number = 0;
  nextEntityId: number = 1;

  constructor(seed: number) {
    this.rng = new DeterministicRNG(seed);
  }

  /** Spawn an entity at the given fixed-point position */
  spawnEntity(ownerId: number, x: number, y: number): Entity {
    const entity: Entity = {
      id: this.nextEntityId++,
      ownerId,
      x,
      y,
      hp: 100,
      state: 0,
    };
    this.entities.set(entity.id, entity);
    return entity;
  }

  /** Apply a single player's input to all their entities */
  applyInput(playerId: number, input: PlayerInput): void {
    // Iterate in deterministic order (sorted by entity ID)
    const playerEntities = this.getEntitiesSorted(playerId);

    let dx = 0;
    let dy = 0;
    if (input.buttons & BUTTON_UP) dy = FP.sub(0, MOVE_SPEED);
    if (input.buttons & BUTTON_DOWN) dy = FP.add(dy, MOVE_SPEED);
    if (input.buttons & BUTTON_LEFT) dx = FP.sub(0, MOVE_SPEED);
    if (input.buttons & BUTTON_RIGHT) dx = FP.add(dx, MOVE_SPEED);

    for (const entity of playerEntities) {
      entity.x = FP.add(entity.x, dx);
      entity.y = FP.add(entity.y, dy);
    }
  }

  /** Advance the simulation by one tick (non-input game logic) */
  step(): void {
    // Example: process all entities in deterministic order
    const allEntities = [...this.entities.values()].sort(
      (a, b) => a.id - b.id
    );

    for (const entity of allEntities) {
      // Example: random damage from environment (deterministic via seeded RNG)
      if (this.rng.range(0, 100) < 2) {
        entity.hp -= 1;
      }

      // Remove dead entities
      if (entity.hp <= 0) {
        this.entities.delete(entity.id);
      }
    }

    this.tick++;
  }

  /** Get entities owned by a player, sorted by ID */
  private getEntitiesSorted(playerId: number): Entity[] {
    const result: Entity[] = [];
    for (const entity of this.entities.values()) {
      if (entity.ownerId === playerId) {
        result.push(entity);
      }
    }
    return result.sort((a, b) => a.id - b.id);
  }

  /** Compute a deterministic checksum of the game state */
  computeChecksum(): number {
    let hash = 0x811C9DC5; // FNV offset basis

    hash = fnvCombine(hash, this.tick);
    hash = fnvCombine(hash, this.rng.state);
    hash = fnvCombine(hash, this.nextEntityId);

    const sorted = [...this.entities.values()].sort(
      (a, b) => a.id - b.id
    );

    for (const e of sorted) {
      hash = fnvCombine(hash, e.id);
      hash = fnvCombine(hash, e.ownerId);
      hash = fnvCombine(hash, e.x);
      hash = fnvCombine(hash, e.y);
      hash = fnvCombine(hash, e.hp);
      hash = fnvCombine(hash, e.state);
    }

    return hash >>> 0;
  }
}

function fnvCombine(hash: number, value: number): number {
  hash ^= value & 0xFF;
  hash = Math.imul(hash, 0x01000193);
  hash ^= (value >> 8) & 0xFF;
  hash = Math.imul(hash, 0x01000193);
  hash ^= (value >> 16) & 0xFF;
  hash = Math.imul(hash, 0x01000193);
  hash ^= (value >> 24) & 0xFF;
  hash = Math.imul(hash, 0x01000193);
  return hash >>> 0;
}

// ─────────────────────────────────────────────
// lockstep.ts — Lockstep network manager
// ─────────────────────────────────────────────

const EMPTY_INPUT: PlayerInput = { buttons: 0, targetX: 0, targetY: 0 };

interface LockstepConfig {
  tickRate: number;        // Ticks per second (e.g., 10)
  inputDelayTicks: number; // How many ticks ahead inputs are scheduled (e.g., 3)
  playerIds: number[];     // All player IDs in the match
  localPlayerId: number;   // This client's player ID
  seed: number;            // Shared RNG seed
}

type SendFunction = (message: NetworkMessage) => void;

class LockstepManager {
  private simulation: GameSimulation;
  private config: LockstepConfig;
  private send: SendFunction;

  /** tick → (playerId → input) */
  private inputBuffer: Map<number, Map<number, PlayerInput>> = new Map();

  /** tick → (playerId → checksum) */
  private checksumBuffer: Map<number, Map<number, number>> = new Map();

  private currentTick: number = 0;
  private accumulator: number = 0;
  private tickDurationMs: number;

  constructor(config: LockstepConfig, send: SendFunction) {
    this.config = config;
    this.send = send;
    this.simulation = new GameSimulation(config.seed);
    this.tickDurationMs = 1000 / config.tickRate;
  }

  /** Call this from your render loop with the elapsed time in ms */
  update(deltaMs: number): void {
    this.accumulator += deltaMs;

    while (this.accumulator >= this.tickDurationMs) {
      if (this.canAdvance()) {
        this.executeTick();
        this.accumulator -= this.tickDurationMs;
      } else {
        // Waiting for remote inputs — do not consume accumulator
        break;
      }
    }
  }

  /** Submit a local input. It will be scheduled INPUT_DELAY ticks ahead. */
  submitLocalInput(input: PlayerInput): void {
    const scheduledTick = this.currentTick + this.config.inputDelayTicks;

    this.storeInput(scheduledTick, this.config.localPlayerId, input);

    this.send({
      type: "input",
      playerId: this.config.localPlayerId,
      tick: scheduledTick,
      input,
    });
  }

  /** Handle an incoming network message */
  onMessage(msg: NetworkMessage): void {
    switch (msg.type) {
      case "input":
        this.storeInput(msg.tick, msg.playerId, msg.input);
        break;

      case "checksum":
        this.storeChecksum(msg.tick, msg.playerId, msg.checksum);
        break;
    }
  }

  /** Get the simulation for rendering */
  getSimulation(): GameSimulation {
    return this.simulation;
  }

  getCurrentTick(): number {
    return this.currentTick;
  }

  // ── Private ─────────────────────────────────

  private canAdvance(): boolean {
    for (const playerId of this.config.playerIds) {
      const tickInputs = this.inputBuffer.get(this.currentTick);
      if (!tickInputs || !tickInputs.has(playerId)) {
        return false;
      }
    }
    return true;
  }

  private executeTick(): void {
    const tickInputs = this.inputBuffer.get(this.currentTick)!;

    // Apply inputs in deterministic order (ascending player ID)
    const sortedPlayerIds = [...this.config.playerIds].sort((a, b) => a - b);
    for (const playerId of sortedPlayerIds) {
      const input = tickInputs.get(playerId) ?? EMPTY_INPUT;
      this.simulation.applyInput(playerId, input);
    }

    // Advance non-input simulation logic
    this.simulation.step();

    // Desync detection: compute and broadcast checksum
    const checksum = this.simulation.computeChecksum();
    this.storeChecksum(this.currentTick, this.config.localPlayerId, checksum);
    this.send({
      type: "checksum",
      playerId: this.config.localPlayerId,
      tick: this.currentTick,
      checksum,
    });

    // Verify checksums from other players for past ticks
    this.verifyChecksums(this.currentTick);

    // Cleanup old data
    this.cleanup(this.currentTick - 120);

    this.currentTick++;
  }

  private storeInput(tick: number, playerId: number, input: PlayerInput): void {
    if (!this.inputBuffer.has(tick)) {
      this.inputBuffer.set(tick, new Map());
    }
    this.inputBuffer.get(tick)!.set(playerId, input);
  }

  private storeChecksum(tick: number, playerId: number, checksum: number): void {
    if (!this.checksumBuffer.has(tick)) {
      this.checksumBuffer.set(tick, new Map());
    }
    this.checksumBuffer.get(tick)!.set(playerId, checksum);
  }

  private verifyChecksums(tick: number): void {
    const tickChecksums = this.checksumBuffer.get(tick);
    if (!tickChecksums) return;

    const localChecksum = tickChecksums.get(this.config.localPlayerId);
    if (localChecksum === undefined) return;

    for (const [playerId, checksum] of tickChecksums) {
      if (playerId !== this.config.localPlayerId && checksum !== localChecksum) {
        console.error(
          `DESYNC DETECTED at tick ${tick}! ` +
          `Local (player ${this.config.localPlayerId}): 0x${localChecksum.toString(16)}, ` +
          `Remote (player ${playerId}): 0x${checksum.toString(16)}`
        );
        // In production: halt the game and display an error
        // In development: dump full state for comparison
      }
    }
  }

  private cleanup(beforeTick: number): void {
    for (const [tick] of this.inputBuffer) {
      if (tick < beforeTick) this.inputBuffer.delete(tick);
    }
    for (const [tick] of this.checksumBuffer) {
      if (tick < beforeTick) this.checksumBuffer.delete(tick);
    }
  }
}

// ─────────────────────────────────────────────
// Usage example
// ─────────────────────────────────────────────

/*
// Initialize
const config: LockstepConfig = {
  tickRate: 10,
  inputDelayTicks: 3,
  playerIds: [1, 2],
  localPlayerId: 1,
  seed: 12345,
};

const lockstep = new LockstepManager(config, (msg) => {
  // Send message over WebSocket / WebRTC / etc.
  socket.send(JSON.stringify(msg));
});

// Spawn initial entities for each player
lockstep.getSimulation().spawnEntity(1, FP.fromInt(100), FP.fromInt(100));
lockstep.getSimulation().spawnEntity(2, FP.fromInt(200), FP.fromInt(200));

// Game loop
function gameLoop(timestamp: number): void {
  const delta = timestamp - lastTimestamp;
  lastTimestamp = timestamp;

  // Capture input
  const input: PlayerInput = {
    buttons: (keys.up ? BUTTON_UP : 0)
           | (keys.down ? BUTTON_DOWN : 0)
           | (keys.left ? BUTTON_LEFT : 0)
           | (keys.right ? BUTTON_RIGHT : 0),
    targetX: 0,
    targetY: 0,
  };
  lockstep.submitLocalInput(input);

  // Update lockstep (may advance 0 or more ticks)
  lockstep.update(delta);

  // Render based on simulation state
  render(lockstep.getSimulation());

  requestAnimationFrame(gameLoop);
}

// Handle incoming network messages
socket.onmessage = (event) => {
  const msg = JSON.parse(event.data) as NetworkMessage;
  lockstep.onMessage(msg);
};

requestAnimationFrame(gameLoop);
*/
```

### Desync Debugging Helper

```typescript
/**
 * Dumps the full game state as a string for comparison.
 * Run this on both peers when a desync is detected, then diff the output.
 */
function dumpState(sim: GameSimulation): string {
  const lines: string[] = [];
  lines.push(`tick: ${sim.tick}`);
  lines.push(`rng_state: ${sim.rng.state}`);
  lines.push(`next_entity_id: ${sim.nextEntityId}`);
  lines.push(`entity_count: ${sim.entities.size}`);

  const sorted = [...sim.entities.values()].sort((a, b) => a.id - b.id);
  for (const e of sorted) {
    lines.push(
      `  entity[${e.id}]: owner=${e.ownerId} ` +
      `pos=(${FP.toFloat(e.x).toFixed(4)}, ${FP.toFloat(e.y).toFixed(4)}) ` +
      `hp=${e.hp} state=${e.state}`
    );
  }

  return lines.join("\n");
}
```

---

## 13. Modern Relevance

### Factorio (2020)

**Factorio** is the most prominent modern example of deterministic lockstep at scale. The game simulates massive factories with tens of thousands of entities (belts, inserters, assemblers, robots, trains) -- far too many to synchronize via state replication.

Key technical details:
- The entire simulation is deterministic -- "all of the players need to simulate every single tick of the game identically"
- Uses seeded PRNGs so that "every client will always choose the same robot at the same time"
- A **"Latency State"** layer provides immediate visual feedback by speculatively applying local inputs before server confirmation, then reconciling when confirmed actions arrive
- Desync detection generates automatic reports containing both client and server state snapshots
- Bandwidth is minimal: "people with keyboards can only generate a few hundred bytes per second"
- Game speed is limited by the slowest player's simulation speed
- Heavy desync debugging mode saves and loads the game every tick to detect state corruption

### RTS Renaissance

The 2020s have seen renewed interest in RTS games, and deterministic lockstep remains the architecture of choice:

- **Age of Empires IV** (2021) -- continues the lockstep tradition
- **Stormgate** (2024) -- from former StarCraft II developers at Frost Giant Studios
- **Manor Lords** (2024) -- city-builder/RTS hybrid
- Indie RTS games frequently use lockstep to minimize server costs

### Blockchain and On-Chain Gaming

Deterministic lockstep shares a philosophical foundation with blockchain games: both require **deterministic state transitions from inputs**. The blockchain acts as a consensus layer ensuring all participants agree on the sequence of inputs:

- **Fully on-chain games** submit each player action to the blockchain; after consensus, all clients derive the same state deterministically
- **State channels** (layer-2) allow players to exchange inputs off-chain, only settling the final state on-chain -- essentially a two-player lockstep protocol with cryptographic verification
- Games with low tick rates (turn-based, strategy) are most suitable for on-chain execution

The determinism requirement is identical: game logic must produce bit-identical results given the same inputs, regardless of which node executes it.

### Fighting Games and GGPO-Style Rollback

While fighting games have largely moved to rollback netcode (GGPO), many still use **deterministic simulation** as the foundation. The difference is in how they handle latency:

- **Pure lockstep**: Wait for inputs (too laggy for fighting games)
- **Lockstep + rollback**: Predict inputs, simulate ahead, roll back and correct when real inputs arrive

Games like **Guilty Gear Strive**, **Street Fighter 6**, and **Mortal Kombat 1** use deterministic simulation with rollback -- a direct descendant of the lockstep tradition.

### Replay and Spectator Systems

Deterministic lockstep naturally enables **replay systems**: recording only the inputs (a few kilobytes) allows perfect reproduction of an entire match. StarCraft and Age of Empires replay files are tiny because they contain only the command stream.

This property also enables:
- **Asynchronous spectating**: Share the input file and let the viewer simulate locally
- **Server-side replay validation**: Re-simulate on the server to verify match results (anti-cheat)
- **Machine learning training**: Feed input sequences to AI agents for training

---

## 14. When to Use / When Not to Use

### Ideal Use Cases

| Use Case | Why Lockstep Works |
|---|---|
| **RTS games** (Age of Empires, StarCraft) | Thousands of entities make state sync impractical; players tolerate 200+ ms input delay |
| **City builders / factory games** (Factorio, Satisfactory co-op) | Massive entity counts; low interaction frequency; latency tolerance high |
| **Turn-based games** (Civilization multiplayer) | Input delay is invisible because turns are long |
| **Fighting games** (with rollback) | Requires determinism for rollback; only 2 players; small state for fast save/restore |
| **Board game adaptations** | Natural lockstep fit; tiny input payloads |
| **Blockchain games** | Deterministic state transitions required by design |
| **Games with replay systems** | Lockstep makes replays trivial: record inputs only |

### Poor Fit

| Use Case | Why Lockstep Struggles |
|---|---|
| **FPS games** (Overwatch, Valorant) | Players demand < 50 ms perceived input delay; 16+ players make lockstep latency unacceptable |
| **MMOs / large-player-count games** | The slowest of N players determines everyone's experience; N > 8 becomes painful |
| **Physics-heavy games** (Rocket League) | Achieving deterministic physics across platforms is extremely difficult; server-authoritative with reconciliation is proven |
| **Cross-platform play** (PC + console + mobile) | Floating-point behavior differs across CPU architectures; fixed-point math required everywhere |
| **Games with frequent player churn** | Late joiners require full state transfer or input replay, both disruptive |
| **Games using third-party physics engines** | Most physics engines (PhysX, Box2D, Havok) are not deterministic across platforms |

### Decision Framework

```
Do you need to synchronize > 1,000 entities?
├── YES → Is latency tolerance > 100ms?
│         ├── YES → DETERMINISTIC LOCKSTEP ✓
│         └── NO  → Lockstep + Rollback, or reconsider entity count
└── NO  → Is the game competitive (frame-precise)?
          ├── YES → Is it 1v1 or 2v2?
          │         ├── YES → LOCKSTEP + ROLLBACK ✓ (GGPO-style)
          │         └── NO  → Server-authoritative with CSP
          └── NO  → Server-authoritative with interpolation
```

### Trade-Off Summary

| Property | Lockstep | Server-Authoritative | Lockstep + Rollback |
|---|---|---|---|
| Bandwidth | Very low (inputs only) | High (state updates) | Very low (inputs only) |
| Input latency | High (visible delay) | Low (with CSP) | Low (predicted) |
| Determinism required | Yes (strict) | No | Yes (strict) |
| Implementation complexity | Medium | Medium | High |
| Entity scaling | Excellent | Poor (bandwidth) | Excellent |
| Player scaling | Poor (> 8 players) | Good | Fair (2-8 players) |
| Cheat resistance | Low (all state visible) | High (server authority) | Low (all state visible) |
| Late join support | Difficult | Easy | Difficult |
| Replay support | Trivial (record inputs) | Complex (record state) | Trivial (record inputs) |

---

## 15. References

### Primary Sources

- [1500 Archers on a 28.8: Network Programming in Age of Empires and Beyond](https://www.gamedeveloper.com/programming/1500-archers-on-a-28-8-network-programming-in-age-of-empires-and-beyond) -- Paul Bettner & Mark Terrano, GDC 2001. The seminal paper on lockstep networking in RTS games. ([PDF at Yale](https://zoo.cs.yale.edu/classes/cs538/readings/papers/terrano_1500arch.pdf))

- [Deterministic Lockstep](https://gafferongames.com/post/deterministic_lockstep/) -- Glenn Fiedler, Gaffer on Games, 2014. Detailed technical explanation with UDP optimization techniques.

- [Floating Point Determinism](https://gafferongames.com/post/floating_point_determinism/) -- Glenn Fiedler, Gaffer on Games. Comprehensive analysis of floating-point non-determinism sources and mitigations.

- [Back to the Future! Working with Deterministic Simulation in 'For Honor'](https://www.gdcvault.com/play/1026322/Back-to-the-Future-Working) -- Jennifer Henry, GDC 2019. For Honor's lockstep + rollback architecture.

### Factorio Developer Blog

- [Friday Facts #76 - MP Inside Out](https://www.factorio.com/blog/post/fff-76) -- Factorio's lockstep multiplayer architecture overview.
- [Friday Facts #188 - Bug, Bug, Desync](https://factorio.com/blog/post/fff-188) -- Desync debugging in Factorio.
- [Friday Facts #302 - The Multiplayer Megapacket](https://www.factorio.com/blog/post/fff-302) -- Latency state, bandwidth optimization, and the megapacket bug.
- [Desynchronization - Factorio Wiki](https://wiki.factorio.com/Desynchronization) -- Comprehensive documentation on Factorio desync detection and reporting.

### Technical Articles

- [Netcode Architectures Part 1: Lockstep](https://www.snapnet.dev/blog/netcode-architectures-part-1-lockstep/) -- SnapNet. Detailed comparison of lockstep variants and scaling analysis.

- [Game Networking Demystified, Part III: Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html) -- Ruoyu Sun. Clear explanation of lockstep with LAN vs. internet timing.

- [Game Networking Demystified, Part II: Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html) -- Ruoyu Sun. Determinism requirements for lockstep.

- [Minimizing the Pain of Lockstep Multiplayer](https://www.gamedeveloper.com/programming/minimizing-the-pain-of-lockstep-multiplayer) -- Practical advice on fixed-point math, desync detection, and code organization for a mobile RTS.

- [Deterministic Lockstep Networking Demystified](https://zacksinisi.com/deterministic-lockstep-networking-demystified/) -- Zack Sinisi. Implementation walkthrough with Unity/C# focus.

- [Preparing Your Game for Deterministic Netcode](https://yal.cc/preparing-your-game-for-deterministic-netcode/) -- Practical tier-based approach to making a game deterministic.

- [Deterministic Physics in TypeScript: Why I Wrote a Fixed-Point Engine](https://dev.to/shaisrc/deterministic-physics-in-ts-why-i-wrote-a-fixed-point-engine-4b0l) -- TypeScript fixed-point math library with benchmarks.

### Additional Resources

- [Introduction to Networked Physics](https://gafferongames.com/post/introduction_to_networked_physics/) -- Glenn Fiedler. Context for choosing between lockstep, snapshot interpolation, and state synchronization.

- [Choosing the Right Network Model for Your Multiplayer Game](https://mas-bandwidth.com/choosing-the-right-network-model-for-your-multiplayer-game/) -- Decision framework for networking architectures.

- [Don't Use Lockstep in RTS Games](https://medium.com/@treeform/dont-use-lockstep-in-rts-games-b40f3dd6fddb) -- A contrarian perspective arguing against lockstep for modern RTS development.

- [SIMNET - Wikipedia](https://en.wikipedia.org/wiki/SIMNET) -- Historical context on early networked military simulation.

- [About StarCraft Networking Model](https://www.moddb.com/features/about-starcraft-networking-model) -- StarCraft's peer-to-peer lockstep implementation details.

- [Cross-Platform RTS Synchronization and Floating Point Indeterminism](https://www.gamedeveloper.com/programming/cross-platform-rts-synchronization-and-floating-point-indeterminism) -- Challenges of cross-platform determinism.
