# 外观系统与商店实现

## 概述

在现代多人游戏中，外观定制系统（Cosmetics System）是玩家个性化表达和游戏变现的核心功能之一。Lyra 提供了一套完整的外观系统架构，支持角色部件（Character Parts）的动态组合、材质替换、网络同步以及与库存系统的深度集成。

本章将深入探讨 Lyra 的外观系统实现，包括：
- **Character Parts 系统**：模块化的角色部件管理
- **Loadout 系统**：外观配置的存储与应用
- **商店系统实现**：货币、购买流程与拥有验证
- **网络同步策略**：高效的外观数据复制
- **材质动态替换**：运行时更换皮肤和贴图
- **实战案例**：完整的角色定制系统和战利品箱系统

## 外观系统架构

### 核心组件关系

Lyra 的外观系统采用分层架构，主要包含以下核心组件：

```
┌─────────────────────────────────────────────────────────────┐
│                    Controller Layer                         │
│  ┌────────────────────────────────────────────────────┐    │
│  │  ULyraControllerComponent_CharacterParts           │    │
│  │  - 存储玩家的外观配置（Loadout）                     │    │
│  │  - 监听 Pawn 切换事件                               │    │
│  │  - 应用外观到当前 Pawn                              │    │
│  └────────────────────────────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────────┘
                       │ OnPossessedPawnChanged
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                      Pawn Layer                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │  ULyraPawnComponent_CharacterParts                 │    │
│  │  - 管理当前 Pawn 的外观部件实例                      │    │
│  │  - 负责生成和销毁 Character Parts                   │    │
│  │  - 处理网络复制（FastArraySerializer）              │    │
│  │  - 根据标签选择合适的 Body Mesh                     │    │
│  └────────────────────────────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────────┘
                       │ SpawnActorForEntry
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   Character Part Actors                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Head Actor   │  │ Body Actor   │  │ Weapon Skin  │     │
│  │ (Hat/Hair)   │  │ (Outfit)     │  │ (Visual)     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│  - 通过 ChildActorComponent 附加到角色 Mesh               │
│  - 可实现 IGameplayTagAssetInterface 提供标签             │
└─────────────────────────────────────────────────────────────┘
```

### 数据结构设计

#### FLyraCharacterPart - 外观部件定义

```cpp
// LyraCharacterPartTypes.h
USTRUCT(BlueprintType)
struct FLyraCharacterPart
{
    GENERATED_BODY()

    // 要生成的部件类（Actor 类）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSubclassOf<AActor> PartClass;

    // 附加的骨骼插槽名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FName SocketName;

    // 碰撞模式设置
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    ECharacterCustomizationCollisionMode CollisionMode = 
        ECharacterCustomizationCollisionMode::NoCollision;

    // 比较两个部件是否等价（忽略碰撞模式）
    static bool AreEquivalentParts(const FLyraCharacterPart& A, const FLyraCharacterPart& B)
    {
        return (A.PartClass == B.PartClass) && (A.SocketName == B.SocketName);
    }
};
```

**设计要点：**
- `PartClass`：可以是任何 Actor 类，提供最大灵活性
- `SocketName`：支持附加到骨骼的不同插槽（如 "head_socket", "weapon_socket"）
- `CollisionMode`：默认禁用碰撞，避免外观部件影响游戏逻辑

#### FLyraAppliedCharacterPartEntry - 已应用的部件实例

```cpp
// LyraPawnComponent_CharacterParts.h
USTRUCT()
struct FLyraAppliedCharacterPartEntry : public FFastArraySerializerItem
{
    GENERATED_BODY()

private:
    friend FLyraCharacterPartList;
    friend ULyraPawnComponent_CharacterParts;

    // 部件定义
    UPROPERTY()
    FLyraCharacterPart Part;

    // 服务器端的句柄索引
    UPROPERTY(NotReplicated)
    int32 PartHandle = INDEX_NONE;

    // 客户端生成的组件实例
    UPROPERTY(NotReplicated)
    TObjectPtr<UChildActorComponent> SpawnedComponent = nullptr;
};
```

**关键设计：**
- 继承自 `FFastArraySerializerItem`，支持高效的网络增量复制
- `PartHandle` 仅在服务器端使用，不复制到客户端
- `SpawnedComponent` 仅在客户端存在，服务器不生成实际 Actor

#### FLyraCharacterPartList - 部件列表（网络复制）

```cpp
USTRUCT(BlueprintType)
struct FLyraCharacterPartList : public FFastArraySerializer
{
    GENERATED_BODY()

public:
    // FastArraySerializer 合约实现
    void PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize);
    void PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize);
    void PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize);

    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
    {
        return FFastArraySerializer::FastArrayDeltaSerialize<
            FLyraAppliedCharacterPartEntry, 
            FLyraCharacterPartList>(Entries, DeltaParms, *this);
    }

    // 添加部件，返回句柄用于后续移除
    FLyraCharacterPartHandle AddEntry(FLyraCharacterPart NewPart);
    
    // 根据句柄移除部件
    void RemoveEntry(FLyraCharacterPartHandle Handle);
    
    // 清空所有部件
    void ClearAllEntries(bool bBroadcastChangeDelegate);

    // 收集所有部件的 Gameplay Tags
    FGameplayTagContainer CollectCombinedTags() const;

private:
    UPROPERTY()
    TArray<FLyraAppliedCharacterPartEntry> Entries;

    UPROPERTY(NotReplicated)
    TObjectPtr<ULyraPawnComponent_CharacterParts> OwnerComponent;

    int32 PartHandleCounter = 0;
};
```

**网络优化：**
- 使用 `FastArraySerializer` 进行增量复制，只同步变化的部件
- 避免每帧复制整个列表，节省带宽
- 支持客户端预测（虽然外观通常不需要）

## ULyraPawnComponent_CharacterParts - Pawn 外观组件

这是外观系统的核心组件，负责在 Pawn 上生成和管理外观部件。

### 组件实现

```cpp
// LyraPawnComponent_CharacterParts.h
UCLASS(meta=(BlueprintSpawnableComponent))
class ULyraPawnComponent_CharacterParts : public UPawnComponent
{
    GENERATED_BODY()

public:
    ULyraPawnComponent_CharacterParts(const FObjectInitializer& ObjectInitializer);

    //~UActorComponent interface
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
    virtual void OnRegister() override;
    //~End of UActorComponent interface

    // 添加外观部件（仅服务器）
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Cosmetics)
    FLyraCharacterPartHandle AddCharacterPart(const FLyraCharacterPart& NewPart);

    // 移除外观部件（仅服务器）
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Cosmetics)
    void RemoveCharacterPart(FLyraCharacterPartHandle Handle);

    // 移除所有外观部件（仅服务器）
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Cosmetics)
    void RemoveAllCharacterParts();

    // 获取所有已生成的外观 Actor
    UFUNCTION(BlueprintCallable, BlueprintPure=false, BlueprintCosmetic, Category=Cosmetics)
    TArray<AActor*> GetCharacterPartActors() const;

    // 获取父 Mesh 组件（ACharacter 的 SkeletalMeshComponent）
    USkeletalMeshComponent* GetParentMeshComponent() const;

    // 获取要附加的场景组件
    USceneComponent* GetSceneComponentToAttachTo() const;

    // 获取合并后的 Gameplay Tags
    UFUNCTION(BlueprintCallable, BlueprintPure=false, BlueprintCosmetic, Category=Cosmetics)
    FGameplayTagContainer GetCombinedTags(FGameplayTag RequiredPrefix) const;

    void BroadcastChanged();

public:
    // 外观部件变化时的委托
    UPROPERTY(BlueprintAssignable, Category=Cosmetics, BlueprintCallable)
    FLyraSpawnedCharacterPartsChanged OnCharacterPartsChanged;

private:
    // 网络复制的部件列表
    UPROPERTY(Replicated, Transient)
    FLyraCharacterPartList CharacterPartList;

    // 根据外观标签选择身体网格的规则
    UPROPERTY(EditAnywhere, Category=Cosmetics)
    FLyraAnimBodyStyleSelectionSet BodyMeshes;
};
```

### 核心功能实现

#### 1. 添加外观部件

```cpp
// LyraPawnComponent_CharacterParts.cpp
FLyraCharacterPartHandle ULyraPawnComponent_CharacterParts::AddCharacterPart(
    const FLyraCharacterPart& NewPart)
{
    return CharacterPartList.AddEntry(NewPart);
}

FLyraCharacterPartHandle FLyraCharacterPartList::AddEntry(FLyraCharacterPart NewPart)
{
    FLyraCharacterPartHandle Result;
    Result.PartHandle = PartHandleCounter++;

    if (ensure(OwnerComponent && OwnerComponent->GetOwner() && 
               OwnerComponent->GetOwner()->HasAuthority()))
    {
        // 添加新条目
        FLyraAppliedCharacterPartEntry& NewEntry = Entries.AddDefaulted_GetRef();
        NewEntry.Part = NewPart;
        NewEntry.PartHandle = Result.PartHandle;
    
        // 在客户端生成 Actor
        if (SpawnActorForEntry(NewEntry))
        {
            OwnerComponent->BroadcastChanged();
        }

        // 标记为需要复制
        MarkItemDirty(NewEntry);
    }

    return Result;
}
```

**流程解析：**
1. 生成唯一的 PartHandle（句柄计数器递增）
2. 添加到 Entries 数组
3. 如果在客户端，调用 `SpawnActorForEntry` 生成实际 Actor
4. 标记条目为脏，触发网络复制
5. 广播变化事件，通知 UI 或其他系统

#### 2. 生成外观 Actor

```cpp
bool FLyraCharacterPartList::SpawnActorForEntry(FLyraAppliedCharacterPartEntry& Entry)
{
    bool bCreatedAnyActors = false;

    if (ensure(OwnerComponent) && !OwnerComponent->IsNetMode(NM_DedicatedServer))
    {
        if (Entry.Part.PartClass != nullptr)
        {
            if (USceneComponent* ComponentToAttachTo = 
                OwnerComponent->GetSceneComponentToAttachTo())
            {
                // 创建 ChildActorComponent
                UChildActorComponent* PartComponent = 
                    NewObject<UChildActorComponent>(OwnerComponent->GetOwner());

                // 设置附加关系
                PartComponent->SetupAttachment(ComponentToAttachTo, Entry.Part.SocketName);
                PartComponent->SetChildActorClass(Entry.Part.PartClass);
                PartComponent->RegisterComponent();

                if (AActor* SpawnedActor = PartComponent->GetChildActor())
                {
                    // 处理碰撞设置
                    switch (Entry.Part.CollisionMode)
                    {
                    case ECharacterCustomizationCollisionMode::NoCollision:
                        SpawnedActor->SetActorEnableCollision(false);
                        break;
                    case ECharacterCustomizationCollisionMode::UseCollisionFromCharacterPart:
                        // 保持部件的原始碰撞设置
                        break;
                    }

                    // 设置 Tick 依赖，确保外观部件在父组件之后更新
                    if (USceneComponent* SpawnedRootComponent = 
                        SpawnedActor->GetRootComponent())
                    {
                        SpawnedRootComponent->AddTickPrerequisiteComponent(ComponentToAttachTo);
                    }
                }

                Entry.SpawnedComponent = PartComponent;
                bCreatedAnyActors = true;
            }
        }
    }

    return bCreatedAnyActors;
}
```

**技术要点：**
- 使用 `UChildActorComponent` 动态生成 Actor，而非直接 SpawnActor
- 这样可以自动处理附加关系和生命周期
- Dedicated Server 不生成外观 Actor，节省性能
- Tick 依赖确保动画更新顺序正确

#### 3. 网络复制回调

```cpp
void FLyraCharacterPartList::PostReplicatedAdd(
    const TArrayView<int32> AddedIndices, int32 FinalSize)
{
    bool bCreatedAnyActors = false;
    for (int32 Index : AddedIndices)
    {
        FLyraAppliedCharacterPartEntry& Entry = Entries[Index];
        bCreatedAnyActors |= SpawnActorForEntry(Entry);
    }

    if (bCreatedAnyActors && ensure(OwnerComponent))
    {
        OwnerComponent->BroadcastChanged();
    }
}

void FLyraCharacterPartList::PreReplicatedRemove(
    const TArrayView<int32> RemovedIndices, int32 FinalSize)
{
    bool bDestroyedAnyActors = false;
    for (int32 Index : RemovedIndices)
    {
        FLyraAppliedCharacterPartEntry& Entry = Entries[Index];
        bDestroyedAnyActors |= DestroyActorForEntry(Entry);
    }

    if (bDestroyedAnyActors && ensure(OwnerComponent))
    {
        OwnerComponent->BroadcastChanged();
    }
}

void FLyraCharacterPartList::PostReplicatedChange(
    const TArrayView<int32> ChangedIndices, int32 FinalSize)
{
    bool bChangedAnyActors = false;

    // 不支持属性变化的复制，直接销毁并重新创建
    for (int32 Index : ChangedIndices)
    {
        FLyraAppliedCharacterPartEntry& Entry = Entries[Index];
        bChangedAnyActors |= DestroyActorForEntry(Entry);
        bChangedAnyActors |= SpawnActorForEntry(Entry);
    }

    if (bChangedAnyActors && ensure(OwnerComponent))
    {
        OwnerComponent->BroadcastChanged();
    }
}
```

**FastArraySerializer 优势：**
- 只复制增量变化，而非整个数组
- 自动处理添加、删除、修改事件
- 客户端接收后立即生成对应的外观 Actor

#### 4. 身体网格选择

```cpp
void ULyraPawnComponent_CharacterParts::BroadcastChanged()
{
    const bool bReinitPose = true;

    // 根据外观标签选择合适的身体网格
    if (USkeletalMeshComponent* MeshComponent = GetParentMeshComponent())
    {
        // 收集所有外观部件的标签
        const FGameplayTagContainer MergedTags = GetCombinedTags(FGameplayTag());
        
        // 根据规则选择最佳的身体网格
        USkeletalMesh* DesiredMesh = BodyMeshes.SelectBestBodyStyle(MergedTags);

        // 应用网格（如果网格未变化则为空操作）
        MeshComponent->SetSkeletalMesh(DesiredMesh, bReinitPose);

        // 应用强制的物理资产
        if (UPhysicsAsset* PhysicsAsset = BodyMeshes.ForcedPhysicsAsset)
        {
            MeshComponent->SetPhysicsAsset(PhysicsAsset, bReinitPose);
        }
    }

    // 通知观察者（如团队颜色应用等）
    OnCharacterPartsChanged.Broadcast(this);
}
```

**智能网格选择：**
- 根据外观部件的 Gameplay Tags 自动选择合适的身体网格
- 例如：穿重型装甲时使用更壮的身体网格
- 支持性别、体型、风格等多维度选择

## ULyraControllerComponent_CharacterParts - Controller 外观组件

Controller 组件负责存储玩家的外观配置（Loadout），并在 Pawn 切换时自动应用。

### 组件实现

```cpp
// LyraControllerComponent_CharacterParts.h
UCLASS(meta = (BlueprintSpawnableComponent))
class ULyraControllerComponent_CharacterParts : public UControllerComponent
{
    GENERATED_BODY()

public:
    // 添加外观部件到配置
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Cosmetics)
    void AddCharacterPart(const FLyraCharacterPart& NewPart);

    // 从配置中移除外观部件
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Cosmetics)
    void RemoveCharacterPart(const FLyraCharacterPart& PartToRemove);

    // 清空所有外观配置
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Cosmetics)
    void RemoveAllCharacterParts();

protected:
    // 外观部件配置列表
    UPROPERTY(EditAnywhere, Category=Cosmetics)
    TArray<FLyraControllerCharacterPartEntry> CharacterParts;

private:
    // 获取当前 Pawn 的外观组件
    ULyraPawnComponent_CharacterParts* GetPawnCustomizer() const;

    // Pawn 切换回调
    UFUNCTION()
    void OnPossessedPawnChanged(APawn* OldPawn, APawn* NewPawn);
};
```

### Pawn 切换时自动应用外观

```cpp
// LyraControllerComponent_CharacterParts.cpp
void ULyraControllerComponent_CharacterParts::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        if (AController* OwningController = GetController<AController>())
        {
            // 监听 Pawn 切换事件
            OwningController->OnPossessedPawnChanged.AddDynamic(
                this, &ThisClass::OnPossessedPawnChanged);

            // 如果已经有 Pawn，立即应用
            if (APawn* ControlledPawn = GetPawn<APawn>())
            {
                OnPossessedPawnChanged(nullptr, ControlledPawn);
            }
        }
    }
}

void ULyraControllerComponent_CharacterParts::OnPossessedPawnChanged(
    APawn* OldPawn, APawn* NewPawn)
{
    // 从旧 Pawn 移除外观
    if (ULyraPawnComponent_CharacterParts* OldCustomizer = 
        OldPawn ? OldPawn->FindComponentByClass<ULyraPawnComponent_CharacterParts>() : nullptr)
    {
        for (FLyraControllerCharacterPartEntry& Entry : CharacterParts)
        {
            OldCustomizer->RemoveCharacterPart(Entry.Handle);
            Entry.Handle.Reset();
        }
    }

    // 应用到新 Pawn
    if (ULyraPawnComponent_CharacterParts* NewCustomizer = 
        NewPawn ? NewPawn->FindComponentByClass<ULyraPawnComponent_CharacterParts>() : nullptr)
    {
        for (FLyraControllerCharacterPartEntry& Entry : CharacterParts)
        {
            // 避免重复添加
            if (!Entry.Handle.IsValid() && 
                Entry.Source != ECharacterPartSource::NaturalSuppressedViaCheat)
            {
                Entry.Handle = NewCustomizer->AddCharacterPart(Entry.Part);
            }
        }
    }
}
```

**设计理念：**
- 外观配置存储在 Controller 上，独立于 Pawn 的生命周期
- 玩家重生时自动应用之前的外观
- 支持观察者模式或多 Pawn 控制场景

## Character Parts 系统详解

### 外观部件类型

Lyra 支持多种类型的外观部件：

#### 1. 头部部件（Head Parts）

```cpp
// 示例：帽子 Actor
UCLASS()
class AHatCharacterPart : public AActor, public IGameplayTagAssetInterface
{
    GENERATED_BODY()

public:
    AHatCharacterPart()
    {
        // 创建静态网格组件
        MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Mesh"));
        RootComponent = MeshComponent;
        MeshComponent->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }

    // 实现 Gameplay Tag 接口
    virtual void GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const override
    {
        TagContainer.AppendTags(CosmeticTags);
    }

protected:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<UStaticMeshComponent> MeshComponent;

    // 外观标签（如 "Cosmetic.Hat.Baseball"）
    UPROPERTY(EditDefaultsOnly, Category=Cosmetics)
    FGameplayTagContainer CosmeticTags;
};
```

**应用场景：**
- 帽子、头盔、面具
- 发型、发色
- 眼镜、耳环等配饰

#### 2. 身体部件（Body Parts）

```cpp
// 示例：服装 Actor
UCLASS()
class AOutfitCharacterPart : public AActor, public IGameplayTagAssetInterface
{
    GENERATED_BODY()

public:
    AOutfitCharacterPart()
    {
        // 骨骼网格组件，可以有动画
        SkeletalMeshComponent = CreateDefaultSubobject<USkeletalMeshComponent>(TEXT("SkeletalMesh"));
        RootComponent = SkeletalMeshComponent;
        SkeletalMeshComponent->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }

    virtual void GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const override
    {
        TagContainer.AppendTags(CosmeticTags);
    }

protected:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<USkeletalMeshComponent> SkeletalMeshComponent;

    UPROPERTY(EditDefaultsOnly, Category=Cosmetics)
    FGameplayTagContainer CosmeticTags;
};
```

**应用场景：**
- 上衣、裤子、鞋子
- 盔甲、护具
- 背包、腰带

#### 3. 武器皮肤（Weapon Skins）

```cpp
// 示例：武器外观 Actor
UCLASS()
class AWeaponSkinCharacterPart : public AActor
{
    GENERATED_BODY()

public:
    AWeaponSkinCharacterPart()
    {
        WeaponMeshComponent = CreateDefaultSubobject<USkeletalMeshComponent>(TEXT("WeaponMesh"));
        RootComponent = WeaponMeshComponent;
        WeaponMeshComponent->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }

    // 应用材质到武器网格
    UFUNCTION(BlueprintCallable, Category=Cosmetics)
    void ApplySkinMaterial(UMaterialInterface* SkinMaterial)
    {
        if (SkinMaterial && WeaponMeshComponent)
        {
            for (int32 i = 0; i < WeaponMeshComponent->GetNumMaterials(); ++i)
            {
                WeaponMeshComponent->SetMaterial(i, SkinMaterial);
            }
        }
    }

protected:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<USkeletalMeshComponent> WeaponMeshComponent;

    UPROPERTY(EditDefaultsOnly, Category=Cosmetics)
    TObjectPtr<UMaterialInterface> DefaultSkinMaterial;
};
```

**应用场景：**
- 武器皮肤、迷彩
- 武器挂件、饰品
- 特效、粒子效果

### 外观部件的 Gameplay Tags

使用 Gameplay Tags 标记外观部件，可以实现智能的身体网格选择和外观组合：

```cpp
// LyraCosmeticAnimationTypes.h
USTRUCT(BlueprintType)
struct FLyraAnimBodyStyleSelectionEntry
{
    GENERATED_BODY()

    // 要使用的骨骼网格
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<USkeletalMesh> Mesh = nullptr;

    // 需要满足的标签（所有标签都必须存在）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta=(Categories="Cosmetic"))
    FGameplayTagContainer RequiredTags;
};

USTRUCT(BlueprintType)
struct FLyraAnimBodyStyleSelectionSet
{
    GENERATED_BODY()
        
    // 规则列表，第一个匹配的规则将被使用
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta=(TitleProperty=Mesh))
    TArray<FLyraAnimBodyStyleSelectionEntry> MeshRules;

    // 默认网格（当没有规则匹配时使用）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<USkeletalMesh> DefaultMesh = nullptr;

    // 强制使用的物理资产
    UPROPERTY(EditAnywhere)
    TObjectPtr<UPhysicsAsset> ForcedPhysicsAsset = nullptr;

    // 根据标签选择最佳身体网格
    USkeletalMesh* SelectBestBodyStyle(const FGameplayTagContainer& CosmeticTags) const;
};
```

**实现示例：**

```cpp
// LyraCosmeticAnimationTypes.cpp
USkeletalMesh* FLyraAnimBodyStyleSelectionSet::SelectBestBodyStyle(
    const FGameplayTagContainer& CosmeticTags) const
{
    // 遍历规则，找到第一个匹配的
    for (const FLyraAnimBodyStyleSelectionEntry& Rule : MeshRules)
    {
        if (CosmeticTags.HasAll(Rule.RequiredTags))
        {
            if (Rule.Mesh != nullptr)
            {
                return Rule.Mesh;
            }
        }
    }

    // 没有匹配的规则，返回默认网格
    return DefaultMesh;
}
```

**配置示例（Blueprint）：**

```
MeshRules:
  [0]:
    Mesh: SK_Manny_HeavyArmor
    RequiredTags: [Cosmetic.Armor.Heavy, Cosmetic.Gender.Male]
  [1]:
    Mesh: SK_Quinn_HeavyArmor
    RequiredTags: [Cosmetic.Armor.Heavy, Cosmetic.Gender.Female]
  [2]:
    Mesh: SK_Manny_LightArmor
    RequiredTags: [Cosmetic.Armor.Light, Cosmetic.Gender.Male]
  [3]:
    Mesh: SK_Quinn_LightArmor
    RequiredTags: [Cosmetic.Armor.Light, Cosmetic.Gender.Female]

DefaultMesh: SK_Manny_Default
```

## 材质和贴图动态替换

### 材质实例动态（MID）

动态替换材质和贴图是外观系统的核心功能之一：

```cpp
// 角色外观材质管理器
UCLASS()
class UCharacterCosmeticMaterialManager : public UActorComponent
{
    GENERATED_BODY()

public:
    // 应用皮肤材质
    UFUNCTION(BlueprintCallable, Category=Cosmetics)
    void ApplySkinMaterial(UMaterialInterface* BaseMaterial, 
                           UTexture2D* AlbedoTexture,
                           UTexture2D* NormalTexture,
                           UTexture2D* ORMTexture)
    {
        if (!BaseMaterial || !GetOwner())
            return;

        // 获取角色的骨骼网格组件
        ACharacter* Character = Cast<ACharacter>(GetOwner());
        if (!Character || !Character->GetMesh())
            return;

        USkeletalMeshComponent* MeshComp = Character->GetMesh();

        // 为每个材质槽创建动态材质实例
        for (int32 i = 0; i < MeshComp->GetNumMaterials(); ++i)
        {
            UMaterialInstanceDynamic* MID = 
                MeshComp->CreateAndSetMaterialInstanceDynamic(i);

            if (MID)
            {
                // 设置基础材质
                if (BaseMaterial)
                {
                    MID->SetParentMaterial(BaseMaterial);
                }

                // 设置纹理参数
                if (AlbedoTexture)
                {
                    MID->SetTextureParameterValue(TEXT("AlbedoTexture"), AlbedoTexture);
                }
                if (NormalTexture)
                {
                    MID->SetTextureParameterValue(TEXT("NormalTexture"), NormalTexture);
                }
                if (ORMTexture)
                {
                    MID->SetTextureParameterValue(TEXT("ORMTexture"), ORMTexture);
                }
            }
        }
    }

    // 应用颜色变化
    UFUNCTION(BlueprintCallable, Category=Cosmetics)
    void ApplyColorTint(FLinearColor TintColor)
    {
        ACharacter* Character = Cast<ACharacter>(GetOwner());
        if (!Character || !Character->GetMesh())
            return;

        for (int32 i = 0; i < Character->GetMesh()->GetNumMaterials(); ++i)
        {
            if (UMaterialInstanceDynamic* MID = 
                Cast<UMaterialInstanceDynamic>(Character->GetMesh()->GetMaterial(i)))
            {
                MID->SetVectorParameterValue(TEXT("TintColor"), TintColor);
            }
        }
    }

    // 应用团队颜色
    UFUNCTION(BlueprintCallable, Category=Cosmetics)
    void ApplyTeamColors(const FLinearColor& PrimaryColor, 
                         const FLinearColor& SecondaryColor)
    {
        ACharacter* Character = Cast<ACharacter>(GetOwner());
        if (!Character || !Character->GetMesh())
            return;

        for (int32 i = 0; i < Character->GetMesh()->GetNumMaterials(); ++i)
        {
            if (UMaterialInstanceDynamic* MID = 
                Cast<UMaterialInstanceDynamic>(Character->GetMesh()->GetMaterial(i)))
            {
                MID->SetVectorParameterValue(TEXT("TeamPrimaryColor"), PrimaryColor);
                MID->SetVectorParameterValue(TEXT("TeamSecondaryColor"), SecondaryColor);
            }
        }
    }
};
```

### 与团队系统集成

Lyra 提供了 `UAsyncAction_ObserveTeamColors` 来自动监听团队颜色变化：

```cpp
// 在 Character Parts 组件中监听团队颜色
void ULyraPawnComponent_CharacterParts::BeginPlay()
{
    Super::BeginPlay();

    // 订阅团队颜色变化
    if (UAsyncAction_ObserveTeamColors* TeamColorObserver = 
        UAsyncAction_ObserveTeamColors::ObserveTeamColors(GetOwner()))
    {
        TeamColorObserver->OnTeamChanged.AddDynamic(
            this, &ThisClass::OnTeamColorsChanged);
        TeamColorObserver->Activate();
    }
}

void ULyraPawnComponent_CharacterParts::OnTeamColorsChanged(
    bool bTeamSet, int32 TeamId, const ULyraTeamDisplayAsset* DisplayAsset)
{
    if (!bTeamSet || !DisplayAsset)
        return;

    // 应用团队颜色到所有外观部件
    for (AActor* PartActor : GetCharacterPartActors())
    {
        if (UCharacterCosmeticMaterialManager* MaterialManager = 
            PartActor->FindComponentByClass<UCharacterCosmeticMaterialManager>())
        {
            MaterialManager->ApplyTeamColors(
                DisplayAsset->GetPrimaryColor(),
                DisplayAsset->GetSecondaryColor());
        }
    }
}
```

### 材质参数优化

为了提高性能，建议使用材质参数集合（Material Parameter Collection）：

```cpp
// 全局材质参数管理器
UCLASS()
class UGlobalCosmeticParameterManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 设置全局参数集合
    UFUNCTION(BlueprintCallable, Category=Cosmetics)
    void SetGlobalMaterialParameter(FName ParameterName, float Value)
    {
        if (MaterialParameterCollection)
        {
            UKismetMaterialLibrary::SetScalarParameterValue(
                this, MaterialParameterCollection, ParameterName, Value);
        }
    }

    UFUNCTION(BlueprintCallable, Category=Cosmetics)
    void SetGlobalVectorParameter(FName ParameterName, FLinearColor Value)
    {
        if (MaterialParameterCollection)
        {
            UKismetMaterialLibrary::SetVectorParameterValue(
                this, MaterialParameterCollection, ParameterName, Value);
        }
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category=Cosmetics)
    TObjectPtr<UMaterialParameterCollection> MaterialParameterCollection;
};
```

**优势：**
- 所有材质共享同一个参数集合，减少 Draw Call
- 可以全局控制时间、天气、光照等参数
- 性能开销低

## 商店系统实现

### 商店架构设计

```
┌──────────────────────────────────────────────────────────────┐
│                    Shop System Architecture                  │
└──────────────────────────────────────────────────────────────┘

┌─────────────────────────┐
│  ULyraShopSubsystem     │  游戏实例子系统
│  - 加载商店配置         │
│  - 管理商品列表         │
│  - 处理购买请求         │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  ULyraCurrencyComponent │  玩家状态组件
│  - 管理多种货币         │
│  - 同步货币余额         │
│  - 验证交易             │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  ULyraInventorySystem   │  库存管理组件
│  - 存储已拥有的物品     │
│  - 验证拥有状态         │
│  - 发放购买的物品       │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  ULyraSaveGame          │  存档系统
│  - 持久化货币数据       │
│  - 持久化库存数据       │
│  - 云存档同步           │
└─────────────────────────┘
```

### 货币系统

#### 货币数据定义

```cpp
// 货币类型枚举
UENUM(BlueprintType)
enum class ELyraCurrencyType : uint8
{
    SoftCurrency,    // 软货币（游戏内获得）
    HardCurrency,    // 硬货币（充值获得）
    SeasonCurrency,  // 赛季货币
    EventCurrency    // 活动货币
};

// 货币数据结构
USTRUCT(BlueprintType)
struct FLyraCurrencyAmount
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    ELyraCurrencyType CurrencyType = ELyraCurrencyType::SoftCurrency;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 Amount = 0;

    bool IsValid() const { return Amount > 0; }
};

// 货币余额
USTRUCT(BlueprintType)
struct FLyraCurrencyBalance
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TMap<ELyraCurrencyType, int32> Balances;

    // 检查是否有足够的货币
    bool HasEnough(const FLyraCurrencyAmount& Cost) const
    {
        const int32* Balance = Balances.Find(Cost.CurrencyType);
        return Balance && (*Balance >= Cost.Amount);
    }

    // 扣除货币
    bool Deduct(const FLyraCurrencyAmount& Cost)
    {
        int32* Balance = Balances.Find(Cost.CurrencyType);
        if (Balance && *Balance >= Cost.Amount)
        {
            *Balance -= Cost.Amount;
            return true;
        }
        return false;
    }

    // 增加货币
    void Add(const FLyraCurrencyAmount& Reward)
    {
        int32& Balance = Balances.FindOrAdd(Reward.CurrencyType, 0);
        Balance += Reward.Amount;
    }
};
```

#### 货币组件实现

```cpp
// LyraCurrencyComponent.h
UCLASS(ClassGroup=Currency, meta=(BlueprintSpawnableComponent))
class ULyraCurrencyComponent : public UPlayerStateComponent
{
    GENERATED_BODY()

public:
    ULyraCurrencyComponent();

    // 获取货币余额
    UFUNCTION(BlueprintCallable, Category=Currency)
    int32 GetCurrencyBalance(ELyraCurrencyType CurrencyType) const;

    // 检查是否有足够的货币
    UFUNCTION(BlueprintCallable, Category=Currency)
    bool HasEnoughCurrency(const FLyraCurrencyAmount& Cost) const;

    // 扣除货币（仅服务器）
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Currency)
    bool DeductCurrency(const FLyraCurrencyAmount& Cost);

    // 增加货币（仅服务器）
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Currency)
    void AddCurrency(const FLyraCurrencyAmount& Reward);

    // 货币变化委托
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(
        FOnCurrencyChanged, 
        ELyraCurrencyType, CurrencyType, 
        int32, OldAmount, 
        int32, NewAmount);

    UPROPERTY(BlueprintAssignable, Category=Currency)
    FOnCurrencyChanged OnCurrencyChanged;

protected:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    UFUNCTION()
    void OnRep_CurrencyBalance(const FLyraCurrencyBalance& OldBalance);

private:
    // 网络复制的货币余额
    UPROPERTY(ReplicatedUsing=OnRep_CurrencyBalance)
    FLyraCurrencyBalance CurrencyBalance;
};
```

```cpp
// LyraCurrencyComponent.cpp
ULyraCurrencyComponent::ULyraCurrencyComponent()
{
    SetIsReplicatedByDefault(true);
}

void ULyraCurrencyComponent::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(ThisClass, CurrencyBalance);
}

int32 ULyraCurrencyComponent::GetCurrencyBalance(ELyraCurrencyType CurrencyType) const
{
    const int32* Balance = CurrencyBalance.Balances.Find(CurrencyType);
    return Balance ? *Balance : 0;
}

bool ULyraCurrencyComponent::HasEnoughCurrency(const FLyraCurrencyAmount& Cost) const
{
    return CurrencyBalance.HasEnough(Cost);
}

bool ULyraCurrencyComponent::DeductCurrency(const FLyraCurrencyAmount& Cost)
{
    if (!HasAuthority())
    {
        UE_LOG(LogLyra, Warning, TEXT("DeductCurrency called on client!"));
        return false;
    }

    int32 OldAmount = GetCurrencyBalance(Cost.CurrencyType);
    
    if (CurrencyBalance.Deduct(Cost))
    {
        int32 NewAmount = GetCurrencyBalance(Cost.CurrencyType);
        OnCurrencyChanged.Broadcast(Cost.CurrencyType, OldAmount, NewAmount);
        return true;
    }

    return false;
}

void ULyraCurrencyComponent::AddCurrency(const FLyraCurrencyAmount& Reward)
{
    if (!HasAuthority())
    {
        UE_LOG(LogLyra, Warning, TEXT("AddCurrency called on client!"));
        return;
    }

    int32 OldAmount = GetCurrencyBalance(Reward.CurrencyType);
    CurrencyBalance.Add(Reward);
    int32 NewAmount = GetCurrencyBalance(Reward.CurrencyType);
    
    OnCurrencyChanged.Broadcast(Reward.CurrencyType, OldAmount, NewAmount);
}

void ULyraCurrencyComponent::OnRep_CurrencyBalance(const FLyraCurrencyBalance& OldBalance)
{
    // 比较差异并广播变化事件
    for (const auto& Pair : CurrencyBalance.Balances)
    {
        ELyraCurrencyType CurrencyType = Pair.Key;
        int32 NewAmount = Pair.Value;
        
        const int32* OldAmountPtr = OldBalance.Balances.Find(CurrencyType);
        int32 OldAmount = OldAmountPtr ? *OldAmountPtr : 0;

        if (OldAmount != NewAmount)
        {
            OnCurrencyChanged.Broadcast(CurrencyType, OldAmount, NewAmount);
        }
    }
}
```

### 商品定义

```cpp
// 商品数据资产
UCLASS(BlueprintType)
class ULyraShopItemDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 商品唯一 ID
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Shop)
    FName ItemId;

    // 商品显示名称
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display)
    FText DisplayName;

    // 商品描述
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display)
    FText Description;

    // 商品图标
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display)
    TObjectPtr<UTexture2D> Icon;

    // 商品预览网格
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display)
    TObjectPtr<USkeletalMesh> PreviewMesh;

    // 价格
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Economy)
    TArray<FLyraCurrencyAmount> Prices;

    // 商品类型
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Item)
    ELyraShopItemType ItemType;

    // 要授予的外观部件
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Item, 
              meta=(EditCondition="ItemType == ELyraShopItemType::Cosmetic"))
    FLyraCharacterPart CosmeticPart;

    // 要授予的库存物品
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Item,
              meta=(EditCondition="ItemType == ELyraShopItemType::InventoryItem"))
    TSubclassOf<ULyraInventoryItemDefinition> InventoryItemDef;

    // 是否限时
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Availability)
    bool bIsLimitedTime = false;

    // 开始时间
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Availability,
              meta=(EditCondition="bIsLimitedTime"))
    FDateTime StartTime;

    // 结束时间
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Availability,
              meta=(EditCondition="bIsLimitedTime"))
    FDateTime EndTime;

    // 是否当前可用
    UFUNCTION(BlueprintCallable, Category=Availability)
    bool IsCurrentlyAvailable() const
    {
        if (!bIsLimitedTime)
            return true;

        FDateTime Now = FDateTime::Now();
        return Now >= StartTime && Now <= EndTime;
    }
};
```

### 商店子系统

```cpp
// LyraShopSubsystem.h
UCLASS()
class ULyraShopSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 获取所有商品
    UFUNCTION(BlueprintCallable, Category=Shop)
    TArray<ULyraShopItemDefinition*> GetAllShopItems() const;

    // 获取当前可用的商品
    UFUNCTION(BlueprintCallable, Category=Shop)
    TArray<ULyraShopItemDefinition*> GetAvailableShopItems() const;

    // 根据 ID 获取商品
    UFUNCTION(BlueprintCallable, Category=Shop)
    ULyraShopItemDefinition* GetShopItemById(FName ItemId) const;

    // 请求购买商品
    UFUNCTION(BlueprintCallable, Category=Shop)
    void RequestPurchase(APlayerController* PlayerController, FName ItemId);

protected:
    // 验证购买条件
    bool ValidatePurchase(APlayerState* PlayerState, 
                          ULyraShopItemDefinition* ItemDef,
                          FText& OutErrorMessage);

    // 执行购买
    void ExecutePurchase(APlayerState* PlayerState, 
                         ULyraShopItemDefinition* ItemDef);

    // 授予商品
    void GrantItem(APlayerState* PlayerState, 
                   ULyraShopItemDefinition* ItemDef);

private:
    // 商品数据表
    UPROPERTY()
    TMap<FName, TObjectPtr<ULyraShopItemDefinition>> ShopItems;
};
```

```cpp
// LyraShopSubsystem.cpp
void ULyraShopSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 加载商品数据（可以从 DataTable、DataAsset 或远程服务器加载）
    // 这里简化为硬编码示例
    LoadShopItemsFromAssets();
}

TArray<ULyraShopItemDefinition*> ULyraShopSubsystem::GetAvailableShopItems() const
{
    TArray<ULyraShopItemDefinition*> AvailableItems;
    
    for (const auto& Pair : ShopItems)
    {
        if (Pair.Value && Pair.Value->IsCurrentlyAvailable())
        {
            AvailableItems.Add(Pair.Value);
        }
    }

    return AvailableItems;
}

void ULyraShopSubsystem::RequestPurchase(APlayerController* PlayerController, FName ItemId)
{
    if (!PlayerController || !PlayerController->HasAuthority())
    {
        UE_LOG(LogLyra, Warning, TEXT("RequestPurchase called on client!"));
        return;
    }

    APlayerState* PlayerState = PlayerController->GetPlayerState<APlayerState>();
    if (!PlayerState)
        return;

    ULyraShopItemDefinition* ItemDef = GetShopItemById(ItemId);
    if (!ItemDef)
    {
        UE_LOG(LogLyra, Warning, TEXT("Shop item not found: %s"), *ItemId.ToString());
        return;
    }

    // 验证购买条件
    FText ErrorMessage;
    if (!ValidatePurchase(PlayerState, ItemDef, ErrorMessage))
    {
        // 通知客户端购买失败
        UE_LOG(LogLyra, Warning, TEXT("Purchase validation failed: %s"), *ErrorMessage.ToString());
        return;
    }

    // 执行购买
    ExecutePurchase(PlayerState, ItemDef);
}

bool ULyraShopSubsystem::ValidatePurchase(APlayerState* PlayerState, 
                                           ULyraShopItemDefinition* ItemDef,
                                           FText& OutErrorMessage)
{
    // 1. 检查商品是否可用
    if (!ItemDef->IsCurrentlyAvailable())
    {
        OutErrorMessage = FText::FromString(TEXT("Item is not currently available"));
        return false;
    }

    // 2. 检查是否已拥有（对于一次性购买的商品）
    if (ULyraInventoryManagerComponent* InventoryComp = 
        PlayerState->FindComponentByClass<ULyraInventoryManagerComponent>())
    {
        // 这里需要实现拥有状态检查
        // if (InventoryComp->HasItem(ItemDef->ItemId))
        // {
        //     OutErrorMessage = FText::FromString(TEXT("Already owned"));
        //     return false;
        // }
    }

    // 3. 检查货币是否足够
    if (ULyraCurrencyComponent* CurrencyComp = 
        PlayerState->FindComponentByClass<ULyraCurrencyComponent>())
    {
        for (const FLyraCurrencyAmount& Price : ItemDef->Prices)
        {
            if (!CurrencyComp->HasEnoughCurrency(Price))
            {
                OutErrorMessage = FText::Format(
                    FText::FromString(TEXT("Not enough {0}")),
                    FText::AsNumber((int32)Price.CurrencyType));
                return false;
            }
        }
    }
    else
    {
        OutErrorMessage = FText::FromString(TEXT("Currency component not found"));
        return false;
    }

    return true;
}

void ULyraShopSubsystem::ExecutePurchase(APlayerState* PlayerState, 
                                          ULyraShopItemDefinition* ItemDef)
{
    // 1. 扣除货币
    if (ULyraCurrencyComponent* CurrencyComp = 
        PlayerState->FindComponentByClass<ULyraCurrencyComponent>())
    {
        for (const FLyraCurrencyAmount& Price : ItemDef->Prices)
        {
            if (!CurrencyComp->DeductCurrency(Price))
            {
                UE_LOG(LogLyra, Error, TEXT("Failed to deduct currency during purchase!"));
                // 这里应该有回滚机制
                return;
            }
        }
    }

    // 2. 授予商品
    GrantItem(PlayerState, ItemDef);

    // 3. 记录购买日志（用于数据分析和防作弊）
    UE_LOG(LogLyra, Log, TEXT("Player %s purchased item %s"), 
           *PlayerState->GetPlayerName(), *ItemDef->ItemId.ToString());
}

void ULyraShopSubsystem::GrantItem(APlayerState* PlayerState, 
                                    ULyraShopItemDefinition* ItemDef)
{
    switch (ItemDef->ItemType)
    {
    case ELyraShopItemType::Cosmetic:
        {
            // 授予外观部件
            if (AController* Controller = Cast<AController>(PlayerState->GetOwner()))
            {
                if (ULyraControllerComponent_CharacterParts* CosmeticComp = 
                    Controller->FindComponentByClass<ULyraControllerComponent_CharacterParts>())
                {
                    CosmeticComp->AddCharacterPart(ItemDef->CosmeticPart);
                }
            }
        }
        break;

    case ELyraShopItemType::InventoryItem:
        {
            // 授予库存物品
            if (APawn* Pawn = Cast<APawn>(PlayerState->GetPawn()))
            {
                if (ULyraInventoryManagerComponent* InventoryComp = 
                    Pawn->FindComponentByClass<ULyraInventoryManagerComponent>())
                {
                    InventoryComp->AddItemDefinition(ItemDef->InventoryItemDef);
                }
            }
        }
        break;

    default:
        UE_LOG(LogLyra, Warning, TEXT("Unknown item type"));
        break;
    }
}
```

## 与库存系统集成

### 库存物品作为外观来源

```cpp
// 外观库存片段
UCLASS()
class UInventoryFragment_CosmeticItem : public ULyraInventoryItemFragment
{
    GENERATED_BODY()

public:
    // 外观部件定义
    UPROPERTY(EditDefaultsOnly, Category=Cosmetic)
    FLyraCharacterPart CosmeticPart;

    // 是否自动装备
    UPROPERTY(EditDefaultsOnly, Category=Cosmetic)
    bool bAutoEquip = false;

    virtual void OnInstanceCreated(ULyraInventoryItemInstance* Instance) const override
    {
        Super::OnInstanceCreated(Instance);

        if (bAutoEquip)
        {
            // 自动装备外观
            EquipCosmetic(Instance);
        }
    }

    void EquipCosmetic(ULyraInventoryItemInstance* Instance) const
    {
        if (!Instance)
            return;

        // 获取拥有者的 Controller
        if (AActor* Owner = Instance->GetOwner())
        {
            if (APawn* Pawn = Cast<APawn>(Owner))
            {
                if (AController* Controller = Pawn->GetController())
                {
                    if (ULyraControllerComponent_CharacterParts* CosmeticComp = 
                        Controller->FindComponentByClass<ULyraControllerComponent_CharacterParts>())
                    {
                        CosmeticComp->AddCharacterPart(CosmeticPart);
                    }
                }
            }
        }
    }
};
```

### 外观装备系统

```cpp
// 外观装备槽
UENUM(BlueprintType)
enum class ELyraCosmeticSlot : uint8
{
    Head,
    Body,
    Weapon,
    Back,
    Emote1,
    Emote2,
    Emote3
};

// 外观装备配置
USTRUCT(BlueprintType)
struct FLyraCosmeticLoadout
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TMap<ELyraCosmeticSlot, FLyraCharacterPart> EquippedCosmetics;
};

// 外观装备管理器
UCLASS()
class ULyraCosmeticLoadoutManager : public UPlayerStateComponent
{
    GENERATED_BODY()

public:
    // 装备外观到槽位
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Cosmetics)
    void EquipCosmeticToSlot(ELyraCosmeticSlot Slot, const FLyraCharacterPart& Cosmetic)
    {
        if (!HasAuthority())
            return;

        // 卸载当前槽位的外观
        UnequipCosmeticFromSlot(Slot);

        // 装备新外观
        CurrentLoadout.EquippedCosmetics.Add(Slot, Cosmetic);

        // 应用到角色
        ApplyLoadout();
    }

    // 卸载槽位的外观
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Cosmetics)
    void UnequipCosmeticFromSlot(ELyraCosmeticSlot Slot)
    {
        if (!HasAuthority())
            return;

        if (const FLyraCharacterPart* ExistingCosmetic = 
            CurrentLoadout.EquippedCosmetics.Find(Slot))
        {
            // 从角色上移除
            if (AController* Controller = Cast<AController>(GetOwner()->GetOwner()))
            {
                if (ULyraControllerComponent_CharacterParts* CosmeticComp = 
                    Controller->FindComponentByClass<ULyraControllerComponent_CharacterParts>())
                {
                    CosmeticComp->RemoveCharacterPart(*ExistingCosmetic);
                }
            }

            CurrentLoadout.EquippedCosmetics.Remove(Slot);
        }
    }

    // 应用整套配置
    UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Cosmetics)
    void ApplyLoadout()
    {
        if (!HasAuthority())
            return;

        AController* Controller = Cast<AController>(GetOwner()->GetOwner());
        if (!Controller)
            return;

        ULyraControllerComponent_CharacterParts* CosmeticComp = 
            Controller->FindComponentByClass<ULyraControllerComponent_CharacterParts>();
        if (!CosmeticComp)
            return;

        // 清空当前外观
        CosmeticComp->RemoveAllCharacterParts();

        // 应用所有槽位的外观
        for (const auto& Pair : CurrentLoadout.EquippedCosmetics)
        {
            CosmeticComp->AddCharacterPart(Pair.Value);
        }
    }

    // 获取当前配置
    UFUNCTION(BlueprintCallable, Category=Cosmetics)
    FLyraCosmeticLoadout GetCurrentLoadout() const
    {
        return CurrentLoadout;
    }

protected:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    UFUNCTION()
    void OnRep_CurrentLoadout();

private:
    UPROPERTY(ReplicatedUsing=OnRep_CurrentLoadout)
    FLyraCosmeticLoadout CurrentLoadout;
};
```

## 外观数据持久化

### 存档结构

```cpp
// 外观存档数据
USTRUCT(BlueprintType)
struct FLyraCosmeticSaveData
{
    GENERATED_BODY()

    // 已拥有的外观 ID 列表
    UPROPERTY(SaveGame)
    TArray<FName> OwnedCosmeticIds;

    // 当前装备的外观配置
    UPROPERTY(SaveGame)
    FLyraCosmeticLoadout EquippedLoadout;

    // 货币余额
    UPROPERTY(SaveGame)
    FLyraCurrencyBalance CurrencyBalance;
};

// Lyra 存档
UCLASS()
class ULyraSaveGame : public USaveGame
{
    GENERATED_BODY()

public:
    UPROPERTY(SaveGame)
    FString PlayerName;

    UPROPERTY(SaveGame)
    FLyraCosmeticSaveData CosmeticData;

    // 其他存档数据...
};
```

### 存档管理器

```cpp
// 外观存档管理器
UCLASS()
class ULyraCosmeticSaveManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 保存外观数据
    UFUNCTION(BlueprintCallable, Category=Save)
    void SaveCosmeticData(APlayerState* PlayerState)
    {
        if (!PlayerState)
            return;

        ULyraSaveGame* SaveGame = Cast<ULyraSaveGame>(
            UGameplayStatics::CreateSaveGameObject(ULyraSaveGame::StaticClass()));
        
        if (!SaveGame)
            return;

        // 收集货币数据
        if (ULyraCurrencyComponent* CurrencyComp = 
            PlayerState->FindComponentByClass<ULyraCurrencyComponent>())
        {
            SaveGame->CosmeticData.CurrencyBalance = CurrencyComp->GetCurrencyBalance();
        }

        // 收集装备配置
        if (ULyraCosmeticLoadoutManager* LoadoutManager = 
            PlayerState->FindComponentByClass<ULyraCosmeticLoadoutManager>())
        {
            SaveGame->CosmeticData.EquippedLoadout = LoadoutManager->GetCurrentLoadout();
        }

        // 收集已拥有的外观列表
        // （需要从库存系统或单独的拥有状态系统获取）

        // 异步保存到磁盘
        FString SlotName = FString::Printf(TEXT("CosmeticSave_%s"), *PlayerState->GetUniqueId().ToString());
        UGameplayStatics::SaveGameToSlot(SaveGame, SlotName, 0);

        UE_LOG(LogLyra, Log, TEXT("Cosmetic data saved for player %s"), *PlayerState->GetPlayerName());
    }

    // 加载外观数据
    UFUNCTION(BlueprintCallable, Category=Save)
    void LoadCosmeticData(APlayerState* PlayerState)
    {
        if (!PlayerState)
            return;

        FString SlotName = FString::Printf(TEXT("CosmeticSave_%s"), *PlayerState->GetUniqueId().ToString());
        
        ULyraSaveGame* SaveGame = Cast<ULyraSaveGame>(
            UGameplayStatics::LoadGameFromSlot(SlotName, 0));

        if (!SaveGame)
        {
            UE_LOG(LogLyra, Warning, TEXT("No save data found for player %s"), *PlayerState->GetPlayerName());
            return;
        }

        // 恢复货币余额
        if (ULyraCurrencyComponent* CurrencyComp = 
            PlayerState->FindComponentByClass<ULyraCurrencyComponent>())
        {
            CurrencyComp->SetCurrencyBalance(SaveGame->CosmeticData.CurrencyBalance);
        }

        // 恢复装备配置
        if (ULyraCosmeticLoadoutManager* LoadoutManager = 
            PlayerState->FindComponentByClass<ULyraCosmeticLoadoutManager>())
        {
            // 等待 Controller 和 Pawn 准备好
            FTimerHandle TimerHandle;
            GetWorld()->GetTimerManager().SetTimer(TimerHandle, [this, LoadoutManager, SaveGame]()
            {
                LoadoutManager->SetCurrentLoadout(SaveGame->CosmeticData.EquippedLoadout);
                LoadoutManager->ApplyLoadout();
            }, 0.5f, false);
        }

        UE_LOG(LogLyra, Log, TEXT("Cosmetic data loaded for player %s"), *PlayerState->GetPlayerName());
    }
};
```

### 云存档同步

对于多平台游戏，需要实现云存档同步：

```cpp
// 云存档接口
UCLASS()
class ULyraCloudSaveInterface : public UObject
{
    GENERATED_BODY()

public:
    // 上传存档到云端
    UFUNCTION(BlueprintCallable, Category=CloudSave)
    void UploadSaveToCloud(const FString& PlayerId, const FLyraCosmeticSaveData& SaveData)
    {
        // 序列化存档数据
        FString JsonString;
        FJsonObjectConverter::UStructToJsonObjectString(SaveData, JsonString);

        // 发送 HTTP 请求到后端服务器
        TSharedRef<IHttpRequest, ESPMode::ThreadSafe> Request = 
            FHttpModule::Get().CreateRequest();
        
        Request->SetURL(TEXT("https://your-backend.com/api/save/upload"));
        Request->SetVerb(TEXT("POST"));
        Request->SetHeader(TEXT("Content-Type"), TEXT("application/json"));
        Request->SetContentAsString(JsonString);
        
        Request->OnProcessRequestComplete().BindUObject(
            this, &ULyraCloudSaveInterface::OnUploadComplete);
        
        Request->ProcessRequest();
    }

    // 从云端下载存档
    UFUNCTION(BlueprintCallable, Category=CloudSave)
    void DownloadSaveFromCloud(const FString& PlayerId)
    {
        TSharedRef<IHttpRequest, ESPMode::ThreadSafe> Request = 
            FHttpModule::Get().CreateRequest();
        
        Request->SetURL(FString::Printf(TEXT("https://your-backend.com/api/save/download?playerId=%s"), *PlayerId));
        Request->SetVerb(TEXT("GET"));
        
        Request->OnProcessRequestComplete().BindUObject(
            this, &ULyraCloudSaveInterface::OnDownloadComplete);
        
        Request->ProcessRequest();
    }

private:
    void OnUploadComplete(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bWasSuccessful)
    {
        if (bWasSuccessful && Response->GetResponseCode() == 200)
        {
            UE_LOG(LogLyra, Log, TEXT("Cloud save uploaded successfully"));
        }
        else
        {
            UE_LOG(LogLyra, Error, TEXT("Failed to upload cloud save"));
        }
    }

    void OnDownloadComplete(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bWasSuccessful)
    {
        if (bWasSuccessful && Response->GetResponseCode() == 200)
        {
            FString JsonString = Response->GetContentAsString();
            
            FLyraCosmeticSaveData SaveData;
            if (FJsonObjectConverter::JsonObjectStringToUStruct(JsonString, &SaveData))
            {
                UE_LOG(LogLyra, Log, TEXT("Cloud save downloaded successfully"));
                // 应用存档数据
                OnCloudSaveDownloaded.Broadcast(SaveData);
            }
        }
        else
        {
            UE_LOG(LogLyra, Error, TEXT("Failed to download cloud save"));
        }
    }

public:
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(
        FOnCloudSaveDownloaded, 
        const FLyraCosmeticSaveData&, SaveData);

    UPROPERTY(BlueprintAssignable)
    FOnCloudSaveDownloaded OnCloudSaveDownloaded;
};
```

## 网络同步策略

### 同步优化

外观系统的网络同步需要在带宽和延迟之间取得平衡：

**1. FastArraySerializer 增量复制**

```cpp
// 已在 FLyraCharacterPartList 中实现
// 优势：
// - 只同步变化的部件，而非整个列表
// - 自动处理添加、删除、修改
// - 减少带宽消耗
```

**2. 延迟加载（Lazy Loading）**

```cpp
// 外观 Actor 延迟生成
bool FLyraCharacterPartList::SpawnActorForEntry(FLyraAppliedCharacterPartEntry& Entry)
{
    // Dedicated Server 不生成外观 Actor
    if (OwnerComponent->IsNetMode(NM_DedicatedServer))
        return false;

    // 只在本地和可见的客户端生成
    // ...
}
```

**3. 相关性过滤（Relevancy Filtering）**

```cpp
// 在 Character 类中重写相关性检查
bool ALyraCharacter::IsNetRelevantFor(
    const AActor* RealViewer,
    const AActor* ViewTarget,
    const FVector& SrcLocation) const
{
    // 调用父类检查
    if (!Super::IsNetRelevantFor(RealViewer, ViewTarget, SrcLocation))
        return false;

    // 对于距离很远的角色，可以跳过外观同步
    if (const APawn* ViewerPawn = Cast<APawn>(ViewTarget))
    {
        float DistanceSq = FVector::DistSquared(GetActorLocation(), ViewerPawn->GetActorLocation());
        
        // 超过 5000 单位，不同步外观细节
        if (DistanceSq > 5000.0f * 5000.0f)
        {
            // 这里可以设置一个标志，让外观组件跳过复制
            return false;
        }
    }

    return true;
}
```

**4. 条件复制（Conditional Replication）**

```cpp
// 外观组件可以根据条件决定是否复制
void ULyraPawnComponent_CharacterParts::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 只复制给拥有者和观察者
    DOREPLIFETIME_CONDITION(ThisClass, CharacterPartList, COND_SkipOwner);
}
```

### 预测与校正

虽然外观系统通常不需要客户端预测，但在某些情况下可以提升响应性：

```cpp
// 乐观预测：客户端立即应用外观，等待服务器确认
void ULyraCosmeticLoadoutManager::EquipCosmeticToSlot_Client(
    ELyraCosmeticSlot Slot, 
    const FLyraCharacterPart& Cosmetic)
{
    // 1. 客户端立即应用（乐观预测）
    OptimisticLoadout.EquippedCosmetics.Add(Slot, Cosmetic);
    ApplyLoadoutOptimistic();

    // 2. 请求服务器验证
    ServerEquipCosmeticToSlot(Slot, Cosmetic);
}

void ULyraCosmeticLoadoutManager::ServerEquipCosmeticToSlot_Implementation(
    ELyraCosmeticSlot Slot, 
    const FLyraCharacterPart& Cosmetic)
{
    // 服务器验证并应用
    if (ValidateCosmetic(Cosmetic))
    {
        EquipCosmeticToSlot(Slot, Cosmetic);
    }
    else
    {
        // 验证失败，通知客户端回滚
        ClientRollbackCosmeticChange(Slot);
    }
}

void ULyraCosmeticLoadoutManager::ClientRollbackCosmeticChange_Implementation(
    ELyraCosmeticSlot Slot)
{
    // 客户端回滚到服务器确认的状态
    OptimisticLoadout = CurrentLoadout;
    ApplyLoadoutOptimistic();
}
```

## 实战案例 1：角色外观定制系统

### 需求分析

实现一个完整的角色外观定制系统，包括：
- 多种外观类型（头部、身体、武器）
- 预览功能
- 颜色定制
- 保存和加载

### 实现步骤

#### 1. 创建外观定制 UI

```cpp
// 外观定制 UI Widget
UCLASS()
class ULyraCosmeticCustomizationWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override
    {
        Super::NativeConstruct();

        // 加载可用的外观选项
        LoadAvailableCosmetics();

        // 绑定按钮事件
        if (ApplyButton)
        {
            ApplyButton->OnClicked.AddDynamic(this, &ThisClass::OnApplyClicked);
        }
        if (ResetButton)
        {
            ResetButton->OnClicked.AddDynamic(this, &ThisClass::OnResetClicked);
        }
    }

protected:
    // UI 组件
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonButtonBase> ApplyButton;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonButtonBase> ResetButton;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<ULyraCosmeticSlotWidget> HeadSlotWidget;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<ULyraCosmeticSlotWidget> BodySlotWidget;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<ULyraCosmeticSlotWidget> WeaponSlotWidget;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<ULyraCharacterPreviewWidget> PreviewWidget;

    // 加载可用外观
    void LoadAvailableCosmetics()
    {
        if (ULyraShopSubsystem* ShopSubsystem = 
            GetGameInstance()->GetSubsystem<ULyraShopSubsystem>())
        {
            // 获取已拥有的外观
            TArray<ULyraShopItemDefinition*> OwnedCosmetics = GetOwnedCosmetics();

            // 填充槽位选项
            if (HeadSlotWidget)
            {
                TArray<ULyraShopItemDefinition*> HeadCosmetics = 
                    FilterCosmeticsBySlot(OwnedCosmetics, ELyraCosmeticSlot::Head);
                HeadSlotWidget->SetAvailableCosmetics(HeadCosmetics);
            }

            if (BodySlotWidget)
            {
                TArray<ULyraShopItemDefinition*> BodyCosmetics = 
                    FilterCosmeticsBySlot(OwnedCosmetics, ELyraCosmeticSlot::Body);
                BodySlotWidget->SetAvailableCosmetics(BodyCosmetics);
            }

            if (WeaponSlotWidget)
            {
                TArray<ULyraShopItemDefinition*> WeaponCosmetics = 
                    FilterCosmeticsBySlot(OwnedCosmetics, ELyraCosmeticSlot::Weapon);
                WeaponSlotWidget->SetAvailableCosmetics(WeaponCosmetics);
            }
        }
    }

    // 应用外观
    UFUNCTION()
    void OnApplyClicked()
    {
        APlayerController* PC = GetOwningPlayer();
        if (!PC)
            return;

        APlayerState* PS = PC->GetPlayerState<APlayerState>();
        if (!PS)
            return;

        ULyraCosmeticLoadoutManager* LoadoutManager = 
            PS->FindComponentByClass<ULyraCosmeticLoadoutManager>();
        if (!LoadoutManager)
            return;

        // 收集所有槽位的选择
        if (HeadSlotWidget && HeadSlotWidget->GetSelectedCosmetic())
        {
            LoadoutManager->EquipCosmeticToSlot(
                ELyraCosmeticSlot::Head, 
                HeadSlotWidget->GetSelectedCosmetic()->CosmeticPart);
        }

        if (BodySlotWidget && BodySlotWidget->GetSelectedCosmetic())
        {
            LoadoutManager->EquipCosmeticToSlot(
                ELyraCosmeticSlot::Body, 
                BodySlotWidget->GetSelectedCosmetic()->CosmeticPart);
        }

        if (WeaponSlotWidget && WeaponSlotWidget->GetSelectedCosmetic())
        {
            LoadoutManager->EquipCosmeticToSlot(
                ELyraCosmeticSlot::Weapon, 
                WeaponSlotWidget->GetSelectedCosmetic()->CosmeticPart);
        }

        // 保存配置
        if (ULyraCosmeticSaveManager* SaveManager = 
            GetGameInstance()->GetSubsystem<ULyraCosmeticSaveManager>())
        {
            SaveManager->SaveCosmeticData(PS);
        }

        // 关闭 UI
        DeactivateWidget();
    }

    // 重置为默认外观
    UFUNCTION()
    void OnResetClicked()
    {
        HeadSlotWidget->SelectDefaultCosmetic();
        BodySlotWidget->SelectDefaultCosmetic();
        WeaponSlotWidget->SelectDefaultCosmetic();

        UpdatePreview();
    }

    // 更新预览
    void UpdatePreview()
    {
        if (!PreviewWidget)
            return;

        // 创建预览配置
        FLyraCosmeticLoadout PreviewLoadout;
        
        if (HeadSlotWidget && HeadSlotWidget->GetSelectedCosmetic())
        {
            PreviewLoadout.EquippedCosmetics.Add(
                ELyraCosmeticSlot::Head, 
                HeadSlotWidget->GetSelectedCosmetic()->CosmeticPart);
        }

        if (BodySlotWidget && BodySlotWidget->GetSelectedCosmetic())
        {
            PreviewLoadout.EquippedCosmetics.Add(
                ELyraCosmeticSlot::Body, 
                BodySlotWidget->GetSelectedCosmetic()->CosmeticPart);
        }

        if (WeaponSlotWidget && WeaponSlotWidget->GetSelectedCosmetic())
        {
            PreviewLoadout.EquippedCosmetics.Add(
                ELyraCosmeticSlot::Weapon, 
                WeaponSlotWidget->GetSelectedCosmetic()->CosmeticPart);
        }

        // 应用到预览角色
        PreviewWidget->ApplyLoadout(PreviewLoadout);
    }

private:
    TArray<ULyraShopItemDefinition*> GetOwnedCosmetics()
    {
        // 从库存或拥有状态系统获取
        // 这里简化为返回所有可用外观
        if (ULyraShopSubsystem* ShopSubsystem = 
            GetGameInstance()->GetSubsystem<ULyraShopSubsystem>())
        {
            return ShopSubsystem->GetAllShopItems();
        }
        return TArray<ULyraShopItemDefinition*>();
    }

    TArray<ULyraShopItemDefinition*> FilterCosmeticsBySlot(
        const TArray<ULyraShopItemDefinition*>& AllCosmetics,
        ELyraCosmeticSlot Slot)
    {
        TArray<ULyraShopItemDefinition*> Filtered;
        
        for (ULyraShopItemDefinition* Cosmetic : AllCosmetics)
        {
            if (Cosmetic && Cosmetic->ItemType == ELyraShopItemType::Cosmetic)
            {
                // 根据槽位过滤（需要在 ShopItemDefinition 中添加 Slot 字段）
                // if (Cosmetic->CosmeticSlot == Slot)
                // {
                    Filtered.Add(Cosmetic);
                // }
            }
        }

        return Filtered;
    }
};
```

#### 2. 角色预览系统

```cpp
// 角色预览 Widget
UCLASS()
class ULyraCharacterPreviewWidget : public UCommonUserWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override
    {
        Super::NativeConstruct();

        // 创建预览场景
        SetupPreviewScene();
    }

    virtual void NativeDestruct() override
    {
        CleanupPreviewScene();
        Super::NativeDestruct();
    }

    // 应用外观配置到预览角色
    void ApplyLoadout(const FLyraCosmeticLoadout& Loadout)
    {
        if (!PreviewPawn)
            return;

        ULyraPawnComponent_CharacterParts* CosmeticComp = 
            PreviewPawn->FindComponentByClass<ULyraPawnComponent_CharacterParts>();
        if (!CosmeticComp)
            return;

        // 清空当前外观
        CosmeticComp->RemoveAllCharacterParts();

        // 应用新外观
        for (const auto& Pair : Loadout.EquippedCosmetics)
        {
            CosmeticComp->AddCharacterPart(Pair.Value);
        }
    }

    // 旋转预览角色
    UFUNCTION(BlueprintCallable)
    void RotatePreview(float DeltaYaw)
    {
        if (PreviewPawn)
        {
            FRotator NewRotation = PreviewPawn->GetActorRotation();
            NewRotation.Yaw += DeltaYaw;
            PreviewPawn->SetActorRotation(NewRotation);
        }
    }

protected:
    void SetupPreviewScene()
    {
        // 创建预览世界
        PreviewWorld = UWorld::CreateWorld(
            EWorldType::Preview,
            false,
            TEXT("CosmeticPreviewWorld"));

        if (!PreviewWorld)
            return;

        PreviewWorld->InitializeNewWorld();

        // 生成预览 Pawn
        FActorSpawnParameters SpawnParams;
        SpawnParams.SpawnCollisionHandlingOverride = 
            ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

        PreviewPawn = PreviewWorld->SpawnActor<ALyraCharacter>(
            PreviewCharacterClass, 
            FVector::ZeroVector, 
            FRotator::ZeroRotator, 
            SpawnParams);

        if (PreviewPawn)
        {
            // 添加外观组件
            ULyraPawnComponent_CharacterParts* CosmeticComp = 
                NewObject<ULyraPawnComponent_CharacterParts>(
                    PreviewPawn, 
                    ULyraPawnComponent_CharacterParts::StaticClass());
            CosmeticComp->RegisterComponent();
        }

        // 设置场景捕获组件
        SetupSceneCapture();
    }

    void SetupSceneCapture()
    {
        if (!PreviewWorld)
            return;

        // 创建场景捕获 Actor
        SceneCaptureActor = PreviewWorld->SpawnActor<AActor>(AActor::StaticClass());
        if (!SceneCaptureActor)
            return;

        // 添加场景捕获组件
        SceneCaptureComponent = NewObject<USceneCaptureComponent2D>(
            SceneCaptureActor, 
            USceneCaptureComponent2D::StaticClass());
        
        SceneCaptureComponent->RegisterComponent();
        SceneCaptureComponent->SetRelativeLocation(FVector(-200, 0, 50));
        SceneCaptureComponent->SetRelativeRotation(FRotator(0, 0, 0));

        // 创建渲染目标
        RenderTarget = NewObject<UTextureRenderTarget2D>(
            this, 
            UTextureRenderTarget2D::StaticClass());
        
        RenderTarget->InitAutoFormat(512, 512);
        RenderTarget->UpdateResourceImmediate(true);

        SceneCaptureComponent->TextureTarget = RenderTarget;

        // 绑定到 UI Image
        if (PreviewImage)
        {
            PreviewImage->SetBrushFromTexture(RenderTarget);
        }
    }

    void CleanupPreviewScene()
    {
        if (PreviewPawn)
        {
            PreviewPawn->Destroy();
            PreviewPawn = nullptr;
        }

        if (SceneCaptureActor)
        {
            SceneCaptureActor->Destroy();
            SceneCaptureActor = nullptr;
        }

        if (PreviewWorld)
        {
            PreviewWorld->DestroyWorld(false);
            PreviewWorld = nullptr;
        }
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category=Preview)
    TSubclassOf<ALyraCharacter> PreviewCharacterClass;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UImage> PreviewImage;

private:
    UPROPERTY(Transient)
    TObjectPtr<UWorld> PreviewWorld;

    UPROPERTY(Transient)
    TObjectPtr<ALyraCharacter> PreviewPawn;

    UPROPERTY(Transient)
    TObjectPtr<AActor> SceneCaptureActor;

    UPROPERTY(Transient)
    TObjectPtr<USceneCaptureComponent2D> SceneCaptureComponent;

    UPROPERTY(Transient)
    TObjectPtr<UTextureRenderTarget2D> RenderTarget;
};
```

## 实战案例 2：战利品箱和开箱系统

### 战利品箱定义

```cpp
// 战利品箱物品
UCLASS()
class ULyraLootBoxItemDefinition : public ULyraInventoryItemDefinition
{
    GENERATED_BODY()

public:
    // 战利品表
    UPROPERTY(EditDefaultsOnly, Category=LootBox)
    TObjectPtr<UDataTable> LootTable;

    // 开箱动画
    UPROPERTY(EditDefaultsOnly, Category=LootBox)
    TSubclassOf<UUserWidget> OpeningAnimationWidget;

    // 是否需要钥匙
    UPROPERTY(EditDefaultsOnly, Category=LootBox)
    bool bRequiresKey = false;

    // 钥匙物品定义
    UPROPERTY(EditDefaultsOnly, Category=LootBox, meta=(EditCondition="bRequiresKey"))
    TSubclassOf<ULyraInventoryItemDefinition> KeyItemDef;
};

// 战利品条目
USTRUCT(BlueprintType)
struct FLyraLootEntry : public FTableRowBase
{
    GENERATED_BODY()

    // 外观物品
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TObjectPtr<ULyraShopItemDefinition> CosmeticItem;

    // 掉落权重
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    float Weight = 1.0f;

    // 稀有度
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    ELyraItemRarity Rarity = ELyraItemRarity::Common;

    // 是否为保底奖励
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    bool bIsGuaranteedDrop = false;
};
```

### 开箱系统实现

```cpp
// 战利品箱管理器
UCLASS()
class ULyraLootBoxManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 打开战利品箱
    UFUNCTION(BlueprintCallable, Category=LootBox)
    void OpenLootBox(APlayerController* PlayerController, 
                     ULyraLootBoxItemDefinition* LootBoxDef)
    {
        if (!PlayerController || !LootBoxDef)
            return;

        APlayerState* PS = PlayerController->GetPlayerState<APlayerState>();
        if (!PS)
            return;

        // 验证是否拥有战利品箱
        ULyraInventoryManagerComponent* InventoryComp = 
            PS->FindComponentByClass<ULyraInventoryManagerComponent>();
        if (!InventoryComp || !InventoryComp->HasItem(LootBoxDef))
        {
            UE_LOG(LogLyra, Warning, TEXT("Player doesn't own this loot box"));
            return;
        }

        // 检查是否需要钥匙
        if (LootBoxDef->bRequiresKey)
        {
            if (!InventoryComp->HasItem(LootBoxDef->KeyItemDef))
            {
                UE_LOG(LogLyra, Warning, TEXT("Player doesn't have the required key"));
                return;
            }

            // 消耗钥匙
            InventoryComp->RemoveItem(LootBoxDef->KeyItemDef, 1);
        }

        // 消耗战利品箱
        InventoryComp->RemoveItem(LootBoxDef, 1);

        // 抽取奖励
        TArray<ULyraShopItemDefinition*> Rewards = RollLootBox(LootBoxDef);

        // 授予奖励
        for (ULyraShopItemDefinition* Reward : Rewards)
        {
            GrantCosmeticItem(PS, Reward);
        }

        // 显示开箱动画
        ShowLootBoxAnimation(PlayerController, LootBoxDef, Rewards);
    }

protected:
    // 抽取战利品
    TArray<ULyraShopItemDefinition*> RollLootBox(ULyraLootBoxItemDefinition* LootBoxDef)
    {
        TArray<ULyraShopItemDefinition*> Rewards;

        if (!LootBoxDef || !LootBoxDef->LootTable)
            return Rewards;

        // 加载战利品表
        TArray<FLyraLootEntry*> LootEntries;
        LootBoxDef->LootTable->GetAllRows<FLyraLootEntry>(TEXT("RollLootBox"), LootEntries);

        if (LootEntries.Num() == 0)
            return Rewards;

        // 计算总权重
        float TotalWeight = 0.0f;
        for (const FLyraLootEntry* Entry : LootEntries)
        {
            if (Entry && Entry->CosmeticItem)
            {
                TotalWeight += Entry->Weight;
            }
        }

        // 添加保底奖励
        for (const FLyraLootEntry* Entry : LootEntries)
        {
            if (Entry && Entry->bIsGuaranteedDrop && Entry->CosmeticItem)
            {
                Rewards.Add(Entry->CosmeticItem);
            }
        }

        // 随机抽取奖励
        int32 NumRolls = 3; // 每个箱子抽 3 个奖励
        for (int32 i = 0; i < NumRolls; ++i)
        {
            float RandomValue = FMath::FRandRange(0.0f, TotalWeight);
            float CurrentWeight = 0.0f;

            for (const FLyraLootEntry* Entry : LootEntries)
            {
                if (!Entry || !Entry->CosmeticItem)
                    continue;

                CurrentWeight += Entry->Weight;
                if (RandomValue <= CurrentWeight)
                {
                    Rewards.Add(Entry->CosmeticItem);
                    break;
                }
            }
        }

        return Rewards;
    }

    // 授予外观物品
    void GrantCosmeticItem(APlayerState* PS, ULyraShopItemDefinition* CosmeticItem)
    {
        if (!PS || !CosmeticItem)
            return;

        // 添加到拥有列表
        ULyraInventoryManagerComponent* InventoryComp = 
            PS->FindComponentByClass<ULyraInventoryManagerComponent>();
        if (InventoryComp)
        {
            // 将外观添加到库存
            InventoryComp->AddItemDefinition(CosmeticItem->InventoryItemDef);
        }

        UE_LOG(LogLyra, Log, TEXT("Granted cosmetic: %s"), *CosmeticItem->DisplayName.ToString());
    }

    // 显示开箱动画
    void ShowLootBoxAnimation(APlayerController* PC, 
                               ULyraLootBoxItemDefinition* LootBoxDef,
                               const TArray<ULyraShopItemDefinition*>& Rewards)
    {
        if (!PC || !LootBoxDef || !LootBoxDef->OpeningAnimationWidget)
            return;

        // 创建开箱动画 Widget
        UUserWidget* AnimWidget = CreateWidget<UUserWidget>(
            PC, 
            LootBoxDef->OpeningAnimationWidget);

        if (!AnimWidget)
            return;

        // 传递奖励数据到 Widget
        if (ULyraLootBoxAnimationWidget* LootAnimWidget = 
            Cast<ULyraLootBoxAnimationWidget>(AnimWidget))
        {
            LootAnimWidget->SetRewards(Rewards);
        }

        AnimWidget->AddToViewport(100); // 高 Z-Order
    }
};
```

### 开箱动画 UI

```cpp
// 开箱动画 Widget
UCLASS()
class ULyraLootBoxAnimationWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()

public:
    // 设置奖励列表
    void SetRewards(const TArray<ULyraShopItemDefinition*>& InRewards)
    {
        Rewards = InRewards;
        StartAnimation();
    }

protected:
    virtual void NativeConstruct() override
    {
        Super::NativeConstruct();

        // 绑定跳过按钮
        if (SkipButton)
        {
            SkipButton->OnClicked.AddDynamic(this, &ThisClass::OnSkipClicked);
        }
    }

    // 开始动画
    void StartAnimation()
    {
        if (Rewards.Num() == 0)
        {
            CloseWidget();
            return;
        }

        CurrentRewardIndex = 0;
        ShowNextReward();
    }

    // 显示下一个奖励
    void ShowNextReward()
    {
        if (CurrentRewardIndex >= Rewards.Num())
        {
            // 所有奖励已显示，延迟后关闭
            FTimerHandle TimerHandle;
            GetWorld()->GetTimerManager().SetTimer(
                TimerHandle, 
                this, 
                &ThisClass::CloseWidget, 
                2.0f, 
                false);
            return;
        }

        ULyraShopItemDefinition* CurrentReward = Rewards[CurrentRewardIndex];
        if (!CurrentReward)
        {
            CurrentRewardIndex++;
            ShowNextReward();
            return;
        }

        // 更新 UI 显示
        if (RewardNameText)
        {
            RewardNameText->SetText(CurrentReward->DisplayName);
        }

        if (RewardImage)
        {
            RewardImage->SetBrushFromTexture(CurrentReward->Icon);
        }

        // 播放揭示动画
        if (RevealAnimation)
        {
            PlayAnimation(RevealAnimation);
        }

        // 播放音效
        PlayRevealSound(CurrentReward);

        // 延迟显示下一个奖励
        FTimerHandle TimerHandle;
        GetWorld()->GetTimerManager().SetTimer(
            TimerHandle, 
            this, 
            &ThisClass::OnRevealComplete, 
            1.5f, 
            false);
    }

    void OnRevealComplete()
    {
        CurrentRewardIndex++;
        ShowNextReward();
    }

    // 播放揭示音效
    void PlayRevealSound(ULyraShopItemDefinition* Reward)
    {
        if (!Reward)
            return;

        // 根据稀有度播放不同音效
        USoundBase* SoundToPlay = nullptr;
        switch (Reward->Rarity)
        {
        case ELyraItemRarity::Common:
            SoundToPlay = CommonRevealSound;
            break;
        case ELyraItemRarity::Rare:
            SoundToPlay = RareRevealSound;
            break;
        case ELyraItemRarity::Epic:
            SoundToPlay = EpicRevealSound;
            break;
        case ELyraItemRarity::Legendary:
            SoundToPlay = LegendaryRevealSound;
            break;
        }

        if (SoundToPlay)
        {
            UGameplayStatics::PlaySound2D(this, SoundToPlay);
        }
    }

    UFUNCTION()
    void OnSkipClicked()
    {
        // 跳过动画，直接关闭
        CloseWidget();
    }

    void CloseWidget()
    {
        DeactivateWidget();
        RemoveFromParent();
    }

protected:
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonButtonBase> SkipButton;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonTextBlock> RewardNameText;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UImage> RewardImage;

    UPROPERTY(Transient, meta=(BindWidgetAnim))
    TObjectPtr<UWidgetAnimation> RevealAnimation;

    UPROPERTY(EditDefaultsOnly, Category=Audio)
    TObjectPtr<USoundBase> CommonRevealSound;

    UPROPERTY(EditDefaultsOnly, Category=Audio)
    TObjectPtr<USoundBase> RareRevealSound;

    UPROPERTY(EditDefaultsOnly, Category=Audio)
    TObjectPtr<USoundBase> EpicRevealSound;

    UPROPERTY(EditDefaultsOnly, Category=Audio)
    TObjectPtr<USoundBase> LegendaryRevealSound;

private:
    TArray<ULyraShopItemDefinition*> Rewards;
    int32 CurrentRewardIndex = 0;
};
```

## UI 集成

### 商店 UI

```cpp
// 商店 UI Widget
UCLASS()
class ULyraShopWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override
    {
        Super::NativeConstruct();

        // 加载商品列表
        RefreshShopItems();

        // 订阅货币变化事件
        SubscribeToCurrencyChanges();
    }

protected:
    void RefreshShopItems()
    {
        if (!ItemListView)
            return;

        ULyraShopSubsystem* ShopSubsystem = 
            GetGameInstance()->GetSubsystem<ULyraShopSubsystem>();
        if (!ShopSubsystem)
            return;

        TArray<ULyraShopItemDefinition*> AvailableItems = 
            ShopSubsystem->GetAvailableShopItems();

        // 清空现有列表
        ItemListView->ClearListItems();

        // 添加商品到列表
        for (ULyraShopItemDefinition* Item : AvailableItems)
        {
            if (Item)
            {
                // 创建列表项数据对象
                ULyraShopItemListEntry* EntryData = 
                    NewObject<ULyraShopItemListEntry>(this);
                EntryData->ItemDefinition = Item;

                ItemListView->AddItem(EntryData);
            }
        }
    }

    void SubscribeToCurrencyChanges()
    {
        APlayerController* PC = GetOwningPlayer();
        if (!PC)
            return;

        APlayerState* PS = PC->GetPlayerState<APlayerState>();
        if (!PS)
            return;

        ULyraCurrencyComponent* CurrencyComp = 
            PS->FindComponentByClass<ULyraCurrencyComponent>();
        if (CurrencyComp)
        {
            CurrencyComp->OnCurrencyChanged.AddDynamic(
                this, 
                &ThisClass::OnCurrencyChanged);

            // 初始化货币显示
            UpdateCurrencyDisplay();
        }
    }

    UFUNCTION()
    void OnCurrencyChanged(ELyraCurrencyType CurrencyType, 
                           int32 OldAmount, 
                           int32 NewAmount)
    {
        UpdateCurrencyDisplay();
    }

    void UpdateCurrencyDisplay()
    {
        APlayerController* PC = GetOwningPlayer();
        if (!PC)
            return;

        APlayerState* PS = PC->GetPlayerState<APlayerState>();
        if (!PS)
            return;

        ULyraCurrencyComponent* CurrencyComp = 
            PS->FindComponentByClass<ULyraCurrencyComponent>();
        if (!CurrencyComp)
            return;

        // 更新各种货币的显示
        if (SoftCurrencyText)
        {
            int32 SoftCurrency = CurrencyComp->GetCurrencyBalance(
                ELyraCurrencyType::SoftCurrency);
            SoftCurrencyText->SetText(FText::AsNumber(SoftCurrency));
        }

        if (HardCurrencyText)
        {
            int32 HardCurrency = CurrencyComp->GetCurrencyBalance(
                ELyraCurrencyType::HardCurrency);
            HardCurrencyText->SetText(FText::AsNumber(HardCurrency));
        }
    }

protected:
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UListView> ItemListView;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonTextBlock> SoftCurrencyText;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonTextBlock> HardCurrencyText;
};

// 商品列表项
UCLASS()
class ULyraShopItemListEntry : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY()
    TObjectPtr<ULyraShopItemDefinition> ItemDefinition;
};

// 商品列表项 Widget
UCLASS()
class ULyraShopItemWidget : public UCommonUserWidget, public IUserObjectListEntry
{
    GENERATED_BODY()

public:
    virtual void NativeOnListItemObjectSet(UObject* ListItemObject) override
    {
        ULyraShopItemListEntry* Entry = Cast<ULyraShopItemListEntry>(ListItemObject);
        if (!Entry || !Entry->ItemDefinition)
            return;

        ItemDefinition = Entry->ItemDefinition;

        // 更新显示
        if (ItemNameText)
        {
            ItemNameText->SetText(ItemDefinition->DisplayName);
        }

        if (ItemDescriptionText)
        {
            ItemDescriptionText->SetText(ItemDefinition->Description);
        }

        if (ItemIconImage)
        {
            ItemIconImage->SetBrushFromTexture(ItemDefinition->Icon);
        }

        if (PriceText && ItemDefinition->Prices.Num() > 0)
        {
            const FLyraCurrencyAmount& Price = ItemDefinition->Prices[0];
            PriceText->SetText(FText::AsNumber(Price.Amount));
        }

        // 绑定购买按钮
        if (PurchaseButton)
        {
            PurchaseButton->OnClicked.AddDynamic(this, &ThisClass::OnPurchaseClicked);
        }
    }

protected:
    UFUNCTION()
    void OnPurchaseClicked()
    {
        if (!ItemDefinition)
            return;

        APlayerController* PC = GetOwningPlayer();
        if (!PC)
            return;

        ULyraShopSubsystem* ShopSubsystem = 
            GetGameInstance()->GetSubsystem<ULyraShopSubsystem>();
        if (ShopSubsystem)
        {
            ShopSubsystem->RequestPurchase(PC, ItemDefinition->ItemId);
        }
    }

protected:
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonTextBlock> ItemNameText;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonTextBlock> ItemDescriptionText;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UImage> ItemIconImage;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonTextBlock> PriceText;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonButtonBase> PurchaseButton;

private:
    UPROPERTY(Transient)
    TObjectPtr<ULyraShopItemDefinition> ItemDefinition;
};
```

## 性能优化

### 1. LOD 管理

```cpp
// 外观 LOD 管理器
UCLASS()
class ULyraCosmeticLODManager : public UActorComponent
{
    GENERATED_BODY()

public:
    virtual void TickComponent(float DeltaTime, 
                                ELevelTick TickType, 
                                FActorComponentTickFunction* ThisTickFunction) override
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

        // 根据距离调整 LOD
        UpdateCosmeticLOD();
    }

protected:
    void UpdateCosmeticLOD()
    {
        AActor* Owner = GetOwner();
        if (!Owner)
            return;

        // 获取本地玩家视角
        APlayerController* PC = GetWorld()->GetFirstPlayerController();
        if (!PC || !PC->PlayerCameraManager)
            return;

        FVector CameraLocation = PC->PlayerCameraManager->GetCameraLocation();
        float DistanceSq = FVector::DistSquared(Owner->GetActorLocation(), CameraLocation);

        // 根据距离设置 LOD
        ULyraPawnComponent_CharacterParts* CosmeticComp = 
            Owner->FindComponentByClass<ULyraPawnComponent_CharacterParts>();
        if (!CosmeticComp)
            return;

        for (AActor* PartActor : CosmeticComp->GetCharacterPartActors())
        {
            if (!PartActor)
                continue;

            TArray<UActorComponent*> MeshComponents;
            PartActor->GetComponents(USkeletalMeshComponent::StaticClass(), MeshComponents);

            for (UActorComponent* Comp : MeshComponents)
            {
                USkeletalMeshComponent* MeshComp = Cast<USkeletalMeshComponent>(Comp);
                if (MeshComp)
                {
                    // 距离阈值
                    if (DistanceSq > 10000.0f * 10000.0f) // 100米以上
                    {
                        MeshComp->SetForcedLOD(3); // 最低 LOD
                    }
                    else if (DistanceSq > 5000.0f * 5000.0f) // 50米以上
                    {
                        MeshComp->SetForcedLOD(2);
                    }
                    else if (DistanceSq > 2000.0f * 2000.0f) // 20米以上
                    {
                        MeshComp->SetForcedLOD(1);
                    }
                    else
                    {
                        MeshComp->SetForcedLOD(0); // 最高 LOD
                    }
                }
            }
        }
    }
};
```

### 2. 材质实例缓存

```cpp
// 材质实例缓存管理器
UCLASS()
class ULyraMaterialInstanceCacheManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 获取或创建材质实例
    UMaterialInstanceDynamic* GetOrCreateMaterialInstance(
        UMaterialInterface* BaseMaterial,
        UObject* Outer)
    {
        if (!BaseMaterial)
            return nullptr;

        // 检查缓存
        FString CacheKey = BaseMaterial->GetPathName();
        if (TObjectPtr<UMaterialInstanceDynamic>* CachedMID = MaterialInstanceCache.Find(CacheKey))
        {
            if (*CachedMID && !(*CachedMID)->IsPendingKill())
            {
                return *CachedMID;
            }
        }

        // 创建新实例
        UMaterialInstanceDynamic* NewMID = UMaterialInstanceDynamic::Create(
            BaseMaterial, 
            Outer);

        // 添加到缓存
        MaterialInstanceCache.Add(CacheKey, NewMID);

        return NewMID;
    }

    // 清理缓存
    UFUNCTION(BlueprintCallable, Category=Cache)
    void ClearCache()
    {
        MaterialInstanceCache.Empty();
    }

private:
    UPROPERTY(Transient)
    TMap<FString, TObjectPtr<UMaterialInstanceDynamic>> MaterialInstanceCache;
};
```

### 3. 异步加载

```cpp
// 异步加载外观资产
UCLASS()
class ULyraCosmeticAssetLoader : public UObject
{
    GENERATED_BODY()

public:
    // 异步加载外观部件
    void LoadCosmeticPartAsync(const FLyraCharacterPart& Part, 
                                TFunction<void(TSubclassOf<AActor>)> OnLoaded)
    {
        if (!Part.PartClass.IsNull())
        {
            // 如果已经加载，直接回调
            if (Part.PartClass.IsValid())
            {
                OnLoaded(Part.PartClass.Get());
                return;
            }

            // 异步加载
            FStreamableManager& StreamableManager = 
                UAssetManager::GetStreamableManager();

            StreamableManager.RequestAsyncLoad(
                Part.PartClass.ToSoftObjectPath(),
                [this, Part, OnLoaded]()
                {
                    if (Part.PartClass.IsValid())
                    {
                        OnLoaded(Part.PartClass.Get());
                    }
                    else
                    {
                        UE_LOG(LogLyra, Warning, TEXT("Failed to load cosmetic part"));
                    }
                });
        }
    }
};
```

## 总结

本章深入探讨了 Lyra 的外观系统与商店实现，涵盖了以下核心内容：

1. **外观系统架构**
   - Controller 和 Pawn 双层组件设计
   - Character Parts 模块化管理
   - FastArraySerializer 高效网络复制

2. **Character Parts 系统**
   - 多种外观部件类型（头部、身体、武器）
   - Gameplay Tags 驱动的身体网格选择
   - ChildActorComponent 动态生成机制

3. **材质动态替换**
   - 材质实例动态（MID）
   - 团队颜色自动应用
   - 材质参数集合优化

4. **商店系统**
   - 多货币系统设计
   - 商品定义与配置
   - 购买流程与验证
   - 与库存系统集成

5. **数据持久化**
   - 存档结构设计
   - 本地存档管理
   - 云存档同步

6. **网络同步优化**
   - 增量复制
   - 相关性过滤
   - 客户端预测

7. **实战案例**
   - 完整的角色外观定制系统
   - 战利品箱和开箱系统
   - UI 集成方案

8. **性能优化**
   - LOD 管理
   - 材质实例缓存
   - 异步资产加载

外观系统是现代游戏变现和玩家留存的重要手段。Lyra 提供的架构不仅功能完善，而且高度可扩展，适合各种类型的游戏项目。在实际开发中，建议根据项目需求进行针对性优化，平衡功能丰富度和性能开销。

## 下一步

- **第 27 章：赛季与战令系统** - 学习如何实现长期运营系统
- **第 28 章：社交系统集成** - 实现好友、队伍、公会等社交功能
- **第 29 章：数据分析与遥测** - 收集和分析玩家行为数据
