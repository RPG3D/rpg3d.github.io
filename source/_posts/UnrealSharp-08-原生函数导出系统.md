---
title: UnrealSharp 插件技术深度解析 - 原生函数导出系统
date: 2026-03-30 13:00:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, C#, P/Invoke, UNREALSHARP_FUNCTION]
series: UnrealSharp 插件技术深度解析
series_number: 8
---

# UNREALSHARP_FUNCTION 宏的秘密 - 原生函数导出机制

> **技术深度**：⭐⭐⭐⭐
> **前置知识**：P/Invoke、函数指针、调用约定

---

## 引言

在前面的文章中，我们分析了 C# 类型如何注册到 UE 反射系统。现在，我们将探讨另一个核心问题：**C# 如何调用 UE 的 C++ API？**

UnrealSharp 通过 `UNREALSHARP_FUNCTION` 宏实现了一套优雅的函数导出机制，本章将深入剖析其原理。

---

## 一、问题引入

### 1.1 C# 调用 C++ 的挑战

C# 调用 C++ API 通常有以下方式：

1. **P/Invoke**：通过 `DllImport` 声明外部函数
2. **C++/CLI**：使用托管 C++ 桥接
3. **COM 接口**：通过 COM 组件通信

UnrealSharp 选择了 **P/Invoke + 函数指针查找** 的混合方案：

```csharp
// C# 侧调用
UObjectExporter.CallGetName(nativeObjectPtr);
```

这个调用如何映射到 C++ 的函数？

### 1.2 设计目标

1. **零开销查找**：编译时确定函数地址，避免运行时字符串匹配
2. **类型安全**：参数类型在编译时检查
3. **简洁声明**：一个宏搞定导出
4. **Blittable 传递**：参数直接内存拷贝，无封送开销

---

## 二、UNREALSHARP_FUNCTION 宏解析

### 2.1 宏定义

```cpp
// Source/UnrealSharpBinds/Public/CSBindsManager.h

// Native bound function. If you want to bind a function to C#, use this macro.
// The managed delegate signature must match the native function signature + outer name,
// and all params need to be blittable.
#define UNREALSHARP_FUNCTION()
```

**看似简单，实则精妙**：这个宏本身是空的，它的作用是**标记**。

### 2.2 导出函数声明示例

```cpp
// Source/UnrealSharpCore/Public/Export/FStringExporter.h

UCLASS()
class UFStringExporter : public UObject
{
    GENERATED_BODY()
public:
    UNREALSHARP_FUNCTION()
    static void MarshalToNativeString(FString* NativeString, const char* ManagedString);
    
    UNREALSHARP_FUNCTION()
    static void MarshalToNativeStringView(FString* NativeString, 
                                          const UTF16CHAR* ManagedString, 
                                          int32 Length);
};
```

### 2.3 导出函数实现

```cpp
// Source/UnrealSharpCore/Public/Export/FStringExporter.h (inline)

UNREALSHARP_FUNCTION()
static void MarshalToNativeString(FString* NativeString, const char* ManagedString)
{
    if (!NativeString || !ManagedString)
    {
        *NativeString = FString();
        return;
    }
    
    // UTF-8 → TCHAR
    *NativeString = UTF8_TO_TCHAR(ManagedString);
}
```

---

## 三、函数注册机制

### 3.1 FCSExportedFunction 结构

```cpp
// Source/UnrealSharpBinds/Public/CSExportedFunction.h

struct UNREALSHARPBINDS_API FCSExportedFunction
{
    FName Name;              // 函数名
    void* FunctionPointer;   // 函数指针
    int32 ParameterSize;     // 参数总大小（字节）

    FCSExportedFunction(const FName& OuterName, 
                        const FName& Name, 
                        void* InFunctionPointer, 
                        int32 InParameterSize);
};
```

### 3.2 参数大小计算

```cpp
// 计算参数大小
template <typename T>
struct TArgSize
{
    constexpr static size_t Size = sizeof(T);
};

// 引用类型特殊处理：大小为指针大小
template <typename T>
struct TArgSize<T&>
{
    constexpr static size_t Size = sizeof(T*);
};

// 计算函数参数总大小
template <typename ReturnType, typename... Args>
constexpr size_t GetFunctionSize(ReturnType (*)(Args...))
{
    if constexpr (std::is_void_v<ReturnType>)
    {
        return (ArgSize<Args> + ... + 0);
    }
    else
    {
        return ArgSize<ReturnType> + (ArgSize<Args> + ... + 0);
    }
}
```

**设计意图**：参数大小用于验证 C# 端的调用签名是否匹配。

### 3.3 FCSBindsManager 绑定管理器

```cpp
// Source/UnrealSharpBinds/Public/CSBindsManager.h

class FCSBindsManager
{
public:
    // 注册导出函数
    static void RegisterExportedFunction(const FName& ClassName, 
                                         const FCSExportedFunction& ExportedFunction);
    
    // 获取绑定的函数指针
    static void* GetBoundFunction(const TCHAR* InOuterName, 
                                  const TCHAR* InFunctionName, 
                                  int32 InParametersSize);

private:
    static FCSBindsManager* Get();
    static FCSBindsManager* BindsManagerInstance;
    
    // 类名 → 函数列表映射
    TMap<FName, TArray<FCSExportedFunction>> ExportedFunctions;
};
```

---

## 四、Exporter 类体系

UnrealSharp 将 UE API 按功能域分组到不同的 Exporter 类中：

### 4.1 UObjectExporter - UObject 操作

```cpp
// Source/UnrealSharpCore/Public/Export/UObjectExporter.h (示意)

UCLASS()
class UUObjectExporter : public UObject
{
    GENERATED_BODY()
public:
    UNREALSHARP_FUNCTION()
    static FName GetObjectName(UObject* Object);
    
    UNREALSHARP_FUNCTION()
    static UClass* GetClass(UObject* Object);
    
    UNREALSHARP_FUNCTION()
    static UObject* GetOuter(UObject* Object);
    
    UNREALSHARP_FUNCTION()
    static bool IsValid(UObject* Object);
    
    UNREALSHARP_FUNCTION()
    static void MarkGarbageCollectionNeeded(UObject* Object);
};
```

### 4.2 UClassExporter - UClass 操作

```cpp
// Source/UnrealSharpCore/Public/Export/UClassExporter.h

UCLASS()
class UUClassExporter : public UObject
{
    GENERATED_BODY()
public:
    UNREALSHARP_FUNCTION()
    static UFunction* GetNativeFunctionFromClassAndName(const UClass* Class, 
                                                         const char* FunctionName);
    
    UNREALSHARP_FUNCTION()
    static void* GetDefaultFromInstance(UObject* Object);
    
    UNREALSHARP_FUNCTION()
    static bool IsChildOf(UClass* ChildClass, UClass* ParentClass);
};
```

### 4.3 FPropertyExporter - 属性操作

```cpp
// Source/UnrealSharpCore/Public/Export/FPropertyExporter.h

UCLASS()
class UFPropertyExporter : public UObject
{
    GENERATED_BODY()
public:
    UNREALSHARP_FUNCTION()
    static FProperty* GetNativePropertyFromName(UStruct* Struct, const char* PropertyName);
    
    UNREALSHARP_FUNCTION()
    static int32 GetPropertyOffset(FProperty* Property);
    
    UNREALSHARP_FUNCTION()
    static int32 GetSize(FProperty* Property);
    
    UNREALSHARP_FUNCTION()
    static void CopySingleValue(FProperty* Property, void* Dest, void* Src);
    
    UNREALSHARP_FUNCTION()
    static void InitializeValue(FProperty* Property, void* Value);
    
    UNREALSHARP_FUNCTION()
    static void DestroyValue(FProperty* Property, void* Value);
};
```

### 4.4 FStringExporter - 字符串操作

```cpp
// Source/UnrealSharpCore/Public/Export/FStringExporter.h

UCLASS()
class UFStringExporter : public UObject
{
    GENERATED_BODY()
public:
    // UTF-8 字符串转换
    UNREALSHARP_FUNCTION()
    static void MarshalToNativeString(FString* NativeString, const char* ManagedString);
    
    // UTF-16 字符串转换（推荐，避免编码问题）
    UNREALSHARP_FUNCTION()
    static void MarshalToNativeStringView(FString* NativeString, 
                                          const UTF16CHAR* ManagedString, 
                                          int32 Length);
};
```

### 4.5 FArrayPropertyExporter - 数组操作

```cpp
// Source/UnrealSharpCore/Public/Export/FArrayPropertyExporter.h

UCLASS()
class UFArrayPropertyExporter : public UObject
{
    GENERATED_BODY()
public:
    UNREALSHARP_FUNCTION()
    static void InitializeArray(FArrayProperty* ArrayProperty, 
                                const void* ScriptArray, int Length);
    
    UNREALSHARP_FUNCTION()
    static void AddToArray(FArrayProperty* ArrayProperty, const void* ScriptArray);
    
    UNREALSHARP_FUNCTION()
    static void RemoveFromArray(FArrayProperty* ArrayProperty, 
                                const void* ScriptArray, int index);
    
    UNREALSHARP_FUNCTION()
    static void ResizeArray(FArrayProperty* ArrayProperty, 
                            const void* ScriptArray, int Length);
};
```

### 4.6 FMulticastDelegatePropertyExporter - 委托操作

```cpp
// Source/UnrealSharpCore/Public/Export/FMulticastDelegatePropertyExporter.h

UCLASS()
class UFMulticastDelegatePropertyExporter : public UObject
{
    GENERATED_BODY()
public:
    UNREALSHARP_FUNCTION()
    static void AddDelegate(FMulticastDelegateProperty* DelegateProperty, 
                           FMulticastScriptDelegate* Delegate, 
                           UObject* Target, const char* FunctionName);
    
    UNREALSHARP_FUNCTION()
    static void RemoveDelegate(FMulticastDelegateProperty* DelegateProperty, 
                              FMulticastScriptDelegate* Delegate, 
                              UObject* Target, const char* FunctionName);
    
    UNREALSHARP_FUNCTION()
    static void BroadcastDelegate(FMulticastDelegateProperty* DelegateProperty, 
                                  const FMulticastScriptDelegate* Delegate, 
                                  void* Parameters);
    
    UNREALSHARP_FUNCTION()
    static bool ContainsDelegate(FMulticastDelegateProperty* DelegateProperty, 
                                 const FMulticastScriptDelegate* Delegate, 
                                 UObject* Target, const char* FunctionName);
};
```

### 4.7 其他重要 Exporter

| Exporter 类 | 职责 |
|------------|------|
| `UUFunctionExporter` | UFunction 操作 |
| `UFMapPropertyExporter` | TMap 操作 |
| `UFSetPropertyExporter` | TSet 操作 |
| `UFOptionalPropertyExporter` | TOptional 操作 |
| `UFTextExporter` | FText 操作 |
| `UFNameExporter` | FName 操作 |
| `UAsyncExporter` | 异步操作 |
| `UFScriptArrayExporter` | 脚本数组底层操作 |

---

## 五、C# 侧调用链

### 5.1 NativeCallbacksWrapperGenerator

UnrealSharp 使用 Roslyn Source Generator 自动生成调用代码：

```csharp
// Source/UnrealSharp/UnrealSharp.SourceGenerators/NativeCallbacksWrapperGenerator.cs

[Generator]
public class NativeCallbacksWrapperGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var classDeclarations = context.SyntaxProvider.CreateSyntaxProvider(
            static (syntaxNode, _) => 
                syntaxNode is ClassDeclarationSyntax cds && cds.AttributeLists.Count > 0,
            static (syntaxContext, _) => GetClassInfoOrNull(syntaxContext));

        var classAndCompilation = classDeclarations.Combine(context.CompilationProvider);

        context.RegisterSourceOutput(classAndCompilation, (spc, pair) =>
        {
            GenerateForClass(spc, compilation, maybeClassInfo.Value);
        });
    }
}
```

### 5.2 生成的调用代码

假设有以下 C# 声明：

```csharp
[NativeCallbacks]
public static unsafe partial class UObjectExporter
{
    public static delegate* unmanaged<IntPtr, FName> GetName;
}
```

Source Generator 会生成：

```csharp
public static unsafe partial class UObjectExporter
{
    static UObjectExporter()
    {
        int GetNameTotalSize = sizeof(IntPtr) + sizeof(FName);
        IntPtr GetNameFuncPtr = UnrealSharp.Binds.NativeBinds.TryGetBoundFunction(
            "UObjectExporter", "GetName", GetNameTotalSize);
        GetName = (delegate* unmanaged<IntPtr, FName>)GetNameFuncPtr;
    }
    
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static FName CallGetName(IntPtr nativeObject)
    {
        return GetName(nativeObject);
    }
}
```

### 5.3 调用流程

```
C# UObjectExporter.CallGetName(ptr)
    ↓
GetName 函数指针调用（直接内存跳转）
    ↓
C++ UUObjectExporter::GetName()
    ↓
返回 FName（直接内存拷贝）
```

**关键点**：编译后，C# 调用直接跳转到 C++ 函数地址，零开销！

---

## 六、参数封送规则

### 6.1 Blittable 类型

Blittable 类型可以直接内存传递：

| C# 类型 | C++ 类型 | 说明 |
|--------|---------|------|
| `int` | `int32` | 直接传递 |
| `float` | `float` | 直接传递 |
| `double` | `double` | 直接传递 |
| `bool` | `bool` | 注意：C# bool 是 1 字节 |
| `IntPtr` | `void*` | 指针 |
| 枚举 | 枚举 | 底层整数传递 |

### 6.2 非Blittable 类型处理

```cpp
// FString 的封送
UNREALSHARP_FUNCTION()
static void MarshalToNativeStringView(FString* NativeString, 
                                      const UTF16CHAR* ManagedString, 
                                      int32 Length)
{
    // C# string (UTF-16) → FString (TCHAR)
    const auto Converted = StringCast<TCHAR>(ManagedString, Length);
    *NativeString = FString(Converted.Length(), Converted.Get());
}
```

### 6.3 引用参数

```cpp
// C++ 端
UNREALSHARP_FUNCTION()
static void GetArrayData(FArrayProperty* Property, void* ArrayPtr, void*& OutData);

// C# 端
public static delegate* unmanaged<IntPtr, IntPtr, out IntPtr, void> GetArrayData;
```

---

## 七、实践：添加新的导出函数

### 7.1 声明

```cpp
// MyExporter.h
UCLASS()
class UMyExporter : public UObject
{
    GENERATED_BODY()
public:
    UNREALSHARP_FUNCTION()
    static float CalculateDistance(FVector* A, FVector* B);
};
```

### 7.2 实现

```cpp
// MyExporter.cpp
float UMyExporter::CalculateDistance(FVector* A, FVector* B)
{
    if (!A || !B)
    {
        return 0.0f;
    }
    return FVector::Dist(*A, *B);
}
```

### 7.3 注册（自动）

模块启动时，所有带 `UNREALSHARP_FUNCTION` 的静态函数会自动注册。

### 7.4 C# 侧声明

```csharp
[NativeCallbacks]
public static unsafe partial class MyExporter
{
    public static delegate* unmanaged<FVector*, FVector*, float> CalculateDistance;
}
```

### 7.5 使用

```csharp
float distance = MyExporter.CallCalculateDistance(ptrA, ptrB);
```

---

## 八、性能考量

### 8.1 零开销抽象

```
传统 P/Invoke:
C# → P/Invoke 封送 → DLL 导出表查找 → C++ 函数

UNREALSHARP_FUNCTION:
C# → 函数指针直接调用 → C++ 函数
```

### 8.2 调用开销分析

| 操作 | 开销 |
|------|------|
| 函数指针调用 | ~1-2 CPU周期 |
| Blittable 参数传递 | 0（寄存器传递） |
| 非Blittable 封送 | 取决于类型 |

### 8.3 最佳实践

1. **优先使用 Blittable 类型**
2. **批量操作代替多次小调用**
3. **避免频繁字符串转换**

---

## 九、总结

### 核心设计

```
┌─────────────────────────────────────────────────────────────────┐
│                    C++ 导出函数声明                               │
│  UNREALSHARP_FUNCTION() static void Func(Args...);             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    模块启动时自动注册                             │
│  FCSBindsManager::RegisterExportedFunction()                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Roslyn Source Generator                       │
│  生成 C# 静态构造函数，获取函数指针                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    C# 调用                                       │
│  MyExporter.CallFunc(args) → 函数指针 → C++ Func()               │
└─────────────────────────────────────────────────────────────────┘
```

### 关键源码文件

| 文件 | 职责 |
|------|------|
| `CSBindsManager.h` | 绑定管理器 |
| `CSExportedFunction.h` | 导出函数结构 |
| `Export/*.h` | 各领域导出函数 |
| `NativeCallbacksWrapperGenerator.cs` | Source Generator |

---

**下一篇**：方法调用的完整流程 - 一个方法调用的前世今生
