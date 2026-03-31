---
title: UnrealSharp 插件技术深度解析 - 方法调用完整流程
date: 2026-03-30 14:00:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, C#, 方法调用, P/Invoke]
series: UnrealSharp 插件技术深度解析
series_number: 9
---

# 一个方法调用的前世今生 - 从C#到C++的完整旅程

> **技术深度**：⭐⭐⭐⭐⭐
> **前置知识**：前八篇文章所有内容

---

## 引言

经过前八篇文章的铺垫，我们已经掌握了 UnrealSharp 的所有核心组件。现在，让我们将这些知识串联起来，追踪一个方法调用的完整生命周期。

---

## 一、调用场景分类

### 1.1 四种调用场景

```
┌─────────────────────────────────────────────────────────────┐
│                     调用方向矩阵                             │
├─────────────────────────────────────────────────────────────┤
│  1. C# → C++ 原生函数                                        │
│     示例：UObject.GetName()                                 │
│                                                             │
│  2. C++ → C# 托管方法                                        │
│     示例：蓝图调用 C# 定义的 UFunction                        │
│                                                             │
│  3. 蓝图 → C# 方法                                           │
│     示例：蓝图节点调用 C# 函数                                │
│                                                             │
│  4. C# 重写 → 蓝图可调用函数                                  │
│     示例：C# 重写 BeginPlay，蓝图触发调用                     │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 调用链示意图

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   C# 层     │ ←→ │   Glue 层    │ ←→ │   C++ 层    │
│  (托管)     │     │  (桥接)     │     │  (原生)     │
└─────────────┘     └─────────────┘     └─────────────┘
      ↑                   ↑                   ↑
      │                   │                   │
  C# 方法            生成代码/回调         UE 反射类型
```

---

## 二、C# → C++ 调用链

### 2.1 简单属性访问

**场景**：C# 获取 UObject 的名称

```csharp
// C# 代码
UObject obj = ...;
FName name = obj.GetName();
```

**调用链**：

```
1. C# UObject.GetName()
   ↓
2. 生成的 Glue 代码
   public FName GetName()
   {
       return UObjectExporter.CallGetName(NativeObject);
   }
   ↓
3. UObjectExporter.CallGetName(IntPtr nativeObject)
   ↓
4. 函数指针调用
   GetName(nativeObject)  // delegate* unmanaged<IntPtr, FName>
   ↓
5. C++ UUObjectExporter::GetName(UObject* Object)
   {
       return Object->GetFName();
   }
   ↓
6. 返回 FName（内存拷贝到 C#）
```

**时间分析**：

| 步骤 | 操作 | 开销 |
|------|------|------|
| 1-2 | C# 方法调用 | ~1 ns |
| 3 | 函数指针调用 | ~2 ns |
| 4 | C++ 函数执行 | ~10 ns |
| 5 | 返回值拷贝 | ~5 ns (FName 是值类型) |
| **总计** | | **~18 ns** |

### 2.2 带参数的函数调用

**场景**：C# 调用 KismetSystemLibrary 的打印日志

```csharp
// C# 代码
UKismetSystemLibrary.PrintString(this, "Hello World!");
```

**调用链**：

```
1. C# UKismetSystemLibrary.PrintString()
   ↓
2. 参数准备
   - FString* NativeString = ... (字符串封送)
   - UObject* WorldContext = ...
   ↓
3. UFunctionExporter 获取 UFunction
   UFunction* Func = UClassExporter.CallGetNativeFunctionFromClassAndName(
       UKismetSystemLibrary::StaticClass(), "PrintString");
   ↓
4. 参数结构体初始化
   void* Params = FMemoryAllocator.Allocate(Func.ParamsSize);
   ↓
5. 填充参数
   // 在参数结构体偏移处写入值
   ↓
6. 调用 UFunction
   UObject.CallFunction(Func, Params);
   ↓
7. 清理参数内存
   FMemoryAllocator.Free(Params);
```

### 2.3 数组操作

**场景**：C# 访问 TArray 属性

```csharp
// C# 代码
TArray<FVector> points = myComponent.Points;
points.Add(new FVector(1, 2, 3));
```

**调用链**：

```
1. 获取属性地址
   IntPtr propAddr = FPropertyExporter.CallGetNativePropertyFromName(
       NativeStruct, "Points");
   ↓
2. 获取数组数据指针
   void* arrayData = FArrayPropertyExporter.CallGetArrayData(propAddr);
   ↓
3. 获取数组长度
   int length = FArrayPropertyExporter.CallGetArrayLength(propAddr);
   ↓
4. 添加元素
   FArrayPropertyExporter.CallAddToArray(propAddr, arrayData);
   ↓
5. 设置新元素值
   // 在数组末尾写入 FVector 数据
   IntPtr elemAddr = FArrayPropertyExporter.CallGetArrayElement(propAddr, length);
   FMemory.CopyTo(elemAddr, newVectorData, sizeof(FVector));
```

---

## 三、C++ → C# 调用链

### 3.1 托管回调系统回顾

```cpp
// C++ 回调结构
struct FManagedCallbacks
{
    void (*CreateNewManagedObject)(UObject*, UClass*, FGCHandle*);
    void (*InvokeManagedMethod)(FGCHandle, void*, void*);
    void (*LookupManagedMethod)(FGCHandle, const char*, void*);
    // ...
};
```

### 3.2 蓝图调用 C# 方法

**场景**：蓝图节点调用 C# 定义的 `TakeDamage` 函数

```csharp
// C# 代码
[UClass]
public partial class AMyCharacter : ACharacter
{
    [UFunction]
    public void TakeDamage(float damage)
    {
        Health -= damage;
    }
}
```

**调用链**：

```
1. 蓝图执行节点
   ProcessEvent(TakeDamageUFunction, &Params);
   ↓
2. UFunction 的 Native 执行
   // UFunction 被设置为调用 C# 实现
   ↓
3. UCSManager.InvokeManagedMethod()
   ↓
4. 查找托管方法
   ManagedCallbacks.LookupManagedMethod(
       TypeGCHandle, "TakeDamage", &MethodHandle);
   ↓
5. 准备参数
   void* Params = ... // damage 值
   ↓
6. 调用托管方法
   ManagedCallbacks.InvokeManagedMethod(
       MethodHandle, Params, ReturnValuePtr);
   ↓
7. C# 方法执行
   public void TakeDamage(float damage)
   {
       Health -= damage;  // 可能触发更多 C++ → C# 调用
   }
   ↓
8. 返回（如有）
```

### 3.3 虚函数重写

**场景**：C# 重写 `BeginPlay`

```csharp
// C# 代码
[UClass]
public partial class AMyActor : AActor
{
    public override void BeginPlay()
    {
        base.BeginPlay();
        // C# 逻辑
    }
}
```

**调用链**：

```
1. UE 游戏启动，AActor.BeginPlay() 被调用
   ↓
2. 检查是否有托管重写
   // UClass 有标记表明有 C# 重写
   ↓
3. 调用托管的 BeginPlay
   UCSManager.InvokeManagedMethod(...);
   ↓
4. C# BeginPlay 执行
   - 可选调用 base.BeginPlay()
   - 执行自定义逻辑
   ↓
5. base.BeginPlay() 触发 C++ → C# → C++ 链
   // 调用父类 C++ 实现
```

---

## 四、蓝图 ↔ C# 交互

### 4.1 C# 方法对蓝图可见

```csharp
// C# 代码
[UClass]
public partial class AMyActor : AActor
{
    // 蓝图可调用
    [UFunction]
    public void BlueprintCallableFunction() { }
    
    // 蓝图可实现（C# 提供默认实现）
    [UFunction]
    public virtual void BlueprintImplementableEvent() { }
    
    // 蓝图纯虚（必须由蓝图实现）
    [UFunction]
    public abstract void BlueprintNativeEvent();
}
```

### 4.2 蓝图调用 C# 的条件

1. **UFunction 标记**：方法必须有 `[UFunction]` 特性
2. **正确注册**：类型通过 UCSManagedClassCompiler 编译
3. **函数签名匹配**：参数类型必须是支持的类型

### 4.3 C# 调用蓝图函数

```csharp
// C# 调用蓝图定义的函数
UFunction func = UClassExporter.CallGetNativeFunctionFromClassAndName(
    NativeClass, "BlueprintDefinedFunction");
    
if (func != null)
{
    void* params = ...;
    UObject.CallFunction(func, params);
}
```

---

## 五、完整示例：一个 Tick 的旅程

让我们追踪一个完整的 Tick 调用：

### 5.1 C# Actor 类定义

```csharp
[UClass]
public partial class AMyActor : AActor
{
    private float _elapsedTime;
    
    public override void Tick(float deltaTime)
    {
        base.Tick(deltaTime);
        
        _elapsedTime += deltaTime;
        
        // 调用 C++ API
        SetActorLocation(new FVector(
            Math.Sin(_elapsedTime) * 100,
            0, 100
        ), false, IntPtr.Zero, false);
    }
}
```

### 5.2 调用时序图

```
时间轴    C++ 层                    Glue 层                 C# 层
  │
  │     游戏循环开始
  │          │
  │          ▼
  │     AActor::Tick()
  │          │
  │          │ 检测到托管重写
  │          │
  │          ├──────────────────────────────→ 查找 Tick 方法
  │          │                                    │
  │          │                                    ▼
  │          │                              获取 MethodHandle
  │          │                                    │
  │          │   ←────────────────────────────────┘
  │          │
  │          ▼
  │     InvokeManagedMethod()
  │          │
  │          ├──────────────────────────────→ C# Tick(deltaTime)
  │          │                                    │
  │          │                                    ├── base.Tick(deltaTime)
  │          │                                    │      │
  │          │   ←────────────────────────────────┼──────┘
  │          │                                    │
  │          │                                    ├── _elapsedTime += deltaTime
  │          │                                    │
  │          │                                    ├── SetActorLocation(...)
  │          │   ←────────────────────────────────┤
  │          │                                    │
  │          ▼                                    │
  │     USceneComponent::SetWorldLocation()       │
  │          │                                    │
  │          ├──────────────────────────────→ 返回到 C#
  │          │                                    │
  │          │   ←────────────────────────────────┘
  │          │
  │     Tick 完成
  │
  ▼
```

### 5.3 性能分析

| 操作 | 调用次数 | 单次开销 | 总开销 |
|------|---------|---------|--------|
| C++ → C# 回调 | 1 | ~50 ns | ~50 ns |
| base.Tick() | 1 | ~30 ns | ~30 ns |
| SetActorLocation | 1 | ~100 ns | ~100 ns |
| **总计** | | | **~180 ns** |

**对比**：纯 C++ Tick 约为 10-20 ns，C# 增加的开销约 10x。

---

## 六、性能优化策略

### 6.1 减少跨边界调用

**问题代码**：
```csharp
// 每帧多次跨边界调用
for (int i = 0; i < 100; i++)
{
    FVector pos = actors[i].GetActorLocation();
    // ...
}
```

**优化方案**：
```csharp
// 批量获取，减少调用
IntPtr[] nativePtrs = actors.Select(a => a.NativeObject).ToArray();
FVector[] positions = USceneComponentExporter.CallGetWorldLocations(nativePtrs);
```

### 6.2 使用 Blittable 类型

```csharp
// 避免：需要封送
void ProcessString(string text);

// 推荐：直接传递指针
void ProcessString(IntPtr nativeStringPtr);
```

### 6.3 缓存 UFunction

```csharp
// 不推荐：每次查找
public void DoSomething()
{
    var func = GetFunction("SomeFunction");
    CallFunction(func);
}

// 推荐：缓存
private static UFunction _cachedFunc;
public void DoSomething()
{
    _cachedFunc ??= GetFunction("SomeFunction");
    CallFunction(_cachedFunc);
}
```

### 6.4 避免频繁字符串转换

```csharp
// 不推荐
for (int i = 0; i < 1000; i++)
{
    string name = obj.GetName().ToString(); // 每次转换
}

// 推荐
FName name = obj.GetName(); // 保持原生类型
// 只在需要时转换
string nameStr = name.ToString();
```

---

## 七、调试技巧

### 7.1 启用调用日志

```cpp
// DefaultEngine.ini
[/Script/UnrealSharpCore.CSDeveloperSettings]
bLogManagedCalls=true
bLogCrossBoundaryCalls=true
```

### 7.2 使用调试命令

```
// 控制台命令
UnrealSharp.DumpCallStack
UnrealSharp.ProfileCalls [duration]
```

### 7.3 性能分析

使用 Unreal Insights 追踪跨边界调用：

```
// 标记调用边界
TRACE_CPUPROFILER_EVENT_SCOPE(ManagedToNativeCall);
```

---

## 八、常见问题与解决

### 8.1 调用失败：方法未找到

**原因**：C# 方法未正确注册

**解决**：
1. 确认 `[UFunction]` 特性存在
2. 检查类型是否已编译
3. 使用 `UnrealSharp.DumpClass <ClassName>` 验证

### 8.2 参数封送错误

**原因**：C# 和 C++ 参数类型不匹配

**解决**：
1. 确保使用 Blittable 类型
2. 检查生成的 Glue 代码
3. 使用调试输出验证参数值

### 8.3 内存泄漏

**原因**：未释放非托管资源

**解决**：
1. 使用 `using` 语句管理非托管句柄
2. 确保回调注销
3. 检查 GCHandle 使用

---

## 九、总结

### 核心要点

1. **四种调用场景**各有特点，理解调用方向是优化基础
2. **C# → C++** 通过函数指针实现零开销调用
3. **C++ → C#** 依赖托管回调系统
4. **蓝图交互**需要正确使用 `[UFunction]` 特性
5. **性能优化**关键是减少跨边界调用次数

### 调用流程总图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         方法调用完整生命周期                              │
└─────────────────────────────────────────────────────────────────────────┘

                    ┌──────────────────┐
                    │   C# 方法调用     │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ 属性访问  │  │ 函数调用  │  │ 虚函数    │
        └────┬─────┘  └────┬─────┘  └────┬─────┘
             │             │             │
             ▼             ▼             ▼
        ┌──────────────────────────────────────────┐
        │            Glue 代码层                    │
        │  ┌─────────────────────────────────────┐ │
        │  │  Exporter 函数指针调用               │ │
        │  │  参数封送/解封                       │ │
        │  └─────────────────────────────────────┘ │
        └──────────────────┬───────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────────┐
        │            C++ 原生层                     │
        │  ┌─────────────────────────────────────┐ │
        │  │  UE 反射系统                         │ │
        │  │  UObject/UFunction 操作              │ │
        │  │  可能触发 C++ → C# 回调              │ │
        │  └─────────────────────────────────────┘ │
        └──────────────────┬───────────────────────┘
                           │
                           ▼
                    ┌──────────────────┐
                    │    返回结果       │
                    └──────────────────┘
```

### 关键源码文件汇总

| 文件 | 章节 | 职责 |
|------|------|------|
| `CSManagedCallbacksCache.h` | 第四篇 | 托管回调缓存 |
| `CSBindsManager.h` | 第八篇 | 函数绑定管理 |
| `UnmanagedCallbacks.cs` | 第四篇 | C# 回调导出 |
| `NativeCallbacksWrapperGenerator.cs` | 第八篇 | 调用代码生成 |
| `CSManagedClassCompiler.cpp` | 第七篇 | 类编译与注册 |

---

**下一篇**：热重载与编辑器集成 - 开发效率的倍增器
