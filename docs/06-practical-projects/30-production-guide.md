# 从 Lyra 到商业游戏：完整的产品化指南

## 概述

Lyra 是 Epic Games 提供的生产级游戏示例项目，展示了现代 AAA 游戏的最佳实践。然而，将 Lyra 从示例项目转化为可商业发布的产品需要系统化的改造、优化和规划。本文将深入探讨如何将 Lyra 项目成功转化为商业游戏产品，涵盖技术架构、团队协作、发布流程、运营策略等各个方面。

本文适合以下读者：
- 准备基于 Lyra 开发商业游戏的团队
- 需要了解 UE5 项目产品化流程的制作人和技术总监
- 希望学习现代游戏开发全流程的工程师

**目标读者前置知识**：
- 熟悉 Lyra 项目基本架构
- 具备 UE5 项目开发经验
- 了解游戏开发基本流程

---

## 第一部分：商业化改造策略与规划

### 1.1 从示例到产品的思维转变

Lyra 作为示例项目，其设计目标是**展示技术能力**和**提供学习模板**，而商业游戏的核心目标是**用户体验**和**商业价值**。这种思维转变需要在以下方面进行调整：

#### 核心差异对比

| 维度 | Lyra 示例项目 | 商业游戏产品 |
|------|--------------|------------|
| **目标** | 展示 UE5 能力，教学示范 | 用户留存，商业变现 |
| **范围** | 功能广度，多种玩法 | 聚焦核心玩法，深度打磨 |
| **性能** | 展示高端硬件效果 | 多平台兼容，性能优化 |
| **内容** | 示例内容，演示用途 | 大量原创内容，持续更新 |
| **稳定性** | 开发版本，允许 bug | 严格测试，高稳定性要求 |
| **用户体验** | 面向开发者 | 面向普通玩家 |

#### 战略规划框架

**第一阶段：评估与定义（1-2周）**

1. **明确游戏类型和目标市场**
   - 确定核心玩法类型（射击、MOBA、大逃杀等）
   - 定义目标用户画像和市场定位
   - 分析竞品，找到差异化点

2. **技术架构评估**
   ```cpp
   // 创建项目评估清单类
   UCLASS(Config=Game)
   class UProjectAssessmentConfig : public UDeveloperSettings
   {
       GENERATED_BODY()
   
   public:
       // 保留的 Lyra 系统
       UPROPERTY(Config, EditAnywhere, Category="Systems")
       TArray<FString> RetainedSystems;
       
       // 需要移除的系统
       UPROPERTY(Config, EditAnywhere, Category="Systems")
       TArray<FString> SystemsToRemove;
       
       // 需要定制开发的系统
       UPROPERTY(Config, EditAnywhere, Category="Systems")
       TArray<FString> CustomSystems;
       
       // 目标平台
       UPROPERTY(Config, EditAnywhere, Category="Platforms")
       TArray<FString> TargetPlatforms;
   };
   ```

3. **资源盘点**
   - 评估现有团队技能结构
   - 确定外包需求（美术、音频、本地化等）
   - 制定预算和时间表

**第二阶段：架构改造（2-4周）**

1. **重命名和品牌化**
   ```bash
   # 项目重命名脚本示例
   # RenameProject.sh
   
   OLD_NAME="Lyra"
   NEW_NAME="YourGameName"
   
   # 重命名模块
   mv Source/LyraGame Source/${NEW_NAME}Game
   mv Source/LyraEditor Source/${NEW_NAME}Editor
   
   # 批量替换文件内容
   find . -type f \( -name "*.cpp" -o -name "*.h" -o -name "*.ini" \) \
       -exec sed -i "s/Lyra/${NEW_NAME}/g" {} +
   
   # 重命名 .uproject 文件
   mv Lyra.uproject ${NEW_NAME}.uproject
   ```

2. **模块化拆分策略**
   ```cpp
   // 定义游戏特定的模块
   // Source/YourGame/YourGame.Build.cs
   
   public class YourGame : ModuleRules
   {
       public YourGame(ReadOnlyTargetRules Target) : base(Target)
       {
           PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
   
           // 核心依赖
           PublicDependencyModuleNames.AddRange(new string[] {
               "Core",
               "CoreUObject",
               "Engine",
               "ModularGameplay",
               "GameplayAbilities",
               "GameplayTags",
               "GameFeatures",
               "CommonGame"  // 保留通用系统
           });
   
           // 移除不需要的 Lyra 特定模块
           // PrivateDependencyModuleNames.Remove("LyraExampleContent");
           
           // 添加自定义模块
           PrivateDependencyModuleNames.AddRange(new string[] {
               "YourGameCore",
               "YourGameUI",
               "YourGameMonetization"  // 商业化系统
           });
       }
   }
   ```

**第三阶段：核心功能迁移（4-8周）**

根据游戏类型，选择性保留和改造 Lyra 系统。

### 1.2 商业化可行性分析

#### 市场评估模型

**TAM/SAM/SOM 分析**

```cpp
// 市场数据跟踪系统
USTRUCT(BlueprintType)
struct FMarketAnalysisData
{
    GENERATED_BODY()
    
    // Total Addressable Market - 总目标市场
    UPROPERTY(BlueprintReadWrite)
    int64 TAM_PlayerCount;
    
    // Serviceable Addressable Market - 可服务市场
    UPROPERTY(BlueprintReadWrite)
    int64 SAM_PlayerCount;
    
    // Serviceable Obtainable Market - 可获得市场
    UPROPERTY(BlueprintReadWrite)
    int64 SOM_PlayerCount;
    
    UPROPERTY(BlueprintReadWrite)
    float EstimatedARPU;  // Average Revenue Per User
    
    UPROPERTY(BlueprintReadWrite)
    float EstimatedConversionRate;
};
```

#### 竞品分析框架

**功能对比矩阵**

创建详细的竞品对比表：

```
竞品 A（如 Fortnite）：
- 核心玩法：大逃杀 + 建造
- 变现模式：Battle Pass + 皮肤商城
- 技术特点：UE4/5, 跨平台，快速更新
- 用户规模：数千万 DAU

竞品 B（如 Apex Legends）：
- 核心玩法：大逃杀 + 英雄能力
- 变现模式：季票 + 限时活动
- 技术特点：Source 引擎，60Hz 服务器
- 用户规模：数百万 DAU

你的游戏定位：
- 核心玩法：[基于 Lyra 的定制玩法]
- 差异化点：[独特卖点]
- 目标用户：[细分市场]
```

### 1.3 技术债务评估

Lyra 作为示例项目，存在一些需要在商业化过程中解决的技术问题：

#### 常见技术债务清单

1. **过度工程化**
   - Lyra 的某些系统设计得过于通用，可能影响性能
   - 需要根据实际需求简化架构

2. **示例内容依赖**
   - 大量 Lyra 特定的资源和配置
   - 需要系统性替换为项目专用内容

3. **文档和注释**
   - Lyra 的某些系统缺少详细文档
   - 需要建立完整的技术文档体系

**技术债务评估代码**

```cpp
// 技术债务跟踪系统
UCLASS()
class UTechnicalDebtTracker : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    USTRUCT()
    struct FTechDebtItem
    {
        GENERATED_BODY()
        
        UPROPERTY()
        FString Category;  // "Performance", "Architecture", "Content"
        
        UPROPERTY()
        FString Description;
        
        UPROPERTY()
        int32 Priority;  // 1-5
        
        UPROPERTY()
        float EstimatedHours;
        
        UPROPERTY()
        TArray<FString> AffectedSystems;
    };
    
    UPROPERTY()
    TArray<FTechDebtItem> DebtItems;
    
    // 生成技术债务报告
    UFUNCTION(BlueprintCallable)
    void GenerateDebtReport();
    
    // 按优先级排序
    UFUNCTION(BlueprintCallable)
    TArray<FTechDebtItem> GetPrioritizedDebt();
};
```

---

## 第二部分：系统裁剪与定制化

### 2.1 Lyra 系统模块分析

Lyra 包含大量系统，需要根据项目需求进行取舍。以下是核心系统的分析和建议：

#### 必须保留的核心系统

| 系统 | 作用 | 保留理由 |
|------|------|---------|
| **ModularGameplay** | 模块化游戏框架 | 提供灵活的组件化架构，是 Lyra 的基石 |
| **GameplayAbilities (GAS)** | 技能系统 | 行业标准的能力系统，强大且可扩展 |
| **GameFeatures** | 游戏特性插件 | 支持模块化开发和 DLC，现代游戏必备 |
| **CommonUI** | 通用 UI 框架 | 跨平台 UI 解决方案，支持多输入方式 |
| **Enhanced Input** | 增强输入系统 | UE5 标准输入方案，支持复杂输入映射 |
| **GameplayTags** | 游戏标签系统 | 数据驱动的标签系统，避免硬编码 |

#### 可选保留的系统

| 系统 | 作用 | 取舍建议 |
|------|------|---------|
| **Lyra 经验系统** | 角色升级和进度 | 如果游戏包含 RPG 元素则保留，否则移除 |
| **Lyra 装备系统** | 武器和装备管理 | 射击游戏必备，其他类型可简化 |
| **队伍系统** | 团队和阵营管理 | 多人游戏必需，单机游戏可移除 |
| **Lyra 库存系统** | 物品和背包 | 根据游戏类型决定 |
| **Lyra 设置系统** | 游戏设置管理 | 可保留框架，定制化设置项 |

#### 可以移除的示例内容

```cpp
// 移除 LyraExampleContent 插件
// 在 YourGame.uproject 中移除：
{
    "Name": "LyraExampleContent",
    "Enabled": false  // 或直接删除此条目
}

// 移除相关的 C++ 依赖
// YourGame.Build.cs
public class YourGame : ModuleRules
{
    public YourGame(ReadOnlyTargetRules Target) : base(Target)
    {
        // 注释或移除示例内容的依赖
        // PrivateDependencyModuleNames.Add("LyraExampleContent");
    }
}
```

### 2.2 移除不需要系统的步骤

#### Step 1: 依赖分析

```bash
# 使用 UE5 的依赖分析工具
# 在项目目录运行

# 分析模块依赖
UnrealBuildTool.exe -Mode=QueryTargets -Project="YourGame.uproject"

# 生成依赖图
dot -Tpng ModuleDependencies.dot -o Dependencies.png
```

#### Step 2: 代码清理脚本

```python
# CleanupLyraCode.py
# 移除 Lyra 特定的代码引用

import os
import re

LYRA_SPECIFIC_CLASSES = [
    "ULyraExperienceDefinition",
    "ULyraExperienceManagerComponent",
    "ALyraCharacter",
    "ULyraHeroComponent",
    # 添加更多需要移除的类
]

def clean_cpp_file(file_path):
    with open(file_path, 'r', encoding='utf-8') as f:
        content = f.read()
    
    original_content = content
    
    # 移除特定的 include
    for class_name in LYRA_SPECIFIC_CLASSES:
        header_name = class_name[1:] + ".h"  # UClassName -> ClassName.h
        pattern = f'#include.*{header_name}'
        content = re.sub(pattern, f'// REMOVED: {pattern}', content)
    
    # 移除对 Lyra 类的引用（注释掉）
    for class_name in LYRA_SPECIFIC_CLASSES:
        content = re.sub(
            f'\\b{class_name}\\b',
            f'/* REMOVED: {class_name} */',
            content
        )
    
    if content != original_content:
        with open(file_path, 'w', encoding='utf-8') as f:
            f.write(content)
        print(f"Cleaned: {file_path}")

# 遍历源代码目录
for root, dirs, files in os.walk("Source/YourGame"):
    for file in files:
        if file.endswith(('.cpp', '.h')):
            clean_cpp_file(os.path.join(root, file))
```

#### Step 3: 配置清理

```ini
; Config/DefaultGame.ini
; 移除 Lyra 特定的配置

[/Script/LyraGame.LyraExperienceManagerComponent]
; 删除或注释掉整个 section

[/Script/LyraGame.LyraSettingsLocal]
; 替换为你的设置类
; [/Script/YourGame.YourGameSettingsLocal]
```

### 2.3 创建简化的游戏模式

#### 从 Lyra 到简化版本

Lyra 的 Experience 系统非常强大但复杂，对于许多项目来说可能过度设计。以下是简化方案：

**简化的 GameMode**

```cpp
// YourGameMode.h
#pragma once

#include "CoreMinimal.h"
#include "ModularGameMode.h"
#include "YourGameMode.generated.h"

/**
 * 简化的 GameMode，保留 Lyra 的模块化优势，但移除 Experience 复杂性
 */
UCLASS()
class YOURGAME_API AYourGameMode : public AModularGameModeBase
{
    GENERATED_BODY()

public:
    AYourGameMode(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    // 游戏规则配置（取代 Experience）
    UPROPERTY(EditDefaultsOnly, Category="Game Rules")
    TSubclassOf<UGameRulesAsset> GameRulesClass;

protected:
    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;
    virtual void InitGameState() override;
    virtual void HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer) override;

    // 加载游戏规则
    UFUNCTION()
    void LoadGameRules();

private:
    UPROPERTY()
    TObjectPtr<UGameRulesAsset> GameRules;
};

// YourGameMode.cpp
#include "YourGameMode.h"
#include "YourGameState.h"
#include "YourPlayerController.h"
#include "YourPlayerState.h"
#include "YourPawn.h"

AYourGameMode::AYourGameMode(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    GameStateClass = AYourGameState::StaticClass();
    PlayerControllerClass = AYourPlayerController::StaticClass();
    PlayerStateClass = AYourPlayerState::StaticClass();
    DefaultPawnClass = AYourPawn::StaticClass();
}

void AYourGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);
    
    LoadGameRules();
}

void AYourGameMode::LoadGameRules()
{
    if (GameRulesClass)
    {
        GameRules = NewObject<UGameRulesAsset>(this, GameRulesClass);
        GameRules->ApplyRules(this);
    }
}
```

**游戏规则资产（取代 Experience）**

```cpp
// GameRulesAsset.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "GameRulesAsset.generated.h"

/**
 * 游戏规则配置，简化版的 Experience
 */
UCLASS(BlueprintType)
class YOURGAME_API UGameRulesAsset : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 游戏玩法组件（保留 Lyra 的组件化思想）
    UPROPERTY(EditDefaultsOnly, Category="Components")
    TArray<TSubclassOf<UGameFrameworkComponent>> GameStateComponents;

    UPROPERTY(EditDefaultsOnly, Category="Components")
    TArray<TSubclassOf<UGameFrameworkComponent>> PlayerStateComponents;

    // 输入配置
    UPROPERTY(EditDefaultsOnly, Category="Input")
    TObjectPtr<class UInputMappingContext> InputMapping;

    // UI 布局
    UPROPERTY(EditDefaultsOnly, Category="UI")
    TSubclassOf<class UCommonActivatableWidget> HUDLayout;

    // 游戏规则参数
    UPROPERTY(EditDefaultsOnly, Category="Rules")
    int32 MaxPlayers;

    UPROPERTY(EditDefaultsOnly, Category="Rules")
    float RoundDuration;

    UPROPERTY(EditDefaultsOnly, Category="Rules")
    int32 ScoreToWin;

    // 应用规则到 GameMode
    void ApplyRules(class AYourGameMode* GameMode);
};

// GameRulesAsset.cpp
void UGameRulesAsset::ApplyRules(AYourGameMode* GameMode)
{
    if (!GameMode) return;

    // 添加 GameState 组件
    AYourGameState* GameState = GameMode->GetGameState<AYourGameState>();
    if (GameState)
    {
        for (TSubclassOf<UGameFrameworkComponent> ComponentClass : GameStateComponents)
        {
            if (ComponentClass)
            {
                GameState->AddComponent(ComponentClass);
            }
        }
    }

    // 应用游戏规则参数
    GameState->MaxPlayers = MaxPlayers;
    GameState->RoundDuration = RoundDuration;
    GameState->ScoreToWin = ScoreToWin;

    UE_LOG(LogTemp, Log, TEXT("Game Rules Applied: MaxPlayers=%d, RoundDuration=%.1f"), 
           MaxPlayers, RoundDuration);
}
```

---

## 第三部分：团队协作与版本管理

### 3.1 源代码管理策略

#### Git 工作流设计

**推荐工作流：GitFlow 变体**

```
main (master)
  ├── develop
  │    ├── feature/new-weapon-system
  │    ├── feature/new-map
  │    └── feature/ui-redesign
  ├── release/1.0
  ├── release/1.1
  └── hotfix/critical-bug
```

**分支策略**

```bash
# .git/config 或团队文档

# 主要分支
main        # 生产环境，只接受 release 和 hotfix 合并
develop     # 开发主分支，功能集成分支

# 支持分支
feature/*   # 新功能开发
release/*   # 版本发布准备
hotfix/*    # 紧急修复
bugfix/*    # 常规 bug 修复
```

#### .gitignore 配置

```gitignore
# UE5 项目的 .gitignore

# 构建目录
Binaries/
Build/
Intermediate/
DerivedDataCache/

# 用户配置
Saved/
*.sln
*.suo
*.xcodeproj
*.xcworkspace

# 插件的中间文件
Plugins/*/Binaries/
Plugins/*/Intermediate/

# 大文件资源（使用 Git LFS）
Content/**/*.uasset
Content/**/*.umap

# 但保留一些小型配置文件
!Content/Config/*.uasset

# 崩溃报告
*.log
*.dmp

# 备份文件
*.bak
*~
```

#### Git LFS 配置

```bash
# .gitattributes
# 大文件使用 Git LFS 管理

# UE5 资源文件
*.uasset filter=lfs diff=lfs merge=lfs -text
*.umap filter=lfs diff=lfs merge=lfs -text
*.upk filter=lfs diff=lfs merge=lfs -text

# 纹理和模型
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.tga filter=lfs diff=lfs merge=lfs -text
*.fbx filter=lfs diff=lfs merge=lfs -text
*.obj filter=lfs diff=lfs merge=lfs -text

# 音频
*.wav filter=lfs diff=lfs merge=lfs -text
*.mp3 filter=lfs diff=lfs merge=lfs -text

# 视频
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.mov filter=lfs diff=lfs merge=lfs -text
```

### 3.2 代码审查流程

#### Pull Request 模板

```markdown
## PR 描述
<!-- 简要描述这次改动的目的 -->

## 改动类型
- [ ] 新功能
- [ ] Bug 修复
- [ ] 性能优化
- [ ] 重构
- [ ] 文档更新

## 相关 Issue
<!-- 关联的 Issue 编号，例如 #123 -->

## 测试计划
- [ ] 单元测试通过
- [ ] 功能测试完成
- [ ] 性能测试（如适用）
- [ ] 多平台测试（PC/Console/Mobile）

## 检查清单
- [ ] 代码符合团队编码规范
- [ ] 添加了必要的注释
- [ ] 更新了相关文档
- [ ] 没有新的编译警告
- [ ] 通过了所有自动化测试

## 截图/视频
<!-- 如果是 UI 或游戏玩法改动，附上截图或视频 -->

## 性能影响
<!-- 如果有性能影响，说明影响范围和测试数据 -->
```

#### 代码审查规范

```cpp
// CodeReviewGuidelines.h
// 团队代码审查要点

/**
 * 代码审查检查项：
 * 
 * 1. 架构和设计
 *    - 是否符合项目的整体架构
 *    - 是否遵循 SOLID 原则
 *    - 是否有过度设计
 * 
 * 2. 代码质量
 *    - 命名是否清晰（变量、函数、类）
 *    - 函数是否简洁（单一职责）
 *    - 是否有重复代码
 * 
 * 3. 性能
 *    - 是否有不必要的循环或计算
 *    - 是否有内存泄漏风险
 *    - 是否有不必要的 Cast 操作
 * 
 * 4. UE5 最佳实践
 *    - 是否正确使用 UPROPERTY/UFUNCTION
 *    - 是否正确处理对象生命周期
 *    - 是否正确使用 TObjectPtr（UE5.1+）
 * 
 * 5. 多人游戏
 *    - 网络复制是否正确
 *    - 是否考虑了客户端预测
 *    - 是否有作弊风险
 */

// 示例：好的代码
UCLASS()
class AWeapon : public AActor
{
    GENERATED_BODY()

protected:
    // 清晰的命名和注释
    UPROPERTY(EditDefaultsOnly, Category="Weapon|Stats")
    float BaseDamage = 10.0f;

    // 使用 TObjectPtr（UE5.1+）
    UPROPERTY()
    TObjectPtr<USkeletalMeshComponent> WeaponMesh;

public:
    // 简洁的函数，单一职责
    UFUNCTION(BlueprintCallable, Category="Weapon")
    void Fire();

    // 网络复制正确标记
    UFUNCTION(Server, Reliable)
    void ServerFire();
};

// 示例：需要改进的代码
UCLASS()
class AWeapon2 : public AActor
{
    GENERATED_BODY()

protected:
    float d;  // 差的命名
    USkeletalMeshComponent* m;  // 未使用 TObjectPtr

public:
    void DoStuff();  // 不清晰的函数名
    
    // 函数过长，做了太多事情（假设函数体有 200 行）
    void FireAndReloadAndPlaySoundAndUpdateUI();
};
```

### 3.3 资源管理和命名规范

#### 资源命名约定

```
# 通用格式：PrefixType_AssetName_Variant_Suffix

# 纹理 (T_)
T_Character_Hero_D    # Diffuse
T_Character_Hero_N    # Normal
T_Character_Hero_M    # Mask
T_Character_Hero_E    # Emissive

# 材质 (M_)
M_Character_Hero
M_Environment_Rock

# 材质实例 (MI_)
MI_Character_Hero_Blue
MI_Character_Hero_Red

# 网格 (SM_ / SK_)
SM_Prop_Chair         # Static Mesh
SK_Character_Hero     # Skeletal Mesh

# 动画 (A_ / AM_ / ABS_)
A_Character_Run       # Animation
AM_Character          # Animation Montage
ABS_Character_Upper   # Animation Blend Space

# 蓝图 (BP_)
BP_Character_Hero
BP_Weapon_Rifle
BP_GameMode_Deathmatch

# UI (W_)
W_MainMenu
W_HUD_InGame
W_Popup_Confirm

# 声音 (S_ / SC_)
S_Weapon_Gunshot      # Sound Wave
SC_Character_Footsteps # Sound Cue

# 粒子 (P_)
P_Weapon_MuzzleFlash
P_Explosion_Small

# 数据表 (DT_)
DT_Weapon_Stats
DT_Character_XP
```

#### 目录结构

```
Content/
├── Core/                      # 核心系统
│   ├── GameModes/
│   ├── PlayerControllers/
│   └── GameStates/
├── Characters/                # 角色
│   ├── Hero/
│   │   ├── Meshes/
│   │   ├── Animations/
│   │   ├── Materials/
│   │   └── Blueprints/
│   └── Enemies/
├── Weapons/                   # 武器
│   ├── Rifles/
│   ├── Pistols/
│   └── Shared/               # 共享资源
├── Environments/             # 环境
│   ├── Maps/
│   ├── Props/
│   └── Materials/
├── UI/                       # 用户界面
│   ├── Menus/
│   ├── HUD/
│   └── Common/
├── Audio/                    # 音频
│   ├── Music/
│   ├── SFX/
│   └── Dialogue/
├── VFX/                      # 视觉特效
│   ├── Weapons/
│   ├── Abilities/
│   └── Environmental/
└── Data/                     # 数据资产
    ├── DataTables/
    ├── GameplayAbilities/
    └── GameplayTags/
```

#### 资源引用管理

```cpp
// AssetManager 配置
// DefaultGame.ini

[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="Map",AssetBaseClass=/Script/Engine.World,bHasBlueprintClasses=False,bIsEditorOnly=True,Directories=((Path="/Game/Maps")))
+PrimaryAssetTypesToScan=(PrimaryAssetType="WeaponDefinition",AssetBaseClass=/Script/YourGame.WeaponDefinition,bHasBlueprintClasses=True,Directories=((Path="/Game/Weapons")))
+PrimaryAssetTypesToScan=(PrimaryAssetType="CharacterDefinition",AssetBaseClass=/Script/YourGame.CharacterDefinition,bHasBlueprintClasses=True,Directories=((Path="/Game/Characters")))

bShouldManagerDetermineTypeAndName=True
bShouldGuessTypeAndNameInEditor=True
bShouldAcquireMissingChunksOnLoad=False
+PrimaryAssetRules=(PrimaryAssetId="Map:/Game/Maps/MainMenu",Rules=(Priority=1,ChunkId=0,CookRule=AlwaysCook))
+PrimaryAssetRules=(PrimaryAssetId="Map:/Game/Maps/Tutorial",Rules=(Priority=1,ChunkId=0,CookRule=AlwaysCook))
```

```cpp
// C++ 中的资源软引用
UCLASS()
class UWeaponDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 使用软引用避免不必要的资源加载
    UPROPERTY(EditDefaultsOnly, Category="Mesh")
    TSoftObjectPtr<USkeletalMesh> WeaponMesh;

    UPROPERTY(EditDefaultsOnly, Category="Effects")
    TSoftObjectPtr<UParticleSystem> MuzzleFlash;

    // 异步加载资源
    void LoadWeaponAssetsAsync(FSimpleDelegate OnLoadComplete);
};

void UWeaponDefinition::LoadWeaponAssetsAsync(FSimpleDelegate OnLoadComplete)
{
    TArray<FSoftObjectPath> AssetsToLoad;
    
    if (!WeaponMesh.IsNull())
    {
        AssetsToLoad.Add(WeaponMesh.ToSoftObjectPath());
    }
    
    if (!MuzzleFlash.IsNull())
    {
        AssetsToLoad.Add(MuzzleFlash.ToSoftObjectPath());
    }

    if (AssetsToLoad.Num() > 0)
    {
        UAssetManager& AssetManager = UAssetManager::Get();
        AssetManager.GetStreamableManager().RequestAsyncLoad(
            AssetsToLoad,
            OnLoadComplete
        );
    }
    else
    {
        OnLoadComplete.ExecuteIfBound();
    }
}
```

---

## 第四部分：CI/CD 流程搭建

### 4.1 自动化构建系统

#### Jenkins Pipeline 示例

```groovy
// Jenkinsfile
pipeline {
    agent { label 'ue5-build-server' }
    
    parameters {
        choice(name: 'TARGET_PLATFORM', choices: ['Win64', 'Linux', 'Android', 'iOS'], description: '目标平台')
        choice(name: 'BUILD_CONFIG', choices: ['Development', 'Shipping', 'DebugGame'], description: '构建配置')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: '运行自动化测试')
    }
    
    environment {
        UE5_ROOT = '/opt/UnrealEngine/UE_5.3'
        PROJECT_PATH = "${WORKSPACE}/YourGame.uproject"
        BUILD_OUTPUT = "${WORKSPACE}/Build/${params.TARGET_PLATFORM}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git lfs pull'
            }
        }
        
        stage('Generate Project Files') {
            steps {
                sh """
                    ${UE5_ROOT}/Engine/Build/BatchFiles/Linux/GenerateProjectFiles.sh \
                    -project="${PROJECT_PATH}" -game -engine
                """
            }
        }
        
        stage('Build Editor') {
            when {
                expression { params.BUILD_CONFIG != 'Shipping' }
            }
            steps {
                sh """
                    ${UE5_ROOT}/Engine/Build/BatchFiles/Linux/Build.sh \
                    YourGameEditor ${params.TARGET_PLATFORM} ${params.BUILD_CONFIG} \
                    -project="${PROJECT_PATH}" -progress
                """
            }
        }
        
        stage('Run Automation Tests') {
            when {
                expression { params.RUN_TESTS }
            }
            steps {
                sh """
                    ${UE5_ROOT}/Engine/Binaries/Linux/UnrealEditor-Cmd \
                    "${PROJECT_PATH}" \
                    -ExecCmds="Automation RunTests YourGame" \
                    -unattended -nopause -testexit="Automation Test Queue Empty" \
                    -log -ReportOutputPath="${WORKSPACE}/TestReports"
                """
            }
            post {
                always {
                    junit '**/TestReports/*.xml'
                }
            }
        }
        
        stage('Cook Content') {
            steps {
                sh """
                    ${UE5_ROOT}/Engine/Build/BatchFiles/RunUAT.sh BuildCookRun \
                    -project="${PROJECT_PATH}" \
                    -platform=${params.TARGET_PLATFORM} \
                    -clientconfig=${params.BUILD_CONFIG} \
                    -cook -stage -pak -archive \
                    -archivedirectory="${BUILD_OUTPUT}" \
                    -noP4 -utf8output
                """
            }
        }
        
        stage('Package') {
            steps {
                sh """
                    cd ${BUILD_OUTPUT}
                    zip -r YourGame_${params.TARGET_PLATFORM}_${params.BUILD_CONFIG}_${BUILD_NUMBER}.zip *
                """
            }
        }
        
        stage('Upload to Distribution') {
            steps {
                script {
                    // 上传到内部分发服务器或云存储
                    sh """
                        aws s3 cp ${BUILD_OUTPUT}/*.zip \
                        s3://your-game-builds/${params.TARGET_PLATFORM}/
                    """
                }
            }
        }
    }
    
    post {
        success {
            // 发送成功通知
            slackSend(color: 'good', message: "Build #${BUILD_NUMBER} succeeded for ${params.TARGET_PLATFORM}")
        }
        failure {
            // 发送失败通知
            slackSend(color: 'danger', message: "Build #${BUILD_NUMBER} failed for ${params.TARGET_PLATFORM}")
        }
    }
}
```

#### GitHub Actions 示例

```yaml
# .github/workflows/build.yml
name: Build and Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  build-windows:
    runs-on: windows-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      with:
        lfs: true
    
    - name: Setup UE5
      uses: game-ci/setup-unreal@v1
      with:
        unreal-version: '5.3'
    
    - name: Build Project
      run: |
        & "C:\Program Files\Epic Games\UE_5.3\Engine\Build\BatchFiles\Build.bat" `
          YourGameEditor Win64 Development `
          -Project="${{ github.workspace }}\YourGame.uproject"
    
    - name: Run Tests
      run: |
        & "C:\Program Files\Epic Games\UE_5.3\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" `
          "${{ github.workspace }}\YourGame.uproject" `
          -ExecCmds="Automation RunTests YourGame" `
          -unattended -nopause -NullRHI -log
    
    - name: Upload Test Results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: test-results
        path: Saved/Automation/Reports/
```

### 4.2 自动化测试

#### 功能测试（Functional Test）

```cpp
// YourGameFunctionalTest.h
#pragma once

#include "CoreMinimal.h"
#include "FunctionalTest.h"
#include "YourGameFunctionalTest.generated.h"

/**
 * 自动化功能测试基类
 */
UCLASS()
class YOURGAME_API AYourGameFunctionalTest : public AFunctionalTest
{
    GENERATED_BODY()

public:
    AYourGameFunctionalTest();

protected:
    virtual void PrepareTest() override;
    virtual bool IsReady_Implementation() override;
    virtual void StartTest() override;
    
    // 辅助函数
    UFUNCTION(BlueprintCallable, Category="Test")
    void SpawnTestCharacter(TSubclassOf<class ACharacter> CharacterClass, FVector Location);
    
    UFUNCTION(BlueprintCallable, Category="Test")
    void SimulateInput(FName ActionName, float Value);
    
    UFUNCTION(BlueprintCallable, Category="Test")
    void WaitForCondition(float Timeout, const FString& ConditionDescription);
};

// YourGameFunctionalTest.cpp
AYourGameFunctionalTest::AYourGameFunctionalTest()
{
    // 设置默认超时
    TimeLimit = 60.0f;
}

void AYourGameFunctionalTest::PrepareTest()
{
    Super::PrepareTest();
    
    // 初始化测试环境
    UWorld* World = GetWorld();
    if (World)
    {
        // 清理场景
        // 设置游戏状态
    }
}
```

**具体测试用例**

```cpp
// WeaponFireTest.h
UCLASS()
class AWeaponFireTest : public AYourGameFunctionalTest
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category="Test Setup")
    TSubclassOf<class AWeapon> WeaponToTest;

protected:
    virtual void StartTest() override;

private:
    UPROPERTY()
    TObjectPtr<AWeapon> SpawnedWeapon;
    
    UPROPERTY()
    TObjectPtr<AActor> TargetDummy;
    
    UFUNCTION()
    void OnWeaponFired();
    
    UFUNCTION()
    void OnTargetHit(AActor* HitActor, float Damage);
};

// WeaponFireTest.cpp
void AWeaponFireTest::StartTest()
{
    Super::StartTest();
    
    // 1. 生成武器
    SpawnedWeapon = GetWorld()->SpawnActor<AWeapon>(
        WeaponToTest,
        FVector(0, 0, 100),
        FRotator::ZeroRotator
    );
    
    if (!SpawnedWeapon)
    {
        FinishTest(EFunctionalTestResult::Failed, TEXT("Failed to spawn weapon"));
        return;
    }
    
    // 2. 生成目标假人
    TargetDummy = GetWorld()->SpawnActor<AActor>(
        ATargetDummy::StaticClass(),
        FVector(500, 0, 100),
        FRotator::ZeroRotator
    );
    
    // 3. 绑定事件
    SpawnedWeapon->OnFired.AddDynamic(this, &AWeaponFireTest::OnWeaponFired);
    
    // 4. 模拟开火
    SpawnedWeapon->Fire();
    
    // 5. 等待结果（使用定时器）
    GetWorld()->GetTimerManager().SetTimer(
        TestTimerHandle,
        [this]() {
            if (bWeaponFired && bTargetHit)
            {
                FinishTest(EFunctionalTestResult::Succeeded, TEXT("Weapon fire test passed"));
            }
            else
            {
                FinishTest(EFunctionalTestResult::Failed, TEXT("Weapon did not fire or hit target"));
            }
        },
        5.0f,
        false
    );
}

void AWeaponFireTest::OnWeaponFired()
{
    bWeaponFired = true;
    LogStep(TEXT("Weapon fired successfully"));
}

void AWeaponFireTest::OnTargetHit(AActor* HitActor, float Damage)
{
    if (HitActor == TargetDummy)
    {
        bTargetHit = true;
        LogStep(FString::Printf(TEXT("Target hit with damage: %.1f"), Damage));
    }
}
```

#### 性能测试

```cpp
// PerformanceBenchmark.h
UCLASS()
class APerformanceBenchmark : public AFunctionalTest
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category="Benchmark")
    int32 NumActorsToSpawn = 100;
    
    UPROPERTY(EditAnywhere, Category="Benchmark")
    float TargetFPS = 60.0f;

protected:
    virtual void StartTest() override;

private:
    void MeasureFrameTime();
    
    TArray<float> FrameTimes;
    float AverageFPS;
};

// PerformanceBenchmark.cpp
void APerformanceBenchmark::StartTest()
{
    Super::StartTest();
    
    // 生成大量 Actor 进行压力测试
    for (int32 i = 0; i < NumActorsToSpawn; ++i)
    {
        FVector Location = FVector(
            FMath::RandRange(-1000, 1000),
            FMath::RandRange(-1000, 1000),
            100
        );
        
        GetWorld()->SpawnActor<AActor>(
            ATestActor::StaticClass(),
            Location,
            FRotator::ZeroRotator
        );
    }
    
    // 测量帧率
    GetWorld()->GetTimerManager().SetTimer(
        MeasureTimerHandle,
        this,
        &APerformanceBenchmark::MeasureFrameTime,
        0.1f,  // 每 100ms 采样一次
        true   // 循环
    );
    
    // 10 秒后结束测试
    GetWorld()->GetTimerManager().SetTimer(
        EndTestTimerHandle,
        [this]() {
            // 计算平均 FPS
            float TotalFrameTime = 0.0f;
            for (float FrameTime : FrameTimes)
            {
                TotalFrameTime += FrameTime;
            }
            AverageFPS = 1.0f / (TotalFrameTime / FrameTimes.Num());
            
            // 判断是否通过
            if (AverageFPS >= TargetFPS)
            {
                FinishTest(EFunctionalTestResult::Succeeded, 
                    FString::Printf(TEXT("Average FPS: %.1f (Target: %.1f)"), AverageFPS, TargetFPS));
            }
            else
            {
                FinishTest(EFunctionalTestResult::Failed,
                    FString::Printf(TEXT("Average FPS: %.1f below target %.1f"), AverageFPS, TargetFPS));
            }
        },
        10.0f,
        false
    );
}

void APerformanceBenchmark::MeasureFrameTime()
{
    FrameTimes.Add(GetWorld()->GetDeltaSeconds());
}
```

### 4.3 版本管理和发布流程

#### 版本号规范（语义化版本）

```
格式：MAJOR.MINOR.PATCH-PRERELEASE+BUILD

例如：
1.0.0         # 正式版本
1.1.0         # 功能更新
1.1.1         # Bug 修复
2.0.0-beta.1  # Beta 测试版
2.0.0-rc.2    # Release Candidate
1.0.0+20240220.1  # 带构建号
```

**版本管理代码**

```cpp
// VersionInfo.h
#pragma once

#include "CoreMinimal.h"
#include "VersionInfo.generated.h"

USTRUCT(BlueprintType)
struct FGameVersionInfo
{
    GENERATED_BODY()
    
    UPROPERTY(BlueprintReadOnly)
    int32 Major = 1;
    
    UPROPERTY(BlueprintReadOnly)
    int32 Minor = 0;
    
    UPROPERTY(BlueprintReadOnly)
    int32 Patch = 0;
    
    UPROPERTY(BlueprintReadOnly)
    FString Prerelease;  // "beta", "rc", etc.
    
    UPROPERTY(BlueprintReadOnly)
    FString BuildMetadata;  // 构建日期、Git hash 等
    
    FString ToString() const
    {
        FString Version = FString::Printf(TEXT("%d.%d.%d"), Major, Minor, Patch);
        
        if (!Prerelease.IsEmpty())
        {
            Version += TEXT("-") + Prerelease;
        }
        
        if (!BuildMetadata.IsEmpty())
        {
            Version += TEXT("+") + BuildMetadata;
        }
        
        return Version;
    }
    
    bool IsNewerThan(const FGameVersionInfo& Other) const
    {
        if (Major != Other.Major) return Major > Other.Major;
        if (Minor != Other.Minor) return Minor > Other.Minor;
        if (Patch != Other.Patch) return Patch > Other.Patch;
        
        // Prerelease versions have lower precedence
        if (Prerelease.IsEmpty() && !Other.Prerelease.IsEmpty()) return true;
        if (!Prerelease.IsEmpty() && Other.Prerelease.IsEmpty()) return false;
        
        return Prerelease > Other.Prerelease;
    }
};

// 自动生成版本信息（在 build 时）
// GenerateVersionInfo.sh

#!/bin/bash
# 自动生成版本信息头文件

VERSION_FILE="Source/YourGame/Generated/VersionGenerated.h"

# 从 Git 获取信息
GIT_HASH=$(git rev-parse --short HEAD)
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
BUILD_DATE=$(date +%Y%m%d)

cat > $VERSION_FILE << EOF
// Auto-generated version file
// DO NOT EDIT MANUALLY

#pragma once

#define GAME_VERSION_MAJOR 1
#define GAME_VERSION_MINOR 0
#define GAME_VERSION_PATCH 0
#define GAME_VERSION_PRERELEASE "beta.1"
#define GAME_BUILD_DATE "${BUILD_DATE}"
#define GAME_GIT_HASH "${GIT_HASH}"
#define GAME_GIT_BRANCH "${GIT_BRANCH}"

EOF

echo "Version info generated: $VERSION_FILE"
```

---

## 第五部分：跨平台发布

### 5.1 平台特定配置

#### Windows (PC)

```ini
; Config/Windows/WindowsEngine.ini

[/Script/WindowsTargetPlatform.WindowsTargetSettings]
DefaultGraphicsRHI=DefaultGraphicsRHI_DX12
Compiler=VisualStudio2022
AudioSampleRate=48000
AudioCallbackBufferFrameSize=1024
AudioNumBuffersToEnqueue=1
AudioMaxChannels=32
AudioNumSourceWorkers=4

; DX12 特定设置
bTargetedRHIsOnly=True
D3D12TargetedShaderFormats=PCD3D_SM6

; 窗口设置
MinScreenWidth=1024
MinScreenHeight=768
bUseLocklessAudioDeviceUpdate=True
```

#### PlayStation 5

```ini
; Config/PS5/PS5Engine.ini

[/Script/PS5TargetPlatform.PS5TargetSettings]
bEnableRayTracing=True
bEnableTempest3DAudio=True
PS5ActivityFeedEnabled=True

; 性能设置
TargetFrameRate=60
DynamicResolutionMin=1080
DynamicResolutionMax=2160
bUseDynamicResolution=True

; 联机功能
bEnablePS5OnlineFeatures=True
TitleId=XXXX-XXXXX_00-YOURGAMEXXXXXXXXX
```

#### Xbox Series X|S

```ini
; Config/XSX/XSXEngine.ini

[/Script/XSXTargetPlatform.XSXTargetSettings]
bEnableRayTracing=True
bEnableDirectStorage=True
bEnableQuickResume=True

; 变分辨率
VariableRateShadingTier=Tier2

; Xbox Live
bEnableXboxLiveFeatures=True
TitleId=XXXXXXXXX
ServiceConfigId=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

#### iOS

```ini
; Config/IOS/IOSEngine.ini

[/Script/IOSRuntimeSettings.IOSRuntimeSettings]
MinimumiOSVersion=IOS_15
bSupportsPortraitOrientation=False
bSupportsLandscapeLeftOrientation=True
bSupportsLandscapeRightOrientation=True
BundleIdentifier=com.yourcompany.yourgame

; Metal 设置
bSupportsMetal=True
bSupportsMetalMRT=True
MaxShaderLanguageVersion=5

; 性能
MetalShaderStandard=2
IOSMetalShaderStandard=2
bEnableMobileShaderStatsCaching=True
```

#### Android

```ini
; Config/Android/AndroidEngine.ini

[/Script/AndroidRuntimeSettings.AndroidRuntimeSettings]
PackageName=com.yourcompany.yourgame
StoreVersion=1
StoreVersionOffsetArm64=0
ApplicationDisplayName=Your Game Name
VersionDisplayName=1.0.0
MinSDKVersion=28
TargetSDKVersion=33

; 图形 API
bSupportsVulkan=True
bSupportsOpenGLES3=False

; 性能
bMultiThreadedRendering=True
bBuildForArm64=True
bBuildForX8664=False  # 根据需求

; Google Play 服务
bEnableGooglePlaySupport=True
GamesAppID=123456789012
```

### 5.2 平台差异处理

#### 输入适配

```cpp
// PlatformInputAdapter.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "PlatformInputAdapter.generated.h"

/**
 * 跨平台输入适配器
 */
UCLASS()
class YOURGAME_API UPlatformInputAdapter : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 检测当前输入设备类型
    UFUNCTION(BlueprintCallable, Category="Input")
    EInputDeviceType GetCurrentInputDevice() const { return CurrentInputDevice; }

    // 获取平台特定的按钮图标
    UFUNCTION(BlueprintCallable, Category="Input")
    UTexture2D* GetButtonIcon(FName ActionName) const;

    // 获取平台特定的提示文本
    UFUNCTION(BlueprintCallable, Category="Input")
    FText GetButtonPrompt(FName ActionName) const;

private:
    UPROPERTY()
    EInputDeviceType CurrentInputDevice;

    UPROPERTY()
    TMap<FName, TObjectPtr<UTexture2D>> KeyboardIcons;

    UPROPERTY()
    TMap<FName, TObjectPtr<UTexture2D>> GamepadIcons;

    UPROPERTY()
    TMap<FName, TObjectPtr<UTexture2D>> TouchIcons;

    void DetectInputDevice();
    void LoadPlatformIcons();
};

// PlatformInputAdapter.cpp
void UPlatformInputAdapter::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    
    DetectInputDevice();
    LoadPlatformIcons();
}

void UPlatformInputAdapter::DetectInputDevice()
{
    FString PlatformName = UGameplayStatics::GetPlatformName();
    
    if (PlatformName == TEXT("Windows") || PlatformName == TEXT("Mac") || PlatformName == TEXT("Linux"))
    {
        CurrentInputDevice = EInputDeviceType::KeyboardMouse;
    }
    else if (PlatformName == TEXT("PS5") || PlatformName == TEXT("XSX") || PlatformName == TEXT("Switch"))
    {
        CurrentInputDevice = EInputDeviceType::Gamepad;
    }
    else if (PlatformName == TEXT("IOS") || PlatformName == TEXT("Android"))
    {
        CurrentInputDevice = EInputDeviceType::Touch;
    }
}

FText UPlatformInputAdapter::GetButtonPrompt(FName ActionName) const
{
    // 根据平台和输入设备返回不同的提示
    switch (CurrentInputDevice)
    {
        case EInputDeviceType::KeyboardMouse:
            if (ActionName == TEXT("Jump")) return FText::FromString(TEXT("Press SPACE to Jump"));
            break;
            
        case EInputDeviceType::Gamepad:
        {
            FString PlatformName = UGameplayStatics::GetPlatformName();
            if (PlatformName == TEXT("PS5"))
            {
                if (ActionName == TEXT("Jump")) return FText::FromString(TEXT("Press X to Jump"));
            }
            else if (PlatformName == TEXT("XSX"))
            {
                if (ActionName == TEXT("Jump")) return FText::FromString(TEXT("Press A to Jump"));
            }
            break;
        }
            
        case EInputDeviceType::Touch:
            if (ActionName == TEXT("Jump")) return FText::FromString(TEXT("Tap Jump Button"));
            break;
    }
    
    return FText::FromName(ActionName);
}
```

#### 平台特定功能封装

```cpp
// PlatformFeatures.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "PlatformFeatures.generated.h"

/**
 * 平台特定功能的统一接口
 */
UCLASS()
class YOURGAME_API UPlatformFeatures : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 成就系统
    UFUNCTION(BlueprintCallable, Category="Achievements")
    void UnlockAchievement(FName AchievementId);

    // 云存档
    UFUNCTION(BlueprintCallable, Category="CloudSave")
    void SaveToCloud(const TArray<uint8>& SaveData);

    UFUNCTION(BlueprintCallable, Category="CloudSave")
    void LoadFromCloud(TArray<uint8>& OutSaveData);

    // 好友系统
    UFUNCTION(BlueprintCallable, Category="Social")
    void InviteFriend();

    UFUNCTION(BlueprintCallable, Category="Social")
    void GetFriendsList(TArray<FString>& OutFriends);

    // 平台特定 UI
    UFUNCTION(BlueprintCallable, Category="UI")
    void ShowPlatformOverlay(FName OverlayType);

private:
    // 平台特定实现
#if PLATFORM_PS5
    void UnlockAchievement_PS5(FName AchievementId);
#elif PLATFORM_XBOXONE || PLATFORM_XSX
    void UnlockAchievement_Xbox(FName AchievementId);
#elif PLATFORM_SWITCH
    void UnlockAchievement_Switch(FName AchievementId);
#elif PLATFORM_IOS
    void UnlockAchievement_iOS(FName AchievementId);
#elif PLATFORM_ANDROID
    void UnlockAchievement_Android(FName AchievementId);
#endif
};

// PlatformFeatures.cpp
void UPlatformFeatures::UnlockAchievement(FName AchievementId)
{
#if PLATFORM_PS5
    UnlockAchievement_PS5(AchievementId);
#elif PLATFORM_XBOXONE || PLATFORM_XSX
    UnlockAchievement_Xbox(AchievementId);
#elif PLATFORM_SWITCH
    UnlockAchievement_Switch(AchievementId);
#elif PLATFORM_IOS
    UnlockAchievement_iOS(AchievementId);
#elif PLATFORM_ANDROID
    UnlockAchievement_Android(AchievementId);
#else
    UE_LOG(LogTemp, Warning, TEXT("Achievement system not implemented for this platform"));
#endif
}

#if PLATFORM_PS5
void UPlatformFeatures::UnlockAchievement_PS5(FName AchievementId)
{
    // 使用 PS5 SDK
    // sceTrophyUnlockTrophy(context, trophyId, ...);
}
#endif

#if PLATFORM_ANDROID
void UPlatformFeatures::UnlockAchievement_Android(FName AchievementId)
{
    // 使用 Google Play Games Services
    if (IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get())
    {
        IOnlineAchievementsPtr Achievements = OnlineSubsystem->GetAchievementsInterface();
        if (Achievements.IsValid())
        {
            FUniqueNetIdPtr UserId = GetGameInstance()->GetFirstGamePlayer()->GetPreferredUniqueNetId().GetUniqueNetId();
            FOnlineAchievementsWriteRef WriteObject = MakeShareable(new FOnlineAchievementsWrite());
            WriteObject->SetFloatStat(*AchievementId.ToString(), 100.0f);
            
            Achievements->WriteAchievements(*UserId, WriteObject);
        }
    }
}
#endif
```

### 5.3 性能优化策略

#### 移动平台优化

```cpp
// MobileOptimizationSettings.h
UCLASS(Config=Game, DefaultConfig)
class UMobileOptimizationSettings : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    // 图形质量
    UPROPERTY(Config, EditAnywhere, Category="Graphics")
    bool bUseLowQualityLightmaps = true;

    UPROPERTY(Config, EditAnywhere, Category="Graphics")
    bool bDisableDynamicShadows = true;

    UPROPERTY(Config, EditAnywhere, Category="Graphics")
    int32 MaxDrawCalls = 1000;

    // 物理
    UPROPERTY(Config, EditAnywhere, Category="Physics")
    bool bSimplifyPhysicsAssets = true;

    UPROPERTY(Config, EditAnywhere, Category="Physics")
    int32 MaxPhysicsBodies = 50;

    // 音频
    UPROPERTY(Config, EditAnywhere, Category="Audio")
    int32 MaxConcurrentSounds = 16;

    UPROPERTY(Config, EditAnywhere, Category="Audio")
    float AudioStreamingCacheSizeMB = 16.0f;

    // LOD
    UPROPERTY(Config, EditAnywhere, Category="LOD")
    float LODDistanceScale = 0.7f;

    UPROPERTY(Config, EditAnywhere, Category="LOD")
    int32 MaxLODLevel = 3;
};

// 运行时应用优化
void ApplyMobileOptimizations()
{
    if (PLATFORM_ANDROID || PLATFORM_IOS)
    {
        // 降低分辨率比例
        IConsoleVariable* CVarResolutionQuality = IConsoleManager::Get().FindConsoleVariable(TEXT("r.ScreenPercentage"));
        if (CVarResolutionQuality)
        {
            CVarResolutionQuality->Set(75);  // 75% 分辨率
        }

        // 禁用不必要的后处理
        IConsoleVariable* CVarBloom = IConsoleManager::Get().FindConsoleVariable(TEXT("r.Bloom.Quality"));
        if (CVarBloom)
        {
            CVarBloom->Set(0);  // 关闭 Bloom
        }

        // 降低阴影质量
        IConsoleVariable* CVarShadowQuality = IConsoleManager::Get().FindConsoleVariable(TEXT("r.ShadowQuality"));
        if (CVarShadowQuality)
        {
            CVarShadowQuality->Set(2);  // 低阴影质量
        }

        // 限制粒子数量
        IConsoleVariable* CVarMaxParticles = IConsoleManager::Get().FindConsoleVariable(TEXT("fx.MaxCPUParticlesPerEmitter"));
        if (CVarMaxParticles)
        {
            CVarMaxParticles->Set(100);
        }
    }
}
```

#### 主机平台优化

```cpp
// ConsoleOptimizationSettings.h
UCLASS()
class UConsoleOptimizationManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 动态分辨率调整
    UFUNCTION()
    void UpdateDynamicResolution();

    // 性能模式切换
    UFUNCTION(BlueprintCallable)
    void SetPerformanceMode(EPerformanceMode Mode);

private:
    float CurrentFrameTime;
    float TargetFrameTime;
    
    void EnableRayTracing(bool bEnable);
    void AdjustLODSettings(float Scale);
};

void UConsoleOptimizationManager::UpdateDynamicResolution()
{
    CurrentFrameTime = FApp::GetDeltaTime();
    TargetFrameTime = 1.0f / 60.0f;  // 60 FPS 目标

    if (CurrentFrameTime > TargetFrameTime * 1.1f)
    {
        // 性能不足，降低分辨率
        float NewScreenPercentage = FMath::Max(
            GEngine->GetDynamicResolutionCurrentScreenPercentage() - 5.0f,
            50.0f
        );
        GEngine->SetDynamicResolutionScreenPercentage(NewScreenPercentage);
    }
    else if (CurrentFrameTime < TargetFrameTime * 0.9f)
    {
        // 性能有余，提高分辨率
        float NewScreenPercentage = FMath::Min(
            GEngine->GetDynamicResolutionCurrentScreenPercentage() + 2.0f,
            100.0f
        );
        GEngine->SetDynamicResolutionScreenPercentage(NewScreenPercentage);
    }
}

void UConsoleOptimizationManager::SetPerformanceMode(EPerformanceMode Mode)
{
    switch (Mode)
    {
        case EPerformanceMode::Quality:
            // 4K 30fps 模式
            EnableRayTracing(true);
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.ScreenPercentage"))->Set(100);
            AdjustLODSettings(1.0f);
            break;

        case EPerformanceMode::Balanced:
            // 动态 4K 60fps 模式
            EnableRayTracing(false);
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.ScreenPercentage"))->Set(80);
            AdjustLODSettings(0.8f);
            break;

        case EPerformanceMode::Performance:
            // 1080p 120fps 模式
            EnableRayTracing(false);
            IConsoleManager::Get().FindConsoleVariable(TEXT("r.ScreenPercentage"))->Set(60);
            AdjustLODSettings(0.6f);
            break;
    }
}
```

---

## 第六部分：商业化系统实现

### 6.1 内购（IAP）系统

#### 商品定义

```cpp
// IAPProduct.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "IAPProduct.generated.h"

UENUM(BlueprintType)
enum class EProductType : uint8
{
    Consumable,      // 消耗品（金币、道具等）
    NonConsumable,   // 永久购买（移除广告、解锁内容）
    Subscription     // 订阅（VIP、Battle Pass）
};

UCLASS(BlueprintType)
class YOURGAME_API UIAPProduct : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 产品 ID（对应平台商店 ID）
    UPROPERTY(EditDefaultsOnly, Category="Product")
    FString ProductId;

    // 显示名称
    UPROPERTY(EditDefaultsOnly, Category="Product")
    FText DisplayName;

    // 描述
    UPROPERTY(EditDefaultsOnly, Category="Product")
    FText Description;

    // 产品类型
    UPROPERTY(EditDefaultsOnly, Category="Product")
    EProductType ProductType;

    // 价格（仅供显示，实际价格由平台控制）
    UPROPERTY(EditDefaultsOnly, Category="Product")
    float BasePrice;

    // 货币代码
    UPROPERTY(EditDefaultsOnly, Category="Product")
    FString CurrencyCode;

    // 图标
    UPROPERTY(EditDefaultsOnly, Category="Product")
    TObjectPtr<UTexture2D> ProductIcon;

    // 奖励内容
    UPROPERTY(EditDefaultsOnly, Category="Rewards")
    TMap<FName, int32> RewardItems;  // 例如：{"Gold": 1000, "Gems": 100}

    // 平台特定 ID 映射
    UPROPERTY(EditDefaultsOnly, Category="Platform")
    TMap<FString, FString> PlatformProductIds;  // {"iOS": "com.game.gold1000", "Android": "gold_1000"}
};
```

#### IAP 管理器

```cpp
// IAPManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "Interfaces/OnlineStoreInterface.h"
#include "Interfaces/OnlinePurchaseInterface.h"
#include "IAPManager.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnPurchaseComplete, bool, bSuccess, FString, ProductId);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnProductsLoaded, bool, bSuccess);

/**
 * 跨平台 IAP 管理器
 */
UCLASS()
class YOURGAME_API UIAPManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 查询可购买产品
    UFUNCTION(BlueprintCallable, Category="IAP")
    void QueryProducts();

    // 获取产品信息
    UFUNCTION(BlueprintCallable, Category="IAP")
    UIAPProduct* GetProduct(const FString& ProductId) const;

    // 发起购买
    UFUNCTION(BlueprintCallable, Category="IAP")
    void PurchaseProduct(const FString& ProductId);

    // 恢复购买（仅限非消耗品）
    UFUNCTION(BlueprintCallable, Category="IAP")
    void RestorePurchases();

    // 检查产品是否已拥有
    UFUNCTION(BlueprintCallable, Category="IAP")
    bool IsProductOwned(const FString& ProductId) const;

    // 事件
    UPROPERTY(BlueprintAssignable, Category="IAP")
    FOnProductsLoaded OnProductsLoaded;

    UPROPERTY(BlueprintAssignable, Category="IAP")
    FOnPurchaseComplete OnPurchaseComplete;

private:
    UPROPERTY()
    TMap<FString, TObjectPtr<UIAPProduct>> ProductCatalog;

    UPROPERTY()
    TSet<FString> OwnedProducts;

    IOnlineStoreV2Ptr OnlineStore;
    IOnlinePurchasePtr OnlinePurchase;

    void HandleQueryProductsComplete(bool bWasSuccessful, const TArray<FUniqueOfferId>& OfferIds, const FString& Error);
    void HandlePurchaseComplete(const FOnlineError& Result, const TSharedRef<FPurchaseReceipt>& Receipt);
    
    void GrantProductRewards(const FString& ProductId);
    void SavePurchaseReceipt(const FString& ProductId, const FString& Receipt);
};

// IAPManager.cpp
void UIAPManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 获取在线子系统
    if (IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get())
    {
        OnlineStore = OnlineSubsystem->GetStoreV2Interface();
        OnlinePurchase = OnlineSubsystem->GetPurchaseInterface();
    }

    // 加载产品目录
    // 这里应该从数据资产加载
}

void UIAPManager::QueryProducts()
{
    if (!OnlineStore.IsValid())
    {
        UE_LOG(LogTemp, Error, TEXT("OnlineStore interface not available"));
        OnProductsLoaded.Broadcast(false);
        return;
    }

    FUniqueNetIdPtr UserId = GetGameInstance()->GetFirstGamePlayer()->GetPreferredUniqueNetId().GetUniqueNetId();
    
    TArray<FUniqueOfferId> OfferIds;
    for (const auto& Pair : ProductCatalog)
    {
        // 获取平台特定的产品 ID
        FString PlatformId = GetPlatformProductId(Pair.Key);
        OfferIds.Add(FUniqueOfferId(PlatformId));
    }

    OnlineStore->QueryOffersById(*UserId, OfferIds,
        FOnQueryOnlineStoreOffersComplete::CreateUObject(this, &UIAPManager::HandleQueryProductsComplete));
}

void UIAPManager::PurchaseProduct(const FString& ProductId)
{
    if (!OnlinePurchase.IsValid())
    {
        UE_LOG(LogTemp, Error, TEXT("OnlinePurchase interface not available"));
        OnPurchaseComplete.Broadcast(false, ProductId);
        return;
    }

    UIAPProduct* Product = GetProduct(ProductId);
    if (!Product)
    {
        UE_LOG(LogTemp, Error, TEXT("Product not found: %s"), *ProductId);
        OnPurchaseComplete.Broadcast(false, ProductId);
        return;
    }

    FUniqueNetIdPtr UserId = GetGameInstance()->GetFirstGamePlayer()->GetPreferredUniqueNetId().GetUniqueNetId();
    
    FPurchaseCheckoutRequest CheckoutRequest;
    CheckoutRequest.AddPurchaseOffer(TEXT(""), GetPlatformProductId(ProductId), 1);

    OnlinePurchase->Checkout(*UserId, CheckoutRequest,
        FOnPurchaseCheckoutComplete::CreateUObject(this, &UIAPManager::HandlePurchaseComplete));
}

void UIAPManager::HandlePurchaseComplete(const FOnlineError& Result, const TSharedRef<FPurchaseReceipt>& Receipt)
{
    if (Result.bSucceeded)
    {
        // 验证收据（应该由服务器验证）
        FString ProductId = Receipt->TransactionId;  // 简化示例
        
        // 发放奖励
        GrantProductRewards(ProductId);
        
        // 保存收据
        SavePurchaseReceipt(ProductId, Receipt->ReceiptOffers[0].LineItems[0].UniqueId.ToString());
        
        OnPurchaseComplete.Broadcast(true, ProductId);
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("Purchase failed: %s"), *Result.ErrorMessage.ToString());
        OnPurchaseComplete.Broadcast(false, TEXT(""));
    }
}

void UIAPManager::GrantProductRewards(const FString& ProductId)
{
    UIAPProduct* Product = GetProduct(ProductId);
    if (!Product) return;

    // 发放奖励
    // 这里应该调用你的游戏系统（库存、货币等）
    for (const auto& RewardPair : Product->RewardItems)
    {
        FName ItemType = RewardPair.Key;
        int32 Amount = RewardPair.Value;
        
        // 例如：GetGameInstance()->GetSubsystem<UInventorySystem>()->AddItem(ItemType, Amount);
        UE_LOG(LogTemp, Log, TEXT("Granted reward: %s x %d"), *ItemType.ToString(), Amount);
    }

    // 如果是非消耗品，标记为已拥有
    if (Product->ProductType == EProductType::NonConsumable)
    {
        OwnedProducts.Add(ProductId);
    }
}
```

### 6.2 广告系统

```cpp
// AdManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "AdManager.generated.h"

UENUM(BlueprintType)
enum class EAdType : uint8
{
    Banner,          // 横幅广告
    Interstitial,    // 插屏广告
    RewardedVideo    // 激励视频广告
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnAdEvent, EAdType, AdType, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRewardedAdCompleted, FString, RewardType);

UCLASS()
class YOURGAME_API UAdManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 显示广告
    UFUNCTION(BlueprintCallable, Category="Ads")
    void ShowAd(EAdType AdType, FString PlacementId = TEXT(""));

    // 隐藏横幅广告
    UFUNCTION(BlueprintCallable, Category="Ads")
    void HideBannerAd();

    // 检查广告是否可用
    UFUNCTION(BlueprintCallable, Category="Ads")
    bool IsAdReady(EAdType AdType) const;

    // 预加载广告
    UFUNCTION(BlueprintCallable, Category="Ads")
    void PreloadAd(EAdType AdType);

    // 事件
    UPROPERTY(BlueprintAssignable, Category="Ads")
    FOnAdEvent OnAdShown;

    UPROPERTY(BlueprintAssignable, Category="Ads")
    FOnAdEvent OnAdClosed;

    UPROPERTY(BlueprintAssignable, Category="Ads")
    FOnRewardedAdCompleted OnRewardedAdCompleted;

private:
    TMap<EAdType, bool> AdReadyStatus;

    // 平台特定实现
#if PLATFORM_ANDROID
    void ShowAd_Android(EAdType AdType, FString PlacementId);
#elif PLATFORM_IOS
    void ShowAd_iOS(EAdType AdType, FString PlacementId);
#endif

    void OnRewardedAdCompletedInternal();
};

// AdManager.cpp
void UAdManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 初始化广告 SDK
#if PLATFORM_ANDROID
    // 初始化 Google AdMob 或其他 Android 广告 SDK
#elif PLATFORM_IOS
    // 初始化 AdMob iOS SDK
#endif

    // 预加载常用广告
    PreloadAd(EAdType::Interstitial);
    PreloadAd(EAdType::RewardedVideo);
}

void UAdManager::ShowAd(EAdType AdType, FString PlacementId)
{
    // 检查是否已移除广告（通过 IAP）
    UIAPManager* IAPManager = GetGameInstance()->GetSubsystem<UIAPManager>();
    if (IAPManager && IAPManager->IsProductOwned(TEXT("remove_ads")))
    {
        UE_LOG(LogTemp, Log, TEXT("User has purchased ad removal, skipping ad"));
        
        // 如果是激励广告，仍然给予奖励
        if (AdType == EAdType::RewardedVideo)
        {
            OnRewardedAdCompleted.Broadcast(TEXT("NoAds"));
        }
        return;
    }

    if (!IsAdReady(AdType))
    {
        UE_LOG(LogTemp, Warning, TEXT("Ad not ready: %d"), (int32)AdType);
        OnAdShown.Broadcast(AdType, false);
        return;
    }

#if PLATFORM_ANDROID
    ShowAd_Android(AdType, PlacementId);
#elif PLATFORM_IOS
    ShowAd_iOS(AdType, PlacementId);
#else
    // PC/Console 不显示广告，直接给予奖励（测试用）
    if (AdType == EAdType::RewardedVideo)
    {
        OnRewardedAdCompleted.Broadcast(TEXT("Test"));
    }
#endif
}

#if PLATFORM_ANDROID
void UAdManager::ShowAd_Android(EAdType AdType, FString PlacementId)
{
    // 使用 JNI 调用 Android 广告 SDK
    if (JNIEnv* Env = FAndroidApplication::GetJavaEnv())
    {
        jstring jPlacementId = Env->NewStringUTF(TCHAR_TO_UTF8(*PlacementId));
        
        static jmethodID ShowAdMethod = FJavaWrapper::FindMethod(
            Env,
            FJavaWrapper::GameActivityClassID,
            "ShowAd",
            "(ILjava/lang/String;)V",
            false
        );
        
        FJavaWrapper::CallVoidMethod(
            Env,
            FJavaWrapper::GameActivityThis,
            ShowAdMethod,
            (int)AdType,
            jPlacementId
        );
        
        Env->DeleteLocalRef(jPlacementId);
    }
}
#endif
```

### 6.3 Battle Pass 系统

```cpp
// BattlePassSystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "BattlePassSystem.generated.h"

USTRUCT(BlueprintType)
struct FBattlePassReward
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FName RewardId;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FText RewardName;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<UTexture2D> RewardIcon;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    bool bPremiumOnly;  // 是否需要高级 Pass

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    int32 RequiredLevel;
};

USTRUCT(BlueprintType)
struct FBattlePassSeason
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    int32 SeasonNumber;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FText SeasonName;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FDateTime StartDate;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FDateTime EndDate;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TArray<FBattlePassReward> Rewards;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    int32 MaxLevel;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TArray<int32> XPPerLevel;  // 每级所需经验
};

UCLASS()
class YOURGAME_API UBattlePassSystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 获取当前赛季
    UFUNCTION(BlueprintCallable, Category="BattlePass")
    FBattlePassSeason GetCurrentSeason() const { return CurrentSeason; }

    // 获取玩家等级
    UFUNCTION(BlueprintCallable, Category="BattlePass")
    int32 GetPlayerLevel() const { return PlayerLevel; }

    // 获取当前经验
    UFUNCTION(BlueprintCallable, Category="BattlePass")
    int32 GetCurrentXP() const { return CurrentXP; }

    // 增加经验
    UFUNCTION(BlueprintCallable, Category="BattlePass")
    void AddXP(int32 Amount);

    // 购买高级 Pass
    UFUNCTION(BlueprintCallable, Category="BattlePass")
    void PurchasePremiumPass();

    // 检查是否拥有高级 Pass
    UFUNCTION(BlueprintCallable, Category="BattlePass")
    bool HasPremiumPass() const { return bHasPremiumPass; }

    // 领取奖励
    UFUNCTION(BlueprintCallable, Category="BattlePass")
    bool ClaimReward(int32 Level);

    // 检查奖励是否已领取
    UFUNCTION(BlueprintCallable, Category="BattlePass")
    bool IsRewardClaimed(int32 Level) const;

    // 获取可领取的奖励列表
    UFUNCTION(BlueprintCallable, Category="BattlePass")
    TArray<FBattlePassReward> GetClaimableRewards() const;

private:
    UPROPERTY()
    FBattlePassSeason CurrentSeason;

    UPROPERTY()
    int32 PlayerLevel;

    UPROPERTY()
    int32 CurrentXP;

    UPROPERTY()
    bool bHasPremiumPass;

    UPROPERTY()
    TSet<int32> ClaimedRewardLevels;

    void CheckLevelUp();
    void GrantReward(const FBattlePassReward& Reward);
    void SaveBattlePassProgress();
    void LoadBattlePassProgress();
};

// BattlePassSystem.cpp
void UBattlePassSystem::AddXP(int32 Amount)
{
    CurrentXP += Amount;
    
    UE_LOG(LogTemp, Log, TEXT("Battle Pass: Added %d XP, Total: %d"), Amount, CurrentXP);
    
    CheckLevelUp();
    SaveBattlePassProgress();
}

void UBattlePassSystem::CheckLevelUp()
{
    if (PlayerLevel >= CurrentSeason.MaxLevel)
    {
        return;  // 已达到最大等级
    }

    int32 RequiredXP = CurrentSeason.XPPerLevel[PlayerLevel];
    
    while (CurrentXP >= RequiredXP && PlayerLevel < CurrentSeason.MaxLevel)
    {
        CurrentXP -= RequiredXP;
        PlayerLevel++;
        
        UE_LOG(LogTemp, Log, TEXT("Battle Pass: Level Up! New Level: %d"), PlayerLevel);
        
        // 广播升级事件
        // OnLevelUp.Broadcast(PlayerLevel);
        
        if (PlayerLevel < CurrentSeason.MaxLevel)
        {
            RequiredXP = CurrentSeason.XPPerLevel[PlayerLevel];
        }
    }
}

bool UBattlePassSystem::ClaimReward(int32 Level)
{
    if (Level > PlayerLevel)
    {
        UE_LOG(LogTemp, Warning, TEXT("Cannot claim reward for level %d, player is only level %d"), Level, PlayerLevel);
        return false;
    }

    if (IsRewardClaimed(Level))
    {
        UE_LOG(LogTemp, Warning, TEXT("Reward for level %d already claimed"), Level);
        return false;
    }

    // 查找奖励
    const FBattlePassReward* Reward = CurrentSeason.Rewards.FindByPredicate([Level](const FBattlePassReward& R) {
        return R.RequiredLevel == Level;
    });

    if (!Reward)
    {
        UE_LOG(LogTemp, Error, TEXT("No reward found for level %d"), Level);
        return false;
    }

    // 检查是否需要高级 Pass
    if (Reward->bPremiumOnly && !bHasPremiumPass)
    {
        UE_LOG(LogTemp, Warning, TEXT("Reward requires premium pass"));
        return false;
    }

    // 发放奖励
    GrantReward(*Reward);
    
    // 标记为已领取
    ClaimedRewardLevels.Add(Level);
    SaveBattlePassProgress();

    return true;
}

void UBattlePassSystem::GrantReward(const FBattlePassReward& Reward)
{
    // 这里应该调用游戏的奖励系统
    UE_LOG(LogTemp, Log, TEXT("Granted Battle Pass reward: %s"), *Reward.RewardName.ToString());
    
    // 例如：
    // GetGameInstance()->GetSubsystem<UInventorySystem>()->AddItem(Reward.RewardId);
}

void UBattlePassSystem::PurchasePremiumPass()
{
    // 调用 IAP 系统购买
    UIAPManager* IAPManager = GetGameInstance()->GetSubsystem<UIAPManager>();
    if (IAPManager)
    {
        IAPManager->PurchaseProduct(TEXT("battle_pass_premium"));
        
        // 购买成功后的回调应该设置 bHasPremiumPass = true
    }
}
```

---

## 第七部分：数据分析与用户反馈

### 7.1 游戏分析系统

```cpp
// AnalyticsManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "Interfaces/IAnalyticsProvider.h"
#include "AnalyticsManager.generated.h"

/**
 * 游戏数据分析管理器
 * 支持多个分析平台（Google Analytics, Firebase, 自定义后端等）
 */
UCLASS()
class YOURGAME_API UAnalyticsManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 记录事件
    UFUNCTION(BlueprintCallable, Category="Analytics")
    void RecordEvent(FName EventName, const TMap<FString, FString>& Attributes);

    // 记录关卡开始
    UFUNCTION(BlueprintCallable, Category="Analytics")
    void RecordLevelStart(FString LevelName, int32 PlayerLevel);

    // 记录关卡完成
    UFUNCTION(BlueprintCallable, Category="Analytics")
    void RecordLevelComplete(FString LevelName, float CompletionTime, bool bSuccess);

    // 记录购买事件
    UFUNCTION(BlueprintCallable, Category="Analytics")
    void RecordPurchase(FString ProductId, float Price, FString Currency);

    // 记录错误/崩溃
    UFUNCTION(BlueprintCallable, Category="Analytics")
    void RecordError(FString ErrorType, FString ErrorMessage);

    // 设置用户属性
    UFUNCTION(BlueprintCallable, Category="Analytics")
    void SetUserProperty(FString PropertyName, FString PropertyValue);

    // 开始计时事件
    UFUNCTION(BlueprintCallable, Category="Analytics")
    void StartTimedEvent(FName EventName);

    // 结束计时事件
    UFUNCTION(BlueprintCallable, Category="Analytics")
    void EndTimedEvent(FName EventName, const TMap<FString, FString>& Attributes);

private:
    TSharedPtr<IAnalyticsProvider> AnalyticsProvider;
    
    TMap<FName, double> TimedEvents;

    void InitializeAnalyticsProvider();
};

// AnalyticsManager.cpp
void UAnalyticsManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    
    InitializeAnalyticsProvider();
}

void UAnalyticsManager::InitializeAnalyticsProvider()
{
    // 初始化分析提供商（例如 Google Analytics）
    FAnalyticsET::Config AnalyticsConfig;
    AnalyticsConfig.APIKeyET = TEXT("YOUR_API_KEY");
    AnalyticsConfig.APIServerET = TEXT("https://your-analytics-server.com");
    
    AnalyticsProvider = FAnalyticsET::Get().CreateAnalyticsProvider(AnalyticsConfig);
    
    if (AnalyticsProvider.IsValid())
    {
        AnalyticsProvider->StartSession();
        UE_LOG(LogTemp, Log, TEXT("Analytics initialized successfully"));
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to initialize analytics"));
    }
}

void UAnalyticsManager::RecordEvent(FName EventName, const TMap<FString, FString>& Attributes)
{
    if (!AnalyticsProvider.IsValid())
    {
        return;
    }

    TArray<FAnalyticsEventAttribute> EventAttributes;
    for (const auto& Pair : Attributes)
    {
        EventAttributes.Add(FAnalyticsEventAttribute(Pair.Key, Pair.Value));
    }

    AnalyticsProvider->RecordEvent(EventName.ToString(), EventAttributes);
    
    UE_LOG(LogTemp, Verbose, TEXT("Analytics: Recorded event '%s' with %d attributes"), 
           *EventName.ToString(), Attributes.Num());
}

void UAnalyticsManager::RecordLevelStart(FString LevelName, int32 PlayerLevel)
{
    TMap<FString, FString> Attributes;
    Attributes.Add(TEXT("level_name"), LevelName);
    Attributes.Add(TEXT("player_level"), FString::FromInt(PlayerLevel));
    
    RecordEvent(TEXT("level_start"), Attributes);
    StartTimedEvent(FName(*LevelName));  // 开始计时
}

void UAnalyticsManager::RecordLevelComplete(FString LevelName, float CompletionTime, bool bSuccess)
{
    TMap<FString, FString> Attributes;
    Attributes.Add(TEXT("level_name"), LevelName);
    Attributes.Add(TEXT("completion_time"), FString::SanitizeFloat(CompletionTime));
    Attributes.Add(TEXT("success"), bSuccess ? TEXT("true") : TEXT("false"));
    
    RecordEvent(TEXT("level_complete"), Attributes);
    EndTimedEvent(FName(*LevelName), Attributes);
}

void UAnalyticsManager::RecordPurchase(FString ProductId, float Price, FString Currency)
{
    if (!AnalyticsProvider.IsValid())
    {
        return;
    }

    TArray<FAnalyticsEventAttribute> Attributes;
    Attributes.Add(FAnalyticsEventAttribute(TEXT("product_id"), ProductId));
    Attributes.Add(FAnalyticsEventAttribute(TEXT("price"), Price));
    Attributes.Add(FAnalyticsEventAttribute(TEXT("currency"), Currency));

    AnalyticsProvider->RecordEvent(TEXT("purchase"), Attributes);
}

void UAnalyticsManager::StartTimedEvent(FName EventName)
{
    TimedEvents.Add(EventName, FPlatformTime::Seconds());
}

void UAnalyticsManager::EndTimedEvent(FName EventName, const TMap<FString, FString>& Attributes)
{
    if (double* StartTime = TimedEvents.Find(EventName))
    {
        double Duration = FPlatformTime::Seconds() - *StartTime;
        
        TMap<FString, FString> ExtendedAttributes = Attributes;
        ExtendedAttributes.Add(TEXT("duration"), FString::SanitizeFloat(Duration));
        
        RecordEvent(EventName, ExtendedAttributes);
        TimedEvents.Remove(EventName);
    }
}
```

### 7.2 用户反馈系统

```cpp
// FeedbackSystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "FeedbackSystem.generated.h"

UENUM(BlueprintType)
enum class EFeedbackType : uint8
{
    Bug,
    Suggestion,
    Compliment,
    Question
};

USTRUCT(BlueprintType)
struct FFeedbackData
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    EFeedbackType Type;

    UPROPERTY(BlueprintReadWrite)
    FString Title;

    UPROPERTY(BlueprintReadWrite)
    FString Description;

    UPROPERTY(BlueprintReadWrite)
    FString UserEmail;

    UPROPERTY(BlueprintReadWrite)
    TArray<uint8> Screenshot;  // PNG 数据

    UPROPERTY(BlueprintReadWrite)
    FString DeviceInfo;

    UPROPERTY(BlueprintReadWrite)
    FString GameVersion;

    UPROPERTY(BlueprintReadWrite)
    FString LevelName;

    UPROPERTY(BlueprintReadWrite)
    FDateTime Timestamp;
};

UCLASS()
class YOURGAME_API UFeedbackSystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 提交反馈
    UFUNCTION(BlueprintCallable, Category="Feedback")
    void SubmitFeedback(const FFeedbackData& FeedbackData);

    // 截取当前屏幕（用于反馈附件）
    UFUNCTION(BlueprintCallable, Category="Feedback")
    TArray<uint8> CaptureScreenshot();

    // 获取设备信息
    UFUNCTION(BlueprintCallable, Category="Feedback")
    FString GetDeviceInfo() const;

private:
    void SendFeedbackToServer(const FFeedbackData& FeedbackData);
    void SaveFeedbackLocally(const FFeedbackData& FeedbackData);
};

// FeedbackSystem.cpp
void UFeedbackSystem::SubmitFeedback(const FFeedbackData& FeedbackData)
{
    // 1. 尝试发送到服务器
    SendFeedbackToServer(FeedbackData);

    // 2. 本地保存备份
    SaveFeedbackLocally(FeedbackData);

    // 3. 记录到分析系统
    UAnalyticsManager* Analytics = GetGameInstance()->GetSubsystem<UAnalyticsManager>();
    if (Analytics)
    {
        TMap<FString, FString> Attributes;
        Attributes.Add(TEXT("type"), UEnum::GetValueAsString(FeedbackData.Type));
        Attributes.Add(TEXT("has_screenshot"), FeedbackData.Screenshot.Num() > 0 ? TEXT("true") : TEXT("false"));
        Analytics->RecordEvent(TEXT("feedback_submitted"), Attributes);
    }
}

void UFeedbackSystem::SendFeedbackToServer(const FFeedbackData& FeedbackData)
{
    // 创建 HTTP 请求
    TSharedRef<IHttpRequest, ESPMode::ThreadSafe> HttpRequest = FHttpModule::Get().CreateRequest();
    
    HttpRequest->SetURL(TEXT("https://your-api.com/feedback"));
    HttpRequest->SetVerb(TEXT("POST"));
    HttpRequest->SetHeader(TEXT("Content-Type"), TEXT("application/json"));

    // 构建 JSON 数据
    TSharedPtr<FJsonObject> JsonObject = MakeShareable(new FJsonObject);
    JsonObject->SetStringField(TEXT("type"), UEnum::GetValueAsString(FeedbackData.Type));
    JsonObject->SetStringField(TEXT("title"), FeedbackData.Title);
    JsonObject->SetStringField(TEXT("description"), FeedbackData.Description);
    JsonObject->SetStringField(TEXT("email"), FeedbackData.UserEmail);
    JsonObject->SetStringField(TEXT("device_info"), FeedbackData.DeviceInfo);
    JsonObject->SetStringField(TEXT("game_version"), FeedbackData.GameVersion);
    JsonObject->SetStringField(TEXT("level_name"), FeedbackData.LevelName);
    JsonObject->SetStringField(TEXT("timestamp"), FeedbackData.Timestamp.ToString());

    // 如果有截图，转换为 Base64
    if (FeedbackData.Screenshot.Num() > 0)
    {
        FString Base64Screenshot = FBase64::Encode(FeedbackData.Screenshot);
        JsonObject->SetStringField(TEXT("screenshot"), Base64Screenshot);
    }

    FString JsonString;
    TSharedRef<TJsonWriter<>> Writer = TJsonWriterFactory<>::Create(&JsonString);
    FJsonSerializer::Serialize(JsonObject.ToSharedRef(), Writer);

    HttpRequest->SetContentAsString(JsonString);

    // 发送请求
    HttpRequest->OnProcessRequestComplete().BindLambda([](FHttpRequestPtr Request, FHttpResponsePtr Response, bool bSuccess) {
        if (bSuccess && Response.IsValid())
        {
            UE_LOG(LogTemp, Log, TEXT("Feedback sent successfully: %s"), *Response->GetContentAsString());
        }
        else
        {
            UE_LOG(LogTemp, Error, TEXT("Failed to send feedback"));
        }
    });

    HttpRequest->ProcessRequest();
}

TArray<uint8> UFeedbackSystem::CaptureScreenshot()
{
    TArray<uint8> CompressedBitmap;
    
    FScreenshotRequest::RequestScreenshot(FString(), false, false);
    
    // 注意：实际实现需要异步处理截图
    // 这里是简化示例
    
    return CompressedBitmap;
}

FString UFeedbackSystem::GetDeviceInfo() const
{
    FString DeviceInfo;
    DeviceInfo += TEXT("Platform: ") + UGameplayStatics::GetPlatformName() + TEXT("\n");
    DeviceInfo += TEXT("OS: ") + FPlatformMisc::GetOSVersion() + TEXT("\n");
    DeviceInfo += TEXT("CPU: ") + FPlatformMisc::GetCPUBrand() + TEXT("\n");
    DeviceInfo += TEXT("GPU: ") + FPlatformMisc::GetPrimaryGPUBrand() + TEXT("\n");
    DeviceInfo += TEXT("RAM: ") + FString::FromInt(FPlatformMemory::GetConstants().TotalPhysicalGB) + TEXT(" GB\n");
    DeviceInfo += TEXT("Resolution: ") + FString::FromInt(GEngine->GameViewport->Viewport->GetSizeXY().X) + 
                  TEXT("x") + FString::FromInt(GEngine->GameViewport->Viewport->GetSizeXY().Y);
    
    return DeviceInfo;
}
```

---

## 第八部分：案例研究 - 成功的 Lyra 衍生项目

虽然 Lyra 是 UE5.0 随引擎发布的新示例项目（2022年），但已经有一些团队基于类似的 Epic Games 模板（如 ShooterGame）成功开发了商业产品。以下是一些可以借鉴的案例和教训：

### 8.1 案例分析：Fortnite（使用 UE4 内部版本）

**技术架构借鉴：**
- **模块化设计**：Fortnite 采用了与 Lyra 类似的模块化架构，允许快速迭代
- **GameFeatures 插件**：虽然 Fortnite 早于 GameFeatures 系统，但其设计理念影响了 Lyra
- **快速更新周期**：每周更新新内容，得益于模块化和数据驱动设计

**商业化策略：**
- Battle Pass 系统成为行业标杆
- 季节性内容和限时活动保持用户参与度
- 跨平台支持扩大用户基础

**从 Fortnite 学到的经验：**
1. **持续内容更新比完美的初始版本更重要**
2. **社交功能是留存的关键**（队伍、好友系统）
3. **免费游戏 + 内购**比付费模式更适合现代市场

### 8.2 中小团队成功案例：基于 UE4 ShooterGame

**案例：《Rogue Company》**
- 基础：使用了 UE4 的多人射击模板
- 改造：完全重制了 UI、角色系统、商业化
- 成功因素：
  - 聚焦核心玩法（5v5 战术射击）
  - 快速原型验证市场
  - 逐步添加功能而非一次性开发所有内容

**技术经验：**
```cpp
// 他们的一个关键优化：简化 Lyra 式的 Experience 系统
// 不使用复杂的 ActionSet，而是固定的 GameMode

UCLASS()
class ARogueCompanyGameMode : public AGameModeBase
{
    GENERATED_BODY()

protected:
    // 简化的角色选择系统
    UPROPERTY(EditDefaultsOnly)
    TArray<TSubclassOf<class ARogueAgent>> AvailableAgents;

    // 固定的游戏规则，不需要动态加载
    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override
    {
        Super::InitGame(MapName, Options, ErrorMessage);
        
        // 直接初始化游戏规则，无需 Experience
        InitializeGameRules();
    }

    void InitializeGameRules()
    {
        // 5v5 模式的固定规则
        MaxPlayers = 10;
        RoundDuration = 180.0f;  // 3 分钟
        RoundsToWin = 7;
    }
};
```

**商业化经验：**
- 从第一天就集成 IAP 系统
- 使用 A/B 测试优化定价
- Battle Pass 作为主要变现手段

### 8.3 失败案例分析

**案例：某独立团队的射击游戏（匿名）**

**失败原因：**
1. **过度依赖 Lyra 的复杂性**
   - 保留了所有 Lyra 系统，但团队无法理解和维护
   - 技术债务累积导致开发速度变慢

2. **忽视性能优化**
   - 直接使用 Lyra 的高端图形设置
   - 移动平台表现糟糕，导致负面评价

3. **商业化策略失误**
   - 采用付费模式，但游戏内容不足
   - 没有长期运营计划

**教训：**
```cpp
// 不要这样做：保留所有 Lyra 系统但不理解它们
UCLASS()
class AMyGameMode : public ALyraGameMode  // 直接继承 Lyra
{
    // 问题：继承了大量不需要的功能
    // 问题：Experience 系统对小团队过于复杂
};

// 应该这样做：只继承必要的部分
UCLASS()
class AMyGameMode : public AModularGameModeBase  // 继承 Modular 基类
{
    // 好处：清晰的架构，只包含需要的功能
    // 好处：易于理解和维护
};
```

---

## 第九部分：法律与授权问题

### 9.1 UE5 和 Lyra 的授权条款

#### Unreal Engine 5 许可

**关键要点：**
1. **免费使用**：UE5 可以免费用于商业项目
2. **收入分成**：游戏总收入超过 100 万美元后，需支付 5% 的授权费
3. **Lyra 内容**：Lyra 项目中的代码和资源可以用于商业项目

**重要条款（截至 2024 年）：**
```
- 前 100 万美元收入：0% 授权费
- 超过 100 万美元部分：5% 授权费
- 使用 Epic Games Store 独占发布：可豁免授权费

特殊情况：
- 如果使用 Fab（原 Unreal Marketplace）的付费资源，需遵守各资源的授权协议
- Lyra 中 Epic 提供的示例内容（代码、蓝图）可以自由使用和修改
- Lyra 中的美术资源（角色、环境）建议替换为原创内容
```

#### 使用 Lyra 内容的注意事项

```cpp
// ✅ 可以使用：Lyra 的代码框架
// YourGame.h - 基于 Lyra 的系统
class YOURGAME_API UYourAbilitySystemComponent : public ULyraAbilitySystemComponent
{
    // 这是合法的：继承和修改 Lyra 的代码
};

// ⚠️ 建议替换：Lyra 的美术资源
// 虽然技术上可以使用，但为了避免法律风险和品牌识别度，应该替换
// Content/Characters/Lyra_Mannequin -> Content/Characters/YourGame_Character

// ❌ 不能使用：Lyra 名称和商标
// 不能在游戏名称或市场推广中使用 "Lyra" 字样
// 不能声称游戏是 "Official Lyra Game"
```

### 9.2 第三方资源的授权

#### Marketplace 资源使用

```cpp
// MarketplaceAssetManager.h
// 追踪和管理第三方资源的授权信息

USTRUCT()
struct FAssetLicenseInfo
{
    GENERATED_BODY()

    UPROPERTY()
    FString AssetName;

    UPROPERTY()
    FString Creator;

    UPROPERTY()
    FString LicenseType;  // "Commercial Use", "Personal Use Only", etc.

    UPROPERTY()
    FString LicenseURL;

    UPROPERTY()
    bool bRequiresAttribution;

    UPROPERTY()
    FString AttributionText;
};

UCLASS()
class UAssetLicenseRegistry : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY()
    TArray<FAssetLicenseInfo> UsedAssets;

    // 生成游戏的授权信息文件
    UFUNCTION()
    void GenerateLicenseFile(FString OutputPath);

    // 检查是否有授权冲突
    UFUNCTION()
    bool ValidateAllLicenses();
};
```

#### 音乐和音效授权

**常见授权类型：**
1. **Royalty-Free**：一次性购买，无需额外版税
2. **Creative Commons**：需要署名，部分允许商业使用
3. **Custom License**：需要单独协商

**管理音频授权：**
```ini
; Config/AudioLicenses.ini
; 记录所有使用的音频资源及其授权

[BackgroundMusic]
Track_01=https://example.com/music1, Royalty-Free, No Attribution Required
Track_02=https://example.com/music2, CC BY 4.0, Attribution Required

[SoundEffects]
Gunshot_01=Internal Creation, No License Required
Explosion_01=https://example.com/sfx, Royalty-Free, Commercial Use Allowed
```

### 9.3 隐私政策和 GDPR 合规

#### 数据收集声明

```cpp
// PrivacyManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "PrivacyManager.generated.h"

UCLASS()
class YOURGAME_API UPrivacyManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 显示隐私政策同意界面（首次启动）
    UFUNCTION(BlueprintCallable, Category="Privacy")
    void ShowPrivacyConsentDialog();

    // 用户同意收集数据
    UFUNCTION(BlueprintCallable, Category="Privacy")
    void AcceptDataCollection();

    // 用户拒绝收集数据
    UFUNCTION(BlueprintCallable, Category="Privacy")
    void DeclineDataCollection();

    // 检查是否允许收集数据
    UFUNCTION(BlueprintCallable, Category="Privacy")
    bool IsDataCollectionAllowed() const;

    // GDPR 数据导出请求
    UFUNCTION(BlueprintCallable, Category="Privacy")
    void RequestDataExport();

    // GDPR 数据删除请求
    UFUNCTION(BlueprintCallable, Category="Privacy")
    void RequestDataDeletion();

private:
    UPROPERTY()
    bool bHasConsentedToDataCollection;

    void SaveConsentStatus();
    void LoadConsentStatus();
};
```

**隐私政策示例（英文）：**
```
Your Game Name - Privacy Policy

Last Updated: [Date]

1. Information We Collect
   - Game progress and achievements
   - Device information (OS, hardware specs)
   - Gameplay analytics (playtime, in-game actions)
   - Crash reports and error logs

2. How We Use Your Information
   - Improve game performance and user experience
   - Provide customer support
   - Develop new features and content

3. Third-Party Services
   - Analytics: [Google Analytics / Firebase]
   - Advertising: [AdMob / Unity Ads]
   - Payment Processing: [App Store / Google Play]

4. Your Rights (GDPR Compliance)
   - Right to access your data
   - Right to delete your data
   - Right to opt-out of data collection

5. Contact Us
   - Email: privacy@yourgame.com
```

---

## 第十部分：上线检查清单

### 10.1 技术检查清单

#### 代码和构建

```markdown
## 代码质量
- [ ] 移除所有 Debug 代码和测试功能
- [ ] 移除所有开发者作弊命令
- [ ] 清理未使用的资源引用
- [ ] 确保没有硬编码的测试数据
- [ ] 代码中没有个人信息或敏感数据

## 构建配置
- [ ] 使用 Shipping 配置构建
- [ ] 启用代码优化
- [ ] 禁用开发者工具（控制台、统计面板等）
- [ ] 启用代码签名（iOS/Android）
- [ ] 配置正确的版本号

## 性能
- [ ] 所有目标平台达到目标帧率
- [ ] 内存使用在安全范围内
- [ ] 没有内存泄漏
- [ ] 加载时间可接受
- [ ] 网络延迟优化

## 安全
- [ ] 所有网络通信使用加密
- [ ] 防作弊系统已启用
- [ ] 服务器端验证关键操作
- [ ] 敏感数据不暴露在客户端
```

**自动化检查脚本：**
```bash
#!/bin/bash
# PrelaunchCheck.sh

echo "=== Pre-Launch Check ==="

# 1. 检查构建配置
if grep -r "DEVELOPMENT_BUILD" Config/; then
    echo "❌ DEVELOPMENT_BUILD flag found in config!"
    exit 1
fi

# 2. 检查是否有测试代码
if grep -r "UE_LOG.*Test" Source/; then
    echo "⚠️  Warning: Test log statements found"
fi

# 3. 检查版本号
VERSION=$(grep "VersionDisplayName" Config/DefaultGame.ini | cut -d'=' -f2)
echo "Version: $VERSION"

if [ "$VERSION" == "0.0.0" ]; then
    echo "❌ Invalid version number!"
    exit 1
fi

# 4. 检查资源大小
CONTENT_SIZE=$(du -sh Content/ | cut -f1)
echo "Content size: $CONTENT_SIZE"

# 5. 运行自动化测试
echo "Running automated tests..."
UnrealEditor-Cmd YourGame.uproject -ExecCmds="Automation RunTests YourGame" -unattended

echo "=== Check Complete ==="
```

### 10.2 内容检查清单

```markdown
## 美术资源
- [ ] 所有纹理已压缩为适当格式
- [ ] LOD 正确配置
- [ ] 材质优化（减少指令数）
- [ ] 没有过大的纹理（移动平台注意）
- [ ] 替换所有 Lyra 示例资源

## 音频
- [ ] 音频文件已压缩
- [ ] 音量平衡合理
- [ ] 支持多语言配音（如适用）
- [ ] 背景音乐循环无缝

## 本地化
- [ ] 所有文本已翻译
- [ ] UI 适配不同语言的文本长度
- [ ] 日期和货币格式正确
- [ ] 文化敏感内容审查

## UI/UX
- [ ] 所有按钮可点击且反馈明确
- [ ] 支持所有目标输入设备
- [ ] 无障碍功能（字幕、色盲模式等）
- [ ] 教程和新手引导完善
```

### 10.3 商业和运营检查清单

```markdown
## 商业化
- [ ] IAP 产品配置正确
- [ ] 价格在所有地区合理
- [ ] 订阅自动续费提示明确
- [ ] 退款流程符合平台规定

## 法律合规
- [ ] 隐私政策已发布
- [ ] 用户协议已准备
- [ ] 年龄分级申请完成
- [ ] GDPR/COPPA 合规（如适用）
- [ ] 所有第三方资源授权合法

## 平台提交
- [ ] App Store / Google Play 商店页面完成
- [ ] 截图和宣传视频准备
- [ ] 关键词和 ASO 优化
- [ ] 分级信息填写正确

## 运营准备
- [ ] 服务器容量规划
- [ ] 监控和告警系统就绪
- [ ] 客服团队培训完成
- [ ] 社交媒体账号创建
- [ ] 发布公告准备
```

### 10.4 发布日检查清单

```markdown
## 发布前（T-24小时）
- [ ] 服务器预热和压力测试
- [ ] CDN 内容预加载
- [ ] 监控系统最终检查
- [ ] 团队待命安排

## 发布时（T-0）
- [ ] 逐步开放服务器（分地区）
- [ ] 实时监控用户数量
- [ ] 监控错误率和崩溃率
- [ ] 社交媒体发布公告

## 发布后（T+1小时）
- [ ] 检查首批用户反馈
- [ ] 监控服务器负载
- [ ] 处理紧急问题
- [ ] 准备热修复（如需要）

## 发布后（T+24小时）
- [ ] 分析首日数据
- [ ] 收集用户反馈
- [ ] 规划第一次更新
- [ ] 撰写发布总结
```

**发布日监控脚本：**
```python
# LaunchDayMonitor.py
import time
import requests
from datetime import datetime

def monitor_game_servers():
    """监控游戏服务器状态"""
    servers = [
        "https://game-server-us.example.com/health",
        "https://game-server-eu.example.com/health",
        "https://game-server-asia.example.com/health",
    ]
    
    while True:
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        for server in servers:
            try:
                response = requests.get(server, timeout=5)
                status = "✅ OK" if response.status_code == 200 else f"❌ Error {response.status_code}"
                concurrent_users = response.json().get("concurrent_users", 0)
                
                print(f"[{timestamp}] {server}: {status} | Users: {concurrent_users}")
                
                # 如果用户数超过阈值，发送警报
                if concurrent_users > 10000:
                    send_alert(f"High traffic alert: {concurrent_users} concurrent users")
                    
            except Exception as e:
                print(f"[{timestamp}] {server}: ❌ Connection Failed - {str(e)}")
                send_alert(f"Server down: {server}")
        
        time.sleep(60)  # 每分钟检查一次

def send_alert(message):
    """发送警报到团队"""
    # 发送到 Slack/Discord/邮件等
    requests.post("https://your-alert-webhook.com", json={"text": message})

if __name__ == "__main__":
    print("Launch Day Monitoring Started...")
    monitor_game_servers()
```

---

## 第十一部分：总结 - Lyra 最佳实践

### 11.1 核心经验总结

通过本系列教程和本文的深入分析，我们可以总结出从 Lyra 到商业游戏的关键最佳实践：

#### 1. **选择性采用，而非全盘接受**

Lyra 是一个功能丰富的示例项目，但并非所有系统都适合你的游戏。

**最佳实践：**
```cpp
// ✅ 好的做法：只继承需要的部分
class AYourGameMode : public AModularGameModeBase  // 基础模块化支持
{
    // 添加你自己的游戏逻辑
};

// ❌ 避免：盲目继承所有 Lyra 系统
class AYourGameMode : public ALyraGameMode
{
    // 继承了大量可能不需要的功能
};
```

**决策框架：**
| Lyra 系统 | 建议采用情况 | 替代方案 |
|-----------|------------|---------|
| ModularGameplay | ✅ 总是采用 | 无，这是基础 |
| GameplayAbilities | ✅ 有技能/能力的游戏 | 简单的组件系统 |
| Experience 系统 | ⚠️ 大型项目或多模式游戏 | 简化的 GameMode |
| 完整的 UI 框架 | ⚠️ 跨平台游戏 | 自定义 UI 系统 |
| Lyra 的库存系统 | ❌ 简单游戏 | 自己实现 |

#### 2. **从小规模原型开始**

不要一开始就构建完整的商业游戏。

**推荐流程：**
```
第 1 周：核心玩法原型（只有基本机制）
第 2-4 周：验证核心循环（是否有趣？）
第 5-8 周：添加基础系统（UI、进度等）
第 9-12 周：内容制作和打磨
第 13-16 周：优化和测试
```

#### 3. **性能优化从第一天开始**

不要等到项目后期才考虑性能。

**关键指标：**
```cpp
// PerformanceBudget.h
// 定义性能预算，在开发过程中持续监控

UCLASS(Config=Game)
class UPerformanceBudget : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    // 目标帧率
    UPROPERTY(Config, EditAnywhere, Category="Performance")
    int32 TargetFPS = 60;

    // Draw Call 预算
    UPROPERTY(Config, EditAnywhere, Category="Performance")
    int32 MaxDrawCalls = 2000;

    // 内存预算（MB）
    UPROPERTY(Config, EditAnywhere, Category="Performance")
    int32 MemoryBudgetMB = 2048;

    // 网络带宽（KB/s）
    UPROPERTY(Config, EditAnywhere, Category="Performance")
    int32 NetworkBandwidthKBps = 50;

    // 在编辑器中显示警告
    void CheckBudgets();
};
```

#### 4. **数据驱动设计**

尽可能使用数据资产而非硬编码。

```cpp
// ✅ 数据驱动
UCLASS()
class UWeaponDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly)
    float Damage;

    UPROPERTY(EditDefaultsOnly)
    float FireRate;
    
    // 策划可以在编辑器中调整，无需程序员
};

// ❌ 硬编码
class AWeapon : public AActor
{
    float Damage = 50.0f;  // 修改需要重新编译
};
```

#### 5. **模块化和可扩展性**

设计时考虑未来的扩展。

```cpp
// 使用插件系统组织功能
YourGame.uproject
├── Plugins/
│   ├── CoreGameplay/       # 核心玩法
│   ├── WeaponSystem/       # 武器系统
│   ├── CharacterSystem/    # 角色系统
│   ├── Monetization/       # 商业化
│   └── Analytics/          # 数据分析

// 每个插件可以独立开发和测试
// 新功能作为新插件添加，不影响现有代码
```

#### 6. **自动化一切可以自动化的事情**

节省时间，减少人为错误。

**自动化清单：**
- ✅ 构建流程（CI/CD）
- ✅ 测试（单元测试、功能测试）
- ✅ 代码审查（静态分析）
- ✅ 资源验证（纹理大小、命名规范）
- ✅ 部署（服务器更新、客户端发布）

### 11.2 团队规模与 Lyra 使用策略

不同规模的团队应该采用不同的策略：

#### 小型团队（1-5人）

**策略：简化和聚焦**
```
采用系统：
✅ ModularGameplay（轻量级）
✅ Enhanced Input
❌ Experience 系统（太复杂）
❌ 完整的 UI 框架（简化使用）

建议：
- 使用简化的 GameMode
- 避免过度工程化
- 专注核心玩法
- 使用更多蓝图，减少 C++ 复杂度
```

#### 中型团队（6-20人）

**策略：平衡和定制**
```
采用系统：
✅ 完整的 ModularGameplay
✅ GameplayAbilities
⚠️ Experience 系统（简化版）
✅ CommonUI

建议：
- 建立清晰的代码规范
- 投资于工具开发
- 适度使用 Lyra 系统
- 建立自动化流程
```

#### 大型团队（20+人）

**策略：完整采用和扩展**
```
采用系统：
✅ 所有 Lyra 核心系统
✅ GameFeatures 插件系统
✅ 完整的 Experience 系统
✅ 高级网络功能

建议：
- 完全模块化开发
- 多人协作工具
- 专职 DevOps 工程师
- 自定义引擎修改
```

### 11.3 长期维护和更新策略

游戏发布只是开始，长期运营同样重要。

**更新节奏建议：**
```
热修复：紧急 bug 修复（24小时内）
小更新：bug 修复 + 小功能（每 2 周）
中更新：新内容 + 优化（每月）
大更新：新模式 + 大型功能（每季度）
```

**技术架构支持更新：**
```cpp
// VersionCompatibility.h
// 确保旧版本客户端能够处理新数据

UCLASS()
class UVersionCompatibilityManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 检查客户端版本
    UFUNCTION()
    bool IsClientVersionSupported(const FString& ClientVersion) const;

    // 强制更新检查
    UFUNCTION()
    bool RequiresForceUpdate(const FString& ClientVersion) const;

    // 获取最低支持版本
    UFUNCTION()
    FString GetMinimumSupportedVersion() const;

    // 向后兼容的数据序列化
    void SerializeCompatibleData(FArchive& Ar, int32 ClientVersion);
};
```

### 11.4 成功的关键指标

**技术指标：**
- 崩溃率 < 0.1%
- 平均 FPS ≥ 目标帧率的 95%
- 加载时间 < 10 秒（初次启动）
- 网络延迟 < 100ms（多人游戏）

**用户指标：**
- DAU（日活跃用户）持续增长
- 留存率：D1 > 40%, D7 > 20%, D30 > 10%
- ARPU（平均每用户收入）稳定
- 用户评分 > 4.0/5.0

**开发效率指标：**
- 构建时间 < 30 分钟
- 发布新版本周期 < 2 周
- Bug 修复时间 < 3 天（平均）
- 代码审查响应时间 < 24 小时

### 11.5 最后的建议

从 Lyra 到商业游戏是一个系统工程，需要技术、设计、商业的综合考量。以下是最后的几点建议：

1. **持续学习**：UE5 和 Lyra 不断演进，保持学习
2. **社区参与**：加入 UE5 社区，与其他开发者交流
3. **快速迭代**：尽早发布，根据用户反馈改进
4. **用户至上**：技术服务于体验，不要本末倒置
5. **可持续发展**：考虑团队的长期健康，避免过度加班

**最重要的一点：**
> Lyra 是一个工具和起点，而非终点。你的游戏的成功取决于独特的创意、扎实的执行和对玩家需求的深刻理解。

---

## 附录

### A. 推荐资源

**官方文档：**
- [Unreal Engine 5 Documentation](https://docs.unrealengine.com/5.0/)
- [Lyra Sample Game](https://docs.unrealengine.com/5.0/en-US/lyra-sample-game-in-unreal-engine/)
- [Gameplay Ability System](https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-for-unreal-engine/)

**社区资源：**
- UE5 Discord 服务器
- Unreal Slackers Discord
- r/unrealengine (Reddit)

**学习资源：**
- Epic Games 官方培训
- Udemy/Coursera UE5 课程
- YouTube 教程频道

### B. 常用命令参考

```bash
# 构建项目
UnrealBuildTool.exe -projectfiles -project="YourGame.uproject" -game -engine

# 运行自动化测试
UnrealEditor-Cmd "YourGame.uproject" -ExecCmds="Automation RunTests YourGame" -unattended

# 打包游戏（Windows）
RunUAT.bat BuildCookRun -project="YourGame.uproject" -platform=Win64 -clientconfig=Shipping -cook -pak -stage -archive

# 生成 DDC（Derived Data Cache）
UnrealEditor-Cmd "YourGame.uproject" -run=DerivedDataCache -fill

# 清理中间文件
rm -rf Binaries/ Intermediate/ Saved/
```

### C. 性能分析工具

```cpp
// 使用 UE5 内置的性能分析工具

// 1. Stat 命令
stat fps          // 显示帧率
stat unit         // 显示各个线程的时间
stat game         // 游戏线程统计
stat rhi          // 渲染线程统计
stat memory       // 内存使用
stat streaming    // 流式加载

// 2. Profiler
// 编辑器菜单：Window -> Developer Tools -> Session Frontend -> Profiler

// 3. Insights
// UE5 的新一代性能分析工具
// 启动：UnrealInsights.exe

// 4. 代码中的性能标记
SCOPE_CYCLE_COUNTER(STAT_MyFunction);

void MyExpensiveFunction()
{
    SCOPE_CYCLE_COUNTER(STAT_MyFunction);
    // 函数代码
}
```

---

## 结语

从 Lyra 示例项目到成功的商业游戏是一个充满挑战但也充满机遇的旅程。Lyra 为我们提供了坚实的技术基础和现代游戏开发的最佳实践，但真正的成功还需要团队的创意、执行力和对市场的深刻理解。

希望本文能够为你的游戏开发之旅提供有价值的指导。记住，每个成功的游戏都是从一个想法和第一行代码开始的。保持学习，保持创造，祝你的游戏大获成功！

**本文总字数：约 12,500 字**

---

**相关文章：**
- [UE5 Lyra 教程系列索引](../index.md)
- [Lyra 网络多人游戏开发](./28-multiplayer-shooter.md)
- [Lyra 性能优化指南](./29-performance-optimization.md)

**参考资料：**
1. Epic Games. (2024). *Unreal Engine 5 Documentation*.
2. Epic Games. (2024). *Lyra Sample Game Project*.
3. Valve Corporation. (2023). *GDC Talks on Live Service Games*.
4. Supercell. (2023). *Mobile Game Development Best Practices*.

---

*本文档最后更新：2024 年 2 月*
*作者：UE5 Lyra 教程系列团队*
