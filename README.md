# Client-Side Prediction Research

> **Disclaimer:** This content has been "vibe-researched" -- it was generated with the assistance of AI agents performing web research and synthesis. While efforts were made to fetch and cross-reference primary sources, details may be inaccurate, outdated, or misrepresented. Always verify against the original references linked throughout before relying on any technical claims.

A comprehensive reference of practical approaches to client-side prediction in networked games and real-time applications. Each topic has an in-depth deep-dive document with implementation details and code examples.

## Table of Contents

1. [Foundational Literature](#foundational-literature)
2. [Techniques at a Glance](#techniques-at-a-glance)
3. [Game Engine & Framework Implementations](#game-engine--framework-implementations)
4. [Shipped Game Case Studies](#shipped-game-case-studies)
5. [MMO Case Studies](#mmo-case-studies)
6. [Comparison Tables](#comparison-tables)
7. [Curated Resource Lists](#curated-resource-lists)

---

## Foundational Literature

| Document | Author / Origin | Year | Key Contribution |
|----------|----------------|------|-----------------|
| [Valve Source Engine](valve-source-engine.md) | Yahn Bernier / Valve | 2001 | The four pillars: prediction, reconciliation, interpolation, lag compensation |
| [Quake / id Software](quake-id-software.md) | John Carmack | 1996 | Introduced client-side prediction to FPS (QuakeWorld) |
| [Fast-Paced Multiplayer](gabriel-gambetta.md) | Gabriel Gambetta | 2014 | Most-referenced tutorial series with live demos |
| [Gaffer on Games](glenn-fiedler-gaffer-on-games.md) | Glenn Fiedler | 2014-2018 | Lockstep, snapshot interpolation, state sync, priority accumulators |
| [1500 Archers on a 28.8](age-of-empires.md) | Bettner & Terrano | 2001 | Deterministic lockstep for RTS (Age of Empires) |

---

## Techniques at a Glance

| Technique | What it Does | Best For | Trade-off |
|-----------|-------------|----------|-----------|
| **[Client-Side Prediction + Reconciliation](TECHNIQUES/client-side-prediction-reconciliation.md)** | Client applies input locally, replays unacknowledged inputs on server correction | FPS, action games | Complexity; occasional visual snaps |
| **[Entity Interpolation](TECHNIQUES/entity-interpolation.md)** | Renders remote entities between two past server snapshots | All multiplayer games | ~100ms visual delay for remote entities |
| **[Dead Reckoning](TECHNIQUES/dead-reckoning.md)** | Extrapolates position from last known velocity/acceleration | Military sims, vehicles, MMO remote entities | Fails on unpredictable motion |
| **[Rollback Netcode](TECHNIQUES/rollback-netcode.md)** | Saves state, predicts remote inputs, resimulates on misprediction | Fighting games, competitive action | CPU cost; requires determinism + fast serialization |
| **[Deterministic Lockstep](TECHNIQUES/deterministic-lockstep.md)** | All peers run identical simulation from shared inputs | RTS, turn-based | Requires perfect determinism; pauses on slow peers |
| **[Snapshot Interpolation](TECHNIQUES/snapshot-interpolation.md)** | Client buffers and interpolates full server snapshots | Spectators, simple games | High bandwidth or high latency (10pps=350ms, 60pps=85ms) |
| **[Lag Compensation (Rewinding)](TECHNIQUES/lag-compensation.md)** | Server rewinds world to shooter's timestamp for hit detection | FPS with hitscan weapons | "Shot behind walls" from target's perspective |

---

## Game Engine & Framework Implementations

| Framework | Deep-Dive | Prediction Model | Built-in? |
|-----------|-----------|-----------------|-----------|
| Unity Netcode (NGO) | [unity-ngo.md](unity-ngo.md) | Client Anticipation (snap/blend, no replay) | Partial |
| Unreal Engine | [unreal-engine.md](unreal-engine.md) | CMC prediction + GAS PredictionKey + Network Prediction Plugin | Yes (movement); experimental (general) |
| Godot | [godot.md](godot.md) | None built-in; community addons (Netfox, Rollback Netcode, MonkeNet) | No |
| Photon Fusion | [photon-fusion.md](photon-fusion.md) | Full CSP with automatic resimulation + sub-tick lag compensation | Yes |
| Mirror | [mirror-networking.md](mirror-networking.md) | PredictedRigidbody with delta reapplication (no Physics.Simulate) | Experimental |
| FishNet | [fishnet.md](fishnet.md) | Replicate/Reconcile method pattern with state forwarding | Yes |
| Coherence | [coherence.md](coherence.md) | State-based prediction OR GGPO-style rollback (two models) | Yes (state); experimental (rollback) |
| SnapNet | [snapnet.md](snapnet.md) | Full CSP with automatic rollback + lag compensation (multi-engine) | Yes |
| Colyseus | [colyseus.md](colyseus.md) | Manual (shared logic + reconciliation pattern) | No (planned) |

---

## Shipped Game Case Studies

| Game | Deep-Dive | Studio | Architecture | Key Innovation |
|------|-----------|--------|-------------|---------------|
| Overwatch | [overwatch.md](overwatch.md) | Blizzard | Server-authoritative, 62.5 Hz | ECS enabling prediction + replay; predicted projectiles |
| Rocket League | [rocket-league.md](rocket-league.md) | Psyonix | Server-authoritative, 60 Hz | Bullet Physics rollback; all objects predicted (not interpolated) |
| Halo: Reach | [halo-reach.md](halo-reach.md) | Bungie | Host-authoritative | Target-relative gunshots; bandwidth reduced to 20% of Halo 3 |
| For Honor | [for-honor.md](for-honor.md) | Ubisoft | P2P deterministic lockstep | "Time travel" rollback; zero state traffic; deterministic AI |
| Valorant | [valorant.md](valorant.md) | Riot Games | Server-authoritative, 128 Hz | Peeker's advantage formula; Riot Direct backbone |

---

## MMO Case Studies

> MMOs face a fundamentally different networking challenge than action/competitive games. With hundreds or thousands of players per zone, traditional techniques like rollback and lag compensation become computationally infeasible. Most MMOs instead **design their gameplay around network limitations** — using client-authoritative movement, target-locked combat, animation masking, and creative latency tolerance rather than trying to eliminate lag.

| Game | Deep-Dive | Studio | Architecture | Key Innovation |
|------|-----------|--------|-------------|---------------|
| World of Warcraft | [MMO/world-of-warcraft.md](MMO/world-of-warcraft.md) | Blizzard | Hybrid authority, TCP, ~20-50 Hz | Client-auth movement; spell batching (400ms→10ms); melee leeway |
| Final Fantasy XIV | [MMO/final-fantasy-xiv.md](MMO/final-fantasy-xiv.md) | Square Enix | TCP-only, ~3.3 Hz position tick | Slidecast tolerance; snapshot AoE; game design shaped around netcode |
| Guild Wars 2 | [MMO/guild-wars-2.md](MMO/guild-wars-2.md) | ArenaNet | TCP, hybrid authority, ~20-30 Hz | Animation fast-forwarding; 5-target AoE cap as networking constraint |
| Guild Wars 1 | [MMO/guild-wars-1.md](MMO/guild-wars-1.md) | ArenaNet | TCP-only, server-auth, instanced | Zero-downtime rolling upgrades; animation-length latency masking |
| EVE Online | [MMO/eve-online.md](MMO/eve-online.md) | CCP Games | Single-shard, server-auth, 1 Hz | Time Dilation (TiDi) scales simulation to 10% for 6000+ player battles |
| Elder Scrolls Online | [MMO/elder-scrolls-online.md](MMO/elder-scrolls-online.md) | ZeniMax Online | TCP megaserver, ~60 Hz | Megaserver dynamic instancing; AoE message batching for Cyrodiil PvP |
| Black Desert Online | [MMO/black-desert-online.md](MMO/black-desert-online.md) | Pearl Abyss | Server-auth, ~25-30 Hz, seamless | Aggressive client prediction for action combat; accepts desync as tradeoff |
| New World | [MMO/new-world.md](MMO/new-world.md) | Amazon Games | Server-auth on AWS (EC2/DynamoDB) | Hub-based zone grid; 800K DynamoDB writes/30s |

---

## Comparison Tables

### Framework Comparison: Architecture & Prediction

| | Unity NGO | Unreal Engine | Godot | Photon Fusion | Mirror | FishNet | Coherence | SnapNet | Colyseus |
|---|---|---|---|---|---|---|---|---|---|
| **Language** | C# | C++ | GDScript/C# | C# | C# | C# | C# | C/C++ (+ C# / Blueprints) | TypeScript |
| **Platform** | Unity | Unreal | Godot | Unity | Unity | Unity | Unity | Unreal, Unity, Custom (C API) | Any (Node.js) |
| **Authority** | Server | Server | Server | Server/Host/Shared | Server | Server | Server/Client/Hybrid | Server | Server |
| **Sync Model** | State replication | Property replication + RPCs | Property replication | State snapshots | SyncVars + RPCs | State + RPCs | State replication | Snapshot + delta (FSnapNetProperty) | Schema delta patches |
| **Tick System** | NetworkTime | Fixed tick | No fixed net tick | Fixed network tick | Server tick | Fixed tick (TimeManager) | Fixed simulation frame | Fixed tick (multi-world) | patchRate interval |
| **Prediction** | Anticipation (no replay) | CMC replay + GAS keys | None built-in | Full CSP + resimulation | Delta reapplication | Replicate + Reconcile | State-based OR GGPO rollback | Full CSP + automatic rollback | Manual implementation |
| **Reconciliation** | Snap or Smooth | Snap + replay pending moves | N/A | Automatic resimulation | History correction + smoothing | Snap + input replay | Snap/Blend or full rollback | Automatic rewind-resimulate-repredict | Snap + replay pending inputs |
| **Lag Compensation** | No | No (manual via GAS) | No | Yes (hitbox history, sub-tick) | No | No | No | Yes (full entity rewind) | No |
| **Rollback** | No | Experimental (NP Plugin) | Via addons (Netfox, GGPO) | Yes (automatic) | No (delta reapplication) | Yes (via Reconcile) | Yes (GGPO path) | Yes (automatic, multi-world) | No |
| **Determinism Required** | No | Only for NP Plugin | Only for rollback addons | No | No | No | Only for GGPO path | No | No |
| **Physics Prediction** | No | CMC only; NP broken | Via addons | Yes | PredictedRigidbody | PredictionRigidbody | No (Unity physics prohibited in GGPO) | Yes (separate simulation world) | No |
| **Production Ready** | Yes | Yes (CMC/GAS); No (NP) | Addons vary | Yes | Experimental | Yes | Yes (state); No (GGPO) | Yes (commercial) | Yes (manual pattern) |

### Game Comparison: Networking Architecture

| | Overwatch | Rocket League | Halo: Reach | For Honor | Valorant |
|---|---|---|---|---|---|
| **Tick Rate** | 62.5 Hz | 60 Hz | 30 Hz (variable) | Deterministic step | 128 Hz |
| **Architecture** | Client-Server | Client-Server | Host-Client (P2P) | P2P Lockstep → Dedicated relay | Client-Server |
| **Authority** | Server | Server | Host | All peers (deterministic) | Server |
| **Local Player** | Predicted | Predicted | Predicted | Deterministic (local input immediate) | Predicted |
| **Remote Players** | Interpolated | Predicted (with decay) | Interpolated | Deterministic (rollback on late input) | Interpolated |
| **Projectiles** | Predicted (slow); server (fast) | Predicted | Host-authoritative | Deterministic | Server-authoritative |
| **Hit Registration** | Favor-the-shooter + rewind | Server validates | Target-relative + host validates | Deterministic (identical on all peers) | Rewind with distance limits |
| **Reconciliation** | Snap + replay | Physics rollback + resimulation | Host correction + smoothing | Rollback + resimulation ("time travel") | Snap (corrections rare) |
| **Correction Smoothing** | Component-level visual blend | Visual offset decay (lerp/slerp) | Network smoothing mode | Resimulation within budget | Only affects mispredicting player |
| **Bandwidth Strategy** | Delta compression | 60 Hz full state | Priority accumulator (20% of Halo 3) | Zero state traffic (inputs only) | Delta compression + Riot Direct |

### MMO Comparison: Networking Architecture

| | WoW | FFXIV | GW2 | GW1 | EVE Online | ESO | BDO | New World |
|---|---|---|---|---|---|---|---|---|
| **Tick Rate** | ~20-50 Hz (was ~2.5 Hz in 2004) | ~3.3 Hz (position) | ~20-30 Hz (estimated) | Event-driven (TCP) | 1 Hz | ~60 Hz (estimated) | ~25-30 Hz (estimated) | Undisclosed |
| **Architecture** | Zone-partitioned (JAM routing) | Client-Server (TCP only) | Client-Server (TCP, microservices) | Instanced (18 server types) | Single-shard cluster | TCP megaserver | Seamless open-world | AWS EC2 hubs + DynamoDB |
| **Movement Authority** | Client | Client | Client-trusted | Server | Server | Hybrid | Server (client predicts) | Server |
| **Combat Authority** | Server | Server | Server | Server | Server | Server | Server | Server |
| **Remote Players** | Dead reckoning | Low-freq updates + smoothing | Dead reckoning | Server-confirmed (no DR) | Dead reckoning (1 Hz) | Server-confirmed | Client-predicted | Server-confirmed |
| **Hit Registration** | Target-locked + melee leeway | Position snapshot at cast end | Server-side (no rewind) | Server-side range/LoS | Formula-based (no hitboxes) | Server-side AoE checks | Client predicts; server resolves | Server-side hit volumes |
| **Latency Masking** | Spell Queue Window; spell batching | Slidecast window; animation lock | Animation fast-forwarding | Two-stage commit (animation > RTT) | TiDi (slow sim under load) | AoE message batching | Instant client feedback | Standard |
| **Scale** | 100s open world (sharded) | 100s (instanced raids) | 100+ WvW (degrades) | 16 PvP / 150 PvE | 6000+ (under TiDi) | ~600 Cyrodiil | ~300+ siege | ~2500 per world |
| **Transport** | TCP | TCP | TCP | TCP | TCP (optimized) | TCP | Undisclosed | Custom (GridMate) |
| **Key Limitation** | Client-auth movement → cheats | 3.3 Hz position → ghost AoE hits | Single-threaded → WvW skill lag | No DR → visible remote latency | 1 Hz → sluggish feel | TCP HOL blocking in PvP | Desync inherent in design | Launch exploit controversies |

### Technique Comparison: When to Use What

| Technique | Latency Feel | Bandwidth | CPU Cost | Complexity | Determinism Required | Best Genre |
|-----------|-------------|-----------|----------|------------|---------------------|-----------|
| **[CSP + Reconciliation](TECHNIQUES/client-side-prediction-reconciliation.md)** | Excellent (local input instant) | Medium (inputs + state) | Medium (replay N inputs) | High | No | FPS, action |
| **[Entity Interpolation](TECHNIQUES/entity-interpolation.md)** | Good (100ms delay for remotes) | Medium (snapshots) | Low | Low | No | All games (for remote entities) |
| **[Dead Reckoning](TECHNIQUES/dead-reckoning.md)** | Variable (good for smooth motion) | Low (threshold updates) | Low | Medium | No | Vehicles, MMOs, military sims |
| **[Rollback (GGPO)](TECHNIQUES/rollback-netcode.md)** | Excellent (zero input delay) | Low (inputs only) | High (resimulate N frames) | Very High | Yes | Fighting, competitive action |
| **[Deterministic Lockstep](TECHNIQUES/deterministic-lockstep.md)** | Sluggish (input delay = RTT) | Very Low (inputs only) | Low | High (determinism) | Yes | RTS, turn-based |
| **[Snapshot Interpolation](TECHNIQUES/snapshot-interpolation.md)** | Poor-Medium (85-350ms) | High (full snapshots) | Very Low | Low | No | Spectating, casual |
| **[Lag Compensation](TECHNIQUES/lag-compensation.md)** | N/A (server-side) | None (server process) | Medium (rewind + test) | Medium | No | FPS (hit detection) |

### Decision Matrix: Choosing an Approach

| Your Game Type | Recommended Primary Technique | Complement With | Example |
|---------------|------------------------------|----------------|---------|
| **Competitive FPS** | CSP + Reconciliation | Lag Compensation + Entity Interpolation | Valorant, CS2, Overwatch |
| **Physics-based competitive** | CSP + Physics Rollback | Entity Prediction (all objects) | Rocket League |
| **Fighting game** | Rollback (GGPO-style) | Input delay (hybrid) | Guilty Gear Strive, SF6 |
| **RTS (many units)** | Deterministic Lockstep | Checksums | Age of Empires, StarCraft |
| **Co-op / casual** | Snapshot Interpolation | Entity Interpolation | Many indie games |
| **MMO (tab-target)** | Client-auth movement + server-auth combat | Dead reckoning (remotes) + latency masking (animations, spell queue) | World of Warcraft, FFXIV, ESO |
| **MMO (action combat)** | Server-auth + aggressive client prediction | Accept desync as tradeoff; AoE caps for scale | Black Desert Online, New World, GW2 |
| **Melee action (P2P)** | Deterministic Lockstep + Rollback | Deterministic AI | For Honor |
| **Mobile / web** | CSP + Reconciliation (manual) | Delta compression | Colyseus-based games |

---

## Curated Resource Lists

- [Awesome Game Networking (GitHub)](https://github.com/rumaniel/Awesome-Game-Networking)
- [Game Networking Resources -- ThusWroteNomad](https://github.com/ThusWroteNomad/GameNetworkingResources)
- [Multiplayer Networking Resources](https://multiplayernetworking.com/)
- [Web Game Dev: Prediction & Reconciliation](https://www.webgamedev.com/backend/prediction-reconciliation)
- [CrystalOrb (Rust rollback library)](https://github.com/ernwong/crystalorb)
