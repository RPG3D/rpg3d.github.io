---
title: UnrealSharp 插件技术深度解析 - 系列大纲
date: 2026-03-29 21:00:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, C#, CoreCLR, Mono, 反射系统]
---

# UnrealSharp 插件技术深度解析 - 博客大纲

> **作者**：GLM-5.0

## 系列前言

**目标读者画像**：
- 熟悉UE5反射系统、插件开发
- C#基础语法水平
- 有CoreCLR/Mono简单embed经验（能完成运行时初始化、程序集加载、方法调用）

**学习路径**：从已知 → 未知，从基础 → 进阶，从原理 → 实践

---

## 文章列表

### 第一篇：概述与架构总览

**标题**：UnrealSharp 插件全景图 - 当UE5遇见.NET

**内容要点**：
1. 插件定位与核心价值：为什么要在UE中使用C#
2. 整体架构图解：
   - C++原生层（UnrealSharpCore等模块）
   - C#托管层（UnrealSharp.Core等程序集）
   - 两层之间的桥梁
3. 模块划分速览（结合UE模块知识）
4. 数据流全景图：从编辑器编译到运行时调用
5. 与UE原生蓝图/ C++的对比

**技术深度**：⭐⭐ (概览级)

**关键源码**：
- `UnrealSharp.uplugin`
- `Source/` 目录下各模块的 `Build.cs`
- `Managed/UnrealSharp/` 目录结构

---

### 第二篇：运行时初始化 - CoreCLR与Mono的双重实现

**标题**：深入理解 .NET 运行时嵌入 - CoreCLR 与 Mono 的初始化之道

**内容要点**：
1. 从基础embed模式出发：回顾CoreCLR/Mono基础用法
2. UnrealSharp的运行时选择机制：
   - 编译宏 `UNREALSHARP_MONO` 的条件分支
   - 配置驱动：`DefaultEngine.ini` 中的 `bUseMono`
3. **CoreCLR集成详解**：
   - `hostfxr` 加载与函数指针获取
   - `InitializeNativeHost()` 实现分析
   - 运行时配置（runtimeconfig.json）
   - 入口点调用：`Main.InitializeUnrealSharp`
4. **Mono集成详解**：
   - `mono_jit_init_version()` 与域名管理
   - 平台适配：Windows/Mac的JIT模式，iOS的AOT+解释器模式
   - 程序集搜索路径设置
5. 两种运行时的对比与选择建议

**技术深度**：⭐⭐⭐ (从已知扩展)

**关键源码**：
- `Source/UnrealSharpCore/Private/CSManager.cpp`
- `Source/UnrealSharpCore/Private/CSMonoRuntime.cpp`
- `Managed/UnrealSharp/UnrealSharp.Plugins/Main.cs`

---

### 第三篇：GCHandle与对象生命周期管理

**标题**：跨越边界的引用 - GCHandle 与托管对象生命周期管理

**内容要点**：
1. 问题引入：为什么需要GCHandle？
2. **GCHandle 基础**：
   - 四种类型：Strong/Weak/Pinned/Null
   - 内存布局与使用场景
3. **UnrealSharp的封装**：
   - `FGCHandle` 结构体分析
   - `FGCHandleUtilities` 工具类
   - GCHandle的创建、使用、释放流程
4. **对象生命周期同步**：
   - UObject → C# 对象的映射表：`ManagedObjectHandles`
   - `FUObjectDeleteListener` 接口实现
   - UObject销毁时如何同步释放GCHandle
5. **跨边界传递实践**：
   - 如何传递复杂对象而不仅仅是基本类型
   - 托管对象创建的完整流程

**技术深度**：⭐⭐⭐⭐ (核心机制)

**关键源码**：
- `Source/UnrealSharpCore/Public/CSManagedGCHandle.h`
- `Managed/UnrealSharp/UnrealSharp.Core/GCHandleUtilities.cs`
- `Source/UnrealSharpCore/Public/CSManagedAssembly.h`

---

### 第四篇：托管回调系统 - C++调用C#的桥梁

**标题**：双向通信的基石 - 托管回调系统深度解析

**内容要点**：
1. 从`UnmanagedCallersOnly`说起
2. **回调注册流程**：
   - C#侧：`UnmanagedCallbacks.cs` 中的导出函数
   - C++侧：`FManagedCallbacks` 结构体
   - 初始化时的回调获取与缓存
3. **核心回调函数解析**：
   - `CreateNewManagedObject` - 创建托管对象
   - `InvokeManagedMethod` - 方法调用
   - `LookupManagedMethod` / `LookupManagedType` - 类型查找
   - `Dispose` - 资源释放
4. **调用约定与参数传递**：
   - Calling Convention选择
   - 参数封送基础
5. 实践：如何添加一个新的托管回调函数

**技术深度**：⭐⭐⭐⭐ (核心机制)

**关键源码**：
- `Managed/UnrealSharp/UnrealSharp.Core/UnmanagedCallbacks.cs`
- `Source/UnrealSharpCore/Public/CSManagedCallbacksCache.h`
- `Managed/UnrealSharp/UnrealSharp.Core/ManagedCallbacks.cs`

---

### 第五篇：Glue代码生成系统（上）- UHT插件的魔法

**标题**：自动化绑定的艺术 - Glue代码生成系统原理

**内容要点**：
1. **UHT与UBT的关系**：
   - UHT的工作流程
   - UBT插件扩展机制
   - `UnrealSharpManagedGlue` 如何注册为UBT插件
2. **代码生成入口**：
   - `CSharpExporter.StartExport()` 流程
   - 遍历所有模块和包的策略
3. **类型导出器总览**：
   - ClassExporter - 导出UClass
   - StructExporter - 导出UStruct
   - EnumExporter - 导出UEnum
   - InterfaceExporter - 导出UInterface
   - DelegateExporter - 导出委托
4. **生成产物分析**：
   - 生成的C#文件结构
   - 项目中的 `Script/SharpDemo.RuntimeGlue/` 目录解析

**技术深度**：⭐⭐⭐⭐ (核心机制)

**关键源码**：
- `Source/UnrealSharpManagedGlue/CSharpExporter.cs`
- `Source/UnrealSharpManagedGlue/Exporters/` 目录下各导出器

---

### 第六篇：Glue代码生成系统（下）- PropertyTranslator与类型映射

**标题**：类型系统的翻译官 - PropertyTranslator 深度剖析

**内容要点**：
1. **问题引入**：UE属性类型 → C#类型的映射挑战
2. **PropertyTranslator 设计模式**：
   - 抽象基类与具体实现
   - 工厂模式与策略模式的结合
3. **核心类型翻译器详解**：
   - `BlittableTypePropertyTranslator` - 直接内存拷贝
   - `BoolPropertyTranslator` - 位域处理的艺术
   - `ObjectPropertyTranslator` - UObject引用
   - `StructPropertyTranslator` - 嵌套结构体
   - `ContainerPropertyTranslator` 系列 - TArray/TMap/TSet
   - `StringPropertyTranslator` - FString/FText与C# string
4. **特殊类型处理**：
   - 委托类型（DelegatePropertyTranslator）
   - 软引用（SoftObjectPropertyTranslator）
   - 枚举与位掩码
5. 实践：如何扩展支持自定义类型

**技术深度**：⭐⭐⭐⭐⭐ (高级机制)

**关键源码**：
- `Source/UnrealSharpManagedGlue/PropertyTranslators/PropertyTranslator.cs`
- `Source/UnrealSharpManagedGlue/PropertyTranslators/` 目录下各翻译器

---

### 第七篇：UE5反射系统的动态类型注册

**标题**：运行时的类型魔法 - C#类型到UE反射系统的注册

**内容要点**：
1. **UE5反射系统回顾**：
   - UClass/UStruct/UEnum/UInterface
   - 反射数据的内存布局
   - 动态类型注册的可能性
2. **UnrealSharp的逆向绑定**：
   - 从C#类型定义 → UE反射类型
   - `FCSManagedTypeDefinition` 结构
   - `FCSTypeReferenceReflectionData` 反射数据格式
3. **类型编译器系列**：
   - `CSManagedClassCompiler` - 编译UClass
   - `CSManagedStructCompiler` - 编译UStruct
   - `CSManagedEnumCompiler` - 编译UEnum
   - `CSManagedInterfaceCompiler` - 编译UInterface
4. **注册流程详解**：
   - 程序集加载时的类型扫描
   - `[GeneratedType]` 特性的作用
   - 编译后的类型如何被UE系统识别
5. 实践：调试一个C#类被注册为UClass的完整过程

**技术深度**：⭐⭐⭐⭐⭐ (高级机制，结合UE反射知识)

**关键源码**：
- `Source/UnrealSharpCore/Public/CSManagedTypeDefinition.h`
- `Source/UnrealSharpCore/Public/Compilers/` 目录下所有编译器
- `Source/UnrealSharpCore/Public/ReflectionData/CSTypeReferenceReflectionData.h`

---

### 第八篇：原生函数导出系统 - C#调用C++的桥梁

**标题**：UNREALSHARP_FUNCTION 宏的秘密 - 原生函数导出机制

**内容要点**：
1. **问题引入**：C#如何调用UE的C++ API？
2. **UNREALSHARP_FUNCTION 宏解析**：
   - 宏定义展开
   - 函数注册机制
   - 与UE的 `UFUNCTION` 宏的对比
3. **Exporter 类体系**：
   - `UUObjectExporter` - UObject操作
   - `UUClassExporter` - UClass操作
   - `UFPropertyExporter` - 属性操作
   - `UFStringExporter` - 字符串操作
   - 更多专用Exporter...
4. **函数查找与调用流程**：
   - C#侧如何通过名称查找函数
   - 参数封送细节
   - 返回值处理
5. 实践：如何为UE API添加新的导出函数

**技术深度**：⭐⭐⭐⭐ (核心机制)

**关键源码**：
- `Source/UnrealSharpBinds/Public/CSBindsManager.h`
- `Source/UnrealSharpBinds/Private/CSBindsManager.cpp`
- `Source/UnrealSharpCore/Public/Export/` 目录下各Exporter

---

### 第九篇：方法调用的完整流程

**标题**：一个方法调用的前世今生 - 从C#到C++的完整旅程

**内容要点**：
1. **调用场景分类**：
   - C#调用C++原生函数
   - C++调用C#托管方法
   - 蓝图调用C#方法
   - C#重写蓝图可调用函数
2. **C# → C++ 调用链**：
   ```
   C# MethodCall()
     → Generated Glue Code
     → UNREALSHARP_FUNCTION 导出函数
     → UE 内部实现
     → 返回值封送
   ```
3. **C++ → C# 调用链**：
   ```
   C++ InvokeManagedMethod()
     → 查找托管方法 (LookupManagedMethod)
     → 参数准备与封送
     → mono_runtime_invoke / CoreCLR delegate
     → 返回值处理
   ```
4. **蓝图 ↔ C# 交互**：
   - 蓝图如何"看到"C#方法
   - C#如何实现蓝图可调用函数
5. **性能考量**：
   - 跨边界调用的开销
   - 优化策略

**技术深度**：⭐⭐⭐⭐⭐ (综合应用)

**关键源码**：
- 涉及前述所有核心模块的协作

---

### 第十篇：热重载与编辑器集成

**标题**：开发效率的倍增器 - 热重载与编辑器集成

**内容要点**：
1. **热重载原理**：
   - `AssemblyLoadContext` 的使用
   - 程序集卸载与重新加载
   - 状态保持策略
2. **编辑器集成**：
   - `UnrealSharpEditor` 模块功能
   - 编译触发机制
   - 蓝图编译上下文扩展
3. **开发工作流**：
   - 修改C#代码 → 自动编译 → 热重载
   - 与UE原生C++热重载的对比
4. **调试支持**：
   - C#调试器附加
   - 日志系统集成

**技术深度**：⭐⭐⭐⭐ (实战应用)

**关键源码**：
- `Source/UnrealSharpEditor/` 模块
- `Managed/UnrealSharp/UnrealSharp.Plugins/PluginLoader.cs`

---

### 第十一篇：异步编程与委托系统

**标题**：异步编程与事件系统 - 跨语言的协作模式

**内容要点**：
1. **异步支持**：
   - `UnrealSharpAsync` 模块
   - async/await 与 UE异步系统的对接
   - Latent Action的C#实现
2. **委托系统**：
   - 单播委托 vs 多播委托
   - C# delegate 与 UE Delegate的桥接
   - 事件订阅与触发
3. **委托封送详解**：
   - `SinglecastDelegatePropertyTranslator`
   - `MulticastDelegatePropertyTranslator`
   - 绑定C#方法到UE委托

**技术深度**：⭐⭐⭐⭐ (实战应用)

**关键源码**：
- `Source/UnrealSharpAsync/` 模块
- `Managed/UnrealSharp/UnrealSharp/Delegate.cs`
- `Source/UnrealSharpManagedGlue/PropertyTranslators/` 委托相关翻译器

---

### 第十二篇：完整案例分析 - 从零创建一个C# GameMode

**标题**：实战演练 - 创建你的第一个纯C# GameMode

**内容要点**：
1. **案例目标**：创建一个完整的C# GameMode，包含：
   - 自定义GameMode类
   - 自定义Pawn类
   - 重写BeginPlay、Tick等事件
   - 绑定输入事件
   - 与蓝图交互
2. **逐步实现**：
   - 第一步：C#类定义与特性标注
   - 第二步：编译与Glue代码生成
   - 第三步：UE类型注册验证
   - 第四步：蓝图配置
   - 第五步：运行测试
3. **关键代码解析**：
   - 结合项目中的 `AScriptGameMode.cs` 进行分析
4. **常见问题与调试**：
   - 类型未注册
   - 方法调用失败
   - 属性访问问题

**技术深度**：⭐⭐⭐ (综合实践)

**关键源码**：
- `Script/ManagedSharpDemo/AScriptGameMode.cs`
- `Script/SharpDemo.RuntimeGlue/` 生成的胶水代码

---

### 第十三篇：性能优化与最佳实践

**标题**：性能优化与最佳实践 - 让C#飞起来

**内容要点**：
1. **性能瓶颈分析**：
   - 跨边界调用开销
   - 装箱/拆箱成本
   - 字符串操作
2. **优化策略**：
   - 减少不必要的跨边界调用
   - 批量操作
   - 使用blittable类型
   - 对象池化
3. **内存管理最佳实践**：
   - GCHandle的合理使用
   - 避免内存泄漏
   - 大对象处理
4. **代码组织建议**：
   - 项目结构
   - 命名约定
   - 与C++代码的协作模式

**技术深度**：⭐⭐⭐⭐ (进阶优化)

---

### 第十四篇：源码生成器与Roslyn分析器

**标题**：编译时魔法 - Source Generators 与 Roslyn Analyzers

**内容要点**：
1. **Source Generators 基础**：
   - 什么是Source Generator
   - 编译时代码生成原理
2. **UnrealSharp中的应用**：
   - `NativeCallbacksWrapperGenerator` 分析
   - 自动生成的代码分析
3. **Roslyn Analyzers**：
   - 代码诊断与修复
   - UnrealSharp中的分析器规则
4. **如何扩展**：
   - 编写自定义Source Generator
   - 添加自定义Analyzer

**技术深度**：⭐⭐⭐⭐⭐ (高级主题)

**关键源码**：
- `Managed/UnrealSharp/UnrealSharp.SourceGenerators/`
- `Managed/UnrealSharp/UnrealSharp.Analyzers/`

---

### 第十五篇：总结与展望

**标题**：UnrealSharp 总结与 C# 游戏开发生态展望

**内容要点**：
1. **核心技术回顾**：
   - 架构设计总结
   - 关键技术点清单
   - 设计模式应用
2. **与类似方案对比**：
   - UnrealCLR
   - CSharpForUE
   - Godot C#
   - Unity C#
3. **局限性分析**：
   - 性能边界
   - 平台支持
   - 生态系统
4. **未来展望**：
   - .NET 8+ 的新特性应用
   - AOT编译优化
   - 社区发展方向
5. **学习资源推荐**

**技术深度**：⭐⭐ (总结级)

---

## 技术深度图例

| 星级 | 说明 |
|-----|------|
| ⭐⭐ | 概览级，适合建立整体认知 |
| ⭐⭐⭐ | 基础级，需要一定背景知识 |
| ⭐⭐⭐⭐ | 进阶级，深入核心机制 |
| ⭐⭐⭐⭐⭐ | 专家级，需要多领域知识综合 |

---

## 读者知识图谱

```
                    ┌─────────────────────────────────────┐
                    │         读者已有知识                │
                    │  ┌─────────────────────────────┐   │
                    │  │ UE5 反射系统                │   │
                    │  │ UClass/UStruct/UEnum       │   │
                    │  │ 动态类型注册               │   │
                    │  └─────────────────────────────┘   │
                    │  ┌─────────────────────────────┐   │
                    │  │ UE5 插件开发                │   │
                    │  │ UBT/UHT 工作流程           │   │
                    │  │ 模块化架构                 │   │
                    │  └─────────────────────────────┘   │
                    │  ┌─────────────────────────────┐   │
                    │  │ CoreCLR/Mono Embed基础      │   │
                    │  │ 运行时初始化               │   │
                    │  │ 程序集加载                 │   │
                    │  │ 方法调用                   │   │
                    │  └─────────────────────────────┘   │
                    └─────────────────────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │         本系列博客目标              │
                    │                                     │
                    │  1. 深入理解托管运行时嵌入的高级模式 │
                    │  2. 掌握跨语言对象生命周期管理      │
                    │  3. 理解代码生成系统的设计思想      │
                    │  4. 学会动态类型注册的实现技巧      │
                    │  5. 具备扩展和维护类似系统的能力    │
                    │                                     │
                    └─────────────────────────────────────┘
```

---

## 文件索引

| 篇章 | 文件名 | 状态 |
|-----|--------|------|
| 大纲 | `UnrealSharp-00-大纲.md` | ✅ 已完成 |
| 第一篇 | `UnrealSharp-01-概述与架构总览.md` | ✅ 已完成 |
| 第二篇 | `UnrealSharp-02-运行时初始化.md` | ✅ 已完成 |
| 第三篇 | `UnrealSharp-03-GCHandle与对象生命周期.md` | ⏳ 待撰写 |
| 第四篇 | `UnrealSharp-04-托管回调系统.md` | ⏳ 待撰写 |
| 第五篇 | `UnrealSharp-05-Glue代码生成系统_上.md` | ⏳ 待撰写 |
| 第六篇 | `UnrealSharp-06-Glue代码生成系统_下.md` | ⏳ 待撰写 |
| 第七篇 | `UnrealSharp-07-动态类型注册.md` | ⏳ 待撰写 |
| 第八篇 | `UnrealSharp-08-原生函数导出系统.md` | ⏳ 待撰写 |
| 第九篇 | `UnrealSharp-09-方法调用完整流程.md` | ⏳ 待撰写 |
| 第十篇 | `UnrealSharp-10-热重载与编辑器集成.md` | ⏳ 待撰写 |
| 第十一篇 | `UnrealSharp-11-异步编程与委托系统.md` | ⏳ 待撰写 |
| 第十二篇 | `UnrealSharp-12-实战案例分析.md` | ⏳ 待撰写 |
| 第十三篇 | `UnrealSharp-13-性能优化与最佳实践.md` | ⏳ 待撰写 |
| 第十四篇 | `UnrealSharp-14-源码生成器与分析器.md` | ⏳ 待撰写 |
| 第十五篇 | `UnrealSharp-15-总结与展望.md` | ⏳ 待撰写 |
