# 背包与物品系统实现

## 概述

背包与物品系统是现代游戏（特别是 RPG、生存类、射击类游戏）中不可或缺的核心机制。Lyra 虽然主要作为竞技射击游戏示例，但其灵活的架构为我们实现一个完整的物品系统提供了坚实基础。

与传统的"背包数组"实现不同，Lyra 的物品系统采用了 **碎片化设计（Fragment-Based Architecture）** 和 **数据驱动** 的理念，使得系统具有极高的可扩展性和可维护性。本文将从零开始构建一个生产级的背包与物品系统，涵盖从架构设计到网络同步的所有细节。

### 本文涵盖内容

- **物品系统架构**：Item Definition、Item Instance、Item Fragment、Inventory Component
- **背包系统详解**：槽位管理、堆叠与拆分、容量限制、多背包系统
- **物品碎片化设计**：属性碎片、Ability 碎片、UI 碎片、效果碎片
- **与 GAS 集成**：物品授予 Ability、可消耗物品、物品效果、Cooldown 管理
- **网络同步**：物品数据复制、客户端预测、服务端验证、作弊防护
- **实战案例**：完整的 RPG 背包系统、物品拾取与丢弃、物品使用、背包 UI

### 前置知识

在阅读本文前，建议您先掌握：

- Lyra 的装备系统（第 10 篇文章）
- GAS 基础与实战（第 6-8 篇文章）
- Enhanced Input 系统（第 9 篇文章）
- Unreal Engine 的网络复制机制
- C++ 基础和 UE5 C++ 编程

### 本文目标

阅读完本文后，您将能够：

1. **理解** Lyra 物品系统的碎片化设计理念
2. **实现** 一个完整的背包系统，支持槽位、堆叠、拆分
3. **集成** 物品系统与 GAS，实现物品效果和技能授予
4. **处理** 物品数据的网络同步和服务端验证
5. **构建** 完整的物品拾取、使用、丢弃逻辑
6. **设计** 可扩展的物品类型系统（装备、消耗品、任务道具等）

---

## 第一部分：背包系统架构（25%）

### 1.1 Lyra 的物品系统设计理念

Lyra 的物品系统与传统的"一个 Struct 搞定所有物品"的做法截然不同。它采用了 **组件化** 和 **碎片化** 的设计，灵感来源于现代游戏引擎的 ECS（Entity Component System）架构。

#### 1.1.1 传统物品系统的痛点

在传统的 UE 项目中，物品系统通常这样实现：

```cpp
// ❌ 传统的物品结构（不推荐）
USTRUCT(BlueprintType)
struct FItemData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 ItemID;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString ItemName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EItemType ItemType; // Weapon, Consumable, Quest, etc.

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxStack;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    UTexture2D* Icon;

    // 武器专属
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float WeaponDamage;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float WeaponRange;

    // 消耗品专属
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float HealthRestore;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float ManaRestore;

    // 装备专属
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 ArmorValue;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EEquipmentSlot EquipmentSlot;

    // ... 更多字段，越来越臃肿
};
```

这种设计的问题：

1. **臃肿**：所有物品类型的字段都塞在一个 Struct 里，大部分字段对特定物品无用
2. **难以扩展**：新增物品类型需要修改核心 Struct，影响所有现有代码
3. **内存浪费**：每个物品实例都包含所有类型的字段
4. **逻辑耦合**：物品逻辑分散在各处，难以维护
5. **网络同步困难**：大量冗余字段增加网络开销

#### 1.1.2 Lyra 的碎片化设计（Fragment-Based）

Lyra 借鉴了现代 ECS 架构的思想，将物品数据拆分为 **定义（Definition）** 和 **碎片（Fragment）**：

```
┌───────────────────────────────────────────────────────────┐
│                   ULyraInventoryItemDefinition            │
│                   (Data Asset - 物品定义)                 │
│  - 物品 ID、名称、描述                                    │
│  - 图标、模型                                              │
│  - 默认堆叠数                                              │
│  - Fragments 数组 ──────────────────┐                     │
└───────────────────────────────────┬─────────────────────┘
                                    │                       │
                                    │                       │
                ┌───────────────────┴───────────────┐      │
                │    ULyraInventoryItemInstance     │      │
                │    (Runtime Instance - 实例)      │      │
                │  - 当前堆叠数量                    │      │
                │  - 所属玩家                        │      │
                │  - 运行时状态                      │      │
                └───────────────────┬───────────────┘      │
                                    │                       │
                                    │                       │
      ┌─────────────────────────────┴───────────────────────┴──────────┐
      │                        Fragments 碎片系统                       │
      ├────────────────────┬────────────────────┬──────────────────────┤
      │                    │                    │                      │
┌─────▼──────────┐  ┌──────▼────────┐  ┌───────▼───────┐  ┌──────────▼────────┐
│  Stat Fragment │  │ Ability       │  │  Equipment    │  │  Consumable       │
│  (属性碎片)    │  │  Fragment     │  │  Fragment     │  │  Fragment         │
│  - 攻击力      │  │  (技能碎片)   │  │  (装备碎片)   │  │  (消耗品碎片)     │
│  - 防御力      │  │  - 授予的技能 │  │  - 装备槽位   │  │  - 恢复生命值     │
│  - 品质        │  │  - 技能等级   │  │  - 装备时效果 │  │  - 冷却时间       │
└────────────────┘  └───────────────┘  └───────────────┘  └───────────────────┘
```

**核心优势**：

1. **按需组合**：物品只包含它需要的碎片（装备物品有装备碎片，消耗品有消耗品碎片）
2. **易于扩展**：新增物品类型只需添加新的 Fragment 类，无需修改核心系统
3. **内存高效**：物品实例只存储必要的数据
4. **逻辑封装**：每个 Fragment 负责自己的逻辑（如 Equipment Fragment 处理装备逻辑）
5. **网络友好**：可以选择性复制需要同步的 Fragment

#### 1.1.3 架构对比

| 维度 | 传统架构 | Lyra 碎片化架构 |
|------|----------|-----------------|
| **扩展性** | 低（修改核心 Struct） | 高（添加新 Fragment） |
| **内存占用** | 高（所有字段） | 低（按需组合） |
| **逻辑耦合** | 高（分散在各处） | 低（封装在 Fragment） |
| **网络开销** | 大（冗余字段） | 小（选择性复制） |
| **维护成本** | 高（牵一发动全身） | 低（模块化） |
| **学习曲线** | 低 | 中 |

### 1.2 核心类设计

#### 1.2.1 ULyraInventoryItemDefinition：物品定义

物品定义是一个 **Data Asset**，包含物品的静态配置数据。

```cpp
// LyraInventoryItemDefinition.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "LyraInventoryItemDefinition.generated.h"

class ULyraInventoryItemInstance;
class ULyraInventoryItemFragment;

/**
 * 物品定义：定义一个物品的静态数据
 * 
 * 这是一个 Data Asset，游戏策划可以在编辑器中创建和配置
 * 每个物品类型（如"治疗药水"、"铁剑"）对应一个 Definition
 */
UCLASS(BlueprintType, Const, Meta = (DisplayName = "Lyra Inventory Item Definition"))
class LYRAGAME_API ULyraInventoryItemDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    ULyraInventoryItemDefinition(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    //~UObject interface
    virtual FPrimaryAssetId GetPrimaryAssetId() const override;
    //~End of UObject interface

public:
    /**
     * 物品的显示名称
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Display")
    FText DisplayName;

    /**
     * 物品描述
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Display", meta = (MultiLine = "true"))
    FText Description;

    /**
     * 物品图标（用于 UI 显示）
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Display")
    TObjectPtr<UTexture2D> ItemIcon;

    /**
     * 物品 3D 模型（用于掉落物、装备等）
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Display")
    TObjectPtr<UStaticMesh> ItemMesh;

    /**
     * 物品的运行时实例类型
     * 
     * 默认为 ULyraInventoryItemInstance
     * 可以继承并扩展自定义逻辑（如武器实例、消耗品实例）
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Instance")
    TSubclassOf<ULyraInventoryItemInstance> InstanceType;

    /**
     * 物品碎片列表
     * 
     * Fragments 定义了物品的各种特性（属性、技能、效果等）
     * 这是碎片化设计的核心
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Fragments", Instanced)
    TArray<TObjectPtr<ULyraInventoryItemFragment>> Fragments;

    /**
     * 默认堆叠数量上限
     * 
     * - 1：不可堆叠（如装备）
     * - >1：可堆叠（如消耗品、材料）
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Stacking")
    int32 MaxStackCount = 1;

    /**
     * 物品是否可以丢弃
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Gameplay")
    bool bCanDrop = true;

    /**
     * 物品是否可以出售
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Gameplay")
    bool bCanSell = true;

    /**
     * 物品出售价格（游戏币）
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Gameplay", meta = (EditCondition = "bCanSell"))
    int32 SellPrice = 0;

public:
    /**
     * 获取指定类型的 Fragment
     * 
     * @tparam FragmentType Fragment 类型
     * @return Fragment 实例，如果不存在则返回 nullptr
     */
    template <typename FragmentType>
    const FragmentType* FindFragmentByClass() const
    {
        return (FragmentType*)FindFragmentByClass(FragmentType::StaticClass());
    }

    /**
     * 获取指定类型的 Fragment（UClass 版本）
     */
    const ULyraInventoryItemFragment* FindFragmentByClass(TSubclassOf<ULyraInventoryItemFragment> FragmentClass) const;
};
```

**设计要点**：

1. **只读性**：Definition 是 `Const`，运行时不可修改（保证数据一致性）
2. **Data Asset**：继承自 `UPrimaryDataAsset`，支持资产管理器（Asset Manager）
3. **实例类型**：`InstanceType` 允许自定义物品实例类（如武器实例、装备实例）
4. **Fragments**：`Instanced` 关键字使得每个 Fragment 可以在编辑器中独立配置

#### 1.2.2 ULyraInventoryItemInstance：物品实例

物品实例是运行时对象，存储物品的动态状态。

```cpp
// LyraInventoryItemInstance.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "LyraInventoryItemInstance.generated.h"

class ULyraInventoryItemDefinition;
class ULyraInventoryItemFragment;
struct FGameplayTag;

/**
 * 物品实例：运行时的物品对象
 * 
 * 每个玩家背包中的物品都是一个 Instance
 * 例如：玩家有 3 把"铁剑"，则有 3 个 ItemInstance（如果不堆叠）
 */
UCLASS(BlueprintType)
class LYRAGAME_API ULyraInventoryItemInstance : public UObject
{
    GENERATED_BODY()

public:
    ULyraInventoryItemInstance(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    //~UObject interface
    virtual bool IsSupportedForNetworking() const override { return true; }
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    //~End of UObject interface

public:
    /**
     * 物品定义（指向 Data Asset）
     */
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Inventory")
    TObjectPtr<const ULyraInventoryItemDefinition> ItemDef;

    /**
     * 当前堆叠数量
     * 
     * - 1：单个物品
     * - >1：堆叠的物品（如 50 个箭矢）
     */
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Inventory")
    int32 StackCount = 1;

    /**
     * 物品的唯一 ID（用于网络同步和查找）
     * 
     * 由 Inventory Component 在添加物品时自动分配
     */
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Inventory")
    int32 ItemID = 0;

public:
    /**
     * 获取物品定义
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    const ULyraInventoryItemDefinition* GetItemDef() const { return ItemDef; }

    /**
     * 获取当前堆叠数量
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    int32 GetStackCount() const { return StackCount; }

    /**
     * 设置堆叠数量（内部使用，带验证）
     */
    void SetStackCount(int32 NewCount);

    /**
     * 获取最大堆叠数量
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    int32 GetMaxStackCount() const;

    /**
     * 是否可以堆叠
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    bool CanStack() const;

    /**
     * 是否可以与另一个物品堆叠
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    bool CanStackWith(const ULyraInventoryItemInstance* OtherItem) const;

    /**
     * 尝试添加堆叠（返回剩余无法添加的数量）
     */
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    int32 TryAddStack(int32 Amount);

    /**
     * 尝试减少堆叠（返回实际减少的数量）
     */
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    int32 TryRemoveStack(int32 Amount);

public:
    /**
     * 获取指定类型的 Fragment
     */
    template <typename FragmentType>
    const FragmentType* FindFragmentByClass() const
    {
        if (ItemDef != nullptr)
        {
            return ItemDef->FindFragmentByClass<FragmentType>();
        }
        return nullptr;
    }

    /**
     * 获取 Fragment（蓝图版本）
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory", meta = (DeterminesOutputType = "FragmentClass"))
    const ULyraInventoryItemFragment* FindFragmentByClass(TSubclassOf<ULyraInventoryItemFragment> FragmentClass) const;

    /**
     * 物品初始化回调（当物品被添加到背包时调用）
     */
    virtual void OnItemAdded(AActor* OwningActor);

    /**
     * 物品移除回调（当物品从背包移除时调用）
     */
    virtual void OnItemRemoved();

    /**
     * 物品使用回调（基类为空，子类可重写）
     */
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Inventory")
    void OnItemUsed(AActor* User);

protected:
    /**
     * 所属的 Actor（通常是玩家）
     */
    UPROPERTY(BlueprintReadOnly, Category = "Inventory")
    TObjectPtr<AActor> OwningActor;

public:
    /** 获取所属 Actor */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    AActor* GetOwningActor() const { return OwningActor; }

    /** 设置所属 Actor（内部使用） */
    void SetOwningActor(AActor* NewOwner);

    /**
     * 获取物品的统计标签（用于 GAS 集成）
     * 例如："Item.Weapon.Sword"、"Item.Consumable.Potion"
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    virtual FGameplayTagContainer GetItemTags() const;
};
```

**设计要点**：

1. **网络复制**：`Replicated` 确保服务器和客户端数据一致
2. **轻量级**：只存储动态状态（堆叠数、所有者），静态数据由 Definition 提供
3. **生命周期回调**：`OnItemAdded`/`OnItemRemoved`/`OnItemUsed` 允许子类扩展逻辑
4. **Fragment 访问**：通过 `FindFragmentByClass` 访问物品的特性

#### 1.2.3 ULyraInventoryItemFragment：物品碎片基类

Fragment 是物品特性的封装单元。

```cpp
// LyraInventoryItemFragment.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "LyraInventoryItemFragment.generated.h"

class ULyraInventoryItemInstance;

/**
 * 物品碎片基类
 * 
 * Fragment 封装了物品的某一特性（如属性、技能、效果）
 * 所有 Fragment 都继承自这个基类
 * 
 * 设计理念：
 * - 每个 Fragment 负责一个独立的功能
 * - Fragment 可以组合，形成复杂的物品
 * - Fragment 可以在编辑器中配置
 */
UCLASS(DefaultToInstanced, EditInlineNew, Abstract)
class LYRAGAME_API ULyraInventoryItemFragment : public UObject
{
    GENERATED_BODY()

public:
    /**
     * 当物品实例被创建时调用
     * 
     * @param ItemInstance 新创建的物品实例
     */
    virtual void OnInstanceCreated(ULyraInventoryItemInstance* ItemInstance) const {}

    /**
     * 当物品被添加到背包时调用
     * 
     * @param ItemInstance 物品实例
     */
    virtual void OnItemAdded(ULyraInventoryItemInstance* ItemInstance) const {}

    /**
     * 当物品从背包移除时调用
     * 
     * @param ItemInstance 物品实例
     */
    virtual void OnItemRemoved(ULyraInventoryItemInstance* ItemInstance) const {}

    /**
     * 当物品被使用时调用
     * 
     * @param ItemInstance 物品实例
     * @param User 使用者（通常是玩家）
     */
    virtual void OnItemUsed(ULyraInventoryItemInstance* ItemInstance, AActor* User) const {}

    /**
     * 当物品被装备时调用（仅装备类物品）
     * 
     * @param ItemInstance 物品实例
     * @param Wearer 装备者
     */
    virtual void OnItemEquipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const {}

    /**
     * 当物品被卸下时调用（仅装备类物品）
     * 
     * @param ItemInstance 物品实例
     * @param Wearer 装备者
     */
    virtual void OnItemUnequipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const {}
};
```

**设计要点**：

1. **生命周期钩子**：提供多个回调点，Fragment 可以在物品生命周期的关键时刻执行逻辑
2. **抽象基类**：`Abstract` 确保不能直接使用，必须继承
3. **Instanced**：`DefaultToInstanced` 允许在编辑器中内联编辑

### 1.3 Inventory Component 核心组件

`ULyraInventoryManagerComponent` 是背包系统的管理中枢，负责物品的增删查改、槽位管理和网络同步。

#### 1.3.1 核心类定义

```cpp
// LyraInventoryManagerComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "Net/Serialization/FastArraySerializer.h"
#include "LyraInventoryManagerComponent.generated.h"

class ULyraInventoryItemInstance;
class ULyraInventoryItemDefinition;
struct FGameplayTag;

/**
 * 背包物品条目（Fast Array 元素）
 * 
 * Fast Array 是 UE 的高效网络复制容器
 * 适合需要频繁增删改的数组
 */
USTRUCT(BlueprintType)
struct FLyraInventoryEntry : public FFastArraySerializerItem
{
    GENERATED_BODY()

    FLyraInventoryEntry()
        : ItemInstance(nullptr)
        , SlotIndex(-1)
    {}

    /**
     * 物品实例
     */
    UPROPERTY(BlueprintReadOnly, Category = "Inventory")
    TObjectPtr<ULyraInventoryItemInstance> ItemInstance;

    /**
     * 槽位索引（-1 表示不在特定槽位）
     */
    UPROPERTY(BlueprintReadOnly, Category = "Inventory")
    int32 SlotIndex;

    /**
     * Fast Array 回调：条目被添加
     */
    void PreReplicatedRemove(const struct FLyraInventoryList& InArraySerializer);
    void PostReplicatedAdd(const struct FLyraInventoryList& InArraySerializer);
    void PostReplicatedChange(const struct FLyraInventoryList& InArraySerializer);

    /** 生成唯一 ID */
    FString GetDebugString() const;
};

/**
 * 背包物品列表（Fast Array 容器）
 */
USTRUCT(BlueprintType)
struct FLyraInventoryList : public FFastArraySerializer
{
    GENERATED_BODY()

    FLyraInventoryList()
        : OwnerComponent(nullptr)
    {}

    /**
     * 物品条目数组
     */
    UPROPERTY()
    TArray<FLyraInventoryEntry> Entries;

    /**
     * 所属的 Inventory Component
     */
    UPROPERTY(NotReplicated)
    TObjectPtr<ULyraInventoryManagerComponent> OwnerComponent;

public:
    //~FFastArraySerializer contract
    void PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize);
    void PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize);
    void PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize);
    //~End of FFastArraySerializer contract

    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
    {
        return FFastArraySerializer::FastArrayDeltaSerialize<FLyraInventoryEntry, FLyraInventoryList>(Entries, DeltaParms, *this);
    }

    /** 添加物品条目 */
    FLyraInventoryEntry& AddEntry(ULyraInventoryItemInstance* ItemInstance, int32 SlotIndex = -1);

    /** 移除物品条目 */
    void RemoveEntry(ULyraInventoryItemInstance* ItemInstance);

    /** 查找物品条目 */
    FLyraInventoryEntry* FindEntry(ULyraInventoryItemInstance* ItemInstance);
    const FLyraInventoryEntry* FindEntry(ULyraInventoryItemInstance* ItemInstance) const;

    /** 查找槽位的物品 */
    ULyraInventoryItemInstance* FindItemInSlot(int32 SlotIndex) const;

    /** 获取所有物品 */
    TArray<ULyraInventoryItemInstance*> GetAllItems() const;
};

template<>
struct TStructOpsTypeTraits<FLyraInventoryList> : public TStructOpsTypeTraitsBase2<FLyraInventoryList>
{
    enum
    {
        WithNetDeltaSerializer = true,
    };
};

/**
 * 背包管理器组件
 * 
 * 负责管理角色的所有物品
 * - 添加/移除物品
 * - 槽位管理
 * - 堆叠与拆分
 * - 网络同步
 */
UCLASS(BlueprintType, meta = (BlueprintSpawnableComponent))
class LYRAGAME_API ULyraInventoryManagerComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    ULyraInventoryManagerComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    //~UActorComponent interface
    virtual void InitializeComponent() override;
    virtual void UninitializeComponent() override;
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual bool ReplicateSubobjects(class UActorChannel* Channel, class FOutBunch* Bunch, FReplicationFlags* RepFlags) override;
    virtual void ReadyForReplication() override;
    //~End of UActorComponent interface

public:
    /**
     * 背包容量上限（-1 表示无限）
     */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Inventory")
    int32 MaxInventorySlots = 40;

    /**
     * 是否启用槽位系统（格子背包 vs 列表背包）
     * 
     * - true：格子背包（每个物品占据特定槽位，如《暗黑破坏神》）
     * - false：列表背包（物品自动排列，如《魔兽世界》）
     */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Inventory")
    bool bUseSlotSystem = true;

    /**
     * 是否允许物品自动堆叠
     */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Inventory")
    bool bAutoStack = true;

public:
    /**
     * 添加物品到背包
     * 
     * @param ItemDef 物品定义
     * @param Count 数量
     * @param OutRemainingCount [输出] 无法添加的剩余数量（背包满时）
     * @return 新创建的物品实例（可能为 nullptr）
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
    ULyraInventoryItemInstance* AddItemByDefinition(
        TSubclassOf<ULyraInventoryItemDefinition> ItemDef,
        int32 Count = 1,
        int32& OutRemainingCount);

    /**
     * 添加物品实例到背包
     * 
     * @param ItemInstance 物品实例
     * @param SlotIndex 目标槽位（-1 表示自动分配）
     * @return 是否成功添加
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
    bool AddItemInstance(ULyraInventoryItemInstance* ItemInstance, int32 SlotIndex = -1);

    /**
     * 移除物品
     * 
     * @param ItemInstance 物品实例
     * @param Count 移除数量（-1 表示全部移除）
     * @return 实际移除的数量
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
    int32 RemoveItemInstance(ULyraInventoryItemInstance* ItemInstance, int32 Count = -1);

    /**
     * 移除指定槽位的物品
     * 
     * @param SlotIndex 槽位索引
     * @param Count 移除数量（-1 表示全部移除）
     * @return 实际移除的数量
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
    int32 RemoveItemFromSlot(int32 SlotIndex, int32 Count = -1);

    /**
     * 查找物品（按定义）
     * 
     * @param ItemDef 物品定义
     * @return 找到的第一个物品实例（可能为 nullptr）
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    ULyraInventoryItemInstance* FindItemByDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef) const;

    /**
     * 查找所有匹配的物品
     * 
     * @param ItemDef 物品定义
     * @return 所有匹配的物品实例
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    TArray<ULyraInventoryItemInstance*> FindAllItemsByDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef) const;

    /**
     * 获取指定槽位的物品
     * 
     * @param SlotIndex 槽位索引
     * @return 物品实例（可能为 nullptr）
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    ULyraInventoryItemInstance* GetItemInSlot(int32 SlotIndex) const;

    /**
     * 获取所有物品
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    TArray<ULyraInventoryItemInstance*> GetAllItems() const;

    /**
     * 获取物品总数（按定义）
     * 
     * @param ItemDef 物品定义
     * @return 总数（所有堆叠之和）
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    int32 GetItemCount(TSubclassOf<ULyraInventoryItemDefinition> ItemDef) const;

    /**
     * 移动物品到指定槽位
     * 
     * @param ItemInstance 物品实例
     * @param TargetSlotIndex 目标槽位
     * @return 是否成功移动
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
    bool MoveItemToSlot(ULyraInventoryItemInstance* ItemInstance, int32 TargetSlotIndex);

    /**
     * 交换两个槽位的物品
     * 
     * @param SlotIndexA 槽位 A
     * @param SlotIndexB 槽位 B
     * @return 是否成功交换
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
    bool SwapItemSlots(int32 SlotIndexA, int32 SlotIndexB);

    /**
     * 拆分物品堆叠
     * 
     * @param ItemInstance 源物品
     * @param SplitCount 拆分数量
     * @param TargetSlotIndex 目标槽位（-1 表示自动分配）
     * @return 新创建的物品实例（可能为 nullptr）
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
    ULyraInventoryItemInstance* SplitItemStack(
        ULyraInventoryItemInstance* ItemInstance,
        int32 SplitCount,
        int32 TargetSlotIndex = -1);

    /**
     * 合并物品堆叠
     * 
     * @param SourceItem 源物品
     * @param TargetItem 目标物品
     * @return 是否成功合并
     */
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
    bool MergeItemStacks(ULyraInventoryItemInstance* SourceItem, ULyraInventoryItemInstance* TargetItem);

    /**
     * 查找空闲槽位
     * 
     * @return 空闲槽位索引（-1 表示没有空闲槽位）
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    int32 FindEmptySlot() const;

    /**
     * 检查背包是否已满
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    bool IsInventoryFull() const;

    /**
     * 获取当前物品数量
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    int32 GetCurrentItemCount() const { return InventoryList.Entries.Num(); }

    /**
     * 获取最大容量
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    int32 GetMaxCapacity() const { return MaxInventorySlots; }

protected:
    /**
     * 背包物品列表（网络复制）
     */
    UPROPERTY(Replicated)
    FLyraInventoryList InventoryList;

    /**
     * 下一个物品 ID（用于生成唯一 ID）
     */
    int32 NextItemID = 1;

    /**
     * 创建物品实例
     */
    ULyraInventoryItemInstance* CreateItemInstance(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 Count);

    /**
     * 尝试自动堆叠物品
     * 
     * @param ItemDef 物品定义
     * @param Count [输入/输出] 剩余数量
     * @return 是否完全堆叠（Count == 0）
     */
    bool TryAutoStack(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32& Count);

    /**
     * 验证槽位索引
     */
    bool IsValidSlotIndex(int32 SlotIndex) const;

    /**
     * 验证槽位是否为空
     */
    bool IsSlotEmpty(int32 SlotIndex) const;

public:
    // ==================== 事件委托 ====================

    /**
     * 物品添加事件
     */
    DECLARE_MULTICAST_DELEGATE_TwoParams(FOnItemAdded, ULyraInventoryItemInstance* /*ItemInstance*/, int32 /*SlotIndex*/);
    FOnItemAdded OnItemAdded;

    /**
     * 物品移除事件
     */
    DECLARE_MULTICAST_DELEGATE_TwoParams(FOnItemRemoved, ULyraInventoryItemInstance* /*ItemInstance*/, int32 /*Count*/);
    FOnItemRemoved OnItemRemoved;

    /**
     * 物品移动事件
     */
    DECLARE_MULTICAST_DELEGATE_ThreeParams(FOnItemMoved, ULyraInventoryItemInstance* /*ItemInstance*/, int32 /*FromSlot*/, int32 /*ToSlot*/);
    FOnItemMoved OnItemMoved;

    /**
     * 物品堆叠改变事件
     */
    DECLARE_MULTICAST_DELEGATE_TwoParams(FOnItemStackChanged, ULyraInventoryItemInstance* /*ItemInstance*/, int32 /*NewCount*/);
    FOnItemStackChanged OnItemStackChanged;
};
```

**设计要点**：

1. **Fast Array**：`FLyraInventoryList` 使用 Fast Array 实现高效的网络复制
2. **槽位系统**：支持格子背包和列表背包两种模式
3. **自动堆叠**：可配置的自动堆叠功能
4. **事件驱动**：通过委托（Delegate）通知 UI 和其他系统
5. **权限控制**：所有修改操作都标记 `BlueprintAuthorityOnly`（仅服务器）

#### 1.3.2 核心实现

```cpp
// LyraInventoryManagerComponent.cpp
#include "LyraInventoryManagerComponent.h"
#include "LyraInventoryItemDefinition.h"
#include "LyraInventoryItemInstance.h"
#include "Net/UnrealNetwork.h"
#include "Engine/ActorChannel.h"

// ==================== FLyraInventoryEntry ====================

void FLyraInventoryEntry::PreReplicatedRemove(const FLyraInventoryList& InArraySerializer)
{
    // 物品被移除前的清理
    if (ItemInstance != nullptr)
    {
        ItemInstance->OnItemRemoved();
    }
}

void FLyraInventoryEntry::PostReplicatedAdd(const FLyraInventoryList& InArraySerializer)
{
    // 物品被添加后的初始化
    if (ItemInstance != nullptr && InArraySerializer.OwnerComponent != nullptr)
    {
        AActor* OwningActor = InArraySerializer.OwnerComponent->GetOwner();
        ItemInstance->SetOwningActor(OwningActor);
        ItemInstance->OnItemAdded(OwningActor);

        // 触发事件
        InArraySerializer.OwnerComponent->OnItemAdded.Broadcast(ItemInstance, SlotIndex);
    }
}

void FLyraInventoryEntry::PostReplicatedChange(const FLyraInventoryList& InArraySerializer)
{
    // 物品数据改变后的更新
    if (ItemInstance != nullptr && InArraySerializer.OwnerComponent != nullptr)
    {
        // 触发堆叠改变事件
        InArraySerializer.OwnerComponent->OnItemStackChanged.Broadcast(ItemInstance, ItemInstance->GetStackCount());
    }
}

FString FLyraInventoryEntry::GetDebugString() const
{
    if (ItemInstance != nullptr && ItemInstance->GetItemDef() != nullptr)
    {
        return FString::Printf(TEXT("[Slot %d] %s x%d (ID: %d)"),
            SlotIndex,
            *ItemInstance->GetItemDef()->GetName(),
            ItemInstance->GetStackCount(),
            ItemInstance->ItemID);
    }
    return TEXT("Invalid Entry");
}

// ==================== FLyraInventoryList ====================

void FLyraInventoryList::PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize)
{
    for (int32 Index : RemovedIndices)
    {
        if (Entries.IsValidIndex(Index))
        {
            Entries[Index].PreReplicatedRemove(*this);
        }
    }
}

void FLyraInventoryList::PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize)
{
    for (int32 Index : AddedIndices)
    {
        if (Entries.IsValidIndex(Index))
        {
            Entries[Index].PostReplicatedAdd(*this);
        }
    }
}

void FLyraInventoryList::PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize)
{
    for (int32 Index : ChangedIndices)
    {
        if (Entries.IsValidIndex(Index))
        {
            Entries[Index].PostReplicatedChange(*this);
        }
    }
}

FLyraInventoryEntry& FLyraInventoryList::AddEntry(ULyraInventoryItemInstance* ItemInstance, int32 SlotIndex)
{
    FLyraInventoryEntry& NewEntry = Entries.AddDefaulted_GetRef();
    NewEntry.ItemInstance = ItemInstance;
    NewEntry.SlotIndex = SlotIndex;
    MarkItemDirty(NewEntry);
    return NewEntry;
}

void FLyraInventoryList::RemoveEntry(ULyraInventoryItemInstance* ItemInstance)
{
    for (int32 i = Entries.Num() - 1; i >= 0; --i)
    {
        if (Entries[i].ItemInstance == ItemInstance)
        {
            Entries.RemoveAt(i);
            MarkArrayDirty();
            break;
        }
    }
}

FLyraInventoryEntry* FLyraInventoryList::FindEntry(ULyraInventoryItemInstance* ItemInstance)
{
    for (FLyraInventoryEntry& Entry : Entries)
    {
        if (Entry.ItemInstance == ItemInstance)
        {
            return &Entry;
        }
    }
    return nullptr;
}

const FLyraInventoryEntry* FLyraInventoryList::FindEntry(ULyraInventoryItemInstance* ItemInstance) const
{
    for (const FLyraInventoryEntry& Entry : Entries)
    {
        if (Entry.ItemInstance == ItemInstance)
        {
            return &Entry;
        }
    }
    return nullptr;
}

ULyraInventoryItemInstance* FLyraInventoryList::FindItemInSlot(int32 SlotIndex) const
{
    for (const FLyraInventoryEntry& Entry : Entries)
    {
        if (Entry.SlotIndex == SlotIndex)
        {
            return Entry.ItemInstance;
        }
    }
    return nullptr;
}

TArray<ULyraInventoryItemInstance*> FLyraInventoryList::GetAllItems() const
{
    TArray<ULyraInventoryItemInstance*> Items;
    Items.Reserve(Entries.Num());
    for (const FLyraInventoryEntry& Entry : Entries)
    {
        if (Entry.ItemInstance != nullptr)
        {
            Items.Add(Entry.ItemInstance);
        }
    }
    return Items;
}

// ==================== ULyraInventoryManagerComponent ====================

ULyraInventoryManagerComponent::ULyraInventoryManagerComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    SetIsReplicatedByDefault(true);
    bWantsInitializeComponent = true;
}

void ULyraInventoryManagerComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ULyraInventoryManagerComponent, InventoryList);
}

bool ULyraInventoryManagerComponent::ReplicateSubobjects(UActorChannel* Channel, FOutBunch* Bunch, FReplicationFlags* RepFlags)
{
    bool bWroteSomething = Super::ReplicateSubobjects(Channel, Bunch, RepFlags);

    for (const FLyraInventoryEntry& Entry : InventoryList.Entries)
    {
        if (Entry.ItemInstance != nullptr)
        {
            bWroteSomething |= Channel->ReplicateSubobject(Entry.ItemInstance, *Bunch, *RepFlags);
        }
    }

    return bWroteSomething;
}

void ULyraInventoryManagerComponent::ReadyForReplication()
{
    Super::ReadyForReplication();

    // 注册子对象复制
    if (IsUsingRegisteredSubObjectList())
    {
        for (const FLyraInventoryEntry& Entry : InventoryList.Entries)
        {
            if (Entry.ItemInstance != nullptr)
            {
                AddReplicatedSubObject(Entry.ItemInstance);
            }
        }
    }
}

void ULyraInventoryManagerComponent::InitializeComponent()
{
    Super::InitializeComponent();

    InventoryList.OwnerComponent = this;
}

void ULyraInventoryManagerComponent::UninitializeComponent()
{
    // 清理所有物品
    for (FLyraInventoryEntry& Entry : InventoryList.Entries)
    {
        if (Entry.ItemInstance != nullptr)
        {
            Entry.ItemInstance->OnItemRemoved();
        }
    }
    InventoryList.Entries.Empty();

    Super::UninitializeComponent();
}

// ==================== 添加物品 ====================

ULyraInventoryItemInstance* ULyraInventoryManagerComponent::AddItemByDefinition(
    TSubclassOf<ULyraInventoryItemDefinition> ItemDef,
    int32 Count,
    int32& OutRemainingCount)
{
    OutRemainingCount = Count;

    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("AddItemByDefinition: Not called on server!"));
        return nullptr;
    }

    // 参数验证
    if (ItemDef == nullptr || Count <= 0)
    {
        UE_LOG(LogTemp, Warning, TEXT("AddItemByDefinition: Invalid parameters!"));
        return nullptr;
    }

    const ULyraInventoryItemDefinition* ItemDefCDO = GetDefault<ULyraInventoryItemDefinition>(ItemDef);
    if (ItemDefCDO == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("AddItemByDefinition: Invalid ItemDef CDO!"));
        return nullptr;
    }

    // 如果启用自动堆叠，先尝试堆叠到现有物品
    if (bAutoStack && ItemDefCDO->MaxStackCount > 1)
    {
        if (TryAutoStack(ItemDef, OutRemainingCount))
        {
            // 完全堆叠成功
            return FindItemByDefinition(ItemDef);
        }

        // 部分堆叠成功，继续处理剩余数量
        if (OutRemainingCount <= 0)
        {
            return FindItemByDefinition(ItemDef);
        }
    }

    // 创建新的物品实例
    ULyraInventoryItemInstance* NewItem = nullptr;
    while (OutRemainingCount > 0)
    {
        // 检查背包是否已满
        if (bUseSlotSystem && IsInventoryFull())
        {
            UE_LOG(LogTemp, Warning, TEXT("AddItemByDefinition: Inventory is full!"));
            break;
        }

        // 计算这次添加的数量
        int32 AddCount = FMath::Min(OutRemainingCount, ItemDefCDO->MaxStackCount);

        // 创建物品实例
        ULyraInventoryItemInstance* ItemInstance = CreateItemInstance(ItemDef, AddCount);
        if (ItemInstance == nullptr)
        {
            UE_LOG(LogTemp, Error, TEXT("AddItemByDefinition: Failed to create item instance!"));
            break;
        }

        // 添加到背包
        int32 SlotIndex = bUseSlotSystem ? FindEmptySlot() : -1;
        if (AddItemInstance(ItemInstance, SlotIndex))
        {
            OutRemainingCount -= AddCount;
            if (NewItem == nullptr)
            {
                NewItem = ItemInstance; // 返回第一个创建的实例
            }
        }
        else
        {
            // 添加失败，销毁实例
            ItemInstance->MarkAsGarbage();
            break;
        }
    }

    return NewItem;
}

bool ULyraInventoryManagerComponent::AddItemInstance(ULyraInventoryItemInstance* ItemInstance, int32 SlotIndex)
{
    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("AddItemInstance: Not called on server!"));
        return false;
    }

    // 参数验证
    if (ItemInstance == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("AddItemInstance: Invalid ItemInstance!"));
        return false;
    }

    // 槽位验证
    if (bUseSlotSystem)
    {
        if (SlotIndex == -1)
        {
            SlotIndex = FindEmptySlot();
            if (SlotIndex == -1)
            {
                UE_LOG(LogTemp, Warning, TEXT("AddItemInstance: No empty slot available!"));
                return false;
            }
        }
        else if (!IsValidSlotIndex(SlotIndex) || !IsSlotEmpty(SlotIndex))
        {
            UE_LOG(LogTemp, Warning, TEXT("AddItemInstance: Slot %d is invalid or occupied!"), SlotIndex);
            return false;
        }
    }

    // 分配物品 ID
    ItemInstance->ItemID = NextItemID++;

    // 设置所有者
    ItemInstance->SetOwningActor(GetOwner());

    // 添加到列表
    InventoryList.AddEntry(ItemInstance, SlotIndex);

    // 注册复制
    if (IsUsingRegisteredSubObjectList())
    {
        AddReplicatedSubObject(ItemInstance);
    }

    // 触发回调
    ItemInstance->OnItemAdded(GetOwner());

    // 触发事件
    OnItemAdded.Broadcast(ItemInstance, SlotIndex);

    UE_LOG(LogTemp, Log, TEXT("AddItemInstance: Added %s to slot %d"), *ItemInstance->GetItemDef()->GetName(), SlotIndex);

    return true;
}

// ==================== 移除物品 ====================

int32 ULyraInventoryManagerComponent::RemoveItemInstance(ULyraInventoryItemInstance* ItemInstance, int32 Count)
{
    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("RemoveItemInstance: Not called on server!"));
        return 0;
    }

    // 参数验证
    if (ItemInstance == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("RemoveItemInstance: Invalid ItemInstance!"));
        return 0;
    }

    // 查找物品
    FLyraInventoryEntry* Entry = InventoryList.FindEntry(ItemInstance);
    if (Entry == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("RemoveItemInstance: Item not found in inventory!"));
        return 0;
    }

    // 计算移除数量
    int32 CurrentCount = ItemInstance->GetStackCount();
    int32 RemoveCount = (Count == -1) ? CurrentCount : FMath::Min(Count, CurrentCount);

    // 减少堆叠
    ItemInstance->TryRemoveStack(RemoveCount);

    // 如果堆叠为 0，完全移除物品
    if (ItemInstance->GetStackCount() <= 0)
    {
        // 取消复制注册
        if (IsUsingRegisteredSubObjectList())
        {
            RemoveReplicatedSubObject(ItemInstance);
        }

        // 从列表移除
        InventoryList.RemoveEntry(ItemInstance);

        // 触发回调
        ItemInstance->OnItemRemoved();

        // 触发事件
        OnItemRemoved.Broadcast(ItemInstance, RemoveCount);

        UE_LOG(LogTemp, Log, TEXT("RemoveItemInstance: Completely removed %s"), *ItemInstance->GetItemDef()->GetName());
    }
    else
    {
        // 部分移除，更新堆叠
        InventoryList.MarkItemDirty(*Entry);

        // 触发事件
        OnItemStackChanged.Broadcast(ItemInstance, ItemInstance->GetStackCount());

        UE_LOG(LogTemp, Log, TEXT("RemoveItemInstance: Reduced %s stack to %d"),
            *ItemInstance->GetItemDef()->GetName(), ItemInstance->GetStackCount());
    }

    return RemoveCount;
}

int32 ULyraInventoryManagerComponent::RemoveItemFromSlot(int32 SlotIndex, int32 Count)
{
    ULyraInventoryItemInstance* ItemInstance = GetItemInSlot(SlotIndex);
    if (ItemInstance == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("RemoveItemFromSlot: No item in slot %d"), SlotIndex);
        return 0;
    }

    return RemoveItemInstance(ItemInstance, Count);
}

// ==================== 查找物品 ====================

ULyraInventoryItemInstance* ULyraInventoryManagerComponent::FindItemByDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef) const
{
    if (ItemDef == nullptr)
    {
        return nullptr;
    }

    const ULyraInventoryItemDefinition* ItemDefCDO = GetDefault<ULyraInventoryItemDefinition>(ItemDef);

    for (const FLyraInventoryEntry& Entry : InventoryList.Entries)
    {
        if (Entry.ItemInstance != nullptr && Entry.ItemInstance->GetItemDef() == ItemDefCDO)
        {
            return Entry.ItemInstance;
        }
    }

    return nullptr;
}

TArray<ULyraInventoryItemInstance*> ULyraInventoryManagerComponent::FindAllItemsByDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef) const
{
    TArray<ULyraInventoryItemInstance*> FoundItems;

    if (ItemDef == nullptr)
    {
        return FoundItems;
    }

    const ULyraInventoryItemDefinition* ItemDefCDO = GetDefault<ULyraInventoryItemDefinition>(ItemDef);

    for (const FLyraInventoryEntry& Entry : InventoryList.Entries)
    {
        if (Entry.ItemInstance != nullptr && Entry.ItemInstance->GetItemDef() == ItemDefCDO)
        {
            FoundItems.Add(Entry.ItemInstance);
        }
    }

    return FoundItems;
}

ULyraInventoryItemInstance* ULyraInventoryManagerComponent::GetItemInSlot(int32 SlotIndex) const
{
    return InventoryList.FindItemInSlot(SlotIndex);
}

TArray<ULyraInventoryItemInstance*> ULyraInventoryManagerComponent::GetAllItems() const
{
    return InventoryList.GetAllItems();
}

int32 ULyraInventoryManagerComponent::GetItemCount(TSubclassOf<ULyraInventoryItemDefinition> ItemDef) const
{
    int32 TotalCount = 0;

    TArray<ULyraInventoryItemInstance*> Items = FindAllItemsByDefinition(ItemDef);
    for (const ULyraInventoryItemInstance* Item : Items)
    {
        if (Item != nullptr)
        {
            TotalCount += Item->GetStackCount();
        }
    }

    return TotalCount;
}

// ==================== 槽位操作 ====================

bool ULyraInventoryManagerComponent::MoveItemToSlot(ULyraInventoryItemInstance* ItemInstance, int32 TargetSlotIndex)
{
    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("MoveItemToSlot: Not called on server!"));
        return false;
    }

    // 参数验证
    if (ItemInstance == nullptr || !IsValidSlotIndex(TargetSlotIndex))
    {
        return false;
    }

    // 查找物品
    FLyraInventoryEntry* Entry = InventoryList.FindEntry(ItemInstance);
    if (Entry == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("MoveItemToSlot: Item not found!"));
        return false;
    }

    // 检查目标槽位
    if (!IsSlotEmpty(TargetSlotIndex))
    {
        UE_LOG(LogTemp, Warning, TEXT("MoveItemToSlot: Target slot %d is occupied!"), TargetSlotIndex);
        return false;
    }

    // 移动
    int32 OldSlot = Entry->SlotIndex;
    Entry->SlotIndex = TargetSlotIndex;
    InventoryList.MarkItemDirty(*Entry);

    // 触发事件
    OnItemMoved.Broadcast(ItemInstance, OldSlot, TargetSlotIndex);

    UE_LOG(LogTemp, Log, TEXT("MoveItemToSlot: Moved %s from slot %d to %d"),
        *ItemInstance->GetItemDef()->GetName(), OldSlot, TargetSlotIndex);

    return true;
}

bool ULyraInventoryManagerComponent::SwapItemSlots(int32 SlotIndexA, int32 SlotIndexB)
{
    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("SwapItemSlots: Not called on server!"));
        return false;
    }

    // 参数验证
    if (!IsValidSlotIndex(SlotIndexA) || !IsValidSlotIndex(SlotIndexB))
    {
        return false;
    }

    // 获取物品
    ULyraInventoryItemInstance* ItemA = GetItemInSlot(SlotIndexA);
    ULyraInventoryItemInstance* ItemB = GetItemInSlot(SlotIndexB);

    // 至少有一个槽位有物品
    if (ItemA == nullptr && ItemB == nullptr)
    {
        return true; // 两个槽位都是空的，无需交换
    }

    // 交换槽位
    if (ItemA != nullptr)
    {
        FLyraInventoryEntry* EntryA = InventoryList.FindEntry(ItemA);
        if (EntryA != nullptr)
        {
            EntryA->SlotIndex = SlotIndexB;
            InventoryList.MarkItemDirty(*EntryA);
            OnItemMoved.Broadcast(ItemA, SlotIndexA, SlotIndexB);
        }
    }

    if (ItemB != nullptr)
    {
        FLyraInventoryEntry* EntryB = InventoryList.FindEntry(ItemB);
        if (EntryB != nullptr)
        {
            EntryB->SlotIndex = SlotIndexA;
            InventoryList.MarkItemDirty(*EntryB);
            OnItemMoved.Broadcast(ItemB, SlotIndexB, SlotIndexA);
        }
    }

    UE_LOG(LogTemp, Log, TEXT("SwapItemSlots: Swapped slots %d and %d"), SlotIndexA, SlotIndexB);

    return true;
}

// ==================== 堆叠操作 ====================

ULyraInventoryItemInstance* ULyraInventoryManagerComponent::SplitItemStack(
    ULyraInventoryItemInstance* ItemInstance,
    int32 SplitCount,
    int32 TargetSlotIndex)
{
    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("SplitItemStack: Not called on server!"));
        return nullptr;
    }

    // 参数验证
    if (ItemInstance == nullptr || SplitCount <= 0)
    {
        return nullptr;
    }

    int32 CurrentStack = ItemInstance->GetStackCount();
    if (SplitCount >= CurrentStack)
    {
        UE_LOG(LogTemp, Warning, TEXT("SplitItemStack: Split count %d >= current stack %d!"), SplitCount, CurrentStack);
        return nullptr;
    }

    // 验证目标槽位
    if (bUseSlotSystem)
    {
        if (TargetSlotIndex == -1)
        {
            TargetSlotIndex = FindEmptySlot();
            if (TargetSlotIndex == -1)
            {
                UE_LOG(LogTemp, Warning, TEXT("SplitItemStack: No empty slot available!"));
                return nullptr;
            }
        }
        else if (!IsValidSlotIndex(TargetSlotIndex) || !IsSlotEmpty(TargetSlotIndex))
        {
            UE_LOG(LogTemp, Warning, TEXT("SplitItemStack: Target slot %d is invalid or occupied!"), TargetSlotIndex);
            return nullptr;
        }
    }

    // 减少源物品堆叠
    ItemInstance->TryRemoveStack(SplitCount);

    // 更新源物品
    FLyraInventoryEntry* SourceEntry = InventoryList.FindEntry(ItemInstance);
    if (SourceEntry != nullptr)
    {
        InventoryList.MarkItemDirty(*SourceEntry);
        OnItemStackChanged.Broadcast(ItemInstance, ItemInstance->GetStackCount());
    }

    // 创建新物品
    ULyraInventoryItemInstance* NewItem = CreateItemInstance(ItemInstance->GetItemDef()->GetClass(), SplitCount);
    if (NewItem == nullptr)
    {
        // 回滚
        ItemInstance->TryAddStack(SplitCount);
        UE_LOG(LogTemp, Error, TEXT("SplitItemStack: Failed to create new item!"));
        return nullptr;
    }

    // 添加新物品
    if (!AddItemInstance(NewItem, TargetSlotIndex))
    {
        // 回滚
        ItemInstance->TryAddStack(SplitCount);
        NewItem->MarkAsGarbage();
        UE_LOG(LogTemp, Error, TEXT("SplitItemStack: Failed to add new item to slot!"));
        return nullptr;
    }

    UE_LOG(LogTemp, Log, TEXT("SplitItemStack: Split %s, %d -> %d + %d"),
        *ItemInstance->GetItemDef()->GetName(), CurrentStack, ItemInstance->GetStackCount(), NewItem->GetStackCount());

    return NewItem;
}

bool ULyraInventoryManagerComponent::MergeItemStacks(ULyraInventoryItemInstance* SourceItem, ULyraInventoryItemInstance* TargetItem)
{
    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("MergeItemStacks: Not called on server!"));
        return false;
    }

    // 参数验证
    if (SourceItem == nullptr || TargetItem == nullptr || SourceItem == TargetItem)
    {
        return false;
    }

    // 检查是否可以堆叠
    if (!TargetItem->CanStackWith(SourceItem))
    {
        UE_LOG(LogTemp, Warning, TEXT("MergeItemStacks: Items cannot stack!"));
        return false;
    }

    // 尝试堆叠
    int32 SourceCount = SourceItem->GetStackCount();
    int32 Remaining = TargetItem->TryAddStack(SourceCount);

    // 更新目标物品
    FLyraInventoryEntry* TargetEntry = InventoryList.FindEntry(TargetItem);
    if (TargetEntry != nullptr)
    {
        InventoryList.MarkItemDirty(*TargetEntry);
        OnItemStackChanged.Broadcast(TargetItem, TargetItem->GetStackCount());
    }

    // 如果源物品完全合并，移除它
    if (Remaining == 0)
    {
        RemoveItemInstance(SourceItem, -1);
        UE_LOG(LogTemp, Log, TEXT("MergeItemStacks: Fully merged %s"), *SourceItem->GetItemDef()->GetName());
    }
    else
    {
        // 部分合并，更新源物品
        SourceItem->SetStackCount(Remaining);
        FLyraInventoryEntry* SourceEntry = InventoryList.FindEntry(SourceItem);
        if (SourceEntry != nullptr)
        {
            InventoryList.MarkItemDirty(*SourceEntry);
            OnItemStackChanged.Broadcast(SourceItem, Remaining);
        }
        UE_LOG(LogTemp, Log, TEXT("MergeItemStacks: Partially merged %s, %d remaining"),
            *SourceItem->GetItemDef()->GetName(), Remaining);
    }

    return true;
}

// ==================== 辅助函数 ====================

ULyraInventoryItemInstance* ULyraInventoryManagerComponent::CreateItemInstance(
    TSubclassOf<ULyraInventoryItemDefinition> ItemDef,
    int32 Count)
{
    const ULyraInventoryItemDefinition* ItemDefCDO = GetDefault<ULyraInventoryItemDefinition>(ItemDef);
    if (ItemDefCDO == nullptr)
    {
        return nullptr;
    }

    // 创建实例
    TSubclassOf<ULyraInventoryItemInstance> InstanceType = ItemDefCDO->InstanceType;
    if (InstanceType == nullptr)
    {
        InstanceType = ULyraInventoryItemInstance::StaticClass();
    }

    ULyraInventoryItemInstance* NewInstance = NewObject<ULyraInventoryItemInstance>(this, InstanceType);
    if (NewInstance != nullptr)
    {
        NewInstance->ItemDef = ItemDefCDO;
        NewInstance->SetStackCount(Count);

        // 触发 Fragment 回调
        for (const ULyraInventoryItemFragment* Fragment : ItemDefCDO->Fragments)
        {
            if (Fragment != nullptr)
            {
                Fragment->OnInstanceCreated(NewInstance);
            }
        }
    }

    return NewInstance;
}

bool ULyraInventoryManagerComponent::TryAutoStack(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32& Count)
{
    if (Count <= 0)
    {
        return true;
    }

    // 查找所有相同类型的物品
    TArray<ULyraInventoryItemInstance*> ExistingItems = FindAllItemsByDefinition(ItemDef);

    for (ULyraInventoryItemInstance* Item : ExistingItems)
    {
        if (Item == nullptr)
        {
            continue;
        }

        // 尝试堆叠
        int32 Remaining = Item->TryAddStack(Count);

        // 更新 UI
        FLyraInventoryEntry* Entry = InventoryList.FindEntry(Item);
        if (Entry != nullptr)
        {
            InventoryList.MarkItemDirty(*Entry);
            OnItemStackChanged.Broadcast(Item, Item->GetStackCount());
        }

        Count = Remaining;

        if (Count <= 0)
        {
            return true; // 完全堆叠成功
        }
    }

    return Count <= 0;
}

bool ULyraInventoryManagerComponent::IsValidSlotIndex(int32 SlotIndex) const
{
    if (!bUseSlotSystem)
    {
        return true; // 列表模式下，所有槽位都有效
    }

    return SlotIndex >= 0 && SlotIndex < MaxInventorySlots;
}

bool ULyraInventoryManagerComponent::IsSlotEmpty(int32 SlotIndex) const
{
    return GetItemInSlot(SlotIndex) == nullptr;
}

int32 ULyraInventoryManagerComponent::FindEmptySlot() const
{
    if (!bUseSlotSystem)
    {
        return -1; // 列表模式不使用槽位
    }

    for (int32 i = 0; i < MaxInventorySlots; ++i)
    {
        if (IsSlotEmpty(i))
        {
            return i;
        }
    }

    return -1; // 没有空槽位
}

bool ULyraInventoryManagerComponent::IsInventoryFull() const
{
    if (!bUseSlotSystem)
    {
        return false; // 列表模式下，背包永不满（除非有其他限制）
    }

    return FindEmptySlot() == -1;
}
```

**实现要点**：

1. **Fast Array 回调**：`PreReplicatedRemove`/`PostReplicatedAdd`/`PostReplicatedChange` 处理网络同步后的逻辑
2. **权限控制**：所有修改操作都检查 `HasAuthority()`，确保只在服务器执行
3. **堆叠逻辑**：`TryAutoStack` 自动合并堆叠，减少槽位占用
4. **事件广播**：通过委托通知 UI 和其他系统
5. **内存管理**：使用 `MarkAsGarbage()` 清理未使用的物品实例

### 1.4 Item Instance 详解

#### 1.4.1 堆叠逻辑实现

```cpp
// LyraInventoryItemInstance.cpp
#include "LyraInventoryItemInstance.h"
#include "LyraInventoryItemDefinition.h"
#include "LyraInventoryItemFragment.h"
#include "Net/UnrealNetwork.h"

ULyraInventoryItemInstance::ULyraInventoryItemInstance(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
}

void ULyraInventoryItemInstance::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ULyraInventoryItemInstance, ItemDef);
    DOREPLIFETIME(ULyraInventoryItemInstance, StackCount);
    DOREPLIFETIME(ULyraInventoryItemInstance, ItemID);
}

void ULyraInventoryItemInstance::SetStackCount(int32 NewCount)
{
    int32 MaxStack = GetMaxStackCount();
    StackCount = FMath::Clamp(NewCount, 0, MaxStack);

    UE_LOG(LogTemp, Verbose, TEXT("SetStackCount: %s -> %d (Max: %d)"),
        ItemDef ? *ItemDef->GetName() : TEXT("Unknown"), StackCount, MaxStack);
}

int32 ULyraInventoryItemInstance::GetMaxStackCount() const
{
    return ItemDef ? ItemDef->MaxStackCount : 1;
}

bool ULyraInventoryItemInstance::CanStack() const
{
    return GetMaxStackCount() > 1;
}

bool ULyraInventoryItemInstance::CanStackWith(const ULyraInventoryItemInstance* OtherItem) const
{
    if (OtherItem == nullptr || OtherItem == this)
    {
        return false;
    }

    // 必须是同一种物品
    if (ItemDef != OtherItem->ItemDef)
    {
        return false;
    }

    // 必须可堆叠
    if (!CanStack())
    {
        return false;
    }

    // 目标物品未满
    if (OtherItem->GetStackCount() >= OtherItem->GetMaxStackCount())
    {
        return false;
    }

    return true;
}

int32 ULyraInventoryItemInstance::TryAddStack(int32 Amount)
{
    if (Amount <= 0)
    {
        return Amount;
    }

    int32 MaxStack = GetMaxStackCount();
    int32 AvailableSpace = MaxStack - StackCount;

    if (AvailableSpace <= 0)
    {
        return Amount; // 无法添加
    }

    int32 AddAmount = FMath::Min(Amount, AvailableSpace);
    StackCount += AddAmount;

    int32 Remaining = Amount - AddAmount;

    UE_LOG(LogTemp, Verbose, TEXT("TryAddStack: %s, Added %d, Remaining %d, New Stack: %d"),
        ItemDef ? *ItemDef->GetName() : TEXT("Unknown"), AddAmount, Remaining, StackCount);

    return Remaining;
}

int32 ULyraInventoryItemInstance::TryRemoveStack(int32 Amount)
{
    if (Amount <= 0)
    {
        return 0;
    }

    int32 RemoveAmount = FMath::Min(Amount, StackCount);
    StackCount -= RemoveAmount;

    UE_LOG(LogTemp, Verbose, TEXT("TryRemoveStack: %s, Removed %d, New Stack: %d"),
        ItemDef ? *ItemDef->GetName() : TEXT("Unknown"), RemoveAmount, StackCount);

    return RemoveAmount;
}

void ULyraInventoryItemInstance::OnItemAdded(AActor* OwningActor)
{
    this->OwningActor = OwningActor;

    // 触发 Fragment 回调
    if (ItemDef != nullptr)
    {
        for (const ULyraInventoryItemFragment* Fragment : ItemDef->Fragments)
        {
            if (Fragment != nullptr)
            {
                Fragment->OnItemAdded(this);
            }
        }
    }

    UE_LOG(LogTemp, Log, TEXT("OnItemAdded: %s added to %s"),
        ItemDef ? *ItemDef->GetName() : TEXT("Unknown"),
        OwningActor ? *OwningActor->GetName() : TEXT("Unknown"));
}

void ULyraInventoryItemInstance::OnItemRemoved()
{
    // 触发 Fragment 回调
    if (ItemDef != nullptr)
    {
        for (const ULyraInventoryItemFragment* Fragment : ItemDef->Fragments)
        {
            if (Fragment != nullptr)
            {
                Fragment->OnItemRemoved(this);
            }
        }
    }

    UE_LOG(LogTemp, Log, TEXT("OnItemRemoved: %s removed from %s"),
        ItemDef ? *ItemDef->GetName() : TEXT("Unknown"),
        OwningActor ? *OwningActor->GetName() : TEXT("Unknown"));

    OwningActor = nullptr;
}

void ULyraInventoryItemInstance::OnItemUsed_Implementation(AActor* User)
{
    // 触发 Fragment 回调
    if (ItemDef != nullptr)
    {
        for (const ULyraInventoryItemFragment* Fragment : ItemDef->Fragments)
        {
            if (Fragment != nullptr)
            {
                Fragment->OnItemUsed(this, User);
            }
        }
    }

    UE_LOG(LogTemp, Log, TEXT("OnItemUsed: %s used by %s"),
        ItemDef ? *ItemDef->GetName() : TEXT("Unknown"),
        User ? *User->GetName() : TEXT("Unknown"));
}

void ULyraInventoryItemInstance::SetOwningActor(AActor* NewOwner)
{
    OwningActor = NewOwner;
}

FGameplayTagContainer ULyraInventoryItemInstance::GetItemTags() const
{
    FGameplayTagContainer Tags;

    // 可以在子类或 Fragment 中扩展
    // 例如：Tags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Item.Weapon")));

    return Tags;
}

const ULyraInventoryItemFragment* ULyraInventoryItemInstance::FindFragmentByClass(TSubclassOf<ULyraInventoryItemFragment> FragmentClass) const
{
    if (ItemDef != nullptr)
    {
        return ItemDef->FindFragmentByClass(FragmentClass);
    }
    return nullptr;
}
```

#### 1.4.2 Definition 实现

```cpp
// LyraInventoryItemDefinition.cpp
#include "LyraInventoryItemDefinition.h"
#include "LyraInventoryItemFragment.h"

ULyraInventoryItemDefinition::ULyraInventoryItemDefinition(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    InstanceType = ULyraInventoryItemInstance::StaticClass();
}

FPrimaryAssetId ULyraInventoryItemDefinition::GetPrimaryAssetId() const
{
    // 返回资产 ID（用于资产管理器）
    return FPrimaryAssetId(TEXT("InventoryItem"), GetFName());
}

const ULyraInventoryItemFragment* ULyraInventoryItemDefinition::FindFragmentByClass(TSubclassOf<ULyraInventoryItemFragment> FragmentClass) const
{
    if (FragmentClass == nullptr)
    {
        return nullptr;
    }

    for (const ULyraInventoryItemFragment* Fragment : Fragments)
    {
        if (Fragment != nullptr && Fragment->IsA(FragmentClass))
        {
            return Fragment;
        }
    }

    return nullptr;
}
```

### 1.5 背包槽位管理（格子系统 vs 列表系统）

Lyra 的背包系统支持两种模式：

#### 1.5.1 格子背包（Grid-Based）

**特点**：
- 每个物品占据一个或多个固定槽位
- 类似《暗黑破坏神》、《泰拉瑞亚》
- 适合强调物品管理的游戏（生存类、RPG）

**实现要点**：
```cpp
// 启用槽位系统
InventoryComponent->bUseSlotSystem = true;
InventoryComponent->MaxInventorySlots = 40; // 例如 5x8 格子

// 添加物品到指定槽位
InventoryComponent->AddItemInstance(Item, 5); // 添加到第 5 格

// 移动物品
InventoryComponent->MoveItemToSlot(Item, 10); // 移动到第 10 格

// 交换物品
InventoryComponent->SwapItemSlots(5, 10); // 交换第 5 格和第 10 格
```

**优势**：
- 视觉化：玩家可以看到物品的位置
- 策略性：玩家需要规划物品摆放
- 沉浸感：更真实的背包体验

**劣势**：
- 管理复杂：玩家需要频繁整理背包
- UI 复杂：需要实现拖拽、交换等交互

#### 1.5.2 列表背包（List-Based）

**特点**：
- 物品自动排列，无需手动管理槽位
- 类似《魔兽世界》、《上古卷轴》
- 适合轻松的游戏体验

**实现要点**：
```cpp
// 禁用槽位系统
InventoryComponent->bUseSlotSystem = false;

// 添加物品（自动分配槽位）
InventoryComponent->AddItemByDefinition(ItemDef, 10, RemainingCount);

// 物品自动堆叠
InventoryComponent->bAutoStack = true;
```

**优势**：
- 简单易用：玩家无需管理槽位
- 自动优化：系统自动堆叠和排序
- UI 简单：只需一个列表视图

**劣势**：
- 缺少策略性：无法精细控制物品位置
- 视觉化差：物品位置不固定

#### 1.5.3 混合模式

实际游戏中，可以结合两种模式：

```cpp
/**
 * 混合背包系统
 * 
 * - 主背包：格子系统（40 格）
 * - 快捷栏：格子系统（10 格）
 * - 材料包：列表系统（自动堆叠）
 * - 任务物品：列表系统（不占用主背包）
 */
UCLASS()
class ALyraCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    /** 主背包（格子系统） */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Inventory")
    TObjectPtr<ULyraInventoryManagerComponent> MainInventory;

    /** 快捷栏（格子系统） */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Inventory")
    TObjectPtr<ULyraInventoryManagerComponent> QuickBar;

    /** 材料包（列表系统） */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Inventory")
    TObjectPtr<ULyraInventoryManagerComponent> MaterialBag;

    /** 任务物品（列表系统） */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Inventory")
    TObjectPtr<ULyraInventoryManagerComponent> QuestItems;
};
```

### 1.6 物品堆叠与拆分

堆叠系统是背包管理的核心功能之一。

#### 1.6.1 自动堆叠逻辑

```cpp
/**
 * 自动堆叠算法
 * 
 * 当添加物品时：
 * 1. 查找所有相同类型的物品
 * 2. 按槽位顺序尝试堆叠
 * 3. 如果所有现有物品都已满，创建新堆叠
 */
bool ULyraInventoryManagerComponent::TryAutoStack(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32& Count)
{
    if (Count <= 0)
    {
        return true;
    }

    // 查找所有相同类型的物品
    TArray<ULyraInventoryItemInstance*> ExistingItems = FindAllItemsByDefinition(ItemDef);

    // 按槽位排序（可选，保持整洁）
    ExistingItems.Sort([this](const ULyraInventoryItemInstance& A, const ULyraInventoryItemInstance& B) {
        const FLyraInventoryEntry* EntryA = InventoryList.FindEntry(&A);
        const FLyraInventoryEntry* EntryB = InventoryList.FindEntry(&B);
        if (EntryA && EntryB)
        {
            return EntryA->SlotIndex < EntryB->SlotIndex;
        }
        return false;
    });

    for (ULyraInventoryItemInstance* Item : ExistingItems)
    {
        if (Item == nullptr)
        {
            continue;
        }

        // 尝试堆叠
        int32 Remaining = Item->TryAddStack(Count);

        // 更新 UI
        FLyraInventoryEntry* Entry = InventoryList.FindEntry(Item);
        if (Entry != nullptr)
        {
            InventoryList.MarkItemDirty(*Entry);
            OnItemStackChanged.Broadcast(Item, Item->GetStackCount());
        }

        Count = Remaining;

        if (Count <= 0)
        {
            return true; // 完全堆叠成功
        }
    }

    return Count <= 0;
}
```

#### 1.6.2 手动拆分

```cpp
/**
 * 拆分物品堆叠
 * 
 * 使用场景：
 * - 玩家按住 Shift + 右键拆分堆叠
 * - 交易时拆分物品数量
 * - 制作时拆分材料
 * 
 * @param ItemInstance 源物品
 * @param SplitCount 拆分数量
 * @param TargetSlotIndex 目标槽位（-1 表示自动分配）
 * @return 新创建的物品实例（可能为 nullptr）
 */
ULyraInventoryItemInstance* ULyraInventoryManagerComponent::SplitItemStack(
    ULyraInventoryItemInstance* ItemInstance,
    int32 SplitCount,
    int32 TargetSlotIndex)
{
    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("SplitItemStack: Not called on server!"));
        return nullptr;
    }

    // 参数验证
    if (ItemInstance == nullptr || SplitCount <= 0)
    {
        return nullptr;
    }

    int32 CurrentStack = ItemInstance->GetStackCount();
    if (SplitCount >= CurrentStack)
    {
        UE_LOG(LogTemp, Warning, TEXT("SplitItemStack: Split count %d >= current stack %d!"), SplitCount, CurrentStack);
        return nullptr;
    }

    // 验证目标槽位
    if (bUseSlotSystem)
    {
        if (TargetSlotIndex == -1)
        {
            TargetSlotIndex = FindEmptySlot();
            if (TargetSlotIndex == -1)
            {
                UE_LOG(LogTemp, Warning, TEXT("SplitItemStack: No empty slot available!"));
                return nullptr;
            }
        }
        else if (!IsValidSlotIndex(TargetSlotIndex) || !IsSlotEmpty(TargetSlotIndex))
        {
            UE_LOG(LogTemp, Warning, TEXT("SplitItemStack: Target slot %d is invalid or occupied!"), TargetSlotIndex);
            return nullptr;
        }
    }

    // 减少源物品堆叠
    ItemInstance->TryRemoveStack(SplitCount);

    // 更新源物品
    FLyraInventoryEntry* SourceEntry = InventoryList.FindEntry(ItemInstance);
    if (SourceEntry != nullptr)
    {
        InventoryList.MarkItemDirty(*SourceEntry);
        OnItemStackChanged.Broadcast(ItemInstance, ItemInstance->GetStackCount());
    }

    // 创建新物品
    ULyraInventoryItemInstance* NewItem = CreateItemInstance(ItemInstance->GetItemDef()->GetClass(), SplitCount);
    if (NewItem == nullptr)
    {
        // 回滚
        ItemInstance->TryAddStack(SplitCount);
        UE_LOG(LogTemp, Error, TEXT("SplitItemStack: Failed to create new item!"));
        return nullptr;
    }

    // 添加新物品
    if (!AddItemInstance(NewItem, TargetSlotIndex))
    {
        // 回滚
        ItemInstance->TryAddStack(SplitCount);
        NewItem->MarkAsGarbage();
        UE_LOG(LogTemp, Error, TEXT("SplitItemStack: Failed to add new item to slot!"));
        return nullptr;
    }

    UE_LOG(LogTemp, Log, TEXT("SplitItemStack: Split %s, %d -> %d + %d"),
        *ItemInstance->GetItemDef()->GetName(), CurrentStack, ItemInstance->GetStackCount(), NewItem->GetStackCount());

    return NewItem;
}
```

#### 1.6.3 堆叠合并

```cpp
/**
 * 合并物品堆叠
 * 
 * 使用场景：
 * - 玩家拖拽物品到另一个相同物品上
 * - 自动整理背包
 * 
 * @param SourceItem 源物品（会被减少或移除）
 * @param TargetItem 目标物品（会增加）
 * @return 是否成功合并
 */
bool ULyraInventoryManagerComponent::MergeItemStacks(ULyraInventoryItemInstance* SourceItem, ULyraInventoryItemInstance* TargetItem)
{
    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("MergeItemStacks: Not called on server!"));
        return false;
    }

    // 参数验证
    if (SourceItem == nullptr || TargetItem == nullptr || SourceItem == TargetItem)
    {
        return false;
    }

    // 检查是否可以堆叠
    if (!TargetItem->CanStackWith(SourceItem))
    {
        UE_LOG(LogTemp, Warning, TEXT("MergeItemStacks: Items cannot stack!"));
        return false;
    }

    // 尝试堆叠
    int32 SourceCount = SourceItem->GetStackCount();
    int32 Remaining = TargetItem->TryAddStack(SourceCount);

    // 更新目标物品
    FLyraInventoryEntry* TargetEntry = InventoryList.FindEntry(TargetItem);
    if (TargetEntry != nullptr)
    {
        InventoryList.MarkItemDirty(*TargetEntry);
        OnItemStackChanged.Broadcast(TargetItem, TargetItem->GetStackCount());
    }

    // 如果源物品完全合并，移除它
    if (Remaining == 0)
    {
        RemoveItemInstance(SourceItem, -1);
        UE_LOG(LogTemp, Log, TEXT("MergeItemStacks: Fully merged %s"), *SourceItem->GetItemDef()->GetName());
    }
    else
    {
        // 部分合并，更新源物品
        SourceItem->SetStackCount(Remaining);
        FLyraInventoryEntry* SourceEntry = InventoryList.FindEntry(SourceItem);
        if (SourceEntry != nullptr)
        {
            InventoryList.MarkItemDirty(*SourceEntry);
            OnItemStackChanged.Broadcast(SourceItem, Remaining);
        }
        UE_LOG(LogTemp, Log, TEXT("MergeItemStacks: Partially merged %s, %d remaining"),
            *SourceItem->GetItemDef()->GetName(), Remaining);
    }

    return true;
}
```

---

## 第二部分：物品系统详解（30%）

### 2.1 Item Fragment 碎片化设计

Fragment 是 Lyra 物品系统的核心创新，它将物品的不同特性封装成独立的组件，实现了高度的模块化和可扩展性。

#### 2.1.1 Fragment 架构设计

```
ULyraInventoryItemFragment (基类)
    ├── ULyraInventoryFragment_Stats (属性碎片)
    ├── ULyraInventoryFragment_Equipment (装备碎片)
    ├── ULyraInventoryFragment_Consumable (消耗品碎片)
    ├── ULyraInventoryFragment_Abilities (技能碎片)
    ├── ULyraInventoryFragment_Visual (视觉碎片)
    ├── ULyraInventoryFragment_Crafting (制作碎片)
    └── ... (自定义碎片)
```

#### 2.1.2 属性碎片（Stats Fragment）

属性碎片用于存储物品的数值属性（攻击力、防御力、重量等）。

```cpp
// LyraInventoryFragment_Stats.h
#pragma once

#include "CoreMinimal.h"
#include "LyraInventoryItemFragment.h"
#include "GameplayTagContainer.h"
#include "LyraInventoryFragment_Stats.generated.h"

/**
 * 物品属性定义
 */
USTRUCT(BlueprintType)
struct FLyraItemStat
{
    GENERATED_BODY()

    /** 属性标签（如 "Stat.Damage"、"Stat.Armor"） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Stat")
    FGameplayTag StatTag;

    /** 属性值 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Stat")
    float Value = 0.0f;

    /** 属性描述（用于 UI 显示） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Stat")
    FText Description;

    /** 是否在 UI 中显示 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Stat")
    bool bShowInUI = true;
};

/**
 * 属性碎片：定义物品的数值属性
 * 
 * 适用场景：
 * - 武器：攻击力、暴击率、攻速
 * - 防具：防御力、生命值加成、移速
 * - 消耗品：恢复量、持续时间
 */
UCLASS(DisplayName = "Stats Fragment")
class LYRAGAME_API ULyraInventoryFragment_Stats : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    /**
     * 物品属性列表
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Stats")
    TArray<FLyraItemStat> Stats;

public:
    /**
     * 获取指定属性值
     * 
     * @param StatTag 属性标签
     * @param OutValue [输出] 属性值
     * @return 是否找到属性
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Stats")
    bool GetStatValue(FGameplayTag StatTag, float& OutValue) const;

    /**
     * 获取所有属性
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Stats")
    const TArray<FLyraItemStat>& GetAllStats() const { return Stats; }

    /**
     * 查找属性
     */
    const FLyraItemStat* FindStat(FGameplayTag StatTag) const;
};
```

```cpp
// LyraInventoryFragment_Stats.cpp
#include "LyraInventoryFragment_Stats.h"

bool ULyraInventoryFragment_Stats::GetStatValue(FGameplayTag StatTag, float& OutValue) const
{
    const FLyraItemStat* Stat = FindStat(StatTag);
    if (Stat != nullptr)
    {
        OutValue = Stat->Value;
        return true;
    }
    return false;
}

const FLyraItemStat* ULyraInventoryFragment_Stats::FindStat(FGameplayTag StatTag) const
{
    for (const FLyraItemStat& Stat : Stats)
    {
        if (Stat.StatTag == StatTag)
        {
            return &Stat;
        }
    }
    return nullptr;
}
```

**使用示例**：

```cpp
// 在编辑器中配置：
// Content/Items/Weapons/Sword_Iron.uasset

Fragments:
  [0] Stats Fragment
    Stats:
      [0] 
        StatTag: Stat.Damage
        Value: 25.0
        Description: "攻击力"
        bShowInUI: true
      [1]
        StatTag: Stat.AttackSpeed
        Value: 1.2
        Description: "攻击速度"
        bShowInUI: true
      [2]
        StatTag: Stat.CritChance
        Value: 0.15
        Description: "暴击率 (15%)"
        bShowInUI: true
```

#### 2.1.3 装备碎片（Equipment Fragment）

装备碎片用于定义物品作为装备的行为。

```cpp
// LyraInventoryFragment_Equipment.h
#pragma once

#include "CoreMinimal.h"
#include "LyraInventoryItemFragment.h"
#include "GameplayTagContainer.h"
#include "LyraInventoryFragment_Equipment.generated.h"

class ULyraEquipmentDefinition;
class ULyraAbilitySet;

/**
 * 装备槽位枚举
 */
UENUM(BlueprintType)
enum class ELyraEquipmentSlot : uint8
{
    None           UMETA(DisplayName = "无"),
    MainHand       UMETA(DisplayName = "主手"),
    OffHand        UMETA(DisplayName = "副手"),
    TwoHand        UMETA(DisplayName = "双手"),
    Head           UMETA(DisplayName = "头部"),
    Chest          UMETA(DisplayName = "胸部"),
    Legs           UMETA(DisplayName = "腿部"),
    Feet           UMETA(DisplayName = "脚部"),
    Hands          UMETA(DisplayName = "手部"),
    Back           UMETA(DisplayName = "背部"),
    Accessory1     UMETA(DisplayName = "饰品 1"),
    Accessory2     UMETA(DisplayName = "饰品 2"),
    QuickSlot1     UMETA(DisplayName = "快捷栏 1"),
    QuickSlot2     UMETA(DisplayName = "快捷栏 2"),
    QuickSlot3     UMETA(DisplayName = "快捷栏 3"),
};

/**
 * 装备碎片：定义物品作为装备的行为
 * 
 * 适用场景：
 * - 武器：剑、枪、法杖
 * - 防具：头盔、胸甲、护腿
 * - 饰品：项链、戒指
 */
UCLASS(DisplayName = "Equipment Fragment")
class LYRAGAME_API ULyraInventoryFragment_Equipment : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    /**
     * 装备槽位
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Equipment")
    ELyraEquipmentSlot EquipmentSlot = ELyraEquipmentSlot::None;

    /**
     * 装备定义（引用 Lyra 的 Equipment 系统）
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Equipment")
    TSubclassOf<ULyraEquipmentDefinition> EquipmentDefinition;

    /**
     * 装备时授予的技能集
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Abilities")
    TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant;

    /**
     * 是否可以在战斗中装备
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Equipment")
    bool bCanEquipInCombat = true;

    /**
     * 装备等级要求
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Equipment")
    int32 RequiredLevel = 1;

public:
    //~ULyraInventoryItemFragment interface
    virtual void OnItemEquipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const override;
    virtual void OnItemUnequipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const override;
    //~End of ULyraInventoryItemFragment interface

    /**
     * 检查角色是否满足装备要求
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Equipment")
    bool CanEquip(AActor* Wearer) const;
};
```

```cpp
// LyraInventoryFragment_Equipment.cpp
#include "LyraInventoryFragment_Equipment.h"
#include "LyraEquipmentManagerComponent.h"
#include "LyraEquipmentDefinition.h"
#include "LyraInventoryItemInstance.h"
#include "AbilitySystemGlobals.h"
#include "AbilitySystemComponent.h"
#include "LyraAbilitySet.h"

void ULyraInventoryFragment_Equipment::OnItemEquipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const
{
    if (Wearer == nullptr || ItemInstance == nullptr)
    {
        return;
    }

    UE_LOG(LogTemp, Log, TEXT("OnItemEquipped: %s equipped %s"),
        *Wearer->GetName(), *ItemInstance->GetItemDef()->GetName());

    // 如果有装备定义，使用 Lyra 的装备系统
    if (EquipmentDefinition != nullptr)
    {
        ULyraEquipmentManagerComponent* EquipmentManager = Wearer->FindComponentByClass<ULyraEquipmentManagerComponent>();
        if (EquipmentManager != nullptr)
        {
            // 装备物品
            EquipmentManager->EquipItem(EquipmentDefinition);
        }
    }

    // 授予技能
    if (AbilitySetsToGrant.Num() > 0)
    {
        UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Wearer);
        if (ASC != nullptr)
        {
            for (const ULyraAbilitySet* AbilitySet : AbilitySetsToGrant)
            {
                if (AbilitySet != nullptr)
                {
                    AbilitySet->GiveToAbilitySystem(ASC, nullptr);
                }
            }
        }
    }
}

void ULyraInventoryFragment_Equipment::OnItemUnequipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const
{
    if (Wearer == nullptr || ItemInstance == nullptr)
    {
        return;
    }

    UE_LOG(LogTemp, Log, TEXT("OnItemUnequipped: %s unequipped %s"),
        *Wearer->GetName(), *ItemInstance->GetItemDef()->GetName());

    // 移除装备
    if (EquipmentDefinition != nullptr)
    {
        ULyraEquipmentManagerComponent* EquipmentManager = Wearer->FindComponentByClass<ULyraEquipmentManagerComponent>();
        if (EquipmentManager != nullptr)
        {
            // 卸下物品
            EquipmentManager->UnequipItem(EquipmentDefinition);
        }
    }

    // 移除技能（需要保存授予时的 Handle）
    // 这里简化处理，实际项目中应该在 ItemInstance 中存储 GrantedHandles
}

bool ULyraInventoryFragment_Equipment::CanEquip(AActor* Wearer) const
{
    if (Wearer == nullptr)
    {
        return false;
    }

    // 检查等级要求
    // 这里需要根据你的项目实现角色等级系统
    // 例如：
    // ALyraCharacter* Character = Cast<ALyraCharacter>(Wearer);
    // if (Character && Character->GetLevel() < RequiredLevel)
    // {
    //     return false;
    // }

    return true;
}
```

#### 2.1.4 消耗品碎片（Consumable Fragment）

消耗品碎片用于定义可使用的物品（药水、食物、卷轴等）。

```cpp
// LyraInventoryFragment_Consumable.h
#pragma once

#include "CoreMinimal.h"
#include "LyraInventoryItemFragment.h"
#include "GameplayTagContainer.h"
#include "GameplayEffectTypes.h"
#include "LyraInventoryFragment_Consumable.generated.h"

class UGameplayEffect;
class ULyraGameplayAbility;

/**
 * 消耗品碎片：定义可使用的物品
 * 
 * 适用场景：
 * - 药水：恢复生命、法力
 * - 食物：增加 Buff
 * - 卷轴：施放技能
 */
UCLASS(DisplayName = "Consumable Fragment")
class LYRAGAME_API ULyraInventoryFragment_Consumable : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    /**
     * 使用时应用的 Gameplay Effects
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Consumable")
    TArray<TSubclassOf<UGameplayEffect>> GameplayEffectsToApply;

    /**
     * 使用时激活的技能
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Consumable")
    TSubclassOf<ULyraGameplayAbility> AbilityToActivate;

    /**
     * 使用后是否消耗物品
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Consumable")
    bool bConsumeOnUse = true;

    /**
     * 冷却时间（秒）
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Consumable")
    float CooldownDuration = 0.0f;

    /**
     * 冷却标签（用于共享冷却）
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Consumable")
    FGameplayTag CooldownTag;

    /**
     * 是否可以在战斗中使用
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Consumable")
    bool bCanUseInCombat = true;

    /**
     * 使用动画蒙太奇
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Animation")
    TObjectPtr<UAnimMontage> UseMontage;

public:
    //~ULyraInventoryItemFragment interface
    virtual void OnItemUsed(ULyraInventoryItemInstance* ItemInstance, AActor* User) const override;
    //~End of ULyraInventoryItemFragment interface

    /**
     * 检查是否可以使用
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Consumable")
    bool CanUse(AActor* User) const;

    /**
     * 应用物品效果
     */
    UFUNCTION(BlueprintCallable, Category = "Consumable")
    void ApplyEffects(AActor* User) const;
};
```

```cpp
// LyraInventoryFragment_Consumable.cpp
#include "LyraInventoryFragment_Consumable.h"
#include "LyraInventoryItemInstance.h"
#include "LyraInventoryManagerComponent.h"
#include "AbilitySystemGlobals.h"
#include "AbilitySystemComponent.h"
#include "GameplayEffect.h"
#include "LyraGameplayAbility.h"

void ULyraInventoryFragment_Consumable::OnItemUsed(ULyraInventoryItemInstance* ItemInstance, AActor* User) const
{
    if (User == nullptr || ItemInstance == nullptr)
    {
        return;
    }

    // 检查是否可以使用
    if (!CanUse(User))
    {
        UE_LOG(LogTemp, Warning, TEXT("OnItemUsed: Cannot use %s"), *ItemInstance->GetItemDef()->GetName());
        return;
    }

    UE_LOG(LogTemp, Log, TEXT("OnItemUsed: %s used %s"),
        *User->GetName(), *ItemInstance->GetItemDef()->GetName());

    // 播放使用动画
    if (UseMontage != nullptr)
    {
        // 获取角色的 AnimInstance
        if (ACharacter* Character = Cast<ACharacter>(User))
        {
            if (UAnimInstance* AnimInstance = Character->GetMesh()->GetAnimInstance())
            {
                AnimInstance->Montage_Play(UseMontage);
            }
        }
    }

    // 应用效果
    ApplyEffects(User);

    // 激活技能
    if (AbilityToActivate != nullptr)
    {
        UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(User);
        if (ASC != nullptr)
        {
            // 临时授予技能并激活
            FGameplayAbilitySpec AbilitySpec(AbilityToActivate, 1);
            FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(AbilitySpec);
            ASC->TryActivateAbility(Handle);

            // 技能执行完毕后移除（可以在技能的 EndAbility 中处理）
        }
    }

    // 消耗物品
    if (bConsumeOnUse)
    {
        ULyraInventoryManagerComponent* InventoryManager = User->FindComponentByClass<ULyraInventoryManagerComponent>();
        if (InventoryManager != nullptr)
        {
            InventoryManager->RemoveItemInstance(ItemInstance, 1);
        }
    }

    // 应用冷却
    if (CooldownDuration > 0.0f && CooldownTag.IsValid())
    {
        UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(User);
        if (ASC != nullptr)
        {
            // 创建冷却 Gameplay Effect
            // 这里简化处理，实际项目中应该使用预定义的冷却 GE
            FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
            EffectContext.AddSourceObject(ItemInstance);

            // 应用冷却标签
            ASC->AddLooseGameplayTag(CooldownTag);

            // 设置定时器移除冷却标签
            FTimerHandle TimerHandle;
            User->GetWorldTimerManager().SetTimer(TimerHandle, [ASC, CooldownTag]() {
                if (ASC != nullptr)
                {
                    ASC->RemoveLooseGameplayTag(CooldownTag);
                }
            }, CooldownDuration, false);
        }
    }
}

bool ULyraInventoryFragment_Consumable::CanUse(AActor* User) const
{
    if (User == nullptr)
    {
        return false;
    }

    // 检查冷却
    if (CooldownTag.IsValid())
    {
        UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(User);
        if (ASC != nullptr && ASC->HasMatchingGameplayTag(CooldownTag))
        {
            return false; // 冷却中
        }
    }

    // 检查是否在战斗中
    if (!bCanUseInCombat)
    {
        // 这里需要根据你的项目实现战斗状态检测
        // 例如：
        // if (User->IsInCombat())
        // {
        //     return false;
        // }
    }

    return true;
}

void ULyraInventoryFragment_Consumable::ApplyEffects(AActor* User) const
{
    if (User == nullptr || GameplayEffectsToApply.Num() == 0)
    {
        return;
    }

    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(User);
    if (ASC == nullptr)
    {
        return;
    }

    // 应用所有 Gameplay Effects
    for (const TSubclassOf<UGameplayEffect>& EffectClass : GameplayEffectsToApply)
    {
        if (EffectClass != nullptr)
        {
            FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
            EffectContext.AddSourceObject(this);

            FGameplayEffectSpecHandle EffectSpec = ASC->MakeOutgoingSpec(EffectClass, 1.0f, EffectContext);
            if (EffectSpec.IsValid())
            {
                ASC->ApplyGameplayEffectSpecToSelf(*EffectSpec.Data.Get());
                UE_LOG(LogTemp, Log, TEXT("ApplyEffects: Applied %s"), *EffectClass->GetName());
            }
        }
    }
}
```

**使用示例**：

```cpp
// 在编辑器中配置：
// Content/Items/Consumables/Potion_Health.uasset

Display Name: "生命药水"
Description: "恢复 50 点生命值"
Max Stack Count: 99

Fragments:
  [0] Stats Fragment
    Stats:
      [0] 
        StatTag: Stat.HealAmount
        Value: 50.0
        Description: "恢复量"

  [1] Consumable Fragment
    Gameplay Effects To Apply:
      [0] GE_Item_Potion_Health (自定义 Gameplay Effect)
    Ability To Activate: None
    bConsume On Use: true
    Cooldown Duration: 1.0
    Cooldown Tag: Cooldown.Item.Potion
    bCan Use In Combat: true
    Use Montage: AM_Drink (喝药动画)
```

#### 2.1.5 技能碎片（Abilities Fragment）

技能碎片用于物品授予技能（如技能书、符文）。

```cpp
// LyraInventoryFragment_Abilities.h
#pragma once

#include "CoreMinimal.h"
#include "LyraInventoryItemFragment.h"
#include "LyraInventoryFragment_Abilities.generated.h"

class ULyraAbilitySet;
class ULyraGameplayAbility;

/**
 * 技能碎片：定义物品授予的技能
 * 
 * 适用场景：
 * - 技能书：学习新技能
 * - 符文：临时授予技能
 * - 装备：装备时授予技能
 */
UCLASS(DisplayName = "Abilities Fragment")
class LYRAGAME_API ULyraInventoryFragment_Abilities : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    /**
     * 授予的技能集
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Abilities")
    TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant;

    /**
     * 授予的单个技能
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Abilities")
    TArray<TSubclassOf<ULyraGameplayAbility>> AbilitiesToGrant;

    /**
     * 技能等级
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Abilities")
    int32 AbilityLevel = 1;

    /**
     * 是否永久授予（技能书）还是临时授予（符文）
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Abilities")
    bool bPermanentGrant = false;

public:
    //~ULyraInventoryItemFragment interface
    virtual void OnItemUsed(ULyraInventoryItemInstance* ItemInstance, AActor* User) const override;
    virtual void OnItemEquipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const override;
    virtual void OnItemUnequipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const override;
    //~End of ULyraInventoryItemFragment interface

    /**
     * 授予技能给角色
     */
    UFUNCTION(BlueprintCallable, Category = "Abilities")
    void GrantAbilities(AActor* Target) const;

    /**
     * 移除授予的技能
     */
    UFUNCTION(BlueprintCallable, Category = "Abilities")
    void RemoveAbilities(AActor* Target) const;
};
```

```cpp
// LyraInventoryFragment_Abilities.cpp
#include "LyraInventoryFragment_Abilities.h"
#include "LyraInventoryItemInstance.h"
#include "LyraInventoryManagerComponent.h"
#include "AbilitySystemGlobals.h"
#include "AbilitySystemComponent.h"
#include "LyraAbilitySet.h"
#include "LyraGameplayAbility.h"

void ULyraInventoryFragment_Abilities::OnItemUsed(ULyraInventoryItemInstance* ItemInstance, AActor* User) const
{
    if (User == nullptr || ItemInstance == nullptr)
    {
        return;
    }

    UE_LOG(LogTemp, Log, TEXT("OnItemUsed: %s used ability item %s"),
        *User->GetName(), *ItemInstance->GetItemDef()->GetName());

    // 授予技能
    GrantAbilities(User);

    // 如果是永久授予（如技能书），消耗物品
    if (bPermanentGrant)
    {
        ULyraInventoryManagerComponent* InventoryManager = User->FindComponentByClass<ULyraInventoryManagerComponent>();
        if (InventoryManager != nullptr)
        {
            InventoryManager->RemoveItemInstance(ItemInstance, 1);
        }
    }
}

void ULyraInventoryFragment_Abilities::OnItemEquipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const
{
    if (!bPermanentGrant)
    {
        // 临时授予（如符文装备时）
        GrantAbilities(Wearer);
    }
}

void ULyraInventoryFragment_Abilities::OnItemUnequipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const
{
    if (!bPermanentGrant)
    {
        // 移除临时技能
        RemoveAbilities(Wearer);
    }
}

void ULyraInventoryFragment_Abilities::GrantAbilities(AActor* Target) const
{
    if (Target == nullptr)
    {
        return;
    }

    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Target);
    if (ASC == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("GrantAbilities: Target has no AbilitySystemComponent!"));
        return;
    }

    // 授予 Ability Sets
    for (const ULyraAbilitySet* AbilitySet : AbilitySetsToGrant)
    {
        if (AbilitySet != nullptr)
        {
            // 使用 Lyra 的 Ability Set 系统
            FLyraAbilitySet_GrantedHandles GrantedHandles;
            AbilitySet->GiveToAbilitySystem(ASC, &GrantedHandles);

            // 这里应该保存 GrantedHandles，以便后续移除
            // 实际项目中，可以在 ItemInstance 中存储
            UE_LOG(LogTemp, Log, TEXT("GrantAbilities: Granted AbilitySet %s"), *AbilitySet->GetName());
        }
    }

    // 授予单个技能
    for (const TSubclassOf<ULyraGameplayAbility>& AbilityClass : AbilitiesToGrant)
    {
        if (AbilityClass != nullptr)
        {
            FGameplayAbilitySpec AbilitySpec(AbilityClass, AbilityLevel);
            FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(AbilitySpec);

            // 这里应该保存 Handle，以便后续移除
            UE_LOG(LogTemp, Log, TEXT("GrantAbilities: Granted Ability %s"), *AbilityClass->GetName());
        }
    }
}

void ULyraInventoryFragment_Abilities::RemoveAbilities(AActor* Target) const
{
    if (Target == nullptr)
    {
        return;
    }

    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Target);
    if (ASC == nullptr)
    {
        return;
    }

    // 移除技能
    // 这里需要之前保存的 Handle
    // 实际项目中，应该在 ItemInstance 中存储授予的技能 Handle，然后在这里移除

    UE_LOG(LogTemp, Log, TEXT("RemoveAbilities: Removed abilities from %s"), *Target->GetName());
}
```

#### 2.1.6 视觉碎片（Visual Fragment）

视觉碎片用于定义物品的视觉表现（模型、图标、特效）。

```cpp
// LyraInventoryFragment_Visual.h
#pragma once

#include "CoreMinimal.h"
#include "LyraInventoryItemFragment.h"
#include "LyraInventoryFragment_Visual.generated.h"

class UStaticMesh;
class USkeletalMesh;
class UTexture2D;
class UMaterialInterface;
class UParticleSystem;
class UNiagaraSystem;

/**
 * 物品品质枚举
 */
UENUM(BlueprintType)
enum class ELyraItemRarity : uint8
{
    Common      UMETA(DisplayName = "普通 (白色)"),
    Uncommon    UMETA(DisplayName = "优秀 (绿色)"),
    Rare        UMETA(DisplayName = "稀有 (蓝色)"),
    Epic        UMETA(DisplayName = "史诗 (紫色)"),
    Legendary   UMETA(DisplayName = "传说 (橙色)"),
    Mythic      UMETA(DisplayName = "神话 (红色)"),
};

/**
 * 视觉碎片：定义物品的视觉表现
 * 
 * 适用场景：
 * - UI 显示：图标、边框、颜色
 * - 世界模型：掉落物的 3D 模型
 * - 特效：发光、粒子效果
 */
UCLASS(DisplayName = "Visual Fragment")
class LYRAGAME_API ULyraInventoryFragment_Visual : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    /**
     * 物品图标
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|UI")
    TObjectPtr<UTexture2D> ItemIcon;

    /**
     * 物品品质
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|UI")
    ELyraItemRarity Rarity = ELyraItemRarity::Common;

    /**
     * 品质颜色（覆盖默认颜色）
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|UI")
    FLinearColor RarityColor = FLinearColor::White;

    /**
     * 物品边框材质
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|UI")
    TObjectPtr<UMaterialInterface> BorderMaterial;

    /**
     * 世界模型（掉落物）
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|World")
    TObjectPtr<UStaticMesh> WorldMesh;

    /**
     * 骨骼网格（用于装备）
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|World")
    TObjectPtr<USkeletalMesh> SkeletalMesh;

    /**
     * 物品材质
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|World")
    TObjectPtr<UMaterialInterface> ItemMaterial;

    /**
     * 粒子特效（环绕物品）
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|Effects")
    TObjectPtr<UParticleSystem> ParticleEffect;

    /**
     * Niagara 特效
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|Effects")
    TObjectPtr<UNiagaraSystem> NiagaraEffect;

    /**
     * 发光颜色
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|Effects")
    FLinearColor GlowColor = FLinearColor::White;

    /**
     * 发光强度
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Visual|Effects", meta = (ClampMin = "0.0", ClampMax = "10.0"))
    float GlowIntensity = 1.0f;

public:
    /**
     * 获取品质颜色
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Visual")
    FLinearColor GetRarityColor() const;

    /**
     * 获取品质名称
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Visual")
    FText GetRarityName() const;

    /**
     * 生成世界模型
     */
    UFUNCTION(BlueprintCallable, Category = "Visual")
    AActor* SpawnWorldModel(UWorld* World, const FTransform& Transform) const;
};
```

```cpp
// LyraInventoryFragment_Visual.cpp
#include "LyraInventoryFragment_Visual.h"
#include "Engine/World.h"
#include "Engine/StaticMeshActor.h"

FLinearColor ULyraInventoryFragment_Visual::GetRarityColor() const
{
    // 如果设置了自定义颜色，使用自定义颜色
    if (!RarityColor.Equals(FLinearColor::White))
    {
        return RarityColor;
    }

    // 否则使用默认的品质颜色
    switch (Rarity)
    {
    case ELyraItemRarity::Common:
        return FLinearColor(0.8f, 0.8f, 0.8f); // 灰白色
    case ELyraItemRarity::Uncommon:
        return FLinearColor(0.1f, 0.9f, 0.1f); // 绿色
    case ELyraItemRarity::Rare:
        return FLinearColor(0.0f, 0.5f, 1.0f); // 蓝色
    case ELyraItemRarity::Epic:
        return FLinearColor(0.6f, 0.2f, 0.9f); // 紫色
    case ELyraItemRarity::Legendary:
        return FLinearColor(1.0f, 0.6f, 0.0f); // 橙色
    case ELyraItemRarity::Mythic:
        return FLinearColor(1.0f, 0.0f, 0.0f); // 红色
    default:
        return FLinearColor::White;
    }
}

FText ULyraInventoryFragment_Visual::GetRarityName() const
{
    switch (Rarity)
    {
    case ELyraItemRarity::Common:
        return FText::FromString(TEXT("普通"));
    case ELyraItemRarity::Uncommon:
        return FText::FromString(TEXT("优秀"));
    case ELyraItemRarity::Rare:
        return FText::FromString(TEXT("稀有"));
    case ELyraItemRarity::Epic:
        return FText::FromString(TEXT("史诗"));
    case ELyraItemRarity::Legendary:
        return FText::FromString(TEXT("传说"));
    case ELyraItemRarity::Mythic:
        return FText::FromString(TEXT("神话"));
    default:
        return FText::FromString(TEXT("未知"));
    }
}

AActor* ULyraInventoryFragment_Visual::SpawnWorldModel(UWorld* World, const FTransform& Transform) const
{
    if (World == nullptr || WorldMesh == nullptr)
    {
        return nullptr;
    }

    // 生成静态网格 Actor
    FActorSpawnParameters SpawnParams;
    SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

    AStaticMeshActor* MeshActor = World->SpawnActor<AStaticMeshActor>(Transform.GetLocation(), Transform.GetRotation().Rotator(), SpawnParams);
    if (MeshActor != nullptr)
    {
        MeshActor->GetStaticMeshComponent()->SetStaticMesh(WorldMesh);
        MeshActor->SetActorScale3D(Transform.GetScale3D());

        // 应用材质
        if (ItemMaterial != nullptr)
        {
            MeshActor->GetStaticMeshComponent()->SetMaterial(0, ItemMaterial);
        }

        // 添加粒子特效
        if (ParticleEffect != nullptr)
        {
            // 创建粒子组件
            // 这里简化处理，实际项目中应该创建并附加组件
        }

        UE_LOG(LogTemp, Log, TEXT("SpawnWorldModel: Spawned world model at %s"), *Transform.GetLocation().ToString());
    }

    return MeshActor;
}
```

### 2.2 物品属性系统

物品属性系统是 RPG 游戏的核心，它定义了物品的数值特性（攻击力、防御力、暴击率等）。

#### 2.2.1 属性标签体系

使用 Gameplay Tags 管理物品属性：

```cpp
// GameplayTags.ini

[/Script/GameplayTags.GameplayTagsSettings]
GameplayTagList=(Tag="Stat.Damage",DevComment="攻击力")
GameplayTagList=(Tag="Stat.Armor",DevComment="防御力")
GameplayTagList=(Tag="Stat.AttackSpeed",DevComment="攻击速度")
GameplayTagList=(Tag="Stat.CritChance",DevComment="暴击率")
GameplayTagList=(Tag="Stat.CritDamage",DevComment="暴击伤害")
GameplayTagList=(Tag="Stat.Health",DevComment="生命值")
GameplayTagList=(Tag="Stat.MaxHealth",DevComment="最大生命值")
GameplayTagList=(Tag="Stat.Mana",DevComment="法力值")
GameplayTagList=(Tag="Stat.MaxMana",DevComment="最大法力值")
GameplayTagList=(Tag="Stat.MoveSpeed",DevComment="移动速度")
GameplayTagList=(Tag="Stat.Weight",DevComment="重量")
GameplayTagList=(Tag="Stat.Durability",DevComment="耐久度")
GameplayTagList=(Tag="Stat.MaxDurability",DevComment="最大耐久度")
```

#### 2.2.2 属性计算器

```cpp
// LyraItemStatCalculator.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/NoExportTypes.h"
#include "GameplayTagContainer.h"
#include "LyraItemStatCalculator.generated.h"

class ULyraInventoryItemInstance;
class UAbilitySystemComponent;

/**
 * 物品属性计算器
 * 
 * 负责计算物品的最终属性（基础属性 + 强化 + 附魔 + Buff）
 */
UCLASS(BlueprintType)
class LYRAGAME_API ULyraItemStatCalculator : public UObject
{
    GENERATED_BODY()

public:
    /**
     * 计算物品的最终属性值
     * 
     * @param ItemInstance 物品实例
     * @param StatTag 属性标签
     * @param OutValue [输出] 属性值
     * @return 是否找到属性
     */
    UFUNCTION(BlueprintCallable, Category = "ItemStats")
    static bool CalculateItemStat(ULyraInventoryItemInstance* ItemInstance, FGameplayTag StatTag, float& OutValue);

    /**
     * 将物品属性应用到 ASC
     * 
     * @param ItemInstance 物品实例
     * @param ASC 目标 ASC
     */
    UFUNCTION(BlueprintCallable, Category = "ItemStats")
    static void ApplyItemStatsToASC(ULyraInventoryItemInstance* ItemInstance, UAbilitySystemComponent* ASC);

    /**
     * 从 ASC 移除物品属性
     */
    UFUNCTION(BlueprintCallable, Category = "ItemStats")
    static void RemoveItemStatsFromASC(ULyraInventoryItemInstance* ItemInstance, UAbilitySystemComponent* ASC);

    /**
     * 获取物品的所有属性
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "ItemStats")
    static TMap<FGameplayTag, float> GetAllItemStats(ULyraInventoryItemInstance* ItemInstance);
};
```

```cpp
// LyraItemStatCalculator.cpp
#include "LyraItemStatCalculator.h"
#include "LyraInventoryItemInstance.h"
#include "LyraInventoryItemDefinition.h"
#include "LyraInventoryFragment_Stats.h"
#include "AbilitySystemComponent.h"
#include "GameplayEffect.h"

bool ULyraItemStatCalculator::CalculateItemStat(ULyraInventoryItemInstance* ItemInstance, FGameplayTag StatTag, float& OutValue)
{
    if (ItemInstance == nullptr || !StatTag.IsValid())
    {
        return false;
    }

    // 获取属性碎片
    const ULyraInventoryFragment_Stats* StatsFragment = ItemInstance->FindFragmentByClass<ULyraInventoryFragment_Stats>();
    if (StatsFragment == nullptr)
    {
        return false;
    }

    // 获取基础属性值
    if (!StatsFragment->GetStatValue(StatTag, OutValue))
    {
        return false;
    }

    // 这里可以添加强化、附魔等修正
    // 例如：
    // OutValue *= (1.0f + GetEnhancementBonus(ItemInstance));
    // OutValue += GetEnchantmentBonus(ItemInstance, StatTag);

    return true;
}

void ULyraItemStatCalculator::ApplyItemStatsToASC(ULyraInventoryItemInstance* ItemInstance, UAbilitySystemComponent* ASC)
{
    if (ItemInstance == nullptr || ASC == nullptr)
    {
        return;
    }

    // 获取所有属性
    TMap<FGameplayTag, float> Stats = GetAllItemStats(ItemInstance);

    // 为每个属性创建并应用 Gameplay Effect
    for (const auto& StatPair : Stats)
    {
        FGameplayTag StatTag = StatPair.Key;
        float StatValue = StatPair.Value;

        // 这里应该使用预定义的 Gameplay Effect
        // 例如：GE_Item_StatModifier
        // 并设置 Modifier 的 Magnitude 为 StatValue

        // 简化示例：
        // FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
        // FGameplayEffectSpecHandle EffectSpec = ASC->MakeOutgoingSpec(GE_Item_StatModifier, 1.0f, EffectContext);
        // EffectSpec.Data->SetSetByCallerMagnitude(StatTag, StatValue);
        // ASC->ApplyGameplayEffectSpecToSelf(*EffectSpec.Data.Get());

        UE_LOG(LogTemp, Log, TEXT("ApplyItemStatsToASC: Applied %s = %f"), *StatTag.ToString(), StatValue);
    }
}

void ULyraItemStatCalculator::RemoveItemStatsFromASC(ULyraInventoryItemInstance* ItemInstance, UAbilitySystemComponent* ASC)
{
    if (ItemInstance == nullptr || ASC == nullptr)
    {
        return;
    }

    // 移除之前应用的 Gameplay Effects
    // 这里需要保存应用时的 FActiveGameplayEffectHandle，然后在这里移除
    // 实际项目中，应该在 ItemInstance 中存储这些 Handle

    UE_LOG(LogTemp, Log, TEXT("RemoveItemStatsFromASC: Removed stats from ASC"));
}

TMap<FGameplayTag, float> ULyraItemStatCalculator::GetAllItemStats(ULyraInventoryItemInstance* ItemInstance)
{
    TMap<FGameplayTag, float> Stats;

    if (ItemInstance == nullptr)
    {
        return Stats;
    }

    // 获取属性碎片
    const ULyraInventoryFragment_Stats* StatsFragment = ItemInstance->FindFragmentByClass<ULyraInventoryFragment_Stats>();
    if (StatsFragment != nullptr)
    {
        for (const FLyraItemStat& Stat : StatsFragment->GetAllStats())
        {
            Stats.Add(Stat.StatTag, Stat.Value);
        }
    }

    return Stats;
}
```

### 2.3 物品品质与稀有度

品质系统是 RPG 游戏中常见的设计，用于区分物品的价值和稀有程度。

#### 2.3.1 品质系统实现

品质系统已经在 `ULyraInventoryFragment_Visual` 中定义了 `ELyraItemRarity` 枚举。

**品质颜色标准**（参考主流游戏）：

| 品质 | 颜色 | 掉落率 | 用途 |
|------|------|--------|------|
| 普通 (Common) | 灰白色 | 70% | 基础物品 |
| 优秀 (Uncommon) | 绿色 | 20% | 稍好的物品 |
| 稀有 (Rare) | 蓝色 | 7% | 好物品 |
| 史诗 (Epic) | 紫色 | 2% | 极品物品 |
| 传说 (Legendary) | 橙色 | 0.9% | 顶级物品 |
| 神话 (Mythic) | 红色 | 0.1% | 超稀有物品 |

#### 2.3.2 品质影响属性

```cpp
/**
 * 根据品质修正物品属性
 * 
 * 例如：
 * - 普通品质：属性 x1.0
 * - 优秀品质：属性 x1.2
 * - 稀有品质：属性 x1.5
 * - 史诗品质：属性 x2.0
 * - 传说品质：属性 x3.0
 * - 神话品质：属性 x5.0
 */
float GetRarityMultiplier(ELyraItemRarity Rarity)
{
    switch (Rarity)
    {
    case ELyraItemRarity::Common:
        return 1.0f;
    case ELyraItemRarity::Uncommon:
        return 1.2f;
    case ELyraItemRarity::Rare:
        return 1.5f;
    case ELyraItemRarity::Epic:
        return 2.0f;
    case ELyraItemRarity::Legendary:
        return 3.0f;
    case ELyraItemRarity::Mythic:
        return 5.0f;
    default:
        return 1.0f;
    }
}

/**
 * 应用品质修正到属性
 */
float ApplyRarityToStat(float BaseStat, ELyraItemRarity Rarity)
{
    return BaseStat * GetRarityMultiplier(Rarity);
}
```

#### 2.3.3 随机品质生成

```cpp
/**
 * 随机生成物品品质
 * 
 * 基于概率表
 */
ELyraItemRarity GenerateRandomRarity()
{
    float Roll = FMath::FRand(); // 0.0 ~ 1.0

    if (Roll < 0.001f)          // 0.1%
        return ELyraItemRarity::Mythic;
    else if (Roll < 0.01f)      // 0.9%
        return ELyraItemRarity::Legendary;
    else if (Roll < 0.03f)      // 2%
        return ELyraItemRarity::Epic;
    else if (Roll < 0.10f)      // 7%
        return ELyraItemRarity::Rare;
    else if (Roll < 0.30f)      // 20%
        return ELyraItemRarity::Uncommon;
    else                        // 70%
        return ELyraItemRarity::Common;
}
```

### 2.4 物品标签与分类

#### 2.4.1 物品分类标签

```cpp
// GameplayTags.ini

[/Script/GameplayTags.GameplayTagsSettings]
# 物品类型
GameplayTagList=(Tag="Item.Type.Weapon",DevComment="武器")
GameplayTagList=(Tag="Item.Type.Weapon.Melee",DevComment="近战武器")
GameplayTagList=(Tag="Item.Type.Weapon.Ranged",DevComment="远程武器")
GameplayTagList=(Tag="Item.Type.Armor",DevComment="防具")
GameplayTagList=(Tag="Item.Type.Consumable",DevComment="消耗品")
GameplayTagList=(Tag="Item.Type.Material",DevComment="材料")
GameplayTagList=(Tag="Item.Type.Quest",DevComment="任务物品")
GameplayTagList=(Tag="Item.Type.Misc",DevComment="杂物")

# 物品标签
GameplayTagList=(Tag="Item.Tag.Soulbound",DevComment="绑定")
GameplayTagList=(Tag="Item.Tag.Unique",DevComment="唯一")
GameplayTagList=(Tag="Item.Tag.Tradable",DevComment="可交易")
GameplayTagList=(Tag="Item.Tag.Sellable",DevComment="可出售")
GameplayTagList=(Tag="Item.Tag.Stackable",DevComment="可堆叠")
```

#### 2.4.2 标签管理碎片

```cpp
// LyraInventoryFragment_Tags.h
#pragma once

#include "CoreMinimal.h"
#include "LyraInventoryItemFragment.h"
#include "GameplayTagContainer.h"
#include "LyraInventoryFragment_Tags.generated.h"

/**
 * 标签碎片：定义物品的分类和特性标签
 */
UCLASS(DisplayName = "Tags Fragment")
class LYRAGAME_API ULyraInventoryFragment_Tags : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    /**
     * 物品标签
     */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Tags")
    FGameplayTagContainer ItemTags;

public:
    /**
     * 检查是否有指定标签
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Tags")
    bool HasTag(FGameplayTag Tag) const;

    /**
     * 检查是否有任意匹配的标签
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Tags")
    bool HasAnyTag(const FGameplayTagContainer& Tags) const;

    /**
     * 检查是否有所有匹配的标签
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Tags")
    bool HasAllTags(const FGameplayTagContainer& Tags) const;
};
```

```cpp
// LyraInventoryFragment_Tags.cpp
#include "LyraInventoryFragment_Tags.h"

bool ULyraInventoryFragment_Tags::HasTag(FGameplayTag Tag) const
{
    return ItemTags.HasTag(Tag);
}

bool ULyraInventoryFragment_Tags::HasAnyTag(const FGameplayTagContainer& Tags) const
{
    return ItemTags.HasAny(Tags);
}

bool ULyraInventoryFragment_Tags::HasAllTags(const FGameplayTagContainer& Tags) const
{
    return ItemTags.HasAll(Tags);
}
```

### 2.5 物品图标与 UI 集成

物品 UI 是玩家与背包系统交互的主要界面。

#### 2.5.1 物品槽位 Widget

```cpp
// WBP_InventorySlot.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "WBP_InventorySlot.generated.h"

class ULyraInventoryItemInstance;
class UImage;
class UTextBlock;
class UBorder;

/**
 * 物品槽位 Widget
 * 
 * 显示单个物品槽位（图标、堆叠数、品质边框）
 */
UCLASS()
class LYRAGAME_API UWBP_InventorySlot : public UUserWidget
{
    GENERATED_BODY()

public:
    /**
     * 设置物品
     */
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void SetItem(ULyraInventoryItemInstance* NewItem);

    /**
     * 清空槽位
     */
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void ClearSlot();

    /**
     * 获取当前物品
     */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    ULyraInventoryItemInstance* GetItem() const { return ItemInstance; }

protected:
    /**
     * 更新显示
     */
    UFUNCTION(BlueprintCallable, BlueprintImplementableEvent, Category = "Inventory")
    void UpdateDisplay();

    /**
     * 当物品改变时调用（蓝图实现）
     */
    UFUNCTION(BlueprintCallable, BlueprintImplementableEvent, Category = "Inventory")
    void OnItemChanged(ULyraInventoryItemInstance* NewItem);

protected:
    /** 当前物品实例 */
    UPROPERTY(BlueprintReadOnly, Category = "Inventory")
    TObjectPtr<ULyraInventoryItemInstance> ItemInstance;

    /** 物品图标 */
    UPROPERTY(BlueprintReadOnly, meta = (BindWidget))
    TObjectPtr<UImage> Image_Icon;

    /** 堆叠数文本 */
    UPROPERTY(BlueprintReadOnly, meta = (BindWidget))
    TObjectPtr<UTextBlock> Text_StackCount;

    /** 品质边框 */
    UPROPERTY(BlueprintReadOnly, meta = (BindWidget))
    TObjectPtr<UBorder> Border_Rarity;

    /** 槽位索引 */
    UPROPERTY(BlueprintReadOnly, Category = "Inventory")
    int32 SlotIndex = -1;

public:
    /** 设置槽位索引 */
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void SetSlotIndex(int32 Index) { SlotIndex = Index; }

    /** 获取槽位索引 */
    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
    int32 GetSlotIndex() const { return SlotIndex; }
};
```

```cpp
// WBP_InventorySlot.cpp
#include "WBP_InventorySlot.h"
#include "LyraInventoryItemInstance.h"
#include "LyraInventoryItemDefinition.h"
#include "LyraInventoryFragment_Visual.h"
#include "Components/Image.h"
#include "Components/TextBlock.h"
#include "Components/Border.h"

void UWBP_InventorySlot::SetItem(ULyraInventoryItemInstance* NewItem)
{
    ItemInstance = NewItem;

    if (ItemInstance != nullptr)
    {
        // 获取视觉碎片
        const ULyraInventoryFragment_Visual* VisualFragment = ItemInstance->FindFragmentByClass<ULyraInventoryFragment_Visual>();

        // 更新图标
        if (Image_Icon != nullptr && VisualFragment != nullptr && VisualFragment->ItemIcon != nullptr)
        {
            Image_Icon->SetBrushFromTexture(VisualFragment->ItemIcon);
            Image_Icon->SetVisibility(ESlateVisibility::Visible);
        }
        else
        {
            Image_Icon->SetVisibility(ESlateVisibility::Hidden);
        }

        // 更新堆叠数
        if (Text_StackCount != nullptr)
        {
            int32 StackCount = ItemInstance->GetStackCount();
            if (StackCount > 1)
            {
                Text_StackCount->SetText(FText::AsNumber(StackCount));
                Text_StackCount->SetVisibility(ESlateVisibility::Visible);
            }
            else
            {
                Text_StackCount->SetVisibility(ESlateVisibility::Hidden);
            }
        }

        // 更新品质边框
        if (Border_Rarity != nullptr && VisualFragment != nullptr)
        {
            FLinearColor RarityColor = VisualFragment->GetRarityColor();
            Border_Rarity->SetBrushColor(RarityColor);
        }
    }
    else
    {
        // 清空显示
        if (Image_Icon != nullptr)
        {
            Image_Icon->SetVisibility(ESlateVisibility::Hidden);
        }

        if (Text_StackCount != nullptr)
        {
            Text_StackCount->SetVisibility(ESlateVisibility::Hidden);
        }

        if (Border_Rarity != nullptr)
        {
            Border_Rarity->SetBrushColor(FLinearColor::White);
        }
    }

    // 调用蓝图事件
    OnItemChanged(NewItem);

    // 更新显示
    UpdateDisplay();
}

void UWBP_InventorySlot::ClearSlot()
{
    SetItem(nullptr);
}
```

#### 2.5.2 背包界面 Widget

```cpp
// WBP_InventoryPanel.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "WBP_InventoryPanel.generated.h"

class ULyraInventoryManagerComponent;
class UWBP_InventorySlot;
class UUniformGridPanel;

/**
 * 背包界面 Widget
 * 
 * 显示整个背包（多个槽位组成的网格）
 */
UCLASS()
class LYRAGAME_API UWBP_InventoryPanel : public UUserWidget
{
    GENERATED_BODY()

public:
    /**
     * 初始化背包界面
     */
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void InitializeInventory(ULyraInventoryManagerComponent* InInventoryManager);

    /**
     * 刷新背包显示
     */
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void RefreshInventory();

protected:
    virtual void NativeConstruct() override;
    virtual void NativeDestruct() override;

    /**
     * 创建槽位 Widget
     */
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void CreateSlots();

    /**
     * 事件回调：物品添加
     */
    UFUNCTION()
    void OnItemAdded(ULyraInventoryItemInstance* ItemInstance, int32 SlotIndex);

    /**
     * 事件回调：物品移除
     */
    UFUNCTION()
    void OnItemRemoved(ULyraInventoryItemInstance* ItemInstance, int32 Count);

    /**
     * 事件回调：物品移动
     */
    UFUNCTION()
    void OnItemMoved(ULyraInventoryItemInstance* ItemInstance, int32 FromSlot, int32 ToSlot);

    /**
     * 事件回调：物品堆叠改变
     */
    UFUNCTION()
    void OnItemStackChanged(ULyraInventoryItemInstance* ItemInstance, int32 NewCount);

protected:
    /** 背包管理器 */
    UPROPERTY(BlueprintReadOnly, Category = "Inventory")
    TObjectPtr<ULyraInventoryManagerComponent> InventoryManager;

    /** 槽位 Widget 类 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Inventory")
    TSubclassOf<UWBP_InventorySlot> SlotWidgetClass;

    /** 槽位网格 */
    UPROPERTY(BlueprintReadOnly, meta = (BindWidget))
    TObjectPtr<UUniformGridPanel> GridPanel_Slots;

    /** 所有槽位 Widget */
    UPROPERTY(BlueprintReadOnly, Category = "Inventory")
    TArray<TObjectPtr<UWBP_InventorySlot>> SlotWidgets;

    /** 网格列数 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Inventory")
    int32 GridColumns = 8;
};
```

```cpp
// WBP_InventoryPanel.cpp
#include "WBP_InventoryPanel.h"
#include "LyraInventoryManagerComponent.h"
#include "LyraInventoryItemInstance.h"
#include "WBP_InventorySlot.h"
#include "Components/UniformGridPanel.h"

void UWBP_InventoryPanel::NativeConstruct()
{
    Super::NativeConstruct();

    CreateSlots();
}

void UWBP_InventoryPanel::NativeDestruct()
{
    // 解绑事件
    if (InventoryManager != nullptr)
    {
        InventoryManager->OnItemAdded.RemoveAll(this);
        InventoryManager->OnItemRemoved.RemoveAll(this);
        InventoryManager->OnItemMoved.RemoveAll(this);
        InventoryManager->OnItemStackChanged.RemoveAll(this);
    }

    Super::NativeDestruct();
}

void UWBP_InventoryPanel::InitializeInventory(ULyraInventoryManagerComponent* InInventoryManager)
{
    InventoryManager = InInventoryManager;

    if (InventoryManager != nullptr)
    {
        // 绑定事件
        InventoryManager->OnItemAdded.AddDynamic(this, &UWBP_InventoryPanel::OnItemAdded);
        InventoryManager->OnItemRemoved.AddDynamic(this, &UWBP_InventoryPanel::OnItemRemoved);
        InventoryManager->OnItemMoved.AddDynamic(this, &UWBP_InventoryPanel::OnItemMoved);
        InventoryManager->OnItemStackChanged.AddDynamic(this, &UWBP_InventoryPanel::OnItemStackChanged);

        // 刷新显示
        RefreshInventory();
    }
}

void UWBP_InventoryPanel::RefreshInventory()
{
    if (InventoryManager == nullptr)
    {
        return;
    }

    // 清空所有槽位
    for (UWBP_InventorySlot* SlotWidget : SlotWidgets)
    {
        if (SlotWidget != nullptr)
        {
            SlotWidget->ClearSlot();
        }
    }

    // 填充物品
    TArray<ULyraInventoryItemInstance*> AllItems = InventoryManager->GetAllItems();
    for (ULyraInventoryItemInstance* Item : AllItems)
    {
        if (Item != nullptr)
        {
            // 查找物品所在槽位
            // 这里需要 InventoryManager 提供槽位信息
            // 简化处理：遍历所有 Entry
            // 实际项目中应该优化

            int32 SlotIndex = -1;
            // TODO: 获取物品槽位索引

            if (SlotIndex >= 0 && SlotIndex < SlotWidgets.Num())
            {
                SlotWidgets[SlotIndex]->SetItem(Item);
            }
        }
    }
}

void UWBP_InventoryPanel::CreateSlots()
{
    if (GridPanel_Slots == nullptr || SlotWidgetClass == nullptr)
    {
        return;
    }

    // 清空现有槽位
    GridPanel_Slots->ClearChildren();
    SlotWidgets.Empty();

    // 计算槽位数量
    int32 SlotCount = (InventoryManager != nullptr) ? InventoryManager->GetMaxCapacity() : 40;

    // 创建槽位
    for (int32 i = 0; i < SlotCount; ++i)
    {
        UWBP_InventorySlot* SlotWidget = CreateWidget<UWBP_InventorySlot>(this, SlotWidgetClass);
        if (SlotWidget != nullptr)
        {
            SlotWidget->SetSlotIndex(i);

            // 添加到网格
            int32 Row = i / GridColumns;
            int32 Column = i % GridColumns;
            GridPanel_Slots->AddChildToUniformGrid(SlotWidget, Row, Column);

            SlotWidgets.Add(SlotWidget);
        }
    }

    UE_LOG(LogTemp, Log, TEXT("CreateSlots: Created %d slots"), SlotWidgets.Num());
}

void UWBP_InventoryPanel::OnItemAdded(ULyraInventoryItemInstance* ItemInstance, int32 SlotIndex)
{
    if (SlotIndex >= 0 && SlotIndex < SlotWidgets.Num())
    {
        SlotWidgets[SlotIndex]->SetItem(ItemInstance);
    }
}

void UWBP_InventoryPanel::OnItemRemoved(ULyraInventoryItemInstance* ItemInstance, int32 Count)
{
    // 找到物品所在槽位并更新
    for (UWBP_InventorySlot* SlotWidget : SlotWidgets)
    {
        if (SlotWidget != nullptr && SlotWidget->GetItem() == ItemInstance)
        {
            if (ItemInstance->GetStackCount() <= 0)
            {
                SlotWidget->ClearSlot();
            }
            else
            {
                SlotWidget->SetItem(ItemInstance); // 刷新显示
            }
            break;
        }
    }
}

void UWBP_InventoryPanel::OnItemMoved(ULyraInventoryItemInstance* ItemInstance, int32 FromSlot, int32 ToSlot)
{
    // 清空源槽位
    if (FromSlot >= 0 && FromSlot < SlotWidgets.Num())
    {
        SlotWidgets[FromSlot]->ClearSlot();
    }

    // 更新目标槽位
    if (ToSlot >= 0 && ToSlot < SlotWidgets.Num())
    {
        SlotWidgets[ToSlot]->SetItem(ItemInstance);
    }
}

void UWBP_InventoryPanel::OnItemStackChanged(ULyraInventoryItemInstance* ItemInstance, int32 NewCount)
{
    // 找到物品所在槽位并更新
    for (UWBP_InventorySlot* SlotWidget : SlotWidgets)
    {
        if (SlotWidget != nullptr && SlotWidget->GetItem() == ItemInstance)
        {
            SlotWidget->SetItem(ItemInstance); // 刷新显示
            break;
        }
    }
}
```

---

## 第三部分：与 GAS 集成（20%）

Lyra 的强大之处在于其与 GAS（Gameplay Ability System）的深度集成。物品系统通过 GAS 实现了复杂的游戏逻辑，如技能授予、属性修改、效果应用等。

### 3.1 物品如何授予 Ability

物品可以通过多种方式授予 Ability：

1. **装备时授予**：装备武器授予攻击技能
2. **使用时激活**：消耗品激活治疗技能
3. **永久学习**：技能书永久授予技能

#### 3.1.1 装备授予 Ability

```cpp
/**
 * 装备时授予 Ability 的流程：
 * 
 * 1. 玩家装备物品
 * 2. EquipmentFragment 触发 OnItemEquipped
 * 3. 从 Ability Sets 中授予技能
 * 4. 保存授予的 Ability Handles（用于卸下时移除）
 */

// LyraInventoryFragment_Equipment.cpp
void ULyraInventoryFragment_Equipment::OnItemEquipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const
{
    if (Wearer == nullptr || ItemInstance == nullptr)
    {
        return;
    }

    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Wearer);
    if (ASC == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("OnItemEquipped: No ASC found!"));
        return;
    }

    // 授予技能
    FLyraAbilitySet_GrantedHandles GrantedHandles;

    for (const ULyraAbilitySet* AbilitySet : AbilitySetsToGrant)
    {
        if (AbilitySet != nullptr)
        {
            AbilitySet->GiveToAbilitySystem(ASC, &GrantedHandles);
            UE_LOG(LogTemp, Log, TEXT("OnItemEquipped: Granted AbilitySet %s"), *AbilitySet->GetName());
        }
    }

    // 保存 Handles（需要扩展 ItemInstance 来存储）
    // 例如：ItemInstance->EquipmentGrantedHandles = GrantedHandles;
}

void ULyraInventoryFragment_Equipment::OnItemUnequipped(ULyraInventoryItemInstance* ItemInstance, AActor* Wearer) const
{
    if (Wearer == nullptr || ItemInstance == nullptr)
    {
        return;
    }

    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Wearer);
    if (ASC == nullptr)
    {
        return;
    }

    // 移除技能
    // 从 ItemInstance 中获取之前保存的 Handles
    // 例如：ItemInstance->EquipmentGrantedHandles.TakeFromAbilitySystem(ASC);

    UE_LOG(LogTemp, Log, TEXT("OnItemUnequipped: Removed abilities"));
}
```

#### 3.1.2 扩展 ItemInstance 存储 Ability Handles

```cpp
// LyraInventoryItemInstance.h（扩展）

/**
 * 装备授予的技能 Handles（用于卸下时移除）
 */
UPROPERTY(NotReplicated)
FLyraAbilitySet_GrantedHandles EquipmentGrantedHandles;

/**
 * 获取授予的 Handles
 */
FLyraAbilitySet_GrantedHandles& GetEquipmentGrantedHandles() { return EquipmentGrantedHandles; }
```

#### 3.1.3 完整的装备/卸下流程

```cpp
// LyraEquipmentManagerComponent.cpp（简化示例）

/**
 * 装备物品
 */
void ULyraEquipmentManagerComponent::EquipItemFromInventory(ULyraInventoryItemInstance* ItemInstance, ELyraEquipmentSlot Slot)
{
    if (ItemInstance == nullptr || GetOwner() == nullptr)
    {
        return;
    }

    // 获取装备碎片
    const ULyraInventoryFragment_Equipment* EquipmentFragment = ItemInstance->FindFragmentByClass<ULyraInventoryFragment_Equipment>();
    if (EquipmentFragment == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("EquipItemFromInventory: Item is not equipable!"));
        return;
    }

    // 检查槽位是否匹配
    if (EquipmentFragment->EquipmentSlot != Slot)
    {
        UE_LOG(LogTemp, Warning, TEXT("EquipItemFromInventory: Slot mismatch!"));
        return;
    }

    // 卸下当前装备（如果有）
    ULyraInventoryItemInstance* CurrentItem = GetEquippedItem(Slot);
    if (CurrentItem != nullptr)
    {
        UnequipItem(CurrentItem);
    }

    // 装备新物品
    EquippedItems.Add(Slot, ItemInstance);

    // 触发装备回调
    ItemInstance->OnItemEquipped(GetOwner());

    // 触发 Fragment 回调
    if (ItemInstance->GetItemDef() != nullptr)
    {
        for (const ULyraInventoryItemFragment* Fragment : ItemInstance->GetItemDef()->Fragments)
        {
            if (Fragment != nullptr)
            {
                Fragment->OnItemEquipped(ItemInstance, GetOwner());
            }
        }
    }

    UE_LOG(LogTemp, Log, TEXT("EquipItemFromInventory: Equipped %s to slot %d"),
        *ItemInstance->GetItemDef()->GetName(), (int32)Slot);
}

/**
 * 卸下物品
 */
void ULyraEquipmentManagerComponent::UnequipItem(ULyraInventoryItemInstance* ItemInstance)
{
    if (ItemInstance == nullptr)
    {
        return;
    }

    // 查找槽位
    ELyraEquipmentSlot Slot = ELyraEquipmentSlot::None;
    for (const auto& Pair : EquippedItems)
    {
        if (Pair.Value == ItemInstance)
        {
            Slot = Pair.Key;
            break;
        }
    }

    if (Slot == ELyraEquipmentSlot::None)
    {
        UE_LOG(LogTemp, Warning, TEXT("UnequipItem: Item not equipped!"));
        return;
    }

    // 触发卸下回调
    ItemInstance->OnItemUnequipped(GetOwner());

    // 触发 Fragment 回调
    if (ItemInstance->GetItemDef() != nullptr)
    {
        for (const ULyraInventoryItemFragment* Fragment : ItemInstance->GetItemDef()->Fragments)
        {
            if (Fragment != nullptr)
            {
                Fragment->OnItemUnequipped(ItemInstance, GetOwner());
            }
        }
    }

    // 移除装备
    EquippedItems.Remove(Slot);

    UE_LOG(LogTemp, Log, TEXT("UnequipItem: Unequipped %s from slot %d"),
        *ItemInstance->GetItemDef()->GetName(), (int32)Slot);
}
```

### 3.2 可消耗物品（药水、食物）

消耗品通过 Gameplay Effects 来实现效果。

#### 3.2.1 创建治疗药水的 Gameplay Effect

```cpp
// GE_Item_Potion_Health.h（蓝图配置）

/**
 * 治疗药水 Gameplay Effect
 * 
 * 配置：
 * - Duration Policy: Instant（瞬间）
 * - Modifiers:
 *   [0] Attribute: Health
 *       Modifier Op: Add
 *       Modifier Magnitude: 50.0
 */
```

在编辑器中创建 Gameplay Effect：

```
Content/GameplayEffects/Items/GE_Item_Potion_Health

Duration Policy: Instant
Modifiers:
  [0] 
    Attribute: LyraHealthSet.Health
    Modifier Op: Add
    Modifier Magnitude:
      Magnitude Calculation Type: Scalable Float
      Scalable Float Magnitude: 
        Value: 50.0
        Curve: (可选，用于等级缩放)
```

#### 3.2.2 高级消耗品：持续回复（HoT）

```cpp
// GE_Item_Potion_HealthOverTime.h

/**
 * 持续回复生命药水 Gameplay Effect
 * 
 * 配置：
 * - Duration Policy: Has Duration（持续 10 秒）
 * - Modifiers:
 *   [0] Attribute: Health
 *       Modifier Op: Add
 *       Modifier Magnitude: 5.0（每秒）
 *       Period: 1.0（每 1 秒触发一次）
 */
```

在编辑器中配置：

```
Content/GameplayEffects/Items/GE_Item_Potion_HealthOverTime

Duration Policy: Has Duration
  Duration Magnitude:
    Scalable Float Magnitude: 10.0 (持续 10 秒)

Modifiers:
  [0]
    Attribute: LyraHealthSet.Health
    Modifier Op: Add
    Modifier Magnitude:
      Scalable Float Magnitude: 5.0 (每次恢复 5 点)
    Period: 1.0 (每 1 秒触发一次)

Gameplay Tags:
  Granted Tags: Effect.Potion.Regen (标识正在回复中)
  Asset Tags: Item.Consumable.Potion
```

### 3.3 物品效果应用

#### 3.3.1 创建 Gameplay Ability：使用药水

```cpp
// LyraGameplayAbility_UsePotion.h
#pragma once

#include "CoreMinimal.h"
#include "LyraGameplayAbility.h"
#include "LyraGameplayAbility_UsePotion.generated.h"

/**
 * 使用药水的技能
 * 
 * 流程：
 * 1. 播放使用动画
 * 2. 应用药水效果（Gameplay Effect）
 * 3. 消耗药水
 */
UCLASS()
class LYRAGAME_API ULyraGameplayAbility_UsePotion : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    ULyraGameplayAbility_UsePotion(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

protected:
    //~UGameplayAbility interface
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;
    //~End of UGameplayAbility interface

    /**
     * 使用动画蒙太奇
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Animation")
    TObjectPtr<UAnimMontage> UseMontage;

    /**
     * 应用的 Gameplay Effect
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Effects")
    TArray<TSubclassOf<UGameplayEffect>> EffectsToApply;

    /**
     * 是否消耗物品
     */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Consumable")
    bool bConsumeItem = true;

protected:
    /**
     * 播放使用动画
     */
    UFUNCTION(BlueprintCallable, Category = "Ability")
    void PlayUseMontage();

    /**
     * 应用效果
     */
    UFUNCTION(BlueprintCallable, Category = "Ability")
    void ApplyPotionEffects();

    /**
     * 消耗物品
     */
    UFUNCTION(BlueprintCallable, Category = "Ability")
    void ConsumeItem();

    /**
     * 动画完成回调
     */
    UFUNCTION()
    void OnMontageCompleted(FGameplayTag EventTag, FGameplayEventData EventData);
};
```

```cpp
// LyraGameplayAbility_UsePotion.cpp
#include "LyraGameplayAbility_UsePotion.h"
#include "AbilitySystemComponent.h"
#include "GameplayEffect.h"
#include "LyraInventoryManagerComponent.h"
#include "LyraInventoryItemInstance.h"

ULyraGameplayAbility_UsePotion::ULyraGameplayAbility_UsePotion(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated;
}

void ULyraGameplayAbility_UsePotion::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 播放动画
    PlayUseMontage();

    // 应用效果
    ApplyPotionEffects();

    // 消耗物品
    if (bConsumeItem)
    {
        ConsumeItem();
    }

    // 结束技能
    EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}

void ULyraGameplayAbility_UsePotion::EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled)
{
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}

void ULyraGameplayAbility_UsePotion::PlayUseMontage()
{
    if (UseMontage == nullptr || GetAvatarActorFromActorInfo() == nullptr)
    {
        return;
    }

    // 播放蒙太奇
    UAbilityTask_PlayMontageAndWait* Task = UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
        this,
        NAME_None,
        UseMontage,
        1.0f,
        NAME_None,
        false,
        1.0f
    );

    if (Task != nullptr)
    {
        Task->OnCompleted.AddDynamic(this, &ULyraGameplayAbility_UsePotion::OnMontageCompleted);
        Task->OnInterrupted.AddDynamic(this, &ULyraGameplayAbility_UsePotion::OnMontageCompleted);
        Task->ReadyForActivation();
    }
}

void ULyraGameplayAbility_UsePotion::ApplyPotionEffects()
{
    if (GetAbilitySystemComponentFromActorInfo() == nullptr)
    {
        return;
    }

    UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();

    for (const TSubclassOf<UGameplayEffect>& EffectClass : EffectsToApply)
    {
        if (EffectClass != nullptr)
        {
            FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
            EffectContext.AddSourceObject(this);

            FGameplayEffectSpecHandle EffectSpec = ASC->MakeOutgoingSpec(EffectClass, GetAbilityLevel(), EffectContext);
            if (EffectSpec.IsValid())
            {
                ASC->ApplyGameplayEffectSpecToSelf(*EffectSpec.Data.Get());
                UE_LOG(LogTemp, Log, TEXT("ApplyPotionEffects: Applied %s"), *EffectClass->GetName());
            }
        }
    }
}

void ULyraGameplayAbility_UsePotion::ConsumeItem()
{
    AActor* AvatarActor = GetAvatarActorFromActorInfo();
    if (AvatarActor == nullptr)
    {
        return;
    }

    // 获取背包管理器
    ULyraInventoryManagerComponent* InventoryManager = AvatarActor->FindComponentByClass<ULyraInventoryManagerComponent>();
    if (InventoryManager == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("ConsumeItem: No InventoryManager found!"));
        return;
    }

    // 这里需要知道是哪个物品触发的技能
    // 可以通过 TriggerEventData 传递物品信息
    // 或者在技能授予时保存物品引用

    // 简化示例：查找第一个药水并消耗
    // 实际项目中应该精确定位
    // ULyraInventoryItemInstance* PotionItem = InventoryManager->FindItemByDefinition(...);
    // if (PotionItem != nullptr)
    // {
    //     InventoryManager->RemoveItemInstance(PotionItem, 1);
    // }

    UE_LOG(LogTemp, Log, TEXT("ConsumeItem: Consumed potion"));
}

void ULyraGameplayAbility_UsePotion::OnMontageCompleted(FGameplayTag EventTag, FGameplayEventData EventData)
{
    // 动画播放完毕
    UE_LOG(LogTemp, Log, TEXT("OnMontageCompleted: Potion use animation finished"));
}
```

### 3.4 物品 Cooldown 管理

#### 3.4.1 使用 Gameplay Effect 实现冷却

```cpp
// GE_Item_Cooldown.h

/**
 * 物品冷却 Gameplay Effect
 * 
 * 配置：
 * - Duration Policy: Has Duration
 * - Granted Tags: Cooldown.Item.Potion（用于检测冷却）
 */
```

在编辑器中配置：

```
Content/GameplayEffects/Items/GE_Item_Cooldown

Duration Policy: Has Duration
  Duration Magnitude:
    Set By Caller Magnitude: Cooldown.Duration (可在运行时设置)

Granted Tags:
  Add: Cooldown.Item.Potion

Application Tag Requirements:
  Ignore Tags: Cooldown.Item.Potion (已在冷却中则无法再次应用)
```

#### 3.4.2 在消耗品碎片中应用冷却

```cpp
// LyraInventoryFragment_Consumable.cpp

void ULyraInventoryFragment_Consumable::OnItemUsed(ULyraInventoryItemInstance* ItemInstance, AActor* User) const
{
    if (User == nullptr || ItemInstance == nullptr)
    {
        return;
    }

    // 检查冷却
    if (CooldownTag.IsValid())
    {
        UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(User);
        if (ASC != nullptr && ASC->HasMatchingGameplayTag(CooldownTag))
        {
            UE_LOG(LogTemp, Warning, TEXT("OnItemUsed: Item is on cooldown!"));
            return;
        }
    }

    // 应用效果
    ApplyEffects(User);

    // 应用冷却
    if (CooldownDuration > 0.0f && CooldownTag.IsValid())
    {
        UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(User);
        if (ASC != nullptr)
        {
            // 创建冷却 GE
            FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
            EffectContext.AddSourceObject(ItemInstance);

            // 使用预定义的冷却 GE
            TSubclassOf<UGameplayEffect> CooldownGE = LoadClass<UGameplayEffect>(nullptr, TEXT("/Game/GameplayEffects/Items/GE_Item_Cooldown.GE_Item_Cooldown_C"));
            if (CooldownGE != nullptr)
            {
                FGameplayEffectSpecHandle EffectSpec = ASC->MakeOutgoingSpec(CooldownGE, 1.0f, EffectContext);
                if (EffectSpec.IsValid())
                {
                    // 设置冷却时长
                    EffectSpec.Data->SetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag(TEXT("Cooldown.Duration")), CooldownDuration);

                    // 应用冷却
                    ASC->ApplyGameplayEffectSpecToSelf(*EffectSpec.Data.Get());

                    UE_LOG(LogTemp, Log, TEXT("OnItemUsed: Applied cooldown %f seconds"), CooldownDuration);
                }
            }
        }
    }

    // 消耗物品
    if (bConsumeOnUse)
    {
        ULyraInventoryManagerComponent* InventoryManager = User->FindComponentByClass<ULyraInventoryManagerComponent>();
        if (InventoryManager != nullptr)
        {
            InventoryManager->RemoveItemInstance(ItemInstance, 1);
        }
    }
}
```

#### 3.4.3 UI 显示冷却进度

```cpp
// WBP_InventorySlot.h（扩展）

/**
 * 获取物品冷却进度（0.0 ~ 1.0）
 */
UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
float GetCooldownProgress() const;

/**
 * 是否在冷却中
 */
UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Inventory")
bool IsOnCooldown() const;
```

```cpp
// WBP_InventorySlot.cpp

float UWBP_InventorySlot::GetCooldownProgress() const
{
    if (ItemInstance == nullptr)
    {
        return 0.0f;
    }

    // 获取冷却碎片
    const ULyraInventoryFragment_Consumable* ConsumableFragment = ItemInstance->FindFragmentByClass<ULyraInventoryFragment_Consumable>();
    if (ConsumableFragment == nullptr || !ConsumableFragment->CooldownTag.IsValid())
    {
        return 0.0f;
    }

    // 获取所有者的 ASC
    AActor* Owner = ItemInstance->GetOwningActor();
    if (Owner == nullptr)
    {
        return 0.0f;
    }

    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Owner);
    if (ASC == nullptr)
    {
        return 0.0f;
    }

    // 查找冷却 GE
    FGameplayTagContainer CooldownTags;
    CooldownTags.AddTag(ConsumableFragment->CooldownTag);

    FGameplayEffectQuery Query;
    Query.OwningTagQuery = FGameplayTagQuery::MakeQuery_MatchAnyTags(CooldownTags);

    TArray<FActiveGameplayEffectHandle> ActiveEffects = ASC->GetActiveEffects(Query);
    if (ActiveEffects.Num() > 0)
    {
        // 获取第一个冷却 GE 的剩余时间
        const FActiveGameplayEffect* ActiveGE = ASC->GetActiveGameplayEffect(ActiveEffects[0]);
        if (ActiveGE != nullptr)
        {
            float Duration = ActiveGE->GetDuration();
            float TimeRemaining = ActiveGE->GetTimeRemaining(ASC->GetWorld()->GetTimeSeconds());

            if (Duration > 0.0f)
            {
                return FMath::Clamp(1.0f - (TimeRemaining / Duration), 0.0f, 1.0f);
            }
        }
    }

    return 0.0f;
}

bool UWBP_InventorySlot::IsOnCooldown() const
{
    return GetCooldownProgress() > 0.0f && GetCooldownProgress() < 1.0f;
}
```

在 Widget 蓝图中：

```
WBP_InventorySlot:
  - Image_Cooldown (覆盖在图标上的半透明遮罩)
    - Render Opacity: Bind to GetCooldownProgress()
    - Visibility: Bind to (IsOnCooldown ? Visible : Collapsed)

  - Text_CooldownTime (显示剩余冷却时间)
    - Text: Bind to (CooldownRemaining formatted as "3.2s")
    - Visibility: Bind to (IsOnCooldown ? Visible : Collapsed)
```

---

## 第四部分：网络同步（15%）

网络同步是多人游戏的关键，确保所有玩家看到一致的背包状态。

### 4.1 背包数据的网络复制

Lyra 使用 **Fast Array Serialization** 实现高效的网络复制。

#### 4.1.1 Fast Array 原理

Fast Array 是 UE 的优化网络复制容器，特点：

1. **增量复制**：只复制变化的元素
2. **回调机制**：提供 `PreReplicatedRemove`/`PostReplicatedAdd`/`PostReplicatedChange` 回调
3. **高效**：比普通数组复制更节省带宽

#### 4.1.2 Fast Array 实现（已在第一部分实现）

```cpp
// FLyraInventoryList 已经实现了 Fast Array

USTRUCT(BlueprintType)
struct FLyraInventoryList : public FFastArraySerializer
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<FLyraInventoryEntry> Entries;

    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
    {
        return FFastArraySerializer::FastArrayDeltaSerialize<FLyraInventoryEntry, FLyraInventoryList>(Entries, DeltaParms, *this);
    }
};

template<>
struct TStructOpsTypeTraits<FLyraInventoryList> : public TStructOpsTypeTraitsBase2<FLyraInventoryList>
{
    enum
    {
        WithNetDeltaSerializer = true,
    };
};
```

### 4.2 物品添加/移除同步

#### 4.2.1 服务器添加物品

```cpp
// 服务器执行
ULyraInventoryItemInstance* ULyraInventoryManagerComponent::AddItemByDefinition(...)
{
    // 权限检查
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("AddItemByDefinition: Not called on server!"));
        return nullptr;
    }

    // 添加物品逻辑...

    // Fast Array 自动标记为 Dirty
    InventoryList.MarkArrayDirty();

    // 复制到客户端后，客户端会触发 PostReplicatedAdd 回调
}
```

#### 4.2.2 客户端接收添加

```cpp
// FLyraInventoryEntry.cpp

void FLyraInventoryEntry::PostReplicatedAdd(const FLyraInventoryList& InArraySerializer)
{
    // 客户端接收到新物品

    if (ItemInstance != nullptr && InArraySerializer.OwnerComponent != nullptr)
    {
        AActor* OwningActor = InArraySerializer.OwnerComponent->GetOwner();
        ItemInstance->SetOwningActor(OwningActor);
        ItemInstance->OnItemAdded(OwningActor);

        // 触发事件（更新 UI）
        InArraySerializer.OwnerComponent->OnItemAdded.Broadcast(ItemInstance, SlotIndex);
    }

    UE_LOG(LogTemp, Log, TEXT("PostReplicatedAdd: Received new item on client"));
}
```

### 4.3 客户端预测与服务端验证

为了提升用户体验，可以在客户端先"预测"操作结果，然后等待服务器验证。

#### 4.3.1 客户端预测添加物品

```cpp
/**
 * 客户端预测添加物品
 * 
 * 流程：
 * 1. 客户端：立即更新 UI（预测）
 * 2. 客户端：向服务器发送 RPC
 * 3. 服务器：验证并执行
 * 4. 服务器：复制结果到客户端
 * 5. 客户端：对比预测结果，如果不一致则回滚
 */

// LyraInventoryManagerComponent.h

/**
 * 客户端请求添加物品（预测）
 */
UFUNCTION(Client, Reliable)
void Client_PredictAddItem(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 Count);

/**
 * 服务器验证并添加物品
 */
UFUNCTION(Server, Reliable, WithValidation)
void Server_RequestAddItem(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 Count);
```

```cpp
// LyraInventoryManagerComponent.cpp

void ULyraInventoryManagerComponent::Client_PredictAddItem_Implementation(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 Count)
{
    // 客户端预测（不实际修改数据，只更新 UI）
    UE_LOG(LogTemp, Log, TEXT("Client_PredictAddItem: Predicting add %s x%d"), *ItemDef->GetName(), Count);

    // 创建临时的"预测物品"用于 UI 显示
    // 实际项目中需要更复杂的预测系统

    // 发送 RPC 到服务器
    Server_RequestAddItem(ItemDef, Count);
}

bool ULyraInventoryManagerComponent::Server_RequestAddItem_Validate(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 Count)
{
    // 验证参数合法性（防止作弊）
    if (ItemDef == nullptr || Count <= 0 || Count > 9999)
    {
        UE_LOG(LogTemp, Warning, TEXT("Server_RequestAddItem_Validate: Invalid parameters!"));
        return false;
    }

    return true;
}

void ULyraInventoryManagerComponent::Server_RequestAddItem_Implementation(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 Count)
{
    // 服务器执行真正的添加逻辑
    int32 RemainingCount = 0;
    ULyraInventoryItemInstance* AddedItem = AddItemByDefinition(ItemDef, Count, RemainingCount);

    if (AddedItem != nullptr)
    {
        UE_LOG(LogTemp, Log, TEXT("Server_RequestAddItem: Added %s x%d"), *ItemDef->GetName(), Count - RemainingCount);
    }
    else
    {
        UE_LOG(LogTemp, Warning, TEXT("Server_RequestAddItem: Failed to add %s"), *ItemDef->GetName());
    }

    // Fast Array 会自动复制到客户端
}
```

#### 4.3.2 服务端验证

服务器需要验证客户端的所有操作，防止作弊。

```cpp
/**
 * 服务器验证：移除物品
 */
bool ULyraInventoryManagerComponent::Server_RequestRemoveItem_Validate(ULyraInventoryItemInstance* ItemInstance, int32 Count)
{
    // 验证物品是否存在
    if (ItemInstance == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("Server_RequestRemoveItem_Validate: Invalid item!"));
        return false;
    }

    // 验证物品是否属于该玩家
    FLyraInventoryEntry* Entry = InventoryList.FindEntry(ItemInstance);
    if (Entry == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("Server_RequestRemoveItem_Validate: Item not in inventory!"));
        return false;
    }

    // 验证数量
    if (Count <= 0 || Count > ItemInstance->GetStackCount())
    {
        UE_LOG(LogTemp, Warning, TEXT("Server_RequestRemoveItem_Validate: Invalid count %d (have %d)"), Count, ItemInstance->GetStackCount());
        return false;
    }

    return true;
}

void ULyraInventoryManagerComponent::Server_RequestRemoveItem_Implementation(ULyraInventoryItemInstance* ItemInstance, int32 Count)
{
    // 验证通过，执行移除
    int32 RemovedCount = RemoveItemInstance(ItemInstance, Count);

    UE_LOG(LogTemp, Log, TEXT("Server_RequestRemoveItem: Removed %d items"), RemovedCount);
}
```

### 4.4 网络同步最佳实践

#### 4.4.1 减少网络带宽

1. **只复制必要的数据**

```cpp
// LyraInventoryItemInstance.h

// ❌ 不要复制所有数据
UPROPERTY(Replicated)
FString DebugInfo; // 客户端不需要

// ✅ 只复制必要的
UPROPERTY(Replicated)
int32 StackCount;

UPROPERTY(Replicated)
int32 ItemID;

// ❌ 不要复制 ItemDef 的所有内容
// ItemDef 是 Data Asset，客户端已经有了

// ✅ 只复制 ItemDef 的引用
UPROPERTY(Replicated)
TObjectPtr<const ULyraInventoryItemDefinition> ItemDef;
```

2. **使用条件复制**

```cpp
// 只对所有者复制
UPROPERTY(Replicated, ReplicatedUsing = OnRep_PrivateData)
FPrivateInventoryData PrivateData;

void ULyraInventoryManagerComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ULyraInventoryManagerComponent, InventoryList);

    // 只对所有者复制
    DOREPLIFETIME_CONDITION(ULyraInventoryManagerComponent, PrivateData, COND_OwnerOnly);
}
```

3. **使用 Fast Array**

```cpp
// ✅ Fast Array 增量复制
UPROPERTY(Replicated)
FLyraInventoryList InventoryList;

// ❌ 不要使用普通数组（全量复制）
UPROPERTY(Replicated)
TArray<ULyraInventoryItemInstance*> Items; // 每次都复制整个数组
```

#### 4.4.2 处理网络延迟

1. **客户端预测**

```cpp
// 立即更新 UI（预测）
void ULyraInventoryManagerComponent::Client_PredictUseItem(ULyraInventoryItemInstance* ItemInstance)
{
    // 立即播放使用动画
    // 立即更新 UI（显示冷却）

    // 发送 RPC 到服务器
    Server_RequestUseItem(ItemInstance);

    // 等待服务器验证
    // 如果验证失败，回滚预测
}
```

2. **延迟补偿**

```cpp
// 服务器记录操作时间戳，客户端根据延迟调整
UPROPERTY(Replicated)
float LastOperationTimestamp;
```

#### 4.4.3 防止作弊

1. **服务器权威**

```cpp
// ✅ 所有修改操作都在服务器执行
UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
ULyraInventoryItemInstance* AddItemByDefinition(...);

// ❌ 不要在客户端修改数据
void Client_AddItem() // 不安全！
{
    InventoryList.Entries.Add(...); // 作弊漏洞！
}
```

2. **服务端验证**

```cpp
bool ULyraInventoryManagerComponent::Server_RequestUseItem_Validate(ULyraInventoryItemInstance* ItemInstance)
{
    // 验证物品存在
    if (ItemInstance == nullptr)
        return false;

    // 验证物品属于该玩家
    if (!InventoryList.FindEntry(ItemInstance))
        return false;

    // 验证冷却
    const ULyraInventoryFragment_Consumable* Consumable = ItemInstance->FindFragmentByClass<ULyraInventoryFragment_Consumable>();
    if (Consumable && !Consumable->CanUse(GetOwner()))
        return false;

    return true;
}
```

3. **限制操作频率**

```cpp
// 防止客户端频繁发送 RPC（DDoS）
float LastOperationTime = 0.0f;
const float MinOperationInterval = 0.1f; // 最小间隔 0.1 秒

bool ULyraInventoryManagerComponent::Server_RequestOperation_Validate()
{
    float CurrentTime = GetWorld()->GetTimeSeconds();
    if (CurrentTime - LastOperationTime < MinOperationInterval)
    {
        UE_LOG(LogTemp, Warning, TEXT("Server_RequestOperation_Validate: Too frequent!"));
        return false; // 拒绝操作
    }

    LastOperationTime = CurrentTime;
    return true;
}
```

---

## 第五部分：实战案例（10%）

现在我们将所有知识点整合，实现一个完整的 RPG 背包系统。

### 5.1 完整的 RPG 背包系统

#### 5.1.1 项目需求

- **背包容量**：40 格（8x5）
- **物品类型**：武器、防具、消耗品、材料、任务物品
- **物品品质**：普通、优秀、稀有、史诗、传说、神话
- **物品堆叠**：消耗品和材料可堆叠（最多 99 个）
- **装备系统**：可装备武器和防具
- **物品使用**：可使用消耗品
- **物品交易**：可丢弃、出售物品
- **UI 功能**：拖拽、拆分、合并、排序

#### 5.1.2 创建物品定义

**1. 武器：铁剑**

```cpp
// Content/Items/Weapons/ID_Weapon_IronSword.uasset

Display Name: "铁剑"
Description: "一把普通的铁制长剑，适合新手冒险者使用。"
Max Stack Count: 1
bCan Drop: true
bCan Sell: true
Sell Price: 50

Fragments:
  [0] Stats Fragment
    Stats:
      [0] StatTag: Stat.Damage, Value: 25.0, Description: "攻击力"
      [1] StatTag: Stat.AttackSpeed, Value: 1.2, Description: "攻击速度"
      [2] StatTag: Stat.Weight, Value: 5.0, Description: "重量"

  [1] Equipment Fragment
    Equipment Slot: MainHand
    Equipment Definition: ED_Weapon_Sword
    Ability Sets To Grant: [AS_Weapon_Sword_Basic]
    bCan Equip In Combat: true
    Required Level: 1

  [2] Visual Fragment
    Item Icon: T_Icon_Sword_Iron
    Rarity: Common
    World Mesh: SM_Sword_Iron
    Skeletal Mesh: SKM_Sword_Iron

  [3] Tags Fragment
    Item Tags:
      - Item.Type.Weapon
      - Item.Type.Weapon.Melee
      - Item.Tag.Tradable
      - Item.Tag.Sellable
```

**2. 消耗品：生命药水**

```cpp
// Content/Items/Consumables/ID_Potion_Health.uasset

Display Name: "生命药水"
Description: "恢复 50 点生命值。"
Max Stack Count: 99
bCan Drop: true
bCan Sell: true
Sell Price: 10

Fragments:
  [0] Stats Fragment
    Stats:
      [0] StatTag: Stat.HealAmount, Value: 50.0, Description: "恢复量"

  [1] Consumable Fragment
    Gameplay Effects To Apply: [GE_Item_Potion_Health]
    Ability To Activate: None
    bConsume On Use: true
    Cooldown Duration: 1.0
    Cooldown Tag: Cooldown.Item.Potion
    bCan Use In Combat: true
    Use Montage: AM_DrinkPotion

  [2] Visual Fragment
    Item Icon: T_Icon_Potion_Health
    Rarity: Common

  [3] Tags Fragment
    Item Tags:
      - Item.Type.Consumable
      - Item.Tag.Stackable
      - Item.Tag.Tradable
      - Item.Tag.Sellable
```

**3. 材料：铁矿石**

```cpp
// Content/Items/Materials/ID_Material_IronOre.uasset

Display Name: "铁矿石"
Description: "用于锻造铁制装备的基础材料。"
Max Stack Count: 99
bCan Drop: true
bCan Sell: true
Sell Price: 5

Fragments:
  [0] Visual Fragment
    Item Icon: T_Icon_Ore_Iron
    Rarity: Common

  [1] Tags Fragment
    Item Tags:
      - Item.Type.Material
      - Item.Tag.Stackable
      - Item.Tag.Tradable
      - Item.Tag.Sellable
```

### 5.2 物品拾取与丢弃

#### 5.2.1 创建掉落物 Actor

```cpp
// LyraItemPickup.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "LyraItemPickup.generated.h"

class ULyraInventoryItemDefinition;
class USphereComponent;
class UStaticMeshComponent;

/**
 * 物品掉落物 Actor
 * 
 * 玩家可以拾取该 Actor 来获得物品
 */
UCLASS()
class LYRAGAME_API ALyraItemPickup : public AActor
{
    GENERATED_BODY()

public:
    ALyraItemPickup();

protected:
    virtual void BeginPlay() override;

public:
    /**
     * 初始化掉落物
     */
    UFUNCTION(BlueprintCallable, Category = "Pickup")
    void InitializePickup(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 Count);

    /**
     * 拾取物品
     */
    UFUNCTION(BlueprintCallable, Category = "Pickup")
    void PickupItem(AActor* Picker);

protected:
    /**
     * 碰撞组件
     */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Pickup")
    TObjectPtr<USphereComponent> CollisionComponent;

    /**
     * 网格组件
     */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Pickup")
    TObjectPtr<UStaticMeshComponent> MeshComponent;

    /**
     * 物品定义
     */
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Pickup")
    TSubclassOf<ULyraInventoryItemDefinition> ItemDefinition;

    /**
     * 物品数量
     */
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Pickup")
    int32 ItemCount = 1;

    /**
     * 是否可以拾取
     */
    UPROPERTY(BlueprintReadOnly, Category = "Pickup")
    bool bCanPickup = true;

    /**
     * 拾取提示文本
     */
    UPROPERTY(BlueprintReadOnly, Category = "Pickup")
    FText PickupPromptText;

protected:
    /**
     * 重叠开始（显示拾取提示）
     */
    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

    /**
     * 重叠结束（隐藏拾取提示）
     */
    UFUNCTION()
    void OnOverlapEnd(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);

    /**
     * 更新显示
     */
    void UpdateVisuals();
};
```

```cpp
// LyraItemPickup.cpp
#include "LyraItemPickup.h"
#include "LyraInventoryItemDefinition.h"
#include "LyraInventoryFragment_Visual.h"
#include "LyraInventoryManagerComponent.h"
#include "Components/SphereComponent.h"
#include "Components/StaticMeshComponent.h"
#include "Net/UnrealNetwork.h"

ALyraItemPickup::ALyraItemPickup()
{
    // 设置复制
    bReplicates = true;
    bAlwaysRelevant = true;

    // 创建碰撞组件
    CollisionComponent = CreateDefaultSubobject<USphereComponent>(TEXT("CollisionComponent"));
    CollisionComponent->InitSphereRadius(100.0f);
    CollisionComponent->SetCollisionProfileName(TEXT("OverlapAllDynamic"));
    RootComponent = CollisionComponent;

    // 创建网格组件
    MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("MeshComponent"));
    MeshComponent->SetupAttachment(RootComponent);
    MeshComponent->SetCollisionEnabled(ECollisionEnabled::NoCollision);

    // 绑定重叠事件
    CollisionComponent->OnComponentBeginOverlap.AddDynamic(this, &ALyraItemPickup::OnOverlapBegin);
    CollisionComponent->OnComponentEndOverlap.AddDynamic(this, &ALyraItemPickup::OnOverlapEnd);
}

void ALyraItemPickup::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ALyraItemPickup, ItemDefinition);
    DOREPLIFETIME(ALyraItemPickup, ItemCount);
}

void ALyraItemPickup::BeginPlay()
{
    Super::BeginPlay();

    UpdateVisuals();
}

void ALyraItemPickup::InitializePickup(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 Count)
{
    ItemDefinition = ItemDef;
    ItemCount = Count;

    UpdateVisuals();

    // 更新拾取提示文本
    if (ItemDefinition != nullptr)
    {
        const ULyraInventoryItemDefinition* ItemDefCDO = GetDefault<ULyraInventoryItemDefinition>(ItemDefinition);
        if (ItemDefCDO != nullptr)
        {
            PickupPromptText = FText::Format(
                FText::FromString(TEXT("按 E 拾取 {0} x{1}")),
                ItemDefCDO->DisplayName,
                FText::AsNumber(ItemCount)
            );
        }
    }
}

void ALyraItemPickup::PickupItem(AActor* Picker)
{
    if (!HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("PickupItem: Not called on server!"));
        return;
    }

    if (!bCanPickup || Picker == nullptr || ItemDefinition == nullptr)
    {
        return;
    }

    // 获取背包管理器
    ULyraInventoryManagerComponent* InventoryManager = Picker->FindComponentByClass<ULyraInventoryManagerComponent>();
    if (InventoryManager == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("PickupItem: Picker has no InventoryManager!"));
        return;
    }

    // 添加物品到背包
    int32 RemainingCount = 0;
    ULyraInventoryItemInstance* AddedItem = InventoryManager->AddItemByDefinition(ItemDefinition, ItemCount, RemainingCount);

    if (RemainingCount == 0)
    {
        // 完全拾取
        UE_LOG(LogTemp, Log, TEXT("PickupItem: Picked up %s x%d"), *ItemDefinition->GetName(), ItemCount);
        Destroy();
    }
    else
    {
        // 部分拾取（背包满）
        ItemCount = RemainingCount;
        UE_LOG(LogTemp, Warning, TEXT("PickupItem: Inventory full! %d items remaining"), RemainingCount);

        // 更新提示文本
        InitializePickup(ItemDefinition, ItemCount);
    }
}

void ALyraItemPickup::UpdateVisuals()
{
    if (ItemDefinition == nullptr)
    {
        return;
    }

    const ULyraInventoryItemDefinition* ItemDefCDO = GetDefault<ULyraInventoryItemDefinition>(ItemDefinition);
    if (ItemDefCDO == nullptr)
    {
        return;
    }

    // 获取视觉碎片
    const ULyraInventoryFragment_Visual* VisualFragment = ItemDefCDO->FindFragmentByClass<ULyraInventoryFragment_Visual>();
    if (VisualFragment != nullptr && VisualFragment->WorldMesh != nullptr)
    {
        MeshComponent->SetStaticMesh(VisualFragment->WorldMesh);

        // 应用材质
        if (VisualFragment->ItemMaterial != nullptr)
        {
            MeshComponent->SetMaterial(0, VisualFragment->ItemMaterial);
        }

        // TODO: 添加粒子特效、发光效果等
    }
}

void ALyraItemPickup::OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
    // 显示拾取提示
    // 这里应该调用 UI 系统显示提示
    // 例如：ShowPickupPrompt(PickupPromptText);

    UE_LOG(LogTemp, Log, TEXT("OnOverlapBegin: %s entered pickup range"), *OtherActor->GetName());
}

void ALyraItemPickup::OnOverlapEnd(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
{
    // 隐藏拾取提示
    // 例如：HidePickupPrompt();

    UE_LOG(LogTemp, Log, TEXT("OnOverlapEnd: %s left pickup range"), *OtherActor->GetName());
}
```

#### 5.2.2 拾取输入处理

```cpp
// LyraCharacter.h

/**
 * 尝试拾取物品
 */
UFUNCTION(BlueprintCallable, Category = "Inventory")
void TryPickupItem();

/**
 * 当前可拾取的物品
 */
UPROPERTY(BlueprintReadOnly, Category = "Inventory")
TObjectPtr<ALyraItemPickup> CurrentPickupTarget;
```

```cpp
// LyraCharacter.cpp

void ALyraCharacter::TryPickupItem()
{
    if (CurrentPickupTarget != nullptr)
    {
        CurrentPickupTarget->PickupItem(this);
    }
}

// 在 Enhanced Input 中绑定拾取按键
// Input Action: IA_Pickup (E 键)
// Triggered -> TryPickupItem
```

#### 5.2.3 丢弃物品

```cpp
// LyraInventoryManagerComponent.h

/**
 * 丢弃物品
 */
UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
void DropItem(ULyraInventoryItemInstance* ItemInstance, int32 Count = -1);
```

```cpp
// LyraInventoryManagerComponent.cpp

void ULyraInventoryManagerComponent::DropItem(ULyraInventoryItemInstance* ItemInstance, int32 Count)
{
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("DropItem: Not called on server!"));
        return;
    }

    if (ItemInstance == nullptr)
    {
        return;
    }

    // 检查是否可以丢弃
    const ULyraInventoryItemDefinition* ItemDef = ItemInstance->GetItemDef();
    if (ItemDef != nullptr && !ItemDef->bCanDrop)
    {
        UE_LOG(LogTemp, Warning, TEXT("DropItem: Item cannot be dropped!"));
        return;
    }

    // 计算丢弃数量
    int32 DropCount = (Count == -1) ? ItemInstance->GetStackCount() : FMath::Min(Count, ItemInstance->GetStackCount());

    // 在角色前方生成掉落物
    AActor* Owner = GetOwner();
    if (Owner != nullptr)
    {
        FVector SpawnLocation = Owner->GetActorLocation() + Owner->GetActorForwardVector() * 100.0f;
        FRotator SpawnRotation = Owner->GetActorRotation();

        FActorSpawnParameters SpawnParams;
        SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

        ALyraItemPickup* DroppedItem = GetWorld()->SpawnActor<ALyraItemPickup>(ALyraItemPickup::StaticClass(), SpawnLocation, SpawnRotation, SpawnParams);
        if (DroppedItem != nullptr)
        {
            DroppedItem->InitializePickup(ItemDef->GetClass(), DropCount);

            // 给掉落物一个初速度（抛出效果）
            if (UStaticMeshComponent* MeshComp = DroppedItem->FindComponentByClass<UStaticMeshComponent>())
            {
                MeshComp->SetSimulatePhysics(true);
                MeshComp->AddImpulse(Owner->GetActorForwardVector() * 500.0f + FVector(0, 0, 300.0f), NAME_None, true);
            }

            UE_LOG(LogTemp, Log, TEXT("DropItem: Dropped %s x%d"), *ItemDef->GetName(), DropCount);
        }
    }

    // 从背包移除
    RemoveItemInstance(ItemInstance, DropCount);
}
```

### 5.3 物品使用系统

#### 5.3.1 使用物品的输入处理

```cpp
// WBP_InventorySlot.h

/**
 * 右键点击槽位（使用物品）
 */
UFUNCTION(BlueprintCallable, Category = "Inventory")
void OnRightClick();
```

```cpp
// WBP_InventorySlot.cpp

void UWBP_InventorySlot::OnRightClick()
{
    if (ItemInstance == nullptr)
    {
        return;
    }

    // 检查物品是否可以使用
    const ULyraInventoryFragment_Consumable* ConsumableFragment = ItemInstance->FindFragmentByClass<ULyraInventoryFragment_Consumable>();
    const ULyraInventoryFragment_Equipment* EquipmentFragment = ItemInstance->FindFragmentByClass<ULyraInventoryFragment_Equipment>();

    if (ConsumableFragment != nullptr)
    {
        // 使用消耗品
        AActor* Owner = ItemInstance->GetOwningActor();
        if (Owner != nullptr)
        {
            ItemInstance->OnItemUsed(Owner);
        }
    }
    else if (EquipmentFragment != nullptr)
    {
        // 装备物品
        AActor* Owner = ItemInstance->GetOwningActor();
        if (Owner != nullptr)
        {
            // 调用装备管理器装备物品
            // ULyraEquipmentManagerComponent* EquipmentManager = Owner->FindComponentByClass<ULyraEquipmentManagerComponent>();
            // if (EquipmentManager != nullptr)
            // {
            //     EquipmentManager->EquipItemFromInventory(ItemInstance, EquipmentFragment->EquipmentSlot);
            // }
        }
    }
    else
    {
        UE_LOG(LogTemp, Warning, TEXT("OnRightClick: Item cannot be used"));
    }
}
```

### 5.4 背包 UI 实现

#### 5.4.1 完整的背包界面蓝图

```
WBP_InventoryPanel (UserWidget)
├── Canvas Panel
│   ├── Background (Image) - 背景图
│   ├── Title (TextBlock) - "背包"
│   ├── CloseButton (Button) - 关闭按钮
│   ├── GridPanel_Slots (UniformGridPanel) - 槽位网格
│   │   └── [动态创建 WBP_InventorySlot ×40]
│   ├── ItemInfo (Widget) - 物品详情面板
│   │   ├── ItemName (TextBlock)
│   │   ├── ItemDescription (TextBlock)
│   │   ├── ItemStats (VerticalBox)
│   │   └── ItemActions (HorizontalBox)
│   │       ├── UseButton (Button)
│   │       ├── EquipButton (Button)
│   │       ├── DropButton (Button)
│   │       └── SplitButton (Button)
│   └── SortButtons (HorizontalBox) - 排序按钮
│       ├── SortByName (Button)
│       ├── SortByType (Button)
│       └── SortByRarity (Button)
└── [事件绑定]
    ├── OnSlotLeftClick -> ShowItemInfo
    ├── OnSlotRightClick -> UseItem
    ├── OnSlotDragStart -> BeginDragItem
    ├── OnSlotDragEnd -> EndDragItem
    └── OnSortButtonClick -> SortInventory
```

#### 5.4.2 拖拽功能实现

```cpp
// WBP_InventorySlot.h

/**
 * 开始拖拽
 */
UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Inventory")
void OnDragDetected(const FGeometry& MyGeometry, const FPointerEvent& MouseEvent, UDragDropOperation*& Operation);
```

```cpp
// WBP_InventorySlot.cpp

void UWBP_InventorySlot::OnDragDetected_Implementation(const FGeometry& MyGeometry, const FPointerEvent& MouseEvent, UDragDropOperation*& Operation)
{
    if (ItemInstance == nullptr)
    {
        return;
    }

    // 创建拖拽操作
    UDragDropOperation* DragOp = NewObject<UDragDropOperation>();
    if (DragOp != nullptr)
    {
        // 设置拖拽数据
        DragOp->Payload = ItemInstance;
        DragOp->DefaultDragVisual = this; // 使用当前 Widget 作为拖拽视觉

        // 创建拖拽视觉 Widget
        UWBP_InventorySlot* DragWidget = CreateWidget<UWBP_InventorySlot>(this, GetClass());
        if (DragWidget != nullptr)
        {
            DragWidget->SetItem(ItemInstance);
            DragOp->DefaultDragVisual = DragWidget;
        }

        Operation = DragOp;

        UE_LOG(LogTemp, Log, TEXT("OnDragDetected: Started dragging %s"), *ItemInstance->GetItemDef()->GetName());
    }
}
```

```cpp
// WBP_InventorySlot.h

/**
 * 接受放置
 */
UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Inventory")
bool OnDrop(const FGeometry& MyGeometry, const FPointerEvent& MouseEvent, UDragDropOperation* Operation);
```

```cpp
// WBP_InventorySlot.cpp

bool UWBP_InventorySlot::OnDrop_Implementation(const FGeometry& MyGeometry, const FPointerEvent& MouseEvent, UDragDropOperation* Operation)
{
    if (Operation == nullptr || Operation->Payload == nullptr)
    {
        return false;
    }

    ULyraInventoryItemInstance* DraggedItem = Cast<ULyraInventoryItemInstance>(Operation->Payload);
    if (DraggedItem == nullptr)
    {
        return false;
    }

    // 获取背包管理器
    AActor* Owner = DraggedItem->GetOwningActor();
    if (Owner == nullptr)
    {
        return false;
    }

    ULyraInventoryManagerComponent* InventoryManager = Owner->FindComponentByClass<ULyraInventoryManagerComponent>();
    if (InventoryManager == nullptr)
    {
        return false;
    }

    // 如果目标槽位为空，移动物品
    if (ItemInstance == nullptr)
    {
        InventoryManager->MoveItemToSlot(DraggedItem, SlotIndex);
        UE_LOG(LogTemp, Log, TEXT("OnDrop: Moved item to slot %d"), SlotIndex);
        return true;
    }

    // 如果目标槽位有物品
    if (DraggedItem->CanStackWith(ItemInstance))
    {
        // 可以堆叠，合并
        InventoryManager->MergeItemStacks(DraggedItem, ItemInstance);
        UE_LOG(LogTemp, Log, TEXT("OnDrop: Merged stacks"));
        return true;
    }
    else
    {
        // 不能堆叠，交换
        // 获取源槽位索引
        int32 SourceSlotIndex = -1;
        // TODO: 从 InventoryManager 获取 DraggedItem 的槽位索引

        if (SourceSlotIndex != -1)
        {
            InventoryManager->SwapItemSlots(SourceSlotIndex, SlotIndex);
            UE_LOG(LogTemp, Log, TEXT("OnDrop: Swapped slots %d and %d"), SourceSlotIndex, SlotIndex);
            return true;
        }
    }

    return false;
}
```

#### 5.4.3 背包排序

```cpp
// LyraInventoryManagerComponent.h

/**
 * 排序背包
 */
UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Inventory")
void SortInventory(EInventorySortMode SortMode);

/**
 * 排序模式
 */
UENUM(BlueprintType)
enum class EInventorySortMode : uint8
{
    ByName          UMETA(DisplayName = "按名称"),
    ByType          UMETA(DisplayName = "按类型"),
    ByRarity        UMETA(DisplayName = "按品质"),
    ByStackCount    UMETA(DisplayName = "按数量"),
};
```

```cpp
// LyraInventoryManagerComponent.cpp

void ULyraInventoryManagerComponent::SortInventory(EInventorySortMode SortMode)
{
    if (!GetOwner()->HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("SortInventory: Not called on server!"));
        return;
    }

    // 获取所有物品
    TArray<ULyraInventoryItemInstance*> Items = GetAllItems();

    // 按照指定模式排序
    switch (SortMode)
    {
    case EInventorySortMode::ByName:
        Items.Sort([](const ULyraInventoryItemInstance& A, const ULyraInventoryItemInstance& B) {
            return A.GetItemDef()->DisplayName.ToString() < B.GetItemDef()->DisplayName.ToString();
        });
        break;

    case EInventorySortMode::ByType:
        // 按照类型标签排序
        Items.Sort([](const ULyraInventoryItemInstance& A, const ULyraInventoryItemInstance& B) {
            // 获取类型标签
            FGameplayTagContainer TagsA = A.GetItemTags();
            FGameplayTagContainer TagsB = B.GetItemTags();
            // 简化：比较第一个标签
            return TagsA.First().ToString() < TagsB.First().ToString();
        });
        break;

    case EInventorySortMode::ByRarity:
        Items.Sort([](const ULyraInventoryItemInstance& A, const ULyraInventoryItemInstance& B) {
            const ULyraInventoryFragment_Visual* VisualA = A.FindFragmentByClass<ULyraInventoryFragment_Visual>();
            const ULyraInventoryFragment_Visual* VisualB = B.FindFragmentByClass<ULyraInventoryFragment_Visual>();
            if (VisualA && VisualB)
            {
                return (int32)VisualA->Rarity > (int32)VisualB->Rarity; // 稀有度高的排前面
            }
            return false;
        });
        break;

    case EInventorySortMode::ByStackCount:
        Items.Sort([](const ULyraInventoryItemInstance& A, const ULyraInventoryItemInstance& B) {
            return A.GetStackCount() > B.GetStackCount(); // 数量多的排前面
        });
        break;
    }

    // 重新分配槽位
    for (int32 i = 0; i < Items.Num(); ++i)
    {
        FLyraInventoryEntry* Entry = InventoryList.FindEntry(Items[i]);
        if (Entry != nullptr)
        {
            int32 OldSlot = Entry->SlotIndex;
            Entry->SlotIndex = i;
            InventoryList.MarkItemDirty(*Entry);

            // 触发事件
            OnItemMoved.Broadcast(Items[i], OldSlot, i);
        }
    }

    UE_LOG(LogTemp, Log, TEXT("SortInventory: Sorted by mode %d"), (int32)SortMode);
}
```

---

## 总结

恭喜！你已经完成了 Lyra 教程系列第 11 篇：《背包与物品系统实现》的学习。

### 本文知识点回顾

1. **背包系统架构**
   - 碎片化设计理念
   - Item Definition、Item Instance、Item Fragment
   - Inventory Component 核心组件
   - 槽位管理（格子系统 vs 列表系统）
   - 物品堆叠与拆分

2. **物品系统详解**
   - Item Fragment 碎片化设计
   - 属性碎片、装备碎片、消耗品碎片、技能碎片、视觉碎片
   - 物品品质与稀有度
   - 物品标签与分类
   - 物品图标与 UI 集成

3. **与 GAS 集成**
   - 物品授予 Ability
   - 可消耗物品（药水、食物）
   - 物品效果应用（Gameplay Effects）
   - 物品 Cooldown 管理

4. **网络同步**
   - Fast Array 高效复制
   - 物品添加/移除同步
   - 客户端预测与服务端验证
   - 作弊防护

5. **实战案例**
   - 完整的 RPG 背包系统
   - 物品拾取与丢弃
   - 物品使用系统
   - 背包 UI 实现（拖拽、排序）

### 关键设计模式

1. **碎片化设计（Fragment Pattern）**
   - 将物品特性拆分为独立的 Fragment
   - 按需组合，避免臃肿的 Struct
   - 易于扩展，符合开闭原则

2. **事件驱动（Event-Driven）**
   - 使用委托（Delegate）通知 UI 和其他系统
   - 解耦背包逻辑和 UI 显示
   - 支持多个监听者

3. **数据与逻辑分离（Data-Logic Separation）**
   - Definition 存储静态配置（Data Asset）
   - Instance 存储动态状态（UObject）
   - Fragment 封装特定逻辑

4. **服务器权威（Server Authority）**
   - 所有修改操作在服务器执行
   - 客户端通过 RPC 请求操作
   - 服务器验证并同步结果

### 性能优化建议

1. **网络带宽优化**
   - 使用 Fast Array 增量复制
   - 只复制必要的数据
   - 条件复制（OwnerOnly）

2. **内存优化**
   - Fragment 按需组合，避免冗余字段
   - 使用 Data Asset 共享静态数据
   - 及时清理未使用的物品实例

3. **UI 性能**
   - 使用对象池复用槽位 Widget
   - 只刷新变化的槽位
   - 延迟加载物品图标

### 扩展方向

基于本文的基础，你可以继续扩展：

1. **高级物品系统**
   - 随机属性系统（词缀、随机属性）
   - 物品强化与升级
   - 物品镶嵌与宝石系统
   - 套装效果

2. **交易系统**
   - 玩家之间的物品交易
   - NPC 商店
   - 拍卖行

3. **制作系统**
   - 配方与制作
   - 材料分解
   - 装备修理

4. **仓库系统**
   - 共享仓库
   - 角色仓库
   - 容量扩展

5. **邮件系统**
   - 物品邮寄
   - 附件管理
   - 过期处理

### 下一步学习

- **第 12 篇：AI 系统与行为树**
- **第 13 篇：任务系统实现**
- **第 14 篇：对话系统与剧情编辑器**
- **第 15 篇：存档系统与云存档**

---

**感谢阅读！如有疑问，欢迎在评论区讨论。**

**记住：好的背包系统不是一蹴而就的，需要不断迭代和优化。从简单的开始，逐步添加功能，保持代码的整洁和可维护性。**

**祝你在 Lyra 的学习之旅中越走越远！💪**
