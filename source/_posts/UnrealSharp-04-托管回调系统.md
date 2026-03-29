---
title: 双向通信的基石 - 托管回调系统深度解析
date: 2026-03-29 23:00:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, C#, UnmanagedCallersOnly, 互操作]
series: UnrealSharp 插件技术深度解析
series_number: 5
---

# 双向通信的基石 - 托管回调系统深度解析

> **作者**：GLM-5.0

前两章我们了解了运行时初始化和对象生命周期管理。现在，一个关键问题浮出水面：**C++ 和 C# 之间如何进行函数调用？** 本章将深入剖析 UnrealSharp 的双向通信机制。

## 一、从 UnmanagedCallersOnly 说起

### 1.1 你的练习代码中的基础模式

回顾你的 CoreClrDemo 练习代码：

```cpp
// C++ 调用 C# 函数的基础模式
typedef void (*PrintMessageFunc)(const char* message);

// 通过 hostfxr 获取函数指针
load_assembly_and_get_function_pointer(
    assemblyPath,
    "ManagedDemo.ManagedClass, ManagedDemo",
    "PrintMessage",
    UNMANAGEDCALLERSONLY_METHOD,
    nullptr,
    &PrintMessageFunc
);

// 调用
PrintMessageFunc("Hello from C++!");
```

```csharp
// C# 侧
[UnmanagedCallersOnly]
public static void PrintMessage(string message)
{
    Console.WriteLine($"C# received: {message}");
}
```

**局限性**：
- 需要为每个函数单独获取函数指针
- 无法动态发现和调用方法
- 不支持面向对象的方法调用

### 1.2 UnrealSharp 的演进方案

UnrealSharp 构建了一套**完整的回调注册和发现系统**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    初始化时的回调注册                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  C++ 入口点                                                     │
│  InitializeDotNetRuntime()                                      │
│         │                                                       │
│         ▼                                                       │
│  调用 C# Main.InitializeUnrealSharp()                          │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  C# 注册三大回调系统:                                     │   │
│  │  1. PluginsCallbacks - 插件加载/卸载                     │   │
│  │  2. ManagedCallbacks - 核心操作回调                      │   │
│  │  3. NativeBinds - 原生函数绑定                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│         │                                                       │
│         ▼                                                       │
│  C++ 缓存回调函数指针，后续直接调用                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、三大回调系统概览

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                          C++ 侧                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  FCSManagedCallbacks::ManagedCallbacks                   │   │
│  │  - CreateNewManagedObject                                │   │
│  │  - InvokeManagedMethod                                   │   │
│  │  - LookupManagedMethod                                   │   │
│  │  - LookupManagedType                                     │   │
│  │  - Dispose / FreeHandle                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ▲                                  │
│                              │ 函数指针                         │
│                              │                                  │
└──────────────────────────────│──────────────────────────────────┘
                               │
┌──────────────────────────────│──────────────────────────────────┐
│                          C# 侧                                 │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ManagedCallbacks 结构体                                  │   │
│  │  [StructLayout(Sequential)]                              │   │
│  │  {                                                       │   │
│  │    delegate* unmanaged CreateManagedObject;             │   │
│  │    delegate* unmanaged InvokeManagedMethod;             │   │
│  │    ...                                                   │   │
│  │  }                                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  UnmanagedCallbacks 静态类                                │   │
│  │  [UnmanagedCallersOnly] 方法实现                         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 回调分类

| 回调系统 | 方向 | 用途 |
|---------|------|------|
| **PluginsCallbacks** | C++ → C# | 加载/卸载程序集 |
| **ManagedCallbacks** | C++ → C# | 创建对象、调用方法、查找类型 |
| **NativeBinds** | C# → C++ | 调用 UE 原生函数 |

---

## 三、ManagedCallbacks：C++ 调用 C# 的核心

### 3.1 C# 侧定义

```csharp
// ManagedCallbacks.cs
[StructLayout(LayoutKind.Sequential)]
public unsafe struct ManagedCallbacks
{   
    // 创建托管对象
    public delegate* unmanaged<IntPtr, IntPtr, char**, IntPtr> CreateManagedObject;
    
    // 创建接口包装器
    public delegate* unmanaged<IntPtr, IntPtr, IntPtr> CreateNewManagedObjectWrapper;
    
    // 调用托管方法
    public delegate* unmanaged<IntPtr, IntPtr, IntPtr, IntPtr, IntPtr, int> InvokeManagedMethod;
    
    // 调用委托
    public delegate* unmanaged<IntPtr, void> InvokeDelegate;
    
    // 查找方法
    public delegate* unmanaged<IntPtr, nint, IntPtr> LookupManagedMethod;
    
    // 查找类型
    public delegate* unmanaged<IntPtr, nint, IntPtr> LookupManagedType;
    
    // 初始化结构体
    public delegate* unmanaged<IntPtr, IntPtr, void> InitializeStruct;
    
    // 释放资源
    public delegate* unmanaged<IntPtr, IntPtr, void> Dispose;
    public delegate* unmanaged<IntPtr, void> FreeHandle;

    public static void Initialize(IntPtr outManagedCallbacks)
    {
        // 将函数指针填充到结构体
        *(ManagedCallbacks*)outManagedCallbacks = new ManagedCallbacks
        {
            CreateManagedObject = &UnmanagedCallbacks.CreateNewManagedObject,
            CreateNewManagedObjectWrapper = &UnmanagedCallbacks.CreateNewManagedObjectWrapper,
            InvokeManagedMethod = &UnmanagedCallbacks.InvokeManagedMethod,
            InvokeDelegate = &UnmanagedCallbacks.InvokeDelegate,
            LookupManagedMethod = &UnmanagedCallbacks.LookupManagedMethod,
            LookupManagedType = &UnmanagedCallbacks.LookupManagedType,
            InitializeStruct = &UnmanagedCallbacks.InitializeStruct,
            Dispose = &UnmanagedCallbacks.Dispose,
            FreeHandle = &UnmanagedCallbacks.FreeHandle,
        };
    }
}
```

### 3.2 C++ 侧定义

```cpp
// CSManagedCallbacksCache.h
class FCSManagedCallbacks
{
public:
    struct FManagedCallbacks
    {
        // 函数指针类型定义
        using ManagedCallbacks_CreateNewManagedObject = FGCHandleIntPtr(__stdcall*)(const void*, void*, TCHAR**);
        using ManagedCallbacks_InvokeManagedMethod = int(__stdcall*)(void*, void*, void*, void*, void*);
        using ManagedCallbacks_LookupMethod = uint8*(__stdcall*)(void*, const TCHAR*);
        using ManagedCallbacks_LookupType = uint8*(__stdcall*)(uint8*, const TCHAR*);
        using ManagedCallbacks_Dispose = void(__stdcall*)(FGCHandleIntPtr, FGCHandleIntPtr);
        using ManagedCallbacks_FreeHandle = void(__stdcall*)(FGCHandleIntPtr);
        
        // 函数指针成员
        ManagedCallbacks_CreateNewManagedObject CreateNewManagedObject;
        ManagedCallbacks_CreateNewManagedObjectWrapper CreateNewManagedObjectWrapper;
        ManagedCallbacks_InvokeManagedMethod InvokeManagedMethod;
        ManagedCallbacks_InvokeDelegate InvokeDelegate;
        ManagedCallbacks_LookupMethod LookupManagedMethod;
        ManagedCallbacks_LookupType LookupManagedType;
        ManagedCallbacks_InitializeStructure InitializeStructure;
        
    private:
        friend FGCHandle;
        ManagedCallbacks_Dispose Dispose;
        ManagedCallbacks_FreeHandle FreeHandle;
    };
    
    static inline FManagedCallbacks ManagedCallbacks;
};
```

### 3.3 关键回调函数详解

#### CreateNewManagedObject：创建托管对象

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
        
        // 从类型句柄获取 Type 对象
        Type? type = GCHandleUtilities.GetObjectFromHandlePtr<Type>(typeHandlePtr);
        if (type == null)
        {
            throw new InvalidOperationException("The provided type handle does not point to a valid type.");
        }

        // 创建托管对象
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

**调用流程**：
```
C++: UCSManager::FindManagedObject()
         │
         ▼
C++: CreateManagedObjectFromNative()
         │
         ▼
C++: FCSManagedCallbacks::ManagedCallbacks.CreateNewManagedObject()
         │
         ▼ 跨越边界
C#: UnmanagedCallbacks.CreateNewManagedObject()
         │
         ▼
C#: UnrealSharpObject.Create()
         │
         ├─ RuntimeHelpers.GetUninitializedObject()
         ├─ 设置 NativeObject 指针
         ├─ 调用构造函数
         └─ GCHandleUtilities.AllocateStrongPointer()
         │
         ▼
返回 GCHandle 指针
```

#### InvokeManagedMethod：调用托管方法

```csharp
// UnmanagedCallbacks.cs:245-274
[UnmanagedCallersOnly]
public static unsafe int InvokeManagedMethod(
    IntPtr managedObjectHandle,   // 托管对象的 GCHandle
    IntPtr methodHandlePtr,       // 方法的 GCHandle
    IntPtr argumentsBuffer,       // 参数缓冲区
    IntPtr returnValueBuffer,     // 返回值缓冲区
    IntPtr exceptionTextBuffer)   // 异常信息缓冲区
{
    try
    {
#if UNREALSHARP_MONO
        // Mono 模式：LookupManagedMethod 存储的是 MethodInfo
        MethodInfo? methodInfo = GCHandleUtilities.GetObjectFromHandlePtrFast<MethodInfo>(methodHandlePtr);
        object managedObject = GCHandleUtilities.GetObjectFromHandlePtrFast<object>(managedObjectHandle)!;
        methodInfo!.Invoke(managedObject, new object[] { argumentsBuffer, returnValueBuffer });
#else
        // CoreCLR 模式：LookupManagedMethod 存储的是函数指针
        IntPtr methodHandle = GCHandleUtilities.GetObjectFromHandlePtrFast<IntPtr>(methodHandlePtr)!;
        object managedObject = GCHandleUtilities.GetObjectFromHandlePtrFast<object>(managedObjectHandle)!;
        delegate*<object, IntPtr, IntPtr, void> methodPtr = 
            (delegate*<object, IntPtr, IntPtr, void>)methodHandle;
        methodPtr(managedObject, argumentsBuffer, returnValueBuffer);
#endif
        return 0;  // 成功
    }
    catch (Exception ex)
    {
        StringMarshaller.ToNative(exceptionTextBuffer, 0, ex.ToString());
        return 1;  // 失败
    }
}
```

**CoreCLR vs Mono 的关键差异**：

| 方面 | CoreCLR | Mono |
|-----|---------|------|
| 方法查找 | `MethodHandle.GetFunctionPointer()` | 存储 MethodInfo 对象 |
| 方法调用 | 直接函数指针调用 | `MethodInfo.Invoke()` |
| 性能 | 更快 | 较慢（反射开销） |
| 原因 | JIT 可生成直接调用 | AOT+解释器模式限制 |

#### LookupManagedMethod：查找方法

```csharp
// UnmanagedCallbacks.cs:85-158
[UnmanagedCallersOnly]
public static unsafe IntPtr LookupManagedMethod(IntPtr typeHandlePtr, nint methodNamePtr)
{
    // CRITICAL: 先复制字符串！
    // 在 Mono INTERP 模式下，其他 native 调用可能导致 methodNamePtr 指向的内存被重分配
    string methodNameString = new string((char*)methodNamePtr);
    
    try
    {
        Type? type = GCHandleUtilities.GetObjectFromHandlePtr<Type>(typeHandlePtr);
        if (type == null) throw new Exception("Invalid type handle");

        Type? currentType = type;
        while (currentType != null)
        {
#if UNREALSHARP_MONO
            // Mono 模式：使用 TypeInfo.DeclaredMethods 绕过反射 bug
            MethodInfo? method = null;
            foreach (var candidate in currentType.GetTypeInfo().DeclaredMethods)
            {
                if (candidate.Name == methodNameString)
                {
                    method = candidate;
                    break;
                }
            }
#else
            // CoreCLR 模式：标准反射
            MethodInfo? method = currentType.GetMethod(methodNameString, 
                BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance | BindingFlags.Static);
#endif

            if (method != null)
            {
#if UNREALSHARP_MONO
                // Mono：存储 MethodInfo
                GCHandle methodHandle = GCHandleUtilities.AllocateStrongPointer(method, type.Assembly);
#else
                // CoreCLR：存储函数指针
                IntPtr functionPtr = method.MethodHandle.GetFunctionPointer();
                GCHandle methodHandle = GCHandleUtilities.AllocateStrongPointer(functionPtr, type.Assembly);
#endif
                return GCHandle.ToIntPtr(methodHandle);
            }

            currentType = currentType.BaseType;  // 向上查找基类
        }

        return IntPtr.Zero;
    }
    catch (Exception e)
    {
        LogUnrealSharpCore.LogError($"Exception while looking up managed method: {e.Message}");
        return IntPtr.Zero;
    }
}
```

---

## 四、NativeBinds：C# 调用 C++ 的桥梁

### 4.1 UNREALSHARP_FUNCTION 宏机制

```cpp
// CSBindsManager.h
#define UNREALSHARP_FUNCTION()  // 宏本身是空的！
```

**宏是空的，那函数是如何被注册的？**

答案在于 **UHT 插件**的代码生成：

```
┌─────────────────────────────────────────────────────────────────┐
│                    编译时代码生成流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. C++ 头文件中的标记                                          │
│     // UObjectExporter.h                                        │
│     UNREALSHARP_FUNCTION()                                      │
│     static void* CreateNewObject(UObject* Outer, UClass* Class);│
│                                                                 │
│         │                                                       │
│         ▼                                                       │
│                                                                 │
│  2. UHT 解析 (NativeBindExporter.cs)                           │
│     - 识别 UNREALSHARP_FUNCTION 宏                              │
│     - 提取函数名和签名                                          │
│     - 计算参数大小                                              │
│                                                                 │
│         │                                                       │
│         ▼                                                       │
│                                                                 │
│  3. 生成 .unrealsharp.cpp 文件                                  │
│     struct Z_Construct_UUObjectExporter_UnrealSharp_Binds...    │
│     {                                                           │
│         static const FCSExportedFunction UnrealSharpBind_CreateNewObject;
│     };                                                          │
│     const FCSExportedFunction ...::UnrealSharpBind_CreateNewObject│
│         = FCSExportedFunction("UUObjectExporter",               │
│                              "CreateNewObject",                 │
│                              (void*)&UUObjectExporter::CreateNewObject,│
│                              GetFunctionSize(...));             │
│                                                                 │
│         │                                                       │
│         ▼                                                       │
│                                                                 │
│  4. 静态初始化                                                  │
│     程序启动时，静态变量构造，自动注册到 FCSBindsManager         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 NativeBindExporter 详解

```csharp
// NativeBindExporter.cs:50-102
[UhtKeyword(Extends = UhtTableNames.Default, Keyword = "UNREALSHARP_FUNCTION")]
private static UhtParseResult UNREALSHARP_FUNCTIONKeyword(UhtParsingScope topScope, ...)
{
    return ParseUnrealSharpBind(topScope, actionScope, ref token);
}

private static UhtParseResult ParseUnrealSharpBind(UhtParsingScope topScope, ...)
{
    UhtHeaderFile headerFile = topScope.ScopeType.HeaderFile;

    // 启用 token 录制
    topScope.TokenReader.EnableRecording();
    topScope.TokenReader
        .Require('(')
        .Require(')')
        .Require("static")  // 必须是静态函数
        .ConsumeUntil('(');

    // 提取方法名
    int recordedTokensCount = topScope.TokenReader.RecordedTokens.Count;
    string methodName = topScope.TokenReader.RecordedTokens[recordedTokensCount - 2].Value.ToString();
    topScope.TokenReader.DisableRecording();
    
    NativeBindMethod methodInfo = new(methodName);
    
    // 存储到字典，等待导出
    // ...
    
    return UhtParseResult.Handled;
}
```

生成的代码：

```csharp
// NativeBindExporter.cs:126-138
foreach (NativeBindMethod method in methods)
{
    string functionReference = $"{topType.SourceName}::{method.MethodName}";
    builder.AppendLine($"const FCSExportedFunction {typeName}::UnrealSharpBind_{method.MethodName}");
    builder.Append($" = FCSExportedFunction(\"{topType.EngineName}\", " +
                   $"\"{method.MethodName}\", " +
                   $"(void*)&{functionReference}, " +
                   $"GetFunctionSize({functionReference}));");
}
```

### 4.3 FCSBindsManager：函数注册与查找

```cpp
// CSBindsManager.cpp
void FCSBindsManager::RegisterExportedFunction(const FName& ClassName, const FCSExportedFunction& ExportedFunction)
{
    FCSBindsManager* Instance = Get();
    TArray<FCSExportedFunction>& ExportedFunctions = Instance->ExportedFunctions.FindOrAdd(ClassName);
    ExportedFunctions.Add(ExportedFunction);
}

void* FCSBindsManager::GetBoundFunction(const TCHAR* InOuterName, const TCHAR* InFunctionName, int32 InParametersSize)
{
    FName ManagedOuterName = FName(InOuterName);
    FName ManagedFunctionName = FName(InFunctionName);
    
    TArray<FCSExportedFunction>* ExportedFunctions = Instance->ExportedFunctions.Find(ManagedOuterName);
    if (!ExportedFunctions) return nullptr;

    for (FCSExportedFunction& NativeFunction : *ExportedFunctions)
    {
        if (NativeFunction.Name != ManagedFunctionName) continue;
            
        // 参数大小校验（防止签名不匹配）
        if (NativeFunction.ParameterSize != InParametersSize)
        {
            UE_LOGFMT(LogUnrealSharpBinds, Error, 
                "Function size mismatch for {0}.{1} (expected {2}, got {3})",
                *ManagedOuterName.ToString(), *ManagedFunctionName.ToString(), 
                NativeFunction.ParameterSize, InParametersSize);
            break;
        }
            
        return NativeFunction.FunctionPointer;
    }
    
    return nullptr;
}
```

### 4.4 C# 侧的 Source Generator

```csharp
// NativeCallbacksWrapperGenerator.cs
// 扫描带有 [NativeCallbacks] 特性的类，生成包装代码

[Generator]
public class NativeCallbacksWrapperGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var classDeclarations = context.SyntaxProvider.CreateSyntaxProvider(
            static (syntaxNode, _) => syntaxNode is ClassDeclarationSyntax cds && cds.AttributeLists.Count > 0,
            static (syntaxContext, _) => GetClassInfoOrNull(syntaxContext));

        context.RegisterSourceOutput(classAndCompilation, (spc, pair) =>
        {
            GenerateForClass(spc, compilation, classInfo);
        });
    }
}
```

**生成的代码示例**：

```csharp
// 为 UUObjectExporter 类生成的代码
public static unsafe partial class UUObjectExporter
{
    static UUObjectExporter()
    {
        // 计算参数大小
        int CreateNewObjectTotalSize = sizeof(IntPtr) + sizeof(IntPtr) + sizeof(IntPtr);
        
        // 获取函数指针
        IntPtr CreateNewObjectFuncPtr = UnrealSharp.Binds.NativeBinds.TryGetBoundFunction(
            "UUObjectExporter", "CreateNewObject", CreateNewObjectTotalSize);
        
        // 转换为函数指针类型
        CreateNewObject = (delegate* unmanaged<IntPtr, IntPtr, IntPtr, IntPtr>)CreateNewObjectFuncPtr;
    }
    
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static IntPtr CallCreateNewObject(IntPtr outer, IntPtr clazz, IntPtr template)
    {
        return CreateNewObject(outer, clazz, template);
    }
    
    private static delegate* unmanaged<IntPtr, IntPtr, IntPtr, IntPtr> CreateNewObject;
}
```

### 4.5 NativeBinds 工具类

```csharp
// BindsManager.cs
public static class NativeBinds
{
    private static unsafe delegate* unmanaged[Cdecl]<char*, char*, int, IntPtr> _getBoundFunction = null;

    public static unsafe void Initialize(IntPtr bindsCallbacks)
    {
        // bindsCallbacks 就是 FCSBindsManager::GetBoundFunction 的函数指针
        _getBoundFunction = (delegate* unmanaged[Cdecl]<char*, char*, int, IntPtr>)bindsCallbacks;
    }

    public static unsafe IntPtr TryGetBoundFunction(string outerName, string functionName, int functionSize)
    {
        if (_getBoundFunction == null)
        {
            throw new Exception("NativeBinds not initialized");
        }

        IntPtr functionPtr;
        fixed (char* outerNamePtr = outerName)
        fixed (char* functionNamePtr = functionName)
        {
            functionPtr = _getBoundFunction(outerNamePtr, functionNamePtr, functionSize);
        }

        if (functionPtr == IntPtr.Zero)
        {
            throw new Exception($"Failed to find bound function {functionName} in {outerName}");
        }

        return functionPtr;
    }
}
```

---

## 五、PluginsCallbacks：程序集管理

```csharp
// PluginsCallbacks.cs
[StructLayout(LayoutKind.Sequential)]
public unsafe struct PluginsCallbacks
{
    public delegate* unmanaged<char*, NativeBool, IntPtr> LoadPlugin;
    public delegate* unmanaged<char*, NativeBool> UnloadPlugin;
    
    [UnmanagedCallersOnly]
    private static nint ManagedLoadPlugin(char* assemblyPath, NativeBool isCollectible)
    {
        Assembly? newPlugin = PluginLoader.LoadPlugin(
            new string(assemblyPath), 
            isCollectible.ToManagedBool());

        if (newPlugin == null) return IntPtr.Zero;

        return GCHandle.ToIntPtr(GCHandleUtilities.AllocateStrongPointer(newPlugin, newPlugin));
    }

    [UnmanagedCallersOnly]
    private static NativeBool ManagedUnloadPlugin(char* assemblyPath)
    {
        string assemblyPathStr = new(assemblyPath);
        return PluginLoader.UnloadPlugin(assemblyPathStr).ToNativeBool();
    }

    public static void Initialize(PluginsCallbacks* outCallbacks)
    {
        *outCallbacks = new PluginsCallbacks
        {
            LoadPlugin = &ManagedLoadPlugin,
            UnloadPlugin = &ManagedUnloadPlugin,
        };
    }
}
```

---

## 六、调用约定与参数封送

### 6.1 Calling Convention 选择

UnrealSharp 使用 `__stdcall` 作为回调函数的调用约定：

```cpp
// CSManagedCallbacksCache.h
using ManagedCallbacks_CreateNewManagedObject = FGCHandleIntPtr(__stdcall*)(const void*, void*, TCHAR**);
```

C# 侧使用 `unmanaged` 约定：

```csharp
public delegate* unmanaged<IntPtr, IntPtr, char**, IntPtr> CreateManagedObject;
```

### 6.2 参数封送原则

**Blittable 类型优先**：

| 类型 | C# | C++ | 封送方式 |
|-----|----|----|---------|
| 整数 | int, long | int32, int64 | 直接拷贝 |
| 浮点 | float, double | float, double | 直接拷贝 |
| 指针 | IntPtr | void* | 直接传递 |
| 布尔 | bool | bool | 直接拷贝 |
| 字符串 | string | TCHAR* | 需要转换 |
| 对象 | object | UObject* | 通过 GCHandle |

**缓冲区模式**：

对于复杂参数，UnrealSharp 使用**预分配缓冲区**模式：

```csharp
// 方法调用时的参数传递
public static unsafe int InvokeManagedMethod(
    IntPtr managedObjectHandle,
    IntPtr methodHandlePtr,
    IntPtr argumentsBuffer,    // 所有参数打包到一个缓冲区
    IntPtr returnValueBuffer,  // 返回值缓冲区
    IntPtr exceptionTextBuffer)
```

这避免了频繁的内存分配，提高了性能。

---

## 七、完整调用链分析

### 7.1 C# 调用 C++ 函数

```
C# 代码: UUObjectExporter.CallCreateNewObject(outer, clazz, template)
    │
    ▼
生成的静态构造函数（仅首次）
    │
    ├─ 计算 sizeof(IntPtr) * 3 = 24 字节
    │
    └─ NativeBinds.TryGetBoundFunction("UUObjectExporter", "CreateNewObject", 24)
           │
           ▼
       C++: FCSBindsManager::GetBoundFunction()
           │
           └─ 返回 UUObjectExporter::CreateNewObject 的函数指针
    │
    ▼
C#: CreateNewObject(outer, clazz, template)
    │
    ▼ 直接函数指针调用
C++: UUObjectExporter::CreateNewObject()
    │
    ├─ NewObject<UObject>(...)
    │
    └─ UCSManager::FindManagedObject()
    │
    ▼
返回 C# 托管对象的 GCHandle
```

### 7.2 C++ 调用 C# 方法

```
C++: 蓝图调用 C# 方法
    │
    ▼
FCSManagedCallbacks::ManagedCallbacks.InvokeManagedMethod()
    │
    ▼ 跨越边界
C#: UnmanagedCallbacks.InvokeManagedMethod()
    │
    ├─ 从 GCHandle 获取托管对象
    │
    ├─ 从方法句柄获取函数指针（CoreCLR）或 MethodInfo（Mono）
    │
    └─ 调用实际的 C# 方法
    │
    ▼
C# 方法执行
    │
    ▼
返回值写入 returnValueBuffer
```

---

## 八、总结

### 8.1 设计亮点

| 设计点 | 实现方式 | 优势 |
|-------|---------|------|
| **双向通信** | ManagedCallbacks + NativeBinds | 完整的双向调用能力 |
| **类型安全** | 参数大小校验 | 编译时/运行时双重检查 |
| **性能优化** | 缓冲区模式 + 函数指针缓存 | 减少内存分配和查找开销 |
| **平台适配** | CoreCLR/Mono 分支处理 | 兼顾性能和兼容性 |

### 8.2 与你的练习代码的演进

| 方面 | 你的练习代码 | UnrealSharp |
|-----|-------------|-------------|
| 函数发现 | 手动指定 | 自动注册和查找 |
| 面向对象 | 不支持 | 完整的方法调用支持 |
| 异常处理 | 无 | 跨边界异常传播 |
| 类型安全 | 无 | 参数大小校验 |

### 8.3 下一篇预告

本章我们了解了 C++ 和 C# 之间的双向通信机制。下一篇将深入剖析 Glue 代码生成系统，了解 UHT 如何自动生成类型绑定代码。

---

## 参考资料

- [UnmanagedCallersOnly Attribute (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.unmanagedcallersonlyattribute)
- [Function Pointers (C# 9.0)](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/unsafe-code#function-pointers)
- [Unreal Header Tool (UE Docs)](https://docs.unrealengine.com/)

---

*下一篇：自动化绑定的艺术 - Glue代码生成系统原理*
