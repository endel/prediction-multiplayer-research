# Mirror Networking: Client-Side Prediction in Unity

## 1. Overview of Mirror

Mirror is a **high-level networking library for Unity**, optimized for ease of use and probability of success. It is an open-source fork of Unity's deprecated UNET (UNet HLAPI), maintained as a community-driven project on GitHub under the MIT license.

Key characteristics:

- **Origin**: Forked from Unity's built-in networking (UNET) after Unity deprecated it in favor of Netcode for GameObjects. Mirror preserved and extended the UNET API while fixing longstanding bugs and performance issues.
- **Open-source**: Fully open-source on GitHub at [MirrorNetworking/Mirror](https://github.com/MirrorNetworking/Mirror), with regular monthly updates.
- **Transport-agnostic**: Supports over a dozen low-level transports (KCP, Telepathy, WebSockets, Steam, etc.).
- **Server-authoritative**: All game state is owned by the server. Clients send inputs via Commands and receive state via SyncVars and RPCs.
- **Features**: Interest Management (5 built-in varieties), Additive Scene support with Physics Isolation, Script Templates, and extensive examples.
- **Unity LTS alignment**: Follows Unity's Long-Term Support release schedule.

---

## 2. Mirror's Networking Model

Mirror uses a server-authoritative architecture with three core synchronization primitives:

### SyncVars

Variables marked with `[SyncVar]` are automatically synchronized from server to all observing clients.

```csharp
public class Enemy : NetworkBehaviour
{
    [SyncVar]
    public int health = 100;

    [SyncVar(hook = nameof(OnNameChanged))]
    public string displayName;

    void OnNameChanged(string oldName, string newName)
    {
        // Called on clients when value changes
        nameLabel.text = newName;
    }
}
```

- Data flows **server to clients only** (one-way).
- Up to 64 SyncVars per `NetworkBehaviour` script.
- Uses dirty-bit tracking: only changed values are transmitted.
- State is applied before `OnStartClient()` is called.
- Also supports `SyncList`, `SyncDictionary`, `SyncHashSet`, and `SyncSortedSet` for collections.
- `NetworkSyncMode` can be set to "Owner" to restrict sync to only the owning client (useful for inventories, quest state, etc.).

### Commands (Client to Server)

```csharp
[Command]
void CmdFire(Vector3 direction)
{
    // Runs on server
    GameObject bullet = Instantiate(bulletPrefab, firePoint.position, Quaternion.identity);
    bullet.GetComponent<Rigidbody>().velocity = direction * speed;
    NetworkServer.Spawn(bullet);
}
```

- Marked with `[Command]` attribute, prefixed with `Cmd` by convention.
- By default, only the owning player can send Commands (security).
- `[Command(requiresAuthority = false)]` allows non-owner invocation.
- Optional `NetworkConnectionToClient sender = null` parameter identifies the sender.

### ClientRpc (Server to All Clients)

```csharp
[ClientRpc]
public void RpcTakeDamage(int amount)
{
    // Runs on all observing clients
    healthBar.SetValue(health);
    PlayDamageEffect();
}
```

- Marked with `[ClientRpc]`, prefixed with `Rpc` by convention.
- Sent to all clients observing the object (respects Interest Management).
- `[ClientRpc(includeOwner = false)]` excludes the owner.

### TargetRpc (Server to Specific Client)

```csharp
[TargetRpc]
public void TargetShowReward(NetworkConnectionToClient target, int gold)
{
    // Runs only on the targeted client
    rewardUI.Show(gold);
}
```

---

## 3. PredictedRigidbody System

Mirror's `PredictedRigidbody` component is the primary mechanism for client-side prediction of physics objects. It is designed for physics-heavy games (billiards, destruction, VR) where 50ms+ of latency makes the game feel unresponsive.

### 3.1 Three Simultaneous Objects

When `PredictedRigidbody` is active on a client in Smooth Mode, three distinct GameObjects exist simultaneously:

| Object | Visibility | Purpose |
|--------|-----------|---------|
| **Predicted Physics Object** | Transparent (invisible to player) | A "ghost" GameObject that holds the Rigidbody and Colliders. Runs physics simulation ahead of the server, receives corrections. |
| **Rendered Original** | Visible | The original GameObject with renderers. Smoothly interpolates toward the physics ghost. This is what the player sees. |
| **Remote State Object** | Transparent (debug only) | Optional visualization showing the latest authoritative server state. Enabled via `showGhost` for debugging. |

This separation allows hard physics corrections on the ghost without causing visible snapping on the rendered object.

### 3.2 Position History Tracking

The system maintains a `SortedList<double, RigidbodyState>` keyed by timestamp, recording snapshots of the Rigidbody's state at regular intervals.

**Key parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `recordInterval` | 0.050s (50ms) | Time between state recordings |
| `stateHistoryLimit` | 32 | Max entries (~1.6 seconds of history at 50ms) |
| `onlyRecordChanges` | true | Skip recording if state hasn't changed |
| `compareLastFirst` | true | Fast-path: compare against latest state before searching history |

Each `RigidbodyState` entry captures:
- Timestamp
- Position (Vector3)
- Rotation (Quaternion)
- Velocity (Vector3)
- Angular Velocity (Vector3)
- Position delta and rotation delta (relative to previous entry)

### 3.3 Correction via Delta Reapplication

When the server sends authoritative state, the client must reconcile its predicted history. Mirror does **not** re-simulate physics. Instead, it uses a mathematical delta reapplication algorithm:

**Step 1: Sample History**

Find the two recorded states that bracket the server's timestamp. Interpolate between them to estimate what the client thought the state was at that exact server time.

**Step 2: Compare**

If the difference between the interpolated local state and the server state exceeds thresholds (`positionCorrectionThreshold` = 0.10m, `rotationCorrectionThreshold` = 5 degrees), a correction is needed.

**Step 3: Correct and Rewind Deltas**

The `Prediction.CorrectHistory()` method:

1. Inserts the corrected server state into the history at the appropriate timestamp.
2. Recalculates the delta of the entry immediately following the correction point, scaling it proportionally to the new time gap. For example, if the original delta spanned 2.0 seconds but the correction splits it so only 0.5 seconds remain, the delta is scaled by `0.5 / 2.0`.
3. Rewinds forward through all subsequent history entries, recomputing absolute positions as: `entry.position = previous.position + entry.positionDelta`.
4. Returns the final recomputed state, which is applied to the Rigidbody.

This is the core insight: rather than re-simulating physics from the correction point forward, Mirror replays the *deltas* (position changes, velocity changes) on top of the corrected base state. The result is a fast approximation that preserves the shape of the player's interactions.

**Simplified reconciliation flow:**

```
Server state arrives (timestamp T=2.5)
  |
  v
Find history entries at T=2.0 and T=3.0
  |
  v
Interpolate local state at T=2.5
  |
  v
Compare: |server.pos - interpolated.pos| > 0.10m?
  |
  YES --> CorrectHistory():
  |        1. Insert server state at T=2.5
  |        2. Scale delta for T=2.5->3.0 (was 2.0->3.0)
  |        3. Replay deltas: T=3.0, T=3.05, T=3.1, ...
  |        4. Apply final recomputed state to Rigidbody
  |
  NO --> Do nothing (prediction was close enough)
```

The reconciliation code (simplified from source):

```csharp
void OnReceivedState(double timestamp, RigidbodyState state)
{
    // Fast path: if current position is already close, skip
    if (compareLastFirst &&
        (state.position - physicsPosition).sqrMagnitude < positionCorrectionThresholdSqr &&
        Quaternion.Angle(state.rotation, physicsRotation) < rotationCorrectionThreshold)
    {
        return;
    }

    RecordState(); // Capture current state before correction

    // Find bracketing history entries
    if (!Prediction.Sample(stateHistory, timestamp,
            out RigidbodyState before, out RigidbodyState after,
            out int afterIndex, out double t))
    {
        // No valid history bracket found - hard correct
        ApplyState(state.timestamp, state.position, state.rotation,
                   state.velocity, state.angularVelocity);
        return;
    }

    // Interpolate what we thought the state was at server time
    RigidbodyState interpolated = RigidbodyState.Interpolate(before, after, (float)t);

    // Check if correction is needed
    float posDiffSqr = (state.position - interpolated.position).sqrMagnitude;
    float rotDiff = Quaternion.Angle(state.rotation, interpolated.rotation);

    if (posDiffSqr >= positionCorrectionThresholdSqr ||
        rotDiff >= rotationCorrectionThreshold)
    {
        // Rewind deltas on top of corrected state
        RigidbodyState recomputed = Prediction.CorrectHistory(
            stateHistory, stateHistoryLimit, state, before, after, afterIndex);

        ApplyState(recomputed.timestamp, recomputed.position, recomputed.rotation,
                   recomputed.velocity, recomputed.angularVelocity);
        OnCorrected(); // Virtual callback
    }
}
```

### 3.4 Smooth Mode vs Fast Mode

Mirror offers two rendering modes, with the stated goal of eventually settling on one:

#### Smooth Mode

- Physics components (Rigidbody, Colliders, Joints) are **moved to a separate ghost GameObject** at runtime.
- The original GameObject keeps only its renderers and smoothly interpolates toward the physics ghost.
- Hard corrections happen on the invisible ghost; the visible object slides smoothly to match.
- Higher visual quality but more computational overhead from ghost creation/destruction.
- **Caveat**: `GetComponent<Rigidbody>()` on the original object will fail. Must use `GetComponent<PredictedRigidbody>().predictedRigidbody` instead.
- **Caveat**: `OnCollisionEnter/Exit` and `OnTriggerEnter/Exit` will not fire reliably on the original object.

#### Fast Mode

- All components stay on the original GameObject.
- Uses `Rigidbody.MovePosition()` and `MoveRotation()` for softer corrections.
- No ghost creation overhead.
- Snappier visual appearance with slightly harsher corrections visible.
- Simpler to work with (standard component access works).

**Smoothing parameters for both modes:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `positionInterpolationSpeed` | 15 | How fast the rendered object catches up (higher = sharper) |
| `rotationInterpolationSpeed` | 10 | Rotation catch-up speed |
| `snapThreshold` | 2 m/s | Velocity below which to snap rather than interpolate |
| `teleportDistanceMultiplier` | 10 | Threshold (as multiplier of collider size) for teleporting instead of interpolating |
| `ghostVelocityThreshold` | 0.1 m/s | Minimum velocity to create/maintain ghosts |
| `motionSmoothingVelocityThreshold` | 0.1 m/s | Minimum velocity to activate smoothing |
| `motionSmoothingAngularVelocityThreshold` | 5.0 deg/s | Angular velocity threshold |
| `motionSmoothingTimeTolerance` | 0.5s | Minimum duration before deactivating ghosts |

---

## 4. Why Mirror Avoids Physics.Simulate()

This is one of Mirror's most distinctive and intentional design decisions. Most prediction systems (including Unreal Engine's approach) re-simulate physics from the correction point forward using the engine's physics step. Mirror explicitly rejects this.

### The Problem with Physics.Simulate()

1. **Performance at scale**: `Physics.Simulate()` re-simulates the **entire physics scene**. In a game with 1000 Rigidbodies where the player only interacts with 3, resimulating all 1000 is wasteful.

2. **Non-determinism**: Unity's PhysX is not deterministic across machines. Floating-point differences between client and server accumulate, causing desync "within seconds."

3. **Scene separation complexity**: Unity (pre-6) had limited support for per-scene physics. Isolating predicted objects into their own physics scene for selective re-simulation added significant complexity.

### Mirror's Alternative: Manual Vector Math

Instead of re-simulating physics, Mirror replays the **deltas** (changes in position, rotation, velocity) using simple vector addition and quaternion multiplication:

```
corrected_position = server_position + delta_1 + delta_2 + delta_3 + ...
```

This is:
- **Fast**: O(n) where n is the number of history entries after the correction point (typically < 32). No physics engine involvement.
- **Scalable**: Only the corrected object is affected. Other Rigidbodies are untouched.
- **Approximate**: Does not account for collisions, gravity changes, or forces applied between the correction point and now. This is an intentional tradeoff.

As the Mirror developers state:

> "Mirror's prediction works really well for large physics scenes where the player only interacts with a few objects at a time. Our algorithm sacrifices accuracy for performance."

> "History rewinding is done manually via Vector math, instead of real physics. It's not 100% correct -- but it sure is fast!"

> "Not using Physics.Simulate() is a risky approach."

---

## 5. C# Code Examples

### 5.1 Setting Up PredictedRigidbody

The basic setup requires adding the component and implementing dual-sided physics:

```csharp
using Mirror;
using UnityEngine;

public class PredictedBall : NetworkBehaviour
{
    public float forceMultiplier = 10f;

    void Update()
    {
        if (!isOwned) return;
        if (!Input.GetMouseButtonDown(0)) return;

        // Calculate force from click
        Vector3 force = CalculateForceFromClick() * forceMultiplier;

        // Apply physics locally (prediction) AND send to server
        Rigidbody rb = GetComponent<PredictedRigidbody>().predictedRigidbody;
        rb.AddForce(force);

        // Tell server to apply the same force
        CmdApplyForce(force);
    }

    [Command]
    void CmdApplyForce(Vector3 force)
    {
        // Server applies the same force authoritatively
        GetComponent<Rigidbody>().AddForce(force);
    }

    Vector3 CalculateForceFromClick()
    {
        Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
        return ray.direction;
    }
}
```

**Important**: On the client, use `GetComponent<PredictedRigidbody>().predictedRigidbody` to access the Rigidbody (because in Smooth Mode it has been moved to the ghost). On the server, use `GetComponent<Rigidbody>()` directly (no ghosts exist on the server).

### 5.2 Ghost Object Visualization

Enable ghost rendering for debugging by configuring the `PredictedRigidbody` component in the Inspector:

```csharp
using Mirror;
using UnityEngine;

// PredictedRigidbody Inspector settings for ghost visualization:
// - Show Ghost: true (enables the remote state ghost)
// - Local Ghost Material: assign a transparent material (e.g., green wireframe)
// - Remote Ghost Material: assign a transparent material (e.g., red wireframe)

// The component creates ghosts automatically. To customize ghost behavior:
public class DebugPrediction : NetworkBehaviour
{
    PredictedRigidbody predicted;

    void Start()
    {
        predicted = GetComponent<PredictedRigidbody>();
    }

    void OnGUI()
    {
        if (!isOwned) return;

        // Display prediction debug info
        GUILayout.Label($"History entries: {predicted.stateHistory.Count}");
        GUILayout.Label($"Physics ghost exists: {predicted.physicsCopy != null}");

        if (predicted.physicsCopy != null)
        {
            Vector3 physicsPos = predicted.predictedRigidbody.position;
            Vector3 renderPos = transform.position;
            float gap = Vector3.Distance(physicsPos, renderPos);
            GUILayout.Label($"Render-to-physics gap: {gap:F3}m");
        }
    }
}
```

The three objects visible during debugging:
- **Original object** (rendered, visible): what the player sees, smoothly following
- **Physics ghost** (transparent local ghost material): where physics "really" is
- **Remote ghost** (transparent remote ghost material): where the server says it is

### 5.3 Custom Prediction for Player Movement

Player character prediction is more complex and still experimental in Mirror. Here is the general pattern:

```csharp
using Mirror;
using UnityEngine;

public class PredictedPlayerController : NetworkBehaviour
{
    public float moveSpeed = 5f;
    public float jumpForce = 8f;

    // In Smooth Mode, always go through PredictedRigidbody
    PredictedRigidbody predicted;
    Rigidbody rb;

    void Start()
    {
        predicted = GetComponent<PredictedRigidbody>();
    }

    void FixedUpdate()
    {
        if (!isOwned) return;

        float h = Input.GetAxisRaw("Horizontal");
        float v = Input.GetAxisRaw("Vertical");
        bool jump = Input.GetKey(KeyCode.Space);

        // Get the correct Rigidbody reference
        rb = isServer
            ? GetComponent<Rigidbody>()
            : predicted.predictedRigidbody;

        if (rb == null) return;

        // Apply movement locally (client prediction)
        Vector3 move = new Vector3(h, 0, v).normalized * moveSpeed;
        rb.linearVelocity = new Vector3(move.x, rb.linearVelocity.y, move.z);

        if (jump && IsGrounded())
        {
            rb.AddForce(Vector3.up * jumpForce, ForceMode.Impulse);
        }

        // Send input to server for authoritative simulation
        if (!isServer)
        {
            CmdMove(h, v, jump);
        }
    }

    [Command]
    void CmdMove(float h, float v, bool jump)
    {
        Rigidbody serverRb = GetComponent<Rigidbody>();
        Vector3 move = new Vector3(h, 0, v).normalized * moveSpeed;
        serverRb.linearVelocity = new Vector3(move.x, serverRb.linearVelocity.y, move.z);

        if (jump && IsGrounded())
        {
            serverRb.AddForce(Vector3.up * jumpForce, ForceMode.Impulse);
        }
    }

    bool IsGrounded()
    {
        return Physics.Raycast(transform.position, Vector3.down, 1.1f);
    }
}
```

**Note**: The Mirror team acknowledges that predicted player movement "may or may not work" and anticipates needing a misprediction tolerance (e.g., accept 10% of mispredictions without correcting) to avoid constant micro-corrections during continuous movement.

### 5.4 Handling Corrections and Smoothing

```csharp
using Mirror;
using UnityEngine;

public class CorrectionHandler : NetworkBehaviour
{
    PredictedRigidbody predicted;

    void Start()
    {
        predicted = GetComponent<PredictedRigidbody>();
    }

    // Override virtual callbacks for correction events
    // These are called on the PredictedRigidbody component itself.
    // To use them, subclass PredictedRigidbody or listen externally:

    void OnEnable()
    {
        // Monitor correction frequency for debugging
        StartCoroutine(MonitorCorrections());
    }

    System.Collections.IEnumerator MonitorCorrections()
    {
        int lastCount = 0;
        while (true)
        {
            yield return new WaitForSeconds(1f);
            // Log if corrections are happening too frequently
            // (indicates prediction quality issues)
        }
    }
}

// For collision detection with predicted objects,
// put collision logic on the OTHER object:
public class PredictedCollisionReceiver : MonoBehaviour
{
    void OnCollisionEnter(Collision collision)
    {
        // Check if we collided with a predicted object's ghost
        if (PredictedRigidbody.IsPredicted(collision.collider,
                out PredictedRigidbody predicted))
        {
            Debug.Log($"Hit predicted object: {predicted.name}");
            // Handle collision with the original predicted object
        }
    }
}
```

**Virtual callbacks available on PredictedRigidbody:**

| Callback | When Called |
|----------|------------|
| `OnBeginPrediction()` | Ghost created, prediction begins |
| `OnEndPrediction()` | Ghost destroyed, prediction ends |
| `OnCorrected()` | Server correction was applied |
| `OnSnappedIntoPlace()` | Object snapped due to low velocity |

---

## 6. Known Issues

The Mirror team is transparent about current limitations:

### Stacked Physics Objects

Objects stacked on top of each other (e.g., a pile of crates) "generally sync well, but don't properly come to rest just yet." The delta reapplication algorithm does not account for contact forces between objects, so stacked configurations can jitter or fail to settle.

### Compound Objects

Complex multi-body configurations with Joints and child Rigidbodies are partially supported but remain undertested. The `PredictionUtils` class handles moving Joints (CharacterJoint, ConfigurableJoint, FixedJoint, HingeJoint, SpringJoint) to ghost objects, but edge cases exist.

### Collision Callbacks

In Smooth Mode, `OnCollisionEnter`, `OnCollisionExit`, `OnTriggerEnter`, and `OnTriggerExit` do not fire reliably on the original object because the Colliders have been moved to the ghost. Workaround: place collision logic on the *other* object and use `PredictedRigidbody.IsPredicted()` to check.

### GetComponent Limitations

In Smooth Mode, `GetComponent<Rigidbody>()` and `GetComponent<Collider>()` on the original object will return null. All physics access must go through `GetComponent<PredictedRigidbody>().predictedRigidbody`.

### Accuracy vs. Correctness

Because corrections use vector math rather than physics re-simulation, the corrected state does not account for:
- Collisions that would have occurred between the correction point and now
- Gravity or other forces applied during that window
- Friction, drag, or other continuous forces

This is acceptable for objects the player "flicks" (billiard balls, thrown objects) but may be problematic for objects under continuous force (vehicles, characters).

---

## 7. Experimental Status and Future Direction

Mirror's prediction system carries explicit warnings:

> "Mirror is currently experimenting with various Prediction algorithms. This is all purely experimental, we don't recommend using this just yet."

### Development Timeline

The system has undergone significant development effort:
- **4 months** refining the initial Billiards demo, fixing miscalculations
- **3 months** porting to complex scenes, adding Collider/Joint support, handling child object Rigidbodies
- The Smooth Mode vs Fast Mode split exists because neither has proven universally better; the team aims to "settle with one mode eventually"

### Planned Work

- **Stacked object physics**: Getting stacked objects to properly come to rest
- **Predicted player movement**: Currently untested; anticipated to require a misprediction tolerance (~10%) to avoid constant micro-corrections
- **Performance benchmarking**: The `Examples/BenchmarkPrediction` scenario creates hundreds or thousands of predicted objects to stress-test the system
- **Bandwidth optimization**: `reduceSendsWhileIdle` already syncs only once per second for idle objects vs. every frame for moving ones

### The `oneFrameAhead` Mystery

The `oneFrameAhead` parameter (default: true) applies corrections one frame early and produces "much better results," though the developers note uncertainty about why this works. This is characteristic of the experimental, empirical nature of the current system.

---

## 8. Comparison to FishNet's Approach

Mirror and [FishNet](https://github.com/FirstGearGames/FishNet) (another Unity networking library, also derived from concepts in UNET) take different approaches to client-side prediction:

| Aspect | Mirror | FishNet |
|--------|--------|---------|
| **Physics re-simulation** | Avoids `Physics.Simulate()` entirely; uses manual delta reapplication via vector math | Uses `Physics.Simulate()` with isolated physics scenes for re-simulation |
| **Accuracy** | Approximate (no collision detection during rewind) | More physically accurate (full physics re-simulation) |
| **Performance** | O(n) vector math per corrected object; scales to large scenes | Re-simulation cost scales with scene complexity; mitigated by physics scene isolation |
| **Complexity** | Simpler algorithm, fewer Unity API dependencies | More complex setup with physics scene management |
| **Ghost objects** | Creates separate ghost GameObjects for physics/rendering split | Uses different approach to visual smoothing |
| **Prediction scope** | Currently focused on physics objects (Rigidbodies); player movement experimental | Supports both physics prediction and CharacterController/transform-based prediction |
| **Maturity** | Experimental, not recommended for production | More established prediction system with production usage |
| **CSP (Client-Side Prediction)** | Built into `PredictedRigidbody` component | Dedicated prediction and reconciliation framework with tick-based input replay |
| **Input handling** | Commands sent independently; no tick-based input buffer | Tick-aligned input replays for deterministic reconciliation |

**Key philosophical difference**: Mirror prioritizes **performance over accuracy**, making it suitable for large physics scenes where perfect re-simulation is too expensive. FishNet prioritizes **correctness**, using real physics re-simulation to ensure collisions and forces are properly accounted for during correction.

---

## 9. References

### Official Documentation
- [Mirror Networking Documentation](https://mirror-networking.gitbook.io/docs) -- Main documentation hub
- [Client-Side Prediction Guide](https://mirror-networking.gitbook.io/docs/manual/general/client-side-prediction) -- Prediction system overview and philosophy
- [Synchronization (SyncVars)](https://mirror-networking.gitbook.io/docs/manual/guides/synchronization) -- State synchronization guide
- [Remote Actions (Commands, ClientRpc, TargetRpc)](https://mirror-networking.gitbook.io/docs/manual/guides/communications/remote-actions) -- RPC documentation

### Source Code
- [Mirror GitHub Repository](https://github.com/MirrorNetworking/Mirror) -- Full source code
- [PredictedRigidbody.cs](https://github.com/MirrorNetworking/Mirror/blob/master/Assets/Mirror/Components/PredictedRigidbody/PredictedRigidbody.cs) -- Main prediction component
- [Prediction.cs](https://github.com/MirrorNetworking/Mirror/blob/master/Assets/Mirror/Core/Prediction/Prediction.cs) -- Core prediction algorithms (Sample, CorrectHistory)
- [PredictedSyncData.cs](https://github.com/MirrorNetworking/Mirror/blob/master/Assets/Mirror/Components/PredictedRigidbody/PredictedSyncData.cs) -- Blittable sync data struct
- [PredictionUtils.cs](https://github.com/MirrorNetworking/Mirror/blob/master/Assets/Mirror/Components/PredictedRigidbody/PredictionUtils.cs) -- Physics component migration utilities

### Examples
- [BilliardsPredicted Example](https://github.com/MirrorNetworking/Mirror/tree/master/Assets/Mirror/Examples/BilliardsPredicted) -- Primary prediction demo
- [BenchmarkPrediction Example](https://github.com/MirrorNetworking/Mirror/tree/master/Assets/Mirror/Examples/BenchmarkPrediction) -- Stress test with hundreds/thousands of predicted objects

### Related Projects
- [FishNet Networking](https://github.com/FirstGearGames/FishNet) -- Alternative Unity networking library with different prediction approach
- [Mirror Discord](https://discord.gg/mirror) -- Community support and discussion
