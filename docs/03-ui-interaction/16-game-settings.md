# 16. 游戏设置系统：保存与同步

> 深入剖析 Lyra 的设置管理架构，从本地配置到跨平台同步的完整解决方案

---

## 📋 本章概览

游戏设置系统是现代游戏不可或缺的基础功能，承载着玩家的个性化偏好——从画质调节、音量控制到按键绑定。Lyra 提供了一套完整且可扩展的设置系统，巧妙平衡了**本地特性**（如分辨率、帧率限制）和**用户偏好**（如灵敏度、字幕选项），并通过 SaveGame 机制实现跨设备同步。

本章将深入解析 Lyra 设置系统的核心设计：

- **双层设置架构**：`ULyraSettingsLocal` 与 `ULyraSettingsShared` 的职责划分
- **注册表模式**：`ULyraGameSettingRegistry` 如何集中管理设置项
- **数据绑定机制**：`GameSettingDataSource` 动态路径解析
- **实时同步**：设置变更通知与生效流程
- **实战案例**：添加自定义设置、实现持久化存储、调试技巧

通过本章学习，你将能够：

✅ 理解 Lyra 设置系统的分层设计哲学  
✅ 掌握添加新设置项的标准流程  
✅ 实现跨平台的用户偏好同步  
✅ 调试设置保存加载问题  

---

## 🏗️ 系统架构概览

### 核心组件关系

Lyra 的设置系统采用**双层存储 + 注册表管理**的架构：

```
                    ┌───────────────────────────────┐
                    │  ULyraGameSettingRegistry     │
                    │  (设置注册表 - 集中管理)        │
                    └──────────┬────────────────────┘
                               │ 注册所有设置项
                ┌──────────────┴──────────────┐
                │                              │
    ┌───────────▼────────────┐     ┌──────────▼──────────┐
    │  ULyraSettingsLocal    │     │  ULyraSettingsShared │
    │  (本地设置 - 机器相关)  │     │  (共享设置 - 用户相关)│
    │                         │     │                      │
    │  • 分辨率 / 帧率        │     │  • 灵敏度 / 按键绑定 │
    │  • 显卡质量             │     │  • 字幕选项          │
    │  • 音频设备             │     │  • 辅助功能          │
    │                         │     │                      │
    │  存储: GameUserSettings │     │  存储: SaveGame      │
    └─────────────────────────┘     └──────────────────────┘
                │                              │
                └──────────┬───────────────────┘
                           │ 应用到引擎
                   ┌───────▼────────┐
                   │  Scalability    │
                   │  Audio Mixer    │
                   │  Input System   │
                   └─────────────────┘
```

**设计要点**：

1. **Local vs Shared 分离**：
   - `Local`：机器相关（显卡型号、显示器分辨率），存储在 `Saved/Config/` 的 `GameUserSettings.ini`
   - `Shared`：用户相关（操作习惯、偏好选项），存储在 `SaveGames/` 目录，可云同步

2. **注册表模式**：
   - 所有设置项统一在 `ULyraGameSettingRegistry` 中注册
   - UI 通过遍历注册表动态生成设置界面
   - 避免硬编码，易于扩展

3. **数据源动态绑定**：
   - 使用 `FGameSettingDataSourceDynamic` 通过反射路径访问 Getter/Setter
   - 解耦 UI 与数据层，支持热重载

---

## 📦 ULyraSettingsLocal：本地设置

### 核心职责

`ULyraSettingsLocal` 继承自 `UGameUserSettings`（引擎标准的本地设置类），负责管理**机器相关**的配置：

```cpp
// LyraSettingsLocal.h
UCLASS()
class ULyraSettingsLocal : public UGameUserSettings
{
    GENERATED_BODY()

public:
    // 单例访问
    static ULyraSettingsLocal* Get();

    // 图形设置
    UFUNCTION() float GetDisplayGamma() const;
    UFUNCTION() void SetDisplayGamma(float InGamma);

    // 帧率限制
    UFUNCTION() float GetFrameRateLimit_OnBattery() const;
    UFUNCTION() void SetFrameRateLimit_OnBattery(float NewLimitFPS);
    
    UFUNCTION() float GetFrameRateLimit_InMenu() const;
    UFUNCTION() void SetFrameRateLimit_InMenu(float NewLimitFPS);

    // 音频设备
    UFUNCTION() FString GetAudioOutputDeviceId() const;
    UFUNCTION() void SetAudioOutputDeviceId(const FString& InAudioOutputDeviceId);

    // 性能统计显示
    ELyraStatDisplayMode GetPerfStatDisplayState(ELyraDisplayablePerformanceStat Stat) const;
    void SetPerfStatDisplayState(ELyraDisplayablePerformanceStat Stat, ELyraStatDisplayMode DisplayMode);

    // 安全区（主机平台）
    UFUNCTION() float GetSafeZone() const { return SafeZoneScale >= 0 ? SafeZoneScale : 0; }
    UFUNCTION() void SetSafeZone(float Value);

    // 应用设置
    virtual void ApplyNonResolutionSettings() override;

private:
    UPROPERTY(Config) float DisplayGamma = 2.2;
    UPROPERTY(Config) float FrameRateLimit_OnBattery;
    UPROPERTY(Config) float FrameRateLimit_InMenu;
    UPROPERTY(Config) FString AudioOutputDeviceId;
    UPROPERTY(Config) float SafeZoneScale = -1;
    
    // 音量控制总线
    UPROPERTY(Transient)
    TMap<FName, TObjectPtr<USoundControlBus>> ControlBusMap;
    
    UPROPERTY(Transient)
    TObjectPtr<USoundControlBusMix> ControlBusMix = nullptr;
};
```

### 关键机制

#### 1. 帧率限制策略

Lyra 针对不同场景设置独立的帧率上限：

```cpp
void ULyraSettingsLocal::UpdateEffectiveFrameRateLimit()
{
    float EffectiveFrameRateLimit = 0.0f;

    if (!IsRunningDedicatedServer())
    {
        // 电池供电时
        if (FPlatformMisc::IsRunningOnBattery())
        {
            EffectiveFrameRateLimit = FrameRateLimit_OnBattery;
        }
        // 菜单界面
        else if (ShouldUseFrontendPerformanceSettings())
        {
            EffectiveFrameRateLimit = FrameRateLimit_InMenu;
        }
        // 游戏中
        else
        {
            EffectiveFrameRateLimit = GetFrameRateLimit_Always();
        }
    }

    // 应用到引擎全局
    GEngine->SetMaxFPS(EffectiveFrameRateLimit);
}
```

**设计亮点**：

- **动态切换**：游戏、菜单、电池模式自动调整帧率，优化性能与功耗
- **移动端特化**：针对 iOS/Android 的 FPS 模式（30/60/120）有独立逻辑

#### 2. 音频控制总线 (Sound Control Bus)

Lyra 使用 UE5 的 **Audio Modulation** 系统管理音量：

```cpp
void ULyraSettingsLocal::SetVolumeForControlBus(USoundControlBus* InSoundControlBus, float InVolume)
{
    if (!ControlBusMix)
    {
        LoadUserControlBusMix();
    }

    if (ControlBusMix && InSoundControlBus)
    {
        // 设置控制总线的音量
        FSoundControlBusMixStage BusMixStage;
        BusMixStage.Bus = InSoundControlBus;
        BusMixStage.Value.TargetValue = InVolume;
        BusMixStage.Value.AttackTime = 0.5f; // 淡入时间
        BusMixStage.Value.ReleaseTime = 0.5f;

        ControlBusMix->AddOrUpdateStage(BusMixStage);
    }
}

void ULyraSettingsLocal::SetOverallVolume(float InVolume)
{
    OverallVolume = InVolume;
    
    // 从映射表中获取对应的控制总线
    if (USoundControlBus* OverallBus = ControlBusMap.FindRef(TEXT("Overall")))
    {
        SetVolumeForControlBus(OverallBus, InVolume);
    }
}
```

**优势**：

- **实时调节**：无需重启，通过 Control Bus 直接修改混音
- **分类管理**：Overall（总音量）、Music（音乐）、SFX（音效）、Dialogue（对话）独立控制
- **淡入淡出**：AttackTime/ReleaseTime 避免音量突变

#### 3. 性能统计显示

Lyra 允许玩家自定义 HUD 上显示的性能指标：

```cpp
enum class ELyraDisplayablePerformanceStat : uint8
{
    ClientFPS,          // 客户端帧率
    ServerFPS,          // 服务器帧率
    IdleTime,           // 空闲时间
    FrameTime,          // 帧时间
    Ping,               // 网络延迟
    PacketLoss_Incoming,// 丢包率（接收）
    PacketLoss_Outgoing,// 丢包率（发送）
    // ...
};

enum class ELyraStatDisplayMode : uint8
{
    Hidden,     // 隐藏
    TextOnly,   // 仅文本
    GraphOnly,  // 仅图表
    TextAndGraph// 文本+图表
};

ELyraStatDisplayMode ULyraSettingsLocal::GetPerfStatDisplayState(ELyraDisplayablePerformanceStat Stat) const
{
    if (const ELyraStatDisplayMode* DisplayMode = DisplayStatList.Find(Stat))
    {
        return *DisplayMode;
    }
    return ELyraStatDisplayMode::Hidden;
}

void ULyraSettingsLocal::SetPerfStatDisplayState(ELyraDisplayablePerformanceStat Stat, ELyraStatDisplayMode DisplayMode)
{
    if (DisplayMode == ELyraStatDisplayMode::Hidden)
    {
        DisplayStatList.Remove(Stat);
    }
    else
    {
        DisplayStatList.Add(Stat, DisplayMode);
    }

    // 通知 HUD 刷新
    PerfStatSettingsChangedEvent.Broadcast();
}
```

**应用场景**：

- 开发阶段：显示 Frame Time、Server FPS 调试性能
- 竞技玩家：显示 Ping、丢包率监控网络质量
- 主播/录制：隐藏所有统计信息保持画面简洁

---

## 🌐 ULyraSettingsShared：共享设置

### 核心职责

`ULyraSettingsShared` 继承自 `ULocalPlayerSaveGame`（存储为 SaveGame 文件），管理**用户相关**的偏好：

```cpp
// LyraSettingsShared.h
UCLASS()
class ULyraSettingsShared : public ULocalPlayerSaveGame
{
    GENERATED_BODY()

public:
    // 鼠标灵敏度
    UFUNCTION() double GetMouseSensitivityX() const;
    UFUNCTION() void SetMouseSensitivityX(double NewValue);

    // 手柄灵敏度预设
    UFUNCTION() ELyraGamepadSensitivity GetGamepadLookSensitivityPreset() const;
    UFUNCTION() void SetLookSensitivityPreset(ELyraGamepadSensitivity NewValue);

    // 反转轴向
    UFUNCTION() bool GetInvertVerticalAxis() const;
    UFUNCTION() void SetInvertVerticalAxis(bool NewValue);

    // 手柄震动
    UFUNCTION() bool GetForceFeedbackEnabled() const;
    UFUNCTION() void SetForceFeedbackEnabled(const bool NewValue);

    // 字幕选项
    UFUNCTION() bool GetSubtitlesEnabled() const;
    UFUNCTION() void SetSubtitlesEnabled(bool Value);
    UFUNCTION() ESubtitleDisplayTextSize GetSubtitlesTextSize() const;

    // 辅助功能：色盲模式
    UFUNCTION() EColorBlindMode GetColorBlindMode() const;
    UFUNCTION() void SetColorBlindMode(EColorBlindMode InMode);
    UFUNCTION() int32 GetColorBlindStrength() const;

    // 语言/文化
    const FString& GetPendingCulture() const;
    void SetPendingCulture(const FString& NewCulture);
    void ApplyCultureSettings();

    // 异步加载
    static ULyraSettingsShared* LoadOrCreateSettings(const ULyraLocalPlayer* LocalPlayer);
    static bool AsyncLoadOrCreateSettings(const ULyraLocalPlayer* LocalPlayer, FOnSettingsLoadedEvent Delegate);

    // 保存到磁盘
    void SaveSettings();
    void ApplySettings();

private:
    UPROPERTY() double MouseSensitivityX = 1.0;
    UPROPERTY() double MouseSensitivityY = 1.0;
    UPROPERTY() bool bInvertVerticalAxis = false;
    UPROPERTY() bool bForceFeedbackEnabled = true;
    UPROPERTY() bool bEnableSubtitles = true;
    UPROPERTY() EColorBlindMode ColorBlindMode = EColorBlindMode::Off;
    UPROPERTY() int32 ColorBlindStrength = 10;

    // 脏标记
    bool bIsDirty = false;
};
```

### 关键机制

#### 1. SaveGame 存储

`ULocalPlayerSaveGame` 是 UE 的标准存档基类，Lyra 利用它实现设置同步：

```cpp
void ULyraSettingsShared::SaveSettings()
{
    if (bIsDirty)
    {
        // 获取存档槽名（每个玩家独立存档）
        const FString SlotName = FString::Printf(TEXT("LyraSettings_%d"), GetLocalPlayer()->GetControllerId());
        
        // 异步保存到磁盘
        UGameplayStatics::AsyncSaveGameToSlot(this, SlotName, 0,
            FAsyncSaveGameToSlotDelegate::CreateLambda([this](const FString& SaveSlotName, const int32 UserIndex, bool bSuccess)
            {
                if (bSuccess)
                {
                    UE_LOG(LogLyraSettings, Log, TEXT("Settings saved successfully: %s"), *SaveSlotName);
                    bIsDirty = false;
                }
                else
                {
                    UE_LOG(LogLyraSettings, Warning, TEXT("Failed to save settings: %s"), *SaveSlotName);
                }
            }));
    }
}

ULyraSettingsShared* ULyraSettingsShared::LoadOrCreateSettings(const ULyraLocalPlayer* LocalPlayer)
{
    const FString SlotName = FString::Printf(TEXT("LyraSettings_%d"), LocalPlayer->GetControllerId());

    // 尝试从磁盘加载
    if (ULyraSettingsShared* LoadedSettings = Cast<ULyraSettingsShared>(UGameplayStatics::LoadGameFromSlot(SlotName, 0)))
    {
        UE_LOG(LogLyraSettings, Log, TEXT("Loaded settings from slot: %s"), *SlotName);
        return LoadedSettings;
    }

    // 加载失败则创建默认设置
    ULyraSettingsShared* NewSettings = Cast<ULyraSettingsShared>(UGameplayStatics::CreateSaveGameObject(ULyraSettingsShared::StaticClass()));
    NewSettings->ApplySettings(); // 应用默认值
    return NewSettings;
}
```

**存储路径**：

```
<ProjectDir>/Saved/SaveGames/LyraSettings_0.sav  (玩家 1)
<ProjectDir>/Saved/SaveGames/LyraSettings_1.sav  (玩家 2)
```

#### 2. 灵敏度系统

Lyra 提供预设灵敏度档位，简化玩家调节：

```cpp
enum class ELyraGamepadSensitivity : uint8
{
    Slow            UMETA(DisplayName = "01 - Slow"),
    SlowPlus        UMETA(DisplayName = "02 - Slow+"),
    SlowPlusPlus    UMETA(DisplayName = "03 - Slow++"),
    Normal          UMETA(DisplayName = "04 - Normal"),
    NormalPlus      UMETA(DisplayName = "05 - Normal+"),
    NormalPlusPlus  UMETA(DisplayName = "06 - Normal++"),
    Fast            UMETA(DisplayName = "07 - Fast"),
    FastPlus        UMETA(DisplayName = "08 - Fast+"),
    FastPlusPlus    UMETA(DisplayName = "09 - Fast++"),
    Insane          UMETA(DisplayName = "10 - Insane"),
};

void ULyraSettingsShared::ApplyInputSensitivity()
{
    ULyraLocalPlayer* LocalPlayer = GetLocalPlayer<ULyraLocalPlayer>();
    if (!LocalPlayer) return;

    // 根据预设转换为实际倍率
    float LookMultiplier = 1.0f;
    switch (GamepadLookSensitivityPreset)
    {
    case ELyraGamepadSensitivity::Slow:         LookMultiplier = 0.5f; break;
    case ELyraGamepadSensitivity::SlowPlus:     LookMultiplier = 0.65f; break;
    case ELyraGamepadSensitivity::SlowPlusPlus: LookMultiplier = 0.8f; break;
    case ELyraGamepadSensitivity::Normal:       LookMultiplier = 1.0f; break;
    case ELyraGamepadSensitivity::Fast:         LookMultiplier = 1.5f; break;
    case ELyraGamepadSensitivity::Insane:       LookMultiplier = 2.5f; break;
    // ...
    }

    // 应用到 Enhanced Input System
    if (UEnhancedInputLocalPlayerSubsystem* InputSubsystem = LocalPlayer->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>())
    {
        // 获取 Look Input Modifier
        if (UInputModifier* LookModifier = FindModifierByClass(InputSubsystem, UInputModifierScalar::StaticClass()))
        {
            CastChecked<UInputModifierScalar>(LookModifier)->SetScalar(FVector2D(LookMultiplier, LookMultiplier));
        }
    }
}
```

**设计思路**：

- **预设化**：避免玩家盲目调节数值，提供经过测试的档位
- **分离控制**：平视灵敏度与瞄准灵敏度独立（`TargetingMultiplier`）
- **实时生效**：修改后立即更新 Input Modifier，无需重启关卡

#### 3. 辅助功能：色盲模式

Lyra 集成 UE5 的 **Color Blindness Simulation**：

```cpp
void ULyraSettingsShared::SetColorBlindMode(EColorBlindMode InMode)
{
    if (ChangeValueAndDirty(ColorBlindMode, InMode))
    {
        // 应用到后处理
        ApplyColorBlindSettings();
    }
}

void ULyraSettingsShared::ApplyColorBlindSettings()
{
    // 获取游戏视口
    if (UGameViewportClient* ViewportClient = GetWorld()->GetGameViewport())
    {
        FColorVisionDeficiency CVDSettings;

        switch (ColorBlindMode)
        {
        case EColorBlindMode::Deuteranope:  // 绿色弱
            CVDSettings.Type = EColorVisionDeficiency::Deuteranope;
            break;
        case EColorBlindMode::Protanope:    // 红色弱
            CVDSettings.Type = EColorVisionDeficiency::Protanope;
            break;
        case EColorBlindMode::Tritanope:    // 蓝色弱
            CVDSettings.Type = EColorVisionDeficiency::Tritanope;
            break;
        default:
            CVDSettings.Type = EColorVisionDeficiency::NormalVision;
            break;
        }

        CVDSettings.Severity = ColorBlindStrength / 10.0f; // 0-1 强度
        ViewportClient->GetEngineShowFlags()->SetColorVisionDeficiency(CVDSettings);
    }
}
```

---

## 🗂️ ULyraGameSettingRegistry：注册表

### 核心职责

`ULyraGameSettingRegistry` 是设置系统的**中央注册表**，负责：

1. **集中注册**所有设置项（视频、音频、游戏玩法、输入）
2. **动态生成** UI 界面（遍历注册表创建设置控件）
3. **保存协调**：统一触发 Local 和 Shared 的保存操作

```cpp
// LyraGameSettingRegistry.h
UCLASS()
class ULyraGameSettingRegistry : public UGameSettingRegistry
{
    GENERATED_BODY()

public:
    static ULyraGameSettingRegistry* Get(ULyraLocalPlayer* InLocalPlayer);

    virtual void SaveChanges() override;

protected:
    virtual void OnInitialize(ULocalPlayer* InLocalPlayer) override;

    // 初始化各类设置
    UGameSettingCollection* InitializeVideoSettings(ULyraLocalPlayer* InLocalPlayer);
    UGameSettingCollection* InitializeAudioSettings(ULyraLocalPlayer* InLocalPlayer);
    UGameSettingCollection* InitializeGameplaySettings(ULyraLocalPlayer* InLocalPlayer);
    UGameSettingCollection* InitializeMouseAndKeyboardSettings(ULyraLocalPlayer* InLocalPlayer);
    UGameSettingCollection* InitializeGamepadSettings(ULyraLocalPlayer* InLocalPlayer);

private:
    UPROPERTY()
    TObjectPtr<UGameSettingCollection> VideoSettings;

    UPROPERTY()
    TObjectPtr<UGameSettingCollection> AudioSettings;

    UPROPERTY()
    TObjectPtr<UGameSettingCollection> GameplaySettings;

    // ...
};
```

### 设置项注册流程

#### 示例：添加"鼠标灵敏度"设置

```cpp
UGameSettingCollection* ULyraGameSettingRegistry::InitializeMouseAndKeyboardSettings(ULyraLocalPlayer* InLocalPlayer)
{
    UGameSettingCollection* Screen = NewObject<UGameSettingCollection>();
    Screen->SetDevName(TEXT("MouseAndKeyboardSettings"));
    Screen->SetDisplayName(LOCTEXT("MouseAndKeyboardSettings_Name", "Mouse & Keyboard"));
    Screen->Initialize(InLocalPlayer);

    // 创建设置项
    UGameSettingValueScalarDynamic* MouseSensitivityX = NewObject<UGameSettingValueScalarDynamic>();
    MouseSensitivityX->SetDevName(TEXT("MouseSensitivityX"));
    MouseSensitivityX->SetDisplayName(LOCTEXT("MouseSensitivityX_Name", "鼠标水平灵敏度"));
    MouseSensitivityX->SetDescriptionRichText(LOCTEXT("MouseSensitivityX_Description", "调整鼠标左右移动的灵敏度"));

    // 设置数值范围
    MouseSensitivityX->SetDefaultValue(1.0);
    MouseSensitivityX->SetMinValue(0.1);
    MouseSensitivityX->SetMaxValue(3.0);
    MouseSensitivityX->SetStepValue(0.1);
    MouseSensitivityX->SetDisplayFormat([](double Value) { return FText::AsNumber(Value); });

    // 绑定到数据源（动态路径）
    MouseSensitivityX->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetMouseSensitivityX));
    MouseSensitivityX->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetMouseSensitivityX));

    // 添加到集合
    Screen->AddSetting(MouseSensitivityX);

    return Screen;
}
```

**宏展开解析**：

```cpp
// GET_SHARED_SETTINGS_FUNCTION_PATH 展开后
MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({
    TEXT("ULyraLocalPlayer::GetSharedSettings"),  // 第一步：获取 Shared Settings 对象
    TEXT("ULyraSettingsShared::GetMouseSensitivityX") // 第二步：调用 Getter
}))
```

运行时，`FGameSettingDataSourceDynamic` 通过**反射**依次调用路径上的函数：

```cpp
// 伪代码
UObject* CurrentObject = LocalPlayer;
for (const FString& FunctionName : Path)
{
    UFunction* Function = CurrentObject->FindFunction(*FunctionName);
    CurrentObject->ProcessEvent(Function, &ReturnValue);
    CurrentObject = ReturnValue;
}
```

#### 离散型设置示例：分辨率

```cpp
UGameSettingValueDiscreteDynamic_Resolution* ResolutionSetting = NewObject<UGameSettingValueDiscreteDynamic_Resolution>();
ResolutionSetting->SetDevName(TEXT("Resolution"));
ResolutionSetting->SetDisplayName(LOCTEXT("Resolution_Name", "分辨率"));

// 动态填充选项（从显示器查询支持的分辨率）
ResolutionSetting->SetDynamicOptionSource([]() -> TArray<FGameSettingDataSourceOptionData>
{
    TArray<FGameSettingDataSourceOptionData> Options;
    FScreenResolutionArray Resolutions;
    
    if (RHIGetAvailableResolutions(Resolutions, true))
    {
        for (const FScreenResolutionRHI& Resolution : Resolutions)
        {
            FGameSettingDataSourceOptionData Option;
            Option.Value = FString::Printf(TEXT("%dx%d"), Resolution.Width, Resolution.Height);
            Option.DisplayName = FText::FromString(Option.Value);
            Options.Add(Option);
        }
    }
    
    return Options;
});

ResolutionSetting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetScreenResolution));
ResolutionSetting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetScreenResolution));
```

---

## 🔄 设置同步流程

### 完整生命周期

```
  玩家启动游戏
       │
       ▼
┌─────────────────┐
│ LocalPlayer 初始化│
└────────┬─────────┘
         │
         ▼
┌─────────────────────────┐
│ ULyraSettingsLocal::Get()│ ◄── 加载 GameUserSettings.ini
└────────┬─────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ ULyraSettingsShared::LoadOrCreate()│ ◄── 加载 SaveGame 文件
└────────┬───────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ ULyraGameSettingRegistry::  │
│ OnInitialize()              │ ◄── 注册所有设置项
└────────┬────────────────────┘
         │
         ▼
┌─────────────────┐
│  UI 生成完成     │ ◄── 遍历注册表创建控件
└────────┬─────────┘
         │
         │ 玩家修改设置
         ▼
┌─────────────────────────┐
│ SettingValue::SetValue() │
└────────┬─────────────────┘
         │
         ▼
┌────────────────────────────┐
│ Local/Shared::SetXXX()     │ ◄── 调用 Setter 更新数据
│ ChangeValueAndDirty()      │
└────────┬───────────────────┘
         │
         ▼
┌────────────────────────┐
│ OnSettingChanged.Broadcast()│ ◄── 通知 UI 刷新
└────────┬───────────────────┘
         │
         │ 玩家点击"应用"或"保存"
         ▼
┌─────────────────────────┐
│ Registry::SaveChanges()  │
└────────┬─────────────────┘
         │
    ┌────┴─────┐
    ▼          ▼
┌────────┐ ┌────────────┐
│Local:: │ │Shared::    │
│Apply   │ │SaveSettings│
│Settings│ │()          │
└────────┘ └────────────┘
    │          │
    ▼          ▼
 写入.ini   写入.sav
```

### 核心代码

```cpp
void ULyraGameSettingRegistry::SaveChanges()
{
    if (ULyraLocalPlayer* LocalPlayer = Cast<ULyraLocalPlayer>(OwningLocalPlayer))
    {
        // 1. 保存本地设置
        if (ULyraSettingsLocal* LocalSettings = LocalPlayer->GetLocalSettings())
        {
            LocalSettings->ApplySettings(false); // 不立即保存
            LocalSettings->SaveSettings();       // 写入 .ini
        }

        // 2. 保存共享设置
        if (ULyraSettingsShared* SharedSettings = LocalPlayer->GetSharedSettings())
        {
            SharedSettings->ApplySettings();     // 立即生效
            SharedSettings->SaveSettings();      // 写入 .sav
        }
    }
}

// ULyraSettingsShared::ChangeValueAndDirty 模板函数
template<typename T>
bool ULyraSettingsShared::ChangeValueAndDirty(T& CurrentValue, const T& NewValue)
{
    if (CurrentValue != NewValue)
    {
        CurrentValue = NewValue;
        bIsDirty = true; // 标记脏
        OnSettingChanged.Broadcast(this); // 广播变更事件
        return true;
    }
    return false;
}
```

---

## 🛠️ 实战案例

### 案例 1：添加"启用运动模糊"设置

#### 步骤 1：在 SettingsLocal 中添加字段

```cpp
// LyraSettingsLocal.h
public:
    UFUNCTION() bool IsMotionBlurEnabled() const;
    UFUNCTION() void SetMotionBlurEnabled(bool bEnabled);

private:
    UPROPERTY(Config)
    bool bEnableMotionBlur = true;
```

```cpp
// LyraSettingsLocal.cpp
bool ULyraSettingsLocal::IsMotionBlurEnabled() const
{
    return bEnableMotionBlur;
}

void ULyraSettingsLocal::SetMotionBlurEnabled(bool bEnabled)
{
    if (bEnableMotionBlur != bEnabled)
    {
        bEnableMotionBlur = bEnabled;
        ApplyMotionBlurSetting();
    }
}

void ULyraSettingsLocal::ApplyMotionBlurSetting()
{
    IConsoleVariable* CVar = IConsoleManager::Get().FindConsoleVariable(TEXT("r.MotionBlurQuality"));
    if (CVar)
    {
        CVar->Set(bEnableMotionBlur ? 3 : 0); // 3 = 高质量，0 = 关闭
    }
}
```

#### 步骤 2：在注册表中注册

```cpp
// LyraGameSettingRegistry.cpp
UGameSettingCollection* ULyraGameSettingRegistry::InitializeVideoSettings(ULyraLocalPlayer* InLocalPlayer)
{
    UGameSettingCollection* VideoSettings = NewObject<UGameSettingCollection>();
    // ... 其他设置

    // 创建开关型设置
    UGameSettingValueDiscreteDynamic_Bool* MotionBlurSetting = NewObject<UGameSettingValueDiscreteDynamic_Bool>();
    MotionBlurSetting->SetDevName(TEXT("MotionBlur"));
    MotionBlurSetting->SetDisplayName(LOCTEXT("MotionBlur_Name", "运动模糊"));
    MotionBlurSetting->SetDescriptionRichText(LOCTEXT("MotionBlur_Desc", "启用后可增强运动感，但可能导致画面模糊"));

    // 设置选项
    MotionBlurSetting->SetOptions({
        {TEXT("关闭"), LOCTEXT("Off", "关闭")},
        {TEXT("开启"), LOCTEXT("On", "开启")}
    });

    // 绑定数据源
    MotionBlurSetting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(IsMotionBlurEnabled));
    MotionBlurSetting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetMotionBlurEnabled));

    VideoSettings->AddSetting(MotionBlurSetting);

    return VideoSettings;
}
```

#### 步骤 3：验证

1. **启动游戏** → 打开设置界面
2. **图形选项** → 找到"运动模糊"开关
3. **切换设置** → 观察画面变化
4. **重启游戏** → 验证设置是否持久化

---

### 案例 2：实现"重置为默认设置"

```cpp
// 在 SettingsLocal 中添加
void ULyraSettingsLocal::ResetToDefaults()
{
    // 调用父类方法（重置引擎标准设置）
    Super::SetToDefaults();

    // 重置自定义字段
    bEnableMotionBlur = true;
    DisplayGamma = 2.2f;
    FrameRateLimit_OnBattery = 30.0f;
    FrameRateLimit_InMenu = 60.0f;
    SafeZoneScale = -1.0f;

    // 重置音量
    OverallVolume = 1.0f;
    MusicVolume = 1.0f;
    SoundFXVolume = 1.0f;
    DialogueVolume = 1.0f;
    VoiceChatVolume = 1.0f;

    // 应用默认设置
    ApplySettings(false);
    SaveSettings();
}

// 在 SettingsShared 中添加
void ULyraSettingsShared::ResetToDefaults()
{
    MouseSensitivityX = 1.0;
    MouseSensitivityY = 1.0;
    bInvertVerticalAxis = false;
    bForceFeedbackEnabled = true;
    GamepadLookSensitivityPreset = ELyraGamepadSensitivity::Normal;
    bEnableSubtitles = true;
    ColorBlindMode = EColorBlindMode::Off;

    bIsDirty = true;
    ApplySettings();
    SaveSettings();
}
```

**UI 按钮绑定**：

```cpp
// 在设置界面的蓝图或 C++ 中
void ULyraSettingsUI::OnResetButtonClicked()
{
    if (ULyraLocalPlayer* LocalPlayer = GetOwningLocalPlayer<ULyraLocalPlayer>())
    {
        // 重置本地设置
        LocalPlayer->GetLocalSettings()->ResetToDefaults();

        // 重置共享设置
        LocalPlayer->GetSharedSettings()->ResetToDefaults();

        // 刷新 UI 显示
        ULyraGameSettingRegistry* Registry = ULyraGameSettingRegistry::Get(LocalPlayer);
        Registry->OnSettingsChanged().Broadcast();
    }
}
```

---

### 案例 3：调试设置未保存问题

#### 常见原因

1. **忘记调用 `SaveSettings()`**
2. **权限不足**（某些平台的 SaveGames 目录需要特殊权限）
3. **异步保存回调失败**
4. **脏标记未设置**（`bIsDirty = false`）

#### 调试技巧

**启用详细日志**：

```cpp
// 在 LyraSettingsShared.cpp 中
void ULyraSettingsShared::SaveSettings()
{
    UE_LOG(LogLyraSettings, Verbose, TEXT("SaveSettings() called, bIsDirty=%d"), bIsDirty);

    if (bIsDirty)
    {
        const FString SlotName = FString::Printf(TEXT("LyraSettings_%d"), GetLocalPlayer()->GetControllerId());
        
        UE_LOG(LogLyraSettings, Warning, TEXT("Saving to slot: %s"), *SlotName);

        UGameplayStatics::AsyncSaveGameToSlot(this, SlotName, 0,
            FAsyncSaveGameToSlotDelegate::CreateLambda([this, SlotName](const FString& SaveSlotName, const int32 UserIndex, bool bSuccess)
            {
                if (bSuccess)
                {
                    UE_LOG(LogLyraSettings, Display, TEXT("✅ Settings saved: %s"), *SaveSlotName);
                    bIsDirty = false;
                }
                else
                {
                    UE_LOG(LogLyraSettings, Error, TEXT("❌ Failed to save: %s"), *SaveSlotName);
                }
            }));
    }
    else
    {
        UE_LOG(LogLyraSettings, Verbose, TEXT("No changes to save (bIsDirty=false)"));
    }
}
```

**检查文件是否生成**：

```bash
# Windows
dir Saved\SaveGames\LyraSettings_*.sav

# macOS/Linux
ls Saved/SaveGames/LyraSettings_*.sav
```

**手动触发保存**（控制台命令）：

```cpp
// 注册控制台命令
static FAutoConsoleCommand SaveSettingsCommand(
    TEXT("Lyra.SaveSettings"),
    TEXT("Force save all settings"),
    FConsoleCommandDelegate::CreateLambda([]()
    {
        if (UWorld* World = GEngine->GetWorldContexts()[0].World())
        {
            if (APlayerController* PC = World->GetFirstPlayerController())
            {
                if (ULyraLocalPlayer* LP = Cast<ULyraLocalPlayer>(PC->GetLocalPlayer()))
                {
                    LP->GetLocalSettings()->SaveSettings();
                    LP->GetSharedSettings()->SaveSettings();
                    UE_LOG(LogLyra, Display, TEXT("Settings saved manually"));
                }
            }
        }
    })
);
```

在游戏中按 `` ` `` 打开控制台，输入 `Lyra.SaveSettings` 强制保存。

---

## 🎯 最佳实践

### 1. 设置分层原则

| 类型               | 存储位置             | 适用场景                                   |
| ------------------ | -------------------- | ------------------------------------------ |
| **机器相关**       | `ULyraSettingsLocal` | 分辨率、帧率、显卡质量、音频设备           |
| **用户习惯**       | `ULyraSettingsShared`| 灵敏度、按键绑定、字幕、辅助功能           |
| **游戏状态**       | 自定义 SaveGame      | 解锁进度、成就、角色数据（不属于设置系统） |

### 2. 避免频繁保存

```cpp
// ❌ 错误：每次滑动滑块都保存（卡顿）
void OnSensitivitySliderValueChanged(float NewValue)
{
    SharedSettings->SetMouseSensitivityX(NewValue);
    SharedSettings->SaveSettings(); // 磁盘 I/O 开销大
}

// ✅ 正确：仅在释放滑块或退出界面时保存
void OnSensitivitySliderReleased()
{
    SharedSettings->SetMouseSensitivityX(CachedSliderValue);
    SharedSettings->SaveSettings();
}

void OnSettingsMenuClosed()
{
    ULyraGameSettingRegistry::Get(LocalPlayer)->SaveChanges();
}
```

### 3. 跨平台注意事项

#### 主机平台（PS/Xbox）

- **TRC/XR 要求**：必须在特定时机保存（如返回主菜单）
- **存档大小限制**：单个 SaveGame 不超过 100KB
- **自动云同步**：平台会自动处理，无需额外代码

#### 移动平台（iOS/Android）

```cpp
// 在应用进入后台时保存
void ULyraSettingsLocal::OnAppActivationStateChanged(bool bIsActive)
{
    if (!bIsActive)
    {
        // 立即保存所有设置
        SaveSettings();
        if (ULyraSettingsShared* SharedSettings = GetLocalPlayer()->GetSharedSettings())
        {
            SharedSettings->SaveSettings();
        }
    }
}
```

### 4. 本地化文本

```cpp
// LyraSettingsLocal.cpp
#define LOCTEXT_NAMESPACE "LyraSettings"

UGameSettingValueDiscrete* QualitySetting = NewObject<UGameSettingValueDiscrete>();
QualitySetting->SetDisplayName(LOCTEXT("Quality_Name", "图形质量"));
QualitySetting->SetDescriptionRichText(LOCTEXT("Quality_Desc", 
    "调整整体画质。<span class=\"warning\">高质量可能影响帧率</>"
));

#undef LOCTEXT_NAMESPACE
```

**多语言支持**：

在 `Content/Localization/Game/` 中创建对应语言的 `.po` 文件：

```po
# zh-CN.po
msgid "Quality_Name"
msgstr "图形质量"

msgid "Quality_Desc"
msgstr "调整整体画质。<span class=\"warning\">高质量可能影响帧率</>"
```

---

## 🐞 常见问题

### Q1：设置修改后不生效

**可能原因**：

1. **未调用 `ApplySettings()`**：
   ```cpp
   SharedSettings->SetMouseSensitivityX(2.0);
   SharedSettings->ApplySettings(); // ← 必须调用
   ```

2. **Console Variable 被覆盖**：
   某些 CVar 在 `DefaultEngine.ini` 中被强制锁定
   ```ini
   [SystemSettings]
   r.MotionBlurQuality=0 ; 强制关闭，代码无法修改
   ```
   **解决**：移除配置文件中的锁定项。

3. **平台特定限制**：
   移动端某些设置（如阴影质量）受 Device Profile 限制

### Q2：设置在重启后丢失

**排查步骤**：

1. **检查保存是否成功**：
   ```cpp
   UE_LOG(LogLyraSettings, Display, TEXT("Saved file size: %d bytes"), 
       IFileManager::Get().FileSize(*SaveGameFilePath));
   ```

2. **验证加载路径**：
   确保 `LoadGameFromSlot` 的槽名与 `SaveGameToSlot` 一致

3. **权限问题**（主机平台）：
   需要玩家登录 Platform Account 才能写入存档

### Q3：如何实现"应用"与"取消"

```cpp
class ULyraSettingsUI : public UUserWidget
{
public:
    // 打开设置界面时备份当前值
    void OnSettingsOpened()
    {
        BackupSettings = LocalPlayer->GetSharedSettings()->Clone();
    }

    // 点击"应用"
    void OnApplyClicked()
    {
        ULyraGameSettingRegistry::Get(LocalPlayer)->SaveChanges();
        BackupSettings = nullptr; // 清空备份
    }

    // 点击"取消"
    void OnCancelClicked()
    {
        // 恢复备份
        LocalPlayer->SetSharedSettings(BackupSettings);
        BackupSettings->ApplySettings();
    }

private:
    UPROPERTY()
    ULyraSettingsShared* BackupSettings;
};
```

---

## 📚 总结

Lyra 的设置系统展示了**分层设计、注册表模式、数据绑定**的精妙结合：

1. **Local/Shared 分离**：清晰划分机器相关与用户习惯，支持云同步
2. **注册表集中管理**：新增设置只需注册，UI 自动生成
3. **动态数据源**：反射路径解耦 UI 与业务逻辑，支持热重载
4. **实时通知机制**：`OnSettingChanged` 事件驱动 UI 刷新

在实际项目中，你可以：

- **扩展辅助功能**：添加高对比度模式、字体大小调节
- **自定义预设方案**：保存多套配置（如"竞技模式"、"画质优先"）
- **云同步增强**：集成 EOS/Steam Cloud Save 实现跨平台同步
- **A/B 测试**：通过设置系统动态调整游戏参数

---

## 🔗 延伸阅读

- **官方文档**：[Game User Settings (UE5)](https://docs.unrealengine.com/5.0/en-US/API/Runtime/Engine/GameFramework/UGameUserSettings/)
- **Common UI**：第 14 章 - 了解设置 UI 的交互框架
- **Enhanced Input**：第 09 章 - 深入按键绑定的实现
- **SaveGame System**：[Unreal Engine SaveGame Documentation](https://docs.unrealengine.com/5.0/en-US/saving-and-loading-your-game-in-unreal-engine/)

---

**下一章预告**：[17. 交互系统：WorldInteraction 组件](./17-interaction-system.md) - 探索 Lyra 如何实现拾取、对话、交互提示的统一框架。

---

*本章完整代码示例：[GitHub - Lyra Tutorial Examples](https://github.com/example/lyra-tutorial)*  
*有疑问？加入 [Discord 社区](https://discord.gg/lyra-tutorial) 讨论*
