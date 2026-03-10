# 从零开始：实现一个 MOBA 模式

> **实战项目**：基于 Lyra 框架开发完整的 5v5 MOBA 游戏模式，包含英雄系统、经济系统、防御塔、小兵生成、技能升级、装备商店等核心玩法。

---

## 本章目标

通过本章学习，你将能够：

1. **设计完整的 MOBA 游戏架构**
   - 理解 MOBA 核心玩法循环
   - 设计模块化的系统架构
   - 规划数据驱动的配置体系

2. **实现 MOBA 核心系统**
   - 英雄选择与技能升级系统
   - 经济系统（金币、经验值）
   - 防御塔与基地防御
   - 小兵生成与路径寻路
   - 装备商店与物品合成

3. **掌握高级游戏机制**
   - 三路地图设计与 NavMesh
   - 视野系统（战争迷雾）
   - 击杀奖励与助攻判定
   - 大小龙中立野怪
   - 推塔与胜利条件

4. **优化网络性能**
   - 100+ 单位同步优化
   - 技能预测与回滚
   - 相关性过滤策略

---

## 一、MOBA 模式设计总览

### 1.1 核心玩法循环

MOBA（Multiplayer Online Battle Arena，多人在线战术竞技）的核心玩法可以总结为：

```
[对线发育] → [打钱升级] → [购买装备] → [击杀敌人] → [推塔拆家] → [胜利]
    ↑                                                            ↓
    └───────────────── [复活重来] ←───────────────────────────────┘
```

#### 玩法要素分解

| 系统 | 功能描述 | 实现优先级 |
|------|---------|-----------|
| 英雄系统 | 英雄选择、属性成长、技能树 | 🔴 P0 |
| 经济系统 | 金币获取、商店购买、装备合成 | 🔴 P0 |
| 战斗系统 | 技能释放、伤害计算、击杀判定 | 🔴 P0 |
| 地图系统 | 三路对线、野区、中立生物 | 🔴 P0 |
| 防御系统 | 防御塔、水晶基地、超级兵 | 🟡 P1 |
| 视野系统 | 战争迷雾、侦查守卫、隐身 | 🟡 P1 |
| 复活系统 | 死亡惩罚、复活时间、泉水 | 🟢 P2 |
| 匹配系统 | 组队匹配、Ban/Pick、排位 | 🟢 P2 |

---

### 1.2 架构设计

#### 模块划分

```
MobaGameMode (Experience)
├── MobaHeroComponent          # 英雄系统
│   ├── HeroDefinition         # 英雄配置
│   ├── AbilitySet             # 技能集
│   └── LevelUpData            # 升级数据
│
├── MobaEconomyComponent       # 经济系统
│   ├── GoldAttribute          # 金币属性
│   ├── ShopSystem             # 商店系统
│   └── ItemDefinition         # 物品配置
│
├── MobaTowerSystem            # 防御塔系统
│   ├── TowerActor             # 防御塔 Actor
│   ├── TowerAI                # 塔 AI
│   └── TowerHealthSet         # 塔血量
│
├── MobaMinionSpawner          # 小兵生成器
│   ├── MinionWaveConfig       # 兵线配置
│   ├── MinionPathfinding      # 路径寻路
│   └── MinionAI               # 小兵 AI
│
├── MobaMapManager             # 地图管理器
│   ├── LaneDefinition         # 路线定义
│   ├── NeutralCampSpawner     # 野怪刷新
│   └── FogOfWarComponent      # 战争迷雾
│
└── MobaGamePhaseAbility       # 游戏阶段
    ├── HeroSelectPhase        # 英雄选择
    ├── BattlePhase            # 对战阶段
    └── EndGamePhase           # 结算阶段
```

#### 数据驱动设计

所有游戏内容通过 Data Assets 配置，支持热更新：

```cpp
// 英雄配置资产
UCLASS()
class UMobaHeroDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()
public:
    // 基础属性
    UPROPERTY(EditDefaultsOnly, Category="Hero")
    FText HeroName;
    
    UPROPERTY(EditDefaultsOnly, Category="Hero")
    TSoftObjectPtr<UTexture2D> HeroIcon;
    
    // 成长属性
    UPROPERTY(EditDefaultsOnly, Category="Stats")
    FMobaHeroStats BaseStats;
    
    UPROPERTY(EditDefaultsOnly, Category="Stats")
    FMobaHeroStats StatsPerLevel;
    
    // 技能配置
    UPROPERTY(EditDefaultsOnly, Category="Abilities")
    TArray<TSoftClassPtr<UGameplayAbility>> Abilities;
    
    // 模型资源
    UPROPERTY(EditDefaultsOnly, Category="Visual")
    TSoftObjectPtr<USkeletalMesh> HeroMesh;
};

// 物品配置资产
UCLASS()
class UMobaItemDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()
public:
    UPROPERTY(EditDefaultsOnly, Category="Item")
    FText ItemName;
    
    UPROPERTY(EditDefaultsOnly, Category="Item")
    int32 GoldCost;
    
    UPROPERTY(EditDefaultsOnly, Category="Item")
    TArray<FGameplayEffectSpecHandle> GrantedEffects;
    
    // 合成配方
    UPROPERTY(EditDefaultsOnly, Category="Recipe")
    TArray<TSoftObjectPtr<UMobaItemDefinition>> ComponentItems;
};
```

---

## 二、英雄系统实现

### 2.1 英雄组件架构

#### 核心组件代码

```cpp
// MobaHeroComponent.h
UCLASS()
class UMobaHeroComponent : public UPawnComponent
{
    GENERATED_BODY()
    
public:
    UMobaHeroComponent();
    
    // 初始化英雄
    UFUNCTION(BlueprintCallable, Category="Moba|Hero")
    void InitializeHero(const UMobaHeroDefinition* InHeroDefinition);
    
    // 升级处理
    UFUNCTION(BlueprintCallable, Category="Moba|Hero")
    void LevelUp();
    
    UFUNCTION(BlueprintCallable, Category="Moba|Hero")
    bool CanLevelUpAbility(int32 AbilitySlot) const;
    
    UFUNCTION(BlueprintCallable, Category="Moba|Hero")
    void LevelUpAbility(int32 AbilitySlot);
    
    // 属性查询
    UFUNCTION(BlueprintPure, Category="Moba|Hero")
    int32 GetHeroLevel() const { return CurrentLevel; }
    
    UFUNCTION(BlueprintPure, Category="Moba|Hero")
    float GetExperiencePercent() const;
    
protected:
    // 英雄定义
    UPROPERTY(ReplicatedUsing=OnRep_HeroDefinition)
    const UMobaHeroDefinition* HeroDefinition;
    
    UFUNCTION()
    void OnRep_HeroDefinition();
    
    // 等级相关
    UPROPERTY(Replicated)
    int32 CurrentLevel;
    
    UPROPERTY(Replicated)
    float CurrentExperience;
    
    // 技能等级
    UPROPERTY(Replicated)
    TArray<int32> AbilityLevels;
    
    UPROPERTY()
    int32 SkillPointsAvailable;
    
    // 属性计算
    void UpdateHeroStats();
    void ApplyLevelScaling();
};
```

#### 实现细节

```cpp
// MobaHeroComponent.cpp
void UMobaHeroComponent::InitializeHero(const UMobaHeroDefinition* InHeroDefinition)
{
    if (!GetOwner()->HasAuthority())
        return;
    
    HeroDefinition = InHeroDefinition;
    CurrentLevel = 1;
    CurrentExperience = 0.0f;
    
    // 初始化技能等级数组
    AbilityLevels.SetNum(HeroDefinition->Abilities.Num());
    for (int32& Level : AbilityLevels)
    {
        Level = 0;
    }
    
    // 授予技能
    if (UAbilitySystemComponent* ASC = GetOwnerAbilitySystemComponent())
    {
        for (const TSoftClassPtr<UGameplayAbility>& AbilityClass : HeroDefinition->Abilities)
        {
            if (AbilityClass.IsNull())
                continue;
            
            UClass* LoadedClass = AbilityClass.LoadSynchronous();
            if (LoadedClass)
            {
                FGameplayAbilitySpec Spec(LoadedClass, 1, INDEX_NONE, this);
                ASC->GiveAbility(Spec);
            }
        }
    }
    
    // 应用基础属性
    UpdateHeroStats();
}

void UMobaHeroComponent::LevelUp()
{
    if (!GetOwner()->HasAuthority())
        return;
    
    CurrentLevel++;
    SkillPointsAvailable++;
    
    // 重置当前经验值
    CurrentExperience = 0.0f;
    
    // 应用等级成长
    ApplyLevelScaling();
    
    // 广播升级事件
    OnHeroLevelUp.Broadcast(CurrentLevel);
    
    UE_LOG(LogMoba, Log, TEXT("Hero leveled up to %d"), CurrentLevel);
}

void UMobaHeroComponent::ApplyLevelScaling()
{
    UAbilitySystemComponent* ASC = GetOwnerAbilitySystemComponent();
    if (!ASC || !HeroDefinition)
        return;
    
    // 创建属性修改 Effect
    UGameplayEffect* LevelScalingEffect = NewObject<UGameplayEffect>(GetTransientPackage(), TEXT("HeroLevelScaling"));
    LevelScalingEffect->DurationPolicy = EGameplayEffectDurationType::Instant;
    
    // 添加属性修改器
    for (const TPair<FGameplayAttribute, float>& StatPair : HeroDefinition->StatsPerLevel.AttributeMap)
    {
        FGameplayModifierInfo ModInfo;
        ModInfo.Attribute = StatPair.Key;
        ModInfo.ModifierOp = EGameplayModOp::Additive;
        
        FSetByCallerFloat Magnitude;
        Magnitude.DataTag = FGameplayTag::RequestGameplayTag(FName("Moba.Hero.LevelScaling"));
        ModInfo.ModifierMagnitude = FGameplayEffectModifierMagnitude(Magnitude);
        
        LevelScalingEffect->Modifiers.Add(ModInfo);
    }
    
    // 应用 Effect
    FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
    FGameplayEffectSpec Spec(LevelScalingEffect, EffectContext, CurrentLevel);
    
    for (const TPair<FGameplayAttribute, float>& StatPair : HeroDefinition->StatsPerLevel.AttributeMap)
    {
        Spec.SetSetByCallerMagnitude(
            FGameplayTag::RequestGameplayTag(FName("Moba.Hero.LevelScaling")),
            StatPair.Value * (CurrentLevel - 1)
        );
    }
    
    ASC->ApplyGameplayEffectSpecToSelf(Spec);
}

void UMobaHeroComponent::LevelUpAbility(int32 AbilitySlot)
{
    if (!GetOwner()->HasAuthority())
        return;
    
    if (!CanLevelUpAbility(AbilitySlot))
    {
        UE_LOG(LogMoba, Warning, TEXT("Cannot level up ability at slot %d"), AbilitySlot);
        return;
    }
    
    // 增加技能等级
    AbilityLevels[AbilitySlot]++;
    SkillPointsAvailable--;
    
    // 更新技能 Level
    UAbilitySystemComponent* ASC = GetOwnerAbilitySystemComponent();
    if (ASC)
    {
        TSoftClassPtr<UGameplayAbility> AbilityClass = HeroDefinition->Abilities[AbilitySlot];
        if (!AbilityClass.IsNull())
        {
            UClass* LoadedClass = AbilityClass.LoadSynchronous();
            FGameplayAbilitySpec* Spec = ASC->FindAbilitySpecFromClass(LoadedClass);
            if (Spec)
            {
                Spec->Level = AbilityLevels[AbilitySlot];
                ASC->MarkAbilitySpecDirty(*Spec);
            }
        }
    }
    
    OnAbilityLeveledUp.Broadcast(AbilitySlot, AbilityLevels[AbilitySlot]);
}

bool UMobaHeroComponent::CanLevelUpAbility(int32 AbilitySlot) const
{
    if (SkillPointsAvailable <= 0)
        return false;
    
    if (!AbilityLevels.IsValidIndex(AbilitySlot))
        return false;
    
    // 技能等级不能超过英雄等级的一半（大招除外）
    int32 MaxAbilityLevel = (AbilitySlot == 3) ? FMath::CeilToInt(CurrentLevel / 6.0f) : CurrentLevel / 2;
    
    return AbilityLevels[AbilitySlot] < MaxAbilityLevel;
}
```

---

### 2.2 经验值系统

#### 经验值计算

```cpp
// MobaExperienceComponent.h
UCLASS()
class UMobaExperienceComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    // 授予经验值
    UFUNCTION(BlueprintCallable, Category="Moba|Experience")
    void GrantExperience(float Amount, AActor* Source);
    
    // 经验分配
    UFUNCTION(BlueprintCallable, Category="Moba|Experience")
    static void DistributeExperience(float TotalExperience, const TArray<AActor*>& Recipients, float Radius);
    
protected:
    // 经验值表（等级 → 所需经验）
    UPROPERTY(EditDefaultsOnly, Category="Config")
    UCurveFloat* ExperienceCurve;
    
    // 经验分享范围
    UPROPERTY(EditDefaultsOnly, Category="Config")
    float ExperienceShareRadius = 1500.0f;
    
private:
    float CalculateRequiredExperience(int32 Level) const;
    void CheckLevelUp();
};

// 实现
void UMobaExperienceComponent::GrantExperience(float Amount, AActor* Source)
{
    APawn* OwnerPawn = Cast<APawn>(GetOwner());
    if (!OwnerPawn || !OwnerPawn->HasAuthority())
        return;
    
    UMobaHeroComponent* HeroComp = OwnerPawn->FindComponentByClass<UMobaHeroComponent>();
    if (!HeroComp)
        return;
    
    float OldExp = HeroComp->GetCurrentExperience();
    float NewExp = OldExp + Amount;
    
    HeroComp->SetCurrentExperience(NewExp);
    
    // 检查是否升级
    CheckLevelUp();
    
    // 广播经验获得事件
    OnExperienceGained.Broadcast(Amount, Source);
}

void UMobaExperienceComponent::DistributeExperience(float TotalExperience, const TArray<AActor*>& Recipients, float Radius)
{
    if (Recipients.Num() == 0)
        return;
    
    // MOBA 经验分配规则：
    // - 击杀者获得 100% 经验
    // - 范围内友军平分 50% 额外经验
    
    AActor* Killer = Recipients[0];
    if (UMobaExperienceComponent* KillerExpComp = Killer->FindComponentByClass<UMobaExperienceComponent>())
    {
        KillerExpComp->GrantExperience(TotalExperience, nullptr);
    }
    
    // 分享经验给范围内友军
    TArray<AActor*> AlliesInRange;
    for (AActor* Recipient : Recipients)
    {
        if (Recipient != Killer)
        {
            float Distance = FVector::Dist(Killer->GetActorLocation(), Recipient->GetActorLocation());
            if (Distance <= Radius)
            {
                AlliesInRange.Add(Recipient);
            }
        }
    }
    
    if (AlliesInRange.Num() > 0)
    {
        float SharedExp = (TotalExperience * 0.5f) / AlliesInRange.Num();
        for (AActor* Ally : AlliesInRange)
        {
            if (UMobaExperienceComponent* AllyExpComp = Ally->FindComponentByClass<UMobaExperienceComponent>())
            {
                AllyExpComp->GrantExperience(SharedExp, Killer);
            }
        }
    }
}

void UMobaExperienceComponent::CheckLevelUp()
{
    UMobaHeroComponent* HeroComp = GetOwner()->FindComponentByClass<UMobaHeroComponent>();
    if (!HeroComp)
        return;
    
    int32 CurrentLevel = HeroComp->GetHeroLevel();
    float CurrentExp = HeroComp->GetCurrentExperience();
    float RequiredExp = CalculateRequiredExperience(CurrentLevel + 1);
    
    while (CurrentExp >= RequiredExp && CurrentLevel < 18)
    {
        // 升级
        HeroComp->LevelUp();
        
        CurrentLevel = HeroComp->GetHeroLevel();
        CurrentExp = HeroComp->GetCurrentExperience();
        RequiredExp = CalculateRequiredExperience(CurrentLevel + 1);
    }
}

float UMobaExperienceComponent::CalculateRequiredExperience(int32 Level) const
{
    if (!ExperienceCurve)
    {
        // 默认公式：100 * Level^1.5
        return 100.0f * FMath::Pow(Level, 1.5f);
    }
    
    return ExperienceCurve->GetFloatValue(Level);
}
```

#### 经验曲线配置

在虚幻编辑器中创建 `MobaExperienceCurve` 曲线资产：

| 等级 | 所需经验 | 累计经验 |
|------|---------|---------|
| 1→2  | 280     | 280     |
| 2→3  | 380     | 660     |
| 3→4  | 480     | 1140    |
| 4→5  | 580     | 1720    |
| ...  | ...     | ...     |
| 17→18| 2260    | 19620   |

---

## 三、经济系统实现

### 3.1 金币属性

使用 GAS Attribute 管理金币：

```cpp
// MobaEconomyAttributeSet.h
UCLASS()
class UMobaEconomyAttributeSet : public UAttributeSet
{
    GENERATED_BODY()
    
public:
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_Gold, Category="Economy")
    FGameplayAttributeData Gold;
    ATTRIBUTE_ACCESSORS(UMobaEconomyAttributeSet, Gold)
    
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_TotalGoldEarned, Category="Economy")
    FGameplayAttributeData TotalGoldEarned;
    ATTRIBUTE_ACCESSORS(UMobaEconomyAttributeSet, TotalGoldEarned)
    
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_GoldPerSecond, Category="Economy")
    FGameplayAttributeData GoldPerSecond;
    ATTRIBUTE_ACCESSORS(UMobaEconomyAttributeSet, GoldPerSecond)
    
protected:
    UFUNCTION()
    void OnRep_Gold(const FGameplayAttributeData& OldGold);
    
    UFUNCTION()
    void OnRep_TotalGoldEarned(const FGameplayAttributeData& OldTotal);
    
    UFUNCTION()
    void OnRep_GoldPerSecond(const FGameplayAttributeData& OldGPS);
    
    // 预执行属性修改
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;
};

// 实现
void UMobaEconomyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);
    
    if (Attribute == GetGoldAttribute())
    {
        // 金币不能为负
        NewValue = FMath::Max(NewValue, 0.0f);
    }
}

void UMobaEconomyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);
    
    if (Data.EvaluatedData.Attribute == GetGoldAttribute())
    {
        float DeltaGold = Data.EvaluatedData.Magnitude;
        
        // 记录总获得金币（仅正值）
        if (DeltaGold > 0.0f)
        {
            SetTotalGoldEarned(GetTotalGoldEarned() + DeltaGold);
        }
        
        // 广播金币变化事件
        if (UAbilitySystemComponent* ASC = Data.Target.AbilityActorInfo->AbilitySystemComponent.Get())
        {
            FGameplayEventData EventData;
            EventData.EventMagnitude = DeltaGold;
            
            ASC->HandleGameplayEvent(
                FGameplayTag::RequestGameplayTag(FName("Moba.Event.GoldChanged")),
                &EventData
            );
        }
    }
}
```

---

### 3.2 商店系统

#### 商店组件

```cpp
// MobaShopComponent.h
UCLASS()
class UMobaShopComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    // 购买物品
    UFUNCTION(BlueprintCallable, Server, Reliable, Category="Moba|Shop")
    void ServerPurchaseItem(const UMobaItemDefinition* ItemDef);
    
    // 出售物品
    UFUNCTION(BlueprintCallable, Server, Reliable, Category="Moba|Shop")
    void ServerSellItem(int32 InventorySlot);
    
    // 查询可购买
    UFUNCTION(BlueprintPure, Category="Moba|Shop")
    bool CanAffordItem(const UMobaItemDefinition* ItemDef) const;
    
    UFUNCTION(BlueprintPure, Category="Moba|Shop")
    bool CanPurchaseItem(const UMobaItemDefinition* ItemDef) const;
    
protected:
    // 背包管理
    UPROPERTY(Replicated)
    TArray<const UMobaItemDefinition*> Inventory;
    
    UPROPERTY(EditDefaultsOnly, Category="Config")
    int32 MaxInventorySlots = 6;
    
    // 物品合成检测
    bool TryAutoComposeItems();
    void ApplyItemEffects(const UMobaItemDefinition* ItemDef);
    void RemoveItemEffects(const UMobaItemDefinition* ItemDef);
};

// 实现
void UMobaShopComponent::ServerPurchaseItem_Implementation(const UMobaItemDefinition* ItemDef)
{
    if (!ItemDef || !GetOwner()->HasAuthority())
        return;
    
    if (!CanPurchaseItem(ItemDef))
    {
        UE_LOG(LogMoba, Warning, TEXT("Cannot purchase item: %s"), *ItemDef->ItemName.ToString());
        return;
    }
    
    // 扣除金币
    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(GetOwner());
    if (ASC)
    {
        FGameplayEffectSpecHandle SpecHandle = MakeOutgoingSpec(UGameplayEffect_SpendGold::StaticClass(), 1.0f, ASC->MakeEffectContext());
        SpecHandle.Data->SetSetByCallerMagnitude(
            FGameplayTag::RequestGameplayTag(FName("Moba.Economy.Gold.Cost")),
            -ItemDef->GoldCost
        );
        
        ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
    }
    
    // 添加到背包
    Inventory.Add(ItemDef);
    
    // 应用物品效果
    ApplyItemEffects(ItemDef);
    
    // 尝试自动合成
    TryAutoComposeItems();
    
    OnItemPurchased.Broadcast(ItemDef);
}

bool UMobaShopComponent::TryAutoComposeItems()
{
    // 检查背包中是否有可合成的物品
    for (const UMobaItemDefinition* ItemDef : Inventory)
    {
        if (ItemDef->ComponentItems.Num() == 0)
            continue;
        
        bool bCanCompose = true;
        TArray<int32> ComponentIndices;
        
        for (const TSoftObjectPtr<UMobaItemDefinition>& ComponentPtr : ItemDef->ComponentItems)
        {
            const UMobaItemDefinition* ComponentItem = ComponentPtr.LoadSynchronous();
            int32 Index = Inventory.IndexOfByKey(ComponentItem);
            
            if (Index == INDEX_NONE)
            {
                bCanCompose = false;
                break;
            }
            
            ComponentIndices.Add(Index);
        }
        
        if (bCanCompose)
        {
            // 移除合成材料
            for (int32 Idx : ComponentIndices)
            {
                Inventory.RemoveAt(Idx);
            }
            
            // 添加合成物品
            Inventory.Add(ItemDef);
            ApplyItemEffects(ItemDef);
            
            OnItemComposed.Broadcast(ItemDef);
            return true;
        }
    }
    
    return false;
}

void UMobaShopComponent::ApplyItemEffects(const UMobaItemDefinition* ItemDef)
{
    UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(GetOwner());
    if (!ASC)
        return;
    
    for (const FGameplayEffectSpecHandle& EffectSpec : ItemDef->GrantedEffects)
    {
        if (EffectSpec.IsValid())
        {
            ASC->ApplyGameplayEffectSpecToSelf(*EffectSpec.Data.Get());
        }
    }
}
```

#### 物品配置示例

```cpp
// 配置长剑（+10 攻击力）
UMobaItemDefinition* LongSword = NewObject<UMobaItemDefinition>();
LongSword->ItemName = FText::FromString(TEXT("长剑"));
LongSword->GoldCost = 350;

// 创建属性加成 Effect
UGameplayEffect* AttackDamageEffect = NewObject<UGameplayEffect>();
AttackDamageEffect->DurationPolicy = EGameplayEffectDurationType::Infinite;

FGameplayModifierInfo AttackMod;
AttackMod.Attribute = ULyraCombatSet::GetBaseDamageAttribute();
AttackMod.ModifierOp = EGameplayModOp::Additive;
AttackMod.ModifierMagnitude = FScalableFloat(10.0f);

AttackDamageEffect->Modifiers.Add(AttackMod);

FGameplayEffectContextHandle Context;
LongSword->GrantedEffects.Add(FGameplayEffectSpecHandle(new FGameplayEffectSpec(AttackDamageEffect, Context, 1.0f)));
```

---

## 四、防御塔系统

### 4.1 防御塔 Actor

```cpp
// MobaTowerActor.h
UCLASS()
class AMobaTowerActor : public AActor, public IAbilitySystemInterface
{
    GENERATED_BODY()
    
public:
    AMobaTowerActor();
    
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;
    
    // IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    
protected:
    // 组件
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    UStaticMeshComponent* TowerMesh;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    USphereComponent* AttackRangeCollision;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    UAbilitySystemComponent* AbilitySystemComponent;
    
    UPROPERTY()
    UMobaTowerAttributeSet* TowerAttributes;
    
    // 配置
    UPROPERTY(EditAnywhere, Category="Config")
    float AttackRange = 800.0f;
    
    UPROPERTY(EditAnywhere, Category="Config")
    float AttackDamage = 150.0f;
    
    UPROPERTY(EditAnywhere, Category="Config")
    float AttackInterval = 1.0f;
    
    UPROPERTY(EditAnywhere, Category="Config")
    ETeamAttitude::Type TeamId = ETeamAttitude::Friendly;
    
    // AI 逻辑
    UPROPERTY()
    AActor* CurrentTarget;
    
    float LastAttackTime;
    
    void SearchForTarget();
    void AttackTarget();
    bool IsValidTarget(AActor* Target) const;
    
    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, 
                        int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);
    
    UFUNCTION()
    void OnOverlapEnd(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);
};

// 实现
AMobaTowerActor::AMobaTowerActor()
{
    PrimaryActorTick.bCanEverTick = true;
    
    TowerMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("TowerMesh"));
    RootComponent = TowerMesh;
    
    AttackRangeCollision = CreateDefaultSubobject<USphereComponent>(TEXT("AttackRange"));
    AttackRangeCollision->SetupAttachment(RootComponent);
    AttackRangeCollision->InitSphereRadius(AttackRange);
    AttackRangeCollision->SetCollisionResponseToAllChannels(ECR_Ignore);
    AttackRangeCollision->SetCollisionResponseToChannel(ECC_Pawn, ECR_Overlap);
    
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
    
    TowerAttributes = CreateDefaultSubobject<UMobaTowerAttributeSet>(TEXT("TowerAttributes"));
    
    bReplicates = true;
    SetReplicateMovement(false);
}

void AMobaTowerActor::BeginPlay()
{
    Super::BeginPlay();
    
    if (HasAuthority())
    {
        // 初始化属性
        if (AbilitySystemComponent && TowerAttributes)
        {
            AbilitySystemComponent->InitStats(UMobaTowerAttributeSet::StaticClass(), nullptr);
            
            // 设置初始血量
            AbilitySystemComponent->SetNumericAttributeBase(
                TowerAttributes->GetHealthAttribute(),
                2500.0f
            );
        }
        
        // 绑定碰撞事件
        AttackRangeCollision->OnComponentBeginOverlap.AddDynamic(this, &AMobaTowerActor::OnOverlapBegin);
        AttackRangeCollision->OnComponentEndOverlap.AddDynamic(this, &AMobaTowerActor::OnOverlapEnd);
    }
}

void AMobaTowerActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    if (!HasAuthority())
        return;
    
    // 血量为 0 时摧毁
    if (TowerAttributes && TowerAttributes->GetHealth() <= 0.0f)
    {
        Destroy();
        return;
    }
    
    // 攻击逻辑
    if (!CurrentTarget || !IsValidTarget(CurrentTarget))
    {
        SearchForTarget();
    }
    
    if (CurrentTarget)
    {
        float CurrentTime = GetWorld()->GetTimeSeconds();
        if (CurrentTime - LastAttackTime >= AttackInterval)
        {
            AttackTarget();
            LastAttackTime = CurrentTime;
        }
    }
}

void AMobaTowerActor::SearchForTarget()
{
    CurrentTarget = nullptr;
    
    TArray<AActor*> OverlappingActors;
    AttackRangeCollision->GetOverlappingActors(OverlappingActors, APawn::StaticClass());
    
    // 优先级：英雄 > 小兵
    AActor* BestTarget = nullptr;
    int32 BestPriority = -1;
    
    for (AActor* Actor : OverlappingActors)
    {
        if (!IsValidTarget(Actor))
            continue;
        
        int32 Priority = Cast<ACharacter>(Actor) ? 1 : 0;
        
        if (Priority > BestPriority)
        {
            BestTarget = Actor;
            BestPriority = Priority;
        }
    }
    
    CurrentTarget = BestTarget;
}

void AMobaTowerActor::AttackTarget()
{
    if (!CurrentTarget)
        return;
    
    // 应用伤害 Gameplay Effect
    UAbilitySystemComponent* TargetASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(CurrentTarget);
    if (TargetASC)
    {
        FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
        EffectContext.AddInstigator(this, this);
        
        FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
            UGameplayEffect_TowerDamage::StaticClass(),
            1.0f,
            EffectContext
        );
        
        if (SpecHandle.IsValid())
        {
            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(FName("Damage.Physical")),
                AttackDamage
            );
            
            AbilitySystemComponent->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
        }
    }
    
    // 播放攻击动画/特效
    OnTowerAttack.Broadcast(CurrentTarget);
}

bool AMobaTowerActor::IsValidTarget(AActor* Target) const
{
    if (!Target || Target->IsPendingKill())
        return false;
    
    // 检查队伍
    IGenericTeamAgentInterface* TeamAgent = Cast<IGenericTeamAgentInterface>(Target);
    if (!TeamAgent)
        return false;
    
    FGenericTeamId TargetTeamId = TeamAgent->GetGenericTeamId();
    FGenericTeamId MyTeamId = FGenericTeamId(static_cast<uint8>(TeamId));
    
    return FGenericTeamId::GetAttitude(MyTeamId, TargetTeamId) == ETeamAttitude::Hostile;
}
```

---

### 4.2 防御塔配置

在地图中放置防御塔时，配置以下参数：

| 参数 | 上路塔 | 中路塔 | 下路塔 | 基地塔 |
|------|-------|-------|-------|-------|
| 血量 | 2500  | 2500  | 2500  | 3500  |
| 攻击力 | 150  | 150   | 150   | 200   |
| 攻击间隔 | 1.0s | 1.0s | 1.0s | 0.8s |
| 攻击范围 | 800  | 800   | 800   | 1000  |
| 金币奖励 | 300  | 300   | 300   | 500   |

---

## 五、小兵生成系统

### 5.1 兵线生成器

```cpp
// MobaMinionSpawner.h
UCLASS()
class AMobaMinionSpawner : public AActor
{
    GENERATED_BODY()
    
public:
    AMobaMinionSpawner();
    
    virtual void BeginPlay() override;
    
    // 生成兵线
    UFUNCTION(BlueprintCallable, Category="Moba|Minion")
    void SpawnWave();
    
protected:
    // 配置
    UPROPERTY(EditAnywhere, Category="Config")
    TSubclassOf<AMobaMinionCharacter> MeleeMinionClass;
    
    UPROPERTY(EditAnywhere, Category="Config")
    TSubclassOf<AMobaMinionCharacter> RangedMinionClass;
    
    UPROPERTY(EditAnywhere, Category="Config")
    TSubclassOf<AMobaMinionCharacter> SiegeMinionClass;
    
    UPROPERTY(EditAnywhere, Category="Config")
    ETeamAttitude::Type SpawnerTeam = ETeamAttitude::Friendly;
    
    UPROPERTY(EditAnywhere, Category="Config")
    float WaveInterval = 30.0f;
    
    UPROPERTY(EditAnywhere, Category="Config")
    int32 MeleeMinionsPerWave = 3;
    
    UPROPERTY(EditAnywhere, Category="Config")
    int32 RangedMinionsPerWave = 1;
    
    UPROPERTY(EditAnywhere, Category="Config")
    int32 WaveNumberForSiegeMinion = 3;
    
    // 路径配置
    UPROPERTY(EditAnywhere, Category="Config")
    TArray<FVector> WaypointPath;
    
    // 运行时
    FTimerHandle SpawnTimerHandle;
    int32 CurrentWaveNumber;
    
    void SpawnMinion(TSubclassOf<AMobaMinionCharacter> MinionClass);
};

// 实现
void AMobaMinionSpawner::BeginPlay()
{
    Super::BeginPlay();
    
    if (HasAuthority())
    {
        CurrentWaveNumber = 0;
        
        // 延迟首次生成（等待游戏开始）
        GetWorldTimerManager().SetTimer(
            SpawnTimerHandle,
            this,
            &AMobaMinionSpawner::SpawnWave,
            WaveInterval,
            true,
            90.0f  // 游戏开始 90 秒后刷新首波兵
        );
    }
}

void AMobaMinionSpawner::SpawnWave()
{
    CurrentWaveNumber++;
    
    // 生成近战兵
    for (int32 i = 0; i < MeleeMinionsPerWave; i++)
    {
        SpawnMinion(MeleeMinionClass);
    }
    
    // 生成远程兵
    for (int32 i = 0; i < RangedMinionsPerWave; i++)
    {
        SpawnMinion(RangedMinionClass);
    }
    
    // 每 3 波生成攻城兵
    if (CurrentWaveNumber % WaveNumberForSiegeMinion == 0)
    {
        SpawnMinion(SiegeMinionClass);
    }
    
    UE_LOG(LogMoba, Log, TEXT("Spawned wave %d for team %d"), CurrentWaveNumber, (int32)SpawnerTeam);
}

void AMobaMinionSpawner::SpawnMinion(TSubclassOf<AMobaMinionCharacter> MinionClass)
{
    if (!MinionClass)
        return;
    
    FActorSpawnParameters SpawnParams;
    SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn;
    
    FVector SpawnLocation = GetActorLocation();
    FRotator SpawnRotation = GetActorRotation();
    
    AMobaMinionCharacter* Minion = GetWorld()->SpawnActor<AMobaMinionCharacter>(
        MinionClass,
        SpawnLocation,
        SpawnRotation,
        SpawnParams
    );
    
    if (Minion)
    {
        // 设置队伍
        Minion->SetGenericTeamId(FGenericTeamId(static_cast<uint8>(SpawnerTeam)));
        
        // 设置路径
        if (UMobaMinionAIComponent* AIComp = Minion->FindComponentByClass<UMobaMinionAIComponent>())
        {
            AIComp->SetPatrolPath(WaypointPath);
        }
    }
}
```

---

### 5.2 小兵 AI

```cpp
// MobaMinionAIComponent.h
UCLASS()
class UMobaMinionAIComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
    
    UFUNCTION(BlueprintCallable, Category="Moba|AI")
    void SetPatrolPath(const TArray<FVector>& InPath);
    
protected:
    // 路径
    TArray<FVector> PatrolPath;
    int32 CurrentWaypointIndex;
    
    // 目标
    UPROPERTY()
    AActor* CombatTarget;
    
    float SearchRadius = 800.0f;
    float AttackRange = 150.0f;
    
    void MoveToNextWaypoint();
    void SearchForEnemies();
    void AttackEnemy();
    bool IsInAttackRange() const;
};

// 实现
void UMobaMinionAIComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    
    APawn* OwnerPawn = Cast<APawn>(GetOwner());
    if (!OwnerPawn || !OwnerPawn->HasAuthority())
        return;
    
    // AI 状态机
    if (!CombatTarget)
    {
        // 搜索敌人
        SearchForEnemies();
        
        if (!CombatTarget && PatrolPath.Num() > 0)
        {
            // 沿路径移动
            MoveToNextWaypoint();
        }
    }
    else
    {
        // 追击并攻击
        if (IsInAttackRange())
        {
            AttackEnemy();
        }
        else
        {
            // 移动到目标
            if (AAIController* AIController = Cast<AAIController>(OwnerPawn->GetController()))
            {
                AIController->MoveToActor(CombatTarget, AttackRange * 0.9f);
            }
        }
        
        // 检查目标有效性
        if (!CombatTarget || CombatTarget->IsPendingKill())
        {
            CombatTarget = nullptr;
        }
    }
}

void UMobaMinionAIComponent::SearchForEnemies()
{
    APawn* OwnerPawn = Cast<APawn>(GetOwner());
    if (!OwnerPawn)
        return;
    
    // 范围检测
    TArray<FOverlapResult> Overlaps;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(OwnerPawn);
    
    GetWorld()->OverlapMultiByChannel(
        Overlaps,
        OwnerPawn->GetActorLocation(),
        FQuat::Identity,
        ECC_Pawn,
        FCollisionShape::MakeSphere(SearchRadius),
        QueryParams
    );
    
    // 优先级：英雄 > 防御塔 > 小兵
    AActor* BestTarget = nullptr;
    int32 BestPriority = -1;
    
    for (const FOverlapResult& Overlap : Overlaps)
    {
        AActor* OtherActor = Overlap.GetActor();
        if (!OtherActor)
            continue;
        
        // 检查是否敌对
        IGenericTeamAgentInterface* TeamAgent = Cast<IGenericTeamAgentInterface>(OtherActor);
        if (!TeamAgent)
            continue;
        
        FGenericTeamId MyTeamId = Cast<IGenericTeamAgentInterface>(OwnerPawn)->GetGenericTeamId();
        FGenericTeamId OtherTeamId = TeamAgent->GetGenericTeamId();
        
        if (FGenericTeamId::GetAttitude(MyTeamId, OtherTeamId) != ETeamAttitude::Hostile)
            continue;
        
        // 计算优先级
        int32 Priority = 0;
        if (OtherActor->IsA(ACharacter::StaticClass()))
            Priority = 2;
        else if (OtherActor->IsA(AMobaTowerActor::StaticClass()))
            Priority = 1;
        
        if (Priority > BestPriority)
        {
            BestTarget = OtherActor;
            BestPriority = Priority;
        }
    }
    
    CombatTarget = BestTarget;
}

void UMobaMinionAIComponent::MoveToNextWaypoint()
{
    if (PatrolPath.Num() == 0)
        return;
    
    APawn* OwnerPawn = Cast<APawn>(GetOwner());
    AAIController* AIController = OwnerPawn ? Cast<AAIController>(OwnerPawn->GetController()) : nullptr;
    
    if (!AIController)
        return;
    
    FVector CurrentWaypoint = PatrolPath[CurrentWaypointIndex];
    float DistanceToWaypoint = FVector::Dist(OwnerPawn->GetActorLocation(), CurrentWaypoint);
    
    if (DistanceToWaypoint < 100.0f)
    {
        // 到达当前路点，移动到下一个
        CurrentWaypointIndex = (CurrentWaypointIndex + 1) % PatrolPath.Num();
        CurrentWaypoint = PatrolPath[CurrentWaypointIndex];
    }
    
    AIController->MoveToLocation(CurrentWaypoint);
}
```

---

## 六、地图与视野系统

### 6.1 三路地图设计

#### 地图布局

```
         [基地蓝]
           |
    [上路塔蓝]--[中路塔蓝]--[下路塔蓝]
       /          |            \
    [野区]     [河道]        [野区]
       \          |            /
    [上路塔红]--[中路塔红]--[下路塔红]
           |
         [基地红]
```

#### 路线配置

```cpp
// 在 MobaMapManager 中定义路线
void UMobaMapManager::InitializeLanes()
{
    // 上路路径（蓝方）
    TopLaneBlue.Waypoints = {
        FVector(0, 3000, 100),      // 蓝方上路塔 1
        FVector(1500, 3000, 100),
        FVector(3000, 3000, 100),   // 河道
        FVector(4500, 3000, 100),
        FVector(6000, 3000, 100),   // 红方上路塔 1
        FVector(7500, 3000, 100)    // 红方基地
    };
    
    // 中路路径（蓝方）
    MidLaneBlue.Waypoints = {
        FVector(0, 0, 100),         // 蓝方中路塔 1
        FVector(2000, 0, 100),
        FVector(4000, 0, 100),      // 河道
        FVector(6000, 0, 100),
        FVector(8000, 0, 100)       // 红方中路塔 1
    };
    
    // 下路路径（蓝方）
    BotLaneBlue.Waypoints = {
        FVector(0, -3000, 100),     // 蓝方下路塔 1
        FVector(1500, -3000, 100),
        FVector(3000, -3000, 100),  // 河道
        FVector(4500, -3000, 100),
        FVector(6000, -3000, 100)   // 红方下路塔 1
    };
}
```

---

### 6.2 战争迷雾（视野系统）

#### 视野组件

```cpp
// MobaVisionComponent.h
UCLASS()
class UMobaVisionComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
    
    UFUNCTION(BlueprintPure, Category="Moba|Vision")
    bool CanSeeActor(AActor* Target) const;
    
    UFUNCTION(BlueprintPure, Category="Moba|Vision")
    TArray<AActor*> GetVisibleEnemies() const;
    
protected:
    UPROPERTY(EditAnywhere, Category="Config")
    float VisionRadius = 1200.0f;
    
    UPROPERTY(EditAnywhere, Category="Config")
    float UpdateInterval = 0.2f;
    
    float LastUpdateTime;
    
    UPROPERTY()
    TArray<AActor*> CachedVisibleActors;
    
    void UpdateVisibility();
    bool HasLineOfSight(AActor* Target) const;
};

// 实现
void UMobaVisionComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    
    float CurrentTime = GetWorld()->GetTimeSeconds();
    if (CurrentTime - LastUpdateTime >= UpdateInterval)
    {
        UpdateVisibility();
        LastUpdateTime = CurrentTime;
    }
}

void UMobaVisionComponent::UpdateVisibility()
{
    CachedVisibleActors.Empty();
    
    APawn* OwnerPawn = Cast<APawn>(GetOwner());
    if (!OwnerPawn)
        return;
    
    // 范围检测
    TArray<FOverlapResult> Overlaps;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(OwnerPawn);
    
    GetWorld()->OverlapMultiByChannel(
        Overlaps,
        OwnerPawn->GetActorLocation(),
        FQuat::Identity,
        ECC_Pawn,
        FCollisionShape::MakeSphere(VisionRadius),
        QueryParams
    );
    
    for (const FOverlapResult& Overlap : Overlaps)
    {
        AActor* OtherActor = Overlap.GetActor();
        if (!OtherActor)
            continue;
        
        // 检查是否敌对
        IGenericTeamAgentInterface* TeamAgent = Cast<IGenericTeamAgentInterface>(OtherActor);
        if (!TeamAgent)
            continue;
        
        FGenericTeamId MyTeamId = Cast<IGenericTeamAgentInterface>(OwnerPawn)->GetGenericTeamId();
        FGenericTeamId OtherTeamId = TeamAgent->GetGenericTeamId();
        
        if (FGenericTeamId::GetAttitude(MyTeamId, OtherTeamId) != ETeamAttitude::Hostile)
            continue;
        
        // 检查视线
        if (HasLineOfSight(OtherActor))
        {
            CachedVisibleActors.Add(OtherActor);
        }
    }
}

bool UMobaVisionComponent::HasLineOfSight(AActor* Target) const
{
    if (!Target)
        return false;
    
    FVector Start = GetOwner()->GetActorLocation();
    FVector End = Target->GetActorLocation();
    
    FHitResult HitResult;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(GetOwner());
    QueryParams.AddIgnoredActor(Target);
    
    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        Start,
        End,
        ECC_Visibility,
        QueryParams
    );
    
    return !bHit;  // 没有遮挡物才能看见
}

bool UMobaVisionComponent::CanSeeActor(AActor* Target) const
{
    return CachedVisibleActors.Contains(Target);
}
```

#### 客户端渲染

在客户端隐藏不可见的敌方单位：

```cpp
// MobaPlayerController.cpp
void AMobaPlayerController::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    if (!IsLocalController())
        return;
    
    // 获取己方所有视野组件
    TArray<UMobaVisionComponent*> AlliedVisionComps;
    for (TActorIterator<APawn> It(GetWorld()); It; ++It)
    {
        APawn* Pawn = *It;
        if (IGenericTeamAgentInterface* TeamAgent = Cast<IGenericTeamAgentInterface>(Pawn))
        {
            if (FGenericTeamId::GetAttitude(GetGenericTeamId(), TeamAgent->GetGenericTeamId()) == ETeamAttitude::Friendly)
            {
                if (UMobaVisionComponent* VisionComp = Pawn->FindComponentByClass<UMobaVisionComponent>())
                {
                    AlliedVisionComps.Add(VisionComp);
                }
            }
        }
    }
    
    // 更新敌方单位可见性
    for (TActorIterator<APawn> It(GetWorld()); It; ++It)
    {
        APawn* EnemyPawn = *It;
        if (IGenericTeamAgentInterface* TeamAgent = Cast<IGenericTeamAgentInterface>(EnemyPawn))
        {
            if (FGenericTeamId::GetAttitude(GetGenericTeamId(), TeamAgent->GetGenericTeamId()) == ETeamAttitude::Hostile)
            {
                bool bVisible = false;
                for (UMobaVisionComponent* VisionComp : AlliedVisionComps)
                {
                    if (VisionComp->CanSeeActor(EnemyPawn))
                    {
                        bVisible = true;
                        break;
                    }
                }
                
                // 设置可见性
                EnemyPawn->SetActorHiddenInGame(!bVisible);
            }
        }
    }
}
```

---

## 七、网络优化

### 7.1 相关性过滤

使用 Replication Graph 优化网络同步：

```cpp
// MobaReplicationGraph.h
UCLASS()
class UMobaReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()
    
public:
    virtual void InitGlobalActorClassSettings() override;
    virtual void InitGlobalGraphNodes() override;
    virtual void RouteAddNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo, FGlobalActorReplicationInfo& GlobalInfo) override;
    virtual void RouteRemoveNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo) override;
    
protected:
    UPROPERTY()
    UReplicationGraphNode_GridSpatialization2D* GridNode;
    
    UPROPERTY()
    UReplicationGraphNode_ActorList* AlwaysRelevantNode;
};

// 实现
void UMobaReplicationGraph::InitGlobalGraphNodes()
{
    Super::InitGlobalGraphNodes();
    
    // 空间分区节点（用于小兵、英雄）
    GridNode = CreateNewNode<UReplicationGraphNode_GridSpatialization2D>();
    GridNode->CellSize = 10000.0f;       // 10米一格
    GridNode->SpatialBias = FVector2D(-50000.0f, -50000.0f);
    GridNode->AddDynamicActor_GridSpatialization2D();
    AddGlobalGraphNode(GridNode);
    
    // 始终相关节点（防御塔、基地）
    AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_ActorList>();
    AddGlobalGraphNode(AlwaysRelevantNode);
}

void UMobaReplicationGraph::RouteAddNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo, FGlobalActorReplicationInfo& GlobalInfo)
{
    if (AMobaTowerActor* Tower = Cast<AMobaTowerActor>(ActorInfo.Actor))
    {
        AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
    }
    else if (AMobaMinionCharacter* Minion = Cast<AMobaMinionCharacter>(ActorInfo.Actor))
    {
        GridNode->AddActor_Dormancy(ActorInfo, GlobalInfo);
    }
    else if (ACharacter* Hero = Cast<ACharacter>(ActorInfo.Actor))
    {
        GridNode->AddActor_Dormancy(ActorInfo, GlobalInfo);
    }
}
```

### 7.2 技能预测

为关键技能添加客户端预测：

```cpp
// MobaDashAbility.h
UCLASS()
class UMobaDashAbility : public ULyraGameplayAbility
{
    GENERATED_BODY()
    
protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, 
                                 const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    
    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, 
                                    const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, 
                                    FGameplayTagContainer* OptionalRelevantTags) const override;
    
    // 客户端预测
    virtual void InputReleased(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, 
                               const FGameplayAbilityActivationInfo ActivationInfo) override;
    
    UPROPERTY(EditDefaultsOnly, Category="Dash")
    float DashDistance = 600.0f;
    
    UPROPERTY(EditDefaultsOnly, Category="Dash")
    float DashDuration = 0.3f;
    
    UFUNCTION()
    void OnDashComplete();
};

// 实现（使用 Root Motion 实现位移）
void UMobaDashAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, 
                                       const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
        return;
    }
    
    ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
    if (!Character)
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
        return;
    }
    
    // 计算冲刺方向
    FVector DashDirection = Character->GetActorForwardVector();
    FVector DashDestination = Character->GetActorLocation() + DashDirection * DashDistance;
    
    // 使用 Root Motion Source 实现位移（自动支持客户端预测）
    UAbilityTask_ApplyRootMotionConstantForce* DashTask = UAbilityTask_ApplyRootMotionConstantForce::ApplyRootMotionConstantForce(
        this,
        FName("Dash"),
        DashDirection,
        DashDistance / DashDuration,
        DashDuration,
        false,
        nullptr,
        ERootMotionFinishVelocityMode::SetVelocity,
        FVector::ZeroVector,
        0.0f,
        false
    );
    
    DashTask->OnFinish.AddDynamic(this, &UMobaDashAbility::OnDashComplete);
    DashTask->ReadyForActivation();
}

void UMobaDashAbility::OnDashComplete()
{
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}
```

---

## 八、完整项目示例

### 8.1 Experience 配置

创建 `MobaExperience` 数据资产：

```cpp
// B_MobaExperience.uasset（蓝图配置）

Experience Definition: B_MobaExperience
├── Game Feature Actions:
│   ├── AddComponents
│   │   ├── MobaHeroComponent
│   │   ├── MobaEconomyComponent
│   │   └── MobaVisionComponent
│   │
│   ├── AddAbilities
│   │   └── Ability Sets: MobaHeroAbilitySet
│   │
│   └── AddInputConfig
│       └── Input Mapping: MobaInputConfig
│
├── Default Pawn Data: MobaHeroPawnData
├── Action Sets:
│   ├── MobaTowerSpawner (生成防御塔)
│   ├── MobaMinionSpawner (生成小兵)
│   └── MobaMapManager (初始化地图)
│
└── Game Phases:
    ├── HeroSelectPhase (30秒选英雄)
    ├── BattlePhase (对战阶段)
    └── EndGamePhase (结算)
```

### 8.2 测试流程

1. **编译代码**
```bash
# 在 Unreal 编辑器中
Tools → Compile → Compile C++ Code
```

2. **配置地图**
- 在 Content Browser 中创建 `Map_Moba` 地图
- 放置 6 个 `MobaTowerActor`（每队 3 个塔）
- 放置 3 个 `MobaMinionSpawner`（每路 1 个）
- 配置 Nav Mesh Bounds Volume

3. **创建英雄资产**
- 创建 `Hero_Warrior` Data Asset
- 配置基础属性、技能、模型

4. **启动测试**
```bash
# 命令行启动服务器
UE5Editor.exe "MobaProject.uproject" Map_Moba -server -log

# 命令行启动客户端
UE5Editor.exe "MobaProject.uproject" 127.0.0.1 -game -log
```

5. **验证功能**
- ✅ 英雄选择
- ✅ 小兵生成
- ✅ 防御塔攻击
- ✅ 金币获取
- ✅ 商店购买
- ✅ 技能释放
- ✅ 推塔胜利

---

## 九、性能指标

### 9.1 目标性能

| 指标 | 目标值 | 说明 |
|------|--------|------|
| 服务器帧率 | 60 FPS | 10 玩家 + 100 小兵 |
| 客户端帧率 | 60 FPS | 中等配置 |
| 网络带宽 | < 50 KB/s | 每客户端 |
| 同步延迟 | < 50 ms | 技能释放 |
| 内存占用 | < 2 GB | 服务器端 |

### 9.2 性能分析

使用 Unreal Insights 分析瓶颈：

```cpp
// 在关键函数添加性能追踪
SCOPE_CYCLE_COUNTER(STAT_MobaTowerTick);
SCOPE_CYCLE_COUNTER(STAT_MobaMinionAI);
SCOPE_CYCLE_COUNTER(STAT_MobaVisionUpdate);
```

---

## 十、扩展方向

完成基础 MOBA 模式后，可以继续扩展：

### 10.1 进阶功能

1. **装备系统增强**
   - 主动装备（可释放）
   - 装备特效（吸血、暴击）
   - 装备配方推荐

2. **技能系统扩展**
   - 技能连招检测
   - 技能伤害分析
   - 技能皮肤系统

3. **地图机制**
   - 大小龙野怪
   - Buff 加成
   - 河道争夺

4. **社交功能**
   - 语音聊天
   - 快捷信号
   - 表情系统

### 10.2 商业化功能

1. **皮肤系统**
2. **战斗通行证**
3. **排位赛系统**
4. **战绩分析**

---

## 总结

通过本章，你已经实现了一个完整的 5v5 MOBA 游戏模式，涵盖：

✅ **核心系统**
- 英雄系统与技能升级
- 经济系统与商店
- 防御塔与小兵 AI
- 地图管理与视野系统

✅ **网络优化**
- Replication Graph 相关性过滤
- 技能客户端预测
- 带宽优化策略

✅ **数据驱动**
- 所有内容通过 Data Assets 配置
- 支持热更新
- 易于平衡性调整

**下一步建议**：
1. 继续完善游戏细节（UI、音效、特效）
2. 添加更多英雄和技能
3. 优化网络性能（目标 100 玩家）
4. 制作精美地图和模型资源
5. 进行大规模玩家测试

---

## 附录：完整代码仓库

完整项目代码已上传至 GitHub：

```
https://github.com/YourName/LyraMobaProject
```

包含：
- 完整 C++ 源码
- 蓝图资产
- 示例地图
- 配置文件
- 单元测试

**Star 支持我们！** ⭐

---

**恭喜完成 MOBA 模式开发！** 🎉

你现在具备了独立开发大型多人在线游戏的能力。继续学习，挑战更复杂的项目吧！

---

## 参考资料

1. [Lyra 官方文档](https://docs.unrealengine.com/5.0/lyra-sample-game-in-unreal-engine/)
2. [Gameplay Ability System 文档](https://docs.unrealengine.com/5.0/gameplay-ability-system-for-unreal-engine/)
3. [Replication Graph 优化指南](https://docs.unrealengine.com/5.0/replication-graph-in-unreal-engine/)
4. [MOBA 游戏设计理论](https://www.gamasutra.com/blogs/author/JoshBycer/881713/)

---

*本教程持续更新中，欢迎提交 Issue 和 PR！*
