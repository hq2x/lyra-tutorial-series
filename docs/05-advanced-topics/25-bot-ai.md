# Bot AI：行为树与 EQS 查询

> Lyra 系列教程第 25 篇：深入探讨 Lyra 的 AI Bot 系统设计、行为树（Behavior Tree）编程、环境查询系统（EQS）与 GAS 整合的完整解决方案

---

## 📘 本章概览

在多人在线游戏中，AI Bot 不仅可以作为单人模式的对手，还能在多人模式中填充玩家空缺、调整游戏难度、提供训练对象。Lyra 的 Bot 系统展示了如何将模块化设计、GAS 技能系统与传统 AI 架构（行为树、EQS）有机结合,打造灵活且高性能的 AI 解决方案。

本文将深入剖析:
- Lyra Bot 创建流程与架构设计
- ModularAIController 的扩展机制
- 行为树（Behavior Tree）的实战应用
- 环境查询系统（EQS）的高级用法
- AI 与 GAS 的深度整合
- 多人游戏中的 AI 网络优化
- 完整实战项目：从零开始实现战术 AI

**预计阅读时间**: 45-60 分钟  
**难度等级**: ⭐⭐⭐⭐☆ (进阶)

---

## 1. Lyra Bot 系统架构概览

### 1.1 Bot 在 Lyra 中的定位

Lyra 的 Bot 系统遵循以下设计原则：

1. **与玩家一致性**: Bot 使用相同的 Pawn、Experience、GAS 技能
2. **模块化扩展**: 通过 ModularAIController 支持 Game Feature 动态添加 AI 行为
3. **数据驱动**: AI 配置通过 Data Assets 管理,支持热更新
4. **性能优先**: 区分本地/远程 Bot,优化网络流量和计算开销

### 1.2 核心类层次结构

```
AAIController (引擎基类)
    └── AModularAIController (Lyra 扩展)
        └── ALyraPlayerBotController (ShooterCore 实现)
            └── BP_BotController_Shooter (蓝图子类)
```

```cpp
// ModularAIController.h - Lyra AI 控制器基类
UCLASS(MinimalAPI, Blueprintable)
class AModularAIController : public AAIController
{
    GENERATED_BODY()

public:
    // 支持模块化组件扩展
    virtual void PreInitializeComponents() override;
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
};
```

**关键特性**:
- 继承自 `AAIController`,复用引擎的导航、感知、行为树系统
- 实现生命周期钩子,允许 Game Feature Actions 注入组件
- 蓝图友好,策划可以快速迭代 AI 行为

---

## 2. Bot 创建与管理

### 2.1 LyraBotCreationComponent 深度解析

Lyra 通过 `ULyraBotCreationComponent` 实现自动化 Bot 生成:

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

    // Bot AI 控制器类
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Gameplay)
    TSubclassOf<AAIController> BotControllerClass;

    // 随机 Bot 名称池
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Gameplay)
    TArray<FString> RandomBotNames;

    // 已生成的 Bot 列表
    UPROPERTY(Transient)
    TArray<TObjectPtr<AAIController>> SpawnedBotList;

public:
    // 生成单个 Bot
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Gameplay)
    virtual void SpawnOneBot();

    // 移除一个 Bot
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Gameplay)
    virtual void RemoveOneBot();

    // 批量生成 Bot (可被蓝图重写)
    UFUNCTION(BlueprintNativeEvent, BlueprintAuthorityOnly, Category=Gameplay)
    void ServerCreateBots();
};
```

### 2.2 Bot 生成流程详解

```cpp
// LyraBotCreationComponent.cpp
void ULyraBotCreationComponent::SpawnOneBot()
{
    // 1. 配置生成参数
    FActorSpawnParameters SpawnInfo;
    SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    SpawnInfo.OverrideLevel = GetComponentLevel();
    SpawnInfo.ObjectFlags |= RF_Transient; // 标记为临时对象,不保存

    // 2. 生成 AI 控制器
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

        // 3. 设置玩家状态和名称
        if (NewController->PlayerState != nullptr)
        {
            NewController->PlayerState->SetPlayerName(
                CreateBotName(NewController->PlayerState->GetPlayerId())
            );
        }

        // 4. 初始化（分配队伍、Pawn 等）
        GameMode->GenericPlayerInitialization(NewController);

        // 5. 重生 Bot (生成 Pawn)
        GameMode->RestartPlayer(NewController);

        // 6. 强制初始化 Pawn 扩展组件
        if (NewController->GetPawn() != nullptr)
        {
            if (ULyraPawnExtensionComponent* PawnExtComponent = 
                NewController->GetPawn()->FindComponentByClass<ULyraPawnExtensionComponent>())
            {
                PawnExtComponent->CheckDefaultInitialization();
            }
        }

        // 7. 记录到列表
        SpawnedBotList.Add(NewController);
    }
}
```

**流程图**:

```
[Experience 加载完成]
        ↓
[OnExperienceLoaded 回调]
        ↓
[ServerCreateBots_Implementation]
        ↓
    ┌───────────────┐
    │ 读取配置:     │
    │ • NumBots     │
    │ • 开发者设置  │
    │ • URL 参数    │
    └───┬───────────┘
        ↓
    [循环生成 Bot]
        ↓
    ┌───────────────────┐
    │ SpawnOneBot():    │
    │ 1. Spawn 控制器   │
    │ 2. 设置名称/队伍  │
    │ 3. 生成 Pawn      │
    │ 4. 初始化组件     │
    └───────────────────┘
```

### 2.3 动态 Bot 数量控制

Lyra 支持多种方式覆盖 Bot 数量:

```cpp
void ULyraBotCreationComponent::ServerCreateBots_Implementation()
{
    int32 EffectiveBotCount = NumBotsToCreate;

    // 1. 编辑器开发者设置覆盖
    if (GIsEditor)
    {
        const ULyraDeveloperSettings* DeveloperSettings = GetDefault<ULyraDeveloperSettings>();
        if (DeveloperSettings->bOverrideBotCount)
        {
            EffectiveBotCount = DeveloperSettings->OverrideNumPlayerBotsToSpawn;
        }
    }

    // 2. URL 参数覆盖 (优先级最高)
    if (AGameModeBase* GameModeBase = GetGameMode<AGameModeBase>())
    {
        EffectiveBotCount = UGameplayStatics::GetIntOption(
            GameModeBase->OptionsString, 
            TEXT("NumBots"), 
            EffectiveBotCount
        );
    }

    // 3. 生成指定数量的 Bot
    for (int32 Count = 0; Count < EffectiveBotCount; ++Count)
    {
        SpawnOneBot();
    }
}
```

**命令行启动示例**:
```bash
# 启动 8 个 Bot 的多人游戏
LyraGame.exe /Game/Maps/L_ShooterGym?NumBots=8 -log
```

---

## 3. ModularAIController 扩展机制

### 3.1 为什么需要 Modular AI

传统 AI 控制器的问题:
- **硬编码耦合**: AI 逻辑写死在控制器类中
- **扩展困难**: 新 AI 行为需要修改核心代码
- **不支持热插拔**: Game Features 无法动态添加 AI 模块

ModularAIController 解决方案:
```cpp
// ModularAIController.cpp
void AModularAIController::PreInitializeComponents()
{
    Super::PreInitializeComponents();

    // 触发组件注册
    // Game Feature Actions 可以在此注入:
    // - 感知组件 (UAIPerceptionComponent)
    // - 黑板数据组件
    // - 自定义决策组件
    UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}

void AModularAIController::BeginPlay()
{
    // 检查所有组件是否就绪
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        this, 
        UGameFrameworkComponentManager::NAME_GameActorReady
    );

    Super::BeginPlay();
}

void AModularAIController::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    // 清理组件注册
    UGameFrameworkComponentManager::RemoveGameFrameworkComponentReceiver(this);
    Super::EndPlay(EndPlayReason);
}
```

### 3.2 实战：通过 Game Feature 注入 AI 组件

创建 `GameFeatureAction_AddAIComponents`:

```cpp
// GameFeatureAction_AddAIComponents.h
UCLASS(MinimalAPI, meta = (DisplayName = "Add AI Components"))
class UGameFeatureAction_AddAIComponents final : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    // 要添加到 AI 控制器的组件列表
    UPROPERTY(EditAnywhere, Category = "Components")
    TArray<FComponentEntry> ComponentList;

    // FComponentEntry 定义
    struct FComponentEntry
    {
        // 目标 Actor 类 (如 AModularAIController)
        TSoftClassPtr<AActor> ActorClass;

        // 要添加的组件类
        TSubclassOf<UActorComponent> ComponentClass;

        // 是否只在服务器添加
        bool bServerOnly = false;
    };

    //~UGameFeatureAction interface
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override;
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;
    //~End of UGameFeatureAction interface

private:
    struct FContextHandles
    {
        TArray<TSharedPtr<FComponentRequestHandle>> ComponentRequestHandles;
    };

    TMap<FGameFeatureStateChangeContext, FContextHandles> ContextData;
};
```

```cpp
// GameFeatureAction_AddAIComponents.cpp
void UGameFeatureAction_AddAIComponents::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
    FContextHandles& Handles = ContextData.FindOrAdd(Context);

    for (const FComponentEntry& Entry : ComponentList)
    {
        // 检查是否仅服务器且当前是客户端
        if (Entry.bServerOnly && !Context.WorldContext->World()->IsNetMode(NM_DedicatedServer))
        {
            continue;
        }

        // 通过 ComponentManager 注册组件请求
        UGameFrameworkComponentManager* Manager = 
            UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(Context.WorldContext->OwningGameInstance);

        if (Manager && Entry.ActorClass.IsValid() && Entry.ComponentClass)
        {
            TSharedPtr<FComponentRequestHandle> Handle = Manager->AddComponentRequest(
                Entry.ActorClass.LoadSynchronous(),
                Entry.ComponentClass
            );

            Handles.ComponentRequestHandles.Add(Handle);
        }
    }
}

void UGameFeatureAction_AddAIComponents::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    if (FContextHandles* Handles = ContextData.Find(Context))
    {
        // 移除所有组件请求
        Handles->ComponentRequestHandles.Empty();
        ContextData.Remove(Context);
    }
}
```

**在 ShooterCore Game Feature 中配置**:

```
ShooterCore.uplugin
└── Actions
    └── AddAIComponents
        ├── ActorClass: /Script/ModularGameplayActors.ModularAIController
        └── ComponentList
            ├── [0] UAIPerceptionComponent (视觉/听觉感知)
            ├── [1] UBehaviorTreeComponent (行为树)
            └── [2] ULyraAITargetingComponent (自定义瞄准逻辑)
```

---

## 4. 行为树（Behavior Tree）实战

### 4.1 Lyra 行为树架构

Lyra 的 AI 决策采用经典的行为树模式:

```
BT_ShooterBot (行为树资产)
├── Root (Sequence)
│   ├── 检查是否存活 (Decorator: Blackboard Check "IsAlive")
│   ├── 更新威胁感知 (Service: Update Enemy Info)
│   └── 战术决策选择器 (Selector)
│       ├── 战斗模式 (Sequence)
│       │   ├── Condition: Has Enemy
│       │   ├── 选择战斗位置 (Task: Find Cover)
│       │   ├── 移动到掩体 (Task: Move To)
│       │   └── 开火 (Task: Use Weapon Ability)
│       ├── 巡逻模式 (Sequence)
│       │   ├── Condition: No Enemy
│       │   ├── 选择巡逻点 (EQS Query)
│       │   └── 移动到巡逻点
│       └── 闲置模式 (Task: Idle Animation)
```

### 4.2 自定义行为树任务：释放 GAS 技能

创建 `BTTask_ActivateAbility`:

```cpp
// BTTask_ActivateAbility.h
UCLASS()
class UBTTask_ActivateAbility : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UBTTask_ActivateAbility();

    // 要激活的技能标签
    UPROPERTY(EditAnywhere, Category = "Ability")
    FGameplayTag AbilityTag;

    // 是否等待技能执行完成
    UPROPERTY(EditAnywhere, Category = "Ability")
    bool bWaitForCompletion = true;

    // 超时时间（秒）
    UPROPERTY(EditAnywhere, Category = "Ability", meta = (EditCondition = "bWaitForCompletion"))
    float Timeout = 5.0f;

    //~ UBTTaskNode interface
    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
    virtual EBTNodeResult::Type AbortTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
    virtual void TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds) override;
    //~ End of UBTTaskNode interface

protected:
    struct FBTAbilityMemory
    {
        FGameplayAbilitySpecHandle ActivatedHandle;
        float ElapsedTime = 0.0f;
        bool bAbilityEnded = false;
    };

    void OnAbilityEnded(const FAbilityEndedData& AbilityEndedData, UBehaviorTreeComponent* OwnerComp);
};
```

```cpp
// BTTask_ActivateAbility.cpp
EBTNodeResult::Type UBTTask_ActivateAbility::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController)
    {
        return EBTNodeResult::Failed;
    }

    APawn* ControlledPawn = AIController->GetPawn();
    if (!ControlledPawn)
    {
        return EBTNodeResult::Failed;
    }

    // 获取 ASC
    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(ControlledPawn);
    if (!ASC)
    {
        return EBTNodeResult::Failed;
    }

    // 查找匹配技能标签的 Ability
    FGameplayAbilitySpec* Spec = ASC->FindAbilitySpecFromInputID(AbilityTag);
    if (!Spec)
    {
        // 尝试通过标签查找
        TArray<FGameplayAbilitySpec*> Specs;
        ASC->GetActivatableGameplayAbilitySpecsByAllMatchingTags(
            FGameplayTagContainer(AbilityTag), 
            Specs
        );

        if (Specs.Num() > 0)
        {
            Spec = Specs[0];
        }
        else
        {
            UE_LOG(LogBehaviorTree, Warning, TEXT("BTTask_ActivateAbility: No ability found with tag %s"), 
                   *AbilityTag.ToString());
            return EBTNodeResult::Failed;
        }
    }

    // 尝试激活技能
    if (ASC->TryActivateAbility(Spec->Handle))
    {
        FBTAbilityMemory* Memory = CastInstanceNodeMemory<FBTAbilityMemory>(NodeMemory);
        Memory->ActivatedHandle = Spec->Handle;
        Memory->ElapsedTime = 0.0f;
        Memory->bAbilityEnded = false;

        if (bWaitForCompletion)
        {
            // 绑定技能结束回调
            Spec->Ability->OnGameplayAbilityEnded.AddUObject(
                this, 
                &UBTTask_ActivateAbility::OnAbilityEnded, 
                &OwnerComp
            );

            return EBTNodeResult::InProgress;
        }
        else
        {
            // 不等待，立即返回成功
            return EBTNodeResult::Succeeded;
        }
    }

    return EBTNodeResult::Failed;
}

void UBTTask_ActivateAbility::TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds)
{
    Super::TickTask(OwnerComp, NodeMemory, DeltaSeconds);

    FBTAbilityMemory* Memory = CastInstanceNodeMemory<FBTAbilityMemory>(NodeMemory);
    Memory->ElapsedTime += DeltaSeconds;

    // 检查技能是否结束
    if (Memory->bAbilityEnded)
    {
        FinishLatentTask(OwnerComp, EBTNodeResult::Succeeded);
        return;
    }

    // 检查超时
    if (Memory->ElapsedTime >= Timeout)
    {
        UE_LOG(LogBehaviorTree, Warning, TEXT("BTTask_ActivateAbility: Ability %s timed out"), 
               *AbilityTag.ToString());
        FinishLatentTask(OwnerComp, EBTNodeResult::Failed);
    }
}

void UBTTask_ActivateAbility::OnAbilityEnded(const FAbilityEndedData& AbilityEndedData, UBehaviorTreeComponent* OwnerComp)
{
    if (uint8* NodeMemory = OwnerComp->GetNodeMemory(this, OwnerComp->FindInstanceContainingNode(this)))
    {
        FBTAbilityMemory* Memory = CastInstanceNodeMemory<FBTAbilityMemory>(NodeMemory);
        Memory->bAbilityEnded = true;
    }
}

EBTNodeResult::Type UBTTask_ActivateAbility::AbortTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    FBTAbilityMemory* Memory = CastInstanceNodeMemory<FBTAbilityMemory>(NodeMemory);

    // 如果技能还在执行，取消它
    if (Memory->ActivatedHandle.IsValid())
    {
        AAIController* AIController = OwnerComp.GetAIOwner();
        if (APawn* ControlledPawn = AIController->GetPawn())
        {
            if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(ControlledPawn))
            {
                ASC->CancelAbilityHandle(Memory->ActivatedHandle);
            }
        }
    }

    return EBTNodeResult::Aborted;
}
```

**使用示例（行为树编辑器）**:

```
[Selector: Choose Combat Action]
├── [Sequence: Melee Attack]
│   ├── [Decorator] Is Close To Enemy (<= 300 units)
│   └── [Task] Activate Ability (Tag: Ability.Melee.Slash)
├── [Sequence: Ranged Attack]
│   ├── [Decorator] Has Clear Line of Sight
│   └── [Task] Activate Ability (Tag: Ability.Weapon.Fire)
└── [Task] Move Closer To Enemy
```

### 4.3 自定义行为树服务：动态威胁评估

```cpp
// BTService_UpdateThreatLevel.h
UCLASS()
class UBTService_UpdateThreatLevel : public UBTService
{
    GENERATED_BODY()

public:
    UBTService_UpdateThreatLevel();

    // 黑板键：存储最大威胁目标
    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector HighestThreatEnemyKey;

    // 黑板键：存储威胁等级 (0-1)
    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector ThreatLevelKey;

protected:
    virtual void TickNode(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds) override;

private:
    float CalculateThreatScore(AActor* Enemy, APawn* SelfPawn);
};
```

```cpp
// BTService_UpdateThreatLevel.cpp
void UBTService_UpdateThreatLevel::TickNode(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds)
{
    Super::TickNode(OwnerComp, NodeMemory, DeltaSeconds);

    AAIController* AIController = OwnerComp.GetAIOwner();
    APawn* ControlledPawn = AIController ? AIController->GetPawn() : nullptr;

    if (!ControlledPawn)
    {
        return;
    }

    // 获取感知组件
    UAIPerceptionComponent* PerceptionComp = AIController->FindComponentByClass<UAIPerceptionComponent>();
    if (!PerceptionComp)
    {
        return;
    }

    // 获取所有感知到的敌人
    TArray<AActor*> PerceivedActors;
    PerceptionComp->GetCurrentlyPerceivedActors(UAISense_Sight::StaticClass(), PerceivedActors);

    AActor* HighestThreatEnemy = nullptr;
    float MaxThreatScore = 0.0f;

    for (AActor* Actor : PerceivedActors)
    {
        // 检查是否为敌对单位
        if (ULyraTeamSubsystem* TeamSubsystem = UWorld::GetSubsystem<ULyraTeamSubsystem>(GetWorld()))
        {
            int32 SelfTeam = TeamSubsystem->FindTeamFromObject(ControlledPawn);
            int32 TargetTeam = TeamSubsystem->FindTeamFromObject(Actor);

            if (SelfTeam == TargetTeam)
            {
                continue; // 跳过队友
            }
        }

        // 计算威胁分数
        float ThreatScore = CalculateThreatScore(Actor, ControlledPawn);

        if (ThreatScore > MaxThreatScore)
        {
            MaxThreatScore = ThreatScore;
            HighestThreatEnemy = Actor;
        }
    }

    // 更新黑板
    OwnerComp.GetBlackboardComponent()->SetValueAsObject(HighestThreatEnemyKey.SelectedKeyName, HighestThreatEnemy);
    OwnerComp.GetBlackboardComponent()->SetValueAsFloat(ThreatLevelKey.SelectedKeyName, MaxThreatScore);
}

float UBTService_UpdateThreatLevel::CalculateThreatScore(AActor* Enemy, APawn* SelfPawn)
{
    float ThreatScore = 0.0f;

    // 因素 1: 距离（越近威胁越高）
    float Distance = FVector::Dist(Enemy->GetActorLocation(), SelfPawn->GetActorLocation());
    float DistanceScore = FMath::Clamp(1.0f - (Distance / 3000.0f), 0.0f, 1.0f); // 30m 内威胁递减
    ThreatScore += DistanceScore * 0.4f;

    // 因素 2: 血量（血量越低越容易击杀）
    if (ULyraHealthComponent* HealthComp = ULyraHealthComponent::FindHealthComponent(Enemy))
    {
        float HealthPercent = HealthComp->GetHealth() / HealthComp->GetMaxHealth();
        float LowHealthScore = 1.0f - HealthPercent; // 血量越低分数越高
        ThreatScore += LowHealthScore * 0.3f;
    }

    // 因素 3: 视线角度（正面视野内威胁更高）
    FVector ToEnemy = (Enemy->GetActorLocation() - SelfPawn->GetActorLocation()).GetSafeNormal();
    float DotProduct = FVector::DotProduct(SelfPawn->GetActorForwardVector(), ToEnemy);
    float AngleScore = (DotProduct + 1.0f) / 2.0f; // 转换到 0-1 范围
    ThreatScore += AngleScore * 0.3f;

    return FMath::Clamp(ThreatScore, 0.0f, 1.0f);
}
```

**行为树集成**:

```
[Root Sequence]
├── [Service] Update Threat Level (每 0.5 秒)
│   └── 输出: BB_HighestThreatEnemy, BB_ThreatLevel
└── [Selector] Combat Strategy
    ├── [Sequence] Aggressive (当 ThreatLevel < 0.3)
    │   └── [Task] Rush Enemy
    ├── [Sequence] Cautious (当 0.3 <= ThreatLevel < 0.7)
    │   └── [Task] Strafe and Shoot
    └── [Sequence] Defensive (当 ThreatLevel >= 0.7)
        └── [Task] Find Cover
```

---

## 5. 环境查询系统（EQS）高级应用

### 5.1 EQS 基础概念

EQS (Environment Query System) 是虚幻引擎的空间推理系统,允许 AI 根据多维度评分选择最优位置:

**核心组件**:
- **Generators**: 生成候选位置点（网格、圆环、Actor 周围）
- **Tests**: 对每个点进行测试（距离、视线、掩体质量）
- **Context**: 查询上下文（Querier = AI 自己, Target = 敌人）

### 5.2 实战案例：智能掩体选择

创建 EQS Query `EQS_FindCover`:

```
Query: EQS_FindCover
├── Generator: Points in Circle
│   ├── Center: Context_Querier (AI 自己)
│   ├── Radius: 1500.0
│   ├── NumberOfPoints: 50
│   └── TraceData: 地面对齐
├── Test 1: Distance To (Context_Target)
│   ├── Scoring: Prefer Closer (距离适中)
│   ├── Min: 500.0 (最小安全距离)
│   └── Max: 2000.0
├── Test 2: Trace (视线遮挡)
│   ├── From: Generated Point
│   ├── To: Context_Target
│   ├── Scoring: Prefer Blocked (偏好无视线点)
│   └── Weight: 2.0 (高权重)
├── Test 3: Dot Product (角度检查)
│   ├── Line A: Querier → Target
│   ├── Line B: Generated Point → Target
│   ├── Scoring: Prefer Side Positions (侧翼更优)
│   └── Ideal Angle: 90 度
└── Test 4: Path Exists (可到达性)
    ├── To: Generated Point
    ├── Scoring: Binary (不可达直接淘汰)
    └── Weight: 10.0 (最高权重)
```

**视觉化评分结果**:

```
        [敌人]
          🎯
          |
    🟢🟢🔴🔴🔴   <- 前方：低分（视线暴露）
   🟢🟡🟡🔴🔴
  🟢🟡🟢🟡🔴
 🟢🟢🟡🟡🔴    <- 侧翼：高分（有掩体+角度好）
  [AI]
```

### 5.3 在行为树中使用 EQS

创建 `BTTask_RunEQSQuery`:

```cpp
// BTTask_RunEQSQuery.h
UCLASS()
class UBTTask_RunEQSQuery : public UBTTaskNode
{
    GENERATED_BODY()

public:
    // EQS 查询资产
    UPROPERTY(EditAnywhere, Category = "EQS")
    TObjectPtr<UEnvQuery> QueryTemplate;

    // 查询参数
    UPROPERTY(EditAnywhere, Category = "EQS")
    TArray<FEnvNamedValue> QueryParams;

    // 结果存储黑板键
    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector ResultLocationKey;

    //~ UBTTaskNode interface
    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
    //~ End of UBTTaskNode interface

private:
    void OnQueryFinished(TSharedPtr<FEnvQueryResult> Result, UBehaviorTreeComponent* OwnerComp);
};
```

```cpp
// BTTask_RunEQSQuery.cpp
EBTNodeResult::Type UBTTask_RunEQSQuery::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController || !QueryTemplate)
    {
        return EBTNodeResult::Failed;
    }

    // 配置查询请求
    FEnvQueryRequest QueryRequest(QueryTemplate, AIController);

    // 设置参数
    for (const FEnvNamedValue& Param : QueryParams)
    {
        QueryRequest.SetNamedParam(Param.ParamName, Param.Value);
    }

    // 执行异步查询
    QueryRequest.Execute(
        EEnvQueryRunMode::SingleResult, // 只返回最优结果
        FQueryFinishedSignature::CreateUObject(this, &UBTTask_RunEQSQuery::OnQueryFinished, &OwnerComp)
    );

    return EBTNodeResult::InProgress;
}

void UBTTask_RunEQSQuery::OnQueryFinished(TSharedPtr<FEnvQueryResult> Result, UBehaviorTreeComponent* OwnerComp)
{
    if (Result->IsSuccessful())
    {
        // 获取最优位置
        FVector BestLocation = Result->GetItemAsLocation(0);

        // 写入黑板
        OwnerComp->GetBlackboardComponent()->SetValueAsVector(ResultLocationKey.SelectedKeyName, BestLocation);

        FinishLatentTask(*OwnerComp, EBTNodeResult::Succeeded);
    }
    else
    {
        UE_LOG(LogBehaviorTree, Warning, TEXT("EQS Query failed: %s"), *QueryTemplate->GetName());
        FinishLatentTask(*OwnerComp, EBTNodeResult::Failed);
    }
}
```

**行为树集成示例**:

```
[Sequence: Take Cover]
├── [Task] Run EQS Query
│   ├── Query: EQS_FindCover
│   ├── Params: ["Enemy", BB_CurrentEnemy]
│   └── Result → BB_CoverLocation
├── [Task] Move To (BB_CoverLocation)
│   └── Acceptance Radius: 100.0
└── [Task] Face Enemy While In Cover
```

### 5.4 自定义 EQS Test：掩体质量评估

```cpp
// EnvQueryTest_CoverQuality.h
UCLASS()
class UEnvQueryTest_CoverQuality : public UEnvQueryTest
{
    GENERATED_BODY()

public:
    UEnvQueryTest_CoverQuality();

    // 评估高度（全身掩体 vs 半身掩体）
    UPROPERTY(EditDefaultsOnly, Category = "Cover")
    bool bPreferFullCover = true;

    // 掩体稳定性（移动物体扣分）
    UPROPERTY(EditDefaultsOnly, Category = "Cover")
    bool bCheckStability = true;

    //~ UEnvQueryTest interface
    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override;
    //~ End of UEnvQueryTest interface

private:
    float CalculateCoverQuality(const FVector& TestLocation, const FVector& ThreatLocation, UWorld* World) const;
};
```

```cpp
// EnvQueryTest_CoverQuality.cpp
void UEnvQueryTest_CoverQuality::RunTest(FEnvQueryInstance& QueryInstance) const
{
    UObject* QueryOwner = QueryInstance.Owner.Get();
    if (!QueryOwner)
    {
        return;
    }

    UWorld* World = GEngine->GetWorldFromContextObject(QueryOwner, EGetWorldErrorMode::LogAndReturnNull);
    if (!World)
    {
        return;
    }

    // 获取威胁位置（通常是敌人）
    TArray<FVector> ThreatLocations;
    QueryInstance.PrepareContext(UEnvQueryContext_Querier::StaticClass(), ThreatLocations);

    if (ThreatLocations.Num() == 0)
    {
        return;
    }

    FVector ThreatLocation = ThreatLocations[0];

    // 遍历所有生成的测试点
    for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
    {
        FVector TestLocation = GetItemLocation(QueryInstance, It.GetIndex());

        // 计算掩体质量分数
        float CoverQuality = CalculateCoverQuality(TestLocation, ThreatLocation, World);

        // 归一化到 0-1 范围
        It.SetScore(TestType, FilterType, CoverQuality, 0.0f, 1.0f);
    }
}

float UEnvQueryTest_CoverQuality::CalculateCoverQuality(const FVector& TestLocation, const FVector& ThreatLocation, UWorld* World) const
{
    float QualityScore = 0.0f;

    // 1. 检查视线遮挡高度
    FCollisionQueryParams TraceParams;
    TraceParams.bTraceComplex = true;

    // 测试三个高度的视线
    TArray<float> TestHeights = {50.0f, 100.0f, 180.0f}; // 脚/腰/头
    int32 BlockedCount = 0;

    for (float Height : TestHeights)
    {
        FVector StartLoc = TestLocation + FVector(0, 0, Height);
        FVector EndLoc = ThreatLocation + FVector(0, 0, Height);

        FHitResult HitResult;
        if (World->LineTraceSingleByChannel(HitResult, StartLoc, EndLoc, ECC_Visibility, TraceParams))
        {
            BlockedCount++;
        }
    }

    // 全身掩体（3/3）vs 半身掩体（2/3）
    float CoverRatio = static_cast<float>(BlockedCount) / TestHeights.Num();
    QualityScore += CoverRatio * 0.6f;

    // 2. 检查掩体稳定性（是否为移动物体）
    if (bCheckStability)
    {
        FHitResult GroundHit;
        if (World->LineTraceSingleByChannel(GroundHit, TestLocation + FVector(0, 0, 50), TestLocation - FVector(0, 0, 100), ECC_WorldStatic, TraceParams))
        {
            if (GroundHit.GetActor() && GroundHit.GetActor()->IsRootComponentMovable())
            {
                QualityScore -= 0.3f; // 移动物体扣分
            }
        }
    }

    // 3. 检查周围是否有额外障碍物（360度防护）
    int32 ProtectedAngles = 0;
    for (int32 Angle = 0; Angle < 360; Angle += 45)
    {
        FVector Direction = FRotator(0, Angle, 0).Vector();
        FVector CheckLoc = TestLocation + Direction * 200.0f + FVector(0, 0, 100);

        FHitResult HitResult;
        if (World->LineTraceSingleByChannel(HitResult, TestLocation + FVector(0, 0, 100), CheckLoc, ECC_Visibility, TraceParams))
        {
            ProtectedAngles++;
        }
    }

    float ProtectionRatio = static_cast<float>(ProtectedAngles) / 8.0f;
    QualityScore += ProtectionRatio * 0.3f;

    return FMath::Clamp(QualityScore, 0.0f, 1.0f);
}
```

---

## 6. AI 与 GAS 的深度整合

### 6.1 AI 技能决策系统

创建智能技能选择器 `ULyraAIAbilitySelector`:

```cpp
// LyraAIAbilitySelector.h
UCLASS(Blueprintable)
class ULyraAIAbilitySelector : public UObject
{
    GENERATED_BODY()

public:
    // 技能优先级配置
    USTRUCT(BlueprintType)
    struct FAbilityPriority
    {
        // 技能标签
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        FGameplayTag AbilityTag;

        // 基础优先级
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        float BasePriority = 1.0f;

        // 触发条件
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        FGameplayTagQuery RequiredTags;

        // 冷却时间容忍度（0-1，越高越能接受还在冷却的技能）
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        float CooldownTolerance = 0.0f;
    };

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AI")
    TArray<FAbilityPriority> AbilityPriorities;

    // 选择最佳技能
    UFUNCTION(BlueprintCallable, Category = "AI")
    FGameplayTag SelectBestAbility(UAbilitySystemComponent* ASC, const FGameplayTagContainer& CurrentContext);

private:
    float EvaluateAbilityScore(const FAbilityPriority& Priority, UAbilitySystemComponent* ASC, const FGameplayTagContainer& Context);
};
```

```cpp
// LyraAIAbilitySelector.cpp
FGameplayTag ULyraAIAbilitySelector::SelectBestAbility(UAbilitySystemComponent* ASC, const FGameplayTagContainer& CurrentContext)
{
    if (!ASC)
    {
        return FGameplayTag::EmptyTag;
    }

    FGameplayTag BestAbility;
    float BestScore = -1.0f;

    for (const FAbilityPriority& Priority : AbilityPriorities)
    {
        // 检查是否满足触发条件
        if (!Priority.RequiredTags.Matches(CurrentContext))
        {
            continue;
        }

        // 评估技能得分
        float Score = EvaluateAbilityScore(Priority, ASC, CurrentContext);

        if (Score > BestScore)
        {
            BestScore = Score;
            BestAbility = Priority.AbilityTag;
        }
    }

    return BestAbility;
}

float ULyraAIAbilitySelector::EvaluateAbilityScore(const FAbilityPriority& Priority, UAbilitySystemComponent* ASC, const FGameplayTagContainer& Context)
{
    float Score = Priority.BasePriority;

    // 查找技能
    TArray<FGameplayAbilitySpec*> Specs;
    ASC->GetActivatableGameplayAbilitySpecsByAllMatchingTags(FGameplayTagContainer(Priority.AbilityTag), Specs);

    if (Specs.Num() == 0)
    {
        return -1.0f; // 技能不存在
    }

    FGameplayAbilitySpec* Spec = Specs[0];

    // 检查是否可激活
    if (!ASC->CanActivateAbility(Spec->Handle))
    {
        Score *= 0.1f; // 大幅降低不可激活技能的分数
    }

    // 检查冷却时间
    if (const UGameplayAbility* Ability = Spec->Ability)
    {
        FGameplayTagContainer CooldownTags;
        Ability->GetCooldownTags(&CooldownTags);

        if (ASC->HasAnyMatchingGameplayTags(CooldownTags))
        {
            // 计算剩余冷却时间
            float RemainingCooldown, CooldownDuration;
            ASC->GetCooldownRemainingForTag(CooldownTags, RemainingCooldown, CooldownDuration);

            float CooldownRatio = RemainingCooldown / CooldownDuration;

            // 根据容忍度调整分数
            Score *= (1.0f - CooldownRatio * (1.0f - Priority.CooldownTolerance));
        }
    }

    // 上下文加成（例如：血量低时治疗技能分数更高）
    if (Context.HasTag(FGameplayTag::RequestGameplayTag("State.LowHealth")))
    {
        if (Priority.AbilityTag.MatchesTag(FGameplayTag::RequestGameplayTag("Ability.Heal")))
        {
            Score *= 2.0f; // 双倍优先级
        }
    }

    return Score;
}
```

**在行为树中使用**:

```cpp
// BTTask_SelectAndUseAbility.h
UCLASS()
class UBTTask_SelectAndUseAbility : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "AI")
    TObjectPtr<ULyraAIAbilitySelector> AbilitySelector;

    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector ContextTagsKey; // 存储当前上下文标签

    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
};
```

```cpp
// BTTask_SelectAndUseAbility.cpp
EBTNodeResult::Type UBTTask_SelectAndUseAbility::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    APawn* ControlledPawn = AIController ? AIController->GetPawn() : nullptr;

    if (!ControlledPawn || !AbilitySelector)
    {
        return EBTNodeResult::Failed;
    }

    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(ControlledPawn);
    if (!ASC)
    {
        return EBTNodeResult::Failed;
    }

    // 获取当前上下文标签
    FGameplayTagContainer ContextTags;
    if (UBlackboardComponent* BlackboardComp = OwnerComp.GetBlackboardComponent())
    {
        // 从黑板读取上下文（需要自定义 BlackboardKeyType）
        // 这里简化为直接从 ASC 获取
        ASC->GetOwnedGameplayTags(ContextTags);
    }

    // 选择最佳技能
    FGameplayTag SelectedAbility = AbilitySelector->SelectBestAbility(ASC, ContextTags);

    if (SelectedAbility.IsValid())
    {
        // 激活技能
        TArray<FGameplayAbilitySpec*> Specs;
        ASC->GetActivatableGameplayAbilitySpecsByAllMatchingTags(FGameplayTagContainer(SelectedAbility), Specs);

        if (Specs.Num() > 0 && ASC->TryActivateAbility(Specs[0]->Handle))
        {
            return EBTNodeResult::Succeeded;
        }
    }

    return EBTNodeResult::Failed;
}
```

### 6.2 AI 属性监控与响应

创建组件 `ULyraAIHealthMonitorComponent`:

```cpp
// LyraAIHealthMonitorComponent.h
UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class ULyraAIHealthMonitorComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    ULyraAIHealthMonitorComponent();

    // 低血量阈值
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
    float LowHealthThreshold = 0.3f;

    // 危急血量阈值
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
    float CriticalHealthThreshold = 0.15f;

    // 当血量变化时的回调
    UPROPERTY(BlueprintAssignable, Category = "Health")
    FHealthChangedSignature OnHealthStateChanged;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(FHealthChangedSignature, 
        float, HealthPercent, 
        EHealthState, OldState, 
        EHealthState, NewState);

    UENUM(BlueprintType)
    enum class EHealthState : uint8
    {
        Healthy,
        LowHealth,
        Critical
    };

protected:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

private:
    void OnHealthChanged(ULyraHealthComponent* HealthComp, float OldValue, float NewValue, AActor* Instigator);

    UPROPERTY()
    EHealthState CurrentHealthState = EHealthState::Healthy;
};
```

```cpp
// LyraAIHealthMonitorComponent.cpp
void ULyraAIHealthMonitorComponent::BeginPlay()
{
    Super::BeginPlay();

    // 查找并绑定健康组件
    if (AActor* Owner = GetOwner())
    {
        if (ULyraHealthComponent* HealthComp = ULyraHealthComponent::FindHealthComponent(Owner))
        {
            HealthComp->OnHealthChanged.AddDynamic(this, &ULyraAIHealthMonitorComponent::OnHealthChanged);

            // 初始检查
            float CurrentHealth = HealthComp->GetHealth();
            float MaxHealth = HealthComp->GetMaxHealth();
            float HealthPercent = CurrentHealth / MaxHealth;

            if (HealthPercent <= CriticalHealthThreshold)
            {
                CurrentHealthState = EHealthState::Critical;
            }
            else if (HealthPercent <= LowHealthThreshold)
            {
                CurrentHealthState = EHealthState::LowHealth;
            }
        }
    }
}

void ULyraAIHealthMonitorComponent::OnHealthChanged(ULyraHealthComponent* HealthComp, float OldValue, float NewValue, AActor* Instigator)
{
    float MaxHealth = HealthComp->GetMaxHealth();
    float HealthPercent = NewValue / MaxHealth;

    EHealthState NewState = EHealthState::Healthy;

    if (HealthPercent <= CriticalHealthThreshold)
    {
        NewState = EHealthState::Critical;
    }
    else if (HealthPercent <= LowHealthThreshold)
    {
        NewState = EHealthState::LowHealth;
    }

    // 如果状态改变，触发回调
    if (NewState != CurrentHealthState)
    {
        EHealthState OldState = CurrentHealthState;
        CurrentHealthState = NewState;

        OnHealthStateChanged.Broadcast(HealthPercent, OldState, NewState);

        // 更新黑板（用于行为树决策）
        if (AAIController* AIController = Cast<AAIController>(GetOwner()->GetInstigatorController()))
        {
            if (UBlackboardComponent* BlackboardComp = AIController->GetBlackboardComponent())
            {
                // 添加/移除状态标签
                switch (NewState)
                {
                case EHealthState::Critical:
                    BlackboardComp->SetValueAsBool("IsCriticalHealth", true);
                    BlackboardComp->SetValueAsBool("IsLowHealth", true);
                    break;
                case EHealthState::LowHealth:
                    BlackboardComp->SetValueAsBool("IsCriticalHealth", false);
                    BlackboardComp->SetValueAsBool("IsLowHealth", true);
                    break;
                default:
                    BlackboardComp->SetValueAsBool("IsCriticalHealth", false);
                    BlackboardComp->SetValueAsBool("IsLowHealth", false);
                    break;
                }
            }
        }
    }
}
```

**行为树响应低血量**:

```
[Root Selector]
├── [Sequence: Emergency Retreat] (优先级最高)
│   ├── [Decorator] Blackboard Check: IsCriticalHealth == true
│   ├── [Task] Stop Combat
│   ├── [Task] Find Safe Location (EQS: 远离所有敌人)
│   ├── [Task] Move To Safe Location
│   └── [Task] Use Heal Ability (如果有)
├── [Sequence: Cautious Combat]
│   ├── [Decorator] Blackboard Check: IsLowHealth == true
│   ├── [Task] Select And Use Ability (偏好防御/治疗技能)
│   └── [Task] Keep Distance From Enemy
└── [Sequence: Normal Combat]
    └── [Task] Select And Use Ability (正常策略)
```

---

## 7. 多人游戏中的 AI 网络优化

### 7.1 AI 网络复制策略

Lyra 的 Bot 在网络游戏中需要特殊处理:

```cpp
// LyraBotController.h
UCLASS()
class ALyraBotController : public AModularAIController
{
    GENERATED_BODY()

public:
    ALyraBotController(const FObjectInitializer& ObjectInitializer);

    //~ AAIController interface
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    //~ End of AAIController interface

protected:
    // 只在服务器执行行为树
    virtual void BeginPlay() override;

private:
    // 当前 AI 状态（复制给客户端用于表现）
    UPROPERTY(Replicated)
    EBotVisualState VisualState;

    UENUM()
    enum class EBotVisualState : uint8
    {
        Idle,
        Patrolling,
        Investigating,
        Combating,
        Retreating
    };
};
```

```cpp
// LyraBotController.cpp
void ALyraBotController::BeginPlay()
{
    Super::BeginPlay();

    // 只在服务器运行行为树
    if (HasAuthority())
    {
        if (BehaviorTree)
        {
            RunBehaviorTree(BehaviorTree);
        }
    }
}

void ALyraBotController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 只复制视觉状态，不复制完整行为树数据
    DOREPLIFETIME(ALyraBotController, VisualState);
}
```

### 7.2 客户端 AI 表现层

为了减少网络开销，客户端使用简化的表现逻辑:

```cpp
// LyraBotAnimInstance.h - Bot 动画蓝图
UCLASS()
class ULyraBotAnimInstance : public ULyraAnimInstance
{
    GENERATED_BODY()

public:
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;

protected:
    // 从复制的 VisualState 推断动画状态
    UPROPERTY(BlueprintReadOnly, Category = "Animation")
    bool bIsInCombat;

    UPROPERTY(BlueprintReadOnly, Category = "Animation")
    bool bIsRetreating;

private:
    void UpdateCombatState();
};
```

```cpp
// LyraBotAnimInstance.cpp
void ULyraBotAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);

    UpdateCombatState();
}

void ULyraBotAnimInstance::UpdateCombatState()
{
    APawn* OwningPawn = TryGetPawnOwner();
    if (!OwningPawn)
    {
        return;
    }

    // 获取 AI 控制器的状态
    if (ALyraBotController* BotController = Cast<ALyraBotController>(OwningPawn->GetController()))
    {
        EBotVisualState State = BotController->GetVisualState();

        bIsInCombat = (State == EBotVisualState::Combating || State == EBotVisualState::Investigating);
        bIsRetreating = (State == EBotVisualState::Retreating);
    }
    else
    {
        // 如果无法获取控制器（客户端预测），使用启发式判断
        if (ULyraHealthComponent* HealthComp = ULyraHealthComponent::FindHealthComponent(OwningPawn))
        {
            float HealthPercent = HealthComp->GetHealth() / HealthComp->GetMaxHealth();
            bIsRetreating = (HealthPercent < 0.2f);
        }

        // 检查是否持有武器
        if (ULyraEquipmentManagerComponent* EquipmentComp = OwningPawn->FindComponentByClass<ULyraEquipmentManagerComponent>())
        {
            bIsInCombat = (EquipmentComp->GetFirstInstanceOfType(ULyraWeaponInstance::StaticClass()) != nullptr);
        }
    }
}
```

### 7.3 AI 感知优化

服务器端感知系统的性能优化:

```cpp
// LyraAIPerceptionComponent.h
UCLASS()
class ULyraAIPerceptionComponent : public UAIPerceptionComponent
{
    GENERATED_BODY()

public:
    ULyraAIPerceptionComponent();

    // 动态调整感知频率
    UPROPERTY(EditAnywhere, Category = "Perception")
    float CombatPerceptionInterval = 0.1f; // 战斗中频繁更新

    UPROPERTY(EditAnywhere, Category = "Perception")
    float IdlePerceptionInterval = 0.5f; // 闲置时降低频率

    void SetCombatMode(bool bInCombat);

protected:
    virtual void BeginPlay() override;
};
```

```cpp
// LyraAIPerceptionComponent.cpp
void ULyraAIPerceptionComponent::BeginPlay()
{
    Super::BeginPlay();

    // 默认闲置模式
    SetPerceptionListenerUpdateInterval(IdlePerceptionInterval);
}

void ULyraAIPerceptionComponent::SetCombatMode(bool bInCombat)
{
    float NewInterval = bInCombat ? CombatPerceptionInterval : IdlePerceptionInterval;
    SetPerceptionListenerUpdateInterval(NewInterval);

    // 如果进入战斗，立即强制更新一次
    if (bInCombat)
    {
        RequestStimuliListenerUpdate();
    }
}
```

---

## 8. 完整实战项目：战术射击 AI

### 8.1 项目目标

创建一个智能战术 Bot，具备以下能力:
- 掩体战术：动态选择和使用掩体
- 团队协作：呼叫队友支援
- 武器管理：根据距离切换武器
- 战术撤退：血量低时智能脱离战斗

### 8.2 黑板设计

```cpp
// BB_TacticalBot.uasset (蓝图黑板)
Blackboard Keys:
- SelfActor (Object: AActor) - AI 自己
- CurrentEnemy (Object: AActor) - 当前目标敌人
- CoverLocation (Vector) - 掩体位置
- LastKnownEnemyLocation (Vector) - 敌人最后已知位置
- ThreatLevel (Float) - 威胁等级 0-1
- IsInCover (Bool) - 是否在掩体中
- IsLowHealth (Bool) - 是否低血量
- IsCriticalHealth (Bool) - 是否危急血量
- PreferredWeapon (Object: ULyraWeaponInstance) - 首选武器
- TeamMates (Array<AActor>) - 队友列表
- PatrolRoute (Object: ATargetPoint) - 巡逻路线
```

### 8.3 行为树完整结构

```
BT_TacticalShooterBot
├── [Root Sequence]
│   ├── [Service] Update Perception (0.2s)
│   │   └── 更新 CurrentEnemy, ThreatLevel, TeamMates
│   ├── [Service] Update Health State (0.5s)
│   │   └── 更新 IsLowHealth, IsCriticalHealth
│   └── [Selector] Main Strategy
│       ├── [Sequence] 1. Emergency Retreat ⚠️
│       │   ├── [Decorator] IsCriticalHealth == true
│       │   ├── [Task] Cancel All Abilities
│       │   ├── [Task] Call For Help (团队通信)
│       │   ├── [Task] Find Safe Retreat Location (EQS)
│       │   ├── [Task] Sprint To Safety (使用冲刺技能)
│       │   └── [Task] Use Heal Item (如果有)
│       ├── [Sequence] 2. Tactical Combat 🎯
│       │   ├── [Decorator] Has CurrentEnemy
│       │   ├── [Selector] Weapon Management
│       │   │   ├── [Sequence] Switch to Ranged (距离 > 10m)
│       │   │   │   ├── [Task] Select Ranged Weapon
│       │   │   │   └── [Task] Equip Weapon
│       │   │   └── [Sequence] Switch to Melee (距离 <= 10m)
│       │   │       ├── [Task] Select Melee Weapon
│       │   │       └── [Task] Equip Weapon
│       │   ├── [Selector] Cover Tactics
│       │   │   ├── [Sequence] Move to Better Cover
│       │   │   │   ├── [Decorator] IsInCover == false OR ThreatLevel > 0.7
│       │   │   │   ├── [Task] Find Cover (EQS_FindCover)
│       │   │   │   └── [Task] Move To Cover (Tactical Sprint)
│       │   │   └── [Sequence] Fight from Cover
│       │   │       ├── [Decorator] IsInCover == true
│       │   │       ├── [Task] Peek and Shoot (间歇性射击)
│       │   │       └── [Task] Suppressive Fire (队友进攻时掩护)
│       │   └── [Sequence] Advanced Combat
│       │       ├── [Task] Check Flank Opportunity (侧翼机会)
│       │       ├── [Task] Throw Grenade (如果有)
│       │       └── [Task] Coordinate with Team (团队配合)
│       ├── [Sequence] 3. Investigation 🔍
│       │   ├── [Decorator] Has LastKnownEnemyLocation AND No CurrentEnemy
│       │   ├── [Task] Move to Last Known Position (谨慎接近)
│       │   ├── [Task] Search Area (查看周围)
│       │   └── [Task] Wait and Listen (3 秒)
│       └── [Sequence] 4. Patrol 🚶
│           ├── [Task] Select Next Patrol Point
│           ├── [Task] Move To Patrol Point
│           ├── [Task] Look Around (随机转身)
│           └── [Wait] 2-5 秒随机延迟
```

### 8.4 核心任务实现

#### 8.4.1 战术冲刺到掩体

```cpp
// BTTask_TacticalSprintToCover.h
UCLASS()
class UBTTask_TacticalSprintToCover : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector CoverLocationKey;

    UPROPERTY(EditAnywhere, Category = "Movement")
    bool bUseSerpentinePattern = true; // 之字形跑动躲避子弹

    UPROPERTY(EditAnywhere, Category = "Movement")
    float SerpentineAmplitude = 200.0f; // 左右摆动幅度

    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
    virtual void TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds) override;

private:
    struct FTaskMemory
    {
        FVector TargetLocation;
        float TimeElapsed = 0.0f;
        bool bSprintActivated = false;
    };
};
```

```cpp
// BTTask_TacticalSprintToCover.cpp
EBTNodeResult::Type UBTTask_TacticalSprintToCover::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    APawn* ControlledPawn = AIController ? AIController->GetPawn() : nullptr;

    if (!ControlledPawn)
    {
        return EBTNodeResult::Failed;
    }

    // 获取目标掩体位置
    FVector CoverLocation = OwnerComp.GetBlackboardComponent()->GetValueAsVector(CoverLocationKey.SelectedKeyName);

    FTaskMemory* Memory = CastInstanceNodeMemory<FTaskMemory>(NodeMemory);
    Memory->TargetLocation = CoverLocation;
    Memory->TimeElapsed = 0.0f;
    Memory->bSprintActivated = false;

    // 激活冲刺技能
    if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(ControlledPawn))
    {
        FGameplayTag SprintTag = FGameplayTag::RequestGameplayTag("Ability.Movement.Sprint");
        TArray<FGameplayAbilitySpec*> Specs;
        ASC->GetActivatableGameplayAbilitySpecsByAllMatchingTags(FGameplayTagContainer(SprintTag), Specs);

        if (Specs.Num() > 0 && ASC->TryActivateAbility(Specs[0]->Handle))
        {
            Memory->bSprintActivated = true;
        }
    }

    // 开始移动
    EPathFollowingRequestResult::Type MoveResult = AIController->MoveToLocation(
        CoverLocation, 
        50.0f, // 接受半径
        true, // 停止到达
        bUseSerpentinePattern, // 使用自定义移动
        false, 
        true, 
        nullptr, 
        true
    );

    return (MoveResult == EPathFollowingRequestResult::RequestSuccessful) ? 
        EBTNodeResult::InProgress : EBTNodeResult::Failed;
}

void UBTTask_TacticalSprintToCover::TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds)
{
    Super::TickTask(OwnerComp, NodeMemory, DeltaSeconds);

    AAIController* AIController = OwnerComp.GetAIOwner();
    APawn* ControlledPawn = AIController->GetPawn();

    if (!ControlledPawn)
    {
        FinishLatentTask(OwnerComp, EBTNodeResult::Failed);
        return;
    }

    FTaskMemory* Memory = CastInstanceNodeMemory<FTaskMemory>(NodeMemory);
    Memory->TimeElapsed += DeltaSeconds;

    // 添加之字形运动
    if (bUseSerpentinePattern)
    {
        float Frequency = 2.0f; // 每秒 2 次摆动
        float Offset = FMath::Sin(Memory->TimeElapsed * Frequency * PI) * SerpentineAmplitude;

        FVector CurrentLocation = ControlledPawn->GetActorLocation();
        FVector ToTarget = (Memory->TargetLocation - CurrentLocation).GetSafeNormal2D();
        FVector RightVector = FVector::CrossProduct(ToTarget, FVector::UpVector);

        FVector AdjustedTarget = Memory->TargetLocation + RightVector * Offset;
        AIController->SetFocalPoint(AdjustedTarget);
    }

    // 检查是否到达
    if (FVector::Dist(ControlledPawn->GetActorLocation(), Memory->TargetLocation) < 100.0f)
    {
        // 停止冲刺
        if (Memory->bSprintActivated)
        {
            if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(ControlledPawn))
            {
                ASC->CancelAbilities(&FGameplayTagContainer(FGameplayTag::RequestGameplayTag("Ability.Movement.Sprint")));
            }
        }

        // 标记在掩体中
        OwnerComp.GetBlackboardComponent()->SetValueAsBool("IsInCover", true);

        FinishLatentTask(OwnerComp, EBTNodeResult::Succeeded);
    }
}
```

#### 8.4.2 团队支援呼叫

```cpp
// BTTask_CallForHelp.h
UCLASS()
class UBTTask_CallForHelp : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Team")
    float HelpRadius = 2000.0f; // 呼救半径

    UPROPERTY(EditAnywhere, Category = "Team")
    FGameplayTag SupportRequestTag; // 支援请求标签

    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
};
```

```cpp
// BTTask_CallForHelp.cpp
EBTNodeResult::Type UBTTask_CallForHelp::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    APawn* ControlledPawn = AIController ? AIController->GetPawn() : nullptr;

    if (!ControlledPawn)
    {
        return EBTNodeResult::Failed;
    }

    // 获取队伍系统
    ULyraTeamSubsystem* TeamSubsystem = UWorld::GetSubsystem<ULyraTeamSubsystem>(GetWorld());
    if (!TeamSubsystem)
    {
        return EBTNodeResult::Failed;
    }

    int32 MyTeam = TeamSubsystem->FindTeamFromObject(ControlledPawn);

    // 查找附近队友
    TArray<AActor*> NearbyAllies;
    TArray<AActor*> AllPawns;
    UGameplayStatics::GetAllActorsOfClass(GetWorld(), APawn::StaticClass(), AllPawns);

    for (AActor* Actor : AllPawns)
    {
        if (Actor == ControlledPawn)
        {
            continue;
        }

        int32 ActorTeam = TeamSubsystem->FindTeamFromObject(Actor);
        if (ActorTeam == MyTeam)
        {
            float Distance = FVector::Dist(Actor->GetActorLocation(), ControlledPawn->GetActorLocation());
            if (Distance <= HelpRadius)
            {
                NearbyAllies.Add(Actor);
            }
        }
    }

    // 向队友发送支援请求
    for (AActor* Ally : NearbyAllies)
    {
        if (AAIController* AllyController = Cast<AAIController>(Cast<APawn>(Ally)->GetController()))
        {
            if (UBlackboardComponent* AllyBlackboard = AllyController->GetBlackboardComponent())
            {
                // 设置队友黑板的支援目标
                AllyBlackboard->SetValueAsObject("AllyNeedingHelp", ControlledPawn);
                AllyBlackboard->SetValueAsVector("HelpLocation", ControlledPawn->GetActorLocation());
                AllyBlackboard->SetValueAsBool("SupportRequested", true);
            }
        }
    }

    // 播放呼救动画/音效
    if (ULyraAbilitySystemComponent* ASC = Cast<ULyraAbilitySystemComponent>(
        UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(ControlledPawn)))
    {
        FGameplayCueParameters CueParams;
        CueParams.Location = ControlledPawn->GetActorLocation();
        ASC->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Voice.CallForHelp"), CueParams);
    }

    UE_LOG(LogBehaviorTree, Log, TEXT("%s called for help, notified %d allies"), 
           *ControlledPawn->GetName(), NearbyAllies.Num());

    return EBTNodeResult::Succeeded;
}
```

### 8.5 EQS 查询：高级掩体选择

```
EQS_AdvancedCover
├── Generator: Grid (2D)
│   ├── GridSize: 50 x 50
│   ├── SpaceBetween: 100.0
│   ├── Center: Context_Querier
│   └── GenerateAround: Context_Querier (半径 1500)
├── Test 1: Trace (掩体检测)
│   ├── From: Generated Point + (0, 0, 100) [腰部高度]
│   ├── To: Context_Enemy + (0, 0, 100)
│   ├── Test Purpose: Filter Only (淘汰无掩体点)
│   └── Condition: Blocked
├── Test 2: Distance To Context_Enemy
│   ├── Test Purpose: Score
│   ├── Scoring Equation: Inverse Linear (距离适中更优)
│   ├── Clamp Min: 500.0
│   ├── Clamp Max: 1500.0
│   └── Weight: 1.0
├── Test 3: Dot Product (侧翼优势)
│   ├── Line A: Querier Forward Vector
│   ├── Line B: Generated Point → Enemy
│   ├── Test Mode: Dot 3D
│   ├── Absolute Value: false
│   ├── Scoring: Prefer -0.5 to 0.5 (侧面)
│   └── Weight: 1.5
├── Test 4: Distance To Context_Querier
│   ├── Test Purpose: Score
│   ├── Scoring Equation: Linear (偏好较近的掩体)
│   ├── Clamp Max: 800.0
│   └── Weight: 0.8
├── Test 5: Path Exists
│   ├── Context: Context_Querier
│   ├── Path To: Generated Point
│   ├── Test Purpose: Filter Only
│   └── Filter Type: Minimum (不可达直接淘汰)
├── Test 6: Gameplay Tag Match (高级：掩体质量标签)
│   ├── Tagged Actors: 查找带 "Cover.HighQuality" 标签的 Actor
│   ├── Test Mode: Overlap
│   ├── Scoring: Prefer Match
│   └── Weight: 2.0
└── Test 7: Team Proximity (团队协同)
    ├── Context: Custom Context_TeamMates
    ├── Distance To: Closest TeamMate
    ├── Scoring: Prefer Closer (靠近队友)
    ├── Clamp Max: 1000.0
    └── Weight: 0.5
```

### 8.6 测试与调试

#### 开启 AI 调试可视化

```cpp
// LyraBotCheats.cpp - 开发作弊命令
void ULyraBotCheats::ShowAIDebug()
{
    if (AAIController* AIController = Cast<AAIController>(GetOuterAPlayerController()))
    {
        // 显示行为树调试
        AIController->GetBrainComponent()->StartLogic();
        
        // 显示 EQS 调试
        UEnvQueryManager* EQSManager = UEnvQueryManager::GetCurrent(GetWorld());
        if (EQSManager)
        {
            EQSManager->SetAllowTimeSlicing(false); // 禁用时间切片以便实时查看
        }

        // 显示感知调试
        if (UAIPerceptionComponent* PerceptionComp = AIController->FindComponentByClass<UAIPerceptionComponent>())
        {
            PerceptionComp->SetSenseEnabled(UAISense_Sight::StaticClass(), true);
        }

        // 启用调试绘制
        UKismetSystemLibrary::ExecuteConsoleCommand(GetWorld(), TEXT("ai.debug.show Perception"));
        UKismetSystemLibrary::ExecuteConsoleCommand(GetWorld(), TEXT("ai.debug.show EQS"));
        UKismetSystemLibrary::ExecuteConsoleCommand(GetWorld(), TEXT("showdebug AI"));
    }
}
```

**控制台命令**:
```bash
# 显示所有 AI 调试信息
showdebug AI

# 显示行为树执行流程
ai.debug.show BehaviorTree

# 显示 EQS 查询结果（3D 可视化）
ai.debug.show EQS

# 显示感知系统（视锥、听觉范围）
ai.debug.show Perception

# 显示导航网格
show Navigation
```

---

## 9. 性能优化与最佳实践

### 9.1 AI 性能分析

```cpp
// 使用 Insights 分析 AI 开销
TRACE_CPUPROFILER_EVENT_SCOPE(LyraBot_BehaviorTreeTick);
TRACE_BOOKMARK(TEXT("Bot %s - Combat Decision"), *GetName());
```

**常见瓶颈**:
- EQS 查询过于复杂（>100 个测试点）
- 感知组件更新频率过高（<0.1s）
- 行为树服务重复执行昂贵操作
- 多个 Bot 同时执行相同 EQS 查询

### 9.2 优化检查清单

| 优化项 | 说明 | 推荐值 |
|--------|------|--------|
| **EQS 查询间隔** | 避免每帧执行 | 0.5-2.0 秒 |
| **感知更新频率** | 根据游戏状态动态调整 | 闲置 0.5s，战斗 0.1s |
| **行为树服务间隔** | Service 节点的 Interval | 0.2-0.5 秒 |
| **导航网格复杂度** | 减少不必要的精细度 | Cell Size 19-25 |
| **Bot 视距** | 限制感知范围 | 3000-5000 单位 |
| **同时激活 Bot 数量** | 服务器负载均衡 | <=32 (根据硬件) |

### 9.3 内存优化

```cpp
// 使用对象池复用 Bot Controller
class FLyraBotControllerPool
{
public:
    AAIController* AcquireController(UWorld* World, TSubclassOf<AAIController> ControllerClass)
    {
        // 从池中获取或新建
        if (AvailableControllers.Num() > 0)
        {
            AAIController* Controller = AvailableControllers.Pop();
            Controller->SetActorTickEnabled(true);
            return Controller;
        }

        // 池为空，创建新控制器
        FActorSpawnParameters SpawnInfo;
        SpawnInfo.ObjectFlags |= RF_Transient;
        return World->SpawnActor<AAIController>(ControllerClass, SpawnInfo);
    }

    void ReleaseController(AAIController* Controller)
    {
        if (Controller)
        {
            // 清理状态
            Controller->UnPossess();
            Controller->SetActorTickEnabled(false);
            Controller->SetActorLocation(FVector(0, 0, -10000)); // 移到远处

            // 放回池
            AvailableControllers.Add(Controller);
        }
    }

private:
    TArray<AAIController*> AvailableControllers;
};
```

---

## 10. 总结与扩展方向

### 10.1 本章关键要点

1. **模块化设计**: ModularAIController + Game Feature Actions 实现热插拔 AI 模块
2. **GAS 整合**: AI 通过 Gameplay Abilities 执行技能，与玩家共享相同系统
3. **行为树 + EQS**: 结合使用实现复杂战术决策和空间推理
4. **网络优化**: 服务器执行完整 AI，客户端仅复制视觉状态

### 10.2 进阶学习方向

#### 10.2.1 机器学习 AI

集成 UE5 的 Learning Agents 插件:
```cpp
// 使用强化学习训练 Bot
#include "LearningAgentsManager.h"
#include "LearningAgentsNeuralNetwork.h"

UCLASS()
class ULyraLearningBotManager : public ULearningAgentsManager
{
    GENERATED_BODY()

    // 定义观察空间（敌人位置、血量、弹药等）
    virtual void MakeObservations() override;

    // 定义动作空间（移动、射击、切换武器）
    virtual void MakeActions(const FLearningAgentsActionObject& Actions) override;

    // 定义奖励函数（击杀+10，死亡-5，命中+0.1）
    virtual float CalculateReward() override;
};
```

#### 10.2.2 分层任务网络（HTN）

```cpp
// 实现 HTN Planner 替代传统行为树
// 适用于更复杂的战略决策（如 RTS 游戏）
class FLyraHTNPlanner
{
public:
    // 分解高层任务到具体动作
    // 例如: "攻占据点" → ["清理敌人", "占领旗帜", "防守"]
    TArray<FHTNTask> DecomposeTask(const FHTNGoal& Goal);
};
```

#### 10.2.3 Utility AI

```cpp
// 使用效用函数选择行为
struct FUtilityAction
{
    FGameplayTag ActionTag;
    TFunction<float(const FAIContext&)> UtilityFunction;
};

// 示例：根据多因素计算"治疗"行为的效用值
float CalculateHealUtility(const FAIContext& Context)
{
    float HealthFactor = 1.0f - (Context.Health / Context.MaxHealth); // 血量越低越高
    float SafetyFactor = Context.IsInCover ? 1.0f : 0.3f; // 在掩体中更安全
    float CooldownFactor = Context.HealCooldownRemaining > 0 ? 0.0f : 1.0f;

    return HealthFactor * SafetyFactor * CooldownFactor * 100.0f;
}
```

### 10.3 参考资源

- **官方文档**: [Behavior Tree Quick Start](https://docs.unrealengine.com/5.3/en-US/behavior-tree-quick-start-guide-in-unreal-engine/)
- **EQS 教程**: [Environment Query System](https://docs.unrealengine.com/5.3/en-US/environment-query-system-in-unreal-engine/)
- **Lyra 源码**: `LyraBotCreationComponent.cpp`, `ModularAIController.cpp`
- **社区插件**: 
  - [HTN Planner](https://www.unrealengine.com/marketplace/en-US/product/htn-planner)
  - [Smart AI](https://www.unrealengine.com/marketplace/en-US/product/smart-ai-combat-system)

---

## 📚 下一步

下一篇文章将探讨 **Replay 系统与观战模式**，学习如何录制回放、实现 eSports 级观战功能。

继续学习: [26. Replay 系统：录像回放功能](./26-replay-system.md)

---

**相关文章**:
- [06. GAS 入门：Gameplay Ability System 基础](../02-core-systems/06-gas-basics.md)
- [12. 队伍系统与阵营管理](../02-core-systems/12-team-system.md)
- [20. 性能优化：Profiling 与调优](../04-network-performance/20-performance-optimization.md)
