---
title: "Native Type Mappings"
weight: 130
category: Language
status: normative
---

This chapter defines how F# types map to native representations in Clef compilation.

## Overview

Clef uses familiar F# syntax with native semantics. The compiler (CCS) resolves types to native representations at compile time, not to BCL types.

**Principle**: Users write standard F# type names. CCS provides native semantics transparently.

## The Universal Base Type `obj` Is Not Available

In managed F#, all types inherit from `System.Object` (aliased as `obj`). This enables:
- Boxing value types to heap-allocated objects
- Runtime type information and reflection
- Heterogeneous collections (`obj list`)
- Generic `%A` formatting via runtime inspection

**Clef eliminates `obj` entirely.** There is no universal base type. The compiler SHALL reject any code that references `obj` or `System.Object`.

The foreign pair of [JavaScript Boundary Semantics](javascript-boundary.md) (`JsValue`, `JsRef<'T>`) does not reintroduce a universal type, and this section's prohibition stands unchanged wherever that chapter is in force. The universal type's hazard is subsumption, the unmarked conversion that lets any value become the universal type silently; the pair participates in no subtyping relationship, its injection is explicit at declared boundary functions, and its only elimination is narrowing.

### Rationale

| Managed F# Capability | Why It Requires `obj` | Clef Alternative |
|-----------------------|----------------------|----------------------|
| Boxing (`box x`) | Wraps value in heap object | Not needed; value types stay value types |
| Unboxing (`unbox x`) | Extracts value from object | Not available; no boxed values exist |
| `%A` / `%O` formatting | Runtime type inspection | SRTP-based formatting with compile-time dispatch |
| `obj list` | Heterogeneous collection | Discriminated union with explicit cases |
| Downcasting (`:?>`) | Runtime type check | Pattern matching on discriminated unions |
| `typeof<'T>` | Runtime type token | Not available; types are compile-time only |

### Why `obj` Cannot Exist in Native Compilation

1. **No runtime type information**: Native binaries do not carry type metadata. There is no mechanism to inspect a value's type at runtime.

2. **No garbage collector**: The `obj` type implies heap allocation with GC-managed lifetime. Clef uses deterministic, scope-based memory management.

3. **Full static resolution**: All types are resolved at compile time. Generic type parameters are monomorphized (specialized at each call site). Type erasure to `obj` is unnecessary and would lose type safety.

4. **SRTP replaces runtime dispatch**: Where managed F# uses `obj` and runtime dispatch (like `printf "%A"`), Clef uses statically resolved type parameters with compile-time method resolution.

### Migrating Code That Uses `obj`

Code using `obj` must be refactored to use type-safe alternatives:

**Heterogeneous collections:**
```fsharp
// DOES NOT COMPILE in Clef
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
// DOES NOT COMPILE in Clef  
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
// DOES NOT COMPILE in Clef
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

The absence of `obj` and `System.Reflection` does not leave Clef without metaprogramming capabilities. Three F# features provide **typed, compile-time metaprogramming** that surpasses what reflection-based approaches can offer:

| Feature | Role | Reflection Equivalent |
|---------|------|----------------------|
| **Quotations** (`Expr<'T>`) | Encode program fragments as inspectable data | `MethodInfo`, `Expression<T>` |
| **Active Patterns** | Compositional structural recognition | `GetType()`, type discrimination |
| **Computation Expressions** | Continuation capture as notation | Callback-based async, monadic patterns |

### Why This Matters

Other native-compiled ML-family languages lack typed metaprogramming:

| Capability | OCaml | Rust | Clef |
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

The compiler extracts semantic information from quotations during [PSG construction](program-semantic-graph.md). No runtime reflection is needed - the information is available at compile time and can guide code generation (e.g., emitting volatile loads for peripheral access).

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
| Sequential effects (async, state) | Saturate through the suspension recipe — segments at cuts, a frame, a delimiter edge — and witness as `scf.index_switch` over a discriminant ([DCont Representation](dcont-representation.md) §2, §5, §6) |
| Parallel pure (validated, reader) | Compile to data flow (Inet regime) |

### Normative Requirements

NORMATIVE: `System.Reflection` and all reflection-based APIs SHALL NOT be available in Clef. The compiler SHALL reject any code that references reflection types or methods.

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

[Discriminated unions use tagged representation](discriminated-union-representation.md):

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

## Struct Alignment

### Default Alignment

Structs use natural alignment based on their largest field:

| Largest Field | Default Alignment |
|---------------|-------------------|
| `i8`, `u8` | 1 byte |
| `i16`, `u16` | 2 bytes |
| `i32`, `u32`, `f32` | 4 bytes |
| `i64`, `u64`, `f64` | 8 bytes |
| pointer | Platform word alignment: 8 bytes on 64-bit, 4 bytes on thumbv8m/32-bit |

Pointer alignment follows the [Pointer width dimension](ntu-types.md); it is the platform word, not a fixed 8 bytes.

### Explicit Alignment

The `[<Align(n)>]` attribute requests specific alignment:

```fsharp
[<Align(64)>]
[<Struct>]
type CacheAligned = { Value: int64 }
```

NORMATIVE: The compiler SHALL respect alignment requests that are:
- Powers of two
- Greater than or equal to natural alignment
- Less than or equal to platform page size (typically 4096)

NORMATIVE: Alignment requests that cannot be satisfied SHALL produce a compile-time error.

### Alignment and SIMD

For SIMD operations, alignment affects performance significantly:

| Vector Width | Recommended Alignment |
|--------------|----------------------|
| 128-bit (SSE, NEON) | 16 bytes |
| 256-bit (AVX2) | 32 bytes |
| 512-bit (AVX-512) | 64 bytes |

Misaligned vector loads may incur penalties or faults depending on the instruction.

### Stack and Arena Allocation

NORMATIVE: Stack-allocated aligned types SHALL be placed at appropriately aligned addresses.

NORMATIVE: Arena allocators SHALL provide an alignment-aware allocation function:
```fsharp
Arena.allocAligned<'T> : Arena -> alignment:int -> count:int -> Ptr<'T, Arena, ReadWrite>
```

## Intrinsic Operations

Certain operations have direct hardware support that F# loops cannot match. CCS intrinsics provide guaranteed-efficient implementations.

### Bit Manipulation Intrinsics

The middle end emits each intrinsic as a target-agnostic operation. The LLVM-pathway realization is the LLVM intrinsic shown below; other target pathways (CIRCT for FPGA, JSIR for JS) realize the same operation with their own target primitives.

| Function | LLVM-pathway realization | Description |
|----------|---------------------|-------------|
| `clz : uint32 -> int` | `llvm.ctlz.i32` | Count leading zeros |
| `clz64 : uint64 -> int` | `llvm.ctlz.i64` | Count leading zeros (64-bit) |
| `ctz : uint32 -> int` | `llvm.cttz.i32` | Count trailing zeros |
| `ctz64 : uint64 -> int` | `llvm.cttz.i64` | Count trailing zeros (64-bit) |
| `popcount : uint32 -> int` | `llvm.ctpop.i32` | Population count |
| `popcount64 : uint64 -> int` | `llvm.ctpop.i64` | Population count (64-bit) |
| `bswap : uint32 -> uint32` | `llvm.bswap.i32` | Byte swap |
| `bswap64 : uint64 -> uint64` | `llvm.bswap.i64` | Byte swap (64-bit) |

NORMATIVE: These functions SHALL lower to the target's native bit-manipulation primitive, not loop-based implementations. On the LLVM pathway that primitive is the corresponding LLVM intrinsic shown above.

### Arithmetic Intrinsics

| Function | LLVM-pathway realization | Description |
|----------|---------------------|-------------|
| `mulhi : uint64 -> uint64 -> uint64` | (platform-specific) | High 64 bits of 128-bit product |
| `addCarry : uint64 -> uint64 -> uint64 -> struct(uint64 * uint64)` | `llvm.uadd.with.overflow` | Add with carry in/out |

NORMATIVE: Multi-word arithmetic operations SHALL use carry-propagating instructions where available.

### Usage

```fsharp
let extractRegime (bits: uint32) =
    let shifted = bits <<< 1
    let leadingZeros = clz shifted  // Guaranteed 1-2 cycles, not a loop
    // ... regime extraction logic
 
```

### Fallback Behavior

On targets without hardware support for specific intrinsics:

NORMATIVE: The compiler SHALL emit efficient software fallbacks that match the semantic behavior.

NORMATIVE: The compiler MAY emit warnings when intrinsics fall back to software implementation on performance-critical targets.

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

[Option types](option-operations-representation.md) use `voption` (value option) semantics:

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

Functions capturing environment use [closure representation](closure-representation.md):

```fsharp
let makeAdder n = fun x -> x + n
```

**Form** — two SSA values, never packed ([Closure Representation §6.3](closure-representation.md)):
```
fn:  func.constant @makeAdder_lambda : (memref<Exi8>, int) -> int
env: ┌─────────────────────┐
     │ n: captured value   │   memref<Exi8>, E literal at saturation
     └─────────────────────┘
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
| `string` | `memref<?xi8>` |
| `option<'T>` | `memref<Exi8>` — `{tag, payload}`, stack-placed ([Option Operations](option-operations-representation.md)) |
| Tuple | `tuple<...>` |
| Record | `memref<Exi8>` (settled layout) |
| DU | `memref<Exi8>` (`{tag, payload}`) |
| Function | `(A) -> B` — a `func` value |

## Why IL Infrastructure Is Removed from CCS

Clef Compiler Service (CCS) targets [native compilation via MLIR](backend-lowering-architecture.md) (Multi-Level Intermediate Representation), not CLR bytecode. While CCS originated from the F# Compiler Services (FCS) codebase, its type universe, compilation passes, and concurrency primitives are independently defined. Consequently, all IL-based infrastructure has been removed from the typed tree operations.

### The Architecture Boundary

```
┌─────────────────────────────────────────────────┐
│  CCS (Clef Compiler Services)             │
│  - Type checking, resolution, inference         │
│  - Produces typed tree with native types        │
│  - NO code generation, NO IL                    │
└─────────────────────────────────────────────────┘
                      │
                      ▼ Typed Tree (native types)
┌─────────────────────────────────────────────────┐
│  Alex (Code Generation)                         │
│  - PSG traversal via Zipper                     │
│  - Platform bindings for syscalls               │
│  - MLIR emission                                │
└─────────────────────────────────────────────────┘
                      │
                      ▼ MLIR
┌─────────────────────────────────────────────────┐
│  MLIR Optimization Passes                       │
│  - Loop optimization (SCF dialect)              │
│  - Arithmetic optimization (arith dialect)      │
│  - Memory optimization                          │
└─────────────────────────────────────────────────┘
                      │
                      ▼ Portable dialects
┌─────────────────────────────────────────────────┐
│  Target Pathway (target-committing)             │
│  - LLVM pathway: LLVM IR → CPU/MCU binary       │
│  - CIRCT pathway: HW dialects → bitstream       │
│  - JSIR pathway: → JavaScript module            │
└─────────────────────────────────────────────────┘
```

The MLIR optimization passes and everything above them stay portable; a target is committed only at the target pathway. LLVM is one pathway among several.

### Why IL Operations Are Not Stubbed

The original FCS contains IL-based operations for loop optimization, null handling, and arithmetic. These were initially stubbed during the CCS fork, but **stubs produce semantically wrong results**:

| Stubbed Function | Wrong Behavior | Why It's Wrong |
|------------------|----------------|----------------|
| `mkAsmExpr` | Returns `Coerce`/identity | Should compute arithmetic |
| `mkILAsmCeq`, `mkILAsmClt` | Returns constant `false` | Should compare values |
| `mkGetStringLength` | Returns constant `0` | Should return actual length |
| `mkDecr` | Returns expression unchanged | Should decrement value |

**Principle**: "Delete, don't stub" - Broken stubs hide defects and produce silent wrong behavior. Complete removal makes missing functionality explicit.

### What Functionality Moves Downstream

| IL Infrastructure | Native Equivalent | Location |
|-------------------|-------------------|----------|
| `TOp.ILAsm` (arithmetic) | MLIR arith dialect ops | Alex code generation |
| `TOp.ILCall` (method calls) | MLIR func.call / platform bindings | Alex code generation |
| Loop optimization | MLIR SCF dialect transforms | MLIR optimization passes |
| String length/concat | Native string fat pointer ops | Alex code generation |
| Integer conversions | MLIR arith.extsi/extui/trunci | Alex type lowering |
| Null handling | Not needed - Clef has no null | See below |

### Null Is Not Representable

NORMATIVE: Clef has **no null values**. The `null` keyword and null checking operations are not available.

- `mkNull`, `mkNullTest`, `mkNonNullTest`, `mkNonNullCond` - all removed
- Option types (`voption`) replace nullable references
- Pattern matching replaces null checks

This is consistent with Clef's safety guarantees: no null pointer dereferences are possible because null cannot be expressed.

On the JSIR pathway, `null` and `undefined` appear in the emitted JavaScript artifact only as boundary representations selected by generated code and as the proven `Option` erasure of [Option Operations Representation §2.1](option-operations-representation.md), per the confinement rule of [JavaScript Boundary Semantics §8](javascript-boundary.md). No Clef-typed value is `null` or `undefined` on any pathway.

### Removed IL Infrastructure

The following were removed from `TypedTreeOps.fs`:

**IL Instruction Stubs**:
- `ILDataType` type
- `AI_ldnull`, `AI_cgt_un`, `AI_clt_un`, `AI_add`, `AI_sub`, `AI_div_un`, etc.
- `ILInstr` module
- `mkAsmExpr` function

**Loop Optimization (vestigial - no callers)**:
- `DetectAndOptimizeForEachExpression`
- `mkOptimizedRangeLoop`, `mkRangeCount`
- `mkFastForLoop`
- Pattern matchers: `Int32Expr`, `RangeInt32Step`, `CompiledForEachExpr`, etc.
- `IntegralConst` module, `IntegralRange`, `EmptyRange`, `ConstCount` patterns

**Null Operations**:
- `mkNull`, `mkNullTest`, `mkNonNullTest`, `mkNonNullCond`

**Broken Comparison Stubs**:
- `mkILAsmCeq`, `mkILAsmClt`, `mkDecr`, `mkGetStringLength`

### The Key Insight

IL-based loop optimization at the typed tree level was **premature optimization at the wrong layer**. Native loop optimization belongs in MLIR passes where the target architecture is known and appropriate loop transformations (vectorization, unrolling, tiling) can be applied.
