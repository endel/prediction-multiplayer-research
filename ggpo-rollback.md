# GGPO and Rollback Netcode in Fighting Games

## Table of Contents

1. [History](#history)
2. [Why Fighting Games Need Rollback](#why-fighting-games-need-rollback)
3. [How GGPO Works Step by Step](#how-ggpo-works-step-by-step)
4. [The Three Required Game Callbacks](#the-three-required-game-callbacks)
5. [Input Prediction and Decay Strategy](#input-prediction-and-decay-strategy)
6. [Hybrid Approach: Input Delay + Rollback](#hybrid-approach-input-delay--rollback)
7. [Rollback vs Delay-Based Comparison](#rollback-vs-delay-based-comparison)
8. [Delta Rollback Optimization](#delta-rollback-optimization)
9. [GGPO API Overview](#ggpo-api-overview)
10. [Games Using GGPO or Rollback Netcode](#games-using-ggpo-or-rollback-netcode)
11. [Alternative Implementations](#alternative-implementations)
12. [Implementation Pseudo-Code](#implementation-pseudo-code)
13. [References](#references)

---

## History

GGPO (Good Game Peace Out) was created by **Tony Cannon** in **2009**. Cannon is deeply embedded in the fighting game community -- he co-founded **Shoryuken** (shoryuken.com), the premier online fighting game community hub, and is a co-founder of the **Evolution Championship Series (EVO)**, the world's largest and most prestigious fighting game tournament.

Cannon's motivation came directly from frustration with the state of online play in fighting games. Traditional delay-based netcode made competitive online play feel sluggish and fundamentally different from the offline tournament experience. As someone who helped build the competitive fighting game scene, Cannon understood that frame-perfect timing and muscle memory developed offline needed to translate directly to online play.

The initial release of GGPO in late **2006** was designed to test rollback concepts with legacy fighting games running on emulators (such as FightCade for classic arcade titles). Shortly after, Cannon released a version of the library that could be licensed and incorporated into modern commercial games.

In **2019**, Cannon made GGPO **open source** under the **MIT License**, making it freely available for both commercial and non-commercial use. The source code is hosted on GitHub at [pond3r/ggpo](https://github.com/pond3r/ggpo). This move was widely celebrated by the fighting game community and game developers alike, as it removed the licensing barrier for indie studios and enabled the broader adoption of rollback netcode across the genre.

---

## Why Fighting Games Need Rollback

Fighting games are uniquely sensitive to network latency for several reasons:

### Frame-Perfect Inputs

Fighting games run at a locked **60 frames per second**, where each frame is approximately **16.67 milliseconds**. Many techniques require inputs within a window of 1-3 frames (16-50ms). A combo link with a 1-frame window demands the player press a button at exactly the right 16ms interval. Adding even 3-4 frames of network delay makes these precise timings unreliable or impossible.

### Muscle Memory

Competitive fighting game players spend hundreds or thousands of hours developing **muscle memory** for combos, hit-confirms, anti-airs, and reaction-based punishes. This muscle memory is calibrated to the exact timing of offline play. When delay-based netcode adds variable input latency, all of that training is undermined. Players must mentally recalibrate their timing for every match depending on the connection quality -- an unreasonable demand at a competitive level.

### Reactions and Reads

Fighting games rely heavily on **reacting** to what your opponent does. Anti-airing a jump-in, blocking a mix-up, or punishing a whiffed move all depend on seeing the action and responding within a narrow window. Added delay shrinks or eliminates these reaction windows, turning a game built on skill-based reads into guesswork.

### The Offline Standard

Unlike most online game genres, fighting games have a decades-long tradition of **offline, face-to-face competition** at arcades and tournaments. The EVO Championship Series, local weeklies, and arcade culture established an expectation that online play should feel identical to sitting next to your opponent. Rollback netcode is the only networking approach that can deliver this experience over the internet.

As the GGPO README states:

> *"Rollback networking uses input prediction and speculative execution to send player inputs to the game immediately, providing the illusion of a zero-latency network. Using rollback, the same timings, reactions, visual and audio cues, and muscle memory your players build up playing offline will translate directly online."*

---

## How GGPO Works Step by Step

Rollback netcode operates as an extension of lockstep networking, but instead of waiting for remote inputs before advancing the simulation, it **predicts** them and corrects mistakes retroactively. Here is the complete cycle:

### 1. Zero Input Delay for the Local Player

When the local player presses a button, that input is applied to the game state **immediately** on the very next frame. There is no buffering or waiting for the remote player's input to arrive. This is the fundamental advantage: the local player always experiences zero added input latency, making the game feel identical to offline play.

### 2. Input Prediction for the Remote Player

Since the remote player's input for the current frame has not yet arrived over the network, the game **predicts** what the remote player is doing. The standard prediction strategy is simple: assume the remote player is still doing whatever they were doing on the last frame for which real input was received. This works surprisingly well because at 60fps, players rarely change their input from one frame to the next.

### 3. Speculative Execution

With the local player's real input and the remote player's predicted input, the game **advances the simulation** and renders the frame. From the local player's perspective, the game is running at full speed with zero latency. The game continues to advance speculatively, potentially running several frames ahead of confirmed inputs.

### 4. Misprediction Detection

Every frame, the game checks incoming network packets for the remote player's actual inputs. When the real input arrives, the game compares it against what was predicted for that frame:
- **If the prediction was correct**: nothing needs to happen; the speculated game state is already accurate.
- **If the prediction was wrong**: a rollback must occur.

### 5. State Save / Load / Replay Cycle (The Rollback)

When a misprediction is detected, the following sequence executes within a single frame:

```
Frame timeline: ... [4] [5] [6] [7]
                     ^              ^
                     |              |
              Last confirmed    Current frame
              frame (all        (being rendered)
              inputs known)
```

1. **Load State**: Restore the complete game state from the last frame where all inputs were confirmed (e.g., frame 4). This is done via the `load_game_state` callback.

2. **Replay with Corrections**: Starting from the loaded state, re-simulate forward through each frame using:
   - The **real local inputs** that were recorded for each frame
   - The **real remote inputs** for any frame where they have now been received
   - **Predicted remote inputs** for any frame where real inputs still have not arrived

3. **Save States**: After each re-simulated frame, the game state is saved again (via `save_game_state`) in case another rollback is needed.

4. **Render**: Once re-simulation reaches the current frame, the corrected state is rendered to the screen.

All of this -- loading, re-simulating potentially 6-8 frames of game logic, and saving states -- must complete within the budget of a single display frame (~16.67ms at 60fps). This is why rollback implementations demand that game logic be **separable from rendering** and extremely well optimized.

### 6. Time Synchronization

Both clients must stay in sync with each other, running approximately the same frame at the same time. If one client gets ahead, the lagging client will experience excessive one-sided rollbacks. GGPO solves this by briefly pausing the client that is ahead for 1-2 frames, allowing the slower client to catch up. These micro-pauses are imperceptible to players. The `GGPO_EVENTCODE_TIMESYNC` event notifies the game when this correction is needed, providing a `frames_ahead` value.

---

## The Three Required Game Callbacks

To integrate with GGPO (or any rollback system), a game must implement three core callbacks. These are the contract between the networking library and the game engine:

### 1. `save_game_state`

```c
bool save_game_state(unsigned char **buffer, int *len, int *checksum, int frame);
```

The game must serialize its **entire game state** into a buffer. This includes:
- All player positions, velocities, health values, meter
- Active hitboxes and hurtboxes
- Projectile positions and states
- Animation frame indices
- Round timer
- Random number generator state
- Any other gameplay-relevant data

The game allocates the buffer, copies the state into it, and reports the length. Optionally, it can compute a **checksum** for desync detection.

**Critical constraint**: This must be extremely fast. NetherRealm Studios reported spending approximately **2 man-years** optimizing serialization for Mortal Kombat X. Ideally, the game state is stored contiguously in memory so it can be copied with a single `memcpy`.

Non-gameplay elements (audio, particles, UI effects) should **not** be part of the serialized state. Instead, track only flags indicating that a sound or effect should be triggered, and handle playback separately based on rollback results.

### 2. `load_game_state`

```c
bool load_game_state(unsigned char *buffer, int len);
```

The inverse of save: given a previously saved buffer, the game must **restore its complete state** to match that snapshot. After this call returns, the game must be in a state identical to when `save_game_state` produced that buffer. The simulation should be able to resume from this point and produce deterministic results.

### 3. `advance_frame`

```c
bool advance_frame(int flags);
```

Called during rollback re-simulation. The game must advance its game logic by **exactly one frame**, without rendering to the screen, playing audio, or performing any presentation-layer work. Before advancing, the game should call `ggpo_synchronize_input` to get the inputs for this frame (which may be real or predicted, as determined by GGPO). After advancing, the game calls `ggpo_advance_frame` to notify GGPO.

This callback enforces the architectural requirement that **game logic must be fully decoupled from rendering and audio**. The game must be able to run N frames of pure simulation with no side effects.

---

## Input Prediction and Decay Strategy

### Default Prediction: Carry Forward

The standard prediction strategy is to carry the **last known input** forward. If the remote player was holding down-back on the last confirmed frame, the prediction assumes they are still holding down-back. At 60fps, this is correct the vast majority of the time -- players hold directions and buttons for multiple frames.

The GGPO library and most rollback implementations use this approach because the alternative -- trying to predict actions with AI or heuristics -- creates worse outcomes. Incorrectly predicting that a player *did* something (threw a punch, jumped) looks far worse when rolled back than incorrectly predicting they did *nothing*. A predicted action that suddenly aborts or disappears is jarring; a slightly delayed appearance of an action is much more natural.

### Input Decay Strategy

For games with continuous movement (as opposed to the discrete, state-based movement of traditional 2D fighters), an **input decay** strategy can reduce visual artifacts from mispredictions:

| Prediction Frame | Applied Movement |
|------------------|-----------------|
| Frame +1 | 100% of last known input |
| Frame +2 | 66% (2/3) of last known input |
| Frame +3 | 33% (1/3) of last known input |
| Frame +4+ | 0% (assume idle) |

The rationale is that **undershooting then correcting forward** looks much better than **overshooting then snapping back**. If a player was moving right and stops, decayed prediction will show them slowing to a stop and then nudging forward to the correct position. Without decay, the player would overshoot and then visibly rubber-band backward.

**Rocket League** implements this technique for its vehicle physics prediction. It is less relevant for traditional 2D fighting games where characters are either moving or not (digital input), but valuable for 3D games, analog stick movement, or any system with continuous values.

---

## Hybrid Approach: Input Delay + Rollback

Pure zero-delay rollback works well on low-latency connections, but on higher-latency connections, the prediction horizon grows large enough that mispredictions become frequent and visually disruptive. Prediction beyond approximately **100-150ms** (6-9 frames) tends to become unplayable due to the amount of visual inconsistency introduced.

The solution used by virtually all production rollback implementations is to combine a **small, fixed input delay** with rollback:

```
Total latency coverage = input_delay_frames + rollback_frames
```

The input delay absorbs a baseline amount of latency without any visual artifacts (no mispredictions can occur for frames covered by delay), while rollback handles the remainder.

### Typical Configuration

| Network Latency (RTT) | Input Delay | Max Rollback | Player Experience |
|------------------------|-------------|--------------|-------------------|
| 0-50ms | 3 frames | 0 frames | Perfect; feels offline |
| 50-100ms | 3 frames | 1-3 frames | Excellent; rare corrections |
| 100-150ms | 3 frames | 3-6 frames | Good; occasional visual pops |
| 150ms+ | 4+ frames | 6+ frames | Noticeable delay and corrections |

### Why 2-3 Frames of Delay is Acceptable

Most players cannot perceive 2-3 frames (33-50ms) of input delay. Console TVs often add 2-4 frames of display lag on their own. Fighting games have historically shipped with 3-4 frames of built-in input delay even in offline play. By using a small fixed delay, the rollback window is reduced, which means:

- Fewer mispredictions occur
- When mispredictions do occur, fewer frames need to be re-simulated
- The CPU budget for re-simulation is more manageable

GGPO exposes this directly via the API:

```c
// Add 2 frames of input delay for the local player
ggpo_set_frame_delay(session, local_player_handle, 2);
```

---

## Rollback vs Delay-Based Comparison

| Aspect | Delay-Based Netcode | Rollback Netcode |
|--------|-------------------|------------------|
| **Local input latency** | Increases with ping (delay = RTT/2) | Zero (or fixed small amount) |
| **Visual consistency** | Always consistent (no corrections) | Occasional visual pops on misprediction |
| **Feel at low latency (<80ms)** | Slightly sluggish | Identical to offline |
| **Feel at moderate latency (80-150ms)** | Very sluggish, combos drop | Good, minor visual artifacts |
| **Feel at high latency (150ms+)** | Nearly unplayable | Degraded but still functional |
| **CPU cost** | Minimal | High (must re-simulate multiple frames) |
| **Implementation complexity** | Simple | Complex (state serialization, logic separation) |
| **Bandwidth usage** | Low | Slightly higher (input history sent) |
| **Determinism required** | Yes | Yes |
| **Game state serialization** | Not required | Required (fast save/load) |
| **Audio handling** | Simple | Complex (sounds may need cancellation) |
| **Muscle memory preservation** | No (timing changes per match) | Yes (consistent input response) |
| **Offline-to-online skill transfer** | Poor | Excellent |
| **Variable latency handling** | Stutters or freezes | Absorbs spikes via rollback |
| **Spectator support** | Simple | Supported (GGPO has dedicated spectator mode) |
| **Worst-case scenario** | Frozen screen waiting for input | Visual teleporting/popping |

The key insight: delay-based netcode makes **every frame worse** (constant sluggishness), while rollback netcode makes **some frames wrong** (occasional corrections). Players overwhelmingly prefer the latter, especially at competitive levels.

---

## Delta Rollback Optimization

Standard rollback saves and loads the **complete game state** every frame. For games with large or complex states, this becomes a performance bottleneck. **Delta rollback** is an optimization that reduces the cost of state management.

### How Delta Rollback Works

Instead of saving the entire game state each frame, the system tracks only what **changed** between frames:

1. **Base Snapshot**: Save a full game state periodically (e.g., every N frames or at key moments).
2. **Delta Records**: For each subsequent frame, record only the fields/objects that changed from the previous frame.
3. **Rollback via Reverse Deltas**: When rolling back, apply the inverse of the delta changes to walk the state backward to the target frame, then re-simulate forward.

### Advantages

- **Reduced memory usage**: Deltas are typically much smaller than full snapshots, especially in games where most of the state is static on any given frame.
- **Faster save operations**: Only changed data needs to be copied, rather than the entire state.
- **Cache-friendly**: Smaller data footprints reduce cache pressure during the rapid save/load/replay cycle.

### Trade-offs

- **More complex implementation**: Tracking changes requires instrumentation of all state mutations.
- **Load is not always faster**: Applying multiple deltas to reconstruct a state can be slower than loading a single snapshot, depending on the number of frames being rolled back.
- **Debugging difficulty**: Delta-based state reconstruction introduces more potential points of failure for desync bugs.

### Practical Approach

Many implementations use a **hybrid** strategy: maintain a ring buffer of full snapshots (e.g., every 8 frames) and use deltas between them. This bounds the maximum number of deltas that need to be applied during a rollback while still reducing the per-frame save cost.

---

## GGPO API Overview

The GGPO SDK provides a C API (with C++ compatibility) for integrating rollback netcode. The library is built around sessions, players, and callbacks.

### Constants

```c
#define GGPO_MAX_PLAYERS            4
#define GGPO_MAX_PREDICTION_FRAMES  8
#define GGPO_MAX_SPECTATORS        32
```

### Session Lifecycle

#### Starting a Session

```c
GGPOSession *session = NULL;
GGPOSessionCallbacks callbacks;

// Set up callbacks (all must be implemented)
callbacks.begin_game      = on_begin_game;
callbacks.save_game_state = on_save_game_state;
callbacks.load_game_state = on_load_game_state;
callbacks.advance_frame   = on_advance_frame;
callbacks.on_event        = on_event;
callbacks.log_game_state  = on_log_game_state;
callbacks.free_buffer     = on_free_buffer;

GGPOErrorCode result = ggpo_start_session(
    &session,
    &callbacks,
    "MyFightingGame",  // Game name (for logging)
    2,                 // Number of players
    sizeof(uint32_t),  // Input size per player
    7000               // Local UDP port
);
```

#### Adding Players

```c
GGPOPlayer player;
GGPOPlayerHandle player_handle;

// Add local player
player.size = sizeof(GGPOPlayer);
player.type = GGPO_PLAYERTYPE_LOCAL;
player.player_num = 1;
ggpo_add_player(session, &player, &player_handle);

// Add remote player
player.type = GGPO_PLAYERTYPE_REMOTE;
player.player_num = 2;
strcpy(player.u.remote.ip_address, "192.168.1.100");
player.u.remote.port = 7001;
ggpo_add_player(session, &player, &remote_handle);
```

#### Setting Frame Delay (Hybrid Mode)

```c
// Apply 2 frames of input delay to the local player
ggpo_set_frame_delay(session, local_handle, 2);
```

### Per-Frame Operations

#### Main Game Loop Integration

```c
// 1. Give GGPO time to process network packets
ggpo_idle(session, timeout_ms);

// 2. Add local input
uint32_t local_input = read_controller();
ggpo_add_local_input(session, local_handle, &local_input, sizeof(local_input));

// 3. Synchronize inputs (gets local + remote/predicted)
uint32_t inputs[2];
int disconnect_flags;
ggpo_synchronize_input(session, inputs, sizeof(inputs), &disconnect_flags);

// 4. Advance game state using synchronized inputs
advance_game_state(inputs[0], inputs[1]);

// 5. Notify GGPO that the frame is complete
// (GGPO may call save_game_state after this)
ggpo_advance_frame(session);
```

### Callback Implementations

```c
bool on_save_game_state(unsigned char **buffer, int *len, int *checksum, int frame) {
    // Allocate buffer and copy complete game state
    *len = sizeof(GameState);
    *buffer = (unsigned char *)malloc(*len);
    memcpy(*buffer, &game_state, *len);
    // Optional: compute checksum for desync detection
    *checksum = compute_fletcher32((uint16_t *)*buffer, *len / 2);
    return true;
}

bool on_load_game_state(unsigned char *buffer, int len) {
    // Restore game state from buffer
    memcpy(&game_state, buffer, len);
    return true;
}

bool on_advance_frame(int flags) {
    // Advance one frame of game logic WITHOUT rendering
    uint32_t inputs[2];
    int disconnect_flags;
    ggpo_synchronize_input(session, inputs, sizeof(inputs), &disconnect_flags);
    advance_game_state(inputs[0], inputs[1]);
    ggpo_advance_frame(session);
    return true;
}

void on_free_buffer(void *buffer) {
    free(buffer);
}

bool on_event(GGPOEvent *info) {
    switch (info->code) {
    case GGPO_EVENTCODE_CONNECTED_TO_PEER:
        // Peer connected
        break;
    case GGPO_EVENTCODE_SYNCHRONIZING_WITH_PEER:
        // Show sync progress: info->u.synchronizing.count / total
        break;
    case GGPO_EVENTCODE_RUNNING:
        // All peers synchronized, game can begin
        break;
    case GGPO_EVENTCODE_TIMESYNC:
        // Should pause for info->u.timesync.frames_ahead frames
        // to let the remote client catch up
        break;
    case GGPO_EVENTCODE_DISCONNECTED_FROM_PEER:
        // Remote player disconnected
        break;
    case GGPO_EVENTCODE_CONNECTION_INTERRUPTED:
        // Connection quality degraded
        break;
    case GGPO_EVENTCODE_CONNECTION_RESUMED:
        // Connection quality restored
        break;
    }
    return true;
}
```

### Network Statistics

```c
GGPONetworkStats stats;
ggpo_get_network_stats(session, remote_handle, &stats);

printf("Ping: %d ms\n", stats.network.ping);
printf("Send queue: %d\n", stats.network.send_queue_len);
printf("Recv queue: %d\n", stats.network.recv_queue_len);
printf("Bandwidth: %d kbps\n", stats.network.kbps_sent);
printf("Local frames behind: %d\n", stats.timesync.local_frames_behind);
printf("Remote frames behind: %d\n", stats.timesync.remote_frames_behind);
```

### Sync Testing

GGPO provides a dedicated sync test mode that runs every frame twice (once with prediction, once with verification) to detect determinism bugs:

```c
ggpo_start_synctest(&session, &callbacks, "TestGame", 2, sizeof(uint32_t), 1);
```

If the checksums from `save_game_state` do not match between the predicted and verified runs, the test aborts -- immediately revealing non-deterministic code paths.

### Spectator Mode

```c
ggpo_start_spectating(
    &session, &callbacks, "MyFightingGame",
    2,                  // num_players
    sizeof(uint32_t),   // input_size
    7002,               // local_port
    "192.168.1.50",     // host_ip
    7000                // host_port
);
```

### Cleanup

```c
ggpo_close_session(session);
```

---

## Games Using GGPO or Rollback Netcode

### Games Using the GGPO Library Directly

- **Skullgirls** (Lab Zero Games, 2012) -- one of the earliest modern fighting games to adopt GGPO; integrated in approximately 2 weeks
- **Street Fighter III: 3rd Strike Online Edition** (Iron Galaxy, 2011) -- Tony Cannon consulted directly
- **Guilty Gear XX Accent Core +R** (Arc System Works, 2020 rollback patch) -- community-driven GGPO integration
- **Them's Fightin' Herds** (Mane6, 2020)
- **Tough Love Arena** (2020) -- browser-based fighting game using GGPO concepts
- **Blazing Strike** (RareBreed Makes Games)
- **Diesel Legacy** (KillSwitch Engage)

### Games Using Custom Rollback Implementations

- **Mortal Kombat X / XL** (NetherRealm Studios, 2015/2016) -- retrofitted with rollback post-launch, ~8 man-years of effort
- **Mortal Kombat 11** (NetherRealm Studios, 2019)
- **Injustice 2** (NetherRealm Studios, 2017)
- **Killer Instinct** (Iron Galaxy / Double Helix, 2013) -- custom in-house rollback, widely praised
- **Brawlhalla** (Blue Mammoth Games, 2017)
- **Power Rangers: Battle for the Grid** (nWay, 2019)
- **Street Fighter V** (Capcom, 2016) -- rollback implementation with known issues around clock synchronization
- **Street Fighter 6** (Capcom, 2023) -- significantly improved rollback over SFV
- **Guilty Gear -Strive-** (Arc System Works, 2021) -- custom rollback for 3D-rendered 2D fighter
- **The King of Fighters XV** (SNK, 2022)
- **DNF Duel** (Arc System Works / Eighting / Neople, 2022)
- **MultiVersus** (Player First Games, 2022)
- **Nickelodeon All-Star Brawl** (Ludosity / Fair Play Labs, 2021)
- **Rivals of Aether** (Dan Fornace, 2022 rollback patch) -- converted from delay-based after extended development effort
- **Melty Blood: Type Lumina** (French-Bread, 2021)
- **Under Night In-Birth II** (French-Bread, 2024)
- **Tekken 8** (Bandai Namco, 2024)
- **Granblue Fantasy Versus: Rising** (Arc System Works, 2023)
- **Mortal Kombat 1** (NetherRealm Studios, 2023)

### Non-Fighting Games Using Rollback

- **Rocket League** (Psyonix, 2015) -- uses rollback with input decay for vehicle physics
- **For Honor** (Ubisoft, 2017) -- transitioned from peer-to-peer to dedicated servers with rollback elements
- **Dungeons & Dragons: Chronicles of Mystara** (Iron Galaxy, 2013) -- 4-player beat-em-up with GGPO

---

## Alternative Implementations

### CrystalOrb (Rust)

An open-source rollback networking library written in **Rust**, designed for use in Rust game engines (such as Bevy or Amethyst). CrystalOrb implements the core rollback loop with a focus on type safety and Rust's ownership model for state management.

- **Repository**: [github.com/ErnWong/crystalorb](https://github.com/ErnWong/crystalorb)
- **Language**: Rust
- **Key features**: Generic over game state types, built-in clock sync, snapshot + rollback architecture

### Godot Rollback Netcode

A rollback networking addon for the **Godot Engine**, providing GGPO-style rollback for Godot games written in GDScript or C#.

- **Repository**: [github.com/dsnopek/godot-rollback-netcode](https://github.com/dsnopek/godot-rollback-netcode)
- **Engine**: Godot 3.x / 4.x
- **Key features**: Integration with Godot's scene tree, state serialization helpers, input manager, SyncTest mode

### GGRS (Good Game Rollback System)

A **Rust** reimplementation of GGPO, designed as a direct port of the GGPO design to Rust with idiomatic API design.

- **Repository**: [github.com/gschup/ggrs](https://github.com/gschup/ggrs)
- **Language**: Rust
- **Key features**: Compatible with Bevy ECS via `bevy_ggrs`, supports P2P and spectators, WASM-compatible

### Backroll

Another Rust-native rollback networking library, focusing on async I/O integration.

- **Repository**: [github.com/HouraiTeahouse/backroll-rs](https://github.com/HouraiTeahouse/backroll-rs)
- **Language**: Rust
- **Key features**: Async transport layer, compatible with tokio, session-based API similar to GGPO

### Unity Netcode for GameObjects + Prediction

Unity's first-party networking solution includes client-side prediction and reconciliation features that can be used to implement rollback-style netcode, though it is not a drop-in GGPO replacement.

### Photon Quantum (Unity)

A commercial deterministic ECS framework with built-in rollback/prediction for Unity. Used in production multiplayer games.

### netsim (JavaScript/TypeScript)

Lightweight rollback implementations exist for web-based games, often used with WebRTC data channels for peer-to-peer connectivity.

---

## Implementation Pseudo-Code

The following pseudo-code demonstrates a complete rollback game loop, illustrating how all the concepts fit together in practice.

```
// ============================================================
// ROLLBACK GAME LOOP - COMPLETE PSEUDO-CODE
// ============================================================

const FRAME_RATE         = 60
const FRAME_DURATION_MS  = 1000 / FRAME_RATE          // ~16.67ms
const MAX_ROLLBACK       = 8                           // Max prediction frames
const INPUT_DELAY        = 2                           // Hybrid delay frames

// ----- State Management -----

struct GameState:
    player_positions[2]
    player_velocities[2]
    player_health[2]
    player_animation_frame[2]
    active_projectiles[]
    round_timer
    rng_state

state_buffer     = RingBuffer<GameState>(capacity: MAX_ROLLBACK + INPUT_DELAY + 2)
input_history    = RingBuffer<InputPair>(capacity: 128)
current_frame    = 0
last_confirmed   = -1     // Last frame with all real inputs received

// ----- Serialization -----

function save_state(frame):
    snapshot = deep_copy(game_state)
    state_buffer.store(frame, snapshot)

function load_state(frame):
    game_state = deep_copy(state_buffer.get(frame))

// ----- Input Prediction -----

function predict_remote_input(frame):
    // Carry forward the last known real input
    last_real = input_history.get_last_confirmed_remote()
    if last_real != null:
        return last_real
    else:
        return INPUT_NEUTRAL   // No input (idle)

// ----- Main Game Loop -----

function game_loop():
    while game_is_running:
        frame_start = now()

        // --- Step 1: Process network messages ---
        received_inputs = network.poll()

        for (remote_frame, remote_input) in received_inputs:
            input_history.set_confirmed_remote(remote_frame, remote_input)
            // Update the last confirmed frame
            // (the latest frame where ALL inputs are known)
            update_last_confirmed()

        // --- Step 2: Check for rollback ---
        needs_rollback = false
        rollback_frame = current_frame   // Will be set to earliest mispredicted frame

        for (remote_frame, remote_input) in received_inputs:
            predicted = input_history.get_predicted_remote(remote_frame)
            if predicted != remote_input:
                needs_rollback = true
                rollback_frame = min(rollback_frame, remote_frame)

        // --- Step 3: Perform rollback if needed ---
        if needs_rollback:
            frames_to_resimulate = current_frame - rollback_frame

            if frames_to_resimulate > MAX_ROLLBACK:
                // Too many frames -- clamp and accept the desync
                frames_to_resimulate = MAX_ROLLBACK
                rollback_frame = current_frame - MAX_ROLLBACK

            // Load the state from the rollback target frame
            load_state(rollback_frame)

            // Re-simulate from rollback_frame to current_frame
            for f in range(rollback_frame, current_frame):
                local_input  = input_history.get_confirmed_local(f)
                remote_input = input_history.get_remote_or_predict(f)

                simulate_frame(local_input, remote_input)
                save_state(f + 1)

            // game_state is now corrected up to current_frame

        // --- Step 4: Read local input and apply delay ---
        raw_local_input = read_controller()
        delayed_frame   = current_frame + INPUT_DELAY
        input_history.set_confirmed_local(delayed_frame, raw_local_input)

        // --- Step 5: Send local input to remote peer ---
        // Include input + frame number + recent input history for redundancy
        network.send(InputMessage {
            frame: delayed_frame,
            input: raw_local_input,
            // Send last N inputs for packet loss resilience
            history: input_history.get_local_range(
                delayed_frame - 5, delayed_frame
            )
        })

        // --- Step 6: Advance current frame ---
        local_input  = input_history.get_confirmed_local(current_frame)
        remote_input = input_history.get_remote_or_predict(current_frame)

        simulate_frame(local_input, remote_input)
        current_frame += 1
        save_state(current_frame)

        // --- Step 7: Render ---
        render(game_state)

        // --- Step 8: Time sync ---
        // If we are ahead of the remote, pause briefly
        if remote_frames_behind > TIMESYNC_THRESHOLD:
            sleep(1 frame)

        // --- Step 9: Frame pacing ---
        elapsed = now() - frame_start
        if elapsed < FRAME_DURATION_MS:
            sleep(FRAME_DURATION_MS - elapsed)


// ----- Simulation (pure logic, no side effects) -----

function simulate_frame(local_input, remote_input):
    // Apply inputs to player states
    apply_input(game_state.player[0], local_input)
    apply_input(game_state.player[1], remote_input)

    // Update physics
    update_positions(game_state)
    update_projectiles(game_state)

    // Resolve collisions and hits
    check_hit_detection(game_state)

    // Advance animations
    advance_animations(game_state)

    // Update timer
    game_state.round_timer -= 1


// ----- Helper -----

function update_last_confirmed():
    // Walk forward from the last confirmed frame
    // and find the latest frame where both inputs are known
    f = last_confirmed + 1
    while input_history.has_confirmed_local(f)
      and input_history.has_confirmed_remote(f):
        last_confirmed = f
        f += 1
```

### Performance Budget

For a 60fps game supporting up to 8 frames of rollback:

```
Base frame budget:     16.67ms

Worst case per frame:
  - Normal game logic:     ~2ms
  - State save:            ~0.5ms
  - Rollback load:         ~0.5ms
  - Re-simulate 8 frames:  8 x 2ms = 16ms  (game logic only, no rendering)
  - 8 state saves:          8 x 0.5ms = 4ms
  ----------------------------------------
  Total worst case:        ~23ms  (OVER BUDGET)

This is why optimization is critical.
Targets for competitive implementations:
  - Game logic per frame:  < 1ms
  - State save/load:       < 0.2ms
```

This illustrates why NetherRealm's initial naive rollback implementation more than tripled MKX's per-frame cost from 10ms to 32ms, and why extensive optimization was required across nearly every in-game system.

---

## References

1. **GGPO Official Website**
   https://www.ggpo.net

2. **GGPO Source Code (GitHub)**
   https://github.com/pond3r/ggpo

3. **Snapnet - Netcode Architectures Part 2: Rollback**
   https://www.snapnet.dev/blog/netcode-architectures-part-2-rollback/

4. **Infil - Fightin' Words: The Technical Side of Rollback Netcode**
   https://words.infil.net/w02-netcode-p5.html

5. **Tony Cannon - Rollback Networking in GGPO (Game Developer Magazine)**
   http://mauve.mizuumi.net/2012/07/05/understanding-fighting-game-networking/

6. **GGPO Developer Guide**
   https://github.com/pond3r/ggpo/blob/master/doc/DeveloperGuide.md

7. **GDC Vault - Rollback Networking Talks**
   https://www.gdcvault.com

8. **Michael Stallone (NetherRealm Studios) - Rollback Implementation in MKX**
   Referenced in Infil's Fightin' Words (see reference 4)

9. **GGRS - Good Game Rollback System (Rust)**
   https://github.com/gschup/ggrs

10. **CrystalOrb (Rust Rollback Library)**
    https://github.com/ErnWong/crystalorb

11. **Godot Rollback Netcode Addon**
    https://github.com/dsnopek/godot-rollback-netcode

12. **Backroll (Rust)**
    https://github.com/HouraiTeahouse/backroll-rs
