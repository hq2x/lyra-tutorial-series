# 相机系统：自适应视角控制

> 深度解析 Lyra 的相机系统架构、自适应穿透避免、相机模式栈与平滑过渡机制

---

## 📚 本章内容

- **LyraCameraComponent 架构解析**：相机组件的核心设计与职责划分
- **CameraMode 系统**：抽象相机模式与可扩展的视角策略
- **CameraModeStack 混合机制**：多模式平滑过渡与权重计算
- **第三人称相机实现**：基于 Curve 的动态偏移与穿透避免
- **自适应碰撞检测**：PenetrationAvoidance 防穿墙系统
- **实战案例**：自定义瞄准模式、动态 FOV、死亡相机等

---

## 🎯 学习目标

完成本章后，你将能够：

✅ 理解 Lyra 相机系统的分层架构与职责分离  
✅ 掌握 CameraMode 的生命周期与混合算法  
✅ 实现自定义相机模式（如 TPS、FPS、俯视角、观战）  
✅ 调试和优化相机穿透避免系统  
✅ 集成输入系统与相机控制的联动  

---

## 1. Lyra 相机系统架构总览

### 1.1 核心组件关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                      PlayerController                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │        LyraPlayerCameraManager (APlayerCameraManager)    │   │
│  │  - BlueprintUpdateCamera() 入口                           │   │
│  │  - 协调多个 CameraModifier                                │   │
│  └────────────────────┬─────────────────────────────────────┘   │
└───────────────────────┼──────────────────────────────────────────┘
                        │ 委托给
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Pawn / Character                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │    LyraCameraComponent (UCameraComponent)                │   │
│  │  - 持有 CameraModeStack                                   │   │
│  │  - 通过 DetermineCameraModeDelegate 查询当前模式          │   │
│  │  - 驱动 CameraMode 更新并输出 FMinimalViewInfo            │   │
│  └────────────────────┬─────────────────────────────────────┘   │
└───────────────────────┼──────────────────────────────────────────┘
                        │ 管理
                        ▼
        ┌───────────────────────────────────────┐
        │     ULyraCameraModeStack              │
        │  - 维护 CameraMode 实例池              │
        │  - 管理模式的激活/入栈/混合/出栈       │
        │  - EvaluateStack() 输出最终视图        │
        └────────────────┬──────────────────────┘
                         │ 包含
                         ▼
        ┌────────────────────────────────────────┐
        │  ULyraCameraMode（抽象基类）            │
        │  - UpdateView()：计算位置/旋转/FOV      │
        │  - UpdateBlending()：计算混合权重       │
        │  - BlendFunction：Linear/EaseIn/EaseOut │
        │  - CameraTypeTag：标识模式类型          │
        └─────────────┬──────────────────────────┘
                      │ 派生
                      ▼
      ┌────────────────────────────┬──────────────────────┐
      ▼                            ▼                      ▼
ULyraCameraMode_    ULyraCameraMode_      自定义模式
  ThirdPerson           TopDownArena      (FPS/Death/等)
  - TargetOffsetCurve    - 正交投影
  - PenetrationAvoidance - 固定高度
  - CrouchOffset
```

### 1.2 数据流概览

```
每帧渲染流程：

1. PlayerCameraManager::UpdateCamera()
   └─> LyraCameraComponent::GetCameraView()
       ├─> UpdateCameraModes()  // 查询应该使用哪个 CameraMode
       │   └─> DetermineCameraModeDelegate.Execute()
       │       └─> ULyraHeroComponent::DetermineCameraMode()
       │           └─> 根据 Ability Tags 返回 CameraMode 类
       │
       ├─> CameraModeStack::EvaluateStack(DeltaTime)
       │   ├─> UpdateStack()  // 更新所有模式的状态和权重
       │   │   └─> CameraMode::UpdateCameraMode()
       │   │       ├─> UpdateView()        // 计算理想位置/旋转
       │   │       └─> UpdateBlending()    // 更新混合权重
       │   │
       │   └─> BlendStack()   // 按权重混合所有活跃模式
       │       └─> FLyraCameraModeView::Blend()  // 线性插值
       │
       └─> 输出 FMinimalViewInfo
           ├─> Location / Rotation / FOV
           ├─> PostProcess Settings
           └─> 同步到 PlayerController::ControlRotation
```

### 1.3 关键设计原则

| 设计原则 | 体现 |
|---------|------|
| **职责分离** | CameraComponent 只负责协调，具体逻辑在 CameraMode 中 |
| **策略模式** | CameraMode 是可替换的策略，通过 Delegate 动态选择 |
| **状态机思想** | CameraModeStack 管理模式的激活/过渡/混合 |
| **平滑过渡** | 多模式堆栈支持权重混合，避免视角跳变 |
| **数据驱动** | Curve 曲线控制偏移、混合函数可配置 |

---

## 2. LyraCameraComponent 详解

### 2.1 核心职责

`ULyraCameraComponent` 继承自 `UCameraComponent`，主要负责：

1. **持有 CameraModeStack**，管理相机模式的生命周期
2. **查询当前应该使用的 CameraMode**（通过 Delegate）
3. **每帧驱动 Stack 更新**并获取最终的 View 数据
4. **同步 ControlRotation** 到 PlayerController（保持输入与视角一致）
5. **应用临时 FOV 偏移**（如武器开镜、冲刺等效果）

### 2.2 源码解析

```cpp
UCLASS()
class ULyraCameraComponent : public UCameraComponent
{
    GENERATED_BODY()

public:
    // 查询最佳相机模式的 Delegate（由 ULyraHeroComponent 绑定）
    FLyraCameraModeDelegate DetermineCameraModeDelegate;

    // 添加单帧 FOV 偏移（用于武器后坐力、冲刺等效果）
    void AddFieldOfViewOffset(float FovOffset) { FieldOfViewOffset += FovOffset; }

    // 获取当前顶层模式的标签和权重（用于 UI 显示或逻辑判断）
    void GetBlendInfo(float& OutWeightOfTopLayer, FGameplayTag& OutTagOfTopLayer) const;

protected:
    // 持有的 CameraMode 堆栈
    UPROPERTY()
    TObjectPtr<ULyraCameraModeStack> CameraModeStack;

    // 单帧 FOV 偏移（应用后自动清零）
    float FieldOfViewOffset;

    // 核心函数：每帧被 PlayerCameraManager 调用
    virtual void GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView) override;

    // 查询并推入当前应该使用的 CameraMode
    virtual void UpdateCameraModes();
};
```

### 2.3 GetCameraView 流程详解

这是相机系统的核心入口函数，每帧被调用一次：

```cpp
void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)
{
    check(CameraModeStack);

    // 1. 更新当前应该使用的 CameraMode（可能会推新模式入栈）
    UpdateCameraModes();

    // 2. 让 Stack 计算混合后的最终 View
    FLyraCameraModeView CameraModeView;
    CameraModeStack->EvaluateStack(DeltaTime, CameraModeView);

    // 3. 同步 ControlRotation 到 PlayerController
    //    确保输入方向与相机朝向一致（尤其对第三人称重要）
    if (APawn* TargetPawn = Cast<APawn>(GetTargetActor()))
    {
        if (APlayerController* PC = TargetPawn->GetController<APlayerController>())
        {
            PC->SetControlRotation(CameraModeView.ControlRotation);
        }
    }

    // 4. 应用临时 FOV 偏移（如武器开镜）
    CameraModeView.FieldOfView += FieldOfViewOffset;
    FieldOfViewOffset = 0.0f;  // 单帧有效，立即清零

    // 5. 同步组件自身的 Transform（用于 Audio Listener 等）
    SetWorldLocationAndRotation(CameraModeView.Location, CameraModeView.Rotation);
    FieldOfView = CameraModeView.FieldOfView;

    // 6. 填充输出参数（最终传给 SceneCapture）
    DesiredView.Location = CameraModeView.Location;
    DesiredView.Rotation = CameraModeView.Rotation;
    DesiredView.FOV = CameraModeView.FieldOfView;
    // ... PostProcess、AspectRatio 等配置
}
```

**关键点**：
- `UpdateCameraModes()` 只在 Stack 未激活或需要切换模式时推入新模式
- `EvaluateStack()` 是每帧都执行的核心混合逻辑
- `ControlRotation` 同步保证了输入和视角的一致性（第三人称相机朝向 ≠ 角色朝向）

### 2.4 UpdateCameraModes 动态模式选择

```cpp
void ULyraCameraComponent::UpdateCameraModes()
{
    check(CameraModeStack);

    // 只有在 Stack 已激活时才查询（避免未初始化时查询）
    if (CameraModeStack->IsStackActivate())
    {
        if (DetermineCameraModeDelegate.IsBound())
        {
            // 通过 Delegate 查询当前应该使用的 CameraMode 类
            if (const TSubclassOf<ULyraCameraMode> CameraMode = DetermineCameraModeDelegate.Execute())
            {
                // 如果和栈顶不同，会自动推入并开始混合
                CameraModeStack->PushCameraMode(CameraMode);
            }
        }
    }
}
```

**Delegate 绑定示例**（在 `ULyraHeroComponent` 中）：

```cpp
void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    // ...

    if (ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(GetPawn()))
    {
        CameraComponent->DetermineCameraModeDelegate.BindUObject(this, &ThisClass::DetermineCameraMode);
    }
}

TSubclassOf<ULyraCameraMode> ULyraHeroComponent::DetermineCameraMode() const
{
    // 可根据 Ability Tags、武器状态等动态决定
    if (AbilitySystemComponent->HasMatchingGameplayTag(TAG_Gameplay_Aiming))
    {
        return AimingCameraMode;  // 瞄准模式
    }

    return DefaultCameraMode;  // 默认第三人称模式
}
```

---

## 3. CameraMode 基类与生命周期

### 3.1 ULyraCameraMode 核心成员

```cpp
UCLASS(Abstract, NotBlueprintable)
class ULyraCameraMode : public UObject
{
    GENERATED_BODY()

public:
    // 查询绑定的 CameraComponent
    ULyraCameraComponent* GetLyraCameraComponent() const;

    // 查询目标 Actor（通常是 Pawn）
    AActor* GetTargetActor() const;

    // 生命周期回调
    virtual void OnActivation() {}   // 首次入栈时调用
    virtual void OnDeactivation() {} // 从栈中移除时调用

    // 每帧更新
    void UpdateCameraMode(float DeltaTime);

    // 获取/设置混合权重
    float GetBlendWeight() const { return BlendWeight; }
    void SetBlendWeight(float Weight);

    // 获取模式类型标签（如 Camera.Mode.ThirdPerson）
    FGameplayTag GetCameraTypeTag() const { return CameraTypeTag; }

protected:
    // 模式类型标签（用于外部查询当前模式）
    UPROPERTY(EditDefaultsOnly, Category = "Blending")
    FGameplayTag CameraTypeTag;

    // 输出的视图数据（Location/Rotation/FOV 等）
    FLyraCameraModeView View;

    // 视野角度
    UPROPERTY(EditDefaultsOnly, Category = "View")
    float FieldOfView = 80.0f;

    // Pitch 角度限制（防止看到天花板/地板内部）
    UPROPERTY(EditDefaultsOnly, Category = "View")
    float ViewPitchMin = -89.0f;
    UPROPERTY(EditDefaultsOnly, Category = "View")
    float ViewPitchMax = 89.0f;

    // 混合时间（从权重 0 → 1 需要多久）
    UPROPERTY(EditDefaultsOnly, Category = "Blending")
    float BlendTime = 0.5f;

    // 混合函数（Linear / EaseIn / EaseOut / EaseInOut）
    UPROPERTY(EditDefaultsOnly, Category = "Blending")
    ELyraCameraModeBlendFunction BlendFunction = ELyraCameraModeBlendFunction::EaseOut;

    // 曲线的陡峭程度（Exponent，值越大变化越剧烈）
    UPROPERTY(EditDefaultsOnly, Category = "Blending")
    float BlendExponent = 4.0f;

    // 混合进度（0 → 1）
    float BlendAlpha = 1.0f;

    // 最终混合权重（经过 BlendFunction 计算后的值）
    float BlendWeight = 1.0f;

    // 子类实现：计算相机的 Pivot 位置（通常是角色眼睛位置）
    virtual FVector GetPivotLocation() const;

    // 子类实现：计算相机的 Pivot 旋转（通常是 ControlRotation）
    virtual FRotator GetPivotRotation() const;

    // 子类实现：计算相机的最终位置/旋转/FOV
    virtual void UpdateView(float DeltaTime);

    // 子类可覆盖：自定义混合权重计算（默认按时间线性增长）
    virtual void UpdateBlending(float DeltaTime);
};
```

### 3.2 UpdateCameraMode 执行流程

```cpp
void ULyraCameraMode::UpdateCameraMode(float DeltaTime)
{
    UpdateView(DeltaTime);      // 计算 View 数据
    UpdateBlending(DeltaTime);  // 计算混合权重
}
```

### 3.3 UpdateView 基础实现

```cpp
void ULyraCameraMode::UpdateView(float DeltaTime)
{
    // 1. 获取 Pivot（通常是角色眼睛位置）
    FVector PivotLocation = GetPivotLocation();
    FRotator PivotRotation = GetPivotRotation();

    // 2. 限制 Pitch 角度（防止向上/下看得太过）
    PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

    // 3. 基础实现：相机位置 = Pivot 位置，朝向 = Pivot 朝向
    View.Location = PivotLocation;
    View.Rotation = PivotRotation;
    View.ControlRotation = View.Rotation;  // 保持输入和视角一致
    View.FieldOfView = FieldOfView;
}
```

**子类覆盖示例**：
- **第三人称模式**：在 Pivot 基础上加偏移（肩膀位置、后方距离等）
- **第一人称模式**：直接使用 Pivot（眼睛位置）
- **俯视角模式**：固定高度俯视，忽略 Pitch

### 3.4 UpdateBlending 混合权重计算

```cpp
void ULyraCameraMode::UpdateBlending(float DeltaTime)
{
    if (BlendTime > 0.0f)
    {
        // 线性增长 Alpha（0 → 1）
        BlendAlpha += (DeltaTime / BlendTime);
        BlendAlpha = FMath::Min(BlendAlpha, 1.0f);
    }
    else
    {
        // 无混合时间，直接跳到 1
        BlendAlpha = 1.0f;
    }

    // 根据混合函数计算最终权重
    const float Exponent = (BlendExponent > 0.0f) ? BlendExponent : 1.0f;

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

**混合函数对比**：

| 函数 | 效果 | 适用场景 |
|------|------|---------|
| **Linear** | 匀速过渡 | 快速切换，无需细腻感觉 |
| **EaseIn** | 慢启动，快结束 | 相机开始移动时平滑 |
| **EaseOut** | 快启动，慢结束 | 相机到位时平滑（**Lyra 默认**） |
| **EaseInOut** | 两头慢，中间快 | 长距离平移，追求极致平滑 |

---

## 4. CameraModeStack 混合机制

### 4.1 堆栈管理原则

`ULyraCameraModeStack` 维护一个 **有序栈**，栈顶是最新推入的模式：

- **栈顶**（Index 0）：优先级最高，BlendWeight 从 0 逐渐增长到 1
- **栈底**（Index N-1）：始终保持 BlendWeight = 1.0（作为基础视角）
- **中间模式**：逐渐被新模式覆盖，当权重降到 0 时自动移除

### 4.2 PushCameraMode 推入逻辑

```cpp
void ULyraCameraModeStack::PushCameraMode(TSubclassOf<ULyraCameraMode> CameraModeClass)
{
    if (!CameraModeClass) return;

    // 1. 获取或创建模式实例（复用池中的实例）
    ULyraCameraMode* CameraMode = GetCameraModeInstance(CameraModeClass);
    check(CameraMode);

    int32 StackSize = CameraModeStack.Num();

    // 2. 如果已经是栈顶，直接返回（避免重复推入）
    if ((StackSize > 0) && (CameraModeStack[0] == CameraMode))
    {
        return;
    }

    // 3. 查找是否已在栈中（如果在，计算其当前贡献度）
    int32 ExistingStackIndex = INDEX_NONE;
    float ExistingStackContribution = 1.0f;

    for (int32 i = 0; i < StackSize; ++i)
    {
        if (CameraModeStack[i] == CameraMode)
        {
            ExistingStackIndex = i;
            ExistingStackContribution *= CameraMode->GetBlendWeight();
            break;
        }
        else
        {
            // 计算该模式在栈中的"可见度"（被上层遮挡后的剩余权重）
            ExistingStackContribution *= (1.0f - CameraModeStack[i]->GetBlendWeight());
        }
    }

    // 4. 如果已在栈中，移除旧位置
    if (ExistingStackIndex != INDEX_NONE)
    {
        CameraModeStack.RemoveAt(ExistingStackIndex);
        StackSize--;
    }
    else
    {
        ExistingStackContribution = 0.0f;  // 全新模式，从 0 开始混合
    }

    // 5. 决定初始权重（如果需要混合，继承之前的贡献度）
    const bool bShouldBlend = ((CameraMode->GetBlendTime() > 0.0f) && (StackSize > 0));
    const float InitialBlendWeight = bShouldBlend ? ExistingStackContribution : 1.0f;

    CameraMode->SetBlendWeight(InitialBlendWeight);

    // 6. 插入到栈顶
    CameraModeStack.Insert(CameraMode, 0);

    // 7. 确保栈底权重始终为 1.0（作为基础视角）
    CameraModeStack.Last()->SetBlendWeight(1.0f);

    // 8. 触发生命周期回调
    if (ExistingStackIndex == INDEX_NONE)
    {
        CameraMode->OnActivation();  // 首次入栈
    }
}
```

**关键点**：
- **ExistingStackContribution**：如果模式已在栈中，计算其"当前可见度"，避免从 0 重新混合导致的视角跳变
- **栈底保底**：最后一个模式权重锁定为 1.0，确保即使栈顶还在混合，总有一个完整的视角作为基础

### 4.3 EvaluateStack 混合计算

```cpp
bool ULyraCameraModeStack::EvaluateStack(float DeltaTime, FLyraCameraModeView& OutCameraModeView)
{
    if (!bIsActive) return false;

    // 1. 更新所有模式的状态和权重
    UpdateStack(DeltaTime);

    // 2. 从栈底到栈顶逐层混合
    BlendStack(OutCameraModeView);

    return true;
}
```

**UpdateStack 更新逻辑**：

```cpp
void ULyraCameraModeStack::UpdateStack(float DeltaTime)
{
    const int32 StackSize = CameraModeStack.Num();
    if (StackSize <= 0) return;

    int32 RemoveCount = 0;
    int32 RemoveIndex = INDEX_NONE;

    // 遍历所有模式，更新其 View 和 BlendWeight
    for (int32 i = 0; i < StackSize; ++i)
    {
        ULyraCameraMode* CameraMode = CameraModeStack[i];
        check(CameraMode);

        CameraMode->UpdateCameraMode(DeltaTime);

        // 如果某个模式的权重达到 1.0，下面的模式已经完全被覆盖
        if (CameraMode->GetBlendWeight() >= 1.0f)
        {
            RemoveIndex = i + 1;
            RemoveCount = StackSize - RemoveIndex;
            break;
        }
    }

    // 移除被完全覆盖的模式（优化性能）
    if (RemoveCount > 0)
    {
        for (int32 i = RemoveIndex; i < StackSize; ++i)
        {
            CameraModeStack[i]->OnDeactivation();
        }
        CameraModeStack.RemoveAt(RemoveIndex, RemoveCount);
    }
}
```

**BlendStack 混合视图**：

```cpp
void ULyraCameraModeStack::BlendStack(FLyraCameraModeView& OutCameraModeView) const
{
    const int32 StackSize = CameraModeStack.Num();
    if (StackSize <= 0) return;

    // 从栈底开始（权重 1.0 的基础视角）
    const ULyraCameraMode* BaseMode = CameraModeStack.Last();
    OutCameraModeView = BaseMode->GetCameraModeView();

    // 从栈底向栈顶逐层混合
    for (int32 i = StackSize - 2; i >= 0; --i)
    {
        const ULyraCameraMode* CameraMode = CameraModeStack[i];
        OutCameraModeView.Blend(CameraMode->GetCameraModeView(), CameraMode->GetBlendWeight());
    }
}
```

**FLyraCameraModeView::Blend 实现**：

```cpp
void FLyraCameraModeView::Blend(const FLyraCameraModeView& Other, float OtherWeight)
{
    if (OtherWeight <= 0.0f) return;       // 无影响
    if (OtherWeight >= 1.0f)               // 完全覆盖
    {
        *this = Other;
        return;
    }

    // 线性插值位置
    Location = FMath::Lerp(Location, Other.Location, OtherWeight);

    // 旋转需要归一化差值后插值（避免 180° 翻转）
    const FRotator DeltaRotation = (Other.Rotation - Rotation).GetNormalized();
    Rotation = Rotation + (OtherWeight * DeltaRotation);

    const FRotator DeltaControlRotation = (Other.ControlRotation - ControlRotation).GetNormalized();
    ControlRotation = ControlRotation + (OtherWeight * DeltaControlRotation);

    // 线性插值 FOV
    FieldOfView = FMath::Lerp(FieldOfView, Other.FieldOfView, OtherWeight);
}
```

### 4.4 混合示例

假设有两个模式：
- **Mode A**（第三人称）：权重 0.3
- **Mode B**（瞄准）：权重 1.0（栈底）

混合过程：
```
OutView = Mode B 的 View  // 从栈底开始
OutView.Blend(Mode A 的 View, 0.3)  // 混入 30% 的 Mode A

最终 OutView.Location = Lerp(B.Location, A.Location, 0.3)
```

当 Mode A 的权重逐渐增长到 1.0 时，Mode B 会被自动移除，Mode A 成为新的栈底。

---

## 5. 第三人称相机深度解析

### 5.1 ULyraCameraMode_ThirdPerson 核心功能

这是 Lyra 中最复杂的相机模式，包含以下特性：

1. **基于 Curve 的动态偏移**：根据 Pitch 角度调整肩膀位置、后方距离、高度
2. **下蹲偏移补偿**：角色下蹲时平滑调整相机高度
3. **穿透避免系统**：多射线检测遮挡，动态调整相机距离
4. **预测性避障**：提前检测潜在碰撞，防止视角突然拉近

### 5.2 TargetOffsetCurve 动态偏移

```cpp
UCLASS(Blueprintable)
class ULyraCameraMode_ThirdPerson : public ULyraCameraMode
{
    GENERATED_BODY()

protected:
    // Curve 曲线：X 轴是 Pitch 角度，Y 轴是相对偏移（X/Y/Z）
    UPROPERTY(EditDefaultsOnly, Category = "Third Person")
    TObjectPtr<const UCurveVector> TargetOffsetCurve;

    // 是否使用运行时 Float Curve（可 PIE 时编辑）
    UPROPERTY(EditDefaultsOnly, Category = "Third Person")
    bool bUseRuntimeFloatCurves = false;

    UPROPERTY(EditDefaultsOnly, Category = "Third Person", Meta = (EditCondition = "bUseRuntimeFloatCurves"))
    FRuntimeFloatCurve TargetOffsetX;  // 前后距离

    UPROPERTY(EditDefaultsOnly, Category = "Third Person", Meta = (EditCondition = "bUseRuntimeFloatCurves"))
    FRuntimeFloatCurve TargetOffsetY;  // 左右偏移（肩膀）

    UPROPERTY(EditDefaultsOnly, Category = "Third Person", Meta = (EditCondition = "bUseRuntimeFloatCurves"))
    FRuntimeFloatCurve TargetOffsetZ;  // 高度偏移
};
```

**UpdateView 实现**：

```cpp
void ULyraCameraMode_ThirdPerson::UpdateView(float DeltaTime)
{
    // 1. 更新下蹲偏移（平滑过渡）
    UpdateForTarget(DeltaTime);
    UpdateCrouchOffset(DeltaTime);

    // 2. 获取 Pivot（角色眼睛位置 + 下蹲偏移）
    FVector PivotLocation = GetPivotLocation() + CurrentCrouchOffset;
    FRotator PivotRotation = GetPivotRotation();

    PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

    View.Location = PivotLocation;
    View.Rotation = PivotRotation;
    View.ControlRotation = View.Rotation;
    View.FieldOfView = FieldOfView;

    // 3. 应用基于 Pitch 的动态偏移
    if (bUseRuntimeFloatCurves)
    {
        FVector TargetOffset(0.0f);
        TargetOffset.X = TargetOffsetX.GetRichCurveConst()->Eval(PivotRotation.Pitch);
        TargetOffset.Y = TargetOffsetY.GetRichCurveConst()->Eval(PivotRotation.Pitch);
        TargetOffset.Z = TargetOffsetZ.GetRichCurveConst()->Eval(PivotRotation.Pitch);

        // 将偏移从相机空间转换到世界空间
        View.Location = PivotLocation + PivotRotation.RotateVector(TargetOffset);
    }
    else if (TargetOffsetCurve)
    {
        const FVector TargetOffset = TargetOffsetCurve->GetVectorValue(PivotRotation.Pitch);
        View.Location = PivotLocation + PivotRotation.RotateVector(TargetOffset);
    }

    // 4. 穿透避免（可能会调整 View.Location）
    UpdatePreventPenetration(DeltaTime);
}
```

**Curve 配置示例**：

假设 `TargetOffsetCurve` 配置如下：

| Pitch | Offset X | Offset Y | Offset Z |
|-------|----------|----------|----------|
| -90° | -150 | 50 | 0 |
| -45° | -200 | 50 | 20 |
| 0° | -250 | 50 | 30 |
| +45° | -300 | 50 | 40 |
| +90° | -350 | 50 | 50 |

效果：
- **向上看**（-90°）：相机拉近，减少遮挡
- **平视**（0°）：标准第三人称距离
- **向下看**（+90°）：相机拉远，防止看穿地面

### 5.3 下蹲偏移补偿

```cpp
void ULyraCameraMode_ThirdPerson::UpdateForTarget(float DeltaTime)
{
    if (const ACharacter* TargetCharacter = Cast<ACharacter>(GetTargetActor()))
    {
        if (TargetCharacter->IsCrouched())
        {
            // 计算下蹲时的眼睛高度差异
            const ACharacter* CDO = TargetCharacter->GetClass()->GetDefaultObject<ACharacter>();
            const float CrouchedHeightAdjustment = CDO->CrouchedEyeHeight - CDO->BaseEyeHeight;

            // 设置目标偏移（会在 UpdateCrouchOffset 中平滑插值）
            SetTargetCrouchOffset(FVector(0.f, 0.f, CrouchedHeightAdjustment));
            return;
        }
    }

    // 未下蹲，偏移归零
    SetTargetCrouchOffset(FVector::ZeroVector);
}

void ULyraCameraMode_ThirdPerson::UpdateCrouchOffset(float DeltaTime)
{
    const FVector TargetCrouchOffsetVector = TargetCrouchOffset - CurrentCrouchOffset;

    if (!TargetCrouchOffsetVector.IsNearlyZero(0.01f))
    {
        const FVector Offset = TargetCrouchOffsetVector * FMath::Min(1.0f, CrouchOffsetBlendMultiplier * DeltaTime);
        CurrentCrouchOffset += Offset;
    }
    else
    {
        CurrentCrouchOffset = TargetCrouchOffset;
        CrouchOffsetBlendPct = 1.0f;
    }
}
```

**效果**：
- 角色按下蹲键后，相机不会立即下降，而是平滑过渡（避免视角抖动）
- `CrouchOffsetBlendMultiplier` 控制过渡速度（默认 5.0，即 0.2 秒完成）

### 5.4 穿透避免系统

**核心思想**：
1. 从理想位置向角色发射多条射线（Feelers）
2. 如果射线碰撞到墙体，调整相机位置到碰撞点前方
3. 使用多射线预测性扫描，提前避开潜在遮挡

**Feeler 配置**：

```cpp
ULyraCameraMode_ThirdPerson::ULyraCameraMode_ThirdPerson()
{
    // 添加默认 Feelers
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
        FRotator(0.0f, 0.0f, 0.0f),   // 正前方
        1.00f,  // 扩展范围
        1.00f,  // 权重
        14.f,   // 半径
        0       // 优先级
    ));

    // 左右扫描（检测侧面遮挡）
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
        FRotator(0.0f, +16.0f, 0.0f), 0.75f, 0.75f, 0.f, 3
    ));
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
        FRotator(0.0f, -16.0f, 0.0f), 0.75f, 0.75f, 0.f, 3
    ));

    // 更大角度扫描（预测性避障）
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
        FRotator(0.0f, +32.0f, 0.0f), 0.50f, 0.50f, 0.f, 5
    ));
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
        FRotator(0.0f, -32.0f, 0.0f), 0.50f, 0.50f, 0.f, 5
    ));

    // 上下扫描
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
        FRotator(+20.0f, 0.0f, 0.0f), 1.00f, 1.00f, 0.f, 4
    ));
    PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
        FRotator(-20.0f, 0.0f, 0.0f), 0.50f, 0.50f, 0.f, 4
    ));
}
```

**UpdatePreventPenetration 核心逻辑**：

```cpp
void ULyraCameraMode_ThirdPerson::UpdatePreventPenetration(float DeltaTime)
{
    if (!bPreventPenetration) return;

    AActor* TargetActor = GetTargetActor();

    // 获取"安全位置"（通常是角色胶囊体中心）
    FVector SafeLocation = TargetActor->GetActorLocation();

    // 调用穿透检测（会修改 View.Location）
    float DistBlockedPct = 0.0f;
    PreventCameraPenetration(*TargetActor, SafeLocation, View.Location, DeltaTime, DistBlockedPct, false);
}
```

**PreventCameraPenetration 实现**（简化版）：

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

    // 1. 主射线：从角色到相机的直线
    FVector BaseRay = CameraLoc - SafeLoc;
    FRotationMatrix BaseRotation(BaseRay.Rotation());
    float DistanceToCenter = BaseRay.Size();

    FCollisionQueryParams QueryParams(SCENE_QUERY_STAT(SpringArm), false, &ViewTarget);

    // 2. 遍历所有 Feelers
    for (int32 i = 0; i < PenetrationAvoidanceFeelers.Num(); ++i)
    {
        const FLyraPenetrationAvoidanceFeeler& Feeler = PenetrationAvoidanceFeelers[i];

        // 计算射线方向（基于 Feeler 的旋转）
        FVector FeelerRay = BaseRotation.TransformVector(Feeler.AdjustmentRot.Vector());
        FeelerRay *= DistanceToCenter * Feeler.Extent;

        FVector EndLocation = SafeLoc + FeelerRay;

        // 执行球形扫描
        FHitResult Hit;
        bool bHit = GetWorld()->SweepSingleByChannel(
            Hit,
            SafeLoc,
            EndLocation,
            FQuat::Identity,
            ECC_Camera,
            FCollisionShape::MakeSphere(Feeler.Radius),
            QueryParams
        );

        if (bHit && Hit.bBlockingHit)
        {
            // 计算被阻挡的百分比
            float NewBlockPct = Hit.Time;

            if (i == 0)
            {
                // 主射线硬阻挡
                HardBlockedPct = FMath::Max(NewBlockPct, HardBlockedPct);
            }
            else if (bDoPredictiveAvoidance)
            {
                // 预测性阻挡
                SoftBlockedPct = FMath::Max(NewBlockPct, SoftBlockedPct);
            }
        }
    }

    // 3. 应用阻挡（硬阻挡优先）
    DistBlockedPct = FMath::Max(HardBlockedPct, SoftBlockedPct);

    if (DistBlockedPct < 1.0f)
    {
        // 拉近相机到阻挡点
        CameraLoc = SafeLoc + (CameraLoc - SafeLoc) * DistBlockedPct;

        // 再向内推一点（避免贴在墙上）
        CameraLoc += (SafeLoc - CameraLoc).GetSafeNormal() * CollisionPushOutDistance;
    }
}
```

**可视化 Debug**：

```cpp
void ULyraCameraMode_ThirdPerson::DrawDebug(UCanvas* Canvas) const
{
    Super::DrawDebug(Canvas);

    FDisplayDebugManager& DisplayDebugManager = Canvas->DisplayDebugManager;

    // 显示所有碰撞到的 Actor
    for (int i = 0; i < DebugActorsHitDuringCameraPenetration.Num(); i++)
    {
        DisplayDebugManager.DrawString(
            FString::Printf(TEXT("HitActor[%d]: %s"), i,
                *DebugActorsHitDuringCameraPenetration[i]->GetName())
        );
    }
}
```

在游戏中按 `` ` `` 键打开控制台，输入：
```
showdebug camera
```
可以看到：
- 当前相机位置/旋转/FOV
- 每个 CameraMode 的权重
- 穿透检测碰撞的 Actor 列表

---

## 6. 实战案例：自定义相机模式

### 6.1 案例 1：第一人称相机

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
        // 第一人称通常有更大的 FOV
        FieldOfView = 90.0f;

        // 快速混合（0.2 秒）
        BlendTime = 0.2f;
        BlendFunction = ELyraCameraModeBlendFunction::EaseOut;
        BlendExponent = 2.0f;

        // 设置类型标签
        CameraTypeTag = FGameplayTag::RequestGameplayTag(TEXT("Camera.Mode.FirstPerson"));
    }

protected:
    // 第一人称不需要额外偏移，直接使用 Pivot（眼睛位置）
    virtual void UpdateView(float DeltaTime) override
    {
        FVector PivotLocation = GetPivotLocation();
        FRotator PivotRotation = GetPivotRotation();

        PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

        View.Location = PivotLocation;
        View.Rotation = PivotRotation;
        View.ControlRotation = View.Rotation;
        View.FieldOfView = FieldOfView;
    }
};
```

**在蓝图中配置**：

创建一个 `BP_CameraMode_FirstPerson`，继承自 `ULyraCameraMode_FirstPerson`，然后在武器装备时切换：

```cpp
// 在武器的 Equip Ability 中
void ULyraGameplayAbility_RangedWeapon::ActivateAbility(...)
{
    // 如果是狙击枪，切换到第一人称
    if (WeaponData->bRequiresFirstPersonMode)
    {
        if (ULyraCameraComponent* CameraComp = ULyraCameraComponent::FindCameraComponent(GetAvatarActorFromActorInfo()))
        {
            CameraComp->DetermineCameraModeDelegate.BindLambda([this]()
            {
                return FirstPersonCameraMode;
            });
        }
    }
}
```

### 6.2 案例 2：死亡相机（第三人称旋转观察）

```cpp
// LyraCameraMode_Death.h
#pragma once

#include "Camera/LyraCameraMode.h"
#include "LyraCameraMode_Death.generated.h"

UCLASS(Blueprintable)
class ULyraCameraMode_Death : public ULyraCameraMode
{
    GENERATED_BODY()

public:
    ULyraCameraMode_Death()
    {
        FieldOfView = 80.0f;
        BlendTime = 1.0f;  // 慢速混合（营造死亡的沉重感）
        BlendFunction = ELyraCameraModeBlendFunction::EaseInOut;
        BlendExponent = 3.0f;

        CameraTypeTag = FGameplayTag::RequestGameplayTag(TEXT("Camera.Mode.Death"));

        // 相机距离和高度
        DeathCameraDistance = 300.0f;
        DeathCameraHeight = 100.0f;

        // 旋转速度（度/秒）
        RotationSpeed = 30.0f;
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Death Camera")
    float DeathCameraDistance;

    UPROPERTY(EditDefaultsOnly, Category = "Death Camera")
    float DeathCameraHeight;

    UPROPERTY(EditDefaultsOnly, Category = "Death Camera")
    float RotationSpeed;

    // 累积的旋转角度
    float AccumulatedYaw = 0.0f;

    virtual void OnActivation() override
    {
        Super::OnActivation();
        AccumulatedYaw = 0.0f;
    }

    virtual void UpdateView(float DeltaTime) override
    {
        // 1. 相机围绕尸体旋转
        AccumulatedYaw += RotationSpeed * DeltaTime;

        FVector PivotLocation = GetPivotLocation();

        // 2. 计算相机位置（圆形轨迹）
        FVector Offset = FVector(
            FMath::Cos(FMath::DegreesToRadians(AccumulatedYaw)) * DeathCameraDistance,
            FMath::Sin(FMath::DegreesToRadians(AccumulatedYaw)) * DeathCameraDistance,
            DeathCameraHeight
        );

        View.Location = PivotLocation + Offset;

        // 3. 相机始终看向尸体
        FVector DirectionToTarget = (PivotLocation - View.Location).GetSafeNormal();
        View.Rotation = DirectionToTarget.Rotation();

        View.ControlRotation = View.Rotation;
        View.FieldOfView = FieldOfView;
    }
};
```

**触发逻辑**：

```cpp
// 在 Death Ability 中
void ULyraGameplayAbility_Death::ActivateAbility(...)
{
    // 1. 播放死亡动画
    PlayMontage(DeathMontage);

    // 2. 切换相机模式
    if (ULyraCameraComponent* CameraComp = ULyraCameraComponent::FindCameraComponent(GetAvatarActorFromActorInfo()))
    {
        CameraComp->DetermineCameraModeDelegate.BindLambda([this]()
        {
            return DeathCameraMode;  // ULyraCameraMode_Death 类
        });
    }

    // 3. 禁用角色输入
    if (APawn* Pawn = GetAvatarActorFromActorInfo<APawn>())
    {
        Pawn->DisableInput(nullptr);
    }
}
```

**效果**：
- 角色死亡后，相机自动切换到死亡模式
- 相机围绕尸体以 30°/秒的速度旋转
- 1 秒内平滑过渡到新视角

### 6.3 案例 3：动态 FOV（冲刺时拉伸）

```cpp
// LyraGameplayAbility_Sprint.cpp
void ULyraGameplayAbility_Sprint::ActivateAbility(...)
{
    Super::ActivateAbility(...);

    // 冲刺时增加 FOV（营造速度感）
    if (ULyraCameraComponent* CameraComp = ULyraCameraComponent::FindCameraComponent(GetAvatarActorFromActorInfo()))
    {
        // 每帧增加 10 度 FOV
        CameraComp->AddFieldOfViewOffset(10.0f);
    }
}

void ULyraGameplayAbility_Sprint::EndAbility(...)
{
    // 停止冲刺时恢复 FOV
    if (ULyraCameraComponent* CameraComp = ULyraCameraComponent::FindCameraComponent(GetAvatarActorFromActorInfo()))
    {
        CameraComp->AddFieldOfViewOffset(-10.0f);
    }

    Super::EndAbility(...);
}
```

**注意**：`AddFieldOfViewOffset` 是单帧有效的，所以需要在 Tick 中持续调用：

```cpp
void ULyraGameplayAbility_Sprint::TickAbility(float DeltaTime)
{
    Super::TickAbility(DeltaTime);

    // 持续应用 FOV 偏移
    if (ULyraCameraComponent* CameraComp = ULyraCameraComponent::FindCameraComponent(GetAvatarActorFromActorInfo()))
    {
        CameraComp->AddFieldOfViewOffset(SprintFOVOffset);  // 例如 +15
    }
}
```

### 6.4 案例 4：瞄准模式（ADS Camera）

```cpp
// LyraCameraMode_ADS.h
#pragma once

#include "Camera/LyraCameraMode_ThirdPerson.h"
#include "LyraCameraMode_ADS.generated.h"

UCLASS(Blueprintable)
class ULyraCameraMode_ADS : public ULyraCameraMode_ThirdPerson
{
    GENERATED_BODY()

public:
    ULyraCameraMode_ADS()
    {
        // 瞄准时缩小 FOV
        FieldOfView = 50.0f;

        // 快速混合
        BlendTime = 0.3f;
        BlendFunction = ELyraCameraModeBlendFunction::EaseOut;
        BlendExponent = 3.0f;

        CameraTypeTag = FGameplayTag::RequestGameplayTag(TEXT("Camera.Mode.ADS"));

        // 瞄准时相机拉近
        AimZoomDistance = 100.0f;
    }

protected:
    UPROPERTY(EditDefaultsOnly, Category = "ADS")
    float AimZoomDistance;

    virtual void UpdateView(float DeltaTime) override
    {
        // 1. 调用父类（第三人称）逻辑
        Super::UpdateView(DeltaTime);

        // 2. 在父类计算的位置基础上，向前拉近
        FVector ForwardDirection = View.Rotation.Vector();
        View.Location += ForwardDirection * AimZoomDistance;
    }
};
```

**配合 Gameplay Ability 使用**：

```cpp
// 在 Aim Ability 中
void ULyraGameplayAbility_Aim::ActivateAbility(...)
{
    // 1. 添加 Aim Tag（触发相机模式切换）
    if (UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo())
    {
        ASC->AddLooseGameplayTag(FGameplayTag::RequestGameplayTag(TEXT("Gameplay.State.Aiming")));
    }

    // 2. 在 DetermineCameraMode 中检测到 Aiming Tag，返回 ADS CameraMode
}

// ULyraHeroComponent::DetermineCameraMode()
TSubclassOf<ULyraCameraMode> ULyraHeroComponent::DetermineCameraMode() const
{
    if (AbilitySystemComponent->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag(TEXT("Gameplay.State.Aiming"))))
    {
        return AimingCameraMode;  // ULyraCameraMode_ADS
    }

    return DefaultCameraMode;  // ULyraCameraMode_ThirdPerson
}
```

---

## 7. 调试与优化技巧

### 7.1 开启相机 Debug 显示

```cpp
// 控制台命令
showdebug camera
```

显示内容：
```
LyraCameraComponent: BP_LyraCharacter_C_0
   Location: (1234.56, 7890.12, 345.67)
   Rotation: (Pitch=-12.34, Yaw=56.78, Roll=0.00)
   FOV: 80.0
   --- Camera Modes (Begin) ---
      LyraCameraMode_ThirdPerson: BP_CameraMode_ThirdPerson (1.000000)
   --- Camera Modes (End) ---
```

### 7.2 可视化 Feeler 射线

```cpp
// 在 LyraCameraMode_ThirdPerson.cpp 的 PreventCameraPenetration 中添加
void ULyraCameraMode_ThirdPerson::PreventCameraPenetration(...)
{
    // ...

    for (int32 i = 0; i < PenetrationAvoidanceFeelers.Num(); ++i)
    {
        // ...

        if (bHit && Hit.bBlockingHit)
        {
            DrawDebugLine(
                GetWorld(),
                SafeLoc,
                Hit.Location,
                FColor::Red,
                false,
                0.1f,
                0,
                2.0f
            );

            DrawDebugSphere(
                GetWorld(),
                Hit.Location,
                Feeler.Radius,
                12,
                FColor::Orange,
                false,
                0.1f
            );
        }
        else
        {
            DrawDebugLine(
                GetWorld(),
                SafeLoc,
                EndLocation,
                FColor::Green,
                false,
                0.1f,
                0,
                1.0f
            );
        }
    }
}
```

### 7.3 性能分析

```cpp
// 控制台命令
stat camera
```

关注指标：
- **Camera Update Time**：每帧相机更新耗时（应 < 0.5ms）
- **Penetration Sweep Count**：穿透检测射线数量
- **Mode Blend Count**：当前混合的模式数量

**优化建议**：
1. **减少 Feeler 数量**：如果性能紧张，可以只保留核心的 3-5 条射线
2. **使用 CollisionChannel**：确保 `ECC_Camera` 只与必要的物体碰撞（排除小物件、特效等）
3. **限制模式栈深度**：一般情况下栈中最多 2-3 个模式即可，避免复杂的多层混合

### 7.4 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 相机切换时跳变 | BlendTime 为 0 | 增加 BlendTime（建议 0.3-0.5s） |
| 相机穿墙 | bPreventPenetration 未开启 | 检查第三人称模式的配置 |
| 相机抖动 | Feeler 半径过小 | 增加 Feeler.Radius（建议 10-20） |
| 输入和视角不一致 | ControlRotation 未同步 | 检查 GetCameraView 中的同步逻辑 |
| 死亡后相机不切换 | Delegate 未更新 | 确保在 Death Ability 中重新绑定 |

---

## 8. 与其他系统的集成

### 8.1 与 Enhanced Input 联动

```cpp
// 在 InputConfig 中绑定相机控制
void ULyraHeroComponent::Input_Look(const FInputActionValue& InputActionValue)
{
    const FVector2D Value = InputActionValue.Get<FVector2D>();

    if (APawn* Pawn = GetPawn<APawn>())
    {
        // 添加控制旋转（会被 CameraComponent 同步到 View.ControlRotation）
        Pawn->AddControllerYawInput(Value.X);
        Pawn->AddControllerPitchInput(Value.Y);
    }
}
```

**Modifier 配置**（鼠标灵敏度）：

在 `IA_Look` Input Action 的 Modifiers 中添加：
- **Scale**：(X=0.5, Y=0.5)（降低灵敏度）
- **Negate**：Y 轴（反转 Y 轴，匹配习惯）

### 8.2 与 GAS 联动

**通过 Tag 查询相机状态**：

```cpp
// 在某个 Ability 中
void ULyraGameplayAbility_MyAbility::ActivateAbility(...)
{
    if (ULyraCameraComponent* CameraComp = ULyraCameraComponent::FindCameraComponent(GetAvatarActorFromActorInfo()))
    {
        float TopWeight;
        FGameplayTag TopTag;
        CameraComp->GetBlendInfo(TopWeight, TopTag);

        if (TopTag.MatchesTagExact(FGameplayTag::RequestGameplayTag(TEXT("Camera.Mode.ADS"))))
        {
            // 当前是瞄准模式，应用精准度加成
            ApplyAccuracyBonus();
        }
    }
}
```

**通过 GameplayEffect 修改 FOV**：

```cpp
// 创建一个 GE_SprintFOV
UCLASS()
class UGameplayEffect_SprintFOV : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UGameplayEffect_SprintFOV()
    {
        DurationPolicy = EGameplayEffectDurationType::Infinite;

        // 添加一个 Modifier（需要自定义 Attribute Set）
        FGameplayModifierInfo Modifier;
        Modifier.Attribute = ULyraCameraAttributeSet::GetFOVOffsetAttribute();
        Modifier.ModifierOp = EGameplayModOp::Additive;
        Modifier.ModifierMagnitude = FScalableFloat(15.0f);  // +15 FOV

        Modifiers.Add(Modifier);
    }
};
```

然后在 `LyraCameraComponent::GetCameraView` 中读取：

```cpp
void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)
{
    // ...

    // 读取 GAS 中的 FOV Offset Attribute
    if (UAbilitySystemComponent* ASC = GetOwner()->FindComponentByClass<UAbilitySystemComponent>())
    {
        if (const ULyraCameraAttributeSet* CameraAttrs = ASC->GetSet<ULyraCameraAttributeSet>())
        {
            CameraModeView.FieldOfView += CameraAttrs->GetFOVOffset();
        }
    }

    // ...
}
```

### 8.3 与 Common UI 联动

**UI 显示当前相机模式**：

```cpp
// 在 HUD Widget 中
UCLASS()
class ULyraHUDWidget : public UCommonUserWidget
{
    GENERATED_BODY()

protected:
    UPROPERTY(meta = (BindWidget))
    UTextBlock* CameraModeText;

    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override
    {
        Super::NativeTick(MyGeometry, InDeltaTime);

        if (APlayerController* PC = GetOwningPlayer())
        {
            if (APawn* Pawn = PC->GetPawn())
            {
                if (ULyraCameraComponent* CameraComp = Pawn->FindComponentByClass<ULyraCameraComponent>())
                {
                    float Weight;
                    FGameplayTag ModeTag;
                    CameraComp->GetBlendInfo(Weight, ModeTag);

                    CameraModeText->SetText(FText::FromString(ModeTag.ToString()));
                }
            }
        }
    }
};
```

---

## 9. 最佳实践与设计建议

### 9.1 相机模式设计原则

| 原则 | 说明 |
|------|------|
| **单一职责** | 每个 CameraMode 只负责一种视角逻辑（不要在第三人称模式中硬塞瞄准逻辑） |
| **数据驱动** | 使用 Curve、Data Asset 配置参数，而不是硬编码 |
| **平滑过渡** | 所有模式切换都应有 BlendTime（除非特殊需求） |
| **性能优先** | 穿透避免的 Feeler 数量应控制在 5-10 条 |
| **调试友好** | 实现 DrawDebug，方便可视化问题 |

### 9.2 Curve 配置建议

**第三人称 TargetOffsetCurve**：

| 场景 | Pitch 范围 | Offset X | Offset Y | Offset Z |
|------|-----------|----------|----------|----------|
| 平台跳跃 | -60° ~ +60° | -250 | 50 | 30 |
| 开放世界探索 | -45° ~ +45° | -300 | 60 | 40 |
| 近战战斗 | -30° ~ +30° | -200 | 40 | 20 |
| 室内场景 | -20° ~ +20° | -150 | 30 | 10 |

**BlendTime 建议**：

| 模式切换 | BlendTime | BlendFunction |
|---------|-----------|---------------|
| 第三人称 ↔ 瞄准 | 0.3s | EaseOut |
| 第三人称 ↔ 死亡 | 1.0s | EaseInOut |
| 瞄准 ↔ 开镜 | 0.2s | Linear |
| 第三人称 ↔ 第一人称 | 0.5s | EaseInOut |

### 9.3 穿透避免优化

**Feeler 优先级策略**：

1. **核心 Feeler（优先级 0）**：正前方，必须保证无遮挡
2. **侧面 Feeler（优先级 3）**：检测墙角
3. **预测性 Feeler（优先级 5）**：提前检测潜在碰撞（性能允许时开启）

**CollisionChannel 设置**：

```cpp
// 在 DefaultEngine.ini 中
[/Script/Engine.CollisionProfile]
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel1,DefaultResponse=ECR_Block,bTraceType=True,bStaticObject=True,Name="Camera")

// 设置哪些物体阻挡相机
+EditProfiles=(Name="WorldStatic",CustomResponses=((Channel="Camera",Response=ECR_Block)))
+EditProfiles=(Name="Character",CustomResponses=((Channel="Camera",Response=ECR_Ignore)))
+EditProfiles=(Name="PhysicsBody",CustomResponses=((Channel="Camera",Response=ECR_Ignore)))
```

---

## 10. 总结与扩展阅读

### 10.1 核心要点回顾

✅ **LyraCameraComponent** 是协调器，通过 Delegate 查询当前应该使用的 CameraMode  
✅ **CameraModeStack** 管理多模式混合，支持平滑过渡，避免视角跳变  
✅ **第三人称相机** 包含 Curve 偏移、下蹲补偿、穿透避免三大核心功能  
✅ **自定义相机模式** 只需继承 `ULyraCameraMode`，覆盖 `UpdateView` 即可  
✅ **调试工具**：`showdebug camera`、DrawDebug、可视化 Feeler  

### 10.2 实战项目检查清单

- [ ] 为游戏模式配置默认 CameraMode（第三人称/第一人称/俯视角）
- [ ] 实现瞄准相机模式（缩小 FOV、拉近距离）
- [ ] 配置 TargetOffsetCurve（根据 Pitch 调整偏移）
- [ ] 测试穿透避免（走到墙角、钻进狭窄空间）
- [ ] 实现死亡相机（自动旋转观察）
- [ ] 集成 GAS Tag 查询（根据状态切换相机）
- [ ] 优化 CollisionChannel（排除不必要的碰撞检测）
- [ ] 添加 Debug 可视化（Feeler 射线、模式权重）

### 10.3 扩展阅读

| 主题 | 推荐资源 |
|------|---------|
| **UE 相机系统官方文档** | [Cameras in Unreal Engine](https://docs.unrealengine.com/5.0/en-US/cameras-in-unreal-engine/) |
| **Lyra 相机源码** | `Source/LyraGame/Camera/*` |
| **第三人称相机最佳实践** | GDC Talk: "The Perfect Camera" |
| **穿透避免算法** | [Camera Collision in Third Person Games](https://www.gamasutra.com/view/feature/131801/camera_collision_in_thirdperson_.php) |
| **混合函数曲线** | [Easing Functions Cheat Sheet](https://easings.net/) |

### 10.4 下一步学习

完成本章后，你已经掌握了 Lyra 相机系统的核心原理。接下来可以学习：

1. **动画系统**（第 23 章）：相机与动画的联动（如蒙太奇相机、动画通知触发相机抖动）
2. **音频系统**（第 24 章）：相机位置作为 Audio Listener，影响 3D 声音定位
3. **Replay 系统**（第 26 章）：自由相机模式、慢动作回放、多角度切换

---

## 📝 实战作业

### 作业 1：实现观战相机

**需求**：
- 玩家死亡后，自动切换到队友视角
- 按 Tab 键可以切换观战目标
- 显示当前观战的玩家名称

**提示**：
1. 创建 `ULyraCameraMode_Spectator`
2. 在 `UpdateView` 中获取观战目标的 Pivot
3. 监听 Tab 键输入，切换 `SpectatorTarget`

### 作业 2：实现过肩切换

**需求**：
- 按 Q 键切换左肩/右肩视角
- 切换时平滑过渡（0.3 秒）

**提示**：
1. 在 `ULyraCameraMode_ThirdPerson` 中添加 `bIsRightShoulder` 变量
2. 在 Curve 中配置左肩偏移（Y = -50）和右肩偏移（Y = +50）
3. 切换时重新推入 CameraMode（触发混合）

### 作业 3：实现动态 FOV 效果

**需求**：
- 受到伤害时，FOV 瞬间缩小 5 度（0.1 秒恢复）
- 使用治疗包时，FOV 瞬间放大 5 度（营造"清醒"效果）

**提示**：
1. 在伤害 GameplayEffect 中触发事件
2. 监听事件，调用 `AddFieldOfViewOffset`
3. 使用 Timer 在 0.1 秒后恢复

---

**🎓 恭喜！你已经掌握了 Lyra 的相机系统！**  
现在你可以为游戏打造流畅、自适应、沉浸感十足的视角体验了！🚀📹
