---
title: "Special Attributes and Types"
weight: 330
---

This chapter describes attributes and types that have special significance to the Clef compiler.

> **Clef Note**: Clef does not use CLI assemblies or the .NET runtime. Attributes related to assembly metadata, P/Invoke interop, serialization, and runtime reflection are not applicable. This chapter covers only those attributes meaningful for native compilation.

## Custom Attributes Recognized by Clef

The following custom attributes have special meanings recognized by the Clef compiler.

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
| `[<Struct>]` | Indicates that a type is a struct type with value semantics. In Clef, struct types have deterministic stack or arena allocation. |
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
| `[<DefaultValue(...)>]` | When added to a field declaration, specifies that the field should be zero-initialized. In Clef, this initializes to the zero bit pattern for the field's type. |

> **Clef Note**: The `StructLayout` and `FieldOffset` attributes are critical for defining types that must match specific memory layouts, such as hardware register descriptors or wire protocol structures. See [Memory Regions](memory-regions.md) for memory placement semantics.

### Inline and Optimization Attributes

| Attribute/Keyword | Description |
| --- | --- |
| `inline` | The `inline` keyword on function definitions enables body expansion at call sites. See below for escape analysis semantics. |
| `[<InlineIfLambda>]` | Indicates that a lambda argument should be inlined at call sites for performance. |
| `[<NoDynamicInvocation>]` | When applied to an inline function or member definition, indicates that the function cannot be invoked dynamically. In Clef, this is the default behavior since there is no dynamic invocation. |

#### Inline Functions and Escape Analysis

In Clef, the `inline` keyword has additional semantic significance beyond performance optimization. When a function is marked `inline`, CCS captures its body for **transparent expansion** at call sites. This is critical for **escape analysis** of stack-allocated memory.

**The Escape Problem**: When a function allocates memory via `NativePtr.stackalloc` and returns a pointer to that memory, the pointer becomes invalid when the function returns (the stack frame is deallocated).

```fsharp
// WITHOUT inline - pointer escapes and dangles
let readln () : string =
    let buffer = NativePtr.stackalloc<byte> 256  // Allocated in readln's frame
    let len = readLineInto buffer 256
    NativeStr.fromPointer buffer len             // Returns pointer to readln's stack!
    // When readln returns, buffer is deallocated - pointer is now INVALID

let hello() =
    let name = readln()  // name points to deallocated memory!
    greet name           // Undefined behavior
```

**The Solution**: Marking the function `inline` causes CCS to expand the function body at the call site, lifting the allocation to the caller's frame:

```fsharp
// WITH inline - allocation lifted to caller's frame
let inline readln () : string =
    let buffer = NativePtr.stackalloc<byte> 256
    let len = readLineInto buffer 256
    NativeStr.fromPointer buffer len

let hello() =
    // AFTER inline expansion, semantically becomes:
    let buffer = NativePtr.stackalloc<byte> 256  // Now in hello's frame!
    let len = readLineInto buffer 256
    let name = NativeStr.fromPointer buffer len  // Pointer valid through hello's scope
    greet name                                    // Safe - hello's frame is alive
```

**When to Use `inline` for Escape Analysis**:

Functions should be marked `inline` when they:
1. Allocate memory via `NativePtr.stackalloc` or `Arena.alloc`
2. Return a pointer, reference, or fat pointer (like `string`) to that memory
3. The caller needs the returned value to remain valid

This pattern is common in platform libraries (e.g., `Console.readln`) where the implementation detail of stack allocation should be transparent to application code.

> **Design Note**: This mechanism supports Level 1 (Implicit) memory management from the [Memory Regions](memory-regions.md) design - developers write standard F# code while the compiler ensures memory safety through inline expansion.

### Entry Point Attribute

| Attribute | Description |
| --- | --- |
| `[<EntryPoint>]` | Indicates that a function is the program's entry point. The function must have type `array<string> -> int`. Only one function in the last compilation file may have this attribute. See [Program Structure and Execution](program-structure-and-execution.md#explicit-entry-point). |

### Platform Binding Notes

> **Clef Note**: Clef does not use `DllImport` or P/Invoke. Platform operations use **CCS intrinsics** (`Sys.write`, `NativePtr.set`, etc.) which are recognized by module pattern and compiled to platform-specific code. External library bindings use **quotation semantic carriers**. See [Platform Bindings](platform-bindings.md).

### Memory Region Attributes

These attributes control memory placement in Clef:

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

## Custom Attributes Emitted by Clef

The Clef compiler emits the following information as part of compilation:

| Information | Description |
| --- | --- |
| Debug symbols | Source location information for debugging, emitted in platform-native debug format (DWARF on Linux/macOS, PDB on Windows). |
| Compilation mapping | Metadata indicating how compiled constructs correspond to F# source constructs. |

> **Clef Note**: Debug information is provided through native debug formats (DWARF on Linux/macOS, PDB on Windows).

## Attributes Not Applicable to Clef

The following attribute categories from managed F# are **not applicable** to Clef compilation:

### Assembly Attributes

Assembly metadata attributes (`AssemblyVersion`, `AssemblyTitle`, `AssemblyCompany`, etc.) are not applicable as Clef produces native binaries, not CLI assemblies.

### P/Invoke and Interop Attributes

| Not Applicable | Reason |
| --- | --- |
| `[<DllImport(...)>]` | P/Invoke is a CLI mechanism. Use CCS intrinsics (`Sys.*`) or quotation-based binding libraries instead. See [Platform Bindings](platform-bindings.md). |
| `[<MarshalAs(...)>]` | CLI marshalling is not applicable. Clef types have deterministic native representations. |
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
| `[<ReflectedDefinition>]` | Runtime quotation access. Clef has no runtime reflection. |
| `[<TypeProviderXmlDocAttribute>]` | Type providers require CLI runtime. |
| `[<TypeProviderDefinitionLocationAttribute>]` | Type providers require CLI runtime. |
| `[<TypeForwardedTo(...)>]` | CLI type forwarding mechanism. |

### Threading Attributes

| Not Applicable | Reason |
| --- | --- |
| `[<ThreadStatic>]` | CLI thread-local storage. Use platform-specific TLS mechanisms. |
| `[<ContextStatic>]` | CLI context-local storage. |

## Error Conditions in Clef

> **Clef Note**: Clef uses the `Result<'T, 'E>` type for explicit error handling rather than exceptions. Operations that can fail return `Result` values. The following describes how traditional exception scenarios are handled.

### Arithmetic Errors

| Condition | Clef Handling |
| --- | --- |
| Division by zero (integer) | Returns `Error DivisionByZero` or causes hardware trap depending on platform configuration. |
| Arithmetic overflow (checked context) | Returns `Error Overflow` in checked arithmetic operations. Unchecked operations wrap. |

### Array and Memory Errors

| Condition | Clef Handling |
| --- | --- |
| Index out of bounds | Returns `Error IndexOutOfBounds` or causes program termination depending on bounds-checking configuration. |
| Null reference | **Cannot occur** - Clef is null-free by construction. All references are valid. |
| Out of memory | Platform-dependent behavior. Stack allocation may cause stack overflow. Arena allocation failures are explicit. |

### Type Errors

| Condition | Clef Handling |
| --- | --- |
| Invalid cast | **Cannot occur** - All type conversions are verified at compile time. |
| Type mismatch | **Cannot occur** - Type system prevents type mismatches statically. |

### Stack Overflow

Stack overflow can occur with deeply recursive functions. Clef provides:
- Tail call optimization to prevent stack growth for tail-recursive functions
- Compile-time analysis to warn about potentially unbounded recursion
- Platform-specific stack size configuration

> **Clef Note**: The absence of `NullReferenceException`, `InvalidCastException`, and similar runtime type errors is a fundamental property of Clef's null-free, statically verified type system.

## Platform-Specific Types

Clef provides the following types for platform interaction:

| Type | Description |
| --- | --- |
| `nativeint` | Platform-sized signed integer (32 bits on 32-bit platforms, 64 bits on 64-bit platforms). |
| `unativeint` | Platform-sized unsigned integer. |
| `nativeptr<'T>` | Typed native pointer. |
| `voidptr` | Untyped native pointer. |

### Pointer Types with Access Kinds

Clef extends pointer types with access kind annotations:

```fsharp
type Ptr<'T, 'Region, 'Access> = ...
```

Where:
- `'T` is the pointed-to type
- `'Region` is a phantom type indicating memory region (`stack`, `arena`, `peripheral`, `sram`, `flash`)
- `'Access` is a phantom type indicating access permissions (`readOnly`, `writeOnly`, `readWrite`)

See [Access Kinds](access-kinds.md) for details on access annotations.
