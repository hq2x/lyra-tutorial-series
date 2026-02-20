# 第17章：交互系统与提示 UI

## 17.1 Lyra 交互系统概述

### 17.1.1 交互系统的设计哲学

Lyra 的交互系统是一个高度模块化且可扩展的框架，它巧妙地结合了 Gameplay Ability System (GAS)、接口设计模式和 UI Extension 系统。这套系统的核心目标是：

1. **解耦性**：交互逻辑与具体对象实现分离
2. **可扩展性**：轻松添加新的交互类型
3. **网络友好**：内置网络同步支持
4. **UI 驱动**：自动化的提示界面管理
5. **性能优化**：高效的检测和过滤机制

### 17.1.2 系统架构全景

Lyra 交互系统由以下几个核心组件构成：

```
┌─────────────────────────────────────────────────────┐
│           玩家角色 (Pawn + ASC)                      │
│  ┌────────────────────────────────────────────┐    │
│  │  ULyraGameplayAbility_Interact             │    │
│  │  - 持续运行的交互检测能力                   │    │
│  └────────────────────────────────────────────┘    │
│           │                    │                     │
│           ▼                    ▼                     │
│  ┌─────────────────┐  ┌─────────────────────┐     │
│  │ 授予附近交互能力 │  │ 检测交互对象         │     │
│  │ GrantNearby     │  │ WaitForTargets      │     │
│  │ Interaction     │  │ (LineTrace)         │     │
│  └─────────────────┘  └─────────────────────┘     │
└─────────────────────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │  IInteractableTarget  │ (接口)
        │  - 可交互对象契约      │
        └───────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌──────────────┐      ┌──────────────┐
│ 可拾取物品    │      │ 可开启的门    │
│ Pickup Item  │      │ Openable Door│
└──────────────┘      └──────────────┘
        │                       │
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │  FInteractionOption   │
        │  - 交互选项数据        │
        │  - UI Widget Class    │
        │  - 交互能力引用        │
        └───────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │   Indicator System    │
        │   - 动态 UI 提示       │
        └───────────────────────┘
```

### 17.1.3 核心数据流

让我们跟踪一次完整的交互流程：

1. **检测阶段**：`AbilityTask_WaitForInteractableTargets_SingleLineTrace` 定期执行射线检测
2. **收集阶段**：找到实现 `IInteractableTarget` 接口的对象
3. **查询阶段**：调用 `GatherInteractionOptions()` 收集交互选项
4. **过滤阶段**：检查能力是否可激活
5. **UI 更新**：通过 Indicator System 显示提示 Widget
6. **触发阶段**：玩家按键触发 `TriggerInteraction()`
7. **执行阶段**：激活目标对象的交互能力

### 17.1.4 关键文件结构

```
LyraGame/Interaction/
├── IInteractableTarget.h          // 可交互对象接口
├── IInteractionInstigator.h       // 交互发起者接口
├── InteractionOption.h            // 交互选项数据结构
├── InteractionQuery.h             // 交互查询参数
├── InteractionStatics.h/cpp       // 工具函数库
├── LyraInteractionDurationMessage.h // 持续交互消息
├── Abilities/
│   ├── LyraGameplayAbility_Interact.h/cpp    // 主交互能力
│   └── GameplayAbilityTargetActor_Interact.h/cpp
└── Tasks/
    ├── AbilityTask_GrantNearbyInteraction.h/cpp          // 授予附近交互能力
    ├── AbilityTask_WaitForInteractableTargets.h/cpp      // 基类
    └── AbilityTask_WaitForInteractableTargets_SingleLineTrace.h/cpp
```

---

## 17.2 IInteractableTarget 接口详解

### 17.2.1 接口设计分析

`IInteractableTarget` 是 Lyra 交互系统的核心接口，它采用了 UE 的标准接口模式：

```cpp
// IInteractableTarget.h
#pragma once

#include "CoreMinimal.h"
#include "Abilities/GameplayAbility.h"
#include "InteractionOption.h"
#include "IInteractableTarget.generated.h"

struct FInteractionQuery;

/**
 * FInteractionOptionBuilder
 * 
 * 构建器模式，用于收集交互选项
 * 自动关联交互目标与选项
 */
class FInteractionOptionBuilder
{
public:
    FInteractionOptionBuilder(TScriptInterface<IInteractableTarget> InterfaceTargetScope, 
                              TArray<FInteractionOption>& InteractOptions)
        : Scope(InterfaceTargetScope)
        , Options(InteractOptions)
    {
    }

    void AddInteractionOption(const FInteractionOption& Option)
    {
        FInteractionOption& OptionEntry = Options.Add_GetRef(Option);
        OptionEntry.InteractableTarget = Scope;  // 自动设置目标引用
    }

private:
    TScriptInterface<IInteractableTarget> Scope;
    TArray<FInteractionOption>& Options;
};

/**
 * UInteractableTarget
 * 
 * UInterface 类（用于反射系统）
 * CannotImplementInterfaceInBlueprint：不允许蓝图直接实现，必须通过 C++
 */
UINTERFACE(MinimalAPI, meta = (CannotImplementInterfaceInBlueprint))
class UInteractableTarget : public UInterface
{
    GENERATED_BODY()
};

/**
 * IInteractableTarget
 * 
 * 实际的接口类，定义可交互对象的契约
 */
class IInteractableTarget
{
    GENERATED_BODY()

public:
    /**
     * 收集交互选项（核心方法）
     * 
     * @param InteractQuery 交互查询参数（谁在查询、什么条件）
     * @param OptionBuilder 选项构建器（用于添加可用的交互选项）
     */
    virtual void GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                          FInteractionOptionBuilder& OptionBuilder) = 0;

    /**
     * 自定义交互事件数据（可选）
     * 
     * 允许交互对象在触发时修改事件数据
     * 例如：按钮可能想要将门设为实际目标
     * 
     * @param InteractionEventTag 交互事件标签
     * @param InOutEventData 可修改的事件数据
     */
    virtual void CustomizeInteractionEventData(const FGameplayTag& InteractionEventTag, 
                                               FGameplayEventData& InOutEventData) 
    { 
        // 默认实现为空，子类按需重写
    }
};
```

### 17.2.2 交互查询结构

`FInteractionQuery` 封装了查询时的上下文信息：

```cpp
// InteractionQuery.h
#pragma once

#include "CoreMinimal.h"
#include "Abilities/GameplayAbility.h"
#include "InteractionQuery.generated.h"

/**
 * FInteractionQuery
 * 
 * 交互查询参数结构
 * 传递给 GatherInteractionOptions，提供查询上下文
 */
USTRUCT(BlueprintType)
struct FInteractionQuery
{
    GENERATED_BODY()

public:
    /** 发起交互的 Pawn/Actor */
    UPROPERTY(BlueprintReadWrite)
    TWeakObjectPtr<AActor> RequestingAvatar;

    /** 
     * 控制器（可能与 Avatar 的 Owner 不同）
     * 例如：AI 控制的 Pawn
     */
    UPROPERTY(BlueprintReadWrite)
    TWeakObjectPtr<AController> RequestingController;

    /** 
     * 可选的额外数据对象
     * 用于传递自定义的查询参数
     */
    UPROPERTY(BlueprintReadWrite)
    TWeakObjectPtr<UObject> OptionalObjectData;
};
```

### 17.2.3 交互选项结构

`FInteractionOption` 是交互系统的核心数据结构：

```cpp
// InteractionOption.h
#pragma once

#include "CoreMinimal.h"
#include "Abilities/GameplayAbility.h"
#include "InteractionOption.generated.h"

class IInteractableTarget;
class UUserWidget;

/**
 * FInteractionOption
 * 
 * 单个交互选项的完整描述
 * 包含能力、UI、目标等所有信息
 */
USTRUCT(BlueprintType)
struct FInteractionOption
{
    GENERATED_BODY()

public:
    // ======== 基础信息 ========
    
    /** 可交互目标的接口引用 */
    UPROPERTY(BlueprintReadWrite)
    TScriptInterface<IInteractableTarget> InteractableTarget;

    /** 主要文本（如 "拾取武器"） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FText Text;

    /** 副文本（如 "突击步枪 [30/120]"） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FText SubText;

    // ======== 能力激活方式（二选一） ========
    
    // 方式1：授予玩家一个交互能力
    /** 要授予给玩家的交互能力类 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TSubclassOf<UGameplayAbility> InteractionAbilityToGrant;

    // 方式2：激活目标对象自己的能力
    /** 目标对象的能力系统组件 */
    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<UAbilitySystemComponent> TargetAbilitySystem = nullptr;

    /** 目标对象上要激活的能力句柄 */
    UPROPERTY(BlueprintReadOnly)
    FGameplayAbilitySpecHandle TargetInteractionAbilityHandle;

    // ======== UI 配置 ========
    
    /** 交互提示 Widget 类 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSoftClassPtr<UUserWidget> InteractionWidgetClass;

    // ======== 比较运算符 ========
    
    FORCEINLINE bool operator==(const FInteractionOption& Other) const
    {
        return InteractableTarget == Other.InteractableTarget &&
            InteractionAbilityToGrant == Other.InteractionAbilityToGrant &&
            TargetAbilitySystem == Other.TargetAbilitySystem &&
            TargetInteractionAbilityHandle == Other.TargetInteractionAbilityHandle &&
            InteractionWidgetClass == Other.InteractionWidgetClass &&
            Text.IdenticalTo(Other.Text) &&
            SubText.IdenticalTo(Other.SubText);
    }

    FORCEINLINE bool operator!=(const FInteractionOption& Other) const
    {
        return !operator==(Other);
    }

    FORCEINLINE bool operator<(const FInteractionOption& Other) const
    {
        return InteractableTarget.GetInterface() < Other.InteractableTarget.GetInterface();
    }
};
```

### 17.2.4 实现接口的示例

让我们创建一个简单的可拾取物品类来展示接口实现：

```cpp
// PickupableItem.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Interaction/IInteractableTarget.h"
#include "PickupableItem.generated.h"

/**
 * APickupableItem
 * 
 * 可拾取物品示例
 * 展示如何实现 IInteractableTarget 接口
 */
UCLASS()
class YOURGAME_API APickupableItem : public AActor, public IInteractableTarget
{
    GENERATED_BODY()
    
public:    
    APickupableItem();

    // ======== IInteractableTarget 接口实现 ========
    
    /**
     * 收集交互选项
     * 这里我们返回一个"拾取"选项
     */
    virtual void GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                          FInteractionOptionBuilder& OptionBuilder) override;

    /**
     * 自定义事件数据
     * 可以在触发时添加物品信息
     */
    virtual void CustomizeInteractionEventData(const FGameplayTag& InteractionEventTag, 
                                               FGameplayEventData& InOutEventData) override;

protected:
    /** 物品名称 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Item")
    FText ItemName;

    /** 物品描述 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Item")
    FText ItemDescription;

    /** 拾取时授予的能力 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    TSubclassOf<UGameplayAbility> PickupAbility;

    /** 交互提示 Widget */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    TSoftClassPtr<UUserWidget> PickupWidgetClass;

    /** 物品网格体 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UStaticMeshComponent> MeshComponent;

    /** 碰撞体 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<class USphereComponent> InteractionVolume;
};
```

```cpp
// PickupableItem.cpp
#include "PickupableItem.h"
#include "Components/SphereComponent.h"
#include "Components/StaticMeshComponent.h"

APickupableItem::APickupableItem()
{
    PrimaryActorTick.bCanEverTick = false;
    bReplicates = true;  // 网络同步

    // 创建根组件
    InteractionVolume = CreateDefaultSubobject<USphereComponent>(TEXT("InteractionVolume"));
    RootComponent = InteractionVolume;
    InteractionVolume->SetSphereRadius(100.0f);
    InteractionVolume->SetCollisionProfileName(TEXT("OverlapAllDynamic"));

    // 创建网格体
    MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("MeshComponent"));
    MeshComponent->SetupAttachment(RootComponent);
    MeshComponent->SetCollisionEnabled(ECollisionEnabled::NoCollision);
}

void APickupableItem::GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                               FInteractionOptionBuilder& OptionBuilder)
{
    // 创建交互选项
    FInteractionOption Option;
    Option.Text = FText::Format(NSLOCTEXT("Interaction", "PickupFormat", "拾取 {0}"), 
                                ItemName);
    Option.SubText = ItemDescription;
    Option.InteractionAbilityToGrant = PickupAbility;
    Option.InteractionWidgetClass = PickupWidgetClass;

    // 添加到构建器
    OptionBuilder.AddInteractionOption(Option);
}

void APickupableItem::CustomizeInteractionEventData(const FGameplayTag& InteractionEventTag, 
                                                    FGameplayEventData& InOutEventData)
{
    // 将当前物品作为 OptionalObject 传递
    InOutEventData.OptionalObject = this;
    
    // 可以添加更多自定义数据
    // InOutEventData.EventMagnitude = ItemValue;
}
```

### 17.2.5 接口的高级用法

**多选项支持**：一个对象可以提供多个交互选项

```cpp
void AMultiOptionObject::GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                                  FInteractionOptionBuilder& OptionBuilder)
{
    // 选项1：拾取
    if (bCanPickup)
    {
        FInteractionOption PickupOption;
        PickupOption.Text = NSLOCTEXT("Interaction", "Pickup", "拾取");
        PickupOption.InteractionAbilityToGrant = PickupAbility;
        OptionBuilder.AddInteractionOption(PickupOption);
    }

    // 选项2：检查
    if (bCanInspect)
    {
        FInteractionOption InspectOption;
        InspectOption.Text = NSLOCTEXT("Interaction", "Inspect", "检查");
        InspectOption.InteractionAbilityToGrant = InspectAbility;
        OptionBuilder.AddInteractionOption(InspectOption);
    }

    // 选项3：销毁
    if (bCanDestroy)
    {
        FInteractionOption DestroyOption;
        DestroyOption.Text = NSLOCTEXT("Interaction", "Destroy", "销毁");
        DestroyOption.InteractionAbilityToGrant = DestroyAbility;
        OptionBuilder.AddInteractionOption(DestroyOption);
    }
}
```

**条件判断**：根据查询参数返回不同选项

```cpp
void AConditionalObject::GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                                  FInteractionOptionBuilder& OptionBuilder)
{
    // 获取请求者
    AActor* RequestingActor = InteractQuery.RequestingAvatar.Get();
    if (!RequestingActor)
    {
        return;
    }

    // 检查是否有钥匙
    UInventoryComponent* Inventory = RequestingActor->FindComponentByClass<UInventoryComponent>();
    bool bHasKey = Inventory && Inventory->HasItem(TEXT("KeyItem"));

    FInteractionOption Option;
    if (bHasKey)
    {
        Option.Text = NSLOCTEXT("Interaction", "Unlock", "解锁");
        Option.InteractionAbilityToGrant = UnlockAbility;
    }
    else
    {
        Option.Text = NSLOCTEXT("Interaction", "Locked", "已锁定（需要钥匙）");
        // 不提供能力，无法交互
    }

    OptionBuilder.AddInteractionOption(Option);
}
```

---

## 17.3 交互检测与发现

### 17.3.1 检测任务基类分析

`UAbilityTask_WaitForInteractableTargets` 是所有交互检测任务的基类：

```cpp
// AbilityTask_WaitForInteractableTargets.h
#pragma once

#include "Abilities/Tasks/AbilityTask.h"
#include "Engine/CollisionProfile.h"
#include "Interaction/InteractionOption.h"
#include "AbilityTask_WaitForInteractableTargets.generated.h"

class AActor;
class IInteractableTarget;
class UObject;
class UWorld;
struct FCollisionQueryParams;
struct FHitResult;
struct FInteractionQuery;
template <typename InterfaceType> class TScriptInterface;

/**
 * 交互对象变化事件
 * 当检测到的交互对象列表发生变化时触发
 */
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FInteractableObjectsChangedEvent, 
                                            const TArray<FInteractionOption>&, InteractableOptions);

/**
 * UAbilityTask_WaitForInteractableTargets
 * 
 * 交互检测任务基类
 * 提供通用的检测逻辑和射线追踪功能
 */
UCLASS(Abstract)
class UAbilityTask_WaitForInteractableTargets : public UAbilityTask
{
    GENERATED_UCLASS_BODY()

public:
    /** 交互对象变化时广播 */
    UPROPERTY(BlueprintAssignable)
    FInteractableObjectsChangedEvent InteractableObjectsChanged;

protected:
    // ======== 射线追踪工具方法 ========
    
    /**
     * 执行射线追踪
     * 
     * @param OutHitResult 输出的命中结果
     * @param World 世界对象
     * @param Start 起始点
     * @param End 结束点
     * @param ProfileName 碰撞配置名称
     * @param Params 碰撞查询参数
     */
    static void LineTrace(FHitResult& OutHitResult, const UWorld* World, 
                         const FVector& Start, const FVector& End, 
                         FName ProfileName, const FCollisionQueryParams Params);

    /**
     * 使用玩家控制器的视角进行瞄准
     * 
     * 考虑相机位置与角色位置的偏移
     * 
     * @param InSourceActor 源 Actor
     * @param Params 碰撞查询参数
     * @param TraceStart 追踪起始点
     * @param MaxRange 最大范围
     * @param OutTraceEnd 输出的追踪终点
     * @param bIgnorePitch 是否忽略俯仰角
     */
    void AimWithPlayerController(const AActor* InSourceActor, 
                                 FCollisionQueryParams Params, 
                                 const FVector& TraceStart, 
                                 float MaxRange, 
                                 FVector& OutTraceEnd, 
                                 bool bIgnorePitch = false) const;

    /**
     * 将相机射线裁剪到能力范围
     * 
     * 处理相机位置可能远离角色的情况
     * 
     * @param CameraLocation 相机位置
     * @param CameraDirection 相机方向
     * @param AbilityCenter 能力中心点
     * @param AbilityRange 能力范围
     * @param ClippedPosition 输出的裁剪位置
     * @return 是否成功裁剪
     */
    static bool ClipCameraRayToAbilityRange(FVector CameraLocation, 
                                           FVector CameraDirection, 
                                           FVector AbilityCenter, 
                                           float AbilityRange, 
                                           FVector& ClippedPosition);

    /**
     * 更新交互选项
     * 
     * 核心方法：收集、过滤、比较交互选项，并在变化时触发事件
     * 
     * @param InteractQuery 交互查询参数
     * @param InteractableTargets 可交互目标列表
     */
    void UpdateInteractableOptions(const FInteractionQuery& InteractQuery, 
                                   const TArray<TScriptInterface<IInteractableTarget>>& InteractableTargets);

    /** 碰撞配置名称 */
    FCollisionProfileName TraceProfile;

    /** 追踪是否影响瞄准俯仰角 */
    bool bTraceAffectsAimPitch = true;

    /** 当前的交互选项列表 */
    TArray<FInteractionOption> CurrentOptions;
};
```

让我们深入分析关键方法的实现：

```cpp
// AbilityTask_WaitForInteractableTargets.cpp

void UAbilityTask_WaitForInteractableTargets::LineTrace(FHitResult& OutHitResult, 
                                                        const UWorld* World, 
                                                        const FVector& Start, 
                                                        const FVector& End, 
                                                        FName ProfileName, 
                                                        const FCollisionQueryParams Params)
{
    check(World);

    OutHitResult = FHitResult();
    TArray<FHitResult> HitResults;
    
    // 执行多次命中的射线追踪
    World->LineTraceMultiByProfile(HitResults, Start, End, ProfileName, Params);

    OutHitResult.TraceStart = Start;
    OutHitResult.TraceEnd = End;

    // 取第一个命中结果
    if (HitResults.Num() > 0)
    {
        OutHitResult = HitResults[0];
    }
}

void UAbilityTask_WaitForInteractableTargets::AimWithPlayerController(
    const AActor* InSourceActor, 
    FCollisionQueryParams Params, 
    const FVector& TraceStart, 
    float MaxRange, 
    FVector& OutTraceEnd, 
    bool bIgnorePitch) const
{
    if (!Ability) // 仅在服务器和启动客户端上运行
    {
        return;
    }

    // 获取玩家控制器
    APlayerController* PC = Ability->GetCurrentActorInfo()->PlayerController.Get();
    check(PC);

    // 获取相机视角
    FVector ViewStart;
    FRotator ViewRot;
    PC->GetPlayerViewPoint(ViewStart, ViewRot);

    const FVector ViewDir = ViewRot.Vector();
    FVector ViewEnd = ViewStart + (ViewDir * MaxRange);

    // 将相机射线裁剪到能力范围内
    ClipCameraRayToAbilityRange(ViewStart, ViewDir, TraceStart, MaxRange, ViewEnd);

    // 从相机位置向前追踪
    FHitResult HitResult;
    LineTrace(HitResult, InSourceActor->GetWorld(), ViewStart, ViewEnd, TraceProfile.Name, Params);

    // 检查命中点是否在范围内
    const bool bUseTraceResult = HitResult.bBlockingHit && 
        (FVector::DistSquared(TraceStart, HitResult.Location) <= (MaxRange * MaxRange));

    const FVector AdjustedEnd = (bUseTraceResult) ? HitResult.Location : ViewEnd;

    // 计算调整后的瞄准方向
    FVector AdjustedAimDir = (AdjustedEnd - TraceStart).GetSafeNormal();
    if (AdjustedAimDir.IsZero())
    {
        AdjustedAimDir = ViewDir;
    }

    // 可选：忽略俯仰角，使用原始俯仰
    if (!bTraceAffectsAimPitch && bUseTraceResult)
    {
        FVector OriginalAimDir = (ViewEnd - TraceStart).GetSafeNormal();

        if (!OriginalAimDir.IsZero())
        {
            const FRotator OriginalAimRot = OriginalAimDir.Rotation();
            FRotator AdjustedAimRot = AdjustedAimDir.Rotation();
            AdjustedAimRot.Pitch = OriginalAimRot.Pitch;  // 保留原始俯仰角
            AdjustedAimDir = AdjustedAimRot.Vector();
        }
    }

    OutTraceEnd = TraceStart + (AdjustedAimDir * MaxRange);
}

bool UAbilityTask_WaitForInteractableTargets::ClipCameraRayToAbilityRange(
    FVector CameraLocation, 
    FVector CameraDirection, 
    FVector AbilityCenter, 
    float AbilityRange, 
    FVector& ClippedPosition)
{
    FVector CameraToCenter = AbilityCenter - CameraLocation;
    float DotToCenter = FVector::DotProduct(CameraToCenter, CameraDirection);
    
    // 如果相机朝向中心
    if (DotToCenter >= 0)
    {
        float DistanceSquared = CameraToCenter.SizeSquared() - (DotToCenter * DotToCenter);
        float RadiusSquared = (AbilityRange * AbilityRange);
        
        // 如果射线与范围球相交
        if (DistanceSquared <= RadiusSquared)
        {
            // 计算交点
            float DistanceFromCamera = FMath::Sqrt(RadiusSquared - DistanceSquared);
            float DistanceAlongRay = DotToCenter + DistanceFromCamera;
            ClippedPosition = CameraLocation + (DistanceAlongRay * CameraDirection);
            return true;
        }
    }
    return false;
}

void UAbilityTask_WaitForInteractableTargets::UpdateInteractableOptions(
    const FInteractionQuery& InteractQuery, 
    const TArray<TScriptInterface<IInteractableTarget>>& InteractableTargets)
{
    TArray<FInteractionOption> NewOptions;

    // ======== 步骤1：收集所有交互选项 ========
    for (const TScriptInterface<IInteractableTarget>& InteractiveTarget : InteractableTargets)
    {
        TArray<FInteractionOption> TempOptions;
        FInteractionOptionBuilder InteractionBuilder(InteractiveTarget, TempOptions);
        InteractiveTarget->GatherInteractionOptions(InteractQuery, InteractionBuilder);

        // ======== 步骤2：验证和填充能力信息 ========
        for (FInteractionOption& Option : TempOptions)
        {
            FGameplayAbilitySpec* InteractionAbilitySpec = nullptr;

            // 情况1：目标对象有自己的能力系统
            if (Option.TargetAbilitySystem && Option.TargetInteractionAbilityHandle.IsValid())
            {
                InteractionAbilitySpec = Option.TargetAbilitySystem->FindAbilitySpecFromHandle(
                    Option.TargetInteractionAbilityHandle);
            }
            // 情况2：需要授予玩家交互能力
            else if (Option.InteractionAbilityToGrant)
            {
                InteractionAbilitySpec = AbilitySystemComponent->FindAbilitySpecFromClass(
                    Option.InteractionAbilityToGrant);

                if (InteractionAbilitySpec)
                {
                    // 更新选项，指向玩家的能力系统
                    Option.TargetAbilitySystem = AbilitySystemComponent.Get();
                    Option.TargetInteractionAbilityHandle = InteractionAbilitySpec->Handle;
                }
            }

            // ======== 步骤3：过滤不可激活的能力 ========
            if (InteractionAbilitySpec)
            {
                if (InteractionAbilitySpec->Ability->CanActivateAbility(
                    InteractionAbilitySpec->Handle, 
                    AbilitySystemComponent->AbilityActorInfo.Get()))
                {
                    NewOptions.Add(Option);
                }
            }
        }
    }

    // ======== 步骤4：检测变化 ========
    bool bOptionsChanged = false;
    if (NewOptions.Num() == CurrentOptions.Num())
    {
        NewOptions.Sort();  // 确保顺序一致

        for (int OptionIndex = 0; OptionIndex < NewOptions.Num(); OptionIndex++)
        {
            if (NewOptions[OptionIndex] != CurrentOptions[OptionIndex])
            {
                bOptionsChanged = true;
                break;
            }
        }
    }
    else
    {
        bOptionsChanged = true;
    }

    // ======== 步骤5：触发变化事件 ========
    if (bOptionsChanged)
    {
        CurrentOptions = NewOptions;
        InteractableObjectsChanged.Broadcast(CurrentOptions);
    }
}
```

### 17.3.2 单线射线检测实现

`UAbilityTask_WaitForInteractableTargets_SingleLineTrace` 是最常用的检测方式：

```cpp
// AbilityTask_WaitForInteractableTargets_SingleLineTrace.h
#pragma once

#include "Interaction/InteractionQuery.h"
#include "Interaction/Tasks/AbilityTask_WaitForInteractableTargets.h"
#include "AbilityTask_WaitForInteractableTargets_SingleLineTrace.generated.h"

struct FCollisionProfileName;
class UGameplayAbility;
class UObject;
struct FFrame;

/**
 * UAbilityTask_WaitForInteractableTargets_SingleLineTrace
 * 
 * 使用单线射线追踪检测交互对象
 * 适用于第一人称/第三人称的准星瞄准交互
 */
UCLASS()
class UAbilityTask_WaitForInteractableTargets_SingleLineTrace : public UAbilityTask_WaitForInteractableTargets
{
    GENERATED_UCLASS_BODY()

public:
    virtual void Activate() override;

    /**
     * 创建单线追踪任务
     * 
     * @param OwningAbility 所属能力
     * @param InteractionQuery 交互查询参数
     * @param TraceProfile 碰撞配置
     * @param StartLocation 起始位置信息
     * @param InteractionScanRange 扫描范围
     * @param InteractionScanRate 扫描频率（秒）
     * @param bShowDebug 是否显示调试信息
     */
    UFUNCTION(BlueprintCallable, Category="Ability|Tasks", 
              meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", 
                      BlueprintInternalUseOnly = "TRUE"))
    static UAbilityTask_WaitForInteractableTargets_SingleLineTrace* WaitForInteractableTargets_SingleLineTrace(
        UGameplayAbility* OwningAbility, 
        FInteractionQuery InteractionQuery, 
        FCollisionProfileName TraceProfile, 
        FGameplayAbilityTargetingLocationInfo StartLocation, 
        float InteractionScanRange = 100, 
        float InteractionScanRate = 0.100, 
        bool bShowDebug = false);

private:
    virtual void OnDestroy(bool AbilityEnded) override;

    /** 执行追踪检测 */
    void PerformTrace();

    /** 交互查询参数 */
    UPROPERTY()
    FInteractionQuery InteractionQuery;

    /** 起始位置信息 */
    UPROPERTY()
    FGameplayAbilityTargetingLocationInfo StartLocation;

    /** 交互扫描范围 */
    float InteractionScanRange = 100;

    /** 交互扫描频率 */
    float InteractionScanRate = 0.100;

    /** 是否显示调试信息 */
    bool bShowDebug = false;

    /** 定时器句柄 */
    FTimerHandle TimerHandle;
};
```

```cpp
// AbilityTask_WaitForInteractableTargets_SingleLineTrace.cpp
#include "AbilityTask_WaitForInteractableTargets_SingleLineTrace.h"
#include "Interaction/InteractionStatics.h"
#include "DrawDebugHelpers.h"
#include "Engine/World.h"
#include "TimerManager.h"

UAbilityTask_WaitForInteractableTargets_SingleLineTrace::UAbilityTask_WaitForInteractableTargets_SingleLineTrace(
    const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
}

UAbilityTask_WaitForInteractableTargets_SingleLineTrace* 
UAbilityTask_WaitForInteractableTargets_SingleLineTrace::WaitForInteractableTargets_SingleLineTrace(
    UGameplayAbility* OwningAbility, 
    FInteractionQuery InteractionQuery, 
    FCollisionProfileName TraceProfile, 
    FGameplayAbilityTargetingLocationInfo StartLocation, 
    float InteractionScanRange, 
    float InteractionScanRate, 
    bool bShowDebug)
{
    UAbilityTask_WaitForInteractableTargets_SingleLineTrace* MyObj = 
        NewAbilityTask<UAbilityTask_WaitForInteractableTargets_SingleLineTrace>(OwningAbility);
    
    MyObj->InteractionScanRange = InteractionScanRange;
    MyObj->InteractionScanRate = InteractionScanRate;
    MyObj->StartLocation = StartLocation;
    MyObj->InteractionQuery = InteractionQuery;
    MyObj->TraceProfile = TraceProfile;
    MyObj->bShowDebug = bShowDebug;

    return MyObj;
}

void UAbilityTask_WaitForInteractableTargets_SingleLineTrace::Activate()
{
    SetWaitingOnAvatar();

    // 启动定时器，定期执行检测
    UWorld* World = GetWorld();
    World->GetTimerManager().SetTimer(TimerHandle, this, 
                                     &ThisClass::PerformTrace, 
                                     InteractionScanRate, true);
}

void UAbilityTask_WaitForInteractableTargets_SingleLineTrace::OnDestroy(bool AbilityEnded)
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(TimerHandle);
    }

    Super::OnDestroy(AbilityEnded);
}

void UAbilityTask_WaitForInteractableTargets_SingleLineTrace::PerformTrace()
{
    AActor* AvatarActor = Ability->GetCurrentActorInfo()->AvatarActor.Get();
    if (!AvatarActor)
    {
        return;
    }

    UWorld* World = GetWorld();

    // 设置忽略列表
    TArray<AActor*> ActorsToIgnore;
    ActorsToIgnore.Add(AvatarActor);

    const bool bTraceComplex = false;
    FCollisionQueryParams Params(SCENE_QUERY_STAT(UAbilityTask_WaitForInteractableTargets_SingleLineTrace), 
                                bTraceComplex);
    Params.AddIgnoredActors(ActorsToIgnore);

    // 计算追踪起点和终点
    FVector TraceStart = StartLocation.GetTargetingTransform().GetLocation();
    FVector TraceEnd;
    AimWithPlayerController(AvatarActor, Params, TraceStart, InteractionScanRange, OUT TraceEnd);

    // 执行射线追踪
    FHitResult OutHitResult;
    LineTrace(OutHitResult, World, TraceStart, TraceEnd, TraceProfile.Name, Params);

    // 从命中结果中提取可交互目标
    TArray<TScriptInterface<IInteractableTarget>> InteractableTargets;
    UInteractionStatics::AppendInteractableTargetsFromHitResult(OutHitResult, InteractableTargets);

    // 更新交互选项
    UpdateInteractableOptions(InteractionQuery, InteractableTargets);

#if ENABLE_DRAW_DEBUG
    if (bShowDebug)
    {
        FColor DebugColor = OutHitResult.bBlockingHit ? FColor::Red : FColor::Green;
        if (OutHitResult.bBlockingHit)
        {
            DrawDebugLine(World, TraceStart, OutHitResult.Location, DebugColor, false, InteractionScanRate);
            DrawDebugSphere(World, OutHitResult.Location, 5, 16, DebugColor, false, InteractionScanRate);
        }
        else
        {
            DrawDebugLine(World, TraceStart, TraceEnd, DebugColor, false, InteractionScanRate);
        }
    }
#endif // ENABLE_DRAW_DEBUG
}
```

### 17.3.3 范围检测实现

`UAbilityTask_GrantNearbyInteraction` 用于在角色周围查找交互对象：

```cpp
// AbilityTask_GrantNearbyInteraction.h
#pragma once

#include "Abilities/Tasks/AbilityTask.h"
#include "AbilityTask_GrantNearbyInteraction.generated.h"

class UGameplayAbility;
class UObject;
struct FFrame;
struct FGameplayAbilitySpecHandle;
struct FObjectKey;

/**
 * UAbilityTask_GrantNearbyInteraction
 * 
 * 在角色周围范围内查找交互对象
 * 自动授予玩家相应的交互能力
 */
UCLASS()
class UAbilityTask_GrantNearbyInteraction : public UAbilityTask
{
    GENERATED_UCLASS_BODY()

public:
    virtual void Activate() override;

    /**
     * 为附近的交互对象授予能力
     * 
     * @param OwningAbility 所属能力
     * @param InteractionScanRange 扫描范围
     * @param InteractionScanRate 扫描频率
     */
    UFUNCTION(BlueprintCallable, Category="Ability|Tasks", 
              meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", 
                      BlueprintInternalUseOnly = "TRUE"))
    static UAbilityTask_GrantNearbyInteraction* GrantAbilitiesForNearbyInteractors(
        UGameplayAbility* OwningAbility, 
        float InteractionScanRange, 
        float InteractionScanRate);

private:
    virtual void OnDestroy(bool AbilityEnded) override;

    /** 查询交互对象 */
    void QueryInteractables();

    /** 交互扫描范围 */
    float InteractionScanRange = 100;

    /** 交互扫描频率 */
    float InteractionScanRate = 0.100;

    /** 定时器句柄 */
    FTimerHandle QueryTimerHandle;

    /** 交互能力缓存（避免重复授予） */
    TMap<FObjectKey, FGameplayAbilitySpecHandle> InteractionAbilityCache;
};
```

```cpp
// AbilityTask_GrantNearbyInteraction.cpp
#include "AbilityTask_GrantNearbyInteraction.h"
#include "AbilitySystemComponent.h"
#include "Engine/OverlapResult.h"
#include "Engine/World.h"
#include "GameFramework/Controller.h"
#include "Interaction/IInteractableTarget.h"
#include "Interaction/InteractionOption.h"
#include "Interaction/InteractionQuery.h"
#include "Interaction/InteractionStatics.h"
#include "Physics/LyraCollisionChannels.h"
#include "TimerManager.h"

UAbilityTask_GrantNearbyInteraction::UAbilityTask_GrantNearbyInteraction(
    const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
}

UAbilityTask_GrantNearbyInteraction* 
UAbilityTask_GrantNearbyInteraction::GrantAbilitiesForNearbyInteractors(
    UGameplayAbility* OwningAbility, 
    float InteractionScanRange, 
    float InteractionScanRate)
{
    UAbilityTask_GrantNearbyInteraction* MyObj = 
        NewAbilityTask<UAbilityTask_GrantNearbyInteraction>(OwningAbility);
    MyObj->InteractionScanRange = InteractionScanRange;
    MyObj->InteractionScanRate = InteractionScanRate;
    return MyObj;
}

void UAbilityTask_GrantNearbyInteraction::Activate()
{
    SetWaitingOnAvatar();

    // 启动定时器，定期查询
    UWorld* World = GetWorld();
    World->GetTimerManager().SetTimer(QueryTimerHandle, this, 
                                     &ThisClass::QueryInteractables, 
                                     InteractionScanRate, true);
}

void UAbilityTask_GrantNearbyInteraction::OnDestroy(bool AbilityEnded)
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(QueryTimerHandle);
    }

    Super::OnDestroy(AbilityEnded);
}

void UAbilityTask_GrantNearbyInteraction::QueryInteractables()
{
    UWorld* World = GetWorld();
    AActor* ActorOwner = GetAvatarActor();
    
    if (World && ActorOwner)
    {
        // 设置碰撞查询参数
        FCollisionQueryParams Params(SCENE_QUERY_STAT(UAbilityTask_GrantNearbyInteraction), false);

        // 执行球形重叠检测
        TArray<FOverlapResult> OverlapResults;
        World->OverlapMultiByChannel(
            OUT OverlapResults, 
            ActorOwner->GetActorLocation(), 
            FQuat::Identity, 
            Lyra_TraceChannel_Interaction,  // Lyra 自定义的交互通道
            FCollisionShape::MakeSphere(InteractionScanRange), 
            Params);

        if (OverlapResults.Num() > 0)
        {
            // 从重叠结果中提取交互目标
            TArray<TScriptInterface<IInteractableTarget>> InteractableTargets;
            UInteractionStatics::AppendInteractableTargetsFromOverlapResults(OverlapResults, 
                                                                            OUT InteractableTargets);
            
            // 构建交互查询
            FInteractionQuery InteractionQuery;
            InteractionQuery.RequestingAvatar = ActorOwner;
            InteractionQuery.RequestingController = Cast<AController>(ActorOwner->GetOwner());

            // 收集所有交互选项
            TArray<FInteractionOption> Options;
            for (TScriptInterface<IInteractableTarget>& InteractiveTarget : InteractableTargets)
            {
                FInteractionOptionBuilder InteractionBuilder(InteractiveTarget, Options);
                InteractiveTarget->GatherInteractionOptions(InteractionQuery, InteractionBuilder);
            }

            // 授予交互能力
            for (FInteractionOption& Option : Options)
            {
                if (Option.InteractionAbilityToGrant)
                {
                    // 检查是否已经授予过
                    FObjectKey ObjectKey(Option.InteractionAbilityToGrant);
                    if (!InteractionAbilityCache.Find(ObjectKey))
                    {
                        // 授予能力
                        FGameplayAbilitySpec Spec(Option.InteractionAbilityToGrant, 1, INDEX_NONE, this);
                        FGameplayAbilitySpecHandle Handle = AbilitySystemComponent->GiveAbility(Spec);
                        InteractionAbilityCache.Add(ObjectKey, Handle);
                    }
                }
            }
        }
    }
}
```

### 17.3.4 交互工具函数库

```cpp
// InteractionStatics.h
#pragma once

#include "Kismet/BlueprintFunctionLibrary.h"
#include "InteractionStatics.generated.h"

template <typename InterfaceType> class TScriptInterface;
class AActor;
class IInteractableTarget;
class UObject;
struct FFrame;
struct FHitResult;
struct FOverlapResult;

/**
 * UInteractionStatics
 * 
 * 交互系统工具函数库
 * 提供常用的查询和转换函数
 */
UCLASS()
class UInteractionStatics : public UBlueprintFunctionLibrary
{
    GENERATED_BODY()

public:
    UInteractionStatics();

    /**
     * 从交互目标获取 Actor
     */
    UFUNCTION(BlueprintCallable)
    static AActor* GetActorFromInteractableTarget(TScriptInterface<IInteractableTarget> InteractableTarget);

    /**
     * 从 Actor 获取所有交互目标接口
     */
    UFUNCTION(BlueprintCallable)
    static void GetInteractableTargetsFromActor(AActor* Actor, 
                                               TArray<TScriptInterface<IInteractableTarget>>& OutInteractableTargets);

    /**
     * 从重叠结果中提取交互目标
     */
    static void AppendInteractableTargetsFromOverlapResults(
        const TArray<FOverlapResult>& OverlapResults, 
        TArray<TScriptInterface<IInteractableTarget>>& OutInteractableTargets);
    
    /**
     * 从命中结果中提取交互目标
     */
    static void AppendInteractableTargetsFromHitResult(
        const FHitResult& HitResult, 
        TArray<TScriptInterface<IInteractableTarget>>& OutInteractableTargets);
};
```

```cpp
// InteractionStatics.cpp
#include "InteractionStatics.h"
#include "Interaction/IInteractableTarget.h"
#include "Engine/OverlapResult.h"
#include "Engine/HitResult.h"

UInteractionStatics::UInteractionStatics()
{
}

AActor* UInteractionStatics::GetActorFromInteractableTarget(
    TScriptInterface<IInteractableTarget> InteractableTarget)
{
    if (UObject* Object = InteractableTarget.GetObject())
    {
        if (AActor* Actor = Cast<AActor>(Object))
        {
            return Actor;
        }
        else if (UActorComponent* Component = Cast<UActorComponent>(Object))
        {
            return Component->GetOwner();
        }
    }

    return nullptr;
}

void UInteractionStatics::GetInteractableTargetsFromActor(
    AActor* Actor, 
    TArray<TScriptInterface<IInteractableTarget>>& OutInteractableTargets)
{
    if (!Actor)
    {
        return;
    }

    // 检查 Actor 本身
    if (Actor->Implements<UInteractableTarget>())
    {
        TScriptInterface<IInteractableTarget> InteractableActor;
        InteractableActor.SetObject(Actor);
        InteractableActor.SetInterface(Cast<IInteractableTarget>(Actor));
        OutInteractableTargets.AddUnique(InteractableActor);
    }

    // 检查所有组件
    TArray<UActorComponent*> Components = Actor->GetComponentsByInterface(UInteractableTarget::StaticClass());
    for (UActorComponent* Component : Components)
    {
        TScriptInterface<IInteractableTarget> InteractableComponent;
        InteractableComponent.SetObject(Component);
        InteractableComponent.SetInterface(Cast<IInteractableTarget>(Component));
        OutInteractableTargets.AddUnique(InteractableComponent);
    }
}

void UInteractionStatics::AppendInteractableTargetsFromOverlapResults(
    const TArray<FOverlapResult>& OverlapResults, 
    TArray<TScriptInterface<IInteractableTarget>>& OutInteractableTargets)
{
    for (const FOverlapResult& Overlap : OverlapResults)
    {
        if (AActor* OverlappedActor = Overlap.GetActor())
        {
            GetInteractableTargetsFromActor(OverlappedActor, OutInteractableTargets);
        }
    }
}

void UInteractionStatics::AppendInteractableTargetsFromHitResult(
    const FHitResult& HitResult, 
    TArray<TScriptInterface<IInteractableTarget>>& OutInteractableTargets)
{
    if (AActor* HitActor = HitResult.GetActor())
    {
        GetInteractableTargetsFromActor(HitActor, OutInteractableTargets);
    }
}
```

### 17.3.5 自定义检测方式示例

**示例1：多线扇形检测**

```cpp
// AbilityTask_WaitForInteractableTargets_Cone.h
#pragma once

#include "Interaction/Tasks/AbilityTask_WaitForInteractableTargets.h"
#include "AbilityTask_WaitForInteractableTargets_Cone.generated.h"

/**
 * 扇形范围检测
 * 适用于近战交互或范围交互
 */
UCLASS()
class UAbilityTask_WaitForInteractableTargets_Cone : public UAbilityTask_WaitForInteractableTargets
{
    GENERATED_UCLASS_BODY()

public:
    virtual void Activate() override;

    UFUNCTION(BlueprintCallable, Category="Ability|Tasks", 
              meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", 
                      BlueprintInternalUseOnly = "TRUE"))
    static UAbilityTask_WaitForInteractableTargets_Cone* WaitForInteractableTargets_Cone(
        UGameplayAbility* OwningAbility,
        FInteractionQuery InteractionQuery,
        float InteractionRange,
        float ConeHalfAngleDegrees,
        int32 NumTraces,
        float InteractionScanRate = 0.1f);

private:
    virtual void OnDestroy(bool AbilityEnded) override;
    void PerformConeTrace();

    FInteractionQuery InteractionQuery;
    float InteractionRange;
    float ConeHalfAngleDegrees;
    int32 NumTraces;
    float InteractionScanRate;
    FTimerHandle TimerHandle;
};
```

```cpp
// AbilityTask_WaitForInteractableTargets_Cone.cpp
#include "AbilityTask_WaitForInteractableTargets_Cone.h"
#include "Interaction/InteractionStatics.h"
#include "DrawDebugHelpers.h"

UAbilityTask_WaitForInteractableTargets_Cone::UAbilityTask_WaitForInteractableTargets_Cone(
    const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
}

UAbilityTask_WaitForInteractableTargets_Cone* 
UAbilityTask_WaitForInteractableTargets_Cone::WaitForInteractableTargets_Cone(
    UGameplayAbility* OwningAbility,
    FInteractionQuery InteractionQuery,
    float InteractionRange,
    float ConeHalfAngleDegrees,
    int32 NumTraces,
    float InteractionScanRate)
{
    UAbilityTask_WaitForInteractableTargets_Cone* MyObj = 
        NewAbilityTask<UAbilityTask_WaitForInteractableTargets_Cone>(OwningAbility);
    
    MyObj->InteractionQuery = InteractionQuery;
    MyObj->InteractionRange = InteractionRange;
    MyObj->ConeHalfAngleDegrees = ConeHalfAngleDegrees;
    MyObj->NumTraces = NumTraces;
    MyObj->InteractionScanRate = InteractionScanRate;

    return MyObj;
}

void UAbilityTask_WaitForInteractableTargets_Cone::Activate()
{
    SetWaitingOnAvatar();

    UWorld* World = GetWorld();
    World->GetTimerManager().SetTimer(TimerHandle, this, 
                                     &ThisClass::PerformConeTrace, 
                                     InteractionScanRate, true);
}

void UAbilityTask_WaitForInteractableTargets_Cone::OnDestroy(bool AbilityEnded)
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(TimerHandle);
    }

    Super::OnDestroy(AbilityEnded);
}

void UAbilityTask_WaitForInteractableTargets_Cone::PerformConeTrace()
{
    AActor* AvatarActor = Ability->GetCurrentActorInfo()->AvatarActor.Get();
    if (!AvatarActor)
    {
        return;
    }

    UWorld* World = GetWorld();
    
    // 获取起始位置和前向向量
    FVector StartLocation = AvatarActor->GetActorLocation();
    FVector ForwardVector = AvatarActor->GetActorForwardVector();
    
    TArray<AActor*> ActorsToIgnore;
    ActorsToIgnore.Add(AvatarActor);
    
    FCollisionQueryParams Params(SCENE_QUERY_STAT(ConeTrace), false);
    Params.AddIgnoredActors(ActorsToIgnore);

    TSet<AActor*> HitActors;
    
    // 执行多条射线构成扇形
    for (int32 i = 0; i < NumTraces; ++i)
    {
        // 计算扇形角度
        float Angle = FMath::Lerp(-ConeHalfAngleDegrees, ConeHalfAngleDegrees, 
                                  static_cast<float>(i) / FMath::Max(1, NumTraces - 1));
        
        // 绕 Z 轴旋转
        FVector TraceDirection = ForwardVector.RotateAngleAxis(Angle, FVector::UpVector);
        FVector TraceEnd = StartLocation + TraceDirection * InteractionRange;
        
        FHitResult HitResult;
        LineTrace(HitResult, World, StartLocation, TraceEnd, TraceProfile.Name, Params);
        
        if (HitResult.bBlockingHit && HitResult.GetActor())
        {
            HitActors.Add(HitResult.GetActor());
        }

#if ENABLE_DRAW_DEBUG
        DrawDebugLine(World, StartLocation, TraceEnd, 
                     HitResult.bBlockingHit ? FColor::Red : FColor::Green, 
                     false, InteractionScanRate);
#endif
    }

    // 收集所有交互目标
    TArray<TScriptInterface<IInteractableTarget>> InteractableTargets;
    for (AActor* HitActor : HitActors)
    {
        UInteractionStatics::GetInteractableTargetsFromActor(HitActor, InteractableTargets);
    }

    UpdateInteractableOptions(InteractionQuery, InteractableTargets);
}
```

---

## 17.4 交互提示 UI 实现

### 17.4.1 Indicator System 集成

Lyra 使用 Indicator System 来管理交互提示 UI。让我们先了解这个系统的架构：

```cpp
// UIndicatorDescriptor：UI 指示器的描述符
// ULyraIndicatorManagerComponent：管理所有指示器的组件
```

`ULyraGameplayAbility_Interact` 中的 UI 更新逻辑：

```cpp
void ULyraGameplayAbility_Interact::UpdateInteractions(const TArray<FInteractionOption>& InteractiveOptions)
{
    if (ALyraPlayerController* PC = GetLyraPlayerControllerFromActorInfo())
    {
        if (ULyraIndicatorManagerComponent* IndicatorManager = ULyraIndicatorManagerComponent::GetComponent(PC))
        {
            // ======== 步骤1：清除旧的指示器 ========
            for (UIndicatorDescriptor* Indicator : Indicators)
            {
                IndicatorManager->RemoveIndicator(Indicator);
            }
            Indicators.Reset();

            // ======== 步骤2：为每个交互选项创建新指示器 ========
            for (const FInteractionOption& InteractionOption : InteractiveOptions)
            {
                AActor* InteractableTargetActor = 
                    UInteractionStatics::GetActorFromInteractableTarget(InteractionOption.InteractableTarget);

                // 确定 Widget 类（使用选项指定的，或默认的）
                TSoftClassPtr<UUserWidget> InteractionWidgetClass = 
                    InteractionOption.InteractionWidgetClass.IsNull() ? 
                    DefaultInteractionWidgetClass : 
                    InteractionOption.InteractionWidgetClass;

                // 创建指示器描述符
                UIndicatorDescriptor* Indicator = NewObject<UIndicatorDescriptor>();
                Indicator->SetDataObject(InteractableTargetActor);  // 绑定数据对象
                Indicator->SetSceneComponent(InteractableTargetActor->GetRootComponent());  // 绑定位置
                Indicator->SetIndicatorClass(InteractionWidgetClass);  // 设置 Widget 类
                
                // 添加到管理器
                IndicatorManager->AddIndicator(Indicator);

                Indicators.Add(Indicator);
            }
        }
    }

    CurrentOptions = InteractiveOptions;
}
```

### 17.4.2 交互提示 Widget 基类

让我们创建一个功能完整的交互提示 Widget：

```cpp
// InteractionPromptWidget.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "InteractionPromptWidget.generated.h"

class UTextBlock;
class UImage;
class UBorder;

/**
 * UInteractionPromptWidget
 * 
 * 交互提示 Widget 基类
 * 显示交互文本、按键提示等
 */
UCLASS()
class YOURGAME_API UInteractionPromptWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override;
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;

    /**
     * 设置交互文本
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void SetInteractionText(const FText& MainText, const FText& SubText);

    /**
     * 设置按键提示
     * 
     * @param KeyName 按键名称（如 "E", "F", "X"）
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void SetKeyPrompt(const FText& KeyName);

    /**
     * 设置进度（用于蓄力交互）
     * 
     * @param Progress 进度 0.0 - 1.0
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void SetProgress(float Progress);

    /**
     * 设置交互是否可用
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void SetInteractionEnabled(bool bEnabled);

protected:
    /** 主文本块 */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> MainTextBlock;

    /** 副文本块 */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> SubTextBlock;

    /** 按键提示文本 */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> KeyPromptText;

    /** 按键提示背景 */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UBorder> KeyPromptBorder;

    /** 进度条边框 */
    UPROPERTY(meta = (BindWidgetOptional))
    TObjectPtr<UBorder> ProgressBorder;

    /** 进度条填充 */
    UPROPERTY(meta = (BindWidgetOptional))
    TObjectPtr<UImage> ProgressFill;

    /** 根容器 */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UBorder> RootBorder;

    // ======== 样式配置 ========
    
    /** 启用时的颜色 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Appearance")
    FLinearColor EnabledColor = FLinearColor::White;

    /** 禁用时的颜色 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Appearance")
    FLinearColor DisabledColor = FLinearColor(0.5f, 0.5f, 0.5f, 1.0f);

    /** 进度条颜色 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Appearance")
    FLinearColor ProgressColor = FLinearColor::Green;

    /** 是否启用脉冲动画 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Animation")
    bool bEnablePulseAnimation = true;

    /** 脉冲速度 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Animation")
    float PulseSpeed = 2.0f;

private:
    /** 当前进度值 */
    float CurrentProgress = 0.0f;

    /** 动画时间累积 */
    float AnimationTime = 0.0f;

    /** 是否启用 */
    bool bIsEnabled = true;
};
```

```cpp
// InteractionPromptWidget.cpp
#include "InteractionPromptWidget.h"
#include "Components/TextBlock.h"
#include "Components/Image.h"
#include "Components/Border.h"
#include "Materials/MaterialInstanceDynamic.h"

void UInteractionPromptWidget::NativeConstruct()
{
    Super::NativeConstruct();

    // 初始化默认状态
    SetInteractionEnabled(true);
    SetProgress(0.0f);
}

void UInteractionPromptWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    Super::NativeTick(MyGeometry, InDeltaTime);

    // 脉冲动画
    if (bEnablePulseAnimation && bIsEnabled && KeyPromptBorder)
    {
        AnimationTime += InDeltaTime * PulseSpeed;
        float PulseValue = (FMath::Sin(AnimationTime) + 1.0f) * 0.5f;  // 0.0 - 1.0
        float Alpha = FMath::Lerp(0.7f, 1.0f, PulseValue);
        
        FLinearColor BorderColor = KeyPromptBorder->GetBrushColor();
        BorderColor.A = Alpha;
        KeyPromptBorder->SetBrushColor(BorderColor);
    }

    // 更新进度条
    if (ProgressFill && CurrentProgress > 0.0f)
    {
        // 设置进度填充的缩放
        FVector2D CurrentScale = ProgressFill->GetRenderTransform().Scale;
        CurrentScale.X = CurrentProgress;
        
        FWidgetTransform Transform = ProgressFill->GetRenderTransform();
        Transform.Scale = CurrentScale;
        ProgressFill->SetRenderTransform(Transform);
    }
}

void UInteractionPromptWidget::SetInteractionText(const FText& MainText, const FText& SubText)
{
    if (MainTextBlock)
    {
        MainTextBlock->SetText(MainText);
    }

    if (SubTextBlock)
    {
        SubTextBlock->SetText(SubText);
        SubTextBlock->SetVisibility(SubText.IsEmpty() ? ESlateVisibility::Collapsed : ESlateVisibility::Visible);
    }
}

void UInteractionPromptWidget::SetKeyPrompt(const FText& KeyName)
{
    if (KeyPromptText)
    {
        KeyPromptText->SetText(KeyName);
    }
}

void UInteractionPromptWidget::SetProgress(float Progress)
{
    CurrentProgress = FMath::Clamp(Progress, 0.0f, 1.0f);

    if (ProgressBorder)
    {
        ProgressBorder->SetVisibility(CurrentProgress > 0.0f ? 
                                      ESlateVisibility::Visible : 
                                      ESlateVisibility::Collapsed);
    }

    if (ProgressFill)
    {
        ProgressFill->SetColorAndOpacity(ProgressColor);
    }
}

void UInteractionPromptWidget::SetInteractionEnabled(bool bEnabled)
{
    bIsEnabled = bEnabled;

    FLinearColor TargetColor = bEnabled ? EnabledColor : DisabledColor;

    if (RootBorder)
    {
        RootBorder->SetBrushColor(TargetColor);
    }

    if (MainTextBlock)
    {
        MainTextBlock->SetColorAndOpacity(FSlateColor(TargetColor));
    }

    if (SubTextBlock)
    {
        SubTextBlock->SetColorAndOpacity(FSlateColor(TargetColor));
    }
}
```

### 17.4.3 世界空间 UI 实现

创建一个浮动在交互对象上方的 3D Widget：

```cpp
// InteractionWorldWidget.h
#pragma once

#include "CoreMinimal.h"
#include "Components/WidgetComponent.h"
#include "InteractionWorldWidget.generated.h"

/**
 * UInteractionWorldWidget
 * 
 * 世界空间交互提示组件
 * 自动朝向玩家相机
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class YOURGAME_API UInteractionWorldWidget : public UWidgetComponent
{
    GENERATED_BODY()

public:
    UInteractionWorldWidget();

    virtual void TickComponent(float DeltaTime, ELevelTick TickType, 
                              FActorComponentTickFunction* ThisTickFunction) override;

    /**
     * 设置交互信息
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void SetInteractionInfo(const FText& MainText, const FText& SubText, const FText& KeyHint);

    /**
     * 设置可见性（带淡入淡出）
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void SetPromptVisible(bool bVisible, float FadeDuration = 0.2f);

protected:
    virtual void BeginPlay() override;

    /** 更新朝向 */
    void UpdateFacingDirection();

    /** 更新淡入淡出 */
    void UpdateFade(float DeltaTime);

protected:
    /** 是否自动朝向相机 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    bool bAlwaysFaceCamera = true;

    /** 距离衰减起始距离 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    float DistanceFadeStart = 500.0f;

    /** 距离衰减结束距离 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    float DistanceFadeEnd = 1000.0f;

    /** Widget 垂直偏移 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    float VerticalOffset = 100.0f;

private:
    /** 目标可见性 */
    bool bTargetVisible = false;

    /** 当前不透明度 */
    float CurrentOpacity = 0.0f;

    /** 淡入淡出速度 */
    float FadeSpeed = 5.0f;
};
```

```cpp
// InteractionWorldWidget.cpp
#include "InteractionWorldWidget.h"
#include "Camera/CameraComponent.h"
#include "GameFramework/PlayerController.h"
#include "Kismet/GameplayStatics.h"
#include "InteractionPromptWidget.h"

UInteractionWorldWidget::UInteractionWorldWidget()
{
    PrimaryComponentTick.bCanEverTick = true;
    
    // 世界空间设置
    SetWidgetSpace(EWidgetSpace::World);
    SetDrawSize(FVector2D(300.0f, 100.0f));
    SetPivot(FVector2D(0.5f, 1.0f));  // 底部中心为锚点
    
    // 碰撞设置
    SetCollisionEnabled(ECollisionEnabled::NoCollision);
    
    // 初始不可见
    SetVisibility(false);
}

void UInteractionWorldWidget::BeginPlay()
{
    Super::BeginPlay();

    // 设置相对位置偏移
    AddRelativeLocation(FVector(0.0f, 0.0f, VerticalOffset));
}

void UInteractionWorldWidget::TickComponent(float DeltaTime, ELevelTick TickType, 
                                           FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    // 更新朝向
    if (bAlwaysFaceCamera)
    {
        UpdateFacingDirection();
    }

    // 更新淡入淡出
    UpdateFade(DeltaTime);

    // 距离衰减
    if (APlayerController* PC = UGameplayStatics::GetPlayerController(GetWorld(), 0))
    {
        if (APawn* Pawn = PC->GetPawn())
        {
            float Distance = FVector::Dist(GetComponentLocation(), Pawn->GetActorLocation());
            float Alpha = 1.0f - FMath::GetRangePct(DistanceFadeStart, DistanceFadeEnd, Distance);
            Alpha = FMath::Clamp(Alpha, 0.0f, 1.0f);
            
            // 应用最终不透明度
            float FinalOpacity = CurrentOpacity * Alpha;
            SetRenderCustomDepth(FinalOpacity > 0.01f);
        }
    }
}

void UInteractionWorldWidget::UpdateFacingDirection()
{
    if (APlayerController* PC = UGameplayStatics::GetPlayerController(GetWorld(), 0))
    {
        FVector CameraLocation;
        FRotator CameraRotation;
        PC->GetPlayerViewPoint(CameraLocation, CameraRotation);

        FVector ToCamera = CameraLocation - GetComponentLocation();
        ToCamera.Z = 0.0f;  // 只在水平面旋转
        
        if (!ToCamera.IsNearlyZero())
        {
            FRotator LookAtRotation = ToCamera.Rotation();
            SetWorldRotation(LookAtRotation);
        }
    }
}

void UInteractionWorldWidget::UpdateFade(float DeltaTime)
{
    float TargetOpacity = bTargetVisible ? 1.0f : 0.0f;
    CurrentOpacity = FMath::FInterpTo(CurrentOpacity, TargetOpacity, DeltaTime, FadeSpeed);

    // 更新可见性
    if (CurrentOpacity < 0.01f)
    {
        SetVisibility(false);
    }
    else
    {
        SetVisibility(true);
        // 这里需要通知 Widget 更新不透明度
        if (UUserWidget* Widget = GetUserWidgetObject())
        {
            Widget->SetRenderOpacity(CurrentOpacity);
        }
    }
}

void UInteractionWorldWidget::SetInteractionInfo(const FText& MainText, const FText& SubText, const FText& KeyHint)
{
    if (UUserWidget* Widget = GetUserWidgetObject())
    {
        if (UInteractionPromptWidget* PromptWidget = Cast<UInteractionPromptWidget>(Widget))
        {
            PromptWidget->SetInteractionText(MainText, SubText);
            PromptWidget->SetKeyPrompt(KeyHint);
        }
    }
}

void UInteractionWorldWidget::SetPromptVisible(bool bVisible, float FadeDuration)
{
    bTargetVisible = bVisible;
    FadeSpeed = FadeDuration > 0.0f ? (1.0f / FadeDuration) : 10.0f;
}
```

### 17.4.4 屏幕空间 UI 实现

创建一个始终位于屏幕中央的交互提示：

```cpp
// InteractionScreenWidget.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "InteractionScreenWidget.generated.h"

class UVerticalBox;
class UInteractionPromptWidget;

/**
 * UInteractionScreenWidget
 * 
 * 屏幕空间交互提示
 * 显示在屏幕中央或底部
 */
UCLASS()
class YOURGAME_API UInteractionScreenWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override;

    /**
     * 显示交互提示
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void ShowInteractionPrompt(const FText& MainText, const FText& SubText, const FText& KeyHint);

    /**
     * 隐藏交互提示
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void HideInteractionPrompt();

    /**
     * 显示多个交互选项
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void ShowMultipleOptions(const TArray<FText>& Options);

protected:
    /** 交互提示容器 */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UVerticalBox> InteractionContainer;

    /** 主提示 Widget */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UInteractionPromptWidget> MainPromptWidget;

    /** 提示 Widget 类（用于多选项） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    TSubclassOf<UInteractionPromptWidget> PromptWidgetClass;

private:
    /** 动态创建的提示 Widget 列表 */
    UPROPERTY()
    TArray<TObjectPtr<UInteractionPromptWidget>> DynamicPromptWidgets;
};
```

```cpp
// InteractionScreenWidget.cpp
#include "InteractionScreenWidget.h"
#include "Components/VerticalBox.h"
#include "InteractionPromptWidget.h"

void UInteractionScreenWidget::NativeConstruct()
{
    Super::NativeConstruct();

    HideInteractionPrompt();
}

void UInteractionScreenWidget::ShowInteractionPrompt(const FText& MainText, const FText& SubText, const FText& KeyHint)
{
    if (MainPromptWidget)
    {
        MainPromptWidget->SetVisibility(ESlateVisibility::Visible);
        MainPromptWidget->SetInteractionText(MainText, SubText);
        MainPromptWidget->SetKeyPrompt(KeyHint);
        MainPromptWidget->SetInteractionEnabled(true);
    }

    // 隐藏多选项
    for (UInteractionPromptWidget* Widget : DynamicPromptWidgets)
    {
        if (Widget)
        {
            Widget->SetVisibility(ESlateVisibility::Collapsed);
        }
    }
}

void UInteractionScreenWidget::HideInteractionPrompt()
{
    if (MainPromptWidget)
    {
        MainPromptWidget->SetVisibility(ESlateVisibility::Collapsed);
    }

    for (UInteractionPromptWidget* Widget : DynamicPromptWidgets)
    {
        if (Widget)
        {
            Widget->SetVisibility(ESlateVisibility::Collapsed);
        }
    }
}

void UInteractionScreenWidget::ShowMultipleOptions(const TArray<FText>& Options)
{
    // 隐藏主提示
    if (MainPromptWidget)
    {
        MainPromptWidget->SetVisibility(ESlateVisibility::Collapsed);
    }

    // 确保有足够的 Widget
    while (DynamicPromptWidgets.Num() < Options.Num())
    {
        if (PromptWidgetClass && InteractionContainer)
        {
            UInteractionPromptWidget* NewWidget = CreateWidget<UInteractionPromptWidget>(
                GetOwningPlayer(), PromptWidgetClass);
            
            if (NewWidget)
            {
                InteractionContainer->AddChild(NewWidget);
                DynamicPromptWidgets.Add(NewWidget);
            }
        }
    }

    // 更新每个选项
    for (int32 i = 0; i < Options.Num(); ++i)
    {
        if (DynamicPromptWidgets.IsValidIndex(i))
        {
            UInteractionPromptWidget* Widget = DynamicPromptWidgets[i];
            Widget->SetVisibility(ESlateVisibility::Visible);
            Widget->SetInteractionText(Options[i], FText::GetEmpty());
            Widget->SetKeyPrompt(FText::AsNumber(i + 1));  // 1, 2, 3...
        }
    }

    // 隐藏多余的 Widget
    for (int32 i = Options.Num(); i < DynamicPromptWidgets.Num(); ++i)
    {
        DynamicPromptWidgets[i]->SetVisibility(ESlateVisibility::Collapsed);
    }
}
```

### 17.4.5 平台适配的按键提示

创建一个自动适配不同平台按键的系统：

```cpp
// InteractionInputHelper.h
#pragma once

#include "CoreMinimal.h"
#include "Kismet/BlueprintFunctionLibrary.h"
#include "InteractionInputHelper.generated.h"

/**
 * 输入平台类型
 */
UENUM(BlueprintType)
enum class EInputPlatformType : uint8
{
    KeyboardMouse,
    Gamepad,
    Touch
};

/**
 * UInteractionInputHelper
 * 
 * 交互输入辅助函数库
 * 处理不同平台的按键提示
 */
UCLASS()
class YOURGAME_API UInteractionInputHelper : public UBlueprintFunctionLibrary
{
    GENERATED_BODY()

public:
    /**
     * 获取当前输入平台
     */
    UFUNCTION(BlueprintPure, Category = "Interaction|Input")
    static EInputPlatformType GetCurrentInputPlatform();

    /**
     * 获取交互按键提示文本
     */
    UFUNCTION(BlueprintPure, Category = "Interaction|Input")
    static FText GetInteractionKeyPrompt();

    /**
     * 获取格式化的按键提示
     * 
     * @param ActionName 动作名称（如 "Interact"）
     * @return 格式化的文本（如 "[E] 拾取", "按 A 拾取"）
     */
    UFUNCTION(BlueprintPure, Category = "Interaction|Input")
    static FText GetFormattedKeyPrompt(const FName& ActionName);

    /**
     * 获取按键图标路径
     */
    UFUNCTION(BlueprintPure, Category = "Interaction|Input")
    static FSoftObjectPath GetKeyIconPath(const FName& ActionName);
};
```

```cpp
// InteractionInputHelper.cpp
#include "InteractionInputHelper.h"
#include "GameFramework/PlayerController.h"
#include "GameFramework/PlayerInput.h"
#include "Kismet/GameplayStatics.h"

EInputPlatformType UInteractionInputHelper::GetCurrentInputPlatform()
{
    // 检测当前使用的输入设备
    if (APlayerController* PC = UGameplayStatics::GetPlayerController(GWorld, 0))
    {
        // 这里可以使用 Common UI Plugin 的检测逻辑
        // 或者自己实现平台检测
        
        #if PLATFORM_DESKTOP
            return EInputPlatformType::KeyboardMouse;
        #elif PLATFORM_ANDROID || PLATFORM_IOS
            return EInputPlatformType::Touch;
        #else
            return EInputPlatformType::Gamepad;
        #endif
    }

    return EInputPlatformType::KeyboardMouse;
}

FText UInteractionInputHelper::GetInteractionKeyPrompt()
{
    EInputPlatformType Platform = GetCurrentInputPlatform();

    switch (Platform)
    {
    case EInputPlatformType::KeyboardMouse:
        return FText::FromString(TEXT("E"));
    
    case EInputPlatformType::Gamepad:
        return FText::FromString(TEXT("X"));  // PlayStation
        // 或者使用图标字体
    
    case EInputPlatformType::Touch:
        return FText::FromString(TEXT("点击"));
    
    default:
        return FText::FromString(TEXT("?"));
    }
}

FText UInteractionInputHelper::GetFormattedKeyPrompt(const FName& ActionName)
{
    EInputPlatformType Platform = GetCurrentInputPlatform();
    FText KeyText = GetInteractionKeyPrompt();

    switch (Platform)
    {
    case EInputPlatformType::KeyboardMouse:
        return FText::Format(NSLOCTEXT("Interaction", "PCPrompt", "[{0}]"), KeyText);
    
    case EInputPlatformType::Gamepad:
        return FText::Format(NSLOCTEXT("Interaction", "GamepadPrompt", "按 {0}"), KeyText);
    
    case EInputPlatformType::Touch:
        return KeyText;
    
    default:
        return KeyText;
    }
}

FSoftObjectPath UInteractionInputHelper::GetKeyIconPath(const FName& ActionName)
{
    EInputPlatformType Platform = GetCurrentInputPlatform();

    // 根据平台返回对应的图标资源路径
    switch (Platform)
    {
    case EInputPlatformType::KeyboardMouse:
        return FSoftObjectPath(TEXT("/Game/UI/Icons/Keys/Key_E.Key_E"));
    
    case EInputPlatformType::Gamepad:
        // 可以根据具体的手柄类型返回不同图标
        // Xbox: Button_X, PlayStation: Button_Square
        return FSoftObjectPath(TEXT("/Game/UI/Icons/Buttons/Button_X.Button_X"));
    
    case EInputPlatformType::Touch:
        return FSoftObjectPath(TEXT("/Game/UI/Icons/Touch/Touch_Tap.Touch_Tap"));
    
    default:
        return FSoftObjectPath();
    }
}
```

---

## 17.5 交互流程与网络同步

### 17.5.1 完整交互能力实现

让我们创建一个完整的交互能力类，展示从检测到触发的全流程：

```cpp
// LyraGameplayAbility_Interact.h
#pragma once

#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "Interaction/InteractionOption.h"
#include "LyraGameplayAbility_Interact.generated.h"

class UIndicatorDescriptor;
class UObject;
class UUserWidget;
struct FFrame;
struct FGameplayAbilityActorInfo;
struct FGameplayEventData;

/**
 * ULyraGameplayAbility_Interact
 *
 * 完整的交互能力实现
 * - 自动检测附近交互对象
 * - 授予交互能力
 * - 射线检测准星指向的对象
 * - 显示 UI 提示
 * - 处理交互触发
 */
UCLASS(Abstract)
class ULyraGameplayAbility_Interact : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    ULyraGameplayAbility_Interact(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    // ======== Gameplay Ability 生命周期 ========
    
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, 
                                const FGameplayAbilityActorInfo* ActorInfo, 
                                const FGameplayAbilityActivationInfo ActivationInfo, 
                                const FGameplayEventData* TriggerEventData) override;

    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, 
                           const FGameplayAbilityActorInfo* ActorInfo, 
                           const FGameplayAbilityActivationInfo ActivationInfo, 
                           bool bReplicateEndAbility, 
                           bool bWasCancelled) override;

    // ======== 蓝图回调 ========
    
    /**
     * 交互选项更新回调
     * 
     * 从 AbilityTask 接收到新的交互选项时调用
     * 
     * @param InteractiveOptions 当前可用的交互选项列表
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void UpdateInteractions(const TArray<FInteractionOption>& InteractiveOptions);

    /**
     * 触发交互
     * 
     * 由玩家输入调用（按 E 键）
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void TriggerInteraction();

    /**
     * 触发指定索引的交互
     * 
     * 用于多选项交互
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void TriggerInteractionAtIndex(int32 Index);

protected:
    // ======== 当前状态 ========
    
    /** 当前可用的交互选项 */
    UPROPERTY(BlueprintReadWrite, Category = "Interaction")
    TArray<FInteractionOption> CurrentOptions;

    /** 当前显示的 UI 指示器 */
    UPROPERTY()
    TArray<TObjectPtr<UIndicatorDescriptor>> Indicators;

    // ======== 配置参数 ========
    
    /** 交互扫描频率（秒） */
    UPROPERTY(EditDefaultsOnly, Category = "Interaction")
    float InteractionScanRate = 0.1f;

    /** 交互扫描范围（cm） */
    UPROPERTY(EditDefaultsOnly, Category = "Interaction")
    float InteractionScanRange = 500.0f;

    /** 附近交互对象搜索范围（cm） */
    UPROPERTY(EditDefaultsOnly, Category = "Interaction")
    float NearbyInteractionRange = 300.0f;

    /** 默认交互 Widget 类 */
    UPROPERTY(EditDefaultsOnly, Category = "Interaction|UI")
    TSoftClassPtr<UUserWidget> DefaultInteractionWidgetClass;

    /** 是否使用准星射线检测 */
    UPROPERTY(EditDefaultsOnly, Category = "Interaction|Detection")
    bool bUseLineTrace = true;

    /** 碰撞配置名称 */
    UPROPERTY(EditDefaultsOnly, Category = "Interaction|Detection")
    FCollisionProfileName TraceProfile = FName(TEXT("Interaction"));

    // ======== 辅助方法 ========
    
    /**
     * 清除所有 UI 指示器
     */
    void ClearIndicators();

    /**
     * 获取 Indicator Manager
     */
    class ULyraIndicatorManagerComponent* GetIndicatorManager() const;
};
```

完整的实现：

```cpp
// LyraGameplayAbility_Interact.cpp
#include "LyraGameplayAbility_Interact.h"
#include "AbilitySystemComponent.h"
#include "Interaction/IInteractableTarget.h"
#include "Interaction/InteractionStatics.h"
#include "Interaction/Tasks/AbilityTask_GrantNearbyInteraction.h"
#include "Interaction/Tasks/AbilityTask_WaitForInteractableTargets_SingleLineTrace.h"
#include "NativeGameplayTags.h"
#include "Player/LyraPlayerController.h"
#include "UI/IndicatorSystem/IndicatorDescriptor.h"
#include "UI/IndicatorSystem/LyraIndicatorManagerComponent.h"

// 定义 Gameplay Tags
UE_DEFINE_GAMEPLAY_TAG_STATIC(TAG_Ability_Interaction_Activate, "Ability.Interaction.Activate");
UE_DEFINE_GAMEPLAY_TAG(TAG_INTERACTION_DURATION_MESSAGE, "Ability.Interaction.Duration.Message");

ULyraGameplayAbility_Interact::ULyraGameplayAbility_Interact(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 能力配置
    ActivationPolicy = ELyraAbilityActivationPolicy::OnSpawn;  // 角色生成时自动激活
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;  // 每个 Actor 一个实例
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;  // 本地预测
}

void ULyraGameplayAbility_Interact::ActivateAbility(
    const FGameplayAbilitySpecHandle Handle, 
    const FGameplayAbilityActorInfo* ActorInfo, 
    const FGameplayAbilityActivationInfo ActivationInfo, 
    const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    UAbilitySystemComponent* AbilitySystem = GetAbilitySystemComponentFromActorInfo();
    if (AbilitySystem && AbilitySystem->GetOwnerRole() == ROLE_Authority)
    {
        // ======== 服务器：启动附近交互对象搜索 ========
        // 这个任务会定期扫描角色周围，自动授予交互能力
        UAbilityTask_GrantNearbyInteraction* GrantTask = 
            UAbilityTask_GrantNearbyInteraction::GrantAbilitiesForNearbyInteractors(
                this, NearbyInteractionRange, InteractionScanRate);
        GrantTask->ReadyForActivation();
    }

    if (bUseLineTrace)
    {
        // ======== 客户端+服务器：启动射线检测 ========
        // 检测准星指向的交互对象
        
        // 构建交互查询
        FInteractionQuery InteractionQuery;
        InteractionQuery.RequestingAvatar = GetAvatarActorFromActorInfo();
        InteractionQuery.RequestingController = GetControllerFromActorInfo();

        // 创建起始位置信息（从角色位置开始）
        FGameplayAbilityTargetingLocationInfo StartLocation;
        StartLocation.LocationType = EGameplayAbilityTargetingLocationType::ActorTransform;
        StartLocation.SourceActor = GetAvatarActorFromActorInfo();

        // 创建射线检测任务
        UAbilityTask_WaitForInteractableTargets_SingleLineTrace* LineTraceTask = 
            UAbilityTask_WaitForInteractableTargets_SingleLineTrace::WaitForInteractableTargets_SingleLineTrace(
                this, 
                InteractionQuery, 
                TraceProfile, 
                StartLocation, 
                InteractionScanRange, 
                InteractionScanRate, 
                false);  // bShowDebug

        // 绑定回调
        LineTraceTask->InteractableObjectsChanged.AddDynamic(this, &ThisClass::UpdateInteractions);
        LineTraceTask->ReadyForActivation();
    }
}

void ULyraGameplayAbility_Interact::EndAbility(
    const FGameplayAbilitySpecHandle Handle, 
    const FGameplayAbilityActorInfo* ActorInfo, 
    const FGameplayAbilityActivationInfo ActivationInfo, 
    bool bReplicateEndAbility, 
    bool bWasCancelled)
{
    // 清理 UI
    ClearIndicators();

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}

void ULyraGameplayAbility_Interact::UpdateInteractions(const TArray<FInteractionOption>& InteractiveOptions)
{
    if (ALyraPlayerController* PC = GetLyraPlayerControllerFromActorInfo())
    {
        if (ULyraIndicatorManagerComponent* IndicatorManager = GetIndicatorManager())
        {
            // ======== 步骤1：清除旧的指示器 ========
            ClearIndicators();

            // ======== 步骤2：为每个交互选项创建新指示器 ========
            for (const FInteractionOption& InteractionOption : InteractiveOptions)
            {
                AActor* InteractableTargetActor = 
                    UInteractionStatics::GetActorFromInteractableTarget(InteractionOption.InteractableTarget);

                if (!InteractableTargetActor)
                {
                    continue;
                }

                // 确定 Widget 类
                TSoftClassPtr<UUserWidget> InteractionWidgetClass = 
                    InteractionOption.InteractionWidgetClass.IsNull() ? 
                    DefaultInteractionWidgetClass : 
                    InteractionOption.InteractionWidgetClass;

                // 创建指示器描述符
                UIndicatorDescriptor* Indicator = NewObject<UIndicatorDescriptor>();
                Indicator->SetDataObject(InteractableTargetActor);
                Indicator->SetSceneComponent(InteractableTargetActor->GetRootComponent());
                Indicator->SetIndicatorClass(InteractionWidgetClass);
                
                // 可以在这里设置更多属性
                // Indicator->bClampToScreen = true;
                // Indicator->bShowClampToScreenArrow = false;

                // 添加到管理器
                IndicatorManager->AddIndicator(Indicator);
                Indicators.Add(Indicator);
            }
        }
    }

    // 保存当前选项
    CurrentOptions = InteractiveOptions;

    // 可以在这里触发蓝图事件
    // OnInteractionsUpdated(CurrentOptions);
}

void ULyraGameplayAbility_Interact::TriggerInteraction()
{
    TriggerInteractionAtIndex(0);  // 触发第一个选项
}

void ULyraGameplayAbility_Interact::TriggerInteractionAtIndex(int32 Index)
{
    if (!CurrentOptions.IsValidIndex(Index))
    {
        return;
    }

    UAbilitySystemComponent* AbilitySystem = GetAbilitySystemComponentFromActorInfo();
    if (!AbilitySystem)
    {
        return;
    }

    const FInteractionOption& InteractionOption = CurrentOptions[Index];

    AActor* Instigator = GetAvatarActorFromActorInfo();
    AActor* InteractableTargetActor = 
        UInteractionStatics::GetActorFromInteractableTarget(InteractionOption.InteractableTarget);

    // ======== 构建 Gameplay Event Data ========
    FGameplayEventData Payload;
    Payload.EventTag = TAG_Ability_Interaction_Activate;
    Payload.Instigator = Instigator;
    Payload.Target = InteractableTargetActor;

    // 允许目标自定义事件数据
    InteractionOption.InteractableTarget->CustomizeInteractionEventData(
        TAG_Ability_Interaction_Activate, Payload);

    // ======== 激活目标的交互能力 ========
    AActor* TargetActor = const_cast<AActor*>(ToRawPtr(Payload.Target));

    // 构建 Actor Info
    FGameplayAbilityActorInfo ActorInfo;
    ActorInfo.InitFromActor(InteractableTargetActor, TargetActor, InteractionOption.TargetAbilitySystem);

    // 通过 Gameplay Event 触发能力
    const bool bSuccess = InteractionOption.TargetAbilitySystem->TriggerAbilityFromGameplayEvent(
        InteractionOption.TargetInteractionAbilityHandle,
        &ActorInfo,
        TAG_Ability_Interaction_Activate,
        &Payload,
        *InteractionOption.TargetAbilitySystem
    );

    if (bSuccess)
    {
        // 交互成功，可以播放反馈
        // PlayInteractionFeedback();
    }
}

void ULyraGameplayAbility_Interact::ClearIndicators()
{
    if (ULyraIndicatorManagerComponent* IndicatorManager = GetIndicatorManager())
    {
        for (UIndicatorDescriptor* Indicator : Indicators)
        {
            if (Indicator)
            {
                IndicatorManager->RemoveIndicator(Indicator);
            }
        }
    }
    
    Indicators.Reset();
}

ULyraIndicatorManagerComponent* ULyraGameplayAbility_Interact::GetIndicatorManager() const
{
    if (ALyraPlayerController* PC = GetLyraPlayerControllerFromActorInfo())
    {
        return ULyraIndicatorManagerComponent::GetComponent(PC);
    }
    return nullptr;
}
```

### 17.5.2 网络同步实现

交互系统的网络同步主要依赖 GAS 的网络机制，但我们需要注意一些细节：

```cpp
// InteractableItemBase.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Interaction/IInteractableTarget.h"
#include "AbilitySystemInterface.h"
#include "InteractableItemBase.generated.h"

class UAbilitySystemComponent;
class UGameplayAbility;

/**
 * AInteractableItemBase
 * 
 * 可交互物品基类
 * 带网络同步的完整实现
 */
UCLASS(Abstract)
class YOURGAME_API AInteractableItemBase : public AActor, 
                                          public IInteractableTarget,
                                          public IAbilitySystemInterface
{
    GENERATED_BODY()
    
public:    
    AInteractableItemBase();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // ======== IInteractableTarget 接口 ========
    
    virtual void GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                          FInteractionOptionBuilder& OptionBuilder) override;

    virtual void CustomizeInteractionEventData(const FGameplayTag& InteractionEventTag, 
                                               FGameplayEventData& InOutEventData) override;

    // ======== IAbilitySystemInterface 接口 ========
    
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

protected:
    virtual void BeginPlay() override;

    /**
     * 交互执行（服务器端）
     * 
     * 这个方法会被交互能力调用
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Interaction")
    void OnInteracted(AActor* InteractingActor, AController* InteractingController);
    virtual void OnInteracted_Implementation(AActor* InteractingActor, AController* InteractingController);

    /**
     * 多播通知所有客户端
     */
    UFUNCTION(NetMulticast, Reliable)
    void Multicast_OnInteracted(AActor* InteractingActor);

    /**
     * 客户端视觉反馈
     */
    UFUNCTION(BlueprintImplementableEvent, Category = "Interaction")
    void OnInteractionVisualFeedback(AActor* InteractingActor);

protected:
    /** 能力系统组件 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    /** 交互能力类 */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Interaction")
    TSubclassOf<UGameplayAbility> InteractionAbilityClass;

    /** 交互能力句柄 */
    FGameplayAbilitySpecHandle InteractionAbilityHandle;

    /** 交互显示文本 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction", Replicated)
    FText InteractionText;

    /** 交互副文本 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction", Replicated)
    FText InteractionSubText;

    /** 是否可以交互 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction", ReplicatedUsing = OnRep_bIsInteractable)
    bool bIsInteractable = true;

    /** 网格体组件 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UStaticMeshComponent> MeshComponent;

private:
    UFUNCTION()
    void OnRep_bIsInteractable();
};
```

```cpp
// InteractableItemBase.cpp
#include "InteractableItemBase.h"
#include "AbilitySystemComponent.h"
#include "Net/UnrealNetwork.h"
#include "Components/StaticMeshComponent.h"

AInteractableItemBase::AInteractableItemBase()
{
    PrimaryActorTick.bCanEverTick = false;
    bReplicates = true;  // 启用网络复制
    SetReplicateMovement(true);

    // 创建能力系统组件
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);

    // 创建网格体
    MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("MeshComponent"));
    RootComponent = MeshComponent;
    MeshComponent->SetCollisionProfileName(TEXT("Interactable"));
}

void AInteractableItemBase::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AInteractableItemBase, InteractionText);
    DOREPLIFETIME(AInteractableItemBase, InteractionSubText);
    DOREPLIFETIME(AInteractableItemBase, bIsInteractable);
}

void AInteractableItemBase::BeginPlay()
{
    Super::BeginPlay();

    // 仅在服务器上授予能力
    if (HasAuthority() && InteractionAbilityClass && AbilitySystemComponent)
    {
        FGameplayAbilitySpec AbilitySpec(InteractionAbilityClass, 1, INDEX_NONE, this);
        InteractionAbilityHandle = AbilitySystemComponent->GiveAbility(AbilitySpec);
    }
}

UAbilitySystemComponent* AInteractableItemBase::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}

void AInteractableItemBase::GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                                     FInteractionOptionBuilder& OptionBuilder)
{
    if (!bIsInteractable)
    {
        return;  // 不可交互时不返回选项
    }

    FInteractionOption Option;
    Option.Text = InteractionText;
    Option.SubText = InteractionSubText;
    
    // 使用目标的能力系统
    Option.TargetAbilitySystem = AbilitySystemComponent;
    Option.TargetInteractionAbilityHandle = InteractionAbilityHandle;

    OptionBuilder.AddInteractionOption(Option);
}

void AInteractableItemBase::CustomizeInteractionEventData(const FGameplayTag& InteractionEventTag, 
                                                          FGameplayEventData& InOutEventData)
{
    // 将当前对象作为 OptionalObject 传递
    InOutEventData.OptionalObject = this;
}

void AInteractableItemBase::OnInteracted_Implementation(AActor* InteractingActor, AController* InteractingController)
{
    // 服务器端逻辑
    if (HasAuthority())
    {
        // 执行交互逻辑
        // ...

        // 通知所有客户端
        Multicast_OnInteracted(InteractingActor);
    }
}

void AInteractableItemBase::Multicast_OnInteracted_Implementation(AActor* InteractingActor)
{
    // 所有客户端（包括服务器）都会执行
    OnInteractionVisualFeedback(InteractingActor);
}

void AInteractableItemBase::OnRep_bIsInteractable()
{
    // 可交互状态变化时的视觉反馈
    // 例如：改变材质颜色、启用/禁用发光效果等
}
```

### 17.5.3 交互能力示例

创建一个实际的交互能力：

```cpp
// GA_InteractPickup.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_InteractPickup.generated.h"

/**
 * UGA_InteractPickup
 * 
 * 拾取物品交互能力
 * 在物品上执行，由玩家触发
 */
UCLASS()
class YOURGAME_API UGA_InteractPickup : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_InteractPickup();

    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, 
                                const FGameplayAbilityActorInfo* ActorInfo, 
                                const FGameplayAbilityActivationInfo ActivationInfo, 
                                const FGameplayEventData* TriggerEventData) override;

protected:
    /**
     * 执行拾取（服务器）
     */
    UFUNCTION(BlueprintCallable, Category = "Ability")
    void PerformPickup(AActor* ItemActor, AActor* PickingActor);

    /**
     * 服务器 RPC
     */
    UFUNCTION(Server, Reliable)
    void ServerPerformPickup(AActor* ItemActor, AActor* PickingActor);

    /**
     * 拾取成功回调
     */
    UFUNCTION(BlueprintImplementableEvent, Category = "Ability")
    void OnPickupSuccess(AActor* ItemActor, AActor* PickingActor);

protected:
    /** 拾取动画蒙太奇 */
    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    TObjectPtr<UAnimMontage> PickupMontage;

    /** 拾取音效 */
    UPROPERTY(EditDefaultsOnly, Category = "Audio")
    TObjectPtr<USoundBase> PickupSound;
};
```

```cpp
// GA_InteractPickup.cpp
#include "GA_InteractPickup.h"
#include "AbilitySystemComponent.h"
#include "Kismet/GameplayStatics.h"

UGA_InteractPickup::UGA_InteractPickup()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated;
}

void UGA_InteractPickup::ActivateAbility(
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

    // 获取物品和拾取者
    AActor* ItemActor = GetAvatarActorFromActorInfo();  // 物品本身
    AActor* PickingActor = nullptr;

    if (TriggerEventData)
    {
        PickingActor = const_cast<AActor*>(ToRawPtr(TriggerEventData->Instigator));
    }

    if (!PickingActor)
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 执行拾取
    if (HasAuthority(&ActivationInfo))
    {
        PerformPickup(ItemActor, PickingActor);
    }
    else
    {
        ServerPerformPickup(ItemActor, PickingActor);
    }
}

void UGA_InteractPickup::PerformPickup(AActor* ItemActor, AActor* PickingActor)
{
    if (!HasAuthority(&CurrentActivationInfo))
    {
        return;
    }

    // ======== 服务器端逻辑 ========
    
    // 1. 播放动画（可选）
    if (PickupMontage && PickingActor)
    {
        if (ACharacter* Character = Cast<ACharacter>(PickingActor))
        {
            Character->PlayAnimMontage(PickupMontage);
        }
    }

    // 2. 播放音效
    if (PickupSound)
    {
        UGameplayStatics::PlaySoundAtLocation(this, PickupSound, ItemActor->GetActorLocation());
    }

    // 3. 将物品添加到玩家背包
    // if (UInventoryComponent* Inventory = PickingActor->FindComponentByClass<UInventoryComponent>())
    // {
    //     Inventory->AddItem(ItemActor);
    // }

    // 4. 销毁或隐藏物品
    ItemActor->SetActorHiddenInGame(true);
    ItemActor->SetActorEnableCollision(false);

    // 延迟销毁（给时间同步）
    ItemActor->SetLifeSpan(1.0f);

    // 5. 触发成功事件
    OnPickupSuccess(ItemActor, PickingActor);

    // 结束能力
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}

void UGA_InteractPickup::ServerPerformPickup_Implementation(AActor* ItemActor, AActor* PickingActor)
{
    PerformPickup(ItemActor, PickingActor);
}
```

### 17.5.4 交互冷却系统

实现交互冷却，防止玩家快速重复交互：

```cpp
// InteractionCooldownComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "InteractionCooldownComponent.generated.h"

/**
 * UInteractionCooldownComponent
 * 
 * 交互冷却管理组件
 * 附加到玩家角色上
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class YOURGAME_API UInteractionCooldownComponent : public UActorComponent
{
    GENERATED_BODY()

public:    
    UInteractionCooldownComponent();

    /**
     * 检查是否可以交互
     */
    UFUNCTION(BlueprintPure, Category = "Interaction")
    bool CanInteract(AActor* Target = nullptr) const;

    /**
     * 开始冷却
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void StartCooldown(AActor* Target = nullptr, float CustomDuration = -1.0f);

    /**
     * 获取剩余冷却时间
     */
    UFUNCTION(BlueprintPure, Category = "Interaction")
    float GetRemainingCooldown(AActor* Target = nullptr) const;

protected:
    virtual void BeginPlay() override;

    /** 清理过期的冷却记录 */
    void CleanupExpiredCooldowns();

protected:
    /** 全局冷却时间 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cooldown")
    float GlobalCooldownDuration = 0.5f;

    /** 是否使用 per-target 冷却 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cooldown")
    bool bUsePerTargetCooldown = true;

    /** Per-target 冷却时间 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cooldown")
    float PerTargetCooldownDuration = 1.0f;

private:
    /** 全局冷却结束时间 */
    float GlobalCooldownEndTime = 0.0f;

    /** Per-target 冷却记录 */
    TMap<TWeakObjectPtr<AActor>, float> TargetCooldowns;

    /** 清理定时器 */
    FTimerHandle CleanupTimerHandle;
};
```

```cpp
// InteractionCooldownComponent.cpp
#include "InteractionCooldownComponent.h"
#include "TimerManager.h"
#include "Engine/World.h"

UInteractionCooldownComponent::UInteractionCooldownComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

void UInteractionCooldownComponent::BeginPlay()
{
    Super::BeginPlay();

    // 启动清理定时器
    GetWorld()->GetTimerManager().SetTimer(
        CleanupTimerHandle, 
        this, 
        &UInteractionCooldownComponent::CleanupExpiredCooldowns, 
        5.0f,  // 每5秒清理一次
        true);
}

bool UInteractionCooldownComponent::CanInteract(AActor* Target) const
{
    float CurrentTime = GetWorld()->GetTimeSeconds();

    // 检查全局冷却
    if (CurrentTime < GlobalCooldownEndTime)
    {
        return false;
    }

    // 检查 per-target 冷却
    if (bUsePerTargetCooldown && Target)
    {
        if (const float* CooldownEndTime = TargetCooldowns.Find(Target))
        {
            if (CurrentTime < *CooldownEndTime)
            {
                return false;
            }
        }
    }

    return true;
}

void UInteractionCooldownComponent::StartCooldown(AActor* Target, float CustomDuration)
{
    float CurrentTime = GetWorld()->GetTimeSeconds();

    // 设置全局冷却
    float GlobalDuration = CustomDuration > 0.0f ? CustomDuration : GlobalCooldownDuration;
    GlobalCooldownEndTime = CurrentTime + GlobalDuration;

    // 设置 per-target 冷却
    if (bUsePerTargetCooldown && Target)
    {
        float TargetDuration = CustomDuration > 0.0f ? CustomDuration : PerTargetCooldownDuration;
        TargetCooldowns.Add(Target, CurrentTime + TargetDuration);
    }
}

float UInteractionCooldownComponent::GetRemainingCooldown(AActor* Target) const
{
    float CurrentTime = GetWorld()->GetTimeSeconds();
    float RemainingGlobal = FMath::Max(0.0f, GlobalCooldownEndTime - CurrentTime);

    if (bUsePerTargetCooldown && Target)
    {
        if (const float* CooldownEndTime = TargetCooldowns.Find(Target))
        {
            float RemainingTarget = FMath::Max(0.0f, *CooldownEndTime - CurrentTime);
            return FMath::Max(RemainingGlobal, RemainingTarget);
        }
    }

    return RemainingGlobal;
}

void UInteractionCooldownComponent::CleanupExpiredCooldowns()
{
    float CurrentTime = GetWorld()->GetTimeSeconds();

    // 清理过期的记录
    for (auto It = TargetCooldowns.CreateIterator(); It; ++It)
    {
        if (!It.Key().IsValid() || CurrentTime >= It.Value())
        {
            It.RemoveCurrent();
        }
    }
}
```

---

## 17.6 实战：可拾取物品系统

### 17.6.1 完整的可拾取物品实现

让我们创建一个功能完整的可拾取物品系统，包含物品数据、拾取逻辑、UI 提示等：

```cpp
// PickupItemData.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "PickupItemData.generated.h"

/**
 * 物品类型
 */
UENUM(BlueprintType)
enum class EItemType : uint8
{
    Weapon,
    Ammo,
    Health,
    Armor,
    Key,
    Collectible
};

/**
 * UPickupItemData
 * 
 * 可拾取物品数据资产
 * 定义物品的所有静态属性
 */
UCLASS(BlueprintType)
class YOURGAME_API UPickupItemData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    /** 物品 ID */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item")
    FName ItemID;

    /** 物品名称 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item")
    FText ItemName;

    /** 物品描述 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item")
    FText ItemDescription;

    /** 物品类型 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item")
    EItemType ItemType;

    /** 物品图标 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item")
    TObjectPtr<UTexture2D> ItemIcon;

    /** 物品网格体 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item")
    TObjectPtr<UStaticMesh> ItemMesh;

    /** 物品数量 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item")
    int32 ItemCount = 1;

    /** 最大堆叠数量 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item")
    int32 MaxStackSize = 999;

    /** 拾取音效 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item|Audio")
    TObjectPtr<USoundBase> PickupSound;

    /** 拾取粒子效果 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item|VFX")
    TObjectPtr<UParticleSystem> PickupParticle;

    /** 拾取动画 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item|Animation")
    TObjectPtr<UAnimMontage> PickupMontage;

    /** 交互 Widget 类 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Item|UI")
    TSoftClassPtr<UUserWidget> InteractionWidgetClass;
};
```

```cpp
// PickupItemActor.h
#pragma once

#include "CoreMinimal.h"
#include "InteractableItemBase.h"
#include "PickupItemData.h"
#include "PickupItemActor.generated.h"

class USphereComponent;
class URotatingMovementComponent;

/**
 * APickupItemActor
 * 
 * 可拾取物品 Actor
 * 完整的实现，包含旋转、悬浮、高亮等效果
 */
UCLASS()
class YOURGAME_API APickupItemActor : public AInteractableItemBase
{
    GENERATED_BODY()
    
public:    
    APickupItemActor();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual void Tick(float DeltaTime) override;

    // ======== IInteractableTarget 接口 ========
    
    virtual void GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                          FInteractionOptionBuilder& OptionBuilder) override;

    /**
     * 设置物品数据
     */
    UFUNCTION(BlueprintCallable, Category = "Pickup")
    void SetItemData(UPickupItemData* NewItemData);

    /**
     * 获取物品数据
     */
    UFUNCTION(BlueprintPure, Category = "Pickup")
    UPickupItemData* GetItemData() const { return ItemData; }

protected:
    virtual void BeginPlay() override;
    virtual void OnInteracted_Implementation(AActor* InteractingActor, AController* InteractingController) override;

    /**
     * 应用物品数据到组件
     */
    void ApplyItemDataToComponents();

    /**
     * 播放拾取效果
     */
    void PlayPickupEffects();

protected:
    /** 物品数据 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Pickup", ReplicatedUsing = OnRep_ItemData)
    TObjectPtr<UPickupItemData> ItemData;

    /** 拾取范围 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<USphereComponent> PickupRange;

    /** 旋转组件 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<URotatingMovementComponent> RotatingMovement;

    // ======== 视觉效果配置 ========
    
    /** 是否启用悬浮效果 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Effects")
    bool bEnableFloating = true;

    /** 悬浮高度 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Effects")
    float FloatAmplitude = 20.0f;

    /** 悬浮速度 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Effects")
    float FloatFrequency = 1.0f;

    /** 是否启用旋转 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Effects")
    bool bEnableRotation = true;

    /** 旋转速度 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Effects")
    float RotationSpeed = 90.0f;

    /** 高亮颜色 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Effects")
    FLinearColor HighlightColor = FLinearColor(1.0f, 0.8f, 0.2f);

private:
    /** 初始 Z 坐标 */
    float InitialZ;

    /** 悬浮时间累积 */
    float FloatTime;

    /** 动态材质实例 */
    UPROPERTY()
    TObjectPtr<UMaterialInstanceDynamic> DynamicMaterial;

    UFUNCTION()
    void OnRep_ItemData();

    /** 处理拾取范围重叠 */
    UFUNCTION()
    void OnPickupRangeBeginOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
                                  UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, 
                                  bool bFromSweep, const FHitResult& SweepResult);
};
```

```cpp
// PickupItemActor.cpp
#include "PickupItemActor.h"
#include "Components/SphereComponent.h"
#include "Components/StaticMeshComponent.h"
#include "GameFramework/RotatingMovementComponent.h"
#include "Particles/ParticleSystemComponent.h"
#include "Kismet/GameplayStatics.h"
#include "Net/UnrealNetwork.h"

APickupItemActor::APickupItemActor()
{
    PrimaryActorTick.bCanEverTick = true;

    // 创建拾取范围
    PickupRange = CreateDefaultSubobject<USphereComponent>(TEXT("PickupRange"));
    PickupRange->SetupAttachment(RootComponent);
    PickupRange->SetSphereRadius(150.0f);
    PickupRange->SetCollisionProfileName(TEXT("OverlapAllDynamic"));

    // 创建旋转组件
    RotatingMovement = CreateDefaultSubobject<URotatingMovementComponent>(TEXT("RotatingMovement"));
    RotatingMovement->RotationRate = FRotator(0.0f, RotationSpeed, 0.0f);

    // 默认交互文本
    InteractionText = NSLOCTEXT("Pickup", "DefaultPickup", "拾取物品");
}

void APickupItemActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(APickupItemActor, ItemData);
}

void APickupItemActor::BeginPlay()
{
    Super::BeginPlay();

    // 记录初始位置
    InitialZ = GetActorLocation().Z;
    FloatTime = 0.0f;

    // 应用物品数据
    ApplyItemDataToComponents();

    // 绑定重叠事件
    if (PickupRange)
    {
        PickupRange->OnComponentBeginOverlap.AddDynamic(this, &APickupItemActor::OnPickupRangeBeginOverlap);
    }

    // 更新旋转速度
    if (RotatingMovement)
    {
        RotatingMovement->RotationRate = FRotator(0.0f, RotationSpeed, 0.0f);
        RotatingMovement->SetActive(bEnableRotation);
    }

    // 创建动态材质
    if (MeshComponent)
    {
        UMaterialInterface* Material = MeshComponent->GetMaterial(0);
        if (Material)
        {
            DynamicMaterial = UMaterialInstanceDynamic::Create(Material, this);
            MeshComponent->SetMaterial(0, DynamicMaterial);
        }
    }
}

void APickupItemActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 悬浮效果
    if (bEnableFloating)
    {
        FloatTime += DeltaTime * FloatFrequency;
        float NewZ = InitialZ + FMath::Sin(FloatTime) * FloatAmplitude;
        
        FVector NewLocation = GetActorLocation();
        NewLocation.Z = NewZ;
        SetActorLocation(NewLocation);
    }

    // 高亮效果（可选）
    if (DynamicMaterial)
    {
        float Glow = (FMath::Sin(FloatTime * 2.0f) + 1.0f) * 0.5f;
        DynamicMaterial->SetScalarParameterValue(FName("EmissiveStrength"), Glow);
    }
}

void APickupItemActor::GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                               FInteractionOptionBuilder& OptionBuilder)
{
    if (!bIsInteractable || !ItemData)
    {
        return;
    }

    FInteractionOption Option;
    
    // 使用物品数据的文本
    Option.Text = FText::Format(
        NSLOCTEXT("Pickup", "PickupFormat", "拾取 {0}"),
        ItemData->ItemName
    );

    // 显示数量
    if (ItemData->ItemCount > 1)
    {
        Option.SubText = FText::Format(
            NSLOCTEXT("Pickup", "CountFormat", "x{0}"),
            FText::AsNumber(ItemData->ItemCount)
        );
    }

    // 使用物品数据的 Widget 类
    if (!ItemData->InteractionWidgetClass.IsNull())
    {
        Option.InteractionWidgetClass = ItemData->InteractionWidgetClass;
    }

    // 使用目标的能力系统
    Option.TargetAbilitySystem = AbilitySystemComponent;
    Option.TargetInteractionAbilityHandle = InteractionAbilityHandle;

    OptionBuilder.AddInteractionOption(Option);
}

void APickupItemActor::SetItemData(UPickupItemData* NewItemData)
{
    ItemData = NewItemData;
    ApplyItemDataToComponents();
}

void APickupItemActor::ApplyItemDataToComponents()
{
    if (!ItemData)
    {
        return;
    }

    // 应用网格体
    if (MeshComponent && ItemData->ItemMesh)
    {
        MeshComponent->SetStaticMesh(ItemData->ItemMesh);
    }

    // 更新交互文本
    InteractionText = FText::Format(
        NSLOCTEXT("Pickup", "PickupFormat", "拾取 {0}"),
        ItemData->ItemName
    );
    InteractionSubText = ItemData->ItemDescription;
}

void APickupItemActor::OnInteracted_Implementation(AActor* InteractingActor, AController* InteractingController)
{
    if (!HasAuthority())
    {
        return;
    }

    // ======== 服务器端拾取逻辑 ========

    // 1. 添加到背包（这里需要你的背包系统）
    // if (UInventoryComponent* Inventory = InteractingActor->FindComponentByClass<UInventoryComponent>())
    // {
    //     bool bSuccess = Inventory->AddItem(ItemData, ItemData->ItemCount);
    //     if (!bSuccess)
    //     {
    //         // 背包满了
    //         return;
    //     }
    // }

    // 2. 播放效果
    PlayPickupEffects();

    // 3. 通知所有客户端
    Multicast_OnInteracted(InteractingActor);

    // 4. 销毁物品
    SetLifeSpan(0.5f);
    bIsInteractable = false;
    SetActorHiddenInGame(true);
    SetActorEnableCollision(false);
}

void APickupItemActor::PlayPickupEffects()
{
    if (!ItemData)
    {
        return;
    }

    // 播放音效
    if (ItemData->PickupSound)
    {
        UGameplayStatics::PlaySoundAtLocation(this, ItemData->PickupSound, GetActorLocation());
    }

    // 播放粒子效果
    if (ItemData->PickupParticle)
    {
        UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ItemData->PickupParticle, GetActorLocation());
    }
}

void APickupItemActor::OnRep_ItemData()
{
    ApplyItemDataToComponents();
}

void APickupItemActor::OnPickupRangeBeginOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
                                                UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, 
                                                bool bFromSweep, const FHitResult& SweepResult)
{
    // 可以在这里实现自动拾取逻辑
    // 或者显示特殊的 UI 提示
}
```

### 17.6.2 物品生成器

创建一个物品生成器，方便在关卡中放置物品：

```cpp
// PickupItemSpawner.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "PickupItemSpawner.generated.h"

class APickupItemActor;
class UPickupItemData;

/**
 * APickupItemSpawner
 * 
 * 物品生成器
 * 在关卡中放置，可以定时生成物品
 */
UCLASS()
class YOURGAME_API APickupItemSpawner : public AActor
{
    GENERATED_BODY()
    
public:    
    APickupItemSpawner();

    virtual void BeginPlay() override;

    /**
     * 生成物品
     */
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    void SpawnItem();

    /**
     * 开始自动生成
     */
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    void StartAutoSpawn();

    /**
     * 停止自动生成
     */
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    void StopAutoSpawn();

protected:
    /**
     * 物品被拾取时调用
     */
    UFUNCTION()
    void OnItemPickedUp(AActor* PickedItem);

    /**
     * 检查是否可以生成
     */
    bool CanSpawn() const;

protected:
    /** 生成的物品类 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawner")
    TSubclassOf<APickupItemActor> ItemClass;

    /** 物品数据列表（随机选择一个） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawner")
    TArray<TObjectPtr<UPickupItemData>> ItemDataArray;

    /** 是否自动生成 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawner")
    bool bAutoSpawn = true;

    /** 生成间隔 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawner")
    float SpawnInterval = 30.0f;

    /** 初始延迟 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawner")
    float InitialDelay = 0.0f;

    /** 最大同时存在数量 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawner")
    int32 MaxActiveItems = 1;

    /** 生成位置随机偏移 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawner")
    FVector SpawnRandomOffset = FVector(50.0f, 50.0f, 0.0f);

    /** 显示调试信息 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawner|Debug")
    bool bShowDebug = true;

private:
    /** 当前生成的物品列表 */
    UPROPERTY()
    TArray<TObjectPtr<APickupItemActor>> ActiveItems;

    /** 生成定时器 */
    FTimerHandle SpawnTimerHandle;
};
```

```cpp
// PickupItemSpawner.cpp
#include "PickupItemSpawner.h"
#include "PickupItemActor.h"
#include "PickupItemData.h"
#include "TimerManager.h"
#include "Engine/World.h"
#include "DrawDebugHelpers.h"

APickupItemSpawner::APickupItemSpawner()
{
    PrimaryActorTick.bCanEverTick = false;
    bReplicates = true;

    // 创建根组件（用于可视化）
    USceneComponent* SceneRoot = CreateDefaultSubobject<USceneComponent>(TEXT("SceneRoot"));
    RootComponent = SceneRoot;
}

void APickupItemSpawner::BeginPlay()
{
    Super::BeginPlay();

    // 仅在服务器上生成
    if (HasAuthority())
    {
        if (bAutoSpawn)
        {
            StartAutoSpawn();
        }
    }
}

void APickupItemSpawner::SpawnItem()
{
    if (!CanSpawn())
    {
        return;
    }

    if (!ItemClass || ItemDataArray.Num() == 0)
    {
        UE_LOG(LogTemp, Warning, TEXT("PickupItemSpawner: ItemClass or ItemDataArray not set!"));
        return;
    }

    // 随机选择物品数据
    int32 RandomIndex = FMath::RandRange(0, ItemDataArray.Num() - 1);
    UPickupItemData* SelectedItemData = ItemDataArray[RandomIndex];

    if (!SelectedItemData)
    {
        return;
    }

    // 计算生成位置
    FVector SpawnLocation = GetActorLocation();
    if (!SpawnRandomOffset.IsNearlyZero())
    {
        SpawnLocation += FVector(
            FMath::RandRange(-SpawnRandomOffset.X, SpawnRandomOffset.X),
            FMath::RandRange(-SpawnRandomOffset.Y, SpawnRandomOffset.Y),
            FMath::RandRange(-SpawnRandomOffset.Z, SpawnRandomOffset.Z)
        );
    }

    // 生成物品
    FActorSpawnParameters SpawnParams;
    SpawnParams.Owner = this;
    SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn;

    APickupItemActor* SpawnedItem = GetWorld()->SpawnActor<APickupItemActor>(
        ItemClass,
        SpawnLocation,
        GetActorRotation(),
        SpawnParams
    );

    if (SpawnedItem)
    {
        // 设置物品数据
        SpawnedItem->SetItemData(SelectedItemData);

        // 添加到活动列表
        ActiveItems.Add(SpawnedItem);

        // 绑定销毁事件
        SpawnedItem->OnDestroyed.AddDynamic(this, &APickupItemSpawner::OnItemPickedUp);

        UE_LOG(LogTemp, Log, TEXT("PickupItemSpawner: Spawned %s at %s"), 
               *SelectedItemData->ItemName.ToString(), *SpawnLocation.ToString());
    }
}

void APickupItemSpawner::StartAutoSpawn()
{
    if (!HasAuthority())
    {
        return;
    }

    // 清除旧的定时器
    GetWorld()->GetTimerManager().ClearTimer(SpawnTimerHandle);

    // 设置新的定时器
    GetWorld()->GetTimerManager().SetTimer(
        SpawnTimerHandle,
        this,
        &APickupItemSpawner::SpawnItem,
        SpawnInterval,
        true,  // Loop
        InitialDelay
    );
}

void APickupItemSpawner::StopAutoSpawn()
{
    GetWorld()->GetTimerManager().ClearTimer(SpawnTimerHandle);
}

void APickupItemSpawner::OnItemPickedUp(AActor* PickedItem)
{
    // 从活动列表中移除
    ActiveItems.Remove(Cast<APickupItemActor>(PickedItem));
}

bool APickupItemSpawner::CanSpawn() const
{
    // 清理无效引用
    for (int32 i = ActiveItems.Num() - 1; i >= 0; --i)
    {
        if (!ActiveItems[i] || !IsValid(ActiveItems[i]))
        {
            const_cast<APickupItemSpawner*>(this)->ActiveItems.RemoveAt(i);
        }
    }

    // 检查数量限制
    return ActiveItems.Num() < MaxActiveItems;
}
```

---

## 17.7 实战：可交互门系统

### 17.7.1 门系统实现

创建一个完整的可交互门系统，支持锁定、权限检查、动画等功能：

```cpp
// InteractableDoor.h
#pragma once

#include "CoreMinimal.h"
#include "InteractableItemBase.h"
#include "InteractableDoor.generated.h"

class UTimelineComponent;
class UCurveFloat;
class UAudioComponent;

/**
 * 门状态
 */
UENUM(BlueprintType)
enum class EDoorState : uint8
{
    Closed,
    Opening,
    Open,
    Closing
};

/**
 * AInteractableDoor
 * 
 * 可交互门 Actor
 * 支持开关、锁定、权限检查等功能
 */
UCLASS()
class YOURGAME_API AInteractableDoor : public AInteractableItemBase
{
    GENERATED_BODY()
    
public:    
    AInteractableDoor();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual void Tick(float DeltaTime) override;

    // ======== IInteractableTarget 接口 ========
    
    virtual void GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                          FInteractionOptionBuilder& OptionBuilder) override;

    /**
     * 打开门
     */
    UFUNCTION(BlueprintCallable, Category = "Door")
    void OpenDoor(AActor* Opener = nullptr);

    /**
     * 关闭门
     */
    UFUNCTION(BlueprintCallable, Category = "Door")
    void CloseDoor();

    /**
     * 切换门状态
     */
    UFUNCTION(BlueprintCallable, Category = "Door")
    void ToggleDoor(AActor* Opener = nullptr);

    /**
     * 锁定门
     */
    UFUNCTION(BlueprintCallable, Category = "Door")
    void LockDoor();

    /**
     * 解锁门
     */
    UFUNCTION(BlueprintCallable, Category = "Door")
    void UnlockDoor();

    /**
     * 检查是否有权限打开
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Door")
    bool HasPermissionToOpen(AActor* Opener) const;
    virtual bool HasPermissionToOpen_Implementation(AActor* Opener) const;

protected:
    virtual void BeginPlay() override;
    virtual void OnInteracted_Implementation(AActor* InteractingActor, AController* InteractingController) override;

    /**
     * 更新门旋转
     */
    UFUNCTION()
    void UpdateDoorRotation(float Alpha);

    /**
     * 门动画完成回调
     */
    UFUNCTION()
    void OnDoorAnimationFinished();

    /**
     * 播放门音效
     */
    void PlayDoorSound(bool bOpening);

protected:
    /** 门框网格体 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UStaticMeshComponent> DoorFrameMesh;

    /** 门板网格体 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UStaticMeshComponent> DoorMesh;

    /** 音频组件 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UAudioComponent> AudioComponent;

    /** Timeline 组件 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UTimelineComponent> DoorTimeline;

    // ======== 门状态 ========
    
    /** 当前门状态 */
    UPROPERTY(ReplicatedUsing = OnRep_DoorState, BlueprintReadOnly, Category = "Door")
    EDoorState DoorState = EDoorState::Closed;

    /** 是否锁定 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door", ReplicatedUsing = OnRep_bIsLocked)
    bool bIsLocked = false;

    /** 需要的钥匙 ID */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door")
    FName RequiredKeyID;

    // ======== 门配置 ========
    
    /** 开门角度 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door|Config")
    float OpenAngle = 90.0f;

    /** 开门时间 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door|Config")
    float OpenDuration = 1.0f;

    /** 是否自动关闭 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door|Config")
    bool bAutoClose = false;

    /** 自动关闭延迟 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door|Config", meta = (EditCondition = "bAutoClose"))
    float AutoCloseDelay = 3.0f;

    /** 动画曲线 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door|Config")
    TObjectPtr<UCurveFloat> DoorCurve;

    // ======== 音效 ========
    
    /** 开门音效 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door|Audio")
    TObjectPtr<USoundBase> DoorOpenSound;

    /** 关门音效 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door|Audio")
    TObjectPtr<USoundBase> DoorCloseSound;

    /** 锁定音效 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door|Audio")
    TObjectPtr<USoundBase> DoorLockedSound;

private:
    /** 初始旋转 */
    FRotator InitialRotation;

    /** 目标旋转 */
    FRotator TargetRotation;

    /** 自动关闭定时器 */
    FTimerHandle AutoCloseTimerHandle;

    UFUNCTION()
    void OnRep_DoorState();

    UFUNCTION()
    void OnRep_bIsLocked();

    /** 服务器 RPC */
    UFUNCTION(Server, Reliable)
    void ServerOpenDoor(AActor* Opener);

    UFUNCTION(Server, Reliable)
    void ServerCloseDoor();
};
```

完整实现：

```cpp
// InteractableDoor.cpp
#include "InteractableDoor.h"
#include "Components/StaticMeshComponent.h"
#include "Components/AudioComponent.h"
#include "Components/TimelineComponent.h"
#include "Curves/CurveFloat.h"
#include "Kismet/GameplayStatics.h"
#include "Net/UnrealNetwork.h"
#include "TimerManager.h"

AInteractableDoor::AInteractableDoor()
{
    PrimaryActorTick.bCanEverTick = true;

    // 创建门框
    DoorFrameMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("DoorFrameMesh"));
    RootComponent = DoorFrameMesh;
    DoorFrameMesh->SetCollisionProfileName(TEXT("BlockAll"));

    // 创建门板
    DoorMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("DoorMesh"));
    DoorMesh->SetupAttachment(DoorFrameMesh);
    DoorMesh->SetCollisionProfileName(TEXT("BlockAll"));

    // 创建音频组件
    AudioComponent = CreateDefaultSubobject<UAudioComponent>(TEXT("AudioComponent"));
    AudioComponent->SetupAttachment(DoorMesh);
    AudioComponent->bAutoActivate = false;

    // 创建 Timeline
    DoorTimeline = CreateDefaultSubobject<UTimelineComponent>(TEXT("DoorTimeline"));

    // 默认交互文本
    InteractionText = NSLOCTEXT("Door", "OpenDoor", "开门");
}

void AInteractableDoor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AInteractableDoor, DoorState);
    DOREPLIFETIME(AInteractableDoor, bIsLocked);
}

void AInteractableDoor::BeginPlay()
{
    Super::BeginPlay();

    // 记录初始旋转
    InitialRotation = DoorMesh->GetRelativeRotation();

    // 设置 Timeline
    if (DoorCurve)
    {
        FOnTimelineFloat TimelineProgress;
        TimelineProgress.BindUFunction(this, FName("UpdateDoorRotation"));
        DoorTimeline->AddInterpFloat(DoorCurve, TimelineProgress);

        FOnTimelineEvent TimelineFinished;
        TimelineFinished.BindUFunction(this, FName("OnDoorAnimationFinished"));
        DoorTimeline->SetTimelineFinishedFunc(TimelineFinished);

        DoorTimeline->SetTimelineLength(OpenDuration);
    }
}

void AInteractableDoor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
}

void AInteractableDoor::GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                                FInteractionOptionBuilder& OptionBuilder)
{
    if (!bIsInteractable)
    {
        return;
    }

    AActor* Requester = InteractQuery.RequestingAvatar.Get();

    FInteractionOption Option;
    Option.TargetAbilitySystem = AbilitySystemComponent;
    Option.TargetInteractionAbilityHandle = InteractionAbilityHandle;

    // 根据状态设置文本
    if (bIsLocked)
    {
        Option.Text = NSLOCTEXT("Door", "Locked", "已锁定");
        
        if (!RequiredKeyID.IsNone())
        {
            Option.SubText = FText::Format(
                NSLOCTEXT("Door", "RequiresKey", "需要钥匙：{0}"),
                FText::FromName(RequiredKeyID)
            );
        }

        // 检查是否有钥匙
        if (HasPermissionToOpen(Requester))
        {
            Option.Text = NSLOCTEXT("Door", "Unlock", "解锁并打开");
        }
    }
    else
    {
        switch (DoorState)
        {
        case EDoorState::Closed:
            Option.Text = NSLOCTEXT("Door", "Open", "打开");
            break;
        case EDoorState::Open:
            Option.Text = NSLOCTEXT("Door", "Close", "关闭");
            break;
        case EDoorState::Opening:
        case EDoorState::Closing:
            Option.Text = NSLOCTEXT("Door", "Wait", "请稍等...");
            // 不返回选项，门正在移动中
            return;
        }
    }

    OptionBuilder.AddInteractionOption(Option);
}

void AInteractableDoor::OnInteracted_Implementation(AActor* InteractingActor, AController* InteractingController)
{
    if (!HasAuthority())
    {
        return;
    }

    // 检查权限
    if (bIsLocked && !HasPermissionToOpen(InteractingActor))
    {
        // 播放锁定音效
        if (DoorLockedSound)
        {
            UGameplayStatics::PlaySoundAtLocation(this, DoorLockedSound, GetActorLocation());
        }
        return;
    }

    // 如果锁定但有权限，先解锁
    if (bIsLocked && HasPermissionToOpen(InteractingActor))
    {
        UnlockDoor();
    }

    // 切换门状态
    ToggleDoor(InteractingActor);
}

void AInteractableDoor::OpenDoor(AActor* Opener)
{
    if (HasAuthority())
    {
        ServerOpenDoor(Opener);
    }
    else
    {
        ServerOpenDoor(Opener);
    }
}

void AInteractableDoor::ServerOpenDoor_Implementation(AActor* Opener)
{
    if (DoorState == EDoorState::Open || DoorState == EDoorState::Opening)
    {
        return;
    }

    if (bIsLocked && !HasPermissionToOpen(Opener))
    {
        return;
    }

    // 更新状态
    DoorState = EDoorState::Opening;

    // 计算目标旋转
    TargetRotation = InitialRotation + FRotator(0.0f, OpenAngle, 0.0f);

    // 播放动画
    if (DoorTimeline)
    {
        DoorTimeline->PlayFromStart();
    }

    // 播放音效
    PlayDoorSound(true);

    // 设置自动关闭
    if (bAutoClose)
    {
        GetWorld()->GetTimerManager().SetTimer(
            AutoCloseTimerHandle,
            this,
            &AInteractableDoor::CloseDoor,
            OpenDuration + AutoCloseDelay,
            false
        );
    }
}

void AInteractableDoor::CloseDoor()
{
    if (HasAuthority())
    {
        ServerCloseDoor();
    }
    else
    {
        ServerCloseDoor();
    }
}

void AInteractableDoor::ServerCloseDoor_Implementation()
{
    if (DoorState == EDoorState::Closed || DoorState == EDoorState::Closing)
    {
        return;
    }

    // 清除自动关闭定时器
    GetWorld()->GetTimerManager().ClearTimer(AutoCloseTimerHandle);

    // 更新状态
    DoorState = EDoorState::Closing;

    // 目标旋转为初始旋转
    TargetRotation = InitialRotation;

    // 反向播放动画
    if (DoorTimeline)
    {
        DoorTimeline->ReverseFromEnd();
    }

    // 播放音效
    PlayDoorSound(false);
}

void AInteractableDoor::ToggleDoor(AActor* Opener)
{
    if (DoorState == EDoorState::Closed || DoorState == EDoorState::Closing)
    {
        OpenDoor(Opener);
    }
    else if (DoorState == EDoorState::Open || DoorState == EDoorState::Opening)
    {
        CloseDoor();
    }
}

void AInteractableDoor::LockDoor()
{
    bIsLocked = true;
    OnRep_bIsLocked();
}

void AInteractableDoor::UnlockDoor()
{
    bIsLocked = false;
    OnRep_bIsLocked();
}

bool AInteractableDoor::HasPermissionToOpen_Implementation(AActor* Opener) const
{
    if (!bIsLocked)
    {
        return true;
    }

    if (RequiredKeyID.IsNone())
    {
        return false;  // 锁定但没有指定钥匙，无法打开
    }

    // 检查玩家是否有钥匙（这里需要你的背包系统）
    // if (UInventoryComponent* Inventory = Opener->FindComponentByClass<UInventoryComponent>())
    // {
    //     return Inventory->HasItem(RequiredKeyID);
    // }

    return false;
}

void AInteractableDoor::UpdateDoorRotation(float Alpha)
{
    if (DoorMesh)
    {
        FRotator NewRotation = FMath::Lerp(InitialRotation, TargetRotation, Alpha);
        DoorMesh->SetRelativeRotation(NewRotation);
    }
}

void AInteractableDoor::OnDoorAnimationFinished()
{
    if (DoorState == EDoorState::Opening)
    {
        DoorState = EDoorState::Open;
    }
    else if (DoorState == EDoorState::Closing)
    {
        DoorState = EDoorState::Closed;
    }

    OnRep_DoorState();
}

void AInteractableDoor::PlayDoorSound(bool bOpening)
{
    USoundBase* SoundToPlay = bOpening ? DoorOpenSound : DoorCloseSound;
    
    if (SoundToPlay && AudioComponent)
    {
        AudioComponent->SetSound(SoundToPlay);
        AudioComponent->Play();
    }
}

void AInteractableDoor::OnRep_DoorState()
{
    // 客户端状态变化的视觉反馈
    // 可以在这里触发粒子效果、额外音效等
}

void AInteractableDoor::OnRep_bIsLocked()
{
    // 更新材质或视觉效果，表示锁定状态
}
```

---

## 17.8 实战：NPC 对话触发

### 17.8.1 对话系统接口

创建一个简单但完整的 NPC 对话系统：

```cpp
// DialogueData.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "DialogueData.generated.h"

/**
 * 对话节点
 */
USTRUCT(BlueprintType)
struct FDialogueNode
{
    GENERATED_BODY()

    /** 节点 ID */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FName NodeID;

    /** NPC 说的话 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FText NPCText;

    /** 玩家选项列表 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FText> PlayerOptions;

    /** 每个选项跳转到的节点 ID */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FName> NextNodeIDs;

    /** 是否是结束节点 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bIsEndNode = false;

    /** 对话音效 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<USoundBase> DialogueSound;
};

/**
 * UDialogueData
 * 
 * 对话数据资产
 */
UCLASS(BlueprintType)
class YOURGAME_API UDialogueData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    /** 对话 ID */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Dialogue")
    FName DialogueID;

    /** 对话标题 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Dialogue")
    FText DialogueTitle;

    /** 起始节点 ID */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Dialogue")
    FName StartNodeID;

    /** 所有对话节点 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Dialogue")
    TArray<FDialogueNode> DialogueNodes;

    /**
     * 根据 ID 获取节点
     */
    UFUNCTION(BlueprintPure, Category = "Dialogue")
    bool GetDialogueNode(FName NodeID, FDialogueNode& OutNode) const;
};
```

```cpp
// DialogueData.cpp
#include "DialogueData.h"

bool UDialogueData::GetDialogueNode(FName NodeID, FDialogueNode& OutNode) const
{
    for (const FDialogueNode& Node : DialogueNodes)
    {
        if (Node.NodeID == NodeID)
        {
            OutNode = Node;
            return true;
        }
    }

    return false;
}
```

### 17.8.2 NPC 角色实现

```cpp
// InteractableNPC.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "Interaction/IInteractableTarget.h"
#include "DialogueData.h"
#include "InteractableNPC.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnDialogueStarted, AInteractableNPC*, NPC, UDialogueData*, DialogueData);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnDialogueEnded, AInteractableNPC*, NPC);

/**
 * AInteractableNPC
 * 
 * 可交互的 NPC 角色
 * 支持对话系统
 */
UCLASS()
class YOURGAME_API AInteractableNPC : public ACharacter, public IInteractableTarget
{
    GENERATED_BODY()

public:
    AInteractableNPC();

    // ======== IInteractableTarget 接口 ========
    
    virtual void GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                          FInteractionOptionBuilder& OptionBuilder) override;

    virtual void CustomizeInteractionEventData(const FGameplayTag& InteractionEventTag, 
                                               FGameplayEventData& InOutEventData) override;

    /**
     * 开始对话
     */
    UFUNCTION(BlueprintCallable, Category = "Dialogue")
    void StartDialogue(AActor* InteractingActor);

    /**
     * 结束对话
     */
    UFUNCTION(BlueprintCallable, Category = "Dialogue")
    void EndDialogue();

    /**
     * 获取当前对话节点
     */
    UFUNCTION(BlueprintPure, Category = "Dialogue")
    bool GetCurrentDialogueNode(FDialogueNode& OutNode) const;

    /**
     * 选择对话选项
     */
    UFUNCTION(BlueprintCallable, Category = "Dialogue")
    void SelectDialogueOption(int32 OptionIndex);

public:
    /** 对话开始事件 */
    UPROPERTY(BlueprintAssignable, Category = "Dialogue")
    FOnDialogueStarted OnDialogueStarted;

    /** 对话结束事件 */
    UPROPERTY(BlueprintAssignable, Category = "Dialogue")
    FOnDialogueEnded OnDialogueEnded;

protected:
    virtual void BeginPlay() override;

    /**
     * 朝向玩家
     */
    void LookAtActor(AActor* TargetActor);

protected:
    /** 对话数据 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Dialogue")
    TObjectPtr<UDialogueData> DialogueData;

    /** NPC 名称 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "NPC")
    FText NPCName;

    /** 交互提示文本 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    FText InteractionPrompt;

    /** 是否可以对话 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Dialogue")
    bool bCanTalk = true;

    /** 对话中是否朝向玩家 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Dialogue")
    bool bLookAtPlayerDuringDialogue = true;

    /** 交互能力类 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    TSubclassOf<UGameplayAbility> InteractionAbilityClass;

private:
    /** 当前对话的玩家 */
    UPROPERTY()
    TObjectPtr<AActor> CurrentInteractingActor;

    /** 当前节点 ID */
    FName CurrentNodeID;

    /** 是否正在对话中 */
    bool bIsInDialogue = false;

    /** 能力系统组件 */
    UPROPERTY()
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    /** 交互能力句柄 */
    FGameplayAbilitySpecHandle InteractionAbilityHandle;
};
```

```cpp
// InteractableNPC.cpp
#include "InteractableNPC.h"
#include "AbilitySystemComponent.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Kismet/KismetMathLibrary.h"

AInteractableNPC::AInteractableNPC()
{
    PrimaryActorTick.bCanEverTick = true;

    // 创建能力系统组件
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);

    // 默认交互提示
    InteractionPrompt = NSLOCTEXT("NPC", "TalkTo", "对话");
}

void AInteractableNPC::BeginPlay()
{
    Super::BeginPlay();

    // 授予交互能力
    if (HasAuthority() && InteractionAbilityClass && AbilitySystemComponent)
    {
        FGameplayAbilitySpec AbilitySpec(InteractionAbilityClass, 1, INDEX_NONE, this);
        InteractionAbilityHandle = AbilitySystemComponent->GiveAbility(AbilitySpec);
    }
}

void AInteractableNPC::GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                               FInteractionOptionBuilder& OptionBuilder)
{
    if (!bCanTalk || bIsInDialogue)
    {
        return;
    }

    FInteractionOption Option;
    
    // 设置文本
    if (NPCName.IsEmpty())
    {
        Option.Text = InteractionPrompt;
    }
    else
    {
        Option.Text = FText::Format(
            NSLOCTEXT("NPC", "TalkToFormat", "与 {0} {1}"),
            NPCName,
            InteractionPrompt
        );
    }

    // 使用 NPC 的能力系统
    Option.TargetAbilitySystem = AbilitySystemComponent;
    Option.TargetInteractionAbilityHandle = InteractionAbilityHandle;

    OptionBuilder.AddInteractionOption(Option);
}

void AInteractableNPC::CustomizeInteractionEventData(const FGameplayTag& InteractionEventTag, 
                                                     FGameplayEventData& InOutEventData)
{
    InOutEventData.OptionalObject = this;
}

void AInteractableNPC::StartDialogue(AActor* InteractingActor)
{
    if (!DialogueData || bIsInDialogue)
    {
        return;
    }

    CurrentInteractingActor = InteractingActor;
    CurrentNodeID = DialogueData->StartNodeID;
    bIsInDialogue = true;

    // 朝向玩家
    if (bLookAtPlayerDuringDialogue && InteractingActor)
    {
        LookAtActor(InteractingActor);
    }

    // 停止移动
    if (UCharacterMovementComponent* Movement = GetCharacterMovement())
    {
        Movement->StopMovementImmediately();
    }

    // 触发事件
    OnDialogueStarted.Broadcast(this, DialogueData);
}

void AInteractableNPC::EndDialogue()
{
    if (!bIsInDialogue)
    {
        return;
    }

    bIsInDialogue = false;
    CurrentInteractingActor = nullptr;
    CurrentNodeID = NAME_None;

    // 触发事件
    OnDialogueEnded.Broadcast(this);
}

bool AInteractableNPC::GetCurrentDialogueNode(FDialogueNode& OutNode) const
{
    if (!bIsInDialogue || !DialogueData)
    {
        return false;
    }

    return DialogueData->GetDialogueNode(CurrentNodeID, OutNode);
}

void AInteractableNPC::SelectDialogueOption(int32 OptionIndex)
{
    if (!bIsInDialogue || !DialogueData)
    {
        return;
    }

    FDialogueNode CurrentNode;
    if (!DialogueData->GetDialogueNode(CurrentNodeID, CurrentNode))
    {
        return;
    }

    // 检查选项索引是否有效
    if (!CurrentNode.NextNodeIDs.IsValidIndex(OptionIndex))
    {
        return;
    }

    // 跳转到下一个节点
    FName NextNodeID = CurrentNode.NextNodeIDs[OptionIndex];
    
    FDialogueNode NextNode;
    if (DialogueData->GetDialogueNode(NextNodeID, NextNode))
    {
        CurrentNodeID = NextNodeID;

        // 播放音效
        if (NextNode.DialogueSound)
        {
            // UGameplayStatics::PlaySoundAtLocation...
        }

        // 如果是结束节点，结束对话
        if (NextNode.bIsEndNode)
        {
            EndDialogue();
        }
    }
}

void AInteractableNPC::LookAtActor(AActor* TargetActor)
{
    if (!TargetActor)
    {
        return;
    }

    FVector Direction = TargetActor->GetActorLocation() - GetActorLocation();
    Direction.Z = 0.0f;
    
    FRotator TargetRotation = Direction.Rotation();
    SetActorRotation(TargetRotation);
}
```

### 17.8.3 对话 UI Widget

```cpp
// DialogueWidget.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "DialogueData.h"
#include "DialogueWidget.generated.h"

class UTextBlock;
class UVerticalBox;
class UButton;

/**
 * UDialogueWidget
 * 
 * 对话界面 Widget
 */
UCLASS()
class YOURGAME_API UDialogueWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override;

    /**
     * 显示对话节点
     */
    UFUNCTION(BlueprintCallable, Category = "Dialogue")
    void ShowDialogueNode(const FDialogueNode& Node, const FText& NPCName);

    /**
     * 设置当前 NPC
     */
    UFUNCTION(BlueprintCallable, Category = "Dialogue")
    void SetCurrentNPC(class AInteractableNPC* NPC);

protected:
    /**
     * 选项按钮点击回调
     */
    UFUNCTION()
    void OnOptionButtonClicked(int32 OptionIndex);

    /**
     * 创建选项按钮
     */
    UButton* CreateOptionButton(const FText& OptionText, int32 OptionIndex);

protected:
    /** NPC 名称文本 */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> NPCNameText;

    /** 对话文本 */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> DialogueText;

    /** 选项容器 */
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UVerticalBox> OptionsContainer;

    /** 选项按钮类 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Dialogue")
    TSubclassOf<UButton> OptionButtonClass;

private:
    /** 当前 NPC */
    UPROPERTY()
    TObjectPtr<AInteractableNPC> CurrentNPC;

    /** 动态创建的按钮列表 */
    UPROPERTY()
    TArray<TObjectPtr<UButton>> OptionButtons;
};
```

```cpp
// DialogueWidget.cpp
#include "DialogueWidget.h"
#include "Components/TextBlock.h"
#include "Components/VerticalBox.h"
#include "Components/Button.h"
#include "InteractableNPC.h"

void UDialogueWidget::NativeConstruct()
{
    Super::NativeConstruct();
}

void UDialogueWidget::ShowDialogueNode(const FDialogueNode& Node, const FText& NPCName)
{
    // 设置 NPC 名称
    if (NPCNameText)
    {
        NPCNameText->SetText(NPCName);
    }

    // 设置对话文本
    if (DialogueText)
    {
        DialogueText->SetText(Node.NPCText);
    }

    // 清除旧选项
    if (OptionsContainer)
    {
        OptionsContainer->ClearChildren();
    }
    OptionButtons.Reset();

    // 如果是结束节点，显示"关闭"按钮
    if (Node.bIsEndNode)
    {
        UButton* CloseButton = CreateOptionButton(
            NSLOCTEXT("Dialogue", "Close", "关闭"),
            -1
        );
        
        if (CloseButton && OptionsContainer)
        {
            OptionsContainer->AddChild(CloseButton);
        }
    }
    else
    {
        // 创建选项按钮
        for (int32 i = 0; i < Node.PlayerOptions.Num(); ++i)
        {
            UButton* OptionButton = CreateOptionButton(Node.PlayerOptions[i], i);
            
            if (OptionButton && OptionsContainer)
            {
                OptionsContainer->AddChild(OptionButton);
                OptionButtons.Add(OptionButton);
            }
        }
    }
}

void UDialogueWidget::SetCurrentNPC(AInteractableNPC* NPC)
{
    CurrentNPC = NPC;
}

void UDialogueWidget::OnOptionButtonClicked(int32 OptionIndex)
{
    if (!CurrentNPC)
    {
        return;
    }

    if (OptionIndex == -1)
    {
        // 关闭按钮
        CurrentNPC->EndDialogue();
        RemoveFromParent();
    }
    else
    {
        // 选择选项
        CurrentNPC->SelectDialogueOption(OptionIndex);

        // 更新显示
        FDialogueNode CurrentNode;
        if (CurrentNPC->GetCurrentDialogueNode(CurrentNode))
        {
            // 获取 NPC 名称
            FText NPCName = NSLOCTEXT("Dialogue", "Unknown", "未知");
            // ShowDialogueNode(CurrentNode, NPCName);
        }
    }
}

UButton* UDialogueWidget::CreateOptionButton(const FText& OptionText, int32 OptionIndex)
{
    if (!OptionButtonClass)
    {
        return nullptr;
    }

    UButton* Button = CreateWidget<UButton>(GetOwningPlayer(), OptionButtonClass);
    if (Button)
    {
        // 设置按钮文本
        // Button->SetText(OptionText);

        // 绑定点击事件
        Button->OnClicked.AddDynamic(this, &UDialogueWidget::OnOptionButtonClicked);
        // 传递 OptionIndex（需要使用 Lambda 或自定义数据）
    }

    return Button;
}
```

---

## 17.9 进阶：多人交互与条件判断

### 17.9.1 互斥访问实现

实现一个只允许一个玩家同时交互的对象：

```cpp
// ExclusiveInteractableActor.h
#pragma once

#include "CoreMinimal.h"
#include "InteractableItemBase.h"
#include "ExclusiveInteractableActor.generated.h"

/**
 * AExclusiveInteractableActor
 * 
 * 互斥交互对象
 * 同一时间只允许一个玩家交互
 */
UCLASS()
class YOURGAME_API AExclusiveInteractableActor : public AInteractableItemBase
{
    GENERATED_BODY()
    
public:    
    AExclusiveInteractableActor();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // ======== IInteractableTarget 接口 ========
    
    virtual void GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                          FInteractionOptionBuilder& OptionBuilder) override;

    /**
     * 尝试占用（仅服务器）
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    bool TryOccupy(AActor* OccupyingActor);

    /**
     * 释放占用
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void Release();

    /**
     * 检查是否被占用
     */
    UFUNCTION(BlueprintPure, Category = "Interaction")
    bool IsOccupied() const { return CurrentOccupier != nullptr; }

    /**
     * 获取当前占用者
     */
    UFUNCTION(BlueprintPure, Category = "Interaction")
    AActor* GetCurrentOccupier() const { return CurrentOccupier; }

protected:
    virtual void BeginPlay() override;
    virtual void OnInteracted_Implementation(AActor* InteractingActor, AController* InteractingController) override;

    /**
     * 交互完成回调
     */
    UFUNCTION()
    void OnInteractionCompleted();

protected:
    /** 当前占用者 */
    UPROPERTY(ReplicatedUsing = OnRep_CurrentOccupier)
    TObjectPtr<AActor> CurrentOccupier;

    /** 交互持续时间 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    float InteractionDuration = 3.0f;

    /** 是否自动释放 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    bool bAutoRelease = true;

    /** 占用提示文本 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    FText OccupiedText;

private:
    /** 交互定时器 */
    FTimerHandle InteractionTimerHandle;

    UFUNCTION()
    void OnRep_CurrentOccupier();

    /** 服务器 RPC */
    UFUNCTION(Server, Reliable)
    void ServerTryOccupy(AActor* OccupyingActor);
};
```

```cpp
// ExclusiveInteractableActor.cpp
#include "ExclusiveInteractableActor.h"
#include "Net/UnrealNetwork.h"
#include "TimerManager.h"

AExclusiveInteractableActor::AExclusiveInteractableActor()
{
    OccupiedText = NSLOCTEXT("Interaction", "Occupied", "正在被其他玩家使用");
}

void AExclusiveInteractableActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AExclusiveInteractableActor, CurrentOccupier);
}

void AExclusiveInteractableActor::BeginPlay()
{
    Super::BeginPlay();
}

void AExclusiveInteractableActor::GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                                          FInteractionOptionBuilder& OptionBuilder)
{
    if (!bIsInteractable)
    {
        return;
    }

    AActor* Requester = InteractQuery.RequestingAvatar.Get();

    FInteractionOption Option;
    Option.TargetAbilitySystem = AbilitySystemComponent;
    Option.TargetInteractionAbilityHandle = InteractionAbilityHandle;

    // 检查是否被占用
    if (IsOccupied())
    {
        if (CurrentOccupier == Requester)
        {
            // 当前玩家正在使用
            Option.Text = NSLOCTEXT("Interaction", "InUse", "使用中...");
            Option.SubText = FText::AsPercent(GetInteractionProgress());
        }
        else
        {
            // 其他玩家正在使用
            Option.Text = OccupiedText;
            
            if (CurrentOccupier)
            {
                Option.SubText = FText::Format(
                    NSLOCTEXT("Interaction", "OccupiedBy", "（{0} 正在使用）"),
                    FText::FromString(CurrentOccupier->GetName())
                );
            }
            
            // 不返回选项，无法交互
            return;
        }
    }
    else
    {
        // 未被占用，可以交互
        Option.Text = InteractionText;
        Option.SubText = InteractionSubText;
    }

    OptionBuilder.AddInteractionOption(Option);
}

bool AExclusiveInteractableActor::TryOccupy(AActor* OccupyingActor)
{
    if (!HasAuthority())
    {
        ServerTryOccupy(OccupyingActor);
        return false;
    }

    if (IsOccupied())
    {
        return false;
    }

    CurrentOccupier = OccupyingActor;
    OnRep_CurrentOccupier();

    // 设置自动释放定时器
    if (bAutoRelease && InteractionDuration > 0.0f)
    {
        GetWorld()->GetTimerManager().SetTimer(
            InteractionTimerHandle,
            this,
            &AExclusiveInteractableActor::OnInteractionCompleted,
            InteractionDuration,
            false
        );
    }

    return true;
}

void AExclusiveInteractableActor::Release()
{
    if (HasAuthority())
    {
        GetWorld()->GetTimerManager().ClearTimer(InteractionTimerHandle);
        CurrentOccupier = nullptr;
        OnRep_CurrentOccupier();
    }
}

float AExclusiveInteractableActor::GetInteractionProgress() const
{
    if (!IsOccupied() || InteractionDuration <= 0.0f)
    {
        return 0.0f;
    }

    float Remaining = GetWorld()->GetTimerManager().GetTimerRemaining(InteractionTimerHandle);
    return 1.0f - (Remaining / InteractionDuration);
}

void AExclusiveInteractableActor::OnInteracted_Implementation(AActor* InteractingActor, AController* InteractingController)
{
    if (!HasAuthority())
    {
        return;
    }

    // 尝试占用
    if (TryOccupy(InteractingActor))
    {
        // 占用成功
        // 执行交互逻辑
    }
}

void AExclusiveInteractableActor::OnInteractionCompleted()
{
    if (!HasAuthority())
    {
        return;
    }

    // 交互完成，释放占用
    Release();

    // 执行完成逻辑
    // ...
}

void AExclusiveInteractableActor::OnRep_CurrentOccupier()
{
    // 更新视觉效果，表示占用状态
    if (IsOccupied())
    {
        // 显示"使用中"效果
    }
    else
    {
        // 显示"可用"效果
    }
}

void AExclusiveInteractableActor::ServerTryOccupy_Implementation(AActor* OccupyingActor)
{
    TryOccupy(OccupyingActor);
}
```

### 17.9.2 条件判断系统

实现一个基于条件的交互系统：

```cpp
// InteractionCondition.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "InteractionCondition.generated.h"

struct FInteractionQuery;

/**
 * UInteractionCondition
 * 
 * 交互条件基类
 * 用于判断是否允许交互
 */
UCLASS(Abstract, Blueprintable, EditInlineNew)
class YOURGAME_API UInteractionCondition : public UObject
{
    GENERATED_BODY()

public:
    /**
     * 检查条件是否满足
     * 
     * @param InteractQuery 交互查询参数
     * @return 是否满足条件
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Interaction")
    bool CheckCondition(const FInteractionQuery& InteractQuery) const;
    virtual bool CheckCondition_Implementation(const FInteractionQuery& InteractQuery) const;

    /**
     * 获取失败提示文本
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Interaction")
    FText GetFailureText() const;
    virtual FText GetFailureText_Implementation() const;

protected:
    /** 失败提示文本 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Condition")
    FText FailureText;
};
```

```cpp
// InteractionCondition_HasItem.h
#pragma once

#include "CoreMinimal.h"
#include "InteractionCondition.h"
#include "InteractionCondition_HasItem.generated.h"

/**
 * UInteractionCondition_HasItem
 * 
 * 检查玩家是否拥有特定物品
 */
UCLASS()
class YOURGAME_API UInteractionCondition_HasItem : public UInteractionCondition
{
    GENERATED_BODY()

public:
    virtual bool CheckCondition_Implementation(const FInteractionQuery& InteractQuery) const override;
    virtual FText GetFailureText_Implementation() const override;

protected:
    /** 需要的物品 ID */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Condition")
    FName RequiredItemID;

    /** 需要的物品数量 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Condition")
    int32 RequiredItemCount = 1;

    /** 是否消耗物品 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Condition")
    bool bConsumeItem = false;
};
```

```cpp
// InteractionCondition_HasItem.cpp
#include "InteractionCondition_HasItem.h"
#include "Interaction/InteractionQuery.h"

bool UInteractionCondition_HasItem::CheckCondition_Implementation(const FInteractionQuery& InteractQuery) const
{
    AActor* Requester = InteractQuery.RequestingAvatar.Get();
    if (!Requester)
    {
        return false;
    }

    // 检查背包（需要你的背包系统）
    // if (UInventoryComponent* Inventory = Requester->FindComponentByClass<UInventoryComponent>())
    // {
    //     return Inventory->HasItem(RequiredItemID, RequiredItemCount);
    // }

    return false;
}

FText UInteractionCondition_HasItem::GetFailureText_Implementation() const
{
    if (!FailureText.IsEmpty())
    {
        return FailureText;
    }

    return FText::Format(
        NSLOCTEXT("Interaction", "NeedItem", "需要 {0} x{1}"),
        FText::FromName(RequiredItemID),
        FText::AsNumber(RequiredItemCount)
    );
}
```

### 17.9.3 带条件的交互对象

```cpp
// ConditionalInteractableActor.h
#pragma once

#include "CoreMinimal.h"
#include "InteractableItemBase.h"
#include "InteractionCondition.h"
#include "ConditionalInteractableActor.generated.h"

/**
 * AConditionalInteractableActor
 * 
 * 带条件判断的交互对象
 */
UCLASS()
class YOURGAME_API AConditionalInteractableActor : public AInteractableItemBase
{
    GENERATED_BODY()
    
public:    
    AConditionalInteractableActor();

    // ======== IInteractableTarget 接口 ========
    
    virtual void GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                          FInteractionOptionBuilder& OptionBuilder) override;

    /**
     * 检查所有条件
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    bool CheckAllConditions(const FInteractionQuery& InteractQuery, FText& OutFailureReason) const;

protected:
    /** 交互条件列表 */
    UPROPERTY(EditAnywhere, Instanced, BlueprintReadWrite, Category = "Interaction")
    TArray<TObjectPtr<UInteractionCondition>> InteractionConditions;

    /** 条件全部满足时的文本 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    FText ConditionsMetText;

    /** 是否显示失败原因 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    bool bShowFailureReason = true;
};
```

```cpp
// ConditionalInteractableActor.cpp
#include "ConditionalInteractableActor.h"

AConditionalInteractableActor::AConditionalInteractableActor()
{
    ConditionsMetText = NSLOCTEXT("Interaction", "Use", "使用");
}

void AConditionalInteractableActor::GatherInteractionOptions(const FInteractionQuery& InteractQuery, 
                                                            FInteractionOptionBuilder& OptionBuilder)
{
    if (!bIsInteractable)
    {
        return;
    }

    FInteractionOption Option;
    Option.TargetAbilitySystem = AbilitySystemComponent;
    Option.TargetInteractionAbilityHandle = InteractionAbilityHandle;

    // 检查条件
    FText FailureReason;
    bool bAllConditionsMet = CheckAllConditions(InteractQuery, FailureReason);

    if (bAllConditionsMet)
    {
        // 条件满足
        Option.Text = ConditionsMetText.IsEmpty() ? InteractionText : ConditionsMetText;
        Option.SubText = InteractionSubText;
    }
    else
    {
        // 条件不满足
        Option.Text = NSLOCTEXT("Interaction", "CannotUse", "无法使用");
        
        if (bShowFailureReason && !FailureReason.IsEmpty())
        {
            Option.SubText = FailureReason;
        }
        
        // 不返回选项，无法交互
        return;
    }

    OptionBuilder.AddInteractionOption(Option);
}

bool AConditionalInteractableActor::CheckAllConditions(const FInteractionQuery& InteractQuery, 
                                                       FText& OutFailureReason) const
{
    for (const UInteractionCondition* Condition : InteractionConditions)
    {
        if (!Condition)
        {
            continue;
        }

        if (!Condition->CheckCondition(InteractQuery))
        {
            OutFailureReason = Condition->GetFailureText();
            return false;
        }
    }

    return true;
}
```

### 17.9.4 进度条交互

实现需要蓄力的交互：

```cpp
// ProgressInteractionComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "ProgressInteractionComponent.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnProgressChanged, float, Progress);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnProgressCompleted);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnProgressCancelled);

/**
 * UProgressInteractionComponent
 * 
 * 进度条交互组件
 * 需要玩家持续按住按键完成交互
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class YOURGAME_API UProgressInteractionComponent : public UActorComponent
{
    GENERATED_BODY()

public:    
    UProgressInteractionComponent();

    virtual void TickComponent(float DeltaTime, ELevelTick TickType, 
                              FActorComponentTickFunction* ThisTickFunction) override;

    /**
     * 开始进度
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void StartProgress();

    /**
     * 取消进度
     */
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void CancelProgress();

    /**
     * 获取当前进度
     */
    UFUNCTION(BlueprintPure, Category = "Interaction")
    float GetProgress() const { return CurrentProgress; }

    /**
     * 是否正在进行中
     */
    UFUNCTION(BlueprintPure, Category = "Interaction")
    bool IsInProgress() const { return bIsInProgress; }

public:
    /** 进度变化事件 */
    UPROPERTY(BlueprintAssignable, Category = "Interaction")
    FOnProgressChanged OnProgressChanged;

    /** 进度完成事件 */
    UPROPERTY(BlueprintAssignable, Category = "Interaction")
    FOnProgressCompleted OnProgressCompleted;

    /** 进度取消事件 */
    UPROPERTY(BlueprintAssignable, Category = "Interaction")
    FOnProgressCancelled OnProgressCancelled;

protected:
    virtual void BeginPlay() override;

protected:
    /** 完成所需时间 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    float RequiredDuration = 3.0f;

    /** 是否可以被打断 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    bool bCanBeInterrupted = true;

    /** 打断后是否重置进度 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    bool bResetOnInterrupt = true;

private:
    /** 当前进度 (0.0 - 1.0) */
    float CurrentProgress = 0.0f;

    /** 是否正在进行中 */
    bool bIsInProgress = false;
};
```

```cpp
// ProgressInteractionComponent.cpp
#include "ProgressInteractionComponent.h"

UProgressInteractionComponent::UProgressInteractionComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
    PrimaryComponentTick.bStartWithTickEnabled = false;
}

void UProgressInteractionComponent::BeginPlay()
{
    Super::BeginPlay();
}

void UProgressInteractionComponent::TickComponent(float DeltaTime, ELevelTick TickType, 
                                                 FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    if (!bIsInProgress)
    {
        return;
    }

    // 更新进度
    CurrentProgress += DeltaTime / RequiredDuration;

    // 触发进度变化事件
    OnProgressChanged.Broadcast(CurrentProgress);

    // 检查是否完成
    if (CurrentProgress >= 1.0f)
    {
        CurrentProgress = 1.0f;
        bIsInProgress = false;
        SetComponentTickEnabled(false);

        // 触发完成事件
        OnProgressCompleted.Broadcast();
    }
}

void UProgressInteractionComponent::StartProgress()
{
    if (bIsInProgress)
    {
        return;
    }

    bIsInProgress = true;
    CurrentProgress = 0.0f;
    SetComponentTickEnabled(true);

    // 触发进度变化事件
    OnProgressChanged.Broadcast(CurrentProgress);
}

void UProgressInteractionComponent::CancelProgress()
{
    if (!bIsInProgress)
    {
        return;
    }

    bIsInProgress = false;
    SetComponentTickEnabled(false);

    if (bResetOnInterrupt)
    {
        CurrentProgress = 0.0f;
    }

    // 触发取消事件
    OnProgressCancelled.Broadcast();
}
```

---

## 17.10 总结与最佳实践

### 17.10.1 系统设计原则

Lyra 的交互系统体现了以下设计原则：

#### 1. **接口解耦**
- `IInteractableTarget` 接口将交互逻辑与具体实现分离
- 任何 Actor 或 Component 都可以实现交互功能
- 降低了系统间的耦合度

#### 2. **数据驱动**
- 使用 `FInteractionOption` 结构体封装交互数据
- 通过 DataAsset 配置物品、对话等内容
- 便于策划调整和内容扩展

#### 3. **GAS 集成**
- 交互触发通过 Gameplay Ability 实现
- 利用 GAS 的网络同步、冷却、标签等功能
- 统一的能力管理框架

#### 4. **UI 自动化**
- Indicator System 自动管理交互提示
- Widget 动态创建和销毁
- 减少手动 UI 管理的工作量

#### 5. **网络友好**
- 服务器权威的交互验证
- 客户端预测的流畅体验
- 自动的状态同步

### 17.10.2 常见问题与解决方案

#### 问题1：射线检测不到交互对象

**原因**：
- 碰撞通道配置错误
- 对象没有实现 `IInteractableTarget` 接口
- 射线起点或方向不正确

**解决方案**：
```cpp
// 1. 确保使用正确的碰撞通道
FCollisionProfileName TraceProfile = FName(TEXT("Interaction"));

// 2. 检查对象是否实现接口
if (HitActor->Implements<UInteractableTarget>())
{
    // ...
}

// 3. 调试射线
#if ENABLE_DRAW_DEBUG
DrawDebugLine(World, TraceStart, TraceEnd, FColor::Green, false, 1.0f);
#endif
```

#### 问题2：多人游戏中交互状态不同步

**原因**：
- 变量没有设置为 Replicated
- 忘记在服务器上执行逻辑
- 客户端试图直接修改服务器状态

**解决方案**：
```cpp
// 1. 标记变量为 Replicated
UPROPERTY(ReplicatedUsing = OnRep_InteractionState)
EInteractionState InteractionState;

// 2. 使用 Server RPC
UFUNCTION(Server, Reliable)
void ServerPerformInteraction();

// 3. 检查权限
if (HasAuthority())
{
    // 仅在服务器上执行
}
```

#### 问题3：交互 UI 闪烁或频繁更新

**原因**：
- 检测频率过高
- 没有检查选项是否真正变化
- 每帧都创建新的 Widget

**解决方案**：
```cpp
// 1. 降低检测频率
float InteractionScanRate = 0.1f;  // 每秒检测10次

// 2. 比较选项是否变化
if (NewOptions != CurrentOptions)
{
    UpdateUI(NewOptions);
}

// 3. 复用 Widget
if (!CachedWidget)
{
    CachedWidget = CreateWidget<UInteractionWidget>(...);
}
```

#### 问题4：交互距离判断不准确

**原因**：
- 使用了错误的距离计算方法
- 相机位置与角色位置偏差较大
- 没有考虑碰撞体的大小

**解决方案**：
```cpp
// 1. 从相机位置开始计算
FVector CameraLocation;
FRotator CameraRotation;
PC->GetPlayerViewPoint(CameraLocation, CameraRotation);

// 2. 考虑碰撞体半径
float EffectiveRange = InteractionRange + CollisionRadius;

// 3. 使用球形检测而非点检测
FCollisionShape::MakeSphere(InteractionRadius);
```

### 17.10.3 性能优化建议

#### 1. **减少检测频率**
```cpp
// 根据场景复杂度调整
float InteractionScanRate = 0.1f;  // 简单场景
// float InteractionScanRate = 0.2f;  // 复杂场景
```

#### 2. **使用碰撞通道过滤**
```cpp
// 设置专用的交互通道
Lyra_TraceChannel_Interaction

// 只让需要交互的对象响应此通道
```

#### 3. **限制同时显示的提示数量**
```cpp
// 只显示最近的 N 个交互对象
TArray<FInteractionOption> TopOptions = GetTopNOptions(AllOptions, 3);
```

#### 4. **缓存动态材质和 Widget**
```cpp
// 避免每次都创建新对象
if (!CachedDynamicMaterial)
{
    CachedDynamicMaterial = UMaterialInstanceDynamic::Create(...);
}
```

#### 5. **使用对象池**
```cpp
// 对于频繁生成的交互对象，使用对象池
class FPickupItemPool
{
    TArray<APickupItemActor*> AvailableItems;
    // ...
};
```

### 17.10.4 扩展建议

#### 1. **添加交互声音系统**
```cpp
UCLASS()
class UInteractionAudioComponent : public UActorComponent
{
    // 根据交互类型播放不同音效
    void PlayInteractionSound(EInteractionType Type);
};
```

#### 2. **实现交互历史记录**
```cpp
USTRUCT()
struct FInteractionRecord
{
    FDateTime Timestamp;
    AActor* InteractedObject;
    AActor* InteractingActor;
};

TArray<FInteractionRecord> InteractionHistory;
```

#### 3. **添加交互分析系统**
```cpp
// 记录玩家交互数据，用于关卡设计优化
class FInteractionAnalytics
{
    void RecordInteraction(FName ObjectID, float Duration);
    void GenerateHeatmap();
};
```

#### 4. **支持远程交互**
```cpp
// 通过UI面板远程触发交互
UCLASS()
class URemoteInteractionComponent : public UActorComponent
{
    void TriggerRemoteInteraction(AActor* TargetObject);
};
```

#### 5. **实现交互教程系统**
```cpp
UCLASS()
class UInteractionTutorial : public UObject
{
    // 首次遇到新交互类型时显示教程
    void ShowTutorialFor(FName InteractionType);
};
```

### 17.10.5 调试技巧

#### 1. **启用调试可视化**
```cpp
// DefaultEngine.ini
[/Script/Engine.CollisionProfile]
+Profiles=(Name="Interaction",CollisionEnabled=QueryOnly,...)

// 控制台命令
show collision
stat game
```

#### 2. **添加日志**
```cpp
UE_LOG(LogInteraction, Log, TEXT("Interaction detected: %s"), *InteractedObject->GetName());
UE_LOG(LogInteraction, Warning, TEXT("Interaction failed: %s"), *FailureReason.ToString());
```

#### 3. **使用调试绘制**
```cpp
#if ENABLE_DRAW_DEBUG
DrawDebugSphere(World, InteractionLocation, Radius, 16, FColor::Yellow, false, 1.0f);
DrawDebugString(World, Location, DebugText, nullptr, FColor::White, 1.0f);
#endif
```

#### 4. **创建调试 Widget**
```cpp
UCLASS()
class UInteractionDebugWidget : public UUserWidget
{
    // 显示当前检测到的所有交互对象
    // 显示交互距离、条件等信息
};
```

### 17.10.6 最佳实践清单

✅ **设计阶段**
- [ ] 明确交互类型和规则
- [ ] 设计交互数据结构
- [ ] 规划网络同步策略
- [ ] 定义 UI 规范

✅ **实现阶段**
- [ ] 使用接口而非硬编码类型
- [ ] 所有交互逻辑在服务器验证
- [ ] 实现客户端预测提升体验
- [ ] 添加足够的日志和调试信息

✅ **优化阶段**
- [ ] 调整检测频率和范围
- [ ] 优化碰撞检测
- [ ] 缓存常用对象
- [ ] 性能分析和压力测试

✅ **测试阶段**
- [ ] 单人模式测试
- [ ] 多人模式测试
- [ ] 边界情况测试
- [ ] 网络延迟测试

✅ **维护阶段**
- [ ] 记录常见问题和解决方案
- [ ] 定期review代码
- [ ] 收集玩家反馈
- [ ] 持续优化和改进

---

## 结语

Lyra 的交互系统是一个设计精良、功能强大的框架。通过本章的学习，你应该掌握了：

1. **核心架构**：理解 `IInteractableTarget` 接口、`FInteractionOption` 数据结构、Ability Task 检测机制
2. **UI 集成**：学会使用 Indicator System 自动管理交互提示
3. **网络同步**：掌握服务器权威、客户端预测的实现方式
4. **实战应用**：实现了可拾取物品、可交互门、NPC 对话等完整系统
5. **进阶技巧**：学会处理互斥访问、条件判断、进度条交互等复杂场景

记住：**好的交互系统是无感的**。玩家不应该思考"我该怎么交互"，而应该自然地知道"我能做什么"。通过清晰的视觉提示、即时的反馈、流畅的动画，让交互成为游戏体验的自然延伸。

继续探索 Lyra 的其他系统，你会发现更多巧妙的设计和实现细节。祝你在游戏开发的道路上越走越远！

---

**相关章节推荐**：
- 第16章：UI Extension 系统详解
- 第18章：Gameplay Ability System 深入
- 第19章：网络同步与复制优化

**参考资源**：
- [UE5 官方文档 - Gameplay Ability System](https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-for-unreal-engine/)
- [Lyra 示例项目源码](https://github.com/EpicGames/UnrealEngine)
- [GAS Documentation](https://github.com/tranek/GASDocumentation)

<function_calls>
<invoke name="read">
<parameter name="path">/root/.openclaw/workspace/ue5-lyra-tutorial-docs/docs/03-ui-systems/17-interaction-system.md