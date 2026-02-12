# UE5 Lyra 系列教程（五）：数据驱动设计与 Data Assets

> **作者**: lobsterchen  
> **创建时间**: 2025-02-12  
> **系列**: UE5 Lyra 深度解析  
> **难度**: ⭐⭐⭐ 中级  
> **预计阅读时间**: 20 分钟

---

## 📚 目录

- [为什么需要数据驱动？](#为什么需要数据驱动)
- [Primary Data Asset 详解](#primary-data-asset-详解)
- [Data Registry 系统](#data-registry-系统)
- [Gameplay Tags 最佳实践](#gameplay-tags-最佳实践)
- [实战：数据驱动的武器系统](#实战数据驱动的武器系统)
- [配置文件管理](#配置文件管理)

---

## 🤔 为什么需要数据驱动？

### 硬编码的噩梦

假设你要创建一把新武器，传统方式：

```cpp
// ❌ 硬编码方式
class ARifle : public AWeapon
{
public:
    ARifle()
    {
        Damage = 35.0f;
        FireRate = 0.1f;
        MagazineSize = 30;
        ReloadTime = 2.5f;
        ProjectileSpeed = 10000.0f;
        // ... 100 行配置代码
    }
};
```

**问题显而易见**：
- 🚫 **策划无法修改**：改数值 = 重新编译 C++
- 🚫 **难以平衡**：调整武器数值需要程序员介入
- 🚫 **无法热更新**：每次调整都要重新打包
- 🚫 **复用困难**：想复制一把武器？复制粘贴整个类

### 数据驱动的优势

```cpp
// ✅ 数据驱动方式
UCLASS()
class UWeaponData : public UPrimaryDataAsset
{
    UPROPERTY(EditDefaultsOnly)
    float Damage = 35.0f;
    
    UPROPERTY(EditDefaultsOnly)
    float FireRate = 0.1f;
    
    // ... 其他配置
};

// 武器类只负责逻辑
class ARifle : public AWeapon
{
    void Initialize(UWeaponData* Data)
    {
        Damage = Data->Damage;
        FireRate = Data->FireRate;
        // ...
    }
};
```

**优势一目了然**：
- ✅ **策划友好**：在编辑器中点点鼠标即可配置
- ✅ **快速迭代**：无需编译，立即生效
- ✅ **支持热更新**：修改 Data Asset 可以通过 DLC 分发
- ✅ **易于复用**：复制 Data Asset 即可创建变种

### Lyra 的数据驱动架构

```
配置层（Data Assets）
    ↓ 定义数据
逻辑层（C++ Classes）
    ↓ 读取数据并执行
表现层（Blueprints/Materials）
```

**关键原则**：
- 数据和逻辑分离
- 优先使用数据配置
- C++ 只写通用逻辑

---

## 📦 Primary Data Asset 详解

### 什么是 Primary Data Asset？

**Primary Data Asset** 是 UE 的一种特殊资产类型，专为数据驱动设计：

| 特性 | 说明 |
|------|------|
| **可异步加载** | 支持按需加载，不占用启动时间 |
| **Asset Manager 管理** | 统一的资产生命周期管理 |
| **支持依赖追踪** | 自动加载关联资产 |
| **Cook 时优化** | 打包时可以按规则分组 |

### 创建 Primary Data Asset

```cpp
// WeaponDefinition.h

#pragma once

#include "Engine/DataAsset.h"
#include "WeaponDefinition.generated.h"

/**
 * 武器定义（Data Asset）
 * 包含武器的所有配置数据
 */
UCLASS(BlueprintType)
class ULyraWeaponDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // ========== 基础属性 ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Display")
    FText DisplayName;
    
    UPROPERTY(EditDefaultsOnly, Category="Display")
    FSlateBrush WeaponIcon;
    
    UPROPERTY(EditDefaultsOnly, Category="Display")
    FText Description;
    
    // ========== 游戏性参数 ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Combat")
    float BaseDamage = 25.0f;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat")
    float FireRate = 0.1f;  // 射击间隔（秒）
    
    UPROPERTY(EditDefaultsOnly, Category="Combat")
    int32 MagazineSize = 30;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat")
    float ReloadTime = 2.0f;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat")
    float Range = 10000.0f;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat")
    float Spread = 1.0f;  // 散布（度）
    
    // ========== 资产引用 ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Visuals")
    TSoftObjectPtr<USkeletalMesh> WeaponMesh;
    
    UPROPERTY(EditDefaultsOnly, Category="Visuals")
    TSoftObjectPtr<UAnimMontage> FireMontage;
    
    UPROPERTY(EditDefaultsOnly, Category="Visuals")
    TSoftObjectPtr<UAnimMontage> ReloadMontage;
    
    UPROPERTY(EditDefaultsOnly, Category="Effects")
    TSoftObjectPtr<UNiagaraSystem> MuzzleFlashVFX;
    
    UPROPERTY(EditDefaultsOnly, Category="Effects")
    TSoftObjectPtr<USoundBase> FireSound;
    
    // ========== Gameplay Tags ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Tags")
    FGameplayTag WeaponTypeTag;  // 如 Weapon.Type.Rifle
    
    UPROPERTY(EditDefaultsOnly, Category="Tags")
    FGameplayTag WeaponSlotTag;  // 如 Weapon.Slot.Primary
    
    // ========== AssetManager 配置 ==========
    
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        // 返回资产 ID（用于 AssetManager）
        return FPrimaryAssetId(TEXT("WeaponDefinition"), GetFName());
    }
};
```

### 在编辑器中创建实例

1. Content Browser 右键 → **Miscellaneous** → **Data Asset**
2. 选择父类：`ULyraWeaponDefinition`
3. 命名：`DA_Weapon_Rifle_AK47`
4. 打开并配置数值

示例配置：

```
DA_Weapon_Rifle_AK47:
    DisplayName: "AK-47"
    BaseDamage: 35.0
    FireRate: 0.1
    MagazineSize: 30
    ReloadTime: 2.3
    Range: 50000.0
    Spread: 2.5
    WeaponMesh: SK_AK47
    FireMontage: AM_Fire_Rifle
    MuzzleFlashVFX: NS_MuzzleFlash
    FireSound: SFX_AK47_Fire
    WeaponTypeTag: Weapon.Type.Rifle
    WeaponSlotTag: Weapon.Slot.Primary
```

### 在代码中使用

```cpp
// WeaponInstance.cpp

void ULyraWeaponInstance::Initialize(const ULyraWeaponDefinition* WeaponDef)
{
    if (!WeaponDef)
    {
        return;
    }
    
    // 1. 应用数值配置
    CurrentDamage = WeaponDef->BaseDamage;
    CurrentFireRate = WeaponDef->FireRate;
    CurrentAmmo = WeaponDef->MagazineSize;
    MaxAmmo = WeaponDef->MagazineSize;
    
    // 2. 异步加载网格体
    if (!WeaponDef->WeaponMesh.IsNull())
    {
        UAssetManager::GetStreamableManager().RequestAsyncLoad(
            WeaponDef->WeaponMesh.ToSoftObjectPath(),
            FStreamableDelegate::CreateUObject(this, &ThisClass::OnMeshLoaded)
        );
    }
    
    // 3. 缓存 Gameplay Tags
    WeaponTags.AddTag(WeaponDef->WeaponTypeTag);
    WeaponTags.AddTag(WeaponDef->WeaponSlotTag);
}
```

---

## 📊 Data Registry 系统

### 什么是 Data Registry？

**Data Registry** 是 UE5 引入的高级数据管理系统，类似"全局数据库"：

```
传统方式：Data Assets 分散在各个文件夹
Data Registry：统一管理，支持查询和遍历
```

### 创建 Data Registry

#### Step 1: 定义数据结构

```cpp
// WeaponTableRow.h

USTRUCT(BlueprintType)
struct FLyraWeaponTableRow : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly)
    FPrimaryAssetId WeaponDefinitionId;
    
    UPROPERTY(EditDefaultsOnly)
    int32 UnlockLevel = 1;
    
    UPROPERTY(EditDefaultsOnly)
    int32 PurchaseCost = 1000;
    
    UPROPERTY(EditDefaultsOnly)
    bool bIsStarterWeapon = false;
};
```

#### Step 2: 创建 Data Registry Asset

1. Content Browser 右键 → **Miscellaneous** → **Data Registry**
2. 选择数据类型：`FLyraWeaponTableRow`
3. 命名：`DR_Weapons`

#### Step 3: 添加数据条目

在 `DR_Weapons` 中添加：

```
Row Name: Rifle_AK47
    WeaponDefinitionId: WeaponDefinition:DA_Weapon_Rifle_AK47
    UnlockLevel: 1
    PurchaseCost: 1500
    bIsStarterWeapon: true

Row Name: Rifle_M4A1
    WeaponDefinitionId: WeaponDefinition:DA_Weapon_Rifle_M4A1
    UnlockLevel: 5
    PurchaseCost: 2000
    bIsStarterWeapon: false

Row Name: Sniper_AWP
    WeaponDefinitionId: WeaponDefinition:DA_Weapon_Sniper_AWP
    UnlockLevel: 10
    PurchaseCost: 5000
    bIsStarterWeapon: false
```

### 查询 Data Registry

```cpp
// 查询单条数据
void UWeaponShopSubsystem::GetWeaponInfo(FName WeaponRowName)
{
    UDataRegistrySubsystem* DRSubsystem = UDataRegistrySubsystem::Get();
    
    FDataRegistryId RegistryId(TEXT("Weapons"), WeaponRowName);
    
    // 同步查询（如果数据已缓存）
    const FLyraWeaponTableRow* RowData = DRSubsystem->GetCachedItem<FLyraWeaponTableRow>(RegistryId);
    
    if (RowData)
    {
        UE_LOG(LogTemp, Log, TEXT("武器 %s 解锁等级：%d"), *WeaponRowName.ToString(), RowData->UnlockLevel);
    }
    else
    {
        // 异步查询
        DRSubsystem->AcquireItem(
            RegistryId,
            FDataRegistryItemAcquiredCallback::CreateLambda([](const FDataRegistryAcquireResult& Result)
            {
                if (Result.Status == EDataRegistryAcquireStatus::Success)
                {
                    const FLyraWeaponTableRow* Data = Result.GetItem<FLyraWeaponTableRow>();
                    // 使用数据...
                }
            })
        );
    }
}

// 遍历所有数据
void UWeaponShopSubsystem::GetAllStarterWeapons(TArray<FName>& OutWeaponNames)
{
    UDataRegistrySubsystem* DRSubsystem = UDataRegistrySubsystem::Get();
    
    TArray<FDataRegistryLookup> AllItems;
    DRSubsystem->GetAllCachedItems(FDataRegistryType(TEXT("Weapons")), AllItems);
    
    for (const FDataRegistryLookup& Lookup : AllItems)
    {
        const FLyraWeaponTableRow* RowData = DRSubsystem->GetCachedItem<FLyraWeaponTableRow>(Lookup.DataRegistryId);
        
        if (RowData && RowData->bIsStarterWeapon)
        {
            OutWeaponNames.Add(Lookup.DataRegistryId.ItemName);
        }
    }
}
```

### Data Registry 的优势

| 特性 | 说明 |
|------|------|
| **集中管理** | 所有武器数据在一个地方，易于维护 |
| **支持查询** | 可以按条件筛选（如"所有新手武器"） |
| **热更新友好** | 修改 Data Registry 无需重启 |
| **支持 CSV 导入** | 可以从 Excel 批量导入数据 |

---

## 🏷️ Gameplay Tags 最佳实践

### 什么是 Gameplay Tags？

**Gameplay Tags** 是层次化的字符串标签系统，用于标识游戏对象和状态。

```
层次结构示例：
Weapon
├── Weapon.Type
│   ├── Weapon.Type.Rifle
│   ├── Weapon.Type.Shotgun
│   └── Weapon.Type.Sniper
├── Weapon.Slot
│   ├── Weapon.Slot.Primary
│   └── Weapon.Slot.Secondary
└── Weapon.State
    ├── Weapon.State.Firing
    ├── Weapon.State.Reloading
    └── Weapon.State.Empty
```

### 创建 Gameplay Tags

#### 方法1：在项目设置中添加

1. **Edit** → **Project Settings** → **Gameplay Tags**
2. 点击 **Add New Gameplay Tag**
3. 输入标签路径：`Weapon.Type.Rifle`
4. 添加注释（可选）

#### 方法2：通过 INI 文件批量添加

编辑 `Config/DefaultGameplayTags.ini`：

```ini
[/Script/GameplayTags.GameplayTagsSettings]
+GameplayTagList=(Tag="Weapon.Type.Rifle",DevComment="步枪类武器")
+GameplayTagList=(Tag="Weapon.Type.Shotgun",DevComment="霰弹枪类武器")
+GameplayTagList=(Tag="Weapon.Type.Sniper",DevComment="狙击枪类武器")
+GameplayTagList=(Tag="Weapon.Slot.Primary",DevComment="主武器槽位")
+GameplayTagList=(Tag="Weapon.Slot.Secondary",DevComment="副武器槽位")
+GameplayTagList=(Tag="Weapon.State.Firing",DevComment="正在射击")
+GameplayTagList=(Tag="Weapon.State.Reloading",DevComment="正在换弹")
```

### 在 C++ 中使用

```cpp
// WeaponComponent.h

#include "GameplayTagContainer.h"

UCLASS()
class ULyraWeaponComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    // 检查武器类型
    bool IsRifle() const
    {
        return WeaponTags.HasTag(FGameplayTag::RequestGameplayTag(TEXT("Weapon.Type.Rifle")));
    }
    
    // 检查是否在某状态
    bool IsFiring() const
    {
        return WeaponTags.HasTag(FGameplayTag::RequestGameplayTag(TEXT("Weapon.State.Firing")));
    }
    
    // 添加状态标签
    void StartFiring()
    {
        WeaponTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Weapon.State.Firing")));
    }
    
    // 移除状态标签
    void StopFiring()
    {
        WeaponTags.RemoveTag(FGameplayTag::RequestGameplayTag(TEXT("Weapon.State.Firing")));
    }
    
    // 检查标签匹配（支持通配符）
    bool MatchesAnyWeaponType(const FGameplayTagContainer& TagsToCheck) const
    {
        return WeaponTags.HasAny(TagsToCheck);
    }

private:
    UPROPERTY()
    FGameplayTagContainer WeaponTags;
};
```

### Gameplay Tags 的高级用法

#### 1. 标签查询（Tag Query）

```cpp
// 创建复杂查询：(步枪 OR 霰弹枪) AND (不在换弹状态)
FGameplayTagQuery Query;
Query = FGameplayTagQuery::MakeQuery_MatchAnyTags(
    FGameplayTagContainer::CreateFromArray({
        FGameplayTag::RequestGameplayTag(TEXT("Weapon.Type.Rifle")),
        FGameplayTag::RequestGameplayTag(TEXT("Weapon.Type.Shotgun"))
    })
).Matches(FGameplayTagQuery::MakeQuery_MatchNoTags(
    FGameplayTagContainer(FGameplayTag::RequestGameplayTag(TEXT("Weapon.State.Reloading")))
));

// 执行查询
if (Query.Matches(WeaponTags))
{
    // 符合条件
}
```

#### 2. 标签层次匹配

```cpp
// 检查是否匹配父标签（会匹配所有子标签）
FGameplayTag WeaponTypeTag = FGameplayTag::RequestGameplayTag(TEXT("Weapon.Type"));

// 这会匹配 Weapon.Type.Rifle, Weapon.Type.Shotgun 等所有子标签
if (WeaponTags.HasTag(WeaponTypeTag))
{
    // ...
}
```

#### 3. 网络同步

```cpp
UCLASS()
class ALyraWeapon : public AActor
{
    // Gameplay Tags 支持网络同步
    UPROPERTY(ReplicatedUsing=OnRep_WeaponTags)
    FGameplayTagContainer WeaponTags;
    
    UFUNCTION()
    void OnRep_WeaponTags()
    {
        // 标签变化时触发
        OnWeaponTagsChanged.Broadcast(WeaponTags);
    }
};
```

---

## 🔫 实战：数据驱动的武器系统

现在我们综合运用所有知识，构建一个完整的武器系统。

### 系统架构

```
Data Assets (配置层)
    ├── WeaponDefinition (武器定义)
    ├── ProjectileDefinition (子弹定义)
    └── ImpactEffectDefinition (命中特效定义)
    ↓
Data Registry (数据库层)
    └── DR_Weapons (武器表)
    ↓
C++ Classes (逻辑层)
    ├── LyraWeaponInstance (武器实例)
    ├── LyraRangedWeaponInstance (远程武器)
    └── LyraMeleeWeaponInstance (近战武器)
    ↓
Blueprints (表现层)
    └── BP_Weapon_XXX (蓝图子类)
```

### Step 1: 完善武器定义

```cpp
// LyraWeaponDefinition.h

UCLASS()
class ULyraWeaponDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // ========== 基础信息 ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Identity")
    FGameplayTag WeaponID;  // 如 Weapon.Rifle.AK47
    
    UPROPERTY(EditDefaultsOnly, Category="Display")
    FText DisplayName;
    
    UPROPERTY(EditDefaultsOnly, Category="Display", meta=(MultiLine=true))
    FText Description;
    
    UPROPERTY(EditDefaultsOnly, Category="Display")
    TSoftObjectPtr<UTexture2D> WeaponIcon;
    
    // ========== 游戏性参数 ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Combat|Damage")
    float BaseDamage = 25.0f;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat|Damage")
    TSubclassOf<UGameplayEffect> DamageGameplayEffect;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat|FireRate")
    float TimeBetweenShots = 0.1f;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat|Ammo")
    int32 MagazineSize = 30;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat|Ammo")
    int32 MaxAmmoReserve = 300;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat|Ammo")
    float ReloadDuration = 2.0f;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat|Accuracy")
    float BaseSpread = 1.0f;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat|Accuracy")
    float SpreadMultiplierWhileMoving = 1.5f;
    
    UPROPERTY(EditDefaultsOnly, Category="Combat|Accuracy")
    float SpreadMultiplierWhileAiming = 0.5f;
    
    // ========== 子弹配置 ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Projectile")
    TSoftClassPtr<class ALyraProjectile> ProjectileClass;
    
    UPROPERTY(EditDefaultsOnly, Category="Projectile")
    float ProjectileSpeed = 10000.0f;
    
    UPROPERTY(EditDefaultsOnly, Category="Projectile")
    float ProjectileGravityScale = 1.0f;
    
    // ========== 视觉/音效 ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Visuals")
    TSoftObjectPtr<USkeletalMesh> WeaponMesh;
    
    UPROPERTY(EditDefaultsOnly, Category="Animation")
    TSoftObjectPtr<UAnimMontage> FireMontage;
    
    UPROPERTY(EditDefaultsOnly, Category="Animation")
    TSoftObjectPtr<UAnimMontage> ReloadMontage;
    
    UPROPERTY(EditDefaultsOnly, Category="Effects")
    TSoftObjectPtr<UNiagaraSystem> MuzzleFlashVFX;
    
    UPROPERTY(EditDefaultsOnly, Category="Effects")
    TSoftObjectPtr<UNiagaraSystem> TracerVFX;
    
    UPROPERTY(EditDefaultsOnly, Category="Audio")
    TSoftObjectPtr<USoundBase> FireSound;
    
    UPROPERTY(EditDefaultsOnly, Category="Audio")
    TSoftObjectPtr<USoundBase> ReloadSound;
    
    // ========== Gameplay Tags ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Tags")
    FGameplayTagContainer WeaponTags;
    
    UPROPERTY(EditDefaultsOnly, Category="Tags")
    FGameplayTagContainer GrantedTags;  // 装备时赋予角色的标签
    
    // ========== 能力系统 ==========
    
    UPROPERTY(EditDefaultsOnly, Category="Abilities")
    TArray<TSubclassOf<ULyraGameplayAbility>> GrantedAbilities;
};
```

### Step 2: 创建武器实例类

```cpp
// LyraWeaponInstance.cpp

void ULyraRangedWeaponInstance::Initialize(const ULyraWeaponDefinition* InWeaponDef)
{
    WeaponDefinition = InWeaponDef;
    
    // 应用配置
    CurrentAmmo = WeaponDefinition->MagazineSize;
    MaxAmmo = WeaponDefinition->MaxAmmoReserve;
    
    // 异步加载资源
    LoadWeaponAssets();
    
    // 应用 Gameplay Tags
    WeaponTags.AppendTags(WeaponDefinition->WeaponTags);
}

void ULyraRangedWeaponInstance::Fire()
{
    if (!CanFire())
    {
        return;
    }
    
    // 1. 消耗弹药
    CurrentAmmo--;
    
    // 2. 计算散布
    float CurrentSpread = CalculateSpread();
    
    // 3. 发射子弹
    SpawnProjectile(CurrentSpread);
    
    // 4. 播放特效和音效
    PlayFireEffects();
    
    // 5. 应用后坐力
    ApplyRecoil();
    
    // 6. 设置射击冷却
    LastFireTime = GetWorld()->GetTimeSeconds();
}

bool ULyraRangedWeaponInstance::CanFire() const
{
    // 检查弹药
    if (CurrentAmmo <= 0)
    {
        return false;
    }
    
    // 检查冷却
    float TimeSinceLastShot = GetWorld()->GetTimeSeconds() - LastFireTime;
    if (TimeSinceLastShot < WeaponDefinition->TimeBetweenShots)
    {
        return false;
    }
    
    // 检查状态（不能在换弹时射击）
    if (WeaponTags.HasTag(FGameplayTag::RequestGameplayTag(TEXT("Weapon.State.Reloading"))))
    {
        return false;
    }
    
    return true;
}

float ULyraRangedWeaponInstance::CalculateSpread() const
{
    float Spread = WeaponDefinition->BaseSpread;
    
    // 移动时散布增加
    if (ACharacter* Owner = Cast<ACharacter>(GetOwner()))
    {
        if (Owner->GetVelocity().SizeSquared() > 100.0f)
        {
            Spread *= WeaponDefinition->SpreadMultiplierWhileMoving;
        }
    }
    
    // 瞄准时散布减少
    if (WeaponTags.HasTag(FGameplayTag::RequestGameplayTag(TEXT("Weapon.State.Aiming"))))
    {
        Spread *= WeaponDefinition->SpreadMultiplierWhileAiming;
    }
    
    return Spread;
}
```

### Step 3: 配置具体武器

创建 `DA_Weapon_Rifle_AK47`：

```
DisplayName: "AK-47"
Description: "7.62mm 突击步枪，中等伤害，中等后坐力"

BaseDamage: 35.0
TimeBetweenShots: 0.1  (600 RPM)
MagazineSize: 30
ReloadDuration: 2.3
BaseSpread: 2.5
ProjectileSpeed: 50000.0

WeaponMesh: SK_AK47
FireMontage: AM_Fire_Rifle
MuzzleFlashVFX: NS_MuzzleFlash_Rifle
FireSound: SFX_AK47_Fire

WeaponTags:
    - Weapon.Type.Rifle
    - Weapon.Slot.Primary
    - Weapon.Caliber.762
```

创建 `DA_Weapon_Sniper_AWP`：

```
DisplayName: "AWP"
Description: ".338 狙击步枪，极高伤害，低射速"

BaseDamage: 115.0  // 一枪致命
TimeBetweenShots: 1.5  (栓动)
MagazineSize: 5
ReloadDuration: 3.5
BaseSpread: 0.1  // 极高精度
ProjectileSpeed: 100000.0

WeaponMesh: SK_AWP
FireMontage: AM_Fire_Sniper_BoltAction
MuzzleFlashVFX: NS_MuzzleFlash_Sniper
FireSound: SFX_AWP_Fire

WeaponTags:
    - Weapon.Type.Sniper
    - Weapon.Slot.Primary
    - Weapon.Caliber.338
```

### Step 4: 集成到 Data Registry

在 `DR_Weapons` 中添加条目：

```csv
RowName,WeaponDefinitionId,UnlockLevel,PurchaseCost,DamagePerSecond,Accuracy,Mobility
AK47,WeaponDefinition:DA_Weapon_Rifle_AK47,1,1500,350,70,85
M4A1,WeaponDefinition:DA_Weapon_Rifle_M4A1,5,2000,300,85,80
AWP,WeaponDefinition:DA_Weapon_Sniper_AWP,10,5000,77,95,50
```

### Step 5: 使用示例

```cpp
// 玩家装备武器
void ALyraCharacter::EquipWeapon(FGameplayTag WeaponID)
{
    // 1. 从 Data Registry 查询武器配置
    UDataRegistrySubsystem* DRSubsystem = UDataRegistrySubsystem::Get();
    FDataRegistryId RegistryId(TEXT("Weapons"), WeaponID.GetTagName());
    
    const FLyraWeaponTableRow* WeaponRow = DRSubsystem->GetCachedItem<FLyraWeaponTableRow>(RegistryId);
    
    if (!WeaponRow)
    {
        return;
    }
    
    // 2. 加载 Weapon Definition
    UAssetManager& AssetManager = UAssetManager::Get();
    AssetManager.GetPrimaryAssetData(WeaponRow->WeaponDefinitionId, /* 异步回调 */);
    
    // 3. 创建武器实例
    ULyraRangedWeaponInstance* WeaponInstance = NewObject<ULyraRangedWeaponInstance>(this);
    WeaponInstance->Initialize(WeaponDefinition);
    
    // 4. 添加到装备管理器
    ULyraEquipmentManagerComponent* EquipmentMgr = FindComponentByClass<ULyraEquipmentManagerComponent>();
    EquipmentMgr->EquipItem(WeaponInstance);
}
```

---

## ⚙️ 配置文件管理

### DefaultGame.ini 配置

```ini
[/Script/Engine.AssetManagerSettings]
; 注册 Primary Asset 类型
+PrimaryAssetTypesToScan=(PrimaryAssetType="WeaponDefinition",AssetBaseClass=/Script/LyraGame.LyraWeaponDefinition,bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/Weapons")))
+PrimaryAssetTypesToScan=(PrimaryAssetType="ExperienceDefinition",AssetBaseClass=/Script/LyraGame.LyraExperienceDefinition,bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/Experiences")))

; 指定哪些资产需要打包
+PrimaryAssetRules=(PrimaryAssetId="WeaponDefinition:DA_Weapon_Rifle_AK47",Rules=(Priority=-1,ChunkId=-1,bApplyRecursively=True,CookRule=AlwaysCook))
```

### 热更新支持

```cpp
// 监听配置文件变化
void ULyraWeaponSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    
    // 监听 DataRegistry 变化
    UDataRegistrySubsystem* DRSubsystem = UDataRegistrySubsystem::Get();
    DRSubsystem->OnDataRegistryUpdated().AddUObject(this, &ThisClass::OnWeaponDataUpdated);
}

void ULyraWeaponSubsystem::OnWeaponDataUpdated(const FDataRegistryId& UpdatedRegistry)
{
    if (UpdatedRegistry.RegistryType.Name == TEXT("Weapons"))
    {
        // 数据更新，重新加载所有武器配置
        ReloadAllWeapons();
    }
}
```

---

## 💬 总结

### 核心要点

1. **数据驱动的价值**
   - 策划友好，快速迭代
   - 支持热更新和 DLC
   - 易于维护和扩展

2. **Primary Data Asset**
   - 用于定义游戏对象的配置
   - 支持异步加载和依赖管理
   - Asset Manager 统一管理生命周期

3. **Data Registry**
   - 集中管理数据，支持查询
   - 适合需要遍历的场景（如商店、解锁系统）
   - 支持 CSV 导入，方便批量编辑

4. **Gameplay Tags**
   - 层次化标签系统
   - 用于标识对象和状态
   - 支持网络同步和复杂查询

### 实战价值

通过构建数据驱动的武器系统，我们实现了：
- ✅ 无需编程即可配置新武器
- ✅ 策划可以独立调整武器平衡
- ✅ 支持热更新武器数值
- ✅ 通过 Data Registry 统一管理所有武器

### 下一篇预告

第六篇：**Gameplay Ability System (GAS) 入门**

- GAS 核心概念
- LyraAbilitySystemComponent 源码分析
- Ability Set 的设计与使用
- 实战：实现跳跃和冲刺技能

准备好深入 Lyra 最复杂的系统了吗？💪

---

> **本文是《UE5 Lyra 深度解析》系列教程的第 5 篇**  
> 上一篇：[Game Features 插件系统深度剖析](04_game_features_plugin_system.md)  
> 下一篇：《Gameplay Ability System (GAS) 入门》  
> 作者：lobsterchen
