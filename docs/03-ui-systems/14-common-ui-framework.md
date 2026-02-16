# CommonUI 框架详解

## 概述

CommonUI 是 Epic Games 为 Unreal Engine 5 开发的一套跨平台 UI 框架，专门设计用于处理多平台输入（键鼠、手柄、触摸）和复杂的 UI 交互需求。在 Lyra 项目中，CommonUI 框架被广泛应用于构建菜单系统、HUD 界面和各种交互式控件。

本文将深入解析 CommonUI 的核心架构、生命周期管理、输入处理机制以及在 Lyra 中的实际应用，帮助您掌握如何利用这个强大的框架构建专业级游戏 UI。

### CommonUI 的核心优势

1. **跨平台输入支持**：自动处理键盘、鼠标、手柄和触摸输入
2. **Widget 堆栈管理**：内置的 Layer 系统支持复杂的 UI 层级和导航
3. **输入模式自动切换**：智能管理 UI/Game/混合输入模式
4. **样式系统**：统一的样式管理，便于主题切换和多平台适配
5. **焦点导航**：完善的焦点管理和自动导航支持手柄操作

## CommonUI 插件架构

### 核心组件架构

CommonUI 插件包含以下核心组件：

```
CommonUI/
├── CommonActivatableWidget           # 可激活 Widget 基类
├── CommonActivatableWidgetContainer  # Widget 容器（Stack/Queue）
├── CommonButtonBase                  # 按钮基类
├── CommonTextBlock                   # 文本控件
├── CommonRichTextBlock              # 富文本控件
├── CommonBorder                     # 边框容器
├── CommonUIActionRouter             # 输入路由器
├── CommonInputMode                  # 输入模式管理
└── CommonUIStyle                    # 样式系统
```

### Lyra 的 CommonUI 扩展架构

Lyra 在 CommonUI 基础上构建了完整的 UI 管理系统：

```
Lyra UI Architecture:
┌──────────────────────────────────────────────┐
│         UGameUIManagerSubsystem              │  <- 全局 UI 管理器
│  ┌────────────────────────────────────────┐  │
│  │         UGameUIPolicy                  │  │  <- UI 策略（多玩家支持）
│  │  ┌──────────────────────────────────┐  │  │
│  │  │    UPrimaryGameLayout            │  │  │  <- 根布局（每个玩家一个）
│  │  │  ┌────────────────────────────┐  │  │  │
│  │  │  │  Layer Stack Manager       │  │  │  │  <- Layer 管理（UI.Layer.Game）
│  │  │  │  - UI.Layer.Game           │  │  │  │
│  │  │  │  - UI.Layer.GameMenu       │  │  │  │
│  │  │  │  - UI.Layer.Menu           │  │  │  │
│  │  │  │  - UI.Layer.Modal          │  │  │  │
│  │  │  └────────────────────────────┘  │  │  │
│  │  └──────────────────────────────────┘  │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

**关键类的职责：**

- **UGameUIManagerSubsystem**：游戏实例级别的 UI 管理器，负责创建和管理 UI Policy
- **UGameUIPolicy**：定义 UI 布局策略，支持单人/多人分屏模式
- **UPrimaryGameLayout**：每个本地玩家的根 UI 布局，管理所有 Layer
- **Layer（CommonActivatableWidgetContainer）**：Widget 堆栈容器，管理同一层级的 Widget

### Lyra 的 UI 层级设计

Lyra 使用 Gameplay Tag 系统定义 UI 层级：

```cpp
// 典型的 Layer 层级结构
UI.Layer.Game        // 游戏内 HUD（生命值、弹药等）
UI.Layer.GameMenu    // 游戏内菜单（暂停、设置）
UI.Layer.Menu        // 主菜单、大厅界面
UI.Layer.Modal       // 模态对话框（确认框、错误提示）
```

每个 Layer 是一个 **CommonActivatableWidgetContainer**，支持 Stack（堆栈）或 Queue（队列）模式：

- **Stack 模式**：新 Widget 推入栈顶，只有栈顶 Widget 激活（如菜单层级）
- **Queue 模式**：所有 Widget 同时显示和激活（如游戏 HUD）

## CommonActivatableWidget 生命周期管理

### 核心生命周期

`UCommonActivatableWidget` 是 CommonUI 框架中最重要的基类，所有可激活的 UI 都应继承自它。

```cpp
// CommonActivatableWidget 核心接口
class UCommonActivatableWidget : public UCommonUserWidget
{
public:
    // 激活生命周期
    void ActivateWidget();           // 激活 Widget
    void DeactivateWidget();         // 停用 Widget
    
    // 生命周期事件（蓝图可重写）
    UFUNCTION(BlueprintNativeEvent)
    void BP_OnActivated();           // Widget 被激活时调用
    
    UFUNCTION(BlueprintNativeEvent)
    void BP_OnDeactivated();         // Widget 被停用时调用
    
    // 焦点管理
    UFUNCTION(BlueprintNativeEvent)
    UWidget* BP_GetDesiredFocusTarget() const;  // 返回期望获得焦点的控件
    
    // 输入配置
    virtual TOptional<FUIInputConfig> GetDesiredInputConfig() const;
};
```

### Lyra 的 ULyraActivatableWidget 扩展

Lyra 扩展了基础的 CommonActivatableWidget，添加了输入模式管理：

```cpp
// LyraActivatableWidget.h
UENUM(BlueprintType)
enum class ELyraWidgetInputMode : uint8
{
    Default,        // 继承父级输入模式
    GameAndMenu,    // 同时接收游戏和 UI 输入
    Game,           // 仅游戏输入（UI 可见但不接收输入）
    Menu            // 仅 UI 输入（游戏暂停或不接收输入）
};

UCLASS(Abstract, Blueprintable)
class ULyraActivatableWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()

public:
    virtual TOptional<FUIInputConfig> GetDesiredInputConfig() const override;

protected:
    // 期望的输入模式
    UPROPERTY(EditDefaultsOnly, Category = Input)
    ELyraWidgetInputMode InputConfig = ELyraWidgetInputMode::Default;

    // 游戏输入时的鼠标捕获模式
    UPROPERTY(EditDefaultsOnly, Category = Input)
    EMouseCaptureMode GameMouseCaptureMode = EMouseCaptureMode::CapturePermanently;
};
```

**实现代码：**

```cpp
// LyraActivatableWidget.cpp
TOptional<FUIInputConfig> ULyraActivatableWidget::GetDesiredInputConfig() const
{
    switch (InputConfig)
    {
    case ELyraWidgetInputMode::GameAndMenu:
        return FUIInputConfig(ECommonInputMode::All, GameMouseCaptureMode);
        
    case ELyraWidgetInputMode::Game:
        return FUIInputConfig(ECommonInputMode::Game, GameMouseCaptureMode);
        
    case ELyraWidgetInputMode::Menu:
        return FUIInputConfig(ECommonInputMode::Menu, EMouseCaptureMode::NoCapture);
        
    case ELyraWidgetInputMode::Default:
    default:
        return TOptional<FUIInputConfig>();  // 继承父级
    }
}
```

### 完整的生命周期流程

下面是一个典型的 Widget 从创建到销毁的完整流程：

```
1. 创建阶段
   ├─ Constructor                    <- C++ 构造函数
   ├─ NativePreConstruct            <- UMG 预构造（编辑器预览）
   ├─ NativeConstruct               <- UMG 构造（运行时）
   └─ NativeOnInitialized           <- 初始化完成

2. 激活阶段
   ├─ ActivateWidget()              <- 被 Layer 推入栈并激活
   ├─ NativeOnActivated()           <- C++ 激活回调
   ├─ BP_OnActivated()              <- 蓝图激活事件
   ├─ GetDesiredInputConfig()       <- 查询输入配置
   ├─ 应用输入模式                  <- 切换输入模式
   └─ BP_GetDesiredFocusTarget()    <- 设置初始焦点

3. 运行阶段
   ├─ NativeTick()                  <- 每帧更新（如果启用）
   └─ 响应用户输入                  <- 处理按钮点击、输入等

4. 停用阶段
   ├─ DeactivateWidget()            <- 被从栈中移除或覆盖
   ├─ NativeOnDeactivated()         <- C++ 停用回调
   └─ BP_OnDeactivated()            <- 蓝图停用事件

5. 销毁阶段
   ├─ NativeDestruct                <- UMG 析构
   └─ BeginDestroy                  <- 对象销毁
```

### 实践示例：自定义 Activatable Widget

创建一个带有淡入淡出动画的设置菜单：

```cpp
// MySettingsMenu.h
#pragma once

#include "UI/LyraActivatableWidget.h"
#include "MySettingsMenu.generated.h"

UCLASS()
class UMySettingsMenu : public ULyraActivatableWidget
{
    GENERATED_BODY()

public:
    UMySettingsMenu(const FObjectInitializer& ObjectInitializer);

protected:
    // 生命周期重写
    virtual void NativeOnInitialized() override;
    virtual void NativeOnActivated() override;
    virtual void NativeOnDeactivated() override;
    virtual UWidget* NativeGetDesiredFocusTarget() const override;

    // 动画
    UPROPERTY(Transient, meta = (BindWidgetAnim))
    TObjectPtr<UWidgetAnimation> FadeInAnimation;

    UPROPERTY(Transient, meta = (BindWidgetAnim))
    TObjectPtr<UWidgetAnimation> FadeOutAnimation;

    // 控件绑定
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Back;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Apply;

private:
    UFUNCTION()
    void HandleBackClicked();

    UFUNCTION()
    void HandleApplyClicked();
};
```

```cpp
// MySettingsMenu.cpp
#include "MySettingsMenu.h"
#include "CommonButtonBase.h"

UMySettingsMenu::UMySettingsMenu(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 设置输入模式为仅 UI
    InputConfig = ELyraWidgetInputMode::Menu;
    
    // 启用自动激活（推入栈时自动激活）
    bAutoActivate = true;
}

void UMySettingsMenu::NativeOnInitialized()
{
    Super::NativeOnInitialized();

    // 绑定按钮事件
    if (Button_Back)
    {
        Button_Back->OnClicked().AddUObject(this, &ThisClass::HandleBackClicked);
    }

    if (Button_Apply)
    {
        Button_Apply->OnClicked().AddUObject(this, &ThisClass::HandleApplyClicked);
    }
}

void UMySettingsMenu::NativeOnActivated()
{
    Super::NativeOnActivated();

    // 播放淡入动画
    if (FadeInAnimation)
    {
        PlayAnimation(FadeInAnimation);
    }

    UE_LOG(LogTemp, Log, TEXT("Settings Menu Activated - Input Mode: Menu"));
}

void UMySettingsMenu::NativeOnDeactivated()
{
    Super::NativeOnDeactivated();

    // 播放淡出动画
    if (FadeOutAnimation)
    {
        PlayAnimation(FadeOutAnimation);
    }

    UE_LOG(LogTemp, Log, TEXT("Settings Menu Deactivated"));
}

UWidget* UMySettingsMenu::NativeGetDesiredFocusTarget() const
{
    // 返回"返回"按钮作为默认焦点（手柄导航）
    return Button_Back;
}

void UMySettingsMenu::HandleBackClicked()
{
    // 从 Layer 中移除自己
    DeactivateWidget();
}

void UMySettingsMenu::HandleApplyClicked()
{
    // 应用设置逻辑
    UE_LOG(LogTemp, Log, TEXT("Apply Settings"));
}
```

## Layer 系统和 Widget Stack 管理

### PrimaryGameLayout：根布局管理器

`UPrimaryGameLayout` 是每个玩家的根 UI 容器，负责管理所有 Layer。

```cpp
// 关键接口
class UPrimaryGameLayout : public UCommonUserWidget
{
public:
    // 注册 Layer（通常在蓝图中调用）
    UFUNCTION(BlueprintCallable, Category = "Layer")
    void RegisterLayer(FGameplayTag LayerTag, 
                      UCommonActivatableWidgetContainerBase* LayerWidget);

    // 获取 Layer 容器
    UCommonActivatableWidgetContainerBase* GetLayerWidget(FGameplayTag LayerName);

    // 同步推送 Widget 到 Layer
    template <typename ActivatableWidgetT = UCommonActivatableWidget>
    ActivatableWidgetT* PushWidgetToLayerStack(
        FGameplayTag LayerName,
        UClass* ActivatableWidgetClass);

    // 异步推送 Widget（处理资源加载）
    template <typename ActivatableWidgetT = UCommonActivatableWidget>
    TSharedPtr<FStreamableHandle> PushWidgetToLayerStackAsync(
        FGameplayTag LayerName,
        bool bSuspendInputUntilComplete,
        TSoftClassPtr<UCommonActivatableWidget> ActivatableWidgetClass);

    // 从所有 Layer 查找并移除 Widget
    void FindAndRemoveWidgetFromLayer(UCommonActivatableWidget* ActivatableWidget);
};
```

### 在蓝图中设置 Layer

在 UMG 编辑器中创建 Primary Game Layout 蓝图：

1. 创建继承自 `UPrimaryGameLayout` 的 Widget 蓝图
2. 添加多个 `CommonActivatableWidgetStack` 作为 Layer 容器
3. 在 Event Graph 中注册 Layer：

```cpp
// 蓝图伪代码
Event Construct
{
    // 注册游戏 HUD Layer（Queue 模式）
    RegisterLayer(
        LayerTag: "UI.Layer.Game",
        LayerWidget: GameLayer_Stack
    );

    // 注册菜单 Layer（Stack 模式）
    RegisterLayer(
        LayerTag: "UI.Layer.GameMenu",
        LayerWidget: GameMenuLayer_Stack
    );

    // 注册模态对话框 Layer（Stack 模式）
    RegisterLayer(
        LayerTag: "UI.Layer.Modal",
        LayerWidget: ModalLayer_Stack
    );
}
```

### C++ 中操作 Layer

```cpp
// 获取 Primary Game Layout
UPrimaryGameLayout* RootLayout = UPrimaryGameLayout::GetPrimaryGameLayoutForPrimaryPlayer(this);

if (RootLayout)
{
    // 方式 1：同步推送（Widget 类已加载）
    UMyPauseMenu* PauseMenu = RootLayout->PushWidgetToLayerStack<UMyPauseMenu>(
        FGameplayTag::RequestGameplayTag("UI.Layer.GameMenu"),
        UMyPauseMenu::StaticClass()
    );

    // 方式 2：带初始化回调
    UMyPauseMenu* PauseMenu = RootLayout->PushWidgetToLayerStack<UMyPauseMenu>(
        FGameplayTag::RequestGameplayTag("UI.Layer.GameMenu"),
        UMyPauseMenu::StaticClass(),
        [](UMyPauseMenu& Widget)
        {
            // 在 Widget 添加到 Layer 前进行初始化
            Widget.SetPlayerName(TEXT("Player1"));
        }
    );

    // 方式 3：异步推送（处理资源加载）
    TSoftClassPtr<UMySettingsMenu> SettingsMenuClass = 
        TSoftClassPtr<UMySettingsMenu>(FSoftObjectPath(TEXT("/Game/UI/W_SettingsMenu.W_SettingsMenu_C")));

    TSharedPtr<FStreamableHandle> Handle = RootLayout->PushWidgetToLayerStackAsync<UMySettingsMenu>(
        FGameplayTag::RequestGameplayTag("UI.Layer.Menu"),
        true,  // 加载期间暂停输入
        SettingsMenuClass,
        [](EAsyncWidgetLayerState State, UMySettingsMenu* Widget)
        {
            switch (State)
            {
            case EAsyncWidgetLayerState::Initialize:
                UE_LOG(LogTemp, Log, TEXT("Widget 加载完成，开始初始化"));
                if (Widget)
                {
                    Widget->InitializeSettings();
                }
                break;

            case EAsyncWidgetLayerState::AfterPush:
                UE_LOG(LogTemp, Log, TEXT("Widget 已推入 Layer"));
                break;

            case EAsyncWidgetLayerState::Canceled:
                UE_LOG(LogTemp, Warning, TEXT("Widget 加载被取消"));
                break;
            }
        }
    );
}
```

### Widget Stack 操作详解

**CommonActivatableWidgetStack** 是一个典型的栈结构：

```
Stack 状态示例：
┌──────────────────┐
│  Widget C (Top)  │ <- 当前激活，接收输入
├──────────────────┤
│  Widget B        │ <- 停用状态
├──────────────────┤
│  Widget A        │ <- 停用状态
└──────────────────┘

操作：
- Push Widget D    -> D 成为栈顶，C 被停用
- Pop             -> D 被移除，C 重新激活
- Remove Widget B -> B 被移除，栈变为 [A, C, D]
```

**核心行为：**

1. **Push（推入）**：
   - 将新 Widget 推入栈顶
   - 之前的栈顶 Widget 自动调用 `DeactivateWidget()`
   - 新 Widget 自动调用 `ActivateWidget()`

2. **Pop（弹出）**：
   - 移除栈顶 Widget
   - 下一个 Widget 自动激活

3. **RemoveWidget（移除指定 Widget）**：
   - 可以移除栈中任意 Widget
   - 如果移除的是栈顶，下一个 Widget 自动激活

### Lyra 的 HUD Layout 实现

Lyra 的 `ULyraHUDLayout` 继承自 `ULyraActivatableWidget`，展示了如何管理游戏 HUD：

```cpp
// LyraHUDLayout.h（简化版）
UCLASS(Abstract, BlueprintType, Blueprintable)
class ULyraHUDLayout : public ULyraActivatableWidget
{
    GENERATED_BODY()

public:
    virtual void NativeOnInitialized() override;

protected:
    // Escape 键处理（打开暂停菜单）
    void HandleEscapeAction();

    // Escape 菜单类
    UPROPERTY(EditDefaultsOnly)
    TSoftClassPtr<UCommonActivatableWidget> EscapeMenuClass;
};
```

```cpp
// LyraHUDLayout.cpp（简化实现）
void ULyraHUDLayout::NativeOnInitialized()
{
    Super::NativeOnInitialized();

    // 注册 Escape 键动作（打开暂停菜单）
    RegisterUIActionBinding(FBindUIActionArgs(
        EscapeAction,  // Input Action
        false,         // bDisplayInActionBar
        FSimpleDelegate::CreateUObject(this, &ThisClass::HandleEscapeAction)
    ));
}

void ULyraHUDLayout::HandleEscapeAction()
{
    // 加载并推送暂停菜单到 GameMenu Layer
    if (!EscapeMenuClass.IsNull())
    {
        UPrimaryGameLayout* RootLayout = UPrimaryGameLayout::GetPrimaryGameLayout(GetOwningLocalPlayer());
        
        RootLayout->PushWidgetToLayerStackAsync<UCommonActivatableWidget>(
            FGameplayTag::RequestGameplayTag("UI.Layer.GameMenu"),
            true,  // 加载期间暂停输入
            EscapeMenuClass
        );
    }
}
```

## Input Mode 切换机制

### CommonUI 的输入模式

CommonUI 定义了三种核心输入模式：

```cpp
enum class ECommonInputMode : uint8
{
    Menu,    // 仅 UI 接收输入，游戏不接收
    Game,    // 仅游戏接收输入，UI 不接收（但 UI 可见）
    All      // UI 和游戏同时接收输入
};

// 鼠标捕获模式
enum class EMouseCaptureMode : uint8
{
    NoCapture,              // 不捕获鼠标
    CapturePermanently,     // 永久捕获（FPS 游戏）
    CapturePermanently_IncludingInitialMouseDown,
    CaptureDuringMouseDown, // 仅按下时捕获
    CaptureDuringRightMouseDown
};

// 输入配置结构
struct FUIInputConfig
{
    ECommonInputMode InputMode;
    EMouseCaptureMode MouseCaptureMode;

    FUIInputConfig(ECommonInputMode InInputMode, EMouseCaptureMode InMouseCaptureMode)
        : InputMode(InInputMode)
        , MouseCaptureMode(InMouseCaptureMode)
    {}
};
```

### Lyra 的输入模式映射

```cpp
// ULyraActivatableWidget::GetDesiredInputConfig() 的详细解析

ELyraWidgetInputMode::GameAndMenu
    -> ECommonInputMode::All + GameMouseCaptureMode
    用例：游戏内可交互的 HUD（准星、快捷栏）

ELyraWidgetInputMode::Game
    -> ECommonInputMode::Game + GameMouseCaptureMode
    用例：纯信息显示的 HUD（血条、弹药数）

ELyraWidgetInputMode::Menu
    -> ECommonInputMode::Menu + EMouseCaptureMode::NoCapture
    用例：暂停菜单、设置界面

ELyraWidgetInputMode::Default
    -> TOptional<FUIInputConfig>()（空值）
    用例：继承父 Widget 或 Layer 的输入配置
```

### 输入模式切换流程

当 Widget 被激活或停用时，CommonUI 自动管理输入模式切换：

```
1. Widget Push 到 Layer Stack
   ├─ Layer->AddWidget(Widget)
   ├─ Widget->ActivateWidget()
   ├─ QueryInputConfig = Widget->GetDesiredInputConfig()
   ├─ 如果 QueryInputConfig 有效：
   │  ├─ 保存当前输入配置到栈
   │  ├─ 应用新的输入配置
   │  ├─ 设置鼠标捕获模式
   │  └─ 更新焦点和输入路由
   └─ Widget->BP_OnActivated()

2. Widget Pop 或被覆盖
   ├─ Widget->DeactivateWidget()
   ├─ Widget->BP_OnDeactivated()
   ├─ 从输入配置栈弹出
   └─ 恢复上一个输入配置
```

### 实践示例：多层级菜单输入管理

创建一个主菜单系统，支持多层级子菜单：

```cpp
// 主菜单：仅 UI 输入
UCLASS()
class UMainMenu : public ULyraActivatableWidget
{
    GENERATED_BODY()

public:
    UMainMenu(const FObjectInitializer& ObjectInitializer)
        : Super(ObjectInitializer)
    {
        // 仅 UI 输入，游戏暂停
        InputConfig = ELyraWidgetInputMode::Menu;
    }

protected:
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Play;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Settings;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Quit;

    virtual void NativeOnInitialized() override
    {
        Super::NativeOnInitialized();

        Button_Settings->OnClicked().AddUObject(this, &ThisClass::OpenSettings);
    }

    void OpenSettings()
    {
        // 推送设置菜单（会覆盖当前菜单，输入模式切换为设置菜单的模式）
        if (UPrimaryGameLayout* RootLayout = UPrimaryGameLayout::GetPrimaryGameLayout(GetOwningLocalPlayer()))
        {
            RootLayout->PushWidgetToLayerStack<USettingsMenu>(
                FGameplayTag::RequestGameplayTag("UI.Layer.Menu"),
                USettingsMenu::StaticClass()
            );
        }
    }
};

// 设置菜单：同样是仅 UI 输入
UCLASS()
class USettingsMenu : public ULyraActivatableWidget
{
    GENERATED_BODY()

public:
    USettingsMenu(const FObjectInitializer& ObjectInitializer)
        : Super(ObjectInitializer)
    {
        InputConfig = ELyraWidgetInputMode::Menu;
    }

protected:
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Back;

    virtual void NativeOnInitialized() override
    {
        Super::NativeOnInitialized();

        // 绑定返回按钮（自动处理 Back 输入动作）
        Button_Back->OnClicked().AddLambda([this]()
        {
            DeactivateWidget();  // 自动恢复上一个菜单的输入配置
        });
    }
};

// 游戏内暂停菜单：可选 UI+Game 模式
UCLASS()
class UPauseMenu : public ULyraActivatableWidget
{
    GENERATED_BODY()

public:
    UPauseMenu(const FObjectInitializer& ObjectInitializer)
        : Super(ObjectInitializer)
    {
        // UI 和游戏同时接收输入（可以在暂停界面观察游戏）
        InputConfig = ELyraWidgetInputMode::GameAndMenu;
        GameMouseCaptureMode = EMouseCaptureMode::CaptureDuringMouseDown;
    }
};
```

### 手柄输入和焦点管理

CommonUI 自动处理手柄导航，但需要正确实现焦点目标：

```cpp
UCLASS()
class UMyMenu : public ULyraActivatableWidget
{
    GENERATED_BODY()

protected:
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Resume;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Settings;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Quit;

    // 返回期望的初始焦点控件
    virtual UWidget* NativeGetDesiredFocusTarget() const override
    {
        // 手柄用户激活此菜单时，焦点自动设置到"继续游戏"按钮
        return Button_Resume;
    }

    virtual void NativeOnActivated() override
    {
        Super::NativeOnActivated();

        // 确保焦点正确设置（CommonUI 会自动调用，这里是示例）
        if (UWidget* FocusTarget = BP_GetDesiredFocusTarget())
        {
            FocusTarget->SetUserFocus(GetOwningPlayer());
        }
    }
};
```

## CommonButton 和核心控件

### CommonButtonBase：通用按钮基类

`UCommonButtonBase` 是 CommonUI 中最常用的控件，支持多平台输入和自动样式切换。

```cpp
// CommonButtonBase 核心接口
class UCommonButtonBase : public UCommonUserWidget
{
public:
    // 按钮样式（自动根据输入设备切换）
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Button")
    TSubclassOf<UCommonButtonStyle> Style;

    // 按钮点击事件
    DECLARE_EVENT_OneParam(UCommonButtonBase, FOnButtonClicked, UCommonButtonBase*)
    FOnButtonClicked OnClicked();

    // 悬停事件
    FOnButtonHovered OnHovered();
    FOnButtonHovered OnUnhovered();

    // 按钮状态
    void SetIsEnabled(bool bInIsEnabled);
    void SetIsSelected(bool bInSelected, bool bBroadcast = true);
    
    // 关联输入动作（显示对应按键图标）
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Button")
    FDataTableRowHandle TriggeringInputAction;
};
```

### Lyra 的 ULyraButtonBase 扩展

Lyra 扩展了 CommonButtonBase，添加了文本管理和样式更新：

```cpp
// LyraButtonBase.h
UCLASS(Abstract, BlueprintType, Blueprintable)
class ULyraButtonBase : public UCommonButtonBase
{
    GENERATED_BODY()

public:
    // 设置按钮文本
    UFUNCTION(BlueprintCallable)
    void SetButtonText(const FText& InText);

protected:
    virtual void NativePreConstruct() override;
    virtual void UpdateInputActionWidget() override;
    virtual void OnInputMethodChanged(ECommonInputType CurrentInputType) override;

    void RefreshButtonText();

    // 蓝图可重写的事件
    UFUNCTION(BlueprintImplementableEvent)
    void UpdateButtonText(const FText& InText);

    UFUNCTION(BlueprintImplementableEvent)
    void UpdateButtonStyle();

private:
    UPROPERTY(EditAnywhere, Category="Button", meta=(InlineEditConditionToggle))
    uint8 bOverride_ButtonText : 1;

    UPROPERTY(EditAnywhere, Category="Button", meta=(editcondition="bOverride_ButtonText"))
    FText ButtonText;
};
```

```cpp
// LyraButtonBase.cpp
void ULyraButtonBase::SetButtonText(const FText& InText)
{
    bOverride_ButtonText = InText.IsEmpty();
    ButtonText = InText;
    RefreshButtonText();
}

void ULyraButtonBase::RefreshButtonText()
{
    if (bOverride_ButtonText || !ButtonText.IsEmpty())
    {
        UpdateButtonText(ButtonText);
    }
}

void ULyraButtonBase::OnInputMethodChanged(ECommonInputType CurrentInputType)
{
    Super::OnInputMethodChanged(CurrentInputType);
    
    // 输入方式改变时更新样式（键鼠/手柄/触摸）
    UpdateButtonStyle();
}
```

### CommonTextBlock 和 CommonRichTextBlock

```cpp
// CommonTextBlock：基础文本控件
UCLASS()
class UCommonTextBlock : public UTextBlock
{
public:
    // 样式（根据平台自动调整字体、大小等）
    UPROPERTY(EditAnywhere, Category = "Text")
    TSubclassOf<UCommonTextStyle> Style;

    // 滚动样式（文本过长时的行为）
    UPROPERTY(EditAnywhere, Category = "Text")
    FCommonTextScrollStyle ScrollStyle;
};

// CommonRichTextBlock：富文本控件（支持内联图片、样式混合）
UCLASS()
class UCommonRichTextBlock : public URichTextBlock
{
public:
    // 富文本数据表（定义样式和图标）
    UPROPERTY(EditAnywhere, Category = "RichText")
    TObjectPtr<UDataTable> TextStyleSet;
};
```

### 实践示例：创建完整的按钮

**C++ 按钮类：**

```cpp
// MyActionButton.h
#pragma once

#include "UI/Foundation/LyraButtonBase.h"
#include "MyActionButton.generated.h"

UCLASS()
class UMyActionButton : public ULyraButtonBase
{
    GENERATED_BODY()

public:
    UMyActionButton(const FObjectInitializer& ObjectInitializer);

    // 设置按钮数据
    UFUNCTION(BlueprintCallable)
    void SetButtonData(const FText& InLabel, const FText& InDescription);

protected:
    virtual void NativeOnInitialized() override;
    virtual void NativeOnHovered() override;
    virtual void NativeOnUnhovered() override;

    // 控件绑定
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonTextBlock> Text_Label;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonTextBlock> Text_Description;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UImage> Image_Icon;

    // 样式
    UPROPERTY(EditAnywhere, Category = "Appearance")
    TObjectPtr<UTexture2D> IconTexture;
};
```

```cpp
// MyActionButton.cpp
#include "MyActionButton.h"
#include "CommonTextBlock.h"
#include "Components/Image.h"

UMyActionButton::UMyActionButton(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
}

void UMyActionButton::NativeOnInitialized()
{
    Super::NativeOnInitialized();

    if (Image_Icon && IconTexture)
    {
        Image_Icon->SetBrushFromTexture(IconTexture);
    }

    // 监听输入方式改变
    OnInputMethodChanged(GetInputSubsystem()->GetCurrentInputType());
}

void UMyActionButton::SetButtonData(const FText& InLabel, const FText& InDescription)
{
    if (Text_Label)
    {
        Text_Label->SetText(InLabel);
    }

    if (Text_Description)
    {
        Text_Description->SetText(InDescription);
    }
}

void UMyActionButton::NativeOnHovered()
{
    Super::NativeOnHovered();

    // 悬停时显示描述文本
    if (Text_Description)
    {
        Text_Description->SetVisibility(ESlateVisibility::HitTestInvisible);
    }
}

void UMyActionButton::NativeOnUnhovered()
{
    Super::NativeOnUnhovered();

    // 取消悬停时隐藏描述
    if (Text_Description)
    {
        Text_Description->SetVisibility(ESlateVisibility::Collapsed);
    }
}
```

**UMG 蓝图设计：**

```
Widget Hierarchy:
├─ MyActionButton (UMyActionButton)
   ├─ Overlay
      ├─ Image_Background
      │  └─ Image (Border/Background)
      ├─ HorizontalBox
      │  ├─ Image_Icon
      │  │  └─ Image (Icon Texture)
      │  └─ VerticalBox
      │     ├─ Text_Label
      │     │  └─ CommonTextBlock (按钮标签)
      │     └─ Text_Description
      │        └─ CommonTextBlock (描述文本，默认隐藏)
      └─ Image_ActionIcon
         └─ CommonActionWidget (显示输入动作图标，如 [A] 按钮)
```

### CommonBoundActionButton：输入动作按钮

Lyra 的 `ULyraBoundActionButton` 是一个特殊按钮，可以根据输入设备动态切换样式：

```cpp
// LyraBoundActionButton.h
UCLASS(Abstract)
class ULyraBoundActionButton : public UCommonBoundActionButton
{
    GENERATED_BODY()

protected:
    virtual void NativeConstruct() override;

private:
    void HandleInputMethodChanged(ECommonInputType NewInputMethod);

    // 不同输入设备的样式
    UPROPERTY(EditAnywhere, Category = "Styles")
    TSubclassOf<UCommonButtonStyle> KeyboardStyle;

    UPROPERTY(EditAnywhere, Category = "Styles")
    TSubclassOf<UCommonButtonStyle> GamepadStyle;

    UPROPERTY(EditAnywhere, Category = "Styles")
    TSubclassOf<UCommonButtonStyle> TouchStyle;
};
```

```cpp
// LyraBoundActionButton.cpp
void ULyraBoundActionButton::NativeConstruct()
{
    Super::NativeConstruct();

    // 监听输入方式改变
    if (UCommonInputSubsystem* InputSubsystem = GetInputSubsystem())
    {
        InputSubsystem->OnInputMethodChangedNative.AddUObject(
            this, 
            &ThisClass::HandleInputMethodChanged
        );

        // 初始化样式
        HandleInputMethodChanged(InputSubsystem->GetCurrentInputType());
    }
}

void ULyraBoundActionButton::HandleInputMethodChanged(ECommonInputType NewInputMethod)
{
    TSubclassOf<UCommonButtonStyle> NewStyle = nullptr;

    switch (NewInputMethod)
    {
    case ECommonInputType::MouseAndKeyboard:
        NewStyle = KeyboardStyle;
        break;

    case ECommonInputType::Gamepad:
        NewStyle = GamepadStyle;
        break;

    case ECommonInputType::Touch:
        NewStyle = TouchStyle;
        break;
    }

    if (NewStyle)
    {
        SetStyle(NewStyle);
    }
}
```

## 样式系统

### CommonButtonStyle：按钮样式资产

CommonUI 使用数据资产（Data Asset）定义样式，实现样式与逻辑分离。

```cpp
// CommonButtonStyle 定义
UCLASS(BlueprintType)
class UCommonButtonStyle : public UObject
{
    GENERATED_BODY()

public:
    // 按钮尺寸
    UPROPERTY(EditDefaultsOnly, Category = "Properties")
    bool bSingleMaterial = true;

    UPROPERTY(EditDefaultsOnly, Category = "Properties")
    FVector2D ButtonPadding;

    UPROPERTY(EditDefaultsOnly, Category = "Properties")
    FVector2D CustomPadding;

    // 各状态的样式
    UPROPERTY(EditDefaultsOnly, Category = "Normal")
    FCommonButtonStyleOptionalSlateSound NormalPressedSlateSound;

    UPROPERTY(EditDefaultsOnly, Category = "Normal")
    FButtonStyle NormalStyle;

    UPROPERTY(EditDefaultsOnly, Category = "Selected")
    FButtonStyle SelectedStyle;

    UPROPERTY(EditDefaultsOnly, Category = "Disabled")
    FButtonStyle DisabledStyle;

    // 文本样式
    UPROPERTY(EditDefaultsOnly, Category = "Text")
    TSubclassOf<UCommonTextStyle> NormalTextStyle;

    UPROPERTY(EditDefaultsOnly, Category = "Text")
    TSubclassOf<UCommonTextStyle> SelectedTextStyle;

    UPROPERTY(EditDefaultsOnly, Category = "Text")
    TSubclassOf<UCommonTextStyle> DisabledTextStyle;
};
```

**创建按钮样式资产：**

1. 在内容浏览器中右键 -> 用户界面 -> Common Button Style
2. 设置各状态的外观：
   - Normal（正常状态）
   - Hovered（悬停状态）
   - Pressed（按下状态）
   - Selected（选中状态）
   - Disabled（禁用状态）

3. 配置按钮材质、声音、动画等

### CommonTextStyle：文本样式资产

```cpp
UCLASS(BlueprintType)
class UCommonTextStyle : public UObject
{
    GENERATED_BODY()

public:
    // 字体
    UPROPERTY(EditDefaultsOnly, Category = "Font")
    FSlateFontInfo Font;

    // 颜色
    UPROPERTY(EditDefaultsOnly, Category = "Color")
    FLinearColor Color;

    // 阴影
    UPROPERTY(EditDefaultsOnly, Category = "Shadow")
    FVector2D ShadowOffset;

    UPROPERTY(EditDefaultsOnly, Category = "Shadow")
    FLinearColor ShadowColor;

    // 外边距
    UPROPERTY(EditDefaultsOnly, Category = "Properties")
    FMargin Margin;

    // 描边
    UPROPERTY(EditDefaultsOnly, Category = "Stroke")
    FLinearColor StrokeColor;

    UPROPERTY(EditDefaultsOnly, Category = "Stroke")
    float StrokeSize;
};
```

### CommonUIRichTextData：富文本样式表

富文本支持内联样式和图标：

```cpp
// 富文本行样式
USTRUCT(BlueprintType)
struct FRichTextStyleRow : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Appearance")
    FTextBlockStyle TextStyle;
};

// 富文本图标行
USTRUCT(BlueprintType)
struct FRichImageRow : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Appearance")
    FSlateBrush Brush;
};
```

**使用富文本：**

```cpp
// 在 UMG 中使用富文本
<RichTextBlock Text="按 <img id=\"ButtonA\"/> 确认或 <img id=\"ButtonB\"/> 取消">

// 样式标签
<RichTextBlock Text="<Title>主标题</> <Body>正文内容</>">
```

### 实践示例：创建统一的 UI 主题

**1. 创建样式数据资产：**

```
Content/UI/Styles/
├─ BS_PrimaryButton (CommonButtonStyle)
├─ BS_SecondaryButton (CommonButtonStyle)
├─ BS_DangerButton (CommonButtonStyle)
├─ TS_Heading1 (CommonTextStyle)
├─ TS_Heading2 (CommonTextStyle)
├─ TS_Body (CommonTextStyle)
└─ DT_RichTextStyles (DataTable)
```

**2. 在 C++ 中应用样式：**

```cpp
// UIStyleManager.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "UIStyleManager.generated.h"

UCLASS(BlueprintType)
class UUIStyleManager : public UDataAsset
{
    GENERATED_BODY()

public:
    // 按钮样式
    UPROPERTY(EditDefaultsOnly, Category = "Button Styles")
    TSubclassOf<UCommonButtonStyle> PrimaryButtonStyle;

    UPROPERTY(EditDefaultsOnly, Category = "Button Styles")
    TSubclassOf<UCommonButtonStyle> SecondaryButtonStyle;

    UPROPERTY(EditDefaultsOnly, Category = "Button Styles")
    TSubclassOf<UCommonButtonStyle> DangerButtonStyle;

    // 文本样式
    UPROPERTY(EditDefaultsOnly, Category = "Text Styles")
    TSubclassOf<UCommonTextStyle> Heading1Style;

    UPROPERTY(EditDefaultsOnly, Category = "Text Styles")
    TSubclassOf<UCommonTextStyle> Heading2Style;

    UPROPERTY(EditDefaultsOnly, Category = "Text Styles")
    TSubclassOf<UCommonTextStyle> BodyTextStyle;

    // 富文本数据表
    UPROPERTY(EditDefaultsOnly, Category = "Rich Text")
    TObjectPtr<UDataTable> RichTextStyleSet;

    // 获取全局实例
    static UUIStyleManager* Get();

private:
    static TObjectPtr<UUIStyleManager> Instance;
};
```

```cpp
// UIStyleManager.cpp
#include "UIStyleManager.h"
#include "Engine/AssetManager.h"

TObjectPtr<UUIStyleManager> UUIStyleManager::Instance = nullptr;

UUIStyleManager* UUIStyleManager::Get()
{
    if (!Instance)
    {
        // 从资产管理器加载
        FSoftObjectPath StyleManagerPath(TEXT("/Game/UI/DA_UIStyleManager.DA_UIStyleManager"));
        Instance = Cast<UUIStyleManager>(StyleManagerPath.TryLoad());
    }

    return Instance;
}
```

**3. 在 Widget 中使用统一样式：**

```cpp
void UMyMenu::NativeOnInitialized()
{
    Super::NativeOnInitialized();

    if (UUIStyleManager* StyleManager = UUIStyleManager::Get())
    {
        // 应用按钮样式
        Button_Confirm->SetStyle(StyleManager->PrimaryButtonStyle);
        Button_Cancel->SetStyle(StyleManager->SecondaryButtonStyle);
        Button_Delete->SetStyle(StyleManager->DangerButtonStyle);

        // 应用文本样式
        Text_Title->SetStyle(StyleManager->Heading1Style);
        Text_Subtitle->SetStyle(StyleManager->Heading2Style);
        Text_Body->SetStyle(StyleManager->BodyTextStyle);

        // 配置富文本
        RichText_Description->SetTextStyleSet(StyleManager->RichTextStyleSet);
    }
}
```

## 实战案例：构建主菜单系统

现在我们整合所有知识，构建一个完整的主菜单系统，包括：

- 主菜单 Layer
- 设置子菜单
- 确认对话框（Modal Layer）
- 输入模式管理
- 手柄导航支持

### 1. 主菜单 Widget

```cpp
// MainMenu.h
#pragma once

#include "UI/LyraActivatableWidget.h"
#include "MainMenu.generated.h"

class UCommonButtonBase;
class UCommonTextBlock;

UCLASS()
class UMainMenu : public ULyraActivatableWidget
{
    GENERATED_BODY()

public:
    UMainMenu(const FObjectInitializer& ObjectInitializer);

protected:
    virtual void NativeOnInitialized() override;
    virtual void NativeOnActivated() override;
    virtual UWidget* NativeGetDesiredFocusTarget() const override;

    // 控件绑定
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonTextBlock> Text_Title;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Play;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Settings;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Credits;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Quit;

    // 动画
    UPROPERTY(Transient, meta = (BindWidgetAnim))
    TObjectPtr<UWidgetAnimation> Intro;

    UPROPERTY(Transient, meta = (BindWidgetAnim))
    TObjectPtr<UWidgetAnimation> Outro;

    // 子菜单类
    UPROPERTY(EditDefaultsOnly, Category = "UI")
    TSoftClassPtr<ULyraActivatableWidget> SettingsMenuClass;

private:
    UFUNCTION()
    void HandlePlayClicked();

    UFUNCTION()
    void HandleSettingsClicked();

    UFUNCTION()
    void HandleCreditsClicked();

    UFUNCTION()
    void HandleQuitClicked();

    void ShowQuitConfirmation();
    void OnQuitConfirmed(ECommonMessagingResult Result);
};
```

```cpp
// MainMenu.cpp
#include "MainMenu.h"
#include "CommonButtonBase.h"
#include "CommonTextBlock.h"
#include "PrimaryGameLayout.h"
#include "GameplayTagContainer.h"
#include "Messaging/CommonMessagingSubsystem.h"

UMainMenu::UMainMenu(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 设置为仅菜单输入
    InputConfig = ELyraWidgetInputMode::Menu;
    bAutoActivate = true;
}

void UMainMenu::NativeOnInitialized()
{
    Super::NativeOnInitialized();

    // 绑定按钮事件
    Button_Play->OnClicked().AddUObject(this, &ThisClass::HandlePlayClicked);
    Button_Settings->OnClicked().AddUObject(this, &ThisClass::HandleSettingsClicked);
    Button_Credits->OnClicked().AddUObject(this, &ThisClass::HandleCreditsClicked);
    Button_Quit->OnClicked().AddUObject(this, &ThisClass::HandleQuitClicked);
}

void UMainMenu::NativeOnActivated()
{
    Super::NativeOnActivated();

    // 播放入场动画
    if (Intro)
    {
        PlayAnimation(Intro);
    }
}

UWidget* UMainMenu::NativeGetDesiredFocusTarget() const
{
    // 手柄默认焦点到"开始游戏"按钮
    return Button_Play;
}

void UMainMenu::HandlePlayClicked()
{
    UE_LOG(LogTemp, Log, TEXT("Starting game..."));

    // 播放退场动画
    if (Outro)
    {
        PlayAnimation(Outro);
    }

    // 动画完成后开始游戏
    FTimerHandle TimerHandle;
    GetWorld()->GetTimerManager().SetTimer(TimerHandle, [this]()
    {
        // 加载游戏关卡
        UGameplayStatics::OpenLevel(this, TEXT("MainGameMap"));
    }, 0.5f, false);
}

void UMainMenu::HandleSettingsClicked()
{
    // 异步加载并推送设置菜单
    if (!SettingsMenuClass.IsNull())
    {
        if (UPrimaryGameLayout* RootLayout = UPrimaryGameLayout::GetPrimaryGameLayout(GetOwningLocalPlayer()))
        {
            RootLayout->PushWidgetToLayerStackAsync<ULyraActivatableWidget>(
                FGameplayTag::RequestGameplayTag(TEXT("UI.Layer.Menu")),
                true,  // 加载期间暂停输入
                SettingsMenuClass,
                [](EAsyncWidgetLayerState State, ULyraActivatableWidget* Widget)
                {
                    if (State == EAsyncWidgetLayerState::AfterPush)
                    {
                        UE_LOG(LogTemp, Log, TEXT("Settings menu opened"));
                    }
                }
            );
        }
    }
}

void UMainMenu::HandleCreditsClicked()
{
    UE_LOG(LogTemp, Log, TEXT("Opening credits..."));
    // 实现制作人员名单界面
}

void UMainMenu::HandleQuitClicked()
{
    ShowQuitConfirmation();
}

void UMainMenu::ShowQuitConfirmation()
{
    // 使用 CommonMessagingSubsystem 显示确认对话框
    if (UCommonMessagingSubsystem* Messaging = GetOwningLocalPlayer()->GetSubsystem<UCommonMessagingSubsystem>())
    {
        UCommonGameDialogDescriptor* DialogDescriptor = NewObject<UCommonGameDialogDescriptor>();
        DialogDescriptor->Header = NSLOCTEXT("MainMenu", "QuitHeader", "退出游戏");
        DialogDescriptor->Body = NSLOCTEXT("MainMenu", "QuitBody", "确定要退出游戏吗？");

        // 添加确认和取消按钮
        FConfirmationDialogAction ConfirmAction;
        ConfirmAction.Result = ECommonMessagingResult::Confirmed;
        ConfirmAction.OptionalDisplayText = NSLOCTEXT("MainMenu", "Confirm", "确定");
        DialogDescriptor->ButtonActions.Add(ConfirmAction);

        FConfirmationDialogAction CancelAction;
        CancelAction.Result = ECommonMessagingResult::Cancelled;
        CancelAction.OptionalDisplayText = NSLOCTEXT("MainMenu", "Cancel", "取消");
        DialogDescriptor->ButtonActions.Add(CancelAction);

        // 显示对话框
        Messaging->ShowConfirmation(
            DialogDescriptor,
            FCommonMessagingResultDelegate::CreateUObject(this, &ThisClass::OnQuitConfirmed)
        );
    }
}

void UMainMenu::OnQuitConfirmed(ECommonMessagingResult Result)
{
    if (Result == ECommonMessagingResult::Confirmed)
    {
        // 退出游戏
        UKismetSystemLibrary::QuitGame(
            this,
            GetOwningPlayer(),
            EQuitPreference::Quit,
            false
        );
    }
}
```

### 2. 设置菜单 Widget

```cpp
// SettingsMenu.h
#pragma once

#include "UI/LyraActivatableWidget.h"
#include "SettingsMenu.generated.h"

class ULyraTabListWidgetBase;
class UCommonButtonBase;

UCLASS()
class USettingsMenu : public ULyraActivatableWidget
{
    GENERATED_BODY()

public:
    USettingsMenu(const FObjectInitializer& ObjectInitializer);

protected:
    virtual void NativeOnInitialized() override;
    virtual void NativeOnActivated() override;
    virtual void NativeOnDeactivated() override;
    virtual UWidget* NativeGetDesiredFocusTarget() const override;

    // 控件绑定
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<ULyraTabListWidgetBase> TabList_Settings;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Back;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UCommonButtonBase> Button_Apply;

    // 动画
    UPROPERTY(Transient, meta = (BindWidgetAnim))
    TObjectPtr<UWidgetAnimation> FadeIn;

    UPROPERTY(Transient, meta = (BindWidgetAnim))
    TObjectPtr<UWidgetAnimation> FadeOut;

private:
    UFUNCTION()
    void HandleBackClicked();

    UFUNCTION()
    void HandleApplyClicked();

    void ApplySettings();
};
```

```cpp
// SettingsMenu.cpp
#include "SettingsMenu.h"
#include "CommonButtonBase.h"
#include "UI/Common/LyraTabListWidgetBase.h"

USettingsMenu::USettingsMenu(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    InputConfig = ELyraWidgetInputMode::Menu;
    bAutoActivate = true;
}

void USettingsMenu::NativeOnInitialized()
{
    Super::NativeOnInitialized();

    // 绑定按钮
    Button_Back->OnClicked().AddUObject(this, &ThisClass::HandleBackClicked);
    Button_Apply->OnClicked().AddUObject(this, &ThisClass::HandleApplyClicked);

    // 注册 Tab（在蓝图中配置 Tab 内容）
    // TabList_Settings 会自动管理各个设置页签（视频、音频、控制等）
}

void USettingsMenu::NativeOnActivated()
{
    Super::NativeOnActivated();

    if (FadeIn)
    {
        PlayAnimation(FadeIn);
    }
}

void USettingsMenu::NativeOnDeactivated()
{
    Super::NativeOnDeactivated();

    if (FadeOut)
    {
        PlayAnimation(FadeOut);
    }
}

UWidget* USettingsMenu::NativeGetDesiredFocusTarget() const
{
    // 默认焦点到第一个 Tab
    if (TabList_Settings)
    {
        return TabList_Settings->GetActiveTab();
    }

    return Button_Back;
}

void USettingsMenu::HandleBackClicked()
{
    // 返回主菜单（自动恢复主菜单的输入配置）
    DeactivateWidget();
}

void USettingsMenu::HandleApplyClicked()
{
    ApplySettings();

    // 显示保存成功提示
    UE_LOG(LogTemp, Log, TEXT("Settings applied successfully"));
}

void USettingsMenu::ApplySettings()
{
    // 应用各个设置页的更改
    // 这里可以遍历 TabList 中的各个 Tab Widget 并调用它们的 Apply 方法
    UE_LOG(LogTemp, Log, TEXT("Applying settings..."));

    // 示例：保存到配置文件
    if (UGameUserSettings* Settings = UGameUserSettings::GetGameUserSettings())
    {
        Settings->ApplySettings(false);
    }
}
```

### 3. 确认对话框（使用 Lyra 的实现）

Lyra 提供了 `ULyraConfirmationScreen`，我们可以直接使用：

```cpp
// 在任何地方显示确认框
void ShowConfirmDialog(UObject* WorldContextObject, 
                      const FText& Title, 
                      const FText& Message,
                      FCommonMessagingResultDelegate Callback)
{
    if (UCommonLocalPlayer* LocalPlayer = Cast<UCommonLocalPlayer>(
        UGameplayStatics::GetPlayerController(WorldContextObject, 0)->GetLocalPlayer()))
    {
        if (UCommonMessagingSubsystem* Messaging = LocalPlayer->GetSubsystem<UCommonMessagingSubsystem>())
        {
            UCommonGameDialogDescriptor* Descriptor = NewObject<UCommonGameDialogDescriptor>();
            Descriptor->Header = Title;
            Descriptor->Body = Message;

            // 确认按钮
            FConfirmationDialogAction ConfirmAction;
            ConfirmAction.Result = ECommonMessagingResult::Confirmed;
            ConfirmAction.OptionalDisplayText = NSLOCTEXT("Dialog", "OK", "确定");
            Descriptor->ButtonActions.Add(ConfirmAction);

            // 取消按钮
            FConfirmationDialogAction CancelAction;
            CancelAction.Result = ECommonMessagingResult::Cancelled;
            CancelAction.OptionalDisplayText = NSLOCTEXT("Dialog", "Cancel", "取消");
            Descriptor->ButtonActions.Add(CancelAction);

            Messaging->ShowConfirmation(Descriptor, Callback);
        }
    }
}

// 使用示例
ShowConfirmDialog(
    this,
    NSLOCTEXT("Game", "DeleteSave", "删除存档"),
    NSLOCTEXT("Game", "DeleteSaveMsg", "确定要删除此存档吗？此操作无法撤销。"),
    FCommonMessagingResultDelegate::CreateLambda([](ECommonMessagingResult Result)
    {
        if (Result == ECommonMessagingResult::Confirmed)
        {
            // 执行删除
            UE_LOG(LogTemp, Log, TEXT("Save deleted"));
        }
    })
);
```

### 4. 完整的 Layer 管理示例

创建 Primary Game Layout 蓝图（W_PrimaryGameLayout）：

```cpp
// 在蓝图的 Event Graph 中设置
Event Construct
{
    // 注册主菜单 Layer
    RegisterLayer(
        LayerTag: "UI.Layer.Menu",
        LayerWidget: MenuLayer_Stack  // CommonActivatableWidgetStack
    );

    // 注册模态对话框 Layer
    RegisterLayer(
        LayerTag: "UI.Layer.Modal",
        LayerWidget: ModalLayer_Stack  // CommonActivatableWidgetStack
    );

    // 注册游戏 HUD Layer
    RegisterLayer(
        LayerTag: "UI.Layer.Game",
        LayerWidget: GameLayer_Stack  // CommonActivatableWidgetStack
    );
}
```

### 5. 在游戏启动时显示主菜单

```cpp
// MyGameMode.cpp（或 PlayerController）
void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();

    // 获取第一个本地玩家
    if (UGameInstance* GameInstance = GetGameInstance())
    {
        if (ULocalPlayer* LocalPlayer = GameInstance->GetFirstGamePlayer())
        {
            ShowMainMenu(LocalPlayer);
        }
    }
}

void AMyGameMode::ShowMainMenu(ULocalPlayer* LocalPlayer)
{
    if (UPrimaryGameLayout* RootLayout = UPrimaryGameLayout::GetPrimaryGameLayout(LocalPlayer))
    {
        // 异步加载主菜单
        TSoftClassPtr<UMainMenu> MainMenuClass(FSoftObjectPath(TEXT("/Game/UI/W_MainMenu.W_MainMenu_C")));

        RootLayout->PushWidgetToLayerStackAsync<UMainMenu>(
            FGameplayTag::RequestGameplayTag(TEXT("UI.Layer.Menu")),
            true,  // 加载期间暂停输入
            MainMenuClass,
            [](EAsyncWidgetLayerState State, UMainMenu* Widget)
            {
                if (State == EAsyncWidgetLayerState::AfterPush)
                {
                    UE_LOG(LogTemp, Log, TEXT("Main menu displayed"));
                }
            }
        );
    }
}
```

## 与 Lyra UI 系统集成

### UIManager 和 UIPolicy

Lyra 的 UI 管理系统分为三个层级：

```cpp
// 1. UGameUIManagerSubsystem（游戏实例级别）
UCLASS()
class UGameUIManagerSubsystem : public UGameInstanceSubsystem
{
public:
    // 当前 UI Policy
    UFUNCTION(BlueprintCallable)
    UGameUIPolicy* GetCurrentUIPolicy() const;

protected:
    UPROPERTY(Transient)
    TObjectPtr<UGameUIPolicy> CurrentPolicy;
};

// 2. UGameUIPolicy（策略级别，支持多玩家）
UCLASS()
class UGameUIPolicy : public UObject
{
public:
    // 获取玩家的根布局
    UPrimaryGameLayout* GetRootLayout(const UCommonLocalPlayer* LocalPlayer) const;

    // 多人模式
    ELocalMultiplayerInteractionMode GetLocalMultiplayerInteractionMode() const;

protected:
    void CreateLayoutWidget(UCommonLocalPlayer* LocalPlayer);

    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UPrimaryGameLayout> LayoutClass;

    UPROPERTY(Transient)
    TArray<FRootViewportLayoutInfo> RootViewportLayouts;
};

// 3. UPrimaryGameLayout（玩家级别）
// 前面已详细介绍
```

### 在 Lyra Experience 中使用 UI

Lyra 使用 Experience 系统管理游戏模式，可以在 Experience 中配置 UI：

```cpp
// LyraExperienceActionSet 示例
UCLASS()
class ULyraExperienceActionSet : public UPrimaryDataAsset
{
public:
    // 要添加到 HUD 的 Widget 列表
    UPROPERTY(EditAnywhere, Category = "UI")
    TArray<FLyraHUDLayoutRequest> LayoutRequests;
};

// HUD Layout 请求
USTRUCT()
struct FLyraHUDLayoutRequest
{
    GENERATED_BODY()

    // Widget 类
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<ULyraActivatableWidget> LayoutClass;

    // 目标 Layer
    UPROPERTY(EditAnywhere)
    FGameplayTag LayerID;
};
```

**在 Experience 加载时添加 UI：**

```cpp
// LyraExperienceManager.cpp（简化）
void ULyraExperienceManager::OnExperienceLoadComplete()
{
    // 加载 Experience 中定义的 UI
    for (const FLyraHUDLayoutRequest& LayoutRequest : Experience->ActionSets)
    {
        if (UPrimaryGameLayout* RootLayout = UPrimaryGameLayout::GetPrimaryGameLayout(LocalPlayer))
        {
            RootLayout->PushWidgetToLayerStackAsync<ULyraActivatableWidget>(
                LayoutRequest.LayerID,
                false,
                LayoutRequest.LayoutClass
            );
        }
    }
}
```

### 自定义 UI 扩展 Lyra

```cpp
// MyGameUIPolicy.h（扩展 Lyra 的 UI Policy）
#pragma once

#include "GameUIPolicy.h"
#include "MyGameUIPolicy.generated.h"

UCLASS()
class UMyGameUIPolicy : public UGameUIPolicy
{
    GENERATED_BODY()

protected:
    virtual void OnRootLayoutAddedToViewport(
        UCommonLocalPlayer* LocalPlayer, 
        UPrimaryGameLayout* Layout) override;

private:
    void SetupCustomLayers(UPrimaryGameLayout* Layout);
};
```

```cpp
// MyGameUIPolicy.cpp
#include "MyGameUIPolicy.h"
#include "PrimaryGameLayout.h"

void UMyGameUIPolicy::OnRootLayoutAddedToViewport(
    UCommonLocalPlayer* LocalPlayer, 
    UPrimaryGameLayout* Layout)
{
    Super::OnRootLayoutAddedToViewport(LocalPlayer, Layout);

    // 添加自定义 Layer 或 UI 逻辑
    SetupCustomLayers(Layout);
}

void UMyGameUIPolicy::SetupCustomLayers(UPrimaryGameLayout* Layout)
{
    // 注册自定义 Layer
    // 例如：通知系统、任务追踪等
    UE_LOG(LogTemp, Log, TEXT("Setting up custom UI layers"));
}
```

## 性能优化和最佳实践

### 1. Widget 池化

对于频繁创建/销毁的 Widget，使用对象池：

```cpp
// WidgetPool.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "WidgetPool.generated.h"

UCLASS()
class UWidgetPool : public UObject
{
    GENERATED_BODY()

public:
    // 获取或创建 Widget
    template<typename T>
    T* GetOrCreateWidget(TSubclassOf<UUserWidget> WidgetClass, UWorld* World)
    {
        // 查找可用的 Widget
        for (UUserWidget* Widget : InactiveWidgets)
        {
            if (Widget->GetClass() == WidgetClass)
            {
                InactiveWidgets.Remove(Widget);
                ActiveWidgets.Add(Widget);
                return Cast<T>(Widget);
            }
        }

        // 创建新 Widget
        if (T* NewWidget = CreateWidget<T>(World, WidgetClass))
        {
            ActiveWidgets.Add(NewWidget);
            return NewWidget;
        }

        return nullptr;
    }

    // 归还 Widget 到池中
    void ReturnWidget(UUserWidget* Widget)
    {
        if (ActiveWidgets.Contains(Widget))
        {
            ActiveWidgets.Remove(Widget);
            InactiveWidgets.Add(Widget);
            Widget->RemoveFromParent();
        }
    }

    // 清空池
    void ClearPool()
    {
        ActiveWidgets.Empty();
        InactiveWidgets.Empty();
    }

private:
    UPROPERTY()
    TArray<TObjectPtr<UUserWidget>> ActiveWidgets;

    UPROPERTY()
    TArray<TObjectPtr<UUserWidget>> InactiveWidgets;
};
```

**使用示例：**

```cpp
// 创建池
UPROPERTY()
TObjectPtr<UWidgetPool> DamageNumberPool;

void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();
    DamageNumberPool = NewObject<UWidgetPool>(this);
}

// 显示伤害数字
void AMyPlayerController::ShowDamageNumber(float Damage, FVector WorldLocation)
{
    UDamageNumberWidget* Widget = DamageNumberPool->GetOrCreateWidget<UDamageNumberWidget>(
        DamageNumberWidgetClass, 
        GetWorld()
    );

    if (Widget)
    {
        Widget->SetDamage(Damage);
        Widget->SetWorldLocation(WorldLocation);
        Widget->AddToViewport();

        // 2 秒后归还到池
        FTimerHandle TimerHandle;
        GetWorldTimerManager().SetTimer(TimerHandle, [this, Widget]()
        {
            DamageNumberPool->ReturnWidget(Widget);
        }, 2.0f, false);
    }
}
```

### 2. 异步加载 Widget

对于大型 Widget，始终使用异步加载：

```cpp
void LoadAndShowWidget(TSoftClassPtr<UUserWidget> WidgetClass)
{
    // 使用 Streamable Manager
    FStreamableManager& StreamableManager = UAssetManager::Get().GetStreamableManager();

    TSharedPtr<FStreamableHandle> Handle = StreamableManager.RequestAsyncLoad(
        WidgetClass.ToSoftObjectPath(),
        FStreamableDelegate::CreateLambda([this, WidgetClass]()
        {
            if (UClass* LoadedClass = WidgetClass.Get())
            {
                UUserWidget* Widget = CreateWidget<UUserWidget>(GetWorld(), LoadedClass);
                Widget->AddToViewport();
            }
        })
    );
}
```

### 3. 优化 Tick

默认禁用 Widget 的 Tick，仅在必要时启用：

```cpp
UCLASS()
class UMyWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    UMyWidget(const FObjectInitializer& ObjectInitializer)
        : Super(ObjectInitializer)
    {
        // 禁用 Tick
        bCanEverTick = false;
    }

    // 仅在需要时启用 Tick
    void StartUpdating()
    {
        SetTickMode(ETickMode::Enabled);
    }

    void StopUpdating()
    {
        SetTickMode(ETickMode::Disabled);
    }
};
```

### 4. 减少 Invalidation

使用 Invalidation Box 减少不必要的重绘：

```
Widget Hierarchy:
├─ Canvas Panel
   └─ Invalidation Box  <- 只有内部改变时才重绘
      └─ Vertical Box
         ├─ Text_Health (经常更新)
         └─ Text_Ammo (经常更新)
```

### 5. CommonUI 最佳实践清单

✅ **输入管理：**
- 始终为 Activatable Widget 设置正确的 `InputConfig`
- 实现 `BP_GetDesiredFocusTarget()` 支持手柄导航
- 使用 `CommonBoundActionButton` 自动显示输入提示

✅ **Layer 管理：**
- 使用 Gameplay Tag 管理 Layer（便于配置和查询）
- 对频繁切换的 UI 使用 Stack 模式
- 对 HUD 元素使用 Queue 模式

✅ **性能：**
- 大型 Widget 使用异步加载
- 禁用不需要 Tick 的 Widget
- 使用 Invalidation Box 优化重绘
- 对频繁创建的小 Widget 使用对象池

✅ **样式：**
- 使用 Data Asset 定义样式，避免硬编码
- 创建统一的样式管理器
- 为不同平台准备不同的样式变体

✅ **调试：**
- 使用 `Lyra.UI` 日志类别记录 UI 事件
- 启用 CommonUI 的调试可视化（`CommonUI.DebugFocusPath 1`）
- 测试键鼠、手柄、触摸三种输入方式

## 调试技巧

### Console Commands

```cpp
// CommonUI 调试命令
CommonUI.DebugFocusPath 1              // 显示焦点路径
CommonUI.DebugInputMode 1              // 显示当前输入模式
CommonUI.DumpActivatableTree           // 打印 Widget 激活树

// Lyra UI 调试
Lyra.UI.DumpLayoutInfo                 // 打印 Layout 信息
Lyra.UI.ShowLayerBounds 1              // 显示 Layer 边界
```

### 日志记录

```cpp
// 在 Widget 中添加详细日志
void UMyWidget::NativeOnActivated()
{
    Super::NativeOnActivated();

    UE_LOG(LogLyraUI, Log, TEXT("[%s] Activated - Input Mode: %s"),
        *GetName(),
        *UEnum::GetValueAsString(InputConfig));
}

void UMyWidget::NativeOnDeactivated()
{
    Super::NativeOnDeactivated();

    UE_LOG(LogLyraUI, Log, TEXT("[%s] Deactivated"), *GetName());
}
```

## 总结

CommonUI 是 UE5 中构建专业游戏 UI 的强大框架，Lyra 项目展示了如何正确使用这个框架：

### 核心要点

1. **架构层次**：GameUIManagerSubsystem → GameUIPolicy → PrimaryGameLayout → Layer → Widgets
2. **生命周期**：Construct → Initialize → Activate → Deactivate → Destruct
3. **输入管理**：通过 `InputConfig` 自动切换 UI/Game/All 模式
4. **Layer 系统**：使用 Gameplay Tag 管理 UI 层级，Stack/Queue 模式管理 Widget
5. **样式系统**：Data Asset 定义样式，支持主题切换和多平台适配

### 实践建议

- 从简单的菜单开始，逐步添加复杂功能
- 先在蓝图中原型化，再转换为 C++（如果需要）
- 充分测试多平台输入（键鼠、手柄、触摸）
- 使用 Lyra 作为参考，但根据项目需求定制
- 重视性能优化，特别是频繁更新的 HUD 元素

通过掌握 CommonUI 框架和 Lyra 的 UI 系统，您可以构建出专业、高效、跨平台的游戏用户界面。

---

**相关文章：**
- [Lyra UI 系统架构概览](./13-ui-architecture-overview.md)
- [Enhanced Input 系统详解](../02-input-systems/05-enhanced-input-deep-dive.md)
- [Gameplay Tag 系统应用](../01-core-systems/03-gameplay-tags.md)

**参考资源：**
- [UE5 CommonUI 官方文档](https://docs.unrealengine.com/5.3/en-US/common-ui-plugin-for-unreal-engine/)
- [Lyra 源代码](https://github.com/EpicGames/UnrealEngine/tree/release/Samples/Games/Lyra)
