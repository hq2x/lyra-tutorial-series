# 实战：大逃杀模式开发

## 概述

大逃杀（Battle Royale）模式是当前最受欢迎的多人游戏玩法之一，其核心特征是：100名玩家在同一地图上竞争，最后存活者获胜。本章将详细介绍如何基于 Lyra 框架开发一个完整的大逃杀游戏模式，涵盖从游戏阶段管理到网络优化的全部内容。

### 核心特性

- **大规模多人对战**：支持 100 名玩家同时在线
- **动态安全区系统**：逐渐收缩的安全区域
- **完整游戏流程**：等待 → 飞机航线 → 跳伞 → 搜索装备 → 战斗 → 结算
- **拾取和装备系统**：地图上随机生成的装备和道具
- **库存和背包管理**：复杂的物品管理系统
- **载具系统**：支持多种交通工具
- **观战系统**：死亡后观看其他玩家
- **性能优化**：针对大规模多人的网络和渲染优化

### 技术挑战

开发大逃杀模式需要解决以下核心技术挑战：

1. **网络同步**：100 玩家的高效网络复制
2. **性能优化**：大地图和大量玩家的渲染优化
3. **游戏阶段管理**：复杂的多阶段游戏流程
4. **公平性保证**：随机分布、平衡性调整
5. **服务器架构**：支持多局游戏和快速匹配

## 一、核心架构设计

### 1.1 Experience 定义

首先创建大逃杀模式的 Experience 定义：

**B_BattleRoyaleExperience**（Data Asset）：

```cpp
// LyraBattleRoyaleExperienceDefinition.h
#pragma once

#include "GameModes/LyraExperienceDefinition.h"
#include "LyraBattleRoyaleExperienceDefinition.generated.h"

/**
 * 大逃杀模式的 Experience 定义
 */
UCLASS(BlueprintType)
class ULyraBattleRoyaleExperienceDefinition : public ULyraExperienceDefinition
{
    GENERATED_BODY()

public:
    // 最大玩家数量
    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    int32 MaxPlayers = 100;

    // 最小开始游戏人数
    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    int32 MinPlayersToStart = 50;

    // 等待大厅时间（秒）
    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    float LobbyWaitTime = 60.0f;

    // 飞机飞行时间（秒）
    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    float PlaneFlightDuration = 60.0f;

    // 安全区配置
    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    TArray<FSafeZonePhaseConfig> SafeZonePhases;

    // 战利品生成配置
    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    TSubclassOf<class ULyraBRLootSpawnConfig> LootSpawnConfig;
};
```

### 1.2 游戏模式类

创建专门的 Battle Royale 游戏模式：

```cpp
// LyraBattleRoyaleGameMode.h
#pragma once

#include "GameModes/LyraGameMode.h"
#include "LyraBattleRoyaleGameMode.generated.h"

UENUM(BlueprintType)
enum class EBattleRoyalePhase : uint8
{
    WaitingInLobby      UMETA(DisplayName = "Waiting in Lobby"),
    PlaneFlying         UMETA(DisplayName = "Plane Flying"),
    Dropping            UMETA(DisplayName = "Dropping"),
    InProgress          UMETA(DisplayName = "In Progress"),
    EndGame             UMETA(DisplayName = "End Game")
};

/**
 * 大逃杀游戏模式
 * 管理游戏的整体流程和阶段转换
 */
UCLASS()
class ALyraBattleRoyaleGameMode : public ALyraGameMode
{
    GENERATED_BODY()

public:
    ALyraBattleRoyaleGameMode();

    //~ Begin AGameModeBase Interface
    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;
    virtual void PostLogin(APlayerController* NewPlayer) override;
    virtual void Logout(AController* Exiting) override;
    virtual void HandleMatchIsWaitingToStart() override;
    virtual void HandleMatchHasStarted() override;
    virtual bool ReadyToStartMatch_Implementation() override;
    //~ End AGameModeBase Interface

    // 获取当前游戏阶段
    UFUNCTION(BlueprintPure, Category = "Battle Royale")
    EBattleRoyalePhase GetCurrentPhase() const { return CurrentPhase; }

    // 切换到下一个阶段
    UFUNCTION(BlueprintCallable, Category = "Battle Royale")
    void TransitionToPhase(EBattleRoyalePhase NewPhase);

    // 玩家死亡处理
    UFUNCTION(BlueprintCallable, Category = "Battle Royale")
    void OnPlayerDied(AController* VictimController, AController* KillerController);

    // 检查是否只剩一名玩家
    UFUNCTION(BlueprintPure, Category = "Battle Royale")
    bool CheckForWinner();

protected:
    // 当前游戏阶段
    UPROPERTY(ReplicatedUsing = OnRep_CurrentPhase)
    EBattleRoyalePhase CurrentPhase;

    UFUNCTION()
    void OnRep_CurrentPhase();

    // 存活玩家数量
    UPROPERTY(Replicated)
    int32 AlivePlayers;

    // 飞机生成点
    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    TSubclassOf<class ALyraBRPlaneActor> PlaneClass;

    UPROPERTY()
    TObjectPtr<class ALyraBRPlaneActor> CurrentPlane;

    // 安全区管理器
    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    TSubclassOf<class ALyraBRSafeZoneManager> SafeZoneManagerClass;

    UPROPERTY()
    TObjectPtr<class ALyraBRSafeZoneManager> SafeZoneManager;

    // 战利品生成管理器
    UPROPERTY()
    TObjectPtr<class ULyraBRLootSpawnSubsystem> LootSpawnSubsystem;

private:
    // 开始大厅倒计时
    void StartLobbyCountdown();
    
    // 生成并启动飞机
    void SpawnAndStartPlane();
    
    // 开始游戏进行阶段
    void BeginGameplayPhase();
    
    // 结束游戏并宣布胜者
    void EndGameWithWinner(APlayerState* WinnerPlayerState);

    FTimerHandle LobbyCountdownTimer;
};
```

**实现文件（LyraBattleRoyaleGameMode.cpp）**：

```cpp
#include "LyraBattleRoyaleGameMode.h"
#include "LyraBattleRoyaleGameState.h"
#include "LyraBattleRoyalePlayerState.h"
#include "GameFramework/PlayerStart.h"
#include "Kismet/GameplayStatics.h"
#include "Net/UnrealNetwork.h"

ALyraBattleRoyaleGameMode::ALyraBattleRoyaleGameMode()
{
    // 设置默认类
    GameStateClass = ALyraBattleRoyaleGameState::StaticClass();
    PlayerStateClass = ALyraBattleRoyalePlayerState::StaticClass();
    
    CurrentPhase = EBattleRoyalePhase::WaitingInLobby;
    AlivePlayers = 0;
    
    // 禁用默认的重生
    bDelayedStart = true;
}

void ALyraBattleRoyaleGameMode::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    
    DOREPLIFETIME(ALyraBattleRoyaleGameMode, CurrentPhase);
    DOREPLIFETIME(ALyraBattleRoyaleGameMode, AlivePlayers);
}

void ALyraBattleRoyaleGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);
    
    // 初始化战利品生成系统
    LootSpawnSubsystem = GetWorld()->GetSubsystem<ULyraBRLootSpawnSubsystem>();
    if (LootSpawnSubsystem)
    {
        LootSpawnSubsystem->Initialize();
    }
    
    // 创建安全区管理器
    if (SafeZoneManagerClass)
    {
        FActorSpawnParameters SpawnParams;
        SpawnParams.Owner = this;
        SafeZoneManager = GetWorld()->SpawnActor<ALyraBRSafeZoneManager>(
            SafeZoneManagerClass, FVector::ZeroVector, FRotator::ZeroRotator, SpawnParams);
    }
}

void ALyraBattleRoyaleGameMode::PostLogin(APlayerController* NewPlayer)
{
    Super::PostLogin(NewPlayer);
    
    if (CurrentPhase == EBattleRoyalePhase::WaitingInLobby)
    {
        AlivePlayers++;
        
        // 检查是否达到最小开始人数
        ALyraBattleRoyaleGameState* BRGameState = GetGameState<ALyraBattleRoyaleGameState>();
        if (BRGameState && AlivePlayers >= BRGameState->GetMinPlayersToStart())
        {
            if (!GetWorldTimerManager().IsTimerActive(LobbyCountdownTimer))
            {
                StartLobbyCountdown();
            }
        }
        
        // 将玩家传送到大厅出生点
        TArray<AActor*> LobbySpawns;
        UGameplayStatics::GetAllActorsOfClassWithTag(GetWorld(), APlayerStart::StaticClass(), 
            FName("Lobby"), LobbySpawns);
        
        if (LobbySpawns.Num() > 0)
        {
            int32 RandomIndex = FMath::RandRange(0, LobbySpawns.Num() - 1);
            APlayerStart* SpawnPoint = Cast<APlayerStart>(LobbySpawns[RandomIndex]);
            if (SpawnPoint)
            {
                APawn* PlayerPawn = NewPlayer->GetPawn();
                if (PlayerPawn)
                {
                    PlayerPawn->SetActorLocation(SpawnPoint->GetActorLocation());
                    PlayerPawn->SetActorRotation(SpawnPoint->GetActorRotation());
                }
            }
        }
    }
}

void ALyraBattleRoyaleGameMode::Logout(AController* Exiting)
{
    if (CurrentPhase != EBattleRoyalePhase::EndGame)
    {
        AlivePlayers = FMath::Max(0, AlivePlayers - 1);
        CheckForWinner();
    }
    
    Super::Logout(Exiting);
}

bool ALyraBattleRoyaleGameMode::ReadyToStartMatch_Implementation()
{
    // 大逃杀模式需要手动控制开始
    return false;
}

void ALyraBattleRoyaleGameMode::StartLobbyCountdown()
{
    ALyraBattleRoyaleGameState* BRGameState = GetGameState<ALyraBattleRoyaleGameState>();
    if (!BRGameState) return;
    
    float LobbyWaitTime = BRGameState->GetLobbyWaitTime();
    
    GetWorldTimerManager().SetTimer(LobbyCountdownTimer, [this]()
    {
        TransitionToPhase(EBattleRoyalePhase::PlaneFlying);
    }, LobbyWaitTime, false);
    
    // 通知所有客户端倒计时开始
    BRGameState->StartLobbyCountdown(LobbyWaitTime);
}

void ALyraBattleRoyaleGameMode::TransitionToPhase(EBattleRoyalePhase NewPhase)
{
    if (CurrentPhase == NewPhase) return;
    
    CurrentPhase = NewPhase;
    OnRep_CurrentPhase();
    
    ALyraBattleRoyaleGameState* BRGameState = GetGameState<ALyraBattleRoyaleGameState>();
    if (BRGameState)
    {
        BRGameState->SetCurrentPhase(NewPhase);
    }
    
    switch (NewPhase)
    {
        case EBattleRoyalePhase::PlaneFlying:
            SpawnAndStartPlane();
            break;
            
        case EBattleRoyalePhase::InProgress:
            BeginGameplayPhase();
            break;
            
        case EBattleRoyalePhase::EndGame:
            // 游戏结束处理
            break;
    }
}

void ALyraBattleRoyaleGameMode::SpawnAndStartPlane()
{
    if (!PlaneClass) return;
    
    // 生成随机航线
    FVector StartLocation, EndLocation;
    float MapSize = 20000.0f; // 地图大小（厘米）
    
    // 随机选择方向
    float Angle = FMath::RandRange(0.0f, 360.0f);
    FVector Direction = FVector(FMath::Cos(FMath::DegreesToRadians(Angle)), 
                                FMath::Sin(FMath::DegreesToRadians(Angle)), 0);
    
    StartLocation = -Direction * MapSize + FVector(0, 0, 300000); // 3km 高度
    EndLocation = Direction * MapSize + FVector(0, 0, 300000);
    
    // 生成飞机
    FActorSpawnParameters SpawnParams;
    SpawnParams.Owner = this;
    CurrentPlane = GetWorld()->SpawnActor<ALyraBRPlaneActor>(
        PlaneClass, StartLocation, Direction.Rotation(), SpawnParams);
    
    if (CurrentPlane)
    {
        CurrentPlane->SetFlightPath(StartLocation, EndLocation);
        CurrentPlane->StartFlight();
        
        // 飞机飞行结束后进入游戏阶段
        ALyraBattleRoyaleGameState* BRGameState = GetGameState<ALyraBattleRoyaleGameState>();
        float FlightDuration = BRGameState ? BRGameState->GetPlaneFlightDuration() : 60.0f;
        
        FTimerHandle PlaneTimer;
        GetWorldTimerManager().SetTimer(PlaneTimer, [this]()
        {
            TransitionToPhase(EBattleRoyalePhase::InProgress);
        }, FlightDuration, false);
    }
}

void ALyraBattleRoyaleGameMode::BeginGameplayPhase()
{
    // 生成战利品
    if (LootSpawnSubsystem)
    {
        LootSpawnSubsystem->SpawnAllLoot();
    }
    
    // 启动安全区
    if (SafeZoneManager)
    {
        SafeZoneManager->StartSafeZoneShrink();
    }
}

void ALyraBattleRoyaleGameMode::OnPlayerDied(AController* VictimController, AController* KillerController)
{
    AlivePlayers = FMath::Max(0, AlivePlayers - 1);
    
    // 更新 GameState
    ALyraBattleRoyaleGameState* BRGameState = GetGameState<ALyraBattleRoyaleGameState>();
    if (BRGameState)
    {
        BRGameState->SetAlivePlayers(AlivePlayers);
    }
    
    // 通知击杀信息
    if (KillerController && VictimController)
    {
        APlayerState* KillerPS = KillerController->GetPlayerState<APlayerState>();
        APlayerState* VictimPS = VictimController->GetPlayerState<APlayerState>();
        
        if (BRGameState && KillerPS && VictimPS)
        {
            BRGameState->BroadcastKillMessage(KillerPS, VictimPS);
        }
    }
    
    // 检查胜利条件
    CheckForWinner();
}

bool ALyraBattleRoyaleGameMode::CheckForWinner()
{
    if (CurrentPhase == EBattleRoyalePhase::InProgress && AlivePlayers <= 1)
    {
        // 找到最后存活的玩家
        for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
        {
            APlayerController* PC = It->Get();
            if (PC && PC->GetPawn() && !PC->GetPawn()->IsPendingKillPending())
            {
                ALyraBattleRoyalePlayerState* BRPlayerState = PC->GetPlayerState<ALyraBattleRoyalePlayerState>();
                if (BRPlayerState && BRPlayerState->IsAlive())
                {
                    EndGameWithWinner(BRPlayerState);
                    return true;
                }
            }
        }
        
        // 如果没有找到存活玩家（罕见情况），结束游戏
        TransitionToPhase(EBattleRoyalePhase::EndGame);
        return true;
    }
    
    return false;
}

void ALyraBattleRoyaleGameMode::EndGameWithWinner(APlayerState* WinnerPlayerState)
{
    TransitionToPhase(EBattleRoyalePhase::EndGame);
    
    ALyraBattleRoyaleGameState* BRGameState = GetGameState<ALyraBattleRoyaleGameState>();
    if (BRGameState)
    {
        BRGameState->SetWinner(WinnerPlayerState);
        BRGameState->BroadcastGameEndMessage();
    }
}

void ALyraBattleRoyaleGameMode::OnRep_CurrentPhase()
{
    // 可以在这里添加阶段切换时的效果
    UE_LOG(LogTemp, Log, TEXT("Battle Royale Phase Changed: %d"), (int32)CurrentPhase);
}
```

### 1.3 GameState 扩展

```cpp
// LyraBattleRoyaleGameState.h
#pragma once

#include "GameModes/LyraGameState.h"
#include "LyraBattleRoyaleGameState.generated.h"

USTRUCT(BlueprintType)
struct FBRKillMessage
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString KillerName;

    UPROPERTY(BlueprintReadOnly)
    FString VictimName;

    UPROPERTY(BlueprintReadOnly)
    float Timestamp;
};

/**
 * 大逃杀游戏状态
 * 跟踪游戏的全局信息并复制给所有客户端
 */
UCLASS()
class ALyraBattleRoyaleGameState : public ALyraGameState
{
    GENERATED_BODY()

public:
    ALyraBattleRoyaleGameState();

    //~ Begin AActor Interface
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual void BeginPlay() override;
    //~ End AActor Interface

    // 设置当前阶段
    void SetCurrentPhase(EBattleRoyalePhase NewPhase);

    // 获取当前阶段
    UFUNCTION(BlueprintPure, Category = "Battle Royale")
    EBattleRoyalePhase GetCurrentPhase() const { return CurrentPhase; }

    // 设置存活玩家数量
    void SetAlivePlayers(int32 Count);

    UFUNCTION(BlueprintPure, Category = "Battle Royale")
    int32 GetAlivePlayers() const { return AlivePlayers; }

    // 开始大厅倒计时
    void StartLobbyCountdown(float Duration);

    UFUNCTION(BlueprintPure, Category = "Battle Royale")
    float GetLobbyCountdownRemaining() const;

    // 广播击杀消息
    UFUNCTION(NetMulticast, Reliable)
    void BroadcastKillMessage(APlayerState* Killer, APlayerState* Victim);

    // 设置胜者
    void SetWinner(APlayerState* Winner);

    UFUNCTION(BlueprintPure, Category = "Battle Royale")
    APlayerState* GetWinner() const { return WinnerPlayerState; }

    // 广播游戏结束消息
    UFUNCTION(NetMulticast, Reliable)
    void BroadcastGameEndMessage();

    // 获取配置参数
    int32 GetMinPlayersToStart() const { return MinPlayersToStart; }
    float GetLobbyWaitTime() const { return LobbyWaitTime; }
    float GetPlaneFlightDuration() const { return PlaneFlightDuration; }

protected:
    // 当前游戏阶段
    UPROPERTY(ReplicatedUsing = OnRep_CurrentPhase, BlueprintReadOnly)
    EBattleRoyalePhase CurrentPhase;

    UFUNCTION()
    void OnRep_CurrentPhase();

    // 存活玩家数量
    UPROPERTY(ReplicatedUsing = OnRep_AlivePlayers, BlueprintReadOnly)
    int32 AlivePlayers;

    UFUNCTION()
    void OnRep_AlivePlayers();

    // 大厅倒计时
    UPROPERTY(Replicated, BlueprintReadOnly)
    float LobbyCountdownStartTime;

    UPROPERTY(Replicated, BlueprintReadOnly)
    float LobbyCountdownDuration;

    // 胜者
    UPROPERTY(Replicated, BlueprintReadOnly)
    TObjectPtr<APlayerState> WinnerPlayerState;

    // 最近的击杀消息
    UPROPERTY(Replicated, BlueprintReadOnly)
    TArray<FBRKillMessage> RecentKills;

    // 配置参数
    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    int32 MinPlayersToStart = 50;

    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    float LobbyWaitTime = 60.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Battle Royale")
    float PlaneFlightDuration = 60.0f;

public:
    // 事件委托
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPhaseChanged, EBattleRoyalePhase, NewPhase);
    UPROPERTY(BlueprintAssignable, Category = "Battle Royale")
    FOnPhaseChanged OnPhaseChanged;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnKillBroadcast, const FString&, KillerName, const FString&, VictimName);
    UPROPERTY(BlueprintAssignable, Category = "Battle Royale")
    FOnKillBroadcast OnKillBroadcast;
};
```

**实现文件（LyraBattleRoyaleGameState.cpp）**：

```cpp
#include "LyraBattleRoyaleGameState.h"
#include "Net/UnrealNetwork.h"
#include "GameFramework/PlayerState.h"

ALyraBattleRoyaleGameState::ALyraBattleRoyaleGameState()
{
    CurrentPhase = EBattleRoyalePhase::WaitingInLobby;
    AlivePlayers = 0;
    LobbyCountdownStartTime = 0.0f;
    LobbyCountdownDuration = 0.0f;
}

void ALyraBattleRoyaleGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    
    DOREPLIFETIME(ALyraBattleRoyaleGameState, CurrentPhase);
    DOREPLIFETIME(ALyraBattleRoyaleGameState, AlivePlayers);
    DOREPLIFETIME(ALyraBattleRoyaleGameState, LobbyCountdownStartTime);
    DOREPLIFETIME(ALyraBattleRoyaleGameState, LobbyCountdownDuration);
    DOREPLIFETIME(ALyraBattleRoyaleGameState, WinnerPlayerState);
    DOREPLIFETIME(ALyraBattleRoyaleGameState, RecentKills);
}

void ALyraBattleRoyaleGameState::BeginPlay()
{
    Super::BeginPlay();
}

void ALyraBattleRoyaleGameState::SetCurrentPhase(EBattleRoyalePhase NewPhase)
{
    if (HasAuthority())
    {
        CurrentPhase = NewPhase;
        OnRep_CurrentPhase();
    }
}

void ALyraBattleRoyaleGameState::OnRep_CurrentPhase()
{
    OnPhaseChanged.Broadcast(CurrentPhase);
}

void ALyraBattleRoyaleGameState::SetAlivePlayers(int32 Count)
{
    if (HasAuthority())
    {
        AlivePlayers = Count;
    }
}

void ALyraBattleRoyaleGameState::OnRep_AlivePlayers()
{
    // 可以在这里触发 UI 更新
}

void ALyraBattleRoyaleGameState::StartLobbyCountdown(float Duration)
{
    if (HasAuthority())
    {
        LobbyCountdownStartTime = GetServerWorldTimeSeconds();
        LobbyCountdownDuration = Duration;
    }
}

float ALyraBattleRoyaleGameState::GetLobbyCountdownRemaining() const
{
    if (LobbyCountdownDuration <= 0.0f) return 0.0f;
    
    float Elapsed = GetServerWorldTimeSeconds() - LobbyCountdownStartTime;
    return FMath::Max(0.0f, LobbyCountdownDuration - Elapsed);
}

void ALyraBattleRoyaleGameState::BroadcastKillMessage_Implementation(APlayerState* Killer, APlayerState* Victim)
{
    if (!Killer || !Victim) return;
    
    FBRKillMessage KillMsg;
    KillMsg.KillerName = Killer->GetPlayerName();
    KillMsg.VictimName = Victim->GetPlayerName();
    KillMsg.Timestamp = GetServerWorldTimeSeconds();
    
    RecentKills.Add(KillMsg);
    
    // 只保留最近 10 条
    if (RecentKills.Num() > 10)
    {
        RecentKills.RemoveAt(0);
    }
    
    OnKillBroadcast.Broadcast(KillMsg.KillerName, KillMsg.VictimName);
}

void ALyraBattleRoyaleGameState::SetWinner(APlayerState* Winner)
{
    if (HasAuthority())
    {
        WinnerPlayerState = Winner;
    }
}

void ALyraBattleRoyaleGameState::BroadcastGameEndMessage_Implementation()
{
    // 在客户端显示游戏结束界面
    UE_LOG(LogTemp, Log, TEXT("Game Ended! Winner: %s"), 
           WinnerPlayerState ? *WinnerPlayerState->GetPlayerName() : TEXT("None"));
}
```

## 二、飞机和跳伞系统

### 2.1 飞机 Actor

```cpp
// LyraBRPlaneActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "LyraBRPlaneActor.generated.h"

/**
 * 大逃杀飞机 Actor
 * 负责运输玩家并允许他们跳伞
 */
UCLASS()
class ALyraBRPlaneActor : public AActor
{
    GENERATED_BODY()

public:
    ALyraBRPlaneActor();

    virtual void Tick(float DeltaTime) override;
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 设置飞行路径
    void SetFlightPath(const FVector& Start, const FVector& End);

    // 开始飞行
    UFUNCTION(BlueprintCallable, Category = "Battle Royale")
    void StartFlight();

    // 玩家跳伞
    UFUNCTION(Server, Reliable, BlueprintCallable, Category = "Battle Royale")
    void PlayerJump(APlayerController* PlayerController);

    // 是否允许跳伞
    UFUNCTION(BlueprintPure, Category = "Battle Royale")
    bool CanPlayersJump() const { return bCanPlayersJump; }

protected:
    virtual void BeginPlay() override;

    // 飞机网格体
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UStaticMeshComponent> PlaneMesh;

    // 飞行起点和终点
    UPROPERTY(Replicated)
    FVector FlightStart;

    UPROPERTY(Replicated)
    FVector FlightEnd;

    // 飞行参数
    UPROPERTY(EditDefaultsOnly, Category = "Flight")
    float FlightSpeed = 5000.0f; // cm/s

    UPROPERTY(Replicated)
    bool bIsFlying;

    UPROPERTY(Replicated)
    bool bCanPlayersJump;

    UPROPERTY(Replicated)
    float FlightStartTime;

    // 飞机上的玩家列表
    UPROPERTY(Replicated)
    TArray<TObjectPtr<APlayerController>> PlayersOnPlane;

private:
    void UpdateFlightPosition(float DeltaTime);
    void EjectPlayer(APlayerController* PlayerController, const FVector& EjectLocation);
};
```

**实现文件（LyraBRPlaneActor.cpp）**：

```cpp
#include "LyraBRPlaneActor.h"
#include "Net/UnrealNetwork.h"
#include "GameFramework/PlayerController.h"
#include "GameFramework/Character.h"
#include "Components/StaticMeshComponent.h"

ALyraBRPlaneActor::ALyraBRPlaneActor()
{
    PrimaryActorTick.bCanEverTick = true;
    bReplicates = true;
    bAlwaysRelevant = true;

    // 创建飞机网格体
    PlaneMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("PlaneMesh"));
    RootComponent = PlaneMesh;
    PlaneMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);

    bIsFlying = false;
    bCanPlayersJump = false;
    FlightStartTime = 0.0f;
}

void ALyraBRPlaneActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ALyraBRPlaneActor, FlightStart);
    DOREPLIFETIME(ALyraBRPlaneActor, FlightEnd);
    DOREPLIFETIME(ALyraBRPlaneActor, bIsFlying);
    DOREPLIFETIME(ALyraBRPlaneActor, bCanPlayersJump);
    DOREPLIFETIME(ALyraBRPlaneActor, FlightStartTime);
    DOREPLIFETIME(ALyraBRPlaneActor, PlayersOnPlane);
}

void ALyraBRPlaneActor::BeginPlay()
{
    Super::BeginPlay();
}

void ALyraBRPlaneActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (HasAuthority() && bIsFlying)
    {
        UpdateFlightPosition(DeltaTime);
    }
}

void ALyraBRPlaneActor::SetFlightPath(const FVector& Start, const FVector& End)
{
    if (HasAuthority())
    {
        FlightStart = Start;
        FlightEnd = End;
    }
}

void ALyraBRPlaneActor::StartFlight()
{
    if (HasAuthority())
    {
        bIsFlying = true;
        bCanPlayersJump = true;
        FlightStartTime = GetWorld()->GetTimeSeconds();

        // 设置初始位置
        SetActorLocation(FlightStart);
        SetActorRotation((FlightEnd - FlightStart).Rotation());

        // 收集所有活着的玩家
        for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
        {
            APlayerController* PC = It->Get();
            if (PC && PC->GetPawn())
            {
                PlayersOnPlane.Add(PC);
                
                // 禁用玩家控制，附加到飞机
                PC->SetIgnoreMoveInput(true);
                PC->GetPawn()->AttachToActor(this, FAttachmentTransformRules::KeepWorldTransform);
                PC->GetPawn()->SetActorHiddenInGame(true); // 在飞机上时隐藏玩家
            }
        }
    }
}

void ALyraBRPlaneActor::UpdateFlightPosition(float DeltaTime)
{
    float ElapsedTime = GetWorld()->GetTimeSeconds() - FlightStartTime;
    FVector Direction = (FlightEnd - FlightStart).GetSafeNormal();
    float DistanceTraveled = ElapsedTime * FlightSpeed;

    FVector NewLocation = FlightStart + Direction * DistanceTraveled;
    SetActorLocation(NewLocation);

    // 检查是否到达终点
    float TotalDistance = FVector::Dist(FlightStart, FlightEnd);
    if (DistanceTraveled >= TotalDistance)
    {
        // 强制所有剩余玩家跳伞
        TArray<APlayerController*> RemainingPlayers = PlayersOnPlane;
        for (APlayerController* PC : RemainingPlayers)
        {
            if (PC)
            {
                PlayerJump(PC);
            }
        }

        bIsFlying = false;
        bCanPlayersJump = false;

        // 销毁飞机
        Destroy();
    }
}

void ALyraBRPlaneActor::PlayerJump_Implementation(APlayerController* PlayerController)
{
    if (!HasAuthority() || !PlayerController || !bCanPlayersJump) return;

    if (PlayersOnPlane.Contains(PlayerController))
    {
        PlayersOnPlane.Remove(PlayerController);

        APawn* PlayerPawn = PlayerController->GetPawn();
        if (PlayerPawn)
        {
            // 分离玩家
            PlayerPawn->DetachFromActor(FDetachmentTransformRules::KeepWorldTransform);
            PlayerPawn->SetActorHiddenInGame(false);

            // 设置跳伞位置（飞机后方稍微偏移）
            FVector EjectLocation = GetActorLocation() - GetActorForwardVector() * 500.0f;
            EjectPlayer(PlayerController, EjectLocation);
        }
    }
}

void ALyraBRPlaneActor::EjectPlayer(APlayerController* PlayerController, const FVector& EjectLocation)
{
    APawn* PlayerPawn = PlayerController->GetPawn();
    if (!PlayerPawn) return;

    // 设置位置
    PlayerPawn->SetActorLocation(EjectLocation);

    // 恢复玩家控制
    PlayerController->SetIgnoreMoveInput(false);

    // 给予初始速度（向下和向前）
    if (UPrimitiveComponent* RootPrimitive = Cast<UPrimitiveComponent>(PlayerPawn->GetRootComponent()))
    {
        FVector InitialVelocity = GetActorForwardVector() * FlightSpeed * 0.5f + FVector(0, 0, -1000.0f);
        RootPrimitive->SetPhysicsLinearVelocity(InitialVelocity);
    }

    // 启用跳伞状态（通过 GameplayTag 或自定义组件）
    // 这里需要与你的角色系统集成
}
```

### 2.2 跳伞状态组件

```cpp
// LyraBRParachuteComponent.h
#pragma once

#include "Components/ActorComponent.h"
#include "GameplayTagContainer.h"
#include "LyraBRParachuteComponent.generated.h"

/**
 * 跳伞组件
 * 处理玩家的跳伞和降落逻辑
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class ULyraBRParachuteComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    ULyraBRParachuteComponent();

    virtual void TickComponent(float DeltaTime, ELevelTick TickType, 
                              FActorComponentTickFunction* ThisTickFunction) override;

    // 打开降落伞
    UFUNCTION(BlueprintCallable, Category = "Parachute")
    void OpenParachute();

    // 关闭降落伞
    UFUNCTION(BlueprintCallable, Category = "Parachute")
    void CloseParachute();

    // 是否正在跳伞
    UFUNCTION(BlueprintPure, Category = "Parachute")
    bool IsParachuteOpen() const { return bParachuteOpen; }

protected:
    virtual void BeginPlay() override;

    // 降落伞网格体
    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    TObjectPtr<UStaticMeshComponent> ParachuteMesh;

    // 降落伞参数
    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    float ParachuteDragCoefficient = 0.95f;

    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    float MaxFallSpeed = 500.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    float ForwardSpeedMultiplier = 800.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    float MinHeightToOpen = 50000.0f; // 500 米

    UPROPERTY(EditDefaultsOnly, Category = "Parachute")
    float AutoOpenHeight = 20000.0f; // 200 米自动开伞

    UPROPERTY(Replicated)
    bool bParachuteOpen;

    UPROPERTY(Replicated)
    bool bFreeFalling;

private:
    void ApplyParachutePhysics(float DeltaTime);
    void CheckAutoOpen();
};
```

**实现文件（LyraBRParachuteComponent.cpp）**：

```cpp
#include "LyraBRParachuteComponent.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Components/StaticMeshComponent.h"
#include "Net/UnrealNetwork.h"

ULyraBRParachuteComponent::ULyraBRParachuteComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
    SetIsReplicatedByDefault(true);

    bParachuteOpen = false;
    bFreeFalling = true;
}

void ULyraBRParachuteComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ULyraBRParachuteComponent, bParachuteOpen);
    DOREPLIFETIME(ULyraBRParachuteComponent, bFreeFalling);
}

void ULyraBRParachuteComponent::BeginPlay()
{
    Super::BeginPlay();

    // 创建降落伞网格体
    ACharacter* OwnerCharacter = Cast<ACharacter>(GetOwner());
    if (OwnerCharacter)
    {
        ParachuteMesh = NewObject<UStaticMeshComponent>(OwnerCharacter, TEXT("ParachuteMesh"));
        ParachuteMesh->RegisterComponent();
        ParachuteMesh->AttachToComponent(OwnerCharacter->GetMesh(), 
            FAttachmentTransformRules::SnapToTargetIncludingScale, TEXT("ParachuteSocket"));
        ParachuteMesh->SetVisibility(false);
    }
}

void ULyraBRParachuteComponent::TickComponent(float DeltaTime, ELevelTick TickType,
                                             FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    ACharacter* OwnerCharacter = Cast<ACharacter>(GetOwner());
    if (!OwnerCharacter) return;

    // 检查是否在空中
    UCharacterMovementComponent* MovementComp = OwnerCharacter->GetCharacterMovement();
    if (!MovementComp) return;

    bFreeFalling = MovementComp->IsFalling();

    if (bFreeFalling)
    {
        CheckAutoOpen();

        if (bParachuteOpen)
        {
            ApplyParachutePhysics(DeltaTime);
        }
    }
    else if (bParachuteOpen)
    {
        // 着陆后自动关闭降落伞
        CloseParachute();
    }
}

void ULyraBRParachuteComponent::OpenParachute()
{
    if (bParachuteOpen || !bFreeFalling) return;

    ACharacter* OwnerCharacter = Cast<ACharacter>(GetOwner());
    if (!OwnerCharacter) return;

    // 检查高度
    FVector Location = OwnerCharacter->GetActorLocation();
    FHitResult HitResult;
    FVector TraceEnd = Location - FVector(0, 0, 100000.0f);
    
    if (GetWorld()->LineTraceSingleByChannel(HitResult, Location, TraceEnd, ECC_Visibility))
    {
        float HeightAboveGround = Location.Z - HitResult.Location.Z;
        if (HeightAboveGround < MinHeightToOpen)
        {
            return; // 太低无法开伞
        }
    }

    bParachuteOpen = true;

    if (ParachuteMesh)
    {
        ParachuteMesh->SetVisibility(true);
    }

    // 修改移动模式
    UCharacterMovementComponent* MovementComp = OwnerCharacter->GetCharacterMovement();
    if (MovementComp)
    {
        MovementComp->GravityScale = 0.3f;
        MovementComp->AirControl = 0.8f;
    }
}

void ULyraBRParachuteComponent::CloseParachute()
{
    if (!bParachuteOpen) return;

    bParachuteOpen = false;

    if (ParachuteMesh)
    {
        ParachuteMesh->SetVisibility(false);
    }

    // 恢复正常移动
    ACharacter* OwnerCharacter = Cast<ACharacter>(GetOwner());
    if (OwnerCharacter)
    {
        UCharacterMovementComponent* MovementComp = OwnerCharacter->GetCharacterMovement();
        if (MovementComp)
        {
            MovementComp->GravityScale = 1.0f;
            MovementComp->AirControl = 0.05f;
        }
    }
}

void ULyraBRParachuteComponent::ApplyParachutePhysics(float DeltaTime)
{
    ACharacter* OwnerCharacter = Cast<ACharacter>(GetOwner());
    if (!OwnerCharacter) return;

    UCharacterMovementComponent* MovementComp = OwnerCharacter->GetCharacterMovement();
    if (!MovementComp) return;

    FVector Velocity = MovementComp->Velocity;

    // 限制下降速度
    if (Velocity.Z < -MaxFallSpeed)
    {
        Velocity.Z = FMath::Lerp(Velocity.Z, -MaxFallSpeed, ParachuteDragCoefficient * DeltaTime);
    }

    // 应用水平控制
    FVector InputVector = OwnerCharacter->GetLastMovementInputVector();
    if (!InputVector.IsNearlyZero())
    {
        FVector HorizontalVelocity = FVector(Velocity.X, Velocity.Y, 0);
        HorizontalVelocity += InputVector * ForwardSpeedMultiplier * DeltaTime;
        HorizontalVelocity = HorizontalVelocity.GetClampedToMaxSize(ForwardSpeedMultiplier);
        
        Velocity.X = HorizontalVelocity.X;
        Velocity.Y = HorizontalVelocity.Y;
    }

    MovementComp->Velocity = Velocity;
}

void ULyraBRParachuteComponent::CheckAutoOpen()
{
    if (bParachuteOpen) return;

    ACharacter* OwnerCharacter = Cast<ACharacter>(GetOwner());
    if (!OwnerCharacter) return;

    FVector Location = OwnerCharacter->GetActorLocation();
    FHitResult HitResult;
    FVector TraceEnd = Location - FVector(0, 0, 100000.0f);
    
    if (GetWorld()->LineTraceSingleByChannel(HitResult, Location, TraceEnd, ECC_Visibility))
    {
        float HeightAboveGround = Location.Z - HitResult.Location.Z;
        if (HeightAboveGround <= AutoOpenHeight)
        {
            OpenParachute();
        }
    }
}
```

## 三、安全区系统

### 3.1 安全区管理器

```cpp
// LyraBRSafeZoneManager.h
#pragma once

#include "GameFramework/Actor.h"
#include "LyraBRSafeZoneManager.generated.h"

USTRUCT(BlueprintType)
struct FSafeZonePhaseConfig
{
    GENERATED_BODY()

    // 阶段持续时间（秒）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float Duration = 180.0f;

    // 收缩到的半径（厘米）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float Radius = 500000.0f;

    // 每秒伤害
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float DamagePerSecond = 1.0f;
};

/**
 * 安全区管理器
 * 控制安全区的收缩和伤害
 */
UCLASS()
class ALyraBRSafeZoneManager : public AActor
{
    GENERATED_BODY()

public:
    ALyraBRSafeZoneManager();

    virtual void Tick(float DeltaTime) override;
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 开始安全区收缩
    UFUNCTION(BlueprintCallable, Category = "Safe Zone")
    void StartSafeZoneShrink();

    // 获取当前安全区中心
    UFUNCTION(BlueprintPure, Category = "Safe Zone")
    FVector GetCurrentCenter() const { return CurrentCenter; }

    // 获取当前安全区半径
    UFUNCTION(BlueprintPure, Category = "Safe Zone")
    float GetCurrentRadius() const { return CurrentRadius; }

    // 获取目标安全区中心
    UFUNCTION(BlueprintPure, Category = "Safe Zone")
    FVector GetTargetCenter() const { return TargetCenter; }

    // 获取目标安全区半径
    UFUNCTION(BlueprintPure, Category = "Safe Zone")
    float GetTargetRadius() const { return TargetRadius; }

    // 检查位置是否在安全区内
    UFUNCTION(BlueprintPure, Category = "Safe Zone")
    bool IsLocationSafe(const FVector& Location) const;

protected:
    virtual void BeginPlay() override;

    // 安全区阶段配置
    UPROPERTY(EditDefaultsOnly, Category = "Safe Zone")
    TArray<FSafeZonePhaseConfig> PhaseConfigs;

    // 当前阶段索引
    UPROPERTY(Replicated)
    int32 CurrentPhaseIndex;

    // 当前安全区参数
    UPROPERTY(ReplicatedUsing = OnRep_CurrentCenter)
    FVector CurrentCenter;

    UPROPERTY(ReplicatedUsing = OnRep_CurrentRadius)
    float CurrentRadius;

    // 目标安全区参数
    UPROPERTY(Replicated)
    FVector TargetCenter;

    UPROPERTY(Replicated)
    float TargetRadius;

    // 阶段开始时间
    UPROPERTY(Replicated)
    float PhaseStartTime;

    // 是否正在收缩
    UPROPERTY(Replicated)
    bool bIsShrinking;

    // 当前阶段伤害
    UPROPERTY(Replicated)
    float CurrentDamagePerSecond;

    // 可视化组件
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UStaticMeshComponent> SafeZoneVisualization;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UStaticMeshComponent> TargetZoneVisualization;

private:
    void StartNextPhase();
    void UpdateSafeZone(float DeltaTime);
    void ApplyDamageToPlayersOutside();
    FVector CalculateRandomCenterInCircle(const FVector& OldCenter, float OldRadius, float NewRadius);

    UFUNCTION()
    void OnRep_CurrentCenter();

    UFUNCTION()
    void OnRep_CurrentRadius();

    FTimerHandle DamageTimerHandle;
};
```

**实现文件（LyraBRSafeZoneManager.cpp）**：

```cpp
#include "LyraBRSafeZoneManager.h"
#include "Components/StaticMeshComponent.h"
#include "Net/UnrealNetwork.h"
#include "GameFramework/Character.h"
#include "LyraHealthComponent.h"
#include "Kismet/GameplayStatics.h"

ALyraBRSafeZoneManager::ALyraBRSafeZoneManager()
{
    PrimaryActorTick.bCanEverTick = true;
    bReplicates = true;
    bAlwaysRelevant = true;

    // 创建可视化组件
    SafeZoneVisualization = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("SafeZoneVisualization"));
    RootComponent = SafeZoneVisualization;
    SafeZoneVisualization->SetCollisionEnabled(ECollisionEnabled::NoCollision);

    TargetZoneVisualization = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("TargetZoneVisualization"));
    TargetZoneVisualization->SetupAttachment(RootComponent);
    TargetZoneVisualization->SetCollisionEnabled(ECollisionEnabled::NoCollision);

    CurrentPhaseIndex = -1;
    CurrentRadius = 1000000.0f; // 初始半径 10km
    CurrentCenter = FVector::ZeroVector;
    bIsShrinking = false;
}

void ALyraBRSafeZoneManager::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ALyraBRSafeZoneManager, CurrentPhaseIndex);
    DOREPLIFETIME(ALyraBRSafeZoneManager, CurrentCenter);
    DOREPLIFETIME(ALyraBRSafeZoneManager, CurrentRadius);
    DOREPLIFETIME(ALyraBRSafeZoneManager, TargetCenter);
    DOREPLIFETIME(ALyraBRSafeZoneManager, TargetRadius);
    DOREPLIFETIME(ALyraBRSafeZoneManager, PhaseStartTime);
    DOREPLIFETIME(ALyraBRSafeZoneManager, bIsShrinking);
    DOREPLIFETIME(ALyraBRSafeZoneManager, CurrentDamagePerSecond);
}

void ALyraBRSafeZoneManager::BeginPlay()
{
    Super::BeginPlay();

    // 设置初始位置（地图中心）
    SetActorLocation(FVector::ZeroVector);
}

void ALyraBRSafeZoneManager::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (HasAuthority() && bIsShrinking)
    {
        UpdateSafeZone(DeltaTime);
    }
}

void ALyraBRSafeZoneManager::StartSafeZoneShrink()
{
    if (!HasAuthority() || PhaseConfigs.Num() == 0) return;

    bIsShrinking = true;
    StartNextPhase();

    // 启动伤害计时器
    GetWorldTimerManager().SetTimer(DamageTimerHandle, this, 
        &ALyraBRSafeZoneManager::ApplyDamageToPlayersOutside, 1.0f, true);
}

void ALyraBRSafeZoneManager::StartNextPhase()
{
    CurrentPhaseIndex++;

    if (CurrentPhaseIndex >= PhaseConfigs.Num())
    {
        // 所有阶段完成
        bIsShrinking = false;
        GetWorldTimerManager().ClearTimer(DamageTimerHandle);
        return;
    }

    const FSafeZonePhaseConfig& PhaseConfig = PhaseConfigs[CurrentPhaseIndex];

    // 设置目标参数
    TargetRadius = PhaseConfig.Radius;
    TargetCenter = CalculateRandomCenterInCircle(CurrentCenter, CurrentRadius, TargetRadius);
    CurrentDamagePerSecond = PhaseConfig.DamagePerSecond;
    PhaseStartTime = GetWorld()->GetTimeSeconds();

    UE_LOG(LogTemp, Log, TEXT("Safe Zone Phase %d started. Target Radius: %.2f"), 
           CurrentPhaseIndex, TargetRadius);
}

void ALyraBRSafeZoneManager::UpdateSafeZone(float DeltaTime)
{
    if (CurrentPhaseIndex < 0 || CurrentPhaseIndex >= PhaseConfigs.Num()) return;

    const FSafeZonePhaseConfig& PhaseConfig = PhaseConfigs[CurrentPhaseIndex];
    float ElapsedTime = GetWorld()->GetTimeSeconds() - PhaseStartTime;
    float Progress = FMath::Clamp(ElapsedTime / PhaseConfig.Duration, 0.0f, 1.0f);

    // 线性插值当前位置和半径
    CurrentCenter = FMath::Lerp(CurrentCenter, TargetCenter, Progress);
    CurrentRadius = FMath::Lerp(CurrentRadius, TargetRadius, Progress);

    // 阶段完成
    if (Progress >= 1.0f)
    {
        CurrentCenter = TargetCenter;
        CurrentRadius = TargetRadius;
        StartNextPhase();
    }
}

void ALyraBRSafeZoneManager::ApplyDamageToPlayersOutside()
{
    if (!HasAuthority()) return;

    TArray<AActor*> AllCharacters;
    UGameplayStatics::GetAllActorsOfClass(GetWorld(), ACharacter::StaticClass(), AllCharacters);

    for (AActor* Actor : AllCharacters)
    {
        ACharacter* Character = Cast<ACharacter>(Actor);
        if (!Character || Character->IsPendingKillPending()) continue;

        FVector CharacterLocation = Character->GetActorLocation();
        
        if (!IsLocationSafe(CharacterLocation))
        {
            // 对角色造成伤害
            ULyraHealthComponent* HealthComp = ULyraHealthComponent::FindHealthComponent(Character);
            if (HealthComp)
            {
                // 使用 Gameplay Effect 造成伤害（这里简化处理）
                float CurrentHealth = HealthComp->GetHealth();
                float NewHealth = FMath::Max(0.0f, CurrentHealth - CurrentDamagePerSecond);
                
                // 这里应该通过 GAS 系统正确应用伤害
                // 示例代码仅作说明
                if (NewHealth <= 0.0f)
                {
                    HealthComp->StartDeath();
                }
            }
        }
    }
}

FVector ALyraBRSafeZoneManager::CalculateRandomCenterInCircle(const FVector& OldCenter, 
                                                              float OldRadius, float NewRadius)
{
    // 确保新圆心在旧圆内，并且新圆完全包含在旧圆内
    float MaxOffset = OldRadius - NewRadius;
    if (MaxOffset <= 0.0f) return OldCenter;

    // 随机偏移
    float RandomAngle = FMath::RandRange(0.0f, 360.0f);
    float RandomDistance = FMath::RandRange(0.0f, MaxOffset * 0.7f); // 70% 以保证有趣的偏移

    FVector2D Offset;
    Offset.X = FMath::Cos(FMath::DegreesToRadians(RandomAngle)) * RandomDistance;
    Offset.Y = FMath::Sin(FMath::DegreesToRadians(RandomAngle)) * RandomDistance;

    return OldCenter + FVector(Offset.X, Offset.Y, 0);
}

bool ALyraBRSafeZoneManager::IsLocationSafe(const FVector& Location) const
{
    float Distance = FVector::Dist2D(Location, CurrentCenter);
    return Distance <= CurrentRadius;
}

void ALyraBRSafeZoneManager::OnRep_CurrentCenter()
{
    // 更新可视化
    SafeZoneVisualization->SetWorldLocation(CurrentCenter);
}

void ALyraBRSafeZoneManager::OnRep_CurrentRadius()
{
    // 更新可视化大小
    float Scale = CurrentRadius / 100.0f; // 假设基础网格是 100cm
    SafeZoneVisualization->SetWorldScale3D(FVector(Scale, Scale, 1.0f));
}
```

## 四、拾取和装备系统

### 4.1 战利品生成子系统

```cpp
// LyraBRLootSpawnSubsystem.h
#pragma once

#include "Subsystems/WorldSubsystem.h"
#include "LyraBRLootSpawnSubsystem.generated.h"

USTRUCT(BlueprintType)
struct FLootSpawnPoint
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FVector Location;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FRotator Rotation;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FName> Tags;
};

USTRUCT(BlueprintType)
struct FLootTableEntry
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TSubclassOf<class ULyraInventoryItemDefinition> ItemDefinition;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float SpawnWeight = 1.0f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    int32 MinStack = 1;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    int32 MaxStack = 1;
};

/**
 * 战利品生成子系统
 * 管理地图上的战利品生成
 */
UCLASS()
class ULyraBRLootSpawnSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 生成所有战利品
    UFUNCTION(BlueprintCallable, Category = "Loot")
    void SpawnAllLoot();

    // 在特定位置生成战利品
    UFUNCTION(BlueprintCallable, Category = "Loot")
    AActor* SpawnLootAtLocation(const FVector& Location, const FRotator& Rotation);

protected:
    // 战利品表
    UPROPERTY(EditDefaultsOnly, Category = "Loot")
    TArray<FLootTableEntry> CommonLootTable;

    UPROPERTY(EditDefaultsOnly, Category = "Loot")
    TArray<FLootTableEntry> RareLootTable;

    UPROPERTY(EditDefaultsOnly, Category = "Loot")
    TArray<FLootTableEntry> EpicLootTable;

    // 生成点
    UPROPERTY()
    TArray<FLootSpawnPoint> SpawnPoints;

    // 生成的战利品 Actor 类
    UPROPERTY(EditDefaultsOnly, Category = "Loot")
    TSubclassOf<class ALyraBRLootActor> LootActorClass;

private:
    void CollectSpawnPoints();
    TSubclassOf<ULyraInventoryItemDefinition> SelectRandomLoot(const TArray<FLootTableEntry>& LootTable);
    int32 GetRandomStackCount(const FLootTableEntry& Entry);
};
```

### 4.2 战利品 Actor

```cpp
// LyraBRLootActor.h
#pragma once

#include "GameFramework/Actor.h"
#include "LyraBRLootActor.generated.h"

/**
 * 可拾取的战利品 Actor
 */
UCLASS()
class ALyraBRLootActor : public AActor
{
    GENERATED_BODY()

public:
    ALyraBRLootActor();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 设置战利品内容
    UFUNCTION(BlueprintCallable, Category = "Loot")
    void SetLootItem(TSubclassOf<class ULyraInventoryItemDefinition> ItemDef, int32 StackCount);

    // 拾取战利品
    UFUNCTION(Server, Reliable, BlueprintCallable, Category = "Loot")
    void PickupLoot(AController* Picker);

protected:
    virtual void BeginPlay() override;

    // 网格体组件
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<UStaticMeshComponent> MeshComponent;

    // 交互范围
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<class USphereComponent> InteractionSphere;

    // 战利品内容
    UPROPERTY(ReplicatedUsing = OnRep_ItemDefinition)
    TSubclassOf<class ULyraInventoryItemDefinition> ItemDefinition;

    UPROPERTY(Replicated)
    int32 StackCount;

    UFUNCTION()
    void OnRep_ItemDefinition();

    UFUNCTION()
    void OnInteractionSphereBeginOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor,
        UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

private:
    void UpdateVisuals();
};
```

**实现文件（LyraBRLootActor.cpp）**：

```cpp
#include "LyraBRLootActor.h"
#include "Components/StaticMeshComponent.h"
#include "Components/SphereComponent.h"
#include "Net/UnrealNetwork.h"
#include "Inventory/LyraInventoryManagerComponent.h"
#include "Inventory/LyraInventoryItemDefinition.h"
#include "GameFramework/Character.h"

ALyraBRLootActor::ALyraBRLootActor()
{
    bReplicates = true;

    // 创建网格体
    MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("MeshComponent"));
    RootComponent = MeshComponent;
    MeshComponent->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
    MeshComponent->SetCollisionResponseToAllChannels(ECR_Ignore);
    MeshComponent->SetCollisionResponseToChannel(ECC_WorldStatic, ECR_Block);

    // 创建交互范围
    InteractionSphere = CreateDefaultSubobject<USphereComponent>(TEXT("InteractionSphere"));
    InteractionSphere->SetupAttachment(RootComponent);
    InteractionSphere->SetSphereRadius(200.0f);
    InteractionSphere->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
    InteractionSphere->SetCollisionResponseToAllChannels(ECR_Ignore);
    InteractionSphere->SetCollisionResponseToChannel(ECC_Pawn, ECR_Overlap);

    StackCount = 1;
}

void ALyraBRLootActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ALyraBRLootActor, ItemDefinition);
    DOREPLIFETIME(ALyraBRLootActor, StackCount);
}

void ALyraBRLootActor::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        InteractionSphere->OnComponentBeginOverlap.AddDynamic(this, 
            &ALyraBRLootActor::OnInteractionSphereBeginOverlap);
    }

    UpdateVisuals();
}

void ALyraBRLootActor::SetLootItem(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 Count)
{
    if (HasAuthority())
    {
        ItemDefinition = ItemDef;
        StackCount = Count;
        UpdateVisuals();
    }
}

void ALyraBRLootActor::PickupLoot_Implementation(AController* Picker)
{
    if (!HasAuthority() || !Picker || !ItemDefinition) return;

    APawn* PickerPawn = Picker->GetPawn();
    if (!PickerPawn) return;

    // 获取库存管理组件
    ULyraInventoryManagerComponent* InventoryComp = 
        PickerPawn->FindComponentByClass<ULyraInventoryManagerComponent>();

    if (InventoryComp)
    {
        // 尝试添加到库存
        if (InventoryComp->CanAddItemDefinition(ItemDefinition, StackCount))
        {
            InventoryComp->AddItemDefinition(ItemDefinition, StackCount);

            // 拾取成功，销毁 Actor
            Destroy();
        }
    }
}

void ALyraBRLootActor::OnRep_ItemDefinition()
{
    UpdateVisuals();
}

void ALyraBRLootActor::OnInteractionSphereBeginOverlap(UPrimitiveComponent* OverlappedComponent,
    AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex,
    bool bFromSweep, const FHitResult& SweepResult)
{
    ACharacter* Character = Cast<ACharacter>(OtherActor);
    if (Character)
    {
        AController* Controller = Character->GetController();
        if (Controller)
        {
            // 自动拾取或显示拾取提示
            PickupLoot(Controller);
        }
    }
}

void ALyraBRLootActor::UpdateVisuals()
{
    if (!ItemDefinition) return;

    // 根据物品定义更新网格体和材质
    // 这里需要从 ItemDefinition 中获取显示信息
    const ULyraInventoryItemDefinition* ItemDef = ItemDefinition.GetDefaultObject();
    if (ItemDef)
    {
        // 设置网格体（需要在 ItemDefinition 中添加显示信息）
        // MeshComponent->SetStaticMesh(ItemDef->DisplayMesh);
    }
}
```

## 五、库存和背包系统

### 5.1 扩展库存组件

```cpp
// LyraBRInventoryComponent.h
#pragma once

#include "Inventory/LyraInventoryManagerComponent.h"
#include "LyraBRInventoryComponent.generated.h"

/**
 * 大逃杀库存组件
 * 扩展 Lyra 的基础库存系统，添加背包容量限制
 */
UCLASS()
class ULyraBRInventoryComponent : public ULyraInventoryManagerComponent
{
    GENERATED_BODY()

public:
    ULyraBRInventoryComponent();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 检查是否有足够空间
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    bool HasSpaceForItem(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount = 1);

    // 获取当前负重
    UFUNCTION(BlueprintPure, Category = "Inventory")
    float GetCurrentWeight() const { return CurrentWeight; }

    // 获取最大负重
    UFUNCTION(BlueprintPure, Category = "Inventory")
    float GetMaxWeight() const { return MaxWeight; }

    // 丢弃物品
    UFUNCTION(Server, Reliable, BlueprintCallable, Category = "Inventory")
    void DropItem(ULyraInventoryItemInstance* ItemInstance, int32 DropCount);

protected:
    // 背包容量限制
    UPROPERTY(EditDefaultsOnly, Replicated, Category = "Inventory")
    int32 MaxSlots = 30;

    UPROPERTY(EditDefaultsOnly, Replicated, Category = "Inventory")
    float MaxWeight = 100.0f;

    UPROPERTY(Replicated)
    float CurrentWeight;

private:
    void RecalculateWeight();
    float GetItemWeight(const ULyraInventoryItemDefinition* ItemDef) const;
};
```

### 5.2 背包 UI

创建蓝图 Widget：**WBP_BattleRoyaleInventory**

```cpp
// LyraBRInventoryWidget.h
#pragma once

#include "CommonUserWidget.h"
#include "LyraBRInventoryWidget.generated.h"

/**
 * 大逃杀库存 UI
 */
UCLASS(Abstract)
class ULyraBRInventoryWidget : public UCommonUserWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override;
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;

    // 刷新库存显示
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void RefreshInventory();

    // 使用物品
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void UseItem(ULyraInventoryItemInstance* ItemInstance);

    // 丢弃物品
    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void DropItem(ULyraInventoryItemInstance* ItemInstance, int32 Count);

protected:
    // 库存网格容器
    UPROPERTY(BlueprintReadOnly, meta = (BindWidget))
    TObjectPtr<class UGridPanel> InventoryGrid;

    // 装备栏容器
    UPROPERTY(BlueprintReadOnly, meta = (BindWidget))
    TObjectPtr<class UHorizontalBox> EquipmentSlots;

    // 负重文本
    UPROPERTY(BlueprintReadOnly, meta = (BindWidget))
    TObjectPtr<class UTextBlock> WeightText;

    // 物品槽 Widget 类
    UPROPERTY(EditDefaultsOnly, Category = "Inventory")
    TSubclassOf<class ULyraBRItemSlotWidget> ItemSlotWidgetClass;

    UPROPERTY()
    TObjectPtr<ULyraBRInventoryComponent> InventoryComponent;

private:
    void CreateItemSlots();
    void UpdateWeightDisplay();
};
```

## 六、载具系统集成

### 6.1 载具基类

```cpp
// LyraBRVehicle.h
#pragma once

#include "GameFramework/WheeledVehiclePawn.h"
#include "LyraBRVehicle.generated.h"

/**
 * 大逃杀载具基类
 */
UCLASS(Abstract)
class ALyraBRVehicle : public AWheeledVehiclePawn
{
    GENERATED_BODY()

public:
    ALyraBRVehicle();

    virtual void Tick(float DeltaTime) override;
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 进入载具
    UFUNCTION(Server, Reliable, BlueprintCallable, Category = "Vehicle")
    void EnterVehicle(ACharacter* Character, int32 SeatIndex = 0);

    // 离开载具
    UFUNCTION(Server, Reliable, BlueprintCallable, Category = "Vehicle")
    void ExitVehicle(ACharacter* Character);

    // 获取可用座位
    UFUNCTION(BlueprintPure, Category = "Vehicle")
    TArray<int32> GetAvailableSeats() const;

protected:
    virtual void BeginPlay() override;

    // 座位配置
    UPROPERTY(EditDefaultsOnly, Category = "Vehicle")
    int32 MaxSeats = 4;

    // 当前乘客
    UPROPERTY(Replicated)
    TArray<TObjectPtr<ACharacter>> Passengers;

    // 载具血量
    UPROPERTY(EditDefaultsOnly, Replicated, Category = "Vehicle")
    float MaxHealth = 1000.0f;

    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float CurrentHealth;

    UFUNCTION()
    void OnRep_Health();

    // 载具交互范围
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    TObjectPtr<class USphereComponent> InteractionSphere;

private:
    void AttachPassengerToSeat(ACharacter* Character, int32 SeatIndex);
    void DetachPassengerFromSeat(ACharacter* Character);
    FTransform GetSeatTransform(int32 SeatIndex) const;
};
```

**载具生成管理器**：

```cpp
// LyraBRVehicleSpawnManager.h
#pragma once

#include "GameFramework/Actor.h"
#include "LyraBRVehicleSpawnManager.generated.h"

USTRUCT(BlueprintType)
struct FVehicleSpawnConfig
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<class ALyraBRVehicle> VehicleClass;

    UPROPERTY(EditDefaultsOnly)
    int32 SpawnCount = 10;

    UPROPERTY(EditDefaultsOnly)
    float SpawnWeight = 1.0f;
};

/**
 * 载具生成管理器
 */
UCLASS()
class ALyraBRVehicleSpawnManager : public AActor
{
    GENERATED_BODY()

public:
    ALyraBRVehicleSpawnManager();

    // 生成所有载具
    UFUNCTION(BlueprintCallable, Category = "Vehicle")
    void SpawnAllVehicles();

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Vehicle")
    TArray<FVehicleSpawnConfig> VehicleConfigs;

private:
    void CollectSpawnLocations();
    TArray<FVector> SpawnLocations;
};
```

## 七、观战系统

### 7.1 观战 Player Controller

```cpp
// LyraBRSpectatorController.h
#pragma once

#include "GameFramework/PlayerController.h"
#include "LyraBRSpectatorController.generated.h"

/**
 * 观战控制器
 * 玩家死亡后用于观看其他玩家
 */
UCLASS()
class ALyraBRSpectatorController : public APlayerController
{
    GENERATED_BODY()

public:
    ALyraBRSpectatorController();

    virtual void SetupInputComponent() override;

    // 切换到下一个玩家
    UFUNCTION(BlueprintCallable, Category = "Spectator")
    void SpectateNextPlayer();

    // 切换到上一个玩家
    UFUNCTION(BlueprintCallable, Category = "Spectator")
    void SpectatePreviousPlayer();

    // 观看特定玩家
    UFUNCTION(Server, Reliable, BlueprintCallable, Category = "Spectator")
    void SpectatePlayer(APlayerState* TargetPlayerState);

    // 获取当前观看的玩家
    UFUNCTION(BlueprintPure, Category = "Spectator")
    APlayerState* GetSpectatedPlayerState() const { return CurrentSpectatedPlayerState; }

protected:
    // 当前观看的玩家
    UPROPERTY(Replicated)
    TObjectPtr<APlayerState> CurrentSpectatedPlayerState;

    // 可观看的玩家列表
    UPROPERTY()
    TArray<TObjectPtr<APlayerState>> SpectateablePlayerStates;

private:
    void UpdateSpectateableList();
    void SetSpectatedPlayer(APlayerState* PlayerState);
    int32 GetCurrentSpectateIndex() const;
};
```

**实现文件（LyraBRSpectatorController.cpp）**：

```cpp
#include "LyraBRSpectatorController.h"
#include "GameFramework/PlayerState.h"
#include "GameFramework/GameStateBase.h"
#include "GameFramework/SpectatorPawn.h"
#include "Net/UnrealNetwork.h"

ALyraBRSpectatorController::ALyraBRSpectatorController()
{
    bShowMouseCursor = false;
}

void ALyraBRSpectatorController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ALyraBRSpectatorController, CurrentSpectatedPlayerState);
}

void ALyraBRSpectatorController::SetupInputComponent()
{
    Super::SetupInputComponent();

    // 绑定观战输入
    InputComponent->BindAction("SpectateNext", IE_Pressed, this, &ALyraBRSpectatorController::SpectateNextPlayer);
    InputComponent->BindAction("SpectatePrevious", IE_Pressed, this, &ALyraBRSpectatorController::SpectatePreviousPlayer);
}

void ALyraBRSpectatorController::SpectateNextPlayer()
{
    UpdateSpectateableList();

    if (SpectateablePlayerStates.Num() == 0) return;

    int32 CurrentIndex = GetCurrentSpectateIndex();
    int32 NextIndex = (CurrentIndex + 1) % SpectateablePlayerStates.Num();

    SpectatePlayer(SpectateablePlayerStates[NextIndex]);
}

void ALyraBRSpectatorController::SpectatePreviousPlayer()
{
    UpdateSpectateableList();

    if (SpectateablePlayerStates.Num() == 0) return;

    int32 CurrentIndex = GetCurrentSpectateIndex();
    int32 PrevIndex = (CurrentIndex - 1 + SpectateablePlayerStates.Num()) % SpectateablePlayerStates.Num();

    SpectatePlayer(SpectateablePlayerStates[PrevIndex]);
}

void ALyraBRSpectatorController::SpectatePlayer_Implementation(APlayerState* TargetPlayerState)
{
    if (!TargetPlayerState) return;

    SetSpectatedPlayer(TargetPlayerState);
}

void ALyraBRSpectatorController::UpdateSpectateableList()
{
    SpectateablePlayerStates.Empty();

    AGameStateBase* GameState = GetWorld()->GetGameState();
    if (!GameState) return;

    for (APlayerState* PS : GameState->PlayerArray)
    {
        if (PS && PS->GetPawn() && !PS->GetPawn()->IsPendingKillPending())
        {
            // 只添加存活的玩家
            SpectateablePlayerStates.Add(PS);
        }
    }
}

void ALyraBRSpectatorController::SetSpectatedPlayer(APlayerState* PlayerState)
{
    if (!PlayerState) return;

    CurrentSpectatedPlayerState = PlayerState;

    APawn* TargetPawn = PlayerState->GetPawn();
    if (TargetPawn)
    {
        SetViewTarget(TargetPawn);
    }
}

int32 ALyraBRSpectatorController::GetCurrentSpectateIndex() const
{
    if (!CurrentSpectatedPlayerState) return 0;

    return SpectateablePlayerStates.IndexOfByKey(CurrentSpectatedPlayerState);
}
```

## 八、网络优化：Replication Graph

### 8.1 自定义 Replication Graph

```cpp
// LyraBRReplicationGraph.h
#pragma once

#include "ReplicationGraph.h"
#include "LyraBRReplicationGraph.generated.h"

/**
 * 大逃杀模式的网络复制图
 * 优化 100 玩家同时在线的网络性能
 */
UCLASS(Transient, Config=Engine)
class ULyraBRReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()

public:
    ULyraBRReplicationGraph();

    virtual void InitGlobalActorClassSettings() override;
    virtual void InitGlobalGraphNodes() override;
    virtual void InitConnectionGraphNodes(UNetReplicationGraphConnection* ConnectionManager) override;
    virtual void RouteAddNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo, FGlobalActorReplicationInfo& GlobalInfo) override;
    virtual void RouteRemoveNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo) override;

protected:
    // 全局始终相关节点（GameState 等）
    UPROPERTY()
    TObjectPtr<class UReplicationGraphNode_ActorList> AlwaysRelevantNode;

    // 玩家状态节点
    UPROPERTY()
    TObjectPtr<class UReplicationGraphNode_ActorList> PlayerStateNode;

    // 空间分区节点（用于优化位置相关的复制）
    UPROPERTY()
    TObjectPtr<class UReplicationGraphNode_GridSpatialization2D> GridNode;

    // 每个连接的特定节点映射
    TMap<UNetConnection*, TObjectPtr<class ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection>> ConnectionToAlwaysRelevantNodeMap;

private:
    void SetupClassReplicationInfo();
};

/**
 * 每个连接的始终相关节点
 * 包含玩家自己的 Pawn、控制器等
 */
UCLASS()
class ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection : public UReplicationGraphNode
{
    GENERATED_BODY()

public:
    virtual void GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params) override;
    virtual void OnCollectActorRepListStats(struct FActorRepListStatCollector& StatCollector) const override;

    void AddAlwaysRelevantActor(AActor* Actor);
    void RemoveAlwaysRelevantActor(AActor* Actor);

protected:
    UPROPERTY()
    TArray<TObjectPtr<AActor>> AlwaysRelevantActors;
};
```

**实现文件（LyraBRReplicationGraph.cpp）**：

```cpp
#include "LyraBRReplicationGraph.h"
#include "Net/UnrealNetwork.h"
#include "Engine/World.h"
#include "GameFramework/PlayerController.h"
#include "GameFramework/PlayerState.h"
#include "GameFramework/Pawn.h"
#include "ReplicationGraphNode_ActorList.h"
#include "ReplicationGraphNode_GridSpatialization.h"

ULyraBRReplicationGraph::ULyraBRReplicationGraph()
{
}

void ULyraBRReplicationGraph::InitGlobalActorClassSettings()
{
    Super::InitGlobalActorClassSettings();

    SetupClassReplicationInfo();
}

void ULyraBRReplicationGraph::InitGlobalGraphNodes()
{
    Super::InitGlobalGraphNodes();

    // 创建全局始终相关节点
    AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_ActorList>();
    AddGlobalGraphNode(AlwaysRelevantNode);

    // 创建 PlayerState 节点
    PlayerStateNode = CreateNewNode<UReplicationGraphNode_ActorList>();
    AddGlobalGraphNode(PlayerStateNode);

    // 创建空间分区节点（用于位置相关的 Actor）
    GridNode = CreateNewNode<UReplicationGraphNode_GridSpatialization2D>();
    GridNode->CellSize = 10000.0f; // 100 米格子
    GridNode->SpatialBias = FVector2D(0.0f, 0.0f);
    AddGlobalGraphNode(GridNode);
}

void ULyraBRReplicationGraph::InitConnectionGraphNodes(UNetReplicationGraphConnection* ConnectionManager)
{
    Super::InitConnectionGraphNodes(ConnectionManager);

    // 为每个连接创建特定的始终相关节点
    ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection* AlwaysRelevantForConnection = 
        CreateNewNode<ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection>();

    ConnectionToAlwaysRelevantNodeMap.Add(ConnectionManager->NetConnection, AlwaysRelevantForConnection);
    ConnectionManager->OnClientVisibleLevelNamesAdd.AddUObject(AlwaysRelevantForConnection, 
        &UReplicationGraphNode::NotifyAddNetworkActor);
}

void ULyraBRReplicationGraph::RouteAddNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo, 
                                                          FGlobalActorReplicationInfo& GlobalInfo)
{
    AActor* Actor = ActorInfo.Actor;

    // GameState 始终复制给所有人
    if (Actor->IsA(AGameStateBase::StaticClass()))
    {
        AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
        return;
    }

    // PlayerState 复制给所有人（用于排行榜等）
    if (Actor->IsA(APlayerState::StaticClass()))
    {
        PlayerStateNode->NotifyAddNetworkActor(ActorInfo);
        return;
    }

    // PlayerController 和 Pawn 添加到所属连接的特定节点
    if (Actor->IsA(APlayerController::StaticClass()) || Actor->IsA(APawn::StaticClass()))
    {
        if (APlayerController* PC = Cast<APlayerController>(Actor))
        {
            if (UNetConnection* NetConnection = PC->GetNetConnection())
            {
                if (ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection** Node = 
                    ConnectionToAlwaysRelevantNodeMap.Find(NetConnection))
                {
                    (*Node)->AddAlwaysRelevantActor(Actor);
                }
            }
        }
        return;
    }

    // 其他 Actor 添加到空间分区节点
    GridNode->AddActor_Dormancy(ActorInfo, GlobalInfo);
}

void ULyraBRReplicationGraph::RouteRemoveNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo)
{
    AActor* Actor = ActorInfo.Actor;

    if (Actor->IsA(AGameStateBase::StaticClass()))
    {
        AlwaysRelevantNode->NotifyRemoveNetworkActor(ActorInfo);
    }
    else if (Actor->IsA(APlayerState::StaticClass()))
    {
        PlayerStateNode->NotifyRemoveNetworkActor(ActorInfo);
    }
    else
    {
        GridNode->RemoveActor_Dormancy(ActorInfo);
    }
}

void ULyraBRReplicationGraph::SetupClassReplicationInfo()
{
    // 配置不同类的复制频率
    // 玩家角色：高频率
    TArray<UClass*> HighFrequencyClasses = { APawn::StaticClass(), ACharacter::StaticClass() };
    for (UClass* Class : HighFrequencyClasses)
    {
        FClassReplicationInfo ClassInfo;
        ClassInfo.ReplicationPeriodFrame = 1; // 每帧复制
        GlobalActorReplicationInfoMap.SetClassInfo(Class, ClassInfo);
    }

    // 战利品：低频率
    FClassReplicationInfo LootClassInfo;
    LootClassInfo.ReplicationPeriodFrame = 10; // 每 10 帧复制一次
    GlobalActorReplicationInfoMap.SetClassInfo(ALyraBRLootActor::StaticClass(), LootClassInfo);

    // 载具：中等频率
    FClassReplicationInfo VehicleClassInfo;
    VehicleClassInfo.ReplicationPeriodFrame = 3; // 每 3 帧复制一次
    GlobalActorReplicationInfoMap.SetClassInfo(ALyraBRVehicle::StaticClass(), VehicleClassInfo);
}

// ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection 实现

void ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection::GatherActorListsForConnection(
    const FConnectionGatherActorListParameters& Params)
{
    for (AActor* Actor : AlwaysRelevantActors)
    {
        if (Actor && !Actor->IsPendingKillPending())
        {
            Params.OutGatheredReplicationLists.AddReplicationActorList(Actor);
        }
    }
}

void ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection::OnCollectActorRepListStats(
    struct FActorRepListStatCollector& StatCollector) const
{
    StatCollector.AddList(AlwaysRelevantActors.Num());
}

void ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection::AddAlwaysRelevantActor(AActor* Actor)
{
    AlwaysRelevantActors.AddUnique(Actor);
}

void ULyraBRReplicationGraphNode_AlwaysRelevant_ForConnection::RemoveAlwaysRelevantActor(AActor* Actor)
{
    AlwaysRelevantActors.Remove(Actor);
}
```

### 8.2 配置 Replication Graph

在项目配置文件 **DefaultEngine.ini** 中：

```ini
[/Script/OnlineSubsystemUtils.IpNetDriver]
ReplicationDriverClassName="/Script/YourProject.LyraBRReplicationGraph"

[/Script/Engine.ReplicationGraph]
; 空间分区设置
GridSizeX=20
GridSizeY=20

; 连接设置
MaxConnectionsForSpatialRelevancy=100

; 距离裁剪
NetCullDistanceSquared=225000000.0 ; 15000cm = 150m
```

## 九、性能优化

### 9.1 LOD（细节层次）配置

```cpp
// LyraBRLODManager.h
#pragma once

#include "Subsystems/WorldSubsystem.h"
#include "LyraBRLODManager.generated.h"

/**
 * LOD 管理子系统
 * 根据玩家距离动态调整角色和载具的细节层次
 */
UCLASS()
class ULyraBRLODManager : public UTickableWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Tick(float DeltaTime) override;
    virtual TStatId GetStatId() const override;

protected:
    // LOD 距离阈值
    UPROPERTY(Config)
    float LOD0Distance = 5000.0f; // 50m

    UPROPERTY(Config)
    float LOD1Distance = 10000.0f; // 100m

    UPROPERTY(Config)
    float LOD2Distance = 20000.0f; // 200m

private:
    void UpdateCharacterLODs();
    void UpdateVehicleLODs();
    int32 CalculateLODLevel(float Distance) const;
};
```

### 9.2 流式加载优化

```cpp
// LyraBRStreamingManager.h
#pragma once

#include "Subsystems/WorldSubsystem.h"
#include "LyraBRStreamingManager.generated.h"

/**
 * 流式加载管理器
 * 根据玩家位置和安全区动态加载/卸载地图区块
 */
UCLASS()
class ULyraBRStreamingManager : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 更新流式加载
    UFUNCTION(BlueprintCallable, Category = "Streaming")
    void UpdateStreaming();

protected:
    // 预加载距离
    UPROPERTY(Config)
    float PreloadDistance = 50000.0f; // 500m

    // 卸载距离
    UPROPERTY(Config)
    float UnloadDistance = 100000.0f; // 1000m

private:
    void LoadLevelsNearPlayers();
    void UnloadDistantLevels();
    TArray<FName> GetLevelsInRadius(const FVector& Center, float Radius);

    FTimerHandle StreamingUpdateTimer;
};
```

### 9.3 渲染优化

在项目设置中配置：

**DefaultEngine.ini**：

```ini
[/Script/Engine.RendererSettings]
; 动态阴影距离
r.Shadow.DistanceScale=0.6

; 级联阴影图距离
r.Shadow.CSM.MaxCascades=3

; 网格体 LOD 距离缩放
r.StaticMeshLODDistanceScale=0.8

; 粒子效果距离裁剪
r.MaxParticleCullDistance=10000

[/Script/Engine.Engine]
; 网络更新频率
NetClientTicksPerSecond=60

[SystemSettings]
; 角色渲染设置
r.SkeletalMeshLODBias=1
r.SkeletalMeshMinLODSize=512

; 纹理设置
r.Streaming.PoolSize=3000
r.Streaming.MaxNumTexturesToStreamPerFrame=10
```

## 十、服务器架构

### 10.1 专用服务器配置

**Target.cs 配置**：

```csharp
// YourProjectServer.Target.cs
using UnrealBuildTool;

public class YourProjectServerTarget : TargetRules
{
    public YourProjectServerTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Server;
        DefaultBuildSettings = BuildSettingsVersion.V2;
        ExtraModuleNames.Add("YourProject");

        bUseLoggingInShipping = true;
        bCompilePhysX = true;
        bCompileAPEX = false;
        bCompileNvCloth = false;
        
        // 性能优化
        bWithServerCode = true;
        bBuildDeveloperTools = false;
        bBuildWithEditorOnlyData = false;
    }
}
```

### 10.2 游戏服务器管理

```cpp
// LyraBRServerManager.h
#pragma once

#include "Subsystems/GameInstanceSubsystem.h"
#include "LyraBRServerManager.generated.h"

USTRUCT(BlueprintType)
struct FBRMatchInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString MatchID;

    UPROPERTY(BlueprintReadOnly)
    int32 CurrentPlayers;

    UPROPERTY(BlueprintReadOnly)
    int32 MaxPlayers;

    UPROPERTY(BlueprintReadOnly)
    EBattleRoyalePhase CurrentPhase;

    UPROPERTY(BlueprintReadOnly)
    float MatchStartTime;
};

/**
 * 服务器管理子系统
 * 处理匹配创建、玩家连接和服务器状态
 */
UCLASS()
class ULyraBRServerManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 创建新匹配
    UFUNCTION(BlueprintCallable, Category = "Server")
    FString CreateNewMatch(const FString& MapName, int32 MaxPlayers);

    // 获取当前匹配信息
    UFUNCTION(BlueprintPure, Category = "Server")
    FBRMatchInfo GetCurrentMatchInfo() const;

    // 向主服务器报告状态
    UFUNCTION(BlueprintCallable, Category = "Server")
    void ReportServerStatus();

protected:
    UPROPERTY()
    FBRMatchInfo CurrentMatch;

private:
    void SendHeartbeat();
    FTimerHandle HeartbeatTimer;
};
```

### 10.3 匹配系统集成

与在线子系统（Online Subsystem）集成：

```cpp
// LyraBRMatchmakingSubsystem.h
#pragma once

#include "Subsystems/GameInstanceSubsystem.h"
#include "Interfaces/OnlineSessionInterface.h"
#include "LyraBRMatchmakingSubsystem.generated.h"

/**
 * 匹配系统
 * 处理玩家匹配和房间查找
 */
UCLASS()
class ULyraBRMatchmakingSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 开始匹配
    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    void StartMatchmaking();

    // 取消匹配
    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    void CancelMatchmaking();

    // 创建私人房间
    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    bool CreatePrivateMatch(int32 MaxPlayers);

    // 加入私人房间
    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    void JoinPrivateMatch(const FString& RoomCode);

protected:
    TSharedPtr<class IOnlineSession> OnlineSessionInterface;

private:
    void OnCreateSessionComplete(FName SessionName, bool bWasSuccessful);
    void OnFindSessionsComplete(bool bWasSuccessful);
    void OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result);

    FOnCreateSessionCompleteDelegate OnCreateSessionCompleteDelegate;
    FOnFindSessionsCompleteDelegate OnFindSessionsCompleteDelegate;
    FOnJoinSessionCompleteDelegate OnJoinSessionCompleteDelegate;

    TSharedPtr<class FOnlineSessionSearch> SessionSearch;
};
```

## 十一、实战案例：从 Lyra 改造

### 11.1 项目配置清单

1. **添加 C++ 类**：
   - 复制上述所有头文件和实现文件到项目
   - 在 `Build.cs` 中添加模块依赖：
     ```csharp
     PublicDependencyModuleNames.AddRange(new string[] 
     {
         "Core", "CoreUObject", "Engine", "InputCore",
         "LyraGame", "ModularGameplay", "GameplayAbilities",
         "ReplicationGraph", "OnlineSubsystem", "OnlineSubsystemUtils"
     });
     ```

2. **创建 Data Assets**：
   - `DA_BattleRoyaleExperience`：配置游戏体验
   - `DA_BRLootConfig`：配置战利品生成
   - `DA_BRSafeZoneConfig`：配置安全区阶段

3. **创建蓝图**：
   - `BP_BRGameMode`：基于 `ALyraBattleRoyaleGameMode`
   - `BP_BRGameState`：基于 `ALyraBattleRoyaleGameState`
   - `BP_BRPlane`：基于 `ALyraBRPlaneActor`
   - `BP_BRSafeZoneManager`：基于 `ALyraBRSafeZoneManager`
   - `WBP_BRInventory`：库存 UI
   - `WBP_BRHud`：主 HUD

4. **配置地图**：
   - 创建大型开放世界地图（建议 8x8km 或更大）
   - 放置战利品生成点（使用 Tag: "LootSpawn"）
   - 放置载具生成点（使用 Tag: "VehicleSpawn"）
   - 配置流式加载子关卡

### 11.2 测试和调试

**控制台命令**：

```
; 显示网络统计
stat net

; 显示 Replication Graph 统计
replicationgraph.displaydebug

; 模拟延迟
net pktlag 100

; 模拟丢包
net pktloss 5

; 显示帧率
stat fps

; 显示安全区调试信息
showdebug safezone
```

**日志类别**：

```cpp
// 在代码中添加日志类别
DECLARE_LOG_CATEGORY_EXTERN(LogLyraBR, Log, All);
DEFINE_LOG_CATEGORY(LogLyraBR);

// 使用示例
UE_LOG(LogLyraBR, Log, TEXT("Battle Royale phase changed to: %d"), (int32)CurrentPhase);
```

## 十二、总结与最佳实践

### 12.1 核心要点

1. **网络优化是关键**：
   - 使用 Replication Graph 减少网络开销
   - 合理设置复制频率
   - 使用相关性裁剪

2. **性能至上**：
   - LOD 系统必须配置得当
   - 流式加载确保内存不溢出
   - 使用性能分析工具持续监控

3. **游戏平衡性**：
   - 战利品分布要公平但有趣
   - 安全区收缩节奏决定游戏时长
   - 武器和装备需要大量测试平衡

4. **玩家体验**：
   - UI 要清晰直观
   - 反馈要及时（拾取、击杀、伤害）
   - 观战系统让死亡玩家不无聊

### 12.2 进阶功能

- **排行榜和赛季系统**
- **好友组队**
- **语音通讯**
- **回放系统**
- **反作弊系统**
- **分区匹配（按技能分级）**

### 12.3 常见问题

**Q: 如何处理网络延迟补偿？**
A: 使用 UE5 的 Lag Compensation 系统，结合 Character Movement Component 的预测功能。

**Q: 100 玩家对服务器配置要求？**
A: 建议至少 16核 CPU、32GB RAM、高速 SSD。使用云服务可按需扩展。

**Q: 如何防止外挂？**
A: 服务器权威验证所有重要操作、使用 Easy Anti-Cheat 等第三方工具、记录异常数据分析。

**Q: 移动端能否支持大逃杀？**
A: 可以，但需要大幅降低渲染质量、减少同屏玩家数量、优化网络数据包大小。

## 总结

本章详细介绍了如何在 Lyra 框架下开发一个完整的大逃杀游戏模式，涵盖了从核心架构到网络优化的所有关键技术点。通过合理利用 Lyra 的模块化设计、Experience 系统和 GAS，我们可以快速构建出高质量的大型多人游戏。

记住，大逃杀模式的成功不仅在于技术实现，更在于游戏设计和玩家体验的打磨。持续迭代、收集反馈、优化平衡性，才能创造出真正吸引玩家的游戏。
