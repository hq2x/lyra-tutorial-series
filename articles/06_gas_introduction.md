# UE5 Lyra 系列教程（六）：Gameplay Ability System (GAS) 入门

> **作者**: lobsterchen  
> **创建时间**: 2025-02-12  
> **系列**: UE5 Lyra 深度解析  
> **难度**: ⭐⭐⭐⭐ 进阶  
> **预计阅读时间**: 30 分钟

---

## 📚 目录

- [GAS 是什么？为什么需要它？](#gas-是什么为什么需要它)
- [GAS 核心概念速览](#gas-核心概念速览)
- [Lyra 中的 GAS 架构](#lyra-中的-gas-架构)
- [Ability System Component 详解](#ability-system-component-详解)
- [Ability Set 的设计与使用](#ability-set-的设计与使用)
- [实战：实现跳跃技能](#实战实现跳跃技能)
- [实战：实现冲刺系统](#实战实现冲刺系统)

---

## 🤔 GAS 是什么？为什么需要它？

### 传统技能系统的问题

假设你要实现一个"火球术"技能，传统做法：

```cpp
// ❌ 传统方式
class AMyCharacter : public ACharacter
{
    void CastFireball()
    {
        // 1. 检查能否释放
        if (Mana < 50) return;
        if (IsStunned) return;
        if (IsSilenced) return;
        
        // 2. 消耗资源
        Mana -= 50;
        
        // 3. 播放动画
        PlayMontage(FireballCastMontage);
        
        // 4. 生成火球
        SpawnFireballProjectile();
        
        // 5. 进入冷却
        FireballCooldownRemaining = 5.0f;
    }
};
```

**问题一大堆**：
- 🚫 **网络同步困难**：客户端预测、服务器验证要手写
- 🚫 **buff/debuff 难实现**：怎么处理"沉默"、"技能加速"等效果？
- 🚫 **技能打断**：被眩晕时如何取消技能？
- 🚫 **combo 系统**：技能之间的连招怎么管理？
- 🚫 **权限管理**：谁能释放这个技能？如何阻止作弊？

### GAS 的解决方案

**Gameplay Ability System (GAS)** 是 Epic 官方的技能系统框架，解决了所有这些问题。

```cpp
// ✅ GAS 方式
UCLASS()
class UGA_Fireball : public ULyraGameplayAbility
{
    virtual bool CanActivateAbility(...) override
    {
        // GAS 自动检查：
        // - 是否有足够的资源（Mana）
        // - 是否被阻止（Stunned/Silenced）
        // - 是否在冷却中
        return Super::CanActivateAbility(...);
    }
    
    virtual void ActivateAbility(...) override
    {
        // 只需关注技能逻辑
        PlayMontage();
        SpawnFireball();
        CommitAbility();  // 自动处理消耗、冷却、网络同步
    }
};
```

**GAS 提供的核心功能**：
- ✅ **网络同步**：客户端预测 + 服务器验证，自动处理
- ✅ **属性系统**：Health、Mana、Stamina 等，支持监听变化
- ✅ **效果系统**：Buff、Debuff、DOT、HOT 统一管理
- ✅ **标签系统**：用于控制技能激活条件和状态
- ✅ **视觉反馈**：Gameplay Cues 统一管理特效和音效

---

## 🧩 GAS 核心概念速览

在深入 Lyra 实现之前，先理解 GAS 的 5 大核心概念：

### 1. Ability System Component (ASC)

**核心组件**，挂载在 Actor 上，管理所有 GAS 功能。

```cpp
UCLASS()
class ULyraAbilitySystemComponent : public UAbilitySystemComponent
{
    // 拥有的技能列表
    TArray<FGameplayAbilitySpecHandle> GrantedAbilities;
    
    // 当前激活的技能
    TArray<FGameplayAbilitySpec*> ActiveAbilities;
    
    // 当前生效的 Gameplay Effects
    FActiveGameplayEffectsContainer ActiveGameplayEffects;
    
    // 当前的 Gameplay Tags
    FGameplayTagCountContainer GameplayTagCountContainer;
};
```

### 2. Gameplay Ability (GA)

**技能的具体实现**，定义技能做什么。

```cpp
UCLASS()
class ULyraGameplayAbility : public UGameplayAbility
{
    // 技能何时可以激活？
    virtual bool CanActivateAbility(...);
    
    // 技能激活时执行
    virtual void ActivateAbility(...);
    
    // 技能结束时执行
    virtual void EndAbility(...);
    
    // 技能被取消时执行
    virtual void CancelAbility(...);
};
```

### 3. Attribute Set (AS)

**属性集合**，定义 Health、Mana、Speed 等数值。

```cpp
UCLASS()
class ULyraHealthSet : public UAttributeSet
{
    UPROPERTY()
    FGameplayAttributeData Health;  // 当前生命值
    
    UPROPERTY()
    FGameplayAttributeData MaxHealth;  // 最大生命值
    
    // 属性变化时的逻辑
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue);
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data);
};
```

### 4. Gameplay Effect (GE)

**效果**，用于修改属性、施加 Buff/Debuff。

```
GE_DamageInstant (瞬时效果)
    ├── Modifiers:
    │   └── Health: -50 (立即扣血)
    └── Execution: DamageExecution (伤害计算)

GE_SpeedBuff (持续效果，10秒)
    ├── Duration: 10.0s
    ├── Modifiers:
    │   └── MovementSpeed: +50%
    └── GrantedTags: Status.Buff.Speed

GE_Burning (DOT，每秒触发)
    ├── Duration: 5.0s
    ├── Period: 1.0s (每秒)
    └── Modifiers:
        └── Health: -10 (每秒扣10血)
```

### 5. Gameplay Tags

**标签系统**，用于控制技能激活和状态管理。

```
Ability.Jump (技能标签)
Ability.Sprint
Ability.Attack

Status.Stunned (状态标签)
Status.Silenced
Status.Invincible

Trigger.Hit (事件标签)
Trigger.Death
```

---

## 🏗️ Lyra 中的 GAS 架构

### 组件层次

```
ALyraCharacter (Pawn)
    ├── ULyraAbilitySystemComponent (ASC)
    │   ├── Attribute Sets
    │   │   ├── ULyraHealthSet (生命值)
    │   │   ├── ULyraCombatSet (战斗属性)
    │   │   └── ULyraMovementSet (移动属性)
    │   ├── Granted Abilities
    │   │   ├── GA_Jump (跳跃)
    │   │   ├── GA_Sprint (冲刺)
    │   │   └── GA_Shoot (射击)
    │   └── Active Effects
    │       ├── GE_HealthRegen (生命回复)
    │       └── GE_SpeedBuff (速度buff)
    └── ULyraPawnExtensionComponent
        └── 负责初始化 ASC
```

### 初始化流程

```cpp
// LyraCharacter.cpp

void ALyraCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    
    // 服务器端：初始化 ASC
    ALyraPlayerState* PS = GetPlayerState<ALyraPlayerState>();
    if (PS)
    {
        // ASC 挂载在 PlayerState 上（多人游戏最佳实践）
        AbilitySystemComponent = PS->GetLyraAbilitySystemComponent();
        
        // 初始化 ASC
        AbilitySystemComponent->InitAbilityActorInfo(PS, this);
        
        // 赋予初始技能
        GrantStarterAbilities();
    }
}

void ALyraCharacter::OnRep_PlayerState()
{
    Super::OnRep_PlayerState();
    
    // 客户端：同步 ASC
    ALyraPlayerState* PS = GetPlayerState<ALyraPlayerState>();
    if (PS)
    {
        AbilitySystemComponent = PS->GetLyraAbilitySystemComponent();
        AbilitySystemComponent->InitAbilityActorInfo(PS, this);
    }
}
```

**为什么 ASC 放在 PlayerState 上？**

| 放置位置 | 优点 | 缺点 | 适用场景 |
|---------|------|------|---------|
| **Character** | 简单直接 | 角色死亡后 ASC 丢失 | 单机游戏、PvE |
| **PlayerState** | 角色死亡后 ASC 保留 | 稍复杂 | 多人游戏、PvP |

Lyra 选择 PlayerState，因为：
- 角色死亡重生后，技能和属性不丢失
- PlayerState 在网络中自动同步
- 符合 Epic 的最佳实践

---

## ⚙️ Ability System Component 详解

### LyraAbilitySystemComponent 的扩展

Lyra 对 ASC 做了几个关键扩展：

```cpp
// LyraAbilitySystemComponent.h

UCLASS()
class ULyraAbilitySystemComponent : public UAbilitySystemComponent
{
    GENERATED_BODY()

public:
    // ========== Ability 管理 ==========
    
    // 通过 Ability Set 批量赋予技能
    void GrantAbilitySet(const ULyraAbilitySet* AbilitySet, 
                         FLyraAbilitySet_GrantedHandles& OutGrantedHandles);
    
    // 批量移除技能
    void RemoveAbilitySet(const FLyraAbilitySet_GrantedHandles& GrantedHandles);
    
    // ========== 输入绑定 ==========
    
    // 绑定输入到技能
    void AbilityInputTagPressed(const FGameplayTag& InputTag);
    void AbilityInputTagReleased(const FGameplayTag& InputTag);
    
    // ========== 动画通知 ==========
    
    // 在动画中触发技能事件
    UFUNCTION(BlueprintCallable, Category="Lyra|Ability")
    void AbilityLocalInputPressed(int32 InputID);
    
    // ========== 调试 ==========
    
    // 显示当前激活的技能和效果
    void DebugPrintActiveAbilities();

protected:
    // 输入标签 -> 技能的映射
    TMap<FGameplayTag, FGameplayAbilitySpecHandle> InputTagToAbilityMap;
};
```

### 技能激活流程

```
1. 玩家按下按键
    ↓
2. Input Component 捕获
    ↓
3. LyraHeroComponent::AbilityInputTagPressed(InputTag)
    ↓
4. ASC::AbilityInputTagPressed(InputTag)
    ↓
5. 查找 InputTag 对应的技能
    ↓
6. ASC::TryActivateAbility(AbilitySpec)
    ↓
7. UGameplayAbility::CanActivateAbility()
    检查：
    - 是否在冷却中？
    - 是否有足够的资源（Mana/Stamina）？
    - 是否被阻止（Stunned/Silenced）？
    ↓
8. UGameplayAbility::ActivateAbility()
    执行技能逻辑
    ↓
9. UGameplayAbility::CommitAbility()
    - 消耗资源
    - 应用冷却
    - 网络同步
    ↓
10. UGameplayAbility::EndAbility()
    技能结束
```

---

## 📦 Ability Set 的设计与使用

### 什么是 Ability Set？

**Ability Set** 是一组技能的集合，用于批量赋予技能。

```
AbilitySet_ShooterHero (射击英雄技能集)
    ├── GA_Jump (跳跃)
    ├── GA_Sprint (冲刺)
    ├── GA_Shoot (射击)
    ├── GA_Reload (换弹)
    ├── GA_Melee (近战)
    └── GA_Aim (瞄准)
```

**为什么需要 Ability Set？**
- 批量管理：一次赋予/移除一组技能
- 数据驱动：在 Data Asset 中配置，无需写代码
- 复用性高：同一个 Set 可以用在不同角色上

### Ability Set 数据结构

```cpp
// LyraAbilitySet.h

USTRUCT(BlueprintType)
struct FLyraAbilitySet_GameplayAbility
{
    GENERATED_BODY()

    // 技能类
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<ULyraGameplayAbility> Ability = nullptr;
    
    // 技能等级
    UPROPERTY(EditDefaultsOnly)
    int32 AbilityLevel = 1;
    
    // 输入标签（用于绑定按键）
    UPROPERTY(EditDefaultsOnly, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};

USTRUCT(BlueprintType)
struct FLyraAbilitySet_GameplayEffect
{
    GENERATED_BODY()

    // 效果类
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

    // 属性集类
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UAttributeSet> AttributeSet;
    
    // 初始化数据表（可选）
    UPROPERTY(EditDefaultsOnly)
    UDataTable* InitializationData = nullptr;
};

UCLASS()
class ULyraAbilitySet : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 要赋予的技能列表
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Abilities")
    TArray<FLyraAbilitySet_GameplayAbility> GrantedGameplayAbilities;
    
    // 要应用的效果列表
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Effects")
    TArray<FLyraAbilitySet_GameplayEffect> GrantedGameplayEffects;
    
    // 要添加的属性集列表
    UPROPERTY(EditDefaultsOnly, Category = "Attribute Sets")
    TArray<FLyraAbilitySet_AttributeSet> GrantedAttributes;
};
```

### 赋予 Ability Set

```cpp
// LyraAbilitySystemComponent.cpp

void ULyraAbilitySystemComponent::GrantAbilitySet(
    const ULyraAbilitySet* AbilitySet,
    FLyraAbilitySet_GrantedHandles& OutGrantedHandles)
{
    if (!AbilitySet)
    {
        return;
    }
    
    // 1. 赋予技能
    for (const FLyraAbilitySet_GameplayAbility& AbilityToGrant : AbilitySet->GrantedGameplayAbilities)
    {
        if (!IsValid(AbilityToGrant.Ability))
        {
            continue;
        }
        
        FGameplayAbilitySpec AbilitySpec(
            AbilityToGrant.Ability,
            AbilityToGrant.AbilityLevel,
            INDEX_NONE,
            this
        );
        
        // 赋予技能并记录 Handle
        FGameplayAbilitySpecHandle AbilityHandle = GiveAbility(AbilitySpec);
        OutGrantedHandles.AbilityHandles.Add(AbilityHandle);
        
        // 绑定输入标签
        if (AbilityToGrant.InputTag.IsValid())
        {
            InputTagToAbilityMap.Add(AbilityToGrant.InputTag, AbilityHandle);
        }
    }
    
    // 2. 应用效果
    for (const FLyraAbilitySet_GameplayEffect& EffectToGrant : AbilitySet->GrantedGameplayEffects)
    {
        if (!IsValid(EffectToGrant.GameplayEffect))
        {
            continue;
        }
        
        FGameplayEffectContextHandle EffectContext = MakeEffectContext();
        FGameplayEffectSpecHandle EffectSpec = MakeOutgoingSpec(
            EffectToGrant.GameplayEffect,
            EffectToGrant.EffectLevel,
            EffectContext
        );
        
        FActiveGameplayEffectHandle EffectHandle = ApplyGameplayEffectSpecToSelf(*EffectSpec.Data.Get());
        OutGrantedHandles.EffectHandles.Add(EffectHandle);
    }
    
    // 3. 添加属性集
    for (const FLyraAbilitySet_AttributeSet& AttributeSetToGrant : AbilitySet->GrantedAttributes)
    {
        if (!IsValid(AttributeSetToGrant.AttributeSet))
        {
            continue;
        }
        
        UAttributeSet* NewSet = NewObject<UAttributeSet>(this, AttributeSetToGrant.AttributeSet);
        AddAttributeSetSubobject(NewSet);
        OutGrantedHandles.AttributeSets.Add(NewSet);
        
        // 如果有初始化数据表，应用初始值
        if (AttributeSetToGrant.InitializationData)
        {
            InitStats(AttributeSetToGrant.AttributeSet, AttributeSetToGrant.InitializationData);
        }
    }
}
```

---

## 🦘 实战：实现跳跃技能

### 需求分析

- 🎯 按下空格键触发跳跃
- ⚡ 消耗体力值（Stamina）
- ❄️ 1秒冷却
- 🚫 在空中或疲劳时无法跳跃

### Step 1: 创建 Gameplay Ability

```cpp
// GA_Jump.h

#pragma once

#include "LyraGameplayAbility.h"
#include "GA_Jump.generated.h"

UCLASS()
class UGA_Jump : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Jump();

protected:
    virtual bool CanActivateAbility(...) const override;
    virtual void ActivateAbility(...) override;
    virtual void EndAbility(...) override;

private:
    UPROPERTY(EditDefaultsOnly, Category="Jump")
    float StaminaCost = 10.0f;
    
    UPROPERTY(EditDefaultsOnly, Category="Jump")
    float CooldownDuration = 1.0f;
};
```

```cpp
// GA_Jump.cpp

UGA_Jump::UGA_Jump()
{
    // 设置网络策略
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    
    // 设置技能标签
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Ability.Jump")));
    
    // 激活时阻止其他跳跃
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Ability.Jump")));
    
    // 被眩晕时无法跳跃
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Status.Stunned")));
}

bool UGA_Jump::CanActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayTagContainer* SourceTags,
    const FGameplayTagContainer* TargetTags,
    FGameplayTagContainer* OptionalRelevantTags) const
{
    if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
    {
        return false;
    }
    
    // 检查是否在地面上
    ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
    if (!Character || !Character->CanJump())
    {
        return false;
    }
    
    // 检查体力值
    ULyraAbilitySystemComponent* ASC = Cast<ULyraAbilitySystemComponent>(ActorInfo->AbilitySystemComponent.Get());
    const ULyraMovementSet* MovementSet = ASC->GetSet<ULyraMovementSet>();
    
    if (MovementSet && MovementSet->GetStamina() < StaminaCost)
    {
        return false;  // 体力不足
    }
    
    return true;
}

void UGA_Jump::ActivateAbility(
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
    
    // 执行跳跃
    ACharacter* Character = CastChecked<ACharacter>(ActorInfo->AvatarActor.Get());
    Character->Jump();
    
    // 应用冷却效果
    ApplyCooldown(Handle, ActorInfo, ActivationInfo);
    
    // 结束技能
    EndAbility(Handle, ActorInfo, ActivationInfo, false, false);
}

void UGA_Jump::EndAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    bool bReplicateEndAbility,
    bool bWasCancelled)
{
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

### Step 2: 创建 Gameplay Effect（消耗体力）

创建蓝图 `GE_Jump_Cost`：

```
GameplayEffect: GE_Jump_Cost
    DurationPolicy: Instant
    
    Modifiers:
        [0]:
            Attribute: LyraMovementSet.Stamina
            ModifierOp: Additive
            ModifierMagnitude: -10.0 (消耗10点体力)
```

### Step 3: 创建冷却效果

创建蓝图 `GE_Jump_Cooldown`：

```
GameplayEffect: GE_Jump_Cooldown
    DurationPolicy: HasDuration
    Duration: 1.0s
    
    GrantedTags:
        - Cooldown.Ability.Jump
```

### Step 4: 配置 Ability Set

在 `AbilitySet_HeroDefault` 中添加：

```
GrantedGameplayAbilities:
    [0]:
        Ability: GA_Jump
        AbilityLevel: 1
        InputTag: InputTag.Ability.Jump
```

### Step 5: 绑定输入

在 `IMC_Default_KBM` (Input Mapping Context) 中：

```
Mappings:
    [0]:
        Action: IA_Jump
        Key: SpaceBar
        Triggers: Pressed
        
        Gameplay Tags:
            - InputTag.Ability.Jump
```

---

## 🏃 实战：实现冲刺系统

### 需求分析

- 🎯 按住 Shift 键冲刺
- ⚡ 持续消耗体力（每秒10点）
- 🚀 移动速度提升50%
- ❄️ 体力耗尽时自动停止

### Step 1: 创建 Gameplay Ability

```cpp
// GA_Sprint.h

UCLASS()
class UGA_Sprint : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Sprint();

protected:
    virtual void ActivateAbility(...) override;
    virtual void EndAbility(...) override;
    virtual void InputReleased(...) override;

private:
    // 每秒消耗体力
    void OnStaminaTick();
    
    FTimerHandle StaminaTickHandle;
    
    UPROPERTY(EditDefaultsOnly)
    float StaminaCostPerSecond = 10.0f;
    
    UPROPERTY(EditDefaultsOnly)
    float SpeedMultiplier = 1.5f;
};
```

```cpp
// GA_Sprint.cpp

UGA_Sprint::UGA_Sprint()
{
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Ability.Sprint")));
    
    // 冲刺时阻止瞄准
    BlockAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Ability.Aim")));
}

void UGA_Sprint::ActivateAbility(...)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }
    
    // 应用速度 buff
    UAbilitySystemComponent* ASC = ActorInfo->AbilitySystemComponent.Get();
    FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
    
    FGameplayEffectSpecHandle EffectSpec = ASC->MakeOutgoingSpec(
        UGE_SpeedBuff::StaticClass(),
        1.0f,
        EffectContext
    );
    
    ASC->ApplyGameplayEffectSpecToSelf(*EffectSpec.Data.Get());
    
    // 启动体力消耗定时器
    GetWorld()->GetTimerManager().SetTimer(
        StaminaTickHandle,
        this,
        &ThisClass::OnStaminaTick,
        1.0f,
        true,
        0.0f  // 立即执行一次
    );
}

void UGA_Sprint::OnStaminaTick()
{
    ULyraAbilitySystemComponent* ASC = Cast<ULyraAbilitySystemComponent>(GetAbilitySystemComponentFromActorInfo());
    const ULyraMovementSet* MovementSet = ASC->GetSet<ULyraMovementSet>();
    
    if (MovementSet->GetStamina() < StaminaCostPerSecond)
    {
        // 体力不足，停止冲刺
        CancelAbility(GetCurrentAbilitySpecHandle(), GetCurrentActorInfo(), GetCurrentActivationInfo(), true);
        return;
    }
    
    // 消耗体力
    FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
    FGameplayEffectSpecHandle EffectSpec = ASC->MakeOutgoingSpec(
        UGE_StaminaCost::StaticClass(),
        1.0f,
        EffectContext
    );
    
    // 设置消耗量
    EffectSpec.Data->SetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag(TEXT("Data.Stamina.Cost")),
        StaminaCostPerSecond
    );
    
    ASC->ApplyGameplayEffectSpecToSelf(*EffectSpec.Data.Get());
}

void UGA_Sprint::InputReleased(...)
{
    // 松开 Shift 键，停止冲刺
    CancelAbility(GetCurrentAbilitySpecHandle(), GetCurrentActorInfo(), GetCurrentActivationInfo(), false);
}

void UGA_Sprint::EndAbility(...)
{
    // 清理定时器
    GetWorld()->GetTimerManager().ClearTimer(StaminaTickHandle);
    
    // 移除速度 buff（GE 会自动清理）
    
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

### Step 2: 创建速度 buff 效果

```
GameplayEffect: GE_SpeedBuff
    DurationPolicy: Infinite (无限持续，直到技能结束)
    
    Modifiers:
        [0]:
            Attribute: LyraMovementSet.MaxMoveSpeed
            ModifierOp: Multiply
            ModifierMagnitude: 1.5 (速度 x1.5)
    
    GrantedTags:
        - Status.Buff.Sprint
```

---

## 💬 总结

### 核心要点

1. **GAS 是什么？**
   - Epic 官方的技能系统框架
   - 解决网络同步、buff/debuff、技能管理等问题

2. **5 大核心概念**
   - Ability System Component：核心管理器
   - Gameplay Ability：技能逻辑
   - Attribute Set：属性集合
   - Gameplay Effect：效果系统
   - Gameplay Tags：标签控制

3. **Lyra 的 GAS 架构**
   - ASC 挂载在 PlayerState 上
   - 使用 Ability Set 批量管理技能
   - 通过 InputTag 绑定输入

4. **实战价值**
   - 跳跃技能：展示基础技能实现
   - 冲刺系统：展示持续技能和资源消耗

### 下一篇预告

第七篇：**GAS 进阶：Attributes 与 Gameplay Effects**

- Attribute Set 设计原则
- LyraHealthSet 和 LyraCombatSet 分析
- Gameplay Effects 的 7 种应用场景
- 实战：创建护盾系统和伤害计算

准备好深入 GAS 的核心机制了吗？💪

---

> **本文是《UE5 Lyra 深度解析》系列教程的第 6 篇**  
> 上一篇：[数据驱动设计与 Data Assets](05_data_driven_design.md)  
> 下一篇：《GAS 进阶：Attributes 与 Gameplay Effects》  
> 作者：lobsterchen
