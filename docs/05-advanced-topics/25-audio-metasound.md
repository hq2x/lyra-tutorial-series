# 音频系统与 MetaSound

## 概述

Lyra 展示了 Unreal Engine 5 现代化音频系统的最佳实践，包括 **Sound Control Bus**（声音控制总线）、**Audio Modulation**（音频调制）、**MetaSound**（元音效）以及 **Submix Effect Chain**（子混音效果链）等先进技术。本文将深入解析 Lyra 的音频架构，并提供完整的实战案例。

## 核心概念

### 1. Sound Control Bus（声音控制总线）

Sound Control Bus 是 UE5 音频调制系统的核心组件，允许动态控制音频参数（如音量、音高等）。

**优势：**
- 运行时动态调整音频参数
- 统一管理不同类别音频（音乐、音效、对话等）
- 与游戏设置系统无缝集成

### 2. Sound Control Bus Mix（总线混音）

Mix 是 Control Bus 的集合，可以同时控制多个总线的参数值。

### 3. Submix（子混音）

音频渲染管线中的中间节点，可以应用效果链处理。

### 4. MetaSound

UE5 全新的程序化音频生成系统，使用节点图形式创建复杂音频逻辑。

## Lyra 音频架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    LyraAudioSettings                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ • DefaultControlBusMix                                 │  │
│  │ • LoadingScreenControlBusMix                           │  │
│  │ • UserSettingsControlBusMix                            │  │
│  │ • OverallVolumeControlBus                              │  │
│  │ • MusicVolumeControlBus                                │  │
│  │ • SoundFXVolumeControlBus                              │  │
│  │ • DialogueVolumeControlBus                             │  │
│  │ • VoiceChatVolumeControlBus                            │  │
│  │ • HDRAudioSubmixEffectChain                            │  │
│  │ • LDRAudioSubmixEffectChain                            │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│            LyraAudioMixEffectsSubsystem                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Initialize()      - 加载 Control Bus 资源              │  │
│  │ OnWorldBeginPlay()- 激活 Bus Mix                       │  │
│  │ ApplyDynamicRangeEffectsChains() - 应用 HDR/LDR 链     │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   Audio Mixer Pipeline                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐   │
│  │ Sound Source │→ │   Submix     │→ │ Master Submix   │   │
│  │   (Cue)      │  │ (with FX)    │  │                 │   │
│  └──────────────┘  └──────────────┘  └─────────────────┘   │
│         ↑                                                    │
│    Control Bus                                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    用户设置界面                               │
│  (LyraGameSettingRegistry_Audio)                             │
└─────────────────────────────────────────────────────────────┘
```

## 核心类解析

### 1. ULyraAudioSettings

音频系统的配置中心，定义在 `LyraAudioSettings.h`。

```cpp
// LyraAudioSettings.h
UCLASS(MinimalAPI, config = Game, defaultconfig, meta = (DisplayName = "LyraAudioSettings"))
class ULyraAudioSettings : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    /** 默认基础控制总线混音 */
    UPROPERTY(config, EditAnywhere, Category = MixSettings, 
              meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBusMix"))
    FSoftObjectPath DefaultControlBusMix;

    /** 加载屏幕控制总线混音 - 在加载屏幕期间覆盖背景音频事件 */
    UPROPERTY(config, EditAnywhere, Category = MixSettings, 
              meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBusMix"))
    FSoftObjectPath LoadingScreenControlBusMix;

    /** 用户设置控制总线混音 */
    UPROPERTY(config, EditAnywhere, Category = UserMixSettings, 
              meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBusMix"))
    FSoftObjectPath UserSettingsControlBusMix;

    /** 总音量控制总线 */
    UPROPERTY(config, EditAnywhere, Category = UserMixSettings, 
              meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
    FSoftObjectPath OverallVolumeControlBus;

    /** 音乐音量控制总线 */
    UPROPERTY(config, EditAnywhere, Category = UserMixSettings, 
              meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
    FSoftObjectPath MusicVolumeControlBus;

    /** 音效音量控制总线 */
    UPROPERTY(config, EditAnywhere, Category = UserMixSettings, 
              meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
    FSoftObjectPath SoundFXVolumeControlBus;

    /** 对话音量控制总线 */
    UPROPERTY(config, EditAnywhere, Category = UserMixSettings, 
              meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
    FSoftObjectPath DialogueVolumeControlBus;

    /** 语音聊天音量控制总线 */
    UPROPERTY(config, EditAnywhere, Category = UserMixSettings, 
              meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
    FSoftObjectPath VoiceChatVolumeControlBus;

    /** HDR 音频的 Submix 效果链 */
    UPROPERTY(config, EditAnywhere, Category = EffectSettings)
    TArray<FLyraSubmixEffectChainMap> HDRAudioSubmixEffectChain;
    
    /** LDR 音频的 Submix 效果链 */
    UPROPERTY(config, EditAnywhere, Category = EffectSettings)
    TArray<FLyraSubmixEffectChainMap> LDRAudioSubmixEffectChain;
};

/** Submix 效果链映射 */
USTRUCT()
struct FLyraSubmixEffectChainMap
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, meta = (AllowedClasses = "/Script/Engine.SoundSubmix"))
    TSoftObjectPtr<USoundSubmix> Submix = nullptr;

    UPROPERTY(EditAnywhere, meta = (AllowedClasses = "/Script/Engine.SoundEffectSubmixPreset"))
    TArray<TSoftObjectPtr<USoundEffectSubmixPreset>> SubmixEffectChain;
};
```

**关键设计：**
- **软引用**：使用 `FSoftObjectPath` 和 `TSoftObjectPtr` 避免硬引用，支持按需加载
- **分类管理**：将音频分为 5 个类别（Overall、Music、SoundFX、Dialogue、VoiceChat）
- **动态范围支持**：分别为 HDR 和 LDR 模式配置不同的效果链

### 2. ULyraAudioMixEffectsSubsystem

音频系统的运行时管理器，定义在 `LyraAudioMixEffectsSubsystem.h`。

```cpp
// LyraAudioMixEffectsSubsystem.h
UCLASS(MinimalAPI)
class ULyraAudioMixEffectsSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    // USubsystem 接口
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
    virtual bool ShouldCreateSubsystem(UObject* Outer) const override;
    virtual void PostInitialize() override;
    virtual void OnWorldBeginPlay(UWorld& InWorld) override;

    /** 应用 HDR/LDR 音频效果链 */
    void ApplyDynamicRangeEffectsChains(bool bHDRAudio);
    
protected:
    void OnLoadingScreenStatusChanged(bool bShowingLoadingScreen);
    void ApplyOrRemoveLoadingScreenMix(bool bWantsLoadingScreenMix);
    bool DoesSupportWorldType(const EWorldType::Type WorldType) const override;

    // 默认 Sound Control Bus Mix
    UPROPERTY(Transient)
    TObjectPtr<USoundControlBusMix> DefaultBaseMix = nullptr;

    // 加载屏幕 Sound Control Bus Mix
    UPROPERTY(Transient)
    TObjectPtr<USoundControlBusMix> LoadingScreenMix = nullptr;

    // 用户 Sound Control Bus Mix
    UPROPERTY(Transient)
    TObjectPtr<USoundControlBusMix> UserMix = nullptr;

    // 各类别的 Sound Control Bus
    UPROPERTY(Transient)
    TObjectPtr<USoundControlBus> OverallControlBus = nullptr;

    UPROPERTY(Transient)
    TObjectPtr<USoundControlBus> MusicControlBus = nullptr;

    UPROPERTY(Transient)
    TObjectPtr<USoundControlBus> SoundFXControlBus = nullptr;

    UPROPERTY(Transient)
    TObjectPtr<USoundControlBus> DialogueControlBus = nullptr;

    UPROPERTY(Transient)
    TObjectPtr<USoundControlBus> VoiceChatControlBus = nullptr;

    // HDR/LDR Submix 效果链
    UPROPERTY(Transient)
    TArray<FLyraAudioSubmixEffectsChain> HDRSubmixEffectChain;

    UPROPERTY(Transient)
    TArray<FLyraAudioSubmixEffectsChain> LDRSubmixEffectChain;

    bool bAppliedLoadingScreenMix = false;
};

/** 音频 Submix 效果链结构 */
USTRUCT()
struct FLyraAudioSubmixEffectsChain
{
    GENERATED_BODY()

    // 应用效果链的 Submix
    UPROPERTY(Transient)
    TObjectPtr<USoundSubmix> Submix = nullptr;

    // Submix 效果链（按数组索引顺序处理）
    UPROPERTY(Transient)
    TArray<TObjectPtr<USoundEffectSubmixPreset>> SubmixEffectChain;
};
```

### 3. 初始化流程

```cpp
// LyraAudioMixEffectsSubsystem.cpp
void ULyraAudioMixEffectsSubsystem::PostInitialize()
{
    Super::PostInitialize();

    // 从设置加载资源
    if (const ULyraAudioSettings* LyraAudioSettings = GetDefault<ULyraAudioSettings>())
    {
        // 加载默认 Bus Mix
        if (UObject* ObjPath = LyraAudioSettings->DefaultControlBusMix.TryLoad())
        {
            if (USoundControlBusMix* SoundControlBusMix = Cast<USoundControlBusMix>(ObjPath))
            {
                DefaultBaseMix = SoundControlBusMix;
            }
        }

        // 加载加载屏幕 Bus Mix
        if (UObject* ObjPath = LyraAudioSettings->LoadingScreenControlBusMix.TryLoad())
        {
            LoadingScreenMix = Cast<USoundControlBusMix>(ObjPath);
        }

        // 加载用户设置 Bus Mix
        if (UObject* ObjPath = LyraAudioSettings->UserSettingsControlBusMix.TryLoad())
        {
            UserMix = Cast<USoundControlBusMix>(ObjPath);
        }

        // 加载各个类别的 Control Bus
        if (UObject* ObjPath = LyraAudioSettings->OverallVolumeControlBus.TryLoad())
        {
            OverallControlBus = Cast<USoundControlBus>(ObjPath);
        }

        if (UObject* ObjPath = LyraAudioSettings->MusicVolumeControlBus.TryLoad())
        {
            MusicControlBus = Cast<USoundControlBus>(ObjPath);
        }

        // ... 其他 Control Bus 加载

        // 加载 HDR Submix 效果链
        for (const FLyraSubmixEffectChainMap& SoftSubmixEffectChain : 
             LyraAudioSettings->HDRAudioSubmixEffectChain)
        {
            FLyraAudioSubmixEffectsChain NewEffectChain;

            if (UObject* SubmixObjPath = SoftSubmixEffectChain.Submix.LoadSynchronous())
            {
                if (USoundSubmix* Submix = Cast<USoundSubmix>(SubmixObjPath))
                {
                    NewEffectChain.Submix = Submix;
                    TArray<USoundEffectSubmixPreset*> NewPresetChain;

                    for (const TSoftObjectPtr<USoundEffectSubmixPreset>& SoftEffect : 
                         SoftSubmixEffectChain.SubmixEffectChain)
                    {
                        if (UObject* EffectObjPath = SoftEffect.LoadSynchronous())
                        {
                            if (USoundEffectSubmixPreset* SubmixPreset = 
                                Cast<USoundEffectSubmixPreset>(EffectObjPath))
                            {
                                NewPresetChain.Add(SubmixPreset);
                            }
                        }
                    }

                    NewEffectChain.SubmixEffectChain.Append(NewPresetChain);
                }
            }

            HDRSubmixEffectChain.Add(NewEffectChain);
        }

        // 加载 LDR Submix 效果链（类似流程）
        // ...
    }

    // 注册加载屏幕管理器回调
    if (ULoadingScreenManager* LoadingScreenManager = 
        UGameInstance::GetSubsystem<ULoadingScreenManager>(GetWorld()->GetGameInstance()))
    {
        LoadingScreenManager->OnLoadingScreenVisibilityChangedDelegate()
            .AddUObject(this, &ThisClass::OnLoadingScreenStatusChanged);
        ApplyOrRemoveLoadingScreenMix(LoadingScreenManager->GetLoadingScreenDisplayStatus());
    }
}
```

### 4. 激活音频混音

```cpp
void ULyraAudioMixEffectsSubsystem::OnWorldBeginPlay(UWorld& InWorld)
{
    Super::OnWorldBeginPlay(InWorld);

    if (const UWorld* World = InWorld.GetWorld())
    {
        // 激活默认基础混音
        if (DefaultBaseMix)
        {
            UAudioModulationStatics::ActivateBusMix(World, DefaultBaseMix);
        }

        // 检索用户设置
        if (const ULyraSettingsLocal* LyraSettingsLocal = GetDefault<ULyraSettingsLocal>())
        {
            // 激活用户混音
            if (UserMix)
            {
                UAudioModulationStatics::ActivateBusMix(World, UserMix);

                if (OverallControlBus && MusicControlBus && SoundFXControlBus && 
                    DialogueControlBus && VoiceChatControlBus)
                {
                    // 为每个 Control Bus 创建 Mix Stage
                    const FSoundControlBusMixStage OverallControlBusMixStage = 
                        UAudioModulationStatics::CreateBusMixStage(
                            World, OverallControlBus, LyraSettingsLocal->GetOverallVolume());
                    
                    const FSoundControlBusMixStage MusicControlBusMixStage = 
                        UAudioModulationStatics::CreateBusMixStage(
                            World, MusicControlBus, LyraSettingsLocal->GetMusicVolume());
                    
                    const FSoundControlBusMixStage SoundFXControlBusMixStage = 
                        UAudioModulationStatics::CreateBusMixStage(
                            World, SoundFXControlBus, LyraSettingsLocal->GetSoundFXVolume());
                    
                    const FSoundControlBusMixStage DialogueControlBusMixStage = 
                        UAudioModulationStatics::CreateBusMixStage(
                            World, DialogueControlBus, LyraSettingsLocal->GetDialogueVolume());
                    
                    const FSoundControlBusMixStage VoiceChatControlBusMixStage = 
                        UAudioModulationStatics::CreateBusMixStage(
                            World, VoiceChatControlBus, LyraSettingsLocal->GetVoiceChatVolume());

                    // 将所有 Mix Stage 添加到数组
                    TArray<FSoundControlBusMixStage> ControlBusMixStageArray;
                    ControlBusMixStageArray.Add(OverallControlBusMixStage);
                    ControlBusMixStageArray.Add(MusicControlBusMixStage);
                    ControlBusMixStageArray.Add(SoundFXControlBusMixStage);
                    ControlBusMixStageArray.Add(DialogueControlBusMixStage);
                    ControlBusMixStageArray.Add(VoiceChatControlBusMixStage);

                    // 更新 User Mix
                    UAudioModulationStatics::UpdateMix(World, UserMix, ControlBusMixStageArray);
                }
            }

            // 应用 HDR/LDR 音频效果链
            ApplyDynamicRangeEffectsChains(LyraSettingsLocal->IsHDRAudioModeEnabled());
        }
    }
}
```

### 5. 动态范围效果链切换

```cpp
void ULyraAudioMixEffectsSubsystem::ApplyDynamicRangeEffectsChains(bool bHDRAudio)
{
    TArray<FLyraAudioSubmixEffectsChain> AudioSubmixEffectsChainToApply;
    TArray<FLyraAudioSubmixEffectsChain> AudioSubmixEffectsChainToClear;

    // 根据 HDR/LDR 选择要应用和清除的效果链
    if (bHDRAudio)
    {
        AudioSubmixEffectsChainToApply.Append(HDRSubmixEffectChain);
        AudioSubmixEffectsChainToClear.Append(LDRSubmixEffectChain);
    }
    else
    {
        AudioSubmixEffectsChainToApply.Append(LDRSubmixEffectChain);
        AudioSubmixEffectsChainToClear.Append(HDRSubmixEffectChain);
    }

    // 收集需要清除的 Submix（排除将被新效果链覆盖的）
    TArray<USoundSubmix*> SubmixesLeftToClear;

    for (const FLyraAudioSubmixEffectsChain& EffectChainToClear : AudioSubmixEffectsChainToClear)
    {
        bool bAddToList = true;

        for (const FLyraAudioSubmixEffectsChain& SubmixEffectChain : AudioSubmixEffectsChainToApply)
        {
            if (SubmixEffectChain.Submix == EffectChainToClear.Submix)
            {
                bAddToList = false;
                break;
            }
        }

        if (bAddToList)
        {
            SubmixesLeftToClear.Add(EffectChainToClear.Submix);
        }
    }

    // 覆盖 Submix 效果链
    for (const FLyraAudioSubmixEffectsChain& SubmixEffectChain : AudioSubmixEffectsChainToApply)
    {
        if (SubmixEffectChain.Submix)
        {
            UAudioMixerBlueprintLibrary::SetSubmixEffectChainOverride(
                GetWorld(), SubmixEffectChain.Submix, SubmixEffectChain.SubmixEffectChain, 0.1f);
        }
    }

    // 清除剩余的 Submix
    for (USoundSubmix* Submix : SubmixesLeftToClear)
    {
        UAudioMixerBlueprintLibrary::ClearSubmixEffectChainOverride(GetWorld(), Submix, 0.1f);
    }
}
```

## Sound Control Bus 系统详解

### Control Bus 工作原理

Control Bus 是一个参数调制器，可以控制音频的各种属性（音量、音高、低通滤波等）。

```
Sound Source → Control Bus (Volume Modulation) → Submix → Output
```

### 创建 Control Bus

在编辑器中：
1. **Content Browser** → 右键 → **Audio** → **Sound Control Bus**
2. 配置参数：
   - **Parameter**: 选择要控制的参数（如 Volume）
   - **Min/Max Value**: 参数的最小/最大值
   - **Default Value**: 默认值

### 创建 Control Bus Mix

1. **Content Browser** → 右键 → **Audio** → **Sound Control Bus Mix**
2. 添加 **Bus Stages**：
   - 选择 Control Bus
   - 设置目标值

### 在 Sound Cue 中使用 Control Bus

```cpp
// 在 Sound Cue 中添加 Modulator 节点
// 将 Control Bus 资源连接到 Modulator
```

在蓝图中动态调整：

```cpp
// 更新 Control Bus Mix
UAudioModulationStatics::UpdateMix(
    GetWorld(), 
    MixToUpdate, 
    ControlBusMixStageArray, 
    FadeTime
);

// 激活/停用 Mix
UAudioModulationStatics::ActivateBusMix(GetWorld(), Mix);
UAudioModulationStatics::DeactivateBusMix(GetWorld(), Mix);
```

## Audio Mixer 与 Submix

### Submix 层级结构

```
Master Submix
    ├── Music Submix
    ├── SFX Submix
    │   ├── Weapon Submix
    │   └── Footstep Submix
    ├── Dialogue Submix
    └── VoiceChat Submix
```

### Submix Effect Chain

Submix 可以应用一系列音频效果：

```cpp
// 创建效果链
TArray<USoundEffectSubmixPreset*> EffectChain;
EffectChain.Add(CompressorPreset);
EffectChain.Add(EQPreset);
EffectChain.Add(ReverbPreset);

// 应用到 Submix
UAudioMixerBlueprintLibrary::SetSubmixEffectChainOverride(
    World, 
    MySubmix, 
    EffectChain, 
    FadeTime
);
```

### 常见 Submix 效果

1. **Compressor**（压缩器）- 动态范围压缩
2. **EQ**（均衡器）- 频率平衡
3. **Reverb**（混响）- 空间感
4. **Delay**（延迟）- 回声效果
5. **Distortion**（失真）- 音色变化

## MetaSound 音频合成

MetaSound 是 UE5 的下一代音频引擎，提供完全程序化的音频生成和处理能力。

### MetaSound 优势

- **节点化编程**：可视化音频逻辑
- **实时参数调制**：Gameplay 变量直接驱动音频
- **高性能**：原生 C++ 实现
- **可扩展**：自定义节点

### 创建 MetaSound Source

1. **Content Browser** → 右键 → **Sounds** → **MetaSound Source**
2. 打开 MetaSound 编辑器
3. 构建音频图：

```
[Inputs: Play, Pitch] 
    → [Oscillator] 
    → [ADSR Envelope] 
    → [Filter] 
    → [Output]
```

### MetaSound 基本节点

#### 1. 音频生成器
- **Wave Player**: 播放音频文件
- **Oscillator**: 正弦波/锯齿波/方波生成器
- **Noise Generator**: 白噪声/粉噪声

#### 2. 音频处理
- **Filter**: 低通/高通/带通滤波
- **Delay**: 延迟效果
- **Compressor**: 动态压缩
- **Gain**: 音量调整

#### 3. 包络和调制
- **ADSR Envelope**: 攻击-衰减-延音-释放包络
- **LFO**: 低频振荡器

#### 4. 逻辑和控制
- **Trigger**: 触发事件
- **Branch**: 条件分支
- **Math**: 数学运算

### 在代码中触发 MetaSound

```cpp
// 播放 MetaSound
UAudioComponent* AudioComponent = UGameplayStatics::SpawnSound2D(
    this, 
    MetaSoundSource, 
    VolumeMultiplier, 
    PitchMultiplier
);

// 设置参数
if (AudioComponent)
{
    AudioComponent->SetFloatParameter(FName("Pitch"), 2.0f);
    AudioComponent->SetBoolParameter(FName("EnableReverb"), true);
}
```

## 空间音频与衰减

### Sound Attenuation（声音衰减）

控制 3D 音频的空间特性。

```cpp
// 创建 Attenuation Settings
USoundAttenuation* AttenuationSettings = NewObject<USoundAttenuation>();

// 配置衰减
FAttenuationSettings& Settings = AttenuationSettings->Attenuation;
Settings.bAttenuate = true;
Settings.AttenuationShape = EAttenuationShape::Sphere;
Settings.FalloffDistance = 3000.0f;

// 距离算法
Settings.DistanceAlgorithm = EAttenuationDistanceModel::Linear;
// 或 Logarithmic, Inverse, LogReverse, NaturalSound

// 空间化
Settings.bSpatialize = true;
Settings.SpatializationAlgorithm = ESpatializationAlgorithm::SPATIALIZATION_HRTF;

// 空气吸收
Settings.bAttenuateWithLPF = true;
Settings.LPFRadiusMin = 1000.0f;
Settings.LPFRadiusMax = 3000.0f;
```

### 在代码中播放 3D 音效

```cpp
// 在世界空间位置播放
UGameplayStatics::PlaySoundAtLocation(
    World,
    SoundCue,
    Location,
    Rotation,
    VolumeMultiplier,
    PitchMultiplier,
    StartTime,
    AttenuationSettings
);

// 附加到 Actor
UAudioComponent* AudioComp = UGameplayStatics::SpawnSoundAttached(
    SoundCue,
    AttachToComponent,
    AttachPointName,
    Location,
    Rotation,
    EAttachLocation::KeepRelativeOffset,
    false,  // bStopWhenAttachedToDestroyed
    VolumeMultiplier,
    PitchMultiplier,
    StartTime,
    AttenuationSettings
);
```

### Binaural Spatialization（双声道空间化）

Lyra 支持 HRTF（Head-Related Transfer Function）头部相关传递函数，提供精确的 3D 音频定位。

```cpp
// 在 LyraSettingsLocal 中启用
void ULyraSettingsLocal::SetHeadphoneModeEnabled(bool bEnabled)
{
    bDesiredHeadphoneMode = bEnabled;
    
    if (UHRTFSpatializationSettings* HRTFSettings = 
        GetMutableDefault<UHRTFSpatializationSettings>())
    {
        HRTFSettings->SetHRTFEnabled(bEnabled);
    }
}
```

## 与 Gameplay Cue 集成

### Gameplay Cue 音频流程

```
Gameplay Event 
    → Gameplay Cue Notify 
    → Context Effects System 
    → Audio Component
```

### 在 Gameplay Cue 中播放音效

```cpp
// UGameplayCueNotify_Burst 示例
UCLASS()
class UGC_WeaponFire : public UGameplayCueNotify_Burst
{
    GENERATED_BODY()

public:
    // 武器开火音效
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Audio")
    USoundBase* FireSound;

    // 音量衰减
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Audio")
    USoundAttenuation* AttenuationSettings;

    virtual void OnBurst_Implementation(
        AActor* MyTarget,
        const FGameplayCueParameters& Parameters,
        const TArray<UParticleSystemComponent*>& ParticleComponents,
        const TArray<UAudioComponent*>& AudioComponents,
        UCameraShakeBase* BurstCameraShakeInstance,
        ADecalActor* BurstDecalInstance) override
    {
        if (FireSound && MyTarget)
        {
            UGameplayStatics::SpawnSoundAttached(
                FireSound,
                MyTarget->GetRootComponent(),
                NAME_None,
                FVector::ZeroVector,
                FRotator::ZeroRotator,
                EAttachLocation::SnapToTarget,
                false,
                1.0f,
                1.0f,
                0.0f,
                AttenuationSettings
            );
        }
    }
};
```

### Context Effects 音频系统

Lyra 的 Context Effects 系统根据物理材质自动选择音效。

```cpp
// LyraContextEffectComponent.cpp 片段
void ULyraContextEffectComponent::AnimMotionEffect_Implementation(
    const FName Bone,
    const FGameplayTag MotionEffect,
    USceneComponent* StaticMeshComponent,
    const FVector LocationOffset,
    const FRotator RotationOffset,
    const UAnimSequenceBase* AnimationSequence,
    const bool bHitSuccess,
    const FHitResult HitResult,
    FGameplayTagContainer Contexts,
    FVector VFXScale,
    float AudioVolume,
    float AudioPitch)
{
    TArray<UAudioComponent*> AudioComponentsToAdd;
    FGameplayTagContainer TotalContexts;

    // 聚合上下文
    TotalContexts.AppendTags(Contexts);
    TotalContexts.AppendTags(CurrentContexts);

    // 物理表面类型转换为上下文
    if (bConvertPhysicalSurfaceToContext)
    {
        TWeakObjectPtr<UPhysicalMaterial> PhysicalSurfaceTypePtr = HitResult.PhysMaterial;

        if (PhysicalSurfaceTypePtr.IsValid())
        {
            TEnumAsByte<EPhysicalSurface> PhysicalSurfaceType = 
                PhysicalSurfaceTypePtr->SurfaceType;

            if (const ULyraContextEffectsSettings* Settings = 
                GetDefault<ULyraContextEffectsSettings>())
            {
                if (const FGameplayTag* SurfaceContextPtr = 
                    Settings->SurfaceTypeToContextMap.Find(PhysicalSurfaceType))
                {
                    TotalContexts.AddTag(*SurfaceContextPtr);
                }
            }
        }
    }

    // 使用 Context 查找并播放音效
    // ...
}
```

## 音频设置和用户控制

### 设置界面集成

Lyra 在 `LyraGameSettingRegistry_Audio.cpp` 中定义了完整的音频设置界面。

```cpp
UGameSettingCollection* ULyraGameSettingRegistry::InitializeAudioSettings(
    ULyraLocalPlayer* InLocalPlayer)
{
    UGameSettingCollection* Screen = NewObject<UGameSettingCollection>();
    Screen->SetDevName(TEXT("AudioCollection"));
    Screen->SetDisplayName(LOCTEXT("AudioCollection_Name", "Audio"));
    Screen->Initialize(InLocalPlayer);

    // 音量设置
    {
        UGameSettingCollection* Volume = NewObject<UGameSettingCollection>();
        Volume->SetDevName(TEXT("VolumeCollection"));
        Volume->SetDisplayName(LOCTEXT("VolumeCollection_Name", "Volume"));
        Screen->AddSetting(Volume);

        // Overall Volume
        {
            UGameSettingValueScalarDynamic* Setting = 
                NewObject<UGameSettingValueScalarDynamic>();
            Setting->SetDevName(TEXT("OverallVolume"));
            Setting->SetDisplayName(LOCTEXT("OverallVolume_Name", "Overall"));
            Setting->SetDescriptionRichText(
                LOCTEXT("OverallVolume_Description", "Adjusts the volume of everything."));

            Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetOverallVolume));
            Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetOverallVolume));
            Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->GetOverallVolume());
            Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::ZeroToOnePercent);

            Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

            Volume->AddSetting(Setting);
        }

        // 类似地添加 Music、SoundFX、Dialogue、VoiceChat 设置
        // ...
    }

    // 音频特性设置
    {
        // 3D Headphones
        {
            UGameSettingValueDiscreteDynamic_Bool* Setting = 
                NewObject<UGameSettingValueDiscreteDynamic_Bool>();
            Setting->SetDevName(TEXT("HeadphoneMode"));
            Setting->SetDisplayName(LOCTEXT("HeadphoneMode_Name", "3D Headphones"));
            Setting->SetDescriptionRichText(
                LOCTEXT("HeadphoneMode_Description", 
                "Enable binaural audio. Provides 3D audio spatialization..."));

            Setting->SetDynamicGetter(
                GET_LOCAL_SETTINGS_FUNCTION_PATH(bDesiredHeadphoneMode));
            Setting->SetDynamicSetter(
                GET_LOCAL_SETTINGS_FUNCTION_PATH(bDesiredHeadphoneMode));
            Setting->SetDefaultValue(
                GetDefault<ULyraSettingsLocal>()->IsHeadphoneModeEnabled());

            Sound->AddSetting(Setting);
        }

        // HDR Audio
        {
            UGameSettingValueDiscreteDynamic_Bool* Setting = 
                NewObject<UGameSettingValueDiscreteDynamic_Bool>();
            Setting->SetDevName(TEXT("HDRAudioMode"));
            Setting->SetDisplayName(LOCTEXT("HDRAudioMode_Name", 
                "High Dynamic Range Audio"));
            Setting->SetDescriptionRichText(
                LOCTEXT("HDRAudioMode_Description", 
                "Enable high dynamic range audio..."));

            Setting->SetDynamicGetter(
                GET_LOCAL_SETTINGS_FUNCTION_PATH(bUseHDRAudioMode));
            Setting->SetDynamicSetter(
                GET_LOCAL_SETTINGS_FUNCTION_PATH(SetHDRAudioModeEnabled));
            Setting->SetDefaultValue(
                GetDefault<ULyraSettingsLocal>()->IsHDRAudioModeEnabled());

            Sound->AddSetting(Setting);
        }
    }

    return Screen;
}
```

### 设置值的应用

```cpp
// LyraSettingsLocal.cpp
void ULyraSettingsLocal::SetOverallVolume(float InVolume)
{
    OverallVolume = InVolume;
    
    // 更新 Control Bus
    if (UWorld* World = GetWorld())
    {
        if (ULyraAudioMixEffectsSubsystem* AudioSubsystem = 
            World->GetSubsystem<ULyraAudioMixEffectsSubsystem>())
        {
            if (USoundControlBus* OverallBus = AudioSubsystem->GetOverallControlBus())
            {
                FSoundControlBusMixStage MixStage = 
                    UAudioModulationStatics::CreateBusMixStage(
                        World, OverallBus, InVolume);

                TArray<FSoundControlBusMixStage> Stages;
                Stages.Add(MixStage);

                UAudioModulationStatics::UpdateMix(
                    World, AudioSubsystem->GetUserMix(), Stages);
            }
        }
    }
}
```

## 实战案例 1：武器音效系统

现在让我们实现一个完整的武器音效系统，包括开火、换弹和空枪音效。

### 1. 武器音效组件

```cpp
// WeaponAudioComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "Sound/SoundBase.h"
#include "WeaponAudioComponent.generated.h"

class UAudioComponent;
class USoundAttenuation;
class USoundControlBus;

/**
 * 武器音效组件
 * 管理武器的所有音效（开火、换弹、空枪等）
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class MYPROJECT_API UWeaponAudioComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UWeaponAudioComponent();

protected:
    virtual void BeginPlay() override;

public:
    /** 播放开火音效 */
    UFUNCTION(BlueprintCallable, Category = "Weapon Audio")
    void PlayFireSound(float Pitch = 1.0f);

    /** 播放换弹音效 */
    UFUNCTION(BlueprintCallable, Category = "Weapon Audio")
    void PlayReloadSound();

    /** 播放空枪音效 */
    UFUNCTION(BlueprintCallable, Category = "Weapon Audio")
    void PlayDryFireSound();

    /** 停止所有武器音效 */
    UFUNCTION(BlueprintCallable, Category = "Weapon Audio")
    void StopAllSounds(float FadeOutDuration = 0.2f);

protected:
    // 音效资源
    UPROPERTY(EditDefaultsOnly, Category = "Audio|Sounds")
    USoundBase* FireSound;

    UPROPERTY(EditDefaultsOnly, Category = "Audio|Sounds")
    USoundBase* ReloadSound;

    UPROPERTY(EditDefaultsOnly, Category = "Audio|Sounds")
    USoundBase* DryFireSound;

    UPROPERTY(EditDefaultsOnly, Category = "Audio|Sounds")
    USoundBase* FireTailSound;  // 开火尾音（远处回声）

    // 衰减设置
    UPROPERTY(EditDefaultsOnly, Category = "Audio|Attenuation")
    USoundAttenuation* FireAttenuation;

    UPROPERTY(EditDefaultsOnly, Category = "Audio|Attenuation")
    USoundAttenuation* ReloadAttenuation;

    // Control Bus（用于分类控制）
    UPROPERTY(EditDefaultsOnly, Category = "Audio|Control Bus")
    USoundControlBus* WeaponControlBus;

    // 音频组件池
    UPROPERTY(Transient)
    TArray<UAudioComponent*> AudioComponentPool;

    // 当前活跃的音频组件
    UPROPERTY(Transient)
    UAudioComponent* CurrentFireComponent;

private:
    /** 从池中获取音频组件 */
    UAudioComponent* GetPooledAudioComponent();

    /** 回收音频组件到池 */
    void ReturnToPool(UAudioComponent* Component);
};
```

### 2. 武器音效组件实现

```cpp
// WeaponAudioComponent.cpp
#include "WeaponAudioComponent.h"
#include "Components/AudioComponent.h"
#include "Kismet/GameplayStatics.h"
#include "Sound/SoundAttenuation.h"
#include "AudioModulationStatics.h"

UWeaponAudioComponent::UWeaponAudioComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

void UWeaponAudioComponent::BeginPlay()
{
    Super::BeginPlay();

    // 预创建音频组件池（避免运行时创建开销）
    for (int32 i = 0; i < 3; ++i)
    {
        UAudioComponent* AudioComp = NewObject<UAudioComponent>(this);
        AudioComp->bAutoDestroy = false;
        AudioComp->bStopWhenOwnerDestroyed = true;
        AudioComp->RegisterComponent();
        AudioComponentPool.Add(AudioComp);
    }
}

void UWeaponAudioComponent::PlayFireSound(float Pitch)
{
    if (!FireSound)
    {
        return;
    }

    AActor* Owner = GetOwner();
    if (!Owner)
    {
        return;
    }

    // 获取音频组件
    CurrentFireComponent = GetPooledAudioComponent();
    if (!CurrentFireComponent)
    {
        return;
    }

    // 配置音频组件
    CurrentFireComponent->SetSound(FireSound);
    CurrentFireComponent->SetPitchMultiplier(Pitch);
    CurrentFireComponent->SetVolumeMultiplier(1.0f);
    
    // 应用衰减
    if (FireAttenuation)
    {
        CurrentFireComponent->AttenuationSettings = FireAttenuation;
    }

    // 应用 Control Bus
    if (WeaponControlBus)
    {
        // 使用 Audio Modulation 系统
        CurrentFireComponent->SetModulationRouting({WeaponControlBus});
    }

    // 附加到武器枪口位置
    USceneComponent* MuzzleComponent = Owner->FindComponentByClass<USceneComponent>();
    if (MuzzleComponent)
    {
        CurrentFireComponent->AttachToComponent(
            MuzzleComponent,
            FAttachmentTransformRules::SnapToTargetIncludingScale,
            FName("Muzzle")  // Socket 名称
        );
    }

    // 播放
    CurrentFireComponent->Play();

    // 播放尾音（可选）
    if (FireTailSound)
    {
        UGameplayStatics::PlaySoundAtLocation(
            this,
            FireTailSound,
            Owner->GetActorLocation(),
            FRotator::ZeroRotator,
            0.5f,  // 较低音量
            Pitch
        );
    }
}

void UWeaponAudioComponent::PlayReloadSound()
{
    if (!ReloadSound)
    {
        return;
    }

    UAudioComponent* ReloadComponent = GetPooledAudioComponent();
    if (!ReloadComponent)
    {
        return;
    }

    ReloadComponent->SetSound(ReloadSound);
    ReloadComponent->SetVolumeMultiplier(1.0f);
    ReloadComponent->SetPitchMultiplier(1.0f);

    if (ReloadAttenuation)
    {
        ReloadComponent->AttenuationSettings = ReloadAttenuation;
    }

    // 附加到武器
    if (AActor* Owner = GetOwner())
    {
        ReloadComponent->AttachToComponent(
            Owner->GetRootComponent(),
            FAttachmentTransformRules::SnapToTargetIncludingScale
        );
    }

    ReloadComponent->Play();
}

void UWeaponAudioComponent::PlayDryFireSound()
{
    if (!DryFireSound)
    {
        return;
    }

    // 空枪音效通常较小，使用 2D 音效即可
    UGameplayStatics::PlaySound2D(this, DryFireSound, 0.7f, 1.0f);
}

void UWeaponAudioComponent::StopAllSounds(float FadeOutDuration)
{
    if (CurrentFireComponent && CurrentFireComponent->IsPlaying())
    {
        CurrentFireComponent->FadeOut(FadeOutDuration, 0.0f);
    }

    for (UAudioComponent* AudioComp : AudioComponentPool)
    {
        if (AudioComp && AudioComp->IsPlaying())
        {
            AudioComp->FadeOut(FadeOutDuration, 0.0f);
        }
    }
}

UAudioComponent* UWeaponAudioComponent::GetPooledAudioComponent()
{
    // 查找未使用的组件
    for (UAudioComponent* AudioComp : AudioComponentPool)
    {
        if (AudioComp && !AudioComp->IsPlaying())
        {
            return AudioComp;
        }
    }

    // 如果池已满，创建新的
    UAudioComponent* NewComponent = NewObject<UAudioComponent>(this);
    NewComponent->bAutoDestroy = false;
    NewComponent->bStopWhenOwnerDestroyed = true;
    NewComponent->RegisterComponent();
    AudioComponentPool.Add(NewComponent);

    return NewComponent;
}

void UWeaponAudioComponent::ReturnToPool(UAudioComponent* Component)
{
    if (Component)
    {
        Component->Stop();
        Component->DetachFromComponent(FDetachmentTransformRules::KeepRelativeTransform);
    }
}
```

### 3. 武器类集成

```cpp
// MyWeapon.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyWeapon.generated.h"

class UWeaponAudioComponent;

UCLASS()
class MYPROJECT_API AMyWeapon : public AActor
{
    GENERATED_BODY()

public:
    AMyWeapon();

protected:
    virtual void BeginPlay() override;

public:
    /** 开火 */
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void Fire();

    /** 换弹 */
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void Reload();

protected:
    // 武器音效组件
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    UWeaponAudioComponent* AudioComponent;

    // 弹药
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Weapon")
    int32 CurrentAmmo;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Weapon")
    int32 MaxAmmo;

    // 开火速率控制
    UPROPERTY(EditAnywhere, Category = "Weapon")
    float FireRate;  // 每秒发射数

    FTimerHandle FireRateTimerHandle;
    bool bCanFire;

private:
    void EnableFiring();
};

// MyWeapon.cpp
#include "MyWeapon.h"
#include "WeaponAudioComponent.h"
#include "TimerManager.h"

AMyWeapon::AMyWeapon()
{
    PrimaryActorTick.bCanEverTick = false;

    // 创建音效组件
    AudioComponent = CreateDefaultSubobject<UWeaponAudioComponent>(TEXT("AudioComponent"));

    // 默认值
    CurrentAmmo = 30;
    MaxAmmo = 30;
    FireRate = 10.0f;  // 600 RPM
    bCanFire = true;
}

void AMyWeapon::BeginPlay()
{
    Super::BeginPlay();
}

void AMyWeapon::Fire()
{
    if (!bCanFire)
    {
        return;
    }

    // 检查弹药
    if (CurrentAmmo <= 0)
    {
        // 播放空枪音效
        if (AudioComponent)
        {
            AudioComponent->PlayDryFireSound();
        }
        return;
    }

    // 消耗弹药
    CurrentAmmo--;

    // 播放开火音效
    if (AudioComponent)
    {
        // 根据连续射击调整音高（模拟机械加速）
        float PitchVariation = FMath::RandRange(0.95f, 1.05f);
        AudioComponent->PlayFireSound(PitchVariation);
    }

    // 这里添加武器逻辑（射线检测、伤害等）
    // ...

    // 开火速率限制
    bCanFire = false;
    float FireDelay = 1.0f / FireRate;
    GetWorldTimerManager().SetTimer(
        FireRateTimerHandle,
        this,
        &AMyWeapon::EnableFiring,
        FireDelay,
        false
    );
}

void AMyWeapon::Reload()
{
    if (CurrentAmmo >= MaxAmmo)
    {
        return;  // 已满弹
    }

    // 播放换弹音效
    if (AudioComponent)
    {
        AudioComponent->PlayReloadSound();
    }

    // 换弹逻辑（这里简化处理）
    CurrentAmmo = MaxAmmo;
}

void AMyWeapon::EnableFiring()
{
    bCanFire = true;
}
```

### 4. 配置音频资源

在编辑器中：

1. **创建 Sound Attenuation**：
   - 武器开火：`SA_WeaponFire`
     - Falloff Distance: 2500
     - Spatialization: HRTF
     - Air Absorption: 启用
   - 换弹：`SA_WeaponReload`
     - Falloff Distance: 500
     - Spatialization: HRTF

2. **创建 Control Bus**：
   - `CB_Weapon`: Volume 控制

3. **在武器蓝图中配置**：
   - 设置 Fire/Reload/DryFire Sound 资源
   - 分配 Attenuation Settings
   - 连接 Control Bus

## 实战案例 2：环境音效和音乐系统

### 1. 环境音效管理器

```cpp
// AmbientAudioManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/WorldSubsystem.h"
#include "GameplayTagContainer.h"
#include "AmbientAudioManager.generated.h"

class UAudioComponent;
class USoundBase;

/**
 * 环境音效数据
 */
USTRUCT(BlueprintType)
struct FAmbientAudioData
{
    GENERATED_BODY()

    /** 环境音效类型标签 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FGameplayTag LocationTag;

    /** 环境循环音效 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    USoundBase* AmbientLoop;

    /** 随机事件音效 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<USoundBase*> RandomEvents;

    /** 随机事件间隔（秒） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FVector2D EventIntervalRange = FVector2D(5.0f, 15.0f);

    /** 音量 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float Volume = 1.0f;
};

/**
 * 环境音效管理器
 * 管理关卡的环境音效和音乐
 */
UCLASS()
class MYPROJECT_API UAmbientAudioManager : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    // USubsystem 接口
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
    virtual void OnWorldBeginPlay(UWorld& InWorld) override;

    /** 切换到指定环境音效 */
    UFUNCTION(BlueprintCallable, Category = "Ambient Audio")
    void TransitionToAmbient(FGameplayTag AmbientTag, float FadeTime = 2.0f);

    /** 播放背景音乐 */
    UFUNCTION(BlueprintCallable, Category = "Ambient Audio")
    void PlayMusic(USoundBase* MusicTrack, float FadeInTime = 3.0f, bool bLoop = true);

    /** 停止背景音乐 */
    UFUNCTION(BlueprintCallable, Category = "Ambient Audio")
    void StopMusic(float FadeOutTime = 3.0f);

    /** 注册环境音效数据 */
    UFUNCTION(BlueprintCallable, Category = "Ambient Audio")
    void RegisterAmbientData(const FAmbientAudioData& AmbientData);

protected:
    /** 环境音效数据映射 */
    UPROPERTY()
    TMap<FGameplayTag, FAmbientAudioData> AmbientDataMap;

    /** 当前环境音频组件 */
    UPROPERTY()
    UAudioComponent* CurrentAmbientComponent;

    /** 音乐音频组件 */
    UPROPERTY()
    UAudioComponent* MusicComponent;

    /** 当前环境标签 */
    FGameplayTag CurrentAmbientTag;

    /** 随机事件定时器 */
    FTimerHandle RandomEventTimerHandle;

private:
    void PlayRandomEvent();
    void ScheduleNextRandomEvent();
};
```

### 2. 环境音效管理器实现

```cpp
// AmbientAudioManager.cpp
#include "AmbientAudioManager.h"
#include "Components/AudioComponent.h"
#include "Kismet/GameplayStatics.h"
#include "Sound/SoundBase.h"
#include "TimerManager.h"

void UAmbientAudioManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
}

void UAmbientAudioManager::Deinitialize()
{
    // 清理音频组件
    if (CurrentAmbientComponent)
    {
        CurrentAmbientComponent->Stop();
        CurrentAmbientComponent = nullptr;
    }

    if (MusicComponent)
    {
        MusicComponent->Stop();
        MusicComponent = nullptr;
    }

    Super::Deinitialize();
}

void UAmbientAudioManager::OnWorldBeginPlay(UWorld& InWorld)
{
    Super::OnWorldBeginPlay(InWorld);

    // 创建音频组件
    CurrentAmbientComponent = NewObject<UAudioComponent>(this);
    CurrentAmbientComponent->bAutoDestroy = false;
    CurrentAmbientComponent->RegisterComponent();

    MusicComponent = NewObject<UAudioComponent>(this);
    MusicComponent->bAutoDestroy = false;
    MusicComponent->RegisterComponent();
}

void UAmbientAudioManager::RegisterAmbientData(const FAmbientAudioData& AmbientData)
{
    if (AmbientData.LocationTag.IsValid())
    {
        AmbientDataMap.Add(AmbientData.LocationTag, AmbientData);
    }
}

void UAmbientAudioManager::TransitionToAmbient(FGameplayTag AmbientTag, float FadeTime)
{
    if (!AmbientTag.IsValid() || !AmbientDataMap.Contains(AmbientTag))
    {
        UE_LOG(LogTemp, Warning, TEXT("Ambient tag not found: %s"), *AmbientTag.ToString());
        return;
    }

    const FAmbientAudioData& AmbientData = AmbientDataMap[AmbientTag];

    // 淡出当前环境音
    if (CurrentAmbientComponent && CurrentAmbientComponent->IsPlaying())
    {
        CurrentAmbientComponent->FadeOut(FadeTime, 0.0f);
    }

    // 停止随机事件
    GetWorld()->GetTimerManager().ClearTimer(RandomEventTimerHandle);

    // 播放新环境音
    if (AmbientData.AmbientLoop)
    {
        CurrentAmbientComponent->SetSound(AmbientData.AmbientLoop);
        CurrentAmbientComponent->SetVolumeMultiplier(0.0f);
        CurrentAmbientComponent->Play();
        CurrentAmbientComponent->FadeIn(FadeTime, AmbientData.Volume);

        // 设置循环
        CurrentAmbientComponent->bAutoActivate = true;
    }

    // 更新当前环境标签
    CurrentAmbientTag = AmbientTag;

    // 启动随机事件
    if (AmbientData.RandomEvents.Num() > 0)
    {
        ScheduleNextRandomEvent();
    }
}

void UAmbientAudioManager::PlayMusic(USoundBase* MusicTrack, float FadeInTime, bool bLoop)
{
    if (!MusicTrack || !MusicComponent)
    {
        return;
    }

    // 淡出当前音乐
    if (MusicComponent->IsPlaying())
    {
        MusicComponent->FadeOut(FadeInTime * 0.5f, 0.0f);
    }

    // 播放新音乐
    MusicComponent->SetSound(MusicTrack);
    MusicComponent->SetVolumeMultiplier(0.0f);
    MusicComponent->bAutoActivate = true;
    MusicComponent->Play();
    MusicComponent->FadeIn(FadeInTime, 1.0f);
}

void UAmbientAudioManager::StopMusic(float FadeOutTime)
{
    if (MusicComponent && MusicComponent->IsPlaying())
    {
        MusicComponent->FadeOut(FadeOutTime, 0.0f);
    }
}

void UAmbientAudioManager::PlayRandomEvent()
{
    if (!CurrentAmbientTag.IsValid() || !AmbientDataMap.Contains(CurrentAmbientTag))
    {
        return;
    }

    const FAmbientAudioData& AmbientData = AmbientDataMap[CurrentAmbientTag];

    if (AmbientData.RandomEvents.Num() > 0)
    {
        // 随机选择一个事件音效
        int32 RandomIndex = FMath::RandRange(0, AmbientData.RandomEvents.Num() - 1);
        USoundBase* RandomSound = AmbientData.RandomEvents[RandomIndex];

        if (RandomSound)
        {
            // 在随机位置播放
            FVector PlayerLocation = UGameplayStatics::GetPlayerPawn(this, 0)->GetActorLocation();
            FVector RandomOffset = FVector(
                FMath::RandRange(-1000.0f, 1000.0f),
                FMath::RandRange(-1000.0f, 1000.0f),
                FMath::RandRange(-200.0f, 200.0f)
            );

            UGameplayStatics::PlaySoundAtLocation(
                this,
                RandomSound,
                PlayerLocation + RandomOffset,
                FRotator::ZeroRotator,
                FMath::RandRange(0.5f, 1.0f),  // 随机音量
                FMath::RandRange(0.9f, 1.1f)   // 随机音高
            );
        }
    }

    // 安排下一个事件
    ScheduleNextRandomEvent();
}

void UAmbientAudioManager::ScheduleNextRandomEvent()
{
    if (!CurrentAmbientTag.IsValid() || !AmbientDataMap.Contains(CurrentAmbientTag))
    {
        return;
    }

    const FAmbientAudioData& AmbientData = AmbientDataMap[CurrentAmbientTag];

    float Delay = FMath::RandRange(
        AmbientData.EventIntervalRange.X,
        AmbientData.EventIntervalRange.Y
    );

    GetWorld()->GetTimerManager().SetTimer(
        RandomEventTimerHandle,
        this,
        &UAmbientAudioManager::PlayRandomEvent,
        Delay,
        false
    );
}
```

### 3. 关卡音效触发器

```cpp
// AmbientAudioTrigger.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "GameplayTagContainer.h"
#include "AmbientAudioTrigger.generated.h"

class UBoxComponent;

/**
 * 环境音效触发器
 * 玩家进入触发区域时切换环境音效
 */
UCLASS()
class MYPROJECT_API AAmbientAudioTrigger : public AActor
{
    GENERATED_BODY()

public:
    AAmbientAudioTrigger();

protected:
    virtual void BeginPlay() override;

    UFUNCTION()
    void OnTriggerBeginOverlap(
        UPrimitiveComponent* OverlappedComponent,
        AActor* OtherActor,
        UPrimitiveComponent* OtherComp,
        int32 OtherBodyIndex,
        bool bFromSweep,
        const FHitResult& SweepResult
    );

protected:
    // 触发器组件
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    UBoxComponent* TriggerBox;

    // 环境音效标签
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Ambient Audio")
    FGameplayTag AmbientTag;

    // 切换时间
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Ambient Audio")
    float TransitionTime = 2.0f;
};

// AmbientAudioTrigger.cpp
#include "AmbientAudioTrigger.h"
#include "AmbientAudioManager.h"
#include "Components/BoxComponent.h"
#include "GameFramework/Character.h"

AAmbientAudioTrigger::AAmbientAudioTrigger()
{
    PrimaryActorTick.bCanEverTick = false;

    // 创建触发器
    TriggerBox = CreateDefaultSubobject<UBoxComponent>(TEXT("TriggerBox"));
    RootComponent = TriggerBox;
    TriggerBox->SetBoxExtent(FVector(500.0f, 500.0f, 200.0f));
    TriggerBox->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
    TriggerBox->SetCollisionResponseToAllChannels(ECR_Ignore);
    TriggerBox->SetCollisionResponseToChannel(ECC_Pawn, ECR_Overlap);
}

void AAmbientAudioTrigger::BeginPlay()
{
    Super::BeginPlay();

    // 绑定重叠事件
    TriggerBox->OnComponentBeginOverlap.AddDynamic(
        this, &AAmbientAudioTrigger::OnTriggerBeginOverlap);
}

void AAmbientAudioTrigger::OnTriggerBeginOverlap(
    UPrimitiveComponent* OverlappedComponent,
    AActor* OtherActor,
    UPrimitiveComponent* OtherComp,
    int32 OtherBodyIndex,
    bool bFromSweep,
    const FHitResult& SweepResult)
{
    // 检查是否是玩家角色
    ACharacter* Character = Cast<ACharacter>(OtherActor);
    if (!Character || !Character->IsPlayerControlled())
    {
        return;
    }

    // 获取环境音效管理器
    if (UWorld* World = GetWorld())
    {
        if (UAmbientAudioManager* AmbientManager = 
            World->GetSubsystem<UAmbientAudioManager>())
        {
            AmbientManager->TransitionToAmbient(AmbientTag, TransitionTime);
        }
    }
}
```

## 性能优化

### 1. 音频虚拟化

当音频组件数量过多时，使用虚拟化避免性能问题。

```cpp
// 配置音频虚拟化
USoundBase* Sound = LoadObject<USoundBase>(nullptr, TEXT("/Game/Audio/MySound"));
if (Sound)
{
    Sound->VirtualizationMode = EVirtualizationMode::PlayWhenSilent;
    Sound->Priority = 0.5f;  // 0.0 = 最低优先级, 1.0 = 最高优先级
}
```

### 2. 音频组件池

重用音频组件，避免频繁创建销毁。

```cpp
// AudioComponentPool.h
class FAudioComponentPool
{
public:
    UAudioComponent* Acquire(UWorld* World);
    void Release(UAudioComponent* Component);

private:
    TArray<UAudioComponent*> AvailableComponents;
    TArray<UAudioComponent*> ActiveComponents;
};

// 实现
UAudioComponent* FAudioComponentPool::Acquire(UWorld* World)
{
    if (AvailableComponents.Num() > 0)
    {
        UAudioComponent* Component = AvailableComponents.Pop();
        ActiveComponents.Add(Component);
        return Component;
    }

    // 创建新组件
    UAudioComponent* NewComponent = NewObject<UAudioComponent>(World);
    NewComponent->bAutoDestroy = false;
    NewComponent->RegisterComponent();
    ActiveComponents.Add(NewComponent);
    return NewComponent;
}

void FAudioComponentPool::Release(UAudioComponent* Component)
{
    if (Component)
    {
        Component->Stop();
        ActiveComponents.Remove(Component);
        AvailableComponents.Add(Component);
    }
}
```

### 3. 音频距离剔除

```cpp
// 手动距离检查
void PlaySoundIfInRange(USoundBase* Sound, FVector Location, float MaxDistance)
{
    APawn* PlayerPawn = UGameplayStatics::GetPlayerPawn(this, 0);
    if (!PlayerPawn)
    {
        return;
    }

    float Distance = FVector::Dist(Location, PlayerPawn->GetActorLocation());
    if (Distance > MaxDistance)
    {
        return;  // 超出范围，不播放
    }

    UGameplayStatics::PlaySoundAtLocation(this, Sound, Location);
}
```

### 4. 音频质量分级

```cpp
// 根据平台调整音频质量
void ConfigureAudioQuality()
{
    FString PlatformName = UGameplayStatics::GetPlatformName();

    if (PlatformName.Contains(TEXT("Mobile")))
    {
        // 移动平台：低质量音频
        SetMaxConcurrentSounds(16);
        SetAudioSampleRate(24000);
    }
    else if (PlatformName.Contains(TEXT("Console")))
    {
        // 主机：中等质量
        SetMaxConcurrentSounds(32);
        SetAudioSampleRate(44100);
    }
    else
    {
        // PC：高质量
        SetMaxConcurrentSounds(64);
        SetAudioSampleRate(48000);
    }
}
```

### 5. 异步加载音频资源

```cpp
// 异步加载 Sound Cue
void AsyncLoadSound(const FSoftObjectPath& SoundPath)
{
    FStreamableManager& Streamable = UAssetManager::GetStreamableManager();
    
    Streamable.RequestAsyncLoad(
        SoundPath,
        FStreamableDelegate::CreateLambda([this, SoundPath]()
        {
            USoundBase* LoadedSound = Cast<USoundBase>(SoundPath.ResolveObject());
            if (LoadedSound)
            {
                // 音频加载完成，可以使用
                UGameplayStatics::PlaySound2D(this, LoadedSound);
            }
        })
    );
}
```

## 调试和分析工具

### 1. 音频调试命令

```cpp
// 控制台命令
au.Debug.Sounds 1           // 显示所有活跃音频
au.Debug.SoundCues 1        // 显示 Sound Cue 信息
au.Debug.SoundMixes 1       // 显示 Sound Mix 状态
au.3dVisualize.Attenuation 1  // 可视化衰减半径

// 性能分析
stat SoundWaves             // 音频波形统计
stat SoundMixes             // 混音统计
stat Audio                  // 总体音频统计
```

### 2. 自定义音频日志

```cpp
// 添加到 WeaponAudioComponent
DECLARE_LOG_CATEGORY_EXTERN(LogWeaponAudio, Log, All);

DEFINE_LOG_CATEGORY(LogWeaponAudio);

void UWeaponAudioComponent::PlayFireSound(float Pitch)
{
    UE_LOG(LogWeaponAudio, Log, TEXT("Playing fire sound with pitch: %f"), Pitch);

    // 性能计时
    SCOPE_CYCLE_COUNTER(STAT_WeaponFireSound);

    // 音频播放逻辑
    // ...
}
```

## 总结

Lyra 的音频系统展示了 UE5 音频功能的强大和灵活性：

### 核心技术点

1. **Sound Control Bus** - 动态音频参数控制
2. **Audio Modulation** - 实时调制系统
3. **Submix Effect Chain** - 灵活的音频处理管线
4. **MetaSound** - 程序化音频生成
5. **Context Effects** - 上下文感知音效系统
6. **Spatial Audio** - 3D 空间音频和 HRTF

### 最佳实践

- **分类管理**：使用 Control Bus 按类别管理音频
- **音频池**：重用音频组件避免性能开销
- **异步加载**：大型音频资源按需加载
- **距离剔除**：根据距离动态启用/禁用音频
- **优先级系统**：为重要音效设置更高优先级

### 扩展方向

- **自适应音乐**：根据游戏状态动态切换音乐
- **音频遮挡**：实现声音被墙体阻挡的效果
- **音频混响区域**：不同环境自动应用混响
- **语音聊天集成**：与在线语音系统整合

通过深入理解 Lyra 的音频架构，你可以构建出专业级的游戏音频系统，为玩家提供沉浸式的听觉体验。

---

**相关文章：**
- [[24-gameplay-cue-system|Gameplay Cue 系统详解]]
- [[26-performance-optimization|性能优化最佳实践]]
- [[23-gas-advanced-topics|GAS 高级主题]]
