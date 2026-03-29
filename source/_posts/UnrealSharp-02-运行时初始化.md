---
title: 深入理解 .NET 运行时嵌入 - CoreCLR 与 Mono 的初始化之道
date: 2026-03-29 22:00:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, CoreCLR, Mono, 运行时嵌入]
---

# 深入理解 .NET 运行时嵌入 - CoreCLR 与 Mono 的初始化之道

> **作者**：GLM-5.0

作为虚幻引擎开发者，你可能已经熟悉了 UE 的模块加载机制。但当需要在 UE 中嵌入 .NET 运行时时，事情变得有趣起来。本章将深入剖析 UnrealSharp 如何将 CoreCLR 和 Mono 两种运行时嵌入到 UE 进程中。

## 一、从你的练习代码说起

你可能已经写过类似的 CoreCLR embed 代码（来自 `ClrAppDemo.cpp`）：

```cpp
// 1. 获取 hostfxr 路径
char_t hostfxrPath[1024];
get_hostfxr_path(hostfxrPath, &hostfxrPathSize, nullptr);

// 2. 加载 hostfxr.dll 并获取函数指针
libHostFxr = LoadLibraryW(hostfxrPath);
init_config_fptr = (hostfxr_initialize_for_runtime_config_fn)
    GetProcAddress(hModule, "hostfxr_initialize_for_runtime_config");
get_delegate_fptr = (hostfxr_get_runtime_delegate_fn)
    GetProcAddress(hModule, "hostfxr_get_runtime_delegate");

// 3. 初始化运行时
hostfxr_handle handle = nullptr;
init_config_fptr(config_path, nullptr, &handle);

// 4. 获取程序集加载函数
get_delegate_fptr(handle, hdt_load_assembly_and_get_function_pointer, 
    (void**)&load_assembly_and_get_function_pointer);

// 5. 加载程序集并调用方法
load_assembly_and_get_function_pointer(assemblyPath, dotnet_type, 
    methodName, UNMANAGEDCALLERSONLY_METHOD, nullptr, &PrintMessageFuncPtr);
```

以及 Mono embed 代码（来自 `MonoAppDemo.cpp`）：

```cpp
// 1. 设置程序集搜索路径
mono_set_dirs("E:/Code/MonoSDK/Win64/lib", "E:/Code/MonoSDK/Win64/dll");

// 2. 初始化 JIT
MonoDomain* domain = mono_jit_init("MyMonoApp");

// 3. 加载程序集
MonoAssembly* assembly = mono_domain_assembly_open(domain, assemblyPath);
MonoImage* image = mono_assembly_get_image(assembly);

// 4. 获取类和方法
MonoClass* ManagedClass = mono_class_from_name(image, "ManagedDemo", "ManagedClass");
MonoMethod* MsgMethod = mono_class_get_method_from_name(ManagedClass, "PrintMessage", 1);

// 5. 调用方法
mono_runtime_invoke(MsgMethod, nullptr, args, &exc);
```

这是最基础的 embed 模式。UnrealSharp 在此基础上构建了一套**工业级**的运行时管理系统。让我们看看它是如何演进的。

---

## 二、运行时选择机制

### 2.1 编译时选择：UNREALSHARP_MONO 宏

UnrealSharp 通过编译宏 `UNREALSHARP_MONO` 在编译时决定使用哪种运行时：

```cpp
// CSManager.cpp
#if UNREALSHARP_MONO
#include "CSMonoRuntime.h"
#endif
```

这个宏由 `MonoSDK.Build.cs` 根据 `DefaultEngine.ini` 配置自动定义：

```csharp
// MonoSDK.Build.cs
bool bUseMono = false;
ConfigHierarchy EngineIni = ConfigCache.ReadHierarchy(...);
EngineIni.GetBool("UnrealSharp", "bUseMono", out bUseMono);

if (bUseMono)
{
    PublicDefinitions.Add("UNREALSHARP_MONO=1");
}
else
{
    PublicDefinitions.Add("UNREALSHARP_MONO=0");
}
```

### 2.2 配置驱动

在项目的 `DefaultEngine.ini` 中：

```ini
[UnrealSharp]
bUseMono=true  ; true 使用 Mono, false 使用 CoreCLR
```

### 2.3 运行时初始化入口

`CSManager::InitializeDotNetRuntime()` 是运行时初始化的核心入口：

```cpp
// CSManager.cpp:181-241
bool UCSManager::InitializeDotNetRuntime()
{
    // 第一步：加载运行时宿主
    if (!LoadRuntimeHost())
    {
        UE_LOG(LogUnrealSharp, Fatal, TEXT("Failed to load Runtime Host"));
        return false;
    }

#if UNREALSHARP_MONO
    // --- Mono 路径 ---
    if (!InitializeMonoHost())
    {
        UE_LOG(LogUnrealSharp, Fatal, TEXT("[Mono] Failed to initialize Mono host."));
        return false;
    }
    return true;
#else
    // --- CoreCLR 路径 ---
    load_assembly_and_get_function_pointer_fn LoadAssemblyAndGetFunctionPointer = InitializeNativeHost();
    if (!LoadAssemblyAndGetFunctionPointer)
    {
        UE_LOG(LogUnrealSharp, Fatal, TEXT("Failed to initialize Runtime Host."));
        return false;
    }

    // 调用 C# 入口点...
    FInitializeRuntimeHost InitializeUnrealSharp = nullptr;
    const int32 ErrorCode = LoadAssemblyAndGetFunctionPointer(
        PLATFORM_STRING(*UnrealSharpLibraryAssembly),
        PLATFORM_STRING(*EntryPointClassName),
        PLATFORM_STRING(*EntryPointFunctionName),
        UNMANAGEDCALLERSONLY_METHOD,
        nullptr,
        reinterpret_cast<void**>(&InitializeUnrealSharp));

    // 执行初始化
    if (!InitializeUnrealSharp(*UserWorkingDirectory,
        *UnrealSharpLibraryAssembly,
        &ManagedPluginsCallbacks,
        (const void*)&FCSBindsManager::GetBoundFunction,
        &FCSManagedCallbacks::ManagedCallbacks))
    {
        return false;
    }

    return true;
#endif
}
```

---

## 三、CoreCLR 集成详解

### 3.1 加载 hostfxr

CoreCLR 的嵌入从加载 `hostfxr` 开始。`hostfxr` 是 .NET 的宿主解析器，负责找到并初始化正确的 .NET 运行时：

```cpp
// CSManager.cpp:243-290
bool UCSManager::LoadRuntimeHost()
{
#if UNREALSHARP_MONO
    // Mono 运行时是编译时链接的，无需动态加载
    return true;
#else
    const FString RuntimeHostPath = UCSProcUtilities::GetRuntimeHostPath();
    if (!FPaths::FileExists(RuntimeHostPath))
    {
        UE_LOG(LogUnrealSharp, Error, TEXT("Couldn't find Hostfxr.dll"));
        return false;
    }

    // 加载 DLL
    RuntimeHost = FPlatformProcess::GetDllHandle(*RuntimeHostPath);

    // 获取关键函数指针
    void* DLLHandle = FPlatformProcess::GetDllExport(RuntimeHost, 
        TEXT("hostfxr_initialize_for_runtime_config"));
    Hostfxr_Initialize_For_Runtime_Config = 
        static_cast<hostfxr_initialize_for_runtime_config_fn>(DLLHandle);

    DLLHandle = FPlatformProcess::GetDllExport(RuntimeHost, 
        TEXT("hostfxr_get_runtime_delegate"));
    Hostfxr_Get_Runtime_Delegate = 
        static_cast<hostfxr_get_runtime_delegate_fn>(DLLHandle);

    DLLHandle = FPlatformProcess::GetDllExport(RuntimeHost, TEXT("hostfxr_close"));
    Hostfxr_Close = static_cast<hostfxr_close_fn>(DLLHandle);

    return Hostfxr_Initialize_For_Runtime_Config && 
           Hostfxr_Get_Runtime_Delegate && 
           Hostfxr_Close;
#endif
}
```

**与你的练习代码对比**：

| 你的代码 | UnrealSharp |
|---------|-------------|
| `LoadLibraryW()` | `FPlatformProcess::GetDllHandle()` |
| `GetProcAddress()` | `FPlatformProcess::GetDllExport()` |
| 硬编码路径 | `UCSProcUtilities::GetRuntimeHostPath()` |

UnrealSharp 使用 UE 的跨平台抽象层，支持 Windows/Mac/Linux。

### 3.2 初始化运行时

`InitializeNativeHost()` 完成运行时的初始化：

```cpp
// CSManager.cpp:397-478
load_assembly_and_get_function_pointer_fn UCSManager::InitializeNativeHost() const
{
    // 确定 .NET 目录
#if WITH_EDITOR
    FString DotNetPath = UCSProcUtilities::GetDotNetDirectory();
#else
    FString DotNetPath = UCSProcUtilities::GetPluginAssembliesPath();
#endif

    // 准备初始化参数
    hostfxr_initialize_parameters InitializeParameters;
    InitializeParameters.dotnet_root = PLATFORM_STRING(*DotNetPath);
    InitializeParameters.host_path = PLATFORM_STRING(*RuntimeHostPath);
    InitializeParameters.size = sizeof(hostfxr_initialize_parameters);

    hostfxr_handle HostFXR_Handle = nullptr;
    int32 ErrorCode = 0;

#if WITH_EDITOR
    // 编辑器模式：使用 runtimeconfig.json 初始化
    FString RuntimeConfigPath = UCSProcUtilities::GetRuntimeConfigPath();
    ErrorCode = Hostfxr_Initialize_For_Runtime_Config(
        PLATFORM_STRING(*RuntimeConfigPath), 
        &InitializeParameters, 
        &HostFXR_Handle);
#else
    // 打包模式：使用命令行方式初始化
    std::vector Args { PLATFORM_STRING(*PluginAssemblyPath) };
    ErrorCode = Hostfxr_Initialize_For_Dotnet_Command_Line(
        Args.size(), Args.data(), &InitializeParameters, &HostFXR_Handle);
#endif

    if (ErrorCode != 0)
    {
        UE_LOG(LogUnrealSharp, Error, 
            TEXT("hostfxr_initialize_for_runtime_config failed with code: %d"), ErrorCode);
        return nullptr;
    }

    // 获取程序集加载函数
    void* LoadAssemblyAndGetFunctionPointer = nullptr;
    ErrorCode = Hostfxr_Get_Runtime_Delegate(
        HostFXR_Handle, 
        hdt_load_assembly_and_get_function_pointer, 
        &LoadAssemblyAndGetFunctionPointer);
    
    Hostfxr_Close(HostFXR_Handle);  // 关闭句柄，但运行时继续运行

    return (load_assembly_and_get_function_pointer_fn)LoadAssemblyAndGetFunctionPointer;
}
```

**关键设计点**：

1. **编辑器 vs 打包模式**：
   - 编辑器：使用 `hostfxr_initialize_for_runtime_config`，更灵活
   - 打包：使用 `hostfxr_initialize_for_dotnet_command_line`，更简洁

2. **初始化后立即关闭句柄**：
   - `Hostfxr_Close()` 关闭的是配置句柄，不是运行时
   - 运行时一旦初始化就会一直运行直到进程结束

### 3.3 调用 C# 入口点

运行时初始化后，UnrealSharp 立即调用 C# 的入口函数：

```cpp
// CSManager.cpp:208-237
const FString EntryPointClassName = TEXT("UnrealSharp.Plugins.Main, UnrealSharp.Plugins");
const FString EntryPointFunctionName = TEXT("InitializeUnrealSharp");

FInitializeRuntimeHost InitializeUnrealSharp = nullptr;
const int32 ErrorCode = LoadAssemblyAndGetFunctionPointer(
    PLATFORM_STRING(*UnrealSharpLibraryAssembly),
    PLATFORM_STRING(*EntryPointClassName),
    PLATFORM_STRING(*EntryPointFunctionName),
    UNMANAGEDCALLERSONLY_METHOD,
    nullptr,
    reinterpret_cast<void**>(&InitializeUnrealSharp));

// 执行初始化
if (!InitializeUnrealSharp(
    *UserWorkingDirectory,
    *UnrealSharpLibraryAssembly,
    &ManagedPluginsCallbacks,                    // 插件回调
    (const void*)&FCSBindsManager::GetBoundFunction,  // 原生函数绑定
    &FCSManagedCallbacks::ManagedCallbacks))          // 托管回调
{
    UE_LOG(LogUnrealSharp, Fatal, TEXT("Failed to initialize UnrealSharp!"));
    return false;
}
```

**参数解析**：

| 参数 | 类型 | 作用 |
|-----|------|------|
| `UserWorkingDirectory` | `char*` | 用户程序集目录 |
| `UnrealSharpLibraryAssembly` | `nint` | 插件程序集路径 |
| `ManagedPluginsCallbacks` | `PluginsCallbacks*` | C++ 提供给 C# 的回调 |
| `GetBoundFunction` | `IntPtr` | 原生函数查找函数指针 |
| `ManagedCallbacks` | `IntPtr` | C# 回调结构体指针 |

---

## 四、Mono 集成详解

### 4.1 为什么需要 Mono？

CoreCLR 虽然强大，但在某些平台上有局限性：

| 平台 | CoreCLR | Mono |
|-----|---------|------|
| Windows | ✅ 完全支持 | ✅ 支持 |
| macOS | ✅ 完全支持 | ✅ 支持 |
| iOS | ❌ 不支持 JIT | ✅ AOT + 解释器 |
| Android | ⚠️ 有限支持 | ✅ 完全支持 |

Mono 的 AOT（Ahead-of-Time）编译模式可以在禁止 JIT 的平台（如 iOS）上运行。

### 4.2 Mono 运行时初始化

Mono 的初始化流程比 CoreCLR 更复杂，需要处理更多平台差异：

```cpp
// CSMonoRuntime.cpp:228-511
MonoDomain* InitializeMonoRuntime(const FString& RuntimeDir, const FString& ExtraSearchPaths)
{
    // 步骤 0：启用 Mono 日志
    mono_trace_set_log_handler(OnMonoLog, nullptr);
    mono_trace_set_print_handler(OnMonoPrint);
    mono_trace_set_level_string("warning");

    // 步骤 1：设置程序集搜索路径
    // Windows 用 ';' 分隔，其他平台用 ':'
#if PLATFORM_WINDOWS
    const TCHAR* MonoPathSep = TEXT(";");
#else
    const TCHAR* MonoPathSep = TEXT(":");
#endif
    FString AssembliesPath = RuntimeDir;
    if (!ExtraSearchPaths.IsEmpty())
    {
        AssembliesPath += MonoPathSep + ExtraSearchPaths;
    }
    mono_set_assemblies_path(TCHAR_TO_UTF8(*AssembliesPath));

    // 步骤 2：macOS/Linux 线程模式设置
    // 避免与 UE5 的信号处理器冲突
#if PLATFORM_MAC || PLATFORM_LINUX
    FPlatformMisc::SetEnvironmentVar(TEXT("MONO_THREADS_SUSPEND"), TEXT("preemptive"));
#endif

    // 步骤 3：iOS 特殊处理
#if PLATFORM_IOS
    // AOT + 解释器模式
    mono_jit_set_aot_mode(MONO_AOT_MODE_INTERP);
    
    // 注册 AOT 模块
    extern void* mono_aot_module_System_Private_CoreLib_info;
    mono_aot_register_module(static_cast<void**>(mono_aot_module_System_Private_CoreLib_info));
    
    // 注册 DL fallback 处理器
    mono_dl_fallback_register(OnMonoDlFallbackLoad, OnMonoDlFallbackSymbol, 
        OnMonoDlFallbackClose, nullptr);
    
    // 禁用 ICU（iOS 没有打包 ICU 数据）
    setenv("DOTNET_SYSTEM_GLOBALIZATION_INVARIANT", "1", 1);
#else
    // 非 iOS：使用 JIT
    mono_jit_set_aot_mode(MONO_AOT_MODE_NONE);
#endif

    // 步骤 4：解析配置
    mono_config_parse(nullptr);

    // 步骤 5：初始化 JIT
    MonoDomain* Domain = mono_jit_init_version("UnrealSharp", "v4.0.30319");

    // 步骤 6：注册主线程
    mono_thread_set_main(mono_thread_current());

    return Domain;
}
```

### 4.3 iOS 平台的特殊挑战

iOS 是 UnrealSharp 最复杂的支持平台，面临三大挑战：

**挑战 1：禁止 JIT**

Apple 的 W^X（Write XOR Execute）策略禁止在运行时生成可执行代码。Mono 使用 AOT + 解释器模式绕过：

```cpp
mono_jit_set_aot_mode(MONO_AOT_MODE_INTERP);
mono_aot_register_module(static_cast<void**>(mono_aot_module_System_Private_CoreLib_info));
```

`MONO_AOT_MODE_INTERP` 的含义：
- 使用预编译的 AOT 表进行引导
- 对于缺少 AOT 覆盖的方法，回退到字节码解释器

**挑战 2：BCL 原生库加载**

.NET 基类库（BCL）依赖一些原生库（如 `libSystem.Native.dylib`），它们被打包到 `Mono.framework/Frameworks/` 目录下。但 dyld 的 `@rpath` 只包含 `Frameworks/` 一级，无法直接找到子目录中的库。

UnrealSharp 通过 `mono_dl_fallback_register` 解决：

```cpp
// CSMonoRuntime.cpp:69-98
static void* OnMonoDlFallbackLoad(const char* Name, int Flags, char** Err, void*)
{
    if (!IsBclNativeLib(Name)) return nullptr;

    // 构建完整路径: <App>/Frameworks/Mono.framework/Frameworks/libFoo.dylib
    FString FullPath = FPaths::Combine(GMonoFrameworksPath, 
        FString::Printf(TEXT("lib%s.dylib"), UTF8_TO_TCHAR(CleanName)));
    
    return dlopen(TCHAR_TO_UTF8(*FullPath), Flags ? Flags : RTLD_LAZY);
}
```

**挑战 3：全球化支持**

iOS 设备没有打包 ICU 数据，需要使用不变全球化模式：

```cpp
setenv("DOTNET_SYSTEM_GLOBALIZATION_INVARIANT", "1", 1);
```

### 4.4 UFS 程序集加载器

Mono 的 `mono_assembly_open` 使用 POSIX `fopen`，无法读取 UE 的 pak 文件。UnrealSharp 实现了 UFS（Unreal File System）感知的程序集加载器：

```cpp
// CSMonoRuntime.cpp:144-202
static MonoAssembly* LoadAssemblyFromBytes(const FString& FilePath)
{
    TArray<uint8> FileBytes;
    // FFileHelper 支持 pak 文件读取
    if (!FFileHelper::LoadFileToArray(FileBytes, *FilePath, FILEREAD_Silent))
    {
        return nullptr;
    }

    MonoImageOpenStatus Status = MONO_IMAGE_OK;
    MonoImage* Image = mono_image_open_from_data_with_name(
        reinterpret_cast<char*>(FileBytes.GetData()),
        static_cast<uint32_t>(FileBytes.Num()),
        /*need_copy=*/true,
        &Status,
        /*refonly=*/false,
        TCHAR_TO_UTF8(*FilePath));

    MonoAssembly* Assembly = mono_assembly_load_from(Image, TCHAR_TO_UTF8(*FilePath), &Status);
    return Assembly;
}

// 注册预加载钩子
mono_install_assembly_preload_hook(OnMonoAssemblyPreload, nullptr);
```

### 4.5 调用 C# 入口点

Mono 模式下，调用 C# 入口点的方式与 CoreCLR 不同：

```cpp
// CSManager_Mono.cpp:120-195
MonoMethod* InitMethod = FindMonoMethod(
    MonoRootDomain,
    TCHAR_TO_UTF8(*UnrealSharpLibraryAssembly),
    "UnrealSharp.Plugins.Main",
    "InitializeUnrealSharp",
    5);  // 5 个参数

// 准备参数
void* Args[5] = {
    (void*)WorkDirPtr,           // char* workDir
    &AssemblyNint,               // nint assemblyPath
    (void*)PluginsCallbacksPtr,  // PluginsCallbacks*
    &BindsFn,                    // IntPtr bindsCallbacks
    &ManagedCallbacksPtr         // IntPtr managedCallbacks
};

// 调用方法
MonoObject* Exception = nullptr;
MonoObject* Result = mono_runtime_invoke(InitMethod, nullptr, Args, &Exception);
```

**与 CoreCLR 的关键差异**：

| 特性 | CoreCLR | Mono |
|-----|---------|------|
| 方法调用 | 直接函数指针 | `mono_runtime_invoke` |
| 参数传递 | 按签名传递 | 数组传递，值类型需要取地址 |
| 异常处理 | 自动传播 | 需要 `MonoObject**` 捕获 |

---

## 五、C# 侧的初始化

无论使用 CoreCLR 还是 Mono，最终都会调用同一个 C# 入口点：

```csharp
// Main.cs:10-51
public static class Main
{
    [UnmanagedCallersOnly]
    private static unsafe NativeBool InitializeUnrealSharp(
        char* workingDirectoryPath, 
        nint assemblyPath, 
        PluginsCallbacks* pluginCallbacks, 
        IntPtr bindsCallbacks, 
        IntPtr managedCallbacks)
    {
        try
        {
            // 设置 AppDomain 基目录
            AppDomain.CurrentDomain.SetData("APP_CONTEXT_BASE_DIRECTORY", 
                new string(workingDirectoryPath));
            
            // 初始化回调指针
            PluginsCallbacks.Initialize(pluginCallbacks);
            ManagedCallbacks.Initialize(managedCallbacks);
            NativeBinds.Initialize(bindsCallbacks);

            Console.WriteLine("UnrealSharp initialized successfully.");
            return NativeBool.True;
        }
        catch (Exception exception)
        {
            Console.WriteLine(exception);
#if UNREALSHARP_MONO
            // Mono 模式下写入临时文件供 C++ 读取
            try { File.WriteAllText("/tmp/UnrealSharp_InitException.txt", 
                exception.ToString()); } catch { }
#endif
            return NativeBool.False;
        }
    }
}
```

**初始化的三大回调系统**：

| 回调系统 | 作用 |
|---------|------|
| `PluginsCallbacks` | C# 提供给 C++ 的插件管理回调 |
| `ManagedCallbacks` | C# 提供给 C++ 的核心操作回调（创建对象、调用方法等） |
| `NativeBinds` | C++ 提供给 C# 的原生函数绑定 |

---

## 六、两种运行时对比

### 6.1 初始化流程对比

```
CoreCLR 初始化流程:
┌─────────────────┐
│ get_hostfxr_path│ ← 找到 hostfxr.dll
└────────┬────────┘
         ▼
┌─────────────────┐
│ LoadLibrary     │ ← 加载 hostfxr.dll
└────────┬────────┘
         ▼
┌─────────────────────────────────────┐
│ hostfxr_initialize_for_runtime_config│ ← 初始化运行时配置
└────────┬────────────────────────────┘
         ▼
┌─────────────────────────────────────────┐
│ hostfxr_get_runtime_delegate            │ ← 获取程序集加载函数
│ (hdt_load_assembly_and_get_function_pointer)│
└────────┬────────────────────────────────┘
         ▼
┌─────────────────────────────────────┐
│ load_assembly_and_get_function_pointer│ ← 加载入口程序集
└────────┬────────────────────────────┘
         ▼
┌─────────────────────┐
│ 直接函数指针调用    │ ← 调用 InitializeUnrealSharp
└─────────────────────┘

Mono 初始化流程:
┌─────────────────────┐
│ mono_set_assemblies_path│ ← 设置搜索路径
└────────┬────────────┘
         ▼
┌─────────────────────────┐
│ mono_jit_set_aot_mode   │ ← 设置运行模式（JIT/AOT/解释器）
└────────┬────────────────┘
         ▼
┌─────────────────────┐
│ mono_config_parse   │ ← 解析配置
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ mono_jit_init_version│ ← 初始化 JIT，创建根域
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ FindMonoMethod      │ ← 通过反射找到入口方法
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ mono_runtime_invoke │ ← 反射调用 InitializeUnrealSharp
└─────────────────────┘
```

### 6.2 性能对比

| 指标 | CoreCLR | Mono (JIT) | Mono (AOT+Interp) |
|-----|---------|------------|-------------------|
| 启动时间 | 快 | 中等 | 慢 |
| 方法调用 | 函数指针，快 | 反射调用，中等 | 解释器，慢 |
| 内存占用 | 中等 | 较低 | 较低 |
| 平台支持 | Win/Mac/Linux | 全平台 | iOS/Android |

### 6.3 选择建议

| 场景 | 推荐运行时 |
|-----|-----------|
| Windows 编辑器开发 | CoreCLR |
| macOS 编辑器开发 | CoreCLR 或 Mono |
| iOS 发布 | Mono (AOT+Interp) |
| Android 发布 | Mono |
| 追求最佳性能 | CoreCLR |
| 追求最广兼容性 | Mono |

---

## 七、总结

### 7.1 核心要点

1. **双运行时架构**：通过 `UNREALSHARP_MONO` 宏在编译时选择运行时
2. **平台适配**：针对 iOS 等特殊平台实现了 AOT+解释器模式
3. **UFS 支持**：Mono 模式下实现了 pak 文件感知的程序集加载
4. **回调初始化**：运行时初始化后立即建立 C++ 与 C# 的双向通信

### 7.2 与基础 embed 的演进

| 方面 | 基础 embed | UnrealSharp |
|-----|-----------|-------------|
| 运行时选择 | 单一运行时 | 双运行时支持 |
| 平台适配 | 无 | 全平台支持 |
| 错误处理 | 简单 | 完整的日志和异常捕获 |
| 程序集管理 | 手动 | 自动化的程序集加载器 |
| 生命周期 | 无管理 | 与 UE 模块系统集成 |

### 7.3 下一篇预告

运行时初始化完成后，下一个关键问题是如何管理跨语言的对象生命周期。下一篇将深入剖析 GCHandle 机制和对象生命周期同步。

---

## 参考资料

- [.NET Hosting Design](https://github.com/dotnet/runtime/blob/main/docs/design/features/hosting.md)
- [Mono Embedding API](https://www.mono-project.com/docs/advanced/embedding/)
- [hostfxr.h](https://github.com/dotnet/runtime/blob/main/src/native/corehost/hostfxr.h)

---

*下一篇：跨越边界的引用 - GCHandle 与托管对象生命周期管理*
