# GAS 高级：能力系统的实战应用

## 概述

在前面的章节中，我们已经学习了 Gameplay Ability System (GAS) 的基础概念和核心组件。本章将深入实战，通过构建完整的战斗系统来展示 GAS 在真实项目中的高级应用。

本文将涵盖：

- **复杂技能系统设计**：多阶段技能、连招系统、技能打断机制
- **完整战斗系统**：伤害计算、Buff/Debuff、控制效果
- **高级 GAS 技巧**：自定义 Gameplay Task、技能队列、Target Data 处理
- **性能优化**：实例化策略、网络优化、调试工具

我们将构建一个类似格斗游戏的战斗系统作为实战案例，包含：
- 轻重攻击连招系统
- 蓄力技能（可打断、可取消）
- 完整的伤害计算（基础伤害、暴击、护甲减免）
- Buff/Debuff 系统（增益、减益、控制效果）
- 技能队列与优先级管理

## 1. 复杂技能系统设计

### 1.1 多阶段技能系统

多阶段技能是现代游戏中常见的设计模式，例如：
- **蓄力技能**：准备阶段 → 蓄力阶段 → 释放阶段 → 恢复阶段
- **持续施法**：开始施法 → 引导中 → 完成/打断
- **连击技能**：第一段 → 第二段 → 第三段（终结技）

#### 1.1.1 蓄力技能实现

让我们实现一个完整的蓄力重击技能，支持可变蓄力时长和伤害倍率：

```cpp
// ChargedHeavyAttack.h
#pragma once

#include "CoreMinimal.h"
#include "Abilities/LyraGameplayAbility.h"
#include "ChargedHeavyAttack.generated.h"

/**
 * 蓄力重击技能
 * 特性：
 * - 按住按键蓄力，松开释放
 * - 蓄力时长影响伤害倍率
 * - 蓄力过程中可以移动但速度减半
 * - 可以被打断
 */
UCLASS()
class LYRAGAME_API UChargedHeavyAttack : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UChargedHeavyAttack();

    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData
    ) override;

    virtual void InputReleased(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo
    ) override;

    virtual void CancelAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateCancelAbility
    ) override;

protected:
    // 阶段枚举
    UENUM(BlueprintType)
    enum class EChargePhase : uint8
    {
        None,
        Preparing,      // 准备阶段（播放起手动画）
        Charging,       // 蓄力阶段
        Releasing,      // 释放阶段
        Recovering      // 恢复阶段
    };

    // 当前阶段
    EChargePhase CurrentPhase;

    // 蓄力开始时间
    float ChargeStartTime;

    // 配置参数
    UPROPERTY(EditDefaultsOnly, Category = "Charge Settings")
    float MinChargeDuration;          // 最小蓄力时长（秒）

    UPROPERTY(EditDefaultsOnly, Category = "Charge Settings")
    float MaxChargeDuration;          // 最大蓄力时长（秒）

    UPROPERTY(EditDefaultsOnly, Category = "Charge Settings")
    float MinDamageMultiplier;        // 最小伤害倍率

    UPROPERTY(EditDefaultsOnly, Category = "Charge Settings")
    float MaxDamageMultiplier;        // 最大伤害倍率

    UPROPERTY(EditDefaultsOnly, Category = "Charge Settings")
    float ChargingMovementSpeedMultiplier;  // 蓄力时移动速度倍率

    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    UAnimMontage* PrepareAnimMontage;

    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    UAnimMontage* ChargingLoopMontage;

    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    UAnimMontage* ReleaseAnimMontage;

    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    TSubclassOf<UGameplayEffect> ChargeMovementSpeedEffect;

    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    TSubclassOf<UGameplayEffect> DamageEffect;

    // 内部函数
    void StartPreparePhase();
    void StartChargingPhase();
    void ReleaseCharge();
    void OnPrepareAnimCompleted();
    void ApplyMovementSpeedModifier();
    void RemoveMovementSpeedModifier();
    float CalculateCurrentChargePower() const;
    void ExecuteAttack(float ChargePower);

    // Effect Handle
    FActiveGameplayEffectHandle MovementSpeedEffectHandle;

    // Montage通知回调
    UFUNCTION()
    void OnMontageCompleted(FGameplayTag EventTag, FGameplayEventData EventData);

    UFUNCTION()
    void OnMontageCancelled(FGameplayTag EventTag, FGameplayEventData EventData);
};
```

```cpp
// ChargedHeavyAttack.cpp
#include "ChargedHeavyAttack.h"
#include "AbilitySystemComponent.h"
#include "GameFramework/Character.h"
#include "GameplayEffect.h"
#include "Abilities/Tasks/AbilityTask_PlayMontageAndWait.h"
#include "Abilities/Tasks/AbilityTask_WaitGameplayEvent.h"

UChargedHeavyAttack::UChargedHeavyAttack()
{
    // 默认配置
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;

    MinChargeDuration = 0.5f;
    MaxChargeDuration = 3.0f;
    MinDamageMultiplier = 1.0f;
    MaxDamageMultiplier = 3.0f;
    ChargingMovementSpeedMultiplier = 0.5f;

    CurrentPhase = EChargePhase::None;
    ChargeStartTime = 0.0f;

    // 设置技能标签
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Melee.HeavyAttack")));
    
    // 激活时阻止其他攻击技能
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Melee")));
    
    // 被控制时无法使用
    ActivationRequiredTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.CanAttack")));
    BlockAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability")));
}

void UChargedHeavyAttack::ActivateAbility(
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

    // 开始准备阶段
    StartPreparePhase();
}

void UChargedHeavyAttack::StartPreparePhase()
{
    CurrentPhase = EChargePhase::Preparing;

    // 播放准备动画
    if (PrepareAnimMontage)
    {
        UAbilityTask_PlayMontageAndWait* PrepareTask = UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
            this,
            NAME_None,
            PrepareAnimMontage,
            1.0f
        );

        PrepareTask->OnCompleted.AddDynamic(this, &UChargedHeavyAttack::OnPrepareAnimCompleted);
        PrepareTask->OnInterrupted.AddDynamic(this, &UChargedHeavyAttack::OnMontageCancelled);
        PrepareTask->ReadyForActivation();
    }
    else
    {
        // 没有准备动画，直接进入蓄力阶段
        StartChargingPhase();
    }
}

void UChargedHeavyAttack::OnPrepareAnimCompleted()
{
    StartChargingPhase();
}

void UChargedHeavyAttack::StartChargingPhase()
{
    CurrentPhase = EChargePhase::Charging;
    ChargeStartTime = GetWorld()->GetTimeSeconds();

    // 应用移动速度减益
    ApplyMovementSpeedModifier();

    // 播放蓄力循环动画
    if (ChargingLoopMontage)
    {
        ACharacter* Character = Cast<ACharacter>(GetAvatarActorFromActorInfo());
        if (Character && Character->GetMesh())
        {
            Character->GetMesh()->GetAnimInstance()->Montage_Play(ChargingLoopMontage, 1.0f);
            // 设置循环播放
            Character->GetMesh()->GetAnimInstance()->Montage_SetNextSection(
                ChargingLoopMontage->GetSectionName(0),
                ChargingLoopMontage->GetSectionName(0),
                ChargingLoopMontage
            );
        }
    }

    // 给玩家添加视觉反馈标签
    GetAbilitySystemComponentFromActorInfo()->AddLooseGameplayTag(
        FGameplayTag::RequestGameplayTag(FName("State.Charging"))
    );
}

void UChargedHeavyAttack::InputReleased(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo)
{
    // 只有在蓄力阶段才响应释放
    if (CurrentPhase == EChargePhase::Charging)
    {
        ReleaseCharge();
    }
}

void UChargedHeavyAttack::ReleaseCharge()
{
    if (CurrentPhase != EChargePhase::Charging)
    {
        return;
    }

    CurrentPhase = EChargePhase::Releasing;

    // 移除蓄力状态标签
    GetAbilitySystemComponentFromActorInfo()->RemoveLooseGameplayTag(
        FGameplayTag::RequestGameplayTag(FName("State.Charging"))
    );

    // 移除移动速度减益
    RemoveMovementSpeedModifier();

    // 停止蓄力动画
    ACharacter* Character = Cast<ACharacter>(GetAvatarActorFromActorInfo());
    if (Character && Character->GetMesh() && ChargingLoopMontage)
    {
        Character->GetMesh()->GetAnimInstance()->Montage_Stop(0.2f, ChargingLoopMontage);
    }

    // 计算蓄力强度
    float ChargePower = CalculateCurrentChargePower();

    // 执行攻击
    ExecuteAttack(ChargePower);

    // 播放释放动画
    if (ReleaseAnimMontage)
    {
        UAbilityTask_PlayMontageAndWait* ReleaseTask = UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
            this,
            NAME_None,
            ReleaseAnimMontage,
            1.0f
        );

        ReleaseTask->OnCompleted.AddDynamic(this, &UChargedHeavyAttack::OnMontageCompleted);
        ReleaseTask->OnInterrupted.AddDynamic(this, &UChargedHeavyAttack::OnMontageCancelled);
        ReleaseTask->ReadyForActivation();
    }
    else
    {
        // 没有释放动画，直接结束技能
        EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
    }
}

float UChargedHeavyAttack::CalculateCurrentChargePower() const
{
    float CurrentTime = GetWorld()->GetTimeSeconds();
    float ChargeDuration = FMath::Clamp(
        CurrentTime - ChargeStartTime,
        MinChargeDuration,
        MaxChargeDuration
    );

    // 线性插值计算蓄力强度
    float ChargeRatio = (ChargeDuration - MinChargeDuration) / (MaxChargeDuration - MinChargeDuration);
    float DamageMultiplier = FMath::Lerp(MinDamageMultiplier, MaxDamageMultiplier, ChargeRatio);

    return DamageMultiplier;
}

void UChargedHeavyAttack::ExecuteAttack(float ChargePower)
{
    // 创建伤害 Gameplay Effect Spec
    if (DamageEffect)
    {
        FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(DamageEffect, GetAbilityLevel());
        if (SpecHandle.IsValid())
        {
            // 将蓄力倍率添加到 Effect Context 中
            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(FName("Data.Damage.Multiplier")),
                ChargePower
            );

            // 应用到目标（通过碰撞检测获取）
            // 这里简化处理，实际项目中应该使用 Target Data
            TArray<AActor*> HitActors;
            PerformMeleeTrace(HitActors);

            for (AActor* HitActor : HitActors)
            {
                UAbilitySystemComponent* TargetASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(HitActor);
                if (TargetASC)
                {
                    GetAbilitySystemComponentFromActorInfo()->ApplyGameplayEffectSpecToTarget(
                        *SpecHandle.Data.Get(),
                        TargetASC
                    );
                }
            }
        }
    }
}

void UChargedHeavyAttack::PerformMeleeTrace(TArray<AActor*>& OutHitActors)
{
    // 简化的近战碰撞检测
    // 实际项目中应该使用更复杂的碰撞检测逻辑
    ACharacter* Character = Cast<ACharacter>(GetAvatarActorFromActorInfo());
    if (!Character)
    {
        return;
    }

    FVector Start = Character->GetActorLocation();
    FVector Forward = Character->GetActorForwardVector();
    FVector End = Start + Forward * 200.0f; // 攻击范围 200cm

    // 球形扫描
    TArray<FHitResult> HitResults;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(Character);

    GetWorld()->SweepMultiByChannel(
        HitResults,
        Start,
        End,
        FQuat::Identity,
        ECC_Pawn,
        FCollisionShape::MakeSphere(100.0f), // 半径 100cm
        QueryParams
    );

    for (const FHitResult& Hit : HitResults)
    {
        if (Hit.GetActor() && Hit.GetActor() != Character)
        {
            OutHitActors.AddUnique(Hit.GetActor());
        }
    }
}

void UChargedHeavyAttack::ApplyMovementSpeedModifier()
{
    if (ChargeMovementSpeedEffect)
    {
        FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(
            ChargeMovementSpeedEffect,
            GetAbilityLevel()
        );

        if (SpecHandle.IsValid())
        {
            // 设置移动速度倍率
            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(FName("Data.MovementSpeed.Multiplier")),
                ChargingMovementSpeedMultiplier
            );

            MovementSpeedEffectHandle = GetAbilitySystemComponentFromActorInfo()->ApplyGameplayEffectSpecToSelf(
                *SpecHandle.Data.Get()
            );
        }
    }
}

void UChargedHeavyAttack::RemoveMovementSpeedModifier()
{
    if (MovementSpeedEffectHandle.IsValid())
    {
        GetAbilitySystemComponentFromActorInfo()->RemoveActiveGameplayEffect(MovementSpeedEffectHandle);
        MovementSpeedEffectHandle.Invalidate();
    }
}

void UChargedHeavyAttack::CancelAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    bool bReplicateCancelAbility)
{
    // 清理状态
    if (CurrentPhase == EChargePhase::Charging)
    {
        RemoveMovementSpeedModifier();
        GetAbilitySystemComponentFromActorInfo()->RemoveLooseGameplayTag(
            FGameplayTag::RequestGameplayTag(FName("State.Charging"))
        );
    }

    Super::CancelAbility(Handle, ActorInfo, ActivationInfo, bReplicateCancelAbility);
}

void UChargedHeavyAttack::OnMontageCompleted(FGameplayTag EventTag, FGameplayEventData EventData)
{
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}

void UChargedHeavyAttack::OnMontageCancelled(FGameplayTag EventTag, FGameplayEventData EventData)
{
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, true);
}
```

#### 1.1.2 配套 Gameplay Effect

蓄力技能需要两个 Gameplay Effect：

**1. 移动速度减益 Effect**

```cpp
// GE_ChargeMovementSpeed.h
UCLASS()
class UGE_ChargeMovementSpeed : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UGE_ChargeMovementSpeed()
    {
        DurationPolicy = EGameplayEffectDurationType::Infinite;

        // 修改移动速度属性
        FGameplayModifierInfo SpeedModifier;
        SpeedModifier.Attribute = UMyAttributeSet::GetMovementSpeedAttribute();
        SpeedModifier.ModifierOp = EGameplayModOp::Multiply;
        
        FSetByCallerFloat SpeedValue;
        SpeedValue.DataTag = FGameplayTag::RequestGameplayTag(FName("Data.MovementSpeed.Multiplier"));
        SpeedModifier.ModifierMagnitude = FGameplayEffectModifierMagnitude(SpeedValue);

        Modifiers.Add(SpeedModifier);
    }
};
```

**2. 伤害 Effect**

```cpp
// GE_ChargedHeavyDamage.h
UCLASS()
class UGE_ChargedHeavyDamage : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UGE_ChargedHeavyDamage()
    {
        DurationPolicy = EGameplayEffectDurationType::Instant;

        // 基础伤害
        FGameplayModifierInfo DamageModifier;
        DamageModifier.Attribute = UMyAttributeSet::GetHealthAttribute();
        DamageModifier.ModifierOp = EGameplayModOp::Additive;

        // 伤害计算：基础伤害 * 蓄力倍率
        FCustomCalculationBasedFloat DamageCalc;
        DamageCalc.CalculationClassMagnitude = UDamageExecution::StaticClass();
        
        FSetByCallerFloat MultiplierValue;
        MultiplierValue.DataTag = FGameplayTag::RequestGameplayTag(FName("Data.Damage.Multiplier"));
        
        DamageModifier.ModifierMagnitude = FGameplayEffectModifierMagnitude(DamageCalc);
        Modifiers.Add(DamageModifier);
    }
};
```

### 1.2 连招系统实现

连招系统是格斗游戏的核心机制，需要处理：
- 连击窗口（Combo Window）
- 输入缓冲（Input Buffer）
- 连击计数与重置
- 不同连击段的动画与判定

#### 1.2.1 连招管理组件

```cpp
// ComboManagerComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "GameplayTagContainer.h"
#include "ComboManagerComponent.generated.h"

/**
 * 连招数据定义
 */
USTRUCT(BlueprintType)
struct FComboSegment
{
    GENERATED_BODY()

    // 连击段索引
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 SegmentIndex;

    // 该段使用的技能
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSubclassOf<UGameplayAbility> AbilityClass;

    // 该段的动画
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    UAnimMontage* AnimMontage;

    // 连击窗口开始时间（相对于动画开始的时间，秒）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float ComboWindowStart;

    // 连击窗口持续时长（秒）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float ComboWindowDuration;

    // 伤害倍率
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float DamageMultiplier;

    // 下一段连击的可能选择（支持分支连招）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<int32> NextSegments;

    FComboSegment()
        : SegmentIndex(0)
        , AbilityClass(nullptr)
        , AnimMontage(nullptr)
        , ComboWindowStart(0.5f)
        , ComboWindowDuration(0.3f)
        , DamageMultiplier(1.0f)
    {}
};

/**
 * 连招配置
 */
USTRUCT(BlueprintType)
struct FComboChain
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FName ComboName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FComboSegment> Segments;

    // 连招超时时间（秒，超过该时间未输入则重置连招）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float ComboResetTimeout;

    FComboChain()
        : ComboName(NAME_None)
        , ComboResetTimeout(2.0f)
    {}
};

/**
 * 连招管理组件
 * 负责追踪当前连击状态、处理连击输入、管理连击窗口
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class LYRAGAME_API UComboManagerComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UComboManagerComponent();

    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;

    // 配置连招链
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combo")
    TMap<FGameplayTag, FComboChain> ComboChains;

    // 尝试推进连招
    UFUNCTION(BlueprintCallable, Category = "Combo")
    bool TryAdvanceCombo(FGameplayTag ComboTag);

    // 重置连招
    UFUNCTION(BlueprintCallable, Category = "Combo")
    void ResetCombo();

    // 获取当前连击段
    UFUNCTION(BlueprintPure, Category = "Combo")
    int32 GetCurrentComboSegment() const { return CurrentSegmentIndex; }

    // 是否在连击窗口内
    UFUNCTION(BlueprintPure, Category = "Combo")
    bool IsInComboWindow() const;

    // 开始连击段（由 Ability 调用）
    UFUNCTION(BlueprintCallable, Category = "Combo")
    void StartComboSegment(FGameplayTag ComboTag, int32 SegmentIndex);

    // 结束连击段
    UFUNCTION(BlueprintCallable, Category = "Combo")
    void EndComboSegment();

protected:
    virtual void BeginPlay() override;

private:
    // 当前激活的连招标签
    FGameplayTag CurrentComboTag;

    // 当前连击段索引
    int32 CurrentSegmentIndex;

    // 连击段开始时间
    float SegmentStartTime;

    // 上次输入时间
    float LastInputTime;

    // 是否在连击窗口内
    bool bIsInComboWindow;

    // 输入缓冲（用户在连击窗口外输入，暂存在这里）
    TArray<FGameplayTag> InputBuffer;

    // 最大输入缓冲数量
    UPROPERTY(EditAnywhere, Category = "Combo")
    int32 MaxInputBufferSize;

    // 输入缓冲超时时间（秒）
    UPROPERTY(EditAnywhere, Category = "Combo")
    float InputBufferTimeout;

    void UpdateComboWindow(float DeltaTime);
    void ProcessInputBuffer();
    void ClearInputBuffer();
    const FComboSegment* GetSegmentData(FGameplayTag ComboTag, int32 SegmentIndex) const;
};
```

```cpp
// ComboManagerComponent.cpp
#include "ComboManagerComponent.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemGlobals.h"

UComboManagerComponent::UComboManagerComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
    CurrentSegmentIndex = -1;
    SegmentStartTime = 0.0f;
    LastInputTime = 0.0f;
    bIsInComboWindow = false;
    MaxInputBufferSize = 3;
    InputBufferTimeout = 0.5f;
}

void UComboManagerComponent::BeginPlay()
{
    Super::BeginPlay();
}

void UComboManagerComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    UpdateComboWindow(DeltaTime);

    // 检查连招超时
    if (CurrentSegmentIndex >= 0)
    {
        const FComboChain* Chain = ComboChains.Find(CurrentComboTag);
        if (Chain)
        {
            float TimeSinceLastInput = GetWorld()->GetTimeSeconds() - LastInputTime;
            if (TimeSinceLastInput > Chain->ComboResetTimeout)
            {
                ResetCombo();
            }
        }
    }

    // 清理过期的输入缓冲
    float CurrentTime = GetWorld()->GetTimeSeconds();
    if (CurrentTime - LastInputTime > InputBufferTimeout)
    {
        ClearInputBuffer();
    }
}

void UComboManagerComponent::UpdateComboWindow(float DeltaTime)
{
    if (CurrentSegmentIndex < 0)
    {
        bIsInComboWindow = false;
        return;
    }

    const FComboSegment* Segment = GetSegmentData(CurrentComboTag, CurrentSegmentIndex);
    if (!Segment)
    {
        bIsInComboWindow = false;
        return;
    }

    float CurrentTime = GetWorld()->GetTimeSeconds();
    float TimeInSegment = CurrentTime - SegmentStartTime;

    // 检查是否在连击窗口内
    bool bWasInWindow = bIsInComboWindow;
    bIsInComboWindow = (TimeInSegment >= Segment->ComboWindowStart) &&
                       (TimeInSegment <= Segment->ComboWindowStart + Segment->ComboWindowDuration);

    // 刚进入连击窗口时，处理缓冲的输入
    if (bIsInComboWindow && !bWasInWindow)
    {
        ProcessInputBuffer();
    }
}

bool UComboManagerComponent::TryAdvanceCombo(FGameplayTag ComboTag)
{
    LastInputTime = GetWorld()->GetTimeSeconds();

    // 如果没有激活的连招，尝试开始新连招
    if (CurrentSegmentIndex < 0)
    {
        const FComboChain* Chain = ComboChains.Find(ComboTag);
        if (Chain && Chain->Segments.Num() > 0)
        {
            StartComboSegment(ComboTag, 0);
            return true;
        }
        return false;
    }

    // 如果在连击窗口内，推进连招
    if (bIsInComboWindow)
    {
        const FComboSegment* CurrentSegment = GetSegmentData(CurrentComboTag, CurrentSegmentIndex);
        if (CurrentSegment && CurrentSegment->NextSegments.Num() > 0)
        {
            // 简化处理：总是选择第一个可能的下一段
            // 实际项目中可能需要根据输入类型选择不同的分支
            int32 NextSegmentIndex = CurrentSegment->NextSegments[0];
            StartComboSegment(CurrentComboTag, NextSegmentIndex);
            return true;
        }
    }
    else
    {
        // 不在连击窗口内，将输入加入缓冲
        if (InputBuffer.Num() < MaxInputBufferSize)
        {
            InputBuffer.Add(ComboTag);
        }
    }

    return false;
}

void UComboManagerComponent::StartComboSegment(FGameplayTag ComboTag, int32 SegmentIndex)
{
    const FComboSegment* Segment = GetSegmentData(ComboTag, SegmentIndex);
    if (!Segment)
    {
        UE_LOG(LogTemp, Warning, TEXT("StartComboSegment: Invalid segment %d for combo %s"), 
               SegmentIndex, *ComboTag.ToString());
        return;
    }

    CurrentComboTag = ComboTag;
    CurrentSegmentIndex = SegmentIndex;
    SegmentStartTime = GetWorld()->GetTimeSeconds();
    bIsInComboWindow = false;

    // 激活对应的 Ability
    AActor* Owner = GetOwner();
    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Owner);
    if (ASC && Segment->AbilityClass)
    {
        FGameplayAbilitySpec Spec(Segment->AbilityClass, 1, INDEX_NONE, Owner);
        
        // 将连击信息传递给 Ability
        FGameplayEventData EventData;
        EventData.Instigator = Owner;
        EventData.Target = Owner;
        EventData.EventMagnitude = Segment->DamageMultiplier;
        
        ASC->GiveAbilityAndActivateOnce(Spec, &EventData);
    }

    UE_LOG(LogTemp, Log, TEXT("Started combo segment %d of %s"), SegmentIndex, *ComboTag.ToString());
}

void UComboManagerComponent::EndComboSegment()
{
    // 连击段结束但不重置连招，等待下一段输入
}

void UComboManagerComponent::ResetCombo()
{
    CurrentComboTag = FGameplayTag();
    CurrentSegmentIndex = -1;
    SegmentStartTime = 0.0f;
    bIsInComboWindow = false;
    ClearInputBuffer();

    UE_LOG(LogTemp, Log, TEXT("Combo reset"));
}

bool UComboManagerComponent::IsInComboWindow() const
{
    return bIsInComboWindow;
}

void UComboManagerComponent::ProcessInputBuffer()
{
    if (InputBuffer.Num() > 0)
    {
        // 处理第一个缓冲的输入
        FGameplayTag BufferedTag = InputBuffer[0];
        InputBuffer.RemoveAt(0);

        TryAdvanceCombo(BufferedTag);
    }
}

void UComboManagerComponent::ClearInputBuffer()
{
    InputBuffer.Empty();
}

const FComboSegment* UComboManagerComponent::GetSegmentData(FGameplayTag ComboTag, int32 SegmentIndex) const
{
    const FComboChain* Chain = ComboChains.Find(ComboTag);
    if (!Chain)
    {
        return nullptr;
    }

    for (const FComboSegment& Segment : Chain->Segments)
    {
        if (Segment.SegmentIndex == SegmentIndex)
        {
            return &Segment;
        }
    }

    return nullptr;
}
```

#### 1.2.2 连招技能基类

```cpp
// ComboAbility.h
#pragma once

#include "CoreMinimal.h"
#include "Abilities/LyraGameplayAbility.h"
#include "ComboAbility.generated.h"

/**
 * 连招技能基类
 * 所有连击技能继承自此类
 */
UCLASS()
class LYRAGAME_API UComboAbility : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UComboAbility();

    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData
    ) override;

protected:
    // 连击段索引
    UPROPERTY(EditDefaultsOnly, Category = "Combo")
    int32 ComboSegmentIndex;

    // 所属连招标签
    UPROPERTY(EditDefaultsOnly, Category = "Combo")
    FGameplayTag ComboTag;

    // 动画
    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    UAnimMontage* AttackMontage;

    // 伤害倍率（可被 Combo Manager 覆盖）
    UPROPERTY(EditDefaultsOnly, Category = "Damage")
    float BaseDamageMultiplier;

    // 伤害 Effect
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    TSubclassOf<UGameplayEffect> DamageEffectClass;

    // 动画通知回调
    UFUNCTION()
    void OnAttackMontageCompleted(FGameplayTag EventTag, FGameplayEventData EventData);

    UFUNCTION()
    void OnHitTargets(FGameplayTag EventTag, FGameplayEventData EventData);

    // 通知 Combo Manager
    void NotifyComboManager();
};
```

```cpp
// ComboAbility.cpp
#include "ComboAbility.h"
#include "ComboManagerComponent.h"
#include "Abilities/Tasks/AbilityTask_PlayMontageAndWait.h"
#include "Abilities/Tasks/AbilityTask_WaitGameplayEvent.h"

UComboAbility::UComboAbility()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;

    ComboSegmentIndex = 0;
    BaseDamageMultiplier = 1.0f;
}

void UComboAbility::ActivateAbility(
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

    // 通知 Combo Manager 开始该段连击
    NotifyComboManager();

    // 从 TriggerEventData 获取伤害倍率（如果有）
    float DamageMultiplier = BaseDamageMultiplier;
    if (TriggerEventData)
    {
        DamageMultiplier = TriggerEventData->EventMagnitude > 0 ? 
                          TriggerEventData->EventMagnitude : BaseDamageMultiplier;
    }

    // 播放攻击动画
    if (AttackMontage)
    {
        UAbilityTask_PlayMontageAndWait* MontageTask = UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
            this,
            NAME_None,
            AttackMontage,
            1.0f
        );

        MontageTask->OnCompleted.AddDynamic(this, &UComboAbility::OnAttackMontageCompleted);
        MontageTask->OnInterrupted.AddDynamic(this, &UComboAbility::OnAttackMontageCompleted);
        MontageTask->ReadyForActivation();
    }

    // 等待动画通知触发攻击判定
    UAbilityTask_WaitGameplayEvent* WaitHitEvent = UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
        this,
        FGameplayTag::RequestGameplayTag(FName("Event.Montage.Attack.Hit"))
    );

    WaitHitEvent->EventReceived.AddDynamic(this, &UComboAbility::OnHitTargets);
    WaitHitEvent->ReadyForActivation();
}

void UComboAbility::NotifyComboManager()
{
    AActor* Owner = GetAvatarActorFromActorInfo();
    if (!Owner)
    {
        return;
    }

    UComboManagerComponent* ComboManager = Owner->FindComponentByClass<UComboManagerComponent>();
    if (ComboManager)
    {
        ComboManager->StartComboSegment(ComboTag, ComboSegmentIndex);
    }
}

void UComboAbility::OnHitTargets(FGameplayTag EventTag, FGameplayEventData EventData)
{
    // 在动画通知中触发攻击判定
    // 这里简化处理，实际应该使用更精确的碰撞检测

    TArray<AActor*> HitActors;
    // 执行攻击判定（类似蓄力技能的碰撞检测）
    // PerformMeleeTrace(HitActors);

    if (DamageEffectClass)
    {
        FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(DamageEffectClass, GetAbilityLevel());
        
        if (SpecHandle.IsValid())
        {
            // 应用伤害倍率
            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(FName("Data.Damage.Multiplier")),
                BaseDamageMultiplier
            );

            // 应用到所有命中目标
            for (AActor* HitActor : HitActors)
            {
                UAbilitySystemComponent* TargetASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(HitActor);
                if (TargetASC)
                {
                    GetAbilitySystemComponentFromActorInfo()->ApplyGameplayEffectSpecToTarget(
                        *SpecHandle.Data.Get(),
                        TargetASC
                    );
                }
            }
        }
    }
}

void UComboAbility::OnAttackMontageCompleted(FGameplayTag EventTag, FGameplayEventData EventData)
{
    AActor* Owner = GetAvatarActorFromActorInfo();
    if (Owner)
    {
        UComboManagerComponent* ComboManager = Owner->FindComponentByClass<UComboManagerComponent>();
        if (ComboManager)
        {
            ComboManager->EndComboSegment();
        }
    }

    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}
```

#### 1.2.3 连招配置示例（蓝图或 Data Asset）

```cpp
// 在角色蓝图中配置连招链
void AMyCharacter::SetupComboChains()
{
    UComboManagerComponent* ComboManager = FindComponentByClass<UComboManagerComponent>();
    if (!ComboManager)
    {
        return;
    }

    // 创建轻攻击连招链
    FComboChain LightAttackChain;
    LightAttackChain.ComboName = TEXT("LightAttack");
    LightAttackChain.ComboResetTimeout = 2.0f;

    // 第一段：轻攻击 1
    FComboSegment Segment1;
    Segment1.SegmentIndex = 0;
    Segment1.AbilityClass = ULightAttack1::StaticClass();
    Segment1.ComboWindowStart = 0.4f;
    Segment1.ComboWindowDuration = 0.3f;
    Segment1.DamageMultiplier = 1.0f;
    Segment1.NextSegments.Add(1);  // 可以连接到第二段
    LightAttackChain.Segments.Add(Segment1);

    // 第二段：轻攻击 2
    FComboSegment Segment2;
    Segment2.SegmentIndex = 1;
    Segment2.AbilityClass = ULightAttack2::StaticClass();
    Segment2.ComboWindowStart = 0.3f;
    Segment2.ComboWindowDuration = 0.3f;
    Segment2.DamageMultiplier = 1.2f;
    Segment2.NextSegments.Add(2);  // 可以连接到第三段
    LightAttackChain.Segments.Add(Segment2);

    // 第三段：轻攻击 3（终结技）
    FComboSegment Segment3;
    Segment3.SegmentIndex = 2;
    Segment3.AbilityClass = ULightAttack3::StaticClass();
    Segment3.ComboWindowStart = 0.5f;
    Segment3.ComboWindowDuration = 0.2f;
    Segment3.DamageMultiplier = 2.0f;
    // 没有下一段，连招结束
    LightAttackChain.Segments.Add(Segment3);

    // 注册连招链
    FGameplayTag LightAttackTag = FGameplayTag::RequestGameplayTag(FName("Ability.Melee.LightAttack"));
    ComboManager->ComboChains.Add(LightAttackTag, LightAttackChain);
}
```

### 1.3 技能打断与取消机制

技能打断是战斗系统中的重要机制，包括：
- **主动取消**：玩家按下取消键或闪避键
- **被动打断**：受到控制效果（眩晕、击飞）
- **优先级打断**：高优先级技能打断低优先级技能

#### 1.3.1 技能打断系统

```cpp
// AbilityInterruptionComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "GameplayTagContainer.h"
#include "AbilityInterruptionComponent.generated.h"

/**
 * 技能优先级枚举
 */
UENUM(BlueprintType)
enum class EAbilityPriority : uint8
{
    Low = 0,        // 低优先级（普通攻击）
    Normal = 1,     // 普通优先级（技能）
    High = 2,       // 高优先级（大招）
    Critical = 3    // 最高优先级（无法被打断，如无敌技能）
};

/**
 * 技能打断组件
 * 管理技能的打断逻辑和优先级
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class LYRAGAME_API UAbilityInterruptionComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UAbilityInterruptionComponent();

    // 尝试打断当前技能
    UFUNCTION(BlueprintCallable, Category = "Ability Interruption")
    bool TryInterruptCurrentAbility(EAbilityPriority InterruptingPriority, AActor* Instigator = nullptr);

    // 注册当前激活的技能
    UFUNCTION(BlueprintCallable, Category = "Ability Interruption")
    void RegisterActiveAbility(TSubclassOf<UGameplayAbility> AbilityClass, EAbilityPriority Priority, bool bCanBeInterrupted);

    // 取消注册技能
    UFUNCTION(BlueprintCallable, Category = "Ability Interruption")
    void UnregisterActiveAbility(TSubclassOf<UGameplayAbility> AbilityClass);

    // 获取当前技能优先级
    UFUNCTION(BlueprintPure, Category = "Ability Interruption")
    EAbilityPriority GetCurrentAbilityPriority() const { return CurrentAbilityPriority; }

    // 是否可以被打断
    UFUNCTION(BlueprintPure, Category = "Ability Interruption")
    bool CanCurrentAbilityBeInterrupted() const { return bCurrentAbilityCanBeInterrupted; }

protected:
    virtual void BeginPlay() override;

private:
    // 当前激活技能的优先级
    EAbilityPriority CurrentAbilityPriority;

    // 当前技能是否可被打断
    bool bCurrentAbilityCanBeInterrupted;

    // 当前激活的技能类
    TSubclassOf<UGameplayAbility> CurrentAbilityClass;

    // 打断技能的实际执行
    void ExecuteInterruption(AActor* Instigator);
};
```

```cpp
// AbilityInterruptionComponent.cpp
#include "AbilityInterruptionComponent.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemGlobals.h"

UAbilityInterruptionComponent::UAbilityInterruptionComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
    CurrentAbilityPriority = EAbilityPriority::Low;
    bCurrentAbilityCanBeInterrupted = true;
    CurrentAbilityClass = nullptr;
}

void UAbilityInterruptionComponent::BeginPlay()
{
    Super::BeginPlay();
}

void UAbilityInterruptionComponent::RegisterActiveAbility(
    TSubclassOf<UGameplayAbility> AbilityClass,
    EAbilityPriority Priority,
    bool bCanBeInterrupted)
{
    CurrentAbilityClass = AbilityClass;
    CurrentAbilityPriority = Priority;
    bCurrentAbilityCanBeInterrupted = bCanBeInterrupted;

    UE_LOG(LogTemp, Log, TEXT("Registered ability with priority %d, CanBeInterrupted: %s"),
           (int32)Priority, bCanBeInterrupted ? TEXT("true") : TEXT("false"));
}

void UAbilityInterruptionComponent::UnregisterActiveAbility(TSubclassOf<UGameplayAbility> AbilityClass)
{
    if (CurrentAbilityClass == AbilityClass)
    {
        CurrentAbilityClass = nullptr;
        CurrentAbilityPriority = EAbilityPriority::Low;
        bCurrentAbilityCanBeInterrupted = true;
    }
}

bool UAbilityInterruptionComponent::TryInterruptCurrentAbility(EAbilityPriority InterruptingPriority, AActor* Instigator)
{
    // 没有激活的技能
    if (!CurrentAbilityClass)
    {
        return false;
    }

    // 当前技能标记为不可打断
    if (!bCurrentAbilityCanBeInterrupted)
    {
        UE_LOG(LogTemp, Warning, TEXT("Current ability cannot be interrupted (marked as uninterruptible)"));
        return false;
    }

    // 检查优先级
    if (InterruptingPriority <= CurrentAbilityPriority)
    {
        UE_LOG(LogTemp, Warning, TEXT("Interrupting priority %d is not higher than current priority %d"),
               (int32)InterruptingPriority, (int32)CurrentAbilityPriority);
        return false;
    }

    // 执行打断
    ExecuteInterruption(Instigator);
    return true;
}

void UAbilityInterruptionComponent::ExecuteInterruption(AActor* Instigator)
{
    AActor* Owner = GetOwner();
    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Owner);
    if (!ASC)
    {
        return;
    }

    // 取消所有激活的技能（或者只取消特定技能）
    ASC->CancelAbilities();

    // 添加打断状态标签
    FGameplayTag InterruptedTag = FGameplayTag::RequestGameplayTag(FName("State.Interrupted"));
    ASC->AddLooseGameplayTag(InterruptedTag);

    // 0.1 秒后移除打断标签
    FTimerHandle TimerHandle;
    GetWorld()->GetTimerManager().SetTimer(
        TimerHandle,
        [ASC, InterruptedTag]()
        {
            if (ASC)
            {
                ASC->RemoveLooseGameplayTag(InterruptedTag);
            }
        },
        0.1f,
        false
    );

    UE_LOG(LogTemp, Log, TEXT("Ability interrupted by %s"), Instigator ? *Instigator->GetName() : TEXT("Unknown"));

    // 清除当前技能信息
    CurrentAbilityClass = nullptr;
    CurrentAbilityPriority = EAbilityPriority::Low;
    bCurrentAbilityCanBeInterrupted = true;
}
```

#### 1.3.2 在技能中集成打断系统

```cpp
// 修改 ComboAbility，添加打断支持
void UComboAbility::ActivateAbility(...)
{
    // ... 原有代码 ...

    // 注册到打断系统
    AActor* Owner = GetAvatarActorFromActorInfo();
    if (Owner)
    {
        UAbilityInterruptionComponent* InterruptComp = Owner->FindComponentByClass<UAbilityInterruptionComponent>();
        if (InterruptComp)
        {
            InterruptComp->RegisterActiveAbility(
                GetClass(),
                EAbilityPriority::Normal,  // 普通攻击为 Normal 优先级
                true                        // 可以被打断
            );
        }
    }
}

void UComboAbility::EndAbility(...)
{
    // 从打断系统注销
    AActor* Owner = GetAvatarActorFromActorInfo();
    if (Owner)
    {
        UAbilityInterruptionComponent* InterruptComp = Owner->FindComponentByClass<UAbilityInterruptionComponent>();
        if (InterruptComp)
        {
            InterruptComp->UnregisterActiveAbility(GetClass());
        }
    }

    Super::EndAbility(...);
}
```

### 1.4 Cooldown 与 Cost 高级管理

#### 1.4.1 动态 Cooldown 系统

```cpp
// DynamicCooldownEffect.h
UCLASS()
class UGE_DynamicCooldown : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UGE_DynamicCooldown()
    {
        DurationPolicy = EGameplayEffectDurationType::HasDuration;

        // 使用 SetByCaller 动态设置冷却时间
        DurationMagnitude = FGameplayEffectModifierMagnitude(
            FSetByCallerFloat{FGameplayTag::RequestGameplayTag(FName("Data.Cooldown.Duration"))}
        );

        // 添加冷却标签
        InheritableGameplayEffectTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Cooldown"))
        );
    }
};

// 在技能中动态设置冷却时间
void UMyAbility::ApplyCooldown(float CustomCooldownDuration)
{
    if (CooldownGameplayEffectClass)
    {
        FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(
            CooldownGameplayEffectClass,
            GetAbilityLevel()
        );

        if (SpecHandle.IsValid())
        {
            // 动态设置冷却时间（例如根据角色属性计算）
            float FinalCooldown = CustomCooldownDuration * GetCooldownReductionMultiplier();
            
            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(FName("Data.Cooldown.Duration")),
                FinalCooldown
            );

            ApplyGameplayEffectSpecToOwner(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, SpecHandle);
        }
    }
}

float UMyAbility::GetCooldownReductionMultiplier() const
{
    // 从角色属性获取冷却缩减
    UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();
    if (!ASC)
    {
        return 1.0f;
    }

    const UMyAttributeSet* Attributes = ASC->GetSet<UMyAttributeSet>();
    if (!Attributes)
    {
        return 1.0f;
    }

    // 假设有 CooldownReduction 属性（百分比，例如 20 表示 20% 冷却缩减）
    float CooldownReduction = Attributes->GetCooldownReduction();
    return 1.0f - (CooldownReduction / 100.0f);
}
```

#### 1.4.2 复杂 Cost 系统（多资源消耗）

```cpp
// MultiResourceCost.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffect.h"
#include "MultiResourceCost.generated.h"

/**
 * 多资源消耗 Effect
 * 支持同时消耗生命值、法力值、耐力等多种资源
 */
UCLASS()
class UGE_MultiResourceCost : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UGE_MultiResourceCost()
    {
        DurationPolicy = EGameplayEffectDurationType::Instant;

        // 生命值消耗
        FGameplayModifierInfo HealthCost;
        HealthCost.Attribute = UMyAttributeSet::GetHealthAttribute();
        HealthCost.ModifierOp = EGameplayModOp::Additive;
        HealthCost.ModifierMagnitude = FGameplayEffectModifierMagnitude(
            FSetByCallerFloat{FGameplayTag::RequestGameplayTag(FName("Data.Cost.Health"))}
        );
        Modifiers.Add(HealthCost);

        // 法力值消耗
        FGameplayModifierInfo ManaCost;
        ManaCost.Attribute = UMyAttributeSet::GetManaAttribute();
        ManaCost.ModifierOp = EGameplayModOp::Additive;
        ManaCost.ModifierMagnitude = FGameplayEffectModifierMagnitude(
            FSetByCallerFloat{FGameplayTag::RequestGameplayTag(FName("Data.Cost.Mana"))}
        );
        Modifiers.Add(ManaCost);

        // 耐力消耗
        FGameplayModifierInfo StaminaCost;
        StaminaCost.Attribute = UMyAttributeSet::GetStaminaAttribute();
        StaminaCost.ModifierOp = EGameplayModOp::Additive;
        StaminaCost.ModifierMagnitude = FGameplayEffectModifierMagnitude(
            FSetByCallerFloat{FGameplayTag::RequestGameplayTag(FName("Data.Cost.Stamina"))}
        );
        Modifiers.Add(StaminaCost);
    }
};
```

```cpp
// 在技能中使用多资源消耗
bool UMyAbility::CheckCost(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, OUT FGameplayTagContainer* OptionalRelevantTags) const
{
    // 首先调用父类检查
    if (!Super::CheckCost(Handle, ActorInfo, OptionalRelevantTags))
    {
        return false;
    }

    // 自定义资源检查
    UAbilitySystemComponent* ASC = ActorInfo->AbilitySystemComponent.Get();
    if (!ASC)
    {
        return false;
    }

    const UMyAttributeSet* Attributes = ASC->GetSet<UMyAttributeSet>();
    if (!Attributes)
    {
        return false;
    }

    // 检查生命值是否足够
    if (Attributes->GetHealth() < HealthCost)
    {
        return false;
    }

    // 检查法力值是否足够
    if (Attributes->GetMana() < ManaCost)
    {
        return false;
    }

    // 检查耐力是否足够
    if (Attributes->GetStamina() < StaminaCost)
    {
        return false;
    }

    return true;
}

void UMyAbility::ApplyCost(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo) const
{
    if (CostGameplayEffectClass)
    {
        FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(CostGameplayEffectClass, GetAbilityLevel());
        
        if (SpecHandle.IsValid())
        {
            // 设置各项资源消耗
            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(FName("Data.Cost.Health")),
                -HealthCost  // 负值表示消耗
            );

            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(FName("Data.Cost.Mana")),
                -ManaCost
            );

            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(FName("Data.Cost.Stamina")),
                -StaminaCost
            );

            ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, SpecHandle);
        }
    }
}
```

## 2. 战斗系统实战

### 2.1 完整伤害计算系统

一个完善的伤害计算系统需要考虑：
- 基础伤害
- 属性加成（攻击力、法术强度）
- 暴击系统
- 护甲减免
- 伤害类型（物理、魔法、真实）
- 伤害加深/减免

#### 2.1.1 自定义伤害计算 Execution

```cpp
// DamageExecution.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffectExecutionCalculation.h"
#include "DamageExecution.generated.h"

/**
 * 伤害计算枚举
 */
UENUM(BlueprintType)
enum class EDamageType : uint8
{
    Physical,    // 物理伤害（受护甲影响）
    Magical,     // 魔法伤害（受魔抗影响）
    True         // 真实伤害（无视防御）
};

/**
 * 完整的伤害计算类
 * 处理暴击、护甲、伤害类型等复杂逻辑
 */
UCLASS()
class LYRAGAME_API UDamageExecution : public UGameplayEffectExecutionCalculation
{
    GENERATED_BODY()

public:
    UDamageExecution();

    virtual void Execute_Implementation(
        const FGameplayEffectCustomExecutionParameters& ExecutionParams,
        OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput
    ) const override;

protected:
    // 计算暴击
    float CalculateCriticalHit(float BaseDamage, float CritChance, float CritMultiplier) const;

    // 计算护甲减免
    float ApplyArmorReduction(float Damage, float Armor, EDamageType DamageType) const;

    // 计算最终伤害
    float CalculateFinalDamage(
        float BaseDamage,
        float AttackPower,
        float CritChance,
        float CritMultiplier,
        float Armor,
        float MagicResistance,
        EDamageType DamageType,
        bool& bOutIsCritical
    ) const;
};
```

```cpp
// DamageExecution.cpp
#include "DamageExecution.h"
#include "MyAttributeSet.h"
#include "AbilitySystemComponent.h"

// 定义属性捕获
struct FDamageStatics
{
    // 攻击者属性
    DECLARE_ATTRIBUTE_CAPTUREDEF(AttackPower);
    DECLARE_ATTRIBUTE_CAPTUREDEF(CriticalChance);
    DECLARE_ATTRIBUTE_CAPTUREDEF(CriticalMultiplier);

    // 目标属性
    DECLARE_ATTRIBUTE_CAPTUREDEF(Armor);
    DECLARE_ATTRIBUTE_CAPTUREDEF(MagicResistance);
    DECLARE_ATTRIBUTE_CAPTUREDEF(Health);

    FDamageStatics()
    {
        // 捕获攻击者的属性（Source）
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, AttackPower, Source, false);
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, CriticalChance, Source, false);
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, CriticalMultiplier, Source, false);

        // 捕获目标的属性（Target）
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, Armor, Target, false);
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, MagicResistance, Target, false);
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, Health, Target, false);
    }
};

static const FDamageStatics& DamageStatics()
{
    static FDamageStatics DStatics;
    return DStatics;
}

UDamageExecution::UDamageExecution()
{
    // 注册需要捕获的属性
    RelevantAttributesToCapture.Add(DamageStatics().AttackPowerDef);
    RelevantAttributesToCapture.Add(DamageStatics().CriticalChanceDef);
    RelevantAttributesToCapture.Add(DamageStatics().CriticalMultiplierDef);
    RelevantAttributesToCapture.Add(DamageStatics().ArmorDef);
    RelevantAttributesToCapture.Add(DamageStatics().MagicResistanceDef);
    RelevantAttributesToCapture.Add(DamageStatics().HealthDef);
}

void UDamageExecution::Execute_Implementation(
    const FGameplayEffectCustomExecutionParameters& ExecutionParams,
    OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

    // 获取Source和Target的ASC
    UAbilitySystemComponent* SourceASC = ExecutionParams.GetSourceAbilitySystemComponent();
    UAbilitySystemComponent* TargetASC = ExecutionParams.GetTargetAbilitySystemComponent();

    if (!SourceASC || !TargetASC)
    {
        return;
    }

    AActor* SourceActor = SourceASC->GetAvatarActor();
    AActor* TargetActor = TargetASC->GetAvatarActor();

    const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    FAggregatorEvaluateParameters EvaluationParameters;
    EvaluationParameters.SourceTags = SourceTags;
    EvaluationParameters.TargetTags = TargetTags;

    // 捕获属性值
    float AttackPower = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().AttackPowerDef, EvaluationParameters, AttackPower);

    float CritChance = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().CriticalChanceDef, EvaluationParameters, CritChance);

    float CritMultiplier = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().CriticalMultiplierDef, EvaluationParameters, CritMultiplier);

    float Armor = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().ArmorDef, EvaluationParameters, Armor);

    float MagicResistance = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().MagicResistanceDef, EvaluationParameters, MagicResistance);

    // 从 SetByCaller 获取基础伤害和伤害类型
    float BaseDamage = Spec.GetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag(FName("Data.Damage.Base")),
        false,
        0.0f
    );

    float DamageTypeValue = Spec.GetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag(FName("Data.Damage.Type")),
        false,
        0.0f  // 默认物理伤害
    );

    EDamageType DamageType = static_cast<EDamageType>(FMath::RoundToInt(DamageTypeValue));

    // 计算最终伤害
    bool bIsCritical = false;
    float FinalDamage = CalculateFinalDamage(
        BaseDamage,
        AttackPower,
        CritChance,
        CritMultiplier,
        Armor,
        MagicResistance,
        DamageType,
        bIsCritical
    );

    // 添加暴击标签（用于UI显示）
    if (bIsCritical)
    {
        FGameplayTag CriticalTag = FGameplayTag::RequestGameplayTag(FName("Event.Damage.Critical"));
        OutExecutionOutput.AddOutputTag(CriticalTag);
    }

    // 输出伤害修改
    if (FinalDamage > 0.0f)
    {
        OutExecutionOutput.AddOutputModifier(
            FGameplayModifierEvaluatedData(
                DamageStatics().HealthProperty,
                EGameplayModOp::Additive,
                -FinalDamage  // 负值表示扣血
            )
        );
    }

    UE_LOG(LogTemp, Log, TEXT("Damage Calculation: Base=%.1f, AttackPower=%.1f, Final=%.1f, IsCrit=%s"),
           BaseDamage, AttackPower, FinalDamage, bIsCritical ? TEXT("true") : TEXT("false"));
}

float UDamageExecution::CalculateCriticalHit(float BaseDamage, float CritChance, float CritMultiplier) const
{
    // 暴击率转换为 0-1 范围
    float CritRoll = FMath::FRand();
    bool bIsCritical = CritRoll <= (CritChance / 100.0f);

    if (bIsCritical)
    {
        return BaseDamage * CritMultiplier;
    }

    return BaseDamage;
}

float UDamageExecution::ApplyArmorReduction(float Damage, float Armor, EDamageType DamageType) const
{
    // 真实伤害无视防御
    if (DamageType == EDamageType::True)
    {
        return Damage;
    }

    // 护甲公式：减伤% = Armor / (Armor + 100)
    // 例如：100 护甲 = 50% 减伤，200 护甲 = 66.7% 减伤
    float DamageReduction = Armor / (Armor + 100.0f);
    float ReducedDamage = Damage * (1.0f - DamageReduction);

    return FMath::Max(ReducedDamage, Damage * 0.1f);  // 至少造成 10% 伤害
}

float UDamageExecution::CalculateFinalDamage(
    float BaseDamage,
    float AttackPower,
    float CritChance,
    float CritMultiplier,
    float Armor,
    float MagicResistance,
    EDamageType DamageType,
    bool& bOutIsCritical) const
{
    // 第一步：加上攻击力加成
    float TotalDamage = BaseDamage + AttackPower;

    // 第二步：计算暴击
    float CritRoll = FMath::FRand();
    bOutIsCritical = CritRoll <= (CritChance / 100.0f);

    if (bOutIsCritical)
    {
        TotalDamage *= CritMultiplier;
    }

    // 第三步：应用防御减免
    float DefensiveAttribute = (DamageType == EDamageType::Physical) ? Armor : MagicResistance;
    float FinalDamage = ApplyArmorReduction(TotalDamage, DefensiveAttribute, DamageType);

    return FMath::Max(FinalDamage, 1.0f);  // 至少造成 1 点伤害
}
```

#### 2.1.2 伤害 Gameplay Effect

```cpp
// GE_Damage.h
UCLASS()
class UGE_Damage : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UGE_Damage()
    {
        DurationPolicy = EGameplayEffectDurationType::Instant;

        // 使用自定义伤害计算
        FGameplayEffectExecutionDefinition DamageExecution;
        DamageExecution.CalculationClass = UDamageExecution::StaticClass();
        Executions.Add(DamageExecution);
    }
};
```

### 2.2 Buff/Debuff 系统实现

Buff 和 Debuff 是游戏战斗系统的核心，需要支持：
- 持续时间和堆叠
- 周期性效果（DOT/HOT）
- 条件触发（生命值低于50%时激活）
- 效果覆盖与刷新

#### 2.2.1 Buff 基类设计

```cpp
// BuffEffectBase.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffect.h"
#include "BuffEffectBase.generated.h"

/**
 * Buff 堆叠策略
 */
UENUM(BlueprintType)
enum class EBuffStackingPolicy : uint8
{
    None,               // 不可堆叠（刷新持续时间）
    Stacks,             // 可堆叠（增加层数）
    Independent         // 独立实例（每个独立计时）
};

/**
 * Buff 效果基类
 * 所有 Buff/Debuff 继承自此类
 */
UCLASS(Abstract, BlueprintType)
class LYRAGAME_API UBuffEffectBase : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UBuffEffectBase();

    // Buff 显示名称
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Buff Info")
    FText BuffName;

    // Buff 描述
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Buff Info")
    FText BuffDescription;

    // Buff 图标
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Buff Info")
    UTexture2D* BuffIcon;

    // 是否为增益效果
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Buff Info")
    bool bIsBeneficial;

    // 堆叠策略
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Buff Stacking")
    EBuffStackingPolicy StackingPolicy;

    // 最大堆叠层数（仅当 StackingPolicy 为 Stacks 时有效）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Buff Stacking")
    int32 MaxStackCount;

protected:
    void SetupBuffTags();
};
```

```cpp
// BuffEffectBase.cpp
#include "BuffEffectBase.h"

UBuffEffectBase::UBuffEffectBase()
{
    // 默认配置
    DurationPolicy = EGameplayEffectDurationType::HasDuration;
    bIsBeneficial = true;
    StackingPolicy = EBuffStackingPolicy::None;
    MaxStackCount = 1;

    // 添加 Buff 标签
    InheritableGameplayEffectTags.Added.AddTag(
        FGameplayTag::RequestGameplayTag(FName("Effect.Buff"))
    );
}

void UBuffEffectBase::SetupBuffTags()
{
    // 根据 StackingPolicy 设置堆叠规则
    switch (StackingPolicy)
    {
    case EBuffStackingPolicy::None:
        // 不堆叠：新的覆盖旧的
        StackingType = EGameplayEffectStackingType::AggregateBySource;
        StackLimitCount = 1;
        StackDurationRefreshPolicy = EGameplayEffectStackingDurationPolicy::RefreshOnSuccessfulApplication;
        StackPeriodResetPolicy = EGameplayEffectStackingPeriodPolicy::ResetOnSuccessfulApplication;
        break;

    case EBuffStackingPolicy::Stacks:
        // 可堆叠
        StackingType = EGameplayEffectStackingType::AggregateBySource;
        StackLimitCount = MaxStackCount;
        StackDurationRefreshPolicy = EGameplayEffectStackingDurationPolicy::RefreshOnSuccessfulApplication;
        StackPeriodResetPolicy = EGameplayEffectStackingPeriodPolicy::ResetOnSuccessfulApplication;
        break;

    case EBuffStackingPolicy::Independent:
        // 独立实例
        StackingType = EGameplayEffectStackingType::None;
        break;
    }
}
```

#### 2.2.2 具体 Buff 示例

**1. 攻击力增益 Buff**

```cpp
// GE_Buff_AttackPower.h
UCLASS()
class UGE_Buff_AttackPower : public UBuffEffectBase
{
    GENERATED_BODY()

public:
    UGE_Buff_AttackPower()
    {
        BuffName = FText::FromString(TEXT("Attack Power Up"));
        BuffDescription = FText::FromString(TEXT("Increases attack power by 50%."));
        bIsBeneficial = true;

        // 持续 10 秒
        DurationMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(10.0f));

        // 可堆叠，最多 3 层
        StackingPolicy = EBuffStackingPolicy::Stacks;
        MaxStackCount = 3;
        SetupBuffTags();

        // 修改攻击力
        FGameplayModifierInfo AttackModifier;
        AttackModifier.Attribute = UMyAttributeSet::GetAttackPowerAttribute();
        AttackModifier.ModifierOp = EGameplayModOp::Multiply;
        AttackModifier.ModifierMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(1.5f)); // 150% 攻击力
        Modifiers.Add(AttackModifier);

        // 添加标签
        InheritableGameplayEffectTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Buff.AttackPower"))
        );
    }
};
```

**2. 持续伤害 Debuff (DOT)**

```cpp
// GE_Debuff_Poison.h
UCLASS()
class UGE_Debuff_Poison : public UBuffEffectBase
{
    GENERATED_BODY()

public:
    UGE_Debuff_Poison()
    {
        BuffName = FText::FromString(TEXT("Poison"));
        BuffDescription = FText::FromString(TEXT("Deals 10 damage per second for 5 seconds."));
        bIsBeneficial = false;

        // 持续 5 秒
        DurationMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(5.0f));

        // 周期性效果，每 1 秒触发一次
        Period = FScalableFloat(1.0f);
        bExecutePeriodicEffectOnApplication = true;  // 应用时立即执行一次

        // 不可堆叠，刷新持续时间
        StackingPolicy = EBuffStackingPolicy::None;
        SetupBuffTags();

        // 周期性扣血
        FGameplayModifierInfo PoisonDamage;
        PoisonDamage.Attribute = UMyAttributeSet::GetHealthAttribute();
        PoisonDamage.ModifierOp = EGameplayModOp::Additive;
        PoisonDamage.ModifierMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(-10.0f)); // 每秒 -10 生命
        Modifiers.Add(PoisonDamage);

        // 添加标签
        InheritableGameplayEffectTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Debuff.Poison"))
        );
    }
};
```

**3. 条件激活的 Buff（低血量时激活）**

```cpp
// GE_Buff_LastStand.h
UCLASS()
class UGE_Buff_LastStand : public UBuffEffectBase
{
    GENERATED_BODY()

public:
    UGE_Buff_LastStand()
    {
        BuffName = FText::FromString(TEXT("Last Stand"));
        BuffDescription = FText::FromString(TEXT("When below 30% health, gain 50% damage reduction."));
        bIsBeneficial = true;

        // 永久效果（但会根据条件自动移除）
        DurationPolicy = EGameplayEffectDurationType::Infinite;

        // 条件：生命值低于 30%
        FGameplayEffectExecutionScopedModifierInfo ConditionMod;
        // （这里需要自定义 Execution 来检查生命值条件）
        
        // 伤害减免
        FGameplayModifierInfo DamageReduction;
        DamageReduction.Attribute = UMyAttributeSet::GetDamageResistanceAttribute();
        DamageReduction.ModifierOp = EGameplayModOp::Additive;
        DamageReduction.ModifierMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(0.5f)); // +50% 抗性
        Modifiers.Add(DamageReduction);

        // 添加标签
        InheritableGameplayEffectTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Buff.LastStand"))
        );

        // 应用条件：生命值 < 30%
        FGameplayTagRequirements ApplicationTagRequirements;
        ApplicationTagRequirements.RequireTags.AddTag(
            FGameplayTag::RequestGameplayTag(FName("State.LowHealth"))
        );
        ApplicationTagRequirements;
    }
};
```

#### 2.2.3 Buff 管理组件

```cpp
// BuffManagerComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "GameplayEffectTypes.h"
#include "BuffManagerComponent.generated.h"

/**
 * Buff 信息结构
 */
USTRUCT(BlueprintType)
struct FBuffInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FActiveGameplayEffectHandle EffectHandle;

    UPROPERTY(BlueprintReadOnly)
    FText BuffName;

    UPROPERTY(BlueprintReadOnly)
    FText BuffDescription;

    UPROPERTY(BlueprintReadOnly)
    UTexture2D* BuffIcon;

    UPROPERTY(BlueprintReadOnly)
    bool bIsBeneficial;

    UPROPERTY(BlueprintReadOnly)
    float RemainingDuration;

    UPROPERTY(BlueprintReadOnly)
    int32 StackCount;
};

/**
 * Buff 管理组件
 * 追踪角色身上的所有 Buff/Debuff，提供UI显示接口
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class LYRAGAME_API UBuffManagerComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UBuffManagerComponent();

    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;

    // 获取所有激活的 Buff
    UFUNCTION(BlueprintPure, Category = "Buff")
    TArray<FBuffInfo> GetAllActiveBuffs() const;

    // 获取特定类型的 Buff
    UFUNCTION(BlueprintPure, Category = "Buff")
    TArray<FBuffInfo> GetBuffsByTag(FGameplayTag BuffTag) const;

    // 移除特定 Buff
    UFUNCTION(BlueprintCallable, Category = "Buff")
    bool RemoveBuff(FActiveGameplayEffectHandle EffectHandle);

    // 清除所有 Debuff
    UFUNCTION(BlueprintCallable, Category = "Buff")
    void ClearAllDebuffs();

    // Buff 添加/移除事件
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnBuffAdded, const FBuffInfo&, BuffInfo);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnBuffRemoved, const FBuffInfo&, BuffInfo);

    UPROPERTY(BlueprintAssignable, Category = "Buff")
    FOnBuffAdded OnBuffAdded;

    UPROPERTY(BlueprintAssignable, Category = "Buff")
    FOnBuffRemoved OnBuffRemoved;

protected:
    virtual void BeginPlay() override;

    void OnGameplayEffectApplied(UAbilitySystemComponent* ASC, const FGameplayEffectSpec& Spec, FActiveGameplayEffectHandle Handle);
    void OnGameplayEffectRemoved(const FActiveGameplayEffect& RemovedEffect);

private:
    UPROPERTY()
    UAbilitySystemComponent* CachedASC;

    void BindToASC();
    FBuffInfo CreateBuffInfo(const FActiveGameplayEffect& ActiveEffect) const;
};
```

```cpp
// BuffManagerComponent.cpp
#include "BuffManagerComponent.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemGlobals.h"
#include "BuffEffectBase.h"

UBuffManagerComponent::UBuffManagerComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
    CachedASC = nullptr;
}

void UBuffManagerComponent::BeginPlay()
{
    Super::BeginPlay();
    BindToASC();
}

void UBuffManagerComponent::BindToASC()
{
    AActor* Owner = GetOwner();
    if (!Owner)
    {
        return;
    }

    CachedASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Owner);
    if (!CachedASC)
    {
        return;
    }

    // 绑定 Gameplay Effect 应用/移除回调
    CachedASC->OnGameplayEffectAppliedDelegateToSelf.AddUObject(
        this,
        &UBuffManagerComponent::OnGameplayEffectApplied
    );

    CachedASC->OnAnyGameplayEffectRemovedDelegate().AddUObject(
        this,
        &UBuffManagerComponent::OnGameplayEffectRemoved
    );
}

void UBuffManagerComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    // 定期更新可以在这里处理，例如更新 UI
}

TArray<FBuffInfo> UBuffManagerComponent::GetAllActiveBuffs() const
{
    TArray<FBuffInfo> Result;

    if (!CachedASC)
    {
        return Result;
    }

    // 遍历所有激活的 Gameplay Effect
    const FActiveGameplayEffectsContainer& ActiveEffects = CachedASC->GetActiveGameplayEffects();
    for (const FActiveGameplayEffect& ActiveEffect : ActiveEffects)
    {
        // 检查是否为 Buff（通过标签判断）
        if (ActiveEffect.Spec.Def->InheritableGameplayEffectTags.HasTag(
            FGameplayTag::RequestGameplayTag(FName("Effect.Buff"))))
        {
            Result.Add(CreateBuffInfo(ActiveEffect));
        }
    }

    return Result;
}

TArray<FBuffInfo> UBuffManagerComponent::GetBuffsByTag(FGameplayTag BuffTag) const
{
    TArray<FBuffInfo> Result;

    if (!CachedASC)
    {
        return Result;
    }

    const FActiveGameplayEffectsContainer& ActiveEffects = CachedASC->GetActiveGameplayEffects();
    for (const FActiveGameplayEffect& ActiveEffect : ActiveEffects)
    {
        if (ActiveEffect.Spec.Def->InheritableGameplayEffectTags.HasTag(BuffTag))
        {
            Result.Add(CreateBuffInfo(ActiveEffect));
        }
    }

    return Result;
}

bool UBuffManagerComponent::RemoveBuff(FActiveGameplayEffectHandle EffectHandle)
{
    if (!CachedASC || !EffectHandle.IsValid())
    {
        return false;
    }

    return CachedASC->RemoveActiveGameplayEffect(EffectHandle);
}

void UBuffManagerComponent::ClearAllDebuffs()
{
    if (!CachedASC)
    {
        return;
    }

    // 获取所有 Debuff 标签的 Effect 并移除
    FGameplayTag DebuffTag = FGameplayTag::RequestGameplayTag(FName("Debuff"));
    CachedASC->RemoveActiveEffectsWithTags(FGameplayTagContainer(DebuffTag));
}

FBuffInfo UBuffManagerComponent::CreateBuffInfo(const FActiveGameplayEffect& ActiveEffect) const
{
    FBuffInfo Info;
    Info.EffectHandle = ActiveEffect.Handle;
    Info.StackCount = ActiveEffect.Spec.GetStackCount();

    // 从 Effect Definition 获取 Buff 信息
    const UBuffEffectBase* BuffDef = Cast<UBuffEffectBase>(ActiveEffect.Spec.Def);
    if (BuffDef)
    {
        Info.BuffName = BuffDef->BuffName;
        Info.BuffDescription = BuffDef->BuffDescription;
        Info.BuffIcon = BuffDef->BuffIcon;
        Info.bIsBeneficial = BuffDef->bIsBeneficial;
    }

    // 获取剩余持续时间
    float Duration = ActiveEffect.GetDuration();
    float TimeApplied = ActiveEffect.GetTimeApplied(GetWorld()->GetTimeSeconds());
    Info.RemainingDuration = Duration - (GetWorld()->GetTimeSeconds() - TimeApplied);

    return Info;
}

void UBuffManagerComponent::OnGameplayEffectApplied(
    UAbilitySystemComponent* ASC,
    const FGameplayEffectSpec& Spec,
    FActiveGameplayEffectHandle Handle)
{
    // 检查是否为 Buff
    if (Spec.Def->InheritableGameplayEffectTags.HasTag(
        FGameplayTag::RequestGameplayTag(FName("Effect.Buff"))))
    {
        const FActiveGameplayEffect* ActiveEffect = ASC->GetActiveGameplayEffect(Handle);
        if (ActiveEffect)
        {
            FBuffInfo BuffInfo = CreateBuffInfo(*ActiveEffect);
            OnBuffAdded.Broadcast(BuffInfo);
        }
    }
}

void UBuffManagerComponent::OnGameplayEffectRemoved(const FActiveGameplayEffect& RemovedEffect)
{
    if (RemovedEffect.Spec.Def->InheritableGameplayEffectTags.HasTag(
        FGameplayTag::RequestGameplayTag(FName("Effect.Buff"))))
    {
        FBuffInfo BuffInfo = CreateBuffInfo(RemovedEffect);
        OnBuffRemoved.Broadcast(BuffInfo);
    }
}
```

### 2.3 控制效果系统（CC）

控制效果（Crowd Control）包括眩晕、沉默、定身、缴械等，需要阻止玩家的特定行为。

#### 2.3.1 控制效果基类

```cpp
// GE_ControlEffect_Base.h
#pragma once

#include "CoreMinimal.h"
#include "BuffEffectBase.h"
#include "GE_ControlEffect_Base.generated.h"

/**
 * 控制效果类型
 */
UENUM(BlueprintType)
enum class EControlEffectType : uint8
{
    Stun,          // 眩晕（无法移动、攻击、使用技能）
    Silence,       // 沉默（无法使用技能，可以移动和普攻）
    Root,          // 定身（无法移动，可以攻击和使用技能）
    Disarm,        // 缴械（无法普攻，可以移动和使用技能）
    Slow           // 减速（移动速度降低）
};

/**
 * 控制效果基类
 */
UCLASS(Abstract)
class LYRAGAME_API UGE_ControlEffect_Base : public UBuffEffectBase
{
    GENERATED_BODY()

public:
    UGE_ControlEffect_Base();

    // 控制效果类型
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Control Effect")
    EControlEffectType ControlType;

protected:
    void SetupControlTags();
};
```

```cpp
// GE_ControlEffect_Base.cpp
#include "GE_ControlEffect_Base.h"

UGE_ControlEffect_Base::UGE_ControlEffect_Base()
{
    bIsBeneficial = false;
    StackingPolicy = EBuffStackingPolicy::None;  // 控制效果一般不堆叠

    // 添加 Debuff 标签
    InheritableGameplayEffectTags.Added.AddTag(
        FGameplayTag::RequestGameplayTag(FName("Debuff"))
    );
}

void UGE_ControlEffect_Base::SetupControlTags()
{
    switch (ControlType)
    {
    case EControlEffectType::Stun:
        // 眩晕：阻止所有行动
        InheritableGameplayEffectTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("State.Stunned"))
        );
        GrantedApplicationImmunityTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Ability"))
        );
        break;

    case EControlEffectType::Silence:
        // 沉默：阻止技能
        InheritableGameplayEffectTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("State.Silenced"))
        );
        GrantedApplicationImmunityTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Ability.Skill"))
        );
        break;

    case EControlEffectType::Root:
        // 定身：阻止移动
        InheritableGameplayEffectTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("State.Rooted"))
        );
        GrantedApplicationImmunityTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Ability.Movement"))
        );
        break;

    case EControlEffectType::Disarm:
        // 缴械：阻止普攻
        InheritableGameplayEffectTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("State.Disarmed"))
        );
        GrantedApplicationImmunityTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Ability.Melee"))
        );
        break;

    case EControlEffectType::Slow:
        // 减速（在具体子类中修改移动速度属性）
        InheritableGameplayEffectTags.Added.AddTag(
            FGameplayTag::RequestGameplayTag(FName("State.Slowed"))
        );
        break;
    }
}
```

#### 2.3.2 具体控制效果示例

**1. 眩晕效果**

```cpp
// GE_Stun.h
UCLASS()
class UGE_Stun : public UGE_ControlEffect_Base
{
    GENERATED_BODY()

public:
    UGE_Stun()
    {
        BuffName = FText::FromString(TEXT("Stunned"));
        BuffDescription = FText::FromString(TEXT("Cannot move, attack, or use abilities."));
        ControlType = EControlEffectType::Stun;

        // 持续 2 秒
        DurationMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(2.0f));

        SetupControlTags();

        // 眩晕期间阻止所有技能激活
        ApplicationTagRequirements.IgnoreTags.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Ability"))
        );
    }
};
```

**2. 减速效果**

```cpp
// GE_Slow.h
UCLASS()
class UGE_Slow : public UGE_ControlEffect_Base
{
    GENERATED_BODY()

public:
    UGE_Slow()
    {
        BuffName = FText::FromString(TEXT("Slowed"));
        BuffDescription = FText::FromString(TEXT("Movement speed reduced by 50%."));
        ControlType = EControlEffectType::Slow;

        // 持续 3 秒
        DurationMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(3.0f));

        SetupControlTags();

        // 降低移动速度
        FGameplayModifierInfo SpeedModifier;
        SpeedModifier.Attribute = UMyAttributeSet::GetMovementSpeedAttribute();
        SpeedModifier.ModifierOp = EGameplayModOp::Multiply;
        SpeedModifier.ModifierMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(0.5f)); // 50% 速度
        Modifiers.Add(SpeedModifier);
    }
};
```

#### 2.3.3 控制效果免疫系统

```cpp
// GE_ControlImmunity.h
UCLASS()
class UGE_ControlImmunity : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UGE_ControlImmunity()
    {
        // 永久效果（或设置持续时间）
        DurationPolicy = EGameplayEffectDurationType::Infinite;

        // 授予免疫标签，阻止控制效果应用
        GrantedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Status.ControlImmune")));

        // 阻止所有控制效果
        ApplicationTagRequirements.IgnoreTags.AddTag(
            FGameplayTag::RequestGameplayTag(FName("State.Stunned"))
        );
        ApplicationTagRequirements.IgnoreTags.AddTag(
            FGameplayTag::RequestGameplayTag(FName("State.Silenced"))
        );
        ApplicationTagRequirements.IgnoreTags.AddTag(
            FGameplayTag::RequestGameplayTag(FName("State.Rooted"))
        );
    }
};
```

### 2.4 完整战斗循环实现

现在让我们将所有系统整合到一起，构建一个完整的战斗循环。

#### 2.4.1 战斗角色基类

```cpp
// CombatCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "Character/LyraCharacter.h"
#include "AbilitySystemInterface.h"
#include "CombatCharacter.generated.h"

/**
 * 战斗角色基类
 * 集成所有战斗系统组件
 */
UCLASS()
class LYRAGAME_API ACombatCharacter : public ALyraCharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    ACombatCharacter();

    virtual void PossessedBy(AController* NewController) override;
    virtual void OnRep_PlayerState() override;

    // IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

    // 授予初始技能
    void GrantInitialAbilities();

    // 初始化属性
    void InitializeAttributes();

protected:
    virtual void BeginPlay() override;

    // Ability System Component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
    UAbilitySystemComponent* AbilitySystemComponent;

    // Attribute Set
    UPROPERTY()
    UMyAttributeSet* AttributeSet;

    // 连招管理器
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Combat")
    UComboManagerComponent* ComboManager;

    // Buff 管理器
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Combat")
    UBuffManagerComponent* BuffManager;

    // 打断系统
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Combat")
    UAbilityInterruptionComponent* InterruptionComponent;

    // 初始技能列表
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Abilities")
    TArray<TSubclassOf<UGameplayAbility>> InitialAbilities;

    // 初始属性 Effect
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Abilities")
    TSubclassOf<UGameplayEffect> DefaultAttributesEffect;

    // 被动技能/永久 Buff
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Abilities")
    TArray<TSubclassOf<UGameplayEffect>> PassiveEffects;

private:
    void SetupAbilitySystemComponent();
};
```

```cpp
// CombatCharacter.cpp
#include "CombatCharacter.h"
#include "AbilitySystemComponent.h"
#include "ComboManagerComponent.h"
#include "BuffManagerComponent.h"
#include "AbilityInterruptionComponent.h"
#include "MyAttributeSet.h"

ACombatCharacter::ACombatCharacter()
{
    // 创建 Ability System Component
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

    // 创建 Attribute Set
    AttributeSet = CreateDefaultSubobject<UMyAttributeSet>(TEXT("AttributeSet"));

    // 创建战斗组件
    ComboManager = CreateDefaultSubobject<UComboManagerComponent>(TEXT("ComboManager"));
    BuffManager = CreateDefaultSubobject<UBuffManagerComponent>(TEXT("BuffManager"));
    InterruptionComponent = CreateDefaultSubobject<UAbilityInterruptionComponent>(TEXT("InterruptionComponent"));
}

void ACombatCharacter::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        SetupAbilitySystemComponent();
    }
}

void ACombatCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);

    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->InitAbilityActorInfo(this, this);
        GrantInitialAbilities();
        InitializeAttributes();
    }
}

void ACombatCharacter::OnRep_PlayerState()
{
    Super::OnRep_PlayerState();

    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->InitAbilityActorInfo(this, this);
    }
}

UAbilitySystemComponent* ACombatCharacter::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}

void ACombatCharacter::SetupAbilitySystemComponent()
{
    if (!AbilitySystemComponent)
    {
        return;
    }

    // 设置连招链（示例）
    if (ComboManager)
    {
        // 创建轻攻击连招
        FComboChain LightAttackChain;
        LightAttackChain.ComboName = TEXT("LightAttack");
        LightAttackChain.ComboResetTimeout = 2.0f;

        // 添加 3 段连击
        for (int32 i = 0; i < 3; ++i)
        {
            FComboSegment Segment;
            Segment.SegmentIndex = i;
            // Segment.AbilityClass = ... (设置对应的技能类)
            Segment.ComboWindowStart = 0.4f;
            Segment.ComboWindowDuration = 0.3f;
            Segment.DamageMultiplier = 1.0f + (i * 0.2f);  // 1.0, 1.2, 1.4

            if (i < 2)  // 前两段可以连接到下一段
            {
                Segment.NextSegments.Add(i + 1);
            }

            LightAttackChain.Segments.Add(Segment);
        }

        FGameplayTag LightAttackTag = FGameplayTag::RequestGameplayTag(FName("Ability.Melee.LightAttack"));
        ComboManager->ComboChains.Add(LightAttackTag, LightAttackChain);
    }
}

void ACombatCharacter::GrantInitialAbilities()
{
    if (!HasAuthority() || !AbilitySystemComponent)
    {
        return;
    }

    // 授予初始技能
    for (TSubclassOf<UGameplayAbility>& AbilityClass : InitialAbilities)
    {
        if (AbilityClass)
        {
            FGameplayAbilitySpec AbilitySpec(AbilityClass, 1, INDEX_NONE, this);
            AbilitySystemComponent->GiveAbility(AbilitySpec);
        }
    }

    // 应用被动效果
    for (TSubclassOf<UGameplayEffect>& PassiveEffect : PassiveEffects)
    {
        if (PassiveEffect)
        {
            FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
            EffectContext.AddSourceObject(this);

            FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
                PassiveEffect,
                1,
                EffectContext
            );

            if (SpecHandle.IsValid())
            {
                AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
            }
        }
    }
}

void ACombatCharacter::InitializeAttributes()
{
    if (!HasAuthority() || !AbilitySystemComponent || !DefaultAttributesEffect)
    {
        return;
    }

    FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
    EffectContext.AddSourceObject(this);

    FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
        DefaultAttributesEffect,
        1,
        EffectContext
    );

    if (SpecHandle.IsValid())
    {
        AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
    }
}
```

#### 2.4.2 战斗测试关卡

现在我们可以在测试关卡中验证整个战斗系统：

```cpp
// CombatTestGameMode.cpp
void ACombatTestGameMode::BeginPlay()
{
    Super::BeginPlay();

    // 生成两个测试角色
    FActorSpawnParameters SpawnParams;
    SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn;

    // 玩家角色
    ACombatCharacter* Player = GetWorld()->SpawnActor<ACombatCharacter>(
        PlayerCharacterClass,
        FVector(0, 0, 100),
        FRotator::ZeroRotator,
        SpawnParams
    );

    // 敌人角色
    ACombatCharacter* Enemy = GetWorld()->SpawnActor<ACombatCharacter>(
        EnemyCharacterClass,
        FVector(500, 0, 100),
        FRotator(0, 180, 0),
        SpawnParams
    );

    // 测试战斗循环
    if (Player && Enemy)
    {
        // 玩家对敌人使用蓄力攻击
        UAbilitySystemComponent* PlayerASC = Player->GetAbilitySystemComponent();
        if (PlayerASC)
        {
            FGameplayTag AbilityTag = FGameplayTag::RequestGameplayTag(FName("Ability.Melee.HeavyAttack"));
            PlayerASC->TryActivateAbilitiesByTag(FGameplayTagContainer(AbilityTag));
        }

        // 2 秒后给玩家施加攻击力 Buff
        FTimerHandle BuffTimer;
        GetWorld()->GetTimerManager().SetTimer(
            BuffTimer,
            [Player]()
            {
                UAbilitySystemComponent* ASC = Player->GetAbilitySystemComponent();
                if (ASC)
                {
                    FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
                    EffectContext.AddSourceObject(Player);

                    FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingSpec(
                        UGE_Buff_AttackPower::StaticClass(),
                        1,
                        EffectContext
                    );

                    ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
                }
            },
            2.0f,
            false
        );

        // 4 秒后给敌人施加眩晕
        FTimerHandle StunTimer;
        GetWorld()->GetTimerManager().SetTimer(
            StunTimer,
            [Player, Enemy]()
            {
                UAbilitySystemComponent* PlayerASC = Player->GetAbilitySystemComponent();
                UAbilitySystemComponent* EnemyASC = Enemy->GetAbilitySystemComponent();

                if (PlayerASC && EnemyASC)
                {
                    FGameplayEffectContextHandle EffectContext = PlayerASC->MakeEffectContext();
                    EffectContext.AddSourceObject(Player);

                    FGameplayEffectSpecHandle SpecHandle = PlayerASC->MakeOutgoingSpec(
                        UGE_Stun::StaticClass(),
                        1,
                        EffectContext
                    );

                    PlayerASC->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), EnemyASC);
                }
            },
            4.0f,
            false
        );
    }
}
```

## 3. 高级 GAS 技巧

### 3.1 自定义 Gameplay Task

Gameplay Task 是异步执行的任务节点，可以在技能执行过程中等待特定事件或条件。

#### 3.1.1 等待输入组合 Task

```cpp
// AbilityTask_WaitInputCombo.h
#pragma once

#include "CoreMinimal.h"
#include "Abilities/Tasks/AbilityTask.h"
#include "AbilityTask_WaitInputCombo.generated.h"

/**
 * 等待输入组合的 Task
 * 例如：等待玩家在特定时间窗口内按下特定按键序列
 */
UCLASS()
class LYRAGAME_API UAbilityTask_WaitInputCombo : public UAbilityTask
{
    GENERATED_BODY()

public:
    // 输入序列完成时触发
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FInputComboDelegate, const TArray<int32>&, InputSequence);

    UPROPERTY(BlueprintAssignable)
    FInputComboDelegate OnComboCompleted;

    UPROPERTY(BlueprintAssignable)
    FInputComboDelegate OnComboFailed;

    /**
     * 等待输入组合
     * @param RequiredSequence 需要的输入序列（例如 [1, 2, 3] 表示按下 1, 2, 3）
     * @param TimeWindow 每次输入之间的最大时间间隔（秒）
     */
    UFUNCTION(BlueprintCallable, Category = "Ability|Tasks", meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", BlueprintInternalUseOnly = "TRUE"))
    static UAbilityTask_WaitInputCombo* WaitInputCombo(
        UGameplayAbility* OwningAbility,
        TArray<int32> RequiredSequence,
        float TimeWindow = 1.0f
    );

    virtual void Activate() override;

protected:
    virtual void OnDestroy(bool AbilityEnded) override;

private:
    TArray<int32> RequiredInputSequence;
    TArray<int32> CurrentInputSequence;
    float InputTimeWindow;
    float LastInputTime;

    FTimerHandle TimeoutTimerHandle;

    void OnInputReceived(int32 InputID);
    void CheckCompletion();
    void ResetCombo();
    void OnTimeout();

    // 绑定到输入系统
    void BindToInputComponent();
    void UnbindFromInputComponent();
};
```

```cpp
// AbilityTask_WaitInputCombo.cpp
#include "AbilityTask_WaitInputCombo.h"
#include "AbilitySystemComponent.h"

UAbilityTask_WaitInputCombo* UAbilityTask_WaitInputCombo::WaitInputCombo(
    UGameplayAbility* OwningAbility,
    TArray<int32> RequiredSequence,
    float TimeWindow)
{
    UAbilityTask_WaitInputCombo* MyObj = NewAbilityTask<UAbilityTask_WaitInputCombo>(OwningAbility);
    MyObj->RequiredInputSequence = RequiredSequence;
    MyObj->InputTimeWindow = TimeWindow;
    MyObj->LastInputTime = 0.0f;

    return MyObj;
}

void UAbilityTask_WaitInputCombo::Activate()
{
    Super::Activate();

    BindToInputComponent();
    ResetCombo();
}

void UAbilityTask_WaitInputCombo::OnDestroy(bool AbilityEnded)
{
    UnbindFromInputComponent();

    if (GetWorld())
    {
        GetWorld()->GetTimerManager().ClearTimer(TimeoutTimerHandle);
    }

    Super::OnDestroy(AbilityEnded);
}

void UAbilityTask_WaitInputCombo::BindToInputComponent()
{
    // 绑定到输入组件（简化示例，实际需要根据项目的输入系统）
    if (AbilitySystemComponent.IsValid())
    {
        // 监听 Generic Input Events
        // AbilitySystemComponent->GenericLocalConfirmCallbacks.AddDynamic(this, &UAbilityTask_WaitInputCombo::OnInputReceived);
    }
}

void UAbilityTask_WaitInputCombo::UnbindFromInputComponent()
{
    // 解绑输入
}

void UAbilityTask_WaitInputCombo::OnInputReceived(int32 InputID)
{
    float CurrentTime = GetWorld()->GetTimeSeconds();

    // 检查输入超时
    if (CurrentInputSequence.Num() > 0 && (CurrentTime - LastInputTime) > InputTimeWindow)
    {
        // 超时，重置
        ResetCombo();
        OnComboFailed.Broadcast(CurrentInputSequence);
        EndTask();
        return;
    }

    LastInputTime = CurrentTime;
    CurrentInputSequence.Add(InputID);

    CheckCompletion();

    // 设置超时定时器
    if (GetWorld())
    {
        GetWorld()->GetTimerManager().SetTimer(
            TimeoutTimerHandle,
            this,
            &UAbilityTask_WaitInputCombo::OnTimeout,
            InputTimeWindow,
            false
        );
    }
}

void UAbilityTask_WaitInputCombo::CheckCompletion()
{
    // 检查是否完成组合
    if (CurrentInputSequence.Num() == RequiredInputSequence.Num())
    {
        bool bMatches = true;
        for (int32 i = 0; i < RequiredInputSequence.Num(); ++i)
        {
            if (CurrentInputSequence[i] != RequiredInputSequence[i])
            {
                bMatches = false;
                break;
            }
        }

        if (bMatches)
        {
            OnComboCompleted.Broadcast(CurrentInputSequence);
            EndTask();
        }
        else
        {
            OnComboFailed.Broadcast(CurrentInputSequence);
            EndTask();
        }
    }
    else if (CurrentInputSequence.Num() < RequiredInputSequence.Num())
    {
        // 检查当前序列是否在正确的轨道上
        for (int32 i = 0; i < CurrentInputSequence.Num(); ++i)
        {
            if (CurrentInputSequence[i] != RequiredInputSequence[i])
            {
                OnComboFailed.Broadcast(CurrentInputSequence);
                EndTask();
                return;
            }
        }
    }
}

void UAbilityTask_WaitInputCombo::ResetCombo()
{
    CurrentInputSequence.Empty();
    LastInputTime = GetWorld()->GetTimeSeconds();
}

void UAbilityTask_WaitInputCombo::OnTimeout()
{
    if (CurrentInputSequence.Num() > 0 && CurrentInputSequence.Num() < RequiredInputSequence.Num())
    {
        OnComboFailed.Broadcast(CurrentInputSequence);
        EndTask();
    }
}
```

### 3.2 技能队列系统

技能队列允许玩家在技能冷却时预先输入，一旦可用立即激活。

#### 3.2.1 技能队列组件

```cpp
// AbilityQueueComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "GameplayAbilitySpec.h"
#include "AbilityQueueComponent.generated.h"

/**
 * 队列中的技能请求
 */
USTRUCT()
struct FQueuedAbilityRequest
{
    GENERATED_BODY()

    UPROPERTY()
    FGameplayAbilitySpecHandle AbilityHandle;

    UPROPERTY()
    float RequestTime;

    UPROPERTY()
    FGameplayEventData EventData;

    FQueuedAbilityRequest()
        : RequestTime(0.0f)
    {}
};

/**
 * 技能队列组件
 * 管理技能激活请求的队列
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class LYRAGAME_API UAbilityQueueComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UAbilityQueueComponent();

    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;

    // 将技能请求加入队列
    UFUNCTION(BlueprintCallable, Category = "Ability Queue")
    bool QueueAbility(FGameplayAbilitySpecHandle AbilityHandle, const FGameplayEventData& EventData);

    // 清空队列
    UFUNCTION(BlueprintCallable, Category = "Ability Queue")
    void ClearQueue();

    // 队列大小限制
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Ability Queue")
    int32 MaxQueueSize;

    // 队列超时时间（秒）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Ability Queue")
    float QueueTimeout;

protected:
    virtual void BeginPlay() override;

private:
    // 队列
    TArray<FQueuedAbilityRequest> AbilityQueue;

    // 缓存的 ASC
    UPROPERTY()
    UAbilitySystemComponent* CachedASC;

    void ProcessQueue();
    bool TryActivateQueuedAbility(const FQueuedAbilityRequest& Request);
    void RemoveExpiredRequests();
};
```

```cpp
// AbilityQueueComponent.cpp
#include "AbilityQueueComponent.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemGlobals.h"

UAbilityQueueComponent::UAbilityQueueComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
    MaxQueueSize = 3;
    QueueTimeout = 2.0f;
}

void UAbilityQueueComponent::BeginPlay()
{
    Super::BeginPlay();

    AActor* Owner = GetOwner();
    if (Owner)
    {
        CachedASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Owner);
    }
}

void UAbilityQueueComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    RemoveExpiredRequests();
    ProcessQueue();
}

bool UAbilityQueueComponent::QueueAbility(FGameplayAbilitySpecHandle AbilityHandle, const FGameplayEventData& EventData)
{
    if (!CachedASC || !AbilityHandle.IsValid())
    {
        return false;
    }

    // 如果技能当前可以激活，直接激活，不加入队列
    if (CachedASC->TryActivateAbility(AbilityHandle))
    {
        return true;
    }

    // 检查队列是否已满
    if (AbilityQueue.Num() >= MaxQueueSize)
    {
        UE_LOG(LogTemp, Warning, TEXT("Ability queue is full, cannot queue more abilities"));
        return false;
    }

    // 添加到队列
    FQueuedAbilityRequest Request;
    Request.AbilityHandle = AbilityHandle;
    Request.RequestTime = GetWorld()->GetTimeSeconds();
    Request.EventData = EventData;

    AbilityQueue.Add(Request);

    UE_LOG(LogTemp, Log, TEXT("Ability queued, queue size: %d"), AbilityQueue.Num());
    return true;
}

void UAbilityQueueComponent::ClearQueue()
{
    AbilityQueue.Empty();
    UE_LOG(LogTemp, Log, TEXT("Ability queue cleared"));
}

void UAbilityQueueComponent::ProcessQueue()
{
    if (!CachedASC || AbilityQueue.Num() == 0)
    {
        return;
    }

    // 尝试激活队列中的第一个技能
    FQueuedAbilityRequest& Request = AbilityQueue[0];
    if (TryActivateQueuedAbility(Request))
    {
        AbilityQueue.RemoveAt(0);
        UE_LOG(LogTemp, Log, TEXT("Queued ability activated, remaining queue size: %d"), AbilityQueue.Num());
    }
}

bool UAbilityQueueComponent::TryActivateQueuedAbility(const FQueuedAbilityRequest& Request)
{
    if (!CachedASC || !Request.AbilityHandle.IsValid())
    {
        return false;
    }

    // 尝试激活技能
    return CachedASC->TryActivateAbility(Request.AbilityHandle, true);
}

void UAbilityQueueComponent::RemoveExpiredRequests()
{
    float CurrentTime = GetWorld()->GetTimeSeconds();

    AbilityQueue.RemoveAll([this, CurrentTime](const FQueuedAbilityRequest& Request)
    {
        return (CurrentTime - Request.RequestTime) > QueueTimeout;
    });
}
```

### 3.3 Target Data 高级处理

Target Data 用于在客户端和服务器之间传递目标信息，包括位置、方向、命中的 Actor 等。

#### 3.3.1 自定义 Target Data

```cpp
// CustomTargetData.h
#pragma once

#include "CoreMinimal.h"
#include "Abilities/GameplayAbilityTargetTypes.h"
#include "CustomTargetData.generated.h"

/**
 * 自定义目标数据
 * 包含额外的战斗信息（如命中部位、击中方向等）
 */
USTRUCT()
struct FGameplayAbilityTargetData_CustomCombat : public FGameplayAbilityTargetData
{
    GENERATED_BODY()

    // 命中的 Actor
    UPROPERTY()
    TWeakObjectPtr<AActor> HitActor;

    // 命中位置
    UPROPERTY()
    FVector_NetQuantize HitLocation;

    // 命中法线
    UPROPERTY()
    FVector_NetQuantizeNormal HitNormal;

    // 命中部位（例如头部、身体、腿部）
    UPROPERTY()
    FName HitBone;

    // 攻击方向
    UPROPERTY()
    FVector_NetQuantizeNormal AttackDirection;

    // 是否为背刺
    UPROPERTY()
    bool bIsBackstab;

    // 是否为暴击
    UPROPERTY()
    bool bIsCritical;

    // 构造函数
    FGameplayAbilityTargetData_CustomCombat()
        : HitActor(nullptr)
        , HitLocation(ForceInitToZero)
        , HitNormal(ForceInitToZero)
        , HitBone(NAME_None)
        , AttackDirection(ForceInitToZero)
        , bIsBackstab(false)
        , bIsCritical(false)
    {}

    // 序列化
    virtual bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess) override;

    // 获取命中结果
    virtual TArray<TWeakObjectPtr<AActor>> GetActors() const override
    {
        TArray<TWeakObjectPtr<AActor>> Actors;
        if (HitActor.IsValid())
        {
            Actors.Add(HitActor);
        }
        return Actors;
    }

    // 判断是否有数据
    virtual bool HasHitResult() const override
    {
        return HitActor.IsValid();
    }

    // 获取命中位置
    virtual const FHitResult* GetHitResult() const override
    {
        // 转换为 FHitResult（简化示例）
        static FHitResult CachedHitResult;
        CachedHitResult.Actor = HitActor;
        CachedHitResult.Location = HitLocation;
        CachedHitResult.Normal = HitNormal;
        CachedHitResult.BoneName = HitBone;
        return &CachedHitResult;
    }
};

template<>
struct TStructOpsTypeTraits<FGameplayAbilityTargetData_CustomCombat> : public TStructOpsTypeTraitsBase2<FGameplayAbilityTargetData_CustomCombat>
{
    enum
    {
        WithNetSerializer = true
    };
};
```

```cpp
// CustomTargetData.cpp
#include "CustomTargetData.h"

bool FGameplayAbilityTargetData_CustomCombat::NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
{
    // 序列化 Actor
    Ar << HitActor;

    // 序列化位置和方向
    HitLocation.NetSerialize(Ar, Map, bOutSuccess);
    HitNormal.NetSerialize(Ar, Map, bOutSuccess);
    AttackDirection.NetSerialize(Ar, Map, bOutSuccess);

    // 序列化骨骼名称
    Ar << HitBone;

    // 序列化布尔值
    Ar << bIsBackstab;
    Ar << bIsCritical;

    bOutSuccess = true;
    return true;
}
```

#### 3.3.2 使用自定义 Target Data

```cpp
// 在技能中生成自定义 Target Data
void UMyMeleeAbility::GenerateTargetData()
{
    // 执行碰撞检测
    TArray<FHitResult> HitResults;
    PerformMeleeTrace(HitResults);

    // 创建 Target Data Handle
    FGameplayAbilityTargetDataHandle TargetDataHandle;

    for (const FHitResult& Hit : HitResults)
    {
        // 创建自定义 Target Data
        FGameplayAbilityTargetData_CustomCombat* CustomData = new FGameplayAbilityTargetData_CustomCombat();
        CustomData->HitActor = Hit.GetActor();
        CustomData->HitLocation = Hit.Location;
        CustomData->HitNormal = Hit.Normal;
        CustomData->HitBone = Hit.BoneName;

        // 计算攻击方向
        AActor* Attacker = GetAvatarActorFromActorInfo();
        if (Attacker)
        {
            CustomData->AttackDirection = (Hit.Location - Attacker->GetActorLocation()).GetSafeNormal();

            // 判断是否为背刺
            AActor* Target = Hit.GetActor();
            if (Target)
            {
                FVector TargetForward = Target->GetActorForwardVector();
                float DotProduct = FVector::DotProduct(CustomData->AttackDirection, TargetForward);
                CustomData->bIsBackstab = (DotProduct > 0.7f);  // 从背后攻击
            }
        }

        // 添加到 Handle
        TargetDataHandle.Add(CustomData);
    }

    // 应用 Gameplay Effect 到目标
    ApplyGameplayEffectToTarget(TargetDataHandle);
}

void UMyMeleeAbility::ApplyGameplayEffectToTarget(const FGameplayAbilityTargetDataHandle& TargetDataHandle)
{
    if (!DamageEffectClass)
    {
        return;
    }

    for (int32 i = 0; i < TargetDataHandle.Num(); ++i)
    {
        const FGameplayAbilityTargetData_CustomCombat* CustomData = 
            static_cast<const FGameplayAbilityTargetData_CustomCombat*>(TargetDataHandle.Get(i));

        if (!CustomData || !CustomData->HitActor.IsValid())
        {
            continue;
        }

        UAbilitySystemComponent* TargetASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(
            CustomData->HitActor.Get()
        );

        if (!TargetASC)
        {
            continue;
        }

        // 创建 Damage Effect Spec
        FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(DamageEffectClass, GetAbilityLevel());

        if (SpecHandle.IsValid())
        {
            // 根据 Target Data 调整伤害
            float BaseDamage = 100.0f;

            // 背刺增加 50% 伤害
            if (CustomData->bIsBackstab)
            {
                BaseDamage *= 1.5f;
                UE_LOG(LogTemp, Log, TEXT("Backstab! Damage increased by 50%%"));
            }

            // 暴击翻倍（这里可以从属性计算）
            if (CustomData->bIsCritical)
            {
                BaseDamage *= 2.0f;
            }

            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(FName("Data.Damage.Base")),
                BaseDamage
            );

            // 应用到目标
            GetAbilitySystemComponentFromActorInfo()->ApplyGameplayEffectSpecToTarget(
                *SpecHandle.Data.Get(),
                TargetASC
            );
        }
    }
}
```

### 3.4 动画通知与 GAS 集成

动画通知（Anim Notify）可以在动画播放到特定帧时触发 Gameplay Event，实现精确的攻击判定时机。

#### 3.4.1 自定义动画通知

```cpp
// AnimNotify_GameplayEvent.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimNotifies/AnimNotify.h"
#include "GameplayTagContainer.h"
#include "AnimNotify_GameplayEvent.generated.h"

/**
 * 触发 Gameplay Event 的动画通知
 */
UCLASS()
class LYRAGAME_API UAnimNotify_GameplayEvent : public UAnimNotify
{
    GENERATED_BODY()

public:
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation) override;

    // 要触发的 Gameplay Event 标签
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Gameplay Event")
    FGameplayTag EventTag;

    // 事件附带的数据（可选）
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Gameplay Event")
    float EventMagnitude;
};
```

```cpp
// AnimNotify_GameplayEvent.cpp
#include "AnimNotify_GameplayEvent.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemGlobals.h"

void UAnimNotify_GameplayEvent::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    Super::Notify(MeshComp, Animation);

    if (!MeshComp || !EventTag.IsValid())
    {
        return;
    }

    AActor* Owner = MeshComp->GetOwner();
    if (!Owner)
    {
        return;
    }

    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Owner);
    if (!ASC)
    {
        return;
    }

    // 构造 Event Data
    FGameplayEventData EventData;
    EventData.Instigator = Owner;
    EventData.Target = Owner;
    EventData.EventMagnitude = EventMagnitude;

    // 发送 Gameplay Event
    ASC->HandleGameplayEvent(EventTag, &EventData);

    UE_LOG(LogTemp, Log, TEXT("AnimNotify triggered Gameplay Event: %s"), *EventTag.ToString());
}
```

#### 3.4.2 在技能中监听动画通知

```cpp
// 修改 ComboAbility，使用动画通知触发攻击判定
void UComboAbility::ActivateAbility(...)
{
    // ... 播放动画 ...

    // 监听攻击判定事件（由动画通知触发）
    UAbilityTask_WaitGameplayEvent* WaitHitEvent = UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
        this,
        FGameplayTag::RequestGameplayTag(FName("Event.Montage.Attack.Hit"))  // 这个标签在动画通知中设置
    );

    WaitHitEvent->EventReceived.AddDynamic(this, &UComboAbility::OnHitTargets);
    WaitHitEvent->ReadyForActivation();
}

void UComboAbility::OnHitTargets(FGameplayTag EventTag, FGameplayEventData EventData)
{
    // 在动画通知触发的时刻执行攻击判定
    GenerateTargetData();
    ApplyDamageToTargets();
}
```

### 3.5 服务端验证与客户端预测

GAS 的网络同步策略：
- **客户端预测**：客户端立即执行技能效果，不等待服务器确认
- **服务端验证**：服务器验证技能是否合法（冷却、消耗、距离等）
- **回滚机制**：如果服务器拒绝，客户端回滚预测的效果

#### 3.5.1 配置技能的网络策略

```cpp
UMyAbility::UMyAbility()
{
    // 网络执行策略
    // LocalPredicted: 客户端预测执行，服务器验证并同步
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;

    // 本地预测键（用于回滚）
    bHasBlueprintActivate = false;
    bHasBlueprintActivateFromEvent = false;
}
```

#### 3.5.2 服务端验证逻辑

```cpp
bool UMyAbility::CanActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayTagContainer* SourceTags,
    const FGameplayTagContainer* TargetTags,
    OUT FGameplayTagContainer* OptionalRelevantTags) const
{
    // 首先调用父类验证
    if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
    {
        return false;
    }

    // 自定义服务端验证逻辑

    // 1. 验证距离（防止作弊）
    AActor* Avatar = ActorInfo->AvatarActor.Get();
    if (!Avatar)
    {
        return false;
    }

    // 获取目标（假设从 EventData 中传递）
    // AActor* Target = GetTargetFromContext();
    // if (Target)
    // {
    //     float Distance = FVector::Dist(Avatar->GetActorLocation(), Target->GetActorLocation());
    //     if (Distance > MaxAbilityRange)
    //     {
    //         UE_LOG(LogTemp, Warning, TEXT("Target too far, ability denied"));
    //         return false;
    //     }
    // }

    // 2. 验证资源（额外检查，确保客户端没有作弊）
    // （GAS 已经有内置的 Cost 检查，这里可以添加额外逻辑）

    // 3. 验证状态（例如是否在空中、是否被控制等）
    UAbilitySystemComponent* ASC = ActorInfo->AbilitySystemComponent.Get();
    if (ASC)
    {
        // 检查是否被眩晕
        if (ASC->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag(FName("State.Stunned"))))
        {
            return false;
        }
    }

    return true;
}
```

#### 3.5.3 处理预测回滚

```cpp
void UMyAbility::OnAbilityActivationFailed(const FGameplayTagContainer* FailedReason)
{
    Super::OnAbilityActivationFailed(FailedReason);

    // 服务器拒绝了技能激活，回滚客户端预测的效果

    // 1. 显示错误提示
    if (APlayerController* PC = Cast<APlayerController>(GetAvatarActorFromActorInfo()->GetInstigatorController()))
    {
        // 显示UI提示（例如"技能冷却中"、"距离过远"等）
    }

    // 2. 播放失败音效
    // PlayFailureSound();

    // 3. 清理预测的视觉效果
    // CleanupPredictedEffects();

    UE_LOG(LogTemp, Warning, TEXT("Ability activation failed and rolled back"));
}
```

## 4. 性能优化

### 4.1 Ability 实例化策略

GAS 提供三种实例化策略，影响性能和内存：

```cpp
// 1. NonInstanced（无实例）
// 优点：零内存开销，最快
// 缺点：不能保存状态，不能有成员变量
// 适用于：简单的被动技能、触发器
InstancingPolicy = EGameplayAbilityInstancingPolicy::NonInstanced;

// 2. InstancedPerActor（每个Actor一个实例）
// 优点：可以保存状态，性能较好
// 缺点：同一个技能只能有一个激活实例
// 适用于：大多数技能
InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;

// 3. InstancedPerExecution（每次执行一个实例）
// 优点：可以同时激活多个相同技能
// 缺点：内存和GC开销最大
// 适用于：需要多实例的技能（例如多段投射物）
InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerExecution;
```

**选择建议**：
- 被动效果、简单触发器 → NonInstanced
- 普通攻击技能、大多数技能 → InstancedPerActor
- 召唤物、多段技能 → InstancedPerExecution

### 4.2 Gameplay Effect 优化

#### 4.2.1 减少 Effect 数量

```cpp
// ❌ 不好：为每个 Buff 创建独立的 Effect
ApplyEffect(AttackPowerBuffEffect);
ApplyEffect(DefenseBuffEffect);
ApplyEffect(SpeedBuffEffect);

// ✅ 更好：合并到一个 Effect 中
// 一个 Effect 可以同时修改多个属性
class UGE_CombinedBuff : public UGameplayEffect
{
public:
    UGE_CombinedBuff()
    {
        // 同时修改攻击、防御、速度
        FGameplayModifierInfo AttackMod, DefenseMod, SpeedMod;
        AttackMod.Attribute = UMyAttributeSet::GetAttackPowerAttribute();
        DefenseMod.Attribute = UMyAttributeSet::GetDefenseAttribute();
        SpeedMod.Attribute = UMyAttributeSet::GetMovementSpeedAttribute();
        
        Modifiers.Add(AttackMod);
        Modifiers.Add(DefenseMod);
        Modifiers.Add(SpeedMod);
    }
};
```

#### 4.2.2 使用合适的 Duration Policy

```cpp
// Instant：立即应用，不占用内存（用于伤害、治疗）
DurationPolicy = EGameplayEffectDurationType::Instant;

// HasDuration：有持续时间，占用内存（用于 Buff）
DurationPolicy = EGameplayEffectDurationType::HasDuration;

// Infinite：永久效果，占用内存（用于被动技能）
DurationPolicy = EGameplayEffectDurationType::Infinite;
```

#### 4.2.3 优化 Periodic Effect

```cpp
// 周期性效果（DOT/HOT）优化

// ❌ 不好：周期太短，每帧都触发
Period = 0.01f;  // 每 0.01 秒

// ✅ 更好：合理的周期
Period = 1.0f;   // 每 1 秒

// 如果需要更平滑的效果，使用 Modifier Curve 而不是 Periodic
```

### 4.3 网络带宽优化

#### 4.3.1 减少 RPC 调用

```cpp
// ❌ 不好：每次攻击都发送 RPC
UFUNCTION(Server, Reliable, WithValidation)
void ServerAttack(AActor* Target);

// ✅ 更好：使用 Gameplay Event（GAS 内置网络优化）
FGameplayEventData EventData;
EventData.Target = Target;
ASC->HandleGameplayEvent(AttackEventTag, &EventData);
```

#### 4.3.2 优化 Gameplay Effect 同步

```cpp
// 设置 Replication Mode
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

// Mixed 模式：
// - Gameplay Effects 只同步到 Owner
// - Gameplay Tags 同步到所有客户端
// - Gameplay Cues 同步到所有客户端

// 这样可以减少大部分网络流量，同时保持必要的同步
```

#### 4.3.3 使用 Gameplay Cue 而不是 RPC

```cpp
// ❌ 不好：为视觉效果使用 RPC
UFUNCTION(NetMulticast, Unreliable)
void MulticastPlayHitEffect(FVector Location);

// ✅ 更好：使用 Gameplay Cue（GAS 优化过的网络系统）
FGameplayCueParameters CueParams;
CueParams.Location = HitLocation;
ASC->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag(FName("GameplayCue.Hit")), CueParams);
```

### 4.4 调试工具使用

#### 4.4.1 控制台命令

```cpp
// 在游戏中按 ` 键打开控制台，输入以下命令：

// 显示 Ability System 调试信息
showdebug abilitysystem

// 显示当前激活的 Gameplay Effect
AbilitySystem.Debug.ShowActiveEffects

// 显示当前的 Gameplay Tags
AbilitySystem.Debug.ShowTags

// 显示网络预测键（用于调试客户端预测）
AbilitySystem.Debug.ShowPredictionKeys
```

#### 4.4.2 自定义调试 HUD

```cpp
// DebugHUD.h
UCLASS()
class ADebugHUD : public AHUD
{
    GENERATED_BODY()

public:
    virtual void DrawHUD() override;

private:
    void DrawAbilitySystemDebug(UAbilitySystemComponent* ASC);
};

// DebugHUD.cpp
void ADebugHUD::DrawHUD()
{
    Super::DrawHUD();

    APawn* Pawn = GetOwningPlayerController()->GetPawn();
    if (!Pawn)
    {
        return;
    }

    UAbilitySystemComponent* ASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Pawn);
    if (ASC)
    {
        DrawAbilitySystemDebug(ASC);
    }
}

void ADebugHUD::DrawAbilitySystemDebug(UAbilitySystemComponent* ASC)
{
    float Y = 100.0f;
    float LineHeight = 20.0f;

    // 显示激活的 Abilities
    DrawText(TEXT("Active Abilities:"), FLinearColor::Yellow, 10, Y);
    Y += LineHeight;

    for (const FGameplayAbilitySpec& Spec : ASC->GetActivatableAbilities())
    {
        if (Spec.IsActive())
        {
            FString AbilityName = Spec.Ability->GetName();
            DrawText(FString::Printf(TEXT("  - %s"), *AbilityName), FLinearColor::White, 20, Y);
            Y += LineHeight;
        }
    }

    Y += LineHeight;

    // 显示激活的 Effects
    DrawText(TEXT("Active Effects:"), FLinearColor::Yellow, 10, Y);
    Y += LineHeight;

    const FActiveGameplayEffectsContainer& ActiveEffects = ASC->GetActiveGameplayEffects();
    for (const FActiveGameplayEffect& ActiveEffect : ActiveEffects)
    {
        FString EffectName = ActiveEffect.Spec.Def->GetName();
        float RemainingTime = ActiveEffect.GetTimeRemaining(GetWorld()->GetTimeSeconds());
        
        DrawText(
            FString::Printf(TEXT("  - %s (%.1fs)"), *EffectName, RemainingTime),
            FLinearColor::White,
            20,
            Y
        );
        Y += LineHeight;
    }

    Y += LineHeight;

    // 显示 Gameplay Tags
    DrawText(TEXT("Gameplay Tags:"), FLinearColor::Yellow, 10, Y);
    Y += LineHeight;

    FGameplayTagContainer OwnedTags;
    ASC->GetOwnedGameplayTags(OwnedTags);

    for (const FGameplayTag& Tag : OwnedTags)
    {
        DrawText(FString::Printf(TEXT("  - %s"), *Tag.ToString()), FLinearColor::White, 20, Y);
        Y += LineHeight;
    }
}
```

#### 4.4.3 使用 Profiler

```cpp
// 在代码中添加性能分析标记
void UMyAbility::ActivateAbility(...)
{
    SCOPE_CYCLE_COUNTER(STAT_MyAbility_Activate);
    
    // ... 技能逻辑 ...
}

// 在控制台中启用 Profiler
stat startfile
stat stopfile

// 或使用 Unreal Insights
```

## 总结

本文深入探讨了 GAS 的高级应用，涵盖了：

1. **复杂技能系统**：
   - 多阶段技能（蓄力、释放、恢复）
   - 连招系统（输入缓冲、连击窗口）
   - 技能打断与取消机制
   - 动态 Cooldown 和多资源消耗

2. **完整战斗系统**：
   - 自定义伤害计算（暴击、护甲、伤害类型）
   - Buff/Debuff 系统（堆叠、刷新、周期性效果）
   - 控制效果（眩晕、沉默、定身）
   - Buff 管理组件（UI 显示）

3. **高级技巧**：
   - 自定义 Gameplay Task
   - 技能队列系统
   - 自定义 Target Data
   - 动画通知集成
   - 服务端验证与客户端预测

4. **性能优化**：
   - Ability 实例化策略选择
   - Gameplay Effect 优化
   - 网络带宽优化
   - 调试工具使用

这些技术组合在一起，可以构建出一个功能完善、性能优秀的战斗系统。在实际项目中，应根据游戏类型（FPS、MOBA、RPG等）和具体需求，灵活调整和扩展这些系统。

## 常见问题

**Q1: 客户端预测失败后如何处理？**

A: GAS 会自动回滚预测的 Gameplay Effect。你需要在 `OnAbilityActivationFailed` 中处理视觉反馈（例如显示错误提示、播放失败音效）。

**Q2: 如何实现技能充能系统（例如闪现有 2 层充能）？**

A: 使用 Gameplay Effect 的堆叠系统，设置 `StackLimitCount = 2`，Cooldown Effect 完成后自动添加一层充能。

**Q3: 如何防止客户端作弊（例如修改冷却时间）？**

A: 在服务端的 `CanActivateAbility` 中添加验证逻辑，检查冷却时间、资源消耗、距离等。GAS 的 LocalPredicted 模式会在服务端验证失败时自动回滚客户端的预测。

**Q4: 如何优化大量 DOT/HOT 效果的性能？**

A: 
- 使用合理的 Period（1 秒而不是 0.1 秒）
- 合并相同类型的 DOT 到一个 Effect 中
- 使用 Instant Effect + Timer 而不是 Periodic Effect（适用于简单情况）

**Q5: 如何实现技能升级系统？**

A: 
- 在 `FGameplayAbilitySpec` 中使用 `Level` 字段
- 使用 `GetAbilityLevel()` 获取当前等级
- 在 Gameplay Effect 中使用 Curve Table 根据等级调整数值

## 下一步

在下一篇文章中，我们将探讨 **Enhanced Input System（新一代输入系统）**，学习如何将输入绑定到 GAS 技能，实现灵活的按键映射和输入上下文切换。

---

> **相关文章**：
> - [第 6 篇：GAS 入门 - Gameplay Ability System 基础](06-gas-basics.md)
> - [第 7 篇：GAS 进阶 - Attributes、Effects 与 Tags](07-gas-advanced.md)
> - [第 9 篇：Enhanced Input System - 新一代输入系统](09-enhanced-input.md)

> **参考资源**：
> - [Unreal Engine GAS 官方文档](https://docs.unrealengine.com/5.3/en-US/gameplay-ability-system-for-unreal-engine/)
> - [GASDocumentation (Community)](https://github.com/tranek/GASDocumentation)
> - [Lyra Sample Game](https://docs.unrealengine.com/5.3/en-US/lyra-sample-game-in-unreal-engine/)
