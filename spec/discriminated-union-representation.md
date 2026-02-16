---
title: "Discriminated Union Representation in Clef"
weight: 9020
draft: true
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
    | Add of Expr * Expr              // two recursive pointers
    | Lambda of string * Expr * env   // complex nested structure
```

A naive "max-size union" representation loses type information:
- Extracting a `FloatVal` payload requires knowing it's a `float`, not just "8 bytes"
- Nested DUs create recursive type relationships
- Generic DU cases require monomorphization

### 1.2 Design Principles

1. **Case eliminators preserve type safety**: Each case has a typed accessor
2. **Pointer-based uniformity**: DU values are pointers to region-allocated storage
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
| `Container<Number>` | `Number` (ptr) | 8 bytes | 16 bytes |

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
    | Var of string                          // 1 + 16 = 17 bytes (fat ptr)
    | Add of Expr * Expr                     // 1 + 8 + 8 = 17 bytes (two ptrs)
    | Lambda of string * Expr * Closure      // 1 + 16 + 8 + 8 = 33 bytes
```

The compiler computes:
- `Expr` values are pointers (8 bytes, uniform)
- `Closure` values are pointers (8 bytes, uniform)
- Largest case is `Lambda` at 33 bytes (padded to 40 for alignment)
- All layouts known at compile time via SRTP resolution of nested types

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

A DU value is represented as a **pointer to a region-allocated block**:

```
DU Value (at runtime)
┌─────────────────────────────┐
│ ptr: pointer to DU storage  │  (word-sized, points to region)
└─────────────────────────────┘
```

This uniform pointer representation enables:
- Passing DUs by reference (no copying for large payloads)
- Case-specific interpretation of the pointed-to memory
- Natural fit with arena/region allocation

### 3.2 DU Storage Layout

The storage block contains a tag followed by case-specific payload:

```
DU Storage Block (in region)
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

For a DU type `T` with case `C` having payload type `P`:

```
func @T.C.eliminate(%du_ptr: ptr) -> P {
    // Load with case-specific struct type
    %typed_ptr = bitcast %du_ptr to ptr<struct<(TagType, P)>>
    %struct = load %typed_ptr : struct<(TagType, P)>
    %payload = extractvalue %struct[1] : P
    return %payload
}
```

Key insight: The `bitcast` reinterprets the pointer, allowing case-specific struct types. This is **transliteration** (same bits, different type interpretation), not translation.

### 4.4 Multi-Field Case Eliminators

For cases with multiple fields:

```fsharp
type Shape =
    | Rectangle of width: float * height: float
```

The eliminator returns a tuple:

```
func @Shape.Rectangle.eliminate(%du_ptr: ptr) -> (float, float) {
    %typed_ptr = bitcast %du_ptr to ptr<struct<(i8, float, float)>>
    %struct = load %typed_ptr
    %width = extractvalue %struct[1]
    %height = extractvalue %struct[2]
    return (%width, %height)
}
```

### 4.5 Recursive Case Eliminators

For recursive DUs:

```fsharp
type Expr =
    | Add of Expr * Expr
```

The eliminator returns pointers to nested DUs:

```
func @Expr.Add.eliminate(%du_ptr: ptr) -> (ptr, ptr) {
    %typed_ptr = bitcast %du_ptr to ptr<struct<(i8, ptr, ptr)>>
    %struct = load %typed_ptr
    %left = extractvalue %struct[1]   // ptr to left Expr
    %right = extractvalue %struct[2]  // ptr to right Expr
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
2. Allocates from the appropriate region/arena
3. Stores the tag value
4. Stores the payload with case-specific type
5. Returns pointer to the DU storage

### 5.3 Constructor Generation

```
func @Number.IntVal.construct(%arena: ptr, %val: i32) -> ptr {
    // Allocate: 1 byte tag + 4 bytes payload (aligned to 8)
    %size = constant 8 : i64
    %du_ptr = call @arena.alloc(%arena, %size)
    
    // Store as case-specific struct type
    %typed_ptr = bitcast %du_ptr to ptr<struct<(i8, i32)>>
    %struct.1 = insertvalue undef<struct<(i8, i32)>>[0], 0 : i8  // tag = 0
    %struct.2 = insertvalue %struct.1[1], %val : i32
    store %struct.2, %typed_ptr
    
    return %du_ptr
}
```

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

### 7.1 Serialization

DU serialization follows BAREWire union conventions:
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

## 8. Arena/Region Integration

### 8.1 Allocation Context

DU construction requires an arena/region for allocation:

```fsharp
// Explicit arena
let n = IntVal 42  // allocated in current region context

// Function parameter (implicit)
let makeNumber (x: int) : Number = IntVal x
// Arena passed implicitly through calling convention
```

### 8.2 Lifetime Constraints

DU values inherit the lifetime of their containing region:

```fsharp
let result = 
    using arena = Arena.create() in
    let n = IntVal 42  // allocated in 'arena'
    n  // ERROR: 'n' cannot escape 'arena' lifetime
```

### 8.3 Recursive Structure Allocation

For recursive DUs, all nodes in a tree typically share the same arena:

```fsharp
let expr = Add(Const 1, Add(Const 2, Const 3))
// All four nodes allocated in same arena
```

---

## 9. Complex Payloads

### 9.1 Closure Payloads

When a DU case contains a function/closure:

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

DUs are represented as opaque pointers at the MLIR level:

```mlir
// DU values are always pointers
%du : !llvm.ptr
```

### 10.2 Generated Functions

For each DU type `T` with cases `C₁, ..., Cₙ`:

```mlir
// Tag accessor
func.func @T.tag(%du: !llvm.ptr) -> TagType

// Per-case eliminators
func.func @T.C₁.eliminate(%du: !llvm.ptr) -> PayloadType₁
func.func @T.C₂.eliminate(%du: !llvm.ptr) -> PayloadType₂
...

// Per-case constructors
func.func @T.C₁.construct(%arena: !llvm.ptr, %payload: PayloadType₁) -> !llvm.ptr
func.func @T.C₂.construct(%arena: !llvm.ptr, %payload: PayloadType₂) -> !llvm.ptr
...

// BAREWire serialization (if enabled)
func.func @T.serialize(%writer: !llvm.ptr, %du: !llvm.ptr) -> i32
func.func @T.deserialize(%reader: !llvm.ptr, %arena: !llvm.ptr) -> !llvm.ptr
```

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

    /// Construct a DU case value (arenaHint for future heap allocation)
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

**Construction:** When the payload type differs from the slot type, the value is bitcast before insertion:

```mlir
// FloatVal 3.14 construction
%f = arith.constant 3.14 : f64
%bits = llvm.bitcast %f : f64 to i64        // ← bitcast to slot type
%result = llvm.insertvalue %bits, %struct[1] : !llvm.struct<(i8, i64)>
```

**Elimination:** Symmetric - extract then bitcast:

```mlir
// FloatVal elimination
%bits = llvm.extractvalue %struct[1] : i64   // ← raw slot bits
%f = llvm.bitcast %bits : i64 to f64        // ← interpret as float
```

**SSA Cost Implication:** The SSA count for `DUConstruct` depends on whether bitcast is needed - this is computed deterministically from the type comparison at SSA assignment time.

---

## 12. Normative Requirements

1. **Pointer Representation**: DU values SHALL be represented as pointers to region-allocated storage.

2. **Type-Safe Elimination**: Each case SHALL have a dedicated eliminator that returns the payload with its declared type.

3. **No Type Coercion in Eliminators**: Eliminators SHALL NOT perform type conversion; they interpret memory with the case-specific type.

4. **Tag Integrity**: The tag value SHALL be set by constructors and read by eliminators; direct tag manipulation is undefined behavior.

5. **Arena Requirement**: DU construction SHALL require an arena/region for allocation.

6. **Lifetime Safety**: A DU value SHALL NOT escape the lifetime of its containing region.

7. **Monomorphization**: Generic DUs SHALL be fully monomorphized; no runtime type parameters.

8. **BAREWire Compatibility**: DU serialization SHALL follow BAREWire union encoding conventions.

9. **Recursive Support**: DUs MAY contain cases with payloads of the same or other DU types.

10. **Inlining Permitted**: Implementations MAY inline eliminators/constructors for optimization while preserving semantics.

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

## See Also

- [Closure Representation](closure-representation.md) - Closures as DU payloads
- [Patterns](patterns.md) - Pattern matching syntax
- [Type Definitions](type-definitions.md) - DU declaration syntax
- [Memory Regions](memory-regions.md) - Arena allocation
- [BAREWire Integration](../rationale/barewire-integration.md) - Serialization protocol
