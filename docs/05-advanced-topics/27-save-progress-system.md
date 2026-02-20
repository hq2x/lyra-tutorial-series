# 27. 存档系统与进度管理

## 概述

在现代游戏中，可靠的存档系统是保证玩家体验的核心功能。UE5 Lyra 虽然主要聚焦于多人竞技，但其 Settings 系统为我们展示了如何设计可扩展、跨平台的数据持久化方案。本章将深入探讨如何在 Lyra 基础上构建完整的存档系统，涵盖本地存档、云存档、数据迁移和安全防护。

### 本章内容

- SaveGame 系统架构设计
- 玩家进度数据结构设计
- 存档序列化和版本管理
- 云存档集成（Steam/EOS）
- 自动存档和手动存档策略
- 跨平台存档同步
- 数据安全和防作弊
- 完整代码示例和实战案例

---

## 1. SaveGame 系统架构

### 1.1 Lyra 的 Settings 系统架构

Lyra 采用了双层设置系统：

```cpp
// 本地设置：继承自 UGameUserSettings，存储在本地配置文件
UCLASS()
class ULyraSettingsLocal : public UGameUserSettings
{
    GENERATED_BODY()
    
public:
    static ULyraSettingsLocal* Get();
    
    virtual void LoadSettings(bool bForceReload) override;
    virtual void SaveSettings() override;
    virtual void ApplyNonResolutionSettings() override;
    
private:
    // 本地配置（不同步）
    UPROPERTY(Config)
    float DisplayGamma = 2.2;
    
    UPROPERTY(Config)
    float FrameRateLimit_OnBattery;
    
    UPROPERTY(Config)
    FString AudioOutputDeviceId;
};
```

```cpp
// 共享设置：继承自 ULocalPlayerSaveGame，通过 SaveGame 系统存储
UCLASS()
class ULyraSettingsShared : public ULocalPlayerSaveGame
{
    GENERATED_BODY()
    
public:
    // 异步加载接口
    static bool AsyncLoadOrCreateSettings(
        const ULyraLocalPlayer* LocalPlayer, 
        FOnSettingsLoadedEvent Delegate
    );
    
    // 同步加载接口
    static ULyraSettingsShared* LoadOrCreateSettings(
        const ULyraLocalPlayer* LocalPlayer
    );
    
    virtual int32 GetLatestDataVersion() const override;
    
    void SaveSettings();
    void ApplySettings();
    
private:
    // 可跨设备同步的配置
    UPROPERTY()
    EColorBlindMode ColorBlindMode = EColorBlindMode::Off;
    
    UPROPERTY()
    bool bForceFeedbackEnabled = true;
    
    UPROPERTY()
    double MouseSensitivityX = 1.0;
    
    bool bIsDirty = false;
};
```

**关键设计点：**

1. **分离关注点**：本地设置（硬件相关）vs 共享设置（用户偏好）
2. **异步加载**：避免阻塞主线程
3. **版本管理**：`GetLatestDataVersion()` 支持数据迁移
4. **脏标记**：`bIsDirty` 优化不必要的保存操作

### 1.2 完整的存档系统架构

扩展 Lyra 的设计，构建三层存档架构：

```cpp
/**
 * 存档基类 - 提供版本管理和序列化基础
 */
UCLASS(Abstract)
class ULyraSaveGameBase : public USaveGame
{
    GENERATED_BODY()
    
public:
    ULyraSaveGameBase();
    
    // 版本控制
    UPROPERTY(SaveGame)
    int32 SaveVersion;
    
    UPROPERTY(SaveGame)
    FDateTime SaveTimestamp;
    
    UPROPERTY(SaveGame)
    FString SavePlatform;
    
    // 虚拟接口
    virtual int32 GetLatestVersion() const { return 1; }
    virtual void MigrateFromVersion(int32 OldVersion) {}
    virtual FString GetSaveSlotName() const { return TEXT("DefaultSlot"); }
    
    // 数据校验
    UPROPERTY(SaveGame)
    FString DataChecksum;
    
    bool ValidateChecksum() const;
    void UpdateChecksum();
};
```

```cpp
/**
 * 玩家进度存档 - 存储游戏进度和统计数据
 */
UCLASS()
class ULyraProgressSaveGame : public ULyraSaveGameBase
{
    GENERATED_BODY()
    
public:
    virtual int32 GetLatestVersion() const override { return 3; }
    virtual FString GetSaveSlotName() const override;
    
    // 玩家基础信息
    UPROPERTY(SaveGame)
    FString PlayerName;
    
    UPROPERTY(SaveGame)
    int32 PlayerLevel;
    
    UPROPERTY(SaveGame)
    int32 PlayerExperience;
    
    // 任务进度
    UPROPERTY(SaveGame)
    TMap<FName, FQuestProgress> QuestData;
    
    // 物品和装备
    UPROPERTY(SaveGame)
    TArray<FInventoryItemData> InventoryItems;
    
    UPROPERTY(SaveGame)
    TMap<EEquipmentSlot, FEquipmentData> EquippedItems;
    
    // 解锁内容
    UPROPERTY(SaveGame)
    TSet<FName> UnlockedAchievements;
    
    UPROPERTY(SaveGame)
    TSet<FName> UnlockedAbilities;
    
    // 游戏统计
    UPROPERTY(SaveGame)
    FGameStatistics Statistics;
};
```

```cpp
/**
 * 关卡存档 - 存储特定关卡的状态
 */
UCLASS()
class ULyraLevelSaveGame : public ULyraSaveGameBase
{
    GENERATED_BODY()
    
public:
    UPROPERTY(SaveGame)
    FName LevelName;
    
    // Actor 状态（位置、状态、属性）
    UPROPERTY(SaveGame)
    TArray<FActorSaveData> SavedActors;
    
    // 触发器状态
    UPROPERTY(SaveGame)
    TMap<FName, bool> TriggerStates;
    
    // 收集品状态
    UPROPERTY(SaveGame)
    TSet<FName> CollectedItems;
    
    // 对话历史
    UPROPERTY(SaveGame)
    TMap<FName, int32> DialogueProgress;
};
```

---

## 2. 玩家进度数据结构设计

### 2.1 核心数据结构

```cpp
/**
 * 任务进度数据
 */
USTRUCT(BlueprintType)
struct FQuestProgress
{
    GENERATED_BODY()
    
    UPROPERTY(SaveGame)
    FName QuestID;
    
    UPROPERTY(SaveGame)
    EQuestState State; // NotStarted, InProgress, Completed, Failed
    
    UPROPERTY(SaveGame)
    TMap<FName, int32> ObjectiveProgress;
    
    UPROPERTY(SaveGame)
    FDateTime StartedTime;
    
    UPROPERTY(SaveGame)
    FDateTime CompletedTime;
    
    UPROPERTY(SaveGame)
    TArray<FName> CompletedObjectives;
};

/**
 * 物品数据
 */
USTRUCT(BlueprintType)
struct FInventoryItemData
{
    GENERATED_BODY()
    
    UPROPERTY(SaveGame)
    FName ItemID;
    
    UPROPERTY(SaveGame)
    int32 Quantity;
    
    UPROPERTY(SaveGame)
    int32 Durability; // 0-100
    
    UPROPERTY(SaveGame)
    TMap<FName, float> ItemStats; // 武器伤害、防御值等
    
    UPROPERTY(SaveGame)
    TArray<FName> AppliedEnchantments;
    
    UPROPERTY(SaveGame)
    FGuid UniqueID; // 用于唯一标识
};

/**
 * 装备数据
 */
USTRUCT(BlueprintType)
struct FEquipmentData
{
    GENERATED_BODY()
    
    UPROPERTY(SaveGame)
    FInventoryItemData ItemData;
    
    UPROPERTY(SaveGame)
    EEquipmentSlot Slot;
    
    UPROPERTY(SaveGame)
    bool bIsLocked; // 防止意外卸下
};

/**
 * 游戏统计
 */
USTRUCT(BlueprintType)
struct FGameStatistics
{
    GENERATED_BODY()
    
    UPROPERTY(SaveGame)
    float TotalPlayTime; // 秒
    
    UPROPERTY(SaveGame)
    int32 TotalKills;
    
    UPROPERTY(SaveGame)
    int32 TotalDeaths;
    
    UPROPERTY(SaveGame)
    int32 TotalGoldEarned;
    
    UPROPERTY(SaveGame)
    TMap<FName, int32> EnemyKillCounts;
    
    UPROPERTY(SaveGame)
    TMap<FName, float> SkillUsageTimes;
};
```

### 2.2 Actor 状态保存

```cpp
/**
 * Actor 保存数据
 */
USTRUCT()
struct FActorSaveData
{
    GENERATED_BODY()
    
    UPROPERTY(SaveGame)
    FName ActorName;
    
    UPROPERTY(SaveGame)
    FTransform ActorTransform;
    
    UPROPERTY(SaveGame)
    TSubclassOf<AActor> ActorClass;
    
    UPROPERTY(SaveGame)
    TArray<uint8> ActorData; // 序列化的属性数据
    
    UPROPERTY(SaveGame)
    bool bIsDestroyed;
};

/**
 * 可保存 Actor 接口
 */
UINTERFACE(BlueprintType)
class ULyraSavableInterface : public UInterface
{
    GENERATED_BODY()
};

class ILyraSavableInterface
{
    GENERATED_BODY()
    
public:
    // 序列化到二进制数据
    UFUNCTION(BlueprintNativeEvent, Category = "SaveGame")
    TArray<uint8> SerializeActorData();
    
    // 从二进制数据反序列化
    UFUNCTION(BlueprintNativeEvent, Category = "SaveGame")
    void DeserializeActorData(const TArray<uint8>& Data);
    
    // 返回唯一标识符
    UFUNCTION(BlueprintNativeEvent, Category = "SaveGame")
    FName GetSavableActorID() const;
};
```

**实现示例：**

```cpp
TArray<uint8> ALyraEnemyCharacter::SerializeActorData_Implementation()
{
    FMemoryWriter MemoryWriter(TArray<uint8>(), true);
    
    // 保存血量
    MemoryWriter << CurrentHealth;
    MemoryWriter << MaxHealth;
    
    // 保存AI状态
    int32 AIState = static_cast<int32>(CurrentAIState);
    MemoryWriter << AIState;
    
    // 保存巡逻点
    MemoryWriter << PatrolPointIndex;
    
    return MemoryWriter.GetData();
}

void ALyraEnemyCharacter::DeserializeActorData_Implementation(const TArray<uint8>& Data)
{
    FMemoryReader MemoryReader(Data, true);
    
    MemoryReader << CurrentHealth;
    MemoryReader << MaxHealth;
    
    int32 AIState;
    MemoryReader << AIState;
    CurrentAIState = static_cast<EAIState>(AIState);
    
    MemoryReader << PatrolPointIndex;
}
```

---

## 3. 存档序列化和版本管理

### 3.1 版本迁移系统

```cpp
void ULyraProgressSaveGame::MigrateFromVersion(int32 OldVersion)
{
    // 从版本 1 迁移到版本 2：添加装备系统
    if (OldVersion < 2)
    {
        UE_LOG(LogLyraSave, Log, TEXT("Migrating save from version 1 to 2"));
        
        // 将旧的武器数据转换为装备数据
        for (const FInventoryItemData& Item : InventoryItems)
        {
            if (Item.ItemID.ToString().StartsWith("Weapon_"))
            {
                FEquipmentData EquipData;
                EquipData.ItemData = Item;
                EquipData.Slot = EEquipmentSlot::MainHand;
                EquippedItems.Add(EEquipmentSlot::MainHand, EquipData);
                break; // 只装备第一个武器
            }
        }
    }
    
    // 从版本 2 迁移到版本 3：添加成就系统
    if (OldVersion < 3)
    {
        UE_LOG(LogLyraSave, Log, TEXT("Migrating save from version 2 to 3"));
        
        // 根据游戏进度解锁相应成就
        if (PlayerLevel >= 10)
        {
            UnlockedAchievements.Add(FName("Achievement_Level10"));
        }
        
        // 统计完成的任务数量
        int32 CompletedQuests = 0;
        for (const auto& Quest : QuestData)
        {
            if (Quest.Value.State == EQuestState::Completed)
            {
                CompletedQuests++;
            }
        }
        
        if (CompletedQuests >= 5)
        {
            UnlockedAchievements.Add(FName("Achievement_5Quests"));
        }
    }
}
```

### 3.2 自定义序列化

```cpp
/**
 * 高级序列化 - 使用自定义格式
 */
struct FLyraSaveDataSerializer
{
    // 压缩保存
    static TArray<uint8> CompressData(const TArray<uint8>& RawData)
    {
        TArray<uint8> CompressedData;
        FArchiveSaveCompressedProxy Compressor(
            CompressedData,
            NAME_Zlib,
            COMPRESS_BiasSpeed
        );
        
        Compressor << const_cast<TArray<uint8>&>(RawData);
        Compressor.Close();
        
        return CompressedData;
    }
    
    // 解压数据
    static TArray<uint8> DecompressData(const TArray<uint8>& CompressedData)
    {
        TArray<uint8> RawData;
        FArchiveLoadCompressedProxy Decompressor(
            CompressedData,
            NAME_Zlib
        );
        
        Decompressor << RawData;
        Decompressor.Close();
        
        return RawData;
    }
    
    // 加密（简单 XOR，生产环境需要更强算法）
    static void EncryptData(TArray<uint8>& Data, const FString& Key)
    {
        uint32 KeyHash = GetTypeHash(Key);
        for (int32 i = 0; i < Data.Num(); ++i)
        {
            Data[i] ^= (KeyHash >> (i % 4) * 8) & 0xFF;
        }
    }
};
```

### 3.3 校验和生成

```cpp
void ULyraSaveGameBase::UpdateChecksum()
{
    // 序列化当前数据
    TArray<uint8> RawData;
    FMemoryWriter MemoryWriter(RawData, true);
    FObjectAndNameAsStringProxyArchive Archive(MemoryWriter, false);
    
    // 排除校验和字段本身
    FString TempChecksum = DataChecksum;
    DataChecksum.Empty();
    
    this->Serialize(Archive);
    
    // 计算 SHA256
    uint8 Hash[32];
    FSHA256::HashBuffer(RawData.GetData(), RawData.Num(), Hash);
    
    // 转换为十六进制字符串
    DataChecksum = BytesToHex(Hash, 32);
}

bool ULyraSaveGameBase::ValidateChecksum() const
{
    if (DataChecksum.IsEmpty())
    {
        return false; // 旧版本存档，跳过校验
    }
    
    ULyraSaveGameBase* TempCopy = DuplicateObject<ULyraSaveGameBase>(
        const_cast<ULyraSaveGameBase*>(this),
        nullptr
    );
    
    FString OriginalChecksum = TempCopy->DataChecksum;
    TempCopy->UpdateChecksum();
    
    return OriginalChecksum.Equals(TempCopy->DataChecksum);
}
```

---

## 4. 云存档集成

### 4.1 抽象云存档接口

```cpp
/**
 * 云存档提供者接口
 */
UCLASS(Abstract)
class ULyraCloudSaveProvider : public UObject
{
    GENERATED_BODY()
    
public:
    // 上传存档
    virtual void UploadSave(
        const FString& SaveName,
        const TArray<uint8>& SaveData,
        FOnCloudSaveCompleted OnCompleted
    ) PURE_VIRTUAL(ULyraCloudSaveProvider::UploadSave, );
    
    // 下载存档
    virtual void DownloadSave(
        const FString& SaveName,
        FOnCloudSaveLoaded OnLoaded
    ) PURE_VIRTUAL(ULyraCloudSaveProvider::DownloadSave, );
    
    // 列出所有云存档
    virtual void ListCloudSaves(
        FOnCloudSavesListed OnListed
    ) PURE_VIRTUAL(ULyraCloudSaveProvider::ListCloudSaves, );
    
    // 删除云存档
    virtual void DeleteCloudSave(
        const FString& SaveName,
        FOnCloudSaveDeleted OnDeleted
    ) PURE_VIRTUAL(ULyraCloudSaveProvider::DeleteCloudSave, );
};
```

### 4.2 Steam 云存档实现

```cpp
/**
 * Steam 云存档实现
 */
UCLASS()
class ULyraSteamCloudSaveProvider : public ULyraCloudSaveProvider
{
    GENERATED_BODY()
    
public:
    virtual void UploadSave(
        const FString& SaveName,
        const TArray<uint8>& SaveData,
        FOnCloudSaveCompleted OnCompleted
    ) override
    {
        if (!IsSteamInitialized())
        {
            OnCompleted.ExecuteIfBound(false, TEXT("Steam not initialized"));
            return;
        }
        
        // 写入 Steam Cloud 文件
        ISteamRemoteStorage* RemoteStorage = SteamRemoteStorage();
        if (!RemoteStorage)
        {
            OnCompleted.ExecuteIfBound(false, TEXT("Steam Remote Storage unavailable"));
            return;
        }
        
        FString CloudFileName = FString::Printf(TEXT("SaveGame_%s.sav"), *SaveName);
        bool bSuccess = RemoteStorage->FileWrite(
            TCHAR_TO_UTF8(*CloudFileName),
            SaveData.GetData(),
            SaveData.Num()
        );
        
        if (bSuccess)
        {
            UE_LOG(LogLyraSave, Log, TEXT("Uploaded save to Steam Cloud: %s (%d bytes)"),
                *CloudFileName, SaveData.Num());
        }
        
        OnCompleted.ExecuteIfBound(bSuccess, bSuccess ? TEXT("") : TEXT("FileWrite failed"));
    }
    
    virtual void DownloadSave(
        const FString& SaveName,
        FOnCloudSaveLoaded OnLoaded
    ) override
    {
        ISteamRemoteStorage* RemoteStorage = SteamRemoteStorage();
        if (!RemoteStorage)
        {
            OnLoaded.ExecuteIfBound(TArray<uint8>(), false, TEXT("Steam unavailable"));
            return;
        }
        
        FString CloudFileName = FString::Printf(TEXT("SaveGame_%s.sav"), *SaveName);
        
        // 检查文件是否存在
        if (!RemoteStorage->FileExists(TCHAR_TO_UTF8(*CloudFileName)))
        {
            OnLoaded.ExecuteIfBound(TArray<uint8>(), false, TEXT("File not found"));
            return;
        }
        
        // 获取文件大小
        int32 FileSize = RemoteStorage->GetFileSize(TCHAR_TO_UTF8(*CloudFileName));
        
        // 读取文件
        TArray<uint8> SaveData;
        SaveData.SetNum(FileSize);
        
        int32 BytesRead = RemoteStorage->FileRead(
            TCHAR_TO_UTF8(*CloudFileName),
            SaveData.GetData(),
            FileSize
        );
        
        bool bSuccess = (BytesRead == FileSize);
        OnLoaded.ExecuteIfBound(SaveData, bSuccess, bSuccess ? TEXT("") : TEXT("Read error"));
    }
    
private:
    bool IsSteamInitialized() const
    {
        return SteamAPI_IsSteamRunning();
    }
};
```

### 4.3 EOS 云存档实现

```cpp
/**
 * Epic Online Services 云存档实现
 */
UCLASS()
class ULyraEOSCloudSaveProvider : public ULyraCloudSaveProvider
{
    GENERATED_BODY()
    
public:
    virtual void UploadSave(
        const FString& SaveName,
        const TArray<uint8>& SaveData,
        FOnCloudSaveCompleted OnCompleted
    ) override
    {
        IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get(EOS_SUBSYSTEM);
        if (!OnlineSubsystem)
        {
            OnCompleted.ExecuteIfBound(false, TEXT("EOS not available"));
            return;
        }
        
        IOnlineUserCloudPtr UserCloud = OnlineSubsystem->GetUserCloudInterface();
        if (!UserCloud.IsValid())
        {
            OnCompleted.ExecuteIfBound(false, TEXT("User Cloud interface unavailable"));
            return;
        }
        
        // 获取本地用户
        ULocalPlayer* LocalPlayer = GetLocalPlayer();
        if (!LocalPlayer)
        {
            OnCompleted.ExecuteIfBound(false, TEXT("No local player"));
            return;
        }
        
        FUniqueNetIdRepl UserId = LocalPlayer->GetPreferredUniqueNetId();
        
        // 写入云存档
        UserCloud->WriteUserFile(
            *UserId,
            SaveName,
            SaveData
        );
        
        // 监听完成事件
        UserCloud->AddOnWriteUserFileCompleteDelegate_Handle(
            FOnWriteUserFileCompleteDelegate::CreateLambda(
                [OnCompleted](bool bWasSuccessful, const FUniqueNetId& InUserId, const FString& FileName)
                {
                    OnCompleted.ExecuteIfBound(
                        bWasSuccessful,
                        bWasSuccessful ? TEXT("") : TEXT("EOS write failed")
                    );
                }
            )
        );
    }
    
    virtual void DownloadSave(
        const FString& SaveName,
        FOnCloudSaveLoaded OnLoaded
    ) override
    {
        IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get(EOS_SUBSYSTEM);
        IOnlineUserCloudPtr UserCloud = OnlineSubsystem->GetUserCloudInterface();
        
        ULocalPlayer* LocalPlayer = GetLocalPlayer();
        FUniqueNetIdRepl UserId = LocalPlayer->GetPreferredUniqueNetId();
        
        // 读取云存档
        UserCloud->ReadUserFile(*UserId, SaveName);
        
        // 监听完成事件
        UserCloud->AddOnReadUserFileCompleteDelegate_Handle(
            FOnReadUserFileCompleteDelegate::CreateLambda(
                [OnLoaded, SaveName, UserCloud](bool bWasSuccessful, const FUniqueNetId& InUserId, const FString& FileName)
                {
                    if (bWasSuccessful)
                    {
                        TArray<uint8> FileContents;
                        UserCloud->GetFileContents(InUserId, FileName, FileContents);
                        OnLoaded.ExecuteIfBound(FileContents, true, TEXT(""));
                    }
                    else
                    {
                        OnLoaded.ExecuteIfBound(TArray<uint8>(), false, TEXT("EOS read failed"));
                    }
                }
            )
        );
    }
};
```

---

## 5. 自动存档和手动存档策略

### 5.1 存档管理器

```cpp
/**
 * 存档管理器 - 单例模式
 */
UCLASS()
class ULyraSaveGameManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    // 初始化
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
    
    // 手动保存
    UFUNCTION(BlueprintCallable, Category = "SaveGame")
    void SaveGame(const FString& SlotName = TEXT(""), bool bAsyncSave = true);
    
    // 加载游戏
    UFUNCTION(BlueprintCallable, Category = "SaveGame")
    bool LoadGame(const FString& SlotName = TEXT(""));
    
    // 自动保存配置
    UFUNCTION(BlueprintCallable, Category = "SaveGame")
    void SetAutoSaveEnabled(bool bEnabled);
    
    UFUNCTION(BlueprintCallable, Category = "SaveGame")
    void SetAutoSaveInterval(float Seconds);
    
    // 获取当前存档
    UFUNCTION(BlueprintPure, Category = "SaveGame")
    ULyraProgressSaveGame* GetCurrentSave() const { return CurrentSave; }
    
    // 云存档同步
    UFUNCTION(BlueprintCallable, Category = "SaveGame")
    void SyncWithCloud();
    
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSaveCompleted, bool, bSuccess);
    UPROPERTY(BlueprintAssignable, Category = "SaveGame")
    FOnSaveCompleted OnSaveCompleted;
    
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLoadCompleted, bool, bSuccess);
    UPROPERTY(BlueprintAssignable, Category = "SaveGame")
    FOnLoadCompleted OnLoadCompleted;
    
private:
    UPROPERTY()
    ULyraProgressSaveGame* CurrentSave;
    
    UPROPERTY()
    ULyraCloudSaveProvider* CloudProvider;
    
    FTimerHandle AutoSaveTimerHandle;
    bool bAutoSaveEnabled = true;
    float AutoSaveInterval = 300.0f; // 5分钟
    
    void TriggerAutoSave();
    void OnAutoSaveCompleted(const FString& SlotName, const int32 UserIndex, bool bSuccess);
};
```

### 5.2 自动保存实现

```cpp
void ULyraSaveGameManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    
    // 加载上次存档
    LoadGame();
    
    // 启动自动保存定时器
    if (bAutoSaveEnabled)
    {
        GetWorld()->GetTimerManager().SetTimer(
            AutoSaveTimerHandle,
            this,
            &ULyraSaveGameManager::TriggerAutoSave,
            AutoSaveInterval,
            true
        );
    }
    
    // 监听关键游戏事件
    if (ULyraGameInstance* GameInstance = Cast<ULyraGameInstance>(GetGameInstance()))
    {
        GameInstance->OnQuestCompleted.AddUObject(this, &ULyraSaveGameManager::OnQuestCompleted);
        GameInstance->OnLevelChanged.AddUObject(this, &ULyraSaveGameManager::OnLevelChanged);
        GameInstance->OnCheckpointReached.AddUObject(this, &ULyraSaveGameManager::OnCheckpointReached);
    }
}

void ULyraSaveGameManager::TriggerAutoSave()
{
    // 检查是否在合适的时机保存
    APlayerController* PC = GetWorld()->GetFirstPlayerController();
    if (!PC)
    {
        return;
    }
    
    APawn* PlayerPawn = PC->GetPawn();
    if (!PlayerPawn || PlayerPawn->GetVelocity().Size() > 100.0f)
    {
        // 玩家在移动中，延迟保存
        return;
    }
    
    // 检查是否在战斗中
    if (ULyraHealthComponent* HealthComp = PlayerPawn->FindComponentByClass<ULyraHealthComponent>())
    {
        if (HealthComp->IsInCombat())
        {
            return; // 战斗中不自动保存
        }
    }
    
    UE_LOG(LogLyraSave, Log, TEXT("Triggering auto-save"));
    SaveGame(TEXT("AutoSave"), true);
}

void ULyraSaveGameManager::SaveGame(const FString& SlotName, bool bAsyncSave)
{
    if (!CurrentSave)
    {
        CurrentSave = Cast<ULyraProgressSaveGame>(
            UGameplayStatics::CreateSaveGameObject(ULyraProgressSaveGame::StaticClass())
        );
    }
    
    // 更新时间戳
    CurrentSave->SaveTimestamp = FDateTime::Now();
    CurrentSave->SaveVersion = CurrentSave->GetLatestVersion();
    CurrentSave->SavePlatform = UGameplayStatics::GetPlatformName();
    
    // 收集玩家数据
    CapturePlayerData();
    
    // 收集世界状态
    CaptureWorldState();
    
    // 更新校验和
    CurrentSave->UpdateChecksum();
    
    // 确定存档槽位
    FString ActualSlotName = SlotName.IsEmpty() ? CurrentSave->GetSaveSlotName() : SlotName;
    
    // 异步或同步保存
    if (bAsyncSave)
    {
        UGameplayStatics::AsyncSaveGameToSlot(
            CurrentSave,
            ActualSlotName,
            0,
            FAsyncSaveGameToSlotDelegate::CreateUObject(
                this,
                &ULyraSaveGameManager::OnAutoSaveCompleted
            )
        );
    }
    else
    {
        bool bSuccess = UGameplayStatics::SaveGameToSlot(CurrentSave, ActualSlotName, 0);
        OnSaveCompleted.Broadcast(bSuccess);
    }
    
    // 同时上传到云端
    if (CloudProvider)
    {
        // 序列化存档
        TArray<uint8> SaveData;
        FMemoryWriter MemoryWriter(SaveData, true);
        FObjectAndNameAsStringProxyArchive Archive(MemoryWriter, false);
        CurrentSave->Serialize(Archive);
        
        // 压缩和加密
        SaveData = FLyraSaveDataSerializer::CompressData(SaveData);
        FLyraSaveDataSerializer::EncryptData(SaveData, TEXT("MySecretKey"));
        
        CloudProvider->UploadSave(
            ActualSlotName,
            SaveData,
            FOnCloudSaveCompleted::CreateLambda([](bool bSuccess, const FString& Error)
            {
                UE_LOG(LogLyraSave, Log, TEXT("Cloud save: %s"), bSuccess ? TEXT("Success") : *Error);
            })
        );
    }
}
```

### 5.3 关键时刻自动保存

```cpp
void ULyraSaveGameManager::OnCheckpointReached(const FName& CheckpointName)
{
    UE_LOG(LogLyraSave, Log, TEXT("Checkpoint reached: %s - Triggering save"), *CheckpointName.ToString());
    
    // 记录检查点
    if (CurrentSave)
    {
        CurrentSave->LastCheckpoint = CheckpointName;
    }
    
    SaveGame(TEXT("Checkpoint"), true);
}

void ULyraSaveGameManager::OnQuestCompleted(const FName& QuestID)
{
    UE_LOG(LogLyraSave, Log, TEXT("Quest completed: %s - Triggering save"), *QuestID.ToString());
    SaveGame(TEXT("AutoSave"), true);
}

void ULyraSaveGameManager::OnLevelChanged(const FName& NewLevel)
{
    UE_LOG(LogLyraSave, Log, TEXT("Level changed to: %s - Triggering save"), *NewLevel.ToString());
    SaveGame(TEXT("LevelTransition"), true);
}
```

---

## 6. 跨平台存档同步

### 6.1 冲突解决策略

```cpp
/**
 * 存档冲突解决器
 */
struct FLyraSaveConflictResolver
{
    enum class EResolveStrategy
    {
        UseLocal,          // 使用本地版本
        UseCloud,          // 使用云端版本
        UseMostRecent,     // 使用最新版本
        Merge              // 合并两者（保留进度最高的）
    };
    
    static ULyraProgressSaveGame* ResolveSaveConflict(
        ULyraProgressSaveGame* LocalSave,
        ULyraProgressSaveGame* CloudSave,
        EResolveStrategy Strategy = EResolveStrategy::UseMostRecent
    )
    {
        if (!LocalSave && !CloudSave)
        {
            return nullptr;
        }
        
        if (!LocalSave)
        {
            return CloudSave;
        }
        
        if (!CloudSave)
        {
            return LocalSave;
        }
        
        switch (Strategy)
        {
        case EResolveStrategy::UseLocal:
            return LocalSave;
            
        case EResolveStrategy::UseCloud:
            return CloudSave;
            
        case EResolveStrategy::UseMostRecent:
            return (LocalSave->SaveTimestamp > CloudSave->SaveTimestamp) ? LocalSave : CloudSave;
            
        case EResolveStrategy::Merge:
            return MergeSaveGames(LocalSave, CloudSave);
        }
        
        return LocalSave;
    }
    
private:
    static ULyraProgressSaveGame* MergeSaveGames(
        ULyraProgressSaveGame* Save1,
        ULyraProgressSaveGame* Save2
    )
    {
        // 使用等级更高的存档作为基础
        ULyraProgressSaveGame* BaseSave = (Save1->PlayerLevel >= Save2->PlayerLevel) ? Save1 : Save2;
        ULyraProgressSaveGame* OtherSave = (BaseSave == Save1) ? Save2 : Save1;
        
        // 合并任务进度（保留两者中进度更高的）
        for (const auto& Quest : OtherSave->QuestData)
        {
            if (!BaseSave->QuestData.Contains(Quest.Key))
            {
                BaseSave->QuestData.Add(Quest.Key, Quest.Value);
            }
            else
            {
                // 如果另一个存档中的任务更进一步，则使用那个
                if (Quest.Value.State == EQuestState::Completed &&
                    BaseSave->QuestData[Quest.Key].State != EQuestState::Completed)
                {
                    BaseSave->QuestData[Quest.Key] = Quest.Value;
                }
            }
        }
        
        // 合并解锁内容
        BaseSave->UnlockedAchievements.Append(OtherSave->UnlockedAchievements);
        BaseSave->UnlockedAbilities.Append(OtherSave->UnlockedAbilities);
        
        // 合并统计数据（取最大值）
        BaseSave->Statistics.TotalPlayTime = FMath::Max(
            BaseSave->Statistics.TotalPlayTime,
            OtherSave->Statistics.TotalPlayTime
        );
        BaseSave->Statistics.TotalKills += OtherSave->Statistics.TotalKills;
        BaseSave->Statistics.TotalGoldEarned = FMath::Max(
            BaseSave->Statistics.TotalGoldEarned,
            OtherSave->Statistics.TotalGoldEarned
        );
        
        return BaseSave;
    }
};
```

### 6.2 同步流程

```cpp
void ULyraSaveGameManager::SyncWithCloud()
{
    if (!CloudProvider)
    {
        UE_LOG(LogLyraSave, Warning, TEXT("No cloud provider configured"));
        return;
    }
    
    UE_LOG(LogLyraSave, Log, TEXT("Starting cloud sync..."));
    
    // 1. 下载云存档
    CloudProvider->DownloadSave(
        TEXT("MainSave"),
        FOnCloudSaveLoaded::CreateLambda([this](const TArray<uint8>& CloudData, bool bSuccess, const FString& Error)
        {
            if (!bSuccess)
            {
                UE_LOG(LogLyraSave, Warning, TEXT("Failed to download cloud save: %s"), *Error);
                return;
            }
            
            // 2. 反序列化云存档
            TArray<uint8> DecryptedData = CloudData;
            FLyraSaveDataSerializer::EncryptData(DecryptedData, TEXT("MySecretKey")); // XOR 解密
            TArray<uint8> DecompressedData = FLyraSaveDataSerializer::DecompressData(DecryptedData);
            
            ULyraProgressSaveGame* CloudSave = Cast<ULyraProgressSaveGame>(
                UGameplayStatics::CreateSaveGameObject(ULyraProgressSaveGame::StaticClass())
            );
            
            FMemoryReader MemoryReader(DecompressedData, true);
            FObjectAndNameAsStringProxyArchive Archive(MemoryReader, false);
            CloudSave->Serialize(Archive);
            
            // 3. 校验云存档
            if (!CloudSave->ValidateChecksum())
            {
                UE_LOG(LogLyraSave, Error, TEXT("Cloud save checksum validation failed!"));
                return;
            }
            
            // 4. 解决冲突
            ULyraProgressSaveGame* ResolvedSave = FLyraSaveConflictResolver::ResolveSaveConflict(
                CurrentSave,
                CloudSave,
                FLyraSaveConflictResolver::EResolveStrategy::Merge
            );
            
            // 5. 应用同步后的存档
            CurrentSave = ResolvedSave;
            ApplySaveGameData();
            
            // 6. 上传最新版本到云端
            SaveGame(TEXT("MainSave"), true);
            
            UE_LOG(LogLyraSave, Log, TEXT("Cloud sync completed successfully"));
        })
    );
}
```

---

## 7. 数据安全和防作弊

### 7.1 数据加密

```cpp
/**
 * AES 加密实现
 */
struct FLyraAESEncryption
{
    // 生成密钥（在真实项目中应从安全源获取）
    static TArray<uint8> GenerateKey()
    {
        // 示例：从游戏配置、服务器或硬件ID派生
        FString KeySource = FPlatformMisc::GetMachineId().ToString() + TEXT("LyraSecretSalt");
        return FSHA256::HashBuffer(TCHAR_TO_UTF8(*KeySource), KeySource.Len() * sizeof(TCHAR));
    }
    
    static TArray<uint8> EncryptData(const TArray<uint8>& PlainData)
    {
        // 使用 UE 的加密模块
        TArray<uint8> Key = GenerateKey();
        TArray<uint8> EncryptedData;
        
        FAES::EncryptData(PlainData, Key, EncryptedData);
        
        return EncryptedData;
    }
    
    static TArray<uint8> DecryptData(const TArray<uint8>& EncryptedData)
    {
        TArray<uint8> Key = GenerateKey();
        TArray<uint8> DecryptedData;
        
        FAES::DecryptData(EncryptedData, Key, DecryptedData);
        
        return DecryptedData;
    }
};
```

### 7.2 防篡改机制

```cpp
/**
 * 存档完整性验证
 */
class FLyraSaveIntegrityChecker
{
public:
    // 生成签名
    static FString SignSaveData(const ULyraSaveGameBase* SaveGame)
    {
        // 序列化存档
        TArray<uint8> RawData;
        FMemoryWriter MemoryWriter(RawData, true);
        FObjectAndNameAsStringProxyArchive Archive(MemoryWriter, false);
        const_cast<ULyraSaveGameBase*>(SaveGame)->Serialize(Archive);
        
        // 添加盐值和时间戳
        FString Salt = TEXT("LyraSecretSalt_2024");
        FString Timestamp = SaveGame->SaveTimestamp.ToString();
        FString SignatureSource = BytesToString(RawData) + Salt + Timestamp;
        
        // 计算签名
        uint8 Hash[32];
        FSHA256::HashBuffer(TCHAR_TO_UTF8(*SignatureSource), SignatureSource.Len() * sizeof(TCHAR), Hash);
        
        return BytesToHex(Hash, 32);
    }
    
    // 验证签名
    static bool VerifySignature(const ULyraSaveGameBase* SaveGame, const FString& Signature)
    {
        FString CalculatedSignature = SignSaveData(SaveGame);
        return CalculatedSignature.Equals(Signature, ESearchCase::CaseSensitive);
    }
    
    // 检查异常数值
    static bool CheckForAnomalies(const ULyraProgressSaveGame* SaveGame)
    {
        // 检查等级与经验值的合理性
        int32 ExpectedExp = CalculateRequiredExp(SaveGame->PlayerLevel);
        if (SaveGame->PlayerExperience < ExpectedExp * 0.5f)
        {
            UE_LOG(LogLyraSave, Warning, TEXT("Anomaly detected: Level %d but only %d exp"),
                SaveGame->PlayerLevel, SaveGame->PlayerExperience);
            return false;
        }
        
        // 检查物品数量是否合理
        for (const FInventoryItemData& Item : SaveGame->InventoryItems)
        {
            if (Item.Quantity > 999)
            {
                UE_LOG(LogLyraSave, Warning, TEXT("Anomaly detected: Item %s quantity %d exceeds limit"),
                    *Item.ItemID.ToString(), Item.Quantity);
                return false;
            }
        }
        
        // 检查统计数据
        if (SaveGame->Statistics.TotalPlayTime < 0.0f ||
            SaveGame->Statistics.TotalKills < 0)
        {
            UE_LOG(LogLyraSave, Warning, TEXT("Anomaly detected: Negative stats"));
            return false;
        }
        
        return true;
    }
    
private:
    static int32 CalculateRequiredExp(int32 Level)
    {
        // 示例：指数增长经验需求
        return FMath::Pow(Level, 2) * 100;
    }
};
```

### 7.3 服务器端验证

```cpp
/**
 * 服务器端存档验证（用于联机游戏）
 */
UCLASS()
class ALyraSaveValidationServer : public AInfo
{
    GENERATED_BODY()
    
public:
    // 验证客户端上传的存档
    UFUNCTION(Server, Reliable)
    void Server_ValidateSaveData(const FString& SaveDataJson, const FString& ClientSignature);
    
    // 返回验证结果
    UFUNCTION(Client, Reliable)
    void Client_SaveValidationResult(bool bIsValid, const FString& Reason);
    
private:
    // 服务器端白名单检查
    bool IsPlayerWhitelisted(const FUniqueNetIdRepl& PlayerId);
    
    // 检查保存频率（防止刷分）
    bool CheckSaveFrequency(const FUniqueNetIdRepl& PlayerId);
    
    TMap<FUniqueNetIdRepl, FDateTime> LastSaveTimes;
};

void ALyraSaveValidationServer::Server_ValidateSaveData_Implementation(
    const FString& SaveDataJson,
    const FString& ClientSignature)
{
    // 1. 反序列化存档
    TSharedPtr<FJsonObject> JsonObject;
    TSharedRef<TJsonReader<>> Reader = TJsonReaderFactory<>::Create(SaveDataJson);
    if (!FJsonSerializer::Deserialize(Reader, JsonObject))
    {
        Client_SaveValidationResult(false, TEXT("Invalid JSON"));
        return;
    }
    
    // 2. 重新计算签名
    FString ServerSignature = FLyraSaveIntegrityChecker::SignSaveData(/* ... */);
    if (!ServerSignature.Equals(ClientSignature))
    {
        UE_LOG(LogLyraSave, Warning, TEXT("Save signature mismatch for player %s"),
            *GetOwner()->GetName());
        Client_SaveValidationResult(false, TEXT("Signature mismatch"));
        return;
    }
    
    // 3. 检查数值异常
    // ... (解析 JsonObject 并验证)
    
    // 4. 检查保存频率
    FUniqueNetIdRepl PlayerId = Cast<APlayerController>(GetOwner())->PlayerState->GetUniqueId();
    if (!CheckSaveFrequency(PlayerId))
    {
        Client_SaveValidationResult(false, TEXT("Save too frequent"));
        return;
    }
    
    // 5. 通过验证
    LastSaveTimes.Add(PlayerId, FDateTime::Now());
    Client_SaveValidationResult(true, TEXT(""));
}

bool ALyraSaveValidationServer::CheckSaveFrequency(const FUniqueNetIdRepl& PlayerId)
{
    if (FDateTime* LastSave = LastSaveTimes.Find(PlayerId))
    {
        FTimespan TimeSinceLastSave = FDateTime::Now() - *LastSave;
        return TimeSinceLastSave.GetTotalSeconds() > 30.0f; // 最少间隔 30 秒
    }
    return true;
}
```

---

## 8. 实战案例 1：RPG 存档系统

### 8.1 完整的 RPG 存档实现

```cpp
/**
 * RPG 游戏存档
 */
UCLASS()
class ULyraRPGSaveGame : public ULyraProgressSaveGame
{
    GENERATED_BODY()
    
public:
    virtual int32 GetLatestVersion() const override { return 4; }
    
    // === 角色信息 ===
    UPROPERTY(SaveGame)
    FString CharacterName;
    
    UPROPERTY(SaveGame)
    TSubclassOf<ACharacter> CharacterClass;
    
    UPROPERTY(SaveGame)
    int32 CharacterLevel;
    
    UPROPERTY(SaveGame)
    int32 CurrentHealth;
    
    UPROPERTY(SaveGame)
    int32 MaxHealth;
    
    UPROPERTY(SaveGame)
    int32 CurrentMana;
    
    UPROPERTY(SaveGame)
    int32 MaxMana;
    
    // === 属性点 ===
    UPROPERTY(SaveGame)
    TMap<FName, int32> Attributes; // Strength, Dexterity, Intelligence, etc.
    
    UPROPERTY(SaveGame)
    int32 UnspentAttributePoints;
    
    // === 技能树 ===
    UPROPERTY(SaveGame)
    TMap<FName, int32> SkillLevels;
    
    UPROPERTY(SaveGame)
    int32 UnspentSkillPoints;
    
    UPROPERTY(SaveGame)
    TArray<FName> ActiveSkills;
    
    // === 任务系统 ===
    UPROPERTY(SaveGame)
    TMap<FName, FRPGQuestData> QuestProgress;
    
    UPROPERTY(SaveGame)
    FName ActiveMainQuest;
    
    UPROPERTY(SaveGame)
    TArray<FName> CompletedQuests;
    
    // === 物品和装备 ===
    UPROPERTY(SaveGame)
    TArray<FRPGInventoryItem> Inventory;
    
    UPROPERTY(SaveGame)
    TMap<ERPGEquipSlot, FRPGEquipmentItem> Equipment;
    
    UPROPERTY(SaveGame)
    int32 Gold;
    
    // === 世界状态 ===
    UPROPERTY(SaveGame)
    TMap<FName, bool> WorldFlags; // NPC死亡、门已开启等
    
    UPROPERTY(SaveGame)
    TMap<FName, int32> NPCRelationships; // 声望系统
    
    UPROPERTY(SaveGame)
    TSet<FName> DiscoveredLocations;
    
    UPROPERTY(SaveGame)
    FName CurrentLocation;
    
    UPROPERTY(SaveGame)
    FVector PlayerPosition;
    
    UPROPERTY(SaveGame)
    FRotator PlayerRotation;
};

/**
 * RPG 任务数据
 */
USTRUCT(BlueprintType)
struct FRPGQuestData
{
    GENERATED_BODY()
    
    UPROPERTY(SaveGame)
    FName QuestID;
    
    UPROPERTY(SaveGame)
    EQuestStatus Status;
    
    UPROPERTY(SaveGame)
    TMap<FName, int32> ObjectiveProgress; // ObjectiveID -> Current Count
    
    UPROPERTY(SaveGame)
    TArray<FName> CompletedObjectives;
    
    UPROPERTY(SaveGame)
    FDateTime AcceptedTime;
    
    UPROPERTY(SaveGame)
    int32 QuestStage;
};

/**
 * RPG 物品数据
 */
USTRUCT(BlueprintType)
struct FRPGInventoryItem
{
    GENERATED_BODY()
    
    UPROPERTY(SaveGame)
    FName ItemID;
    
    UPROPERTY(SaveGame)
    int32 StackCount;
    
    UPROPERTY(SaveGame)
    int32 Durability;
    
    UPROPERTY(SaveGame)
    TArray<FName> Enchantments;
    
    UPROPERTY(SaveGame)
    TMap<FName, float> RandomStats; // 随机属性
    
    UPROPERTY(SaveGame)
    FGuid UniqueID;
};
```

### 8.2 RPG 存档管理器

```cpp
UCLASS()
class ULyraRPGSaveManager : public ULyraSaveGameManager
{
    GENERATED_BODY()
    
public:
    // 保存完整游戏状态
    void SaveCompleteGameState()
    {
        ULyraRPGSaveGame* RPGSave = Cast<ULyraRPGSaveGame>(CurrentSave);
        if (!RPGSave)
        {
            return;
        }
        
        // 保存玩家角色数据
        if (APlayerController* PC = GetWorld()->GetFirstPlayerController())
        {
            if (ALyraRPGCharacter* Character = Cast<ALyraRPGCharacter>(PC->GetPawn()))
            {
                SaveCharacterData(RPGSave, Character);
            }
        }
        
        // 保存世界状态
        SaveWorldState(RPGSave);
        
        // 保存任务进度
        SaveQuestProgress(RPGSave);
        
        // 执行实际保存
        SaveGame(TEXT("RPGSave"), true);
    }
    
private:
    void SaveCharacterData(ULyraRPGSaveGame* SaveGame, ALyraRPGCharacter* Character)
    {
        SaveGame->CharacterName = Character->GetCharacterName();
        SaveGame->CharacterClass = Character->GetClass();
        SaveGame->CharacterLevel = Character->GetLevel();
        
        // 保存生命值和法力值
        if (ULyraHealthComponent* HealthComp = Character->FindComponentByClass<ULyraHealthComponent>())
        {
            SaveGame->CurrentHealth = HealthComp->GetHealth();
            SaveGame->MaxHealth = HealthComp->GetMaxHealth();
        }
        
        if (ULyraManaComponent* ManaComp = Character->FindComponentByClass<ULyraManaComponent>())
        {
            SaveGame->CurrentMana = ManaComp->GetMana();
            SaveGame->MaxMana = ManaComp->GetMaxMana();
        }
        
        // 保存属性
        SaveGame->Attributes = Character->GetAttributes();
        SaveGame->UnspentAttributePoints = Character->GetUnspentAttributePoints();
        
        // 保存技能
        SaveGame->SkillLevels = Character->GetSkillLevels();
        SaveGame->ActiveSkills = Character->GetActiveSkills();
        
        // 保存背包
        if (ULyraInventoryComponent* InventoryComp = Character->FindComponentByClass<ULyraInventoryComponent>())
        {
            SaveGame->Inventory = InventoryComp->GetAllItems();
            SaveGame->Gold = InventoryComp->GetGold();
        }
        
        // 保存装备
        if (ULyraEquipmentComponent* EquipmentComp = Character->FindComponentByClass<ULyraEquipmentComponent>())
        {
            SaveGame->Equipment = EquipmentComp->GetAllEquipment();
        }
        
        // 保存位置
        SaveGame->PlayerPosition = Character->GetActorLocation();
        SaveGame->PlayerRotation = Character->GetActorRotation();
    }
    
    void SaveWorldState(ULyraRPGSaveGame* SaveGame)
    {
        // 保存所有可保存的 Actor
        for (TActorIterator<AActor> It(GetWorld()); It; ++It)
        {
            AActor* Actor = *It;
            if (Actor->Implements<ULyraSavableInterface>())
            {
                FName ActorID = ILyraSavableInterface::Execute_GetSavableActorID(Actor);
                TArray<uint8> ActorData = ILyraSavableInterface::Execute_SerializeActorData(Actor);
                
                // 存储到世界标志或自定义结构
                // SaveGame->WorldFlags.Add(ActorID, ...);
            }
        }
        
        // 保存 NPC 状态
        if (ULyraRPGGameInstance* GameInstance = GetWorld()->GetGameInstance<ULyraRPGGameInstance>())
        {
            SaveGame->NPCRelationships = GameInstance->GetNPCRelationships();
            SaveGame->WorldFlags = GameInstance->GetWorldFlags();
        }
    }
    
    void SaveQuestProgress(ULyraRPGSaveGame* SaveGame)
    {
        if (ULyraQuestManager* QuestManager = GetWorld()->GetSubsystem<ULyraQuestManager>())
        {
            SaveGame->QuestProgress = QuestManager->GetAllQuestData();
            SaveGame->ActiveMainQuest = QuestManager->GetActiveMainQuest();
            SaveGame->CompletedQuests = QuestManager->GetCompletedQuests();
        }
    }
    
public:
    // 加载游戏状态
    void LoadCompleteGameState()
    {
        if (!LoadGame(TEXT("RPGSave")))
        {
            UE_LOG(LogLyraSave, Warning, TEXT("Failed to load RPG save game"));
            return;
        }
        
        ULyraRPGSaveGame* RPGSave = Cast<ULyraRPGSaveGame>(CurrentSave);
        if (!RPGSave)
        {
            return;
        }
        
        // 恢复玩家角色
        RestoreCharacterData(RPGSave);
        
        // 恢复世界状态
        RestoreWorldState(RPGSave);
        
        // 恢复任务进度
        RestoreQuestProgress(RPGSave);
    }
    
private:
    void RestoreCharacterData(ULyraRPGSaveGame* SaveGame)
    {
        APlayerController* PC = GetWorld()->GetFirstPlayerController();
        if (!PC)
        {
            return;
        }
        
        // 生成或获取角色
        ALyraRPGCharacter* Character = Cast<ALyraRPGCharacter>(PC->GetPawn());
        if (!Character)
        {
            // 生成角色
            FActorSpawnParameters SpawnParams;
            SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
            
            Character = GetWorld()->SpawnActor<ALyraRPGCharacter>(
                SaveGame->CharacterClass,
                SaveGame->PlayerPosition,
                SaveGame->PlayerRotation,
                SpawnParams
            );
            
            PC->Possess(Character);
        }
        
        // 恢复基础数据
        Character->SetCharacterName(SaveGame->CharacterName);
        Character->SetLevel(SaveGame->CharacterLevel);
        
        // 恢复生命值
        if (ULyraHealthComponent* HealthComp = Character->FindComponentByClass<ULyraHealthComponent>())
        {
            HealthComp->SetMaxHealth(SaveGame->MaxHealth);
            HealthComp->SetHealth(SaveGame->CurrentHealth);
        }
        
        // 恢复属性
        Character->SetAttributes(SaveGame->Attributes);
        Character->SetUnspentAttributePoints(SaveGame->UnspentAttributePoints);
        
        // 恢复技能
        Character->SetSkillLevels(SaveGame->SkillLevels);
        Character->SetActiveSkills(SaveGame->ActiveSkills);
        
        // 恢复背包
        if (ULyraInventoryComponent* InventoryComp = Character->FindComponentByClass<ULyraInventoryComponent>())
        {
            InventoryComp->SetInventory(SaveGame->Inventory);
            InventoryComp->SetGold(SaveGame->Gold);
        }
        
        // 恢复装备
        if (ULyraEquipmentComponent* EquipmentComp = Character->FindComponentByClass<ULyraEquipmentComponent>())
        {
            EquipmentComp->SetEquipment(SaveGame->Equipment);
        }
    }
    
    void RestoreWorldState(ULyraRPGSaveGame* SaveGame)
    {
        if (ULyraRPGGameInstance* GameInstance = GetWorld()->GetGameInstance<ULyraRPGGameInstance>())
        {
            GameInstance->SetNPCRelationships(SaveGame->NPCRelationships);
            GameInstance->SetWorldFlags(SaveGame->WorldFlags);
        }
    }
    
    void RestoreQuestProgress(ULyraRPGSaveGame* SaveGame)
    {
        if (ULyraQuestManager* QuestManager = GetWorld()->GetSubsystem<ULyraQuestManager>())
        {
            QuestManager->RestoreQuestData(SaveGame->QuestProgress);
            QuestManager->SetActiveMainQuest(SaveGame->ActiveMainQuest);
        }
    }
};
```

---

## 9. 实战案例 2：跨平台云存档

### 9.1 统一云存档接口

```cpp
/**
 * 跨平台云存档适配器
 */
UCLASS()
class ULyraCrossPlatformSaveAdapter : public UObject
{
    GENERATED_BODY()
    
public:
    void Initialize()
    {
        // 根据平台选择对应的云存档提供者
#if PLATFORM_WINDOWS || PLATFORM_MAC || PLATFORM_LINUX
        if (IsSteamAvailable())
        {
            CloudProvider = NewObject<ULyraSteamCloudSaveProvider>(this);
        }
        else
        {
            CloudProvider = NewObject<ULyraEOSCloudSaveProvider>(this);
        }
#elif PLATFORM_PS5
        CloudProvider = NewObject<ULyraPSNCloudSaveProvider>(this);
#elif PLATFORM_XBOXONE || PLATFORM_XSX
        CloudProvider = NewObject<ULyraXboxLiveCloudSaveProvider>(this);
#else
        CloudProvider = NewObject<ULyraEOSCloudSaveProvider>(this); // 默认使用 EOS
#endif
        
        UE_LOG(LogLyraSave, Log, TEXT("Cloud save provider initialized: %s"),
            *CloudProvider->GetClass()->GetName());
    }
    
    // 统一的上传接口
    void UploadSaveToCloud(const FString& SlotName, ULyraProgressSaveGame* SaveGame)
    {
        if (!CloudProvider)
        {
            UE_LOG(LogLyraSave, Error, TEXT("Cloud provider not initialized"));
            return;
        }
        
        // 序列化存档
        TArray<uint8> SaveData = SerializeSaveGame(SaveGame);
        
        // 压缩
        SaveData = FLyraSaveDataSerializer::CompressData(SaveData);
        
        // 加密
        SaveData = FLyraAESEncryption::EncryptData(SaveData);
        
        // 上传
        CloudProvider->UploadSave(
            SlotName,
            SaveData,
            FOnCloudSaveCompleted::CreateUObject(this, &ULyraCrossPlatformSaveAdapter::OnUploadCompleted)
        );
    }
    
    // 统一的下载接口
    void DownloadSaveFromCloud(const FString& SlotName)
    {
        if (!CloudProvider)
        {
            return;
        }
        
        CloudProvider->DownloadSave(
            SlotName,
            FOnCloudSaveLoaded::CreateUObject(this, &ULyraCrossPlatformSaveAdapter::OnDownloadCompleted)
        );
    }
    
private:
    UPROPERTY()
    ULyraCloudSaveProvider* CloudProvider;
    
    TArray<uint8> SerializeSaveGame(ULyraProgressSaveGame* SaveGame)
    {
        TArray<uint8> OutData;
        FMemoryWriter MemoryWriter(OutData, true);
        FObjectAndNameAsStringProxyArchive Archive(MemoryWriter, false);
        SaveGame->Serialize(Archive);
        return OutData;
    }
    
    ULyraProgressSaveGame* DeserializeSaveGame(const TArray<uint8>& Data)
    {
        ULyraProgressSaveGame* SaveGame = NewObject<ULyraProgressSaveGame>();
        FMemoryReader MemoryReader(Data, true);
        FObjectAndNameAsStringProxyArchive Archive(MemoryReader, false);
        SaveGame->Serialize(Archive);
        return SaveGame;
    }
    
    void OnUploadCompleted(bool bSuccess, const FString& Error)
    {
        if (bSuccess)
        {
            UE_LOG(LogLyraSave, Log, TEXT("Cloud save uploaded successfully"));
        }
        else
        {
            UE_LOG(LogLyraSave, Error, TEXT("Cloud save upload failed: %s"), *Error);
        }
    }
    
    void OnDownloadCompleted(const TArray<uint8>& Data, bool bSuccess, const FString& Error)
    {
        if (!bSuccess)
        {
            UE_LOG(LogLyraSave, Error, TEXT("Cloud save download failed: %s"), *Error);
            return;
        }
        
        // 解密
        TArray<uint8> DecryptedData = FLyraAESEncryption::DecryptData(Data);
        
        // 解压
        TArray<uint8> DecompressedData = FLyraSaveDataSerializer::DecompressData(DecryptedData);
        
        // 反序列化
        ULyraProgressSaveGame* SaveGame = DeserializeSaveGame(DecompressedData);
        
        // 触发加载完成事件
        OnCloudSaveDownloaded.Broadcast(SaveGame);
    }
    
public:
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnCloudSaveDownloaded, ULyraProgressSaveGame*, SaveGame);
    UPROPERTY(BlueprintAssignable)
    FOnCloudSaveDownloaded OnCloudSaveDownloaded;
    
    bool IsSteamAvailable() const
    {
#if WITH_STEAM
        return SteamAPI_IsSteamRunning();
#else
        return false;
#endif
    }
};
```

### 9.2 自动同步管理

```cpp
/**
 * 自动云同步管理器
 */
UCLASS()
class ULyraAutoSyncManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override
    {
        Super::Initialize(Collection);
        
        // 初始化云存档适配器
        CrossPlatformAdapter = NewObject<ULyraCrossPlatformSaveAdapter>(this);
        CrossPlatformAdapter->Initialize();
        
        // 绑定网络状态变化
        IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get();
        if (OnlineSubsystem)
        {
            OnlineSubsystem->AddOnConnectionStatusChangedDelegate_Handle(
                FOnConnectionStatusChangedDelegate::CreateUObject(
                    this,
                    &ULyraAutoSyncManager::OnNetworkStatusChanged
                )
            );
        }
        
        // 启动周期同步
        GetWorld()->GetTimerManager().SetTimer(
            SyncTimerHandle,
            this,
            &ULyraAutoSyncManager::PerformAutoSync,
            AutoSyncInterval,
            true
        );
    }
    
    void PerformAutoSync()
    {
        // 检查网络连接
        if (!IsNetworkAvailable())
        {
            UE_LOG(LogLyraSave, Log, TEXT("Skipping auto-sync: No network connection"));
            return;
        }
        
        // 检查是否有待上传的本地存档
        if (bHasPendingLocalSave)
        {
            UE_LOG(LogLyraSave, Log, TEXT("Uploading pending local save to cloud"));
            UploadLocalSaveToCloud();
        }
        
        // 检查云端是否有更新
        CheckForCloudUpdates();
    }
    
private:
    UPROPERTY()
    ULyraCrossPlatformSaveAdapter* CrossPlatformAdapter;
    
    FTimerHandle SyncTimerHandle;
    float AutoSyncInterval = 600.0f; // 10分钟
    
    bool bHasPendingLocalSave = false;
    
    bool IsNetworkAvailable() const
    {
        IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get();
        if (!OnlineSubsystem)
        {
            return false;
        }
        
        return OnlineSubsystem->GetConnectionStatus() == EOnlineServerConnectionStatus::Connected;
    }
    
    void OnNetworkStatusChanged(EOnlineServerConnectionStatus::Type LastConnectionStatus,
                                EOnlineServerConnectionStatus::Type ConnectionStatus)
    {
        if (ConnectionStatus == EOnlineServerConnectionStatus::Connected)
        {
            UE_LOG(LogLyraSave, Log, TEXT("Network connected - triggering cloud sync"));
            PerformAutoSync();
        }
    }
    
    void UploadLocalSaveToCloud()
    {
        ULyraSaveGameManager* SaveManager = GetGameInstance()->GetSubsystem<ULyraSaveGameManager>();
        if (!SaveManager)
        {
            return;
        }
        
        ULyraProgressSaveGame* CurrentSave = SaveManager->GetCurrentSave();
        CrossPlatformAdapter->UploadSaveToCloud(TEXT("MainSave"), CurrentSave);
        
        bHasPendingLocalSave = false;
    }
    
    void CheckForCloudUpdates()
    {
        CrossPlatformAdapter->DownloadSaveFromCloud(TEXT("MainSave"));
    }
};
```

---

## 10. 数据迁移和升级

### 10.1 自动版本迁移

```cpp
void ULyraProgressSaveGame::PostLoad()
{
    Super::PostLoad();
    
    // 检查版本
    int32 LatestVersion = GetLatestVersion();
    if (SaveVersion < LatestVersion)
    {
        UE_LOG(LogLyraSave, Warning, TEXT("Save game version mismatch: %d -> %d. Migrating..."),
            SaveVersion, LatestVersion);
        
        MigrateFromVersion(SaveVersion);
        SaveVersion = LatestVersion;
        
        // 标记为需要重新保存
        bIsDirty = true;
    }
}

void ULyraProgressSaveGame::MigrateFromVersion(int32 OldVersion)
{
    // 版本 1 -> 2：添加装备系统
    if (OldVersion < 2)
    {
        UE_LOG(LogLyraSave, Log, TEXT("Migrating save from v1 to v2: Adding equipment system"));
        
        // 迁移逻辑...
    }
    
    // 版本 2 -> 3：添加成就系统
    if (OldVersion < 3)
    {
        UE_LOG(LogLyraSave, Log, TEXT("Migrating save from v2 to v3: Adding achievement system"));
        
        // 迁移逻辑...
    }
    
    // 版本 3 -> 4：重构物品系统
    if (OldVersion < 4)
    {
        UE_LOG(LogLyraSave, Log, TEXT("Migrating save from v3 to v4: Refactoring item system"));
        
        // 将旧的物品数据转换为新格式
        TArray<FRPGInventoryItem> NewInventory;
        for (const FInventoryItemData& OldItem : InventoryItems)
        {
            FRPGInventoryItem NewItem;
            NewItem.ItemID = OldItem.ItemID;
            NewItem.StackCount = OldItem.Quantity;
            NewItem.Durability = OldItem.Durability;
            NewItem.UniqueID = FGuid::NewGuid();
            
            NewInventory.Add(NewItem);
        }
        
        // 替换为新数据
        Inventory_V4 = NewInventory;
    }
}
```

### 10.2 向后兼容性

```cpp
/**
 * 存档兼容性管理器
 */
class FLyraSaveCompatibilityManager
{
public:
    // 检查存档是否可兼容
    static bool IsSaveCompatible(int32 SaveVersion, int32 GameVersion)
    {
        // 允许加载不超过 2 个大版本的旧存档
        return (GameVersion - SaveVersion) <= 2;
    }
    
    // 尝试修复损坏的存档
    static ULyraProgressSaveGame* TryRepairSave(const TArray<uint8>& CorruptedData)
    {
        UE_LOG(LogLyraSave, Warning, TEXT("Attempting to repair corrupted save game"));
        
        // 尝试跳过损坏的数据段
        FMemoryReader Reader(CorruptedData, true);
        Reader.ArIgnoreArchetypeRef = true;
        Reader.ArIgnoreClassRef = true;
        Reader.ArIgnoreOuterRef = true;
        
        ULyraProgressSaveGame* SaveGame = NewObject<ULyraProgressSaveGame>();
        
        // 逐个字段尝试读取
        try
        {
            Reader << SaveGame->SaveVersion;
            Reader << SaveGame->PlayerLevel;
            Reader << SaveGame->PlayerExperience;
            // ... 继续读取其他字段
        }
        catch (...)
        {
            UE_LOG(LogLyraSave, Error, TEXT("Failed to repair save game"));
            return nullptr;
        }
        
        // 设置默认值for 读取失败的字段
        SaveGame->SetToDefaults();
        
        return SaveGame;
    }
    
    // 创建紧急备份
    static void CreateEmergencyBackup(ULyraProgressSaveGame* SaveGame)
    {
        FString BackupSlotName = FString::Printf(
            TEXT("Emergency_Backup_%s"),
            *FDateTime::Now().ToString()
        );
        
        UGameplayStatics::SaveGameToSlot(SaveGame, BackupSlotName, 0);
        
        UE_LOG(LogLyraSave, Log, TEXT("Emergency backup created: %s"), *BackupSlotName);
    }
};
```

---

## 11. 性能优化

### 11.1 增量保存

```cpp
/**
 * 增量保存 - 只保存变化的数据
 */
class FLyraIncrementalSaveSystem
{
public:
    struct FDeltaData
    {
        TMap<FName, FVariant> ChangedProperties;
        FDateTime ChangeTime;
    };
    
    static void SaveDelta(ULyraProgressSaveGame* CurrentSave, ULyraProgressSaveGame* PreviousSave)
    {
        FDeltaData Delta;
        
        // 比较并记录变化
        if (CurrentSave->PlayerLevel != PreviousSave->PlayerLevel)
        {
            Delta.ChangedProperties.Add(TEXT("PlayerLevel"), FVariant(CurrentSave->PlayerLevel));
        }
        
        if (CurrentSave->Gold != PreviousSave->Gold)
        {
            Delta.ChangedProperties.Add(TEXT("Gold"), FVariant(CurrentSave->Gold));
        }
        
        // 比较数组和Map（更复杂）
        CompareMaps(CurrentSave->QuestData, PreviousSave->QuestData, Delta);
        
        // 保存增量数据
        SaveDeltaToFile(Delta);
    }
    
private:
    static void SaveDeltaToFile(const FDeltaData& Delta)
    {
        FString DeltaFileName = FString::Printf(
            TEXT("Delta_%s.sav"),
            *FDateTime::Now().ToString(TEXT("%Y%m%d_%H%M%S"))
        );
        
        // 序列化增量数据（体积小，速度快）
        // ...
    }
    
    static void CompareMaps(const TMap<FName, FQuestProgress>& Current,
                           const TMap<FName, FQuestProgress>& Previous,
                           FDeltaData& OutDelta)
    {
        for (const auto& CurrentQuest : Current)
        {
            const FQuestProgress* PreviousQuest = Previous.Find(CurrentQuest.Key);
            if (!PreviousQuest || *PreviousQuest != CurrentQuest.Value)
            {
                OutDelta.ChangedProperties.Add(
                    CurrentQuest.Key,
                    FVariant() // 这里需要自定义序列化
                );
            }
        }
    }
};
```

### 11.2 异步保存优化

```cpp
/**
 * 异步保存任务
 */
class FLyraAsyncSaveTask : public FNonAbandonableTask
{
    friend class FAutoDeleteAsyncTask<FLyraAsyncSaveTask>;
    
public:
    FLyraAsyncSaveTask(ULyraProgressSaveGame* InSaveGame, const FString& InSlotName)
        : SaveGame(InSaveGame)
        , SlotName(InSlotName)
    {
    }
    
    void DoWork()
    {
        // 序列化存档（耗时操作）
        TArray<uint8> SaveData;
        FMemoryWriter MemoryWriter(SaveData, true);
        FObjectAndNameAsStringProxyArchive Archive(MemoryWriter, false);
        SaveGame->Serialize(Archive);
        
        // 压缩（耗时操作）
        TArray<uint8> CompressedData = FLyraSaveDataSerializer::CompressData(SaveData);
        
        // 加密（耗时操作）
        TArray<uint8> EncryptedData = FLyraAESEncryption::EncryptData(CompressedData);
        
        // 写入磁盘（IO 操作）
        FString SavePath = FPaths::ProjectSavedDir() / TEXT("SaveGames") / (SlotName + TEXT(".sav"));
        FFileHelper::SaveArrayToFile(EncryptedData, *SavePath);
        
        UE_LOG(LogLyraSave, Log, TEXT("Async save completed: %s (%d bytes -> %d bytes)"),
            *SlotName, SaveData.Num(), EncryptedData.Num());
    }
    
    FORCEINLINE TStatId GetStatId() const
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FLyraAsyncSaveTask, STATGROUP_ThreadPoolAsyncTasks);
    }
    
private:
    ULyraProgressSaveGame* SaveGame;
    FString SlotName;
};

// 使用方式
void ULyraSaveGameManager::AsyncSaveOptimized(const FString& SlotName)
{
    (new FAutoDeleteAsyncTask<FLyraAsyncSaveTask>(CurrentSave, SlotName))->StartBackgroundTask();
}
```

### 11.3 内存优化

```cpp
/**
 * 大型存档的流式加载
 */
class FLyraStreamingSaveLoader
{
public:
    static void StreamLoadSave(const FString& SlotName, TFunction<void(ULyraProgressSaveGame*)> OnComplete)
    {
        // 分块读取大型存档文件
        FString SavePath = FPaths::ProjectSavedDir() / TEXT("SaveGames") / (SlotName + TEXT(".sav"));
        
        // 先读取头部信息
        TArray<uint8> HeaderData;
        if (!ReadFileChunk(SavePath, 0, 1024, HeaderData))
        {
            OnComplete(nullptr);
            return;
        }
        
        // 解析版本和元数据
        int32 SaveVersion = ParseVersion(HeaderData);
        int32 TotalSize = ParseTotalSize(HeaderData);
        
        // 流式读取剩余数据
        const int32 ChunkSize = 64 * 1024; // 64KB chunks
        TArray<uint8> FullData;
        FullData.Reserve(TotalSize);
        
        for (int32 Offset = 1024; Offset < TotalSize; Offset += ChunkSize)
        {
            TArray<uint8> ChunkData;
            ReadFileChunk(SavePath, Offset, ChunkSize, ChunkData);
            FullData.Append(ChunkData);
        }
        
        // 反序列化
        ULyraProgressSaveGame* SaveGame = DeserializeSaveGame(FullData);
        OnComplete(SaveGame);
    }
    
private:
    static bool ReadFileChunk(const FString& FilePath, int64 Offset, int64 Size, TArray<uint8>& OutData)
    {
        IFileHandle* FileHandle = FPlatformFileManager::Get().GetPlatformFile().OpenRead(*FilePath);
        if (!FileHandle)
        {
            return false;
        }
        
        FileHandle->Seek(Offset);
        OutData.SetNum(Size);
        bool bSuccess = FileHandle->Read(OutData.GetData(), Size);
        
        delete FileHandle;
        return bSuccess;
    }
    
    static int32 ParseVersion(const TArray<uint8>& HeaderData)
    {
        FMemoryReader Reader(HeaderData, true);
        int32 Version;
        Reader << Version;
        return Version;
    }
    
    static int32 ParseTotalSize(const TArray<uint8>& HeaderData)
    {
        FMemoryReader Reader(HeaderData, true);
        int32 Version, TotalSize;
        Reader << Version << TotalSize;
        return TotalSize;
    }
};
```

---

## 12. 总结与最佳实践

### 12.1 核心设计原则

1. **分层架构**：基础存档 → 业务存档 → 管理器
2. **异步优先**：避免阻塞主线程，提升用户体验
3. **版本管理**：从一开始就规划版本迁移策略
4. **数据校验**：使用校验和和签名防止数据损坏和作弊
5. **平台抽象**：统一接口，方便跨平台部署

### 12.2 实施建议

```cpp
/**
 * 存档系统初始化检查清单
 */
class FLyraSaveSystemChecklist
{
public:
    static void ValidateSetup(UWorld* World)
    {
        // 1. 检查存档管理器
        ULyraSaveGameManager* SaveManager = World->GetGameInstance()->GetSubsystem<ULyraSaveGameManager>();
        check(SaveManager);
        
        // 2. 检查云存档提供者
        // ...
        
        // 3. 验证存档路径可写
        FString SaveDir = FPaths::ProjectSavedDir() / TEXT("SaveGames");
        check(IFileManager::Get().DirectoryExists(*SaveDir) || IFileManager::Get().MakeDirectory(*SaveDir, true));
        
        // 4. 测试保存和加载
        ULyraProgressSaveGame* TestSave = NewObject<ULyraProgressSaveGame>();
        TestSave->PlayerLevel = 999;
        bool bSaveSuccess = UGameplayStatics::SaveGameToSlot(TestSave, TEXT("__test__"), 0);
        check(bSaveSuccess);
        
        ULyraProgressSaveGame* LoadedSave = Cast<ULyraProgressSaveGame>(
            UGameplayStatics::LoadGameFromSlot(TEXT("__test__"), 0)
        );
        check(LoadedSave && LoadedSave->PlayerLevel == 999);
        
        // 5. 清理测试存档
        UGameplayStatics::DeleteGameInSlot(TEXT("__test__"), 0);
        
        UE_LOG(LogLyraSave, Log, TEXT("Save system validation passed!"));
    }
};
```

### 12.3 常见陷阱

1. **忘记版本迁移**：导致旧存档无法加载
2. **同步保存在主线程**：造成卡顿
3. **未加密敏感数据**：容易被修改
4. **过于频繁的自动保存**：影响性能
5. **未处理磁盘空间不足**：导致数据丢失

### 12.4 性能指标

```cpp
// 性能监控
DECLARE_STATS_GROUP(TEXT("SaveSystem"), STATGROUP_SaveSystem, STATCAT_Advanced);
DECLARE_CYCLE_STAT(TEXT("Save Game"), STAT_SaveGame, STATGROUP_SaveSystem);
DECLARE_CYCLE_STAT(TEXT("Load Game"), STAT_LoadGame, STATGROUP_SaveSystem);
DECLARE_CYCLE_STAT(TEXT("Serialize"), STAT_Serialize, STATGROUP_SaveSystem);
DECLARE_CYCLE_STAT(TEXT("Compress"), STAT_Compress, STATGROUP_SaveSystem);
DECLARE_CYCLE_STAT(TEXT("Encrypt"), STAT_Encrypt, STATGROUP_SaveSystem);

void ULyraSaveGameManager::SaveGame(const FString& SlotName, bool bAsyncSave)
{
    SCOPE_CYCLE_COUNTER(STAT_SaveGame);
    
    // 保存逻辑...
}
```

**推荐目标：**
- 本地保存：< 100ms（异步）
- 云存档上传：< 5s（后台）
- 加载存档：< 200ms
- 存档文件大小：< 1MB（压缩后）

---

## 总结

本章全面介绍了如何在 UE5 Lyra 基础上构建生产级的存档系统。通过学习 Lyra 的 Settings 系统架构，我们了解了分层设计、异步加载、版本管理等核心概念。结合两个实战案例（RPG 存档和跨平台云存档），展示了从基础到高级的完整实现路径。

**关键要点：**

✅ 基于 Lyra 的分层架构（Local/Shared Settings）扩展业务存档  
✅ 使用版本管理支持数据迁移和向后兼容  
✅ 实现跨平台云存档（Steam/EOS/PSN/Xbox）  
✅ 采用加密和校验保证数据安全  
✅ 优化性能（异步、增量、流式加载）  
✅ 提供完整的代码示例和最佳实践

掌握本章内容后，你将能够为自己的游戏项目构建可靠、安全、高效的存档系统，无论是单机 RPG 还是跨平台多人游戏都能游刃有余！
