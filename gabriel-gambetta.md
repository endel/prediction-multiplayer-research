# Gabriel Gambetta's "Fast-Paced Multiplayer" Series

## Overview

Gabriel Gambetta's [Fast-Paced Multiplayer](https://www.gabrielgambetta.com/client-server-game-architecture.html) is a four-part article series (plus a live demo) that has become one of the most widely referenced resources on networked game architecture. The series is popular because it distills the core techniques used by professional game engines (Valve Source, Quake, Overwatch, etc.) into clear, digestible explanations with diagrams and a working JavaScript demo -- all in under 500 lines of code.

The series covers:

| Part | Title | Core Problem Solved |
|------|-------|---------------------|
| I | Client-Server Game Architecture | Why an authoritative server is necessary, and the latency problem it creates |
| II | Client-Side Prediction and Server Reconciliation | Eliminating perceived input lag for the local player |
| III | Entity Interpolation | Smooth rendering of remote entities despite infrequent updates |
| IV | Lag Compensation | Accurate hit detection when players see each other at different points in time |

The reason these articles are so frequently cited is that they build concepts incrementally -- each part introduces a problem created by the previous solution, then solves it -- making the entire architecture feel inevitable rather than arbitrary.

---

## Part I: Client-Server Game Architecture

**Source:** [https://www.gabrielgambetta.com/client-server-game-architecture.html](https://www.gabrielgambetta.com/client-server-game-architecture.html)

### The Problem of Cheating

The series begins with a fundamental principle: **don't trust the player**. In a multiplayer game, a cheating player degrades the experience for everyone else, so the architecture must be designed to prevent cheating at a structural level.

### Authoritative Server Model

The solution is an **authoritative server**: all game state lives on the server. Clients are not trusted with anything -- not health, not position, not inventory. The flow is:

1. Client captures player input (e.g., "press right arrow")
2. Client sends the **input** (not the desired result) to the server
3. Server validates the input, updates the game state
4. Server sends the new game state back to the client
5. Client renders whatever the server says

This prevents cheating because even if a hacked client claims "I'm at position (1000, 1000)", the server knows the player is actually at (10, 10) and ignores the claim. The server only processes movement **inputs**, not position **claims**.

### Dumb Client vs. Smart Client

The "dumb client" approach (send input, wait for server, render result) is correct but creates a serious playability issue: **perceived lag**. If the round-trip time to the server is 100ms, the player sees a 100ms delay between pressing a key and seeing their character move. At 250ms or 500ms (common on the internet), the game becomes unplayable.

The rest of the series is about making the client "smarter" while keeping the server authoritative -- giving the player the **illusion** of instantaneous response without compromising server authority.

### Key Insight

> "The game state is managed by the server alone. Clients send their actions to the server. The server updates the game state periodically, and then sends the new game state back to clients, who just render it on the screen."

---

## Part II: Client-Side Prediction and Server Reconciliation

**Source:** [https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)

This is the most important article in the series and the one most relevant to prediction research.

### Client-Side Prediction

The core observation: **most of the time, the server will accept the client's input and apply it as expected**. If the game world is deterministic, the client can predict what the server will do and render the result immediately, without waiting for confirmation.

**Without prediction (dumb client):**
- t=0: Player presses right arrow
- t=0: Input sent to server
- t=100ms: Server receives input, processes it, sends response
- t=200ms: Client receives response, renders movement

**With prediction:**
- t=0: Player presses right arrow
- t=0: Input sent to server AND client immediately applies the input locally
- t=0: Player sees movement (zero perceived lag)
- t=100ms: Server receives input, processes it, sends response
- t=200ms: Client receives confirmation (position already matches)

### The Synchronization Problem

Prediction alone creates a subtle bug when multiple inputs are in-flight. Consider 250ms lag and two rapid key presses:

1. t=0: Press right (#1). Client predicts x=11. Sends input #1 to server.
2. t=100ms: Press right (#2). Client predicts x=12. Sends input #2 to server.
3. t=250ms: Server responds to #1: "x=11". Client snaps to x=11. **But the client had predicted x=12!**
4. t=350ms: Server responds to #2: "x=12". Client snaps to x=12.

The player sees the character jump backward and then forward again -- completely unacceptable.

### Server Reconciliation with Sequence Numbers

The fix requires two things:

1. **Sequence numbers**: Every input the client sends is tagged with an incrementing sequence number.
2. **Input buffer**: The client keeps a buffer of all sent-but-unacknowledged inputs.

When the server responds, it includes the sequence number of the last input it processed. The client then:

1. Sets its state to the server's authoritative state
2. Discards all inputs from the buffer with sequence number <= the server's acknowledged sequence
3. **Re-applies** all remaining (unacknowledged) inputs on top of the authoritative state

### Step-by-Step Walkthrough

Starting state: Player at x=10, speed=1 unit per input, lag=250ms.

| Time | Event | Server State | Client Prediction | Pending Buffer |
|------|-------|-------------|-------------------|----------------|
| t=0 | Press right, send input #1 | x=10 | x=11 | [#1] |
| t=100 | Press right, send input #2 | x=10 | x=12 | [#1, #2] |
| t=250 | Server replies: x=11, last_processed=#1 | x=11 | **Reconcile**: set x=11, discard #1, replay #2 -> x=12 | [#2] |
| t=350 | Server replies: x=12, last_processed=#2 | x=12 | **Reconcile**: set x=12, discard #2, no replay needed -> x=12 | [] |

The client always shows x=12 from t=100 onward. No jitter, no jumps.

### What If the Prediction Is Wrong?

If another player or a game event changes the outcome (e.g., a wall appeared, or the player was stunned), the server's authoritative state will differ from the prediction. The reconciliation algorithm handles this automatically: it resets to the server state and replays unacknowledged inputs, producing a corrected position. The player may see a small "snap" correction, but this is rare in practice.

---

## Part III: Entity Interpolation

**Source:** [https://www.gabrielgambetta.com/entity-interpolation.html](https://www.gabrielgambetta.com/entity-interpolation.html)

### The Problem

Client-side prediction and server reconciliation solve input lag for the **local** player. But what about **other** players? The client cannot predict what other players will do -- it has no access to their inputs.

With a typical server update rate of 10 times per second (100ms time step), remote entities only get new position data every 100ms. Naively snapping to each new position produces extremely choppy, teleporting movement.

### Server Time Step

Gambetta introduces the concept of the server **time step**: the server does not process inputs instantly. Instead, it queues inputs from all clients and processes them at a fixed rate (e.g., 10 Hz). After each update, it broadcasts the new world state to all clients.

### Dead Reckoning

For entities with predictable movement (e.g., cars at high speed), the client can extrapolate the entity's position using its last known velocity and direction. This is called **dead reckoning**. It works well for racing games but fails completely for games where players can stop, turn, or change direction instantly (e.g., FPS games).

### Entity Interpolation (Rendering "In the Past")

The key insight: instead of trying to predict where a remote entity **will be**, show where it **was**, using real data from the server. The client renders other entities slightly in the past, interpolating between two known positions.

**How it works:**

1. The client maintains a **position buffer** for each remote entity: a timestamped list of positions received from the server.
2. The **render timestamp** is set to `now - one_server_update_interval` (e.g., 100ms in the past).
3. For each remote entity, find the two buffered positions that bracket the render timestamp.
4. Linearly interpolate between them.

**Example with 100ms server updates:**

- t=900: Server sends position A for remote player
- t=1000: Server sends position B for remote player
- t=1000 to t=1100: Client smoothly interpolates the remote player from A to B

The remote player is always shown 100ms "late," but the movement is perfectly smooth and uses only **real** server data (no speculation).

### Consequences

> "Every player sees a slightly different rendering of the game world, because each player sees itself in the **present** but sees the other entities in the **past**."

This is generally imperceptible for most gameplay, but it creates a problem for precision events like shooting -- which is addressed in Part IV.

---

## Part IV: Lag Compensation

**Source:** [https://www.gabrielgambetta.com/lag-compensation.html](https://www.gabrielgambetta.com/lag-compensation.html)

### The Problem

Because of entity interpolation, when you aim at another player and shoot, you're aiming at where they were ~100ms ago, not where they are now. On the server, the target has already moved. Your perfectly aimed headshot misses.

### Server-Side Rewinding

The solution is **lag compensation**: the server reconstructs the game world as it appeared to the shooting player at the moment they fired.

The algorithm:

1. The client sends a "shoot" event with a **timestamp** and weapon aim data.
2. The server receives the event and determines what the world looked like to that client at that timestamp (accounting for the client's latency and the interpolation delay).
3. The server **rewinds** the game state to that point in time.
4. The server performs hit detection against the rewound state.
5. If the shot hits, the server applies damage and updates all clients.

### The Tradeoff

This system is fair for the shooter: if you aim at someone's head and fire, you get the hit. But it introduces a small unfairness for the target: a player who ducks behind a wall might still get hit for a fraction of a second after reaching safety, because the shooter saw them in the open due to latency.

Gambetta argues this tradeoff is worthwhile:

> "It would be much worse to miss an unmissable shot!"

### Summary of the Full Architecture

From the article's own summary:

- Server gets inputs from all clients, with timestamps
- Server processes inputs and updates world state
- Server sends regular world snapshots to all clients
- Client sends input and simulates effects locally
- Client gets world updates and:
  - Syncs predicted state to authoritative state (reconciliation)
  - Interpolates known past states for other entities

The result:
- **Player sees themselves in the present**
- **Player sees other entities in the past**

---

## Implementation Examples

These examples are derived from Gambetta's live demo source code, which is pure JavaScript and runs entirely in the browser.

### The Entity

```javascript
class Entity {
  constructor() {
    this.x = 0;
    this.speed = 2; // units per second
    this.position_buffer = []; // for interpolation of remote entities
  }

  applyInput(input) {
    this.x += input.press_time * this.speed;
  }
}
```

### The Prediction Loop with Sequence Numbers

```javascript
// Client: processInputs()
processInputs() {
  const now_ts = +new Date();
  const last_ts = this.last_ts || now_ts;
  const dt_sec = (now_ts - last_ts) / 1000.0;
  this.last_ts = now_ts;

  // Build the input object
  let input;
  if (this.key_right) {
    input = { press_time: dt_sec };
  } else if (this.key_left) {
    input = { press_time: -dt_sec };
  } else {
    return; // no input this frame
  }

  // Tag with a sequence number and the entity's ID
  input.input_sequence_number = this.input_sequence_number++;
  input.entity_id = this.entity_id;

  // Send to server (through simulated laggy network)
  this.server.network.send(this.lag, input);

  // CLIENT-SIDE PREDICTION: apply input immediately
  if (this.client_side_prediction) {
    this.entities[this.entity_id].applyInput(input);
  }

  // Save for later reconciliation
  this.pending_inputs.push(input);
}
```

Key points:
- Each input gets a monotonically increasing `input_sequence_number`
- The input is sent to the server AND applied locally (prediction)
- The input is saved in `pending_inputs` for reconciliation later

### The Reconciliation Algorithm

```javascript
// Client: processServerMessages()
processServerMessages() {
  while (true) {
    const message = this.network.receive();
    if (!message) break;

    for (const state of message) {
      if (state.entity_id === this.entity_id) {
        // This is OUR entity -- reconcile
        const entity = this.entities[this.entity_id];

        // Step 1: Accept the server's authoritative position
        entity.x = state.position;

        if (this.server_reconciliation) {
          // Step 2: Discard inputs already processed by the server
          let j = 0;
          while (j < this.pending_inputs.length) {
            const input = this.pending_inputs[j];
            if (input.input_sequence_number <= state.last_processed_input) {
              // Server has already accounted for this input -- drop it
              this.pending_inputs.splice(j, 1);
            } else {
              // Server hasn't seen this input yet -- re-apply it
              entity.applyInput(input);
              j++;
            }
          }
        } else {
          // No reconciliation: just accept server state, discard buffer
          this.pending_inputs = [];
        }
      } else {
        // This is a REMOTE entity -- buffer for interpolation
        entity.position_buffer.push([+new Date(), state.position]);
      }
    }
  }
}
```

The reconciliation loop in detail:
1. Set entity position to the server's authoritative value
2. Walk through `pending_inputs`
3. If an input's sequence number is <= `last_processed_input`, the server has already applied it -- remove it from the buffer
4. If an input's sequence number is > `last_processed_input`, the server hasn't seen it yet -- re-apply it on top of the authoritative state

### Entity Interpolation Logic

```javascript
// Client: interpolateEntities()
interpolateEntities() {
  const now = +new Date();
  // Render other entities "in the past" by one server update interval
  const render_timestamp = now - (1000.0 / server.update_rate);

  for (const i in this.entities) {
    const entity = this.entities[i];

    // Skip our own entity (it uses prediction, not interpolation)
    if (entity.entity_id === this.entity_id) continue;

    const buffer = entity.position_buffer;

    // Drop positions older than what we need
    while (buffer.length >= 2 && buffer[1][0] <= render_timestamp) {
      buffer.shift();
    }

    // Interpolate between the two positions that bracket render_timestamp
    if (buffer.length >= 2 &&
        buffer[0][0] <= render_timestamp &&
        render_timestamp <= buffer[1][0]) {

      const x0 = buffer[0][1]; // position at earlier timestamp
      const x1 = buffer[1][1]; // position at later timestamp
      const t0 = buffer[0][0]; // earlier timestamp
      const t1 = buffer[1][0]; // later timestamp

      // Linear interpolation
      const fraction = (render_timestamp - t0) / (t1 - t0);
      entity.x = x0 + (x1 - x0) * fraction;
    }
  }
}
```

Key points:
- `render_timestamp` is set to `now - one_server_interval` (e.g., 100ms in the past)
- The buffer is pruned: old entries that are no longer needed are discarded
- Linear interpolation is performed between the two surrounding snapshots
- This only applies to **remote** entities -- the local entity uses prediction

### Server-Side Processing

```javascript
class Server {
  constructor() {
    this.entities = [];
    this.clients = [];
    this.last_processed_input = []; // per-client sequence tracker
  }

  update() {
    this.processInputs();
    this.sendWorldState();
  }

  validateInput(input) {
    // Reject suspiciously large inputs (anti-cheat)
    return Math.abs(input.press_time) <= 1/40;
  }

  processInputs() {
    while (true) {
      const message = this.network.receive();
      if (!message) break;

      if (this.validateInput(message)) {
        const id = message.entity_id;
        this.entities[id].applyInput(message);
        this.last_processed_input[id] = message.input_sequence_number;
      }
    }
  }

  sendWorldState() {
    const world_state = [];
    for (let i = 0; i < this.clients.length; i++) {
      world_state.push({
        entity_id: this.entities[i].entity_id,
        position: this.entities[i].x,
        last_processed_input: this.last_processed_input[i]
      });
    }
    // Broadcast to all clients (through their simulated laggy networks)
    for (const client of this.clients) {
      client.network.send(client.lag, world_state);
    }
  }
}
```

Critical detail: each world state snapshot includes `last_processed_input` per entity. This is what enables the client to know which inputs have been acknowledged and which still need to be replayed during reconciliation.

### The Live Demo's Approach

The [live demo](https://www.gabrielgambetta.com/client-side-prediction-live-demo.html) runs entirely in the browser with no actual network -- it simulates network lag using a `LagNetwork` class that delays message delivery:

```javascript
class LagNetwork {
  constructor() {
    this.messages = [];
  }

  send(lag_ms, message) {
    this.messages.push({
      recv_ts: +new Date() + lag_ms,
      payload: message
    });
  }

  receive() {
    const now = +new Date();
    for (let i = 0; i < this.messages.length; i++) {
      if (this.messages[i].recv_ts <= now) {
        return this.messages.splice(i, 1)[0].payload;
      }
    }
  }
}
```

The demo lets users toggle prediction, reconciliation, and interpolation independently, and adjust lag and server update rate, to see the effect of each technique in isolation. It uses two "clients" and one "server" all running locally, with configurable simulated lag (default 250ms for Player 1, 150ms for Player 2) and a default server update rate of 3 Hz (intentionally low to exaggerate the effects).

---

## Mapping to Real Engines

### Valve Source Engine

Gambetta's techniques directly correspond to the Source Engine's networking model, documented in Valve's [Latency Compensating Methods](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization):

| Gambetta Concept | Source Engine Equivalent |
|-----------------|------------------------|
| Authoritative server | Source's server is always authoritative |
| Client-side prediction | `cl_predict 1` -- client runs the same movement code as the server |
| Sequence numbers + input buffer | Source tracks `m_nCommandNumber` for each `CUserCmd` |
| Server reconciliation | After receiving server snapshot, Source replays all unacknowledged `CUserCmd`s |
| Entity interpolation | `cl_interp` / `cl_interp_ratio` -- entities rendered in the past (default ~100ms) |
| Lag compensation | Server rewinds hitboxes to the shooter's view time (`sv_maxunlag`) |

### Quake / QuakeWorld

QuakeWorld (1996) was one of the first games to implement client-side prediction. The client ran its own physics simulation and corrected itself when the server disagreed. Gambetta's reconciliation algorithm is a more formalized version of what QuakeWorld did.

### Overwatch (GDC 2017)

Tim Ford and Philip Orwig's GDC talk "Overwatch Gameplay Architecture and Netcode" describes a system that is essentially Gambetta's architecture at scale:
- Deterministic simulation with client-side prediction
- Server reconciliation via command buffer replay
- Entity interpolation for remote players
- Server-side lag compensation with world state rewinding

### General Applicability

These techniques are not engine-specific. Any authoritative client-server game (whether using Unity, Unreal, Godot, or a custom engine) will implement some variation of these four concepts. The specifics vary (snapshot vs. delta compression, UDP vs. WebSocket, fixed vs. variable time step), but the conceptual framework remains the same.

---

## References

1. **Gabriel Gambetta -- Fast-Paced Multiplayer (Full Series)**
   - Part I: Client-Server Game Architecture
     [https://www.gabrielgambetta.com/client-server-game-architecture.html](https://www.gabrielgambetta.com/client-server-game-architecture.html)
   - Part II: Client-Side Prediction and Server Reconciliation
     [https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
   - Part III: Entity Interpolation
     [https://www.gabrielgambetta.com/entity-interpolation.html](https://www.gabrielgambetta.com/entity-interpolation.html)
   - Part IV: Lag Compensation
     [https://www.gabrielgambetta.com/lag-compensation.html](https://www.gabrielgambetta.com/lag-compensation.html)
   - Live Demo
     [https://www.gabrielgambetta.com/client-side-prediction-live-demo.html](https://www.gabrielgambetta.com/client-side-prediction-live-demo.html)

2. **Glenn Fiedler -- What Every Programmer Needs to Know About Game Networking**
   [https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/)

3. **Valve Developer Wiki -- Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization**
   [https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization)

4. **Valve Developer Wiki -- Source Multiplayer Networking**
   [https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)

5. **Tim Ford & Philip Orwig -- Overwatch Gameplay Architecture and Netcode (GDC 2017)**
   [https://www.youtube.com/watch?v=W3aieHjyNvw](https://www.youtube.com/watch?v=W3aieHjyNvw)
