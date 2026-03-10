# 实战项目：Battle Royale 大逃杀模式开发

> **本章目标**：基于 Lyra 从零开发一个完整的 Battle Royale 大逃杀模式，包含缩圈系统、空投拾取、组队复活、观战系统和网络优化。

---

## 📋 目录

1. [Battle Royale 玩法设计](#1-battle-royale-玩法设计)
2. [安全区与缩圈系统](#2-安全区与缩圈系统)
3. [跳伞与落地系统](#3-跳伞与落地系统)
4. [战利品与空投系统](#4-战利品与空投系统)
5. [组队与复活机制](#5-组队与复活机制)
6. [倒地与救援系统](#6-倒地与救援系统)
7. [观战系统](#7-观战系统)
8. [100 人网络优化](#8-100-人网络优化)
9. [完整项目集成](#9-完整项目集成)
10. [测试与调优](#10-测试与调优)

---

## 1. Battle Royale 玩法设计

### 1.1 核心玩法定义

Battle Royale（大逃杀）是一种多人生存竞技玩法：

**核心规则**：
- **100 名玩家**同时进入地图
- **安全区逐渐缩小**，逼迫玩家相遇战斗
- **拾取装备**提升战斗力
- **最后存活的玩家或队伍**获胜

**关键系统**：
```
1. 跳伞系统 → 玩家从运输机跳下，选择降落点
2. 战利品系统 → 地图上随机生成武器、装备、物资
3. 安全区系统 → 定时缩圈，圈外持续扣血
4. 空投系统 → 定期投放高级装备
5. 组队系统 → 支持单人/双人/四人组队
6. 倒地系统 → 队友可救援
7. 观战系统 → 死亡后观看队友或其他玩家
8. 排名系统 → 根据存活时间和击杀数计分
```

### 1.2 技术挑战

**网络同步**：
- 100 人同时在线，带宽压力大
- 物品拾取需要快速响应
- 安全区边界需要精确同步

**性能优化**：
- 大地图渲染优化
- 大量 Actor 管理
- 客户端预测和插值

**游戏平衡**：
- 战利品分布密度
- 安全区缩圈速度
- 武器伤害平衡

---

## 2. 安全区与缩圈系统

### 2.1 安全区数据定义

创建 **Data Asset** 定义安全区配置：

```cpp
// BRSafeZoneData.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "BRSafeZoneData.generated.h"

// 单个缩圈阶段配置
USTRUCT(BlueprintType)
struct FBRSafeZonePhase
{
    GENERATED_BODY()

    // 该阶段圈的半径（cm）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "SafeZone")
    float Radius = 50000.0f;

    // 缩圈持续时间（秒）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "SafeZone")
    float ShrinkDuration = 120.0f;

    // 安全期（圈停止后的等待时间）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "SafeZone")
    float SafeDuration = 60.0f;

    // 圈外每秒伤害
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "SafeZone")
    float DamagePerSecond = 5.0f;

    // 伤害类型（GameplayEffect）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "SafeZone")
    TSubclassOf<class UGameplayEffect> DamageEffect;
};

UCLASS(BlueprintType)
class BATTLEROYALEPROJECT_API UBRSafeZoneData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 所有缩圈阶段
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "SafeZone")
    TArray<FBRSafeZonePhase> Phases;

    // 初始安全区半径
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "SafeZone")
    float InitialRadius = 100000.0f;

    // 地图中心点（如果为零则随机）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "SafeZone")
    FVector MapCenter = FVector::ZeroVector;

    // 最大随机偏移（最终圈随机位置）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "SafeZone")
    float MaxCenterOffset = 50000.0f;
};
```

### 2.2 安全区管理器

创建 **GameStateComponent** 管理安全区：

```cpp
// BRSafeZoneManagerComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/GameStateComponent.h"
#include "BRSafeZoneData.h"
#include "BRSafeZoneManagerComponent.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnSafeZoneUpdated, FVector, NewCenter, float, NewRadius);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPhaseChanged, int32, PhaseIndex);

UCLASS(BlueprintType, ClassGroup = (BattleRoyale))
class BATTLEROYALEPROJECT_API UBRSafeZoneManagerComponent : public UGameStateComponent
{
    GENERATED_BODY()

public:
    UBRSafeZoneManagerComponent(const FObjectInitializer& ObjectInitializer);

    virtual void BeginPlay() override;
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;

    // 初始化安全区
    UFUNCTION(BlueprintCallable, Category = "SafeZone")
    void InitializeSafeZone(UBRSafeZoneData* InSafeZoneData);

    // 手动触发下一阶段（测试用）
    UFUNCTION(BlueprintCallable, Category = "SafeZone")
    void ForceNextPhase();

    // 查询点是否在安全区内
    UFUNCTION(BlueprintPure, Category = "SafeZone")
    bool IsLocationInSafeZone(FVector Location) const;

    // 获取到安全区中心的距离
    UFUNCTION(BlueprintPure, Category = "SafeZone")
    float GetDistanceToSafeZoneCenter(FVector Location) const;

    // 广播事件
    UPROPERTY(BlueprintAssignable, Category = "SafeZone")
    FOnSafeZoneUpdated OnSafeZoneUpdated;

    UPROPERTY(BlueprintAssignable, Category = "SafeZone")
    FOnPhaseChanged OnPhaseChanged;

    // 当前安全区状态（Replicated）
    UPROPERTY(ReplicatedUsing = OnRep_CurrentCenter, BlueprintReadOnly, Category = "SafeZone")
    FVector CurrentCenter;

    UPROPERTY(ReplicatedUsing = OnRep_CurrentRadius, BlueprintReadOnly, Category = "SafeZone")
    float CurrentRadius;

    UPROPERTY(ReplicatedUsing = OnRep_TargetCenter, BlueprintReadOnly, Category = "SafeZone")
    FVector TargetCenter;

    UPROPERTY(ReplicatedUsing = OnRep_TargetRadius, BlueprintReadOnly, Category = "SafeZone")
    float TargetRadius;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "SafeZone")
    int32 CurrentPhaseIndex = -1;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "SafeZone")
    float PhaseTimeRemaining = 0.0f;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "SafeZone")
    bool bIsShrinking = false;

protected:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    UFUNCTION()
    void OnRep_CurrentCenter();

    UFUNCTION()
    void OnRep_CurrentRadius();

    UFUNCTION()
    void OnRep_TargetCenter();

    UFUNCTION()
    void OnRep_TargetRadius();

private:
    UPROPERTY()
    UBRSafeZoneData* SafeZoneData;

    float PhaseTimer = 0.0f;

    void StartNextPhase();
    void UpdateShrinkingZone(float DeltaTime);
    FVector GenerateRandomCenter(FVector PreviousCenter, float MaxRadius) const;
};
```

**实现缩圈逻辑**：

```cpp
// BRSafeZoneManagerComponent.cpp
#include "BRSafeZoneManagerComponent.h"
#include "Net/UnrealNetwork.h"
#include "Kismet/GameplayStatics.h"

UBRSafeZoneManagerComponent::UBRSafeZoneManagerComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    SetIsReplicatedByDefault(true);
    PrimaryComponentTick.bCanEverTick = true;
    PrimaryComponentTick.bStartWithTickEnabled = false;
}

void UBRSafeZoneManagerComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(UBRSafeZoneManagerComponent, CurrentCenter);
    DOREPLIFETIME(UBRSafeZoneManagerComponent, CurrentRadius);
    DOREPLIFETIME(UBRSafeZoneManagerComponent, TargetCenter);
    DOREPLIFETIME(UBRSafeZoneManagerComponent, TargetRadius);
    DOREPLIFETIME(UBRSafeZoneManagerComponent, CurrentPhaseIndex);
    DOREPLIFETIME(UBRSafeZoneManagerComponent, PhaseTimeRemaining);
    DOREPLIFETIME(UBRSafeZoneManagerComponent, bIsShrinking);
}

void UBRSafeZoneManagerComponent::BeginPlay()
{
    Super::BeginPlay();
}

void UBRSafeZoneManagerComponent::InitializeSafeZone(UBRSafeZoneData* InSafeZoneData)
{
    if (!GetOwner()->HasAuthority())
    {
        return; // 只在服务器执行
    }

    SafeZoneData = InSafeZoneData;
    if (!SafeZoneData || SafeZoneData->Phases.Num() == 0)
    {
        UE_LOG(LogTemp, Error, TEXT("SafeZoneData is invalid!"));
        return;
    }

    // 初始化第一个安全区
    if (SafeZoneData->MapCenter.IsZero())
    {
        // 随机地图中心
        CurrentCenter = FVector(
            FMath::RandRange(-SafeZoneData->MaxCenterOffset, SafeZoneData->MaxCenterOffset),
            FMath::RandRange(-SafeZoneData->MaxCenterOffset, SafeZoneData->MaxCenterOffset),
            0.0f
        );
    }
    else
    {
        CurrentCenter = SafeZoneData->MapCenter;
    }

    CurrentRadius = SafeZoneData->InitialRadius;
    CurrentPhaseIndex = -1;

    SetComponentTickEnabled(true);

    // 启动第一阶段
    StartNextPhase();

    OnSafeZoneUpdated.Broadcast(CurrentCenter, CurrentRadius);
}

void UBRSafeZoneManagerComponent::StartNextPhase()
{
    if (!GetOwner()->HasAuthority()) return;

    CurrentPhaseIndex++;
    if (CurrentPhaseIndex >= SafeZoneData->Phases.Num())
    {
        // 所有阶段完成
        SetComponentTickEnabled(false);
        UE_LOG(LogTemp, Log, TEXT("All SafeZone phases completed!"));
        return;
    }

    const FBRSafeZonePhase& Phase = SafeZoneData->Phases[CurrentPhaseIndex];

    // 设置目标圈
    TargetCenter = GenerateRandomCenter(CurrentCenter, CurrentRadius * 0.5f);
    TargetRadius = Phase.Radius;

    // 先等待安全期
    bIsShrinking = false;
    PhaseTimeRemaining = Phase.SafeDuration;
    PhaseTimer = 0.0f;

    OnPhaseChanged.Broadcast(CurrentPhaseIndex);

    UE_LOG(LogTemp, Log, TEXT("SafeZone Phase %d: Waiting %.1fs, then shrinking to %.0f over %.1fs"),
        CurrentPhaseIndex, Phase.SafeDuration, Phase.Radius, Phase.ShrinkDuration);
}

void UBRSafeZoneManagerComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    if (!GetOwner()->HasAuthority()) return;

    PhaseTimer += DeltaTime;
    PhaseTimeRemaining = FMath::Max(0.0f, PhaseTimeRemaining - DeltaTime);

    if (!bIsShrinking)
    {
        // 安全期倒计时
        if (PhaseTimer >= SafeZoneData->Phases[CurrentPhaseIndex].SafeDuration)
        {
            bIsShrinking = true;
            PhaseTimeRemaining = SafeZoneData->Phases[CurrentPhaseIndex].ShrinkDuration;
            PhaseTimer = 0.0f;
        }
    }
    else
    {
        // 缩圈中
        UpdateShrinkingZone(DeltaTime);

        if (PhaseTimer >= SafeZoneData->Phases[CurrentPhaseIndex].ShrinkDuration)
        {
            // 缩圈完成，进入下一阶段
            CurrentCenter = TargetCenter;
            CurrentRadius = TargetRadius;
            OnSafeZoneUpdated.Broadcast(CurrentCenter, CurrentRadius);
            StartNextPhase();
        }
    }
}

void UBRSafeZoneManagerComponent::UpdateShrinkingZone(float DeltaTime)
{
    const FBRSafeZonePhase& Phase = SafeZoneData->Phases[CurrentPhaseIndex];
    float Alpha = FMath::Clamp(PhaseTimer / Phase.ShrinkDuration, 0.0f, 1.0f);

    // 线性插值（可改为缓动曲线）
    float PrevRadius = (CurrentPhaseIndex == 0) ? SafeZoneData->InitialRadius : SafeZoneData->Phases[CurrentPhaseIndex - 1].Radius;
    CurrentRadius = FMath::Lerp(PrevRadius, TargetRadius, Alpha);
    CurrentCenter = FMath::Lerp(CurrentCenter, TargetCenter, Alpha);

    OnSafeZoneUpdated.Broadcast(CurrentCenter, CurrentRadius);
}

FVector UBRSafeZoneManagerComponent::GenerateRandomCenter(FVector PreviousCenter, float MaxRadius) const
{
    FVector2D Random2D = FMath::RandPointInCircle(MaxRadius);
    return PreviousCenter + FVector(Random2D.X, Random2D.Y, 0.0f);
}

bool UBRSafeZoneManagerComponent::IsLocationInSafeZone(FVector Location) const
{
    float Distance = FVector::Dist2D(Location, CurrentCenter);
    return Distance <= CurrentRadius;
}

float UBRSafeZoneManagerComponent::GetDistanceToSafeZoneCenter(FVector Location) const
{
    return FVector::Dist2D(Location, CurrentCenter);
}

void UBRSafeZoneManagerComponent::ForceNextPhase()
{
    if (GetOwner()->HasAuthority())
    {
        StartNextPhase();
    }
}

void UBRSafeZoneManagerComponent::OnRep_CurrentCenter()
{
    OnSafeZoneUpdated.Broadcast(CurrentCenter, CurrentRadius);
}

void UBRSafeZoneManagerComponent::OnRep_CurrentRadius()
{
    OnSafeZoneUpdated.Broadcast(CurrentCenter, CurrentRadius);
}

void UBRSafeZoneManagerComponent::OnRep_TargetCenter()
{
    // UI 可以显示下一个圈的位置
}

void UBRSafeZoneManagerComponent::OnRep_TargetRadius()
{
    // UI 可以显示下一个圈的大小
}
```

### 2.3 圈外伤害系统

创建 **GameplayAbility** 实现圈外伤害：

```cpp
// GA_SafeZoneDamage.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_SafeZoneDamage.generated.h"

UCLASS()
class BATTLEROYALEPROJECT_API UGA_SafeZoneDamage : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_SafeZoneDamage();

    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;

protected:
    // 检查间隔（秒）
    UPROPERTY(EditDefaultsOnly, Category = "SafeZone")
    float CheckInterval = 1.0f;

    UPROPERTY(EditDefaultsOnly, Category = "SafeZone")
    TSubclassOf<UGameplayEffect> DamageEffect;

private:
    FTimerHandle DamageTimerHandle;

    UFUNCTION()
    void CheckAndApplyDamage();
};
```

**实现伤害检测**：

```cpp
// GA_SafeZoneDamage.cpp
#include "GA_SafeZoneDamage.h"
#include "AbilitySystemComponent.h"
#include "BRSafeZoneManagerComponent.h"
#include "GameFramework/GameStateBase.h"

UGA_SafeZoneDamage::UGA_SafeZoneDamage()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerOnly;
}

void UGA_SafeZoneDamage::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 启动定时检测
    GetWorld()->GetTimerManager().SetTimer(
        DamageTimerHandle,
        this,
        &UGA_SafeZoneDamage::CheckAndApplyDamage,
        CheckInterval,
        true
    );
}

void UGA_SafeZoneDamage::CheckAndApplyDamage()
{
    AActor* OwnerActor = GetAvatarActorFromActorInfo();
    if (!OwnerActor) return;

    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (!GameState) return;

    UBRSafeZoneManagerComponent* SafeZoneManager = GameState->FindComponentByClass<UBRSafeZoneManagerComponent>();
    if (!SafeZoneManager) return;

    FVector ActorLocation = OwnerActor->GetActorLocation();
    if (!SafeZoneManager->IsLocationInSafeZone(ActorLocation))
    {
        // 在圈外，应用伤害
        if (DamageEffect)
        {
            UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();
            if (ASC)
            {
                FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
                EffectContext.AddSourceObject(this);

                FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingSpec(DamageEffect, GetAbilityLevel(), EffectContext);
                if (SpecHandle.IsValid())
                {
                    ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
                }
            }
        }
    }
}

void UGA_SafeZoneDamage::EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled)
{
    if (DamageTimerHandle.IsValid())
    {
        GetWorld()->GetTimerManager().ClearTimer(DamageTimerHandle);
    }

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

### 2.4 UI 显示：小地图与安全区边界

创建 **Widget** 显示小地图上的安全区：

```cpp
// WBP_BRMinimap.h (Blueprint)
// 蓝图实现要点：
// 1. 绑定 SafeZoneManager 的 OnSafeZoneUpdated 事件
// 2. 绘制当前圈（实线圆）
// 3. 绘制目标圈（虚线圆）
// 4. 显示玩家位置和队友位置
// 5. 显示剩余时间和阶段信息
```

---

## 3. 跳伞与落地系统

### 3.1 运输机路径生成

创建 **Actor** 管理运输机：

```cpp
// BRTransportPlane.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "BRTransportPlane.generated.h"

UCLASS()
class BATTLEROYALEPROJECT_API ABRTransportPlane : public AActor
{
    GENERATED_BODY()

public:
    ABRTransportPlane();

    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

    // 生成随机飞行路径
    UFUNCTION(BlueprintCallable, Category = "Plane")
    void GenerateFlightPath(FVector MapCenter, float MapRadius);

    // 获取当前位置
    UFUNCTION(BlueprintPure, Category = "Plane")
    FVector GetCurrentPosition() const { return CurrentPosition; }

    // 玩家跳伞
    UFUNCTION(BlueprintCallable, Category = "Plane")
    void EjectPlayer(APlayerController* PlayerController);

protected:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Plane")
    float FlightSpeed = 5000.0f; // cm/s

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Plane")
    float FlightHeight = 300000.0f; // 3000m

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Plane")
    FVector StartPosition;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Plane")
    FVector EndPosition;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Plane")
    FVector CurrentPosition;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Plane")
    float FlightProgress = 0.0f;

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

private:
    float TotalFlightTime = 0.0f;
};
```

**实现飞行逻辑**：

```cpp
// BRTransportPlane.cpp
#include "BRTransportPlane.h"
#include "Net/UnrealNetwork.h"
#include "Kismet/GameplayStatics.h"

ABRTransportPlane::ABRTransportPlane()
{
    PrimaryActorTick.bCanEverTick = true;
    bReplicates = true;
}

void ABRTransportPlane::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ABRTransportPlane, StartPosition);
    DOREPLIFETIME(ABRTransportPlane, EndPosition);
    DOREPLIFETIME(ABRTransportPlane, CurrentPosition);
    DOREPLIFETIME(ABRTransportPlane, FlightProgress);
}

void ABRTransportPlane::BeginPlay()
{
    Super::BeginPlay();
}

void ABRTransportPlane::GenerateFlightPath(FVector MapCenter, float MapRadius)
{
    if (!HasAuthority()) return;

    // 随机飞行角度
    float Angle = FMath::RandRange(0.0f, 360.0f);
    FVector Direction = FVector(FMath::Cos(FMath::DegreesToRadians(Angle)), FMath::Sin(FMath::DegreesToRadians(Angle)), 0.0f);

    // 飞行路径穿过地图中心
    float PathLength = MapRadius * 2.5f;
    StartPosition = MapCenter - Direction * PathLength * 0.5f + FVector(0, 0, FlightHeight);
    EndPosition = MapCenter + Direction * PathLength * 0.5f + FVector(0, 0, FlightHeight);

    CurrentPosition = StartPosition;
    FlightProgress = 0.0f;

    TotalFlightTime = PathLength / FlightSpeed;

    SetActorLocation(CurrentPosition);
}

void ABRTransportPlane::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (!HasAuthority()) return;

    FlightProgress += DeltaTime / TotalFlightTime;
    if (FlightProgress >= 1.0f)
    {
        // 飞行结束
        Destroy();
        return;
    }

    CurrentPosition = FMath::Lerp(StartPosition, EndPosition, FlightProgress);
    SetActorLocation(CurrentPosition);
}

void ABRTransportPlane::EjectPlayer(APlayerController* PlayerController)
{
    if (!HasAuthority()) return;

    APawn* PlayerPawn = PlayerController->GetPawn();
    if (!PlayerPawn) return;

    // 生成跳伞角色
    // TODO: 替换为跳伞状态的 Pawn
    FVector EjectLocation = CurrentPosition + FVector(0, 0, -1000.0f);
    PlayerPawn->SetActorLocation(EjectLocation);

    // 激活跳伞技能
    // TODO: 使用 GAS Ability 实现跳伞控制
}
```

### 3.2 跳伞 Ability

创建 **GameplayAbility** 实现跳伞控制：

```cpp
// GA_Parachute.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_Parachute.generated.h"

UCLASS()
class BATTLEROYALEPROJECT_API UGA_Parachute : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Parachute();

    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    float FallSpeed = 1000.0f; // cm/s

    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    float ForwardSpeed = 2000.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    float MaxDeployHeight = 200000.0f; // 2000m

    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    float AutoDeployHeight = 10000.0f; // 100m

    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    TSubclassOf<class UGameplayEffect> ParachuteMovementEffect;

private:
    FGameplayEffectContextHandle EffectContext;
    FActiveGameplayEffectHandle ActiveEffectHandle;
};
```

**实现跳伞机制**：

```cpp
// GA_Parachute.cpp (关键部分)
void UGA_Parachute::ActivateAbility(...)
{
    // 应用跳伞移动效果（修改重力和移动速度）
    if (ParachuteMovementEffect)
    {
        UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();
        EffectContext = ASC->MakeEffectContext();
        FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingSpec(ParachuteMovementEffect, GetAbilityLevel(), EffectContext);
        ActiveEffectHandle = ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
    }

    // 启动定时检测落地
    GetWorld()->GetTimerManager().SetTimer(
        LandingCheckTimer,
        this,
        &UGA_Parachute::CheckLanding,
        0.1f,
        true
    );
}

void UGA_Parachute::CheckLanding()
{
    AActor* Avatar = GetAvatarActorFromActorInfo();
    if (!Avatar) return;

    // 检测高度
    FVector Location = Avatar->GetActorLocation();
    FHitResult HitResult;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(Avatar);

    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        Location,
        Location - FVector(0, 0, 10000.0f),
        ECC_Visibility,
        Params
    );

    if (bHit && HitResult.Distance < AutoDeployHeight)
    {
        // 自动落地
        EndAbility(...);
    }
}

void UGA_Parachute::EndAbility(...)
{
    // 移除跳伞效果
    if (ActiveEffectHandle.IsValid())
    {
        UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();
        ASC->RemoveActiveGameplayEffect(ActiveEffectHandle);
    }

    // 清除定时器
    if (LandingCheckTimer.IsValid())
    {
        GetWorld()->GetTimerManager().ClearTimer(LandingCheckTimer);
    }

    Super::EndAbility(...);
}
```

---

## 4. 战利品与空投系统

### 4.1 战利品生成器

创建 **Actor** 管理战利品点：

```cpp
// BRLootSpawner.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "BRLootSpawner.generated.h"

USTRUCT(BlueprintType)
struct FBRLootEntry
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TSubclassOf<AActor> LootClass;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float SpawnWeight = 1.0f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    int32 MinCount = 1;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    int32 MaxCount = 1;
};

UCLASS()
class BATTLEROYALEPROJECT_API ABRLootSpawner : public AActor
{
    GENERATED_BODY()

public:
    ABRLootSpawner();

    virtual void BeginPlay() override;

    // 生成战利品
    UFUNCTION(BlueprintCallable, Category = "Loot")
    void SpawnLoot();

protected:
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Loot")
    TArray<FBRLootEntry> LootTable;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Loot")
    float SpawnRadius = 500.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Loot")
    int32 MinSpawnCount = 3;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Loot")
    int32 MaxSpawnCount = 8;

private:
    TSubclassOf<AActor> SelectRandomLoot() const;
};
```

**实现战利品生成**：

```cpp
// BRLootSpawner.cpp
#include "BRLootSpawner.h"
#include "Kismet/KismetMathLibrary.h"

ABRLootSpawner::ABRLootSpawner()
{
    PrimaryActorTick.bCanEverTick = false;
}

void ABRLootSpawner::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        SpawnLoot();
    }
}

void ABRLootSpawner::SpawnLoot()
{
    int32 SpawnCount = FMath::RandRange(MinSpawnCount, MaxSpawnCount);

    for (int32 i = 0; i < SpawnCount; ++i)
    {
        TSubclassOf<AActor> LootClass = SelectRandomLoot();
        if (!LootClass) continue;

        // 随机位置
        FVector2D RandomOffset = FMath::RandPointInCircle(SpawnRadius);
        FVector SpawnLocation = GetActorLocation() + FVector(RandomOffset.X, RandomOffset.Y, 100.0f);

        // 地面检测
        FHitResult HitResult;
        FCollisionQueryParams Params;
        Params.AddIgnoredActor(this);

        bool bHit = GetWorld()->LineTraceSingleByChannel(
            HitResult,
            SpawnLocation,
            SpawnLocation - FVector(0, 0, 10000.0f),
            ECC_WorldStatic,
            Params
        );

        if (bHit)
        {
            SpawnLocation = HitResult.Location + FVector(0, 0, 50.0f);
        }

        // 生成战利品
        FActorSpawnParameters SpawnParams;
        SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn;
        GetWorld()->SpawnActor<AActor>(LootClass, SpawnLocation, FRotator::ZeroRotator, SpawnParams);
    }
}

TSubclassOf<AActor> ABRLootSpawner::SelectRandomLoot() const
{
    if (LootTable.Num() == 0) return nullptr;

    // 加权随机选择
    float TotalWeight = 0.0f;
    for (const FBRLootEntry& Entry : LootTable)
    {
        TotalWeight += Entry.SpawnWeight;
    }

    float RandomValue = FMath::RandRange(0.0f, TotalWeight);
    float CumulativeWeight = 0.0f;

    for (const FBRLootEntry& Entry : LootTable)
    {
        CumulativeWeight += Entry.SpawnWeight;
        if (RandomValue <= CumulativeWeight)
        {
            return Entry.LootClass;
        }
    }

    return LootTable[0].LootClass;
}
```

### 4.2 空投系统

创建 **Actor** 管理空投：

```cpp
// BRAirdrop.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "BRAirdrop.generated.h"

UCLASS()
class BATTLEROYALEPROJECT_API ABRAirdrop : public AActor
{
    GENERATED_BODY()

public:
    ABRAirdrop();

    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

    // 初始化空投
    UFUNCTION(BlueprintCallable, Category = "Airdrop")
    void InitializeAirdrop(FVector TargetLocation);

protected:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Airdrop")
    UStaticMeshComponent* SupplyBoxMesh;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Airdrop")
    float DescentSpeed = 500.0f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Airdrop")
    float InitialHeight = 300000.0f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Airdrop")
    TArray<TSubclassOf<AActor>> HighTierLoot;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Airdrop")
    bool bHasLanded = false;

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

private:
    FVector TargetGroundLocation;

    UFUNCTION()
    void OnLanded();

    void SpawnLoot();
};
```

**实现空投降落**：

```cpp
// BRAirdrop.cpp
#include "BRAirdrop.h"
#include "Net/UnrealNetwork.h"
#include "Components/StaticMeshComponent.h"

ABRAirdrop::ABRAirdrop()
{
    PrimaryActorTick.bCanEverTick = true;
    bReplicates = true;

    SupplyBoxMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("SupplyBox"));
    RootComponent = SupplyBoxMesh;
}

void ABRAirdrop::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(ABRAirdrop, bHasLanded);
}

void ABRAirdrop::BeginPlay()
{
    Super::BeginPlay();
}

void ABRAirdrop::InitializeAirdrop(FVector TargetLocation)
{
    TargetGroundLocation = TargetLocation;
    FVector StartLocation = TargetLocation + FVector(0, 0, InitialHeight);
    SetActorLocation(StartLocation);
}

void ABRAirdrop::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (!HasAuthority() || bHasLanded) return;

    FVector CurrentLocation = GetActorLocation();
    FVector NewLocation = CurrentLocation - FVector(0, 0, DescentSpeed * DeltaTime);

    if (NewLocation.Z <= TargetGroundLocation.Z)
    {
        NewLocation.Z = TargetGroundLocation.Z;
        bHasLanded = true;
        OnLanded();
    }

    SetActorLocation(NewLocation);
}

void ABRAirdrop::OnLanded()
{
    SpawnLoot();

    // 播放特效和音效
    // TODO: 添加粒子效果和音效
}

void ABRAirdrop::SpawnLoot()
{
    for (TSubclassOf<AActor> LootClass : HighTierLoot)
    {
        if (!LootClass) continue;

        FVector SpawnLocation = GetActorLocation() + FVector(FMath::RandRange(-200.0f, 200.0f), FMath::RandRange(-200.0f, 200.0f), 50.0f);
        GetWorld()->SpawnActor<AActor>(LootClass, SpawnLocation, FRotator::ZeroRotator);
    }
}
```

---

## 5. 组队与复活机制

### 5.1 队伍管理器扩展

扩展 Lyra 的 `LyraTeamSubsystem` 支持组队：

```cpp
// BRSquadComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/GameStateComponent.h"
#include "BRSquadComponent.generated.h"

USTRUCT(BlueprintType)
struct FBRSquad
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 SquadID;

    UPROPERTY(BlueprintReadOnly)
    TArray<APlayerState*> Members;

    UPROPERTY(BlueprintReadOnly)
    int32 AliveCount = 0;

    UPROPERTY(BlueprintReadOnly)
    bool bIsEliminated = false;
};

UCLASS()
class BATTLEROYALEPROJECT_API UBRSquadComponent : public UGameStateComponent
{
    GENERATED_BODY()

public:
    UBRSquadComponent();

    // 创建队伍
    UFUNCTION(BlueprintCallable, Category = "Squad")
    int32 CreateSquad(const TArray<APlayerState*>& Players);

    // 获取玩家所在队伍
    UFUNCTION(BlueprintPure, Category = "Squad")
    FBRSquad GetPlayerSquad(APlayerState* PlayerState) const;

    // 检查队伍是否全灭
    UFUNCTION(BlueprintPure, Category = "Squad")
    bool IsSquadEliminated(int32 SquadID) const;

    // 通知队友死亡
    UFUNCTION(BlueprintCallable, Category = "Squad")
    void OnPlayerDied(APlayerState* PlayerState);

protected:
    UPROPERTY(Replicated)
    TArray<FBRSquad> Squads;

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

private:
    int32 NextSquadID = 1;
};
```

### 5.2 复活信标系统

创建 **可交互物体** 实现复活机制：

```cpp
// BRRespawnBeacon.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Interaction/IInteractableTarget.h"
#include "BRRespawnBeacon.generated.h"

UCLASS()
class BATTLEROYALEPROJECT_API ABRRespawnBeacon : public AActor, public IInteractableTarget
{
    GENERATED_BODY()

public:
    ABRRespawnBeacon();

    // IInteractableTarget 接口
    virtual FText GetInteractionText_Implementation() const override;
    virtual bool CanInteract_Implementation(APawn* InteractingPawn) const override;
    virtual void OnInteract_Implementation(APawn* InteractingPawn) override;

    // 设置待复活玩家
    UFUNCTION(BlueprintCallable, Category = "Respawn")
    void SetPendingPlayer(APlayerState* PlayerState);

protected:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Respawn")
    float RespawnDelay = 5.0f;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Respawn")
    APlayerState* PendingPlayerState;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Respawn")
    bool bIsActivated = false;

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

private:
    FTimerHandle RespawnTimerHandle;

    UFUNCTION()
    void ExecuteRespawn();
};
```

**实现复活交互**：

```cpp
// BRRespawnBeacon.cpp
#include "BRRespawnBeacon.h"
#include "Net/UnrealNetwork.h"

ABRRespawnBeacon::ABRRespawnBeacon()
{
    bReplicates = true;
}

void ABRRespawnBeacon::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(ABRRespawnBeacon, PendingPlayerState);
    DOREPLIFETIME(ABRRespawnBeacon, bIsActivated);
}

FText ABRRespawnBeacon::GetInteractionText_Implementation() const
{
    if (bIsActivated)
    {
        return FText::FromString(TEXT("Respawning..."));
    }
    return FText::FromString(TEXT("Press [E] to Respawn Teammate"));
}

bool ABRRespawnBeacon::CanInteract_Implementation(APawn* InteractingPawn) const
{
    return !bIsActivated && PendingPlayerState != nullptr;
}

void ABRRespawnBeacon::OnInteract_Implementation(APawn* InteractingPawn)
{
    if (!HasAuthority() || bIsActivated) return;

    bIsActivated = true;

    GetWorld()->GetTimerManager().SetTimer(
        RespawnTimerHandle,
        this,
        &ABRRespawnBeacon::ExecuteRespawn,
        RespawnDelay,
        false
    );
}

void ABRRespawnBeacon::ExecuteRespawn()
{
    if (!HasAuthority() || !PendingPlayerState) return;

    // 重生玩家
    APlayerController* PC = Cast<APlayerController>(PendingPlayerState->GetOwner());
    if (PC)
    {
        // 调用 GameMode 的重生逻辑
        // TODO: 集成 Lyra 的 Respawn 系统
        GetWorld()->GetAuthGameMode()->RestartPlayer(PC);
    }

    // 销毁信标
    Destroy();
}

void ABRRespawnBeacon::SetPendingPlayer(APlayerState* PlayerState)
{
    PendingPlayerState = PlayerState;
}
```

---

## 6. 倒地与救援系统

### 6.1 倒地状态 Ability

创建 **GameplayAbility** 实现倒地机制：

```cpp
// GA_DBNO.h (Down But Not Out)
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_DBNO.generated.h"

UCLASS()
class BATTLEROYALEPROJECT_API UGA_DBNO : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_DBNO();

    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;

protected:
    // 倒地持续时间（秒）
    UPROPERTY(EditDefaultsOnly, Category = "DBNO")
    float BleedoutDuration = 60.0f;

    // 倒地移动速度（原速度的百分比）
    UPROPERTY(EditDefaultsOnly, Category = "DBNO")
    float MovementSpeedMultiplier = 0.2f;

    UPROPERTY(EditDefaultsOnly, Category = "DBNO")
    TSubclassOf<UGameplayEffect> DBNOMovementEffect;

    UPROPERTY(EditDefaultsOnly, Category = "DBNO")
    TSubclassOf<UGameplayEffect> BleedoutDamageEffect;

private:
    FTimerHandle BleedoutTimerHandle;
    FActiveGameplayEffectHandle MovementEffectHandle;

    UFUNCTION()
    void OnBleedoutComplete();
};
```

**实现倒地逻辑**：

```cpp
// GA_DBNO.cpp
#include "GA_DBNO.h"
#include "AbilitySystemComponent.h"
#include "Character/LyraHealthComponent.h"

UGA_DBNO::UGA_DBNO()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerOnly;
}

void UGA_DBNO::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();

    // 应用移动限制效果
    if (DBNOMovementEffect)
    {
        FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
        FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingSpec(DBNOMovementEffect, GetAbilityLevel(), EffectContext);
        MovementEffectHandle = ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
    }

    // 启动流血倒计时
    GetWorld()->GetTimerManager().SetTimer(
        BleedoutTimerHandle,
        this,
        &UGA_DBNO::OnBleedoutComplete,
        BleedoutDuration,
        false
    );

    // 切换角色姿态为趴下
    AActor* Avatar = GetAvatarActorFromActorInfo();
    if (ACharacter* Character = Cast<ACharacter>(Avatar))
    {
        Character->Crouch();
    }
}

void UGA_DBNO::OnBleedoutComplete()
{
    // 流血时间结束，彻底死亡
    AActor* Avatar = GetAvatarActorFromActorInfo();
    if (ULyraHealthComponent* HealthComp = Avatar->FindComponentByClass<ULyraHealthComponent>())
    {
        // 设置生命值为 0
        // TODO: 触发死亡逻辑
    }

    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}

void UGA_DBNO::EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled)
{
    // 清除效果
    if (MovementEffectHandle.IsValid())
    {
        UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();
        ASC->RemoveActiveGameplayEffect(MovementEffectHandle);
    }

    if (BleedoutTimerHandle.IsValid())
    {
        GetWorld()->GetTimerManager().ClearTimer(BleedoutTimerHandle);
    }

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

### 6.2 救援 Ability

创建 **交互技能** 实现救援：

```cpp
// GA_Revive.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_Revive.generated.h"

UCLASS()
class BATTLEROYALEPROJECT_API UGA_Revive : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Revive();

    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, FGameplayTagContainer* OptionalRelevantTags) const override;
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;

protected:
    // 救援时间（秒）
    UPROPERTY(EditDefaultsOnly, Category = "Revive")
    float ReviveDuration = 5.0f;

    // 救援后恢复的生命值百分比
    UPROPERTY(EditDefaultsOnly, Category = "Revive")
    float ReviveHealthPercent = 0.3f;

    UPROPERTY(EditDefaultsOnly, Category = "Revive")
    TSubclassOf<UGameplayEffect> ReviveHealEffect;

private:
    FTimerHandle ReviveTimerHandle;
    TWeakObjectPtr<AActor> TargetActor;

    UFUNCTION()
    void OnReviveComplete();
};
```

**实现救援交互**：

```cpp
// GA_Revive.cpp (关键部分)
bool UGA_Revive::CanActivateAbility(...) const
{
    // 检测附近是否有倒地队友
    AActor* Avatar = GetAvatarActorFromActorInfo();
    if (!Avatar) return false;

    // 射线检测
    FVector Start = Avatar->GetActorLocation();
    FVector End = Start + Avatar->GetActorForwardVector() * 200.0f;

    FHitResult HitResult;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(Avatar);

    if (GetWorld()->LineTraceSingleByChannel(HitResult, Start, End, ECC_Pawn, Params))
    {
        if (AActor* HitActor = HitResult.GetActor())
        {
            // 检查是否有 DBNO Tag
            UAbilitySystemComponent* TargetASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(HitActor);
            if (TargetASC && TargetASC->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag(TEXT("Status.DBNO"))))
            {
                return true;
            }
        }
    }

    return false;
}

void UGA_Revive::ActivateAbility(...)
{
    if (!CommitAbility(...)) { EndAbility(...); return; }

    // 找到目标
    AActor* Avatar = GetAvatarActorFromActorInfo();
    FVector Start = Avatar->GetActorLocation();
    FVector End = Start + Avatar->GetActorForwardVector() * 200.0f;

    FHitResult HitResult;
    if (GetWorld()->LineTraceSingleByChannel(HitResult, Start, End, ECC_Pawn, FCollisionQueryParams()))
    {
        TargetActor = HitResult.GetActor();
    }

    // 启动救援倒计时
    GetWorld()->GetTimerManager().SetTimer(
        ReviveTimerHandle,
        this,
        &UGA_Revive::OnReviveComplete,
        ReviveDuration,
        false
    );

    // 播放救援动画
    // TODO: 触发救援动画 Montage
}

void UGA_Revive::OnReviveComplete()
{
    if (!TargetActor.IsValid()) return;

    UAbilitySystemComponent* TargetASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(TargetActor.Get());
    if (!TargetASC) return;

    // 移除 DBNO Tag
    TargetASC->RemoveLooseGameplayTag(FGameplayTag::RequestGameplayTag(TEXT("Status.DBNO")));

    // 恢复生命值
    if (ReviveHealEffect)
    {
        FGameplayEffectContextHandle EffectContext = TargetASC->MakeEffectContext();
        FGameplayEffectSpecHandle SpecHandle = TargetASC->MakeOutgoingSpec(ReviveHealEffect, GetAbilityLevel(), EffectContext);
        TargetASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
    }

    EndAbility(...);
}
```

---

## 7. 观战系统

### 7.1 观战 PlayerController

创建 **专用 PlayerController** 实现观战功能：

```cpp
// BRSpectatorController.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerController.h"
#include "BRSpectatorController.generated.h"

UCLASS()
class BATTLEROYALEPROJECT_API ABRSpectatorController : public APlayerController
{
    GENERATED_BODY()

public:
    ABRSpectatorController();

    virtual void BeginPlay() override;
    virtual void SetupInputComponent() override;

    // 切换观战目标
    UFUNCTION(BlueprintCallable, Category = "Spectator")
    void SpectateNextPlayer();

    UFUNCTION(BlueprintCallable, Category = "Spectator")
    void SpectatePreviousPlayer();

    UFUNCTION(BlueprintCallable, Category = "Spectator")
    void SpectatePlayer(APlayerState* TargetPlayerState);

    // 获取当前观战目标
    UFUNCTION(BlueprintPure, Category = "Spectator")
    APlayerState* GetCurrentSpectatedPlayer() const { return CurrentSpectatedPlayer; }

protected:
    UPROPERTY(BlueprintReadOnly, Category = "Spectator")
    TArray<APlayerState*> AlivePlayersList;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Spectator")
    APlayerState* CurrentSpectatedPlayer;

    UPROPERTY(EditDefaultsOnly, Category = "Spectator")
    TSubclassOf<class UCameraComponent> SpectateCameraClass;

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

private:
    int32 CurrentSpectateIndex = 0;

    void UpdateAlivePlayersList();
    void SetViewTargetToPlayer(APlayerState* PlayerState);

    void OnNextPlayerInput();
    void OnPreviousPlayerInput();
};
```

**实现观战切换**：

```cpp
// BRSpectatorController.cpp
#include "BRSpectatorController.h"
#include "Net/UnrealNetwork.h"
#include "GameFramework/PlayerState.h"
#include "GameFramework/GameStateBase.h"

ABRSpectatorController::ABRSpectatorController()
{
}

void ABRSpectatorController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(ABRSpectatorController, CurrentSpectatedPlayer);
}

void ABRSpectatorController::BeginPlay()
{
    Super::BeginPlay();

    UpdateAlivePlayersList();

    if (AlivePlayersList.Num() > 0)
    {
        SpectatePlayer(AlivePlayersList[0]);
    }
}

void ABRSpectatorController::SetupInputComponent()
{
    Super::SetupInputComponent();

    InputComponent->BindAction("SpectateNext", IE_Pressed, this, &ABRSpectatorController::OnNextPlayerInput);
    InputComponent->BindAction("SpectatePrevious", IE_Pressed, this, &ABRSpectatorController::OnPreviousPlayerInput);
}

void ABRSpectatorController::UpdateAlivePlayersList()
{
    AlivePlayersList.Empty();

    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (!GameState) return;

    for (APlayerState* PS : GameState->PlayerArray)
    {
        if (PS && !PS->IsOnlyASpectator())
        {
            // 检查是否存活（可以通过 Health 或 GameplayTag 判断）
            AlivePlayersList.Add(PS);
        }
    }
}

void ABRSpectatorController::SpectateNextPlayer()
{
    UpdateAlivePlayersList();
    if (AlivePlayersList.Num() == 0) return;

    CurrentSpectateIndex = (CurrentSpectateIndex + 1) % AlivePlayersList.Num();
    SpectatePlayer(AlivePlayersList[CurrentSpectateIndex]);
}

void ABRSpectatorController::SpectatePreviousPlayer()
{
    UpdateAlivePlayersList();
    if (AlivePlayersList.Num() == 0) return;

    CurrentSpectateIndex--;
    if (CurrentSpectateIndex < 0)
    {
        CurrentSpectateIndex = AlivePlayersList.Num() - 1;
    }
    SpectatePlayer(AlivePlayersList[CurrentSpectateIndex]);
}

void ABRSpectatorController::SpectatePlayer(APlayerState* TargetPlayerState)
{
    if (!TargetPlayerState) return;

    CurrentSpectatedPlayer = TargetPlayerState;

    APawn* TargetPawn = TargetPlayerState->GetPawn();
    if (TargetPawn)
    {
        SetViewTargetWithBlend(TargetPawn, 0.5f);
    }
}

void ABRSpectatorController::OnNextPlayerInput()
{
    SpectateNextPlayer();
}

void ABRSpectatorController::OnPreviousPlayerInput()
{
    SpectatePreviousPlayer();
}
```

### 7.2 队友观战 UI

创建 **Widget** 显示队友状态：

```cpp
// WBP_SquadStatus.h (Blueprint)
// 功能：
// 1. 显示所有队友的头像、名称、生命值
// 2. 高亮当前观战的队友
// 3. 显示队友的倒地状态
// 4. 点击头像切换观战目标
```

---

## 8. 100 人网络优化

### 8.1 Replication Graph 配置

为 Battle Royale 创建专用 **Replication Graph**：

```cpp
// BRReplicationGraph.h
#pragma once

#include "CoreMinimal.h"
#include "ReplicationGraph.h"
#include "BRReplicationGraph.generated.h"

UCLASS()
class BATTLEROYALEPROJECT_API UBRReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()

public:
    virtual void InitGlobalActorClassSettings() override;
    virtual void InitGlobalGraphNodes() override;
    virtual void RouteAddNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo, FGlobalActorReplicationInfo& GlobalInfo) override;
    virtual void RouteRemoveNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo) override;

protected:
    // Grid 节点（空间分割）
    UPROPERTY()
    UReplicationGraphNode_GridSpatialization2D* GridNode;

    // 全局始终相关节点
    UPROPERTY()
    UReplicationGraphNode_ActorList* AlwaysRelevantNode;
};
```

**实现 Grid 分割**：

```cpp
// BRReplicationGraph.cpp
#include "BRReplicationGraph.h"
#include "Net/UnrealNetwork.h"
#include "ReplicationGraphNode_GridSpatialization2D.h"

void UBRReplicationGraph::InitGlobalActorClassSettings()
{
    Super::InitGlobalActorClassSettings();

    // 配置 Player 始终相关
    FClassReplicationInfo PlayerRepInfo;
    PlayerRepInfo.SetCullDistanceSquared(0.0f); // 不按距离剔除
    GlobalActorReplicationInfoMap.SetClassInfo(APlayerController::StaticClass(), PlayerRepInfo);

    // 配置武器的复制距离
    FClassReplicationInfo WeaponRepInfo;
    WeaponRepInfo.SetCullDistanceSquared(5000.0f * 5000.0f); // 50m
    // GlobalActorReplicationInfoMap.SetClassInfo(AWeaponActorClass::StaticClass(), WeaponRepInfo);
}

void UBRReplicationGraph::InitGlobalGraphNodes()
{
    // 创建 Grid 节点（空间分割）
    GridNode = CreateNewNode<UReplicationGraphNode_GridSpatialization2D>();
    GridNode->CellSize = 10000.0f; // 100m 一个格子
    GridNode->SpatialBias = FVector2D(0.0f, 0.0f);
    AddGlobalGraphNode(GridNode);

    // 创建全局始终相关节点
    AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_ActorList>();
    AddGlobalGraphNode(AlwaysRelevantNode);
}

void UBRReplicationGraph::RouteAddNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo, FGlobalActorReplicationInfo& GlobalInfo)
{
    if (ActorInfo.Actor->IsA(APlayerController::StaticClass()))
    {
        // PlayerController 始终相关
        AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
    }
    else if (ActorInfo.Actor->IsA(APawn::StaticClass()))
    {
        // Pawn 加入 Grid
        GridNode->AddActor_Dormancy(ActorInfo, GlobalInfo);
    }
    else
    {
        // 其他 Actor（武器、战利品）加入 Grid
        GridNode->AddActor_Static(ActorInfo, GlobalInfo);
    }
}

void UBRReplicationGraph::RouteRemoveNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo)
{
    GridNode->RemoveActor_Static(ActorInfo);
    AlwaysRelevantNode->NotifyRemoveNetworkActor(ActorInfo);
}
```

### 8.2 客户端预测优化

使用 **GAS 预测** 减少网络延迟：

```cpp
// 在 Ability 中启用预测
NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;

// 在 GameplayEffect 中启用预测
PredictionType = EGameplayEffectDurationType::Instant;

// 关键操作使用 RPC 确认
UFUNCTION(Server, Reliable, WithValidation)
void ServerConfirmAction(FGameplayAbilitySpecHandle Handle);
```

### 8.3 带宽优化

**优化 Actor 复制频率**：

```cpp
// BRCharacter.cpp
ABRCharacter::ABRCharacter()
{
    // 降低非关键 Actor 的更新频率
    NetUpdateFrequency = 10.0f; // 每秒 10 次（默认 100）
    MinNetUpdateFrequency = 2.0f;

    // 启用网络优先级
    NetPriority = 1.0f;
    NetCullDistanceSquared = 10000.0f * 10000.0f; // 100m
}

// 动态调整优先级
void ABRCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 仅复制给队友
    DOREPLIFETIME_CONDITION(ABRCharacter, TeamID, COND_OwnerOnly);

    // 跳过初始复制（优化初始连接）
    DOREPLIFETIME_CONDITION(ABRCharacter, EquippedWeapon, COND_SkipOwner);
}
```

**启用 Actor 休眠**：

```cpp
// 启用休眠机制
NetDormancy = DORM_DormantAll;

// 唤醒条件
void ABRWeapon::OnPickup()
{
    FlushNetDormancy(); // 拾取时唤醒
}

void ABRWeapon::OnDrop()
{
    SetNetDormancy(DORM_DormantAll); // 丢弃后休眠
}
```

---

## 9. 完整项目集成

### 9.1 创建 Battle Royale Experience

创建 **Experience Definition**：

```
Content/Experiences/B_BattleRoyale/
├── B_BattleRoyale_Experience.uasset (ULyraExperienceDefinition)
├── PDA_BattleRoyale_SafeZone.uasset (UBRSafeZoneData)
├── DA_BattleRoyale_LootTable.uasset
└── GFAs/
    ├── GFA_AddBRComponents.uasset (GameFeatureAction)
    └── GFA_LoadBRMap.uasset
```

**Experience 配置**：

```cpp
// B_BattleRoyale_Experience (Data Asset)
Experience Definition:
- Default Pawn Data: PDA_BattleRoyale_Character
- Actions:
  1. AddComponents:
     - GameState: BRSafeZoneManagerComponent, BRSquadComponent
     - PlayerState: BRPlayerStatComponent
  2. AddAbilitySets:
     - AS_BattleRoyale (包含 DBNO、Revive、SafeZoneDamage)
  3. LoadMap:
     - Map: BattleRoyale_BigMap
```

### 9.2 Game Feature Plugin 结构

创建 **Game Feature Plugin**：

```
Plugins/GameFeatures/BattleRoyale/
├── Content/
│   ├── Actors/
│   │   ├── BP_BRTransportPlane.uasset
│   │   ├── BP_BRAirdrop.uasset
│   │   ├── BP_BRLootSpawner.uasset
│   │   └── BP_BRRespawnBeacon.uasset
│   ├── Abilities/
│   │   ├── GA_Parachute.uasset
│   │   ├── GA_DBNO.uasset
│   │   ├── GA_Revive.uasset
│   │   └── GA_SafeZoneDamage.uasset
│   ├── UI/
│   │   ├── WBP_BRMinimap.uasset
│   │   ├── WBP_SafeZoneTimer.uasset
│   │   └── WBP_SquadStatus.uasset
│   └── Data/
│       ├── PDA_BattleRoyale_SafeZone.uasset
│       └── DT_LootTable.uasset (DataTable)
└── Source/
    └── BattleRoyaleRuntime/
        ├── Public/
        │   ├── BRSafeZoneManagerComponent.h
        │   ├── BRSquadComponent.h
        │   ├── BRTransportPlane.h
        │   └── ...
        └── Private/
            └── ...
```

### 9.3 GameMode 配置

创建 **Battle Royale GameMode**：

```cpp
// BRGameMode.h
#pragma once

#include "CoreMinimal.h"
#include "GameModes/LyraGameMode.h"
#include "BRGameMode.generated.h"

UENUM(BlueprintType)
enum class EBRMatchPhase : uint8
{
    WaitingPlayers,
    Boarding,
    Flying,
    InProgress,
    EndGame
};

UCLASS()
class BATTLEROYALEPROJECT_API ABRGameMode : public ALyraGameMode
{
    GENERATED_BODY()

public:
    ABRGameMode();

    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;
    virtual void HandleMatchIsWaitingToStart() override;
    virtual void HandleMatchHasStarted() override;

    // 更新比赛阶段
    UFUNCTION(BlueprintCallable, Category = "GameMode")
    void SetMatchPhase(EBRMatchPhase NewPhase);

protected:
    UPROPERTY(EditDefaultsOnly, Category = "BattleRoyale")
    int32 MinPlayersToStart = 10;

    UPROPERTY(EditDefaultsOnly, Category = "BattleRoyale")
    int32 MaxPlayers = 100;

    UPROPERTY(EditDefaultsOnly, Category = "BattleRoyale")
    float BoardingDuration = 30.0f;

    UPROPERTY(EditDefaultsOnly, Category = "BattleRoyale")
    TSubclassOf<class ABRTransportPlane> TransportPlaneClass;

    UPROPERTY(BlueprintReadOnly, Category = "BattleRoyale")
    EBRMatchPhase CurrentPhase;

private:
    UPROPERTY()
    ABRTransportPlane* TransportPlane;

    FTimerHandle PhaseTimerHandle;

    void StartBoardingPhase();
    void StartFlyingPhase();
    void StartInProgressPhase();
    void CheckWinCondition();
};
```

---

## 10. 测试与调优

### 10.1 本地测试配置

**编辑器多人测试**：

```ini
; DefaultEngine.ini
[/Script/Engine.GameEngine]
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="/Script/OnlineSubsystemUtils.IpNetDriver",DriverClassNameFallback="/Script/OnlineSubsystemUtils.IpNetDriver")

[/Script/OnlineSubsystemUtils.IpNetDriver]
MaxClientRate=100000
MaxInternetClientRate=100000
NetServerMaxTickRate=30
LanServerMaxTickRate=30
```

**启动多客户端**：

```bash
# 服务器
UE5Editor.exe ProjectName -server -log

# 客户端 1
UE5Editor.exe ProjectName 127.0.0.1 -game -log -ResX=800 -ResY=600 -WinX=0 -WinY=0

# 客户端 2
UE5Editor.exe ProjectName 127.0.0.1 -game -log -ResX=800 -ResY=600 -WinX=810 -WinY=0
```

### 10.2 性能分析命令

**网络调试命令**：

```
stat net                  // 网络统计
stat netplayermovement    // 玩家移动同步统计
stat game                 // 游戏逻辑统计
stat fps                  // 帧率
stat unit                 // CPU/GPU 时间

// Replication Graph 调试
net.RepGraph.PrintAll 1
net.RepGraph.DrawDebug 1

// 带宽分析
net.PktLag=50             // 模拟 50ms 延迟
net.PktLoss=5             // 模拟 5% 丢包
```

### 10.3 压力测试

使用 **Gauntlet** 框架进行压力测试：

```cpp
// BRGauntletTest.cpp
class FBRStressTest : public FLyraTestControllerBase
{
public:
    virtual void OnInit() override
    {
        // 生成 100 个 Bot
        for (int32 i = 0; i < 100; ++i)
        {
            SpawnBotPlayer();
        }
    }

    virtual void OnTick(float TimeDelta) override
    {
        // 检测网络性能
        if (GetAverageLatency() > 200.0f)
        {
            UE_LOG(LogTemp, Warning, TEXT("High latency detected: %.2fms"), GetAverageLatency());
        }

        // 检测帧率
        if (GetAverageFPS() < 30.0f)
        {
            UE_LOG(LogTemp, Warning, TEXT("Low FPS detected: %.2f"), GetAverageFPS());
        }
    }
};
```

---

## 📊 总结与优化建议

### 已实现功能清单

✅ **核心玩法**：
- 安全区缩圈系统（多阶段，可配置）
- 跳伞与落地系统
- 战利品随机生成
- 空投系统

✅ **组队与复活**：
- 队伍管理系统
- 倒地与救援机制
- 复活信标系统

✅ **观战系统**：
- 死亡后观战
- 队友视角切换

✅ **网络优化**：
- Replication Graph 空间分割
- 客户端预测
- 带宽优化策略

### 下一步优化方向

1. **战斗平衡**：
   - 调整武器伤害
   - 优化战利品分布
   - 平衡安全区速度

2. **UI 完善**：
   - 击杀提示
   - 伤害数字显示
   - 排名界面

3. **音效与特效**：
   - 枪声传播系统
   - 缩圈警告音效
   - 空投飞机音效

4. **反外挂措施**：
   - 服务端权威验证
   - 射线检测防穿墙
   - 移动速度限制

### 推荐学习资源

- **PUBG/Apex Legends** 的 GDC 演讲
- **Fortnite** 的网络架构案例
- **Unreal Engine Replication Graph** 官方文档

---

## 🎉 恭喜！

你已经完成了一个完整的 **Battle Royale** 模式开发！这个项目涵盖了：

- 大规模多人网络同步
- 复杂的游戏阶段管理
- 动态内容生成（战利品、空投）
- 玩家交互系统（救援、观战）

现在你可以：

1. 扩展更多玩法（载具、技能道具）
2. 优化性能支持更大地图
3. 添加排位赛系统
4. 集成社交功能（好友组队、语音聊天）

**继续挑战更高难度的项目吧！** 🚀
