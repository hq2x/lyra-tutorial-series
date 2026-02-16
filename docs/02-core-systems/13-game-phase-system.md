# UE5 Lyra 游戏阶段管理系统（Game Phase System）

## 目录

- [概述](#概述)
- [核心架构](#核心架构)
- [ULyraGamePhaseSubsystem 详解](#ulyragamephasesubsystem-详解)
- [ULyraGamePhaseAbility 详解](#ulyragamephaseability-详解)
- [与 Experience System 的集成](#与-experience-system-的集成)
- [网络同步机制](#网络同步机制)
- [完整代码示例](#完整代码示例)
- [实战案例：比赛模式的完整阶段流程](#实战案例比赛模式的完整阶段流程)
- [调试和性能优化](#调试和性能优化)
- [最佳实践与设计模式](#最佳实践与设计模式)
- [常见问题与解决方案](#常见问题与解决方案)
- [总结](#总结)

---

## 概述

游戏阶段管理系统是 Lyra 中用于组织和控制游戏流程的核心架构。它允许开发者将游戏的不同状态（如等待开始、游戏进行中、游戏结束等）建模为可管理、可扩展的"阶段"（Phase）。这些阶段通过 GameplayTag 层次结构进行组织，支持父子嵌套关系，并与 GAS（Gameplay Ability System）深度集成。

### 核心特性

- **基于 GameplayTag 的阶段标识**：使用层次化的 GameplayTag 来定义游戏阶段
- **嵌套阶段支持**：父阶段和子阶段可以同时活跃，但兄弟阶段互斥
- **GAS 集成**：每个游戏阶段都是一个 GameplayAbility，天然支持网络复制和生命周期管理
- **Observer 模式**：提供灵活的事件监听机制，支持精确匹配和部分匹配
- **服务器权威**：阶段转换仅在服务器执行，保证网络环境下的一致性

### 设计理念

Lyra 的 GamePhase 系统采用了一种优雅的设计：**将游戏状态作为 Gameplay Ability 来管理**。这种设计带来了几个关键优势：

1. **统一的生命周期管理**：利用 GAS 的激活/结束机制自动管理阶段的开始和结束
2. **自动网络同步**：无需额外编写网络代码，GAS 已经处理了复制逻辑
3. **与其他系统的无缝集成**：可以在阶段中使用 GameplayTag、GameplayEffect 等 GAS 特性
4. **可扩展性**：继承 `ULyraGamePhaseAbility` 即可创建自定义阶段逻辑

## 核心架构

### 系统组成

```
┌─────────────────────────────────────────────────────────┐
│           ULyraGamePhaseSubsystem (WorldSubsystem)      │
│  - 管理所有活跃的游戏阶段                                 │
│  - 处理阶段转换逻辑                                       │
│  - 维护 Observer 列表                                     │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ 使用
                 ▼
┌─────────────────────────────────────────────────────────┐
│        ULyraGamePhaseAbility (GameplayAbility)          │
│  - 表示单个游戏阶段                                       │
│  - 包含 GamePhaseTag (FGameplayTag)                      │
│  - 在激活/结束时通知 Subsystem                            │
└─────────────────────────────────────────────────────────┘
                 │
                 │ 执行在
                 ▼
┌─────────────────────────────────────────────────────────┐
│      ULyraAbilitySystemComponent (GameState)            │
│  - GameState 拥有的 ASC                                  │
│  - 用于全局游戏阶段管理                                   │
└─────────────────────────────────────────────────────────┘
```

### 阶段层次结构示例

```
Game
├── Game.WaitingToStart          (等待开始)
├── Game.Playing                 (游戏进行中 - 父阶段)
│   ├── Game.Playing.Warmup      (热身阶段)
│   ├── Game.Playing.NormalPlay  (正常游戏)
│   └── Game.Playing.SuddenDeath (突然死亡)
└── Game.GameOver                (游戏结束)
    ├── Game.GameOver.ShowScore  (显示分数)
    └── Game.GameOver.Victory    (胜利画面)
```

**关键规则：**
- 父子阶段可以同时存在：`Game.Playing` 和 `Game.Playing.Warmup` 可以共存
- 兄弟阶段互斥：启动 `Game.Playing.SuddenDeath` 会自动结束 `Game.Playing.Warmup`
- 切换父阶段会结束所有子阶段：启动 `Game.GameOver` 会结束 `Game.Playing` 及其所有子阶段

## ULyraGamePhaseSubsystem 详解

### 类定义

`ULyraGamePhaseSubsystem` 是一个 `UWorldSubsystem`，负责管理整个 World 中的游戏阶段。

```cpp
UCLASS()
class ULyraGamePhaseSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    // 启动一个新的游戏阶段
    void StartPhase(TSubclassOf<ULyraGamePhaseAbility> PhaseAbility, 
                    FLyraGamePhaseDelegate PhaseEndedCallback = FLyraGamePhaseDelegate());

    // 注册阶段开始或已活跃时的回调
    void WhenPhaseStartsOrIsActive(FGameplayTag PhaseTag, 
                                   EPhaseTagMatchType MatchType, 
                                   const FLyraGamePhaseTagDelegate& WhenPhaseActive);

    // 注册阶段结束时的回调
    void WhenPhaseEnds(FGameplayTag PhaseTag, 
                      EPhaseTagMatchType MatchType, 
                      const FLyraGamePhaseTagDelegate& WhenPhaseEnd);

    // 检查某个阶段是否活跃
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, BlueprintPure = false)
    bool IsPhaseActive(const FGameplayTag& PhaseTag) const;

protected:
    // 当阶段 Ability 开始时被调用
    void OnBeginPhase(const ULyraGamePhaseAbility* PhaseAbility, 
                     const FGameplayAbilitySpecHandle PhaseAbilityHandle);

    // 当阶段 Ability 结束时被调用
    void OnEndPhase(const ULyraGamePhaseAbility* PhaseAbility, 
                   const FGameplayAbilitySpecHandle PhaseAbilityHandle);

private:
    // 活跃阶段映射表
    TMap<FGameplayAbilitySpecHandle, FLyraGamePhaseEntry> ActivePhaseMap;

    // 阶段开始观察者列表
    TArray<FPhaseObserver> PhaseStartObservers;

    // 阶段结束观察者列表
    TArray<FPhaseObserver> PhaseEndObservers;

    friend class ULyraGamePhaseAbility;
};
```

### 核心数据结构

#### FLyraGamePhaseEntry

```cpp
struct FLyraGamePhaseEntry
{
    // 阶段的 GameplayTag
    FGameplayTag PhaseTag;
    
    // 阶段结束时的回调
    FLyraGamePhaseDelegate PhaseEndedCallback;
};
```

这个结构体存储在 `ActivePhaseMap` 中，用 `FGameplayAbilitySpecHandle` 作为键，追踪每个活跃阶段的信息。

#### FPhaseObserver

```cpp
struct FPhaseObserver
{
    // 匹配指定的阶段 Tag
    bool IsMatch(const FGameplayTag& ComparePhaseTag) const;

    // 要监听的阶段 Tag
    FGameplayTag PhaseTag;
    
    // 匹配类型（精确匹配或部分匹配）
    EPhaseTagMatchType MatchType = EPhaseTagMatchType::ExactMatch;
    
    // 匹配成功时的回调
    FLyraGamePhaseTagDelegate PhaseCallback;
};
```

#### EPhaseTagMatchType

```cpp
UENUM(BlueprintType)
enum class EPhaseTagMatchType : uint8
{
    // 精确匹配：只接收完全相同的阶段
    // 例如，注册 "Game.Playing" 只会匹配 "Game.Playing"，不会匹配 "Game.Playing.Warmup"
    ExactMatch,

    // 部分匹配：接收所有子阶段
    // 例如，注册 "Game.Playing" 会匹配 "Game.Playing" 以及 "Game.Playing.Warmup"
    PartialMatch
};
```

### StartPhase 实现分析

```cpp
void ULyraGamePhaseSubsystem::StartPhase(TSubclassOf<ULyraGamePhaseAbility> PhaseAbility, 
                                          FLyraGamePhaseDelegate PhaseEndedCallback)
{
    UWorld* World = GetWorld();
    
    // 获取 GameState 上的 AbilitySystemComponent
    ULyraAbilitySystemComponent* GameState_ASC = 
        World->GetGameState()->FindComponentByClass<ULyraAbilitySystemComponent>();
    
    if (ensure(GameState_ASC))
    {
        // 创建 Ability Spec
        FGameplayAbilitySpec PhaseSpec(PhaseAbility, 1, 0, this);
        
        // 激活并立即执行 Ability
        FGameplayAbilitySpecHandle SpecHandle = GameState_ASC->GiveAbilityAndActivateOnce(PhaseSpec);
        
        // 检查 Ability 是否成功激活
        FGameplayAbilitySpec* FoundSpec = GameState_ASC->FindAbilitySpecFromHandle(SpecHandle);
        
        if (FoundSpec && FoundSpec->IsActive())
        {
            // 将阶段添加到活跃阶段映射表
            FLyraGamePhaseEntry& Entry = ActivePhaseMap.FindOrAdd(SpecHandle);
            Entry.PhaseEndedCallback = PhaseEndedCallback;
        }
        else
        {
            // Ability 激活失败，立即执行结束回调
            PhaseEndedCallback.ExecuteIfBound(nullptr);
        }
    }
}
```

**关键点：**
1. 游戏阶段 Ability 被赋予到 **GameState 的 ASC** 上，而不是某个特定 Pawn
2. 使用 `GiveAbilityAndActivateOnce` 确保 Ability 立即激活
3. 成功激活后才将其添加到 `ActivePhaseMap` 中追踪

### OnBeginPhase 阶段转换逻辑

这是整个系统最核心的部分，处理阶段转换时的互斥逻辑：

```cpp
void ULyraGamePhaseSubsystem::OnBeginPhase(const ULyraGamePhaseAbility* PhaseAbility, 
                                            const FGameplayAbilitySpecHandle PhaseAbilityHandle)
{
    const FGameplayTag IncomingPhaseTag = PhaseAbility->GetGamePhaseTag();

    UE_LOG(LogLyraGamePhase, Log, TEXT("Beginning Phase '%s' (%s)"), 
           *IncomingPhaseTag.ToString(), *GetNameSafe(PhaseAbility));

    const UWorld* World = GetWorld();
    ULyraAbilitySystemComponent* GameState_ASC = 
        World->GetGameState()->FindComponentByClass<ULyraAbilitySystemComponent>();
    
    if (ensure(GameState_ASC))
    {
        // 收集所有当前活跃的阶段
        TArray<FGameplayAbilitySpec*> ActivePhases;
        for (const auto& KVP : ActivePhaseMap)
        {
            const FGameplayAbilitySpecHandle ActiveAbilityHandle = KVP.Key;
            if (FGameplayAbilitySpec* Spec = GameState_ASC->FindAbilitySpecFromHandle(ActiveAbilityHandle))
            {
                ActivePhases.Add(Spec);
            }
        }

        // 检查每个活跃阶段是否需要结束
        for (const FGameplayAbilitySpec* ActivePhase : ActivePhases)
        {
            const ULyraGamePhaseAbility* ActivePhaseAbility = 
                CastChecked<ULyraGamePhaseAbility>(ActivePhase->Ability);
            const FGameplayTag ActivePhaseTag = ActivePhaseAbility->GetGamePhaseTag();
            
            // 关键判断：如果新阶段 Tag 不匹配（不是祖先关系），则结束活跃阶段
            // 例如：
            // - IncomingPhaseTag = "Game.Playing.SuddenDeath"
            // - ActivePhaseTag = "Game.Playing.Warmup"
            // - "Game.Playing.SuddenDeath" 不匹配 "Game.Playing.Warmup"，所以结束 Warmup
            // 
            // 但是：
            // - ActivePhaseTag = "Game.Playing"
            // - "Game.Playing.SuddenDeath" 匹配 "Game.Playing"（是其子节点），所以保留 Playing
            if (!IncomingPhaseTag.MatchesTag(ActivePhaseTag))
            {
                UE_LOG(LogLyraGamePhase, Log, TEXT("\tEnding Phase '%s' (%s)"), 
                       *ActivePhaseTag.ToString(), *GetNameSafe(ActivePhaseAbility));

                FGameplayAbilitySpecHandle HandleToEnd = ActivePhase->Handle;
                
                // 取消该阶段的 Ability
                GameState_ASC->CancelAbilitiesByFunc(
                    [HandleToEnd](const ULyraGameplayAbility* LyraAbility, FGameplayAbilitySpecHandle Handle) {
                        return Handle == HandleToEnd;
                    }, true);
            }
        }

        // 将新阶段添加到活跃阶段映射表
        FLyraGamePhaseEntry& Entry = ActivePhaseMap.FindOrAdd(PhaseAbilityHandle);
        Entry.PhaseTag = IncomingPhaseTag;

        // 通知所有监听该阶段开始的观察者
        for (const FPhaseObserver& Observer : PhaseStartObservers)
        {
            if (Observer.IsMatch(IncomingPhaseTag))
            {
                Observer.PhaseCallback.ExecuteIfBound(IncomingPhaseTag);
            }
        }
    }
}
```

**核心逻辑解析：**

使用 `MatchesTag` 判断层次关系：
- `"Game.Playing.SuddenDeath".MatchesTag("Game.Playing")` → `true`（子匹配父）
- `"Game.Playing".MatchesTag("Game.Playing.SuddenDeath")` → `false`（父不匹配子）
- `"Game.Playing.Warmup".MatchesTag("Game.Playing.SuddenDeath")` → `false`（兄弟不匹配）

因此，当启动 `Game.Playing.SuddenDeath` 时：
- 不会结束 `Game.Playing`（因为新阶段匹配它）
- 会结束 `Game.Playing.Warmup`（因为新阶段不匹配它）
- 会结束 `Game.GameOver`（因为新阶段不匹配它）

### OnEndPhase 阶段结束处理

```cpp
void ULyraGamePhaseSubsystem::OnEndPhase(const ULyraGamePhaseAbility* PhaseAbility, 
                                          const FGameplayAbilitySpecHandle PhaseAbilityHandle)
{
    const FGameplayTag EndedPhaseTag = PhaseAbility->GetGamePhaseTag();
    
    UE_LOG(LogLyraGamePhase, Log, TEXT("Ended Phase '%s' (%s)"), 
           *EndedPhaseTag.ToString(), *GetNameSafe(PhaseAbility));

    // 执行阶段结束回调
    const FLyraGamePhaseEntry& Entry = ActivePhaseMap.FindChecked(PhaseAbilityHandle);
    Entry.PhaseEndedCallback.ExecuteIfBound(PhaseAbility);

    // 从活跃阶段映射表中移除
    ActivePhaseMap.Remove(PhaseAbilityHandle);

    // 通知所有监听该阶段结束的观察者
    for (const FPhaseObserver& Observer : PhaseEndObservers)
    {
        if (Observer.IsMatch(EndedPhaseTag))
        {
            Observer.PhaseCallback.ExecuteIfBound(EndedPhaseTag);
        }
    }
}
```

### Observer 模式实现

#### 注册观察者

```cpp
void ULyraGamePhaseSubsystem::WhenPhaseStartsOrIsActive(FGameplayTag PhaseTag, 
                                                         EPhaseTagMatchType MatchType, 
                                                         const FLyraGamePhaseTagDelegate& WhenPhaseActive)
{
    // 创建观察者
    FPhaseObserver Observer;
    Observer.PhaseTag = PhaseTag;
    Observer.MatchType = MatchType;
    Observer.PhaseCallback = WhenPhaseActive;
    
    // 添加到观察者列表
    PhaseStartObservers.Add(Observer);

    // 如果阶段已经活跃，立即调用回调
    if (IsPhaseActive(PhaseTag))
    {
        WhenPhaseActive.ExecuteIfBound(PhaseTag);
    }
}
```

**关键特性：** 如果注册时阶段已经活跃，会立即触发回调，避免错过事件。

#### 匹配逻辑

```cpp
bool ULyraGamePhaseSubsystem::FPhaseObserver::IsMatch(const FGameplayTag& ComparePhaseTag) const
{
    switch(MatchType)
    {
    case EPhaseTagMatchType::ExactMatch:
        // 精确匹配：Tag 必须完全相同
        return ComparePhaseTag == PhaseTag;
        
    case EPhaseTagMatchType::PartialMatch:
        // 部分匹配：ComparePhaseTag 匹配 PhaseTag 或其任何父节点
        return ComparePhaseTag.MatchesTag(PhaseTag);
    }

    return false;
}
```

**示例：**
```cpp
// 观察者注册：PhaseTag = "Game.Playing", MatchType = PartialMatch

// 阶段 "Game.Playing" 开始 → IsMatch 返回 true
// 阶段 "Game.Playing.Warmup" 开始 → IsMatch 返回 true (子阶段匹配)
// 阶段 "Game.GameOver" 开始 → IsMatch 返回 false

// 观察者注册：PhaseTag = "Game.Playing", MatchType = ExactMatch

// 阶段 "Game.Playing" 开始 → IsMatch 返回 true
// 阶段 "Game.Playing.Warmup" 开始 → IsMatch 返回 false (只精确匹配)
// 阶段 "Game.GameOver" 开始 → IsMatch 返回 false
```

## ULyraGamePhaseAbility 详解

### 类定义

```cpp
UCLASS(Abstract, HideCategories = Input)
class ULyraGamePhaseAbility : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    ULyraGamePhaseAbility(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    // 获取此阶段的 GameplayTag
    const FGameplayTag& GetGamePhaseTag() const { return GamePhaseTag; }

protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, 
                                const FGameplayAbilityActorInfo* ActorInfo, 
                                const FGameplayAbilityActivationInfo ActivationInfo, 
                                const FGameplayEventData* TriggerEventData) override;
    
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, 
                           const FGameplayAbilityActorInfo* ActorInfo, 
                           const FGameplayAbilityActivationInfo ActivationInfo, 
                           bool bReplicateEndAbility, 
                           bool bWasCancelled) override;

    // 定义此 Ability 所属的游戏阶段
    // 例如：GamePhase.Playing.Warmup
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Game Phase")
    FGameplayTag GamePhaseTag;
};
```

### 构造函数配置

```cpp
ULyraGamePhaseAbility::ULyraGamePhaseAbility(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 阶段不需要复制 Ability 本身（服务器权威）
    ReplicationPolicy = EGameplayAbilityReplicationPolicy::ReplicateNo;
    
    // 每个 Actor 一个实例（这里是 GameState）
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    
    // 仅服务器发起
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated;
    
    // 仅服务器执行
    NetSecurityPolicy = EGameplayAbilityNetSecurityPolicy::ServerOnly;
}
```

**网络策略解析：**
- **ReplicateNo**：Ability 本身不复制，但可以通过其他机制（如 GameplayTag）同步状态
- **ServerInitiated**：只能由服务器启动阶段转换
- **ServerOnly**：阶段逻辑仅在服务器执行，客户端通过其他方式感知阶段变化

### ActivateAbility 实现

```cpp
void ULyraGamePhaseAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle, 
                                             const FGameplayAbilityActorInfo* ActorInfo, 
                                             const FGameplayAbilityActivationInfo ActivationInfo, 
                                             const FGameplayEventData* TriggerEventData)
{
    // 只在服务器执行阶段管理逻辑
    if (ActorInfo->IsNetAuthority())
    {
        UWorld* World = ActorInfo->AbilitySystemComponent->GetWorld();
        ULyraGamePhaseSubsystem* PhaseSubsystem = UWorld::GetSubsystem<ULyraGamePhaseSubsystem>(World);
        
        // 通知 Subsystem 阶段开始
        PhaseSubsystem->OnBeginPhase(this, Handle);
    }

    // 调用父类激活逻辑
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
}
```

### EndAbility 实现

```cpp
void ULyraGamePhaseAbility::EndAbility(const FGameplayAbilitySpecHandle Handle, 
                                        const FGameplayAbilityActorInfo* ActorInfo, 
                                        const FGameplayAbilityActivationInfo ActivationInfo, 
                                        bool bReplicateEndAbility, 
                                        bool bWasCancelled)
{
    // 只在服务器执行阶段管理逻辑
    if (ActorInfo->IsNetAuthority())
    {
        UWorld* World = ActorInfo->AbilitySystemComponent->GetWorld();
        ULyraGamePhaseSubsystem* PhaseSubsystem = UWorld::GetSubsystem<ULyraGamePhaseSubsystem>(World);
        
        // 通知 Subsystem 阶段结束
        PhaseSubsystem->OnEndPhase(this, Handle);
    }

    // 调用父类结束逻辑
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

### 数据验证

```cpp
#if WITH_EDITOR
EDataValidationResult ULyraGamePhaseAbility::IsDataValid(FDataValidationContext& Context) const
{
    EDataValidationResult Result = CombineDataValidationResults(Super::IsDataValid(Context), 
                                                                  EDataValidationResult::Valid);

    // 确保 GamePhaseTag 已设置
    if (!GamePhaseTag.IsValid())
    {
        Result = EDataValidationResult::Invalid;
        Context.AddError(LOCTEXT("GamePhaseTagNotSet", 
                                 "GamePhaseTag must be set to a tag representing the current phase."));
    }

    return Result;
}
#endif
```

在编辑器中，如果创建的 GamePhaseAbility 没有设置 `GamePhaseTag`，数据验证会失败并报错。

## 与 Experience System 的集成

### Experience 加载流程中的阶段管理

Lyra 的 Experience System 负责加载游戏玩法配置。游戏阶段系统与之紧密集成：

```cpp
// 伪代码示例：Experience 加载完成后启动初始阶段

void ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted()
{
    // ... Experience 资源加载完成

    // 触发 Experience 加载完成的回调
    OnExperienceLoaded.Broadcast(CurrentExperience);

    // 在某些模式下，可能会在这里启动初始游戏阶段
    // 例如：
    // ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    // PhaseSubsystem->StartPhase(InitialPhaseAbilityClass);
}
```

### Experience 定义中的阶段配置

虽然 Experience Definition 本身不直接包含阶段配置，但可以通过 GameFeature Actions 在 Experience 加载时自动启动阶段：

```cpp
// 示例：自定义 GameFeatureAction 在 Experience 激活时启动阶段

UCLASS()
class UGameFeatureAction_StartGamePhase : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override
    {
        // 当 Experience 的 GameFeature 激活时
        for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
        {
            if (UWorld* World = WorldContext.World())
            {
                if (ULyraGamePhaseSubsystem* PhaseSubsystem = World->GetSubsystem<ULyraGamePhaseSubsystem>())
                {
                    PhaseSubsystem->StartPhase(PhaseAbilityClass);
                }
            }
        }
    }

protected:
    UPROPERTY(EditAnywhere, Category = "Game Phase")
    TSubclassOf<ULyraGamePhaseAbility> PhaseAbilityClass;
};
```

## 网络同步机制

### 服务器权威架构

游戏阶段系统采用**服务器权威**设计：

1. **只在服务器执行**：所有阶段转换逻辑只在服务器执行（`IsNetAuthority()` 检查）
2. **不复制 Ability**：`ReplicationPolicy = ReplicateNo`，Ability 本身不需要复制
3. **通过其他机制同步**：客户端通过以下方式感知阶段变化：
   - **Replicated GameplayTags**：如果需要，可以在 GameState 上添加复制的 GameplayTag 容器
   - **GameplayCue**：在阶段转换时触发 GameplayCue 通知客户端
   - **Replicated Properties**：在自定义阶段 Ability 中复制关键属性

### 客户端同步示例

#### 方案 1：使用 Replicated GameplayTags

```cpp
// 在 GameState 上添加复制的 Tag 容器

UPROPERTY(ReplicatedUsing = OnRep_CurrentPhaseTags)
FGameplayTagContainer CurrentPhaseTags;

UFUNCTION()
void ALyraGameState::OnRep_CurrentPhaseTags()
{
    // 客户端检测到阶段 Tag 变化
    OnPhaseTagsChanged.Broadcast(CurrentPhaseTags);
}
```

在服务器上，当阶段开始时更新这个容器：

```cpp
void UMyGamePhaseAbility::ActivateAbility(...)
{
    Super::ActivateAbility(...);

    if (HasAuthority())
    {
        ALyraGameState* GameState = GetWorld()->GetGameState<ALyraGameState>();
        GameState->CurrentPhaseTags.AddTag(GamePhaseTag);
    }
}
```

#### 方案 2：使用 GameplayCue

```cpp
void UMyGamePhaseAbility::ActivateAbility(...)
{
    Super::ActivateAbility(...);

    if (HasAuthority())
    {
        // 触发 GameplayCue，客户端会收到并播放特效/声音
        FGameplayCueParameters CueParams;
        CueParams.RawMagnitude = 1.0f;
        
        GetAbilitySystemComponentFromActorInfo()->ExecuteGameplayCue(
            FGameplayTag::RequestGameplayTag(TEXT("GameplayCue.Phase.Started")),
            CueParams
        );
    }
}
```

#### 方案 3：通过 RPC 通知客户端

虽然阶段 Ability 本身是服务器独有的，但可以在自定义子类中添加客户端 RPC：

```cpp
UCLASS()
class UMyCustomGamePhaseAbility : public ULyraGamePhaseAbility
{
    GENERATED_BODY()

protected:
    virtual void ActivateAbility(...) override
    {
        Super::ActivateAbility(...);

        if (HasAuthority())
        {
            // 通知所有客户端
            MulticastOnPhaseStarted(GamePhaseTag);
        }
    }

    UFUNCTION(NetMulticast, Reliable)
    void MulticastOnPhaseStarted(FGameplayTag PhaseTag);
};

void UMyCustomGamePhaseAbility::MulticastOnPhaseStarted_Implementation(FGameplayTag PhaseTag)
{
    // 在所有客户端执行
    UE_LOG(LogTemp, Log, TEXT("Phase Started on Client: %s"), *PhaseTag.ToString());
    
    // 播放客户端特效、音效等
}
```

### 网络同步的最佳实践

1. **最小化网络流量**：不要为每个阶段变化都发送 RPC，优先使用复制的属性
2. **使用 GameplayTag 复制**：GameplayTag 容器已经优化了网络传输
3. **利用 GameplayCue**：GAS 的 GameplayCue 系统已经处理了网络逻辑
4. **客户端预测**：对于不影响游戏逻辑的视觉效果，可以让客户端根据其他信息预测阶段变化

## 完整代码示例

### 示例 1：创建自定义游戏阶段

#### 定义 GameplayTags

首先在项目的 GameplayTags 配置中定义阶段标签：

```ini
; DefaultGameplayTags.ini

[/Script/GameplayTags.GameplayTagsList]
+GameplayTagList=(Tag="Game.Phase.WaitingToStart",DevComment="等待游戏开始")
+GameplayTagList=(Tag="Game.Phase.Playing",DevComment="游戏进行中")
+GameplayTagList=(Tag="Game.Phase.Playing.Warmup",DevComment="热身阶段")
+GameplayTagList=(Tag="Game.Phase.Playing.MainGame",DevComment="主游戏阶段")
+GameplayTagList=(Tag="Game.Phase.PostGame",DevComment="游戏结束")
+GameplayTagList=(Tag="Game.Phase.PostGame.ShowResults",DevComment="显示结果")
```

#### 创建 WaitingToStart 阶段

```cpp
// GA_GamePhase_WaitingToStart.h

#pragma once

#include "AbilitySystem/Phases/LyraGamePhaseAbility.h"
#include "GA_GamePhase_WaitingToStart.generated.h"

/**
 * 等待开始阶段
 * 玩家加入游戏，等待足够数量的玩家准备就绪
 */
UCLASS()
class UGA_GamePhase_WaitingToStart : public ULyraGamePhaseAbility
{
    GENERATED_BODY()

public:
    UGA_GamePhase_WaitingToStart();

protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                const FGameplayAbilityActorInfo* ActorInfo,
                                const FGameplayAbilityActivationInfo ActivationInfo,
                                const FGameplayEventData* TriggerEventData) override;

    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle,
                           const FGameplayAbilityActorInfo* ActorInfo,
                           const FGameplayAbilityActivationInfo ActivationInfo,
                           bool bReplicateEndAbility,
                           bool bWasCancelled) override;

private:
    // 检查是否可以开始游戏
    void CheckStartConditions();

    // 定时器句柄
    FTimerHandle CheckTimerHandle;

    // 需要的最小玩家数量
    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    int32 MinPlayerCount = 2;

    // 等待超时时间（秒）
    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    float WaitTimeout = 60.0f;
};
```

```cpp
// GA_GamePhase_WaitingToStart.cpp

#include "GA_GamePhase_WaitingToStart.h"
#include "GameFramework/GameStateBase.h"
#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"
#include "TimerManager.h"

UGA_GamePhase_WaitingToStart::UGA_GamePhase_WaitingToStart()
{
    // 设置阶段标签
    GamePhaseTag = FGameplayTag::RequestGameplayTag(TEXT("Game.Phase.WaitingToStart"));
    
    // 阶段配置
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerOnly;
}

void UGA_GamePhase_WaitingToStart::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                                     const FGameplayAbilityActorInfo* ActorInfo,
                                                     const FGameplayAbilityActivationInfo ActivationInfo,
                                                     const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (HasAuthority(&ActivationInfo))
    {
        UE_LOG(LogTemp, Log, TEXT("Game Phase: Waiting To Start"));

        // 启动定时检查
        UWorld* World = GetWorld();
        World->GetTimerManager().SetTimer(
            CheckTimerHandle,
            this,
            &UGA_GamePhase_WaitingToStart::CheckStartConditions,
            1.0f,  // 每秒检查一次
            true   // 循环
        );

        // 设置超时
        FTimerHandle TimeoutHandle;
        World->GetTimerManager().SetTimer(
            TimeoutHandle,
            [this]()
            {
                UE_LOG(LogTemp, Warning, TEXT("WaitingToStart timeout, forcing game start"));
                // 强制进入下一阶段
                ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
                // PhaseSubsystem->StartPhase(UGA_GamePhase_Playing::StaticClass());
            },
            WaitTimeout,
            false  // 不循环
        );
    }
}

void UGA_GamePhase_WaitingToStart::CheckStartConditions()
{
    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (!GameState)
    {
        return;
    }

    // 检查玩家数量
    int32 PlayerCount = GameState->PlayerArray.Num();
    
    if (PlayerCount >= MinPlayerCount)
    {
        UE_LOG(LogTemp, Log, TEXT("Start conditions met (%d/%d players), transitioning to Playing phase"),
               PlayerCount, MinPlayerCount);

        // 清除定时器
        GetWorld()->GetTimerManager().ClearTimer(CheckTimerHandle);

        // 转换到 Playing 阶段
        ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
        // PhaseSubsystem->StartPhase(UGA_GamePhase_Playing::StaticClass());
    }
}

void UGA_GamePhase_WaitingToStart::EndAbility(const FGameplayAbilitySpecHandle Handle,
                                                const FGameplayAbilityActorInfo* ActorInfo,
                                                const FGameplayAbilityActivationInfo ActivationInfo,
                                                bool bReplicateEndAbility,
                                                bool bWasCancelled)
{
    // 清理定时器
    if (CheckTimerHandle.IsValid())
    {
        GetWorld()->GetTimerManager().ClearTimer(CheckTimerHandle);
    }

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);

    UE_LOG(LogTemp, Log, TEXT("Game Phase: Exiting Waiting To Start"));
}
```

#### 创建 Playing 阶段

```cpp
// GA_GamePhase_Playing.h

#pragma once

#include "AbilitySystem/Phases/LyraGamePhaseAbility.h"
#include "GA_GamePhase_Playing.generated.h"

/**
 * 游戏进行中阶段
 * 主要游戏逻辑运行阶段
 */
UCLASS()
class UGA_GamePhase_Playing : public ULyraGamePhaseAbility
{
    GENERATED_BODY()

public:
    UGA_GamePhase_Playing();

protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                const FGameplayAbilityActorInfo* ActorInfo,
                                const FGameplayAbilityActivationInfo ActivationInfo,
                                const FGameplayEventData* TriggerEventData) override;

    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle,
                           const FGameplayAbilityActorInfo* ActorInfo,
                           const FGameplayAbilityActivationInfo ActivationInfo,
                           bool bReplicateEndAbility,
                           bool bWasCancelled) override;

private:
    // 检查游戏结束条件
    void CheckEndConditions();

    // 定时器句柄
    FTimerHandle CheckTimerHandle;

    // 游戏持续时间（秒），0 表示无限制
    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    float GameDuration = 300.0f;  // 5 分钟

    // 游戏开始时间
    float GameStartTime;
};
```

```cpp
// GA_GamePhase_Playing.cpp

#include "GA_GamePhase_Playing.h"
#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"
#include "TimerManager.h"

UGA_GamePhase_Playing::UGA_GamePhase_Playing()
{
    GamePhaseTag = FGameplayTag::RequestGameplayTag(TEXT("Game.Phase.Playing"));
}

void UGA_GamePhase_Playing::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                              const FGameplayAbilityActorInfo* ActorInfo,
                                              const FGameplayAbilityActivationInfo ActivationInfo,
                                              const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (HasAuthority(&ActivationInfo))
    {
        UE_LOG(LogTemp, Log, TEXT("Game Phase: Playing - Game Started!"));

        GameStartTime = GetWorld()->GetTimeSeconds();

        // 启动定时检查
        GetWorld()->GetTimerManager().SetTimer(
            CheckTimerHandle,
            this,
            &UGA_GamePhase_Playing::CheckEndConditions,
            1.0f,
            true
        );

        // 如果设置了游戏时长，启动子阶段 Warmup
        if (GameDuration > 0)
        {
            ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
            // PhaseSubsystem->StartPhase(UGA_GamePhase_Playing_Warmup::StaticClass());
        }

        // 触发 GameplayCue 通知客户端
        FGameplayCueParameters CueParams;
        GetAbilitySystemComponentFromActorInfo()->ExecuteGameplayCue(
            FGameplayTag::RequestGameplayTag(TEXT("GameplayCue.Phase.GameStart")),
            CueParams
        );
    }
}

void UGA_GamePhase_Playing::CheckEndConditions()
{
    // 检查时间限制
    if (GameDuration > 0)
    {
        float ElapsedTime = GetWorld()->GetTimeSeconds() - GameStartTime;
        if (ElapsedTime >= GameDuration)
        {
            UE_LOG(LogTemp, Log, TEXT("Game time limit reached, ending game"));

            // 转换到 PostGame 阶段
            ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
            // PhaseSubsystem->StartPhase(UGA_GamePhase_PostGame::StaticClass());
        }
    }

    // 这里可以添加其他结束条件，例如：
    // - 某队达到目标分数
    // - 所有敌人被消灭
    // - 关键目标完成
}

void UGA_GamePhase_Playing::EndAbility(const FGameplayAbilitySpecHandle Handle,
                                        const FGameplayAbilityActorInfo* ActorInfo,
                                        const FGameplayAbilityActivationInfo ActivationInfo,
                                        bool bReplicateEndAbility,
                                        bool bWasCancelled)
{
    if (CheckTimerHandle.IsValid())
    {
        GetWorld()->GetTimerManager().ClearTimer(CheckTimerHandle);
    }

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);

    UE_LOG(LogTemp, Log, TEXT("Game Phase: Exiting Playing"));
}
```

#### 创建 PostGame 阶段

```cpp
// GA_GamePhase_PostGame.h

#pragma once

#include "AbilitySystem/Phases/LyraGamePhaseAbility.h"
#include "GA_GamePhase_PostGame.generated.h"

/**
 * 游戏结束阶段
 * 显示结果，等待返回主菜单或开始新游戏
 */
UCLASS()
class UGA_GamePhase_PostGame : public ULyraGamePhaseAbility
{
    GENERATED_BODY()

public:
    UGA_GamePhase_PostGame();

protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                const FGameplayAbilityActorInfo* ActorInfo,
                                const FGameplayAbilityActivationInfo ActivationInfo,
                                const FGameplayEventData* TriggerEventData) override;

private:
    // 结果显示持续时间（秒）
    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    float ResultsDisplayDuration = 10.0f;
};
```

```cpp
// GA_GamePhase_PostGame.cpp

#include "GA_GamePhase_PostGame.h"
#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"
#include "TimerManager.h"

UGA_GamePhase_PostGame::UGA_GamePhase_PostGame()
{
    GamePhaseTag = FGameplayTag::RequestGameplayTag(TEXT("Game.Phase.PostGame"));
}

void UGA_GamePhase_PostGame::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                               const FGameplayAbilityActorInfo* ActorInfo,
                                               const FGameplayAbilityActivationInfo ActivationInfo,
                                               const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (HasAuthority(&ActivationInfo))
    {
        UE_LOG(LogTemp, Log, TEXT("Game Phase: Post Game - Showing Results"));

        // 触发 GameplayCue 显示结果界面
        FGameplayCueParameters CueParams;
        GetAbilitySystemComponentFromActorInfo()->ExecuteGameplayCue(
            FGameplayTag::RequestGameplayTag(TEXT("GameplayCue.Phase.GameEnd")),
            CueParams
        );

        // 延迟后可以返回主菜单或重新开始
        FTimerHandle ReturnTimerHandle;
        GetWorld()->GetTimerManager().SetTimer(
            ReturnTimerHandle,
            [this]()
            {
                UE_LOG(LogTemp, Log, TEXT("Returning to main menu or restarting game"));
                // 可以在这里触发返回主菜单的逻辑
                // 或者重新启动 WaitingToStart 阶段
            },
            ResultsDisplayDuration,
            false
        );
    }
}
```

### 示例 2：在 GameMode 中启动阶段

```cpp
// MyLyraGameMode.h

#pragma once

#include "GameModes/LyraGameMode.h"
#include "MyLyraGameMode.generated.h"

UCLASS()
class AMyLyraGameMode : public ALyraGameMode
{
    GENERATED_BODY()

protected:
    virtual void HandleMatchAssignmentIfNotExpectingOne() override;

private:
    void OnExperienceLoaded(const ULyraExperienceDefinition* Experience);
};
```

```cpp
// MyLyraGameMode.cpp

#include "MyLyraGameMode.h"
#include "GameModes/LyraExperienceManagerComponent.h"
#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"
#include "GA_GamePhase_WaitingToStart.h"

void AMyLyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
    Super::HandleMatchAssignmentIfNotExpectingOne();

    // 监听 Experience 加载完成
    ULyraExperienceManagerComponent* ExperienceComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    if (ensure(ExperienceComponent))
    {
        ExperienceComponent->CallOrRegister_OnExperienceLoaded(
            FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded)
        );
    }
}

void AMyLyraGameMode::OnExperienceLoaded(const ULyraExperienceDefinition* Experience)
{
    UE_LOG(LogTemp, Log, TEXT("Experience loaded, starting initial game phase"));

    // Experience 加载完成后，启动初始游戏阶段
    ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    if (ensure(PhaseSubsystem))
    {
        PhaseSubsystem->StartPhase(UGA_GamePhase_WaitingToStart::StaticClass());
    }
}
```

### 示例 3：使用 Observer 监听阶段变化

```cpp
// MyGameModeComponent.h

#pragma once

#include "Components/GameStateComponent.h"
#include "GameplayTagContainer.h"
#include "MyGameModeComponent.generated.h"

UCLASS()
class UMyGameModeComponent : public UGameStateComponent
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;

private:
    void OnPlayingPhaseStarted(const FGameplayTag& PhaseTag);
    void OnPlayingPhaseEnded(const FGameplayTag& PhaseTag);
};
```

```cpp
// MyGameModeComponent.cpp

#include "MyGameModeComponent.h"
#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"

void UMyGameModeComponent::BeginPlay()
{
    Super::BeginPlay();

    ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    if (PhaseSubsystem)
    {
        // 监听 Playing 阶段开始（包括所有子阶段）
        FGameplayTag PlayingTag = FGameplayTag::RequestGameplayTag(TEXT("Game.Phase.Playing"));
        
        PhaseSubsystem->WhenPhaseStartsOrIsActive(
            PlayingTag,
            EPhaseTagMatchType::PartialMatch,  // 部分匹配，包括子阶段
            FLyraGamePhaseTagDelegate::CreateUObject(this, &UMyGameModeComponent::OnPlayingPhaseStarted)
        );

        // 监听 Playing 阶段结束
        PhaseSubsystem->WhenPhaseEnds(
            PlayingTag,
            EPhaseTagMatchType::PartialMatch,
            FLyraGamePhaseTagDelegate::CreateUObject(this, &UMyGameModeComponent::OnPlayingPhaseEnded)
        );
    }
}

void UMyGameModeComponent::OnPlayingPhaseStarted(const FGameplayTag& PhaseTag)
{
    UE_LOG(LogTemp, Log, TEXT("Playing phase started: %s"), *PhaseTag.ToString());

    // 可以在这里执行游戏开始时的逻辑：
    // - 启用玩家输入
    // - 开始 AI 行为
    // - 启动游戏计时器
    // - 显示 HUD
}

void UMyGameModeComponent::OnPlayingPhaseEnded(const FGameplayTag& PhaseTag)
{
    UE_LOG(LogTemp, Log, TEXT("Playing phase ended: %s"), *PhaseTag.ToString());

    // 可以在这里执行游戏结束时的逻辑：
    // - 禁用玩家输入
    // - 停止 AI 行为
    // - 停止游戏计时器
    // - 隐藏游戏 HUD
}
```

### 示例 4：蓝图中使用游戏阶段

GamePhase 系统提供了蓝图友好的接口：

```cpp
// 蓝图节点：Start Phase (K2_StartPhase)

// 输入：
// - Phase: TSubclassOf<ULyraGamePhaseAbility>
// - Phase Ended: Event Delegate

// 输出：
// - 无（异步执行）

// 示例蓝图逻辑：
// Event BeginPlay
//   → Get World Subsystem (LyraGamePhaseSubsystem)
//   → Start Phase (GA_GamePhase_WaitingToStart)
//       → Phase Ended: Print String ("Phase Ended")
```

```cpp
// 蓝图节点：When Phase Starts Or Is Active (K2_WhenPhaseStartsOrIsActive)

// 输入：
// - Phase Tag: FGameplayTag (例如 "Game.Phase.Playing")
// - Match Type: EPhaseTagMatchType (Exact Match / Partial Match)
// - When Phase Active: Event Delegate

// 输出：
// - 无（异步回调）

// 示例蓝图逻辑：
// Event BeginPlay
//   → Get World Subsystem (LyraGamePhaseSubsystem)
//   → When Phase Starts Or Is Active
//       → Phase Tag: "Game.Phase.Playing"
//       → Match Type: Partial Match
//       → When Phase Active: 
//           → Print String ("Playing phase is now active!")
//           → Enable Player Input
```

```cpp
// 蓝图节点：Is Phase Active

// 输入：
// - Phase Tag: FGameplayTag

// 输出：
// - Return Value: bool

// 示例蓝图逻辑：
// Event Tick
//   → Get World Subsystem (LyraGamePhaseSubsystem)
//   → Is Phase Active ("Game.Phase.Playing")
//       → Branch
//           → True: Update Game Timer
//           → False: Do Nothing
```

## 实战案例：比赛模式的完整阶段流程

下面是一个完整的比赛模式实现，包含赛前准备、比赛进行、结算三个主要阶段。

### 阶段结构设计

```
Game.Match
├── Game.Match.PreMatch          (赛前阶段 - 父阶段)
│   ├── Game.Match.PreMatch.Lobby         (大厅等待)
│   └── Game.Match.PreMatch.Countdown     (倒计时)
├── Game.Match.InProgress        (比赛进行中 - 父阶段)
│   ├── Game.Match.InProgress.Round1      (第一回合)
│   ├── Game.Match.InProgress.Round2      (第二回合)
│   └── Game.Match.InProgress.Overtime    (加时赛)
└── Game.Match.PostMatch         (赛后阶段 - 父阶段)
    ├── Game.Match.PostMatch.Results      (结果显示)
    └── Game.Match.PostMatch.Awards       (颁奖环节)
```

### PreMatch 阶段实现

```cpp
// GA_GamePhase_PreMatch.h

#pragma once

#include "AbilitySystem/Phases/LyraGamePhaseAbility.h"
#include "GA_GamePhase_PreMatch.generated.h"

UCLASS()
class UGA_GamePhase_PreMatch : public ULyraGamePhaseAbility
{
    GENERATED_BODY()

public:
    UGA_GamePhase_PreMatch();

protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                const FGameplayAbilityActorInfo* ActorInfo,
                                const FGameplayAbilityActivationInfo ActivationInfo,
                                const FGameplayEventData* TriggerEventData) override;

    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle,
                           const FGameplayAbilityActorInfo* ActorInfo,
                           const FGameplayAbilityActivationInfo ActivationInfo,
                           bool bReplicateEndAbility,
                           bool bWasCancelled) override;

private:
    void OnLobbyPhaseEnded(const ULyraGamePhaseAbility* Phase);
    void OnCountdownPhaseEnded(const ULyraGamePhaseAbility* Phase);

    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    TSubclassOf<ULyraGamePhaseAbility> LobbyPhaseClass;

    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    TSubclassOf<ULyraGamePhaseAbility> CountdownPhaseClass;

    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    TSubclassOf<ULyraGamePhaseAbility> InProgressPhaseClass;
};
```

```cpp
// GA_GamePhase_PreMatch.cpp

#include "GA_GamePhase_PreMatch.h"
#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"

UGA_GamePhase_PreMatch::UGA_GamePhase_PreMatch()
{
    GamePhaseTag = FGameplayTag::RequestGameplayTag(TEXT("Game.Match.PreMatch"));
}

void UGA_GamePhase_PreMatch::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                              const FGameplayAbilityActorInfo* ActorInfo,
                                              const FGameplayAbilityActivationInfo ActivationInfo,
                                              const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (HasAuthority(&ActivationInfo))
    {
        UE_LOG(LogTemp, Log, TEXT("=== PreMatch Phase Started ==="));

        // 启动 Lobby 子阶段
        ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
        PhaseSubsystem->StartPhase(
            LobbyPhaseClass,
            FLyraGamePhaseDelegate::CreateUObject(this, &UGA_GamePhase_PreMatch::OnLobbyPhaseEnded)
        );

        // 通知客户端显示赛前界面
        FGameplayCueParameters CueParams;
        GetAbilitySystemComponentFromActorInfo()->ExecuteGameplayCue(
            FGameplayTag::RequestGameplayTag(TEXT("GameplayCue.Phase.PreMatch.Enter")),
            CueParams
        );
    }
}

void UGA_GamePhase_PreMatch::OnLobbyPhaseEnded(const ULyraGamePhaseAbility* Phase)
{
    UE_LOG(LogTemp, Log, TEXT("Lobby phase ended, starting countdown"));

    // Lobby 阶段结束，启动倒计时阶段
    ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    PhaseSubsystem->StartPhase(
        CountdownPhaseClass,
        FLyraGamePhaseDelegate::CreateUObject(this, &UGA_GamePhase_PreMatch::OnCountdownPhaseEnded)
    );
}

void UGA_GamePhase_PreMatch::OnCountdownPhaseEnded(const ULyraGamePhaseAbility* Phase)
{
    UE_LOG(LogTemp, Log, TEXT("Countdown finished, starting match"));

    // 倒计时结束，启动比赛阶段（这会自动结束 PreMatch 阶段）
    ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    PhaseSubsystem->StartPhase(InProgressPhaseClass);
}

void UGA_GamePhase_PreMatch::EndAbility(const FGameplayAbilitySpecHandle Handle,
                                         const FGameplayAbilityActorInfo* ActorInfo,
                                         const FGameplayAbilityActivationInfo ActivationInfo,
                                         bool bReplicateEndAbility,
                                         bool bWasCancelled)
{
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);

    UE_LOG(LogTemp, Log, TEXT("=== PreMatch Phase Ended ==="));
}
```

### Countdown 子阶段实现

```cpp
// GA_GamePhase_PreMatch_Countdown.h

#pragma once

#include "AbilitySystem/Phases/LyraGamePhaseAbility.h"
#include "GA_GamePhase_PreMatch_Countdown.generated.h"

UCLASS()
class UGA_GamePhase_PreMatch_Countdown : public ULyraGamePhaseAbility
{
    GENERATED_BODY()

public:
    UGA_GamePhase_PreMatch_Countdown();

protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                const FGameplayAbilityActorInfo* ActorInfo,
                                const FGameplayAbilityActivationInfo ActivationInfo,
                                const FGameplayEventData* TriggerEventData) override;

    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle,
                           const FGameplayAbilityActorInfo* ActorInfo,
                           const FGameplayAbilityActivationInfo ActivationInfo,
                           bool bReplicateEndAbility,
                           bool bWasCancelled) override;

private:
    void UpdateCountdown();

    FTimerHandle CountdownTimerHandle;

    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    int32 CountdownDuration = 5;  // 5 秒倒计时

    int32 RemainingTime;

    // 复制给客户端的倒计时
    UPROPERTY(Replicated)
    int32 ReplicatedCountdown;
};
```

```cpp
// GA_GamePhase_PreMatch_Countdown.cpp

#include "GA_GamePhase_PreMatch_Countdown.h"
#include "Net/UnrealNetwork.h"
#include "TimerManager.h"

UGA_GamePhase_PreMatch_Countdown::UGA_GamePhase_PreMatch_Countdown()
{
    GamePhaseTag = FGameplayTag::RequestGameplayTag(TEXT("Game.Match.PreMatch.Countdown"));
    
    // 需要复制倒计时数值给客户端
    bReplicateUsingRegisteredSubObjectList = true;
}

void UGA_GamePhase_PreMatch_Countdown::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(UGA_GamePhase_PreMatch_Countdown, ReplicatedCountdown);
}

void UGA_GamePhase_PreMatch_Countdown::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                                        const FGameplayAbilityActorInfo* ActorInfo,
                                                        const FGameplayAbilityActivationInfo ActivationInfo,
                                                        const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (HasAuthority(&ActivationInfo))
    {
        RemainingTime = CountdownDuration;
        ReplicatedCountdown = RemainingTime;

        UE_LOG(LogTemp, Log, TEXT("Countdown started: %d seconds"), CountdownDuration);

        // 启动倒计时定时器
        GetWorld()->GetTimerManager().SetTimer(
            CountdownTimerHandle,
            this,
            &UGA_GamePhase_PreMatch_Countdown::UpdateCountdown,
            1.0f,
            true
        );

        // 触发倒计时开始的 GameplayCue
        FGameplayCueParameters CueParams;
        CueParams.RawMagnitude = CountdownDuration;
        GetAbilitySystemComponentFromActorInfo()->ExecuteGameplayCue(
            FGameplayTag::RequestGameplayTag(TEXT("GameplayCue.Phase.Countdown.Start")),
            CueParams
        );
    }
}

void UGA_GamePhase_PreMatch_Countdown::UpdateCountdown()
{
    RemainingTime--;
    ReplicatedCountdown = RemainingTime;

    UE_LOG(LogTemp, Log, TEXT("Countdown: %d"), RemainingTime);

    // 每秒触发一次倒计时音效
    FGameplayCueParameters CueParams;
    CueParams.RawMagnitude = RemainingTime;
    GetAbilitySystemComponentFromActorInfo()->ExecuteGameplayCue(
        FGameplayTag::RequestGameplayTag(TEXT("GameplayCue.Phase.Countdown.Tick")),
        CueParams
    );

    if (RemainingTime <= 0)
    {
        // 倒计时结束，结束此 Ability
        GetWorld()->GetTimerManager().ClearTimer(CountdownTimerHandle);
        EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
    }
}

void UGA_GamePhase_PreMatch_Countdown::EndAbility(const FGameplayAbilitySpecHandle Handle,
                                                   const FGameplayAbilityActorInfo* ActorInfo,
                                                   const FGameplayAbilityActivationInfo ActivationInfo,
                                                   bool bReplicateEndAbility,
                                                   bool bWasCancelled)
{
    if (CountdownTimerHandle.IsValid())
    {
        GetWorld()->GetTimerManager().ClearTimer(CountdownTimerHandle);
    }

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);

    if (HasAuthority(&ActivationInfo))
    {
        // 触发倒计时结束的 GameplayCue
        FGameplayCueParameters CueParams;
        GetAbilitySystemComponentFromActorInfo()->ExecuteGameplayCue(
            FGameplayTag::RequestGameplayTag(TEXT("GameplayCue.Phase.Countdown.End")),
            CueParams
        );
    }
}
```

### InProgress 阶段实现（带回合管理）

```cpp
// GA_GamePhase_InProgress.h

#pragma once

#include "AbilitySystem/Phases/LyraGamePhaseAbility.h"
#include "GA_GamePhase_InProgress.generated.h"

UCLASS()
class UGA_GamePhase_InProgress : public ULyraGamePhaseAbility
{
    GENERATED_BODY()

public:
    UGA_GamePhase_InProgress();

protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                const FGameplayAbilityActorInfo* ActorInfo,
                                const FGameplayAbilityActivationInfo ActivationInfo,
                                const FGameplayEventData* TriggerEventData) override;

    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle,
                           const FGameplayAbilityActorInfo* ActorInfo,
                           const FGameplayAbilityActivationInfo ActivationInfo,
                           bool bReplicateEndAbility,
                           bool bWasCancelled) override;

private:
    void OnRoundEnded(const ULyraGamePhaseAbility* Phase);
    void StartNextRound();
    void CheckMatchEndConditions();

    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    TArray<TSubclassOf<ULyraGamePhaseAbility>> RoundPhaseClasses;

    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    TSubclassOf<ULyraGamePhaseAbility> PostMatchPhaseClass;

    int32 CurrentRoundIndex = 0;

    FTimerHandle CheckTimerHandle;
};
```

```cpp
// GA_GamePhase_InProgress.cpp

#include "GA_GamePhase_InProgress.h"
#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"
#include "TimerManager.h"

UGA_GamePhase_InProgress::UGA_GamePhase_InProgress()
{
    GamePhaseTag = FGameplayTag::RequestGameplayTag(TEXT("Game.Match.InProgress"));
}

void UGA_GamePhase_InProgress::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                                const FGameplayAbilityActorInfo* ActorInfo,
                                                const FGameplayAbilityActivationInfo ActivationInfo,
                                                const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (HasAuthority(&ActivationInfo))
    {
        UE_LOG(LogTemp, Log, TEXT("=== Match InProgress Phase Started ==="));

        CurrentRoundIndex = 0;

        // 启动第一个回合
        StartNextRound();

        // 定期检查比赛结束条件
        GetWorld()->GetTimerManager().SetTimer(
            CheckTimerHandle,
            this,
            &UGA_GamePhase_InProgress::CheckMatchEndConditions,
            1.0f,
            true
        );

        // 通知客户端比赛开始
        FGameplayCueParameters CueParams;
        GetAbilitySystemComponentFromActorInfo()->ExecuteGameplayCue(
            FGameplayTag::RequestGameplayTag(TEXT("GameplayCue.Phase.MatchStart")),
            CueParams
        );
    }
}

void UGA_GamePhase_InProgress::StartNextRound()
{
    if (CurrentRoundIndex < RoundPhaseClasses.Num())
    {
        UE_LOG(LogTemp, Log, TEXT("Starting Round %d"), CurrentRoundIndex + 1);

        ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
        PhaseSubsystem->StartPhase(
            RoundPhaseClasses[CurrentRoundIndex],
            FLyraGamePhaseDelegate::CreateUObject(this, &UGA_GamePhase_InProgress::OnRoundEnded)
        );

        CurrentRoundIndex++;
    }
    else
    {
        // 所有回合结束，进入 PostMatch 阶段
        UE_LOG(LogTemp, Log, TEXT("All rounds completed, ending match"));

        ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
        PhaseSubsystem->StartPhase(PostMatchPhaseClass);
    }
}

void UGA_GamePhase_InProgress::OnRoundEnded(const ULyraGamePhaseAbility* Phase)
{
    UE_LOG(LogTemp, Log, TEXT("Round ended, preparing next round"));

    // 回合间隙，可以显示回合结果
    FTimerHandle DelayHandle;
    GetWorld()->GetTimerManager().SetTimer(
        DelayHandle,
        this,
        &UGA_GamePhase_InProgress::StartNextRound,
        3.0f,  // 3 秒间隙
        false
    );
}

void UGA_GamePhase_InProgress::CheckMatchEndConditions()
{
    // 检查提前结束条件，例如：
    // - 某队达到胜利分数
    // - 超时
    // - 所有玩家离开

    // 示例：如果没有玩家，结束比赛
    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (GameState && GameState->PlayerArray.Num() == 0)
    {
        UE_LOG(LogTemp, Warning, TEXT("No players remaining, ending match"));

        ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
        PhaseSubsystem->StartPhase(PostMatchPhaseClass);
    }
}

void UGA_GamePhase_InProgress::EndAbility(const FGameplayAbilitySpecHandle Handle,
                                          const FGameplayAbilityActorInfo* ActorInfo,
                                          const FGameplayAbilityActivationInfo ActivationInfo,
                                          bool bReplicateEndAbility,
                                          bool bWasCancelled)
{
    if (CheckTimerHandle.IsValid())
    {
        GetWorld()->GetTimerManager().ClearTimer(CheckTimerHandle);
    }

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);

    UE_LOG(LogTemp, Log, TEXT("=== Match InProgress Phase Ended ==="));
}
```

### PostMatch 阶段实现

```cpp
// GA_GamePhase_PostMatch.h

#pragma once

#include "AbilitySystem/Phases/LyraGamePhaseAbility.h"
#include "GA_GamePhase_PostMatch.generated.h"

UCLASS()
class UGA_GamePhase_PostMatch : public ULyraGamePhaseAbility
{
    GENERATED_BODY()

public:
    UGA_GamePhase_PostMatch();

protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                const FGameplayAbilityActorInfo* ActorInfo,
                                const FGameplayAbilityActivationInfo ActivationInfo,
                                const FGameplayEventData* TriggerEventData) override;

private:
    void OnResultsPhaseEnded(const ULyraGamePhaseAbility* Phase);
    void OnAwardsPhaseEnded(const ULyraGamePhaseAbility* Phase);

    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    TSubclassOf<ULyraGamePhaseAbility> ResultsPhaseClass;

    UPROPERTY(EditDefaultsOnly, Category = "Game Phase")
    TSubclassOf<ULyraGamePhaseAbility> AwardsPhaseClass;
};
```

```cpp
// GA_GamePhase_PostMatch.cpp

#include "GA_GamePhase_PostMatch.h"
#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"

UGA_GamePhase_PostMatch::UGA_GamePhase_PostMatch()
{
    GamePhaseTag = FGameplayTag::RequestGameplayTag(TEXT("Game.Match.PostMatch"));
}

void UGA_GamePhase_PostMatch::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                               const FGameplayAbilityActorInfo* ActorInfo,
                                               const FGameplayAbilityActivationInfo ActivationInfo,
                                               const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (HasAuthority(&ActivationInfo))
    {
        UE_LOG(LogTemp, Log, TEXT("=== PostMatch Phase Started ==="));

        // 禁用玩家输入
        // 统计比赛数据
        // 保存玩家战绩

        // 启动结果显示子阶段
        ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
        PhaseSubsystem->StartPhase(
            ResultsPhaseClass,
            FLyraGamePhaseDelegate::CreateUObject(this, &UGA_GamePhase_PostMatch::OnResultsPhaseEnded)
        );

        // 通知客户端显示比赛结果
        FGameplayCueParameters CueParams;
        GetAbilitySystemComponentFromActorInfo()->ExecuteGameplayCue(
            FGameplayTag::RequestGameplayTag(TEXT("GameplayCue.Phase.MatchEnd")),
            CueParams
        );
    }
}

void UGA_GamePhase_PostMatch::OnResultsPhaseEnded(const ULyraGamePhaseAbility* Phase)
{
    UE_LOG(LogTemp, Log, TEXT("Results phase ended, showing awards"));

    // 显示奖励和成就
    ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    PhaseSubsystem->StartPhase(
        AwardsPhaseClass,
        FLyraGamePhaseDelegate::CreateUObject(this, &UGA_GamePhase_PostMatch::OnAwardsPhaseEnded)
    );
}

void UGA_GamePhase_PostMatch::OnAwardsPhaseEnded(const ULyraGamePhaseAbility* Phase)
{
    UE_LOG(LogTemp, Log, TEXT("Awards phase ended, match complete"));

    // 可以在这里：
    // - 返回主菜单
    // - 开始新的匹配
    // - 卸载当前地图
}
```

## 调试和性能优化

### 调试技巧

#### 1. 启用 GamePhase 日志

```cpp
// 在控制台或 DefaultEngine.ini 中启用日志

// 控制台命令：
log LogLyraGamePhase Verbose

// DefaultEngine.ini：
[Core.Log]
LogLyraGamePhase=Verbose
```

日志输出示例：
```
LogLyraGamePhase: Beginning Phase 'Game.Phase.Playing' (GA_GamePhase_Playing_C)
LogLyraGamePhase:     Ending Phase 'Game.Phase.WaitingToStart' (GA_GamePhase_WaitingToStart_C)
LogLyraGamePhase: Ended Phase 'Game.Phase.WaitingToStart' (GA_GamePhase_WaitingToStart_C)
```

#### 2. 可视化调试工具

创建一个调试 HUD 显示当前活跃的阶段：

```cpp
// DebugGamePhaseWidget.h

#pragma once

#include "Blueprint/UserWidget.h"
#include "GameplayTagContainer.h"
#include "DebugGamePhaseWidget.generated.h"

UCLASS()
class UDebugGamePhaseWidget : public UUserWidget
{
    GENERATED_BODY()

protected:
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;

    UFUNCTION(BlueprintImplementableEvent)
    void UpdatePhaseDisplay(const TArray<FGameplayTag>& ActivePhases);
};
```

```cpp
// DebugGamePhaseWidget.cpp

#include "DebugGamePhaseWidget.h"
#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"

void UDebugGamePhaseWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    Super::NativeTick(MyGeometry, InDeltaTime);

    ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    if (PhaseSubsystem)
    {
        TArray<FGameplayTag> ActivePhases;

        // 收集所有活跃阶段的 Tag
        // 注意：需要在 Subsystem 中添加一个公共接口来访问 ActivePhaseMap
        // 或者通过 IsPhaseActive 逐个检查已知的阶段

        UpdatePhaseDisplay(ActivePhases);
    }
}
```

在蓝图中实现 `UpdatePhaseDisplay`，显示活跃阶段列表。

#### 3. 控制台命令

添加自定义控制台命令用于调试：

```cpp
// MyGamePhaseDebugCommands.cpp

#include "AbilitySystem/Phases/LyraGamePhaseSubsystem.h"
#include "Engine/World.h"

// 强制启动指定阶段
static FAutoConsoleCommandWithWorldAndArgs CmdStartPhase(
    TEXT("GamePhase.Start"),
    TEXT("Start a specific game phase. Usage: GamePhase.Start <PhaseClassName>"),
    FConsoleCommandWithWorldAndArgsDelegate::CreateLambda([](const TArray<FString>& Args, UWorld* World)
    {
        if (Args.Num() < 1)
        {
            UE_LOG(LogTemp, Warning, TEXT("Usage: GamePhase.Start <PhaseClassName>"));
            return;
        }

        FString ClassName = Args[0];
        UClass* PhaseClass = FindObject<UClass>(ANY_PACKAGE, *ClassName);

        if (PhaseClass && PhaseClass->IsChildOf(ULyraGamePhaseAbility::StaticClass()))
        {
            ULyraGamePhaseSubsystem* PhaseSubsystem = World->GetSubsystem<ULyraGamePhaseSubsystem>();
            if (PhaseSubsystem)
            {
                PhaseSubsystem->StartPhase(PhaseClass);
                UE_LOG(LogTemp, Log, TEXT("Started phase: %s"), *ClassName);
            }
        }
        else
        {
            UE_LOG(LogTemp, Error, TEXT("Invalid phase class: %s"), *ClassName);
        }
    })
);

// 列出所有活跃阶段
static FAutoConsoleCommandWithWorld CmdListPhases(
    TEXT("GamePhase.List"),
    TEXT("List all active game phases"),
    FConsoleCommandWithWorldDelegate::CreateLambda([](UWorld* World)
    {
        ULyraGamePhaseSubsystem* PhaseSubsystem = World->GetSubsystem<ULyraGamePhaseSubsystem>();
        if (PhaseSubsystem)
        {
            UE_LOG(LogTemp, Log, TEXT("=== Active Game Phases ==="));
            
            // 需要在 Subsystem 中添加公共接口来遍历活跃阶段
            // 或者检查预定义的阶段列表

            // 示例：
            TArray<FGameplayTag> KnownPhases = {
                FGameplayTag::RequestGameplayTag(TEXT("Game.Phase.WaitingToStart")),
                FGameplayTag::RequestGameplayTag(TEXT("Game.Phase.Playing")),
                FGameplayTag::RequestGameplayTag(TEXT("Game.Phase.PostGame"))
            };

            for (const FGameplayTag& PhaseTag : KnownPhases)
            {
                if (PhaseSubsystem->IsPhaseActive(PhaseTag))
                {
                    UE_LOG(LogTemp, Log, TEXT("  - %s"), *PhaseTag.ToString());
                }
            }
        }
    })
);
```

使用方法：
```
// 在编辑器或运行时控制台中：
GamePhase.List
GamePhase.Start GA_GamePhase_Playing_C
```

### 性能优化

#### 1. Observer 列表管理

Observer 列表会持续增长，需要定期清理无效的委托：

```cpp
// 在 LyraGamePhaseSubsystem 中添加清理方法

void ULyraGamePhaseSubsystem::CleanupInvalidObservers()
{
    // 移除无效的观察者（UObject 已被销毁）
    PhaseStartObservers.RemoveAll([](const FPhaseObserver& Observer)
    {
        return !Observer.PhaseCallback.IsBound();
    });

    PhaseEndObservers.RemoveAll([](const FPhaseObserver& Observer)
    {
        return !Observer.PhaseCallback.IsBound();
    });
}
```

可以在每次阶段转换时调用：

```cpp
void ULyraGamePhaseSubsystem::OnBeginPhase(...)
{
    // 定期清理（例如每 10 次阶段变化）
    static int32 CleanupCounter = 0;
    if (++CleanupCounter >= 10)
    {
        CleanupInvalidObservers();
        CleanupCounter = 0;
    }

    // ... 原有逻辑
}
```

#### 2. 避免频繁的阶段切换

频繁切换阶段会导致大量 Ability 激活/结束操作。设计时应该：

- **合并相似阶段**：不要为每个细微状态都创建独立阶段
- **使用状态机内部状态**：在阶段 Ability 内部管理子状态，而不是创建子阶段
- **批处理阶段转换**：避免在同一帧内多次切换阶段

```cpp
// 不推荐：频繁切换
StartPhase(Phase_A);
// 立即
StartPhase(Phase_B);

// 推荐：在 Phase_A 结束回调中启动 Phase_B
StartPhase(Phase_A, FDelegate::CreateLambda([]() {
    StartPhase(Phase_B);
}));
```

#### 3. 优化 IsPhaseActive 查询

`IsPhaseActive` 遍历整个 `ActivePhaseMap`，频繁调用会有性能开销。优化方法：

```cpp
// 方案 1：缓存查询结果（在频繁查询的情况下）

UPROPERTY()
TMap<FGameplayTag, bool> CachedPhaseStates;

bool ULyraGamePhaseSubsystem::IsPhaseActive_Cached(const FGameplayTag& PhaseTag)
{
    if (bool* Cached = CachedPhaseStates.Find(PhaseTag))
    {
        return *Cached;
    }

    bool bIsActive = IsPhaseActive(PhaseTag);
    CachedPhaseStates.Add(PhaseTag, bIsActive);
    return bIsActive;
}

// 在阶段变化时清除缓存
void ULyraGamePhaseSubsystem::OnBeginPhase(...)
{
    CachedPhaseStates.Reset();
    // ...
}
```

```cpp
// 方案 2：使用 Observer 而不是轮询

// 不推荐：每帧检查
void ATick(float DeltaTime)
{
    if (PhaseSubsystem->IsPhaseActive(PlayingTag))
    {
        // ...
    }
}

// 推荐：注册 Observer，状态驱动
void BeginPlay()
{
    PhaseSubsystem->WhenPhaseStartsOrIsActive(PlayingTag, MatchType, [this](const FGameplayTag&) {
        bIsInPlayingPhase = true;
    });
    
    PhaseSubsystem->WhenPhaseEnds(PlayingTag, MatchType, [this](const FGameplayTag&) {
        bIsInPlayingPhase = false;
    });
}

void ATick(float DeltaTime)
{
    if (bIsInPlayingPhase)
    {
        // ...
    }
}
```

#### 4. 网络带宽优化

如果需要同步阶段状态到客户端，使用高效的复制策略：

```cpp
// 使用 GameplayTag 容器复制（已优化）
UPROPERTY(Replicated)
FGameplayTagContainer ActivePhaseTags;  // 自动压缩传输

// 避免复制大量数据
// 不推荐：
UPROPERTY(Replicated)
TArray<FString> ActivePhaseNames;  // 字符串传输效率低

// 不推荐：
UPROPERTY(Replicated)
TArray<TSubclassOf<ULyraGamePhaseAbility>> ActivePhaseClasses;  // 类引用较大
```

#### 5. 内存优化

控制同时活跃的阶段数量：

```cpp
// 在设计上，避免过深的阶段嵌套
// 不推荐：
// Game.Match.InProgress.Round1.Phase1.SubPhase1.DetailedState  (7 层)

// 推荐：
// Game.Match.InProgress (3 层，其他状态在 Ability 内部管理)
```

限制 Observer 数量：

```cpp
// 添加最大 Observer 数量限制
constexpr int32 MaxObserversPerPhase = 100;

void ULyraGamePhaseSubsystem::WhenPhaseStartsOrIsActive(...)
{
    if (PhaseStartObservers.Num() >= MaxObserversPerPhase * 10)  // 假设 10 个常用阶段
    {
        UE_LOG(LogLyraGamePhase, Warning, TEXT("Too many phase observers, consider cleanup"));
        CleanupInvalidObservers();
    }

    // ...
}
```

## 最佳实践与设计模式

### 1. 阶段粒度设计

**原则：** 粗粒度阶段 + 内部状态管理

```cpp
// ✅ 推荐：粗粒度阶段
Game.Playing
  - 内部管理：Warmup / NormalPlay / Overtime（通过变量或枚举）

// ❌ 不推荐：过细粒度阶段
Game.Playing.Warmup.PreparePhase
Game.Playing.Warmup.ActualWarmup
Game.Playing.NormalPlay.EarlyGame
Game.Playing.NormalPlay.MidGame
Game.Playing.NormalPlay.LateGame
```

### 2. 阶段命名约定

使用清晰的层次结构命名：

```
<Game/System>.<MainPhase>.<SubPhase>

例如：
- Game.Match.PreMatch
- Game.Match.InProgress
- Game.Training.Tutorial
- System.Loading.Assets
```

### 3. 单一职责原则

每个阶段 Ability 应该只负责一件事：

```cpp
// ✅ 推荐：单一职责
class UGA_GamePhase_Warmup : public ULyraGamePhaseAbility
{
    // 只负责热身阶段的逻辑：
    // - 限制武器使用
    // - 显示倒计时
    // - 准备就绪检查
};

// ❌ 不推荐：职责过多
class UGA_GamePhase_Everything : public ULyraGamePhaseAbility
{
    // 负责太多事情：
    // - 热身逻辑
    // - 计分逻辑
    // - UI 管理
    // - 网络同步
    // - 数据统计
    // ...
};
```

### 4. 使用组合而非继承

优先使用组件组合来扩展阶段功能：

```cpp
// ✅ 推荐：使用 Component 扩展功能
class UGA_GamePhase_Playing : public ULyraGamePhaseAbility
{
    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UGamePhaseTimerComponent> TimerComponent;

    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UGamePhaseScoreComponent> ScoreComponent;

    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UGamePhaseRespawnComponent> RespawnComponent;
};

// ❌ 不推荐：深层继承
class UGA_GamePhase_Base : public ULyraGamePhaseAbility { };
class UGA_GamePhase_Timed : public UGA_GamePhase_Base { };
class UGA_GamePhase_Scored : public UGA_GamePhase_Timed { };
class UGA_GamePhase_TeamBased : public UGA_GamePhase_Scored { };
class UGA_GamePhase_MySpecificMode : public UGA_GamePhase_TeamBased { };
```

### 5. 事件驱动架构

使用 Observer 模式实现松耦合：

```cpp
// ✅ 推荐：事件驱动
class UMyGameComponent : public UActorComponent
{
    void BeginPlay() override
    {
        // 订阅阶段变化事件
        PhaseSubsystem->WhenPhaseStartsOrIsActive(
            PlayingTag,
            PartialMatch,
            FDelegate::CreateUObject(this, &UMyGameComponent::OnPlayingPhaseStarted)
        );
    }

    void OnPlayingPhaseStarted(const FGameplayTag& PhaseTag)
    {
        // 响应阶段变化
    }
};

// ❌ 不推荐：直接耦合
class UGA_GamePhase_Playing : public ULyraGamePhaseAbility
{
    void ActivateAbility(...) override
    {
        // 直接引用其他系统
        MyGameComponent->OnPlayingStarted();
        UIManager->ShowGameHUD();
        AudioManager->PlayGameMusic();
        // ... 紧耦合，难以维护
    }
};
```

### 6. 错误处理和回退机制

确保阶段切换失败时有合理的回退：

```cpp
void UGA_GamePhase_PreMatch::OnCountdownPhaseEnded(const ULyraGamePhaseAbility* Phase)
{
    // 尝试启动比赛阶段
    ULyraGamePhaseSubsystem* PhaseSubsystem = GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    
    if (InProgressPhaseClass)
    {
        PhaseSubsystem->StartPhase(InProgressPhaseClass);
    }
    else
    {
        // 回退：如果没有配置 InProgress 阶段，记录错误并保持当前状态
        UE_LOG(LogTemp, Error, TEXT("InProgressPhaseClass not set, cannot start match!"));
        
        // 可以返回到 Lobby 阶段或显示错误界面
        // PhaseSubsystem->StartPhase(FallbackPhaseClass);
    }
}
```

## 常见问题与解决方案

### 问题 1：阶段未正确结束

**症状：** 新阶段启动，但旧阶段仍然活跃

**原因：** GamePhaseTag 层次结构设置错误

**解决方案：**
```cpp
// 错误示例：
GamePhaseTag = "Game.Playing";       // 父阶段
NewPhaseTag = "Game.Playing.Round2";  // 子阶段
// Round1 没有被结束，因为 Round2 匹配 Playing

// 正确做法：
// 如果 Round1 和 Round2 应该互斥，它们应该是同级：
Round1Tag = "Game.Playing.Round1";
Round2Tag = "Game.Playing.Round2";
// 启动 Round2 会自动结束 Round1
```

### 问题 2：Observer 回调未触发

**症状：** 注册了 WhenPhaseStartsOrIsActive，但回调没有执行

**可能原因：**
1. MatchType 设置错误（ExactMatch vs PartialMatch）
2. 委托绑定的对象已被销毁
3. 阶段在注册之前就已经启动

**解决方案：**
```cpp
// 确保使用正确的 MatchType
PhaseSubsystem->WhenPhaseStartsOrIsActive(
    FGameplayTag::RequestGameplayTag(TEXT("Game.Playing")),
    EPhaseTagMatchType::PartialMatch,  // 使用 PartialMatch 匹配子阶段
    Callback
);

// 使用 UObject 而非 Lambda（避免悬空指针）
FLyraGamePhaseTagDelegate::CreateUObject(this, &UMyClass::OnPhaseStarted)

// 如果阶段可能已经活跃，WhenPhaseStartsOrIsActive 会立即触发回调
// 无需额外检查
```

### 问题 3：网络同步问题

**症状：** 服务器阶段已切换，但客户端不知道

**原因：** 阶段系统本身不自动同步到客户端

**解决方案：**
```cpp
// 在 GameState 上添加复制的 GameplayTag 容器
UPROPERTY(Replicated)
FGameplayTagContainer CurrentPhaseTags;

// 在阶段 Ability 中更新
void UMyGamePhaseAbility::ActivateAbility(...)
{
    Super::ActivateAbility(...);
    
    if (HasAuthority())
    {
        ALyraGameState* GS = GetWorld()->GetGameState<ALyraGameState>();
        GS->CurrentPhaseTags.AddTag(GamePhaseTag);
    }
}

// 客户端监听复制
void ALyraGameState::OnRep_CurrentPhaseTags()
{
    // 客户端逻辑
    OnPhaseTagsChanged.Broadcast(CurrentPhaseTags);
}
```

### 问题 4：阶段切换过于频繁导致性能问题

**症状：** 游戏卡顿，大量阶段激活/结束日志

**解决方案：**
```cpp
// 1. 合并阶段，减少切换次数
// 2. 在阶段内部使用状态变量而非创建子阶段
class UGA_GamePhase_Playing : public ULyraGamePhaseAbility
{
    enum class ESubPhase { Warmup, NormalPlay, Overtime };
    ESubPhase CurrentSubPhase;  // 内部管理，不创建新的 Phase Ability
};

// 3. 使用定时器批处理阶段切换
void DelayedPhaseTransition()
{
    FTimerHandle Handle;
    GetWorld()->GetTimerManager().SetTimer(Handle, []() {
        // 在下一帧或延迟后切换
        PhaseSubsystem->StartPhase(NextPhase);
    }, 0.1f, false);
}
```

### 问题 5：阶段无法在 PIE（Play In Editor）中正常工作

**症状：** 在编辑器中玩家测试时阶段系统不工作

**原因：** Subsystem 创建条件或 GameState ASC 未正确初始化

**解决方案：**
```cpp
// 检查 Subsystem 的 DoesSupportWorldType
bool ULyraGamePhaseSubsystem::DoesSupportWorldType(const EWorldType::Type WorldType) const
{
    // 确保支持 PIE
    return WorldType == EWorldType::Game || WorldType == EWorldType::PIE;
}

// 确保 GameState 有 ASC
void AMyGameState::PostInitializeComponents()
{
    Super::PostInitializeComponents();
    
    // 创建 ASC（如果还没有）
    if (!AbilitySystemComponent)
    {
        AbilitySystemComponent = CreateDefaultSubobject<ULyraAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
        AbilitySystemComponent->SetIsReplicated(true);
    }
}
```

## 总结

Lyra 的游戏阶段管理系统通过将阶段建模为 Gameplay Ability，创造了一个强大、灵活且易于扩展的游戏流程控制架构。

### 核心优势

1. **层次化设计**：通过 GameplayTag 层次结构实现父子阶段嵌套，兄弟阶段互斥
2. **GAS 集成**：利用 Gameplay Ability System 的成熟机制处理生命周期和网络同步
3. **Observer 模式**：提供灵活的事件监听机制，支持精确匹配和部分匹配
4. **服务器权威**：确保网络游戏中的阶段状态一致性
5. **易于调试**：日志系统和可视化工具帮助开发者快速定位问题

### 适用场景

- **多人竞技游戏**：大厅→准备→比赛→结算
- **回合制游戏**：回合开始→行动阶段→结算阶段→回合结束
- **教学系统**：介绍→练习→测试→完成
- **剧情关卡**：过场→探索→战斗→剧情

### 设计原则总结

1. **粗粒度阶段**：避免过细的阶段划分，使用内部状态管理子状态
2. **单一职责**：每个阶段只负责一个明确的游戏流程部分
3. **事件驱动**：使用 Observer 模式而非轮询，实现松耦合
4. **错误处理**：为阶段切换失败提供合理的回退机制
5. **性能优先**：控制阶段切换频率，优化 Observer 列表管理

### 扩展方向

- **可保存的阶段状态**：支持游戏存档和断线重连
- **阶段历史记录**：追踪阶段转换历史，便于调试和分析
- **可视化编辑器**：创建节点编辑器来设计阶段流程
- **分析系统集成**：追踪玩家在各阶段的行为数据
- **动态阶段加载**：根据游戏模式动态加载阶段配置

### 关键要点

**✅ 推荐做法：**
- 使用 Observer 模式监听阶段变化
- 通过 GameplayTag 层次结构组织阶段
- 在 GameState 的 ASC 上管理阶段 Ability
- 使用 GameplayCue 通知客户端阶段变化
- 定期清理无效的 Observer

**❌ 避免做法：**
- 创建过深的阶段嵌套（> 4 层）
- 在同一帧内频繁切换阶段
- 在客户端执行阶段转换逻辑
- 使用 IsPhaseActive 进行高频轮询
- 硬编码阶段 Tag 字符串

通过深入理解和正确使用 GamePhase 系统，开发者可以构建出结构清晰、易于维护、功能强大的游戏流程管理逻辑。

---

## 参考资源

### Lyra 源码位置

- **核心实现**：`LyraGame/Source/LyraGame/AbilitySystem/Phases/`
  - `LyraGamePhaseSubsystem.h/cpp`
  - `LyraGamePhaseAbility.h/cpp`
  - `LyraGamePhaseLog.h`

### 相关文档

- [UE5 Gameplay Ability System 官方文档](https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-for-unreal-engine/)
- [UE5 Gameplay Tags 文档](https://docs.unrealengine.com/5.0/en-US/gameplay-tags-in-unreal-engine/)
- [UE5 World Subsystems](https://docs.unrealengine.com/5.0/en-US/world-subsystems-in-unreal-engine/)

### 相关教程

- [UE5 Lyra Experience 系统详解](./01-experience-system.md)
- [UE5 Gameplay Ability System 基础](./06-gas-basics.md)
- [UE5 Gameplay Ability System 高级应用](./07-gas-advanced.md)
- [UE5 Lyra Team 系统](./12-team-system.md)

### 社区资源

- [Lyra Starter Game - Epic Games](https://www.unrealengine.com/marketplace/en-US/product/lyra)
- [Unreal Engine Forums - Lyra Discussion](https://forums.unrealengine.com/)
- [Unreal Slackers Discord - Lyra Channel](https://unrealslackers.org/)

---

**文档版本**：v1.0  
**最后更新**：2024年2月  
**Lyra 版本**：UE 5.3+  
**作者**：UE5 Lyra 教程系列

**字数统计**：约 10,200 字
