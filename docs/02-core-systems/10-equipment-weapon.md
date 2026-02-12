# 装备与武器系统详解

## 概述

装备与武器系统是 Lyra 中最复杂且最具实战价值的系统之一。它不仅仅是简单的"拿起武器开枪"，而是一个深度集成了 GAS（Gameplay Ability System）、网络复制、状态管理和数据驱动设计的完整解决方案。

本文将深入剖析 Lyra 的装备系统架构，从源码级别解析武器系统的实现细节，并通过完整的实战案例展示如何构建一个生产级别的 FPS 武器系统。

### 本文涵盖内容

- **装备系统架构**：Equipment Definition、Equipment Instance、Equipment Manager Component
- **武器系统深度解析**：WeaponInstance、状态机、弹药系统、附件系统
- **与 GAS 的集成**：装备如何授予 Ability、属性修改、网络同步
- **实战案例**：完整的 FPS 武器系统、装备升级、品质系统、UI 集成

### 前置知识

在阅读本文前，建议您先掌握：

- Lyra 的模块化 Actor 组件系统
- GAS 基础知识（Ability、Attribute、GameplayEffect）
- Unreal Engine 的网络复制机制
- C++ 基础和 Unreal Engine C++ 编程

---

## 第一部分：装备系统架构（25%）

### 1.1 Lyra 装备系统设计理念

Lyra 的装备系统遵循以下核心设计原则：

#### 1.1.1 数据与逻辑分离

```
┌─────────────────────────────────────────────────────┐
│                 Equipment Definition                 │
│              (Data Asset - 配置数据)                 │
│  - 装备名称、描述                                    │
│  - 生成的 Actor 类型                                 │
│  - 授予的 Ability Sets                               │
│  - 装备槽位类型                                      │
└─────────────────┬───────────────────────────────────┘
                  │ 实例化
                  ▼
┌─────────────────────────────────────────────────────┐
│                 Equipment Instance                   │
│              (Runtime Instance - 运行时实例)         │
│  - 装备当前状态                                      │
│  - 生成的 Actor 引用                                 │
│  - 授予的 Ability Handles                            │
│  - 网络复制数据                                      │
└─────────────────┬───────────────────────────────────┘
                  │ 管理
                  ▼
┌─────────────────────────────────────────────────────┐
│              Equipment Manager Component             │
│              (组件 - 装备管理器)                     │
│  - 管理所有装备实例                                  │
│  - 处理装备生命周期                                  │
│  - 槽位管理                                          │
│  - 网络同步                                          │
└─────────────────────────────────────────────────────┘
```

#### 1.1.2 模块化设计

装备系统由多个独立但协作的组件构成：

- **ULyraEquipmentDefinition**：装备定义（Data Asset）
- **ULyraEquipmentInstance**：装备实例（Runtime Object）
- **ULyraEquipmentManagerComponent**：装备管理器（Actor Component）
- **ALyraWeaponInstance**：武器特定实例（继承自 EquipmentInstance）

这种设计使得系统高度可扩展，可以轻松添加新的装备类型（近战武器、盾牌、工具等）。

### 1.2 Equipment Definition：装备定义

`ULyraEquipmentDefinition` 是一个 Data Asset，定义了装备的静态数据。

#### 1.2.1 核心类定义

```cpp
// LyraEquipmentDefinition.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "LyraEquipmentDefinition.generated.h"

class ULyraAbilitySet;
class ULyraEquipmentInstance;

/**
 * 装备生成的 Actor 配置
 */
USTRUCT(BlueprintType)
struct FLyraEquipmentActorToSpawn
{
    GENERATED_BODY()

    // 要生成的 Actor 类型
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Equipment)
    TSubclassOf<AActor> ActorToSpawn;

    // 附加的 Socket 名称（通常附加到角色的手部或背部）
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Equipment)
    FName AttachSocket;

    // 附加规则
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Equipment)
    FTransform AttachTransform;
};

/**
 * 装备定义：定义一个装备的静态数据
 */
UCLASS(Blueprintable, Const, Meta = (DisplayName = "Lyra Equipment Definition", ShortTooltip = "Data asset used to define an equipment"))
class LYRAGAME_API ULyraEquipmentDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    ULyraEquipmentDefinition(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    // 装备实例的类型（允许自定义装备实例逻辑）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Equipment)
    TSubclassOf<ULyraEquipmentInstance> InstanceType;

    // 需要生成的 Actors（如武器的视觉模型）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Equipment)
    TArray<FLyraEquipmentActorToSpawn> ActorsToSpawn;

    // 授予的 Ability Sets（装备时授予角色的技能集合）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Abilities)
    TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant;
};
```

#### 1.2.2 实际使用示例

创建一个步枪装备定义：

```cpp
// Content/Equipment/Weapons/Rifle_Standard.uasset (蓝图配置)

Instance Type: ULyraWeaponInstance

Actors To Spawn:
  [0]
    ActorToSpawn: BP_Weapon_Rifle_Mesh
    AttachSocket: hand_r_weapon
    AttachTransform: (Location=(X=0,Y=0,Z=0), Rotation=(P=0,Y=0,R=0), Scale=(X=1,Y=1,Z=1))

Ability Sets To Grant:
  [0] AbilitySet_Weapon_Rifle
    - GA_Weapon_Fire (开火能力)
    - GA_Weapon_Reload (换弹能力)
    - GA_Weapon_Aim (瞄准能力)
```

### 1.3 Equipment Instance：装备实例

`ULyraEquipmentInstance` 是装备的运行时实例，持有装备的动态状态。

#### 1.3.1 核心类定义

```cpp
// LyraEquipmentInstance.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "AbilitySystemInterface.h"
#include "LyraEquipmentInstance.generated.h"

struct FLyraEquipmentActorToSpawn;
class APawn;
class ULyraAbilitySystemComponent;

/**
 * 装备实例：装备的运行时实例
 * 这是一个 UObject，不是 Actor（轻量级设计）
 */
UCLASS(BlueprintType, Blueprintable)
class LYRAGAME_API ULyraEquipmentInstance : public UObject
{
    GENERATED_BODY()

public:
    ULyraEquipmentInstance(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    //~UObject interface
    virtual bool IsSupportedForNetworking() const override { return true; }
    virtual UWorld* GetWorld() const override final;
    //~End of UObject interface

    // 获取类型化的 Pawn
    UFUNCTION(BlueprintPure, Category = Equipment)
    APawn* GetPawn() const;

    // 获取类型化的 Pawn（泛型版本）
    UFUNCTION(BlueprintPure, Category = Equipment, meta = (DeterminesOutputType = PawnType))
    APawn* GetTypedPawn(TSubclassOf<APawn> PawnType) const;

    // 生成的 Actor 列表
    UFUNCTION(BlueprintPure, Category = Equipment)
    TArray<AActor*> GetSpawnedActors() const { return SpawnedActors; }

    // 当装备被装备时调用
    virtual void OnEquipped();

    // 当装备被卸下时调用
    virtual void OnUnequipped();

protected:
    // 蓝图事件：装备时
    UFUNCTION(BlueprintImplementableEvent, Category = Equipment, meta = (DisplayName = "OnEquipped"))
    void K2_OnEquipped();

    // 蓝图事件：卸下时
    UFUNCTION(BlueprintImplementableEvent, Category = Equipment, meta = (DisplayName = "OnUnequipped"))
    void K2_OnUnequipped();

private:
    friend class ULyraEquipmentManagerComponent;

    // 拥有此装备的 Pawn
    UPROPERTY()
    TObjectPtr<APawn> Instigator;

    // 生成的 Actors（如武器模型）
    UPROPERTY()
    TArray<TObjectPtr<AActor>> SpawnedActors;

    // 授予的 Ability Handles（用于卸载装备时移除技能）
    FGameplayAbilitySpecHandles GrantedHandles;
};
```

#### 1.3.2 装备实例的生命周期

```cpp
// LyraEquipmentInstance.cpp

void ULyraEquipmentInstance::OnEquipped()
{
    K2_OnEquipped();
}

void ULyraEquipmentInstance::OnUnequipped()
{
    K2_OnUnequipped();
}

APawn* ULyraEquipmentInstance::GetPawn() const
{
    return Instigator;
}

APawn* ULyraEquipmentInstance::GetTypedPawn(TSubclassOf<APawn> PawnType) const
{
    APawn* Result = nullptr;
    if (UClass* ActualPawnType = PawnType)
    {
        if (GetPawn()->IsA(ActualPawnType))
        {
            Result = GetPawn();
        }
    }
    return Result;
}

UWorld* ULyraEquipmentInstance::GetWorld() const
{
    if (APawn* OwningPawn = GetPawn())
    {
        return OwningPawn->GetWorld();
    }
    return nullptr;
}
```

### 1.4 Equipment Manager Component：装备管理器

`ULyraEquipmentManagerComponent` 是核心管理组件，负责装备的整个生命周期。

#### 1.4.1 核心类定义

```cpp
// LyraEquipmentManagerComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/PawnComponent.h"
#include "Net/Serialization/FastArraySerializer.h"
#include "LyraEquipmentManagerComponent.generated.h"

class ULyraEquipmentDefinition;
class ULyraEquipmentInstance;
class ULyraAbilitySystemComponent;

/**
 * 装备条目：Fast Array 元素
 * 用于网络复制装备列表
 */
USTRUCT(BlueprintType)
struct FLyraEquipmentEntry : public FFastArraySerializerItem
{
    GENERATED_BODY()

    FLyraEquipmentEntry()
    {}

private:
    friend FLyraEquipmentList;
    friend ULyraEquipmentManagerComponent;

    // 装备实例
    UPROPERTY()
    TObjectPtr<ULyraEquipmentInstance> Instance = nullptr;

    // 装备定义（引用，用于客户端重建）
    UPROPERTY(NotReplicated)
    TObjectPtr<const ULyraEquipmentDefinition> EquipmentDefinition = nullptr;
};

/**
 * 装备列表：Fast Array 容器
 * 使用 Fast Array 实现高效的网络复制
 */
USTRUCT(BlueprintType)
struct FLyraEquipmentList : public FFastArraySerializer
{
    GENERATED_BODY()

    FLyraEquipmentList()
        : OwnerComponent(nullptr)
    {
    }

    FLyraEquipmentList(UActorComponent* InOwnerComponent)
        : OwnerComponent(InOwnerComponent)
    {
    }

public:
    //~FFastArraySerializer contract
    void PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize);
    void PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize);
    void PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize);
    //~End of FFastArraySerializer contract

    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
    {
        return FFastArraySerializer::FastArrayDeltaSerialize<FLyraEquipmentEntry, FLyraEquipmentList>(Entries, DeltaParms, *this);
    }

    ULyraEquipmentInstance* AddEntry(const ULyraEquipmentDefinition* EquipmentDefinition);
    void RemoveEntry(ULyraEquipmentInstance* Instance);

private:
    friend ULyraEquipmentManagerComponent;

    // 装备条目数组
    UPROPERTY()
    TArray<FLyraEquipmentEntry> Entries;

    // 拥有此列表的组件
    UPROPERTY(NotReplicated)
    TObjectPtr<UActorComponent> OwnerComponent;
};

template<>
struct TStructOpsTypeTraits<FLyraEquipmentList> : public TStructOpsTypeTraitsBase2<FLyraEquipmentList>
{
    enum
    {
        WithNetDeltaSerializer = true,
    };
};

/**
 * 装备管理器组件：管理 Pawn 的所有装备
 */
UCLASS(BlueprintType, Const)
class LYRAGAME_API ULyraEquipmentManagerComponent : public UPawnComponent
{
    GENERATED_BODY()

public:
    ULyraEquipmentManagerComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    // 装备一个装备（返回装备实例）
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = Equipment)
    ULyraEquipmentInstance* EquipItem(const ULyraEquipmentDefinition* EquipmentDefinition);

    // 卸下一个装备
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = Equipment)
    void UnequipItem(ULyraEquipmentInstance* ItemInstance);

    // 获取所有装备实例
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = Equipment)
    TArray<ULyraEquipmentInstance*> GetEquipmentInstancesOfType(TSubclassOf<ULyraEquipmentInstance> InstanceType) const;

    // 获取第一个指定类型的装备实例
    template <typename T>
    T* GetFirstInstanceOfType()
    {
        for (FLyraEquipmentEntry& Entry : EquipmentList.Entries)
        {
            if (T* Result = Cast<T>(Entry.Instance))
            {
                return Result;
            }
        }
        return nullptr;
    }

    //~UObject interface
    virtual bool ReplicateSubobjects(class UActorChannel* Channel, class FOutBunch* Bunch, FReplicationFlags* RepFlags) override;
    virtual void ReadyForReplication() override;
    //~End of UObject interface

private:
    // 创建装备实例
    ULyraEquipmentInstance* CreateEquipmentInstance(const ULyraEquipmentDefinition* EquipmentDefinition);

    // 装备列表（网络复制）
    UPROPERTY(Replicated)
    FLyraEquipmentList EquipmentList;
};
```

#### 1.4.2 装备管理器实现

```cpp
// LyraEquipmentManagerComponent.cpp

#include "LyraEquipmentManagerComponent.h"
#include "LyraEquipmentDefinition.h"
#include "LyraEquipmentInstance.h"
#include "AbilitySystem/LyraAbilitySystemComponent.h"
#include "AbilitySystem/LyraAbilitySet.h"
#include "Engine/ActorChannel.h"
#include "Net/UnrealNetwork.h"

ULyraEquipmentManagerComponent::ULyraEquipmentManagerComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
    , EquipmentList(this)
{
    SetIsReplicatedByDefault(true);
    bWantsInitializeComponent = true;
}

void ULyraEquipmentManagerComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ThisClass, EquipmentList);
}

ULyraEquipmentInstance* ULyraEquipmentManagerComponent::EquipItem(const ULyraEquipmentDefinition* EquipmentDefinition)
{
    ULyraEquipmentInstance* Result = nullptr;
    if (EquipmentDefinition != nullptr)
    {
        // 只在服务器上执行
        if (GetOwner()->HasAuthority())
        {
            Result = EquipmentList.AddEntry(EquipmentDefinition);
            if (Result != nullptr)
            {
                // 生成 Actors
                for (const FLyraEquipmentActorToSpawn& SpawnInfo : EquipmentDefinition->ActorsToSpawn)
                {
                    AActor* NewActor = GetWorld()->SpawnActorDeferred<AActor>(
                        SpawnInfo.ActorToSpawn, 
                        FTransform::Identity, 
                        GetOwner(), 
                        GetOwner<APawn>(), 
                        ESpawnActorCollisionHandlingMethod::AlwaysSpawn);

                    NewActor->FinishSpawning(FTransform::Identity, true);
                    NewActor->SetActorRelativeTransform(SpawnInfo.AttachTransform);
                    NewActor->AttachToComponent(GetOwner<APawn>()->GetRootComponent(), 
                        FAttachmentTransformRules::KeepRelativeTransform, 
                        SpawnInfo.AttachSocket);

                    Result->SpawnedActors.Add(NewActor);
                }

                // 授予 Abilities
                if (ULyraAbilitySystemComponent* ASC = GetAbilitySystemComponent())
                {
                    for (const ULyraAbilitySet* AbilitySet : EquipmentDefinition->AbilitySetsToGrant)
                    {
                        AbilitySet->GiveToAbilitySystem(ASC, &Result->GrantedHandles, Result);
                    }
                }

                // 调用装备事件
                Result->OnEquipped();
            }
        }
    }
    return Result;
}

void ULyraEquipmentManagerComponent::UnequipItem(ULyraEquipmentInstance* ItemInstance)
{
    if (ItemInstance != nullptr)
    {
        if (GetOwner()->HasAuthority())
        {
            // 调用卸下事件
            ItemInstance->OnUnequipped();

            // 移除授予的 Abilities
            if (ULyraAbilitySystemComponent* ASC = GetAbilitySystemComponent())
            {
                ItemInstance->GrantedHandles.TakeFromAbilitySystem(ASC);
            }

            // 销毁生成的 Actors
            for (AActor* Actor : ItemInstance->SpawnedActors)
            {
                if (Actor)
                {
                    Actor->Destroy();
                }
            }

            // 从列表中移除
            EquipmentList.RemoveEntry(ItemInstance);
        }
    }
}

ULyraAbilitySystemComponent* ULyraEquipmentManagerComponent::GetAbilitySystemComponent() const
{
    if (APawn* OwningPawn = GetPawn<APawn>())
    {
        return Cast<ULyraAbilitySystemComponent>(UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(OwningPawn));
    }
    return nullptr;
}

TArray<ULyraEquipmentInstance*> ULyraEquipmentManagerComponent::GetEquipmentInstancesOfType(TSubclassOf<ULyraEquipmentInstance> InstanceType) const
{
    TArray<ULyraEquipmentInstance*> Results;
    for (const FLyraEquipmentEntry& Entry : EquipmentList.Entries)
    {
        if (ULyraEquipmentInstance* Instance = Entry.Instance)
        {
            if (Instance->IsA(InstanceType))
            {
                Results.Add(Instance);
            }
        }
    }
    return Results;
}

bool ULyraEquipmentManagerComponent::ReplicateSubobjects(UActorChannel* Channel, FOutBunch* Bunch, FReplicationFlags* RepFlags)
{
    bool WroteSomething = Super::ReplicateSubobjects(Channel, Bunch, RepFlags);

    for (FLyraEquipmentEntry& Entry : EquipmentList.Entries)
    {
        ULyraEquipmentInstance* Instance = Entry.Instance;

        if (IsValid(Instance))
        {
            WroteSomething |= Channel->ReplicateSubobject(Instance, *Bunch, *RepFlags);
        }
    }

    return WroteSomething;
}

void ULyraEquipmentManagerComponent::ReadyForReplication()
{
    Super::ReadyForReplication();

    // 注册现有的装备实例到网络复制
    if (IsUsingRegisteredSubObjectList())
    {
        for (const FLyraEquipmentEntry& Entry : EquipmentList.Entries)
        {
            ULyraEquipmentInstance* Instance = Entry.Instance;

            if (IsValid(Instance))
            {
                AddReplicatedSubObject(Instance);
            }
        }
    }
}

// ========== FLyraEquipmentList 实现 ==========

ULyraEquipmentInstance* FLyraEquipmentList::AddEntry(const ULyraEquipmentDefinition* EquipmentDefinition)
{
    ULyraEquipmentInstance* Result = nullptr;

    check(EquipmentDefinition != nullptr);
    check(OwnerComponent);
    check(OwnerComponent->GetOwner()->HasAuthority());

    AActor* OwningActor = OwnerComponent->GetOwner();
    check(OwningActor->HasAuthority());

    // 创建实例类型
    const TSubclassOf<ULyraEquipmentInstance> InstanceType = EquipmentDefinition->InstanceType;

    ULyraEquipmentInstance* NewInstance = NewObject<ULyraEquipmentInstance>(OwnerComponent->GetOwner(), InstanceType);

    NewInstance->Instigator = Cast<APawn>(OwningActor);

    // 添加到数组
    FLyraEquipmentEntry& NewEntry = Entries.AddDefaulted_GetRef();
    NewEntry.Instance = NewInstance;
    NewEntry.EquipmentDefinition = EquipmentDefinition;
    MarkItemDirty(NewEntry);

    Result = NewInstance;

    return Result;
}

void FLyraEquipmentList::RemoveEntry(ULyraEquipmentInstance* Instance)
{
    for (auto EntryIt = Entries.CreateIterator(); EntryIt; ++EntryIt)
    {
        FLyraEquipmentEntry& Entry = *EntryIt;
        if (Entry.Instance == Instance)
        {
            EntryIt.RemoveCurrent();
            MarkArrayDirty();
        }
    }
}

void FLyraEquipmentList::PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize)
{
    // 客户端收到移除通知
}

void FLyraEquipmentList::PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize)
{
    // 客户端收到新增通知
    for (int32 Index : AddedIndices)
    {
        FLyraEquipmentEntry& Entry = Entries[Index];
        if (Entry.Instance != nullptr)
        {
            Entry.Instance->OnEquipped();
        }
    }
}

void FLyraEquipmentList::PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize)
{
    // 客户端收到修改通知
}
```

### 1.5 装备槽位系统

Lyra 并没有内置复杂的槽位系统，但我们可以轻松扩展。

#### 1.5.1 槽位枚举定义

```cpp
// LyraEquipmentSlot.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayTagContainer.h"

/**
 * 装备槽位类型（使用 Gameplay Tags）
 */
namespace LyraEquipmentSlotTags
{
    // 主武器槽
    LYRAGAME_API extern FGameplayTag PrimaryWeapon;
    
    // 副武器槽
    LYRAGAME_API extern FGameplayTag SecondaryWeapon;
    
    // 近战武器槽
    LYRAGAME_API extern FGameplayTag MeleeWeapon;
    
    // 投掷物槽
    LYRAGAME_API extern FGameplayTag Throwable;
    
    // 头盔槽
    LYRAGAME_API extern FGameplayTag Helmet;
    
    // 护甲槽
    LYRAGAME_API extern FGameplayTag Armor;
}
```

```cpp
// LyraEquipmentSlot.cpp
#include "LyraEquipmentSlot.h"

namespace LyraEquipmentSlotTags
{
    UE_DEFINE_GAMEPLAY_TAG(PrimaryWeapon, "Equipment.Slot.Weapon.Primary");
    UE_DEFINE_GAMEPLAY_TAG(SecondaryWeapon, "Equipment.Slot.Weapon.Secondary");
    UE_DEFINE_GAMEPLAY_TAG(MeleeWeapon, "Equipment.Slot.Weapon.Melee");
    UE_DEFINE_GAMEPLAY_TAG(Throwable, "Equipment.Slot.Throwable");
    UE_DEFINE_GAMEPLAY_TAG(Helmet, "Equipment.Slot.Armor.Helmet");
    UE_DEFINE_GAMEPLAY_TAG(Armor, "Equipment.Slot.Armor.Body");
}
```

#### 1.5.2 扩展装备定义支持槽位

```cpp
// 在 ULyraEquipmentDefinition 中添加
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Equipment)
FGameplayTag EquipmentSlot;
```

#### 1.5.3 扩展装备管理器支持槽位

```cpp
// LyraEquipmentManagerComponent.h 扩展

// 槽位到装备实例的映射
UPROPERTY()
TMap<FGameplayTag, TObjectPtr<ULyraEquipmentInstance>> SlotToInstanceMap;

// 在指定槽位装备（如果槽位已有装备则先卸下）
UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = Equipment)
ULyraEquipmentInstance* EquipItemInSlot(const ULyraEquipmentDefinition* EquipmentDefinition, FGameplayTag Slot);

// 卸下指定槽位的装备
UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = Equipment)
void UnequipSlot(FGameplayTag Slot);

// 获取指定槽位的装备
UFUNCTION(BlueprintCallable, BlueprintPure, Category = Equipment)
ULyraEquipmentInstance* GetEquipmentInSlot(FGameplayTag Slot) const;
```

```cpp
// LyraEquipmentManagerComponent.cpp 实现

ULyraEquipmentInstance* ULyraEquipmentManagerComponent::EquipItemInSlot(
    const ULyraEquipmentDefinition* EquipmentDefinition, 
    FGameplayTag Slot)
{
    if (!EquipmentDefinition || !Slot.IsValid())
    {
        return nullptr;
    }

    // 如果槽位已有装备，先卸下
    if (ULyraEquipmentInstance* ExistingItem = SlotToInstanceMap.FindRef(Slot))
    {
        UnequipItem(ExistingItem);
    }

    // 装备新物品
    ULyraEquipmentInstance* NewInstance = EquipItem(EquipmentDefinition);
    if (NewInstance)
    {
        SlotToInstanceMap.Add(Slot, NewInstance);
    }

    return NewInstance;
}

void ULyraEquipmentManagerComponent::UnequipSlot(FGameplayTag Slot)
{
    if (ULyraEquipmentInstance* Instance = SlotToInstanceMap.FindRef(Slot))
    {
        UnequipItem(Instance);
        SlotToInstanceMap.Remove(Slot);
    }
}

ULyraEquipmentInstance* ULyraEquipmentManagerComponent::GetEquipmentInSlot(FGameplayTag Slot) const
{
    return SlotToInstanceMap.FindRef(Slot);
}
```

### 1.6 装备生命周期管理

#### 1.6.1 装备生命周期状态

```
装备 → 激活使用 → 停用 → 卸下 → 销毁
  ↓        ↓        ↓      ↓      ↓
OnEquipped → OnActivated → OnDeactivated → OnUnequipped → OnDestroyed
```

#### 1.6.2 扩展装备实例支持激活状态

```cpp
// LyraEquipmentInstance.h 扩展

protected:
    // 装备是否已激活（如：武器是否在手中，而不是在背包）
    UPROPERTY(BlueprintReadOnly, Replicated, Category = Equipment)
    bool bIsActive = false;

public:
    // 激活装备（如：切换到这把武器）
    UFUNCTION(BlueprintCallable, Category = Equipment)
    virtual void Activate();

    // 停用装备（如：切换到其他武器）
    UFUNCTION(BlueprintCallable, Category = Equipment)
    virtual void Deactivate();

    // 是否已激活
    UFUNCTION(BlueprintPure, Category = Equipment)
    bool IsActive() const { return bIsActive; }

protected:
    virtual void OnActivated();
    virtual void OnDeactivated();

    UFUNCTION(BlueprintImplementableEvent, Category = Equipment, meta = (DisplayName = "OnActivated"))
    void K2_OnActivated();

    UFUNCTION(BlueprintImplementableEvent, Category = Equipment, meta = (DisplayName = "OnDeactivated"))
    void K2_OnDeactivated();
```

```cpp
// LyraEquipmentInstance.cpp 实现

void ULyraEquipmentInstance::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(ThisClass, bIsActive);
}

void ULyraEquipmentInstance::Activate()
{
    if (!bIsActive)
    {
        bIsActive = true;
        OnActivated();
    }
}

void ULyraEquipmentInstance::Deactivate()
{
    if (bIsActive)
    {
        bIsActive = false;
        OnDeactivated();
    }
}

void ULyraEquipmentInstance::OnActivated()
{
    K2_OnActivated();
    
    // 显示生成的 Actors
    for (AActor* Actor : SpawnedActors)
    {
        if (Actor)
        {
            Actor->SetActorHiddenInGame(false);
        }
    }
}

void ULyraEquipmentInstance::OnDeactivated()
{
    K2_OnDeactivated();
    
    // 隐藏生成的 Actors
    for (AActor* Actor : SpawnedActors)
    {
        if (Actor)
        {
            Actor->SetActorHiddenInGame(true);
        }
    }
}
```

---

## 第二部分：武器系统深度解析（30%）

### 2.1 WeaponInstance 详解

`ALyraWeaponInstance` 继承自 `ULyraEquipmentInstance`，专门用于武器。

#### 2.1.1 核心类定义

```cpp
// LyraWeaponInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Equipment/LyraEquipmentInstance.h"
#include "LyraWeaponInstance.generated.h"

class ULyraWeaponStateComponent;

/**
 * 武器实例：武器的运行时实例
 */
UCLASS(BlueprintType, Blueprintable)
class LYRAGAME_API ULyraWeaponInstance : public ULyraEquipmentInstance
{
    GENERATED_BODY()

public:
    ULyraWeaponInstance(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    //~UObject interface
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    //~End of UObject interface

    // 当前弹药数（弹夹中）
    UPROPERTY(BlueprintReadOnly, Replicated, Category = "Weapon|Ammo")
    int32 CurrentAmmo;

    // 最大弹药数（弹夹容量）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Ammo")
    int32 MaxAmmo = 30;

    // 备用弹药数（背包中）
    UPROPERTY(BlueprintReadOnly, Replicated, Category = "Weapon|Ammo")
    int32 ReserveAmmo;

    // 最大备用弹药数
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Ammo")
    int32 MaxReserveAmmo = 120;

    // 射速（每分钟发射次数）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Stats")
    float FireRate = 600.0f;

    // 伤害值
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Stats")
    float Damage = 25.0f;

    // 射程
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Stats")
    float Range = 10000.0f;

    // 后坐力（水平）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Recoil")
    float RecoilHorizontal = 1.0f;

    // 后坐力（垂直）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Recoil")
    float RecoilVertical = 2.0f;

    // 扩散（精度）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Spread")
    float BaseSpread = 0.5f;

    // 瞄准时扩散倍率
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Spread")
    float AimSpreadMultiplier = 0.5f;

    // 消耗弹药
    UFUNCTION(BlueprintCallable, Category = "Weapon|Ammo")
    bool ConsumeAmmo(int32 Amount = 1);

    // 添加弹药（拾取）
    UFUNCTION(BlueprintCallable, Category = "Weapon|Ammo")
    void AddAmmo(int32 Amount);

    // 换弹
    UFUNCTION(BlueprintCallable, Category = "Weapon|Ammo")
    void Reload();

    // 是否可以开火
    UFUNCTION(BlueprintPure, Category = "Weapon")
    bool CanFire() const;

    // 是否可以换弹
    UFUNCTION(BlueprintPure, Category = "Weapon")
    bool CanReload() const;

    // 获取武器状态组件
    UFUNCTION(BlueprintPure, Category = "Weapon")
    ULyraWeaponStateComponent* GetWeaponStateComponent() const;

protected:
    virtual void OnEquipped() override;
    virtual void OnUnequipped() override;

private:
    // 武器状态组件（管理武器状态机）
    UPROPERTY()
    TObjectPtr<ULyraWeaponStateComponent> WeaponStateComponent;
};
```

#### 2.1.2 武器实例实现

```cpp
// LyraWeaponInstance.cpp

#include "LyraWeaponInstance.h"
#include "LyraWeaponStateComponent.h"
#include "Net/UnrealNetwork.h"

ULyraWeaponInstance::ULyraWeaponInstance(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    CurrentAmmo = MaxAmmo;
    ReserveAmmo = MaxReserveAmmo;
}

void ULyraWeaponInstance::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ThisClass, CurrentAmmo);
    DOREPLIFETIME(ThisClass, ReserveAmmo);
}

bool ULyraWeaponInstance::ConsumeAmmo(int32 Amount)
{
    if (CurrentAmmo >= Amount)
    {
        CurrentAmmo -= Amount;
        return true;
    }
    return false;
}

void ULyraWeaponInstance::AddAmmo(int32 Amount)
{
    ReserveAmmo = FMath::Min(ReserveAmmo + Amount, MaxReserveAmmo);
}

void ULyraWeaponInstance::Reload()
{
    if (!CanReload())
    {
        return;
    }

    int32 AmmoNeeded = MaxAmmo - CurrentAmmo;
    int32 AmmoToReload = FMath::Min(AmmoNeeded, ReserveAmmo);

    CurrentAmmo += AmmoToReload;
    ReserveAmmo -= AmmoToReload;
}

bool ULyraWeaponInstance::CanFire() const
{
    return CurrentAmmo > 0 && IsActive();
}

bool ULyraWeaponInstance::CanReload() const
{
    return CurrentAmmo < MaxAmmo && ReserveAmmo > 0 && IsActive();
}

ULyraWeaponStateComponent* ULyraWeaponInstance::GetWeaponStateComponent() const
{
    return WeaponStateComponent;
}

void ULyraWeaponInstance::OnEquipped()
{
    Super::OnEquipped();

    // 创建武器状态组件（如果需要）
    if (APawn* OwningPawn = GetPawn())
    {
        WeaponStateComponent = NewObject<ULyraWeaponStateComponent>(OwningPawn, TEXT("WeaponStateComponent"));
        WeaponStateComponent->RegisterComponent();
        WeaponStateComponent->SetWeaponInstance(this);
    }
}

void ULyraWeaponInstance::OnUnequipped()
{
    Super::OnUnequipped();

    // 销毁武器状态组件
    if (WeaponStateComponent)
    {
        WeaponStateComponent->DestroyComponent();
        WeaponStateComponent = nullptr;
    }
}
```

### 2.2 武器状态机

武器的行为由状态机管理，包括：待机、瞄准、射击、换弹等状态。

#### 2.2.1 武器状态枚举

```cpp
// LyraWeaponStateComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "LyraWeaponStateComponent.generated.h"

class ULyraWeaponInstance;

/**
 * 武器状态枚举
 */
UENUM(BlueprintType)
enum class ELyraWeaponState : uint8
{
    Idle            UMETA(DisplayName = "Idle"),        // 待机
    Firing          UMETA(DisplayName = "Firing"),      // 射击中
    Reloading       UMETA(DisplayName = "Reloading"),   // 换弹中
    Equipping       UMETA(DisplayName = "Equipping"),   // 装备中
    Unequipping     UMETA(DisplayName = "Unequipping")  // 卸下中
};

/**
 * 武器状态组件：管理武器状态机
 */
UCLASS(BlueprintType, meta = (BlueprintSpawnableComponent))
class LYRAGAME_API ULyraWeaponStateComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    ULyraWeaponStateComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    //~UActorComponent interface
    virtual void BeginPlay() override;
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    //~End of UActorComponent interface

    // 设置武器实例
    void SetWeaponInstance(ULyraWeaponInstance* InWeaponInstance);

    // 获取当前状态
    UFUNCTION(BlueprintPure, Category = "Weapon|State")
    ELyraWeaponState GetCurrentState() const { return CurrentState; }

    // 切换状态
    UFUNCTION(BlueprintCallable, Category = "Weapon|State")
    void ChangeState(ELyraWeaponState NewState);

    // 是否处于指定状态
    UFUNCTION(BlueprintPure, Category = "Weapon|State")
    bool IsInState(ELyraWeaponState State) const { return CurrentState == State; }

    // 是否可以切换到指定状态
    UFUNCTION(BlueprintPure, Category = "Weapon|State")
    bool CanTransitionTo(ELyraWeaponState NewState) const;

protected:
    // 当前状态
    UPROPERTY(BlueprintReadOnly, Replicated, Category = "Weapon|State")
    ELyraWeaponState CurrentState;

    // 状态进入时间
    UPROPERTY(BlueprintReadOnly, Category = "Weapon|State")
    float StateEnterTime;

    // 武器实例引用
    UPROPERTY()
    TObjectPtr<ULyraWeaponInstance> WeaponInstance;

    // 状态进入事件
    virtual void OnStateEnter(ELyraWeaponState NewState);

    // 状态退出事件
    virtual void OnStateExit(ELyraWeaponState OldState);

    // 状态更新事件
    virtual void OnStateTick(float DeltaTime);

    // 蓝图事件
    UFUNCTION(BlueprintImplementableEvent, Category = "Weapon|State", meta = (DisplayName = "OnStateChanged"))
    void K2_OnStateChanged(ELyraWeaponState OldState, ELyraWeaponState NewState);
};
```

#### 2.2.2 武器状态组件实现

```cpp
// LyraWeaponStateComponent.cpp

#include "LyraWeaponStateComponent.h"
#include "LyraWeaponInstance.h"
#include "Net/UnrealNetwork.h"

ULyraWeaponStateComponent::ULyraWeaponStateComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    PrimaryComponentTick.bCanEverTick = true;
    PrimaryComponentTick.bStartWithTickEnabled = true;
    SetIsReplicatedByDefault(true);

    CurrentState = ELyraWeaponState::Idle;
    StateEnterTime = 0.0f;
}

void ULyraWeaponStateComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ThisClass, CurrentState);
}

void ULyraWeaponStateComponent::BeginPlay()
{
    Super::BeginPlay();
    StateEnterTime = GetWorld()->GetTimeSeconds();
}

void ULyraWeaponStateComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    OnStateTick(DeltaTime);
}

void ULyraWeaponStateComponent::SetWeaponInstance(ULyraWeaponInstance* InWeaponInstance)
{
    WeaponInstance = InWeaponInstance;
}

void ULyraWeaponStateComponent::ChangeState(ELyraWeaponState NewState)
{
    if (CurrentState == NewState)
    {
        return;
    }

    if (!CanTransitionTo(NewState))
    {
        return;
    }

    ELyraWeaponState OldState = CurrentState;

    // 退出旧状态
    OnStateExit(OldState);

    // 更新状态
    CurrentState = NewState;
    StateEnterTime = GetWorld()->GetTimeSeconds();

    // 进入新状态
    OnStateEnter(NewState);

    // 触发蓝图事件
    K2_OnStateChanged(OldState, NewState);
}

bool ULyraWeaponStateComponent::CanTransitionTo(ELyraWeaponState NewState) const
{
    // 定义状态转换规则
    switch (CurrentState)
    {
        case ELyraWeaponState::Idle:
            // 待机状态可以切换到任何状态
            return true;

        case ELyraWeaponState::Firing:
            // 射击时只能切换到待机或换弹
            return NewState == ELyraWeaponState::Idle || NewState == ELyraWeaponState::Reloading;

        case ELyraWeaponState::Reloading:
            // 换弹时不能中断（除非切换武器）
            return NewState == ELyraWeaponState::Idle || NewState == ELyraWeaponState::Unequipping;

        case ELyraWeaponState::Equipping:
            // 装备中不能中断
            return NewState == ELyraWeaponState::Idle;

        case ELyraWeaponState::Unequipping:
            // 卸下中不能中断
            return false;

        default:
            return false;
    }
}

void ULyraWeaponStateComponent::OnStateEnter(ELyraWeaponState NewState)
{
    // 状态进入逻辑
    switch (NewState)
    {
        case ELyraWeaponState::Idle:
            // 待机：无特殊处理
            break;

        case ELyraWeaponState::Firing:
            // 射击：可以在此处播放射击动画
            break;

        case ELyraWeaponState::Reloading:
            // 换弹：可以在此处播放换弹动画
            break;

        case ELyraWeaponState::Equipping:
            // 装备：可以在此处播放装备动画
            break;

        case ELyraWeaponState::Unequipping:
            // 卸下：可以在此处播放卸下动画
            break;
    }
}

void ULyraWeaponStateComponent::OnStateExit(ELyraWeaponState OldState)
{
    // 状态退出逻辑
}

void ULyraWeaponStateComponent::OnStateTick(float DeltaTime)
{
    // 状态更新逻辑
    switch (CurrentState)
    {
        case ELyraWeaponState::Reloading:
        {
            // 换弹完成检测（假设换弹时间为 2 秒）
            float TimeInState = GetWorld()->GetTimeSeconds() - StateEnterTime;
            if (TimeInState >= 2.0f)
            {
                if (WeaponInstance)
                {
                    WeaponInstance->Reload();
                }
                ChangeState(ELyraWeaponState::Idle);
            }
            break;
        }

        case ELyraWeaponState::Equipping:
        {
            // 装备完成检测（假设装备时间为 0.5 秒）
            float TimeInState = GetWorld()->GetTimeSeconds() - StateEnterTime;
            if (TimeInState >= 0.5f)
            {
                ChangeState(ELyraWeaponState::Idle);
            }
            break;
        }

        default:
            break;
    }
}
```

### 2.3 弹药系统与弹夹管理

弹药系统已经在 `ULyraWeaponInstance` 中实现了基础功能，现在我们扩展它以支持更复杂的弹药类型。

#### 2.3.1 弹药类型系统

```cpp
// LyraAmmoType.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "GameplayTagContainer.h"
#include "LyraAmmoType.generated.h"

/**
 * 弹药类型定义
 */
UCLASS(BlueprintType)
class LYRAGAME_API ULyraAmmoType : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 弹药类型标签（如：Ammo.Rifle.556、Ammo.Shotgun.12Gauge）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Ammo")
    FGameplayTag AmmoTag;

    // 弹药名称
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Ammo")
    FText AmmoName;

    // 弹药描述
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Ammo")
    FText AmmoDescription;

    // 弹药图标
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Ammo")
    TObjectPtr<UTexture2D> AmmoIcon;

    // 最大堆叠数量
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Ammo")
    int32 MaxStackSize = 999;
};
```

#### 2.3.2 扩展武器实例支持弹药类型

```cpp
// LyraWeaponInstance.h 扩展

// 弹药类型
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon|Ammo")
TObjectPtr<ULyraAmmoType> AmmoType;

// 根据弹药类型添加弹药（从背包系统调用）
UFUNCTION(BlueprintCallable, Category = "Weapon|Ammo")
bool AddAmmoByType(ULyraAmmoType* InAmmoType, int32 Amount);
```

```cpp
// LyraWeaponInstance.cpp 实现

bool ULyraWeaponInstance::AddAmmoByType(ULyraAmmoType* InAmmoType, int32 Amount)
{
    if (InAmmoType != AmmoType)
    {
        return false; // 弹药类型不匹配
    }

    AddAmmo(Amount);
    return true;
}
```

### 2.4 武器附件系统

附件系统允许玩家为武器添加瞄具、握把、消音器等配件，这些配件会修改武器属性。

#### 2.4.1 附件定义

```cpp
// LyraWeaponAttachment.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "GameplayTagContainer.h"
#include "LyraWeaponAttachment.generated.h"

/**
 * 附件槽位类型
 */
UENUM(BlueprintType)
enum class ELyraAttachmentSlot : uint8
{
    Optic       UMETA(DisplayName = "Optic"),       // 瞄具
    Muzzle      UMETA(DisplayName = "Muzzle"),      // 枪口（消音器、补偿器）
    Grip        UMETA(DisplayName = "Grip"),        // 握把
    Magazine    UMETA(DisplayName = "Magazine"),    // 弹夹
    Stock       UMETA(DisplayName = "Stock")        // 枪托
};

/**
 * 附件属性修改
 */
USTRUCT(BlueprintType)
struct FLyraAttachmentStatModifier
{
    GENERATED_BODY()

    // 伤害倍率
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float DamageMultiplier = 1.0f;

    // 射速倍率
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float FireRateMultiplier = 1.0f;

    // 后坐力倍率（水平）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float RecoilHorizontalMultiplier = 1.0f;

    // 后坐力倍率（垂直）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float RecoilVerticalMultiplier = 1.0f;

    // 扩散倍率
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float SpreadMultiplier = 1.0f;

    // 弹夹容量加成
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    int32 MagazineCapacityBonus = 0;
};

/**
 * 武器附件定义
 */
UCLASS(BlueprintType)
class LYRAGAME_API ULyraWeaponAttachment : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 附件名称
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attachment")
    FText AttachmentName;

    // 附件描述
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attachment")
    FText AttachmentDescription;

    // 附件图标
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attachment")
    TObjectPtr<UTexture2D> AttachmentIcon;

    // 附件槽位
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attachment")
    ELyraAttachmentSlot AttachmentSlot;

    // 附件网格
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attachment")
    TObjectPtr<UStaticMesh> AttachmentMesh;

    // 附加 Socket 名称
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attachment")
    FName AttachSocket;

    // 属性修改
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attachment")
    FLyraAttachmentStatModifier StatModifier;

    // 兼容的武器标签（如：Weapon.Type.Rifle）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attachment")
    FGameplayTagContainer CompatibleWeaponTags;
};
```

#### 2.4.2 扩展武器实例支持附件

```cpp
// LyraWeaponInstance.h 扩展

// 附件槽位映射
UPROPERTY(BlueprintReadOnly, Replicated, Category = "Weapon|Attachments")
TMap<ELyraAttachmentSlot, TObjectPtr<ULyraWeaponAttachment>> AttachedAttachments;

// 武器标签（用于附件兼容性检查）
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
FGameplayTagContainer WeaponTags;

// 附加附件
UFUNCTION(BlueprintCallable, Category = "Weapon|Attachments")
bool AttachAttachment(ULyraWeaponAttachment* Attachment);

// 移除附件
UFUNCTION(BlueprintCallable, Category = "Weapon|Attachments")
bool RemoveAttachment(ELyraAttachmentSlot Slot);

// 获取指定槽位的附件
UFUNCTION(BlueprintPure, Category = "Weapon|Attachments")
ULyraWeaponAttachment* GetAttachment(ELyraAttachmentSlot Slot) const;

// 检查附件是否兼容
UFUNCTION(BlueprintPure, Category = "Weapon|Attachments")
bool IsAttachmentCompatible(ULyraWeaponAttachment* Attachment) const;

// 获取修改后的武器属性
UFUNCTION(BlueprintPure, Category = "Weapon|Stats")
float GetModifiedDamage() const;

UFUNCTION(BlueprintPure, Category = "Weapon|Stats")
float GetModifiedFireRate() const;

UFUNCTION(BlueprintPure, Category = "Weapon|Stats")
float GetModifiedRecoilHorizontal() const;

UFUNCTION(BlueprintPure, Category = "Weapon|Stats")
float GetModifiedRecoilVertical() const;

UFUNCTION(BlueprintPure, Category = "Weapon|Stats")
float GetModifiedSpread() const;

UFUNCTION(BlueprintPure, Category = "Weapon|Stats")
int32 GetModifiedMaxAmmo() const;
```

#### 2.4.3 附件系统实现

```cpp
// LyraWeaponInstance.cpp 实现

void ULyraWeaponInstance::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    // ... 其他属性
    DOREPLIFETIME(ThisClass, AttachedAttachments);
}

bool ULyraWeaponInstance::AttachAttachment(ULyraWeaponAttachment* Attachment)
{
    if (!Attachment || !IsAttachmentCompatible(Attachment))
    {
        return false;
    }

    ELyraAttachmentSlot Slot = Attachment->AttachmentSlot;

    // 如果槽位已有附件，先移除
    RemoveAttachment(Slot);

    // 添加新附件
    AttachedAttachments.Add(Slot, Attachment);

    // 生成附件网格（如果有）
    if (Attachment->AttachmentMesh && SpawnedActors.Num() > 0)
    {
        AActor* WeaponActor = SpawnedActors[0];
        if (WeaponActor)
        {
            UStaticMeshComponent* AttachmentMeshComp = NewObject<UStaticMeshComponent>(WeaponActor);
            AttachmentMeshComp->SetStaticMesh(Attachment->AttachmentMesh);
            AttachmentMeshComp->AttachToComponent(
                WeaponActor->GetRootComponent(),
                FAttachmentTransformRules::SnapToTargetIncludingScale,
                Attachment->AttachSocket
            );
            AttachmentMeshComp->RegisterComponent();
        }
    }

    return true;
}

bool ULyraWeaponInstance::RemoveAttachment(ELyraAttachmentSlot Slot)
{
    if (ULyraWeaponAttachment* Attachment = AttachedAttachments.FindRef(Slot))
    {
        AttachedAttachments.Remove(Slot);
        // TODO: 销毁附件网格
        return true;
    }
    return false;
}

ULyraWeaponAttachment* ULyraWeaponInstance::GetAttachment(ELyraAttachmentSlot Slot) const
{
    return AttachedAttachments.FindRef(Slot);
}

bool ULyraWeaponInstance::IsAttachmentCompatible(ULyraWeaponAttachment* Attachment) const
{
    if (!Attachment)
    {
        return false;
    }

    // 检查武器标签是否匹配
    return WeaponTags.HasAny(Attachment->CompatibleWeaponTags);
}

float ULyraWeaponInstance::GetModifiedDamage() const
{
    float ModifiedDamage = Damage;
    for (const auto& Pair : AttachedAttachments)
    {
        if (Pair.Value)
        {
            ModifiedDamage *= Pair.Value->StatModifier.DamageMultiplier;
        }
    }
    return ModifiedDamage;
}

float ULyraWeaponInstance::GetModifiedFireRate() const
{
    float ModifiedFireRate = FireRate;
    for (const auto& Pair : AttachedAttachments)
    {
        if (Pair.Value)
        {
            ModifiedFireRate *= Pair.Value->StatModifier.FireRateMultiplier;
        }
    }
    return ModifiedFireRate;
}

float ULyraWeaponInstance::GetModifiedRecoilHorizontal() const
{
    float ModifiedRecoil = RecoilHorizontal;
    for (const auto& Pair : AttachedAttachments)
    {
        if (Pair.Value)
        {
            ModifiedRecoil *= Pair.Value->StatModifier.RecoilHorizontalMultiplier;
        }
    }
    return ModifiedRecoil;
}

float ULyraWeaponInstance::GetModifiedRecoilVertical() const
{
    float ModifiedRecoil = RecoilVertical;
    for (const auto& Pair : AttachedAttachments)
    {
        if (Pair.Value)
        {
            ModifiedRecoil *= Pair.Value->StatModifier.RecoilVerticalMultiplier;
        }
    }
    return ModifiedRecoil;
}

float ULyraWeaponInstance::GetModifiedSpread() const
{
    float ModifiedSpread = BaseSpread;
    for (const auto& Pair : AttachedAttachments)
    {
        if (Pair.Value)
        {
            ModifiedSpread *= Pair.Value->StatModifier.SpreadMultiplier;
        }
    }
    return ModifiedSpread;
}

int32 ULyraWeaponInstance::GetModifiedMaxAmmo() const
{
    int32 ModifiedMaxAmmo = MaxAmmo;
    for (const auto& Pair : AttachedAttachments)
    {
        if (Pair.Value)
        {
            ModifiedMaxAmmo += Pair.Value->StatModifier.MagazineCapacityBonus;
        }
    }
    return ModifiedMaxAmmo;
}
```

### 2.5 武器皮肤与外观定制

皮肤系统允许玩家自定义武器外观，而不影响游戏性。

#### 2.5.1 皮肤定义

```cpp
// LyraWeaponSkin.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "LyraWeaponSkin.generated.h"

/**
 * 武器皮肤定义
 */
UCLASS(BlueprintType)
class LYRAGAME_API ULyraWeaponSkin : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 皮肤名称
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Skin")
    FText SkinName;

    // 皮肤描述
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Skin")
    FText SkinDescription;

    // 皮肤图标
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Skin")
    TObjectPtr<UTexture2D> SkinIcon;

    // 皮肤稀有度
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Skin")
    int32 Rarity = 1; // 1=普通, 2=稀有, 3=史诗, 4=传说

    // 武器材质（覆盖原武器材质）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Skin")
    TArray<TObjectPtr<UMaterialInterface>> Materials;

    // 特效（如粒子效果）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Skin")
    TSubclassOf<AActor> VisualEffect;
};
```

#### 2.5.2 扩展武器实例支持皮肤

```cpp
// LyraWeaponInstance.h 扩展

// 当前皮肤
UPROPERTY(BlueprintReadOnly, Replicated, Category = "Weapon|Skin")
TObjectPtr<ULyraWeaponSkin> CurrentSkin;

// 应用皮肤
UFUNCTION(BlueprintCallable, Category = "Weapon|Skin")
void ApplySkin(ULyraWeaponSkin* Skin);

// 移除皮肤
UFUNCTION(BlueprintCallable, Category = "Weapon|Skin")
void RemoveSkin();
```

```cpp
// LyraWeaponInstance.cpp 实现

void ULyraWeaponInstance::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    // ... 其他属性
    DOREPLIFETIME(ThisClass, CurrentSkin);
}

void ULyraWeaponInstance::ApplySkin(ULyraWeaponSkin* Skin)
{
    if (!Skin || SpawnedActors.Num() == 0)
    {
        return;
    }

    CurrentSkin = Skin;

    AActor* WeaponActor = SpawnedActors[0];
    if (!WeaponActor)
    {
        return;
    }

    // 应用材质
    if (UStaticMeshComponent* MeshComp = WeaponActor->FindComponentByClass<UStaticMeshComponent>())
    {
        for (int32 i = 0; i < Skin->Materials.Num(); ++i)
        {
            MeshComp->SetMaterial(i, Skin->Materials[i]);
        }
    }

    // 生成特效
    if (Skin->VisualEffect)
    {
        AActor* EffectActor = GetWorld()->SpawnActor<AActor>(
            Skin->VisualEffect,
            WeaponActor->GetActorTransform()
        );
        EffectActor->AttachToActor(WeaponActor, FAttachmentTransformRules::SnapToTargetIncludingScale);
    }
}

void ULyraWeaponInstance::RemoveSkin()
{
    CurrentSkin = nullptr;
    // TODO: 恢复默认材质
}
```

---

## 第三部分：与 GAS 的集成（20%）

### 3.1 装备如何授予 Ability

装备系统与 GAS 深度集成，装备时自动授予技能，卸下时移除技能。

#### 3.1.1 Ability Set 定义

```cpp
// LyraAbilitySet.h（Lyra 已有，这里展示关键部分）

USTRUCT(BlueprintType)
struct FLyraAbilitySet_GameplayAbility
{
    GENERATED_BODY()

    // 授予的 Ability 类
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<ULyraGameplayAbility> Ability = nullptr;

    // Ability 等级
    UPROPERTY(EditDefaultsOnly)
    int32 AbilityLevel = 1;

    // 输入标签（绑定到输入）
    UPROPERTY(EditDefaultsOnly, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};

USTRUCT(BlueprintType)
struct FLyraAbilitySet_GameplayEffect
{
    GENERATED_BODY()

    // 授予的 Gameplay Effect 类
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UGameplayEffect> GameplayEffect = nullptr;

    // Effect 等级
    UPROPERTY(EditDefaultsOnly)
    float EffectLevel = 1.0f;
};

USTRUCT(BlueprintType)
struct FLyraAbilitySet_AttributeSet
{
    GENERATED_BODY()

    // 授予的 Attribute Set 类
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UAttributeSet> AttributeSet;
};

/**
 * Ability Set：一组要授予的 Abilities、Effects、Attributes
 */
UCLASS(BlueprintType, Const)
class ULyraAbilitySet : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 授予的 Abilities
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Abilities")
    TArray<FLyraAbilitySet_GameplayAbility> GrantedGameplayAbilities;

    // 授予的 Gameplay Effects
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Effects")
    TArray<FLyraAbilitySet_GameplayEffect> GrantedGameplayEffects;

    // 授予的 Attribute Sets
    UPROPERTY(EditDefaultsOnly, Category = "Attribute Sets")
    TArray<FLyraAbilitySet_AttributeSet> GrantedAttributes;

    // 授予到 ASC
    void GiveToAbilitySystem(ULyraAbilitySystemComponent* ASC, FGameplayAbilitySpecHandles* OutGrantedHandles, UObject* SourceObject = nullptr) const;
};
```

#### 3.1.2 授予 Ability 实现

```cpp
// LyraAbilitySet.cpp（简化版）

void ULyraAbilitySet::GiveToAbilitySystem(ULyraAbilitySystemComponent* ASC, FGameplayAbilitySpecHandles* OutGrantedHandles, UObject* SourceObject) const
{
    check(ASC);

    if (!ASC->IsOwnerActorAuthoritative())
    {
        return; // 只在服务器上授予
    }

    // 授予 Abilities
    for (const FLyraAbilitySet_GameplayAbility& AbilityToGrant : GrantedGameplayAbilities)
    {
        if (!IsValid(AbilityToGrant.Ability))
        {
            continue;
        }

        ULyraGameplayAbility* AbilityCDO = AbilityToGrant.Ability->GetDefaultObject<ULyraGameplayAbility>();

        FGameplayAbilitySpec AbilitySpec(AbilityCDO, AbilityToGrant.AbilityLevel);
        AbilitySpec.SourceObject = SourceObject;
        AbilitySpec.DynamicAbilityTags.AddTag(AbilityToGrant.InputTag);

        const FGameplayAbilitySpecHandle AbilitySpecHandle = ASC->GiveAbility(AbilitySpec);

        if (OutGrantedHandles)
        {
            OutGrantedHandles->AddAbilitySpecHandle(AbilitySpecHandle);
        }
    }

    // 授予 Gameplay Effects
    for (const FLyraAbilitySet_GameplayEffect& EffectToGrant : GrantedGameplayEffects)
    {
        if (!IsValid(EffectToGrant.GameplayEffect))
        {
            continue;
        }

        const UGameplayEffect* EffectCDO = EffectToGrant.GameplayEffect->GetDefaultObject<UGameplayEffect>();
        const FActiveGameplayEffectHandle EffectHandle = ASC->ApplyGameplayEffectToSelf(EffectCDO, EffectToGrant.EffectLevel, ASC->MakeEffectContext());

        if (OutGrantedHandles)
        {
            OutGrantedHandles->AddGameplayEffectHandle(EffectHandle);
        }
    }

    // 授予 Attribute Sets
    for (const FLyraAbilitySet_AttributeSet& SetToGrant : GrantedAttributes)
    {
        if (!IsValid(SetToGrant.AttributeSet))
        {
            continue;
        }

        UAttributeSet* NewSet = NewObject<UAttributeSet>(ASC->GetOwner(), SetToGrant.AttributeSet);
        ASC->AddAttributeSetSubobject(NewSet);
    }
}
```

#### 3.1.3 武器 Ability Set 示例

创建一个步枪 Ability Set：

```cpp
// Content/AbilitySets/AbilitySet_Weapon_Rifle.uasset (蓝图配置)

Granted Gameplay Abilities:
  [0]
    Ability: GA_Weapon_Fire_Rifle
    AbilityLevel: 1
    InputTag: InputTag.Ability.Fire

  [1]
    Ability: GA_Weapon_Reload
    AbilityLevel: 1
    InputTag: InputTag.Ability.Reload

  [2]
    Ability: GA_Weapon_Aim
    AbilityLevel: 1
    InputTag: InputTag.Ability.Aim

Granted Gameplay Effects:
  [0]
    GameplayEffect: GE_Weapon_Rifle_Stats
    EffectLevel: 1.0
```

### 3.2 装备属性修改（攻击力、射程）

装备可以通过 Gameplay Effects 修改角色属性。

#### 3.2.1 武器属性 Gameplay Effect

```cpp
// GE_Weapon_Rifle_Stats (蓝图配置)

Duration Policy: Instant

Modifiers:
  [0]
    Attribute: LyraHealthSet.AttackPower
    Modifier Op: Add
    Magnitude: 25.0

  [1]
    Attribute: LyraHealthSet.CritChance
    Modifier Op: Add
    Magnitude: 0.15 (15% 暴击率)
```

#### 3.2.2 动态属性修改（基于附件）

```cpp
// 在 ULyraWeaponInstance 中添加动态 GE

void ULyraWeaponInstance::OnEquipped()
{
    Super::OnEquipped();

    // 应用基础武器属性 GE
    // ...

    // 应用附件属性 GE
    ApplyAttachmentEffects();
}

void ULyraWeaponInstance::ApplyAttachmentEffects()
{
    ULyraAbilitySystemComponent* ASC = GetAbilitySystemComponent();
    if (!ASC)
    {
        return;
    }

    for (const auto& Pair : AttachedAttachments)
    {
        if (ULyraWeaponAttachment* Attachment = Pair.Value)
        {
            // 创建动态 GE 修改属性
            UGameplayEffect* DynamicGE = NewObject<UGameplayEffect>();
            DynamicGE->DurationPolicy = EGameplayEffectDurationType::Infinite;

            // 添加伤害修改器
            FGameplayModifierInfo DamageModifier;
            DamageModifier.ModifierMagnitude = FScalableFloat(Attachment->StatModifier.DamageMultiplier);
            DamageModifier.ModifierOp = EGameplayModOp::Multiply;
            DamageModifier.Attribute = ULyraHealthSet::GetAttackPowerAttribute();
            DynamicGE->Modifiers.Add(DamageModifier);

            // 应用 GE
            FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
            EffectContext.AddSourceObject(this);
            FActiveGameplayEffectHandle EffectHandle = ASC->ApplyGameplayEffectToSelf(
                DynamicGE,
                1.0f,
                EffectContext
            );

            AttachmentEffectHandles.Add(Pair.Key, EffectHandle);
        }
    }
}
```

### 3.3 装备 Gameplay Tags 管理

装备可以授予或移除 Gameplay Tags，影响角色行为。

#### 3.3.1 装备标签定义

```cpp
// 在 ULyraEquipmentDefinition 中添加
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Tags")
FGameplayTagContainer GrantedTags;

UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Tags")
FGameplayTagContainer BlockedTags;
```

#### 3.3.2 应用装备标签

```cpp
// LyraEquipmentManagerComponent.cpp 中修改 EquipItem 函数

ULyraEquipmentInstance* ULyraEquipmentManagerComponent::EquipItem(const ULyraEquipmentDefinition* EquipmentDefinition)
{
    // ... 前面的代码

    if (Result != nullptr)
    {
        // ... 生成 Actors、授予 Abilities

        // 授予标签
        if (ULyraAbilitySystemComponent* ASC = GetAbilitySystemComponent())
        {
            for (const FGameplayTag& Tag : EquipmentDefinition->GrantedTags)
            {
                ASC->AddLooseGameplayTag(Tag);
            }

            for (const FGameplayTag& Tag : EquipmentDefinition->BlockedTags)
            {
                ASC->BlockAbilitiesWithTags(FGameplayTagContainer(Tag));
            }
        }

        Result->OnEquipped();
    }

    return Result;
}
```

#### 3.3.3 标签应用示例

```cpp
// 步枪装备定义

Granted Tags:
  - State.Weapon.Equipped
  - State.Weapon.Rifle

Blocked Tags:
  - Ability.Melee (装备步枪时不能使用近战)
```

### 3.4 装备切换的网络同步

装备系统使用 Fast Array 实现高效的网络同步。

#### 3.4.1 Fast Array 复制原理

```cpp
// Fast Array 的优势：
// 1. 只同步变化的元素（Delta Replication）
// 2. 自动处理添加、删除、修改事件
// 3. 高效的带宽利用

// FLyraEquipmentList 使用 Fast Array
USTRUCT(BlueprintType)
struct FLyraEquipmentList : public FFastArraySerializer
{
    GENERATED_BODY()

    // 重要的回调函数
    void PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize);
    void PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize);
    void PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize);

    // 序列化函数
    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms);
};
```

#### 3.4.2 客户端预测与服务器校验

```cpp
// 客户端请求装备武器（通过 RPC）

// LyraEquipmentManagerComponent.h
UFUNCTION(Server, Reliable, WithValidation, BlueprintCallable, Category = Equipment)
void ServerEquipItem(const ULyraEquipmentDefinition* EquipmentDefinition);

// LyraEquipmentManagerComponent.cpp
void ULyraEquipmentManagerComponent::ServerEquipItem_Implementation(const ULyraEquipmentDefinition* EquipmentDefinition)
{
    EquipItem(EquipmentDefinition);
}

bool ULyraEquipmentManagerComponent::ServerEquipItem_Validate(const ULyraEquipmentDefinition* EquipmentDefinition)
{
    // 验证装备定义是否有效
    return EquipmentDefinition != nullptr;
}

// 客户端调用
void AMyPlayerController::RequestEquipWeapon(const ULyraEquipmentDefinition* WeaponDef)
{
    if (ULyraEquipmentManagerComponent* EquipmentManager = GetPawn()->FindComponentByClass<ULyraEquipmentManagerComponent>())
    {
        EquipmentManager->ServerEquipItem(WeaponDef);
    }
}
```

---

## 第四部分：实战案例（25%）

### 4.1 完整的 FPS 武器系统实现

现在我们来实现一个完整的 FPS 武器系统，包括开火、换弹、瞄准、后坐力等所有功能。

#### 4.1.1 开火 Ability

```cpp
// GA_Weapon_Fire.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_Weapon_Fire.generated.h"

class ULyraWeaponInstance;

/**
 * 武器开火 Ability
 */
UCLASS()
class LYRAGAME_API UGA_Weapon_Fire : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Weapon_Fire(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    //~UGameplayAbility interface
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;
    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, FGameplayTagContainer* OptionalRelevantTags) const override;
    //~End of UGameplayAbility interface

protected:
    // 单次射击
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void FireWeapon();

    // 执行射线检测
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    FHitResult PerformTrace();

    // 应用伤害
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void ApplyDamage(const FHitResult& HitResult);

    // 应用后坐力
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void ApplyRecoil();

    // 射击间隔定时器
    FTimerHandle FireTimerHandle;

    // 是否在连续射击
    bool bIsFiring = false;

    // 伤害 Gameplay Effect
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TSubclassOf<UGameplayEffect> DamageGameplayEffect;

    // 枪口火焰特效
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TObjectPtr<UParticleSystem> MuzzleFlashEffect;

    // 弹孔贴花
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TObjectPtr<UMaterialInterface> ImpactDecal;

    // 射击音效
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TObjectPtr<USoundBase> FireSound;
};
```

```cpp
// GA_Weapon_Fire.cpp

#include "GA_Weapon_Fire.h"
#include "AbilitySystemComponent.h"
#include "Equipment/LyraWeaponInstance.h"
#include "Equipment/LyraEquipmentManagerComponent.h"
#include "Character/LyraCharacter.h"
#include "Camera/CameraComponent.h"
#include "Kismet/GameplayStatics.h"
#include "DrawDebugHelpers.h"

UGA_Weapon_Fire::UGA_Weapon_Fire(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

bool UGA_Weapon_Fire::CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, FGameplayTagContainer* OptionalRelevantTags) const
{
    if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
    {
        return false;
    }

    // 检查是否有武器实例
    if (ALyraCharacter* Character = Cast<ALyraCharacter>(ActorInfo->AvatarActor.Get()))
    {
        if (ULyraEquipmentManagerComponent* EquipmentManager = Character->FindComponentByClass<ULyraEquipmentManagerComponent>())
        {
            ULyraWeaponInstance* WeaponInstance = EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>();
            if (WeaponInstance && WeaponInstance->CanFire())
            {
                return true;
            }
        }
    }

    return false;
}

void UGA_Weapon_Fire::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    bIsFiring = true;

    // 立即开火一次
    FireWeapon();

    // 设置连续射击定时器
    if (ALyraCharacter* Character = Cast<ALyraCharacter>(ActorInfo->AvatarActor.Get()))
    {
        if (ULyraEquipmentManagerComponent* EquipmentManager = Character->FindComponentByClass<ULyraEquipmentManagerComponent>())
        {
            if (ULyraWeaponInstance* WeaponInstance = EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>())
            {
                float FireRate = WeaponInstance->GetModifiedFireRate(); // 每分钟射击次数
                float TimeBetweenShots = 60.0f / FireRate; // 秒

                GetWorld()->GetTimerManager().SetTimer(
                    FireTimerHandle,
                    this,
                    &UGA_Weapon_Fire::FireWeapon,
                    TimeBetweenShots,
                    true
                );
            }
        }
    }
}

void UGA_Weapon_Fire::EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled)
{
    bIsFiring = false;

    // 清除定时器
    GetWorld()->GetTimerManager().ClearTimer(FireTimerHandle);

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}

void UGA_Weapon_Fire::FireWeapon()
{
    if (!bIsFiring)
    {
        return;
    }

    ALyraCharacter* Character = Cast<ALyraCharacter>(GetAvatarActorFromActorInfo());
    if (!Character)
    {
        return;
    }

    ULyraEquipmentManagerComponent* EquipmentManager = Character->FindComponentByClass<ULyraEquipmentManagerComponent>();
    if (!EquipmentManager)
    {
        return;
    }

    ULyraWeaponInstance* WeaponInstance = EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>();
    if (!WeaponInstance || !WeaponInstance->CanFire())
    {
        EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
        return;
    }

    // 消耗弹药
    if (!WeaponInstance->ConsumeAmmo(1))
    {
        EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
        return;
    }

    // 执行射线检测
    FHitResult HitResult = PerformTrace();

    // 应用伤害
    if (HitResult.bBlockingHit)
    {
        ApplyDamage(HitResult);
    }

    // 应用后坐力
    ApplyRecoil();

    // 播放特效
    if (MuzzleFlashEffect)
    {
        // 获取武器的枪口位置
        if (WeaponInstance->GetSpawnedActors().Num() > 0)
        {
            AActor* WeaponActor = WeaponInstance->GetSpawnedActors()[0];
            USceneComponent* MuzzleSocket = WeaponActor->GetRootComponent();
            UGameplayStatics::SpawnEmitterAttached(MuzzleFlashEffect, MuzzleSocket, TEXT("Muzzle"));
        }
    }

    // 播放音效
    if (FireSound)
    {
        UGameplayStatics::PlaySoundAtLocation(GetWorld(), FireSound, Character->GetActorLocation());
    }

    // 生成弹孔
    if (HitResult.bBlockingHit && ImpactDecal)
    {
        UGameplayStatics::SpawnDecalAtLocation(
            GetWorld(),
            ImpactDecal,
            FVector(10.0f, 10.0f, 10.0f),
            HitResult.Location,
            HitResult.ImpactNormal.Rotation(),
            10.0f
        );
    }
}

FHitResult UGA_Weapon_Fire::PerformTrace()
{
    ALyraCharacter* Character = Cast<ALyraCharacter>(GetAvatarActorFromActorInfo());
    ULyraEquipmentManagerComponent* EquipmentManager = Character->FindComponentByClass<ULyraEquipmentManagerComponent>();
    ULyraWeaponInstance* WeaponInstance = EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>();

    // 从相机位置开始射线检测
    UCameraComponent* CameraComp = Character->FindComponentByClass<UCameraComponent>();
    FVector StartLocation = CameraComp->GetComponentLocation();
    FVector ForwardVector = CameraComp->GetForwardVector();

    // 添加扩散（精度影响）
    float Spread = WeaponInstance->GetModifiedSpread();
    ForwardVector.X += FMath::RandRange(-Spread, Spread);
    ForwardVector.Y += FMath::RandRange(-Spread, Spread);
    ForwardVector.Normalize();

    FVector EndLocation = StartLocation + (ForwardVector * WeaponInstance->Range);

    // 执行射线检测
    FHitResult HitResult;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(Character);

    GetWorld()->LineTraceSingleByChannel(
        HitResult,
        StartLocation,
        EndLocation,
        ECC_Visibility,
        QueryParams
    );

    // 绘制调试线
    DrawDebugLine(GetWorld(), StartLocation, HitResult.bBlockingHit ? HitResult.Location : EndLocation, FColor::Red, false, 1.0f, 0, 1.0f);

    return HitResult;
}

void UGA_Weapon_Fire::ApplyDamage(const FHitResult& HitResult)
{
    AActor* HitActor = HitResult.GetActor();
    if (!HitActor)
    {
        return;
    }

    // 检查目标是否有 ASC
    UAbilitySystemComponent* TargetASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(HitActor);
    if (!TargetASC)
    {
        return;
    }

    ULyraEquipmentManagerComponent* EquipmentManager = GetAvatarActorFromActorInfo()->FindComponentByClass<ULyraEquipmentManagerComponent>();
    ULyraWeaponInstance* WeaponInstance = EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>();

    // 创建 Gameplay Effect Context
    FGameplayEffectContextHandle EffectContext = GetAbilitySystemComponentFromActorInfo()->MakeEffectContext();
    EffectContext.AddSourceObject(WeaponInstance);
    EffectContext.AddHitResult(HitResult);

    // 创建 Gameplay Effect Spec
    FGameplayEffectSpecHandle SpecHandle = GetAbilitySystemComponentFromActorInfo()->MakeOutgoingSpec(
        DamageGameplayEffect,
        GetAbilityLevel(),
        EffectContext
    );

    if (SpecHandle.IsValid())
    {
        // 设置伤害值（从武器实例获取）
        SpecHandle.Data->SetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag(TEXT("Data.Damage")), WeaponInstance->GetModifiedDamage());

        // 应用 Gameplay Effect
        GetAbilitySystemComponentFromActorInfo()->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
    }
}

void UGA_Weapon_Fire::ApplyRecoil()
{
    ALyraCharacter* Character = Cast<ALyraCharacter>(GetAvatarActorFromActorInfo());
    ULyraEquipmentManagerComponent* EquipmentManager = Character->FindComponentByClass<ULyraEquipmentManagerComponent>();
    ULyraWeaponInstance* WeaponInstance = EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>();

    // 计算后坐力
    float RecoilPitch = -WeaponInstance->GetModifiedRecoilVertical();
    float RecoilYaw = FMath::RandRange(-WeaponInstance->GetModifiedRecoilHorizontal(), WeaponInstance->GetModifiedRecoilHorizontal());

    // 应用到控制器旋转
    if (AController* Controller = Character->GetController())
    {
        Controller->AddPitchInput(RecoilPitch);
        Controller->AddYawInput(RecoilYaw);
    }
}
```

#### 4.1.2 换弹 Ability

```cpp
// GA_Weapon_Reload.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_Weapon_Reload.generated.h"

/**
 * 武器换弹 Ability
 */
UCLASS()
class LYRAGAME_API UGA_Weapon_Reload : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Weapon_Reload(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    //~UGameplayAbility interface
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, FGameplayTagContainer* OptionalRelevantTags) const override;
    //~End of UGameplayAbility interface

protected:
    // 换弹动画蒙太奇
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TObjectPtr<UAnimMontage> ReloadMontage;

    // 换弹完成回调
    UFUNCTION()
    void OnReloadComplete();
};
```

```cpp
// GA_Weapon_Reload.cpp

#include "GA_Weapon_Reload.h"
#include "Equipment/LyraWeaponInstance.h"
#include "Equipment/LyraEquipmentManagerComponent.h"
#include "Character/LyraCharacter.h"
#include "AbilitySystemComponent.h"

UGA_Weapon_Reload::UGA_Weapon_Reload(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

bool UGA_Weapon_Reload::CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, FGameplayTagContainer* OptionalRelevantTags) const
{
    if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
    {
        return false;
    }

    // 检查是否可以换弹
    if (ALyraCharacter* Character = Cast<ALyraCharacter>(ActorInfo->AvatarActor.Get()))
    {
        if (ULyraEquipmentManagerComponent* EquipmentManager = Character->FindComponentByClass<ULyraEquipmentManagerComponent>())
        {
            ULyraWeaponInstance* WeaponInstance = EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>();
            if (WeaponInstance && WeaponInstance->CanReload())
            {
                return true;
            }
        }
    }

    return false;
}

void UGA_Weapon_Reload::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    ALyraCharacter* Character = Cast<ALyraCharacter>(GetAvatarActorFromActorInfo());
    if (!Character)
    {
        EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, true);
        return;
    }

    // 播放换弹动画
    if (ReloadMontage)
    {
        Character->PlayAnimMontage(ReloadMontage);
    }

    // 设置换弹定时器（2 秒）
    FTimerHandle ReloadTimerHandle;
    GetWorld()->GetTimerManager().SetTimer(
        ReloadTimerHandle,
        this,
        &UGA_Weapon_Reload::OnReloadComplete,
        2.0f,
        false
    );
}

void UGA_Weapon_Reload::OnReloadComplete()
{
    ALyraCharacter* Character = Cast<ALyraCharacter>(GetAvatarActorFromActorInfo());
    ULyraEquipmentManagerComponent* EquipmentManager = Character->FindComponentByClass<ULyraEquipmentManagerComponent>();
    ULyraWeaponInstance* WeaponInstance = EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>();

    if (WeaponInstance)
    {
        WeaponInstance->Reload();
    }

    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}
```

#### 4.1.3 瞄准 Ability

```cpp
// GA_Weapon_Aim.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_Weapon_Aim.generated.h"

/**
 * 武器瞄准 Ability
 */
UCLASS()
class LYRAGAME_API UGA_Weapon_Aim : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Weapon_Aim(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    //~UGameplayAbility interface
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;
    //~End of UGameplayAbility interface

protected:
    // 瞄准时的 FOV
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    float AimFOV = 60.0f;

    // 正常 FOV
    float NormalFOV = 90.0f;

    // FOV 插值速度
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    float FOVInterpSpeed = 10.0f;
};
```

```cpp
// GA_Weapon_Aim.cpp

#include "GA_Weapon_Aim.h"
#include "Character/LyraCharacter.h"
#include "Camera/CameraComponent.h"
#include "Equipment/LyraWeaponInstance.h"
#include "Equipment/LyraEquipmentManagerComponent.h"

UGA_Weapon_Aim::UGA_Weapon_Aim(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalOnly; // 瞄准只在本地执行
}

void UGA_Weapon_Aim::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    ALyraCharacter* Character = Cast<ALyraCharacter>(GetAvatarActorFromActorInfo());
    if (!Character)
    {
        EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, true);
        return;
    }

    // 保存正常 FOV
    if (UCameraComponent* CameraComp = Character->FindComponentByClass<UCameraComponent>())
    {
        NormalFOV = CameraComp->FieldOfView;
    }

    // 应用瞄准标签（影响武器扩散）
    GetAbilitySystemComponentFromActorInfo()->AddLooseGameplayTag(FGameplayTag::RequestGameplayTag(TEXT("State.Weapon.Aiming")));
}

void UGA_Weapon_Aim::EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled)
{
    ALyraCharacter* Character = Cast<ALyraCharacter>(GetAvatarActorFromActorInfo());
    if (Character)
    {
        // 恢复正常 FOV（在 Tick 中插值）
        if (UCameraComponent* CameraComp = Character->FindComponentByClass<UCameraComponent>())
        {
            CameraComp->SetFieldOfView(NormalFOV);
        }
    }

    // 移除瞄准标签
    GetAbilitySystemComponentFromActorInfo()->RemoveLooseGameplayTag(FGameplayTag::RequestGameplayTag(TEXT("State.Weapon.Aiming")));

    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

### 4.2 近战武器与远程武器对比

#### 4.2.1 近战武器实现

```cpp
// GA_Weapon_Melee.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystem/Abilities/LyraGameplayAbility.h"
#include "GA_Weapon_Melee.generated.h"

/**
 * 近战武器攻击 Ability
 */
UCLASS()
class LYRAGAME_API UGA_Weapon_Melee : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Weapon_Melee(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;

protected:
    // 攻击范围
    UPROPERTY(EditDefaultsOnly, Category = "Melee")
    float AttackRange = 200.0f;

    // 攻击角度
    UPROPERTY(EditDefaultsOnly, Category = "Melee")
    float AttackAngle = 60.0f;

    // 伤害值
    UPROPERTY(EditDefaultsOnly, Category = "Melee")
    float Damage = 50.0f;

    // 攻击动画
    UPROPERTY(EditDefaultsOnly, Category = "Melee")
    TObjectPtr<UAnimMontage> AttackMontage;

    // 执行近战检测
    void PerformMeleeTrace();
};
```

```cpp
// GA_Weapon_Melee.cpp

#include "GA_Weapon_Melee.h"
#include "Character/LyraCharacter.h"
#include "AbilitySystemComponent.h"
#include "Kismet/KismetSystemLibrary.h"

UGA_Weapon_Melee::UGA_Weapon_Melee(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
}

void UGA_Weapon_Melee::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    ALyraCharacter* Character = Cast<ALyraCharacter>(GetAvatarActorFromActorInfo());
    if (!Character)
    {
        EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, true);
        return;
    }

    // 播放攻击动画
    if (AttackMontage)
    {
        Character->PlayAnimMontage(AttackMontage);
    }

    // 延迟执行近战检测（等待动画播放到合适帧）
    FTimerHandle MeleeTimerHandle;
    GetWorld()->GetTimerManager().SetTimer(
        MeleeTimerHandle,
        this,
        &UGA_Weapon_Melee::PerformMeleeTrace,
        0.3f,
        false
    );
}

void UGA_Weapon_Melee::PerformMeleeTrace()
{
    ALyraCharacter* Character = Cast<ALyraCharacter>(GetAvatarActorFromActorInfo());
    if (!Character)
    {
        return;
    }

    FVector StartLocation = Character->GetActorLocation();
    FVector ForwardVector = Character->GetActorForwardVector();
    FVector EndLocation = StartLocation + (ForwardVector * AttackRange);

    // 使用球形扫描检测近战范围内的敌人
    TArray<FHitResult> HitResults;
    TArray<AActor*> ActorsToIgnore;
    ActorsToIgnore.Add(Character);

    UKismetSystemLibrary::SphereTraceMulti(
        GetWorld(),
        StartLocation,
        EndLocation,
        50.0f,
        UEngineTypes::ConvertToTraceType(ECC_Pawn),
        false,
        ActorsToIgnore,
        EDrawDebugTrace::ForDuration,
        HitResults,
        true
    );

    // 对每个命中目标应用伤害
    for (const FHitResult& HitResult : HitResults)
    {
        AActor* HitActor = HitResult.GetActor();
        if (!HitActor)
        {
            continue;
        }

        // 检查角度（只攻击前方的敌人）
        FVector DirectionToTarget = (HitActor->GetActorLocation() - Character->GetActorLocation()).GetSafeNormal();
        float DotProduct = FVector::DotProduct(ForwardVector, DirectionToTarget);
        float AngleToTarget = FMath::Acos(DotProduct) * (180.0f / PI);

        if (AngleToTarget > AttackAngle)
        {
            continue; // 超出攻击角度
        }

        // 应用伤害（通过 Gameplay Effect）
        UAbilitySystemComponent* TargetASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(HitActor);
        if (TargetASC)
        {
            // TODO: 应用伤害 GE
        }
    }

    // 结束 Ability
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}
```

#### 4.2.2 远程与近战对比表

| 特性           | 远程武器（Ranged）               | 近战武器（Melee）                |
|----------------|----------------------------------|----------------------------------|
| **检测方式**   | Line Trace（射线检测）           | Sphere Trace（球形扫描）         |
| **伤害范围**   | 单体（精确命中）                 | 范围（扇形或球形）               |
| **弹药系统**   | 需要弹药和换弹                   | 无弹药限制                       |
| **状态机**     | 复杂（待机、射击、换弹、瞄准）   | 简单（待机、攻击）               |
| **网络同步**   | 需要预测和校验                   | 较少网络同步需求                 |
| **伤害计算**   | 距离衰减、精度影响               | 固定伤害、连击系统               |

### 4.3 装备升级系统

装备升级系统允许玩家提升装备属性。

#### 4.3.1 装备等级系统

```cpp
// LyraEquipmentInstance.h 扩展

// 装备等级
UPROPERTY(BlueprintReadOnly, Replicated, Category = "Equipment|Level")
int32 EquipmentLevel = 1;

// 最大等级
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment|Level")
int32 MaxLevel = 10;

// 升级装备
UFUNCTION(BlueprintCallable, Category = "Equipment|Level")
bool UpgradeEquipment();

// 获取升级所需经验
UFUNCTION(BlueprintPure, Category = "Equipment|Level")
int32 GetRequiredExperienceForNextLevel() const;
```

```cpp
// LyraEquipmentInstance.cpp 实现

void ULyraEquipmentInstance::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(ThisClass, EquipmentLevel);
}

bool ULyraEquipmentInstance::UpgradeEquipment()
{
    if (EquipmentLevel >= MaxLevel)
    {
        return false; // 已达到最大等级
    }

    EquipmentLevel++;

    // 重新计算属性（通过 GE）
    // TODO: 应用升级 GE

    return true;
}

int32 ULyraEquipmentInstance::GetRequiredExperienceForNextLevel() const
{
    // 指数增长的经验需求
    return 100 * FMath::Pow(EquipmentLevel, 1.5f);
}
```

#### 4.3.2 武器升级属性加成

```cpp
// ULyraWeaponInstance 扩展

// 获取等级加成后的伤害
float ULyraWeaponInstance::GetModifiedDamage() const
{
    float ModifiedDamage = Damage;

    // 附件加成
    for (const auto& Pair : AttachedAttachments)
    {
        if (Pair.Value)
        {
            ModifiedDamage *= Pair.Value->StatModifier.DamageMultiplier;
        }
    }

    // 等级加成（每级 +5% 伤害）
    ModifiedDamage *= (1.0f + (EquipmentLevel - 1) * 0.05f);

    return ModifiedDamage;
}
```

### 4.4 装备品质与稀有度

品质系统为装备添加稀有度等级，影响属性和外观。

#### 4.4.1 品质枚举

```cpp
// LyraEquipmentRarity.h
#pragma once

#include "CoreMinimal.h"

/**
 * 装备品质枚举
 */
UENUM(BlueprintType)
enum class ELyraEquipmentRarity : uint8
{
    Common      UMETA(DisplayName = "Common"),      // 普通（白色）
    Uncommon    UMETA(DisplayName = "Uncommon"),    // 非凡（绿色）
    Rare        UMETA(DisplayName = "Rare"),        // 稀有（蓝色）
    Epic        UMETA(DisplayName = "Epic"),        // 史诗（紫色）
    Legendary   UMETA(DisplayName = "Legendary")    // 传说（橙色）
};
```

#### 4.4.2 扩展装备定义支持品质

```cpp
// LyraEquipmentDefinition.h 扩展

// 装备品质
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment")
ELyraEquipmentRarity Rarity = ELyraEquipmentRarity::Common;

// 品质属性倍率
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment")
float RarityStatMultiplier = 1.0f;
```

#### 4.4.3 品质影响属性

```cpp
// ULyraWeaponInstance 实现

ULyraWeaponInstance* ULyraWeaponInstance::CreateFromDefinition(const ULyraEquipmentDefinition* Definition)
{
    // ... 创建实例

    // 根据品质调整属性
    switch (Definition->Rarity)
    {
        case ELyraEquipmentRarity::Common:
            Instance->Damage *= 1.0f;
            break;
        case ELyraEquipmentRarity::Uncommon:
            Instance->Damage *= 1.15f;
            break;
        case ELyraEquipmentRarity::Rare:
            Instance->Damage *= 1.3f;
            break;
        case ELyraEquipmentRarity::Epic:
            Instance->Damage *= 1.5f;
            break;
        case ELyraEquipmentRarity::Legendary:
            Instance->Damage *= 2.0f;
            break;
    }

    return Instance;
}
```

### 4.5 UI 集成（装备栏、武器 HUD）

#### 4.5.1 武器 HUD Widget

```cpp
// WBP_WeaponHUD.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "WBP_WeaponHUD.generated.h"

class ULyraWeaponInstance;
class UTextBlock;
class UProgressBar;
class UImage;

/**
 * 武器 HUD Widget（显示弹药、准星等）
 */
UCLASS()
class LYRAGAME_API UWBP_WeaponHUD : public UUserWidget
{
    GENERATED_BODY()

public:
    // 设置武器实例
    UFUNCTION(BlueprintCallable, Category = "Weapon HUD")
    void SetWeaponInstance(ULyraWeaponInstance* InWeaponInstance);

    // 更新 HUD
    UFUNCTION(BlueprintCallable, Category = "Weapon HUD")
    void UpdateHUD();

protected:
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;

    // UI 组件
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> AmmoText;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> ReserveAmmoText;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UProgressBar> ReloadProgress;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UImage> CrosshairImage;

private:
    UPROPERTY()
    TObjectPtr<ULyraWeaponInstance> WeaponInstance;
};
```

```cpp
// WBP_WeaponHUD.cpp

#include "WBP_WeaponHUD.h"
#include "Equipment/LyraWeaponInstance.h"
#include "Components/TextBlock.h"
#include "Components/ProgressBar.h"
#include "Components/Image.h"

void UWBP_WeaponHUD::SetWeaponInstance(ULyraWeaponInstance* InWeaponInstance)
{
    WeaponInstance = InWeaponInstance;
    UpdateHUD();
}

void UWBP_WeaponHUD::UpdateHUD()
{
    if (!WeaponInstance)
    {
        return;
    }

    // 更新弹药显示
    if (AmmoText)
    {
        AmmoText->SetText(FText::AsNumber(WeaponInstance->CurrentAmmo));
    }

    if (ReserveAmmoText)
    {
        ReserveAmmoText->SetText(FText::AsNumber(WeaponInstance->ReserveAmmo));
    }

    // 更新换弹进度条
    if (ReloadProgress)
    {
        // TODO: 从武器状态组件获取换弹进度
        ReloadProgress->SetPercent(0.0f);
    }
}

void UWBP_WeaponHUD::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    Super::NativeTick(MyGeometry, InDeltaTime);
    UpdateHUD();
}
```

#### 4.5.2 装备栏 Widget

```cpp
// WBP_EquipmentSlot.h（装备槽位 UI）

UCLASS()
class LYRAGAME_API UWBP_EquipmentSlot : public UUserWidget
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Category = "Equipment")
    void SetEquipmentInstance(ULyraEquipmentInstance* InEquipmentInstance);

protected:
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UImage> EquipmentIcon;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> EquipmentName;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UBorder> RarityBorder;

private:
    UPROPERTY()
    TObjectPtr<ULyraEquipmentInstance> EquipmentInstance;
};
```

---

## 总结

本文深度剖析了 Lyra 的装备与武器系统，涵盖了以下核心内容：

### 核心架构（25%）
- **装备系统设计理念**：数据与逻辑分离、模块化设计
- **Equipment Definition**：装备定义（Data Asset）
- **Equipment Instance**：装备运行时实例
- **Equipment Manager Component**：装备管理器
- **装备槽位系统**：基于 Gameplay Tags 的槽位管理
- **装备生命周期**：装备 → 激活 → 停用 → 卸下 → 销毁

### 武器系统（30%）
- **WeaponInstance 详解**：武器特化实例，支持弹药、属性、附件
- **武器状态机**：待机、射击、换弹、装备、卸下等状态管理
- **弹药系统**：弹夹管理、弹药类型、换弹逻辑
- **武器附件系统**：瞄具、握把、消音器等配件，动态修改武器属性
- **武器皮肤系统**：外观定制，不影响游戏性

### GAS 集成（20%）
- **装备授予 Ability**：通过 Ability Set 授予技能
- **装备属性修改**：通过 Gameplay Effects 修改角色属性
- **装备 Gameplay Tags**：影响角色行为和技能可用性
- **装备网络同步**：Fast Array 实现高效的网络复制

### 实战案例（25%）
- **完整的 FPS 武器系统**：开火、换弹、瞄准 Abilities 完整实现
- **近战与远程武器对比**：不同检测方式、伤害计算
- **装备升级系统**：等级系统、属性加成
- **装备品质与稀有度**：普通、稀有、史诗、传说等品质
- **UI 集成**：武器 HUD、装备栏、弹药显示

### 关键技术要点

1. **数据驱动设计**：通过 Data Assets 配置装备，无需修改代码
2. **模块化架构**：装备系统高度可扩展，易于添加新装备类型
3. **网络复制优化**：Fast Array 实现增量同步，减少带宽消耗
4. **GAS 深度集成**：装备通过 GAS 影响角色能力和属性
5. **状态机管理**：清晰的状态转换规则，避免非法操作
6. **客户端预测**：减少网络延迟对游戏体验的影响

### 最佳实践建议

- ✅ **使用 Data Assets 配置装备**：便于策划调整数值
- ✅ **分离数据与逻辑**：Equipment Definition 存配置，Equipment Instance 存状态
- ✅ **利用 GAS 管理装备效果**：统一的属性修改和技能授予
- ✅ **实现完善的网络同步**：Fast Array + RPC 确保客户端与服务器一致
- ✅ **设计清晰的状态机**：避免武器处于非法状态
- ✅ **提供扩展接口**：蓝图事件和虚函数方便子类扩展
- ✅ **性能优化**：装备实例是 UObject 而非 Actor，减少开销

### 扩展方向

基于本文的装备系统，您可以进一步扩展：

- **装备套装系统**：多件装备组合激活套装效果
- **装备强化系统**：消耗材料提升装备属性
- **装备镶嵌系统**：宝石镶嵌槽位，增加额外属性
- **装备耐久度系统**：装备使用后损耗，需要修理
- **装备绑定系统**：装备绑定角色，无法交易
- **装备交易系统**：玩家间交易装备
- **装备锻造系统**：合成新装备

Lyra 的装备系统为构建复杂的游戏玩法提供了坚实的基础，希望本文能帮助您深入理解并应用这套系统！

---

**字数统计**：约 20,000 字

**代码示例**：30+ 个完整的类定义和实现

**架构图**：装备系统架构图、状态机图、类图

**实战案例**：完整的 FPS 武器系统、近战武器、装备升级、品质系统、UI 集成

这篇文章提供了从理论到实践的完整指南，涵盖了 Lyra 装备与武器系统的所有核心知识点。🎯
