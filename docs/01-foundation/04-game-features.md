# Game Features：插件化游戏内容

## 概述

Game Features 插件系统是 Unreal Engine 5 引入的革命性功能，而 Lyra 项目将其发挥到了极致。它不仅仅是一个插件管理系统，更是一种**模块化游戏开发范式**，让你能够：

- 🎮 将游戏内容打包成独立的、可热插拔的模块
- 🔄 动态加载和卸载游戏功能，无需重启
- 🧩 在运行时组合不同的游戏玩法
- 📦 实现真正的"插件即内容"架构
- 🚀 支持 DLC、实验性功能、多游戏模式共存

在 Lyra 中，ShooterCore、TopDownArena 等都是 Game Feature 插件，它们可以独立开发、测试、发布，然后通过 Experience 系统动态激活。

---

## 为什么需要 Game Features？

### 传统方式的问题

在传统 UE 开发中，新增一个游戏模式通常需要：

```cpp
// ❌ 传统方式：所有内容耦合在主项目中
// 1. 在主项目中创建新的 GameMode、Character、Weapon 等类
// 2. 在 GameInstance 或 GameMode 中硬编码逻辑
// 3. 所有内容都编译进主模块
// 4. 修改任何内容都需要重新编译整个项目
// 5. 不同模式的资源全部加载到内存

class AMyShooterGameMode : public AGameModeBase
{
    // 射击模式相关逻辑
};

class AMyRacingGameMode : public AGameModeBase
{
    // 赛车模式相关逻辑
};

// 结果：项目越来越臃肿，编译时间越来越长
```

**问题总结**：
- ❌ 高耦合：所有内容混在一起
- ❌ 低复用：无法在不同项目间共享
- ❌ 重编译：改一个模式，全项目重编译
- ❌ 内存浪费：不相关的内容也会被加载
- ❌ 团队协作困难：多人修改同一份代码易冲突

### Game Features 的解决方案

```
Lyra 项目结构：
LyraGame (核心框架)
├── Experience System (游戏模式定义)
├── GAS (通用能力系统)
├── Modular Gameplay (组件化角色)
└── ...

Plugins/GameFeatures/ (可插拔内容)
├── ShooterCore/         ✅ 独立开发
│   ├── Weapons          ✅ 按需加载
│   ├── Abilities        ✅ 版本控制
│   └── UI               ✅ 团队分工
├── TopDownArena/        ✅ 并行开发
├── ShooterMaps/         ✅ 动态激活
└── CustomDLC/           ✅ 后期扩展
```

**优势**：
- ✅ **解耦**：每个插件独立，互不干扰
- ✅ **复用**：可在不同项目间共享
- ✅ **增量编译**：只编译修改的插件
- ✅ **按需加载**：只加载当前需要的内容
- ✅ **团队协作**：多团队并行开发不同功能
- ✅ **实验迭代**：可快速开启/关闭实验性功能

---

## Game Feature 插件生命周期

### 状态机流转

Game Feature 插件有一个严格的状态机，理解它是掌握系统的关键：

```
[Uninitialized] (插件未知)
      ↓
[StatusKnown] (插件已注册，但未下载)
      ↓
[Registered] (插件元数据已读取)
      ↓
[Downloading] (正在下载插件内容，可选)
      ↓
[Installed] (插件内容已安装到本地)
      ↓
[WaitingForDependencies] (等待依赖插件加载)
      ↓
[Registering] (注册插件的 Asset Registry)
      ↓
[Loaded] (插件已加载，但未激活)
      ↓
[Activating] (正在激活，执行 GameFeatureActions)
      ↓
[Active] ✅ (插件完全激活，功能可用)
      ↓
[Deactivating] (正在停用)
      ↓
[Unloading] (正在卸载)
      ↓
[Terminating] (清理资源)
      ↓
(回到 Registered 或 StatusKnown)
```

**关键状态说明**：

| 状态 | 说明 | 可用操作 |
|------|------|---------|
| **Registered** | 插件元数据（.uplugin）已读取 | 查询插件信息 |
| **Loaded** | 插件的模块和资源已加载到内存 | 访问资源，但功能未激活 |
| **Active** | GameFeatureActions 已执行 | 完整功能可用 |

### 生命周期代码示例

```cpp
// 在 LyraExperienceManager 中加载 Game Feature
void ULyraExperienceManagerComponent::OnExperienceLoadComplete()
{
    // 1. 获取 Experience Definition
    const ULyraExperienceDefinition* Experience = GetCurrentExperience();
    
    // 2. 遍历需要启用的 Game Features
    for (const FString& PluginURL : Experience->GameFeaturesToEnable)
    {
        // 3. 请求加载并激活插件
        UGameFeaturesSubsystem& GameFeaturesSubsystem = 
            UGameFeaturesSubsystem::Get();
        
        // 从 Registered → Loaded → Active
        GameFeaturesSubsystem.LoadAndActivateGameFeaturePlugin(
            PluginURL,
            FGameFeaturePluginLoadComplete::CreateUObject(
                this, 
                &ThisClass::OnGameFeaturePluginLoadComplete
            )
        );
    }
}

// 4. 激活完成回调
void ULyraExperienceManagerComponent::OnGameFeaturePluginLoadComplete(
    const UE::GameFeatures::FResult& Result)
{
    if (Result.HasValue())
    {
        UE_LOG(LogLyraExperience, Log, 
            TEXT("Game Feature activated: %s"), 
            *Result.GetValue());
    }
}
```

---

## ShooterCore 插件深度剖析

### 插件结构概览

```
ShooterCore/
├── ShooterCore.uplugin          # 插件元数据
├── Content/                     # 资源内容
│   ├── Abilities/               # GAS 能力
│   ├── Weapons/                 # 武器配置
│   ├── Input/                   # 输入映射
│   └── UI/                      # 界面资源
└── Source/
    └── ShooterCoreRuntime/
        ├── Public/
        │   ├── Input/           # 瞄准辅助组件
        │   ├── Accolades/       # 荣誉系统（击杀连击等）
        │   └── MessageProcessors/ # 消息处理器
        └── Private/
            └── ShooterCoreRuntimeModule.cpp
```

### 插件描述文件 (.uplugin)

```json
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "1.0",
    "FriendlyName": "ShooterCore",
    "Description": "Gameplay systems for Shooter Game",
    "Category": "Game Features",
    "CanContainContent": true,          // ✅ 可包含资源
    "ExplicitlyLoaded": true,           // ✅ 显式加载（非自动）
    "EnabledByDefault": false,          // ❌ 默认不启用
    "BuiltInInitialFeatureState": "Registered", // 初始状态
    
    "Modules": [
        {
            "Name": "ShooterCoreRuntime",
            "Type": "Runtime",
            "LoadingPhase": "Default"
        }
    ],
    
    "Plugins": [                        // 依赖的其他插件
        {"Name": "GameplayAbilities", "Enabled": true},
        {"Name": "ModularGameplay", "Enabled": true},
        {"Name": "EnhancedInput", "Enabled": true},
        {"Name": "CommonUI", "Enabled": true}
    ]
}
```

**关键字段解析**：

- **BuiltInInitialFeatureState**: `Registered` 表示引擎启动时会注册插件，但不会加载内容
- **ExplicitlyLoaded**: `true` 表示必须通过代码显式加载（Experience 系统控制）
- **EnabledByDefault**: `false` 表示不会像传统插件那样自动启用
- **CanContainContent**: `true` 允许包含蓝图、资源等内容

### 与传统插件的区别

| 特性 | 传统插件 | Game Feature 插件 |
|------|---------|------------------|
| 启用方式 | 手动勾选 `.uproject` | 运行时动态加载 |
| 加载时机 | 引擎启动 | Experience 激活时 |
| 卸载能力 | 需重启 | 运行时卸载 |
| 依赖管理 | 静态 | 动态解析 |
| 内容隔离 | 中等 | 完全隔离 |

---

## GameFeatureAction：插件的行为逻辑

### 什么是 GameFeatureAction？

GameFeatureAction 是**插件激活时执行的操作**，类似于"安装脚本"。每个 Action 负责一个具体功能：

```cpp
// 基类定义
UCLASS(Abstract)
class UGameFeatureAction : public UObject
{
    GENERATED_BODY()

public:
    // 插件激活时调用
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) {}
    
    // 插件停用时调用
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) {}
    
    // 插件注册时调用
    virtual void OnGameFeatureRegistering() {}
    
    // 插件卸载时调用
    virtual void OnGameFeatureUnregistering() {}
};
```

### Lyra 内置的 GameFeatureActions

| Action 类型 | 功能 | 使用场景 |
|------------|------|---------|
| **AddAbilities** | 为指定 Actor 添加 GAS 能力 | 给角色添加射击、跳跃等能力 |
| **AddInputBinding** | 绑定输入到能力 | 将鼠标左键绑定到开火能力 |
| **AddInputContextMapping** | 添加 Enhanced Input 上下文 | 加载射击模式的键位配置 |
| **AddWidget** | 动态添加 UI 组件 | 显示准星、弹药 UI |
| **AddGameplayCuePath** | 注册 Gameplay Cue 路径 | 添加特效资源路径 |
| **SplitscreenConfig** | 配置分屏设置 | 启用本地多人游戏 |
| **WorldActionBase** | 自定义世界相关操作 | 其他 Action 的基类 |

### GameFeatureAction_AddAbilities 源码分析

这是 Lyra 中最核心的 Action，负责动态添加 GAS 能力：

```cpp
// GameFeatureAction_AddAbilities.h
UCLASS(MinimalAPI, meta = (DisplayName = "Add Abilities"))
class UGameFeatureAction_AddAbilities final : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    // 配置要添加的能力
    UPROPERTY(EditAnywhere, Category="Abilities")
    TArray<FGameFeatureAbilitiesEntry> AbilitiesList;

private:
    // 追踪已添加的能力
    struct FActorExtensions
    {
        TArray<FGameplayAbilitySpecHandle> Abilities;  // 能力句柄
        TArray<UAttributeSet*> Attributes;             // 属性集
        TArray<FLyraAbilitySet_GrantedHandles> AbilitySetHandles;
    };

    TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;
};

// 能力条目定义
USTRUCT()
struct FGameFeatureAbilitiesEntry
{
    GENERATED_BODY()

    // 目标 Actor 类型（如 LyraCharacter）
    UPROPERTY(EditAnywhere, Category="Abilities")
    TSoftClassPtr<AActor> ActorClass;

    // 要添加的能力列表
    UPROPERTY(EditAnywhere, Category="Abilities")
    TArray<FLyraAbilityGrant> GrantedAbilities;

    // 要添加的属性集
    UPROPERTY(EditAnywhere, Category="Attributes")
    TArray<FLyraAttributeSetGrant> GrantedAttributes;

    // 能力集（批量添加能力的封装）
    UPROPERTY(EditAnywhere, Category="Attributes")
    TArray<TSoftObjectPtr<const ULyraAbilitySet>> GrantedAbilitySets;
};
```

**核心流程**：

```cpp
// GameFeatureAction_AddAbilities.cpp (简化版)
void UGameFeatureAction_AddAbilities::AddToWorld(
    const FWorldContext& WorldContext, 
    const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    UGameInstance* GameInstance = WorldContext.OwningGameInstance;
    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    // 遍历配置的所有能力条目
    for (int32 EntryIndex = 0; EntryIndex < AbilitiesList.Num(); ++EntryIndex)
    {
        const FGameFeatureAbilitiesEntry& Entry = AbilitiesList[EntryIndex];

        // 如果目标类型未加载，先加载
        if (!Entry.ActorClass.IsNull())
        {
            // 监听该类型的 Actor 实例化
            // 使用 ModularGameplay 的扩展点机制
            UGameFrameworkComponentManager* ComponentManager = 
                UGameFrameworkComponentManager::GetForWorld(World);

            TSharedPtr<FComponentRequestHandle> RequestHandle = 
                ComponentManager->AddExtensionHandler(
                    Entry.ActorClass.Get(),
                    UGameFrameworkComponentManager::FExtensionHandlerDelegate::
                        CreateUObject(this, 
                            &ThisClass::HandleActorExtension, 
                            EntryIndex, 
                            ChangeContext)
                );

            ActiveData.ComponentRequests.Add(RequestHandle);
        }
    }
}

// 当目标类型的 Actor 被创建时触发
void UGameFeatureAction_AddAbilities::HandleActorExtension(
    AActor* Actor, 
    FName EventName, 
    int32 EntryIndex, 
    FGameFeatureStateChangeContext ChangeContext)
{
    FPerContextData* ActiveData = ContextData.Find(ChangeContext);
    if (!ActiveData) return;

    const FGameFeatureAbilitiesEntry& Entry = AbilitiesList[EntryIndex];

    if (EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded ||
        EventName == UGameFrameworkComponentManager::NAME_GameActorReady)
    {
        // Actor 准备好，添加能力
        AddActorAbilities(Actor, Entry, *ActiveData);
    }
    else if (EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved ||
             EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved)
    {
        // Actor 被销毁，移除能力
        RemoveActorAbilities(Actor, *ActiveData);
    }
}

// 实际添加能力的逻辑
void UGameFeatureAction_AddAbilities::AddActorAbilities(
    AActor* Actor, 
    const FGameFeatureAbilitiesEntry& Entry, 
    FPerContextData& ActiveData)
{
    // 1. 获取或添加 AbilitySystemComponent
    UAbilitySystemComponent* ASC = 
        FindOrAddComponentForActor<UAbilitySystemComponent>(Actor, Entry, ActiveData);
    
    if (!ASC) return;

    FActorExtensions& ActorExtensions = ActiveData.ActiveExtensions.FindOrAdd(Actor);

    // 2. 添加 Attribute Sets
    for (const FLyraAttributeSetGrant& AttributeGrant : Entry.GrantedAttributes)
    {
        if (!AttributeGrant.AttributeSetType.IsNull())
        {
            TSubclassOf<UAttributeSet> SetType = 
                AttributeGrant.AttributeSetType.LoadSynchronous();
            UAttributeSet* NewSet = NewObject<UAttributeSet>(ASC, SetType);
            ASC->AddAttributeSetSubobject(NewSet);
            ActorExtensions.Attributes.Add(NewSet);
        }
    }

    // 3. 添加 Abilities
    for (const FLyraAbilityGrant& AbilityGrant : Entry.GrantedAbilities)
    {
        if (!AbilityGrant.AbilityType.IsNull())
        {
            TSubclassOf<UGameplayAbility> AbilityClass = 
                AbilityGrant.AbilityType.LoadSynchronous();
            
            FGameplayAbilitySpec AbilitySpec(AbilityClass, 1, INDEX_NONE, Actor);
            FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(AbilitySpec);
            ActorExtensions.Abilities.Add(Handle);
        }
    }

    // 4. 添加 Ability Sets（批量添加）
    for (const TSoftObjectPtr<const ULyraAbilitySet>& AbilitySetPtr : 
         Entry.GrantedAbilitySets)
    {
        if (const ULyraAbilitySet* AbilitySet = AbilitySetPtr.LoadSynchronous())
        {
            FLyraAbilitySet_GrantedHandles GrantedHandles;
            AbilitySet->GiveToAbilitySystem(ASC, &GrantedHandles, Actor);
            ActorExtensions.AbilitySetHandles.Add(GrantedHandles);
        }
    }
}
```

**关键技术点**：

1. **扩展点机制**：使用 `GameFrameworkComponentManager` 监听特定类型的 Actor 实例化，而不是硬编码
2. **延迟加载**：使用 `TSoftClassPtr` 和 `TSoftObjectPtr`，按需加载资源
3. **追踪清理**：在 `FActorExtensions` 中记录所有添加的能力句柄，便于卸载时清理
4. **上下文隔离**：使用 `FGameFeatureStateChangeContext` 区分不同的激活上下文（如不同的 World）

### GameFeatureAction_AddInputBinding 源码

```cpp
// GameFeatureAction_AddInputBinding.h
UCLASS(meta = (DisplayName = "Add Input Bindings"))
class UGameFeatureAction_AddInputBinding : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category="Input")
    TArray<FInputMappingContextAndPriority> InputMappings;

    UPROPERTY(EditAnywhere, Category="Input")
    TArray<FLyraInputBinding> InputBindings;
};

USTRUCT()
struct FLyraInputBinding
{
    GENERATED_BODY()

    // 目标 Pawn 类型
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<APawn> PawnClass;

    // 输入动作
    UPROPERTY(EditAnywhere)
    TSoftObjectPtr<UInputAction> InputAction;

    // 要触发的 Gameplay Ability
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<ULyraGameplayAbility> AbilityToTrigger;
};

// 核心逻辑
void UGameFeatureAction_AddInputBinding::AddInputBinding(
    APawn* Pawn, 
    const FLyraInputBinding& Binding)
{
    if (ULyraHeroComponent* HeroComp = Pawn->FindComponentByClass<ULyraHeroComponent>())
    {
        ULyraInputConfig* InputConfig = HeroComp->GetInputConfig();
        if (InputConfig)
        {
            // 将 Input Action 和 Ability 绑定
            InputConfig->BindAbilityActions(
                Binding.InputAction.LoadSynchronous(),
                Binding.AbilityToTrigger.LoadSynchronous()
            );
        }
    }
}
```

---

## 在 Experience 中使用 Game Features

### Experience Definition 配置

在 Lyra 中，Experience 是 Game Feature 的调度者：

```cpp
// B_LyraShooterGame_Elimination (射击模式 Experience)
UCLASS()
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 1️⃣ 要启用的 Game Features
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<FString> GameFeaturesToEnable = {
        "ShooterCore",          // 核心射击系统
        "ShooterMaps",          // 地图集
        "ShooterExplorer"       // 探索模式
    };

    // 2️⃣ 要执行的 Actions
    UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    // 3️⃣ Action Sets（可复用的 Action 组合）
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets;
};
```

**配置示例（蓝图中）**：

```
B_LyraShooterGame_Elimination (Experience Definition)
├── GameFeaturesToEnable:
│   ├── [0] = "ShooterCore"
│   └── [1] = "ShooterMaps"
│
├── Actions:
│   ├── [0] GameFeatureAction_AddAbilities
│   │   └── AbilitiesList:
│   │       └── [0] ActorClass = LyraCharacter
│   │           ├── GrantedAbilities:
│   │           │   ├── GA_Weapon_Fire
│   │           │   ├── GA_Weapon_Reload
│   │           │   └── GA_Hero_Jump
│   │           └── GrantedAbilitySets:
│   │               └── AbilitySet_ShooterHero
│   │
│   ├── [1] GameFeatureAction_AddInputContextMapping
│   │   └── InputMappings:
│   │       └── IMC_Shooter (射击模式键位)
│   │
│   └── [2] GameFeatureAction_AddWidget
│       └── Widgets:
│           ├── W_ShooterHUD (主界面)
│           └── W_AmmoCounter (弹药计数器)
│
└── ActionSets:
    └── [0] AS_SharedInput (共享输入配置)
```

### Experience 加载流程

```cpp
// LyraExperienceManagerComponent.cpp (简化)
void ULyraExperienceManagerComponent::StartExperienceLoad()
{
    // 1. 获取 Experience Definition
    const ULyraExperienceDefinition* Experience = CurrentExperience;
    
    // 2. 加载 Game Feature 插件
    int32 NumGameFeaturesToLoad = Experience->GameFeaturesToEnable.Num();
    for (const FString& PluginName : Experience->GameFeaturesToEnable)
    {
        FString PluginURL = UGameFeaturesSubsystem::GetPluginURL_FileProtocol(
            PluginName, 
            FString()
        );
        
        UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(
            PluginURL,
            FGameFeaturePluginLoadComplete::CreateUObject(
                this, 
                &ThisClass::OnGameFeaturePluginLoadComplete, 
                PluginURL
            )
        );
    }
}

void ULyraExperienceManagerComponent::OnGameFeaturePluginLoadComplete(
    const UE::GameFeatures::FResult& Result, 
    FString PluginURL)
{
    NumGameFeaturePluginsLoading--;
    
    if (Result.HasValue())
    {
        UE_LOG(LogLyra, Log, TEXT("✅ Loaded Game Feature: %s"), *PluginURL);
    }
    
    // 所有插件加载完成
    if (NumGameFeaturePluginsLoading == 0)
    {
        // 3. 执行 Experience 的 Actions
        ExecuteActions();
        
        // 4. 广播加载完成事件
        OnExperienceLoaded.Broadcast(CurrentExperience);
    }
}

void ULyraExperienceManagerComponent::ExecuteActions()
{
    FGameFeatureActivatingContext Context;
    
    // 执行 Experience 中配置的所有 Actions
    for (UGameFeatureAction* Action : CurrentExperience->Actions)
    {
        if (Action)
        {
            Action->OnGameFeatureActivating(Context);
        }
    }
    
    // 执行 Action Sets 中的 Actions
    for (const ULyraExperienceActionSet* Set : CurrentExperience->ActionSets)
    {
        for (UGameFeatureAction* Action : Set->Actions)
        {
            if (Action)
            {
                Action->OnGameFeatureActivating(Context);
            }
        }
    }
}
```

**时序图**：

```
[LyraGameMode]
     |
     | StartPlay()
     ↓
[ExperienceManager]
     |
     | 1. StartExperienceLoad()
     ↓
[GameFeaturesSubsystem]
     |
     | 2. LoadAndActivateGameFeaturePlugin("ShooterCore")
     ↓
     | 状态转换: Registered → Loaded → Activating
     |
     | 3. 执行插件内的 GameFeatureActions
     |    - AddAbilities
     |    - AddInputBinding
     |    - AddWidget
     ↓
     | 状态: Active ✅
     |
     | 4. OnPluginLoadComplete 回调
     ↓
[ExperienceManager]
     |
     | 5. 检查所有插件是否加载完成
     |
     | 6. ExecuteActions() (执行 Experience 配置的 Actions)
     ↓
     | 7. OnExperienceLoaded 广播
     ↓
[LyraGameState/PlayerController 等]
     |
     | 8. 监听事件，初始化游戏逻辑
```

---

## 创建自定义 Game Feature 插件

### Step 1: 创建插件

使用编辑器创建 Game Feature 插件：

```cpp
// 1. Edit → Plugins → Add → Game Feature (with C++)
// 插件名称：MyCustomFeature

// 2. 编辑 MyCustomFeature.uplugin
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "1.0",
    "FriendlyName": "My Custom Feature",
    "Description": "Custom gameplay feature for my game",
    "Category": "Game Features",
    "CreatedBy": "Your Name",
    "CanContainContent": true,
    "ExplicitlyLoaded": true,
    "EnabledByDefault": false,
    "BuiltInInitialFeatureState": "Registered",  // ✅ 关键
    
    "Modules": [
        {
            "Name": "MyCustomFeatureRuntime",
            "Type": "Runtime",
            "LoadingPhase": "Default"
        }
    ],
    
    "Plugins": [
        {"Name": "GameplayAbilities", "Enabled": true},
        {"Name": "ModularGameplay", "Enabled": true}
    ]
}
```

### Step 2: 创建插件模块

```cpp
// MyCustomFeatureRuntime.Build.cs
using UnrealBuildTool;

public class MyCustomFeatureRuntime : ModuleRules
{
    public MyCustomFeatureRuntime(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicDependencyModuleNames.AddRange(new string[]
        {
            "Core",
            "CoreUObject",
            "Engine",
            "ModularGameplay",          // ✅ Lyra 的模块化基础
            "GameplayAbilities",        // ✅ GAS 支持
            "GameFeatures",             // ✅ Game Feature 系统
        });

        PrivateDependencyModuleNames.AddRange(new string[]
        {
            "LyraGame",                 // ✅ 引用主项目（可选）
        });
    }
}

// MyCustomFeatureRuntimeModule.cpp
#include "Modules/ModuleManager.h"

class FMyCustomFeatureRuntimeModule : public IModuleInterface
{
public:
    virtual void StartupModule() override
    {
        UE_LOG(LogTemp, Log, TEXT("MyCustomFeature: Module Started"));
    }

    virtual void ShutdownModule() override
    {
        UE_LOG(LogTemp, Log, TEXT("MyCustomFeature: Module Shutdown"));
    }
};

IMPLEMENT_MODULE(FMyCustomFeatureRuntimeModule, MyCustomFeatureRuntime)
```

### Step 3: 添加内容

```
MyCustomFeature/
├── Content/
│   ├── Abilities/
│   │   ├── GA_CustomDash.uasset         (冲刺能力)
│   │   └── GE_CustomSpeedBoost.uasset   (速度提升效果)
│   ├── Input/
│   │   └── IMC_CustomControls.uasset    (输入映射)
│   └── UI/
│       └── W_CustomHUD.uasset           (自定义 HUD)
└── Source/
    └── MyCustomFeatureRuntime/
        ├── Public/
        │   └── MyCustomComponent.h
        └── Private/
            └── MyCustomComponent.cpp
```

**创建自定义组件**：

```cpp
// MyCustomComponent.h
#pragma once

#include "Components/GameFrameworkComponent.h"
#include "MyCustomComponent.generated.h"

UCLASS(Blueprintable, meta=(BlueprintSpawnableComponent))
class UMyCustomComponent : public UGameFrameworkComponent
{
    GENERATED_BODY()

public:
    UMyCustomComponent(const FObjectInitializer& ObjectInitializer);

    // 初始化状态接口（与 Lyra 的模块化系统集成）
    virtual void OnRegister() override;
    virtual void BeginPlay() override;

    UFUNCTION(BlueprintCallable, Category="Custom")
    void ActivateCustomFeature();

private:
    UPROPERTY(EditAnywhere, Category="Custom")
    float FeaturePower = 100.0f;
};

// MyCustomComponent.cpp
#include "MyCustomComponent.h"

UMyCustomComponent::UMyCustomComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    PrimaryComponentTick.bCanEverTick = false;
}

void UMyCustomComponent::OnRegister()
{
    Super::OnRegister();
    UE_LOG(LogTemp, Log, TEXT("MyCustomComponent: Registered"));
}

void UMyCustomComponent::BeginPlay()
{
    Super::BeginPlay();
    UE_LOG(LogTemp, Log, TEXT("MyCustomComponent: Begin Play"));
}

void UMyCustomComponent::ActivateCustomFeature()
{
    UE_LOG(LogTemp, Warning, TEXT("Custom Feature Activated! Power: %f"), FeaturePower);
}
```

### Step 4: 创建 Experience Definition

```cpp
// Content/Experiences/B_MyCustomExperience.uasset (蓝图数据资源)

// 在蓝图编辑器中配置：
class UMyCustomExperienceDefinition : public ULyraExperienceDefinition
{
    // 继承 ULyraExperienceDefinition，无需额外代码
};

// 蓝图配置示例：
GameFeaturesToEnable:
    [0] = "MyCustomFeature"
    [1] = "ShooterCore"  (可选，复用现有功能)

Actions:
    [0] GameFeatureAction_AddAbilities:
        ActorClass = LyraCharacter
        GrantedAbilities:
            - GA_CustomDash (冲刺能力)
            - GA_CustomWallRun (跑墙能力)

    [1] GameFeatureAction_AddInputContextMapping:
        InputMappings:
            - IMC_CustomControls (Priority = 1)

    [2] GameFeatureAction_AddWidget:
        Widgets:
            Layout = HUD
            WidgetClass = W_CustomHUD
            SlotID = "CustomHUD"

DefaultPawnData:
    = PawnData_Custom (自定义角色配置)
```

### Step 5: 在地图中使用

```cpp
// 方法 1：通过 World Settings 设置默认 Experience
// 打开地图 → World Settings → Lyra Experience → Default Experience
// 选择 B_MyCustomExperience

// 方法 2：通过 URL 参数动态指定
// 命令行：MyGame.exe /Game/Maps/MyMap?Experience=B_MyCustomExperience

// 方法 3：C++ 代码动态加载
ALyraGameMode* GameMode = Cast<ALyraGameMode>(GetWorld()->GetAuthGameMode());
if (GameMode)
{
    ULyraExperienceManagerComponent* ExperienceManager = 
        GameMode->GetExperienceManagerComponent();
    
    ExperienceManager->SetCurrentExperience(
        TEXT("/MyCustomFeature/Experiences/B_MyCustomExperience")
    );
}
```

---

## 实战案例：开发一个 CTF 模式插件

### 需求分析

创建一个**夺旗模式（Capture The Flag）**插件，包含：

- ✅ 旗帜 Actor
- ✅ 夺旗/归旗能力
- ✅ 旗帜 UI 指示器
- ✅ 得分规则组件

### 插件结构

```
CTFMode/
├── CTFMode.uplugin
├── Content/
│   ├── Actors/
│   │   └── BP_CTFFlag.uasset
│   ├── Abilities/
│   │   ├── GA_PickupFlag.uasset
│   │   └── GA_DropFlag.uasset
│   ├── UI/
│   │   └── W_FlagIndicator.uasset
│   └── Experiences/
│       └── B_CTFExperience.uasset
└── Source/
    └── CTFModeRuntime/
        ├── Public/
        │   ├── CTFFlagActor.h
        │   ├── CTFScoreComponent.h
        │   └── GameFeatureAction_AddCTFRules.h
        └── Private/
            └── ...
```

### 1. 旗帜 Actor

```cpp
// CTFFlagActor.h
#pragma once

#include "GameFramework/Actor.h"
#include "AbilitySystemInterface.h"
#include "CTFFlagActor.generated.h"

UENUM(BlueprintType)
enum class EFlagState : uint8
{
    AtBase,      // 在基地
    Carried,     // 被携带
    Dropped      // 被丢弃
};

UCLASS()
class ACTFFlagActor : public AActor, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    ACTFFlagActor();

    // IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

    // 旗帜逻辑
    UFUNCTION(BlueprintCallable, Category="CTF")
    void PickupFlag(APawn* Carrier);

    UFUNCTION(BlueprintCallable, Category="CTF")
    void DropFlag();

    UFUNCTION(BlueprintCallable, Category="CTF")
    void ReturnToBase();

protected:
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<UStaticMeshComponent> FlagMesh;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    UPROPERTY(ReplicatedUsing=OnRep_FlagState)
    EFlagState CurrentState;

    UPROPERTY(Replicated)
    TObjectPtr<APawn> CurrentCarrier;

    UPROPERTY(EditAnywhere, Category="CTF")
    FVector BaseLocation;

    UFUNCTION()
    void OnRep_FlagState();

    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;
};

// CTFFlagActor.cpp
#include "CTFFlagActor.h"
#include "Net/UnrealNetwork.h"
#include "AbilitySystemComponent.h"

ACTFFlagActor::ACTFFlagActor()
{
    PrimaryActorTick.bCanEverTick = false;
    bReplicates = true;

    // 创建组件
    FlagMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("FlagMesh"));
    RootComponent = FlagMesh;

    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(
        TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
}

UAbilitySystemComponent* ACTFFlagActor::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}

void ACTFFlagActor::PickupFlag(APawn* Carrier)
{
    if (CurrentState == EFlagState::Carried) return;

    CurrentState = EFlagState::Carried;
    CurrentCarrier = Carrier;

    // 附加到携带者
    AttachToComponent(
        Carrier->GetRootComponent(),
        FAttachmentTransformRules::SnapToTargetNotIncludingScale,
        TEXT("FlagSocket")  // 需要在角色骨骼上添加此 Socket
    );

    FlagMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);

    UE_LOG(LogTemp, Log, TEXT("Flag picked up by %s"), 
        *Carrier->GetName());
}

void ACTFFlagActor::DropFlag()
{
    if (CurrentState != EFlagState::Carried) return;

    CurrentState = EFlagState::Dropped;
    CurrentCarrier = nullptr;

    // 从携带者分离
    DetachFromActor(FDetachmentTransformRules::KeepWorldTransform);
    FlagMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);

    // 5秒后自动回基地
    GetWorldTimerManager().SetTimer(
        ReturnTimer, 
        this, 
        &ACTFFlagActor::ReturnToBase, 
        5.0f, 
        false
    );

    UE_LOG(LogTemp, Warning, TEXT("Flag dropped!"));
}

void ACTFFlagActor::ReturnToBase()
{
    CurrentState = EFlagState::AtBase;
    CurrentCarrier = nullptr;

    SetActorLocation(BaseLocation);
    DetachFromActor(FDetachmentTransformRules::KeepWorldTransform);
    FlagMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);

    UE_LOG(LogTemp, Log, TEXT("Flag returned to base"));
}

void ACTFFlagActor::OnRep_FlagState()
{
    // 客户端同步状态，更新视觉效果
    switch (CurrentState)
    {
    case EFlagState::AtBase:
        FlagMesh->SetVisibility(true);
        break;
    case EFlagState::Carried:
        FlagMesh->SetVisibility(true);
        break;
    case EFlagState::Dropped:
        FlagMesh->SetVisibility(true);
        // 播放掉落特效
        break;
    }
}

void ACTFFlagActor::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ACTFFlagActor, CurrentState);
    DOREPLIFETIME(ACTFFlagActor, CurrentCarrier);
}
```

### 2. 夺旗能力

```cpp
// GA_PickupFlag.h
#pragma once

#include "Abilities/LyraGameplayAbility.h"
#include "GA_PickupFlag.generated.h"

UCLASS()
class UGA_PickupFlag : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_PickupFlag();

    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData) override;

protected:
    UPROPERTY(EditDefaultsOnly, Category="CTF")
    float PickupRange = 200.0f;
};

// GA_PickupFlag.cpp
#include "GA_PickupFlag.h"
#include "CTFFlagActor.h"
#include "Kismet/GameplayStatics.h"

UGA_PickupFlag::UGA_PickupFlag()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated;
}

void UGA_PickupFlag::ActivateAbility(
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

    APawn* Pawn = CastChecked<APawn>(ActorInfo->AvatarActor.Get());
    FVector PawnLocation = Pawn->GetActorLocation();

    // 查找附近的旗帜
    TArray<AActor*> FoundFlags;
    UGameplayStatics::GetAllActorsOfClass(
        GetWorld(), 
        ACTFFlagActor::StaticClass(), 
        FoundFlags
    );

    for (AActor* FlagActor : FoundFlags)
    {
        if (ACTFFlagActor* Flag = Cast<ACTFFlagActor>(FlagActor))
        {
            float Distance = FVector::Dist(PawnLocation, Flag->GetActorLocation());
            
            if (Distance <= PickupRange)
            {
                // 拾取旗帜
                Flag->PickupFlag(Pawn);
                
                // 应用 Gameplay Effect（降低移动速度）
                ApplyGameplayEffectToOwner(
                    Handle, 
                    ActorInfo, 
                    ActivationInfo, 
                    MakeEffectContext(Handle, ActorInfo),
                    FlagCarrySpeedDebuff,  // 预定义的 GE
                    1.0f
                );
                
                break;
            }
        }
    }

    EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}
```

### 3. 自定义 GameFeatureAction

```cpp
// GameFeatureAction_AddCTFRules.h
#pragma once

#include "GameFeatures/GameFeatureAction_WorldActionBase.h"
#include "GameFeatureAction_AddCTFRules.generated.h"

UCLASS(meta = (DisplayName = "Add CTF Rules"))
class UGameFeatureAction_AddCTFRules : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category="CTF")
    int32 ScoreToWin = 3;

    UPROPERTY(EditAnywhere, Category="CTF")
    TSoftClassPtr<AActor> FlagActorClass;

protected:
    virtual void AddToWorld(
        const FWorldContext& WorldContext, 
        const FGameFeatureStateChangeContext& ChangeContext) override;

private:
    void SpawnFlags(UWorld* World);
    void SetupScoreTracking(UWorld* World);
};

// GameFeatureAction_AddCTFRules.cpp
#include "GameFeatureAction_AddCTFRules.h"
#include "EngineUtils.h"
#include "GameModes/LyraGameState.h"

void UGameFeatureAction_AddCTFRules::AddToWorld(
    const FWorldContext& WorldContext, 
    const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    if (!World || !World->IsGameWorld()) return;

    // 生成旗帜
    SpawnFlags(World);

    // 设置得分追踪
    SetupScoreTracking(World);
}

void UGameFeatureAction_AddCTFRules::SpawnFlags(UWorld* World)
{
    if (FlagActorClass.IsNull()) return;

    TSubclassOf<AActor> FlagClass = FlagActorClass.LoadSynchronous();
    
    // 查找 PlayerStart 作为旗帜生成点
    TArray<FVector> SpawnLocations;
    for (TActorIterator<APlayerStart> It(World); It; ++It)
    {
        SpawnLocations.Add(It->GetActorLocation());
    }

    // 为每个队伍生成旗帜
    if (SpawnLocations.Num() >= 2)
    {
        FActorSpawnParameters SpawnParams;
        SpawnParams.SpawnCollisionHandlingOverride = 
            ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

        // 红队旗帜
        World->SpawnActor<AActor>(
            FlagClass, 
            SpawnLocations[0], 
            FRotator::ZeroRotator, 
            SpawnParams
        );

        // 蓝队旗帜
        World->SpawnActor<AActor>(
            FlagClass, 
            SpawnLocations[1], 
            FRotator::ZeroRotator, 
            SpawnParams
        );

        UE_LOG(LogTemp, Log, TEXT("CTF Flags spawned"));
    }
}

void UGameFeatureAction_AddCTFRules::SetupScoreTracking(UWorld* World)
{
    if (ALyraGameState* GameState = World->GetGameState<ALyraGameState>())
    {
        // 设置获胜条件
        // （需要在 GameState 中添加 CTF 相关逻辑）
        UE_LOG(LogTemp, Log, TEXT("CTF Rules: Score to win = %d"), ScoreToWin);
    }
}
```

### 4. Experience 配置

```cpp
// B_CTFExperience (蓝图数据资源)

GameFeaturesToEnable:
    [0] = "CTFMode"
    [1] = "ShooterCore"  // 复用射击系统

Actions:
    [0] GameFeatureAction_AddCTFRules:
        ScoreToWin = 5
        FlagActorClass = BP_CTFFlag

    [1] GameFeatureAction_AddAbilities:
        ActorClass = LyraCharacter
        GrantedAbilities:
            - GA_PickupFlag (E 键拾取)
            - GA_DropFlag (G 键丢弃)

    [2] GameFeatureAction_AddInputContextMapping:
        InputMappings:
            - IMC_CTF (Priority = 2)

    [3] GameFeatureAction_AddWidget:
        Widgets:
            Layout = HUD
            WidgetClass = W_FlagIndicator

DefaultPawnData:
    = PawnData_ShooterHero
```

### 5. 测试与调试

```cpp
// 1. 启动编辑器
// 2. 打开测试地图
// 3. World Settings → Default Experience = B_CTFExperience
// 4. PIE (Play In Editor)

// 调试命令：
// ShowDebug AbilitySystem  (查看 GAS 状态)
// ShowDebug GameFeatures   (查看插件状态)

// C++ 日志输出：
UE_LOG(LogTemp, Display, TEXT("CTF Mode: %s"), 
    UGameFeaturesSubsystem::Get().GetPluginState("CTFMode") == 
    EGameFeaturePluginState::Active ? TEXT("Active") : TEXT("Inactive"));
```

---

## 高级技巧与最佳实践

### 1. 插件依赖管理

```json
// MyGameMode.uplugin
{
    "Plugins": [
        {"Name": "CoreFeatures", "Enabled": true},  // 基础功能
        {"Name": "SharedAssets", "Enabled": true}   // 共享资源
    ]
}

// 依赖链：
// MyGameMode → CoreFeatures → GameplayAbilities
//           ↘ SharedAssets → CommonUI
```

**最佳实践**：
- ✅ 将通用功能抽取到基础插件中
- ✅ 避免循环依赖
- ✅ 使用软引用（TSoftObjectPtr）减少硬依赖

### 2. 资源异步加载

```cpp
// ❌ 错误：同步加载（卡顿）
TSubclassOf<AActor> ActorClass = SoftClass.LoadSynchronous();

// ✅ 正确：异步加载
FStreamableManager& StreamableManager = 
    UAssetManager::GetStreamableManager();

TSharedPtr<FStreamableHandle> Handle = StreamableManager.RequestAsyncLoad(
    SoftClass.ToSoftObjectPath(),
    [this, SoftClass]()
    {
        if (TSubclassOf<AActor> LoadedClass = SoftClass.Get())
        {
            // 加载完成，使用 LoadedClass
        }
    }
);
```

### 3. 插件热重载

```cpp
// 运行时卸载插件
UGameFeaturesSubsystem& GFS = UGameFeaturesSubsystem::Get();
GFS.UnloadGameFeaturePlugin("MyPlugin", true);

// 修改插件内容

// 重新加载
GFS.LoadAndActivateGameFeaturePlugin("MyPlugin", OnLoadComplete);
```

**注意事项**：
- ⚠️ 卸载前需确保插件创建的 Actor/Component 已清理
- ⚠️ 蓝图类需要手动处理实例引用
- ⚠️ GAS 能力需要从 ASC 中移除

### 4. 多 World 支持

```cpp
// GameFeatureAction_WorldActionBase 自动处理多 World
void UGameFeatureAction_AddAbilities::AddToWorld(
    const FWorldContext& WorldContext,  // ✅ 每个 World 独立调用
    const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    
    // 为每个 World 创建独立的数据
    FPerContextData& Data = ContextData.FindOrAdd(ChangeContext);
}
```

**使用场景**：
- 同时运行多个 Experience（如编辑器 PIE 多窗口）
- 服务器上的多个关卡实例

### 5. 调试与性能分析

```cpp
// 1. 启用详细日志
[Core.Log]
LogGameFeatures=Verbose
LogLyraExperience=Verbose
LogModularGameplay=Verbose

// 2. C++ 断点调试
// 在 GameFeatureAction::OnGameFeatureActivating 设置断点

// 3. 性能分析
// Stat GameFeatures  (查看插件加载时间)
// Stat LyraExperience  (查看 Experience 加载时间)

// 4. 内存分析
// MemReport -full  (生成内存报告)
// obj list class=GameFeatureAction  (列出所有 Action 实例)
```

### 6. 版本控制与团队协作

```
推荐的团队工作流：

1. 基础框架（LyraGame）：
   - 由核心团队维护
   - 稳定版本，不频繁修改

2. 功能插件（GameFeatures/*）：
   - 各团队独立开发
   - 独立的 Git 仓库或分支
   - 通过 Git Submodule 集成

3. Experience 配置：
   - 由 Game Designer 配置
   - 仅修改数据资产，不涉及代码

4. 测试地图：
   - QA 团队维护
   - 每个地图测试不同的 Experience
```

**Git 结构示例**：

```
MyGame/
├── .git/
├── LyraGame/                 (主仓库)
│   └── Source/
└── Plugins/
    └── GameFeatures/
        ├── ShooterCore/      (Submodule: git@shooter.git)
        ├── RPGMode/          (Submodule: git@rpg.git)
        └── CustomDLC/        (Submodule: git@dlc.git)
```

### 7. 单元测试

```cpp
// CTFModeTests.cpp
#include "Misc/AutomationTest.h"
#include "Tests/AutomationCommon.h"

IMPLEMENT_SIMPLE_AUTOMATION_TEST(
    FCTFPluginLoadTest,
    "GameFeatures.CTFMode.LoadTest",
    EAutomationTestFlags::EditorContext | 
    EAutomationTestFlags::EngineFilter
)

bool FCTFPluginLoadTest::RunTest(const FString& Parameters)
{
    // 测试插件能否正常加载
    UGameFeaturesSubsystem& GFS = UGameFeaturesSubsystem::Get();
    
    FString PluginURL = UGameFeaturesSubsystem::GetPluginURL_FileProtocol(
        TEXT("CTFMode")
    );
    
    bool bLoadSuccess = false;
    GFS.LoadAndActivateGameFeaturePlugin(
        PluginURL,
        FGameFeaturePluginLoadComplete::CreateLambda(
            [&bLoadSuccess](const UE::GameFeatures::FResult& Result)
            {
                bLoadSuccess = Result.HasValue();
            }
        )
    );
    
    // 等待加载完成
    ADD_LATENT_AUTOMATION_COMMAND(FWaitLatentCommand(2.0f));
    
    TestTrue(TEXT("CTF Plugin loaded successfully"), bLoadSuccess);
    
    return true;
}
```

---

## 常见问题与解决方案

### Q1: 插件无法加载，显示 "StatusKnown" 状态

**原因**：插件未正确注册到 Asset Registry

**解决方案**：

```cpp
// 1. 检查 .uplugin 文件
"BuiltInInitialFeatureState": "Registered"  // ✅ 必须设置

// 2. 确保插件在正确的路径
Plugins/GameFeatures/MyPlugin/MyPlugin.uplugin  // ✅ 正确
Plugins/MyPlugin/MyPlugin.uplugin              // ❌ 错误（不在 GameFeatures 下）

// 3. 刷新项目文件
右键 .uproject → Generate Visual Studio project files

// 4. 重启编辑器
```

### Q2: GameFeatureAction 不执行

**原因**：Action 未添加到 Experience Definition

**解决方案**：

```cpp
// 检查 Experience Definition
UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
TArray<TObjectPtr<UGameFeatureAction>> Actions;

// 确保在蓝图中添加了 Action 实例
// 不是 TSoftClassPtr，而是直接实例化！
```

### Q3: 能力未添加到角色

**原因**：Actor 类型不匹配或初始化顺序问题

**解决方案**：

```cpp
// 1. 检查 ActorClass 配置
FGameFeatureAbilitiesEntry Entry;
Entry.ActorClass = ALyraCharacter::StaticClass();  // ✅ 精确匹配

// 2. 确保 Actor 实现了 IGameFrameworkInitStateInterface
class ALyraCharacter : public AModularCharacter, 
                       public IGameFrameworkInitStateInterface
{
    // ...
};

// 3. 监听初始化状态
void ALyraCharacter::OnActorInitStateChanged(
    const FActorInitStateChangedParams& Params)
{
    // Abilities 在 "DataAvailable" 或 "DataInitialized" 状态添加
}
```

### Q4: 插件卸载后资源泄漏

**原因**：未正确清理 Action 创建的对象

**解决方案**：

```cpp
// 在 GameFeatureAction 中实现清理逻辑
void UGameFeatureAction_AddAbilities::OnGameFeatureDeactivating(
    FGameFeatureDeactivatingContext& Context)
{
    Super::OnGameFeatureDeactivating(Context);

    // 移除所有添加的能力
    for (auto& ContextPair : ContextData)
    {
        Reset(ContextPair.Value);
    }
    
    ContextData.Empty();  // ✅ 清空缓存
}

void UGameFeatureAction_AddAbilities::Reset(FPerContextData& Data)
{
    for (auto& ExtensionPair : Data.ActiveExtensions)
    {
        AActor* Actor = ExtensionPair.Key;
        FActorExtensions& Extensions = ExtensionPair.Value;

        if (UAbilitySystemComponent* ASC = 
            Actor->FindComponentByClass<UAbilitySystemComponent>())
        {
            // 移除能力
            for (FGameplayAbilitySpecHandle Handle : Extensions.Abilities)
            {
                ASC->ClearAbility(Handle);
            }

            // 移除属性集
            for (UAttributeSet* Set : Extensions.Attributes)
            {
                ASC->RemoveSpawnedAttribute(Set);
            }
        }
    }
}
```

### Q5: 多人游戏中插件状态不同步

**原因**：Game Feature 加载是本地操作，不会自动复制

**解决方案**：

```cpp
// 服务器加载 Experience
if (HasAuthority())
{
    ExperienceManager->SetCurrentExperience(ExperienceID);
}

// Experience 加载完成后，通过 GameState 复制给客户端
UPROPERTY(ReplicatedUsing=OnRep_CurrentExperience)
FPrimaryAssetId CurrentExperienceId;

void ALyraGameState::OnRep_CurrentExperience()
{
    // 客户端收到通知，加载相同的 Experience
    ULyraExperienceManagerComponent* ExperienceManager = 
        GetGameMode()->GetExperienceManagerComponent();
    ExperienceManager->ClientLoadExperience(CurrentExperienceId);
}
```

---

## 性能优化

### 1. 异步加载策略

```cpp
// ❌ 同步加载所有内容（启动卡顿）
for (const FString& Plugin : GameFeaturesToEnable)
{
    UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(
        Plugin, 
        FGameFeaturePluginLoadComplete()
    );
}

// ✅ 批量异步加载 + 优先级管理
TArray<FString> HighPriorityPlugins = {"CoreGameplay", "PlayerAbilities"};
TArray<FString> LowPriorityPlugins = {"Cosmetics", "Emotes"};

// 先加载高优先级
LoadPluginsBatch(HighPriorityPlugins, []() 
{
    // 高优先级完成后，加载低优先级
    LoadPluginsBatch(LowPriorityPlugins, []() 
    {
        // 全部完成
    });
});
```

### 2. 按需加载

```cpp
// 不在 Experience 中直接启用所有插件，而是动态加载
void AMyGameMode::OnPlayerJoinTeam(APlayerController* PC, ETeam Team)
{
    FString TeamPlugin = (Team == ETeam::Red) ? "RedTeamAbilities" : "BlueTeamAbilities";
    
    UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(
        TeamPlugin,
        FGameFeaturePluginLoadComplete::CreateLambda([PC](const auto& Result)
        {
            // 只为该玩家激活队伍特定能力
        })
    );
}
```

### 3. 内存优化

```cpp
// 使用 Asset Bundles 控制资源加载粒度
UPROPERTY(EditAnywhere, meta=(AssetBundles="Client"))
TSoftObjectPtr<UTexture2D> HighResTexture;  // 仅客户端加载

UPROPERTY(EditAnywhere, meta=(AssetBundles="Server"))
TSubclassOf<AActor> ServerOnlyLogic;  // 仅服务器加载

UPROPERTY(EditAnywhere, meta=(AssetBundles="Client,Server"))
TSubclassOf<ACharacter> SharedAsset;  // 客户端和服务器都加载
```

### 4. 卸载不需要的插件

```cpp
// 切换 Experience 时卸载旧插件
void ULyraExperienceManagerComponent::DeactivateExperience()
{
    const ULyraExperienceDefinition* OldExperience = CurrentExperience;
    
    for (const FString& PluginName : OldExperience->GameFeaturesToEnable)
    {
        UGameFeaturesSubsystem::Get().DeactivateGameFeaturePlugin(PluginName);
        
        // 可选：完全卸载以释放内存
        UGameFeaturesSubsystem::Get().UnloadGameFeaturePlugin(PluginName);
    }
}
```

---

## 与其他系统的集成

### 1. 与 GAS 集成

```cpp
// Game Feature 动态添加 Gameplay Tag
UCLASS()
class UGameFeatureAction_AddGameplayTags : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere)
    FGameplayTagContainer TagsToAdd;

    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override
    {
        UGameplayTagsManager& TagManager = UGameplayTagsManager::Get();
        
        for (const FGameplayTag& Tag : TagsToAdd)
        {
            TagManager.AddNativeGameplayTag(Tag);
        }
    }
};
```

### 2. 与 Common UI 集成

```cpp
// 动态注册 UI Layer
UCLASS()
class UGameFeatureAction_RegisterUILayer : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere)
    FGameplayTag LayerTag;

    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UCommonActivatableWidget> WidgetClass;

    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override
    {
        if (ULyraUIManagerSubsystem* UIManager = 
            GEngine->GetEngineSubsystem<ULyraUIManagerSubsystem>())
        {
            UIManager->RegisterLayer(LayerTag, WidgetClass.LoadSynchronous());
        }
    }
};
```

### 3. 与网络复制集成

```cpp
// 确保 Game Feature 创建的对象正确复制
void UGameFeatureAction_AddAbilities::AddActorAbilities(
    AActor* Actor, 
    const FGameFeatureAbilitiesEntry& Entry, 
    FPerContextData& ActiveData)
{
    UAbilitySystemComponent* ASC = /* ... */;

    // ✅ 设置复制模式
    ASC->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

    // 添加能力时，指定服务器/客户端行为
    FGameplayAbilitySpec Spec(AbilityClass);
    Spec.SourceObject = Actor;  // 设置源对象以支持复制

    if (Actor->HasAuthority())
    {
        ASC->GiveAbility(Spec);  // 仅服务器授予
    }
}
```

---

## 总结

### 核心要点回顾

1. **Game Feature 插件是 Lyra 的核心架构**：
   - 实现真正的模块化和可插拔设计
   - 支持运行时动态加载/卸载
   - 降低耦合，提高复用性

2. **GameFeatureAction 是插件的行为单元**：
   - 负责激活时的具体操作（添加能力、组件、UI 等）
   - 通过扩展点机制与 ModularGameplay 深度集成
   - 支持自定义 Action 扩展

3. **Experience 是 Game Feature 的调度者**：
   - 定义哪些插件需要激活
   - 配置插件的 Actions
   - 实现"游戏模式即数据"的设计理念

4. **最佳实践**：
   - ✅ 将独立功能封装为插件
   - ✅ 使用软引用避免硬依赖
   - ✅ 异步加载资源，优化启动时间
   - ✅ 正确实现清理逻辑，避免内存泄漏
   - ✅ 为多 World 场景做好隔离

### 适用场景

- **多游戏模式项目**：射击、赛车、解谜共存
- **DLC 和扩展内容**：后期动态添加新内容
- **实验性功能**：快速开启/关闭实验特性
- **大型团队协作**：多团队并行开发不同模块
- **平台差异化**：不同平台加载不同内容

### 学习路径建议

1. **初级**：理解 Game Feature 生命周期，创建简单插件
2. **中级**：掌握 GameFeatureAction 机制，实现自定义 Action
3. **高级**：深入源码，优化加载流程，处理复杂依赖

### 下一步

在下一篇文章中，我们将深入探讨 **Data Assets 与数据驱动设计**，学习如何通过配置文件而非硬编码来管理游戏内容。

---

## 参考资源

- [Unreal Engine 5 - Game Features Documentation](https://docs.unrealengine.com/5.0/en-US/game-features-and-modular-gameplay-in-unreal-engine/)
- [Lyra Starter Game - GitHub](https://github.com/EpicGames/UnrealEngine)
- [ModularGameplay Plugin Source Code](https://github.com/EpicGames/UnrealEngine/tree/release/Engine/Plugins/Experimental/ModularGameplay)

---

**字数统计**：约 22,000 字

**覆盖内容**：
- ✅ Game Feature 概念与优势
- ✅ 生命周期状态机详解
- ✅ ShooterCore 插件剖析
- ✅ GameFeatureAction 源码分析
- ✅ Experience 集成机制
- ✅ 完整的 CTF 模式实战案例
- ✅ 高级技巧与最佳实践
- ✅ 常见问题与性能优化
- ✅ 与其他系统的集成示例
