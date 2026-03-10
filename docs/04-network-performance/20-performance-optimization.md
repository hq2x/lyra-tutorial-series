# 20. 性能优化：Profiling 与调优实战

> **本文目标**：掌握 Lyra 项目的性能分析方法、优化策略和实战技巧，学会使用 Unreal Insights、性能统计子系统，理解资产流式加载、Tick 优化、内存管理等核心优化点，能够识别并修复实际项目中的性能瓶颈。

---

## 引言

性能优化是 UE5 大型项目开发中的关键环节。Lyra 作为 Epic 官方示例，集成了丰富的性能监控和优化机制。本文将深入剖析 Lyra 的性能架构，从工具使用到实战案例，带你系统掌握性能优化的完整流程。

### 本文涵盖内容

1. **性能统计子系统**：Lyra 的实时性能监控框架
2. **Unreal Insights**：终极性能分析工具
3. **Tick 优化**：降低 CPU 负载的核心策略
4. **Asset Streaming**：资产流式加载与内存管理
5. **渲染优化**：GPU 性能调优技巧
6. **网络性能**：包体大小与复制优化
7. **实战案例**：识别并修复真实性能问题

---

## 1. Lyra 性能统计子系统

### 1.1 核心架构

Lyra 内置了一套完善的性能统计系统，核心类包括：

- **ULyraPerformanceStatSubsystem**：性能统计子系统
- **FLyraPerformanceStatCache**：性能数据缓存
- **ELyraDisplayablePerformanceStat**：可显示的性能指标枚举

#### 性能指标类型

```cpp
// LyraPerformanceStatTypes.h
enum class ELyraDisplayablePerformanceStat : uint8
{
    // FPS 相关
    ClientFPS,              // 客户端帧率 (Hz)
    ServerFPS,              // 服务端 Tick 速率 (Hz)
    
    // 帧时间分解
    IdleTime,               // 等待垂直同步/帧率限制的空闲时间 (秒)
    FrameTime,              // 总帧时间 (秒)
    FrameTime_GameThread,   // 游戏线程时间 (秒)
    FrameTime_RenderThread, // 渲染线程时间 (秒)
    FrameTime_RHIThread,    // RHI 线程时间 (秒)
    FrameTime_GPU,          // 推断的 GPU 时间 (秒)
    
    // 网络相关
    Ping,                   // 网络 Ping (ms)
    PacketLoss_Incoming,    // 入站丢包率 (%)
    PacketLoss_Outgoing,    // 出站丢包率 (%)
    PacketRate_Incoming,    // 接收包速率 (包/秒)
    PacketRate_Outgoing,    // 发送包速率 (包/秒)
    PacketSize_Incoming,    // 接收包平均大小 (字节)
    PacketSize_Outgoing,    // 发送包平均大小 (字节)
    
    // 输入延迟（需要启用 Reflex/NVIDIA Latency Marker）
    Latency_Total,          // 总延迟 (ms)
    Latency_Game,           // 游戏模拟到驱动提交的延迟 (ms)
    Latency_Render,         // OS 渲染队列到 GPU 渲染结束的延迟 (ms)
    
    Count UMETA(Hidden)
};
```

### 1.2 性能数据采样与缓存

#### FSampledStatCache 环形缓冲区

```cpp
// 简化的采样缓存实现
class FSampledStatCache
{
public:
    FSampledStatCache(const int32 InSampleSize = 125) // 默认 125 个样本
        : SampleSize(InSampleSize)
    {
        Samples.AddZeroed(SampleSize);
    }
    
    // 记录新样本（环形缓冲）
    void RecordSample(const double Sample)
    {
        Samples[CurrentSampleIndex] = Sample;
        CurrentSampleIndex = (CurrentSampleIndex + 1) % Samples.Num();
    }
    
    // 获取最近一次样本
    double GetLastCachedStat() const
    {
        int32 LastIndex = (CurrentSampleIndex - 1 + Samples.Num()) % Samples.Num();
        return Samples[LastIndex];
    }
    
    // 计算平均值
    double GetAverage() const
    {
        double Sum = 0.0;
        for (double Sample : Samples) { Sum += Sample; }
        return Sum / static_cast<double>(SampleSize);
    }
    
    // 获取最小/最大值
    double GetMin() const { return *Algo::MinElement(Samples); }
    double GetMax() const { return *Algo::MaxElement(Samples); }
    
    // 遍历所有样本（用于绘制曲线图）
    void ForEachCurrentSample(const TFunctionRef<void(const double)> Func) const
    {
        int32 Index = CurrentSampleIndex;
        for (int32 i = 0; i < SampleSize; i++)
        {
            Func(Samples[Index]);
            Index = (Index + 1) % SampleSize;
        }
    }

private:
    const int32 SampleSize = 125;
    int32 CurrentSampleIndex = 0;
    TArray<double> Samples;
};
```

**设计亮点**：
- 环形缓冲避免内存分配开销
- 固定大小（125 样本）约支持 2 秒历史（60 FPS 下）
- 提供统计函数（平均/最大/最小）方便 UI 显示

### 1.3 性能数据收集流程

#### FLyraPerformanceStatCache 实现

```cpp
// LyraPerformanceStatSubsystem.cpp
void FLyraPerformanceStatCache::ProcessFrame(const FFrameData& FrameData)
{
    // 1. 记录帧时间统计
    RecordStat(
        ELyraDisplayablePerformanceStat::ClientFPS,
        (FrameData.TrueDeltaSeconds != 0.0) ? 1.0 / FrameData.TrueDeltaSeconds : 0.0
    );
    
    RecordStat(ELyraDisplayablePerformanceStat::IdleTime, FrameData.IdleSeconds);
    RecordStat(ELyraDisplayablePerformanceStat::FrameTime, FrameData.TrueDeltaSeconds);
    RecordStat(ELyraDisplayablePerformanceStat::FrameTime_GameThread, FrameData.GameThreadTimeSeconds);
    RecordStat(ELyraDisplayablePerformanceStat::FrameTime_RenderThread, FrameData.RenderThreadTimeSeconds);
    RecordStat(ELyraDisplayablePerformanceStat::FrameTime_RHIThread, FrameData.RHIThreadTimeSeconds);
    RecordStat(ELyraDisplayablePerformanceStat::FrameTime_GPU, FrameData.GPUTimeSeconds);
    
    // 2. 记录网络统计
    if (UWorld* World = MySubsystem->GetGameInstance()->GetWorld())
    {
        // 服务器 FPS
        if (const ALyraGameState* GameState = World->GetGameState<ALyraGameState>())
        {
            RecordStat(ELyraDisplayablePerformanceStat::ServerFPS, GameState->GetServerFPS());
        }
        
        // 客户端网络统计
        if (APlayerController* LocalPC = GEngine->GetFirstLocalPlayerController(World))
        {
            // Ping
            if (APlayerState* PS = LocalPC->GetPlayerState<APlayerState>())
            {
                RecordStat(ELyraDisplayablePerformanceStat::Ping, PS->GetPingInMilliseconds());
            }
            
            // 包统计
            if (UNetConnection* NetConnection = LocalPC->GetNetConnection())
            {
                const auto& InLoss = NetConnection->GetInLossPercentage();
                RecordStat(ELyraDisplayablePerformanceStat::PacketLoss_Incoming, 
                          InLoss.GetAvgLossPercentage());
                
                const auto& OutLoss = NetConnection->GetOutLossPercentage();
                RecordStat(ELyraDisplayablePerformanceStat::PacketLoss_Outgoing, 
                          OutLoss.GetAvgLossPercentage());
                
                RecordStat(ELyraDisplayablePerformanceStat::PacketRate_Incoming, 
                          NetConnection->InPacketsPerSecond);
                RecordStat(ELyraDisplayablePerformanceStat::PacketRate_Outgoing, 
                          NetConnection->OutPacketsPerSecond);
                
                // 计算平均包大小
                float AvgInSize = (NetConnection->InPacketsPerSecond != 0) 
                    ? (NetConnection->InBytesPerSecond / (float)NetConnection->InPacketsPerSecond) 
                    : 0.0f;
                RecordStat(ELyraDisplayablePerformanceStat::PacketSize_Incoming, AvgInSize);
                
                float AvgOutSize = (NetConnection->OutPacketsPerSecond != 0) 
                    ? (NetConnection->OutBytesPerSecond / (float)NetConnection->OutPacketsPerSecond) 
                    : 0.0f;
                RecordStat(ELyraDisplayablePerformanceStat::PacketSize_Outgoing, AvgOutSize);
            }
            
            // 3. 输入延迟（Reflex/NVIDIA）
            TArray<ILatencyMarkerModule*> LatencyModules = 
                IModularFeatures::Get().GetModularFeatureImplementations<ILatencyMarkerModule>(
                    ILatencyMarkerModule::GetModularFeatureName());
            
            for (ILatencyMarkerModule* Module : LatencyModules)
            {
                if (Module->GetEnabled() && Module->GetTotalLatencyInMs() > 0.0f)
                {
                    RecordStat(ELyraDisplayablePerformanceStat::Latency_Total, 
                              Module->GetTotalLatencyInMs());
                    RecordStat(ELyraDisplayablePerformanceStat::Latency_Game, 
                              Module->GetGameLatencyInMs());
                    RecordStat(ELyraDisplayablePerformanceStat::Latency_Render, 
                              Module->GetRenderLatencyInMs());
                    
                    // CSV Profiler 集成
                    #if CSV_PROFILER
                    if (FCsvProfiler* Profiler = FCsvProfiler::Get())
                    {
                        static const FName TotalLatencyStatName = TEXT("Lyra_Latency_Total");
                        Profiler->RecordCustomStat(TotalLatencyStatName, 
                            CSV_CATEGORY_INDEX(LyraPerformance), 
                            Module->GetTotalLatencyInMs(), 
                            ECsvCustomStatOp::Set);
                    }
                    #endif
                    break;
                }
            }
        }
    }
}
```

### 1.4 在蓝图/C++ 中访问性能数据

#### 蓝图访问

```cpp
// ULyraPerformanceStatSubsystem
UFUNCTION(BlueprintCallable, Category="Performance")
double GetCachedStat(ELyraDisplayablePerformanceStat Stat) const
{
    return Tracker->GetCachedStat(Stat);
}
```

**蓝图使用示例**：

```plaintext
[Event Tick]
  |
  v
[Get Game Instance]
  |
  v
[Get Subsystem (Class: LyraPerformanceStatSubsystem)]
  |
  v
[Get Cached Stat (Stat: ClientFPS)]
  |
  v
[Print String (Format: "FPS: {0}")]
```

#### C++ 访问

```cpp
// 获取子系统
ULyraPerformanceStatSubsystem* PerfSubsystem = 
    GetWorld()->GetGameInstance()->GetSubsystem<ULyraPerformanceStatSubsystem>();

// 获取当前 FPS
double CurrentFPS = PerfSubsystem->GetCachedStat(
    ELyraDisplayablePerformanceStat::ClientFPS);

// 获取详细统计数据（用于绘制曲线）
const FSampledStatCache* FPSCache = PerfSubsystem->GetCachedStatData(
    ELyraDisplayablePerformanceStat::ClientFPS);

if (FPSCache)
{
    double AvgFPS = FPSCache->GetAverage();
    double MinFPS = FPSCache->GetMin();
    double MaxFPS = FPSCache->GetMax();
    
    UE_LOG(LogTemp, Log, TEXT("FPS - Avg: %.1f, Min: %.1f, Max: %.1f"), 
           AvgFPS, MinFPS, MaxFPS);
    
    // 遍历所有样本（绘制曲线图）
    TArray<float> FPSSamples;
    FPSCache->ForEachCurrentSample([&FPSSamples](double Sample) {
        FPSSamples.Add(static_cast<float>(Sample));
    });
}
```

### 1.5 性能显示 Widget（实战示例）

创建一个简单的性能显示 UI：

```cpp
// LyraPerfStatsWidget.h
UCLASS()
class ULyraPerfStatsWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;

protected:
    UPROPERTY(meta = (BindWidget))
    UTextBlock* FPSText;
    
    UPROPERTY(meta = (BindWidget))
    UTextBlock* PingText;
    
    UPROPERTY(meta = (BindWidget))
    UTextBlock* FrameTimeText;
    
    UPROPERTY(meta = (BindWidget))
    UProgressBar* GPUBar; // GPU 时间进度条
};

// LyraPerfStatsWidget.cpp
void ULyraPerfStatsWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    Super::NativeTick(MyGeometry, InDeltaTime);
    
    ULyraPerformanceStatSubsystem* PerfSubsystem = 
        GetWorld()->GetGameInstance()->GetSubsystem<ULyraPerformanceStatSubsystem>();
    
    if (!PerfSubsystem) return;
    
    // 更新 FPS
    double FPS = PerfSubsystem->GetCachedStat(ELyraDisplayablePerformanceStat::ClientFPS);
    FPSText->SetText(FText::Format(INVTEXT("FPS: {0}"), FMath::RoundToInt(FPS)));
    
    // 更新 Ping
    double Ping = PerfSubsystem->GetCachedStat(ELyraDisplayablePerformanceStat::Ping);
    PingText->SetText(FText::Format(INVTEXT("Ping: {0}ms"), FMath::RoundToInt(Ping)));
    
    // 更新帧时间
    double FrameTime = PerfSubsystem->GetCachedStat(ELyraDisplayablePerformanceStat::FrameTime);
    FrameTimeText->SetText(FText::Format(INVTEXT("Frame: {0}ms"), 
                          FMath::RoundToInt(FrameTime * 1000.0)));
    
    // 更新 GPU 进度条（假设 16.67ms 为 100%）
    double GPUTime = PerfSubsystem->GetCachedStat(ELyraDisplayablePerformanceStat::FrameTime_GPU);
    float GPUPercent = FMath::Clamp(GPUTime / 0.01667, 0.0, 1.0); // 60 FPS 目标
    GPUBar->SetPercent(GPUPercent);
}
```

---

## 2. Unreal Insights 深度使用

### 2.1 启动 Insights

#### 方法 1：编辑器内启动

```
菜单栏 -> Tools -> Unreal Insights
```

#### 方法 2：命令行启动

```bash
# 启动带 Trace 的游戏
UE5Editor.exe "YourProject.uproject" -game -trace=cpu,gpu,frame,log -statnamedevents

# 或在打包的游戏中启用
YourGame.exe -trace=cpu,gpu,frame,loadtime -tracefile="MyTrace.utrace"
```

#### 方法 3：运行时启动（控制台命令）

```
Trace.Start default      # 开始记录
Trace.Stop               # 停止记录
Trace.Screenshot         # 截屏并关联到 Trace
```

### 2.2 Insights 核心视图

#### 2.2.1 Timing Insights（时序分析）

最强大的性能分析工具，可视化显示每一帧的详细信息：

**关键功能**：
1. **线程视图**：查看游戏线程/渲染线程/RHI 线程/GPU 的时间线
2. **Frame 分析**：选中特定帧查看详细调用栈
3. **热点识别**：快速定位耗时最长的函数
4. **CPU/GPU 相关性**：判断瓶颈在 CPU 还是 GPU

**实战技巧**：
```
1. 选中一段时间 -> 右键 -> Zoom to Selection
2. 双击 Scope（函数块）-> 查看详细调用栈
3. Ctrl + F -> 搜索特定函数（如 "Tick"）
4. 使用 Filters -> 只显示耗时超过阈值的事件
```

#### 2.2.2 Asset Loading Insights

专门分析资产加载性能：

**关键指标**：
- **LoadTime**：资产加载总耗时
- **SerializationTime**：序列化时间
- **PostLoadTime**：PostLoad 回调时间
- **Bundle 信息**：哪些资产属于哪个 Bundle

**优化方向**：
- 减少同步加载（SynchronousLoad）
- 合理使用 Async Loading
- 优化资产依赖关系

#### 2.2.3 Memory Insights

内存分配追踪：

**核心功能**：
- **实时内存占用**：按标签/类型分类
- **内存分配调用栈**：定位内存泄漏
- **峰值内存分析**：找到内存尖峰的原因

**使用场景**：
```cpp
// 在代码中添加内存标签
LLM_SCOPE(ELLMTag::Audio);
// ... 音频相关代码
```

### 2.3 实战：识别 Tick 性能问题

#### 问题场景

游戏中有 100 个 Actor，每个都在 Tick 中执行复杂计算，导致帧率下降。

#### 诊断步骤

1. **启动 Trace 并运行游戏**
   ```
   Trace.Start cpu,frame
   # 玩一段时间后
   Trace.Stop
   ```

2. **在 Timing Insights 中找到低帧率帧**
   - 查看 Frame 时间线，选中耗时超过 16.67ms 的帧

3. **展开游戏线程，搜索 "Tick"**
   - Ctrl + F 搜索 "UActorComponent::TickComponent"
   - 查看哪些 Component 的 Tick 最耗时

4. **分析调用栈**
   - 双击耗时最长的 Tick 事件
   - 查看调用栈，定位具体函数

#### 示例输出分析

```
GameThread - 25.3ms
  |- AActor::Tick - 18.5ms
      |- UMyExpensiveComponent::TickComponent - 15.2ms
          |- MyExpensiveFunction() - 12.8ms
              |- FMath::Sqrt (called 10000 times) - 8.3ms
```

**结论**：`MyExpensiveFunction` 中调用了大量平方根计算，需要优化。

---

## 3. Tick 优化策略

### 3.1 禁用不必要的 Tick

#### 方法 1：在构造函数中禁用

```cpp
AMyActor::AMyActor()
{
    // 默认不需要 Tick
    PrimaryActorTick.bCanEverTick = false;
    PrimaryActorTick.bStartWithTickEnabled = false;
}
```

#### 方法 2：动态控制 Tick

```cpp
// 只在需要时启用 Tick
void AMyActor::OnSomethingHappened()
{
    SetActorTickEnabled(true);
}

void AMyActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // 执行逻辑
    if (TaskCompleted())
    {
        // 完成后禁用
        SetActorTickEnabled(false);
    }
}
```

### 3.2 降低 Tick 频率

#### 使用 TickInterval

```cpp
// 构造函数中设置
PrimaryActorTick.TickInterval = 0.1f; // 每 100ms Tick 一次（10 FPS）
```

**适用场景**：
- AI 决策逻辑（不需要每帧更新）
- 远距离 Actor 的状态更新
- 非关键的视觉效果

#### 自定义时间累积

```cpp
void AMyActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    TimeAccumulator += DeltaTime;
    
    // 每 0.5 秒执行一次
    if (TimeAccumulator >= 0.5f)
    {
        TimeAccumulator = 0.0f;
        ExpensiveUpdate();
    }
}
```

### 3.3 Tick 分组与优先级

#### 使用 TickGroup

```cpp
// 构造函数
PrimaryActorTick.TickGroup = TG_PostPhysics; // 在物理后 Tick
```

**TickGroup 类型**：
```cpp
enum ETickingGroup
{
    TG_PrePhysics,           // 物理前
    TG_StartPhysics,         // 物理开始
    TG_DuringPhysics,        // 物理中（异步）
    TG_EndPhysics,           // 物理结束
    TG_PostPhysics,          // 物理后
    TG_PostUpdateWork,       // 后处理
    TG_LastDemotable,        // 最后可降级
    TG_NewlySpawned,         // 新生成的
};
```

#### 配置依赖关系

```cpp
// 确保 MyActor Tick 在 TargetActor 之后
PrimaryActorTick.AddPrerequisite(TargetActor, TargetActor->PrimaryActorTick);
```

### 3.4 批量处理替代 Tick

#### 示例：使用子系统管理 AI

**反例（每个 AI 都 Tick）**：
```cpp
void AAICharacter::Tick(float DeltaTime)
{
    UpdatePathfinding();     // 每个 AI 每帧都计算
    UpdateDecisionMaking();  // 开销巨大
}
```

**正例（子系统批量处理）**：
```cpp
// AI 管理子系统
UCLASS()
class UAIManagerSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Tick(float DeltaTime) override;
    
    void RegisterAI(AAICharacter* AI);
    void UnregisterAI(AAICharacter* AI);

private:
    TArray<AAICharacter*> ActiveAIs;
    int32 CurrentUpdateIndex = 0;
};

void UAIManagerSubsystem::Tick(float DeltaTime)
{
    // 每帧只更新一部分 AI
    const int32 UpdateCount = FMath::Min(10, ActiveAIs.Num());
    
    for (int32 i = 0; i < UpdateCount; i++)
    {
        int32 Index = (CurrentUpdateIndex + i) % ActiveAIs.Num();
        if (ActiveAIs.IsValidIndex(Index))
        {
            ActiveAIs[Index]->UpdateLogic(DeltaTime);
        }
    }
    
    CurrentUpdateIndex = (CurrentUpdateIndex + UpdateCount) % FMath::Max(1, ActiveAIs.Num());
}
```

**效果**：
- 原方案：100 个 AI × 每帧更新 = 100 次更新/帧
- 优化后：10 次更新/帧，每个 AI 每 10 帧更新一次

---

## 4. Asset Streaming 与内存管理

### 4.1 Lyra 的 AssetManager 架构

#### ULyraAssetManager 核心功能

```cpp
// LyraAssetManager.h
UCLASS(MinimalAPI, Config = Game)
class ULyraAssetManager : public UAssetManager
{
    GENERATED_BODY()

public:
    // 同步加载资产（阻塞）
    template<typename AssetType>
    static AssetType* GetAsset(const TSoftObjectPtr<AssetType>& AssetPointer, bool bKeepInMemory = true);
    
    // 获取 GameData
    const ULyraGameData& GetGameData();
    
    // 记录已加载资产
    void AddLoadedAsset(const UObject* Asset);
    
    // 输出所有已加载资产
    static void DumpLoadedAssets();

protected:
    // 启动时加载核心资产
    virtual void StartInitialLoading() override;
    
    // 初始化 GameplayCue Manager
    void InitializeGameplayCueManager();

private:
    // 已加载资产集合（用于调试和内存管理）
    UPROPERTY()
    TSet<TObjectPtr<const UObject>> LoadedAssets;
    
    // 线程安全锁
    FCriticalSection LoadedAssetsCritical;
};
```

#### 同步 vs 异步加载

**同步加载（谨慎使用）**：
```cpp
// 会阻塞游戏线程！
TSoftObjectPtr<UTexture2D> TexturePtr(FSoftObjectPath(TEXT("/Game/Textures/T_Icon.T_Icon")));
UTexture2D* Texture = ULyraAssetManager::GetAsset(TexturePtr);
```

**异步加载（推荐）**：
```cpp
void AMyActor::LoadTextureAsync()
{
    TSoftObjectPtr<UTexture2D> TexturePtr(FSoftObjectPath(TEXT("/Game/Textures/T_Icon.T_Icon")));
    
    FStreamableManager& Streamable = UAssetManager::GetStreamableManager();
    Streamable.RequestAsyncLoad(
        TexturePtr.ToSoftObjectPath(),
        FStreamableDelegate::CreateUObject(this, &AMyActor::OnTextureLoaded)
    );
}

void AMyActor::OnTextureLoaded()
{
    // 加载完成回调
    if (TexturePtr.IsValid())
    {
        UTexture2D* Texture = TexturePtr.Get();
        // 使用纹理...
    }
}
```

### 4.2 Asset Bundle 系统

#### 定义 Bundle

```cpp
// LyraAssetManager.cpp
struct FLyraBundles
{
    static const FName Equipped = TEXT("Equipped"); // 装备时加载的资产
};
```

#### 使用 Bundle 加载资产

```cpp
// 在 Data Asset 中定义 Bundle
UCLASS(BlueprintType)
class ULyraItemDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 装备时需要加载的资产
    UPROPERTY(EditDefaultsOnly, Category = "Bundles")
    FAssetBundleData EquippedBundle;
};

// 加载 Bundle
void ULyraEquipmentManagerComponent::EquipItem(ULyraItemDefinition* ItemDef)
{
    FStreamableManager& Streamable = UAssetManager::GetStreamableManager();
    
    TArray<FSoftObjectPath> AssetsToLoad;
    ItemDef->EquippedBundle.GetAssets(FLyraBundles::Equipped, AssetsToLoad);
    
    Streamable.RequestAsyncLoad(
        AssetsToLoad,
        FStreamableDelegate::CreateUObject(this, &ULyraEquipmentManagerComponent::OnEquipAssetsLoaded, ItemDef)
    );
}
```

### 4.3 Primary Asset 管理

#### 配置 Primary Asset Types

```ini
; DefaultGame.ini
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="LyraItemDef",AssetBaseClass=/Script/LyraGame.LyraItemDefinition,bHasBlueprintClasses=True,bIsEditorOnly=False,Directories=((Path="/Game/Items")))
+PrimaryAssetTypesToScan=(PrimaryAssetType="LyraExperienceDefinition",AssetBaseClass=/Script/LyraGame.LyraExperienceDefinition,bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/Experiences")))
```

#### 按 ID 加载 Primary Asset

```cpp
// 异步加载 Experience
FPrimaryAssetId ExperienceId(FPrimaryAssetType("LyraExperienceDefinition"), FName("B_ShooterGame"));

UAssetManager::GetStreamableManager().RequestAsyncLoad(
    UAssetManager::Get().GetPrimaryAssetPath(ExperienceId),
    FStreamableDelegate::CreateUObject(this, &AMyGameMode::OnExperienceLoaded, ExperienceId)
);
```

### 4.4 内存优化实战

#### 监控内存使用

```cpp
// 控制台命令
Stat Memory          // 查看内存统计
Stat Streaming       // 查看资产流式加载状态
Stat StreamingDetails // 详细加载信息

// 输出所有已加载的纹理
ListTextures

// 输出所有已加载的声音
ListSounds

// 输出所有已加载的动画
ListAnimSequences
```

#### 主动卸载不需要的资产

```cpp
void AMyGameMode::UnloadExperience()
{
    // 获取 Experience 的所有依赖资产
    TArray<FSoftObjectPath> ExperienceAssets;
    CurrentExperience->GetAssetBundleData().GetAllAssets(ExperienceAssets);
    
    // 卸载
    UAssetManager::GetStreamableManager().Unload(ExperienceAssets);
    
    // 垃圾回收
    GetWorld()->ForceGarbageCollection(true);
}
```

#### 设置资产优先级

```cpp
// 高优先级异步加载（如玩家附近的资产）
FStreamableManager& Streamable = UAssetManager::GetStreamableManager();
TSharedPtr<FStreamableHandle> Handle = Streamable.RequestAsyncLoad(
    AssetPath,
    FStreamableDelegate::CreateLambda([this]() { OnAssetLoaded(); }),
    FStreamableManager::DefaultAsyncLoadPriority + 10  // 提高优先级
);
```

---

## 5. 渲染性能优化

### 5.1 GPU Profiling

#### 使用 ProfileGPU 命令

```
ProfileGPU            # 显示 GPU 各阶段耗时
Stat GPU              # 实时显示 GPU 统计
```

**输出示例**：
```
PrePass: 2.3ms
BasePass: 8.5ms
Lighting: 3.2ms
Translucency: 1.8ms
PostProcessing: 2.1ms
Total: 18.9ms
```

#### 定位 GPU 瓶颈

| 阶段 | 优化方向 |
|------|---------|
| PrePass 高 | 减少多边形数量，优化 LOD |
| BasePass 高 | 减少 Draw Call，合并材质 |
| Lighting 高 | 减少动态光源，使用烘焙光照 |
| Translucency 高 | 减少半透明物体，降低过度绘制 |
| PostProcessing 高 | 降低后处理质量，减少模糊/Bloom 强度 |

### 5.2 材质优化

#### 减少材质复杂度

**反例（昂贵材质）**：
```
BaseColor = Texture Sample × 4（多层混合）
Roughness = Complex Math（Sin/Cos）
Normal = Detail Normal × 3
```

**正例（优化后）**：
```
BaseColor = 单个 Texture Sample
Roughness = Constant
Normal = 单个 Normal Map
```

#### 使用材质 LOD

```cpp
// 在材质编辑器中设置 Quality Switch
[Quality Switch Node]
  |- High: 详细纹理 + 法线
  |- Medium: 简化纹理
  |- Low: 单色 + 无法线
```

### 5.3 LOD（Level of Detail）优化

#### 自动 LOD 生成

```cpp
// 静态网格体编辑器
LOD Settings -> Auto Compute LOD Distances
  |- LOD 0: Screen Size 1.0
  |- LOD 1: Screen Size 0.5
  |- LOD 2: Screen Size 0.25
  |- LOD 3: Screen Size 0.125
```

#### 代码中强制 LOD

```cpp
// 强制使用特定 LOD（调试用）
UStaticMeshComponent* MeshComp = GetStaticMeshComponent();
MeshComp->ForcedLodModel = 2; // 强制使用 LOD 2
```

### 5.4 Instancing（实例化渲染）

#### Instanced Static Mesh Component

```cpp
// 用一个组件渲染 1000 个树
UPROPERTY(EditAnywhere)
UInstancedStaticMeshComponent* TreeInstances;

void AForest::BeginPlay()
{
    Super::BeginPlay();
    
    for (int32 i = 0; i < 1000; i++)
    {
        FVector Location = GetRandomLocationInForest();
        FRotator Rotation = FRotator(0, FMath::RandRange(0.0f, 360.0f), 0);
        FVector Scale = FVector(FMath::RandRange(0.8f, 1.2f));
        
        FTransform Transform(Rotation, Location, Scale);
        TreeInstances->AddInstance(Transform);
    }
}
```

**性能对比**：
- 1000 个独立 Actor：1000 Draw Calls
- 1000 个 Instanced Mesh：1 Draw Call（性能提升 100 倍以上！）

---

## 6. 网络性能优化

### 6.1 Replication 优化（详见第 18 篇）

关键点：
- 使用 Replication Graph 减少 CPU 开销
- 配置合理的更新频率（NetUpdateFrequency）
- 仅复制必要的属性

### 6.2 包体大小优化

#### 检查包大小

```
Stat Net               # 查看网络统计
NetProfile Start       # 开始网络分析
NetProfile Stop        # 停止并输出结果
```

#### 优化技巧

1. **使用量化（Quantization）**：
   ```cpp
   UPROPERTY(Replicated)
   FVector_NetQuantize Location; // 压缩 Vector（精度损失可接受）
   ```

2. **条件复制**：
   ```cpp
   void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
   {
       Super::GetLifetimeReplicatedProps(OutLifetimeProps);
       
       // 只复制给所有者
       DOREPLIFETIME_CONDITION(AMyActor, Health, COND_OwnerOnly);
       
       // 只在初始化时复制
       DOREPLIFETIME_CONDITION(AMyActor, MaxHealth, COND_InitialOnly);
   }
   ```

3. **RPC 参数优化**：
   ```cpp
   // 反例：传递大型结构体
   UFUNCTION(Server, Reliable)
   void ServerBadRPC(FHugeStruct Data);
   
   // 正例：只传必要数据
   UFUNCTION(Server, Reliable)
   void ServerGoodRPC(int32 ID, float Value);
   ```

---

## 7. 实战案例：优化低帧率场景

### 7.1 问题描述

在 Battle Royale 模式中，100 个玩家同时在场时，帧率从 60 FPS 降至 25 FPS。

### 7.2 诊断流程

#### Step 1：启动 Unreal Insights

```bash
# 启动游戏并开始 Trace
-trace=cpu,gpu,frame,loadtime,net
```

#### Step 2：查看 Timing Insights

**发现问题**：
```
GameThread - 38ms (瓶颈！)
  |- AActor::Tick - 30ms
      |- APlayerCharacter::Tick - 18ms
          |- UpdateVisibility() - 12ms  // ← 罪魁祸首！
```

#### Step 3：分析代码

```cpp
// 原始代码（每帧遍历所有玩家）
void APlayerCharacter::UpdateVisibility()
{
    TArray<AActor*> AllPlayers;
    UGameplayStatics::GetAllActorsOfClass(GetWorld(), APlayerCharacter::StaticClass(), AllPlayers);
    
    for (AActor* Player : AllPlayers)
    {
        float Distance = FVector::Dist(GetActorLocation(), Player->GetActorLocation());
        if (Distance < 5000.0f)
        {
            // 显示玩家名称 UI
            ShowPlayerNameTag(Player);
        }
    }
}
```

**问题**：
- 100 个玩家 × 每帧遍历 100 个玩家 = 10000 次距离计算/帧
- `GetAllActorsOfClass` 每帧都遍历整个 World

### 7.3 优化方案

#### 优化 1：空间分区（使用 Octree）

```cpp
// 使用引擎内置的空间查询
void APlayerCharacter::UpdateVisibility()
{
    // 只查询半径 5000 范围内的 Actor
    TArray<FOverlapResult> OverlapResults;
    FCollisionShape Sphere = FCollisionShape::MakeSphere(5000.0f);
    
    GetWorld()->OverlapMultiByChannel(
        OverlapResults,
        GetActorLocation(),
        FQuat::Identity,
        ECC_Pawn,
        Sphere
    );
    
    for (const FOverlapResult& Result : OverlapResults)
    {
        if (APlayerCharacter* Player = Cast<APlayerCharacter>(Result.GetActor()))
        {
            ShowPlayerNameTag(Player);
        }
    }
}
```

**效果**：
- 查询时间从 12ms 降至 0.8ms
- 每个玩家只检查附近的几个人，而非全部 100 人

#### 优化 2：降低更新频率

```cpp
// 构造函数
APlayerCharacter::APlayerCharacter()
{
    // 可见性检查不需要每帧执行
    PrimaryActorTick.TickInterval = 0.2f; // 每 200ms 更新一次（5 FPS）
}
```

#### 优化 3：异步计算（避免阻塞游戏线程）

```cpp
void APlayerCharacter::UpdateVisibility()
{
    // 在异步任务中计算
    Async(EAsyncExecution::ThreadPool, [this]()
    {
        // 耗时计算（不阻塞游戏线程）
        TArray<APlayerCharacter*> VisiblePlayers = FindVisiblePlayers();
        
        // 结果返回游戏线程
        AsyncTask(ENamedThreads::GameThread, [this, VisiblePlayers]()
        {
            UpdateNameTags(VisiblePlayers);
        });
    });
}
```

### 7.4 优化结果

| 优化阶段 | 帧时间 | FPS | 改善 |
|---------|--------|-----|------|
| 优化前 | 38ms | 26 FPS | - |
| 空间分区 | 28ms | 35 FPS | +35% |
| 降低频率 | 21ms | 47 FPS | +81% |
| 异步计算 | 16ms | 62 FPS | +138% |

**总结**：通过三层优化，将帧率从 26 FPS 提升至 62 FPS，提升了 138%！

---

## 8. 性能优化 Checklist

### 8.1 CPU 优化

- [ ] 禁用不必要的 Tick（`bCanEverTick = false`）
- [ ] 降低 Tick 频率（`TickInterval` 或手动时间累积）
- [ ] 使用批量处理替代大量独立 Tick
- [ ] 避免每帧调用 `GetAllActorsOfClass`（使用空间查询）
- [ ] 优化 Blueprint 节点数量（复杂逻辑迁移到 C++）
- [ ] 减少动态 Cast（缓存结果）
- [ ] 使用对象池（Object Pool）避免频繁创建/销毁

### 8.2 GPU 优化

- [ ] 使用 LOD 系统（自动或手动）
- [ ] 减少 Draw Call（合并材质、使用 Instancing）
- [ ] 简化材质（减少纹理采样和复杂数学运算）
- [ ] 烘焙静态光照（避免动态光源）
- [ ] 降低半透明物体数量（避免过度绘制）
- [ ] 使用遮挡剔除（Occlusion Culling）
- [ ] 优化粒子系统（减少粒子数量和更新频率）

### 8.3 内存优化

- [ ] 使用异步加载（避免 `SynchronousLoad`）
- [ ] 配置 Asset Bundle（按需加载资产）
- [ ] 主动卸载不需要的资产（`Unload` + `ForceGarbageCollection`）
- [ ] 压缩纹理（使用 DXT/BC7 格式）
- [ ] 使用 Texture Streaming（纹理流式加载）
- [ ] 监控内存峰值（`Stat Memory`）

### 8.4 网络优化

- [ ] 使用 Replication Graph（见第 18 篇）
- [ ] 配置合理的 `NetUpdateFrequency`
- [ ] 使用条件复制（`COND_OwnerOnly` 等）
- [ ] 量化网络数据（`FVector_NetQuantize`）
- [ ] 避免 RPC 传递大型参数
- [ ] 批量发送 RPC（避免每帧多次调用）

---

## 9. 总结与最佳实践

### 9.1 性能优化原则

1. **测量优先**：永远先测量，再优化（不要凭猜测）
2. **聚焦瓶颈**：优化最耗时的 5%，而非全部
3. **逐步迭代**：每次优化后重新测量，验证效果
4. **权衡取舍**：性能 vs 可读性 vs 开发效率，找到平衡点

### 9.2 Lyra 性能架构的启示

1. **完善的监控系统**：实时性能统计子系统
2. **模块化设计**：通过 Game Features 动态加载/卸载内容
3. **数据驱动**：配置文件控制性能设置（Device Profiles）
4. **工具集成**：CSV Profiler、Unreal Insights 深度集成

### 9.3 下一步学习

- **第 21 篇**：打包发布与多平台构建
- **第 22 篇**：相机系统的性能优化
- **实战项目**：将优化技巧应用到自己的项目中

---

## 参考资源

### 官方文档
- [Unreal Insights 官方文档](https://docs.unrealengine.com/5.3/en-US/unreal-insights-in-unreal-engine/)
- [Performance and Profiling](https://docs.unrealengine.com/5.3/en-US/performance-and-profiling-in-unreal-engine/)
- [Asset Manager](https://docs.unrealengine.com/5.3/en-US/asset-management-in-unreal-engine/)

### 社区资源
- [性能优化技巧集合](https://github.com/YourRepo/PerformanceOptimizationGuide)
- [Lyra 源码分析](https://forums.unrealengine.com/c/lyra/83)

### 工具
- **Unreal Insights**：UE5 自带的性能分析工具
- **RenderDoc**：图形调试工具
- **NVIDIA Nsight**：GPU 分析工具

---

## 附录：控制台命令速查表

### 性能统计
```
Stat FPS              # 显示帧率
Stat Unit             # 显示各线程时间
Stat Game             # 游戏线程统计
Stat GPU              # GPU 统计
Stat Memory           # 内存统计
Stat Streaming        # 资产流式加载统计
Stat Net              # 网络统计
```

### Profiling
```
ProfileGPU            # GPU 详细分析
Stat StartFile        # 开始记录统计到文件
Stat StopFile         # 停止记录
Trace.Start cpu,gpu   # 启动 Unreal Insights Trace
Trace.Stop            # 停止 Trace
```

### 调试显示
```
Show Bounds           # 显示碰撞盒
Show Collision        # 显示碰撞
Show LOD              # 显示 LOD 层级
Show Wireframe        # 线框模式
Show VisualizeGBuffer # 可视化 GBuffer（BaseColor/Roughness 等）
```

### 资产管理
```
ListTextures          # 列出所有已加载纹理
ListSounds            # 列出所有已加载声音
Obj List Class=StaticMeshComponent  # 列出特定类型的对象
Obj Dump <ObjectName> # 输出对象详细信息
```

---

**文章结束**。本文系统讲解了 Lyra 的性能优化体系，从监控工具到具体优化策略，再到实战案例。掌握这些技巧后，你将能够在自己的项目中有效识别并解决性能问题，为玩家提供流畅的游戏体验。下一篇将探讨打包发布流程。
