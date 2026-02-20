# 第24章：动画系统与 Motion Matching

## 概述

Lyra 的动画系统是一个高度模块化、可扩展的架构，充分利用了 Unreal Engine 5 的现代动画特性。本章将深入探讨 Lyra 如何构建动画系统，包括动画蓝图架构、Motion Matching 系统、与 GAS 的集成，以及网络同步机制。

### 本章内容

- Lyra 动画架构设计
- Animation Blueprint 与 Linked Anim Graph
- Motion Matching 系统详解
- Pose Search 数据库构建
- GAS 动画通知集成
- IK 和程序化动画
- 武器动画层系统
- 网络动画同步
- 完整实战案例

---

## 1. Lyra 动画架构概览

### 1.1 核心架构组件

Lyra 的动画系统基于以下核心组件：

```
LyraCharacter (角色类)
    └── SkeletalMeshComponent (骨骼网格组件)
        └── LyraAnimInstance (主动画实例)
            ├── Linked Anim Graphs (链接动画图表)
            │   ├── Locomotion Layer (移动层)
            │   ├── Weapon Anim Layer (武器层)
            │   └── Vehicle Anim Layer (载具层)
            └── Animation Modifiers (动画修改器)
                ├── IK Nodes (IK节点)
                ├── Procedural Animation (程序化动画)
                └── Control Rig (控制绑定)
```

### 1.2 ULyraAnimInstance 核心实现

`ULyraAnimInstance` 是 Lyra 动画系统的基类，负责与 GAS 集成：

```cpp
// LyraAnimInstance.h
#pragma once

#include "Animation/AnimInstance.h"
#include "GameplayEffectTypes.h"
#include "LyraAnimInstance.generated.h"

class UAbilitySystemComponent;

/**
 * ULyraAnimInstance
 * Lyra 项目使用的基础游戏动画实例类
 */
UCLASS(Config = Game)
class ULyraAnimInstance : public UAnimInstance
{
    GENERATED_BODY()

public:
    ULyraAnimInstance(const FObjectInitializer& ObjectInitializer);

    // 使用 Ability System Component 初始化
    virtual void InitializeWithAbilitySystem(UAbilitySystemComponent* ASC);

protected:
    virtual void NativeInitializeAnimation() override;
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;

protected:
    // Gameplay Tags 映射到蓝图变量
    // 这些变量会在标签添加或移除时自动更新
    UPROPERTY(EditDefaultsOnly, Category = "GameplayTags")
    FGameplayTagBlueprintPropertyMap GameplayTagPropertyMap;

    // 角色状态数据
    UPROPERTY(BlueprintReadOnly, Category = "Character State Data")
    float GroundDistance = -1.0f;
};
```

### 1.3 动画实例实现细节

```cpp
// LyraAnimInstance.cpp
#include "LyraAnimInstance.h"
#include "AbilitySystemGlobals.h"
#include "Character/LyraCharacter.h"
#include "Character/LyraCharacterMovementComponent.h"

ULyraAnimInstance::ULyraAnimInstance(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
}

void ULyraAnimInstance::InitializeWithAbilitySystem(UAbilitySystemComponent* ASC)
{
    check(ASC);
    
    // 初始化 Gameplay Tag 属性映射
    GameplayTagPropertyMap.Initialize(this, ASC);
}

void ULyraAnimInstance::NativeInitializeAnimation()
{
    Super::NativeInitializeAnimation();

    // 自动获取并绑定 ASC
    if (AActor* OwningActor = GetOwningActor())
    {
        if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(OwningActor))
        {
            InitializeWithAbilitySystem(ASC);
        }
    }
}

void ULyraAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);

    const ALyraCharacter* Character = Cast<ALyraCharacter>(GetOwningActor());
    if (!Character)
    {
        return;
    }

    // 更新角色状态数据
    ULyraCharacterMovementComponent* CharMoveComp = CastChecked<ULyraCharacterMovementComponent>(Character->GetCharacterMovement());
    const FLyraCharacterGroundInfo& GroundInfo = CharMoveComp->GetGroundInfo();
    GroundDistance = GroundInfo.GroundDistance;
}
```

**关键设计要点：**

1. **GameplayTagPropertyMap**：自动将 Gameplay Tags 映射到动画蓝图变量
2. **延迟初始化**：在 `NativeInitializeAnimation` 中自动查找并绑定 ASC
3. **状态缓存**：缓存常用的角色状态数据（如地面距离）以供动画蓝图使用

---

## 2. Animation Blueprint 设计模式

### 2.1 Linked Anim Graph 架构

Lyra 使用 Linked Anim Graph（链接动画图表）实现动画层的模块化：

```cpp
// 动画层选择系统
USTRUCT(BlueprintType)
struct FLyraAnimLayerSelectionEntry
{
    GENERATED_BODY()

    // 匹配时应用的动画层
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSubclassOf<UAnimInstance> Layer;

    // 需要的装饰标签（所有标签都必须存在才算匹配）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta=(Categories="Cosmetic"))
    FGameplayTagContainer RequiredTags;
};

USTRUCT(BlueprintType)
struct FLyraAnimLayerSelectionSet
{
    GENERATED_BODY()
    
    // 要应用的层规则列表，使用第一个匹配的
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta=(TitleProperty=Layer))
    TArray<FLyraAnimLayerSelectionEntry> LayerRules;

    // 如果没有规则匹配时使用的默认层
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSubclassOf<UAnimInstance> DefaultLayer;

    // 根据规则选择最佳层
    TSubclassOf<UAnimInstance> SelectBestLayer(const FGameplayTagContainer& CosmeticTags) const;
};
```

### 2.2 动画层选择实现

```cpp
// LyraCosmeticAnimationTypes.cpp
TSubclassOf<UAnimInstance> FLyraAnimLayerSelectionSet::SelectBestLayer(const FGameplayTagContainer& CosmeticTags) const
{
    // 遍历所有规则
    for (const FLyraAnimLayerSelectionEntry& Rule : LayerRules)
    {
        // 如果层有效且所有需要的标签都存在
        if ((Rule.Layer != nullptr) && CosmeticTags.HasAll(Rule.RequiredTags))
        {
            return Rule.Layer;
        }
    }

    // 没有匹配规则时返回默认层
    return DefaultLayer;
}
```

### 2.3 创建自定义动画蓝图

**步骤 1：创建基础动画蓝图**

```cpp
// MyGameAnimInstance.h
#pragma once

#include "Animation/LyraAnimInstance.h"
#include "MyGameAnimInstance.generated.h"

UCLASS()
class UMyGameAnimInstance : public ULyraAnimInstance
{
    GENERATED_BODY()

public:
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;

protected:
    // 移动数据
    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    FVector Velocity;

    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    float GroundSpeed;

    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    float Direction;

    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    bool bIsAccelerating;

    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    bool bIsFalling;

    // 武器数据
    UPROPERTY(BlueprintReadOnly, Category = "Combat")
    bool bIsAiming;

    UPROPERTY(BlueprintReadOnly, Category = "Combat")
    float AimPitch;

    UPROPERTY(BlueprintReadOnly, Category = "Combat")
    float AimYaw;
};
```

**步骤 2：实现动画更新逻辑**

```cpp
// MyGameAnimInstance.cpp
#include "MyGameAnimInstance.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Kismet/KismetMathLibrary.h"

void UMyGameAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);

    ACharacter* Character = Cast<ACharacter>(GetOwningActor());
    if (!Character)
    {
        return;
    }

    UCharacterMovementComponent* MovementComp = Character->GetCharacterMovement();
    if (!MovementComp)
    {
        return;
    }

    // 更新移动数据
    Velocity = MovementComp->Velocity;
    GroundSpeed = Velocity.Size2D();
    
    // 计算移动方向（相对于角色旋转）
    if (GroundSpeed > 0.0f)
    {
        FRotator CharacterRotation = Character->GetActorRotation();
        FRotator VelocityRotation = Velocity.Rotation();
        Direction = UKismetMathLibrary::NormalizedDeltaRotator(VelocityRotation, CharacterRotation).Yaw;
    }

    bIsAccelerating = MovementComp->GetCurrentAcceleration().SizeSquared() > 0.0f;
    bIsFalling = MovementComp->IsFalling();

    // 更新瞄准数据（从 Gameplay Tags）
    // 这些会通过 GameplayTagPropertyMap 自动更新
}
```

### 2.4 创建 Linked Anim Layer

在 Unreal Editor 中：

1. **创建 Linked Anim Layer Interface**
   - 内容浏览器 → 右键 → Animation → Anim Layer Interface
   - 命名为 `BALI_Locomotion`（Blueprint Anim Layer Interface）
   - 添加动画层函数：`Locomotion`、`FullBody`、`UpperBody`

2. **创建动画层实现**

```cpp
// 在蓝图中实现动画层接口
// 例如：ABP_Locomotion_Default（默认移动动画层）

Event Graph:
- 实现 Locomotion 函数
  - 返回姿势：BlendSpace（基于 Speed 和 Direction）
  
- 实现 FullBody 函数
  - Layered Blend Per Bone
    - Base Pose: Locomotion
    - Overlay: Combat Actions
    
- 实现 UpperBody 函数
  - Slot 'UpperBodySlot'
  - Default: Aim Offset
```

3. **在主动画蓝图中链接**

```
AnimGraph:
  Locomotion State Machine
    → Linked Anim Layer (Interface: BALI_Locomotion, Layer: Locomotion)
      → Layered Blend Per Bone
        → Base Layer: Full Body
        → Upper Body: Linked Anim Layer (Layer: UpperBody)
          → Final Pose
```

---

## 3. Motion Matching 系统详解

### 3.1 什么是 Motion Matching？

Motion Matching 是 UE5 引入的下一代动画技术，通过姿势搜索实现高质量、响应式的角色动画。

**核心概念：**

- **Pose Search Database（姿势搜索数据库）**：包含所有动画片段的姿势信息
- **Query（查询）**：当前角色状态（位置、速度、加速度等）
- **Best Match（最佳匹配）**：数据库中最接近查询的姿势
- **Seamless Transition（无缝过渡）**：自动混合到最佳匹配姿势

### 3.2 Motion Matching 优势

| 传统状态机 | Motion Matching |
|-----------|-----------------|
| 预定义转换规则 | 动态姿势搜索 |
| 转换延迟明显 | 即时响应 |
| 需要大量转换动画 | 重用现有动画 |
| 方向变化不自然 | 平滑的方向调整 |
| 难以适应地形 | 自适应步幅和节奏 |

### 3.3 启用 Motion Matching 插件

1. **编辑器设置**
   - Edit → Plugins
   - 搜索 "Pose Search"
   - 启用 `Pose Search` 和 `Animation Warping`
   - 重启编辑器

2. **项目设置**
   ```ini
   ; DefaultEngine.ini
   [/Script/PoseSearch.PoseSearchSettings]
   bEnableMotionMatching=True
   MaxSearchTime=0.1
   ```

### 3.4 创建 Pose Search Database

**步骤 1：创建数据库资产**

1. 内容浏览器 → 右键 → Animation → Pose Search → Pose Search Database
2. 命名为 `PSD_Locomotion`

**步骤 2：配置数据库**

```cpp
// 数据库配置（在编辑器中设置）
Pose Search Database Settings:
  - Schema: 选择或创建 Pose Search Schema
  - Sampling Interval: 0.1 (每 0.1 秒采样一次)
  - Trajectory Times: [-0.5, -0.25, 0.0, 0.25, 0.5] (过去和未来轨迹)
```

**步骤 3：添加动画序列**

```cpp
// 在 Pose Search Database 中添加动画
Sequences:
  - Idle
  - Walk_Fwd
  - Walk_Bwd
  - Walk_Left
  - Walk_Right
  - Run_Fwd
  - Run_Bwd
  - Run_Left
  - Run_Right
  - Sprint
  - Jump_Start
  - Jump_Loop
  - Jump_Land
  - Stop_Fwd
  - Stop_Bwd
  - Pivot_Left
  - Pivot_Right
```

### 3.5 Pose Search Schema 配置

Pose Search Schema 定义了如何评估姿势相似度：

```cpp
// Schema Configuration
Channels:
  1. Trajectory:
     - Sample Times: [-0.5, -0.25, 0.0, 0.25, 0.5]
     - Weight: 1.0
     - Include Position: Yes
     - Include Velocity: Yes
     - Include Facing Direction: Yes

  2. Bone Poses:
     - Bones: [Pelvis, Spine, Head, LeftHand, RightHand, LeftFoot, RightFoot]
     - Weight: 2.0
     - Include Position: Yes
     - Include Velocity: Yes

  3. Bone Velocities:
     - Bones: [LeftFoot, RightFoot]
     - Weight: 1.5
     - Max Speed: 500.0
```

### 3.6 在动画蓝图中使用 Motion Matching

**AnimGraph 配置：**

```
State Machine: Locomotion
  State: Movement
    Motion Matching Node:
      - Database: PSD_Locomotion
      - Trajectory Generator:
          - Current Velocity (from Pawn)
          - Desired Velocity (from Input)
          - Acceleration (calculated)
      
      Query Settings:
        - Weights:
            Trajectory: 1.0
            Pose: 2.0
        - Blend Time: 0.2
        - Max Active Blends: 4
      
      Output:
        → Inertialization (平滑过渡)
          → Final Pose
```

### 3.7 自定义 Motion Matching Node

```cpp
// CustomMotionMatchingNode.h
#pragma once

#include "Animation/AnimNode_PoseSearchBase.h"
#include "CustomMotionMatchingNode.generated.h"

USTRUCT(BlueprintInternalUseOnly)
struct FAnimNode_CustomMotionMatching : public FAnimNode_PoseSearchBase
{
    GENERATED_BODY()

public:
    // 自定义查询参数
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Settings, meta = (PinShownByDefault))
    float SpeedScale = 1.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Settings)
    bool bEnableRootMotion = true;

    // 覆盖查询生成
    virtual void BuildPoseSearchQuery(FPoseSearchQueryTrajectory& OutQuery) const override;

    // 覆盖结果处理
    virtual void ProcessSearchResult(const FPoseSearchResult& Result) override;

protected:
    virtual void Update_AnyThread(const FAnimationUpdateContext& Context) override;
    virtual void Evaluate_AnyThread(FPoseContext& Output) override;
};
```

```cpp
// CustomMotionMatchingNode.cpp
#include "CustomMotionMatchingNode.h"
#include "PoseSearch/PoseSearchLibrary.h"

void FAnimNode_CustomMotionMatching::BuildPoseSearchQuery(FPoseSearchQueryTrajectory& OutQuery) const
{
    Super::BuildPoseSearchQuery(OutQuery);

    // 自定义轨迹预测
    if (const USkeletalMeshComponent* SkelMeshComp = GetOwningComponent())
    {
        if (const AActor* Owner = SkelMeshComp->GetOwner())
        {
            FVector CurrentVelocity = Owner->GetVelocity();
            FVector ScaledVelocity = CurrentVelocity * SpeedScale;

            // 修改查询轨迹
            for (FPoseSearchQueryTrajectorySample& Sample : OutQuery.Samples)
            {
                Sample.Velocity = ScaledVelocity;
            }
        }
    }
}

void FAnimNode_CustomMotionMatching::ProcessSearchResult(const FPoseSearchResult& Result)
{
    Super::ProcessSearchResult(Result);

    // 自定义结果处理（例如调试输出）
    UE_LOG(LogAnimation, Verbose, TEXT("Motion Matching: Sequence=%s, Time=%.2f, Cost=%.2f"),
        *Result.SequenceName.ToString(), Result.Time, Result.Cost);
}

void FAnimNode_CustomMotionMatching::Update_AnyThread(const FAnimationUpdateContext& Context)
{
    Super::Update_AnyThread(Context);

    // 更新自定义参数
}

void FAnimNode_CustomMotionMatching::Evaluate_AnyThread(FPoseContext& Output)
{
    Super::Evaluate_AnyThread(Output);

    // 应用 Root Motion
    if (bEnableRootMotion)
    {
        // Root Motion 处理逻辑
    }
}
```

---

## 4. 与 GAS 动画通知集成

### 4.1 Lyra 的 Context Effects 通知

Lyra 使用 `AnimNotify_LyraContextEffects` 实现基于上下文的动画效果：

```cpp
// AnimNotify_LyraContextEffects.h
UCLASS(MinimalAPI, const, hidecategories=Object, CollapseCategories, 
       Config = Game, meta=(DisplayName="Play Context Effects"))
class UAnimNotify_LyraContextEffects : public UAnimNotify
{
    GENERATED_BODY()

public:
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                       const FAnimNotifyEventReference& EventReference) override;

    // 要播放的效果（通过 Gameplay Tag 标识）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify", meta = (ExposeOnSpawn = true))
    FGameplayTag Effect;

    // 位置偏移
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify")
    FVector LocationOffset = FVector::ZeroVector;

    // 旋转偏移
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify")
    FRotator RotationOffset = FRotator::ZeroRotator;

    // VFX 属性
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify")
    FLyraContextEffectAnimNotifyVFXSettings VFXProperties;

    // 音频属性
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify")
    FLyraContextEffectAnimNotifyAudioSettings AudioProperties;

    // 是否附加到骨骼/插槽
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AttachmentProperties")
    uint32 bAttached : 1;

    // 附加的插槽名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AttachmentProperties", 
              meta = (EditCondition = "bAttached"))
    FName SocketName;

    // 是否执行追踪（用于获取表面类型）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify")
    uint32 bPerformTrace : 1;

    // 追踪属性
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify", 
              meta = (EditCondition = "bPerformTrace"))
    FLyraContextEffectAnimNotifyTraceSettings TraceProperties;
};
```

### 4.2 创建 GAS 动画通知

```cpp
// AnimNotify_GameplayEvent.h
#pragma once

#include "Animation/AnimNotifies/AnimNotify.h"
#include "GameplayTagContainer.h"
#include "AnimNotify_GameplayEvent.generated.h"

UCLASS()
class UAnimNotify_GameplayEvent : public UAnimNotify
{
    GENERATED_BODY()

public:
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
                       const FAnimNotifyEventReference& EventReference) override;

    // 要发送的 Gameplay Event
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Gameplay")
    FGameplayTag EventTag;

    // Event Payload（可选数据）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Gameplay")
    float EventMagnitude = 1.0f;
};
```

```cpp
// AnimNotify_GameplayEvent.cpp
#include "AnimNotify_GameplayEvent.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemGlobals.h"

void UAnimNotify_GameplayEvent::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
                                       const FAnimNotifyEventReference& EventReference)
{
    Super::Notify(MeshComp, Animation, EventReference);

    if (!MeshComp || !MeshComp->GetOwner())
    {
        return;
    }

    // 获取 Ability System Component
    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(MeshComp->GetOwner());
    if (!ASC)
    {
        return;
    }

    // 构建 Event Data
    FGameplayEventData EventData;
    EventData.EventTag = EventTag;
    EventData.EventMagnitude = EventMagnitude;
    EventData.Instigator = MeshComp->GetOwner();
    EventData.Target = MeshComp->GetOwner();

    // 发送 Gameplay Event
    ASC->HandleGameplayEvent(EventTag, &EventData);

    UE_LOG(LogAnimation, Log, TEXT("AnimNotify: Sent Gameplay Event '%s' with Magnitude %.2f"),
        *EventTag.ToString(), EventMagnitude);
}
```

### 4.3 在 Gameplay Ability 中响应动画事件

```cpp
// GA_MeleeAttack.h
UCLASS()
class UGA_MeleeAttack : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                 const FGameplayAbilityActorInfo* ActorInfo,
                                 const FGameplayAbilityActivationInfo ActivationInfo,
                                 const FGameplayEventData* TriggerEventData) override;

protected:
    UFUNCTION()
    void OnWeaponHit(FGameplayEventData Payload);

    UPROPERTY(EditDefaultsOnly, Category = "Combat")
    FGameplayTag WeaponHitEventTag;

    UPROPERTY(EditDefaultsOnly, Category = "Combat")
    UAnimMontage* AttackMontage;
};
```

```cpp
// GA_MeleeAttack.cpp
#include "GA_MeleeAttack.h"
#include "AbilitySystemComponent.h"

void UGA_MeleeAttack::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                      const FGameplayAbilityActorInfo* ActorInfo,
                                      const FGameplayAbilityActivationInfo ActivationInfo,
                                      const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
        return;
    }

    // 播放攻击动画
    PlayMontage(AttackMontage);

    // 监听武器命中事件
    FGameplayEventMulticastDelegate& Delegate = GetAbilitySystemComponentFromActorInfo()->GenericGameplayEventCallbacks.FindOrAdd(WeaponHitEventTag);
    Delegate.AddUObject(this, &UGA_MeleeAttack::OnWeaponHit);
}

void UGA_MeleeAttack::OnWeaponHit(FGameplayEventData Payload)
{
    // 执行命中检测
    // 应用伤害
    // 播放命中效果

    UE_LOG(LogAbilitySystem, Log, TEXT("Weapon Hit Event Received!"));
}
```

---

## 5. IK 和程序化动画

### 5.1 IK Rig 设置

**步骤 1：创建 IK Rig 资产**

1. 内容浏览器 → 右键 → Animation → IK Rig
2. 选择骨骼网格：`SKM_Character`
3. 命名为 `IKR_Character`

**步骤 2：配置 IK 目标**

```cpp
// IK Rig Configuration (在编辑器中设置)
Root Bone: root

Solvers:
  1. Two Bone IK (Left Leg):
     - Root Bone: thigh_l
     - End Bone: foot_l
     - Goal: LeftFootIK
  
  2. Two Bone IK (Right Leg):
     - Root Bone: thigh_r
     - End Bone: foot_r
     - Goal: RightFootIK
  
  3. FABRIK (Left Arm):
     - Root Bone: clavicle_l
     - End Bone: hand_l
     - Goal: LeftHandIK
  
  4. FABRIK (Right Arm):
     - Root Bone: clavicle_r
     - End Bone: hand_r
     - Goal: RightHandIK
  
  5. Aim (Head):
     - Aim Bone: head
     - Goal: HeadLookAt

Goals:
  - LeftFootIK (Foot placement)
  - RightFootIK (Foot placement)
  - LeftHandIK (Weapon holding)
  - RightHandIK (Weapon holding)
  - HeadLookAt (Head tracking)
```

### 5.2 IK Retargeter

**创建 IK Retargeter 用于动画重定向：**

1. 内容浏览器 → 右键 → Animation → IK Retargeter
2. Source IK Rig: `IKR_SourceCharacter`
3. Target IK Rig: `IKR_Character`
4. 命名为 `IKRig_SourceToCharacter`

### 5.3 Foot Placement IK（脚步放置）

```cpp
// FootPlacementIK.h
#pragma once

#include "Animation/AnimInstance.h"
#include "FootPlacementIK.generated.h"

USTRUCT(BlueprintType)
struct FFootPlacementData
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FVector IKLocation;

    UPROPERTY(BlueprintReadWrite)
    FRotator IKRotation;

    UPROPERTY(BlueprintReadWrite)
    float IKAlpha;
};

UCLASS()
class UFootPlacementIK : public UObject
{
    GENERATED_BODY()

public:
    // 计算脚步 IK
    UFUNCTION(BlueprintCallable, Category = "IK")
    static FFootPlacementData CalculateFootIK(
        USkeletalMeshComponent* SkeletalMesh,
        FName SocketName,
        FName RootBoneName,
        float TraceDistance,
        float IKInterpSpeed,
        float DeltaTime);

private:
    static bool PerformFootTrace(
        const UWorld* World,
        const FVector& StartLocation,
        float TraceDistance,
        FHitResult& OutHit);
};
```

```cpp
// FootPlacementIK.cpp
#include "FootPlacementIK.h"
#include "Kismet/KismetSystemLibrary.h"
#include "Kismet/KismetMathLibrary.h"

FFootPlacementData UFootPlacementIK::CalculateFootIK(
    USkeletalMeshComponent* SkeletalMesh,
    FName SocketName,
    FName RootBoneName,
    float TraceDistance,
    float IKInterpSpeed,
    float DeltaTime)
{
    FFootPlacementData Result;
    Result.IKAlpha = 0.0f;

    if (!SkeletalMesh)
    {
        return Result;
    }

    const UWorld* World = SkeletalMesh->GetWorld();
    if (!World)
    {
        return Result;
    }

    // 获取脚部插槽位置
    FVector FootLocation = SkeletalMesh->GetSocketLocation(SocketName);
    FVector RootLocation = SkeletalMesh->GetSocketLocation(RootBoneName);

    // 向下追踪地面
    FHitResult HitResult;
    if (PerformFootTrace(World, FootLocation, TraceDistance, HitResult))
    {
        // 计算 IK 偏移
        float OffsetZ = (FootLocation.Z - HitResult.ImpactPoint.Z);
        
        if (FMath::Abs(OffsetZ) > 1.0f) // 阈值：1cm
        {
            // 目标 IK 位置
            FVector TargetLocation = FVector(0.0f, 0.0f, -OffsetZ);
            
            // 平滑插值
            Result.IKLocation = FMath::VInterpTo(
                FVector::ZeroVector,
                TargetLocation,
                DeltaTime,
                IKInterpSpeed
            );

            // 计算地面旋转
            FRotator GroundRotation = UKismetMathLibrary::MakeRotFromZX(
                HitResult.ImpactNormal,
                SkeletalMesh->GetForwardVector()
            );

            Result.IKRotation = GroundRotation;
            Result.IKAlpha = 1.0f;
        }
    }

    return Result;
}

bool UFootPlacementIK::PerformFootTrace(
    const UWorld* World,
    const FVector& StartLocation,
    float TraceDistance,
    FHitResult& OutHit)
{
    FVector StartTrace = StartLocation + FVector(0.0f, 0.0f, 50.0f);
    FVector EndTrace = StartLocation - FVector(0.0f, 0.0f, TraceDistance);

    TArray<AActor*> ActorsToIgnore;

    return UKismetSystemLibrary::LineTraceSingle(
        World,
        StartTrace,
        EndTrace,
        UEngineTypes::ConvertToTraceType(ECC_Visibility),
        false,
        ActorsToIgnore,
        EDrawDebugTrace::None,
        OutHit,
        true
    );
}
```

### 5.4 在动画蓝图中应用 IK

```
AnimGraph:
  Cached Pose 'BasePose' → Get Cached Pose
  
  Leg IK Branch:
    Get Cached Pose 'BasePose'
      → Two Bone IK (Left Foot)
        - IK Bone: foot_l
        - Effector Location: LeftFootIKLocation
        - Joint Target Location: LeftKneeTarget
      → Two Bone IK (Right Foot)
        - IK Bone: foot_r
        - Effector Location: RightFootIKLocation
        - Joint Target Location: RightKneeTarget
  
  Hand IK Branch (for Weapon):
    Leg IK Output
      → Hand IK Retargeting
        - Left Hand: LeftHandIKLocation (from Weapon Socket)
        - Right Hand: RightHandIKLocation (calculated)
  
  Final:
    Hand IK Output → Final Pose
```

**Event Graph（计算 IK 数据）：**

```cpp
Event Blueprint Update Animation:
  // 左脚 IK
  LeftFootIKData = Calculate Foot IK(
    SocketName: foot_l,
    RootBoneName: root,
    TraceDistance: 100.0,
    IKInterpSpeed: 15.0,
    DeltaTime: DeltaTime
  )
  
  LeftFootIKLocation = LeftFootIKData.IKLocation
  LeftFootIKRotation = LeftFootIKData.IKRotation
  LeftFootIKAlpha = LeftFootIKData.IKAlpha

  // 右脚 IK（同样的逻辑）
  
  // 武器 IK（如果装备了武器）
  If (IsWeaponEquipped):
    LeftHandIKLocation = Get Socket Location (WeaponMesh, "GripPoint")
    RightHandIKLocation = Calculate from LeftHandIKLocation
```

---

## 6. 武器动画层系统

### 6.1 武器实例与动画层

Lyra 使用 `ULyraWeaponInstance` 管理武器相关的动画层：

```cpp
// LyraWeaponInstance.h（简化版）
UCLASS(MinimalAPI)
class ULyraWeaponInstance : public ULyraEquipmentInstance
{
    GENERATED_BODY()

public:
    virtual void OnEquipped() override;
    virtual void OnUnequipped() override;

protected:
    // 装备时的动画集
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Animation)
    FLyraAnimLayerSelectionSet EquippedAnimSet;

    // 未装备时的动画集
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Animation)
    FLyraAnimLayerSelectionSet UnequippedAnimSet;

    // 根据装备状态和装饰标签选择最佳动画层
    UFUNCTION(BlueprintCallable, BlueprintPure=false, Category=Animation)
    TSubclassOf<UAnimInstance> PickBestAnimLayer(bool bEquipped, const FGameplayTagContainer& CosmeticTags) const;
};
```

```cpp
// LyraWeaponInstance.cpp（部分实现）
TSubclassOf<UAnimInstance> ULyraWeaponInstance::PickBestAnimLayer(
    bool bEquipped, 
    const FGameplayTagContainer& CosmeticTags) const
{
    const FLyraAnimLayerSelectionSet& SetToQuery = bEquipped ? EquippedAnimSet : UnequippedAnimSet;
    return SetToQuery.SelectBestLayer(CosmeticTags);
}
```

### 6.2 创建武器动画层接口

**步骤 1：创建动画层接口**

1. 内容浏览器 → 右键 → Animation → Anim Layer Interface
2. 命名为 `BALI_WeaponAnimLayers`

**步骤 2：定义动画层函数**

```cpp
// 在 BALI_WeaponAnimLayers 中定义以下函数：

Functions:
  - ItemAnimLayer (Item Full Body)
    - Input: Base Pose
    - Output: Final Pose
  
  - ItemUpperBodyLayer (Item Upper Body Only)
    - Input: Base Pose
    - Output: Upper Body Pose
  
  - ItemAdditive (Item Additive Layer)
    - Input: Base Pose
    - Output: Additive Pose
```

### 6.3 实现步枪动画层

```cpp
// ABP_Rifle.uasset (Animation Blueprint)

// 实现 ItemAnimLayer
ItemAnimLayer:
  Input Pose
    → Layered Blend Per Bone
      - Base Layer: Input Pose
      - Overlay: Rifle Idle Pose
      - Blend Weight: 1.0
      - Bone Name: spine_01
      - Blend Depth: 1
    → Final Pose

// 实现 ItemUpperBodyLayer
ItemUpperBodyLayer:
  Input Pose
    → Slot 'UpperBodyRifleSlot'
      - Default: Rifle Aim Offset (Pitch: AimPitch, Yaw: AimYaw)
    → Hand IK Retargeting
      - Left Hand Target: Weapon_GripPoint
      - IK Alpha: 1.0
    → Final Pose

// 实现 ItemAdditive
ItemAdditive:
  Select:
    - If IsAiming:
        Rifle Aim Additive Pose
    - Else:
        Zero Additive Pose
```

### 6.4 在主动画蓝图中链接武器层

```cpp
// ABP_Lyra_Character.uasset

AnimGraph:
  Locomotion State Machine
    → Cache Pose 'FullBodyPose'
  
  // 链接武器动画层
  Get Cached Pose 'FullBodyPose'
    → Linked Anim Layer
        - Layer: ItemAnimLayer
        - Interface: BALI_WeaponAnimLayers
        - Instance Class: (Dynamic from Equipment)
    → Cache Pose 'ItemFullBodyPose'
  
  // 上半身层
  Get Cached Pose 'ItemFullBodyPose'
    → Layered Blend Per Bone
        - Base Layer: ItemFullBodyPose
        - Overlay: Linked Anim Layer (ItemUpperBodyLayer)
        - Bone Name: spine_01
        - Blend Depth: 1
    → Final Pose

Event Graph:
  Event Blueprint Update Animation:
    // 动态设置武器动画层
    WeaponInstance = Get Equipped Weapon Instance()
    
    If (IsValid(WeaponInstance)):
      AnimLayerClass = WeaponInstance.PickBestAnimLayer(True, CosmeticTags)
      Link Anim Class Layers(AnimLayerClass)
    Else:
      Unlink Anim Class Layers(BALI_WeaponAnimLayers)
```

### 6.5 武器切换动画

```cpp
// WeaponSwitchAnimController.h
UCLASS()
class UWeaponSwitchAnimController : public UObject
{
    GENERATED_BODY()

public:
    // 切换武器动画层
    UFUNCTION(BlueprintCallable, Category = "Animation")
    static void SwitchWeaponAnimLayer(
        UAnimInstance* AnimInstance,
        TSubclassOf<UAnimInstance> NewWeaponLayer,
        float BlendTime = 0.2f);

    // 播放武器装备动画
    UFUNCTION(BlueprintCallable, Category = "Animation")
    static void PlayEquipMontage(
        UAnimInstance* AnimInstance,
        UAnimMontage* EquipMontage,
        float PlayRate = 1.0f);
};
```

```cpp
// WeaponSwitchAnimController.cpp
void UWeaponSwitchAnimController::SwitchWeaponAnimLayer(
    UAnimInstance* AnimInstance,
    TSubclassOf<UAnimInstance> NewWeaponLayer,
    float BlendTime)
{
    if (!AnimInstance || !NewWeaponLayer)
    {
        return;
    }

    // 使用 Inertialization 进行平滑过渡
    AnimInstance->SetLinkedAnimClassLayers(NewWeaponLayer);
    AnimInstance->GetProxyOnAnyThread<FAnimInstanceProxy>()->RequestInertialization(BlendTime);

    UE_LOG(LogAnimation, Log, TEXT("Switched weapon anim layer to %s with blend time %.2f"),
        *NewWeaponLayer->GetName(), BlendTime);
}

void UWeaponSwitchAnimController::PlayEquipMontage(
    UAnimInstance* AnimInstance,
    UAnimMontage* EquipMontage,
    float PlayRate)
{
    if (!AnimInstance || !EquipMontage)
    {
        return;
    }

    AnimInstance->Montage_Play(EquipMontage, PlayRate);
    
    UE_LOG(LogAnimation, Log, TEXT("Playing equip montage: %s at rate %.2f"),
        *EquipMontage->GetName(), PlayRate);
}
```

---

## 7. 网络动画同步

### 7.1 Lyra 的网络同步架构

Lyra 使用优化的复制结构来减少带宽：

```cpp
// LyraCharacter.h（网络同步部分）
USTRUCT()
struct FLyraReplicatedAcceleration
{
    GENERATED_BODY()

    UPROPERTY()
    uint8 AccelXYRadians = 0;  // XY加速度方向，量化为 [0, 2*pi]

    UPROPERTY()
    uint8 AccelXYMagnitude = 0; // XY加速度大小，量化为 [0, MaxAcceleration]

    UPROPERTY()
    int8 AccelZ = 0;  // Z加速度，量化为 [-MaxAcceleration, MaxAcceleration]
};

// 用于快速共享移动更新的类型
USTRUCT()
struct FSharedRepMovement
{
    GENERATED_BODY()

    FSharedRepMovement();

    bool FillForCharacter(ACharacter* Character);
    bool Equals(const FSharedRepMovement& Other, ACharacter* Character) const;
    bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);

    UPROPERTY(Transient)
    FRepMovement RepMovement;

    UPROPERTY(Transient)
    float RepTimeStamp = 0.0f;

    UPROPERTY(Transient)
    uint8 RepMovementMode = 0;

    UPROPERTY(Transient)
    bool bProxyIsJumpForceApplied = false;

    UPROPERTY(Transient)
    bool bIsCrouched = false;
};
```

### 7.2 动画蒙太奇网络同步

```cpp
// NetworkedAnimMontageController.h
UCLASS()
class UNetworkedAnimMontageController : public UActorComponent
{
    GENERATED_BODY()

public:
    UNetworkedAnimMontageController();

    // 播放动画蒙太奇（会自动同步到所有客户端）
    UFUNCTION(BlueprintCallable, Category = "Animation")
    void PlayMontageMulticast(UAnimMontage* Montage, float PlayRate = 1.0f, FName StartSection = NAME_None);

protected:
    // 服务器 RPC
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerPlayMontage(UAnimMontage* Montage, float PlayRate, FName StartSection);

    // 多播 RPC
    UFUNCTION(NetMulticast, Reliable)
    void MulticastPlayMontage(UAnimMontage* Montage, float PlayRate, FName StartSection);

    // 复制的蒙太奇数据
    UPROPERTY(ReplicatedUsing = OnRep_MontageData)
    FAnimMontageNetData RepMontageData;

    UFUNCTION()
    void OnRep_MontageData();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
};

USTRUCT(BlueprintType)
struct FAnimMontageNetData
{
    GENERATED_BODY()

    UPROPERTY()
    TObjectPtr<UAnimMontage> Montage = nullptr;

    UPROPERTY()
    float Position = 0.0f;

    UPROPERTY()
    float PlayRate = 1.0f;

    UPROPERTY()
    FName SectionName = NAME_None;

    UPROPERTY()
    uint8 bIsPlaying : 1;

    UPROPERTY()
    uint8 RepCounter = 0; // 用于强制更新
};
```

```cpp
// NetworkedAnimMontageController.cpp
#include "NetworkedAnimMontageController.h"
#include "Net/UnrealNetwork.h"
#include "Animation/AnimInstance.h"

UNetworkedAnimMontageController::UNetworkedAnimMontageController()
{
    SetIsReplicatedByDefault(true);
}

void UNetworkedAnimMontageController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(UNetworkedAnimMontageController, RepMontageData);
}

void UNetworkedAnimMontageController::PlayMontageMulticast(UAnimMontage* Montage, float PlayRate, FName StartSection)
{
    if (!Montage)
    {
        return;
    }

    AActor* Owner = GetOwner();
    if (!Owner)
    {
        return;
    }

    // 如果在服务器或具有权限，调用服务器 RPC
    if (Owner->HasAuthority())
    {
        MulticastPlayMontage(Montage, PlayRate, StartSection);
    }
    else
    {
        ServerPlayMontage(Montage, PlayRate, StartSection);
    }
}

void UNetworkedAnimMontageController::ServerPlayMontage_Implementation(UAnimMontage* Montage, float PlayRate, FName StartSection)
{
    MulticastPlayMontage(Montage, PlayRate, StartSection);
}

bool UNetworkedAnimMontageController::ServerPlayMontage_Validate(UAnimMontage* Montage, float PlayRate, FName StartSection)
{
    return true;
}

void UNetworkedAnimMontageController::MulticastPlayMontage_Implementation(UAnimMontage* Montage, float PlayRate, FName StartSection)
{
    ACharacter* Character = Cast<ACharacter>(GetOwner());
    if (!Character || !Character->GetMesh())
    {
        return;
    }

    UAnimInstance* AnimInstance = Character->GetMesh()->GetAnimInstance();
    if (!AnimInstance)
    {
        return;
    }

    // 播放蒙太奇
    float Duration = AnimInstance->Montage_Play(Montage, PlayRate);
    
    if (StartSection != NAME_None)
    {
        AnimInstance->Montage_JumpToSection(StartSection, Montage);
    }

    // 更新复制数据
    RepMontageData.Montage = Montage;
    RepMontageData.Position = 0.0f;
    RepMontageData.PlayRate = PlayRate;
    RepMontageData.SectionName = StartSection;
    RepMontageData.bIsPlaying = true;
    RepMontageData.RepCounter++;

    UE_LOG(LogAnimation, Log, TEXT("Multicast Play Montage: %s (Duration: %.2f)"),
        *Montage->GetName(), Duration);
}

void UNetworkedAnimMontageController::OnRep_MontageData()
{
    // 当复制数据更新时，在客户端上播放蒙太奇
    if (RepMontageData.bIsPlaying && RepMontageData.Montage)
    {
        ACharacter* Character = Cast<ACharacter>(GetOwner());
        if (Character && Character->GetMesh())
        {
            UAnimInstance* AnimInstance = Character->GetMesh()->GetAnimInstance();
            if (AnimInstance)
            {
                AnimInstance->Montage_Play(RepMontageData.Montage, RepMontageData.PlayRate);
                
                if (RepMontageData.SectionName != NAME_None)
                {
                    AnimInstance->Montage_JumpToSection(RepMontageData.SectionName, RepMontageData.Montage);
                }
            }
        }
    }
}
```

### 7.3 动画状态网络同步优化

```cpp
// AnimationStateReplication.h
USTRUCT(BlueprintType)
struct FCompressedAnimationState
{
    GENERATED_BODY()

    // 量化的速度 (0-255 映射到 0-MaxSpeed)
    UPROPERTY()
    uint8 Speed = 0;

    // 量化的方向 (0-255 映射到 -180 到 180)
    UPROPERTY()
    uint8 Direction = 0;

    // 状态标志
    UPROPERTY()
    uint8 StateFlags = 0; // Bit 0: IsJumping, Bit 1: IsCrouching, Bit 2: IsAiming, etc.

    // 辅助函数
    void CompressFrom(float InSpeed, float InDirection, bool bIsJumping, bool bIsCrouching, bool bIsAiming);
    void DecompressTo(float& OutSpeed, float& OutDirection, bool& bOutIsJumping, bool& bOutIsCrouching, bool& bOutIsAiming) const;

    bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);
};

template<>
struct TStructOpsTypeTraits<FCompressedAnimationState> : public TStructOpsTypeTraitsBase2<FCompressedAnimationState>
{
    enum
    {
        WithNetSerializer = true,
    };
};
```

```cpp
// AnimationStateReplication.cpp
void FCompressedAnimationState::CompressFrom(float InSpeed, float InDirection, bool bIsJumping, bool bIsCrouching, bool bIsAiming)
{
    // 压缩速度 (假设 MaxSpeed = 600)
    Speed = FMath::Clamp(FMath::RoundToInt((InSpeed / 600.0f) * 255.0f), 0, 255);

    // 压缩方向 (-180 到 180 → 0 到 255)
    Direction = FMath::Clamp(FMath::RoundToInt(((InDirection + 180.0f) / 360.0f) * 255.0f), 0, 255);

    // 状态标志
    StateFlags = 0;
    StateFlags |= bIsJumping ? (1 << 0) : 0;
    StateFlags |= bIsCrouching ? (1 << 1) : 0;
    StateFlags |= bIsAiming ? (1 << 2) : 0;
}

void FCompressedAnimationState::DecompressTo(float& OutSpeed, float& OutDirection, bool& bOutIsJumping, bool& bOutIsCrouching, bool& bOutIsAiming) const
{
    // 解压速度
    OutSpeed = (Speed / 255.0f) * 600.0f;

    // 解压方向
    OutDirection = (Direction / 255.0f) * 360.0f - 180.0f;

    // 状态标志
    bOutIsJumping = (StateFlags & (1 << 0)) != 0;
    bOutIsCrouching = (StateFlags & (1 << 1)) != 0;
    bOutIsAiming = (StateFlags & (1 << 2)) != 0;
}

bool FCompressedAnimationState::NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
{
    Ar << Speed;
    Ar << Direction;
    Ar << StateFlags;

    bOutSuccess = true;
    return true;
}
```

---

## 8. 实战案例 1：第一人称/第三人称动画系统

### 8.1 双视角动画架构

```cpp
// DualPerspectiveAnimInstance.h
UCLASS()
class UDualPerspectiveAnimInstance : public ULyraAnimInstance
{
    GENERATED_BODY()

public:
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;

protected:
    // 是否使用第一人称视角
    UPROPERTY(BlueprintReadOnly, Category = "Perspective")
    bool bIsFirstPerson = false;

    // 第一人称专用变量
    UPROPERTY(BlueprintReadOnly, Category = "First Person")
    FRotator FirstPersonAimRotation;

    UPROPERTY(BlueprintReadOnly, Category = "First Person")
    float FirstPersonSway = 0.0f;

    // 第三人称专用变量
    UPROPERTY(BlueprintReadOnly, Category = "Third Person")
    FRotator ThirdPersonSpineRotation;

    UPROPERTY(BlueprintReadOnly, Category = "Third Person")
    float ThirdPersonLeanAmount = 0.0f;

private:
    void UpdateFirstPersonAnimation(float DeltaSeconds);
    void UpdateThirdPersonAnimation(float DeltaSeconds);
};
```

### 8.2 实现双视角动画逻辑

```cpp
// DualPerspectiveAnimInstance.cpp
void UDualPerspectiveAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);

    ACharacter* Character = Cast<ACharacter>(GetOwningActor());
    if (!Character)
    {
        return;
    }

    // 判断是否为第一人称（基于相机位置）
    APlayerController* PC = Cast<APlayerController>(Character->GetController());
    if (PC && PC->PlayerCameraManager)
    {
        FVector CameraLocation = PC->PlayerCameraManager->GetCameraLocation();
        FVector HeadLocation = Character->GetMesh()->GetSocketLocation(FName("head"));
        
        float DistanceToHead = FVector::Distance(CameraLocation, HeadLocation);
        bIsFirstPerson = (DistanceToHead < 30.0f); // 阈值：30cm
    }

    // 根据视角更新不同的动画数据
    if (bIsFirstPerson)
    {
        UpdateFirstPersonAnimation(DeltaSeconds);
    }
    else
    {
        UpdateThirdPersonAnimation(DeltaSeconds);
    }
}

void UDualPerspectiveAnimInstance::UpdateFirstPersonAnimation(float DeltaSeconds)
{
    ACharacter* Character = Cast<ACharacter>(GetOwningActor());
    APlayerController* PC = Cast<APlayerController>(Character->GetController());
    
    if (!PC)
    {
        return;
    }

    // 获取控制器旋转（第一人称需要精确的瞄准）
    FirstPersonAimRotation = PC->GetControlRotation();

    // 计算武器摇摆（基于速度）
    FVector Velocity = Character->GetVelocity();
    float Speed = Velocity.Size();
    FirstPersonSway = FMath::Sin(GetWorld()->GetTimeSeconds() * 10.0f) * (Speed / 600.0f) * 2.0f;
}

void UDualPerspectiveAnimInstance::UpdateThirdPersonAnimation(float DeltaSeconds)
{
    ACharacter* Character = Cast<ACharacter>(GetOwningActor());
    
    if (!Character)
    {
        return;
    }

    // 计算脊椎旋转（用于上半身跟随瞄准）
    FRotator ActorRotation = Character->GetActorRotation();
    FRotator ControlRotation = Character->GetControlRotation();
    
    ThirdPersonSpineRotation = FRotator(
        ControlRotation.Pitch - ActorRotation.Pitch,
        ControlRotation.Yaw - ActorRotation.Yaw,
        0.0f
    );

    // 限制旋转范围
    ThirdPersonSpineRotation.Pitch = FMath::ClampAngle(ThirdPersonSpineRotation.Pitch, -60.0f, 60.0f);
    ThirdPersonSpineRotation.Yaw = FMath::ClampAngle(ThirdPersonSpineRotation.Yaw, -90.0f, 90.0f);

    // 计算倾斜量（基于加速度方向）
    UCharacterMovementComponent* MovementComp = Character->GetCharacterMovement();
    FVector Acceleration = MovementComp->GetCurrentAcceleration();
    
    if (Acceleration.SizeSquared() > 0.0f)
    {
        FVector AccelDirection = Acceleration.GetSafeNormal();
        FVector RightVector = Character->GetActorRightVector();
        ThirdPersonLeanAmount = FVector::DotProduct(AccelDirection, RightVector) * 0.5f;
    }
    else
    {
        // 平滑回到中立位置
        ThirdPersonLeanAmount = FMath::FInterpTo(ThirdPersonLeanAmount, 0.0f, DeltaSeconds, 5.0f);
    }
}
```

### 8.3 AnimGraph 配置

**第一人称动画图表：**

```
State Machine: FirstPerson_Locomotion
  State: Idle/Walk/Run
    Motion Matching (Database: PSD_FirstPerson_Locomotion)
      → Control Rig (FirstPerson_Arms_CR)
        - Apply weapon sway
        - Apply recoil animation
      → Weapon Pose
        → Final Pose (Arms Only)

Layers:
  - Base Layer: FirstPerson_FullBody
  - Weapon Layer: Linked Anim Layer (FirstPerson_WeaponLayer)
  - Additive Layer: FirstPerson_Recoil_Additive
```

**第三人称动画图表：**

```
State Machine: ThirdPerson_Locomotion
  State: Idle/Walk/Run
    Motion Matching (Database: PSD_ThirdPerson_Locomotion)
      → Layered Blend Per Bone (spine_01)
        - Base: Full Body Locomotion
        - Overlay: Upper Body Aiming
      → Two Bone IK (Foot Placement)
      → Modify Bone (Spine Rotation)
        → Final Pose

Layers:
  - Base Layer: ThirdPerson_FullBody
  - Weapon Layer: Linked Anim Layer (ThirdPerson_WeaponLayer)
  - Lean Layer: ThirdPerson_Lean_Additive
```

### 8.4 视角切换逻辑

```cpp
// PerspectiveSwitchComponent.h
UCLASS()
class UPerspectiveSwitchComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UPerspectiveSwitchComponent();

    // 切换视角
    UFUNCTION(BlueprintCallable, Category = "Perspective")
    void SwitchPerspective(bool bNewIsFirstPerson);

    // 获取当前视角
    UFUNCTION(BlueprintPure, Category = "Perspective")
    bool IsFirstPerson() const { return bIsFirstPerson; }

protected:
    virtual void BeginPlay() override;

    UPROPERTY(EditAnywhere, Category = "Perspective")
    TSubclassOf<UAnimInstance> FirstPersonAnimClass;

    UPROPERTY(EditAnywhere, Category = "Perspective")
    TSubclassOf<UAnimInstance> ThirdPersonAnimClass;

    UPROPERTY(EditAnywhere, Category = "Perspective")
    float FirstPersonCameraDistance = 0.0f;

    UPROPERTY(EditAnywhere, Category = "Perspective")
    float ThirdPersonCameraDistance = 300.0f;

private:
    bool bIsFirstPerson = false;

    void ApplyFirstPersonSettings();
    void ApplyThirdPersonSettings();
};
```

```cpp
// PerspectiveSwitchComponent.cpp
void UPerspectiveSwitchComponent::SwitchPerspective(bool bNewIsFirstPerson)
{
    if (bIsFirstPerson == bNewIsFirstPerson)
    {
        return;
    }

    bIsFirstPerson = bNewIsFirstPerson;

    if (bIsFirstPerson)
    {
        ApplyFirstPersonSettings();
    }
    else
    {
        ApplyThirdPersonSettings();
    }
}

void UPerspectiveSwitchComponent::ApplyFirstPersonSettings()
{
    ACharacter* Character = Cast<ACharacter>(GetOwner());
    if (!Character)
    {
        return;
    }

    // 切换动画蓝图
    if (FirstPersonAnimClass)
    {
        Character->GetMesh()->SetAnimInstanceClass(FirstPersonAnimClass);
    }

    // 调整相机距离
    // (假设使用 Spring Arm Component)
    // SpringArm->TargetArmLength = FirstPersonCameraDistance;

    // 隐藏第三人称身体（只显示手臂）
    Character->GetMesh()->SetOwnerNoSee(false);
    Character->GetMesh()->SetOnlyOwnerSee(true);

    UE_LOG(LogAnimation, Log, TEXT("Switched to First Person"));
}

void UPerspectiveSwitchComponent::ApplyThirdPersonSettings()
{
    ACharacter* Character = Cast<ACharacter>(GetOwner());
    if (!Character)
    {
        return;
    }

    // 切换动画蓝图
    if (ThirdPersonAnimClass)
    {
        Character->GetMesh()->SetAnimInstanceClass(ThirdPersonAnimClass);
    }

    // 调整相机距离
    // SpringArm->TargetArmLength = ThirdPersonCameraDistance;

    // 显示完整身体
    Character->GetMesh()->SetOwnerNoSee(false);
    Character->GetMesh()->SetOnlyOwnerSee(false);

    UE_LOG(LogAnimation, Log, TEXT("Switched to Third Person"));
}
```

---

## 9. 实战案例 2：载具动画集成

### 9.1 载具动画层接口

```cpp
// VehicleAnimLayerInterface (BALI_VehicleAnimLayers)

Functions:
  - VehicleFullBody
    - Input: Base Pose
    - Output: Vehicle Driving Pose
  
  - VehicleUpperBody
    - Input: Base Pose
    - Output: Steering & Control Pose
  
  - VehicleTransition
    - Input: Base Pose, Transition Alpha
    - Output: Enter/Exit Blend Pose
```

### 9.2 载具动画实例

```cpp
// VehicleAnimInstance.h
UCLASS()
class UVehicleAnimInstance : public ULyraAnimInstance
{
    GENERATED_BODY()

public:
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;

protected:
    // 载具数据
    UPROPERTY(BlueprintReadOnly, Category = "Vehicle")
    bool bIsInVehicle = false;

    UPROPERTY(BlueprintReadOnly, Category = "Vehicle")
    float SteeringAngle = 0.0f;

    UPROPERTY(BlueprintReadOnly, Category = "Vehicle")
    float VehicleSpeed = 0.0f;

    UPROPERTY(BlueprintReadOnly, Category = "Vehicle")
    FVector VehicleAcceleration;

    // 过渡数据
    UPROPERTY(BlueprintReadOnly, Category = "Vehicle")
    float EnterExitAlpha = 0.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Vehicle")
    float EnterExitBlendTime = 0.5f;
};
```

```cpp
// VehicleAnimInstance.cpp
void UVehicleAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);

    ACharacter* Character = Cast<ACharacter>(GetOwningActor());
    if (!Character)
    {
        return;
    }

    // 检查是否在载具中（通过 Gameplay Tag）
    bIsInVehicle = Character->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag(FName("State.Vehicle.Driving")));

    if (bIsInVehicle)
    {
        // 获取载具数据
        AActor* Vehicle = Character->GetAttachParentActor();
        if (Vehicle)
        {
            // 假设载具有 VehicleMovementComponent
            // VehicleSpeed = VehicleMovementComponent->GetForwardSpeed();
            // SteeringAngle = VehicleMovementComponent->GetSteeringInput() * 45.0f;

            // 简化版：从载具速度获取
            VehicleSpeed = Vehicle->GetVelocity().Size();
            VehicleAcceleration = (Vehicle->GetVelocity() - Character->GetVelocity()) / DeltaSeconds;
        }

        // 过渡到载具姿势
        EnterExitAlpha = FMath::FInterpTo(EnterExitAlpha, 1.0f, DeltaSeconds, 1.0f / EnterExitBlendTime);
    }
    else
    {
        // 过渡回正常姿势
        EnterExitAlpha = FMath::FInterpTo(EnterExitAlpha, 0.0f, DeltaSeconds, 1.0f / EnterExitBlendTime);
        SteeringAngle = 0.0f;
        VehicleSpeed = 0.0f;
    }
}
```

### 9.3 载具动画图表

```
AnimGraph:
  // 基础移动
  State Machine: Locomotion
    → Cache Pose 'LocomotionPose'
  
  // 载具姿势
  Select:
    - If bIsInVehicle:
        Linked Anim Layer (VehicleFullBody)
          - Steering Blend Space (Steering Angle)
          - Speed Blend Space (Vehicle Speed)
    - Else:
        Get Cached Pose 'LocomotionPose'
    
  → Blend (EnterExitAlpha)
    - A: Locomotion Pose
    - B: Vehicle Pose
  
  → Slot 'VehicleTransitionSlot' (for Enter/Exit Montages)
    → Final Pose
```

### 9.4 进入/退出载具动画

```cpp
// VehicleTransitionController.h
UCLASS()
class UVehicleTransitionController : public UObject
{
    GENERATED_BODY()

public:
    // 播放进入载具动画
    UFUNCTION(BlueprintCallable, Category = "Vehicle")
    static void PlayEnterVehicleMontage(
        ACharacter* Character,
        AActor* Vehicle,
        UAnimMontage* EnterMontage,
        FName SeatSocket);

    // 播放退出载具动画
    UFUNCTION(BlueprintCallable, Category = "Vehicle")
    static void PlayExitVehicleMontage(
        ACharacter* Character,
        UAnimMontage* ExitMontage);

protected:
    UFUNCTION()
    static void OnEnterMontageCompleted(UAnimMontage* Montage, bool bInterrupted, ACharacter* Character, AActor* Vehicle, FName SeatSocket);

    UFUNCTION()
    static void OnExitMontageCompleted(UAnimMontage* Montage, bool bInterrupted, ACharacter* Character);
};
```

```cpp
// VehicleTransitionController.cpp
void UVehicleTransitionController::PlayEnterVehicleMontage(
    ACharacter* Character,
    AActor* Vehicle,
    UAnimMontage* EnterMontage,
    FName SeatSocket)
{
    if (!Character || !Vehicle || !EnterMontage)
    {
        return;
    }

    UAnimInstance* AnimInstance = Character->GetMesh()->GetAnimInstance();
    if (!AnimInstance)
    {
        return;
    }

    // 播放进入动画
    float Duration = AnimInstance->Montage_Play(EnterMontage);

    // 绑定完成回调
    FOnMontageEnded EndDelegate;
    EndDelegate.BindStatic(&UVehicleTransitionController::OnEnterMontageCompleted, Character, Vehicle, SeatSocket);
    AnimInstance->Montage_SetEndDelegate(EndDelegate, EnterMontage);

    UE_LOG(LogAnimation, Log, TEXT("Playing Enter Vehicle Montage (Duration: %.2f)"), Duration);
}

void UVehicleTransitionController::OnEnterMontageCompleted(
    UAnimMontage* Montage,
    bool bInterrupted,
    ACharacter* Character,
    AActor* Vehicle,
    FName SeatSocket)
{
    if (!Character || !Vehicle || bInterrupted)
    {
        return;
    }

    // 附加角色到载具座位
    Character->AttachToComponent(
        Vehicle->GetRootComponent(),
        FAttachmentTransformRules::SnapToTargetNotIncludingScale,
        SeatSocket
    );

    // 禁用角色移动
    UCharacterMovementComponent* MovementComp = Character->GetCharacterMovement();
    if (MovementComp)
    {
        MovementComp->DisableMovement();
    }

    // 设置 Gameplay Tag
    if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Character))
    {
        ASC->AddLooseGameplayTag(FGameplayTag::RequestGameplayTag(FName("State.Vehicle.Driving")));
    }

    UE_LOG(LogAnimation, Log, TEXT("Character entered vehicle"));
}

void UVehicleTransitionController::PlayExitVehicleMontage(
    ACharacter* Character,
    UAnimMontage* ExitMontage)
{
    if (!Character || !ExitMontage)
    {
        return;
    }

    UAnimInstance* AnimInstance = Character->GetMesh()->GetAnimInstance();
    if (!AnimInstance)
    {
        return;
    }

    // 播放退出动画
    float Duration = AnimInstance->Montage_Play(ExitMontage);

    // 绑定完成回调
    FOnMontageEnded EndDelegate;
    EndDelegate.BindStatic(&UVehicleTransitionController::OnExitMontageCompleted, Character);
    AnimInstance->Montage_SetEndDelegate(EndDelegate, ExitMontage);

    UE_LOG(LogAnimation, Log, TEXT("Playing Exit Vehicle Montage (Duration: %.2f)"), Duration);
}

void UVehicleTransitionController::OnExitMontageCompleted(
    UAnimMontage* Montage,
    bool bInterrupted,
    ACharacter* Character)
{
    if (!Character || bInterrupted)
    {
        return;
    }

    // 分离角色
    Character->DetachFromActor(FDetachmentTransformRules::KeepWorldTransform);

    // 恢复角色移动
    UCharacterMovementComponent* MovementComp = Character->GetCharacterMovement();
    if (MovementComp)
    {
        MovementComp->SetMovementMode(MOVE_Walking);
    }

    // 移除 Gameplay Tag
    if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Character))
    {
        ASC->RemoveLooseGameplayTag(FGameplayTag::RequestGameplayTag(FName("State.Vehicle.Driving")));
    }

    UE_LOG(LogAnimation, Log, TEXT("Character exited vehicle"));
}
```

---

## 10. 性能优化技巧

### 10.1 动画蓝图优化

**1. 使用 Cached Pose（缓存姿势）**

```
// 好的做法
AnimGraph:
  Locomotion State Machine
    → Cache Pose 'BasePose'
  
  Get Cached Pose 'BasePose'
    → Upper Body Modifications
  
  Get Cached Pose 'BasePose'
    → Lower Body Modifications

// 避免的做法（重复计算）
AnimGraph:
  Locomotion State Machine
    → Upper Body Modifications
  
  Locomotion State Machine (重复!)
    → Lower Body Modifications
```

**2. 减少每帧更新的节点**

```cpp
// Event Graph 优化
Event Blueprint Update Animation:
  // 昂贵的计算：每 N 帧更新一次
  If (FrameCounter % 5 == 0):
    UpdateComplexIKTargets()
    UpdateDistanceMatching()
  
  FrameCounter++
  
  // 轻量级更新：每帧
  UpdateBasicMovementData()
```

**3. 使用 LOD（细节层次）**

```cpp
// AnimInstance.cpp
void UMyAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);

    // 根据 LOD 级别调整更新频率
    USkeletalMeshComponent* SkelMesh = GetSkelMeshComponent();
    if (SkelMesh)
    {
        int32 LODLevel = SkelMesh->GetPredictedLODLevel();
        
        if (LODLevel >= 2) // 远距离 LOD
        {
            // 跳过昂贵的计算
            return;
        }
        else if (LODLevel == 1) // 中距离 LOD
        {
            // 只执行关键更新
            UpdateCriticalAnimationData(DeltaSeconds);
        }
        else // LOD 0 (近距离)
        {
            // 完整更新
            UpdateFullAnimationData(DeltaSeconds);
        }
    }
}
```

### 10.2 Motion Matching 优化

**1. 优化 Pose Search Database**

```cpp
// Pose Search Database 设置
Settings:
  - Sampling Interval: 0.1 → 0.15 (减少采样频率)
  - Trajectory Times: 减少采样点数量
    Before: [-0.5, -0.25, 0.0, 0.25, 0.5]
    After: [-0.3, 0.0, 0.3]
  
  - Bone Selection: 只包含关键骨骼
    Before: [All Bones]
    After: [Pelvis, Spine, Head, LeftFoot, RightFoot]
  
  - Max Search Time: 0.1 → 0.05 (限制搜索时间)
```

**2. 使用异步搜索**

```cpp
// 在 Pose Search 设置中启用异步查询
Pose Search Settings:
  - Enable Async Query: True
  - Query Batch Size: 4
  - Max Concurrent Queries: 2
```

### 10.3 网络同步优化

**1. 减少复制频率**

```cpp
// Character.cpp
ALyraCharacter::ALyraCharacter()
{
    // 对于远程客户端，降低复制频率
    NetUpdateFrequency = 100.0f; // 默认值
    MinNetUpdateFrequency = 2.0f; // 最小值
}

void ALyraCharacter::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);

    // 根据重要性调整复制频率
    APlayerController* PC = Cast<APlayerController>(GetController());
    if (PC && PC->IsLocalController())
    {
        // 本地控制的角色：高频率
        NetUpdateFrequency = 100.0f;
    }
    else
    {
        // 其他角色：根据距离调整
        float Distance = GetDistanceTo(GetWorld()->GetFirstPlayerController()->GetPawn());
        
        if (Distance < 1000.0f)
        {
            NetUpdateFrequency = 50.0f;
        }
        else if (Distance < 3000.0f)
        {
            NetUpdateFrequency = 20.0f;
        }
        else
        {
            NetUpdateFrequency = 5.0f;
        }
    }
}
```

**2. 使用 Relevancy（相关性）**

```cpp
// Character.cpp
bool ALyraCharacter::IsNetRelevantFor(const AActor* RealViewer, const AActor* ViewTarget, const FVector& SrcLocation) const
{
    // 自定义相关性检查
    if (Super::IsNetRelevantFor(RealViewer, ViewTarget, SrcLocation))
    {
        // 距离检查
        float DistanceSquared = (SrcLocation - GetActorLocation()).SizeSquared();
        float MaxDistanceSquared = FMath::Square(NetCullDistanceSquared);
        
        if (DistanceSquared > MaxDistanceSquared)
        {
            return false;
        }

        // 视野检查（可选）
        // ...

        return true;
    }

    return false;
}
```

### 10.4 内存优化

**1. 使用动画压缩**

```cpp
// 在动画序列资产设置中：
Compression:
  - Compression Scheme: Automatic Compression
  - Rotation Format: Fixed 48 bits
  - Translation Format: Fixed 48 bits
  - Scale Format: None (如果不需要缩放)
  
  - Remove Trivial Keys: True
  - Max Error: 0.001 → 0.01 (增加误差容限以提高压缩率)
```

**2. 动画流式加载**

```cpp
// 将不常用的动画设置为流式加载
Animation Asset Settings:
  - Streaming: True
  - Chunk Size: 512 KB
  - Preload Duration: 1.0s
```

### 10.5 性能分析工具

**使用 Unreal Insights 分析动画性能：**

```bash
# 启动 Unreal Insights
UnrealEditor.exe -game -trace=cpu,frame,animation -statnamedevents

# 在编辑器中
Console Commands:
  - stat anim (显示动画统计)
  - stat animgraph (显示动画图表统计)
  - showdebug animation (显示调试信息)
```

**常见性能瓶颈：**

1. **过多的 Blend Nodes**：合并或使用 Layered Blend Per Bone
2. **未优化的 IK**：限制 IK 更新频率和迭代次数
3. **大量 Montage 实例**：重用 Montage，避免同时播放过多
4. **复杂的 Blueprint 逻辑**：将频繁调用的逻辑移到 C++

---

## 11. 总结与最佳实践

### 11.1 Lyra 动画系统设计原则

1. **模块化**：使用 Linked Anim Graphs 分离不同功能层
2. **数据驱动**：通过 Gameplay Tags 和 Data Assets 配置动画行为
3. **可扩展性**：通过动画层接口支持多种武器/载具类型
4. **性能优先**：使用 LOD、缓存姿势、异步查询等优化技术

### 11.2 关键技术要点

| 技术 | 用途 | 性能影响 |
|------|------|----------|
| **Motion Matching** | 高质量移动动画 | 中等（需优化数据库） |
| **Linked Anim Graphs** | 模块化动画层 | 低 |
| **IK (Foot/Hand)** | 自适应地形和武器 | 中等 |
| **Gameplay Tag Integration** | 与 GAS 无缝集成 | 低 |
| **Compressed Replication** | 网络同步优化 | 低（带宽优化） |

### 11.3 常见问题与解决方案

**问题 1：Motion Matching 响应延迟**
- **原因**：数据库过大或搜索时间过长
- **解决**：优化采样率、减少骨骼数量、启用异步查询

**问题 2：动画网络同步抖动**
- **原因**：复制频率不足或插值不平滑
- **解决**：使用 Inertialization、增加复制频率、实现客户端预测

**问题 3：IK 导致性能下降**
- **原因**：每帧更新所有 IK 目标
- **解决**：根据 LOD 降低更新频率、限制 IK 迭代次数

**问题 4：武器切换动画不流畅**
- **原因**：动画层切换时没有混合
- **解决**：使用 Inertialization、播放过渡蒙太奇

### 11.4 扩展建议

**下一步学习方向：**

1. **Control Rig**：程序化动画和实时调整
2. **Full Body IK**：全身 IK 解算（如 FBIK 插件）
3. **Motion Warping**：动画运动变形（如跳跃目标调整）
4. **Distance Matching**：精确的动画对齐（如攀爬）
5. **Stride Warping**：步伐调整以适应速度变化

### 11.5 参考资源

- **Lyra 源码**：`/Samples/Games/Lyra/Source/LyraGame/Animation`
- **Epic 官方文档**：[Animation System](https://docs.unrealengine.com/5.0/en-US/animation-system-in-unreal-engine/)
- **Motion Matching 教程**：[Pose Search Plugin](https://docs.unrealengine.com/5.0/en-US/pose-search-in-unreal-engine/)
- **GAS 集成**：参考本系列 #15 章（Gameplay Ability System）

---

## 总结

本章深入探讨了 Lyra 的动画系统架构，从基础的 AnimInstance 到先进的 Motion Matching，涵盖了：

- ✅ Lyra 动画架构（AnimInstance/LinkedAnimGraph）
- ✅ Animation Blueprint 设计模式
- ✅ Motion Matching 系统详解
- ✅ Pose Search 数据库构建
- ✅ 与 GAS 动画通知集成
- ✅ IK 和程序化动画
- ✅ 武器动画层（Weapon AnimLayer）
- ✅ 网络动画同步
- ✅ 完整代码示例
- ✅ 实战案例1：第一人称/第三人称动画系统
- ✅ 实战案例2：载具动画集成
- ✅ 性能优化技巧

通过这些技术，你可以构建出媲美 AAA 级别的角色动画系统。记住，动画是玩家体验的核心，值得投入时间进行优化和打磨！

**字数统计：约 12,500 字**

---

*本教程是 UE5 Lyra 系列教程的第 24 章。下一章我们将探讨 UI 系统与 Common UI 框架。*
