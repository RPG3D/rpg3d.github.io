---
title: 跨越边界的引用 - GCHandle 与托管对象生命周期管理
date: 2026-03-29 21:30:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, C#, GCHandle, 内存管理]
series: UnrealSharp 插件技术深度解析
series_number: 4
---

# 跨越边界的引用 - GCHandle 与托管对象生命周期管理

> **作者**：GLM-5.0

在上一章中，我们了解了 .NET 运行时是如何嵌入到 UE 进程中的。运行时初始化完成后，下一个关键问题是：**C++ 的 UObject 和 C# 的托管对象如何保持生命周期同步？** 这正是本章要深入剖析的核心问题。

## 一、问题引入：为什么需要 GCHandle？

### 1.1 你的练习代码中的局限

回顾你的 CoreClrDemo 练习代码：

```cpp
// 你的练习代码只能传递简单类型
typedef void (*PrintMessageFunc)(const char* message);
PrintMessageFunc func = (PrintMessageFunc)get_function_pointer(...);
func("Hello from native!");
```

这种方式有以下局限：
- 只能传递基本类型（int, float, 指针等）
- 无法传递复杂对象
- 无法让 C++ 持有 C# 对象的引用
- 无法让 C# 对象的生命周期与 C++ 对象绑定

### 1.2 GCHandle 的本质

**GCHandle（Garbage Collector Handle）** 是 .NET 提供的机制，允许非托管代码持有托管对象的引用，防止 GC 回收该对象。

```
┌─────────────────────────────────────────────────────────────────┐
│                        托管堆 (Managed Heap)                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  C# Object (UnrealSharpObject)                           │  │
│  │  - NativeObject: IntPtr                                   │  │
│  │  - 其他托管字段...                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          ▲                                      │
│                          │ GCHandle (强引用)                     │
│                          │                                      │
└──────────────────────────│──────────────────────────────────────┘
                           │
┌──────────────────────────│──────────────────────────────────────┐
│                     非托管内存 (Native)                          │
│  ┌───────────────────────┴──────────────────────────────────┐  │
│  │  FGCHandle 结构体                                         │  │
│  │  - Handle: uint8* (指向 GCHandle 的指针)                   │  │
│  │  - Type: GCHandleType                                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                      │
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  UObject (AActor, UActorComponent, etc.)                 │  │
│  │  - UObject 基类字段                                        │  │
│  │  - Native 属性...                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 GCHandle 的四种类型

| 类型 | 说明 | GC 行为 | UnrealSharp 用途 |
|-----|------|--------|-----------------|
| **Weak** | 弱引用 | 可以被 GC 回收 | 类型句柄、方法句柄 |
| **Strong** | 强引用 | 不会被 GC 回收 | 托管对象实例 |
| **Pinned** | 固定引用 | 不会被 GC 回收且不移动 | 传递给非托管代码的缓冲区 |
| **Null** | 空句柄 | 无效句柄 | 默认值 |

---

## 二、UnrealSharp 的 GCHandle 封装

### 2.1 FGCHandle 结构体（C++ 侧）

```cpp
// CSManagedGCHandle.h:5-11
enum class GCHandleType : char
{
    Null,
    StrongHandle,
    WeakHandle,
    PinnedHandle,
};
```

```cpp
// CSManagedGCHandle.h:31-86
struct FGCHandle
{
    FGCHandleIntPtr Handle;        // 指向托管 GCHandle 的指针
    GCHandleType Type = GCHandleType::Null;

    static FGCHandle InvalidHandle() { return FGCHandle(nullptr, GCHandleType::Null); }

    bool IsNull() const { return !Handle.IntPtr; }
    bool IsWeakPointer() const { return Type == GCHandleType::WeakHandle; }
    
    FGCHandleIntPtr GetHandle() const { return Handle; }
    uint8* GetPointer() const { return Handle.IntPtr; };
    
    // 释放 GCHandle，通知 C# 侧
    void Dispose(FGCHandleIntPtr AssemblyHandle = FGCHandleIntPtr())
    {
        TRACE_CPUPROFILER_EVENT_SCOPE(FGCHandle::Dispose);
        
        if (!Handle.IntPtr || Type == GCHandleType::Null)
        {
            return;
        }

        // 调用 C# 的 Dispose 回调
        FCSManagedCallbacks::ManagedCallbacks.Dispose(Handle, AssemblyHandle);
        Invalidate();
    }
    
    void Invalidate()
    {
        Handle.IntPtr = nullptr;
        Type = GCHandleType::Null;
    }
    
    // ... 构造函数和运算符重载
};
```

**设计亮点**：
- 轻量级：只存储一个指针和一个枚举，无虚函数开销
- 类型安全：通过 `GCHandleType` 区分句柄类型
- RAII 风格：`Dispose()` 方法确保资源释放

### 2.2 FScopedGCHandle：RAII 封装

```cpp
// CSManagedGCHandle.h:88-108
struct FScopedGCHandle
{
    FGCHandleIntPtr Handle;

    explicit FScopedGCHandle(FGCHandleIntPtr InHandle) : Handle(InHandle) {}
    
    // 禁止拷贝
    FScopedGCHandle(const FScopedGCHandle&) = delete;
    FScopedGCHandle(FScopedGCHandle&&) = delete;

    // 析构时自动释放
    ~FScopedGCHandle()
    {
        if (Handle.IntPtr != nullptr) 
        {
            FCSManagedCallbacks::ManagedCallbacks.FreeHandle(Handle);
        }
    }
    
    FScopedGCHandle& operator=(const FScopedGCHandle&) = delete;
    FScopedGCHandle& operator=(FScopedGCHandle&&) = delete;
};
```

**使用场景**：临时性的 GCHandle，离开作用域自动释放。

### 2.3 FSharedGCHandle：共享指针封装

```cpp
// CSManagedGCHandle.h:110-129
USTRUCT()
struct FSharedGCHandle
{
    GENERATED_BODY()

    FSharedGCHandle() = default;
    explicit FSharedGCHandle(FGCHandleIntPtr InHandle) 
        : Handle(MakeShared<FScopedGCHandle>(InHandle)) {}

    FGCHandleIntPtr GetHandle() const
    {
        if (Handle == nullptr) 
        {
            return FGCHandleIntPtr();
        }
        return Handle->Handle;
    }
    
private:
    TSharedPtr<FScopedGCHandle> Handle;
};
```

**使用场景**：需要在多个地方共享的 GCHandle，引用计数归零时自动释放。

---

## 三、GCHandleUtilities（C# 侧）

### 3.1 核心方法

```csharp
// GCHandleUtilities.cs:9-100
public static class GCHandleUtilities
{
    // 按 AssemblyLoadContext 分组的强引用字典
    private static readonly ConcurrentDictionary<AssemblyLoadContext, 
        ConcurrentDictionary<GCHandle, object>> StrongRefsByAssembly = new();

    // 创建强引用句柄（主重载：接受 AssemblyLoadContext）
    public static GCHandle AllocateStrongPointer(object value, AssemblyLoadContext loadContext)
    {
        // 先创建弱引用
        GCHandle weakHandle = GCHandle.Alloc(value, GCHandleType.Weak);

        // 获取或创建该 ALC 的强引用字典
        ConcurrentDictionary<GCHandle, object> strongReferences =
            StrongRefsByAssembly.GetOrAdd(loadContext, alcInstance =>
        {
#if !UNREALSHARP_MONO
            // CoreCLR: 订阅 ALC 卸载事件
            alcInstance.Unloading += OnAlcUnloading;
#endif
            return new ConcurrentDictionary<GCHandle, object>();
        });

        // 存储强引用（防止 GC 回收）
        strongReferences.TryAdd(weakHandle, value);
        return weakHandle;
    }

    // 便捷重载：接受 Assembly，内部通过 AssemblyLoadContext.GetLoadContext(assembly) 转换
    public static GCHandle AllocateStrongPointer(object value, Assembly assembly)
    {
        AssemblyLoadContext? assemblyLoadContext = AssemblyLoadContext.GetLoadContext(assembly);
        return AllocateStrongPointer(value, assemblyLoadContext!);
    }

    // 创建弱引用句柄
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static GCHandle AllocateWeakPointer(object value) => 
        GCHandle.Alloc(value, GCHandleType.Weak);

    // 释放句柄
    public static void Free(GCHandle handle, Assembly? assembly)
    {
        if (assembly != null)
        {
            AssemblyLoadContext? alc = AssemblyLoadContext.GetLoadContext(assembly);
            if (alc != null && StrongRefsByAssembly.TryGetValue(alc, out var strongRefs))
            {
                strongRefs.TryRemove(handle, out _);
            }
        }
        handle.Free();
    }
    
    // 从 IntPtr 获取对象
    public static T? GetObjectFromHandlePtr<T>(IntPtr handle)
    {
        if (handle == IntPtr.Zero) return default;
        
        GCHandle gcHandle = GCHandle.FromIntPtr(handle);
        if (!gcHandle.IsAllocated) return default;
        
        object? obj = gcHandle.Target;
        return obj is T typedObj ? typedObj : default;
    }
}
```

### 3.2 设计精髓：为什么强引用要按 ALC 分组？

**问题**：直接使用 `GCHandle.Alloc(obj, GCHandleType.Normal)` 不就可以防止 GC 回收吗？

**答案**：不够！热重载时需要卸载整个程序集。

```
┌─────────────────────────────────────────────────────────────────┐
│                    AssemblyLoadContext (ALC)                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  用户程序集 (ManagedSharpDemo.dll)                        │  │
│  │  - AScriptGameMode                                        │  │
│  │  - AScriptCharacter                                       │  │
│  │  - ...                                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                      │
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  StrongRefsByAssembly[ALC]                                │  │
│  │  - GCHandle1 → AScriptGameMode 实例1                      │  │
│  │  - GCHandle2 → AScriptCharacter 实例1                     │  │
│  │  - ...                                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

热重载时：
1. alc.Unloading 事件触发
2. StrongRefsByAssembly.TryRemove(alc, out _) 清除所有强引用
3. ALC 卸载，所有托管对象被 GC 回收
4. 重新加载新版本的程序集
```

---

## 四、对象生命周期同步机制

### 4.1 UObject → C# 对象映射表

```cpp
// CSManager.h:213-214
// Handles to all active UObjects that has a C# counterpart.
// The key is the unique ID of the UObject.
TMap<uint32, TSharedPtr<FGCHandle>> ManagedObjectHandles;
```

这是一个全局映射表，存储所有 UObject 对应的 C# 对象句柄：
- **Key**：`UObject::GetUniqueID()` 返回的唯一 ID
- **Value**：指向 C# 托管对象的 `FGCHandle` 的共享指针

### 4.2 监听 UObject 销毁事件

```cpp
// CSManager.h:49
UCLASS()
class UNREALSHARPCORE_API UCSManager : public UObject, 
    public FUObjectArray::FUObjectDeleteListener  // 继承监听器接口
{
    // ...
    
    // FUObjectDeleteListener 接口实现
    virtual void NotifyUObjectDeleted(const UObjectBase* Object, int32 Index) override;
    virtual void OnUObjectArrayShutdown() override 
    { 
        GUObjectArray.RemoveUObjectDeleteListener(this); 
    }
};
```

### 4.3 UObject 销毁时的同步释放

```cpp
// CSManager.cpp:313-345
void UCSManager::NotifyUObjectDeleted(const UObjectBase* Object, int32 Index)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(UCSManager::NotifyUObjectDeleted);

    // 1. 从映射表中移除并获取句柄
    TSharedPtr<FGCHandle> Handle;
    if (!ManagedObjectHandles.RemoveAndCopyValueByHash(Index, Index, Handle))
    {
        return;  // 该 UObject 没有对应的 C# 对象
    }

    // 2. 找到所属程序集
    UCSManagedAssembly* Assembly = FindOwningAssembly(Object->GetClass());
    if (!IsValid(Assembly))
    {
        UE_LOG(LogUnrealSharp, Error, 
            TEXT("Failed to find owning assembly for object %s. Will cause managed memory leak."), 
            *Object->GetFName().ToString());
        return;
    }

    // 3. 获取程序集句柄
    TSharedPtr<const FGCHandle> AssemblyHandle = Assembly->GetManagedAssemblyHandle();
    if (!AssemblyHandle.IsValid()) return;
    
    // 4. 调用 C# 的 Dispose 回调，释放 GCHandle
    Handle->Dispose(AssemblyHandle->GetHandle());

    // 5. 清理接口包装器（如果存在）
    TMap<uint32, TSharedPtr<FGCHandle>>* FoundHandles = 
        ManagedInterfaceWrappers.FindByHash(Index, Index);
    if (FoundHandles != nullptr)
    {
        // 释放所有接口包装器...
    }
}
```

**流程图**：

```
UObject 被销毁 (GC、Level 卸载等)
        │
        ▼
┌───────────────────────────────────┐
│ FUObjectArray 触发删除通知        │
│ NotifyUObjectDeleted()            │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 从 ManagedObjectHandles 查找      │
│ 对应的 FGCHandle                  │
└───────────────────────────────────┘
        │
        ├─ 未找到 → 返回（无 C# 对象）
        │
        ▼ 找到
┌───────────────────────────────────┐
│ 查找所属 UCSManagedAssembly       │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 调用 Handle->Dispose()            │
│ ↓                                 │
│ FCSManagedCallbacks::Dispose()    │
│ ↓                                 │
│ C#: UnmanagedCallbacks.Dispose()  │
│ ↓                                 │
│ GCHandleUtilities.Free()          │
│ ↓                                 │
│ handle.Free()                     │
└───────────────────────────────────┘
        │
        ▼
C# 对象不再被强引用，可被 GC 回收
```

---

## 五、托管对象的创建流程

### 5.1 FindManagedObject：查找或创建

```cpp
// CSManager.cpp:567-601
FGCHandle UCSManager::FindManagedObject(const UObject* Object)
{
    if (!IsValid(Object))
    {
        return FGCHandle::InvalidHandle();
    }

    uint32 ObjectID = Object->GetUniqueID();
    
    // 1. 先在映射表中查找
    if (TSharedPtr<FGCHandle>* FoundHandle = ManagedObjectHandles.FindByHash(ObjectID, ObjectID))
    {
#if WITH_EDITOR
        // 编辑器热重载时，C# 对象可能已被回收，需要检查
        TSharedPtr<FGCHandle> HandlePtr = *FoundHandle;
        if (HandlePtr.IsValid() && !HandlePtr->IsNull())
        {
            return *HandlePtr;
        }
#else
        return **FoundHandle;
#endif
    }

    // 2. 未找到，创建新的托管对象
    UCSManagedAssembly* OwningAssembly = FindOwningAssembly(Object->GetClass());
    if (!IsValid(OwningAssembly))
    {
        UE_LOGFMT(LogUnrealSharp, Error, "Failed to find assembly for {0}", *Object->GetName());
        return FGCHandle::InvalidHandle();
    }

    return *OwningAssembly->CreateManagedObjectFromNative(Object);
}
```

### 5.2 CreateManagedObjectFromNative：创建托管对象

```cpp
// CSManagedAssembly.cpp:194-226
TSharedPtr<FGCHandle> UCSManagedAssembly::CreateManagedObjectFromNative(const UObject* Object)
{
    // 1. 获取第一个非蓝图的 C++ 基类
    UClass* Class = FCSClassUtilities::GetFirstNonBlueprintClass(Object->GetClass());
    
    // 2. 获取类型定义和类型句柄
    TSharedPtr<FCSManagedTypeDefinition> ManagedTypeDefinition = FindOrAddManagedTypeDefinition(Class);
    TSharedPtr<FGCHandle> TypeGCHandle = ManagedTypeDefinition->GetTypeGCHandle();
    
    return CreateManagedObjectFromNative(Object, TypeGCHandle);
}

TSharedPtr<FGCHandle> UCSManagedAssembly::CreateManagedObjectFromNative(
    const UObject* Object, 
    const TSharedPtr<FGCHandle>& TypeGCHandle)
{
    // 3. 调用 C# 创建托管对象
    TCHAR* Error = nullptr;
    FGCHandle NewObjectHandle = FCSManagedCallbacks::ManagedCallbacks.CreateNewManagedObject(
        Object,                    // 原生对象指针
        TypeGCHandle->GetPointer(), // 类型句柄
        &Error                     // 错误输出
    );
    NewObjectHandle.Type = GCHandleType::StrongHandle;

    if (NewObjectHandle.IsNull())
    {
        UE_LOGFMT(LogUnrealSharp, Fatal, "Failed to create managed counterpart for {0}:\n{1}", 
            *Object->GetName(), Error);
    }

    // 4. 记录到分配列表和全局映射表
    TSharedPtr<FGCHandle> Handle = MakeShared<FGCHandle>(NewObjectHandle);
    AllocatedGCHandles.Add(Handle);

    uint32 ObjectID = Object->GetUniqueID();
    UCSManager::Get().ManagedObjectHandles.AddByHash(ObjectID, ObjectID, Handle);
    
    return Handle;
}
```

### 5.3 C# 侧的对象创建

```csharp
// UnrealSharpObject.cs:12-31
public class UnrealSharpObject : IDisposable
{
    internal static unsafe IntPtr Create(Type typeToCreate, IntPtr nativeObjectPtr)
    {
        // 1. 获取默认构造函数
        const BindingFlags bindingFlags = BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance;
        ConstructorInfo? ctor = typeToCreate.GetConstructor(bindingFlags, Type.EmptyTypes);
            
        if (ctor == null)
        {
            LogUnrealSharpCore.LogError("Failed to find default constructor for type: " + typeToCreate.FullName);
            return IntPtr.Zero;
        }
            
        // 2. 获取构造函数的函数指针
        delegate*<object, void> ctorPtr = (delegate*<object, void>)ctor.MethodHandle.GetFunctionPointer();
            
        // 3. 创建未初始化的对象实例
        UnrealSharpObject createdObject = (UnrealSharpObject)RuntimeHelpers.GetUninitializedObject(typeToCreate);
        
        // 4. 设置 NativeObject 指针
        createdObject.NativeObject = nativeObjectPtr;
            
        // 5. 调用构造函数
        ctorPtr(createdObject);
            
        // 6. 创建并返回 GCHandle
        return GCHandle.ToIntPtr(GCHandleUtilities.AllocateStrongPointer(createdObject, typeToCreate.Assembly));
    }
    
    public IntPtr NativeObject { get; private set; }
}
```

```csharp
// UnmanagedCallbacks.cs:12-37
[UnmanagedCallersOnly]
public static unsafe IntPtr CreateNewManagedObject(IntPtr nativeObject, IntPtr typeHandlePtr, char** error)
{
    try
    {
        if (nativeObject == IntPtr.Zero)
        {
            throw new ArgumentNullException(nameof(nativeObject));
        }
        
        Type? type = GCHandleUtilities.GetObjectFromHandlePtr<Type>(typeHandlePtr);
        if (type == null)
        {
            throw new InvalidOperationException("The provided type handle does not point to a valid type.");
        }

        return UnrealSharpObject.Create(type, nativeObject);
    }
    catch (Exception ex)
    {
        LogUnrealSharpCore.LogError($"Failed to create new managed object: {ex.Message}");
        *error = (char*)Marshal.StringToHGlobalUni(ex.ToString());
    }

    return IntPtr.Zero;
}
```

### 5.4 完整流程图

```
C++ 需要 UObject 的 C# 包装器
        │
        ▼
┌───────────────────────────────────┐
│ UCSManager::FindManagedObject()   │
└───────────────────────────────────┘
        │
        ├─ 已存在 → 返回缓存的 FGCHandle
        │
        ▼ 不存在
┌───────────────────────────────────┐
│ 找到 UObject 所属的程序集         │
│ (UCSManagedAssembly)              │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ CreateManagedObjectFromNative()   │
│ - 获取类型句柄                     │
│ - 调用 C# 创建函数                 │
└───────────────────────────────────┘
        │
        ▼ 跨越 C++/C# 边界
┌───────────────────────────────────┐
│ C#: UnmanagedCallbacks.           │
│     CreateNewManagedObject()      │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ C#: UnrealSharpObject.Create()    │
│ - RuntimeHelpers.GetUninitialized│
│ - 设置 NativeObject 指针          │
│ - 调用构造函数                     │
│ - 创建 GCHandle                    │
└───────────────────────────────────┘
        │
        ▼ 返回 GCHandle 指针
┌───────────────────────────────────┐
│ C++: 记录到 ManagedObjectHandles  │
│ 返回 FGCHandle                     │
└───────────────────────────────────┘
```

---

## 六、热重载与内存管理

### 6.1 AssemblyLoadContext 卸载

```csharp
// GCHandleUtilities.cs:13-17
[MethodImpl(MethodImplOptions.NoInlining)]
private static void OnAlcUnloading(AssemblyLoadContext alc)
{
    // 移除该 ALC 的所有强引用
    // 允许 GC 回收这些对象
    StrongRefsByAssembly.TryRemove(alc, out _);
}
```

### 6.2 程序集卸载时的清理

```cpp
// CSManagedAssembly.cpp:73-100
bool UCSManagedAssembly::UnloadManagedAssembly()
{
    if (!bIsCollectible)
    {
        UE_LOGFMT(LogUnrealSharp, Warning, 
            "Assembly {0} is not collectible and will not be unloaded.", 
            *AssemblyName.ToString());
        return true;
    }
    
    FGCHandleIntPtr AssemblyHandle = AssemblyGCHandle->GetHandle();
    
    // 释放所有分配的 GCHandle
    for (TSharedPtr<FGCHandle>& Handle : AllocatedGCHandles)
    {
        Handle->Dispose(AssemblyHandle);
    }

    ManagedTypeGCHandles.Reset();
    AllocatedGCHandles.Reset();

    // 释放程序集句柄
    AssemblyGCHandle->Dispose(AssemblyHandle->GetHandle());
    AssemblyGCHandle.Reset();

    UCSManager::Get().OnManagedAssemblyUnloadedEvent().Broadcast(this);
    return true;
}
```

---

## 七、接口包装器的特殊处理

当一个 UObject 实现了 C# 定义的接口时，需要创建接口包装器：

```cpp
// CSManager.h:217-218
// The primary key is the unique ID of the UObject.
// The second key is the unique ID of the interface class.
TMap<uint32, TMap<uint32, TSharedPtr<FGCHandle>>> ManagedInterfaceWrappers;
```

**为什么需要接口包装器？**

```
C# 定义的接口：IInteractable
C++ 类：ADoor : AActor, IInteractable

当 C# 代码需要访问 IInteractable 接口时：
1. 不能直接使用 ADoor 的 C# 包装器（类型不匹配）
2. 需要创建专门的接口包装器
3. 接口包装器调用正确的方法实现
```

```cpp
// CSManagedAssembly.cpp:228-260
TSharedPtr<FGCHandle> UCSManagedAssembly::GetOrCreateManagedInterface(
    UObject* Object, 
    UClass* InterfaceClass)
{
    uint32 ObjectID = Object->GetUniqueID();
    TMap<uint32, TSharedPtr<FGCHandle>>& TypeMap = 
        UCSManager::Get().ManagedInterfaceWrappers.FindOrAddByHash(ObjectID, ObjectID);
    
    uint32 TypeId = InterfaceClass->GetUniqueID();
    if (TSharedPtr<FGCHandle>* Existing = TypeMap.FindByHash(TypeId, TypeId))
    {
        return *Existing;
    }

    // 获取对象的 GCHandle
    TSharedPtr<FGCHandle>* ObjectHandle = 
        UCSManager::Get().ManagedObjectHandles.FindByHash(ObjectID, ObjectID);
    if (ObjectHandle == nullptr)
    {
        // 需要先创建对象
        UCSManager::Get().FindManagedObject(Object);
        ObjectHandle = UCSManager::Get().ManagedObjectHandles.FindByHash(ObjectID, ObjectID);
    }

    // 创建接口包装器
    FGCHandle InterfaceWrapper = FCSManagedCallbacks::ManagedCallbacks.CreateNewManagedObjectWrapper(
        ObjectHandle->Get()->GetPointer(),
        TypeHandle->GetPointer()
    );
    
    // 存储到二级映射表
    TSharedPtr<FGCHandle> Wrapper = MakeShared<FGCHandle>(InterfaceWrapper);
    TypeMap.AddByHash(TypeId, TypeId, Wrapper);
    
    return Wrapper;
}
```

---

## 八、总结

### 8.1 核心设计要点

| 要点 | 实现方式 |
|-----|---------|
| **跨语言引用** | GCHandle 机制 |
| **生命周期同步** | FUObjectDeleteListener 监听 |
| **对象缓存** | ManagedObjectHandles 映射表 |
| **热重载支持** | 按 ALC 分组的强引用 |
| **RAII 封装** | FScopedGCHandle / FSharedGCHandle |

### 8.2 与你的练习代码的演进

| 方面 | 你的练习代码 | UnrealSharp |
|-----|-------------|-------------|
| 对象传递 | 仅基本类型 | 完整对象引用 |
| 生命周期 | 无管理 | 与 UObject 同步 |
| 内存泄漏风险 | 高 | 低（自动清理） |
| 热重载支持 | 无 | 完整支持 |

### 8.3 下一篇预告

本章我们了解了对象如何跨越 C++/C# 边界。下一章将深入剖析托管回调系统，了解 C++ 如何调用 C# 方法，以及 C# 如何调用 C++ 函数。

---

## 参考资料

- [GCHandle Struct (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.gchandle)
- [FUObjectArray::FUObjectDeleteListener (UE Docs)](https://docs.unrealengine.com/)
- [AssemblyLoadContext (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/core/dependency-loading/understanding-assemblyloadcontext)

---

*下一篇：双向通信的基石 - 托管回调系统深度解析*
