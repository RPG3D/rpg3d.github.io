---
title: "类型系统的翻译官 - PropertyTranslator 深度剖析"
date: 2025-03-29 13:00:00 +0800
categories: UnrealSharp
tags: [unreal-engine, csharp, dotnet, marshalling, type-translation]
series: UnrealSharp 插件技术深度解析
---

> **作者**：GLM-5.0

在上一篇文章中，我们了解了 Glue 代码生成系统的整体架构。今天我们深入核心组件 —— **PropertyTranslator（属性翻译器）**，看看它是如何将虚幻引擎的 C++ 类型系统精确映射到 C# 类型系统的。

## 问题引入：类型映射的挑战

将 UE 的 C++ 类型映射到 C# 并非简单的名称替换：

```
UE C++                          →    C#
─────────────────────────────────────────────────
int32                           →    int          ✓ 简单
bool                            →    bool         ？位域处理
FString                         →    string       ？内存管理差异
TArray<int32>                   →    IList<int>   ？容器语义
TWeakObjectPtr<UActor>          →    TWeakObjectPtr<AActor>  ？泛型包装
UObject*                        →    UObject      ？生命周期同步
TMultiCastDelegate<...>         →    TMulticastDelegate<...> ？委托绑定
```

每个类型都有独特的内存布局、生命周期和访问语义。PropertyTranslator 就是解决这些差异的核心抽象。

## PropertyTranslator 基类设计

### 继承体系

```
PropertyTranslator (抽象基类)
├── SimpleTypePropertyTranslator (简单类型基类)
│   ├── BlittableTypePropertyTranslator (可直接拷贝类型)
│   │   ├── FloatPropertyTranslator
│   │   ├── IntPropertyTranslator
│   │   └── BlittableStructPropertyTranslator
│   ├── BoolPropertyTranslator (位域处理)
│   ├── StringPropertyTranslator
│   ├── ObjectPropertyTranslator
│   ├── StructPropertyTranslator
│   └── InterfacePropertyTranslator
├── ContainerPropertyTranslator (TArray/TMap/TSet/TOptional)
├── DelegateBasePropertyTranslator (委托基类)
│   ├── MulticastDelegatePropertyTranslator
│   └── SinglecastDelegatePropertyTranslator
├── EnumPropertyTranslator
└── ObjectContainerPropertyTranslator (对象容器基类)
    ├── SubclassOfPropertyTranslator (TSubclassOf)
    └── SoftObjectPropertyTranslator (TSoftObjectPtr)
```

### 核心抽象方法

```csharp
public abstract class PropertyTranslator
{
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 支持能力声明
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    private readonly EPropertyUsageFlags _supportedPropertyUsage;
    
    // 可作为：类属性、结构体属性、函数参数、返回值、容器内部类型
    public bool IsSupportedAsProperty => _supportedPropertyUsage.HasFlag(EPropertyUsageFlags.Property);
    public bool IsSupportedAsParameter => _supportedPropertyUsage.HasFlag(EPropertyUsageFlags.Parameter);
    public bool IsSupportedAsReturnValue => _supportedPropertyUsage.HasFlag(EPropertyUsageFlags.ReturnValue);
    public bool IsSupportedAsInner => _supportedPropertyUsage.HasFlag(EPropertyUsageFlags.Inner);
    
    // 是否可直接内存拷贝（无需 Marshalling）
    public virtual bool IsBlittable => false;
    
    // 是否支持 setter
    public virtual bool SupportsSetter => true;
    
    // 是否缓存属性指针（容器类型需要）
    public virtual bool CacheProperty => false;
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 核心抽象方法 - 子类必须实现
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    // 能否导出此属性
    public abstract bool CanExport(UhtProperty property);
    
    // 获取 C# 托管类型名称
    public abstract string GetManagedType(UhtProperty property);
    
    // 获取 Marshaller 类型名称
    public abstract string GetMarshaller(UhtProperty property);
    
    // 导出 Marshaller 委托
    public abstract string ExportMarshallerDelegates(UhtProperty property);
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // Marshalling 方法 - 生成实际的转换代码
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    // 从原生内存读取到托管对象
    public abstract void ExportFromNative(
        GeneratorStringBuilder builder, 
        UhtProperty property, 
        string propertyName,
        string assignmentOrReturn, 
        string sourceBuffer, 
        string offset, 
        bool cleanupSourceBuffer,
        bool reuseRefMarshallers);
    
    // 从托管对象写入到原生内存
    public abstract void ExportToNative(
        GeneratorStringBuilder builder, 
        UhtProperty property, 
        string propertyName,
        string destinationBuffer, 
        string offset, 
        string source, 
        bool reuseRefMarshallers);
}
```

### 使用场景标志

```csharp
[Flags]
public enum EPropertyUsageFlags
{
    None = 0,
    Property = 1 << 0,        // 作为类属性
    StructProperty = 1 << 1,  // 作为结构体属性
    Parameter = 1 << 2,       // 作为函数参数
    ReturnValue = 1 << 3,     // 作为返回值
    Inner = 1 << 4,           // 作为容器内部类型
    Any = Property | StructProperty | Parameter | ReturnValue | Inner,
}
```

## SimpleTypePropertyTranslator：简单类型基类

大多数基础类型共享相似的 Marshalling 逻辑：

```csharp
public class SimpleTypePropertyTranslator : PropertyTranslator
{
    public override bool IsBlittable => true;  // 默认可 Blittable
    
    protected readonly string ManagedType;     // C# 类型名
    protected readonly string Marshaller;      // Marshaller 名
    private readonly Type _propertyType;       // UHT 属性类型
    
    protected SimpleTypePropertyTranslator(
        Type propertyType, 
        string managedType = "", 
        string marshaller = "") 
        : base(EPropertyUsageFlags.Any)
    {
        _propertyType = propertyType;
        ManagedType = managedType;
        Marshaller = marshaller;
    }
    
    // 泛型类型使用 "DOT" 占位符
    public override string GetManagedType(UhtProperty property)
    {
        return property.IsGenericType() ? "DOT" : ManagedType;
    }
    
    // 标准的 FromNative 实现
    public override void ExportFromNative(
        GeneratorStringBuilder builder, 
        UhtProperty property, 
        string propertyName,
        string assignmentOrReturn, 
        string sourceBuffer, 
        string offset, 
        bool cleanupSourceBuffer,
        bool reuseRefMarshallers)
    {
        builder.AppendLine(
            $"{assignmentOrReturn} {GetMarshaller(property)}.FromNative({sourceBuffer} + {offset}, 0);");
    }
    
    // 标准的 ToNative 实现
    public override void ExportToNative(
        GeneratorStringBuilder builder, 
        UhtProperty property, 
        string propertyName,
        string destinationBuffer, 
        string offset, 
        string source, 
        bool reuseRefMarshallers)
    {
        builder.AppendLine(
            $"{GetMarshaller(property)}.ToNative({destinationBuffer} + {offset}, 0, {source});");
    }
    
    // 默认值处理
    public override string ConvertCppDefaultValue(string defaultValue, UhtFunction function, UhtProperty parameter)
    {
        return defaultValue == "None" ? GetNullValue(parameter) : defaultValue;
    }
    
    public override string GetNullValue(UhtProperty property)
    {
        return $"default({GetManagedType(property)})";
    }
}
```

## Blittable 类型：零开销 Marshalling

### BlittableTypePropertyTranslator

Blittable 类型在托管和非托管内存中布局完全一致，可直接内存拷贝：

```csharp
public class BlittableTypePropertyTranslator : SimpleTypePropertyTranslator
{
    public BlittableTypePropertyTranslator(Type propertyType, string managedType) 
        : base(propertyType, managedType, string.Empty)
    {
    }
    
    public override bool ExportDefaultParameter => true;
    
    public override string GetMarshaller(UhtProperty property) 
        => $"BlittableMarshaller<{GetManagedType(property)}>";
    
    public override bool CanSupportGenericType(UhtProperty property) => false;
}
```

**生成的代码**：

```csharp
// C++: int32 MyValue;
// C#:
public int MyValue
{
    get => BlittableMarshaller<int>.FromNative(NativeObject + MyValue_Offset, 0);
    set => BlittableMarshaller<int>.ToNative(NativeObject + MyValue_Offset, 0, value);
}
```

**BlittableMarshaller 实现**：

```csharp
public static class BlittableMarshaller<T> where T : unmanaged
{
    public static unsafe T FromNative(IntPtr nativeBuffer, int arrayIndex)
    {
        return ((T*)nativeBuffer)[arrayIndex];
    }
    
    public static unsafe void ToNative(IntPtr nativeBuffer, int arrayIndex, T value)
    {
        ((T*)nativeBuffer)[arrayIndex] = value;
    }
}
```

### FloatPropertyTranslator：特殊处理默认值

```csharp
public class FloatPropertyTranslator : BlittableTypePropertyTranslator
{
    public FloatPropertyTranslator() : base(typeof(UhtFloatProperty), "float") { }
    
    // C++ 默认值如 "3.14" 需要加 "f" 后缀
    public override string ConvertCppDefaultValue(string defaultValue, UhtFunction function, UhtProperty parameter)
    {
        return base.ConvertCppDefaultValue(defaultValue, function, parameter) + "f";
    }
}
```

### IntPropertyTranslator：支持 CustomStruct 参数

```csharp
public class IntPropertyTranslator : BlittableTypePropertyTranslator
{
    public IntPropertyTranslator() : base(typeof(UhtIntProperty), "int") { }
    
    public override string GetManagedType(UhtProperty property)
    {
        // CustomStruct 参数使用 CSP 泛型占位符
        if (property.Outer is UhtFunction function && property.IsCustomStructureType())
        {
            return function.GetCustomStructParamCount() == 1 
                ? "CSP" 
                : $"CSP{property.GetPrecedingCustomStructParams()}";
        }
        return base.GetManagedType(property);
    }
    
    public override string GetMarshaller(UhtProperty property)
    {
        if (property.Outer is UhtFunction && property.IsCustomStructureType())
        {
            return $"StructMarshaller<{GetManagedType(property)}>";
        }
        return base.GetMarshaller(property);
    }
    
    public override bool CanSupportCustomStruct(UhtProperty property) => true;
}
```

## BoolPropertyTranslator：位域的艺术

UE 的 `bool` 属性可能是独立字节，也可能是位域中的单个位：

```csharp
public class BoolPropertyTranslator : SimpleTypePropertyTranslator
{
    private const string FieldMaskPostfix = "_FieldMask";
    
    public BoolPropertyTranslator() : base(typeof(UhtBoolProperty), "bool", string.Empty) { }
    
    // 判断是否为位域
    public bool IsBitfield(UhtProperty property)
    {
        return property.IsBitfield && !property.HasGetterSetterPair();
    }
    
    // 根据是否位域选择不同的 Marshaller
    public override string GetMarshaller(UhtProperty property)
    {
        return IsBitfield(property) ? "BitfieldBoolMarshaller" : "BoolMarshaller";
    }
    
    // 为位域属性生成额外的字段掩码变量
    public override void ExportPropertyVariables(GeneratorStringBuilder builder, UhtProperty property, string propertyEngineName)
    {
        if (IsBitfield(property))
        {
            builder.AppendLine($"static byte {propertyEngineName}_FieldMask;");
        }
        base.ExportPropertyVariables(builder, property, propertyEngineName);
    }
    
    // 在静态构造函数中获取字段掩码
    public override void ExportPropertyStaticConstructor(GeneratorStringBuilder builder, UhtProperty property, string nativePropertyName)
    {
        if (IsBitfield(property))
        {
            builder.AppendLine(
                $"{nativePropertyName}_FieldMask = CallGetBoolPropertyFieldMaskFromName(NativeClassPtr, \"{nativePropertyName}\");");
        }
        base.ExportPropertyStaticConstructor(builder, property, nativePropertyName);
    }
    
    // 位域的 FromNative
    public override void ExportFromNative(
        GeneratorStringBuilder builder,
        UhtProperty property,
        string propertyName,
        string assignmentOrReturn,
        string sourceBuffer,
        string offset,
        bool cleanupSourceBuffer,
        bool reuseRefMarshallers)
    {
        if (IsBitfield(property))
        {
            builder.AppendLine(
                $"{assignmentOrReturn} BitfieldBoolMarshaller.FromNative({sourceBuffer} + {offset}, {propertyName}_FieldMask);");
            return;
        }
        base.ExportFromNative(builder, property, propertyName, assignmentOrReturn, sourceBuffer, offset, cleanupSourceBuffer, reuseRefMarshallers);
    }
    
    // 位域的 ToNative
    public override void ExportToNative(
        GeneratorStringBuilder builder,
        UhtProperty property,
        string propertyName,
        string destinationBuffer,
        string offset,
        string source,
        bool reuseRefMarshallers)
    {
        if (IsBitfield(property))
        {
            builder.AppendLine(
                $"BitfieldBoolMarshaller.ToNative({destinationBuffer} + {offset}, {propertyName}_FieldMask, {source});");
            return;
        }
        base.ExportToNative(builder, property, propertyName, destinationBuffer, offset, source, false);
    }
}
```

**生成的代码**：

```csharp
// 位域属性
static byte bIsAlive_FieldMask;
static int bIsAlive_Offset;

static AActor()
{
    bIsAlive_FieldMask = CallGetBoolPropertyFieldMaskFromName(NativeClassPtr, "bIsAlive");
    bIsAlive_Offset = CallGetPropertyOffsetFromName(NativeClassPtr, "bIsAlive");
}

public bool bIsAlive
{
    get => BitfieldBoolMarshaller.FromNative(NativeObject + bIsAlive_Offset, bIsAlive_FieldMask);
    set => BitfieldBoolMarshaller.ToNative(NativeObject + bIsAlive_Offset, bIsAlive_FieldMask, value);
}
```

**BitfieldBoolMarshaller 实现**：

```csharp
public static class BitfieldBoolMarshaller
{
    public static unsafe bool FromNative(IntPtr nativeBuffer, byte fieldMask)
    {
        byte* ptr = (byte*)nativeBuffer;
        return (*ptr & fieldMask) != 0;
    }
    
    public static unsafe void ToNative(IntPtr nativeBuffer, byte fieldMask, bool value)
    {
        byte* ptr = (byte*)nativeBuffer;
        if (value)
            *ptr |= fieldMask;
        else
            *ptr &= (byte)~fieldMask;
    }
}
```

## StringPropertyTranslator：跨语言字符串

FString 和 C# string 的内存模型完全不同：

```csharp
public class StringPropertyTranslator : PropertyTranslator
{
    public StringPropertyTranslator() : base(EPropertyUsageFlags.Any) { }
    
    public override bool CacheProperty => true;  // 需要缓存属性指针
    
    public override bool CanExport(UhtProperty property) => property is UhtStrProperty;
    
    public override string GetManagedType(UhtProperty property) => "string";
    
    public override string GetMarshaller(UhtProperty property) => "StringMarshaller";
    
    public override string GetNullValue(UhtProperty property) => "\"\"";  // 空字符串而非 null
    
    public override void ExportFromNative(
        GeneratorStringBuilder builder,
        UhtProperty property,
        string propertyName,
        string assignmentOrReturn,
        string sourceBuffer,
        string offset,
        bool cleanupSourceBuffer,
        bool reuseRefMarshallers)
    {
        if (!reuseRefMarshallers)
        {
            builder.AppendLine($"IntPtr {propertyName}_NativePtr = {sourceBuffer} + {offset};");
        }
        builder.AppendLine($"{assignmentOrReturn} StringMarshaller.FromNative({propertyName}_NativePtr, 0);");
        
        if (cleanupSourceBuffer)
        {
            builder.AppendLine($"StringMarshaller.DestructInstance({propertyName}_NativePtr, 0);");
        }
    }
    
    public override void ExportToNative(
        GeneratorStringBuilder builder,
        UhtProperty property,
        string propertyName,
        string destinationBuffer,
        string offset,
        string source,
        bool reuseRefMarshallers)
    {
        builder.AppendLine($"IntPtr {propertyName}_NativePtr = {destinationBuffer} + {offset};");
        builder.AppendLine($"StringMarshaller.ToNative({propertyName}_NativePtr, 0, {source});");
    }
    
    public override string ConvertCppDefaultValue(string defaultValue, UhtFunction function, UhtProperty parameter)
    {
        return "\"" + defaultValue + "\"";  // 添加引号
    }
}
```

**StringMarshaller 核心实现**：

```csharp
public static class StringMarshaller
{
    public static unsafe string FromNative(IntPtr nativeBuffer, int arrayIndex)
    {
        FString* fstr = (FString*)(nativeBuffer + arrayIndex * sizeof(FString));
        if (fstr->Data == null)
            return string.Empty;
        
        // FString 是 UTF-16 字符串
        return new string(fstr->Data, 0, fstr->Num);
    }
    
    public static unsafe void ToNative(IntPtr nativeBuffer, int arrayIndex, string value)
    {
        FString* fstr = (FString*)(nativeBuffer + arrayIndex * sizeof(FString));
        
        // 分配 FString 内存
        fstr->Num = value.Length;
        fstr->Max = value.Length + 1;
        fstr->Data = (char*)FMemory.Malloc((value.Length + 1) * 2);
        
        // 拷贝字符
        fixed (char* src = value)
        {
            Memory.Memcpy(fstr->Data, src, value.Length * 2);
        }
        fstr->Data[value.Length] = '\0';  // null 终止符
    }
    
    public static unsafe void DestructInstance(IntPtr nativeBuffer, int arrayIndex)
    {
        FString* fstr = (FString*)(nativeBuffer + arrayIndex * sizeof(FString));
        if (fstr->Data != null)
        {
            FMemory.Free(fstr->Data);
            fstr->Data = null;
        }
    }
}
```

## ObjectPropertyTranslator：UObject 引用

UObject 引用需要处理对象生命周期同步：

```csharp
public class ObjectPropertyTranslator : SimpleTypePropertyTranslator
{
    public ObjectPropertyTranslator() : base(typeof(UhtObjectProperty)) { }
    
    public override bool CanExport(UhtProperty property)
    {
        UhtObjectPropertyBase objectProperty = (UhtObjectPropertyBase)property;
        UhtClass? metaClass = objectProperty.Class;
        
        // 接口类型由 InterfacePropertyTranslator 处理
        if (metaClass.HasAnyFlags(EClassFlags.Interface) ||
            metaClass.EngineType == UhtEngineType.Interface ||
            metaClass.EngineType == UhtEngineType.NativeInterface)
        {
            return false;
        }
        
        return base.CanExport(property);
    }
    
    public override string GetNullValue(UhtProperty property) => "null";
    
    public override string GetManagedType(UhtProperty property)
    {
        return GetManagedType(property, property.HasMetadata("Nullable"));   
    }
    
    private static string GetManagedType(UhtProperty property, bool isNullable)
    {
        string nullableAnnotation = isNullable ? "?" : string.Empty;
        
        // 泛型类型使用 DOT 占位符
        if (property.IsGenericType())
        {
            return $"DOT{nullableAnnotation}";
        }
        
        UhtObjectProperty objectProperty = (UhtObjectProperty)property;
        return $"{objectProperty.Class.GetFullManagedName()}{nullableAnnotation}";
    }
    
    public override string GetMarshaller(UhtProperty property)
    {
        if (property.Outer is UhtProperty outerProperty && outerProperty.IsGenericType())
        {
            return "ObjectMarshaller<DOT>";
        }
        return $"ObjectMarshaller<{GetManagedType(property, false)}>";
    }
    
    public override bool CanSupportGenericType(UhtProperty property) => true;
}
```

**ObjectMarshaller 实现**：

```csharp
public static class ObjectMarshaller<T> where T : UnrealSharpObject
{
    public static T? FromNative(IntPtr nativeBuffer, int arrayIndex)
    {
        IntPtr nativeObject = *(IntPtr*)(nativeBuffer + arrayIndex * IntPtr.Size);
        if (nativeObject == IntPtr.Zero)
            return null;
        
        // 通过 GCHandle 映射表获取托管对象
        return (T?)GCHandleUtilities.GetManagedObject(nativeObject);
    }
    
    public static void ToNative(IntPtr nativeBuffer, int arrayIndex, T? value)
    {
        IntPtr* ptr = (IntPtr*)(nativeBuffer + arrayIndex * IntPtr.Size);
        
        if (value == null)
        {
            *ptr = IntPtr.Zero;
            return;
        }
        
        // 存储原生对象指针
        *ptr = value.NativeObject;
    }
}
```

## ContainerPropertyTranslator：容器类型

TArray、TMap、TSet、TOptional 共享相似的处理逻辑：

```csharp
public class ContainerPropertyTranslator : PropertyTranslator
{
    private readonly string _copyMarshallerName;
    private readonly string _readOnlyMarshallerName;
    private readonly string _marshallerName;
    private readonly string _readOnlyInterfaceName;
    private readonly string _interfaceName;
    
    public override bool IsBlittable => false;
    public override bool SupportsSetter => true;
    public override bool CacheProperty => true;  // 必须缓存属性指针
    
    public ContainerPropertyTranslator(
        string copyMarshallerName, 
        string readOnlyMarshallerName, 
        string marshallerName, 
        string readOnlyInterfaceName, 
        string interfaceName) 
        : base(ContainerSupportedUsages)
    {
        _copyMarshallerName = copyMarshallerName;
        _readOnlyMarshallerName = readOnlyMarshallerName;
        _marshallerName = marshallerName;
        _readOnlyInterfaceName = readOnlyInterfaceName;
        _interfaceName = interfaceName;
    }
    
    // 检查所有内部类型是否支持
    public override bool CanExport(UhtProperty property)
    {
        UhtContainerBaseProperty containerProperty = (UhtContainerBaseProperty)property;
        List<UhtProperty> innerProperties = containerProperty.GetInnerProperties();
        
        foreach (UhtProperty innerProperty in innerProperties)
        {
            PropertyTranslator? translator = innerProperty.GetTranslator();
            if (translator == null || !translator.CanExport(innerProperty) || !translator.IsSupportedAsInner)
            {
                return false;
            }
        }
        return true;
    }
    
    // 托管类型是接口（IList、IReadOnlyList 等）
    public override string GetManagedType(UhtProperty property) => GetWrapperInterface(property);
    
    // 根据 BlueprintReadOnly 标志选择不同的接口类型
    string GetWrapperInterface(UhtProperty property)
    {
        string innerManagedType = GetInnerPropertiesManagedTypes((UhtContainerBaseProperty)property);
        string interfaceType = property.PropertyFlags.HasAnyFlags(EPropertyFlags.BlueprintReadOnly) 
            ? _readOnlyInterfaceName 
            : _interfaceName;
        return $"{interfaceType}<{innerManagedType}>";
    }
    
    // Marshaller 类型选择
    string GetWrapperType(UhtProperty property)
    {
        bool isStructProperty = property.IsOuter<UhtScriptStruct>();
        bool isParameter = property.IsOuter<UhtFunction>();
        bool isNativeGetterSetter = property.HasAnyNativeGetterSetter();
        
        string innerManagedType = GetInnerPropertiesManagedTypes((UhtContainerBaseProperty)property);
        
        // 结构体属性、参数、Native Getter/Setter 使用 Copy Marshaller
        // BlueprintReadOnly 使用 ReadOnly Marshaller
        // 其他使用普通 Marshaller
        string containerType = isStructProperty || isParameter || isNativeGetterSetter 
            ? _copyMarshallerName 
            : property.PropertyFlags.HasAnyFlags(EPropertyFlags.BlueprintReadOnly) 
                ? _readOnlyMarshallerName 
                : _marshallerName;
        
        return $"{containerType}<{innerManagedType}>";
    }
    
    // 生成 Marshaller 实例创建代码
    void ExportMarshallerCreation(UhtProperty property, GeneratorStringBuilder builder, string propertyManagedName)
    {
        string wrapperType = GetWrapperType(property);
        string marshallingDelegates = ExportMarshallerDelegates(property);
        builder.AppendLine(
            $"{propertyManagedName}_Marshaller ??= new {wrapperType}({propertyManagedName}_NativeProperty, {marshallingDelegates});");
    }
    
    // 导出内部类型的 Marshaller 委托
    public override string ExportMarshallerDelegates(UhtProperty property)
    {
        UhtContainerBaseProperty containerProperty = (UhtContainerBaseProperty)property;
        List<UhtProperty> properties = containerProperty.GetInnerProperties();
        
        return string.Join(", ", properties.ConvertAll(p =>
        {
            PropertyTranslator translator = p.GetTranslator()!;
            return translator.ExportMarshallerDelegates(p);
        }));
    }
}
```

**注册容器翻译器**：

```csharp
// TArray → IList<T> / IReadOnlyList<T>
AddPropertyTranslator(typeof(UhtArrayProperty), new ContainerPropertyTranslator(
    "ArrayCopyMarshaller",
    "ArrayReadOnlyMarshaller", 
    "ArrayMarshaller",
    "System.Collections.Generic.IReadOnlyList",
    "System.Collections.Generic.IList"));

// TMap → IDictionary<K, V> / IReadOnlyDictionary<K, V>
AddPropertyTranslator(typeof(UhtMapProperty), new ContainerPropertyTranslator(
    "MapCopyMarshaller",
    "MapReadOnlyMarshaller",
    "MapMarshaller",
    "System.Collections.Generic.IReadOnlyDictionary",
    "System.Collections.Generic.IDictionary"));

// TSet → ISet<T> / IReadOnlySet<T>
AddPropertyTranslator(typeof(UhtSetProperty), new ContainerPropertyTranslator(
    "SetCopyMarshaller",
    "SetReadOnlyMarshaller",
    "SetMarshaller",
    "System.Collections.Generic.IReadOnlySet",
    "System.Collections.Generic.ISet"));

// TOptional → TOptional<T>
AddPropertyTranslator(typeof(UhtOptionalProperty), new ContainerPropertyTranslator(
    "OptionalMarshaller",
    "OptionalMarshaller",
    "OptionalMarshaller",
    "UnrealSharp.TOptional",
    "UnrealSharp.TOptional"));
```

**生成的代码**：

```csharp
// TArray<int32> MyArray
static IntPtr MyArray_NativeProperty;
static ArrayMarshaller<int> MyArray_Marshaller;

static AActor()
{
    MyArray_NativeProperty = CallGetNativePropertyFromName(NativeClassPtr, "MyArray");
}

public IList<int> MyArray
{
    get
    {
        MyArray_Marshaller ??= new ArrayMarshaller<int>(
            MyArray_NativeProperty, 
            BlittableMarshaller<int>.ToNative, 
            BlittableMarshaller<int>.FromNative);
        return MyArray_Marshaller.FromNative(NativeObject + MyArray_Offset, 0);
    }
    set
    {
        MyArray_Marshaller ??= new ArrayMarshaller<int>(...);
        MyArray_Marshaller.ToNative(NativeObject + MyArray_Offset, 0, value);
    }
}
```

## StructPropertyTranslator：嵌套结构体

结构体需要区分 Blittable 和非 Blittable：

```csharp
// Blittable 结构体（如 FVector）
public class BlittableStructPropertyTranslator : BlittableTypePropertyTranslator
{
    public BlittableStructPropertyTranslator() : base(typeof(UhtStructProperty), "") { }
    
    public override bool ExportDefaultParameter => false;
    
    public override bool CanExport(UhtProperty property)
    {
        UhtStructProperty structProperty = (UhtStructProperty)property;
        return structProperty.ScriptStruct.IsStructBlittable();
    }
    
    public override string GetManagedType(UhtProperty property)
    {
        UhtStructProperty structProperty = (UhtStructProperty)property;
        return structProperty.ScriptStruct.GetFullManagedName();
    }
}

// 非 Blittable 结构体
public class StructPropertyTranslator : SimpleTypePropertyTranslator
{
    public StructPropertyTranslator() : base(typeof(UhtStructProperty)) { }
    
    public override bool ExportDefaultParameter => false;
    public override bool IsBlittable => false;
    
    public override string GetManagedType(UhtProperty property)
    {
        UhtStructProperty structProperty = (UhtStructProperty)property;
        return structProperty.ScriptStruct.GetFullManagedName();
    }
    
    public override string GetMarshaller(UhtProperty property)
    {
        return $"StructMarshaller<{GetManagedType(property)}>";
    }
}
```

### Blittable 结构体判断

```csharp
public static class StructUtilities
{
    public static bool IsStructBlittable(this UhtStruct structObj)
    {
        // 从 JSON 配置中读取手动标记的 Blittable 类型
        if (PropertyTranslatorManager.SpecialTypeInfo.Structs.BlittableTypes.ContainsKey(structObj.SourceName))
        {
            return true;
        }
        
        // TODO: 未来可以添加头文件解析器，检查是否有非 UPROPERTY 成员
        // 当前保守策略：只信任配置文件中的标记
        return false;
    }
}
```

## 委托类型翻译器

### DelegateBasePropertyTranslator

```csharp
public class DelegateBasePropertyTranslator : PropertyTranslator
{
    public DelegateBasePropertyTranslator(EPropertyUsageFlags supportedPropertyUsage) 
        : base(supportedPropertyUsage) { }
    
    public override bool CacheProperty => true;
    
    private const string DelegateSignatureSuffix = "__DelegateSignature";
    private const string StructPrefix = "F";
    
    // 提取委托名称
    public static string GetDelegateName(UhtFunction function)
    {
        string engineName = function.EngineName;
        
        int suffixIndex = engineName.IndexOf(DelegateSignatureSuffix, StringComparison.Ordinal);
        
        if (suffixIndex == -1)
        {
            return StructPrefix + engineName;
        }
        
        string strippedDelegateName = engineName.Substring(0, suffixIndex);
        
        // 添加外层类名前缀以区分同名委托
        if (function.Outer != null && function.Outer is not UhtPackage)
        {
            string outerName = function.Outer.SourceName;
            
            // 移除 "U" 前缀
            if (outerName.StartsWith("U") && outerName.Length > 1 && char.IsUpper(outerName[1]))
            {
                outerName = outerName.Substring(1);
            }
            
            strippedDelegateName = $"{outerName}_{strippedDelegateName}";
        }
        
        return StructPrefix + strippedDelegateName;
    }
    
    public static string GetFullDelegateName(UhtFunction function)
    {
        return $"{function.GetNamespace()}.{GetDelegateName(function)}";
    }
}
```

### MulticastDelegatePropertyTranslator

```csharp
public class MulticastDelegatePropertyTranslator : DelegateBasePropertyTranslator
{
    public MulticastDelegatePropertyTranslator() : base(EPropertyUsageFlags.Property) { }
    
    public override bool CanExport(UhtProperty property)
    {
        UhtMulticastDelegateProperty multicastDelegateProperty = (UhtMulticastDelegateProperty)property;
        return ScriptGeneratorUtilities.CanExportParameters(multicastDelegateProperty.Function);
    }
    
    public override string GetManagedType(UhtProperty property)
    {
        UhtMulticastDelegateProperty multicastDelegateProperty = (UhtMulticastDelegateProperty)property;
        return $"TMulticastDelegate<{GetFullDelegateName(multicastDelegateProperty.Function)}>";
    }
    
    public override string GetNullValue(UhtProperty property) => "null";
    
    // 多播委托需要 BackingField 来缓存
    public override void ExportPropertyVariables(GeneratorStringBuilder builder, UhtProperty property, string propertyEngineName)
    {
        base.ExportPropertyVariables(builder, property, propertyEngineName);
        string backingField = $"{property.SourceName}_BackingField";
        string fullDelegateName = GetFullDelegateName(((UhtMulticastDelegateProperty)property).Function);
        builder.AppendLine($"private TMulticastDelegate<{fullDelegateName}> {backingField};");
    }
    
    public override void ExportPropertyGetter(GeneratorStringBuilder builder, UhtProperty property, string propertyManagedName)
    {
        string backingField = $"{property.SourceName}_BackingField";
        string propertyFieldName = GetNativePropertyField(propertyManagedName);
        string fullDelegateName = GetFullDelegateName(((UhtMulticastDelegateProperty)property).Function);
        
        // 懒加载
        builder.AppendLine($"if ({backingField} == null)");
        builder.OpenBrace();
        builder.AppendLine($"{backingField} = MulticastDelegateMarshaller<{fullDelegateName}>.FromNative(NativeObject + {propertyManagedName}_Offset, 0, {propertyFieldName});");
        builder.CloseBrace();
        builder.AppendLine($"return {backingField};");
    }
    
    public override void ExportPropertySetter(GeneratorStringBuilder builder, UhtProperty property, string propertyManagedName)
    {
        string backingField = $"{property.SourceName}_BackingField";
        string fullDelegateName = GetFullDelegateName(((UhtMulticastDelegateProperty)property).Function);
        
        // 防止重复赋值
        builder.AppendLine($"if (value == {backingField})");
        builder.OpenBrace();
        builder.AppendLine("return;");
        builder.CloseBrace();
        
        builder.AppendLine($"{backingField} = value;");
        builder.AppendLine($"MulticastDelegateMarshaller<{fullDelegateName}>.ToNative(NativeObject + {propertyManagedName}_Offset, 0, value);");
    }
}
```

**生成的委托代码**：

```csharp
// C++: DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnActorClicked, AActor*, ClickedActor);
public delegate void FOnActorClicked(AActor ClickedActor);

// 属性
private TMulticastDelegate<FOnActorClicked> OnActorClicked_BackingField;
static IntPtr OnActorClicked_NativeProperty;

public TMulticastDelegate<FOnActorClicked> OnActorClicked
{
    get
    {
        if (OnActorClicked_BackingField == null)
        {
            OnActorClicked_BackingField = MulticastDelegateMarshaller<FOnActorClicked>
                .FromNative(NativeObject + OnActorClicked_Offset, 0, OnActorClicked_NativeProperty);
        }
        return OnActorClicked_BackingField;
    }
    set
    {
        if (value == OnActorClicked_BackingField) return;
        OnActorClicked_BackingField = value;
        MulticastDelegateMarshaller<FOnActorClicked>
            .ToNative(NativeObject + OnActorClicked_Offset, 0, value);
    }
}
```

## EnumPropertyTranslator：枚举处理

```csharp
public class EnumPropertyTranslator : BlittableTypePropertyTranslator
{
    public EnumPropertyTranslator() : base(typeof(UhtByteProperty), string.Empty) { }
    
    public override bool CanExport(UhtProperty property)
    {
        return property is UhtEnumProperty or UhtByteProperty && GetEnum(property) != null;
    }
    
    public override string GetManagedType(UhtProperty property)
    {
        UhtEnum enumObj = GetEnum(property)!;
        return enumObj.GetFullManagedName();
    }
    
    public override string GetMarshaller(UhtProperty property)
    {
        return $"EnumMarshaller<{GetManagedType(property)}>";
    }
    
    // 将 C++ 枚举值名转换为 C# 枚举值
    public override string ConvertCppDefaultValue(string defaultValue, UhtFunction function, UhtProperty parameter)
    {
        UhtEnum enumObj = GetEnum(parameter)!;
        int index = enumObj.GetIndexByName(defaultValue);
        string valueName = ScriptGeneratorUtilities.GetCleanEnumValueName(enumObj, enumObj.EnumValues[index]);
        return $"{GetManagedType(parameter)}.{valueName}";
    }
}
```

## 特殊对象容器类型

### ObjectContainerPropertyTranslator 基类

```csharp
public abstract class ObjectContainerPropertyTranslator : SimpleTypePropertyTranslator
{
    private readonly string _marshaller;
    
    public ObjectContainerPropertyTranslator(Type propertyType, string managedType, string marshaller) 
        : base(propertyType, managedType)
    {
        _marshaller = marshaller;
    }
    
    protected abstract UhtClass GetMetaClass(UhtObjectPropertyBase property);
    
    public override string GetManagedType(UhtProperty property)
    {
        UhtObjectPropertyBase objectContainerProperty = (UhtObjectPropertyBase)property;
        return $"{ManagedType}<{GetFullName(objectContainerProperty)}>";
    }
    
    public override string GetMarshaller(UhtProperty property)
    {
        UhtObjectPropertyBase objectContainerProperty = (UhtObjectPropertyBase)property;
        return $"{_marshaller}<{GetFullName(objectContainerProperty)}>";
    }
    
    string GetFullName(UhtObjectPropertyBase property)
    {
        UhtClass metaClass = GetMetaClass(property);
        return property.IsGenericType() ? "DOT" : metaClass.GetFullManagedName();
    }
}
```

### TSubclassOf 和 TSoftObjectPtr

```csharp
// TSubclassOf<UActor> → TSubclassOf<AActor>
public class SubclassOfPropertyTranslator : ObjectContainerPropertyTranslator
{
    public SubclassOfPropertyTranslator() 
        : base(typeof(UhtClassProperty), "TSubclassOf", "SubclassOfMarshaller") { }
    
    protected override UhtClass GetMetaClass(UhtObjectPropertyBase property)
    {
        UhtClassProperty classProperty = (UhtClassProperty)property;
        return classProperty.MetaClass!;
    }
}

// TSoftObjectPtr<UActor> → TSoftObjectPtr<AActor>
public class SoftObjectPropertyTranslator : ObjectContainerPropertyTranslator
{
    public SoftObjectPropertyTranslator() 
        : base(typeof(UhtSoftObjectProperty), "TSoftObjectPtr", "SoftObjectMarshaller") { }
    
    protected override UhtClass GetMetaClass(UhtObjectPropertyBase property)
    {
        UhtSoftObjectProperty softObjectProperty = (UhtSoftObjectProperty)property;
        return softObjectProperty.Class;
    }
}
```

## PropertyTranslatorManager：翻译器注册与匹配

### 链式匹配机制

```csharp
public static class PropertyTranslatorManager
{
    private static readonly Dictionary<Type, List<PropertyTranslator>?> RegisteredTranslators = new();
    
    static PropertyTranslatorManager()
    {
        // 按优先级注册翻译器
        
        // 1. 枚举类型（优先级最高）
        EnumPropertyTranslator enumPropertyTranslator = new();
        AddPropertyTranslator(typeof(UhtEnumProperty), enumPropertyTranslator);
        AddPropertyTranslator(typeof(UhtByteProperty), enumPropertyTranslator);
        
        // 2. Blittable 数值类型
        AddBlittablePropertyTranslator(typeof(UhtInt8Property), "sbyte");
        AddBlittablePropertyTranslator(typeof(UhtInt16Property), "short");
        AddBlittablePropertyTranslator(typeof(UhtInt64Property), "long");
        AddBlittablePropertyTranslator(typeof(UhtUInt16Property), "ushort");
        AddBlittablePropertyTranslator(typeof(UhtUInt32Property), "uint");
        AddBlittablePropertyTranslator(typeof(UhtUInt64Property), "ulong");
        AddBlittablePropertyTranslator(typeof(UhtDoubleProperty), "double");
        AddBlittablePropertyTranslator(typeof(UhtByteProperty), "byte");
        
        // 3. 特殊处理类型
        AddPropertyTranslator(typeof(UhtFloatProperty), new FloatPropertyTranslator());
        AddPropertyTranslator(typeof(UhtIntProperty), new IntPropertyTranslator());
        
        // 4. 布尔类型
        AddPropertyTranslator(typeof(UhtBoolProperty), new BoolPropertyTranslator());
        
        // 5. 字符串类型
        AddPropertyTranslator(typeof(UhtStrProperty), new StringPropertyTranslator());
        AddPropertyTranslator(typeof(UhtNameProperty), new NamePropertyTranslator());
        AddPropertyTranslator(typeof(UhtTextProperty), new TextPropertyTranslator());
        
        // 6. 对象引用类型（多种实现）
        AddPropertyTranslator(typeof(UhtWeakObjectPtrProperty), new WeakObjectPropertyTranslator());
        AddPropertyTranslator(typeof(UhtObjectProperty), new ObjectPropertyTranslator());
        AddPropertyTranslator(typeof(UhtInterfaceProperty), new InterfacePropertyTranslator());
        
        // 7. 类引用
        AddPropertyTranslator(typeof(UhtClassProperty), new SubclassOfPropertyTranslator());
        AddPropertyTranslator(typeof(UhtSoftClassProperty), new SoftClassPropertyTranslator());
        AddPropertyTranslator(typeof(UhtSoftObjectProperty), new SoftObjectPropertyTranslator());
        
        // 8. 容器类型
        AddPropertyTranslator(typeof(UhtArrayProperty), new ContainerPropertyTranslator(...));
        AddPropertyTranslator(typeof(UhtMapProperty), new ContainerPropertyTranslator(...));
        AddPropertyTranslator(typeof(UhtSetProperty), new ContainerPropertyTranslator(...));
        AddPropertyTranslator(typeof(UhtOptionalProperty), new ContainerPropertyTranslator(...));
        
        // 9. 结构体类型（优先级最低，最后匹配）
        AddPropertyTranslator(typeof(UhtStructProperty), new BlittableStructPropertyTranslator());
        AddPropertyTranslator(typeof(UhtStructProperty), new StructPropertyTranslator());
    }
    
    public static PropertyTranslator? GetTranslator(this UhtProperty property)
    {
        if (!RegisteredTranslators.TryGetValue(property.GetType(), out var translators))
            return null;
        
        // 遍历所有注册的翻译器，返回第一个 CanExport 返回 true 的
        foreach (var translator in translators!)
        {
            if (translator.CanExport(property))
                return translator;
        }
        return null;
    }
    
    static void AddPropertyTranslator(Type propertyClass, PropertyTranslator translator)
    {
        if (RegisteredTranslators.TryGetValue(propertyClass, out var translators))
        {
            translators!.Add(translator);
            return;
        }
        RegisteredTranslators.Add(propertyClass, new List<PropertyTranslator> { translator });
    }
}
```

### 匹配示例

```
UhtStructProperty (FVector)
    → BlittableStructPropertyTranslator.CanExport() → true ✓
    
UhtStructProperty (FHitResult)  
    → BlittableStructPropertyTranslator.CanExport() → false
    → StructPropertyTranslator.CanExport() → true ✓
    
UhtObjectProperty (AActor*)
    → ObjectPropertyTranslator.CanExport() → true ✓
    
UhtInterfaceProperty (IInterface*)
    → InterfacePropertyTranslator.CanExport() → true ✓
```

## 自定义类型扩展

通过 JSON 配置扩展类型映射：

```json
// Config/DefaultEngine.UnrealSharpTypes.json
{
  "Structs": {
    "BlittableTypes": [
      {
        "Name": "FVector",
        "ManagedType": "System.Numerics.Vector3",
        "Attributes": {
          "UserSpecifiedLayout": "true"
        }
      },
      {
        "Name": "FVector2D",
        "ManagedType": "System.Numerics.Vector2"
      },
      {
        "Name": "FQuat",
        "ManagedType": "System.Numerics.Quaternion"
      }
    ],
    "NativelyTranslatableTypes": [
      {
        "Name": "FHitResult",
        "HasDestructor": true
      },
      {
        "Name": "FFrameRate",
        "HasDestructor": false
      }
    ],
    "CustomTypes": [
      "FReplicationEntry",
      "FActorDestructionInfo"
    ]
  }
}
```

配置加载后自动生成对应的 `BlittableCustomStructTypePropertyTranslator`：

```csharp
foreach (BlittableStructInfo managedType in SpecialTypeInfo.Structs.BlittableTypes.Values)
{
    if (managedType.ManagedType is not null)
    {
        AddPropertyTranslator(typeof(UhtStructProperty), 
            new BlittableCustomStructTypePropertyTranslator(managedType.Name, managedType.ManagedType));
    }
}
```

## 总结

PropertyTranslator 是 UnrealSharp 类型系统的核心，其设计亮点：

| 特性 | 实现方式 |
|-----|---------|
| **策略模式** | 每种类型一个翻译器，职责单一 |
| **链式匹配** | 按优先级遍历，灵活扩展 |
| **模板方法** | 基类定义流程，子类填充细节 |
| **配置驱动** | JSON 扩展类型映射，无需修改代码 |
| **增量 Marshaller** | 容器类型懒加载 Marshaller 实例 |

**关键类型处理策略**：

| 类型 | 翻译器 | 特殊处理 |
|-----|--------|---------|
| Blittable 类型 | `BlittableTypePropertyTranslator` | 直接内存拷贝 |
| bool | `BoolPropertyTranslator` | 位域掩码处理 |
| string | `StringPropertyTranslator` | UTF-16 转换、内存管理 |
| UObject* | `ObjectPropertyTranslator` | GCHandle 映射 |
| TArray/TMap/TSet | `ContainerPropertyTranslator` | Marshaller 委托、接口包装 |
| 结构体 | `BlittableStructPropertyTranslator` / `StructPropertyTranslator` | 配置驱动的 Blittable 判断 |
| 委托 | `MulticastDelegatePropertyTranslator` | BackingField 缓存、懒加载 |

下一篇文章我们将深入 **UE5 反射系统的动态类型注册**，看看 C# 类型是如何被注册到 UE 反射系统中的。

---

**系列导航**：
- [第一篇：概述与架构总览](/posts/UnrealSharp-01-概述与架构总览/)
- [第二篇：运行时初始化](/posts/UnrealSharp-02-运行时初始化/)
- [第三篇：GCHandle与对象生命周期](/posts/UnrealSharp-03-GCHandle与对象生命周期/)
- [第四篇：托管回调系统](/posts/UnrealSharp-04-托管回调系统/)
- [第五篇：Glue代码生成系统](/posts/UnrealSharp-05-Glue代码生成系统/)
- 第六篇：PropertyTranslator与类型映射（本文）
