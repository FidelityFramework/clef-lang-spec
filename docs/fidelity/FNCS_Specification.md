# F# Native Language Specification

## Overview

This document specifies the **F# Native** dialect - extensions and modifications to standard F# semantics for native compilation via the Fidelity framework. F# Native is implemented by FNCS (F# Native Compiler Services) and compiled to native binaries by Firefly.

**Relationship to Standard F#**: F# Native is a superset of F# syntax with modified type semantics. Valid F# Native code parses identically to standard F#, but type resolution follows native rules.

## Companion Documents

- `fsnative/docs/FNCS_Pruning_Plan.md` - Implementation plan for FNCS
- `Firefly/docs/FNCS_Architecture.md` - Integration with Firefly
- `fslang-spec/spec/*` - Standard F# specification (reference)

---

## Part 1: Native Type Universe

### 1.1 Primitive Type Mapping

F# Native uses Alloy's native types instead of BCL types:

| F# Syntax | Standard F# | F# Native |
|-----------|-------------|-----------|
| `int` | `System.Int32` | `Alloy.Int32` |
| `int64` | `System.Int64` | `Alloy.Int64` |
| `float` | `System.Double` | `Alloy.Float64` |
| `float32` | `System.Single` | `Alloy.Float32` |
| `string` | `System.String` | `Alloy.NativeStr` |
| `char` | `System.Char` | `Alloy.Char` (UTF-8 code point) |
| `bool` | `System.Boolean` | `Alloy.Bool` |
| `unit` | `Microsoft.FSharp.Core.Unit` | `Alloy.Unit` |
| `byte` | `System.Byte` | `Alloy.UInt8` |

### 1.2 String Literals

**Standard F#**: String literals have type `System.String`.

**F# Native**: String literals have type `NativeStr`.

```fsharp
// F# Native semantics
let greeting = "Hello"  // Type: NativeStr, not System.String
```

`NativeStr` is a fat pointer to UTF-8 encoded bytes:

```fsharp
[<Struct>]
type NativeStr = {
    Ptr: nativeptr<byte>
    Length: int
}
```

**Implications**:
- No null strings (fat pointer is always valid or zero-length)
- UTF-8 encoding (not UTF-16)
- Known length (no null terminator scanning)
- Stack or arena allocated (no GC)

### 1.3 Option Types

**Standard F#**: `option<'T>` is a reference type, `None` is null.

**F# Native**: `option<'T>` maps to `voption<'T>` (value option).

```fsharp
// F# Native semantics
let maybeValue: int option = Some 42  // Type: voption<int>
let nothing: int option = None        // Type: voption<int>, not null
```

**Implications**:
- No null representation
- Stack allocated
- Pattern matching works identically

### 1.4 Array Types

**Standard F#**: `'T[]` is `System.Array` (heap allocated, GC managed).

**F# Native**: `'T[]` is `NativeArray<'T>` (fat pointer).

```fsharp
// F# Native semantics
let numbers = [| 1; 2; 3 |]  // Type: NativeArray<int>
```

```fsharp
[<Struct>]
type NativeArray<'T> = {
    Ptr: nativeptr<'T>
    Length: int
    Capacity: int
}
```

**Implications**:
- Explicit memory management (stack, arena, or explicit allocation)
- No automatic resizing
- Bounds checking preserved

---

## Part 2: Null-Free Semantics

### 2.1 Null Prohibition

F# Native enforces null-free semantics for native types:

```fsharp
// COMPILE ERROR in F# Native
let s: NativeStr = null        // Error: Cannot assign null to NativeStr
let arr: NativeArray<int> = null // Error: Cannot assign null to NativeArray
```

### 2.2 Interop Boundary

When interfacing with platform APIs that may return null:

```fsharp
// Platform binding returns nullable
[<PlatformBinding>]
let tryGetEnv (name: NativeStr) : NativeStr voption = ...

// Usage - must handle None case
match tryGetEnv "PATH" with
| ValueSome path -> use path
| ValueNone -> handle missing
```

### 2.3 Default Values

Native types have sensible defaults, not null:

| Type | Default |
|------|---------|
| `NativeStr` | Empty string (zero-length) |
| `NativeArray<'T>` | Empty array (zero-length) |
| `voption<'T>` | `ValueNone` |
| Numeric | `0` |
| `bool` | `false` |

---

## Part 3: SRTP Resolution

### 3.1 Witness Hierarchy

Standard F# SRTP resolves against .NET method tables. F# Native SRTP resolves against the **Alloy witness hierarchy**.

**Alloy Witness Chain** (searched in order):
1. Concrete type's members
2. `BasicOps` (primitive operations)
3. `NumericOps` (numeric operations)
4. `CollectionOps` (collection operations)
5. `ComparableOps` (comparison operations)

### 3.2 Resolution Example

```fsharp
// SRTP constraint
let inline add (a: ^T) (b: ^T) : ^T
    when ^T : (static member (+) : ^T * ^T -> ^T) =
    a + b

// Standard F# resolution for `add 1 2`:
// 1. Look for System.Int32.op_Addition
// 2. Found in BCL

// F# Native resolution for `add 1 2`:
// 1. Look for Alloy.Int32 members - not found
// 2. Look in BasicOps - found: BasicOps.Add<int>
// 3. Resolved witness: BasicOps, method: Add
```

### 3.3 Alloy Operator Resolution

Alloy defines operators like `$` for string operations:

```fsharp
// Alloy definition
type WritableString =
    static member inline ($) (ws: WritableString, s: NativeStr) : unit = ...

// Usage
WritableString $ "Hello"

// F# Native resolution:
// 1. TraitCall for op_Dollar
// 2. Search WritableString members
// 3. Found: WritableString.op_Dollar
// 4. Resolution includes: witness type, method, coeffect
```

### 3.4 Resolution Metadata

F# Native SRTP resolution captures additional metadata:

```fsharp
type SRTPResolution = {
    WitnessType: FidType       // e.g., BasicOps
    Method: string             // e.g., "Add"
    Coeffect: Coeffect         // e.g., Pure
    OwnershipTransfer: bool    // Does this consume arguments?
    MemoryLayout: LayoutInfo   // Alignment, size
}
```

This metadata flows to Firefly's PSG for code generation.

---

## Part 4: Memory Semantics

### 4.1 Stack Allocation

F# Native defaults to stack allocation for value types:

```fsharp
let point = { X = 1.0; Y = 2.0 }  // Stack allocated
let numbers = [| 1; 2; 3 |]        // Stack allocated (small)
```

### 4.2 Arena Allocation (Future)

Large or dynamically-sized data uses arena allocation:

```fsharp
arena {
    let buffer = NativeArray.create 1_000_000  // Arena allocated
    let result = process buffer
    return result  // Only result escapes
}  // Arena freed here
```

### 4.3 Ownership Types (Future)

F# Native will support ownership annotations:

```fsharp
// Owned value - caller receives ownership
let createBuffer () : Owned<NativeArray<byte>> = ...

// Borrowed reference - caller borrows, doesn't own
let processBuffer (buf: Borrowed<NativeArray<byte>>) : unit = ...

// Move semantics
let newOwner = move existingBuffer
```

---

## Part 5: Coeffects (Future)

### 5.1 Coeffect Annotations

Functions can declare their effects:

```fsharp
// Pure function - no side effects
let add (a: int) (b: int) : int -[Pure]-> int = a + b

// IO function - performs I/O
let readFile (path: NativeStr) : NativeArray<byte> -[IO.File]-> NativeArray<byte> = ...

// Composite coeffects
let fetchAndParse (url: NativeStr) : Data -[IO.Network, Async]-> Data = ...
```

### 5.2 Coeffect Inference

When not annotated, coeffects are inferred:

```fsharp
// Inferred: -[Pure]->
let double x = x * 2

// Inferred: -[IO.Console]->
let greet name = Console.WriteLine $"Hello, {name}"
```

### 5.3 Coeffect Compatibility

Callers must be compatible with callee coeffects:

```fsharp
let pureFunction () : int -[Pure]-> int =
    readFile "data.txt"  // ERROR: Pure cannot call IO.File
```

---

## Part 6: Platform Bindings

### 6.1 Binding Convention

Platform bindings use module convention (no DllImport):

```fsharp
module Platform.Bindings =
    let writeBytes (fd: int) (buf: nativeptr<byte>) (count: int) : int =
        Unchecked.defaultof<int>  // Placeholder - Alex provides implementation
```

### 6.2 Binding Resolution

Firefly's Alex layer provides platform-specific implementations:

| Binding | Linux x86_64 | macOS arm64 | Windows x86_64 |
|---------|--------------|-------------|----------------|
| `writeBytes` | syscall 1 (write) | syscall 0x2000004 | WriteFile |
| `readBytes` | syscall 0 (read) | syscall 0x2000003 | ReadFile |

### 6.3 Binding Safety

Platform bindings are unsafe by default:

```fsharp
// Caller must be in unsafe context or explicitly allow
let writeData (data: NativeArray<byte>) : unit -[IO, Unsafe]-> unit =
    Platform.Bindings.writeBytes 1 data.Ptr data.Length |> ignore
```

---

## Part 7: Compatibility

### 7.1 Syntax Compatibility

F# Native accepts all valid F# syntax. These are identical:

```fsharp
// Valid in both F# and F# Native
let rec factorial n =
    if n <= 1 then 1
    else n * factorial (n - 1)

type Person = { Name: string; Age: int }

let people =
    [ { Name = "Alice"; Age = 30 }
      { Name = "Bob"; Age = 25 } ]
    |> List.filter (fun p -> p.Age > 20)
```

### 7.2 Semantic Differences

The same syntax has different type semantics:

```fsharp
let s = "hello"
// F#: s : System.String
// F# Native: s : NativeStr

let opt = Some 42
// F#: opt : int option (reference type, None = null)
// F# Native: opt : int voption (value type, None = ValueNone)
```

### 7.3 Migration Path

Code targeting F# Native should:
1. Avoid BCL type annotations (`System.String`, etc.)
2. Use Alloy library instead of FSharp.Core for collections
3. Handle option as value type
4. Be prepared for non-null semantics

---

## Part 8: Diagnostics

### 8.1 Native-Specific Errors

FNCS produces native-specific error messages:

```
FS0001: This expression was expected to have type 'NativeStr'
        but here has type 'System.String'.

Hint: F# Native uses NativeStr for string literals.
      If interoperating with .NET, use NativeStr.ofString.
```

```
FS0002: Cannot assign null to native type 'NativeArray<int>'.

Hint: F# Native types are non-nullable.
      Use voption<T> for optional values.
```

### 8.2 SRTP Resolution Errors

```
FS0003: No witness found for trait constraint
        'static member (+) : MyType * MyType -> MyType'.

Searched witnesses:
  - MyType (no op_Addition member)
  - BasicOps (not applicable to MyType)

Hint: Implement op_Addition on MyType or add to witness hierarchy.
```

---

## Appendix A: Type Mapping Reference

### Primitive Types

| F# Keyword | F# Native Type | Size | Alignment |
|------------|----------------|------|-----------|
| `sbyte` | `Int8` | 1 | 1 |
| `byte` | `UInt8` | 1 | 1 |
| `int16` | `Int16` | 2 | 2 |
| `uint16` | `UInt16` | 2 | 2 |
| `int` / `int32` | `Int32` | 4 | 4 |
| `uint` / `uint32` | `UInt32` | 4 | 4 |
| `int64` | `Int64` | 8 | 8 |
| `uint64` | `UInt64` | 8 | 8 |
| `nativeint` | `NativeInt` | ptr | ptr |
| `unativeint` | `NativeUInt` | ptr | ptr |
| `float32` | `Float32` | 4 | 4 |
| `float` | `Float64` | 8 | 8 |
| `bool` | `Bool` | 1 | 1 |
| `char` | `Char` | 4 | 4 |
| `unit` | `Unit` | 0 | 1 |

### Compound Types

| F# Syntax | F# Native Type | Notes |
|-----------|----------------|-------|
| `string` | `NativeStr` | Fat pointer to UTF-8 |
| `'T option` | `voption<'T>` | Value option |
| `'T[]` | `NativeArray<'T>` | Fat pointer |
| `'T list` | `NativeList<'T>` | Cons cells (arena) |
| `Map<'K,'V>` | `NativeMap<'K,'V>` | Tree (arena) |
| `Set<'T>` | `NativeSet<'T>` | Tree (arena) |

---

## Appendix B: Grammar Extensions (Future)

Reserved syntax for future F# Native features:

```
// Ownership expressions
move <expr>
&<expr>
&mut <expr>
drop <expr>
arena { <expr> }

// Coeffect annotations
<type> -[<coeffect>]-> <type>

// Lifetime annotations
'<ident>
'static
'_

// Alignment annotations
@Align1, @Align2, @Align4, @Align8, @Packed
```

These parse as standard F# (attributes, operators) but have special semantics in F# Native.

---

## Appendix C: Specification Status

| Section | Status | Implementation |
|---------|--------|----------------|
| Native Type Universe | **Specified** | Pending in FNCS |
| Null-Free Semantics | **Specified** | Pending in FNCS |
| SRTP Resolution | **Specified** | Pending in FNCS |
| Memory Semantics | Draft | Future |
| Coeffects | Draft | Future |
| Platform Bindings | **Specified** | Implemented in Firefly |

---

*This specification is maintained alongside the fsnative-spec repository and evolves with the Fidelity framework.*
