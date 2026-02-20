# 第22章：相机系统详解

## 22.1 Lyra 相机系统概述

### 22.1.1 相机系统架构简介

Lyra 的相机系统是一个高度模块化、灵活且强大的架构，它突破了传统 UE 相机系统的限制，引入了 **Camera Mode Stack**（相机模式栈）的概念，使得多个相机模式可以同时工作并平滑混合。这种设计让游戏可以轻松实现复杂的相机行为，如从第三人称过渡到瞄准模式、死亡观战、电影镜头等。

Lyra 相机系统的核心组件包括：

1. **ULyraCameraComponent**：相机组件，挂载在角色上，负责管理 Camera Mode Stack
2. **ULyraCameraMode**：相机模式基类，定义了相机的位置、旋转、FOV 等参数
3. **ULyraCameraModeStack**：相机模式栈，负责多个 Camera Mode 的混合与过渡
4. **ALyraPlayerCameraManager**：玩家相机管理器，集成了 UI 相机等高级功能

### 22.1.2 核心设计理念

Lyra 相机系统的设计理念可以概括为以下几点：

#### 1. 栈式混合（Stack-Based Blending）

不同于传统的单一相机模式切换，Lyra 使用栈结构来管理多个相机模式。新的相机模式被推入栈顶，系统会自动在不同模式之间进行加权混合，实现平滑过渡。

```
栈顶 [瞄准模式 - 权重 0.8]
     [第三人称模式 - 权重 0.2]  ← 混合计算
栈底 [基础模式 - 权重 1.0]
```

#### 2. 数据驱动（Data-Driven）

相机参数通过可编辑的属性进行配置，设计师无需修改代码即可调整相机行为：

- 混合时间（Blend Time）
- 混合函数（Linear/EaseIn/EaseOut/EaseInOut）
- FOV、Pitch 限制
- 第三人称偏移曲线

#### 3. 碰撞避障（Penetration Avoidance）

Lyra 实现了先进的相机碰撞避障系统，使用多条射线（Feelers）预测性地避免相机穿透墙体，提供更流畅的游戏体验。

### 22.1.3 相机系统类图

```
ALyraPlayerCameraManager (继承自 APlayerCameraManager)
    └── ULyraUICameraManagerComponent
    └── (管理) ULyraCameraComponent
    
ULyraCameraComponent (继承自 UCameraComponent)
    └── ULyraCameraModeStack
        └── TArray<ULyraCameraMode*> CameraModeStack
        
ULyraCameraMode (抽象基类)
    ├── ULyraCameraMode_ThirdPerson
    ├── 自定义相机模式...
```

### 22.1.4 相机更新流程

```cpp
// 简化的相机更新流程
void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)
{
    // 1. 更新相机模式（通过 Delegate 决定使用哪个模式）
    UpdateCameraModes();
    
    // 2. 评估相机模式栈，混合所有模式
    FLyraCameraModeView CameraModeView;
    CameraModeStack->EvaluateStack(DeltaTime, CameraModeView);
    
    // 3. 同步 PlayerController 的旋转
    if (APlayerController* PC = TargetPawn->GetController<APlayerController>())
    {
        PC->SetControlRotation(CameraModeView.ControlRotation);
    }
    
    // 4. 输出到最终的相机视图
    DesiredView.Location = CameraModeView.Location;
    DesiredView.Rotation = CameraModeView.Rotation;
    DesiredView.FOV = CameraModeView.FieldOfView;
}
```

---

## 22.2 ULyraCameraMode 基类深度解析

### 22.2.1 ULyraCameraMode 核心属性

`ULyraCameraMode` 是所有相机模式的抽象基类，定义了相机模式的核心行为和数据结构。

```cpp
UCLASS(MinimalAPI, Abstract, NotBlueprintable)
class ULyraCameraMode : public UObject
{
    GENERATED_BODY()

protected:
    // 相机类型标签（用于查询当前激活的相机类型）
    UPROPERTY(EditDefaultsOnly, Category = "Blending")
    FGameplayTag CameraTypeTag;

    // 视图输出数据
    FLyraCameraModeView View;

    // 水平视野角度（度数）
    UPROPERTY(EditDefaultsOnly, Category = "View", Meta = (UIMin = "5.0", UIMax = "170"))
    float FieldOfView;

    // 最小俯仰角（度数）
    UPROPERTY(EditDefaultsOnly, Category = "View", Meta = (UIMin = "-89.9", UIMax = "89.9"))
    float ViewPitchMin;

    // 最大俯仰角（度数）
    UPROPERTY(EditDefaultsOnly, Category = "View", Meta = (UIMin = "-89.9", UIMax = "89.9"))
    float ViewPitchMax;

    // 混合时间（秒）
    UPROPERTY(EditDefaultsOnly, Category = "Blending")
    float BlendTime;

    // 混合函数类型
    UPROPERTY(EditDefaultsOnly, Category = "Blending")
    ELyraCameraModeBlendFunction BlendFunction;

    // 混合指数（控制曲线形状）
    UPROPERTY(EditDefaultsOnly, Category = "Blending")
    float BlendExponent;

    // 线性混合 Alpha（0-1）
    float BlendAlpha;

    // 计算后的混合权重（0-1）
    float BlendWeight;
};
```

### 22.2.2 FLyraCameraModeView 数据结构

`FLyraCameraModeView` 是相机视图的数据结构，用于存储和混合相机参数。

```cpp
struct FLyraCameraModeView
{
    FVector Location;         // 相机位置
    FRotator Rotation;        // 相机旋转
    FRotator ControlRotation; // 控制旋转（影响 PlayerController）
    float FieldOfView;        // 视野角度

    FLyraCameraModeView()
        : Location(ForceInit)
        , Rotation(ForceInit)
        , ControlRotation(ForceInit)
        , FieldOfView(LYRA_CAMERA_DEFAULT_FOV)
    {
    }

    // 混合两个视图
    void Blend(const FLyraCameraModeView& Other, float OtherWeight)
    {
        if (OtherWeight <= 0.0f) return;
        if (OtherWeight >= 1.0f)
        {
            *this = Other;
            return;
        }

        // 位置插值
        Location = FMath::Lerp(Location, Other.Location, OtherWeight);

        // 旋转插值（考虑角度归一化）
        const FRotator DeltaRotation = (Other.Rotation - Rotation).GetNormalized();
        Rotation = Rotation + (OtherWeight * DeltaRotation);

        const FRotator DeltaControlRotation = (Other.ControlRotation - ControlRotation).GetNormalized();
        ControlRotation = ControlRotation + (OtherWeight * DeltaControlRotation);

        // FOV 插值
        FieldOfView = FMath::Lerp(FieldOfView, Other.FieldOfView, OtherWeight);
    }
};
```

### 22.2.3 混合函数（Blend Function）

Lyra 支持四种混合函数，用于控制相机模式切换时的过渡曲线：

```cpp
UENUM(BlueprintType)
enum class ELyraCameraModeBlendFunction : uint8
{
    // 线性插值
    Linear,

    // 快速加速，平滑减速（易入）
    EaseIn,

    // 平滑加速，不减速（易出）
    EaseOut,

    // 平滑加速和减速（易入易出）
    EaseInOut,
};
```

混合权重的计算逻辑：

```cpp
void ULyraCameraMode::UpdateBlending(float DeltaTime)
{
    // 更新 BlendAlpha（线性增长）
    if (BlendTime > 0.0f)
    {
        BlendAlpha += (DeltaTime / BlendTime);
        BlendAlpha = FMath::Min(BlendAlpha, 1.0f);
    }
    else
    {
        BlendAlpha = 1.0f;
    }

    const float Exponent = (BlendExponent > 0.0f) ? BlendExponent : 1.0f;

    // 根据混合函数计算最终权重
    switch (BlendFunction)
    {
    case ELyraCameraModeBlendFunction::Linear:
        BlendWeight = BlendAlpha;
        break;

    case ELyraCameraModeBlendFunction::EaseIn:
        BlendWeight = FMath::InterpEaseIn(0.0f, 1.0f, BlendAlpha, Exponent);
        break;

    case ELyraCameraModeBlendFunction::EaseOut:
        BlendWeight = FMath::InterpEaseOut(0.0f, 1.0f, BlendAlpha, Exponent);
        break;

    case ELyraCameraModeBlendFunction::EaseInOut:
        BlendWeight = FMath::InterpEaseInOut(0.0f, 1.0f, BlendAlpha, Exponent);
        break;
    }
}
```

**混合函数效果对比**：

- **Linear**：恒定速度，适合快速切换
- **EaseIn**（Exponent=4.0）：开始快速，结束缓慢，适合突然进入瞄准状态
- **EaseOut**（Exponent=4.0，Lyra 默认）：开始缓慢，结束快速，适合退出瞄准状态
- **EaseInOut**：两端缓慢，中间快速，适合电影镜头

### 22.2.4 Pivot 计算（锚点）

相机模式需要确定相机的锚点（Pivot），即相机围绕哪个点进行旋转和偏移。

```cpp
FVector ULyraCameraMode::GetPivotLocation() const
{
    const AActor* TargetActor = GetTargetActor();
    check(TargetActor);

    if (const APawn* TargetPawn = Cast<APawn>(TargetActor))
    {
        // 对于角色，考虑蹲伏时的高度调整
        if (const ACharacter* TargetCharacter = Cast<ACharacter>(TargetPawn))
        {
            const ACharacter* TargetCharacterCDO = TargetCharacter->GetClass()->GetDefaultObject<ACharacter>();
            const UCapsuleComponent* CapsuleComp = TargetCharacter->GetCapsuleComponent();
            const UCapsuleComponent* CapsuleCompCDO = TargetCharacterCDO->GetCapsuleComponent();

            // 计算蹲伏导致的高度变化
            const float DefaultHalfHeight = CapsuleCompCDO->GetUnscaledCapsuleHalfHeight();
            const float ActualHalfHeight = CapsuleComp->GetUnscaledCapsuleHalfHeight();
            const float HeightAdjustment = (DefaultHalfHeight - ActualHalfHeight) + TargetCharacterCDO->BaseEyeHeight;

            return TargetCharacter->GetActorLocation() + (FVector::UpVector * HeightAdjustment);
        }

        // 使用 Pawn 的视图位置（通常是眼睛位置）
        return TargetPawn->GetPawnViewLocation();
    }

    return TargetActor->GetActorLocation();
}

FRotator ULyraCameraMode::GetPivotRotation() const
{
    const AActor* TargetActor = GetTargetActor();
    check(TargetActor);

    if (const APawn* TargetPawn = Cast<APawn>(TargetActor))
    {
        // Pawn 的视图旋转（受 PlayerController 控制）
        return TargetPawn->GetViewRotation();
    }

    return TargetActor->GetActorRotation();
}
```

### 22.2.5 UpdateView 与 UpdateCameraMode

相机模式每帧更新的核心逻辑：

```cpp
void ULyraCameraMode::UpdateCameraMode(float DeltaTime)
{
    // 1. 更新相机视图（位置、旋转、FOV）
    UpdateView(DeltaTime);
    
    // 2. 更新混合权重
    UpdateBlending(DeltaTime);
}

void ULyraCameraMode::UpdateView(float DeltaTime)
{
    FVector PivotLocation = GetPivotLocation();
    FRotator PivotRotation = GetPivotRotation();

    // 限制俯仰角
    PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

    // 基础视图（默认实现是将相机放在 Pivot 位置）
    View.Location = PivotLocation;
    View.Rotation = PivotRotation;
    View.ControlRotation = View.Rotation;
    View.FieldOfView = FieldOfView;
}
```

---

## 22.3 Camera Mode Stack 混合栈

### 22.3.1 ULyraCameraModeStack 类设计

`ULyraCameraModeStack` 是 Lyra 相机系统的核心，负责管理多个相机模式的栈结构，并计算最终的混合视图。

```cpp
UCLASS()
class ULyraCameraModeStack : public UObject
{
    GENERATED_BODY()

protected:
    // 栈是否激活
    bool bIsActive;

    // 相机模式实例池（避免重复创建）
    UPROPERTY()
    TArray<TObjectPtr<ULyraCameraMode>> CameraModeInstances;

    // 当前激活的相机模式栈（从栈底到栈顶）
    UPROPERTY()
    TArray<TObjectPtr<ULyraCameraMode>> CameraModeStack;

public:
    // 激活/停用栈
    void ActivateStack();
    void DeactivateStack();

    // 推入新的相机模式
    void PushCameraMode(TSubclassOf<ULyraCameraMode> CameraModeClass);

    // 评估栈并输出混合视图
    bool EvaluateStack(float DeltaTime, FLyraCameraModeView& OutCameraModeView);
};
```

### 22.3.2 PushCameraMode：相机模式推入逻辑

当需要激活新的相机模式时，调用 `PushCameraMode`，它会将新模式推入栈顶，并计算初始混合权重。

```cpp
void ULyraCameraModeStack::PushCameraMode(TSubclassOf<ULyraCameraMode> CameraModeClass)
{
    if (!CameraModeClass)
    {
        return;
    }

    // 获取或创建相机模式实例
    ULyraCameraMode* CameraMode = GetCameraModeInstance(CameraModeClass);
    check(CameraMode);

    int32 StackSize = CameraModeStack.Num();

    // 如果已经是栈顶，则不做任何操作
    if ((StackSize > 0) && (CameraModeStack[0] == CameraMode))
    {
        return;
    }

    // 检查模式是否已经在栈中，如果在，则移除并计算其贡献值
    int32 ExistingStackIndex = INDEX_NONE;
    float ExistingStackContribution = 1.0f;

    for (int32 StackIndex = 0; StackIndex < StackSize; ++StackIndex)
    {
        if (CameraModeStack[StackIndex] == CameraMode)
        {
            ExistingStackIndex = StackIndex;
            ExistingStackContribution *= CameraMode->GetBlendWeight();
            break;
        }
        else
        {
            // 计算该模式在栈中的贡献度
            ExistingStackContribution *= (1.0f - CameraModeStack[StackIndex]->GetBlendWeight());
        }
    }

    // 如果在栈中，则移除
    if (ExistingStackIndex != INDEX_NONE)
    {
        CameraModeStack.RemoveAt(ExistingStackIndex);
        StackSize--;
    }
    else
    {
        ExistingStackContribution = 0.0f;
    }

    // 决定初始混合权重
    const bool bShouldBlend = ((CameraMode->GetBlendTime() > 0.0f) && (StackSize > 0));
    const float BlendWeight = (bShouldBlend ? ExistingStackContribution : 1.0f);

    CameraMode->SetBlendWeight(BlendWeight);

    // 插入到栈顶（索引0）
    CameraModeStack.Insert(CameraMode, 0);

    // 确保栈底总是权重 100%
    CameraModeStack.Last()->SetBlendWeight(1.0f);

    // 通知相机模式被激活
    if (ExistingStackIndex == INDEX_NONE)
    {
        CameraMode->OnActivation();
    }
}
```

**关键点**：

1. 栈的组织是 **从栈顶（索引0）到栈底**
2. 栈底的相机模式权重始终为 1.0
3. 新模式的初始权重基于其在栈中的贡献值（如果已存在）或 0（如果是新加入）

### 22.3.3 EvaluateStack：栈评估与混合

每帧调用 `EvaluateStack` 来更新栈中的所有相机模式，并计算最终混合视图。

```cpp
bool ULyraCameraModeStack::EvaluateStack(float DeltaTime, FLyraCameraModeView& OutCameraModeView)
{
    if (!bIsActive)
    {
        return false;
    }

    // 1. 更新栈中的所有模式
    UpdateStack(DeltaTime);
    
    // 2. 混合栈中的所有模式
    BlendStack(OutCameraModeView);

    return true;
}
```

### 22.3.4 UpdateStack：更新栈并移除完全混合的模式

```cpp
void ULyraCameraModeStack::UpdateStack(float DeltaTime)
{
    const int32 StackSize = CameraModeStack.Num();
    if (StackSize <= 0)
    {
        return;
    }

    int32 RemoveCount = 0;
    int32 RemoveIndex = INDEX_NONE;

    // 从栈顶到栈底更新每个相机模式
    for (int32 StackIndex = 0; StackIndex < StackSize; ++StackIndex)
    {
        ULyraCameraMode* CameraMode = CameraModeStack[StackIndex];
        check(CameraMode);

        // 更新相机模式（包括混合权重）
        CameraMode->UpdateCameraMode(DeltaTime);

        // 如果该模式的混合权重达到 100%，则它下面的所有模式都可以移除
        if (CameraMode->GetBlendWeight() >= 1.0f)
        {
            RemoveIndex = (StackIndex + 1);
            RemoveCount = (StackSize - RemoveIndex);
            break;
        }
    }

    // 移除被完全覆盖的模式
    if (RemoveCount > 0)
    {
        // 通知模式被停用
        for (int32 StackIndex = RemoveIndex; StackIndex < StackSize; ++StackIndex)
        {
            ULyraCameraMode* CameraMode = CameraModeStack[StackIndex];
            check(CameraMode);
            CameraMode->OnDeactivation();
        }

        CameraModeStack.RemoveAt(RemoveIndex, RemoveCount);
    }
}
```

**优化机制**：一旦某个相机模式的权重达到 1.0，其下方的所有模式都不再影响最终结果，可以安全移除，减少不必要的计算。

### 22.3.5 BlendStack：混合栈中的所有视图

```cpp
void ULyraCameraModeStack::BlendStack(FLyraCameraModeView& OutCameraModeView) const
{
    const int32 StackSize = CameraModeStack.Num();
    if (StackSize <= 0)
    {
        return;
    }

    // 从栈底开始，向上混合
    const ULyraCameraMode* CameraMode = CameraModeStack[StackSize - 1];
    check(CameraMode);

    // 使用栈底的视图作为基础
    OutCameraModeView = CameraMode->GetCameraModeView();

    // 依次向上混合
    for (int32 StackIndex = (StackSize - 2); StackIndex >= 0; --StackIndex)
    {
        CameraMode = CameraModeStack[StackIndex];
        check(CameraMode);

        // 根据权重混合视图
        OutCameraModeView.Blend(CameraMode->GetCameraModeView(), CameraMode->GetBlendWeight());
    }
}
```

**混合示例**：

假设栈结构为：
```
栈顶 [瞄准模式 - 权重 0.6]
     [第三人称模式 - 权重 1.0]
栈底
```

混合计算：
```cpp
OutCameraModeView = 第三人称模式视图;
OutCameraModeView.Blend(瞄准模式视图, 0.6);

// 最终位置 = Lerp(第三人称位置, 瞄准位置, 0.6)
// 最终 FOV = Lerp(第三人称 FOV, 瞄准 FOV, 0.6)
```

---

## 22.4 第三人称相机实现

### 22.4.1 ULyraCameraMode_ThirdPerson 类概述

`ULyraCameraMode_ThirdPerson` 是 Lyra 提供的第三人称相机实现，包含了偏移曲线、碰撞避障、蹲伏动画等高级特性。

```cpp
UCLASS(Abstract, Blueprintable)
class ULyraCameraMode_ThirdPerson : public ULyraCameraMode
{
    GENERATED_BODY()

protected:
    // 基于俯仰角的偏移曲线
    UPROPERTY(EditDefaultsOnly, Category = "Third Person")
    TObjectPtr<const UCurveVector> TargetOffsetCurve;

    // 使用运行时曲线（用于实时编辑）
    UPROPERTY(EditDefaultsOnly, Category = "Third Person")
    bool bUseRuntimeFloatCurves;

    UPROPERTY(EditDefaultsOnly, Category = "Third Person", Meta = (EditCondition = "bUseRuntimeFloatCurves"))
    FRuntimeFloatCurve TargetOffsetX;

    UPROPERTY(EditDefaultsOnly, Category = "Third Person", Meta = (EditCondition = "bUseRuntimeFloatCurves"))
    FRuntimeFloatCurve TargetOffsetY;

    UPROPERTY(EditDefaultsOnly, Category = "Third Person", Meta = (EditCondition = "bUseRuntimeFloatCurves"))
    FRuntimeFloatCurve TargetOffsetZ;

    // 蹲伏偏移混合速度
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Third Person")
    float CrouchOffsetBlendMultiplier = 5.0f;

    // 碰撞避障配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Collision")
    bool bPreventPenetration = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Collision")
    bool bDoPredictiveAvoidance = true;

    UPROPERTY(EditDefaultsOnly, Category = "Collision")
    TArray<FLyraPenetrationAvoidanceFeeler> PenetrationAvoidanceFeelers;
};
```

### 22.4.2 偏移曲线（Target Offset Curve）

第三人称相机的位置是基于角色的 Pivot 点加上一个偏移量计算的，这个偏移量根据相机的俯仰角（Pitch）从曲线中取值。

```cpp
void ULyraCameraMode_ThirdPerson::UpdateView(float DeltaTime)
{
    // 1. 更新蹲伏偏移
    UpdateForTarget(DeltaTime);
    UpdateCrouchOffset(DeltaTime);

    // 2. 获取 Pivot 点
    FVector PivotLocation = GetPivotLocation() + CurrentCrouchOffset;
    FRotator PivotRotation = GetPivotRotation();

    PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

    // 3. 设置初始视图
    View.Location = PivotLocation;
    View.Rotation = PivotRotation;
    View.ControlRotation = View.Rotation;
    View.FieldOfView = FieldOfView;

    // 4. 应用基于 Pitch 的偏移
    if (!bUseRuntimeFloatCurves)
    {
        if (TargetOffsetCurve)
        {
            // 从曲线资产中读取偏移
            const FVector TargetOffset = TargetOffsetCurve->GetVectorValue(PivotRotation.Pitch);
            View.Location = PivotLocation + PivotRotation.RotateVector(TargetOffset);
        }
    }
    else
    {
        // 从运行时曲线中读取偏移
        FVector TargetOffset(0.0f);
        TargetOffset.X = TargetOffsetX.GetRichCurveConst()->Eval(PivotRotation.Pitch);
        TargetOffset.Y = TargetOffsetY.GetRichCurveConst()->Eval(PivotRotation.Pitch);
        TargetOffset.Z = TargetOffsetZ.GetRichCurveConst()->Eval(PivotRotation.Pitch);

        View.Location = PivotLocation + PivotRotation.RotateVector(TargetOffset);
    }

    // 5. 应用碰撞避障
    UpdatePreventPenetration(DeltaTime);
}
```

**偏移曲线示例配置**：

在编辑器中创建一个 `CurveVector` 资产，配置如下：

| Pitch（输入） | X（后方距离） | Y（右侧距离） | Z（上方距离） |
|-------------|------------|------------|------------|
| -30°        | 200        | 50         | 80         |
| 0°          | 250        | 50         | 60         |
| 30°         | 300        | 50         | 40         |
| 60°         | 350        | 50         | 20         |

这样配置会让相机在抬头时拉得更远，俯视时更近。

### 22.4.3 蹲伏偏移（Crouch Offset）

当角色蹲下时，相机需要平滑地降低高度。Lyra 使用插值来实现这一效果。

```cpp
void ULyraCameraMode_ThirdPerson::UpdateForTarget(float DeltaTime)
{
    if (const ACharacter* TargetCharacter = Cast<ACharacter>(GetTargetActor()))
    {
        if (TargetCharacter->IsCrouched())
        {
            const ACharacter* TargetCharacterCDO = TargetCharacter->GetClass()->GetDefaultObject<ACharacter>();
            
            // 计算蹲伏导致的高度差
            const float CrouchedHeightAdjustment = TargetCharacterCDO->CrouchedEyeHeight - TargetCharacterCDO->BaseEyeHeight;

            SetTargetCrouchOffset(FVector(0.f, 0.f, CrouchedHeightAdjustment));
            return;
        }
    }

    SetTargetCrouchOffset(FVector::ZeroVector);
}

void ULyraCameraMode_ThirdPerson::SetTargetCrouchOffset(FVector NewTargetOffset)
{
    CrouchOffsetBlendPct = 0.0f;
    InitialCrouchOffset = CurrentCrouchOffset;
    TargetCrouchOffset = NewTargetOffset;
}

void ULyraCameraMode_ThirdPerson::UpdateCrouchOffset(float DeltaTime)
{
    if (CrouchOffsetBlendPct < 1.0f)
    {
        CrouchOffsetBlendPct = FMath::Min(CrouchOffsetBlendPct + DeltaTime * CrouchOffsetBlendMultiplier, 1.0f);
        
        // 使用 EaseInOut 插值实现平滑过渡
        CurrentCrouchOffset = FMath::InterpEaseInOut(InitialCrouchOffset, TargetCrouchOffset, CrouchOffsetBlendPct, 1.0f);
    }
    else
    {
        CurrentCrouchOffset = TargetCrouchOffset;
        CrouchOffsetBlendPct = 1.0f;
    }
}
```

### 22.4.4 碰撞避障（Penetration Avoidance）

Lyra 的碰撞避障系统是其相机系统的亮点之一，使用多条射线（Feelers）来预测和避免相机穿透墙体。

#### Feeler 数据结构

```cpp
USTRUCT()
struct FLyraPenetrationAvoidanceFeeler
{
    GENERATED_BODY()

    // 相对于主射线的角度偏移
    UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)
    FRotator AdjustmentRot;

    // 碰到世界物体时的权重
    UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)
    float WorldWeight;

    // 碰到 Pawn 时的权重
    UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)
    float PawnWeight;

    // 射线球体半径
    UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)
    float Extent;

    // 射线检测间隔（帧数）
    UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)
    int32 TraceInterval;
};
```

#### 默认 Feeler 配置

```cpp
ULyraCameraMode_ThirdPerson::ULyraCameraMode_ThirdPerson()
{
    // 主射线：直线向后，权重 100%
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, +00.0f, 0.0f), 1.00f, 1.00f, 14.f, 0));
    
    // 左右偏移射线：每 3 帧检测一次
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, +16.0f, 0.0f), 0.75f, 0.75f, 00.f, 3));
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, -16.0f, 0.0f), 0.75f, 0.75f, 00.f, 3));
    
    // 更大角度的左右射线：每 5 帧检测一次
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, +32.0f, 0.0f), 0.50f, 0.50f, 00.f, 5));
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, -32.0f, 0.0f), 0.50f, 0.50f, 00.f, 5));
    
    // 上下偏移射线：每 4 帧检测一次
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+20.0f, +00.0f, 0.0f), 1.00f, 1.00f, 00.f, 4));
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(-20.0f, +00.0f, 0.0f), 0.50f, 0.50f, 00.f, 4));
}
```

#### PreventCameraPenetration 实现（简化版）

```cpp
void ULyraCameraMode_ThirdPerson::PreventCameraPenetration(
    AActor const& ViewTarget, 
    FVector const& SafeLoc, 
    FVector& CameraLoc, 
    float const& DeltaTime, 
    float& DistBlockedPct, 
    bool bSingleRayOnly)
{
    float HardBlockedPct = DistBlockedPct;
    float SoftBlockedPct = DistBlockedPct;

    FVector BaseRay = CameraLoc - SafeLoc;
    FRotationMatrix BaseRayMatrix(BaseRay.Rotation());
    FVector BaseRayLocalUp, BaseRayLocalFwd, BaseRayLocalRight;
    BaseRayMatrix.GetScaledAxes(BaseRayLocalFwd, BaseRayLocalRight, BaseRayLocalUp);

    float DistBlockedPctThisFrame = 1.f;

    int32 const NumRaysToShoot = bSingleRayOnly ? 1 : PenetrationAvoidanceFeelers.Num();
    
    FCollisionQueryParams SphereParams(SCENE_QUERY_STAT(CameraPen), false, nullptr);
    SphereParams.AddIgnoredActor(&ViewTarget);

    FCollisionShape SphereShape = FCollisionShape::MakeSphere(0.f);
    UWorld* World = GetWorld();

    // 对每条 Feeler 进行射线检测
    for (int32 RayIdx = 0; RayIdx < NumRaysToShoot; ++RayIdx)
    {
        FLyraPenetrationAvoidanceFeeler& Feeler = PenetrationAvoidanceFeelers[RayIdx];
        
        if (Feeler.FramesUntilNextTrace <= 0)
        {
            // 计算射线目标点
            FVector RotatedRay = BaseRay.RotateAngleAxis(Feeler.AdjustmentRot.Yaw, BaseRayLocalUp);
            RotatedRay = RotatedRay.RotateAngleAxis(Feeler.AdjustmentRot.Pitch, BaseRayLocalRight);
            FVector RayTarget = SafeLoc + RotatedRay;

            SphereShape.Sphere.Radius = Feeler.Extent;

            // 球体扫描
            FHitResult Hit;
            const bool bHit = World->SweepSingleByChannel(Hit, SafeLoc, RayTarget, FQuat::Identity, ECC_Camera, SphereShape, SphereParams);

            Feeler.FramesUntilNextTrace = Feeler.TraceInterval;

            if (bHit && Hit.GetActor())
            {
                // 计算阻挡百分比
                float Weight = Cast<APawn>(Hit.GetActor()) ? Feeler.PawnWeight : Feeler.WorldWeight;
                float NewBlockPct = Hit.Time;
                NewBlockPct += (1.f - NewBlockPct) * (1.f - Weight);

                NewBlockPct = ((Hit.Location - SafeLoc).Size() - CollisionPushOutDistance) / (RayTarget - SafeLoc).Size();
                DistBlockedPctThisFrame = FMath::Min(NewBlockPct, DistBlockedPctThisFrame);

                Feeler.FramesUntilNextTrace = 0; // 下一帧继续检测
            }

            if (RayIdx == 0)
            {
                HardBlockedPct = DistBlockedPctThisFrame; // 主射线：立即应用
            }
            else
            {
                SoftBlockedPct = DistBlockedPctThisFrame; // 副射线：平滑应用
            }
        }
        else
        {
            --Feeler.FramesUntilNextTrace;
        }
    }

    // 根据混合时间平滑过渡阻挡百分比
    if (DistBlockedPct < DistBlockedPctThisFrame)
    {
        // 向外混合（远离墙体）
        if (PenetrationBlendOutTime > DeltaTime)
        {
            DistBlockedPct = DistBlockedPct + DeltaTime / PenetrationBlendOutTime * (DistBlockedPctThisFrame - DistBlockedPct);
        }
        else
        {
            DistBlockedPct = DistBlockedPctThisFrame;
        }
    }
    else
    {
        // 向内混合（靠近墙体）
        if (DistBlockedPct > HardBlockedPct)
        {
            DistBlockedPct = HardBlockedPct; // 立即应用硬阻挡
        }
        else if (DistBlockedPct > SoftBlockedPct)
        {
            if (PenetrationBlendInTime > DeltaTime)
            {
                DistBlockedPct = DistBlockedPct - DeltaTime / PenetrationBlendInTime * (DistBlockedPct - SoftBlockedPct);
            }
            else
            {
                DistBlockedPct = SoftBlockedPct;
            }
        }
    }

    DistBlockedPct = FMath::Clamp<float>(DistBlockedPct, 0.f, 1.f);
    
    // 应用最终的相机位置
    if (DistBlockedPct < (1.f - ZERO_ANIMWEIGHT_THRESH))
    {
        CameraLoc = SafeLoc + (CameraLoc - SafeLoc) * DistBlockedPct;
    }
}
```

**碰撞避障工作原理**：

1. 从安全位置（角色位置）向期望的相机位置发射多条射线
2. 主射线（0号）直接检测，副射线以一定间隔检测（性能优化）
3. 如果射线碰到障碍物，计算阻挡百分比（0-1）
4. 根据阻挡百分比调整相机位置，使其位于安全位置和期望位置之间
5. 使用平滑混合避免相机"弹跳"

---

## 22.5 第一人称与瞄准相机

### 22.5.1 第一人称相机模式

第一人称相机模式非常简单，只需将相机放在角色的眼睛位置即可。

```cpp
// LyraCameraMode_FirstPerson.h
#pragma once

#include "Camera/LyraCameraMode.h"
#include "LyraCameraMode_FirstPerson.generated.h"

UCLASS(Blueprintable)
class ULyraCameraMode_FirstPerson : public ULyraCameraMode
{
    GENERATED_BODY()

public:
    ULyraCameraMode_FirstPerson()
    {
        FieldOfView = 90.0f;
        ViewPitchMin = -89.0f;
        ViewPitchMax = 89.0f;
        BlendTime = 0.3f;
        BlendFunction = ELyraCameraModeBlendFunction::EaseOut;
    }

protected:
    virtual void UpdateView(float DeltaTime) override
    {
        FVector PivotLocation = GetPivotLocation();
        FRotator PivotRotation = GetPivotRotation();

        PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

        // 第一人称：相机直接放在 Pivot 位置（眼睛位置）
        View.Location = PivotLocation;
        View.Rotation = PivotRotation;
        View.ControlRotation = View.Rotation;
        View.FieldOfView = FieldOfView;
    }
};
```

### 22.5.2 瞄准相机模式（ADS - Aim Down Sights）

瞄准模式通常会缩小 FOV，并可能调整相机偏移以对齐瞄准镜。

```cpp
// LyraCameraMode_ADS.h
#pragma once

#include "Camera/LyraCameraMode.h"
#include "LyraCameraMode_ADS.generated.h"

UCLASS(Blueprintable)
class ULyraCameraMode_ADS : public ULyraCameraMode
{
    GENERATED_BODY()

public:
    ULyraCameraMode_ADS()
    {
        // 瞄准时缩小 FOV，增加精确度
        FieldOfView = 60.0f;
        ViewPitchMin = -89.0f;
        ViewPitchMax = 89.0f;
        
        // 快速混合进入瞄准状态
        BlendTime = 0.25f;
        BlendFunction = ELyraCameraModeBlendFunction::EaseIn;
        BlendExponent = 3.0f;
        
        // 设置相机类型标签，用于查询是否在瞄准状态
        CameraTypeTag = FGameplayTag::RequestGameplayTag(TEXT("Camera.Mode.ADS"));
    }

protected:
    // 瞄准时的相机偏移（相对于角色）
    UPROPERTY(EditDefaultsOnly, Category = "ADS")
    FVector AimOffset = FVector(50.0f, 40.0f, 10.0f);

    // 瞄准时的视野角度
    UPROPERTY(EditDefaultsOnly, Category = "ADS")
    float AimFieldOfView = 60.0f;

    virtual void UpdateView(float DeltaTime) override
    {
        FVector PivotLocation = GetPivotLocation();
        FRotator PivotRotation = GetPivotRotation();

        PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

        // 应用瞄准偏移
        View.Location = PivotLocation + PivotRotation.RotateVector(AimOffset);
        View.Rotation = PivotRotation;
        View.ControlRotation = View.Rotation;
        View.FieldOfView = AimFieldOfView;
    }
};
```

### 22.5.3 切换相机模式的实现

在角色类中，通过 `ULyraCameraComponent` 的 `DetermineCameraModeDelegate` 来动态决定使用哪个相机模式。

```cpp
// LyraCharacter.h
UCLASS()
class ALyraCharacter : public AModularCharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    TSubclassOf<ULyraCameraMode> DefaultCameraMode;

    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    TSubclassOf<ULyraCameraMode> AimingCameraMode;

    bool bIsAiming = false;

public:
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

    // 绑定相机模式决策
    TSubclassOf<ULyraCameraMode> DetermineCameraMode();

    // 输入处理
    void OnStartAiming();
    void OnStopAiming();
};

// LyraCharacter.cpp
void ALyraCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    // 绑定相机模式决策委托
    if (ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(this))
    {
        CameraComponent->DetermineCameraModeDelegate.BindUObject(this, &ALyraCharacter::DetermineCameraMode);
    }
}

TSubclassOf<ULyraCameraMode> ALyraCharacter::DetermineCameraMode()
{
    // 根据状态返回不同的相机模式
    if (bIsAiming && AimingCameraMode)
    {
        return AimingCameraMode;
    }

    return DefaultCameraMode;
}

void ALyraCharacter::OnStartAiming()
{
    bIsAiming = true;
}

void ALyraCharacter::OnStopAiming()
{
    bIsAiming = false;
}
```

### 22.5.4 蓝图配置

在角色蓝图中设置相机模式类：

```
DefaultCameraMode = BP_CameraMode_ThirdPerson
AimingCameraMode = BP_CameraMode_ADS
```

当按下瞄准键时，调用 `OnStartAiming()`，系统会自动将 `BP_CameraMode_ADS` 推入相机模式栈，并平滑过渡。

---

## 22.6 相机输入与控制

### 22.6.1 Enhanced Input 集成

Lyra 使用 Enhanced Input 系统来处理相机输入，提供了更灵活的输入映射和修饰器功能。

#### Input Action 配置

```cpp
// IA_Look.uasset
Input Action: IA_Look
Value Type: Axis2D (Vector2D)
```

#### Input Mapping Context 配置

```
Input Mapping Context: IMC_Default

Mappings:
  IA_Look
    - Mouse XY 2D-Axis
      Modifiers:
        - Negate Y (反转鼠标 Y 轴)
        - Scalar (灵敏度调整)
          X: 1.0
          Y: 1.0
    
    - Gamepad Right Stick 2D-Axis
      Modifiers:
        - Response Curve Exponential (指数曲线)
          Exponent: 1.5
        - Scalar
          X: 2.0 (手柄灵敏度更高)
          Y: 2.0
```

### 22.6.2 相机输入处理代码

```cpp
// LyraCharacter.h
UCLASS()
class ALyraCharacter : public AModularCharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    TObjectPtr<UInputAction> LookAction;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    TObjectPtr<UInputMappingContext> DefaultMappingContext;

    // 相机灵敏度配置
    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    float LookSensitivityMouse = 1.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    float LookSensitivityGamepad = 2.0f;

public:
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

    void Input_Look(const FInputActionValue& InputActionValue);
};

// LyraCharacter.cpp
void ALyraCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(PlayerInputComponent);

    // 绑定 Look 输入
    EnhancedInputComponent->BindAction(LookAction, ETriggerEvent::Triggered, this, &ALyraCharacter::Input_Look);

    // 添加 Input Mapping Context
    if (APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
        {
            Subsystem->AddMappingContext(DefaultMappingContext, 0);
        }
    }
}

void ALyraCharacter::Input_Look(const FInputActionValue& InputActionValue)
{
    FVector2D LookAxisVector = InputActionValue.Get<FVector2D>();

    if (Controller != nullptr)
    {
        // 应用灵敏度（Enhanced Input 的 Modifier 已经处理了大部分）
        AddControllerYawInput(LookAxisVector.X);
        AddControllerPitchInput(LookAxisVector.Y);
    }
}
```

### 22.6.3 平台差异处理

不同平台的输入特性需要针对性配置：

```cpp
// LyraInputConfig.h
USTRUCT(BlueprintType)
struct FLyraInputPlatformSettings
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly)
    float MouseSensitivity = 1.0f;

    UPROPERTY(EditDefaultsOnly)
    float GamepadSensitivity = 2.0f;

    UPROPERTY(EditDefaultsOnly)
    bool bInvertMouseY = false;

    UPROPERTY(EditDefaultsOnly)
    bool bInvertGamepadY = false;

    UPROPERTY(EditDefaultsOnly)
    float GamepadDeadZone = 0.15f;
};

// LyraInputSettings.h
UCLASS(Config=Game, DefaultConfig)
class ULyraInputSettings : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    UPROPERTY(Config, EditAnywhere, Category = "PC")
    FLyraInputPlatformSettings PCSettings;

    UPROPERTY(Config, EditAnywhere, Category = "Console")
    FLyraInputPlatformSettings ConsoleSettings;

    UPROPERTY(Config, EditAnywhere, Category = "Mobile")
    FLyraInputPlatformSettings MobileSettings;

    const FLyraInputPlatformSettings& GetCurrentPlatformSettings() const
    {
#if PLATFORM_DESKTOP
        return PCSettings;
#elif PLATFORM_XBOXONE || PLATFORM_PS4 || PLATFORM_SWITCH
        return ConsoleSettings;
#else
        return MobileSettings;
#endif
    }
};
```

### 22.6.4 动态调整相机灵敏度

```cpp
// 玩家可以在游戏设置中调整灵敏度
void ULyraSettingsWidget::OnSensitivityChanged(float NewSensitivity)
{
    if (APlayerController* PC = GetOwningPlayer())
    {
        if (ALyraCharacter* Character = Cast<ALyraCharacter>(PC->GetPawn()))
        {
            Character->SetLookSensitivity(NewSensitivity);
        }
    }
}

// LyraCharacter.cpp
void ALyraCharacter::SetLookSensitivity(float NewSensitivity)
{
    LookSensitivityMouse = NewSensitivity;
    LookSensitivityGamepad = NewSensitivity * 2.0f; // 手柄通常需要更高的灵敏度
}
```

---

## 22.7 相机特效（震动与后处理）

### 22.7.1 相机震动（Camera Shake）

Lyra 支持 UE5 的相机震动系统，可以通过蓝图或 C++ 触发。

#### 创建相机震动类

```cpp
// LyraCameraShake_Explosion.h
#pragma once

#include "Camera/CameraShakeBase.h"
#include "LyraCameraShake_Explosion.generated.h"

UCLASS()
class ULyraCameraShake_Explosion : public ULegacyCameraShake
{
    GENERATED_BODY()

public:
    ULyraCameraShake_Explosion()
    {
        // 震动持续时间
        OscillationDuration = 0.5f;
        OscillationBlendInTime = 0.05f;
        OscillationBlendOutTime = 0.2f;

        // 旋转震动
        RotOscillation.Pitch.Amplitude = 3.0f;
        RotOscillation.Pitch.Frequency = 10.0f;

        RotOscillation.Yaw.Amplitude = 3.0f;
        RotOscillation.Yaw.Frequency = 10.0f;

        RotOscillation.Roll.Amplitude = 2.0f;
        RotOscillation.Roll.Frequency = 12.0f;

        // 位置震动
        LocOscillation.X.Amplitude = 5.0f;
        LocOscillation.X.Frequency = 10.0f;

        LocOscillation.Y.Amplitude = 5.0f;
        LocOscillation.Y.Frequency = 10.0f;

        LocOscillation.Z.Amplitude = 3.0f;
        LocOscillation.Z.Frequency = 15.0f;

        // FOV 震动
        FOVOscillation.Amplitude = 5.0f;
        FOVOscillation.Frequency = 5.0f;
    }
};
```

#### 触发相机震动

```cpp
// 在爆炸时触发相机震动
void ALyraWeapon::OnExplosion(FVector ExplosionLocation)
{
    if (APlayerController* PC = GetWorld()->GetFirstPlayerController())
    {
        // 计算距离衰减
        float Distance = FVector::Dist(ExplosionLocation, PC->GetPawn()->GetActorLocation());
        float MaxDistance = 2000.0f;
        float Scale = FMath::Clamp(1.0f - (Distance / MaxDistance), 0.0f, 1.0f);

        // 播放相机震动
        PC->ClientStartCameraShake(ULyraCameraShake_Explosion::StaticClass(), Scale);
    }
}
```

### 22.7.2 后处理效果（Post Process）

相机组件可以直接配置后处理效果，并在不同相机模式下切换。

```cpp
// LyraCameraMode_LowHealth.h
UCLASS(Blueprintable)
class ULyraCameraMode_LowHealth : public ULyraCameraMode
{
    GENERATED_BODY()

public:
    ULyraCameraMode_LowHealth()
    {
        FieldOfView = 85.0f;
        BlendTime = 1.0f;
        BlendFunction = ELyraCameraModeBlendFunction::Linear;
    }

protected:
    virtual void OnActivation() override
    {
        Super::OnActivation();

        // 激活低血量后处理效果
        if (ULyraCameraComponent* CameraComponent = GetLyraCameraComponent())
        {
            // 启用后处理
            CameraComponent->PostProcessBlendWeight = 1.0f;

            // 降低饱和度
            CameraComponent->PostProcessSettings.bOverride_ColorSaturation = true;
            CameraComponent->PostProcessSettings.ColorSaturation = FVector4(0.5f, 0.5f, 0.5f, 1.0f);

            // 添加暗角效果
            CameraComponent->PostProcessSettings.bOverride_VignetteIntensity = true;
            CameraComponent->PostProcessSettings.VignetteIntensity = 0.8f;

            // 色差效果（模拟失血）
            CameraComponent->PostProcessSettings.bOverride_SceneFringeIntensity = true;
            CameraComponent->PostProcessSettings.SceneFringeIntensity = 2.0f;
        }
    }

    virtual void OnDeactivation() override
    {
        Super::OnDeactivation();

        // 恢复正常后处理
        if (ULyraCameraComponent* CameraComponent = GetLyraCameraComponent())
        {
            CameraComponent->PostProcessBlendWeight = 0.0f;
        }
    }
};
```

### 22.7.3 动态 FOV 调整

在冲刺或受伤时动态调整 FOV 以增强视觉反馈。

```cpp
// LyraCharacter.h
UCLASS()
class ALyraCharacter : public AModularCharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Camera")
    float SprintFOVOffset = 10.0f;

    bool bIsSprinting = false;

public:
    void OnStartSprint();
    void OnStopSprint();
};

// LyraCharacter.cpp
void ALyraCharacter::OnStartSprint()
{
    bIsSprinting = true;

    if (ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(this))
    {
        // 增加 FOV，营造速度感
        CameraComponent->AddFieldOfViewOffset(SprintFOVOffset);
    }
}

void ALyraCharacter::OnStopSprint()
{
    bIsSprinting = false;

    if (ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(this))
    {
        // 恢复 FOV
        CameraComponent->AddFieldOfViewOffset(-SprintFOVOffset);
    }
}
```

**注意**：`AddFieldOfViewOffset` 只影响一帧，所以需要持续调用或改为设置目标 FOV 并插值。

### 22.7.4 案例5：相机震动效果配置

实现一个完整的相机震动系统，支持不同类型的震动效果。

#### 震动基类

```cpp
// LyraCameraShakeBase.h
#pragma once

#include "Camera/CameraShakeBase.h"
#include "LyraCameraShakeBase.generated.h"

/**
 * Lyra 相机震动基类
 * 支持更灵活的配置和运行时参数调整
 */
UCLASS(Abstract, Blueprintable)
class ULyraCameraShakeBase : public UCameraShakeBase
{
    GENERATED_BODY()

public:
    // 根据距离计算震动强度
    UFUNCTION(BlueprintCallable, Category = "Camera Shake")
    static float CalculateScaleFromDistance(FVector EpicenterLocation, FVector PlayerLocation, float InnerRadius, float OuterRadius);

    // 带衰减的震动播放
    UFUNCTION(BlueprintCallable, Category = "Camera Shake", Meta = (WorldContext = "WorldContextObject"))
    static void PlayCameraShakeWithFalloff(
        const UObject* WorldContextObject,
        TSubclassOf<UCameraShakeBase> ShakeClass,
        FVector EpicenterLocation,
        float InnerRadius,
        float OuterRadius,
        float Falloff = 1.0f);
};

// LyraCameraShakeBase.cpp
#include "LyraCameraShakeBase.h"
#include "Kismet/GameplayStatics.h"
#include "GameFramework/PlayerController.h"

float ULyraCameraShakeBase::CalculateScaleFromDistance(FVector EpicenterLocation, FVector PlayerLocation, float InnerRadius, float OuterRadius)
{
    const float Distance = FVector::Dist(EpicenterLocation, PlayerLocation);

    if (Distance <= InnerRadius)
    {
        return 1.0f; // 满强度
    }

    if (Distance >= OuterRadius)
    {
        return 0.0f; // 无震动
    }

    // 线性衰减
    return 1.0f - ((Distance - InnerRadius) / (OuterRadius - InnerRadius));
}

void ULyraCameraShakeBase::PlayCameraShakeWithFalloff(
    const UObject* WorldContextObject,
    TSubclassOf<UCameraShakeBase> ShakeClass,
    FVector EpicenterLocation,
    float InnerRadius,
    float OuterRadius,
    float Falloff)
{
    if (!WorldContextObject || !ShakeClass)
    {
        return;
    }

    UWorld* World = WorldContextObject->GetWorld();
    if (!World)
    {
        return;
    }

    // 对所有玩家应用震动
    for (FConstPlayerControllerIterator It = World->GetPlayerControllerIterator(); It; ++It)
    {
        APlayerController* PC = It->Get();
        if (!PC || !PC->GetPawn())
        {
            continue;
        }

        FVector PlayerLocation = PC->GetPawn()->GetActorLocation();
        float Scale = CalculateScaleFromDistance(EpicenterLocation, PlayerLocation, InnerRadius, OuterRadius);

        if (Scale > 0.0f)
        {
            Scale = FMath::Pow(Scale, Falloff); // 应用衰减曲线
            PC->ClientStartCameraShake(ShakeClass, Scale);
        }
    }
}
```

#### 预定义震动效果

```cpp
// LyraCameraShake_Hit.h - 受击震动
UCLASS()
class ULyraCameraShake_Hit : public UMatineeCameraShake
{
    GENERATED_BODY()

public:
    ULyraCameraShake_Hit()
    {
        OscillationDuration = 0.25f;
        OscillationBlendInTime = 0.05f;
        OscillationBlendOutTime = 0.1f;

        RotOscillation.Pitch.Amplitude = 2.0f;
        RotOscillation.Pitch.Frequency = 20.0f;

        RotOscillation.Yaw.Amplitude = 2.0f;
        RotOscillation.Yaw.Frequency = 20.0f;

        RotOscillation.Roll.Amplitude = 1.5f;
        RotOscillation.Roll.Frequency = 25.0f;
    }
};

// LyraCameraShake_Explosion.h - 爆炸震动
UCLASS()
class ULyraCameraShake_Explosion : public UMatineeCameraShake
{
    GENERATED_BODY()

public:
    ULyraCameraShake_Explosion()
    {
        OscillationDuration = 0.8f;
        OscillationBlendInTime = 0.1f;
        OscillationBlendOutTime = 0.3f;

        // 强烈的旋转震动
        RotOscillation.Pitch.Amplitude = 5.0f;
        RotOscillation.Pitch.Frequency = 10.0f;

        RotOscillation.Yaw.Amplitude = 5.0f;
        RotOscillation.Yaw.Frequency = 10.0f;

        RotOscillation.Roll.Amplitude = 3.0f;
        RotOscillation.Roll.Frequency = 12.0f;

        // 位置震动
        LocOscillation.X.Amplitude = 8.0f;
        LocOscillation.X.Frequency = 10.0f;

        LocOscillation.Y.Amplitude = 8.0f;
        LocOscillation.Y.Frequency = 10.0f;

        LocOscillation.Z.Amplitude = 5.0f;
        LocOscillation.Z.Frequency = 15.0f;

        // FOV 震动
        FOVOscillation.Amplitude = 10.0f;
        FOVOscillation.Frequency = 8.0f;
    }
};

// LyraCameraShake_Footstep.h - 脚步震动
UCLASS()
class ULyraCameraShake_Footstep : public UMatineeCameraShake
{
    GENERATED_BODY()

public:
    ULyraCameraShake_Footstep()
    {
        OscillationDuration = 0.15f;
        OscillationBlendInTime = 0.02f;
        OscillationBlendOutTime = 0.08f;

        // 轻微的上下震动
        LocOscillation.Z.Amplitude = 1.0f;
        LocOscillation.Z.Frequency = 15.0f;

        // 轻微的旋转
        RotOscillation.Roll.Amplitude = 0.5f;
        RotOscillation.Roll.Frequency = 15.0f;
    }
};

// LyraCameraShake_Vehicle.h - 载具震动（持续）
UCLASS()
class ULyraCameraShake_Vehicle : public UMatineeCameraShake
{
    GENERATED_BODY()

public:
    ULyraCameraShake_Vehicle()
    {
        OscillationDuration = -1.0f; // 持续震动（需要手动停止）
        OscillationBlendInTime = 0.5f;
        OscillationBlendOutTime = 0.5f;

        // 模拟载具颠簸
        LocOscillation.X.Amplitude = 2.0f;
        LocOscillation.X.Frequency = 5.0f;

        LocOscillation.Y.Amplitude = 2.0f;
        LocOscillation.Y.Frequency = 4.0f;

        LocOscillation.Z.Amplitude = 3.0f;
        LocOscillation.Z.Frequency = 6.0f;

        RotOscillation.Roll.Amplitude = 1.0f;
        RotOscillation.Roll.Frequency = 5.0f;
    }
};
```

#### 使用示例

```cpp
// 在武器开火时触发震动
void ALyraWeapon::OnFire()
{
    if (APlayerController* PC = GetOwningPlayerController())
    {
        PC->ClientStartCameraShake(ULyraCameraShake_Hit::StaticClass(), 0.5f);
    }
}

// 在爆炸时触发距离衰减的震动
void ALyraProjectile::OnExplode()
{
    FVector ExplosionLocation = GetActorLocation();

    ULyraCameraShakeBase::PlayCameraShakeWithFalloff(
        this,
        ULyraCameraShake_Explosion::StaticClass(),
        ExplosionLocation,
        500.0f,  // 内半径（满强度）
        2000.0f, // 外半径（无震动）
        1.5f     // 衰减指数（越大衰减越快）
    );
}

// 在角色移动时持续触发脚步震动
void ALyraCharacter::PlayFootstepEffect()
{
    if (APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        float ShakeScale = GetCharacterMovement()->Velocity.Size() / GetCharacterMovement()->MaxWalkSpeed;
        PC->ClientStartCameraShake(ULyraCameraShake_Footstep::StaticClass(), ShakeScale);
    }
}

// 载具震动的启动和停止
void ALyraVehicle::OnStartDriving()
{
    if (APlayerController* PC = GetController<APlayerController>())
    {
        VehicleShakeInstance = PC->ClientStartCameraShake(ULyraCameraShake_Vehicle::StaticClass(), 1.0f);
    }
}

void ALyraVehicle::OnStopDriving()
{
    if (APlayerController* PC = GetController<APlayerController>())
    {
        if (VehicleShakeInstance)
        {
            PC->ClientStopCameraShake(ULyraCameraShake_Vehicle::StaticClass());
            VehicleShakeInstance = nullptr;
        }
    }
}
```

#### Data Asset 配置

创建 Data Asset 来集中管理震动配置：

```cpp
// LyraCameraShakeDataAsset.h
USTRUCT(BlueprintType)
struct FLyraCameraShakeConfig
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TSubclassOf<UCameraShakeBase> ShakeClass;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    float DefaultScale = 1.0f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    bool bSingleInstance = false; // 是否只允许一个实例
};

UCLASS(BlueprintType)
class ULyraCameraShakeDataAsset : public UDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Combat")
    FLyraCameraShakeConfig WeaponFireShake;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Combat")
    FLyraCameraShakeConfig WeaponReloadShake;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Combat")
    FLyraCameraShakeConfig HitShake;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Combat")
    FLyraCameraShakeConfig ExplosionShake;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Movement")
    FLyraCameraShakeConfig FootstepShake;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Movement")
    FLyraCameraShakeConfig LandingShake;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Vehicle")
    FLyraCameraShakeConfig VehicleIdleShake;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Vehicle")
    FLyraCameraShakeConfig VehicleDrivingShake;

    // 辅助函数
    UFUNCTION(BlueprintCallable, Category = "Camera Shake")
    void PlayShake(APlayerController* PC, const FLyraCameraShakeConfig& ShakeConfig, float ScaleMultiplier = 1.0f);
};

// LyraCameraShakeDataAsset.cpp
void ULyraCameraShakeDataAsset::PlayShake(APlayerController* PC, const FLyraCameraShakeConfig& ShakeConfig, float ScaleMultiplier)
{
    if (!PC || !ShakeConfig.ShakeClass)
    {
        return;
    }

    float FinalScale = ShakeConfig.DefaultScale * ScaleMultiplier;

    if (ShakeConfig.bSingleInstance)
    {
        PC->ClientStopCameraShake(ShakeConfig.ShakeClass);
    }

    PC->ClientStartCameraShake(ShakeConfig.ShakeClass, FinalScale);
}
```

---

## 22.8 实战：自定义相机模式

### 22.8.1 案例1：自定义第三人称相机模式

创建一个支持动态距离调整和肩部切换的第三人称相机。

```cpp
// CustomCameraMode_ThirdPerson.h
#pragma once

#include "Camera/LyraCameraMode_ThirdPerson.h"
#include "CustomCameraMode_ThirdPerson.generated.h"

UCLASS(Blueprintable)
class UCustomCameraMode_ThirdPerson : public ULyraCameraMode_ThirdPerson
{
    GENERATED_BODY()

public:
    UCustomCameraMode_ThirdPerson()
    {
        FieldOfView = 90.0f;
        BlendTime = 0.5f;
        BlendFunction = ELyraCameraModeBlendFunction::EaseOut;

        TargetArmLength = 300.0f;
        CurrentArmLength = 300.0f;
        MinArmLength = 150.0f;
        MaxArmLength = 600.0f;
    }

protected:
    // 摄像机臂长（距离）
    UPROPERTY(EditDefaultsOnly, Category = "Third Person")
    float TargetArmLength;

    UPROPERTY(EditDefaultsOnly, Category = "Third Person")
    float MinArmLength;

    UPROPERTY(EditDefaultsOnly, Category = "Third Person")
    float MaxArmLength;

    float CurrentArmLength;

    // 肩部切换（左/右）
    UPROPERTY(EditDefaultsOnly, Category = "Third Person")
    float ShoulderOffsetRight = 50.0f;

    bool bShoulderLeft = false;

    virtual void UpdateView(float DeltaTime) override
    {
        // 调用父类逻辑（处理蹲伏、碰撞避障等）
        Super::UpdateView(DeltaTime);

        // 平滑插值臂长
        CurrentArmLength = FMath::FInterpTo(CurrentArmLength, TargetArmLength, DeltaTime, 5.0f);

        // 计算最终偏移
        FVector PivotLocation = GetPivotLocation();
        FRotator PivotRotation = GetPivotRotation();

        // 基础偏移（向后）
        FVector BackwardOffset = -PivotRotation.Vector() * CurrentArmLength;

        // 肩部偏移（向右或向左）
        float ShoulderOffset = bShoulderLeft ? -ShoulderOffsetRight : ShoulderOffsetRight;
        FVector RightOffset = FRotationMatrix(PivotRotation).GetScaledAxis(EAxis::Y) * ShoulderOffset;

        // 组合偏移
        View.Location = PivotLocation + BackwardOffset + RightOffset;
    }

public:
    // 动态调整距离
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void ZoomIn(float Amount)
    {
        TargetArmLength = FMath::Clamp(TargetArmLength - Amount, MinArmLength, MaxArmLength);
    }

    UFUNCTION(BlueprintCallable, Category = "Camera")
    void ZoomOut(float Amount)
    {
        TargetArmLength = FMath::Clamp(TargetArmLength + Amount, MinArmLength, MaxArmLength);
    }

    // 切换肩部
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void ToggleShoulder()
    {
        bShoulderLeft = !bShoulderLeft;
    }
};
```

#### 在角色中使用

```cpp
// 在角色蓝图或 C++ 中绑定输入
void ALyraCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    // 绑定鼠标滚轮缩放
    PlayerInputComponent->BindAxis("MouseWheel", this, &ALyraCharacter::OnMouseWheel);

    // 绑定肩部切换
    PlayerInputComponent->BindAction("ToggleShoulder", IE_Pressed, this, &ALyraCharacter::OnToggleShoulder);
}

void ALyraCharacter::OnMouseWheel(float Value)
{
    if (ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(this))
    {
        // 获取当前相机模式栈的顶层模式
        float Weight;
        FGameplayTag Tag;
        CameraComponent->GetBlendInfo(Weight, Tag);

        // 假设自定义相机模式是当前激活的
        if (UCustomCameraMode_ThirdPerson* CustomMode = Cast<UCustomCameraMode_ThirdPerson>(/* 获取当前模式 */))
        {
            if (Value > 0)
            {
                CustomMode->ZoomIn(20.0f);
            }
            else if (Value < 0)
            {
                CustomMode->ZoomOut(20.0f);
            }
        }
    }
}

void ALyraCharacter::OnToggleShoulder()
{
    // 类似地切换肩部
}
```

### 22.8.2 案例2：实现瞄准镜缩放相机

创建一个支持多级缩放的瞄准镜相机（2x、4x、8x）。

```cpp
// CameraMode_Scope.h
#pragma once

#include "Camera/LyraCameraMode.h"
#include "CameraMode_Scope.generated.h"

UENUM(BlueprintType)
enum class EScopeZoomLevel : uint8
{
    None    UMETA(DisplayName = "1x"),
    Zoom2x  UMETA(DisplayName = "2x"),
    Zoom4x  UMETA(DisplayName = "4x"),
    Zoom8x  UMETA(DisplayName = "8x")
};

UCLASS(Blueprintable)
class UCameraMode_Scope : public ULyraCameraMode
{
    GENERATED_BODY()

public:
    UCameraMode_Scope()
    {
        FieldOfView = 90.0f;
        BlendTime = 0.2f;
        BlendFunction = ELyraCameraModeBlendFunction::EaseIn;
        BlendExponent = 2.0f;

        CameraTypeTag = FGameplayTag::RequestGameplayTag(TEXT("Camera.Mode.Scope"));
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Scope")
    float BaseFOV = 90.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Scope")
    float Zoom2xFOV = 45.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Scope")
    float Zoom4xFOV = 22.5f;

    UPROPERTY(EditDefaultsOnly, Category = "Scope")
    float Zoom8xFOV = 11.25f;

    UPROPERTY(EditDefaultsOnly, Category = "Scope")
    FVector ScopeOffset = FVector(80.0f, 5.0f, 0.0f);

    EScopeZoomLevel CurrentZoomLevel = EScopeZoomLevel::None;
    float TargetFOV = 90.0f;
    float CurrentFOV = 90.0f;

    virtual void UpdateView(float DeltaTime) override
    {
        FVector PivotLocation = GetPivotLocation();
        FRotator PivotRotation = GetPivotRotation();

        PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

        // 应用瞄准镜偏移
        View.Location = PivotLocation + PivotRotation.RotateVector(ScopeOffset);
        View.Rotation = PivotRotation;
        View.ControlRotation = View.Rotation;

        // 平滑插值 FOV
        CurrentFOV = FMath::FInterpTo(CurrentFOV, TargetFOV, DeltaTime, 10.0f);
        View.FieldOfView = CurrentFOV;
    }

public:
    UFUNCTION(BlueprintCallable, Category = "Scope")
    void SetZoomLevel(EScopeZoomLevel NewZoomLevel)
    {
        CurrentZoomLevel = NewZoomLevel;

        switch (CurrentZoomLevel)
        {
        case EScopeZoomLevel::None:
            TargetFOV = BaseFOV;
            break;
        case EScopeZoomLevel::Zoom2x:
            TargetFOV = Zoom2xFOV;
            break;
        case EScopeZoomLevel::Zoom4x:
            TargetFOV = Zoom4xFOV;
            break;
        case EScopeZoomLevel::Zoom8x:
            TargetFOV = Zoom8xFOV;
            break;
        }
    }

    UFUNCTION(BlueprintCallable, Category = "Scope")
    void CycleZoomLevel()
    {
        int32 CurrentLevel = static_cast<int32>(CurrentZoomLevel);
        CurrentLevel = (CurrentLevel + 1) % 4;
        SetZoomLevel(static_cast<EScopeZoomLevel>(CurrentLevel));
    }
};
```

#### UI 显示缩放等级

```cpp
// ScopeWidget.h
UCLASS()
class UScopeWidget : public UUserWidget
{
    GENERATED_BODY()

protected:
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> ZoomLevelText;

public:
    void UpdateZoomLevel(EScopeZoomLevel ZoomLevel)
    {
        FString ZoomText;
        switch (ZoomLevel)
        {
        case EScopeZoomLevel::None:
            ZoomText = TEXT("1x");
            break;
        case EScopeZoomLevel::Zoom2x:
            ZoomText = TEXT("2x");
            break;
        case EScopeZoomLevel::Zoom4x:
            ZoomText = TEXT("4x");
            break;
        case EScopeZoomLevel::Zoom8x:
            ZoomText = TEXT("8x");
            break;
        }

        ZoomLevelText->SetText(FText::FromString(ZoomText));
    }
};
```

### 22.8.3 案例3：死亡观战相机

当玩家死亡时，相机平滑过渡到观战模式，可以自由旋转观察战场。

```cpp
// CameraMode_Spectator.h
#pragma once

#include "Camera/LyraCameraMode.h"
#include "CameraMode_Spectator.generated.h"

UCLASS(Blueprintable)
class UCameraMode_Spectator : public ULyraCameraMode
{
    GENERATED_BODY()

public:
    UCameraMode_Spectator()
    {
        FieldOfView = 90.0f;
        BlendTime = 2.0f; // 缓慢过渡
        BlendFunction = ELyraCameraModeBlendFunction::EaseOut;
        BlendExponent = 3.0f;

        SpectatorHeight = 300.0f;
        SpectatorDistance = 500.0f;
        RotationSpeed = 10.0f;
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Spectator")
    float SpectatorHeight;

    UPROPERTY(EditDefaultsOnly, Category = "Spectator")
    float SpectatorDistance;

    UPROPERTY(EditDefaultsOnly, Category = "Spectator")
    float RotationSpeed;

    FVector DeathLocation;
    FRotator CurrentRotation;

    virtual void OnActivation() override
    {
        Super::OnActivation();

        // 记录死亡位置
        if (AActor* TargetActor = GetTargetActor())
        {
            DeathLocation = TargetActor->GetActorLocation();
            CurrentRotation = TargetActor->GetActorRotation();
        }
    }

    virtual void UpdateView(float DeltaTime) override
    {
        // 相机绕死亡位置旋转
        CurrentRotation.Yaw += RotationSpeed * DeltaTime;

        // 计算相机位置（螺旋上升）
        FVector Offset = CurrentRotation.Vector() * -SpectatorDistance;
        Offset.Z = SpectatorHeight;

        View.Location = DeathLocation + Offset;
        View.Rotation = (DeathLocation - View.Location).Rotation();
        View.ControlRotation = View.Rotation;
        View.FieldOfView = FieldOfView;
    }
};
```

#### 在死亡时激活观战相机

```cpp
// LyraCharacter.cpp
void ALyraCharacter::OnDeath()
{
    // 切换到观战相机模式
    SpectatorCameraMode = UCameraMode_Spectator::StaticClass();

    // 触发相机切换（通过更新 DetermineCameraModeDelegate）
    bIsDead = true;
}

TSubclassOf<ULyraCameraMode> ALyraCharacter::DetermineCameraMode()
{
    if (bIsDead && SpectatorCameraMode)
    {
        return SpectatorCameraMode;
    }

    // ... 其他逻辑
}
```

---

## 22.9 实战：电影镜头系统

### 22.9.1 电影相机模式（Cinematic Camera）

创建一个支持固定位置、路径跟随、Look At 目标的电影相机系统。

```cpp
// CameraMode_Cinematic.h
#pragma once

#include "Camera/LyraCameraMode.h"
#include "Components/SplineComponent.h"
#include "CameraMode_Cinematic.generated.h"

UCLASS(Blueprintable)
class UCameraMode_Cinematic : public ULyraCameraMode
{
    GENERATED_BODY()

public:
    UCameraMode_Cinematic()
    {
        FieldOfView = 90.0f;
        BlendTime = 1.5f;
        BlendFunction = ELyraCameraModeBlendFunction::EaseInOut;
        BlendExponent = 2.0f;
    }

protected:
    // 电影镜头路径（Spline）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cinematic")
    TObjectPtr<USplineComponent> CameraPath;

    // Look At 目标
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cinematic")
    TObjectPtr<AActor> LookAtTarget;

    // 沿路径移动速度
    UPROPERTY(EditDefaultsOnly, Category = "Cinematic")
    float PathSpeed = 50.0f;

    // 固定位置模式
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cinematic")
    bool bUseFixedLocation = false;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cinematic", Meta = (EditCondition = "bUseFixedLocation"))
    FVector FixedCameraLocation;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cinematic", Meta = (EditCondition = "bUseFixedLocation"))
    FRotator FixedCameraRotation;

    float CurrentPathDistance = 0.0f;

    virtual void OnActivation() override
    {
        Super::OnActivation();
        CurrentPathDistance = 0.0f;
    }

    virtual void UpdateView(float DeltaTime) override
    {
        if (bUseFixedLocation)
        {
            // 固定位置模式
            View.Location = FixedCameraLocation;
            
            if (LookAtTarget)
            {
                View.Rotation = (LookAtTarget->GetActorLocation() - View.Location).Rotation();
            }
            else
            {
                View.Rotation = FixedCameraRotation;
            }
        }
        else if (CameraPath)
        {
            // 路径跟随模式
            CurrentPathDistance += PathSpeed * DeltaTime;
            float SplineLength = CameraPath->GetSplineLength();

            // 循环或停止
            if (CurrentPathDistance >= SplineLength)
            {
                CurrentPathDistance = SplineLength; // 停止在终点
                // CurrentPathDistance = 0.0f; // 或循环
            }

            // 获取路径上的位置和旋转
            View.Location = CameraPath->GetLocationAtDistanceAlongSpline(CurrentPathDistance, ESplineCoordinateSpace::World);

            if (LookAtTarget)
            {
                View.Rotation = (LookAtTarget->GetActorLocation() - View.Location).Rotation();
            }
            else
            {
                View.Rotation = CameraPath->GetRotationAtDistanceAlongSpline(CurrentPathDistance, ESplineCoordinateSpace::World);
            }
        }
        else
        {
            // 默认：跟随目标
            View.Location = GetPivotLocation();
            View.Rotation = GetPivotRotation();
        }

        View.ControlRotation = View.Rotation;
        View.FieldOfView = FieldOfView;
    }
};
```

### 22.9.2 电影序列管理器

使用 Sequencer 或自定义管理器来触发电影镜头。

```cpp
// CinematicManager.h
UCLASS()
class ACinematicManager : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Cinematic")
    TArray<TSubclassOf<UCameraMode_Cinematic>> CinematicShots;

    UPROPERTY(EditAnywhere, Category = "Cinematic")
    TArray<float> ShotDurations;

    int32 CurrentShotIndex = 0;
    float CurrentShotTime = 0.0f;

    void StartCinematic(APlayerController* PC);
    void StopCinematic(APlayerController* PC);

    virtual void Tick(float DeltaTime) override;
};

// CinematicManager.cpp
void ACinematicManager::StartCinematic(APlayerController* PC)
{
    if (!PC) return;

    CurrentShotIndex = 0;
    CurrentShotTime = 0.0f;

    // 隐藏 HUD
    if (AHUD* HUD = PC->GetHUD())
    {
        HUD->SetActorHiddenInGame(true);
    }

    // 激活第一个镜头
    if (CinematicShots.Num() > 0)
    {
        if (APawn* Pawn = PC->GetPawn())
        {
            if (ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(Pawn))
            {
                CameraComponent->DetermineCameraModeDelegate.BindLambda([this]() -> TSubclassOf<ULyraCameraMode>
                {
                    return CinematicShots[CurrentShotIndex];
                });
            }
        }
    }
}

void ACinematicManager::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (CurrentShotIndex >= CinematicShots.Num())
    {
        return; // 电影结束
    }

    CurrentShotTime += DeltaTime;

    // 检查是否需要切换到下一个镜头
    if (CurrentShotTime >= ShotDurations[CurrentShotIndex])
    {
        CurrentShotIndex++;
        CurrentShotTime = 0.0f;

        if (CurrentShotIndex < CinematicShots.Num())
        {
            // 通知相机切换镜头
            // ...
        }
        else
        {
            // 电影结束
            StopCinematic(GetWorld()->GetFirstPlayerController());
        }
    }
}

void ACinematicManager::StopCinematic(APlayerController* PC)
{
    if (!PC) return;

    // 显示 HUD
    if (AHUD* HUD = PC->GetHUD())
    {
        HUD->SetActorHiddenInGame(false);
    }

    // 恢复正常相机
    // ...
}
```

### 22.9.3 案例：过场动画触发器

当玩家进入触发区域时，播放电影镜头。

```cpp
// CinematicTrigger.h
UCLASS()
class ACinematicTrigger : public AActor
{
    GENERATED_BODY()

public:
    ACinematicTrigger()
    {
        TriggerBox = CreateDefaultSubobject<UBoxComponent>(TEXT("TriggerBox"));
        RootComponent = TriggerBox;

        TriggerBox->OnComponentBeginOverlap.AddDynamic(this, &ACinematicTrigger::OnOverlapBegin);
    }

protected:
    UPROPERTY(VisibleAnywhere)
    TObjectPtr<UBoxComponent> TriggerBox;

    UPROPERTY(EditAnywhere, Category = "Cinematic")
    TObjectPtr<ACinematicManager> CinematicManager;

    UPROPERTY(EditAnywhere, Category = "Cinematic")
    bool bTriggerOnce = true;

    bool bHasTriggered = false;

    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
    {
        if (bTriggerOnce && bHasTriggered)
        {
            return;
        }

        if (APlayerController* PC = Cast<APlayerController>(OtherActor->GetInstigatorController()))
        {
            if (CinematicManager)
            {
                CinematicManager->StartCinematic(PC);
                bHasTriggered = true;
            }
        }
    }
};
```

---

## 22.9.4 UI 相机系统

Lyra 还提供了 `ULyraUICameraManagerComponent`，用于在 UI 需要显示时临时接管相机控制。

```cpp
// LyraUICameraManagerComponent.h
UCLASS(Transient, Within=LyraPlayerCameraManager)
class ULyraUICameraManagerComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    // 获取 UI 相机组件
    static ULyraUICameraManagerComponent* GetComponent(APlayerController* PC);

    // 设置 UI 视图目标
    void SetViewTarget(AActor* InViewTarget, FViewTargetTransitionParams TransitionParams = FViewTargetTransitionParams());

    // 是否需要更新视图目标
    bool NeedsToUpdateViewTarget() const;

    // 更新视图目标
    void UpdateViewTarget(struct FTViewTarget& OutVT, float DeltaTime);

protected:
    UPROPERTY(Transient)
    TObjectPtr<AActor> ViewTarget;
    
    UPROPERTY(Transient)
    bool bUpdatingViewTarget;
};
```

#### 使用场景

UI 相机系统主要用于以下场景：

1. **角色选择界面**：展示角色模型
2. **武器检视界面**：360° 旋转查看武器
3. **载具展示**：展示载具外观
4. **装备预览**：预览装备效果

#### 实现角色选择相机

```cpp
// CharacterSelectionCamera.h
UCLASS()
class ACharacterSelectionCamera : public AActor
{
    GENERATED_BODY()

public:
    ACharacterSelectionCamera()
    {
        CameraComponent = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
        RootComponent = CameraComponent;

        ArmLength = 300.0f;
        RotationSpeed = 30.0f;
    }

protected:
    UPROPERTY(VisibleAnywhere)
    TObjectPtr<UCameraComponent> CameraComponent;

    UPROPERTY(EditAnywhere, Category = "Camera")
    float ArmLength;

    UPROPERTY(EditAnywhere, Category = "Camera")
    float RotationSpeed;

    UPROPERTY()
    TObjectPtr<AActor> TargetCharacter;

    float CurrentYaw = 0.0f;

public:
    void SetTargetCharacter(AActor* Character)
    {
        TargetCharacter = Character;
        UpdateCameraPosition();
    }

    void RotateLeft(float DeltaTime)
    {
        CurrentYaw -= RotationSpeed * DeltaTime;
        UpdateCameraPosition();
    }

    void RotateRight(float DeltaTime)
    {
        CurrentYaw += RotationSpeed * DeltaTime;
        UpdateCameraPosition();
    }

protected:
    void UpdateCameraPosition()
    {
        if (!TargetCharacter)
        {
            return;
        }

        FVector CharacterLocation = TargetCharacter->GetActorLocation();
        FRotator CameraRotation = FRotator(0.0f, CurrentYaw, 0.0f);
        FVector CameraOffset = CameraRotation.RotateVector(FVector(-ArmLength, 0.0f, 100.0f));

        SetActorLocation(CharacterLocation + CameraOffset);
        SetActorRotation((CharacterLocation - GetActorLocation()).Rotation());
    }

    virtual void Tick(float DeltaTime) override
    {
        Super::Tick(DeltaTime);
        UpdateCameraPosition();
    }
};

// 在 UI 中使用
void UCharacterSelectionWidget::OnShowWidget()
{
    APlayerController* PC = GetOwningPlayer();
    if (!PC) return;

    // 创建选择相机
    if (!SelectionCamera)
    {
        SelectionCamera = GetWorld()->SpawnActor<ACharacterSelectionCamera>();
        SelectionCamera->SetTargetCharacter(CurrentPreviewCharacter);
    }

    // 设置 UI 相机
    if (ULyraUICameraManagerComponent* UICamera = ULyraUICameraManagerComponent::GetComponent(PC))
    {
        UICamera->SetViewTarget(SelectionCamera);
    }
}

void UCharacterSelectionWidget::OnHideWidget()
{
    APlayerController* PC = GetOwningPlayer();
    if (!PC) return;

    // 恢复正常相机
    if (ULyraUICameraManagerComponent* UICamera = ULyraUICameraManagerComponent::GetComponent(PC))
    {
        UICamera->SetViewTarget(nullptr);
    }

    // 销毁选择相机
    if (SelectionCamera)
    {
        SelectionCamera->Destroy();
        SelectionCamera = nullptr;
    }
}
```

---

## 22.10 网络同步与性能优化

### 22.10.1 相机模式的网络同步策略

在多人游戏中，相机是**客户端本地**的，不需要同步到服务器或其他客户端。但某些影响 Gameplay 的状态（如瞄准状态）需要同步。

```cpp
// LyraCharacter.h
UCLASS()
class ALyraCharacter : public AModularCharacter
{
    GENERATED_BODY()

protected:
    // 瞄准状态（需要复制到其他客户端，用于动画同步）
    UPROPERTY(ReplicatedUsing = OnRep_IsAiming)
    bool bIsAiming;

    UFUNCTION()
    void OnRep_IsAiming();

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 服务器 RPC：通知服务器瞄准状态
    UFUNCTION(Server, Reliable)
    void ServerSetAiming(bool bNewAiming);
};

// LyraCharacter.cpp
void ALyraCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ALyraCharacter, bIsAiming);
}

void ALyraCharacter::OnRep_IsAiming()
{
    // 客户端接收到瞄准状态更新后的处理
    // 注意：相机模式不需要在这里切换（相机是本地的）
    // 但动画需要响应
    if (UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance())
    {
        // 通知动画状态机
    }
}

void ALyraCharacter::ServerSetAiming_Implementation(bool bNewAiming)
{
    bIsAiming = bNewAiming;
}

void ALyraCharacter::OnStartAiming()
{
    // 本地客户端：立即切换相机（无延迟）
    bIsAiming = true;

    // 通知服务器（用于同步动画等）
    if (!HasAuthority())
    {
        ServerSetAiming(true);
    }
}
```

**关键点**：
- 相机模式切换是本地的，不通过网络同步
- 只同步影响 Gameplay 的状态（如瞄准）
- 使用客户端预测确保响应性

### 22.10.2 客户端预测

相机输入应该立即响应，不等待服务器确认。

```cpp
void ALyraCharacter::Input_Look(const FInputActionValue& InputActionValue)
{
    FVector2D LookAxisVector = InputActionValue.Get<FVector2D>();

    if (Controller != nullptr)
    {
        // 客户端直接应用旋转（无需等待服务器）
        AddControllerYawInput(LookAxisVector.X);
        AddControllerPitchInput(LookAxisVector.Y);
    }
}
```

UE 的 `Controller Rotation` 默认是本地的，不会自动复制到服务器，这正好符合相机的需求。

### 22.10.3 Tick 优化

相机更新是性能敏感的操作，需要优化 Tick 频率和计算开销。

#### 优化1：使用 Component Tick

```cpp
ULyraCameraComponent::ULyraCameraComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    PrimaryComponentTick.bCanEverTick = true;
    PrimaryComponentTick.TickGroup = TG_PostUpdateWork; // 在物理更新后执行
    PrimaryComponentTick.bStartWithTickEnabled = true;
}
```

#### 优化2：Feeler 射线间隔

Lyra 已经实现了 `TraceInterval`，避免每帧检测所有射线：

```cpp
// 主射线每帧检测
PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, +00.0f, 0.0f), 1.00f, 1.00f, 14.f, 0));

// 副射线每 3 帧检测一次
PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, +16.0f, 0.0f), 0.75f, 0.75f, 00.f, 3));
```

#### 优化3：LOD 相机更新

根据玩家距离调整相机更新频率（适用于观战模式）。

```cpp
void ULyraCameraComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    // 如果是本地玩家，始终全帧率更新
    if (IsLocallyControlled())
    {
        return;
    }

    // 观战其他玩家时，根据距离降低更新频率
    APlayerController* LocalPC = GetWorld()->GetFirstPlayerController();
    if (LocalPC && LocalPC->GetPawn())
    {
        float Distance = FVector::Dist(LocalPC->GetPawn()->GetActorLocation(), GetOwner()->GetActorLocation());

        if (Distance > 5000.0f)
        {
            // 距离很远，降低更新频率
            SetComponentTickInterval(0.1f); // 每 0.1 秒更新一次
        }
        else
        {
            SetComponentTickInterval(0.0f); // 每帧更新
        }
    }
}
```

#### 优化4：避免冗余计算

缓存常用的计算结果：

```cpp
void ULyraCameraMode::UpdateView(float DeltaTime)
{
    // 缓存 Pivot 位置和旋转（避免多次调用虚函数）
    FVector CachedPivotLocation = GetPivotLocation();
    FRotator CachedPivotRotation = GetPivotRotation();

    // 只在必要时重新计算
    // ...
}
```

### 22.10.4 Async Trace 优化（高级）

对于复杂场景，可以使用异步射线检测来优化相机碰撞避障的性能。

```cpp
// AsyncCameraMode_ThirdPerson.h
UCLASS()
class UAsyncCameraMode_ThirdPerson : public ULyraCameraMode_ThirdPerson
{
    GENERATED_BODY()

protected:
    // 异步 Trace 句柄
    FTraceHandle AsyncTraceHandle;
    
    // 上一帧的阻挡百分比
    float LastFrameBlockedPct = 1.0f;

    virtual void UpdatePreventPenetration(float DeltaTime) override
    {
        if (!bPreventPenetration)
        {
            return;
        }

        // 检查上一帧的异步 Trace 结果
        if (AsyncTraceHandle._Data.FrameNumber != 0)
        {
            FTraceDatum TraceDatum;
            if (GetWorld()->QueryTraceData(AsyncTraceHandle, TraceDatum))
            {
                ProcessAsyncTraceResults(TraceDatum);
                AsyncTraceHandle._Data.FrameNumber = 0; // 重置句柄
            }
        }

        // 启动新的异步 Trace
        StartAsyncPenetrationTrace();

        // 使用上一帧的结果调整相机位置
        ApplyPenetrationAdjustment(LastFrameBlockedPct, DeltaTime);
    }

    void StartAsyncPenetrationTrace()
    {
        AActor* TargetActor = GetTargetActor();
        if (!TargetActor)
        {
            return;
        }

        FVector SafeLoc = TargetActor->GetActorLocation();
        FVector CameraLoc = View.Location;

        // 设置异步 Trace 参数
        FCollisionQueryParams QueryParams(SCENE_QUERY_STAT(AsyncCameraPen), false, nullptr);
        QueryParams.AddIgnoredActor(TargetActor);

        FCollisionShape SphereShape = FCollisionShape::MakeSphere(14.0f);

        // 启动异步 Trace
        AsyncTraceHandle = GetWorld()->AsyncSweepByChannel(
            EAsyncTraceType::Single,
            SafeLoc,
            CameraLoc,
            FQuat::Identity,
            ECC_Camera,
            SphereShape,
            QueryParams
        );
    }

    void ProcessAsyncTraceResults(const FTraceDatum& TraceDatum)
    {
        if (TraceDatum.OutHits.Num() > 0)
        {
            const FHitResult& Hit = TraceDatum.OutHits[0];
            
            // 计算阻挡百分比
            FVector SafeLoc = GetTargetActor()->GetActorLocation();
            FVector CameraLoc = View.Location;
            float TotalDistance = FVector::Dist(SafeLoc, CameraLoc);
            float HitDistance = FVector::Dist(SafeLoc, Hit.Location);
            
            LastFrameBlockedPct = HitDistance / TotalDistance;
        }
        else
        {
            LastFrameBlockedPct = 1.0f; // 无阻挡
        }
    }

    void ApplyPenetrationAdjustment(float BlockedPct, float DeltaTime)
    {
        FVector SafeLoc = GetTargetActor()->GetActorLocation();
        FVector IdealCameraLoc = View.Location;

        View.Location = FMath::Lerp(SafeLoc, IdealCameraLoc, BlockedPct);
    }
};
```

### 22.10.5 Camera Mode 实例池

避免频繁创建和销毁 Camera Mode 实例。

```cpp
// LyraCameraModeStack.cpp (优化版本)
ULyraCameraMode* ULyraCameraModeStack::GetCameraModeInstance(TSubclassOf<ULyraCameraMode> CameraModeClass)
{
    check(CameraModeClass);

    // 首先在实例池中查找
    for (ULyraCameraMode* CameraMode : CameraModeInstances)
    {
        if ((CameraMode != nullptr) && (CameraMode->GetClass() == CameraModeClass))
        {
            return CameraMode;
        }
    }

    // 未找到，创建新实例
    ULyraCameraMode* NewCameraMode = NewObject<ULyraCameraMode>(GetOuter(), CameraModeClass, NAME_None, RF_NoFlags);
    check(NewCameraMode);

    CameraModeInstances.Add(NewCameraMode);

    return NewCameraMode;
}

// 定期清理未使用的实例（可选）
void ULyraCameraModeStack::CleanupUnusedInstances()
{
    TArray<ULyraCameraMode*> UnusedModes;

    for (ULyraCameraMode* CameraMode : CameraModeInstances)
    {
        // 如果实例不在栈中，则标记为未使用
        if (!CameraModeStack.Contains(CameraMode))
        {
            UnusedModes.Add(CameraMode);
        }
    }

    // 移除未使用的实例
    for (ULyraCameraMode* Mode : UnusedModes)
    {
        CameraModeInstances.Remove(Mode);
        Mode->ConditionalBeginDestroy();
    }
}
```

### 22.10.6 相机更新频率调整

根据游戏状态动态调整相机更新频率。

```cpp
// LyraCameraComponent.h
UCLASS()
class ULyraCameraComponent : public UCameraComponent
{
    GENERATED_BODY()

public:
    // 设置更新频率（每秒帧数）
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void SetUpdateRate(float FramesPerSecond);

    // 暂停相机更新
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void PauseCameraUpdates(bool bPause);

protected:
    float TargetUpdateInterval = 0.0f;
    float TimeSinceLastUpdate = 0.0f;
    bool bPaused = false;

    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override
    {
        if (bPaused)
        {
            return;
        }

        TimeSinceLastUpdate += DeltaTime;

        // 如果设置了更新间隔，则按间隔更新
        if (TargetUpdateInterval > 0.0f && TimeSinceLastUpdate < TargetUpdateInterval)
        {
            return;
        }

        Super::TickComponent(TimeSinceLastUpdate, TickType, ThisTickFunction);
        TimeSinceLastUpdate = 0.0f;
    }
};

// LyraCameraComponent.cpp
void ULyraCameraComponent::SetUpdateRate(float FramesPerSecond)
{
    if (FramesPerSecond <= 0.0f)
    {
        TargetUpdateInterval = 0.0f; // 全帧率
    }
    else
    {
        TargetUpdateInterval = 1.0f / FramesPerSecond;
    }
}

void ULyraCameraComponent::PauseCameraUpdates(bool bPause)
{
    bPaused = bPause;
}

// 使用示例：在菜单打开时降低相机更新频率
void UGameMenuWidget::OnMenuOpened()
{
    if (APlayerController* PC = GetOwningPlayer())
    {
        if (APawn* Pawn = PC->GetPawn())
        {
            if (ULyraCameraComponent* Camera = ULyraCameraComponent::FindCameraComponent(Pawn))
            {
                Camera->SetUpdateRate(10.0f); // 降低到 10 FPS
            }
        }
    }
}

void UGameMenuWidget::OnMenuClosed()
{
    if (APlayerController* PC = GetOwningPlayer())
    {
        if (APawn* Pawn = PC->GetPawn())
        {
            if (ULyraCameraComponent* Camera = ULyraCameraComponent::FindCameraComponent(Pawn))
            {
                Camera->SetUpdateRate(0.0f); // 恢复全帧率
            }
        }
    }
}
```

---

## 22.11 总结与最佳实践

### 22.11.1 Lyra 相机系统优势

1. **模块化设计**：每个相机模式都是独立的类，易于扩展和维护
2. **平滑混合**：Camera Mode Stack 提供了自动的混合机制，无需手动编写插值代码
3. **数据驱动**：大部分参数可在编辑器中配置，无需修改代码
4. **碰撞避障**：内置的 Penetration Avoidance 系统解决了第三人称相机的常见问题
5. **性能优化**：通过 Trace Interval 和栈优化，减少了不必要的计算

### 22.11.2 最佳实践

#### 1. 相机模式的粒度

- **推荐**：为不同的游戏状态创建独立的相机模式
  - `CameraMode_ThirdPerson`（默认）
  - `CameraMode_Aiming`（瞄准）
  - `CameraMode_Driving`（驾驶）
  - `CameraMode_Climbing`（攀爬）
- **避免**：在一个相机模式中使用大量 `if/else` 来处理不同状态

#### 2. 混合时间的选择

| 场景 | 推荐混合时间 | 混合函数 |
|------|------------|---------|
| 快速战斗切换 | 0.2-0.3s | EaseIn |
| 瞄准进入 | 0.25-0.35s | EaseIn |
| 瞄准退出 | 0.3-0.4s | EaseOut |
| 死亡观战 | 1.5-2.0s | EaseInOut |
| 过场动画 | 1.0-2.0s | EaseInOut |

#### 3. FOV 配置

| 模式 | 推荐 FOV | 说明 |
|------|---------|------|
| 第三人称 | 80-90° | UE 默认 90° |
| 第一人称 | 90-100° | 更广阔的视野 |
| 瞄准 | 50-70° | 缩小视野增加精确度 |
| 狙击镜 2x | 40-50° | |
| 狙击镜 4x | 20-30° | |
| 狙击镜 8x | 10-15° | |

#### 4. 碰撞避障配置

- **主射线（Feeler 0）**：权重 1.0，每帧检测，球体半径 14
- **副射线（Feeler 1+）**：权重 0.5-0.75，间隔检测（3-5帧）
- **混合时间**：
  - `PenetrationBlendInTime = 0.1s`（靠近墙体快速）
  - `PenetrationBlendOutTime = 0.15s`（离开墙体稍慢，避免弹跳）

#### 5. 调试技巧

启用相机调试显示：

```cpp
// 控制台命令
showdebug camera

// 或在代码中
void ALyraPlayerCameraManager::DisplayDebug(UCanvas* Canvas, const FDebugDisplayInfo& DebugDisplay, float& YL, float& YPos)
{
    Super::DisplayDebug(Canvas, DebugDisplay, YL, YPos);

    const APawn* Pawn = (PCOwner ? PCOwner->GetPawn() : nullptr);
    if (const ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(Pawn))
    {
        CameraComponent->DrawDebug(Canvas);
    }
}
```

显示内容：
- 当前激活的相机模式及权重
- Camera Mode Stack 的所有层级
- 碰撞避障的 Feeler 射线

#### 6. 性能建议

- 避免在 `UpdateView` 中执行昂贵的操作（如复杂的物理查询）
- 使用 `TraceInterval` 优化副射线检测频率
- 考虑在低配置平台上禁用预测性避障（`bDoPredictiveAvoidance = false`）
- 使用 Async Trace 来异步执行射线检测（高级优化）

#### 7. 多人游戏注意事项

- 相机状态不要复制到网络
- 只复制影响 Gameplay 的状态（如瞄准标志）
- 使用客户端预测确保相机响应性
- 观战系统需要特殊处理（可能需要从服务器获取其他玩家的视角）

### 22.11.3 常见问题与解决方案

#### 问题1：相机穿透墙体

**解决方案**：
- 检查 `bPreventPenetration` 是否启用
- 增加 Feeler 数量或调整角度
- 减小 `CollisionPushOutDistance`
- 确保墙体有正确的碰撞通道配置（`ECC_Camera`）

#### 问题2：相机切换不平滑

**解决方案**：
- 增加 `BlendTime`
- 使用 `EaseInOut` 混合函数
- 检查是否有多个相机模式在短时间内推入栈

#### 问题3：瞄准时相机抖动

**解决方案**：
- 调整瞄准模式的 `BlendExponent`
- 使用 `EaseIn` 而不是 `Linear`
- 检查输入是否有噪声（增加死区）

#### 问题4：网络游戏中相机延迟

**解决方案**：
- 确保相机模式切换是本地的（不通过 RPC）
- 使用客户端预测
- 只同步必要的状态

### 22.11.4 扩展方向

1. **动画驱动的相机**：使用动画曲线控制相机偏移（如跑步时的镜头晃动）
2. **程序化相机效果**：根据速度、受击方向等动态调整相机
3. **VR 相机模式**：为 VR 创建专用的相机模式，处理 HMD 旋转
4. **电影化镜头语言**：实现景深、运镜路径、多机位切换等电影技巧
5. **智能相机**：根据敌人位置、战斗状态自动调整相机角度

## 22.12 高级主题：动画驱动的相机

### 22.12.1 镜头晃动（Camera Shake from Animation）

基于角色动画的相机晃动可以提供更自然的视觉反馈。

```cpp
// AnimatedCameraMode_ThirdPerson.h
UCLASS()
class UAnimatedCameraMode_ThirdPerson : public ULyraCameraMode_ThirdPerson
{
    GENERATED_BODY()

public:
    UAnimatedCameraMode_ThirdPerson()
    {
        HeadBobIntensity = 1.0f;
        WalkBobSpeed = 10.0f;
        SprintBobSpeed = 15.0f;
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    float HeadBobIntensity;

    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    float WalkBobSpeed;

    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    float SprintBobSpeed;

    float BobTime = 0.0f;

    virtual void UpdateView(float DeltaTime) override
    {
        Super::UpdateView(DeltaTime);

        // 获取角色移动状态
        if (const ACharacter* Character = Cast<ACharacter>(GetTargetActor()))
        {
            const UCharacterMovementComponent* MovementComp = Character->GetCharacterMovement();
            
            if (MovementComp && MovementComp->Velocity.SizeSquared() > 0.0f)
            {
                // 计算晃动
                bool bIsSprinting = MovementComp->Velocity.Size() > MovementComp->MaxWalkSpeed * 0.9f;
                float BobSpeed = bIsSprinting ? SprintBobSpeed : WalkBobSpeed;
                
                BobTime += DeltaTime * BobSpeed;

                // 上下晃动（正弦波）
                float VerticalBob = FMath::Sin(BobTime) * HeadBobIntensity * 2.0f;
                
                // 左右晃动（余弦波，频率是上下的一半）
                float HorizontalBob = FMath::Cos(BobTime * 0.5f) * HeadBobIntensity * 1.0f;

                // 应用晃动偏移
                FVector BobOffset = FVector(0.0f, HorizontalBob, VerticalBob);
                View.Location += GetPivotRotation().RotateVector(BobOffset);
            }
            else
            {
                // 停止时衰减晃动
                BobTime = FMath::FInterpTo(BobTime, 0.0f, DeltaTime, 5.0f);
            }
        }
    }
};
```

### 22.12.2 动态相机偏移（从 AnimNotify 触发）

创建 AnimNotify 来触发相机偏移事件。

```cpp
// AnimNotify_CameraOffset.h
UCLASS()
class UAnimNotify_CameraOffset : public UAnimNotify
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Camera")
    FVector Offset = FVector(0.0f, 0.0f, 10.0f);

    UPROPERTY(EditAnywhere, Category = "Camera")
    float Duration = 0.5f;

    UPROPERTY(EditAnywhere, Category = "Camera")
    UCurveFloat* OffsetCurve;

    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation) override
    {
        if (AActor* Owner = MeshComp->GetOwner())
        {
            if (ULyraCameraComponent* Camera = ULyraCameraComponent::FindCameraComponent(Owner))
            {
                // 添加临时偏移
                Camera->AddTemporaryCameraOffset(Offset, Duration, OffsetCurve);
            }
        }
    }
};

// LyraCameraComponent.h - 扩展
USTRUCT()
struct FTemporaryCameraOffset
{
    GENERATED_BODY()

    FVector Offset;
    float Duration;
    float ElapsedTime;
    TObjectPtr<UCurveFloat> Curve;
};

UCLASS()
class ULyraCameraComponent : public UCameraComponent
{
    GENERATED_BODY()

protected:
    TArray<FTemporaryCameraOffset> TemporaryOffsets;

public:
    void AddTemporaryCameraOffset(FVector Offset, float Duration, UCurveFloat* Curve);

protected:
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

        // 更新临时偏移
        UpdateTemporaryOffsets(DeltaTime);
    }

    void UpdateTemporaryOffsets(float DeltaTime)
    {
        FVector TotalOffset = FVector::ZeroVector;

        for (int32 i = TemporaryOffsets.Num() - 1; i >= 0; --i)
        {
            FTemporaryCameraOffset& Offset = TemporaryOffsets[i];
            Offset.ElapsedTime += DeltaTime;

            if (Offset.ElapsedTime >= Offset.Duration)
            {
                TemporaryOffsets.RemoveAt(i);
                continue;
            }

            float Alpha = Offset.ElapsedTime / Offset.Duration;
            float CurveValue = Offset.Curve ? Offset.Curve->GetFloatValue(Alpha) : (1.0f - Alpha);

            TotalOffset += Offset.Offset * CurveValue;
        }

        // 应用偏移到相机
        if (!TotalOffset.IsNearlyZero())
        {
            AddRelativeLocation(TotalOffset);
        }
    }
};

// LyraCameraComponent.cpp
void ULyraCameraComponent::AddTemporaryCameraOffset(FVector Offset, float Duration, UCurveFloat* Curve)
{
    FTemporaryCameraOffset TempOffset;
    TempOffset.Offset = Offset;
    TempOffset.Duration = Duration;
    TempOffset.ElapsedTime = 0.0f;
    TempOffset.Curve = Curve;

    TemporaryOffsets.Add(TempOffset);
}
```

### 22.12.3 实战：受击相机效果

当角色受到伤害时，相机产生冲击效果。

```cpp
// LyraCharacter.cpp - 扩展
void ALyraCharacter::OnTakeDamage(float Damage, FVector HitDirection)
{
    // 播放受击相机震动
    if (APlayerController* PC = GetController<APlayerController>())
    {
        float ShakeScale = FMath::Clamp(Damage / 50.0f, 0.2f, 1.0f);
        PC->ClientStartCameraShake(ULyraCameraShake_Hit::StaticClass(), ShakeScale);
    }

    // 添加相机偏移（朝受击方向倾斜）
    if (ULyraCameraComponent* Camera = ULyraCameraComponent::FindCameraComponent(this))
    {
        // 计算相机空间中的受击方向
        FVector CameraSpaceHitDir = GetControlRotation().UnrotateVector(HitDirection);
        
        // 相机朝受击方向偏移
        FVector ImpactOffset = CameraSpaceHitDir * Damage * 0.5f;
        ImpactOffset.Z = 0.0f; // 不改变高度

        Camera->AddTemporaryCameraOffset(ImpactOffset, 0.3f, ImpactOffsetCurve);
    }
}
```

## 22.13 高级主题：智能相机系统

### 22.13.1 自动聚焦敌人

相机自动调整角度以将敌人纳入视野。

```cpp
// SmartCameraMode_Combat.h
UCLASS()
class USmartCameraMode_Combat : public ULyraCameraMode_ThirdPerson
{
    GENERATED_BODY()

public:
    USmartCameraMode_Combat()
    {
        EnemyInfluenceRadius = 1500.0f;
        MaxYawAdjustment = 15.0f;
        AdjustmentSpeed = 2.0f;
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Smart Camera")
    float EnemyInfluenceRadius;

    UPROPERTY(EditDefaultsOnly, Category = "Smart Camera")
    float MaxYawAdjustment;

    UPROPERTY(EditDefaultsOnly, Category = "Smart Camera")
    float AdjustmentSpeed;

    float CurrentYawAdjustment = 0.0f;

    virtual void UpdateView(float DeltaTime) override
    {
        Super::UpdateView(DeltaTime);

        // 查找附近的敌人
        TArray<AActor*> NearbyEnemies;
        FindNearbyEnemies(NearbyEnemies);

        if (NearbyEnemies.Num() > 0)
        {
            // 计算敌人的平均位置
            FVector AverageEnemyLocation = CalculateAverageLocation(NearbyEnemies);

            // 计算需要调整的角度
            FVector ToEnemy = AverageEnemyLocation - View.Location;
            FRotator ToEnemyRotation = ToEnemy.Rotation();
            
            float YawDelta = FMath::FindDeltaAngleDegrees(View.Rotation.Yaw, ToEnemyRotation.Yaw);
            float TargetYawAdjustment = FMath::Clamp(YawDelta, -MaxYawAdjustment, MaxYawAdjustment);

            // 平滑插值
            CurrentYawAdjustment = FMath::FInterpTo(CurrentYawAdjustment, TargetYawAdjustment, DeltaTime, AdjustmentSpeed);

            // 应用调整
            View.Rotation.Yaw += CurrentYawAdjustment;
        }
        else
        {
            // 没有敌人，恢复正常
            CurrentYawAdjustment = FMath::FInterpTo(CurrentYawAdjustment, 0.0f, DeltaTime, AdjustmentSpeed);
            View.Rotation.Yaw += CurrentYawAdjustment;
        }
    }

    void FindNearbyEnemies(TArray<AActor*>& OutEnemies)
    {
        AActor* TargetActor = GetTargetActor();
        if (!TargetActor)
        {
            return;
        }

        TArray<FOverlapResult> Overlaps;
        FCollisionShape SphereShape = FCollisionShape::MakeSphere(EnemyInfluenceRadius);
        
        GetWorld()->OverlapMultiByChannel(
            Overlaps,
            TargetActor->GetActorLocation(),
            FQuat::Identity,
            ECC_Pawn,
            SphereShape
        );

        for (const FOverlapResult& Overlap : Overlaps)
        {
            AActor* OtherActor = Overlap.GetActor();
            if (OtherActor && OtherActor != TargetActor && IsEnemy(OtherActor))
            {
                OutEnemies.Add(OtherActor);
            }
        }
    }

    bool IsEnemy(AActor* Actor)
    {
        // 实现敌友判断逻辑
        // 可以使用 Team 系统、Gameplay Tags 等
        return true; // 简化示例
    }

    FVector CalculateAverageLocation(const TArray<AActor*>& Actors)
    {
        FVector Sum = FVector::ZeroVector;
        for (AActor* Actor : Actors)
        {
            Sum += Actor->GetActorLocation();
        }
        return Sum / Actors.Num();
    }
};
```

### 22.13.2 遮挡感知相机

当视线被遮挡时，相机自动调整位置以保持视野。

```cpp
// OcclusionAwareCameraMode.h
UCLASS()
class UOcclusionAwareCameraMode : public ULyraCameraMode_ThirdPerson
{
    GENERATED_BODY()

public:
    UOcclusionAwareCameraMode()
    {
        OcclusionCheckInterval = 0.1f;
        HorizontalShiftSpeed = 200.0f;
        MaxHorizontalShift = 100.0f;
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Occlusion")
    float OcclusionCheckInterval;

    UPROPERTY(EditDefaultsOnly, Category = "Occlusion")
    float HorizontalShiftSpeed;

    UPROPERTY(EditDefaultsOnly, Category = "Occlusion")
    float MaxHorizontalShift;

    float TimeSinceLastCheck = 0.0f;
    float CurrentHorizontalShift = 0.0f;
    bool bIsOccluded = false;

    virtual void UpdateView(float DeltaTime) override
    {
        Super::UpdateView(DeltaTime);

        TimeSinceLastCheck += DeltaTime;

        if (TimeSinceLastCheck >= OcclusionCheckInterval)
        {
            CheckOcclusion();
            TimeSinceLastCheck = 0.0f;
        }

        // 应用水平偏移
        if (bIsOccluded)
        {
            // 增加偏移
            CurrentHorizontalShift = FMath::FInterpConstantTo(
                CurrentHorizontalShift,
                MaxHorizontalShift,
                DeltaTime,
                HorizontalShiftSpeed
            );
        }
        else
        {
            // 恢复偏移
            CurrentHorizontalShift = FMath::FInterpConstantTo(
                CurrentHorizontalShift,
                0.0f,
                DeltaTime,
                HorizontalShiftSpeed
            );
        }

        if (!FMath::IsNearlyZero(CurrentHorizontalShift))
        {
            FVector RightVector = FRotationMatrix(View.Rotation).GetScaledAxis(EAxis::Y);
            View.Location += RightVector * CurrentHorizontalShift;
        }
    }

    void CheckOcclusion()
    {
        AActor* TargetActor = GetTargetActor();
        if (!TargetActor)
        {
            return;
        }

        FVector TargetLocation = TargetActor->GetActorLocation();
        FVector CameraLocation = View.Location;

        // 从相机到目标的射线检测
        FHitResult Hit;
        FCollisionQueryParams QueryParams;
        QueryParams.AddIgnoredActor(TargetActor);

        bIsOccluded = GetWorld()->LineTraceSingleByChannel(
            Hit,
            CameraLocation,
            TargetLocation,
            ECC_Visibility,
            QueryParams
        );
    }
};
```

### 22.13.3 电影化构图相机

应用电影构图规则（如三分法、引导线）自动调整相机。

```cpp
// CinematicCompositionCamera.h
UCLASS()
class UCinematicCompositionCamera : public ULyraCameraMode
{
    GENERATED_BODY()

public:
    UCinematicCompositionCamera()
    {
        bUseRuleOfThirds = true;
        CompositionOffsetStrength = 0.3f;
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Composition")
    bool bUseRuleOfThirds;

    UPROPERTY(EditDefaultsOnly, Category = "Composition")
    float CompositionOffsetStrength;

    UPROPERTY(EditDefaultsOnly, Category = "Composition")
    TObjectPtr<AActor> SecondaryFocusTarget; // 次要焦点（如对话对象）

    virtual void UpdateView(float DeltaTime) override
    {
        AActor* PrimaryTarget = GetTargetActor();
        if (!PrimaryTarget)
        {
            return;
        }

        FVector PrimaryLocation = PrimaryTarget->GetActorLocation();
        FVector CameraLocation = PrimaryLocation + FVector(-300.0f, 0.0f, 100.0f);

        if (SecondaryFocusTarget && bUseRuleOfThirds)
        {
            // 应用三分法构图
            FVector SecondaryLocation = SecondaryFocusTarget->GetActorLocation();
            
            // 计算两个目标的中点
            FVector MidPoint = (PrimaryLocation + SecondaryLocation) * 0.5f;

            // 相机看向中点
            FRotator LookAtRotation = (MidPoint - CameraLocation).Rotation();

            // 偏移相机位置，使主目标位于画面三分之一处
            FVector OffsetDirection = (SecondaryLocation - PrimaryLocation).GetSafeNormal();
            FVector CompositionOffset = OffsetDirection * CompositionOffsetStrength * 100.0f;

            CameraLocation += CompositionOffset;

            View.Location = CameraLocation;
            View.Rotation = LookAtRotation;
        }
        else
        {
            // 标准视图
            View.Location = CameraLocation;
            View.Rotation = (PrimaryLocation - CameraLocation).Rotation();
        }

        View.ControlRotation = View.Rotation;
        View.FieldOfView = FieldOfView;
    }
};
```

---

## 总结

本章深入剖析了 Lyra 的相机系统，从基础的 `ULyraCameraMode` 和 `ULyraCameraModeStack` 到高级的碰撞避障、电影镜头系统，涵盖了相机系统的方方面面。通过 5 个实战案例，你应该能够：

1. 理解 Lyra 相机系统的核心架构和设计理念
2. 创建自定义的相机模式并配置混合参数
3. 实现第三人称、第一人称、瞄准等常见相机模式
4. 使用碰撞避障系统防止相机穿透
5. 构建电影镜头系统和过场动画
6. 优化相机性能并处理网络同步

Lyra 的相机系统展示了现代游戏引擎中相机设计的最佳实践，其模块化、数据驱动的理念值得在你的项目中借鉴和应用。

---

**下一章预告**：第23章将探讨 Lyra 的 UI 系统与 Common UI 框架，学习如何构建跨平台的游戏界面。
