---
title: "Glue代码生成系统 - 从 C++ 到 C# 的自动化桥梁"
date: 2025-03-29 12:00:00 +0800
categories: [Unreal Engine, UnrealSharp]
tags: [unreal-engine, csharp, dotnet, coreclr, mono, glue-code, uht]
series: UnrealSharp 插件技术深度解析
---

> **作者**：GLM-5.0

当你在 C# 中写下 `var actor = new AActor();` 时，有没有想过这个 `AActor` 类是如何从 C++ 的 `AActor` 映射过来的？这就是 Glue 代码生成系统的职责 —— 它是一座自动化桥梁，将虚幻引擎的 C++ API 转换为 C# 可调用的托管代码。

## 整体架构

UnrealSharp 的 Glue 代码生成系统由两大核心组件构成：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Glue Code Generation                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────┐     ┌─────────────────────────────┐   │
│  │ UnrealSharpManagedGlue │     │ UnrealSharp.SourceGenerators │   │
│  │   (UHT Plugin)        │     │   (Roslyn Generator)         │   │
│  │                       │     │                               │   │
│  │  C++ → C# 编译时生成   │     │  C# → C# 编译时增强           │   │
│  └─────────────────────┘     └─────────────────────────────┘   │
│           │                              │                      │
│           ▼                              ▼                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Generated C# Bindings                        │   │
│  │    (YourProject.Glue/YourModule/*.generated.cs)          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**两套系统的分工**：

| 组件 | 触发时机 | 输出 | 职责 |
|-----|---------|------|------|
| `UnrealSharpManagedGlue` | UHT 运行时 | `.generated.cs` | 从 C++ 反射信息生成 C# 类、属性、方法 |
| `UnrealSharp.SourceGenerators` | C# 编译时 | `.g.cs` | 增强 C# 类（日志、回调包装等） |

## UnrealSharpManagedGlue：UHT 插件

### 入口与启动流程

`UnrealSharpManagedGlue` 是一个 **UHT（Unreal Header Tool）插件**，在 UE 编译时自动运行：

```csharp
// Program.cs
[UnrealHeaderTool]
public static class Program
{
    [UhtExporter(Name = "UnrealSharpCore", ...)]
    private static void Main(IUhtExportFactory factory)
    {
        GeneratorStatics.Initialize(factory);
        USharpBuildToolUtilities.CompileUSharpBuildTool();
        
        // 1. 开始导出
        CSharpExporter.StartExport();
        
        // 2. 清理旧文件
        FileExporter.CleanOldExportedFiles();
        
        // 3. 创建/更新 Glue 项目文件
        GlueModuleFactory.CreateGlueProjects();
        
        // 4. 如果引擎 Glue 有变化，重新编译绑定库
        if (CSharpExporter.HasModifiedEngineGlue)
        {
            DotNetUtilities.BuildSolution(...);
        }
    }
}
```

**关键点**：
- `[UnrealHeaderTool]` 特性标记这是一个 UHT 插件
- `[UhtExporter]` 指定导出器名称和关联模块
- 整个过程在 UE 编译期间自动执行，无需手动干预

### CSharpExporter：核心导出引擎

`CSharpExporter` 是代码生成的核心引擎：

```csharp
public static class CSharpExporter
{
    public static void StartExport()
    {
        // 检查生成器源码是否变化，决定是否全量重新生成
        if (HasChangedGeneratorSourceRecently())
        {
            // 清理所有生成的文件
            foreach (var module in Factory.Session.Modules)
            {
                FileExporter.CleanGeneratedFolder(moduleInfo.GlueBaseDirectory);
            }
        }
        else
        {
            // 增量生成：只处理变化的头文件
            PackageHeadersTracker.DeserializeModuleHeaders();
        }
        
        // 遍历所有模块和包
        foreach (UhtModule module in Factory.Session.Modules)
        {
            foreach (UhtPackage package in module.Packages)
            {
                ExportPackage(package);
            }
        }
        
        // 并行等待所有任务完成
        TaskManager.WaitForTasks();
        
        // 导出扩展方法和自动转换函数
        FunctionExporter.StartExportingExtensionMethods();
        AutocastExporter.StartExportingAutocastFunctions();
    }
}
```

**增量生成策略**：

```csharp
private static void ExportPackage(UhtPackage package)
{
    bool generatedGlueFolderExists = Directory.Exists(generatedPath);
    
    for (int i = 0; i < package.Children.Count; i++)
    {
        UhtType child = package.Children[i];
        
        // 增量判断：头文件是否变化
        if (!generatedGlueFolderExists || 
            PackageHeadersTracker.HasHeaderFileChanged(packageName, child.HeaderFile))
        {
            ForEachChild(child, ExportType);  // 重新生成
        }
        else
        {
            ForEachChild(child, FileExporter.AddUnchangedType);  // 跳过
        }
    }
}
```

### 类型导出器家族

UnrealSharp 支持导出多种 UE 类型：

```csharp
private static void ExportType(UhtType type)
{
    if (type is UhtClass classObj)
    {
        if (classObj.HasAllFlags(EClassFlags.Interface))
        {
            InterfaceExporter.ExportInterface(classObj);
        }
        else
        {
            ClassExporter.ExportClass(classObj, isManualExport);
        }
    }
    else if (type is UhtEnum enumObj)
    {
        EnumExporter.ExportEnum(enumObj);
    }
    else if (type is UhtScriptStruct structObj)
    {
        StructExporter.ExportStruct(structObj, isManualExport);
    }
    else if (type.EngineType is UhtEngineType.Delegate)
    {
        DelegateExporter.ExportDelegate(delegateFunction);
    }
}
```

### ClassExporter：类导出详解

以类导出为例，看看完整的代码生成流程：

```csharp
public static void ExportClass(UhtClass classObj, bool isManualExport)
{
    GeneratorStringBuilder stringBuilder = new();
    
    // 1. 收集需要导出的成员
    List<UhtFunction> exportedFunctions = new();
    List<UhtFunction> exportedOverrides = new();
    Dictionary<string, GetterSetterPair> exportedGetterSetters = new();
    
    ScriptGeneratorUtilities.GetExportedFunctions(classObj, 
        exportedFunctions, exportedOverrides, 
        exportedGetterSetters, getSetOverrides);
    
    // 2. 开始生成文件
    stringBuilder.StartGlueFile(classObj, nullableEnabled: nullableEnabled);
    stringBuilder.AppendTooltip(classObj);
    
    // 3. 添加特性
    AttributeBuilder attributeBuilder = new AttributeBuilder(classObj);
    if (classObj.ClassFlags.HasAnyFlags(EClassFlags.Abstract))
    {
        attributeBuilder.AddArgument("ClassFlags.Abstract");
    }
    attributeBuilder.AddGeneratedTypeAttribute(classObj);
    attributeBuilder.Finish();
    
    // 4. 声明类
    stringBuilder.DeclareType(classObj, "class", classObj.GetStructName(), 
        superClassName, nativeInterfaces: interfaces);
    stringBuilder.AppendNativeTypePtr(classObj);
    
    // 5. 导出静态构造函数（缓存反射信息）
    StaticConstructorUtilities.ExportStaticConstructor(stringBuilder, classObj, 
        exportedProperties, exportedFunctions, ...);
    
    // 6. 导出属性和方法
    ExportClassProperties(stringBuilder, exportedProperties, exportedPropertyNames);
    ExportClassFunctions(classObj, stringBuilder, exportedFunctions, ...);
    ExportOverrides(stringBuilder, exportedOverrides, ...);
    
    // 7. 写入文件
    stringBuilder.CloseBrace();
    stringBuilder.EndGlueFile(classObj);
    FileExporter.SaveGlueToDisk(classObj, stringBuilder);
}
```

**生成的代码示例**：

```csharp
// AActor.generated.cs (简化版)
using UnrealSharp.Core;
using UnrealSharp.Interop;
using static UnrealSharp.Interop.FPropertyCallbacks;

namespace UnrealEngine.Engine;

[UClass]
public partial class AActor : UObject
{
    static readonly IntPtr NativeClassPtr = 
        CoreUObjectCallbacks.CallGetType(..., "AActor");
    
    // 缓存的反射信息
    static IntPtr K2_DestroyActor_NativeFunction;
    static int K2_DestroyActor_ParamsSize;
    
    // 静态构造函数
    static AActor()
    {
        K2_DestroyActor_NativeFunction = 
            UClassCallbacks.CallGetNativeFunctionFromClassAndName(
                NativeClassPtr, "K2_DestroyActor");
        K2_DestroyActor_ParamsSize = 
            UFunctionCallbacks.CallGetNativeFunctionParamsSize(
                K2_DestroyActor_NativeFunction);
    }
    
    // 生成的属性
    public bool bReplicates
    {
        get => BoolMarshaller.FromNative(NativeObject + bReplicates_Offset, 0);
        set => BoolMarshaller.ToNative(NativeObject + bReplicates_Offset, 0, value);
    }
    
    // 生成的方法
    public void K2_DestroyActor()
    {
        byte* paramsBuffer = stackalloc byte[K2_DestroyActor_ParamsSize];
        UFunctionCallbacks.CallInvokeFunction(
            NativeObject, K2_DestroyActor_NativeFunction, paramsBuffer);
    }
}
```

## PropertyTranslator：类型翻译系统

### 设计理念

`PropertyTranslator` 是类型映射的核心抽象：

```csharp
public abstract class PropertyTranslator
{
    // 支持的使用场景
    private readonly EPropertyUsageFlags _supportedPropertyUsage;
    
    public bool IsSupportedAsProperty => _supportedPropertyUsage.HasFlag(EPropertyUsageFlags.Property);
    public bool IsSupportedAsParameter => _supportedPropertyUsage.HasFlag(EPropertyUsageFlags.Parameter);
    public bool IsSupportedAsReturnValue => _supportedPropertyUsage.HasFlag(EPropertyUsageFlags.ReturnValue);
    
    // 是否可直接内存拷贝
    public virtual bool IsBlittable => false;
    
    // 核心抽象方法
    public abstract bool CanExport(UhtProperty property);
    public abstract string GetManagedType(UhtProperty property);
    public abstract string GetMarshaller(UhtProperty property);
    
    // Marshalling 方法
    public abstract void ExportFromNative(GeneratorStringBuilder builder, ...);
    public abstract void ExportToNative(GeneratorStringBuilder builder, ...);
}
```

### PropertyTranslatorManager：翻译器注册表

系统预置了大量翻译器，按优先级链式匹配：

```csharp
static PropertyTranslatorManager()
{
    // 基础类型 - Blittable（可直接内存拷贝）
    AddBlittablePropertyTranslator(typeof(UhtInt8Property), "sbyte");
    AddBlittablePropertyTranslator(typeof(UhtInt16Property), "short");
    AddBlittablePropertyTranslator(typeof(UhtInt64Property), "long");
    AddBlittablePropertyTranslator(typeof(UhtDoubleProperty), "double");
    
    // 特殊处理类型
    AddPropertyTranslator(typeof(UhtFloatProperty), new FloatPropertyTranslator());
    AddPropertyTranslator(typeof(UhtBoolProperty), new BoolPropertyTranslator());
    
    // 字符串类型
    AddPropertyTranslator(typeof(UhtStrProperty), new StringPropertyTranslator());
    AddPropertyTranslator(typeof(UhtNameProperty), new NamePropertyTranslator());
    AddPropertyTranslator(typeof(UhtTextProperty), new TextPropertyTranslator());
    
    // 对象引用类型
    AddPropertyTranslator(typeof(UhtObjectProperty), new ObjectPropertyTranslator());
    AddPropertyTranslator(typeof(UhtWeakObjectPtrProperty), new WeakObjectPropertyTranslator());
    AddPropertyTranslator(typeof(UhtInterfaceProperty), new InterfacePropertyTranslator());
    
    // 容器类型
    AddPropertyTranslator(typeof(UhtArrayProperty), new ContainerPropertyTranslator(
        "ArrayCopyMarshaller", "ArrayReadOnlyMarshaller", "ArrayMarshaller",
        "IReadOnlyList", "IList"));
    AddPropertyTranslator(typeof(UhtMapProperty), new ContainerPropertyTranslator(...));
    AddPropertyTranslator(typeof(UhtSetProperty), new ContainerPropertyTranslator(...));
    
    // 结构体类型 - 最后匹配（优先级最低）
    AddPropertyTranslator(typeof(UhtStructProperty), new BlittableStructPropertyTranslator());
    AddPropertyTranslator(typeof(UhtStructProperty), new StructPropertyTranslator());
}

public static PropertyTranslator? GetTranslator(this UhtProperty property)
{
    if (!RegisteredTranslators.TryGetValue(property.GetType(), out var translators))
        return null;
    
    foreach (var translator in translators)
    {
        if (translator.CanExport(property))
            return translator;  // 返回第一个匹配的
    }
    return null;
}
```

### 翻译器示例：StringPropertyTranslator

```csharp
public class StringPropertyTranslator : PropertyTranslator
{
    public StringPropertyTranslator() : base(EPropertyUsageFlags.Any) { }
    
    public override bool CacheProperty => true;  // 需要缓存属性指针
    
    public override bool CanExport(UhtProperty property) => property is UhtStrProperty;
    
    public override string GetManagedType(UhtProperty property) => "string";
    
    public override string GetMarshaller(UhtProperty property) => "StringMarshaller";
    
    public override string GetNullValue(UhtProperty property) => "\"\"";
    
    // 从原生内存读取
    public override void ExportFromNative(GeneratorStringBuilder builder, 
        UhtProperty property, string propertyName,
        string assignmentOrReturn, string sourceBuffer, string offset, ...)
    {
        builder.AppendLine($"IntPtr {propertyName}_NativePtr = {sourceBuffer} + {offset};");
        builder.AppendLine($"{assignmentOrReturn} StringMarshaller.FromNative({propertyName}_NativePtr, 0);");
    }
    
    // 写入原生内存
    public override void ExportToNative(GeneratorStringBuilder builder, 
        UhtProperty property, string propertyName,
        string destinationBuffer, string offset, string source, ...)
    {
        builder.AppendLine($"IntPtr {propertyName}_NativePtr = {destinationBuffer} + {offset};");
        builder.AppendLine($"StringMarshaller.ToNative({propertyName}_NativePtr, 0, {source});");
    }
}
```

### 自定义类型配置

通过 JSON 配置文件可以扩展类型映射：

```json
// Config/DefaultEngine.UnrealSharpTypes.json
{
  "Structs": {
    "BlittableTypes": [
      {
        "Name": "FVector",
        "ManagedType": "System.Numerics.Vector3"
      },
      {
        "Name": "FRotator",
        "ManagedType": "UnrealSharp.Core.FRotator"
      }
    ],
    "NativelyTranslatableTypes": [
      {
        "Name": "FHitResult",
        "HasDestructor": true
      }
    ],
    "CustomTypes": [
      "FReplicationEntry"  // 跳过生成
    ]
  }
}
```

## FunctionExporter：函数导出系统

### 函数类型分类

```csharp
public enum FunctionType
{
    Normal,                    // 普通方法
    BlueprintEvent,           // 蓝图事件
    ExtensionOnAnotherClass,  // 扩展方法
    InternalWhitelisted,      // 内部白名单方法
    GetterSetter,             // Getter/Setter
}
```

### 函数导出流程

```csharp
public class FunctionExporter
{
    public void Initialize(OverloadMode overloadMode, EFunctionProtectionMode protectionMode)
    {
        // 1. 确定函数名和保护级别
        FunctionName = protectionMode != EFunctionProtectionMode.OverrideWithInternal
            ? Function.GetFunctionName()
            : Function.SourceName;
        
        // 2. 为每个参数获取翻译器
        ParameterTranslators = new List<PropertyTranslator>(Function.Children.Count);
        foreach (UhtProperty parameter in Function.Properties)
        {
            PropertyTranslator translator = parameter.GetTranslator()!;
            ParameterTranslators.Add(translator);
            if (!translator.IsBlittable) isBlittable = false;
        }
        
        // 3. 确定调用方式
        InvokeFunction = DetermineInvokeFunction();
        
        if (Function.HasAllFlags(EFunctionFlags.Static))
        {
            Modifiers += "static ";
            InvokeFirstArgument = "NativeClassPtr";
        }
        else
        {
            InvokeFirstArgument = "NativeObject";
        }
    }
}
```

### 默认参数处理

UE 支持 C++ 默认参数，C# 需要生成重载：

```csharp
// C++ 声明
UFUNCTION(BlueprintCallable)
void MyFunction(int A, int B = 10, FString C = "default");

// 生成的 C# 代码
public void MyFunction(int A, int B, string C)
{
    // 主方法实现
    byte* paramsBuffer = stackalloc byte[MyFunction_ParamsSize];
    // ... marshalling and invoke
}

public void MyFunction(int A, int B) => MyFunction(A, B, "default");
public void MyFunction(int A) => MyFunction(A, 10, "default");
```

### 扩展方法支持

`UBlueprintFunctionLibrary` 的静态方法会生成为扩展方法：

```csharp
// C++
UCLASS()
class UMyBlueprintFunctionLibrary : public UBlueprintFunctionLibrary
{
    UFUNCTION(BlueprintCallable)
    static void MyHelper(AActor* Target, int Value);
};

// 生成的 C#
public static class AActor_Extensions
{
    public static void MyHelper(this AActor Target, int Value)
    {
        // 调用静态方法，第一个参数是 Target
    }
}
```

## GeneratorStringBuilder：代码生成工具

### 设计模式

`GeneratorStringBuilder` 提供流畅的代码生成接口：

```csharp
public class GeneratorStringBuilder : IDisposable
{
    private int _indent;
    private readonly BorrowStringBuilder _borrower = new(StringBuilderCache.Big);
    
    public void OpenBrace()
    {
        AppendLine("{");
        Indent();
    }
    
    public void CloseBrace()
    {
        UnIndent();
        AppendLine("}");
    }
    
    public void AppendLine(string line)
    {
        AppendLine();  // 添加缩进
        StringBuilder.Append(line);
    }
    
    public void DeclareDirective(string directive)
    {
        if (!_directives.Contains(directive))
        {
            _directives.Add(directive);
            AppendLine($"using {directive};");
        }
    }
}
```

### 文件生成模板

```csharp
public static void StartGlueFile(this GeneratorStringBuilder stringBuilder, 
    UhtField type, bool blittable = false, bool nullableEnabled = false)
{
    if (nullableEnabled)
    {
        stringBuilder.AppendLine("#nullable enable");
    }
    
    // 添加标准 using
    stringBuilder.DeclareDirective(ScriptGeneratorUtilities.AttributeNamespace);
    stringBuilder.DeclareDirective(ScriptGeneratorUtilities.CoreNamespace);
    stringBuilder.DeclareDirective(ScriptGeneratorUtilities.InteropNamespace);
    stringBuilder.DeclareDirective(ScriptGeneratorUtilities.MarshallerNamespace);
    
    stringBuilder.AppendLine($"using static UnrealSharp.Interop.{ExporterCallbacks.FPropertyCallbacks};");
    
    if (blittable)
    {
        stringBuilder.DeclareDirective(ScriptGeneratorUtilities.InteropServicesNamespace);
    }
    
    stringBuilder.AppendLine();
    stringBuilder.AppendLine($"namespace {type.GetNamespace()};");
}
```

## Source Generators：Roslyn 增强

除了 UHT 插件，UnrealSharp 还提供了 Roslyn Source Generator：

### CustomLogSourceGenerator

自动为标记 `[CustomLog]` 的类生成日志方法：

```csharp
[Generator]
public class CustomLogSourceGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var discoveryResults = context.SyntaxProvider
            .ForAttributeWithMetadataName("UnrealSharp.Log.CustomLog", 
                Predicate, Transform);
        context.RegisterSourceOutput(discoveryResults, GenerateSource);
    }
    
    private void GenerateSource(SourceProductionContext context, ClassLogInfo info)
    {
        var builder = new StringBuilder();
        builder.AppendLine($"namespace {info.Namespace};");
        builder.AppendLine($"public partial class {info.Name}");
        builder.AppendLine("{");
        builder.AppendLine($"    public static void Log(string message) => UnrealLogger.Log(\"{info.Name}\", message, ...);");
        builder.AppendLine($"    public static void LogWarning(string message) => ...;");
        builder.AppendLine($"    public static void LogError(string message) => ...;");
        builder.AppendLine("}");
        
        context.AddSource($"{info.Name}_CustomLog.g.cs", ...);
    }
}
```

**使用示例**：

```csharp
[CustomLog]
public static partial class LogMyGame { }

// 自动生成：
// LogMyGame.Log("Hello");
// LogMyGame.LogWarning("Warning!");
// LogMyGame.LogError("Error!");
```

### NativeCallbacksWrapperGenerator

为委托类型生成回调包装器（详见第四篇）。

## GlueModuleFactory：项目文件管理

动态创建和维护 Glue 项目文件：

```csharp
public static void CreateGlueProjects()
{
    foreach (ModuleInfo moduleInfo in ModuleUtilities.PackageToModuleInfo.Values)
    {
        if (!moduleInfo.Module.ShouldExportPackage() || moduleInfo.IsPartOfEngine)
            continue;
        
        CreateOrUpdateGlueModule(
            moduleInfo.CsProjPath, 
            moduleInfo.ProjectName, 
            moduleInfo.Dependencies, 
            moduleInfo.ModuleRoot, 
            moduleInfo.Module, 
            out bool createdNewModule);
    }
    
    // 如果有新模块或解决方案不存在，生成解决方案
    if (anyChanges || !File.Exists(GeneratorStatics.ManagedSolutionPath))
    {
        USharpBuildToolUtilities.InvokeUSharpBuildTool("GenerateSolution");
    }
}
```

## 文件清理与增量更新

### FileExporter 清理策略

```csharp
public static void CleanOldExportedFiles()
{
    foreach (ModuleInfo plugin in ModuleUtilities.PackageToModuleInfo.Values)
    {
        CleanOldFilesInDirectories(plugin.GlueModuleDirectory);
    }
}

private static void CleanOldFilesInDirectories(string path)
{
    foreach (string file in Directory.GetFiles(path))
    {
        if (!AffectedFiles.Contains(file))  // 不在本次生成列表中的文件
        {
            File.Delete(file);  // 删除过时的生成文件
        }
    }
}
```

### PackageHeadersTracker：增量追踪

追踪每个头文件的修改时间，实现增量编译：

```csharp
// 只重新生成变化的头文件
if (!generatedGlueFolderExists || 
    PackageHeadersTracker.HasHeaderFileChanged(packageName, child.HeaderFile))
{
    ForEachChild(child, ExportType);
}
else
{
    ForEachChild(child, FileExporter.AddUnchangedType);
}
```

## 总结

Glue 代码生成系统是 UnrealSharp 的核心基础设施：

| 组件 | 职责 | 关键技术 |
|-----|------|---------|
| `CSharpExporter` | 导出流程控制 | 增量生成、并行任务 |
| `ClassExporter` | 类导出 | 特性生成、静态构造函数 |
| `PropertyTranslator` | 类型映射 | 策略模式、链式匹配 |
| `FunctionExporter` | 函数导出 | 参数重载、扩展方法 |
| `GeneratorStringBuilder` | 代码生成 | 缩进管理、模板方法 |
| `Source Generators` | 编译时增强 | Roslyn API |

**设计亮点**：
1. **增量生成**：只重新生成变化的头文件，大幅缩短编译时间
2. **插件化翻译器**：通过 JSON 配置扩展类型映射，无需修改源码
3. **双系统协同**：UHT 插件生成基础绑定，Source Generator 增强托管代码
4. **内存优化**：`StringBuilderCache` 复用、避免不必要的字符串分配

下一篇文章我们将深入 **Marshalling 系统**，看看数据是如何在 C++ 和 C# 之间安全传递的。

---

**系列导航**：
- [第一篇：概述与架构总览](/posts/UnrealSharp-01-概述与架构总览/)
- [第二篇：运行时初始化](/posts/UnrealSharp-02-运行时初始化/)
- [第三篇：GCHandle与对象生命周期](/posts/UnrealSharp-03-GCHandle与对象生命周期/)
- [第四篇：托管回调系统](/posts/UnrealSharp-04-托管回调系统/)
- 第五篇：Glue代码生成系统（本文）
