# Shooter Sam — Learning Log

Progetto realizzato seguendo un corso di Unreal Engine 5. È un third-person shooter in cui il giocatore controlla un personaggio armato che affronta nemici controllati da un'AI basata su Behavior Tree. Il progetto è costruito a partire dal template Third Person di UE5.

**Engine:** Unreal Engine 5  
**Linguaggio:** C++ + Blueprint  
**Moduli UE aggiuntivi:** UMG (UI), EnhancedInput, Niagara, AIModule, GameplayTasks

---

## Il Gioco

Il giocatore si muove in terza persona, mira con il mouse e spara con un'arma attaccata al personaggio. I nemici AI pattugliano l'area, inseguono il giocatore quando lo vedono e sparano quando sono in range. La barra della vita è visibile sull'HUD.

### Controlli

| Input | Azione |
|---|---|
| WASD | Movimento |
| Mouse | Ruota la camera |
| Click Sinistro | Spara |
| Spazio | Salta |

---

## Architettura delle Classi

```
ACharacter
└── AShooterSamCharacter         ← personaggio base condiviso da player e AI

APlayerController
└── AShooterSamPlayerController  ← gestisce IMC e HUD

AAIController
└── AShooterAI                   ← avvia il Behavior Tree, espone riferimenti al personaggio

AActor
└── AGun                         ← arma: line trace, danno, effetti

UGameModeBase
└── AShooterSamGameMode          ← avvia i Behavior Tree di tutti i nemici al BeginPlay

UUserWidget
└── UHUDWidget                   ← barra della vita del giocatore

UBTTaskNode
└── UBTTaskNode_Shoot            ← Task BT: ordina al personaggio AI di sparare

UBTTask_BlackboardBase
└── UBTTask_ClearBlackboardValue ← Task BT: cancella un valore dal Blackboard

UBTService_BlackboardBase
├── UBTService_PlayerLocation          ← Service BT: aggiorna posizione player ogni tick
└── UBTService_PlayerLocationIfSeen    ← Service BT: aggiorna posizione solo se in line of sight
```

---

## Concetti Appresi

### Behavior Tree e Blackboard

Il sistema AI di UE5 è basato su due asset separati: il **Behavior Tree** (l'albero decisionale) e il **Blackboard** (una memoria chiave-valore condivisa tra il BT e l'AIController). L'AI legge e scrive sul Blackboard per prendere decisioni. Ad esempio, `BTService_PlayerLocation` scrive la posizione del giocatore ogni tick, e il BT usa quel valore per decidere se inseguire o pattugliare.

### BTService: aggiornamento continuo

I **Services** sono nodi del BT che eseguono logica a intervalli regolari finché il loro ramo è attivo. Si estende `UBTService_BlackboardBase` e si fa override di `TickNode`. In questo progetto ci sono due service con comportamento diverso:

- `BTService_PlayerLocation` aggiorna la posizione del player **sempre**
- `BTService_PlayerLocationIfSeen` aggiorna la posizione **solo se il nemico ha line of sight** sul player, altrimenti cancella il valore — questo permette al BT di distinguere "so dove sei" da "ti ho perso di vista"

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

### BTTask: azioni atomiche

I **Tasks** sono le foglie del Behavior Tree — eseguono un'azione e restituiscono `Succeeded` o `Failed`. Si estende `UBTTaskNode` (o `UBTTask_BlackboardBase` se si lavora con una chiave del Blackboard) e si fa override di `ExecuteTask`.

`BTTaskNode_Shoot` recupera il riferimento al personaggio AI tramite l'AIController e chiama `Shoot()` solo se il player è ancora vivo:

```cpp
if (OwnerCharacter && PlayerCharacter && PlayerCharacter->IsAlive)
{
    OwnerCharacter->Shoot();
    Result = EBTNodeResult::Succeeded;
}
```

`BTTask_ClearBlackboardValue` è un task generico riutilizzabile che cancella qualsiasi chiave del Blackboard selezionata nell'editor, senza hardcodare il nome della chiave.

### Line Trace (Hitscan)

`AGun::PullTrigger` usa un **line trace** invece di spawnare un proiettile fisico. Il raggio parte dal ViewPoint del controller e va nella direzione della camera per `MaxRange` unità. Questo approccio è detto **hitscan** ed è comune negli sparatutto perché è più prevedibile e performante dei proiettili fisici.

```cpp
OwnerController->GetPlayerViewPoint(ViewPointLocation, ViewPointRotation);
FVector EndLocation = ViewPointLocation + ViewPointRotation.Vector() * MaxRange;
bool IsHit = GetWorld()->LineTraceSingleByChannel(HitResult, ViewPointLocation, EndLocation, ECC_GameTraceChannel2, Params);
```

Si noti l'uso di `ECC_GameTraceChannel2`: un **custom collision channel** definito nel progetto per filtrare cosa può essere colpito dai proiettili, separato dalla visibilità generica.

### Gun come Actor separato attaccato al socket

L'arma è un `AActor` indipendente (`AGun`) spawnato in `BeginPlay` e attaccato alla mesh del personaggio tramite un socket nominato (`WeaponSocket`). L'osso dell'arma nella mano viene nascosto con `HideBoneByName` per evitare la doppia mesh.

```cpp
GetMesh()->HideBoneByName("weapon_r", EPhysBodyOp::PBO_None);
Gun = GetWorld()->SpawnActor<AGun>(GunClass);
Gun->AttachToComponent(GetMesh(), FAttachmentTransformRules::KeepRelativeTransform, TEXT("WeaponSocket"));
```

### HUD con ProgressBar e BindWidget

`UHUDWidget` usa una `UProgressBar` con `meta = (BindWidgetOptional)`: la variante "Optional" significa che il widget non crasha se il componente non è presente nel Blueprint, rendendola più robusta durante lo sviluppo. Il metodo `SetHealthBarPercent` valida il range `[0, 1]` prima di applicarlo.

Il personaggio aggiorna l'HUD chiamando `UpdateHUD()` ogni volta che subisce danno, castando il proprio controller a `AShooterSamPlayerController` per accedere all'istanza del widget.

### Gestione della morte senza Destroy

Quando gli HP arrivano a zero il personaggio non viene distrutto ma disabilitato:

```cpp
IsAlive = false;
Health = 0.0f;
GetCapsuleComponent()->SetCollisionEnabled(ECollisionEnabled::NoCollision);
DetachFromControllerPendingDestroy();
```

`DetachFromControllerPendingDestroy()` stacca il controller dal pawn lasciando il mesh visibile nella scena. `IsAlive` è `BlueprintReadOnly` così può essere letto dai Blueprint senza essere modificabile dall'esterno.
