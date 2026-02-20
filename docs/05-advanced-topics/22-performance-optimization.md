# 性能分析与优化实战

## 引言

性能优化是游戏开发的永恒话题。一款优秀的游戏不仅要有出色的玩法和画面，更需要在各种硬件平台上保持流畅的运行体验。Lyra作为Epic官方的示例项目，在性能优化方面展现了大量最佳实践，从CPU到GPU，从内存到网络，从引擎层面到游戏逻辑，都有深入的优化考量。

本文将深入剖析Lyra中的性能优化技术，结合UE5的性能分析工具，带你掌握从性能瓶颈诊断到系统化优化的完整流程。无论你是要优化大型多人在线游戏，还是适配移动端设备，这里都能找到答案。

## UE5 性能分析工具全景

### Unreal Insights：终极性能分析利器

Unreal Insights是UE5最强大的性能分析工具，它能够记录并可视化游戏运行时的几乎所有性能数据。

#### 启动Unreal Insights

```bash
# 方法1：从编辑器启动
# Tools -> Unreal Insights -> Launch Unreal Insights

# 方法2：直接启动跟踪录制
UnrealInsights.exe -trace=cpu,gpu,frame,bookmark,log

# 方法3：运行时启动跟踪
# 在控制台输入：
Trace.Start
Trace.Stop
```

#### 关键分析视图

**1. Timing View（时序视图）**

这是最常用的视图，展示了每一帧的详细执行时序：

```cpp
// 在代码中添加性能标记
void ULyraAssetManager::StartInitialLoading()
{
    SCOPED_BOOT_TIMING("ULyraAssetManager::StartInitialLoading");
    
    Super::StartInitialLoading();
    
    STARTUP_JOB(InitializeGameplayCueManager());
    {
        STARTUP_JOB_WEIGHTED(GetGameData(), 25.f);
    }
    
    DoAllStartupJobs();
}
```

**2. CPU Profiler**

CPU Profiler能够精确定位每个函数的执行时间：

```cpp
// Lyra中的性能计数器使用示例
void SActorCanvas::UpdateCanvas(double InCurrentTime, float InDeltaTime)
{
    QUICK_SCOPE_CYCLE_COUNTER(STAT_SActorCanvas_UpdateCanvas);
    
    // 更新指示器画布的逻辑
    // ...
}

void SActorCanvas::OnArrangeChildren(const FGeometry& AllottedGeometry, 
                                      FArrangedChildren& ArrangedChildren) const
{
    QUICK_SCOPE_CYCLE_COUNTER(STAT_SActorCanvas_OnArrangeChildren);
    
    // 布局子控件的逻辑
    // ...
}
```

**3. GPU Profiler**

分析GPU渲染管线的性能瓶颈：

```cpp
// 查看GPU时间
// 控制台命令：
stat GPU
profilegpu
```

### Stat 命令：快速性能诊断

UE5提供了丰富的stat命令，用于实时监控各项性能指标。

#### 核心Stat命令

```cpp
// 显示基础性能统计
stat FPS          // 帧率
stat Unit         // 帧时间分解（Game Thread, Render Thread, GPU）
stat UnitGraph    // 帧时间图表

// CPU性能分析
stat Game         // 游戏逻辑耗时
stat GameThread   // 游戏线程详细统计
stat Engine       // 引擎系统统计
stat SceneRendering // 场景渲染统计

// 内存分析
stat Memory       // 内存使用概况
stat LLM          // 低级内存追踪器
stat MemoryStaticMesh // 静态网格内存

// GPU性能
stat GPU          // GPU时间统计
stat RHI          // 渲染硬件接口统计

// 网络性能
stat Net          // 网络统计
stat NetPlayerMovement // 玩家移动网络同步
```

#### Lyra性能统计子系统

Lyra实现了一个完整的性能统计子系统，可以在游戏中实时显示各项性能指标：

```cpp
// LyraPerformanceStatTypes.h
UENUM(BlueprintType)
enum class ELyraDisplayablePerformanceStat : uint8
{
    ClientFPS,                  // 客户端帧率
    ServerFPS,                  // 服务器帧率
    IdleTime,                   // 空闲等待时间（VSync）
    FrameTime,                  // 总帧时间
    FrameTime_GameThread,       // 游戏线程时间
    FrameTime_RenderThread,     // 渲染线程时间
    FrameTime_RHIThread,        // RHI线程时间
    FrameTime_GPU,              // GPU时间
    Ping,                       // 网络延迟
    PacketLoss_Incoming,        // 入站丢包率
    PacketLoss_Outgoing,        // 出站丢包率
    PacketRate_Incoming,        // 入站包速率
    PacketRate_Outgoing,        // 出站包速率
    PacketSize_Incoming,        // 入站包大小
    PacketSize_Outgoing,        // 出站包大小
    Latency_Total,              // 总延迟
    Latency_Game,               // 游戏延迟
    Latency_Render,             // 渲染延迟
};
```

### Frontend Profiler：会话回放分析

Frontend Profiler提供了会话录制和回放功能，适合离线分析：

```cpp
// 启动会话录制
// 控制台命令：
stat startfile
stat stopfile

// 分析文件位置：
// Saved/Profiling/UnrealStats/
```

## CPU性能优化

CPU是游戏逻辑的核心执行单元，优化CPU性能是提升整体游戏表现的关键。

### Tick优化：减少不必要的更新

在UE中，Actor和Component默认每帧都会执行Tick函数，这会带来巨大的性能开销。Lyra通过精细控制Tick行为来优化性能。

#### 禁用不需要Tick的组件

```cpp
// LyraHealthComponent.cpp - 健康组件不需要每帧更新
ULyraHealthComponent::ULyraHealthComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    PrimaryComponentTick.bStartWithTickEnabled = false;
    PrimaryComponentTick.bCanEverTick = false;
    
    // 其他初始化...
}

// LyraPawnExtensionComponent.cpp - Pawn扩展组件不需要Tick
ULyraPawnExtensionComponent::ULyraPawnExtensionComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    PrimaryComponentTick.bStartWithTickEnabled = false;
    PrimaryComponentTick.bCanEverTick = false;
}

// LyraCharacter.cpp - 角色默认不开启Tick
ALyraCharacter::ALyraCharacter(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    PrimaryActorTick.bStartWithTickEnabled = false;
    
    // 在需要时动态开启Tick
}
```

#### 动态控制Tick频率

对于必须Tick的对象，可以通过调整Tick间隔来降低开销：

```cpp
// 自定义Tick间隔组件
UCLASS()
class UOptimizedTickComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    UOptimizedTickComponent()
    {
        PrimaryComponentTick.bCanEverTick = true;
        PrimaryComponentTick.bStartWithTickEnabled = true;
        
        // 设置Tick间隔（每0.1秒更新一次）
        PrimaryComponentTick.TickInterval = 0.1f;
    }
    
    virtual void TickComponent(float DeltaTime, 
                               ELevelTick TickType, 
                               FActorComponentTickFunction* ThisTickFunction) override
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
        
        // 只在需要时执行逻辑
        if (!ShouldTick())
        {
            return;
        }
        
        // 执行更新逻辑
        UpdateLogic(DeltaTime);
    }
    
private:
    bool ShouldTick() const
    {
        // 根据距离、可见性等条件判断
        return IsInRelevantRange() && IsVisible();
    }
    
    bool IsInRelevantRange() const
    {
        if (APlayerController* PC = GetWorld()->GetFirstPlayerController())
        {
            APawn* PlayerPawn = PC->GetPawn();
            if (PlayerPawn && GetOwner())
            {
                float Distance = FVector::Dist(
                    PlayerPawn->GetActorLocation(), 
                    GetOwner()->GetActorLocation()
                );
                return Distance < MaxRelevantDistance;
            }
        }
        return false;
    }
    
    UPROPERTY(EditAnywhere, Category = "Optimization")
    float MaxRelevantDistance = 5000.0f;
};
```

#### Tick组优化

合理设置Tick组可以控制执行顺序，避免一帧内的重复计算：

```cpp
// LyraPlayerSpawningManagerComponent.cpp
ULyraPlayerSpawningManagerComponent::ULyraPlayerSpawningManagerComponent(
    const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 在物理模拟之前执行
    PrimaryComponentTick.TickGroup = TG_PrePhysics;
    PrimaryComponentTick.bCanEverTick = true;
    PrimaryComponentTick.bAllowTickOnDedicatedServer = true;
    PrimaryComponentTick.bStartWithTickEnabled = false;
}
```

### 事件驱动架构：替代轮询

使用事件驱动代替每帧轮询，可以大幅降低CPU开销。

#### 实现事件驱动的状态变化监听

```cpp
// 事件驱动的属性变化监听
UCLASS()
class UEventDrivenAttributeComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(
        FOnHealthChanged, 
        float, OldHealth, 
        float, NewHealth
    );
    
    UEventDrivenAttributeComponent()
    {
        // 不需要Tick
        PrimaryComponentTick.bCanEverTick = false;
    }
    
    void SetHealth(float NewHealth)
    {
        if (!FMath::IsNearlyEqual(CurrentHealth, NewHealth))
        {
            float OldHealth = CurrentHealth;
            CurrentHealth = FMath::Clamp(NewHealth, 0.0f, MaxHealth);
            
            // 触发事件，而不是每帧检查
            OnHealthChanged.Broadcast(OldHealth, CurrentHealth);
            
            if (CurrentHealth <= 0.0f)
            {
                OnHealthDepleted.Broadcast();
            }
        }
    }
    
    UPROPERTY(BlueprintAssignable, Category = "Health")
    FOnHealthChanged OnHealthChanged;
    
    DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnHealthDepleted);
    
    UPROPERTY(BlueprintAssignable, Category = "Health")
    FOnHealthDepleted OnHealthDepleted;
    
private:
    UPROPERTY(EditAnywhere, Category = "Health")
    float CurrentHealth = 100.0f;
    
    UPROPERTY(EditAnywhere, Category = "Health")
    float MaxHealth = 100.0f;
};
```

#### GAS中的事件驱动

Lyra大量使用GAS（Gameplay Ability System），它本身就是事件驱动的：

```cpp
// 监听属性变化（而不是每帧检查）
void ULyraHealthComponent::InitializeWithAbilitySystem(
    ULyraAbilitySystemComponent* InASC)
{
    AActor* Owner = GetOwner();
    check(Owner);
    
    if (AbilitySystemComponent)
    {
        UE_LOG(LogLyra, Error, 
               TEXT("Health component for owner [%s] has already been initialized!"), 
               *GetNameSafe(Owner));
        return;
    }
    
    AbilitySystemComponent = InASC;
    if (!AbilitySystemComponent)
    {
        UE_LOG(LogLyra, Error, 
               TEXT("Cannot initialize health component for owner [%s] with NULL ASC!"), 
               *GetNameSafe(Owner));
        return;
    }
    
    // 订阅健康属性变化事件
    HealthSet = AbilitySystemComponent->GetSet<ULyraHealthSet>();
    if (!HealthSet)
    {
        UE_LOG(LogLyra, Error, 
               TEXT("Cannot initialize health component for owner [%s] - no health set!"), 
               *GetNameSafe(Owner));
        return;
    }
    
    // 监听属性变化（事件驱动）
    AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
        ULyraHealthSet::GetHealthAttribute()
    ).AddUObject(this, &ThisClass::HandleHealthChanged);
    
    AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
        ULyraHealthSet::GetMaxHealthAttribute()
    ).AddUObject(this, &ThisClass::HandleMaxHealthChanged);
}

void ULyraHealthComponent::HandleHealthChanged(const FOnAttributeChangeData& ChangeData)
{
    // 只在属性真正变化时执行
    OnHealthChanged.Broadcast(
        this, 
        ChangeData.OldValue, 
        ChangeData.NewValue, 
        nullptr
    );
}
```

### 对象池：减少内存分配开销

频繁的对象创建和销毁会导致内存碎片和GC压力。对象池是解决这一问题的经典方案。

#### 泛用对象池实现

```cpp
// 通用对象池模板类
template<typename T>
class TObjectPool
{
public:
    TObjectPool(int32 InitialSize = 10, int32 MaxSize = 100)
        : MaxPoolSize(MaxSize)
    {
        Pool.Reserve(InitialSize);
    }
    
    // 从池中获取对象
    T* Acquire()
    {
        T* Object = nullptr;
        
        if (Pool.Num() > 0)
        {
            // 从池中取出
            Object = Pool.Pop();
            ActiveObjects.Add(Object);
        }
        else if (ActiveObjects.Num() + Pool.Num() < MaxPoolSize)
        {
            // 创建新对象
            Object = CreateNewObject();
            ActiveObjects.Add(Object);
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Object pool exceeded max size!"));
        }
        
        if (Object)
        {
            OnAcquire(Object);
        }
        
        return Object;
    }
    
    // 归还对象到池
    void Release(T* Object)
    {
        if (!Object)
        {
            return;
        }
        
        int32 Index = ActiveObjects.Find(Object);
        if (Index != INDEX_NONE)
        {
            ActiveObjects.RemoveAtSwap(Index);
            OnRelease(Object);
            Pool.Add(Object);
        }
    }
    
    // 清空池
    void Empty()
    {
        for (T* Object : Pool)
        {
            DestroyObject(Object);
        }
        Pool.Empty();
        
        for (T* Object : ActiveObjects)
        {
            DestroyObject(Object);
        }
        ActiveObjects.Empty();
    }
    
    int32 GetPoolSize() const { return Pool.Num(); }
    int32 GetActiveCount() const { return ActiveObjects.Num(); }
    
protected:
    virtual T* CreateNewObject() = 0;
    virtual void DestroyObject(T* Object) = 0;
    virtual void OnAcquire(T* Object) {}
    virtual void OnRelease(T* Object) {}
    
private:
    TArray<T*> Pool;
    TArray<T*> ActiveObjects;
    int32 MaxPoolSize;
};
```

#### Actor对象池实现

```cpp
// Actor对象池
UCLASS()
class UActorPool : public UObject
{
    GENERATED_BODY()
    
public:
    // 初始化对象池
    void Initialize(UWorld* InWorld, TSubclassOf<AActor> ActorClass, int32 InitialSize)
    {
        World = InWorld;
        PooledActorClass = ActorClass;
        
        // 预创建Actor
        for (int32 i = 0; i < InitialSize; ++i)
        {
            AActor* Actor = CreateNewActor();
            if (Actor)
            {
                InactiveActors.Add(Actor);
            }
        }
    }
    
    // 从池中获取Actor
    AActor* AcquireActor(const FVector& Location, const FRotator& Rotation)
    {
        AActor* Actor = nullptr;
        
        if (InactiveActors.Num() > 0)
        {
            Actor = InactiveActors.Pop();
        }
        else
        {
            Actor = CreateNewActor();
        }
        
        if (Actor)
        {
            Actor->SetActorLocationAndRotation(Location, Rotation);
            Actor->SetActorHiddenInGame(false);
            Actor->SetActorEnableCollision(true);
            Actor->SetActorTickEnabled(true);
            
            ActiveActors.Add(Actor);
        }
        
        return Actor;
    }
    
    // 归还Actor到池
    void ReleaseActor(AActor* Actor)
    {
        if (!Actor)
        {
            return;
        }
        
        int32 Index = ActiveActors.Find(Actor);
        if (Index != INDEX_NONE)
        {
            ActiveActors.RemoveAtSwap(Index);
            
            // 重置Actor状态
            Actor->SetActorHiddenInGame(true);
            Actor->SetActorEnableCollision(false);
            Actor->SetActorTickEnabled(false);
            Actor->SetActorLocation(FVector(0, 0, -10000)); // 移到地图外
            
            InactiveActors.Add(Actor);
        }
    }
    
    // 清空对象池
    void ClearPool()
    {
        for (AActor* Actor : InactiveActors)
        {
            if (Actor)
            {
                Actor->Destroy();
            }
        }
        InactiveActors.Empty();
        
        for (AActor* Actor : ActiveActors)
        {
            if (Actor)
            {
                Actor->Destroy();
            }
        }
        ActiveActors.Empty();
    }
    
private:
    AActor* CreateNewActor()
    {
        if (!World || !PooledActorClass)
        {
            return nullptr;
        }
        
        FActorSpawnParameters SpawnParams;
        SpawnParams.SpawnCollisionHandlingOverride = 
            ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
        
        AActor* Actor = World->SpawnActor<AActor>(
            PooledActorClass, 
            FVector(0, 0, -10000), 
            FRotator::ZeroRotator, 
            SpawnParams
        );
        
        if (Actor)
        {
            Actor->SetActorHiddenInGame(true);
            Actor->SetActorEnableCollision(false);
            Actor->SetActorTickEnabled(false);
        }
        
        return Actor;
    }
    
    UPROPERTY()
    TArray<AActor*> InactiveActors;
    
    UPROPERTY()
    TArray<AActor*> ActiveActors;
    
    UPROPERTY()
    UWorld* World;
    
    UPROPERTY()
    TSubclassOf<AActor> PooledActorClass;
};
```

#### 粒子效果对象池示例

```cpp
// 粒子效果管理器（使用对象池）
UCLASS()
class UParticlePoolManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override
    {
        Super::Initialize(Collection);
        
        // 预加载常用粒子效果
        PreloadParticleSystem(ImpactParticleClass, 20);
        PreloadParticleSystem(MuzzleFlashClass, 10);
    }
    
    virtual void Deinitialize() override
    {
        // 清理所有对象池
        for (auto& Pair : ParticlePools)
        {
            for (UParticleSystemComponent* PSC : Pair.Value)
            {
                if (PSC)
                {
                    PSC->DestroyComponent();
                }
            }
        }
        ParticlePools.Empty();
        
        Super::Deinitialize();
    }
    
    // 播放粒子效果
    UParticleSystemComponent* PlayPooledParticle(
        UParticleSystem* Template,
        const FVector& Location,
        const FRotator& Rotation,
        float Duration = 2.0f)
    {
        if (!Template)
        {
            return nullptr;
        }
        
        UParticleSystemComponent* PSC = AcquireParticle(Template);
        if (!PSC)
        {
            return nullptr;
        }
        
        PSC->SetWorldLocationAndRotation(Location, Rotation);
        PSC->SetVisibility(true);
        PSC->Activate(true);
        
        // 自动归还到池
        FTimerHandle TimerHandle;
        GetWorld()->GetTimerManager().SetTimer(
            TimerHandle,
            [this, PSC, Template]()
            {
                ReleaseParticle(Template, PSC);
            },
            Duration,
            false
        );
        
        return PSC;
    }
    
private:
    void PreloadParticleSystem(UParticleSystem* Template, int32 Count)
    {
        if (!Template)
        {
            return;
        }
        
        TArray<UParticleSystemComponent*>& Pool = ParticlePools.FindOrAdd(Template);
        
        for (int32 i = 0; i < Count; ++i)
        {
            UParticleSystemComponent* PSC = CreateParticleComponent(Template);
            if (PSC)
            {
                Pool.Add(PSC);
            }
        }
    }
    
    UParticleSystemComponent* AcquireParticle(UParticleSystem* Template)
    {
        TArray<UParticleSystemComponent*>& Pool = ParticlePools.FindOrAdd(Template);
        
        // 查找未激活的粒子组件
        for (UParticleSystemComponent* PSC : Pool)
        {
            if (PSC && !PSC->IsActive())
            {
                return PSC;
            }
        }
        
        // 如果池中没有，创建新的
        UParticleSystemComponent* NewPSC = CreateParticleComponent(Template);
        if (NewPSC)
        {
            Pool.Add(NewPSC);
        }
        
        return NewPSC;
    }
    
    void ReleaseParticle(UParticleSystem* Template, UParticleSystemComponent* PSC)
    {
        if (!PSC)
        {
            return;
        }
        
        PSC->Deactivate();
        PSC->SetVisibility(false);
    }
    
    UParticleSystemComponent* CreateParticleComponent(UParticleSystem* Template)
    {
        UWorld* World = GetWorld();
        if (!World)
        {
            return nullptr;
        }
        
        UParticleSystemComponent* PSC = NewObject<UParticleSystemComponent>(this);
        if (PSC)
        {
            PSC->SetTemplate(Template);
            PSC->bAutoActivate = false;
            PSC->SetVisibility(false);
            PSC->RegisterComponent();
        }
        
        return PSC;
    }
    
    UPROPERTY()
    TMap<UParticleSystem*, TArray<UParticleSystemComponent*>> ParticlePools;
    
    UPROPERTY(EditDefaultsOnly, Category = "Particles")
    UParticleSystem* ImpactParticleClass;
    
    UPROPERTY(EditDefaultsOnly, Category = "Particles")
    UParticleSystem* MuzzleFlashClass;
};
```

## 内存优化

内存管理是性能优化的重要组成部分，不当的内存使用会导致内存泄漏、频繁的垃圾回收（GC）和内存碎片。

### 资源流式加载：按需加载

Lyra使用UE5的Asset Manager进行资源的异步流式加载，避免一次性加载所有资源。

#### Lyra Asset Manager实现

```cpp
// LyraAssetManager.h
UCLASS(MinimalAPI, Config = Game)
class ULyraAssetManager : public UAssetManager
{
    GENERATED_BODY()
    
public:
    // 同步加载资源（阻塞）
    static UObject* SynchronousLoadAsset(const FSoftObjectPath& AssetPath)
    {
        if (AssetPath.IsValid())
        {
            TUniquePtr<FScopeLogTime> LogTimePtr;
            
            if (ShouldLogAssetLoads())
            {
                LogTimePtr = MakeUnique<FScopeLogTime>(
                    *FString::Printf(TEXT("Synchronously loaded asset [%s]"), 
                                     *AssetPath.ToString()), 
                    nullptr, 
                    FScopeLogTime::ScopeLog_Seconds
                );
            }
            
            if (UAssetManager::IsInitialized())
            {
                // 使用Streamable Manager进行流式加载
                return UAssetManager::GetStreamableManager().LoadSynchronous(
                    AssetPath, 
                    false
                );
            }
            
            // 如果Asset Manager未初始化，使用TryLoad
            return AssetPath.TryLoad();
        }
        
        return nullptr;
    }
    
    // 异步加载资源（非阻塞）
    template<typename AssetType>
    static void AsyncLoadAsset(
        const TSoftObjectPtr<AssetType>& AssetPointer,
        TFunction<void(AssetType*)> OnLoadedCallback)
    {
        const FSoftObjectPath& AssetPath = AssetPointer.ToSoftObjectPath();
        
        if (!AssetPath.IsValid())
        {
            OnLoadedCallback(nullptr);
            return;
        }
        
        // 检查是否已加载
        AssetType* LoadedAsset = AssetPointer.Get();
        if (LoadedAsset)
        {
            OnLoadedCallback(LoadedAsset);
            return;
        }
        
        // 异步加载
        TSharedPtr<FStreamableHandle> Handle = 
            UAssetManager::GetStreamableManager().RequestAsyncLoad(
                AssetPath,
                [AssetPath, OnLoadedCallback]()
                {
                    AssetType* Asset = Cast<AssetType>(AssetPath.ResolveObject());
                    OnLoadedCallback(Asset);
                }
            );
    }
    
    // 批量异步加载
    static TSharedPtr<FStreamableHandle> AsyncLoadAssets(
        const TArray<FSoftObjectPath>& AssetPaths,
        TFunction<void()> OnLoadedCallback)
    {
        return UAssetManager::GetStreamableManager().RequestAsyncLoad(
            AssetPaths,
            OnLoadedCallback
        );
    }
};
```

#### 实战：武器资源流式加载

```cpp
// 武器资源配置
USTRUCT(BlueprintType)
struct FWeaponAssetData
{
    GENERATED_BODY()
    
    UPROPERTY(EditDefaultsOnly, Category = "Assets")
    TSoftObjectPtr<USkeletalMesh> WeaponMesh;
    
    UPROPERTY(EditDefaultsOnly, Category = "Assets")
    TSoftObjectPtr<UAnimMontage> FireAnimation;
    
    UPROPERTY(EditDefaultsOnly, Category = "Assets")
    TSoftObjectPtr<UParticleSystem> MuzzleFlash;
    
    UPROPERTY(EditDefaultsOnly, Category = "Assets")
    TSoftObjectPtr<USoundBase> FireSound;
};

// 武器类（使用流式加载）
UCLASS()
class AStreamedWeapon : public AActor
{
    GENERATED_BODY()
    
public:
    void EquipWeapon()
    {
        if (bAssetsLoaded)
        {
            OnWeaponReady();
            return;
        }
        
        // 收集需要加载的资源
        TArray<FSoftObjectPath> AssetsToLoad;
        AssetsToLoad.Add(WeaponData.WeaponMesh.ToSoftObjectPath());
        AssetsToLoad.Add(WeaponData.FireAnimation.ToSoftObjectPath());
        AssetsToLoad.Add(WeaponData.MuzzleFlash.ToSoftObjectPath());
        AssetsToLoad.Add(WeaponData.FireSound.ToSoftObjectPath());
        
        // 异步加载所有资源
        LoadHandle = ULyraAssetManager::AsyncLoadAssets(
            AssetsToLoad,
            [this]()
            {
                OnAssetsLoaded();
            }
        );
    }
    
    void UnequipWeapon()
    {
        // 取消加载（如果还在加载中）
        if (LoadHandle.IsValid() && LoadHandle->IsActive())
        {
            LoadHandle->CancelHandle();
        }
        LoadHandle.Reset();
        
        // 卸载资源
        bAssetsLoaded = false;
    }
    
private:
    void OnAssetsLoaded()
    {
        // 资源加载完成，设置组件
        if (USkeletalMeshComponent* MeshComp = GetMeshComponent())
        {
            if (USkeletalMesh* Mesh = WeaponData.WeaponMesh.Get())
            {
                MeshComp->SetSkeletalMesh(Mesh);
            }
        }
        
        bAssetsLoaded = true;
        OnWeaponReady();
    }
    
    void OnWeaponReady()
    {
        // 武器已准备好，可以使用
        BP_OnWeaponReady();
    }
    
    UFUNCTION(BlueprintImplementableEvent, Category = "Weapon")
    void BP_OnWeaponReady();
    
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    FWeaponAssetData WeaponData;
    
    TSharedPtr<FStreamableHandle> LoadHandle;
    bool bAssetsLoaded = false;
};
```

### 弱引用与资源卸载

使用`TSoftObjectPtr`和`TSoftClassPtr`替代硬引用，避免不必要的资源常驻内存。

#### 硬引用 vs 软引用

```cpp
// ❌ 错误：硬引用会导致资源始终加载
UCLASS()
class ABadWeapon : public AActor
{
    GENERATED_BODY()
    
    UPROPERTY(EditDefaultsOnly)
    UStaticMesh* WeaponMesh;  // 硬引用，游戏启动时就会加载
    
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UDamageType> DamageClass;  // 硬引用类
};

// ✅ 正确：软引用，按需加载
UCLASS()
class AGoodWeapon : public AActor
{
    GENERATED_BODY()
    
    UPROPERTY(EditDefaultsOnly)
    TSoftObjectPtr<UStaticMesh> WeaponMesh;  // 软引用
    
    UPROPERTY(EditDefaultsOnly)
    TSoftClassPtr<UDamageType> DamageClass;  // 软引用类
    
    void LoadWeaponMesh()
    {
        if (WeaponMesh.IsNull())
        {
            return;
        }
        
        // 检查是否已加载
        if (UStaticMesh* Mesh = WeaponMesh.Get())
        {
            SetMesh(Mesh);
            return;
        }
        
        // 异步加载
        ULyraAssetManager::AsyncLoadAsset<UStaticMesh>(
            WeaponMesh,
            [this](UStaticMesh* LoadedMesh)
            {
                if (LoadedMesh)
                {
                    SetMesh(LoadedMesh);
                }
            }
        );
    }
};
```

#### Primary Asset Labels：资源分组管理

```cpp
// 使用Primary Asset Label对资源进行分组
// 可以按需加载和卸载整个资源组

// 创建Data Asset（继承自UPrimaryDataAsset）
UCLASS()
class UWeaponAssetBundle : public UPrimaryDataAsset
{
    GENERATED_BODY()
    
public:
    UPROPERTY(EditDefaultsOnly, Category = "Weapons")
    TArray<TSoftObjectPtr<UStaticMesh>> WeaponMeshes;
    
    UPROPERTY(EditDefaultsOnly, Category = "Weapons")
    TArray<TSoftObjectPtr<UParticleSystem>> WeaponEffects;
    
    UPROPERTY(EditDefaultsOnly, Category = "Weapons")
    TArray<TSoftObjectPtr<USoundBase>> WeaponSounds;
    
    // 加载Bundle中的所有资源
    void LoadBundleAsync(TFunction<void()> OnComplete)
    {
        TArray<FSoftObjectPath> AssetsToLoad;
        
        for (const TSoftObjectPtr<UStaticMesh>& Mesh : WeaponMeshes)
        {
            AssetsToLoad.Add(Mesh.ToSoftObjectPath());
        }
        
        for (const TSoftObjectPtr<UParticleSystem>& Effect : WeaponEffects)
        {
            AssetsToLoad.Add(Effect.ToSoftObjectPath());
        }
        
        for (const TSoftObjectPtr<USoundBase>& Sound : WeaponSounds)
        {
            AssetsToLoad.Add(Sound.ToSoftObjectPath());
        }
        
        UAssetManager::GetStreamableManager().RequestAsyncLoad(
            AssetsToLoad,
            OnComplete
        );
    }
};
```

### 垃圾回收优化

UE的垃圾回收（GC）系统会定期清理不再使用的UObject，但GC过程会造成帧率波动。

#### 控制GC频率

```cpp
// 在GameInstance中配置GC策略
void ULyraGameInstance::Init()
{
    Super::Init();
    
    // 增加GC触发阈值，减少GC频率
    GEngine->SetGarbageCollectionInterval(60.0f);  // 每60秒一次
    
    // 或者根据内存压力动态调整
    FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
    if (MemStats.AvailablePhysical < 1024 * 1024 * 1024)  // 可用内存 < 1GB
    {
        GEngine->SetGarbageCollectionInterval(30.0f);  // 更频繁的GC
    }
}

// 在关键时刻手动触发GC（避免在gameplay中触发）
void ULyraGameInstance::OnLevelTransition()
{
    // 在关卡切换时手动GC
    GEngine->ForceGarbageCollection(true);
}
```

#### 减少GC扫描开销

```cpp
// 使用AddToRoot防止对象被GC
void UMyPersistentManager::Initialize()
{
    // 添加到Root，不会被GC
    AddToRoot();
}

void UMyPersistentManager::Shutdown()
{
    // 移除Root引用，允许GC
    RemoveFromRoot();
}

// 使用TStrongObjectPtr保持对象存活
class UMySystem
{
public:
    void LoadImportantAsset()
    {
        // TStrongObjectPtr会自动管理引用
        ImportantAsset = TStrongObjectPtr<UTexture2D>(
            LoadObject<UTexture2D>(nullptr, TEXT("/Game/Textures/Important"))
        );
    }
    
private:
    TStrongObjectPtr<UTexture2D> ImportantAsset;
};
```

#### 对象池与GC

对象池中的对象需要防止被GC：

```cpp
UCLASS()
class UPooledObjectManager : public UObject
{
    GENERATED_BODY()
    
public:
    UPooledObjectManager()
    {
        // 管理器本身不会被GC
        AddToRoot();
    }
    
    ~UPooledObjectManager()
    {
        RemoveFromRoot();
    }
    
    void Initialize(int32 PoolSize)
    {
        Pool.Reserve(PoolSize);
        
        for (int32 i = 0; i < PoolSize; ++i)
        {
            UMyPooledObject* Object = NewObject<UMyPooledObject>(this);
            // 对象是该Manager的子对象，不会被单独GC
            Pool.Add(Object);
        }
    }
    
private:
    UPROPERTY()  // 必须标记为UPROPERTY才能被GC系统正确追踪
    TArray<UMyPooledObject*> Pool;
};
```

## GPU性能优化

GPU负责游戏的渲染工作，优化GPU性能可以提升画面流畅度和视觉质量。

### Draw Call优化

Draw Call是CPU向GPU发送绘制指令的过程，过多的Draw Call会造成CPU瓶颈。

#### 静态网格体批处理

```cpp
// 使用Instanced Static Mesh减少Draw Call
UCLASS()
class AOptimizedForest : public AActor
{
    GENERATED_BODY()
    
public:
    AOptimizedForest()
    {
        // ❌ 错误：为每棵树创建单独的StaticMeshComponent
        // 会产生大量Draw Call
        /*
        for (int32 i = 0; i < 1000; ++i)
        {
            UStaticMeshComponent* Tree = CreateDefaultSubobject<UStaticMeshComponent>(
                *FString::Printf(TEXT("Tree_%d"), i)
            );
        }
        */
        
        // ✅ 正确：使用Instanced Static Mesh
        InstancedMeshComponent = CreateDefaultSubobject<UInstancedStaticMeshComponent>(
            TEXT("InstancedTrees")
        );
        RootComponent = InstancedMeshComponent;
    }
    
    void SpawnTrees(int32 Count, float Radius)
    {
        if (!InstancedMeshComponent || !TreeMesh)
        {
            return;
        }
        
        InstancedMeshComponent->SetStaticMesh(TreeMesh);
        
        // 批量添加实例
        for (int32 i = 0; i < Count; ++i)
        {
            FVector Location = FVector(
                FMath::RandRange(-Radius, Radius),
                FMath::RandRange(-Radius, Radius),
                0.0f
            );
            
            FRotator Rotation = FRotator(
                0.0f,
                FMath::RandRange(0.0f, 360.0f),
                0.0f
            );
            
            FVector Scale = FVector(
                FMath::RandRange(0.8f, 1.2f)
            );
            
            FTransform InstanceTransform(Rotation, Location, Scale);
            InstancedMeshComponent->AddInstance(InstanceTransform);
        }
        
        // 1000棵树只产生1个Draw Call！
    }
    
private:
    UPROPERTY(VisibleAnywhere, Category = "Rendering")
    UInstancedStaticMeshComponent* InstancedMeshComponent;
    
    UPROPERTY(EditDefaultsOnly, Category = "Assets")
    UStaticMesh* TreeMesh;
};
```

#### 使用Hierarchical Instanced Static Mesh（HISM）

HISM支持自动LOD剔除，进一步优化性能：

```cpp
UCLASS()
class AOptimizedGrass : public AActor
{
    GENERATED_BODY()
    
public:
    AOptimizedGrass()
    {
        // 使用HISM，支持距离剔除和LOD
        HISMComponent = CreateDefaultSubobject<UHierarchicalInstancedStaticMeshComponent>(
            TEXT("HISMGrass")
        );
        RootComponent = HISMComponent;
    }
    
    void SpawnGrass(const FBox& Bounds, float Density)
    {
        if (!HISMComponent || !GrassMesh)
        {
            return;
        }
        
        HISMComponent->SetStaticMesh(GrassMesh);
        
        // 配置剔除参数
        HISMComponent->InstanceStartCullDistance = 1000.0f;  // 开始剔除距离
        HISMComponent->InstanceEndCullDistance = 2000.0f;    // 完全剔除距离
        
        // 生成草地实例
        FVector BoxSize = Bounds.GetSize();
        int32 NumInstances = FMath::RoundToInt(BoxSize.X * BoxSize.Y * Density);
        
        for (int32 i = 0; i < NumInstances; ++i)
        {
            FVector Location = FVector(
                FMath::RandRange(Bounds.Min.X, Bounds.Max.X),
                FMath::RandRange(Bounds.Min.Y, Bounds.Max.Y),
                Bounds.Min.Z
            );
            
            FTransform InstanceTransform(
                FRotator(0, FMath::RandRange(0.0f, 360.0f), 0),
                Location,
                FVector(FMath::RandRange(0.9f, 1.1f))
            );
            
            HISMComponent->AddInstance(InstanceTransform);
        }
        
        // HISM会自动构建层级结构，支持高效剔除
        HISMComponent->BuildTreeIfOutdated(true, true);
    }
    
private:
    UPROPERTY(VisibleAnywhere, Category = "Rendering")
    UHierarchicalInstancedStaticMeshComponent* HISMComponent;
    
    UPROPERTY(EditDefaultsOnly, Category = "Assets")
    UStaticMesh* GrassMesh;
};
```

### 材质复杂度优化

复杂的材质会增加GPU负担，特别是在高分辨率和高帧率下。

#### 材质性能分析

```cpp
// 控制台命令查看材质复杂度
// viewmode shadercomplexity  // 着色器复杂度视图（红色表示很复杂）
// viewmode lit               // 返回正常光照视图

// 材质统计
// stat shaders               // 着色器统计信息
```

#### 优化材质指令数

```cpp
// ❌ 低效材质设置
// Material Editor中的常见问题：
// 1. 过多的纹理采样（超过16个）
// 2. 复杂的数学运算（大量的三角函数、指数运算）
// 3. 动态分支（if语句）
// 4. 过度使用Custom节点

// ✅ 高效材质设置
// 1. 合并纹理通道（RGB通道分别存储不同贴图）
// 2. 使用Material Functions复用逻辑
// 3. 使用静态开关（Static Switch）而非动态分支
// 4. 简化法线计算
```

#### Material Quality Switch：根据平台调整材质

```cpp
// 在材质编辑器中使用Quality Switch节点
// 可以根据平台（PC/Console/Mobile）使用不同的材质逻辑

// 代码中动态设置材质质量
void UGraphicsSettingsManager::SetMaterialQualityLevel(int32 QualityLevel)
{
    // 0 = Low, 1 = Medium, 2 = High, 3 = Epic
    Scalability::SetQualityLevels(
        Scalability::FQualityLevels(),
        true  // bForceUpdate
    );
    
    // 设置材质质量开关
    static const auto CVarMaterialQualityLevel = IConsoleManager::Get().FindConsoleVariable(
        TEXT("r.MaterialQualityLevel")
    );
    
    if (CVarMaterialQualityLevel)
    {
        CVarMaterialQualityLevel->Set(QualityLevel);
    }
}
```

### LOD（Level of Detail）系统

LOD系统根据距离自动切换模型的细节等级，是优化渲染性能的关键技术。

#### 静态网格LOD配置

```cpp
// 在C++中配置LOD设置
void ConfigureStaticMeshLOD(UStaticMesh* Mesh)
{
    if (!Mesh)
    {
        return;
    }
    
    // 设置LOD距离
    const int32 NumLODs = Mesh->GetNumLODs();
    for (int32 LODIndex = 0; LODIndex < NumLODs; ++LODIndex)
    {
        FStaticMeshSourceModel& SourceModel = Mesh->GetSourceModel(LODIndex);
        
        // 设置屏幕尺寸阈值
        switch (LODIndex)
        {
        case 0:  // LOD0: 高细节（距离近）
            SourceModel.ScreenSize = FMath::GetMappedRangeValueClamped(
                FVector2D(0, 1), FVector2D(1.0f, 0.5f), 1.0f
            );
            break;
        case 1:  // LOD1: 中细节
            SourceModel.ScreenSize = 0.5f;
            break;
        case 2:  // LOD2: 低细节
            SourceModel.ScreenSize = 0.25f;
            break;
        case 3:  // LOD3: 最低细节（距离远）
            SourceModel.ScreenSize = 0.1f;
            break;
        }
    }
    
    Mesh->PostEditChange();
}
```

#### 骨骼网格LOD与动画优化

```cpp
// 配置骨骼网格LOD
UCLASS()
class AOptimizedCharacter : public ACharacter
{
    GENERATED_BODY()
    
public:
    AOptimizedCharacter()
    {
        // 配置LOD设置
        if (USkeletalMeshComponent* MeshComp = GetMesh())
        {
            // 强制使用LOD（用于测试）
            // MeshComp->SetForcedLOD(2);
            
            // 最大LOD等级
            MeshComp->SetMinLOD(0);
            
            // 根据距离自动选择LOD
            MeshComp->bOverrideMinLOD = false;
        }
    }
    
    virtual void Tick(float DeltaTime) override
    {
        Super::Tick(DeltaTime);
        
        // 根据到玩家的距离动态调整更新频率
        OptimizeAnimationUpdateRate();
    }
    
private:
    void OptimizeAnimationUpdateRate()
    {
        USkeletalMeshComponent* MeshComp = GetMesh();
        if (!MeshComp)
        {
            return;
        }
        
        APlayerController* PC = GetWorld()->GetFirstPlayerController();
        if (!PC || !PC->GetPawn())
        {
            return;
        }
        
        float Distance = FVector::Dist(
            GetActorLocation(), 
            PC->GetPawn()->GetActorLocation()
        );
        
        // 根据距离调整动画更新频率
        if (Distance > 5000.0f)
        {
            // 远距离：每10帧更新一次
            MeshComp->VisibilityBasedAnimTickOption = 
                EVisibilityBasedAnimTickOption::OnlyTickPoseWhenRendered;
            MeshComp->SetComponentTickInterval(0.1f);
        }
        else if (Distance > 2000.0f)
        {
            // 中距离：每5帧更新一次
            MeshComp->SetComponentTickInterval(0.05f);
        }
        else
        {
            // 近距离：每帧更新
            MeshComp->SetComponentTickInterval(0.0f);
        }
    }
};
```

#### Hierarchical LOD（HLOD）：大场景优化

```cpp
// HLOD配置（通常在编辑器中配置，但可以通过代码查询）
void AnalyzeHLODSetup(UWorld* World)
{
    if (!World)
    {
        return;
    }
    
    // 获取HLOD层级设置
    for (int32 LODIndex = 0; LODIndex < World->GetWorldSettings()->GetNumHierarchicalLODLevels(); ++LODIndex)
    {
        FHierarchicalSimplification& LODSetup = 
            World->GetWorldSettings()->GetHierarchicalLODSetup()[LODIndex];
        
        UE_LOG(LogTemp, Log, TEXT("HLOD Level %d:"), LODIndex);
        UE_LOG(LogTemp, Log, TEXT("  Transition Screen Size: %f"), 
               LODSetup.TransitionScreenSize);
        UE_LOG(LogTemp, Log, TEXT("  Desired Bound Radius: %f"), 
               LODSetup.DesiredBoundRadius);
    }
}
```

## 网络性能优化

多人游戏的网络性能至关重要，Lyra在网络优化方面有大量值得学习的实践。

### Replication Graph：智能网络复制

Lyra使用Replication Graph替代传统的相关性系统，大幅提升了多人游戏的网络性能。

#### Lyra Replication Graph架构

```cpp
// LyraReplicationGraph.h
UCLASS(transient, config=Engine)
class ULyraReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()
    
public:
    virtual void InitGlobalActorClassSettings() override;
    virtual void InitGlobalGraphNodes() override;
    virtual void RouteAddNetworkActorToNodes(
        const FNewReplicatedActorInfo& ActorInfo, 
        FGlobalActorReplicationInfo& GlobalInfo) override;
    
    // 空间分区节点（用于玩家和动态Actor）
    UPROPERTY()
    UReplicationGraphNode_GridSpatialization2D* GridNode;
    
    // 始终相关的Actor节点
    UPROPERTY()
    UReplicationGraphNode_ActorList* AlwaysRelevantNode;
    
    // 始终相关的类列表
    UPROPERTY()
    TArray<UClass*> AlwaysRelevantClasses;
    
    // 流式关卡中的始终相关Actor
    TMap<FName, FActorRepListRefView> AlwaysRelevantStreamingLevelActors;
    
private:
    void AddClassRepInfo(UClass* Class, EClassRepNodeMapping Mapping);
    EClassRepNodeMapping GetMappingPolicy(UClass* Class);
};
```

#### 配置Actor的网络复制策略

```cpp
// LyraReplicationGraph.cpp
void ULyraReplicationGraph::InitGlobalActorClassSettings()
{
    Super::InitGlobalActorClassSettings();
    
    // 配置各种Actor类的复制策略
    
    // 始终相关的类（如GameState）
    AlwaysRelevantClasses.Add(AGameStateBase::StaticClass());
    AlwaysRelevantClasses.Add(APlayerState::StaticClass());
    
    // 空间化的类（根据距离复制）
    AddClassRepInfo(ACharacter::StaticClass(), 
                    EClassRepNodeMapping::Spatialize_Dynamic);
    AddClassRepInfo(APawn::StaticClass(), 
                    EClassRepNodeMapping::Spatialize_Dynamic);
    AddClassRepInfo(AProjectile::StaticClass(), 
                    EClassRepNodeMapping::Spatialize_Dynamic);
    
    // 静态空间化（不经常移动的Actor）
    AddClassRepInfo(APickup::StaticClass(), 
                    EClassRepNodeMapping::Spatialize_Static);
    
    // 休眠的Actor（大部分时间不活动）
    AddClassRepInfo(ATrap::StaticClass(), 
                    EClassRepNodeMapping::Spatialize_Dormancy);
}

void ULyraReplicationGraph::AddClassRepInfo(
    UClass* Class, 
    EClassRepNodeMapping Mapping)
{
    ClassRepNodePolicies.Set(Class, Mapping);
    
    FClassReplicationInfo ClassInfo;
    InitClassReplicationInfo(ClassInfo, Class, IsSpatialized(Mapping));
    GlobalActorReplicationInfoMap.SetClassInfo(Class, ClassInfo);
}
```

#### 频率限制：PlayerState优化

```cpp
// LyraReplicationGraphNode_PlayerStateFrequencyLimiter
// 限制PlayerState的复制频率，避免在大量玩家时造成网络拥塞

UCLASS()
class ULyraReplicationGraphNode_PlayerStateFrequencyLimiter : public UReplicationGraphNode
{
    GENERATED_BODY()
    
public:
    ULyraReplicationGraphNode_PlayerStateFrequencyLimiter()
    {
        // 每帧只复制2个PlayerState（轮流复制）
        TargetActorsPerFrame = 2;
    }
    
    virtual void GatherActorListsForConnection(
        const FConnectionGatherActorListParameters& Params) override
    {
        // 实现轮流复制逻辑
        // 在100个玩家的游戏中，每个PlayerState大约每50帧才复制一次
        // 这样可以大幅减少网络带宽，而PlayerState的更新频率要求本来就不高
    }
    
    virtual void PrepareForReplication() override
    {
        // 每帧准备要复制的PlayerState列表
    }
    
private:
    int32 TargetActorsPerFrame;
};
```

### 带宽控制与优化

#### 条件复制：只在需要时复制

```cpp
// 使用条件复制宏，精确控制属性何时复制
UCLASS()
class AOptimizedWeapon : public AActor
{
    GENERATED_BODY()
    
public:
    AOptimizedWeapon()
    {
        bReplicates = true;
        
        // 设置网络更新频率
        NetUpdateFrequency = 10.0f;  // 每秒最多10次
        MinNetUpdateFrequency = 2.0f;  // 每秒最少2次
        
        // 网络优先级（越高越优先复制）
        NetPriority = 2.0f;
        
        // 设置为始终相关（小心使用）
        bAlwaysRelevant = false;
    }
    
    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        
        // 基础属性：始终复制
        DOREPLIFETIME(AOptimizedWeapon, AmmoCount);
        
        // 条件复制：只给Owner复制
        DOREPLIFETIME_CONDITION(AOptimizedWeapon, CurrentSpread, COND_OwnerOnly);
        
        // 条件复制：只在初始化时复制一次
        DOREPLIFETIME_CONDITION(AOptimizedWeapon, WeaponID, COND_InitialOnly);
        
        // 条件复制：跳过Owner（其他人都能看到）
        DOREPLIFETIME_CONDITION(AOptimizedWeapon, VisibleState, COND_SkipOwner);
        
        // 条件复制：只给自主代理复制
        DOREPLIFETIME_CONDITION(AOptimizedWeapon, InputBuffer, COND_AutonomousOnly);
        
        // 条件复制：只给模拟代理复制
        DOREPLIFETIME_CONDITION(AOptimizedWeapon, ReplicatedMovement, COND_SimulatedOnly);
    }
    
private:
    UPROPERTY(Replicated)
    int32 AmmoCount;
    
    UPROPERTY(Replicated)
    float CurrentSpread;
    
    UPROPERTY(Replicated)
    int32 WeaponID;
    
    UPROPERTY(Replicated)
    uint8 VisibleState;
    
    UPROPERTY(Replicated)
    FVector InputBuffer;
    
    UPROPERTY(Replicated)
    FRepMovement ReplicatedMovement;
};
```

#### 使用RPC的最佳实践

```cpp
UCLASS()
class ANetworkOptimizedCharacter : public ACharacter
{
    GENERATED_BODY()
    
public:
    // ❌ 低效：每次攻击都发送RPC
    UFUNCTION(Server, Unreliable, WithValidation)
    void Server_FireWeapon_Bad(FVector Location, FRotator Rotation);
    
    // ✅ 高效：批量发送多次攻击
    UFUNCTION(Server, Unreliable, WithValidation)
    void Server_FireWeaponBatch(const TArray<FFireData>& FireDataArray);
    
    // ✅ 高效：使用Unreliable RPC减少开销（不重要的事件）
    UFUNCTION(NetMulticast, Unreliable)
    void Multicast_PlayCosmeticEffect(FVector Location);
    
    // ✅ 高效：使用Reliable RPC确保送达（重要事件）
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_PickupItem(AItemActor* Item);
    
    // 实现验证函数
    bool Server_FireWeaponBatch_Validate(const TArray<FFireData>& FireDataArray)
    {
        // 防作弊检查
        if (FireDataArray.Num() > 10)
        {
            return false;  // 拒绝明显的作弊行为
        }
        
        return true;
    }
    
    void Server_FireWeaponBatch_Implementation(const TArray<FFireData>& FireDataArray)
    {
        // 处理批量射击数据
        for (const FFireData& Data : FireDataArray)
        {
            ProcessFire(Data);
        }
    }
};

USTRUCT()
struct FFireData
{
    GENERATED_BODY()
    
    UPROPERTY()
    FVector_NetQuantize Location;  // 使用量化版本减少带宽
    
    UPROPERTY()
    FRotator Rotation;
    
    UPROPERTY()
    uint8 WeaponID;
};
```

### 数据压缩与量化

UE提供了多种网络量化类型，可以大幅减少带宽消耗。

#### 使用量化类型

```cpp
USTRUCT()
struct FNetworkOptimizedTransform
{
    GENERATED_BODY()
    
    // ❌ 标准FVector：12字节（3个float）
    // UPROPERTY()
    // FVector Location;
    
    // ✅ FVector_NetQuantize：6字节（压缩50%）
    // 精度：1单位（厘米级）
    UPROPERTY()
    FVector_NetQuantize Location;
    
    // ✅ FVector_NetQuantize10：4字节（压缩66%）
    // 精度：10单位（分米级）
    // UPROPERTY()
    // FVector_NetQuantize10 Location;
    
    // ✅ FVector_NetQuantize100：4字节（压缩66%）
    // 精度：100单位（米级）
    // UPROPERTY()
    // FVector_NetQuantize100 Location;
    
    // 旋转量化
    UPROPERTY()
    FRotator Rotation;  // 自动量化到65536个离散值
};
```

#### 自定义序列化

对于复杂数据结构，可以实现自定义序列化来优化带宽：

```cpp
USTRUCT()
struct FCompressedPlayerInput
{
    GENERATED_BODY()
    
    // 将多个布尔值打包到一个字节
    UPROPERTY()
    uint8 PackedInputs;
    
    // 位标志
    static constexpr uint8 FLAG_FORWARD  = 1 << 0;
    static constexpr uint8 FLAG_BACKWARD = 1 << 1;
    static constexpr uint8 FLAG_LEFT     = 1 << 2;
    static constexpr uint8 FLAG_RIGHT    = 1 << 3;
    static constexpr uint8 FLAG_JUMP     = 1 << 4;
    static constexpr uint8 FLAG_CROUCH   = 1 << 5;
    static constexpr uint8 FLAG_SPRINT   = 1 << 6;
    static constexpr uint8 FLAG_FIRE     = 1 << 7;
    
    void SetForward(bool bValue)
    {
        if (bValue)
            PackedInputs |= FLAG_FORWARD;
        else
            PackedInputs &= ~FLAG_FORWARD;
    }
    
    bool GetForward() const
    {
        return (PackedInputs & FLAG_FORWARD) != 0;
    }
    
    // 实现其他输入的getter/setter...
    
    // 自定义序列化（可选，用于更精细的控制）
    bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
    {
        Ar << PackedInputs;
        bOutSuccess = true;
        return true;
    }
};

template<>
struct TStructOpsTypeTraits<FCompressedPlayerInput> : public TStructOpsTypeTraitsBase2<FCompressedPlayerInput>
{
    enum
    {
        WithNetSerializer = true,
    };
};
```

## Lyra中的性能优化案例分析

### 案例1：性能统计子系统

Lyra实现了一个完整的性能监控子系统，实时收集各项性能指标。

#### 采样统计缓存

```cpp
// FSampledStatCache：环形缓冲区存储历史采样
class FSampledStatCache
{
public:
    FSampledStatCache(const int32 InSampleSize = 125)
        : SampleSize(InSampleSize)
    {
        check(InSampleSize > 0);
        
        Samples.Empty();
        Samples.AddZeroed(SampleSize);
    }
    
    // 记录一个采样点
    void RecordSample(const double Sample)
    {
        // 环形缓冲区
        Samples[CurrentSampleIndex] = Sample;
        
        CurrentSampleIndex++;
        if (CurrentSampleIndex >= Samples.Num())
        {
            CurrentSampleIndex = 0u;
        }
    }
    
    // 计算平均值
    inline double GetAverage() const
    {
        double Sum = 0.0;
        for (const double& Sample : Samples)
        {
            Sum += Sample;
        }
        return Sum / static_cast<double>(SampleSize);
    }
    
    // 获取最小值
    inline double GetMin() const
    {
        return *Algo::MinElement(Samples);
    }
    
    // 获取最大值
    inline double GetMax() const
    {
        return *Algo::MaxElement(Samples);
    }
    
    // 遍历所有采样
    void ForEachCurrentSample(const TFunctionRef<void(const double Stat)> Func) const
    {
        int32 Index = CurrentSampleIndex;
        
        for (int32 i = 0; i < SampleSize; i++)
        {
            Func(Samples[Index]);
            
            Index++;
            if (Index == SampleSize)
            {
                Index = 0;
            }
        }
    }
    
private:
    const int32 SampleSize;
    int32 CurrentSampleIndex = 0;
    TArray<double> Samples;
};
```

#### 性能数据采集

```cpp
// FLyraPerformanceStatCache：实现IPerformanceDataConsumer接口
struct FLyraPerformanceStatCache : public IPerformanceDataConsumer
{
public:
    virtual void ProcessFrame(const FFrameData& FrameData) override
    {
        // 记录帧时间统计
        RecordStat(
            ELyraDisplayablePerformanceStat::ClientFPS,
            (FrameData.TrueDeltaSeconds != 0.0) ? 
            1.0 / FrameData.TrueDeltaSeconds : 0.0
        );
        
        RecordStat(ELyraDisplayablePerformanceStat::FrameTime, 
                   FrameData.TrueDeltaSeconds);
        RecordStat(ELyraDisplayablePerformanceStat::FrameTime_GameThread, 
                   FrameData.GameThreadTimeSeconds);
        RecordStat(ELyraDisplayablePerformanceStat::FrameTime_RenderThread, 
                   FrameData.RenderThreadTimeSeconds);
        RecordStat(ELyraDisplayablePerformanceStat::FrameTime_GPU, 
                   FrameData.GPUTimeSeconds);
        
        // 记录网络统计
        if (UWorld* World = MySubsystem->GetGameInstance()->GetWorld())
        {
            if (APlayerController* PC = GEngine->GetFirstLocalPlayerController(World))
            {
                if (APlayerState* PS = PC->GetPlayerState<APlayerState>())
                {
                    RecordStat(ELyraDisplayablePerformanceStat::Ping, 
                               PS->GetPingInMilliseconds());
                }
                
                if (UNetConnection* NetConnection = PC->GetNetConnection())
                {
                    // 丢包率
                    const UNetConnection::FNetConnectionPacketLoss& InLoss = 
                        NetConnection->GetInLossPercentage();
                    RecordStat(ELyraDisplayablePerformanceStat::PacketLoss_Incoming, 
                               InLoss.GetAvgLossPercentage());
                    
                    // 包速率
                    RecordStat(ELyraDisplayablePerformanceStat::PacketRate_Incoming, 
                               NetConnection->InPacketsPerSecond);
                    RecordStat(ELyraDisplayablePerformanceStat::PacketRate_Outgoing, 
                               NetConnection->OutPacketsPerSecond);
                }
            }
        }
    }
    
private:
    void RecordStat(const ELyraDisplayablePerformanceStat Stat, const double Value)
    {
        PerfStateCache.FindOrAdd(Stat).RecordSample(Value);
    }
    
    TMap<ELyraDisplayablePerformanceStat, FSampledStatCache> PerfStateCache;
    ULyraPerformanceStatSubsystem* MySubsystem;
};
```

### 案例2：Indicator System优化

Lyra的指示器系统（显示玩家头顶标记等）通过多种技术优化性能。

#### 对象池化Widget

```cpp
// SActorCanvas使用对象池管理Indicator Widget
void SActorCanvas::AddIndicatorForEntry(UIndicatorDescriptor* Indicator)
{
    TSoftClassPtr<UUserWidget> IndicatorClass = Indicator->GetIndicatorClass();
    if (!IndicatorClass.IsNull())
    {
        TWeakObjectPtr<UIndicatorDescriptor> IndicatorPtr(Indicator);
        
        // 异步加载Widget类
        AsyncLoad(IndicatorClass, [this, IndicatorPtr, IndicatorClass]() 
        {
            if (UIndicatorDescriptor* Indicator = IndicatorPtr.Get())
            {
                // 从对象池获取Widget实例（避免每次创建）
                if (UUserWidget* IndicatorWidget = 
                    IndicatorPool.GetOrCreateInstance(
                        TSubclassOf<UUserWidget>(IndicatorClass.Get())
                    ))
                {
                    // 绑定Indicator数据
                    if (IndicatorWidget->GetClass()->ImplementsInterface(
                        UIndicatorWidgetInterface::StaticClass()))
                    {
                        IIndicatorWidgetInterface::Execute_BindIndicator(
                            IndicatorWidget, Indicator
                        );
                    }
                    
                    Indicator->IndicatorWidget = IndicatorWidget;
                    AddActorSlot(Indicator)[IndicatorWidget->TakeWidget()];
                }
            }
        });
    }
}

void SActorCanvas::RemoveIndicatorForEntry(UIndicatorDescriptor* Indicator)
{
    if (UUserWidget* IndicatorWidget = Indicator->IndicatorWidget.Get())
    {
        // 解绑并归还到池
        IIndicatorWidgetInterface::Execute_UnbindIndicator(
            IndicatorWidget, Indicator
        );
        
        Indicator->IndicatorWidget = nullptr;
        IndicatorPool.Release(IndicatorWidget);  // 归还到池，不销毁
    }
}
```

#### 性能标记与优化

```cpp
void SActorCanvas::UpdateCanvas(double InCurrentTime, float InDeltaTime)
{
    // 添加性能追踪标记
    QUICK_SCOPE_CYCLE_COUNTER(STAT_SActorCanvas_UpdateCanvas);
    
    // 只在有绘制几何信息时更新
    if (!OptionalPaintGeometry.IsSet())
    {
        return EActiveTimerReturnType::Continue;
    }
    
    // 获取Indicator管理组件
    ULyraIndicatorManagerComponent* IndicatorComponent = IndicatorComponentPtr.Get();
    if (!IndicatorComponent)
    {
        IndicatorComponent = ULyraIndicatorManagerComponent::GetComponent(
            LocalPlayerContext.GetPlayerController()
        );
        
        if (IndicatorComponent)
        {
            IndicatorComponentPtr = IndicatorComponent;
            
            // 订阅事件（事件驱动，而非每帧轮询）
            IndicatorComponent->OnIndicatorAdded.AddSP(
                this, &SActorCanvas::OnIndicatorAdded
            );
            IndicatorComponent->OnIndicatorRemoved.AddSP(
                this, &SActorCanvas::OnIndicatorRemoved
            );
        }
    }
    
    // 批量处理所有Indicator（减少遍历次数）
    bool IndicatorsChanged = false;
    for (int32 ChildIndex = 0; ChildIndex < CanvasChildren.Num(); ++ChildIndex)
    {
        SActorCanvas::FSlot& CurChild = CanvasChildren[ChildIndex];
        UIndicatorDescriptor* Indicator = CurChild.Indicator;
        
        // 可以自动移除的Indicator（避免内存泄漏）
        if (Indicator->CanAutomaticallyRemove())
        {
            IndicatorsChanged = true;
            RemoveIndicatorForEntry(Indicator);
            --ChildIndex;  // 调整索引
            continue;
        }
        
        // 跳过不可见的Indicator（节省计算）
        if (!Indicator->GetIsVisible())
        {
            continue;
        }
        
        // 投影计算...
    }
    
    // 只在必要时重绘
    if (IndicatorsChanged)
    {
        Invalidate(EInvalidateWidget::Paint);
    }
}
```

## GAS性能优化技巧

Gameplay Ability System是Lyra的核心系统，优化GAS性能对整体性能有重要影响。

### 减少Ability激活检查开销

```cpp
// 使用Tag关系映射，减少激活条件检查
void ULyraAbilitySystemComponent::GetAdditionalActivationTagRequirements(
    const FGameplayTagContainer& AbilityTags,
    FGameplayTagContainer& OutActivationRequired,
    FGameplayTagContainer& OutActivationBlocked) const
{
    if (TagRelationshipMapping)
    {
        // 使用预配置的Tag关系，避免运行时计算
        TagRelationshipMapping->GetRequiredAndBlockedActivationTags(
            AbilityTags,
            &OutActivationRequired,
            &OutActivationBlocked
        );
    }
}
```

### 优化GameplayEffect应用

```cpp
// 批量应用GameplayEffect
void ApplyEffectsBatch(UAbilitySystemComponent* ASC, 
                       const TArray<TSubclassOf<UGameplayEffect>>& Effects,
                       float Level)
{
    if (!ASC)
    {
        return;
    }
    
    // 暂时禁用回调，批量处理完后再触发
    FScopedAbilityListLock ScopedLock(*ASC);
    
    for (const TSubclassOf<UGameplayEffect>& EffectClass : Effects)
    {
        if (EffectClass)
        {
            FGameplayEffectContextHandle ContextHandle = ASC->MakeEffectContext();
            FGameplayEffectSpecHandle SpecHandle = 
                ASC->MakeOutgoingSpec(EffectClass, Level, ContextHandle);
            
            if (SpecHandle.IsValid())
            {
                ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
            }
        }
    }
    
    // ScopedLock析构时会触发所有待处理的回调
}
```

### 缓存Attribute访问

```cpp
// 缓存频繁访问的Attribute
UCLASS()
class UOptimizedAttributeSet : public UAttributeSet
{
    GENERATED_BODY()
    
public:
    // 使用宏定义简化Attribute定义
    UPROPERTY(BlueprintReadOnly, Category = "Health", ReplicatedUsing=OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UOptimizedAttributeSet, Health)
    
    // 缓存Attribute的FGameplayAttribute（避免每次查找）
    static const FGameplayAttribute& GetHealthAttribute()
    {
        static FGameplayAttribute Attribute = 
            FindFieldChecked<FProperty>(
                UOptimizedAttributeSet::StaticClass(), 
                GET_MEMBER_NAME_CHECKED(UOptimizedAttributeSet, Health)
            );
        return Attribute;
    }
    
    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldHealth)
    {
        GAMEPLAYATTRIBUTE_REPNOTIFY(UOptimizedAttributeSet, Health, OldHealth);
    }
    
    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        
        DOREPLIFETIME_CONDITION_NOTIFY(
            UOptimizedAttributeSet, Health, 
            COND_None, REPNOTIFY_Always
        );
    }
};
```

## 实战案例1：优化大型多人游戏场景（100+玩家）

### 场景描述

在一个100人的大逃杀游戏中，所有玩家同时在一个大地图上活动。如何保证服务器和客户端都能流畅运行？

### 优化方案

#### 1. 服务器优化

```cpp
// 大型多人游戏模式
UCLASS()
class ALargescaleGameMode : public ALyraGameMode
{
    GENERATED_BODY()
    
public:
    ALargescaleGameMode()
    {
        // 启用Replication Graph
        // 在DefaultEngine.ini中配置：
        // [/Script/OnlineSubsystemUtils.IpNetDriver]
        // ReplicationDriverClassName="/Script/LyraGame.LyraReplicationGraph"
    }
    
    virtual void InitGame(const FString& MapName, 
                          const FString& Options, 
                          FString& ErrorMessage) override
    {
        Super::InitGame(MapName, Options, ErrorMessage);
        
        // 配置服务器性能参数
        ConfigureServerPerformance();
    }
    
private:
    void ConfigureServerPerformance()
    {
        // 限制每个连接的最大更新频率
        if (UNetDriver* NetDriver = GetWorld()->GetNetDriver())
        {
            // 降低全局更新频率（默认60，改为30）
            NetDriver->NetServerMaxTickRate = 30;
            
            // 每个客户端的目标带宽（字节/秒）
            NetDriver->MaxClientRate = 25000;        // 200 Kbps
            NetDriver->MaxInternetClientRate = 10000; // 80 Kbps
        }
        
        // 配置Actor的更新频率
        for (TActorIterator<APawn> It(GetWorld()); It; ++It)
        {
            APawn* Pawn = *It;
            
            // 降低Pawn的网络更新频率
            Pawn->NetUpdateFrequency = 10.0f;     // 每秒10次
            Pawn->MinNetUpdateFrequency = 2.0f;   // 最少每秒2次
            
            // 根据距离动态调整优先级
            Pawn->SetNetPriority(1.0f);
        }
    }
};
```

#### 2. 客户端优化

```cpp
// 大型场景客户端优化管理器
UCLASS()
class ULargescaleClientOptimizer : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override
    {
        Super::Initialize(Collection);
        
        // 启动优化Ticker
        GetWorld()->GetTimerManager().SetTimer(
            OptimizationTimer,
            this,
            &ThisClass::OptimizeVisibleActors,
            1.0f,  // 每秒执行一次
            true
        );
    }
    
    virtual void Deinitialize() override
    {
        GetWorld()->GetTimerManager().ClearTimer(OptimizationTimer);
        Super::Deinitialize();
    }
    
private:
    void OptimizeVisibleActors()
    {
        UWorld* World = GetWorld();
        APlayerController* PC = World->GetFirstPlayerController();
        if (!PC || !PC->GetPawn())
        {
            return;
        }
        
        FVector PlayerLocation = PC->GetPawn()->GetActorLocation();
        
        // 遍历所有角色，根据距离优化
        for (TActorIterator<ACharacter> It(World); It; ++It)
        {
            ACharacter* Character = *It;
            if (Character == PC->GetPawn())
            {
                continue;  // 跳过本地玩家
            }
            
            float Distance = FVector::Dist(PlayerLocation, Character->GetActorLocation());
            
            // 根据距离分级优化
            OptimizeCharacterByDistance(Character, Distance);
        }
    }
    
    void OptimizeCharacterByDistance(ACharacter* Character, float Distance)
    {
        USkeletalMeshComponent* Mesh = Character->GetMesh();
        if (!Mesh)
        {
            return;
        }
        
        if (Distance < 1000.0f)
        {
            // 近距离：完整渲染
            Mesh->SetComponentTickInterval(0.0f);
            Mesh->bEnableUpdateRateOptimizations = false;
            Mesh->VisibilityBasedAnimTickOption = 
                EVisibilityBasedAnimTickOption::AlwaysTickPoseAndRefreshBones;
        }
        else if (Distance < 3000.0f)
        {
            // 中距离：降低动画更新率
            Mesh->SetComponentTickInterval(0.05f);  // 每5帧更新一次
            Mesh->bEnableUpdateRateOptimizations = true;
            Mesh->VisibilityBasedAnimTickOption = 
                EVisibilityBasedAnimTickOption::OnlyTickPoseWhenRendered;
        }
        else if (Distance < 5000.0f)
        {
            // 远距离：大幅降低更新率
            Mesh->SetComponentTickInterval(0.1f);   // 每10帧更新一次
            Mesh->bEnableUpdateRateOptimizations = true;
        }
        else
        {
            // 超远距离：几乎不更新（依赖网络复制）
            Mesh->SetComponentTickInterval(0.5f);   // 每30帧更新一次
            Mesh->bEnableUpdateRateOptimizations = true;
            
            // 可以考虑完全禁用Tick
            // Mesh->SetComponentTickEnabled(false);
        }
    }
    
    FTimerHandle OptimizationTimer;
};
```

#### 3. 空间分区优化

```cpp
// 使用World Partition进行场景分区（UE5新特性）
// 在项目设置中启用World Partition

// 自定义空间查询优化器
UCLASS()
class USpatialQueryOptimizer : public UActorComponent
{
    GENERATED_BODY()
    
public:
    // 查找范围内的敌人（优化版）
    void FindNearbyEnemies(TArray<ACharacter*>& OutEnemies, float SearchRadius)
    {
        OutEnemies.Empty();
        
        AActor* Owner = GetOwner();
        if (!Owner)
        {
            return;
        }
        
        FVector OwnerLocation = Owner->GetActorLocation();
        
        // 使用碰撞查询替代遍历所有Actor
        TArray<FOverlapResult> OverlapResults;
        FCollisionShape CollisionShape = FCollisionShape::MakeSphere(SearchRadius);
        
        FCollisionQueryParams QueryParams;
        QueryParams.AddIgnoredActor(Owner);
        
        // 只查询Pawn通道
        GetWorld()->OverlapMultiByChannel(
            OverlapResults,
            OwnerLocation,
            FQuat::Identity,
            ECC_Pawn,
            CollisionShape,
            QueryParams
        );
        
        // 过滤结果
        for (const FOverlapResult& Result : OverlapResults)
        {
            if (ACharacter* Character = Cast<ACharacter>(Result.GetActor()))
            {
                // 检查是否是敌人
                if (IsEnemy(Character))
                {
                    OutEnemies.Add(Character);
                }
            }
        }
    }
    
private:
    bool IsEnemy(ACharacter* Character) const
    {
        // 实现队伍判断逻辑
        return true;
    }
};
```

## 实战案例2：移动端性能优化

### 场景描述

将PC游戏移植到移动平台（iOS/Android），需要在保证画面质量的同时，达到稳定60FPS。

### 优化方案

#### 1. 移动端渲染优化

```cpp
// 移动端画质配置管理器
UCLASS()
class UMobileGraphicsOptimizer : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override
    {
        Super::Initialize(Collection);
        
        // 应用移动端优化设置
        ApplyMobileOptimizations();
    }
    
private:
    void ApplyMobileOptimizations()
    {
        // 1. 降低阴影质量
        static IConsoleVariable* ShadowQuality = 
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.ShadowQuality"));
        if (ShadowQuality)
        {
            ShadowQuality->Set(1);  // 0=关闭, 1=低, 2=中, 3=高
        }
        
        // 2. 禁用级联阴影
        static IConsoleVariable* CSM = 
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.Shadow.CSM.MaxCascades"));
        if (CSM)
        {
            CSM->Set(1);  // 只使用1级联联
        }
        
        // 3. 降低后处理质量
        static IConsoleVariable* PostProcessQuality = 
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.PostProcessQuality"));
        if (PostProcessQuality)
        {
            PostProcessQuality->Set(0);  // 0=低, 1=中, 2=高, 3=史诗
        }
        
        // 4. 禁用抗锯齿或使用FXAA
        static IConsoleVariable* AntiAliasingMethod = 
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.AntiAliasingMethod"));
        if (AntiAliasingMethod)
        {
            AntiAliasingMethod->Set(1);  // 0=关闭, 1=FXAA, 2=TAA
        }
        
        // 5. 降低纹理质量
        static IConsoleVariable* TextureQuality = 
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.Streaming.PoolSize"));
        if (TextureQuality)
        {
            TextureQuality->Set(400);  // MB，移动端使用较小的纹理池
        }
        
        // 6. 启用移动端HDR
        static IConsoleVariable* MobileHDR = 
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.MobileHDR"));
        if (MobileHDR)
        {
            MobileHDR->Set(1);  // 启用移动端HDR
        }
        
        // 7. 优化粒子效果
        static IConsoleVariable* ParticleQuality = 
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.EmitterSpawnRateScale"));
        if (ParticleQuality)
        {
            ParticleQuality->Set(0.5f);  // 减少50%的粒子生成
        }
        
        // 8. 禁用景深
        static IConsoleVariable* DepthOfField = 
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.DepthOfFieldQuality"));
        if (DepthOfField)
        {
            DepthOfField->Set(0);  // 禁用景深
        }
        
        // 9. 降低视距
        static IConsoleVariable* ViewDistance = 
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.ViewDistanceScale"));
        if (ViewDistance)
        {
            ViewDistance->Set(0.7f);  // 视距降低到70%
        }
        
        UE_LOG(LogTemp, Log, TEXT("Mobile optimizations applied"));
    }
};
```

#### 2. 移动端材质优化

```cpp
// 移动端材质转换器
// 在材质编辑器中使用Static Switch节点，根据平台切换实现

// 代码中检测平台并应用不同的材质
void ApplyPlatformSpecificMaterial(UPrimitiveComponent* Component)
{
    if (!Component)
    {
        return;
    }
    
#if PLATFORM_ANDROID || PLATFORM_IOS
    // 移动端：使用简化材质
    if (SimplifiedMobileMaterial)
    {
        Component->SetMaterial(0, SimplifiedMobileMaterial);
    }
#else
    // PC/主机：使用完整材质
    if (FullQualityMaterial)
    {
        Component->SetMaterial(0, FullQualityMaterial);
    }
#endif
}
```

#### 3. 移动端UI优化

```cpp
// 移动端UI性能优化
UCLASS()
class UMobileUIOptimizer : public UUserWidget
{
    GENERATED_BODY()
    
protected:
    virtual void NativeConstruct() override
    {
        Super::NativeConstruct();
        
        // 优化UI渲染
        OptimizeUIRendering();
    }
    
private:
    void OptimizeUIRendering()
    {
        // 1. 禁用不必要的Tick
        SetIsEnabled(true);
        // bCanEverTick = false;  // 在构造函数中设置
        
        // 2. 使用批处理渲染
        // 确保所有UI元素使用相同的材质和纹理图集
        
        // 3. 减少Overdraw
        // 移除不可见的UI元素，使用裁剪
        
        // 4. 缓存Widget
        // 不要每帧创建销毁Widget，使用对象池或缓存
    }
    
public:
    // 优化列表视图
    UFUNCTION(BlueprintCallable, Category = "UI")
    void OptimizeListView(UListView* ListView)
    {
        if (!ListView)
        {
            return;
        }
        
        // 启用虚拟化（只渲染可见项）
        ListView->SetWheelScrollMultiplier(2.0f);
        
        // 限制同时渲染的项数
        // ListView->SetMaxVisibleItems(10);
    }
};
```

#### 4. 移动端输入优化

```cpp
// 触摸输入优化
UCLASS()
class AMobileTouchController : public APlayerController
{
    GENERATED_BODY()
    
protected:
    virtual void SetupInputComponent() override
    {
        Super::SetupInputComponent();
        
        // 使用触摸事件而非轮询
        InputComponent->BindTouch(IE_Pressed, this, &ThisClass::OnTouchPressed);
        InputComponent->BindTouch(IE_Released, this, &ThisClass::OnTouchReleased);
        InputComponent->BindTouch(IE_Repeat, this, &ThisClass::OnTouchMoved);
    }
    
private:
    void OnTouchPressed(ETouchIndex::Type FingerIndex, FVector Location)
    {
        // 处理触摸开始
    }
    
    void OnTouchReleased(ETouchIndex::Type FingerIndex, FVector Location)
    {
        // 处理触摸结束
    }
    
    void OnTouchMoved(ETouchIndex::Type FingerIndex, FVector Location)
    {
        // 处理触摸移动（只在移动时调用，不是每帧）
    }
};
```

## 性能监控系统实现

建立完善的性能监控系统，可以及时发现和解决性能问题。

### 实时性能监控

```cpp
// 综合性能监控管理器
UCLASS()
class UPerformanceMonitor : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override
    {
        Super::Initialize(Collection);
        
        // 启动监控
        GetWorld()->GetTimerManager().SetTimer(
            MonitorTimer,
            this,
            &ThisClass::UpdatePerformanceMetrics,
            1.0f,
            true
        );
    }
    
    virtual void Deinitialize() override
    {
        GetWorld()->GetTimerManager().ClearTimer(MonitorTimer);
        SavePerformanceReport();
        Super::Deinitialize();
    }
    
private:
    void UpdatePerformanceMetrics()
    {
        // 1. 收集FPS数据
        float CurrentFPS = 1.0f / GetWorld()->GetDeltaSeconds();
        FPSHistory.Add(CurrentFPS);
        if (FPSHistory.Num() > 300)  // 保留最近5分钟（@1Hz采样）
        {
            FPSHistory.RemoveAt(0);
        }
        
        // 2. 收集内存数据
        FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
        MemoryHistory.Add(MemStats.UsedPhysical / (1024.0f * 1024.0f));  // MB
        if (MemoryHistory.Num() > 300)
        {
            MemoryHistory.RemoveAt(0);
        }
        
        // 3. 检测性能问题
        DetectPerformanceIssues(CurrentFPS, MemStats);
        
        // 4. 记录到CSV（可选）
        LogPerformanceToCSV(CurrentFPS, MemStats);
    }
    
    void DetectPerformanceIssues(float FPS, const FPlatformMemoryStats& MemStats)
    {
        // FPS过低
        if (FPS < 30.0f)
        {
            UE_LOG(LogTemp, Warning, TEXT("Performance: Low FPS detected: %.2f"), FPS);
            
            // 自动触发性能分析
            // Trace.Start
        }
        
        // 内存过高
        float MemoryUsageMB = MemStats.UsedPhysical / (1024.0f * 1024.0f);
        if (MemoryUsageMB > 3000.0f)  // 超过3GB
        {
            UE_LOG(LogTemp, Warning, 
                   TEXT("Performance: High memory usage: %.2f MB"), 
                   MemoryUsageMB);
        }
        
        // 内存泄漏检测（内存持续增长）
        if (MemoryHistory.Num() >= 60)  // 至少1分钟数据
        {
            float RecentAvg = 0.0f;
            float OldAvg = 0.0f;
            
            for (int32 i = 0; i < 30; ++i)
            {
                OldAvg += MemoryHistory[i];
                RecentAvg += MemoryHistory[MemoryHistory.Num() - 30 + i];
            }
            
            OldAvg /= 30.0f;
            RecentAvg /= 30.0f;
            
            if (RecentAvg > OldAvg * 1.2f)  // 增长20%
            {
                UE_LOG(LogTemp, Warning, 
                       TEXT("Performance: Possible memory leak detected! Old: %.2f MB, Recent: %.2f MB"),
                       OldAvg, RecentAvg);
            }
        }
    }
    
    void LogPerformanceToCSV(float FPS, const FPlatformMemoryStats& MemStats)
    {
#if !UE_BUILD_SHIPPING
        if (!bEnableCSVLogging)
        {
            return;
        }
        
        FString LogLine = FString::Printf(
            TEXT("%.3f,%.2f,%.2f,%.2f,%.2f\n"),
            GetWorld()->GetTimeSeconds(),
            FPS,
            MemStats.UsedPhysical / (1024.0f * 1024.0f),
            MemStats.UsedVirtual / (1024.0f * 1024.0f),
            MemStats.PeakUsedPhysical / (1024.0f * 1024.0f)
        );
        
        FString FilePath = FPaths::ProjectSavedDir() / TEXT("Profiling/PerformanceLog.csv");
        FFileHelper::SaveStringToFile(LogLine, *FilePath, 
                                      FFileHelper::EEncodingOptions::AutoDetect,
                                      &IFileManager::Get(), 
                                      FILEWRITE_Append);
#endif
    }
    
    void SavePerformanceReport()
    {
        // 生成性能报告
        FString Report;
        Report += TEXT("=== Performance Report ===\n\n");
        
        // FPS统计
        if (FPSHistory.Num() > 0)
        {
            float AvgFPS = 0.0f;
            float MinFPS = FPSHistory[0];
            float MaxFPS = FPSHistory[0];
            
            for (float FPS : FPSHistory)
            {
                AvgFPS += FPS;
                MinFPS = FMath::Min(MinFPS, FPS);
                MaxFPS = FMath::Max(MaxFPS, FPS);
            }
            AvgFPS /= FPSHistory.Num();
            
            Report += FString::Printf(TEXT("FPS Statistics:\n"));
            Report += FString::Printf(TEXT("  Average: %.2f\n"), AvgFPS);
            Report += FString::Printf(TEXT("  Min: %.2f\n"), MinFPS);
            Report += FString::Printf(TEXT("  Max: %.2f\n"), MaxFPS);
            Report += TEXT("\n");
        }
        
        // 内存统计
        if (MemoryHistory.Num() > 0)
        {
            float AvgMem = 0.0f;
            float MinMem = MemoryHistory[0];
            float MaxMem = MemoryHistory[0];
            
            for (float Mem : MemoryHistory)
            {
                AvgMem += Mem;
                MinMem = FMath::Min(MinMem, Mem);
                MaxMem = FMath::Max(MaxMem, Mem);
            }
            AvgMem /= MemoryHistory.Num();
            
            Report += FString::Printf(TEXT("Memory Statistics (MB):\n"));
            Report += FString::Printf(TEXT("  Average: %.2f\n"), AvgMem);
            Report += FString::Printf(TEXT("  Min: %.2f\n"), MinMem);
            Report += FString::Printf(TEXT("  Max: %.2f\n"), MaxMem);
        }
        
        // 保存报告
        FString FilePath = FPaths::ProjectSavedDir() / TEXT("Profiling/PerformanceReport.txt");
        FFileHelper::SaveStringToFile(Report, *FilePath);
        
        UE_LOG(LogTemp, Log, TEXT("Performance report saved to: %s"), *FilePath);
    }
    
private:
    FTimerHandle MonitorTimer;
    TArray<float> FPSHistory;
    TArray<float> MemoryHistory;
    
    UPROPERTY(Config)
    bool bEnableCSVLogging = false;
};
```

### 性能预算系统

```cpp
// 性能预算配置
USTRUCT(BlueprintType)
struct FPerformanceBudget
{
    GENERATED_BODY()
    
    // FPS预算
    UPROPERTY(EditAnywhere, Category = "Budget")
    float MinAcceptableFPS = 30.0f;
    
    UPROPERTY(EditAnywhere, Category = "Budget")
    float TargetFPS = 60.0f;
    
    // 时间预算（毫秒）
    UPROPERTY(EditAnywhere, Category = "Budget")
    float GameThreadBudgetMS = 10.0f;
    
    UPROPERTY(EditAnywhere, Category = "Budget")
    float RenderThreadBudgetMS = 10.0f;
    
    UPROPERTY(EditAnywhere, Category = "Budget")
    float GPUBudgetMS = 13.0f;
    
    // 内存预算（MB）
    UPROPERTY(EditAnywhere, Category = "Budget")
    float TotalMemoryBudgetMB = 4096.0f;
    
    UPROPERTY(EditAnywhere, Category = "Budget")
    float TextureMemoryBudgetMB = 1024.0f;
    
    // 网络预算
    UPROPERTY(EditAnywhere, Category = "Budget")
    float MaxBandwidthKBps = 200.0f;
    
    UPROPERTY(EditAnywhere, Category = "Budget")
    int32 MaxReplicatedActors = 100;
};

// 性能预算监控器
UCLASS()
class UPerformanceBudgetMonitor : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    void CheckBudgets(const FPerformanceBudget& Budget)
    {
        // 检查FPS
        float CurrentFPS = 1.0f / GetWorld()->GetDeltaSeconds();
        if (CurrentFPS < Budget.MinAcceptableFPS)
        {
            UE_LOG(LogTemp, Error, 
                   TEXT("Budget Violation: FPS %.2f < %.2f"), 
                   CurrentFPS, Budget.MinAcceptableFPS);
            
            // 触发降级措施
            TriggerQualityDowngrade();
        }
        
        // 检查内存
        FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
        float UsedMemoryMB = MemStats.UsedPhysical / (1024.0f * 1024.0f);
        if (UsedMemoryMB > Budget.TotalMemoryBudgetMB)
        {
            UE_LOG(LogTemp, Error, 
                   TEXT("Budget Violation: Memory %.2f MB > %.2f MB"), 
                   UsedMemoryMB, Budget.TotalMemoryBudgetMB);
            
            // 触发内存清理
            TriggerMemoryCleanup();
        }
    }
    
private:
    void TriggerQualityDowngrade()
    {
        // 自动降低画质
        // 例如：降低阴影质量、减少粒子数量等
    }
    
    void TriggerMemoryCleanup()
    {
        // 触发资源清理
        // 例如：卸载远处的资源、执行GC等
        GEngine->ForceGarbageCollection(true);
    }
};
```

## 最佳实践与性能预算

### 性能优化清单

#### 开发阶段

1. **设计阶段**
   - ✅ 确定目标平台和性能指标
   - ✅ 制定性能预算
   - ✅ 选择合适的技术方案

2. **实现阶段**
   - ✅ 默认禁用不必要的Tick
   - ✅ 使用事件驱动替代轮询
   - ✅ 使用软引用替代硬引用
   - ✅ 实现对象池
   - ✅ 使用Replication Graph
   - ✅ 优化材质复杂度
   - ✅ 配置LOD系统

3. **测试阶段**
   - ✅ 定期进行性能Profile
   - ✅ 测试极限场景（最大玩家数、最密集战斗）
   - ✅ 检查内存泄漏
   - ✅ 验证网络性能

4. **优化阶段**
   - ✅ 识别性能瓶颈
   - ✅ 针对性优化
   - ✅ 回归测试
   - ✅ 文档化优化措施

### 性能优化原则

1. **测量先于优化**：先用工具定位瓶颈，再动手优化
2. **优先优化瓶颈**：优化最慢的部分才能得到最大收益
3. **权衡质量与性能**：根据目标平台合理取舍
4. **自动化监控**：建立性能监控系统，及时发现问题
5. **预算驱动开发**：严格遵守性能预算，防患于未然

### 常见性能陷阱

```cpp
// ❌ 错误示例合集

// 1. 每帧进行碰撞检测
void BadTick(float DeltaTime)
{
    // 不要这样！
    TArray<AActor*> Actors;
    UGameplayStatics::GetAllActorsOfClass(GetWorld(), AActor::StaticClass(), Actors);
}

// 2. 每帧进行字符串操作
void BadStringOperation()
{
    // 字符串拼接很慢！
    FString Result;
    for (int32 i = 0; i < 1000; ++i)
    {
        Result += FString::Printf(TEXT("Item %d, "), i);
    }
}

// 3. 不合理的动态创建
void BadActorSpawning()
{
    // 每帧创建销毁Actor
    for (int32 i = 0; i < 10; ++i)
    {
        AActor* Actor = GetWorld()->SpawnActor<AActor>();
        Actor->Destroy();
    }
}

// 4. 过度使用Cast
void BadCasting(AActor* Actor)
{
    // 每帧Cast很慢
    if (AMyActor* MyActor = Cast<AMyActor>(Actor))
    {
        // ...
    }
}

// ✅ 正确示例

// 1. 使用空间查询
void GoodCollisionCheck(const FVector& Center, float Radius)
{
    TArray<FOverlapResult> Results;
    GetWorld()->OverlapMultiByChannel(
        Results, Center, FQuat::Identity, ECC_Pawn,
        FCollisionShape::MakeSphere(Radius)
    );
}

// 2. 使用StringBuilder
void GoodStringOperation()
{
    TStringBuilder<1024> Builder;
    for (int32 i = 0; i < 1000; ++i)
    {
        Builder.Appendf(TEXT("Item %d, "), i);
    }
    FString Result = Builder.ToString();
}

// 3. 使用对象池
void GoodActorSpawning()
{
    for (int32 i = 0; i < 10; ++i)
    {
        AActor* Actor = ActorPool->Acquire();
        // 使用Actor
        ActorPool->Release(Actor);
    }
}

// 4. 缓存Cast结果
void GoodCasting(AActor* Actor)
{
    if (!CachedMyActor)
    {
        CachedMyActor = Cast<AMyActor>(Actor);
    }
    
    if (CachedMyActor)
    {
        // 使用缓存的指针
    }
}
```

## 总结

性能优化是一个系统工程，需要从设计、开发、测试到发布的全流程关注。Lyra为我们展示了一个完整的性能优化案例：

1. **完善的性能监控系统**：实时收集和展示各项性能指标
2. **智能的网络复制**：Replication Graph大幅提升多人游戏性能
3. **精细的资源管理**：Asset Manager实现按需加载和卸载
4. **优化的渲染管线**：LOD、HLOD、Instance化等技术的综合运用
5. **事件驱动的架构**：减少不必要的轮询和计算

通过学习Lyra的性能优化实践，结合UE5强大的性能分析工具，你可以打造出在各种平台上都流畅运行的高品质游戏。

记住：**性能优化永无止境，但要优先优化瓶颈，权衡质量与性能，建立自动化监控，遵循性能预算**。

## 参考资源

- [Unreal Engine 5 Documentation - Performance and Profiling](https://docs.unrealengine.com/5.0/en-US/performance-and-profiling-in-unreal-engine/)
- [Replication Graph](https://docs.unrealengine.com/5.0/en-US/replication-graph-in-unreal-engine/)
- [Asset Manager](https://docs.unrealengine.com/5.0/en-US/asset-management-in-unreal-engine/)
- [Lyra Sample Game](https://docs.unrealengine.com/5.0/en-US/lyra-sample-game-in-unreal-engine/)

---

**下一篇预告**：《多人游戏与网络同步》—— 深入Lyra的网络架构，掌握客户端预测、服务器权威、延迟补偿等关键技术。