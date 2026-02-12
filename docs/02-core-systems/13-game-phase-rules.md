# 游戏阶段与规则管理：Game Phase System 与 Game Mode 深度解析

## 目录

- [1. 游戏阶段系统概述](#1-游戏阶段系统概述)
  - [1.1 什么是游戏阶段](#11-什么是游戏阶段)
  - [1.2 为什么需要游戏阶段系统](#12-为什么需要游戏阶段系统)
  - [1.3 Lyra 的游戏阶段架构](#13-lyra-的游戏阶段架构)
  - [1.4 游戏阶段与其他系统的关系](#14-游戏阶段与其他系统的关系)
- [2. Game Phase Subsystem 深度解析](#2-game-phase-subsystem-深度解析)
  - [2.1 ULyraGamePhaseSubsystem 架构](#21-ulyragamephasesubsystem-架构)
  - [2.2 Phase 的定义与数据结构](#22-phase-的定义与数据结构)
  - [2.3 Phase 状态机制](#23-phase-状态机制)
  - [2.4 Phase 观察者模式](#24-phase-观察者模式)
  - [2.5 网络同步机制](#25-网络同步机制)
- [3. Phase Ability 阶段技能系统](#3-phase-ability-阶段技能系统)
  - [3.1 什么是 Phase Ability](#31-什么是-phase-ability)
  - [3.2 ULyraGamePhaseAbility 详解](#32-ulyragamephaseability-详解)
  - [3.3 Phase Ability 的生命周期](#33-phase-ability-的生命周期)
  - [3.4 创建自定义 Phase Ability](#34-创建自定义-phase-ability)
  - [3.5 Phase Ability 最佳实践](#35-phase-ability-最佳实践)
- [4. Phase Tag 管理系统](#4-phase-tag-管理系统)
  - [4.1 Phase Tag 体系设计](#41-phase-tag-体系设计)
  - [4.2 Phase Tag 的应用场景](#42-phase-tag-的应用场景)
  - [4.3 Phase Tag Query 高级用法](#43-phase-tag-query-高级用法)
  - [4.4 动态 Phase Tag 管理](#44-动态-phase-tag-管理)
- [5. 标准游戏阶段实现](#5-标准游戏阶段实现)
  - [5.1 等待玩家阶段](#51-等待玩家阶段)
  - [5.2 倒计时准备阶段](#52-倒计时准备阶段)
  - [5.3 游戏进行阶段](#53-游戏进行阶段)
  - [5.4 结算与展示阶段](#54-结算与展示阶段)
  - [5.5 阶段转换逻辑](#55-阶段转换逻辑)
- [6. 游戏规则系统架构](#6-游戏规则系统架构)
  - [6.1 UE 的 Game Mode 体系回顾](#61-ue-的-game-mode-体系回顾)
  - [6.2 ALyraGameMode 架构深度分析](#62-alyragamemode-架构深度分析)
  - [6.3 Game State 与 Player State](#63-game-state-与-player-state)
  - [6.4 Experience 与 Game Mode 的集成](#64-experience-与-game-mode-的集成)
  - [6.5 Game Mode 的初始化流程](#65-game-mode-的初始化流程)
- [7. 自定义游戏规则开发](#7-自定义游戏规则开发)
  - [7.1 规则系统设计原则](#71-规则系统设计原则)
  - [7.2 创建自定义 Game Mode](#72-创建自定义-game-mode)
  - [7.3 实现规则配置系统](#73-实现规则配置系统)
  - [7.4 规则验证与反外挂](#74-规则验证与反外挂)
  - [7.5 规则热更新机制](#75-规则热更新机制)
- [8. 胜负判定系统](#8-胜负判定系统)
  - [8.1 胜利条件设计](#81-胜利条件设计)
  - [8.2 实时胜负判定逻辑](#82-实时胜负判定逻辑)
  - [8.3 复杂胜负条件处理](#83-复杂胜负条件处理)
  - [8.4 提前结束机制](#84-提前结束机制)
  - [8.5 平局与加时赛](#85-平局与加时赛)
- [9. 游戏事件系统](#9-游戏事件系统)
  - [9.1 Gameplay Event 消息系统](#91-gameplay-event-消息系统)
  - [9.2 事件广播机制](#92-事件广播机制)
  - [9.3 事件监听与响应](#93-事件监听与响应)
  - [9.4 常见游戏事件实现](#94-常见游戏事件实现)
  - [9.5 UI 事件响应系统](#95-ui-事件响应系统)
- [10. 计分系统深度实现](#10-计分系统深度实现)
  - [10.1 Player State 扩展](#101-player-state-扩展)
  - [10.2 分数统计架构](#102-分数统计架构)
  - [10.3 多维度统计系统](#103-多维度统计系统)
  - [10.4 实时分数同步](#104-实时分数同步)
  - [10.5 分数持久化](#105-分数持久化)
- [11. 排行榜与 MVP 系统](#11-排行榜与-mvp-系统)
  - [11.1 排行榜数据结构](#111-排行榜数据结构)
  - [11.2 动态排行榜更新](#112-动态排行榜更新)
  - [11.3 MVP 评选算法](#113-mvp-评选算法)
  - [11.4 排行榜 UI 实现](#114-排行榜-ui-实现)
  - [11.5 全局排行榜集成](#115-全球排行榜集成)
- [12. 实战案例：TDM 团队死斗模式](#12-实战案例tdm-团队死斗模式)
  - [12.1 TDM 模式需求分析](#121-tdm-模式需求分析)
  - [12.2 实现 TDM Game Mode](#122-实现-tdm-game-mode)
  - [12.3 团队分配与平衡](#123-团队分配与平衡)
  - [12.4 击杀-死亡统计](#124-击杀-死亡统计)
  - [12.5 游戏阶段控制](#125-游戏阶段控制)
  - [12.6 完整测试流程](#126-完整测试流程)
- [13. 实战案例：Battle Royale 大逃杀模式](#13-实战案例battle-royale-大逃杀模式)
  - [13.1 BR 模式核心机制](#131-br-模式核心机制)
  - [13.2 安全区收缩系统](#132-安全区收缩系统)
  - [13.3 毒圈伤害实现](#133-毒圈伤害实现)
  - [13.4 空投补给系统](#134-空投补给系统)
  - [13.5 存活人数追踪](#135-存活人数追踪)
  - [13.6 最终胜利判定](#136-最终胜利判定)
- [14. 实战案例：CTF 夺旗模式](#14-实战案例ctf-夺旗模式)
  - [14.1 CTF 模式设计](#141-ctf-模式设计)
  - [14.2 旗帜对象实现](#142-旗帜对象实现)
  - [14.3 夺旗与护旗逻辑](#143-夺旗与护旗逻辑)
  - [14.4 得分与回合机制](#144-得分与回合机制)
  - [14.5 特殊事件处理](#145-特殊事件处理)
- [15. 实战案例：竞技场匹配系统](#15-实战案例竞技场匹配系统)
  - [15.1 匹配系统架构](#151-匹配系统架构)
  - [15.2 ELO 评级算法](#152-elo-评级算法)
  - [15.3 匹配队列实现](#153-匹配队列实现)
  - [15.4 段位与赛季系统](#154-段位与赛季系统)
  - [15.5 防作弊机制](#155-防作弊机制)
- [16. 高级主题](#16-高级主题)
  - [16.1 动态难度调整 (DDA)](#161-动态难度调整-dda)
  - [16.2 观战系统集成](#162-观战系统集成)
  - [16.3 Replay 录制与回放](#163-replay-录制与回放)
  - [16.4 服务器性能优化](#164-服务器性能优化)
  - [16.5 多模式切换机制](#165-多模式切换机制)
- [17. 调试与测试](#17-调试与测试)
  - [17.1 阶段系统调试工具](#171-阶段系统调试工具)
  - [17.2 规则验证测试](#172-规则验证测试)
  - [17.3 网络环境模拟](#173-网络环境模拟)
  - [17.4 压力测试方案](#174-压力测试方案)
  - [17.5 常见问题排查](#175-常见问题排查)
- [18. 最佳实践与设计模式](#18-最佳实践与设计模式)
  - [18.1 阶段系统设计原则](#181-阶段系统设计原则)
  - [18.2 规则系统架构模式](#182-规则系统架构模式)
  - [18.3 性能优化清单](#183-性能优化清单)
  - [18.4 可维护性建议](#184-可维护性建议)
  - [18.5 团队协作规范](#185-团队协作规范)
- [19. 总结与展望](#19-总结与展望)

---

## 1. 游戏阶段系统概述

### 1.1 什么是游戏阶段

**游戏阶段（Game Phase）**是多人游戏中用于管理游戏流程的核心机制。它将一场游戏从开始到结束划分为多个清晰的阶段，每个阶段有不同的规则、行为和表现。

想象一场《守望先锋》的比赛：

```
准备阶段 → 选择英雄 → 等待倒计时 → 游戏进行 → 胜负判定 → 结算展示
```

每个阶段都有特定的：
- **游戏逻辑**：玩家能做什么、不能做什么
- **UI 表现**：显示哪些界面、提示信息
- **系统行为**：计分、计时、生成对象等
- **网络同步**：如何在所有客户端保持一致

在 Lyra 中，游戏阶段系统通过 **ULyraGamePhaseSubsystem** 实现，它提供了一套完整的、数据驱动的、网络友好的阶段管理方案。

### 1.2 为什么需要游戏阶段系统

#### 1.2.1 传统流程管理的问题

**问题 1：逻辑散乱难以维护**

```cpp
// 传统做法：逻辑分散在各处
void AMyGameMode::Tick(float DeltaTime)
{
    if (bWaitingForPlayers)
    {
        // 等待玩家逻辑
        if (GetNumPlayers() >= MinPlayers)
        {
            StartCountdown();
        }
    }
    else if (bCountingDown)
    {
        // 倒计时逻辑
        CountdownTime -= DeltaTime;
        if (CountdownTime <= 0)
        {
            StartMatch();
        }
    }
    else if (bMatchInProgress)
    {
        // 游戏中逻辑
        CheckWinCondition();
    }
    // ... 越来越多的 if-else
}
```

**问题分析**：
- 🔴 所有阶段逻辑耦合在一起
- 🔴 难以添加新阶段或修改顺序
- 🔴 状态转换条件难以理解
- 🔴 无法复用阶段逻辑

**问题 2：网络同步复杂**

```cpp
// 每个阶段都需要单独的复制变量
UPROPERTY(Replicated)
bool bWaitingForPlayers;

UPROPERTY(Replicated)
bool bCountingDown;

UPROPERTY(Replicated)
float CountdownTime;

// ... 大量的 RPC 调用
UFUNCTION(Server, Reliable)
void ServerStartCountdown();

UFUNCTION(NetMulticast, Reliable)
void MulticastOnCountdownStarted();
```

**问题分析**：
- 🔴 每个阶段需要多个复制变量
- 🔴 RPC 调用容易出错
- 🔴 客户端与服务器状态不一致
- 🔴 难以调试网络问题

**问题 3：UI 与逻辑耦合**

```cpp
void AMyGameMode::StartCountdown()
{
    bCountingDown = true;
    
    // 直接操作 UI - 错误示范！
    for (auto PC : PlayerControllers)
    {
        PC->ShowCountdownWidget();
    }
}
```

**问题分析**：
- 🔴 游戏逻辑直接操作 UI
- 🔴 难以支持不同的 UI 风格
- 🔴 测试困难（需要 UI 存在）
- 🔴 无法在 Dedicated Server 运行

#### 1.2.2 Lyra 游戏阶段系统的优势

**优势 1：清晰的状态管理**

```cpp
// Lyra 的做法：每个阶段是独立的 Ability
UCLASS()
class UGamePhase_WaitingToStart : public ULyraGamePhaseAbility
{
    GENERATED_BODY()
    
protected:
    virtual void OnPhaseBegin() override
    {
        // 阶段开始时的逻辑
        DisablePlayerInput();
        ShowLobbyUI();
    }
    
    virtual void OnPhaseEnd() override
    {
        // 阶段结束时的清理
        HideLobbyUI();
    }
};
```

**优势说明**：
- ✅ 每个阶段逻辑独立、可复用
- ✅ 清晰的开始和结束钩子
- ✅ 易于测试和调试
- ✅ 符合单一职责原则

**优势 2：基于 GAS 的自动网络同步**

```cpp
// Phase 的激活自动同步到所有客户端
void ULyraGamePhaseSubsystem::StartPhase(TSubclassOf<ULyraGamePhaseAbility> PhaseAbility)
{
    // 通过 GAS 激活 Ability
    // GAS 会自动处理网络同步！
    AbilitySystemComponent->GiveAbilityAndActivateOnce(
        FGameplayAbilitySpec(PhaseAbility)
    );
}
```

**优势说明**：
- ✅ 无需手写网络代码
- ✅ 利用 GAS 成熟的同步机制
- ✅ 自动处理权限验证
- ✅ 支持客户端预测

**优势 3：数据驱动的阶段配置**

```cpp
// 在 Data Asset 中配置阶段流程
UPROPERTY(EditDefaultsOnly)
TArray<FGamePhaseEntry> PhaseSequence;

// FGamePhaseEntry 结构
USTRUCT()
struct FGamePhaseEntry
{
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere)
    TSubclassOf<ULyraGamePhaseAbility> PhaseAbility;
    
    UPROPERTY(EditAnywhere)
    FGameplayTag PhaseTag;
    
    UPROPERTY(EditAnywhere)
    FPhaseTransitionCondition TransitionCondition;
};
```

**优势说明**：
- ✅ 策划可视化配置阶段
- ✅ 不同模式复用阶段逻辑
- ✅ 热更新支持
- ✅ 易于 A/B 测试

**优势 4：事件驱动的解耦设计**

```cpp
// UI 监听阶段变化
void UMyHUD::ListenToPhaseChanges()
{
    PhaseSubsystem->OnPhaseStarted.AddDynamic(
        this, &UMyHUD::HandlePhaseStarted
    );
}

void UMyHUD::HandlePhaseStarted(const FGameplayTag& PhaseTag)
{
    if (PhaseTag.MatchesTag(TAG_GamePhase_Countdown))
    {
        ShowCountdownWidget();
    }
}
```

**优势说明**：
- ✅ UI 与逻辑完全解耦
- ✅ 支持多个监听者
- ✅ 易于扩展和维护
- ✅ 可在运行时动态绑定

### 1.3 Lyra 的游戏阶段架构

#### 1.3.1 核心组件关系图

```
┌─────────────────────────────────────────────────────┐
│                 UWorld (游戏世界)                    │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────────────────────────────────┐     │
│  │  ULyraGamePhaseSubsystem (阶段子系统)     │     │
│  │  ┌────────────────────────────────────┐  │     │
│  │  │  ActivePhases: TArray              │  │     │
│  │  │  PhaseObservers: TMap              │  │     │
│  │  │  PhaseHistory: TArray              │  │     │
│  │  └────────────────────────────────────┘  │     │
│  │                    ↕                      │     │
│  │  ┌────────────────────────────────────┐  │     │
│  │  │  Phase Ability 1 (等待玩家)         │  │     │
│  │  │  • OnPhaseBegin()                  │  │     │
│  │  │  • OnPhaseEnd()                    │  │     │
│  │  │  • CheckTransitionCondition()      │  │     │
│  │  └────────────────────────────────────┘  │     │
│  │                                           │     │
│  │  ┌────────────────────────────────────┐  │     │
│  │  │  Phase Ability 2 (倒计时)           │  │     │
│  │  └────────────────────────────────────┘  │     │
│  │                                           │     │
│  │  ┌────────────────────────────────────┐  │     │
│  │  │  Phase Ability 3 (游戏中)           │  │     │
│  │  └────────────────────────────────────┘  │     │
│  └──────────────────────────────────────────┘     │
│                    ↕                               │
│  ┌──────────────────────────────────────────┐     │
│  │  ALyraGameMode (游戏模式)                 │     │
│  │  • InitGame()                            │     │
│  │  • HandleMatchHasStarted()               │     │
│  │  • CheckWinCondition()                   │     │
│  └──────────────────────────────────────────┘     │
│                    ↕                               │
│  ┌──────────────────────────────────────────┐     │
│  │  ALyraGameState (游戏状态)                │     │
│  │  • CurrentPhaseTag                       │     │
│  │  • MatchTime                             │     │
│  │  • TeamScores                            │     │
│  └──────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────┘

           监听阶段变化 ↓

┌─────────────────────────────────────────────────────┐
│              观察者 (Observers)                      │
├─────────────────────────────────────────────────────┤
│  • UI Widgets (显示阶段相关界面)                     │
│  • Game Mode Components (执行特定逻辑)               │
│  • Audio Manager (播放音效/音乐)                     │
│  • AI Controllers (调整 Bot 行为)                    │
│  • Analytics System (记录数据)                       │
└─────────────────────────────────────────────────────┘
```

#### 1.3.2 数据流向

```
1. Game Mode 初始化
   ↓
2. Experience 加载完成
   ↓
3. Game Phase Subsystem 启动第一个阶段
   ↓
4. Phase Ability 激活 (通过 GAS)
   ↓
5. OnPhaseBegin() 执行
   ↓
6. 广播 OnPhaseStarted 事件
   ↓
7. 观察者响应阶段变化
   ↓
8. 检测转换条件
   ↓
9. 条件满足 → 结束当前阶段
   ↓
10. OnPhaseEnd() 执行
   ↓
11. 广播 OnPhaseEnded 事件
   ↓
12. 启动下一个阶段 (回到步骤 3)
```

#### 1.3.3 核心类图

```cpp
// 1. 阶段子系统 (核心管理者)
UCLASS()
class ULyraGamePhaseSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()
    
public:
    // 启动一个新阶段
    void StartPhase(TSubclassOf<ULyraGamePhaseAbility> PhaseAbility, 
                    AActor* PhaseInstigator);
    
    // 检查当前是否在某个阶段
    bool IsPhaseActive(const FGameplayTag& PhaseTag) const;
    
    // 等待某个阶段开始
    void WaitForPhaseStart(const FGameplayTag& PhaseTag, 
                          FPhaseStartedDelegate OnStarted);
    
    // 事件委托
    UPROPERTY(BlueprintAssignable)
    FOnPhaseChangedDelegate OnPhaseStarted;
    
    UPROPERTY(BlueprintAssignable)
    FOnPhaseChangedDelegate OnPhaseEnded;
    
private:
    // 当前激活的阶段列表
    UPROPERTY()
    TArray<FGamePhaseEntry> ActivePhases;
    
    // 阶段观察者
    TMap<FGameplayTag, TArray<FPhaseObserver>> PhaseObservers;
};

// 2. 阶段技能基类
UCLASS()
class ULyraGamePhaseAbility : public ULyraGameplayAbility
{
    GENERATED_BODY()
    
public:
    // 阶段标识
    UPROPERTY(EditDefaultsOnly, Category = "Phase")
    FGameplayTag PhaseTag;
    
protected:
    // 阶段开始时调用
    virtual void OnPhaseBegin();
    
    // 阶段结束时调用
    virtual void OnPhaseEnd();
    
    // 检查是否可以转换到下一阶段
    virtual bool CanTransitionToNextPhase() const;
    
    // 通知子系统阶段开始
    UFUNCTION(BlueprintCallable)
    void NotifyPhaseStarted();
    
    // 通知子系统阶段结束
    UFUNCTION(BlueprintCallable)
    void NotifyPhaseEnded();
};

// 3. 阶段配置数据
USTRUCT(BlueprintType)
struct FGamePhaseEntry
{
    GENERATED_BODY()
    
    // 阶段 Ability 类
    UPROPERTY(EditAnywhere)
    TSubclassOf<ULyraGamePhaseAbility> PhaseAbility;
    
    // 阶段标签
    UPROPERTY(EditAnywhere)
    FGameplayTag PhaseTag;
    
    // 自动转换条件
    UPROPERTY(EditAnywhere)
    FPhaseTransitionCondition TransitionCondition;
    
    // 最短持续时间 (秒)
    UPROPERTY(EditAnywhere)
    float MinDuration = 0.0f;
    
    // 最长持续时间 (秒，0 表示无限)
    UPROPERTY(EditAnywhere)
    float MaxDuration = 0.0f;
};

// 4. 阶段转换条件
USTRUCT(BlueprintType)
struct FPhaseTransitionCondition
{
    GENERATED_BODY()
    
    // 基于玩家数量
    UPROPERTY(EditAnywhere)
    int32 RequiredPlayerCount = 0;
    
    // 基于时间
    UPROPERTY(EditAnywhere)
    bool bUseTimer = false;
    
    UPROPERTY(EditAnywhere, meta = (EditCondition = "bUseTimer"))
    float TimerDuration = 10.0f;
    
    // 基于 Gameplay Tag
    UPROPERTY(EditAnywhere)
    FGameplayTagContainer RequiredTags;
    
    // 自定义条件函数
    UPROPERTY(EditAnywhere)
    TSubclassOf<UPhaseTransitionRule> CustomRule;
};
```

### 1.4 游戏阶段与其他系统的关系

#### 1.4.1 与 Experience 系统的集成

```cpp
// Experience 定义中包含阶段配置
UCLASS()
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()
    
public:
    // 游戏阶段配置
    UPROPERTY(EditDefaultsOnly, Category = "Phases")
    TArray<FGamePhaseEntry> GamePhases;
    
    // 默认的第一个阶段
    UPROPERTY(EditDefaultsOnly, Category = "Phases")
    TSubclassOf<ULyraGamePhaseAbility> InitialPhase;
};

// Game Mode 在 Experience 加载后启动阶段
void ALyraGameMode::OnExperienceLoaded(const ULyraExperienceDefinition* Experience)
{
    // 获取阶段子系统
    ULyraGamePhaseSubsystem* PhaseSubsystem = 
        GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    
    // 启动初始阶段
    if (Experience->InitialPhase)
    {
        PhaseSubsystem->StartPhase(Experience->InitialPhase, this);
    }
}
```

**集成要点**：
- 🔄 Experience 定义了完整的阶段序列
- 🔄 不同 Experience 可以有不同的阶段流程
- 🔄 支持动态切换 Experience 和阶段
- 🔄 阶段配置可以通过 Game Feature 扩展

#### 1.4.2 与 GAS 的深度集成

```cpp
// Phase Ability 继承自 Gameplay Ability
// 自动获得 GAS 的所有特性：

// 1. 网络同步
// - Ability 的激活/结束自动同步
// - 不需要写任何网络代码

// 2. Gameplay Tag 管理
void ULyraGamePhaseAbility::OnPhaseBegin()
{
    // 给 ASC 添加 Phase Tag
    AddAbilityTag(PhaseTag);
    
    // 其他系统可以通过 Tag 检测当前阶段
    // 例如：禁止在等待阶段使用武器
}

// 3. Gameplay Effect 集成
UPROPERTY(EditDefaultsOnly)
TSubclassOf<UGameplayEffect> PhaseEffect;

void ULyraGamePhaseAbility::ActivateAbility(...)
{
    // 应用阶段相关的 GE
    // 例如：倒计时阶段冻结玩家移动
    ApplyGameplayEffectToOwner(PhaseEffect);
}

// 4. Ability Task 支持
void ULyraGamePhaseAbility::OnPhaseBegin()
{
    // 使用 Ability Task 等待事件
    UAbilityTask_WaitGameplayEvent* WaitTask = 
        UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
            this, TAG_Event_AllPlayersReady
        );
    
    WaitTask->EventReceived.AddDynamic(
        this, &ULyraGamePhaseAbility::OnAllPlayersReady
    );
    
    WaitTask->ReadyForActivation();
}
```

**集成优势**：
- ✅ 复用 GAS 的网络架构
- ✅ 统一的 Tag 系统
- ✅ 强大的条件判断能力
- ✅ 丰富的 Ability Task 库

#### 1.4.3 与 Game Mode / Game State 的协作

```cpp
// Game Mode 控制阶段转换
void ALyraGameMode::HandleMatchHasStarted()
{
    Super::HandleMatchHasStarted();
    
    // 从 "等待玩家" 阶段转换到 "倒计时" 阶段
    ULyraGamePhaseSubsystem* PhaseSubsystem = 
        GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
    
    PhaseSubsystem->StartPhase(CountdownPhaseClass, this);
}

// Game State 存储阶段相关数据
UCLASS()
class ALyraGameState : public AModularGameStateBase
{
    GENERATED_BODY()
    
public:
    // 当前阶段标签 (复制给客户端)
    UPROPERTY(ReplicatedUsing = OnRep_CurrentPhaseTag)
    FGameplayTag CurrentPhaseTag;
    
    // 阶段开始时间
    UPROPERTY(Replicated)
    float PhaseStartTime;
    
    // 获取阶段剩余时间
    UFUNCTION(BlueprintCallable)
    float GetPhaseTimeRemaining() const;
    
private:
    UFUNCTION()
    void OnRep_CurrentPhaseTag();
};
```

**协作模式**：
- 📊 Game Mode：负责阶段转换决策
- 📊 Game State：存储阶段状态数据
- 📊 Phase Subsystem：管理阶段生命周期
- 📊 Phase Ability：执行具体阶段逻辑

#### 1.4.4 与 UI 系统的通信

```cpp
// UI Widget 监听阶段变化
UCLASS()
class UMyMatchStateWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()
    
protected:
    virtual void NativeConstruct() override
    {
        Super::NativeConstruct();
        
        // 订阅阶段事件
        ULyraGamePhaseSubsystem* PhaseSubsystem = 
            GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
        
        PhaseSubsystem->OnPhaseStarted.AddDynamic(
            this, &UMyMatchStateWidget::OnPhaseStarted
        );
    }
    
    UFUNCTION()
    void OnPhaseStarted(const FGameplayTag& PhaseTag)
    {
        if (PhaseTag.MatchesTag(TAG_GamePhase_Countdown))
        {
            // 显示倒计时 UI
            ShowCountdownScreen();
        }
        else if (PhaseTag.MatchesTag(TAG_GamePhase_InProgress))
        {
            // 显示游戏 HUD
            ShowGameplayHUD();
        }
        else if (PhaseTag.MatchesTag(TAG_GamePhase_PostMatch))
        {
            // 显示结算界面
            ShowScoreboard();
        }
    }
};
```

**通信方式**：
- 🎨 事件驱动：UI 监听阶段事件
- 🎨 数据绑定：UI 读取 Game State 数据
- 🎨 完全解耦：UI 不需要知道阶段实现细节
- 🎨 灵活替换：可以轻松更换 UI 风格

---

## 2. Game Phase Subsystem 深度解析

### 2.1 ULyraGamePhaseSubsystem 架构

**ULyraGamePhaseSubsystem** 是整个阶段系统的核心管理者，它是一个 **World Subsystem**，在游戏世界创建时自动初始化，生命周期与 World 一致。

#### 2.1.1 完整类定义

```cpp
// LyraGamePhaseSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/WorldSubsystem.h"
#include "GameplayTagContainer.h"
#include "LyraGamePhaseSubsystem.generated.h"

class ULyraGamePhaseAbility;
class UAbilitySystemComponent;

/**
 * 阶段观察者委托
 * 当阶段开始或结束时调用
 */
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(
    FLyraGamePhaseTagDelegate, 
    const FGameplayTag&, PhaseTag
);

/**
 * 阶段观察者句柄
 * 用于取消观察
 */
USTRUCT()
struct FLyraGamePhaseObserverHandle
{
    GENERATED_BODY()
    
    friend class ULyraGamePhaseSubsystem;
    
    bool IsValid() const { return ID != 0; }
    void Invalidate() { ID = 0; }
    
private:
    UPROPERTY()
    int32 ID = 0;
};

/**
 * 激活的阶段数据
 */
USTRUCT()
struct FLyraGamePhaseEntry
{
    GENERATED_BODY()
    
    // 阶段标签
    UPROPERTY()
    FGameplayTag PhaseTag;
    
    // 阶段 Ability 实例
    UPROPERTY()
    ULyraGamePhaseAbility* PhaseAbility = nullptr;
    
    // 阶段开始时间
    UPROPERTY()
    float StartTime = 0.0f;
    
    // 阶段触发者
    UPROPERTY()
    TWeakObjectPtr<AActor> Instigator;
};

/**
 * Lyra 游戏阶段子系统
 * 
 * 职责：
 * 1. 管理游戏阶段的生命周期
 * 2. 提供阶段查询接口
 * 3. 管理阶段观察者
 * 4. 与 GAS 集成实现网络同步
 */
UCLASS()
class LYRAGAME_API ULyraGamePhaseSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()
    
public:
    ULyraGamePhaseSubsystem();
    
    //~USubsystem interface
    virtual bool ShouldCreateSubsystem(UObject* Outer) const override;
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
    //~End of USubsystem interface
    
    // ============================================================
    // 阶段控制接口
    // ============================================================
    
    /**
     * 启动一个新的游戏阶段
     * @param PhaseAbility 阶段 Ability 类
     * @param PhaseInstigator 触发阶段的 Actor (通常是 Game Mode)
     * @return 是否成功启动
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Lyra|GamePhase")
    bool StartPhase(TSubclassOf<ULyraGamePhaseAbility> PhaseAbility,
                    AActor* PhaseInstigator = nullptr);
    
    /**
     * 结束指定的游戏阶段
     * @param PhaseTag 要结束的阶段标签
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Lyra|GamePhase")
    void EndPhase(const FGameplayTag& PhaseTag);
    
    /**
     * 结束所有激活的阶段
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Lyra|GamePhase")
    void EndAllPhases();
    
    // ============================================================
    // 阶段查询接口
    // ============================================================
    
    /**
     * 检查指定阶段是否激活
     * @param PhaseTag 阶段标签
     * @return 是否激活
     */
    UFUNCTION(BlueprintCallable, Category = "Lyra|GamePhase")
    bool IsPhaseActive(const FGameplayTag& PhaseTag) const;
    
    /**
     * 获取当前激活的所有阶段标签
     */
    UFUNCTION(BlueprintCallable, Category = "Lyra|GamePhase")
    void GetActivePhases(TArray<FGameplayTag>& OutPhases) const;
    
    /**
     * 获取指定阶段的启动时间
     * @param PhaseTag 阶段标签
     * @return 启动时间（游戏时间），如果未激活返回 -1
     */
    UFUNCTION(BlueprintCallable, Category = "Lyra|GamePhase")
    float GetPhaseStartTime(const FGameplayTag& PhaseTag) const;
    
    /**
     * 获取指定阶段已运行的时间
     * @param PhaseTag 阶段标签
     * @return 运行时长（秒），如果未激活返回 0
     */
    UFUNCTION(BlueprintCallable, Category = "Lyra|GamePhase")
    float GetPhaseElapsedTime(const FGameplayTag& PhaseTag) const;
    
    // ============================================================
    // 观察者接口 (C++ 版本)
    // ============================================================
    
    /**
     * 等待指定阶段开始
     * @param PhaseTag 阶段标签
     * @param Delegate 回调委托
     * @return 观察者句柄（用于取消观察）
     */
    FLyraGamePhaseObserverHandle WaitForPhaseStart(
        const FGameplayTag& PhaseTag,
        FLyraGamePhaseTagDelegate::FDelegate Delegate
    );
    
    /**
     * 等待指定阶段结束
     */
    FLyraGamePhaseObserverHandle WaitForPhaseEnd(
        const FGameplayTag& PhaseTag,
        FLyraGamePhaseTagDelegate::FDelegate Delegate
    );
    
    /**
     * 取消观察
     * @param Handle 观察者句柄
     */
    void CancelObserver(FLyraGamePhaseObserverHandle& Handle);
    
    // ============================================================
    // 观察者接口 (Blueprint 版本)
    // ============================================================
    
    /**
     * 广播：当任意阶段开始时
     */
    UPROPERTY(BlueprintAssignable, Category = "Lyra|GamePhase")
    FLyraGamePhaseTagDelegate OnPhaseStarted;
    
    /**
     * 广播：当任意阶段结束时
     */
    UPROPERTY(BlueprintAssignable, Category = "Lyra|GamePhase")
    FLyraGamePhaseTagDelegate OnPhaseEnded;
    
protected:
    // ============================================================
    // 内部实现
    // ============================================================
    
    /**
     * 查找用于管理阶段的 ASC
     * 通常是 Game State 的 ASC
     */
    UAbilitySystemComponent* GetPhaseAbilitySystemComponent() const;
    
    /**
     * 当阶段 Ability 激活时的回调
     */
    void OnPhaseAbilityActivated(ULyraGamePhaseAbility* PhaseAbility);
    
    /**
     * 当阶段 Ability 结束时的回调
     */
    void OnPhaseAbilityEnded(ULyraGamePhaseAbility* PhaseAbility);
    
    /**
     * 广播阶段开始事件
     */
    void BroadcastPhaseStarted(const FGameplayTag& PhaseTag);
    
    /**
     * 广播阶段结束事件
     */
    void BroadcastPhaseEnded(const FGameplayTag& PhaseTag);
    
private:
    // 当前激活的阶段列表
    UPROPERTY()
    TArray<FLyraGamePhaseEntry> ActivePhases;
    
    // 阶段观察者
    // Key: PhaseTag, Value: 观察者列表
    struct FPhaseObserver
    {
        int32 ID;
        FLyraGamePhaseTagDelegate::FDelegate Callback;
        bool bWaitingForStart; // true: 等待开始, false: 等待结束
    };
    
    TMap<FGameplayTag, TArray<FPhaseObserver>> PhaseObservers;
    
    // 下一个观察者 ID
    int32 NextObserverID = 1;
    
    // Game State 的 ASC（缓存）
    UPROPERTY()
    TWeakObjectPtr<UAbilitySystemComponent> CachedPhaseASC;
};
```

#### 2.1.2 核心功能实现

**1. 子系统初始化**

```cpp
// LyraGamePhaseSubsystem.cpp

bool ULyraGamePhaseSubsystem::ShouldCreateSubsystem(UObject* Outer) const
{
    // 只在服务器和 Standalone 模式创建
    // 客户端通过网络同步接收阶段信息
    UWorld* World = Cast<UWorld>(Outer);
    return World && (World->GetNetMode() != NM_Client);
}

void ULyraGamePhaseSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    
    UE_LOG(LogLyra, Log, TEXT("[GamePhase] Subsystem initialized"));
    
    // 等待 Game State 创建后再初始化 ASC 引用
    // (Game State 通常在 Game Mode 的 InitGame 后创建)
}

void ULyraGamePhaseSubsystem::Deinitialize()
{
    // 清理所有激活的阶段
    EndAllPhases();
    
    // 清理观察者
    PhaseObservers.Empty();
    
    Super::Deinitialize();
}
```

**2. 启动阶段**

```cpp
bool ULyraGamePhaseSubsystem::StartPhase(
    TSubclassOf<ULyraGamePhaseAbility> PhaseAbility,
    AActor* PhaseInstigator)
{
    // 1. 参数验证
    if (!PhaseAbility)
    {
        UE_LOG(LogLyra, Error, TEXT("[GamePhase] Cannot start phase: Invalid PhaseAbility"));
        return false;
    }
    
    // 2. 权限检查
    UWorld* World = GetWorld();
    if (!World || World->GetNetMode() == NM_Client)
    {
        UE_LOG(LogLyra, Error, TEXT("[GamePhase] Cannot start phase: Not authority"));
        return false;
    }
    
    // 3. 获取 Phase ASC
    UAbilitySystemComponent* PhaseASC = GetPhaseAbilitySystemComponent();
    if (!PhaseASC)
    {
        UE_LOG(LogLyra, Error, TEXT("[GamePhase] Cannot start phase: No PhaseASC found"));
        return false;
    }
    
    // 4. 获取阶段默认对象的 PhaseTag
    ULyraGamePhaseAbility* PhaseCDO = PhaseAbility->GetDefaultObject<ULyraGamePhaseAbility>();
    const FGameplayTag PhaseTag = PhaseCDO->GetPhaseTag();
    
    if (!PhaseTag.IsValid())
    {
        UE_LOG(LogLyra, Error, TEXT("[GamePhase] Cannot start phase: Invalid PhaseTag"));
        return false;
    }
    
    // 5. 检查是否已经激活
    if (IsPhaseActive(PhaseTag))
    {
        UE_LOG(LogLyra, Warning, TEXT("[GamePhase] Phase '%s' is already active"),
            *PhaseTag.ToString());
        return false;
    }
    
    // 6. 授予并激活 Phase Ability
    FGameplayAbilitySpec AbilitySpec(PhaseAbility, 1, INDEX_NONE, PhaseInstigator);
    FGameplayAbilitySpecHandle SpecHandle = PhaseASC->GiveAbility(AbilitySpec);
    
    if (!SpecHandle.IsValid())
    {
        UE_LOG(LogLyra, Error, TEXT("[GamePhase] Failed to give phase ability"));
        return false;
    }
    
    // 激活 Ability
    bool bSuccess = PhaseASC->TryActivateAbility(SpecHandle);
    
    if (!bSuccess)
    {
        UE_LOG(LogLyra, Error, TEXT("[GamePhase] Failed to activate phase ability"));
        PhaseASC->ClearAbility(SpecHandle);
        return false;
    }
    
    // 7. 记录激活的阶段
    FLyraGamePhaseEntry& NewEntry = ActivePhases.AddDefaulted_GetRef();
    NewEntry.PhaseTag = PhaseTag;
    NewEntry.PhaseAbility = PhaseASC->FindAbilitySpecFromHandle(SpecHandle)->GetPrimaryInstance();
    NewEntry.StartTime = World->GetTimeSeconds();
    NewEntry.Instigator = PhaseInstigator;
    
    UE_LOG(LogLyra, Log, TEXT("[GamePhase] Started phase: %s"), *PhaseTag.ToString());
    
    // 8. 广播事件
    BroadcastPhaseStarted(PhaseTag);
    
    return true;
}
```

**3. 结束阶段**

```cpp
void ULyraGamePhaseSubsystem::EndPhase(const FGameplayTag& PhaseTag)
{
    // 1. 查找激活的阶段
    int32 FoundIndex = INDEX_NONE;
    for (int32 i = 0; i < ActivePhases.Num(); ++i)
    {
        if (ActivePhases[i].PhaseTag.MatchesTagExact(PhaseTag))
        {
            FoundIndex = i;
            break;
        }
    }
    
    if (FoundIndex == INDEX_NONE)
    {
        UE_LOG(LogLyra, Warning, TEXT("[GamePhase] Cannot end phase '%s': Not active"),
            *PhaseTag.ToString());
        return;
    }
    
    // 2. 获取阶段数据
    FLyraGamePhaseEntry PhaseEntry = ActivePhases[FoundIndex];
    ActivePhases.RemoveAt(FoundIndex);
    
    // 3. 结束 Ability
    if (PhaseEntry.PhaseAbility)
    {
        PhaseEntry.PhaseAbility->EndAbility(
            PhaseEntry.PhaseAbility->GetCurrentAbilitySpecHandle(),
            PhaseEntry.PhaseAbility->GetCurrentActorInfo(),
            PhaseEntry.PhaseAbility->GetCurrentActivationInfo(),
            true, // bReplicateEndAbility
            false // bWasCancelled
        );
    }
    
    UE_LOG(LogLyra, Log, TEXT("[GamePhase] Ended phase: %s (Duration: %.2fs)"),
        *PhaseTag.ToString(),
        GetWorld()->GetTimeSeconds() - PhaseEntry.StartTime);
    
    // 4. 广播事件
    BroadcastPhaseEnded(PhaseTag);
}

void ULyraGamePhaseSubsystem::EndAllPhases()
{
    // 逆序结束所有阶段（后启动的先结束）
    while (ActivePhases.Num() > 0)
    {
        EndPhase(ActivePhases.Last().PhaseTag);
    }
}
```

**4. 阶段查询**

```cpp
bool ULyraGamePhaseSubsystem::IsPhaseActive(const FGameplayTag& PhaseTag) const
{
    for (const FLyraGamePhaseEntry& Entry : ActivePhases)
    {
        // 支持父标签匹配
        // 例如: GamePhase.InProgress 可以匹配 GamePhase.InProgress.Warmup
        if (Entry.PhaseTag.MatchesTag(PhaseTag))
        {
            return true;
        }
    }
    return false;
}

void ULyraGamePhaseSubsystem::GetActivePhases(TArray<FGameplayTag>& OutPhases) const
{
    OutPhases.Reset(ActivePhases.Num());
    for (const FLyraGamePhaseEntry& Entry : ActivePhases)
    {
        OutPhases.Add(Entry.PhaseTag);
    }
}

float ULyraGamePhaseSubsystem::GetPhaseStartTime(const FGameplayTag& PhaseTag) const
{
    for (const FLyraGamePhaseEntry& Entry : ActivePhases)
    {
        if (Entry.PhaseTag.MatchesTagExact(PhaseTag))
        {
            return Entry.StartTime;
        }
    }
    return -1.0f;
}

float ULyraGamePhaseSubsystem::GetPhaseElapsedTime(const FGameplayTag& PhaseTag) const
{
    float StartTime = GetPhaseStartTime(PhaseTag);
    if (StartTime < 0.0f)
    {
        return 0.0f;
    }
    
    return GetWorld()->GetTimeSeconds() - StartTime;
}
```

**5. 观察者管理**

```cpp
FLyraGamePhaseObserverHandle ULyraGamePhaseSubsystem::WaitForPhaseStart(
    const FGameplayTag& PhaseTag,
    FLyraGamePhaseTagDelegate::FDelegate Delegate)
{
    FLyraGamePhaseObserverHandle Handle;
    Handle.ID = NextObserverID++;
    
    // 如果阶段已经激活，立即调用回调
    if (IsPhaseActive(PhaseTag))
    {
        Delegate.ExecuteIfBound(PhaseTag);
        return Handle; // 返回无效句柄（已执行）
    }
    
    // 添加到观察者列表
    FPhaseObserver Observer;
    Observer.ID = Handle.ID;
    Observer.Callback = Delegate;
    Observer.bWaitingForStart = true;
    
    TArray<FPhaseObserver>& Observers = PhaseObservers.FindOrAdd(PhaseTag);
    Observers.Add(Observer);
    
    return Handle;
}

FLyraGamePhaseObserverHandle ULyraGamePhaseSubsystem::WaitForPhaseEnd(
    const FGameplayTag& PhaseTag,
    FLyraGamePhaseTagDelegate::FDelegate Delegate)
{
    FLyraGamePhaseObserverHandle Handle;
    Handle.ID = NextObserverID++;
    
    // 如果阶段未激活，立即调用回调
    if (!IsPhaseActive(PhaseTag))
    {
        Delegate.ExecuteIfBound(PhaseTag);
        return Handle; // 返回无效句柄（已执行）
    }
    
    // 添加到观察者列表
    FPhaseObserver Observer;
    Observer.ID = Handle.ID;
    Observer.Callback = Delegate;
    Observer.bWaitingForStart = false;
    
    TArray<FPhaseObserver>& Observers = PhaseObservers.FindOrAdd(PhaseTag);
    Observers.Add(Observer);
    
    return Handle;
}

void ULyraGamePhaseSubsystem::CancelObserver(FLyraGamePhaseObserverHandle& Handle)
{
    if (!Handle.IsValid())
    {
        return;
    }
    
    // 遍历所有观察者列表，移除指定 ID 的观察者
    for (auto& Pair : PhaseObservers)
    {
        TArray<FPhaseObserver>& Observers = Pair.Value;
        for (int32 i = Observers.Num() - 1; i >= 0; --i)
        {
            if (Observers[i].ID == Handle.ID)
            {
                Observers.RemoveAt(i);
                Handle.Invalidate();
                return;
            }
        }
    }
}
```

**6. 事件广播**

```cpp
void ULyraGamePhaseSubsystem::BroadcastPhaseStarted(const FGameplayTag& PhaseTag)
{
    // 1. 广播全局事件（Blueprint 使用）
    OnPhaseStarted.Broadcast(PhaseTag);
    
    // 2. 触发特定阶段的观察者
    if (TArray<FPhaseObserver>* Observers = PhaseObservers.Find(PhaseTag))
    {
        // 复制列表（避免回调中修改列表导致迭代器失效）
        TArray<FPhaseObserver> ObserversCopy = *Observers;
        
        for (int32 i = ObserversCopy.Num() - 1; i >= 0; --i)
        {
            FPhaseObserver& Observer = ObserversCopy[i];
            
            // 只触发等待开始的观察者
            if (Observer.bWaitingForStart)
            {
                Observer.Callback.ExecuteIfBound(PhaseTag);
                
                // 触发后移除（一次性观察者）
                Observers->RemoveAll([ID = Observer.ID](const FPhaseObserver& O)
                {
                    return O.ID == ID;
                });
            }
        }
    }
    
    // 3. 触发匹配父标签的观察者
    // 例如: PhaseTag = "GamePhase.InProgress.Round1"
    //       也应该触发 "GamePhase.InProgress" 的观察者
    TArray<FGameplayTag> ParentTags;
    PhaseTag.GetGameplayTagParents(ParentTags);
    
    for (const FGameplayTag& ParentTag : ParentTags)
    {
        if (TArray<FPhaseObserver>* ParentObservers = PhaseObservers.Find(ParentTag))
        {
            TArray<FPhaseObserver> ParentObserversCopy = *ParentObservers;
            
            for (int32 i = ParentObserversCopy.Num() - 1; i >= 0; --i)
            {
                FPhaseObserver& Observer = ParentObserversCopy[i];
                
                if (Observer.bWaitingForStart)
                {
                    Observer.Callback.ExecuteIfBound(PhaseTag); // 传递完整的子标签
                    
                    ParentObservers->RemoveAll([ID = Observer.ID](const FPhaseObserver& O)
                    {
                        return O.ID == ID;
                    });
                }
            }
        }
    }
}

void ULyraGamePhaseSubsystem::BroadcastPhaseEnded(const FGameplayTag& PhaseTag)
{
    // 与 BroadcastPhaseStarted 类似，但触发等待结束的观察者
    OnPhaseEnded.Broadcast(PhaseTag);
    
    if (TArray<FPhaseObserver>* Observers = PhaseObservers.Find(PhaseTag))
    {
        TArray<FPhaseObserver> ObserversCopy = *Observers;
        
        for (int32 i = ObserversCopy.Num() - 1; i >= 0; --i)
        {
            FPhaseObserver& Observer = ObserversCopy[i];
            
            // 只触发等待结束的观察者
            if (!Observer.bWaitingForStart)
            {
                Observer.Callback.ExecuteIfBound(PhaseTag);
                
                Observers->RemoveAll([ID = Observer.ID](const FPhaseObserver& O)
                {
                    return O.ID == ID;
                });
            }
        }
    }
    
    // 同样处理父标签
    TArray<FGameplayTag> ParentTags;
    PhaseTag.GetGameplayTagParents(ParentTags);
    
    for (const FGameplayTag& ParentTag : ParentTags)
    {
        if (TArray<FPhaseObserver>* ParentObservers = PhaseObservers.Find(ParentTag))
        {
            TArray<FPhaseObserver> ParentObserversCopy = *ParentObservers;
            
            for (int32 i = ParentObserversCopy.Num() - 1; i >= 0; --i)
            {
                FPhaseObserver& Observer = ParentObserversCopy[i];
                
                if (!Observer.bWaitingForStart)
                {
                    Observer.Callback.ExecuteIfBound(PhaseTag);
                    
                    ParentObservers->RemoveAll([ID = Observer.ID](const FPhaseObserver& O)
                    {
                        return O.ID == ID;
                    });
                }
            }
        }
    }
}
```

**7. 获取 Phase ASC**

```cpp
UAbilitySystemComponent* ULyraGamePhaseSubsystem::GetPhaseAbilitySystemComponent() const
{
    // 1. 检查缓存
    if (CachedPhaseASC.IsValid())
    {
        return CachedPhaseASC.Get();
    }
    
    // 2. 从 Game State 获取 ASC
    UWorld* World = GetWorld();
    if (!World)
    {
        return nullptr;
    }
    
    AGameStateBase* GameState = World->GetGameState();
    if (!GameState)
    {
        return nullptr;
    }
    
    // Lyra 的 Game State 实现了 IAbilitySystemInterface
    IAbilitySystemInterface* ASI = Cast<IAbilitySystemInterface>(GameState);
    if (ASI)
    {
        UAbilitySystemComponent* ASC = ASI->GetAbilitySystemComponent();
        CachedPhaseASC = ASC;
        return ASC;
    }
    
    return nullptr;
}
```

### 2.2 Phase 的定义与数据结构

#### 2.2.1 Phase 配置数据资产

在 Lyra 中，游戏阶段通常通过 **Data Asset** 进行配置，这样策划可以在编辑器中可视化地设置阶段流程。

```cpp
// LyraGamePhaseConfiguration.h
#pragma once

#include "Engine/DataAsset.h"
#include "GameplayTagContainer.h"
#include "LyraGamePhaseConfiguration.generated.h"

class ULyraGamePhaseAbility;

/**
 * 单个阶段配置
 */
USTRUCT(BlueprintType)
struct FLyraGamePhaseConfig
{
    GENERATED_BODY()
    
    // 阶段 Ability 类
    UPROPERTY(EditDefaultsOnly, Category = "Phase")
    TSubclassOf<ULyraGamePhaseAbility> PhaseAbility;
    
    // 阶段标签（用于查询和事件）
    UPROPERTY(EditDefaultsOnly, Category = "Phase", meta = (Categories = "GamePhase"))
    FGameplayTag PhaseTag;
    
    // 阶段显示名称
    UPROPERTY(EditDefaultsOnly, Category = "Phase")
    FText PhaseName;
    
    // 阶段描述
    UPROPERTY(EditDefaultsOnly, Category = "Phase", meta = (MultiLine = true))
    FText PhaseDescription;
    
    // 最小持续时间（秒，0 表示无限制）
    UPROPERTY(EditDefaultsOnly, Category = "Duration")
    float MinDuration = 0.0f;
    
    // 最大持续时间（秒，0 表示无限制）
    UPROPERTY(EditDefaultsOnly, Category = "Duration")
    float MaxDuration = 0.0f;
    
    // 自动转换到下一个阶段
    UPROPERTY(EditDefaultsOnly, Category = "Transition")
    bool bAutoTransitionToNext = true;
    
    // 转换条件
    UPROPERTY(EditDefaultsOnly, Category = "Transition")
    FPhaseTransitionCondition TransitionCondition;
    
    // 是否允许跳过此阶段
    UPROPERTY(EditDefaultsOnly, Category = "Flow Control")
    bool bCanSkip = false;
    
    // 是否允许暂停
    UPROPERTY(EditDefaultsOnly, Category = "Flow Control")
    bool bCanPause = false;
};

/**
 * 阶段转换条件
 */
USTRUCT(BlueprintType)
struct FPhaseTransitionCondition
{
    GENERATED_BODY()
    
    // ============================================================
    // 基于玩家数量的条件
    // ============================================================
    
    // 最少玩家数量
    UPROPERTY(EditAnywhere, Category = "Player Count")
    int32 MinPlayerCount = 0;
    
    // 最多玩家数量（0 表示无限制）
    UPROPERTY(EditAnywhere, Category = "Player Count")
    int32 MaxPlayerCount = 0;
    
    // 所有玩家必须准备好
    UPROPERTY(EditAnywhere, Category = "Player Count")
    bool bAllPlayersReady = false;
    
    // ============================================================
    // 基于时间的条件
    // ============================================================
    
    // 使用定时器
    UPROPERTY(EditAnywhere, Category = "Timer")
    bool bUseTimer = false;
    
    // 定时器时长（秒）
    UPROPERTY(EditAnywhere, Category = "Timer", meta = (EditCondition = "bUseTimer"))
    float TimerDuration = 10.0f;
    
    // 定时器到期后是否自动转换
    UPROPERTY(EditAnywhere, Category = "Timer", meta = (EditCondition = "bUseTimer"))
    bool bAutoTransitionOnTimerExpire = true;
    
    // ============================================================
    // 基于 Gameplay Tag 的条件
    // ============================================================
    
    // 必须存在的 Tags
    UPROPERTY(EditAnywhere, Category = "Gameplay Tags")
    FGameplayTagContainer RequiredTags;
    
    // 禁止存在的 Tags
    UPROPERTY(EditAnywhere, Category = "Gameplay Tags")
    FGameplayTagContainer BlockedTags;
    
    // ============================================================
    // 基于游戏事件的条件
    // ============================================================
    
    // 等待特定的 Gameplay Event
    UPROPERTY(EditAnywhere, Category = "Events")
    FGameplayTag WaitForEvent;
    
    // 事件需要的参数（可选）
    UPROPERTY(EditAnywhere, Category = "Events")
    FGameplayTagContainer EventRequiredTags;
    
    // ============================================================
    // 自定义条件
    // ============================================================
    
    // 自定义条件检查类
    UPROPERTY(EditAnywhere, Category = "Custom")
    TSubclassOf<UPhaseTransitionRule> CustomRule;
    
    /**
     * 检查条件是否满足
     */
    bool IsMet(UWorld* World) const;
};

/**
 * 自定义阶段转换规则基类
 */
UCLASS(Abstract, Blueprintable)
class UPhaseTransitionRule : public UObject
{
    GENERATED_BODY()
    
public:
    /**
     * 检查转换条件是否满足
     * @param World 游戏世界
     * @param PhaseSubsystem 阶段子系统
     * @return 是否满足条件
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase Transition")
    bool CheckCondition(UWorld* World, ULyraGamePhaseSubsystem* PhaseSubsystem) const;
    
    virtual bool CheckCondition_Implementation(UWorld* World, ULyraGamePhaseSubsystem* PhaseSubsystem) const
    {
        return true;
    }
};

/**
 * 游戏阶段配置数据资产
 * 定义一个游戏模式的完整阶段流程
 */
UCLASS(BlueprintType)
class LYRAGAME_API ULyraGamePhaseConfiguration : public UPrimaryDataAsset
{
    GENERATED_BODY()
    
public:
    // 阶段序列
    UPROPERTY(EditDefaultsOnly, Category = "Phases")
    TArray<FLyraGamePhaseConfig> PhaseSequence;
    
    // 初始阶段（如果为空，使用 PhaseSequence[0]）
    UPROPERTY(EditDefaultsOnly, Category = "Phases")
    TSubclassOf<ULyraGamePhaseAbility> InitialPhase;
    
    // 是否循环阶段（例如：多回合游戏）
    UPROPERTY(EditDefaultsOnly, Category = "Flow Control")
    bool bLoopPhases = false;
    
    // 循环次数（0 表示无限循环）
    UPROPERTY(EditDefaultsOnly, Category = "Flow Control", meta = (EditCondition = "bLoopPhases"))
    int32 MaxLoops = 1;
    
    // 阶段之间的过渡时间（秒）
    UPROPERTY(EditDefaultsOnly, Category = "Flow Control")
    float TransitionDelay = 0.5f;
    
    /**
     * 获取指定标签的阶段配置
     */
    UFUNCTION(BlueprintCallable, Category = "Phases")
    const FLyraGamePhaseConfig* FindPhaseConfig(const FGameplayTag& PhaseTag) const;
    
    /**
     * 获取下一个阶段配置
     */
    UFUNCTION(BlueprintCallable, Category = "Phases")
    const FLyraGamePhaseConfig* GetNextPhaseConfig(const FGameplayTag& CurrentPhaseTag) const;
};
```

**转换条件实现**：

```cpp
// LyraGamePhaseConfiguration.cpp

bool FPhaseTransitionCondition::IsMet(UWorld* World) const
{
    if (!World)
    {
        return false;
    }
    
    // 1. 检查玩家数量条件
    if (MinPlayerCount > 0 || MaxPlayerCount > 0 || bAllPlayersReady)
    {
        AGameStateBase* GameState = World->GetGameState();
        if (!GameState)
        {
            return false;
        }
        
        int32 PlayerCount = GameState->PlayerArray.Num();
        
        // 最少玩家
        if (MinPlayerCount > 0 && PlayerCount < MinPlayerCount)
        {
            return false;
        }
        
        // 最多玩家
        if (MaxPlayerCount > 0 && PlayerCount > MaxPlayerCount)
        {
            return false;
        }
        
        // 所有玩家准备好
        if (bAllPlayersReady)
        {
            for (APlayerState* PS : GameState->PlayerArray)
            {
                // 假设 PlayerState 有 bIsReady 标志
                if (ALyraPlayerState* LyraPS = Cast<ALyraPlayerState>(PS))
                {
                    if (!LyraPS->IsReady())
                    {
                        return false;
                    }
                }
            }
        }
    }
    
    // 2. 检查 Gameplay Tag 条件
    ULyraGamePhaseSubsystem* PhaseSubsystem = 
        World->GetSubsystem<ULyraGamePhaseSubsystem>();
    
    if (PhaseSubsystem)
    {
        // 获取 Game State ASC
        AGameStateBase* GameState = World->GetGameState();
        if (IAbilitySystemInterface* ASI = Cast<IAbilitySystemInterface>(GameState))
        {
            UAbilitySystemComponent* ASC = ASI->GetAbilitySystemComponent();
            if (ASC)
            {
                // 检查必需的 Tags
                if (!RequiredTags.IsEmpty())
                {
                    if (!ASC->HasAllMatchingGameplayTags(RequiredTags))
                    {
                        return false;
                    }
                }
                
                // 检查禁止的 Tags
                if (!BlockedTags.IsEmpty())
                {
                    if (ASC->HasAnyMatchingGameplayTags(BlockedTags))
                    {
                        return false;
                    }
                }
            }
        }
    }
    
    // 3. 定时器条件由 Phase Ability 内部处理
    //    这里只返回基本条件的结果
    
    // 4. 自定义条件
    if (CustomRule)
    {
        UPhaseTransitionRule* Rule = CustomRule->GetDefaultObject<UPhaseTransitionRule>();
        if (Rule && PhaseSubsystem)
        {
            return Rule->CheckCondition(World, PhaseSubsystem);
        }
    }
    
    return true;
}
```

#### 2.2.2 在编辑器中配置阶段

**创建 Phase Configuration 数据资产**：

1. 在内容浏览器右键 → **Miscellaneous** → **Data Asset**
2. 选择 `ULyraGamePhaseConfiguration`
3. 命名为 `DA_GamePhases_TDM`（团队死斗模式）

**配置示例 - TDM 阶段流程**：

```
Phase Sequence:
├─ [0] 等待玩家
│   ├─ Phase Ability: GA_GamePhase_WaitingForPlayers
│   ├─ Phase Tag: GamePhase.WaitingForPlayers
│   ├─ Phase Name: "等待玩家加入"
│   ├─ Min Duration: 5.0
│   ├─ Max Duration: 300.0
│   └─ Transition Condition:
│       ├─ Min Player Count: 2
│       └─ Timer Duration: 60.0 (超时自动开始)
│
├─ [1] 倒计时准备
│   ├─ Phase Ability: GA_GamePhase_Countdown
│   ├─ Phase Tag: GamePhase.Countdown
│   ├─ Phase Name: "倒计时"
│   ├─ Min Duration: 5.0
│   ├─ Max Duration: 10.0
│   └─ Transition Condition:
│       ├─ Use Timer: true
│       └─ Timer Duration: 5.0
│
├─ [2] 游戏进行
│   ├─ Phase Ability: GA_GamePhase_InProgress_TDM
│   ├─ Phase Tag: GamePhase.InProgress
│   ├─ Phase Name: "游戏中"
│   ├─ Max Duration: 600.0 (10 分钟限时)
│   └─ Transition Condition:
│       └─ Custom Rule: Rule_TDM_WinCondition
│           (检查是否有队伍达到目标分数)
│
└─ [3] 结算展示
    ├─ Phase Ability: GA_GamePhase_PostMatch
    ├─ Phase Tag: GamePhase.PostMatch
    ├─ Phase Name: "结算"
    ├─ Min Duration: 10.0
    ├─ Max Duration: 30.0
    └─ Transition Condition:
        ├─ Timer Duration: 15.0
        └─ Wait For Event: Event.UI.ScoreboardDismissed
            (玩家关闭结算界面后结束)
```

#### 2.2.3 自定义转换规则示例

```cpp
// Rule_TDM_WinCondition.h
#pragma once

#include "LyraGamePhaseConfiguration.h"
#include "Rule_TDM_WinCondition.generated.h"

/**
 * TDM 模式胜利条件检查
 */
UCLASS()
class URule_TDM_WinCondition : public UPhaseTransitionRule
{
    GENERATED_BODY()
    
public:
    // 目标击杀数
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "TDM")
    int32 TargetScore = 50;
    
    virtual bool CheckCondition_Implementation(
        UWorld* World, 
        ULyraGamePhaseSubsystem* PhaseSubsystem) const override
    {
        ALyraGameState* GameState = World->GetGameState<ALyraGameState>();
        if (!GameState)
        {
            return false;
        }
        
        // 检查是否有队伍达到目标分数
        for (const ALyraTeamInfo* Team : GameState->GetTeams())
        {
            if (Team->GetScore() >= TargetScore)
            {
                return true;
            }
        }
        
        return false;
    }
};
```

###2.3 Phase 状态机制

#### 2.3.1 阶段生命周期

每个游戏阶段都遵循以下生命周期：

```
┌──────────────────────────────────────────────────────┐
│              Phase Lifecycle (阶段生命周期)            │
└──────────────────────────────────────────────────────┘

1. [创建]
   ├─ Game Mode 或 Phase Subsystem 调用 StartPhase()
   ├─ 创建 Phase Ability Spec
   └─ 添加到 Game State ASC

2. [激活]
   ├─ ASC 激活 Ability
   ├─ ActivateAbility() 被调用
   ├─ OnPhaseBegin() 被调用
   └─ 广播 OnPhaseStarted 事件

3. [运行中]
   ├─ Phase Ability Tick (如果需要)
   ├─ 监听游戏事件
   ├─ 更新 UI 状态
   └─ 检查转换条件

4. [转换检测]
   ├─ CanTransitionToNextPhase() 返回 true
   ├─ 满足转换条件
   └─ 触发转换逻辑

5. [结束]
   ├─ EndAbility() 被调用
   ├─ OnPhaseEnd() 被调用
   ├─ 广播 OnPhaseEnded 事件
   ├─ 清理资源
   └─ 从 ActivePhases 列表移除

6. [下一阶段]
   ├─ Phase Subsystem 启动下一个阶段
   └─ 回到步骤 1
```

#### 2.3.2 阶段状态数据

```cpp
// 阶段运行时状态
USTRUCT()
struct FLyraGamePhaseRuntimeData
{
    GENERATED_BODY()
    
    // 阶段标签
    UPROPERTY()
    FGameplayTag PhaseTag;
    
    // 阶段开始时间（游戏时间）
    UPROPERTY()
    float StartTime = 0.0f;
    
    // 阶段预计结束时间（如果有定时器）
    UPROPERTY()
    float ScheduledEndTime = 0.0f;
    
    // 阶段触发者
    UPROPERTY()
    TWeakObjectPtr<AActor> Instigator;
    
    // 阶段状态标志
    UPROPERTY()
    uint8 bPaused : 1;
    
    UPROPERTY()
    uint8 bCanSkip : 1;
    
    UPROPERTY()
    uint8 bAutoTransition : 1;
    
    // 阶段自定义数据（键值对）
    UPROPERTY()
    TMap<FName, FString> CustomData;
    
    /**
     * 获取阶段已运行时间
     */
    float GetElapsedTime(UWorld* World) const
    {
        if (!World)
        {
            return 0.0f;
        }
        return World->GetTimeSeconds() - StartTime;
    }
    
    /**
     * 获取阶段剩余时间
     */
    float GetRemainingTime(UWorld* World) const
    {
        if (!World || ScheduledEndTime <= 0.0f)
        {
            return 0.0f;
        }
        return FMath::Max(0.0f, ScheduledEndTime - World->GetTimeSeconds());
    }
};
```

#### 2.3.3 阶段状态同步

**Server → Client 同步**：

```cpp
// LyraGameState.h
UCLASS()
class ALyraGameState : public AModularGameStateBase, public IAbilitySystemInterface
{
    GENERATED_BODY()
    
public:
    // 当前激活的阶段标签（复制给所有客户端）
    UPROPERTY(ReplicatedUsing = OnRep_CurrentPhaseTag, BlueprintReadOnly, Category = "Phase")
    FGameplayTag CurrentPhaseTag;
    
    // 阶段开始时间（服务器时间）
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Phase")
    float PhaseStartServerTime = 0.0f;
    
    // 阶段剩余时间（秒）
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Phase")
    float PhaseRemainingTime = 0.0f;
    
protected:
    UFUNCTION()
    void OnRep_CurrentPhaseTag(FGameplayTag OldPhaseTag)
    {
        // 客户端收到阶段变化通知
        OnPhaseChanged(OldPhaseTag, CurrentPhaseTag);
    }
    
    void OnPhaseChanged(const FGameplayTag& OldPhase, const FGameplayTag& NewPhase)
    {
        UE_LOG(LogLyra, Log, TEXT("[Client] Phase changed: %s → %s"),
            *OldPhase.ToString(), *NewPhase.ToString());
        
        // 触发客户端本地事件
        OnClientPhaseChanged.Broadcast(NewPhase);
    }
    
public:
    // 客户端阶段变化事件
    UPROPERTY(BlueprintAssignable, Category = "Phase")
    FLyraGamePhaseTagDelegate OnClientPhaseChanged;
    
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        
        DOREPLIFETIME(ALyraGameState, CurrentPhaseTag);
        DOREPLIFETIME(ALyraGameState, PhaseStartServerTime);
        DOREPLIFETIME(ALyraGameState, PhaseRemainingTime);
    }
};
```

**Phase Ability 更新 Game State**：

```cpp
void ULyraGamePhaseAbility::OnPhaseBegin()
{
    // 更新 Game State 的阶段信息
    ALyraGameState* GameState = GetWorld()->GetGameState<ALyraGameState>();
    if (GameState && GetActorInfo().IsNetAuthority())
    {
        GameState->CurrentPhaseTag = PhaseTag;
        GameState->PhaseStartServerTime = GetWorld()->GetTimeSeconds();
        
        // 如果有定时器，设置剩余时间
        if (PhaseDuration > 0.0f)
        {
            GameState->PhaseRemainingTime = PhaseDuration;
        }
    }
}
```

#### 2.3.4 阶段暂停与恢复

```cpp
// ULyraGamePhaseSubsystem 添加暂停功能
UCLASS()
class ULyraGamePhaseSubsystem : public UWorldSubsystem
{
    // ...
    
public:
    /**
     * 暂停指定阶段
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Lyra|GamePhase")
    bool PausePhase(const FGameplayTag& PhaseTag);
    
    /**
     * 恢复指定阶段
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Lyra|GamePhase")
    bool ResumePhase(const FGameplayTag& PhaseTag);
    
    /**
     * 检查阶段是否暂停
     */
    UFUNCTION(BlueprintCallable, Category = "Lyra|GamePhase")
    bool IsPhasePaused(const FGameplayTag& PhaseTag) const;
    
private:
    // 暂停的阶段集合
    UPROPERTY()
    TSet<FGameplayTag> PausedPhases;
};

// 实现
bool ULyraGamePhaseSubsystem::PausePhase(const FGameplayTag& PhaseTag)
{
    // 1. 检查阶段是否激活
    FLyraGamePhaseEntry* Entry = ActivePhases.FindByPredicate(
        [&PhaseTag](const FLyraGamePhaseEntry& E)
        {
            return E.PhaseTag.MatchesTagExact(PhaseTag);
        }
    );
    
    if (!Entry)
    {
        UE_LOG(LogLyra, Warning, TEXT("[GamePhase] Cannot pause: Phase not active"));
        return false;
    }
    
    // 2. 检查是否已暂停
    if (PausedPhases.Contains(PhaseTag))
    {
        return true; // 已经暂停
    }
    
    // 3. 调用 Phase Ability 的暂停逻辑
    if (Entry->PhaseAbility)
    {
        Entry->PhaseAbility->OnPhasePaused();
    }
    
    // 4. 标记为暂停
    PausedPhases.Add(PhaseTag);
    
    UE_LOG(LogLyra, Log, TEXT("[GamePhase] Paused phase: %s"), *PhaseTag.ToString());
    
    // 5. 广播事件
    OnPhasePaused.Broadcast(PhaseTag);
    
    return true;
}

bool ULyraGamePhaseSubsystem::ResumePhase(const FGameplayTag& PhaseTag)
{
    // 1. 检查是否暂停
    if (!PausedPhases.Contains(PhaseTag))
    {
        return false; // 未暂停
    }
    
    // 2. 查找阶段
    FLyraGamePhaseEntry* Entry = ActivePhases.FindByPredicate(
        [&PhaseTag](const FLyraGamePhaseEntry& E)
        {
            return E.PhaseTag.MatchesTagExact(PhaseTag);
        }
    );
    
    if (!Entry)
    {
        // 阶段已结束，移除暂停标记
        PausedPhases.Remove(PhaseTag);
        return false;
    }
    
    // 3. 调用 Phase Ability 的恢复逻辑
    if (Entry->PhaseAbility)
    {
        Entry->PhaseAbility->OnPhaseResumed();
    }
    
    // 4. 移除暂停标记
    PausedPhases.Remove(PhaseTag);
    
    UE_LOG(LogLyra, Log, TEXT("[GamePhase] Resumed phase: %s"), *PhaseTag.ToString());
    
    // 5. 广播事件
    OnPhaseResumed.Broadcast(PhaseTag);
    
    return true;
}
```

**Phase Ability 中处理暂停**：

```cpp
// ULyraGamePhaseAbility 添加暂停支持
UCLASS()
class ULyraGamePhaseAbility : public ULyraGameplayAbility
{
    // ...
    
protected:
    /**
     * 阶段被暂停时调用
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase")
    void OnPhasePaused();
    
    virtual void OnPhasePaused_Implementation()
    {
        // 默认行为：暂停所有 Ability Tasks
        for (UGameplayTask* Task : ActiveTasks)
        {
            if (Task)
            {
                Task->Pause();
            }
        }
        
        // 广播 Gameplay Event
        FGameplayEventData EventData;
        EventData.EventTag = TAG_Event_Phase_Paused;
        EventData.Instigator = GetAvatarActorFromActorInfo();
        
        UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(
            GetAvatarActorFromActorInfo(),
            TAG_Event_Phase_Paused,
            EventData
        );
    }
    
    /**
     * 阶段恢复时调用
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase")
    void OnPhaseResumed();
    
    virtual void OnPhaseResumed_Implementation()
    {
        // 默认行为：恢复所有 Ability Tasks
        for (UGameplayTask* Task : ActiveTasks)
        {
            if (Task)
            {
                Task->Resume();
            }
        }
        
        // 广播 Gameplay Event
        FGameplayEventData EventData;
        EventData.EventTag = TAG_Event_Phase_Resumed;
        EventData.Instigator = GetAvatarActorFromActorInfo();
        
        UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(
            GetAvatarActorFromActorInfo(),
            TAG_Event_Phase_Resumed,
            EventData
        );
    }
};
```

### 2.4 Phase 观察者模式

#### 2.4.1 观察者模式的优势

Phase 观察者模式允许系统的不同部分在不直接耦合的情况下响应阶段变化：

```
┌─────────────────────────────────────────────────────┐
│          Phase Subsystem (发布者)                    │
│  • 管理阶段生命周期                                   │
│  • 广播阶段事件                                      │
└─────────────────────────────────────────────────────┘
                      │
                      │ 事件广播
                      ↓
┌─────────────────────────────────────────────────────┐
│             观察者 (Observers)                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────────┐  ┌──────────────────┐       │
│  │  UI System       │  │  Audio Manager   │       │
│  │  • 显示/隐藏界面  │  │  • 播放音乐/音效  │       │
│  └──────────────────┘  └──────────────────┘       │
│                                                     │
│  ┌──────────────────┐  ┌──────────────────┐       │
│  │  Game Mode       │  │  AI Controllers  │       │
│  │  • 生成对象       │  │  • 调整行为       │       │
│  └──────────────────┘  └──────────────────┘       │
│                                                     │
│  ┌──────────────────┐  ┌──────────────────┐       │
│  │  Analytics       │  │  Replay System   │       │
│  │  • 记录数据       │  │  • 标记时间点     │       │
│  └──────────────────┘  └──────────────────┘       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**优势**：
- ✅ **解耦**：观察者不需要知道阶段的实现细节
- ✅ **可扩展**：可以随时添加新的观察者
- ✅ **灵活**：观察者可以选择性监听特定阶段
- ✅ **可测试**：可以单独测试观察者逻辑

#### 2.4.2 C++ 观察者示例

**示例 1：Game Mode 监听阶段变化**

```cpp
// MyGameMode.h
UCLASS()
class AMyGameMode : public ALyraGameMode
{
    GENERATED_BODY()
    
protected:
    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override
    {
        Super::InitGame(MapName, Options, ErrorMessage);
        
        // 获取 Phase Subsystem
        ULyraGamePhaseSubsystem* PhaseSubsystem = 
            GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
        
        if (PhaseSubsystem)
        {
            // 监听 "游戏开始" 阶段
            PhaseSubsystem->WaitForPhaseStart(
                TAG_GamePhase_InProgress,
                FLyraGamePhaseTagDelegate::FDelegate::CreateUObject(
                    this, &AMyGameMode::OnGameStarted
                )
            );
            
            // 监听 "游戏结束" 阶段
            PhaseSubsystem->WaitForPhaseStart(
                TAG_GamePhase_PostMatch,
                FLyraGamePhaseTagDelegate::FDelegate::CreateUObject(
                    this, &AMyGameMode::OnGameEnded
                )
            );
        }
    }
    
    void OnGameStarted(const FGameplayTag& PhaseTag)
    {
        UE_LOG(LogTemp, Log, TEXT("Game has started!"));
        
        // 生成 Bot
        SpawnBots();
        
        // 启动游戏逻辑
        StartGameplayTimers();
    }
    
    void OnGameEnded(const FGameplayTag& PhaseTag)
    {
        UE_LOG(LogTemp, Log, TEXT("Game has ended!"));
        
        // 清理游戏对象
        CleanupGameplay();
        
        // 保存统计数据
        SaveMatchStatistics();
    }
    
    void SpawnBots();
    void StartGameplayTimers();
    void CleanupGameplay();
    void SaveMatchStatistics();
};
```

**示例 2：UI 组件监听阶段变化**

```cpp
// MatchStateWidget.h
UCLASS()
class UMatchStateWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()
    
protected:
    virtual void NativeConstruct() override
    {
        Super::NativeConstruct();
        
        // 订阅阶段事件
        ULyraGamePhaseSubsystem* PhaseSubsystem = 
            GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
        
        if (PhaseSubsystem)
        {
            // 使用 Dynamic Delegate（Blueprint 兼容）
            PhaseSubsystem->OnPhaseStarted.AddDynamic(
                this, &UMatchStateWidget::HandlePhaseStarted
            );
            
            PhaseSubsystem->OnPhaseEnded.AddDynamic(
                this, &UMatchStateWidget::HandlePhaseEnded
            );
        }
    }
    
    virtual void NativeDestruct() override
    {
        // 取消订阅
        if (ULyraGamePhaseSubsystem* PhaseSubsystem = 
            GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>())
        {
            PhaseSubsystem->OnPhaseStarted.RemoveDynamic(
                this, &UMatchStateWidget::HandlePhaseStarted
            );
            
            PhaseSubsystem->OnPhaseEnded.RemoveDynamic(
                this, &UMatchStateWidget::HandlePhaseEnded
            );
        }
        
        Super::NativeDestruct();
    }
    
    UFUNCTION()
    void HandlePhaseStarted(const FGameplayTag& PhaseTag)
    {
        if (PhaseTag.MatchesTag(TAG_GamePhase_WaitingForPlayers))
        {
            ShowLobbyScreen();
        }
        else if (PhaseTag.MatchesTag(TAG_GamePhase_Countdown))
        {
            ShowCountdownScreen();
        }
        else if (PhaseTag.MatchesTag(TAG_GamePhase_InProgress))
        {
            ShowGameplayHUD();
        }
        else if (PhaseTag.MatchesTag(TAG_GamePhase_PostMatch))
        {
            ShowScoreboard();
        }
    }
    
    UFUNCTION()
    void HandlePhaseEnded(const FGameplayTag& PhaseTag)
    {
        // 隐藏阶段相关的 UI
        if (PhaseTag.MatchesTag(TAG_GamePhase_Countdown))
        {
            HideCountdownScreen();
        }
    }
    
    UFUNCTION(BlueprintImplementableEvent, Category = "UI")
    void ShowLobbyScreen();
    
    UFUNCTION(BlueprintImplementableEvent, Category = "UI")
    void ShowCountdownScreen();
    
    UFUNCTION(BlueprintImplementableEvent, Category = "UI")
    void HideCountdownScreen();
    
    UFUNCTION(BlueprintImplementableEvent, Category = "UI")
    void ShowGameplayHUD();
    
    UFUNCTION(BlueprintImplementableEvent, Category = "UI")
    void ShowScoreboard();
};
```

**示例 3：Audio Manager 监听阶段变化**

```cpp
// GameAudioManager.h
UCLASS()
class UGameAudioManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    void Initialize(FSubsystemCollectionBase& Collection) override
    {
        Super::Initialize(Collection);
        
        // 延迟绑定（等待 World 创建）
        FWorldDelegates::OnPostWorldInitialization.AddUObject(
            this, &UGameAudioManager::OnWorldInitialized
        );
    }
    
protected:
    void OnWorldInitialized(UWorld* World, const UWorld::InitializationValues IVS)
    {
        if (!World || World->IsNetMode(NM_DedicatedServer))
        {
            return; // Dedicated Server 不需要音频
        }
        
        ULyraGamePhaseSubsystem* PhaseSubsystem = 
            World->GetSubsystem<ULyraGamePhaseSubsystem>();
        
        if (PhaseSubsystem)
        {
            PhaseSubsystem->OnPhaseStarted.AddDynamic(
                this, &UGameAudioManager::OnPhaseStarted
            );
        }
    }
    
    UFUNCTION()
    void OnPhaseStarted(const FGameplayTag& PhaseTag)
    {
        if (PhaseTag.MatchesTag(TAG_GamePhase_WaitingForPlayers))
        {
            PlayMusic(LobbyMusic);
        }
        else if (PhaseTag.MatchesTag(TAG_GamePhase_Countdown))
        {
            PlaySound(CountdownSound);
        }
        else if (PhaseTag.MatchesTag(TAG_GamePhase_InProgress))
        {
            PlayMusic(CombatMusic);
        }
        else if (PhaseTag.MatchesTag(TAG_GamePhase_PostMatch))
        {
            PlayMusic(VictoryMusic);
        }
    }
    
    void PlayMusic(USoundBase* Music);
    void PlaySound(USoundBase* Sound);
    
private:
    UPROPERTY()
    USoundBase* LobbyMusic;
    
    UPROPERTY()
    USoundBase* CountdownSound;
    
    UPROPERTY()
    USoundBase* CombatMusic;
    
    UPROPERTY()
    USoundBase* VictoryMusic;
};
```

#### 2.4.3 Blueprint 观察者示例

**在 Blueprint 中监听阶段事件**：

```
1. 创建 Actor Blueprint (例如: BP_MatchController)

2. 在 Event Graph 中:

   ┌──────────────────────────────────┐
   │ Event BeginPlay                  │
   └────────────┬─────────────────────┘
                │
                ↓
   ┌──────────────────────────────────┐
   │ Get Game Instance                │
   └────────────┬─────────────────────┘
                │
                ↓
   ┌──────────────────────────────────┐
   │ Get Subsystem                    │
   │ (ULyraGamePhaseSubsystem)        │
   └────────────┬─────────────────────┘
                │
                ↓
   ┌──────────────────────────────────┐
   │ Bind Event to OnPhaseStarted     │
   │ → OnPhaseStarted (Custom Event)  │
   └──────────────────────────────────┘

3. 创建自定义事件 OnPhaseStarted:

   ┌──────────────────────────────────┐
   │ OnPhaseStarted                   │
   │ Inputs: PhaseTag (GameplayTag)   │
   └────────────┬─────────────────────┘
                │
                ↓
   ┌──────────────────────────────────┐
   │ Switch on Gameplay Tag           │
   │ Tag: PhaseTag                    │
   ├──────────────────────────────────┤
   │ Case: GamePhase.WaitingForPlayers│
   │   → Print String "Waiting..."    │
   ├──────────────────────────────────┤
   │ Case: GamePhase.Countdown        │
   │   → Start Countdown Timer        │
   ├──────────────────────────────────┤
   │ Case: GamePhase.InProgress       │
   │   → Enable Player Input          │
   ├──────────────────────────────────┤
   │ Case: GamePhase.PostMatch        │
   │   → Show Scoreboard              │
   └──────────────────────────────────┘
```

### 2.5 网络同步机制

#### 2.5.1 GAS 自动同步

Phase Ability 的网络同步由 GAS 自动处理：

```cpp
// 服务器端：激活 Phase Ability
void ULyraGamePhaseSubsystem::StartPhase(TSubclassOf<ULyraGamePhaseAbility> PhaseAbility, ...)
{
    // 在 Game State ASC 上激活 Ability
    PhaseASC->TryActivateAbility(SpecHandle);
    
    // GAS 自动做的事情：
    // 1. 通过 Ability Replication 同步 Ability 激活状态
    // 2. 如果 Ability 有 Replicated 属性，自动同步
    // 3. Gameplay Tags 的变化自动复制
    // 4. Gameplay Events 可以选择性复制
}

// 客户端：自动接收到 Ability 激活
void ULyraGamePhaseAbility::ActivateAbility(...)
{
    Super::ActivateAbility(...);
    
    // 客户端也会执行 OnPhaseBegin()
    // 可以在这里做客户端特定的逻辑（如播放特效）
    OnPhaseBegin();
}
```

**GAS 同步的内容**：
- ✅ Ability 激活/结束状态
- ✅ Gameplay Tags 添加/移除
- ✅ Gameplay Events (如果标记为 Replicate)
- ✅ Gameplay Attributes 变化
- ✅ Ability Instance 数据 (如果标记为 Replicated)

#### 2.5.2 手动同步补充数据

有些数据需要通过 Game State 手动同步：

```cpp
// LyraGameState.h
UCLASS()
class ALyraGameState : public AModularGameStateBase
{
    GENERATED_BODY()
    
public:
    // 当前阶段 Tag（手动复制）
    UPROPERTY(ReplicatedUsing = OnRep_CurrentPhaseTag)
    FGameplayTag CurrentPhaseTag;
    
    // 阶段剩余时间（定期更新）
    UPROPERTY(Replicated)
    float PhaseRemainingTime;
    
    // 阶段自定义数据（例如：倒计时开始时间）
    UPROPERTY(Replicated)
    FGamePhaseReplicationData PhaseData;
    
protected:
    UFUNCTION()
    void OnRep_CurrentPhaseTag()
    {
        // 客户端收到阶段变化
        UE_LOG(LogLyra, Log, TEXT("[Client] Phase changed to: %s"),
            *CurrentPhaseTag.ToString());
        
        // 触发本地事件
        OnPhaseChangedClient.Broadcast(CurrentPhaseTag);
    }
    
public:
    UPROPERTY(BlueprintAssignable)
    FOnPhaseChangedDelegate OnPhaseChangedClient;
};

// Phase Ability 更新 Game State
void ULyraGamePhaseAbility::OnPhaseBegin()
{
    if (GetActorInfo().IsNetAuthority())
    {
        ALyraGameState* GameState = GetWorld()->GetGameState<ALyraGameState>();
        if (GameState)
        {
            GameState->CurrentPhaseTag = PhaseTag;
            GameState->PhaseStartTime = GetWorld()->GetTimeSeconds();
        }
    }
}
```

#### 2.5.3 客户端预测

对于某些阶段，可以使用客户端预测减少延迟：

```cpp
// 例如：倒计时阶段可以使用客户端预测
UCLASS()
class UGamePhaseAbility_Countdown : public ULyraGamePhaseAbility
{
    GENERATED_BODY()
    
public:
    // 倒计时时长
    UPROPERTY(EditDefaultsOnly)
    float CountdownDuration = 5.0f;
    
protected:
    virtual void ActivateAbility(...) override
    {
        Super::ActivateAbility(...);
        
        // 服务器和客户端都启动本地计时器
        CountdownStartTime = GetWorld()->GetTimeSeconds();
        
        GetWorld()->GetTimerManager().SetTimer(
            CountdownTimer,
            this,
            &UGamePhaseAbility_Countdown::OnCountdownFinished,
            CountdownDuration,
            false
        );
    }
    
    void OnCountdownFinished()
    {
        if (GetActorInfo().IsNetAuthority())
        {
            // 只有服务器结束阶段
            EndPhase();
        }
        else
        {
            // 客户端只是本地通知
            // (服务器会通过 GAS 同步实际的阶段结束)
            OnCountdownFinishedClient();
        }
    }
    
    UFUNCTION(BlueprintImplementableEvent)
    void OnCountdownFinishedClient();
    
private:
    FTimerHandle CountdownTimer;
    float CountdownStartTime;
};
```

**客户端预测的好处**：
- ⚡ 客户端立即响应，无需等待服务器
- ⚡ 倒计时 UI 更流畅
- ⚡ 减少感知延迟

**注意事项**：
- ⚠️ 服务器是权威的，客户端预测可能被纠正
- ⚠️ 只用于视觉反馈，不影响游戏逻辑
- ⚠️ 需要处理预测失败的情况

#### 2.5.4 网络优化技巧

**1. 减少不必要的复制**

```cpp
// 只在必要时更新复制变量
void UGamePhaseAbility_InProgress::Tick(float DeltaTime)
{
    if (GetActorInfo().IsNetAuthority())
    {
        PhaseElapsedTime += DeltaTime;
        
        // 每秒更新一次剩余时间（而不是每帧）
        if (FMath::FloorToInt(PhaseElapsedTime) != LastReplicatedTime)
        {
            LastReplicatedTime = FMath::FloorToInt(PhaseElapsedTime);
            
            ALyraGameState* GameState = GetWorld()->GetGameState<ALyraGameState>();
            if (GameState)
            {
                GameState->PhaseRemainingTime = PhaseDuration - PhaseElapsedTime;
            }
        }
    }
}
```

**2. 使用 Gameplay Events 传递数据**

```cpp
// 通过 Gameplay Event 发送阶段特定数据
void UGamePhaseAbility_Countdown::BroadcastCountdownTick(int32 SecondsRemaining)
{
    FGameplayEventData EventData;
    EventData.EventTag = TAG_Event_Countdown_Tick;
    EventData.EventMagnitude = SecondsRemaining;
    
    // 发送给所有玩家
    for (APlayerState* PS : GetWorld()->GetGameState()->PlayerArray)
    {
        if (IAbilitySystemInterface* ASI = Cast<IAbilitySystemInterface>(PS->GetPawn()))
        {
            UAbilitySystemComponent* ASC = ASI->GetAbilitySystemComponent();
            ASC->HandleGameplayEvent(TAG_Event_Countdown_Tick, &EventData);
        }
    }
}
```

**3. 批量同步数据**

```cpp
// 将多个相关数据打包成一个结构体复制
USTRUCT()
struct FGamePhaseReplicationData
{
    GENERATED_BODY()
    
    UPROPERTY()
    FGameplayTag CurrentPhase;
    
    UPROPERTY()
    float StartTime;
    
    UPROPERTY()
    float Duration;
    
    UPROPERTY()
    TArray<FGameplayTag> ActiveModifiers;
};

// 一次性复制整个结构体（减少网络开销）
UPROPERTY(Replicated)
FGamePhaseReplicationData PhaseData;
```

---

## 3. Phase Ability 阶段技能系统

### 3.1 什么是 Phase Ability

**Phase Ability** 是 Lyra 游戏阶段系统的核心实现单元。它是一个特殊的 **Gameplay Ability**，专门用于表示和执行游戏的某个阶段逻辑。

#### 3.1.1 Phase Ability 的特点

**1. 继承自 Gameplay Ability**

```cpp
// Phase Ability 是 Gameplay Ability 的子类
UCLASS()
class ULyraGamePhaseAbility : public ULyraGameplayAbility
{
    GENERATED_BODY()
    
    // 自动获得 GAS 的所有特性：
    // - 网络同步
    // - Gameplay Tags 管理
    // - Ability Tasks 支持
    // - Gameplay Effects 集成
};
```

**2. 长期激活**

```cpp
// 普通 Ability：短暂激活（如技能释放）
void UMyAttackAbility::ActivateAbility(...)
{
    PlayMontage();
    ApplyDamage();
    EndAbility(); // 立即结束
}

// Phase Ability：长期激活（直到阶段结束）
void UGamePhaseAbility_InProgress::ActivateAbility(...)
{
    Super::ActivateAbility(...);
    OnPhaseBegin();
    
    // 不调用 EndAbility()，保持激活状态
    // 直到阶段转换条件满足
}
```

**3. 唯一性**

```cpp
// 同一时间只有一个相同 Phase Tag 的 Ability 激活
UCLASS()
class ULyraGamePhaseAbility : public ULyraGameplayAbility
{
    // 阶段标签（唯一标识）
    UPROPERTY(EditDefaultsOnly, Category = "Phase")
    FGameplayTag PhaseTag;
    
    // 激活策略：单例模式
    UPROPERTY(EditDefaultsOnly, Category = "Phase")
    EGameplayAbilityInstancingPolicy::Type InstancingPolicy = 
        EGameplayAbilityInstancingPolicy::InstancedPerActor;
};
```

**4. 事件驱动**

```cpp
// Phase Ability 通过事件响应外部变化
void UGamePhaseAbility_WaitingForPlayers::OnPhaseBegin()
{
    // 监听玩家加入事件
    ListenForPlayerJoined();
    
    // 监听准备就绪事件
    ListenForAllPlayersReady();
}

void UGamePhaseAbility_WaitingForPlayers::OnPlayerJoined()
{
    // 检查是否可以转换到下一阶段
    if (CanTransitionToNextPhase())
    {
        TransitionToNextPhase();
    }
}
```

#### 3.1.2 Phase Ability vs 普通 Ability

| 特性 | Phase Ability | 普通 Ability |
|------|---------------|--------------|
| **生命周期** | 长期激活（分钟级别） | 短暂激活（秒级别） |
| **激活位置** | Game State ASC | Player/Character ASC |
| **网络权威** | 仅服务器 | 服务器 + 客户端预测 |
| **用途** | 游戏流程管理 | 玩家技能/交互 |
| **实例数量** | 同时只有一个 | 可以有多个 |
| **Tags** | Phase Tags | Ability Tags |

#### 3.1.3 Phase Ability 的职责

**✅ 应该做的事情**：

1. **阶段初始化**
   ```cpp
   void OnPhaseBegin()
   {
       // 设置阶段特定的游戏规则
       SetGameRules();
       
       // 生成阶段相关的对象
       SpawnPhaseActors();
       
       // 配置玩家状态
       ConfigurePlayerStates();
   }
   ```

2. **阶段状态管理**
   ```cpp
   void Tick(float DeltaTime)
   {
       // 更新倒计时
       UpdateTimer(DeltaTime);
       
       // 检查转换条件
       if (ShouldTransitionToNextPhase())
       {
           TransitionToNextPhase();
       }
   }
   ```

3. **事件监听与响应**
   ```cpp
   void OnPhaseBegin()
   {
       // 监听游戏事件
       UAbilityTask_WaitGameplayEvent* WaitTask = 
           UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
               this, TAG_Event_AllPlayersReady
           );
       
       WaitTask->EventReceived.AddDynamic(this, &ThisClass::OnAllPlayersReady);
       WaitTask->ReadyForActivation();
   }
   ```

4. **阶段清理**
   ```cpp
   void OnPhaseEnd()
   {
       // 清理阶段相关的对象
       CleanupPhaseActors();
       
       // 重置玩家状态
       ResetPlayerStates();
       
       // 保存阶段统计数据
       SavePhaseStats();
   }
   ```

**❌ 不应该做的事情**：

1. **直接操作 UI**
   ```cpp
   // ❌ 错误示范
   void OnPhaseBegin()
   {
       // 不要在 Phase Ability 中直接操作 UI
       MyHUDWidget->ShowCountdown();
   }
   
   // ✅ 正确做法：通过事件通知
   void OnPhaseBegin()
   {
       // 发送 Gameplay Event
       FGameplayEventData EventData;
       EventData.EventTag = TAG_Event_Phase_CountdownStarted;
       SendGameplayEvent(EventData);
       
       // UI 监听事件并自行更新
   }
   ```

2. **处理玩家输入**
   ```cpp
   // ❌ 错误示范
   void OnPhaseBegin()
   {
       // Phase Ability 不应该直接处理输入
       BindInput();
   }
   
   // ✅ 正确做法：通过 Gameplay Effect 控制
   void OnPhaseBegin()
   {
       // 应用一个禁用输入的 GE
       ApplyGameplayEffectToAllPlayers(GE_DisableInput);
   }
   ```

3. **包含游戏模式特定逻辑**
   ```cpp
   // ❌ 错误示范
   void OnPhaseBegin()
   {
       // 不要在通用 Phase Ability 中写模式特定逻辑
       if (GameMode == "TDM")
       {
           SetupTDMRules();
       }
       else if (GameMode == "CTF")
       {
           SetupCTFRules();
       }
   }
   
   // ✅ 正确做法：创建子类
   class UGamePhaseAbility_InProgress_TDM : public UGamePhaseAbility_InProgress
   {
       virtual void OnPhaseBegin() override
       {
           Super::OnPhaseBegin();
           SetupTDMRules();
       }
   };
   ```

### 3.2 ULyraGamePhaseAbility 详解

#### 3.2.1 完整类定义

```cpp
// LyraGamePhaseAbility.h
#pragma once

#include "CoreMinimal.h"
#include "LyraGameplayAbility.h"
#include "GameplayTagContainer.h"
#include "LyraGamePhaseAbility.generated.h"

class ULyraGamePhaseSubsystem;

/**
 * Lyra 游戏阶段 Ability 基类
 * 
 * 所有游戏阶段都应该继承此类
 * 提供阶段生命周期钩子和工具函数
 */
UCLASS(Abstract)
class LYRAGAME_API ULyraGamePhaseAbility : public ULyraGameplayAbility
{
    GENERATED_BODY()
    
public:
    ULyraGamePhaseAbility(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
    
    // ============================================================
    // 阶段配置
    // ============================================================
    
    /**
     * 阶段标签
     * 用于唯一标识此阶段
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Phase", meta = (Categories = "GamePhase"))
    FGameplayTag PhaseTag;
    
    /**
     * 阶段显示名称（用于 UI 和调试）
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Phase")
    FText PhaseName;
    
    /**
     * 阶段持续时间（秒，0 表示无限）
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Phase|Duration")
    float PhaseDuration = 0.0f;
    
    /**
     * 是否使用定时器自动结束
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Phase|Duration")
    bool bUseAutoEndTimer = false;
    
    /**
     * 阶段结束前的警告时间（秒）
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Phase|Duration", 
        meta = (EditCondition = "bUseAutoEndTimer"))
    float WarningTimeBeforeEnd = 10.0f;
    
    // ============================================================
    // 阶段行为配置
    // ============================================================
    
    /**
     * 阶段开始时应用的 Gameplay Effects
     * 例如：冻结玩家移动、无敌状态等
     */
    UPROPERTY(EditDefaultsOnly, Category = "Phase|Effects")
    TArray<TSubclassOf<UGameplayEffect>> PhaseEffectsToApply;
    
    /**
     * 阶段开始时添加的 Gameplay Tags
     */
    UPROPERTY(EditDefaultsOnly, Category = "Phase|Tags")
    FGameplayTagContainer PhaseTagsToAdd;
    
    /**
     * 阶段开始时移除的 Gameplay Tags
     */
    UPROPERTY(EditDefaultsOnly, Category = "Phase|Tags")
    FGameplayTagContainer PhaseTagsToRemove;
    
    /**
     * 是否在阶段开始时禁用玩家输入
     */
    UPROPERTY(EditDefaultsOnly, Category = "Phase|Player Control")
    bool bDisablePlayerInputOnStart = false;
    
    /**
     * 是否在阶段开始时冻结玩家移动
     */
    UPROPERTY(EditDefaultsOnly, Category = "Phase|Player Control")
    bool bFreezePlayerMovementOnStart = false;
    
    // ============================================================
    // Ability 重写
    // ============================================================
    
    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData
    ) override;
    
    virtual void EndAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility,
        bool bWasCancelled
    ) override;
    
    virtual bool CanActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayTagContainer* SourceTags,
        const FGameplayTagContainer* TargetTags,
        FGameplayTagContainer* OptionalRelevantTags
    ) const override;
    
    // ============================================================
    // 阶段生命周期钩子
    // ============================================================
    
protected:
    /**
     * 阶段开始时调用
     * 在此处执行阶段初始化逻辑
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase")
    void OnPhaseBegin();
    virtual void OnPhaseBegin_Implementation();
    
    /**
     * 阶段结束时调用
     * 在此处执行清理逻辑
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase")
    void OnPhaseEnd();
    virtual void OnPhaseEnd_Implementation();
    
    /**
     * 阶段每帧更新（如果需要）
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase")
    void OnPhaseTick(float DeltaTime);
    virtual void OnPhaseTick_Implementation(float DeltaTime);
    
    /**
     * 检查是否可以转换到下一阶段
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase")
    bool CanTransitionToNextPhase() const;
    virtual bool CanTransitionToNextPhase_Implementation() const;
    
    /**
     * 阶段即将结束时的警告（在 WarningTimeBeforeEnd 前调用）
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase")
    void OnPhaseEndWarning(float TimeRemaining);
    virtual void OnPhaseEndWarning_Implementation(float TimeRemaining);
    
    // ============================================================
    // 暂停/恢复支持
    // ============================================================
    
    /**
     * 阶段被暂停
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase")
    void OnPhasePaused();
    virtual void OnPhasePaused_Implementation();
    
    /**
     * 阶段恢复
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Phase")
    void OnPhaseResumed();
    virtual void OnPhaseResumed_Implementation();
    
    // ============================================================
    // 工具函数
    // ============================================================
    
public:
    /**
     * 获取阶段标签
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    FGameplayTag GetPhaseTag() const { return PhaseTag; }
    
    /**
     * 获取阶段子系统
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    ULyraGamePhaseSubsystem* GetPhaseSubsystem() const;
    
    /**
     * 获取阶段已运行时间
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    float GetPhaseElapsedTime() const;
    
    /**
     * 获取阶段剩余时间
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    float GetPhaseRemainingTime() const;
    
    /**
     * 获取阶段进度 (0.0 - 1.0)
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    float GetPhaseProgress() const;
    
protected:
    /**
     * 通知阶段子系统阶段已开始
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    void NotifyPhaseStarted();
    
    /**
     * 通知阶段子系统阶段已结束
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    void NotifyPhaseEnded();
    
    /**
     * 转换到下一个阶段
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    void TransitionToNextPhase();
    
    /**
     * 应用阶段 Gameplay Effects 到所有玩家
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    void ApplyPhaseEffectsToAllPlayers();
    
    /**
     * 移除阶段 Gameplay Effects
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    void RemovePhaseEffectsFromAllPlayers();
    
    /**
     * 发送阶段相关的 Gameplay Event
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    void BroadcastPhaseEvent(const FGameplayTag& EventTag, const FGameplayEventData& EventData);
    
    /**
     * 获取所有玩家的 ASC
     */
    UFUNCTION(BlueprintCallable, Category = "Phase")
    TArray<UAbilitySystemComponent*> GetAllPlayerASCs() const;
    
private:
    // 阶段开始时间
    float PhaseStartTime = 0.0f;
    
    // 已应用的 GE Handles
    TArray<FActiveGameplayEffectHandle> AppliedPhaseEffects;
    
    // 自动结束定时器
    FTimerHandle AutoEndTimerHandle;
    
    // 警告定时器
    FTimerHandle WarningTimerHandle;
    
    // Tick 定时器
    FTimerHandle TickTimerHandle;
    
    // 是否已触发警告
    bool bHasTriggeredWarning = false;
};
```

#### 3.2.2 核心函数实现

**1. Ability 激活**

```cpp
// LyraGamePhaseAbility.cpp

void ULyraGamePhaseAbility::ActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
    
    // 1. 记录开始时间
    PhaseStartTime = GetWorld()->GetTimeSeconds();
    
    // 2. 添加 Phase Tags
    if (!PhaseTagsToAdd.IsEmpty())
    {
        ApplyGameplayTagsToOwner(PhaseTagsToAdd);
    }
    
    // 3. 移除指定 Tags
    if (!PhaseTagsToRemove.IsEmpty())
    {
        RemoveGameplayTagsFromOwner(PhaseTagsToRemove);
    }
    
    // 4. 应用 Phase Effects
    if (!PhaseEffectsToApply.IsEmpty())
    {
        ApplyPhaseEffectsToAllPlayers();
    }
    
    // 5. 处理玩家控制
    if (bDisablePlayerInputOnStart || bFreezePlayerMovementOnStart)
    {
        ConfigurePlayerControl();
    }
    
    // 6. 启动自动结束定时器
    if (bUseAutoEndTimer && PhaseDuration > 0.0f)
    {
        GetWorld()->GetTimerManager().SetTimer(
            AutoEndTimerHandle,
            this,
            &ULyraGamePhaseAbility::OnAutoEndTimer,
            PhaseDuration,
            false
        );
        
        // 启动警告定时器
        if (WarningTimeBeforeEnd > 0.0f && WarningTimeBeforeEnd < PhaseDuration)
        {
            float WarningDelay = PhaseDuration - WarningTimeBeforeEnd;
            GetWorld()->GetTimerManager().SetTimer(
                WarningTimerHandle,
                this,
                &ULyraGamePhaseAbility::OnWarningTimer,
                WarningDelay,
                false
            );
        }
    }
    
    // 7. 启动 Tick（如果需要）
    if (ShouldTick())
    {
        GetWorld()->GetTimerManager().SetTimer(
            TickTimerHandle,
            this,
            &ULyraGamePhaseAbility::TickPhase,
            GetTickInterval(),
            true // 循环
        );
    }
    
    // 8. 调用子类钩子
    OnPhaseBegin();
    
    // 9. 通知子系统
    NotifyPhaseStarted();
    
    UE_LOG(LogLyra, Log, TEXT("[Phase] '%s' started (Duration: %.1fs)"),
        *PhaseName.ToString(), PhaseDuration);
}
```

**2. Ability 结束**

```cpp
void ULyraGamePhaseAbility::EndAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    bool bReplicateEndAbility,
    bool bWasCancelled)
{
    // 1. 调用子类钩子
    OnPhaseEnd();
    
    // 2. 清理定时器
    UWorld* World = GetWorld();
    if (World)
    {
        World->GetTimerManager().ClearTimer(AutoEndTimerHandle);
        World->GetTimerManager().ClearTimer(WarningTimerHandle);
        World->GetTimerManager().ClearTimer(TickTimerHandle);
    }
    
    // 3. 移除 Phase Effects
    RemovePhaseEffectsFromAllPlayers();
    
    // 4. 移除 Phase Tags
    if (!PhaseTagsToAdd.IsEmpty())
    {
        RemoveGameplayTagsFromOwner(PhaseTagsToAdd);
    }
    
    // 5. 恢复玩家控制
    if (bDisablePlayerInputOnStart || bFreezePlayerMovementOnStart)
    {
        RestorePlayerControl();
    }
    
    // 6. 通知子系统
    NotifyPhaseEnded();
    
    float Duration = World ? (World->GetTimeSeconds() - PhaseStartTime) : 0.0f;
    UE_LOG(LogLyra, Log, TEXT("[Phase] '%s' ended (Actual Duration: %.1fs, Cancelled: %d)"),
        *PhaseName.ToString(), Duration, bWasCancelled);
    
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

**3. 生命周期钩子默认实现**

```cpp
void ULyraGamePhaseAbility::OnPhaseBegin_Implementation()
{
    // 默认为空，子类重写
    UE_LOG(LogLyra, Verbose, TEXT("[Phase] OnPhaseBegin: %s"), *PhaseName.ToString());
}

void ULyraGamePhaseAbility::OnPhaseEnd_Implementation()
{
    // 默认为空，子类重写
    UE_LOG(LogLyra, Verbose, TEXT("[Phase] OnPhaseEnd: %s"), *PhaseName.ToString());
}

void ULyraGamePhaseAbility::OnPhaseTick_Implementation(float DeltaTime)
{
    // 默认为空，子类可以重写实现每帧逻辑
}

bool ULyraGamePhaseAbility::CanTransitionToNextPhase_Implementation() const
{
    // 默认：如果有定时器，定时器到期后可以转换
    if (bUseAutoEndTimer)
    {
        return GetPhaseRemainingTime() <= 0.0f;
    }
    
    // 否则需要子类重写此函数
    return false;
}

void ULyraGamePhaseAbility::OnPhaseEndWarning_Implementation(float TimeRemaining)
{
    // 发送警告事件
    FGameplayEventData EventData;
    EventData.EventTag = TAG_Event_Phase_EndWarning;
    EventData.EventMagnitude = TimeRemaining;
    
    BroadcastPhaseEvent(TAG_Event_Phase_EndWarning, EventData);
    
    UE_LOG(LogLyra, Log, TEXT("[Phase] End warning: %.1fs remaining"), TimeRemaining);
}
```

**4. 工具函数实现**

```cpp
ULyraGamePhaseSubsystem* ULyraGamePhaseAbility::GetPhaseSubsystem() const
{
    return GetWorld()->GetSubsystem<ULyraGamePhaseSubsystem>();
}

float ULyraGamePhaseAbility::GetPhaseElapsedTime() const
{
    if (PhaseStartTime <= 0.0f)
    {
        return 0.0f;
    }
    
    return GetWorld()->GetTimeSeconds() - PhaseStartTime;
}

float ULyraGamePhaseAbility::GetPhaseRemainingTime() const
{
    if (PhaseDuration <= 0.0f)
    {
        return 0.0f; // 无限持续
    }
    
    return FMath::Max(0.0f, PhaseDuration - GetPhaseElapsedTime());
}

float ULyraGamePhaseAbility::GetPhaseProgress() const
{
    if (PhaseDuration <= 0.0f)
    {
        return 0.0f; // 无法计算进度
    }
    
    return FMath::Clamp(GetPhaseElapsedTime() / PhaseDuration, 0.0f, 1.0f);
}

void ULyraGamePhaseAbility::NotifyPhaseStarted()
{
    if (ULyraGamePhaseSubsystem* PhaseSubsystem = GetPhaseSubsystem())
    {
        PhaseSubsystem->OnPhaseAbilityActivated(this);
    }
}

void ULyraGamePhaseAbility::NotifyPhaseEnded()
{
    if (ULyraGamePhaseSubsystem* PhaseSubsystem = GetPhaseSubsystem())
    {
        PhaseSubsystem->OnPhaseAbilityEnded(this);
    }
}

void ULyraGamePhaseAbility::TransitionToNextPhase()
{
    // 结束当前阶段（触发阶段转换逻辑）
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}
```

**5. 应用 Phase Effects 到所有玩家**

```cpp
void ULyraGamePhaseAbility::ApplyPhaseEffectsToAllPlayers()
{
    if (PhaseEffectsToApply.IsEmpty())
    {
        return;
    }
    
    TArray<UAbilitySystemComponent*> PlayerASCs = GetAllPlayerASCs();
    
    for (TSubclassOf<UGameplayEffect> EffectClass : PhaseEffectsToApply)
    {
        if (!EffectClass)
        {
            continue;
        }
        
        for (UAbilitySystemComponent* ASC : PlayerASCs)
        {
            if (!ASC)
            {
                continue;
            }
            
            FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
            EffectContext.AddSourceObject(this);
            
            FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingSpec(
                EffectClass,
                1.0f, // Level
                EffectContext
            );
            
            if (SpecHandle.IsValid())
            {
                FActiveGameplayEffectHandle ActiveHandle = 
                    ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
                
                if (ActiveHandle.IsValid())
                {
                    AppliedPhaseEffects.Add(ActiveHandle);
                }
            }
        }
    }
    
    UE_LOG(LogLyra, Log, TEXT("[Phase] Applied %d effects to %d players"),
        PhaseEffectsToApply.Num(), PlayerASCs.Num());
}

void ULyraGamePhaseAbility::RemovePhaseEffectsFromAllPlayers()
{
    TArray<UAbilitySystemComponent*> PlayerASCs = GetAllPlayerASCs();
    
    for (const FActiveGameplayEffectHandle& Handle : AppliedPhaseEffects)
    {
        for (UAbilitySystemComponent* ASC : PlayerASCs)
        {
            if (ASC && Handle.IsValid())
            {
                ASC->RemoveActiveGameplayEffect(Handle);
            }
        }
    }
    
    AppliedPhaseEffects.Empty();
}

TArray<UAbilitySystemComponent*> ULyraGamePhaseAbility::GetAllPlayerASCs() const
{
    TArray<UAbilitySystemComponent*> ASCs;
    
    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (!GameState)
    {
        return ASCs;
    }
    
    for (APlayerState* PS : GameState->PlayerArray)
    {
        if (!PS)
        {
            continue;
        }
        
        APawn* Pawn = PS->GetPawn();
        if (!Pawn)
        {
            continue;
        }
        
        if (IAbilitySystemInterface* ASI = Cast<IAbilitySystemInterface>(Pawn))
        {
            if (UAbilitySystemComponent* ASC = ASI->GetAbilitySystemComponent())
            {
                ASCs.Add(ASC);
            }
        }
    }
    
    return ASCs;
}
```

**6. 玩家控制配置**

```cpp
void ULyraGamePhaseAbility::ConfigurePlayerControl()
{
    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (!GameState)
    {
        return;
    }
    
    for (APlayerState* PS : GameState->PlayerArray)
    {
        APlayerController* PC = Cast<APlayerController>(PS->GetPlayerController());
        if (!PC)
        {
            continue;
        }
        
        // 禁用输入
        if (bDisablePlayerInputOnStart)
        {
            PC->DisableInput(PC);
        }
        
        // 冻结移动
        if (bFreezePlayerMovementOnStart)
        {
            if (APawn* Pawn = PC->GetPawn())
            {
                Pawn->DisableInput(PC);
                
                // 也可以通过 GE 实现
                // ApplyGameplayEffectToTarget(GE_FreezeMovement, Pawn);
            }
        }
    }
}

void ULyraGamePhaseAbility::RestorePlayerControl()
{
    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (!GameState)
    {
        return;
    }
    
    for (APlayerState* PS : GameState->PlayerArray)
    {
        APlayerController* PC = Cast<APlayerController>(PS->GetPlayerController());
        if (!PC)
        {
            continue;
        }
        
        if (bDisablePlayerInputOnStart)
        {
            PC->EnableInput(PC);
        }
        
        if (bFreezePlayerMovementOnStart)
        {
            if (APawn* Pawn = PC->GetPawn())
            {
                Pawn->EnableInput(PC);
            }
        }
    }
}
```

**7. 定时器回调**

```cpp
void ULyraGamePhaseAbility::OnAutoEndTimer()
{
    UE_LOG(LogLyra, Log, TEXT("[Phase] Auto-end timer expired for: %s"), *PhaseName.ToString());
    
    // 自动转换到下一阶段
    TransitionToNextPhase();
}

void ULyraGamePhaseAbility::OnWarningTimer()
{
    if (!bHasTriggeredWarning)
    {
        bHasTriggeredWarning = true;
        OnPhaseEndWarning(GetPhaseRemainingTime());
    }
}

void ULyraGamePhaseAbility::TickPhase()
{
    float DeltaTime = GetTickInterval();
    OnPhaseTick(DeltaTime);
    
    // 检查转换条件
    if (CanTransitionToNextPhase())
    {
        TransitionToNextPhase();
    }
}
```

### 3.3 Phase Ability 的生命周期

#### 3.3.1 完整生命周期流程图

```
┌──────────────────────────────────────────────────────────────────┐
│                     Phase Ability Lifecycle                      │
└──────────────────────────────────────────────────────────────────┘

1. [创建阶段]
   ├─ Game Mode 或 Phase Subsystem 调用 StartPhase()
   ├─ 创建 FGameplayAbilitySpec
   ├─ Spec.Ability = Phase Ability Class CDO
   ├─ Spec.SourceObject = Phase Instigator (通常是 Game Mode)
   └─ 添加到 Game State ASC 的 Ability List

2. [Can Activate 检查]
   ├─ ASC 调用 CanActivateAbility()
   ├─ 检查 Gameplay Tags (Blocking Tags, Required Tags)
   ├─ 检查网络权限 (Server Only)
   ├─ 检查实例化策略 (防止重复激活)
   └─ 如果失败 → 激活失败，清理 Spec

3. [激活阶段]
   ├─ ASC 调用 TryActivateAbility()
   ├─ 创建 Ability Instance (如果需要)
   ├─ 调用 ActivateAbility()
   │   ├─ 记录 PhaseStartTime
   │   ├─ 添加/移除 Gameplay Tags
   │   ├─ 应用 Phase Effects 到所有玩家
   │   ├─ 配置玩家控制 (禁用输入/冻结移动)
   │   ├─ 启动自动结束定时器 (如果需要)
   │   ├─ 启动 Tick 定时器 (如果需要)
   │   ├─ 调用 OnPhaseBegin() [子类钩子]
   │   └─ 调用 NotifyPhaseStarted()
   └─ Phase Subsystem 广播 OnPhaseStarted 事件

4. [运行阶段]
   ├─ 定时器 Tick: 调用 OnPhaseTick(DeltaTime)
   ├─ 监听 Gameplay Events
   ├─ 更新阶段状态 (如剩余时间)
   ├─ 检查转换条件: CanTransitionToNextPhase()
   └─ 响应外部事件 (如玩家准备好)

5. [阶段警告] (如果有定时器)
   ├─ WarningTimer 到期
   ├─ 调用 OnPhaseEndWarning(TimeRemaining)
   └─ 广播警告事件 (TAG_Event_Phase_EndWarning)

6. [阶段结束触发]
   ├─ 条件 A: AutoEndTimer 到期 → OnAutoEndTimer()
   ├─ 条件 B: CanTransitionToNextPhase() 返回 true
   ├─ 条件 C: 外部调用 EndPhase()
   └─ 调用 TransitionToNextPhase() 或 EndAbility()

7. [结束阶段]
   ├─ 调用 EndAbility()
   │   ├─ 调用 OnPhaseEnd() [子类钩子]
   │   ├─ 清理所有定时器
   │   ├─ 移除 Phase Effects
   │   ├─ 移除 Gameplay Tags
   │   ├─ 恢复玩家控制
   │   ├─ 调用 NotifyPhaseEnded()
   │   └─ Phase Subsystem 广播 OnPhaseEnded 事件
   ├─ ASC 调用 InternalEndAbility()
   ├─ 从 ActiveGameplayAbilities 移除
   └─ 清理 Ability Spec (如果配置了)

8. [阶段转换]
   ├─ Phase Subsystem 检测到阶段结束
   ├─ 查找 Phase Configuration 中的下一个阶段
   ├─ 调用 StartPhase(NextPhaseAbility)
   └─ 回到步骤 1
```

#### 3.3.2 生命周期状态图

```
┌───────────────┐
│   Inactive    │ (阶段未激活)
└───────┬───────┘
        │ StartPhase()
        ↓
┌───────────────┐
│  Activating   │ (激活中)
│  • CanActivate│
│  • CreateSpec │
└───────┬───────┘
        │ CanActivateAbility() == true
        ↓
┌───────────────┐
│    Active     │ (阶段运行中)
│  • OnPhaseBegin()
│  • Apply Effects
│  • Start Timers
└───────┬───────┘
        │
        ├─────────────┐
        │             │
        ↓             ↓
┌──────────────┐  ┌──────────────┐
│   Running    │  │    Paused    │
│  • Tick      │←→│  • No Tick   │
│  • Events    │  │  • Frozen    │
└──────┬───────┘  └──────────────┘
        │
        │ CanTransitionToNextPhase() == true
        │ OR AutoEndTimer expire
        │ OR EndPhase()
        ↓
┌───────────────┐
│   Ending      │ (结束中)
│  • OnPhaseEnd()
│  • Cleanup
└───────┬───────┘
        │
        ↓
┌───────────────┐
│   Inactive    │ (阶段已结束)
└───────────────┘
```

#### 3.3.3 生命周期事件序列

**示例：倒计时阶段的完整生命周期**

```
时间轴：

T=0.0s  [Server] Game Mode 调用 StartPhase(GA_GamePhase_Countdown)
        ├─ Phase Subsystem 创建 Ability Spec
        └─ 添加到 Game State ASC

T=0.01s [Server] ASC 检查 CanActivateAbility()
        ├─ 检查 Blocking Tags: 通过
        ├─ 检查网络权限: 通过
        └─ 返回 true

T=0.02s [Server] ASC 激活 Ability
        ├─ 创建 Ability Instance
        └─ 调用 ActivateAbility()

T=0.03s [Server] ActivateAbility() 执行
        ├─ PhaseStartTime = 0.03
        ├─ 添加 Tag: GamePhase.Countdown
        ├─ 应用 GE_FreezeMovement 到所有玩家
        ├─ 启动 AutoEndTimer (5 秒)
        ├─ 启动 WarningTimer (4 秒)
        ├─ 调用 OnPhaseBegin()
        └─ 通知 Phase Subsystem

T=0.04s [Server] Phase Subsystem 广播 OnPhaseStarted(GamePhase.Countdown)
        └─ 所有观察者收到通知

T=0.05s [Clients] 通过 GAS 网络同步收到:
        ├─ Ability 激活
        ├─ Tag 添加: GamePhase.Countdown
        ├─ GE_FreezeMovement 应用
        └─ 客户端也调用 OnPhaseBegin()

T=0.10s [Clients] UI Widget 收到 OnPhaseStarted 事件
        └─ 显示倒计时界面

T=1.0s  [All] 倒计时显示: 4
T=2.0s  [All] 倒计时显示: 3
T=3.0s  [All] 倒计时显示: 2

T=4.03s [Server] WarningTimer 到期
        ├─ 调用 OnPhaseEndWarning(1.0)
        └─ 广播 TAG_Event_Phase_EndWarning

T=4.04s [All] UI 收到警告事件
        └─ 播放"最后1秒"音效

T=5.03s [Server] AutoEndTimer 到期
        ├─ 调用 OnAutoEndTimer()
        ├─ 调用 TransitionToNextPhase()
        └─ 调用 EndAbility()

T=5.04s [Server] EndAbility() 执行
        ├─ 调用 OnPhaseEnd()
        ├─ 清理所有定时器
        ├─ 移除 GE_FreezeMovement
        ├─ 移除 Tag: GamePhase.Countdown
        ├─ 通知 Phase Subsystem
        └─ ASC 移除 Ability

T=5.05s [Server] Phase Subsystem 广播 OnPhaseEnded(GamePhase.Countdown)
        └─ 启动下一个阶段: GA_GamePhase_InProgress

T=5.06s [Clients] 通过 GAS 网络同步收到:
        ├─ Ability 结束
        ├─ Tag 移除: GamePhase.Countdown
        ├─ GE_FreezeMovement 移除
        └─ 客户端也调用 OnPhaseEnd()

T=5.10s [Clients] UI Widget 收到 OnPhaseEnded 事件
        └─ 隐藏倒计时界面

T=5.11s [All] 新阶段开始: GamePhase.InProgress
        └─ 重复上述流程...
```

#### 3.3.4 生命周期钩子调用顺序

```cpp
// 详细的钩子调用顺序（包括 GAS 内部调用）

// ============================================================
// 激活流程
// ============================================================

1. UAbilitySystemComponent::TryActivateAbility()
2. ULyraGamePhaseAbility::CanActivateAbility()      [可重写]
3. UAbilitySystemComponent::InternalTryActivateAbility()
4. ULyraGamePhaseAbility::ActivateAbility()         [重写]
   ├─ Super::ActivateAbility()                      [ULyraGameplayAbility]
   │   └─ Super::ActivateAbility()                  [UGameplayAbility]
   │       └─ CommitAbility()                       [消耗 Cost、应用 Cooldown]
   ├─ [内部初始化逻辑]
   ├─ OnPhaseBegin_Implementation()                 [子类钩子]
   └─ NotifyPhaseStarted()
       └─ ULyraGamePhaseSubsystem::OnPhaseAbilityActivated()
           └─ BroadcastPhaseStarted()
               └─ OnPhaseStarted.Broadcast()        [观察者收到通知]

// ============================================================
// 运行流程
// ============================================================

5. [Tick 循环]
   ULyraGamePhaseAbility::TickPhase()              [定时器回调]
   └─ OnPhaseTick_Implementation()                 [子类钩子]
       └─ CanTransitionToNextPhase_Implementation() [检查条件]

6. [事件响应]
   UAbilityTask_WaitGameplayEvent::OnEventReceived()
   └─ ULyraGamePhaseAbility::OnGameplayEvent()     [事件处理]

// ============================================================
// 结束流程
// ============================================================

7. ULyraGamePhaseAbility::TransitionToNextPhase()
8. ULyraGamePhaseAbility::EndAbility()              [重写]
   ├─ OnPhaseEnd_Implementation()                   [子类钩子]
   ├─ [内部清理逻辑]
   ├─ NotifyPhaseEnded()
   │   └─ ULyraGamePhaseSubsystem::OnPhaseAbilityEnded()
   │       └─ BroadcastPhaseEnded()
   │           └─ OnPhaseEnded.Broadcast()          [观察者收到通知]
   └─ Super::EndAbility()                           [UGameplayAbility]
       └─ UAbilitySystemComponent::InternalEndAbility()
```

### 3.4 创建自定义 Phase Ability

#### 3.4.1 C++ 创建示例：倒计时阶段

**1. 头文件定义**

```cpp
// GamePhaseAbility_Countdown.h
#pragma once

#include "LyraGamePhaseAbility.h"
#include "GamePhaseAbility_Countdown.generated.h"

/**
 * 倒计时阶段 Ability
 * 
 * 功能：
 * - 显示倒计时界面
 * - 冻结玩家移动和输入
 * - 播放倒计时音效
 * - 定时器到期后自动转换到游戏阶段
 */
UCLASS()
class MYGAME_API UGamePhaseAbility_Countdown : public ULyraGamePhaseAbility
{
    GENERATED_BODY()
    
public:
    UGamePhaseAbility_Countdown();
    
    // ============================================================
    // 配置
    // ============================================================
    
    /**
     * 倒计时时长（秒）
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Countdown")
    float CountdownDuration = 5.0f;
    
    /**
     * 每秒播放的倒计时音效
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Countdown")
    TArray<USoundBase*> CountdownSounds;
    
    /**
     * 倒计时最后一秒的特殊音效
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Countdown")
    USoundBase* FinalCountdownSound;
    
    /**
     * 倒计时结束时的音效
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Countdown")
    USoundBase* CountdownFinishedSound;
    
    /**
     * 是否在倒计时结束前震动手柄
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Countdown")
    bool bVibrateOnFinalSecond = true;
    
protected:
    // ============================================================
    // Phase Ability 重写
    // ============================================================
    
    virtual void OnPhaseBegin_Implementation() override;
    virtual void OnPhaseEnd_Implementation() override;
    virtual void OnPhaseTick_Implementation(float DeltaTime) override;
    virtual bool CanTransitionToNextPhase_Implementation() const override;
    
    // ============================================================
    // 倒计时逻辑
    // ============================================================
    
    /**
     * 更新倒计时
     */
    UFUNCTION(BlueprintCallable, Category = "Countdown")
    void UpdateCountdown(float DeltaTime);
    
    /**
     * 广播倒计时更新事件
     */
    UFUNCTION(BlueprintCallable, Category = "Countdown")
    void BroadcastCountdownTick(int32 SecondsRemaining);
    
    /**
     * 播放倒计时音效
     */
    UFUNCTION(BlueprintCallable, Category = "Countdown")
    void PlayCountdownSound(int32 SecondValue);
    
    /**
     * 震动所有玩家的手柄
     */
    UFUNCTION(BlueprintCallable, Category = "Countdown")
    void VibrateAllControllers();
    
private:
    // 当前剩余秒数
    float CurrentCountdown = 0.0f;
    
    // 上一次广播的秒数（避免重复广播）
    int32 LastBroadcastedSecond = -1;
};
```

**2. CPP 实现**

```cpp
// GamePhaseAbility_Countdown.cpp

#include "GamePhaseAbility_Countdown.h"
#include "LyraGamePhaseSubsystem.h"
#include "LyraLogChannels.h"
#include "AbilitySystemComponent.h"
#include "GameFramework/PlayerController.h"
#include "GameFramework/GameStateBase.h"
#include "Kismet/GameplayStatics.h"

UGamePhaseAbility_Countdown::UGamePhaseAbility_Countdown()
{
    // 基本配置
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerOnly;
    
    // 阶段配置
    PhaseTag = FGameplayTag::RequestGameplayTag(TEXT("GamePhase.Countdown"));
    PhaseName = NSLOCTEXT("GamePhase", "Countdown", "倒计时");
    PhaseDuration = 5.0f;
    bUseAutoEndTimer = true;
    
    // 玩家控制
    bDisablePlayerInputOnStart = true;
    bFreezePlayerMovementOnStart = true;
    
    // Tags to add
    PhaseTagsToAdd.AddTag(FGameplayTag::RequestGameplayTag(TEXT("State.Countdown")));
    PhaseTagsToAdd.AddTag(FGameplayTag::RequestGameplayTag(TEXT("State.CannotMove")));
    PhaseTagsToAdd.AddTag(FGameplayTag::RequestGameplayTag(TEXT("State.CannotAttack")));
}

void UGamePhaseAbility_Countdown::OnPhaseBegin_Implementation()
{
    Super::OnPhaseBegin_Implementation();
    
    // 初始化倒计时
    CurrentCountdown = CountdownDuration;
    LastBroadcastedSecond = -1;
    
    UE_LOG(LogLyra, Log, TEXT("[Countdown] Phase started, duration: %.1fs"), CountdownDuration);
    
    // 立即广播初始倒计时值
    BroadcastCountdownTick(FMath::CeilToInt(CurrentCountdown));
    
    // 播放开始音效
    if (CountdownSounds.Num() > 0 && CountdownSounds[0])
    {
        PlayCountdownSound(FMath::CeilToInt(CurrentCountdown));
    }
}

void UGamePhaseAbility_Countdown::OnPhaseEnd_Implementation()
{
    Super::OnPhaseEnd_Implementation();
    
    UE_LOG(LogLyra, Log, TEXT("[Countdown] Phase ended"));
    
    // 播放结束音效
    if (CountdownFinishedSound)
    {
        UGameplayStatics::PlaySound2D(this, CountdownFinishedSound);
    }
    
    // 广播倒计时结束事件
    FGameplayEventData EventData;
    EventData.EventTag = FGameplayTag::RequestGameplayTag(TEXT("Event.Countdown.Finished"));
    BroadcastPhaseEvent(EventData.EventTag, EventData);
}

void UGamePhaseAbility_Countdown::OnPhaseTick_Implementation(float DeltaTime)
{
    Super::OnPhaseTick_Implementation(DeltaTime);
    
    // 更新倒计时
    UpdateCountdown(DeltaTime);
}

bool UGamePhaseAbility_Countdown::CanTransitionToNextPhase_Implementation() const
{
    // 倒计时结束后可以转换
    return CurrentCountdown <= 0.0f;
}

void UGamePhaseAbility_Countdown::UpdateCountdown(float DeltaTime)
{
    CurrentCountdown -= DeltaTime;
    
    int32 SecondsRemaining = FMath::CeilToInt(CurrentCountdown);
    
    // 每秒广播一次
    if (SecondsRemaining != LastBroadcastedSecond && SecondsRemaining >= 0)
    {
        LastBroadcastedSecond = SecondsRemaining;
        
        // 广播事件
        BroadcastCountdownTick(SecondsRemaining);
        
        // 播放音效
        PlayCountdownSound(SecondsRemaining);
        
        // 最后一秒震动手柄
        if (SecondsRemaining == 1 && bVibrateOnFinalSecond)
        {
            VibrateAllControllers();
        }
        
        UE_LOG(LogLyra, Verbose, TEXT("[Countdown] %d seconds remaining"), SecondsRemaining);
    }
}

void UGamePhaseAbility_Countdown::BroadcastCountdownTick(int32 SecondsRemaining)
{
    // 创建 Gameplay Event Data
    FGameplayEventData EventData;
    EventData.EventTag = FGameplayTag::RequestGameplayTag(TEXT("Event.Countdown.Tick"));
    EventData.EventMagnitude = SecondsRemaining;
    
    // 发送给所有玩家
    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (GameState)
    {
        for (APlayerState* PS : GameState->PlayerArray)
        {
            if (APawn* Pawn = PS->GetPawn())
            {
                if (IAbilitySystemInterface* ASI = Cast<IAbilitySystemInterface>(Pawn))
                {
                    if (UAbilitySystemComponent* ASC = ASI->GetAbilitySystemComponent())
                    {
                        ASC->HandleGameplayEvent(EventData.EventTag, &EventData);
                    }
                }
            }
        }
    }
}

void UGamePhaseAbility_Countdown::PlayCountdownSound(int32 SecondValue)
{
    USoundBase* SoundToPlay = nullptr;
    
    // 最后一秒使用特殊音效
    if (SecondValue == 1 && FinalCountdownSound)
    {
        SoundToPlay = FinalCountdownSound;
    }
    // 其他秒数使用数组中的音效
    else if (CountdownSounds.IsValidIndex(SecondValue - 1))
    {
        SoundToPlay = CountdownSounds[SecondValue - 1];
    }
    
    if (SoundToPlay)
    {
        UGameplayStatics::PlaySound2D(this, SoundToPlay);
    }
}

void UGamePhaseAbility_Countdown::VibrateAllControllers()
{
    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (!GameState)
    {
        return;
    }
    
    for (APlayerState* PS : GameState->PlayerArray)
    {
        if (APlayerController* PC = Cast<APlayerController>(PS->GetOwner()))
        {
            // 震动参数：强度 0.5，持续 0.2 秒
            PC->ClientPlayForceFeedback(
                nullptr,        // UForceFeedbackEffect (nullptr = 使用默认震动)
                false,          // bLooping
                false,          // bIgnoreTimeDilation
                NAME_None       // Tag
            );
            
            // 或者使用简单震动
            // PC->PlayDynamicForceFeedback(0.5f, 0.2f, true, true, true, true);
        }
    }
}
```

**3. 在编辑器中配置**

创建 Blueprint 子类：`BP_GamePhaseAbility_Countdown`

```
Class Settings:
├─ Parent Class: UGamePhaseAbility_Countdown
└─ Name: BP_GamePhaseAbility_Countdown

Phase Settings:
├─ Phase Tag: GamePhase.Countdown
├─ Phase Name: "倒计时"
├─ Phase Duration: 5.0
└─ Use Auto End Timer: true

Countdown Settings:
├─ Countdown Duration: 5.0
├─ Countdown Sounds:
│   ├─ [0] SFX_Countdown_5
│   ├─ [1] SFX_Countdown_4
│   ├─ [2] SFX_Countdown_3
│   ├─ [3] SFX_Countdown_2
│   └─ [4] SFX_Countdown_1
├─ Final Countdown Sound: SFX_Countdown_Final
├─ Countdown Finished Sound: SFX_Match_Start
└─ Vibrate On Final Second: true

Player Control:
├─ Disable Player Input On Start: true
└─ Freeze Player Movement On Start: true

Phase Effects To Apply:
└─ [0] GE_FreezeMovement
```

#### 3.4.2 Blueprint 创建示例：等待玩家阶段

**1. 创建 Blueprint Class**

```
1. Content Browser → Right Click → Blueprint Class
2. 选择 ULyraGamePhaseAbility
3. 命名: BP_GamePhaseAbility_WaitingForPlayers
```

**2. 配置 Class Defaults**

```
Phase Settings:
├─ Phase Tag: GamePhase.WaitingForPlayers
├─ Phase Name: "等待玩家"
├─ Phase Duration: 0.0 (无限等待)
└─ Use Auto End Timer: false

Phase Tags To Add:
├─ State.WaitingForPlayers
└─ State.CannotStartMatch

Disable Player Input On Start: false
Freeze Player Movement On Start: false
```

**3. 实现 Event Graph**

```
┌──────────────────────────────────────────────────────┐
│ Event OnPhaseBegin                                   │
└────────────────┬─────────────────────────────────────┘
                 │
                 ├─ Print String "等待玩家加入..."
                 │
                 ├─ Start Listening For Players
                 │   └─ 每1秒检查玩家数量
                 │
                 └─ Bind Event to OnPlayerJoined
                     └─ When player joins → Check Min Players

┌──────────────────────────────────────────────────────┐
│ Custom Event: Check Min Players                     │
└────────────────┬─────────────────────────────────────┘
                 │
                 ├─ Get Game State
                 ├─ Get Player Array
                 ├─ Get Array Length
                 │
                 ↓
            ┌────────────┐
            │ Branch     │
            └────┬───────┘
                 │
    ┌────────────┴────────────┐
    │                         │
PlayerCount >= MinPlayers  PlayerCount < MinPlayers
    │                         │
    ↓                         ↓
All Players Ready?        Keep Waiting
    │                         │
    ├─ Yes → Transition       └─ Continue Loop
    └─ No  → Wait

┌──────────────────────────────────────────────────────┐
│ Event OnPhaseEnd                                     │
└────────────────┬─────────────────────────────────────┘
                 │
                 ├─ Stop Listening For Players
                 ├─ Clear Timers
                 └─ Print String "所有玩家准备就绪！"
```

**4. 转换条件实现**

```
┌──────────────────────────────────────────────────────┐
│ Event CanTransitionToNextPhase                       │
└────────────────┬─────────────────────────────────────┘
                 │
                 ├─ Get Game State
                 ├─ Get Player Array Count
                 │
                 ↓
            ┌────────────┐
            │ Branch     │
            └────┬───────┘
                 │
    ┌────────────┴────────────┐
    │                         │
PlayerCount >= 2          PlayerCount < 2
    │                         │
    ↓                         │
Check All Players Ready       │
    │                         │
    ├─ For Each Player:       │
    │   └─ Is Ready?          │
    │                         │
    ↓                         ↓
Return AllReady           Return False
```

### 3.5 Phase Ability 最佳实践

#### 3.5.1 设计原则

**1. 单一职责原则**

```cpp
// ❌ 错误：一个 Phase Ability 做太多事情
class UGamePhaseAbility_GamePlay : public ULyraGamePhaseAbility
{
    void OnPhaseBegin() override
    {
        // 不好：混杂了太多不相关的逻辑
        SpawnWeapons();
        SetupScoring();
        StartMatchTimer();
        ConfigureAI();
        SetupSpectatorCamera();
        InitializeRespawnSystem();
        // ... 几十个功能
    }
};

// ✅ 正确：拆分成多个职责明确的阶段
class UGamePhaseAbility_InProgress : public ULyraGamePhaseAbility
{
    void OnPhaseBegin() override
    {
        // 只负责阶段控制
        EnableGameplay();
        StartMatchTimer();
    }
};

class UGameModeComponent_WeaponSpawner : public UGameFrameworkComponent
{
    void OnPhaseStarted(FGameplayTag PhaseTag)
    {
        // 武器生成由专门的组件负责
        if (PhaseTag == TAG_GamePhase_InProgress)
        {
            SpawnWeapons();
        }
    }
};
```

**2. 数据驱动配置**

```cpp
// ❌ 错误：硬编码配置
class UGamePhaseAbility_Countdown : public ULyraGamePhaseAbility
{
    void OnPhaseBegin() override
    {
        CountdownDuration = 5.0f; // 硬编码
        PlaySound(TEXT("/Game/Sounds/Countdown.wav")); // 硬编码路径
    }
};

// ✅ 正确：可配置的属性
class UGamePhaseAbility_Countdown : public ULyraGamePhaseAbility
{
    UPROPERTY(EditDefaultsOnly, Category = "Countdown")
    float CountdownDuration = 5.0f; // 可在编辑器配置
    
    UPROPERTY(EditDefaultsOnly, Category = "Countdown")
    USoundBase* CountdownSound; // 可在编辑器指定资源
    
    void OnPhaseBegin() override
    {
        if (CountdownSound)
        {
            PlaySound(CountdownSound);
        }
    }
};
```

**3. 事件驱动通信**

```cpp
// ❌ 错误：直接调用 UI
class UGamePhaseAbility_Countdown : public ULyraGamePhaseAbility
{
    void OnPhaseTick(float DeltaTime) override
    {
        // 不要直接操作 UI
        MyHUDWidget->UpdateCountdown(CurrentTime);
    }
};

// ✅ 正确：通过事件通知
class UGamePhaseAbility_Countdown : public ULyraGamePhaseAbility
{
    void OnPhaseTick(float DeltaTime) override
    {
        // 发送事件，UI 自行监听
        FGameplayEventData EventData;
        EventData.EventMagnitude = CurrentTime;
        BroadcastPhaseEvent(TAG_Event_Countdown_Tick, EventData);
    }
};

// UI 监听事件
class UCountdownWidget : public UUserWidget
{
    void NativeConstruct() override
    {
        // 监听倒计时事件
        UAbilitySystemComponent* ASC = GetOwnerASC();
        ASC->GenericGameplayEventCallbacks.FindOrAdd(TAG_Event_Countdown_Tick)
            .AddUObject(this, &UCountdownWidget::OnCountdownTick);
    }
    
    void OnCountdownTick(const FGameplayEventData* EventData)
    {
        float TimeRemaining = EventData->EventMagnitude;
        UpdateCountdownDisplay(TimeRemaining);
    }
};
```

#### 3.5.2 性能优化

**1. 避免每帧 Tick**

```cpp
// ❌ 不好：默认每帧 Tick
class UGamePhaseAbility_InProgress : public ULyraGamePhaseAbility
{
    void OnPhaseTick(float DeltaTime) override
    {
        // 每帧检查，性能浪费
        CheckWinCondition();
    }
};

// ✅ 更好：只在必要时检查
class UGamePhaseAbility_InProgress : public ULyraGamePhaseAbility
{
    void OnPhaseBegin() override
    {
        // 监听击杀事件，只在击杀时检查胜利条件
        UAbilityTask_WaitGameplayEvent* WaitTask = 
            UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
                this, TAG_Event_Player_Killed
            );
        
        WaitTask->EventReceived.AddDynamic(this, &ThisClass::OnPlayerKilled);
        WaitTask->ReadyForActivation();
    }
    
    void OnPlayerKilled(FGameplayEventData Payload)
    {
        // 只在有击杀时检查
        CheckWinCondition();
    }
};
```

**2. 批量处理**

```cpp
// ❌ 不好：逐个处理玩家
void ApplyPhaseEffects()
{
    for (APlayerState* PS : GameState->PlayerArray)
    {
        ApplyEffectToPlayer(PS); // 每个玩家都是独立的 RPC
    }
}

// ✅ 更好：批量应用
void ApplyPhaseEffects()
{
    TArray<AActor*> Players;
    for (APlayerState* PS : GameState->PlayerArray)
    {
        if (APawn* Pawn = PS->GetPawn())
        {
            Players.Add(Pawn);
        }
    }
    
    // 一次性应用到所有玩家
    ApplyGameplayEffectToTargets(GE_PhaseEffect, Players);
}
```

**3. 缓存查询结果**

```cpp
// ❌ 不好：重复查询
void OnPhaseTick(float DeltaTime)
{
    ULyraGamePhaseSubsystem* PhaseSubsystem = GetPhaseSubsystem(); // 每帧查询
    AGameStateBase* GameState = GetWorld()->GetGameState(); // 每帧查询
    
    // ...
}

// ✅ 更好：缓存结果
class UGamePhaseAbility_InProgress : public ULyraGamePhaseAbility
{
    void OnPhaseBegin() override
    {
        // 缓存常用引用
        CachedPhaseSubsystem = GetPhaseSubsystem();
        CachedGameState = GetWorld()->GetGameState();
    }
    
    void OnPhaseTick(float DeltaTime) override
    {
        // 使用缓存
        if (CachedGameState)
        {
            // ...
        }
    }
    
private:
    UPROPERTY()
    ULyraGamePhaseSubsystem* CachedPhaseSubsystem;
    
    UPROPERTY()
    AGameStateBase* CachedGameState;
};
```

#### 3.5.3 错误处理

**1. 优雅的降级**

```cpp
void UGamePhaseAbility_Countdown::OnPhaseBegin_Implementation()
{
    Super::OnPhaseBegin_Implementation();
    
    // 检查必要资源
    if (!CountdownSound)
    {
        UE_LOG(LogLyra, Warning, TEXT("[Countdown] No countdown sound configured"));
        // 继续执行，只是没有音效
    }
    
    // 尝试获取 Game State
    ALyraGameState* GameState = GetWorld()->GetGameState<ALyraGameState>();
    if (!GameState)
    {
        UE_LOG(LogLyra, Error, TEXT("[Countdown] Game State not found!"));
        // 无法继续，提前结束阶段
        EndAbility(...);
        return;
    }
    
    // 继续正常流程...
}
```

**2. 断言关键条件**

```cpp
void UGamePhaseAbility_InProgress::OnPhaseBegin_Implementation()
{
    Super::OnPhaseBegin_Implementation();
    
    // 开发阶段的断言（Release 版本会被编译掉）
    check(PhaseTag.IsValid());
    checkf(PhaseDuration > 0.0f, TEXT("Phase duration must be positive!"));
    
    // 运行时检查
    if (!ensureMsgf(GetWorld(), TEXT("World is null!")))
    {
        return; // 早期退出
    }
}
```

#### 3.5.4 调试技巧

**1. 详细的日志**

```cpp
void UGamePhaseAbility_Countdown::OnPhaseBegin_Implementation()
{
    Super::OnPhaseBegin_Implementation();
    
    UE_LOG(LogLyra, Log, TEXT("=== Countdown Phase Started ==="));
    UE_LOG(LogLyra, Log, TEXT("  Duration: %.1fs"), CountdownDuration);
    UE_LOG(LogLyra, Log, TEXT("  Players: %d"), GetWorld()->GetGameState()->PlayerArray.Num());
    UE_LOG(LogLyra, Log, TEXT("  Network Mode: %s"), 
        *UEnum::GetValueAsString(GetWorld()->GetNetMode()));
}

void UGamePhaseAbility_Countdown::OnPhaseTick_Implementation(float DeltaTime)
{
    UE_LOG(LogLyra, VeryVerbose, TEXT("[Countdown] Tick: %.2fs remaining"), CurrentCountdown);
}
```

**2. 可视化调试**

```cpp
void UGamePhaseAbility_InProgress::OnPhaseTick_Implementation(float DeltaTime)
{
#if !UE_BUILD_SHIPPING
    if (CVarShowPhaseDebug.GetValueOnGameThread())
    {
        // 屏幕上显示阶段信息
        GEngine->AddOnScreenDebugMessage(
            INDEX_NONE,
            0.0f, // 持续时间（0 = 仅当前帧）
            FColor::Yellow,
            FString::Printf(TEXT("Phase: %s | Time: %.1f / %.1f"),
                *PhaseTag.ToString(),
                GetPhaseElapsedTime(),
                PhaseDuration)
        );
        
        // 绘制 Debug 信息
        DrawDebugString(
            GetWorld(),
            FVector(0, 0, 200),
            FString::Printf(TEXT("Active Phase: %s"), *PhaseName.ToString()),
            nullptr,
            FColor::Green,
            0.0f,
            true
        );
    }
#endif
}

// 控制台变量
static TAutoConsoleVariable<bool> CVarShowPhaseDebug(
    TEXT("Lyra.Phase.ShowDebug"),
    false,
    TEXT("Show phase debug information on screen"),
    ECVF_Cheat
);
```

**3. Blueprint Debugging**

在 Blueprint 的 Phase Ability 中：

```
在关键节点添加 "Print String":
├─ OnPhaseBegin: "Countdown Started"
├─ OnPhaseTick: "Time: " + CurrentTime
└─ OnPhaseEnd: "Countdown Finished"

使用 "Breakpoint" 暂停执行:
├─ 在 Branch 节点设置断点
└─ 检查变量值

使用 "Watch" 监视变量:
└─ 右键变量 → "Watch This Value"
```

---

由于篇幅限制，我将继续在下一部分完成剩余的章节。当前已完成：

✅ **1. 游戏阶段系统概述** (完整)
✅ **2. Game Phase Subsystem 深度解析** (完整)
✅ **3. Phase Ability 阶段技能系统** (完整)

接下来将继续完成：
- 4. Phase Tag 管理系统
- 5. 标准游戏阶段实现
- 6-19. 其余所有章节

是否需要我继续完成剩余内容？
