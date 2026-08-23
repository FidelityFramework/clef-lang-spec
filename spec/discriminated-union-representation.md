---
title: "Discriminated Union Representation"
weight: 330
category: Representation
status: normative
---

> **Status**: Draft
> **Last Updated**: 2026-01-22

## Informative References

> **Commentary**: Discriminated Unions (DUs) are arguably F#'s most distinctive feature: the embodiment of algebraic data types that enable type-safe modeling of domain variants. Their native representation must support the full power of F# while enabling efficient compilation without a managed runtime.
>
> **Academic Foundation**: This representation draws from:
> - Pierce, B. C. (2002). *Types and Programming Languages*. MIT Press. (Sum types, case elimination)
> - Tofte, M., & Talpin, J.-P. (1997). *Region-Based Memory Management*. (Arena allocation)
> - BAREWire specification for heterogeneous serialization

---

## 1. Overview

A Discriminated Union (DU) is a sum type where each *case* represents a distinct variant with its own payload type. Unlike product types (records, tuples) where all fields are always present, a DU value contains exactly ONE case at any time.

### 1.1 Fundamental Challenge

The core challenge for native DU representation is **type heterogeneity**: different cases may have payloads of different types and sizes. Consider:

```fsharp
type Number =
    | IntVal of int      // 4-byte payload
    | FloatVal of float  // 8-byte payload

type Expr =
    | Const of int                    // primitive
    | Var of string                   // fat pointer
    | Add of Expr * Expr              // two recursive pointers (2 words)
    | Lambda of string * Expr * env   // complex nested structure
 
```

A naive "max-size union" representation loses type information:
- Extracting a `FloatVal` payload requires knowing it's a `float`, not just "8 bytes"
- Nested DUs create recursive type relationships
- Generic DU cases require monomorphization

### 1.2 Design Principles

1. **Case eliminators preserve type safety**: Each case has a typed accessor
2. **Pointer-based uniformity**: DU values are pointers to a storage block, placed by the lifetime lattice of §3.1 (stack, region, static, or heap)
3. **Heterogeneous storage**: Each case can have its own layout
4. **BAREWire compatibility**: Serialization-aware representation
5. **No runtime type discrimination**: All types resolved at compile time

---

## 2. Compile-Time Layout Determination via SRTP

### 2.1 Static Type Resolution

DUs in Clef are **eagerly typed** - all case payload types are fully resolved at compile time. Combined with SRTP (Statically Resolved Type Parameters), this guarantees deterministic layout computation:

```fsharp
type Container<'T when 'T : (static member Size : int)> =
    | Empty
    | Single of 'T
    | Pair of 'T * 'T
```

At each instantiation site, SRTP resolves `'T` to a concrete type with known `Size`. The layout is computed statically:

| Instantiation | 'T resolved to | Single payload | Pair payload |
|---------------|----------------|----------------|--------------|
| `Container<int>` | `int` (4 bytes) | 4 bytes | 8 bytes |
| `Container<Point>` | `Point` (16 bytes) | 16 bytes | 32 bytes |
| `Container<Number>` | `Number` (ptr) | 1 word | 2 words |

A pointer occupies one platform word: 8 bytes on x86-64, 4 bytes on thumbv8m/M33. The `Container<Number>` row above is the x86-64 case (1 word = 8 bytes, 2 words = 16 bytes); on thumbv8m/M33 the same instantiation is 4 bytes and 8 bytes.

### 2.2 Deterministic Layout Algorithm

For a DU type `T` with cases `C₁, ..., Cₙ`:

```
LAYOUT(T, platform):
    tag_size = TAG_WIDTH(n, platform)   // Platform-aware tag width policy

    for each case Cᵢ with payload types P₁, ..., Pₘ:
        case_size[i] = tag_size + Σ SIZEOF(Pⱼ) + alignment_padding
        case_align[i] = max(tag_align, max(ALIGNOF(Pⱼ)))

    // Storage sized for largest case
    storage_size = max(case_size[i])
    storage_align = max(case_align[i])

    return (storage_size, storage_align, per_case_layouts)
```

### 2.2.1 Platform-Aware Tag Width Policy

Tag width is a **platform policy**, not a universal constant. The semantic requirement is that the tag must distinguish N cases (requiring `ceil(log2(N))` bits of information), but the actual width balances:

- **Minimum representation**: Smallest byte-addressable unit that holds case indices
- **Alignment efficiency**: Sub-word tags create padding on word-aligned platforms
- **Memory pressure**: Constrained embedded targets prioritize minimum bytes
- **Access patterns**: Some architectures penalize unaligned or sub-word access

```
TAG_WIDTH(n, platform):
    // Minimum width to represent case indices 0..n-1
    min_width = if n ≤ 256 then 1 else if n ≤ 65536 then 2 else 4

    // Platform policy MAY choose larger for alignment
    // (e.g., word-sized tags on 64-bit for cache efficiency)
    return platform.tag_width_policy(n, min_width)
```

**Default policy**: Use minimum width on all platforms. Implementations MAY provide platform-specific policies that use larger tags for alignment when memory is not constrained.

**Important**: Tag width is NEVER less than 1 byte (`i8`). Even 2-case DUs (like `option`) use `i8` because:
- Bits (`i1`) are not directly addressable on any platform
- DU tags are case indices (0, 1, 2, ...), not boolean truth values
- The abstraction is "minimum bytes to hold case count", not "minimum bits"

### 2.3 Complex Type Resolution

Even with complex nested types, layouts remain tractable:

```fsharp
type Expr =
    | Const of int                           // 1 + 4 = 5 bytes
    | Var of string                          // 1 + (2 words) (fat ptr)
    | Add of Expr * Expr                     // 1 + (2 words) (two ptrs)
    | Lambda of string * Expr * Closure      // 1 + (2 words) + (1 word) + (1 word)
 
```

The compiler computes:
- `Expr` values are pointers (1 word each, uniform)
- `Closure` values are pointers (1 word each, uniform)
- The largest case is `Lambda`; concrete byte totals depend on the platform word. On x86-64 (word = 8 bytes) `Lambda` is `1 + 16 + 8 + 8 = 33` bytes, padded to 40 for alignment; on thumbv8m/M33 (word = 4 bytes) it is `1 + 8 + 4 + 4 = 17` bytes, padded to 20.
- All layouts known at compile time via SRTP resolution of nested types

The same platform-word discipline applies to the fat pointer of `Var`: a string is a pointer plus a length, so 2 words (16 bytes on x86-64, 8 bytes on thumbv8m/M33). Extend the §2.2.1 platform-aware tag-width policy to pointer width the same way: the tag width is a platform policy, and so is the word that a pointer field occupies.

### 2.4 SRTP Constraints for Layout Computation

Layout-relevant SRTP constraints:

```fsharp
// Size constraint - required for inline storage
type 'T when 'T : (static member Size : int)

// Alignment constraint - for struct packing
type 'T when 'T : (static member Align : int)

// BAREWire constraint - for serialization
type 'T when 'T : (static member Serialize : Writer -> 'T -> unit)
              and 'T : (static member Deserialize : Reader -> 'T)
```

These constraints ensure that any type used as a DU payload has the necessary compile-time information for layout computation.

### 2.5 No Runtime Type Discovery

Because DU layouts are fully determined at compile time:
- No type descriptors needed at runtime
- No reflection for case discrimination
- No dynamic dispatch for elimination
- Eliminators are direct, typed memory access

This is the "no runtime" guarantee: the compiler has complete layout knowledge, and generated code operates on known memory shapes.

---

## 3. Memory Representation

### 3.1 DU Value Representation

A DU value is represented as a **pointer to its storage block**. The block is placed by the four-point lifetime lattice of [closure-representation.md §3.3](closure-representation.md), by the same escape analysis that places a closure environment:

1. **scope-bounded**: the block lives on the stack, reclaimed when the scope exits (the inline stack slot of §11.4 is this case).
2. **region-bounded**: the block lives in a [region/arena](memory-regions.md) whose lifetime covers it.
3. **program-lifetime**: the block lives in static storage ([`Sram`](memory-regions.md) when mutable, [`Flash`](memory-regions.md) when immutable), constructed once and held to program end, never freed; it is a global in the same sense a fixed-address register or a linker-carved buffer is.
4. **genuinely-dynamic**: the block lives on the heap.

On a no-heap target only the stack and static placements have a home; a DU value that classifies as dynamic there is a compile-time lifetime error, not a silent heap allocation. Escaping the defining scope does not by itself imply the heap: a DU value that escapes its scope but has a statically known program-long lifetime is placed in static storage.

```
DU Value (at runtime)
┌─────────────────────────────┐
│ ptr: pointer to DU storage  │  (platform word; points into stack, region,
└─────────────────────────────┘   static storage, or heap per §3.3)
```

This uniform pointer representation enables:
- Passing DUs by reference (no copying for large payloads)
- Case-specific interpretation of the pointed-to memory
- Placement in whichever storage class the lifetime lattice selects

### 3.2 DU Storage Layout

The storage block contains a tag followed by case-specific payload. The block occupies whichever storage class the §3.1 lifetime lattice selects (stack, region, static, or heap); its internal layout is the same in every case:

```
DU Storage Block (in the storage class §3.1 selected)
┌─────────────────────────────────────────────────────────┐
│ tag: TagType                        (1-2 bytes)         │
├─────────────────────────────────────────────────────────┤
│ payload: case-specific              (variable size)     │
│   - Inline for small primitives                         │
│   - Pointer for large/complex payloads                  │
│   - Nested DU pointer for recursive cases               │
└─────────────────────────────────────────────────────────┘
```

### 3.3 Tag Encoding

Tag width is determined by platform policy (see §2.2.1). The **minimum** widths are:

| Case Count | Minimum Tag Type | Range |
|------------|------------------|-------|
| 1-256      | `i8`             | 0-255 |
| 257-65536  | `i16`            | 0-65535 |
| >65536     | `i32`            | (rare, supported) |

Tag values are assigned in declaration order, starting from 0.

**Platform considerations**:
- On memory-constrained targets (e.g., Cortex-M with 32KB RAM), use minimum widths
- On word-aligned 64-bit targets, platforms MAY use `i64` tags to avoid padding
- Example: `option<int64>` with `i8` tag = `{i8, 7 padding, i64}` = 16 bytes
- Same type with `i64` tag = `{i64, i64}` = 16 bytes, but naturally aligned

The choice is a platform optimization, not a semantic requirement. All platforms MUST support at least the minimum widths.

### 3.4 Payload Storage Strategies

Each case independently determines its payload storage:

| Payload Type | Storage Strategy | Layout |
|--------------|------------------|--------|
| Primitive (≤8 bytes) | Inline | `{tag, value}` |
| Record/Struct | Inline or Pointer | Based on size |
| Another DU | Pointer | `{tag, ptr}` |
| Closure | Pointer | `{tag, closure_ptr}` |
| Generic `'T` | Monomorphized | Per-instantiation |
| Collection | Pointer | `{tag, collection_ptr}` |

---

## 4. Case Eliminators

### 4.1 Eliminator Concept

A **case eliminator** is a function that safely extracts the payload from a DU value when the case is known. Each case has its own eliminator with the appropriate return type.

```fsharp
// Conceptual eliminator signatures for Number DU
Number.IntVal.eliminate   : ptr<Number> -> int
Number.FloatVal.eliminate : ptr<Number> -> float
Number.tag                : ptr<Number> -> i8
```

### 4.2 Eliminator Semantics

An eliminator:
1. Receives a pointer to DU storage
2. (Optionally) Verifies the tag matches (debug mode)
3. Interprets the payload bytes with case-specific type
4. Returns the typed payload value

### 4.3 Eliminator Generation

For a DU type `T` with case `C` having payload type `P`, the portable body loads the payload field out of the storage block by its case-specific layout offset. The `du.load_field` / `du.store_field` / `du.store_tag` / `du.alloc` spellings below are shorthand for the field access a real lowering performs over `memref` (a computed offset and a `memref.load`/`memref.store`); they stand for the portable field-access intent, not a `du` dialect:

```
func.func @T.C.eliminate(%du: memref<?xi8>) -> P {
    // Interpret the storage block with the case-C layout and read field 1
    %payload = du.load_field %du, case: C, field: 1 : P
    return %payload : P
}
```

Key insight: elimination reinterprets the same storage bits under the case-specific layout. This is **transliteration** (same bits, different type interpretation), not translation. The concrete realization of that reinterpretation is a backend-pathway concern: on the LLVM pathway it is a `llvm.bitcast` to a case-specific `!llvm.struct` followed by `llvm.extractvalue` (see §10.2.1); another pathway realizes the same field read its own way.

### 4.4 Multi-Field Case Eliminators

For cases with multiple fields:

```fsharp
type Shape =
    | Rectangle of width: float * height: float
```

The eliminator returns a tuple:

```
func.func @Shape.Rectangle.eliminate(%du: memref<?xi8>) -> (float, float) {
    // Read fields 1 and 2 under the Rectangle case layout {i8, float, float}
    %width  = du.load_field %du, case: Rectangle, field: 1 : float
    %height = du.load_field %du, case: Rectangle, field: 2 : float
    return (%width, %height)
}
```

### 4.5 Recursive Case Eliminators

For recursive DUs:

```fsharp
type Expr =
    | Add of Expr * Expr
```

The eliminator returns pointers to nested DUs (each a `memref` into the nested storage block):

```
func.func @Expr.Add.eliminate(%du: memref<?xi8>) -> (memref<?xi8>, memref<?xi8>) {
    // Read the two nested-Expr pointer fields under the Add case layout
    %left  = du.load_field %du, case: Add, field: 1 : memref<?xi8>   // left Expr
    %right = du.load_field %du, case: Add, field: 2 : memref<?xi8>   // right Expr
    return (%left, %right)
}
```

---

## 5. Case Constructors

### 5.1 Constructor Concept

A **case constructor** creates a DU value by allocating storage and initializing the tag and payload.

```fsharp
// Conceptual constructor signatures for Number DU
Number.IntVal.construct   : arena * int -> ptr<Number>
Number.FloatVal.construct : arena * float -> ptr<Number>
```

### 5.2 Constructor Semantics

A constructor:
1. Computes storage size for the case
2. Reserves the storage in the class the §3.1 lifetime lattice selected for the value (a stack slot, a region/arena, static storage, or the heap); an arena is threaded in only when the value classifies as region-bounded
3. Stores the tag value
4. Stores the payload with case-specific type
5. Returns pointer to the DU storage

### 5.3 Constructor Generation

The portable body reserves storage for the case and writes the tag and payload fields under the case layout. The reserved size is the case's computed `storage_size` (§2.2), not a fixed constant; for `IntVal` that is `1` tag byte plus a 4-byte `i32` payload, rounded to the case alignment:

```
func.func @Number.IntVal.construct(%arena: memref<?xi8>, %val: i32) -> memref<?xi8> {
    // Reserve storage sized for the IntVal case (tag + i32, aligned)
    %du = du.alloc %arena, case: IntVal : memref<?xi8>

    // Write the tag and payload fields under the IntVal layout
    du.store_tag   %du, 0 : i8            // tag = 0
    du.store_field %du, case: IntVal, field: 1, %val : i32

    return %du : memref<?xi8>
}
```

On the LLVM pathway this lowers to a `llvm.bitcast` to `!llvm.struct<(i8, i32)>` followed by `llvm.insertvalue`/`llvm.store` (the mirror of §10.2.1); the `insertvalue`/`bitcast` forms belong to that pathway.

---

## 6. Pattern Matching Compilation

### 6.1 Match Expression Decomposition

A `match` expression over a DU compiles to:
1. Tag extraction
2. Tag comparison (switch/if-chain)
3. Case eliminator calls in each branch

```fsharp
match number with
| IntVal i -> formatInt i
| FloatVal f -> formatFloat f
```

Compiles to:

```
func @formatNumber(%du_ptr: ptr) -> string {
    %tag = call @Number.tag(%du_ptr)
    switch %tag {
        case 0: {
            %i = call @Number.IntVal.eliminate(%du_ptr)
            %result = call @formatInt(%i)
            return %result
        }
        case 1: {
            %f = call @Number.FloatVal.eliminate(%du_ptr)
            %result = call @formatFloat(%f)
            return %result
        }
    }
}
```

### 6.2 Nested Pattern Matching

For patterns that destructure nested DUs:

```fsharp
match expr with
| Add(Const a, Const b) -> a + b
| _ -> 0
```

Compiles to nested eliminator calls and tag checks.

### 6.3 Guard Expressions

Pattern guards execute after successful tag match but before committing to the branch:

```fsharp
match number with
| IntVal i when i > 0 -> "positive"
| IntVal i -> "non-positive"
| FloatVal _ -> "float"
```

---

## 7. BAREWire Integration

### 7.0 The Contract Model: BAREWire and BARE

BAREWire is the structured interchange contract at the runtime boundary. BARE is the external binary encoding BAREWire builds on; BAREWire is not that encoding but the typed contract layered over it. The two SHALL NOT be conflated: a discriminated union does not cross a boundary "through the BARE protocol," and BAREWire is not "a discriminated-union-aware memory layout." The native representation of §§1-6 is the in-memory layout; BAREWire is the boundary contract that transports a value described by that layout from one endpoint to another.

Both endpoints of a boundary derive their read and write code from the same union type at compile time. Consequently:

1. **The tag is a compile-time case index, not a runtime type discriminator.** The tag written on the wire (§7.1) is the same declaration-order case index (0..n-1) assigned in §2.2.1 and §3.3, serialized over an *already monomorphized* sum type. It is an interchange convention identifying which case was sent, not a token the receiver uses to recover type information it otherwise lacks. There is no runtime type discrimination, no reflection, and no self-describing type descriptor on the wire (consistent with §1.2 principle 5 and §2.5).

2. **Conformance is by construction, and violations are caught at the message fabric.** Because both sides interpret the same contract, a message whose case set, payload type, or dimensional annotation does not conform to the agreed union contract is rejected at the boundary rather than admitted and failed downstream. The receiving endpoint reconstructs the same native discriminated union, placed by the receiver's own §3.1 lifetime lattice, that it would have held had the value never left the process.

3. **The compiler is never in the runtime loop.** The Composer establishes the contract at compile time; the two runtime endpoints honor it using BAREWire alone. JSON or otherwise self-describing tagging is a legacy-interface accommodation only, never the canonical mechanism.

> **Informative**: This contract model is what lets a discriminated union preserve its case structure, payload types, and dimensional annotations across an address-space, process, or hardware boundary "by construction." Downstream frameworks rely on exactly this property where the typed contract is treated as a structure-preserving map between endpoints; that reliance is on the by-construction contract of this section, not on the byte layout of §7.1 alone.

### 7.1 Serialization

DU serialization follows the BAREWire union contract of §7.0. The wire form is the tag (the compile-time case index) followed by the case-specific payload:
1. Write tag as varint or fixed-width integer
2. Write case-specific payload using case's serializer

```
serialize_Number(writer, du_ptr):
    tag = load du_ptr[0]
    write_u8(writer, tag)
    switch tag:
        case 0: 
            val = Number.IntVal.eliminate(du_ptr)
            write_i32(writer, val)
        case 1:
            val = Number.FloatVal.eliminate(du_ptr)
            write_f64(writer, val)
```

### 7.2 Deserialization

```
deserialize_Number(reader, arena) -> ptr:
    tag = read_u8(reader)
    switch tag:
        case 0:
            val = read_i32(reader)
            return Number.IntVal.construct(arena, val)
        case 1:
            val = read_f64(reader)
            return Number.FloatVal.construct(arena, val)
```

### 7.3 Schema Evolution

BAREWire's union schema supports:
- Adding new cases (new tag values)
- Deprecating cases (tag reserved, payload ignored)
- Payload type evolution (within case)

---

## 8. Storage Placement

### 8.1 Placement Context

DU construction places the storage block in the class the lifetime lattice of §3.1 selects for the value. A region/arena is one of those four classes, used when the value is region-bounded; it is not a precondition of construction:

```fsharp
// Scope-bounded: the block is a stack slot, reclaimed at scope exit
let makeNumber (x: int) : Number = IntVal x   // consumed within the caller's scope

// Region-bounded: the arena that covers the value is threaded in
using arena = Arena.create() in
    let n = IntVal 42   // block placed in 'arena'
    consume n

// Program-lifetime: a value constructed once and held to program end
// is placed in static storage (Sram/Flash), never freed — no arena involved
let table = Add(Const 1, Const 2)   // held by the entry point for the whole run
```

On a no-heap target only the scope-bounded (stack) and program-lifetime (static) classes have a home. A DU value that classifies as genuinely dynamic there is a compile-time lifetime error (§3.1), not a silent heap allocation.

### 8.2 Lifetime Constraints

A DU value SHALL NOT outlive the storage class that holds it. The failure mode differs by class:

```fsharp
let result =
    using arena = Arena.create() in
    let n = IntVal 42   // block placed in 'arena'
    n  // ERROR: 'n' escapes 'arena'; it is neither scope-bounded nor program-lifetime here
```

Escaping a scope does not by itself force this error. A value whose lifetime is statically known to be the whole program is placed in static storage and may be returned and held; only a value that escapes into a lifetime no covering class can satisfy is rejected (and on a no-heap target, "dynamic" is one such rejection).

A closure-valued payload adds a constraint of its own. Constructing a DU value with a closure payload SHALL be an escape point for that closure ([closure-representation.md §3.3](closure-representation.md)): the block stores a reference to the closure, not a copy (§3.4, §9.1), so the closure environment's lifetime class SHALL dominate the storage class of every DU block that holds a reference to it. Escape analysis SHALL place the environment in a class whose lifetime covers each such block; a construction for which no covering class exists SHALL be rejected at compile time.

### 8.3 Recursive Structure Placement

For recursive DUs, all nodes in a tree typically share one storage class: the region that covers the tree when it is region-bounded, or static storage when the whole tree is program-lifetime.

```fsharp
let expr = Add(Const 1, Add(Const 2, Const 3))
// All four nodes share the class §3.1 selected for 'expr'
```

---

## 9. Complex Payloads

### 9.1 Closure Payloads

When a DU case contains a function/[closure](closure-representation.md):

```fsharp
type Lazy<'T> =
    | Evaluated of 'T
    | Deferred of (unit -> 'T)
```

The `Deferred` case stores a closure pointer:

```
Deferred layout: { tag: i8, closure_ptr: ptr }
```

The eliminator returns the closure, which can then be invoked.

### 9.2 Generic DU Instantiation

Generic DUs are monomorphized:

```fsharp
type Option<'T> =
    | None
    | Some of 'T

let x : Option<int> = Some 42      // Option_int
let y : Option<string> = Some "hi" // Option_string
 
```

Each instantiation generates its own eliminators/constructors with concrete types.

### 9.3 DU Containing DU

When a DU case contains another DU:

```fsharp
type Result<'T, 'E> =
    | Ok of 'T
    | Error of 'E

type Response =
    | Success of Result<string, int>
    | Timeout
```

The `Success` eliminator returns a `Result` pointer, which can then be further eliminated.

---

## 10. MLIR Representation

### 10.1 DU Type in MLIR

At the portable middle-end level a DU value is a pointer into its storage block, carried as a `memref` to the block or, where the storage address is a value threaded through the program, as a platform-sized `index` (the deferred pointer form: `NTUptr` maps to `TIndex`, not to `!llvm.ptr`). The middle end commits to no target:

```mlir
// DU value as a memref to its storage block (portable)
%du : memref<?xi8>
```

The `!llvm.ptr` form appears only after a target pathway (the LLVM/CPU/MCU pathway) has been selected; it is a lowering of the portable form, not what the middle end emits. See §10.2.1 for the LLVM-pathway realization.

### 10.2 Generated Functions

For each DU type `T` with cases `C₁, ..., Cₙ`, the middle end emits `func.func` declarations over portable types (the storage pointer as `memref<?xi8>`, the arena handle as `memref`):

```mlir
// Tag accessor
func.func @T.tag(%du: memref<?xi8>) -> TagType

// Per-case eliminators
func.func @T.C₁.eliminate(%du: memref<?xi8>) -> PayloadType₁
func.func @T.C₂.eliminate(%du: memref<?xi8>) -> PayloadType₂
...

// Per-case constructors
func.func @T.C₁.construct(%arena: memref<?xi8>, %payload: PayloadType₁) -> memref<?xi8>
func.func @T.C₂.construct(%arena: memref<?xi8>, %payload: PayloadType₂) -> memref<?xi8>
...

// BAREWire serialization (if enabled)
func.func @T.serialize(%writer: memref<?xi8>, %du: memref<?xi8>) -> i32
func.func @T.deserialize(%reader: memref<?xi8>, %arena: memref<?xi8>) -> memref<?xi8>
```

### 10.2.1 LLVM-Pathway Realization

After the LLVM target pathway is selected (CPU/MCU targets), the portable storage pointers lower to `!llvm.ptr` and the eliminator body realizes the case-specific interpretation with LLVM ops:

```mlir
// LLVM pathway only — after target commitment
func.func @Number.FloatVal.eliminate(%du: !llvm.ptr) -> f64 {
    // opaque !llvm.ptr: load the case-specific struct directly, no ptr bitcast
    %struct = llvm.load %du : !llvm.ptr -> !llvm.struct<(i8, i64)>
    %bits = llvm.extractvalue %struct[1] : !llvm.struct<(i8, i64)>
    %f = llvm.bitcast %bits : i64 to f64
    return %f : f64
}
```

A different pathway realizes the same portable eliminator its own way (the FPGA/CIRCT pathway as a field select over a hardware struct, for instance). The `llvm.bitcast` / `!llvm.ptr` forms belong to this pathway, not to the middle end.

### 10.2.2 JSIR-Pathway Realization

> This section binds implementations claiming the **JavaScript Substrate** profile ([Conformance §7](conformance.md)).

The JSIR pathway realizes the generated-function surface of §10.2 over a host object in place of a storage block, under the carrier-realization rule of [Backend Lowering Architecture §4.5](backend-lowering-architecture.md):

1. A DU value SHALL be realized as a host object carrying a discriminant and the case payload.
2. The discriminant value SHALL be the declaration-order case index of §2.2.1. This is the same index the §7.0 wire contract serializes, so a value's in-artifact discriminant and its wire tag agree by construction.
3. `@T.tag` SHALL realize as a read of the discriminant, `@T.Cᵢ.eliminate` as reads of the payload, and `@T.Cᵢ.construct` as construction of the host object. No arena handle exists on this pathway: the host garbage collector owns placement, and every lifetime class of §3.1 has a home.
4. Where a layout-realizing pathway elides the tag of a single-case union ([Native Type Universe §3.3](native-type-universe.md)), this pathway likewise realizes the value as its payload directly.
5. The property names carrying the discriminant and the payload are implementation-defined and SHALL be documented ([Behavior Classification §2](behavior-classification.md)); the discriminant's value is fixed by item 2.
6. Requirement 4 of §12 (tag integrity) binds emitted Clef code: the discriminant is written only by constructors. It does not extend to a host object that foreign code has held. A DU-shaped value returning from foreign code is foreign, and SHALL re-enter Clef only by narrowing ([JavaScript Boundary Semantics §3.2](javascript-boundary.md)).

### 10.3 Inlining Optimization

For simple DUs (all cases primitive, small payloads), eliminators and constructors MAY be inlined at call sites. The semantic model remains the same.

---

## 11. PSG Representation

### 11.1 SemanticKind Extensions

New semantic kinds for DU operations:

```fsharp
type SemanticKind =
    // ... existing kinds ...

    /// Extract tag from DU value (returns i8)
    | DUGetTag of scrutinee: NodeId * duType: NativeType

    /// Eliminate (extract payload from) a specific case
    | DUEliminate of scrutinee: NodeId * caseIndex: int * caseName: string * payloadType: NativeType

    /// Construct a DU case value (arenaHint names the region when the value is region-bounded per §3.1; None for the stack, static, and heap classes)
    | DUConstruct of caseName: string * caseIndex: int * payload: NodeId option * arenaHint: NodeId option
```

### 11.2 Match Decomposition

Baker decomposes `match` over DUs into:
1. `DUGetTag` for tag extraction
2. Conditional branching on tag value
3. `DUEliminate` in each branch for payload extraction

### 11.3 Type Information Flow

The `DUEliminate` node carries:
- The case name and index (for code generation)
- The payload type (for type-safe extraction)
- Reference to the DU type definition (for eliminator lookup)

### 11.4 Inline Storage and Bitcasting

For inline DUs (small, stack-allocated), all cases share a **fixed-size slot** that accommodates the largest payload:

```
Number = IntVal of int | FloatVal of float

Storage: { tag: i8, slot: i64 }
         ↑        ↑
         1 byte   8 bytes (holds both int64 and float64 bits)
```

**Construction:** When the payload type differs from the slot type, the value is transliterated (same bits, reinterpreted) into the slot type before it is written into the field. In portable terms the constructor writes the payload field under the fixed slot layout; the reinterpretation itself has no portable op and is carried to the target pathway. On the LLVM pathway it realizes as `arith.constant` followed by `llvm.bitcast` and `llvm.insertvalue`:

```mlir
// FloatVal 3.14 construction — LLVM pathway
%f = arith.constant 3.14 : f64
%bits = llvm.bitcast %f : f64 to i64        // ← reinterpret into slot type
%result = llvm.insertvalue %bits, %struct[1] : !llvm.struct<(i8, i64)>
```

**Elimination:** Symmetric. Read the slot field, then reinterpret the raw bits as the payload type. On the LLVM pathway:

```mlir
// FloatVal elimination — LLVM pathway
%bits = llvm.extractvalue %struct[1] : i64   // ← raw slot bits
%f = llvm.bitcast %bits : i64 to f64        // ← interpret as float
 
```

**SSA Cost Implication:** The SSA count for `DUConstruct` depends on whether bitcast is needed - this is computed deterministically from the type comparison at SSA assignment time.

---

## 12. Normative Requirements

1. **Pointer Representation**: DU values SHALL be represented as pointers to a storage block. The block SHALL be placed by the §3.1 lifetime lattice in the storage class whose lifetime covers the value (stack, region, static storage, or heap).

2. **Type-Safe Elimination**: Each case SHALL have a dedicated eliminator that returns the payload with its declared type.

3. **No Type Coercion in Eliminators**: Eliminators SHALL NOT perform type conversion; they interpret memory with the case-specific type.

4. **Tag Integrity**: The tag value SHALL be set by constructors and read by eliminators; direct tag manipulation is undefined behavior.

5. **Lattice-Driven Placement**: DU construction SHALL place the storage block in the storage class the §3.1 lifetime lattice selects for the value (stack when scope-bounded, a region when region-bounded, static storage when program-lifetime, the heap only when genuinely dynamic). An arena/region SHALL be threaded into construction only for a region-bounded value; it SHALL NOT be a precondition of construction for the other classes. On a target without a heap, a DU value that classifies as dynamic SHALL be a compile-time lifetime error, not a heap allocation.

6. **Lifetime Safety**: A DU value SHALL NOT outlive the storage class that holds it. A value whose lifetime is statically known to be the whole program is placed in static storage and MAY be returned and held; a value that escapes into a lifetime no covering class can satisfy SHALL be rejected at compile time.

7. **Monomorphization**: Generic DUs SHALL be fully monomorphized; no runtime type parameters.

8. **BAREWire Compatibility**: DU serialization SHALL follow the BAREWire union contract (§7.0). The wire tag SHALL be the compile-time case index, and endpoints SHALL derive read/write code from the same union type so that conformance holds by construction and non-conforming messages are rejected at the boundary. BAREWire SHALL NOT be conflated with the external BARE encoding it builds on.

9. **Recursive Support**: DUs MAY contain cases with payloads of the same or other DU types.

10. **Inlining Permitted**: Implementations MAY inline eliminators/constructors for optimization while preserving semantics.

11. **JSIR-Pathway Realization**: On the JSIR pathway a union SHALL be realized per §10.2.2. Requirements 1 and 5 are requirements on layout-realizing pathways and bind pathways that realize memory layouts; the remaining requirements bind on every pathway, with placement discharged by the host garbage collector on this pathway.

---

## 13. Examples

### 13.1 Simple DU

```fsharp
type Color = Red | Green | Blue
```

- Tag-only, no payloads
- Storage: 1 byte (tag only)
- Eliminators: None (just tag comparison)

### 13.2 DU with Heterogeneous Payloads

```fsharp
type Value =
    | Int of int
    | Float of float
    | String of string
    | Pair of Value * Value
```

- Four cases with different payload sizes
- `Pair` is recursive (contains two `Value` pointers)
- Each case has its own layout and eliminator

### 13.3 Result Type

```fsharp
type Result<'T, 'E> =
    | Ok of 'T
    | Error of 'E
```

- Two-case generic DU
- Monomorphized per instantiation
- `Result<int, string>` → `Result_int_string`

---
