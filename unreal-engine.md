# Unreal Engine Prediction Systems

## Table of Contents

1. [Overview of Unreal's Networking Model](#1-overview-of-unreals-networking-model)
2. [CharacterMovementComponent Prediction](#2-charactermovementcomponent-prediction)
3. [Gameplay Ability System (GAS) Prediction](#3-gameplay-ability-system-gas-prediction)
4. [Network Prediction Plugin (Experimental)](#4-network-prediction-plugin-experimental)
5. [How These Systems Overlap](#5-how-these-systems-overlap)
6. [Trade-offs and Known Issues](#6-trade-offs-and-known-issues)
7. [References](#7-references)

---

## 1. Overview of Unreal's Networking Model

Unreal Engine uses a **server-authoritative** networking model. The server is the single source of truth for all game state: it runs the canonical simulation, validates client actions, and replicates the results to connected clients.

### Actor Replication

Actors opt into replication by setting `bReplicates = true`. The server decides which actors are **net-relevant** to each client (based on distance, visibility, priority) and serializes their state into network packets. Clients receive these snapshots and update their local copies of the actors.

```cpp
AMyActor::AMyActor()
{
    bReplicates = true;
    bNetLoadOnClient = true;
    NetUpdateFrequency = 30.f; // How often per second to replicate
}
```

### Property Replication

Individual properties are registered for replication in `GetLifetimeReplicatedProps`. The engine tracks dirty state and only sends changed properties. Conditions can restrict which clients receive a property (e.g., owner-only, skip-owner, initial-only).

```cpp
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyActor, Health);
    DOREPLIFETIME_CONDITION(AMyActor, SecretData, COND_OwnerOnly);
}
```

When a replicated property changes on the server, clients receive the new value and optionally trigger a `RepNotify` callback:

```cpp
UPROPERTY(ReplicatedUsing = OnRep_Health)
float Health;

void AMyActor::OnRep_Health()
{
    // React to server-authoritative health change
    UpdateHealthBar();
}
```

### Remote Procedure Calls (RPCs)

RPCs are function calls that cross the network boundary. There are three types:

| RPC Type | Direction | Runs On | Use Case |
|----------|-----------|---------|----------|
| `Server` | Client -> Server | Server | Client requesting actions (fire, jump) |
| `Client` | Server -> Client | Owning client | Server notifying specific client |
| `NetMulticast` | Server -> All | Server + all clients | Cosmetic events (explosions, sounds) |

```cpp
// Client sends input to server
UFUNCTION(Server, Reliable)
void ServerFire(FVector AimDirection);

// Server tells owning client about rejection
UFUNCTION(Client, Reliable)
void ClientFireRejected();

// Server broadcasts cosmetic event to all
UFUNCTION(NetMulticast, Unreliable)
void MulticastPlayFireEffect();
```

### The Fundamental Latency Problem

Every Server RPC has a round-trip cost: the client sends a request, the server processes it, and the result replicates back. At 100ms ping, the client sees a 100ms delay before their action takes effect. This is where prediction systems become essential -- they let the client act immediately and reconcile with the server later.

---

## 2. CharacterMovementComponent Prediction

The `UCharacterMovementComponent` (CMC) is Unreal's built-in solution for predicted player movement. It implements a classic **client-side prediction with server reconciliation** model, similar to what id Software pioneered in Quake.

### Architecture Overview

The CMC prediction loop operates on three roles:

- **Autonomous Proxy (owning client):** Runs movement locally, sends inputs to server, replays on correction
- **Authority (server):** Receives client inputs, executes the same movement logic, sends corrections
- **Simulated Proxy (other clients):** Receives replicated positions and interpolates/extrapolates

### The Client Prediction Loop

#### Step 1: Client Performs Move Locally

Each tick, the owning client samples input (acceleration, rotation, jump, etc.), performs the move locally via `PerformMovement()`, and immediately sees the result. The client also saves the move for later replay.

```cpp
// Simplified flow inside UCharacterMovementComponent::TickComponent
void UCharacterMovementComponent::TickComponent(float DeltaTime, ...)
{
    if (CharacterOwner->GetLocalRole() == ROLE_AutonomousProxy)
    {
        // 1. Sample input
        FCharacterMovementComponentInput Input = GetPendingInputVector();

        // 2. Perform move locally (instant response)
        PerformMovement(DeltaTime);

        // 3. Save this move
        FSavedMovePtr NewMove = CreateSavedMove();
        NewMove->SetMoveFor(CharacterOwner, DeltaTime, Input, GetCompressedFlags());

        // 4. Send to server
        ReplicateMoveToServer(DeltaTime, Input);
    }
}
```

#### Step 2: FSavedMove -- Saving Moves for Replay

`FSavedMove_Character` captures everything needed to reproduce a move:

```cpp
class FSavedMove_Character
{
public:
    // Input state
    FVector Acceleration;
    FRotator ControlRotation;
    uint8 CompressedFlags; // Jump, crouch, etc.

    // Result state (after move executed)
    FVector SavedLocation;
    FRotator SavedRotation;
    FVector SavedVelocity;
    EMovementMode SavedMovementMode;

    // Timing
    float DeltaTime;
    float TimeStamp;

    // Can this move be combined with another? (bandwidth optimization)
    virtual bool CanCombineWith(const FSavedMovePtr& NewMove, ACharacter* Character, float MaxDelta) const;

    // Are two moves identical? Used for deduplication
    virtual bool IsImportantMove(const FSavedMovePtr& LastAckedMove) const;

    // Restore state before replaying
    virtual void PrepMoveFor(ACharacter* Character);

    // Store results after move
    virtual void PostUpdate(ACharacter* Character, EPostUpdateMode PostUpdateMode);
};
```

The client maintains a list of saved moves (the **pending move list**), containing every move that has not yet been acknowledged by the server. This list is managed by `FNetworkPredictionData_Client_Character`:

```cpp
class FNetworkPredictionData_Client_Character
{
public:
    // Moves awaiting server acknowledgment
    TArray<FSavedMovePtr> SavedMoves;

    // The most recently acknowledged move timestamp
    float LastAckedMoveTimestamp;

    // Current time on client (advances with each move)
    float CurrentTimeStamp;

    // Network smoothing data
    FVector MeshTranslationOffset;
    FQuat MeshRotationOffset;
    float SmoothNetUpdateTime;
};
```

#### Step 3: Server Validation

The server receives the client's move via `ServerMove()` (an RPC). It applies the same input and delta time to its own copy of the character, using `MoveAutonomous()`:

```cpp
void UCharacterMovementComponent::ServerMove_Implementation(
    float TimeStamp,
    FVector_NetQuantize InAccel,
    FVector_NetQuantize ClientLoc,
    uint8 CompressedMoveFlags,
    uint8 ClientRoll,
    uint32 View,
    UPrimitiveComponent* ClientMovementBase,
    FName ClientBaseBoneName,
    uint8 ClientMovementMode)
{
    // 1. Decompress and apply the client's input
    // 2. Execute MoveAutonomous() with the client's parameters
    // 3. Compare server result with client-reported position

    if (ServerCheckClientError(TimeStamp, DeltaTime, InAccel, ClientLoc, ...))
    {
        // Error exceeds threshold -- send correction
        SendClientAdjustment();
    }
    else
    {
        // Client was correct -- acknowledge the move
        SendClientGoodMove();
    }
}
```

#### Step 4: Client Correction and Replay

When the server detects a mismatch (client position differs from server result beyond a threshold), it sends a **correction** via `ClientAdjustPosition`:

```cpp
void UCharacterMovementComponent::ClientAdjustPosition_Implementation(
    float TimeStamp,
    FVector NewLoc,
    FVector NewVel,
    UPrimitiveComponent* NewBase,
    FName NewBaseBoneName,
    bool bHasBase,
    bool bBaseRelativePosition,
    uint8 ServerMovementMode)
{
    // 1. Accept server's authoritative state
    UpdatedComponent->SetWorldLocation(NewLoc);
    Velocity = NewVel;
    SetMovementMode(ServerMovementMode);

    // 2. Find the corrected move in saved list
    // 3. Discard all moves up to and including the corrected timestamp
    // 4. REPLAY all subsequent saved moves from the corrected state
    ClientHandleMoveResponse(TimeStamp, ...);
}
```

The replay step is critical: the client has continued predicting during the round-trip time, so there may be 3-10+ pending moves after the corrected timestamp. Each is re-executed from the server's corrected state to produce a new predicted position.

### Error Threshold Configuration

```cpp
// In DefaultEngine.ini or project settings
[/Script/Engine.GameNetworkManager]
MAXPOSITIONERRORSQUARED=9.0        // Square of max position error (units)
MAXCLIENTUPDATEINTERVAL=0.25       // Max time between client updates
ClientAuthorativePosition=false    // Trust client position entirely (dangerous)
```

### Network Smoothing Modes

After a correction, snapping the visual mesh to the corrected position would cause jarring teleports. The CMC provides smoothing modes to hide this:

```cpp
// Set in CharacterMovementComponent
UPROPERTY(EditAnywhere, Category = "Character Movement (Networking)")
ENetworkSmoothingMode NetworkSmoothingMode;
```

| Mode | Behavior | Best For |
|------|----------|----------|
| `Disabled` | No smoothing; snap to authoritative position | Debugging |
| `Linear` | Linearly interpolates between old and corrected position over time | Simple cases |
| `Exponential` | Smooths using exponential decay (default) | Most games; hides small corrections well |

**How smoothing works internally:**

The component maintains a `MeshTranslationOffset` and `MeshRotationOffset` that represent the visual difference between where the capsule actually is (corrected position) and where the mesh appears. Each frame, these offsets decay toward zero:

```cpp
void UCharacterMovementComponent::SmoothClientPosition(float DeltaSeconds)
{
    if (NetworkSmoothingMode == ENetworkSmoothingMode::Exponential)
    {
        // Exponential decay of the offset
        SmoothNetUpdateTime -= DeltaSeconds;
        float LerpAlpha = FMath::Clamp(
            1.0f - SmoothNetUpdateTime / SmoothNetUpdateDuration, 0.0f, 1.0f);

        // Decay offset toward zero
        MeshTranslationOffset = FMath::LerpStable(
            MeshTranslationOffset, FVector::ZeroVector, LerpAlpha);
        MeshRotationOffset = FQuat::Slerp(
            MeshRotationOffset, FQuat::Identity, LerpAlpha);
    }
}
```

For **simulated proxies** (other players' characters), the CMC interpolates between received snapshots rather than predicting. This produces smooth movement at the cost of slightly delayed positions.

### Extending the CMC for Custom Movement

To add custom movement modes (e.g., wall-running, grappling), override `FSavedMove_Character`:

```cpp
class FSavedMove_MyCharacter : public FSavedMove_Character
{
public:
    // Custom input flags
    uint8 bWantsToWallRun : 1;
    float GrappleInputStrength;

    virtual void Clear() override
    {
        Super::Clear();
        bWantsToWallRun = false;
        GrappleInputStrength = 0.f;
    }

    virtual uint8 GetCompressedFlags() const override
    {
        uint8 Flags = Super::GetCompressedFlags();
        if (bWantsToWallRun)
        {
            Flags |= FLAG_Custom_0;
        }
        return Flags;
    }

    virtual bool CanCombineWith(const FSavedMovePtr& NewMove,
        ACharacter* Character, float MaxDelta) const override
    {
        FSavedMove_MyCharacter* MyNewMove =
            static_cast<FSavedMove_MyCharacter*>(NewMove.Get());

        if (bWantsToWallRun != MyNewMove->bWantsToWallRun)
            return false;

        return Super::CanCombineWith(NewMove, Character, MaxDelta);
    }

    virtual void PrepMoveFor(ACharacter* Character) override
    {
        Super::PrepMoveFor(Character);
        UMyCharacterMovement* MoveComp =
            Cast<UMyCharacterMovement>(Character->GetCharacterMovement());
        if (MoveComp)
        {
            MoveComp->bWantsToWallRun = bWantsToWallRun;
        }
    }
};
```

And provide the custom prediction data class:

```cpp
class FNetworkPredictionData_Client_MyCharacter
    : public FNetworkPredictionData_Client_Character
{
public:
    virtual FSavedMovePtr AllocateNewMove() override
    {
        return FSavedMovePtr(new FSavedMove_MyCharacter());
    }
};

// In your custom CMC subclass:
FNetworkPredictionData_Client* UMyCharacterMovement::GetPredictionData_Client() const
{
    if (!ClientPredictionData)
    {
        UMyCharacterMovement* MutableThis =
            const_cast<UMyCharacterMovement*>(this);
        MutableThis->ClientPredictionData =
            new FNetworkPredictionData_Client_MyCharacter(*this);
    }
    return ClientPredictionData;
}
```

---

## 3. Gameplay Ability System (GAS) Prediction

The Gameplay Ability System (GAS) provides a higher-level prediction framework for abilities, attribute changes, gameplay effects, and gameplay cues. While the CMC predicts *movement*, GAS predicts *game actions and their side effects*.

### FPredictionKey: The Core Mechanism

The `FPredictionKey` is the central concept in GAS prediction. It is a unique integer identifier generated on the client that tags a predicted action and all of its side effects. This key is used to:

1. **Associate** client-side predictions with server-side confirmations
2. **Roll back** side effects if the server rejects the prediction
3. **Prevent duplication** when the server's replicated result arrives

A critical property: **FPredictionKey replicates client -> server normally, but when replicating server -> clients, it only replicates to the client that sent the key.** All other clients receive an invalid (0) key. This means only the predicting client can match server results to its local predictions.

```cpp
struct FPredictionKey
{
    int16 Current;      // The key value
    int16 Base;         // Base key (for dependency chains)
    bool bIsServerInitiated;

    // Only replicates back to the originating client
    bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);

    // Register a callback for when this key is rejected
    FPredictionKeyDelegates::FRejectedDelegate& NewRejectedDelegate();

    // Register a callback for when this key is caught up (confirmed)
    FPredictionKeyDelegates::FCaughtUpDelegate& NewCaughtUpDelegate();
};
```

### Ability Activation Flow

The full prediction lifecycle for ability activation:

```
Client                                  Server
  |                                       |
  |-- TryActivateAbility() -------------->|
  |   - Generate FPredictionKey           |
  |   - Call ServerTryActivateAbility RPC |
  |   - Immediately call ActivateAbility  |
  |     locally (don't wait for server)   |
  |   - Side effects get tagged with key  |
  |                                       |
  |                        ServerTryActivateAbility()
  |                        - Validate (cooldowns, costs, etc.)
  |                        - If valid: ActivateAbility on server
  |                        - If invalid: reject
  |                                       |
  |<---- ClientActivateAbilitySucceed ----|  (confirmation)
  |  or                                   |
  |<---- ClientActivateAbilityFailed -----|  (rejection)
  |   - If failed: end ability locally    |
  |   - Roll back all side effects with   |
  |     this FPredictionKey               |
```

### What GAS Can Predict

| Predicted | Not Predicted |
|-----------|---------------|
| Ability activation | GameplayEffect removal |
| Triggered events | Periodic effects (DoT ticks) |
| GameplayEffect application (modifiers) | GameplayEffect Executions |
| Gameplay tags | Meta attributes (damage pipeline) |
| Gameplay cues (VFX/SFX) | |
| Animation montages | |
| Movement (via CMC integration) | |

### GameplayEffect Prediction

GameplayEffects are treated as *side effects* of abilities rather than independently predicted actions. The system works as follows:

1. Client applies a GameplayEffect locally only if it has a **valid prediction key** (i.e., it was triggered during a predicted ability activation)
2. The effect modifies attributes, applies tags, and fires gameplay cues locally
3. The `FActiveGameplayEffect` stores the prediction key
4. When the server applies the same effect, it sets the same prediction key on the replicated effect
5. The client compares incoming replicated effects against local predicted effects by key
6. If keys match, the predicted effect is removed **without** re-triggering "on applied" logic (preventing double VFX, double sounds, etc.)

```cpp
// Inside UAbilitySystemComponent -- applying a predicted effect
FActiveGameplayEffectHandle UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(
    const FGameplayEffectSpec& Spec,
    FPredictionKey PredictionKey)
{
    // If we have a valid prediction key, apply locally
    if (PredictionKey.IsValidKey())
    {
        FActiveGameplayEffect& NewEffect = CreateNewActiveEffect(Spec);
        NewEffect.PredictionKey = PredictionKey;

        // Register rollback handler
        PredictionKey.NewRejectedDelegate().BindUObject(
            this,
            &UAbilitySystemComponent::OnPredictedEffectRejected,
            NewEffect.Handle);
    }
    // ... server applies authoritatively regardless
}
```

### Attribute Prediction: Delta-Based Approach

GAS predicts attribute changes using a **delta** (modifier-based) approach rather than absolute values. Instant modifications are internally treated as infinite-duration effects during prediction, which solves the problem of undoing and redoing predictions:

- The **replicated value** is treated as the **base value**
- Predicted modifiers stack on top of the base
- When the server's replicated value arrives, the base updates and modifiers are re-aggregated

This requires specific replication configuration:

```cpp
void UMyAttributeSet::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // REPNOTIFY_Always ensures we get callbacks even if value unchanged
    // (server may confirm same value we predicted)
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health,
        COND_None, REPNOTIFY_Always);
}

void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
    // This macro handles re-aggregation of predicted modifiers
    // on top of the new server-authoritative base value
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldHealth);
}
```

### Side Effect Rollback

When the server rejects a prediction, all side effects tagged with that `FPredictionKey` must be rolled back. The system uses delegate chains:

```cpp
// During predicted ability activation
void UAbilitySystemComponent::OnClientActivateAbilityFailed(
    FGameplayAbilitySpecHandle Handle,
    FPredictionKey::KeyType PredictionKey)
{
    // Find the ability spec
    FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(Handle);
    if (Spec)
    {
        // End the ability
        Spec->Ability->EndAbility(Handle, ...);
    }

    // The rejection delegate fires, which triggers:
    // - Removal of predicted GameplayEffects with this key
    // - Removal of predicted gameplay cues
    // - Reversal of predicted attribute modifications
    // - Stopping of predicted montages
}
```

### Prediction Key Dependency Chains

When ability X triggers ability Y which triggers ability Z, each generates its own prediction key but maintains a `Base` key relationship:

```
Ability X -> PredictionKey(Base=1, Current=1)
  triggers Ability Y -> PredictionKey(Base=1, Current=2)
    triggers Ability Z -> PredictionKey(Base=1, Current=3)
```

If Y is rejected, Z is also rejected through `FPredictionKeyDelegates::AddDependancy`. The server receives only X's key initially but runs the entire chain using that base key.

### Scoped Prediction Windows

`FScopedPredictionWindow` enables additional prediction windows within a single ability's lifetime. This handles scenarios like charged attacks where player input during execution requires new prediction:

```cpp
void UAbilityTask_WaitInputRelease::OnReleaseCallback()
{
    // Create a new prediction window for the release action
    FScopedPredictionWindow ScopedPrediction(
        AbilitySystemComponent,
        true /* bCanGenerateNewKey */);

    // Actions within this scope get the new scoped prediction key
    // Server receives this key via ServerInputRelease RPC
    ApplyReleaseEffects();
}
```

### Gameplay Cue Prediction

Gameplay cues (VFX, SFX) are predicted using the same key system:

```cpp
// In UAbilitySystemComponent::ExecuteGameplayCue
if (IsAuthority())
{
    // Server: multicast with replication key
    NetMulticast_InvokeGameplayCueExecuted(Tag, PredictionKey, ...);
}
else if (PredictionKey.IsValidKey())
{
    // Client: predict the cue locally
    InvokeGameplayCueEvent(Tag, EGameplayCueEvent::Executed, ...);
}

// When multicast arrives on predicting client:
// Skip execution because we already predicted it (matched by key)
```

---

## 4. Network Prediction Plugin (Experimental)

The Network Prediction (NP) plugin is an experimental Epic-provided framework designed to solve problems that the CMC and standard replication cannot handle well, particularly around physics-based gameplay and fixed-tick simulation. It is still a work-in-progress as of UE 5.x.

### Motivation

The standard CMC has limitations:

- **Variable tick rate:** Movement simulation runs at the game's frame rate, which varies between client and server. This causes determinism issues.
- **Character-only:** The CMC is built for `ACharacter`. Predicting vehicles, projectiles, or physics objects requires a completely different approach.
- **No built-in rollback:** The CMC replays saved moves, but there is no generic rollback framework for arbitrary simulation state.

The NP plugin addresses these by providing:

- A **fixed network tick** independent of frame rate
- A generic simulation/rollback framework for any actor
- Input buffering on both client and server
- Automatic reconciliation and resimulation

### Core Concepts

#### Fixed Network Tick

The plugin runs its simulation at a fixed timestep (configurable, e.g., 30Hz or 60Hz), independent of render frame rate. Both client and server tick at the same rate. This is crucial for determinism -- if client and server both run the same logic at the same fixed timestep with the same inputs, they should produce the same results.

#### SimulationTick

All simulation logic goes into a single pure function:

```cpp
void UMySimulation::SimulationTick(
    const FNetSimTimeStep& TimeStep,
    const TNetSimInput<MyBufferTypes>& Input,
    const TNetSimOutput<MyBufferTypes>& Output)
{
    // Read input
    const FMyInputCmd& InputCmd = *Input.Cmd;
    const FMySyncState& PrevSync = *Input.Sync;

    // Write output
    FMySyncState& OutSync = *Output.Sync;

    // Pure simulation logic -- same on client and server
    OutSync.Location = PrevSync.Location
        + InputCmd.MoveDirection * MoveSpeed * TimeStep.StepMS * 0.001f;
    OutSync.Velocity = InputCmd.MoveDirection * MoveSpeed;
}
```

This function is called on all machines (server, owning client, simulated proxies) and during rollback/resimulation.

#### State Types: Sync State vs Aux State

The plugin separates state into two categories:

**Sync State** -- Data that changes continuously every tick and must be synchronized:

```cpp
struct FMySyncState
{
    FVector Location;
    FVector Velocity;
    FRotator Rotation;

    // Serialize for network transmission
    void NetSerialize(const FNetSerializeParams& P)
    {
        P.Ar << Location;
        P.Ar << Velocity;
        P.Ar << Rotation;
    }

    // Should we reconcile? Compare client prediction against server state
    bool ShouldReconcile(const FMySyncState& AuthoritativeState) const
    {
        const float ErrorTolerance = 1.0f; // units
        return !Location.Equals(AuthoritativeState.Location, ErrorTolerance)
            || !Velocity.Equals(AuthoritativeState.Velocity, ErrorTolerance);
    }

    // For interpolation on simulated proxies
    void Interpolate(const FMySyncState& Other, float Alpha)
    {
        Location = FMath::Lerp(Location, Other.Location, Alpha);
        Velocity = FMath::Lerp(Velocity, Other.Velocity, Alpha);
        Rotation = FMath::Lerp(Rotation, Other.Rotation, Alpha);
    }

    void ToString(FAnsiStringBuilderBase& Out) const
    {
        Out.Appendf("Loc: (%.2f, %.2f, %.2f)", Location.X, Location.Y, Location.Z);
    }
};
```

**Aux State** -- Data that changes infrequently and affects simulation behavior:

```cpp
struct FMyAuxState
{
    float MoveSpeed = 600.f;
    bool bIsStunned = false;

    void NetSerialize(const FNetSerializeParams& P)
    {
        P.Ar << MoveSpeed;
        P.Ar << bIsStunned;
    }

    bool ShouldReconcile(const FMyAuxState& AuthoritativeState) const
    {
        return MoveSpeed != AuthoritativeState.MoveSpeed
            || bIsStunned != AuthoritativeState.bIsStunned;
    }

    void Interpolate(const FMyAuxState& Other, float Alpha)
    {
        // Aux state typically snaps rather than interpolates
        if (Alpha > 0.5f)
        {
            *this = Other;
        }
    }

    void ToString(FAnsiStringBuilderBase& Out) const
    {
        Out.Appendf("Speed: %.1f Stunned: %d", MoveSpeed, bIsStunned);
    }
};
```

#### Input Command

```cpp
struct FMyInputCmd
{
    FVector MoveDirection = FVector::ZeroVector;
    bool bJumpPressed = false;

    void NetSerialize(const FNetSerializeParams& P)
    {
        P.Ar << MoveDirection;
        P.Ar << bJumpPressed;
    }

    void ToString(FAnsiStringBuilderBase& Out) const
    {
        Out.Appendf("Dir: (%.2f, %.2f, %.2f) Jump: %d",
            MoveDirection.X, MoveDirection.Y, MoveDirection.Z, bJumpPressed);
    }
};
```

The system sends the latest 6 frames of input with each packet to handle packet loss.

### Model Definition

The plugin uses a model definition struct to wire everything together:

```cpp
class FMyNetworkModelDef : public FNetworkPredictionModelDef
{
public:
    using Simulation = UMySimulation;
    using StateTypes = TNetworkPredictionStateTypes<FMyInputCmd, FMySyncState, FMyAuxState>;
    using Driver = UMyMovementComponent;

    static const TCHAR* GetName() { return TEXT("MyModel"); }
    static constexpr int32 GetSortPriority() { return 0; }
};
```

### Driver Component

The driver component connects the NP plugin to an actor:

```cpp
UCLASS()
class UMyMovementComponent : public UActorComponent, public INetworkPredictionComponent
{
    GENERATED_BODY()

public:
    // Called once to initialize the NP proxy
    void InitializeNetworkPredictionProxy() override
    {
        NetworkPredictionProxy.Init<FMyNetworkModelDef>(
            GetWorld(), GetReplicationProxies(), this, this);
    }

    // Set initial simulation state
    void InitializeSimulationState(FMySyncState* Sync, FMyAuxState* Aux)
    {
        Sync->Location = GetOwner()->GetActorLocation();
        Sync->Velocity = FVector::ZeroVector;
        Aux->MoveSpeed = 600.f;
    }

    // Called every network tick to provide input (owning client only)
    void ProduceInput(const int32 DeltaTimeMS, FMyInputCmd* Cmd)
    {
        // Sample current input state
        Cmd->MoveDirection = GetPendingInputVector();
        Cmd->bJumpPressed = bWantsToJump;
    }

    // Called after simulation tick to apply results to the actor
    void FinalizeFrame(const FMySyncState* Sync, const FMyAuxState* Aux)
    {
        GetOwner()->SetActorLocation(Sync->Location);
        GetOwner()->SetActorRotation(Sync->Rotation);
    }

    // Optional: called before frame restoration during resimulation
    void RestoreFrame(const FMySyncState* Sync, const FMyAuxState* Aux)
    {
        GetOwner()->SetActorLocation(Sync->Location);
    }

private:
    FNetworkPredictionProxy NetworkPredictionProxy;
};
```

### Rollback Mechanism

When `ShouldReconcile()` returns true (client state diverges from server state), the plugin automatically:

1. **Reverts** to the most recent server-confirmed state
2. **Resimulates** every frame from that point forward, calling `SimulationTick()` for each frame with the saved inputs
3. The client catches up to the current frame with corrected state

This is transparent to the developer -- as long as `SimulationTick` is a pure function of inputs and previous state, rollback works automatically.

#### Detecting Rollback in Code

You can check whether you are in a resimulation context:

```cpp
void UMySimulation::SimulationTick(
    const FNetSimTimeStep& TimeStep,
    const TNetSimInput<MyBufferTypes>& Input,
    const TNetSimOutput<MyBufferTypes>& Output)
{
    // Check if we're resimulating after a correction
    if (Output.CueDispatch.GetContext().TickContext
        == ESimulationTickContext::Resimulate)
    {
        // We're replaying -- skip spawning cosmetic effects
    }
}
```

### Network Cues

The NP plugin provides its own event system for cosmetic effects that need to be robust during rollback:

```cpp
struct FMyFireCue
{
    NETSIMCUE_BODY();

    FVector FireLocation;
    FRotator FireRotation;

    void NetSerialize(FArchive& Ar)
    {
        Ar << FireLocation;
        Ar << FireRotation;
    }

    bool NetIdentical(const FMyFireCue& Other) const
    {
        return FireLocation.Equals(Other.FireLocation, 1.f);
    }
};

// Register cue traits
template<>
struct TNetSimCueTraits<FMyFireCue>
{
    static constexpr ENetSimCueReplicationTarget ReplicationTarget =
        ENetSimCueReplicationTarget::All;
    static constexpr bool bResimOnRollback = false; // Don't replay VFX
};
```

### Modifying State from Outside Simulation

External systems (e.g., an ability applying a speed buff) can write to state through the proxy:

```cpp
// Apply a speed buff from outside the simulation tick
NetworkPredictionProxy.WriteAuxState<FMyAuxState>(
    [](FMyAuxState& Aux)
    {
        Aux.MoveSpeed = 1200.f;
    },
    "SpeedBuffApplied");
```

### Why Physics Syncing Is Broken

The NP plugin has a fundamental problem with Unreal's physics engine (Chaos):

1. **Chaos runs at its own fixed tick rate** that does not align with the NP plugin's fixed tick rate. Synchronizing two independent fixed-rate simulations is extremely difficult.

2. **Physics state is opaque.** The NP plugin needs to save and restore complete simulation state for rollback. But the Chaos physics engine does not expose a clean "snapshot and restore" API for individual bodies within a shared physics scene. Saving the transform of one body is insufficient -- contact pairs, constraint states, solver iterations, and broadphase state all matter.

3. **Rollback requires re-stepping physics.** When the NP plugin detects a mismatch and needs to resimulate, it would need to rewind the entire physics scene (or at least the relevant bodies) and re-step it frame by frame. Chaos does not support this efficiently.

4. **In the source code, the physics reconciliation path hits `check(false)`** -- a deliberate assertion failure indicating that Epic has not implemented this path and considers it non-functional:

```cpp
// In NetworkPredictionPhysics.cpp (approximate)
void FPhysicsReconcileSimulation::ShouldReconcile(...)
{
    // Physics reconciliation is not implemented
    check(false); // Intentional crash -- this code path is broken
}
```

This means that any NP plugin simulation that relies on Unreal's physics engine for its core state (e.g., physics-driven vehicles, ragdolls, physics projectiles) **cannot use the built-in rollback mechanism**. Developers must either:

- Write their own physics simulation that bypasses Chaos (e.g., simple kinematic projectile math)
- Accept that physics-based objects will not reconcile properly
- Wait for Epic to ship a fix (this has been outstanding for multiple engine versions)

### Using NP for Projectiles (Practical Approach)

Steve Streeting documented a practical approach for predicted projectiles that works around the NP plugin's limitations:

**The core idea:** Rather than full rollback, use a "predicted actor + replicated actor" pattern with smooth interpolation.

1. During the fire animation wind-up, the client generates a unique `PredictedActorID` and sends the fire RPC to the server
2. The client delays spawning its local predicted projectile by an estimated amount (approximately half the round-trip time), absorbing latency into the animation
3. The server spawns the authoritative projectile and fast-forwards it by (actual lag - client delay) to account for transit time
4. When the replicated projectile arrives on the owning client, it is matched to the predicted projectile by ID
5. The replicated actor is hidden; the predicted actor disables collision and interpolates toward the replicated actor's position each frame

```cpp
// Predicted actor follows the replicated (server) actor
void AMyPredictedProjectile::TickFollowMode(float DeltaTime)
{
    if (AActor* ServerActor = ReplicatedActor.Get())
    {
        SetActorLocation(
            FMath::VInterpTo(GetActorLocation(),
                ServerActor->GetActorLocation(),
                DeltaTime, 10.f));

        SetActorRotation(
            FMath::RInterpTo(GetActorRotation(),
                ServerActor->GetActorRotation(),
                DeltaTime, 10.f));
    }
}
```

This approach trades perfect reconciliation for visual smoothness: the client sees a responsive projectile that gradually converges with server truth.

---

## 5. How These Systems Overlap

### Coverage Map

| Concern | CMC | GAS | NP Plugin |
|---------|-----|-----|-----------|
| Character walking/running/jumping | Primary | -- | Replacement (planned) |
| Ability activation | -- | Primary | -- |
| Attribute changes (health, mana) | -- | Primary | -- |
| Gameplay cues (VFX/SFX) | -- | Primary | Network Cues |
| Custom movement modes | Extension | -- | Replacement |
| Vehicles / physics objects | -- | -- | Primary (if physics worked) |
| Projectiles | -- | Partial (spawning via ability) | Experimental |
| Animation montages | -- | Primary | -- |

### When to Use Each

**Use CharacterMovementComponent when:**
- You need standard character movement (walk, run, jump, crouch, swim, fly)
- You are extending movement with custom modes but the character paradigm still fits
- Your game tolerates variable-tick simulation (most games do)
- You want the most battle-tested, production-proven solution

**Use GAS prediction when:**
- You are activating abilities (attacks, spells, buffs)
- You need to predict attribute changes (damage, healing, resource costs)
- You need to predict cosmetic effects (VFX, SFX) tied to abilities
- You need dependency chains (ability X triggers ability Y)
- You are already using GAS for your gameplay systems

**Use the Network Prediction Plugin when:**
- You need deterministic fixed-tick simulation
- You are building non-character predicted movement (vehicles, custom movers)
- You need a generic rollback framework
- You are willing to work with experimental, incomplete technology
- Your simulation does NOT depend on Chaos physics for core state

### How They Interact

- **CMC + GAS:** These are designed to work together. GAS abilities can trigger movement changes (dashes, teleports) through the CMC, and the CMC's movement prediction is independent of GAS's ability prediction. GAS's `UAbilityTask_ApplyRootMotionConstantForce` and similar tasks integrate with the CMC's movement pipeline.

- **CMC + NP Plugin:** These are **mutually exclusive** for the same actor. The NP plugin is intended as a long-term replacement for the CMC's networking layer. Epic has stated that the Mover 2.0 system will eventually use the NP plugin instead of the CMC's built-in prediction. You cannot use both on the same character.

- **GAS + NP Plugin:** These do **not integrate well**. The NP plugin's fixed-tick buffering creates a different timeline than GAS's frame-based prediction. Kieran Newland's research confirms that mixing the two causes synchronization issues: "interactions will trigger earlier than when they would inside NP." Epic recommends building a custom ability system if you commit to the NP plugin. This is effectively an "all or nothing" decision.

---

## 6. Trade-offs and Known Issues

### CharacterMovementComponent

| Advantage | Disadvantage |
|-----------|--------------|
| Battle-tested in shipped AAA titles | Variable tick causes subtle determinism issues |
| Handles complex movement modes | Extending requires subclassing multiple classes |
| Automatic move combining reduces bandwidth | Compressed flags limit custom input channels |
| Well-documented with community resources | Only works for ACharacter-derived actors |
| Simulated proxy interpolation is smooth | Large corrections cause visible teleporting despite smoothing |

**Known issues:**
- **Listen server advantage:** The listen server host has zero latency, giving them an inherent advantage in competitive games. Dedicated servers mitigate this.
- **Physics interactions:** Characters interacting with physics objects (standing on moving platforms, being pushed) are a persistent source of mispredictions because the physics state is not part of the saved move.
- **High-latency degradation:** At 200ms+ ping, the pending move list grows large, and corrections trigger long replay chains that can cause CPU spikes.
- **Timestamp manipulation:** Malicious clients can send manipulated timestamps. The server has some protection (`ServerCheckClientError`) but it is not comprehensive.

### Gameplay Ability System

| Advantage | Disadvantage |
|-----------|--------------|
| Comprehensive prediction for abilities | Steep learning curve; many interacting systems |
| Delta-based attribute prediction is robust | Cannot predict effect removal |
| Dependency chains handle complex interactions | Periodic effects (DoTs) are not predicted |
| Gameplay cue prediction prevents double-play | Meta attributes (damage pipeline) cannot be predicted |
| Server-authoritative with client responsiveness | Multiplicative/percentage modifiers have prediction accuracy issues |

**Known issues:**
- **Effect removal not predicted:** If the server removes a GameplayEffect, the client cannot predict this. The effect persists on the client until the server's removal replicates.
- **Stacked percentage modifiers:** Since the server replicates final attribute values but not the full aggregator chain, clients may incorrectly predict stacked percentage bonuses when multiple modifiers interact.
- **Cross-player events:** Triggered events do not replicate. Server-only triggered events from AI or other players never reach the predicting client, preventing prediction of reactions to external events.
- **Weak prediction:** The documentation and implementation for "weak prediction" (prediction that can fail without full rollback) appears incomplete.

### Network Prediction Plugin

| Advantage | Disadvantage |
|-----------|--------------|
| Fixed tick ensures determinism | Physics reconciliation is broken (check(false)) |
| Generic rollback for any simulation | Still experimental / WIP across UE5.x versions |
| Input buffering handles packet loss | Requires "all or nothing" adoption -- cannot mix with standard RPCs easily |
| Network cues handle cosmetic rollback | Does not integrate with GAS |
| Clean separation of concerns (input/state/simulation) | Limited documentation; mostly community-driven |
| Works for non-character actors | Mover 2.0 integration not yet shipped |

**Known issues:**
- **Physics is fundamentally broken:** The `check(false)` in physics reconciliation means any physics-dependent simulation cannot roll back. This is the single largest blocker for adoption.
- **GAS incompatibility:** The buffering/timeline differences make GAS integration unreliable. Epic's own recommendation is to build a custom ability system if using NP.
- **Frame index access:** Getting the current simulation frame index from outside the simulation is inconsistent, making it difficult for external systems to coordinate with NP-managed state.
- **Sparse documentation:** The primary resources are community blog posts and reading the engine source code. Epic has not published comprehensive documentation.
- **API instability:** As an experimental plugin, the API has changed between engine versions without deprecation warnings.

---

## 7. References

### Official Documentation
- [Unreal Engine Networking Overview](https://docs.unrealengine.com/5.3/en-US/networking-overview-for-unreal-engine/)
- [Character Movement Component Networking](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/Networking/CharacterMovementComponent/)
- [Gameplay Ability System Documentation](https://docs.unrealengine.com/5.3/en-US/gameplay-ability-system-for-unreal-engine/)
- [Property Replication](https://docs.unrealengine.com/5.3/en-US/replicated-properties-in-unreal-engine/)

### Community Resources
- [Kieran Newland - Using The Network Prediction Plugin](https://www.kierannewland.co.uk/using-the-network-prediction-plugin/) -- Detailed walkthrough of NP plugin architecture, state types, rollback, and known limitations.
- [Steve Streeting - Unreal Network Prediction for Projectiles](https://www.stevestreeting.com/2024/12/12/unreal-network-prediction-for-projectiles/) -- Practical approaches to predicted projectiles, comparing basic server, client-authoritative, and hybrid prediction strategies.
- [ikrima - GAS Networking Notes](https://ikrima.dev/ue4guide/gameplay-programming/gameplay-ability-system/gas-networking/) -- Comprehensive breakdown of FPredictionKey, side effect rollback, dependency chains, and scoped prediction windows.
- [GASDocumentation by tranek](https://github.com/tranek/GASDocumentation) -- Community-maintained deep dive into GAS, including networking and prediction sections.
- [Unreal Engine Network Compendium by Cedric Neukirchen](https://cedric-neukirchen.net/docs/category/multiplayer-network-compendium) -- Broad overview of Unreal networking concepts.

### Source Code References (Engine)
- `Engine/Source/Runtime/Engine/Private/Components/CharacterMovementComponent.cpp` -- CMC prediction, FSavedMove, server validation
- `Engine/Source/Runtime/Engine/Public/GameFramework/CharacterMovementReplication.h` -- FNetworkPredictionData structures
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp` -- GAS prediction and FPredictionKey handling
- `Engine/Plugins/Runtime/NetworkPrediction/Source/NetworkPrediction/` -- NP plugin core framework
- `Engine/Plugins/Runtime/NetworkPredictionExtras/Source/` -- NP plugin example implementations (MockCharacter, MockPhysics)
