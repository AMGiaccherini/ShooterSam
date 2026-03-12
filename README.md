# Shooter Sam — Learning Log

Project built while following an Unreal Engine 5 course. It's a third-person shooter where the player controls an armed character facing enemies driven by a Behavior Tree AI. Built on top of the UE5 Third Person template.

**Engine:** Unreal Engine 5  
**Language:** C++ + Blueprint  
**Additional UE Modules:** UMG (UI), EnhancedInput, Niagara, AIModule, GameplayTasks

---

## The Game

The player moves in third person, aims with the mouse and shoots with a weapon attached to the character. AI enemies patrol the area, chase the player when spotted and shoot when in range. A health bar is visible on the HUD.

### Controls

| Input | Action |
|---|---|
| WASD | Move |
| Mouse | Rotate camera |
| Left Click | Shoot |
| Space | Jump |

---

## Class Architecture

```
ACharacter
└── AShooterSamCharacter         ← base character shared by player and AI

APlayerController
└── AShooterSamPlayerController  ← manages IMC and HUD

AAIController
└── AShooterAI                   ← starts the Behavior Tree, exposes character references

AActor
└── AGun                         ← weapon: line trace, damage, effects

UGameModeBase
└── AShooterSamGameMode          ← starts all enemy Behavior Trees on BeginPlay

UUserWidget
└── UHUDWidget                   ← player health bar

UBTTaskNode
└── UBTTaskNode_Shoot            ← BT Task: orders the AI character to shoot

UBTTask_BlackboardBase
└── UBTTask_ClearBlackboardValue ← BT Task: clears a value from the Blackboard

UBTService_BlackboardBase
├── UBTService_PlayerLocation          ← BT Service: updates player location every tick
└── UBTService_PlayerLocationIfSeen    ← BT Service: updates location only if in line of sight
```

---

## Concepts Learned

### Behavior Tree and Blackboard

UE5's AI system is based on two separate assets: the **Behavior Tree** (the decision tree) and the **Blackboard** (a key-value memory shared between the BT and the AIController). The AI reads and writes to the Blackboard to make decisions. For example, `BTService_PlayerLocation` writes the player's position every tick, and the BT uses that value to decide whether to chase or patrol.

### BTService: Continuous Updates

**Services** are BT nodes that execute logic at regular intervals while their branch is active. You extend `UBTService_BlackboardBase` and override `TickNode`. This project has two services with different behavior:

- `BTService_PlayerLocation` updates the player position **always**
- `BTService_PlayerLocationIfSeen` updates the position **only if the enemy has line of sight** on the player, otherwise clears the value — this lets the BT distinguish between "I know where you are" and "I've lost you"

```cpp
if (OwnerController->LineOfSightTo(Player))
{
    Blackboard->SetValueAsVector(GetSelectedBlackboardKey(), Player->GetActorLocation());
    OwnerController->SetFocus(Player);
}
else
{
    Blackboard->ClearValue(GetSelectedBlackboardKey());
    OwnerController->ClearFocus(EAIFocusPriority::Gameplay);
}
```

### BTTask: Atomic Actions

**Tasks** are the leaves of the Behavior Tree — they execute an action and return `Succeeded` or `Failed`. You extend `UBTTaskNode` (or `UBTTask_BlackboardBase` when working with a Blackboard key) and override `ExecuteTask`.

`BTTaskNode_Shoot` retrieves the AI character reference through the AIController and calls `Shoot()` only if the player is still alive:

```cpp
if (OwnerCharacter && PlayerCharacter && PlayerCharacter->IsAlive)
{
    OwnerCharacter->Shoot();
    Result = EBTNodeResult::Succeeded;
}
```

`BTTask_ClearBlackboardValue` is a generic reusable task that clears any Blackboard key selected in the editor, without hardcoding the key name.

### Line Trace (Hitscan)

`AGun::PullTrigger` uses a **line trace** instead of spawning a physical projectile. The ray starts from the controller's ViewPoint and travels in the camera direction for `MaxRange` units. This approach is called **hitscan** and is common in shooters because it's more predictable and performant than physical projectiles.

```cpp
OwnerController->GetPlayerViewPoint(ViewPointLocation, ViewPointRotation);
FVector EndLocation = ViewPointLocation + ViewPointRotation.Vector() * MaxRange;
bool IsHit = GetWorld()->LineTraceSingleByChannel(HitResult, ViewPointLocation, EndLocation, ECC_GameTraceChannel2, Params);
```

Note the use of `ECC_GameTraceChannel2`: a **custom collision channel** defined in the project to filter what can be hit by bullets, separate from general visibility.

### Gun as a Separate Actor Attached to a Socket

The weapon is an independent `AActor` (`AGun`) spawned in `BeginPlay` and attached to the character mesh via a named socket (`WeaponSocket`). The weapon bone in the hand is hidden with `HideBoneByName` to avoid double mesh.

```cpp
GetMesh()->HideBoneByName("weapon_r", EPhysBodyOp::PBO_None);
Gun = GetWorld()->SpawnActor<AGun>(GunClass);
Gun->AttachToComponent(GetMesh(), FAttachmentTransformRules::KeepRelativeTransform, TEXT("WeaponSocket"));
```

### HUD with ProgressBar and BindWidget

`UHUDWidget` uses a `UProgressBar` with `meta = (BindWidgetOptional)`: the "Optional" variant means the widget won't crash if the component is missing in the Blueprint, making it more robust during development. The `SetHealthBarPercent` method validates the `[0, 1]` range before applying it.

The character updates the HUD by calling `UpdateHUD()` every time it takes damage, casting its controller to `AShooterSamPlayerController` to access the widget instance.

### Handling Death Without Destroy

When HP reaches zero the character is not destroyed but disabled:

```cpp
IsAlive = false;
Health = 0.0f;
GetCapsuleComponent()->SetCollisionEnabled(ECollisionEnabled::NoCollision);
DetachFromControllerPendingDestroy();
```

`DetachFromControllerPendingDestroy()` detaches the controller from the pawn leaving the mesh visible in the scene. `IsAlive` is `BlueprintReadOnly` so it can be read by Blueprints without being modifiable from outside.

### GameMode with Range-Based For Loop

`AShooterSamGameMode::BeginPlay` shows in code comments the evolution of three equivalent approaches to iterate an array: `while` with manual index, `for` with index, and finally the **C++11 range-based for loop** which is the most readable and the one definitively adopted.

```cpp
for (AActor* ShooterAIActor : ShooterAIActors)
{
    AShooterAI* ShooterAI = Cast<AShooterAI>(ShooterAIActor);
    if (ShooterAI)
    {
        ShooterAI->StartBehaviorTree(Player);
    }
}
```

### Custom Log Category

The project defines a custom log category (`LogShooterSam`) in `ShooterSam.h` with `DECLARE_LOG_CATEGORY_EXTERN` and registers it in `ShooterSam.cpp` with `DEFINE_LOG_CATEGORY`. Using it instead of `LogTemp` allows filtering logs by category in the editor's Output Log.
