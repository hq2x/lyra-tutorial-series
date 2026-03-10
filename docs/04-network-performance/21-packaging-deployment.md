# 21. 打包发布：多平台构建指南

## 概述

打包发布是游戏开发的最后一环，也是最容易被低估的环节。一个优秀的游戏如果无法顺利发布到各个平台，或者打包后出现各种奇怪的问题,就会前功尽弃。Lyra 作为一个商业级的样本项目,在打包发布方面提供了完整的解决方案,涵盖了从开发环境到生产环境的完整流程。

本文将深入探讨 Lyra 的打包发布策略,包括:

- **多平台构建配置** - 如何为 Windows、Linux、移动端等平台配置不同的构建目标
- **Dedicated Server 构建** - 专用服务器的构建和部署策略
- **热更新系统** - 如何实现游戏内容的在线更新
- **自动化构建流程** - 使用 CI/CD 提升发布效率
- **性能与安全优化** - 发布版本的关键配置
- **故障排查与调试** - 常见打包问题的解决方案

---

## 1. Lyra 的构建目标架构

### 1.1 Target 文件系统

在 Unreal Engine 中,每个构建目标都由一个 `.Target.cs` 文件定义。Lyra 提供了多个精心设计的 Target 文件:

```
Lyra/Source/
├── LyraGame.Target.cs          # 标准游戏客户端
├── LyraClient.Target.cs        # 纯客户端(无服务器代码)
├── LyraServer.Target.cs        # 标准服务器
├── LyraServerEOS.Target.cs     # 带 Epic Online Services 的服务器
├── LyraServerSteam.Target.cs   # 带 Steam 支持的服务器
├── LyraEditor.Target.cs        # 编辑器目标
├── LyraGameEOS.Target.cs       # 带 EOS 的游戏客户端
└── LyraGameSteam.Target.cs     # 带 Steam 的游戏客户端
```

这种架构设计的优势:

1. **模块化** - 不同平台和需求使用不同的 Target
2. **可扩展** - 容易添加新的平台或在线服务支持
3. **代码隔离** - 客户端不会包含服务器代码,减小包体积
4. **灵活配置** - 每个 Target 可以有独立的编译选项和插件配置

### 1.2 标准游戏 Target 分析

让我们深入分析 `LyraGame.Target.cs`:

```csharp
// LyraGame.Target.cs
using UnrealBuildTool;
using System;
using System.IO;
using EpicGames.Core;
using System.Collections.Generic;
using UnrealBuildBase;
using Microsoft.Extensions.Logging;

public class LyraGameTarget : TargetRules
{
    public LyraGameTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Game;  // 游戏类型目标
        
        // 引入核心游戏模块
        ExtraModuleNames.AddRange(new string[] { "LyraGame" });
        
        // 应用共享的 Lyra Target 设置
        LyraGameTarget.ApplySharedLyraTargetSettings(this);
    }

    // 共享配置方法 - 所有 Lyra Target 都会调用
    internal static void ApplySharedLyraTargetSettings(TargetRules Target)
    {
        ILogger Logger = Target.Logger;
        
        // 使用最新的构建设置版本
        Target.DefaultBuildSettings = BuildSettingsVersion.V7;
        Target.IncludeOrderVersion = EngineIncludeOrderVersion.Latest;

        bool bIsTest = Target.Configuration == UnrealTargetConfiguration.Test;
        bool bIsShipping = Target.Configuration == UnrealTargetConfiguration.Shipping;
        bool bIsDedicatedServer = Target.Type == TargetType.Server;
        
        // 只在 Unique 构建环境中应用高级配置
        if (Target.BuildEnvironment == TargetBuildEnvironment.Unique)
        {
            // 将影子变量警告提升为错误
            Target.CppCompileWarningSettings.ShadowVariableWarningLevel = WarningLevel.Error;

            // 在 Shipping 版本中也启用日志
            Target.bUseLoggingInShipping = true;
            
            // 跟踪 RHI 资源信息用于测试
            Target.bTrackRHIResourceInfoForTest = true;

            if (bIsShipping && !bIsDedicatedServer)
            {
                // Shipping 客户端的安全配置
                
                // 强制验证 HTTPS 证书
                Target.bDisableUnverifiedCertificates = true;

                // 命令行参数白名单(可选)
                // 这可以防止玩家通过命令行注入恶意参数
                //Target.GlobalDefinitions.Add("UE_COMMAND_LINE_USES_ALLOW_LIST=1");
                //Target.GlobalDefinitions.Add("UE_OVERRIDE_COMMAND_LINE_ALLOW_LIST=\"-space -separated -list -of -commands\"");

                // 过滤敏感的命令行参数(可选)
                // 防止敏感信息泄露到日志文件
                //Target.GlobalDefinitions.Add("FILTER_COMMANDLINE_LOGGING=\"-some_connection_id -some_other_arg\"");
            }

            if (bIsShipping || bIsTest)
            {
                // 禁用运行时读取生成的 ini 文件
                // 这是一个重要的安全措施
                Target.bAllowGeneratedIniWhenCooked = false;
                Target.bAllowNonUFSIniWhenCooked = false;
            }

            if (Target.Type != TargetType.Editor)
            {
                // 运行时不需要路径追踪器
                Target.DisablePlugins.Add("OpenImageDenoise");

                // 减少 AssetRegistry 内存占用
                // 但会增加查询的 CPU 开销
                Target.GlobalDefinitions.Add("UE_ASSETREGISTRY_INDIRECT_ASSETDATA_POINTERS=1");
            }

            // 配置 Game Feature 插件
            LyraGameTarget.ConfigureGameFeaturePlugins(Target);
        }
        else
        {
            // Shared 构建环境的限制
            // 无法动态启用/禁用插件,因为要复用已安装的引擎二进制
            if (Target.Type == TargetType.Editor)
            {
                LyraGameTarget.ConfigureGameFeaturePlugins(Target);
            }
            else
            {
                if (!bHasWarnedAboutShared)
                {
                    bHasWarnedAboutShared = true;
                    Logger.LogWarning("LyraGameEOS and dynamic target options are disabled when packaging from an installed version of the engine");
                }
            }
        }
    }
    
    private static bool bHasWarnedAboutShared = false;
}
```

**关键配置解读:**

1. **构建版本控制**
   - `DefaultBuildSettings = V7`: 使用最新的构建设置
   - `IncludeOrderVersion = Latest`: 使用最新的头文件包含顺序规则

2. **安全性强化**
   - `bDisableUnverifiedCertificates`: 强制 HTTPS 证书验证
   - `bAllowGeneratedIniWhenCooked`: 禁止加载非烘焙的 ini 文件
   - 命令行参数白名单机制

3. **性能优化**
   - 禁用不需要的插件(如 OpenImageDenoise)
   - AssetRegistry 内存优化

4. **调试支持**
   - `bUseLoggingInShipping`: 即使在 Shipping 版本也保留日志
   - `bTrackRHIResourceInfoForTest`: 跟踪图形资源用于性能分析

### 1.3 Dedicated Server Target

服务器目标的配置更加精简:

```csharp
// LyraServer.Target.cs
using UnrealBuildTool;
using System.Collections.Generic;

[SupportedPlatforms(UnrealPlatformClass.Server)]
public class LyraServerTarget : TargetRules
{
    public LyraServerTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Server;  // 服务器类型

        ExtraModuleNames.AddRange(new string[] { "LyraGame" });

        // 复用共享配置
        LyraGameTarget.ApplySharedLyraTargetSettings(this);

        // 服务器特有配置:在 Shipping 版本启用额外检查
        bUseChecksInShipping = true;
    }
}
```

**服务器配置特点:**

1. **平台限制** - `[SupportedPlatforms(UnrealPlatformClass.Server)]` 只允许服务器平台
2. **运行时检查** - `bUseChecksInShipping = true` 在 Shipping 版本保留 `check()` 断言
3. **无图形依赖** - 自动排除渲染相关代码

### 1.4 Client Target

纯客户端目标不包含服务器代码:

```csharp
// LyraClient.Target.cs
using UnrealBuildTool;
using System.Collections.Generic;

[SupportedPlatforms(UnrealPlatformClass.Desktop)]
public class LyraClientTarget : TargetRules
{
    public LyraClientTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Client;  // 客户端类型
        
        ExtraModuleNames.AddRange(new string[] { "LyraGame" });
        
        LyraGameTarget.ApplySharedLyraTargetSettings(this);
    }
}
```

**使用场景:**

- **Listen Server 不需要** - 如果游戏只有专用服务器模式
- **减小包体积** - 移除所有服务器代码
- **提高安全性** - 客户端无法访问服务器逻辑

---

## 2. Game Feature 插件的构建管理

Lyra 的一个核心设计是 Game Features 插件系统。在打包时,需要精确控制哪些插件被编译进最终版本。

### 2.1 插件构建策略

`ConfigureGameFeaturePlugins()` 方法实现了智能的插件管理:

```csharp
static public void ConfigureGameFeaturePlugins(TargetRules Target)
{
    ILogger Logger = Target.Logger;
    Log.TraceInformationOnce("Compiling GameFeaturePlugins in branch {0}", Target.Version.BranchName);

    bool bBuildAllGameFeaturePlugins = ShouldEnableAllGameFeaturePlugins(Target);

    // 1. 发现所有 Game Feature 插件
    List<FileReference> CombinedPluginList = new List<FileReference>();
    List<DirectoryReference> GameFeaturePluginRoots = 
        Unreal.GetExtensionDirs(Target.ProjectFile.Directory, Path.Combine("Plugins", "GameFeatures"));
    
    foreach (DirectoryReference SearchDir in GameFeaturePluginRoots)
    {
        CombinedPluginList.AddRange(PluginsBase.EnumeratePlugins(SearchDir));
    }

    // 2. 解析每个插件的 .uplugin 文件
    foreach (FileReference PluginFile in CombinedPluginList)
    {
        if (PluginFile != null && FileReference.Exists(PluginFile))
        {
            bool bEnabled = false;
            bool bForceDisabled = false;
            
            try
            {
                JsonObject RawObject = JsonObject.Read(PluginFile);

                // 验证 EnabledByDefault
                bool bEnabledByDefault = false;
                if (!RawObject.TryGetBoolField("EnabledByDefault", out bEnabledByDefault) 
                    || bEnabledByDefault == true)
                {
                    // Game Feature 插件应该默认禁用
                    // 这样可以避免插件名嵌入到可执行文件中
                }

                // 验证 ExplicitlyLoaded
                bool bExplicitlyLoaded = false;
                if (!RawObject.TryGetBoolField("ExplicitlyLoaded", out bExplicitlyLoaded) 
                    || bExplicitlyLoaded == false)
                {
                    Logger.LogWarning("GameFeaturePlugin {0} does not set ExplicitlyLoaded to true.", 
                        PluginFile.GetFileNameWithoutExtension());
                }

                // 决定是否启用
                if (bBuildAllGameFeaturePlugins)
                {
                    bEnabled = true;
                }

                // 3. 处理特殊标记

                // EditorOnly 插件
                bool bEditorOnly = false;
                if (RawObject.TryGetBoolField("EditorOnly", out bEditorOnly))
                {
                    if (bEditorOnly && Target.Type != TargetType.Editor)
                    {
                        bForceDisabled = true;
                    }
                }

                // 分支限制
                string RestrictToBranch;
                if (RawObject.TryGetStringField("RestrictToBranch", out RestrictToBranch))
                {
                    if (!Target.Version.BranchName.Equals(RestrictToBranch, 
                        StringComparison.OrdinalIgnoreCase))
                    {
                        bForceDisabled = true;
                        Logger.LogDebug("Plugin {0} restricted to other branches", 
                            PluginFile.GetFileNameWithoutExtension());
                    }
                }

                // NeverBuild 标记
                bool bNeverBuild = false;
                if (RawObject.TryGetBoolField("NeverBuild", out bNeverBuild) && bNeverBuild)
                {
                    bForceDisabled = true;
                    Logger.LogDebug("Plugin {0} marked as NeverBuild", 
                        PluginFile.GetFileNameWithoutExtension());
                }

                // 4. 应用决策
                if (bForceDisabled)
                {
                    bEnabled = false;
                }

                if (bEnabled)
                {
                    Target.EnablePlugins.Add(PluginFile.GetFileNameWithoutExtension());
                }
                else if (bForceDisabled)
                {
                    Target.DisablePlugins.Add(PluginFile.GetFileNameWithoutExtension());
                }
            }
            catch (Exception ParseException)
            {
                Logger.LogWarning("Failed to parse plugin {0}: {1}", 
                    PluginFile.GetFileNameWithoutExtension(), ParseException.Message);
                bForceDisabled = true;
            }
        }
    }
}
```

### 2.2 插件描述文件配置

一个标准的 Game Feature 插件 `.uplugin` 文件应该包含这些字段:

```json
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "1.0",
    "FriendlyName": "ShooterCore",
    
    // ===== 核心配置 =====
    
    // 默认禁用 - 由 Experience 动态加载
    "EnabledByDefault": false,
    
    // 显式加载 - 不随引擎自动加载
    "ExplicitlyLoaded": true,
    
    // ===== 可选的控制字段 =====
    
    // 仅编辑器插件
    "EditorOnly": false,
    
    // 限制到特定分支
    "RestrictToBranch": "main",
    
    // 从不构建
    "NeverBuild": false,
    
    // 自定义发布版本标记(可扩展)
    "MyProjectReleaseVersion": "1.5.0",
    
    // ===== 标准插件字段 =====
    
    "Category": "Game Features",
    "Description": "Shooter game mode implementation",
    
    "Modules": [
        {
            "Name": "ShooterCoreRuntime",
            "Type": "Runtime",
            "LoadingPhase": "Default"
        }
    ],
    
    "Plugins": [
        {
            "Name": "GameFeatures",
            "Enabled": true
        },
        {
            "Name": "ModularGameplayActors",
            "Enabled": true
        }
    ]
}
```

### 2.3 实战案例:按发布版本控制插件

假设你的项目有多个发布版本,某些功能只在特定版本启用:

```csharp
// 扩展 ConfigureGameFeaturePlugins
static public void ConfigureGameFeaturePlugins(TargetRules Target)
{
    // ... 前面的代码 ...

    // 自定义:读取插件的发布版本要求
    string PluginReleaseVersion;
    if (RawObject.TryGetStringField("MyProjectReleaseVersion", out PluginReleaseVersion))
    {
        // 当前构建的目标版本
        string CurrentReleaseVersion = GetCurrentReleaseVersion();
        
        // 比较版本
        if (CompareVersions(PluginReleaseVersion, CurrentReleaseVersion) <= 0)
        {
            // 插件版本要求 <= 当前版本,可以启用
            bEnabled = true;
        }
        else
        {
            // 插件是未来版本的功能,禁用
            Logger.LogInformation("Plugin {0} requires version {1}, current is {2}", 
                PluginFile.GetFileNameWithoutExtension(), 
                PluginReleaseVersion, 
                CurrentReleaseVersion);
            bForceDisabled = true;
        }
    }
}

static string GetCurrentReleaseVersion()
{
    // 从环境变量、配置文件或版本控制系统读取
    string version = Environment.GetEnvironmentVariable("PROJECT_RELEASE_VERSION");
    return version ?? "1.0.0";
}

static int CompareVersions(string v1, string v2)
{
    // 简单的版本比较实现
    var parts1 = v1.Split('.').Select(int.Parse).ToArray();
    var parts2 = v2.Split('.').Select(int.Parse).ToArray();
    
    for (int i = 0; i < Math.Min(parts1.Length, parts2.Length); i++)
    {
        if (parts1[i] != parts2[i])
            return parts1[i].CompareTo(parts2[i]);
    }
    return parts1.Length.CompareTo(parts2.Length);
}
```

这样,你可以在 CI/CD 流程中通过环境变量控制打包内容:

```bash
# 打包 1.0 版本 - 只包含基础功能
export PROJECT_RELEASE_VERSION=1.0.0
RunUAT BuildCookRun -project=Lyra.uproject -platform=Win64 ...

# 打包 1.5 版本 - 包含更多功能
export PROJECT_RELEASE_VERSION=1.5.0
RunUAT BuildCookRun -project=Lyra.uproject -platform=Win64 ...
```

---

## 3. 多平台打包配置

### 3.1 Windows 平台

#### 3.1.1 基础配置

在项目设置中(`Project Settings > Platforms > Windows`):

```ini
[/Script/WindowsTargetPlatform.WindowsTargetSettings]
; 默认 RHI (DirectX 12)
DefaultGraphicsRHI=DefaultGraphicsRHI_DX12

; 目标 Windows 版本
TargetedRHIs=PCD3D_SM6

; 音频配置
AudioSampleRate=48000
AudioCallbackBufferFrameSize=1024
AudioNumBuffersToEnqueue=2

; Shader 编译
bBuildForD3D12RayTracing=False
bCompileDynamicRHI=True

; 压缩设置
bCompressed=True
```

#### 3.1.2 打包命令

使用 UAT (Unreal Automation Tool) 打包:

```bash
# 基础打包命令
RunUAT BuildCookRun \
    -project="C:/Projects/Lyra/Lyra.uproject" \
    -platform=Win64 \
    -clientconfig=Shipping \
    -cook \
    -stage \
    -package \
    -archive \
    -archivedirectory="C:/Builds/Lyra_Win64" \
    -noP4 \
    -utf8output

# 完整参数说明:
# -project: 项目文件路径
# -platform: 目标平台
# -clientconfig: 构建配置 (Development/Shipping/Test)
# -cook: 烘焙内容
# -stage: 准备文件到 Staging 目录
# -package: 打包为可执行文件
# -archive: 创建分发包
# -archivedirectory: 输出目录
# -noP4: 不使用 Perforce
# -utf8output: 使用 UTF-8 输出

# 附加选项:
# -clean: 清理旧的中间文件
# -allmaps: 烘焙所有地图
# -compressed: 压缩 PAK 文件
# -distribution: 发布版本(移除开发工具)
# -build: 重新编译代码
# -prereqs: 包含 DX/VC++ 安装程序
```

#### 3.1.3 Shipping 配置优化

为 Shipping 版本创建专用的配置文件 `Config/Windows/WindowsEngine.ini`:

```ini
[/Script/Engine.Engine]
; 禁用调试工具
bDisableAllScreenMessages=True
bEnableOnScreenDebugMessages=False

[/Script/Engine.GameEngine]
; 服务器超时设置
ServerConnectionTimeout=60.0
ServerGracePeriodBeforeHandlingConnectionErrors=10.0

[SystemSettings]
; 图形设置
r.VSync=0
r.FinishCurrentFrame=0

; 网络设置
net.MaxRepArraySize=2048
net.MaxRepArrayMemory=65536

; 性能设置
gc.TimeBetweenPurgingPendingKillObjects=60
s.AsyncLoadingTimeLimit=10.0

[Core.System]
; 内存配置
MemorySize=8192MB
CacheSizeMegs=256

[ConsoleVariables]
; Shipping 版本的 Console 变量
t.MaxFPS=144
r.Streaming.PoolSize=2000
```

### 3.2 Linux 服务器

#### 3.2.1 交叉编译设置

在 Windows 上为 Linux 打包,需要安装交叉编译工具链:

1. 下载 Linux 工具链:
   - UE5.3+: `v22_clang-16.0.6-centos7`
   - 解压到 `C:\UnrealToolchains\`

2. 设置环境变量:
   ```bash
   LINUX_MULTIARCH_ROOT=C:\UnrealToolchains\v22_clang-16.0.6-centos7\x86_64-unknown-linux-gnu
   ```

3. 验证工具链:
   ```bash
   # 在 UE 项目目录运行
   RunUAT BuildCookRun -project=Lyra.uproject -platform=Linux -help
   ```

#### 3.2.2 Linux Server 打包

```bash
# 打包 Linux 专用服务器
RunUAT BuildCookRun \
    -project="C:/Projects/Lyra/Lyra.uproject" \
    -target=LyraServer \
    -platform=Linux \
    -serverconfig=Shipping \
    -cook \
    -stage \
    -package \
    -archive \
    -archivedirectory="C:/Builds/Lyra_LinuxServer" \
    -noP4 \
    -utf8output \
    -server \
    -serverplatform=Linux \
    -noclient

# 参数说明:
# -target=LyraServer: 使用服务器 Target
# -serverconfig: 服务器配置
# -server: 构建服务器版本
# -noclient: 不构建客户端
```

#### 3.2.3 服务器配置文件

`Config/Linux/LinuxEngine.ini`:

```ini
[Core.System]
; 使用所有 CPU 核心
Paths=../../../Engine/Content
Paths=%GAMEDIR%Content

[/Script/Engine.Engine]
; 服务器特定设置
bUseFixedFrameRate=True
FixedFrameRate=60.0

[SystemSettings]
; 无头渲染
r.GraphicsAdapter=-1

; 网络优化
net.MaxRepArraySize=4096
net.MaxRepArrayMemory=131072

; Tick 优化
t.MaxFPS=60

[/Script/OnlineSubsystemUtils.IpNetDriver]
; 网络驱动配置
MaxClientRate=25000
MaxInternetClientRate=25000
NetServerMaxTickRate=60
LanServerMaxTickRate=120

; 连接数限制
MaxConnections=100
ConnectionTimeout=60.0
```

#### 3.2.4 服务器启动脚本

`start_server.sh`:

```bash
#!/bin/bash

# Lyra Linux Server 启动脚本

# 服务器配置
SERVER_NAME="Lyra Server"
MAP_NAME="/Game/Experiences/B_LyraShooterGame_Elimination"
MAX_PLAYERS=16
PORT=7777

# 日志目录
LOG_DIR="./Logs"
mkdir -p $LOG_DIR

# 启动服务器
./LyraServer.sh \
    $MAP_NAME \
    -server \
    -log \
    -port=$PORT \
    -MaxPlayers=$MAX_PLAYERS \
    -NoSteam \
    -stdout \
    -unattended \
    -FullStdOutLogOutput \
    -SessionName="$SERVER_NAME" \
    > $LOG_DIR/server_$(date +%Y%m%d_%H%M%S).log 2>&1 &

echo "Server started with PID $!"
echo "Logs: $LOG_DIR"
```

### 3.3 移动平台

#### 3.3.1 Android 配置

`Config/Android/AndroidEngine.ini`:

```ini
[/Script/AndroidRuntimeSettings.AndroidRuntimeSettings]
; 包信息
PackageName=com.epicgames.lyra
VersionDisplayName=1.0.0
MinSDKVersion=26
TargetSDKVersion=33

; 图形设置
bSupportsVulkan=True
bBuildForES31=True

; 性能设置
bPackageForMetaQuest=False
bEnableGooglePlaySupport=False

; 权限
bEnablePermissionForLocation=False
bEnablePermissionForMicrophone=True

[SystemSettings]
; 移动端图形设置
r.Mobile.Forward.EnableLocalLights=1
r.Mobile.EnableStaticAndCSMShadowReceivers=1
r.Mobile.AntiAliasing=2

; 性能限制
r.Mobile.ContentScaleFactor=1.0
r.MobileContentScaleFactor=1.0
```

打包命令:

```bash
RunUAT BuildCookRun \
    -project="C:/Projects/Lyra/Lyra.uproject" \
    -platform=Android \
    -clientconfig=Shipping \
    -cook \
    -stage \
    -package \
    -archive \
    -distribution \
    -architectures=arm64 \
    -cookflavor=ASTC \
    -archivedirectory="C:/Builds/Lyra_Android"
```

#### 3.3.2 iOS 配置

`Config/IOS/IOSEngine.ini`:

```ini
[/Script/IOSRuntimeSettings.IOSRuntimeSettings]
; Bundle 信息
BundleIdentifier=com.epicgames.lyra
VersionInfo=1.0.0
BundleDisplayName=Lyra
BundleName=Lyra

; 最低 iOS 版本
MinimumiOSVersion=IOS_15

; 图形设置
bSupportsMetalMRT=True
bUseIntegratedKeyboard=True

; 性能
bBuildAsFramework=False
```

### 3.4 配置文件分层系统

Unreal 的配置系统是分层的,打包时会合并多个配置文件:

```
优先级(从低到高):
1. Engine/Config/BaseEngine.ini          (引擎基础配置)
2. Engine/Config/[Platform]/[Platform]Engine.ini
3. [Project]/Config/DefaultEngine.ini    (项目默认配置)
4. [Project]/Config/[Platform]/[Platform]Engine.ini
5. [User]/Saved/Config/[Platform]/Engine.ini  (用户配置,不打包)
```

**打包时的配置策略:**

1. **开发配置** (`Config/DefaultEngine.ini`)
   - 宽松的设置,便于调试
   - 包含开发工具和调试功能

2. **平台配置** (`Config/Windows/WindowsEngine.ini`)
   - 平台特定的优化
   - 覆盖默认配置

3. **Shipping 覆盖**
   - 在打包脚本中通过 `-ExecCmds` 传递额外的设置
   - 或使用 Config Hierarchy

示例:创建 `Config/Shipping/ShippingEngine.ini`:

```ini
[Configuration]
; 这是 Shipping 特定的配置
BasedOn=../DefaultEngine.ini

[/Script/Engine.Engine]
; 禁用所有调试功能
bDisableAllScreenMessages=True
bEnableOnScreenDebugMessages=False
bEnableVisualLogRecordingOnStart=False

[ConsoleVariables]
; 性能优化
r.Streaming.PoolSize=3000
r.Streaming.MaxNumTexturesToStreamPerFrame=20
```

然后在打包时指定:

```bash
RunUAT BuildCookRun \
    -ini:"Engine=[Configuration]:ActiveGameNameRedirects=(\"Shipping\",\"../Shipping/ShippingEngine.ini\")" \
    ...其他参数...
```

---

## 4. Dedicated Server 深度指南

### 4.1 Server Target 高级配置

#### 4.1.1 性能优化配置

`LyraServer.Target.cs` 可以进一步优化:

```csharp
public class LyraServerTarget : TargetRules
{
    public LyraServerTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Server;
        
        ExtraModuleNames.AddRange(new string[] { "LyraGame" });
        
        LyraGameTarget.ApplySharedLyraTargetSettings(this);
        
        // 服务器特定优化
        bUseChecksInShipping = true;
        
        // 禁用客户端专用插件
        DisablePlugins.AddRange(new string[]
        {
            "SteamAudio",           // 空间音频
            "WaveVR",               // VR 支持
            "Oculus",
            "OnlineSubsystemSteam", // 如果服务器不需要 Steam 集成
            "MoviePlayer",          // 过场动画播放器
            "SlateRemote",          // UI 远程调试
            "BlueprintFileUtils"    // 开发工具
        });
        
        // 启用服务器专用插件
        EnablePlugins.AddRange(new string[]
        {
            "OnlineSubsystemUtils",
            "OnlineSubsystemEOS",   // 如果使用 EOS
            "GameplayAbilities"
        });
        
        // 内存优化
        GlobalDefinitions.Add("UE_SERVER=1");
        GlobalDefinitions.Add("UE_BUILD_SHIPPING_WITH_EDITOR=0");
        
        // 网络优化
        bCompileAgainstEngine = true;
        bUseLoggingInShipping = true;
        
        // 调试支持
        if (Target.Configuration == UnrealTargetConfiguration.Shipping)
        {
            // Shipping 服务器保留符号信息用于崩溃报告
            bDebugBuildsActuallyUseDebugCRT = false;
            bDisableDebugInfo = false;
        }
    }
}
```

#### 4.1.2 服务器专用的 Game Mode

创建一个服务器优化的 Game Mode:

```cpp
// LyraDedicatedServerGameMode.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameModeBase.h"
#include "LyraDedicatedServerGameMode.generated.h"

/**
 * 专用服务器的 Game Mode
 * 移除了所有客户端相关逻辑,专注于服务器性能
 */
UCLASS()
class LYRAGAME_API ALyraDedicatedServerGameMode : public AGameModeBase
{
    GENERATED_BODY()

public:
    ALyraDedicatedServerGameMode();

    //~ Begin AGameModeBase Interface
    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;
    virtual void PreLogin(const FString& Options, const FString& Address, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage) override;
    virtual APlayerController* Login(UPlayer* NewPlayer, ENetRole InRemoteRole, const FString& Portal, const FString& Options, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage) override;
    virtual void PostLogin(APlayerController* NewPlayer) override;
    virtual void Logout(AController* Exiting) override;
    //~ End AGameModeBase Interface

protected:
    // 服务器监控
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaSeconds) override;

    // 性能监控
    UFUNCTION()
    void MonitorServerPerformance();

    // 服务器状态
    UPROPERTY()
    int32 CurrentPlayerCount;

    UPROPERTY()
    float AverageTickTime;

    UPROPERTY()
    float PeakTickTime;

    // 性能阈值
    UPROPERTY(EditDefaultsOnly, Category = "Server")
    float TickTimeWarningThreshold;

    UPROPERTY(EditDefaultsOnly, Category = "Server")
    int32 MaxPlayers;

private:
    FTimerHandle PerformanceMonitorTimer;
    TArray<float> RecentTickTimes;
};
```

```cpp
// LyraDedicatedServerGameMode.cpp
#include "LyraDedicatedServerGameMode.h"
#include "Engine/World.h"
#include "TimerManager.h"
#include "GameFramework/PlayerController.h"
#include "GameFramework/PlayerState.h"
#include "Net/UnrealNetwork.h"

ALyraDedicatedServerGameMode::ALyraDedicatedServerGameMode()
{
    PrimaryActorTick.bCanEverTick = true;
    PrimaryActorTick.TickInterval = 1.0f; // 每秒 Tick 一次即可
    
    TickTimeWarningThreshold = 33.0f; // 33ms = ~30 FPS
    MaxPlayers = 32;
    
    CurrentPlayerCount = 0;
    AverageTickTime = 0.0f;
    PeakTickTime = 0.0f;
}

void ALyraDedicatedServerGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);

    UE_LOG(LogTemp, Log, TEXT("[Server] Initializing dedicated server on map: %s"), *MapName);

    // 解析服务器启动选项
    MaxPlayers = UGameplayStatics::GetIntOption(Options, TEXT("MaxPlayers"), MaxPlayers);
    
    // 记录服务器配置
    UE_LOG(LogTemp, Log, TEXT("[Server] Configuration:"));
    UE_LOG(LogTemp, Log, TEXT("  - MaxPlayers: %d"), MaxPlayers);
    UE_LOG(LogTemp, Log, TEXT("  - TickWarningThreshold: %.1f ms"), TickTimeWarningThreshold);

    // 启动性能监控
    GetWorldTimerManager().SetTimer(
        PerformanceMonitorTimer,
        this,
        &ALyraDedicatedServerGameMode::MonitorServerPerformance,
        10.0f, // 每10秒检查一次
        true
    );
}

void ALyraDedicatedServerGameMode::PreLogin(const FString& Options, const FString& Address, 
    const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    Super::PreLogin(Options, Address, UniqueId, ErrorMessage);

    // 检查服务器是否已满
    if (CurrentPlayerCount >= MaxPlayers)
    {
        ErrorMessage = TEXT("Server is full");
        UE_LOG(LogTemp, Warning, TEXT("[Server] PreLogin rejected: Server full (%d/%d)"), 
            CurrentPlayerCount, MaxPlayers);
        return;
    }

    // 可以在这里添加额外的验证逻辑
    // 例如:Ban 列表检查、白名单验证等

    UE_LOG(LogTemp, Log, TEXT("[Server] PreLogin accepted: %s from %s"), 
        *UniqueId.ToString(), *Address);
}

APlayerController* ALyraDedicatedServerGameMode::Login(UPlayer* NewPlayer, ENetRole InRemoteRole, 
    const FString& Portal, const FString& Options, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    APlayerController* NewPC = Super::Login(NewPlayer, InRemoteRole, Portal, Options, UniqueId, ErrorMessage);

    if (NewPC)
    {
        CurrentPlayerCount++;
        UE_LOG(LogTemp, Log, TEXT("[Server] Player logged in: %s (Total: %d/%d)"), 
            *UniqueId.ToString(), CurrentPlayerCount, MaxPlayers);
    }

    return NewPC;
}

void ALyraDedicatedServerGameMode::PostLogin(APlayerController* NewPlayer)
{
    Super::PostLogin(NewPlayer);

    // 服务器欢迎消息
    if (NewPlayer && NewPlayer->PlayerState)
    {
        FString PlayerName = NewPlayer->PlayerState->GetPlayerName();
        UE_LOG(LogTemp, Log, TEXT("[Server] PostLogin: %s"), *PlayerName);
        
        // 可以在这里发送欢迎消息或初始化玩家数据
    }
}

void ALyraDedicatedServerGameMode::Logout(AController* Exiting)
{
    if (Exiting && Exiting->PlayerState)
    {
        FString PlayerName = Exiting->PlayerState->GetPlayerName();
        UE_LOG(LogTemp, Log, TEXT("[Server] Player logged out: %s"), *PlayerName);
    }

    CurrentPlayerCount = FMath::Max(0, CurrentPlayerCount - 1);
    
    Super::Logout(Exiting);

    UE_LOG(LogTemp, Log, TEXT("[Server] Current player count: %d/%d"), 
        CurrentPlayerCount, MaxPlayers);
}

void ALyraDedicatedServerGameMode::BeginPlay()
{
    Super::BeginPlay();

    UE_LOG(LogTemp, Log, TEXT("[Server] Dedicated Server started successfully"));
}

void ALyraDedicatedServerGameMode::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);

    // 记录 Tick 时间
    float TickTime = DeltaSeconds * 1000.0f; // 转换为毫秒
    RecentTickTimes.Add(TickTime);
    
    // 只保留最近 60 秒的数据
    if (RecentTickTimes.Num() > 60)
    {
        RecentTickTimes.RemoveAt(0);
    }

    // 更新峰值
    if (TickTime > PeakTickTime)
    {
        PeakTickTime = TickTime;
    }

    // 警告:Tick 时间过长
    if (TickTime > TickTimeWarningThreshold)
    {
        UE_LOG(LogTemp, Warning, TEXT("[Server] High tick time detected: %.2f ms"), TickTime);
    }
}

void ALyraDedicatedServerGameMode::MonitorServerPerformance()
{
    // 计算平均 Tick 时间
    if (RecentTickTimes.Num() > 0)
    {
        float Sum = 0.0f;
        for (float Time : RecentTickTimes)
        {
            Sum += Time;
        }
        AverageTickTime = Sum / RecentTickTimes.Num();
    }

    // 输出性能报告
    UE_LOG(LogTemp, Display, TEXT("[Server Performance Report]"));
    UE_LOG(LogTemp, Display, TEXT("  Players: %d/%d"), CurrentPlayerCount, MaxPlayers);
    UE_LOG(LogTemp, Display, TEXT("  Avg Tick: %.2f ms"), AverageTickTime);
    UE_LOG(LogTemp, Display, TEXT("  Peak Tick: %.2f ms"), PeakTickTime);
    UE_LOG(LogTemp, Display, TEXT("  Target FPS: %.1f"), 1000.0f / AverageTickTime);

    // 重置峰值
    PeakTickTime = 0.0f;

    // 内存统计
    FPlatformMemoryStats MemoryStats = FPlatformMemory::GetStats();
    UE_LOG(LogTemp, Display, TEXT("  Memory Used: %.2f MB"), 
        MemoryStats.UsedPhysical / (1024.0f * 1024.0f));
    UE_LOG(LogTemp, Display, TEXT("  Memory Available: %.2f MB"), 
        MemoryStats.AvailablePhysical / (1024.0f * 1024.0f));

    // 网络统计
    if (UWorld* World = GetWorld())
    {
        if (UNetDriver* NetDriver = World->GetNetDriver())
        {
            UE_LOG(LogTemp, Display, TEXT("  Active Connections: %d"), 
                NetDriver->ClientConnections.Num());
            
            // 带宽统计
            float TotalInKBps = NetDriver->InBytesPerSecond / 1024.0f;
            float TotalOutKBps = NetDriver->OutBytesPerSecond / 1024.0f;
            UE_LOG(LogTemp, Display, TEXT("  Bandwidth In: %.2f KB/s"), TotalInKBps);
            UE_LOG(LogTemp, Display, TEXT("  Bandwidth Out: %.2f KB/s"), TotalOutKBps);
        }
    }
}
```

### 4.2 服务器配置管理

#### 4.2.1 命令行参数

服务器可以通过命令行参数配置:

```bash
./LyraServer.sh \
    /Game/Maps/MainMap \
    -server \
    -log \
    -port=7777 \
    -MaxPlayers=32 \
    -ServerPassword="secret" \
    -NoSteam \
    -SessionName="My Lyra Server" \
    -ExecCmds="t.MaxFPS 60, net.MaxRepArraySize 4096" \
    -ini:Engine:[SystemSettings]:r.GraphicsAdapter=-1
```

**常用参数:**

| 参数 | 说明 | 示例 |
|------|------|------|
| `[MapName]` | 启动地图 | `/Game/Maps/Arena` |
| `-server` | 服务器模式 | - |
| `-log` | 启用日志输出 | - |
| `-port=N` | 监听端口 | `-port=7777` |
| `-MaxPlayers=N` | 最大玩家数 | `-MaxPlayers=64` |
| `-SessionName="X"` | 会话名称 | `-SessionName="Official"` |
| `-NoSteam` | 禁用 Steam | - |
| `-stdout` | 标准输出日志 | - |
| `-unattended` | 无人值守模式 | - |
| `-ExecCmds="X"` | 执行控制台命令 | `-ExecCmds="t.MaxFPS 60"` |

#### 4.2.2 配置文件

创建 `Config/Server/ServerEngine.ini`:

```ini
[/Script/Engine.Engine]
; 服务器网络设置
bUseFixedFrameRate=True
FixedFrameRate=60.0
bSmoothFrameRate=False

[/Script/OnlineSubsystemUtils.IpNetDriver]
; 网络驱动
NetConnectionClassName="/Script/OnlineSubsystemUtils.IpConnection"
MaxClientRate=35000
MaxInternetClientRate=25000
RelevantTimeout=5.0
KeepAliveTime=0.2
ConnectionTimeout=60.0
InitialConnectTimeout=60.0

; Tick 频率
NetServerMaxTickRate=60
LanServerMaxTickRate=60

; 优先级
AllowDownloads=False
bClampListenServerTickRate=False

[/Script/Engine.GameNetworkManager]
; 网络管理
TotalNetBandwidth=200000
MaxDynamicBandwidth=100000
MinDynamicBandwidth=10000

; 连接管理
MoveRepSize=256.0
MAXPOSITIONERRORSQUARED=625.0
MAXNEARZEROVELOCITYSQUARED=9.0
CLIENTADJUSTUPDATECOST=180.0
MAXCLIENTUPDATEINTERVAL=0.25

; 站位校正
StandbyRxCheatTime=30.0
StandbyTxCheatTime=30.0
PercentMissingForRxStandby=85.0
PercentMissingForTxStandby=85.0
PercentForBadPing=80.0
JoinInProgressStandbyWaitTime=30.0

[SystemSettings]
; 图形设置 (服务器不需要渲染)
r.GraphicsAdapter=-1
r.AllowOcclusionQueries=0
r.VSync=0

; 网络优化
net.MaxRepArraySize=4096
net.MaxRepArrayMemory=131072

; Tick 优化
t.MaxFPS=60

[ConsoleVariables]
; 控制台变量
net.IpNetDriver.MaxChannelsOverride=1024
```

### 4.3 服务器管理工具

#### 4.3.1 RCON 集成

创建一个简单的 RCON (Remote Console) 系统:

```cpp
// LyraServerRCON.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "Sockets.h"
#include "LyraServerRCON.generated.h"

/**
 * 简单的 RCON (Remote Console) 服务器
 * 允许远程管理员执行控制台命令
 */
UCLASS()
class LYRAGAME_API ULyraServerRCON : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    //~ Begin USubsystem Interface
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
    //~ End USubsystem Interface

    // 启动 RCON 服务器
    UFUNCTION(BlueprintCallable, Category = "Server")
    bool StartRCONServer(int32 Port, const FString& Password);

    // 停止 RCON 服务器
    UFUNCTION(BlueprintCallable, Category = "Server")
    void StopRCONServer();

protected:
    // 监听线程
    void ListenForConnections();

    // 处理客户端连接
    void HandleClient(FSocket* ClientSocket);

    // 验证密码
    bool AuthenticateClient(const FString& Password);

    // 执行命令
    FString ExecuteCommand(const FString& Command);

private:
    FSocket* ListenSocket;
    FString RCONPassword;
    TArray<FSocket*> ConnectedClients;
    bool bIsRunning;
    FRunnableThread* ListenThread;
};
```

```cpp
// LyraServerRCON.cpp
#include "LyraServerRCON.h"
#include "SocketSubsystem.h"
#include "Engine/Engine.h"
#include "Misc/OutputDeviceNull.h"

void ULyraServerRCON::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    ListenSocket = nullptr;
    bIsRunning = false;

    UE_LOG(LogTemp, Log, TEXT("[RCON] Subsystem initialized"));
}

void ULyraServerRCON::Deinitialize()
{
    StopRCONServer();
    Super::Deinitialize();

    UE_LOG(LogTemp, Log, TEXT("[RCON] Subsystem deinitialized"));
}

bool ULyraServerRCON::StartRCONServer(int32 Port, const FString& Password)
{
    if (bIsRunning)
    {
        UE_LOG(LogTemp, Warning, TEXT("[RCON] Server already running"));
        return false;
    }

    RCONPassword = Password;

    // 创建 Socket
    ISocketSubsystem* SocketSubsystem = ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM);
    ListenSocket = SocketSubsystem->CreateSocket(NAME_Stream, TEXT("RCON"), false);

    if (!ListenSocket)
    {
        UE_LOG(LogTemp, Error, TEXT("[RCON] Failed to create socket"));
        return false;
    }

    // 绑定端口
    TSharedRef<FInternetAddr> Addr = SocketSubsystem->CreateInternetAddr();
    Addr->SetPort(Port);
    Addr->SetAnyAddress();

    if (!ListenSocket->Bind(*Addr))
    {
        UE_LOG(LogTemp, Error, TEXT("[RCON] Failed to bind to port %d"), Port);
        ListenSocket->Close();
        ListenSocket = nullptr;
        return false;
    }

    // 开始监听
    if (!ListenSocket->Listen(8))
    {
        UE_LOG(LogTemp, Error, TEXT("[RCON] Failed to listen"));
        ListenSocket->Close();
        ListenSocket = nullptr;
        return false;
    }

    bIsRunning = true;

    UE_LOG(LogTemp, Log, TEXT("[RCON] Server started on port %d"), Port);

    // 启动监听线程
    // 注意:这里简化了实现,实际项目应该使用 FRunnable
    // 这里仅作示例,不应在生产环境使用

    return true;
}

void ULyraServerRCON::StopRCONServer()
{
    if (!bIsRunning)
    {
        return;
    }

    bIsRunning = false;

    // 关闭所有客户端连接
    for (FSocket* ClientSocket : ConnectedClients)
    {
        ClientSocket->Close();
        ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM)->DestroySocket(ClientSocket);
    }
    ConnectedClients.Empty();

    // 关闭监听 Socket
    if (ListenSocket)
    {
        ListenSocket->Close();
        ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM)->DestroySocket(ListenSocket);
        ListenSocket = nullptr;
    }

    UE_LOG(LogTemp, Log, TEXT("[RCON] Server stopped"));
}

FString ULyraServerRCON::ExecuteCommand(const FString& Command)
{
    // 执行控制台命令并捕获输出
    FOutputDeviceNull OutputDevice;
    
    if (GEngine)
    {
        GEngine->Exec(GetWorld(), *Command, OutputDevice);
    }

    // 返回执行结果
    return FString::Printf(TEXT("Command executed: %s"), *Command);
}
```

#### 4.3.2 使用示例

在服务器启动时初始化 RCON:

```cpp
// 在 GameInstance 或 GameMode 的 BeginPlay 中
void ALyraDedicatedServerGameMode::BeginPlay()
{
    Super::BeginPlay();

    // 启动 RCON 服务器
    if (UGameInstance* GameInstance = GetGameInstance())
    {
        if (ULyraServerRCON* RCON = GameInstance->GetSubsystem<ULyraServerRCON>())
        {
            // 从命令行或配置文件读取端口和密码
            int32 RCONPort = 27015;
            FString RCONPassword = TEXT("admin123");

            // 可以从命令行解析
            FParse::Value(FCommandLine::Get(), TEXT("RCONPort="), RCONPort);
            FParse::Value(FCommandLine::Get(), TEXT("RCONPassword="), RCONPassword);

            if (RCON->StartRCONServer(RCONPort, RCONPassword))
            {
                UE_LOG(LogTemp, Log, TEXT("[Server] RCON server started on port %d"), RCONPort);
            }
        }
    }
}
```

### 4.4 服务器监控与日志

#### 4.4.1 结构化日志

创建一个日志工具类:

```cpp
// LyraServerLogger.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "LyraServerLogger.generated.h"

UENUM(BlueprintType)
enum class EServerLogSeverity : uint8
{
    Info,
    Warning,
    Error,
    Critical
};

USTRUCT(BlueprintType)
struct FServerLogEntry
{
    GENERATED_BODY()

    UPROPERTY()
    FDateTime Timestamp;

    UPROPERTY()
    EServerLogSeverity Severity;

    UPROPERTY()
    FString Category;

    UPROPERTY()
    FString Message;

    UPROPERTY()
    TMap<FString, FString> Metadata;
};

/**
 * 服务器日志系统
 * 支持结构化日志和远程日志传输
 */
UCLASS()
class LYRAGAME_API ULyraServerLogger : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 记录日志
    UFUNCTION(BlueprintCallable, Category = "Server|Logging")
    void LogEvent(EServerLogSeverity Severity, const FString& Category, const FString& Message);

    // 记录玩家事件
    UFUNCTION(BlueprintCallable, Category = "Server|Logging")
    void LogPlayerEvent(APlayerController* Player, const FString& Event, const FString& Details);

    // 记录性能指标
    UFUNCTION(BlueprintCallable, Category = "Server|Logging")
    void LogPerformanceMetric(const FString& MetricName, float Value);

    // 获取最近的日志
    UFUNCTION(BlueprintCallable, Category = "Server|Logging")
    TArray<FServerLogEntry> GetRecentLogs(int32 Count) const;

    // 导出日志到文件
    UFUNCTION(BlueprintCallable, Category = "Server|Logging")
    bool ExportLogsToFile(const FString& FilePath);

protected:
    UPROPERTY()
    TArray<FServerLogEntry> LogBuffer;

    UPROPERTY()
    int32 MaxBufferSize;
};
```

```cpp
// LyraServerLogger.cpp
#include "LyraServerLogger.h"
#include "Misc/FileHelper.h"
#include "Misc/Paths.h"
#include "Json.h"
#include "JsonUtilities.h"

void ULyraServerLogger::LogEvent(EServerLogSeverity Severity, const FString& Category, const FString& Message)
{
    FServerLogEntry Entry;
    Entry.Timestamp = FDateTime::Now();
    Entry.Severity = Severity;
    Entry.Category = Category;
    Entry.Message = Message;

    LogBuffer.Add(Entry);

    // 限制缓冲区大小
    if (LogBuffer.Num() > MaxBufferSize)
    {
        LogBuffer.RemoveAt(0, LogBuffer.Num() - MaxBufferSize);
    }

    // 同时输出到 UE 日志
    FString SeverityStr;
    switch (Severity)
    {
        case EServerLogSeverity::Info: SeverityStr = TEXT("INFO"); break;
        case EServerLogSeverity::Warning: SeverityStr = TEXT("WARN"); break;
        case EServerLogSeverity::Error: SeverityStr = TEXT("ERROR"); break;
        case EServerLogSeverity::Critical: SeverityStr = TEXT("CRITICAL"); break;
    }

    UE_LOG(LogTemp, Log, TEXT("[%s][%s] %s"), *SeverityStr, *Category, *Message);
}

void ULyraServerLogger::LogPlayerEvent(APlayerController* Player, const FString& Event, const FString& Details)
{
    if (!Player)
    {
        return;
    }

    FString PlayerName = TEXT("Unknown");
    if (Player->PlayerState)
    {
        PlayerName = Player->PlayerState->GetPlayerName();
    }

    FString Message = FString::Printf(TEXT("Player: %s | Event: %s | Details: %s"), 
        *PlayerName, *Event, *Details);

    LogEvent(EServerLogSeverity::Info, TEXT("Player"), Message);
}

void ULyraServerLogger::LogPerformanceMetric(const FString& MetricName, float Value)
{
    FString Message = FString::Printf(TEXT("%s: %.2f"), *MetricName, Value);
    LogEvent(EServerLogSeverity::Info, TEXT("Performance"), Message);
}

TArray<FServerLogEntry> ULyraServerLogger::GetRecentLogs(int32 Count) const
{
    int32 StartIndex = FMath::Max(0, LogBuffer.Num() - Count);
    return TArray<FServerLogEntry>(LogBuffer.GetData() + StartIndex, Count);
}

bool ULyraServerLogger::ExportLogsToFile(const FString& FilePath)
{
    TArray<TSharedPtr<FJsonValue>> JsonArray;

    for (const FServerLogEntry& Entry : LogBuffer)
    {
        TSharedPtr<FJsonObject> JsonObject = MakeShared<FJsonObject>();
        JsonObject->SetStringField(TEXT("timestamp"), Entry.Timestamp.ToString());
        JsonObject->SetStringField(TEXT("severity"), UEnum::GetValueAsString(Entry.Severity));
        JsonObject->SetStringField(TEXT("category"), Entry.Category);
        JsonObject->SetStringField(TEXT("message"), Entry.Message);

        JsonArray.Add(MakeShared<FJsonValueObject>(JsonObject));
    }

    FString OutputString;
    TSharedRef<TJsonWriter<>> Writer = TJsonWriterFactory<>::Create(&OutputString);
    FJsonSerializer::Serialize(JsonArray, Writer);

    return FFileHelper::SaveStringToFile(OutputString, *FilePath);
}
```

---

## 5. 热更新系统设计

### 5.1 Chunk 分块打包

Unreal 支持将内容分块打包,实现按需下载和热更新。

#### 5.1.1 配置 Chunk

在 `DefaultGame.ini` 中配置:

```ini
[/Script/UnrealEd.ProjectPackagingSettings]
; 启用 Chunk 打包
bGenerateChunks=True

; 主 Chunk(Chunk 0 是基础包)
+MapsToCook=(FilePath="/Game/Maps/MainMenu")
+MapsToCook=(FilePath="/Game/Maps/Lobby")

; 定义 Chunk 1: ShooterCore 内容
+ChunkMapping=(ChunkID=1, AssetPaths="/Game/ShooterCore")
+ChunkMapping=(ChunkID=1, AssetPaths="/Plugins/GameFeatures/ShooterCore")

; 定义 Chunk 2: TopDownArena 内容
+ChunkMapping=(ChunkID=2, AssetPaths="/Game/TopDownArena")
+ChunkMapping=(ChunkID=2, AssetPaths="/Plugins/GameFeatures/TopDownArena")

; Chunk 打包选项
bBuildHTTPChunkInstallData=True
HttpChunkInstallDataDirectory="$(ProjectDir)/ChunkInstallData"
HttpChunkInstallDataVersion="1.0"
```

#### 5.1.2 代码中按需加载 Chunk

```cpp
// LyraChunkDownloader.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "IPlatformFilePak.h"
#include "LyraChunkDownloader.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnChunkDownloadProgress, int32, ChunkID, float, Progress);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnChunkDownloadComplete, int32, ChunkID);

/**
 * Chunk 下载管理器
 * 处理运行时的内容下载和挂载
 */
UCLASS()
class LYRAGAME_API ULyraChunkDownloader : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 开始下载 Chunk
    UFUNCTION(BlueprintCallable, Category = "Chunk")
    void DownloadChunk(int32 ChunkID);

    // 检查 Chunk 是否已下载
    UFUNCTION(BlueprintPure, Category = "Chunk")
    bool IsChunkDownloaded(int32 ChunkID) const;

    // 获取 Chunk 下载进度
    UFUNCTION(BlueprintPure, Category = "Chunk")
    float GetChunkDownloadProgress(int32 ChunkID) const;

    // 挂载已下载的 Chunk
    UFUNCTION(BlueprintCallable, Category = "Chunk")
    bool MountChunk(int32 ChunkID);

    // 卸载 Chunk
    UFUNCTION(BlueprintCallable, Category = "Chunk")
    bool UnmountChunk(int32 ChunkID);

    // 事件委托
    UPROPERTY(BlueprintAssignable, Category = "Chunk")
    FOnChunkDownloadProgress OnChunkDownloadProgress;

    UPROPERTY(BlueprintAssignable, Category = "Chunk")
    FOnChunkDownloadComplete OnChunkDownloadComplete;

protected:
    // 下载回调
    void HandleChunkDownloaded(int32 ChunkID);

private:
    TMap<int32, bool> ChunkDownloadStatus;
    TMap<int32, float> ChunkDownloadProgress;
    TMap<int32, FPakFile*> MountedChunks;
};
```

```cpp
// LyraChunkDownloader.cpp
#include "LyraChunkDownloader.h"
#include "IPlatformFilePak.h"
#include "Misc/CoreDelegates.h"

void ULyraChunkDownloader::DownloadChunk(int32 ChunkID)
{
    UE_LOG(LogTemp, Log, TEXT("[Chunk] Starting download for Chunk %d"), ChunkID);

    // 检查是否已下载
    if (IsChunkDownloaded(ChunkID))
    {
        UE_LOG(LogTemp, Log, TEXT("[Chunk] Chunk %d already downloaded"), ChunkID);
        OnChunkDownloadComplete.Broadcast(ChunkID);
        return;
    }

    // 实际实现需要使用 HTTP 下载
    // 这里简化为直接检查本地文件

    // 假设文件位置
    FString ChunkPakPath = FPaths::ProjectContentDir() / TEXT("Paks") / FString::Printf(TEXT("pakchunk%d-Windows.pak"), ChunkID);

    if (FPaths::FileExists(ChunkPakPath))
    {
        ChunkDownloadStatus.Add(ChunkID, true);
        HandleChunkDownloaded(ChunkID);
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("[Chunk] Chunk %d not found at %s"), ChunkID, *ChunkPakPath);
    }
}

bool ULyraChunkDownloader::IsChunkDownloaded(int32 ChunkID) const
{
    const bool* Status = ChunkDownloadStatus.Find(ChunkID);
    return Status && *Status;
}

float ULyraChunkDownloader::GetChunkDownloadProgress(int32 ChunkID) const
{
    const float* Progress = ChunkDownloadProgress.Find(ChunkID);
    return Progress ? *Progress : 0.0f;
}

bool ULyraChunkDownloader::MountChunk(int32 ChunkID)
{
    if (!IsChunkDownloaded(ChunkID))
    {
        UE_LOG(LogTemp, Error, TEXT("[Chunk] Cannot mount Chunk %d: not downloaded"), ChunkID);
        return false;
    }

    // 检查是否已挂载
    if (MountedChunks.Contains(ChunkID))
    {
        UE_LOG(LogTemp, Warning, TEXT("[Chunk] Chunk %d already mounted"), ChunkID);
        return true;
    }

    FString ChunkPakPath = FPaths::ProjectContentDir() / TEXT("Paks") / FString::Printf(TEXT("pakchunk%d-Windows.pak"), ChunkID);

    // 挂载 PAK 文件
    FPakPlatformFile* PakPlatform = (FPakPlatformFile*)(FPlatformFileManager::Get().FindPlatformFile(TEXT("PakFile")));
    if (PakPlatform)
    {
        FPakFile* PakFile = new FPakFile(&PakPlatform->GetLowerLevel(), *ChunkPakPath, false);
        if (PakFile->IsValid())
        {
            PakPlatform->Mount(*ChunkPakPath, 0, *FPaths::ProjectContentDir());
            MountedChunks.Add(ChunkID, PakFile);

            UE_LOG(LogTemp, Log, TEXT("[Chunk] Successfully mounted Chunk %d"), ChunkID);
            return true;
        }
        else
        {
            delete PakFile;
            UE_LOG(LogTemp, Error, TEXT("[Chunk] Failed to mount Chunk %d: invalid PAK file"), ChunkID);
        }
    }

    return false;
}

bool ULyraChunkDownloader::UnmountChunk(int32 ChunkID)
{
    FPakFile** PakFilePtr = MountedChunks.Find(ChunkID);
    if (!PakFilePtr)
    {
        UE_LOG(LogTemp, Warning, TEXT("[Chunk] Chunk %d is not mounted"), ChunkID);
        return false;
    }

    FString ChunkPakPath = FPaths::ProjectContentDir() / TEXT("Paks") / FString::Printf(TEXT("pakchunk%d-Windows.pak"), ChunkID);

    FPakPlatformFile* PakPlatform = (FPakPlatformFile*)(FPlatformFileManager::Get().FindPlatformFile(TEXT("PakFile")));
    if (PakPlatform)
    {
        PakPlatform->Unmount(*ChunkPakPath);
        delete *PakFilePtr;
        MountedChunks.Remove(ChunkID);

        UE_LOG(LogTemp, Log, TEXT("[Chunk] Successfully unmounted Chunk %d"), ChunkID);
        return true;
    }

    return false;
}

void ULyraChunkDownloader::HandleChunkDownloaded(int32 ChunkID)
{
    UE_LOG(LogTemp, Log, TEXT("[Chunk] Chunk %d download complete"), ChunkID);
    OnChunkDownloadComplete.Broadcast(ChunkID);
}
```

### 5.2 PAK 签名与加密

#### 5.2.1 生成加密密钥

使用 Unreal 的工具生成 AES 密钥:

```bash
# 生成密钥
UnrealPak.exe -GenerateKeys="C:/MyProject/Keys/crypto.json"
```

生成的 `crypto.json`:

```json
{
    "EncryptionKey": {
        "Key": "0x1234567890ABCDEF1234567890ABCDEF1234567890ABCDEF1234567890ABCDEF",
        "Guid": "12345678-1234-1234-1234-123456789012"
    }
}
```

#### 5.2.2 配置加密

在 `DefaultCrypto.ini`:

```ini
[Core.Encryption.Core]
; 启用加密
bEnablePakSigning=True
bEncryptPakIniFiles=True
bEncryptPakIndex=True
bEncryptUAssetFiles=True
bEncryptAllAssetFiles=False

; 签名密钥
SigningPublicExponent="0x010001"
SigningModulus="0x..."
SigningPrivateExponent="0x..."

; 加密密钥 (来自 crypto.json)
EncryptionKey="0x1234567890ABCDEF1234567890ABCDEF1234567890ABCDEF1234567890ABCDEF"
```

#### 5.2.3 打包时应用加密

```bash
RunUAT BuildCookRun \
    -project="C:/Projects/Lyra/Lyra.uproject" \
    -platform=Win64 \
    -clientconfig=Shipping \
    -cook \
    -stage \
    -package \
    -encrypted \
    -cryptokeys="C:/MyProject/Keys/crypto.json" \
    -archivedirectory="C:/Builds/Lyra_Win64_Encrypted"
```

### 5.3 版本管理与更新检测

#### 5.3.1 版本信息文件

创建 `version.json`:

```json
{
    "version": "1.2.3",
    "buildNumber": 456,
    "releaseDate": "2026-03-11T00:00:00Z",
    "minCompatibleVersion": "1.2.0",
    "chunks": [
        {
            "id": 0,
            "name": "BaseGame",
            "version": "1.2.3",
            "size": 1073741824,
            "checksum": "abc123def456"
        },
        {
            "id": 1,
            "name": "ShooterCore",
            "version": "1.2.2",
            "size": 536870912,
            "checksum": "789ghi012jkl"
        },
        {
            "id": 2,
            "name": "TopDownArena",
            "version": "1.2.3",
            "size": 268435456,
            "checksum": "345mno678pqr"
        }
    ],
    "changelog": "# Version 1.2.3\n- Fixed weapon balance\n- Added new map"
}
```

#### 5.3.2 更新检测器

```cpp
// LyraUpdateChecker.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "Http.h"
#include "LyraUpdateChecker.generated.h"

USTRUCT(BlueprintType)
struct FGameVersion
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString Version;

    UPROPERTY(BlueprintReadOnly)
    int32 BuildNumber;

    UPROPERTY(BlueprintReadOnly)
    FDateTime ReleaseDate;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnUpdateAvailable, const FGameVersion&, NewVersion);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnUpdateCheckComplete);

UCLASS()
class LYRAGAME_API ULyraUpdateChecker : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 检查更新
    UFUNCTION(BlueprintCallable, Category = "Update")
    void CheckForUpdates(const FString& UpdateServerURL);

    // 获取当前版本
    UFUNCTION(BlueprintPure, Category = "Update")
    FGameVersion GetCurrentVersion() const;

    // 获取最新版本
    UFUNCTION(BlueprintPure, Category = "Update")
    FGameVersion GetLatestVersion() const;

    // 是否有可用更新
    UFUNCTION(BlueprintPure, Category = "Update")
    bool IsUpdateAvailable() const;

    // 事件
    UPROPERTY(BlueprintAssignable, Category = "Update")
    FOnUpdateAvailable OnUpdateAvailable;

    UPROPERTY(BlueprintAssignable, Category = "Update")
    FOnUpdateCheckComplete OnUpdateCheckComplete;

protected:
    void OnVersionInfoReceived(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bWasSuccessful);

    FGameVersion CurrentVersion;
    FGameVersion LatestVersion;
    bool bUpdateAvailable;
};
```

```cpp
// LyraUpdateChecker.cpp
#include "LyraUpdateChecker.h"
#include "HttpModule.h"
#include "Interfaces/IHttpResponse.h"
#include "JsonObjectConverter.h"

void ULyraUpdateChecker::CheckForUpdates(const FString& UpdateServerURL)
{
    // 读取当前版本
    CurrentVersion.Version = TEXT("1.2.2");
    CurrentVersion.BuildNumber = 455;

    UE_LOG(LogTemp, Log, TEXT("[Update] Checking for updates from %s"), *UpdateServerURL);

    // 创建 HTTP 请求
    TSharedRef<IHttpRequest> HttpRequest = FHttpModule::Get().CreateRequest();
    HttpRequest->SetVerb(TEXT("GET"));
    HttpRequest->SetURL(UpdateServerURL + TEXT("/version.json"));
    HttpRequest->SetHeader(TEXT("Content-Type"), TEXT("application/json"));

    HttpRequest->OnProcessRequestComplete().BindUObject(this, &ULyraUpdateChecker::OnVersionInfoReceived);

    HttpRequest->ProcessRequest();
}

void ULyraUpdateChecker::OnVersionInfoReceived(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bWasSuccessful)
{
    if (!bWasSuccessful || !Response.IsValid())
    {
        UE_LOG(LogTemp, Error, TEXT("[Update] Failed to check for updates"));
        OnUpdateCheckComplete.Broadcast();
        return;
    }

    // 解析 JSON
    FString ResponseContent = Response->GetContentAsString();
    TSharedPtr<FJsonObject> JsonObject;
    TSharedRef<TJsonReader<>> Reader = TJsonReaderFactory<>::Create(ResponseContent);

    if (FJsonSerializer::Deserialize(Reader, JsonObject) && JsonObject.IsValid())
    {
        // 提取版本信息
        LatestVersion.Version = JsonObject->GetStringField(TEXT("version"));
        LatestVersion.BuildNumber = JsonObject->GetIntegerField(TEXT("buildNumber"));

        FString ReleaseDateStr = JsonObject->GetStringField(TEXT("releaseDate"));
        FDateTime::ParseIso8601(*ReleaseDateStr, LatestVersion.ReleaseDate);

        UE_LOG(LogTemp, Log, TEXT("[Update] Current: %s (Build %d)"), 
            *CurrentVersion.Version, CurrentVersion.BuildNumber);
        UE_LOG(LogTemp, Log, TEXT("[Update] Latest: %s (Build %d)"), 
            *LatestVersion.Version, LatestVersion.BuildNumber);

        // 比较版本
        bUpdateAvailable = LatestVersion.BuildNumber > CurrentVersion.BuildNumber;

        if (bUpdateAvailable)
        {
            UE_LOG(LogTemp, Log, TEXT("[Update] Update available!"));
            OnUpdateAvailable.Broadcast(LatestVersion);
        }
        else
        {
            UE_LOG(LogTemp, Log, TEXT("[Update] Game is up to date"));
        }
    }

    OnUpdateCheckComplete.Broadcast();
}

FGameVersion ULyraUpdateChecker::GetCurrentVersion() const
{
    return CurrentVersion;
}

FGameVersion ULyraUpdateChecker::GetLatestVersion() const
{
    return LatestVersion;
}

bool ULyraUpdateChecker::IsUpdateAvailable() const
{
    return bUpdateAvailable;
}
```

---

## 6. CI/CD 自动化构建

### 6.1 Jenkins Pipeline 示例

`Jenkinsfile`:

```groovy
pipeline {
    agent any
    
    environment {
        UE_ROOT = 'C:\\UnrealEngine\\UE_5.3'
        PROJECT_ROOT = "${WORKSPACE}\\Lyra"
        BUILD_CONFIG = 'Shipping'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }
        
        stage('Clean') {
            steps {
                echo 'Cleaning previous builds...'
                bat '''
                    if exist "%PROJECT_ROOT%\\Saved" rmdir /S /Q "%PROJECT_ROOT%\\Saved"
                    if exist "%PROJECT_ROOT%\\Intermediate" rmdir /S /Q "%PROJECT_ROOT%\\Intermediate"
                    if exist "%PROJECT_ROOT%\\Binaries" rmdir /S /Q "%PROJECT_ROOT%\\Binaries"
                '''
            }
        }
        
        stage('Build Game') {
            steps {
                echo 'Building game client...'
                bat '''
                    "%UE_ROOT%\\Engine\\Build\\BatchFiles\\RunUAT.bat" BuildCookRun ^
                        -project="%PROJECT_ROOT%\\Lyra.uproject" ^
                        -platform=Win64 ^
                        -clientconfig=%BUILD_CONFIG% ^
                        -cook ^
                        -stage ^
                        -package ^
                        -build ^
                        -compressed ^
                        -archivedirectory="%WORKSPACE%\\Builds\\Client" ^
                        -noP4 ^
                        -utf8output
                '''
            }
        }
        
        stage('Build Server') {
            steps {
                echo 'Building dedicated server...'
                bat '''
                    "%UE_ROOT%\\Engine\\Build\\BatchFiles\\RunUAT.bat" BuildCookRun ^
                        -project="%PROJECT_ROOT%\\Lyra.uproject" ^
                        -target=LyraServer ^
                        -platform=Linux ^
                        -serverconfig=%BUILD_CONFIG% ^
                        -cook ^
                        -stage ^
                        -package ^
                        -server ^
                        -noclient ^
                        -archivedirectory="%WORKSPACE%\\Builds\\Server" ^
                        -noP4 ^
                        -utf8output
                '''
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running automated tests...'
                bat '''
                    "%UE_ROOT%\\Engine\\Binaries\\Win64\\UnrealEditor-Cmd.exe" ^
                        "%PROJECT_ROOT%\\Lyra.uproject" ^
                        -ExecCmds="Automation RunTests Lyra; Quit" ^
                        -TestExit="Automation Test Queue Empty" ^
                        -ReportOutputPath="%WORKSPACE%\\TestResults" ^
                        -Log
                '''
            }
        }
        
        stage('Package') {
            steps {
                echo 'Creating distribution packages...'
                script {
                    def buildNumber = env.BUILD_NUMBER
                    def buildDate = new Date().format('yyyyMMdd')
                    
                    bat """
                        7z a "%WORKSPACE%\\Lyra_Win64_%BUILD_CONFIG%_${buildNumber}_${buildDate}.zip" ^
                            "%WORKSPACE%\\Builds\\Client\\*"
                        
                        7z a "%WORKSPACE%\\LyraServer_Linux_%BUILD_CONFIG%_${buildNumber}_${buildDate}.tar.gz" ^
                            "%WORKSPACE%\\Builds\\Server\\*"
                    """
                }
            }
        }
        
        stage('Upload') {
            steps {
                echo 'Uploading builds to CDN...'
                // 这里可以添加上传到 S3、Azure Blob 等的逻辑
                script {
                    // 示例:上传到 AWS S3
                    // sh '''
                    //     aws s3 sync ${WORKSPACE}/Builds/Client/ s3://my-game-cdn/builds/${BUILD_NUMBER}/client/
                    //     aws s3 sync ${WORKSPACE}/Builds/Server/ s3://my-game-cdn/builds/${BUILD_NUMBER}/server/
                    // '''
                }
            }
        }
    }
    
    post {
        success {
            echo 'Build successful!'
            // 可以在这里添加通知,例如发送邮件或 Slack 消息
        }
        failure {
            echo 'Build failed!'
            // 发送失败通知
        }
        always {
            // 清理临时文件
            bat '''
                if exist "%WORKSPACE%\\Logs" (
                    xcopy "%WORKSPACE%\\Logs" "%WORKSPACE%\\BuildLogs\\%BUILD_NUMBER%" /E /I
                )
            '''
            
            // 归档构建产物
            archiveArtifacts artifacts: '*.zip,*.tar.gz', fingerprint: true
        }
    }
}
```

### 6.2 GitHub Actions 工作流

`.github/workflows/build.yml`:

```yaml
name: Build and Package Lyra

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

env:
  UE_VERSION: "5.3"
  PROJECT_NAME: "Lyra"

jobs:
  build-windows:
    runs-on: windows-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          lfs: true

      - name: Setup Unreal Engine
        uses: game-ci/unity-builder@v2
        # 注意:实际项目需要使用 UE 特定的 Action
        # 或自定义安装脚本

      - name: Build Windows Client
        run: |
          $env:UE_ROOT = "C:\UnrealEngine\UE_${{ env.UE_VERSION }}"
          & "$env:UE_ROOT\Engine\Build\BatchFiles\RunUAT.bat" BuildCookRun `
            -project="${{ github.workspace }}\${{ env.PROJECT_NAME }}.uproject" `
            -platform=Win64 `
            -clientconfig=Shipping `
            -cook -stage -package -build `
            -archivedirectory="${{ github.workspace }}\Builds\Win64" `
            -noP4

      - name: Upload Windows Build
        uses: actions/upload-artifact@v3
        with:
          name: Lyra-Windows-${{ github.sha }}
          path: Builds/Win64/

  build-linux-server:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Cross-Compile Toolchain
        run: |
          # 下载并设置 Linux 交叉编译工具链
          wget https://cdn.unrealengine.com/CrossToolchain_Linux/v22_clang-16.0.6-centos7.tar.gz
          tar -xzf v22_clang-16.0.6-centos7.tar.gz -C /opt/
          echo "LINUX_MULTIARCH_ROOT=/opt/v22_clang-16.0.6-centos7/x86_64-unknown-linux-gnu" >> $GITHUB_ENV

      - name: Build Linux Server
        run: |
          $UE_ROOT/Engine/Build/BatchFiles/RunUAT.sh BuildCookRun \
            -project="${GITHUB_WORKSPACE}/${{ env.PROJECT_NAME }}.uproject" \
            -target=LyraServer \
            -platform=Linux \
            -serverconfig=Shipping \
            -cook -stage -package -server -noclient \
            -archivedirectory="${GITHUB_WORKSPACE}/Builds/LinuxServer" \
            -noP4

      - name: Upload Linux Server Build
        uses: actions/upload-artifact@v3
        with:
          name: Lyra-LinuxServer-${{ github.sha }}
          path: Builds/LinuxServer/

  deploy:
    needs: [build-windows, build-linux-server]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Download Windows Build
        uses: actions/download-artifact@v3
        with:
          name: Lyra-Windows-${{ github.sha }}
          path: ./builds/windows

      - name: Download Linux Server Build
        uses: actions/download-artifact@v3
        with:
          name: Lyra-LinuxServer-${{ github.sha }}
          path: ./builds/linux

      - name: Deploy to CDN
        run: |
          # 示例:部署到 AWS S3
          # aws s3 sync ./builds/windows s3://my-game-cdn/builds/${{ github.sha }}/windows/
          # aws s3 sync ./builds/linux s3://my-game-cdn/builds/${{ github.sha }}/linux/
          echo "Deployment completed"

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        if: startsWith(github.ref, 'refs/tags/')
        with:
          files: |
            builds/windows/**
            builds/linux/**
```

### 6.3 Docker 容器化服务器

`Dockerfile`:

```dockerfile
# Lyra Linux 专用服务器 Docker 镜像
FROM ubuntu:22.04

# 安装依赖
RUN apt-get update && apt-get install -y \
    libpulse0 \
    libopenal1 \
    libsdl2-2.0-0 \
    libvorbisfile3 \
    libvorbis0a \
    libogg0 \
    && rm -rf /var/lib/apt/lists/*

# 创建服务器目录
WORKDIR /lyra-server

# 复制服务器文件
COPY ./Builds/LinuxServer/ /lyra-server/

# 赋予执行权限
RUN chmod +x /lyra-server/LyraServer.sh

# 暴露端口
EXPOSE 7777/udp
EXPOSE 27015/tcp

# 环境变量
ENV UE_SERVER_MAP="/Game/Experiences/B_LyraShooterGame_Elimination"
ENV UE_SERVER_PORT=7777
ENV UE_MAX_PLAYERS=32

# 启动脚本
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["docker-entrypoint.sh"]
```

`docker-entrypoint.sh`:

```bash
#!/bin/bash
set -e

# Lyra Server Docker 入口脚本

echo "Starting Lyra Dedicated Server..."
echo "Map: $UE_SERVER_MAP"
echo "Port: $UE_SERVER_PORT"
echo "Max Players: $UE_MAX_PLAYERS"

# 启动服务器
exec /lyra-server/LyraServer.sh \
    $UE_SERVER_MAP \
    -server \
    -log \
    -port=$UE_SERVER_PORT \
    -MaxPlayers=$UE_MAX_PLAYERS \
    -NoSteam \
    -stdout \
    -unattended \
    "$@"
```

`docker-compose.yml`:

```yaml
version: '3.8'

services:
  lyra-server-1:
    build: .
    image: lyra-server:latest
    container_name: lyra-server-1
    environment:
      - UE_SERVER_MAP=/Game/Experiences/B_LyraShooterGame_Elimination
      - UE_SERVER_PORT=7777
      - UE_MAX_PLAYERS=32
    ports:
      - "7777:7777/udp"
      - "27015:27015/tcp"
    volumes:
      - ./logs:/lyra-server/Saved/Logs
      - ./config:/lyra-server/Config
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  lyra-server-2:
    image: lyra-server:latest
    container_name: lyra-server-2
    environment:
      - UE_SERVER_MAP=/Game/Experiences/B_LyraShooterGame_ControlPoint
      - UE_SERVER_PORT=7778
      - UE_MAX_PLAYERS=16
    ports:
      - "7778:7778/udp"
      - "27016:27015/tcp"
    volumes:
      - ./logs:/lyra-server/Saved/Logs
      - ./config:/lyra-server/Config
    restart: unless-stopped
```

部署命令:

```bash
# 构建镜像
docker-compose build

# 启动服务器
docker-compose up -d

# 查看日志
docker-compose logs -f lyra-server-1

# 停止服务器
docker-compose down
```

---

## 7. 性能与安全优化

### 7.1 Shipping 配置优化清单

#### 7.1.1 核心配置

`Config/Shipping/ShippingEngine.ini`:

```ini
[/Script/Engine.Engine]
; 禁用调试功能
bDisableAllScreenMessages=True
bEnableOnScreenDebugMessages=False
bEnableVisualLogRecordingOnStart=False
bSuppressMapWarnings=True

; 禁用统计收集
bEnableStatsRecording=False

[/Script/Engine.GameEngine]
; 网络超时
ServerConnectionTimeout=60.0
InitialConnectTimeout=60.0

[Core.System]
; 内存优化
MemorySize=8192
CacheSizeMegs=512

; 垃圾回收
gc.MaxObjectsNotConsideredByGC=1
gc.SizeOfPermanentObjectPool=0
gc.TimeBetweenPurgingPendingKillObjects=60

[SystemSettings]
; 图形优化
r.VSync=0
r.FinishCurrentFrame=0
r.Streaming.PoolSize=3000
r.Streaming.MaxNumTexturesToStreamPerFrame=20

; 网络优化
net.MaxRepArraySize=4096
net.MaxRepArrayMemory=131072
net.IpNetDriver.MaxChannelsOverride=2048

; 物理优化
p.DefaultMaxDepenetrationVelocity=130
p.DefaultMaxAngularVelocity=400

[ConsoleVariables]
; 渲染
r.SupportSkyAtmosphere=1
r.SupportSkyAtmosphereAffectsHeightFog=1
r.MobileHDR=1

; 网络
net.AllowEncryption=1
net.AllowAsyncLoading=1
```

#### 7.1.2 安全强化

```ini
[/Script/Engine.Engine]
; 命令行安全
UE_COMMAND_LINE_USES_ALLOW_LIST=1
UE_OVERRIDE_COMMAND_LINE_ALLOW_LIST="-log -Port -MaxPlayers"

; 禁止加载非烘焙内容
bAllowGeneratedIniWhenCooked=False
bAllowNonUFSIniWhenCooked=False

; 证书验证
bDisableUnverifiedCertificates=True

[/Script/Engine.GameSession]
; 防作弊
bRequiresPushToTalk=False
bIsLANMatch=False
bUsesPresence=True

[ConsoleVariables]
; 禁用作弊命令
net.AllowCheats=0
```

### 7.2 代码混淆与保护

#### 7.2.1 Blueprint Nativization

Blueprint Nativization 将蓝图转换为 C++ 代码,提升性能并增加逆向难度。

在 `DefaultEditor.ini`:

```ini
[/Script/UnrealEd.ProjectPackagingSettings]
BlueprintNativizationMethod=Inclusive
bNativizeBlueprintAssets=True
bNativizeOnlySelectedBlueprints=False

; 包含的蓝图目录
+NativizeBlueprintAssets=(FilePath="/Game/Blueprints")
+NativizeBlueprintAssets=(FilePath="/Game/UI")
```

#### 7.2.2 代码符号剥离

在打包脚本中:

```bash
# Windows
RunUAT BuildCookRun \
    ... \
    -distribution \
    -stripnames

# Linux
RunUAT BuildCookRun \
    ... \
    -distribution \
    -nodebuginfo
```

### 7.3 崩溃报告与分析

#### 7.3.1 集成 Sentry

`LyraSentryIntegration.h`:

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "LyraSentryIntegration.generated.h"

UCLASS()
class LYRAGAME_API ULyraSentryIntegration : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    
    UFUNCTION(BlueprintCallable, Category = "Crash Reporting")
    void ReportError(const FString& Message, const FString& Category);

    UFUNCTION(BlueprintCallable, Category = "Crash Reporting")
    void SetUserContext(const FString& UserID, const FString& Username);

protected:
    void OnCrash();
};
```

```cpp
// LyraSentryIntegration.cpp
#include "LyraSentryIntegration.h"

#if PLATFORM_WINDOWS || PLATFORM_LINUX || PLATFORM_MAC
#include "sentry.h"
#endif

void ULyraSentryIntegration::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

#if PLATFORM_WINDOWS || PLATFORM_LINUX || PLATFORM_MAC
    sentry_options_t* options = sentry_options_new();
    sentry_options_set_dsn(options, "https://your-dsn@sentry.io/project-id");
    sentry_options_set_release(options, "lyra@1.2.3");
    sentry_options_set_environment(options, "production");
    sentry_init(options);

    UE_LOG(LogTemp, Log, TEXT("[Sentry] Crash reporting initialized"));
#endif

    // 注册崩溃回调
    FCoreDelegates::OnHandleSystemError.AddUObject(this, &ULyraSentryIntegration::OnCrash);
}

void ULyraSentryIntegration::ReportError(const FString& Message, const FString& Category)
{
#if PLATFORM_WINDOWS || PLATFORM_LINUX || PLATFORM_MAC
    sentry_value_t event = sentry_value_new_event();
    sentry_value_set_by_key(event, "level", sentry_value_new_string("error"));
    sentry_value_set_by_key(event, "logger", sentry_value_new_string(TCHAR_TO_UTF8(*Category)));
    
    sentry_value_t message_obj = sentry_value_new_message_event(
        SENTRY_LEVEL_ERROR,
        TCHAR_TO_UTF8(*Category),
        TCHAR_TO_UTF8(*Message)
    );
    
    sentry_capture_event(message_obj);

    UE_LOG(LogTemp, Warning, TEXT("[Sentry] Error reported: %s"), *Message);
#endif
}

void ULyraSentryIntegration::SetUserContext(const FString& UserID, const FString& Username)
{
#if PLATFORM_WINDOWS || PLATFORM_LINUX || PLATFORM_MAC
    sentry_value_t user = sentry_value_new_object();
    sentry_value_set_by_key(user, "id", sentry_value_new_string(TCHAR_TO_UTF8(*UserID)));
    sentry_value_set_by_key(user, "username", sentry_value_new_string(TCHAR_TO_UTF8(*Username)));
    sentry_set_user(user);

    UE_LOG(LogTemp, Log, TEXT("[Sentry] User context set: %s"), *UserID);
#endif
}

void ULyraSentryIntegration::OnCrash()
{
    UE_LOG(LogTemp, Fatal, TEXT("[Sentry] Crash detected!"));

    // Sentry 会自动捕获崩溃并上传
    // 这里可以添加额外的崩溃前处理
}
```

---

## 8. 常见问题与调试

### 8.1 打包失败排查

#### 8.1.1 Cook 失败

**症状:** 烘焙过程中报错,提示资源加载失败。

**排查步骤:**

1. 检查日志:
   ```
   Project/Saved/Logs/Cook.txt
   ```

2. 查找失败的资源:
   ```
   LogCook: Error: Failed to load '/Game/Path/To/Asset'
   ```

3. 常见原因:
   - 资源引用丢失
   - 循环依赖
   - 插件未启用

**解决方案:**

```bash
# 清理缓存
rm -rf Project/Saved/Cooked
rm -rf Project/DerivedDataCache

# 验证资源引用
UnrealEditor-Cmd.exe Project.uproject -run=ResavePackages -fixupredirects -autocheckout -projectonly
```

#### 8.1.2 编译错误

**症状:** C++ 代码编译失败。

**排查:**

1. 检查 Build 日志:
   ```
   Project/Saved/Logs/UAT_Log.txt
   ```

2. 常见错误:
   - 头文件包含顺序错误
   - 模块依赖未声明
   - 平台特定代码问题

**解决方案:**

```csharp
// Build.cs 中确保依赖正确
PublicDependencyModuleNames.AddRange(new string[] {
    "Core",
    "CoreUObject",
    "Engine",
    "GameplayAbilities",
    "ModularGameplay"
});
```

#### 8.1.3 PAK 文件损坏

**症状:** 打包成功,但运行时资源加载失败。

**验证 PAK 完整性:**

```bash
# 列出 PAK 内容
UnrealPak.exe pakchunk0-Windows.pak -List > pak_contents.txt

# 验证 PAK
UnrealPak.exe pakchunk0-Windows.pak -Test
```

### 8.2 网络同步问题

#### 8.2.1 服务器客户端不一致

**诊断工具:**

```cpp
// 在服务器和客户端都启用
Exec: net.PackageMap.DebugObject

// 查看网络统计
Exec: stat net

// 查看复制属性
Exec: Net.Replication.ShowActorDebugInfo
```

#### 8.2.2 高延迟优化

`Config/Server/ServerEngine.ini`:

```ini
[/Script/OnlineSubsystemUtils.IpNetDriver]
; 增加缓冲区
NetServerMaxTickRate=60
MaxClientRate=50000
MaxInternetClientRate=35000

; 调整复制频率
NetUpdateFrequency=100.0
MinNetUpdateFrequency=10.0
```

### 8.3 性能分析工具

#### 8.3.1 Unreal Insights

启动 Insights 追踪:

```bash
# 运行游戏时启用追踪
LyraGame.exe -trace=cpu,frame,log

# 或服务器
LyraServer.sh -trace=cpu,frame,net
```

查看追踪:

```bash
# 启动 Unreal Insights
UnrealInsights.exe
```

#### 8.3.2 内存分析

```bash
# 运行时内存快照
Exec: Obj List
Exec: Obj Refs Class=/Game/MyBlueprintClass
Exec: Obj Dump
```

---

## 9. 完整打包 Checklist

### 9.1 准备阶段

- [ ] 更新版本号 (`DefaultGame.ini`)
- [ ] 编写 Changelog
- [ ] 运行所有自动化测试
- [ ] 验证所有资源引用
- [ ] 检查内存泄漏
- [ ] 清理临时文件和日志

### 9.2 配置阶段

- [ ] 设置 Shipping 配置
- [ ] 配置加密密钥
- [ ] 配置 PAK 签名
- [ ] 设置 Chunk 分块
- [ ] 配置在线服务(EOS/Steam)

### 9.3 构建阶段

**客户端:**

- [ ] Windows 64-bit Shipping
- [ ] Linux (可选)
- [ ] macOS (可选)
- [ ] Android (可选)
- [ ] iOS (可选)

**服务器:**

- [ ] Linux Server Shipping
- [ ] Windows Server (可选)

### 9.4 测试阶段

- [ ] 冒烟测试 (启动/主菜单/进入游戏)
- [ ] 网络测试 (客户端连接服务器)
- [ ] 性能测试 (FPS/内存/网络带宽)
- [ ] 兼容性测试 (不同平台/配置)
- [ ] 压力测试 (高负载/长时间运行)

### 9.5 发布阶段

- [ ] 上传到 CDN
- [ ] 更新版本服务器
- [ ] 发布 Release Notes
- [ ] 通知玩家
- [ ] 监控服务器状态

### 9.6 后续阶段

- [ ] 收集崩溃报告
- [ ] 分析性能数据
- [ ] 处理玩家反馈
- [ ] 准备热修复(如需要)

---

## 10. 最佳实践总结

### 10.1 打包流程规范

1. **版本控制**
   - 使用语义化版本号 (Semantic Versioning)
   - 为每个构建打 Tag
   - 维护详细的 Changelog

2. **自动化优先**
   - 使用 CI/CD 替代手动打包
   - 自动化测试流程
   - 自动化部署

3. **分层配置**
   - 基础配置 (`DefaultEngine.ini`)
   - 平台配置 (`Windows/WindowsEngine.ini`)
   - 环境配置 (Development/Shipping)

4. **增量更新**
   - 使用 Chunk 系统
   - 实现差异更新
   - 支持断点续传

### 10.2 服务器部署建议

1. **容器化**
   - 使用 Docker 封装服务器
   - 通过 Kubernetes 管理集群
   - 实现自动扩缩容

2. **监控与日志**
   - 实时性能监控
   - 结构化日志系统
   - 崩溃报告集成

3. **安全防护**
   - 网络层防护 (DDoS 防护)
   - 应用层防护 (防作弊系统)
   - 数据加密传输

4. **灾难恢复**
   - 定期备份
   - 快速回滚机制
   - 多区域部署

### 10.3 性能优化技巧

1. **资源优化**
   - 纹理压缩
   - Mesh LOD
   - 音频压缩

2. **代码优化**
   - Blueprint Nativization
   - C++ 性能关键路径
   - 网络复制优化

3. **配置优化**
   - 图形设置分级
   - 网络参数调优
   - 内存管理策略

### 10.4 故障排查思路

1. **复现问题**
   - 收集复现步骤
   - 确认环境差异
   - 使用调试版本验证

2. **日志分析**
   - 查找关键错误
   - 追踪调用堆栈
   - 分析时序

3. **工具辅助**
   - Unreal Insights
   - Visual Studio Profiler
   - Network Profiler

4. **逐步排查**
   - 二分法隔离问题
   - 移除可疑插件
   - 回滚到已知正常版本

---

## 11. 扩展阅读

### 11.1 官方文档

- [Unreal Engine Packaging Projects](https://docs.unrealengine.com/5.3/en-US/packaging-unreal-engine-projects/)
- [Dedicated Server Guide](https://docs.unrealengine.com/5.3/en-US/setting-up-dedicated-servers-in-unreal-engine/)
- [Build Configuration Reference](https://docs.unrealengine.com/5.3/en-US/build-configuration-for-unreal-engine/)

### 11.2 社区资源

- [Unreal Slackers Discord](https://unrealslackers.org/)
- [r/unrealengine](https://www.reddit.com/r/unrealengine/)
- [Unreal Engine Forums - Networking](https://forums.unrealengine.com/c/development-discussion/networking/17)

### 11.3 工具推荐

- **CI/CD:** Jenkins, GitHub Actions, GitLab CI
- **容器化:** Docker, Kubernetes
- **监控:** Grafana, Prometheus
- **崩溃报告:** Sentry, BugSplat
- **CDN:** AWS CloudFront, Cloudflare

---

## 总结

打包发布是游戏开发的最后一公里,也是容易被忽视但极其重要的环节。Lyra 项目通过精心设计的 Target 文件系统、模块化的插件管理、完善的配置分层,展示了商业级项目的打包最佳实践。

关键要点:

1. **架构设计** - 使用多 Target 支持不同平台和需求
2. **自动化流程** - 通过 CI/CD 提升效率和一致性
3. **性能优化** - Shipping 配置的精细调优
4. **安全防护** - 加密、签名、代码混淆
5. **运维能力** - 监控、日志、崩溃报告

无论你是独立开发者还是团队项目,掌握这些打包发布技能,都能让你的游戏更快、更稳定地交付到玩家手中。记住:好的发布流程是好游戏体验的保障!

---

**下一步:**

现在你已经掌握了 Lyra 的打包发布流程,可以继续学习:
- **第22篇:相机系统** - 探索 Lyra 的自适应视角控制
- **第28篇:实战项目 - MOBA 模式开发** - 综合运用所有技术构建完整游戏模式

继续探索 Lyra 的世界吧! 🚀
