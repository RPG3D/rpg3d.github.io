---
title: UnrealSharp 插件技术深度解析 - 动态类型注册
date: 2026-03-30 12:00:00
author: GLM-5.0
categories: UnrealSharp
tags: [UnrealSharp, UE5, C#, 反射系统, 动态类型]
series: UnrealSharp 插件技术深度解析
series_number: 7
---

# 运行时的类型魔法 - C#类型到UE反射系统的注册

> **技术深度**：⭐⭐⭐⭐⭐
> **前置知识**：UE5反射系统、UClass/UStruct/UEnum/UInterface、动态类型注册

---

## 引言

在前面的文章中，我们详细分析了 Glue 代码生成系统如何将 UE 的 C++ 类型导出为 C# 绑定代码。但 UnrealSharp 的独特之处在于它实现了**双向类型映射**——不仅 C++ 类型可以在 C# 中使用，C# 定义的类型也能被 UE 反射系统识别！

本章将深入剖析 C# 类型如何被"逆向"注册为 UE 反射类型，这是 UnrealSharp 实现纯 C# 游戏逻辑的核心机制。

---

## 一、UE5 反射系统回顾

### 1.1 反射类型层次

UE 的反射系统定义了以下核心类型：

```
UField (基类)
├── UEnum          - 枚举类型
├── UStruct        - 结构体基类
│   ├── UClass     - 类类型
│   ├── UScriptStruct - 脚本结构体
│   ├── UInterface - 接口类型
│   └── UFunction  - 函数类型
└── UDelegateFunction - 委托函数
```

### 1.2 动态类型注册的可能性

UE5 支持运行时动态注册类型，关键 API：

```cpp
// 注册通知事件
NotifyRegistrationEvent(
    *Package->GetName(),
    *Field->GetName(),
    ENotifyRegistrationType::NRT_Class,
    ENotifyRegistrationPhase::NRP_Finished,
    nullptr, false, Field
);
```

UnrealSharp 正是利用这一机制，在运行时将 C# 类型"注入"UE 反射系统。

---

## 二、类型定义核心数据结构

### 2.1 FCSManagedTypeDefinition

这是连接 C# 类型与 UE 类型的桥梁：

```cpp
// Source/UnrealSharpCore/Public/CSManagedTypeDefinition.h

struct FCSManagedTypeDefinition : TSharedFromThis<FCSManagedTypeDefinition>
{
    // 获取 UE 反射类型字段
    UField* GetDefinitionField() const { return DefinitionField.Get(); }
    
    // 获取类型名称
    FCSFieldName GetFieldName() const { return ReflectionData->FieldName; }
    FName GetEngineName() const { return GetFieldName().GetFName(); }
    
    // 获取所属程序集
    UCSManagedAssembly* GetOwningAssembly() const { return OwningAssembly; }
    
    // 获取反射数据（可强转为具体类型）
    template<typename TReflectionData = FCSTypeReferenceReflectionData>
    TSharedPtr<TReflectionData> GetReflectionData() const;
    
    // 获取 C# 类型的 GCHandle
    TSharedPtr<FGCHandle> GetTypeGCHandle();
    
    // 脏标记：是否需要重新编译
    bool HasStructuralChanges() const;
    bool HasConstructorChanges() const;
    bool RequiresRecompile() const;

private:
    // 生成的 UE 反射类型
    TStrongObjectPtr<UField> DefinitionField;
    
    // 负责编译此类型的编译器
    UCSManagedTypeCompiler* Compiler;
    
    // 所属程序集
    UCSManagedAssembly* OwningAssembly;
    
    // 类型描述数据
    TSharedPtr<FCSTypeReferenceReflectionData> ReflectionData;
    
    // 脏标记
    ECSTypeStructuralFlags DirtyFlags;
    
    // C# 类型的 GCHandle
    TSharedPtr<FGCHandle> TypeGCHandle;
};
```

**关键设计点**：
- `DefinitionField`：生成的 UClass/UScriptStruct/UEnum 等
- `ReflectionData`：从 C# 传递过来的类型元数据
- `TypeGCHandle`：持有对 C# Type 对象的强引用

### 2.2 FCSTypeReferenceReflectionData

类型反射数据基类：

```cpp
// Source/UnrealSharpCore/Public/ReflectionData/CSTypeReferenceReflectionData.h

struct FCSTypeReferenceReflectionData : FCSReflectionDataBase
{
    // 序列化支持
    void SerializeFromJsonString(const char* RawJsonString);
    
    // 类型转换
    UClass* GetAsClass() const;
    UScriptStruct* GetAsStruct() const;
    UEnum* GetAsEnum() const;
    UClass* GetAsInterface() const;
    UDelegateFunction* GetAsDelegate() const;
    UPackage* GetAsPackage() const;
    
    // 元数据查询
    bool HasMetaData(const FString& Key) const;

    // 核心字段
    FCSFieldName FieldName;        // 类型名称（命名空间 + 类名）
    FName AssemblyName;            // 所属程序集
    TArray<FCSMetaDataEntry> MetaData;  // 元数据
    TArray<FCSFieldName> SourceGeneratorDependencies;
};
```

### 2.3 类型脏标记

```cpp
enum ECSTypeStructuralFlags : uint8
{
    None = 0,
    StructuralChanges = 1 << 0,   // 结构变化：属性/函数签名改变
    ConstructorChanges = 1 << 1,  // 构造函数变化
};
```

热重载时，系统根据脏标记决定是否需要重新编译类型。

---

## 三、类型编译器系列

### 3.1 UCSManagedTypeCompiler 基类

```cpp
// Source/UnrealSharpCore/Public/Compilers/CSManagedTypeCompiler.h

UCLASS(Abstract, Transient)
class UCSManagedTypeCompiler : public UObject
{
    GENERATED_BODY()
public:
    // 创建类型字段
    UField* CreateField(const TSharedPtr<FCSManagedTypeDefinition>& ManagedTypeDefinition) const;
    
    // 重新编译类型
    void RecompileManagedTypeDefinition(const TSharedRef<FCSManagedTypeDefinition>& ManagedTypeDefinition) const;

protected:
    // 子类实现
    virtual void Recompile(UField* TypeToRecompile, 
                          const TSharedPtr<FCSManagedTypeDefinition>& ManagedTypeDefinition) const { }
    
    virtual FString GetFieldName(TSharedPtr<const FCSTypeReferenceReflectionData>& ReflectionData) const;
    virtual TSharedPtr<FCSTypeReferenceReflectionData> CreateNewReflectionData() const = 0;

    // 注册字段到加载器
    static void RegisterFieldToLoader(UField* Field, ENotifyRegistrationType RegistrationType)
    {
        NotifyRegistrationEvent(
            *Field->GetOutermost()->GetName(),
            *Field->GetName(),
            RegistrationType,
            ENotifyRegistrationPhase::NRP_Finished,
            nullptr, false, Field
        );
    }
};
```

### 3.2 UCSManagedClassCompiler - 编译 UClass

```cpp
// Source/UnrealSharpCore/Public/Compilers/CSManagedClassCompiler.h

UCLASS()
class UCSManagedClassCompiler : public UCSManagedTypeCompiler
{
    GENERATED_BODY()
public:
    virtual void Recompile(UField* TypeToRecompile, 
                          const TSharedPtr<FCSManagedTypeDefinition>& ManagedTypeDefinition) const override;
    
    // 接口实现
    static void ImplementInterfaces(UClass* ManagedClass, 
                                   const TArray<FCSTypeReferenceReflectionData>& Interfaces);
    
    // 类标志设置
    static void SetClassFlags(UClass* ManagedClass, 
                             const TSharedPtr<const FCSClassReflectionData>& ClassReflectionData);
    
    // CDO 创建
    static UObject* CreateDeferredManagedCDO(UCSClass* ManagedClass);
    static void FinalizeManagedCDO(UCSClass* ManagedClass);
    
    // Subsystem 激活
    static void ActivateSubsystem(TSubclassOf<USubsystem> SubsystemClass);
    static void DeactivateSubsystem(TSubclassOf<USubsystem> SubsystemClass);

private:
    static void CompileClass(TSharedPtr<FCSClassReflectionData> ClassReflectionData, 
                            UCSClass* Field, UClass* SuperClass);
    
    // 父类重定向（如 UDeveloperSettings → UCSDeveloperSettings）
    UClass* TryRedirectSuperClass(TSharedPtr<FCSClassReflectionData> ClassReflectionData, 
                                 UClass* SuperClass) const;
    
    TMap<TObjectKey<UClass>, TWeakObjectPtr<UClass>> RedirectClasses;
};
```

**编译 UClass 的完整流程**：

```cpp
void UCSManagedClassCompiler::Recompile(UField* TypeToRecompile, 
                                        const TSharedPtr<FCSManagedTypeDefinition>& ManagedTypeDefinition) const
{
    UCSClass* Field = static_cast<UCSClass*>(TypeToRecompile);
    TSharedPtr<FCSClassReflectionData> ClassReflectionData = 
        ManagedTypeDefinition->GetReflectionData<FCSClassReflectionData>();
    
    // 1. 解析父类
    UClass* NewSuperClass = TryRedirectSuperClass(ClassReflectionData, Field->GetSuperClass());
    
    // 2. 设置父类关系
    if (!IsValid(Field->GetSuperClass()))
    {
        Field->SetSuperStruct(NewSuperClass);
    }
    
#if WITH_EDITOR
    // 3. 创建/更新关联的蓝图
    CreateOrUpdateOwningBlueprint(ClassReflectionData, Field, NewSuperClass);
#endif
    
    // 4. 执行类编译
    CompileClass(ClassReflectionData, Field, NewSuperClass);
}

void UCSManagedClassCompiler::CompileClass(TSharedPtr<FCSClassReflectionData> ClassReflectionData, 
                                          UCSClass* Field, UClass* SuperClass)
{
    // 设置类标志
    SetClassFlags(Field, ClassReflectionData);
    
    // 实现接口
    ImplementInterfaces(Field, ClassReflectionData->Interfaces);
    
    // 创建属性
    FCSPropertyFactory::CreateAndAssignProperties(Field, ClassReflectionData->Properties);
    
    // 编译构造脚本（组件）
    FCSSimpleConstructionScriptCompiler::CompileSimpleConstructionScript(
        Field, &Field->SimpleConstructionScript, ClassReflectionData->Properties);
    
    // 生成函数
    FCSFunctionFactory::GenerateVirtualFunctions(Field, ClassReflectionData);
    FCSFunctionFactory::GenerateFunctions(Field, ClassReflectionData->Functions);
    
    // 设置构造函数
    Field->ClassConstructor = &UCSClass::ManagedObjectConstructor;
    
    // 绑定和链接
    Field->Bind();
    Field->StaticLink(true);
    Field->AssembleReferenceTokenStream();
    
    // 创建 CDO
    CreateDeferredManagedCDO(Field);
    FinalizeManagedCDO(Field);
    
    // 设置运行时复制数据
    Field->SetUpRuntimeReplicationData();
    
    // 注册到加载器
    RegisterFieldToLoader(Field, ENotifyRegistrationType::NRT_Class);
    
    // 激活 Subsystem
    ActivateSubsystem(Field);
}
```

### 3.3 其他编译器

```cpp
// UCSManagedStructCompiler - 编译 UScriptStruct
class UCSManagedStructCompiler : public UCSManagedTypeCompiler
{
    virtual void Recompile(UField* TypeToRecompile, 
                          const TSharedPtr<FCSManagedTypeDefinition>& ManagedTypeDefinition) const override;
private:
    static void PurgeStruct(UCSScriptStruct* Field);
};

// UCSManagedEnumCompiler - 编译 UEnum
class UCSManagedEnumCompiler : public UCSManagedTypeCompiler
{
    virtual void Recompile(UField* TypeToRecompile, 
                          const TSharedPtr<FCSManagedTypeDefinition>& ManagedTypeDefinition) const override;
private:
    static void PurgeEnum(UCSEnum* Enum);
};

// UCSManagedInterfaceCompiler - 编译 UInterface
class UCSManagedInterfaceCompiler : public UCSManagedTypeCompiler
{
    virtual void Recompile(UField* TypeToRecompile, 
                          const TSharedPtr<FCSManagedTypeDefinition>& ManagedTypeDefinition) const override;
};
```

---

## 四、注册流程详解

### 4.1 程序集加载时的类型扫描

当 C# 程序集加载时，系统会扫描所有带有 `[UClass]`、`[UStruct]` 等特性的类型：

```csharp
// C# 侧类型定义
[UClass]
public partial class AScriptGameMode : AGameModeBase
{
    public AScriptGameMode()
    {
        DefaultPawnClass = typeof(AScriptCharacter);
    }

    public override void BeginPlay()
    {
        base.BeginPlay();
    }
}
```

### 4.2 反射数据传递

C# 侧通过 `[GeneratedType]` 特性标记的类型会被收集并传递给 C++：

```cpp
// 创建类型定义
TSharedPtr<FCSManagedTypeDefinition> FCSManagedTypeDefinition::CreateFromReflectionData(
    const TSharedPtr<FCSTypeReferenceReflectionData>& InReflectionData,
    UCSManagedAssembly* InOwningAssembly,
    UCSManagedTypeCompiler* InCompiler)
{
    TSharedPtr<FCSManagedTypeDefinition> Definition = MakeShared<FCSManagedTypeDefinition>();
    Definition->ReflectionData = InReflectionData;
    Definition->OwningAssembly = InOwningAssembly;
    Definition->Compiler = InCompiler;
    return Definition;
}
```

### 4.3 类型编译与注册

```cpp
UField* UCSManagedTypeCompiler::CreateField(
    const TSharedPtr<FCSManagedTypeDefinition>& ManagedTypeDefinition) const
{
    // 创建对应类型的字段对象
    UField* NewField = nullptr;
    
    // 根据编译器类型创建不同字段
    if (FieldType == UCSClass::StaticClass())
    {
        UPackage* Package = ManagedTypeDefinition->GetReflectionData()->GetAsPackage();
        NewField = NewObject<UCSClass>(Package, 
            ManagedTypeDefinition->GetEngineName(), 
            RF_Public | RF_Standalone);
    }
    // ... 其他类型
    
    // 递归编译
    Recompile(NewField, ManagedTypeDefinition);
    
    return NewField;
}
```

### 4.4 与 UE 系统的集成

编译后的类型通过 `RegisterFieldToLoader` 注册到 UE 的类型系统：

```cpp
static void RegisterFieldToLoader(UField* Field, ENotifyRegistrationType RegistrationType)
{
    NotifyRegistrationEvent(
        *Field->GetOutermost()->GetName(),
        *Field->GetName(),
        RegistrationType,
        ENotifyRegistrationPhase::NRP_Finished,
        nullptr, false, Field
    );
}
```

---

## 五、CDO（Class Default Object）创建

### 5.1 CDO 的意义

CDO 是 UE 中每个 UClass 的默认对象实例，存储类的默认属性值。对于 C# 类：

```cpp
UObject* UCSManagedClassCompiler::CreateDeferredManagedCDO(UCSClass* ManagedClass)
{
    // 创建 CDO
    UObject* CDO = ManagedClass->CreateDefaultObject();
    
    // 初始化托管对象
    // 这里会调用 C# 的构造函数
    
    return CDO;
}

void UCSManagedClassCompiler::FinalizeManagedCDO(UCSClass* ManagedClass)
{
    // 完成初始化
    // 包括属性默认值设置、组件初始化等
}
```

### 5.2 托管对象构造函数

```cpp
// UCSClass 的静态构造函数
static void ManagedObjectConstructor(const FObjectInitializer& ObjectInitializer)
{
    // 1. 调用父类构造函数
    // 2. 创建托管对象实例
    // 3. 调用 C# 构造函数
}
```

---

## 六、编辑器支持

### 6.1 蓝图集成

在编辑器模式下，每个 C# 类都会关联一个 `UCSBlueprint`：

```cpp
#if WITH_EDITOR
void UCSManagedClassCompiler::CreateOrUpdateOwningBlueprint(
    TSharedPtr<FCSClassReflectionData> ClassReflectionData, 
    UCSClass* Field, UClass* SuperClass)
{
    UBlueprint* Blueprint = Field->GetOwningBlueprint();
    
    if (!IsValid(Blueprint))
    {
        // 创建关联蓝图
        Blueprint = NewObject<UCSBlueprint>(Package, *BlueprintName);
        Blueprint->GeneratedClass = Field;
        Blueprint->ParentClass = SuperClass;
        Field->SetOwningBlueprint(Blueprint);
    }
}
#endif
```

### 6.2 类操作刷新

```cpp
#if WITH_EDITOR
void UCSManagedClassCompiler::RefreshClassActions(UClass* ClassToRefresh)
{
    // 刷新蓝图动作数据库
    FBlueprintActionDatabase& ActionDB = FBlueprintActionDatabase::Get();
    ActionDB.RefreshClassActions(ClassToRefresh);
}
#endif
```

---

## 七、实践：调试类型注册

### 7.1 控制台命令

UnrealSharp 提供了丰富的调试命令：

```
// 导出类型信息
UnrealSharp.DumpClass <ClassName>
UnrealSharp.DumpStruct <StructName>
UnrealSharp.DumpEnum <EnumName>

// 导出程序集信息
UnrealSharp.DumpAssembly <AssemblyName>
```

### 7.2 类型注册日志

```
LogUnrealSharp: Creating managed type: AScriptGameMode
LogUnrealSharp:   Parent: AGameModeBase
LogUnrealSharp:   Properties:
LogUnrealSharp:     - DefaultPawnClass (ObjectProperty)
LogUnrealSharp:   Functions:
LogUnrealSharp:     - BeginPlay (Event)
LogUnrealSharp: Registered class: AScriptGameMode
```

---

## 八、总结

### 核心要点

1. **双向类型映射**：UnrealSharp 实现了 C# 类型到 UE 反射系统的完整注册流程
2. **编译器模式**：每种 UE 类型都有对应的编译器，负责从反射数据生成实际类型
3. **动态注册**：利用 UE 的 `NotifyRegistrationEvent` API 实现运行时类型注入
4. **CDO 支持**：完整支持类默认对象，确保属性默认值正确
5. **编辑器集成**：与蓝图系统无缝集成，支持热重载

### 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    C# 程序集加载                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              扫描 [UClass]/[UStruct]/[UEnum] 特性                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              创建 FCSTypeReferenceReflectionData                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              创建 FCSManagedTypeDefinition                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              UCSManagedTypeCompiler.CreateField()                │
│  ┌─────────────────┬─────────────────┬─────────────────────┐   │
│  │ ClassCompiler   │ StructCompiler  │ EnumCompiler        │   │
│  └─────────────────┴─────────────────┴─────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              RegisterFieldToLoader() → NotifyRegistrationEvent   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              UE 反射系统识别 C# 类型                              │
│  ┌─────────────────┬─────────────────┬─────────────────────┐   │
│  │ UCSClass        │ UCSScriptStruct │ UCSEnum             │   │
│  └─────────────────┴─────────────────┴─────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 关键源码文件

| 文件 | 职责 |
|------|------|
| `CSManagedTypeDefinition.h` | 类型定义核心结构 |
| `CSTypeReferenceReflectionData.h` | 反射数据基类 |
| `CSManagedTypeCompiler.h` | 编译器基类 |
| `CSManagedClassCompiler.h/cpp` | UClass 编译器 |
| `CSManagedStructCompiler.h` | UScriptStruct 编译器 |
| `CSManagedEnumCompiler.h` | UEnum 编译器 |
| `CSManagedInterfaceCompiler.h` | UInterface 编译器 |

---

**下一篇**：原生函数导出系统 - UNREALSHARP_FUNCTION 宏的秘密
