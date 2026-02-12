# UE5 Lyra 系列教程（三）：Experience System 核心机制

> **作者**: lobsterchen  
> **创建时间**: 2025-02-12  
> **系列**: UE5 Lyra 深度解析  
> **难度**: ⭐⭐⭐⭐ 进阶  
> **预计阅读时间**: 25 分钟

---

## 📚 目录

- [为什么需要 Experience System？](#为什么需要-experience-system)
- [核心概念速览](#核心概念速览)
- [Experience Definition 深度解析](#experience-definition-深度解析)
- [Experience Manager 加载流程](#experience-manager-加载流程)
- [Game Feature Actions 执行机制](#game-feature-actions-执行机制)
- [实战：从零构建一个 Experience](#实战从零构建一个-experience)
- [调试与性能优化](#调试与性能优化)
- [总结](#总结)

---

## 🤔 为什么需要 Experience System？

### 传统游戏模式的痛点

假设你要做一款多模式游戏（类似 Fortnite），包含：
- 大逃杀模式（Battle Royale）
- 团队死斗（Team Deathmatch）
- 夺旗模式（Capture the Flag）
- PvE 合作模式（Co-op）

**传统做法会遇到什么问题？**

```cpp
// ❌ 传统方式：为每个模式创建独立的 GameMode 类
class ABattleRoyaleGameMode : public AGameModeBase
{
    // 包含所有 BR 逻辑、地图配置、规则...
};

class ATeamDeathmatchGameMode : public AGameModeBase
{
    // 包含所有 TDM 逻辑、地图配置、规则...
};

class ACaptureFlagGameMode : public AGameModeBase
{
    // 包含所有 CTF 逻辑、地图配置、规则...
};
```

**问题显而易见**：
1. **代码重复**：每个 GameMode 都要重新实现 UI、计分板、重生系统...
2. **内容耦合**：地图、角色、武器都硬编码在 GameMode 中
3. **无法热更新**：改规则 = 重新编译 C++ 代码
4. **扩展困难**：想加个新模式？复制粘贴几千行代码
5. **策划无法独立工作**：所有配置都需要程序员修改

### Experience System 的解决方案

Lyra 的核心思想：**把"游戏模式"变成数据资产**。

```
传统思路：GameMode = C++ 类（硬编码）
Lyra 思路：Experience = 数据资产（可配置）
```

一个 Experience 就像一个"剧本"，它告诉引擎：
- 📦 需要加载哪些插件（Game Features）
- 🎮 使用什么规则（Pawn、Controller、PlayerState）
- 🗺️ 加载哪些地图和资源
- ⚙️ 应用什么游戏设置（重生时间、得分规则...）

**最重要的是**：这一切都是**数据驱动**的，策划可以在编辑器中点点鼠标就创建新模式，无需写代码。

---

## 🧩 核心概念速览

在深入细节之前，先理解这几个核心概念：

| 概念 | 类型 | 作用 | 示例 |
|------|------|------|------|
| **Experience Definition** | Data Asset | 定义一个完整的游戏体验 | `B_LyraShooterGame_Elimination` |
| **Experience Manager** | Subsystem | 负责加载和管理 Experience | `ULyraExperienceManagerComponent` |
| **Game Feature Plugin** | 插件 | 可动态加载的内容包 | `ShooterCore`、`TopDownArena` |
| **Game Feature Action** | 数据 | Experience 加载时执行的操作 | 添加组件、赋予技能、加载 UI... |
| **Action Set** | Data Asset | 一组可复用的 Actions | 默认输入、基础 UI、通用规则 |
| **Default Pawn Data** | Data Asset | 定义角色的能力和属性 | `HeroData_ShooterGame` |

### 它们之间的关系

```
Experience Definition (顶层配置)
    ├── Game Feature Plugins (依赖的插件)
    │   └── ShooterCore.uplugin
    ├── Action Sets (复用的操作集合)
    │   ├── LAS_InventoryTest (测试用背包)
    │   └── LAS_ShooterGame_SharedInput (通用输入)
    ├── Actions (本 Experience 特有的操作)
    │   ├── AddComponents (添加组件到 Pawn)
    │   ├── AddAbilities (赋予技能)
    │   └── AddInputConfig (配置输入映射)
    └── Default Pawn Data (角色数据)
        └── HeroData_ShooterGame
            ├── Ability Sets (技能集合)
            ├── Input Config (输入配置)
            └── Camera Mode (相机模式)
```

---

## 📋 Experience Definition 深度解析

### 数据结构

```cpp
// LyraExperienceDefinition.h

UCLASS(BlueprintType)
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 1. 依赖的 Game Feature 插件列表
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TArray<FString> GameFeaturesToEnable;

    // 2. 默认的 Pawn 数据（定义角色能力）
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TObjectPtr<const ULyraPawnData> DefaultPawnData;

    // 3. 可复用的 Action Sets
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets;

    // 4. 本 Experience 特有的 Actions
    UPROPERTY(EditDefaultsOnly, Instanced, Category = "Actions")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;
};
```

### 实际例子：Elimination 模式

让我们看看 Lyra 的"死斗模式" Experience：

```
B_LyraShooterGame_Elimination (Experience Definition)
│
├── GameFeaturesToEnable:
│   └── "ShooterCore"  // 射击游戏核心插件
│
├── DefaultPawnData:
│   └── HeroData_ShooterGame
│       ├── PawnClass: B_Hero_ShooterMannequin (蓝图类)
│       ├── AbilitySets:
│       │   └── AbilitySet_ShooterHero (跳跃、冲刺、射击...)
│       ├── InputConfig: InputData_Hero (WASD、鼠标、技能键...)
│       └── CameraMode: CM_ThirdPerson
│
├── ActionSets:
│   ├── LAS_ShooterGame_StandardComponents
│   │   └── 添加：HealthComponent、EquipmentManager、HeroComponent
│   ├── LAS_ShooterGame_StandardHUD
│   │   └── 添加：准星、血条、弹药显示
│   └── LAS_ShooterGame_SharedInput
│       └── 绑定：跳跃、开火、换武器...
│
└── Actions (本模式特有):
    ├── AddGameRules
    │   └── Elimination 规则（击杀得分、重生机制）
    ├── AddTeamSetup
    │   └── 队伍配置（2队或FFA）
    └── AddUILayout
        └── 记分板、击杀提示
```

### 为什么这样设计？

注意 **ActionSets** 和 **Actions** 的区别：

- **ActionSets**：可以在多个 Experience 之间共享
  - 例如 `LAS_ShooterGame_SharedInput` 可以被 Elimination、Control、TDM 等模式复用
  - 修改一次，所有模式都生效
  
- **Actions**：本 Experience 独有
  - 例如 Elimination 的"击杀得分"规则，其他模式不需要

这种设计实现了**最大化复用 + 最小化冗余**。

---

## 🔄 Experience Manager 加载流程

### 生命周期概览

```
1. 服务器启动
   └─→ GameMode::InitGame()
        └─→ ExperienceManager->StartExperienceLoad()

2. 加载 Experience Definition
   └─→ 读取数据资产
        └─→ 解析 GameFeaturesToEnable 列表

3. 激活 Game Features
   └─→ 依次加载插件（异步）
        └─→ ShooterCore.uplugin
        └─→ 等待所有插件加载完成

4. 执行 Actions
   └─→ 遍历 ActionSets
   └─→ 遍历 Actions
   └─→ 执行每个 Action 的 OnGameFeatureActivating()

5. 通知游戏逻辑
   └─→ 广播 OnExperienceLoaded 事件
        └─→ GameState、PlayerController、UI 等开始初始化

6. 玩家加入
   └─→ 分配 Pawn
        └─→ 应用 DefaultPawnData
        └─→ 赋予 AbilitySets
        └─→ 配置 InputConfig
        └─→ 设置 CameraMode

7. 游戏开始！
```

### 源码分析：关键函数

#### 1. 开始加载 Experience

```cpp
// LyraExperienceManagerComponent.cpp

void ULyraExperienceManagerComponent::StartExperienceLoad()
{
    // 从 URL 参数或配置中获取 Experience ID
    FPrimaryAssetId ExperienceId = /* ... */;
    
    // 异步加载 Experience Definition 资产
    TSubclassOf<ULyraExperienceDefinition> ExperienceClass = /* ... */;
    
    UAssetManager& AssetManager = UAssetManager::Get();
    AssetManager.GetPrimaryAssetData(ExperienceId, /* 回调 */);
    
    // 异步加载完成后调用 OnExperienceLoadComplete()
}
```

#### 2. 激活 Game Features

```cpp
void ULyraExperienceManagerComponent::OnExperienceLoadComplete()
{
    const ULyraExperienceDefinition* Experience = LoadedExperience;
    
    // 激活依赖的 Game Feature 插件
    for (const FString& PluginName : Experience->GameFeaturesToEnable)
    {
        UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(
            PluginName,
            FGameFeaturePluginLoadComplete::CreateUObject(this, &ThisClass::OnGameFeaturePluginLoadComplete)
        );
    }
    
    // 等待所有插件加载完成...
}
```

#### 3. 执行 Actions

```cpp
void ULyraExperienceManagerComponent::OnAllGameFeaturesLoaded()
{
    const ULyraExperienceDefinition* Experience = LoadedExperience;
    
    // 1. 执行 ActionSets 中的 Actions
    for (const ULyraExperienceActionSet* ActionSet : Experience->ActionSets)
    {
        for (UGameFeatureAction* Action : ActionSet->Actions)
        {
            Action->OnGameFeatureActivating();
        }
    }
    
    // 2. 执行 Experience 自己的 Actions
    for (UGameFeatureAction* Action : Experience->Actions)
    {
        Action->OnGameFeatureActivating();
    }
    
    // 3. 广播加载完成事件
    OnExperienceLoaded.Broadcast(Experience);
    
    // 4. 所有等待 Experience 的对象现在可以初始化了
    // (例如 PlayerController、GameState 等)
}
```

### 网络同步

**重要**：Experience 的加载是**服务器驱动**的，客户端会自动同步：

```cpp
// LyraGameState.h

UCLASS()
class ALyraGameState : public AModularGameStateBase
{
    // 通过网络同步的当前 Experience
    UPROPERTY(Replicated)
    TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;
    
    // 客户端接收到同步后，自动开始加载相同的 Experience
    UFUNCTION()
    void OnRep_CurrentExperience();
};
```

---

## ⚙️ Game Feature Actions 执行机制

### 什么是 Game Feature Action？

**Game Feature Action** 是一个可执行的操作，当 Experience 加载时，这些 Actions 会被依次执行。

Lyra 内置的 Actions：

| Action 类 | 功能 | 使用场景 |
|-----------|------|---------|
| **AddComponents** | 添加 Component 到指定 Actor | 给 Pawn 添加 HealthComponent |
| **AddAbilities** | 赋予 AbilitySet 到 ASC | 给角色添加跳跃、射击技能 |
| **AddInputConfig** | 配置输入映射 | 绑定 WASD、鼠标操作 |
| **AddWidgets** | 添加 UI Widget | 显示血条、准星、记分板 |
| **AddCheats** | 注册作弊命令 | 开发调试用 |
| **AddDataRegistry** | 注册数据表 | 加载武器、道具配置 |

### 实战：AddComponents Action

让我们看看如何给 Pawn 动态添加 Component：

```cpp
// GameFeatureAction_AddComponents.h

UCLASS()
class UGameFeatureAction_AddComponents : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    // 定义要添加的 Component 列表
    UPROPERTY(EditAnywhere, Category = "Components")
    TArray<FGameFeatureComponentEntry> ComponentList;
};

USTRUCT()
struct FGameFeatureComponentEntry
{
    // 目标 Actor 类（例如 ALyraCharacter）
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<AActor> ActorClass;
    
    // 要添加的 Component 类（例如 ULyraHealthComponent）
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UActorComponent> ComponentClass;
    
    // Component 是否需要网络同步
    UPROPERTY(EditAnywhere)
    bool bClientComponent = false;
    
    UPROPERTY(EditAnywhere)
    bool bServerComponent = true;
};
```

**执行时机**：

```cpp
void UGameFeatureAction_AddComponents::OnGameFeatureActivating()
{
    // 遍历当前世界中所有匹配的 Actor
    for (const FGameFeatureComponentEntry& Entry : ComponentList)
    {
        UClass* ActorClass = Entry.ActorClass.Get();
        UClass* ComponentClass = Entry.ComponentClass.Get();
        
        // 找到所有符合条件的 Actor
        for (TActorIterator<AActor> It(World, ActorClass); It; ++It)
        {
            AActor* Actor = *It;
            
            // 动态添加 Component
            UActorComponent* NewComponent = NewObject<UActorComponent>(
                Actor,
                ComponentClass,
                NAME_None,
                RF_Transient
            );
            
            Actor->AddInstanceComponent(NewComponent);
            NewComponent->RegisterComponent();
        }
    }
    
    // 同时注册一个 Delegate，当新 Actor Spawn 时也自动添加
    FGameFrameworkComponentManager::Get().RegisterComponentInitCallback(
        ActorClass,
        FComponentInitDelegate::CreateUObject(this, &ThisClass::HandleActorExtension)
    );
}
```

### 实战：AddAbilities Action

```cpp
// GameFeatureAction_AddAbilities.h

UCLASS()
class UGameFeatureAction_AddAbilities : public UGameFeatureAction
{
    UPROPERTY(EditAnywhere)
    TArray<FGameFeatureAbilitiesEntry> AbilitiesList;
};

USTRUCT()
struct FGameFeatureAbilitiesEntry
{
    // 目标 Actor（通常是 Pawn）
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<AActor> ActorClass;
    
    // 要赋予的技能集合
    UPROPERTY(EditAnywhere)
    TArray<TObjectPtr<const ULyraAbilitySet>> GrantedAbilitySets;
};
```

**执行流程**：

1. 找到 Actor 的 AbilitySystemComponent
2. 遍历 `GrantedAbilitySets`
3. 调用 `ASC->GiveAbilitySet()` 赋予技能
4. 记录 Handle，卸载时可以撤销

---

## 🛠️ 实战：从零构建一个 Experience

现在我们动手创建一个自定义的 Experience：**1v1 决斗模式**。

### 需求分析

- 🎯 **玩法**：两名玩家在小竞技场中对战，先得 5 杀获胜
- ⚔️ **武器**：只能使用手枪和霰弹枪
- ❤️ **生命值**：100 HP，死亡后 3 秒重生
- 🚫 **特殊规则**：禁用冲刺和跳跃

### Step 1: 创建 Experience Definition

1. 在 Content Browser 中右键 → **Miscellaneous** → **Data Asset**
2. 选择 `LyraExperienceDefinition` 作为父类
3. 命名为 `B_DuelMode`

### Step 2: 配置基础参数

打开 `B_DuelMode`，设置：

```
GameFeaturesToEnable:
    - ShooterCore  // 复用射击核心插件

DefaultPawnData:
    - HeroData_Duel (新建一个精简的 PawnData)

ActionSets:
    - LAS_ShooterGame_StandardComponents  // 复用基础组件
    - LAS_DuelMode_CustomRules (新建，包含决斗规则)
```

### Step 3: 创建 Pawn Data

创建 `HeroData_Duel`（Data Asset → `LyraPawnData`）：

```
PawnClass:
    - B_Hero_ShooterMannequin (复用)

AbilitySets:
    - AbilitySet_DuelMode
        ├── GA_Shoot (射击)
        ├── GA_Reload (换弹)
        └── GA_Melee (近战)
        // 注意：不包含 GA_Sprint 和 GA_Jump

InputConfig:
    - InputData_DuelMode (精简的输入配置)

CameraMode:
    - CM_ThirdPerson
```

### Step 4: 创建自定义 Action Set

创建 `LAS_DuelMode_CustomRules`：

```cpp
// 在 Action Set 中添加以下 Actions

1. AddComponents
   ├─ Target: ALyraCharacter
   └─ Component: UDuelModeComponent
       └─ 负责：
           - 限制武器选择（只能装备手枪/霰弹枪）
           - 监听击杀事件
           - 检查胜利条件（5 杀）

2. AddWidgets
   ├─ W_DuelScoreboard (自定义记分板)
   └─ W_DuelTimer (回合计时器)

3. AddGameRules
   └─ DuelModeRules Data Asset
       ├─ MaxKills: 5
       ├─ RespawnDelay: 3.0
       └─ AllowedWeapons: [Pistol, Shotgun]
```

### Step 5: 实现 Duel Mode Component

```cpp
// DuelModeComponent.h

UCLASS()
class UDuelModeComponent : public UPawnComponent
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;

protected:
    // 监听击杀事件
    UFUNCTION()
    void OnKillScored(AActor* Killer, AActor* Victim);
    
    // 检查胜利条件
    void CheckVictoryCondition();
    
    // 限制武器装备
    UFUNCTION()
    void OnWeaponEquipped(ULyraEquipmentInstance* Equipment);

private:
    UPROPERTY()
    int32 Player1Kills = 0;
    
    UPROPERTY()
    int32 Player2Kills = 0;
    
    UPROPERTY(EditDefaultsOnly)
    int32 KillsToWin = 5;
};
```

```cpp
// DuelModeComponent.cpp

void UDuelModeComponent::BeginPlay()
{
    Super::BeginPlay();
    
    // 绑定击杀事件
    if (ALyraPlayerState* PS = GetPlayerState<ALyraPlayerState>())
    {
        PS->OnKillScoredDelegate.AddDynamic(this, &ThisClass::OnKillScored);
    }
    
    // 绑定武器装备事件
    if (ULyraEquipmentManagerComponent* EquipMgr = GetEquipmentManager())
    {
        EquipMgr->OnEquipmentEquipped.AddDynamic(this, &ThisClass::OnWeaponEquipped);
    }
}

void UDuelModeComponent::OnKillScored(AActor* Killer, AActor* Victim)
{
    // 增加击杀计数
    if (ALyraPlayerState* KillerPS = Cast<ALyraPlayerState>(Killer))
    {
        if (KillerPS->GetPlayerIndex() == 0)
            Player1Kills++;
        else
            Player2Kills++;
        
        // 检查胜利条件
        CheckVictoryCondition();
    }
}

void UDuelModeComponent::CheckVictoryCondition()
{
    if (Player1Kills >= KillsToWin)
    {
        // Player 1 获胜！
        BroadcastVictory(0);
    }
    else if (Player2Kills >= KillsToWin)
    {
        // Player 2 获胜！
        BroadcastVictory(1);
    }
}

void UDuelModeComponent::OnWeaponEquipped(ULyraEquipmentInstance* Equipment)
{
    // 检查武器是否在允许列表中
    const FGameplayTag WeaponTag = Equipment->GetItemTag();
    
    if (!WeaponTag.MatchesTag(TAG_Weapon_Pistol) && 
        !WeaponTag.MatchesTag(TAG_Weapon_Shotgun))
    {
        // 不允许的武器，立即卸载
        Equipment->Destroy();
        UE_LOG(LogDuelMode, Warning, TEXT("武器 %s 在决斗模式中被禁用"), *WeaponTag.ToString());
    }
}
```

### Step 6: 配置 Playlist

创建 `DA_Playlist_Duel`：

```
FrontEndExperience:
    - B_LyraFrontEnd_Experience (复用主菜单)

Entries:
    [0]:
        Experience: B_DuelMode
        MapID: L_DuelArena (创建一个小竞技场地图)
```

### Step 7: 测试

1. 打开 `L_DuelArena` 地图
2. 在 World Settings 中设置：
   - **GameMode Override**: `LyraGameMode`
   - **Default Experience**: `B_DuelMode`
3. PIE 设置：
   - Number of Players: 2
   - Net Mode: Play As Listen Server
4. 运行游戏，验证：
   - ✅ 只能装备手枪和霰弹枪
   - ✅ 无法使用冲刺和跳跃
   - ✅ 击杀计数正确
   - ✅ 达到 5 杀时显示胜利界面

---

## 🐛 调试与性能优化

### 调试命令

Lyra 提供了丰富的调试工具：

```cpp
// Console Commands

// 1. 显示当前 Experience 信息
Lyra.DumpExperience

// 2. 显示所有 Game Feature 状态
GameFeatures.DumpPluginStatus

// 3. 显示 Action 执行顺序
Lyra.ShowExperienceActions

// 4. 重新加载 Experience（开发时很有用）
Lyra.ReloadExperience B_DuelMode
```

### 常见问题排查

#### 问题1：Experience 加载卡住

**症状**：游戏卡在"Loading Experience"界面

**排查步骤**：
1. 打开 Output Log，搜索 `Error` 或 `Warning`
2. 检查 Game Feature 插件是否都启用了
3. 使用 `GameFeatures.DumpPluginStatus` 查看哪个插件加载失败
4. 确认 DefaultPawnData 的资产引用是否有效

#### 问题2：Action 没有执行

**症状**：Component 没有被添加，或技能没有赋予

**排查步骤**：
1. 在 Action 的 `OnGameFeatureActivating()` 中打断点
2. 检查 `ActorClass` 是否匹配（使用 `IsA()` 判断）
3. 确认 Actor 的 InitState 是否已到达要求的状态
4. 查看 `GameFrameworkComponentManager` 日志

#### 问题3：网络同步异常

**症状**：客户端 Experience 与服务器不一致

**原因**：Experience Definition 必须在客户端和服务器都存在

**解决方案**：
- 确保 Experience 资产已打包（添加到 AssetManager 的搜索路径）
- 检查 `DefaultGame.ini` 中的 Primary Asset 配置

### 性能优化建议

#### 1. 延迟加载非关键资源

```cpp
// ❌ 不好：同步加载所有资源
UPROPERTY(EditDefaultsOnly)
TArray<UTexture2D*> CosmeticTextures;

// ✅ 更好：异步加载
UPROPERTY(EditDefaultsOnly)
TArray<TSoftObjectPtr<UTexture2D>> CosmeticTextures;

// 需要时再加载
AsyncLoad(CosmeticTextures[0].ToSoftObjectPath(), ...);
```

#### 2. 复用 Action Sets

不要在每个 Experience 中重复配置相同的 Actions，提取到 Action Set 中：

```
// ❌ 不好：每个 Experience 都配置一遍
B_ModeA → Actions: [AddHealth, AddEquipment, AddUI]
B_ModeB → Actions: [AddHealth, AddEquipment, AddUI]
B_ModeC → Actions: [AddHealth, AddEquipment, AddUI]

// ✅ 更好：复用 Action Set
LAS_CommonGameplay → Actions: [AddHealth, AddEquipment, AddUI]
B_ModeA → ActionSets: [LAS_CommonGameplay]
B_ModeB → ActionSets: [LAS_CommonGameplay]
B_ModeC → ActionSets: [LAS_CommonGameplay]
```

#### 3. 控制 Game Feature 粒度

避免创建过多小插件，也不要创建超大插件：

```
// ❌ 不好：粒度太细
ShooterCore_Movement.uplugin
ShooterCore_Weapons.uplugin
ShooterCore_Abilities.uplugin
ShooterCore_UI.uplugin

// ❌ 也不好：粒度太粗
ShooterCore_Everything.uplugin (包含所有内容)

// ✅ 合适：按功能模块划分
ShooterCore.uplugin (核心玩法)
ShooterMaps.uplugin (地图资源)
ShooterCosmetics.uplugin (装饰品，DLC)
```

---

## 💬 总结

### 核心要点回顾

1. **Experience System 是什么？**
   - 一套数据驱动的游戏模式系统
   - 通过组装 Game Features 和 Actions 来定义游戏体验

2. **为什么它重要？**
   - 策划可以无需编程创建新模式
   - 最大化代码复用，减少冗余
   - 支持热更新和 DLC
   - 网络友好的加载机制

3. **核心组件**
   - **Experience Definition**：游戏模式的"剧本"
   - **Experience Manager**：负责加载和管理
   - **Game Feature Actions**：可执行的操作单元
   - **Action Sets**：可复用的操作集合

4. **最佳实践**
   - 提取通用逻辑到 Action Sets
   - 使用 Game Feature 插件隔离内容
   - 优先使用数据配置而非硬编码
   - 充分利用调试工具排查问题

### 与 Modular Gameplay Actors 的联系

还记得上一篇的 **初始化状态机** 吗？

```
Experience 加载完成
    ↓
触发 OnExperienceLoaded 事件
    ↓
Pawn 的 InitState 从 DataAvailable → DataInitialized
    ↓
应用 DefaultPawnData
    ↓
赋予 AbilitySets
    ↓
配置 InputConfig
    ↓
Pawn 进入 GameplayReady 状态
```

**Experience System 和 Modular Gameplay Actors 是一体的**：
- Experience 定义"加载什么"
- Modular Actors 定义"如何初始化"
- 两者配合，实现了完全解耦的架构

### 下一步

在下一篇文章中，我们将深入 **Game Features 插件系统**：

- 📌 Game Feature 插件的生命周期
- 📌 如何创建一个自定义 Game Feature
- 📌 动态加载/卸载机制
- 📌 与 Experience 的集成细节
- 📌 实战：开发一个"季节性活动"插件

准备好探索 Lyra 的插件化架构了吗？🚀

---

## 📚 参考资料

- [UE5 Game Features 文档](https://docs.unrealengine.com/5.0/en-US/game-features-and-modular-gameplay-in-unreal-engine/)
- [Lyra 源码：ExperienceManagerComponent.cpp](Source/LyraGame/GameModes/)
- [Epic Developer Community - Experience System 解析](https://dev.epicgames.com/community/)

---

> **本文是《UE5 Lyra 深度解析》系列教程的第 3 篇**  
> 上一篇：[Modular Gameplay Actors 详解](02_modular_gameplay_actors.md)  
> 下一篇：《Lyra 系列教程（四）：Game Features 插件系统深度剖析》  
> 作者：lobsterchen | 欢迎分享你的自定义 Experience！
