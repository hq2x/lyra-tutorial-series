# AI 与行为树实现

## 概述

在现代多人游戏中，AI Bot 不仅需要填补玩家空缺、提供训练对手，还要在单人体验中提供智能挑战和协作队友。Lyra 提供了一套完整的 AI 架构，融合了 **模块化 AI Controller**、**GAS 能力系统**、**团队系统** 和 **感知系统**，为构建高质量游戏 AI 提供了坚实基础。

本文将深入解析 Lyra 的 AI 实现方案，从架构设计到实战案例，帮助你构建智能、高效、可扩展的游戏 AI。

### 学习目标

- 理解 Lyra AI 架构设计理念
- 掌握 AIController 和 Perception System 的使用
- 学会设计可复用的 Behavior Tree
- 实现与 GAS 集成的 AI 技能系统
- 优化导航和寻路性能
- 了解 AI 决策系统（Utility AI/GOAP）
- 处理多人游戏中的 AI 同步问题
- 掌握 AI 调试和性能优化技巧

---

## 1. Lyra AI 架构设计

### 1.1 整体架构

Lyra 的 AI 架构基于 UE5 的模块化设计理念，核心组件包括：

```
┌─────────────────────────────────────────────────────────────┐
│                    Lyra AI 架构层次                          │
├─────────────────────────────────────────────────────────────┤
│  Game Mode Layer                                             │
│  └─ LyraBotCreationComponent (Bot 生命周期管理)             │
├─────────────────────────────────────────────────────────────┤
│  Controller Layer                                            │
│  ├─ ModularAIController (基础 AI 控制器)                    │
│  └─ LyraPlayerBotController (玩家 Bot 实现)                 │
├─────────────────────────────────────────────────────────────┤
│  System Integration                                          │
│  ├─ AIPerceptionComponent (感知系统)                         │
│  ├─ LyraTeamAgentInterface (团队接口)                       │
│  └─ AbilitySystemComponent (能力集成)                       │
├─────────────────────────────────────────────────────────────┤
│  Pawn Layer                                                  │
│  ├─ LyraPawn (可被 AI 控制的 Pawn)                          │
│  ├─ LyraPawnExtensionComponent (扩展组件)                   │
│  └─ LyraHealthComponent (生命值管理)                        │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 ModularAIController - 模块化基础

Lyra 使用 `ModularAIController` 作为所有 AI 控制器的基类，支持 Game Feature Plugins 动态扩展：

```cpp
// ModularAIController.h
UCLASS(MinimalAPI, Blueprintable)
class AModularAIController : public AAIController
{
    GENERATED_BODY()

public:
    // 支持组件的动态注入
    virtual void PreInitializeComponents() override;
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
};
```

**设计优势：**
- **Game Feature 集成**：支持通过插件动态添加 AI 功能
- **组件化扩展**：可在运行时注入新的 AI 组件
- **与 Experience 系统配合**：根据游戏模式加载不同的 AI 行为

### 1.3 LyraPlayerBotController - 玩家 Bot 控制器

`LyraPlayerBotController` 是 Lyra 中玩家 Bot 的核心实现，继承自 `ModularAIController`，实现了团队接口：

```cpp
// LyraPlayerBotController.h
UCLASS(Blueprintable)
class ALyraPlayerBotController : public AModularAIController, 
                                  public ILyraTeamAgentInterface
{
    GENERATED_BODY()

public:
    ALyraPlayerBotController(const FObjectInitializer& ObjectInitializer);

    // 团队接口实现
    virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;
    virtual FGenericTeamId GetGenericTeamId() const override;
    virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
    virtual ETeamAttitude::Type GetTeamAttitudeTowards(const AActor& Other) const override;

    // Bot 控制
    void ServerRestartController();
    
    // 更新 AI 感知的团队态度
    UFUNCTION(BlueprintCallable, Category = "Lyra AI Player Controller")
    void UpdateTeamAttitude(UAIPerceptionComponent* AIPerception);

    virtual void OnUnPossess() override;

protected:
    virtual void OnPlayerStateChanged();
    virtual void InitPlayerState() override;
    virtual void CleanupPlayerState() override;
    virtual void OnRep_PlayerState() override;

private:
    UPROPERTY()
    FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;

    UPROPERTY()
    TObjectPtr<APlayerState> LastSeenPlayerState;
};
```

**核心功能：**

1. **团队管理**：通过 `ILyraTeamAgentInterface` 集成团队系统
2. **PlayerState 集成**：Bot 拥有完整的 PlayerState，与真实玩家一致
3. **GAS 支持**：在 UnPossess 时清理 AbilitySystemComponent 的 Avatar
4. **态度判定**：基于团队 ID 判定敌友关系

### 1.4 LyraBotCreationComponent - Bot 生命周期管理

`LyraBotCreationComponent` 负责 Bot 的创建和销毁，作为 GameState 组件存在：

```cpp
// LyraBotCreationComponent.h
UCLASS(Blueprintable, Abstract)
class ULyraBotCreationComponent : public UGameStateComponent
{
    GENERATED_BODY()

protected:
    // Bot 数量配置
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Gameplay)
    int32 NumBotsToCreate = 5;

    // Bot 控制器类
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Gameplay)
    TSubclassOf<AAIController> BotControllerClass;

    // Bot 名称池
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Gameplay)
    TArray<FString> RandomBotNames;

    // 已生成的 Bot 列表
    UPROPERTY(Transient)
    TArray<TObjectPtr<AAIController>> SpawnedBotList;

    // Bot 生成和移除
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Gameplay)
    virtual void SpawnOneBot();

    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Gameplay)
    virtual void RemoveOneBot();

    UFUNCTION(BlueprintNativeEvent, BlueprintAuthorityOnly, Category=Gameplay)
    void ServerCreateBots();
};
```

**实现细节：**

```cpp
// LyraBotCreationComponent.cpp
void ULyraBotCreationComponent::SpawnOneBot()
{
    FActorSpawnParameters SpawnInfo;
    SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    SpawnInfo.OverrideLevel = GetComponentLevel();
    SpawnInfo.ObjectFlags |= RF_Transient;
    
    // 生成 AI 控制器
    AAIController* NewController = GetWorld()->SpawnActor<AAIController>(
        BotControllerClass, 
        FVector::ZeroVector, 
        FRotator::ZeroRotator, 
        SpawnInfo
    );

    if (NewController != nullptr)
    {
        ALyraGameMode* GameMode = GetGameMode<ALyraGameMode>();
        check(GameMode);

        // 设置 Bot 名称
        if (NewController->PlayerState != nullptr)
        {
            NewController->PlayerState->SetPlayerName(
                CreateBotName(NewController->PlayerState->GetPlayerId())
            );
        }

        // 通过 GameMode 初始化 Bot
        GameMode->GenericPlayerInitialization(NewController);
        GameMode->RestartPlayer(NewController);

        // 初始化 Pawn Extension 组件
        if (NewController->GetPawn() != nullptr)
        {
            if (ULyraPawnExtensionComponent* PawnExtComponent = 
                NewController->GetPawn()->FindComponentByClass<ULyraPawnExtensionComponent>())
            {
                PawnExtComponent->CheckDefaultInitialization();
            }
        }

        SpawnedBotList.Add(NewController);
    }
}

void ULyraBotCreationComponent::RemoveOneBot()
{
    if (SpawnedBotList.Num() > 0)
    {
        const int32 BotToRemoveIndex = FMath::RandRange(0, SpawnedBotList.Num() - 1);
        AAIController* BotToRemove = SpawnedBotList[BotToRemoveIndex];
        SpawnedBotList.RemoveAtSwap(BotToRemoveIndex);

        if (BotToRemove)
        {
            // 优先使用 HealthComponent 触发死亡动画
            if (APawn* ControlledPawn = BotToRemove->GetPawn())
            {
                if (ULyraHealthComponent* HealthComponent = 
                    ULyraHealthComponent::FindHealthComponent(ControlledPawn))
                {
                    HealthComponent->DamageSelfDestruct();
                }
                else
                {
                    ControlledPawn->Destroy();
                }
            }

            BotToRemove->Destroy();
        }
    }
}
```

**关键设计点：**

1. **Experience 集成**：在 Experience 加载完成后创建 Bot
2. **标准化流程**：使用 GameMode 的 `GenericPlayerInitialization` 和 `RestartPlayer`
3. **优雅销毁**：通过 HealthComponent 触发死亡逻辑，而非直接销毁
4. **配置灵活性**：支持通过 URL 参数和开发者设置覆盖 Bot 数量

---

## 2. AIController 和 Perception System

### 2.1 AI Perception 配置

UE5 的 AI Perception System 提供了视觉、听觉、触觉等多种感知方式。在 Lyra 中，我们需要配置适合射击游戏的感知系统：

```cpp
// LyraShooterBotController.h
UCLASS()
class ALyraShooterBotController : public ALyraPlayerBotController
{
    GENERATED_BODY()

public:
    ALyraShooterBotController(const FObjectInitializer& ObjectInitializer);

protected:
    virtual void BeginPlay() override;
    virtual void OnPossess(APawn* InPawn) override;

    // 感知组件
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
    TObjectPtr<UAIPerceptionComponent> PerceptionComponent;

    // 视觉配置
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
    TObjectPtr<UAISenseConfig_Sight> SightConfig;

    // 听觉配置
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
    TObjectPtr<UAISenseConfig_Hearing> HearingConfig;

    // 伤害感知配置
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
    TObjectPtr<UAISenseConfig_Damage> DamageConfig;

private:
    // 感知事件回调
    UFUNCTION()
    void OnTargetPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus);

    UFUNCTION()
    void OnTargetPerceptionForgotten(AActor* Actor);

    // 当前目标管理
    UPROPERTY()
    TObjectPtr<AActor> CurrentTarget;

    UPROPERTY()
    TMap<TObjectPtr<AActor>, FAIStimulus> PerceivedActors;
};
```

**实现感知系统：**

```cpp
// LyraShooterBotController.cpp
ALyraShooterBotController::ALyraShooterBotController(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 创建感知组件
    PerceptionComponent = CreateDefaultSubobject<UAIPerceptionComponent>(TEXT("PerceptionComponent"));
    SetPerceptionComponent(*PerceptionComponent);

    // 配置视觉感知
    SightConfig = CreateDefaultSubobject<UAISenseConfig_Sight>(TEXT("SightConfig"));
    SightConfig->SightRadius = 3000.0f;           // 视觉半径
    SightConfig->LoseSightRadius = 3500.0f;       // 失去视线半径
    SightConfig->PeripheralVisionAngleDegrees = 90.0f; // 周边视角
    SightConfig->SetMaxAge(5.0f);                 // 记忆时间
    SightConfig->AutoSuccessRangeFromLastSeenLocation = 500.0f;
    
    // 配置检测规则（基于团队）
    SightConfig->DetectionByAffiliation.bDetectEnemies = true;
    SightConfig->DetectionByAffiliation.bDetectNeutrals = true;
    SightConfig->DetectionByAffiliation.bDetectFriendlies = false;
    
    PerceptionComponent->ConfigureSense(*SightConfig);
    PerceptionComponent->SetDominantSense(SightConfig->GetSenseImplementation());

    // 配置听觉感知
    HearingConfig = CreateDefaultSubobject<UAISenseConfig_Hearing>(TEXT("HearingConfig"));
    HearingConfig->HearingRange = 2000.0f;        // 听觉半径
    HearingConfig->SetMaxAge(3.0f);
    HearingConfig->DetectionByAffiliation.bDetectEnemies = true;
    HearingConfig->DetectionByAffiliation.bDetectNeutrals = true;
    HearingConfig->DetectionByAffiliation.bDetectFriendlies = false;
    
    PerceptionComponent->ConfigureSense(*HearingConfig);

    // 配置伤害感知
    DamageConfig = CreateDefaultSubobject<UAISenseConfig_Damage>(TEXT("DamageConfig"));
    DamageConfig->SetMaxAge(5.0f);
    
    PerceptionComponent->ConfigureSense(*DamageConfig);

    // 绑定感知事件
    PerceptionComponent->OnTargetPerceptionUpdated.AddDynamic(
        this, &ALyraShooterBotController::OnTargetPerceptionUpdated
    );
    PerceptionComponent->OnTargetPerceptionForgotten.AddDynamic(
        this, &ALyraShooterBotController::OnTargetPerceptionForgotten
    );
}

void ALyraShooterBotController::BeginPlay()
{
    Super::BeginPlay();

    // 更新团队态度（影响感知过滤）
    UpdateTeamAttitude(PerceptionComponent);
}

void ALyraShooterBotController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);

    // Possess 时重新更新团队态度
    UpdateTeamAttitude(PerceptionComponent);
}

void ALyraShooterBotController::OnTargetPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus)
{
    if (!Actor) return;

    // 存储感知信息
    PerceivedActors.Add(Actor, Stimulus);

    // 检查是否是有效敌人
    if (GetTeamAttitudeTowards(*Actor) == ETeamAttitude::Hostile)
    {
        if (Stimulus.WasSuccessfullySensed())
        {
            // 更新当前目标（选择最近的敌人）
            if (!CurrentTarget || 
                (Actor->GetDistanceTo(GetPawn()) < CurrentTarget->GetDistanceTo(GetPawn())))
            {
                CurrentTarget = Actor;
                
                // 通知 Behavior Tree 或其他系统
                OnNewTargetDetected(Actor, Stimulus);
            }
        }
    }
}

void ALyraShooterBotController::OnTargetPerceptionForgotten(AActor* Actor)
{
    PerceivedActors.Remove(Actor);

    // 如果当前目标被遗忘，切换到其他目标
    if (CurrentTarget == Actor)
    {
        CurrentTarget = FindNextBestTarget();
        OnTargetLost(Actor);
    }
}

AActor* ALyraShooterBotController::FindNextBestTarget() const
{
    AActor* BestTarget = nullptr;
    float BestDistance = MAX_FLT;

    for (const auto& Pair : PerceivedActors)
    {
        AActor* PerceivedActor = Pair.Key;
        const FAIStimulus& Stimulus = Pair.Value;

        // 只考虑成功感知且为敌人的目标
        if (Stimulus.WasSuccessfullySensed() && 
            GetTeamAttitudeTowards(*PerceivedActor) == ETeamAttitude::Hostile)
        {
            float Distance = PerceivedActor->GetDistanceTo(GetPawn());
            if (Distance < BestDistance)
            {
                BestDistance = Distance;
                BestTarget = PerceivedActor;
            }
        }
    }

    return BestTarget;
}
```

### 2.2 团队感知过滤

Lyra 的团队系统与 AI Perception 深度集成，通过 `GetTeamAttitudeTowards` 实现智能过滤：

```cpp
// LyraPlayerBotController.cpp
ETeamAttitude::Type ALyraPlayerBotController::GetTeamAttitudeTowards(const AActor& Other) const
{
    if (const APawn* OtherPawn = Cast<APawn>(&Other))
    {
        if (const ILyraTeamAgentInterface* TeamAgent = 
            Cast<ILyraTeamAgentInterface>(OtherPawn->GetController()))
        {
            FGenericTeamId OtherTeamID = TeamAgent->GetGenericTeamId();

            // 判定态度
            if (OtherTeamID.GetId() != GetGenericTeamId().GetId())
            {
                return ETeamAttitude::Hostile;  // 敌对
            }
            else
            {
                return ETeamAttitude::Friendly; // 友好
            }
        }
    }

    return ETeamAttitude::Neutral; // 中立
}

void ALyraPlayerBotController::UpdateTeamAttitude(UAIPerceptionComponent* AIPerception)
{
    if (AIPerception)
    {
        // 请求刷新感知刺激监听器（会重新应用团队过滤）
        AIPerception->RequestStimuliListenerUpdate();
    }
}
```

### 2.3 噪音系统集成

在射击游戏中，枪声是重要的感知信号。我们需要在武器开火时报告噪音：

```cpp
// LyraWeaponInstance.cpp（示例扩展）
void ULyraWeaponInstance::OnFire()
{
    // ... 原有逻辑 ...

    // 报告噪音给 AI 系统
    if (APawn* OwnerPawn = GetPawn())
    {
        UAISense_Hearing::ReportNoiseEvent(
            GetWorld(),
            OwnerPawn->GetActorLocation(),
            1.0f,                    // 音量（0-1）
            OwnerPawn,               // 噪音制造者
            1500.0f,                 // 噪音范围
            TEXT("WeaponFire")       // 噪音标签
        );
    }
}
```

---

## 3. Behavior Tree 设计模式

虽然 Lyra 默认实现较为简单，但我们可以基于其架构设计可复用的行为树系统。

### 3.1 射击游戏 Bot 行为树结构

```
Root (Selector)
├── [Decorator: Is Dead?] Return Failure
├── Sequence: Combat
│   ├── [Decorator: Has Target?]
│   ├── Service: Update Target Location
│   ├── Selector: Combat Actions
│   │   ├── Sequence: Close Range Combat
│   │   │   ├── [Decorator: Distance < 500]
│   │   │   ├── Task: Strafe Around Target
│   │   │   └── Task: Fire Weapon
│   │   └── Sequence: Long Range Combat
│   │       ├── Task: Move To Cover
│   │       ├── Task: Aim At Target
│   │       └── Task: Fire Weapon
├── Sequence: Search
│   ├── [Decorator: Has Last Known Location?]
│   ├── Task: Move To Last Known Location
│   └── Task: Look Around
└── Sequence: Patrol
    ├── Task: Get Random Patrol Point
    ├── Task: Move To Patrol Point
    └── Task: Wait (1-3s)
```

### 3.2 核心 Blackboard Keys

```cpp
// LyraAIBlackboardKeys.h
namespace LyraAI
{
    namespace Blackboard
    {
        // 目标相关
        const FName CurrentTarget = TEXT("CurrentTarget");
        const FName LastKnownTargetLocation = TEXT("LastKnownTargetLocation");
        const FName LastSeenTargetTime = TEXT("LastSeenTargetTime");
        
        // 导航相关
        const FName PatrolPoint = TEXT("PatrolPoint");
        const FName CoverLocation = TEXT("CoverLocation");
        const FName FlankingPosition = TEXT("FlankingPosition");
        
        // 状态相关
        const FName IsInCombat = TEXT("IsInCombat");
        const FName IsReloading = TEXT("IsReloading");
        const FName CurrentWeapon = TEXT("CurrentWeapon");
        
        // 团队协作
        const FName SquadLeader = TEXT("SquadLeader");
        const FName SquadMembers = TEXT("SquadMembers");
        const FName AssignedObjective = TEXT("AssignedObjective");
    }
}
```

### 3.3 自定义 BT Task - Fire Weapon

```cpp
// BTTask_FireWeapon.h
UCLASS()
class UBTTask_FireWeapon : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UBTTask_FireWeapon();

    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
                                             uint8* NodeMemory) override;
    virtual void TickTask(UBehaviorTreeComponent& OwnerComp, 
                          uint8* NodeMemory, 
                          float DeltaSeconds) override;

protected:
    // 目标 Blackboard Key
    UPROPERTY(EditAnywhere, Category = "AI")
    FBlackboardKeySelector TargetActorKey;

    // 射击持续时间
    UPROPERTY(EditAnywhere, Category = "AI")
    float FireDuration = 1.5f;

    // 是否需要瞄准
    UPROPERTY(EditAnywhere, Category = "AI")
    bool bRequireAim = true;

    // 精度（0-1，影响射击偏移）
    UPROPERTY(EditAnywhere, Category = "AI", meta = (ClampMin = "0", ClampMax = "1"))
    float AccuracyModifier = 0.8f;
};

// BTTask_FireWeapon.cpp
UBTTask_FireWeapon::UBTTask_FireWeapon()
{
    NodeName = "Fire Weapon";
    bNotifyTick = true;
}

EBTNodeResult::Type UBTTask_FireWeapon::ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
                                                      uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return EBTNodeResult::Failed;

    APawn* AIPawn = AIController->GetPawn();
    if (!AIPawn) return EBTNodeResult::Failed;

    // 获取目标
    AActor* TargetActor = Cast<AActor>(
        OwnerComp.GetBlackboardComponent()->GetValueAsObject(TargetActorKey.SelectedKeyName)
    );
    if (!TargetActor) return EBTNodeResult::Failed;

    // 获取武器实例
    ULyraEquipmentManagerComponent* EquipmentManager = 
        AIPawn->FindComponentByClass<ULyraEquipmentManagerComponent>();
    if (!EquipmentManager) return EBTNodeResult::Failed;

    ULyraWeaponInstance* WeaponInstance = 
        EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>();
    if (!WeaponInstance) return EBTNodeResult::Failed;

    // 开始瞄准和射击
    if (bRequireAim)
    {
        // 设置瞄准方向
        FVector AimDirection = (TargetActor->GetActorLocation() - 
                                AIPawn->GetActorLocation()).GetSafeNormal();
        
        // 添加精度偏移
        if (AccuracyModifier < 1.0f)
        {
            float SpreadAngle = (1.0f - AccuracyModifier) * 15.0f; // 最大15度偏移
            FRotator SpreadRotation(
                FMath::RandRange(-SpreadAngle, SpreadAngle),
                FMath::RandRange(-SpreadAngle, SpreadAngle),
                0.0f
            );
            AimDirection = SpreadRotation.RotateVector(AimDirection);
        }

        AIController->SetFocalPoint(AIPawn->GetActorLocation() + AimDirection * 1000.0f);
    }

    // 开始射击（通过 GAS 能力）
    if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(AIPawn))
    {
        // 激活射击能力
        FGameplayAbilitySpec* AbilitySpec = ASC->FindAbilitySpecFromClass(
            ULyraGameplayAbility_RangedWeapon::StaticClass()
        );
        if (AbilitySpec)
        {
            ASC->TryActivateAbility(AbilitySpec->Handle);
        }
    }

    // 继续 Tick 直到射击完成
    return EBTNodeResult::InProgress;
}

void UBTTask_FireWeapon::TickTask(UBehaviorTreeComponent& OwnerComp, 
                                    uint8* NodeMemory, 
                                    float DeltaSeconds)
{
    // 简化示例：固定时间后结束
    // 实际应该检查弹药、目标状态等
    
    float* ElapsedTime = (float*)NodeMemory;
    *ElapsedTime += DeltaSeconds;

    if (*ElapsedTime >= FireDuration)
    {
        // 停止射击
        AAIController* AIController = OwnerComp.GetAIOwner();
        if (APawn* AIPawn = AIController->GetPawn())
        {
            if (UAbilitySystemComponent* ASC = 
                UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(AIPawn))
            {
                // 取消射击能力
                ASC->CancelAbilities();
            }
        }

        FinishLatentTask(OwnerComp, EBTNodeResult::Succeeded);
    }
}
```

### 3.4 自定义 BT Service - Update Combat State

```cpp
// BTService_UpdateCombatState.h
UCLASS()
class UBTService_UpdateCombatState : public UBTService
{
    GENERATED_BODY()

public:
    UBTService_UpdateCombatState();

protected:
    virtual void TickNode(UBehaviorTreeComponent& OwnerComp, 
                          uint8* NodeMemory, 
                          float DeltaSeconds) override;

    UPROPERTY(EditAnywhere, Category = "AI")
    FBlackboardKeySelector CurrentTargetKey;

    UPROPERTY(EditAnywhere, Category = "AI")
    FBlackboardKeySelector LastKnownLocationKey;

    UPROPERTY(EditAnywhere, Category = "AI")
    FBlackboardKeySelector IsInCombatKey;
};

// BTService_UpdateCombatState.cpp
void UBTService_UpdateCombatState::TickNode(UBehaviorTreeComponent& OwnerComp, 
                                             uint8* NodeMemory, 
                                             float DeltaSeconds)
{
    Super::TickNode(OwnerComp, NodeMemory, DeltaSeconds);

    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return;

    ALyraShooterBotController* BotController = Cast<ALyraShooterBotController>(AIController);
    if (!BotController) return;

    UBlackboardComponent* BlackboardComp = OwnerComp.GetBlackboardComponent();
    if (!BlackboardComp) return;

    // 更新当前目标
    AActor* CurrentTarget = BotController->GetCurrentTarget();
    BlackboardComp->SetValueAsObject(CurrentTargetKey.SelectedKeyName, CurrentTarget);

    // 更新战斗状态
    bool bIsInCombat = (CurrentTarget != nullptr);
    BlackboardComp->SetValueAsBool(IsInCombatKey.SelectedKeyName, bIsInCombat);

    // 更新最后已知位置
    if (CurrentTarget)
    {
        // 检查是否能看到目标
        if (BotController->LineOfSightTo(CurrentTarget))
        {
            BlackboardComp->SetValueAsVector(
                LastKnownLocationKey.SelectedKeyName,
                CurrentTarget->GetActorLocation()
            );
        }
    }
}
```

### 3.5 自定义 BT Decorator - Check Ammo

```cpp
// BTDecorator_CheckAmmo.h
UCLASS()
class UBTDecorator_CheckAmmo : public UBTDecorator
{
    GENERATED_BODY()

public:
    UBTDecorator_CheckAmmo();

protected:
    virtual bool CalculateRawConditionValue(UBehaviorTreeComponent& OwnerComp, 
                                             uint8* NodeMemory) const override;

    // 最小弹药百分比（0-1）
    UPROPERTY(EditAnywhere, Category = "AI")
    float MinAmmoPercentage = 0.3f;

    // 检查的是弹匣还是总弹药
    UPROPERTY(EditAnywhere, Category = "AI")
    bool bCheckMagazine = true;
};

// BTDecorator_CheckAmmo.cpp
bool UBTDecorator_CheckAmmo::CalculateRawConditionValue(UBehaviorTreeComponent& OwnerComp, 
                                                         uint8* NodeMemory) const
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return false;

    APawn* AIPawn = AIController->GetPawn();
    if (!AIPawn) return false;

    // 获取装备管理器
    ULyraEquipmentManagerComponent* EquipmentManager = 
        AIPawn->FindComponentByClass<ULyraEquipmentManagerComponent>();
    if (!EquipmentManager) return false;

    // 获取当前武器
    ULyraWeaponInstance* WeaponInstance = 
        EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>();
    if (!WeaponInstance) return false;

    // 检查弹药
    int32 CurrentAmmo = bCheckMagazine ? 
        WeaponInstance->GetCurrentAmmoInMagazine() : 
        WeaponInstance->GetTotalReserveAmmo();
    
    int32 MaxAmmo = bCheckMagazine ? 
        WeaponInstance->GetMaxAmmoInMagazine() : 
        WeaponInstance->GetMaxReserveAmmo();

    float AmmoPercentage = (MaxAmmo > 0) ? 
        (float)CurrentAmmo / (float)MaxAmmo : 0.0f;

    return AmmoPercentage >= MinAmmoPercentage;
}
```

---

## 4. 与 GAS 集成的 AI 技能系统

### 4.1 AI 使用 Gameplay Abilities

Lyra 的 AI 与玩家共享相同的 Gameplay Ability 系统，这带来了巨大优势：

```cpp
// LyraAIAbilitySet.h
UCLASS()
class ULyraAIAbilitySet : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // AI 可用的能力列表
    UPROPERTY(EditDefaultsOnly, Category = "Abilities")
    TArray<FLyraAbilitySet_GameplayAbility> Abilities;

    // AI 可用的效果
    UPROPERTY(EditDefaultsOnly, Category = "Abilities")
    TArray<FLyraAbilitySet_GameplayEffect> Effects;

    // AI 可用的属性集
    UPROPERTY(EditDefaultsOnly, Category = "Abilities")
    TArray<FLyraAbilitySet_AttributeSet> Attributes;

    // 授予能力给 ASC
    void GiveToAbilitySystem(UAbilitySystemComponent* ASC, 
                             UObject* SourceObject = nullptr) const;
};
```

### 4.2 AI 能力激活策略

```cpp
// LyraAIAbilityComponent.h
UCLASS()
class ULyraAIAbilityComponent : public UPawnComponent
{
    GENERATED_BODY()

public:
    ULyraAIAbilityComponent();

    // 初始化 AI 能力
    void InitializeAIAbilities(UAbilitySystemComponent* ASC);

    // 尝试激活最佳能力
    bool TryActivateBestAbility(const FGameplayTag& AbilityCategory, 
                                 AActor* Target = nullptr);

    // 检查是否可以激活能力
    bool CanActivateAbility(TSubclassOf<UGameplayAbility> AbilityClass, 
                            AActor* Target = nullptr) const;

protected:
    // AI 能力集
    UPROPERTY(EditDefaultsOnly, Category = "AI")
    TObjectPtr<ULyraAIAbilitySet> AIAbilitySet;

    // 能力优先级映射
    UPROPERTY(EditDefaultsOnly, Category = "AI")
    TMap<FGameplayTag, int32> AbilityPriorities;

    // ASC 引用
    UPROPERTY()
    TObjectPtr<UAbilitySystemComponent> CachedASC;

private:
    // 能力使用统计（用于决策）
    UPROPERTY()
    TMap<FGameplayAbilitySpecHandle, float> LastUsedTimes;
};

// LyraAIAbilityComponent.cpp
bool ULyraAIAbilityComponent::TryActivateBestAbility(const FGameplayTag& AbilityCategory, 
                                                      AActor* Target)
{
    if (!CachedASC) return false;

    // 获取该类别下的所有能力
    TArray<FGameplayAbilitySpec*> ActivatableAbilities;
    CachedASC->GetActivatableGameplayAbilitySpecsByAllMatchingTags(
        FGameplayTagContainer(AbilityCategory),
        ActivatableAbilities
    );

    // 根据优先级和冷却时间排序
    ActivatableAbilities.Sort([this](const FGameplayAbilitySpec& A, const FGameplayAbilitySpec& B)
    {
        int32 PriorityA = AbilityPriorities.FindRef(A.Ability->AbilityTags.First());
        int32 PriorityB = AbilityPriorities.FindRef(B.Ability->AbilityTags.First());
        return PriorityA > PriorityB;
    });

    // 尝试激活第一个可用的能力
    for (FGameplayAbilitySpec* Spec : ActivatableAbilities)
    {
        if (CachedASC->TryActivateAbility(Spec->Handle))
        {
            LastUsedTimes.Add(Spec->Handle, GetWorld()->GetTimeSeconds());
            return true;
        }
    }

    return false;
}

bool ULyraAIAbilityComponent::CanActivateAbility(TSubclassOf<UGameplayAbility> AbilityClass, 
                                                  AActor* Target) const
{
    if (!CachedASC || !AbilityClass) return false;

    // 查找能力 Spec
    FGameplayAbilitySpec* Spec = CachedASC->FindAbilitySpecFromClass(AbilityClass);
    if (!Spec) return false;

    // 检查是否在冷却中
    if (CachedASC->GetCooldownTimeRemaining(Spec->Handle) > 0.0f)
    {
        return false;
    }

    // 检查是否满足激活条件
    return CachedASC->CanActivateAbility(Spec->Handle);
}
```

### 4.3 BT Task - Activate Ability

```cpp
// BTTask_ActivateAbility.h
UCLASS()
class UBTTask_ActivateAbility : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UBTTask_ActivateAbility();

    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
                                             uint8* NodeMemory) override;

protected:
    // 要激活的能力类
    UPROPERTY(EditAnywhere, Category = "AI")
    TSubclassOf<UGameplayAbility> AbilityToActivate;

    // 或者使用能力标签
    UPROPERTY(EditAnywhere, Category = "AI")
    FGameplayTag AbilityTag;

    // 目标 Actor（用于 Targeting）
    UPROPERTY(EditAnywhere, Category = "AI")
    FBlackboardKeySelector TargetKey;

    // 是否等待能力结束
    UPROPERTY(EditAnywhere, Category = "AI")
    bool bWaitForCompletion = false;
};

// BTTask_ActivateAbility.cpp
EBTNodeResult::Type UBTTask_ActivateAbility::ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
                                                          uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return EBTNodeResult::Failed;

    APawn* AIPawn = AIController->GetPawn();
    if (!AIPawn) return EBTNodeResult::Failed;

    // 获取 ASC
    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(AIPawn);
    if (!ASC) return EBTNodeResult::Failed;

    // 获取目标（如果有）
    AActor* TargetActor = nullptr;
    if (TargetKey.IsSet())
    {
        TargetActor = Cast<AActor>(
            OwnerComp.GetBlackboardComponent()->GetValueAsObject(TargetKey.SelectedKeyName)
        );
    }

    // 构建 Event Data（包含目标信息）
    FGameplayEventData EventData;
    EventData.Instigator = AIPawn;
    EventData.Target = TargetActor;

    bool bSuccess = false;

    // 通过类激活
    if (AbilityToActivate)
    {
        FGameplayAbilitySpec* Spec = ASC->FindAbilitySpecFromClass(AbilityToActivate);
        if (Spec)
        {
            bSuccess = ASC->TryActivateAbility(Spec->Handle, true);
        }
    }
    // 通过标签激活
    else if (AbilityTag.IsValid())
    {
        bSuccess = ASC->TryActivateAbilitiesByTag(FGameplayTagContainer(AbilityTag));
    }

    if (!bSuccess)
    {
        return EBTNodeResult::Failed;
    }

    // 如果需要等待完成，需要监听能力结束事件
    if (bWaitForCompletion)
    {
        // 这里需要更复杂的逻辑，监听 OnAbilityEnded 委托
        // 简化示例，直接返回成功
        return EBTNodeResult::Succeeded;
    }

    return EBTNodeResult::Succeeded;
}
```

---

## 5. 导航和寻路优化

### 5.1 动态导航网格

在射击游戏中，掩体和动态障碍物是关键。确保正确配置 Nav Mesh：

```cpp
// LyraAINavigationConfig.h
UCLASS()
class ULyraAINavigationConfig : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    // Nav Mesh 配置
    UPROPERTY(EditAnywhere, Config, Category = "Navigation")
    float AgentRadius = 34.0f;

    UPROPERTY(EditAnywhere, Config, Category = "Navigation")
    float AgentHeight = 144.0f;

    UPROPERTY(EditAnywhere, Config, Category = "Navigation")
    float StepHeight = 45.0f;

    // 动态导航更新
    UPROPERTY(EditAnywhere, Config, Category = "Navigation")
    bool bEnableDynamicNavMeshUpdates = true;

    // 路径优化
    UPROPERTY(EditAnywhere, Config, Category = "Pathfinding")
    bool bUsePathSmoothing = true;

    UPROPERTY(EditAnywhere, Config, Category = "Pathfinding")
    float PathSmoothingRadius = 50.0f;
};
```

### 5.2 掩体寻找算法

```cpp
// LyraAICoverSystem.h
UCLASS()
class ULyraAICoverSystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    // 查找最佳掩体位置
    FVector FindBestCoverLocation(const FVector& FromLocation, 
                                   const FVector& ThreatLocation, 
                                   float SearchRadius = 1000.0f) const;

    // 评估掩体质量
    float EvaluateCoverQuality(const FVector& CoverLocation, 
                                const FVector& ThreatLocation) const;

    // 检查位置是否有掩体
    bool HasCover(const FVector& Location, 
                  const FVector& ThreatDirection) const;

private:
    // 掩体评分参数
    UPROPERTY(Config)
    float DistanceWeight = 0.3f;

    UPROPERTY(Config)
    float ProtectionWeight = 0.5f;

    UPROPERTY(Config)
    float VisibilityWeight = 0.2f;
};

// LyraAICoverSystem.cpp
FVector ULyraAICoverSystem::FindBestCoverLocation(const FVector& FromLocation, 
                                                   const FVector& ThreatLocation, 
                                                   float SearchRadius) const
{
    UNavigationSystemV1* NavSys = FNavigationSystem::GetCurrent<UNavigationSystemV1>(GetWorld());
    if (!NavSys) return FVector::ZeroVector;

    // 在导航网格上采样点
    TArray<FNavLocation> SamplePoints;
    NavSys->GetRandomPointsInNavigableRadius(FromLocation, SearchRadius, 32, SamplePoints);

    FVector BestCoverLocation = FVector::ZeroVector;
    float BestScore = -MAX_FLT;

    for (const FNavLocation& SamplePoint : SamplePoints)
    {
        float Score = EvaluateCoverQuality(SamplePoint.Location, ThreatLocation);
        
        if (Score > BestScore)
        {
            BestScore = Score;
            BestCoverLocation = SamplePoint.Location;
        }
    }

    return BestCoverLocation;
}

float ULyraAICoverSystem::EvaluateCoverQuality(const FVector& CoverLocation, 
                                                const FVector& ThreatLocation) const
{
    float TotalScore = 0.0f;

    // 1. 距离评分（距离威胁适中）
    float Distance = FVector::Dist(CoverLocation, ThreatLocation);
    float IdealDistance = 1000.0f; // 理想距离
    float DistanceScore = 1.0f - FMath::Abs(Distance - IdealDistance) / IdealDistance;
    DistanceScore = FMath::Clamp(DistanceScore, 0.0f, 1.0f);
    TotalScore += DistanceScore * DistanceWeight;

    // 2. 保护评分（是否能阻挡射线）
    FVector ThreatDirection = (CoverLocation - ThreatLocation).GetSafeNormal();
    float ProtectionScore = HasCover(CoverLocation, ThreatDirection) ? 1.0f : 0.0f;
    TotalScore += ProtectionScore * ProtectionWeight;

    // 3. 可见性评分（能否看到威胁）
    FHitResult HitResult;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(nullptr);
    
    bool bCanSeeTheat = !GetWorld()->LineTraceSingleByChannel(
        HitResult,
        CoverLocation + FVector(0, 0, 50), // 稍微抬高
        ThreatLocation,
        ECC_Visibility,
        QueryParams
    );
    
    float VisibilityScore = bCanSeeTheat ? 1.0f : 0.5f;
    TotalScore += VisibilityScore * VisibilityWeight;

    return TotalScore;
}

bool ULyraAICoverSystem::HasCover(const FVector& Location, 
                                   const FVector& ThreatDirection) const
{
    // 从多个高度检查是否有遮挡
    TArray<float> CheckHeights = {50.0f, 100.0f, 150.0f};
    
    for (float Height : CheckHeights)
    {
        FVector StartLocation = Location + FVector(0, 0, Height);
        FVector EndLocation = StartLocation + ThreatDirection * 100.0f;

        FHitResult HitResult;
        FCollisionQueryParams QueryParams;
        
        if (GetWorld()->LineTraceSingleByChannel(
            HitResult,
            StartLocation,
            EndLocation,
            ECC_WorldStatic,
            QueryParams))
        {
            // 找到了遮挡物
            return true;
        }
    }

    return false;
}
```

### 5.3 BT Task - Move To Cover

```cpp
// BTTask_MoveToCover.h
UCLASS()
class UBTTask_MoveToCover : public UBTTask_MoveTo
{
    GENERATED_BODY()

public:
    UBTTask_MoveToCover();

    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
                                             uint8* NodeMemory) override;

protected:
    // 威胁目标 Key
    UPROPERTY(EditAnywhere, Category = "AI")
    FBlackboardKeySelector ThreatActorKey;

    // 掩体搜索半径
    UPROPERTY(EditAnywhere, Category = "AI")
    float CoverSearchRadius = 1500.0f;

    // 掩体位置 Key（输出）
    UPROPERTY(EditAnywhere, Category = "AI")
    FBlackboardKeySelector CoverLocationKey;
};

// BTTask_MoveToCover.cpp
EBTNodeResult::Type UBTTask_MoveToCover::ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
                                                      uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return EBTNodeResult::Failed;

    APawn* AIPawn = AIController->GetPawn();
    if (!AIPawn) return EBTNodeResult::Failed;

    // 获取威胁目标
    AActor* ThreatActor = Cast<AActor>(
        OwnerComp.GetBlackboardComponent()->GetValueAsObject(ThreatActorKey.SelectedKeyName)
    );
    if (!ThreatActor) return EBTNodeResult::Failed;

    // 获取掩体系统
    ULyraAICoverSystem* CoverSystem = 
        GetWorld()->GetSubsystem<ULyraAICoverSystem>();
    if (!CoverSystem) return EBTNodeResult::Failed;

    // 寻找最佳掩体
    FVector CoverLocation = CoverSystem->FindBestCoverLocation(
        AIPawn->GetActorLocation(),
        ThreatActor->GetActorLocation(),
        CoverSearchRadius
    );

    if (CoverLocation.IsZero())
    {
        return EBTNodeResult::Failed;
    }

    // 保存掩体位置到 Blackboard
    OwnerComp.GetBlackboardComponent()->SetValueAsVector(
        CoverLocationKey.SelectedKeyName,
        CoverLocation
    );

    // 设置移动目标为掩体位置
    BlackboardKey.SelectedKeyName = CoverLocationKey.SelectedKeyName;

    // 调用父类的移动逻辑
    return Super::ExecuteTask(OwnerComp, NodeMemory);
}
```

---

## 6. AI 决策系统

### 6.1 Utility AI 基础

Utility AI 通过评分函数选择最佳行动，比硬编码的状态机更灵活：

```cpp
// LyraAIUtilitySystem.h
USTRUCT(BlueprintType)
struct FLyraAIUtilityAction
{
    GENERATED_BODY()

    // 行动名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FName ActionName;

    // 行动权重曲线
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    UCurveFloat* UtilityCurve;

    // 输入参数（如：距离、血量等）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TMap<FName, float> InputParameters;

    // 冷却时间
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float CooldownTime = 0.0f;

    // 上次执行时间
    float LastExecutionTime = -999.0f;
};

UCLASS()
class ULyraAIUtilitySystem : public UActorComponent
{
    GENERATED_BODY()

public:
    ULyraAIUtilitySystem();

    // 评估所有行动，返回最佳行动
    FName EvaluateBestAction(const TMap<FName, float>& WorldState);

    // 注册行动
    void RegisterAction(const FLyraAIUtilityAction& Action);

    // 更新冷却时间
    void UpdateCooldowns(float DeltaTime);

protected:
    UPROPERTY(EditAnywhere, Category = "AI")
    TArray<FLyraAIUtilityAction> AvailableActions;

private:
    float EvaluateActionUtility(const FLyraAIUtilityAction& Action, 
                                 const TMap<FName, float>& WorldState) const;
};

// LyraAIUtilitySystem.cpp
FName ULyraAIUtilitySystem::EvaluateBestAction(const TMap<FName, float>& WorldState)
{
    float BestScore = -MAX_FLT;
    FName BestAction = NAME_None;

    float CurrentTime = GetWorld()->GetTimeSeconds();

    for (const FLyraAIUtilityAction& Action : AvailableActions)
    {
        // 检查冷却
        if (CurrentTime - Action.LastExecutionTime < Action.CooldownTime)
        {
            continue;
        }

        // 评估效用值
        float Utility = EvaluateActionUtility(Action, WorldState);

        if (Utility > BestScore)
        {
            BestScore = Utility;
            BestAction = Action.ActionName;
        }
    }

    return BestAction;
}

float ULyraAIUtilitySystem::EvaluateActionUtility(const FLyraAIUtilityAction& Action, 
                                                   const TMap<FName, float>& WorldState) const
{
    if (!Action.UtilityCurve) return 0.0f;

    float TotalUtility = 0.0f;
    int32 ValidInputCount = 0;

    // 对每个输入参数评估效用
    for (const auto& Param : Action.InputParameters)
    {
        const float* WorldValue = WorldState.Find(Param.Key);
        if (WorldValue)
        {
            // 使用曲线计算效用值
            float NormalizedValue = *WorldValue / Param.Value; // Param.Value 作为归一化因子
            float Utility = Action.UtilityCurve->GetFloatValue(NormalizedValue);
            
            TotalUtility += Utility;
            ValidInputCount++;
        }
    }

    // 返回平均效用值
    return (ValidInputCount > 0) ? (TotalUtility / ValidInputCount) : 0.0f;
}
```

### 6.2 Utility AI 示例配置

在蓝图中配置 Utility Actions：

```cpp
// 示例：攻击行动
Action: "Attack"
UtilityCurve: (距离越近，效用越高)
  - 0m -> 1.0
  - 500m -> 0.8
  - 1000m -> 0.3
  - 2000m -> 0.1
InputParameters:
  - DistanceToTarget: 1000.0 (归一化因子)
  - TargetHealth: 100.0
CooldownTime: 1.5s

// 示例：撤退行动
Action: "Retreat"
UtilityCurve: (血量越低，效用越高)
  - 100% HP -> 0.0
  - 50% HP -> 0.3
  - 25% HP -> 0.7
  - 10% HP -> 1.0
InputParameters:
  - SelfHealth: 100.0
  - EnemyCount: 5.0
CooldownTime: 5.0s

// 示例：寻找掩体
Action: "SeekCover"
UtilityCurve: (组合血量和暴露度)
  - Low HP + Exposed -> 0.9
  - High HP + Exposed -> 0.4
  - Low HP + In Cover -> 0.1
InputParameters:
  - SelfHealth: 100.0
  - ExposureLevel: 1.0
CooldownTime: 3.0s
```

### 6.3 GOAP (Goal-Oriented Action Planning) 简介

GOAP 是更高级的 AI 决策系统，适合复杂策略：

```cpp
// LyraAIGOAPAction.h
USTRUCT()
struct FLyraGOAPAction
{
    GENERATED_BODY()

    // 行动名称
    FName ActionName;

    // 前置条件（世界状态）
    TMap<FName, bool> Preconditions;

    // 效果（改变的世界状态）
    TMap<FName, bool> Effects;

    // 成本
    float Cost;
};

// GOAP Planner (简化版)
class FLyraGOAPPlanner
{
public:
    // 规划行动序列
    TArray<FLyraGOAPAction> Plan(const TMap<FName, bool>& CurrentState,
                                  const TMap<FName, bool>& GoalState,
                                  const TArray<FLyraGOAPAction>& AvailableActions);

private:
    // A* 搜索
    struct FGOAPNode
    {
        TMap<FName, bool> State;
        TArray<FLyraGOAPAction> ActionPath;
        float Cost;
    };

    bool MeetsConditions(const TMap<FName, bool>& State, 
                         const TMap<FName, bool>& Conditions) const;
    
    TMap<FName, bool> ApplyEffects(const TMap<FName, bool>& State, 
                                     const TMap<FName, bool>& Effects) const;
};
```

**GOAP 示例场景：**

```
Goal: "EnemyEliminated"

Available Actions:
1. "ShootEnemy"
   Preconditions: { HasAmmo: true, CanSeeEnemy: true, InRange: true }
   Effects: { EnemyEliminated: true }
   Cost: 1.0

2. "ReloadWeapon"
   Preconditions: { HasReserveAmmo: true }
   Effects: { HasAmmo: true }
   Cost: 2.0

3. "MoveTo CloserPosition"
   Preconditions: {}
   Effects: { InRange: true }
   Cost: 3.0

4. "SearchForEnemy"
   Preconditions: {}
   Effects: { CanSeeEnemy: true }
   Cost: 5.0

Current State:
{ HasAmmo: false, CanSeeEnemy: true, InRange: true, HasReserveAmmo: true }

Planner Output:
Path: ReloadWeapon -> ShootEnemy
Total Cost: 3.0
```

---

## 7. 多人游戏中的 AI 同步

### 7.1 AI 网络复制策略

Lyra 的 AI 在多人游戏中**仅在服务器运行**，客户端通过网络复制同步：

```cpp
// LyraPlayerBotController.cpp (网络配置)
ALyraPlayerBotController::ALyraPlayerBotController(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    bWantsPlayerState = true;            // Bot 拥有 PlayerState
    bStopAILogicOnUnposses = false;      // Unpossess 时不停止 AI
    
    // 网络配置
    SetReplicates(true);                 // 复制 Controller
    bReplicateMovement = false;          // 不复制移动（由 Pawn 处理）
}
```

### 7.2 关键数据复制

只复制关键数据，避免带宽浪费：

```cpp
// LyraAIReplicatedData.h
UCLASS()
class ALyraAIReplicatedPawn : public ALyraPawn
{
    GENERATED_BODY()

protected:
    // 复制当前目标（用于客户端动画）
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "AI")
    TObjectPtr<AActor> ReplicatedCurrentTarget;

    // 复制 AI 状态（用于客户端表现）
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "AI")
    EAIState CurrentAIState;

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
};

// LyraAIReplicatedPawn.cpp
void ALyraAIReplicatedPawn::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION(ALyraAIReplicatedPawn, ReplicatedCurrentTarget, COND_SkipOwner);
    DOREPLIFETIME_CONDITION(ALyraAIReplicatedPawn, CurrentAIState, COND_SkipOwner);
}
```

### 7.3 客户端预测 vs 服务器权威

```cpp
// AI 使用服务器权威模型
void ALyraShooterBotController::FireWeapon()
{
    // 仅在服务器执行
    if (!HasAuthority()) return;

    // 激活射击能力（能力系统会处理网络复制）
    if (UAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        ASC->TryActivateAbilitiesByTag(FGameplayTagContainer(TAG_Ability_Fire));
    }
}
```

### 7.4 AI 与玩家的公平性

确保 AI 与玩家使用相同的游戏规则：

```cpp
// 使用相同的 GAS 能力
// 使用相同的武器实例
// 使用相同的伤害计算
// 使用相同的碰撞检测

// 示例：AI 射击延迟模拟人类反应时间
UPROPERTY(EditDefaultsOnly, Category = "AI|Difficulty")
float ReactionTimeMin = 0.2f;

UPROPERTY(EditDefaultsOnly, Category = "AI|Difficulty")
float ReactionTimeMax = 0.5f;

void ALyraShooterBotController::OnTargetDetected(AActor* Target)
{
    float ReactionDelay = FMath::RandRange(ReactionTimeMin, ReactionTimeMax);
    
    FTimerHandle ReactionTimer;
    GetWorldTimerManager().SetTimer(ReactionTimer, [this, Target]()
    {
        StartEngagingTarget(Target);
    }, ReactionDelay, false);
}
```

---

## 8. 实战案例 1：射击游戏 Bot（巡逻/搜索/战斗）

### 8.1 完整 Bot 实现

```cpp
// LyraShooterBotController.h
UCLASS()
class ALyraShooterBotController : public ALyraPlayerBotController
{
    GENERATED_BODY()

public:
    ALyraShooterBotController(const FObjectInitializer& ObjectInitializer);

    // AI 状态枚举
    UENUM(BlueprintType)
    enum class EAIState : uint8
    {
        Idle,
        Patrol,
        Investigate,
        Combat,
        Retreat,
        Dead
    };

protected:
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;
    virtual void OnPossess(APawn* InPawn) override;

    // AI 组件
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    TObjectPtr<UAIPerceptionComponent> PerceptionComponent;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    TObjectPtr<UBehaviorTree> BotBehaviorTree;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    TObjectPtr<UBlackboardComponent> BlackboardComponent;

    // AI 状态
    UPROPERTY(Replicated, BlueprintReadOnly)
    EAIState CurrentState;

    // 当前目标
    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<AActor> CurrentTarget;

    // 巡逻点
    UPROPERTY(EditInstanceOnly, Category = "AI|Patrol")
    TArray<TObjectPtr<AActor>> PatrolPoints;

    UPROPERTY()
    int32 CurrentPatrolIndex = 0;

    // 难度设置
    UPROPERTY(EditDefaultsOnly, Category = "AI|Difficulty")
    float AimAccuracy = 0.7f; // 0-1

    UPROPERTY(EditDefaultsOnly, Category = "AI|Difficulty")
    float ReactionTime = 0.3f; // 秒

    UPROPERTY(EditDefaultsOnly, Category = "AI|Difficulty")
    float AggressionLevel = 0.5f; // 0-1

private:
    // 状态机
    void UpdateAIState(float DeltaTime);
    void ExecutePatrolBehavior(float DeltaTime);
    void ExecuteInvestigateBehavior(float DeltaTime);
    void ExecuteCombatBehavior(float DeltaTime);
    void ExecuteRetreatBehavior(float DeltaTime);

    // 战斗逻辑
    void EngageTarget(AActor* Target);
    void UpdateAimTarget(float DeltaTime);
    bool ShouldRetreat() const;
    
    // 工具函数
    FVector GetNextPatrolPoint() const;
    FVector GetCoverPosition() const;
    bool IsInCover() const;

    // 感知回调
    UFUNCTION()
    void OnPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus);

    // 计时器
    FTimerHandle ReactionTimerHandle;
    FTimerHandle PatrolWaitHandle;
};

// LyraShooterBotController.cpp
void ALyraShooterBotController::UpdateAIState(float DeltaTime)
{
    // 检查死亡
    if (ULyraHealthComponent* HealthComp = ULyraHealthComponent::FindHealthComponent(GetPawn()))
    {
        if (HealthComp->IsDeadOrDying())
        {
            CurrentState = EAIState::Dead;
            return;
        }
    }

    switch (CurrentState)
    {
    case EAIState::Idle:
        // 空闲状态 -> 开始巡逻
        if (PatrolPoints.Num() > 0)
        {
            CurrentState = EAIState::Patrol;
        }
        break;

    case EAIState::Patrol:
        ExecutePatrolBehavior(DeltaTime);
        
        // 如果发现敌人 -> 战斗
        if (CurrentTarget && GetTeamAttitudeTowards(*CurrentTarget) == ETeamAttitude::Hostile)
        {
            CurrentState = EAIState::Combat;
            EngageTarget(CurrentTarget);
        }
        break;

    case EAIState::Investigate:
        ExecuteInvestigateBehavior(DeltaTime);
        
        // 发现敌人 -> 战斗
        if (CurrentTarget)
        {
            CurrentState = EAIState::Combat;
        }
        // 调查完毕 -> 巡逻
        else if (IsAtInvestigationLocation())
        {
            CurrentState = EAIState::Patrol;
        }
        break;

    case EAIState::Combat:
        ExecuteCombatBehavior(DeltaTime);
        
        // 血量过低 -> 撤退
        if (ShouldRetreat())
        {
            CurrentState = EAIState::Retreat;
        }
        // 目标丢失 -> 调查
        else if (!CurrentTarget || !LineOfSightTo(CurrentTarget))
        {
            CurrentState = EAIState::Investigate;
        }
        break;

    case EAIState::Retreat:
        ExecuteRetreatBehavior(DeltaTime);
        
        // 恢复后重新战斗
        if (IsInCover() && GetHealthPercentage() > 0.5f)
        {
            CurrentState = EAIState::Combat;
        }
        break;
    }
}

void ALyraShooterBotController::ExecuteCombatBehavior(float DeltaTime)
{
    if (!CurrentTarget) return;

    APawn* MyPawn = GetPawn();
    if (!MyPawn) return;

    float DistanceToTarget = FVector::Dist(MyPawn->GetActorLocation(), 
                                            CurrentTarget->GetActorLocation());

    // 1. 移动策略
    if (DistanceToTarget > 1500.0f)
    {
        // 远距离 -> 靠近
        MoveToActor(CurrentTarget, 800.0f);
    }
    else if (DistanceToTarget < 300.0f)
    {
        // 过近 -> 后退
        FVector AwayDirection = (MyPawn->GetActorLocation() - 
                                 CurrentTarget->GetActorLocation()).GetSafeNormal();
        FVector RetreatLocation = MyPawn->GetActorLocation() + AwayDirection * 500.0f;
        MoveToLocation(RetreatLocation);
    }
    else
    {
        // 中等距离 -> 扫射移动
        StrafeCombatMovement(DeltaTime);
    }

    // 2. 瞄准和射击
    UpdateAimTarget(DeltaTime);
    
    // 通过 GAS 激活射击能力
    if (UAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        if (LineOfSightTo(CurrentTarget))
        {
            // 持续射击
            if (!ASC->IsAbilityActive(FGameplayTag::RequestGameplayTag("Ability.Weapon.Fire")))
            {
                ASC->TryActivateAbilitiesByTag(
                    FGameplayTagContainer(FGameplayTag::RequestGameplayTag("Ability.Weapon.Fire"))
                );
            }
        }
        else
        {
            // 没视线 -> 停火
            ASC->CancelAbilities(&FGameplayTagContainer(
                FGameplayTag::RequestGameplayTag("Ability.Weapon.Fire")
            ));
        }
    }

    // 3. 定期切换位置
    if (FMath::RandRange(0.0f, 1.0f) < 0.02f * DeltaTime) // 2% 每秒
    {
        FVector CoverPos = GetCoverPosition();
        if (!CoverPos.IsZero())
        {
            MoveToLocation(CoverPos);
        }
    }
}

void ALyraShooterBotController::UpdateAimTarget(float DeltaTime)
{
    if (!CurrentTarget) return;

    APawn* MyPawn = GetPawn();
    if (!MyPawn) return;

    // 计算目标点（预测目标移动）
    FVector TargetVelocity = CurrentTarget->GetVelocity();
    float PredictionTime = 0.2f; // 200ms 预测
    FVector PredictedLocation = CurrentTarget->GetActorLocation() + 
                                 TargetVelocity * PredictionTime;

    // 添加精度偏移（模拟人类不完美瞄准）
    float SpreadAngle = (1.0f - AimAccuracy) * 10.0f; // 最大10度
    FVector AimDirection = (PredictedLocation - MyPawn->GetActorLocation()).GetSafeNormal();
    FRotator SpreadRotation(
        FMath::RandRange(-SpreadAngle, SpreadAngle),
        FMath::RandRange(-SpreadAngle, SpreadAngle),
        0.0f
    );
    AimDirection = SpreadRotation.RotateVector(AimDirection);

    // 设置瞄准点
    SetFocalPoint(MyPawn->GetActorLocation() + AimDirection * 2000.0f);
}

void ALyraShooterBotController::StrafeCombatMovement(float DeltaTime)
{
    APawn* MyPawn = GetPawn();
    if (!MyPawn) return;

    // 生成随机扫射方向
    static float StrafeTimer = 0.0f;
    static FVector StrafeDirection = FVector::ZeroVector;

    StrafeTimer += DeltaTime;
    if (StrafeTimer > 2.0f || StrafeDirection.IsZero())
    {
        // 每2秒改变方向
        FVector ToTarget = (CurrentTarget->GetActorLocation() - 
                            MyPawn->GetActorLocation()).GetSafeNormal();
        FVector RightVector = FVector::CrossProduct(ToTarget, FVector::UpVector);
        
        float StrafeDir = FMath::RandBool() ? 1.0f : -1.0f;
        StrafeDirection = RightVector * StrafeDir;
        
        StrafeTimer = 0.0f;
    }

    // 应用扫射移动（通过 Input Component）
    if (ULyraInputComponent* InputComp = Cast<ULyraInputComponent>(MyPawn->InputComponent))
    {
        // 注入移动输入
        FVector MoveInput = StrafeDirection;
        // InputComp->InjectMoveInput(MoveInput); // 假设有这个接口
    }
}

bool ALyraShooterBotController::ShouldRetreat() const
{
    // 血量低于30% 撤退
    if (ULyraHealthComponent* HealthComp = ULyraHealthComponent::FindHealthComponent(GetPawn()))
    {
        float HealthPercent = HealthComp->GetHealth() / HealthComp->GetMaxHealth();
        return HealthPercent < 0.3f;
    }
    return false;
}
```

### 8.2 Behavior Tree 资产配置

在编辑器中创建 Behavior Tree 资产（`BT_LyraShooterBot`）：

```
Root
└── Selector
    ├── Sequence: Handle Death
    │   ├── [Decorator: Blackboard Based Condition - IsDead == true]
    │   └── Task: Stop Movement
    │
    ├── Sequence: Combat Mode
    │   ├── [Decorator: Blackboard Based Condition - CurrentTarget != nullptr]
    │   ├── Service: Update Combat State (0.2s interval)
    │   └── Selector: Combat Actions
    │       ├── Sequence: Retreat
    │       │   ├── [Decorator: Check Health < 30%]
    │       │   ├── Task: Find Cover
    │       │   └── Task: Move To Cover
    │       │
    │       ├── Sequence: Aggressive Attack
    │       │   ├── [Decorator: Distance > 1000]
    │       │   ├── Task: Move To Target
    │       │   └── Task: Fire Weapon
    │       │
    │       └── Sequence: Defensive Combat
    │           ├── Task: Strafe Around Target
    │           └── Task: Fire Weapon
    │
    ├── Sequence: Investigate
    │   ├── [Decorator: Blackboard Based Condition - LastKnownLocation != (0,0,0)]
    │   ├── Task: Move To LastKnownLocation
    │   └── Task: Clear LastKnownLocation
    │
    └── Sequence: Patrol
        ├── Task: Get Next Patrol Point
        ├── Task: Move To PatrolPoint
        └── Task: Wait (2-4 seconds)
```

---

## 9. 实战案例 2：协作型 AI（团队配合）

### 9.1 Squad 系统设计

```cpp
// LyraAISquadComponent.h
UCLASS()
class ULyraAISquadComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    // Squad 角色
    UENUM(BlueprintType)
    enum class ESquadRole : uint8
    {
        Leader,
        Assaulter,
        Support,
        Sniper
    };

    // Squad 命令
    UENUM(BlueprintType)
    enum class ESquadCommand : uint8
    {
        None,
        AttackTarget,
        DefendPosition,
        Regroup,
        Flank,
        Suppress
    };

    // 加入 Squad
    void JoinSquad(ULyraAISquadComponent* LeaderSquad);
    
    // 离开 Squad
    void LeaveSquad();

    // 发布命令（仅 Leader）
    void IssueCommand(ESquadCommand Command, AActor* Target = nullptr, 
                      FVector Location = FVector::ZeroVector);

    // 获取 Squad 成员
    TArray<AAIController*> GetSquadMembers() const;

    // 分配角色
    void AssignRole(AAIController* Member, ESquadRole Role);

protected:
    UPROPERTY(Replicated)
    TObjectPtr<AAIController> SquadLeader;

    UPROPERTY(Replicated)
    TArray<TObjectPtr<AAIController>> SquadMembers;

    UPROPERTY(Replicated)
    TMap<TObjectPtr<AAIController>, ESquadRole> MemberRoles;

    UPROPERTY(Replicated)
    ESquadCommand CurrentCommand;

    UPROPERTY(Replicated)
    TObjectPtr<AActor> CommandTarget;

    UPROPERTY(Replicated)
    FVector CommandLocation;

private:
    void OnCommandReceived(ESquadCommand Command, AActor* Target, FVector Location);
};

// LyraAISquadComponent.cpp
void ULyraAISquadComponent::IssueCommand(ESquadCommand Command, AActor* Target, FVector Location)
{
    AAIController* MyController = Cast<AAIController>(GetOwner());
    if (MyController != SquadLeader)
    {
        UE_LOG(LogTemp, Warning, TEXT("Only squad leader can issue commands!"));
        return;
    }

    CurrentCommand = Command;
    CommandTarget = Target;
    CommandLocation = Location;

    // 通知所有成员
    for (AAIController* Member : SquadMembers)
    {
        if (ULyraAISquadComponent* MemberSquad = 
            Member->FindComponentByClass<ULyraAISquadComponent>())
        {
            MemberSquad->OnCommandReceived(Command, Target, Location);
        }
    }
}

void ULyraAISquadComponent::OnCommandReceived(ESquadCommand Command, AActor* Target, FVector Location)
{
    AAIController* MyController = Cast<AAIController>(GetOwner());
    if (!MyController) return;

    UBlackboardComponent* BlackboardComp = MyController->GetBlackboardComponent();
    if (!BlackboardComp) return;

    ESquadRole MyRole = MemberRoles.FindRef(MyController);

    switch (Command)
    {
    case ESquadCommand::AttackTarget:
        // 根据角色执行不同的攻击策略
        if (MyRole == ESquadRole::Assaulter)
        {
            // 突击手直接冲锋
            BlackboardComp->SetValueAsObject("CurrentTarget", Target);
            BlackboardComp->SetValueAsBool("AggressiveMode", true);
        }
        else if (MyRole == ESquadRole::Support)
        {
            // 支援手保持距离火力压制
            BlackboardComp->SetValueAsObject("CurrentTarget", Target);
            BlackboardComp->SetValueAsBool("SuppressMode", true);
        }
        else if (MyRole == ESquadRole::Sniper)
        {
            // 狙击手寻找狙击点
            FVector SnipePosition = FindSnipePosition(Location, Target);
            BlackboardComp->SetValueAsVector("SnipePosition", SnipePosition);
        }
        break;

    case ESquadCommand::Flank:
        // 计算包抄位置
        if (MyRole == ESquadRole::Assaulter)
        {
            FVector FlankPosition = CalculateFlankPosition(Target, Location);
            BlackboardComp->SetValueAsVector("FlankPosition", FlankPosition);
        }
        break;

    case ESquadCommand::DefendPosition:
        // 所有成员移动到防守位置
        FVector DefensePosition = CalculateDefensePosition(Location, MyRole);
        BlackboardComp->SetValueAsVector("DefensePosition", DefensePosition);
        break;

    case ESquadCommand::Regroup:
        // 集结到 Leader 位置
        if (SquadLeader && SquadLeader->GetPawn())
        {
            BlackboardComp->SetValueAsVector("RegroupLocation", 
                                              SquadLeader->GetPawn()->GetActorLocation());
        }
        break;
    }
}

FVector ULyraAISquadComponent::CalculateFlankPosition(AActor* Target, FVector RallyPoint) const
{
    if (!Target) return FVector::ZeroVector;

    // 计算包抄角度（90度侧翼）
    FVector ToTarget = (Target->GetActorLocation() - RallyPoint).GetSafeNormal();
    FVector FlankDirection = FVector::CrossProduct(ToTarget, FVector::UpVector).GetSafeNormal();
    
    // 随机选择左侧或右侧
    if (FMath::RandBool())
    {
        FlankDirection *= -1.0f;
    }

    // 包抄位置：目标侧面 800 单位
    return Target->GetActorLocation() + FlankDirection * 800.0f;
}
```

### 9.2 协同战斗示例

```cpp
// Squad Leader 决策逻辑
void ALyraSquadLeaderAI::EvaluateTacticalSituation()
{
    ULyraAISquadComponent* SquadComp = FindComponentByClass<ULyraAISquadComponent>();
    if (!SquadComp) return;

    // 分析战场情况
    TArray<AActor*> NearbyEnemies = GetNearbyEnemies(2000.0f);
    TArray<AAIController*> AvailableMembers = SquadComp->GetSquadMembers();

    if (NearbyEnemies.Num() == 0)
    {
        // 无敌人 -> 巡逻
        SquadComp->IssueCommand(ULyraAISquadComponent::ESquadCommand::None);
        return;
    }

    if (NearbyEnemies.Num() == 1)
    {
        // 单个敌人 -> 包抄战术
        AActor* Target = NearbyEnemies[0];
        SquadComp->IssueCommand(
            ULyraAISquadComponent::ESquadCommand::Flank,
            Target,
            GetPawn()->GetActorLocation()
        );
    }
    else if (NearbyEnemies.Num() > AvailableMembers.Num())
    {
        // 敌众我寡 -> 防守姿态
        FVector DefensePoint = FindBestDefensePosition();
        SquadComp->IssueCommand(
            ULyraAISquadComponent::ESquadCommand::DefendPosition,
            nullptr,
            DefensePoint
        );
    }
    else
    {
        // 势均力敌 -> 正面进攻
        AActor* PriorityTarget = SelectPriorityTarget(NearbyEnemies);
        SquadComp->IssueCommand(
            ULyraAISquadComponent::ESquadCommand::AttackTarget,
            PriorityTarget
        );
    }
}

AActor* ALyraSquadLeaderAI::SelectPriorityTarget(const TArray<AActor*>& Enemies) const
{
    // 优先级：低血量 > 近距离 > 威胁等级
    AActor* BestTarget = nullptr;
    float BestScore = -MAX_FLT;

    for (AActor* Enemy : Enemies)
    {
        float Score = 0.0f;

        // 血量评分
        if (ULyraHealthComponent* HealthComp = ULyraHealthComponent::FindHealthComponent(Enemy))
        {
            float HealthPercent = HealthComp->GetHealth() / HealthComp->GetMaxHealth();
            Score += (1.0f - HealthPercent) * 50.0f; // 血量越低，分数越高
        }

        // 距离评分
        float Distance = FVector::Dist(GetPawn()->GetActorLocation(), Enemy->GetActorLocation());
        Score += (2000.0f - Distance) / 100.0f; // 距离越近，分数越高

        // 威胁等级评分
        // （可以根据敌人武器类型、技能等判定）
        Score += 20.0f;

        if (Score > BestScore)
        {
            BestScore = Score;
            BestTarget = Enemy;
        }
    }

    return BestTarget;
}
```

### 9.3 火力压制示例

```cpp
// BTTask_SuppressFire.h
UCLASS()
class UBTTask_SuppressFire : public UBTTaskNode
{
    GENERATED_BODY()

public:
    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
                                             uint8* NodeMemory) override;
    virtual void TickTask(UBehaviorTreeComponent& OwnerComp, 
                          uint8* NodeMemory, 
                          float DeltaSeconds) override;

protected:
    UPROPERTY(EditAnywhere)
    FBlackboardKeySelector TargetLocationKey;

    UPROPERTY(EditAnywhere)
    float SuppressDuration = 5.0f;

    UPROPERTY(EditAnywhere)
    float BurstInterval = 0.3f; // 点射间隔
};

// BTTask_SuppressFire.cpp
EBTNodeResult::Type UBTTask_SuppressFire::ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
                                                       uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return EBTNodeResult::Failed;

    // 获取压制目标位置
    FVector TargetLocation = OwnerComp.GetBlackboardComponent()->GetValueAsVector(
        TargetLocationKey.SelectedKeyName
    );

    // 设置瞄准方向
    if (APawn* MyPawn = AIController->GetPawn())
    {
        FVector AimDirection = (TargetLocation - MyPawn->GetActorLocation()).GetSafeNormal();
        
        // 添加随机散布（火力压制不需要精准）
        FRotator SpreadRotation(
            FMath::RandRange(-5.0f, 5.0f),
            FMath::RandRange(-5.0f, 5.0f),
            0.0f
        );
        AimDirection = SpreadRotation.RotateVector(AimDirection);

        AIController->SetFocalPoint(MyPawn->GetActorLocation() + AimDirection * 1000.0f);
    }

    // 初始化计时器
    float* ElapsedTime = (float*)NodeMemory;
    *ElapsedTime = 0.0f;

    return EBTNodeResult::InProgress;
}

void UBTTask_SuppressFire::TickTask(UBehaviorTreeComponent& OwnerComp, 
                                     uint8* NodeMemory, 
                                     float DeltaSeconds)
{
    float* ElapsedTime = (float*)NodeMemory;
    *ElapsedTime += DeltaSeconds;

    AAIController* AIController = OwnerComp.GetAIOwner();
    APawn* MyPawn = AIController->GetPawn();

    // 点射模式：开火 -> 停火 -> 开火...
    float CycleTime = fmod(*ElapsedTime, BurstInterval * 2.0f);
    bool bShouldFire = (CycleTime < BurstInterval);

    if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(MyPawn))
    {
        if (bShouldFire)
        {
            // 开火
            if (!ASC->IsAbilityActive(FGameplayTag::RequestGameplayTag("Ability.Weapon.Fire")))
            {
                ASC->TryActivateAbilitiesByTag(
                    FGameplayTagContainer(FGameplayTag::RequestGameplayTag("Ability.Weapon.Fire"))
                );
            }
        }
        else
        {
            // 停火
            ASC->CancelAbilities(&FGameplayTagContainer(
                FGameplayTag::RequestGameplayTag("Ability.Weapon.Fire")
            ));
        }
    }

    // 完成任务
    if (*ElapsedTime >= SuppressDuration)
    {
        ASC->CancelAbilities();
        FinishLatentTask(OwnerComp, EBTNodeResult::Succeeded);
    }
}
```

---

## 10. 调试技巧

### 10.1 AI 调试命令

```cpp
// LyraAIDebugCommands.h
UCLASS()
class ULyraAIDebugCommands : public UCheatManagerExtension
{
    GENERATED_BODY()

public:
    // 显示 AI 调试信息
    UFUNCTION(Exec)
    void AI_ShowDebug(bool bShow = true);

    // 显示 Behavior Tree 状态
    UFUNCTION(Exec)
    void AI_ShowBehaviorTree(bool bShow = true);

    // 显示感知系统
    UFUNCTION(Exec)
    void AI_ShowPerception(bool bShow = true);

    // 显示导航路径
    UFUNCTION(Exec)
    void AI_ShowNavigation(bool bShow = true);

    // 冻结所有 AI
    UFUNCTION(Exec)
    void AI_FreezeAll(bool bFreeze = true);

    // 让 AI 自杀（测试重生）
    UFUNCTION(Exec)
    void AI_KillAll();

    // 让所有 AI 攻击目标
    UFUNCTION(Exec)
    void AI_AttackPlayer(int32 PlayerIndex = 0);
};
```

### 10.2 Visual Logger 集成

```cpp
// 在 AI Controller 中集成 Visual Logger
void ALyraShooterBotController::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

#if ENABLE_VISUAL_LOG
    // 记录当前状态
    UE_VLOG(this, LogLyraAI, Log, TEXT("State: %s"), 
            *UEnum::GetValueAsString(CurrentState));

    // 记录当前目标
    if (CurrentTarget)
    {
        UE_VLOG_LOCATION(this, LogLyraAI, Log, CurrentTarget->GetActorLocation(), 
                         30.0f, FColor::Red, TEXT("Target: %s"), 
                         *CurrentTarget->GetName());
        
        UE_VLOG_SEGMENT(this, LogLyraAI, Log, 
                        GetPawn()->GetActorLocation(), 
                        CurrentTarget->GetActorLocation(), 
                        FColor::Yellow, TEXT(""));
    }

    // 记录感知到的 Actor
    for (const auto& Pair : PerceivedActors)
    {
        AActor* PerceivedActor = Pair.Key;
        UE_VLOG_LOCATION(this, LogLyraAI, Verbose, 
                         PerceivedActor->GetActorLocation(), 
                         20.0f, FColor::Blue, TEXT("Perceived"));
    }
#endif
}
```

**使用 Visual Logger：**
1. 打开：`Window > Developer Tools > Visual Logger`
2. 开始记录：按 `F8`（默认）或 `VisLog Start`
3. 停止记录：再次按 `F8` 或 `VisLog Stop`
4. 回放和分析 AI 行为

### 10.3 在屏幕上绘制调试信息

```cpp
void ALyraShooterBotController::DisplayDebug(UCanvas* Canvas, 
                                              const FDebugDisplayInfo& DebugDisplay, 
                                              float& YL, float& YPos)
{
    Super::DisplayDebug(Canvas, DebugDisplay, YL, YPos);

    if (!GetPawn()) return;

    FDisplayDebugManager& DisplayDebugManager = Canvas->DisplayDebugManager;

    DisplayDebugManager.SetDrawColor(FColor::Yellow);
    DisplayDebugManager.DrawString(FString::Printf(TEXT("AI State: %s"), 
                                    *UEnum::GetValueAsString(CurrentState)));

    if (CurrentTarget)
    {
        DisplayDebugManager.DrawString(FString::Printf(TEXT("Target: %s"), 
                                        *CurrentTarget->GetName()));
        DisplayDebugManager.DrawString(FString::Printf(TEXT("Distance: %.1f"), 
                                        FVector::Dist(GetPawn()->GetActorLocation(), 
                                                      CurrentTarget->GetActorLocation())));
    }

    DisplayDebugManager.DrawString(FString::Printf(TEXT("Health: %.1f%%"), 
                                    GetHealthPercentage() * 100.0f));

    if (ULyraAISquadComponent* SquadComp = FindComponentByClass<ULyraAISquadComponent>())
    {
        DisplayDebugManager.DrawString(FString::Printf(TEXT("Squad Members: %d"), 
                                        SquadComp->GetSquadMembers().Num()));
    }
}
```

### 10.4 Gameplay Debugger 集成

```cpp
// LyraAIGameplayDebuggerCategory.h
#if WITH_GAMEPLAY_DEBUGGER
class FLyraAIGameplayDebuggerCategory : public FGameplayDebuggerCategory
{
public:
    FLyraAIGameplayDebuggerCategory();

    virtual void CollectData(APlayerController* OwnerPC, AActor* DebugActor) override;
    virtual void DrawData(APlayerController* OwnerPC, FGameplayDebuggerCanvasContext& CanvasContext) override;

protected:
    struct FRepData
    {
        FString AIState;
        FString CurrentTarget;
        TArray<FString> PerceivedActors;
        FString BlackboardData;
    };

    FRepData DataPack;
};

// 注册到 Gameplay Debugger Module
IGameplayDebugger& GameplayDebuggerModule = IGameplayDebugger::Get();
GameplayDebuggerModule.RegisterCategory("LyraAI", 
    IGameplayDebugger::FOnGetCategory::CreateStatic(&FLyraAIGameplayDebuggerCategory::MakeInstance));
#endif
```

---

## 11. 性能优化

### 11.1 AI Tick 优化

```cpp
// 使用 Tick Interval 减少更新频率
ALyraShooterBotController::ALyraShooterBotController(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 设置 Tick 间隔（每 0.1 秒 Tick 一次，而非每帧）
    PrimaryActorTick.TickInterval = 0.1f;
    
    // 低优先级 Tick（允许引擎跳过 Tick）
    PrimaryActorTick.bCanEverTick = true;
    PrimaryActorTick.bStartWithTickEnabled = true;
    PrimaryActorTick.TickGroup = TG_PostPhysics;
}

// 根据距离玩家的远近动态调整 Tick 频率
void ALyraShooterBotController::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 动态调整 Tick 间隔
    if (APlayerController* LocalPC = GetWorld()->GetFirstPlayerController())
    {
        float Distance = FVector::Dist(GetPawn()->GetActorLocation(), 
                                        LocalPC->GetPawn()->GetActorLocation());

        if (Distance > 3000.0f)
        {
            // 远距离 AI：降低更新频率
            PrimaryActorTick.TickInterval = 0.5f;
        }
        else if (Distance > 1500.0f)
        {
            PrimaryActorTick.TickInterval = 0.2f;
        }
        else
        {
            // 近距离 AI：正常频率
            PrimaryActorTick.TickInterval = 0.1f;
        }
    }
}
```

### 11.2 Perception 优化

```cpp
// 配置感知更新频率
SightConfig->SetMaxAge(5.0f);                    // 感知记忆时长
PerceptionComponent->SetSenseEnabled(UAISense_Sight::StaticClass(), true);

// 使用请求式更新，而非每帧更新
PerceptionComponent->RequestStimuliListenerUpdate();

// 根据 AI 数量动态调整感知范围
void ULyraAIManager::OptimizePerceptionRanges()
{
    int32 ActiveAICount = GetActiveAICount();

    float OptimalSightRadius = 3000.0f;
    if (ActiveAICount > 20)
    {
        OptimalSightRadius = 2000.0f; // 降低感知范围
    }
    else if (ActiveAICount > 50)
    {
        OptimalSightRadius = 1500.0f;
    }

    // 批量更新所有 AI 的感知配置
    for (AAIController* AI : ActiveAIs)
    {
        if (UAIPerceptionComponent* PerceptionComp = AI->GetPerceptionComponent())
        {
            if (UAISenseConfig_Sight* SightCfg = 
                Cast<UAISenseConfig_Sight>(PerceptionComp->GetSenseConfig(UAISense_Sight::StaticClass())))
            {
                SightCfg->SightRadius = OptimalSightRadius;
            }
        }
    }
}
```

### 11.3 行为树优化

```cpp
// 使用条件装饰器提前剪枝
// 避免不必要的任务执行

// 示例：在 Sequence 开头就检查条件
Root
└── Selector
    ├── Sequence: Combat
    │   ├── [Decorator: Has Target?] ← 提前检查，避免后续无意义执行
    │   ├── [Decorator: Has Ammo?]
    │   └── ...

// 调整 Service 更新频率
// BTService_UpdateCombatState.cpp
UBTService_UpdateCombatState::UBTService_UpdateCombatState()
{
    NodeName = "Update Combat State";
    Interval = 0.5f;           // 0.5 秒更新一次
    RandomDeviation = 0.1f;    // 添加随机偏移，避免所有 AI 同时更新
}
```

### 11.4 导航优化

```cpp
// 使用导航查询过滤器
FPathFindingQuery Query;
Query.NavAgentProperties = GetNavAgentPropertiesRef();
Query.QueryFilter = UNavigationQueryFilter::GetQueryFilter<FDefaultQueryFilter>(*GetWorld()->GetNavigationSystem());

// 启用路径缓存
UPROPERTY(Config)
bool bUsePathCache = true;

// 限制路径复杂度
UPROPERTY(Config)
int32 MaxPathNodes = 256;

// 使用导航层级（Nav Mesh Layers）
// 允许不同类型 AI 使用不同的导航网格
```

### 11.5 AI Pooling（AI 对象池）

```cpp
// LyraAIPoolSubsystem.h
UCLASS()
class ULyraAIPoolSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    // 获取 AI（从池中或新建）
    AAIController* AcquireAI(TSubclassOf<AAIController> AIClass);

    // 归还 AI 到池中
    void ReleaseAI(AAIController* AI);

protected:
    // AI 池
    UPROPERTY()
    TMap<TSubclassOf<AAIController>, TArray<TObjectPtr<AAIController>>> AIPool;

    // 最大池大小
    UPROPERTY(Config)
    int32 MaxPoolSize = 20;
};

// LyraAIPoolSubsystem.cpp
AAIController* ULyraAIPoolSubsystem::AcquireAI(TSubclassOf<AAIController> AIClass)
{
    TArray<TObjectPtr<AAIController>>* PoolArray = AIPool.Find(AIClass);
    
    if (PoolArray && PoolArray->Num() > 0)
    {
        // 从池中取出
        AAIController* AI = (*PoolArray).Pop();
        AI->SetActorHiddenInGame(false);
        AI->SetActorTickEnabled(true);
        return AI;
    }

    // 池中无可用，创建新 AI
    FActorSpawnParameters SpawnParams;
    SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    
    return GetWorld()->SpawnActor<AAIController>(AIClass, FVector::ZeroVector, 
                                                  FRotator::ZeroRotator, SpawnParams);
}

void ULyraAIPoolSubsystem::ReleaseAI(AAIController* AI)
{
    if (!AI) return;

    // 清理 AI 状态
    if (APawn* Pawn = AI->GetPawn())
    {
        AI->UnPossess();
        Pawn->Destroy();
    }

    AI->SetActorHiddenInGame(true);
    AI->SetActorTickEnabled(false);

    // 归还到池中
    TArray<TObjectPtr<AAIController>>& PoolArray = AIPool.FindOrAdd(AI->GetClass());
    
    if (PoolArray.Num() < MaxPoolSize)
    {
        PoolArray.Add(AI);
    }
    else
    {
        // 池已满，销毁
        AI->Destroy();
    }
}
```

---

## 12. 总结与最佳实践

### 12.1 Lyra AI 设计原则

1. **模块化优先**：使用组件和接口，而非硬编码继承
2. **数据驱动**：AI 行为通过数据资产配置，而非代码
3. **GAS 集成**：AI 与玩家共享能力系统，确保一致性
4. **团队系统**：利用 `ILyraTeamAgentInterface` 实现敌友判定
5. **网络优化**：AI 仅在服务器运行，关键数据选择性复制

### 12.2 常见陷阱

**❌ 错误做法：**
```cpp
// 在客户端运行 AI 逻辑
void ALyraPlayerBotController::Tick(float DeltaTime)
{
    // 没有检查 Authority！
    UpdateAIBehavior();
}

// 直接操作输入，绕过 GAS
void FireWeapon()
{
    // 错误：应该通过能力系统
    WeaponComponent->Fire();
}
```

**✅ 正确做法：**
```cpp
void ALyraPlayerBotController::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 仅在服务器执行
    if (!HasAuthority()) return;

    UpdateAIBehavior();
}

void FireWeapon()
{
    // 正确：通过 GAS 激活能力
    if (UAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        ASC->TryActivateAbilitiesByTag(TAG_Ability_Fire);
    }
}
```

### 12.3 性能检查清单

- [ ] AI Tick 间隔是否合理？（0.1-0.5秒）
- [ ] Perception 范围是否过大？（<3000 单位）
- [ ] Behavior Tree Service 频率是否过高？（>0.2秒）
- [ ] 是否使用了导航路径缓存？
- [ ] 远距离 AI 是否降低了更新频率？
- [ ] 是否实现了 AI 对象池？
- [ ] 是否使用了 Visual Logger 分析性能？

### 12.4 进阶扩展方向

1. **机器学习 AI**：集成 UE5 的 ML Adapter，训练自适应 AI
2. **高级决策树**：实现 Hierarchical Task Network (HTN)
3. **社交行为**：AI 之间的交流、协作、竞争
4. **情感系统**：AI 根据情感状态改变行为
5. **自适应难度**：AI 根据玩家表现动态调整

### 12.5 参考资源

- **UE5 官方文档**：[AI Overview](https://docs.unrealengine.com/5.0/en-US/artificial-intelligence-in-unreal-engine/)
- **Lyra 源码**：`/Game/LyraCore/AI/` 和 `/Plugins/ModularGameplayActors/`
- **Gameplay Ability System**：[GAS Documentation](https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-for-unreal-engine/)
- **Visual Logger**：[Visual Logger Guide](https://docs.unrealengine.com/5.0/en-US/visual-logger-in-unreal-engine/)

---

## 结语

Lyra 的 AI 系统展示了如何在现代游戏引擎中构建**可扩展、高性能、易调试**的 AI 架构。通过模块化设计、GAS 集成和团队系统，我们可以快速实现从简单 Bot 到复杂协作 AI 的各种需求。

记住：**好的 AI 不是最聪明的，而是最有趣的**。在追求技术实现的同时，始终关注玩家体验——AI 应该为游戏增添挑战和乐趣，而非挫败感。

**下一步：** 结合本文知识，尝试在 Lyra 中实现你自己的 Bot 行为，并通过 Visual Logger 和 Gameplay Debugger 不断优化！🎮🤖
