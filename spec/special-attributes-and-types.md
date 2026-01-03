# Special Attributes and Types

This chapter describes attributes and types that have special significance to the F# Native compiler.

> **F# Native Note**: F# Native does not use CLI assemblies or the .NET runtime. Attributes related to assembly metadata, P/Invoke interop, serialization, and runtime reflection are not applicable. This chapter covers only those attributes meaningful for native compilation.

## Custom Attributes Recognized by F# Native

The following custom attributes have special meanings recognized by the F# Native compiler.

### F# Language Attributes

These attributes control F# language semantics and are fully supported:

| Attribute | Description |
| --- | --- |
| `[<Obsolete(...)>]` | Indicates that the construct is obsolete and gives a warning or error depending on the settings in the attribute. |
| `[<Conditional(...)>]` | Emits code to call the method only if the corresponding conditional compilation symbol is defined. Conditional compilation uses the `FIDELITY` symbol for native compilation. |
| `[<AutoOpen>]` | When applied to a module, causes the module to be opened automatically when the enclosing namespace or module is opened. When applied with a string argument at the compilation unit level, causes the named namespace or module to be opened automatically. |
| `[<CompiledName(...)>]` | Changes the compiled name of an F# language construct. |
| `[<CompilationRepresentation(...)>]` | Adjusts the compiled representation of a type. |
| `[<CustomComparison>]` | When applied to an F# structural type, indicates that the type has a user-specified comparison implementation. |
| `[<CustomEquality>]` | When applied to an F# structural type, indicates that the type has a user-defined equality implementation. |
| `[<DefaultAugmentation(...)>]` | When applied to an F# discriminated union type with value false, turns off the generation of standard helper member tester, constructor and accessor members. |
| `[<GeneralizableValue>]` | When applied to an F# value, indicates that uses of the attribute can result in generic code through the process of type inference. The value must typically be a type function whose implementation has no observable side effects. |
| `[<Literal>]` | When applied to a value, compiles the value as a compile-time literal constant. |
| `[<CompilerMessage(...)>]` | When applied to an F# construct, indicates that the F# compiler should report a message when the construct is used. |
| `[<Struct>]` | Indicates that a type is a struct type with value semantics. In F# Native, struct types have deterministic stack or arena allocation. |
| `[<Class>]` | Indicates that a type is a class type. |
| `[<Interface>]` | Indicates that a type is an interface type. |
| `[<Measure>]` | Indicates that a type or generic parameter is a unit of measure definition or annotation. Units of measure are erased at compile time. |
| `[<ReferenceEquality>]` | When applied to an F# record or union type, indicates that the type should use reference equality for its default equality implementation. |
| `[<RequireQualifiedAccess>]` | When applied to an F# module, warns if an attempt is made to open the module name. When applied to an F# union or record type, indicates that the field labels or union cases must be referenced by using a qualified path that includes the type name. |
| `[<RequiresExplicitTypeArguments>]` | When applied to an F# function or method, indicates that the function or method must be invoked with explicit type arguments. |
| `[<StructuralComparison>]` | When added to a record, union, exception, or structure type, confirms the automatic generation of structural comparison. |
| `[<StructuralEquality>]` | When added to a record, union, or struct type, confirms the automatic generation of structural equality. |
| `[<NoComparison>]` | When applied to a type, suppresses automatic generation of comparison operations. |
| `[<NoEquality>]` | When applied to a type, suppresses automatic generation of equality operations. |

### Memory and Layout Attributes

These attributes control memory layout for native compilation:

| Attribute | Description |
| --- | --- |
| `[<StructLayout(...)>]` | Specifies the memory layout of a struct type. Supports `LayoutKind.Sequential` (default) and `LayoutKind.Explicit` for precise field placement. |
| `[<FieldOffset(...)>]` | When applied to a field within a struct with explicit layout, specifies the byte offset of the field from the start of the struct. |
| `[<VolatileField>]` | When applied to a mutable field, indicates that accesses to the field should use volatile memory semantics. Essential for memory-mapped I/O and peripheral access. |
| `[<DefaultValue(...)>]` | When added to a field declaration, specifies that the field should be zero-initialized. In F# Native, this initializes to the zero bit pattern for the field's type. |

> **F# Native Note**: The `StructLayout` and `FieldOffset` attributes are critical for defining types that must match specific memory layouts, such as hardware register descriptors or wire protocol structures. See [Memory Regions](memory-regions.md) for memory placement semantics.

### Inline and Optimization Attributes

| Attribute | Description |
| --- | --- |
| `[<InlineIfLambda>]` | Indicates that a lambda argument should be inlined at call sites for performance. |
| `[<NoDynamicInvocation>]` | When applied to an inline function or member definition, indicates that the function cannot be invoked dynamically. In F# Native, this is the default behavior since there is no dynamic invocation. |

### Entry Point Attribute

| Attribute | Description |
| --- | --- |
| `[<EntryPoint>]` | Indicates that a function is the program's entry point. The function must have type `array<string> -> int`. Only one function in the last compilation file may have this attribute. See [Program Structure and Execution](program-structure-and-execution.md#explicit-entry-point). |

### Platform Binding Notes

> **F# Native Note**: F# Native does not use `DllImport` or P/Invoke. Platform operations use **FNCS intrinsics** (`Sys.write`, `NativePtr.set`, etc.) which are recognized by module pattern and compiled to platform-specific code. External library bindings use **quotation semantic carriers**. See [Platform Bindings](platform-bindings.md).

### Memory Region Attributes

These attributes control memory placement in F# Native:

| Attribute | Description |
| --- | --- |
| `[<PeripheralDescriptor(name, baseAddress)>]` | Marks a record type as describing memory-mapped peripheral registers. The compiler ensures appropriate volatile access semantics. |
| `[<Register(name, offset, access)>]` | Within a peripheral descriptor, specifies register metadata including offset from base address and access kind ("r", "w", or "rw"). |

Example of peripheral descriptor usage:

```fsharp
[<PeripheralDescriptor("GPIO", 0x48000000UL)>]
type GPIO_TypeDef = {
    [<Register("MODER", 0x00u, "rw")>]
    MODER: Ptr<uint32, peripheral, readWrite>

    [<Register("IDR", 0x10u, "r")>]
    IDR: Ptr<uint32, peripheral, readOnly>

    [<Register("ODR", 0x14u, "rw")>]
    ODR: Ptr<uint32, peripheral, readWrite>
}
```

## Custom Attributes Emitted by F# Native

The F# Native compiler emits the following information as part of compilation:

| Information | Description |
| --- | --- |
| Debug symbols | Source location information for debugging, emitted in platform-native debug format (DWARF on Linux/macOS, PDB on Windows). |
| Compilation mapping | Metadata indicating how compiled constructs correspond to F# source constructs. |

> **F# Native Note**: Unlike managed F#, the F# Native compiler does not emit CLI metadata attributes. Debug information is provided through native debug formats.

## Attributes Not Applicable to F# Native

The following attribute categories from managed F# are **not applicable** to F# Native compilation:

### Assembly Attributes

Assembly metadata attributes (`AssemblyVersion`, `AssemblyTitle`, `AssemblyCompany`, etc.) are not applicable as F# Native produces native binaries, not CLI assemblies.

### P/Invoke and Interop Attributes

| Not Applicable | Reason |
| --- | --- |
| `[<DllImport(...)>]` | P/Invoke is a CLI mechanism. Use FNCS intrinsics (`Sys.*`) or quotation-based binding libraries instead. See [Platform Bindings](platform-bindings.md). |
| `[<MarshalAs(...)>]` | CLI marshalling is not applicable. F# Native types have deterministic native representations. |
| `[<In>]`, `[<Out>]` | CLI parameter direction attributes. Not needed for native calling conventions. |
| `[<UnmanagedFunctionPointer>]` | CLI interop mechanism. Native function pointers are used directly. |

### Serialization Attributes

| Not Applicable | Reason |
| --- | --- |
| `[<Serializable>]` | CLI serialization. For binary serialization, see BAREWire. |
| `[<NonSerialized>]` | CLI serialization marker. |
| `[<AutoSerializable(false)>]` | CLI serialization control. |

### Reflection and Type Provider Attributes

| Not Applicable | Reason |
| --- | --- |
| `[<ReflectedDefinition>]` | Runtime quotation access. F# Native has no runtime reflection. |
| `[<TypeProviderXmlDocAttribute>]` | Type providers require CLI runtime. |
| `[<TypeProviderDefinitionLocationAttribute>]` | Type providers require CLI runtime. |
| `[<TypeForwardedTo(...)>]` | CLI type forwarding mechanism. |

### Threading Attributes

| Not Applicable | Reason |
| --- | --- |
| `[<ThreadStatic>]` | CLI thread-local storage. Use platform-specific TLS mechanisms. |
| `[<ContextStatic>]` | CLI context-local storage. |

## Error Conditions in F# Native

> **F# Native Note**: F# Native uses the `Result<'T, 'E>` type for explicit error handling rather than exceptions. Operations that can fail return `Result` values. The following describes how traditional exception scenarios are handled.

### Arithmetic Errors

| Condition | F# Native Handling |
| --- | --- |
| Division by zero (integer) | Returns `Error DivisionByZero` or causes hardware trap depending on platform configuration. |
| Arithmetic overflow (checked context) | Returns `Error Overflow` in checked arithmetic operations. Unchecked operations wrap. |

### Array and Memory Errors

| Condition | F# Native Handling |
| --- | --- |
| Index out of bounds | Returns `Error IndexOutOfBounds` or causes program termination depending on bounds-checking configuration. |
| Null reference | **Cannot occur** - F# Native is null-free by construction. All references are valid. |
| Out of memory | Platform-dependent behavior. Stack allocation may cause stack overflow. Arena allocation failures are explicit. |

### Type Errors

| Condition | F# Native Handling |
| --- | --- |
| Invalid cast | **Cannot occur** - All type conversions are verified at compile time. |
| Type mismatch | **Cannot occur** - Type system prevents type mismatches statically. |

### Stack Overflow

Stack overflow can occur with deeply recursive functions. F# Native provides:
- Tail call optimization to prevent stack growth for tail-recursive functions
- Compile-time analysis to warn about potentially unbounded recursion
- Platform-specific stack size configuration

> **F# Native Note**: The absence of `NullReferenceException`, `InvalidCastException`, and similar runtime type errors is a fundamental property of F# Native's null-free, statically verified type system.

## Platform-Specific Types

F# Native provides the following types for platform interaction:

| Type | Description |
| --- | --- |
| `nativeint` | Platform-sized signed integer (32 bits on 32-bit platforms, 64 bits on 64-bit platforms). |
| `unativeint` | Platform-sized unsigned integer. |
| `nativeptr<'T>` | Typed native pointer. |
| `voidptr` | Untyped native pointer. |

### Pointer Types with Access Kinds

F# Native extends pointer types with access kind annotations:

```fsharp
type Ptr<'T, 'Region, 'Access> = ...
```

Where:
- `'T` is the pointed-to type
- `'Region` is a phantom type indicating memory region (`stack`, `arena`, `peripheral`, `sram`, `flash`)
- `'Access` is a phantom type indicating access permissions (`readOnly`, `writeOnly`, `readWrite`)

See [Access Kinds](access-kinds.md) for details on access annotations.
