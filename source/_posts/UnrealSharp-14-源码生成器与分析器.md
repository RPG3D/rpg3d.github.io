---
title: UnrealSharp 插件技术深度解析 - 源码生成器与分析器
date: 2026-03-30 19:00:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, C#, Roslyn, Source Generator]
series: UnrealSharp 插件技术深度解析
series_number: 14
---

# 编译时魔法 - Source Generators 与 Roslyn Analyzers

> **技术深度**：⭐⭐⭐⭐⭐
> **前置知识**：Roslyn 编译器平台、Source Generators、代码分析

---

## 引言

UnrealSharp 利用 Roslyn 编译器平台实现了强大的编译时代码生成和分析能力。本章将深入剖析 Source Generators 和 Analyzers 的实现原理，帮助你理解并扩展这些编译时魔法。

---

## 一、Source Generators 基础

### 1.1 什么是 Source Generator？

Source Generator 是 Roslyn 提供的编译时代码生成机制：

```
源码 → 编译 → Source Generator → 生成代码 → 继续编译 → 程序集
```

**优势**：
- 编译时执行，零运行时开销
- 类型安全，编译时检查
- 与 IDE 集成，智能感知支持
- 减少样板代码

### 1.2 UnrealSharp 的 Source Generators

```
Managed/UnrealSharp/UnrealSharp.SourceGenerators/
├── NativeCallbacksWrapperGenerator.cs
├── CustomLogGenerator.cs
├── GeneratorUtilities.cs
└── UnrealSharp.SourceGenerators.csproj
```

---

## 二、NativeCallbacksWrapperGenerator 深度分析

### 2.1 功能概述

`NativeCallbacksWrapperGenerator` 自动生成 C# 调用 C++ 函数的包装代码：

```csharp
// 用户定义
[NativeCallbacks]
public static unsafe partial class UObjectExporter
{
    public static delegate* unmanaged<IntPtr, FName> GetName;
}

// 自动生成
public static unsafe partial class UObjectExporter
{
    static UObjectExporter()
    {
        GetName = (delegate* unmanaged<IntPtr, FName>)
            NativeBinds.TryGetBoundFunction("UObjectExporter", "GetName", sizeof(IntPtr) + sizeof(FName));
    }
    
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static FName CallGetName(IntPtr nativeObject)
    {
        return GetName(nativeObject);
    }
}
```

### 2.2 实现详解

```csharp
// Source/UnrealSharp/UnrealSharp.SourceGenerators/NativeCallbacksWrapperGenerator.cs

[Generator]
public class NativeCallbacksWrapperGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 1. 创建语法提供器：过滤带有 [NativeCallbacks] 特性的类
        var classDeclarations = context.SyntaxProvider.CreateSyntaxProvider(
            static (syntaxNode, _) => 
                syntaxNode is ClassDeclarationSyntax cds && 
                cds.AttributeLists.Count > 0,
            static (syntaxContext, _) => GetClassInfoOrNull(syntaxContext));

        // 2. 与编译信息结合
        var classAndCompilation = classDeclarations.Combine(context.CompilationProvider);

        // 3. 注册源码输出
        context.RegisterSourceOutput(classAndCompilation, (spc, pair) =>
        {
            var maybeClassInfo = pair.Left;
            var compilation = pair.Right;
            if (!maybeClassInfo.HasValue)
                return;
                
            GenerateForClass(spc, compilation, maybeClassInfo.Value);
        });
    }
    
    private static ClassInfo? GetClassInfoOrNull(GeneratorSyntaxContext context)
    {
        if (context.Node is not ClassDeclarationSyntax classDeclaration)
            return null;

        // 检查特性
        bool hasNativeCallbacksAttribute = classDeclaration.AttributeLists
            .SelectMany(a => a.Attributes)
            .Any(a => a.Name.ToString() is "NativeCallbacks" or "NativeCallbacksAttribute");

        if (!hasNativeCallbacksAttribute)
            return null;

        // 提取类信息
        string namespaceName = classDeclaration.GetFullNamespace();
        if (string.IsNullOrEmpty(namespaceName))
            return null;

        var classInfo = new ClassInfo
        {
            ClassDeclaration = classDeclaration,
            Name = classDeclaration.Identifier.ValueText,
            Namespace = namespaceName,
            Delegates = new List<DelegateInfo>(),
            NullableAwareable = context.SemanticModel.GetNullableContext(
                context.Node.Span.Start).HasFlag(NullableContext.AnnotationsEnabled)
        };

        // 提取委托字段
        foreach (MemberDeclarationSyntax member in classDeclaration.Members)
        {
            if (member is not FieldDeclarationSyntax fieldDeclaration ||
                fieldDeclaration.Declaration.Type is not FunctionPointerTypeSyntax functionPointerType)
                continue;

            var delegateInfo = ParseFunctionPointer(functionPointerType);
            classInfo.Delegates.Add(delegateInfo);
        }

        return classInfo;
    }
    
    private static void GenerateForClass(
        SourceProductionContext context, 
        Compilation compilation, 
        ClassInfo classInfo)
    {
        var sourceBuilder = new StringBuilder();

        // 收集需要的命名空间
        HashSet<string> namespaces = CollectNamespaces(classInfo, compilation);

        // 生成文件头
        if (classInfo.NullableAwareable)
            sourceBuilder.AppendLine("#nullable enable");
        else
            sourceBuilder.AppendLine("#nullable disable");
            
        sourceBuilder.AppendLine("#pragma warning disable CS8500, CS0414");
        sourceBuilder.AppendLine();

        foreach (string ns in namespaces)
            sourceBuilder.AppendLine($"using {ns};");

        sourceBuilder.AppendLine();
        sourceBuilder.AppendLine($"namespace {classInfo.Namespace}");
        sourceBuilder.AppendLine("{");
        sourceBuilder.AppendLine($"    public static unsafe partial class {classInfo.Name}");
        sourceBuilder.AppendLine("    {");

        // 生成静态构造函数
        GenerateStaticConstructor(sourceBuilder, classInfo);

        // 生成调用方法
        foreach (var delegateInfo in classInfo.Delegates)
        {
            GenerateCallMethod(sourceBuilder, delegateInfo, compilation);
        }

        sourceBuilder.AppendLine("    }");
        sourceBuilder.AppendLine("}");

        // 添加生成的源码
        context.AddSource($"{classInfo.Name}.generated.cs", 
            SourceText.From(sourceBuilder.ToString(), Encoding.UTF8));
    }
    
    private static void GenerateStaticConstructor(StringBuilder sb, ClassInfo classInfo)
    {
        sb.AppendLine("        static " + classInfo.Name + "()");
        sb.AppendLine("        {");

        foreach (var delegateInfo in classInfo.Delegates)
        {
            // 计算参数大小
            int totalSize = CalculateParametersSize(delegateInfo);
            
            // 获取函数指针
            sb.AppendLine($"             IntPtr {delegateInfo.Name}FuncPtr = " +
                         $"UnrealSharp.Binds.NativeBinds.TryGetBoundFunction(" +
                         $"\"{classInfo.Name}\", \"{delegateInfo.Name}\", {totalSize});");
            
            // 转换为函数指针
            sb.Append($"             {delegateInfo.Name} = (delegate* unmanaged<");
            sb.Append(string.Join(", ", delegateInfo.Parameters.Select(p => 
                GetParameterSignature(p))));
            if (delegateInfo.Parameters.Count > 0)
                sb.Append(", ");
            sb.Append($"{delegateInfo.ReturnValue.Type}>){delegateInfo.Name}FuncPtr;");
            sb.AppendLine();
        }

        sb.AppendLine("        }");
    }
}
```

### 2.3 生成的代码示例

**输入**：

```csharp
[NativeCallbacks]
public static unsafe partial class FStringExporter
{
    public static delegate* unmanaged<FString*, char*, void> MarshalToNative;
}
```

**输出**：

```csharp
#nullable disable
#pragma warning disable CS8500, CS0414

namespace UnrealSharp.Interop
{
    public static unsafe partial class FStringExporter
    {
        static FStringExporter()
        {
            int MarshalToNativeTotalSize = sizeof(FString*) + sizeof(char*);
            IntPtr MarshalToNativeFuncPtr = UnrealSharp.Binds.NativeBinds.TryGetBoundFunction(
                "FStringExporter", "MarshalToNative", MarshalToNativeTotalSize);
            MarshalToNative = (delegate* unmanaged<FString*, char*, void>)MarshalToNativeFuncPtr;
        }
        
        [System.Runtime.CompilerServices.MethodImpl(
            System.Runtime.CompilerServices.MethodImplOptions.AggressiveInlining)]
        public static void CallMarshalToNative(FString* nativeString, char* managedString)
        {
            MarshalToNative(nativeString, managedString);
        }
    }
}
```

---

## 三、CustomLogGenerator

### 3.1 功能概述

自动生成日志类，简化日志输出：

```csharp
// 用户定义
[LogCategory("MyGame")]
public static partial class LogMyGame;

// 自动生成
public static partial class LogMyGame
{
    public static void Log(string message) => FMsg.Log("MyGame", ELogVerbosity.Log, message);
    public static void Warning(string message) => FMsg.Log("MyGame", ELogVerbosity.Warning, message);
    public static void Error(string message) => FMsg.Log("MyGame", ELogVerbosity.Error, message);
}
```

### 3.2 实现

```csharp
[Generator]
public class CustomLogGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var logClasses = context.SyntaxProvider.CreateSyntaxProvider(
            static (node, _) => node is ClassDeclarationSyntax { AttributeLists.Count: > 0 },
            static (ctx, _) => GetLogClassInfo(ctx));

        context.RegisterSourceOutput(logClasses, GenerateLogClass);
    }

    private static LogClassInfo? GetLogClassInfo(GeneratorSyntaxContext context)
    {
        if (context.Node is not ClassDeclarationSyntax classDecl)
            return null;

        // 查找 [LogCategory] 特性
        var attribute = classDecl.AttributeLists
            .SelectMany(a => a.Attributes)
            .FirstOrDefault(a => a.Name.ToString() == "LogCategory");

        if (attribute == null || attribute.ArgumentList?.Arguments.Count != 1)
            return null;

        var categoryArg = attribute.ArgumentList.Arguments[0];
        var categoryValue = context.SemanticModel.GetConstantValue(categoryArg.Expression);

        if (categoryValue.Value is not string categoryName)
            return null;

        return new LogClassInfo
        {
            ClassName = classDecl.Identifier.ValueText,
            Namespace = classDecl.GetFullNamespace(),
            CategoryName = categoryName
        };
    }

    private static void GenerateLogClass(SourceProductionContext context, LogClassInfo info)
    {
        var sb = new StringBuilder();
        
        sb.AppendLine($"namespace {info.Namespace}");
        sb.AppendLine("{");
        sb.AppendLine($"    public static partial class {info.ClassName}");
        sb.AppendLine("    {");
        
        // 生成方法
        sb.AppendLine($"        public static void Log(string message) => " +
                     $"UnrealSharp.Core.FMsg.Log(\"{info.CategoryName}\", " +
                     $"UnrealSharp.Core.ELogVerbosity.Log, message);");
        sb.AppendLine($"        public static void Warning(string message) => " +
                     $"UnrealSharp.Core.FMsg.Log(\"{info.CategoryName}\", " +
                     $"UnrealSharp.Core.ELogVerbosity.Warning, message);");
        sb.AppendLine($"        public static void Error(string message) => " +
                     $"UnrealSharp.Core.FMsg.Log(\"{info.CategoryName}\", " +
                     $"UnrealSharp.Core.ELogVerbosity.Error, message);");
        sb.AppendLine($"        public static void Fatal(string message) => " +
                     $"UnrealSharp.Core.FMsg.Log(\"{info.CategoryName}\", " +
                     $"UnrealSharp.Core.ELogVerbosity.Fatal, message);");
        
        sb.AppendLine("    }");
        sb.AppendLine("}");

        context.AddSource($"{info.ClassName}.generated.cs", 
            SourceText.From(sb.ToString(), Encoding.UTF8));
    }
}
```

---

## 四、Roslyn Analyzers

### 4.1 什么是 Roslyn Analyzer？

Roslyn Analyzer 是编译时代码分析工具，可以：
- 检测代码问题
- 提供代码修复建议
- 强制执行编码规范

### 4.2 UnrealSharp 分析器示例

```csharp
// 检测 UClass 中是否错误使用了非 UProperty 属性
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class UClassPropertyAnalyzer : DiagnosticAnalyzer
{
    public const string DiagnosticId = "US001";
    
    private static readonly DiagnosticDescriptor Rule = new(
        DiagnosticId,
        "Non-UProperty in UClass",
        "Property '{0}' in UClass must be marked with [UProperty]",
        "Usage",
        DiagnosticSeverity.Warning,
        true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics 
        => ImmutableArray.Create(Rule);

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterSymbolAction(AnalyzeProperty, SymbolKind.Property);
    }

    private static void AnalyzeProperty(SymbolAnalysisContext context)
    {
        var property = (IPropertySymbol)context.Symbol;
        
        // 检查是否在 UClass 中
        var containingType = property.ContainingType;
        if (!HasUClassAttribute(containingType))
            return;

        // 检查是否有 UProperty 特性
        if (HasUPropertyAttribute(property))
            return;

        // 报告诊断
        var diagnostic = Diagnostic.Create(Rule, 
            property.Locations[0], 
            property.Name);
        context.ReportDiagnostic(diagnostic);
    }
}
```

### 4.3 代码修复提供器

```csharp
[ExportCodeFixProvider(LanguageNames.CSharp, Name = nameof(UPropertyCodeFixProvider))]
public class UPropertyCodeFixProvider : CodeFixProvider
{
    public override ImmutableArray<string> FixableDiagnosticIds 
        => ImmutableArray.Create(UClassPropertyAnalyzer.DiagnosticId);

    public override async Task RegisterCodeFixesAsync(CodeFixContext context)
    {
        var root = await context.Document.GetSyntaxRootAsync(context.CancellationToken);
        var diagnostic = context.Diagnostics.First();
        var diagnosticSpan = diagnostic.Location.SourceSpan;

        var propertyDecl = root.FindToken(diagnosticSpan.Start)
            .Parent.AncestorsAndSelf()
            .OfType<PropertyDeclarationSyntax>()
            .First();

        context.RegisterCodeFix(
            CodeAction.Create(
                "Add [UProperty] attribute",
                c => AddUPropertyAttribute(context.Document, propertyDecl, c),
                "Add UProperty"),
            diagnostic);
    }

    private async Task<Document> AddUPropertyAttribute(
        Document document, 
        PropertyDeclarationSyntax propertyDecl, 
        CancellationToken cancellationToken)
    {
        var root = await document.GetSyntaxRootAsync(cancellationToken);
        
        // 添加 [UProperty] 特性
        var newAttribute = SyntaxFactory.Attribute(SyntaxFactory.IdentifierName("UProperty"));
        var newAttributeList = SyntaxFactory.AttributeList(
            SyntaxFactory.SingletonSeparatedList(newAttribute));
        
        var newPropertyDecl = propertyDecl.AddAttributeLists(newAttributeList);
        var newRoot = root.ReplaceNode(propertyDecl, newPropertyDecl);
        
        return document.WithSyntaxRoot(newRoot);
    }
}
```

---

## 五、如何扩展

### 5.1 创建自定义 Source Generator

```csharp
// 1. 创建项目
// UnrealSharp.SourceGenerators.csproj
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <IncludeBuildOutput>false</IncludeBuildOutput>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.0" />
  </ItemGroup>
</Project>

// 2. 实现 IIncrementalGenerator
[Generator]
public class MyCustomGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 实现逻辑
    }
}

// 3. 注册到目标项目
// 在 .csproj 中添加
<ItemGroup>
  <ProjectReference Include="path/to/UnrealSharp.SourceGenerators.csproj"
                    OutputItemType="Analyzer"
                    ReferenceOutputAssembly="false" />
</ItemGroup>
```

### 5.2 创建自定义 Analyzer

```csharp
// 1. 定义诊断规则
private static readonly DiagnosticDescriptor MyRule = new(
    "US100",
    "My custom rule",
    "Description of the issue",
    "Usage",
    DiagnosticSeverity.Info,
    true);

// 2. 注册分析
public override void Initialize(AnalysisContext context)
{
    context.RegisterSyntaxNodeAction(AnalyzeMethodDeclaration, SyntaxKind.MethodDeclaration);
}

// 3. 实现分析逻辑
private void AnalyzeMethodDeclaration(SyntaxNodeAnalysisContext context)
{
    var methodDecl = (MethodDeclarationSyntax)context.Node;
    // 分析逻辑...
}
```

---

## 六、调试 Source Generators

### 6.1 输出生成的代码

```xml
<!-- 在 .csproj 中添加 -->
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
  <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>
```

生成的代码将输出到 `Generated` 目录。

### 6.2 调试器附加

```csharp
// 在 Generator 中添加断点
if (!Debugger.IsAttached)
{
    Debugger.Launch();
}
```

### 6.3 使用 Roslyn 属性

```csharp
[Generator(LanguageNames.CSharp)]
public class MyGenerator : IIncrementalGenerator
{
    // ...
}
```

---

## 七、性能考量

### 7.1 增量生成

```csharp
public void Initialize(IncrementalGeneratorInitializationContext context)
{
    // 使用增量提供器，只处理变化的文件
    var provider = context.SyntaxProvider.CreateSyntaxProvider(
        predicate: static (node, _) => IsTarget(node),
        transform: static (ctx, _) => Transform(ctx));
    
    context.RegisterSourceOutput(provider, Generate);
}
```

### 7.2 缓存编译信息

```csharp
// 缓存编译信息，避免重复计算
private static readonly ConcurrentDictionary<INamedTypeSymbol, bool> Cache = new();

private static bool IsUClass(INamedTypeSymbol type)
{
    return Cache.GetOrAdd(type, t => t.GetAttributes()
        .Any(a => a.AttributeClass?.Name == "UClassAttribute"));
}
```

---

## 八、总结

### 核心要点

1. **Source Generators** 在编译时生成代码，零运行时开销
2. **NativeCallbacksWrapperGenerator** 自动生成跨语言调用代码
3. **CustomLogGenerator** 简化日志类创建
4. **Roslyn Analyzers** 提供编译时代码分析
5. **增量生成** 确保编译性能

### 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    编译时处理流程                                 │
└─────────────────────────────────────────────────────────────────┘

源码 (.cs)
    │
    ├──→ Roslyn 编译器
    │        │
    │        ├──→ Syntax Tree 解析
    │        │
    │        ├──→ Source Generators
    │        │    ├── NativeCallbacksWrapperGenerator
    │        │    └── CustomLogGenerator
    │        │
    │        ├──→ Roslyn Analyzers
    │        │    ├── UClassPropertyAnalyzer
    │        │    └── ...
    │        │
    │        └──→ 生成代码 + 诊断
    │
    └──→ 程序集 (.dll)
```

### 关键源码文件

| 文件 | 职责 |
|------|------|
| `NativeCallbacksWrapperGenerator.cs` | 生成调用包装代码 |
| `CustomLogGenerator.cs` | 生成日志类 |
| `GeneratorUtilities.cs` | 生成器工具函数 |
| `UnrealSharp.SourceGenerators.csproj` | 项目配置 |

---

**下一篇**：总结与展望 - UnrealSharp 总结与 C# 游戏开发生态展望
