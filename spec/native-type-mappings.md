# Native Type Mappings

This chapter defines how F# types map to native representations in F# Native compilation.

## Overview

F# Native uses familiar F# syntax with native semantics. The compiler (FNCS) resolves types to native representations at compile time, not to BCL types.

**Principle**: Users write standard F# type names. FNCS provides native semantics transparently.

## The Universal Base Type `obj` Is Not Available

In managed F#, all types inherit from `System.Object` (aliased as `obj`). This enables:
- Boxing value types to heap-allocated objects
- Runtime type information and reflection
- Heterogeneous collections (`obj list`)
- Generic `%A` formatting via runtime inspection

**F# Native eliminates `obj` entirely.** There is no universal base type. The compiler SHALL reject any code that references `obj` or `System.Object`.

### Rationale

| Managed F# Capability | Why It Requires `obj` | F# Native Alternative |
|-----------------------|----------------------|----------------------|
| Boxing (`box x`) | Wraps value in heap object | Not needed; value types stay value types |
| Unboxing (`unbox x`) | Extracts value from object | Not available; no boxed values exist |
| `%A` / `%O` formatting | Runtime type inspection | SRTP-based formatting with compile-time dispatch |
| `obj list` | Heterogeneous collection | Discriminated union with explicit cases |
| Downcasting (`:?>`) | Runtime type check | Pattern matching on discriminated unions |
| `typeof<'T>` | Runtime type token | Not available; types are compile-time only |

### Why `obj` Cannot Exist in Native Compilation

1. **No runtime type information**: Native binaries do not carry type metadata. There is no mechanism to inspect a value's type at runtime.

2. **No garbage collector**: The `obj` type implies heap allocation with GC-managed lifetime. F# Native uses deterministic, scope-based memory management.

3. **Full static resolution**: All types are resolved at compile time. Generic type parameters are monomorphized (specialized at each call site). Type erasure to `obj` is unnecessary and would lose type safety.

4. **SRTP replaces runtime dispatch**: Where managed F# uses `obj` and runtime dispatch (like `printf "%A"`), F# Native uses statically resolved type parameters with compile-time method resolution.

### Migrating Code That Uses `obj`

Code using `obj` must be refactored to use type-safe alternatives:

**Heterogeneous collections:**
```fsharp
// DOES NOT COMPILE in F# Native
let values : obj list = [box 1; box "hello"; box 3.14]

// Use discriminated union instead
type Value = 
    | Int of int 
    | Str of string 
    | Float of float
let values : Value list = [Int 1; Str "hello"; Float 3.14]
```

**Polymorphic formatting:**
```fsharp
// DOES NOT COMPILE in F# Native  
let show (x: obj) = sprintf "%A" x

// Use SRTP with operator overloading
type Showable = Showable
    with static member inline ($) (Showable, x: int) = intToString x
         static member inline ($) (Showable, x: string) = x
         // ... additional overloads

let inline show x = Showable $ x
```

**Type-based dispatch:**
```fsharp
// DOES NOT COMPILE in F# Native
let process (x: obj) =
    match x with
    | :? int as i -> handleInt i
    | :? string as s -> handleString s
    | _ -> handleOther ()

// Use discriminated union with exhaustive matching
type Input = IntInput of int | StringInput of string
let process (x: Input) =
    match x with
    | IntInput i -> handleInt i
    | StringInput s -> handleString s
```

## Compile-Time Metaprogramming

The absence of `obj` and `System.Reflection` does not leave F# Native without metaprogramming capabilities. Three F# features provide **typed, compile-time metaprogramming** that surpasses what reflection-based approaches can offer:

| Feature | Role | Reflection Equivalent |
|---------|------|----------------------|
| **Quotations** (`Expr<'T>`) | Encode program fragments as inspectable data | `MethodInfo`, `Expression<T>` |
| **Active Patterns** | Compositional structural recognition | `GetType()`, type discrimination |
| **Computation Expressions** | Continuation capture as notation | Callback-based async, monadic patterns |

### Why This Matters

Other native-compiled ML-family languages lack typed metaprogramming:

| Capability | OCaml | Rust | F# Native |
|------------|-------|------|-----------|
| Typed quotations | No | No | Yes |
| Pattern-based recognition | Match only | Match only | Active patterns |
| Continuation notation | No | No | Computation expressions |
| Metaprogramming | PPX (string-based) | proc_macro (token-based) | Quotations (typed) |

F# quotations carry full type information through transformations. OCaml's PPX system and Rust's procedural macros operate on strings or token streams - they lack the type safety that quotations provide.

### Quotations as Semantic Carriers

Quotations encode constraints and metadata as compile-time data that the compiler can inspect:

```fsharp
// Peripheral descriptor carried as typed quotation
let gpioDescriptor: Expr<PeripheralDescriptor> = <@
    { Name = "GPIO"
      BaseAddress = 0x48000000un
      MemoryRegion = Peripheral }
@>
```

The compiler extracts semantic information from quotations during PSG construction. No runtime reflection is needed - the information is available at compile time and can guide code generation (e.g., emitting volatile loads for peripheral access).

### Active Patterns for Structural Recognition

Active patterns enable compositional matching without type discrimination hierarchies:

```fsharp
// Recognize SRTP dispatch in PSG nodes
let (|SRTPDispatch|_|) (node: PSGNode) =
    match node.TypeCorrelation with
    | Some { SRTPResolution = Some srtp } -> Some srtp
    | _ -> None

// Composable usage
match currentNode with
| SRTPDispatch srtp -> emitResolvedCall srtp
| PeripheralAccess info -> emitVolatileAccess info
| _ -> emitDefault node
```

Active patterns compose with `&` and `|`, can be tested in isolation, and encapsulate recognition logic - capabilities that runtime type inspection cannot match.

### Computation Expressions as Continuation Capture

Every `let!` in a computation expression captures a continuation:

```fsharp
maybe {
    let! x = someOption    // Bind(someOption, fun x -> ...)
    let! y = otherOption   // Bind(otherOption, fun y -> ...)
    return x + y
}
```

This desugaring to nested lambdas provides continuation semantics as notation. The compilation strategy depends on the computation pattern:

| Pattern | Compilation Strategy |
|---------|---------------------|
| Sequential effects (async, state) | Preserve continuations (DCont dialect) |
| Parallel pure (validated, reader) | Compile to data flow (Inet dialect) |

### Normative Requirements

NORMATIVE: `System.Reflection` and all reflection-based APIs SHALL NOT be available in F# Native. The compiler SHALL reject any code that references reflection types or methods.

NORMATIVE: Quotations, active patterns, and computation expressions SHALL be fully supported. These features operate at compile time and impose no runtime overhead.

NORMATIVE: Quotation-based metaprogramming SHALL NOT require runtime evaluation. All quotation inspection and transformation occurs during compilation.

## Primitive Types

### Numeric Types

| F# Syntax | Native Representation | Size | Notes |
|-----------|----------------------|------|-------|
| `unit` | Zero-sized type | 0 | No runtime representation |
| `bool` | `i8` | 1 byte | 0 = false, non-zero = true |
| `int` | `isize` | Platform word | 4 bytes (32-bit), 8 bytes (64-bit) |
| `uint` | `usize` | Platform word | Unsigned platform word |
| `int8` / `sbyte` | `i8` | 1 byte | Signed 8-bit |
| `uint8` / `byte` | `u8` | 1 byte | Unsigned 8-bit |
| `int16` | `i16` | 2 bytes | Signed 16-bit |
| `uint16` | `u16` | 2 bytes | Unsigned 16-bit |
| `int32` | `i32` | 4 bytes | Signed 32-bit |
| `uint32` | `u32` | 4 bytes | Unsigned 32-bit |
| `int64` | `i64` | 8 bytes | Signed 64-bit |
| `uint64` | `u64` | 8 bytes | Unsigned 64-bit |
| `nativeint` | `isize` | Platform word | Signed pointer-sized |
| `unativeint` | `usize` | Platform word | Unsigned pointer-sized |

### Floating Point Types

| F# Syntax | Native Representation | Size | Notes |
|-----------|----------------------|------|-------|
| `float` / `double` | `f64` | 8 bytes | IEEE 754 double precision |
| `float32` / `single` | `f32` | 4 bytes | IEEE 754 single precision |

### Character and String Types

| F# Syntax | Native Representation | Size | Notes |
|-----------|----------------------|------|-------|
| `char` | `i32` | 4 bytes | UTF-32 codepoint (Unicode scalar value) |
| `string` | `{ptr: *u8, len: usize}` | 16 bytes | UTF-8 fat pointer |

## Composite Types

### Tuples

Tuples are laid out as contiguous structs with natural alignment:

```fsharp
let pair : int * float = (42, 3.14)
```

**Layout**:
```
┌─────────┬─────────┬─────────┐
│ int (8) │ pad (0) │ float(8)│
└─────────┴─────────┴─────────┘
Total: 16 bytes
```

### Records

Records are named product types with field-order layout:

```fsharp
type Point = { X: float; Y: float }
```

**Layout**: Same as tuple of fields in declaration order.

### Discriminated Unions

Discriminated unions use tagged representation:

```fsharp
type Option<'T> = None | Some of 'T
```

**Layout**:
```
┌──────────┬────────────────────────┐
│ Tag (i8) │ Payload (size of 'T)   │
└──────────┴────────────────────────┘
```

| Property | Value |
|----------|-------|
| Tag size | `i8` for ≤256 variants |
| Tag values | 0, 1, 2... in declaration order |
| Payload | Size of largest variant |

### Single-Case Unions (Newtypes)

Single-case unions have no tag overhead:

```fsharp
type UserId = UserId of int
```

**Layout**: Same as wrapped type (`int`).

## Reference Types

### Arrays

Arrays use fat pointer representation:

```fsharp
let numbers : array<int> = [| 1; 2; 3 |]
```

**Layout**:
```
Header (16 bytes):
┌─────────────────┬─────────────────┐
│ ptr: *T         │ len: usize      │
└─────────────────┴─────────────────┘

Elements (contiguous):
┌─────┬─────┬─────┐
│ [0] │ [1] │ [2] │
└─────┴─────┴─────┘
```

### Strings

Strings use UTF-8 fat pointer representation:

**Layout**:
```
┌─────────────────┬─────────────────┐
│ ptr: *u8        │ len: usize      │
└─────────────────┴─────────────────┘
16 bytes (64-bit platform)
```

| Property | Value |
|----------|-------|
| Encoding | UTF-8 |
| Length | Byte count (not character count) |
| Empty string | `{ptr: valid, len: 0}` |
| Null | Not representable |

## Parameterized Types

### Option

Option types use `voption` (value option) semantics:

```fsharp
let maybe : int option = Some 42
```

**Layout**: Stack-allocated tagged union (see Discriminated Unions).

| Property | Value |
|----------|-------|
| `None` tag | 0 |
| `Some` tag | 1 |
| Heap allocation | Never |
| Null | Not representable |

### Result

Result types are stack-allocated tagged unions:

```fsharp
let result : Result<int, string> = Ok 42
```

**Layout**: Tag + max(sizeof Ok payload, sizeof Error payload).

### List

Lists use cons cell representation:

```fsharp
let numbers : int list = [1; 2; 3]
```

**Layout** (per cons cell):
```
┌─────────────────┬─────────────────────┐
│ head: 'T        │ tail: ptr<list<'T>> │
└─────────────────┴─────────────────────┘
```

## Function Types

### Direct Functions

Known call sites compile to direct calls:

```fsharp
let add x y = x + y
add 1 2  // Direct call, no closure
```

### Closures

Functions capturing environment use closure representation:

```fsharp
let makeAdder n = fun x -> x + n
```

**Layout**:
```
┌─────────────────────┬─────────────────────┐
│ fn_ptr: ptr<fn>     │ env: captured values│
└─────────────────────┴─────────────────────┘
```

## MLIR Type Mappings

| F# Type | MLIR Type |
|---------|-----------|
| `unit` | (none - ZST) |
| `bool` | `i8` |
| `int` | `index` |
| `int32` | `i32` |
| `int64` | `i64` |
| `float` | `f64` |
| `float32` | `f32` |
| `char` | `i32` |
| `string` | `!fidelity.str` |
| `option<'T>` | `!fidelity.option<T>` |
| Tuple | `tuple<...>` |
| Record | `!fidelity.record<...>` |
| DU | `!fidelity.union<...>` |
| Function | `!fidelity.fn<A, B>` |

## See Also

- [Types and Type Constraints](types-and-type-constraints.md) - Type system overview
- [The Native Library Alloy](the-native-library-alloy.md) - Type operations
- [Memory Regions](memory-regions.md) - Pointer types
