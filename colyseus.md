# Colyseus: State Synchronization and Client-Side Prediction

## 1. Overview

[Colyseus](https://colyseus.io/) is an authoritative multiplayer framework for Node.js, written in TypeScript. It provides a room-based architecture where all game logic runs on the server, preventing cheating by design. Clients connect to rooms, send input messages, and receive authoritative state updates via binary delta-encoded patches.

Key characteristics:

- **Authoritative server model** -- the server owns the game state; clients cannot mutate it directly.
- **Schema-based state synchronization** -- a decorator-driven system (`@colyseus/schema`) that automatically tracks mutations and encodes only changed properties.
- **Room-based architecture** -- each game session is an isolated `Room` instance with its own state, lifecycle, and connected clients.
- **Multi-platform SDKs** -- TypeScript/JavaScript, Unity (C#), Godot, Defold, Construct 3, Haxe, Cocos Creator, and a native C library.
- **Built-in matchmaking, reconnection, monitoring, and load testing tools.**

Colyseus currently sits at 6.8k+ GitHub stars with 58+ contributors and active development toward v1.0.

---

## 2. Schema-Based State Synchronization

### 2.1 The `@type` Decorator

State structures are plain classes that extend `Schema`. Only fields decorated with `@type()` participate in synchronization. Undecorated fields remain local to the server.

```typescript
import { Schema, type, MapSchema } from "@colyseus/schema";

export class Player extends Schema {
    @type("number") x: number;
    @type("number") y: number;
    @type("number") tick: number;

    // Not synchronized -- server-only
    inputQueue: InputData[] = [];
}

export class MyRoomState extends Schema {
    @type("number") mapWidth: number;
    @type("number") mapHeight: number;
    @type({ map: Player }) players = new MapSchema<Player>();
}
```

**Primitive types:** `"string"`, `"number"`, `"boolean"`, `"float32"`, `"float64"`, `"int8"`, `"int16"`, `"int32"`, `"int64"`, `"uint8"`, `"uint16"`, `"uint32"`, `"uint64"`, `"bigInt64"`, `"bigUint64"`.

**Complex types:**

| Type | Declaration | Description |
|------|-------------|-------------|
| Nested Schema | `@type(Player)` | Child Schema instance |
| ArraySchema | `@type(["string"])` | Ordered list |
| MapSchema | `@type({ map: Player })` | String-keyed dictionary |
| SetSchema | `@type({ set: Effect })` | Unique value collection |
| CollectionSchema | `@type({ collection: Item })` | Auto-indexed collection |

**Constraints:**

- Maximum of 64 synchronizable fields per Schema class.
- MapSchema keys must be strings.
- Multi-dimensional arrays are not supported.
- Encoder and decoder must share identical schema definitions with matching field ordering.

**TypeScript configuration requirements:**

```json
{
    "compilerOptions": {
        "experimentalDecorators": true,
        "useDefineForClassFields": false
    }
}
```

A JavaScript alternative exists for projects that cannot use decorators:

```javascript
import { schema } from "@colyseus/schema";

const MyState = schema({
    currentTurn: "string",
    players: { map: Player }
});
```

### 2.2 The ChangeTree System

Every Schema instance has an associated `ChangeTree` object (stored under the `$changes` symbol). The ChangeTree is a linked-list-based data structure that tracks which property indexes have been mutated since the last synchronization cycle.

When a decorated property is assigned, the `$track` interceptor calls `changeTree.change()` with the field index and the operation type (e.g., ADD, REPLACE, DELETE). This enqueues the field index into a `ChangeSet` -- an object mapping field indexes to operation codes.

The ChangeTree forms a hierarchy: child Schema instances are linked to their parent's tree via `ChangeTreeNode` references. When a child changes, the change propagates up the tree so the root encoder knows which subtrees have mutations.

Key types from the source:

```typescript
export interface ChangeSet {
    indexes: { [index: number]: number };
    operations: number[];
    queueRootNode?: ChangeTreeNode;
}

export interface ChangeTreeNode {
    changeTree: ChangeTree;
    next?: ChangeTreeNode;
    prev?: ChangeTreeNode;
    position: number;
}
```

### 2.3 Delta Encoding at the Property Level

Colyseus does not send full snapshots on every update. Instead, the `Encoder` walks the ChangeTree and encodes only properties that have been modified since the last patch. Each Schema instance is identified by a unique `refId`, and each field by its positional index within the class definition.

The wire format is a compact binary encoding. Variable-length integer types (`varInt`, `varUint`, `varFloat32`, `varFloat64`) reduce bandwidth further by encoding small values in fewer bytes.

The encoding flow:

1. **Incremental patches** -- `encoder.encode()` iterates over all `ChangeSet` entries in the ChangeTree linked list, encoding only modified fields.
2. **Full state encode** -- `encoder.encodeAll()` encodes every field of every Schema instance, used only when a client first joins.
3. **View-filtered encode** -- `encoder.encodeAllView()` encodes a per-client subset when `StateView` filtering is active.

### 2.4 `patchRate` and How Patches Are Batched

The `patchRate` property on a Room controls how frequently state patches are broadcast to connected clients. It defaults to **50 milliseconds (20 fps)**.

```typescript
export class MyRoom extends Room {
    // Default: patches sent 20 times per second
    patchRate: number = 50;
}
```

Between patch intervals, any number of state mutations can occur (e.g., from message handlers, simulation intervals, or timers). The ChangeTree accumulates all mutations, and **only the latest value of each property is encoded** when `broadcastPatch()` fires. If a property changes 10 times between patches, only the final value is sent.

The `broadcastPatch()` method:

1. Calls `onBeforePatch()` if defined, allowing last-moment state mutations.
2. Ticks the room clock (if no simulation interval is running).
3. Calls `this._serializer.applyPatches(this.clients, this.state)`, which encodes changes and sends them to each client.
4. Dequeues any messages marked with `afterNextPatch`.

Setting `patchRate = null` disables automatic patching; you then call `this.broadcastPatch()` manually.

### 2.5 Full State on Join, Then Incremental Patches

When a client joins a room:

1. The server sends a **handshake** containing Schema type definitions (via `Reflection.encode()`), so the client knows how to decode the binary format. This step is skipped on reconnection if the client has cached the schema classes.
2. The server sends the **full encoded state** via `encoder.encodeAll()`.
3. From that point on, the client receives only **incremental delta patches** at every `patchRate` interval.

This design minimizes bandwidth: a typical ongoing patch might be just a few bytes (a refId, a field index, and the new value), while the full state could be kilobytes.

### 2.6 Schema Inheritance and Backwards Compatibility

Collection types support polymorphism. A `MapSchema<Item>` can hold `Weapon` instances if `Weapon extends Item`. The encoder transmits the concrete type so the decoder instantiates the correct subclass.

For schema evolution, the `@deprecated()` decorator preserves field ordering when removing old fields:

```typescript
class MyState extends Schema {
    @deprecated() @type("string") oldField: string;
    @type("string") newField: string;
}
```

New fields must always be appended to the end of the class to maintain compatibility across client versions.

### 2.7 State View (Per-Client Filtering)

`StateView` allows per-client control over which Schema instances and fields are visible:

```typescript
import { StateView } from "@colyseus/schema";

class Player extends Schema {
    @type("string") name: string;           // visible to all
    @view() @type("number") health: number; // only if view.add(player)
    @view(1) @type("number") secret: number; // only if view.add(player, 1)
}

// In onJoin:
client.view = new StateView();
client.view.add(playerInstance);       // reveals @view() fields
client.view.add(playerInstance, 1);    // reveals @view(1) fields too
```

---

## 3. Client-Side Callbacks: `onAdd`, `onRemove`, `onChange`, `listen`

On the client, state changes are consumed through a callback system accessed via `Callbacks.get(room)`.

### 3.1 `onAdd` -- Collection Item Added

Fires when a new instance is added to a MapSchema, ArraySchema, CollectionSchema, or SetSchema. By default, it is called immediately for items that already exist when the callback is registered.

```typescript
import { Callbacks } from "@colyseus/sdk";

const callbacks = Callbacks.get(room);

callbacks.onAdd("players", (player, sessionId) => {
    console.log(`Player joined: ${sessionId}`);

    // Nest further listeners on the player instance
    callbacks.listen(player, "hp", (current, previous) => {
        console.log(`HP: ${previous} -> ${current}`);
    });
});
```

### 3.2 `onRemove` -- Collection Item Removed

Fires when an item is removed from a collection. Receives the removed item and its key.

```typescript
callbacks.onRemove("players", (player, sessionId) => {
    console.log(`Player left: ${sessionId}`);
    // Clean up sprites, UI elements, etc.
});
```

### 3.3 `onChange` -- Any Property Changed on a Schema Instance

Invokes when any direct `@type` property of a specific Schema instance changes. It does **not** cascade to nested Schema children.

```typescript
callbacks.onChange(player, () => {
    // player.x, player.y, etc. are already updated
    sprite.x = player.x;
    sprite.y = player.y;
});
```

### 3.4 `listen` -- Single Property Change

Listens for changes to a single named property. Returns an unbind function.

```typescript
const unbind = callbacks.listen(player, "currentTurn", (currentValue, previousValue) => {
    console.log(`Turn changed: ${previousValue} -> ${currentValue}`);
});

// Later: stop listening
unbind();
```

### 3.5 `bindTo` -- Automatic Property Binding

Directly maps Schema property changes to a target object (TypeScript only):

```typescript
callbacks.bindTo(player, sprite, ["x", "y"]);
// sprite.x and sprite.y auto-update when player.x/y change
```

---

## 4. Client-Side Prediction: Manual Approach

Colyseus does not (as of early 2026) provide a built-in client-side prediction system. The framework's official stance is that prediction is game-specific logic that developers implement manually using Colyseus's primitives. The [official Phaser tutorial](https://docs.colyseus.io/learn/tutorial/phaser/client-predicted-input) walks through a progressive four-part implementation.

### 4.1 The Core Idea

Client-side prediction reduces **perceived latency** by giving the local player immediate visual feedback for their own inputs, without waiting for the server round-trip. The server remains authoritative -- the client's local state is a "guess" that gets corrected when the server's response arrives.

The pattern has three pillars:

1. **Shared movement logic** -- identical simulation code runs on both client and server.
2. **Immediate local application** -- the client applies input to its local entity instantly.
3. **Server reconciliation** -- when the server's authoritative state arrives, the client corrects its position.

### 4.2 Shared Movement Logic

Since both client and server are TypeScript, the same movement function (or at minimum, the same constants and rules) can be used in both environments:

```typescript
// Shared constants
const VELOCITY = 2;

// Shared movement application (used on both client and server)
function applyInput(entity: { x: number; y: number }, input: InputData) {
    if (input.left) {
        entity.x -= VELOCITY;
    } else if (input.right) {
        entity.x += VELOCITY;
    }
    if (input.up) {
        entity.y -= VELOCITY;
    } else if (input.down) {
        entity.y += VELOCITY;
    }
}
```

### 4.3 Applying Input Locally While Sending to Server

On each tick, the client:

1. Reads the current keyboard state.
2. Sends the input to the server via `room.send()`.
3. Immediately applies the same input to the local player entity.

```typescript
// Client-side (from the Phaser tutorial, Part 3)
update(time: number, delta: number): void {
    if (!this.currentPlayer) { return; }

    const velocity = 2;
    this.inputPayload.left = this.cursorKeys.left.isDown;
    this.inputPayload.right = this.cursorKeys.right.isDown;
    this.inputPayload.up = this.cursorKeys.up.isDown;
    this.inputPayload.down = this.cursorKeys.down.isDown;

    // Send input to server
    this.room.send(0, this.inputPayload);

    // Apply locally for instant feedback
    if (this.inputPayload.left) {
        this.currentPlayer.x -= velocity;
    } else if (this.inputPayload.right) {
        this.currentPlayer.x += velocity;
    }
    if (this.inputPayload.up) {
        this.currentPlayer.y -= velocity;
    } else if (this.inputPayload.down) {
        this.currentPlayer.y += velocity;
    }
}
```

The local player is **excluded from interpolation** -- only remote players get lerped:

```typescript
for (let sessionId in this.playerEntities) {
    if (sessionId === this.room.sessionId) {
        continue; // Skip local player -- we predict it
    }
    const entity = this.playerEntities[sessionId];
    const { serverX, serverY } = entity.data.values;
    entity.x = Phaser.Math.Linear(entity.x, serverX, 0.2);
    entity.y = Phaser.Math.Linear(entity.y, serverY, 0.2);
}
```

### 4.4 Visual Debugging with Remote Reference

The tutorial introduces a `remoteRef` rectangle that shows where the server thinks the player is, compared to where the client has predicted:

```typescript
if (sessionId === this.room.sessionId) {
    this.currentPlayer = entity;

    // Green outline = local predicted position
    this.localRef = this.add.rectangle(0, 0, entity.width, entity.height);
    this.localRef.setStrokeStyle(1, 0x00ff00);

    // Red outline = server authoritative position
    this.remoteRef = this.add.rectangle(0, 0, entity.width, entity.height);
    this.remoteRef.setStrokeStyle(1, 0xff0000);

    callbacks.onChange(player, () => {
        this.remoteRef.x = player.x;
        this.remoteRef.y = player.y;
    });
}
```

### 4.5 Input Sequence Numbering (Tick-Based)

Part 4 of the tutorial introduces a **fixed tick rate** with input sequence numbers. Each input message carries a `tick` number that the server echoes back after processing. This is the foundation for reconciliation.

**Client-side tick counter:**

```typescript
inputPayload: InputData = {
    left: false,
    right: false,
    up: false,
    down: false,
    tick: undefined,
};

currentTick: number = 0;
elapsedTime = 0;
fixedTimeStep = 1000 / 60;

update(time: number, delta: number): void {
    if (!this.currentPlayer) { return; }
    this.elapsedTime += delta;
    while (this.elapsedTime >= this.fixedTimeStep) {
        this.elapsedTime -= this.fixedTimeStep;
        this.fixedTick(time, this.fixedTimeStep);
    }
}

fixedTick(time: number, delta: number) {
    this.currentTick++;

    this.inputPayload.left = this.cursorKeys.left.isDown;
    this.inputPayload.right = this.cursorKeys.right.isDown;
    this.inputPayload.up = this.cursorKeys.up.isDown;
    this.inputPayload.down = this.cursorKeys.down.isDown;
    this.inputPayload.tick = this.currentTick;  // Attach tick number

    this.room.send(0, this.inputPayload);

    // Apply locally (same velocity, same logic as server)
    const velocity = 2;
    if (this.inputPayload.left) {
        this.currentPlayer.x -= velocity;
    } else if (this.inputPayload.right) {
        this.currentPlayer.x += velocity;
    }
    if (this.inputPayload.up) {
        this.currentPlayer.y -= velocity;
    } else if (this.inputPayload.down) {
        this.currentPlayer.y += velocity;
    }
}
```

**Server-side tick echo:**

```typescript
export class Part4Room extends Room {
    state = new MyRoomState();
    fixedTimeStep = 1000 / 60;

    messages = {
        0: (client: Client, input: InputData) => {
            const player = this.state.players.get(client.sessionId);
            player.inputQueue.push(input);
        }
    }

    onCreate(options: any) {
        let elapsedTime = 0;
        this.setSimulationInterval((deltaTime) => {
            elapsedTime += deltaTime;
            while (elapsedTime >= this.fixedTimeStep) {
                elapsedTime -= this.fixedTimeStep;
                this.fixedTick(this.fixedTimeStep);
            }
        });
    }

    fixedTick(timeStep: number) {
        const velocity = 2;
        this.state.players.forEach(player => {
            let input: InputData;
            while (input = player.inputQueue.shift()) {
                if (input.left) player.x -= velocity;
                else if (input.right) player.x += velocity;
                if (input.up) player.y -= velocity;
                else if (input.down) player.y += velocity;

                player.tick = input.tick; // Echo the tick number back
            }
        });
    }
}
```

The `player.tick` field is `@type("number")`, so it gets synchronized back to the client. The client can then compare `player.tick` against its local `currentTick` to determine how many ticks behind the server is and which inputs have been acknowledged.

### 4.6 Reconciliation When Server State Arrives

The Colyseus Phaser tutorial establishes the building blocks (tick numbering, shared movement logic, local prediction) but leaves full reconciliation as an exercise. The standard reconciliation algorithm, following the Gabriel Gambetta pattern, works as follows with Colyseus:

```typescript
// Conceptual reconciliation implementation for Colyseus
class PredictionSystem {
    private pendingInputs: InputData[] = [];
    private currentTick = 0;

    fixedTick() {
        this.currentTick++;
        const input: InputData = {
            left: this.cursorKeys.left.isDown,
            right: this.cursorKeys.right.isDown,
            up: this.cursorKeys.up.isDown,
            down: this.cursorKeys.down.isDown,
            tick: this.currentTick,
        };

        // Send to server
        this.room.send(0, input);

        // Apply locally
        applyInput(this.currentPlayer, input);

        // Save for potential replay
        this.pendingInputs.push(input);
    }

    onServerUpdate(serverPlayer: { x: number; y: number; tick: number }) {
        // Discard all inputs the server has already processed
        this.pendingInputs = this.pendingInputs.filter(
            input => input.tick > serverPlayer.tick
        );

        // Snap to server-authoritative position
        this.currentPlayer.x = serverPlayer.x;
        this.currentPlayer.y = serverPlayer.y;

        // Re-apply unacknowledged inputs
        for (const input of this.pendingInputs) {
            applyInput(this.currentPlayer, input);
        }
    }
}
```

The `onChange` callback is the trigger for reconciliation:

```typescript
callbacks.onChange(player, () => {
    predictionSystem.onServerUpdate({
        x: player.x,
        y: player.y,
        tick: player.tick,
    });
});
```

### 4.7 Latency Simulation for Testing

Colyseus provides a built-in latency simulator for development:

```typescript
// In app.config.ts
server.simulateLatency(200); // 200ms round-trip delay
```

---

## 5. Complete Code Examples

### 5.1 Server Room with Schema State (Basic)

```typescript
import { Room, Client } from "colyseus";
import { Schema, type, MapSchema } from "@colyseus/schema";

export class Player extends Schema {
    @type("number") x: number;
    @type("number") y: number;
}

export class MyRoomState extends Schema {
    @type({ map: Player }) players = new MapSchema<Player>();
}

export class GameRoom extends Room {
    state = new MyRoomState();

    messages = {
        0: (client: Client, payload: any) => {
            const player = this.state.players.get(client.sessionId);
            const velocity = 2;

            if (payload.left) player.x -= velocity;
            else if (payload.right) player.x += velocity;

            if (payload.up) player.y -= velocity;
            else if (payload.down) player.y += velocity;
        }
    }

    onJoin(client: Client, options: any) {
        const player = new Player();
        player.x = Math.random() * 800;
        player.y = Math.random() * 600;
        this.state.players.set(client.sessionId, player);
    }

    onLeave(client: Client) {
        this.state.players.delete(client.sessionId);
    }
}
```

### 5.2 Server Room with Fixed Tick Rate and Input Queue

```typescript
import { Room, Client } from "colyseus";
import { Schema, type, MapSchema } from "@colyseus/schema";

export interface InputData {
    left: boolean;
    right: boolean;
    up: boolean;
    down: boolean;
    tick?: number;
}

export class Player extends Schema {
    @type("number") x: number;
    @type("number") y: number;
    @type("number") tick: number;

    // Server-only: not synchronized
    inputQueue: InputData[] = [];
}

export class MyRoomState extends Schema {
    @type("number") mapWidth: number;
    @type("number") mapHeight: number;
    @type({ map: Player }) players = new MapSchema<Player>();
}

export class GameRoom extends Room {
    state = new MyRoomState();
    fixedTimeStep = 1000 / 60; // 60 Hz

    messages = {
        0: (client: Client, input: InputData) => {
            const player = this.state.players.get(client.sessionId);
            player.inputQueue.push(input);
        }
    }

    onCreate(options: any) {
        this.state.mapWidth = 800;
        this.state.mapHeight = 600;

        let elapsedTime = 0;
        this.setSimulationInterval((deltaTime) => {
            elapsedTime += deltaTime;
            while (elapsedTime >= this.fixedTimeStep) {
                elapsedTime -= this.fixedTimeStep;
                this.fixedTick(this.fixedTimeStep);
            }
        });
    }

    fixedTick(timeStep: number) {
        const velocity = 2;

        this.state.players.forEach(player => {
            let input: InputData;
            while (input = player.inputQueue.shift()) {
                if (input.left) player.x -= velocity;
                else if (input.right) player.x += velocity;

                if (input.up) player.y -= velocity;
                else if (input.down) player.y += velocity;

                player.tick = input.tick; // Echo tick for reconciliation
            }
        });
    }

    onJoin(client: Client, options: any) {
        const player = new Player();
        player.x = Math.random() * this.state.mapWidth;
        player.y = Math.random() * this.state.mapHeight;
        this.state.players.set(client.sessionId, player);
    }

    onLeave(client: Client) {
        this.state.players.delete(client.sessionId);
    }
}
```

### 5.3 Client-Side Prediction with Reconciliation (Phaser)

```typescript
import Phaser from "phaser";
import { Room, Client, Callbacks } from "@colyseus/sdk";
import type { InputData } from "../server/rooms/GameRoom";

const VELOCITY = 2;

function applyInput(entity: { x: number; y: number }, input: InputData) {
    if (input.left) entity.x -= VELOCITY;
    else if (input.right) entity.x += VELOCITY;
    if (input.up) entity.y -= VELOCITY;
    else if (input.down) entity.y += VELOCITY;
}

export class GameScene extends Phaser.Scene {
    room: Room;
    currentPlayer: Phaser.Types.Physics.Arcade.ImageWithDynamicBody;
    playerEntities: Record<string, Phaser.Types.Physics.Arcade.ImageWithDynamicBody> = {};

    remoteRef: Phaser.GameObjects.Rectangle;
    cursorKeys: Phaser.Types.Input.Keyboard.CursorKeys;

    currentTick = 0;
    pendingInputs: InputData[] = [];
    elapsedTime = 0;
    fixedTimeStep = 1000 / 60;

    async create() {
        this.cursorKeys = this.input.keyboard.createCursorKeys();
        this.room = await new Client("ws://localhost:2567")
            .joinOrCreate("game_room");

        const callbacks = Callbacks.get(this.room);

        callbacks.onAdd("players", (player, sessionId) => {
            const entity = this.physics.add.image(player.x, player.y, "ship");
            this.playerEntities[sessionId] = entity;

            if (sessionId === this.room.sessionId) {
                // --- Local player setup ---
                this.currentPlayer = entity;

                this.remoteRef = this.add.rectangle(0, 0, entity.width, entity.height);
                this.remoteRef.setStrokeStyle(1, 0xff0000);

                callbacks.onChange(player, () => {
                    // Reconciliation trigger
                    this.remoteRef.x = player.x;
                    this.remoteRef.y = player.y;

                    // 1. Discard acknowledged inputs
                    this.pendingInputs = this.pendingInputs.filter(
                        input => input.tick > player.tick
                    );

                    // 2. Snap to server position
                    this.currentPlayer.x = player.x;
                    this.currentPlayer.y = player.y;

                    // 3. Re-apply unacknowledged inputs
                    for (const input of this.pendingInputs) {
                        applyInput(this.currentPlayer, input);
                    }
                });
            } else {
                // --- Remote player setup ---
                callbacks.onChange(player, () => {
                    entity.setData("serverX", player.x);
                    entity.setData("serverY", player.y);
                });
            }
        });

        callbacks.onRemove("players", (player, sessionId) => {
            const entity = this.playerEntities[sessionId];
            if (entity) {
                entity.destroy();
                delete this.playerEntities[sessionId];
            }
        });
    }

    update(time: number, delta: number) {
        if (!this.currentPlayer) return;

        this.elapsedTime += delta;
        while (this.elapsedTime >= this.fixedTimeStep) {
            this.elapsedTime -= this.fixedTimeStep;
            this.fixedTick();
        }

        // Interpolate remote players
        for (const sessionId in this.playerEntities) {
            if (sessionId === this.room.sessionId) continue;
            const entity = this.playerEntities[sessionId];
            const { serverX, serverY } = entity.data.values;
            entity.x = Phaser.Math.Linear(entity.x, serverX, 0.2);
            entity.y = Phaser.Math.Linear(entity.y, serverY, 0.2);
        }
    }

    fixedTick() {
        this.currentTick++;

        const input: InputData = {
            left: this.cursorKeys.left.isDown,
            right: this.cursorKeys.right.isDown,
            up: this.cursorKeys.up.isDown,
            down: this.cursorKeys.down.isDown,
            tick: this.currentTick,
        };

        // Send to server
        this.room.send(0, input);

        // Apply locally (prediction)
        applyInput(this.currentPlayer, input);

        // Store for reconciliation
        this.pendingInputs.push(input);
    }
}
```

---

## 6. Planned Built-In Prediction Features

The [Colyseus v1.0 roadmap](https://docs.colyseus.io/roadmap) lists two directly relevant planned features:

1. **Bidirectional schema instances** -- enabling client-to-server state updates, which would formalize the pattern of clients locally modifying state before server confirmation.

2. **Client-side prediction as a first-class concept** -- the roadmap states plans for "client-side prediction techniques that developers can specify on the frontend." This would presumably integrate prediction, reconciliation, and potentially rollback into the Schema callback system, removing the need for manual implementation.

Additionally, the roadmap mentions:

- Decoupling `@colyseus/schema` from the core to support alternative serialization (e.g., Yjs/CRDT).
- Potential RPC calls via schema instances.
- Message priority levels for backpressure handling.

As of early 2026, these features have not shipped. Client-side prediction remains a manual implementation pattern.

---

## 7. How Colyseus Compares to Other Frameworks for Prediction

| Aspect | Colyseus | Netcode for GameObjects (Unity) | Unreal Engine Replication | Photon Fusion |
|--------|----------|---------------------------------|--------------------------|---------------|
| **Built-in prediction** | No (manual pattern) | Yes (`NetworkTransform` prediction) | Yes (character movement component) | Yes (full rollback/resimulation) |
| **State sync model** | Schema-based delta encoding | NetworkVariable with delta compression | Property replication with relevancy | Snapshots + input authority modes |
| **Prediction approach** | Developer implements shared logic + reconciliation | `ClientNetworkTransform` for owner-predicted movement | `UCharacterMovementComponent` with built-in prediction + replay | Shared mode (state authority) or Host mode (input authority + rollback) |
| **Reconciliation** | Manual (filter pending inputs by tick, snap + replay) | Automatic for NetworkTransforms | Automatic with correction smoothing | Automatic full-world rollback and resimulation |
| **Language parity** | TypeScript on both sides (easy to share code) | C# on both sides | C++ on both sides | C# on both sides |
| **Tick system** | Manual fixed-step loop via `setSimulationInterval` | Built-in `NetworkTickSystem` | Built-in engine tick | Built-in tick-aligned simulation |
| **Input handling** | Messages via `room.send()` | RPCs or NetworkVariables | Client-to-server RPCs | `NetworkInput` struct polled per tick |

**Colyseus's advantages for prediction:**

- **Shared TypeScript** -- since both client and server run TypeScript/JavaScript, sharing movement functions is trivial. No cross-language serialization layer needed.
- **Flexible architecture** -- the lack of a built-in prediction system means developers can implement exactly the strategy their game needs (dead reckoning, rollback, interpolation-only, etc.) without fighting framework assumptions.
- **Lightweight** -- no engine lock-in; works with any client renderer (Phaser, Babylon.js, PlayCanvas, Three.js, Cocos Creator, or custom).

**Colyseus's disadvantages for prediction:**

- **No built-in rollback** -- unlike Photon Fusion or Unreal, there is no automatic world-state snapshot + resimulation system. Developers must implement reconciliation manually.
- **No built-in physics prediction** -- frameworks like Unreal and Unity integrate prediction with their physics engines. In Colyseus, physics prediction (if used) requires a JS physics engine running on both sides.
- **No automatic correction smoothing** -- when reconciliation produces a snap, the developer must implement their own visual smoothing (e.g., lerp between corrected position and rendered position).

---

## 8. References

### Official Documentation

- [Colyseus -- State Synchronization](https://docs.colyseus.io/state)
- [Colyseus -- Schema Definition](https://docs.colyseus.io/state/schema)
- [Colyseus -- State View (Filtering)](https://docs.colyseus.io/state/view)
- [Colyseus -- Advanced Schema Usage](https://docs.colyseus.io/state/advanced-usage)
- [Colyseus -- State Sync Callbacks](https://docs.colyseus.io/sdk/state-sync-callbacks)
- [Colyseus -- Room API](https://docs.colyseus.io/room)
- [Colyseus -- Best Practices](https://docs.colyseus.io/best-practices)
- [Colyseus -- Command Pattern](https://docs.colyseus.io/best-practices/command-pattern)
- [Colyseus -- v1.0 Roadmap](https://docs.colyseus.io/roadmap)

### Tutorials

- [Phaser Tutorial -- Part 1: Basic Player Movement](https://docs.colyseus.io/learn/tutorial/phaser/basic-player-movement)
- [Phaser Tutorial -- Part 2: Linear Interpolation](https://docs.colyseus.io/learn/tutorial/phaser/linear-interpolation)
- [Phaser Tutorial -- Part 3: Client Predicted Input](https://docs.colyseus.io/learn/tutorial/phaser/client-predicted-input)
- [Phaser Tutorial -- Part 4: Fixed Tickrate](https://docs.colyseus.io/learn/tutorial/phaser/fixed-tickrate)
- [Tutorial Source Code (GitHub)](https://github.com/colyseus/tutorial-phaser)
- [Live Demo (Glitch)](https://glitch.com/~colyseus-phaser-tutorial)

### Source Code

- [Colyseus Server (GitHub)](https://github.com/colyseus/colyseus)
- [@colyseus/schema (GitHub)](https://github.com/colyseus/schema)
- [@colyseus/command (GitHub)](https://github.com/colyseus/command)

### Background Reading on Client-Side Prediction

- [Gabriel Gambetta -- Client-Side Prediction and Server Reconciliation](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
- [Valve -- Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- [Glenn Fiedler -- State Synchronization](https://gafferongames.com/post/state_synchronization/)
