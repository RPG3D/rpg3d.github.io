---
title: UnrealSharp 插件技术深度解析 - 实战案例分析
date: 2026-03-30 17:00:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, C#, GameMode, 实战]
series: UnrealSharp 插件技术深度解析
series_number: 12
---

# 实战演练 - 创建你的第一个纯C# GameMode

> **技术深度**：⭐⭐⭐
> **前置知识**：前十一篇文章所有内容

---

## 引言

经过前面十一章的理论铺垫，现在让我们动手实践！本章将完整演示如何使用 UnrealSharp 创建一个纯 C# 的 GameMode，涵盖从类型定义到运行测试的全过程。

---

## 一、案例目标

### 1.1 功能需求

创建一个完整的 C# GameMode，包含：

- 自定义 GameMode 类
- 自定义 Pawn 类
- 重写 `BeginPlay`、`Tick` 等事件
- 绑定输入事件
- 与蓝图交互

### 1.2 项目结构

```
Script/
├── ManagedSharpDemo/
│   ├── AScriptGameMode.cs      # GameMode 类
│   ├── AScriptCharacter.cs     # Pawn 类
│   └── ...
├── SharpDemo.RuntimeGlue/
│   └── (生成的胶水代码)
└── ManagedSharpDemo.sln
```

---

## 二、逐步实现

### 2.1 第一步：C# 类定义与特性标注

#### AScriptGameMode.cs

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

using UnrealSharp;
using UnrealSharp.Attributes;
using UnrealSharp.Core;
using UnrealSharp.CoreUObject;
using UnrealSharp.Engine;
using UnrealSharp.EnhancedInput;

namespace ManagedSharpDemo
{
    [UClass]
    public partial class AScriptGameMode : AGameModeBase
    {
        public AScriptGameMode()
        {
            // 设置默认 Pawn 类
            DefaultPawnClass = typeof(AScriptCharacter);
        }

        public override void BeginPlay()
        {
            base.BeginPlay();
            
            // C# 逻辑
            LogUnrealSharp.Log("AScriptGameMode BeginPlay!");
        }
    }
}
```

#### AScriptCharacter.cs

```csharp
using UnrealSharp;
using UnrealSharp.Attributes;
using UnrealSharp.Core;
using UnrealSharp.Engine;
using UnrealSharp.EnhancedInput;

namespace ManagedSharpDemo
{
    [UClass]
    public partial class AScriptCharacter : ACharacter
    {
        // 输入动作引用
        [UProperty]
        public UInputAction MoveAction { get; set; }
        
        [UProperty]
        public UInputAction JumpAction { get; set; }
        
        // 移动速度
        [UProperty]
        public float MoveSpeed { get; set; } = 500.0f;
        
        // 输入映射上下文
        [UProperty]
        public UInputMappingContext InputMappingContext { get; set; }

        public AScriptCharacter()
        {
            // 设置默认值
            MoveSpeed = 600.0f;
        }

        public override void BeginPlay()
        {
            base.BeginPlay();
            
            // 绑定输入
            if (InputMappingContext != null)
            {
                var playerController = GetPlayerController();
                if (playerController != null)
                {
                    var inputComponent = playerController.InputComponent;
                    if (inputComponent != null)
                    {
                        // 添加输入映射上下文
                        inputComponent.AddMappingContext(InputMappingContext, 0);
                    }
                }
            }
            
            LogUnrealSharp.Log("AScriptCharacter BeginPlay!");
        }

        // 绑定输入动作
        [UFunction]
        public void Move(const FInputActionValue& value)
        {
            var movementVector = value.GetAxis2D();
            
            // 获取控制器旋转
            var controller = GetPlayerController();
            if (controller == null) return;
            
            var rotation = controller.GetControlRotation();
            var yawRotation = new FRotator(0, rotation.Yaw, 0);
            
            // 计算移动方向
            var forwardDirection = FRotationMatrix.GetUnitAxis(
                yawRotation, EAxis.X);
            var rightDirection = FRotationMatrix.GetUnitAxis(
                yawRotation, EAxis.Y);
            
            // 添加移动输入
            AddMovementInput(forwardDirection, movementVector.Y);
            AddMovementInput(rightDirection, movementVector.X);
        }

        [UFunction]
        public void Jump(const FInputActionValue& value)
        {
            if (value.Get<bool>())
            {
                Jump();
            }
        }
    }
}
```

### 2.2 第二步：编译与 Glue 代码生成

保存文件后，UnrealSharp 会自动：

1. **检测文件变更**：`IDirectoryWatcher` 监视 `.cs` 文件
2. **触发编译**：调用 `dotnet build`
3. **生成 Glue 代码**：`UnrealSharpManagedGlue` 分析程序集

#### 生成的 Glue 代码示例

```csharp
// Script/SharpDemo.RuntimeGlue/ManagedSharpDemo/AScriptGameMode.cs
// (自动生成)

namespace ManagedSharpDemo
{
    public partial class AScriptGameMode : AGameModeBase
    {
        static AScriptGameMode()
        {
            // 注册类
            NativeBinds.RegisterClass(
                "ManagedSharpDemo",
                "ManagedSharpDemo",
                "AScriptGameMode",
                typeof(AScriptGameMode)
            );
        }
        
        // 构造函数入口点
        [UnmanagedCallersOnly]
        public static void Constructor(IntPtr nativePtr)
        {
            var obj = new AScriptGameMode();
            obj.NativeObject = nativePtr;
            // ... 初始化代码
        }
        
        // BeginPlay 托管入口点
        [UnmanagedCallersOnly]
        public static void BeginPlay_Callback(IntPtr nativePtr)
        {
            var obj = GetManagedObject<AScriptGameMode>(nativePtr);
            obj.BeginPlay();
        }
    }
}
```

### 2.3 第三步：UE 类型注册验证

编译完成后，在 UE 编辑器中验证类型注册：

```
控制台命令：
UnrealSharp.DumpClass AScriptGameMode
```

**预期输出**：

```
LogUnrealSharp: Class: AScriptGameMode
LogUnrealSharp:   Parent: AGameModeBase
LogUnrealSharp:   Assembly: ManagedSharpDemo
LogUnrealSharp:   Properties:
LogUnrealSharp:     - DefaultPawnClass (ObjectProperty) (BlueprintReadOnly)
LogUnrealSharp:   Functions:
LogUnrealSharp:     - BeginPlay (Event) (BlueprintCallable)
LogUnrealSharp:   Class Flags: (CLASS_MatchInProgress, ...)
```

### 2.4 第四步：蓝图配置

1. **创建蓝图**：
   - 右键 Content Browser → Blueprint Class
   - 选择 `AScriptGameMode` 作为父类
   - 命名为 `BP_ScriptGameMode`

2. **创建 Pawn 蓝图**：
   - 右键 → Blueprint Class
   - 选择 `AScriptCharacter` 作为父类
   - 命名为 `BP_ScriptCharacter`

3. **配置输入**：
   - 创建 Input Action（Move, Jump）
   - 创建 Input Mapping Context
   - 在蓝图中设置属性

4. **设置 GameMode**：
   - 打开 World Settings
   - GameMode Override → 选择 `BP_ScriptGameMode`

### 2.5 第五步：运行测试

点击 Play 按钮，观察日志输出：

```
LogUnrealSharp: AScriptGameMode BeginPlay!
LogUnrealSharp: AScriptCharacter BeginPlay!
```

**验证功能**：
- ✅ GameMode 正确加载
- ✅ Pawn 正确生成
- ✅ 输入响应正常
- ✅ C# 逻辑执行

---

## 三、关键代码解析

### 3.1 [UClass] 特性

```csharp
[UClass]
public partial class AScriptGameMode : AGameModeBase
```

**作用**：
- 标记类为 UE 反射类型
- 触发 Glue 代码生成
- 注册到 UE 类型系统

**生成的 C++ 等价**：

```cpp
UCLASS()
class AScriptGameMode : public AGameModeBase
{
    GENERATED_BODY()
    // ...
};
```

### 3.2 [UProperty] 特性

```csharp
[UProperty]
public float MoveSpeed { get; set; } = 500.0f;
```

**作用**：
- 声明 UE 反射属性
- 支持蓝图访问
- 支持序列化

**属性修饰符**：

```csharp
[UProperty(PropertyFlags.EditAnywhere | PropertyFlags.BlueprintReadWrite)]
public float MoveSpeed { get; set; }
```

### 3.3 [UFunction] 特性

```csharp
[UFunction]
public void Move(const FInputActionValue& value)
{
    // ...
}
```

**作用**：
- 声明 UE 反射函数
- 支持蓝图调用
- 支持远程调用

**函数修饰符**：

```csharp
[UFunction(FunctionFlags.BlueprintCallable)]
public void MyFunction() { }

[UFunction(FunctionFlags.BlueprintImplementableEvent)]
public virtual void BlueprintEvent() { }

[UFunction(FunctionFlags.BlueprintNativeEvent)]
public void NativeEvent();
```

### 3.4 构造函数

```csharp
public AScriptGameMode()
{
    DefaultPawnClass = typeof(AScriptCharacter);
}
```

**对应 C++ 的构造函数**：

```cpp
AScriptGameMode::AScriptGameMode()
{
    DefaultPawnClass = AScriptCharacter::StaticClass();
}
```

### 3.5 重写虚函数

```csharp
public override void BeginPlay()
{
    base.BeginPlay();
    // 自定义逻辑
}
```

**机制**：
- C# 重写方法替换 C++ 虚函数实现
- `base.BeginPlay()` 调用父类 C++ 实现
- 形成调用链：`UE → C++ → C#`

---

## 四、与蓝图交互

### 4.1 C# 调用蓝图函数

```csharp
[UClass]
public partial class MyActor : AActor
{
    public void CallBlueprintFunction()
    {
        // 查找蓝图定义的函数
        UFunction func = UClassExporter.CallGetNativeFunctionFromClassAndName(
            NativeClass, "BlueprintDefinedFunction");
            
        if (func != null)
        {
            // 准备参数
            var parms = new FBlueprintFunctionParams();
            parms.SetInt("Param1", 42);
            
            // 调用函数
            UObject.CallFunction(func, parms.Ptr);
        }
    }
}
```

### 4.2 蓝图调用 C# 函数

```csharp
[UClass]
public partial class MyActor : AActor
{
    // 蓝图可调用
    [UFunction(FunctionFlags.BlueprintCallable)]
    public void TakeDamage(float damage)
    {
        Health -= damage;
    }
    
    // 蓝图可实现（蓝图重写）
    [UFunction(FunctionFlags.BlueprintImplementableEvent)]
    public virtual void OnDamaged(float damage) { }
    
    // 蓝图原生事件（C# 默认实现 + 蓝图可扩展）
    [UFunction(FunctionFlags.BlueprintNativeEvent)]
    public void OnDeath();
    
    // 默认实现
    public void OnDeath_Implementation()
    {
        // C# 默认逻辑
    }
}
```

### 4.3 数据交换

```csharp
// C# 定义数据结构
[UStruct]
public struct FPlayerData
{
    [UProperty]
    public FString Name { get; set; }
    
    [UProperty]
    public int Score { get; set; }
    
    [UProperty]
    public FVector Location { get; set; }
}

// 使用
[UClass]
public partial class AMyGameMode : AGameModeBase
{
    [UProperty]
    public TArray<FPlayerData> Players { get; set; }
    
    public void AddPlayer(FPlayerData data)
    {
        Players.Add(data);
    }
}
```

---

## 五、常见问题与调试

### 5.1 类型未注册

**症状**：
```
LogScript: Warning: ScriptClass not found: AScriptGameMode
```

**排查**：
1. 检查 `[UClass]` 特性是否存在
2. 检查命名空间是否正确
3. 使用 `UnrealSharp.DumpAssembly <Name>` 验证
4. 重新生成解决方案

### 5.2 方法调用失败

**症状**：
```
LogUnrealSharp: Warning: Failed to find method: BeginPlay
```

**排查**：
1. 检查 `[UFunction]` 特性
2. 检查函数签名是否匹配
3. 检查函数是否为 public
4. 使用 `UnrealSharp.DumpClass <Name>` 检查函数列表

### 5.3 属性访问问题

**症状**：
```
LogUnrealSharp: Warning: Property not found: MoveSpeed
```

**排查**：
1. 检查 `[UProperty]` 特性
2. 检查属性是否为 public
3. 检查属性类型是否支持
4. 检查生成的 Glue 代码

### 5.4 热重载失效

**症状**：
修改 C# 代码后没有生效

**排查**：
1. 检查热重载状态：`UnrealSharp.DumpHotReloadState`
2. 检查编译是否成功
3. 检查是否有编译错误
4. 手动触发热重载：`UnrealSharp.HotReload`

---

## 六、性能监控

### 6.1 添加性能计数器

```csharp
[UClass]
public partial class AScriptGameMode : AGameModeBase
{
    private int _tickCount;
    private float _totalTime;
    
    public override void Tick(float deltaTime)
    {
        base.Tick(deltaTime);
        
        _tickCount++;
        _totalTime += deltaTime;
        
        // 每 60 帧输出一次
        if (_tickCount % 60 == 0)
        {
            LogUnrealSharp.Log($"Average frame time: {_totalTime / _tickCount * 1000:F2}ms");
        }
    }
}
```

### 6.2 使用 Unreal Insights

1. 启动 Unreal Insights
2. 连接到游戏进程
3. 追踪 C# 函数调用
4. 分析性能瓶颈

---

## 七、总结

### 完整流程回顾

```
┌─────────────────────────────────────────────────────────────────┐
│                    实战案例完整流程                               │
└─────────────────────────────────────────────────────────────────┘

1. C# 类定义
   ├── [UClass] 标记类
   ├── [UProperty] 声明属性
   ├── [UFunction] 声明函数
   └── 重写虚函数

2. 编译与生成
   ├── dotnet build
   ├── UHT 分析程序集
   └── 生成 Glue 代码

3. 类型注册
   ├── FCSManagedTypeDefinition 创建
   ├── UCSManagedClassCompiler 编译
   └── RegisterFieldToLoader 注册

4. 蓝图配置
   ├── 创建蓝图子类
   ├── 设置属性
   └── 配置输入

5. 运行测试
   ├── UE 加载 GameMode
   ├── 调用 C# BeginPlay
   └── 响应输入事件
```

### 关键源码文件

| 文件 | 职责 |
|------|------|
| `AScriptGameMode.cs` | GameMode 实现 |
| `AScriptCharacter.cs` | Pawn 实现 |
| `SharpDemo.RuntimeGlue/*` | 生成的 Glue 代码 |
| `CSManagedClassCompiler.cpp` | 类编译 |

---

**下一篇**：性能优化与最佳实践 - 让C#飞起来
