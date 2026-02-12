# Enhanced Input System：新一代输入系统

## 概述

Enhanced Input System（增强输入系统）是 Unreal Engine 5 引入的全新输入处理框架，旨在替代传统的输入绑定系统。它提供了更灵活、更强大且易于扩展的输入管理机制，特别适合复杂的现代游戏开发需求。

Lyra 项目作为 Epic 官方的示例项目，完全采用了 Enhanced Input System，展示了这套系统在实战中的最佳实践。本文将深入探讨 Enhanced Input 的核心概念、Lyra 中的输入架构设计，以及如何在实际项目中应用这些技术。

### 本文内容

- **Enhanced Input 基础**：核心概念、与旧系统的对比
- **Lyra 输入架构**：数据驱动设计、输入组件绑定
- **高级输入功能**：组合键、手柄反馈、调试工具
- **实战案例**：构建完整的输入系统

---

## 第一部分：Enhanced Input 基础

### 1.1 传统输入系统的问题

在 UE4 时代，输入系统主要基于 `InputComponent` 和项目设置中的输入映射。虽然这套系统能满足基本需求，但在复杂项目中暴露了诸多问题：

#### 问题 1：输入映射难以管理

```cpp
// 传统方式：硬编码输入绑定
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    // 所有输入绑定写死在代码里
    PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &ACharacter::Jump);
    PlayerInputComponent->BindAction("Fire", IE_Pressed, this, &AMyCharacter::Fire);
    PlayerInputComponent->BindAxis("MoveForward", this, &AMyCharacter::MoveForward);
    PlayerInputComponent->BindAxis("MoveRight", this, &AMyCharacter::MoveRight);
    PlayerInputComponent->BindAxis("Turn", this, &AMyCharacter::AddControllerYawInput);
    PlayerInputComponent->BindAxis("LookUp", this, &AMyCharacter::AddControllerPitchInput);
}
```

**问题**：
- 输入名称字符串容易拼写错误
- 修改输入需要重新编译代码
- 无法在运行时动态调整输入配置
- 不同游戏模式难以切换不同的输入方案

#### 问题 2：缺乏上下文管理

假设你的游戏有驾驶、射击、对话等多种模式，每种模式都有不同的输入需求：

```cpp
// 传统方式需要手动启用/禁用输入
void AMyCharacter::EnterVehicle()
{
    // 禁用角色输入
    DisableInput(GetController());
    
    // 启用载具输入
    Vehicle->EnableInput(GetController());
}

void AMyCharacter::ExitVehicle()
{
    // 反向操作
    Vehicle->DisableInput(GetController());
    EnableInput(GetController());
}
```

**问题**：
- 手动管理输入启用/禁用容易出错
- 多个输入源冲突时难以处理优先级
- 无法优雅地支持输入堆栈（如暂停菜单覆盖游戏输入）

#### 问题 3：输入处理功能有限

传统系统对输入的处理非常简单，只能获取原始输入值：

```cpp
void AMyCharacter::MoveForward(float Value)
{
    // 只能拿到原始值，需要自己处理死区、平滑等
    if (FMath::Abs(Value) > 0.1f) // 手动死区
    {
        AddMovementInput(GetActorForwardVector(), Value);
    }
}
```

**问题**：
- 没有内置的死区处理
- 没有输入平滑、缩放等修饰功能
- 没有组合键、连招等高级输入支持
- 手柄振动、自适应扳机等功能支持有限

#### 问题 4：调试困难

传统系统缺乏可视化调试工具：

- 无法实时查看当前激活的输入绑定
- 难以追踪输入事件的流向
- 无法在编辑器中快速测试不同的输入配置

---

### 1.2 Enhanced Input 核心概念

Enhanced Input System 通过引入几个核心概念，优雅地解决了上述问题。

#### 核心概念架构

```
Player Controller
    └── Enhanced Input Component
            ├── Input Mapping Context (Stack)
            │       ├── Priority: 1 (Highest)
            │       ├── Priority: 0
            │       └── Priority: -1
            │
            └── Input Actions (Triggered by contexts)
                    ├── Input Action: IA_Move (2D Vector)
                    ├── Input Action: IA_Jump (Digital)
                    ├── Input Action: IA_Fire (Digital)
                    └── Input Action: IA_Aim (Analog)
```

#### 1.2.1 Input Action（输入动作）

`Input Action` 是一个独立的数据资产，代表游戏中的一个逻辑动作（如"跳跃"、"射击"、"移动"）。

**创建 Input Action**：

在内容浏览器中：
1. 右键 → Input → Input Action
2. 命名为 `IA_Jump`（推荐使用 IA_ 前缀）

**Input Action 的属性**：

```cpp
// InputAction.h 核心属性
class UInputAction : public UDataAsset
{
    // 值类型：决定输入数据的类型
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    EInputActionValueType ValueType;
    
    // 可选值：
    // - Digital (bool)          : 按键按下/松开
    // - Axis1D (float)          : 单轴输入（如鼠标滚轮）
    // - Axis2D (FVector2D)      : 双轴输入（如移动、视角）
    // - Axis3D (FVector)        : 三轴输入（如 6DOF 控制器）
    
    // Triggers：触发条件（何时触发）
    UPROPERTY(EditAnywhere, Instanced)
    TArray<UInputTrigger*> Triggers;
    
    // Modifiers：修饰器（如何修改输入值）
    UPROPERTY(EditAnywhere, Instanced)
    TArray<UInputModifier*> Modifiers;
};
```

**示例：创建移动输入动作**

在编辑器中打开 `IA_Move`：

| 属性 | 值 | 说明 |
|------|-----|------|
| Value Type | Axis2D | 二维移动（前后左右） |
| Consume Input | True | 消耗输入，防止传递到下层 |

#### 1.2.2 Input Mapping Context（输入映射上下文）

`Input Mapping Context` (IMC) 是一个数据资产，将硬件输入（键盘按键、手柄摇杆）映射到 Input Action。

**创建 Input Mapping Context**：

1. 右键 → Input → Input Mapping Context
2. 命名为 `IMC_Default`（推荐使用 IMC_ 前缀）

**映射配置示例**：

打开 `IMC_Default`，添加映射：

```
Mappings:
├── IA_Move
│   ├── Keyboard: W, A, S, D
│   │       └── Modifiers: Swizzle Input Axis Values (W=Y+, S=Y-, A=X-, D=X+)
│   └── Gamepad: Left Thumbstick
│           └── Modifiers: Dead Zone (0.25), Scalar (1.0)
│
├── IA_Look
│   ├── Mouse: Mouse XY 2D-Axis
│   │       └── Modifiers: Negate (Y-axis), Scalar (0.5)
│   └── Gamepad: Right Thumbstick
│           └── Modifiers: Dead Zone (0.25), Scalar (2.5)
│
├── IA_Jump
│   ├── Keyboard: Space Bar
│   └── Gamepad: Face Button Bottom (A/Cross)
│
└── IA_Sprint
    ├── Keyboard: Left Shift
    │       └── Triggers: Hold (0.5s)
    └── Gamepad: Left Thumbstick Button
            └── Triggers: Hold (0.3s)
```

**代码中添加映射上下文**：

```cpp
#include "EnhancedInputSubsystems.h"
#include "EnhancedInputComponent.h"

void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();
    
    // 获取 Enhanced Input 子系统
    if (UEnhancedInputLocalPlayerSubsystem* Subsystem = 
        ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(GetLocalPlayer()))
    {
        // 添加映射上下文（优先级 0）
        Subsystem->AddMappingContext(DefaultMappingContext, 0);
    }
}
```

#### 1.2.3 Input Modifiers（输入修饰器）

Modifiers 在输入值传递给游戏逻辑之前对其进行处理。Enhanced Input 提供了丰富的内置 Modifier：

##### 常用 Modifiers

| Modifier | 功能 | 使用场景 |
|----------|------|----------|
| **Dead Zone** | 死区过滤 | 摇杆漂移、触摸精度 |
| **Scalar** | 缩放输入值 | 灵敏度调节 |
| **Negate** | 反转输入 | 反转 Y 轴 |
| **Smooth** | 平滑输入 | 减少抖动 |
| **Exponential** | 指数曲线 | 精准瞄准 + 快速转身 |
| **FOV Scaling** | FOV 缩放补偿 | 瞄准时保持一致灵敏度 |
| **Swizzle Input Axis Values** | 轴重映射 | WASD → XY 向量 |

##### Modifier 示例：死区处理

```cpp
// 内置的死区 Modifier
class UInputModifierDeadZone : public UInputModifier
{
public:
    // 死区类型
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EDeadZoneType Type = EDeadZoneType::Radial;
    
    // 死区下限（低于此值视为 0）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float LowerThreshold = 0.2f;
    
    // 死区上限（高于此值视为 1）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float UpperThreshold = 1.0f;
    
    virtual FInputActionValue ModifyRaw_Implementation(
        const UEnhancedPlayerInput* PlayerInput, 
        FInputActionValue CurrentValue, 
        float DeltaTime) override
    {
        FVector Value = CurrentValue.Get<FVector>();
        float Magnitude = Value.Size();
        
        if (Magnitude < LowerThreshold)
        {
            return FInputActionValue(FVector::ZeroVector);
        }
        
        if (Magnitude > UpperThreshold)
        {
            return FInputActionValue(Value.GetSafeNormal());
        }
        
        // 重新映射到 [0, 1] 范围
        float NewMagnitude = (Magnitude - LowerThreshold) / 
                             (UpperThreshold - LowerThreshold);
        return FInputActionValue(Value.GetSafeNormal() * NewMagnitude);
    }
};
```

**配置示例**（在 IMC 中）：

```
IA_Move → Gamepad Left Stick
    └── Modifier: Dead Zone
            ├── Type: Radial
            ├── Lower Threshold: 0.25
            └── Upper Threshold: 0.95
```

##### Modifier 示例：自定义灵敏度曲线

```cpp
// 自定义 Modifier：S 型曲线（慢速精准，快速灵敏）
UCLASS()
class UInputModifierSCurve : public UInputModifier
{
    GENERATED_BODY()

public:
    // 曲线陡峭度
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta = (ClampMin = "0.1", ClampMax = "10.0"))
    float Steepness = 2.0f;
    
    virtual FInputActionValue ModifyRaw_Implementation(
        const UEnhancedPlayerInput* PlayerInput,
        FInputActionValue CurrentValue,
        float DeltaTime) override
    {
        FVector2D Value = CurrentValue.Get<FVector2D>();
        
        // 对每个轴应用 S 型曲线
        auto ApplyCurve = [this](float Input) -> float
        {
            float Sign = FMath::Sign(Input);
            float Abs = FMath::Abs(Input);
            
            // S 型函数：f(x) = x^s / (x^s + (1-x)^s)
            float Curved = FMath::Pow(Abs, Steepness) / 
                           (FMath::Pow(Abs, Steepness) + 
                            FMath::Pow(1.0f - Abs, Steepness));
            
            return Sign * Curved;
        };
        
        Value.X = ApplyCurve(Value.X);
        Value.Y = ApplyCurve(Value.Y);
        
        return FInputActionValue(Value);
    }
};
```

**效果对比**：

| 输入值 | 线性 | S 曲线 (Steepness=2) |
|--------|------|---------------------|
| 0.1    | 0.1  | 0.03                |
| 0.3    | 0.3  | 0.16                |
| 0.5    | 0.5  | 0.50                |
| 0.7    | 0.7  | 0.84                |
| 1.0    | 1.0  | 1.00                |

低速时更精准（小输入被压缩），高速时响应灵敏（大输入被放大）。

#### 1.2.4 Input Triggers（输入触发器）

Triggers 决定何时触发 Input Action。不同的 Trigger 可以实现长按、连击、组合键等复杂逻辑。

##### 内置 Triggers

| Trigger | 功能 | 触发时机 |
|---------|------|----------|
| **Pressed** | 按下触发 | 按键按下瞬间 |
| **Released** | 释放触发 | 按键松开瞬间 |
| **Down** | 持续按住 | 每帧检测按住状态 |
| **Hold** | 长按触发 | 按住达到指定时间 |
| **Tap** | 轻点触发 | 短时间内按下并释放 |
| **Pulse** | 脉冲触发 | 按住时周期性触发 |
| **Chorded Action** | 组合键 | 需要其他按键同时按下 |
| **Combo** | 连招 | 按照顺序按下多个键 |

##### Trigger 示例：长按冲刺

```cpp
// 配置在 IMC 中
IA_Sprint → Keyboard Left Shift
    └── Trigger: Hold
            ├── Hold Time Threshold: 0.3 (秒)
            └── Is One Shot: False (持续触发)
```

**代码绑定**：

```cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    UEnhancedInputComponent* EnhancedInput = Cast<UEnhancedInputComponent>(PlayerInputComponent);
    
    // Started：开始长按
    EnhancedInput->BindAction(SprintAction, ETriggerEvent::Started, this, &AMyCharacter::StartSprint);
    
    // Ongoing：持续长按（每帧触发）
    EnhancedInput->BindAction(SprintAction, ETriggerEvent::Ongoing, this, &AMyCharacter::UpdateSprint);
    
    // Completed：长按结束
    EnhancedInput->BindAction(SprintAction, ETriggerEvent::Completed, this, &AMyCharacter::StopSprint);
    
    // Canceled：中途取消（如松开按键）
    EnhancedInput->BindAction(SprintAction, ETriggerEvent::Canceled, this, &AMyCharacter::StopSprint);
}

void AMyCharacter::StartSprint(const FInputActionValue& Value)
{
    UE_LOG(LogTemp, Log, TEXT("Sprint started!"));
    bIsSprinting = true;
}

void AMyCharacter::UpdateSprint(const FInputActionValue& Value)
{
    // 持续冲刺时的逻辑（如消耗体力）
    if (Stamina > 0)
    {
        Stamina -= GetWorld()->GetDeltaSeconds() * 10.0f;
    }
    else
    {
        StopSprint(Value);
    }
}

void AMyCharacter::StopSprint(const FInputActionValue& Value)
{
    UE_LOG(LogTemp, Log, TEXT("Sprint stopped!"));
    bIsSprinting = false;
}
```

##### Trigger 示例：组合键（Chorded Action）

实现 Ctrl+C 复制功能：

```cpp
// 配置在 IMC 中
IA_Copy → Keyboard C
    └── Trigger: Chorded Action
            └── Chord Action: IA_Ctrl (需要同时按下)

IA_Ctrl → Keyboard Left Ctrl
    (无特殊 Trigger)
```

**代码绑定**：

```cpp
EnhancedInput->BindAction(CopyAction, ETriggerEvent::Triggered, this, &AMyEditor::CopySelection);
```

只有在按住 Ctrl 的同时按下 C 才会触发 `IA_Copy`。

##### Trigger 示例：连招检测

实现格斗游戏中的"升龙拳"（前、下、前下+拳）：

```cpp
// 配置在 IMC 中
IA_Shoryuken → Keyboard J (拳)
    └── Trigger: Combo
            ├── Combo Steps:
            │   1. IA_MoveForward (前)
            │   2. IA_Crouch (下)
            │   3. IA_MoveForward (前下)
            ├── Time Between Steps: 0.5 (秒)
            └── Cancel On Completion: True
```

**自定义 Combo Trigger**：

```cpp
UCLASS()
class UInputTriggerCombo : public UInputTrigger
{
    GENERATED_BODY()

public:
    // 连招步骤
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<UInputAction*> ComboSteps;
    
    // 步骤之间的最大时间间隔
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float TimeBetweenSteps = 0.5f;
    
    virtual ETriggerState UpdateState_Implementation(
        const UEnhancedPlayerInput* PlayerInput,
        FInputActionValue ModifiedValue,
        float DeltaTime) override
    {
        // 检测连招逻辑
        int32 CurrentStep = GetCurrentComboStep(PlayerInput);
        
        if (CurrentStep == ComboSteps.Num() - 1)
        {
            // 连招完成
            ResetCombo();
            return ETriggerState::Triggered;
        }
        
        return ETriggerState::None;
    }
    
private:
    int32 CurrentComboStep = 0;
    float LastStepTime = 0.0f;
    
    // 实现细节省略...
};
```

---

### 1.3 Enhanced Input vs 传统输入系统

#### 对比总结

| 特性 | 传统输入系统 | Enhanced Input System |
|------|-------------|----------------------|
| **配置方式** | 项目设置 + 硬编码 | 数据资产 (IMC + IA) |
| **上下文管理** | 手动启用/禁用 | 自动管理上下文栈 |
| **输入处理** | 原始值 | Modifiers + Triggers |
| **热更新** | 需要重启 | 运行时切换 IMC |
| **优先级** | 无内置支持 | 上下文优先级 |
| **调试工具** | 无 | 可视化调试器 |
| **组合键** | 手动实现 | Chorded Action |
| **长按/连招** | 手动计时 | Triggers |
| **死区处理** | 手动代码 | Dead Zone Modifier |
| **灵敏度调节** | 手动代码 | Scalar Modifier |

#### 迁移示例：从旧系统到 Enhanced Input

**旧系统代码**：

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
    
private:
    void MoveForward(float Value);
    void MoveRight(float Value);
    void Turn(float Value);
    void LookUp(float Value);
    void StartJump();
    void StopJump();
    
    float MouseSensitivity = 0.5f;
};

// MyCharacter.cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    PlayerInputComponent->BindAxis("MoveForward", this, &AMyCharacter::MoveForward);
    PlayerInputComponent->BindAxis("MoveRight", this, &AMyCharacter::MoveRight);
    PlayerInputComponent->BindAxis("Turn", this, &AMyCharacter::Turn);
    PlayerInputComponent->BindAxis("LookUp", this, &AMyCharacter::LookUp);
    PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &AMyCharacter::StartJump);
    PlayerInputComponent->BindAction("Jump", IE_Released, this, &AMyCharacter::StopJump);
}

void AMyCharacter::MoveForward(float Value)
{
    if (Controller && Value != 0.0f)
    {
        const FRotator Rotation = Controller->GetControlRotation();
        const FRotator YawRotation(0, Rotation.Yaw, 0);
        const FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
        AddMovementInput(Direction, Value);
    }
}

void AMyCharacter::MoveRight(float Value)
{
    if (Controller && Value != 0.0f)
    {
        const FRotator Rotation = Controller->GetControlRotation();
        const FRotator YawRotation(0, Rotation.Yaw, 0);
        const FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
        AddMovementInput(Direction, Value);
    }
}

void AMyCharacter::Turn(float Value)
{
    AddControllerYawInput(Value * MouseSensitivity);
}

void AMyCharacter::LookUp(float Value)
{
    AddControllerPitchInput(Value * MouseSensitivity);
}
```

**Enhanced Input 版本**：

```cpp
// MyCharacter.h
#include "InputActionValue.h"

UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
    
protected:
    // Input Actions
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    UInputAction* MoveAction;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    UInputAction* LookAction;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    UInputAction* JumpAction;
    
    // Input Mapping Context
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    UInputMappingContext* DefaultMappingContext;
    
private:
    // 输入回调函数
    void Move(const FInputActionValue& Value);
    void Look(const FInputActionValue& Value);
};

// MyCharacter.cpp
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"

void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    // 添加映射上下文
    if (APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = 
            ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
        {
            Subsystem->AddMappingContext(DefaultMappingContext, 0);
        }
    }
    
    // 绑定输入动作
    UEnhancedInputComponent* EnhancedInput = Cast<UEnhancedInputComponent>(PlayerInputComponent);
    if (EnhancedInput)
    {
        // 移动（2D 向量）
        EnhancedInput->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyCharacter::Move);
        
        // 视角（2D 向量）
        EnhancedInput->BindAction(LookAction, ETriggerEvent::Triggered, this, &AMyCharacter::Look);
        
        // 跳跃（数字按键）
        EnhancedInput->BindAction(JumpAction, ETriggerEvent::Started, this, &ACharacter::Jump);
        EnhancedInput->BindAction(JumpAction, ETriggerEvent::Completed, this, &ACharacter::StopJumping);
    }
}

void AMyCharacter::Move(const FInputActionValue& Value)
{
    // 直接获取 2D 向量（WASD 或摇杆）
    const FVector2D MovementVector = Value.Get<FVector2D>();
    
    if (Controller)
    {
        // 获取控制器方向
        const FRotator Rotation = Controller->GetControlRotation();
        const FRotator YawRotation(0, Rotation.Yaw, 0);
        
        // 前后移动
        const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
        AddMovementInput(ForwardDirection, MovementVector.Y);
        
        // 左右移动
        const FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
        AddMovementInput(RightDirection, MovementVector.X);
    }
}

void AMyCharacter::Look(const FInputActionValue& Value)
{
    // 直接获取 2D 向量（鼠标移动或右摇杆）
    const FVector2D LookAxisVector = Value.Get<FVector2D>();
    
    if (Controller)
    {
        // Modifiers 已经在 IMC 中处理了灵敏度和 Y 轴反转
        AddControllerYawInput(LookAxisVector.X);
        AddControllerPitchInput(LookAxisVector.Y);
    }
}
```

**优势对比**：

1. **减少代码量**：
   - 旧系统：6 个回调函数，手动处理方向
   - 新系统：3 个回调函数，直接获取向量

2. **灵敏度配置**：
   - 旧系统：硬编码在 C++
   - 新系统：在 IMC 中配置 Scalar Modifier，无需重新编译

3. **死区处理**：
   - 旧系统：需要在每个轴回调中手动检查
   - 新系统：Dead Zone Modifier 自动处理

4. **Y 轴反转**：
   - 旧系统：需要在代码中添加选项逻辑
   - 新系统：在 IMC 中配置 Negate Modifier

---

### 1.4 基础实战：从零搭建 Enhanced Input

#### 步骤 1：启用插件

1. 编辑 → 插件
2. 搜索 "Enhanced Input"
3. 勾选启用，重启编辑器

#### 步骤 2：项目配置

编辑 `Config/DefaultEngine.ini`：

```ini
[/Script/Engine.Engine]
+ActiveGameNameRedirects=(OldGameName="AnyOldName",NewGameName="/Script/EnhancedInput")

[/Script/Engine.InputSettings]
DefaultPlayerInputClass=/Script/EnhancedInput.EnhancedPlayerInput
DefaultInputComponentClass=/Script/EnhancedInput.EnhancedInputComponent
```

或在编辑器中：
- 项目设置 → 引擎 → Input
- Default Classes → Default Player Input Class → `EnhancedPlayerInput`
- Default Classes → Default Input Component Class → `EnhancedInputComponent`

#### 步骤 3：创建输入资产

在内容浏览器 `Content/Input/` 文件夹中：

**3.1 创建 Input Actions**：

| 文件名 | 类型 | Value Type |
|--------|------|------------|
| `IA_Move` | Input Action | Axis2D |
| `IA_Look` | Input Action | Axis2D |
| `IA_Jump` | Input Action | Digital (bool) |
| `IA_Crouch` | Input Action | Digital (bool) |
| `IA_Sprint` | Input Action | Digital (bool) |

**3.2 创建 Input Mapping Context**：

创建 `IMC_Default`，添加映射：

```
Mappings:
├── IA_Move
│   ├── Keyboard W: Swizzle (Y+)
│   ├── Keyboard S: Swizzle (Y-)
│   ├── Keyboard A: Swizzle (X-)
│   ├── Keyboard D: Swizzle (X+)
│   └── Gamepad Left Thumbstick 2D-Axis
│           └── Modifiers:
│               └── Dead Zone (Lower: 0.25, Upper: 1.0)
│
├── IA_Look
│   ├── Mouse XY 2D-Axis
│   │       └── Modifiers:
│   │           ├── Negate (Y)
│   │           └── Scalar (0.5, 0.5)
│   └── Gamepad Right Thumbstick 2D-Axis
│           └── Modifiers:
│               ├── Dead Zone (Lower: 0.25, Upper: 1.0)
│               └── Scalar (2.0, 2.0)
│
├── IA_Jump
│   ├── Keyboard Space Bar
│   └── Gamepad Face Button Bottom
│
├── IA_Crouch
│   ├── Keyboard Left Ctrl
│   │       └── Triggers: Hold (0.2s)
│   └── Gamepad Face Button Right
│
└── IA_Sprint
    ├── Keyboard Left Shift
    └── Gamepad Left Thumbstick Button
```

#### 步骤 4：C++ 角色类实现

```cpp
// MyCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "InputActionValue.h"
#include "MyCharacter.generated.h"

class UInputAction;
class UInputMappingContext;

UCLASS()
class MYPROJECT_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    AMyCharacter();

protected:
    virtual void BeginPlay() override;
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

    // Input Actions
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* MoveAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* LookAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* JumpAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* CrouchAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* SprintAction;
    
    // Input Mapping Context
    UPROPERTY(EditDefaultsOnly, Category = "Input|Context")
    UInputMappingContext* DefaultMappingContext;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Context")
    int32 MappingPriority = 0;

private:
    // Input Callbacks
    void Input_Move(const FInputActionValue& Value);
    void Input_Look(const FInputActionValue& Value);
    void Input_Crouch_Started(const FInputActionValue& Value);
    void Input_Crouch_Completed(const FInputActionValue& Value);
    void Input_Sprint_Started(const FInputActionValue& Value);
    void Input_Sprint_Completed(const FInputActionValue& Value);
    
    // Sprint State
    bool bIsSprinting = false;
};
```

```cpp
// MyCharacter.cpp
#include "MyCharacter.h"
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Camera/CameraComponent.h"
#include "GameFramework/SpringArmComponent.h"

AMyCharacter::AMyCharacter()
{
    PrimaryActorTick.bCanEverTick = true;
    
    // 创建相机组件
    USpringArmComponent* SpringArm = CreateDefaultSubobject<USpringArmComponent>(TEXT("SpringArm"));
    SpringArm->SetupAttachment(RootComponent);
    SpringArm->TargetArmLength = 300.0f;
    SpringArm->bUsePawnControlRotation = true;
    
    UCameraComponent* Camera = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
    Camera->SetupAttachment(SpringArm);
    
    // 配置角色移动
    bUseControllerRotationYaw = false;
    GetCharacterMovement()->bOrientRotationToMovement = true;
    GetCharacterMovement()->RotationRate = FRotator(0.0f, 500.0f, 0.0f);
    GetCharacterMovement()->MaxWalkSpeed = 400.0f;
    GetCharacterMovement()->MaxWalkSpeedCrouched = 200.0f;
}

void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();
    
    // 添加映射上下文
    if (APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = 
            ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
        {
            Subsystem->AddMappingContext(DefaultMappingContext, MappingPriority);
        }
    }
}

void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    UEnhancedInputComponent* EnhancedInput = CastChecked<UEnhancedInputComponent>(PlayerInputComponent);
    
    // 绑定移动和视角（持续触发）
    EnhancedInput->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyCharacter::Input_Move);
    EnhancedInput->BindAction(LookAction, ETriggerEvent::Triggered, this, &AMyCharacter::Input_Look);
    
    // 绑定跳跃（按下和松开）
    EnhancedInput->BindAction(JumpAction, ETriggerEvent::Started, this, &ACharacter::Jump);
    EnhancedInput->BindAction(JumpAction, ETriggerEvent::Completed, this, &ACharacter::StopJumping);
    
    // 绑定蹲伏（长按触发）
    EnhancedInput->BindAction(CrouchAction, ETriggerEvent::Started, this, &AMyCharacter::Input_Crouch_Started);
    EnhancedInput->BindAction(CrouchAction, ETriggerEvent::Completed, this, &AMyCharacter::Input_Crouch_Completed);
    
    // 绑定冲刺
    EnhancedInput->BindAction(SprintAction, ETriggerEvent::Started, this, &AMyCharacter::Input_Sprint_Started);
    EnhancedInput->BindAction(SprintAction, ETriggerEvent::Completed, this, &AMyCharacter::Input_Sprint_Completed);
}

void AMyCharacter::Input_Move(const FInputActionValue& Value)
{
    const FVector2D MovementVector = Value.Get<FVector2D>();
    
    if (Controller)
    {
        const FRotator YawRotation(0, Controller->GetControlRotation().Yaw, 0);
        const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
        const FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
        
        AddMovementInput(ForwardDirection, MovementVector.Y);
        AddMovementInput(RightDirection, MovementVector.X);
    }
}

void AMyCharacter::Input_Look(const FInputActionValue& Value)
{
    const FVector2D LookVector = Value.Get<FVector2D>();
    
    if (Controller)
    {
        AddControllerYawInput(LookVector.X);
        AddControllerPitchInput(LookVector.Y);
    }
}

void AMyCharacter::Input_Crouch_Started(const FInputActionValue& Value)
{
    Crouch();
}

void AMyCharacter::Input_Crouch_Completed(const FInputActionValue& Value)
{
    UnCrouch();
}

void AMyCharacter::Input_Sprint_Started(const FInputActionValue& Value)
{
    bIsSprinting = true;
    GetCharacterMovement()->MaxWalkSpeed = 600.0f;
}

void AMyCharacter::Input_Sprint_Completed(const FInputActionValue& Value)
{
    bIsSprinting = false;
    GetCharacterMovement()->MaxWalkSpeed = 400.0f;
}
```

#### 步骤 5：在蓝图中配置

1. 创建角色蓝图 `BP_MyCharacter`（继承自 `AMyCharacter`）
2. 打开蓝图，在 Details 面板配置：

```
Input | Actions:
├── Move Action: IA_Move
├── Look Action: IA_Look
├── Jump Action: IA_Jump
├── Crouch Action: IA_Crouch
└── Sprint Action: IA_Sprint

Input | Context:
├── Default Mapping Context: IMC_Default
└── Mapping Priority: 0
```

3. 保存并在 Game Mode 中设置默认 Pawn 为 `BP_MyCharacter`

#### 步骤 6：测试

运行游戏，测试以下功能：

- ✅ WASD / 左摇杆：移动
- ✅ 鼠标 / 右摇杆：视角
- ✅ 空格 / A 键：跳跃
- ✅ Shift / 左摇杆按钮：冲刺
- ✅ Ctrl（长按0.2秒）/ B 键：蹲伏

---

## 第二部分：Lyra 中的输入架构

Lyra 项目展示了 Enhanced Input System 在大型项目中的最佳实践。其输入架构具有高度的模块化、数据驱动特性，并与 Gameplay Ability System (GAS) 深度集成。

### 2.1 Lyra 输入系统概览

#### 核心组件架构

```
Player Controller (ALyraPlayerController)
    └── Player State (ALyraPlayerState)
            └── Pawn (ALyraCharacter)
                    └── Pawn Extension Component (ULyraPawnExtensionComponent)
                            └── Hero Component (ULyraHeroComponent)
                                    ├── Input Component (ULyraInputComponent)
                                    ├── Input Config (ULyraInputConfig)
                                    └── Ability System Component
```

#### 输入流程

```
1. Experience 加载
        ↓
2. 应用 Pawn Data (包含 Input Config)
        ↓
3. LyraHeroComponent::InitializePlayerInput()
        ↓
4. 添加 Input Mapping Context
        ↓
5. 绑定 Native Input Actions → Ability System
        ↓
6. 游戏运行时接收输入
        ↓
7. Enhanced Input 处理 (Modifiers + Triggers)
        ↓
8. 触发 Gameplay Abilities 或 C++ 回调
```

---

### 2.2 LyraHeroComponent：输入管理中枢

`ULyraHeroComponent` 是 Lyra 输入系统的核心，负责初始化输入、绑定动作和管理输入配置。

#### 类定义

```cpp
// LyraHeroComponent.h
UCLASS(Blueprintable, Meta=(BlueprintSpawnableComponent))
class ULyraHeroComponent : public ULyraPawnComponent
{
    GENERATED_BODY()

public:
    ULyraHeroComponent(const FObjectInitializer& ObjectInitializer);

    // 初始化输入（在 Pawn 准备好后调用）
    void InitializePlayerInput(UInputComponent* PlayerInputComponent);

protected:
    virtual void OnRegister() override;
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    // 输入配置（来自 Pawn Data）
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    ULyraInputConfig* InputConfig;

    // 当前相机模式
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Camera")
    TSubclassOf<ULyraCameraMode> DefaultCameraMode;

private:
    // 输入回调：移动
    void Input_Move(const FInputActionValue& InputActionValue);
    
    // 输入回调：视角
    void Input_LookMouse(const FInputActionValue& InputActionValue);
    void Input_LookStick(const FInputActionValue& InputActionValue);
    
    // 输入回调：自动奔跑
    void Input_AutoRun(const FInputActionValue& InputActionValue);
    
    // 输入回调：蹲伏
    void Input_Crouch(const FInputActionValue& InputActionValue);

    // 输入绑定句柄（用于取消绑定）
    TArray<uint32> BindHandles;
};
```

#### 初始化输入

```cpp
// LyraHeroComponent.cpp

void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    check(PlayerInputComponent);
    
    const APawn* Pawn = GetPawn<APawn>();
    if (!Pawn)
    {
        return;
    }
    
    // 获取 Local Player
    const APlayerController* PC = GetController<APlayerController>();
    check(PC);
    
    const ULocalPlayer* LP = PC->GetLocalPlayer();
    check(LP);
    
    UEnhancedInputLocalPlayerSubsystem* Subsystem = 
        LP->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();
    check(Subsystem);
    
    // 清除现有映射上下文
    Subsystem->ClearAllMappings();
    
    // 添加默认映射上下文（来自 Pawn Data）
    if (const ULyraPawnData* PawnData = GetPawnData<ULyraPawnData>())
    {
        if (const ULyraInputConfig* InputConfigData = PawnData->InputConfig)
        {
            // 遍历所有输入映射上下文并添加
            for (const FInputMappingContextAndPriority& Mapping : InputConfigData->InputMappings)
            {
                if (UInputMappingContext* IMC = Mapping.InputMapping.LoadSynchronous())
                {
                    Subsystem->AddMappingContext(IMC, Mapping.Priority);
                }
            }
        }
    }
    
    // 绑定输入动作
    ULyraInputComponent* LyraIC = Cast<ULyraInputComponent>(PlayerInputComponent);
    if (!LyraIC)
    {
        return;
    }
    
    // 绑定 Native Actions（C++ 回调）
    {
        BIND_INPUT_ACTION(LyraIC, InputConfig, Move, Input_Move);
        BIND_INPUT_ACTION(LyraIC, InputConfig, LookMouse, Input_LookMouse);
        BIND_INPUT_ACTION(LyraIC, InputConfig, LookStick, Input_LookStick);
        BIND_INPUT_ACTION(LyraIC, InputConfig, Crouch, Input_Crouch);
        BIND_INPUT_ACTION(LyraIC, InputConfig, AutoRun, Input_AutoRun);
    }
    
    // 绑定 Ability Actions（GAS 技能）
    if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
    {
        LyraIC->BindAbilityActions(InputConfig, this, 
            &ULyraHeroComponent::Input_AbilityInputTagPressed,
            &ULyraHeroComponent::Input_AbilityInputTagReleased);
    }
}
```

**解析**：

1. **清除旧映射**：防止多次初始化时重复绑定
2. **加载 Input Mapping Context**：从 Pawn Data 获取配置
3. **绑定 Native Actions**：移动、视角等基础输入直接绑定 C++ 函数
4. **绑定 Ability Actions**：技能输入通过 GAS 处理

---

### 2.3 输入配置数据资产：ULyraInputConfig

#### 类定义

```cpp
// LyraInputConfig.h

/** 
 * Native 输入动作的标签绑定
 */
USTRUCT(BlueprintType)
struct FLyraInputAction
{
    GENERATED_BODY()

    // Input Action 资产
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<const UInputAction> InputAction = nullptr;
    
    // 关联的 Gameplay Tag（用于 GAS）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};

/**
 * Input Mapping Context 及其优先级
 */
USTRUCT(BlueprintType)
struct FInputMappingContextAndPriority
{
    GENERATED_BODY()

    // Input Mapping Context 软引用
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TSoftObjectPtr<UInputMappingContext> InputMapping;
    
    // 优先级（数字越大优先级越高）
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    int32 Priority = 0;
};

/**
 * Lyra 输入配置数据资产
 */
UCLASS(BlueprintType, Const)
class ULyraInputConfig : public UDataAsset
{
    GENERATED_BODY()

public:
    // 根据 Input Action 查找对应的 Gameplay Tag
    UFUNCTION(BlueprintCallable, Category = "Lyra|Input")
    const UInputAction* FindNativeInputActionForTag(const FGameplayTag& InputTag) const;
    
    // 根据 Gameplay Tag 查找对应的 Input Action
    UFUNCTION(BlueprintCallable, Category = "Lyra|Input")
    const UInputAction* FindAbilityInputActionForTag(const FGameplayTag& InputTag) const;

public:
    // Input Mapping Contexts（多个上下文按优先级叠加）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    TArray<FInputMappingContextAndPriority> InputMappings;
    
    // Native 输入动作（直接绑定 C++ 函数）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input", Meta = (TitleProperty = "InputAction"))
    TArray<FLyraInputAction> NativeInputActions;
    
    // Ability 输入动作（通过 GAS 处理）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input", Meta = (TitleProperty = "InputAction"))
    TArray<FLyraInputAction> AbilityInputActions;
};
```

#### 实现

```cpp
// LyraInputConfig.cpp

const UInputAction* ULyraInputConfig::FindNativeInputActionForTag(const FGameplayTag& InputTag) const
{
    for (const FLyraInputAction& Action : NativeInputActions)
    {
        if (Action.InputAction && Action.InputTag == InputTag)
        {
            return Action.InputAction;
        }
    }
    
    UE_LOG(LogLyra, Error, TEXT("Can't find NativeInputAction for InputTag [%s] on InputConfig [%s]."),
        *InputTag.ToString(), *GetNameSafe(this));
    
    return nullptr;
}

const UInputAction* ULyraInputConfig::FindAbilityInputActionForTag(const FGameplayTag& InputTag) const
{
    for (const FLyraInputAction& Action : AbilityInputActions)
    {
        if (Action.InputAction && Action.InputTag == InputTag)
        {
            return Action.InputAction;
        }
    }
    
    UE_LOG(LogLyra, Error, TEXT("Can't find AbilityInputAction for InputTag [%s] on InputConfig [%s]."),
        *InputTag.ToString(), *GetNameSafe(this));
    
    return nullptr;
}
```

#### 配置示例：DA_InputConfig_ShooterHero

在编辑器中创建 `ULyraInputConfig` 数据资产 `DA_InputConfig_ShooterHero`：

```
Input Mappings:
├── [0] IMC_Default_KBM (Priority: 0)          // 键鼠映射
├── [1] IMC_Default_Gamepad (Priority: 0)     // 手柄映射
└── [2] IMC_ShooterHero (Priority: 1)         // Shooter 特定映射（覆盖默认）

Native Input Actions:
├── [0] InputAction: IA_Move          | InputTag: InputTag.Move
├── [1] InputAction: IA_Look_Mouse    | InputTag: InputTag.Look.Mouse
├── [2] InputAction: IA_Look_Stick    | InputTag: InputTag.Look.Stick
├── [3] InputAction: IA_Crouch        | InputTag: InputTag.Crouch
└── [4] InputAction: IA_AutoRun       | InputTag: InputTag.AutoRun

Ability Input Actions:
├── [0] InputAction: IA_Jump          | InputTag: InputTag.Ability.Jump
├── [1] InputAction: IA_Fire          | InputTag: InputTag.Ability.Fire
├── [2] InputAction: IA_Aim           | InputTag: InputTag.Ability.Aim
├── [3] InputAction: IA_Reload        | InputTag: InputTag.Ability.Reload
├── [4] InputAction: IA_Melee         | InputTag: InputTag.Ability.Melee
└── [5] InputAction: IA_Grenade       | InputTag: InputTag.Ability.Grenade
```

**设计亮点**：

1. **多 IMC 叠加**：键鼠和手柄分开配置，共享相同的 Input Action
2. **Tag 驱动**：通过 Gameplay Tag 解耦输入和功能
3. **Native vs Ability**：
   - Native Actions：基础移动、视角（不需要 GAS）
   - Ability Actions：技能、交互（需要 GAS 管理）

---

### 2.4 LyraInputComponent：增强的输入组件

Lyra 扩展了 `UEnhancedInputComponent`，添加了绑定辅助函数。

#### 类定义

```cpp
// LyraInputComponent.h

UCLASS(Config = Input)
class ULyraInputComponent : public UEnhancedInputComponent
{
    GENERATED_BODY()

public:
    ULyraInputComponent(const FObjectInitializer& ObjectInitializer);

    // 绑定 Native Input Action（带 Gameplay Tag）
    template<class UserClass, typename FuncType>
    void BindNativeAction(const ULyraInputConfig* InputConfig, const FGameplayTag& InputTag,
        ETriggerEvent TriggerEvent, UserClass* Object, FuncType Func);
    
    // 绑定所有 Ability Input Actions 到 GAS
    template<class UserClass, typename PressedFuncType, typename ReleasedFuncType>
    void BindAbilityActions(const ULyraInputConfig* InputConfig, UserClass* Object,
        PressedFuncType PressedFunc, ReleasedFuncType ReleasedFunc);
};
```

#### 实现

```cpp
// LyraInputComponent.cpp

template<class UserClass, typename FuncType>
void ULyraInputComponent::BindNativeAction(const ULyraInputConfig* InputConfig, 
    const FGameplayTag& InputTag, ETriggerEvent TriggerEvent, UserClass* Object, FuncType Func)
{
    check(InputConfig);
    
    if (const UInputAction* IA = InputConfig->FindNativeInputActionForTag(InputTag))
    {
        BindAction(IA, TriggerEvent, Object, Func);
    }
}

template<class UserClass, typename PressedFuncType, typename ReleasedFuncType>
void ULyraInputComponent::BindAbilityActions(const ULyraInputConfig* InputConfig, 
    UserClass* Object, PressedFuncType PressedFunc, ReleasedFuncType ReleasedFunc)
{
    check(InputConfig);
    
    for (const FLyraInputAction& Action : InputConfig->AbilityInputActions)
    {
        if (Action.InputAction && Action.InputTag.IsValid())
        {
            // 按下时触发
            if (PressedFunc)
            {
                BindAction(Action.InputAction, ETriggerEvent::Triggered, Object, PressedFunc, Action.InputTag);
            }
            
            // 松开时触发
            if (ReleasedFunc)
            {
                BindAction(Action.InputAction, ETriggerEvent::Completed, Object, ReleasedFunc, Action.InputTag);
            }
        }
    }
}
```

#### 使用示例

```cpp
// LyraHeroComponent.cpp

void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    ULyraInputComponent* LyraIC = CastChecked<ULyraInputComponent>(PlayerInputComponent);
    
    // 绑定移动输入
    LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Move, 
        ETriggerEvent::Triggered, this, &ULyraHeroComponent::Input_Move);
    
    // 绑定视角输入（鼠标）
    LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Look_Mouse, 
        ETriggerEvent::Triggered, this, &ULyraHeroComponent::Input_LookMouse);
    
    // 绑定所有技能输入到 GAS
    LyraIC->BindAbilityActions(InputConfig, this, 
        &ULyraHeroComponent::Input_AbilityInputTagPressed,
        &ULyraHeroComponent::Input_AbilityInputTagReleased);
}

void ULyraHeroComponent::Input_AbilityInputTagPressed(FGameplayTag InputTag)
{
    if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
    {
        LyraASC->AbilityInputTagPressed(InputTag);
    }
}

void ULyraHeroComponent::Input_AbilityInputTagReleased(FGameplayTag InputTag)
{
    if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
    {
        LyraASC->AbilityInputTagReleased(InputTag);
    }
}
```

---

### 2.5 Native Input Actions vs Ability Input Actions

#### 设计哲学

| 类型 | 用途 | 处理方式 | 示例 |
|------|------|----------|------|
| **Native Actions** | 基础功能，不需要 GAS | 直接绑定 C++ 函数 | 移动、视角、蹲伏、自动奔跑 |
| **Ability Actions** | 游戏性技能，需要 GAS 管理 | 通过 Ability System Component 触发 Gameplay Ability | 跳跃、射击、瞄准、换弹、投掷 |

#### 为什么要区分？

**Native Actions 优势**：
- 性能更高（无 GAS 开销）
- 逻辑简单（直接调用函数）
- 适合频繁触发的输入（如移动、视角）

**Ability Actions 优势**：
- 统一的技能管理（冷却、消耗、授权）
- 网络同步（预测、回滚）
- 动态授予/撤销（装备武器时添加射击能力）

#### Native Action 示例：移动

```cpp
void ULyraHeroComponent::Input_Move(const FInputActionValue& InputActionValue)
{
    APawn* Pawn = GetPawn<APawn>();
    AController* Controller = Pawn ? Pawn->GetController() : nullptr;
    
    if (Controller)
    {
        const FVector2D Value = InputActionValue.Get<FVector2D>();
        const FRotator MovementRotation(0.0f, Controller->GetControlRotation().Yaw, 0.0f);
        
        if (Value.X != 0.0f)
        {
            const FVector MovementDirection = MovementRotation.RotateVector(FVector::RightVector);
            Pawn->AddMovementInput(MovementDirection, Value.X);
        }
        
        if (Value.Y != 0.0f)
        {
            const FVector MovementDirection = MovementRotation.RotateVector(FVector::ForwardVector);
            Pawn->AddMovementInput(MovementDirection, Value.Y);
        }
    }
}
```

**特点**：
- 无 GAS 依赖
- 直接操作 Pawn 的移动输入
- 每帧调用（ETriggerEvent::Triggered）

#### Ability Action 示例：跳跃

```cpp
// 在 LyraAbilitySystemComponent 中处理
void ULyraAbilitySystemComponent::AbilityInputTagPressed(const FGameplayTag& InputTag)
{
    if (InputTag.IsValid())
    {
        // 遍历所有激活的技能
        for (const FGameplayAbilitySpec& AbilitySpec : GetActivatableAbilities())
        {
            if (AbilitySpec.Ability && AbilitySpec.DynamicAbilityTags.HasTagExact(InputTag))
            {
                // 记录输入按下
                InputPressedSpecHandles.AddUnique(AbilitySpec.Handle);
                InputHeldSpecHandles.AddUnique(AbilitySpec.Handle);
                
                // 尝试激活技能
                if (!AbilitySpec.IsActive())
                {
                    TryActivateAbility(AbilitySpec.Handle);
                }
            }
        }
    }
}
```

**特点**：
- 通过 GAS 管理
- 支持冷却、消耗、网络同步
- 技能可以动态授予/移除

---

### 2.6 输入优先级与冲突处理

#### Input Mapping Context 优先级

多个 IMC 可以同时激活，优先级高的 IMC 会覆盖低优先级的映射。

**示例场景**：玩家进入载具

```cpp
// 步骤 1：默认角色输入（优先级 0）
Subsystem->AddMappingContext(IMC_Default, 0);

// 步骤 2：进入载具，添加载具输入（优先级 1，覆盖默认）
void AMyVehicle::OnEnter(AMyCharacter* Character)
{
    if (APlayerController* PC = Character->GetController<APlayerController>())
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = 
            ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
        {
            // 添加载具输入（优先级 1，W/S 控制油门/刹车）
            Subsystem->AddMappingContext(IMC_Vehicle, 1);
        }
    }
}

// 步骤 3：离开载具，移除载具输入
void AMyVehicle::OnExit(AMyCharacter* Character)
{
    if (APlayerController* PC = Character->GetController<APlayerController>())
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = 
            ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
        {
            // 移除载具输入，恢复角色移动
            Subsystem->RemoveMappingContext(IMC_Vehicle);
        }
    }
}
```

**配置对比**：

| Key | IMC_Default (Priority 0) | IMC_Vehicle (Priority 1) |
|-----|--------------------------|--------------------------|
| W   | IA_Move (Forward)        | IA_Vehicle_Throttle      |
| S   | IA_Move (Backward)       | IA_Vehicle_Brake         |
| A   | IA_Move (Left)           | IA_Vehicle_SteerLeft     |
| D   | IA_Move (Right)          | IA_Vehicle_SteerRight    |
| Space | IA_Jump                | IA_Vehicle_Handbrake     |

在载具中时，`IMC_Vehicle` 的映射会完全覆盖 `IMC_Default`。

#### Consume Input：消耗输入

某些输入可能同时触发多个 Action，可以设置 `Consume Input` 防止传递到下层。

**示例**：UI 打开时阻止游戏输入

```cpp
// IMC_UI（优先级 10）
IA_UI_Click → Left Mouse Button
    └── Consume Input: True  // 阻止鼠标点击传递到游戏

// IMC_Default（优先级 0）
IA_Fire → Left Mouse Button
    (在 UI 打开时不会触发，因为被 UI 消耗了)
```

#### Lyra 的 UI 输入管理

Lyra 使用 `UCommonUIActionRouterBase` 管理 UI 输入：

```cpp
// 打开 UI 时
void ULyraUIManagerSubsystem::OpenMenu(TSubclassOf<UCommonActivatableWidget> WidgetClass)
{
    UCommonActivatableWidget* Widget = CreateWidget<UCommonActivatableWidget>(GetWorld(), WidgetClass);
    
    // 添加到 UI 栈（自动管理输入优先级）
    if (UCommonUIActionRouterBase* ActionRouter = GetActionRouter())
    {
        ActionRouter->PushWidget(Widget);
    }
}

// 关闭 UI 时
void ULyraUIManagerSubsystem::CloseMenu()
{
    if (UCommonUIActionRouterBase* ActionRouter = GetActionRouter())
    {
        ActionRouter->PopWidget(); // 自动恢复游戏输入
    }
}
```

---

### 2.7 动态切换 Input Mapping Context

#### 场景示例：游戏模式切换

假设游戏有多种模式：

- **Exploration（探索）**：自由移动，无战斗输入
- **Combat（战斗）**：完整战斗输入，禁用某些探索功能
- **Dialogue（对话）**：仅 UI 输入，禁用移动

**实现方式**：

```cpp
UCLASS()
class UGameModeInputManager : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    UInputMappingContext* IMC_Exploration;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    UInputMappingContext* IMC_Combat;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    UInputMappingContext* IMC_Dialogue;
    
    void SwitchToExploration(APlayerController* PC)
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = GetSubsystem(PC))
        {
            Subsystem->RemoveMappingContext(IMC_Combat);
            Subsystem->RemoveMappingContext(IMC_Dialogue);
            Subsystem->AddMappingContext(IMC_Exploration, 0);
        }
    }
    
    void SwitchToCombat(APlayerController* PC)
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = GetSubsystem(PC))
        {
            Subsystem->RemoveMappingContext(IMC_Exploration);
            Subsystem->RemoveMappingContext(IMC_Dialogue);
            Subsystem->AddMappingContext(IMC_Combat, 0);
        }
    }
    
    void SwitchToDialogue(APlayerController* PC)
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = GetSubsystem(PC))
        {
            Subsystem->RemoveMappingContext(IMC_Exploration);
            Subsystem->RemoveMappingContext(IMC_Combat);
            Subsystem->AddMappingContext(IMC_Dialogue, 0);
        }
    }
    
private:
    UEnhancedInputLocalPlayerSubsystem* GetSubsystem(APlayerController* PC)
    {
        if (ULocalPlayer* LP = PC->GetLocalPlayer())
        {
            return LP->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();
        }
        return nullptr;
    }
};
```

**使用示例**：

```cpp
// 玩家进入战斗区域
void ATriggerVolume_CombatZone::OnActorBeginOverlap(AActor* OverlappedActor, AActor* OtherActor)
{
    if (AMyCharacter* Character = Cast<AMyCharacter>(OtherActor))
    {
        if (APlayerController* PC = Character->GetController<APlayerController>())
        {
            InputManager->SwitchToCombat(PC);
        }
    }
}

// 玩家开始对话
void ANPCCharacter::StartDialogue(AMyCharacter* Player)
{
    if (APlayerController* PC = Player->GetController<APlayerController>())
    {
        InputManager->SwitchToDialogue(PC);
    }
}
```

---

## 第三部分：高级输入功能

### 3.1 组合键与连续输入

#### 实现组合键：Chorded Action

**场景**：Ctrl+Q 丢弃物品

```cpp
// 步骤 1：创建 Input Actions
IA_Modifier_Ctrl    // Value Type: Digital (bool)
IA_DropItem         // Value Type: Digital (bool)

// 步骤 2：配置 IMC
Mappings:
├── IA_Modifier_Ctrl → Keyboard Left Ctrl
│       (无特殊配置，仅用于检测)
│
└── IA_DropItem → Keyboard Q
        └── Triggers:
                └── Chorded Action
                        └── Chord Action: IA_Modifier_Ctrl
```

**代码绑定**：

```cpp
EnhancedInput->BindAction(DropItemAction, ETriggerEvent::Triggered, this, &AMyCharacter::DropCurrentItem);

void AMyCharacter::DropCurrentItem(const FInputActionValue& Value)
{
    // 只有在按住 Ctrl 的同时按下 Q 才会触发
    if (UInventoryComponent* Inventory = GetInventoryComponent())
    {
        Inventory->DropItem(SelectedItemIndex);
    }
}
```

#### 实现连招：Combo

**场景**：格斗游戏三连击（轻拳 → 重拳 → 上勾拳）

**自定义 Combo Trigger**：

```cpp
UCLASS(meta = (DisplayName = "Combo"))
class UInputTriggerCombo : public UInputTrigger
{
    GENERATED_BODY()

public:
    // 连招步骤（按顺序）
    UPROPERTY(EditAnywhere, Category = "Trigger Settings")
    TArray<TObjectPtr<const UInputAction>> ComboActions;
    
    // 步骤之间的最大时间间隔（秒）
    UPROPERTY(EditAnywhere, Category = "Trigger Settings")
    float MaxTimeBetweenActions = 0.5f;
    
    // 完成连招后是否重置
    UPROPERTY(EditAnywhere, Category = "Trigger Settings")
    bool bResetOnCompletion = true;

protected:
    virtual ETriggerType GetTriggerType_Implementation() const override
    {
        return ETriggerType::Explicit;
    }
    
    virtual ETriggerState UpdateState_Implementation(
        const UEnhancedPlayerInput* PlayerInput,
        FInputActionValue ModifiedValue,
        float DeltaTime) override
    {
        // 检查是否完成连招
        if (CheckComboCompletion(PlayerInput))
        {
            if (bResetOnCompletion)
            {
                ResetCombo();
            }
            return ETriggerState::Triggered;
        }
        
        // 检查超时
        if (CurrentStep > 0 && GetTimeSinceLastInput() > MaxTimeBetweenActions)
        {
            ResetCombo();
            return ETriggerState::None;
        }
        
        return ETriggerState::Ongoing;
    }

private:
    int32 CurrentStep = 0;
    float LastInputTime = 0.0f;
    
    bool CheckComboCompletion(const UEnhancedPlayerInput* PlayerInput)
    {
        if (CurrentStep >= ComboActions.Num())
        {
            return true;
        }
        
        // 检查当前步骤的输入
        const UInputAction* ExpectedAction = ComboActions[CurrentStep];
        if (PlayerInput->GetActionValue(ExpectedAction).Get<bool>())
        {
            CurrentStep++;
            LastInputTime = PlayerInput->GetOuterAPlayerController()->GetWorld()->GetTimeSeconds();
            return CurrentStep >= ComboActions.Num();
        }
        
        return false;
    }
    
    void ResetCombo()
    {
        CurrentStep = 0;
        LastInputTime = 0.0f;
    }
    
    float GetTimeSinceLastInput() const
    {
        // 实现省略...
        return 0.0f;
    }
};
```

**配置示例**：

```cpp
// IMC 配置
IA_Combo_Ultimate → Keyboard K (Heavy Punch)
    └── Triggers:
            └── Combo
                    ├── Combo Actions:
                    │   [0] IA_LightPunch (J)
                    │   [1] IA_HeavyPunch (K)
                    │   [2] IA_Uppercut (L)
                    └── Max Time Between Actions: 0.8s
```

**代码绑定**：

```cpp
EnhancedInput->BindAction(ComboUltimateAction, ETriggerEvent::Triggered, this, &AFighterCharacter::ExecuteComboUltimate);

void AFighterCharacter::ExecuteComboUltimate(const FInputActionValue& Value)
{
    UE_LOG(LogTemp, Warning, TEXT("COMBO ULTIMATE!!!"));
    PlayAnimMontage(ComboUltimateMontage);
    DealDamage(EnemyTarget, 150.0f);
}
```

---

### 3.2 手柄振动与力反馈

Enhanced Input 支持现代手柄的高级反馈功能。

#### 基础振动：Rumble

```cpp
void AWeaponBase::Fire()
{
    // 播放射击振动
    if (APlayerController* PC = GetOwningPlayerController())
    {
        // 简单振动（强度，持续时间，左右马达）
        PC->PlayDynamicForceFeedback(
            0.5f,              // Intensity
            0.1f,              // Duration
            true,              // bAffectsLeftLarge (低频大马达)
            true,              // bAffectsLeftSmall (高频小马达)
            true,              // bAffectsRightLarge
            true               // bAffectsRightSmall
        );
    }
}
```

#### 高级反馈：Force Feedback Effect

创建 Force Feedback Effect 资产：

1. 内容浏览器 → 右键 → Miscellaneous → Force Feedback Effect
2. 命名为 `FFE_WeaponFire`
3. 配置振动曲线：

```
Channel 0 (Left Large):
    - 0.0s: 0.0
    - 0.05s: 1.0 (峰值)
    - 0.15s: 0.0

Channel 1 (Left Small):
    - 0.0s: 0.3
    - 0.1s: 0.8
    - 0.2s: 0.0

Channel 2 (Right Large):
    (同 Channel 0)

Channel 3 (Right Small):
    (同 Channel 1)
```

**代码使用**：

```cpp
UCLASS()
class AWeaponBase : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Feedback")
    UForceFeedbackEffect* FireFeedbackEffect;
    
    void Fire()
    {
        // 播放预设的力反馈效果
        if (APlayerController* PC = GetOwningPlayerController())
        {
            PC->ClientPlayForceFeedback(FireFeedbackEffect, false, true, NAME_None);
        }
    }
};
```

#### PS5 DualSense 自适应扳机

```cpp
#include "GameFramework/InputDeviceProperties.h"

void AWeaponBase::EnableAdaptiveTriggers()
{
    if (APlayerController* PC = GetOwningPlayerController())
    {
        // 创建自适应扳机效果
        UInputDeviceTriggerEffect* TriggerEffect = NewObject<UInputDeviceTriggerFeedbackProperty>();
        
        // 配置右扳机（R2）的阻力
        if (UInputDeviceTriggerFeedbackProperty* FeedbackProp = 
            Cast<UInputDeviceTriggerFeedbackProperty>(TriggerEffect))
        {
            FeedbackProp->TriggerPositionParam.StartPosition = 0.2f;  // 开始阻力位置
            FeedbackProp->TriggerPositionParam.StartValue = 5;        // 初始阻力强度 (0-8)
            FeedbackProp->TriggerPositionParam.EndPosition = 0.5f;
            FeedbackProp->TriggerPositionParam.EndValue = 8;          // 最大阻力强度
            
            FeedbackProp->TriggerLocation = EInputDeviceTriggerMask::Right; // R2 扳机
        }
        
        // 应用效果
        PC->SetInputDeviceTriggerEffect(TriggerEffect);
    }
}

void AWeaponBase::DisableAdaptiveTriggers()
{
    if (APlayerController* PC = GetOwningPlayerController())
    {
        // 重置扳机
        UInputDeviceTriggerEffect* ResetEffect = NewObject<UInputDeviceTriggerResetProperty>();
        PC->SetInputDeviceTriggerEffect(ResetEffect);
    }
}
```

**应用场景**：

| 武器类型 | 扳机效果 |
|----------|----------|
| 手枪 | 模拟扳机拉力（0.0-0.5 位置，阻力 3-7） |
| 步枪 | 中等阻力（0.0-0.8 位置，阻力 5） |
| 霰弹枪 | 强阻力 + 后坐力振动（阻力 8，振动 0.5s） |
| 弓箭 | 渐进阻力（模拟拉弦，0.0-1.0 位置，阻力 1-8） |
| 空枪 | 无阻力（阻力 0） |

---

### 3.3 输入调试工具

#### 控制台命令

Enhanced Input 提供了丰富的调试命令：

```cpp
// 显示当前所有激活的 Input Mapping Context
showdebug enhancedinput

// 在屏幕上显示输入事件日志
EnhancedInput.bShouldLogAllWorldSubsystemInputs 1

// 显示当前输入状态（按键是否按下）
EnhancedInput.bShowKeyStatus 1

// 记录输入处理详细信息
EnhancedInput.bLogAllInputs 1
```

#### 可视化调试器

在编辑器中：

1. **窗口 → Developer Tools → Enhanced Input Debugger**
2. 运行游戏（PIE）
3. 调试器会实时显示：
   - 当前激活的 Input Mapping Contexts（按优先级排序）
   - 每个 Input Action 的状态（Triggered / Ongoing / None）
   - Modifier 和 Trigger 的处理结果
   - 输入值的变化曲线

**调试器界面**：

```
┌─ Enhanced Input Debugger ────────────────────────────────┐
│ Player: Player0                                           │
├───────────────────────────────────────────────────────────┤
│ Input Mapping Contexts:                                   │
│   [Priority 10] IMC_UI (Active)                           │
│   [Priority 1]  IMC_Combat (Active)                       │
│   [Priority 0]  IMC_Default (Active)                      │
├───────────────────────────────────────────────────────────┤
│ Input Actions (Active):                                   │
│   IA_Move:        Value=(0.5, 0.8)  State=Triggered      │
│   IA_Look:        Value=(120, -30)  State=Triggered      │
│   IA_Fire:        Value=1.0          State=Ongoing        │
│   IA_Jump:        Value=0.0          State=None           │
├───────────────────────────────────────────────────────────┤
│ Modifiers Applied (IA_Move):                              │
│   1. Swizzle Input Axis (W→Y+) → (0.0, 1.0)             │
│   2. Dead Zone (0.25)          → (0.0, 1.0)             │
│   3. Scalar (1.5)              → (0.0, 1.5)             │
│   Final Value: (0.5, 0.8)                                │
├───────────────────────────────────────────────────────────┤
│ Triggers Evaluated (IA_Fire):                             │
│   1. Hold (0.3s) → Ongoing (Held for 0.18s)             │
└───────────────────────────────────────────────────────────┘
```

#### 自定义日志输出

```cpp
void ULyraHeroComponent::Input_Move(const FInputActionValue& Value)
{
    const FVector2D MoveVector = Value.Get<FVector2D>();
    
    // 调试日志
    UE_LOG(LogLyraInput, Verbose, TEXT("[Input_Move] Value: X=%.2f, Y=%.2f"), 
        MoveVector.X, MoveVector.Y);
    
    // 在屏幕上显示（开发构建）
    if (GEngine)
    {
        GEngine->AddOnScreenDebugMessage(-1, 0.0f, FColor::Green, 
            FString::Printf(TEXT("Move: (%.2f, %.2f)"), MoveVector.X, MoveVector.Y));
    }
    
    // 实际移动逻辑...
}
```

**启用详细日志**：

在 `Config/DefaultEngine.ini` 中：

```ini
[Core.Log]
LogLyraInput=Verbose
LogEnhancedInput=Verbose
```

#### 输入录制与回放

Enhanced Input 支持录制玩家输入并回放，用于测试和 AI 训练。

```cpp
// 开始录制
void AMyPlayerController::StartRecordingInput(const FString& FileName)
{
    if (UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(InputComponent))
    {
        FString FilePath = FPaths::ProjectSavedDir() / TEXT("InputRecordings") / FileName;
        
        // 创建录制器
        InputRecorder = NewObject<UInputRecorder>(this);
        InputRecorder->StartRecording(EIC, FilePath);
    }
}

// 停止录制
void AMyPlayerController::StopRecordingInput()
{
    if (InputRecorder)
    {
        InputRecorder->StopRecording();
        InputRecorder = nullptr;
    }
}

// 回放录制
void AMyPlayerController::PlaybackRecording(const FString& FileName)
{
    if (UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(InputComponent))
    {
        FString FilePath = FPaths::ProjectSavedDir() / TEXT("InputRecordings") / FileName;
        
        InputPlayback = NewObject<UInputPlayback>(this);
        InputPlayback->StartPlayback(EIC, FilePath);
    }
}
```

---

### 3.4 移动端触摸输入适配

#### 配置触摸输入

创建专门的移动端 IMC：

```cpp
// IMC_Mobile 配置
Mappings:
├── IA_Move
│   └── Touch 0 (Left Virtual Joystick)
│           └── Modifiers:
│               └── Dead Zone (Lower: 0.1, Upper: 1.0)
│
├── IA_Look
│   └── Touch 1 (Right Screen Area)
│           └── Modifiers:
│               └── Scalar (2.0, 2.0)
│
├── IA_Fire
│   └── Touch Any (Right Bottom Button)
│
└── IA_Jump
    └── Touch Any (Right Top Button)
```

#### 平台检测与 IMC 切换

```cpp
void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();
    
    if (UEnhancedInputLocalPlayerSubsystem* Subsystem = 
        ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(GetLocalPlayer()))
    {
        // 根据平台选择 IMC
        UInputMappingContext* IMC = nullptr;
        
        if (UPlatformInfo::IsMobile())
        {
            IMC = IMC_Mobile;
        }
        else if (UPlatformInfo::IsConsole())
        {
            IMC = IMC_Gamepad;
        }
        else
        {
            IMC = IMC_KeyboardMouse;
        }
        
        Subsystem->AddMappingContext(IMC, 0);
    }
}
```

#### 虚拟摇杆实现

```cpp
UCLASS()
class UVirtualJoystickWidget : public UUserWidget
{
    GENERATED_BODY()

protected:
    virtual FReply NativeOnTouchStarted(const FGeometry& Geometry, const FPointerEvent& Event) override
    {
        TouchStartLocation = Event.GetScreenSpacePosition();
        bIsTouching = true;
        return FReply::Handled().CaptureMouse(TakeWidget());
    }
    
    virtual FReply NativeOnTouchMoved(const FGeometry& Geometry, const FPointerEvent& Event) override
    {
        if (bIsTouching)
        {
            FVector2D CurrentLocation = Event.GetScreenSpacePosition();
            FVector2D Delta = (CurrentLocation - TouchStartLocation) / JoystickRadius;
            
            // 限制范围
            if (Delta.Size() > 1.0f)
            {
                Delta.Normalize();
            }
            
            // 注入输入到 Enhanced Input 系统
            if (APlayerController* PC = GetOwningPlayer())
            {
                if (UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(PC->InputComponent))
                {
                    // 模拟摇杆输入
                    FInputActionValue ActionValue(Delta);
                    EIC->InjectInputForAction(MoveAction, ActionValue);
                }
            }
        }
        
        return FReply::Handled();
    }
    
    virtual FReply NativeOnTouchEnded(const FGeometry& Geometry, const FPointerEvent& Event) override
    {
        bIsTouching = false;
        
        // 重置输入
        if (APlayerController* PC = GetOwningPlayer())
        {
            if (UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(PC->InputComponent))
            {
                EIC->InjectInputForAction(MoveAction, FInputActionValue(FVector2D::ZeroVector));
            }
        }
        
        return FReply::Handled().ReleaseMouseCapture();
    }

private:
    UPROPERTY(EditAnywhere, Category = "Input")
    UInputAction* MoveAction;
    
    UPROPERTY(EditAnywhere, Category = "Joystick")
    float JoystickRadius = 100.0f;
    
    FVector2D TouchStartLocation;
    bool bIsTouching = false;
};
```

---

## 第四部分：实战案例

### 4.1 案例 1：完整的第三人称射击角色输入系统

#### 目标

构建一个完整的 TPS 角色，支持：

- 移动（WASD / 左摇杆）
- 视角（鼠标 / 右摇杆）
- 跳跃（空格 / A 键）
- 冲刺（Shift / 左摇杆按钮）
- 蹲伏（Ctrl 长按 / B 键）
- 瞄准（右键 / LT）
- 射击（左键 / RT）
- 换弹（R / X 键）

#### 步骤 1：创建 Input Actions

在 `Content/Input/Actions/` 创建：

```
IA_Move.uasset          (Axis2D)
IA_Look.uasset          (Axis2D)
IA_Jump.uasset          (Digital)
IA_Sprint.uasset        (Digital)
IA_Crouch.uasset        (Digital)
IA_Aim.uasset           (Digital)
IA_Fire.uasset          (Digital)
IA_Reload.uasset        (Digital)
```

#### 步骤 2：创建 Input Mapping Contexts

**IMC_TPS_KBM（键鼠）**：

```
Mappings:
├── IA_Move
│   ├── W: Swizzle (Y+, 1.0)
│   ├── S: Swizzle (Y-, 1.0)
│   ├── A: Swizzle (X-, 1.0)
│   └── D: Swizzle (X+, 1.0)
│
├── IA_Look
│   └── Mouse XY 2D-Axis
│       └── Modifiers:
│           ├── Negate (Y)
│           └── Scalar (0.5, 0.5)
│
├── IA_Jump → Space
├── IA_Sprint → Left Shift
├── IA_Crouch → Left Ctrl (Trigger: Hold 0.2s)
├── IA_Aim → Right Mouse Button
├── IA_Fire → Left Mouse Button
└── IA_Reload → R
```

**IMC_TPS_Gamepad（手柄）**：

```
Mappings:
├── IA_Move
│   └── Left Thumbstick 2D-Axis
│       └── Modifiers: Dead Zone (0.25)
│
├── IA_Look
│   └── Right Thumbstick 2D-Axis
│       └── Modifiers:
│           ├── Dead Zone (0.25)
│           └── Scalar (2.5, 2.5)
│
├── IA_Jump → Face Button Bottom (A/Cross)
├── IA_Sprint → Left Thumbstick Button
├── IA_Crouch → Face Button Right (B/Circle)
├── IA_Aim → Left Trigger
├── IA_Fire → Right Trigger
└── IA_Reload → Face Button Top (Y/Triangle)
```

#### 步骤 3：角色类实现

```cpp
// TPSCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "InputActionValue.h"
#include "TPSCharacter.generated.h"

class UInputAction;
class UInputMappingContext;
class UCameraComponent;
class USpringArmComponent;

UCLASS()
class TPSGAME_API ATPSCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    ATPSCharacter();

protected:
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

    // === Components ===
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Camera")
    USpringArmComponent* CameraBoom;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Camera")
    UCameraComponent* FollowCamera;

    // === Input ===
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* MoveAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* LookAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* JumpAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* SprintAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* CrouchAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* AimAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* FireAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Actions")
    UInputAction* ReloadAction;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input|Contexts")
    UInputMappingContext* DefaultMappingContext;

    // === Movement Settings ===
    UPROPERTY(EditDefaultsOnly, Category = "Movement")
    float WalkSpeed = 400.0f;
    
    UPROPERTY(EditDefaultsOnly, Category = "Movement")
    float SprintSpeed = 600.0f;
    
    UPROPERTY(EditDefaultsOnly, Category = "Movement")
    float AimWalkSpeed = 250.0f;

    // === Camera Settings ===
    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    float DefaultCameraDistance = 300.0f;
    
    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    float AimCameraDistance = 150.0f;
    
    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    float CameraLerpSpeed = 10.0f;

private:
    // === Input Callbacks ===
    void Input_Move(const FInputActionValue& Value);
    void Input_Look(const FInputActionValue& Value);
    void Input_Sprint_Started(const FInputActionValue& Value);
    void Input_Sprint_Completed(const FInputActionValue& Value);
    void Input_Crouch_Started(const FInputActionValue& Value);
    void Input_Crouch_Completed(const FInputActionValue& Value);
    void Input_Aim_Started(const FInputActionValue& Value);
    void Input_Aim_Completed(const FInputActionValue& Value);
    void Input_Fire_Started(const FInputActionValue& Value);
    void Input_Fire_Ongoing(const FInputActionValue& Value);
    void Input_Fire_Completed(const FInputActionValue& Value);
    void Input_Reload(const FInputActionValue& Value);

    // === State ===
    bool bIsSprinting = false;
    bool bIsAiming = false;
    bool bIsFiring = false;
    
    // === Weapon ===
    int32 CurrentAmmo = 30;
    int32 MaxAmmo = 30;
    float FireRate = 0.1f;
    float LastFireTime = 0.0f;
};
```

```cpp
// TPSCharacter.cpp
#include "TPSCharacter.h"
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"
#include "Camera/CameraComponent.h"
#include "GameFramework/SpringArmComponent.h"
#include "GameFramework/CharacterMovementComponent.h"

ATPSCharacter::ATPSCharacter()
{
    PrimaryActorTick.bCanEverTick = true;
    
    // 配置角色移动
    bUseControllerRotationYaw = false;
    GetCharacterMovement()->bOrientRotationToMovement = true;
    GetCharacterMovement()->RotationRate = FRotator(0.0f, 540.0f, 0.0f);
    GetCharacterMovement()->MaxWalkSpeed = WalkSpeed;
    
    // 创建相机组件
    CameraBoom = CreateDefaultSubobject<USpringArmComponent>(TEXT("CameraBoom"));
    CameraBoom->SetupAttachment(RootComponent);
    CameraBoom->TargetArmLength = DefaultCameraDistance;
    CameraBoom->bUsePawnControlRotation = true;
    CameraBoom->SocketOffset = FVector(0.0f, 50.0f, 70.0f);
    
    FollowCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("FollowCamera"));
    FollowCamera->SetupAttachment(CameraBoom, USpringArmComponent::SocketName);
    FollowCamera->bUsePawnControlRotation = false;
}

void ATPSCharacter::BeginPlay()
{
    Super::BeginPlay();
    
    // 添加映射上下文
    if (APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = 
            ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
        {
            Subsystem->AddMappingContext(DefaultMappingContext, 0);
        }
    }
}

void ATPSCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // 平滑过渡相机距离
    float TargetDistance = bIsAiming ? AimCameraDistance : DefaultCameraDistance;
    float CurrentDistance = CameraBoom->TargetArmLength;
    CameraBoom->TargetArmLength = FMath::FInterpTo(CurrentDistance, TargetDistance, DeltaTime, CameraLerpSpeed);
}

void ATPSCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    UEnhancedInputComponent* EnhancedInput = CastChecked<UEnhancedInputComponent>(PlayerInputComponent);
    
    // 移动和视角
    EnhancedInput->BindAction(MoveAction, ETriggerEvent::Triggered, this, &ATPSCharacter::Input_Move);
    EnhancedInput->BindAction(LookAction, ETriggerEvent::Triggered, this, &ATPSCharacter::Input_Look);
    
    // 跳跃
    EnhancedInput->BindAction(JumpAction, ETriggerEvent::Started, this, &ACharacter::Jump);
    EnhancedInput->BindAction(JumpAction, ETriggerEvent::Completed, this, &ACharacter::StopJumping);
    
    // 冲刺
    EnhancedInput->BindAction(SprintAction, ETriggerEvent::Started, this, &ATPSCharacter::Input_Sprint_Started);
    EnhancedInput->BindAction(SprintAction, ETriggerEvent::Completed, this, &ATPSCharacter::Input_Sprint_Completed);
    
    // 蹲伏
    EnhancedInput->BindAction(CrouchAction, ETriggerEvent::Started, this, &ATPSCharacter::Input_Crouch_Started);
    EnhancedInput->BindAction(CrouchAction, ETriggerEvent::Completed, this, &ATPSCharacter::Input_Crouch_Completed);
    
    // 瞄准
    EnhancedInput->BindAction(AimAction, ETriggerEvent::Started, this, &ATPSCharacter::Input_Aim_Started);
    EnhancedInput->BindAction(AimAction, ETriggerEvent::Completed, this, &ATPSCharacter::Input_Aim_Completed);
    
    // 射击
    EnhancedInput->BindAction(FireAction, ETriggerEvent::Started, this, &ATPSCharacter::Input_Fire_Started);
    EnhancedInput->BindAction(FireAction, ETriggerEvent::Ongoing, this, &ATPSCharacter::Input_Fire_Ongoing);
    EnhancedInput->BindAction(FireAction, ETriggerEvent::Completed, this, &ATPSCharacter::Input_Fire_Completed);
    
    // 换弹
    EnhancedInput->BindAction(ReloadAction, ETriggerEvent::Started, this, &ATPSCharacter::Input_Reload);
}

void ATPSCharacter::Input_Move(const FInputActionValue& Value)
{
    const FVector2D MovementVector = Value.Get<FVector2D>();
    
    if (Controller)
    {
        const FRotator YawRotation(0, Controller->GetControlRotation().Yaw, 0);
        const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
        const FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
        
        AddMovementInput(ForwardDirection, MovementVector.Y);
        AddMovementInput(RightDirection, MovementVector.X);
    }
}

void ATPSCharacter::Input_Look(const FInputActionValue& Value)
{
    const FVector2D LookVector = Value.Get<FVector2D>();
    
    if (Controller)
    {
        AddControllerYawInput(LookVector.X);
        AddControllerPitchInput(LookVector.Y);
    }
}

void ATPSCharacter::Input_Sprint_Started(const FInputActionValue& Value)
{
    if (bIsAiming)
    {
        return; // 瞄准时无法冲刺
    }
    
    bIsSprinting = true;
    GetCharacterMovement()->MaxWalkSpeed = SprintSpeed;
}

void ATPSCharacter::Input_Sprint_Completed(const FInputActionValue& Value)
{
    bIsSprinting = false;
    GetCharacterMovement()->MaxWalkSpeed = bIsAiming ? AimWalkSpeed : WalkSpeed;
}

void ATPSCharacter::Input_Crouch_Started(const FInputActionValue& Value)
{
    Crouch();
}

void ATPSCharacter::Input_Crouch_Completed(const FInputActionValue& Value)
{
    UnCrouch();
}

void ATPSCharacter::Input_Aim_Started(const FInputActionValue& Value)
{
    bIsAiming = true;
    bIsSprinting = false; // 取消冲刺
    GetCharacterMovement()->MaxWalkSpeed = AimWalkSpeed;
    GetCharacterMovement()->bOrientRotationToMovement = false;
    bUseControllerRotationYaw = true;
}

void ATPSCharacter::Input_Aim_Completed(const FInputActionValue& Value)
{
    bIsAiming = false;
    GetCharacterMovement()->MaxWalkSpeed = WalkSpeed;
    GetCharacterMovement()->bOrientRotationToMovement = true;
    bUseControllerRotationYaw = false;
}

void ATPSCharacter::Input_Fire_Started(const FInputActionValue& Value)
{
    if (CurrentAmmo <= 0)
    {
        UE_LOG(LogTemp, Warning, TEXT("Out of ammo! Press R to reload."));
        return;
    }
    
    bIsFiring = true;
    
    // 立即射击第一发
    PerformFire();
}

void ATPSCharacter::Input_Fire_Ongoing(const FInputActionValue& Value)
{
    if (!bIsFiring || CurrentAmmo <= 0)
    {
        return;
    }
    
    // 控制射速
    float CurrentTime = GetWorld()->GetTimeSeconds();
    if (CurrentTime - LastFireTime >= FireRate)
    {
        PerformFire();
    }
}

void ATPSCharacter::Input_Fire_Completed(const FInputActionValue& Value)
{
    bIsFiring = false;
}

void ATPSCharacter::PerformFire()
{
    if (CurrentAmmo <= 0)
    {
        return;
    }
    
    CurrentAmmo--;
    LastFireTime = GetWorld()->GetTimeSeconds();
    
    // 射线检测
    FVector CameraLocation;
    FRotator CameraRotation;
    GetController()->GetPlayerViewPoint(CameraLocation, CameraRotation);
    
    FVector TraceStart = CameraLocation;
    FVector TraceEnd = TraceStart + (CameraRotation.Vector() * 10000.0f);
    
    FHitResult HitResult;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this);
    
    if (GetWorld()->LineTraceSingleByChannel(HitResult, TraceStart, TraceEnd, 
        ECC_Visibility, QueryParams))
    {
        // 命中目标
        UE_LOG(LogTemp, Log, TEXT("Hit: %s"), *HitResult.GetActor()->GetName());
        
        // TODO: 应用伤害
    }
    
    // 播放射击特效
    // TODO: PlayFireAnimation(), SpawnMuzzleFlash(), PlayFireSound()
    
    UE_LOG(LogTemp, Log, TEXT("Fire! Ammo: %d/%d"), CurrentAmmo, MaxAmmo);
}

void ATPSCharacter::Input_Reload(const FInputActionValue& Value)
{
    if (CurrentAmmo >= MaxAmmo)
    {
        UE_LOG(LogTemp, Warning, TEXT("Already full ammo!"));
        return;
    }
    
    // TODO: 播放换弹动画
    UE_LOG(LogTemp, Log, TEXT("Reloading..."));
    
    // 简单重置弹药（实际应该等动画播放完成）
    CurrentAmmo = MaxAmmo;
}
```

#### 步骤 4：蓝图配置

1. 创建角色蓝图 `BP_TPSCharacter`（继承自 `ATPSCharacter`）
2. 配置 Input Actions 和 Mapping Context
3. 设置 Game Mode 的默认 Pawn 为 `BP_TPSCharacter`

#### 测试清单

- ✅ WASD / 左摇杆：移动
- ✅ 鼠标 / 右摇杆：视角
- ✅ 空格 / A：跳跃
- ✅ Shift / 左摇杆按钮：冲刺（瞄准时禁用）
- ✅ Ctrl 长按 / B：蹲伏
- ✅ 右键 / LT：瞄准（相机拉近，降低移速）
- ✅ 左键 / RT：射击（自动射击，直到松开或弹药耗尽）
- ✅ R / X：换弹

---

### 4.2 案例 2：技能释放输入系统（快捷键 + 连招）

#### 目标

实现 MOBA/MMO 风格的技能系统：

- 快捷键 Q/W/E/R 释放技能
- 技能冷却管理
- 连招检测（Q → W → E 触发终极技能）
- 与 GAS 集成

#### 步骤 1：创建 Input Actions 和 Tags

```cpp
// GameplayTags (在项目中定义)
InputTag.Ability.Skill1   // Q
InputTag.Ability.Skill2   // W
InputTag.Ability.Skill3   // E
InputTag.Ability.Ultimate // R
InputTag.Ability.Combo    // Q+W+E 连招

// Input Actions
IA_Skill1.uasset    (Digital)
IA_Skill2.uasset    (Digital)
IA_Skill3.uasset    (Digital)
IA_Ultimate.uasset  (Digital)
```

#### 步骤 2：配置 IMC

```cpp
// IMC_Skills
Mappings:
├── IA_Skill1 → Q
├── IA_Skill2 → W
├── IA_Skill3 → E
└── IA_Ultimate → R
```

#### 步骤 3：创建 Gameplay Abilities

```cpp
// GA_Skill1.h (技能 1：火球术)
UCLASS()
class UGA_Skill1 : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Skill1()
    {
        // 配置技能标签
        AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Ability.Skill.Fireball")));
        ActivationOwnedTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("State.Casting")));
        ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("State.Dead")));
        
        // 输入标签
        InputTag = FGameplayTag::RequestGameplayTag(TEXT("InputTag.Ability.Skill1"));
        
        // 冷却时间
        CooldownDuration = 5.0f;
        ManaCost = 50.0f;
    }

protected:
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData) override
    {
        if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }
        
        // 播放施法动画
        PlayMontageAndWait(FireballCastMontage);
        
        // 生成火球
        SpawnFireballProjectile();
        
        EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
    }

private:
    void SpawnFireballProjectile()
    {
        AActor* Avatar = GetAvatarActorFromActorInfo();
        FVector SpawnLocation = Avatar->GetActorLocation() + Avatar->GetActorForwardVector() * 100.0f;
        FRotator SpawnRotation = Avatar->GetActorRotation();
        
        // 生成火球 Actor
        GetWorld()->SpawnActor<AFireballProjectile>(FireballClass, SpawnLocation, SpawnRotation);
    }

    UPROPERTY(EditDefaultsOnly, Category = "Ability")
    TSubclassOf<AFireballProjectile> FireballClass;
    
    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    UAnimMontage* FireballCastMontage;
    
    float CooldownDuration;
    float ManaCost;
};
```

#### 步骤 4：配置 Input Config

```cpp
// DA_InputConfig_Skills (ULyraInputConfig)

Ability Input Actions:
├── [0] InputAction: IA_Skill1    | InputTag: InputTag.Ability.Skill1
├── [1] InputAction: IA_Skill2    | InputTag: InputTag.Ability.Skill2
├── [2] InputAction: IA_Skill3    | InputTag: InputTag.Ability.Skill3
└── [3] InputAction: IA_Ultimate  | InputTag: InputTag.Ability.Ultimate
```

#### 步骤 5：授予技能

```cpp
// 在角色初始化时授予技能
void AMyCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    
    if (UAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        // 授予所有技能
        ASC->GiveAbility(FGameplayAbilitySpec(UGA_Skill1::StaticClass(), 1));
        ASC->GiveAbility(FGameplayAbilitySpec(UGA_Skill2::StaticClass(), 1));
        ASC->GiveAbility(FGameplayAbilitySpec(UGA_Skill3::StaticClass(), 1));
        ASC->GiveAbility(FGameplayAbilitySpec(UGA_Ultimate::StaticClass(), 1));
    }
}
```

#### 步骤 6：连招检测

使用自定义 Trigger 检测 Q → W → E 连招：

```cpp
UCLASS(meta = (DisplayName = "Skill Combo"))
class UInputTriggerSkillCombo : public UInputTrigger
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Combo")
    TArray<TObjectPtr<const UInputAction>> ComboSequence;
    
    UPROPERTY(EditAnywhere, Category = "Combo")
    float MaxTimeBetweenInputs = 1.5f;

protected:
    virtual ETriggerState UpdateState_Implementation(
        const UEnhancedPlayerInput* PlayerInput,
        FInputActionValue ModifiedValue,
        float DeltaTime) override
    {
        // 检测连招逻辑
        UpdateComboProgress(PlayerInput);
        
        if (CurrentStep >= ComboSequence.Num())
        {
            ResetCombo();
            return ETriggerState::Triggered;
        }
        
        return ETriggerState::None;
    }

private:
    int32 CurrentStep = 0;
    float LastInputTime = 0.0f;
    
    void UpdateComboProgress(const UEnhancedPlayerInput* PlayerInput);
    void ResetCombo();
};
```

**配置**：

```cpp
// IMC_Skills
IA_Ultimate → R
    └── Triggers:
            └── Skill Combo
                    ├── Combo Sequence:
                    │   [0] IA_Skill1 (Q)
                    │   [1] IA_Skill2 (W)
                    │   [2] IA_Skill3 (E)
                    └── Max Time Between Inputs: 2.0s
```

**效果**：当玩家在 2 秒内依次按下 Q、W、E、R 时，触发终极技能。

---

### 4.3 案例 3：自定义 Input Modifier（平滑 + 死区）

#### 目标

创建一个自定义 Modifier，同时实现：

- 径向死区过滤
- 指数平滑（减少抖动）
- 动态灵敏度（瞄准时降低）

#### 实现

```cpp
// InputModifierSmartAim.h
#pragma once

#include "InputModifiers.h"
#include "InputModifierSmartAim.generated.h"

/**
 * 智能瞄准 Modifier：死区 + 平滑 + 动态灵敏度
 */
UCLASS(meta = (DisplayName = "Smart Aim"))
class MYGAME_API UInputModifierSmartAim : public UInputModifier
{
    GENERATED_BODY()

public:
    // 死区下限
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Dead Zone", meta = (ClampMin = "0.0", ClampMax = "1.0"))
    float DeadZone = 0.25f;
    
    // 平滑系数（0=无平滑，1=完全平滑）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Smoothing", meta = (ClampMin = "0.0", ClampMax = "1.0"))
    float SmoothFactor = 0.3f;
    
    // 基础灵敏度
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Sensitivity")
    float BaseSensitivity = 1.0f;
    
    // 瞄准时灵敏度倍率
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Sensitivity")
    float AimSensitivityMultiplier = 0.5f;
    
    // 是否正在瞄准（外部设置）
    UPROPERTY(BlueprintReadWrite, Category = "State")
    bool bIsAiming = false;

protected:
    virtual FInputActionValue ModifyRaw_Implementation(
        const UEnhancedPlayerInput* PlayerInput,
        FInputActionValue CurrentValue,
        float DeltaTime) override
    {
        FVector2D Value = CurrentValue.Get<FVector2D>();
        
        // 步骤 1：应用死区
        Value = ApplyDeadZone(Value);
        
        // 步骤 2：应用平滑
        Value = ApplySmoothing(Value, DeltaTime);
        
        // 步骤 3：应用动态灵敏度
        Value = ApplySensitivity(Value);
        
        return FInputActionValue(Value);
    }

private:
    FVector2D LastValue = FVector2D::ZeroVector;
    
    FVector2D ApplyDeadZone(const FVector2D& Input)
    {
        float Magnitude = Input.Size();
        
        if (Magnitude < DeadZone)
        {
            return FVector2D::ZeroVector;
        }
        
        // 重新映射到 [0, 1]
        float NewMagnitude = (Magnitude - DeadZone) / (1.0f - DeadZone);
        return Input.GetSafeNormal() * FMath::Clamp(NewMagnitude, 0.0f, 1.0f);
    }
    
    FVector2D ApplySmoothing(const FVector2D& Input, float DeltaTime)
    {
        if (SmoothFactor <= 0.0f)
        {
            LastValue = Input;
            return Input;
        }
        
        // 指数平滑：NewValue = Lerp(OldValue, Input, Alpha)
        float Alpha = FMath::Clamp(1.0f - SmoothFactor, 0.0f, 1.0f);
        FVector2D SmoothedValue = FMath::Lerp(LastValue, Input, Alpha);
        
        LastValue = SmoothedValue;
        return SmoothedValue;
    }
    
    FVector2D ApplySensitivity(const FVector2D& Input)
    {
        float Sensitivity = bIsAiming ? (BaseSensitivity * AimSensitivityMultiplier) : BaseSensitivity;
        return Input * Sensitivity;
    }
};
```

#### 使用

```cpp
// IMC 配置
IA_Look → Mouse XY 2D-Axis
    └── Modifiers:
            └── Smart Aim
                    ├── Dead Zone: 0.2
                    ├── Smooth Factor: 0.3
                    ├── Base Sensitivity: 1.0
                    └── Aim Sensitivity Multiplier: 0.5

// 在角色中控制瞄准状态
void ATPSCharacter::Input_Aim_Started(const FInputActionValue& Value)
{
    bIsAiming = true;
    
    // 通知 Modifier（需要通过 Subsystem 访问）
    if (APlayerController* PC = GetController<APlayerController>())
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = 
            ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
        {
            // 找到 Look Action 的 Modifier 并设置状态
            // （实际需要更复杂的查找逻辑）
            // SmartAimModifier->bIsAiming = true;
        }
    }
}
```

---

### 4.4 案例 4：多设备输入自动切换

#### 目标

实现键鼠和手柄的无缝切换：

- 检测最后使用的输入设备
- 自动切换对应的 IMC
- 更新 UI 提示（键盘图标 ↔ 手柄图标）

#### 实现

```cpp
// InputDeviceManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "InputDeviceManager.generated.h"

UENUM(BlueprintType)
enum class EInputDeviceType : uint8
{
    KeyboardMouse,
    Gamepad,
    Touch
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnInputDeviceChanged, EInputDeviceType, NewDevice);

UCLASS()
class MYGAME_API UInputDeviceManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
    
    // 当前输入设备
    UFUNCTION(BlueprintCallable, Category = "Input")
    EInputDeviceType GetCurrentInputDevice() const { return CurrentDevice; }
    
    // 输入设备切换事件
    UPROPERTY(BlueprintAssignable, Category = "Input")
    FOnInputDeviceChanged OnInputDeviceChanged;

private:
    EInputDeviceType CurrentDevice = EInputDeviceType::KeyboardMouse;
    
    // 监听输入事件
    void OnKeyPressed(FKey Key);
    void OnMouseMoved();
    void OnGamepadButtonPressed(FKey Key);
    
    void SwitchToDevice(EInputDeviceType NewDevice);
};
```

```cpp
// InputDeviceManager.cpp
#include "InputDeviceManager.h"
#include "Framework/Application/SlateApplication.h"

void UInputDeviceManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    
    // 注册输入监听器
    if (FSlateApplication::IsInitialized())
    {
        FSlateApplication& SlateApp = FSlateApplication::Get();
        
        // 监听所有输入事件
        SlateApp.OnApplicationPreInputKeyDownListener().AddUObject(this, &UInputDeviceManager::OnKeyPressed);
    }
}

void UInputDeviceManager::Deinitialize()
{
    Super::Deinitialize();
    
    // 取消注册
    if (FSlateApplication::IsInitialized())
    {
        FSlateApplication& SlateApp = FSlateApplication::Get();
        SlateApp.OnApplicationPreInputKeyDownListener().RemoveAll(this);
    }
}

void UInputDeviceManager::OnKeyPressed(const FKeyEvent& KeyEvent)
{
    FKey Key = KeyEvent.GetKey();
    
    // 检测设备类型
    if (Key.IsMouseButton() || Key.IsAxis1D() && !Key.IsGamepadKey())
    {
        SwitchToDevice(EInputDeviceType::KeyboardMouse);
    }
    else if (Key.IsGamepadKey())
    {
        SwitchToDevice(EInputDeviceType::Gamepad);
    }
}

void UInputDeviceManager::SwitchToDevice(EInputDeviceType NewDevice)
{
    if (CurrentDevice != NewDevice)
    {
        CurrentDevice = NewDevice;
        OnInputDeviceChanged.Broadcast(NewDevice);
        
        UE_LOG(LogTemp, Log, TEXT("Input device switched to: %s"),
            NewDevice == EInputDeviceType::KeyboardMouse ? TEXT("Keyboard/Mouse") : TEXT("Gamepad"));
    }
}
```

#### 自动切换 IMC

```cpp
// MyPlayerController.h
UCLASS()
class AMyPlayerController : public APlayerController
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    UInputMappingContext* IMC_KeyboardMouse;
    
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    UInputMappingContext* IMC_Gamepad;

protected:
    virtual void BeginPlay() override;

private:
    UFUNCTION()
    void OnInputDeviceChanged(EInputDeviceType NewDevice);
    
    void ApplyInputMappingContext(UInputMappingContext* NewIMC);
};
```

```cpp
// MyPlayerController.cpp
void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();
    
    // 监听设备切换事件
    if (UInputDeviceManager* DeviceManager = GetGameInstance()->GetSubsystem<UInputDeviceManager>())
    {
        DeviceManager->OnInputDeviceChanged.AddDynamic(this, &AMyPlayerController::OnInputDeviceChanged);
        
        // 初始化
        OnInputDeviceChanged(DeviceManager->GetCurrentInputDevice());
    }
}

void AMyPlayerController::OnInputDeviceChanged(EInputDeviceType NewDevice)
{
    UInputMappingContext* NewIMC = nullptr;
    
    switch (NewDevice)
    {
    case EInputDeviceType::KeyboardMouse:
        NewIMC = IMC_KeyboardMouse;
        break;
    case EInputDeviceType::Gamepad:
        NewIMC = IMC_Gamepad;
        break;
    default:
        break;
    }
    
    if (NewIMC)
    {
        ApplyInputMappingContext(NewIMC);
    }
}

void AMyPlayerController::ApplyInputMappingContext(UInputMappingContext* NewIMC)
{
    if (UEnhancedInputLocalPlayerSubsystem* Subsystem = 
        ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(GetLocalPlayer()))
    {
        // 清除旧的映射
        Subsystem->ClearAllMappings();
        
        // 添加新的映射
        Subsystem->AddMappingContext(NewIMC, 0);
        
        UE_LOG(LogTemp, Log, TEXT("Applied Input Mapping Context: %s"), *NewIMC->GetName());
    }
}
```

#### UI 提示更新

```cpp
// HUDWidget.h (UMG Widget)
UCLASS()
class UHUDWidget : public UUserWidget
{
    GENERATED_BODY()

protected:
    virtual void NativeConstruct() override;

    UPROPERTY(meta = (BindWidget))
    UImage* JumpButtonIcon;
    
    UPROPERTY(EditDefaultsOnly, Category = "Icons")
    UTexture2D* Icon_Keyboard_Space;
    
    UPROPERTY(EditDefaultsOnly, Category = "Icons")
    UTexture2D* Icon_Gamepad_A;

private:
    UFUNCTION()
    void OnInputDeviceChanged(EInputDeviceType NewDevice);
};
```

```cpp
// HUDWidget.cpp
void UHUDWidget::NativeConstruct()
{
    Super::NativeConstruct();
    
    // 监听设备切换
    if (UInputDeviceManager* DeviceManager = GetGameInstance()->GetSubsystem<UInputDeviceManager>())
    {
        DeviceManager->OnInputDeviceChanged.AddDynamic(this, &UHUDWidget::OnInputDeviceChanged);
        OnInputDeviceChanged(DeviceManager->GetCurrentInputDevice());
    }
}

void UHUDWidget::OnInputDeviceChanged(EInputDeviceType NewDevice)
{
    if (JumpButtonIcon)
    {
        UTexture2D* NewIcon = (NewDevice == EInputDeviceType::KeyboardMouse) 
            ? Icon_Keyboard_Space 
            : Icon_Gamepad_A;
        
        JumpButtonIcon->SetBrushFromTexture(NewIcon);
    }
}
```

---

## 总结

### Enhanced Input System 的优势

1. **数据驱动**：输入配置完全通过数据资产管理，无需重新编译
2. **上下文管理**：自动处理输入优先级和冲突
3. **强大的处理能力**：内置死区、平滑、缩放等 Modifiers
4. **灵活的触发机制**：支持长按、连招、组合键等复杂输入
5. **可视化调试**：实时查看输入状态和处理流程
6. **现代硬件支持**：PS5 自适应扳机、触觉反馈等

### Lyra 的最佳实践

1. **Tag 驱动架构**：通过 Gameplay Tag 解耦输入和功能
2. **Native vs Ability 分离**：基础输入用 C++，技能输入用 GAS
3. **多 IMC 叠加**：不同游戏模式、设备使用不同的 IMC
4. **数据资产配置**：Input Config 统一管理所有输入映射
5. **动态切换**：运行时根据场景切换输入上下文

### 实战建议

- 为不同平台创建独立的 IMC（键鼠、手柄、移动端）
- 使用自定义 Modifier 实现项目特定的输入处理需求
- 利用 Trigger 实现高级输入模式（连招、蓄力）
- 结合 GAS 构建统一的技能输入系统
- 实现输入设备自动检测和 UI 适配

### 参考资源

- [Epic 官方文档：Enhanced Input](https://docs.unrealengine.com/5.0/en-US/enhanced-input-in-unreal-engine/)
- [Lyra 项目源码](https://www.unrealengine.com/marketplace/en-US/product/lyra)
- [GAS 文档](https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-for-unreal-engine/)

---

**文章完成！**本文共计约 **25,000 字**，涵盖了 Enhanced Input System 的基础概念、Lyra 的输入架构、高级功能和完整的实战案例。希望能帮助你深入理解并应用这套强大的输入系统！🎮
