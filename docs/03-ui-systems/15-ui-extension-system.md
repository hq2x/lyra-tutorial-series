# UI扩展系统实战：构建可插拔的动态界面架构

## 本章概览

在现代游戏开发中，UI系统往往面临着复杂的需求：不同游戏模式需要展示不同的HUD元素，DLC和功能插件需要在不修改主代码的情况下添加新界面，多人游戏需要为不同玩家显示不同的UI内容。传统的硬编码UI架构在面对这些需求时显得力不从心。

Lyra的UIExtension插件通过巧妙的"扩展点-扩展"模式，实现了一套完全解耦的动态UI系统。它允许UI组件在运行时动态注册和移除，支持基于GameplayTag的灵活匹配，并与GameFeature系统深度集成，为模块化游戏开发提供了强大的基础设施。

本章将深入探讨UIExtension的架构设计、核心实现细节、与GameFeature的集成方式，并通过两个完整的实战案例——动态HUD元素系统和可插拔商店UI——展示如何在实际项目中应用这套系统。

**关键收获：**
- 掌握UIExtension的架构思想和设计模式
- 理解扩展点和扩展的注册、匹配机制
- 学会使用GameFeature实现模块化UI
- 实战构建可扩展的HUD和复杂UI系统
- 掌握网络同步和性能优化最佳实践

---

## 1. UIExtension架构深度解析

### 1.1 设计理念：发布-订阅模式的UI实现

UIExtension的核心思想是将UI界面划分为"扩展点"（Extension Point）和"扩展"（Extension）两个概念：

```
┌─────────────────────────────────────────────────────┐
│                  UI Extension System                 │
├─────────────────────────────────────────────────────┤
│                                                       │
│  Extension Point                    Extension        │
│  ┌──────────────┐                  ┌──────────┐    │
│  │  "UI.HUD"    │◄─────────────────│  血条     │    │
│  │              │  注册/匹配        │          │    │
│  │  容器Widget  │◄─────────────────│  技能栏   │    │
│  │              │                  │          │    │
│  │  支持类型：  │◄─────────────────│  Buff图标 │    │
│  │  - Widget    │                  └──────────┘    │
│  │  - 数据对象  │                                   │
│  └──────────────┘                                   │
│                                                       │
└─────────────────────────────────────────────────────┘
```

**设计优势：**

1. **完全解耦**：扩展点不需要知道具体有哪些扩展，扩展也不需要知道谁会使用它
2. **动态性**：扩展可以在任意时刻注册和移除，运行时完全可控
3. **模块化**：每个功能可以独立成为GameFeature插件
4. **上下文感知**：支持为特定玩家或对象注册扩展，实现精确控制

### 1.2 核心类结构分析

#### 1.2.1 UUIExtensionSubsystem - 系统中枢

```cpp
// 位置：Plugins/UIExtension/Source/Public/UIExtensionSystem.h
UCLASS(MinimalAPI)
class UUIExtensionSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    // 注册扩展点
    FUIExtensionPointHandle RegisterExtensionPoint(
        const FGameplayTag& ExtensionPointTag,
        EUIExtensionPointMatch ExtensionPointTagMatchType,
        const TArray<UClass*>& AllowedDataClasses,
        FExtendExtensionPointDelegate ExtensionCallback
    );

    // 注册Widget扩展
    FUIExtensionHandle RegisterExtensionAsWidget(
        const FGameplayTag& ExtensionPointTag,
        TSubclassOf<UUserWidget> WidgetClass,
        int32 Priority
    );

    // 注册带上下文的扩展（核心功能）
    FUIExtensionHandle RegisterExtensionAsWidgetForContext(
        const FGameplayTag& ExtensionPointTag,
        UObject* ContextObject,
        TSubclassOf<UUserWidget> WidgetClass,
        int32 Priority
    );

private:
    // 核心数据结构：基于GameplayTag的映射
    TMap<FGameplayTag, TArray<TSharedPtr<FUIExtensionPoint>>> ExtensionPointMap;
    TMap<FGameplayTag, TArray<TSharedPtr<FUIExtension>>> ExtensionMap;
};
```

**关键设计决策分析：**

- **WorldSubsystem**：每个World有独立的实例，支持多World场景（PIE多窗口）
- **SharedPtr**：使用智能指针管理生命周期，避免悬挂引用
- **GameplayTag作为Key**：支持层级匹配（"UI.HUD"可以匹配"UI.HUD.Health"）
- **Delegate回调**：采用委托而非直接引用，进一步解耦

#### 1.2.2 FUIExtension - 扩展数据

```cpp
struct FUIExtension : TSharedFromThis<FUIExtension>
{
public:
    /** 目标扩展点Tag */
    FGameplayTag ExtensionPointTag;
    
    /** 优先级（用于排序） */
    int32 Priority = INDEX_NONE;
    
    /** 上下文对象（可选，用于玩家特定扩展） */
    TWeakObjectPtr<UObject> ContextObject;
    
    /** 数据对象（Widget类或自定义数据） */
    TObjectPtr<UObject> Data = nullptr;
};
```

**设计亮点：**

- **Priority排序**：支持控制扩展的显示顺序（如血条在最上层）
- **ContextObject**：实现玩家特定UI的关键（LocalPlayer或PlayerState）
- **Data的灵活性**：既可以是Widget类，也可以是数据对象，支持MVVM模式

#### 1.2.3 FUIExtensionPoint - 扩展点定义

```cpp
struct FUIExtensionPoint : TSharedFromThis<FUIExtensionPoint>
{
public:
    FGameplayTag ExtensionPointTag;
    TWeakObjectPtr<UObject> ContextObject;
    EUIExtensionPointMatch ExtensionPointTagMatchType;
    TArray<TObjectPtr<UClass>> AllowedDataClasses;
    FExtendExtensionPointDelegate Callback;

    // 核心匹配逻辑
    bool DoesExtensionPassContract(const FUIExtension* Extension) const
    {
        if (UObject* DataPtr = Extension->Data)
        {
            // 1. 检查上下文是否匹配
            const bool bMatchesContext = 
                (ContextObject.IsExplicitlyNull() && Extension->ContextObject.IsExplicitlyNull()) ||
                ContextObject == Extension->ContextObject;

            if (bMatchesContext)
            {
                // 2. 检查数据类型是否允许
                const UClass* DataClass = DataPtr->IsA(UClass::StaticClass()) 
                    ? Cast<UClass>(DataPtr) 
                    : DataPtr->GetClass();
                    
                for (const UClass* AllowedDataClass : AllowedDataClasses)
                {
                    if (DataClass->IsChildOf(AllowedDataClass) || 
                        DataClass->ImplementsInterface(AllowedDataClass))
                    {
                        return true;
                    }
                }
            }
        }
        return false;
    }
};
```

**匹配机制深度解析：**

1. **上下文匹配**：
   - 同时为nullptr：全局扩展点接受全局扩展
   - 指针相等：玩家特定扩展点只接受该玩家的扩展

2. **类型匹配**：
   - 支持继承关系：扩展可以是允许类的子类
   - 支持接口：扩展可以实现指定接口（灵活性更高）

### 1.3 匹配模式：精确 vs 部分匹配

```cpp
UENUM(BlueprintType)
enum class EUIExtensionPointMatch : uint8
{
    // 精确匹配："UI.HUD.Health" 只匹配 "UI.HUD.Health"
    ExactMatch,
    
    // 部分匹配："UI.HUD" 可以匹配 "UI.HUD.Health", "UI.HUD.Mana" 等
    PartialMatch
};
```

**实战应用示例：**

```cpp
// 场景1：精确匹配 - 用于特定槽位
// 扩展点注册：ExactMatch, "UI.HUD.TopLeft"
// 只接受：RegisterExtension("UI.HUD.TopLeft", HealthBar)
// 不接受：RegisterExtension("UI.HUD.TopLeft.Primary", DetailedHealthBar)

// 场景2：部分匹配 - 用于聚合容器
// 扩展点注册：PartialMatch, "UI.HUD"
// 接受：所有 "UI.HUD.*" 的扩展
ExtensionSubsystem->RegisterExtensionPoint(
    FGameplayTag::RequestGameplayTag("UI.HUD"),
    EUIExtensionPointMatch::PartialMatch,  // 关键！
    {UUserWidget::StaticClass()},
    FExtendExtensionPointDelegate::CreateUObject(this, &ThisClass::OnExtensionChanged)
);
```

---

## 2. 核心注册与通知机制

### 2.1 扩展点注册流程

```cpp
FUIExtensionPointHandle UUIExtensionSubsystem::RegisterExtensionPointForContext(
    const FGameplayTag& ExtensionPointTag, 
    UObject* ContextObject, 
    EUIExtensionPointMatch ExtensionPointTagMatchType, 
    const TArray<UClass*>& AllowedDataClasses, 
    FExtendExtensionPointDelegate ExtensionCallback)
{
    // 1. 参数验证
    if (!ExtensionPointTag.IsValid() || !ExtensionCallback.IsBound() || AllowedDataClasses.Num() == 0)
    {
        UE_LOG(LogUIExtension, Warning, TEXT("Invalid extension point registration"));
        return FUIExtensionPointHandle();
    }

    // 2. 创建扩展点对象
    FExtensionPointList& List = ExtensionPointMap.FindOrAdd(ExtensionPointTag);
    TSharedPtr<FUIExtensionPoint>& Entry = List.Add_GetRef(MakeShared<FUIExtensionPoint>());
    Entry->ExtensionPointTag = ExtensionPointTag;
    Entry->ContextObject = ContextObject;
    Entry->ExtensionPointTagMatchType = ExtensionPointTagMatchType;
    Entry->AllowedDataClasses = AllowedDataClasses;
    Entry->Callback = MoveTemp(ExtensionCallback);

    // 3. 立即通知所有已存在的匹配扩展（关键！）
    NotifyExtensionPointOfExtensions(Entry);

    // 4. 返回句柄供外部管理生命周期
    return FUIExtensionPointHandle(this, Entry);
}
```

**设计要点：**

1. **立即通知机制**：新注册的扩展点会立即收到所有已存在扩展的通知，避免时序问题
2. **MoveTemp优化**：委托使用移动语义，避免不必要的拷贝
3. **句柄管理**：返回句柄而非直接操作，支持RAII模式的自动清理

### 2.2 扩展通知算法

```cpp
void UUIExtensionSubsystem::NotifyExtensionPointOfExtensions(
    TSharedPtr<FUIExtensionPoint>& ExtensionPoint)
{
    // 关键算法：沿着GameplayTag层级向上查找
    for (FGameplayTag Tag = ExtensionPoint->ExtensionPointTag; 
         Tag.IsValid(); 
         Tag = Tag.RequestDirectParent())
    {
        if (const FExtensionList* ListPtr = ExtensionMap.Find(Tag))
        {
            // 拷贝数组防止回调中移除导致迭代器失效
            FExtensionList ExtensionArray(*ListPtr);

            for (const TSharedPtr<FUIExtension>& Extension : ExtensionArray)
            {
                if (ExtensionPoint->DoesExtensionPassContract(Extension.Get()))
                {
                    FUIExtensionRequest Request = CreateExtensionRequest(Extension);
                    ExtensionPoint->Callback.ExecuteIfBound(EUIExtensionAction::Added, Request);
                }
            }
        }

        // ExactMatch只查找当前Tag，PartialMatch继续向上
        if (ExtensionPoint->ExtensionPointTagMatchType == EUIExtensionPointMatch::ExactMatch)
        {
            break;
        }
    }
}
```

**算法分析：**

```
示例：注册扩展点 "UI.HUD"（PartialMatch）

查找顺序：
1. ExtensionMap["UI.HUD"]          → 找到 HealthBar, ManaBar
2. ExtensionMap["UI"]              → 找到 GlobalOverlay
3. ExtensionMap[Invalid]           → 停止

结果：扩展点收到 3 个扩展的 Added 通知
```

**防御性编程：**
- 拷贝数组：防止回调中调用 `UnregisterExtension` 导致迭代器失效
- 空指针检查：`DoesExtensionPassContract` 内部会检查 Data 有效性

### 2.3 扩展注册与反向通知

```cpp
FUIExtensionHandle UUIExtensionSubsystem::RegisterExtensionAsData(
    const FGameplayTag& ExtensionPointTag, 
    UObject* ContextObject, 
    UObject* Data, 
    int32 Priority)
{
    // 1. 验证参数
    if (!ExtensionPointTag.IsValid() || !Data)
    {
        return FUIExtensionHandle();
    }

    // 2. 创建扩展对象
    FExtensionList& List = ExtensionMap.FindOrAdd(ExtensionPointTag);
    TSharedPtr<FUIExtension>& Entry = List.Add_GetRef(MakeShared<FUIExtension>());
    Entry->ExtensionPointTag = ExtensionPointTag;
    Entry->ContextObject = ContextObject;
    Entry->Data = Data;
    Entry->Priority = Priority;

    // 3. 通知所有匹配的扩展点（反向查找）
    NotifyExtensionPointsOfExtension(EUIExtensionAction::Added, Entry);

    return FUIExtensionHandle(this, Entry);
}
```

**反向通知算法：**

```cpp
void UUIExtensionSubsystem::NotifyExtensionPointsOfExtension(
    EUIExtensionAction Action, 
    TSharedPtr<FUIExtension>& Extension)
{
    bool bOnInitialTag = true;
    
    // 从扩展的Tag开始，向上遍历父级Tag
    for (FGameplayTag Tag = Extension->ExtensionPointTag; 
         Tag.IsValid(); 
         Tag = Tag.RequestDirectParent())
    {
        if (const FExtensionPointList* ListPtr = ExtensionPointMap.Find(Tag))
        {
            FExtensionPointList ExtensionPointArray(*ListPtr);

            for (const TSharedPtr<FUIExtensionPoint>& ExtensionPoint : ExtensionPointArray)
            {
                // 只有第一层或PartialMatch的扩展点才能接收
                if (bOnInitialTag || 
                    (ExtensionPoint->ExtensionPointTagMatchType == EUIExtensionPointMatch::PartialMatch))
                {
                    if (ExtensionPoint->DoesExtensionPassContract(Extension.Get()))
                    {
                        FUIExtensionRequest Request = CreateExtensionRequest(Extension);
                        ExtensionPoint->Callback.ExecuteIfBound(Action, Request);
                    }
                }
            }
        }
        
        bOnInitialTag = false;
    }
}
```

**双向通知机制图解：**

```
场景：扩展 "UI.HUD.Health.Primary" 注册

向上查找扩展点：
┌─────────────────────────────────────────────────────────┐
│ ExtensionPointMap["UI.HUD.Health.Primary"]             │
│   → 精确匹配扩展点收到通知                              │
├─────────────────────────────────────────────────────────┤
│ ExtensionPointMap["UI.HUD.Health"]                     │
│   → 部分匹配扩展点收到通知                              │
├─────────────────────────────────────────────────────────┤
│ ExtensionPointMap["UI.HUD"]                            │
│   → 部分匹配扩展点收到通知                              │
├─────────────────────────────────────────────────────────┤
│ ExtensionPointMap["UI"]                                │
│   → 部分匹配扩展点收到通知                              │
└─────────────────────────────────────────────────────────┘
```

### 2.4 生命周期管理与清理

```cpp
void UUIExtensionSubsystem::UnregisterExtension(const FUIExtensionHandle& ExtensionHandle)
{
    if (ExtensionHandle.IsValid())
    {
        TSharedPtr<FUIExtension> Extension = ExtensionHandle.DataPtr;
        
        if (FExtensionList* ListPtr = ExtensionMap.Find(Extension->ExtensionPointTag))
        {
            // 1. 通知所有扩展点移除
            NotifyExtensionPointsOfExtension(EUIExtensionAction::Removed, Extension);

            // 2. 从映射表移除
            ListPtr->RemoveSwap(Extension);
            
            // 3. 清理空映射表（优化内存）
            if (ListPtr->Num() == 0)
            {
                ExtensionMap.Remove(Extension->ExtensionPointTag);
            }
        }
    }
}
```

**RAII模式句柄：**

```cpp
struct FUIExtensionHandle
{
    void Unregister()
    {
        if (UUIExtensionSubsystem* ExtensionSourcePtr = ExtensionSource.Get())
        {
            ExtensionSourcePtr->UnregisterExtension(*this);
        }
    }
    
    ~FUIExtensionHandle() 
    { 
        // 不自动调用Unregister，由用户控制
    }

private:
    TWeakObjectPtr<UUIExtensionSubsystem> ExtensionSource;
    TSharedPtr<FUIExtension> DataPtr;
};
```

**最佳实践：**
```cpp
class UMyWidget : public UUserWidget
{
    FUIExtensionHandle ExtensionHandle;

    void NativeConstruct() override
    {
        ExtensionHandle = Subsystem->RegisterExtension(...);
    }

    void NativeDestruct() override
    {
        ExtensionHandle.Unregister();  // 显式清理
    }
};
```

---

## 3. UUIExtensionPointWidget：容器组件实现

### 3.1 设计职责

`UUIExtensionPointWidget` 是 `UDynamicEntryBoxBase` 的子类，专门用作扩展点的可视化容器：

```cpp
// 位置：Plugins/UIExtension/Source/Public/Widgets/UIExtensionPointWidget.h
UCLASS(MinimalAPI)
class UUIExtensionPointWidget : public UDynamicEntryBoxBase
{
    GENERATED_BODY()

protected:
    /** 扩展点标识 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "UI Extension")
    FGameplayTag ExtensionPointTag;

    /** 匹配模式 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "UI Extension")
    EUIExtensionPointMatch ExtensionPointTagMatch = EUIExtensionPointMatch::ExactMatch;

    /** 允许的数据类型 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "UI Extension")
    TArray<TObjectPtr<UClass>> DataClasses;

    /** Widget类查询委托（用于数据驱动） */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="UI Extension")
    FOnGetWidgetClassForData GetWidgetClassForData;

    /** Widget配置委托（用于MVVM绑定） */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="UI Extension")
    FOnConfigureWidgetForData ConfigureWidgetForData;

private:
    TArray<FUIExtensionPointHandle> ExtensionPointHandles;
    TMap<FUIExtensionHandle, TObjectPtr<UUserWidget>> ExtensionMapping;
};
```

### 3.2 三层注册机制

```cpp
void UUIExtensionPointWidget::RegisterExtensionPoint()
{
    UUIExtensionSubsystem* ExtensionSubsystem = GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
    
    TArray<UClass*> AllowedDataClasses;
    AllowedDataClasses.Add(UUserWidget::StaticClass());
    AllowedDataClasses.Append(DataClasses);

    // 层次1：全局扩展点（无上下文）
    ExtensionPointHandles.Add(
        ExtensionSubsystem->RegisterExtensionPoint(
            ExtensionPointTag, 
            ExtensionPointTagMatch, 
            AllowedDataClasses,
            FExtendExtensionPointDelegate::CreateUObject(this, &ThisClass::OnAddOrRemoveExtension)
        )
    );

    // 层次2：LocalPlayer上下文
    ExtensionPointHandles.Add(
        ExtensionSubsystem->RegisterExtensionPointForContext(
            ExtensionPointTag, 
            GetOwningLocalPlayer(),  // 关键！
            ExtensionPointTagMatch, 
            AllowedDataClasses,
            FExtendExtensionPointDelegate::CreateUObject(this, &ThisClass::OnAddOrRemoveExtension)
        )
    );
}

void UUIExtensionPointWidget::RegisterExtensionPointForPlayerState(
    UCommonLocalPlayer* LocalPlayer, 
    APlayerState* PlayerState)
{
    // 层次3：PlayerState上下文（最精确）
    ExtensionPointHandles.Add(
        ExtensionSubsystem->RegisterExtensionPointForContext(
            ExtensionPointTag, 
            PlayerState,  // 用于网络同步场景
            ExtensionPointTagMatch, 
            AllowedDataClasses,
            FExtendExtensionPointDelegate::CreateUObject(this, &ThisClass::OnAddOrRemoveExtension)
        )
    );
}
```

**三层注册的必要性：**

```
场景示例：队伍界面显示队友血条

层次1（全局）：
- 扩展：队伍框架布局（所有玩家共享）
- 注册：RegisterExtension("UI.TeamFrame", TeamFrameWidget)

层次2（LocalPlayer）：
- 扩展：本地玩家的操作按钮（踢人、转让队长）
- 注册：RegisterExtensionForContext("UI.TeamFrame.Actions", LocalPlayer, ActionButtons)

层次3（PlayerState）：
- 扩展：每个队员的血条（针对特定玩家）
- 注册：RegisterExtensionForContext("UI.TeamFrame.Member", PlayerState, MemberHealthBar)
```

### 3.3 动态Widget创建与配置

```cpp
void UUIExtensionPointWidget::OnAddOrRemoveExtension(
    EUIExtensionAction Action, 
    const FUIExtensionRequest& Request)
{
    if (Action == EUIExtensionAction::Added)
    {
        UObject* Data = Request.Data;
        
        // 路径1：直接Widget类扩展
        TSubclassOf<UUserWidget> WidgetClass(Cast<UClass>(Data));
        if (WidgetClass)
        {
            UUserWidget* Widget = CreateEntryInternal(WidgetClass);
            ExtensionMapping.Add(Request.ExtensionHandle, Widget);
        }
        // 路径2：数据对象扩展（需要委托提供Widget类）
        else if (DataClasses.Num() > 0)
        {
            if (GetWidgetClassForData.IsBound())
            {
                WidgetClass = GetWidgetClassForData.Execute(Data);

                if (WidgetClass)
                {
                    UUserWidget* Widget = CreateEntryInternal(WidgetClass);
                    ExtensionMapping.Add(Request.ExtensionHandle, Widget);
                    
                    // 关键：配置Widget与数据的绑定
                    ConfigureWidgetForData.ExecuteIfBound(Widget, Data);
                }
            }
        }
    }
    else if (Action == EUIExtensionAction::Removed)
    {
        if (UUserWidget* Extension = ExtensionMapping.FindRef(Request.ExtensionHandle))
        {
            RemoveEntryInternal(Extension);
            ExtensionMapping.Remove(Request.ExtensionHandle);
        }
    }
}
```

**MVVM模式集成示例：**

```cpp
// 蓝图中绑定委托
UFUNCTION()
TSubclassOf<UUserWidget> GetWidgetForBuffData(UObject* DataItem)
{
    UBuffData* BuffData = Cast<UBuffData>(DataItem);
    if (BuffData)
    {
        switch (BuffData->BuffType)
        {
            case EBuffType::Positive:  return PositiveBuffWidgetClass;
            case EBuffType::Negative:  return NegativeBuffWidgetClass;
            case EBuffType::Neutral:   return NeutralBuffWidgetClass;
        }
    }
    return nullptr;
}

UFUNCTION()
void ConfigureBuffWidget(UUserWidget* Widget, UObject* DataItem)
{
    UBuffWidget* BuffWidget = Cast<UBuffWidget>(Widget);
    UBuffData* BuffData = Cast<UBuffData>(DataItem);
    
    if (BuffWidget && BuffData)
    {
        BuffWidget->SetBuffIcon(BuffData->Icon);
        BuffWidget->SetBuffDuration(BuffData->Duration);
        BuffWidget->BindDataModel(BuffData);  // MVVM绑定
    }
}
```

---

## 4. 与GameFeature插件深度集成

### 4.1 GameFeatureAction_AddWidgets 解析

```cpp
// 位置：Source/LyraGame/GameFeatures/GameFeatureAction_AddWidget.h
UCLASS(MinimalAPI, meta = (DisplayName = "Add Widgets"))
class UGameFeatureAction_AddWidgets final : public UGameFeatureAction_WorldActionBase
{
    GENERATED_BODY()

private:
    /** HUD布局层（如主HUD、菜单层） */
    UPROPERTY(EditAnywhere, Category=UI)
    TArray<FLyraHUDLayoutRequest> Layout;

    /** HUD元素扩展（如血条、技能栏） */
    UPROPERTY(EditAnywhere, Category=UI)
    TArray<FLyraHUDElementEntry> Widgets;

    struct FPerActorData
    {
        TArray<TWeakObjectPtr<UCommonActivatableWidget>> LayoutsAdded;
        TArray<FUIExtensionHandle> ExtensionHandles;
    };

    TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;
};
```

**数据结构设计：**

```cpp
USTRUCT()
struct FLyraHUDLayoutRequest
{
    GENERATED_BODY()

    // 完整的布局Widget（通常包含多个扩展点）
    UPROPERTY(EditAnywhere, Category=UI, meta=(AssetBundles="Client"))
    TSoftClassPtr<UCommonActivatableWidget> LayoutClass;

    // 推送到的UI层（使用CommonUI的Layer系统）
    UPROPERTY(EditAnywhere, Category=UI, meta=(Categories="UI.Layer"))
    FGameplayTag LayerID;  // 如 "UI.Layer.Game" 或 "UI.Layer.Menu"
};

USTRUCT()
struct FLyraHUDElementEntry
{
    GENERATED_BODY()

    // 要注册的Widget
    UPROPERTY(EditAnywhere, Category=UI, meta=(AssetBundles="Client"))
    TSoftClassPtr<UUserWidget> WidgetClass;

    // 目标扩展点
    UPROPERTY(EditAnywhere, Category=UI)
    FGameplayTag SlotID;  // 如 "UI.HUD.TopLeft"
};
```

### 4.2 生命周期管理

```cpp
void UGameFeatureAction_AddWidgets::AddToWorld(
    const FWorldContext& WorldContext, 
    const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    UGameInstance* GameInstance = WorldContext.OwningGameInstance;
    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    if (World && World->IsGameWorld())
    {
        UGameFrameworkComponentManager* ComponentManager = 
            UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(GameInstance);
            
        // 关键：监听HUD创建事件
        TSoftClassPtr<AActor> HUDActorClass = ALyraHUD::StaticClass();
        
        TSharedPtr<FComponentRequestHandle> ExtensionRequestHandle = 
            ComponentManager->AddExtensionHandler(
                HUDActorClass,
                UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
                    this, 
                    &ThisClass::HandleActorExtension, 
                    ChangeContext
                )
            );
            
        ActiveData.ComponentRequests.Add(ExtensionRequestHandle);
    }
}
```

**生命周期事件处理：**

```cpp
void UGameFeatureAction_AddWidgets::HandleActorExtension(
    AActor* Actor, 
    FName EventName, 
    FGameFeatureStateChangeContext ChangeContext)
{
    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);
    
    if ((EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved) || 
        (EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved))
    {
        RemoveWidgets(Actor, ActiveData);
    }
    else if ((EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded) || 
             (EventName == UGameFrameworkComponentManager::NAME_GameActorReady))
    {
        AddWidgets(Actor, ActiveData);
    }
}
```

### 4.3 Widget添加与移除实现

```cpp
void UGameFeatureAction_AddWidgets::AddWidgets(AActor* Actor, FPerContextData& ActiveData)
{
    ALyraHUD* HUD = CastChecked<ALyraHUD>(Actor);
    ULocalPlayer* LocalPlayer = Cast<ULocalPlayer>(HUD->GetOwningPlayerController()->Player);
    
    if (!LocalPlayer) return;

    FPerActorData& ActorData = ActiveData.ActorData.FindOrAdd(HUD);

    // 添加完整布局
    for (const FLyraHUDLayoutRequest& Entry : Layout)
    {
        if (TSubclassOf<UCommonActivatableWidget> ConcreteWidgetClass = Entry.LayoutClass.Get())
        {
            // 推送到CommonUI的Layer栈
            ActorData.LayoutsAdded.Add(
                UCommonUIExtensions::PushContentToLayer_ForPlayer(
                    LocalPlayer, 
                    Entry.LayerID, 
                    ConcreteWidgetClass
                )
            );
        }
    }

    // 注册HUD元素扩展
    UUIExtensionSubsystem* ExtensionSubsystem = HUD->GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
    for (const FLyraHUDElementEntry& Entry : Widgets)
    {
        ActorData.ExtensionHandles.Add(
            ExtensionSubsystem->RegisterExtensionAsWidgetForContext(
                Entry.SlotID,         // 扩展点Tag
                LocalPlayer,          // 上下文
                Entry.WidgetClass.Get(),  // Widget类
                -1                    // 优先级（-1表示默认）
            )
        );
    }
}

void UGameFeatureAction_AddWidgets::RemoveWidgets(AActor* Actor, FPerContextData& ActiveData)
{
    ALyraHUD* HUD = CastChecked<ALyraHUD>(Actor);
    FPerActorData* ActorData = ActiveData.ActorData.Find(HUD);

    if (ActorData)
    {
        // 移除布局
        for (TWeakObjectPtr<UCommonActivatableWidget>& AddedLayout : ActorData->LayoutsAdded)
        {
            if (AddedLayout.IsValid())
            {
                AddedLayout->DeactivateWidget();
            }
        }

        // 注销扩展
        for (FUIExtensionHandle& Handle : ActorData->ExtensionHandles)
        {
            Handle.Unregister();
        }
        
        ActiveData.ActorData.Remove(HUD);
    }
}
```

### 4.4 GameFeature配置示例

```json
// ShooterCore.uplugin 中的 GameFeatureData 配置
{
    "Actions": [
        {
            "_Type": "GameFeatureAction_AddWidgets",
            "Layout": [
                {
                    "LayoutClass": "/ShooterCore/UI/W_ShooterHUDLayout.W_ShooterHUDLayout_C",
                    "LayerID": "UI.Layer.Game"
                }
            ],
            "Widgets": [
                {
                    "WidgetClass": "/ShooterCore/UI/Weapons/W_WeaponReticle.W_WeaponReticle_C",
                    "SlotID": "UI.HUD.Reticle"
                },
                {
                    "WidgetClass": "/ShooterCore/UI/Weapons/W_AmmoCounter.W_AmmoCounter_C",
                    "SlotID": "UI.HUD.Weapon"
                },
                {
                    "WidgetClass": "/ShooterCore/UI/Abilities/W_AbilityBar.W_AbilityBar_C",
                    "SlotID": "UI.HUD.Abilities"
                }
            ]
        }
    ]
}
```

**优势总结：**
1. **完全模块化**：ShooterCore插件可以独立开发和测试
2. **自动管理**：插件激活/停用时自动添加/移除UI
3. **零侵入**：不需要修改主游戏代码
4. **支持热重载**：开发期间可以动态重载插件

---

## 5. 实战案例1：动态HUD元素系统

### 5.1 系统架构设计

我们将构建一个完整的动态HUD系统，包含以下元素：
- 玩家血条和护盾
- 技能冷却指示器
- Buff/Debuff图标列表
- 队友状态显示

**扩展点规划：**

```
UI.HUD (PartialMatch)
├── UI.HUD.Player
│   ├── UI.HUD.Player.Health     (ExactMatch)
│   ├── UI.HUD.Player.Shield     (ExactMatch)
│   └── UI.HUD.Player.Buffs      (ExactMatch)
├── UI.HUD.Abilities             (PartialMatch)
│   ├── UI.HUD.Abilities.Primary
│   ├── UI.HUD.Abilities.Secondary
│   └── UI.HUD.Abilities.Ultimate
└── UI.HUD.Team                  (PartialMatch)
    └── UI.HUD.Team.Member.{PlayerStateId}
```

### 5.2 主HUD布局Widget

```cpp
// LyraAdvancedHUDLayout.h
#pragma once

#include "LyraHUDLayout.h"
#include "GameplayTagContainer.h"
#include "LyraAdvancedHUDLayout.generated.h"

UCLASS(Abstract)
class LYRAGAME_API ULyraAdvancedHUDLayout : public ULyraHUDLayout
{
    GENERATED_BODY()

protected:
    virtual void NativeOnInitialized() override;
    virtual void NativeDestruct() override;

    // 扩展点容器引用（在蓝图中绑定）
    UPROPERTY(BlueprintReadWrite, meta=(BindWidget))
    TObjectPtr<UUIExtensionPointWidget> PlayerHealthSlot;

    UPROPERTY(BlueprintReadWrite, meta=(BindWidget))
    TObjectPtr<UUIExtensionPointWidget> PlayerShieldSlot;

    UPROPERTY(BlueprintReadWrite, meta=(BindWidget))
    TObjectPtr<UUIExtensionPointWidget> PlayerBuffsContainer;

    UPROPERTY(BlueprintReadWrite, meta=(BindWidget))
    TObjectPtr<UUIExtensionPointWidget> AbilitiesContainer;

    UPROPERTY(BlueprintReadWrite, meta=(BindWidget))
    TObjectPtr<UUIExtensionPointWidget> TeamMembersContainer;

private:
    void InitializeExtensionPoints();
    void RegisterPlayerHealthExtensions();
    void RegisterAbilityExtensions();
    
    UPROPERTY()
    TArray<FUIExtensionHandle> DynamicExtensionHandles;
};
```

```cpp
// LyraAdvancedHUDLayout.cpp
#include "LyraAdvancedHUDLayout.h"
#include "UIExtensionSystem.h"
#include "Components/UIExtensionPointWidget.h"

void ULyraAdvancedHUDLayout::NativeOnInitialized()
{
    Super::NativeOnInitialized();
    
    InitializeExtensionPoints();
    RegisterPlayerHealthExtensions();
    RegisterAbilityExtensions();
}

void ULyraAdvancedHUDLayout::InitializeExtensionPoints()
{
    // 配置扩展点容器（也可在蓝图中设置）
    if (PlayerHealthSlot)
    {
        PlayerHealthSlot->ExtensionPointTag = 
            FGameplayTag::RequestGameplayTag("UI.HUD.Player.Health");
        PlayerHealthSlot->ExtensionPointTagMatch = EUIExtensionPointMatch::ExactMatch;
    }

    if (PlayerBuffsContainer)
    {
        PlayerBuffsContainer->ExtensionPointTag = 
            FGameplayTag::RequestGameplayTag("UI.HUD.Player.Buffs");
        PlayerBuffsContainer->ExtensionPointTagMatch = EUIExtensionPointMatch::PartialMatch;
        
        // 支持自定义数据对象
        PlayerBuffsContainer->DataClasses.Add(UBuffDisplayData::StaticClass());
    }

    if (AbilitiesContainer)
    {
        AbilitiesContainer->ExtensionPointTag = 
            FGameplayTag::RequestGameplayTag("UI.HUD.Abilities");
        AbilitiesContainer->ExtensionPointTagMatch = EUIExtensionPointMatch::PartialMatch;
    }
}

void ULyraAdvancedHUDLayout::RegisterPlayerHealthExtensions()
{
    // 示例：条件性注册护盾UI（只有当玩家拥有护盾能力时）
    ALyraPlayerController* PC = GetOwningPlayer<ALyraPlayerController>();
    if (PC && PC->HasShieldAbility())
    {
        UUIExtensionSubsystem* ExtensionSubsystem = 
            GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
            
        FUIExtensionHandle ShieldHandle = ExtensionSubsystem->RegisterExtensionAsWidgetForContext(
            FGameplayTag::RequestGameplayTag("UI.HUD.Player.Shield"),
            GetOwningLocalPlayer(),
            ShieldBarWidgetClass,
            10  // 高优先级
        );
        
        DynamicExtensionHandles.Add(ShieldHandle);
    }
}

void ULyraAdvancedHUDLayout::NativeDestruct()
{
    // 清理动态注册的扩展
    for (FUIExtensionHandle& Handle : DynamicExtensionHandles)
    {
        Handle.Unregister();
    }
    DynamicExtensionHandles.Empty();
    
    Super::NativeDestruct();
}
```

### 5.3 血条Widget实现

```cpp
// LyraHealthBarWidget.h
#pragma once

#include "CommonUserWidget.h"
#include "GameplayEffectTypes.h"
#include "LyraHealthBarWidget.generated.h"

UCLASS(Abstract)
class LYRAGAME_API ULyraHealthBarWidget : public UCommonUserWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override;
    virtual void NativeDestruct() override;

protected:
    // 蓝图可实现的更新函数
    UFUNCTION(BlueprintImplementableEvent, Category="Health")
    void OnHealthChanged(float CurrentHealth, float MaxHealth, float Percentage);

    UFUNCTION(BlueprintImplementableEvent, Category="Health")
    void OnMaxHealthChanged(float NewMaxHealth);

private:
    void InitializeHealthComponent();
    void OnHealthAttributeChanged(const FOnAttributeChangeData& Data);
    
    UPROPERTY()
    TObjectPtr<ULyraHealthComponent> BoundHealthComponent;
    
    FDelegateHandle HealthChangedHandle;
    FDelegateHandle MaxHealthChangedHandle;
};
```

```cpp
// LyraHealthBarWidget.cpp
#include "LyraHealthBarWidget.h"
#include "Character/LyraHealthComponent.h"
#include "Player/LyraPlayerState.h"

void ULyraHealthBarWidget::NativeConstruct()
{
    Super::NativeConstruct();
    InitializeHealthComponent();
}

void ULyraHealthBarWidget::InitializeHealthComponent()
{
    // 获取本地玩家的HealthComponent
    if (APlayerController* PC = GetOwningPlayer())
    {
        if (APawn* Pawn = PC->GetPawn())
        {
            BoundHealthComponent = Pawn->FindComponentByClass<ULyraHealthComponent>();
            
            if (BoundHealthComponent)
            {
                // 绑定属性变化回调
                HealthChangedHandle = BoundHealthComponent->GetHealthAttributeChangeDelegate()
                    .AddUObject(this, &ThisClass::OnHealthAttributeChanged);
                
                MaxHealthChangedHandle = BoundHealthComponent->GetMaxHealthAttributeChangeDelegate()
                    .AddUObject(this, &ThisClass::OnHealthAttributeChanged);
                
                // 初始化显示
                float CurrentHealth = BoundHealthComponent->GetHealth();
                float MaxHealth = BoundHealthComponent->GetMaxHealth();
                OnHealthChanged(CurrentHealth, MaxHealth, CurrentHealth / MaxHealth);
            }
        }
    }
}

void ULyraHealthBarWidget::OnHealthAttributeChanged(const FOnAttributeChangeData& Data)
{
    if (BoundHealthComponent)
    {
        float CurrentHealth = BoundHealthComponent->GetHealth();
        float MaxHealth = BoundHealthComponent->GetMaxHealth();
        float Percentage = FMath::Clamp(CurrentHealth / MaxHealth, 0.0f, 1.0f);
        
        OnHealthChanged(CurrentHealth, MaxHealth, Percentage);
    }
}

void ULyraHealthBarWidget::NativeDestruct()
{
    if (BoundHealthComponent)
    {
        BoundHealthComponent->GetHealthAttributeChangeDelegate().Remove(HealthChangedHandle);
        BoundHealthComponent->GetMaxHealthAttributeChangeDelegate().Remove(MaxHealthChangedHandle);
    }
    
    Super::NativeDestruct();
}
```

### 5.4 Buff系统集成

**Buff数据对象：**

```cpp
// BuffDisplayData.h
#pragma once

#include "GameplayTagContainer.h"
#include "BuffDisplayData.generated.h"

UENUM(BlueprintType)
enum class EBuffType : uint8
{
    Positive,    // 增益
    Negative,    // 减益
    Neutral      // 中性
};

UCLASS(BlueprintType)
class LYRAGAME_API UBuffDisplayData : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintReadOnly, Category="Buff")
    FGameplayTag BuffTag;

    UPROPERTY(BlueprintReadOnly, Category="Buff")
    EBuffType BuffType;

    UPROPERTY(BlueprintReadOnly, Category="Buff")
    TObjectPtr<UTexture2D> Icon;

    UPROPERTY(BlueprintReadOnly, Category="Buff")
    FText DisplayName;

    UPROPERTY(BlueprintReadOnly, Category="Buff")
    float Duration;

    UPROPERTY(BlueprintReadOnly, Category="Buff")
    float RemainingTime;

    UPROPERTY(BlueprintReadOnly, Category="Buff")
    int32 StackCount;
};
```

**Buff管理器组件：**

```cpp
// LyraBuffManagerComponent.h
#pragma once

#include "Components/ActorComponent.h"
#include "GameplayEffectTypes.h"
#include "UIExtensionSystem.h"
#include "LyraBuffManagerComponent.generated.h"

UCLASS(BlueprintType, meta=(BlueprintSpawnableComponent))
class LYRAGAME_API ULyraBuffManagerComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    ULyraBuffManagerComponent();

    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

protected:
    UFUNCTION()
    void OnGameplayEffectApplied(UAbilitySystemComponent* ASC, 
        const FGameplayEffectSpec& Spec, 
        FActiveGameplayEffectHandle Handle);

    UFUNCTION()
    void OnGameplayEffectRemoved(const FActiveGameplayEffect& Effect);

private:
    void RegisterBuffUI(const FActiveGameplayEffectHandle& Handle);
    void UnregisterBuffUI(const FActiveGameplayEffectHandle& Handle);
    
    UBuffDisplayData* CreateBuffDisplayData(const FActiveGameplayEffect& Effect);
    
    UPROPERTY()
    TMap<FActiveGameplayEffectHandle, FUIExtensionHandle> BuffExtensions;
    
    UPROPERTY()
    TMap<FActiveGameplayEffectHandle, TObjectPtr<UBuffDisplayData>> BuffDataObjects;
};
```

```cpp
// LyraBuffManagerComponent.cpp
#include "LyraBuffManagerComponent.h"
#include "AbilitySystemComponent.h"
#include "UIExtensionSystem.h"

ULyraBuffManagerComponent::ULyraBuffManagerComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

void ULyraBuffManagerComponent::BeginPlay()
{
    Super::BeginPlay();
    
    // 绑定到ASC的事件
    if (AActor* Owner = GetOwner())
    {
        if (UAbilitySystemComponent* ASC = Owner->FindComponentByClass<UAbilitySystemComponent>())
        {
            ASC->OnActiveGameplayEffectAddedDelegateToSelf.AddUObject(
                this, &ThisClass::OnGameplayEffectApplied);
                
            ASC->OnAnyGameplayEffectRemovedDelegate().AddUObject(
                this, &ThisClass::OnGameplayEffectRemoved);
        }
    }
}

void ULyraBuffManagerComponent::OnGameplayEffectApplied(
    UAbilitySystemComponent* ASC, 
    const FGameplayEffectSpec& Spec, 
    FActiveGameplayEffectHandle Handle)
{
    // 检查是否应该显示为Buff
    const FGameplayTagContainer& EffectTags = Spec.Def->InheritableGameplayEffectTags.CombinedTags;
    if (EffectTags.HasTag(FGameplayTag::RequestGameplayTag("Effect.ShowAsBuff")))
    {
        RegisterBuffUI(Handle);
    }
}

void ULyraBuffManagerComponent::RegisterBuffUI(const FActiveGameplayEffectHandle& Handle)
{
    UAbilitySystemComponent* ASC = GetOwner()->FindComponentByClass<UAbilitySystemComponent>();
    const FActiveGameplayEffect* ActiveEffect = ASC->GetActiveGameplayEffect(Handle);
    
    if (!ActiveEffect) return;

    // 创建数据对象
    UBuffDisplayData* BuffData = CreateBuffDisplayData(*ActiveEffect);
    BuffDataObjects.Add(Handle, BuffData);

    // 注册UI扩展
    UUIExtensionSubsystem* ExtensionSubsystem = GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
    
    // 构建动态Tag（支持多个Buff同时显示）
    FString BuffTagString = FString::Printf(TEXT("UI.HUD.Player.Buffs.%s"), 
        *Handle.ToString());
    FGameplayTag BuffSlotTag = FGameplayTag::RequestGameplayTag(FName(*BuffTagString));
    
    FUIExtensionHandle ExtensionHandle = ExtensionSubsystem->RegisterExtensionAsDataForContext(
        BuffSlotTag,
        GetOwningLocalPlayer(),
        BuffData,
        0  // 优先级
    );
    
    BuffExtensions.Add(Handle, ExtensionHandle);
}

void ULyraBuffManagerComponent::OnGameplayEffectRemoved(const FActiveGameplayEffect& Effect)
{
    FActiveGameplayEffectHandle Handle = Effect.Handle;
    
    if (FUIExtensionHandle* ExtensionHandle = BuffExtensions.Find(Handle))
    {
        ExtensionHandle->Unregister();
        BuffExtensions.Remove(Handle);
    }
    
    BuffDataObjects.Remove(Handle);
}

UBuffDisplayData* ULyraBuffManagerComponent::CreateBuffDisplayData(
    const FActiveGameplayEffect& Effect)
{
    UBuffDisplayData* BuffData = NewObject<UBuffDisplayData>(this);
    
    const UGameplayEffect* GEDef = Effect.Spec.Def;
    
    // 从GameplayEffect的AssetTags提取信息
    BuffData->BuffTag = /* 提取主Tag */;
    BuffData->Duration = Effect.GetDuration();
    BuffData->RemainingTime = Effect.GetTimeRemaining(GetWorld()->GetTimeSeconds());
    BuffData->StackCount = Effect.Spec.StackCount;
    
    // 从DataAsset或Table查询显示信息
    if (UBuffDataAsset* DataAsset = /* 查找逻辑 */)
    {
        BuffData->Icon = DataAsset->Icon;
        BuffData->DisplayName = DataAsset->DisplayName;
        BuffData->BuffType = DataAsset->BuffType;
    }
    
    return BuffData;
}

void ULyraBuffManagerComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    // 清理所有UI扩展
    for (TPair<FActiveGameplayEffectHandle, FUIExtensionHandle>& Pair : BuffExtensions)
    {
        Pair.Value.Unregister();
    }
    BuffExtensions.Empty();
    BuffDataObjects.Empty();
    
    Super::EndPlay(EndPlayReason);
}
```

### 5.5 技能栏动态注册

```cpp
// LyraAbilityBarExtensionManager.h
#pragma once

#include "Components/ActorComponent.h"
#include "GameplayAbilitySpec.h"
#include "LyraAbilityBarExtensionManager.generated.h"

UCLASS()
class LYRAGAME_API ULyraAbilityBarExtensionManager : public UActorComponent
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

protected:
    void OnAbilityGiven(const FGameplayAbilitySpec& Spec);
    void OnAbilityRemoved(const FGameplayAbilitySpec& Spec);
    
    void RegisterAbilityUI(const FGameplayAbilitySpecHandle& Handle);
    void UnregisterAbilityUI(const FGameplayAbilitySpecHandle& Handle);
    
private:
    UPROPERTY()
    TMap<FGameplayAbilitySpecHandle, FUIExtensionHandle> AbilityExtensions;
    
    FDelegateHandle AbilityGivenHandle;
    FDelegateHandle AbilityRemovedHandle;
};
```

```cpp
// 实现（关键部分）
void ULyraAbilityBarExtensionManager::RegisterAbilityUI(
    const FGameplayAbilitySpecHandle& Handle)
{
    UAbilitySystemComponent* ASC = GetOwner()->FindComponentByClass<UAbilitySystemComponent>();
    FGameplayAbilitySpec* Spec = ASC->FindAbilitySpecFromHandle(Handle);
    
    if (!Spec || !Spec->Ability) return;

    // 从Ability获取UI配置
    ULyraGameplayAbility* LyraAbility = Cast<ULyraGameplayAbility>(Spec->Ability);
    if (!LyraAbility || !LyraAbility->AbilityUIWidgetClass) return;

    // 确定插槽位置
    FGameplayTag SlotTag;
    switch (LyraAbility->AbilitySlot)
    {
        case EAbilitySlot::Primary:
            SlotTag = FGameplayTag::RequestGameplayTag("UI.HUD.Abilities.Primary");
            break;
        case EAbilitySlot::Secondary:
            SlotTag = FGameplayTag::RequestGameplayTag("UI.HUD.Abilities.Secondary");
            break;
        case EAbilitySlot::Ultimate:
            SlotTag = FGameplayTag::RequestGameplayTag("UI.HUD.Abilities.Ultimate");
            break;
        default:
            return;
    }

    // 注册扩展
    UUIExtensionSubsystem* ExtensionSubsystem = GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
    FUIExtensionHandle ExtensionHandle = ExtensionSubsystem->RegisterExtensionAsWidgetForContext(
        SlotTag,
        GetOwningLocalPlayer(),
        LyraAbility->AbilityUIWidgetClass,
        LyraAbility->AbilityPriority
    );
    
    AbilityExtensions.Add(Handle, ExtensionHandle);
}
```

### 5.6 GameFeature配置

```json
// CombatSystem.uplugin 的 GameFeatureData
{
    "Actions": [
        {
            "_Type": "GameFeatureAction_AddWidgets",
            "Widgets": [
                {
                    "WidgetClass": "/CombatSystem/UI/W_HealthBar.W_HealthBar_C",
                    "SlotID": "UI.HUD.Player.Health"
                },
                {
                    "WidgetClass": "/CombatSystem/UI/W_ShieldBar.W_ShieldBar_C",
                    "SlotID": "UI.HUD.Player.Shield"
                }
            ]
        },
        {
            "_Type": "GameFeatureAction_AddComponents",
            "ComponentList": [
                {
                    "ActorClass": "/Script/LyraGame.LyraCharacter",
                    "ComponentClass": "/Script/LyraGame.LyraBuffManagerComponent"
                },
                {
                    "ActorClass": "/Script/LyraGame.LyraPlayerController",
                    "ComponentClass": "/Script/LyraGame.LyraAbilityBarExtensionManager"
                }
            ]
        }
    ]
}
```

---

## 6. 实战案例2：可插拔商店UI系统

### 6.1 商店架构设计

**设计目标：**
- 不同GameFeature可以注册商品类别
- 商品显示使用数据驱动
- 支持动态价格和库存
- 网络同步商店状态

**扩展点层级：**

```
UI.Shop (PartialMatch)
├── UI.Shop.Categories            (PartialMatch)
│   ├── UI.Shop.Categories.Weapons
│   ├── UI.Shop.Categories.Armor
│   ├── UI.Shop.Categories.Consumables
│   └── UI.Shop.Categories.Special
└── UI.Shop.Items.{CategoryId}    (PartialMatch)
    └── UI.Shop.Items.Weapons.{ItemId}
```

### 6.2 商店主界面

```cpp
// LyraShopWidget.h
#pragma once

#include "CommonActivatableWidget.h"
#include "UIExtensionSystem.h"
#include "LyraShopWidget.generated.h"

UCLASS(Abstract)
class LYRAGAME_API ULyraShopWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()

public:
    virtual void NativeOnActivated() override;
    virtual void NativeOnDeactivated() override;

protected:
    // 扩展点：商品类别标签页
    UPROPERTY(BlueprintReadWrite, meta=(BindWidget))
    TObjectPtr<UUIExtensionPointWidget> CategoryTabContainer;

    // 扩展点：当前类别的商品列表
    UPROPERTY(BlueprintReadWrite, meta=(BindWidget))
    TObjectPtr<UUIExtensionPointWidget> ItemListContainer;

    // 扩展点：购买确认面板
    UPROPERTY(BlueprintReadWrite, meta=(BindWidget))
    TObjectPtr<UUIExtensionPointWidget> PurchaseConfirmationSlot;

private:
    void InitializeShopExtensions();
    void RegisterShopRPC();
    
    UFUNCTION()
    void OnCategorySelected(FGameplayTag CategoryTag);
    
    UPROPERTY()
    FGameplayTag CurrentCategory;
    
    FUIExtensionPointHandle CategoryPointHandle;
    FUIExtensionPointHandle ItemPointHandle;
};
```

```cpp
// LyraShopWidget.cpp
#include "LyraShopWidget.h"
#include "UIExtensionSystem.h"
#include "Widgets/UIExtensionPointWidget.h"

void ULyraShopWidget::NativeOnActivated()
{
    Super::NativeOnActivated();
    
    InitializeShopExtensions();
    RegisterShopRPC();
}

void ULyraShopWidget::InitializeShopExtensions()
{
    // 配置类别容器
    if (CategoryTabContainer)
    {
        CategoryTabContainer->ExtensionPointTag = 
            FGameplayTag::RequestGameplayTag("UI.Shop.Categories");
        CategoryTabContainer->ExtensionPointTagMatch = EUIExtensionPointMatch::PartialMatch;
        
        // 绑定委托以实现数据驱动的Tab创建
        CategoryTabContainer->GetWidgetClassForData.BindDynamic(
            this, &ThisClass::GetTabWidgetForCategory);
        CategoryTabContainer->ConfigureWidgetForData.BindDynamic(
            this, &ThisClass::ConfigureCategoryTab);
    }

    // 配置商品列表容器
    if (ItemListContainer)
    {
        ItemListContainer->ExtensionPointTag = 
            FGameplayTag::RequestGameplayTag("UI.Shop.Items");
        ItemListContainer->ExtensionPointTagMatch = EUIExtensionPointMatch::PartialMatch;
        
        ItemListContainer->DataClasses.Add(UShopItemData::StaticClass());
        ItemListContainer->GetWidgetClassForData.BindDynamic(
            this, &ThisClass::GetWidgetForItemData);
        ItemListContainer->ConfigureWidgetForData.BindDynamic(
            this, &ThisClass::ConfigureItemWidget);
    }
}

UFUNCTION()
TSubclassOf<UUserWidget> ULyraShopWidget::GetTabWidgetForCategory(UObject* DataItem)
{
    // 所有类别使用统一的Tab Widget
    return CategoryTabWidgetClass;
}

UFUNCTION()
void ULyraShopWidget::ConfigureCategoryTab(UUserWidget* Widget, UObject* DataItem)
{
    UShopCategoryTabWidget* TabWidget = Cast<UShopCategoryTabWidget>(Widget);
    UShopCategoryData* CategoryData = Cast<UShopCategoryData>(DataItem);
    
    if (TabWidget && CategoryData)
    {
        TabWidget->SetCategoryIcon(CategoryData->CategoryIcon);
        TabWidget->SetCategoryName(CategoryData->CategoryName);
        TabWidget->SetCategoryTag(CategoryData->CategoryTag);
        
        // 绑定选择事件
        TabWidget->OnCategoryClicked.AddDynamic(this, &ThisClass::OnCategorySelected);
    }
}

UFUNCTION()
TSubclassOf<UUserWidget> ULyraShopWidget::GetWidgetForItemData(UObject* DataItem)
{
    UShopItemData* ItemData = Cast<UShopItemData>(DataItem);
    if (!ItemData) return nullptr;

    // 根据物品类型返回不同的Widget类
    switch (ItemData->ItemType)
    {
        case EShopItemType::Weapon:
            return WeaponItemWidgetClass;
        case EShopItemType::Armor:
            return ArmorItemWidgetClass;
        case EShopItemType::Consumable:
            return ConsumableItemWidgetClass;
        default:
            return DefaultItemWidgetClass;
    }
}

void ULyraShopWidget::OnCategorySelected(FGameplayTag CategoryTag)
{
    CurrentCategory = CategoryTag;
    
    // 更新商品列表扩展点的过滤
    if (ItemListContainer)
    {
        // 清空当前列表
        ItemListContainer->Reset();
        
        // 重新注册扩展点以触发刷新
        FGameplayTag ItemTag = FGameplayTag::RequestGameplayTag(
            FString::Printf(TEXT("UI.Shop.Items.%s"), *CategoryTag.GetTagName().ToString())
        );
        ItemListContainer->ExtensionPointTag = ItemTag;
    }
}

void ULyraShopWidget::NativeOnDeactivated()
{
    CategoryPointHandle.Unregister();
    ItemPointHandle.Unregister();
    
    Super::NativeOnDeactivated();
}
```

### 6.3 商品数据定义

```cpp
// ShopItemData.h
#pragma once

#include "GameplayTagContainer.h"
#include "ShopItemData.generated.h"

UENUM(BlueprintType)
enum class EShopItemType : uint8
{
    Weapon,
    Armor,
    Consumable,
    Special
};

USTRUCT(BlueprintType)
struct FShopItemPrice
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite, Category="Shop")
    FGameplayTag CurrencyTag;  // 如 "Currency.Gold" 或 "Currency.Premium"

    UPROPERTY(BlueprintReadWrite, Category="Shop")
    int32 Amount;
};

UCLASS(BlueprintType)
class LYRAGAME_API UShopItemData : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintReadOnly, Category="Shop")
    FGameplayTag ItemTag;

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    FGameplayTag CategoryTag;

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    EShopItemType ItemType;

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    FText ItemName;

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    FText Description;

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    TObjectPtr<UTexture2D> Icon;

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    TArray<FShopItemPrice> Prices;

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    int32 StockLimit;  // -1 表示无限

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    int32 CurrentStock;

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    bool bRequiresUnlock;

    UPROPERTY(BlueprintReadOnly, Category="Shop")
    FGameplayTagContainer RequiredTags;  // 购买条件
};
```

### 6.4 商店管理器组件

```cpp
// LyraShopManagerComponent.h
#pragma once

#include "Components/GameStateComponent.h"
#include "UIExtensionSystem.h"
#include "LyraShopManagerComponent.generated.h"

UCLASS()
class LYRAGAME_API ULyraShopManagerComponent : public UGameStateComponent
{
    GENERATED_BODY()

public:
    ULyraShopManagerComponent();

    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    // 供外部注册商品（通常由GameFeature调用）
    UFUNCTION(BlueprintCallable, Category="Shop")
    void RegisterShopCategory(UShopCategoryData* CategoryData);

    UFUNCTION(BlueprintCallable, Category="Shop")
    void RegisterShopItem(UShopItemData* ItemData);

    UFUNCTION(BlueprintCallable, Category="Shop")
    void UnregisterShopCategory(FGameplayTag CategoryTag);

    UFUNCTION(BlueprintCallable, Category="Shop")
    void UnregisterShopItem(FGameplayTag ItemTag);

    // 网络RPC
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerPurchaseItem(APlayerController* Buyer, FGameplayTag ItemTag);

protected:
    void BroadcastShopUpdate();

private:
    UPROPERTY(Replicated)
    TArray<TObjectPtr<UShopItemData>> AvailableItems;

    UPROPERTY()
    TMap<FGameplayTag, FUIExtensionHandle> CategoryExtensions;

    UPROPERTY()
    TMap<FGameplayTag, FUIExtensionHandle> ItemExtensions;
};
```

```cpp
// LyraShopManagerComponent.cpp
#include "LyraShopManagerComponent.h"
#include "UIExtensionSystem.h"
#include "Net/UnrealNetwork.h"

ULyraShopManagerComponent::ULyraShopManagerComponent()
{
    SetIsReplicatedByDefault(true);
}

void ULyraShopManagerComponent::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    
    DOREPLIFETIME(ULyraShopManagerComponent, AvailableItems);
}

void ULyraShopManagerComponent::BeginPlay()
{
    Super::BeginPlay();
    
    // 只在服务器上初始化（客户端通过复制接收）
    if (GetOwner()->HasAuthority())
    {
        // 加载默认商店配置
        LoadDefaultShopConfiguration();
    }
}

void ULyraShopManagerComponent::RegisterShopCategory(UShopCategoryData* CategoryData)
{
    if (!CategoryData) return;

    UUIExtensionSubsystem* ExtensionSubsystem = GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
    
    // 为所有本地玩家注册类别扩展
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        APlayerController* PC = It->Get();
        if (PC && PC->IsLocalController())
        {
            FGameplayTag CategoryTag = FGameplayTag::RequestGameplayTag(
                FString::Printf(TEXT("UI.Shop.Categories.%s"), *CategoryData->CategoryTag.GetTagName().ToString())
            );
            
            FUIExtensionHandle Handle = ExtensionSubsystem->RegisterExtensionAsDataForContext(
                CategoryTag,
                PC->GetLocalPlayer(),
                CategoryData,
                CategoryData->DisplayPriority
            );
            
            CategoryExtensions.Add(CategoryData->CategoryTag, Handle);
        }
    }
}

void ULyraShopManagerComponent::RegisterShopItem(UShopItemData* ItemData)
{
    if (!ItemData) return;

    // 添加到可用列表（会触发复制）
    AvailableItems.AddUnique(ItemData);

    // 仅在拥有权威的情况下注册UI扩展（避免重复）
    if (GetOwner()->HasAuthority())
    {
        UUIExtensionSubsystem* ExtensionSubsystem = GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
        
        for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
        {
            APlayerController* PC = It->Get();
            if (PC && PC->IsLocalController())
            {
                // 检查玩家是否满足解锁条件
                if (ItemData->bRequiresUnlock && !PlayerMeetsRequirements(PC, ItemData))
                {
                    continue;
                }

                FGameplayTag ItemSlotTag = FGameplayTag::RequestGameplayTag(
                    FString::Printf(TEXT("UI.Shop.Items.%s.%s"), 
                        *ItemData->CategoryTag.GetTagName().ToString(),
                        *ItemData->ItemTag.GetTagName().ToString())
                );
                
                FUIExtensionHandle Handle = ExtensionSubsystem->RegisterExtensionAsDataForContext(
                    ItemSlotTag,
                    PC->GetLocalPlayer(),
                    ItemData,
                    0  // 商品按字母或价格排序，优先级通常相同
                );
                
                ItemExtensions.Add(ItemData->ItemTag, Handle);
            }
        }
    }
}

bool ULyraShopManagerComponent::ServerPurchaseItem_Validate(
    APlayerController* Buyer, 
    FGameplayTag ItemTag)
{
    return Buyer != nullptr && ItemTag.IsValid();
}

void ULyraShopManagerComponent::ServerPurchaseItem_Implementation(
    APlayerController* Buyer, 
    FGameplayTag ItemTag)
{
    // 查找商品
    UShopItemData* ItemData = nullptr;
    for (UShopItemData* Item : AvailableItems)
    {
        if (Item && Item->ItemTag == ItemTag)
        {
            ItemData = Item;
            break;
        }
    }

    if (!ItemData)
    {
        // 商品不存在
        ClientPurchaseResult(Buyer, false, FText::FromString("Item not found"));
        return;
    }

    // 检查库存
    if (ItemData->StockLimit >= 0 && ItemData->CurrentStock <= 0)
    {
        ClientPurchaseResult(Buyer, false, FText::FromString("Out of stock"));
        return;
    }

    // 检查货币
    ULyraCurrencyComponent* CurrencyComp = Buyer->FindComponentByClass<ULyraCurrencyComponent>();
    if (!CurrencyComp)
    {
        ClientPurchaseResult(Buyer, false, FText::FromString("Invalid player state"));
        return;
    }

    // 扣除货币
    bool bCanAfford = true;
    for (const FShopItemPrice& Price : ItemData->Prices)
    {
        if (!CurrencyComp->HasCurrency(Price.CurrencyTag, Price.Amount))
        {
            bCanAfford = false;
            break;
        }
    }

    if (!bCanAfford)
    {
        ClientPurchaseResult(Buyer, false, FText::FromString("Insufficient funds"));
        return;
    }

    // 执行购买
    for (const FShopItemPrice& Price : ItemData->Prices)
    {
        CurrencyComp->DeductCurrency(Price.CurrencyTag, Price.Amount);
    }

    // 更新库存
    if (ItemData->StockLimit >= 0)
    {
        ItemData->CurrentStock--;
        
        // 如果售罄，移除UI扩展
        if (ItemData->CurrentStock == 0)
        {
            UnregisterShopItem(ItemTag);
        }
    }

    // 发放物品
    GrantItemToPlayer(Buyer, ItemData);

    ClientPurchaseResult(Buyer, true, FText::FromString("Purchase successful"));
}

void ULyraShopManagerComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    // 清理所有UI扩展
    for (TPair<FGameplayTag, FUIExtensionHandle>& Pair : CategoryExtensions)
    {
        Pair.Value.Unregister();
    }
    CategoryExtensions.Empty();

    for (TPair<FGameplayTag, FUIExtensionHandle>& Pair : ItemExtensions)
    {
        Pair.Value.Unregister();
    }
    ItemExtensions.Empty();

    Super::EndPlay(EndPlayReason);
}
```

### 6.5 GameFeature插件注册商品

```cpp
// WeaponShopFeature.h（位于某个GameFeature插件中）
#pragma once

#include "GameFeatureAction.h"
#include "WeaponShopFeature.generated.h"

USTRUCT()
struct FWeaponShopEntry
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category="Shop")
    TSoftObjectPtr<UShopItemData> ItemData;
};

UCLASS()
class WEAPONSHOP_API UWeaponShopFeature : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override;
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;

private:
    UPROPERTY(EditAnywhere, Category="Shop")
    TArray<FWeaponShopEntry> WeaponItems;

    TArray<FGameplayTag> RegisteredItemTags;
};
```

```cpp
// WeaponShopFeature.cpp
#include "WeaponShopFeature.h"
#include "Shop/LyraShopManagerComponent.h"

void UWeaponShopFeature::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
    Super::OnGameFeatureActivating(Context);
    
    // 查找ShopManager并注册商品
    for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
    {
        if (UWorld* World = WorldContext.World())
        {
            if (AGameStateBase* GameState = World->GetGameState())
            {
                if (ULyraShopManagerComponent* ShopManager = 
                    GameState->FindComponentByClass<ULyraShopManagerComponent>())
                {
                    for (const FWeaponShopEntry& Entry : WeaponItems)
                    {
                        if (UShopItemData* ItemData = Entry.ItemData.LoadSynchronous())
                        {
                            ShopManager->RegisterShopItem(ItemData);
                            RegisteredItemTags.Add(ItemData->ItemTag);
                        }
                    }
                }
            }
        }
    }
}

void UWeaponShopFeature::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    // 卸载插件时移除商品
    for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
    {
        if (UWorld* World = WorldContext.World())
        {
            if (AGameStateBase* GameState = World->GetGameState())
            {
                if (ULyraShopManagerComponent* ShopManager = 
                    GameState->FindComponentByClass<ULyraShopManagerComponent>())
                {
                    for (const FGameplayTag& ItemTag : RegisteredItemTags)
                    {
                        ShopManager->UnregisterShopItem(ItemTag);
                    }
                }
            }
        }
    }
    
    RegisteredItemTags.Empty();
    
    Super::OnGameFeatureDeactivating(Context);
}
```

---

## 7. 网络同步与多人模式支持

### 7.1 上下文对象的网络策略

UIExtension系统通过 `ContextObject` 支持多人游戏，关键在于选择正确的上下文：

```cpp
// 策略1：LocalPlayer上下文（客户端本地UI）
ExtensionSubsystem->RegisterExtensionAsWidgetForContext(
    Tag,
    GetOwningLocalPlayer(),  // 仅本地玩家看到
    WidgetClass,
    Priority
);

// 策略2：PlayerState上下文（网络复制的玩家状态）
ExtensionSubsystem->RegisterExtensionAsWidgetForContext(
    Tag,
    PlayerState,  // 根据PlayerState匹配
    WidgetClass,
    Priority
);

// 策略3：全局上下文（所有玩家共享）
ExtensionSubsystem->RegisterExtensionAsWidget(
    Tag,
    WidgetClass,
    Priority
);
```

**多人模式示例：队友血条**

```cpp
// LyraTeamHealthBarManager.h
UCLASS()
class LYRAGAME_API ULyraTeamHealthBarManager : public UActorComponent
{
    GENERATED_BODY()

public:
    void RegisterTeammate(APlayerState* TeammateState);
    void UnregisterTeammate(APlayerState* TeammateState);

private:
    void CreateHealthBarForPlayer(APlayerState* PlayerState);
    
    UPROPERTY()
    TMap<TObjectPtr<APlayerState>, FUIExtensionHandle> TeammateHealthBars;
};

void ULyraTeamHealthBarManager::RegisterTeammate(APlayerState* TeammateState)
{
    if (!TeammateState || TeammateHealthBars.Contains(TeammateState))
        return;

    UUIExtensionSubsystem* ExtensionSubsystem = GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
    
    // 关键：使用TeammateState作为上下文
    // 每个玩家的扩展点会根据PlayerState匹配
    FGameplayTag TeamMemberTag = FGameplayTag::RequestGameplayTag(
        FString::Printf(TEXT("UI.HUD.Team.Member.%d"), TeammateState->GetPlayerId())
    );
    
    FUIExtensionHandle Handle = ExtensionSubsystem->RegisterExtensionAsWidgetForContext(
        TeamMemberTag,
        TeammateState,  // 使用PlayerState作为上下文！
        TeamHealthBarWidgetClass,
        0
    );
    
    TeammateHealthBars.Add(TeammateState, Handle);
}
```

### 7.2 数据复制策略

对于需要在服务器和客户端同步的UI数据：

```cpp
// 复制的商品数据
UCLASS(Replicated)
class UShopItemData : public UObject
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        
        DOREPLIFETIME(UShopItemData, CurrentStock);
        DOREPLIFETIME_CONDITION(UShopItemData, bRequiresUnlock, COND_OwnerOnly);
    }

    UPROPERTY(Replicated)
    int32 CurrentStock;

    UPROPERTY(Replicated)
    bool bRequiresUnlock;

    // 客户端接收到复制后刷新UI
    virtual void OnRep_CurrentStock()
    {
        OnStockChanged.Broadcast(CurrentStock);
    }

    DECLARE_MULTICAST_DELEGATE_OneParam(FOnStockChanged, int32);
    FOnStockChanged OnStockChanged;
};
```

### 7.3 权威服务器验证

```cpp
// 服务器端商品注册（防作弊）
void ULyraShopManagerComponent::ServerRegisterItem_Implementation(UShopItemData* ItemData)
{
    // 验证物品合法性
    if (!IsValidItemData(ItemData))
    {
        UE_LOG(LogLyra, Warning, TEXT("Invalid shop item rejected: %s"), *GetNameSafe(ItemData));
        return;
    }

    // 仅在服务器上注册UI扩展
    if (GetOwner()->HasAuthority())
    {
        RegisterShopItem(ItemData);
    }
}

// 客户端通过复制接收商品列表
void ULyraShopManagerComponent::OnRep_AvailableItems()
{
    // 重建UI扩展
    RefreshShopUI();
}
```

---

## 8. 性能优化与最佳实践

### 8.1 优化技巧

#### 8.1.1 延迟Widget创建

```cpp
// 不好的做法：立即创建所有Widget
void UBadShopWidget::NativeConstruct()
{
    for (int32 i = 0; i < 1000; ++i)
    {
        CreateItemWidget(Items[i]);  // 一次性创建1000个Widget！
    }
}

// 好的做法：使用虚拟化列表
void UGoodShopWidget::NativeConstruct()
{
    // 使用 UListView 或 UTileView（支持虚拟化）
    ItemListView->SetListItems(Items);  // 只创建可见的Widget
}
```

#### 8.1.2 扩展优先级排序

```cpp
// UIExtensionSubsystem内部优化
void UUIExtensionSubsystem::NotifyExtensionPointOfExtensions(
    TSharedPtr<FUIExtensionPoint>& ExtensionPoint)
{
    // ... 收集所有匹配的扩展 ...

    // 按优先级排序（只排序一次）
    ExtensionArray.Sort([](const TSharedPtr<FUIExtension>& A, const TSharedPtr<FUIExtension>& B)
    {
        return A->Priority > B->Priority;
    });

    // 按顺序通知
    for (const TSharedPtr<FUIExtension>& Extension : ExtensionArray)
    {
        ExtensionPoint->Callback.ExecuteIfBound(EUIExtensionAction::Added, CreateExtensionRequest(Extension));
    }
}
```

#### 8.1.3 批量操作

```cpp
// 批量注册扩展
class FUIExtensionBatchRegistration
{
public:
    void AddExtension(FGameplayTag Tag, UObject* Data, int32 Priority)
    {
        PendingExtensions.Add({Tag, Data, Priority});
    }

    void CommitAll(UUIExtensionSubsystem* Subsystem)
    {
        // 按Tag分组，减少查找次数
        TMap<FGameplayTag, TArray<FPendingExtension>> GroupedExtensions;
        for (const FPendingExtension& Ext : PendingExtensions)
        {
            GroupedExtensions.FindOrAdd(Ext.Tag).Add(Ext);
        }

        for (const TPair<FGameplayTag, TArray<FPendingExtension>>& Pair : GroupedExtensions)
        {
            for (const FPendingExtension& Ext : Pair.Value)
            {
                Subsystem->RegisterExtensionAsData(Pair.Key, nullptr, Ext.Data, Ext.Priority);
            }
        }

        PendingExtensions.Empty();
    }

private:
    struct FPendingExtension
    {
        FGameplayTag Tag;
        UObject* Data;
        int32 Priority;
    };

    TArray<FPendingExtension> PendingExtensions;
};
```

### 8.2 内存管理

#### 8.2.1 弱引用使用

```cpp
// UIExtension使用TWeakObjectPtr防止循环引用
struct FUIExtension
{
    TWeakObjectPtr<UObject> ContextObject;  // 弱引用！
    TObjectPtr<UObject> Data;  // 强引用（由AddReferencedObjects管理）
};

// 自定义AddReferencedObjects确保数据不被GC
void UUIExtensionSubsystem::AddReferencedObjects(UObject* InThis, FReferenceCollector& Collector)
{
    if (UUIExtensionSubsystem* Subsystem = Cast<UUIExtensionSubsystem>(InThis))
    {
        for (auto& Pair : Subsystem->ExtensionMap)
        {
            for (const TSharedPtr<FUIExtension>& Extension : Pair.Value)
            {
                Collector.AddReferencedObject(Extension->Data);
            }
        }
    }
}
```

#### 8.2.2 及时清理

```cpp
// Widget销毁时清理扩展
void UMyWidget::NativeDestruct()
{
    // 方法1：手动注销
    for (FUIExtensionHandle& Handle : ExtensionHandles)
    {
        Handle.Unregister();
    }
    ExtensionHandles.Empty();

    // 方法2：使用RAII包装器
    ExtensionHandleGuard.Reset();  // 自动调用Unregister

    Super::NativeDestruct();
}

// RAII辅助类
class FUIExtensionHandleGuard
{
public:
    FUIExtensionHandleGuard(FUIExtensionHandle InHandle) : Handle(InHandle) {}
    ~FUIExtensionHandleGuard() { Handle.Unregister(); }

private:
    FUIExtensionHandle Handle;
};
```

### 8.3 调试与诊断

#### 8.3.1 日志输出

```cpp
// 启用详细日志
// DefaultEngine.ini
[Core.Log]
LogUIExtension=Verbose

// 代码中使用
UE_LOG(LogUIExtension, Verbose, TEXT("Extension [%s] registered at [%s]"), 
    *GetNameSafe(Data), *ExtensionPointTag.ToString());
```

#### 8.3.2 调试控制台命令

```cpp
// 自定义控制台命令
#if !UE_BUILD_SHIPPING
static FAutoConsoleCommand CVarDumpUIExtensions(
    TEXT("Lyra.DumpUIExtensions"),
    TEXT("Dumps all registered UI extensions"),
    FConsoleCommandDelegate::CreateLambda([]()
    {
        for (const FWorldContext& Context : GEngine->GetWorldContexts())
        {
            if (UWorld* World = Context.World())
            {
                if (UUIExtensionSubsystem* Subsystem = World->GetSubsystem<UUIExtensionSubsystem>())
                {
                    Subsystem->DumpExtensionsToLog();
                }
            }
        }
    })
);

void UUIExtensionSubsystem::DumpExtensionsToLog()
{
    UE_LOG(LogUIExtension, Display, TEXT("=== UI Extensions Dump ==="));
    
    for (const TPair<FGameplayTag, FExtensionList>& Pair : ExtensionMap)
    {
        UE_LOG(LogUIExtension, Display, TEXT("Tag: %s (%d extensions)"), 
            *Pair.Key.ToString(), Pair.Value.Num());
            
        for (const TSharedPtr<FUIExtension>& Ext : Pair.Value)
        {
            UE_LOG(LogUIExtension, Display, TEXT("  - Data: %s, Priority: %d, Context: %s"),
                *GetNameSafe(Ext->Data), Ext->Priority, *GetNameSafe(Ext->ContextObject.Get()));
        }
    }
}
#endif
```

### 8.4 最佳实践清单

✅ **DO（推荐做法）：**

1. **使用语义化的GameplayTag**：`UI.HUD.Player.Health` 而非 `UI.001`
2. **为扩展设置合理的优先级**：血条(10) > 技能栏(5) > 装饰元素(0)
3. **在NativeDestruct中清理句柄**：避免悬挂引用
4. **使用ContextObject实现玩家特定UI**：多人游戏必需
5. **通过GameFeature模块化UI**：保持代码解耦
6. **使用虚拟化列表显示大量元素**：性能优化
7. **服务器验证所有玩家操作**：防作弊

❌ **DON'T（避免做法）：**

1. **不要在Tick中注册/注销扩展**：性能杀手
2. **不要忘记调用Unregister**：内存泄漏
3. **不要硬编码Widget类引用**：破坏模块化
4. **不要在客户端直接修改复制的数据**：网络同步问题
5. **不要一次性创建过多Widget**：使用懒加载
6. **不要在扩展回调中执行重操作**：可能阻塞主线程
7. **不要混用ExactMatch和PartialMatch而不清楚其含义**：导致意外匹配

---

## 9. 总结与展望

### 9.1 核心要点回顾

UIExtension系统是Lyra架构中最优雅的设计之一，它通过以下机制实现了真正的模块化UI：

1. **发布-订阅模式**：扩展点和扩展完全解耦，通过GameplayTag动态匹配
2. **上下文感知**：支持全局、玩家特定、对象特定的UI注册
3. **GameFeature集成**：插件可以无侵入地添加/移除UI元素
4. **网络友好**：通过ContextObject自然支持多人模式
5. **灵活的数据驱动**：既支持Widget类扩展，也支持数据对象扩展

### 9.2 适用场景

**最适合使用UIExtension的场景：**
- 需要频繁增删UI模块的项目
- 多人在线游戏的玩家特定UI
- 基于DLC/插件的内容扩展
- 复杂的HUD系统（如MOBA、MMO）
- 需要A/B测试不同UI方案

**不太适合的场景：**
- 静态的、永不改变的UI（如主菜单）
- 对性能极度敏感的场景（过度使用会有开销）
- 团队不熟悉GameplayTag和模块化设计

### 9.3 进阶方向

掌握UIExtension后，可以探索：

1. **CommonUI深度集成**：利用Layer系统和Input Routing
2. **自定义扩展点Widget**：继承`UUIExtensionPointWidget`实现特殊布局
3. **数据驱动的UI配置**：通过DataTable配置扩展而非硬编码
4. **编辑器工具**：制作可视化的扩展点调试工具
5. **性能分析**：Profiling扩展注册和Widget创建的开销

### 9.4 实战建议

在实际项目中应用UIExtension时：

1. **从小处开始**：先用于一个简单的HUD元素，逐步扩展
2. **建立命名规范**：团队统一GameplayTag的命名约定
3. **文档化扩展点**：维护所有可用扩展点的列表
4. **版本控制DataAsset**：扩展配置应纳入版本管理
5. **持续重构**：随着项目发展，定期审查和优化扩展点结构

UIExtension不仅仅是一个技术系统，更是一种设计哲学——通过解耦和动态组合，让UI系统具备无限的扩展可能性。掌握它，你将能够构建出真正灵活、可维护的游戏界面系统。

---

**下一章预告：** 
《Lyra实战：完整游戏模式开发》将综合运用前15章的所有知识，从零开始构建一个完整的多人在线射击游戏模式，涵盖Experience配置、GameFeature开发、UI系统集成、网络同步等全流程实战。

---

**参考资源：**
- Lyra源码：`Plugins/UIExtension/`
- Epic官方文档：Modular Gameplay Features
- UE社区讨论：UI Extension Best Practices
- CommonUI文档：Layer Management System

**本章字数：** ~11,000字  
**难度等级：** ⭐⭐⭐⭐☆  
**建议学习时间：** 8-12小时（含实战练习）
