# Lyra 项目概述与环境搭建

> **系列文章**: Lyra Deep Dive - 深入解析 Unreal Engine 5 的现代游戏架构  
> **作者**: OpenClaw AI Assistant  
> **版本**: UE 5.5  
> **难度**: ⭐⭐ (入门到进阶)  
> **预计阅读时间**: 60-90 分钟

---

## 📑 目录

1. [引言：为什么要学习 Lyra？](#引言为什么要学习-lyra)
2. [Lyra 项目概述](#lyra-项目概述)
3. [核心架构设计理念](#核心架构设计理念)
4. [项目结构深度解析](#项目结构深度解析)
5. [环境搭建详细指南](#环境搭建详细指南)
6. [第一次运行 Lyra](#第一次运行-lyra)
7. [开发环境配置](#开发环境配置)
8. [常见问题与解决方案](#常见问题与解决方案)
9. [最佳实践与建议](#最佳实践与建议)
10. [总结与下一步](#总结与下一步)

---

## 引言：为什么要学习 Lyra？

### Lyra 的历史背景

Lyra Starter Game 是 Epic Games 在 2022 年随 Unreal Engine 5.0 发布的官方示例项目。与传统的演示项目不同，Lyra 不仅仅是一个"技术展示"，而是一个**生产级的游戏框架**，它融合了 Epic Games 多年来在《堡垒之夜》(Fortnite)、《火箭联盟》(Rocket League) 等大型多人在线游戏中积累的架构经验。

到了 UE 5.5 版本，Lyra 已经成为学习现代游戏开发的**黄金标准**：

- **模块化设计**: 完全基于组件和插件系统，代码复用率极高
- **数据驱动**: 几乎所有游戏逻辑都通过 Data Assets 配置，无需硬编码
- **网络优化**: 内置 Replication Graph、预测机制等先进网络技术
- **可扩展性**: 支持从单机小游戏到大型 MMO 的各种规模

### 适合谁学习？

- **UE 初学者**: 想要学习工业级项目结构的开发者
- **独立开发者**: 需要快速搭建原型和可扩展架构的团队
- **技术美术**: 想要理解程序与美术工作流衔接的设计师
- **网络程序员**: 关注多人游戏同步和性能优化的工程师
- **架构师**: 研究大型项目代码组织和模式的技术负责人

### 学习目标

通过本系列文章，你将：

1. **深入理解** Lyra 的核心架构和设计模式
2. **掌握** UE5 的现代开发技术栈 (GAS、Enhanced Input、Common UI 等)
3. **学会** 如何基于 Lyra 快速开发自己的游戏
4. **避开** 常见的架构陷阱和性能问题
5. **建立** 可维护、可扩展的代码习惯

---

## Lyra 项目概述

### 什么是 Lyra？

Lyra 是一个**多类型游戏原型框架**，它并不是一个特定类型的游戏（如 FPS 或 TPS），而是一个可以快速切换游戏模式的平台。你可以在同一个项目中：

- 玩一局第三人称射击 (TPS)
- 切换到第一人称模式 (FPS)
- 体验自上而下的 Top-Down 模式
- 甚至开发 MOBA、大逃杀、Racing 等完全不同的玩法

这种灵活性来源于 Lyra 的核心设计哲学：**游戏模式即数据**（Game as Data）。

### Lyra 的核心特性

#### 1. **Experience 系统**

Experience（体验）是 Lyra 最重要的概念，它定义了一个完整的游戏模式：

- 使用什么 Pawn（玩家角色）
- 加载哪些 Game Features（游戏功能插件）
- 应用什么输入映射和 UI 布局
- 包含哪些游戏规则和阶段管理

```cpp
// 示例：Experience Definition 的核心结构
UCLASS()
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 要加载的游戏功能插件列表
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TArray<FString> GameFeaturesToEnable;

    // 默认的 Pawn 数据
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TObjectPtr<ULyraPawnData> DefaultPawnData;

    // 动作列表（加载时执行）
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    // 体验的友好名称
    UPROPERTY(EditDefaultsOnly, Category = "UI")
    FText DisplayName;
};
```

#### 2. **模块化 Actor 组件系统**

Lyra 抛弃了传统的"胖类"设计，采用 **Component-Based Architecture**：

- **Pawn Extension** 组件：为角色添加特定功能（血量、装备、技能等）
- **动态组件加载**：根据游戏模式和配置动态添加/移除组件
- **解耦设计**：组件之间通过 Gameplay Tags 和消息系统通信

```cpp
// 示例：角色组件的初始化流程
void ALyraCharacter::BeginPlay()
{
    Super::BeginPlay();

    // 组件通过 InitState 机制按依赖顺序初始化
    ULyraPawnExtensionComponent* PawnExtComp = FindComponentByClass<ULyraPawnExtensionComponent>();
    if (PawnExtComp)
    {
        PawnExtComp->OnPawnReadyToInitialize.AddDynamic(this, &ALyraCharacter::OnPawnInitialized);
    }
}
```

#### 3. **Gameplay Ability System (GAS)**

Lyra 全面集成了 UE 的 GAS 框架，用于处理：

- **技能系统**：射击、跳跃、治疗等所有游戏动作
- **属性管理**：血量、护甲、移动速度等角色数值
- **效果应用**：伤害、Buff、Debuff 的计算和同步
- **网络预测**：客户端预测 + 服务器权威验证

```cpp
// 示例：通过 GAS 激活一个射击技能
UCLASS()
class ULyraGameplayAbility_RangedWeapon : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
                                  const FGameplayAbilityActorInfo* ActorInfo,
                                  const FGameplayAbilityActivationInfo ActivationInfo,
                                  const FGameplayEventData* TriggerEventData) override
    {
        // 执行技能逻辑
        if (HasAuthority(&ActivationInfo))
        {
            // 服务器：生成子弹，造成伤害
            SpawnProjectile();
        }

        // 播放动画、音效等表现层逻辑
        PlayFireMontage();
        PlayFireSound();

        EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
    }
};
```

#### 4. **Game Features 插件系统**

Game Features 是 UE5 引入的模块化内容管理机制，Lyra 将其发挥到极致：

- 每个游戏模式、地图、角色类型都可以是一个独立的插件
- 插件可以**热加载/卸载**，无需重启游戏
- 支持**依赖管理**：自动加载所需的前置插件
- 可以包含代码、资源、配置、UI 等所有内容

```cpp
// 示例：Game Feature 插件的元数据定义
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "1.0",
    "FriendlyName": "ShooterCore",
    "Description": "射击游戏核心功能包",
    "Category": "Game Features",
    "CreatedBy": "Epic Games",
    "BuiltInInitialFeatureState": "Active",
    "Plugins": [
        {
            "Name": "GameplayAbilities",
            "Enabled": true
        },
        {
            "Name": "ModularGameplayActors",
            "Enabled": true
        }
    ]
}
```

#### 5. **Enhanced Input System**

Lyra 使用 UE5 的新一代输入系统，提供：

- **Input Actions**：抽象的输入动作（"跳跃"、"射击"等）
- **Input Mapping Contexts**：不同场景的输入映射（战斗、UI、载具等）
- **Modifiers & Triggers**：输入修饰符（死区、灵敏度）和触发器（长按、双击等）
- **动态切换**：根据游戏状态动态改变输入映射

```cpp
// 示例：绑定 Enhanced Input Actions
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    if (UEnhancedInputComponent* EnhancedInput = Cast<UEnhancedInputComponent>(PlayerInputComponent))
    {
        // 绑定移动输入
        EnhancedInput->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyCharacter::Move);

        // 绑定跳跃输入
        EnhancedInput->BindAction(JumpAction, ETriggerEvent::Started, this, &AMyCharacter::Jump);
    }
}
```

#### 6. **Common UI Framework**

Lyra 的 UI 系统基于 **Common UI** 插件，它提供：

- **多平台适配**：自动处理 PC、Console、Mobile 的输入和布局
- **焦点管理**：智能的控制器导航和焦点系统
- **UI 扩展点**：通过 Extension Points 动态插入 UI 组件
- **数据绑定**：Model-View 分离，UI 自动响应数据变化

```cpp
// 示例：Common UI 激活栈的使用
UCLASS()
class UMyGameUIManagerSubsystem : public UGameUIManagerSubsystem
{
    GENERATED_BODY()

public:
    void ShowMainMenu()
    {
        // 将主菜单推入 UI 栈
        UCommonActivatableWidget* MenuWidget = CreateWidget<UCommonActivatableWidget>(
            GetWorld(), MainMenuClass);

        RootLayout->AddWidget(MenuWidget);
        MenuWidget->ActivateWidget();  // 自动处理焦点和输入
    }
};
```

#### 7. **网络优化架构**

Lyra 针对多人游戏进行了深度优化：

- **Replication Graph**：智能的网络同步剔除，大幅降低带宽
- **GAS 网络预测**：客户端预测 + Rollback 机制
- **Fast Array Serialization**：优化的数组同步
- **Relevancy 管理**：精细的网络相关性控制

```cpp
// 示例：Lyra 的自定义 Replication Graph
UCLASS()
class ULyraReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()

public:
    virtual void InitGlobalActorClassSettings() override
    {
        Super::InitGlobalActorClassSettings();

        // 配置不同 Actor 的同步策略
        ReplicationGraphGlobalSettings.MaxRelevantActors = 256;
        
        // 玩家角色使用最高优先级
        AddClassReplicationInfo(ALyraCharacter::StaticClass(),
                                EClassRepPolicy::RelevantAllConnections,
                                EPriorityLevel::High);
    }
};
```

---

## 核心架构设计理念

### 1. 数据驱动开发 (Data-Driven Development)

Lyra 的核心哲学是**将代码和内容分离**，让设计师和策划能够在不修改代码的情况下创建新内容。

#### 数据驱动的层次

| 层次 | 说明 | 示例 |
|------|------|------|
| **配置层** | 数值、开关、参数 | 武器伤害、跳跃高度、UI 颜色 |
| **逻辑层** | 游戏规则、流程控制 | 胜利条件、回合机制、重生规则 |
| **内容层** | 完整的游戏模式和关卡 | TDM 模式、生存模式、教程关卡 |
| **系统层** | 可插拔的功能模块 | 背包系统、成就系统、社交系统 |

#### 实现机制

```cpp
// 示例：通过 Data Asset 定义武器属性
UCLASS()
class ULyraWeaponData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 基础属性（策划可直接编辑）
    UPROPERTY(EditDefaultsOnly, Category = "Stats")
    float Damage = 10.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Stats")
    float FireRate = 0.1f;  // 射击间隔（秒）

    UPROPERTY(EditDefaultsOnly, Category = "Stats")
    int32 MagazineSize = 30;

    // 引用其他资源
    UPROPERTY(EditDefaultsOnly, Category = "Assets")
    TObjectPtr<USkeletalMesh> WeaponMesh;

    UPROPERTY(EditDefaultsOnly, Category = "Assets")
    TSubclassOf<UGameplayAbility> PrimaryAbility;  // 射击技能

    // 应用到角色时执行的 GAS Effects
    UPROPERTY(EditDefaultsOnly, Category = "Abilities")
    TArray<TSubclassOf<UGameplayEffect>> EquipEffects;
};
```

**使用场景**：
- 策划在编辑器中创建 `DA_AssaultRifle`、`DA_Shotgun` 等数据资源
- 修改武器数值无需编译代码
- 支持表格批量导入（CSV、Excel → Data Table）

### 2. 组合优于继承 (Composition over Inheritance)

传统 UE 项目常见的"类爆炸"问题：

```cpp
// ❌ 不推荐：继承链过长，难以维护
ACharacter
  └─ AMyCharacter
       ├─ AWarrior
       │    ├─ AWarriorWithShield
       │    └─ AWarriorWithTwoHands
       ├─ AMage
       │    ├─ AFireMage
       │    └─ AIceMage
       └─ ARanger
            ├─ AHunter
            └─ ASniper
```

Lyra 的解决方案：

```cpp
// ✅ 推荐：组件化设计
ALyraCharacter (基类，非常薄)
  + ULyraPawnExtensionComponent        // 核心扩展
  + ULyraHealthComponent               // 血量管理
  + ULyraAbilitySystemComponent        // 技能系统
  + ULyraInventoryComponent            // 背包（可选）
  + ULyraTeamComponent                 // 队伍归属（可选）
  + Custom Components...               // 你的自定义组件
```

**组件的生命周期管理**：

```cpp
// Lyra 使用 Init State 机制控制组件的初始化顺序
enum class ELyraInitState : uint8
{
    Uninitialized,          // 未初始化
    DataAvailable,          // 数据已准备（Pawn Data 已加载）
    DataInitialized,        // 组件数据已初始化
    GameplayReady           // 可以接受输入和交互
};

// 示例：组件等待依赖初始化
void ULyraHealthComponent::OnRegister()
{
    Super::OnRegister();

    // 注册到 Init State 系统
    BindOnActorInitStateChanged(ULyraPawnExtensionComponent::NAME_ActorFeatureName,
                                 FGameplayTag::RequestGameplayTag("InitState.DataAvailable"),
                                 false);
}

void ULyraHealthComponent::OnActorInitStateChanged(const FActorInitStateChangedParams& Params)
{
    if (Params.FeatureName == ULyraPawnExtensionComponent::NAME_ActorFeatureName)
    {
        if (Params.NewState == LyraGameplayTags::InitState_DataAvailable)
        {
            // 依赖的组件已就绪，开始初始化血量属性
            InitializeHealthAttributes();
        }
    }
}
```

### 3. 接口驱动设计 (Interface-Driven Design)

Lyra 大量使用 UE 的 Interface 机制实现松耦合：

```cpp
// 示例：交互接口
UINTERFACE(BlueprintType)
class UInteractableInterface : public UInterface
{
    GENERATED_BODY()
};

class IInteractableInterface
{
    GENERATED_BODY()

public:
    // 获取交互提示文本
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Interaction")
    FText GetInteractionText() const;

    // 执行交互
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Interaction")
    void OnInteract(AActor* Interactor);

    // 是否可以交互
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Interaction")
    bool CanInteract(AActor* Interactor) const;
};

// 武器可以被拾取
UCLASS()
class ALyraWeaponPickup : public AActor, public IInteractableInterface
{
    GENERATED_BODY()

public:
    virtual FText GetInteractionText_Implementation() const override
    {
        return FText::Format(LOCTEXT("PickupWeapon", "拾取 {0}"), WeaponData->DisplayName);
    }

    virtual void OnInteract_Implementation(AActor* Interactor) override
    {
        // 将武器添加到玩家背包
        if (ALyraCharacter* Character = Cast<ALyraCharacter>(Interactor))
        {
            Character->GetInventoryComponent()->AddWeapon(WeaponData);
            Destroy();
        }
    }
};
```

### 4. 事件驱动架构 (Event-Driven Architecture)

Lyra 通过 **Gameplay Tags** 和 **Delegates** 实现解耦的事件系统：

```cpp
// 示例：血量变化事件
DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(
    FLyraHealth_DeathEvent,
    AActor*, OwningActor,
    AActor*, InstigatorActor,
    AActor*, DamageCauser
);

UCLASS()
class ULyraHealthComponent : public UGameFrameworkComponent
{
    GENERATED_BODY()

public:
    // 死亡事件（任何系统都可以监听）
    UPROPERTY(BlueprintAssignable, Category = "Health")
    FLyraHealth_DeathEvent OnDeathStarted;

    void HandleDeath()
    {
        // 广播死亡事件
        OnDeathStarted.Broadcast(GetOwner(), LastInstigator, LastDamageCauser);

        // 通过 Gameplay Tags 也能监听
        FGameplayEventData EventData;
        EventData.Instigator = LastInstigator;
        EventData.Target = GetOwner();

        UAbilitySystemComponent* ASC = GetAbilitySystemComponent();
        ASC->HandleGameplayEvent(LyraGameplayTags::GameplayEvent_Death, &EventData);
    }
};

// 其他系统监听死亡事件
void AMyGameMode::OnCharacterDeath(AActor* DeadActor, AActor* Killer, AActor* DamageCauser)
{
    // 更新分数、重生计时、UI 提示等
    UpdateScore(Killer);
    StartRespawnTimer(DeadActor);
    ShowKillFeed(Killer, DeadActor);
}
```

### 5. 分层架构 (Layered Architecture)

Lyra 将代码分为清晰的层次，每层职责单一：

```
┌─────────────────────────────────────────┐
│         Game-Specific Logic             │  游戏特定逻辑（你的项目代码）
│  (Custom Game Modes, Characters, etc.)  │
├─────────────────────────────────────────┤
│         Lyra Framework                  │  Lyra 框架层（可复用的架构）
│  (Experience, Pawn Extensions, etc.)    │
├─────────────────────────────────────────┤
│         Core Systems                    │  核心系统（GAS、Input、UI）
│  (GAS, Enhanced Input, Common UI)       │
├─────────────────────────────────────────┤
│         Unreal Engine Core              │  UE 引擎核心
│  (Actor, Component, Replication, etc.)  │
└─────────────────────────────────────────┘
```

**代码组织原则**：
- **向下依赖**：上层可以依赖下层，反之不可
- **接口隔离**：跨层通信通过接口或事件，不直接依赖具体类
- **模块化**：每个功能尽量独立，减少模块间耦合

---

## 项目结构深度解析

### 源码目录结构

```
LyraStarterGame/
├── Source/
│   ├── LyraGame/                      # 核心游戏代码
│   │   ├── AbilitySystem/             # GAS 相关
│   │   │   ├── LyraAbilitySystemComponent.h/cpp
│   │   │   ├── Abilities/             # 游戏技能
│   │   │   ├── Attributes/            # 属性集
│   │   │   └── Executions/            # 伤害计算
│   │   ├── Character/                 # 角色系统
│   │   │   ├── LyraCharacter.h/cpp
│   │   │   ├── LyraHealthComponent.h/cpp
│   │   │   └── LyraPawnExtensionComponent.h/cpp
│   │   ├── Equipment/                 # 装备系统
│   │   │   ├── LyraEquipmentManagerComponent.h/cpp
│   │   │   └── LyraEquipmentDefinition.h/cpp
│   │   ├── Input/                     # 输入系统
│   │   │   ├── LyraInputComponent.h/cpp
│   │   │   └── LyraInputConfig.h/cpp
│   │   ├── Inventory/                 # 背包系统
│   │   ├── GameModes/                 # 游戏模式
│   │   │   ├── LyraExperienceDefinition.h/cpp
│   │   │   ├── LyraExperienceManagerComponent.h/cpp
│   │   │   └── LyraGameState.h/cpp
│   │   ├── Player/                    # 玩家控制器
│   │   ├── Teams/                     # 队伍系统
│   │   ├── UI/                        # UI 框架
│   │   ├── Weapons/                   # 武器系统
│   │   └── System/                    # 核心系统
│   └── LyraEditor/                    # 编辑器扩展
│
├── Content/
│   ├── __ExternalActors__/            # 关卡 Actor 数据
│   ├── __ExternalObjects__/           # 外部对象
│   ├── Characters/                    # 角色资源
│   ├── UI/                            # UI 资源
│   ├── Weapons/                       # 武器资源
│   └── System/                        # 系统配置
│
└── Plugins/
    ├── GameFeatures/                  # Game Features 插件
    │   ├── ShooterCore/               # 射击游戏核心
    │   ├── ShooterMaps/               # 射击地图
    │   ├── TopDownArena/              # 俯视角竞技场
    │   └── ShooterTests/              # 测试内容
    ├── CommonGame/                    # 通用游戏插件
    ├── CommonUser/                    # 用户系统插件
    └── ModularGameplayActors/         # 模块化 Actor 插件
```

### 关键模块说明

#### 1. **AbilitySystem/** - GAS 实现

这是 Lyra 最核心的模块之一，包含：

```cpp
// LyraAbilitySystemComponent.h - 扩展的 ASC
UCLASS()
class ULyraAbilitySystemComponent : public UAbilitySystemComponent
{
    GENERATED_BODY()

public:
    // 初始化默认技能集
    void InitAbilityActorInfo(AActor* InOwnerActor, AActor* InAvatarActor) override;

    // 按输入 Tag 触发技能
    void AbilityInputTagPressed(const FGameplayTag& InputTag);
    void AbilityInputTagReleased(const FGameplayTag& InputTag);

    // 取消所有技能（用于死亡、眩晕等状态）
    void CancelAbilitiesWithTags(const FGameplayTagContainer& Tags);

    // 获取指定 Tag 的技能 Spec
    FGameplayAbilitySpecHandle FindAbilitySpecHandleForClass(
        TSubclassOf<UGameplayAbility> AbilityClass) const;
};
```

**重要子目录**：

- `Attributes/`: 定义角色属性（血量、护甲、移速等）
  - `LyraHealthSet.h`: 血量相关属性
  - `LyraCombatSet.h`: 战斗属性（攻击力、暴击率等）
  
- `Abilities/`: 具体的游戏技能
  - `LyraGameplayAbility_Jump.h`: 跳跃技能
  - `LyraGameplayAbility_Death.h`: 死亡处理技能
  
- `Executions/`: 伤害计算和效果执行
  - `LyraDamageExecution.h`: 伤害公式计算

#### 2. **Character/** - 角色系统

```cpp
// LyraCharacter.h - 核心角色类
UCLASS()
class ALyraCharacter : public AModularCharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    ALyraCharacter(const FObjectInitializer& ObjectInitializer);

    // 组件引用
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Character")
    TObjectPtr<ULyraPawnExtensionComponent> PawnExtComponent;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Character")
    TObjectPtr<ULyraHealthComponent> HealthComponent;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Character")
    TObjectPtr<ULyraCameraComponent> CameraComponent;

    // IAbilitySystemInterface 实现
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

protected:
    // 初始化流程
    virtual void PreInitializeComponents() override;
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    // 输入绑定
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
};
```

**核心组件**：

- **ULyraPawnExtensionComponent**: 角色扩展组件，管理初始化流程
- **ULyraHealthComponent**: 血量和死亡处理
- **ULyraCameraComponent**: 相机管理（支持多种相机模式）

#### 3. **GameModes/** - Experience 系统

```cpp
// LyraExperienceManagerComponent.h - Experience 管理器
UCLASS()
class ULyraExperienceManagerComponent : public UGameStateComponent
{
    GENERATED_BODY()

public:
    // 开始加载 Experience
    void StartExperienceLoad(const ULyraExperienceDefinition* Experience);

    // Experience 是否已加载完成
    bool IsExperienceLoaded() const;

    // 加载完成回调
    FOnLyraExperienceLoaded OnExperienceLoaded;

private:
    // 当前 Experience
    UPROPERTY()
    TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;

    // 加载状态
    ELyraExperienceLoadState LoadState;

    // Game Features 加载追踪
    TArray<FString> GameFeaturePluginURLs;
};
```

#### 4. **UI/** - Common UI 实现

```cpp
// LyraActivatableWidget.h - 可激活的 UI 基类
UCLASS()
class ULyraActivatableWidget : public UCommonActivatableWidget
{
    GENERATED_BODY()

public:
    // 输入配置（用于控制器导航）
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    FDataTableRowHandle BackInputAction;

protected:
    virtual void NativeOnActivated() override;
    virtual void NativeOnDeactivated() override;

    // 处理返回输入
    UFUNCTION()
    void HandleBackAction();
};
```

#### 5. **Equipment/** & **Weapons/** - 装备和武器

```cpp
// LyraEquipmentDefinition.h - 装备定义
UCLASS()
class ULyraEquipmentDefinition : public UObject
{
    GENERATED_BODY()

public:
    // 装备实例类
    UPROPERTY(EditDefaultsOnly, Category = "Equipment")
    TSubclassOf<ULyraEquipmentInstance> InstanceType;

    // 生成的 Actor（如武器 Mesh）
    UPROPERTY(EditDefaultsOnly, Category = "Equipment")
    TArray<FLyraEquipmentActorToSpawn> ActorsToSpawn;

    // 装备时授予的技能
    UPROPERTY(EditDefaultsOnly, Category = "Equipment")
    TArray<FLyraAbilitySet_GrantedHandles> AbilitySetsToGrant;
};

// LyraWeaponInstance.h - 武器实例
UCLASS()
class ULyraWeaponInstance : public ULyraEquipmentInstance
{
    GENERATED_BODY()

public:
    // 当前弹药
    UPROPERTY(BlueprintReadWrite, Category = "Weapon")
    int32 CurrentAmmo;

    // 武器数据
    UPROPERTY()
    TObjectPtr<const ULyraWeaponData> WeaponData;

    // 开火逻辑
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void Fire();
};
```

### 插件系统结构

#### Game Features 插件示例

以 `ShooterCore` 插件为例：

```
ShooterCore/
├── ShooterCore.uplugin             # 插件描述文件
├── Content/
│   ├── Game/
│   │   ├── B_ShooterGame.uasset    # 游戏模式 Blueprint
│   │   └── B_Hero_ShooterMannequin.uasset  # 英雄角色 BP
│   ├── Input/
│   │   └── IMC_Default.uasset      # 输入映射上下文
│   ├── Weapons/
│   │   ├── Pistol/
│   │   ├── Rifle/
│   │   └── Shotgun/
│   └── Abilities/
│       ├── GA_Weapon_Fire.uasset   # 射击技能
│       └── GA_Hero_Jump.uasset     # 跳跃技能
└── Source/
    └── ShooterCoreRuntime/
        ├── ShooterCoreRuntimeModule.cpp
        └── Private/
            └── ... (C++ 实现)
```

**插件元数据** (`ShooterCore.uplugin`):

```json
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "1.0",
    "FriendlyName": "Shooter Core",
    "Description": "射击游戏核心功能",
    "Category": "Game Features",
    "CreatedBy": "Epic Games",
    "CreatedByURL": "https://www.unrealengine.com",
    "BuiltInInitialFeatureState": "Active",
    "Modules": [
        {
            "Name": "ShooterCoreRuntime",
            "Type": "Runtime",
            "LoadingPhase": "Default"
        }
    ],
    "Plugins": [
        {
            "Name": "ModularGameplayActors",
            "Enabled": true
        },
        {
            "Name": "GameplayAbilities",
            "Enabled": true
        },
        {
            "Name": "GameFeatures",
            "Enabled": true
        }
    ]
}
```

---

## 环境搭建详细指南

### 系统要求

#### 硬件最低配置

| 组件 | 最低要求 | 推荐配置 |
|------|---------|---------|
| **操作系统** | Windows 10 64-bit (1909+) | Windows 11 64-bit |
| **CPU** | Quad-core Intel/AMD, 2.5 GHz+ | 8-core Intel/AMD, 3.5 GHz+ |
| **内存** | 16 GB RAM | 32 GB RAM |
| **显卡** | DirectX 12 兼容, 4GB VRAM | NVIDIA RTX 3060 / AMD RX 6700 XT+ |
| **存储** | 200 GB SSD 可用空间 | 500 GB NVMe SSD |

#### 软件依赖

- **Visual Studio 2022** (Community 版免费)
  - 工作负载：使用 C++ 的游戏开发
  - 组件：Windows 10/11 SDK
  
- **.NET Framework 4.6.2+**

- **DirectX Runtime** (通常已预装)

### 步骤 1: 下载和安装 Unreal Engine 5.5

#### 1.1 安装 Epic Games Launcher

1. 访问 [https://www.unrealengine.com/download](https://www.unrealengine.com/download)
2. 下载 Epic Games Launcher 安装程序
3. 运行安装程序，按提示完成安装
4. 登录你的 Epic Games 账户（没有则注册一个）

#### 1.2 安装 Unreal Engine 5.5

1. 打开 Epic Games Launcher
2. 点击左侧的 **"Unreal Engine"** 标签
3. 点击顶部的 **"Library"** 标签
4. 点击 **"+"** 按钮添加引擎版本
5. 选择 **"5.5.x"** 版本（选择最新的子版本）
6. 点击 **"Install"** 按钮

**安装位置建议**：
- 默认路径：`C:\Program Files\Epic Games\UE_5.5`
- 如果 C 盘空间不足，可选择其他磁盘（如 `D:\EpicGames\UE_5.5`）

**安装时间**：根据网速，约 30 分钟到 2 小时。

#### 1.3 验证安装

安装完成后，在 Launcher 的 Library 中会显示已安装的引擎：

```
✅ Unreal Engine 5.5.1
   Size: ~45 GB
   Platform: Win64
```

### 步骤 2: 从 Epic Games Launcher 安装 Lyra 项目

#### 2.1 查找 Lyra 项目模板

1. 在 Epic Games Launcher 中，点击 **"Unreal Engine"** → **"Library"**
2. 向下滚动到 **"Vault"** 部分
3. 点击 **"Learn"** 标签
4. 搜索 **"Lyra Starter Game"**

#### 2.2 下载 Lyra 项目

1. 点击 Lyra Starter Game 的图标
2. 点击 **"Create Project"** 按钮
3. 选择引擎版本：**5.5.x**
4. 选择项目位置（建议选择 SSD）
5. 项目名称：默认 `LyraStarterGame` 或自定义
6. 点击 **"Create"** 按钮

**项目大小**：约 20-30 GB（包含所有资源和源码）

#### 2.3 首次编译

Lyra 是一个 **C++ 项目**，首次打开时会自动编译：

1. 下载完成后，点击 **"Launch"** 按钮
2. 引擎会自动检测需要编译
3. 弹出提示：**"Would you like to rebuild now?"**
4. 点击 **"Yes"**
5. Visual Studio 会自动打开并开始编译

**编译时间**：首次编译约 10-30 分钟（取决于 CPU 性能）。

#### 常见编译问题

**问题 1**: `"Unable to start program ... (error code 0xc0000135)"`

**原因**: 缺少 .NET Framework 运行时

**解决**:
```powershell
# 以管理员身份运行 PowerShell
dism /online /enable-feature /featurename:NetFx4 /all
```

**问题 2**: `"C1083: Cannot open include file: 'windows.h'"`

**原因**: Windows SDK 未安装

**解决**:
1. 打开 Visual Studio Installer
2. 点击 **"Modify"**
3. 勾选 **"Windows 10 SDK (10.0.19041.0 或更高)"**
4. 点击 **"Modify"** 安装

**问题 3**: `"Error: Out of memory"` 编译时内存不足

**解决**:
```ini
; 编辑 LyraStarterGame.Target.cs
bLegacyParallelExecutor = true;  // 使用旧的并行编译器（更省内存）
```

或者在 Visual Studio 中限制并行编译任务数：
- 工具 → 选项 → 项目和解决方案 → 生成和运行
- 设置 **"最大并行项目生成数"** 为 2 或 4

### 步骤 3: 配置 Visual Studio

#### 3.1 安装推荐的 VS 扩展

打开 Visual Studio，安装以下扩展（可选但强烈推荐）：

1. **Visual Assist** (付费，但有试用期)
   - 增强的代码补全和重构功能
   - 下载：[https://www.wholetomato.com/](https://www.wholetomato.com/)

2. **UnrealVS** (免费，Epic 官方)
   - UE 项目专用工具栏
   - 快速启动引擎和编辑器
   - 位置：`UE_5.5\Engine\Extras\UnrealVS\UnrealVS.vsix`

3. **Trailing Whitespace Visualizer** (免费)
   - 显示多余的空格（UE 编码规范要求）

#### 3.2 配置编译器选项

**优化编译速度**：

编辑 `%APPDATA%\Unreal Engine\UnrealBuildTool\BuildConfiguration.xml`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Configuration xmlns="https://www.unrealengine.com/BuildConfiguration">
    <BuildConfiguration>
        <!-- 使用增量编译 -->
        <bUseIncrementalLinking>true</bUseIncrementalLinking>
        
        <!-- 使用预编译头 -->
        <bUsePCHFiles>true</bUsePCHFiles>
        <bUseSharedPCHs>true</bUseSharedPCHs>
        
        <!-- 并行编译任务数（根据 CPU 核心数调整） -->
        <MaxParallelActions>8</MaxParallelActions>
        
        <!-- 使用 FastPDB（加速调试信息生成） -->
        <bUseFastPDB>true</bUseFastPDB>
    </BuildConfiguration>
</Configuration>
```

#### 3.3 设置断点和调试

1. 在 Visual Studio 中打开 `LyraStarterGame.sln`
2. 设置启动项目：右键 **"LyraStarterGame"** → **"设为启动项目"**
3. 配置启动参数（可选）：
   - 右键项目 → **"属性"**
   - 调试 → 命令参数：
     ```
     -game -log  // 以游戏模式启动，显示日志窗口
     ```

### 步骤 4: 配置编辑器设置

#### 4.1 编辑器首选项

首次打开 Lyra 编辑器后，建议配置以下选项：

**编辑 → 编辑器偏好设置**：

1. **通用 → 加载和保存**：
   - ✅ 启用 **"自动保存"**
   - 间隔设置为 **5 分钟**

2. **通用 → 性能**：
   - ✅ 启用 **"使用更少的 CPU 资源（当编辑器在后台时）"**

3. **内容浏览器**：
   - ✅ 启用 **"显示插件内容"**
   - ✅ 启用 **"显示引擎内容"**

4. **源代码**：
   - 首选访问器：**Visual Studio 2022**
   - ✅ 启用 **"编辑文件时在 VS 中刷新项目"**

#### 4.2 项目设置

**编辑 → 项目设置**：

1. **地图和模式 → 默认地图**：
   - 编辑器启动地图：`/ShooterCore/Maps/L_ShooterGym`
   - 游戏默认地图：`/ShooterCore/Maps/L_Expanse`

2. **输入**：
   - 默认输入组件类：`LyraInputComponent`
   - ✅ 启用 **Enhanced Input**

3. **GAS (Gameplay Abilities)**：
   - ✅ 启用 **"Gameplay Debugger"**
   - ✅ 启用 **"Gameplay Tags Editor"**

#### 4.3 开发者模式设置

启用更多调试工具：

**窗口 → 开发者工具**：
- ✅ **Output Log** (输出日志)
- ✅ **Message Log** (消息日志)
- ✅ **Class Viewer** (类查看器)
- ✅ **Gameplay Tags** (Gameplay 标签编辑器)

---

## 第一次运行 Lyra

### 启动编辑器

有两种方式启动：

#### 方式 1: 从 Epic Games Launcher 启动
1. 打开 Launcher → **Library** → **My Projects**
2. 找到 **LyraStarterGame**
3. 点击项目图标下的 **"Launch"** 按钮

#### 方式 2: 从 Visual Studio 启动（推荐开发时使用）
1. 打开 `LyraStarterGame.sln`
2. 按 **F5** 或点击 **"本地 Windows 调试器"** 按钮

**首次启动**会执行以下操作：
- 编译着色器（Shader）：约 5-15 分钟
- 加载资源：约 2-5 分钟
- 初始化插件

总计首次启动需要 **10-20 分钟**，请耐心等待。

### 体验预设游戏模式

Lyra 内置了多个示例体验：

#### 1. **ShooterCore Experience** (第三人称射击)

- **地图**: `L_Expanse`（开阔竞技场）、`L_Convolution`（室内地图）
- **玩法**: 团队死斗（TDM）
- **特色**: 武器系统、装备切换、团队机制

**操作方式**：
1. 在编辑器中打开 `Content/ShooterCore/Maps/L_Expanse`
2. 点击 **"Play"** 按钮（或按 **Alt + P**）
3. 使用 WASD 移动，鼠标控制视角
4. 左键射击，右键瞄准
5. 数字键 1-3 切换武器

#### 2. **TopDownArena Experience** (俯视角竞技场)

- **地图**: `L_TopDownArena`
- **玩法**: 自上而下视角的射击游戏
- **特色**: 不同的相机模式、简化的输入

**操作方式**：
1. 打开 `Content/TopDownArena/Maps/L_TopDownArena`
2. 点击 **Play**
3. 使用 WASD 移动
4. 鼠标点击地面移动（类似 MOBA 游戏）
5. 技能键施放能力

#### 3. **ShooterTests Experience** (测试关卡)

- **地图**: `L_ShooterGym`（射击训练场）
- **用途**: 用于测试武器、AI、Gameplay 系统
- **特色**: 简化的环境，专注功能验证

### 编辑器内测试

#### 单人测试

1. **PIE (Play In Editor)**:
   - 按 **Alt + P**：在编辑器窗口内运行
   - 按 **Alt + S**：在独立窗口运行（推荐，更接近真实游戏）
   - 按 **Alt + M**：以多人模式运行（本地模拟网络）

2. **快速重启**:
   - 按 **Esc** 停止运行
   - 按 **Alt + P** 再次启动

#### 多人测试（本地）

测试网络同步和多人交互：

1. 点击 **Play** 按钮旁的下拉箭头
2. 设置 **"Number of Players"**: `2` 或更多
3. 勾选 **"Run Dedicated Server"**（可选，模拟独立服务器）
4. 点击 **"Play"**

编辑器会启动多个游戏窗口，每个代表一个客户端。

### 常用控制台命令

在游戏运行时，按 **`~`** 键（波浪号）打开控制台，输入以下命令：

| 命令 | 功能 |
|------|------|
| `stat fps` | 显示帧率 |
| `stat unit` | 显示性能统计 |
| `showdebug abilitysystem` | 显示 GAS 调试信息 |
| `AbilitySystem.Debug.NextCategory` | 切换 GAS 调试类别 |
| `Lyra.DumpExperience` | 输出当前 Experience 的详细信息 |
| `DamageTarget X` | 对准星目标造成 X 点伤害 |
| `God` | 无敌模式 |
| `Fly` | 飞行模式（再次输入 `Walk` 恢复） |
| `Teleport` | 传送到准星位置 |
| `ToggleDebugCamera` | 切换自由相机 |

---

## 开发环境配置

### 推荐的工作流

#### 热重载（Hot Reload）

UE5 支持在不重启编辑器的情况下重新编译代码：

1. **Live Coding** (推荐):
   - 工具栏点击 **"Live Coding"** 按钮
   - 或使用快捷键 **Ctrl + Alt + F11**
   - 修改代码后，点击 **"Compile"** 即可热更新

2. **传统热重载**:
   - 在 Visual Studio 中修改并保存代码
   - 回到编辑器，编辑器会自动检测到文件变化
   - 提示 **"Recompile?"**，点击 **"Yes"**

**注意**：
- 热重载不支持添加新类或修改反射宏（`UPROPERTY`、`UFUNCTION` 等）
- 这些情况需要完全重启编辑器

#### 推荐的代码编辑器配置

**Visual Studio Code** (轻量级，适合快速编辑)：

1. 安装扩展：
   - **C/C++** (Microsoft)
   - **Unreal Engine 4 Snippets**
   - **Bracket Pair Colorizer 2**

2. 配置 IntelliSense：
   - 在 Lyra 项目根目录，右键 `.uproject` 文件
   - 选择 **"Generate Visual Studio Code Project Files"**
   - 在 VS Code 中打开项目文件夹

**Visual Studio 2022** (完整 IDE，适合调试)：

1. 使用 **Solution Filters** 减少加载时间：
   - 右键 `LyraStarterGame.sln`
   - 选择 **"Create Solution Filter"**
   - 只保留 **LyraGame** 和 **ShooterCore** 项目

2. 使用 **IntelliCode** 提升补全质量：
   - 扩展 → 管理扩展
   - 搜索并安装 **"IntelliCode"**

### 版本控制配置

#### 推荐使用 Git LFS

Lyra 项目包含大量二进制资源，必须使用 **Git Large File Storage** (LFS)：

1. **安装 Git LFS**:
   ```bash
   git lfs install
   ```

2. **配置 `.gitattributes`** (项目根目录):
   ```gitattributes
   # 二进制资源使用 LFS
   *.uasset filter=lfs diff=lfs merge=lfs -text
   *.umap filter=lfs diff=lfs merge=lfs -text
   *.ubulk filter=lfs diff=lfs merge=lfs -text
   *.uexp filter=lfs diff=lfs merge=lfs -text
   *.upk filter=lfs diff=lfs merge=lfs -text
   *.uexp filter=lfs diff=lfs merge=lfs -text

   # 大型音视频文件
   *.wav filter=lfs diff=lfs merge=lfs -text
   *.mp4 filter=lfs diff=lfs merge=lfs -text
   *.mov filter=lfs diff=lfs merge=lfs -text

   # 编译结果（不提交）
   *.obj
   *.pdb
   *.dll
   ```

3. **配置 `.gitignore`**:
   ```gitignore
   # UE5 临时文件
   Intermediate/
   Saved/
   DerivedDataCache/
   
   # Visual Studio
   .vs/
   *.suo
   *.user
   *.sln.docstates
   
   # Rider
   .idea/
   *.sln.iml
   
   # Build 结果
   Binaries/
   Build/
   
   # 插件的 Intermediate
   Plugins/*/Intermediate/
   Plugins/*/Binaries/
   ```

#### 推荐的 Git 工作流

**分支策略**：

```
main (受保护，只接受 PR)
  ├─ develop (日常开发)
  ├─ feature/shooter-maps (功能分支)
  ├─ feature/inventory-system
  └─ hotfix/crash-fix
```

**提交规范**：

```bash
# 好的提交消息
git commit -m "feat(AbilitySystem): 添加冲刺技能"
git commit -m "fix(Equipment): 修复武器切换时的崩溃"
git commit -m "docs(README): 更新环境搭建步骤"

# 不好的提交消息
git commit -m "update"
git commit -m "fix bug"
```

### 调试技巧

#### 1. Visual Studio 调试

**设置条件断点**：

```cpp
// 在 LyraHealthComponent.cpp 中
void ULyraHealthComponent::HandleHealthChanged(const FOnAttributeChangeData& ChangeData)
{
    float NewHealth = ChangeData.NewValue;
    float OldHealth = ChangeData.OldValue;
    
    // 右键此行 → "断点" → "条件..."
    // 条件: NewHealth <= 0.0f
    if (NewHealth <= OldHealth)
    {
        OnHealthDecreased(NewHealth, OldHealth);
    }
}
```

**监视窗口技巧**：

- 添加 `this` 查看当前对象的所有成员
- 添加 `GetOwner()` 查看所属 Actor
- 添加 `GetWorld()->GetTimeSeconds()` 查看游戏时间

#### 2. 编辑器内调试

**Gameplay Debugger** (按 **`'`** 键，单引号)：

```cpp
// 在代码中添加调试类别
#include "GameplayDebuggerCategory.h"

class FGameplayDebuggerCategory_Lyra : public FGameplayDebuggerCategory
{
public:
    virtual void CollectData(APlayerController* OwnerPC, AActor* DebugActor) override
    {
        ALyraCharacter* Character = Cast<ALyraCharacter>(DebugActor);
        if (Character)
        {
            // 显示当前血量
            AddTextLine(FString::Printf(TEXT("Health: %.1f / %.1f"),
                Character->GetHealth(), Character->GetMaxHealth()));
            
            // 显示当前武器
            ULyraEquipmentManagerComponent* EquipMgr = Character->GetEquipmentManager();
            AddTextLine(FString::Printf(TEXT("Weapon: %s"),
                *EquipMgr->GetActiveWeapon()->GetName()));
        }
    }
};
```

#### 3. 日志系统

**定义日志类别** (在 `LyraLogChannels.h`):

```cpp
// 声明日志类别
LYRA_API DECLARE_LOG_CATEGORY_EXTERN(LogLyraExperience, Log, All);
LYRA_API DECLARE_LOG_CATEGORY_EXTERN(LogLyraAbilitySystem, Log, All);
LYRA_API DECLARE_LOG_CATEGORY_EXTERN(LogLyraTeams, Log, All);

// 在 .cpp 文件中定义
DEFINE_LOG_CATEGORY(LogLyraExperience);
```

**使用日志**：

```cpp
// 不同级别的日志
UE_LOG(LogLyraExperience, Log, TEXT("Experience 开始加载: %s"), *Experience->GetName());
UE_LOG(LogLyraExperience, Warning, TEXT("未找到 Pawn Data!"));
UE_LOG(LogLyraExperience, Error, TEXT("加载 Game Feature 失败: %s"), *ErrorMsg);

// 条件日志（只在满足条件时输出）
UE_CLOG(bIsServer, LogLyraExperience, Log, TEXT("[Server] Experience 已加载"));

// 带变量的日志
float Health = 85.5f;
UE_LOG(LogTemp, Display, TEXT("玩家血量: %.1f"), Health);
```

**运行时查看日志**：
- 编辑器：**窗口 → 开发者工具 → Output Log**
- 游戏内：按 **`~`** 打开控制台，输入 `log LogLyraExperience Verbose`

---

## 常见问题与解决方案

### 编译错误

#### Q1: `"LNK2001: unresolved external symbol"` 链接错误

**原因**: 缺少模块依赖或库文件。

**解决**:

1. 检查 `Build.cs` 文件的 `PublicDependencyModuleNames`:
   ```csharp
   PublicDependencyModuleNames.AddRange(new string[] {
       "Core",
       "CoreUObject",
       "Engine",
       "GameplayAbilities",    // 使用 GAS 时必须添加
       "GameplayTags",
       "GameplayTasks",
       "ModularGameplayActors",
       "CommonUI",
       "CommonInput"
   });
   ```

2. 如果使用插件的类，添加插件依赖:
   ```csharp
   PrivateDependencyModuleNames.AddRange(new string[] {
       "GameFeatures",
       "ModularGameplay"
   });
   ```

3. 清理并重新生成项目文件:
   ```bash
   # 删除以下目录
   rm -rf Binaries/ Intermediate/ .vs/ *.sln
   
   # 右键 .uproject → "Generate Visual Studio project files"
   # 重新打开 .sln 并编译
   ```

#### Q2: `"Cannot open include file: 'LyraXXX.h'"`

**原因**: 头文件路径错误或模块未正确配置。

**解决**:

1. 检查 `#include` 路径：
   ```cpp
   // ❌ 错误：绝对路径
   #include "LyraGame/Character/LyraCharacter.h"
   
   // ✅ 正确：相对于模块根目录
   #include "Character/LyraCharacter.h"
   ```

2. 检查 `Build.cs` 的 `PublicIncludePaths`:
   ```csharp
   PublicIncludePaths.AddRange(new string[] {
       "LyraGame"  // 允许直接包含 "Character/LyraCharacter.h"
   });
   ```

#### Q3: `"Trying to serialize an unknown property 'XXX'"`

**原因**: 反射宏（`UPROPERTY`、`UFUNCTION` 等）变化后未重新生成代码。

**解决**:

1. 关闭编辑器和 Visual Studio
2. 删除 `Intermediate/` 和 `Binaries/` 文件夹
3. 右键 `.uproject` → **"Generate Visual Studio project files"**
4. 重新打开 `.sln` 并编译

### 运行时错误

#### Q4: 编辑器启动时崩溃

**检查步骤**：

1. 查看崩溃报告（`Saved/Crashes/` 目录）
2. 检查 `Saved/Logs/` 中的日志文件

**常见原因**：

- **插件冲突**: 禁用第三方插件后重试
- **资源损坏**: 删除 `DerivedDataCache/` 重新生成
- **驱动问题**: 更新显卡驱动

#### Q5: PIE 时提示 "Game Feature could not be loaded"

**原因**: Game Feature 插件未激活或路径错误。

**解决**:

1. 打开 **窗口 → Game Features**
2. 检查插件状态，确保需要的插件为 **"Active"**
3. 如果插件未出现在列表中，检查插件目录：
   ```
   Plugins/GameFeatures/<PluginName>/<PluginName>.uplugin
   ```

4. 验证 Experience Definition 中的插件路径：
   ```cpp
   // 正确的格式
   GameFeaturesToEnable.Add(TEXT("/Game/ShooterCore"));
   
   // ❌ 错误：不要包含 "Content" 或文件扩展名
   GameFeaturesToEnable.Add(TEXT("/Game/Content/ShooterCore.uasset"));
   ```

#### Q6: 网络同步问题（多人测试时）

**症状**: 客户端看到的和服务器不一致。

**排查步骤**:

1. 确认属性已标记为 Replicated：
   ```cpp
   UPROPERTY(ReplicatedUsing = OnRep_Health)
   float Health;

   void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override
   {
       Super::GetLifetimeReplicatedProps(OutLifetimeProps);
       DOREPLIFETIME(ULyraHealthComponent, Health);
   }
   ```

2. 检查 RPC 调用：
   ```cpp
   // 服务器 RPC
   UFUNCTION(Server, Reliable, WithValidation)
   void ServerPickupWeapon(ALyraWeaponPickup* Weapon);

   bool ServerPickupWeapon_Validate(ALyraWeaponPickup* Weapon)
   {
       return Weapon != nullptr;  // 验证参数有效性
   }

   void ServerPickupWeapon_Implementation(ALyraWeaponPickup* Weapon)
   {
       // 服务器执行逻辑
   }
   ```

3. 使用网络模拟工具：
   - 编辑 → 项目设置 → 引擎 → Network
   - 设置 **"Packet Lag"** 和 **"Packet Loss"** 模拟网络延迟

### 性能问题

#### Q7: 帧率过低

**优化步骤**：

1. **禁用编辑器功能**:
   - 编辑 → 编辑器偏好设置 → 视口
   - 取消勾选 **"实时"** 和 **"显示帧率和内存"**

2. **降低编辑器画质**:
   - 视口左上角 → 设置 → 引擎可扩展性设置
   - 选择 **"低"** 或 **"中"**

3. **检查性能瓶颈**:
   ```
   # 在控制台输入
   stat fps          # 查看帧率
   stat unit         # 查看 CPU/GPU 时间
   stat scenerendering  # 查看渲染统计
   ```

4. **优化代码**:
   ```cpp
   // ❌ 错误：每帧查找组件
   void UMyComponent::TickComponent(float DeltaTime, ...)
   {
       ULyraHealthComponent* Health = GetOwner()->FindComponentByClass<ULyraHealthComponent>();
       // ...
   }

   // ✅ 正确：缓存组件引用
   UPROPERTY()
   TObjectPtr<ULyraHealthComponent> CachedHealthComponent;

   void UMyComponent::BeginPlay()
   {
       Super::BeginPlay();
       CachedHealthComponent = GetOwner()->FindComponentByClass<ULyraHealthComponent>();
   }

   void UMyComponent::TickComponent(float DeltaTime, ...)
   {
       if (CachedHealthComponent)
       {
           // 直接使用缓存
       }
   }
   ```

#### Q8: 内存占用过高

**排查步骤**：

1. **查看内存分配**:
   ```
   # 控制台命令
   stat memory
   memreport -full  # 生成详细内存报告（位于 Saved/Profiling/）
   ```

2. **检查资源引用**:
   - 编辑 → 项目设置 → 引擎 → Garbage Collection
   - 启用 **"Verify Garbage Collection Assumptions"**
   - 查找内存泄漏

3. **优化资源加载**:
   ```cpp
   // 使用软引用（不会立即加载）
   UPROPERTY(EditDefaultsOnly, Category = "Assets")
   TSoftObjectPtr<UStaticMesh> WeaponMesh;

   // 需要时再加载
   void LoadWeaponMesh()
   {
       if (!WeaponMesh.IsValid())
       {
           UStaticMesh* LoadedMesh = WeaponMesh.LoadSynchronous();
           // 使用 LoadedMesh
       }
   }
   ```

---

## 最佳实践与建议

### 代码风格

遵循 Epic 的 **[Coding Standard](https://docs.unrealengine.com/5.5/en-US/epic-cplusplus-coding-standard-for-unreal-engine/)**：

#### 命名规范

```cpp
// ✅ 正确的命名
UCLASS()
class ULyraHealthComponent : public UGameFrameworkComponent  // 类名：UPrefix + PascalCase
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintReadWrite, Category = "Health")
    float MaxHealth;  // 成员变量：PascalCase

    UFUNCTION(BlueprintCallable, Category = "Health")
    void TakeDamage(float DamageAmount);  // 函数：PascalCase

private:
    bool bIsInvulnerable;  // Boolean 变量：b + PascalCase
    int32 RegenerationRate;  // int32 而非 int
    TObjectPtr<AActor> OwnerActor;  // 智能指针：TObjectPtr<>

    void HandleHealthChanged();  // 私有函数
};

// ❌ 错误的命名
class healthComponent {};  // 缺少前缀
float max_health;  // 使用下划线
bool isInvulnerable;  // Boolean 缺少 b 前缀
int counter;  // 应使用 int32
```

#### 注释规范

```cpp
/**
 * 血量组件，管理角色的生命值和死亡逻辑。
 * 
 * 使用 GAS (Gameplay Ability System) 处理伤害和治疗。
 * 支持护盾、伤害减免等高级功能。
 */
UCLASS(BlueprintType, meta = (BlueprintSpawnableComponent))
class LYRAGAME_API ULyraHealthComponent : public UGameFrameworkComponent
{
    GENERATED_BODY()

public:
    /**
     * 对目标造成伤害。
     * 
     * @param DamageAmount 伤害数值（正数）
     * @param Instigator 造成伤害的 Actor（可为空）
     * @param DamageCauser 伤害源头（如子弹、炸弹，可为空）
     * @return 实际造成的伤害（考虑护甲减免后）
     */
    UFUNCTION(BlueprintCallable, Category = "Health")
    float TakeDamage(float DamageAmount, AActor* Instigator, AActor* DamageCauser);

private:
    // 当前血量值（同步到客户端）
    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float CurrentHealth;

    // 死亡动画播放完成后执行的回调
    void OnDeathAnimationFinished();
};
```

### 架构建议

#### 1. 组件化设计原则

**单一职责**：每个组件只负责一个领域。

```cpp
// ✅ 好的设计：职责清晰
ULyraHealthComponent       // 只管血量和死亡
ULyraEquipmentManagerComponent  // 只管装备系统
ULyraAbilitySystemComponent     // 只管技能系统

// ❌ 不好的设计：职责混乱
ULyraCharacterComponent    // 什么都管，难以维护
```

**组件通信**：优先使用事件和接口，避免直接引用。

```cpp
// ✅ 好的通信方式：通过事件
UCLASS()
class ULyraHealthComponent : public UGameFrameworkComponent
{
public:
    DECLARE_MULTICAST_DELEGATE_OneParam(FOnHealthChanged, float /*NewHealth*/);
    FOnHealthChanged OnHealthChanged;

    void SetHealth(float NewHealth)
    {
        CurrentHealth = NewHealth;
        OnHealthChanged.Broadcast(NewHealth);  // 通知所有监听者
    }
};

// 其他组件监听
void UMyUIComponent::BeginPlay()
{
    Super::BeginPlay();

    ULyraHealthComponent* Health = GetOwner()->FindComponentByClass<ULyraHealthComponent>();
    if (Health)
    {
        Health->OnHealthChanged.AddUObject(this, &UMyUIComponent::OnHealthUpdated);
    }
}

void UMyUIComponent::OnHealthUpdated(float NewHealth)
{
    UpdateHealthBar(NewHealth);
}
```

#### 2. 数据资产使用建议

**创建 Data Asset 的时机**：

- ✅ 需要策划调整的数值（武器伤害、角色速度等）
- ✅ 需要复用的配置（输入映射、UI 布局等）
- ✅ 需要版本控制的内容（游戏模式、关卡列表等）
- ❌ 纯代码逻辑（算法、流程控制等）

**Data Asset 继承结构**：

```cpp
// 基类：通用属性
UCLASS()
class ULyraItemDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Display")
    FText DisplayName;

    UPROPERTY(EditDefaultsOnly, Category = "Display")
    TObjectPtr<UTexture2D> Icon;
};

// 子类：武器特有属性
UCLASS()
class ULyraWeaponDefinition : public ULyraItemDefinition
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Stats")
    float Damage;

    UPROPERTY(EditDefaultsOnly, Category = "Stats")
    float FireRate;
};

// 子类：消耗品特有属性
UCLASS()
class ULyraConsumableDefinition : public ULyraItemDefinition
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    TSubclassOf<UGameplayEffect> ConsumeEffect;
};
```

#### 3. Experience 设计模式

**单一游戏模式 Experience**:

```cpp
// DA_Experience_TeamDeathmatch
GameFeaturesToEnable:
  - /Game/ShooterCore
  - /Game/TeamSystem

DefaultPawnData: DA_Hero_Mannequin

Actions:
  - AddInputConfig(IMC_Default_KBM)
  - AddUILayout(W_HUD_Shooter)
  - LoadActionSet(AS_ShooterCore)
```

**组合式 Experience** (复用多个小 Experience):

```cpp
// DA_Experience_Base (基础功能)
Actions:
  - AddInputConfig
  - AddUILayout
  - AddGameplayRules

// DA_Experience_Shooter (射击专属)
ParentExperience: DA_Experience_Base
GameFeaturesToEnable:
  - /Game/ShooterCore
  - /Game/WeaponSystem

// DA_Experience_MOBA (MOBA 专属)
ParentExperience: DA_Experience_Base
GameFeaturesToEnable:
  - /Game/MOBACore
  - /Game/MinionSystem
```

#### 4. 网络开发建议

**服务器权威原则**：

```cpp
// ✅ 正确：服务器验证客户端请求
UFUNCTION(Server, Reliable, WithValidation)
void ServerPickupItem(ALyraItemPickup* Item)
{
    // 验证：客户端是否在拾取范围内？
    if (!IsValid(Item) || !IsInPickupRange(Item))
    {
        return;  // 拒绝非法请求
    }

    // 执行拾取逻辑
    AddItemToInventory(Item->GetItemData());
    Item->Destroy();
}

// ❌ 错误：信任客户端数据
void ClientPickupItem(ALyraItemPickup* Item)
{
    // 客户端直接执行，容易被作弊
    AddItemToInventory(Item->GetItemData());
}
```

**预测与回滚**：

```cpp
// 使用 GAS 的预测机制
UCLASS()
class UGA_Jump : public ULyraGameplayAbility
{
public:
    virtual bool CanActivateAbility(...) const override
    {
        // 预测条件检查
        return Super::CanActivateAbility(...) && !Character->IsJumping();
    }

    virtual void ActivateAbility(...) override
    {
        // 客户端预测：立即播放跳跃动画
        if (!HasAuthority())
        {
            PlayMontage(JumpMontage);
        }

        // 服务器执行：真正的跳跃逻辑
        CommitAbility(...);  // 消耗 Stamina 等资源
        Character->Jump();

        // 如果服务器拒绝，客户端会自动 Rollback
    }
};
```

### 调试与测试

#### 单元测试

Lyra 使用 **Automation Framework** 进行自动化测试：

```cpp
// LyraHealthComponent.spec.cpp
#include "CoreMinimal.h"
#include "Misc/AutomationTest.h"
#include "Character/LyraHealthComponent.h"

BEGIN_DEFINE_SPEC(FLyraHealthComponentSpec, "Lyra.Character.HealthComponent",
                   EAutomationTestFlags::ProductFilter | EAutomationTestFlags::ApplicationContextMask)

    ULyraHealthComponent* HealthComponent;

END_DEFINE_SPEC(FLyraHealthComponentSpec)

void FLyraHealthComponentSpec::Define()
{
    Describe("TakeDamage", [this]()
    {
        BeforeEach([this]()
        {
            // 创建测试对象
            HealthComponent = NewObject<ULyraHealthComponent>();
            HealthComponent->MaxHealth = 100.0f;
            HealthComponent->CurrentHealth = 100.0f;
        });

        It("should reduce health by damage amount", [this]()
        {
            float Damage = 25.0f;
            HealthComponent->TakeDamage(Damage, nullptr, nullptr);
            
            TestEqual("Health should be 75", HealthComponent->GetCurrentHealth(), 75.0f);
        });

        It("should not go below zero", [this]()
        {
            HealthComponent->TakeDamage(150.0f, nullptr, nullptr);
            
            TestTrue("Health should be clamped to 0", HealthComponent->GetCurrentHealth() >= 0.0f);
        });

        It("should trigger death event when health reaches zero", [this]()
        {
            bool bDeathTriggered = false;
            HealthComponent->OnDeathStarted.AddLambda([&](AActor*, AActor*, AActor*)
            {
                bDeathTriggered = true;
            });

            HealthComponent->TakeDamage(100.0f, nullptr, nullptr);

            TestTrue("Death event should be triggered", bDeathTriggered);
        });
    });
}
```

**运行测试**：
- 编辑器：窗口 → 测试自动化 → 运行所有测试
- 命令行：`UnrealEditor.exe LyraStarterGame.uproject -ExecCmds="Automation RunTests Lyra" -unattended`

#### 集成测试

使用 **Functional Testing** 插件创建可视化测试：

1. 内容浏览器 → 右键 → **Blueprints** → **Functional Test**
2. 放置测试 Actor 到测试地图
3. 编写测试逻辑（蓝图或 C++）

```cpp
UCLASS()
class ALyraFunctionalTest_Respawn : public AFunctionalTest
{
    GENERATED_BODY()

public:
    virtual void PrepareTest() override
    {
        Super::PrepareTest();

        // 生成测试用玩家
        TestPlayer = GetWorld()->SpawnActor<ALyraCharacter>(...);
    }

    virtual void StartTest() override
    {
        Super::StartTest();

        // 让玩家死亡
        TestPlayer->GetHealthComponent()->TakeDamage(999.0f, nullptr, nullptr);

        // 5 秒后检查是否重生
        GetWorld()->GetTimerManager().SetTimer(RespawnCheckTimer, [this]()
        {
            if (TestPlayer->IsAlive())
            {
                FinishTest(EFunctionalTestResult::Succeeded, TEXT("玩家成功重生"));
            }
            else
            {
                FinishTest(EFunctionalTestResult::Failed, TEXT("玩家未能重生"));
            }
        }, 5.0f, false);
    }

private:
    UPROPERTY()
    TObjectPtr<ALyraCharacter> TestPlayer;

    FTimerHandle RespawnCheckTimer;
};
```

---

## 总结与下一步

### 你学到了什么

通过本文，你已经：

1. ✅ 了解了 Lyra 的核心设计理念（数据驱动、组件化、模块化）
2. ✅ 熟悉了项目结构和关键模块（Experience、GAS、Game Features）
3. ✅ 完成了完整的环境搭建（引擎、项目、开发工具）
4. ✅ 运行了第一个 Lyra 游戏并体验了多种模式
5. ✅ 配置了开发环境和调试工具
6. ✅ 掌握了常见问题的解决方案
7. ✅ 学习了最佳实践和编码规范

### 下一步学习路径

本系列接下来的文章将深入每个核心系统：

#### 🏗️ **第二篇：模块化 Actor 组件系统详解**
- 深入理解 ULyraPawnExtensionComponent
- Init State 机制的工作原理
- 如何创建自己的 Pawn 组件
- 组件生命周期和依赖管理

#### 🎮 **第三篇：Experience 系统核心**
- Experience Definition 的完整结构
- 动态加载和卸载 Experience
- 自定义 Game Feature Actions
- 多模式游戏的架构设计

#### 🔌 **第四篇：Game Features 插件系统**
- 创建自己的 Game Feature 插件
- 插件的生命周期管理
- 依赖关系和版本控制
- 热加载和热更新机制

#### 💾 **第五篇：数据驱动设计实战**
- Data Assets 的高级用法
- Data Table 批量数据管理
- Curve Tables 和数值曲线
- 本地化和多语言支持

#### ⚔️ **第六篇：GAS 入门 - Gameplay Ability System**
- GAS 的核心概念（Ability、Attribute、Effect、Tag）
- 创建你的第一个 Gameplay Ability
- 网络同步和预测机制
- 与动画、音效的集成

... 还有 24 篇深度文章等着你！

### 推荐的实践项目

**初级项目**：
1. 创建一个新的武器（修改伤害、射速、模型）
2. 添加一个自定义角色组件（如"耐力系统"）
3. 设计一个简单的小地图（Team Deathmatch）

**中级项目**：
1. 开发一个新的游戏模式（如"夺旗模式"）
2. 实现一个装备系统（头盔、护甲、靴子）
3. 创建一个技能树系统

**高级项目**：
1. 开发一个完整的 MOBA 模式（英雄、小兵、防御塔）
2. 实现大逃杀机制（毒圈、空投、跳伞）
3. 构建自己的 UI 框架扩展

### 学习资源

#### 官方资源

- **UE5 官方文档**: [https://docs.unrealengine.com/5.5/](https://docs.unrealengine.com/5.5/)
- **Lyra 官方视频教程**: [Epic Games YouTube - Lyra Playlist](https://www.youtube.com/c/UnrealEngine)
- **GAS 官方文档**: [Gameplay Ability System](https://docs.unrealengine.com/5.5/en-US/gameplay-ability-system-for-unreal-engine/)

#### 社区资源

- **Unreal Slackers Discord**: [https://unrealslackers.org/](https://unrealslackers.org/)
- **GASDocumentation** (社区文档): [https://github.com/tranek/GASDocumentation](https://github.com/tranek/GASDocumentation)
- **Lyra Discussions** (Reddit): [r/unrealengine](https://www.reddit.com/r/unrealengine/)

#### 推荐阅读

- **《Unreal Engine 5 Game Programming》** - 官方编程指南
- **《Multiplayer Game Development with Unreal Engine 5》** - 网络游戏开发
- **《GAS Cookbook》** - 社区整理的 GAS 实战案例

---

## 附录

### A. 快捷键速查表

| 操作 | 快捷键 |
|------|--------|
| 编译代码 | Ctrl + Alt + F11 (Live Coding) |
| 运行游戏（编辑器内） | Alt + P |
| 运行游戏（独立窗口） | Alt + S |
| 停止游戏 | Esc |
| 打开控制台 | ` (反引号) |
| 保存当前关卡 | Ctrl + S |
| 保存所有 | Ctrl + Shift + S |
| 内容浏览器 | Ctrl + Space |
| 蓝图编辑器编译 | F7 |
| 查找资源 | Ctrl + P |
| 重新加载蓝图 | Ctrl + R |

### B. 常用控制台命令

| 命令 | 功能 |
|------|------|
| `stat fps` | 显示帧率 |
| `stat unit` | 显示 CPU/GPU/渲染时间 |
| `stat game` | 显示游戏线程统计 |
| `stat scenerendering` | 显示渲染统计 |
| `stat memory` | 显示内存使用 |
| `showdebug abilitysystem` | GAS 调试信息 |
| `Lyra.DumpExperience` | 输出当前 Experience 详情 |
| `netprofile` | 网络性能分析 |
| `r.ScreenPercentage 50` | 降低渲染分辨率（提升性能） |
| `t.MaxFPS 60` | 限制帧率 |
| `log LogLyra Verbose` | 启用详细日志 |

### C. 推荐的 VS Code 设置

```json
// .vscode/settings.json
{
    "C_Cpp.default.defines": [
        "WITH_EDITOR=1",
        "UE_BUILD_DEVELOPMENT=1"
    ],
    "C_Cpp.default.includePath": [
        "${workspaceFolder}/**",
        "C:/Program Files/Epic Games/UE_5.5/Engine/Source/**"
    ],
    "files.associations": {
        "*.h": "cpp",
        "*.cpp": "cpp",
        "*.uproject": "json",
        "*.uplugin": "json"
    },
    "files.exclude": {
        "**/.vs": true,
        "**/Binaries": true,
        "**/Intermediate": true,
        "**/DerivedDataCache": true,
        "**/Saved/Crashes": true
    },
    "search.exclude": {
        "**/Binaries": true,
        "**/Intermediate": true
    }
}
```

### D. Git 初始化脚本

```bash
#!/bin/bash
# init-lyra-repo.sh - 初始化 Lyra 项目的 Git 仓库

# 初始化 Git
git init

# 安装 Git LFS
git lfs install

# 添加 .gitattributes
cat > .gitattributes << 'EOF'
*.uasset filter=lfs diff=lfs merge=lfs -text
*.umap filter=lfs diff=lfs merge=lfs -text
*.ubulk filter=lfs diff=lfs merge=lfs -text
*.uexp filter=lfs diff=lfs merge=lfs -text
*.wav filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
EOF

# 添加 .gitignore
cat > .gitignore << 'EOF'
# UE5 临时文件
Intermediate/
Saved/
DerivedDataCache/
.vs/
*.sln
*.sln.DotSettings.user
*.suo

# Build 结果
Binaries/
Build/

# 插件临时文件
Plugins/*/Intermediate/
Plugins/*/Binaries/
EOF

# 首次提交
git add .
git commit -m "chore: 初始化 Lyra 项目"

echo "✅ Git 仓库初始化完成！"
echo "🔗 添加远程仓库: git remote add origin <your-repo-url>"
echo "📤 推送代码: git push -u origin main"
```

---

**🎉 恭喜你完成了第一篇文章的学习！**

你现在已经掌握了 Lyra 项目的基础知识，环境也搭建完成。接下来，我们将深入每个核心系统，一步步构建你自己的游戏架构。

**准备好了吗？让我们继续探索 Lyra 的强大功能！** 🚀

---

> **版本信息**  
> 文章版本: 1.0  
> UE 版本: 5.5.1  
> Lyra 版本: 5.5  
> 最后更新: 2026-02-12  
> 作者: OpenClaw AI Assistant  

---

**下一篇**: [《模块化 Actor 组件系统详解》](./02-modular-actor-components.md)

**返回目录**: [Lyra Deep Dive 系列](../README.md)