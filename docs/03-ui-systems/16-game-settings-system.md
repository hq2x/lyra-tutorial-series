# 游戏设置系统：保存与同步

## 概述

在现代游戏中，完善的设置系统不仅影响玩家体验，更是跨平台支持、个性化定制和用户留存的关键。UE5 Lyra 的游戏设置系统通过 **GameSettings 插件** 和自定义的持久化方案，构建了一套功能强大、易于扩展且架构优雅的设置管理框架。

本文将深入探讨 Lyra 设置系统的核心组件、设计模式、持久化机制和同步策略，并通过实战案例演示如何在项目中应用这套系统。

## 核心架构

### GameSettings 插件概述

GameSettings 插件位于 `Lyra/Plugins/GameSettings/`，提供了一套完整的设置管理基础设施：

```
GameSettings/
├── Source/
│   ├── Public/
│   │   ├── GameSettingRegistry.h        // 设置注册中心
│   │   ├── GameSetting.h                // 设置基类
│   │   ├── GameSettingValue.h           // 值设置基类
│   │   ├── GameSettingCollection.h      // 设置集合
│   │   ├── GameSettingValueDiscrete.h   // 离散值设置
│   │   ├── GameSettingValueScalar.h     // 标量值设置
│   │   └── ...
│   └── Private/
│       ├── Registry/
│       ├── Widgets/
│       ├── EditCondition/
│       └── DataSource/
```

**插件职责**：
- **设置抽象层**：提供统一的设置定义接口
- **数据绑定**：通过 DataSource 机制连接设置与实际数据
- **UI 生成**：自动生成设置界面的 UMG 组件
- **编辑条件**：控制设置的可见性和可编辑性
- **变更追踪**：监听设置变化并触发相应逻辑

### 核心类关系图

```
UGameSettingRegistry (注册中心)
    │
    ├── UGameSettingCollection (设置集合)
    │   ├── UGameSettingCollection (子集合)
    │   ├── UGameSettingValue (值设置)
    │   └── UGameSettingAction (行为设置)
    │
    ├── UGameSettingValue (可修改的设置)
    │   ├── UGameSettingValueDiscrete (离散值)
    │   │   ├── 窗口模式
    │   │   ├── 分辨率
    │   │   └── 语言选择
    │   └── UGameSettingValueScalar (标量值)
    │       ├── 音量
    │       ├── 鼠标灵敏度
    │       └── 帧率限制
    │
    └── UGameSettingAction (触发行为)
        ├── 编辑安全区域
        └── 恢复默认设置
```

## Setting Registry 设计模式

### 注册中心架构

`UGameSettingRegistry` 是整个设置系统的核心，负责管理所有设置对象的生命周期和事件分发。

```cpp
// GameSettingRegistry.h
UCLASS(MinimalAPI, Abstract, BlueprintType)
class UGameSettingRegistry : public UObject
{
    GENERATED_BODY()

public:
    // 设置变化事件
    DECLARE_EVENT_TwoParams(UGameSettingRegistry, FOnSettingChanged, 
        UGameSetting*, EGameSettingChangeReason);
    FOnSettingChanged OnSettingChangedEvent;

    // 编辑条件变化事件
    DECLARE_EVENT_OneParam(UGameSettingRegistry, FOnSettingEditConditionChanged, UGameSetting*);
    FOnSettingEditConditionChanged OnSettingEditConditionChangedEvent;

    // 导航事件
    DECLARE_EVENT_OneParam(UGameSettingRegistry, FOnExecuteNavigation, UGameSetting*);
    FOnExecuteNavigation OnExecuteNavigationEvent;

    // 初始化
    void Initialize(ULocalPlayer* InLocalPlayer);
    
    // 保存所有变更
    virtual void SaveChanges();
    
    // 查找设置
    UGameSetting* FindSettingByDevName(const FName& SettingDevName);

protected:
    // 子类实现具体的设置注册逻辑
    virtual void OnInitialize(ULocalPlayer* InLocalPlayer) PURE_VIRTUAL(, )

    // 注册设置到系统
    void RegisterSetting(UGameSetting* InSetting);

    // 顶层设置集合（视频、音频、控制等）
    UPROPERTY(Transient)
    TArray<TObjectPtr<UGameSetting>> TopLevelSettings;

    // 所有已注册的设置（扁平列表）
    UPROPERTY(Transient)
    TArray<TObjectPtr<UGameSetting>> RegisteredSettings;

    // 拥有此注册中心的本地玩家
    UPROPERTY(Transient)
    TObjectPtr<ULocalPlayer> OwningLocalPlayer;
};
```

### Lyra 的注册中心实现

Lyra 通过 `ULyraGameSettingRegistry` 具体化了设置注册逻辑：

```cpp
// LyraGameSettingRegistry.h
UCLASS()
class ULyraGameSettingRegistry : public UGameSettingRegistry
{
    GENERATED_BODY()

public:
    // 获取单例（每个 LocalPlayer 一个实例）
    static ULyraGameSettingRegistry* Get(ULyraLocalPlayer* InLocalPlayer);
    
    virtual void SaveChanges() override;

protected:
    virtual void OnInitialize(ULocalPlayer* InLocalPlayer) override;
    virtual bool IsFinishedInitializing() const override;

    // 各分类设置的初始化函数
    UGameSettingCollection* InitializeVideoSettings(ULyraLocalPlayer* InLocalPlayer);
    UGameSettingCollection* InitializeAudioSettings(ULyraLocalPlayer* InLocalPlayer);
    UGameSettingCollection* InitializeGameplaySettings(ULyraLocalPlayer* InLocalPlayer);
    UGameSettingCollection* InitializeMouseAndKeyboardSettings(ULyraLocalPlayer* InLocalPlayer);
    UGameSettingCollection* InitializeGamepadSettings(ULyraLocalPlayer* InLocalPlayer);

    // 设置集合对象
    UPROPERTY()
    TObjectPtr<UGameSettingCollection> VideoSettings;
    
    UPROPERTY()
    TObjectPtr<UGameSettingCollection> AudioSettings;
    
    UPROPERTY()
    TObjectPtr<UGameSettingCollection> GameplaySettings;
    
    UPROPERTY()
    TObjectPtr<UGameSettingCollection> MouseAndKeyboardSettings;
    
    UPROPERTY()
    TObjectPtr<UGameSettingCollection> GamepadSettings;
};
```

**初始化流程**：

```cpp
// LyraGameSettingRegistry.cpp
void ULyraGameSettingRegistry::OnInitialize(ULocalPlayer* InLocalPlayer)
{
    ULyraLocalPlayer* LyraLocalPlayer = Cast<ULyraLocalPlayer>(InLocalPlayer);

    // 按分类初始化设置
    VideoSettings = InitializeVideoSettings(LyraLocalPlayer);
    RegisterSetting(VideoSettings);

    AudioSettings = InitializeAudioSettings(LyraLocalPlayer);
    RegisterSetting(AudioSettings);

    GameplaySettings = InitializeGameplaySettings(LyraLocalPlayer);
    RegisterSetting(GameplaySettings);

    MouseAndKeyboardSettings = InitializeMouseAndKeyboardSettings(LyraLocalPlayer);
    RegisterSetting(MouseAndKeyboardSettings);

    GamepadSettings = InitializeGamepadSettings(LyraLocalPlayer);
    RegisterSetting(GamepadSettings);
}
```

### Setting Value 设计模式

`UGameSettingValue` 是所有可修改设置的基类，提供了值的存储、重置和恢复功能。

```cpp
// GameSettingValue.h
UCLASS(MinimalAPI, Abstract)
class UGameSettingValue : public UGameSetting
{
    GENERATED_BODY()

public:
    // 存储初始值（在应用设置时调用）
    virtual void StoreInitial() PURE_VIRTUAL(, );

    // 重置为默认值
    virtual void ResetToDefault() PURE_VIRTUAL(, );

    // 恢复为初始值（打开设置界面时的值）
    virtual void RestoreToInitial() PURE_VIRTUAL(, );

protected:
    virtual void OnInitialized() override;
};
```

#### 离散值设置（Discrete）

用于有限选项的设置，如窗口模式、分辨率、质量预设等：

```cpp
// GameSettingValueDiscrete.h
UCLASS(Abstract)
class UGameSettingValueDiscrete : public UGameSettingValue
{
    GENERATED_BODY()

public:
    // 获取所有可选项
    virtual TArray<FText> GetOptionDisplayTexts() const PURE_VIRTUAL(, return TArray<FText>(); );
    
    // 获取/设置当前选项索引
    virtual int32 GetDiscreteOptionIndex() const PURE_VIRTUAL(, return 0; );
    virtual void SetDiscreteOptionByIndex(int32 Index) PURE_VIRTUAL(, );

    // 获取/设置默认值索引
    virtual int32 GetDiscreteOptionDefaultIndex() const PURE_VIRTUAL(, return 0; );
    virtual void SetDefaultValueFromIndex(int32 Index) PURE_VIRTUAL(, );
};
```

**实战示例：窗口模式设置**

```cpp
// 在 InitializeVideoSettings 中
UGameSettingValueDiscreteDynamic_Enum* WindowModeSetting = 
    NewObject<UGameSettingValueDiscreteDynamic_Enum>();
WindowModeSetting->SetDevName(TEXT("WindowMode"));
WindowModeSetting->SetDisplayName(LOCTEXT("WindowMode_Name", "Window Mode"));
WindowModeSetting->SetDescriptionRichText(
    LOCTEXT("WindowMode_Description", 
    "In Windowed mode you can interact with other windows more easily, "
    "and drag the edges of the window to set the size. In Windowed Fullscreen mode "
    "the game window covers the whole screen, but you can still switch to other windows "
    "quickly. In Fullscreen mode the game takes over the screen."));

// 设置可选项（枚举类型）
WindowModeSetting->SetEnum(StaticEnum<EWindowMode::Type>());

// 绑定数据源（动态获取/设置实际值）
WindowModeSetting->SetDynamicGetter(
    GET_LOCAL_SETTINGS_FUNCTION_PATH(GetFullscreenMode));
WindowModeSetting->SetDynamicSetter(
    GET_LOCAL_SETTINGS_FUNCTION_PATH(SetFullscreenMode));

// 添加到集合
Screen->AddSetting(WindowModeSetting);
```

#### 标量值设置（Scalar）

用于连续数值设置，如音量、灵敏度、亮度等：

```cpp
// GameSettingValueScalar.h
UCLASS(Abstract)
class UGameSettingValueScalar : public UGameSettingValue
{
    GENERATED_BODY()

public:
    // 获取/设置当前值
    virtual double GetValue() const PURE_VIRTUAL(, return 0.0; );
    virtual void SetValue(double Value, EGameSettingChangeReason Reason) PURE_VIRTUAL(, );

    // 范围定义
    virtual double GetMinimumLimit() const { return MinimumLimit; }
    virtual double GetMaximumLimit() const { return MaximumLimit; }
    virtual double GetSourceRangeAndStep(double& OutMin, double& OutMax) const;

protected:
    UPROPERTY(EditAnywhere, Category=Value)
    double MinimumLimit = 0.0;

    UPROPERTY(EditAnywhere, Category=Value)
    double MaximumLimit = 1.0;

    UPROPERTY(EditAnywhere, Category=Value)
    double SourceStepSize = 0.01;
};
```

**实战示例：音量设置**

```cpp
// 在 InitializeAudioSettings 中
UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
Setting->SetDevName(TEXT("OverallVolume"));
Setting->SetDisplayName(LOCTEXT("OverallVolume_Name", "Overall"));
Setting->SetDescriptionRichText(
    LOCTEXT("OverallVolume_Description", "Adjusts the volume of everything."));

// 动态绑定到 LyraSettingsLocal 的 Getter/Setter
Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetOverallVolume));
Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetOverallVolume));

// 设置默认值和显示格式
Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->GetOverallVolume());
Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::ZeroToOnePercent); // 显示为百分比

// 添加编辑条件：仅主玩家可修改
Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

Volume->AddSetting(Setting);
```

### DataSource 机制

GameSettings 插件使用 **DataSource** 模式实现设置与实际数据的解耦：

```cpp
// DataSource/GameSettingDataSourceDynamic.h
// 通过反射路径动态调用 Getter/Setter 函数
class FGameSettingDataSourceDynamic : public IGameSettingDataSource
{
public:
    FGameSettingDataSourceDynamic(const TArray<FString>& InFunctionOrPropertyPath)
        : FunctionOrPropertyPath(InFunctionOrPropertyPath)
    {}

    // 通过反射链式调用函数获取值
    virtual void GetValue(UObject* InObject, UProperty* InProperty, void* OutValue) const override;
    
    // 通过反射链式调用函数设置值
    virtual void SetValue(UObject* InObject, UProperty* InProperty, const void* InValue) override;

private:
    TArray<FString> FunctionOrPropertyPath;
};
```

**宏辅助定义路径**：

```cpp
// LyraGameSettingRegistry.h

// 共享设置路径（SaveGame 持久化）
#define GET_SHARED_SETTINGS_FUNCTION_PATH(FunctionOrPropertyName) \
    MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({ \
        GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings), \
        GET_FUNCTION_NAME_STRING_CHECKED(ULyraSettingsShared, FunctionOrPropertyName) \
    }))

// 本地设置路径（GameUserSettings 持久化）
#define GET_LOCAL_SETTINGS_FUNCTION_PATH(FunctionOrPropertyName) \
    MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({ \
        GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetLocalSettings), \
        GET_FUNCTION_NAME_STRING_CHECKED(ULyraSettingsLocal, FunctionOrPropertyName) \
    }))
```

**使用示例**：

```cpp
// 设置鼠标灵敏度（绑定到 SharedSettings）
Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetMouseSensitivityX));
Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetMouseSensitivityX));

// 等价于调用链：
// Get: LocalPlayer->GetSharedSettings()->GetMouseSensitivityX()
// Set: LocalPlayer->GetSharedSettings()->SetMouseSensitivityX(Value)
```

## LyraSettingsLocal 源码分析

### 架构定位

`ULyraSettingsLocal` 继承自 `UGameUserSettings`，负责管理 **机器特定** 的设置，这些设置不适合在不同设备间共享：

- **图形设置**：分辨率、窗口模式、垂直同步、抗锯齿等
- **性能统计**：FPS 显示、延迟标记等
- **帧率限制**：各种场景下的帧率上限
- **音频设备**：输出设备选择
- **安全区域**：屏幕边缘调整

```cpp
// LyraSettingsLocal.h
UCLASS()
class ULyraSettingsLocal : public UGameUserSettings
{
    GENERATED_BODY()

public:
    // 获取单例
    static ULyraSettingsLocal* Get();

    //~UGameUserSettings interface
    virtual void SetToDefaults() override;
    virtual void LoadSettings(bool bForceReload) override;
    virtual void ApplyNonResolutionSettings() override;
    virtual int32 GetOverallScalabilityLevel() const override;
    virtual void SetOverallScalabilityLevel(int32 Value) override;
    //~End of UGameUserSettings interface

    // 应用纯质量设置（不包括分辨率）
    void ApplyScalabilitySettings();

private:
    // 亮度/Gamma
    UPROPERTY(Config)
    float DisplayGamma = 2.2;

    // 帧率限制（不同场景）
    UPROPERTY(Config)
    float FrameRateLimit_OnBattery;        // 电池供电时
    
    UPROPERTY(Config)
    float FrameRateLimit_InMenu;           // 菜单中
    
    UPROPERTY(Config)
    float FrameRateLimit_WhenBackgrounded; // 后台运行时

    // 性能统计显示
    UPROPERTY(Config)
    TMap<ELyraDisplayablePerformanceStat, ELyraStatDisplayMode> DisplayStatList;

    // 延迟标记（用于测量输入延迟）
    UPROPERTY(Config)
    bool bEnableLatencyFlashIndicators = false;

    UPROPERTY(Config)
    bool bEnableLatencyTrackingStats;

    // 音频
    UPROPERTY(Config)
    float OverallVolume = 1.0f;
    
    UPROPERTY(Config)
    float MusicVolume = 1.0f;
    
    UPROPERTY(Config)
    float SoundFXVolume = 1.0f;
    
    UPROPERTY(Config)
    float DialogueVolume = 1.0f;
    
    UPROPERTY(Config)
    float VoiceChatVolume = 1.0f;

    // 音频输出设备
    UPROPERTY(Config)
    FString AudioOutputDeviceId;

    // HDR Audio
    UPROPERTY(Config)
    bool bUseHDRAudioMode;

    // 安全区域
    UPROPERTY(Config)
    float SafeZoneScale = -1;

    // 控制器平台
    UPROPERTY(Config)
    FName ControllerPlatform;

    // 录像回放
    UPROPERTY(Config)
    bool bShouldAutoRecordReplays = false;
    
    UPROPERTY(Config)
    int32 NumberOfReplaysToKeep = 5;
};
```

### 关键功能实现

#### 1. 帧率限制管理

Lyra 针对不同场景实现了细粒度的帧率控制：

```cpp
// LyraSettingsLocal.cpp

float ULyraSettingsLocal::GetEffectiveFrameRateLimit()
{
    // 根据当前场景选择合适的帧率限制

    // 如果在 PIE 中且不应用帧率设置，返回 0（无限制）
#if WITH_EDITOR
    if (GIsEditor && !CVarApplyFrameRateSettingsInPIE.GetValueOnGameThread())
    {
        return 0.0f;
    }
#endif

    // 如果应用未激活（后台运行）
    if (!FPlatformApplicationMisc::IsThisApplicationForeground())
    {
        return FrameRateLimit_WhenBackgrounded;
    }

    // 如果正在菜单中
    if (ShouldUseFrontendPerformanceSettings())
    {
        return FrameRateLimit_InMenu;
    }

    // 如果使用电池供电
    if (FPlatformMisc::IsRunningOnBattery())
    {
        return FrameRateLimit_OnBattery;
    }

    // 默认帧率限制
    return FrameRateLimit_Always;
}

void ULyraSettingsLocal::UpdateEffectiveFrameRateLimit()
{
    const float EffectiveLimit = GetEffectiveFrameRateLimit();
    
    if (EffectiveLimit <= 0.0f)
    {
        // 无限制
        GEngine->SetMaxFPS(0);
    }
    else
    {
        GEngine->SetMaxFPS(EffectiveLimit);
    }
}
```

#### 2. 音量控制（Audio Control Bus）

Lyra 使用 UE5 的 **Audio Modulation** 系统实现音量控制：

```cpp
void ULyraSettingsLocal::SetOverallVolume(float InVolume)
{
    OverallVolume = InVolume;
    SetVolumeForSoundClass(TEXT("Overall"), InVolume);
}

void ULyraSettingsLocal::SetVolumeForSoundClass(FName ChannelName, float InVolume)
{
    // 查找对应的 Sound Control Bus
    if (USoundControlBus* SoundControlBus = ControlBusMap.FindRef(ChannelName))
    {
        SetVolumeForControlBus(SoundControlBus, InVolume);
    }
}

void ULyraSettingsLocal::SetVolumeForControlBus(USoundControlBus* InSoundControlBus, 
                                                  float InVolume)
{
    if (!InSoundControlBus || !ControlBusMix)
    {
        return;
    }

    // 创建 Bus Modulator
    FSoundModulationDefaultSettings ModulationSettings;
    ModulationSettings.VolumeModulationDestination.Value = InVolume;

    // 应用到 Mix
    UAudioModulationStatics::UpdateModulator(
        OwningPlayer->GetWorld(),
        ControlBusMix,
        InSoundControlBus,
        ModulationSettings
    );
}
```

#### 3. 移动端性能优化

Lyra 针对移动平台实现了自适应质量调整：

```cpp
void ULyraSettingsLocal::SetDesiredMobileFrameRateLimit(int32 NewLimitFPS)
{
    const int32 OldLimitFPS = DesiredMobileFrameRateLimit;
    DesiredMobileFrameRateLimit = NewLimitFPS;

    // 根据帧率限制调整质量设置
    ClampMobileQuality();

    // 更新设备配置
    UpdateMobileFramePacing();
}

void ULyraSettingsLocal::ClampMobileQuality()
{
    // 获取当前质量等级
    const Scalability::FQualityLevels CurrentLevels = Scalability::GetQualityLevels();
    Scalability::FQualityLevels ClampedLevels = CurrentLevels;

    // 根据目标帧率限制分辨率质量
    ClampMobileResolutionQuality(DesiredMobileFrameRateLimit);

    // 应用设备配置的质量限制
    const Scalability::FQualityLevels DeviceDefaultLevels = 
        DeviceDefaultScalabilitySettings.Qualities;
    ClampQualityLevelsToDeviceProfile(DeviceDefaultLevels, ClampedLevels);

    // 应用新的质量等级
    Scalability::SetQualityLevels(ClampedLevels);
}
```

## LyraSettingsShared 源码分析

### 架构定位

`ULyraSettingsShared` 继承自 `ULocalPlayerSaveGame`（UE5 的 SaveGame 系统），管理 **用户个性化** 的设置，这些设置适合跨设备同步：

- **无障碍选项**：色盲模式、字幕样式等
- **手柄设置**：震动、死区、灵敏度等
- **输入设置**：鼠标灵敏度、轴反转等
- **游戏偏好**：语言、背景音频等

```cpp
// LyraSettingsShared.h
UCLASS()
class ULyraSettingsShared : public ULocalPlayerSaveGame
{
    GENERATED_BODY()

public:
    DECLARE_EVENT_OneParam(ULyraSettingsShared, FOnSettingChangedEvent, 
                           ULyraSettingsShared* Settings);
    FOnSettingChangedEvent OnSettingChanged;

    // 创建临时设置对象（用于首次启动）
    static ULyraSettingsShared* CreateTemporarySettings(const ULyraLocalPlayer* LocalPlayer);
    
    // 同步加载设置
    static ULyraSettingsShared* LoadOrCreateSettings(const ULyraLocalPlayer* LocalPlayer);

    // 异步加载设置
    DECLARE_DELEGATE_OneParam(FOnSettingsLoadedEvent, ULyraSettingsShared* Settings);
    static bool AsyncLoadOrCreateSettings(const ULyraLocalPlayer* LocalPlayer, 
                                          FOnSettingsLoadedEvent Delegate);

    // 保存设置到磁盘
    void SaveSettings();

    // 应用设置到游戏
    void ApplySettings();

    // 脏标记（用于判断是否需要保存）
    bool IsDirty() const { return bIsDirty; }
    void ClearDirtyFlag() { bIsDirty = false; }

private:
    // 色盲模式
    UPROPERTY()
    EColorBlindMode ColorBlindMode = EColorBlindMode::Off;

    UPROPERTY()
    int32 ColorBlindStrength = 10;

    // 手柄震动
    UPROPERTY()
    bool bForceFeedbackEnabled = true;

    // 手柄死区
    UPROPERTY()
    float GamepadMoveStickDeadZone;

    UPROPERTY()
    float GamepadLookStickDeadZone;

    // 扳机触觉反馈（PS5 DualSense）
    UPROPERTY()
    bool bTriggerHapticsEnabled = false;

    UPROPERTY()
    bool bTriggerPullUsesHapticThreshold = true;

    UPROPERTY()
    uint8 TriggerHapticStrength = 8;

    UPROPERTY()
    uint8 TriggerHapticStartPosition = 0;

    // 字幕
    UPROPERTY()
    bool bEnableSubtitles = true;

    UPROPERTY()
    ESubtitleDisplayTextSize SubtitleTextSize = ESubtitleDisplayTextSize::Medium;

    UPROPERTY()
    ESubtitleDisplayTextColor SubtitleTextColor = ESubtitleDisplayTextColor::White;

    UPROPERTY()
    ESubtitleDisplayTextBorder SubtitleTextBorder = ESubtitleDisplayTextBorder::None;

    UPROPERTY()
    ESubtitleDisplayBackgroundOpacity SubtitleBackgroundOpacity = 
        ESubtitleDisplayBackgroundOpacity::Medium;

    // 背景音频
    UPROPERTY()
    ELyraAllowBackgroundAudioSetting AllowAudioInBackground = 
        ELyraAllowBackgroundAudioSetting::Off;

    // 语言
    UPROPERTY(Transient)
    FString PendingCulture;

    bool bResetToDefaultCulture = false;

    // 鼠标灵敏度
    UPROPERTY()
    double MouseSensitivityX = 1.0;

    UPROPERTY()
    double MouseSensitivityY = 1.0;

    UPROPERTY()
    double TargetingMultiplier = 0.5; // 瞄准时的灵敏度倍数

    UPROPERTY()
    bool bInvertVerticalAxis = false;

    UPROPERTY()
    bool bInvertHorizontalAxis = false;

    // 手柄灵敏度预设
    UPROPERTY()
    ELyraGamepadSensitivity GamepadLookSensitivityPreset = ELyraGamepadSensitivity::Normal;

    UPROPERTY()
    ELyraGamepadSensitivity GamepadTargetingSensitivityPreset = 
        ELyraGamepadSensitivity::Normal;

    bool bIsDirty = false;
};
```

### 关键功能实现

#### 1. 持久化机制

Lyra 使用 UE5 的 `ULocalPlayerSaveGame` 系统：

```cpp
// LyraSettingsShared.cpp

static FString SHARED_SETTINGS_SLOT_NAME = TEXT("SharedGameSettings");

ULyraSettingsShared* ULyraSettingsShared::LoadOrCreateSettings(
    const ULyraLocalPlayer* LocalPlayer)
{
    // 同步加载（会阻塞主线程）
    ULyraSettingsShared* SharedSettings = Cast<ULyraSettingsShared>(
        LoadOrCreateSaveGameForLocalPlayer(
            ULyraSettingsShared::StaticClass(),
            LocalPlayer,
            SHARED_SETTINGS_SLOT_NAME
        )
    );

    // 加载后立即应用设置
    SharedSettings->ApplySettings();

    return SharedSettings;
}

bool ULyraSettingsShared::AsyncLoadOrCreateSettings(
    const ULyraLocalPlayer* LocalPlayer, 
    FOnSettingsLoadedEvent Delegate)
{
    // 异步加载（推荐方式）
    FOnLocalPlayerSaveGameLoadedNative Lambda = 
        FOnLocalPlayerSaveGameLoadedNative::CreateLambda(
            [Delegate](ULocalPlayerSaveGame* LoadedSave)
            {
                ULyraSettingsShared* LoadedSettings = 
                    CastChecked<ULyraSettingsShared>(LoadedSave);
                
                LoadedSettings->ApplySettings();
                Delegate.ExecuteIfBound(LoadedSettings);
            }
        );

    return ULocalPlayerSaveGame::AsyncLoadOrCreateSaveGameForLocalPlayer(
        ULyraSettingsShared::StaticClass(),
        LocalPlayer,
        SHARED_SETTINGS_SLOT_NAME,
        Lambda
    );
}

void ULyraSettingsShared::SaveSettings()
{
    // 异步保存（不会阻塞主线程）
    AsyncSaveGameToSlotForLocalPlayer();

    // 同时保存 Enhanced Input 的键位绑定
    if (UEnhancedInputLocalPlayerSubsystem* System = 
        ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(OwningPlayer))
    {
        if (UEnhancedInputUserSettings* InputSettings = System->GetUserSettings())
        {
            InputSettings->AsyncSaveSettings();
        }
    }
}
```

#### 2. 脏标记与变更追踪

Lyra 实现了细粒度的变更追踪：

```cpp
// LyraSettingsShared.h

template<typename T>
bool ChangeValueAndDirty(T& CurrentValue, const T& NewValue)
{
    if (CurrentValue != NewValue)
    {
        CurrentValue = NewValue;
        bIsDirty = true;
        
        // 广播变更事件
        OnSettingChanged.Broadcast(this);
        
        return true;
    }

    return false;
}

// 使用示例
void ULyraSettingsShared::SetMouseSensitivityX(double NewValue)
{
    if (ChangeValueAndDirty(MouseSensitivityX, NewValue))
    {
        // 值确实发生了变化
        ApplyInputSensitivity();
    }
}
```

#### 3. 色盲模式实现

Lyra 利用 UE5 的 Slate 渲染器实现色盲辅助功能：

```cpp
void ULyraSettingsShared::SetColorBlindMode(EColorBlindMode InMode)
{
    if (ColorBlindMode != InMode)
    {
        ColorBlindMode = InMode;
        
        // 应用到 Slate 渲染器
        FSlateApplication::Get().GetRenderer()->SetColorVisionDeficiencyType(
            (EColorVisionDeficiency)(int32)ColorBlindMode,
            (int32)ColorBlindStrength,
            true,  // bCorrectDeficiency
            false  // bShowCorrectionWithDeficiency
        );
    }
}

void ULyraSettingsShared::SetColorBlindStrength(int32 InColorBlindStrength)
{
    InColorBlindStrength = FMath::Clamp(InColorBlindStrength, 0, 10);
    if (ColorBlindStrength != InColorBlindStrength)
    {
        ColorBlindStrength = InColorBlindStrength;
        FSlateApplication::Get().GetRenderer()->SetColorVisionDeficiencyType(
            (EColorVisionDeficiency)(int32)ColorBlindMode,
            (int32)ColorBlindStrength,
            true,
            false
        );
    }
}
```

#### 4. 字幕系统集成

```cpp
void ULyraSettingsShared::ApplySubtitleOptions()
{
    if (USubtitleDisplaySubsystem* SubtitleSystem = 
        USubtitleDisplaySubsystem::Get(OwningPlayer))
    {
        FSubtitleFormat SubtitleFormat;
        SubtitleFormat.SubtitleTextSize = SubtitleTextSize;
        SubtitleFormat.SubtitleTextColor = SubtitleTextColor;
        SubtitleFormat.SubtitleTextBorder = SubtitleTextBorder;
        SubtitleFormat.SubtitleBackgroundOpacity = SubtitleBackgroundOpacity;

        SubtitleSystem->SetSubtitleDisplayOptions(SubtitleFormat);
    }
}
```

## 设置持久化机制

### Local Settings（GameUserSettings）

`ULyraSettingsLocal` 的数据存储在 **GameUserSettings.ini** 文件中：

```ini
[/Script/LyraGame.LyraSettingsLocal]
ResolutionSizeX=1920
ResolutionSizeY=1080
FullscreenMode=1
FrameRateLimit_OnBattery=30.0
FrameRateLimit_InMenu=60.0
FrameRateLimit_WhenBackgrounded=30.0
DisplayGamma=2.2
OverallVolume=1.0
MusicVolume=0.8
SoundFXVolume=0.9
DialogueVolume=1.0
VoiceChatVolume=1.0
AudioOutputDeviceId=""
SafeZoneScale=0.0
ControllerPlatform="XboxOne"
bShouldAutoRecordReplays=False
NumberOfReplaysToKeep=5
```

**存储位置**：
- **Windows**: `%LOCALAPPDATA%/LyraGame/Saved/Config/Windows/GameUserSettings.ini`
- **Mac**: `~/Library/Application Support/LyraGame/Saved/Config/Mac/GameUserSettings.ini`
- **Linux**: `~/.config/LyraGame/Saved/Config/Linux/GameUserSettings.ini`

### Shared Settings（SaveGame）

`ULyraSettingsShared` 使用 **SaveGame** 系统，存储为二进制文件：

**存储位置**：
- **Windows**: `%LOCALAPPDATA%/LyraGame/Saved/SaveGames/SharedGameSettings.sav`
- **支持云同步**：可通过 Steam Cloud、Epic Online Services 等平台进行同步

**序列化过程**：

```cpp
// SaveGame 系统自动序列化所有 UPROPERTY 标记的成员
// Lyra 还支持版本管理：

int32 ULyraSettingsShared::GetLatestDataVersion() const
{
    // 版本号用于升级旧存档
    // 0 = 子类化 ULocalPlayerSaveGame 之前
    // 1 = 第一个正式版本
    return 1;
}
```

### 保存触发时机

Lyra 在以下场景触发设置保存：

1. **用户主动应用设置**：

```cpp
void ULyraGameSettingRegistry::SaveChanges()
{
    Super::SaveChanges();
    
    if (ULyraLocalPlayer* LocalPlayer = Cast<ULyraLocalPlayer>(OwningLocalPlayer))
    {
        // Local Settings 需要先 Apply 再保存（间接保存）
        LocalPlayer->GetLocalSettings()->ApplySettings(false);
        
        // Shared Settings 直接保存
        LocalPlayer->GetSharedSettings()->ApplySettings();
        LocalPlayer->GetSharedSettings()->SaveSettings();
    }
}
```

2. **退出游戏时**：

```cpp
void ULyraLocalPlayer::BeginDestroy()
{
    // 确保设置被保存
    if (SharedSettings && SharedSettings->IsDirty())
    {
        SharedSettings->SaveSettings();
    }

    Super::BeginDestroy();
}
```

3. **关键设置变更时立即保存**：

```cpp
void ULyraSettingsShared::SetPendingCulture(const FString& NewCulture)
{
    PendingCulture = NewCulture;
    bResetToDefaultCulture = false;
    bIsDirty = true;
    
    // 语言变更立即保存，避免丢失
    SaveSettings();
}
```

## 设置同步策略

### 本地同步

Lyra 通过事件系统实现设置的实时同步：

```cpp
// 在 GameSettingRegistry 中监听设置变化
void UGameSettingRegistry::HandleSettingChanged(UGameSetting* Setting, 
                                                 EGameSettingChangeReason Reason)
{
    // 立即应用设置（如果需要）
    if (Reason == EGameSettingChangeReason::Change)
    {
        Setting->NotifyChanged(Reason);
    }

    // 广播到外部监听者
    OnSettingChangedEvent.Broadcast(Setting, Reason);
}
```

**UI 自动更新**：

```cpp
// GameSettingPanel.cpp
void UGameSettingPanel::HandleSettingChanged(UGameSetting* Setting, 
                                              EGameSettingChangeReason Reason)
{
    // 自动刷新 UI 显示
    if (Reason != EGameSettingChangeReason::RestoreToInitial)
    {
        RebuildWidget();
    }
}
```

### 服务器同步

对于多人游戏，部分设置需要同步到服务器：

```cpp
// 示例：同步语音聊天音量到服务器
UCLASS()
class ALyraPlayerController : public AModularPlayerController
{
    GENERATED_BODY()

public:
    // 客户端调用
    UFUNCTION(Server, Reliable)
    void ServerSetVoiceChatVolume(float NewVolume);

protected:
    void ServerSetVoiceChatVolume_Implementation(float NewVolume)
    {
        // 服务器验证并应用
        if (NewVolume >= 0.0f && NewVolume <= 1.0f)
        {
            // 更新玩家状态
            PlayerState->SetVoiceChatVolume(NewVolume);
        }
    }
};
```

### 跨平台同步

Lyra 通过 SaveGame 系统支持跨平台同步：

**Epic Online Services (EOS) 集成**：

```cpp
// 使用 EOS PlayerDataStorage 接口
void ULyraSettingsShared::SaveSettings()
{
    AsyncSaveGameToSlotForLocalPlayer();

    // 如果启用了 EOS，上传到云端
    if (IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get())
    {
        if (IOnlineUserCloudPtr UserCloud = OnlineSubsystem->GetUserCloudInterface())
        {
            // 读取本地文件
            TArray<uint8> FileData;
            if (ReadSaveGameFile(SHARED_SETTINGS_SLOT_NAME, FileData))
            {
                // 上传到云端
                UserCloud->WriteUserFile(
                    *OwningPlayer->GetPreferredUniqueNetId(),
                    SHARED_SETTINGS_SLOT_NAME,
                    FileData
                );
            }
        }
    }
}
```

**冲突解决策略**：

```cpp
// 云端数据与本地数据冲突时
enum class ESettingsSyncConflictResolution
{
    PreferLocal,     // 使用本地设置
    PreferRemote,    // 使用云端设置
    PreferNewer,     // 使用最新时间戳的设置
    Merge            // 尝试合并（需要自定义逻辑）
};

void ULyraSettingsShared::ResolveConflict(ULyraSettingsShared* RemoteSettings,
                                           ESettingsSyncConflictResolution Strategy)
{
    switch (Strategy)
    {
    case ESettingsSyncConflictResolution::PreferNewer:
        // 比较时间戳
        if (RemoteSettings->LastModifiedTimestamp > LastModifiedTimestamp)
        {
            // 使用远程设置
            CopyFrom(RemoteSettings);
        }
        break;

    case ESettingsSyncConflictResolution::Merge:
        // 合并设置（保留各自的最新值）
        MergeFrom(RemoteSettings);
        break;

    // ...
    }
}
```

## 音频设置实现

### 音量控制架构

Lyra 使用 UE5 的 **Audio Modulation** 系统实现细粒度音量控制：

```cpp
// LyraGameSettingRegistry_Audio.cpp

UGameSettingCollection* ULyraGameSettingRegistry::InitializeAudioSettings(
    ULyraLocalPlayer* InLocalPlayer)
{
    UGameSettingCollection* Screen = NewObject<UGameSettingCollection>();
    Screen->SetDevName(TEXT("AudioCollection"));
    Screen->SetDisplayName(LOCTEXT("AudioCollection_Name", "Audio"));

    // 音量集合
    UGameSettingCollection* Volume = NewObject<UGameSettingCollection>();
    Volume->SetDevName(TEXT("VolumeCollection"));
    Volume->SetDisplayName(LOCTEXT("VolumeCollection_Name", "Volume"));
    Screen->AddSetting(Volume);

    // 总音量
    {
        UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
        Setting->SetDevName(TEXT("OverallVolume"));
        Setting->SetDisplayName(LOCTEXT("OverallVolume_Name", "Overall"));
        Setting->SetDescriptionRichText(
            LOCTEXT("OverallVolume_Description", "Adjusts the volume of everything."));

        Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetOverallVolume));
        Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetOverallVolume));
        Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->GetOverallVolume());
        Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::ZeroToOnePercent);

        // 仅主玩家可修改
        Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

        Volume->AddSetting(Setting);
    }

    // 音乐、音效、对话、语音聊天音量类似...

    return Screen;
}
```

### Sound Control Bus 系统

```cpp
// LyraSettingsLocal.cpp

void ULyraSettingsLocal::LoadUserControlBusMix()
{
    if (bSoundControlBusMixLoaded)
    {
        return;
    }

    // 加载 Control Bus Mix 资产
    const FSoftObjectPath BusMixPath(TEXT("/Game/Audio/LyraControlBusMix.LyraControlBusMix"));
    if (UObject* BusMixObject = BusMixPath.TryLoad())
    {
        ControlBusMix = CastChecked<USoundControlBusMix>(BusMixObject);
    }

    // 加载各音频通道的 Control Bus
    const TArray<TPair<FName, FSoftObjectPath>> BusPaths = {
        { TEXT("Overall"), FSoftObjectPath(TEXT("/Game/Audio/Busses/Overall.Overall")) },
        { TEXT("Music"), FSoftObjectPath(TEXT("/Game/Audio/Busses/Music.Music")) },
        { TEXT("SoundFX"), FSoftObjectPath(TEXT("/Game/Audio/Busses/SoundFX.SoundFX")) },
        { TEXT("Dialogue"), FSoftObjectPath(TEXT("/Game/Audio/Busses/Dialogue.Dialogue")) },
        { TEXT("VoiceChat"), FSoftObjectPath(TEXT("/Game/Audio/Busses/VoiceChat.VoiceChat")) }
    };

    for (const auto& BusPath : BusPaths)
    {
        if (UObject* BusObject = BusPath.Value.TryLoad())
        {
            USoundControlBus* ControlBus = CastChecked<USoundControlBus>(BusObject);
            ControlBusMap.Add(BusPath.Key, ControlBus);
        }
    }

    bSoundControlBusMixLoaded = true;
}
```

### 音频设备切换

```cpp
void ULyraSettingsLocal::SetAudioOutputDeviceId(const FString& InAudioOutputDeviceId)
{
    if (AudioOutputDeviceId != InAudioOutputDeviceId)
    {
        AudioOutputDeviceId = InAudioOutputDeviceId;

        // 切换音频输出设备
        if (FAudioDevice* AudioDevice = GetWorld()->GetAudioDeviceRaw())
        {
            AudioDevice->SetAudioOutputDevice(AudioOutputDeviceId);
        }

        // 广播设备变更事件
        OnAudioOutputDeviceChanged.Broadcast(AudioOutputDeviceId);
    }
}
```

**动态获取可用设备**：

```cpp
// CustomSettings/LyraSettingValueDiscreteDynamic_AudioOutputDevice.cpp

void ULyraSettingValueDiscreteDynamic_AudioOutputDevice::OnInitialized()
{
    Super::OnInitialized();

    // 查询系统音频设备
    if (FAudioDeviceHandle AudioDevice = GEngine->GetMainAudioDeviceHandle())
    {
        TArray<FAudioOutputDeviceInfo> DeviceInfos;
        AudioDevice->GetOutputDevices(DeviceInfos);

        OptionValues.Reset();
        OptionDisplayTexts.Reset();

        // 添加默认设备
        OptionValues.Add(TEXT(""));
        OptionDisplayTexts.Add(LOCTEXT("DefaultAudioDevice", "System Default"));

        // 添加检测到的设备
        for (const FAudioOutputDeviceInfo& DeviceInfo : DeviceInfos)
        {
            OptionValues.Add(DeviceInfo.DeviceId);
            OptionDisplayTexts.Add(FText::FromString(DeviceInfo.Name));
        }
    }
}
```

## 视频设置实现

### 分辨率与窗口模式

```cpp
// CustomSettings/LyraSettingValueDiscrete_Resolution.cpp

void ULyraSettingValueDiscrete_Resolution::OnInitialized()
{
    Super::OnInitialized();

    // 获取支持的分辨率列表
    TArray<FIntPoint> Resolutions;
    if (UGameUserSettings* UserSettings = GetLocalPlayerChecked()->GetLocalSettings())
    {
        FScreenResolutionArray ResArray;
        if (RHIGetAvailableResolutions(ResArray, true))
        {
            for (const FScreenResolutionRHI& Resolution : ResArray)
            {
                Resolutions.AddUnique(FIntPoint(Resolution.Width, Resolution.Height));
            }
        }
    }

    // 排序（从小到大）
    Resolutions.Sort([](const FIntPoint& A, const FIntPoint& B) {
        return (A.X * A.Y) < (B.X * B.Y);
    });

    // 生成选项
    OptionValues.Reset();
    OptionDisplayTexts.Reset();

    for (const FIntPoint& Resolution : Resolutions)
    {
        OptionValues.Add(Resolution);
        OptionDisplayTexts.Add(FText::Format(
            LOCTEXT("ResolutionFormat", "{0} x {1}"),
            Resolution.X,
            Resolution.Y
        ));
    }
}

void ULyraSettingValueDiscrete_Resolution::SetDiscreteOptionByIndex(int32 Index)
{
    if (OptionValues.IsValidIndex(Index))
    {
        const FIntPoint& Resolution = OptionValues[Index];
        
        if (UGameUserSettings* UserSettings = GetLocalPlayerChecked()->GetLocalSettings())
        {
            UserSettings->SetScreenResolution(Resolution);
            UserSettings->ApplyResolutionSettings(false); // 不立即全屏切换
            NotifySettingChanged(EGameSettingChangeReason::Change);
        }
    }
}
```

### 图形质量预设

```cpp
// CustomSettings/LyraSettingValueDiscrete_OverallQuality.cpp

void ULyraSettingValueDiscrete_OverallQuality::SetDiscreteOptionByIndex(int32 Index)
{
    if (ULyraSettingsLocal* UserSettings = GetLocalPlayerChecked()->GetLocalSettings())
    {
        // 0 = Low, 1 = Medium, 2 = High, 3 = Epic
        UserSettings->SetOverallScalabilityLevel(Index);
        
        // 立即应用
        UserSettings->ApplyScalabilitySettings();
        
        NotifySettingChanged(EGameSettingChangeReason::Change);
    }
}

int32 ULyraSettingValueDiscrete_OverallQuality::GetDiscreteOptionIndex() const
{
    if (const ULyraSettingsLocal* UserSettings = GetLocalPlayerChecked()->GetLocalSettings())
    {
        return UserSettings->GetOverallScalabilityLevel();
    }
    return 0;
}
```

### 动态分辨率

```cpp
// LyraGameSettingRegistry_Video.cpp

void ULyraGameSettingRegistry::InitializeVideoSettings_FrameRates(
    UGameSettingCollection* Screen, 
    ULyraLocalPlayer* InLocalPlayer)
{
    // 动态分辨率目标帧率
    {
        UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
        Setting->SetDevName(TEXT("DynamicResolutionFrameRate"));
        Setting->SetDisplayName(
            LOCTEXT("DynamicResFrameRate_Name", "Dynamic Resolution Target FPS"));
        Setting->SetDescriptionRichText(
            LOCTEXT("DynamicResFrameRate_Description", 
            "When dynamic resolution is enabled, the game will adjust the "
            "resolution to try to maintain this target frame rate."));

        Setting->SetDynamicGetter(
            GET_LOCAL_SETTINGS_FUNCTION_PATH(GetDynamicResolutionFrameRateTarget));
        Setting->SetDynamicSetter(
            GET_LOCAL_SETTINGS_FUNCTION_PATH(SetDynamicResolutionFrameRateTarget));
        
        Setting->SetDefaultValue(60.0f);
        Setting->SetDisplayFormat([](double SourceValue, double NormalizedValue) {
            return FText::Format(LOCTEXT("FPSFormat", "{0} FPS"), 
                                 FText::AsNumber((int32)SourceValue));
        });

        // 仅在支持动态分辨率的平台显示
        Setting->AddEditCondition(
            MakeShared<FGameSettingEditCondition_DynamicResolution>(
                ELyraFramePacingMode::DesktopStyle));

        Screen->AddSetting(Setting);
    }
}
```

## 控制设置实现

### 鼠标与键盘设置

```cpp
// LyraGameSettingRegistry_MouseAndKeyboard.cpp

UGameSettingCollection* ULyraGameSettingRegistry::InitializeMouseAndKeyboardSettings(
    ULyraLocalPlayer* InLocalPlayer)
{
    UGameSettingCollection* Screen = NewObject<UGameSettingCollection>();
    Screen->SetDevName(TEXT("MouseAndKeyboardCollection"));
    Screen->SetDisplayName(LOCTEXT("MouseAndKeyboard_Name", "Mouse & Keyboard"));

    // 鼠标灵敏度
    {
        UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
        Setting->SetDevName(TEXT("MouseSensitivityX"));
        Setting->SetDisplayName(LOCTEXT("MouseSensX_Name", "Horizontal Sensitivity"));
        Setting->SetDescriptionRichText(
            LOCTEXT("MouseSensX_Description", 
            "Sets the sensitivity of the mouse when looking left and right."));

        Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetMouseSensitivityX));
        Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetMouseSensitivityX));
        
        Setting->SetDefaultValue(1.0);
        Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::RawTwoDecimals);
        Setting->SetMinimumLimit(0.1);
        Setting->SetMaximumLimit(10.0);
        Setting->SetSourceStepSize(0.01);

        Screen->AddSetting(Setting);
    }

    // 垂直灵敏度
    {
        UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
        Setting->SetDevName(TEXT("MouseSensitivityY"));
        Setting->SetDisplayName(LOCTEXT("MouseSensY_Name", "Vertical Sensitivity"));
        Setting->SetDescriptionRichText(
            LOCTEXT("MouseSensY_Description", 
            "Sets the sensitivity of the mouse when looking up and down."));

        Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetMouseSensitivityY));
        Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetMouseSensitivityY));
        
        Setting->SetDefaultValue(1.0);
        Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::RawTwoDecimals);
        Setting->SetMinimumLimit(0.1);
        Setting->SetMaximumLimit(10.0);
        Setting->SetSourceStepSize(0.01);

        Screen->AddSetting(Setting);
    }

    // 轴反转
    {
        UGameSettingValueDiscreteDynamic_Bool* Setting = 
            NewObject<UGameSettingValueDiscreteDynamic_Bool>();
        Setting->SetDevName(TEXT("InvertVerticalAxis"));
        Setting->SetDisplayName(LOCTEXT("InvertVert_Name", "Invert Vertical Axis"));
        Setting->SetDescriptionRichText(
            LOCTEXT("InvertVert_Description", 
            "Enable the inversion of the vertical look axis."));

        Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetInvertVerticalAxis));
        Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetInvertVerticalAxis));
        Setting->SetDefaultValue(false);

        Screen->AddSetting(Setting);
    }

    // 键位绑定（通过 Enhanced Input User Settings）
    {
        ULyraSettingKeyboardInput* Setting = NewObject<ULyraSettingKeyboardInput>();
        Setting->SetDevName(TEXT("KeyboardInput"));
        Setting->SetDisplayName(LOCTEXT("KeyboardInput_Name", "Keyboard Bindings"));
        
        Screen->AddSetting(Setting);
    }

    return Screen;
}
```

### 手柄设置

```cpp
// LyraGameSettingRegistry_Gamepad.cpp

UGameSettingCollection* ULyraGameSettingRegistry::InitializeGamepadSettings(
    ULyraLocalPlayer* InLocalPlayer)
{
    UGameSettingCollection* Screen = NewObject<UGameSettingCollection>();
    Screen->SetDevName(TEXT("GamepadCollection"));
    Screen->SetDisplayName(LOCTEXT("Gamepad_Name", "Gamepad"));

    // 手柄灵敏度预设
    {
        UGameSettingValueDiscreteDynamic_Enum* Setting = 
            NewObject<UGameSettingValueDiscreteDynamic_Enum>();
        Setting->SetDevName(TEXT("GamepadLookSensitivity"));
        Setting->SetDisplayName(LOCTEXT("GamepadLookSens_Name", "Look Sensitivity"));
        Setting->SetDescriptionRichText(
            LOCTEXT("GamepadLookSens_Description", 
            "Sets the sensitivity of the gamepad when looking around."));

        Setting->SetEnum(StaticEnum<ELyraGamepadSensitivity>());
        Setting->SetDynamicGetter(
            GET_SHARED_SETTINGS_FUNCTION_PATH(GetGamepadLookSensitivityPreset));
        Setting->SetDynamicSetter(
            GET_SHARED_SETTINGS_FUNCTION_PATH(SetLookSensitivityPreset));
        Setting->SetDefaultValue((int32)ELyraGamepadSensitivity::Normal);

        Screen->AddSetting(Setting);
    }

    // 死区设置
    {
        UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
        Setting->SetDevName(TEXT("GamepadMoveStickDeadZone"));
        Setting->SetDisplayName(LOCTEXT("MoveDeadZone_Name", "Move Stick Dead Zone"));
        Setting->SetDescriptionRichText(
            LOCTEXT("MoveDeadZone_Description", 
            "Sets the size of the dead zone for the move stick."));

        Setting->SetDynamicGetter(
            GET_SHARED_SETTINGS_FUNCTION_PATH(GetGamepadMoveStickDeadZone));
        Setting->SetDynamicSetter(
            GET_SHARED_SETTINGS_FUNCTION_PATH(SetGamepadMoveStickDeadZone));
        
        Setting->SetDefaultValue(0.25);
        Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::ZeroToOnePercent);
        Setting->SetMinimumLimit(0.0);
        Setting->SetMaximumLimit(0.95);
        Setting->SetSourceStepSize(0.01);

        Screen->AddSetting(Setting);
    }

    // 震动开关
    {
        UGameSettingValueDiscreteDynamic_Bool* Setting = 
            NewObject<UGameSettingValueDiscreteDynamic_Bool>();
        Setting->SetDevName(TEXT("ForceFeedback"));
        Setting->SetDisplayName(LOCTEXT("ForceFeedback_Name", "Vibration"));
        Setting->SetDescriptionRichText(
            LOCTEXT("ForceFeedback_Description", 
            "Enables controller vibration feedback."));

        Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetForceFeedbackEnabled));
        Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetForceFeedbackEnabled));
        Setting->SetDefaultValue(true);

        Screen->AddSetting(Setting);
    }

    // DualSense 扳机触觉反馈（PS5 专用）
    {
        UGameSettingValueDiscreteDynamic_Bool* Setting = 
            NewObject<UGameSettingValueDiscreteDynamic_Bool>();
        Setting->SetDevName(TEXT("TriggerHaptics"));
        Setting->SetDisplayName(LOCTEXT("TriggerHaptics_Name", "Trigger Haptics"));
        Setting->SetDescriptionRichText(
            LOCTEXT("TriggerHaptics_Description", 
            "Enables adaptive trigger haptic feedback on DualSense controllers."));

        Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetTriggerHapticsEnabled));
        Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetTriggerHapticsEnabled));
        Setting->SetDefaultValue(false);

        // 仅在 PS5 平台显示
        Setting->AddEditCondition(
            MakeShared<FWhenPlatformHasTrait>(
                FGameplayTag::RequestGameplayTag(TEXT("Platform.Trait.SupportsDualSense"))));

        Screen->AddSetting(Setting);
    }

    return Screen;
}
```

## 实战案例1：自定义游戏设置项

### 需求：添加游戏难度和辅助功能设置

假设我们要添加以下自定义设置：
1. **游戏难度**：简单、普通、困难、专家
2. **自动瞄准辅助**：开/关
3. **减少动作模糊**：开/关
4. **色盲友好 UI**：开/关

### 步骤1：定义数据模型

```cpp
// MyGameSettingsShared.h
#pragma once

#include "LyraSettingsShared.h"
#include "MyGameSettingsShared.generated.h"

UENUM(BlueprintType)
enum class EMyGameDifficulty : uint8
{
    Easy        UMETA(DisplayName = "Easy"),
    Normal      UMETA(DisplayName = "Normal"),
    Hard        UMETA(DisplayName = "Hard"),
    Expert      UMETA(DisplayName = "Expert")
};

UCLASS()
class UMyGameSettingsShared : public ULyraSettingsShared
{
    GENERATED_BODY()

public:
    // 游戏难度
    UFUNCTION()
    EMyGameDifficulty GetGameDifficulty() const { return GameDifficulty; }
    
    UFUNCTION()
    void SetGameDifficulty(EMyGameDifficulty NewDifficulty)
    {
        ChangeValueAndDirty(GameDifficulty, NewDifficulty);
        ApplyGameDifficulty();
    }

    // 自动瞄准辅助
    UFUNCTION()
    bool GetAutoAimAssist() const { return bAutoAimAssist; }
    
    UFUNCTION()
    void SetAutoAimAssist(bool bEnabled)
    {
        ChangeValueAndDirty(bAutoAimAssist, bEnabled);
    }

    // 减少动作模糊
    UFUNCTION()
    bool GetReduceMotionBlur() const { return bReduceMotionBlur; }
    
    UFUNCTION()
    void SetReduceMotionBlur(bool bEnabled)
    {
        if (ChangeValueAndDirty(bReduceMotionBlur, bEnabled))
        {
            ApplyMotionBlurSetting();
        }
    }

    // 色盲友好 UI
    UFUNCTION()
    bool GetColorBlindFriendlyUI() const { return bColorBlindFriendlyUI; }
    
    UFUNCTION()
    void SetColorBlindFriendlyUI(bool bEnabled)
    {
        ChangeValueAndDirty(bColorBlindFriendlyUI, bEnabled);
    }

protected:
    void ApplyGameDifficulty();
    void ApplyMotionBlurSetting();

private:
    UPROPERTY()
    EMyGameDifficulty GameDifficulty = EMyGameDifficulty::Normal;

    UPROPERTY()
    bool bAutoAimAssist = false;

    UPROPERTY()
    bool bReduceMotionBlur = false;

    UPROPERTY()
    bool bColorBlindFriendlyUI = false;
};
```

### 步骤2：实现应用逻辑

```cpp
// MyGameSettingsShared.cpp

void UMyGameSettingsShared::ApplyGameDifficulty()
{
    if (UWorld* World = GetWorld())
    {
        if (AMyGameMode* GameMode = Cast<AMyGameMode>(World->GetAuthGameMode()))
        {
            // 应用难度到游戏模式
            GameMode->SetDifficulty(GameDifficulty);
        }
    }
}

void UMyGameSettingsShared::ApplyMotionBlurSetting()
{
    // 调整后处理设置
    static IConsoleVariable* MotionBlurQuality = 
        IConsoleManager::Get().FindConsoleVariable(TEXT("r.MotionBlurQuality"));
    
    if (MotionBlurQuality)
    {
        MotionBlurQuality->Set(bReduceMotionBlur ? 0 : 4);
    }

    static IConsoleVariable* MotionBlurAmount = 
        IConsoleManager::Get().FindConsoleVariable(TEXT("r.MotionBlur.Amount"));
    
    if (MotionBlurAmount)
    {
        MotionBlurAmount->Set(bReduceMotionBlur ? 0.0f : 0.5f);
    }
}
```

### 步骤3：注册到设置系统

```cpp
// MyGameSettingRegistry.cpp

UGameSettingCollection* UMyGameSettingRegistry::InitializeGameplaySettings(
    ULyraLocalPlayer* InLocalPlayer)
{
    UGameSettingCollection* Screen = NewObject<UGameSettingCollection>();
    Screen->SetDevName(TEXT("GameplayCollection"));
    Screen->SetDisplayName(LOCTEXT("Gameplay_Name", "Gameplay"));

    // 游戏难度
    {
        UGameSettingValueDiscreteDynamic_Enum* Setting = 
            NewObject<UGameSettingValueDiscreteDynamic_Enum>();
        Setting->SetDevName(TEXT("GameDifficulty"));
        Setting->SetDisplayName(LOCTEXT("Difficulty_Name", "Game Difficulty"));
        Setting->SetDescriptionRichText(
            LOCTEXT("Difficulty_Description", 
            "Adjusts the challenge level of the game. This affects enemy health, "
            "damage dealt, and AI behavior."));

        Setting->SetEnum(StaticEnum<EMyGameDifficulty>());
        
        // 使用自定义的 SharedSettings 子类
        Setting->SetDynamicGetter(
            MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({
                GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings),
                GET_FUNCTION_NAME_STRING_CHECKED(UMyGameSettingsShared, GetGameDifficulty)
            }))
        );
        Setting->SetDynamicSetter(
            MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({
                GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings),
                GET_FUNCTION_NAME_STRING_CHECKED(UMyGameSettingsShared, SetGameDifficulty)
            }))
        );
        
        Setting->SetDefaultValue((int32)EMyGameDifficulty::Normal);

        Screen->AddSetting(Setting);
    }

    // 辅助功能集合
    {
        UGameSettingCollection* Accessibility = NewObject<UGameSettingCollection>();
        Accessibility->SetDevName(TEXT("AccessibilityCollection"));
        Accessibility->SetDisplayName(LOCTEXT("Accessibility_Name", "Accessibility"));
        Screen->AddSetting(Accessibility);

        // 自动瞄准辅助
        {
            UGameSettingValueDiscreteDynamic_Bool* Setting = 
                NewObject<UGameSettingValueDiscreteDynamic_Bool>();
            Setting->SetDevName(TEXT("AutoAimAssist"));
            Setting->SetDisplayName(LOCTEXT("AutoAim_Name", "Auto-Aim Assist"));
            Setting->SetDescriptionRichText(
                LOCTEXT("AutoAim_Description", 
                "Provides subtle aim correction to help target enemies."));

            Setting->SetDynamicGetter(
                MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({
                    GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings),
                    GET_FUNCTION_NAME_STRING_CHECKED(UMyGameSettingsShared, GetAutoAimAssist)
                }))
            );
            Setting->SetDynamicSetter(
                MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({
                    GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings),
                    GET_FUNCTION_NAME_STRING_CHECKED(UMyGameSettingsShared, SetAutoAimAssist)
                }))
            );
            
            Setting->SetDefaultValue(false);

            Accessibility->AddSetting(Setting);
        }

        // 减少动作模糊
        {
            UGameSettingValueDiscreteDynamic_Bool* Setting = 
                NewObject<UGameSettingValueDiscreteDynamic_Bool>();
            Setting->SetDevName(TEXT("ReduceMotionBlur"));
            Setting->SetDisplayName(LOCTEXT("MotionBlur_Name", "Reduce Motion Blur"));
            Setting->SetDescriptionRichText(
                LOCTEXT("MotionBlur_Description", 
                "Reduces motion blur effects, which may help with motion sickness."));

            Setting->SetDynamicGetter(
                MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({
                    GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings),
                    GET_FUNCTION_NAME_STRING_CHECKED(UMyGameSettingsShared, GetReduceMotionBlur)
                }))
            );
            Setting->SetDynamicSetter(
                MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({
                    GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings),
                    GET_FUNCTION_NAME_STRING_CHECKED(UMyGameSettingsShared, SetReduceMotionBlur)
                }))
            );
            
            Setting->SetDefaultValue(false);

            Accessibility->AddSetting(Setting);
        }

        // 色盲友好 UI
        {
            UGameSettingValueDiscreteDynamic_Bool* Setting = 
                NewObject<UGameSettingValueDiscreteDynamic_Bool>();
            Setting->SetDevName(TEXT("ColorBlindFriendlyUI"));
            Setting->SetDisplayName(
                LOCTEXT("ColorBlindUI_Name", "Color-Blind Friendly UI"));
            Setting->SetDescriptionRichText(
                LOCTEXT("ColorBlindUI_Description", 
                "Uses alternative colors and symbols in the UI to improve "
                "visibility for color-blind players."));

            Setting->SetDynamicGetter(
                MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({
                    GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings),
                    GET_FUNCTION_NAME_STRING_CHECKED(UMyGameSettingsShared, GetColorBlindFriendlyUI)
                }))
            );
            Setting->SetDynamicSetter(
                MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({
                    GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings),
                    GET_FUNCTION_NAME_STRING_CHECKED(UMyGameSettingsShared, SetColorBlindFriendlyUI)
                }))
            );
            
            Setting->SetDefaultValue(false);

            Accessibility->AddSetting(Setting);
        }
    }

    return Screen;
}
```

### 步骤4：在游戏中应用设置

```cpp
// MyWeaponComponent.cpp

void UMyWeaponComponent::ApplyAimAssist(FVector& InOutAimDirection)
{
    UMyGameSettingsShared* SharedSettings = GetSharedSettings();
    if (!SharedSettings || !SharedSettings->GetAutoAimAssist())
    {
        return; // 未启用自动瞄准
    }

    // 查找视野内的敌人
    AActor* BestTarget = FindBestTargetInView();
    if (!BestTarget)
    {
        return;
    }

    // 计算目标方向
    FVector TargetDirection = (BestTarget->GetActorLocation() - GetOwner()->GetActorLocation()).GetSafeNormal();

    // 应用轻微的瞄准修正
    float AssistStrength = 0.3f; // 30% 修正强度
    InOutAimDirection = FMath::Lerp(InOutAimDirection, TargetDirection, AssistStrength).GetSafeNormal();
}
```

## 实战案例2：跨平台设置云同步

### 需求：实现 Epic Online Services (EOS) 云同步

### 步骤1：集成 EOS 插件

```cpp
// MyGameInstance.h
#pragma once

#include "Engine/GameInstance.h"
#include "Interfaces/OnlineUserCloudInterface.h"
#include "MyGameInstance.generated.h"

UCLASS()
class UMyGameInstance : public UGameInstance
{
    GENERATED_BODY()

public:
    virtual void Init() override;
    virtual void Shutdown() override;

    // 同步设置到云端
    void SyncSettingsToCloud();

    // 从云端拉取设置
    void SyncSettingsFromCloud();

protected:
    void OnUserCloudReadComplete(bool bSuccess, const FUniqueNetId& UserId, const FString& FileName);
    void OnUserCloudWriteComplete(bool bSuccess, const FUniqueNetId& UserId, const FString& FileName);

    FDelegateHandle ReadDelegateHandle;
    FDelegateHandle WriteDelegateHandle;
};
```

### 步骤2：实现云同步逻辑

```cpp
// MyGameInstance.cpp

void UMyGameInstance::Init()
{
    Super::Init();

    // 注册云存储回调
    if (IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get())
    {
        if (IOnlineUserCloudPtr UserCloud = OnlineSub->GetUserCloudInterface())
        {
            ReadDelegateHandle = UserCloud->AddOnReadUserFileCompleteDelegate_Handle(
                FOnReadUserFileCompleteDelegate::CreateUObject(
                    this, &ThisClass::OnUserCloudReadComplete));

            WriteDelegateHandle = UserCloud->AddOnWriteUserFileCompleteDelegate_Handle(
                FOnWriteUserFileCompleteDelegate::CreateUObject(
                    this, &ThisClass::OnUserCloudWriteComplete));
        }
    }
}

void UMyGameInstance::SyncSettingsFromCloud()
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub)
    {
        UE_LOG(LogTemp, Warning, TEXT("No online subsystem available"));
        return;
    }

    IOnlineUserCloudPtr UserCloud = OnlineSub->GetUserCloudInterface();
    if (!UserCloud.IsValid())
    {
        UE_LOG(LogTemp, Warning, TEXT("User cloud interface not available"));
        return;
    }

    ULocalPlayer* LocalPlayer = GetFirstGamePlayer();
    if (!LocalPlayer)
    {
        return;
    }

    FUniqueNetIdPtr UserId = LocalPlayer->GetPreferredUniqueNetId().GetUniqueNetId();
    if (!UserId.IsValid())
    {
        UE_LOG(LogTemp, Warning, TEXT("Invalid user ID"));
        return;
    }

    // 枚举云端文件
    TArray<FCloudFileHeader> CloudFiles;
    UserCloud->GetUserFileList(*UserId, CloudFiles);

    // 查找设置文件
    FString SettingsFileName = TEXT("SharedGameSettings.sav");
    bool bFoundInCloud = false;

    for (const FCloudFileHeader& CloudFile : CloudFiles)
    {
        if (CloudFile.FileName == SettingsFileName)
        {
            bFoundInCloud = true;
            
            // 读取云端文件
            UserCloud->ReadUserFile(*UserId, SettingsFileName);
            break;
        }
    }

    if (!bFoundInCloud)
    {
        UE_LOG(LogTemp, Log, TEXT("Settings file not found in cloud, will use local"));
    }
}

void UMyGameInstance::OnUserCloudReadComplete(bool bSuccess, 
                                               const FUniqueNetId& UserId, 
                                               const FString& FileName)
{
    if (!bSuccess)
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to read cloud file: %s"), *FileName);
        return;
    }

    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    IOnlineUserCloudPtr UserCloud = OnlineSub->GetUserCloudInterface();
    
    // 获取云端数据
    TArray<uint8> CloudData;
    if (!UserCloud->GetFileContents(UserId, FileName, CloudData))
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to get cloud file contents"));
        return;
    }

    // 读取本地设置
    ULyraLocalPlayer* LocalPlayer = Cast<ULyraLocalPlayer>(GetFirstGamePlayer());
    ULyraSettingsShared* LocalSettings = LocalPlayer->GetSharedSettings();

    // 从云端数据反序列化
    ULyraSettingsShared* CloudSettings = NewObject<ULyraSettingsShared>();
    FMemoryReader MemoryReader(CloudData);
    FObjectAndNameAsStringProxyArchive Ar(MemoryReader, false);
    CloudSettings->Serialize(Ar);

    // 比较时间戳（需要在 SettingsShared 中添加时间戳字段）
    if (CloudSettings->GetLastModifiedTimestamp() > LocalSettings->GetLastModifiedTimestamp())
    {
        // 云端更新，使用云端设置
        UE_LOG(LogTemp, Log, TEXT("Cloud settings are newer, applying..."));
        LocalSettings->CopyFrom(CloudSettings);
        LocalSettings->ApplySettings();
        LocalSettings->SaveSettings();
    }
    else
    {
        // 本地更新，上传到云端
        UE_LOG(LogTemp, Log, TEXT("Local settings are newer, uploading..."));
        SyncSettingsToCloud();
    }
}

void UMyGameInstance::SyncSettingsToCloud()
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub)
    {
        return;
    }

    IOnlineUserCloudPtr UserCloud = OnlineSub->GetUserCloudInterface();
    if (!UserCloud.IsValid())
    {
        return;
    }

    ULocalPlayer* LocalPlayer = GetFirstGamePlayer();
    if (!LocalPlayer)
    {
        return;
    }

    FUniqueNetIdPtr UserId = LocalPlayer->GetPreferredUniqueNetId().GetUniqueNetId();
    if (!UserId.IsValid())
    {
        return;
    }

    // 序列化设置到字节数组
    ULyraSettingsShared* SharedSettings = 
        Cast<ULyraLocalPlayer>(LocalPlayer)->GetSharedSettings();

    TArray<uint8> SettingsData;
    FMemoryWriter MemoryWriter(SettingsData);
    FObjectAndNameAsStringProxyArchive Ar(MemoryWriter, false);
    SharedSettings->Serialize(Ar);

    // 上传到云端
    FString SettingsFileName = TEXT("SharedGameSettings.sav");
    UserCloud->WriteUserFile(*UserId, SettingsFileName, SettingsData);
}

void UMyGameInstance::OnUserCloudWriteComplete(bool bSuccess, 
                                                const FUniqueNetId& UserId, 
                                                const FString& FileName)
{
    if (bSuccess)
    {
        UE_LOG(LogTemp, Log, TEXT("Successfully uploaded settings to cloud: %s"), *FileName);
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to upload settings to cloud: %s"), *FileName);
    }
}

void UMyGameInstance::Shutdown()
{
    // 清理回调
    if (IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get())
    {
        if (IOnlineUserCloudPtr UserCloud = OnlineSub->GetUserCloudInterface())
        {
            UserCloud->ClearOnReadUserFileCompleteDelegate_Handle(ReadDelegateHandle);
            UserCloud->ClearOnWriteUserFileCompleteDelegate_Handle(WriteDelegateHandle);
        }
    }

    Super::Shutdown();
}
```

### 步骤3：添加时间戳支持

```cpp
// LyraSettingsShared.h (扩展)

UCLASS()
class ULyraSettingsShared : public ULocalPlayerSaveGame
{
    GENERATED_BODY()

public:
    // 获取最后修改时间戳
    int64 GetLastModifiedTimestamp() const { return LastModifiedTimestamp; }

    // 从另一个设置对象复制
    void CopyFrom(const ULyraSettingsShared* OtherSettings);

protected:
    template<typename T>
    bool ChangeValueAndDirty(T& CurrentValue, const T& NewValue)
    {
        if (CurrentValue != NewValue)
        {
            CurrentValue = NewValue;
            bIsDirty = true;
            
            // 更新时间戳
            LastModifiedTimestamp = FDateTime::UtcNow().ToUnixTimestamp();
            
            OnSettingChanged.Broadcast(this);
            return true;
        }
        return false;
    }

private:
    UPROPERTY()
    int64 LastModifiedTimestamp = 0;
};
```

### 步骤4：在登录时触发同步

```cpp
// MyPlayerController.cpp

void AMyPlayerController::OnLoginComplete(int32 LocalUserNum, 
                                          bool bWasSuccessful, 
                                          const FUniqueNetId& UserId, 
                                          const FString& Error)
{
    if (bWasSuccessful)
    {
        UE_LOG(LogTemp, Log, TEXT("Login successful, syncing settings from cloud..."));
        
        if (UMyGameInstance* GameInstance = Cast<UMyGameInstance>(GetGameInstance()))
        {
            GameInstance->SyncSettingsFromCloud();
        }
    }
}
```

## 性能优化与最佳实践

### 1. 延迟应用设置

频繁应用某些设置（如分辨率、质量等级）会导致卡顿，应批量应用：

```cpp
// LyraGameSettingRegistry.cpp

void ULyraGameSettingRegistry::SaveChanges()
{
    Super::SaveChanges();
    
    if (ULyraLocalPlayer* LocalPlayer = Cast<ULyraLocalPlayer>(OwningLocalPlayer))
    {
        // 批量应用所有变更，只触发一次重建
        LocalPlayer->GetLocalSettings()->ApplySettings(false);
        
        LocalPlayer->GetSharedSettings()->ApplySettings();
        LocalPlayer->GetSharedSettings()->SaveSettings();
    }
}
```

### 2. 异步加载与保存

避免阻塞主线程：

```cpp
// 启动时异步加载设置
void UMyGameInstance::Init()
{
    Super::Init();

    ULyraLocalPlayer* LocalPlayer = Cast<ULyraLocalPlayer>(GetFirstGamePlayer());
    if (!LocalPlayer)
    {
        return;
    }

    // 异步加载 SharedSettings
    ULyraSettingsShared::AsyncLoadOrCreateSettings(LocalPlayer,
        ULyraSettingsShared::FOnSettingsLoadedEvent::CreateLambda(
            [](ULyraSettingsShared* LoadedSettings)
            {
                UE_LOG(LogTemp, Log, TEXT("Settings loaded asynchronously"));
                // 设置已自动应用
            }
        )
    );
}
```

### 3. 分帧应用复杂设置

对于需要重建大量资源的设置（如材质质量），分帧处理：

```cpp
void ULyraSettingsLocal::ApplyScalabilitySettings()
{
    Scalability::SetQualityLevels(GetQualityLevels());

    // 分帧刷新材质缓存
    GEngine->TriggerStreamingDataRebuild();
    
    // 使用协程分帧处理
    StartCoroutine(FLyraCoroutine::CreateLambda([this]()
    {
        int32 FrameCount = 0;
        for (TObjectIterator<UMaterialInstance> It; It; ++It)
        {
            It->InitResources();
            
            // 每处理10个材质等待一帧
            if (++FrameCount % 10 == 0)
            {
                co_await FLyraCoroutine::NextFrame();
            }
        }
    }));
}
```

### 4. 设置验证

在设置器中添加验证逻辑，防止无效值：

```cpp
void ULyraSettingsLocal::SetFrameRateLimit_Always(float NewLimitFPS)
{
    // 验证范围
    NewLimitFPS = FMath::Clamp(NewLimitFPS, 30.0f, 240.0f);
    
    // 验证平台支持
    if (NewLimitFPS > GetMaxSupportedFrameRate())
    {
        NewLimitFPS = GetMaxSupportedFrameRate();
    }

    if (FrameRateLimit_Always != NewLimitFPS)
    {
        FrameRateLimit_Always = NewLimitFPS;
        UpdateEffectiveFrameRateLimit();
        SaveConfig();
    }
}
```

### 5. 避免循环依赖

设置变更可能触发其他设置的更新，使用防护标志：

```cpp
// LyraSettingsLocal.cpp

void ULyraSettingsLocal::SetOverallScalabilityLevel(int32 Value)
{
    if (bSettingOverallQualityGuard)
    {
        return; // 防止递归
    }

    TGuardValue<bool> Guard(bSettingOverallQualityGuard, true);

    // 更新所有质量通道
    Scalability::FQualityLevels Levels = Scalability::GetQualityLevels();
    Levels.SetFromSingleQualityLevel(Value);
    
    Scalability::SetQualityLevels(Levels);
}
```

### 6. 设置重置优化

提供智能重置功能，只重置变更的项：

```cpp
void ULyraSettingsLocal::ResetToCurrentSettings()
{
    Super::ResetToCurrentSettings();

    // 记录当前状态为"初始"状态
    for (UGameSetting* Setting : GetRegistry()->GetAllSettings())
    {
        if (UGameSettingValue* ValueSetting = Cast<UGameSettingValue>(Setting))
        {
            ValueSetting->StoreInitial();
        }
    }
}
```

### 7. 平台特定优化

根据平台特性优化设置逻辑：

```cpp
void ULyraSettingsLocal::ApplySettings(bool bCheckForCommandLineOverrides)
{
    Super::ApplySettings(bCheckForCommandLineOverrides);

#if PLATFORM_MOBILE
    // 移动平台特殊处理
    ApplyMobileSpecificSettings();
#elif PLATFORM_CONSOLE
    // 主机平台特殊处理
    ApplyConsoleSpecificSettings();
#else
    // PC 平台特殊处理
    ApplyDesktopSpecificSettings();
#endif
}
```

### 8. 内存优化

避免在设置对象中存储大对象引用：

```cpp
// ❌ 不好的做法
UCLASS()
class UBadSettings : public ULyraSettingsShared
{
    UPROPERTY()
    TArray<UTexture2D*> CachedTextures; // 持有大量纹理引用
};

// ✅ 好的做法
UCLASS()
class UGoodSettings : public ULyraSettingsShared
{
    UPROPERTY()
    TArray<FSoftObjectPath> TexturePaths; // 仅存储路径
};
```

## 总结

UE5 Lyra 的游戏设置系统展示了一套完善的架构设计：

### 核心优势

1. **高度解耦**：通过 DataSource 模式实现 UI、逻辑与数据的分离
2. **易于扩展**：清晰的类层次结构，添加新设置只需最少代码
3. **双重持久化**：Local Settings（机器相关）+ Shared Settings（用户相关）
4. **跨平台支持**：内置平台特性检测和自适应调整
5. **事件驱动**：细粒度的变更通知和自动 UI 更新
6. **云同步友好**：基于 SaveGame 系统，天然支持云存储

### 最佳实践总结

- **设置分类明确**：机器设置用 GameUserSettings，用户设置用 SaveGame
- **异步优先**：加载和保存都应使用异步 API
- **批量应用**：避免频繁触发昂贵操作（分辨率切换、材质重建等）
- **验证输入**：在 Setter 中验证值的合法性和平台兼容性
- **分帧处理**：复杂操作使用协程或定时器分散到多帧
- **时间戳管理**：支持云同步时必须记录修改时间
- **防护递归**：设置间的依赖关系需要使用防护标志

### 扩展方向

1. **A/B 测试**：在设置系统中集成实验框架，测试不同默认值的效果
2. **推荐引擎**：根据硬件自动推荐最佳设置组合
3. **设置模板**：预设多个场景（电竞、观影、省电）的设置组合
4. **远程配置**：通过服务器下发设置覆盖，用于紧急调整
5. **无障碍增强**：添加更多辅助功能（文字转语音、高对比度模式等）

通过深入理解 Lyra 的设置系统，我们可以为自己的项目构建出既用户友好又开发高效的配置管理方案。

---

**相关文章**：
- [UI 框架架构：CommonUI 深度解析](./14-ui-framework-architecture.md)
- [输入系统：Enhanced Input 完全指南](./15-input-system-enhanced-input.md)

**源码参考**：
- `Lyra/Plugins/GameSettings/` - GameSettings 插件
- `Lyra/Source/LyraGame/Settings/` - Lyra 设置实现
- `Lyra/Source/LyraGame/Player/LyraLocalPlayer.h` - 本地玩家集成
