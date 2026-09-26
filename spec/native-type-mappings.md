---
title: "Native Type Mappings"
weight: 130
category: Language
status: normative
---

This chapter defines how Clef source types map to native representations.

## Overview

Clef uses familiar F# syntax with native semantics. The compiler (CCS) resolves types to native representations at compile time, not to BCL types.

**Principle**: Source types follow the [NTU kind and dimension model](ntu-types.md).
Familiar syntax does not import managed types or width-named numeric types.

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
| Sequential effects (async, state) | Saturate through the suspension recipe (segments at cuts, a frame, a delimiter edge) and witness as `scf.index_switch` over a discriminant ([DCont Representation](dcont-representation.md) §2, §5, §6) |
| Parallel pure (validated, reader) | Compile to data flow (Inet regime) |

### Normative Requirements

NORMATIVE: `System.Reflection` and all reflection-based APIs SHALL NOT be available in Clef. The compiler SHALL reject any code that references reflection types or methods.

NORMATIVE: Quotations, active patterns, and computation expressions SHALL be fully supported. These features operate at compile time and impose no runtime overhead.

NORMATIVE: Quotation-based metaprogramming SHALL NOT require runtime evaluation. All quotation inspection and transformation occurs during compilation.

## Primitive Types

### Numeric Types

| Clef source | Native representation | Selection |
|-------------|-----------------------|-----------|
| `unit` | Zero-sized value | No payload storage |
| `bool` | `i8` on a CPU | Boolean storage; not an integer source type |
| `int`, `int<dim>` | `i<w>` | Analysed range and the selected platform's declared representations |

Signedness and width are storage facts. `byte`, `uint8`, `uint`, `int64`,
`nativeint` and the other width-named spellings are not Clef source types.
A byte-unit buffer is `array<int>`; its encoding or boundary contract supplies
the cell range and representation evidence. An ordinary integer is not an
address; addresses use the semantic types in [NTU Types §2.3](ntu-types.md#23-pointer-and-semantic-types-implicit-pointer-width).

### Floating Point Types

| Clef source | Native representation | Selection |
|-------------|-----------------------|-----------|
| `float`, `float<dim>` | The settled real representation | [Numeric Selection](numeric-selection.md) over the declared offerings and analysed range |

`float32`, `single`, `double` and other representation spellings do not select
source types. The numeric selection rules govern any unobservable-range case.

### Character and String Types

| F# Syntax | Native Representation | Size | Notes |
|-----------|----------------------|------|-------|
| `char` | `i32` | 4 bytes | UTF-32 codepoint (Unicode scalar value) |
| `string` | `memref<?xi8>` | Byte length is the memref's dimension | UTF-8 buffer; the buffer is the value (see [Strings](#strings)) |

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

A discriminated-union value is a `memref<Exi8>` view of its `{tag, payload}` storage block, with E settled at saturation ([Discriminated Union Representation §3](discriminated-union-representation.md)). A payload that is itself arena-resident (a recursive case, a collection) is linked by a bounded `index`, never by an address:

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
| Payload | Size of largest variant; an arena-resident payload is an `index` link carrying VC-LINK (`0 <= i < extent(arena)`), never an address |

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
| `index` (arena link), `memref` descriptor word | Platform word alignment: 8 bytes on 64-bit, 4 bytes on thumbv8m/32-bit |

Word alignment follows the `Pointer` width dimension of [NTU Types §2.3](ntu-types.md); it is the platform word, not a fixed 8 bytes.

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

An array is a `memref<?xT>` view: the element buffer is the value, and its length is the memref's dimension. There is no header struct and no separate length word.

```fsharp
let numbers : array<int> = [| 1; 2; 3 |]
```

**Layout** (the buffer; `memref<?xT>` at saturation):
```
array<'T>   memref<?xT>
┌─────┬─────┬─────┐
│ [0] │ [1] │ [2] │   contiguous, naturally aligned; extent = length
└─────┴─────┴─────┘
```

| Property | Value |
|----------|-------|
| Value | The `memref<?xT>` view; `Array.length` is `memref.dim` |
| Element layout | Contiguous, naturally aligned; monomorphized, elements are never boxed |
| Placement | Stack, arena, or the platform's declared program-lifetime space, selected by the lifetime lattice of [Closure Representation §3.3](closure-representation.md) |
| Bounds checking | Every access is guarded by `0 <= i < memref.dim`; there is no source-level exemption ([Native Type Universe §4.2](native-type-universe.md)) |
| Empty array | A view of extent 0; never a null |
| Null | Not representable |

The view's descriptor (base, offset, extent, stride) is a lowering artifact of the target pathway, not a Clef value; no Clef operation observes it.

### Strings

A string is a `memref<?xi8>` view of its UTF-8 byte buffer: the buffer is the value, and its byte length is the memref's dimension. There is no separate length header.

**Layout** (the buffer; `memref<?xi8>` at saturation):
```
string   memref<?xi8>
┌────┬────┬────┬─────┐
│ b0 │ b1 │ b2 │ ... │   UTF-8 bytes; extent = byte length
└────┴────┴────┴─────┘
```

| Property | Value |
|----------|-------|
| Encoding | UTF-8 |
| Length | Byte count (not character count); `String.byteLength` is `memref.dim` |
| Slicing | A `memref.subview` of the same buffer; zero-copy |
| Placement | A literal is program-lifetime and immutable: it resides in the platform's declared immutable program-lifetime space, cited by name from the platform description (rodata on an ELF target, flash on an MCU, constant memory on a GPU, initialised BRAM on an FPGA), emitted as a `memref.global`. A constructed string is placed by the lifetime lattice of [Closure Representation §3.3](closure-representation.md) |
| Empty string | A view of extent 0; never a null |
| Null | Not representable |

The view's descriptor is a lowering artifact of the target pathway, not a Clef value; no Clef operation observes it. On the JSIR pathway `string` is a host string and this layout does not bind ([Native Type Universe §4.1](native-type-universe.md)).

#### Integer byte-unit conversions

The source signatures use dimensionless integers:

```fsharp
String.fromBytes : array<int> -> string
String.toBytes : string -> array<int>
```

These array elements are encoded byte units, not Unicode scalar values. The
string laws in [Native Type Universe §4.1](native-type-universe.md#41-string)
remain authoritative: strings contain valid UTF-8, and character operations
observe codepoints.

`String.fromBytes` requires evidence that its input elements fit the encoding's
byte units and that the resulting sequence is valid UTF-8. An integer array's
source type alone establishes neither property. Baker retains the particular
allocation, aliases, slices, contributing writes and dependent reads in the PSG;
representation selection applies to those buffer occurrences, never to every
array with the same logical element type. The selected platform must offer the
required unsigned byte representation.

Both conversions establish independent snapshots. Later writes through the
input of `fromBytes` SHALL NOT change the constructed string. Mutating the array
returned by `toBytes` SHALL NOT change its source string. `toBytes` preserves the
UTF-8 byte sequence, order and length. Eliminating a copy requires ownership and
preservation evidence that establishes these observable laws; a matching
physical layout alone is insufficient.

Baker expresses the required storage, copy and final view relationships in the
graph before Alex witnesses them. Alex consumes the settled storage and width
adaptations. Its internal byte/string view requires matching physical carriers
and cannot repair a wider integer buffer by relabeling its return type.

Byte-range evidence alone does not prove UTF-8 validity: a lone `255` or
continuation unit `169` is not admitted as text.

The internal byte view carries the encoding's eight-bit storage evidence. The public
array's representation follows its complete write range, including subsequent
ordinary integer writes above 255; it is not restricted to the view's carrier.
The graph retains the selected representation declaration and copy dependencies,
and normal range analysis settles the resulting reads and writes.

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

A list value is the `index` of its first node in the arena that holds the list. A node is a flat aggregate with a settled layout, a `memref<Exi8>` view at saturation ([List Operations Representation §5](list-operations-representation.md)):

```fsharp
let numbers : int list = [1; 2; 3]
```

**Layout** (per node):
```
list<'T> node   memref<Exi8>
┌──────────────────────┬─────────────────┬──────────────────────────┐
│ tag: i8 (Empty|Cons) │ head: 'T        │ tail: index (arena link) │
└──────────────────────┴─────────────────┴──────────────────────────┘
```

| Property | Value |
|----------|-------|
| Value | `index`: the arena-relative offset of the first node |
| Placement | Arena selected by the lifetime lattice of [Closure Representation §3.3](closure-representation.md) |
| `tail` | `index` into the arena buffer the list lives in (an arena-relative offset), never an address; a link load is one `memref.load` of an `index` |
| Link obligation | VC-LINK: `0 <= tail < extent(arena)`, quantifier-free over graph literals, discharged at saturation before witnessing; `0` is the sentinel |
| Empty list | The sentinel node at offset 0 of the arena: `tag = Empty`, `tail = 0` (its own index), `head` zero-initialised and never read. `List.empty` is the index literal `0` and allocates nothing |
| Sentinel image | Exactly one program-lifetime, immutable image per element type, copied to offset 0 of every arena that hosts the type; it resides in the platform's declared immutable program-lifetime space, cited by name from the platform description through a `Resides` edge ([Program Hypergraph §6](program-hypergraph.md)): rodata on an ELF target, flash on an MCU, constant memory on a GPU, initialised BRAM on an FPGA. Every arena that hosts `list<'T>` nodes carries a copy at offset 0, initialised when the arena is created |
| Sentinel immutability | The slot `[0, sizeof(node))` is `ReadOnly` ([Access Kinds](access-kinds.md)); a store through it is the compile-time diagnostic CCS8020. `cons` onto the sentinel places a fresh arena node whose tail is 0; the sentinel is never mutated in place |
| `isEmpty` | The literal comparison `tag = Empty` (one load, one compare), equivalently `index = 0`; never a null check |
| Guard obligation | VC-GUARD: the `tag = Cons` test dominates every read of `head` on the saturated graph; a dominance check, no quantifiers |
| Structural sharing | Index aliasing within one arena |
| Null | Not representable |

Every `tail` is a valid node: operations read `tail` unconditionally and recursion terminates at the sentinel. Layout obligations (VC-LINK, VC-GUARD, sentinel residence and immutability) are quantifier-free at saturation over the graph's literals; the list algebra is a schema lemma proven once per recipe shape, never a per-program fixpoint.

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

**Form**: two SSA values, never packed ([Closure Representation §6.3](closure-representation.md)):
```
fn:  func.constant @makeAdder_lambda : (memref<Exi8>, int) -> int
env: ┌─────────────────────┐
     │ n: captured value   │   memref<Exi8>, E literal at saturation
     └─────────────────────┘
```

## MLIR Type Mappings

The middle end emits these portable forms; a target pathway lowers them ([Backend Lowering Architecture](backend-lowering-architecture.md)). No row is a pointer: a buffer is a `memref` view, and a link between arena-resident nodes is a bounded `index` carrying VC-LINK.

| Clef type | MLIR type |
|---------|-----------|
| `unit` | (none - ZST) |
| `bool` | `i8` |
| `int`, `int<dim>` | `i<w>` from settled range and representation evidence |
| `float`, `float<dim>` | The settled real representation |
| `char` | `i32` |
| `string` | `memref<?xi8>`; byte length is the dimension |
| `array<'T>` | `memref<?xT>`; length is the dimension |
| `option<'T>` | `memref<Exi8>`: `{tag, payload}`, stack-placed ([Option Operations](option-operations-representation.md)) |
| `Result<'T, 'E>` | `memref<Exi8>`: `{tag, payload}` as a two-case union |
| `list<'T>` | `index`: arena link to the first node, itself a `memref<Exi8>` ([List](#list); [List Operations Representation §5](list-operations-representation.md)) |
| `Map<'K, 'V>` | `index`: arena link to the root node, itself a `memref<Exi8>` ([Map Representation §2](map-representation.md)) |
| `Set<'T>` | `index`: arena link to the root node, itself a `memref<Exi8>` ([Set Representation §2](set-representation.md)) |
| Tuple | `tuple<...>` |
| Record | `memref<Exi8>` (settled layout) |
| DU | `memref<Exi8>` (`{tag, payload}`); an arena-resident payload is an `index` link |
| Function | `(A) -> B`: a `func` value |
| Closure | `(fn, env)`: a `func` value `(memref<Exi8>, A) -> B` and an environment `memref<Exi8>`, never packed ([Closure Representation §6.3](closure-representation.md)) |

## Native Compilation Boundary

Clef Compiler Service (CCS) performs type checking, resolution, and inference over
native types. CCS does not emit CLR bytecode or IL operations. Alex emits portable
MLIR from the Program Semantic Graph, and target pathways realize those operations
as specified in [Backend Lowering Architecture](backend-lowering-architecture.md).

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

### Native Operation Lowering

| Operation | Native Representation | Location |
|-------------------|-------------------|----------|
| Arithmetic | MLIR arith dialect ops | Alex code generation |
| Method calls | MLIR func.call / platform bindings | Alex code generation |
| Loop optimization | MLIR SCF dialect transforms | MLIR optimization passes |
| String length/concat | `memref` view operations: `memref.dim` for length, a buffer copy for concatenation | Alex code generation |
| Integer representation adaptation | MLIR arith.extsi/extui/trunci | Alex type lowering |

Loop optimization operates on MLIR. Target-specific transformations must use the
selected target's established capabilities and preserve the program's semantics.

### Null Is Not Representable

Clef has no null values. The `null` keyword and null checking operations are not available.

- Option types (`voption`) replace nullable references
- Pattern matching replaces null checks

This is consistent with Clef's safety guarantees: no null dereference is possible because null cannot be expressed.

On the JSIR pathway, `null` and `undefined` appear in the emitted JavaScript artifact only as boundary representations selected by generated code and as the proven `Option` erasure of [Option Operations Representation §2.1](option-operations-representation.md), per the confinement rule of [JavaScript Boundary Semantics §8](javascript-boundary.md). No Clef-typed value is `null` or `undefined` on any pathway.
