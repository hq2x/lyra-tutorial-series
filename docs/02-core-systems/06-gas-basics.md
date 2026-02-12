# GAS 入门：Gameplay Ability System 基础

## 目录

- [1. GAS 系统概述](#1-gas-系统概述)
  - [1.1 什么是 GAS](#11-什么是-gas)
  - [1.2 为什么需要 GAS](#12-为什么需要-gas)
  - [1.3 GAS 的核心概念](#13-gas-的核心概念)
  - [1.4 GAS 的应用场景](#14-gas-的应用场景)
- [2. GAS 架构深度分析](#2-gas-架构深度分析)
  - [2.1 Ability System Component (ASC)](#21-ability-system-component-asc)
  - [2.2 Gameplay Ability](#22-gameplay-ability)
  - [2.3 Attribute Set](#23-attribute-set)
  - [2.4 Gameplay Effect](#24-gameplay-effect)
  - [2.5 Gameplay Tags](#25-gameplay-tags)
  - [2.6 架构组件关系图](#26-架构组件关系图)
- [3. Lyra 中的 GAS 集成](#3-lyra-中的-gas-集成)
  - [3.1 LyraAbilitySystemComponent](#31-lyraabilitysystemcomponent)
  - [3.2 LyraGameplayAbility](#32-lyragameplayability)
  - [3.3 LyraAttributeSet](#33-lyraattributeset)
  - [3.4 Lyra 的 GAS 初始化流程](#34-lyra-的-gas-初始化流程)
  - [3.5 Experience 系统与 GAS 的整合](#35-experience-系统与-gas-的整合)
- [4. 创建第一个 Gameplay Ability](#4-创建第一个-gameplay-ability)
  - [4.1 使用蓝图创建 Ability](#41-使用蓝图创建-ability)
  - [4.2 使用 C++ 创建 Ability](#42-使用-c-创建-ability)
  - [4.3 配置 Ability 的激活策略](#43-配置-ability-的激活策略)
  - [4.4 授予和激活 Ability](#44-授予和激活-ability)
  - [4.5 调试 Ability](#45-调试-ability)
- [5. Attribute 系统详解](#5-attribute-系统详解)
  - [5.1 什么是 Attribute](#51-什么是-attribute)
  - [5.2 创建自定义 AttributeSet](#52-创建自定义-attributeset)
  - [5.3 Attribute 的复制和网络同步](#53-attribute-的复制和网络同步)
  - [5.4 属性修改器与 Pre/Post AttributeChange](#54-属性修改器与-prepost-attributechange)
  - [5.5 Lyra 中的 AttributeSet 实现](#55-lyra-中的-attributeset-实现)
  - [5.6 属性初始化数据表](#56-属性初始化数据表)
- [6. Gameplay Tags 深度应用](#6-gameplay-tags-深度应用)
  - [6.1 什么是 Gameplay Tags](#61-什么是-gameplay-tags)
  - [6.2 创建和管理 Tags](#62-创建和管理-tags)
  - [6.3 Tag 在 Ability 中的应用](#63-tag-在-ability-中的应用)
  - [6.4 Tag Query 和复杂条件](#64-tag-query-和复杂条件)
  - [6.5 Lyra 中的 Tag 体系](#65-lyra-中的-tag-体系)
  - [6.6 Tag 最佳实践](#66-tag-最佳实践)
- [7. 实战案例：实现简单的技能系统](#7-实战案例实现简单的技能系统)
  - [7.1 需求分析：冲刺技能](#71-需求分析冲刺技能)
  - [7.2 创建 AttributeSet（耐力系统）](#72-创建-attributeset耐力系统)
  - [7.3 实现冲刺 Ability](#73-实现冲刺-ability)
  - [7.4 添加 Gameplay Effect（消耗耐力）](#74-添加-gameplay-effect消耗耐力)
  - [7.5 实现冷却机制](#75-实现冷却机制)
  - [7.6 添加 UI 反馈](#76-添加-ui-反馈)
  - [7.7 网络同步测试](#77-网络同步测试)
- [8. 最佳实践与常见问题](#8-最佳实践与常见问题)
  - [8.1 GAS 最佳实践](#81-gas-最佳实践)
  - [8.2 常见错误与解决方案](#82-常见错误与解决方案)
  - [8.3 性能优化建议](#83-性能优化建议)
  - [8.4 调试技巧](#84-调试技巧)
  - [8.5 推荐学习资源](#85-推荐学习资源)
- [9. 总结与展望](#9-总结与展望)

---

## 1. GAS 系统概述

### 1.1 什么是 GAS

**Gameplay Ability System (GAS)** 是 Unreal Engine 提供的一个强大的插件框架，用于实现游戏中的能力（Abilities）、属性（Attributes）和效果（Effects）系统。它最初是为《堡垒之夜》(Fortnite) 和《帕拉贡》(Paragon) 开发的，后来被 Epic Games 开源并集成到了引擎中。

GAS 提供了一套完整的、可扩展的、网络友好的解决方案，用于处理游戏中常见的需求：

- **能力系统**：技能释放、动作执行、状态管理
- **属性系统**：生命值、魔法值、力量、敏捷等角色属性
- **效果系统**：增益、减益、持续伤害、属性修改
- **标签系统**：状态标识、条件判断、能力控制

GAS 的设计哲学是**数据驱动**和**模块化**，它将复杂的游戏逻辑拆分成可复用的组件，让开发者能够快速构建和迭代游戏玩法。

### 1.2 为什么需要 GAS

在没有 GAS 的情况下，开发者通常会遇到以下问题：

#### 1.2.1 传统能力系统的问题

**问题 1：网络同步困难**
```cpp
// 传统做法：手动管理网络同步
void AMyCharacter::PerformSkill()
{
    if (HasAuthority())
    {
        // 服务器逻辑
        ApplyDamage();
        StartCooldown();
    }
    else
    {
        // 客户端需要调用 RPC
        ServerPerformSkill();
    }
}

// 需要大量的 RPC 和复制变量
UFUNCTION(Server, Reliable)
void ServerPerformSkill();

UPROPERTY(Replicated)
float CooldownRemaining;

UPROPERTY(Replicated)
bool bIsSkillActive;
```

**问题 2：属性管理混乱**
```cpp
// 多个系统修改同一属性，容易出现 Bug
void TakeDamage(float Damage)
{
    Health -= Damage;
    if (Health < 0) Health = 0;
    OnHealthChanged.Broadcast(Health);
}

void ApplyBuff(float HealthBonus)
{
    Health += HealthBonus;
    if (Health > MaxHealth) Health = MaxHealth; // 容易忘记边界检查
}
```

**问题 3：状态管理复杂**
```cpp
// 使用多个布尔变量管理状态
bool bIsStunned;
bool bIsSilenced;
bool bIsInvulnerable;
bool bCanMove;
bool bCanAttack;

// 状态判断逻辑分散各处
if (!bIsStunned && !bIsSilenced && bCanAttack)
{
    PerformAttack();
}
```

#### 1.2.2 GAS 如何解决这些问题

**解决方案 1：自动网络同步**
```cpp
// GAS 自动处理网络同步
UFUNCTION(BlueprintCallable)
bool UMyGameplayAbility::CanActivateAbility()
{
    // GAS 框架会在服务器和客户端之间自动同步
    return Super::CanActivateAbility();
}

void UMyGameplayAbility::ActivateAbility()
{
    // 自动在服务器上执行，无需手动写 RPC
    CommitAbility(); // 自动处理消耗和冷却
}
```

**解决方案 2：统一的属性管理**
```cpp
// 所有属性修改都通过 Gameplay Effect
UPROPERTY(BlueprintReadOnly, Category = "Attributes")
FGameplayAttributeData Health;

// 自动处理边界检查、网络同步、修改器
void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        SetHealth(FMath::Clamp(GetHealth(), 0.0f, GetMaxHealth()));
    }
}
```

**解决方案 3：标签驱动的状态管理**
```cpp
// 使用 Gameplay Tags 管理状态
FGameplayTagContainer ActivationOwnedTags;
ActivationOwnedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Stunned"));

// 清晰的条件判断
if (!HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag("State.Stunned")) &&
    !HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag("State.Silenced")))
{
    TryActivateAbility();
}
```

### 1.3 GAS 的核心概念

GAS 由五个核心组件构成，它们相互协作，形成了一个完整的能力系统：

#### 1.3.1 核心组件一览

| 组件 | 英文名称 | 作用 | 类比 |
|------|----------|------|------|
| **能力系统组件** | Ability System Component (ASC) | GAS 的中枢，管理所有能力和属性 | 游戏角色的"大脑" |
| **游戏能力** | Gameplay Ability | 可执行的动作或技能 | RPG 游戏中的"技能" |
| **属性集** | Attribute Set | 存储和管理角色属性 | 角色的"属性面板" |
| **游戏效果** | Gameplay Effect | 修改属性或添加状态的效果 | Buff/Debuff，伤害/治疗 |
| **游戏标签** | Gameplay Tags | 分层的字符串标识符，用于状态和条件判断 | 技能树中的"节点标签" |

#### 1.3.2 组件关系示意

```
┌─────────────────────────────────────────────────────────┐
│                  Actor (Character)                      │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │      Ability System Component (ASC)               │ │
│  │  ┌─────────────────────────────────────────────┐  │ │
│  │  │         Attribute Sets                      │  │ │
│  │  │  • Health, Mana, Stamina                    │  │ │
│  │  │  • Attack, Defense, Speed                   │  │ │
│  │  └─────────────────────────────────────────────┘  │ │
│  │  ┌─────────────────────────────────────────────┐  │ │
│  │  │         Gameplay Abilities                  │  │ │
│  │  │  • Jump, Sprint, Attack                     │  │ │
│  │  │  • Skills, Spells                           │  │ │
│  │  └─────────────────────────────────────────────┘  │ │
│  │  ┌─────────────────────────────────────────────┐  │ │
│  │  │         Active Gameplay Effects             │  │ │
│  │  │  • Buffs, Debuffs                           │  │ │
│  │  │  • Periodic Damage, Healing                 │  │ │
│  │  └─────────────────────────────────────────────┘  │ │
│  │  ┌─────────────────────────────────────────────┐  │ │
│  │  │         Gameplay Tags                       │  │ │
│  │  │  • State.Stunned, State.Dead                │  │ │
│  │  │  • Ability.Melee.Attack                     │  │ │
│  │  └─────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

#### 1.3.3 数据流示例

让我们看一个完整的数据流示例：**玩家按下技能键，释放一个火球术**

```
1. Input Event (玩家按键)
   ↓
2. Try Activate Ability (尝试激活能力)
   ↓
3. Check Gameplay Tags (检查状态标签)
   - 是否被沉默？(State.Silenced)
   - 是否在冷却？(Cooldown.Skill.Fireball)
   ↓
4. Check Cost (检查消耗)
   - 魔法值是否足够？(Attribute: Mana >= 50)
   ↓
5. Commit Ability (提交能力)
   - 应用消耗 (Gameplay Effect: -50 Mana)
   - 应用冷却 (Gameplay Effect: Add Cooldown Tag)
   ↓
6. Activate Ability (激活能力)
   - 播放动画
   - 生成火球特效
   ↓
7. Apply Gameplay Effect to Target (对目标应用效果)
   - 计算伤害 (Base: 100, Modifier: +AttackPower * 0.5)
   - 修改目标 Health 属性
   ↓
8. End Ability (结束能力)
```

### 1.4 GAS 的应用场景

GAS 非常适合以下类型的游戏：

#### 1.4.1 RPG / MMORPG
- **技能系统**：法师的法术、战士的技能、盗贼的暗影步
- **属性系统**：力量、敏捷、智力、生命值、魔法值
- **装备效果**：武器附魔、护甲加成、饰品特效
- **Buff/Debuff**：祝福、诅咒、中毒、眩晕

#### 1.4.2 MOBA / 竞技游戏
- **英雄技能**：Q/W/E/R 技能、被动技能
- **物品系统**：装备效果、消耗品
- **状态管理**：无敌、隐身、嘲讽、沉默
- **战斗计算**：伤害计算、护甲计算、暴击系统

#### 1.4.3 FPS / TPS 射击游戏
- **武器能力**：开火、换弹、瞄准、切换武器
- **角色技能**：冲刺、滑铲、战术技能、大招
- **装备道具**：护盾、医疗包、手雷
- **状态效果**：流血、灼烧、减速

#### 1.4.4 生存 / 建造游戏
- **生存系统**：饥饿、口渴、体温、疲劳
- **建造能力**：建造、修理、拆除
- **采集制作**：砍树、挖矿、合成
- **环境效果**：寒冷、炎热、潮湿

#### 1.4.5 Lyra 示例项目中的应用

Lyra 是一个多模式的射击游戏示例，它使用 GAS 实现了：

- **射击机制**：开火、换弹、瞄准
- **角色移动**：冲刺、跳跃、蹲伏
- **生命系统**：生命值、护盾、死亡与重生
- **武器系统**：武器切换、弹药管理
- **游戏模式**：团队死斗、控制点、夺旗

在后续章节中，我们将深入研究 Lyra 如何使用 GAS 实现这些功能。

---

## 2. GAS 架构深度分析

### 2.1 Ability System Component (ASC)

**Ability System Component** 是 GAS 的核心，它是一个 Actor Component，负责管理所有与能力系统相关的数据和逻辑。

#### 2.1.1 ASC 的核心职责

```cpp
// AbilitySystemComponent.h (简化版)
UCLASS()
class UAbilitySystemComponent : public UGameplayTasksComponent
{
    GENERATED_BODY()

public:
    // 1. 属性管理
    UPROPERTY()
    TArray<UAttributeSet*> SpawnedAttributes; // 持有的属性集
    
    // 2. 能力管理
    UPROPERTY(ReplicatedUsing=OnRep_ActivatableAbilities)
    FGameplayAbilitySpecContainer ActivatableAbilities; // 已授予的能力
    
    // 3. 效果管理
    UPROPERTY(ReplicatedUsing=OnRep_ActiveGameplayEffects)
    FActiveGameplayEffectsContainer ActiveGameplayEffects; // 活动的效果
    
    // 4. 标签管理
    FGameplayTagCountContainer GameplayTagCountContainer; // 标签计数器
    
    // 5. 核心接口
    void GiveAbility(const FGameplayAbilitySpec& Spec); // 授予能力
    bool TryActivateAbility(FGameplayAbilitySpecHandle Handle); // 激活能力
    FActiveGameplayEffectHandle ApplyGameplayEffectToSelf(...); // 应用效果
    void AddLooseGameplayTag(const FGameplayTag& Tag); // 添加标签
};
```

#### 2.1.2 ASC 的初始化位置

ASC 应该添加到什么 Actor 上？这是一个重要的设计决策：

**选项 1：添加到 PlayerState**（推荐用于多人游戏）

```cpp
// AMyPlayerState.h
UCLASS()
class AMyPlayerState : public APlayerState
{
    GENERATED_BODY()

public:
    AMyPlayerState();
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
    UAbilitySystemComponent* AbilitySystemComponent;
    
    UPROPERTY()
    UMyAttributeSet* AttributeSet;
    
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override
    {
        return AbilitySystemComponent;
    }
};

// AMyPlayerState.cpp
AMyPlayerState::AMyPlayerState()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
    
    AttributeSet = CreateDefaultSubobject<UMyAttributeSet>(TEXT("AttributeSet"));
}
```

**优势**：
- ✅ PlayerState 不会被销毁（即使角色死亡）
- ✅ 支持角色重生（属性和效果保持）
- ✅ 适合多人游戏
- ✅ Lyra 采用的方案

**劣势**：
- ❌ 需要额外的引用管理（Character 需要获取 PlayerState 的 ASC）

**选项 2：添加到 Character**（简单，但不推荐多人游戏）

```cpp
// AMyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
    UAbilitySystemComponent* AbilitySystemComponent;
    
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override
    {
        return AbilitySystemComponent;
    }
};
```

**优势**：
- ✅ 简单直接，容易理解
- ✅ 适合单机游戏或 PvE 游戏

**劣势**：
- ❌ 角色死亡时 ASC 会销毁
- ❌ 不适合需要重生的游戏

#### 2.1.3 ASC 的初始化流程

```cpp
// 在 Character 中初始化 ASC（从 PlayerState 获取）
void AMyCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    
    // 服务器端初始化
    if (AMyPlayerState* PS = GetPlayerState<AMyPlayerState>())
    {
        // 获取 ASC
        AbilitySystemComponent = PS->GetAbilitySystemComponent();
        
        // 初始化 Actor Info（关键步骤！）
        AbilitySystemComponent->InitAbilityActorInfo(PS, this);
        
        // 授予初始能力
        GiveDefaultAbilities();
        
        // 应用初始效果
        ApplyStartupEffects();
    }
}

// 客户端初始化（在 PlayerState 复制后）
void AMyCharacter::OnRep_PlayerState()
{
    Super::OnRep_PlayerState();
    
    // 客户端初始化
    if (AMyPlayerState* PS = GetPlayerState<AMyPlayerState>())
    {
        AbilitySystemComponent = PS->GetAbilitySystemComponent();
        AbilitySystemComponent->InitAbilityActorInfo(PS, this);
    }
}
```

**InitAbilityActorInfo 的重要性**：
- 设置 **Owner Actor**（通常是 PlayerState）
- 设置 **Avatar Actor**（通常是 Character）
- 建立 ASC 与 Actor 的关联
- **必须在使用 ASC 之前调用！**

#### 2.1.4 ASC 的网络复制模式

ASC 支持三种复制模式，根据游戏类型选择：

```cpp
enum class EGameplayEffectReplicationMode : uint8
{
    // 模式 1：最小复制（性能最佳，但客户端信息少）
    Minimal,    // 只复制 Gameplay Effects 给 Owner
                // 适用：单人或 PvP 竞技游戏（只需要知道自己的状态）
    
    // 模式 2：混合复制（推荐）
    Mixed,      // Gameplay Cues 和 Tags 复制给所有人
                // Gameplay Effects 只复制给 Owner
                // 适用：多人游戏，需要看到他人的视觉效果
                // Lyra 使用这个模式
    
    // 模式 3：完全复制（信息最全，性能最差）
    Full        // 所有信息复制给所有人
                // 适用：观战系统、回放系统
};
```

#### 2.1.5 ASC 的常用接口

```cpp
// 能力相关
FGameplayAbilitySpecHandle GiveAbility(const FGameplayAbilitySpec& Spec);
void ClearAbility(const FGameplayAbilitySpecHandle& Handle);
bool TryActivateAbility(FGameplayAbilitySpecHandle Handle);
void CancelAbility(UGameplayAbility* Ability);

// 效果相关
FActiveGameplayEffectHandle ApplyGameplayEffectToSelf(
    const UGameplayEffect* Effect, 
    float Level, 
    FGameplayEffectContextHandle Context);

FActiveGameplayEffectHandle ApplyGameplayEffectToTarget(
    const UGameplayEffect* Effect, 
    UAbilitySystemComponent* Target, 
    float Level, 
    FGameplayEffectContextHandle Context);

bool RemoveActiveGameplayEffect(FActiveGameplayEffectHandle Handle);

// 标签相关
void AddLooseGameplayTag(const FGameplayTag& Tag);
void RemoveLooseGameplayTag(const FGameplayTag& Tag);
bool HasMatchingGameplayTag(const FGameplayTag& Tag) const;
void GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const;

// 属性相关
float GetNumericAttribute(const FGameplayAttribute& Attribute) const;
void SetNumericAttributeBase(const FGameplayAttribute& Attribute, float NewValue);
```

### 2.2 Gameplay Ability

**Gameplay Ability** 代表一个可执行的动作或技能。它是一个 UObject，不是 Actor Component。

#### 2.2.1 Ability 的生命周期

```cpp
// UGameplayAbility.h (简化版)
UCLASS()
class UGameplayAbility : public UObject
{
    GENERATED_BODY()

public:
    // 1. 激活检查
    virtual bool CanActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayTagContainer* SourceTags,
        const FGameplayTagContainer* TargetTags,
        FGameplayTagContainer* OptionalRelevantTags) const;
    
    // 2. 消耗检查（魔法值、耐力等）
    virtual bool CheckCost(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        OUT FGameplayTagContainer* OptionalRelevantTags) const;
    
    // 3. 提交能力（应用消耗和冷却）
    virtual bool CommitAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        OUT FGameplayTagContainer* OptionalRelevantTags);
    
    // 4. 激活能力（主要逻辑）
    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData);
    
    // 5. 结束能力
    virtual void EndAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility,
        bool bWasCancelled);
    
    // 6. 取消能力
    virtual void CancelAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateCancelAbility);
};
```

#### 2.2.2 Ability 的执行流程

```
┌────────────────────────────────────────────────┐
│ 1. TryActivateAbility()                        │
│    ↓                                            │
│ 2. CanActivateAbility()                        │
│    • 检查 Cooldown Tags                         │
│    • 检查 Blocking Tags                         │
│    • 检查 Required Tags                         │
│    ↓                                            │
│ 3. CheckCost()                                 │
│    • 检查魔法值、耐力等资源                        │
│    ↓                                            │
│ 4. CommitAbility()                             │
│    • CommitCost() - 扣除资源                    │
│    • CommitCooldown() - 应用冷却                │
│    ↓                                            │
│ 5. ActivateAbility()                           │
│    • 执行技能逻辑                                │
│    • 播放动画、特效                              │
│    • 应用 Gameplay Effects                     │
│    ↓                                            │
│ 6. EndAbility() 或 CancelAbility()            │
│    • 清理状态                                    │
│    • 移除临时效果                                │
└────────────────────────────────────────────────┘
```

#### 2.2.3 Ability 的重要属性

```cpp
UCLASS()
class UMyGameplayAbility : public UGameplayAbility
{
    GENERATED_BODY()

public:
    // 能力标签（定义这个能力是什么）
    UPROPERTY(EditDefaultsOnly, Category = "Tags")
    FGameplayTagContainer AbilityTags;
    
    // 激活时拥有的标签（例如：Ability.Active）
    UPROPERTY(EditDefaultsOnly, Category = "Tags")
    FGameplayTagContainer ActivationOwnedTags;
    
    // 阻止此能力激活的标签（例如：State.Stunned）
    UPROPERTY(EditDefaultsOnly, Category = "Tags")
    FGameplayTagContainer ActivationBlockedTags;
    
    // 激活此能力需要的标签
    UPROPERTY(EditDefaultsOnly, Category = "Tags")
    FGameplayTagContainer ActivationRequiredTags;
    
    // 冷却效果（Gameplay Effect Class）
    UPROPERTY(EditDefaultsOnly, Category = "Cooldown")
    TSubclassOf<UGameplayEffect> CooldownGameplayEffectClass;
    
    // 消耗效果（例如：-50 魔法值）
    UPROPERTY(EditDefaultsOnly, Category = "Cost")
    TSubclassOf<UGameplayEffect> CostGameplayEffectClass;
    
    // 网络执行策略
    UPROPERTY(EditDefaultsOnly, Category = "Advanced")
    EGameplayAbilityNetExecutionPolicy NetExecutionPolicy;
    
    // 网络安全策略
    UPROPERTY(EditDefaultsOnly, Category = "Advanced")
    EGameplayAbilityNetSecurityPolicy NetSecurityPolicy;
    
    // 实例化策略
    UPROPERTY(EditDefaultsOnly, Category = "Advanced")
    EGameplayAbilityInstancingPolicy InstancingPolicy;
};
```

#### 2.2.4 Ability 的网络执行策略

GAS 提供了灵活的网络执行策略：

```cpp
enum class EGameplayAbilityNetExecutionPolicy : uint8
{
    // 1. 本地预测（最常用）
    LocalPredicted,    // 客户端预测，服务器校验
                       // 适用：大部分需要快速响应的能力
                       // 例如：跳跃、冲刺、射击
    
    // 2. 只在本地执行
    LocalOnly,         // 只在客户端执行，不发送到服务器
                       // 适用：纯视觉效果，不影响游戏状态
                       // 例如：UI 动画、本地特效
    
    // 3. 只在服务器执行
    ServerOnly,        // 只在服务器执行
                       // 适用：敏感操作，防作弊
                       // 例如：商城购买、GM 命令
    
    // 4. 服务器发起
    ServerInitiated    // 由服务器发起，客户端不能主动激活
                       // 适用：AI 能力、服务器触发的事件
};
```

#### 2.2.5 Ability 的实例化策略

```cpp
enum class EGameplayAbilityInstancingPolicy : uint8
{
    // 1. 每次激活时实例化（推荐）
    InstancedPerExecution,  // 每次激活创建新实例
                            // 优势：能力状态独立，支持多次同时激活
                            // 适用：大部分能力
    
    // 2. 每个 Actor 一个实例
    InstancedPerActor,      // 每个角色只有一个实例
                            // 优势：可以保存状态
                            // 适用：需要持久状态的能力
    
    // 3. 不实例化（性能最佳）
    NonInstanced            // 使用 CDO，所有 Actor 共享
                            // 优势：性能最佳
                            // 劣势：不能保存状态
                            // 适用：简单的、无状态的能力
};
```

### 2.3 Attribute Set

**Attribute Set** 用于存储和管理游戏属性（如生命值、魔法值等）。它是一个 UObject，持有一组 `FGameplayAttributeData`。

#### 2.3.1 AttributeSet 的基本结构

```cpp
// MyAttributeSet.h
UCLASS()
class UMyAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    UMyAttributeSet();
    
    // 网络复制
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    
    // 属性变化回调
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;
    
    // 生命值属性
    UPROPERTY(BlueprintReadOnly, Category = "Attributes", ReplicatedUsing = OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)
    
    // 最大生命值属性
    UPROPERTY(BlueprintReadOnly, Category = "Attributes", ReplicatedUsing = OnRep_MaxHealth)
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth)
    
protected:
    // 网络复制回调
    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);
    
    UFUNCTION()
    virtual void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);
};
```

#### 2.3.2 ATTRIBUTE_ACCESSORS 宏

这个宏会生成一组便捷的访问器方法：

```cpp
// ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health) 会生成：

// 获取属性
static FGameplayAttribute GetHealthAttribute()
{
    static FProperty* Property = FindFieldChecked<FProperty>(
        UMyAttributeSet::StaticClass(), 
        GET_MEMBER_NAME_CHECKED(UMyAttributeSet, Health));
    return FGameplayAttribute(Property);
}

// 获取当前值
float GetHealth() const
{
    return Health.GetCurrentValue();
}

// 设置基础值
void SetHealth(float NewVal)
{
    Health.SetBaseValue(NewVal);
    Health.SetCurrentValue(NewVal);
}

// 初始化
void InitHealth(float NewVal)
{
    Health.SetBaseValue(NewVal);
    Health.SetCurrentValue(NewVal);
}
```

#### 2.3.3 AttributeSet 实现示例

```cpp
// MyAttributeSet.cpp
#include "MyAttributeSet.h"
#include "Net/UnrealNetwork.h"
#include "GameplayEffectExtension.h"

UMyAttributeSet::UMyAttributeSet()
{
    // 初始化默认值
    InitHealth(100.0f);
    InitMaxHealth(100.0f);
}

void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    
    // 注册需要复制的属性
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
}

void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
    // 通知 ASC 属性已复制
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldHealth);
}

void UMyAttributeSet::OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, MaxHealth, OldMaxHealth);
}

void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);
    
    // 在属性改变前进行限制（仅用于即时效果，不会改变 BaseValue）
    if (Attribute == GetHealthAttribute())
    {
        // 限制生命值范围
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
    }
}

void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);
    
    // 在 Gameplay Effect 执行后进行最终调整
    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        // 限制生命值范围
        SetHealth(FMath::Clamp(GetHealth(), 0.0f, GetMaxHealth()));
        
        // 检查死亡
        if (GetHealth() <= 0.0f)
        {
            // 触发死亡逻辑（通过 Gameplay Event 或直接调用）
            if (AActor* OwnerActor = GetOwningActor())
            {
                // 可以发送 Gameplay Event
                FGameplayEventData EventData;
                EventData.Instigator = Data.EffectSpec.GetEffectContext().GetInstigator();
                EventData.Target = OwnerActor;
                
                // 发送死亡事件
                UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(
                    OwnerActor, 
                    FGameplayTag::RequestGameplayTag("Event.Death"), 
                    EventData);
            }
        }
    }
}
```

#### 2.3.4 Base Value vs Current Value

每个 `FGameplayAttributeData` 包含两个值：

```cpp
struct FGameplayAttributeData
{
    // 基础值：永久修改（如升级、装备）
    float BaseValue;
    
    // 当前值：基础值 + 临时修改（如 Buff）
    float CurrentValue;
    
    // CurrentValue = BaseValue + (Additive Modifiers) * (Multiplicative Modifiers)
};
```

**示例**：

```
初始状态：
BaseValue = 100
CurrentValue = 100

应用 +20 临时加成（Buff）：
BaseValue = 100
CurrentValue = 120

Buff 结束：
BaseValue = 100
CurrentValue = 100

永久升级 +50：
BaseValue = 150
CurrentValue = 150
```

### 2.4 Gameplay Effect

**Gameplay Effect (GE)** 是修改属性、添加标签、应用状态的核心机制。它是一个 **Data Asset**，不是代码类。

#### 2.4.1 Gameplay Effect 的类型

GAS 支持三种持续时间类型的效果：

```cpp
enum class EGameplayEffectDurationType : uint8
{
    // 1. 瞬时效果（Instant）
    Instant,        // 立即执行，不会保存在 ASC 中
                    // 用途：伤害、治疗、立即属性修改
                    // 例如：火球术造成 100 点伤害
    
    // 2. 持续效果（Duration）
    Duration,       // 持续一段时间，然后自动移除
                    // 用途：临时 Buff/Debuff
                    // 例如：加速 Buff 持续 10 秒
    
    // 3. 无限效果（Infinite）
    Infinite,       // 永久存在，直到手动移除
                    // 用途：装备加成、被动效果
                    // 例如：装备武器增加 50 攻击力
    
    // 4. 有无限期但有周期性执行（Infinite + Period）
    // 例如：持续燃烧，每秒造成 10 点伤害
};
```

#### 2.4.2 Gameplay Effect 的核心配置

创建一个 Gameplay Effect 的蓝图资产：

**1. Modifiers（修改器）**：修改属性的值

```
Modifier 1:
  - Attribute: Health
  - Operation: Add (加法)
  - Magnitude: -50.0 (造成 50 点伤害)

Modifier 2:
  - Attribute: Health
  - Operation: Multiply (乘法)
  - Magnitude: 1.5 (增加 50% 生命值)

Modifier 3:
  - Attribute: AttackPower
  - Operation: Override (覆盖)
  - Magnitude: 100.0 (直接设为 100)
```

**2. Granted Tags（授予标签）**：效果存在时添加的标签

```
Granted Tags:
  - State.Buffed
  - Effect.SpeedBoost

用途：其他系统可以检测这些标签
```

**3. Application Tag Requirements（应用条件）**：目标必须满足的条件

```
Required Tags: State.Alive (必须存活)
Ignored Tags: State.Immune (免疫状态下不应用)
Blocked Tags: State.Invulnerable (无敌时阻止应用)
```

**4. Ongoing Tag Requirements（持续条件）**：效果持续需要的条件

```
Ongoing Required Tags: State.Alive
如果目标死亡（失去 State.Alive），效果自动移除
```

**5. Removal Tag Requirements（移除条件）**：拥有这些标签时效果会被移除

```
Remove with Tags: Effect.Cleanse (净化效果会移除)
```

#### 2.4.3 Magnitude Calculation（幅度计算）

Gameplay Effect 的 Modifier 可以使用多种方式计算幅度：

```cpp
enum class EGameplayEffectMagnitudeCalculation : uint8
{
    // 1. 标量浮点数（最简单）
    ScalableFloat,       // 直接指定数值
                         // 例如：-50.0（固定伤害）
    
    // 2. 基于属性（读取属性值）
    AttributeBased,      // 基于 Source 或 Target 的属性
                         // 例如：Source.AttackPower * 1.5（基于攻击力的伤害）
    
    // 3. 自定义计算（代码）
    CustomCalculationClass,  // 使用 C++ 类进行复杂计算
                             // 例如：考虑护甲、暴击、随机性的伤害计算
    
    // 4. Set By Caller（运行时设置）
    SetByCaller          // 激活时动态指定
                         // 例如：武器伤害根据武器类型动态决定
};
```

**示例：基于属性的伤害计算**

```
Modifier:
  - Attribute: Health
  - Operation: Add
  - Magnitude Calculation: Attribute Based
    - Backing Attribute: Source.AttackPower (使用攻击者的攻击力)
    - Coefficient: 1.5 (系数)
    - Pre Multiply Additive Value: 0
    - Post Multiply Additive Value: 10 (基础伤害)

最终伤害 = (AttackPower * 1.5) + 10
```

#### 2.4.4 Periodic Effects（周期性效果）

持续伤害/治疗效果的实现：

```
Duration: Infinite (无限持续)
Period: 1.0 seconds (每秒执行一次)
Execute on Application: true (立即执行一次)

Modifier:
  - Attribute: Health
  - Operation: Add
  - Magnitude: -10.0 (每秒 10 点伤害)
  
Granted Tags: State.Burning (燃烧状态)

示例：中毒效果，每秒扣 10 点生命，持续 5 秒
Duration: 5.0 seconds (持续 5 秒)
Period: 1.0 seconds
```

#### 2.4.5 Stacking（堆叠）

控制同一效果多次应用时的行为：

```cpp
enum class EGameplayEffectStackingType : uint8
{
    // 1. 不堆叠
    None,               // 已有效果时，新效果不生效
                        // 例如：加速 Buff 不能叠加
    
    // 2. 聚合（更新持续时间）
    AggregateBySource,  // 来自同一源的效果刷新持续时间
                        // 例如：同一个玩家的点燃效果刷新
    
    // 3. 聚合（不更新持续时间）
    AggregateByTarget,  // 来自不同源的效果独立计算
                        // 例如：多个玩家的点燃效果独立
};

// 堆叠配置
Stacking Type: Aggregate By Source
Stack Limit Count: 5 (最多 5 层)
Stack Duration Refresh Policy: Refresh on Successful Application (每次应用刷新)
Stack Period Reset Policy: Reset on Successful Application (重置周期)
Stack Expiration Policy: Clear Entire Stack (全部移除)

// 堆叠效果计算
Overflow Application: Deny on Overflow (超过上限时拒绝)
Stacking Magnitude: Stack on Source (基于源堆叠)
```

### 2.5 Gameplay Tags

**Gameplay Tags** 是 GAS 的标签系统，用于标识状态、分类能力、控制逻辑。

#### 2.5.1 什么是 Gameplay Tags

Gameplay Tags 是**分层的字符串标识符**，使用点号分隔：

```
Ability.Melee.Attack
Ability.Melee.HeavyAttack
Ability.Range.Shoot
Ability.Range.Snipe

State.Dead
State.Stunned
State.Silenced
State.Invulnerable

Cooldown.Skill.Fireball
Cooldown.Skill.IceBlast

Effect.Buff.Speed
Effect.Debuff.Slow
```

**层级结构的优势**：

```cpp
// 可以匹配父级标签
HasMatchingGameplayTag("Ability.Melee") 
// 会匹配：
// - Ability.Melee.Attack
// - Ability.Melee.HeavyAttack

// 可以查询多个标签
FGameplayTagContainer Tags;
Tags.AddTag("State.Stunned");
Tags.AddTag("State.Silenced");

if (HasAnyMatchingGameplayTags(Tags))
{
    // 角色被控制了
}
```

#### 2.5.2 创建 Gameplay Tags

**方法 1：通过 INI 文件**（推荐）

在 `Config/DefaultGameplayTags.ini` 中定义：

```ini
[/Script/GameplayTags.GameplayTagsList]
+GameplayTagList=(Tag="Ability.Melee.Attack",DevComment="近战攻击能力")
+GameplayTagList=(Tag="Ability.Melee.HeavyAttack",DevComment="重击能力")
+GameplayTagList=(Tag="State.Stunned",DevComment="眩晕状态")
+GameplayTagList=(Tag="State.Dead",DevComment="死亡状态")
+GameplayTagList=(Tag="Cooldown.Skill.Fireball",DevComment="火球术冷却")
```

**方法 2：在编辑器中创建**

1. 打开 **Project Settings**
2. 找到 **Gameplay Tags** 部分
3. 点击 **Add New Gameplay Tag**
4. 输入标签名称（如 `Ability.Melee.Attack`）
5. 添加描述

**方法 3：在代码中注册（运行时）**

```cpp
// 在 GameInstance 或 GameMode 中注册
void UMyGameInstance::Init()
{
    Super::Init();
    
    // 添加 Native Tag（本地标签）
    UGameplayTagsManager& TagsManager = UGameplayTagsManager::Get();
    
    // 注意：应该使用 INI 文件，这里仅作演示
    FGameplayTag MeleeAttackTag = TagsManager.AddNativeGameplayTag(
        FName("Ability.Melee.Attack"),
        FString("近战攻击能力"));
}
```

#### 2.5.3 在代码中使用 Tags

```cpp
// 方法 1：直接请求 Tag（会在 TagsManager 中查找）
FGameplayTag StunnedTag = FGameplayTag::RequestGameplayTag("State.Stunned");

// 方法 2：使用静态变量（性能更好）
// 在头文件中定义
namespace GameplayTags
{
    extern FGameplayTag State_Stunned;
    extern FGameplayTag State_Dead;
    extern FGameplayTag Ability_Melee_Attack;
}

// 在 CPP 中定义（通常在 GameplayTagsManager.cpp）
namespace GameplayTags
{
    FGameplayTag State_Stunned;
    FGameplayTag State_Dead;
    FGameplayTag Ability_Melee_Attack;
}

// 在初始化时获取
void InitializeGameplayTags()
{
    GameplayTags::State_Stunned = FGameplayTag::RequestGameplayTag("State.Stunned");
    GameplayTags::State_Dead = FGameplayTag::RequestGameplayTag("State.Dead");
    GameplayTags::Ability_Melee_Attack = FGameplayTag::RequestGameplayTag("Ability.Melee.Attack");
}

// 使用
if (AbilitySystemComponent->HasMatchingGameplayTag(GameplayTags::State_Stunned))
{
    // 角色被眩晕，不能行动
}
```

#### 2.5.4 Tag Container 和 Query

**FGameplayTagContainer**：标签集合

```cpp
FGameplayTagContainer Tags;

// 添加标签
Tags.AddTag(GameplayTags::State_Stunned);
Tags.AddTag(GameplayTags::State_Silenced);

// 检查是否包含
if (Tags.HasTag(GameplayTags::State_Stunned))
{
    // 包含眩晕标签
}

// 检查是否包含任意一个
FGameplayTagContainer CheckTags;
CheckTags.AddTag(GameplayTags::State_Stunned);
CheckTags.AddTag(GameplayTags::State_Dead);

if (Tags.HasAny(CheckTags))
{
    // 包含眩晕或死亡
}

// 检查是否包含所有
if (Tags.HasAll(CheckTags))
{
    // 同时包含眩晕和死亡
}
```

**FGameplayTagQuery**：复杂条件查询

```cpp
// 创建复杂查询：(A AND B) OR (C AND NOT D)
FGameplayTagQuery Query = FGameplayTagQuery::MakeQuery_MatchAnyTags(
    FGameplayTagContainer::CreateFromArray({
        GameplayTags::Ability_Active,
        GameplayTags::Ability_OnCooldown
    }));

if (Query.Matches(AbilitySystemComponent->GetOwnedGameplayTags()))
{
    // 查询匹配
}
```

### 2.6 架构组件关系图

让我们总结一下 GAS 各组件的关系：

```
┌──────────────────────────────────────────────────────────────┐
│                         Actor (Character)                     │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         Ability System Component (ASC)                 │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────────────────┐ │  │
│  │  │  Attribute Sets                                  │ │  │
│  │  │  ┌────────────────────────────────────────────┐  │ │  │
│  │  │  │ Health: 80/100                             │  │ │  │
│  │  │  │ Mana: 50/100                               │  │ │  │
│  │  │  │ Stamina: 30/100                            │  │  │  │
│  │  │  │ AttackPower: 50                            │  │  │  │
│  │  │  └────────────────────────────────────────────┘  │ │  │
│  │  └──────────────────────────────────────────────────┘ │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────────────────┐ │  │
│  │  │  Granted Abilities                               │ │  │
│  │  │  ┌────────────────────────────────────────────┐  │ │  │
│  │  │  │ [1] Jump (Can Activate)                    │  │ │  │
│  │  │  │ [2] Sprint (Active)                        │  │ │  │
│  │  │  │ [3] Attack (On Cooldown)                   │  │ │  │
│  │  │  │ [4] Fireball (Insufficient Mana)           │  │ │  │
│  │  │  └────────────────────────────────────────────┘  │ │  │
│  │  └──────────────────────────────────────────────────┘ │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────────────────┐ │  │
│  │  │  Active Gameplay Effects                         │ │  │
│  │  │  ┌────────────────────────────────────────────┐  │ │  │
│  │  │  │ [1] Speed Boost (+50% Speed, 5s left)      │  │ │  │
│  │  │  │ [2] Burning (-10 HP/s, 3s left)            │  │ │  │
│  │  │  │ [3] Weapon Buff (+20 Attack, Infinite)     │  │ │  │
│  │  │  │ [4] Cooldown Tag (Cooldown.Attack, 2s)     │  │ │  │
│  │  │  └────────────────────────────────────────────┘  │ │  │
│  │  └──────────────────────────────────────────────────┘ │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────────────────┐ │  │
│  │  │  Owned Gameplay Tags                             │ │  │
│  │  │  ┌────────────────────────────────────────────┐  │ │  │
│  │  │  │ • State.Alive                              │  │ │  │
│  │  │  │ • State.Moving                             │  │ │  │
│  │  │  │ • Effect.Buff.Speed                        │  │ │  │
│  │  │  │ • Effect.Debuff.Burning                    │  │ │  │
│  │  │  │ • Cooldown.Ability.Attack                  │  │ │  │
│  │  │  └────────────────────────────────────────────┘  │ │  │
│  │  └──────────────────────────────────────────────────┘ │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘

数据流示例：
1. 玩家按下 "Sprint" 键
2. ASC.TryActivateAbility(SprintAbility)
3. SprintAbility.CanActivateAbility()
   - 检查 Tag: 没有 State.Stunned ✓
   - 检查 Cost: Stamina >= 10 ✓
4. SprintAbility.CommitAbility()
   - 应用消耗 GE: -10 Stamina
   - 应用冷却 GE: Add Cooldown.Sprint Tag
5. SprintAbility.ActivateAbility()
   - 应用加速 GE: +50% MovementSpeed
   - 添加 Tag: State.Sprinting
6. (每帧) Stamina 持续消耗
7. (Stamina 耗尽或玩家松开键) EndAbility()
   - 移除加速 GE
   - 移除 State.Sprinting Tag
```

---

## 3. Lyra 中的 GAS 集成

Lyra 项目对 GAS 进行了深度定制和扩展，形成了一套完整的能力系统架构。本节将详细分析 Lyra 是如何使用 GAS 的。

### 3.1 LyraAbilitySystemComponent

Lyra 扩展了原生的 `UAbilitySystemComponent`，添加了许多项目特定的功能。

#### 3.1.1 类定义

```cpp
// LyraAbilitySystemComponent.h
UCLASS()
class LYRAGAME_API ULyraAbilitySystemComponent : public UAbilitySystemComponent
{
    GENERATED_BODY()

public:
    ULyraAbilitySystemComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    // 初始化
    void InitAbilityActorInfo(AActor* InOwnerActor, AActor* InAvatarActor) override;

    // 能力输入绑定
    void AbilityInputTagPressed(const FGameplayTag& InputTag);
    void AbilityInputTagReleased(const FGameplayTag& InputTag);
    void ProcessAbilityInput(float DeltaTime, bool bGamePaused);
    void ClearAbilityInput();

    // 能力授予和移除
    void GrantAbility(const FLyraAbilitySet_GrantedHandles& GrantedHandles);
    void RemoveAbility(const FLyraAbilitySet_GrantedHandles& GrantedHandles);

    // 动画通知（与 Ability 配合）
    void AbilitySpecInputPressed(FGameplayAbilitySpec& Spec) override;
    void AbilitySpecInputReleased(FGameplayAbilitySpec& Spec) override;

protected:
    // 输入按下/释放的缓存
    TArray<FGameplayAbilitySpecHandle> InputPressedSpecHandles;
    TArray<FGameplayAbilitySpecHandle> InputReleasedSpecHandles;
    TArray<FGameplayAbilitySpecHandle> InputHeldSpecHandles;

    // 能力激活失败的回调
    virtual void NotifyAbilityFailed(const FGameplayAbilitySpecHandle Handle, 
                                      UGameplayAbility* Ability, 
                                      const FGameplayTagContainer& FailureReason) override;
};
```

#### 3.1.2 输入处理系统

Lyra 使用 **Gameplay Tag 驱动的输入系统**，不使用传统的输入 ID。

```cpp
// 输入按下时调用
void ULyraAbilitySystemComponent::AbilityInputTagPressed(const FGameplayTag& InputTag)
{
    if (InputTag.IsValid())
    {
        // 遍历所有已授予的能力
        for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
        {
            // 检查能力是否绑定到这个输入标签
            if (AbilitySpec.Ability && (AbilitySpec.DynamicAbilityTags.HasTagExact(InputTag)))
            {
                // 缓存这个能力，稍后在 ProcessAbilityInput 中处理
                InputPressedSpecHandles.AddUnique(AbilitySpec.Handle);
                InputHeldSpecHandles.AddUnique(AbilitySpec.Handle);
            }
        }
    }
}

// 输入释放时调用
void ULyraAbilitySystemComponent::AbilityInputTagReleased(const FGameplayTag& InputTag)
{
    if (InputTag.IsValid())
    {
        for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
        {
            if (AbilitySpec.Ability && (AbilitySpec.DynamicAbilityTags.HasTagExact(InputTag)))
            {
                InputReleasedSpecHandles.AddUnique(AbilitySpec.Handle);
                InputHeldSpecHandles.Remove(AbilitySpec.Handle);
            }
        }
    }
}

// 每帧调用，处理输入
void ULyraAbilitySystemComponent::ProcessAbilityInput(float DeltaTime, bool bGamePaused)
{
    // 处理输入释放
    for (const FGameplayAbilitySpecHandle& SpecHandle : InputReleasedSpecHandles)
    {
        if (FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(SpecHandle))
        {
            AbilitySpecInputReleased(*AbilitySpec);
        }
    }

    // 处理输入按下
    for (const FGameplayAbilitySpecHandle& SpecHandle : InputPressedSpecHandles)
    {
        if (FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(SpecHandle))
        {
            AbilitySpecInputPressed(*AbilitySpec);

            // 如果能力还没激活，尝试激活
            if (!AbilitySpec->IsActive())
            {
                TryActivateAbility(SpecHandle);
            }
        }
    }

    // 处理持续按住
    for (const FGameplayAbilitySpecHandle& SpecHandle : InputHeldSpecHandles)
    {
        if (FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(SpecHandle))
        {
            // 某些能力需要在按住期间持续激活（如自动射击）
            if (AbilitySpec->IsActive())
            {
                // 通知能力输入正在被持续按住
                // 能力可以在 InputPressed() 回调中处理
            }
        }
    }

    // 清空本帧的输入缓存
    InputPressedSpecHandles.Reset();
    InputReleasedSpecHandles.Reset();
}
```

**输入流程示意图**：

```
1. 玩家按下 "Jump" 键
   ↓
2. Enhanced Input 系统触发
   ↓
3. ULyraHeroComponent::Input_Jump()
   ↓
4. ASC->AbilityInputTagPressed(InputTag_Jump)
   ↓
5. 缓存所有绑定到 InputTag_Jump 的能力
   ↓
6. ProcessAbilityInput() (每帧)
   ↓
7. TryActivateAbility(JumpAbility)
   ↓
8. JumpAbility->ActivateAbility()
```

#### 3.1.3 AbilitySet 系统

Lyra 使用 **AbilitySet** 来批量授予能力、效果和属性。

```cpp
// LyraAbilitySet.h
USTRUCT(BlueprintType)
struct FLyraAbilitySet_GameplayAbility
{
    GENERATED_BODY()

    // 要授予的能力类
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<ULyraGameplayAbility> Ability = nullptr;

    // 能力等级
    UPROPERTY(EditDefaultsOnly)
    int32 AbilityLevel = 1;

    // 输入标签（用于绑定输入）
    UPROPERTY(EditDefaultsOnly, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};

USTRUCT(BlueprintType)
struct FLyraAbilitySet_GameplayEffect
{
    GENERATED_BODY()

    // 要应用的效果类
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UGameplayEffect> GameplayEffect = nullptr;

    // 效果等级
    UPROPERTY(EditDefaultsOnly)
    float EffectLevel = 1.0f;
};

USTRUCT(BlueprintType)
struct FLyraAbilitySet_AttributeSet
{
    GENERATED_BODY()

    // 要添加的属性集类
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UAttributeSet> AttributeSet;
};

// AbilitySet 数据资产
UCLASS(BlueprintType, Const)
class ULyraAbilitySet : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 授予能力的列表
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Abilities")
    TArray<FLyraAbilitySet_GameplayAbility> GrantedGameplayAbilities;

    // 应用效果的列表
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Effects")
    TArray<FLyraAbilitySet_GameplayEffect> GrantedGameplayEffects;

    // 添加属性集的列表
    UPROPERTY(EditDefaultsOnly, Category = "Attribute Sets")
    TArray<FLyraAbilitySet_AttributeSet> GrantedAttributes;

    // 授予这个 AbilitySet 到 ASC
    void GiveToAbilitySystem(ULyraAbilitySystemComponent* ASC, 
                             FLyraAbilitySet_GrantedHandles* OutGrantedHandles, 
                             UObject* SourceObject = nullptr) const;
};
```

**AbilitySet 的使用示例**：

```cpp
// 在 Pawn Equipped 后授予能力
void ULyraEquipmentInstance::OnEquipped()
{
    if (ULyraAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        // 授予武器的 AbilitySet（包含射击、换弹等能力）
        if (WeaponAbilitySet)
        {
            WeaponAbilitySet->GiveToAbilitySystem(ASC, &GrantedHandles, this);
        }
    }
}

// 在 Pawn Unequipped 后移除能力
void ULyraEquipmentInstance::OnUnequipped()
{
    if (ULyraAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        // 移除授予的能力
        GrantedHandles.TakeFromAbilitySystem(ASC);
    }
}
```

### 3.2 LyraGameplayAbility

Lyra 的基础 Ability 类，添加了许多实用功能。

#### 3.2.1 类定义

```cpp
// LyraGameplayAbility.h
UCLASS()
class LYRAGAME_API ULyraGameplayAbility : public UGameplayAbility
{
    GENERATED_BODY()

public:
    ULyraGameplayAbility(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    // 获取 Lyra ASC
    UFUNCTION(BlueprintCallable, Category = "Lyra|Ability")
    ULyraAbilitySystemComponent* GetLyraAbilitySystemComponentFromActorInfo() const;

    // 获取控制器
    UFUNCTION(BlueprintCallable, Category = "Lyra|Ability")
    ALyraPlayerController* GetLyraPlayerControllerFromActorInfo() const;

    // 获取角色
    UFUNCTION(BlueprintCallable, Category = "Lyra|Ability")
    ALyraCharacter* GetLyraCharacterFromActorInfo() const;

    // 激活策略（控制能力如何激活）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Ability Activation")
    ELyraAbilityActivationPolicy ActivationPolicy;

    // 相机模式（能力激活时切换相机）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Camera")
    TSubclassOf<ULyraCameraMode> CameraMode;

protected:
    // 能力激活时调用
    virtual void OnGiveAbility(const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilitySpec& Spec) override;

    // 能力结束时调用
    virtual void OnRemoveAbility(const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilitySpec& Spec) override;

    // 激活失败通知
    virtual void NativeOnAbilityFailedToActivate(const FGameplayTagContainer& FailedReason) const;

    // 在 EndAbility 之前调用
    virtual void OnEndAbility(const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilitySpec& Spec);
};

// 激活策略枚举
UENUM(BlueprintType)
enum class ELyraAbilityActivationPolicy : uint8
{
    // 输入触发（按键）
    OnInputTriggered,

    // 输入按住持续激活
    WhileInputActive,

    // 通过 Avatar 生成时自动激活
    OnSpawn
};
```

#### 3.2.2 激活策略详解

Lyra 引入了 **ActivationPolicy** 来控制能力的激活时机：

**1. OnInputTriggered**：按键触发

```cpp
// 示例：跳跃能力
ActivationPolicy = ELyraAbilityActivationPolicy::OnInputTriggered

// 行为：
// - 玩家按下跳跃键时激活
// - 能力执行一次后结束
// - 松开键不影响能力
```

**2. WhileInputActive**：按住期间激活

```cpp
// 示例：瞄准能力
ActivationPolicy = ELyraAbilityActivationPolicy::WhileInputActive

// 行为：
// - 玩家按下瞄准键时激活
// - 持续按住期间能力保持激活
// - 松开键时能力结束
```

**3. OnSpawn**：生成时自动激活

```cpp
// 示例：被动能力（生命恢复）
ActivationPolicy = ELyraAbilityActivationPolicy::OnSpawn

// 行为：
// - Pawn 生成时自动激活
// - 不需要玩家输入
// - 通常是无限持续的能力
```

#### 3.2.3 Lyra Ability 的标准实现模式

```cpp
// GA_Jump.h - 跳跃能力示例
UCLASS()
class UGA_Jump : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Jump(const FObjectInitializer& ObjectInitializer);

protected:
    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, 
                                     const FGameplayAbilityActorInfo* ActorInfo, 
                                     const FGameplayTagContainer* SourceTags, 
                                     const FGameplayTagContainer* TargetTags, 
                                     FGameplayTagContainer* OptionalRelevantTags) const override;

    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, 
                                  const FGameplayAbilityActorInfo* ActorInfo, 
                                  const FGameplayAbilityActivationInfo ActivationInfo, 
                                  const FGameplayEventData* TriggerEventData) override;
};

// GA_Jump.cpp
UGA_Jump::UGA_Jump(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 设置基本属性
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;

    // 设置能力标签
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag("Ability.Jump"));

    // 激活时阻止的标签（不能在眩晕时跳跃）
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Stunned"));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Dead"));

    // 激活策略
    ActivationPolicy = ELyraAbilityActivationPolicy::OnInputTriggered;
}

bool UGA_Jump::CanActivateAbility(...) const
{
    if (!Super::CanActivateAbility(...))
    {
        return false;
    }

    // 额外检查：角色必须在地面上
    const ALyraCharacter* Character = GetLyraCharacterFromActorInfo();
    if (Character && Character->GetCharacterMovement())
    {
        return Character->GetCharacterMovement()->IsMovingOnGround();
    }

    return false;
}

void UGA_Jump::ActivateAbility(...)
{
    if (HasAuthorityOrPredictionKey(ActorInfo, &ActivationInfo))
    {
        if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }

        // 执行跳跃
        if (ALyraCharacter* Character = GetLyraCharacterFromActorInfo())
        {
            Character->Jump();
        }
    }

    // 能力立即结束
    EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}
```

### 3.3 LyraAttributeSet

Lyra 使用多个 AttributeSet 来组织不同类别的属性。

#### 3.3.1 LyraHealthSet（生命值系统）

```cpp
// LyraHealthSet.h
UCLASS(BlueprintType)
class ULyraHealthSet : public ULyraAttributeSet
{
    GENERATED_BODY()

public:
    ULyraHealthSet();

    // 网络复制
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 当前生命值
    UPROPERTY(BlueprintReadOnly, Category = "Lyra|Health", ReplicatedUsing = OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(ULyraHealthSet, Health)

    // 最大生命值
    UPROPERTY(BlueprintReadOnly, Category = "Lyra|Health", ReplicatedUsing = OnRep_MaxHealth)
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(ULyraHealthSet, MaxHealth)

    // 治疗（Meta Attribute，不会被存储）
    UPROPERTY(BlueprintReadOnly, Category = "Lyra|Health", meta = (HideFromLevelInfos))
    FGameplayAttributeData Healing;
    ATTRIBUTE_ACCESSORS(ULyraHealthSet, Healing)

    // 伤害（Meta Attribute）
    UPROPERTY(BlueprintReadOnly, Category = "Lyra|Health", meta = (HideFromLevelInfos))
    FGameplayAttributeData Damage;
    ATTRIBUTE_ACCESSORS(ULyraHealthSet, Damage)

protected:
    // 复制回调
    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldValue);

    UFUNCTION()
    virtual void OnRep_MaxHealth(const FGameplayAttributeData& OldValue);

    // 属性变化前
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;

    // 效果执行后
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;

    // Clamping 辅助函数
    void ClampAttribute(const FGameplayAttribute& Attribute, float& NewValue) const;
};
```

#### 3.3.2 Meta Attributes 模式

Lyra 使用 **Meta Attributes**（元属性）来处理伤害和治疗：

```cpp
// LyraHealthSet.cpp
void ULyraHealthSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    // 处理伤害（Meta Attribute）
    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        // 获取伤害值
        const float LocalDamage = GetDamage();
        SetDamage(0.0f); // 立即清零（不存储）

        if (LocalDamage > 0.0f)
        {
            // 减少生命值
            const float NewHealth = GetHealth() - LocalDamage;
            SetHealth(FMath::Clamp(NewHealth, 0.0f, GetMaxHealth()));

            // 广播伤害事件
            if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
            {
                FLyraGameplayEffectContext* Context = static_cast<FLyraGameplayEffectContext*>(Data.EffectSpec.GetContext().Get());
                
                // 发送伤害消息（UI 可以监听）
                UGameplayMessageSubsystem& MessageSystem = UGameplayMessageSubsystem::Get(GetWorld());
                MessageSystem.BroadcastMessage(
                    TAG_Lyra_Damage_Message,
                    FLyraDamageMessage(
                        Context->GetInstigator(),
                        GetOwningActor(),
                        LocalDamage,
                        Context->GetHitResult()));
            }
        }
    }
    // 处理治疗（Meta Attribute）
    else if (Data.EvaluatedData.Attribute == GetHealingAttribute())
    {
        const float LocalHealing = GetHealing();
        SetHealing(0.0f);

        if (LocalHealing > 0.0f)
        {
            const float NewHealth = GetHealth() + LocalHealing;
            SetHealth(FMath::Clamp(NewHealth, 0.0f, GetMaxHealth()));
        }
    }
    // 直接修改 Health 的情况
    else if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        SetHealth(FMath::Clamp(GetHealth(), 0.0f, GetMaxHealth()));
    }

    // 检查死亡
    if (GetHealth() <= 0.0f && !bOutOfHealth)
    {
        bOutOfHealth = true;

        // 发送死亡事件
        if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
        {
            FGameplayEventData EventData;
            EventData.Instigator = Data.EffectSpec.GetEffectContext().GetInstigator();
            EventData.Target = GetOwningActor();

            LyraASC->HandleGameplayEvent(TAG_Lyra_Event_Death, &EventData);
        }
    }
}
```

**为什么使用 Meta Attributes？**

传统方式（直接修改 Health）的问题：
```cpp
// 问题：无法区分伤害来源、伤害类型
// - 是火焰伤害还是物理伤害？
// - 是持续伤害还是爆发伤害？
// - 是玩家造成的还是环境伤害？

Modifier: Health, Add, -50
```

Meta Attribute 方式的优势：
```cpp
// 清晰明确：这是伤害
Modifier: Damage, Add, 50

// 在 PostGameplayEffectExecute 中：
// 1. 读取 Damage 值
// 2. 应用护甲减免计算
// 3. 触发伤害事件（UI、音效、特效）
// 4. 更新 Health
// 5. 检查死亡
// 6. 清零 Damage（不存储）
```

### 3.4 Lyra 的 GAS 初始化流程

让我们看看 Lyra 是如何完整初始化 GAS 的：

```
┌──────────────────────────────────────────────────────────────┐
│ 1. GameMode 创建 PlayerState                                 │
│    AGameMode::PostLogin(APlayerController* NewPlayer)        │
│    ↓                                                          │
│    CreatePlayerState(NewPlayer)                              │
│    ↓                                                          │
│    ALyraPlayerState::ALyraPlayerState()                      │
│    {                                                          │
│        AbilitySystemComponent = CreateDefaultSubobject(...); │
│        HealthSet = CreateDefaultSubobject(...);              │
│        CombatSet = CreateDefaultSubobject(...);              │
│    }                                                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 2. GameMode 生成 Pawn                                        │
│    APawn* NewPawn = SpawnDefaultPawnAtTransform(...);        │
│    ↓                                                          │
│    ALyraCharacter::ALyraCharacter()                          │
│    {                                                          │
│        // 添加 PawnExtensionComponent                        │
│        PawnExtComp = CreateDefaultSubobject(...);            │
│    }                                                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 3. Controller Possess Pawn                                   │
│    AController::Possess(APawn* InPawn)                       │
│    ↓                                                          │
│    ALyraCharacter::PossessedBy(AController* NewController)   │
│    {                                                          │
│        Super::PossessedBy(NewController);                    │
│        ↓                                                      │
│        // 获取 PlayerState 的 ASC                            │
│        ALyraPlayerState* PS = GetPlayerState<...>();         │
│        AbilitySystemComponent = PS->GetAbilitySystemComponent(); │
│        ↓                                                      │
│        // 初始化 ActorInfo（关键！）                         │
│        AbilitySystemComponent->InitAbilityActorInfo(PS, this); │
│        ↓                                                      │
│        // 通知 PawnExtensionComponent                        │
│        PawnExtComp->HandleControllerChanged();               │
│    }                                                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 4. Experience 系统加载                                       │
│    ULyraExperienceManagerComponent::OnExperienceLoaded()    │
│    ↓                                                          │
│    // 广播 Experience 加载完成事件                           │
│    OnExperienceLoaded.Broadcast(Experience);                │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 5. PawnExtensionComponent 响应 Experience                   │
│    ULyraPawnExtensionComponent::OnExperienceLoaded()        │
│    {                                                          │
│        // 通知可以设置 PawnData 了                            │
│        CheckDefaultInitialization();                         │
│    }                                                          │
│    ↓                                                          │
│    SetPawnData(PawnData)                                     │
│    {                                                          │
│        // 1. 授予 AbilitySet                                 │
│        for (auto AbilitySet : PawnData->AbilitySets)         │
│        {                                                      │
│            AbilitySet->GiveToAbilitySystem(ASC, ...);        │
│        }                                                      │
│        ↓                                                      │
│        // 2. 应用初始化 Gameplay Effects                     │
│        for (auto Effect : PawnData->DefaultEffects)          │
│        {                                                      │
│            ASC->ApplyGameplayEffectToSelf(Effect, ...);      │
│        }                                                      │
│        ↓                                                      │
│        // 3. 广播 Pawn 初始化完成                            │
│        OnPawnReadyToInitialize.Broadcast();                  │
│    }                                                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 6. 客户端复制                                                 │
│    ALyraCharacter::OnRep_PlayerState()                       │
│    {                                                          │
│        // 客户端在 PlayerState 复制后初始化                  │
│        if (ALyraPlayerState* PS = GetPlayerState<...>())     │
│        {                                                      │
│            AbilitySystemComponent = PS->GetAbilitySystemComponent(); │
│            AbilitySystemComponent->InitAbilityActorInfo(PS, this); │
│        }                                                      │
│    }                                                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 7. 完成！GAS 系统已就绪                                      │
│    - ASC 已初始化                                             │
│    - Attributes 已添加                                        │
│    - Abilities 已授予                                         │
│    - 可以开始游戏了！                                         │
└──────────────────────────────────────────────────────────────┘
```

### 3.5 Experience 系统与 GAS 的整合

Lyra 的 Experience 系统与 GAS 深度整合，允许不同的游戏模式使用不同的能力和属性配置。

#### 3.5.1 PawnData 与 AbilitySet

```cpp
// LyraPawnData.h
UCLASS(BlueprintType)
class ULyraPawnData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // Pawn 类
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Pawn")
    TSubclassOf<APawn> PawnClass;

    // 能力集（包含角色的所有能力）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Abilities")
    TArray<TObjectPtr<ULyraAbilitySet>> AbilitySets;

    // 输入配置
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Input")
    TObjectPtr<ULyraInputConfig> InputConfig;

    // 默认相机模式
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Camera")
    TSubclassOf<ULyraCameraMode> DefaultCameraMode;
};
```

**配置示例**：

```
PawnData_ShooterHero (Data Asset)
├─ PawnClass: BP_LyraCharacter
├─ AbilitySets:
│  ├─ AbilitySet_Hero (基础能力)
│  │  ├─ GA_Hero_Jump (跳跃)
│  │  ├─ GA_Hero_Sprint (冲刺)
│  │  └─ GE_Hero_InitialStats (初始属性)
│  └─ AbilitySet_Weapon (武器能力)
│     ├─ GA_Weapon_Fire (射击)
│     ├─ GA_Weapon_Reload (换弹)
│     └─ GE_Weapon_Ammo (弹药)
├─ InputConfig: InputData_Hero
└─ DefaultCameraMode: CM_ThirdPerson
```

#### 3.5.2 动态切换 Experience

Lyra 支持运行时切换 Experience，GAS 系统会相应地更新：

```cpp
// 切换到新的 Experience
void ALyraGameMode::ChangeExperience(const ULyraExperienceDefinition* NewExperience)
{
    // 1. 卸载旧的 Experience
    UnloadCurrentExperience();
    
    // 2. 清理所有 Pawn 的能力
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        if (ALyraPlayerController* PC = Cast<ALyraPlayerController>(*It))
        {
            if (ALyraCharacter* Character = PC->GetPawn<ALyraCharacter>())
            {
                // 移除所有能力和效果
                if (ULyraAbilitySystemComponent* ASC = Character->GetAbilitySystemComponent())
                {
                    ASC->ClearAllAbilities();
                    ASC->RemoveAllActiveGameplayEffects();
                }
            }
        }
    }
    
    // 3. 加载新的 Experience
    LoadExperience(NewExperience);
    
    // 4. 重新授予能力（在 Experience 加载完成后）
    OnExperienceLoaded.AddLambda([this]()
    {
        for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
        {
            if (ALyraPlayerController* PC = Cast<ALyraPlayerController>(*It))
            {
                if (ALyraCharacter* Character = PC->GetPawn<ALyraCharacter>())
                {
                    // 重新设置 PawnData
                    Character->GetPawnExtensionComponent()->SetPawnData(NewPawnData);
                }
            }
        }
    });
}
```

---

## 4. 创建第一个 Gameplay Ability

让我们从零开始创建一个完整的 Gameplay Ability，学习整个开发流程。

### 4.1 使用蓝图创建 Ability

蓝图是快速原型开发的好选择，适合设计师使用。

#### 4.1.1 创建蓝图 Ability

1. **在内容浏览器中右键**
2. 选择 **Blueprint Class**
3. 搜索并选择 **LyraGameplayAbility** 作为父类
4. 命名为 `GA_Dash`（冲刺技能）

#### 4.1.2 配置基本属性

在蓝图编辑器中配置：

```
Class Defaults:
├─ Ability Tags:
│  └─ Ability.Movement.Dash
├─ Activation Owned Tags:
│  └─ State.Dashing
├─ Activation Blocked Tags:
│  ├─ State.Stunned
│  ├─ State.Dead
│  └─ State.Dashing (防止重复冲刺)
├─ Activation Required Tags: (留空)
├─ Activation Policy: On Input Triggered
├─ Instancing Policy: Instanced Per Execution
└─ Net Execution Policy: Local Predicted
```

#### 4.1.3 实现 ActivateAbility 逻辑

在蓝图事件图表中：

```
Event ActivateAbility
  ↓
[Branch] Check If Has Authority Or Prediction Key
  ├─ True:
  │   ↓
  │  [Commit Ability]
  │   ├─ Success:
  │   │   ↓
  │   │  [Get Character]
  │   │   ↓
  │   │  [Get Character Movement]
  │   │   ↓
  │   │  [Launch Character]
  │   │   • Velocity = Character Forward Vector * 1000
  │   │   • Override XY = true
  │   │   • Override Z = false
  │   │   ↓
  │   │  [Play Montage and Wait]
  │   │   • Montage = AM_Dash
  │   │   ↓
  │   │  [Wait Delay] 0.3s
  │   │   ↓
  │   │  [End Ability]
  │   │
  │   └─ Failed:
  │       ↓
  │      [End Ability] (Cancelled = true)
  │
  └─ False:
      ↓
     [End Ability] (Cancelled = true)
```

#### 4.1.4 添加冷却和消耗

**创建冷却 Gameplay Effect**：

1. 创建 Blueprint，父类：**GameplayEffect**
2. 命名：`GE_Cooldown_Dash`
3. 配置：

```
Duration Policy: Has Duration
Duration Magnitude: 3.0 (冷却 3 秒)

Granted Tags:
  └─ Cooldown.Ability.Dash

Application Tag Requirements:
  Ignore Tags:
    └─ Cooldown.Ability.Dash (如果已在冷却，忽略新的冷却)
```

**创建消耗 Gameplay Effect**：

1. 创建 Blueprint，父类：**GameplayEffect**
2. 命名：`GE_Cost_Dash`
3. 配置：

```
Duration Policy: Instant

Modifiers:
  ├─ Attribute: Stamina
  ├─ Operation: Add
  └─ Magnitude: -20.0 (消耗 20 点耐力)
```

**在 GA_Dash 中引用**：

```
Class Defaults:
├─ Cost Gameplay Effect Class: GE_Cost_Dash
└─ Cooldown Gameplay Effect Class: GE_Cooldown_Dash
```

现在，`CommitAbility()` 会自动检查和应用这些效果。

### 4.2 使用 C++ 创建 Ability

C++ 提供更好的性能和更灵活的控制。

#### 4.2.1 创建 C++ 类

```cpp
// GA_Dash.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_Dash.generated.h"

/**
 * UGA_Dash
 * 
 * 冲刺能力：让角色快速向前冲刺一段距离
 */
UCLASS()
class LYRAGAME_API UGA_Dash : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Dash(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

protected:
    // 能力激活
    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData) override;

    // 能力结束
    virtual void EndAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility,
        bool bWasCancelled) override;

    // 冲刺距离
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Dash")
    float DashDistance = 1000.0f;

    // 冲刺持续时间
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Dash")
    float DashDuration = 0.3f;

    // 冲刺动画
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Dash")
    TObjectPtr<UAnimMontage> DashMontage;

private:
    // 计时器句柄
    FTimerHandle DashTimerHandle;

    // 冲刺结束回调
    void OnDashFinished();
};
```

#### 4.2.2 实现 Ability 逻辑

```cpp
// GA_Dash.cpp
#include "GA_Dash.h"
#include "Character/LyraCharacter.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystem/LyraAbilitySystemComponent.h"

UGA_Dash::UGA_Dash(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 设置能力标签
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Movement.Dash")));

    // 激活时添加的标签
    ActivationOwnedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dashing")));

    // 阻止激活的标签
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Stunned")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dead")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dashing")));

    // 激活策略
    ActivationPolicy = ELyraAbilityActivationPolicy::OnInputTriggered;

    // 实例化策略
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerExecution;

    // 网络执行策略（客户端预测）
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

void UGA_Dash::ActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 获取角色
    ALyraCharacter* Character = CastChecked<ALyraCharacter>(ActorInfo->AvatarActor.Get());
    UCharacterMovementComponent* MovementComp = Character->GetCharacterMovement();

    // 计算冲刺方向和速度
    FVector DashDirection = Character->GetActorForwardVector();
    FVector LaunchVelocity = DashDirection * (DashDistance / DashDuration);

    // 只影响水平方向
    LaunchVelocity.Z = 0.0f;

    // 发射角色
    Character->LaunchCharacter(LaunchVelocity, true, true);

    // 播放动画
    if (DashMontage)
    {
        Character->PlayAnimMontage(DashMontage);
    }

    // 设置定时器，在冲刺结束时调用
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            DashTimerHandle,
            this,
            &UGA_Dash::OnDashFinished,
            DashDuration,
            false);
    }
}

void UGA_Dash::OnDashFinished()
{
    // 结束能力
    bool bReplicateEndAbility = true;
    bool bWasCancelled = false;
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, bReplicateEndAbility, bWasCancelled);
}

void UGA_Dash::EndAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    bool bReplicateEndAbility,
    bool bWasCancelled)
{
    // 清理定时器
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(DashTimerHandle);
    }

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

### 4.3 配置 Ability 的激活策略

我们已经在代码中设置了 `ActivationPolicy`，但还需要配置输入绑定。

#### 4.3.1 创建输入配置

在 Lyra 中，输入通过 **Input Config** 和 **Gameplay Tags** 绑定。

```cpp
// LyraInputConfig.h (已存在)
USTRUCT(BlueprintType)
struct FLyraInputAction
{
    GENERATED_BODY()

    // Enhanced Input Action
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<const UInputAction> InputAction = nullptr;

    // 输入标签
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};

UCLASS(BlueprintType, Const)
class ULyraInputConfig : public UDataAsset
{
    GENERATED_BODY()

public:
    // 查找输入动作
    UFUNCTION(BlueprintCallable)
    const UInputAction* FindInputActionForTag(const FGameplayTag& InputTag) const;

    // 原生输入动作（跳跃、移动等）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Meta = (TitleProperty = "InputAction"))
    TArray<FLyraInputAction> NativeInputActions;

    // 能力输入动作（技能等）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Meta = (TitleProperty = "InputAction"))
    TArray<FLyraInputAction> AbilityInputActions;
};
```

**在编辑器中配置**：

1. 找到或创建 `InputConfig_Hero` Data Asset
2. 添加新的输入动作：

```
Ability Input Actions:
  ├─ [0] Jump
  │  ├─ Input Action: IA_Jump
  │  └─ Input Tag: InputTag.Ability.Jump
  ├─ [1] Sprint
  │  ├─ Input Action: IA_Sprint
  │  └─ Input Tag: InputTag.Ability.Sprint
  └─ [2] Dash (新增)
     ├─ Input Action: IA_Dash
     └─ Input Tag: InputTag.Ability.Dash
```

#### 4.3.2 在 AbilitySet 中绑定输入标签

```cpp
// 在 AbilitySet_Hero 中添加：
Granted Gameplay Abilities:
  └─ [2] Dash
     ├─ Ability: GA_Dash
     ├─ Ability Level: 1
     └─ Input Tag: InputTag.Ability.Dash
```

现在，当玩家按下绑定到 `IA_Dash` 的键（例如 Left Shift）时，输入系统会：

1. 触发 `IA_Dash`
2. 查找 `InputTag.Ability.Dash`
3. 调用 `ASC->AbilityInputTagPressed(InputTag.Ability.Dash)`
4. 激活绑定到该标签的能力 `GA_Dash`

### 4.4 授予和激活 Ability

#### 4.4.1 通过 AbilitySet 自动授予

如前所述，Lyra 使用 AbilitySet 系统自动授予能力。只需将能力添加到 PawnData 的 AbilitySet 中即可。

#### 4.4.2 手动授予 Ability（用于测试）

```cpp
// 在角色生成后手动授予
void ALyraCharacter::GrantTestAbility()
{
    if (ULyraAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        // 创建 Ability Spec
        FGameplayAbilitySpec AbilitySpec(
            UGA_Dash::StaticClass(), // Ability 类
            1,                       // Level
            INDEX_NONE,              // InputID（Lyra 不使用）
            this);                   // Source Object

        // 授予能力
        FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(AbilitySpec);
        
        // 可选：立即激活
        ASC->TryActivateAbility(Handle);
    }
}
```

#### 4.4.3 通过 Gameplay Event 激活

某些能力不通过输入激活，而是通过游戏事件触发：

```cpp
// 定义能力的触发标签
// 在 GA_Dash 构造函数中：
AbilityTriggers.AddTag(FGameplayTag::RequestGameplayTag("Event.Movement.Dash"));

// 在其他地方触发
if (UAbilitySystemBlueprintLibrary* BPLib = UAbilitySystemBlueprintLibrary::StaticClass()->GetDefaultObject<UAbilitySystemBlueprintLibrary>())
{
    FGameplayEventData EventData;
    EventData.Instigator = this;
    EventData.Target = TargetActor;
    
    ASC->HandleGameplayEvent(
        FGameplayTag::RequestGameplayTag("Event.Movement.Dash"),
        &EventData);
}
```

### 4.5 调试 Ability

GAS 提供了强大的调试工具。

#### 4.5.1 控制台命令

在 PIE（Play In Editor）中按 `~` 打开控制台，输入：

**显示 GAS 调试信息**：
```
showdebug abilitysystem
```

显示内容：
- 当前拥有的 Abilities
- 激活中的 Abilities
- Active Gameplay Effects
- Owned Gameplay Tags
- Attribute 当前值

**显示详细信息**：
```
AbilitySystem.Debug.NextCategory
```
切换显示类别（Abilities / Effects / Attributes / Tags）

**激活特定能力**（用于测试）：
```
AbilitySystem.ActivateAbility <AbilityName>
```

#### 4.5.2 日志输出

在代码中添加日志：

```cpp
void UGA_Dash::ActivateAbility(...)
{
    UE_LOG(LogLyraAbilitySystem, Log, TEXT("Dash ability activated"));

    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        UE_LOG(LogLyraAbilitySystem, Warning, TEXT("Dash ability failed to commit"));
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // ...
}
```

启用日志：
```
// 在 DefaultEngine.ini 中添加
[Core.Log]
LogLyraAbilitySystem=Verbose
LogAbilitySystem=Verbose
LogGameplayEffects=Verbose
```

#### 4.5.3 蓝图调试

在蓝图 Ability 中：
- 使用 **Print String** 节点输出调试信息
- 使用 **Breakpoints** 暂停执行
- 检查变量值

---

## 5. Attribute 系统详解

属性系统是 GAS 的核心，用于管理角色的数值数据。

### 5.1 什么是 Attribute

**Attribute（属性）** 是一个可以被 Gameplay Effects 修改的浮点数值，例如：
- 生命值 (Health)
- 魔法值 (Mana)
- 耐力 (Stamina)
- 攻击力 (Attack Power)
- 移动速度 (Movement Speed)

每个 Attribute 包含两个值：
- **Base Value**（基础值）：永久修改，如升级、装备
- **Current Value**（当前值）：基础值 + 临时修改（Buffs/Debuffs）

### 5.2 创建自定义 AttributeSet

让我们创建一个包含耐力（Stamina）的 AttributeSet。

#### 5.2.1 头文件

```cpp
// MyStaminaSet.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystemComponent.h"
#include "AttributeSet.h"
#include "MyStaminaSet.generated.h"

// 使用宏简化属性访问器的声明
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

/**
 * UMyStaminaSet
 * 
 * 耐力属性集
 */
UCLASS()
class LYRAGAME_API UMyStaminaSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    UMyStaminaSet();

    // 网络复制
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 属性变化前调用（用于限制值）
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;

    // Gameplay Effect 执行后调用（用于响应变化）
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;

    // ============================================================
    // 属性定义
    // ============================================================

    // 当前耐力
    UPROPERTY(BlueprintReadOnly, Category = "Stamina", ReplicatedUsing = OnRep_Stamina)
    FGameplayAttributeData Stamina;
    ATTRIBUTE_ACCESSORS(UMyStaminaSet, Stamina)

    // 最大耐力
    UPROPERTY(BlueprintReadOnly, Category = "Stamina", ReplicatedUsing = OnRep_MaxStamina)
    FGameplayAttributeData MaxStamina;
    ATTRIBUTE_ACCESSORS(UMyStaminaSet, MaxStamina)

    // 耐力恢复速度（每秒）
    UPROPERTY(BlueprintReadOnly, Category = "Stamina", ReplicatedUsing = OnRep_StaminaRegenRate)
    FGameplayAttributeData StaminaRegenRate;
    ATTRIBUTE_ACCESSORS(UMyStaminaSet, StaminaRegenRate)

protected:
    // 复制回调
    UFUNCTION()
    virtual void OnRep_Stamina(const FGameplayAttributeData& OldStamina);

    UFUNCTION()
    virtual void OnRep_MaxStamina(const FGameplayAttributeData& OldMaxStamina);

    UFUNCTION()
    virtual void OnRep_StaminaRegenRate(const FGameplayAttributeData& OldStaminaRegenRate);

    // 辅助函数：限制属性值
    void ClampAttribute(const FGameplayAttribute& Attribute, float& NewValue) const;

private:
    // 是否耗尽（用于触发事件）
    bool bStaminaDepleted = false;
};
```

#### 5.2.2 实现文件

```cpp
// MyStaminaSet.cpp
#include "MyStaminaSet.h"
#include "Net/UnrealNetwork.h"
#include "GameplayEffectExtension.h"
#include "AbilitySystem/LyraAbilitySystemComponent.h"

UMyStaminaSet::UMyStaminaSet()
{
    // 设置默认值
    Stamina = 100.0f;
    MaxStamina = 100.0f;
    StaminaRegenRate = 10.0f; // 每秒恢复 10 点
}

void UMyStaminaSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 注册需要复制的属性
    DOREPLIFETIME_CONDITION_NOTIFY(UMyStaminaSet, Stamina, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyStaminaSet, MaxStamina, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyStaminaSet, StaminaRegenRate, COND_None, REPNOTIFY_Always);
}

void UMyStaminaSet::OnRep_Stamina(const FGameplayAttributeData& OldStamina)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyStaminaSet, Stamina, OldStamina);
}

void UMyStaminaSet::OnRep_MaxStamina(const FGameplayAttributeData& OldMaxStamina)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyStaminaSet, MaxStamina, OldMaxStamina);
}

void UMyStaminaSet::OnRep_StaminaRegenRate(const FGameplayAttributeData& OldStaminaRegenRate)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyStaminaSet, StaminaRegenRate, OldStaminaRegenRate);
}

void UMyStaminaSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    // 限制属性范围
    ClampAttribute(Attribute, NewValue);
}

void UMyStaminaSet::ClampAttribute(const FGameplayAttribute& Attribute, float& NewValue) const
{
    if (Attribute == GetStaminaAttribute())
    {
        // 限制 Stamina 在 [0, MaxStamina] 范围内
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxStamina());
    }
    else if (Attribute == GetMaxStaminaAttribute())
    {
        // MaxStamina 不能小于 1
        NewValue = FMath::Max(NewValue, 1.0f);
    }
    else if (Attribute == GetStaminaRegenRateAttribute())
    {
        // 恢复速度不能为负
        NewValue = FMath::Max(NewValue, 0.0f);
    }
}

void UMyStaminaSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    if (Data.EvaluatedData.Attribute == GetStaminaAttribute())
    {
        // 再次限制（因为 PreAttributeChange 只影响 CurrentValue）
        SetStamina(FMath::Clamp(GetStamina(), 0.0f, GetMaxStamina()));

        // 检查是否耗尽
        if (GetStamina() <= 0.0f && !bStaminaDepleted)
        {
            bStaminaDepleted = true;

            // 广播耐力耗尽事件
            if (ULyraAbilitySystemComponent* LyraASC = Cast<ULyraAbilitySystemComponent>(GetOwningAbilitySystemComponent()))
            {
                // 可以添加 Gameplay Tag
                LyraASC->AddLooseGameplayTag(FGameplayTag::RequestGameplayTag("State.StaminaDepleted"));

                // 或发送 Gameplay Event
                FGameplayEventData EventData;
                EventData.Instigator = GetOwningActor();
                LyraASC->HandleGameplayEvent(
                    FGameplayTag::RequestGameplayTag("Event.Stamina.Depleted"),
                    &EventData);
            }
        }
        else if (GetStamina() > 0.0f && bStaminaDepleted)
        {
            bStaminaDepleted = false;

            // 移除耗尽标签
            if (ULyraAbilitySystemComponent* LyraASC = Cast<ULyraAbilitySystemComponent>(GetOwningAbilitySystemComponent()))
            {
                LyraASC->RemoveLooseGameplayTag(FGameplayTag::RequestGameplayTag("State.StaminaDepleted"));
            }
        }
    }
    else if (Data.EvaluatedData.Attribute == GetMaxStaminaAttribute())
    {
        // MaxStamina 变化时，调整 Stamina
        SetStamina(FMath::Clamp(GetStamina(), 0.0f, GetMaxStamina()));
    }
}
```

### 5.3 Attribute 的复制和网络同步

#### 5.3.1 复制流程

GAS 的 Attribute 复制是自动的，但需要正确配置：

```cpp
// 1. 属性必须标记为 Replicated
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Stamina)
FGameplayAttributeData Stamina;

// 2. 在 GetLifetimeReplicatedProps 中注册
DOREPLIFETIME_CONDITION_NOTIFY(UMyStaminaSet, Stamina, COND_None, REPNOTIFY_Always);

// 3. 实现 OnRep 回调
UFUNCTION()
void OnRep_Stamina(const FGameplayAttributeData& OldStamina)
{
    // 通知 ASC 属性已复制
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyStaminaSet, Stamina, OldStamina);
}
```

#### 5.3.2 复制条件

```cpp
// COND_None: 总是复制给所有客户端
DOREPLIFETIME_CONDITION_NOTIFY(UMyStaminaSet, Health, COND_None, REPNOTIFY_Always);

// COND_OwnerOnly: 只复制给 Owner 客户端（用于敏感数据）
DOREPLIFETIME_CONDITION_NOTIFY(UMyStaminaSet, InternalCooldown, COND_OwnerOnly, REPNOTIFY_Always);

// COND_SkipOwner: 复制给所有人除了 Owner（罕见）
DOREPLIFETIME_CONDITION_NOTIFY(UMyStaminaSet, SomeAttribute, COND_SkipOwner, REPNOTIFY_Always);
```

#### 5.3.3 RepNotify 策略

```cpp
// REPNOTIFY_Always: 即使值没变也触发（GAS 推荐）
DOREPLIFETIME_CONDITION_NOTIFY(UMyStaminaSet, Stamina, COND_None, REPNOTIFY_Always);

// REPNOTIFY_OnChanged: 只在值改变时触发（默认）
DOREPLIFETIME_CONDITION_NOTIFY(UMyStaminaSet, Stamina, COND_None, REPNOTIFY_OnChanged);
```

**为什么 GAS 推荐 REPNOTIFY_Always？**

因为 `FGameplayAttributeData` 包含 `BaseValue` 和 `CurrentValue` 两个值，即使 `CurrentValue` 没变，`BaseValue` 也可能变了。

### 5.4 属性修改器与 Pre/Post AttributeChange

#### 5.4.1 PreAttributeChange

在属性修改**之前**调用，用于限制值的范围：

```cpp
void UMyStaminaSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    if (Attribute == GetStaminaAttribute())
    {
        // 限制 Stamina 范围
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxStamina());
    }
}
```

**注意**：
- `PreAttributeChange` 只影响 `CurrentValue`，不影响 `BaseValue`
- 不会阻止属性修改，只是调整值
- 不会触发网络复制

#### 5.4.2 PostGameplayEffectExecute

在 Gameplay Effect **执行后**调用，用于响应属性变化：

```cpp
void UMyStaminaSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    if (Data.EvaluatedData.Attribute == GetStaminaAttribute())
    {
        // 最终限制（影响 BaseValue 和 CurrentValue）
        SetStamina(FMath::Clamp(GetStamina(), 0.0f, GetMaxStamina()));

        // 触发游戏逻辑
        if (GetStamina() <= 0.0f)
        {
            // 耐力耗尽，触发事件
            OnStaminaDepleted();
        }

        // 广播 UI 更新
        if (OnStaminaChanged.IsBound())
        {
            OnStaminaChanged.Broadcast(GetStamina(), GetMaxStamina());
        }
    }
}
```

**常用场景**：
- 限制属性范围（Clamp）
- 触发死亡/耗尽事件
- 更新 UI
- 应用连锁效果（如低血量触发被动）

#### 5.4.3 调用时机对比

```
Gameplay Effect 应用流程：

1. ApplyGameplayEffectToSelf()
   ↓
2. 计算 Modifier（考虑 Attribute Based、Custom Calculation 等）
   ↓
3. PreAttributeChange() ← 在这里调整 NewValue
   ↓
4. 应用修改到 CurrentValue
   ↓
5. PostGameplayEffectExecute() ← 在这里响应变化
   ↓
6. OnRep_Attribute() (如果是网络复制)
```

### 5.5 Lyra 中的 AttributeSet 实现

Lyra 将属性拆分成多个 AttributeSet，遵循**单一职责原则**。

#### 5.5.1 LyraHealthSet

```cpp
// 生命值相关属性
UCLASS()
class ULyraHealthSet : public UAttributeSet
{
    UPROPERTY(BlueprintReadOnly, Replicated)
    FGameplayAttributeData Health;

    UPROPERTY(BlueprintReadOnly, Replicated)
    FGameplayAttributeData MaxHealth;

    // Meta Attributes（不存储）
    UPROPERTY(BlueprintReadOnly)
    FGameplayAttributeData Healing;

    UPROPERTY(BlueprintReadOnly)
    FGameplayAttributeData Damage;
};
```

#### 5.5.2 LyraCombatSet

```cpp
// 战斗相关属性
UCLASS()
class ULyraCombatSet : public UAttributeSet
{
    UPROPERTY(BlueprintReadOnly, Replicated)
    FGameplayAttributeData BaseDamage;

    UPROPERTY(BlueprintReadOnly, Replicated)
    FGameplayAttributeData BaseHeal;

    // 护甲
    UPROPERTY(BlueprintReadOnly, Replicated)
    FGameplayAttributeData Armor;

    // 伤害倍率
    UPROPERTY(BlueprintReadOnly, Replicated)
    FGameplayAttributeData DamageMultiplier;
};
```

#### 5.5.3 为什么要拆分 AttributeSet？

**优势**：
- ✅ 模块化：不同系统独立管理
- ✅ 可选性：某些游戏模式可能不需要某些属性
- ✅ 复制优化：可以针对不同 AttributeSet 设置不同的复制策略

**示例**：
```cpp
// 在 PvP 模式中只需要基础属性
PawnData_PvP:
  AttributeSets:
    - HealthSet
    - CombatSet

// 在 PvE 模式中添加更多属性
PawnData_PvE:
  AttributeSets:
    - HealthSet
    - CombatSet
    - StaminaSet
    - ExperienceSet
```

### 5.6 属性初始化数据表

使用 **Data Table** 初始化属性值。

#### 5.6.1 创建数据表

1. **创建 Row Structure**（通常已由引擎提供）：

GAS 提供了 `FAttributeMetaData` 结构体，但通常我们创建自定义的：

```cpp
// MyAttributeInitData.h
USTRUCT(BlueprintType)
struct FMyAttributeInitData : public FTableRowBase
{
    GENERATED_BODY()

    // 属性名称（如 "Health"）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString AttributeName;

    // 基础值
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float BaseValue = 0.0f;
};
```

2. **创建 Data Table**：
   - 右键 -> Miscellaneous -> Data Table
   - 选择 `FMyAttributeInitData` 作为 Row Structure
   - 命名：`DT_AttributeInit_Default`

3. **填充数据**：

| Row Name | Attribute Name | Base Value |
|----------|----------------|------------|
| Health | Health | 100.0 |
| MaxHealth | MaxHealth | 100.0 |
| Stamina | Stamina | 100.0 |
| MaxStamina | MaxStamina | 100.0 |
| StaminaRegenRate | StaminaRegenRate | 10.0 |

#### 5.6.2 应用数据表

```cpp
void ALyraPlayerState::InitializeAttributes()
{
    if (!AbilitySystemComponent)
    {
        return;
    }

    // 加载数据表
    static ConstructorHelpers::FObjectFinder<UDataTable> DataTable(
        TEXT("/Game/Data/DT_AttributeInit_Default"));

    if (DataTable.Succeeded())
    {
        // 遍历数据表
        TArray<FMyAttributeInitData*> Rows;
        DataTable.Object->GetAllRows<FMyAttributeInitData>(TEXT("InitAttributes"), Rows);

        for (FMyAttributeInitData* Row : Rows)
        {
            // 查找属性
            FGameplayAttribute Attribute = FindAttributeByName(Row->AttributeName);
            if (Attribute.IsValid())
            {
                // 设置基础值
                AbilitySystemComponent->SetNumericAttributeBase(Attribute, Row->BaseValue);
            }
        }
    }
}

FGameplayAttribute ALyraPlayerState::FindAttributeByName(const FString& AttributeName)
{
    // 遍历所有 AttributeSet，查找匹配的属性
    for (const UAttributeSet* Set : AbilitySystemComponent->GetSpawnedAttributes())
    {
        for (TFieldIterator<FProperty> It(Set->GetClass()); It; ++It)
        {
            FProperty* Property = *It;
            if (Property && Property->GetName() == AttributeName)
            {
                return FGameplayAttribute(Property);
            }
        }
    }

    return FGameplayAttribute();
}
```

#### 5.6.3 通过 Gameplay Effect 初始化（推荐）

更优雅的方式是使用 **Instant Gameplay Effect**：

1. **创建 GE_InitializeAttributes**：

```
Duration Policy: Instant

Modifiers:
  ├─ [0] Health
  │  ├─ Attribute: Health
  │  ├─ Operation: Add
  │  └─ Magnitude: 100.0
  ├─ [1] MaxHealth
  │  ├─ Attribute: MaxHealth
  │  ├─ Operation: Add
  │  └─ Magnitude: 100.0
  ├─ [2] Stamina
  │  ├─ Attribute: Stamina
  │  ├─ Operation: Add
  │  └─ Magnitude: 100.0
  └─ [3] MaxStamina
     ├─ Attribute: MaxStamina
     ├─ Operation: Add
     └─ Magnitude: 100.0
```

2. **在 PawnData 中引用**：

```cpp
// LyraPawnData
Default Effects:
  └─ GE_InitializeAttributes
```

3. **自动应用**：

当 Pawn 初始化时，Experience 系统会自动应用这个效果，初始化所有属性。

---

## 6. Gameplay Tags 深度应用

Gameplay Tags 是 GAS 的"语言"，用于描述状态、能力、事件等。

### 6.1 什么是 Gameplay Tags

Gameplay Tags 是**分层的字符串标识符**，用于：
- 标识能力（`Ability.Melee.Attack`）
- 描述状态（`State.Stunned`，`State.Dead`）
- 控制能力激活（`Cooldown.Skill.Fireball`）
- 触发事件（`Event.Death`，`Event.LevelUp`）
- 分类装备（`Weapon.Rifle`，`Weapon.Pistol`）

**示例层级结构**：

```
Ability
├─ Melee
│  ├─ Attack
│  ├─ HeavyAttack
│  └─ Combo
├─ Range
│  ├─ Shoot
│  └─ Snipe
└─ Magic
   ├─ Fireball
   └─ IceBlast

State
├─ Alive
├─ Dead
├─ Stunned
├─ Silenced
└─ Invulnerable

Cooldown
├─ Ability
│  ├─ Jump
│  ├─ Dash
│  └─ Sprint
└─ Item
   ├─ HealthPotion
   └─ ManaPotion

Event
├─ Death
├─ Respawn
├─ LevelUp
└─ ItemPickup
```

### 6.2 创建和管理 Tags

#### 6.2.1 通过 Project Settings 创建

1. **打开 Project Settings**
2. 导航到 **Gameplay Tags**
3. 点击 **Add New Gameplay Tag**
4. 输入 Tag 名称（如 `Ability.Movement.Dash`）
5. 添加描述（如 "Dash ability tag"）

#### 6.2.2 通过 INI 文件批量创建

编辑 `Config/DefaultGameplayTags.ini`：

```ini
[/Script/GameplayTags.GameplayTagsList]
+GameplayTagList=(Tag="Ability.Movement.Jump",DevComment="跳跃能力")
+GameplayTagList=(Tag="Ability.Movement.Dash",DevComment="冲刺能力")
+GameplayTagList=(Tag="Ability.Movement.Sprint",DevComment="奔跑能力")
+GameplayTagList=(Tag="State.Dead",DevComment="死亡状态")
+GameplayTagList=(Tag="State.Stunned",DevComment="眩晕状态")
+GameplayTagList=(Tag="State.Silenced",DevComment="沉默状态")
+GameplayTagList=(Tag="Cooldown.Ability.Dash",DevComment="冲刺冷却")
+GameplayTagList=(Tag="Event.Death",DevComment="死亡事件")
```

#### 6.2.3 在 C++ 中注册（不推荐）

```cpp
// 在 GameInstance::Init() 中
void UMyGameInstance::Init()
{
    Super::Init();

    // 注册 Native Gameplay Tags
    UGameplayTagsManager& TagManager = UGameplayTagsManager::Get();
    
    FGameplayTag DashTag = TagManager.AddNativeGameplayTag(
        FName("Ability.Movement.Dash"),
        FString("Dash ability tag"));
}
```

**不推荐的原因**：
- 难以管理（分散在代码中）
- 不能在编辑器中查看
- 容易出现拼写错误

### 6.3 Tag 在 Ability 中的应用

#### 6.3.1 Ability Tags

定义这个能力**是什么**：

```cpp
// GA_Dash 构造函数
AbilityTags.AddTag(FGameplayTag::RequestGameplayTag("Ability.Movement.Dash"));

// 用途：
// - 查询特定能力是否被授予
// - 过滤能力列表
// - 能力组管理
```

#### 6.3.2 Cancel Abilities With Tag

激活此能力时，**取消拥有指定标签的其他能力**：

```cpp
// 冲刺时取消奔跑
CancelAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag("Ability.Movement.Sprint"));

// 释放法术时取消所有近战攻击
CancelAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag("Ability.Melee"));
```

#### 6.3.3 Block Abilities With Tag

此能力激活时，**阻止拥有指定标签的能力激活**：

```cpp
// 冲刺时阻止跳跃
BlockAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag("Ability.Movement.Jump"));

// 换弹时阻止射击
BlockAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag("Ability.Weapon.Fire"));
```

#### 6.3.4 Activation Owned Tags

能力激活期间，**添加到 ASC 的标签**：

```cpp
// 冲刺时添加 State.Dashing
ActivationOwnedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Dashing"));

// 用途：
// - 其他系统可以检测这个状态
// - UI 可以显示图标
// - 动画系统可以切换状态
```

#### 6.3.5 Activation Required Tags

激活此能力需要 ASC **拥有**的标签：

```cpp
// 必须活着才能使用能力
ActivationRequiredTags.AddTag(FGameplayTag::RequestGameplayTag("State.Alive"));

// 必须在地面上才能冲刺
ActivationRequiredTags.AddTag(FGameplayTag::RequestGameplayTag("State.OnGround"));
```

#### 6.3.6 Activation Blocked Tags

激活此能力时 ASC **不能拥有**的标签：

```cpp
// 被眩晕时不能使用能力
ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Stunned"));

// 被沉默时不能使用魔法
ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Silenced"));

// 死亡时不能使用任何能力
ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Dead"));
```

#### 6.3.7 Source Required Tags

**施法者** ASC 必须拥有的标签：

```cpp
// 必须是队友才能治疗
SourceRequiredTags.AddTag(FGameplayTag::RequestGameplayTag("Team.Ally"));
```

#### 6.3.8 Source Blocked Tags

**施法者** ASC 不能拥有的标签：

```cpp
// 被沉默时不能释放法术
SourceBlockedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Silenced"));
```

#### 6.3.9 Target Required Tags

**目标** ASC 必须拥有的标签：

```cpp
// 只能对敌人使用伤害技能
TargetRequiredTags.AddTag(FGameplayTag::RequestGameplayTag("Team.Enemy"));

// 只能对活着的单位使用
TargetRequiredTags.AddTag(FGameplayTag::RequestGameplayTag("State.Alive"));
```

#### 6.3.10 Target Blocked Tags

**目标** ASC 不能拥有的标签：

```cpp
// 不能对无敌目标造成伤害
TargetBlockedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Invulnerable"));
```

### 6.4 Tag Query 和复杂条件

有时需要更复杂的条件判断，例如：
- (A AND B) OR C
- A AND (B OR C)
- NOT A

#### 6.4.1 使用 FGameplayTagQuery

```cpp
// 创建复杂查询
FGameplayTagQuery Query;

// 示例 1：需要 (Alive AND OnGround) OR (Flying)
Query = FGameplayTagQuery::BuildQuery(
    FGameplayTagQueryExpression()
        .AnyTagsMatch()
        .AddTag(FGameplayTag::RequestGameplayTag("State.Alive"))
            .AllTagsMatch()
            .AddTag(FGameplayTag::RequestGameplayTag("State.OnGround"))
        .AddTag(FGameplayTag::RequestGameplayTag("State.Flying"))
);

// 示例 2：NOT Stunned AND NOT Silenced
Query = FGameplayTagQuery::BuildQuery(
    FGameplayTagQueryExpression()
        .AllTagsMatch()
        .NoTagsMatch()
            .AddTag(FGameplayTag::RequestGameplayTag("State.Stunned"))
            .AddTag(FGameplayTag::RequestGameplayTag("State.Silenced"))
);

// 检查是否匹配
if (Query.Matches(AbilitySystemComponent->GetOwnedGameplayTags()))
{
    // 条件满足
}
```

#### 6.4.2 在蓝图中使用 Tag Query

1. 在 Ability 或 Effect 的属性中找到 **Tag Query** 字段
2. 展开并配置：

```
Activation Required Tags Query:
  ├─ Query Expression: Any Tags Match
  ├─ Tags:
  │  ├─ State.Alive
  │  └─ State.Invulnerable
```

### 6.5 Lyra 中的 Tag 体系

Lyra 使用了一套结构化的 Tag 命名规范。

#### 6.5.1 Lyra Tag 分类

**1. Ability Tags**

```
Ability.Behavior.SurvivesDeath    # 死亡后能力保留
Ability.Type.Action               # 动作类型能力
Ability.Type.StatusChange         # 状态改变能力
```

**2. Input Tags**

```
InputTag.Ability.Jump             # 跳跃输入
InputTag.Ability.Crouch           # 蹲下输入
InputTag.Weapon.Fire              # 射击输入
InputTag.Weapon.Reload            # 换弹输入
InputTag.Weapon.NextWeapon        # 切换武器输入
```

**3. State Tags**

```
State.Dead                        # 死亡状态
State.Dying                       # 正在死亡
State.Alive                       # 存活状态
State.Aiming                      # 瞄准状态
State.Crouching                   # 蹲下状态
```

**4. GameplayEvent Tags**

```
GameplayEvent.Death               # 死亡事件
GameplayEvent.Reset               # 重置事件
GameplayEvent.RequestReset        # 请求重置
```

**5. GameplayCue Tags**

```
GameplayCue.Weapon.Rifle.Fire     # 步枪开火视觉效果
GameplayCue.Character.Dash        # 冲刺特效
GameplayCue.Hit.Impact            # 命中冲击效果
```

#### 6.5.2 Lyra Tag 初始化

Lyra 使用 **Native Gameplay Tags** 在 C++ 中集中定义：

```cpp
// LyraGameplayTags.h
namespace LyraGameplayTags
{
    // Input Tags
    LYRAGAME_API extern FGameplayTag InputTag_Move;
    LYRAGAME_API extern FGameplayTag InputTag_Look_Mouse;
    LYRAGAME_API extern FGameplayTag InputTag_Crouch;
    LYRAGAME_API extern FGameplayTag InputTag_Jump;

    // Ability Tags
    LYRAGAME_API extern FGameplayTag Ability_Behavior_SurvivesDeath;

    // State Tags
    LYRAGAME_API extern FGameplayTag State_Dead;
    LYRAGAME_API extern FGameplayTag State_Dying;

    // Event Tags
    LYRAGAME_API extern FGameplayTag GameplayEvent_Death;
    LYRAGAME_API extern FGameplayTag GameplayEvent_Reset;
}

// LyraGameplayTags.cpp
namespace LyraGameplayTags
{
    // 定义
    FGameplayTag InputTag_Move;
    FGameplayTag InputTag_Look_Mouse;
    // ...

    // 初始化函数
    void InitializeNativeTags()
    {
        UGameplayTagsManager& Manager = UGameplayTagsManager::Get();

        InputTag_Move = Manager.AddNativeGameplayTag(
            FName("InputTag.Move"),
            FString("Move input"));

        InputTag_Look_Mouse = Manager.AddNativeGameplayTag(
            FName("InputTag.Look.Mouse"),
            FString("Look (mouse) input"));

        // ...
    }
}

// 在 GameInstance::Init() 中调用
void ULyraGameInstance::Init()
{
    Super::Init();
    LyraGameplayTags::InitializeNativeTags();
}
```

**使用**：

```cpp
// 不需要 RequestGameplayTag，直接使用
if (ASC->HasMatchingGameplayTag(LyraGameplayTags::State_Dead))
{
    // 角色已死亡
}
```

### 6.6 Tag 最佳实践

#### 6.6.1 命名规范

✅ **推荐**：
```
Ability.Movement.Jump          # 清晰的层级结构
State.Combat.InCombat          # 明确的分类
Cooldown.Skill.Fireball        # 描述性的名称
```

❌ **不推荐**：
```
Jump                           # 太宽泛，容易冲突
ability_jump                   # 使用下划线，不符合约定
Ability.Jump.Movement          # 层级顺序错误
```

#### 6.6.2 使用 Native Tags（C++）

**优势**：
- ✅ 性能更好（编译时解析）
- ✅ 类型安全（避免拼写错误）
- ✅ IDE 支持（自动补全）
- ✅ 便于重构

**实现**：

```cpp
// MyGameplayTags.h
namespace MyGameplayTags
{
    // 声明
    extern FGameplayTag Ability_Dash;
    extern FGameplayTag State_Dashing;
    extern FGameplayTag Cooldown_Dash;

    // 初始化函数
    void InitializeTags();
}

// MyGameplayTags.cpp
namespace MyGameplayTags
{
    // 定义
    FGameplayTag Ability_Dash;
    FGameplayTag State_Dashing;
    FGameplayTag Cooldown_Dash;

    void InitializeTags()
    {
        UGameplayTagsManager& Manager = UGameplayTagsManager::Get();

        Ability_Dash = Manager.AddNativeGameplayTag(
            FName("Ability.Movement.Dash"),
            FString("Dash ability"));

        State_Dashing = Manager.AddNativeGameplayTag(
            FName("State.Dashing"),
            FString("Dashing state"));

        Cooldown_Dash = Manager.AddNativeGameplayTag(
            FName("Cooldown.Ability.Dash"),
            FString("Dash cooldown"));
    }
}

// 使用
#include "MyGameplayTags.h"

if (ASC->HasMatchingGameplayTag(MyGameplayTags::State_Dashing))
{
    // ...
}
```

#### 6.6.3 避免过度使用 Tags

**问题**：每个小状态都创建 Tag

```cpp
// ❌ 过度使用
State.IsJumping
State.IsFalling
State.IsWalking
State.IsRunning
State.IsSwimming
State.IsFlying
// ... 太多了！
```

**解决方案**：使用属性或枚举

```cpp
// ✅ 更好的方式
// 在 AttributeSet 或组件中使用枚举
UENUM()
enum class EMovementState
{
    Walking,
    Running,
    Jumping,
    Falling,
    Swimming,
    Flying
};

UPROPERTY(Replicated)
EMovementState MovementState;

// 只为关键状态创建 Tag
State.InAir           # 跳跃/坠落
State.InWater         # 游泳
```

#### 6.6.4 Tag 文档化

在团队项目中，维护 Tag 文档非常重要：

```markdown
# Gameplay Tags Documentation

## Ability Tags

### Ability.Movement.Dash
- **描述**: 冲刺能力
- **用途**: 标识冲刺能力
- **相关**: State.Dashing, Cooldown.Ability.Dash

### Ability.Weapon.Fire
- **描述**: 射击能力
- **用途**: 标识武器射击能力
- **相关**: State.Firing, InputTag.Weapon.Fire

## State Tags

### State.Dead
- **描述**: 角色死亡状态
- **用途**: 阻止所有能力激活
- **添加时机**: 生命值归零时
- **移除时机**: 重生时

### State.Stunned
- **描述**: 角色被眩晕
- **用途**: 阻止移动和能力使用
- **持续时间**: 通常 1-3 秒
```

---

## 7. 实战案例：实现简单的技能系统

让我们通过一个完整的实战案例，整合前面学到的所有知识，实现一个包含耐力消耗、冷却、UI反馈的冲刺技能系统。

### 7.1 需求分析：冲刺技能

**功能需求**：
- 玩家按下 Shift 键触发冲刺
- 冲刺时角色快速向前移动
- 消耗 20 点耐力
- 冷却时间 3 秒
- 耐力不足时无法冲刺
- 冲刺时播放动画和特效
- UI 显示耐力和冷却状态

**技术需求**：
- Attribute: Stamina（耐力）
- Gameplay Ability: Dash（冲刺）
- Gameplay Effect: 消耗耐力、应用冷却
- Gameplay Tags: 状态管理
- 网络同步

### 7.2 创建 AttributeSet（耐力系统）

#### 7.2.1 创建 StaminaSet

参考前面 5.2 节的代码，创建 `UMyStaminaSet`。

#### 7.2.2 在 PlayerState 中添加 StaminaSet

```cpp
// MyPlayerState.h
UCLASS()
class AMyPlayerState : public ALyraPlayerState
{
    GENERATED_BODY()

public:
    AMyPlayerState();

    // 耐力属性集
    UPROPERTY()
    TObjectPtr<UMyStaminaSet> StaminaSet;
};

// MyPlayerState.cpp
AMyPlayerState::AMyPlayerState()
{
    // 创建 StaminaSet
    StaminaSet = CreateDefaultSubobject<UMyStaminaSet>(TEXT("StaminaSet"));
}
```

#### 7.2.3 创建耐力恢复效果

创建 Blueprint，父类：**GameplayEffect**，命名：`GE_StaminaRegen`

```
Duration Policy: Infinite (无限持续)
Period: 0.1 (每 0.1 秒执行一次)

Modifiers:
  ├─ Attribute: Stamina
  ├─ Operation: Add
  └─ Magnitude: Attribute Based
     ├─ Backing Attribute: Self.StaminaRegenRate
     ├─ Attribute Calculation Type: Attribute Base Value
     ├─ Coefficient: 0.1 (因为每 0.1 秒执行，所以是 RegenRate 的 1/10)
     └─ Pre/Post Additive: 0

Granted Tags:
  └─ Effect.StaminaRegen

Ongoing Tag Requirements:
  Required Tags: (留空)
  Ignore Tags:
    └─ State.Dashing (冲刺时停止恢复)
```

#### 7.2.4 在 PawnData 中应用耐力恢复

```cpp
// 在 PawnData 或 AbilitySet 中添加
Default Effects:
  └─ GE_StaminaRegen
```

### 7.3 实现冲刺 Ability

#### 7.3.1 创建 GA_Dash（C++）

参考前面 4.2 节的代码，创建 `UGA_Dash`。

#### 7.3.2 添加耐力检查

修改 `CanActivateAbility`：

```cpp
bool UGA_Dash::CanActivateAbility(...) const
{
    if (!Super::CanActivateAbility(...))
    {
        return false;
    }

    // 检查耐力是否足够
    const UMyStaminaSet* StaminaSet = ActorInfo->AbilitySystemComponent->GetSet<UMyStaminaSet>();
    if (StaminaSet)
    {
        const float StaminaCost = 20.0f;
        if (StaminaSet->GetStamina() < StaminaCost)
        {
            // 耐力不足
            return false;
        }
    }

    return true;
}
```

**更好的方式**：使用 Gameplay Effect 的 Cost 系统（下一节）

### 7.4 添加 Gameplay Effect（消耗耐力）

#### 7.4.1 创建消耗效果

创建 Blueprint，父类：**GameplayEffect**，命名：`GE_Cost_Dash`

```
Duration Policy: Instant

Modifiers:
  ├─ Attribute: Stamina
  ├─ Operation: Add
  └─ Magnitude: Scalable Float
     └─ Value: -20.0 (消耗 20 点)
```

#### 7.4.2 在 Ability 中引用

```cpp
// GA_Dash 的 Class Defaults 中设置
Cost Gameplay Effect Class: GE_Cost_Dash
```

现在 `CommitAbility()` 会自动：
1. 检查耐力是否足够（通过 CheckCost）
2. 扣除耐力（通过 CommitCost）

#### 7.4.3 添加消耗失败反馈

```cpp
void UGA_Dash::ActivateAbility(...)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        // 提交失败（通常是耐力不足）
        
        // 播放失败音效
        UGameplayStatics::PlaySound2D(GetWorld(), StaminaDepletedSound);
        
        // 显示 UI 提示
        if (ALyraPlayerController* PC = GetLyraPlayerControllerFromActorInfo())
        {
            PC->ClientShowMessage(NSLOCTEXT("Ability", "StaminaDepleted", "耐力不足！"));
        }
        
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 执行冲刺...
}
```

### 7.5 实现冷却机制

#### 7.5.1 创建冷却效果

创建 Blueprint，父类：**GameplayEffect**，命名：`GE_Cooldown_Dash`

```
Duration Policy: Has Duration
Duration Magnitude: Scalable Float
  └─ Value: 3.0 (冷却 3 秒)

Granted Tags:
  └─ Cooldown.Ability.Dash
```

#### 7.5.2 在 Ability 中引用

```cpp
// GA_Dash 的 Class Defaults 中设置
Cooldown Gameplay Effect Class: GE_Cooldown_Dash

// 在 Activation Blocked Tags 中添加
Activation Blocked Tags:
  └─ Cooldown.Ability.Dash (在冷却时不能激活)
```

现在 `CommitAbility()` 会自动：
1. 检查冷却是否结束（通过 CheckCooldown）
2. 应用冷却（通过 CommitCooldown）

#### 7.5.3 获取剩余冷却时间（用于 UI）

```cpp
float UGA_Dash::GetCooldownTimeRemaining() const
{
    if (const FGameplayAbilitySpec* Spec = GetCurrentAbilitySpec())
    {
        // 获取冷却 Tag
        FGameplayTagContainer CooldownTags;
        CooldownTags.AddTag(FGameplayTag::RequestGameplayTag("Cooldown.Ability.Dash"));

        // 查询剩余时间
        FGameplayEffectQuery Query;
        Query.OwningTagQuery = FGameplayTagQuery::MakeQuery_MatchAnyTags(CooldownTags);

        TArray<float> RemainingTimes = GetAbilitySystemComponentFromActorInfo()->GetActiveEffectsTimeRemaining(Query);

        if (RemainingTimes.Num() > 0)
        {
            // 返回最长的剩余时间
            float MaxTime = 0.0f;
            for (float Time : RemainingTimes)
            {
                if (Time > MaxTime)
                {
                    MaxTime = Time;
                }
            }
            return MaxTime;
        }
    }

    return 0.0f;
}
```

### 7.6 添加 UI 反馈

#### 7.6.1 创建 Stamina Widget

```cpp
// WBP_StaminaBar (UMG Widget)

// 进度条
UPROPERTY(meta = (BindWidget))
UProgressBar* StaminaBar;

// 文本
UPROPERTY(meta = (BindWidget))
UTextBlock* StaminaText;

// 更新 UI
void UpdateStamina(float CurrentStamina, float MaxStamina)
{
    if (StaminaBar)
    {
        StaminaBar->SetPercent(CurrentStamina / MaxStamina);
    }

    if (StaminaText)
    {
        StaminaText->SetText(FText::Format(
            INVTEXT("{0} / {1}"),
            FText::AsNumber(FMath::CeilToInt(CurrentStamina)),
            FText::AsNumber(FMath::CeilToInt(MaxStamina))));
    }
}
```

#### 7.6.2 监听属性变化

```cpp
// 在 HUD 或 PlayerController 中
void AMyHUD::BeginPlay()
{
    Super::BeginPlay();

    // 创建 Widget
    StaminaWidget = CreateWidget<UStaminaWidget>(GetOwningPlayerController(), StaminaWidgetClass);
    StaminaWidget->AddToViewport();

    // 绑定属性变化事件
    if (UAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        // 监听 Stamina 变化
        ASC->GetGameplayAttributeValueChangeDelegate(
            UMyStaminaSet::GetStaminaAttribute()
        ).AddUObject(this, &AMyHUD::OnStaminaChanged);
    }
}

void AMyHUD::OnStaminaChanged(const FOnAttributeChangeData& Data)
{
    if (StaminaWidget)
    {
        // 获取 MaxStamina
        float MaxStamina = 100.0f;
        if (UAbilitySystemComponent* ASC = GetAbilitySystemComponent())
        {
            const UMyStaminaSet* StaminaSet = ASC->GetSet<UMyStaminaSet>();
            if (StaminaSet)
            {
                MaxStamina = StaminaSet->GetMaxStamina();
            }
        }

        StaminaWidget->UpdateStamina(Data.NewValue, MaxStamina);
    }
}
```

#### 7.6.3 创建冷却指示器

```cpp
// WBP_DashCooldown (UMG Widget)

// 冷却遮罩（Image）
UPROPERTY(meta = (BindWidget))
UImage* CooldownOverlay;

// 冷却文本
UPROPERTY(meta = (BindWidget))
UTextBlock* CooldownText;

// Tick 更新
void UDashCooldownWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    Super::NativeTick(MyGeometry, InDeltaTime);

    // 获取冷却剩余时间
    float RemainingTime = 0.0f;
    if (UAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        // 查询冷却标签
        FGameplayTagContainer CooldownTags;
        CooldownTags.AddTag(FGameplayTag::RequestGameplayTag("Cooldown.Ability.Dash"));

        FGameplayEffectQuery Query;
        Query.OwningTagQuery = FGameplayTagQuery::MakeQuery_MatchAnyTags(CooldownTags);

        TArray<float> Times = ASC->GetActiveEffectsTimeRemaining(Query);
        if (Times.Num() > 0)
        {
            RemainingTime = Times[0];
        }
    }

    // 更新 UI
    if (RemainingTime > 0.0f)
    {
        CooldownOverlay->SetVisibility(ESlateVisibility::Visible);
        CooldownText->SetText(FText::AsNumber(FMath::CeilToInt(RemainingTime)));

        // 设置透明度（淡出效果）
        float Alpha = FMath::Clamp(RemainingTime / 3.0f, 0.0f, 0.7f);
        CooldownOverlay->SetColorAndOpacity(FLinearColor(0, 0, 0, Alpha));
    }
    else
    {
        CooldownOverlay->SetVisibility(ESlateVisibility::Hidden);
        CooldownText->SetText(FText::GetEmpty());
    }
}
```

### 7.7 网络同步测试

#### 7.7.1 测试清单

**服务器权威验证**：
- [ ] 客户端按下冲刺键
- [ ] 服务器收到激活请求
- [ ] 服务器验证消耗和冷却
- [ ] 服务器执行冲刺
- [ ] 其他客户端看到冲刺效果

**属性复制**：
- [ ] 耐力消耗在所有客户端同步
- [ ] 耐力恢复在所有客户端同步
- [ ] UI 显示正确

**Gameplay Effect 复制**：
- [ ] 冷却效果在所有客户端显示
- [ ] 效果结束后正确移除

#### 7.7.2 测试方法

**在编辑器中测试**：

1. 打开 **Play Settings**
2. 设置：
   - Number of Players: 2
   - Net Mode: Play As Listen Server
   - Run Dedicated Server: false

3. 运行游戏，观察：
   - 客户端 1 冲刺，客户端 2 是否看到
   - 耐力是否同步
   - 冷却是否同步

**使用 Network Emulation**：

```cpp
// 在 DefaultEngine.ini 中添加
[/Script/Engine.Player]
ConfiguredInternetSpeed=10000
ConfiguredLanSpeed=20000

// 模拟延迟
net PktLag=100        // 100ms 延迟
net PktLoss=5         // 5% 丢包率
```

**调试日志**：

```cpp
void UGA_Dash::ActivateAbility(...)
{
    UE_LOG(LogTemp, Log, TEXT("[%s] Dash activated, Authority=%d, Predicted=%d"),
        HasAuthority(ActorInfo, &ActivationInfo) ? TEXT("Server") : TEXT("Client"),
        HasAuthority(ActorInfo, &ActivationInfo),
        ScopedPredictionKey.IsValidKey());

    // ...
}
```

#### 7.7.3 常见网络问题

**问题 1：客户端预测失败，出现回滚**

**原因**：
- 客户端和服务器的验证逻辑不一致
- 客户端没有正确复制属性

**解决方案**：
```cpp
// 确保 CanActivateAbility 在客户端和服务器上逻辑相同
// 确保所有相关属性都正确复制
```

**问题 2：Gameplay Effect 没有复制**

**原因**：
- ASC 的复制模式设置错误
- Effect 的复制策略不正确

**解决方案**：
```cpp
// 在 PlayerState 构造函数中
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
```

**问题 3：动画不同步**

**原因**：
- 动画只在客户端播放
- 需要使用 Replicated Montage

**解决方案**：
```cpp
// 使用 ASC 的 Montage 播放函数（自动复制）
if (UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo())
{
    FGameplayAbilityActorInfo* ActorInfo = GetActorInfo();
    ASC->CurrentMontagePlay(DashMontage);
}
```

---

## 8. 最佳实践与常见问题

### 8.1 GAS 最佳实践

#### 8.1.1 架构设计

**1. 使用 PlayerState 存储 ASC（多人游戏）**

✅ **推荐**：
```cpp
// ASC 在 PlayerState 中
// 优势：角色死亡重生时属性保持
AMyPlayerState::AMyPlayerState()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(...);
}
```

❌ **不推荐**（多人游戏）：
```cpp
// ASC 在 Character 中
// 问题：重生时属性丢失
AMyCharacter::AMyCharacter()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(...);
}
```

**2. 拆分 AttributeSet**

✅ **推荐**：
```cpp
// 按功能拆分
HealthSet    // 生命值相关
CombatSet    // 战斗属性
MovementSet  // 移动属性
```

❌ **不推荐**：
```cpp
// 所有属性放在一个 Set 中
AllAttributesSet // 难以维护，复制效率低
```

**3. 使用 AbilitySet 管理能力**

✅ **推荐**：
```cpp
// 使用数据资产批量授予
AbilitySet_Hero
AbilitySet_Weapon_Rifle
```

❌ **不推荐**：
```cpp
// 硬编码授予能力
GiveAbility(UGA_Jump::StaticClass());
GiveAbility(UGA_Sprint::StaticClass());
// ...
```

#### 8.1.2 网络同步

**1. 选择合适的复制模式**

```cpp
// 竞技游戏（只需要自己的信息）
SetReplicationMode(EGameplayEffectReplicationMode::Minimal);

// 多人合作游戏（需要看到队友的 Buff）
SetReplicationMode(EGameplayEffectReplicationMode::Mixed); // ← Lyra 使用

// 观战系统
SetReplicationMode(EGameplayEffectReplicationMode::Full);
```

**2. 正确使用预测**

```cpp
// 需要快速响应的能力使用预测
NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
// 例如：跳跃、射击、冲刺

// 重要的、不能作弊的能力使用服务器执行
NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerOnly;
// 例如：购买物品、GM 命令
```

**3. 使用 Gameplay Cues 播放特效**

```cpp
// ✅ 使用 Gameplay Cue（自动复制）
FGameplayCueParameters CueParams;
CueParams.Location = GetActorLocation();
ASC->ExecuteGameplayCue(TAG_GameplayCue_Dash, CueParams);

// ❌ 直接生成粒子（不会复制）
UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), DashEffect, GetActorLocation());
```

#### 8.1.3 性能优化

**1. 使用 NonInstanced Ability（当适合时）**

```cpp
// 简单的、无状态的能力
InstancingPolicy = EGameplayAbilityInstancingPolicy::NonInstanced;
// 例如：简单的跳跃、快速攻击

// 复杂的、需要保存状态的能力
InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerExecution;
// 例如：技能连招、充能技能
```

**2. 限制 Gameplay Effect 的数量**

```cpp
// ❌ 避免每帧应用 Instant Effect
void Tick(float DeltaTime)
{
    // 性能杀手！
    ApplyGameplayEffectToSelf(DamageEffect);
}

// ✅ 使用 Periodic Effect
// 在初始化时应用一次，自动每帧执行
ApplyGameplayEffectToSelf(PeriodicDamageEffect); // Period = 1.0s
```

**3. 使用 Attribute Clamp 而不是每帧检查**

```cpp
// ✅ 在 AttributeSet 中 Clamp
void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    if (Attribute == GetHealthAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
    }
}

// ❌ 每帧检查和修正
void Tick(float DeltaTime)
{
    if (GetHealth() > GetMaxHealth())
    {
        SetHealth(GetMaxHealth());
    }
}
```

#### 8.1.4 代码组织

**1. 使用 Native Gameplay Tags**

```cpp
// ✅ 集中管理
namespace MyGameplayTags
{
    extern FGameplayTag Ability_Jump;
    extern FGameplayTag State_Dead;
}

// ❌ 到处 RequestGameplayTag
FGameplayTag::RequestGameplayTag("Ability.Jump"); // 容易拼写错误
```

**2. 创建自定义的 Ability 基类**

```cpp
// MyGameplayAbility_Base.h
UCLASS()
class UMyGameplayAbility_Base : public UGameplayAbility
{
    // 添加项目通用的功能
    UFUNCTION(BlueprintCallable)
    AMyPlayerController* GetMyPlayerController() const;

    UFUNCTION(BlueprintCallable)
    AMyCharacter* GetMyCharacter() const;

    // 自定义的消耗检查
    virtual bool CheckCustomCost() const;
};

// 所有项目能力继承这个基类
UCLASS()
class UGA_MyJump : public UMyGameplayAbility_Base
{
    // ...
};
```

### 8.2 常见错误与解决方案

#### 8.2.1 ASC 未正确初始化

**错误现象**：
- 调用 `TryActivateAbility` 时崩溃
- Ability 无法激活
- 属性不更新

**原因**：
- 忘记调用 `InitAbilityActorInfo`

**解决方案**：
```cpp
void AMyCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);

    if (AMyPlayerState* PS = GetPlayerState<AMyPlayerState>())
    {
        AbilitySystemComponent = PS->GetAbilitySystemComponent();
        
        // ← 关键！必须调用
        AbilitySystemComponent->InitAbilityActorInfo(PS, this);
    }
}
```

#### 8.2.2 属性不复制

**错误现象**：
- 服务器上属性正确，客户端不更新
- UI 显示错误

**原因**：
- 忘记注册属性复制
- 没有实现 `OnRep` 回调

**解决方案**：
```cpp
void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // ← 必须注册
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
}

UFUNCTION()
void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
    // ← 必须调用这个宏
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldHealth);
}
```

#### 8.2.3 Ability 激活失败但没有提示

**错误现象**：
- 按键没有反应
- 没有任何错误信息

**原因**：
- `CanActivateAbility` 返回 false
- 被 Blocking Tags 阻止

**调试方法**：
```cpp
// 1. 启用 GAS 日志
// DefaultEngine.ini
[Core.Log]
LogAbilitySystem=Verbose

// 2. 在控制台显示调试信息
showdebug abilitysystem

// 3. 重写激活失败回调
void UMyGameplayAbility::OnAbilityFailedToActivate(const FGameplayTagContainer& FailedReason)
{
    UE_LOG(LogTemp, Warning, TEXT("Ability %s failed to activate. Reason: %s"),
        *GetName(),
        *FailedReason.ToString());

    // 显示给玩家
    if (APlayerController* PC = GetPlayerController())
    {
        PC->ClientMessage(FString::Printf(TEXT("无法使用技能：%s"), *FailedReason.ToString()));
    }
}
```

#### 8.2.4 Gameplay Effect 立即结束

**错误现象**：
- Duration Effect 应用后立即消失
- Buff 不生效

**原因**：
- Duration Magnitude 设置错误
- Ongoing Tag Requirements 不满足

**解决方案**：
```cpp
// 检查 Effect 配置
Duration Policy: Has Duration
Duration Magnitude: Scalable Float = 10.0 (确保 > 0)

// 检查 Ongoing Tags
Ongoing Required Tags: (检查目标是否拥有这些 Tags)
Ongoing Ignored Tags: (检查目标是否拥有这些 Tags)
```

#### 8.2.5 MetaAttribute 不起作用

**错误现象**：
- 应用 Damage Effect，但 Health 不减少

**原因**：
- 忘记在 `PostGameplayEffectExecute` 中处理

**解决方案**：
```cpp
void UMyHealthSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    // ← 必须手动处理 MetaAttribute
    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        const float DamageValue = GetDamage();
        SetDamage(0.0f); // 清零

        // 应用到 Health
        SetHealth(FMath::Max(GetHealth() - DamageValue, 0.0f));
    }
}
```

### 8.3 性能优化建议

#### 8.3.1 减少 Ability 数量

**问题**：角色拥有数百个 Ability，影响性能

**解决方案**：
- 使用参数化的 Ability（而不是为每个变体创建新类）
- 使用 Gameplay Events 触发，而不是授予大量 Ability

```cpp
// ❌ 为每种武器创建一个 Ability
GA_Rifle_Fire
GA_Pistol_Fire
GA_Shotgun_Fire
// ...

// ✅ 使用一个参数化的 Ability
GA_Weapon_Fire
  ├─ 从 SourceObject（武器）读取配置
  └─ 根据武器类型调整行为
```

#### 8.3.2 使用 Ability Batching

**问题**：频繁激活 Ability 导致大量 RPC

**解决方案**：
```cpp
// GAS 支持批量发送 Ability 激活请求
AbilitySystemComponent->SetUseServerAllowActivateCooldown(true);
AbilitySystemComponent->ServerAllowActivateCooldown = 0.05f; // 50ms 窗口
```

#### 8.3.3 优化 Gameplay Effect 查询

**问题**：频繁查询 Active Effects 影响性能

**解决方案**：
```cpp
// ❌ 每帧查询
void Tick(float DeltaTime)
{
    TArray<FActiveGameplayEffectHandle> Effects = ASC->GetActiveEffects(...);
}

// ✅ 使用回调监听
void BeginPlay()
{
    // 监听 Effect 添加/移除
    ASC->OnActiveGameplayEffectAddedDelegateToSelf.AddUObject(this, &UMyComponent::OnEffectAdded);
    ASC->OnAnyGameplayEffectRemovedDelegate().AddUObject(this, &UMyComponent::OnEffectRemoved);
}
```

#### 8.3.4 限制 Gameplay Cue 数量

**问题**：大量 Gameplay Cue 特效导致性能下降

**解决方案**：
```cpp
// 使用对象池
UCLASS()
class UMyGameplayCueManager : public UGameplayCueManager
{
    virtual AGameplayCueNotify_Actor* GetInstancedCueActor(...) override
    {
        // 从对象池中获取，而不是每次 Spawn
        return ObjectPool.GetOrCreate();
    }
};

// 限制同时存在的 Cue 数量
MaxActiveGameplayCues = 50;
```

### 8.4 调试技巧

#### 8.4.1 控制台命令

```cpp
// 显示 GAS 调试信息
showdebug abilitysystem

// 切换显示类别
AbilitySystem.Debug.NextCategory

// 显示特定 Actor 的 ASC
AbilitySystem.Debug.ShowDebugForActor <ActorName>

// 列出所有 Gameplay Tags
GameplayTags.PrintReplicationIndices

// 模拟网络延迟
net PktLag=100
net PktLoss=5
```

#### 8.4.2 可视化调试

在编辑器中可视化 GAS 状态：

```cpp
// MyAbilitySystemComponent.h
#if WITH_EDITOR
virtual void DrawDebug(class AHUD* HUD, class UCanvas* Canvas) override;
#endif

// MyAbilitySystemComponent.cpp
#if WITH_EDITOR
void UMyAbilitySystemComponent::DrawDebug(AHUD* HUD, UCanvas* Canvas)
{
    Super::DrawDebug(HUD, Canvas);

    // 绘制当前激活的 Abilities
    FString DebugText = "Active Abilities:\n";
    for (const FGameplayAbilitySpec& Spec : GetActivatableAbilities())
    {
        if (Spec.IsActive())
        {
            DebugText += FString::Printf(TEXT("- %s\n"), *Spec.Ability->GetName());
        }
    }

    // 绘制 Owned Tags
    DebugText += "\nOwned Tags:\n";
    FGameplayTagContainer OwnedTags;
    GetOwnedGameplayTags(OwnedTags);
    for (const FGameplayTag& Tag : OwnedTags)
    {
        DebugText += FString::Printf(TEXT("- %s\n"), *Tag.ToString());
    }

    // 显示文本
    Canvas->SetDrawColor(FColor::White);
    Canvas->DrawText(GEngine->GetSmallFont(), DebugText, 10, 100);
}
#endif
```

#### 8.4.3 断点和日志

在关键位置添加日志：

```cpp
// Ability 激活
UE_LOG(LogAbilitySystem, Log, TEXT("[%s] %s activated"),
    HasAuthority() ? TEXT("Server") : TEXT("Client"),
    *GetName());

// Gameplay Effect 应用
UE_LOG(LogGameplayEffects, Log, TEXT("Applied GE %s to %s, Magnitude=%.2f"),
    *EffectSpec.Def->GetName(),
    *Target->GetOwnerActor()->GetName(),
    Magnitude);

// 属性变化
UE_LOG(LogAbilitySystem, Log, TEXT("Attribute %s changed: %.2f -> %.2f"),
    *Attribute.GetName(),
    OldValue,
    NewValue);
```

### 8.5 推荐学习资源

#### 8.5.1 官方文档

- **Unreal Engine Gameplay Ability System**
  - https://docs.unrealengine.com/5.3/en-US/gameplay-ability-system-for-unreal-engine/

- **Lyra Sample Game Documentation**
  - https://docs.unrealengine.com/5.3/en-US/lyra-sample-game-in-unreal-engine/

#### 8.5.2 社区资源

- **GASDocumentation (tranek)**
  - https://github.com/tranek/GASDocumentation
  - 最全面的 GAS 社区文档

- **GAS Companion (Unreal Engine Marketplace)**
  - 提供 GAS UI 组件和调试工具

#### 8.5.3 视频教程

- **Epic Games 官方 Lyra 直播回放**
  - https://www.youtube.com/c/UnrealEngine

- **YouTube - Gameplay Ability System Tutorials**
  - 搜索 "UE5 GAS Tutorial"

#### 8.5.4 示例项目

- **Lyra Starter Game** (Epic Games)
  - 引擎自带，最权威的参考

- **Action RPG** (Epic Games)
  - 展示了 GAS 在 RPG 中的应用

- **GASShooter** (tranek)
  - https://github.com/tranek/GASShooter
  - 社区制作的射击游戏示例

---

## 9. 总结与展望

### 9.1 本文回顾

在这篇文章中，我们深入学习了 **Gameplay Ability System (GAS)** 的基础知识：

**核心概念**：
- ✅ GAS 的五大组件：ASC、Ability、Attribute、Effect、Tag
- ✅ 组件之间的关系和数据流
- ✅ 网络同步机制和预测系统

**Lyra 集成**：
- ✅ LyraAbilitySystemComponent 的扩展功能
- ✅ LyraGameplayAbility 的激活策略
- ✅ AbilitySet 系统和 Experience 整合
- ✅ Lyra 的初始化流程

**实践技能**：
- ✅ 创建自定义 Ability（蓝图和 C++）
- ✅ 实现 AttributeSet 和属性管理
- ✅ 使用 Gameplay Tags 进行状态管理
- ✅ 完整的技能系统实战案例

**最佳实践**：
- ✅ 架构设计原则
- ✅ 性能优化技巧
- ✅ 常见问题解决方案
- ✅ 调试方法和工具

### 9.2 GAS 的强大之处

GAS 之所以被广泛采用，是因为它解决了游戏开发中的核心痛点：

**1. 统一的框架**
- 不需要为每个项目重新发明轮子
- Epic Games 内部多个 AAA 项目验证

**2. 网络友好**
- 自动处理客户端预测和回滚
- 减少 RPC 数量，优化带宽

**3. 数据驱动**
- 设计师可以通过蓝图配置大部分内容
- 快速迭代，无需修改代码

**4. 可扩展性**
- 模块化设计，易于添加新功能
- 支持复杂的游戏逻辑

### 9.3 下一步学习方向

本文只是 GAS 的入门，还有很多高级主题值得深入：

**下一篇文章预告**：《GAS 进阶：Attributes、Effects 与 Tags》

将涵盖：
- Attribute 的高级应用（属性修改器、快照）
- Gameplay Effect 的复杂配置（Stacking、Conditions）
- Custom Calculation Class（自定义伤害计算）
- Gameplay Cues（特效和音效系统）
- Ability Tasks（异步操作和状态机）

**后续文章**：《GAS 高级：能力系统的实战应用》

将涵盖：
- 复杂技能系统（技能树、技能连招）
- 装备系统与 GAS 的整合
- AI 使用 GAS
- 性能分析和优化
- 大型项目的架构设计

### 9.4 实践建议

**学习 GAS 的最佳方式是实践**：

1. **从 Lyra 开始**
   - 研究 Lyra 的能力实现
   - 修改现有能力，观察效果
   - 添加新的能力到游戏中

2. **创建自己的项目**
   - 从简单的能力开始（跳跃、冲刺）
   - 逐步添加复杂功能（技能系统、Buff/Debuff）
   - 在多人环境下测试

3. **阅读源代码**
   - GAS 插件的源代码是最好的教材
   - 阅读 Epic 的示例项目（Lyra、ActionRPG）
   - 参考社区项目（GASShooter）

4. **参与社区**
   - Unreal Slackers Discord
   - Unreal Engine Forums
   - Stack Overflow

### 9.5 结语

Gameplay Ability System 是现代 Unreal Engine 项目的标准配置，尤其是在多人游戏中。虽然学习曲线陡峭，但一旦掌握，你会发现它极大地提升了开发效率和代码质量。

Lyra 示例项目展示了 GAS 在大型项目中的最佳实践，值得深入研究。希望本文能为你打开 GAS 的大门，在后续的文章中，我们将探索更多高级特性。

**记住**：GAS 是一个强大的工具，但也只是工具。最重要的是理解游戏设计的核心逻辑，然后用 GAS 来实现它。不要为了使用 GAS 而使用 GAS，而是让 GAS 为你的游戏服务。

**下一篇文章见！让我们继续深入 GAS 的高级特性。**

---

**相关文章**：

- 上一篇：[数据驱动设计：Data Assets 与配置管理](05-data-driven-design.md)
- 下一篇：[GAS 进阶：Attributes、Effects 与 Tags](07-gas-advanced.md)
- 返回：[目录](../README.md)

---

**作者**: Lyra Tutorial Series  
**最后更新**: 2026-02-12  
**版本**: 1.0  
**字数**: ~20,000 字
