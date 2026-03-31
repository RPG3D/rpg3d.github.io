---
title: UnrealSharp 插件技术深度解析 - 异步编程与委托系统
date: 2026-03-30 16:00:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, C#, async, await, Delegate]
series: UnrealSharp 插件技术深度解析
series_number: 11
---

# 异步编程与事件系统 - 跨语言的协作模式

> **技术深度**：⭐⭐⭐⭐
> **前置知识**：C# async/await、UE 异步系统、委托模式

---

## 引言

现代游戏开发中，异步编程和事件系统是必不可少的。UnrealSharp 实现了 C# async/await 与 UE 异步系统的无缝对接，以及完整的委托桥接机制。本章将深入剖析这些实现。

---

## 一、异步支持概述

### 1.1 UnrealSharpAsync 模块

```
Source/UnrealSharpAsync/
├── Public/
│   ├── UnrealSharpAsync.h
│   ├── CSAsyncActionBase.h
│   ├── CSAsyncLoadPrimaryDataAssets.h
│   └── CSAsyncLoadSoftObjectPtr.h
└── Private/
    ├── UnrealSharpAsync.cpp
    ├── CSAsyncActionBase.cpp
    └── ...
```

### 1.2 异步操作基类

```cpp
// Source/UnrealSharpAsync/Public/CSAsyncActionBase.h

UCLASS()
class UCSAsyncActionBase : public UObject
{
    GENERATED_BODY()
public:
    UFUNCTION(meta = (ScriptMethod))
    void Destroy();
    
protected:
    friend class UUCSAsyncBaseExporter;
    
    // 调用托管回调
    void InvokeManagedCallback(bool bDispose = true);
    void InvokeManagedCallback(UObject* WorldContextObject, bool bDispose = true);
    
    // 初始化托管回调
    void InitializeManagedCallback(FGCHandleIntPtr Callback);
    
    // 托管回调委托
    FCSManagedDelegate ManagedCallback;
};
```

---

## 二、C# async/await 与 UE 异步对接

### 2.1 异步操作导出器

```cpp
// Source/UnrealSharpCore/Public/Export/AsyncExporter.h

UCLASS()
class UAsyncExporter : public UObject
{
    GENERATED_BODY()
public:
    // 在指定线程执行
    UNREALSHARP_FUNCTION()
    static void RunOnThread(TWeakObjectPtr<UObject> WorldContextObject, 
                           ENamedThreads::Type Thread, 
                           FGCHandleIntPtr DelegateHandle);
    
    // 获取当前线程
    UNREALSHARP_FUNCTION()
    static int GetCurrentNamedThread();
};
```

### 2.2 跨线程执行

```csharp
// C# 异步操作示例
public async Task<FVector> CalculatePathAsync(FVector start, FVector end)
{
    // 切换到后台线程
    await Task.Run(() => 
    {
        // 耗时计算
        return Pathfinding.Calculate(start, end);
    });
    
    // 自动回到游戏线程继续执行
    return result;
}
```

**实现原理**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    异步执行流程                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────┐                    ┌─────────────┐
│ 游戏线程    │                    │ 后台线程    │
│ GameThread  │                    │ ThreadPool  │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       │ await Task.Run()                 │
       ├─────────────────────────────────→│
       │                                  │
       │                                  │ 执行计算
       │                                  │
       │     await 完成                   │
       │←─────────────────────────────────┤
       │                                  │
       │ 继续执行                          │
       │                                  │
```

### 2.3 Latent Action 实现

UE 的 Latent Action 用于蓝图中的异步节点：

```cpp
// 异步加载资源示例
UCLASS()
class UCSAsyncLoadSoftObjectPtr : public UCSAsyncActionBase
{
    GENERATED_BODY()
public:
    static UCSAsyncLoadSoftObjectPtr* LoadSoftObjectPtr(
        UObject* WorldContextObject, 
        TSoftObjectPtr<UObject> SoftObjectPtr);
    
private:
    TSoftObjectPtr<UObject> SoftPtr;
};
```

---

## 三、委托系统

### 3.1 UE 委托类型

UE 提供了多种委托类型：

| 类型 | 说明 | C# 对应 |
|------|------|--------|
| 单播委托 | 一对一绑定 | `Delegate<T>` |
| 多播委托 | 一对多绑定 | `MulticastDelegate<T>` |
| 动态委托 | 支持序列化 | 特殊处理 |
| 事件 | 封装的多播 | `MulticastDelegate<T>` |

### 3.2 C# 委托基类

```csharp
// Source/UnrealSharp/UnrealSharp/Delegate.cs

public abstract class Delegate<TDelegate> : DelegateBase<TDelegate> 
    where TDelegate : Delegate
{
    public TWeakObjectPtr<UObject> TargetObject;
    public FName FunctionName;
    
    // 从原生内存读取
    public override void FromNative(IntPtr address, IntPtr nativeProperty)
    {
        FScriptDelegateExporter.CallGetDelegateInfo(address, 
            out IntPtr targetObjectPtr, out FName functionName);
        TargetObject = new TWeakObjectPtr<UObject>(targetObjectPtr);
        FunctionName = functionName;
    }
    
    // 写入原生内存
    public override void ToNative(IntPtr address)
    {
        UObject? targetObject = TargetObject.Object;
        FScriptDelegateExporter.CallMakeDelegate(address, 
            targetObject?.NativeObject ?? IntPtr.Zero, FunctionName);
    }
    
    // 绑定 UFunction
    public override void BindUFunction(UObject targetObject, FName functionName)
    {
        TargetObject = new TWeakObjectPtr<UObject>(targetObject);
        FunctionName = functionName;
    }
    
    // 是否已绑定
    public override bool IsBound => TargetObject.IsValid && !FunctionName.IsNone;
    
    // 清除绑定
    public override void Clear()
    {
        TargetObject = new TWeakObjectPtr<UObject>();
        FunctionName = FName.None;
    }
}
```

### 3.3 多播委托

```csharp
// 多播委托实现
public abstract class MulticastDelegate<TDelegate> : DelegateBase<TDelegate>
{
    private List<DelegateBinding> _bindings = new();
    
    // 添加绑定
    public override void Add(TDelegate handler)
    {
        if (handler.Target is not UObject targetObject)
            throw new ArgumentException("Handler must be a UFunction on a UObject");
            
        _bindings.Add(new DelegateBinding
        {
            TargetObject = new TWeakObjectPtr<UObject>(targetObject),
            FunctionName = new FName(handler.Method.Name)
        });
    }
    
    // 移除绑定
    public override void Remove(TDelegate handler)
    {
        // 查找并移除匹配的绑定
        // ...
    }
    
    // 广播
    protected override void ProcessDelegate(IntPtr parameters)
    {
        foreach (var binding in _bindings)
        {
            if (binding.TargetObject.IsValid)
            {
                // 调用 UFunction
                FScriptDelegateExporter.CallBroadcastDelegate(
                    binding.TargetObject.Object.NativeObject,
                    binding.FunctionName,
                    parameters
                );
            }
        }
    }
}
```

### 3.4 委托导出器

```cpp
// Source/UnrealSharpCore/Public/Export/FMulticastDelegatePropertyExporter.h

UCLASS()
class UFMulticastDelegatePropertyExporter : public UObject
{
    GENERATED_BODY()
public:
    // 添加委托绑定
    UNREALSHARP_FUNCTION()
    static void AddDelegate(FMulticastDelegateProperty* DelegateProperty, 
                           FMulticastScriptDelegate* Delegate, 
                           UObject* Target, 
                           const char* FunctionName);
    
    // 移除委托绑定
    UNREALSHARP_FUNCTION()
    static void RemoveDelegate(FMulticastDelegateProperty* DelegateProperty, 
                              FMulticastScriptDelegate* Delegate, 
                              UObject* Target, 
                              const char* FunctionName);
    
    // 广播委托
    UNREALSHARP_FUNCTION()
    static void BroadcastDelegate(FMulticastDelegateProperty* DelegateProperty, 
                                  const FMulticastScriptDelegate* Delegate, 
                                  void* Parameters);
    
    // 检查是否已绑定
    UNREALSHARP_FUNCTION()
    static bool ContainsDelegate(FMulticastDelegateProperty* DelegateProperty, 
                                 const FMulticastScriptDelegate* Delegate, 
                                 UObject* Target, 
                                 const char* FunctionName);
    
    // 清除所有绑定
    UNREALSHARP_FUNCTION()
    static void ClearDelegate(FMulticastDelegateProperty* DelegateProperty, 
                             FMulticastScriptDelegate* Delegate);
    
    // 是否有绑定
    UNREALSHARP_FUNCTION()
    static bool IsBound(FMulticastScriptDelegate* Delegate);

private:
    static FScriptDelegate MakeScriptDelegate(UObject* Target, const char* FunctionName)
    {
        FScriptDelegate NewDelegate;
        NewDelegate.BindUFunction(Target, FunctionName);
        return NewDelegate;
    }
};
```

---

## 四、委托封送详解

### 4.1 SinglecastDelegatePropertyTranslator

在代码生成阶段，委托类型会被正确翻译：

```cpp
// 单播委托翻译器
class FSinglecastDelegatePropertyTranslator : public FDelegateBasePropertyTranslator
{
protected:
    virtual void ExportPropertyRetrieval(...) override
    {
        // 生成获取委托的代码
        Builder.AppendLine($"{ContextName}.{Property->GetName()} = new {CSharpType}();");
        Builder.AppendLine($"{ContextName}.{Property->GetName()}.FromNative({NativePtr}, ...);");
    }
    
    virtual void ExportPropertyAssignment(...) override
    {
        // 生成设置委托的代码
        Builder.AppendLine($"{ManagedValue}.ToNative({NativePtr});");
    }
};
```

### 4.2 MulticastDelegatePropertyTranslator

```cpp
// 多播委托翻译器
class FMulticastDelegatePropertyTranslator : public FDelegateBasePropertyTranslator
{
protected:
    virtual void ExportPropertyRetrieval(...) override
    {
        // 多播委托需要遍历所有绑定
        Builder.AppendLine($"var delegate = {ContextName}.{Property->GetName()};");
        
        // 生成 Add/Remove/Broadcast 方法
        // ...
    }
};
```

### 4.3 生成的 C# 代码示例

```csharp
// 自动生成的委托属性
public partial class AActor
{
    // 单播委托属性
    public Delegate<OnActorClickedDelegate> OnActorClicked { get; set; }
    
    // 多播委托属性
    public MulticastDelegate<OnActorDestroyedDelegate> OnDestroyed { get; set; }
}

// 使用示例
public class MyGameMode : AGameModeBase
{
    public override void BeginPlay()
    {
        base.BeginPlay();
        
        // 绑定多播委托
        SomeActor.OnDestroyed.Add(OnActorDestroyed);
    }
    
    private void OnActorDestroyed()
    {
        // 处理 Actor 销毁
    }
    
    public override void EndPlay(EEndPlayReason reason)
    {
        // 取消绑定
        SomeActor.OnDestroyed.Remove(OnActorDestroyed);
        base.EndPlay(reason);
    }
}
```

---

## 五、事件订阅与触发

### 5.1 UE 事件系统

UE 的事件是基于多播委托的：

```cpp
// C++ 事件声明
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnGameStarted);

UCLASS()
class AMyGameMode : public AGameModeBase
{
    UPROPERTY(BlueprintAssignable)
    FOnGameStarted OnGameStarted;
};
```

### 5.2 C# 订阅事件

```csharp
// C# 订阅 UE 事件
[UClass]
public partial class MyGameMode : AGameModeBase
{
    public override void BeginPlay()
    {
        base.BeginPlay();
        
        // 订阅全局事件
        GEngine.OnWorldAdded.Add(OnWorldAdded);
    }
    
    private void OnWorldAdded(UWorld world)
    {
        // 处理世界添加事件
    }
}
```

### 5.3 触发事件

```csharp
// C# 触发事件
[UClass]
public partial class MyGameMode : AGameModeBase
{
    [UProperty]
    public MulticastDelegate<GameStartedDelegate> OnGameStarted { get; set; }
    
    public void StartGame()
    {
        // 触发事件
        OnGameStarted.Broadcast();
    }
}
```

---

## 六、异步资源加载

### 6.1 异步加载软引用

```csharp
// 异步加载软对象引用
public async Task<UObject> LoadAssetAsync(TSoftObjectPtr<UObject> softPtr)
{
    var tcs = new TaskCompletionSource<UObject>();
    
    // 创建异步操作
    var asyncAction = UCSAsyncLoadSoftObjectPtr.LoadSoftObjectPtr(this, softPtr);
    
    // 设置完成回调
    asyncAction.OnCompleted += (loadedObject) =>
    {
        tcs.SetResult(loadedObject);
    };
    
    return await tcs.Task;
}
```

### 6.2 批量资源加载

```csharp
// 异步加载多个资源
public async Task<TArray<UObject>> LoadAssetsAsync(TArray<TSoftObjectPtr<UObject>> softPtrs)
{
    var tasks = softPtrs.Select(ptr => LoadAssetAsync(ptr));
    var results = await Task.WhenAll(tasks);
    return new TArray<UObject>(results);
}
```

---

## 七、自定义异步操作

### 7.1 创建自定义异步操作

```cpp
// C++ 自定义异步操作
UCLASS()
class UCSAsyncMyOperation : public UCSAsyncActionBase
{
    GENERATED_BODY()
public:
    static UCSAsyncMyOperation* PerformAsyncOperation(
        UObject* WorldContextObject, 
        FMyParams Params);
    
    void Activate() override
    {
        // 启动异步操作
        // ...
        
        // 完成时调用
        InvokeManagedCallback(WorldContextObject);
    }
};
```

### 7.2 C# 端使用

```csharp
// C# 使用自定义异步操作
public async Task<string> PerformOperationAsync()
{
    var tcs = new TaskCompletionSource<string>();
    
    var operation = UCSAsyncMyOperation.PerformAsyncOperation(this, new FMyParams());
    operation.OnCompleted += (result) => tcs.SetResult(result);
    
    return await tcs.Task;
}
```

---

## 八、性能考量

### 8.1 异步操作开销

| 操作 | 开销 | 建议 |
|------|------|------|
| 线程切换 | ~10 µs | 避免频繁切换 |
| 委托调用 | ~50 ns | 比直接调用慢 5x |
| 事件广播 | ~100 ns/订阅者 | 控制订阅者数量 |

### 8.2 最佳实践

```csharp
// 不推荐：每帧切换线程
public override void Tick(float deltaTime)
{
    Task.Run(() => { /* 计算 */ }).Wait();  // 阻塞！
}

// 推荐：批量异步处理
public override void BeginPlay()
{
    ProcessAsync();
}

private async void ProcessAsync()
{
    while (true)
    {
        var result = await Task.Run(() => HeavyCalculation());
        UpdateFromResult(result);
        await Task.Delay(100);  // 控制频率
    }
}
```

### 8.3 委托优化

```csharp
// 不推荐：频繁 Add/Remove
void Update()
{
    actor.OnEvent.Add(Handler);    // 每帧添加
    actor.OnEvent.Remove(Handler); // 每帧移除
}

// 推荐：稳定绑定
void Start()
{
    actor.OnEvent.Add(Handler);
}

void OnDestroy()
{
    actor.OnEvent.Remove(Handler);
}
```

---

## 九、总结

### 核心要点

1. **async/await 集成**：C# 异步模式与 UE 线程系统无缝对接
2. **委托桥接**：完整的单播/多播委托支持
3. **事件系统**：C# 可订阅和触发 UE 事件
4. **异步资源加载**：支持 await 风格的资源加载
5. **性能优化**：理解跨边界开销，合理使用

### 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    异步编程架构                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         C# 层                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ async/await │  │ Task<T>     │  │ Delegate<T> │            │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │
└─────────┼────────────────┼────────────────┼────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Glue 层                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │AsyncExporter│  │ UCSAsync    │  │ Delegate    │            │
│  │             │  │ ActionBase  │  │ Exporter    │            │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │
└─────────┼────────────────┼────────────────┼────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         C++ 层                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ AsyncTask   │  │ LatentAction│  │ FScript     │            │
│  │             │  │             │  │ Delegate    │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### 关键源码文件

| 文件 | 职责 |
|------|------|
| `CSAsyncActionBase.h/cpp` | 异步操作基类 |
| `AsyncExporter.h` | 异步函数导出 |
| `FMulticastDelegatePropertyExporter.h` | 多播委托导出 |
| `Delegate.cs` | C# 委托基类 |
| `SinglecastDelegatePropertyTranslator.cs` | 单播委托翻译器 |
| `MulticastDelegatePropertyTranslator.cs` | 多播委托翻译器 |

---

**下一篇**：实战案例分析 - 从零创建一个C# GameMode
