# 数据驱动设计：Data Assets 与配置管理

> **本章概览**：深入解析 Lyra 的数据驱动设计理念，掌握 UPrimaryDataAsset、UDataAsset、Gameplay Tags、Data Registry 等核心技术，学习如何构建灵活可扩展的游戏配置系统。

## 目录

- [1. 为什么需要数据驱动设计](#1-为什么需要数据驱动设计)
- [2. Unreal Engine 的数据资产体系](#2-unreal-engine-的数据资产体系)
- [3. Lyra 中的 Data Assets 全景](#3-lyra-中的-data-assets-全景)
- [4. Gameplay Tags 系统详解](#4-gameplay-tags-系统详解)
- [5. 核心 Data Assets 深度剖析](#5-核心-data-assets-深度剖析)
- [6. 实战案例：构建武器配置系统](#6-实战案例构建武器配置系统)
- [7. Data Registry 高级应用](#7-data-registry-高级应用)
- [8. 配置管理最佳实践](#8-配置管理最佳实践)
- [9. 性能优化与资源管理](#9-性能优化与资源管理)
- [10. 常见问题与解决方案](#10-常见问题与解决方案)

---

## 1. 为什么需要数据驱动设计

### 1.1 传统硬编码的痛点

在传统游戏开发中，我们经常会看到这样的代码：

```cpp
// ❌ 传统硬编码方式
class AMyWeapon : public AActor
{
public:
    AMyWeapon()
    {
        // 所有数据都写死在代码里
        Damage = 50.0f;
        FireRate = 0.1f;
        MagazineSize = 30;
        ReloadTime = 2.5f;
        WeaponName = TEXT("步枪");
        
        // 硬编码资源路径
        static ConstructorHelpers::FObjectFinder<UStaticMesh> MeshAsset(
            TEXT("/Game/Weapons/Rifle/SM_Rifle.SM_Rifle")
        );
        if (MeshAsset.Succeeded())
        {
            WeaponMesh->SetStaticMesh(MeshAsset.Object);
        }
    }
    
    float Damage;
    float FireRate;
    int32 MagazineSize;
    float ReloadTime;
    FText WeaponName;
};
```

**这种方式的问题：**

1. **策划无法调整**：每次改数值都需要程序员修改代码并重新编译
2. **扩展性差**：添加新武器需要创建新类，代码重复严重
3. **难以热更新**：改动需要重新打包游戏客户端
4. **团队协作困难**：程序和策划无法并行工作
5. **调试效率低**：改一个数值要等几分钟甚至几十分钟编译
6. **无法复用**：每个武器都是独立实现，逻辑无法共享

### 1.2 数据驱动设计的核心思想

**数据驱动设计（Data-Driven Design）** 的核心理念是：

> **代码定义系统行为（How），数据定义具体内容（What）**

```
┌─────────────────────────────────────────────────────────┐
│                    数据驱动架构                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐         ┌──────────────┐             │
│  │  Data Assets │  引用   │  Runtime     │             │
│  │  (配置数据)   │────────>│  Systems     │             │
│  │              │         │  (系统代码)   │             │
│  └──────────────┘         └──────────────┘             │
│                                                          │
│  策划/美术修改配置          程序员编写系统逻辑           │
│  无需编译，实时预览          一次开发，处理所有数据      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**数据驱动的优势：**

✅ **快速迭代**：策划直接在编辑器修改配置，保存后立即生效  
✅ **易于扩展**：添加新内容只需创建新配置文件，无需写代码  
✅ **职责分离**：程序负责系统框架，策划负责具体内容  
✅ **支持热更新**：配置文件可以独立更新，无需重新打包  
✅ **便于调试**：可以快速对比不同配置的效果  
✅ **代码复用**：一套系统代码处理所有配置数据  

### 1.3 Lyra 的数据驱动哲学

Lyra 将数据驱动设计发挥到了极致：

```
┌─────────────────────────────────────────────────────────┐
│              Lyra 数据驱动层次结构                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  第一层：Experience Definition (体验定义)                │
│  ├─ 定义整个游戏模式的玩法框架                           │
│  └─ 决定加载哪些 Game Features 和 Actions               │
│                                                          │
│  第二层：Pawn Data (角色数据)                            │
│  ├─ 定义角色的基础属性和能力集                           │
│  └─ 指定输入配置和相机模式                               │
│                                                          │
│  第三层：Ability Sets (能力集)                           │
│  ├─ 打包一组 Gameplay Abilities                         │
│  └─ 包含 Attributes 和 Gameplay Effects                 │
│                                                          │
│  第四层：Equipment Definitions (装备定义)                │
│  ├─ 定义武器、道具的具体配置                             │
│  └─ 指定关联的能力和生成的 Actor                         │
│                                                          │
│  第五层：Inventory Item Definitions (物品定义)           │
│  ├─ 定义物品属性和行为片段                               │
│  └─ 通过 Fragment 系统实现模块化                         │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Lyra 的关键设计原则：**

1. **分层配置**：从游戏模式到具体武器，层层配置，各司其职
2. **组合优于继承**：使用 Fragments、ActionSets 等组合模式
3. **强类型数据**：所有配置都是类型安全的 UObject
4. **引用管理**：使用 TSoftObjectPtr 实现异步加载
5. **Tags 驱动**：大量使用 Gameplay Tags 实现松耦合

---

## 2. Unreal Engine 的数据资产体系

### 2.1 UObject、UDataAsset、UPrimaryDataAsset 的关系

```cpp
// 继承层次结构
UObject
  └─ UDataAsset
       └─ UPrimaryDataAsset
```

#### 2.1.1 UObject：一切的基础

```cpp
// 最基础的 UObject
UCLASS()
class UMyConfigObject : public UObject
{
    GENERATED_BODY()
    
public:
    UPROPERTY(EditAnywhere, Category = "Config")
    float Value;
    
    UPROPERTY(EditAnywhere, Category = "Config")
    FString Name;
};
```

**特点：**
- ✅ 支持序列化、反射、垃圾回收
- ✅ 可以在蓝图和编辑器中使用
- ❌ 不能作为独立资产文件保存
- ❌ 必须附属于其他资产（如 Actor、Blueprint）

#### 2.1.2 UDataAsset：可保存的配置资产

```cpp
// UDataAsset 可以独立保存为 .uasset 文件
UCLASS()
class UMyDataAsset : public UDataAsset
{
    GENERATED_BODY()
    
public:
    UPROPERTY(EditAnywhere, Category = "Data")
    float Damage;
    
    UPROPERTY(EditAnywhere, Category = "Data")
    int32 Ammo;
};
```

**特点：**
- ✅ 可以保存为独立的 `.uasset` 文件
- ✅ 可以在内容浏览器中创建和编辑
- ✅ 支持引用其他资产
- ❌ 不支持异步加载管理
- ❌ 不支持 AssetBundle 打包

#### 2.1.3 UPrimaryDataAsset：高级数据资产

```cpp
// UPrimaryDataAsset 支持高级资源管理
UCLASS()
class UMyPrimaryDataAsset : public UPrimaryDataAsset
{
    GENERATED_BODY()
    
public:
    // 返回唯一的 Asset ID
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        return FPrimaryAssetId(
            FPrimaryAssetType("WeaponData"),
            GetFName()
        );
    }
    
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TSoftObjectPtr<UStaticMesh> WeaponMesh;
    
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TSoftObjectPtr<UAnimMontage> FireAnimation;
    
#if WITH_EDITORONLY_DATA
    // 定义资产打包规则
    virtual void UpdateAssetBundleData() override
    {
        Super::UpdateAssetBundleData();
        
        // 自动收集所有 TSoftObjectPtr 引用
        // 用于打包和异步加载管理
    }
#endif
};
```

**特点：**
- ✅ 拥有唯一的 `FPrimaryAssetId`
- ✅ 支持 Asset Manager 异步加载
- ✅ 支持 AssetBundle 打包策略
- ✅ 可以配置 Cook Rules（打包规则）
- ✅ 支持 Chunk 分包

**何时使用 UPrimaryDataAsset？**

| 场景 | 是否使用 | 原因 |
|------|---------|------|
| 简单配置表（伤害值、速度） | ❌ UDataAsset | 不需要复杂的加载管理 |
| 武器定义（包含网格、动画） | ✅ UPrimaryDataAsset | 需要异步加载和打包管理 |
| Experience 定义 | ✅ UPrimaryDataAsset | 需要动态加载 Game Features |
| Gameplay Tags 列表 | ❌ DataTable | 更适合表格结构 |

### 2.2 Asset Manager：资产生命周期管理

Asset Manager 是 UE 的中央资产管理系统，负责：

1. **资产注册**：扫描和注册所有 PrimaryDataAsset
2. **异步加载**：按需加载资产，避免阻塞主线程
3. **内存管理**：控制资产的加载和卸载时机
4. **打包策略**：决定哪些资产打包到哪个 Chunk

#### 2.2.1 配置 Asset Manager

在 `DefaultGame.ini` 中配置：

```ini
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="WeaponData",AssetBaseClass=/Script/MyGame.MyWeaponDefinition,bHasBlueprintClasses=True,bIsEditorOnly=False,Directories=((Path="/Game/Weapons")),SpecificAssets=,Rules=(Priority=-1,ChunkId=-1,bApplyRecursively=True,CookRule=Unknown))

+PrimaryAssetTypesToScan=(PrimaryAssetType="ExperienceDef",AssetBaseClass=/Script/LyraGame.LyraExperienceDefinition,bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/Experiences")),SpecificAssets=,Rules=(Priority=10,ChunkId=-1,bApplyRecursively=True,CookRule=AlwaysCook))
```

**参数说明：**
- `PrimaryAssetType`：资产类型名称，用于分类
- `AssetBaseClass`：基类，只扫描该类及其子类
- `Directories`：扫描的目录路径
- `Priority`：加载优先级（数字越大越优先）
- `CookRule`：打包规则
  - `AlwaysCook`：总是打包
  - `NeverCook`：从不打包
  - `DevelopmentCook`：仅开发版本打包
  - `Unknown`：根据引用决定

#### 2.2.2 异步加载 PrimaryDataAsset

```cpp
void UMyGameSubsystem::LoadWeaponData(const FString& WeaponName)
{
    // 1. 构造 Asset ID
    FPrimaryAssetId AssetId(
        FPrimaryAssetType("WeaponData"),
        FName(*WeaponName)
    );
    
    // 2. 异步加载
    UAssetManager& AssetManager = UAssetManager::Get();
    
    TSharedPtr<FStreamableHandle> Handle = AssetManager.LoadPrimaryAsset(
        AssetId,
        TArray<FName>(), // BundleNames（可选）
        FStreamableDelegate::CreateUObject(
            this,
            &UMyGameSubsystem::OnWeaponDataLoaded,
            AssetId
        )
    );
    
    // 3. 可选：等待加载完成
    if (Handle.IsValid() && Handle->IsActive())
    {
        // 已在加载中
        UE_LOG(LogTemp, Log, TEXT("Loading weapon data: %s"), *WeaponName);
    }
}

void UMyGameSubsystem::OnWeaponDataLoaded(FPrimaryAssetId AssetId)
{
    // 4. 获取加载完成的资产
    UAssetManager& AssetManager = UAssetManager::Get();
    UMyWeaponDefinition* WeaponData = Cast<UMyWeaponDefinition>(
        AssetManager.GetPrimaryAssetObject(AssetId)
    );
    
    if (WeaponData)
    {
        UE_LOG(LogTemp, Log, TEXT("Weapon data loaded: %s"), *WeaponData->GetName());
        // 使用加载的数据
        SpawnWeapon(WeaponData);
    }
}
```

**异步加载的好处：**
- ✅ 不阻塞游戏线程，保持帧率稳定
- ✅ 可以显示加载进度
- ✅ 支持加载优先级调度
- ✅ 自动管理内存，加载完成后可以释放

---

## 3. Lyra 中的 Data Assets 全景

### 3.1 Lyra Data Assets 分类图

```
Lyra Data Assets 体系
├── 📦 Experience 层
│   ├── ULyraExperienceDefinition (体验定义)
│   ├── ULyraExperienceActionSet (动作集)
│   └── ULyraUserFacingExperienceDefinition (用户界面体验)
│
├── 🎮 Pawn 层
│   ├── ULyraPawnData (角色数据)
│   └── ULyraInputConfig (输入配置)
│
├── ⚡ Ability 层
│   ├── ULyraAbilitySet (能力集)
│   └── ULyraAbilityTagRelationshipMapping (能力标签关系)
│
├── 🔫 Equipment 层
│   ├── ULyraEquipmentDefinition (装备定义)
│   └── ULyraPickupDefinition (拾取物定义)
│
├── 📦 Inventory 层
│   ├── ULyraInventoryItemDefinition (物品定义)
│   └── ULyraInventoryItemFragment (物品片段，基类)
│       ├── UInventoryFragment_ReticleConfig (准星配置片段)
│       ├── UInventoryFragment_SetStats (属性设置片段)
│       └── ... (其他自定义片段)
│
└── 🏆 Game Content 层
    ├── ULyraAccoladeDefinition (成就定义，ShooterCore)
    └── ... (其他游戏特定的配置)
```

### 3.2 核心 Data Assets 概览

#### 3.2.1 ULyraExperienceDefinition

**用途**：定义一个完整的游戏模式体验

**源码位置**：`Source/LyraGame/GameModes/LyraExperienceDefinition.h`

```cpp
UCLASS(BlueprintType, Const)
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 要启用的 Game Feature 插件列表
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<FString> GameFeaturesToEnable;

    // 默认的 Pawn 数据
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TObjectPtr<const ULyraPawnData> DefaultPawnData;

    // 要执行的 GameFeatureAction 列表
    UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    // 组合的动作集
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets;
};
```

**关键点：**
- 这是 Lyra 体验系统的核心
- 决定了整个游戏模式的加载流程
- 可以组合多个 ActionSets 实现模块化

#### 3.2.2 ULyraPawnData

**用途**：定义角色的基础配置

**源码位置**：`Source/LyraGame/Character/LyraPawnData.h`

```cpp
UCLASS(BlueprintType, Const)
class ULyraPawnData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // Pawn 类（角色蓝图）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Pawn")
    TSubclassOf<APawn> PawnClass;

    // 要赋予的能力集
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Abilities")
    TArray<TObjectPtr<ULyraAbilitySet>> AbilitySets;

    // 能力标签关系映射
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Abilities")
    TObjectPtr<ULyraAbilityTagRelationshipMapping> TagRelationshipMapping;

    // 输入配置
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Input")
    TObjectPtr<ULyraInputConfig> InputConfig;

    // 默认相机模式
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Camera")
    TSubclassOf<ULyraCameraMode> DefaultCameraMode;
};
```

**使用场景：**
- 第一人称射击角色
- 第三人称动作角色
- 自上而下视角角色
- Bot AI 角色

#### 3.2.3 ULyraAbilitySet

**用途**：打包一组 Gameplay Abilities、Effects 和 Attributes

**源码位置**：`Source/LyraGame/AbilitySystem/LyraAbilitySet.h`

```cpp
UCLASS(BlueprintType, Const)
class ULyraAbilitySet : public UPrimaryDataAsset
{
    GENERATED_BODY()

protected:
    // 要赋予的 Gameplay Abilities
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Abilities")
    TArray<FLyraAbilitySet_GameplayAbility> GrantedGameplayAbilities;

    // 要应用的 Gameplay Effects
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Effects")
    TArray<FLyraAbilitySet_GameplayEffect> GrantedGameplayEffects;

    // 要添加的 Attribute Sets
    UPROPERTY(EditDefaultsOnly, Category = "Attribute Sets")
    TArray<FLyraAbilitySet_AttributeSet> GrantedAttributes;
};

// 能力配置结构
USTRUCT(BlueprintType)
struct FLyraAbilitySet_GameplayAbility
{
    GENERATED_BODY()

    // 要赋予的能力类
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<ULyraGameplayAbility> Ability;

    // 能力等级
    UPROPERTY(EditDefaultsOnly)
    int32 AbilityLevel = 1;

    // 输入标签（用于绑定输入）
    UPROPERTY(EditDefaultsOnly, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};
```

**设计优势：**
- 一个 AbilitySet 可以被多个角色复用
- 可以动态添加/移除 AbilitySet（如装备武器时）
- 便于配置管理和版本控制

#### 3.2.4 ULyraEquipmentDefinition

**用途**：定义装备（武器、道具）的配置

**源码位置**：`Source/LyraGame/Equipment/LyraEquipmentDefinition.h`

```cpp
UCLASS(Blueprintable, Const, Abstract, BlueprintType)
class ULyraEquipmentDefinition : public UObject
{
    GENERATED_BODY()

public:
    // Equipment Instance 类型
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TSubclassOf<ULyraEquipmentInstance> InstanceType;

    // 装备时赋予的能力集
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant;

    // 要生成的 Actor（如武器网格）
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TArray<FLyraEquipmentActorToSpawn> ActorsToSpawn;
};

// Actor 生成配置
USTRUCT()
struct FLyraEquipmentActorToSpawn
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category=Equipment)
    TSubclassOf<AActor> ActorToSpawn; // 要生成的 Actor 类

    UPROPERTY(EditAnywhere, Category=Equipment)
    FName AttachSocket; // 附加到哪个 Socket

    UPROPERTY(EditAnywhere, Category=Equipment)
    FTransform AttachTransform; // 附加的相对变换
};
```

**典型应用：**
- 步枪装备定义
- 霰弹枪装备定义
- 手雷装备定义
- 治疗包装备定义

#### 3.2.5 ULyraInventoryItemDefinition

**用途**：定义物品，使用 Fragment 模式实现模块化

**源码位置**：`Source/LyraGame/Inventory/LyraInventoryItemDefinition.h`

```cpp
// 物品片段基类
UCLASS(MinimalAPI, DefaultToInstanced, EditInlineNew, Abstract)
class ULyraInventoryItemFragment : public UObject
{
    GENERATED_BODY()

public:
    // 当物品实例创建时调用
    virtual void OnInstanceCreated(ULyraInventoryItemInstance* Instance) const {}
};

// 物品定义
UCLASS(Blueprintable, Const, Abstract)
class ULyraInventoryItemDefinition : public UObject
{
    GENERATED_BODY()

public:
    // 显示名称
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display)
    FText DisplayName;

    // 物品片段列表
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display, Instanced)
    TArray<TObjectPtr<ULyraInventoryItemFragment>> Fragments;

public:
    // 查找特定类型的片段
    const ULyraInventoryItemFragment* FindFragmentByClass(
        TSubclassOf<ULyraInventoryItemFragment> FragmentClass
    ) const;
};
```

**Fragment 示例：准星配置片段**

```cpp
// 准星配置片段
UCLASS()
class UInventoryFragment_ReticleConfig : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    // 准星图标
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Reticle)
    TArray<TObjectPtr<ULyraReticleWidgetBase>> ReticleWidgets;
};
```

**Fragment 模式的优势：**
- ✅ 高度模块化，每个 Fragment 负责一个功能
- ✅ 易于扩展，添加新功能只需新增 Fragment
- ✅ 避免继承层次过深
- ✅ 可以灵活组合不同的 Fragments

---

## 4. Gameplay Tags 系统详解

### 4.1 什么是 Gameplay Tags

**Gameplay Tags** 是 Unreal Engine 的层级化字符串标签系统，用于：

1. **能力标识**：`Ability.Type.Action.Jump`
2. **状态标记**：`Status.Death.Dying`
3. **输入映射**：`InputTag.Weapon.Fire`
4. **事件触发**：`GameplayEvent.Death`
5. **条件判断**：`Cheat.GodMode`

**核心特性：**
- 层级化：`Parent.Child.GrandChild`
- 类型安全：编译期检查，避免拼写错误
- 高性能：内部使用 FName 和位掩码优化
- 可配置：在 `DefaultGameplayTags.ini` 中定义
- 支持查询：精确匹配、父子匹配、通配符匹配

### 4.2 Gameplay Tags 的定义方式

#### 4.2.1 方式一：配置文件定义（推荐）

在 `Config/DefaultGameplayTags.ini` 中：

```ini
[/Script/GameplayTags.GameplayTagsSettings]
ImportTagsFromConfig=True

+GameplayTagList=(Tag="Ability.Type.Action.Jump",DevComment="跳跃能力")
+GameplayTagList=(Tag="Ability.Type.Action.Dash",DevComment="冲刺能力")
+GameplayTagList=(Tag="Ability.Type.Passive.AutoReload",DevComment="自动装弹")

+GameplayTagList=(Tag="InputTag.Move",DevComment="移动输入")
+GameplayTagList=(Tag="InputTag.Look.Mouse",DevComment="鼠标视角")
+GameplayTagList=(Tag="InputTag.Weapon.Fire",DevComment="开火输入")

+GameplayTagList=(Tag="Status.Death",DevComment="死亡状态")
+GameplayTagList=(Tag="Status.Death.Dying",DevComment="濒死状态")
+GameplayTagList=(Tag="Status.Death.Dead",DevComment="已死亡状态")
```

**优势：**
- ✅ 集中管理，便于查看所有标签
- ✅ 支持版本控制
- ✅ 非程序员也可以添加标签
- ✅ 自动同步到编辑器

#### 4.2.2 方式二：C++ 原生标签定义

在 `LyraGameplayTags.h` 中：

```cpp
#pragma once

#include "NativeGameplayTags.h"

namespace LyraGameplayTags
{
    // 声明标签
    LYRAGAME_API UE_DECLARE_GAMEPLAY_TAG_EXTERN(Ability_ActivateFail_IsDead);
    LYRAGAME_API UE_DECLARE_GAMEPLAY_TAG_EXTERN(InputTag_Move);
    LYRAGAME_API UE_DECLARE_GAMEPLAY_TAG_EXTERN(Status_Death);
}
```

在 `LyraGameplayTags.cpp` 中：

```cpp
#include "LyraGameplayTags.h"
#include "GameplayTagsManager.h"

namespace LyraGameplayTags
{
    // 定义标签（带注释）
    UE_DEFINE_GAMEPLAY_TAG_COMMENT(
        Ability_ActivateFail_IsDead,
        "Ability.ActivateFail.IsDead",
        "能力激活失败：角色已死亡"
    );
    
    UE_DEFINE_GAMEPLAY_TAG_COMMENT(
        InputTag_Move,
        "InputTag.Move",
        "移动输入标签"
    );
    
    UE_DEFINE_GAMEPLAY_TAG_COMMENT(
        Status_Death,
        "Status.Death",
        "死亡状态标签"
    );
}
```

**使用原生标签：**

```cpp
#include "LyraGameplayTags.h"

void UMyAbility::ActivateAbility()
{
    // 直接使用，无需字符串查找
    if (ASC->HasMatchingGameplayTag(LyraGameplayTags::Status_Death))
    {
        UE_LOG(LogTemp, Warning, TEXT("角色已死亡，无法激活能力"));
        return;
    }
    
    // 添加标签
    ASC->AddLooseGameplayTag(LyraGameplayTags::Status_Crouching);
}
```

**优势：**
- ✅ 编译期检查，避免拼写错误
- ✅ IDE 自动补全和跳转
- ✅ 性能略优（避免运行时查找）
- ❌ 需要重新编译才能添加新标签

#### 4.2.3 方式三：DataTable 定义

创建一个 `GameplayTagTableRow` 类型的 DataTable：

| Tag | DevComment |
|-----|------------|
| Weapon.Type.Rifle | 步枪类武器 |
| Weapon.Type.Shotgun | 霰弹枪类武器 |
| Weapon.Type.Pistol | 手枪类武器 |

在配置文件中引用：

```ini
+GameplayTagTableList=/Game/Tags/DT_WeaponTags.DT_WeaponTags
```

**适用场景：**
- 需要策划在编辑器中维护大量标签
- 标签需要本地化（多语言支持）
- 标签需要与其他数据关联

### 4.3 Gameplay Tags 查询与匹配

#### 4.3.1 精确匹配

```cpp
// 检查是否拥有精确的标签
bool bHasTag = ASC->HasMatchingGameplayTag(
    FGameplayTag::RequestGameplayTag(FName("Status.Death.Dead"))
);
```

#### 4.3.2 父子匹配

```cpp
// 检查是否拥有 Status.Death 或其任何子标签（如 Status.Death.Dying）
FGameplayTag ParentTag = FGameplayTag::RequestGameplayTag(FName("Status.Death"));
bool bHasDeathStatus = ASC->HasMatchingGameplayTag(ParentTag);

// Status.Death.Dying 会匹配成功
// Status.Death.Dead 也会匹配成功
```

#### 4.3.3 容器匹配

```cpp
// 创建标签容器
FGameplayTagContainer TagsToCheck;
TagsToCheck.AddTag(LyraGameplayTags::Status_Death);
TagsToCheck.AddTag(LyraGameplayTags::Status_Stunned);

// 检查是否拥有任意一个标签
bool bHasAny = ASC->HasAnyMatchingGameplayTags(TagsToCheck);

// 检查是否拥有所有标签
bool bHasAll = ASC->HasAllMatchingGameplayTags(TagsToCheck);
```

#### 4.3.4 标签查询（高级）

```cpp
// 创建复杂的标签查询
FGameplayTagQuery Query = FGameplayTagQuery::MakeQuery_MatchAnyTags(
    FGameplayTagContainer(LyraGameplayTags::Ability_Type_Action)
);

// 检查角色是否满足查询条件
bool bMatchesQuery = Query.Matches(ASC->GetOwnedGameplayTags());
```

### 4.4 Lyra 中 Gameplay Tags 的典型应用

#### 4.4.1 输入映射

在 `ULyraInputConfig` 中：

```cpp
USTRUCT(BlueprintType)
struct FLyraInputAction
{
    GENERATED_BODY()

    // Enhanced Input Action
    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<const UInputAction> InputAction;

    // 关联的 Gameplay Tag
    UPROPERTY(EditDefaultsOnly, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};
```

配置示例：

| InputAction | InputTag |
|-------------|----------|
| IA_Move | InputTag.Move |
| IA_Look | InputTag.Look.Mouse |
| IA_Jump | InputTag.Jump |
| IA_Fire | InputTag.Weapon.Fire |

绑定输入时：

```cpp
void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    ULyraInputComponent* LyraIC = Cast<ULyraInputComponent>(PlayerInputComponent);
    
    // 绑定原生输入（Move、Look）
    LyraIC->BindNativeAction(
        InputConfig,
        LyraGameplayTags::InputTag_Move,
        ETriggerEvent::Triggered,
        this,
        &ThisClass::Input_Move
    );
    
    // 自动绑定能力输入（Jump、Fire）
    LyraIC->BindAbilityActions(
        InputConfig,
        this,
        &ThisClass::Input_AbilityInputTagPressed,
        &ThisClass::Input_AbilityInputTagReleased
    );
}
```

#### 4.4.2 能力激活条件

```cpp
UCLASS()
class ULyraGameplayAbility_Jump : public ULyraGameplayAbility
{
    GENERATED_BODY()

protected:
    virtual bool CanActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayTagContainer* SourceTags,
        const FGameplayTagContainer* TargetTags,
        FGameplayTagContainer* OptionalRelevantTags
    ) const override
    {
        // 检查角色状态
        if (ActorInfo->AbilitySystemComponent->HasMatchingGameplayTag(
            LyraGameplayTags::Status_Death))
        {
            // 死亡状态不能跳跃
            return false;
        }
        
        if (ActorInfo->AbilitySystemComponent->HasMatchingGameplayTag(
            LyraGameplayTags::Movement_Mode_Flying))
        {
            // 飞行状态不能跳跃
            return false;
        }
        
        return Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags);
    }
};
```

#### 4.4.3 能力标签阻塞

在能力定义中设置阻塞标签：

```cpp
ULyraGameplayAbility_Reload::ULyraGameplayAbility_Reload()
{
    // 激活时添加的标签
    ActivationOwnedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Type.Action.Reload")));
    
    // 阻塞其他能力激活
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Type.Action.WeaponFire")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Type.Action.Melee")));
    
    // 被其他标签阻塞
    BlockAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Type.Action")));
}
```

**效果：**
- 装弹时，无法开火和近战攻击
- 其他动作能力（如冲刺）正在执行时，无法装弹

#### 4.4.4 GameplayCue 触发

```cpp
// 在受到伤害时触发 GameplayCue
void ULyraHealthComponent::HandleDamage(float DamageAmount)
{
    Health = FMath::Max(0.0f, Health - DamageAmount);
    
    // 触发伤害特效
    FGameplayCueParameters CueParams;
    CueParams.SourceObject = DamageSource;
    CueParams.Instigator = DamageInstigator;
    CueParams.RawMagnitude = DamageAmount;
    
    ASC->ExecuteGameplayCue(
        FGameplayTag::RequestGameplayTag(FName("GameplayCue.Character.DamageTaken")),
        CueParams
    );
}
```

对应的 GameplayCue Actor：

```cpp
UCLASS()
class AGameplayCue_Character_DamageTaken : public AGameplayCueNotify_Actor
{
    GENERATED_BODY()

public:
    virtual bool OnExecute_Implementation(
        AActor* Target,
        const FGameplayCueParameters& Parameters
    ) override
    {
        // 播放受击音效
        UGameplayStatics::PlaySoundAtLocation(
            this,
            HitSound,
            Target->GetActorLocation()
        );
        
        // 生成血液粒子特效
        UNiagaraFunctionLibrary::SpawnSystemAtLocation(
            this,
            BloodEffect,
            Target->GetActorLocation()
        );
        
        return true;
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    USoundBase* HitSound;
    
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    UNiagaraSystem* BloodEffect;
};
```

### 4.5 Gameplay Tags 最佳实践

#### ✅ DO：推荐做法

1. **使用层级结构**
   ```
   ✅ Ability.Type.Action.Jump
   ❌ AbilityJump
   ```
   层级化便于批量查询和管理

2. **遵循命名约定**
   ```
   Ability.* - 能力相关
   InputTag.* - 输入相关
   Status.* - 状态相关
   GameplayCue.* - 特效相关
   GameplayEvent.* - 事件相关
   ```

3. **合理使用原生标签**
   ```cpp
   // 高频使用的标签定义为原生标签
   UE_DEFINE_GAMEPLAY_TAG(Status_Death, "Status.Death");
   
   // 运行时不变的标签定义为原生标签
   UE_DEFINE_GAMEPLAY_TAG(InputTag_Jump, "InputTag.Jump");
   ```

4. **使用容器避免重复查询**
   ```cpp
   // ❌ 多次查询
   bool bCanAct = !ASC->HasMatchingGameplayTag(Tag_Death)
               && !ASC->HasMatchingGameplayTag(Tag_Stunned)
               && !ASC->HasMatchingGameplayTag(Tag_Silenced);
   
   // ✅ 一次查询
   FGameplayTagContainer BlockingTags;
   BlockingTags.AddTag(Tag_Death);
   BlockingTags.AddTag(Tag_Stunned);
   BlockingTags.AddTag(Tag_Silenced);
   
   bool bCanAct = !ASC->HasAnyMatchingGameplayTags(BlockingTags);
   ```

#### ❌ DON'T：避免做法

1. **避免使用魔法字符串**
   ```cpp
   // ❌ 容易拼写错误
   FGameplayTag Tag = FGameplayTag::RequestGameplayTag(FName("Status.Deth")); // 拼错了！
   
   // ✅ 使用原生标签或常量
   FGameplayTag Tag = LyraGameplayTags::Status_Death;
   ```

2. **避免过深的层级**
   ```
   ❌ Ability.Type.Category.SubCategory.Action.Movement.Ground.Jump
   ✅ Ability.Type.Action.Jump
   ```
   过深的层级难以维护

3. **避免在运行时频繁创建标签**
   ```cpp
   // ❌ 每帧都查找标签
   void Tick(float DeltaTime)
   {
       FGameplayTag Tag = FGameplayTag::RequestGameplayTag(FName("Status.Moving"));
       if (ASC->HasMatchingGameplayTag(Tag)) { ... }
   }
   
   // ✅ 缓存标签
   class MyClass
   {
       FGameplayTag CachedMovingTag;
       
       MyClass()
       {
           CachedMovingTag = FGameplayTag::RequestGameplayTag(FName("Status.Moving"));
       }
       
       void Tick(float DeltaTime)
       {
           if (ASC->HasMatchingGameplayTag(CachedMovingTag)) { ... }
       }
   };
   ```

---

## 5. 核心 Data Assets 深度剖析

### 5.1 Experience Definition 的加载流程

#### 5.1.1 Experience 生命周期

```
┌─────────────────────────────────────────────────────────┐
│           Experience Definition 加载流程                 │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. GameMode BeginPlay                                   │
│     └─> LyraExperienceManager::StartExperienceLoad()    │
│                                                          │
│  2. 异步加载 Experience Asset                            │
│     ├─> AssetManager.LoadPrimaryAsset()                 │
│     └─> 等待 Experience Definition 加载完成              │
│                                                          │
│  3. 加载 Game Feature Plugins                            │
│     ├─> 遍历 GameFeaturesToEnable                        │
│     ├─> UGameFeaturesSubsystem::LoadGameFeature()       │
│     └─> 等待所有插件加载完成                             │
│                                                          │
│  4. 激活 Game Feature Actions                            │
│     ├─> 遍历 Experience.Actions                          │
│     ├─> 遍历 ActionSets[*].Actions                       │
│     ├─> OnGameFeatureActivating()                        │
│     └─> OnGameFeatureLoading()                           │
│                                                          │
│  5. 广播 Experience Loaded                               │
│     └─> OnExperienceLoaded.Broadcast()                   │
│         ├─> HeroComponent 初始化 Pawn                    │
│         ├─> UI 显示对应界面                              │
│         └─> 其他系统开始工作                             │
│                                                          │
│  6. Experience Running (游戏进行中)                      │
│                                                          │
│  7. Experience Deactivation (切换/结束)                  │
│     ├─> OnGameFeatureDeactivating()                      │
│     ├─> 卸载 Game Features                               │
│     └─> 清理资源                                         │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### 5.1.2 源码分析：Experience Manager

```cpp
// Source/LyraGame/GameModes/LyraExperienceManagerComponent.cpp

void ULyraExperienceManagerComponent::StartExperienceLoad()
{
    // 1. 构造 Experience Asset ID
    FPrimaryAssetId ExperienceId = FPrimaryAssetId(
        FPrimaryAssetType(ULyraExperienceDefinition::StaticClass()->GetFName()),
        FName(*CurrentExperience.GetAssetName())
    );
    
    // 2. 异步加载 Experience
    UAssetManager& AssetManager = UAssetManager::Get();
    
    TSharedPtr<FStreamableHandle> Handle = AssetManager.LoadPrimaryAsset(
        ExperienceId,
        TArray<FName>(),
        FStreamableDelegate::CreateUObject(
            this,
            &ThisClass::OnExperienceLoadComplete
        )
    );
    
    // 3. 记录加载句柄（用于取消加载）
    LoadingHandle = Handle;
}

void ULyraExperienceManagerComponent::OnExperienceLoadComplete()
{
    // 4. 获取加载完成的 Experience
    const ULyraExperienceDefinition* Experience = GetCurrentExperienceChecked();
    
    // 5. 收集所有需要加载的 Game Features
    TArray<FString> GameFeaturesToLoad;
    
    // 5.1 Experience 自己的 GameFeatures
    GameFeaturesToLoad.Append(Experience->GameFeaturesToEnable);
    
    // 5.2 ActionSets 中的 GameFeatures
    for (const ULyraExperienceActionSet* ActionSet : Experience->ActionSets)
    {
        if (ActionSet)
        {
            GameFeaturesToLoad.Append(ActionSet->GameFeaturesToEnable);
        }
    }
    
    // 6. 加载 Game Feature Plugins
    LoadGameFeatures(GameFeaturesToLoad);
}

void ULyraExperienceManagerComponent::LoadGameFeatures(
    const TArray<FString>& GameFeaturePluginURLs
)
{
    // 遍历所有插件
    for (const FString& PluginURL : GameFeaturePluginURLs)
    {
        // 异步加载插件
        UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(
            PluginURL,
            FGameFeaturePluginLoadComplete::CreateUObject(
                this,
                &ThisClass::OnGameFeaturePluginLoadComplete,
                PluginURL
            )
        );
        
        NumGameFeaturePluginsLoading++;
    }
}

void ULyraExperienceManagerComponent::OnGameFeaturePluginLoadComplete(
    const UE::GameFeatures::FResult& Result,
    const FString& PluginURL
)
{
    NumGameFeaturePluginsLoading--;
    
    // 检查是否所有插件都加载完成
    if (NumGameFeaturePluginsLoading == 0)
    {
        // 执行 Game Feature Actions
        ExecuteGameFeatureActions();
        
        // 广播 Experience 加载完成事件
        OnExperienceLoaded.Broadcast(GetCurrentExperienceChecked());
        OnExperienceLoaded_LowPriority.Broadcast(GetCurrentExperienceChecked());
    }
}

void ULyraExperienceManagerComponent::ExecuteGameFeatureActions()
{
    const ULyraExperienceDefinition* Experience = GetCurrentExperienceChecked();
    
    // 收集所有 Actions
    TArray<UGameFeatureAction*> AllActions;
    
    // 1. Experience 自己的 Actions
    AllActions.Append(Experience->Actions);
    
    // 2. ActionSets 的 Actions
    for (const ULyraExperienceActionSet* ActionSet : Experience->ActionSets)
    {
        if (ActionSet)
        {
            AllActions.Append(ActionSet->Actions);
        }
    }
    
    // 3. 执行所有 Actions
    for (UGameFeatureAction* Action : AllActions)
    {
        if (Action)
        {
            // 调用 Action 的生命周期函数
            Action->OnGameFeatureRegistering();
            Action->OnGameFeatureLoading();
            Action->OnGameFeatureActivating();
        }
    }
}
```

#### 5.1.3 创建自定义 Experience

**步骤1：创建 Experience Definition 资产**

在内容浏览器中：
1. 右键 → Miscellaneous → Data Asset
2. 选择 `LyraExperienceDefinition`
3. 命名：`DA_MyCustomExperience`

**步骤2：配置 Experience**

```
DA_MyCustomExperience (Data Asset)
├─ GameFeaturesToEnable
│  └─ /ShooterCore/ShooterCore (启用射击核心插件)
│
├─ DefaultPawnData
│  └─ DA_HeroPawnData_ShooterGame (使用射击游戏角色)
│
├─ Actions (GameFeatureActions)
│  ├─ AddAbilities (添加能力)
│  ├─ AddInputConfig (配置输入)
│  └─ AddUILayout (设置 UI)
│
└─ ActionSets (可选)
   └─ DA_ShooterCommonActions (复用射击通用配置)
```

**步骤3：配置默认 Experience**

在 `Config/DefaultGame.ini` 中：

```ini
[/Script/LyraGame.LyraGameMode]
DefaultExperience=/Game/Experiences/DA_MyCustomExperience.DA_MyCustomExperience
```

或在地图的 World Settings 中设置 `Override Experience`.

### 5.2 PawnData 的初始化时机

#### 5.2.1 Pawn 初始化流程

```
┌─────────────────────────────────────────────────────────┐
│              Lyra Pawn 初始化流程                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Pawn Spawned (生成)                                  │
│     └─> InitState = Spawned                              │
│                                                          │
│  2. Controller Possession (控制器接管)                    │
│     ├─> OnPossess()                                      │
│     └─> InitState = DataAvailable                        │
│                                                          │
│  3. LyraHeroComponent::OnPawnReadyToInitialize()         │
│     ├─> 从 PlayerState 获取 PawnData                     │
│     ├─> 创建 Pawn Extension Components                   │
│     └─> InitState = DataInitialized                      │
│                                                          │
│  4. PawnData 配置应用                                     │
│     ├─> 设置 Input Config                                │
│     ├─> 设置 Camera Mode                                 │
│     ├─> 授予 Ability Sets                                │
│     └─> 初始化 Ability System Component                  │
│                                                          │
│  5. 所有组件初始化完成                                    │
│     ├─> InitState = GameplayReady                        │
│     └─> Pawn 可以接受玩家输入并执行游戏逻辑              │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### 5.2.2 源码分析：PawnData 应用

```cpp
// Source/LyraGame/Character/LyraHeroComponent.cpp

void ULyraHeroComponent::OnPawnReadyToInitialize()
{
    // 1. 检查是否所有数据都已就绪
    if (!IsPawnComponentReadyToInitialize())
    {
        return;
    }
    
    // 2. 获取 PawnData
    ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>();
    const ULyraPawnData* PawnData = nullptr;
    
    if (LyraPS)
    {
        PawnData = LyraPS->GetPawnData<ULyraPawnData>();
    }
    
    if (!PawnData)
    {
        // 回退到 Experience 的默认 PawnData
        ULyraExperienceManagerComponent* ExperienceComponent = 
            UGameInstance->GetSubsystem<ULyraExperienceManagerComponent>();
        
        const ULyraExperienceDefinition* Experience = 
            ExperienceComponent->GetCurrentExperienceChecked();
        
        PawnData = Experience->DefaultPawnData;
    }
    
    check(PawnData);
    
    // 3. 应用 PawnData 配置
    InitializePawnData(PawnData);
    
    // 4. 标记初始化完成
    CheckGameplayReadiness();
}

void ULyraHeroComponent::InitializePawnData(const ULyraPawnData* PawnData)
{
    APawn* Pawn = GetPawn<APawn>();
    ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent();
    
    // 1. 设置输入配置
    if (PawnData->InputConfig)
    {
        InputConfig = PawnData->InputConfig;
    }
    
    // 2. 设置相机模式
    if (PawnData->DefaultCameraMode)
    {
        ULyraCameraComponent* CameraComponent = 
            Pawn->FindComponentByClass<ULyraCameraComponent>();
        
        if (CameraComponent)
        {
            CameraComponent->SetCameraMode(PawnData->DefaultCameraMode);
        }
    }
    
    // 3. 授予能力集
    for (const ULyraAbilitySet* AbilitySet : PawnData->AbilitySets)
    {
        if (AbilitySet)
        {
            AbilitySet->GiveToAbilitySystem(LyraASC, &GrantedAbilitySetHandles);
        }
    }
    
    // 4. 设置能力标签关系映射
    if (PawnData->TagRelationshipMapping)
    {
        LyraASC->SetTagRelationshipMapping(PawnData->TagRelationshipMapping);
    }
}
```

#### 5.2.3 创建自定义 PawnData

**创建 PawnData 资产：**

```
DA_MyCustomPawn (LyraPawnData)
├─ PawnClass
│  └─ BP_HeroCharacter (使用 Lyra 的 Hero Character 蓝图)
│
├─ AbilitySets
│  ├─ AbilitySet_Hero (基础角色能力：移动、跳跃)
│  └─ AbilitySet_MyCustom (自定义能力：特殊技能)
│
├─ InputConfig
│  └─ InputData_Hero (Enhanced Input 配置)
│
├─ DefaultCameraMode
│  └─ CM_ThirdPerson (第三人称相机)
│
└─ TagRelationshipMapping
   └─ TagRelationships_Default (能力标签关系)
```

**在 Experience 中引用：**

```
DA_MyExperience
└─ DefaultPawnData = DA_MyCustomPawn
```

### 5.3 AbilitySet 的授予机制

#### 5.3.1 AbilitySet 授予流程

```cpp
// Source/LyraGame/AbilitySystem/LyraAbilitySet.cpp

void ULyraAbilitySet::GiveToAbilitySystem(
    ULyraAbilitySystemComponent* LyraASC,
    FLyraAbilitySet_GrantedHandles* OutGrantedHandles,
    UObject* SourceObject
) const
{
    check(LyraASC);
    
    // 1. 授予 Gameplay Abilities
    for (const FLyraAbilitySet_GameplayAbility& AbilityToGrant : GrantedGameplayAbilities)
    {
        if (!IsValid(AbilityToGrant.Ability))
        {
            continue;
        }
        
        // 创建 Ability Spec
        FGameplayAbilitySpec AbilitySpec(
            AbilityToGrant.Ability,
            AbilityToGrant.AbilityLevel,
            INDEX_NONE,
            SourceObject
        );
        
        // 设置输入标签
        AbilitySpec.DynamicAbilityTags.AddTag(AbilityToGrant.InputTag);
        
        // 授予能力
        FGameplayAbilitySpecHandle Handle = LyraASC->GiveAbility(AbilitySpec);
        
        // 记录句柄（用于之后移除）
        if (OutGrantedHandles)
        {
            OutGrantedHandles->AddAbilitySpecHandle(Handle);
        }
    }
    
    // 2. 应用 Gameplay Effects
    for (const FLyraAbilitySet_GameplayEffect& EffectToGrant : GrantedGameplayEffects)
    {
        if (!IsValid(EffectToGrant.GameplayEffect))
        {
            continue;
        }
        
        // 创建 Effect Context
        FGameplayEffectContextHandle EffectContext = LyraASC->MakeEffectContext();
        EffectContext.AddSourceObject(SourceObject);
        
        // 创建 Effect Spec
        FGameplayEffectSpecHandle EffectSpecHandle = LyraASC->MakeOutgoingSpec(
            EffectToGrant.GameplayEffect,
            EffectToGrant.EffectLevel,
            EffectContext
        );
        
        // 应用 Effect
        FActiveGameplayEffectHandle ActiveEffectHandle = 
            LyraASC->ApplyGameplayEffectSpecToSelf(*EffectSpecHandle.Data.Get());
        
        // 记录句柄
        if (OutGrantedHandles)
        {
            OutGrantedHandles->AddGameplayEffectHandle(ActiveEffectHandle);
        }
    }
    
    // 3. 添加 Attribute Sets
    for (const FLyraAbilitySet_AttributeSet& SetToGrant : GrantedAttributes)
    {
        if (!IsValid(SetToGrant.AttributeSet))
        {
            continue;
        }
        
        // 创建 Attribute Set 实例
        UAttributeSet* NewSet = NewObject<UAttributeSet>(
            LyraASC->GetOwner(),
            SetToGrant.AttributeSet
        );
        
        // 添加到 ASC
        LyraASC->AddAttributeSetSubobject(NewSet);
        
        // 记录指针
        if (OutGrantedHandles)
        {
            OutGrantedHandles->AddAttributeSet(NewSet);
        }
    }
}
```

#### 5.3.2 AbilitySet 移除流程

```cpp
void FLyraAbilitySet_GrantedHandles::TakeFromAbilitySystem(
    ULyraAbilitySystemComponent* LyraASC
)
{
    check(LyraASC);
    
    // 1. 移除所有 Abilities
    for (const FGameplayAbilitySpecHandle& Handle : AbilitySpecHandles)
    {
        if (Handle.IsValid())
        {
            LyraASC->ClearAbility(Handle);
        }
    }
    
    // 2. 移除所有 Effects
    for (const FActiveGameplayEffectHandle& Handle : GameplayEffectHandles)
    {
        if (Handle.IsValid())
        {
            LyraASC->RemoveActiveGameplayEffect(Handle);
        }
    }
    
    // 3. 移除 Attribute Sets
    for (UAttributeSet* Set : GrantedAttributeSets)
    {
        if (Set)
        {
            LyraASC->RemoveSpawnedAttribute(Set);
        }
    }
    
    // 4. 清空句柄
    AbilitySpecHandles.Empty();
    GameplayEffectHandles.Empty();
    GrantedAttributeSets.Empty();
}
```

#### 5.3.3 动态授予和移除 AbilitySet

**示例：装备武器时授予能力**

```cpp
// 装备管理器组件
UCLASS()
class ULyraEquipmentManagerComponent : public UPawnComponent
{
    GENERATED_BODY()

public:
    // 装备武器
    void EquipWeapon(const ULyraEquipmentDefinition* EquipmentDef)
    {
        // 1. 生成武器 Actor
        for (const FLyraEquipmentActorToSpawn& ActorToSpawn : EquipmentDef->ActorsToSpawn)
        {
            AActor* NewActor = GetWorld()->SpawnActor<AActor>(
                ActorToSpawn.ActorToSpawn
            );
            
            // 附加到角色
            NewActor->AttachToComponent(
                GetOwner()->GetRootComponent(),
                FAttachmentTransformRules::KeepRelativeTransform,
                ActorToSpawn.AttachSocket
            );
            
            SpawnedActors.Add(NewActor);
        }
        
        // 2. 授予武器能力集
        ULyraAbilitySystemComponent* ASC = GetLyraAbilitySystemComponent();
        
        for (const ULyraAbilitySet* AbilitySet : EquipmentDef->AbilitySetsToGrant)
        {
            FLyraAbilitySet_GrantedHandles GrantedHandles;
            AbilitySet->GiveToAbilitySystem(ASC, &GrantedHandles, EquipmentDef);
            
            // 记录句柄，用于卸载武器时移除能力
            EquipmentAbilityHandles.Add(GrantedHandles);
        }
    }
    
    // 卸载武器
    void UnequipWeapon()
    {
        // 1. 移除武器能力
        ULyraAbilitySystemComponent* ASC = GetLyraAbilitySystemComponent();
        
        for (FLyraAbilitySet_GrantedHandles& Handles : EquipmentAbilityHandles)
        {
            Handles.TakeFromAbilitySystem(ASC);
        }
        EquipmentAbilityHandles.Empty();
        
        // 2. 销毁武器 Actor
        for (AActor* Actor : SpawnedActors)
        {
            if (Actor)
            {
                Actor->Destroy();
            }
        }
        SpawnedActors.Empty();
    }

protected:
    // 授予的能力句柄
    TArray<FLyraAbilitySet_GrantedHandles> EquipmentAbilityHandles;
    
    // 生成的武器 Actor
    TArray<AActor*> SpawnedActors;
};
```

### 5.4 EquipmentDefinition 与 InventoryItemDefinition 的区别

#### 5.4.1 概念对比

| 对比维度 | EquipmentDefinition | InventoryItemDefinition |
|---------|---------------------|-------------------------|
| **用途** | 定义可装备的物品（武器、护甲） | 定义所有物品（包括装备、消耗品） |
| **生命周期** | 只在装备时存在 | 可以长期存在于背包中 |
| **实例化** | 创建 EquipmentInstance | 创建 InventoryItemInstance |
| **能力授予** | 装备时授予，卸载时移除 | 通过 Fragment 决定 |
| **Actor 生成** | 定义要生成的 Actor（武器网格） | 不直接生成 Actor |
| **模块化** | 通过 AbilitySets 组合 | 通过 Fragments 组合 |

#### 5.4.2 关系图

```
┌──────────────────────────────────────────────────────────┐
│                 物品系统架构                              │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  InventoryItemDefinition (物品定义)                       │
│  ├─ DisplayName: "步枪"                                   │
│  └─ Fragments:                                            │
│     ├─ Fragment_EquippableItem                            │
│     │  └─ EquipmentDefinition ────┐                       │
│     ├─ Fragment_ReticleConfig     │                       │
│     ├─ Fragment_PickupIcon        │                       │
│     └─ Fragment_SetStats          │                       │
│                                   │                       │
│                                   ▼                       │
│  EquipmentDefinition (装备定义)                           │
│  ├─ InstanceType: RangedWeaponInstance                    │
│  ├─ AbilitySetsToGrant:                                   │
│  │  └─ AbilitySet_Rifle (射击能力、装弹能力)               │
│  └─ ActorsToSpawn:                                        │
│     └─ BP_Rifle_Weapon (武器网格 Actor)                   │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**工作流程：**

1. **拾取物品**：创建 `InventoryItemInstance`，添加到背包
2. **装备物品**：
   - 从 `Fragment_EquippableItem` 获取 `EquipmentDefinition`
   - 根据 `EquipmentDefinition` 创建 `EquipmentInstance`
   - 生成武器 Actor，授予能力
3. **卸载物品**：销毁 `EquipmentInstance`，但 `InventoryItemInstance` 仍在背包
4. **丢弃物品**：销毁 `InventoryItemInstance`

#### 5.4.3 Fragment 模式深入解析

**设计理念：**

传统方式：
```
ItemDefinition_Rifle (继承 ItemDefinition_Weapon)
ItemDefinition_Shotgun (继承 ItemDefinition_Weapon)
ItemDefinition_HealthPotion (继承 ItemDefinition_Consumable)
```

❌ **问题：**
- 继承层次复杂
- 功能无法灵活组合
- 修改基类影响所有子类

Fragment 方式：
```
ItemDefinition (基类)
└─ Fragments (组合不同功能片段)
   ├─ Fragment_EquippableItem (可装备)
   ├─ Fragment_Stackable (可堆叠)
   ├─ Fragment_Consumable (可消耗)
   ├─ Fragment_ReticleConfig (准星配置)
   └─ ... (自定义 Fragments)
```

✅ **优势：**
- 功能模块化
- 灵活组合
- 易于扩展

**实战案例：手雷物品**

```cpp
// 1. 创建手雷物品定义
DA_Item_Grenade (InventoryItemDefinition)
├─ DisplayName = "手雷"
└─ Fragments:
   ├─ Fragment_EquippableItem
   │  └─ EquipmentDefinition = DA_Equipment_Grenade
   ├─ Fragment_Stackable
   │  └─ MaxStackSize = 5
   ├─ Fragment_PickupIcon
   │  └─ Icon = T_Grenade_Icon
   └─ Fragment_ThrowableWeapon (自定义)
      ├─ ThrowForce = 2000.0
      └─ FuseTime = 3.0

// 2. 创建手雷装备定义
DA_Equipment_Grenade (EquipmentDefinition)
├─ InstanceType = GrenadeEquipmentInstance
├─ AbilitySetsToGrant:
│  └─ AbilitySet_Grenade
│     └─ Ability_Throw_Grenade
└─ ActorsToSpawn:
   └─ BP_Grenade_InHand (手持手雷网格)
```

**自定义 Fragment 示例：可投掷武器**

```cpp
// Fragment 定义
UCLASS()
class UInventoryFragment_ThrowableWeapon : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Throwable")
    float ThrowForce = 2000.0f;
    
    UPROPERTY(EditDefaultsOnly, Category = "Throwable")
    float FuseTime = 3.0f;
    
    UPROPERTY(EditDefaultsOnly, Category = "Throwable")
    TSubclassOf<AActor> ProjectileClass;
};

// 在能力中使用 Fragment
UCLASS()
class ULyraGameplayAbility_ThrowGrenade : public ULyraGameplayAbility
{
    GENERATED_BODY()

protected:
    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData
    ) override
    {
        // 1. 获取当前装备的物品
        ULyraEquipmentManagerComponent* EquipmentManager = 
            ActorInfo->AvatarActor->FindComponentByClass<ULyraEquipmentManagerComponent>();
        
        ULyraInventoryItemInstance* ItemInstance = 
            EquipmentManager->GetCurrentEquippedItem();
        
        // 2. 从物品定义中查找 ThrowableWeapon Fragment
        const ULyraInventoryItemDefinition* ItemDef = ItemInstance->GetItemDef();
        const UInventoryFragment_ThrowableWeapon* ThrowableFragment = 
            Cast<UInventoryFragment_ThrowableWeapon>(
                ItemDef->FindFragmentByClass(UInventoryFragment_ThrowableWeapon::StaticClass())
            );
        
        if (!ThrowableFragment)
        {
            CancelAbility(Handle, ActorInfo, ActivationInfo, true);
            return;
        }
        
        // 3. 使用 Fragment 中的配置生成投掷物
        FVector SpawnLocation = ActorInfo->AvatarActor->GetActorLocation();
        FRotator SpawnRotation = ActorInfo->AvatarActor->GetActorRotation();
        
        AActor* Projectile = GetWorld()->SpawnActor<AActor>(
            ThrowableFragment->ProjectileClass,
            SpawnLocation,
            SpawnRotation
        );
        
        // 4. 应用投掷力
        UPrimitiveComponent* ProjectileComp = 
            Projectile->FindComponentByClass<UPrimitiveComponent>();
        
        if (ProjectileComp)
        {
            FVector ThrowVelocity = SpawnRotation.Vector() * ThrowableFragment->ThrowForce;
            ProjectileComp->SetPhysicsLinearVelocity(ThrowVelocity);
        }
        
        // 5. 设置引信时间
        FTimerHandle FuseTimerHandle;
        GetWorld()->GetTimerManager().SetTimer(
            FuseTimerHandle,
            [Projectile]()
            {
                // 爆炸逻辑
                Projectile->Destroy();
            },
            ThrowableFragment->FuseTime,
            false
        );
        
        EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
    }
};
```

---

## 6. 实战案例：构建武器配置系统

### 6.1 需求分析

我们要实现一个完整的武器系统，支持：

1. **多种武器类型**：步枪、霰弹枪、手枪、狙击枪
2. **武器属性**：伤害、射速、后坐力、弹药容量
3. **武器配件**：瞄准镜、枪口、弹匣、握把
4. **动态切换**：运行时切换武器和配件
5. **数据驱动**：策划可以在编辑器中调整所有参数

### 6.2 架构设计

```
┌────────────────────────────────────────────────────────┐
│              武器系统架构                               │
├────────────────────────────────────────────────────────┤
│                                                         │
│  WeaponItemDefinition (InventoryItemDefinition)        │
│  └─ Fragments:                                          │
│     ├─ Fragment_EquippableItem                          │
│     │  └─ WeaponEquipmentDefinition                     │
│     ├─ Fragment_WeaponStats (自定义)                    │
│     │  ├─ Damage                                        │
│     │  ├─ FireRate                                      │
│     │  ├─ Accuracy                                      │
│     │  └─ Range                                         │
│     ├─ Fragment_AmmoConfig (自定义)                     │
│     │  ├─ MagazineSize                                  │
│     │  ├─ ReloadTime                                    │
│     │  └─ AmmoType                                      │
│     └─ Fragment_AttachmentSlots (自定义)                │
│        ├─ Scope Slot                                    │
│        ├─ Muzzle Slot                                   │
│        └─ Grip Slot                                     │
│                                                         │
│  WeaponEquipmentDefinition (EquipmentDefinition)        │
│  ├─ InstanceType: WeaponEquipmentInstance               │
│  ├─ AbilitySetsToGrant:                                 │
│  │  └─ AbilitySet_RangedWeapon                          │
│  │     ├─ Ability_Fire                                  │
│  │     ├─ Ability_Reload                                │
│  │     └─ Ability_ADS (瞄准)                            │
│  └─ ActorsToSpawn:                                      │
│     └─ WeaponMeshActor                                  │
│                                                         │
│  WeaponAttachmentDefinition (InventoryItemDefinition)   │
│  └─ Fragment_AttachmentMod                              │
│     ├─ Stat Modifiers (属性修改器)                      │
│     └─ Visual Mesh (外观网格)                           │
│                                                         │
└────────────────────────────────────────────────────────┘
```

### 6.3 实现代码

#### 6.3.1 定义武器属性 Fragment

```cpp
// WeaponFragments.h

#pragma once

#include "Inventory/LyraInventoryItemFragment.h"
#include "WeaponFragments.generated.h"

/**
 * 武器基础属性片段
 */
UCLASS()
class UInventoryFragment_WeaponStats : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    // 基础伤害
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Damage")
    float BaseDamage = 25.0f;
    
    // 爆头伤害倍率
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Damage")
    float HeadshotMultiplier = 2.0f;
    
    // 射速（发/分钟）
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Fire")
    float FireRate = 600.0f;
    
    // 射程（单位：厘米）
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Fire")
    float EffectiveRange = 5000.0f;
    
    // 精度（散布半径）
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Fire")
    float BaseSpread = 0.5f;
    
    // 后坐力
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Fire")
    FVector2D RecoilPattern = FVector2D(1.0f, 0.5f);
    
    // 武器类型标签
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Type", Meta = (Categories = "Weapon.Type"))
    FGameplayTag WeaponTypeTag;
    
    // 计算实际射击间隔（秒）
    float GetFireInterval() const
    {
        return 60.0f / FMath::Max(FireRate, 1.0f);
    }
};

/**
 * 弹药配置片段
 */
UCLASS()
class UInventoryFragment_AmmoConfig : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    // 弹匣容量
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Ammo")
    int32 MagazineSize = 30;
    
    // 最大备弹
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Ammo")
    int32 MaxReserveAmmo = 120;
    
    // 装弹时间（秒）
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Ammo")
    float ReloadTime = 2.5f;
    
    // 弹药类型
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Ammo")
    FGameplayTag AmmoType;
    
    // 是否自动装弹
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Ammo")
    bool bAutoReload = true;
};

/**
 * 武器配件槽片段
 */
USTRUCT(BlueprintType)
struct FWeaponAttachmentSlot
{
    GENERATED_BODY()
    
    // 槽位标签（如 Weapon.Attachment.Scope）
    UPROPERTY(EditDefaultsOnly, Category = "Attachment")
    FGameplayTag SlotTag;
    
    // 显示名称
    UPROPERTY(EditDefaultsOnly, Category = "Attachment")
    FText DisplayName;
    
    // 允许的配件类型
    UPROPERTY(EditDefaultsOnly, Category = "Attachment")
    FGameplayTagContainer AllowedAttachmentTypes;
    
    // 附加点名称
    UPROPERTY(EditDefaultsOnly, Category = "Attachment")
    FName AttachSocketName;
};

UCLASS()
class UInventoryFragment_AttachmentSlots : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    // 配件槽位列表
    UPROPERTY(EditDefaultsOnly, Category = "Weapon|Attachments")
    TArray<FWeaponAttachmentSlot> AttachmentSlots;
    
    // 查找槽位
    const FWeaponAttachmentSlot* FindSlotByTag(FGameplayTag SlotTag) const
    {
        return AttachmentSlots.FindByPredicate([SlotTag](const FWeaponAttachmentSlot& Slot)
        {
            return Slot.SlotTag == SlotTag;
        });
    }
};

/**
 * 配件属性修改片段
 */
USTRUCT(BlueprintType)
struct FWeaponStatModifier
{
    GENERATED_BODY()
    
    // 修改的属性名（如 "Damage", "Accuracy"）
    UPROPERTY(EditDefaultsOnly, Category = "Modifier")
    FName StatName;
    
    // 修改类型
    UPROPERTY(EditDefaultsOnly, Category = "Modifier")
    TEnumAsByte<EGameplayModOp::Type> ModifierOp = EGameplayModOp::Additive;
    
    // 修改值
    UPROPERTY(EditDefaultsOnly, Category = "Modifier")
    float ModifierValue = 0.0f;
};

UCLASS()
class UInventoryFragment_AttachmentMod : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    // 配件类型（用于匹配槽位）
    UPROPERTY(EditDefaultsOnly, Category = "Attachment", Meta = (Categories = "Weapon.Attachment"))
    FGameplayTag AttachmentType;
    
    // 属性修改器
    UPROPERTY(EditDefaultsOnly, Category = "Attachment")
    TArray<FWeaponStatModifier> StatModifiers;
    
    // 外观网格
    UPROPERTY(EditDefaultsOnly, Category = "Attachment")
    TSoftObjectPtr<UStaticMesh> AttachmentMesh;
};
```

#### 6.3.2 武器装备实例

```cpp
// WeaponEquipmentInstance.h

#pragma once

#include "Equipment/LyraEquipmentInstance.h"
#include "WeaponEquipmentInstance.generated.h"

/**
 * 武器装备实例
 * 负责管理武器的运行时状态
 */
UCLASS()
class UWeaponEquipmentInstance : public ULyraEquipmentInstance
{
    GENERATED_BODY()

public:
    virtual void OnEquipped() override;
    virtual void OnUnequipped() override;
    
    // 开火
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    bool Fire();
    
    // 装弹
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void Reload();
    
    // 安装配件
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void AttachMod(FGameplayTag SlotTag, ULyraInventoryItemInstance* AttachmentItem);
    
    // 移除配件
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void DetachMod(FGameplayTag SlotTag);
    
    // 获取当前弹药数
    UFUNCTION(BlueprintPure, Category = "Weapon")
    int32 GetCurrentAmmo() const { return CurrentAmmo; }
    
    // 获取备弹数
    UFUNCTION(BlueprintPure, Category = "Weapon")
    int32 GetReserveAmmo() const { return ReserveAmmo; }
    
    // 获取武器属性（考虑配件加成）
    UFUNCTION(BlueprintPure, Category = "Weapon")
    float GetEffectiveDamage() const;
    
    UFUNCTION(BlueprintPure, Category = "Weapon")
    float GetEffectiveFireRate() const;

protected:
    // 从 ItemDefinition 加载配置
    void LoadWeaponConfig();
    
    // 计算配件加成后的属性
    float CalculateModifiedStat(FName StatName, float BaseValue) const;
    
    // 应用配件的视觉效果
    void ApplyAttachmentVisuals(FGameplayTag SlotTag, UStaticMesh* AttachmentMesh);

protected:
    // 武器基础配置（缓存）
    UPROPERTY()
    const UInventoryFragment_WeaponStats* WeaponStats;
    
    UPROPERTY()
    const UInventoryFragment_AmmoConfig* AmmoConfig;
    
    // 当前弹药状态
    UPROPERTY(Replicated)
    int32 CurrentAmmo;
    
    UPROPERTY(Replicated)
    int32 ReserveAmmo;
    
    // 上次射击时间
    UPROPERTY()
    float LastFireTime;
    
    // 已安装的配件
    UPROPERTY(Replicated)
    TMap<FGameplayTag, ULyraInventoryItemInstance*> InstalledAttachments;
    
    // 生成的配件网格组件
    UPROPERTY()
    TMap<FGameplayTag, UStaticMeshComponent*> AttachmentMeshComponents;
};
```

```cpp
// WeaponEquipmentInstance.cpp

#include "WeaponEquipmentInstance.h"
#include "Net/UnrealNetwork.h"
#include "WeaponFragments.h"

void UWeaponEquipmentInstance::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    
    DOREPLIFETIME(UWeaponEquipmentInstance, CurrentAmmo);
    DOREPLIFETIME(UWeaponEquipmentInstance, ReserveAmmo);
    DOREPLIFETIME(UWeaponEquipmentInstance, InstalledAttachments);
}

void UWeaponEquipmentInstance::OnEquipped()
{
    Super::OnEquipped();
    
    // 加载武器配置
    LoadWeaponConfig();
    
    // 初始化弹药
    if (AmmoConfig)
    {
        CurrentAmmo = AmmoConfig->MagazineSize;
        ReserveAmmo = AmmoConfig->MaxReserveAmmo;
    }
    
    LastFireTime = 0.0f;
}

void UWeaponEquipmentInstance::LoadWeaponConfig()
{
    const ULyraInventoryItemDefinition* ItemDef = GetInstigator()->GetItemDef();
    
    if (!ItemDef)
    {
        return;
    }
    
    // 缓存 Fragments
    WeaponStats = Cast<UInventoryFragment_WeaponStats>(
        ItemDef->FindFragmentByClass(UInventoryFragment_WeaponStats::StaticClass())
    );
    
    AmmoConfig = Cast<UInventoryFragment_AmmoConfig>(
        ItemDef->FindFragmentByClass(UInventoryFragment_AmmoConfig::StaticClass())
    );
}

bool UWeaponEquipmentInstance::Fire()
{
    // 1. 检查是否可以射击
    if (CurrentAmmo <= 0)
    {
        // 尝试自动装弹
        if (AmmoConfig && AmmoConfig->bAutoReload && ReserveAmmo > 0)
        {
            Reload();
        }
        return false;
    }
    
    // 2. 检查射速限制
    float CurrentTime = GetWorld()->GetTimeSeconds();
    float FireInterval = WeaponStats ? WeaponStats->GetFireInterval() : 0.1f;
    
    if (CurrentTime - LastFireTime < FireInterval)
    {
        return false;
    }
    
    // 3. 消耗弹药
    CurrentAmmo--;
    LastFireTime = CurrentTime;
    
    // 4. 触发射击逻辑（由 Gameplay Ability 处理）
    // 这里只处理弹药消耗和射速限制
    
    return true;
}

void UWeaponEquipmentInstance::Reload()
{
    if (!AmmoConfig)
    {
        return;
    }
    
    // 计算需要装填的弹药数
    int32 AmmoNeeded = AmmoConfig->MagazineSize - CurrentAmmo;
    int32 AmmoToReload = FMath::Min(AmmoNeeded, ReserveAmmo);
    
    if (AmmoToReload > 0)
    {
        // 延迟装填（由 Reload Ability 处理动画和时间）
        ReserveAmmo -= AmmoToReload;
        CurrentAmmo += AmmoToReload;
    }
}

void UWeaponEquipmentInstance::AttachMod(
    FGameplayTag SlotTag,
    ULyraInventoryItemInstance* AttachmentItem
)
{
    if (!AttachmentItem)
    {
        return;
    }
    
    const ULyraInventoryItemDefinition* AttachmentDef = AttachmentItem->GetItemDef();
    const UInventoryFragment_AttachmentMod* ModFragment = Cast<UInventoryFragment_AttachmentMod>(
        AttachmentDef->FindFragmentByClass(UInventoryFragment_AttachmentMod::StaticClass())
    );
    
    if (!ModFragment)
    {
        return;
    }
    
    // 移除旧配件（如果有）
    DetachMod(SlotTag);
    
    // 安装新配件
    InstalledAttachments.Add(SlotTag, AttachmentItem);
    
    // 应用视觉效果
    if (ModFragment->AttachmentMesh.IsValid())
    {
        UStaticMesh* Mesh = ModFragment->AttachmentMesh.LoadSynchronous();
        ApplyAttachmentVisuals(SlotTag, Mesh);
    }
    
    // 通知客户端更新
    if (GetOwner()->HasAuthority())
    {
        ForceNetUpdate();
    }
}

void UWeaponEquipmentInstance::DetachMod(FGameplayTag SlotTag)
{
    if (InstalledAttachments.Contains(SlotTag))
    {
        InstalledAttachments.Remove(SlotTag);
        
        // 移除视觉效果
        if (UStaticMeshComponent** CompPtr = AttachmentMeshComponents.Find(SlotTag))
        {
            (*CompPtr)->DestroyComponent();
            AttachmentMeshComponents.Remove(SlotTag);
        }
    }
}

float UWeaponEquipmentInstance::GetEffectiveDamage() const
{
    float BaseDamage = WeaponStats ? WeaponStats->BaseDamage : 0.0f;
    return CalculateModifiedStat(FName("Damage"), BaseDamage);
}

float UWeaponEquipmentInstance::GetEffectiveFireRate() const
{
    float BaseFireRate = WeaponStats ? WeaponStats->FireRate : 600.0f;
    return CalculateModifiedStat(FName("FireRate"), BaseFireRate);
}

float UWeaponEquipmentInstance::CalculateModifiedStat(FName StatName, float BaseValue) const
{
    float FinalValue = BaseValue;
    
    // 遍历所有已安装的配件
    for (const auto& Pair : InstalledAttachments)
    {
        ULyraInventoryItemInstance* AttachmentItem = Pair.Value;
        if (!AttachmentItem)
        {
            continue;
        }
        
        const ULyraInventoryItemDefinition* AttachmentDef = AttachmentItem->GetItemDef();
        const UInventoryFragment_AttachmentMod* ModFragment = Cast<UInventoryFragment_AttachmentMod>(
            AttachmentDef->FindFragmentByClass(UInventoryFragment_AttachmentMod::StaticClass())
        );
        
        if (!ModFragment)
        {
            continue;
        }
        
        // 应用配件的属性修改器
        for (const FWeaponStatModifier& Modifier : ModFragment->StatModifiers)
        {
            if (Modifier.StatName == StatName)
            {
                switch (Modifier.ModifierOp)
                {
                case EGameplayModOp::Additive:
                    FinalValue += Modifier.ModifierValue;
                    break;
                    
                case EGameplayModOp::Multiplicitive:
                    FinalValue *= Modifier.ModifierValue;
                    break;
                    
                case EGameplayModOp::Override:
                    FinalValue = Modifier.ModifierValue;
                    break;
                    
                default:
                    break;
                }
            }
        }
    }
    
    return FinalValue;
}

void UWeaponEquipmentInstance::ApplyAttachmentVisuals(
    FGameplayTag SlotTag,
    UStaticMesh* AttachmentMesh
)
{
    // 获取武器根组件
    AActor* SpawnedActor = GetSpawnedActors().Num() > 0 ? GetSpawnedActors()[0] : nullptr;
    if (!SpawnedActor)
    {
        return;
    }
    
    // 查找配件槽位
    const ULyraInventoryItemDefinition* ItemDef = GetInstigator()->GetItemDef();
    const UInventoryFragment_AttachmentSlots* SlotsFragment = Cast<UInventoryFragment_AttachmentSlots>(
        ItemDef->FindFragmentByClass(UInventoryFragment_AttachmentSlots::StaticClass())
    );
    
    if (!SlotsFragment)
    {
        return;
    }
    
    const FWeaponAttachmentSlot* Slot = SlotsFragment->FindSlotByTag(SlotTag);
    if (!Slot)
    {
        return;
    }
    
    // 创建 StaticMeshComponent
    UStaticMeshComponent* AttachmentComp = NewObject<UStaticMeshComponent>(
        SpawnedActor,
        UStaticMeshComponent::StaticClass(),
        NAME_None,
        RF_Transient
    );
    
    AttachmentComp->SetStaticMesh(AttachmentMesh);
    AttachmentComp->RegisterComponent();
    
    // 附加到指定 Socket
    AttachmentComp->AttachToComponent(
        SpawnedActor->GetRootComponent(),
        FAttachmentTransformRules::SnapToTargetIncludingScale,
        Slot->AttachSocketName
    );
    
    // 记录组件
    AttachmentMeshComponents.Add(SlotTag, AttachmentComp);
}
```

#### 6.3.3 创建武器配置资产

**1. 创建步枪物品定义**

```
DA_Item_AssaultRifle (InventoryItemDefinition)
├─ DisplayName = "突击步枪"
└─ Fragments:
   ├─ Fragment_EquippableItem
   │  └─ EquipmentDefinition = DA_Equipment_AssaultRifle
   ├─ Fragment_WeaponStats
   │  ├─ BaseDamage = 30.0
   │  ├─ FireRate = 600.0
   │  ├─ EffectiveRange = 5000.0
   │  ├─ BaseSpread = 0.5
   │  └─ WeaponTypeTag = Weapon.Type.Rifle
   ├─ Fragment_AmmoConfig
   │  ├─ MagazineSize = 30
   │  ├─ MaxReserveAmmo = 120
   │  ├─ ReloadTime = 2.5
   │  └─ AmmoType = Ammo.Type.556
   └─ Fragment_AttachmentSlots
      ├─ Slot 1:
      │  ├─ SlotTag = Weapon.Attachment.Scope
      │  ├─ DisplayName = "瞄准镜"
      │  ├─ AllowedAttachmentTypes = [Weapon.Attachment.Scope.*]
      │  └─ AttachSocketName = "ScopeSocket"
      ├─ Slot 2:
      │  ├─ SlotTag = Weapon.Attachment.Muzzle
      │  └─ ...
      └─ Slot 3:
         ├─ SlotTag = Weapon.Attachment.Grip
         └─ ...
```

**2. 创建配件定义**

```
DA_Attachment_RedDotSight (InventoryItemDefinition)
├─ DisplayName = "红点瞄准镜"
└─ Fragments:
   └─ Fragment_AttachmentMod
      ├─ AttachmentType = Weapon.Attachment.Scope.RedDot
      ├─ StatModifiers:
      │  └─ [0]:
      │     ├─ StatName = "Accuracy"
      │     ├─ ModifierOp = Multiplicitive
      │     └─ ModifierValue = 1.2 (提升 20% 精度)
      └─ AttachmentMesh = SM_RedDotSight
```

**3. 在蓝图中使用**

```cpp
// 玩家拾取武器时
void AMyCharacter::PickupWeapon(AWeaponPickup* Pickup)
{
    // 添加到背包
    ULyraInventoryManagerComponent* InventoryManager = 
        FindComponentByClass<ULyraInventoryManagerComponent>();
    
    ULyraInventoryItemInstance* ItemInstance = InventoryManager->AddItemDefinition(
        Pickup->WeaponItemDefinition,
        1
    );
    
    // 装备武器
    ULyraEquipmentManagerComponent* EquipmentManager = 
        FindComponentByClass<ULyraEquipmentManagerComponent>();
    
    EquipmentManager->EquipItem(ItemInstance);
}

// 安装配件
void AMyCharacter::InstallWeaponAttachment(
    FGameplayTag SlotTag,
    ULyraInventoryItemInstance* AttachmentItem
)
{
    ULyraEquipmentManagerComponent* EquipmentManager = 
        FindComponentByClass<ULyraEquipmentManagerComponent>();
    
    UWeaponEquipmentInstance* CurrentWeapon = Cast<UWeaponEquipmentInstance>(
        EquipmentManager->GetCurrentEquipment()
    );
    
    if (CurrentWeapon)
    {
        CurrentWeapon->AttachMod(SlotTag, AttachmentItem);
    }
}
```

### 6.4 UI 显示武器信息

```cpp
// WeaponInfoWidget.h

UCLASS()
class UWeaponInfoWidget : public UUserWidget
{
    GENERATED_BODY()

protected:
    virtual void NativeConstruct() override;
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;
    
    // 更新武器信息显示
    UFUNCTION(BlueprintImplementableEvent, Category = "Weapon")
    void UpdateWeaponInfo(
        FText WeaponName,
        int32 CurrentAmmo,
        int32 ReserveAmmo,
        float Damage,
        float FireRate
    );
};
```

```cpp
// WeaponInfoWidget.cpp

void UWeaponInfoWidget::NativeConstruct()
{
    Super::NativeConstruct();
    
    // 监听武器切换事件
    if (APawn* OwnerPawn = GetOwningPlayerPawn())
    {
        ULyraEquipmentManagerComponent* EquipmentManager = 
            OwnerPawn->FindComponentByClass<ULyraEquipmentManagerComponent>();
        
        if (EquipmentManager)
        {
            EquipmentManager->OnEquipmentChanged.AddDynamic(
                this,
                &ThisClass::HandleEquipmentChanged
            );
        }
    }
}

void UWeaponInfoWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    Super::NativeTick(MyGeometry, InDeltaTime);
    
    // 获取当前武器
    APawn* OwnerPawn = GetOwningPlayerPawn();
    if (!OwnerPawn)
    {
        return;
    }
    
    ULyraEquipmentManagerComponent* EquipmentManager = 
        OwnerPawn->FindComponentByClass<ULyraEquipmentManagerComponent>();
    
    if (!EquipmentManager)
    {
        return;
    }
    
    UWeaponEquipmentInstance* CurrentWeapon = Cast<UWeaponEquipmentInstance>(
        EquipmentManager->GetCurrentEquipment()
    );
    
    if (!CurrentWeapon)
    {
        return;
    }
    
    // 获取武器信息
    const ULyraInventoryItemDefinition* ItemDef = CurrentWeapon->GetInstigator()->GetItemDef();
    FText WeaponName = ItemDef ? ItemDef->DisplayName : FText::GetEmpty();
    
    int32 CurrentAmmo = CurrentWeapon->GetCurrentAmmo();
    int32 ReserveAmmo = CurrentWeapon->GetReserveAmmo();
    float Damage = CurrentWeapon->GetEffectiveDamage();
    float FireRate = CurrentWeapon->GetEffectiveFireRate();
    
    // 更新UI
    UpdateWeaponInfo(WeaponName, CurrentAmmo, ReserveAmmo, Damage, FireRate);
}
```

---

## 7. Data Registry 高级应用

### 7.1 什么是 Data Registry

**Data Registry** 是 UE5 引入的高级数据管理系统，用于：

1. **集中管理大量数据**：替代 DataTable，支持更复杂的数据结构
2. **动态数据加载**：按需异步加载数据，不会阻塞主线程
3. **数据覆盖与分层**：支持多层数据源（基础数据 + Mod 数据）
4. **运行时查询**：高效的数据检索和缓存机制

### 7.2 Data Registry vs DataTable

| 特性 | DataTable | Data Registry |
|------|-----------|---------------|
| **数据量** | 适合小量数据（<1000行） | 支持海量数据 |
| **加载方式** | 一次性全部加载 | 按需异步加载 |
| **内存占用** | 一直占用 | 可释放不用的数据 |
| **数据覆盖** | 不支持 | 支持分层覆盖 |
| **类型安全** | ✅ | ✅ |
| **蓝图支持** | ✅ | ✅ |
| **运行时修改** | ❌ | ✅（通过 Meta Data） |

### 7.3 创建 Data Registry

#### 7.3.1 定义数据结构

```cpp
// WeaponRegistryTypes.h

#pragma once

#include "Engine/DataTable.h"
#include "GameplayTagContainer.h"
#include "WeaponRegistryTypes.generated.h"

/**
 * 武器注册表数据
 * 用于 Data Registry 的数据行
 */
USTRUCT(BlueprintType)
struct FWeaponRegistryData : public FTableRowBase
{
    GENERATED_BODY()

public:
    // 武器显示名称
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Weapon")
    FText DisplayName;
    
    // 武器描述
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Weapon")
    FText Description;
    
    // 武器图标
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Weapon")
    TSoftObjectPtr<UTexture2D> Icon;
    
    // 武器网格
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Weapon")
    TSoftObjectPtr<UStaticMesh> WeaponMesh;
    
    // 武器类型标签
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Weapon", Meta = (Categories = "Weapon.Type"))
    FGameplayTag WeaponType;
    
    // 基础伤害
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Stats")
    float Damage;
    
    // 射速（发/分钟）
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Stats")
    float FireRate;
    
    // 弹匣容量
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Ammo")
    int32 MagazineSize;
    
    // 物品定义引用（实际的游戏物品）
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Reference")
    TSoftObjectPtr<ULyraInventoryItemDefinition> ItemDefinition;
};

/**
 * 配件注册表数据
 */
USTRUCT(BlueprintType)
struct FAttachmentRegistryData : public FTableRowBase
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Attachment")
    FText DisplayName;
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Attachment")
    TSoftObjectPtr<UTexture2D> Icon;
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Attachment", Meta = (Categories = "Weapon.Attachment"))
    FGameplayTag AttachmentType;
    
    // 属性加成
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Stats")
    TMap<FName, float> StatBonuses;
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Reference")
    TSoftObjectPtr<ULyraInventoryItemDefinition> ItemDefinition;
};
```

#### 7.3.2 创建 Data Registry 资产

**1. 创建 Data Registry**

在内容浏览器中：
1. 右键 → Miscellaneous → Data Registry
2. 命名：`DR_Weapons`

**2. 配置 Data Registry**

```
DR_Weapons (DataRegistry)
├─ Item Struct = FWeaponRegistryData
├─ ID Name = WeaponID
└─ Data Sources:
   ├─ DataTable: DT_WeaponsBase (基础武器数据)
   ├─ DataTable: DT_WeaponsMod1 (Mod 扩展数据)
   └─ CurveTable: CT_WeaponScaling (武器等级缩放)
```

**3. 创建 DataTable 数据源**

创建 `DT_WeaponsBase`（DataTable，行结构 = FWeaponRegistryData）：

| Row Name | DisplayName | Damage | FireRate | MagazineSize | WeaponType | ItemDefinition |
|----------|-------------|--------|----------|--------------|------------|----------------|
| Rifle_AK47 | AK-47 | 35.0 | 600 | 30 | Weapon.Type.Rifle | /Game/Items/DA_Item_AK47 |
| Rifle_M4A1 | M4A1 | 30.0 | 750 | 30 | Weapon.Type.Rifle | /Game/Items/DA_Item_M4A1 |
| Shotgun_M870 | M870 | 80.0 | 80 | 8 | Weapon.Type.Shotgun | /Game/Items/DA_Item_M870 |
| Pistol_Glock | Glock 18 | 25.0 | 1200 | 17 | Weapon.Type.Pistol | /Game/Items/DA_Item_Glock |

#### 7.3.3 查询 Data Registry

```cpp
// WeaponDataRegistry.h

#pragma once

#include "Subsystems/GameInstanceSubsystem.h"
#include "DataRegistrySubsystem.h"
#include "WeaponRegistryTypes.h"
#include "WeaponDataRegistry.generated.h"

/**
 * 武器数据注册表子系统
 * 封装 Data Registry 的查询接口
 */
UCLASS()
class UWeaponDataRegistrySubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 初始化
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    
    // 同步查询武器数据（适用于已缓存的数据）
    UFUNCTION(BlueprintCallable, Category = "Weapon Registry")
    bool GetWeaponData(FName WeaponID, FWeaponRegistryData& OutData);
    
    // 异步加载武器数据
    UFUNCTION(BlueprintCallable, Category = "Weapon Registry")
    void LoadWeaponDataAsync(
        FName WeaponID,
        FDataRegistryItemAcquiredCallback OnLoaded
    );
    
    // 批量加载武器数据
    UFUNCTION(BlueprintCallable, Category = "Weapon Registry")
    void LoadWeaponDataBatch(
        const TArray<FName>& WeaponIDs,
        FDataRegistryItemAcquiredCallback OnLoaded
    );
    
    // 查询所有武器ID
    UFUNCTION(BlueprintCallable, Category = "Weapon Registry")
    TArray<FName> GetAllWeaponIDs();
    
    // 根据标签筛选武器
    UFUNCTION(BlueprintCallable, Category = "Weapon Registry")
    TArray<FName> GetWeaponsByType(FGameplayTag WeaponTypeTag);

protected:
    // Data Registry 类型
    UPROPERTY()
    FDataRegistryType WeaponRegistryType;
    
    // 缓存的武器数据
    UPROPERTY()
    TMap<FName, FWeaponRegistryData> CachedWeaponData;
};
```

```cpp
// WeaponDataRegistry.cpp

#include "WeaponDataRegistry.h"
#include "DataRegistrySubsystem.h"

void UWeaponDataRegistrySubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    
    // 初始化 Data Registry 类型
    WeaponRegistryType = FDataRegistryType(FName("DR_Weapons"));
}

bool UWeaponDataRegistrySubsystem::GetWeaponData(
    FName WeaponID,
    FWeaponRegistryData& OutData
)
{
    // 1. 检查缓存
    if (CachedWeaponData.Contains(WeaponID))
    {
        OutData = CachedWeaponData[WeaponID];
        return true;
    }
    
    // 2. 从 Data Registry 查询
    UDataRegistrySubsystem* DataRegistrySubsystem = 
        UDataRegistrySubsystem::Get();
    
    if (!DataRegistrySubsystem)
    {
        return false;
    }
    
    FDataRegistryId DataId(WeaponRegistryType, WeaponID);
    
    const FWeaponRegistryData* DataPtr = DataRegistrySubsystem->GetCachedItem<FWeaponRegistryData>(DataId);
    
    if (DataPtr)
    {
        OutData = *DataPtr;
        
        // 加入缓存
        CachedWeaponData.Add(WeaponID, OutData);
        
        return true;
    }
    
    return false;
}

void UWeaponDataRegistrySubsystem::LoadWeaponDataAsync(
    FName WeaponID,
    FDataRegistryItemAcquiredCallback OnLoaded
)
{
    UDataRegistrySubsystem* DataRegistrySubsystem = 
        UDataRegistrySubsystem::Get();
    
    if (!DataRegistrySubsystem)
    {
        return;
    }
    
    FDataRegistryId DataId(WeaponRegistryType, WeaponID);
    
    // 异步加载数据
    DataRegistrySubsystem->AcquireItem(
        DataId,
        OnLoaded
    );
}

void UWeaponDataRegistrySubsystem::LoadWeaponDataBatch(
    const TArray<FName>& WeaponIDs,
    FDataRegistryItemAcquiredCallback OnLoaded
)
{
    TArray<FDataRegistryId> DataIds;
    
    for (FName WeaponID : WeaponIDs)
    {
        DataIds.Add(FDataRegistryId(WeaponRegistryType, WeaponID));
    }
    
    UDataRegistrySubsystem* DataRegistrySubsystem = 
        UDataRegistrySubsystem::Get();
    
    if (!DataRegistrySubsystem)
    {
        return;
    }
    
    // 批量异步加载
    DataRegistrySubsystem->AcquireItemBatch(
        DataIds,
        OnLoaded
    );
}

TArray<FName> UWeaponDataRegistrySubsystem::GetAllWeaponIDs()
{
    UDataRegistrySubsystem* DataRegistrySubsystem = 
        UDataRegistrySubsystem::Get();
    
    if (!DataRegistrySubsystem)
    {
        return TArray<FName>();
    }
    
    TArray<FDataRegistryId> AllIds;
    DataRegistrySubsystem->GetAllCachedItems(WeaponRegistryType, AllIds);
    
    TArray<FName> WeaponIDs;
    for (const FDataRegistryId& DataId : AllIds)
    {
        WeaponIDs.Add(DataId.ItemName);
    }
    
    return WeaponIDs;
}

TArray<FName> UWeaponDataRegistrySubsystem::GetWeaponsByType(FGameplayTag WeaponTypeTag)
{
    TArray<FName> FilteredIDs;
    TArray<FName> AllIDs = GetAllWeaponIDs();
    
    for (FName WeaponID : AllIDs)
    {
        FWeaponRegistryData WeaponData;
        if (GetWeaponData(WeaponID, WeaponData))
        {
            if (WeaponData.WeaponType.MatchesTag(WeaponTypeTag))
            {
                FilteredIDs.Add(WeaponID);
            }
        }
    }
    
    return FilteredIDs;
}
```

#### 7.3.4 在 UI 中使用 Data Registry

```cpp
// WeaponSelectionWidget.h

UCLASS()
class UWeaponSelectionWidget : public UUserWidget
{
    GENERATED_BODY()

protected:
    virtual void NativeConstruct() override;
    
    // 加载武器列表
    UFUNCTION()
    void LoadWeaponList();
    
    // 武器数据加载完成回调
    UFUNCTION()
    void OnWeaponDataLoaded(FDataRegistryItemAcquiredCallbackData CallbackData);
    
    // 在 UI 中显示武器
    UFUNCTION(BlueprintImplementableEvent, Category = "UI")
    void AddWeaponToList(
        FName WeaponID,
        FText DisplayName,
        float Damage,
        float FireRate,
        UTexture2D* Icon
    );

protected:
    UPROPERTY()
    TArray<FName> PendingWeaponIDs;
    
    UPROPERTY()
    int32 LoadedCount;
};
```

```cpp
// WeaponSelectionWidget.cpp

void UWeaponSelectionWidget::NativeConstruct()
{
    Super::NativeConstruct();
    
    LoadWeaponList();
}

void UWeaponSelectionWidget::LoadWeaponList()
{
    UWeaponDataRegistrySubsystem* WeaponRegistry = 
        GetGameInstance()->GetSubsystem<UWeaponDataRegistrySubsystem>();
    
    if (!WeaponRegistry)
    {
        return;
    }
    
    // 获取所有武器ID
    PendingWeaponIDs = WeaponRegistry->GetAllWeaponIDs();
    LoadedCount = 0;
    
    // 批量异步加载
    WeaponRegistry->LoadWeaponDataBatch(
        PendingWeaponIDs,
        FDataRegistryItemAcquiredCallback::CreateUObject(
            this,
            &ThisClass::OnWeaponDataLoaded
        )
    );
}

void UWeaponSelectionWidget::OnWeaponDataLoaded(
    FDataRegistryItemAcquiredCallbackData CallbackData
)
{
    if (CallbackData.ItemStatus != EDataRegistryAcquireStatus::Success)
    {
        return;
    }
    
    // 获取武器数据
    const FWeaponRegistryData* WeaponData = CallbackData.GetItem<FWeaponRegistryData>();
    
    if (!WeaponData)
    {
        return;
    }
    
    // 同步加载图标（如果已在内存中）
    UTexture2D* Icon = WeaponData->Icon.LoadSynchronous();
    
    // 添加到 UI 列表
    AddWeaponToList(
        CallbackData.ItemId.ItemName,
        WeaponData->DisplayName,
        WeaponData->Damage,
        WeaponData->FireRate,
        Icon
    );
    
    LoadedCount++;
    
    // 检查是否全部加载完成
    if (LoadedCount >= PendingWeaponIDs.Num())
    {
        UE_LOG(LogTemp, Log, TEXT("所有武器数据加载完成"));
    }
}
```

### 7.4 Data Registry 的高级特性

#### 7.4.1 数据分层与覆盖

**场景：**基础游戏有 10 把武器，Mod 想修改其中 2 把武器的属性，并添加 3 把新武器。

**解决方案：**

```
DR_Weapons (DataRegistry)
└─ Data Sources (按优先级从低到高):
   ├─ [Priority 0] DT_WeaponsBase (基础武器数据)
   └─ [Priority 100] DT_WeaponsMod (Mod 武器数据)
```

**DT_WeaponsBase：**

| Row Name | DisplayName | Damage |
|----------|-------------|--------|
| Rifle_AK47 | AK-47 | 35.0 |
| Rifle_M4A1 | M4A1 | 30.0 |

**DT_WeaponsMod：**

| Row Name | DisplayName | Damage |
|----------|-------------|--------|
| Rifle_AK47 | AK-47 强化版 | **50.0** (覆盖基础值) |
| Rifle_Custom | 自定义步枪 | 40.0 (新武器) |

**查询结果：**
- `Rifle_AK47`：优先使用 Mod 数据（Damage = 50.0）
- `Rifle_M4A1`：使用基础数据（Damage = 30.0）
- `Rifle_Custom`：使用 Mod 数据（新武器）

#### 7.4.2 运行时数据修改（Meta Data）

Data Registry 支持在运行时附加 **Meta Data**，实现临时数据覆盖：

```cpp
void ApplyTemporaryWeaponBuff(FName WeaponID, float DamageMultiplier)
{
    UDataRegistrySubsystem* DataRegistrySubsystem = 
        UDataRegistrySubsystem::Get();
    
    FDataRegistryId DataId(FDataRegistryType(FName("DR_Weapons")), WeaponID);
    
    // 创建 Meta Data
    FWeaponRegistryData MetaData;
    
    // 获取原始数据
    const FWeaponRegistryData* OriginalData = 
        DataRegistrySubsystem->GetCachedItem<FWeaponRegistryData>(DataId);
    
    if (OriginalData)
    {
        MetaData = *OriginalData;
        MetaData.Damage *= DamageMultiplier; // 临时提升伤害
    }
    
    // 附加 Meta Data（覆盖原始数据，但不修改 DataTable）
    DataRegistrySubsystem->SetCachedItemMeta(
        DataId,
        FInstancedStruct::Make(MetaData)
    );
}

void RemoveTemporaryWeaponBuff(FName WeaponID)
{
    UDataRegistrySubsystem* DataRegistrySubsystem = 
        UDataRegistrySubsystem::Get();
    
    FDataRegistryId DataId(FDataRegistryType(FName("DR_Weapons")), WeaponID);
    
    // 移除 Meta Data，恢复原始数据
    DataRegistrySubsystem->ClearCachedItemMeta(DataId);
}
```

**应用场景：**
- 临时 Buff/Debuff
- 活动期间属性加成
- 玩家自定义配置

---

## 8. 配置管理最佳实践

### 8.1 配置文件组织结构

推荐的项目配置目录结构：

```
Content/
└─ Data/
   ├─ Core/                      # 核心配置
   │  ├─ Experiences/             # 体验定义
   │  │  ├─ DA_Experience_Default
   │  │  ├─ DA_Experience_TDM
   │  │  └─ DA_Experience_CTF
   │  ├─ PawnData/                # 角色数据
   │  │  ├─ DA_Pawn_Hero
   │  │  └─ DA_Pawn_Bot
   │  └─ AbilitySets/             # 能力集
   │     ├─ AbilitySet_Hero
   │     └─ AbilitySet_Common
   │
   ├─ Items/                      # 物品配置
   │  ├─ Weapons/                 # 武器
   │  │  ├─ Rifles/
   │  │  │  ├─ DA_Item_AK47
   │  │  │  └─ DA_Equipment_AK47
   │  │  ├─ Shotguns/
   │  │  └─ Pistols/
   │  ├─ Attachments/             # 配件
   │  └─ Consumables/             # 消耗品
   │
   ├─ Registry/                   # Data Registry
   │  ├─ DR_Weapons
   │  ├─ DR_Attachments
   │  └─ DataTables/
   │     ├─ DT_WeaponsBase
   │     └─ DT_AttachmentsBase
   │
   └─ Tags/                       # Gameplay Tags
      └─ DT_GameplayTags
```

### 8.2 命名规范

#### 8.2.1 Data Asset 命名

| 类型 | 前缀 | 示例 |
|------|------|------|
| ExperienceDefinition | `DA_Experience_` | `DA_Experience_TDM` |
| PawnData | `DA_Pawn_` | `DA_Pawn_Hero_Shooter` |
| AbilitySet | `AbilitySet_` | `AbilitySet_Rifle` |
| EquipmentDefinition | `DA_Equipment_` | `DA_Equipment_AK47` |
| InventoryItemDefinition | `DA_Item_` | `DA_Item_AK47` |
| InputConfig | `InputData_` | `InputData_Hero` |

#### 8.2.2 Gameplay Tag 命名

遵循层级化命名：

```
Ability.Type.Action.Jump        # 跳跃能力
Ability.Type.Action.Dash        # 冲刺能力
InputTag.Weapon.Fire            # 开火输入
InputTag.Weapon.Reload          # 装弹输入
Status.Death                    # 死亡状态
Status.Death.Dying              # 濒死状态
GameplayCue.Weapon.Rifle.Fire   # 步枪射击特效
```

### 8.3 版本控制策略

#### 8.3.1 配置文件的 Git 管理

**.gitattributes 配置：**

```gitattributes
# Unreal Engine 资产文件使用 LFS
*.uasset filter=lfs diff=lfs merge=lfs -text
*.umap filter=lfs diff=lfs merge=lfs -text

# 配置文件使用文本模式
*.ini text
*.json text
```

**.gitignore 配置：**

```gitignore
# 不提交编译和缓存文件
Binaries/
Build/
Intermediate/
Saved/

# 不提交用户配置
*.sln
*.suo
*.user
.vs/
```

#### 8.3.2 配置版本管理

**方案一：使用 Git Submodule 管理共享配置**

```bash
# 主项目
MyGame/
├─ Content/
└─ Config/

# 共享配置仓库（Submodule）
MyGame_SharedConfig/
├─ DataAssets/
├─ DataTables/
└─ DefaultGameplayTags.ini

# 添加 Submodule
git submodule add https://github.com/MyTeam/MyGame_SharedConfig.git Content/SharedData
```

**方案二：使用分支管理不同版本配置**

```
main                  # 主分支（生产环境配置）
├─ dev                # 开发分支（测试配置）
├─ feature/pvp        # PVP 功能分支（PVP 配置）
└─ config/balance     # 平衡性调整分支（数值调整）
```

### 8.4 策划友好的配置工具

#### 8.4.1 蓝图编辑器扩展

创建自定义编辑器工具，让策划更方便地编辑配置：

```cpp
// WeaponEditorUtility.h

#pragma once

#include "EditorUtilityWidget.h"
#include "WeaponEditorUtility.generated.h"

/**
 * 武器编辑工具
 * 提供可视化的武器配置界面
 */
UCLASS()
class UWeaponEditorUtility : public UEditorUtilityWidget
{
    GENERATED_BODY()

public:
    // 创建新武器配置
    UFUNCTION(BlueprintCallable, CallInEditor, Category = "Weapon Editor")
    void CreateNewWeapon(
        FString WeaponName,
        EWeaponType WeaponType
    );
    
    // 批量调整武器伤害
    UFUNCTION(BlueprintCallable, CallInEditor, Category = "Weapon Editor")
    void BatchAdjustDamage(
        FGameplayTag WeaponTypeTag,
        float DamageMultiplier
    );
    
    // 导出武器数据到 Excel
    UFUNCTION(BlueprintCallable, CallInEditor, Category = "Weapon Editor")
    void ExportWeaponsToExcel(FString OutputPath);
    
    // 从 Excel 导入武器数据
    UFUNCTION(BlueprintCallable, CallInEditor, Category = "Weapon Editor")
    void ImportWeaponsFromExcel(FString InputPath);
};
```

#### 8.4.2 配置验证工具

自动检查配置的合法性：

```cpp
// ConfigValidationCommandlet.h

#pragma once

#include "Commandlets/Commandlet.h"
#include "ConfigValidationCommandlet.generated.h"

/**
 * 配置验证命令行工具
 * 用于 CI/CD 流程中自动检查配置错误
 */
UCLASS()
class UConfigValidationCommandlet : public UCommandlet
{
    GENERATED_BODY()

public:
    virtual int32 Main(const FString& Params) override;

protected:
    // 验证所有 Experience Definitions
    void ValidateExperiences();
    
    // 验证武器配置
    void ValidateWeapons();
    
    // 验证 Gameplay Tags 引用
    void ValidateGameplayTags();
    
    // 输出错误报告
    void GenerateReport();

protected:
    TArray<FString> Errors;
    TArray<FString> Warnings;
};
```

```cpp
// ConfigValidationCommandlet.cpp

int32 UConfigValidationCommandlet::Main(const FString& Params)
{
    UE_LOG(LogTemp, Log, TEXT("开始配置验证..."));
    
    ValidateExperiences();
    ValidateWeapons();
    ValidateGameplayTags();
    
    GenerateReport();
    
    return Errors.Num() > 0 ? 1 : 0;
}

void UConfigValidationCommandlet::ValidateWeapons()
{
    // 查找所有武器 ItemDefinition
    TArray<ULyraInventoryItemDefinition*> WeaponItems;
    
    // 遍历验证
    for (ULyraInventoryItemDefinition* Item : WeaponItems)
    {
        // 检查是否有 WeaponStats Fragment
        const UInventoryFragment_WeaponStats* WeaponStats = 
            Cast<UInventoryFragment_WeaponStats>(
                Item->FindFragmentByClass(UInventoryFragment_WeaponStats::StaticClass())
            );
        
        if (!WeaponStats)
        {
            Errors.Add(FString::Printf(
                TEXT("武器 %s 缺少 WeaponStats Fragment"),
                *Item->GetName()
            ));
            continue;
        }
        
        // 检查伤害值是否合理
        if (WeaponStats->BaseDamage <= 0.0f)
        {
            Errors.Add(FString::Printf(
                TEXT("武器 %s 的伤害值无效: %f"),
                *Item->GetName(),
                WeaponStats->BaseDamage
            ));
        }
        
        // 检查射速是否合理
        if (WeaponStats->FireRate <= 0.0f || WeaponStats->FireRate > 10000.0f)
        {
            Warnings.Add(FString::Printf(
                TEXT("武器 %s 的射速值异常: %f"),
                *Item->GetName(),
                WeaponStats->FireRate
            ));
        }
    }
}

void UConfigValidationCommandlet::GenerateReport()
{
    UE_LOG(LogTemp, Warning, TEXT("===== 配置验证报告 ====="));
    UE_LOG(LogTemp, Warning, TEXT("错误数: %d"), Errors.Num());
    UE_LOG(LogTemp, Warning, TEXT("警告数: %d"), Warnings.Num());
    
    if (Errors.Num() > 0)
    {
        UE_LOG(LogTemp, Error, TEXT("\n===== 错误列表 ====="));
        for (const FString& Error : Errors)
        {
            UE_LOG(LogTemp, Error, TEXT("  ❌ %s"), *Error);
        }
    }
    
    if (Warnings.Num() > 0)
    {
        UE_LOG(LogTemp, Warning, TEXT("\n===== 警告列表 ====="));
        for (const FString& Warning : Warnings)
        {
            UE_LOG(LogTemp, Warning, TEXT("  ⚠️ %s"), *Warning);
        }
    }
}
```

**使用方法：**

```bash
# 在命令行运行验证
UnrealEditor-Cmd.exe "C:/Projects/MyGame/MyGame.uproject" -run=ConfigValidation
```

---

## 9. 性能优化与资源管理

### 9.1 异步加载策略

#### 9.1.1 分批加载资产

```cpp
// ExperienceAssetLoader.h

UCLASS()
class UExperienceAssetLoader : public UObject
{
    GENERATED_BODY()

public:
    // 分批加载 Experience 资产
    void LoadExperienceAssetsBatched(
        const ULyraExperienceDefinition* Experience,
        FStreamableDelegate OnAllLoaded
    );

protected:
    void LoadBatch(int32 BatchIndex);
    void OnBatchLoaded();

protected:
    TArray<TSoftObjectPtr<UObject>> PendingAssets;
    int32 CurrentBatchIndex;
    int32 BatchSize;
    FStreamableDelegate OnCompleteDelegate;
    
    TArray<TSharedPtr<FStreamableHandle>> ActiveHandles;
};
```

```cpp
// ExperienceAssetLoader.cpp

void UExperienceAssetLoader::LoadExperienceAssetsBatched(
    const ULyraExperienceDefinition* Experience,
    FStreamableDelegate OnAllLoaded
)
{
    // 收集所有需要加载的资产
    PendingAssets.Empty();
    
    // 从 Experience 中提取所有 TSoftObjectPtr
    // ...
    
    OnCompleteDelegate = OnAllLoaded;
    CurrentBatchIndex = 0;
    BatchSize = 10; // 每批加载 10 个资产
    
    LoadBatch(0);
}

void UExperienceAssetLoader::LoadBatch(int32 BatchIndex)
{
    int32 StartIndex = BatchIndex * BatchSize;
    int32 EndIndex = FMath::Min(StartIndex + BatchSize, PendingAssets.Num());
    
    if (StartIndex >= PendingAssets.Num())
    {
        // 所有批次加载完成
        OnCompleteDelegate.ExecuteIfBound();
        return;
    }
    
    // 准备本批次的资产列表
    TArray<FSoftObjectPath> BatchAssets;
    for (int32 i = StartIndex; i < EndIndex; ++i)
    {
        BatchAssets.Add(PendingAssets[i].ToSoftObjectPath());
    }
    
    // 异步加载本批次
    UAssetManager& AssetManager = UAssetManager::Get();
    TSharedPtr<FStreamableHandle> Handle = AssetManager.LoadAssetList(
        BatchAssets,
        FStreamableDelegate::CreateUObject(this, &ThisClass::OnBatchLoaded)
    );
    
    ActiveHandles.Add(Handle);
}

void UExperienceAssetLoader::OnBatchLoaded()
{
    CurrentBatchIndex++;
    LoadBatch(CurrentBatchIndex);
}
```

#### 9.1.2 优先级加载

```cpp
// 设置资产加载优先级
void LoadWeaponWithPriority(const ULyraInventoryItemDefinition* ItemDef)
{
    UAssetManager& AssetManager = UAssetManager::Get();
    
    // 高优先级加载武器网格（玩家立即可见）
    if (const UInventoryFragment_EquippableItem* EquipFragment = 
        Cast<UInventoryFragment_EquippableItem>(
            ItemDef->FindFragmentByClass(UInventoryFragment_EquippableItem::StaticClass())
        ))
    {
        const ULyraEquipmentDefinition* EquipDef = EquipFragment->EquipmentDefinition;
        
        for (const FLyraEquipmentActorToSpawn& ActorToSpawn : EquipDef->ActorsToSpawn)
        {
            AssetManager.LoadAssetAsync(
                ActorToSpawn.ActorToSpawn.ToSoftObjectPath(),
                FStreamableDelegate(),
                10 // 高优先级
            );
        }
    }
    
    // 低优先级加载音效和粒子特效（延迟可接受）
    // ...
}
```

### 9.2 内存管理

#### 9.2.1 及时释放不用的资产

```cpp
// Experience 切换时释放旧资产
void ULyraExperienceManagerComponent::UnloadExperience()
{
    UAssetManager& AssetManager = UAssetManager::Get();
    
    // 释放之前加载的资产
    for (TSharedPtr<FStreamableHandle>& Handle : LoadedAssetHandles)
    {
        if (Handle.IsValid())
        {
            Handle->ReleaseHandle();
        }
    }
    LoadedAssetHandles.Empty();
    
    // 手动触发垃圾回收（可选）
    GetWorld()->ForceGarbageCollection(true);
}
```

#### 9.2.2 资产卸载策略

```cpp
// 配置 Asset Manager 的资产卸载规则
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetRules=(PrimaryAssetType="WeaponData",Rules=(Priority=5,ChunkId=-1,bApplyRecursively=True,CookRule=AlwaysCook),UnloadBehavior=Unload)
```

**UnloadBehavior 选项：**
- `DoNotUnload`：永不卸载（适用于核心资产）
- `Unload`：不再使用时卸载（适用于临时资产）
- `UnloadAfterDelay`：延迟卸载（避免频繁加载卸载）

### 9.3 打包优化

#### 9.3.1 配置 Asset Bundle

在 PrimaryDataAsset 中定义打包规则：

```cpp
// LyraExperienceDefinition.cpp

#if WITH_EDITORONLY_DATA
void ULyraExperienceDefinition::UpdateAssetBundleData()
{
    Super::UpdateAssetBundleData();
    
    // 定义 AssetBundle 规则
    FAssetBundleData BundleData;
    
    // 收集 PawnData 到 "Pawns" Bundle
    if (DefaultPawnData)
    {
        BundleData.AddBundleAsset(
            FName("Pawns"),
            DefaultPawnData.ToSoftObjectPath()
        );
    }
    
    // 收集 AbilitySets 到 "Abilities" Bundle
    for (const ULyraAbilitySet* AbilitySet : AbilitySets)
    {
        if (AbilitySet)
        {
            BundleData.AddBundleAsset(
                FName("Abilities"),
                AbilitySet.ToSoftObjectPath()
            );
        }
    }
    
    // 应用 Bundle 数据
    SetAssetBundleData(BundleData);
}
#endif
```

#### 9.3.2 多 Chunk 打包策略

在 `DefaultGame.ini` 中配置：

```ini
[/Script/UnrealEd.ProjectPackagingSettings]
+MapsToCook=(FilePath="/Game/Maps/MainMenu")
+MapsToCook=(FilePath="/Game/Maps/Gameplay")

# 将不同内容打包到不同 Chunk
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="ExperienceDef",AssetBaseClass=/Script/LyraGame.LyraExperienceDefinition,bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/Experiences")),Rules=(Priority=10,ChunkId=1,bApplyRecursively=True,CookRule=AlwaysCook))

+PrimaryAssetTypesToScan=(PrimaryAssetType="WeaponData",AssetBaseClass=/Script/MyGame.MyWeaponDefinition,bHasBlueprintClasses=True,bIsEditorOnly=False,Directories=((Path="/Game/Weapons")),Rules=(Priority=5,ChunkId=2,bApplyRecursively=True,CookRule=DevelopmentAlwaysProductionUnknown))
```

**Chunk 划分建议：**
- **Chunk 0**：引擎核心 + 主菜单
- **Chunk 1**：游戏核心系统（Experience、PawnData）
- **Chunk 2**：武器和装备（可热更新）
- **Chunk 3**：地图和场景（按需加载）

---

## 10. 常见问题与解决方案

### 10.1 配置未生效问题

**问题描述：**修改了 Data Asset，但运行时没有生效。

**可能原因：**
1. 资产未保存
2. 旧版本资产被缓存
3. 引用路径错误
4. 配置被其他地方覆盖

**解决步骤：**

```cpp
// 1. 检查资产是否已保存
// 在编辑器中：File → Save All

// 2. 清除派生数据缓存
// 关闭编辑器，删除 Saved/Intermediate 文件夹

// 3. 验证引用路径
void DebugPrintAssetPath(const ULyraExperienceDefinition* Experience)
{
    if (!Experience)
    {
        UE_LOG(LogTemp, Error, TEXT("Experience is nullptr!"));
        return;
    }
    
    UE_LOG(LogTemp, Log, TEXT("Experience Path: %s"), *Experience->GetPathName());
    UE_LOG(LogTemp, Log, TEXT("Default PawnData: %s"), 
        Experience->DefaultPawnData ? *Experience->DefaultPawnData->GetPathName() : TEXT("None"));
}

// 4. 检查 Asset Manager 是否正确加载
void VerifyAssetManagerSetup()
{
    UAssetManager& AssetManager = UAssetManager::Get();
    
    FPrimaryAssetType Type(FName("ExperienceDef"));
    TArray<FPrimaryAssetId> AssetIds;
    AssetManager.GetPrimaryAssetIdList(Type, AssetIds);
    
    UE_LOG(LogTemp, Log, TEXT("Registered Experience Count: %d"), AssetIds.Num());
    
    for (const FPrimaryAssetId& AssetId : AssetIds)
    {
        UE_LOG(LogTemp, Log, TEXT("  - %s"), *AssetId.ToString());
    }
}
```

### 10.2 加载卡顿问题

**问题描述：**切换 Experience 或装备武器时，游戏卡顿明显。

**原因分析：**
- 同步加载大量资产
- 资产未提前预加载
- 没有使用资产分批加载

**优化方案：**

```cpp
// 方案一：提前预加载常用资产
void PreloadCommonAssets()
{
    UAssetManager& AssetManager = UAssetManager::Get();
    
    // 预加载所有武器的缩略图和图标
    TArray<FPrimaryAssetId> WeaponIds;
    AssetManager.GetPrimaryAssetIdList(FPrimaryAssetType("WeaponData"), WeaponIds);
    
    for (const FPrimaryAssetId& WeaponId : WeaponIds)
    {
        // 只加载轻量级资产（图标、数据）
        AssetManager.LoadPrimaryAsset(
            WeaponId,
            TArray<FName>{"UI"}, // 只加载 UI Bundle
            FStreamableDelegate(),
            -1 // 低优先级，后台加载
        );
    }
}

// 方案二：使用流式加载显示加载界面
void LoadExperienceWithProgressBar(const ULyraExperienceDefinition* Experience)
{
    // 显示加载界面
    UMyLoadingScreenWidget* LoadingWidget = CreateWidget<UMyLoadingScreenWidget>(...);
    LoadingWidget->AddToViewport();
    
    // 收集所有资产
    TArray<TSoftObjectPtr<UObject>> AssetsToLoad;
    // ...
    
    // 逐个加载，更新进度条
    int32 TotalCount = AssetsToLoad.Num();
    int32 LoadedCount = 0;
    
    for (const TSoftObjectPtr<UObject>& Asset : AssetsToLoad)
    {
        Asset.LoadSynchronous(); // 注意：在这里同步加载是因为有加载界面
        
        LoadedCount++;
        float Progress = (float)LoadedCount / TotalCount;
        LoadingWidget->SetProgress(Progress);
        
        // 每加载几个资产就刷新一次渲染，避免界面卡住
        if (LoadedCount % 5 == 0)
        {
            FSlateApplication::Get().PumpMessages();
            FSlateApplication::Get().Tick();
        }
    }
    
    LoadingWidget->RemoveFromParent();
}
```

### 10.3 网络同步问题

**问题描述：**Data Asset 配置在服务器和客户端不一致。

**原因分析：**
- 客户端和服务器使用了不同版本的配置
- 配置修改后未重新打包
- 网络复制不正确

**解决方案：**

```cpp
// 方案一：版本校验
USTRUCT()
struct FConfigVersion
{
    GENERATED_BODY()
    
    UPROPERTY()
    int32 MajorVersion = 1;
    
    UPROPERTY()
    int32 MinorVersion = 0;
    
    UPROPERTY()
    FString ConfigHash; // 配置文件的 Hash 值
};

class AMyGameState : public AGameStateBase
{
    GENERATED_BODY()

public:
    UPROPERTY(Replicated)
    FConfigVersion ServerConfigVersion;
    
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(AMyGameState, ServerConfigVersion);
    }
    
    // 客户端检查版本
    void CheckConfigVersion()
    {
        FConfigVersion LocalVersion = LoadLocalConfigVersion();
        
        if (LocalVersion.MajorVersion != ServerConfigVersion.MajorVersion ||
            LocalVersion.ConfigHash != ServerConfigVersion.ConfigHash)
        {
            // 版本不匹配，提示玩家更新
            ShowUpdateRequiredDialog();
        }
    }
};

// 方案二：Data Asset 引用一致性检查
void VerifyAssetConsistency()
{
    if (!GetWorld()->IsNetMode(NM_Client))
    {
        return; // 仅客户端检查
    }
    
    // 对比服务器和本地的 AssetPath
    FString ServerAssetPath = GetServerAssetPath();
    FString LocalAssetPath = GetLocalAssetPath();
    
    if (ServerAssetPath != LocalAssetPath)
    {
        UE_LOG(LogTemp, Error, TEXT("Asset mismatch! Server: %s, Local: %s"),
            *ServerAssetPath, *LocalAssetPath);
    }
}
```

### 10.4 Gameplay Tags 找不到

**问题描述：**代码中引用的 Gameplay Tag 在运行时返回空。

**原因分析：**
- Tag 未在配置文件中定义
- Tag 拼写错误
- 配置文件未被加载

**解决方案：**

```cpp
// 方案一：使用原生标签避免拼写错误
// LyraGameplayTags.h
UE_DECLARE_GAMEPLAY_TAG_EXTERN(InputTag_Weapon_Fire);

// LyraGameplayTags.cpp
UE_DEFINE_GAMEPLAY_TAG(InputTag_Weapon_Fire, "InputTag.Weapon.Fire");

// 使用
FGameplayTag FireTag = LyraGameplayTags::InputTag_Weapon_Fire;
check(FireTag.IsValid()); // 编译期保证有效性

// 方案二：运行时验证 Tag 是否存在
FGameplayTag RequestTagSafely(const FString& TagString)
{
    FGameplayTag Tag = FGameplayTag::RequestGameplayTag(FName(*TagString), false);
    
    if (!Tag.IsValid())
    {
        UE_LOG(LogTemp, Error, TEXT("Gameplay Tag not found: %s"), *TagString);
        
        // 尝试模糊匹配
        FGameplayTag FuzzyMatch = LyraGameplayTags::FindTagByString(TagString, true);
        if (FuzzyMatch.IsValid())
        {
            UE_LOG(LogTemp, Warning, TEXT("Did you mean: %s ?"), *FuzzyMatch.ToString());
        }
    }
    
    return Tag;
}

// 方案三：检查配置文件是否加载
void VerifyGameplayTagsLoaded()
{
    UGameplayTagsManager& TagManager = UGameplayTagsManager::Get();
    
    FGameplayTagContainer AllTags;
    TagManager.RequestAllGameplayTags(AllTags, true);
    
    UE_LOG(LogTemp, Log, TEXT("Total Gameplay Tags: %d"), AllTags.Num());
    
    if (AllTags.Num() == 0)
    {
        UE_LOG(LogTemp, Error, TEXT("No Gameplay Tags loaded! Check DefaultGameplayTags.ini"));
    }
}
```

### 10.5 Fragment 未生效

**问题描述：**添加了 Fragment，但查询时返回 nullptr。

**原因分析：**
- Fragment 未正确添加到 ItemDefinition
- Fragment 类型转换错误
- Fragment 数组为空

**调试方法：**

```cpp
void DebugPrintFragments(const ULyraInventoryItemDefinition* ItemDef)
{
    if (!ItemDef)
    {
        UE_LOG(LogTemp, Error, TEXT("ItemDef is nullptr!"));
        return;
    }
    
    UE_LOG(LogTemp, Log, TEXT("Item: %s"), *ItemDef->GetName());
    UE_LOG(LogTemp, Log, TEXT("Fragment Count: %d"), ItemDef->Fragments.Num());
    
    for (int32 i = 0; i < ItemDef->Fragments.Num(); ++i)
    {
        const ULyraInventoryItemFragment* Fragment = ItemDef->Fragments[i];
        if (Fragment)
        {
            UE_LOG(LogTemp, Log, TEXT("  [%d] %s"), i, *Fragment->GetClass()->GetName());
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("  [%d] nullptr Fragment!"), i);
        }
    }
    
    // 测试查找特定 Fragment
    const UInventoryFragment_WeaponStats* WeaponStats = 
        Cast<UInventoryFragment_WeaponStats>(
            ItemDef->FindFragmentByClass(UInventoryFragment_WeaponStats::StaticClass())
        );
    
    if (WeaponStats)
    {
        UE_LOG(LogTemp, Log, TEXT("✅ WeaponStats Fragment found! Damage: %f"), WeaponStats->BaseDamage);
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("❌ WeaponStats Fragment not found!"));
    }
}
```

---

## 总结

本章深入讲解了 Lyra 的数据驱动设计理念和实践：

### 核心要点

1. **数据驱动的优势**：
   - 代码与数据分离，策划和程序员并行工作
   - 快速迭代，无需编译即可调整配置
   - 支持热更新和 Mod 扩展

2. **Unreal Engine 数据资产体系**：
   - `UObject` → `UDataAsset` → `UPrimaryDataAsset`
   - Asset Manager 负责资产的异步加载和打包管理

3. **Lyra Data Assets 体系**：
   - 分层设计：Experience → Pawn → Ability → Equipment → Inventory
   - Fragment 模式实现模块化配置
   - 组合优于继承

4. **Gameplay Tags 系统**：
   - 层级化标签管理
   - 类型安全的标签查询
   - 高性能的标签匹配

5. **实战技巧**：
   - 使用 Data Registry 管理海量数据
   - 异步加载策略避免卡顿
   - 版本控制和配置验证
   - 策划友好的编辑工具

### 推荐的学习路径

1. **理解基础**：先掌握 UDataAsset 和 Gameplay Tags 的基本用法
2. **分析 Lyra**：阅读 Lyra 源码，理解其数据资产的设计思路
3. **动手实践**：创建自己的武器/角色配置系统
4. **进阶优化**：学习 Data Registry 和资产加载优化
5. **工具开发**：为策划开发可视化配置工具

### 下一步

- **第6章 - GAS 入门**：深入学习 Gameplay Ability System，理解能力的运作机制
- **第7章 - GAS 进阶**：掌握 Attributes、Effects 和 Tags 的高级应用
- **第10章 - 装备与武器系统**：结合本章内容实现完整的装备系统

---

**参考资料：**

- Unreal Engine 官方文档：[Data Assets](https://docs.unrealengine.com/5.3/en-US/data-assets-in-unreal-engine/)
- Unreal Engine 官方文档：[Gameplay Tags](https://docs.unrealengine.com/5.3/en-US/using-gameplay-tags-in-unreal-engine/)
- Unreal Engine 官方文档：[Asset Manager](https://docs.unrealengine.com/5.3/en-US/asset-management-in-unreal-engine/)
- Lyra 源码：`Samples/Games/Lyra/Source/LyraGame/`

---

© 2026 Lyra 系列教程 | [返回目录](../README.md) | [下一章：GAS 入门](../02-core-systems/06-gas-basics.md)
