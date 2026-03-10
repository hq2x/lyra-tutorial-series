# Replay 系统：录像回放功能

> **本章目标**：深入理解 Lyra 的 Replay 系统架构，掌握录像录制、回放、管理的完整流程，学会实现观战模式和时间轴控制。

---

## 1. Replay 系统概述

### 1.1 什么是 Replay 系统

Unreal Engine 的 Replay 系统是一个强大的录像回放功能，它能够：

- **录制游戏过程**：捕获网络复制的数据流
- **回放录像**：重现游戏过程，支持暂停、快进、跳转
- **观战模式**：实时观看正在进行的比赛
- **数据分析**：用于调试、测试和性能分析

与传统视频录制不同，Replay 系统记录的是**游戏状态数据**而非视频帧，因此：
- 文件体积小（相比视频文件）
- 可以自由切换视角
- 支持时间轴跳转
- 可以提取游戏数据进行分析

### 1.2 Lyra Replay 系统架构

Lyra 在 UE5 的基础 Replay 功能之上，构建了一套完整的录像管理系统：

```
┌─────────────────────────────────────────────────────────────┐
│                    Lyra Replay System                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────┐       ┌────────────────────────┐  │
│  │ LyraReplaySubsystem │◄──────┤ AsyncAction_QueryReplays│ │
│  │  - RecordReplay     │       │  - 异步查询录像列表     │  │
│  │  - PlayReplay       │       └────────────────────────┘  │
│  │  - DeleteReplay     │                                    │
│  │  - SeekReplay       │       ┌────────────────────────┐  │
│  └──────────┬──────────┘       │  LyraReplayListEntry   │  │
│             │                  │  - FriendlyName        │  │
│             │                  │  - Timestamp           │  │
│             ▼                  │  - Duration            │  │
│  ┌─────────────────────┐       │  - IsLive              │  │
│  │  UDemoNetDriver     │       └────────────────────────┘  │
│  │  - 底层网络录像驱动  │                                    │
│  └──────────┬──────────┘       ┌────────────────────────┐  │
│             │                  │   LyraReplayList       │  │
│             │                  │   - Results[]          │  │
│             ▼                  └────────────────────────┘  │
│  ┌─────────────────────┐                                    │
│  │ INetworkReplayStreamer │                                │
│  │  - 存储抽象层       │                                    │
│  │  - 文件 / 网络      │                                    │
│  └─────────────────────┘                                    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**核心组件**：

1. **ULyraReplaySubsystem**：游戏实例子系统，提供录像管理的高层接口
2. **UAsyncAction_QueryReplays**：异步蓝图节点，用于查询录像列表
3. **ULyraReplayListEntry**：录像列表项，包含元数据
4. **ULyraReplayList**：录像列表容器
5. **UDemoNetDriver**：UE 底层的网络录像驱动
6. **INetworkReplayStreamer**：录像流存储接口

---

## 2. 录像录制 (Recording)

### 2.1 录制流程

#### 2.1.1 客户端录制

Lyra 支持**客户端本地录制**，这是最常见的使用场景：

```cpp
// LyraReplaySubsystem.cpp
void ULyraReplaySubsystem::RecordClientReplay(APlayerController* PlayerController)
{
    if (ensure(DoesPlatformSupportReplays() && PlayerController))
    {
        // 1. 生成友好的录像名称（带时间戳）
        FText FriendlyNameText = FText::Format(
            NSLOCTEXT("Lyra", "LyraReplayName_Format", "Client Replay {0}"), 
            FText::AsDateTime(FDateTime::UtcNow(), EDateTimeStyle::Short, EDateTimeStyle::Short)
        );
        
        // 2. 启动录制
        GetGameInstance()->StartRecordingReplay(FString(), FriendlyNameText.ToString());

        // 3. 自动清理旧录像
        if (ULyraLocalPlayer* LyraLocalPlayer = Cast<ULyraLocalPlayer>(PlayerController->GetLocalPlayer()))
        {
            int32 NumToKeep = LyraLocalPlayer->GetLocalSettings()->GetNumberOfReplaysToKeep();
            CleanupLocalReplays(LyraLocalPlayer, NumToKeep);
        }
    }
}
```

**录制流程图**：

```
玩家启动录制
    │
    ▼
检查平台支持
    │
    ├─✓─► 生成录像名称（时间戳）
    │         │
    │         ▼
    │     调用 GameInstance::StartRecordingReplay()
    │         │
    │         ▼
    │     DemoNetDriver 开始捕获网络数据
    │         │
    │         ▼
    │     触发自动清理（CleanupLocalReplays）
    │         │
    │         ▼
    │     删除超过保留数量的旧录像
    │
    └─✗─► 提示不支持
```

#### 2.1.2 平台支持检查

并非所有平台都支持 Replay：

```cpp
bool ULyraReplaySubsystem::DoesPlatformSupportReplays()
{
    // 检查平台特征标签
    if (ICommonUIModule::GetSettings().GetPlatformTraits().HasTag(GetPlatformSupportTraitTag()))
    {
        return true;
    }
    return false;
}

FGameplayTag ULyraReplaySubsystem::GetPlatformSupportTraitTag()
{
    return TAG_Platform_Trait_ReplaySupport.GetTag();
}
```

**平台支持表**：

| 平台 | 是否支持 | 说明 |
|------|---------|------|
| Windows | ✅ | 完全支持 |
| Linux | ✅ | 完全支持 |
| Mac | ✅ | 完全支持 |
| Console (PlayStation/Xbox) | ⚠️ | 需要额外配置 |
| Mobile (iOS/Android) | ⚠️ | 性能受限，需谨慎使用 |

### 2.2 服务器录制

除了客户端录制，Lyra 也支持**服务器端录制**（用于观战和赛事回放）：

```cpp
// 在 GameMode 中启动服务器录制
void ALyraGameMode::StartMatch()
{
    Super::StartMatch();
    
    // 服务器端录制
    if (bEnableServerReplay)
    {
        FString DemoName = FString::Printf(TEXT("ServerReplay_%s"), *FDateTime::Now().ToString());
        GetWorld()->GetAuthGameMode()->StartRecordingReplayFromBP(DemoName, TEXT("Server Replay"));
    }
}
```

**服务器 vs 客户端录制对比**：

| 特性 | 服务器录制 | 客户端录制 |
|------|-----------|-----------|
| 录制内容 | 完整的服务器权威数据 | 客户端接收到的复制数据 |
| 文件大小 | 较大 | 较小 |
| 准确性 | 最高（权威） | 依赖网络质量 |
| 性能影响 | 服务器承担 | 客户端承担 |
| 用途 | 观战、赛事回放 | 个人录像、Highlight |

### 2.3 录像文件管理

#### 2.3.1 自动清理机制

Lyra 实现了智能的录像清理机制，防止硬盘被占满：

```cpp
void ULyraReplaySubsystem::CleanupLocalReplays(ULocalPlayer* LocalPlayer, int32 NumReplaysToKeep)
{
    // 防止并发清理
    if (LocalPlayer != nullptr && LocalPlayerDeletingReplays == nullptr && NumReplaysToKeep != 0)
    {
        LocalPlayerDeletingReplays = LocalPlayer;
        DeletingReplaysNumberToKeep = NumReplaysToKeep;

        // 创建 Replay Streamer 用于文件操作
        CurrentReplayStreamer = FNetworkReplayStreaming::Get().GetFactory().CreateReplayStreamer();
        
        if (CurrentReplayStreamer.IsValid())
        {
            // 枚举所有录像
            FNetworkReplayVersion EnumerateStreamsVersion;
            CurrentReplayStreamer->EnumerateStreams(
                EnumerateStreamsVersion, 
                LocalPlayer->GetPlatformUserIndex(), 
                FString(), 
                TArray<FString>(), 
                FEnumerateStreamsCallback::CreateUObject(this, &ThisClass::OnEnumerateStreamsCompleteForDelete)
            );
        }
    }
}
```

#### 2.3.2 删除逻辑

```cpp
void ULyraReplaySubsystem::OnEnumerateStreamsCompleteForDelete(const FEnumerateStreamsResult& Result)
{
    if (!CurrentReplayStreamer.IsValid() || !IsValid(LocalPlayerDeletingReplays))
    {
        return; // 上下文丢失，终止
    }

    // 收集可删除的录像（排除标记为保留的）
    TArray<FNetworkReplayStreamInfo> StreamsToDelete;
    for (const FNetworkReplayStreamInfo& StreamInfo : Result.FoundStreams)
    {
        if (!StreamInfo.bShouldKeep) // 不删除被标记保留的录像
        {
            StreamsToDelete.Add(StreamInfo);
        }
    }

    // 按时间排序（最新的在前）
    Algo::SortBy(StreamsToDelete, 
        [](const FNetworkReplayStreamInfo& Data) { return Data.Timestamp.GetTicks(); }, 
        TGreater<>()
    );

    // 处理正在录制的情况
    if (UDemoNetDriver* DemoDriver = GetDemoDriver())
    {
        if (DemoDriver->IsRecording())
        {
            // 正在录制的录像可能不会出现在查询结果中
            // 插入一个虚拟条目以保证计数正确
            if (StreamsToDelete.Num() > 0 && !StreamsToDelete[0].bIsLive)
            {
                StreamsToDelete.Insert(FNetworkReplayStreamInfo(), 0);
            }
        }
    }

    // 如果超过保留数量，删除最老的录像
    if (StreamsToDelete.Num() > DeletingReplaysNumberToKeep)
    {
        FString ReplayName = StreamsToDelete[DeletingReplaysNumberToKeep].Name;
        UE_LOG(LogLyra, Log, TEXT("LyraReplaySubsystem asked to delete replay %s"), *ReplayName);
        
        CurrentReplayStreamer->DeleteFinishedStream(
            ReplayName, 
            LocalPlayerDeletingReplays->GetPlatformUserIndex(), 
            FDeleteFinishedStreamCallback::CreateUObject(this, &ThisClass::OnDeleteReplay)
        );
    }
    else
    {
        // 低于限制，停止清理
        CurrentReplayStreamer = nullptr;
        LocalPlayerDeletingReplays = nullptr;
        DeletingReplaysNumberToKeep = 0;
    }
}
```

**清理流程图**：

```
触发清理（录制开始时）
    │
    ▼
枚举所有本地录像
    │
    ▼
过滤掉被标记保留的录像
    │
    ▼
按时间排序（最新 → 最旧）
    │
    ▼
是否超过保留数量？
    │
    ├─ 是 ─► 删除最老的一个
    │         │
    │         ▼
    │     删除成功？
    │         │
    │         ├─ 是 ─► 重新枚举，继续检查
    │         └─ 否 ─► 停止清理（记录错误）
    │
    └─ 否 ─► 完成清理
```

#### 2.3.3 配置保留数量

玩家可以在设置中配置保留的录像数量：

```cpp
// LyraSettingsLocal.h
UPROPERTY(Config)
int32 NumberOfReplaysToKeep = 10; // 默认保留 10 个录像

UFUNCTION(BlueprintCallable)
int32 GetNumberOfReplaysToKeep() const { return NumberOfReplaysToKeep; }

UFUNCTION(BlueprintCallable)
void SetNumberOfReplaysToKeep(int32 InNumber)
{
    NumberOfReplaysToKeep = FMath::Clamp(InNumber, 0, 100);
    SaveSettings();
}
```

---

## 3. 录像回放 (Playback)

### 3.1 查询录像列表

#### 3.1.1 异步查询节点

Lyra 提供了蓝图友好的异步查询节点：

```cpp
// AsyncAction_QueryReplays.h
UCLASS()
class UAsyncAction_QueryReplays : public UBlueprintAsyncActionBase
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, meta = (BlueprintInternalUseOnly = "true"))
    static UAsyncAction_QueryReplays* QueryReplays(APlayerController* PlayerController);

    virtual void Activate() override;

public:
    // 查询完成时触发的委托
    UPROPERTY(BlueprintAssignable)
    FQueryReplayAsyncDelegate QueryComplete;

private:
    void OnEnumerateStreamsComplete(const FEnumerateStreamsResult& Result);

private:
    UPROPERTY()
    TObjectPtr<ULyraReplayList> ResultList;

    TWeakObjectPtr<APlayerController> PlayerController;
    TSharedPtr<INetworkReplayStreamer> ReplayStreamer;
};
```

#### 3.1.2 实现细节

```cpp
// AsyncAction_QueryReplays.cpp
void UAsyncAction_QueryReplays::Activate()
{
    // 创建 Replay Streamer
    ReplayStreamer = FNetworkReplayStreaming::Get().GetFactory().CreateReplayStreamer();

    ResultList = NewObject<ULyraReplayList>();
    
    if (ReplayStreamer.IsValid())
    {
        // 使用当前引擎版本枚举录像
        FNetworkReplayVersion EnumerateStreamsVersion = FNetworkVersion::GetReplayVersion();

        ReplayStreamer->EnumerateStreams(
            EnumerateStreamsVersion, 
            INDEX_NONE, // 任意用户
            FString(),  // 任意流
            TArray<FString>(), // 无元标签过滤
            FEnumerateStreamsCallback::CreateUObject(this, &ThisClass::OnEnumerateStreamsComplete)
        );
    }
    else
    {
        // Streamer 创建失败，返回空列表
        QueryComplete.Broadcast(ResultList);
    }
}

void UAsyncAction_QueryReplays::OnEnumerateStreamsComplete(const FEnumerateStreamsResult& Result)
{
    // 将查询结果转换为 Lyra 的数据结构
    for (const FNetworkReplayStreamInfo& StreamInfo : Result.FoundStreams)
    {
        ULyraReplayListEntry* NewReplayEntry = NewObject<ULyraReplayListEntry>(ResultList);
        NewReplayEntry->StreamInfo = StreamInfo;
        ResultList->Results.Add(NewReplayEntry);
    }

    // 按时间降序排序（最新的在前）
    Algo::SortBy(ResultList->Results, 
        [](const TObjectPtr<ULyraReplayListEntry>& Data) { 
            return Data->StreamInfo.Timestamp.GetTicks(); 
        }, 
        TGreater<>()
    );

    // 触发委托，通知蓝图
    QueryComplete.Broadcast(ResultList);
}
```

#### 3.1.3 蓝图使用示例

```cpp
// 在 UI Widget 中使用
void ULyraReplayBrowserWidget::RefreshReplayList()
{
    UAsyncAction_QueryReplays* QueryAction = UAsyncAction_QueryReplays::QueryReplays(GetOwningPlayer());
    
    if (QueryAction)
    {
        QueryAction->QueryComplete.AddDynamic(this, &ULyraReplayBrowserWidget::OnReplayListReceived);
        QueryAction->Activate();
    }
}

void ULyraReplayBrowserWidget::OnReplayListReceived(ULyraReplayList* ReplayList)
{
    // 清空现有列表
    ReplayListView->ClearListItems();
    
    // 填充新数据
    for (ULyraReplayListEntry* Entry : ReplayList->Results)
    {
        ULyraReplayListItemWidget* ItemWidget = CreateWidget<ULyraReplayListItemWidget>(this, ItemWidgetClass);
        ItemWidget->SetReplayEntry(Entry);
        ReplayListView->AddItem(ItemWidget);
    }
}
```

### 3.2 播放录像

#### 3.2.1 启动回放

```cpp
// LyraReplaySubsystem.cpp
void ULyraReplaySubsystem::PlayReplay(ULyraReplayListEntry* Replay)
{
    if (Replay != nullptr)
    {
        FString DemoName = Replay->StreamInfo.Name;
        GetGameInstance()->PlayReplay(DemoName);
    }
}
```

**播放流程**：

```
用户选择录像
    │
    ▼
获取录像名称（StreamInfo.Name）
    │
    ▼
GameInstance::PlayReplay(DemoName)
    │
    ▼
创建 DemoNetDriver
    │
    ▼
加载对应的地图
    │
    ▼
开始回放网络数据流
    │
    ▼
生成 Actor 并同步状态
    │
    ▼
进入观战模式（SpectatorPawn）
```

#### 3.2.2 自动地图切换

Replay 系统会自动切换到录像对应的地图：

```cpp
// GameInstance 内部实现（引擎层）
void UGameInstance::PlayReplay(const FString& InName, UWorld* WorldOverride, const TArray<FString>& AdditionalOptions)
{
    // ...
    
    // 从录像元数据中读取地图名称
    FString MapName = ReplayStreamer->GetReplayMapName(InName);
    
    // 切换地图并启动回放
    FURL DemoURL;
    DemoURL.Map = MapName;
    DemoURL.AddOption(TEXT("DemoMode"));
    
    UEngine::Browse(*WorldContext, DemoURL, TravelError);
    
    // ...
}
```

### 3.3 时间轴控制

#### 3.3.1 跳转到指定时间

```cpp
// LyraReplaySubsystem.cpp
void ULyraReplaySubsystem::SeekInActiveReplay(float TimeInSeconds)
{
    if (UDemoNetDriver* DemoDriver = GetDemoDriver())
    {
        DemoDriver->GotoTimeInSeconds(TimeInSeconds);
    }
}
```

#### 3.3.2 获取录像信息

```cpp
// 获取录像总时长
float ULyraReplaySubsystem::GetReplayLengthInSeconds() const
{
    if (UDemoNetDriver* DemoDriver = GetDemoDriver())
    {
        return DemoDriver->GetDemoTotalTime();
    }
    return 0.0f;
}

// 获取当前播放时间
float ULyraReplaySubsystem::GetReplayCurrentTime() const
{
    if (UDemoNetDriver* DemoDriver = GetDemoDriver())
    {
        return DemoDriver->GetDemoCurrentTime();
    }
    return 0.0f;
}
```

#### 3.3.3 时间轴 UI 实现

```cpp
// 录像播放控制器 Widget
UCLASS()
class ULyraReplayControlWidget : public UUserWidget
{
    GENERATED_BODY()

protected:
    UPROPERTY(meta = (BindWidget))
    USlider* TimelineSlider;

    UPROPERTY(meta = (BindWidget))
    UTextBlock* CurrentTimeText;

    UPROPERTY(meta = (BindWidget))
    UTextBlock* TotalTimeText;

    UPROPERTY(meta = (BindWidget))
    UButton* PlayPauseButton;

    UPROPERTY(meta = (BindWidget))
    UButton* StopButton;

protected:
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override
    {
        Super::NativeTick(MyGeometry, InDeltaTime);

        if (ULyraReplaySubsystem* ReplaySubsystem = GetReplaySubsystem())
        {
            float CurrentTime = ReplaySubsystem->GetReplayCurrentTime();
            float TotalTime = ReplaySubsystem->GetReplayLengthInSeconds();

            if (TotalTime > 0.0f)
            {
                // 更新时间轴滑条
                float Progress = CurrentTime / TotalTime;
                TimelineSlider->SetValue(Progress);

                // 更新时间文本
                CurrentTimeText->SetText(FormatTime(CurrentTime));
                TotalTimeText->SetText(FormatTime(TotalTime));
            }
        }
    }

    UFUNCTION()
    void OnTimelineSliderValueChanged(float Value)
    {
        if (ULyraReplaySubsystem* ReplaySubsystem = GetReplaySubsystem())
        {
            float TotalTime = ReplaySubsystem->GetReplayLengthInSeconds();
            float TargetTime = Value * TotalTime;
            
            // 跳转到指定时间
            ReplaySubsystem->SeekInActiveReplay(TargetTime);
        }
    }

private:
    FText FormatTime(float Seconds) const
    {
        int32 Minutes = FMath::FloorToInt(Seconds / 60.0f);
        int32 RemainingSeconds = FMath::FloorToInt(Seconds) % 60;
        return FText::Format(INVTEXT("{0}:{1:02d}"), Minutes, RemainingSeconds);
    }

    ULyraReplaySubsystem* GetReplaySubsystem() const
    {
        if (UGameInstance* GI = GetWorld()->GetGameInstance())
        {
            return GI->GetSubsystem<ULyraReplaySubsystem>();
        }
        return nullptr;
    }
};
```

---

## 4. 观战模式 (Spectator Mode)

### 4.1 观战控制器

在回放过程中，玩家使用的是 **SpectatorPawn**：

```cpp
// LyraGameMode.cpp
ASpectatorPawn* ALyraGameMode::GetSpectatorPawn() const
{
    return Cast<ASpectatorPawn>(GetDefaultPawnClassForController(nullptr));
}

// 配置观战 Pawn 类
void ALyraGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);

    // 如果是 Demo 模式（回放），使用观战 Pawn
    if (GetWorld()->GetNetMode() == NM_DedicatedServer && GetWorld()->IsPlayingReplay())
    {
        SpectatorClass = ALyraSpectatorPawn::StaticClass();
    }
}
```

### 4.2 自定义观战 Pawn

创建一个功能丰富的观战 Pawn：

```cpp
// LyraSpectatorPawn.h
UCLASS()
class LYRAGAME_API ALyraSpectatorPawn : public ASpectatorPawn
{
    GENERATED_BODY()

public:
    ALyraSpectatorPawn();

protected:
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
    virtual void BeginPlay() override;

    // 相机控制
    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    float MaxSpeedMultiplier = 5.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    float MinSpeedMultiplier = 0.1f;

    float CurrentSpeedMultiplier = 1.0f;

    // 跟随目标
    UPROPERTY()
    AActor* FollowTarget;

    UPROPERTY(EditDefaultsOnly, Category = "Follow")
    float FollowDistance = 300.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Follow")
    float FollowHeight = 100.0f;

    // 输入处理
    void OnIncreaseSpeed();
    void OnDecreaseSpeed();
    void OnCycleTarget();
    void OnToggleFreeCam();
    void OnToggleUI();

    // 目标跟随
    void UpdateFollowCamera(float DeltaTime);
    AActor* FindNextTarget();

    // 自由相机模式
    bool bFreeCamMode = false;
    FVector SavedCameraLocation;
    FRotator SavedCameraRotation;

public:
    UFUNCTION(BlueprintCallable, Category = "Spectator")
    void SetFollowTarget(AActor* NewTarget);

    UFUNCTION(BlueprintPure, Category = "Spectator")
    AActor* GetFollowTarget() const { return FollowTarget; }

    UFUNCTION(BlueprintCallable, Category = "Spectator")
    void ToggleFreeCameraMode();
};
```

```cpp
// LyraSpectatorPawn.cpp
ALyraSpectatorPawn::ALyraSpectatorPawn()
{
    PrimaryActorTick.bCanEverTick = true;
    
    // 配置相机组件
    if (GetCameraComponent())
    {
        GetCameraComponent()->SetFieldOfView(90.0f);
    }
}

void ALyraSpectatorPawn::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    // 绑定输入
    PlayerInputComponent->BindAction("IncreaseSpeed", IE_Pressed, this, &ALyraSpectatorPawn::OnIncreaseSpeed);
    PlayerInputComponent->BindAction("DecreaseSpeed", IE_Pressed, this, &ALyraSpectatorPawn::OnDecreaseSpeed);
    PlayerInputComponent->BindAction("CycleTarget", IE_Pressed, this, &ALyraSpectatorPawn::OnCycleTarget);
    PlayerInputComponent->BindAction("ToggleFreeCam", IE_Pressed, this, &ALyraSpectatorPawn::OnToggleFreeCam);
    PlayerInputComponent->BindAction("ToggleUI", IE_Pressed, this, &ALyraSpectatorPawn::OnToggleUI);
}

void ALyraSpectatorPawn::OnIncreaseSpeed()
{
    CurrentSpeedMultiplier = FMath::Min(CurrentSpeedMultiplier * 1.5f, MaxSpeedMultiplier);
    
    if (UFloatingPawnMovement* Movement = GetMovementComponent())
    {
        Movement->MaxSpeed *= CurrentSpeedMultiplier;
    }
}

void ALyraSpectatorPawn::OnDecreaseSpeed()
{
    CurrentSpeedMultiplier = FMath::Max(CurrentSpeedMultiplier / 1.5f, MinSpeedMultiplier);
    
    if (UFloatingPawnMovement* Movement = GetMovementComponent())
    {
        Movement->MaxSpeed *= CurrentSpeedMultiplier;
    }
}

void ALyraSpectatorPawn::OnCycleTarget()
{
    AActor* NextTarget = FindNextTarget();
    SetFollowTarget(NextTarget);
}

AActor* ALyraSpectatorPawn::FindNextTarget()
{
    TArray<AActor*> AllCharacters;
    UGameplayStatics::GetAllActorsOfClass(GetWorld(), ALyraCharacter::StaticClass(), AllCharacters);

    if (AllCharacters.Num() == 0)
    {
        return nullptr;
    }

    // 找到当前目标的索引
    int32 CurrentIndex = AllCharacters.IndexOfByKey(FollowTarget);
    int32 NextIndex = (CurrentIndex + 1) % AllCharacters.Num();

    return AllCharacters[NextIndex];
}

void ALyraSpectatorPawn::SetFollowTarget(AActor* NewTarget)
{
    FollowTarget = NewTarget;
    bFreeCamMode = (FollowTarget == nullptr);

    if (FollowTarget)
    {
        UE_LOG(LogLyra, Log, TEXT("Now following: %s"), *FollowTarget->GetName());
    }
}

void ALyraSpectatorPawn::OnToggleFreeCam()
{
    ToggleFreeCameraMode();
}

void ALyraSpectatorPawn::ToggleFreeCameraMode()
{
    bFreeCamMode = !bFreeCamMode;

    if (bFreeCamMode)
    {
        // 保存当前相机位置
        SavedCameraLocation = GetActorLocation();
        SavedCameraRotation = GetControlRotation();
        FollowTarget = nullptr;
    }
    else
    {
        // 恢复相机位置
        SetActorLocation(SavedCameraLocation);
        GetController()->SetControlRotation(SavedCameraRotation);
    }
}

void ALyraSpectatorPawn::UpdateFollowCamera(float DeltaTime)
{
    if (!bFreeCamMode && FollowTarget)
    {
        // 计算跟随位置
        FVector TargetLocation = FollowTarget->GetActorLocation();
        FVector FollowOffset = -FollowTarget->GetActorForwardVector() * FollowDistance;
        FollowOffset.Z += FollowHeight;

        FVector DesiredLocation = TargetLocation + FollowOffset;

        // 平滑插值
        FVector NewLocation = FMath::VInterpTo(GetActorLocation(), DesiredLocation, DeltaTime, 5.0f);
        SetActorLocation(NewLocation);

        // 让相机朝向目标
        FRotator LookAtRotation = (TargetLocation - NewLocation).Rotation();
        FRotator NewRotation = FMath::RInterpTo(GetControlRotation(), LookAtRotation, DeltaTime, 5.0f);
        GetController()->SetControlRotation(NewRotation);
    }
}

void ALyraSpectatorPawn::BeginPlay()
{
    Super::BeginPlay();

    // 默认跟随第一个角色
    OnCycleTarget();
}
```

### 4.3 观战 UI

创建一个观战界面，显示玩家信息和控制提示：

```cpp
// LyraSpectatorWidget.h
UCLASS()
class ULyraSpectatorWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()

protected:
    UPROPERTY(meta = (BindWidget))
    UTextBlock* TargetNameText;

    UPROPERTY(meta = (BindWidget))
    UProgressBar* HealthBar;

    UPROPERTY(meta = (BindWidget))
    UTextBlock* HealthText;

    UPROPERTY(meta = (BindWidget))
    UTextBlock* ScoreText;

    UPROPERTY(meta = (BindWidget))
    UVerticalBox* ControlHints;

    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;

private:
    void UpdateTargetInfo();
    AActor* GetCurrentFollowTarget() const;
};
```

```cpp
// LyraSpectatorWidget.cpp
void ULyraSpectatorWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    Super::NativeTick(MyGeometry, InDeltaTime);

    UpdateTargetInfo();
}

void ULyraSpectatorWidget::UpdateTargetInfo()
{
    AActor* FollowTarget = GetCurrentFollowTarget();

    if (ALyraCharacter* Character = Cast<ALyraCharacter>(FollowTarget))
    {
        // 显示玩家名称
        if (APlayerState* PS = Character->GetPlayerState())
        {
            TargetNameText->SetText(FText::FromString(PS->GetPlayerName()));
        }

        // 显示生命值
        if (ULyraHealthComponent* HealthComp = Character->FindComponentByClass<ULyraHealthComponent>())
        {
            float Health = HealthComp->GetHealth();
            float MaxHealth = HealthComp->GetMaxHealth();
            float HealthPercent = MaxHealth > 0.0f ? Health / MaxHealth : 0.0f;

            HealthBar->SetPercent(HealthPercent);
            HealthText->SetText(FText::Format(INVTEXT("{0} / {1}"), 
                FMath::RoundToInt(Health), 
                FMath::RoundToInt(MaxHealth)
            ));
        }

        // 显示分数
        if (ALyraPlayerState* LyraPS = Cast<ALyraPlayerState>(Character->GetPlayerState()))
        {
            int32 Score = LyraPS->GetScore();
            ScoreText->SetText(FText::Format(INVTEXT("Score: {0}"), Score));
        }
    }
    else
    {
        // 无目标时隐藏信息
        TargetNameText->SetText(FText::FromString(TEXT("Free Camera")));
        HealthBar->SetPercent(0.0f);
        HealthText->SetText(FText::GetEmpty());
        ScoreText->SetText(FText::GetEmpty());
    }
}

AActor* ULyraSpectatorWidget::GetCurrentFollowTarget() const
{
    if (ALyraSpectatorPawn* SpectatorPawn = Cast<ALyraSpectatorPawn>(GetOwningPlayerPawn()))
    {
        return SpectatorPawn->GetFollowTarget();
    }
    return nullptr;
}
```

---

## 5. 高级功能

### 5.1 自定义录像元数据

为录像添加额外的元数据（如地图名称、游戏模式、玩家列表）：

```cpp
// 在开始录制时设置元数据
void ULyraReplaySubsystem::StartReplayWithMetadata(const FString& FriendlyName, const TMap<FString, FString>& MetaData)
{
    if (UDemoNetDriver* DemoDriver = GetDemoDriver())
    {
        // 添加自定义标签
        for (const auto& Pair : MetaData)
        {
            DemoDriver->AddUserToReplay(Pair.Key, Pair.Value);
        }
    }

    GetGameInstance()->StartRecordingReplay(FString(), FriendlyName);
}

// 使用示例
void ALyraGameMode::StartRecordingMatchReplay()
{
    if (ULyraReplaySubsystem* ReplaySubsystem = GetGameInstance()->GetSubsystem<ULyraReplaySubsystem>())
    {
        TMap<FString, FString> MetaData;
        MetaData.Add(TEXT("MapName"), GetWorld()->GetMapName());
        MetaData.Add(TEXT("GameMode"), GetClass()->GetName());
        MetaData.Add(TEXT("MatchID"), GenerateMatchID());
        MetaData.Add(TEXT("ServerName"), GetServerName());

        ReplaySubsystem->StartReplayWithMetadata(TEXT("Match Replay"), MetaData);
    }
}
```

### 5.2 录像过滤与搜索

实现录像过滤功能：

```cpp
// LyraReplayBrowserWidget.h
UCLASS()
class ULyraReplayBrowserWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()

protected:
    UPROPERTY(meta = (BindWidget))
    UEditableTextBox* SearchBox;

    UPROPERTY(meta = (BindWidget))
    UComboBoxString* FilterComboBox;

    UPROPERTY(meta = (BindWidget))
    UListView* ReplayListView;

    UPROPERTY()
    ULyraReplayList* AllReplays;

    UFUNCTION()
    void OnSearchTextChanged(const FText& Text);

    UFUNCTION()
    void OnFilterChanged(FString SelectedItem, ESelectInfo::Type SelectionType);

    void RefreshFilteredList();
    bool PassesFilter(ULyraReplayListEntry* Entry, const FString& SearchText, const FString& Filter) const;
};
```

```cpp
// LyraReplayBrowserWidget.cpp
void ULyraReplayBrowserWidget::OnSearchTextChanged(const FText& Text)
{
    RefreshFilteredList();
}

void ULyraReplayBrowserWidget::OnFilterChanged(FString SelectedItem, ESelectInfo::Type SelectionType)
{
    RefreshFilteredList();
}

void ULyraReplayBrowserWidget::RefreshFilteredList()
{
    if (!AllReplays)
    {
        return;
    }

    FString SearchText = SearchBox->GetText().ToString();
    FString Filter = FilterComboBox->GetSelectedOption();

    ReplayListView->ClearListItems();

    for (ULyraReplayListEntry* Entry : AllReplays->Results)
    {
        if (PassesFilter(Entry, SearchText, Filter))
        {
            ReplayListView->AddItem(Entry);
        }
    }
}

bool ULyraReplayBrowserWidget::PassesFilter(ULyraReplayListEntry* Entry, const FString& SearchText, const FString& Filter) const
{
    if (!Entry)
    {
        return false;
    }

    // 搜索文本过滤
    if (!SearchText.IsEmpty())
    {
        FString FriendlyName = Entry->GetFriendlyName().ToLower();
        if (!FriendlyName.Contains(SearchText.ToLower()))
        {
            return false;
        }
    }

    // 类型过滤
    if (Filter == TEXT("Live Only"))
    {
        return Entry->GetIsLive();
    }
    else if (Filter == TEXT("Completed Only"))
    {
        return !Entry->GetIsLive();
    }
    else if (Filter == TEXT("Today"))
    {
        FDateTime Today = FDateTime::Today();
        return Entry->GetTimestamp() >= Today;
    }
    else if (Filter == TEXT("This Week"))
    {
        FDateTime WeekAgo = FDateTime::Now() - FTimespan::FromDays(7);
        return Entry->GetTimestamp() >= WeekAgo;
    }

    return true; // "All" filter
}
```

### 5.3 慢放与快进

实现播放速度控制：

```cpp
// LyraReplaySubsystem.h
UFUNCTION(BlueprintCallable, Category = "Replays")
void SetReplayPlaybackSpeed(float Speed);

UFUNCTION(BlueprintPure, Category = "Replays")
float GetReplayPlaybackSpeed() const;
```

```cpp
// LyraReplaySubsystem.cpp
void ULyraReplaySubsystem::SetReplayPlaybackSpeed(float Speed)
{
    Speed = FMath::Clamp(Speed, 0.1f, 10.0f); // 限制范围

    if (UDemoNetDriver* DemoDriver = GetDemoDriver())
    {
        // 设置播放速度
        DemoDriver->SetDemoPlayTimeDilation(Speed);
    }

    // 同时影响 World 的时间膨胀
    if (UWorld* World = GetWorld())
    {
        World->GetWorldSettings()->DemoPlayTimeDilation = Speed;
    }
}

float ULyraReplaySubsystem::GetReplayPlaybackSpeed() const
{
    if (UDemoNetDriver* DemoDriver = GetDemoDriver())
    {
        return DemoDriver->GetDemoPlayTimeDilation();
    }
    return 1.0f;
}
```

UI 集成：

```cpp
// 速度控制 Widget
UCLASS()
class ULyraReplaySpeedControlWidget : public UUserWidget
{
    GENERATED_BODY()

protected:
    UPROPERTY(meta = (BindWidget))
    UButton* SlowMotionButton; // 0.25x

    UPROPERTY(meta = (BindWidget))
    UButton* HalfSpeedButton; // 0.5x

    UPROPERTY(meta = (BindWidget))
    UButton* NormalSpeedButton; // 1.0x

    UPROPERTY(meta = (BindWidget))
    UButton* DoubleSpeedButton; // 2.0x

    UPROPERTY(meta = (BindWidget))
    UButton* FastForwardButton; // 4.0x

    UPROPERTY(meta = (BindWidget))
    UTextBlock* CurrentSpeedText;

    UFUNCTION()
    void OnSlowMotionClicked() { SetSpeed(0.25f); }

    UFUNCTION()
    void OnHalfSpeedClicked() { SetSpeed(0.5f); }

    UFUNCTION()
    void OnNormalSpeedClicked() { SetSpeed(1.0f); }

    UFUNCTION()
    void OnDoubleSpeedClicked() { SetSpeed(2.0f); }

    UFUNCTION()
    void OnFastForwardClicked() { SetSpeed(4.0f); }

    void SetSpeed(float Speed)
    {
        if (ULyraReplaySubsystem* ReplaySubsystem = GetReplaySubsystem())
        {
            ReplaySubsystem->SetReplayPlaybackSpeed(Speed);
            UpdateSpeedText(Speed);
        }
    }

    void UpdateSpeedText(float Speed)
    {
        CurrentSpeedText->SetText(FText::Format(INVTEXT("{0}x"), Speed));
    }

    ULyraReplaySubsystem* GetReplaySubsystem() const
    {
        if (UGameInstance* GI = GetWorld()->GetGameInstance())
        {
            return GI->GetSubsystem<ULyraReplaySubsystem>();
        }
        return nullptr;
    }
};
```

### 5.4 关键时刻标记 (Highlights)

为录像添加书签功能：

```cpp
// LyraReplayHighlight.h
USTRUCT(BlueprintType)
struct FLyraReplayHighlight
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString Name;

    UPROPERTY(BlueprintReadWrite)
    float Timestamp;

    UPROPERTY(BlueprintReadWrite)
    FString Description;

    UPROPERTY(BlueprintReadWrite)
    FLinearColor Color;
};

// LyraReplaySubsystem.h (扩展)
public:
    UFUNCTION(BlueprintCallable, Category = "Replays")
    void AddHighlight(const FString& Name, const FString& Description);

    UFUNCTION(BlueprintPure, Category = "Replays")
    TArray<FLyraReplayHighlight> GetHighlights() const;

    UFUNCTION(BlueprintCallable, Category = "Replays")
    void JumpToHighlight(int32 HighlightIndex);

private:
    UPROPERTY()
    TArray<FLyraReplayHighlight> Highlights;
```

```cpp
// LyraReplaySubsystem.cpp (扩展)
void ULyraReplaySubsystem::AddHighlight(const FString& Name, const FString& Description)
{
    FLyraReplayHighlight NewHighlight;
    NewHighlight.Name = Name;
    NewHighlight.Description = Description;
    NewHighlight.Timestamp = GetReplayCurrentTime();
    NewHighlight.Color = FLinearColor::MakeRandomColor(); // 随机颜色

    Highlights.Add(NewHighlight);

    UE_LOG(LogLyra, Log, TEXT("Added highlight '%s' at %.2f seconds"), *Name, NewHighlight.Timestamp);
}

TArray<FLyraReplayHighlight> ULyraReplaySubsystem::GetHighlights() const
{
    return Highlights;
}

void ULyraReplaySubsystem::JumpToHighlight(int32 HighlightIndex)
{
    if (Highlights.IsValidIndex(HighlightIndex))
    {
        float TargetTime = Highlights[HighlightIndex].Timestamp;
        SeekInActiveReplay(TargetTime);
    }
}
```

### 5.5 自动 Highlight 检测

基于游戏事件自动添加 Highlight：

```cpp
// 在 GameMode 或其他系统中监听事件
void ALyraGameMode::OnPlayerKilledEvent(AController* Killer, AController* Victim, const FGameplayTagContainer& Tags)
{
    // 检测是否为精彩击杀
    bool bIsHighlight = false;
    FString HighlightName;

    if (Tags.HasTag(TAG_Gameplay_Elimination_Headshot))
    {
        bIsHighlight = true;
        HighlightName = TEXT("Headshot Kill");
    }
    else if (Tags.HasTag(TAG_Gameplay_Elimination_DoubleKill))
    {
        bIsHighlight = true;
        HighlightName = TEXT("Double Kill");
    }
    else if (Tags.HasTag(TAG_Gameplay_Elimination_TripleKill))
    {
        bIsHighlight = true;
        HighlightName = TEXT("Triple Kill");
    }

    if (bIsHighlight)
    {
        if (ULyraReplaySubsystem* ReplaySubsystem = GetGameInstance()->GetSubsystem<ULyraReplaySubsystem>())
        {
            FString Description = FString::Printf(TEXT("%s eliminated %s"), 
                *Killer->GetPlayerState<APlayerState>()->GetPlayerName(),
                *Victim->GetPlayerState<APlayerState>()->GetPlayerName()
            );

            ReplaySubsystem->AddHighlight(HighlightName, Description);
        }
    }
}
```

---

## 6. 性能优化

### 6.1 录像文件大小优化

#### 6.1.1 只录制必要的 Actor

```cpp
// 在 Actor 类中控制是否参与录制
void ALyraPickup::BeginPlay()
{
    Super::BeginPlay();

    // 临时拾取物不参与录制
    if (bIsTemporary)
    {
        SetReplicates(false);
        SetReplayRewindable(false);
    }
}
```

#### 6.1.2 降低录制频率

```ini
; DefaultEngine.ini
[/Script/Engine.DemoNetDriver]
MaxClientRate=15000
MaxInternetClientRate=10000
DemoRecordHz=10  ; 降低录制帧率（默认 30）
```

#### 6.1.3 压缩录像数据

```ini
[/Script/Engine.DemoNetDriver]
bCompressReplay=true
CompressionLevel=6  ; 1-9，越高压缩率越高但 CPU 开销越大
```

### 6.2 回放性能优化

#### 6.2.1 Level Streaming

在回放大地图时，使用 Level Streaming 减少内存占用：

```cpp
void ALyraGameMode::PostInitializeComponents()
{
    Super::PostInitializeComponents();

    if (GetWorld()->IsPlayingReplay())
    {
        // 回放时启用更激进的 Level Streaming
        if (ULevelStreaming* StreamingLevel = GetWorld()->GetStreamingLevels()[0])
        {
            StreamingLevel->SetShouldBeLoaded(false);
            StreamingLevel->SetShouldBeVisible(false);
        }
    }
}
```

#### 6.2.2 LOD 优化

在回放时使用更低的 LOD：

```cpp
void ALyraCharacter::PostInitializeComponents()
{
    Super::PostInitializeComponents();

    if (GetWorld()->IsPlayingReplay())
    {
        // 强制使用较低的 LOD
        if (USkeletalMeshComponent* Mesh = GetMesh())
        {
            Mesh->SetForcedLodModel(2); // 使用 LOD 2
        }
    }
}
```

### 6.3 内存优化

#### 6.3.1 限制缓存大小

```ini
[/Script/Engine.DemoNetDriver]
MaxArchiveReadPos=2147483647  ; 最大读取位置（字节）
ReplayStreamerAutoPauseBufferTime=10.0  ; 缓冲时间（秒）
```

#### 6.3.2 定期清理

```cpp
void ULyraReplaySubsystem::PerformPeriodicCleanup()
{
    // 每周自动清理一次旧录像
    FDateTime LastCleanup = LoadLastCleanupTime();
    FDateTime Now = FDateTime::Now();

    if ((Now - LastCleanup).GetDays() >= 7)
    {
        CleanupOldReplays(FTimespan::FromDays(30)); // 删除 30 天前的录像
        SaveLastCleanupTime(Now);
    }
}

void ULyraReplaySubsystem::CleanupOldReplays(const FTimespan& MaxAge)
{
    FDateTime Threshold = FDateTime::Now() - MaxAge;

    CurrentReplayStreamer = FNetworkReplayStreaming::Get().GetFactory().CreateReplayStreamer();
    if (CurrentReplayStreamer.IsValid())
    {
        FNetworkReplayVersion Version;
        CurrentReplayStreamer->EnumerateStreams(
            Version, INDEX_NONE, FString(), TArray<FString>(),
            FEnumerateStreamsCallback::CreateLambda([this, Threshold](const FEnumerateStreamsResult& Result)
            {
                for (const FNetworkReplayStreamInfo& StreamInfo : Result.FoundStreams)
                {
                    if (StreamInfo.Timestamp < Threshold && !StreamInfo.bShouldKeep)
                    {
                        CurrentReplayStreamer->DeleteFinishedStream(
                            StreamInfo.Name, INDEX_NONE,
                            FDeleteFinishedStreamCallback()
                        );
                    }
                }
            })
        );
    }
}
```

---

## 7. 实战案例：完整的 Replay 系统

### 7.1 项目结构

```
LyraReplaySystem/
├── Source/
│   ├── Replays/
│   │   ├── LyraReplaySubsystem.h
│   │   ├── LyraReplaySubsystem.cpp
│   │   ├── AsyncAction_QueryReplays.h
│   │   ├── AsyncAction_QueryReplays.cpp
│   │   ├── LyraSpectatorPawn.h
│   │   ├── LyraSpectatorPawn.cpp
│   │   └── LyraReplayTypes.h
│   └── UI/
│       ├── LyraReplayBrowserWidget.h
│       ├── LyraReplayBrowserWidget.cpp
│       ├── LyraReplayControlWidget.h
│       └── LyraReplayControlWidget.cpp
├── Content/
│   ├── UI/
│   │   ├── W_ReplayBrowser.uasset
│   │   ├── W_ReplayControl.uasset
│   │   ├── W_SpectatorHUD.uasset
│   │   └── W_HighlightList.uasset
│   └── Blueprints/
│       └── BP_SpectatorPawn.uasset
└── Config/
    └── DefaultEngine.ini
```

### 7.2 配置文件

```ini
; Config/DefaultEngine.ini

[/Script/Engine.DemoNetDriver]
DemoRecordHz=10
bCompressReplay=true
CompressionLevel=6
MaxClientRate=15000
DemoSpectatorClass=/Game/Blueprints/BP_SpectatorPawn.BP_SpectatorPawn_C

[/Script/Engine.GameEngine]
!NetDriverDefinitions=ClearArray
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="/Script/OnlineSubsystemUtils.IpNetDriver",DriverClassNameFallback="/Script/OnlineSubsystemUtils.IpNetDriver")
+NetDriverDefinitions=(DefName="DemoNetDriver",DriverClassName="/Script/Engine.DemoNetDriver",DriverClassNameFallback="/Script/Engine.DemoNetDriver")

[/Script/Engine.Engine]
+NetworkReplayStreamer=Null

[/Script/NetworkReplayStreaming.LocalFileNetworkReplayStreaming]
DemoSavePath=Saved/Demos/
```

### 7.3 完整的 UI 集成

#### 7.3.1 主菜单集成

```cpp
// LyraMainMenuWidget.cpp
void ULyraMainMenuWidget::NativeConstruct()
{
    Super::NativeConstruct();

    // 绑定录像浏览器按钮
    ReplayBrowserButton->OnClicked().AddUObject(this, &ULyraMainMenuWidget::OnReplayBrowserClicked);
}

void ULyraMainMenuWidget::OnReplayBrowserClicked()
{
    // 检查平台支持
    if (!ULyraReplaySubsystem::DoesPlatformSupportReplays())
    {
        ShowErrorDialog(LOCTEXT("ReplayNotSupported", "Replay system is not supported on this platform."));
        return;
    }

    // 打开录像浏览器
    if (UCommonActivatableWidget* ReplayBrowser = PushWidget<ULyraReplayBrowserWidget>(ReplayBrowserWidgetClass))
    {
        // 浏览器已激活
    }
}
```

#### 7.3.2 游戏内录制按钮

```cpp
// LyraHUDWidget.cpp
void ULyraHUDWidget::NativeConstruct()
{
    Super::NativeConstruct();

    // 绑定录制按钮
    RecordButton->OnClicked().AddUObject(this, &ULyraHUDWidget::OnRecordButtonClicked);

    // 检查是否正在录制
    UpdateRecordButtonState();
}

void ULyraHUDWidget::OnRecordButtonClicked()
{
    if (ULyraReplaySubsystem* ReplaySubsystem = GetGameInstance()->GetSubsystem<ULyraReplaySubsystem>())
    {
        if (IsRecording())
        {
            // 停止录制
            StopRecording();
            ShowNotification(LOCTEXT("ReplaySaved", "Replay saved successfully!"));
        }
        else
        {
            // 开始录制
            ReplaySubsystem->RecordClientReplay(GetOwningPlayer());
            ShowNotification(LOCTEXT("RecordingStarted", "Recording started..."));
        }

        UpdateRecordButtonState();
    }
}

bool ULyraHUDWidget::IsRecording() const
{
    if (UWorld* World = GetWorld())
    {
        if (UDemoNetDriver* DemoDriver = World->GetDemoNetDriver())
        {
            return DemoDriver->IsRecording();
        }
    }
    return false;
}

void ULyraHUDWidget::UpdateRecordButtonState()
{
    if (IsRecording())
    {
        RecordButton->SetButtonText(LOCTEXT("StopRecording", "⏹ Stop Recording"));
        RecordButton->SetColorAndOpacity(FLinearColor::Red);
    }
    else
    {
        RecordButton->SetButtonText(LOCTEXT("StartRecording", "⏺ Start Recording"));
        RecordButton->SetColorAndOpacity(FLinearColor::White);
    }
}
```

### 7.4 测试与调试

#### 7.4.1 控制台命令

Lyra 录像系统支持以下控制台命令：

```cpp
// 开始录制
UE5> demorec MyReplay

// 停止录制
UE5> demostop

// 播放录像
UE5> demoplay MyReplay

// 暂停/继续
UE5> demopause

// 跳转到指定时间（秒）
UE5> demogoto 60

// 设置播放速度
UE5> demospeed 0.5

// 列出所有录像
UE5> listofreplays
```

#### 7.4.2 调试日志

启用 Replay 相关的日志分类：

```cpp
// DefaultEngine.ini
[Core.Log]
LogDemo=Verbose
LogNetDormancy=Verbose
LogNetTraffic=Verbose
LogNetPlayerMovement=Verbose
```

查看日志：

```bash
# 在编辑器输出窗口过滤
UE5> log LogDemo all

# 或在控制台
UE5> log list
```

---

## 8. 常见问题与解决方案

### 8.1 录像无法播放

**问题**：点击播放后游戏崩溃或黑屏。

**原因**：
1. 录像版本不兼容（引擎版本或游戏版本更新后）
2. 录像文件损坏
3. 地图资源缺失

**解决方案**：

```cpp
// 增加版本检查
void ULyraReplaySubsystem::PlayReplay(ULyraReplayListEntry* Replay)
{
    if (Replay != nullptr)
    {
        // 检查录像版本
        if (!IsReplayVersionCompatible(Replay->StreamInfo))
        {
            UE_LOG(LogLyra, Warning, TEXT("Replay version incompatible: %s"), *Replay->StreamInfo.Name);
            ShowErrorDialog(LOCTEXT("IncompatibleReplay", "This replay was recorded with an incompatible version."));
            return;
        }

        FString DemoName = Replay->StreamInfo.Name;
        GetGameInstance()->PlayReplay(DemoName);
    }
}

bool ULyraReplaySubsystem::IsReplayVersionCompatible(const FNetworkReplayStreamInfo& StreamInfo) const
{
    // 检查引擎版本
    FNetworkReplayVersion CurrentVersion = FNetworkVersion::GetReplayVersion();
    return StreamInfo.NetworkVersion == CurrentVersion.NetworkVersion;
}
```

### 8.2 录像文件过大

**问题**：录像文件占用大量硬盘空间。

**解决方案**：

1. **降低录制频率**：

```ini
[/Script/Engine.DemoNetDriver]
DemoRecordHz=10  ; 从 30 降至 10
```

2. **启用压缩**：

```ini
[/Script/Engine.DemoNetDriver]
bCompressReplay=true
CompressionLevel=9  ; 最高压缩
```

3. **优化 Actor 复制**：

```cpp
void ALyraProjectile::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 在录像中降低复制频率
    DOREPLIFETIME_CONDITION(ALyraProjectile, CurrentVelocity, COND_ReplayOnly);
    DOREPLIFETIME_CONDITION_NOTIFY(ALyraProjectile, ProjectileState, COND_ReplayOnly, REPNOTIFY_OnChanged);
}
```

### 8.3 时间轴跳转不流畅

**问题**：使用 `SeekInActiveReplay` 跳转时，画面卡顿或冻结。

**原因**：
- DemoNetDriver 需要重新加载大量数据
- 跳转距离过远

**解决方案**：

```cpp
void ULyraReplaySubsystem::SeekInActiveReplaySafe(float TimeInSeconds)
{
    if (UDemoNetDriver* DemoDriver = GetDemoDriver())
    {
        float CurrentTime = DemoDriver->GetDemoCurrentTime();
        float Delta = FMath::Abs(TimeInSeconds - CurrentTime);

        // 如果跳转距离过大，显示加载界面
        if (Delta > 30.0f) // 超过 30 秒
        {
            ShowLoadingScreen();
        }

        DemoDriver->GotoTimeInSeconds(TimeInSeconds);

        // 延迟隐藏加载界面
        if (Delta > 30.0f)
        {
            GetWorld()->GetTimerManager().SetTimer(
                LoadingScreenTimerHandle,
                [this]()
                {
                    HideLoadingScreen();
                },
                1.0f, false
            );
        }
    }
}
```

### 8.4 观战模式相机穿墙

**问题**：观战 Pawn 的相机会穿透地形和墙壁。

**解决方案**：

```cpp
void ALyraSpectatorPawn::UpdateFollowCamera(float DeltaTime)
{
    if (!bFreeCamMode && FollowTarget)
    {
        FVector TargetLocation = FollowTarget->GetActorLocation();
        FVector FollowOffset = -FollowTarget->GetActorForwardVector() * FollowDistance;
        FollowOffset.Z += FollowHeight;

        FVector DesiredLocation = TargetLocation + FollowOffset;

        // 射线检测，防止相机穿墙
        FHitResult HitResult;
        FCollisionQueryParams QueryParams;
        QueryParams.AddIgnoredActor(this);
        QueryParams.AddIgnoredActor(FollowTarget);

        GetWorld()->LineTraceSingleByChannel(
            HitResult,
            TargetLocation,
            DesiredLocation,
            ECC_Visibility,
            QueryParams
        );

        if (HitResult.bBlockingHit)
        {
            // 如果有障碍物，将相机拉近
            DesiredLocation = HitResult.Location - HitResult.ImpactNormal * 20.0f;
        }

        // 平滑插值
        FVector NewLocation = FMath::VInterpTo(GetActorLocation(), DesiredLocation, DeltaTime, 5.0f);
        SetActorLocation(NewLocation);

        // 让相机朝向目标
        FRotator LookAtRotation = (TargetLocation - NewLocation).Rotation();
        FRotator NewRotation = FMath::RInterpTo(GetControlRotation(), LookAtRotation, DeltaTime, 5.0f);
        GetController()->SetControlRotation(NewRotation);
    }
}
```

### 8.5 录像中缺少某些 Actor

**问题**：回放时发现某些 Actor（如特效、粒子）没有出现。

**原因**：
- Actor 没有启用网络复制
- Actor 的生命周期管理不当

**解决方案**：

```cpp
// 确保特效 Actor 参与录像
void ALyraEffectActor::BeginPlay()
{
    Super::BeginPlay();

    // 即使在客户端也启用复制（为了录像）
    if (GetWorld()->IsPlayingReplay())
    {
        SetReplicates(true);
        SetReplayRewindable(true);
    }
}

// 或者在 Experience 中配置
void ULyraExperienceDefinition::ApplyReplaySettings()
{
    // 为录像启用额外的 Actor 类型
    TArray<TSubclassOf<AActor>> ReplayActorClasses = {
        ALyraEffectActor::StaticClass(),
        ALyraProjectile::StaticClass(),
        // ...
    };

    for (TSubclassOf<AActor> ActorClass : ReplayActorClasses)
    {
        ActorClass.GetDefaultObject()->SetReplicates(true);
    }
}
```

---

## 9. 总结

### 9.1 Lyra Replay 系统的优势

1. **完整的录像管理**：
   - 自动命名与时间戳
   - 智能清理机制
   - 元数据支持

2. **蓝图友好的 API**：
   - 异步查询节点
   - UI 绑定方便

3. **性能优化**：
   - 压缩支持
   - LOD 自适应
   - 内存管理

4. **观战模式**：
   - 自由相机
   - 目标跟随
   - 速度控制

### 9.2 最佳实践

1. **录制时机**：
   - 在匹配开始时自动启动录制
   - 在关键事件（击杀、胜利）时自动标记 Highlight
   - 提供手动录制开关

2. **存储管理**：
   - 设置合理的保留数量（默认 10-20 个）
   - 定期清理旧录像
   - 允许用户标记"保留"

3. **用户体验**：
   - 提供清晰的录像浏览界面
   - 支持搜索和过滤
   - 显示录像缩略图（如果可能）

4. **性能考虑**：
   - 在低端设备上默认关闭录制
   - 提供录制质量选项（低/中/高）
   - 限制录像文件大小

### 9.3 进一步学习

- **UE 官方文档**：[Replay System](https://docs.unrealengine.com/5.3/en-US/replay-system-in-unreal-engine/)
- **源码阅读**：`Engine/Source/Runtime/Engine/Private/DemoNetDriver.cpp`
- **示例项目**：Lyra 的 `LyraReplaySubsystem` 实现

---

## 10. 练习项目

### 练习 1：基础录像功能

**目标**：在 ShooterCore 中添加录像录制和回放功能。

**要求**：
1. 在主菜单添加"录像浏览器"按钮
2. 实现录像列表显示（名称、日期、时长）
3. 支持播放和删除录像

**提示**：
- 使用 `UAsyncAction_QueryReplays` 查询录像
- 使用 `ULyraReplaySubsystem::PlayReplay()` 播放
- 参考 Lyra 的 UI 结构

### 练习 2：观战模式

**目标**：创建一个功能完善的观战 Pawn。

**要求**：
1. 支持自由相机和跟随模式切换
2. 可以循环切换跟随目标
3. 显示当前跟随玩家的信息（名称、血量、分数）
4. 支持速度调节（慢放/快进）

**提示**：
- 继承 `ASpectatorPawn` 创建自定义类
- 使用射线检测防止相机穿墙
- 实现 HUD Widget 显示玩家信息

### 练习 3：自动 Highlight 系统

**目标**：实现基于游戏事件的自动 Highlight 标记。

**要求**：
1. 检测精彩击杀（爆头、连杀）
2. 自动在录像中添加书签
3. 提供 Highlight 列表，支持快速跳转
4. 保存 Highlight 数据到文件

**提示**：
- 监听 `OnPlayerKilled` 事件
- 使用 Gameplay Tags 识别特殊击杀类型
- 将 Highlight 序列化为 JSON 保存

---

**下一章预告**：《自定义 Game Feature Action》

在下一章中，我们将深入学习如何创建自定义的 Game Feature Action，扩展 Lyra 的插件系统功能。

---

> **版权声明**：本教程基于 Epic Games 的 Lyra 示例项目，仅供学习和研究使用。  
> **最后更新**：2026-03-11  
> **作者**：Lyra 教程编写组
