# 第27章：自定义 Game Feature Action

## 27.1 Game Feature Action 架构概述

Game Feature Action 是 Unreal Engine 5 Game Features 插件系统的核心扩展机制。在 Lyra 项目中，Action 充当了模块化功能的执行单元，负责在插件生命周期的各个阶段执行特定操作，如添加 Gameplay Abilities、注入 UI 组件、绑定输入映射等。

### 27.1.1 UGameFeatureAction 基类架构

`UGameFeatureAction` 是所有 Action 的抽象基类，定义在引擎的 Game Features 插件中。让我们先看其核心接口：

```cpp
// Engine/Plugins/Experimental/GameFeatures/Source/GameFeatures/Public/GameFeatureAction.h
UCLASS(DefaultToInstanced, EditInlineNew, Abstract)
class GAMEFEATURES_API UGameFeatureAction : public UObject
{
    GENERATED_BODY()

public:
    /** Called when the feature is registering */
    virtual void OnGameFeatureRegistering() {}

    /** Called when the feature is loading */
    virtual void OnGameFeatureLoading() {}

    /** Called when the feature is activating */
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) {}

    /** Called when the feature is deactivating */
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) {}

    /** Called when the feature is unloading */
    virtual void OnGameFeatureUnloading() {}

    /** Called when the feature is unregistering */
    virtual void OnGameFeatureUnregistering() {}
};
```

### 27.1.2 Lyra 的 WorldActionBase 扩展

Lyra 引入了 `UGameFeatureAction_WorldActionBase`，为需要与 World 交互的 Action 提供统一基础：

```cpp
// Source/LyraGame/GameFeatures/GameFeatureAction_WorldActionBase.h
UCLASS(Abstract)
class UGameFeatureAction_WorldActionBase : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override;
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;

private:
    void HandleGameInstanceStart(UGameInstance* GameInstance, FGameFeatureStateChangeContext ChangeContext);

    /** 子类实现的核心逻辑 */
    virtual void AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext) PURE_VIRTUAL(UGameFeatureAction_WorldActionBase::AddToWorld,);

private:
    TMap<FGameFeatureStateChangeContext, FDelegateHandle> GameInstanceStartHandles;
};
```

**核心设计思想：**

1. **World 生命周期感知**：自动监听 `OnStartGameInstance` 委托，确保在新 World 创建时自动激活
2. **上下文隔离**：通过 `FGameFeatureStateChangeContext` 区分不同的激活环境（Editor/PIE/Game）
3. **延迟初始化**：支持在 GameInstance 已经启动后的 World 中激活

### 27.1.3 与 Experience System 的集成

在 Lyra 中，Game Feature Actions 通过 Experience Definition 进行组织：

```cpp
// LyraExperienceDefinition.h
UCLASS(BlueprintType)
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    /** Game Features to activate with this experience */
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TArray<FString> GameFeaturesToEnable;

    /** Actions to perform when this experience is loaded */
    UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;
};
```

**执行流程：**

```
Experience 加载
    ↓
激活 GameFeatures（按 GameFeaturesToEnable 列表）
    ↓
执行 Experience 自带的 Actions
    ↓
执行每个 GameFeature 插件中定义的 Actions
    ↓
World 完全就绪
```

## 27.2 Action 生命周期深度解析

### 27.2.1 完整生命周期状态机

Game Feature 插件的状态转换遵循严格的状态机：

```
Uninitialized (未初始化)
    ↓
Registered (已注册) → OnGameFeatureRegistering()
    ↓
Loaded (资产已加载) → OnGameFeatureLoading()
    ↓
Activating (激活中)
    ↓
Active (已激活) → OnGameFeatureActivating()
    ↓
Deactivating (停用中) → OnGameFeatureDeactivating()
    ↓
Loaded (回到已加载状态)
    ↓
Unloading (卸载中) → OnGameFeatureUnloading()
    ↓
Terminal (终止状态) → OnGameFeatureUnregistering()
```

### 27.2.2 各生命周期回调的职责

**OnGameFeatureRegistering:**
- **时机**：插件首次被发现和注册时
- **用途**：注册全局性资源（如 Gameplay Tags、Data Registry 类型）
- **限制**：此时资产尚未加载，不能访问 UClass 或 UObject

```cpp
void UMyGameFeatureAction::OnGameFeatureRegistering()
{
    Super::OnGameFeatureRegistering();
    
    // 示例：注册自定义 Gameplay Tags
    UGameplayTagsManager& TagManager = UGameplayTagsManager::Get();
    TagManager.AddNativeGameplayTag(
        FName("Weather.Sunny"),
        FString("Sunny weather state")
    );
}
```

**OnGameFeatureLoading:**
- **时机**：插件资产开始加载时
- **用途**：预加载关键资产、初始化子系统
- **注意**：异步加载尚未完成，需处理资产引用的异步性

```cpp
void UMyGameFeatureAction::OnGameFeatureLoading()
{
    Super::OnGameFeatureLoading();
    
    // 启动异步资产加载
    FStreamableManager& Streamable = UAssetManager::GetStreamableManager();
    StreamableHandle = Streamable.RequestAsyncLoad(
        WeatherDataAsset.ToSoftObjectPath(),
        FStreamableDelegate::CreateUObject(this, &UMyGameFeatureAction::OnAssetLoaded)
    );
}
```

**OnGameFeatureActivating:**
- **时机**：插件正式激活，World 已准备就绪
- **用途**：执行主要逻辑（生成 Actor、添加组件、注册回调）
- **重点**：这是最常用的生命周期回调

```cpp
void UGameFeatureAction_WorldActionBase::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
    // 1. 注册 GameInstance 启动监听器
    GameInstanceStartHandles.FindOrAdd(Context) = FWorldDelegates::OnStartGameInstance.AddUObject(
        this, 
        &UGameFeatureAction_WorldActionBase::HandleGameInstanceStart, 
        FGameFeatureStateChangeContext(Context)
    );

    // 2. 对已存在的 World 立即执行
    for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
    {
        if (Context.ShouldApplyToWorldContext(WorldContext))
        {
            AddToWorld(WorldContext, Context);
        }
    }
}
```

**OnGameFeatureDeactivating:**
- **时机**：插件开始停用
- **用途**：清理资源、移除组件、注销回调
- **关键**：必须完全还原 Activating 阶段的所有操作

```cpp
void UGameFeatureAction_WorldActionBase::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    // 移除委托监听
    FDelegateHandle* FoundHandle = GameInstanceStartHandles.Find(Context);
    if (ensure(FoundHandle))
    {
        FWorldDelegates::OnStartGameInstance.Remove(*FoundHandle);
    }
}
```

### 27.2.3 上下文管理模式

Lyra Actions 普遍采用"上下文数据映射"模式管理多 World 状态：

```cpp
struct FPerContextData
{
    TArray<TSharedPtr<FComponentRequestHandle>> ComponentRequests;
    TMap<AActor*, FActorExtensions> ActiveExtensions;
};

TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;
```

**为什么需要上下文隔离？**

1. **多 World 共存**：Editor Preview + PIE + Standalone Game 可能同时运行
2. **独立清理**：停用插件时只清理对应 World 的修改
3. **状态追踪**：每个上下文的激活状态独立管理

## 27.3 Lyra 内置 Action 源码分析

### 27.3.1 GameFeatureAction_AddAbilities

这是 Lyra 中最复杂的 Action 之一，负责向 Actor 动态添加 Gameplay Abilities 和 Attribute Sets。

**核心数据结构：**

```cpp
USTRUCT()
struct FGameFeatureAbilitiesEntry
{
    GENERATED_BODY()

    // 目标 Actor 类型
    UPROPERTY(EditAnywhere, Category="Abilities")
    TSoftClassPtr<AActor> ActorClass;

    // 授予的 Abilities
    UPROPERTY(EditAnywhere, Category="Abilities")
    TArray<FLyraAbilityGrant> GrantedAbilities;

    // 授予的 Attribute Sets
    UPROPERTY(EditAnywhere, Category="Attributes")
    TArray<FLyraAttributeSetGrant> GrantedAttributes;

    // 授予的 Ability Sets（批量打包）
    UPROPERTY(EditAnywhere, Category="Attributes")
    TArray<TSoftObjectPtr<const ULyraAbilitySet>> GrantedAbilitySets;
};
```

**Actor 扩展追踪：**

```cpp
struct FActorExtensions
{
    TArray<FGameplayAbilitySpecHandle> Abilities;      // 已授予的 Ability 句柄
    TArray<UAttributeSet*> Attributes;                 // 已添加的 Attribute Set
    TArray<FLyraAbilitySet_GrantedHandles> AbilitySetHandles;  // Ability Set 句柄
};

struct FPerContextData
{
    TMap<AActor*, FActorExtensions> ActiveExtensions;  // Actor 到扩展的映射
    TArray<TSharedPtr<FComponentRequestHandle>> ComponentRequests;  // 组件请求句柄
};
```

**激活流程：**

```cpp
void UGameFeatureAction_AddAbilities::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    UGameFrameworkComponentManager* ComponentManager = UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(WorldContext.OwningGameInstance);

    if (!ComponentManager || !World || !World->IsGameWorld())
        return;

    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    // 为每个 AbilitiesEntry 注册 Actor 扩展请求
    for (int32 EntryIndex = 0; EntryIndex < AbilitiesList.Num(); ++EntryIndex)
    {
        const FGameFeatureAbilitiesEntry& Entry = AbilitiesList[EntryIndex];

        if (!Entry.ActorClass.IsNull())
        {
            // 创建扩展处理器
            UGameFrameworkComponentManager::FExtensionHandlerDelegate ExtensionHandler = 
                UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
                    this, 
                    &UGameFeatureAction_AddAbilities::HandleActorExtension, 
                    EntryIndex, 
                    ChangeContext
                );

            // 注册到 Component Manager
            TSharedPtr<FComponentRequestHandle> RequestHandle = ComponentManager->AddExtensionHandler(Entry.ActorClass, ExtensionHandler);
            ActiveData.ComponentRequests.Add(RequestHandle);
        }
    }
}
```

**Actor 扩展处理：**

```cpp
void UGameFeatureAction_AddAbilities::HandleActorExtension(AActor* Actor, FName EventName, int32 EntryIndex, FGameFeatureStateChangeContext ChangeContext)
{
    FPerContextData* ActiveData = ContextData.Find(ChangeContext);
    if (!ActiveData)
        return;

    if (EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded || 
        EventName == UGameFrameworkComponentManager::NAME_GameActorReady)
    {
        AddActorAbilities(Actor, AbilitiesList[EntryIndex], *ActiveData);
    }
    else if (EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved || 
             EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved)
    {
        RemoveActorAbilities(Actor, *ActiveData);
    }
}
```

**添加 Abilities 的实现：**

```cpp
void UGameFeatureAction_AddAbilities::AddActorAbilities(AActor* Actor, const FGameFeatureAbilitiesEntry& AbilitiesEntry, FPerContextData& ActiveData)
{
    // 查找或添加 AbilitySystemComponent
    UAbilitySystemComponent* ASC = FindOrAddComponentForActor<UAbilitySystemComponent>(Actor, AbilitiesEntry, ActiveData);
    if (!ASC)
        return;

    FActorExtensions& Extensions = ActiveData.ActiveExtensions.FindOrAdd(Actor);

    // 1. 授予 Attribute Sets
    for (const FLyraAttributeSetGrant& AttributeGrant : AbilitiesEntry.GrantedAttributes)
    {
        if (UClass* AttributeClass = AttributeGrant.AttributeSetType.LoadSynchronous())
        {
            UAttributeSet* NewAttributeSet = NewObject<UAttributeSet>(ASC->GetOwner(), AttributeClass);
            ASC->AddAttributeSetSubobject(NewAttributeSet);
            Extensions.Attributes.Add(NewAttributeSet);

            // 初始化属性值（如果有数据表）
            if (AttributeGrant.InitializationData.IsValid())
            {
                UDataTable* DataTable = AttributeGrant.InitializationData.LoadSynchronous();
                ASC->InitStats(AttributeClass, DataTable);
            }
        }
    }

    // 2. 授予独立 Abilities
    for (const FLyraAbilityGrant& AbilityGrant : AbilitiesEntry.GrantedAbilities)
    {
        if (UClass* AbilityClass = AbilityGrant.AbilityType.LoadSynchronous())
        {
            FGameplayAbilitySpec AbilitySpec(AbilityClass);
            FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(AbilitySpec);
            Extensions.Abilities.Add(Handle);
        }
    }

    // 3. 授予 Ability Sets
    for (const TSoftObjectPtr<const ULyraAbilitySet>& SetPtr : AbilitiesEntry.GrantedAbilitySets)
    {
        if (const ULyraAbilitySet* Set = SetPtr.LoadSynchronous())
        {
            FLyraAbilitySet_GrantedHandles GrantedHandles;
            Set->GiveToAbilitySystem(ASC, &GrantedHandles);
            Extensions.AbilitySetHandles.Add(GrantedHandles);
        }
    }
}
```

### 27.3.2 GameFeatureAction_AddInputBinding

处理 Enhanced Input 系统的输入映射注入：

```cpp
UCLASS(MinimalAPI, meta = (DisplayName = "Add Input Binds"))
class UGameFeatureAction_AddInputBinding final : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category="Input")
    TArray<TSoftObjectPtr<const ULyraInputConfig>> InputConfigs;

private:
    struct FPerContextData
    {
        TArray<TSharedPtr<FComponentRequestHandle>> ExtensionRequestHandles;
        TArray<TWeakObjectPtr<APawn>> PawnsAddedTo;
    };

    TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;

    virtual void AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext) override;
    void HandlePawnExtension(AActor* Actor, FName EventName, FGameFeatureStateChangeContext ChangeContext);
    void AddInputMappingForPlayer(APawn* Pawn, FPerContextData& ActiveData);
    void RemoveInputMapping(APawn* Pawn, FPerContextData& ActiveData);
};
```

**关键实现：**

```cpp
void UGameFeatureAction_AddInputBinding::AddInputMappingForPlayer(APawn* Pawn, FPerContextData& ActiveData)
{
    APlayerController* PC = Cast<APlayerController>(Pawn->GetController());
    if (!PC)
        return;

    ULyraHeroComponent* HeroComp = Pawn->FindComponentByClass<ULyraHeroComponent>();
    if (!HeroComp)
        return;

    for (const TSoftObjectPtr<const ULyraInputConfig>& ConfigPtr : InputConfigs)
    {
        if (const ULyraInputConfig* Config = ConfigPtr.LoadSynchronous())
        {
            HeroComp->AddAdditionalInputConfig(Config);
        }
    }

    ActiveData.PawnsAddedTo.AddUnique(Pawn);
}
```

### 27.3.3 GameFeatureAction_AddWidgets

UI 扩展系统的实现：

```cpp
USTRUCT()
struct FLyraHUDLayoutRequest
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category=UI)
    TSoftClassPtr<UCommonActivatableWidget> LayoutClass;

    UPROPERTY(EditAnywhere, Category=UI, meta=(Categories="UI.Layer"))
    FGameplayTag LayerID;
};

USTRUCT()
struct FLyraHUDElementEntry
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category=UI)
    TSoftClassPtr<UUserWidget> WidgetClass;

    UPROPERTY(EditAnywhere, Category=UI)
    FGameplayTag SlotID;
};
```

**Widget 添加逻辑：**

```cpp
void UGameFeatureAction_AddWidgets::AddWidgets(AActor* Actor, FPerContextData& ActiveData)
{
    ULyraHUDLayoutSubsystem* HUDSubsystem = Actor->GetWorld()->GetSubsystem<ULyraHUDLayoutSubsystem>();
    if (!HUDSubsystem)
        return;

    FPerActorData& ActorData = ActiveData.ActorData.FindOrAdd(Actor);

    // 添加 Layout
    for (const FLyraHUDLayoutRequest& LayoutRequest : Layout)
    {
        if (TSubclassOf<UCommonActivatableWidget> LayoutClass = LayoutRequest.LayoutClass.LoadSynchronous())
        {
            UCommonActivatableWidget* LayoutWidget = HUDSubsystem->PushWidgetToLayer(LayoutRequest.LayerID, LayoutClass);
            ActorData.LayoutsAdded.Add(LayoutWidget);
        }
    }

    // 添加 HUD Elements
    for (const FLyraHUDElementEntry& WidgetEntry : Widgets)
    {
        if (TSubclassOf<UUserWidget> WidgetClass = WidgetEntry.WidgetClass.LoadSynchronous())
        {
            FUIExtensionHandle Handle = HUDSubsystem->AddWidgetToSlot(WidgetEntry.SlotID, WidgetClass);
            ActorData.ExtensionHandles.Add(Handle);
        }
    }
}
```

### 27.3.4 WorldActionBase 的执行流程

理解 `AddToWorld` 的调用时机：

```cpp
void UGameFeatureAction_WorldActionBase::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
    // 注册新 GameInstance 的监听器
    GameInstanceStartHandles.FindOrAdd(Context) = FWorldDelegates::OnStartGameInstance.AddUObject(
        this, 
        &UGameFeatureAction_WorldActionBase::HandleGameInstanceStart, 
        FGameFeatureStateChangeContext(Context)
    );

    // 遍历当前所有 World
    for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
    {
        if (Context.ShouldApplyToWorldContext(WorldContext))
        {
            AddToWorld(WorldContext, Context);
        }
    }
}

void UGameFeatureAction_WorldActionBase::HandleGameInstanceStart(UGameInstance* GameInstance, FGameFeatureStateChangeContext ChangeContext)
{
    if (FWorldContext* WorldContext = GameInstance->GetWorldContext())
    {
        if (ChangeContext.ShouldApplyToWorldContext(*WorldContext))
        {
            AddToWorld(*WorldContext, ChangeContext);
        }
    }
}
```

**World 上下文过滤：**

```cpp
bool FGameFeatureStateChangeContext::ShouldApplyToWorldContext(const FWorldContext& WorldContext) const
{
    // 根据 World Type 决定是否应用
    if (WorldContext.WorldType == EWorldType::Game || 
        WorldContext.WorldType == EWorldType::PIE)
    {
        return true;
    }

    // Editor Preview Worlds
    if (WorldContext.WorldType == EWorldType::Editor && bShouldApplyToEditor)
    {
        return true;
    }

    return false;
}
```

## 27.4 自定义 Action 开发流程

### 27.4.1 创建 Action 类的步骤

**步骤 1：定义头文件**

```cpp
// CustomGameFeatureAction_Weather.h
#pragma once

#include "GameFeatureAction_WorldActionBase.h"
#include "CustomGameFeatureAction_Weather.generated.h"

class AWeatherController;

USTRUCT(BlueprintType)
struct FWeatherSystemConfig
{
    GENERATED_BODY()

    /** 天气控制器类型 */
    UPROPERTY(EditAnywhere, Category="Weather")
    TSoftClassPtr<AWeatherController> WeatherControllerClass;

    /** 初始天气状态 */
    UPROPERTY(EditAnywhere, Category="Weather")
    FGameplayTag InitialWeatherState;

    /** 是否启用动态天气转换 */
    UPROPERTY(EditAnywhere, Category="Weather")
    bool bEnableDynamicTransitions = true;

    /** 天气转换间隔（秒） */
    UPROPERTY(EditAnywhere, Category="Weather", meta=(EditCondition="bEnableDynamicTransitions"))
    float TransitionInterval = 300.0f;
};

/**
 * 自定义天气系统 Game Feature Action
 * 在激活时生成天气控制器，在停用时清理
 */
UCLASS(meta=(DisplayName="Add Weather System"))
class YOURGAME_API UCustomGameFeatureAction_Weather : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    /** 天气系统配置 */
    UPROPERTY(EditAnywhere, Category="Weather", meta=(TitleProperty="WeatherControllerClass"))
    TArray<FWeatherSystemConfig> WeatherConfigs;

private:
    struct FPerContextData
    {
        /** 生成的天气控制器 */
        TArray<TWeakObjectPtr<AWeatherController>> SpawnedControllers;
    };

    TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;

    //~ Begin UGameFeatureAction_WorldActionBase Interface
    virtual void AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext) override;
    //~ End UGameFeatureAction_WorldActionBase Interface

    //~ Begin UGameFeatureAction Interface
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;
    //~ End UGameFeatureAction Interface

    void SpawnWeatherControllers(UWorld* World, FPerContextData& ActiveData);
    void CleanupWeatherControllers(FPerContextData& ActiveData);
};
```

**步骤 2：实现 CPP 文件**

```cpp
// CustomGameFeatureAction_Weather.cpp
#include "CustomGameFeatureAction_Weather.h"
#include "Engine/World.h"
#include "WeatherController.h"

void UCustomGameFeatureAction_Weather::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    if (!World || !World->IsGameWorld())
    {
        return;
    }

    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);
    SpawnWeatherControllers(World, ActiveData);
}

void UCustomGameFeatureAction_Weather::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    Super::OnGameFeatureDeactivating(Context);

    FPerContextData* ActiveData = ContextData.Find(Context);
    if (ActiveData)
    {
        CleanupWeatherControllers(*ActiveData);
        ContextData.Remove(Context);
    }
}

void UCustomGameFeatureAction_Weather::SpawnWeatherControllers(UWorld* World, FPerContextData& ActiveData)
{
    for (const FWeatherSystemConfig& Config : WeatherConfigs)
    {
        if (Config.WeatherControllerClass.IsNull())
        {
            continue;
        }

        // 加载天气控制器类
        UClass* ControllerClass = Config.WeatherControllerClass.LoadSynchronous();
        if (!ControllerClass)
        {
            UE_LOG(LogGameFeatures, Warning, TEXT("Failed to load WeatherController class: %s"), 
                   *Config.WeatherControllerClass.ToString());
            continue;
        }

        // 生成天气控制器
        FActorSpawnParameters SpawnParams;
        SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
        
        AWeatherController* Controller = World->SpawnActor<AWeatherController>(
            ControllerClass,
            FVector::ZeroVector,
            FRotator::ZeroRotator,
            SpawnParams
        );

        if (Controller)
        {
            // 初始化天气状态
            Controller->SetWeatherState(Config.InitialWeatherState);
            Controller->SetDynamicTransitionsEnabled(Config.bEnableDynamicTransitions);
            Controller->SetTransitionInterval(Config.TransitionInterval);

            ActiveData.SpawnedControllers.Add(Controller);

            UE_LOG(LogGameFeatures, Log, TEXT("Spawned WeatherController: %s"), 
                   *Controller->GetName());
        }
    }
}

void UCustomGameFeatureAction_Weather::CleanupWeatherControllers(FPerContextData& ActiveData)
{
    for (TWeakObjectPtr<AWeatherController>& ControllerPtr : ActiveData.SpawnedControllers)
    {
        if (AWeatherController* Controller = ControllerPtr.Get())
        {
            Controller->Destroy();
        }
    }

    ActiveData.SpawnedControllers.Empty();
}
```

### 27.4.2 与 World 的交互模式

**模式 1：Actor 生成**

```cpp
void UMyAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    
    FActorSpawnParameters SpawnParams;
    SpawnParams.Owner = World->GetFirstPlayerController();
    SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    
    AActor* SpawnedActor = World->SpawnActor<AActor>(ActorClass, Transform, SpawnParams);
}
```

**模式 2：组件注入（使用 GameFrameworkComponentManager）**

```cpp
void UMyAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UGameFrameworkComponentManager* ComponentManager = 
        UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(WorldContext.OwningGameInstance);

    if (!ComponentManager)
        return;

    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    // 注册组件扩展
    UGameFrameworkComponentManager::FExtensionHandlerDelegate Delegate = 
        UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
            this, 
            &UMyAction::HandleActorExtension, 
            ChangeContext
        );

    TSharedPtr<FComponentRequestHandle> Handle = ComponentManager->AddExtensionHandler(TargetActorClass, Delegate);
    ActiveData.ComponentRequests.Add(Handle);
}

void UMyAction::HandleActorExtension(AActor* Actor, FName EventName, FGameFeatureStateChangeContext ChangeContext)
{
    if (EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded)
    {
        // 添加组件
        UMyComponent* NewComp = NewObject<UMyComponent>(Actor);
        NewComp->RegisterComponent();
        Actor->AddInstanceComponent(NewComp);
    }
    else if (EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved)
    {
        // 移除组件
        if (UMyComponent* Comp = Actor->FindComponentByClass<UMyComponent>())
        {
            Comp->DestroyComponent();
        }
    }
}
```

**模式 3：Subsystem 交互**

```cpp
void UMyAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    
    // 获取自定义子系统
    UMyWorldSubsystem* Subsystem = World->GetSubsystem<UMyWorldSubsystem>();
    if (Subsystem)
    {
        Subsystem->RegisterFeature(FeatureData);
    }
}
```

### 27.4.3 Actor 遍历与组件注入的最佳实践

**方案 A：使用 Component Manager（推荐）**

优点：
- 自动处理 Actor 的生命周期
- 支持延迟注册（Actor 可能在 Action 激活后才生成）
- Lyra 原生支持，与 Experience 系统无缝集成

```cpp
void UMyAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UGameFrameworkComponentManager* ComponentManager = 
        UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(WorldContext.OwningGameInstance);

    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    // 为目标 Actor 类注册接收器
    for (const FMyActorExtensionConfig& Config : ExtensionConfigs)
    {
        // 添加接收器
        UGameFrameworkComponentManager::FExtensionHandlerDelegate Delegate = 
            UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
                this, &UMyAction::HandleActorExtension, ChangeContext
            );

        TSharedPtr<FComponentRequestHandle> Handle = 
            ComponentManager->AddExtensionHandler(Config.ActorClass, Delegate);
        
        ActiveData.ExtensionHandles.Add(Handle);
    }
}
```

**方案 B：手动遍历（仅在必要时使用）**

```cpp
void UMyAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    
    // 遍历现有 Actor
    for (TActorIterator<AMyTargetActor> It(World); It; ++It)
    {
        AMyTargetActor* Actor = *It;
        if (Actor && !Actor->IsPendingKillPending())
        {
            ApplyExtensionToActor(Actor);
        }
    }

    // 监听新 Actor 生成
    World->AddOnActorSpawnedHandler(
        FOnActorSpawned::FDelegate::CreateUObject(this, &UMyAction::OnActorSpawned, ChangeContext)
    );
}
```

### 27.4.4 Data Asset 配置

**创建 Data Asset 类：**

```cpp
// WeatherSystemData.h
UCLASS(BlueprintType)
class UWeatherSystemData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    /** 可用的天气状态 */
    UPROPERTY(EditAnywhere, Category="Weather")
    TMap<FGameplayTag, FWeatherStateConfig> WeatherStates;

    /** 天气转换规则 */
    UPROPERTY(EditAnywhere, Category="Weather")
    TArray<FWeatherTransitionRule> TransitionRules;

    /** 天气效果资产 */
    UPROPERTY(EditAnywhere, Category="Effects")
    TSoftObjectPtr<UNiagaraSystem> RainEffect;

    UPROPERTY(EditAnywhere, Category="Effects")
    TSoftObjectPtr<UNiagaraSystem> SnowEffect;
};
```

**在 Action 中使用 Data Asset：**

```cpp
UCLASS()
class UCustomGameFeatureAction_Weather : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    /** 天气系统数据资产 */
    UPROPERTY(EditAnywhere, Category="Weather")
    TSoftObjectPtr<UWeatherSystemData> WeatherData;

private:
    virtual void AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext) override
    {
        // 加载 Data Asset
        UWeatherSystemData* LoadedData = WeatherData.LoadSynchronous();
        if (!LoadedData)
        {
            UE_LOG(LogGameFeatures, Error, TEXT("Failed to load WeatherSystemData"));
            return;
        }

        // 使用数据配置天气系统
        // ...
    }
};
```

## 27.5 实战：天气系统插件

现在我们创建一个完整的天气系统 Game Feature 插件，包含动态天气生成、光照调整和后处理效果。

### 27.5.1 天气控制器 Actor

```cpp
// WeatherController.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "GameplayTagContainer.h"
#include "WeatherController.generated.h"

class UNiagaraComponent;
class UDirectionalLightComponent;
class UPostProcessComponent;

USTRUCT(BlueprintType)
struct FWeatherStateConfig
{
    GENERATED_BODY()

    /** 天气状态标签 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    FGameplayTag WeatherTag;

    /** 粒子特效 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TSoftObjectPtr<UNiagaraSystem> ParticleEffect;

    /** 太阳光强度 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    float SunIntensity = 10.0f;

    /** 太阳光颜色 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    FLinearColor SunColor = FLinearColor::White;

    /** 后处理设置 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    FPostProcessSettings PostProcessSettings;
};

UCLASS()
class WEATHERSYSTEM_API AWeatherController : public AActor
{
    GENERATED_BODY()
    
public:    
    AWeatherController();

protected:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

public:    
    virtual void Tick(float DeltaTime) override;

    /** 设置天气状态 */
    UFUNCTION(BlueprintCallable, Category="Weather")
    void SetWeatherState(FGameplayTag NewWeatherTag);

    /** 启用/禁用动态转换 */
    UFUNCTION(BlueprintCallable, Category="Weather")
    void SetDynamicTransitionsEnabled(bool bEnabled);

    /** 设置转换间隔 */
    UFUNCTION(BlueprintCallable, Category="Weather")
    void SetTransitionInterval(float Interval);

protected:
    /** 粒子效果组件 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    TObjectPtr<UNiagaraComponent> WeatherEffectComponent;

    /** 后处理组件 */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    TObjectPtr<UPostProcessComponent> PostProcessComponent;

    /** 当前天气状态 */
    UPROPERTY(BlueprintReadOnly, Category="Weather")
    FGameplayTag CurrentWeatherTag;

    /** 天气状态配置映射 */
    UPROPERTY(EditAnywhere, Category="Weather")
    TMap<FGameplayTag, FWeatherStateConfig> WeatherStates;

    /** 可用的天气标签列表 */
    UPROPERTY(EditAnywhere, Category="Weather")
    TArray<FGameplayTag> AvailableWeatherTags;

    /** 是否启用动态转换 */
    UPROPERTY(EditAnywhere, Category="Weather")
    bool bEnableDynamicTransitions = true;

    /** 转换间隔 */
    UPROPERTY(EditAnywhere, Category="Weather", meta=(EditCondition="bEnableDynamicTransitions"))
    float TransitionInterval = 300.0f;

private:
    /** 转换计时器 */
    float TransitionTimer = 0.0f;

    /** 场景中的主光源 */
    UPROPERTY()
    TObjectPtr<ADirectionalLight> MainDirectionalLight;

    void ApplyWeatherState(const FWeatherStateConfig& WeatherConfig);
    void UpdateLighting(const FWeatherStateConfig& WeatherConfig);
    void UpdatePostProcess(const FWeatherStateConfig& WeatherConfig);
    void UpdateParticleEffect(const FWeatherStateConfig& WeatherConfig);
    void FindMainDirectionalLight();
    void SelectRandomWeather();
};
```

```cpp
// WeatherController.cpp
#include "WeatherController.h"
#include "NiagaraComponent.h"
#include "NiagaraSystem.h"
#include "Components/PostProcessComponent.h"
#include "Engine/DirectionalLight.h"
#include "Components/DirectionalLightComponent.h"
#include "Kismet/GameplayStatics.h"

AWeatherController::AWeatherController()
{
    PrimaryActorTick.bCanEverTick = true;

    // 创建根组件
    USceneComponent* Root = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));
    SetRootComponent(Root);

    // 创建粒子效果组件
    WeatherEffectComponent = CreateDefaultSubobject<UNiagaraComponent>(TEXT("WeatherEffect"));
    WeatherEffectComponent->SetupAttachment(Root);
    WeatherEffectComponent->SetAutoActivate(false);

    // 创建后处理组件
    PostProcessComponent = CreateDefaultSubobject<UPostProcessComponent>(TEXT("PostProcess"));
    PostProcessComponent->bUnbound = true;  // 影响整个场景
    PostProcessComponent->Priority = 1.0f;
}

void AWeatherController::BeginPlay()
{
    Super::BeginPlay();

    FindMainDirectionalLight();

    // 初始化天气标签（如果未设置）
    if (!CurrentWeatherTag.IsValid() && AvailableWeatherTags.Num() > 0)
    {
        SetWeatherState(AvailableWeatherTags[0]);
    }
}

void AWeatherController::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    Super::EndPlay(EndPlayReason);

    // 清理粒子效果
    if (WeatherEffectComponent)
    {
        WeatherEffectComponent->Deactivate();
    }
}

void AWeatherController::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 处理动态天气转换
    if (bEnableDynamicTransitions && AvailableWeatherTags.Num() > 1)
    {
        TransitionTimer += DeltaTime;
        
        if (TransitionTimer >= TransitionInterval)
        {
            SelectRandomWeather();
            TransitionTimer = 0.0f;
        }
    }
}

void AWeatherController::SetWeatherState(FGameplayTag NewWeatherTag)
{
    if (!NewWeatherTag.IsValid())
    {
        UE_LOG(LogTemp, Warning, TEXT("Invalid weather tag"));
        return;
    }

    const FWeatherStateConfig* ConfigPtr = WeatherStates.Find(NewWeatherTag);
    if (!ConfigPtr)
    {
        UE_LOG(LogTemp, Warning, TEXT("Weather state not found: %s"), *NewWeatherTag.ToString());
        return;
    }

    CurrentWeatherTag = NewWeatherTag;
    ApplyWeatherState(*ConfigPtr);

    UE_LOG(LogTemp, Log, TEXT("Weather changed to: %s"), *NewWeatherTag.ToString());
}

void AWeatherController::SetDynamicTransitionsEnabled(bool bEnabled)
{
    bEnableDynamicTransitions = bEnabled;
    TransitionTimer = 0.0f;
}

void AWeatherController::SetTransitionInterval(float Interval)
{
    TransitionInterval = FMath::Max(Interval, 10.0f);  // 最小 10 秒
}

void AWeatherController::ApplyWeatherState(const FWeatherStateConfig& WeatherConfig)
{
    UpdateLighting(WeatherConfig);
    UpdatePostProcess(WeatherConfig);
    UpdateParticleEffect(WeatherConfig);
}

void AWeatherController::UpdateLighting(const FWeatherStateConfig& WeatherConfig)
{
    if (!MainDirectionalLight)
    {
        FindMainDirectionalLight();
    }

    if (MainDirectionalLight)
    {
        UDirectionalLightComponent* LightComp = MainDirectionalLight->GetComponent();
        if (LightComp)
        {
            LightComp->SetIntensity(WeatherConfig.SunIntensity);
            LightComp->SetLightColor(WeatherConfig.SunColor);
        }
    }
}

void AWeatherController::UpdatePostProcess(const FWeatherStateConfig& WeatherConfig)
{
    if (PostProcessComponent)
    {
        PostProcessComponent->Settings = WeatherConfig.PostProcessSettings;
    }
}

void AWeatherController::UpdateParticleEffect(const FWeatherStateConfig& WeatherConfig)
{
    if (!WeatherEffectComponent)
        return;

    // 停用当前效果
    WeatherEffectComponent->Deactivate();

    // 加载新的粒子系统
    if (!WeatherConfig.ParticleEffect.IsNull())
    {
        UNiagaraSystem* ParticleSystem = WeatherConfig.ParticleEffect.LoadSynchronous();
        if (ParticleSystem)
        {
            WeatherEffectComponent->SetAsset(ParticleSystem);
            WeatherEffectComponent->Activate(true);

            // 使粒子效果跟随玩家
            if (APlayerCameraManager* CameraManager = UGameplayStatics::GetPlayerCameraManager(this, 0))
            {
                FVector CameraLocation = CameraManager->GetCameraLocation();
                SetActorLocation(FVector(CameraLocation.X, CameraLocation.Y, CameraLocation.Z + 1000.0f));
            }
        }
    }
}

void AWeatherController::FindMainDirectionalLight()
{
    TArray<AActor*> FoundLights;
    UGameplayStatics::GetAllActorsOfClass(GetWorld(), ADirectionalLight::StaticClass(), FoundLights);

    if (FoundLights.Num() > 0)
    {
        MainDirectionalLight = Cast<ADirectionalLight>(FoundLights[0]);
    }
}

void AWeatherController::SelectRandomWeather()
{
    if (AvailableWeatherTags.Num() == 0)
        return;

    // 选择一个不同于当前的天气
    TArray<FGameplayTag> OtherWeathers;
    for (const FGameplayTag& Tag : AvailableWeatherTags)
    {
        if (Tag != CurrentWeatherTag)
        {
            OtherWeathers.Add(Tag);
        }
    }

    if (OtherWeathers.Num() > 0)
    {
        int32 RandomIndex = FMath::RandRange(0, OtherWeathers.Num() - 1);
        SetWeatherState(OtherWeathers[RandomIndex]);
    }
}
```

### 27.5.2 天气系统 Game Feature Action（完整实现）

```cpp
// GameFeatureAction_AddWeatherSystem.h
#pragma once

#include "CoreMinimal.h"
#include "GameFeatureAction_WorldActionBase.h"
#include "GameplayTagContainer.h"
#include "GameFeatureAction_AddWeatherSystem.generated.h"

class AWeatherController;
class UNiagaraSystem;

USTRUCT(BlueprintType)
struct FWeatherSystemEntry
{
    GENERATED_BODY()

    /** 天气控制器类 */
    UPROPERTY(EditAnywhere, Category="Weather", meta=(AssetBundles="Client,Server"))
    TSoftClassPtr<AWeatherController> WeatherControllerClass;

    /** 初始天气状态 */
    UPROPERTY(EditAnywhere, Category="Weather", meta=(Categories="Weather"))
    FGameplayTag InitialWeatherState;

    /** 启用动态转换 */
    UPROPERTY(EditAnywhere, Category="Weather")
    bool bEnableDynamicTransitions = true;

    /** 转换间隔（秒） */
    UPROPERTY(EditAnywhere, Category="Weather", meta=(EditCondition="bEnableDynamicTransitions", ClampMin="10.0"))
    float TransitionInterval = 300.0f;

    /** 天气状态配置 */
    UPROPERTY(EditAnywhere, Category="Weather")
    TMap<FGameplayTag, FWeatherStateConfig> WeatherStates;
};

/**
 * Game Feature Action：添加天气系统
 * 在激活时生成 WeatherController，在停用时清理
 */
UCLASS(meta=(DisplayName="Add Weather System"))
class YOURGAME_API UGameFeatureAction_AddWeatherSystem : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    /** 天气系统配置列表 */
    UPROPERTY(EditAnywhere, Category="Weather", meta=(TitleProperty="WeatherControllerClass"))
    TArray<FWeatherSystemEntry> WeatherSystems;

private:
    struct FPerContextData
    {
        /** 生成的天气控制器 */
        TArray<TWeakObjectPtr<AWeatherController>> SpawnedControllers;
    };

    TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;

    //~ Begin UGameFeatureAction_WorldActionBase Interface
    virtual void AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext) override;
    //~ End UGameFeatureAction_WorldActionBase Interface

    //~ Begin UGameFeatureAction Interface
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;
    //~ End UGameFeatureAction Interface

    void SpawnWeatherControllers(UWorld* World, const FGameFeatureStateChangeContext& ChangeContext);
    void CleanupWeatherControllers(const FGameFeatureStateChangeContext& ChangeContext);
};
```

```cpp
// GameFeatureAction_AddWeatherSystem.cpp
#include "GameFeatureAction_AddWeatherSystem.h"
#include "WeatherController.h"
#include "Engine/World.h"

void UGameFeatureAction_AddWeatherSystem::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    if (!World || !World->IsGameWorld())
    {
        return;
    }

    SpawnWeatherControllers(World, ChangeContext);
}

void UGameFeatureAction_AddWeatherSystem::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    Super::OnGameFeatureDeactivating(Context);
    CleanupWeatherControllers(Context);
}

void UGameFeatureAction_AddWeatherSystem::SpawnWeatherControllers(UWorld* World, const FGameFeatureStateChangeContext& ChangeContext)
{
    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    for (const FWeatherSystemEntry& Entry : WeatherSystems)
    {
        if (Entry.WeatherControllerClass.IsNull())
        {
            continue;
        }

        // 同步加载天气控制器类
        UClass* ControllerClass = Entry.WeatherControllerClass.LoadSynchronous();
        if (!ControllerClass)
        {
            UE_LOG(LogGameFeatures, Warning, TEXT("[WeatherSystem] Failed to load controller class: %s"), 
                   *Entry.WeatherControllerClass.ToString());
            continue;
        }

        // 生成天气控制器
        FActorSpawnParameters SpawnParams;
        SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
        SpawnParams.Name = FName(*FString::Printf(TEXT("WeatherController_%d"), ActiveData.SpawnedControllers.Num()));

        AWeatherController* Controller = World->SpawnActor<AWeatherController>(
            ControllerClass,
            FVector(0.0f, 0.0f, 5000.0f),  // 高空位置
            FRotator::ZeroRotator,
            SpawnParams
        );

        if (Controller)
        {
            // 应用配置
            Controller->WeatherStates = Entry.WeatherStates;
            Controller->SetDynamicTransitionsEnabled(Entry.bEnableDynamicTransitions);
            Controller->SetTransitionInterval(Entry.TransitionInterval);

            // 设置初始天气
            if (Entry.InitialWeatherState.IsValid())
            {
                Controller->SetWeatherState(Entry.InitialWeatherState);
            }

            ActiveData.SpawnedControllers.Add(Controller);

            UE_LOG(LogGameFeatures, Log, TEXT("[WeatherSystem] Spawned controller: %s (Initial: %s)"), 
                   *Controller->GetName(),
                   *Entry.InitialWeatherState.ToString());
        }
        else
        {
            UE_LOG(LogGameFeatures, Error, TEXT("[WeatherSystem] Failed to spawn controller"));
        }
    }
}

void UGameFeatureAction_AddWeatherSystem::CleanupWeatherControllers(const FGameFeatureStateChangeContext& ChangeContext)
{
    FPerContextData* ActiveData = ContextData.Find(ChangeContext);
    if (!ActiveData)
    {
        return;
    }

    // 销毁所有生成的控制器
    for (TWeakObjectPtr<AWeatherController>& ControllerPtr : ActiveData->SpawnedControllers)
    {
        if (AWeatherController* Controller = ControllerPtr.Get())
        {
            UE_LOG(LogGameFeatures, Log, TEXT("[WeatherSystem] Destroying controller: %s"), 
                   *Controller->GetName());
            Controller->Destroy();
        }
    }

    ActiveData->SpawnedControllers.Empty();
    ContextData.Remove(ChangeContext);
}
```

### 27.5.3 插件配置文件

创建 Game Feature 插件描述文件：

```json
// Plugins/WeatherSystemFeature/WeatherSystemFeature.uplugin
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "1.0",
    "FriendlyName": "Weather System Feature",
    "Description": "Dynamic weather system for Lyra",
    "Category": "Game Features",
    "CreatedBy": "Your Studio",
    "CreatedByURL": "",
    "DocsURL": "",
    "MarketplaceURL": "",
    "SupportURL": "",
    "EnabledByDefault": false,
    "CanContainContent": true,
    "IsBetaVersion": false,
    "IsExperimentalVersion": false,
    "Installed": false,
    "ExplicitlyLoaded": true,
    "BuiltInInitialFeatureState": "Active",
    "Modules": [
        {
            "Name": "WeatherSystemRuntime",
            "Type": "Runtime",
            "LoadingPhase": "Default"
        }
    ],
    "Plugins": [
        {
            "Name": "GameFeatures",
            "Enabled": true
        },
        {
            "Name": "ModularGameplay",
            "Enabled": true
        }
    ]
}
```

Game Feature Data 配置：

```cpp
// Content/WeatherSystemFeature/WeatherSystemData.uasset (编辑器中创建)
// 在编辑器中创建 UGameFeatureData 资产，并添加 Actions：

Actions:
  [0] UGameFeatureAction_AddWeatherSystem
      Weather Systems:
        [0] Weather System Entry
            Weather Controller Class: /WeatherSystemFeature/Blueprints/BP_WeatherController
            Initial Weather State: Weather.Sunny
            Enable Dynamic Transitions: true
            Transition Interval: 180.0
            Weather States:
              Weather.Sunny:
                Weather Tag: Weather.Sunny
                Sun Intensity: 12.0
                Sun Color: (R=1.0, G=0.95, B=0.85, A=1.0)
                Particle Effect: None
              Weather.Rainy:
                Weather Tag: Weather.Rainy
                Sun Intensity: 4.0
                Sun Color: (R=0.6, G=0.65, B=0.7, A=1.0)
                Particle Effect: /WeatherSystemFeature/Effects/NS_Rain
              Weather.Stormy:
                Weather Tag: Weather.Stormy
                Sun Intensity: 2.0
                Sun Color: (R=0.4, G=0.4, B=0.45, A=1.0)
                Particle Effect: /WeatherSystemFeature/Effects/NS_Storm
```

## 27.6 实战：动态音乐系统

### 27.6.1 音乐管理器组件

```cpp
// MusicManagerComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "GameplayTagContainer.h"
#include "MusicManagerComponent.generated.h"

class USoundBase;
class UAudioComponent;

USTRUCT(BlueprintType)
struct FMusicTrack
{
    GENERATED_BODY()

    /** 音乐状态标签 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    FGameplayTag MusicStateTag;

    /** 音乐资产 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TSoftObjectPtr<USoundBase> MusicAsset;

    /** 淡入时间 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    float FadeInDuration = 2.0f;

    /** 淡出时间 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    float FadeOutDuration = 2.0f;

    /** 音量倍增 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    float VolumeMultiplier = 1.0f;
};

UCLASS(ClassGroup=(Audio), meta=(BlueprintSpawnableComponent))
class YOURGAME_API UMusicManagerComponent : public UActorComponent
{
    GENERATED_BODY()

public:    
    UMusicManagerComponent();

protected:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

public:
    /** 切换到指定音乐状态 */
    UFUNCTION(BlueprintCallable, Category="Music")
    void SwitchToMusicState(FGameplayTag MusicStateTag);

    /** 停止当前音乐 */
    UFUNCTION(BlueprintCallable, Category="Music")
    void StopCurrentMusic();

    /** 注册音乐轨道 */
    UFUNCTION(BlueprintCallable, Category="Music")
    void RegisterMusicTrack(const FMusicTrack& Track);

protected:
    /** 音乐轨道映射 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Music")
    TMap<FGameplayTag, FMusicTrack> MusicTracks;

    /** 当前播放的音乐状态 */
    UPROPERTY(BlueprintReadOnly, Category="Music")
    FGameplayTag CurrentMusicState;

private:
    /** 音频组件 */
    UPROPERTY()
    TObjectPtr<UAudioComponent> AudioComponent;

    void PlayMusicTrack(const FMusicTrack& Track);
};
```

```cpp
// MusicManagerComponent.cpp
#include "MusicManagerComponent.h"
#include "Components/AudioComponent.h"
#include "Sound/SoundBase.h"
#include "Kismet/GameplayStatics.h"

UMusicManagerComponent::UMusicManagerComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

void UMusicManagerComponent::BeginPlay()
{
    Super::BeginPlay();

    // 创建音频组件
    AudioComponent = NewObject<UAudioComponent>(GetOwner());
    if (AudioComponent)
    {
        AudioComponent->RegisterComponent();
        AudioComponent->bAutoActivate = false;
    }
}

void UMusicManagerComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    Super::EndPlay(EndPlayReason);

    if (AudioComponent)
    {
        AudioComponent->Stop();
        AudioComponent->DestroyComponent();
    }
}

void UMusicManagerComponent::SwitchToMusicState(FGameplayTag MusicStateTag)
{
    if (!MusicStateTag.IsValid())
    {
        UE_LOG(LogTemp, Warning, TEXT("[MusicManager] Invalid music state tag"));
        return;
    }

    // 避免重复播放
    if (CurrentMusicState == MusicStateTag && AudioComponent && AudioComponent->IsPlaying())
    {
        return;
    }

    const FMusicTrack* TrackPtr = MusicTracks.Find(MusicStateTag);
    if (!TrackPtr)
    {
        UE_LOG(LogTemp, Warning, TEXT("[MusicManager] Music track not found: %s"), *MusicStateTag.ToString());
        return;
    }

    CurrentMusicState = MusicStateTag;
    PlayMusicTrack(*TrackPtr);

    UE_LOG(LogTemp, Log, TEXT("[MusicManager] Switched to music state: %s"), *MusicStateTag.ToString());
}

void UMusicManagerComponent::StopCurrentMusic()
{
    if (AudioComponent && AudioComponent->IsPlaying())
    {
        const FMusicTrack* CurrentTrack = MusicTracks.Find(CurrentMusicState);
        float FadeOutTime = CurrentTrack ? CurrentTrack->FadeOutDuration : 2.0f;
        
        AudioComponent->FadeOut(FadeOutTime, 0.0f);
    }

    CurrentMusicState = FGameplayTag::EmptyTag;
}

void UMusicManagerComponent::RegisterMusicTrack(const FMusicTrack& Track)
{
    if (Track.MusicStateTag.IsValid())
    {
        MusicTracks.Add(Track.MusicStateTag, Track);
    }
}

void UMusicManagerComponent::PlayMusicTrack(const FMusicTrack& Track)
{
    if (!AudioComponent)
    {
        UE_LOG(LogTemp, Error, TEXT("[MusicManager] AudioComponent is null"));
        return;
    }

    // 停止当前音乐
    if (AudioComponent->IsPlaying())
    {
        AudioComponent->FadeOut(Track.FadeOutDuration, 0.0f);
    }

    // 加载音乐资产
    USoundBase* MusicSound = Track.MusicAsset.LoadSynchronous();
    if (!MusicSound)
    {
        UE_LOG(LogTemp, Error, TEXT("[MusicManager] Failed to load music asset: %s"), 
               *Track.MusicAsset.ToString());
        return;
    }

    // 播放新音乐
    AudioComponent->SetSound(MusicSound);
    AudioComponent->SetVolumeMultiplier(Track.VolumeMultiplier);
    AudioComponent->FadeIn(Track.FadeInDuration);
}
```

### 27.6.2 音乐系统 Game Feature Action

```cpp
// GameFeatureAction_AddMusicSystem.h
#pragma once

#include "CoreMinimal.h"
#include "GameFeatureAction_WorldActionBase.h"
#include "GameplayTagContainer.h"
#include "MusicManagerComponent.h"
#include "GameFeatureAction_AddMusicSystem.generated.h"

struct FComponentRequestHandle;

USTRUCT(BlueprintType)
struct FMusicSystemConfig
{
    GENERATED_BODY()

    /** 目标 Actor 类（通常是 PlayerController 或 GameState） */
    UPROPERTY(EditAnywhere, Category="Music")
    TSoftClassPtr<AActor> TargetActorClass;

    /** 音乐轨道列表 */
    UPROPERTY(EditAnywhere, Category="Music")
    TArray<FMusicTrack> MusicTracks;

    /** 初始音乐状态 */
    UPROPERTY(EditAnywhere, Category="Music", meta=(Categories="Music"))
    FGameplayTag InitialMusicState;
};

/**
 * Game Feature Action：添加动态音乐系统
 * 向指定 Actor 添加 MusicManagerComponent 并配置音乐轨道
 */
UCLASS(meta=(DisplayName="Add Music System"))
class YOURGAME_API UGameFeatureAction_AddMusicSystem : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    /** 音乐系统配置列表 */
    UPROPERTY(EditAnywhere, Category="Music", meta=(TitleProperty="TargetActorClass"))
    TArray<FMusicSystemConfig> MusicSystemConfigs;

private:
    struct FPerContextData
    {
        TArray<TSharedPtr<FComponentRequestHandle>> ComponentRequests;
        TMap<AActor*, TWeakObjectPtr<UMusicManagerComponent>> AddedComponents;
    };

    TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;

    //~ Begin UGameFeatureAction_WorldActionBase Interface
    virtual void AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext) override;
    //~ End UGameFeatureAction_WorldActionBase Interface

    //~ Begin UGameFeatureAction Interface
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;
    //~ End UGameFeatureAction Interface

    void HandleActorExtension(AActor* Actor, FName EventName, int32 ConfigIndex, FGameFeatureStateChangeContext ChangeContext);
    void AddMusicManagerToActor(AActor* Actor, const FMusicSystemConfig& Config, FPerContextData& ActiveData);
    void RemoveMusicManagerFromActor(AActor* Actor, FPerContextData& ActiveData);
};
```

```cpp
// GameFeatureAction_AddMusicSystem.cpp
#include "GameFeatureAction_AddMusicSystem.h"
#include "Components/GameFrameworkComponentManager.h"
#include "Engine/World.h"
#include "GameFramework/PlayerController.h"

void UGameFeatureAction_AddMusicSystem::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    if (!World || !World->IsGameWorld())
    {
        return;
    }

    UGameFrameworkComponentManager* ComponentManager = 
        UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(WorldContext.OwningGameInstance);

    if (!ComponentManager)
    {
        UE_LOG(LogGameFeatures, Warning, TEXT("[MusicSystem] ComponentManager not found"));
        return;
    }

    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    // 为每个配置注册组件扩展
    for (int32 ConfigIndex = 0; ConfigIndex < MusicSystemConfigs.Num(); ++ConfigIndex)
    {
        const FMusicSystemConfig& Config = MusicSystemConfigs[ConfigIndex];

        if (Config.TargetActorClass.IsNull())
        {
            continue;
        }

        UGameFrameworkComponentManager::FExtensionHandlerDelegate Delegate = 
            UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
                this, 
                &UGameFeatureAction_AddMusicSystem::HandleActorExtension, 
                ConfigIndex, 
                ChangeContext
            );

        TSharedPtr<FComponentRequestHandle> RequestHandle = 
            ComponentManager->AddExtensionHandler(Config.TargetActorClass, Delegate);

        ActiveData.ComponentRequests.Add(RequestHandle);
    }
}

void UGameFeatureAction_AddMusicSystem::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    Super::OnGameFeatureDeactivating(Context);

    FPerContextData* ActiveData = ContextData.Find(Context);
    if (ActiveData)
    {
        // 移除所有添加的组件
        for (auto& Pair : ActiveData->AddedComponents)
        {
            if (AActor* Actor = Pair.Key)
            {
                RemoveMusicManagerFromActor(Actor, *ActiveData);
            }
        }

        ActiveData->ComponentRequests.Empty();
        ActiveData->AddedComponents.Empty();
        ContextData.Remove(Context);
    }
}

void UGameFeatureAction_AddMusicSystem::HandleActorExtension(AActor* Actor, FName EventName, int32 ConfigIndex, FGameFeatureStateChangeContext ChangeContext)
{
    FPerContextData* ActiveData = ContextData.Find(ChangeContext);
    if (!ActiveData || !MusicSystemConfigs.IsValidIndex(ConfigIndex))
    {
        return;
    }

    const FMusicSystemConfig& Config = MusicSystemConfigs[ConfigIndex];

    if (EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded || 
        EventName == UGameFrameworkComponentManager::NAME_GameActorReady)
    {
        AddMusicManagerToActor(Actor, Config, *ActiveData);
    }
    else if (EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved || 
             EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved)
    {
        RemoveMusicManagerFromActor(Actor, *ActiveData);
    }
}

void UGameFeatureAction_AddMusicSystem::AddMusicManagerToActor(AActor* Actor, const FMusicSystemConfig& Config, FPerContextData& ActiveData)
{
    if (!Actor)
    {
        return;
    }

    // 检查是否已添加
    if (ActiveData.AddedComponents.Contains(Actor))
    {
        return;
    }

    // 创建 MusicManagerComponent
    UMusicManagerComponent* MusicManager = NewObject<UMusicManagerComponent>(Actor, UMusicManagerComponent::StaticClass());
    if (!MusicManager)
    {
        UE_LOG(LogGameFeatures, Error, TEXT("[MusicSystem] Failed to create MusicManagerComponent"));
        return;
    }

    // 注册并添加到 Actor
    MusicManager->RegisterComponent();
    Actor->AddInstanceComponent(MusicManager);

    // 注册所有音乐轨道
    for (const FMusicTrack& Track : Config.MusicTracks)
    {
        MusicManager->RegisterMusicTrack(Track);
    }

    // 播放初始音乐
    if (Config.InitialMusicState.IsValid())
    {
        MusicManager->SwitchToMusicState(Config.InitialMusicState);
    }

    ActiveData.AddedComponents.Add(Actor, MusicManager);

    UE_LOG(LogGameFeatures, Log, TEXT("[MusicSystem] Added MusicManager to %s"), *Actor->GetName());
}

void UGameFeatureAction_AddMusicSystem::RemoveMusicManagerFromActor(AActor* Actor, FPerContextData& ActiveData)
{
    TWeakObjectPtr<UMusicManagerComponent>* ComponentPtr = ActiveData.AddedComponents.Find(Actor);
    if (!ComponentPtr)
    {
        return;
    }

    if (UMusicManagerComponent* MusicManager = ComponentPtr->Get())
    {
        MusicManager->StopCurrentMusic();
        MusicManager->DestroyComponent();

        UE_LOG(LogGameFeatures, Log, TEXT("[MusicSystem] Removed MusicManager from %s"), *Actor->GetName());
    }

    ActiveData.AddedComponents.Remove(Actor);
}
```

## 27.7 实战：Procedural 内容生成

### 27.7.1 装饰物生成器 Actor

```cpp
// ProceduralDecorSpawner.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "ProceduralDecorSpawner.generated.h"

USTRUCT(BlueprintType)
struct FDecorSpawnRule
{
    GENERATED_BODY()

    /** 装饰物类 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TSoftClassPtr<AActor> DecorClass;

    /** 生成密度（每平方米） */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, meta=(ClampMin="0.01", ClampMax="10.0"))
    float SpawnDensity = 0.1f;

    /** 随机缩放范围 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    FVector2D ScaleRange = FVector2D(0.8f, 1.2f);

    /** 随机旋转（Yaw） */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    bool bRandomRotation = true;

    /** 对齐到地面 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    bool bAlignToGround = true;
};

UCLASS()
class YOURGAME_API AProceduralDecorSpawner : public AActor
{
    GENERATED_BODY()
    
public:    
    AProceduralDecorSpawner();

protected:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

#if WITH_EDITOR
    virtual void OnConstruction(const FTransform& Transform) override;
#endif

public:
    /** 生成装饰物 */
    UFUNCTION(BlueprintCallable, Category="Procedural")
    void SpawnDecor();

    /** 清理所有生成的装饰物 */
    UFUNCTION(BlueprintCallable, Category="Procedural")
    void ClearDecor();

protected:
    /** 生成区域（Box） */
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    TObjectPtr<class UBoxComponent> SpawnVolume;

    /** 装饰物生成规则 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Procedural")
    TArray<FDecorSpawnRule> SpawnRules;

    /** 随机种子 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Procedural")
    int32 RandomSeed = 12345;

    /** 在编辑器中自动生成 */
    UPROPERTY(EditAnywhere, Category="Procedural")
    bool bAutoSpawnInEditor = false;

private:
    /** 生成的装饰物列表 */
    UPROPERTY()
    TArray<TObjectPtr<AActor>> SpawnedActors;

    void SpawnDecorFromRule(const FDecorSpawnRule& Rule, FRandomStream& RandomStream);
    bool TraceToGround(const FVector& StartLocation, FVector& OutGroundLocation, FRotator& OutGroundRotation);
};
```

```cpp
// ProceduralDecorSpawner.cpp
#include "ProceduralDecorSpawner.h"
#include "Components/BoxComponent.h"
#include "Engine/World.h"
#include "DrawDebugHelpers.h"

AProceduralDecorSpawner::AProceduralDecorSpawner()
{
    PrimaryActorTick.bCanEverTick = false;

    // 创建生成区域
    SpawnVolume = CreateDefaultSubobject<UBoxComponent>(TEXT("SpawnVolume"));
    SetRootComponent(SpawnVolume);
    SpawnVolume->SetBoxExtent(FVector(1000.0f, 1000.0f, 500.0f));
    SpawnVolume->SetCollisionEnabled(ECollisionEnabled::NoCollision);
}

void AProceduralDecorSpawner::BeginPlay()
{
    Super::BeginPlay();

    // 运行时自动生成
    SpawnDecor();
}

void AProceduralDecorSpawner::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    Super::EndPlay(EndPlayReason);
    ClearDecor();
}

#if WITH_EDITOR
void AProceduralDecorSpawner::OnConstruction(const FTransform& Transform)
{
    Super::OnConstruction(Transform);

    if (bAutoSpawnInEditor)
    {
        ClearDecor();
        SpawnDecor();
    }
}
#endif

void AProceduralDecorSpawner::SpawnDecor()
{
    if (!SpawnVolume || SpawnRules.Num() == 0)
    {
        return;
    }

    // 初始化随机流
    FRandomStream RandomStream(RandomSeed);

    // 按规则生成装饰物
    for (const FDecorSpawnRule& Rule : SpawnRules)
    {
        SpawnDecorFromRule(Rule, RandomStream);
    }

    UE_LOG(LogTemp, Log, TEXT("[DecorSpawner] Spawned %d decorative actors"), SpawnedActors.Num());
}

void AProceduralDecorSpawner::ClearDecor()
{
    for (AActor* Actor : SpawnedActors)
    {
        if (Actor && !Actor->IsPendingKillPending())
        {
            Actor->Destroy();
        }
    }

    SpawnedActors.Empty();
}

void AProceduralDecorSpawner::SpawnDecorFromRule(const FDecorSpawnRule& Rule, FRandomStream& RandomStream)
{
    if (Rule.DecorClass.IsNull())
    {
        return;
    }

    // 加载装饰物类
    UClass* DecorClass = Rule.DecorClass.LoadSynchronous();
    if (!DecorClass)
    {
        return;
    }

    // 计算生成区域
    FVector BoxExtent = SpawnVolume->GetScaledBoxExtent();
    FVector BoxOrigin = GetActorLocation();
    float AreaSize = (BoxExtent.X * 2.0f) * (BoxExtent.Y * 2.0f);  // 平方米

    // 计算生成数量
    int32 SpawnCount = FMath::RoundToInt(AreaSize * Rule.SpawnDensity);

    // 生成装饰物
    for (int32 i = 0; i < SpawnCount; ++i)
    {
        // 随机位置（在 Box 内）
        FVector RandomOffset(
            RandomStream.FRandRange(-BoxExtent.X, BoxExtent.X),
            RandomStream.FRandRange(-BoxExtent.Y, BoxExtent.Y),
            BoxExtent.Z  // 从顶部开始
        );

        FVector SpawnLocation = BoxOrigin + RandomOffset;
        FRotator SpawnRotation = FRotator::ZeroRotator;

        // 对齐到地面
        if (Rule.bAlignToGround)
        {
            FVector GroundLocation;
            FRotator GroundRotation;
            if (!TraceToGround(SpawnLocation, GroundLocation, GroundRotation))
            {
                continue;  // 未找到地面，跳过
            }

            SpawnLocation = GroundLocation;
            SpawnRotation = GroundRotation;
        }

        // 随机旋转
        if (Rule.bRandomRotation)
        {
            SpawnRotation.Yaw = RandomStream.FRandRange(0.0f, 360.0f);
        }

        // 生成 Actor
        FActorSpawnParameters SpawnParams;
        SpawnParams.Owner = this;
        SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButDontSpawnIfColliding;

        AActor* SpawnedActor = GetWorld()->SpawnActor<AActor>(
            DecorClass,
            SpawnLocation,
            SpawnRotation,
            SpawnParams
        );

        if (SpawnedActor)
        {
            // 随机缩放
            float RandomScale = RandomStream.FRandRange(Rule.ScaleRange.X, Rule.ScaleRange.Y);
            SpawnedActor->SetActorScale3D(FVector(RandomScale));

            SpawnedActors.Add(SpawnedActor);
        }
    }
}

bool AProceduralDecorSpawner::TraceToGround(const FVector& StartLocation, FVector& OutGroundLocation, FRotator& OutGroundRotation)
{
    FHitResult HitResult;
    FVector EndLocation = StartLocation - FVector(0.0f, 0.0f, 10000.0f);  // 向下追踪

    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this);

    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        StartLocation,
        EndLocation,
        ECC_WorldStatic,
        QueryParams
    );

    if (bHit)
    {
        OutGroundLocation = HitResult.Location;
        
        // 计算对齐到表面法线的旋转
        FVector UpVector = HitResult.Normal;
        FVector ForwardVector = FVector::ForwardVector;
        FVector RightVector = FVector::CrossProduct(UpVector, ForwardVector).GetSafeNormal();
        ForwardVector = FVector::CrossProduct(RightVector, UpVector);

        OutGroundRotation = FRotationMatrix::MakeFromXZ(ForwardVector, UpVector).Rotator();
        
        return true;
    }

    return false;
}
```

### 27.7.2 装饰物生成 Game Feature Action

```cpp
// GameFeatureAction_SpawnProceduralDecor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFeatureAction_WorldActionBase.h"
#include "ProceduralDecorSpawner.h"
#include "GameFeatureAction_SpawnProceduralDecor.generated.h"

USTRUCT(BlueprintType)
struct FProceduralDecorConfig
{
    GENERATED_BODY()

    /** 装饰物生成器类 */
    UPROPERTY(EditAnywhere, Category="Procedural")
    TSoftClassPtr<AProceduralDecorSpawner> SpawnerClass;

    /** 生成位置 */
    UPROPERTY(EditAnywhere, Category="Procedural")
    FTransform SpawnTransform;

    /** 生成规则 */
    UPROPERTY(EditAnywhere, Category="Procedural")
    TArray<FDecorSpawnRule> SpawnRules;

    /** 随机种子 */
    UPROPERTY(EditAnywhere, Category="Procedural")
    int32 RandomSeed = 0;
};

/**
 * Game Feature Action：生成程序化装饰物
 */
UCLASS(meta=(DisplayName="Spawn Procedural Decor"))
class YOURGAME_API UGameFeatureAction_SpawnProceduralDecor : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category="Procedural", meta=(TitleProperty="SpawnerClass"))
    TArray<FProceduralDecorConfig> DecorConfigs;

private:
    struct FPerContextData
    {
        TArray<TWeakObjectPtr<AProceduralDecorSpawner>> SpawnedSpawners;
    };

    TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;

    virtual void AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext) override;
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;
};
```

```cpp
// GameFeatureAction_SpawnProceduralDecor.cpp
#include "GameFeatureAction_SpawnProceduralDecor.h"
#include "Engine/World.h"

void UGameFeatureAction_SpawnProceduralDecor::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    if (!World || !World->IsGameWorld())
    {
        return;
    }

    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    for (const FProceduralDecorConfig& Config : DecorConfigs)
    {
        if (Config.SpawnerClass.IsNull())
        {
            continue;
        }

        UClass* SpawnerClass = Config.SpawnerClass.LoadSynchronous();
        if (!SpawnerClass)
        {
            continue;
        }

        // 生成 Spawner
        FActorSpawnParameters SpawnParams;
        SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

        AProceduralDecorSpawner* Spawner = World->SpawnActor<AProceduralDecorSpawner>(
            SpawnerClass,
            Config.SpawnTransform.GetLocation(),
            Config.SpawnTransform.Rotator(),
            SpawnParams
        );

        if (Spawner)
        {
            Spawner->SpawnRules = Config.SpawnRules;
            Spawner->RandomSeed = Config.RandomSeed;
            Spawner->SpawnDecor();

            ActiveData.SpawnedSpawners.Add(Spawner);
        }
    }
}

void UGameFeatureAction_SpawnProceduralDecor::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    Super::OnGameFeatureDeactivating(Context);

    FPerContextData* ActiveData = ContextData.Find(Context);
    if (ActiveData)
    {
        for (TWeakObjectPtr<AProceduralDecorSpawner>& SpawnerPtr : ActiveData->SpawnedSpawners)
        {
            if (AProceduralDecorSpawner* Spawner = SpawnerPtr.Get())
            {
                Spawner->ClearDecor();
                Spawner->Destroy();
            }
        }

        ActiveData->SpawnedSpawners.Empty();
        ContextData.Remove(Context);
    }
}
```

## 27.8 C++ vs 蓝图 Action 对比

### 27.8.1 C++ Action 的优势

**优势：**
1. **性能最优**：无蓝图虚拟机开销，直接编译为机器码
2. **完全控制**：可访问所有引擎 API，包括低级系统
3. **类型安全**：编译时检查，减少运行时错误
4. **版本控制友好**：文本格式，易于 diff 和 merge

**适用场景：**
- 性能敏感的操作（大量 Actor 遍历、频繁调用）
- 复杂的系统集成（Subsystem、ComponentManager）
- 需要访问引擎私有 API
- 团队有 C++ 经验

**示例：C++ 实现高性能 Tag 注入**

```cpp
// GameFeatureAction_AddGameplayTags.h
UCLASS(meta=(DisplayName="Add Gameplay Tags"))
class UGameFeatureAction_AddGameplayTags : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category="Tags")
    TSoftClassPtr<AActor> TargetActorClass;

    UPROPERTY(EditAnywhere, Category="Tags", meta=(Categories="Gameplay"))
    FGameplayTagContainer TagsToAdd;

private:
    struct FPerContextData
    {
        TArray<TSharedPtr<FComponentRequestHandle>> ComponentRequests;
        TMap<AActor*, FGameplayTagContainer> AddedTags;
    };

    TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;

    virtual void AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext) override;
    void HandleActorExtension(AActor* Actor, FName EventName, FGameFeatureStateChangeContext ChangeContext);
};
```

```cpp
// GameFeatureAction_AddGameplayTags.cpp
void UGameFeatureAction_AddGameplayTags::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UGameFrameworkComponentManager* ComponentManager = 
        UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(WorldContext.OwningGameInstance);

    if (!ComponentManager)
        return;

    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    UGameFrameworkComponentManager::FExtensionHandlerDelegate Delegate = 
        UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
            this, &UGameFeatureAction_AddGameplayTags::HandleActorExtension, ChangeContext
        );

    TSharedPtr<FComponentRequestHandle> Handle = 
        ComponentManager->AddExtensionHandler(TargetActorClass, Delegate);

    ActiveData.ComponentRequests.Add(Handle);
}

void UGameFeatureAction_AddGameplayTags::HandleActorExtension(AActor* Actor, FName EventName, FGameFeatureStateChangeContext ChangeContext)
{
    FPerContextData* ActiveData = ContextData.Find(ChangeContext);
    if (!ActiveData)
        return;

    if (EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded)
    {
        // 查找或添加 GameplayTagComponent
        UAbilitySystemComponent* ASC = Actor->FindComponentByClass<UAbilitySystemComponent>();
        if (ASC)
        {
            // 添加 Gameplay Tags（通过 Loose Tags）
            for (const FGameplayTag& Tag : TagsToAdd)
            {
                ASC->AddLooseGameplayTag(Tag);
            }

            ActiveData->AddedTags.Add(Actor, TagsToAdd);
        }
    }
    else if (EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved)
    {
        // 移除 Tags
        if (FGameplayTagContainer* Tags = ActiveData->AddedTags.Find(Actor))
        {
            if (UAbilitySystemComponent* ASC = Actor->FindComponentByClass<UAbilitySystemComponent>())
            {
                for (const FGameplayTag& Tag : *Tags)
                {
                    ASC->RemoveLooseGameplayTag(Tag);
                }
            }

            ActiveData->AddedTags.Remove(Actor);
        }
    }
}
```

### 27.8.2 蓝图 Action 的实现（通过 C++ 基类）

虽然不能直接创建纯蓝图 `GameFeatureAction`，但可以创建蓝图友好的 C++ 基类：

```cpp
// GameFeatureAction_BlueprintBase.h
UCLASS(Abstract, Blueprintable)
class UGameFeatureAction_BlueprintBase : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override
    {
        BP_OnActivating(Context.WorldContext.World());
    }

    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override
    {
        BP_OnDeactivating(Context.WorldContext.World());
    }

protected:
    /** 蓝图实现的激活逻辑 */
    UFUNCTION(BlueprintImplementableEvent, Category="GameFeature", meta=(DisplayName="On Activating"))
    void BP_OnActivating(UWorld* World);

    /** 蓝图实现的停用逻辑 */
    UFUNCTION(BlueprintImplementableEvent, Category="GameFeature", meta=(DisplayName="On Deactivating"))
    void BP_OnDeactivating(UWorld* World);
};
```

**蓝图使用示例：**

1. 创建蓝图类继承自 `GameFeatureAction_BlueprintBase`
2. 实现 `On Activating` 事件：
   ```
   Event On Activating
       ├─ Get All Actors Of Class (Target Class: BP_LightPost)
       └─ ForEach Actor
           └─ Set Actor Hidden In Game (false)
   ```

3. 实现 `On Deactivating` 事件：
   ```
   Event On Deactivating
       ├─ Get All Actors Of Class (Target Class: BP_LightPost)
       └─ ForEach Actor
           └─ Set Actor Hidden In Game (true)
   ```

**蓝图 Action 的局限性：**
- 无法使用 `GameFrameworkComponentManager`（需 C++ 支持）
- 性能较低（大量 Actor 遍历会卡顿）
- 难以维护复杂状态
- 不支持多 World 上下文隔离

### 27.8.3 混合使用策略

**推荐方案：C++ 框架 + 蓝图配置**

```cpp
// GameFeatureAction_SpawnActors.h
UCLASS(meta=(DisplayName="Spawn Actors"))
class UGameFeatureAction_SpawnActors : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    /** 要生成的 Actor 配置（可在蓝图中配置） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Spawn")
    TArray<FActorSpawnConfig> ActorsToSpawn;

    /** 蓝图可覆盖的生成逻辑 */
    UFUNCTION(BlueprintNativeEvent, Category="Spawn")
    AActor* CustomSpawnActor(UWorld* World, const FActorSpawnConfig& Config);

protected:
    virtual AActor* CustomSpawnActor_Implementation(UWorld* World, const FActorSpawnConfig& Config)
    {
        // 默认实现
        return World->SpawnActor<AActor>(Config.ActorClass.LoadSynchronous(), Config.Transform);
    }

private:
    virtual void AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext) override;
    // ...
};
```

**优势：**
- C++ 处理复杂逻辑（生命周期、上下文管理）
- 蓝图配置数据（Actor 列表、参数）
- 蓝图可选择性覆盖部分逻辑（CustomSpawnActor）

## 27.9 进阶：多 World 与依赖管理

### 27.9.1 多 World 支持的实现

在 UE5 编辑器中，可能同时存在多个 World：

- **Editor World**：编辑器主视口
- **PIE Worlds**：Play In Editor 的多个实例（多人测试）
- **Game World**：打包后的游戏世界

**正确的多 World 处理：**

```cpp
void UMyGameFeatureAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();

    // 1. 验证 World 类型
    if (!World)
    {
        return;
    }

    // 2. 过滤不需要的 World 类型
    if (World->WorldType != EWorldType::Game && World->WorldType != EWorldType::PIE)
    {
        UE_LOG(LogGameFeatures, Verbose, TEXT("Skipping world type: %d"), (int32)World->WorldType);
        return;
    }

    // 3. 检查 GameInstance
    UGameInstance* GameInstance = WorldContext.OwningGameInstance;
    if (!GameInstance)
    {
        return;
    }

    // 4. 使用上下文隔离数据
    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    // 5. 执行 World 特定的操作
    // ...
}
```

**上下文隔离的关键：**

```cpp
// 错误示例：全局变量（多 World 冲突）
TArray<AActor*> SpawnedActors;  // ❌ 所有 World 共享

// 正确示例：上下文映射
struct FPerContextData
{
    TArray<AActor*> SpawnedActors;
};
TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;  // ✅ 每个 World 独立
```

### 27.9.2 Action 依赖管理

**问题场景：**
- Action A 需要 Action B 先执行（例如：音乐系统依赖 Audio Subsystem）
- 如何保证执行顺序？

**解决方案 1：使用 ComponentManager 的依赖特性**

```cpp
void UGameFeatureAction_AddMusicSystem::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UGameFrameworkComponentManager* ComponentManager = 
        UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(WorldContext.OwningGameInstance);

    // 等待 AudioSubsystem 初始化完成
    ComponentManager->AddExtensionHandler(
        AAudioManager::StaticClass(),
        UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
            this, &UGameFeatureAction_AddMusicSystem::OnAudioManagerReady, ChangeContext
        )
    );
}

void UGameFeatureAction_AddMusicSystem::OnAudioManagerReady(AActor* AudioManager, FName EventName, FGameFeatureStateChangeContext ChangeContext)
{
    if (EventName == UGameFrameworkComponentManager::NAME_GameActorReady)
    {
        // AudioManager 已就绪，现在可以初始化音乐系统
        InitializeMusicSystem(AudioManager, ChangeContext);
    }
}
```

**解决方案 2：异步等待 Subsystem**

```cpp
void UGameFeatureAction_MyAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();

    // 检查依赖的 Subsystem
    UMyRequiredSubsystem* Subsystem = World->GetSubsystem<UMyRequiredSubsystem>();
    if (!Subsystem || !Subsystem->IsInitialized())
    {
        // 延迟初始化（下一帧重试）
        FTimerHandle RetryHandle;
        World->GetTimerManager().SetTimer(
            RetryHandle,
            FTimerDelegate::CreateUObject(this, &UGameFeatureAction_MyAction::AddToWorld, WorldContext, ChangeContext),
            0.1f,  // 100ms 后重试
            false
        );
        return;
    }

    // Subsystem 已就绪，继续执行
    // ...
}
```

**解决方案 3：Experience 中控制 Action 顺序**

```cpp
// LyraExperienceDefinition 中的 Actions 按顺序执行
UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
TArray<TObjectPtr<UGameFeatureAction>> Actions;

// 执行顺序保证：
// Actions[0] → Actions[1] → Actions[2] → ...
```

### 27.9.3 热重载与调试技巧

**启用热重载：**

```cpp
// YourModule.Build.cs
PublicDefinitions.Add("UE_GAME_FEATURE_ACTION_HOT_RELOAD=1");
```

**调试日志宏：**

```cpp
#define GFA_LOG(Verbosity, Format, ...) \
    UE_LOG(LogGameFeatures, Verbosity, TEXT("[%s] " Format), *GetClass()->GetName(), ##__VA_ARGS__)

void UMyGameFeatureAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    GFA_LOG(Log, "Adding to world: %s (Context: %s)", 
            *WorldContext.World()->GetName(), 
            *ChangeContext.ToString());

    // ...
}
```

**使用编辑器命令调试：**

```cpp
// 控制台命令：
ListGameFeaturePlugins  // 列出所有 Game Feature 插件
LoadGameFeaturePlugin MyFeature  // 手动加载插件
UnloadGameFeaturePlugin MyFeature  // 手动卸载插件
```

**断点调试关键函数：**
- `UGameFeatureAction::OnGameFeatureActivating` - 激活入口
- `UGameFeatureAction_WorldActionBase::AddToWorld` - World 添加逻辑
- `UGameFrameworkComponentManager::HandleActorExtension` - 组件扩展回调

### 27.9.4 性能优化（延迟加载）

**问题：** 大量 Action 同时激活导致卡顿

**优化 1：异步资产加载**

```cpp
void UMyGameFeatureAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    FStreamableManager& Streamable = UAssetManager::GetStreamableManager();

    TArray<FSoftObjectPath> AssetsToLoad;
    for (const FMyAssetConfig& Config : AssetConfigs)
    {
        AssetsToLoad.Add(Config.Asset.ToSoftObjectPath());
    }

    // 异步加载（不阻塞主线程）
    FStreamableDelegate Delegate = FStreamableDelegate::CreateUObject(
        this, &UMyGameFeatureAction::OnAssetsLoaded, WorldContext, ChangeContext
    );

    StreamableHandle = Streamable.RequestAsyncLoad(AssetsToLoad, Delegate);
}

void UMyGameFeatureAction::OnAssetsLoaded(FWorldContext WorldContext, FGameFeatureStateChangeContext ChangeContext)
{
    // 资产加载完成，继续执行逻辑
    // ...
}
```

**优化 2：分帧生成**

```cpp
void UMyGameFeatureAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();

    // 不要一次生成所有 Actor
    // SpawnAllActors();  // ❌ 卡顿

    // 分帧生成
    FTimerHandle SpawnTimerHandle;
    int32 CurrentIndex = 0;

    World->GetTimerManager().SetTimer(
        SpawnTimerHandle,
        [this, World, CurrentIndex]() mutable {
            if (CurrentIndex < ActorsToSpawn.Num())
            {
                SpawnSingleActor(World, ActorsToSpawn[CurrentIndex]);
                CurrentIndex++;
            }
        },
        0.016f,  // 每帧生成一个（60fps）
        true     // 循环
    );
}
```

**优化 3：延迟初始化（Lazy Initialization）**

```cpp
class UMyGameFeatureAction : public UGameFeatureAction_WorldActionBase
{
private:
    mutable TOptional<FMyExpensiveData> CachedData;  // 延迟初始化

    const FMyExpensiveData& GetCachedData() const
    {
        if (!CachedData.IsSet())
        {
            CachedData = ComputeExpensiveData();  // 第一次访问时才计算
        }
        return CachedData.GetValue();
    }
};
```

## 27.10 调试、性能优化与最佳实践

### 27.10.1 调试工具与技巧

**1. Game Features 编辑器工具**

打开：`Window → Developer Tools → Game Features`

功能：
- 查看所有 Game Feature 插件状态
- 手动激活/停用插件
- 查看 Action 列表和执行状态

**2. 使用 Insights 分析性能**

```cpp
// 添加性能追踪标记
#include "ProfilingDebugging/CpuProfilerTrace.h"

void UMyGameFeatureAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(UMyGameFeatureAction::AddToWorld);

    // ... 你的代码
}
```

启动游戏时添加参数：`-trace=cpu,loadtime`

在 Unreal Insights 中查看：`Timing → CPU Track`

**3. 启用详细日志**

```cpp
// DefaultEngine.ini
[Core.Log]
LogGameFeatures=VeryVerbose
LogModularGameplay=Verbose
```

**4. 可视化调试**

```cpp
void UMyGameFeatureAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
#if ENABLE_DRAW_DEBUG
    UWorld* World = WorldContext.World();
    
    // 在 World 中绘制调试信息
    DrawDebugSphere(
        World,
        SpawnLocation,
        100.0f,
        12,
        FColor::Green,
        false,  // 非持久
        5.0f    // 显示 5 秒
    );

    // 屏幕消息
    if (GEngine)
    {
        GEngine->AddOnScreenDebugMessage(
            -1, 5.0f, FColor::Cyan,
            FString::Printf(TEXT("[%s] Activated"), *GetClass()->GetName())
        );
    }
#endif
}
```

### 27.10.2 性能优化清单

**✅ DO（推荐做法）：**

1. **使用 ComponentManager 而非手动遍历 Actor**
   ```cpp
   // ✅ 推荐
   ComponentManager->AddExtensionHandler(ActorClass, Delegate);

   // ❌ 避免
   for (TActorIterator<AActor> It(World); It; ++It) { ... }
   ```

2. **异步加载资产**
   ```cpp
   // ✅ 异步
   UAssetManager::GetStreamableManager().RequestAsyncLoad(AssetPath, Delegate);

   // ❌ 同步（阻塞主线程）
   Asset.LoadSynchronous();
   ```

3. **缓存常用查询**
   ```cpp
   // ✅ 缓存 Component Manager
   UGameFrameworkComponentManager* ComponentManager = 
       UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(GameInstance);

   // ❌ 每次都查询
   UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(GameInstance);
   ```

4. **正确清理资源**
   ```cpp
   void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override
   {
       Super::OnGameFeatureDeactivating(Context);

       // ✅ 清理所有创建的对象
       for (auto& Actor : SpawnedActors)
       {
           if (Actor)
           {
               Actor->Destroy();
           }
       }
       SpawnedActors.Empty();

       // ✅ 清理上下文数据
       ContextData.Remove(Context);
   }
   ```

5. **使用上下文隔离**
   ```cpp
   // ✅ 每个 World 独立数据
   TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;

   // ❌ 全局变量
   TArray<AActor*> GlobalSpawnedActors;
   ```

**❌ DON'T（避免的做法）：**

1. **不要在 Activating 阶段做耗时操作**
   ```cpp
   // ❌ 阻塞主线程
   for (int32 i = 0; i < 10000; ++i)
   {
       SpawnActor();
   }

   // ✅ 分帧执行
   AsyncTask(ENamedThreads::GameThread, [this]() { SpawnActorsBatch(); });
   ```

2. **不要忘记空指针检查**
   ```cpp
   // ❌ 可能崩溃
   ComponentManager->AddExtensionHandler(ActorClass, Delegate);

   // ✅ 安全检查
   if (ComponentManager && !ActorClass.IsNull())
   {
       ComponentManager->AddExtensionHandler(ActorClass, Delegate);
   }
   ```

3. **不要泄漏委托句柄**
   ```cpp
   // ❌ 忘记移除委托
   FWorldDelegates::OnStartGameInstance.AddUObject(this, &UMyAction::OnGameInstanceStart);

   // ✅ 保存句柄并在 Deactivating 时移除
   DelegateHandle = FWorldDelegates::OnStartGameInstance.AddUObject(...);
   // 在 Deactivating 中：
   FWorldDelegates::OnStartGameInstance.Remove(DelegateHandle);
   ```

4. **不要假设资产已加载**
   ```cpp
   // ❌ 可能返回 nullptr
   UClass* MyClass = MyAsset.Get();

   // ✅ 先加载
   UClass* MyClass = MyAsset.LoadSynchronous();
   if (MyClass) { ... }
   ```

### 27.10.3 Action 设计原则

**1. 单一职责原则**

每个 Action 只做一件事：

```cpp
// ✅ 好：职责清晰
UGameFeatureAction_AddWeatherSystem  // 只负责天气
UGameFeatureAction_AddMusicSystem    // 只负责音乐

// ❌ 差：职责混杂
UGameFeatureAction_AddEverything     // 天气 + 音乐 + UI + ...
```

**2. 可配置性**

使用 UPROPERTY 暴露配置参数：

```cpp
UCLASS()
class UMyGameFeatureAction : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

public:
    // ✅ 暴露配置
    UPROPERTY(EditAnywhere, Category="Config")
    int32 MaxSpawnCount = 100;

    UPROPERTY(EditAnywhere, Category="Config", meta=(ClampMin="0.0", ClampMax="1.0"))
    float SpawnProbability = 0.5f;

    UPROPERTY(EditAnywhere, Category="Config", meta=(AssetBundles="Client,Server"))
    TSoftClassPtr<AActor> ActorClass;
};
```

**3. 防御性编程**

始终验证输入和状态：

```cpp
void UMyGameFeatureAction::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();

    // ✅ 多重验证
    if (!World)
    {
        UE_LOG(LogGameFeatures, Error, TEXT("[%s] World is null"), *GetClass()->GetName());
        return;
    }

    if (!World->IsGameWorld())
    {
        UE_LOG(LogGameFeatures, Warning, TEXT("[%s] Not a game world, skipping"), *GetClass()->GetName());
        return;
    }

    UGameInstance* GameInstance = WorldContext.OwningGameInstance;
    if (!GameInstance)
    {
        UE_LOG(LogGameFeatures, Error, TEXT("[%s] GameInstance is null"), *GetClass()->GetName());
        return;
    }

    // 现在可以安全执行
    // ...
}
```

**4. 文档化**

```cpp
/**
 * Game Feature Action: 添加天气系统
 * 
 * 功能：
 * - 在 World 激活时生成 WeatherController
 * - 支持多种天气状态（晴天、雨天、暴风雨）
 * - 自动管理光照和后处理效果
 * 
 * 依赖：
 * - 需要 DirectionalLight 存在于场景中
 * - 需要 Niagara 插件
 * 
 * 配置：
 * - WeatherConfigs: 天气系统配置列表
 * - InitialWeatherState: 初始天气状态（默认：Weather.Sunny）
 * 
 * 注意事项：
 * - 停用时会自动销毁生成的 WeatherController
 * - 支持多 World（PIE/Game）
 * - 粒子效果异步加载，首次切换天气可能有延迟
 */
UCLASS(meta=(DisplayName="Add Weather System"))
class UGameFeatureAction_AddWeatherSystem : public UGameFeatureAction_WorldActionBase
{
    // ...
};
```

### 27.10.4 内存泄漏预防

**常见泄漏源：**

1. **未移除的委托**
2. **未销毁的 Actor/Component**
3. **未释放的 SharedPtr**
4. **循环引用**

**预防措施：**

```cpp
class UMyGameFeatureAction : public UGameFeatureAction_WorldActionBase
{
private:
    struct FPerContextData
    {
        // ✅ 使用 TWeakObjectPtr 避免阻止 GC
        TArray<TWeakObjectPtr<AActor>> SpawnedActors;

        // ✅ 使用 TSharedPtr 管理生命周期
        TArray<TSharedPtr<FComponentRequestHandle>> ComponentRequests;

        // ✅ 保存委托句柄以便移除
        TArray<FDelegateHandle> DelegateHandles;

        ~FPerContextData()
        {
            // ✅ 析构时自动清理
            ComponentRequests.Empty();
            DelegateHandles.Empty();
        }
    };

    TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;

    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override
    {
        Super::OnGameFeatureDeactivating(Context);

        FPerContextData* ActiveData = ContextData.Find(Context);
        if (ActiveData)
        {
            // ✅ 清理 Actors
            for (TWeakObjectPtr<AActor>& ActorPtr : ActiveData->SpawnedActors)
            {
                if (AActor* Actor = ActorPtr.Get())
                {
                    Actor->Destroy();
                }
            }

            // ✅ 移除委托
            for (FDelegateHandle& Handle : ActiveData->DelegateHandles)
            {
                if (Handle.IsValid())
                {
                    // 从对应的委托中移除
                    // FWorldDelegates::OnSomething.Remove(Handle);
                }
            }

            // ✅ 移除上下文数据
            ContextData.Remove(Context);
        }
    }
};
```

**使用 Unreal Insights 检测泄漏：**

1. 启动游戏：`-trace=memory`
2. 激活/停用插件多次
3. 在 Insights 中查看：`Memory Insights → Allocations`
4. 检查是否有持续增长的内存分配

## 27.11 总结

本章深入探讨了 Lyra 的 Game Feature Action 系统，这是 UE5 模块化游戏开发的核心机制。

**核心要点回顾：**

1. **架构理解**
   - `UGameFeatureAction` 是所有 Action 的基类
   - `UGameFeatureAction_WorldActionBase` 提供 World 感知能力
   - 生命周期：Registering → Loading → Activating → Deactivating → Unloading

2. **Lyra 内置 Actions**
   - `AddAbilities`：动态授予 Gameplay Abilities
   - `AddInputBinding`：注入 Enhanced Input 配置
   - `AddWidgets`：添加 UI 组件
   - 所有内置 Action 都使用 `GameFrameworkComponentManager` 实现组件注入

3. **自定义 Action 开发**
   - 继承 `WorldActionBase` 并实现 `AddToWorld`
   - 使用 `FPerContextData` 实现多 World 隔离
   - 通过 `ComponentManager` 注册 Actor 扩展
   - 在 `OnGameFeatureDeactivating` 中完全清理资源

4. **实战案例**
   - **天气系统**：动态生成 WeatherController，管理光照和粒子效果
   - **音乐系统**：通过 ComponentManager 向 Actor 注入音乐管理器
   - **程序化生成**：在 World 激活时生成装饰物，停用时清理

5. **进阶主题**
   - 多 World 支持通过上下文映射实现
   - Action 依赖可通过 ComponentManager 或异步等待解决
   - 性能优化：异步加载、分帧执行、延迟初始化

6. **最佳实践**
   - 单一职责：每个 Action 只做一件事
   - 防御性编程：验证所有输入和状态
   - 上下文隔离：使用 `TMap<Context, Data>` 管理多 World
   - 资源清理：在 Deactivating 中完全还原 Activating 的操作
   - 异步优先：避免阻塞主线程

**Game Feature Action 的威力：**

通过 Action 系统，你可以：
- ✅ 无需修改核心代码，动态添加游戏功能
- ✅ 实现热插拔式内容（DLC、季节性活动）
- ✅ 团队并行开发互不干扰
- ✅ 自动化资源管理和生命周期

**下一步学习：**

- 第28章：Experience System 深度解析
- 第29章：模块化多人游戏架构
- 第30章：Game Feature 插件打包与分发

Game Feature Action 是 Lyra 模块化设计的基石，掌握它将使你能够构建真正可扩展的 UE5 项目！

---

**参考资源：**

- Lyra 源码：`Source/LyraGame/GameFeatures/`
- Epic 官方文档：[Game Features and Modular Gameplay](https://docs.unrealengine.com/5.0/en-US/game-features-and-modular-gameplay-in-unreal-engine/)
- Lyra 示例插件：`Plugins/GameFeatures/ShooterCore/`

**字数统计：** 约 11,500 字

**代码示例：** 18 个完整示例（包含 .h + .cpp）

**实战案例：** 3 个完整系统（天气、音乐、程序化生成）
