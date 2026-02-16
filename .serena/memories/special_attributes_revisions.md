# Special Attributes and Types - Revision Decisions

## Overview

The `spec/special-attributes-and-types.md` chapter was extensively revised to remove BCL/CLI dependencies and establish native-focused attribute semantics.

## Key Changes

### Removed Entirely

1. **Assembly Metadata Attributes**: All `System.Reflection.Assembly*` attributes removed (AssemblyVersion, AssemblyTitle, AssemblyCompany, etc.) - Clef produces native binaries, not CLI assemblies

2. **P/Invoke Attributes**: `DllImport`, `MarshalAs`, `In`, `Out`, `UnmanagedFunctionPointer` - replaced by Platform.Bindings module convention

3. **Serialization Attributes**: `Serializable`, `NonSerialized`, `AutoSerializable` - CLI serialization not applicable; BAREWire for binary serialization

4. **Reflection Attributes**: `ReflectedDefinition`, TypeProvider attributes - no runtime reflection in Clef

5. **Threading Attributes**: `ThreadStatic`, `ContextStatic` - CLI thread-local storage; use platform-specific TLS mechanisms

6. **CLI-Specific Runtime Attributes**: `TypeForwardedTo`, `Extension`, `CallerFilePath`, `CallerLineNumber`, `CallerMemberName`, etc.

### Retained and Adapted

1. **F# Language Attributes** (fully supported):
   - `Obsolete`, `Conditional`, `AutoOpen`, `CompiledName`
   - `Struct`, `Class`, `Interface`, `Measure`
   - `CustomComparison`, `CustomEquality`, `StructuralComparison`, `StructuralEquality`
   - `ReferenceEquality`, `RequireQualifiedAccess`, `RequiresExplicitTypeArguments`
   - `Literal`, `GeneralizableValue`, `CompilerMessage`
   - `NoComparison`, `NoEquality`

2. **Memory Layout Attributes**:
   - `StructLayout` - for precise memory layout control
   - `FieldOffset` - for explicit field placement
   - `VolatileField` - for memory-mapped I/O semantics
   - `DefaultValue` - for zero-initialization

### Added for Clef

1. **Platform Binding Attribute**: `[<PlatformBinding>]` - marks functions in Platform.Bindings for compiler-provided implementation

2. **Memory Region Attributes**:
   - `[<PeripheralDescriptor(name, baseAddress)>]` - marks peripheral register descriptor types
   - `[<Register(name, offset, access)>]` - specifies register metadata within descriptors

3. **Inline Attributes**: `InlineIfLambda`, `NoDynamicInvocation` retained with native semantics

4. **Entry Point**: `[<EntryPoint>]` retained with cross-reference to program-structure-and-execution.md

## Exception to Error Handling Transformation

Replaced the entire "Exceptions Thrown by F# Language Primitives" section with "Error Conditions in Clef":

| Original Exception | Clef Handling |
|--------------------|-------------------|
| `DivideByZeroException` | `Error DivisionByZero` or hardware trap |
| `OverflowException` | `Error Overflow` in checked context; wrap in unchecked |
| `IndexOutOfRangeException` | `Error IndexOutOfBounds` or program termination |
| `NullReferenceException` | **Cannot occur** - null-free by construction |
| `InvalidCastException` | **Cannot occur** - compile-time verified |
| `OutOfMemoryException` | Platform-dependent; arena failures explicit |
| `StackOverflowException` | TCO prevents; compile-time warnings; platform config |

## Platform-Specific Types Section

Added section documenting:
- `nativeint`, `unativeint` - platform-sized integers
- `nativeptr<'T>`, `voidptr` - native pointer types
- `Ptr<'T, 'Region, 'Access>` - typed pointers with region and access annotations

## Cross-References Added

- Link to `memory-regions.md` for memory placement semantics
- Link to `access-kinds.md` for access annotations
- Link to `platform-bindings.md` for Platform.Bindings pattern
- Link to `program-structure-and-execution.md` for EntryPoint semantics

## Design Rationale

1. **Null-freedom emphasized**: Explicit note that NullReferenceException cannot occur
2. **Result-based error handling**: Exceptions replaced with Result type pattern
3. **Platform.Bindings convention**: Replaces DllImport/P/Invoke entirely
4. **Peripheral descriptors**: Native-specific feature for hardware access
5. **Compile-time verification**: Many runtime errors simply cannot occur
