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
    | Var of string                   // memref<?xi8> view (buffer + extent)
    | Add of Expr * Expr              // two index links (1 word each)
    | Lambda of string * Expr * env   // complex nested structure
 
```

A naive "max-size union" representation loses type information:
- Extracting a `FloatVal` payload requires knowing it's a `float`, not just "8 bytes"
- Nested DUs create recursive type relationships
- Generic DU cases require monomorphization

### 1.2 Design Principles

1. **Case eliminators preserve type safety**: Each case has a typed accessor
2. **View-based uniformity**: a DU value is a `memref` view of its storage block, placed by the lifetime lattice of §3.1 (stack, region, static, or heap); a link to an arena-resident payload is a bounded `index`, never an address
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
| `Container<Number>` | `Number` (`index` link) | 1 word | 2 words |

An `index` link occupies one platform word: 8 bytes on x86-64, 4 bytes on thumbv8m/M33 ([Native Type Universe §3.3](native-type-universe.md)). The `Container<Number>` row above is the x86-64 case (1 word = 8 bytes, 2 words = 16 bytes); on thumbv8m/M33 the same instantiation is 4 bytes and 8 bytes.

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
    | Var of string                          // 1 + (2 words) (memref<?xi8> view: base index + extent)
    | Add of Expr * Expr                     // 1 + (2 words) (two index links)
    | Lambda of string * Expr * Closure      // 1 + (2 words) + (1 word) + (1 word)
```

The compiler computes:
- `Expr` links are `index` values (1 word each, uniform): arena-relative offsets carrying VC-LINK (§3.4)
- A `Closure` payload occupies 1 word: its environment view `memref<Exi8>`; the code component is fixed by the closure form at saturation and is not a field (§9.1)
- The largest case is `Lambda`; concrete byte totals depend on the platform word. On x86-64 (word = 8 bytes) `Lambda` is `1 + 16 + 8 + 8 = 33` bytes, padded to 40 for alignment; on thumbv8m/M33 (word = 4 bytes) it is `1 + 8 + 4 + 4 = 17` bytes, padded to 20.
- All layouts known at compile time via SRTP resolution of nested types

The same platform-word discipline applies to the `memref<?xi8>` view of `Var`: a string is its buffer, and the view carries the buffer's base `index` and its extent, so 2 words (16 bytes on x86-64, 8 bytes on thumbv8m/M33). There is no length word beside an address: the length is the memref's dimension ([Native Type Mappings](native-type-mappings.md)). Extend the §2.2.1 platform-aware tag-width policy to the `index` width the same way: the tag width is a platform policy, and so is the word an `index` link or view field occupies.

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

A DU value is a **`memref` view of its storage block**: `memref<Exi8>`, where `E` is the type's `storage_size` of §2.2, a literal settled at saturation. The block is placed by the four-point lifetime lattice of [Closure Representation §3.3](closure-representation.md), by the same escape analysis that places a closure environment:

1. **scope-bounded**: the block lives on the stack (`memref.alloca`), reclaimed when the scope exits (the inline stack slot of §11.4 is this case).
2. **region-bounded**: the block lives in a [region/arena](memory-regions.md) whose lifetime covers it, at an arena-relative offset that is the value's `index` link within that arena (§3.4).
3. **program-lifetime**: the block lives in the platform's declared program-lifetime space, cited by name from the platform description through a `Resides` edge ([Program Hypergraph §6](program-hypergraph.md)): its declared immutable space when the block is never written, its declared mutable space otherwise (rodata and data on an ELF target; the [`Flash`](memory-regions.md) and [`Sram`](memory-regions.md) regions on an MCU; constant memory on a GPU; initialised BRAM on an FPGA). It is emitted as a `memref.global`, constructed once and held to program end, never freed; it is a global in the same sense a fixed-address register or a linker-carved buffer is.
4. **genuinely-dynamic**: the block lives on the heap.

On a no-heap target only the stack and static placements have a home; a DU value that classifies as dynamic there is a compile-time lifetime error, not a silent heap allocation. Escaping the defining scope does not by itself imply the heap: a DU value that escapes its scope but has a statically known program-long lifetime is placed in program-lifetime space.

```
DU Value (at runtime)
┌──────────────────────────────────┐
│ %du : memref<Exi8>               │  view of the storage block, E = storage_size (§2.2);
└──────────────────────────────────┘  the block sits on the stack, in an arena, in
                                      program-lifetime space, or on the heap per the lattice above
```

No DU value is an address and no DU value is null. A link from one block to another arena-resident block is a bounded `index`, never an address (§3.4). This uniform view representation enables:
- Passing DUs by view (no copying for large payloads)
- Case-specific interpretation of the viewed bytes
- Placement in whichever storage class the lifetime lattice selects

### 3.2 DU Storage Layout

The storage block contains a tag followed by case-specific payload. The block occupies whichever storage class the §3.1 lifetime lattice selects (stack, region, static, or heap); its internal layout is the same in every case:

```
DU Storage Block (in the storage class §3.1 selected)
┌─────────────────────────────────────────────────────────┐
│ tag: TagType                        (1-2 bytes)         │
├─────────────────────────────────────────────────────────┤
│ payload: case-specific              (variable size)     │
│   - Inline for small primitives and small aggregates    │
│   - A memref view for a buffer payload (string, array)  │
│   - An index link for an arena-resident payload         │
│     (a nested DU, a recursive case, a collection)       │
│   - The environment view for a closure payload (§9.1)   │
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
| Record/Struct | Inline or Link | Based on size; a linked record is arena-resident |
| String / Array | View | `{tag, memref<?xT>}`: the buffer is the value, its length is the memref's dimension |
| Another DU | Link | `{tag, index}` |
| Closure | Environment view | `{tag, env: memref<Exi8>}`; the code component is fixed by the closure form (§9.1) |
| Generic `'T` | Monomorphized | Per-instantiation |
| Collection | Link | `{tag, index}` |

A **link** is an `index`: an arena-relative offset into the arena buffer that holds the linked block, one platform word, never an address. Every link word carries the range obligation VC-LINK, `0 <= i < extent(arena)`, quantifier-free (QF_LIA over literals) and discharged at saturation before witnessing ([Native Type Universe §3.3](native-type-universe.md)). A link load is one `memref.load` of an `index`; the linked block is viewed with `memref.view` on the arena at that offset (§4.5). Where a case links a collection, the empty collection is that collection's static sentinel node at offset 0 of the arena, whose image resides in the platform's declared immutable program-lifetime space; emptiness is the collection's literal test, never a null check ([List Operations Representation §5.2](list-operations-representation.md)). No link is ever absent.

---

## 4. Case Eliminators

### 4.1 Eliminator Concept

A **case eliminator** is a function that safely extracts the payload from a DU value when the case is known. Each case has its own eliminator with the appropriate return type. Every eliminator takes the DU value as its `memref<Exi8>` view (§3.1), `E` being the type's `storage_size` of §2.2:

```fsharp
// Conceptual eliminator signatures for Number DU
Number.IntVal.eliminate   : memref<Exi8> -> int
Number.FloatVal.eliminate : memref<Exi8> -> float
Number.tag                : memref<Exi8> -> i8
```

### 4.2 Eliminator Semantics

An eliminator:
1. Receives the DU value, a `memref<Exi8>` view of its storage block
2. (Optionally) Verifies the tag matches (debug mode)
3. Interprets the payload bytes with case-specific type
4. Returns the typed payload value

### 4.3 Eliminator Generation

For a DU type `T` with case `C` having payload type `P`, the portable body loads the payload field out of the storage block by its case-specific layout offset. The `du.load_field` / `du.store_field` / `du.store_tag` / `du.alloc` spellings below are shorthand for the field access a real lowering performs over `memref` (a static `memref.view` at the field's literal offset and a `memref.load`/`memref.store`); they stand for the portable field-access intent, not a `du` dialect:

```
func.func @T.C.eliminate(%du: memref<Exi8>) -> P {
    // Interpret the storage block with the case-C layout and read field 1
    %payload = du.load_field %du, case: C, field: 1 : P
    return %payload : P
}
```

Key insight: elimination reinterprets the same storage bits under the case-specific layout. This is **transliteration** (same bits, different type interpretation), not translation. The concrete realization of that reinterpretation is a backend-pathway concern: on the LLVM pathway it is an `llvm.load` of the case-specific `!llvm.struct` followed by `llvm.extractvalue` (see §10.2.1); another pathway realizes the same field read its own way.

### 4.4 Multi-Field Case Eliminators

For cases with multiple fields:

```fsharp
type Shape =
    | Rectangle of width: float * height: float
```

The eliminator returns a tuple:

```
func.func @Shape.Rectangle.eliminate(%du: memref<Exi8>) -> (float, float) {
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

The eliminator returns the two links of the nested blocks, each an `index` into the arena that holds the tree (§3.4, §8.3), carrying VC-LINK:

```
func.func @Expr.Add.eliminate(%du: memref<Exi8>) -> (index, index) {
    // Read the two nested-Expr link fields under the Add case layout
    %left  = du.load_field %du, case: Add, field: 1 : index   // left Expr link
    %right = du.load_field %du, case: Add, field: 2 : index   // right Expr link
    return (%left, %right)
}
```

The caller recovers a nested value from its link by viewing the arena at that offset; the view is the `memref<Exi8>` every eliminator of §4.1 takes:

```
%left_du = memref.view %arena[%left][] : memref<?xi8> to memref<Exi8>
%tag = func.call @Expr.tag(%left_du) : (memref<Exi8>) -> i8
```

No link is absent and no link is tested for absence: VC-LINK bounds every link word within the arena's extent at saturation, and recursion over a recursive DU terminates at a leaf case, never at a null.

---

## 5. Case Constructors

### 5.1 Constructor Concept

A **case constructor** creates a DU value by reserving storage and initializing the tag and payload. It returns the value, a `memref<Exi8>` view of the block (§3.1):

```fsharp
// Conceptual constructor signatures for Number DU (region-bounded form)
Number.IntVal.construct   : arena * int -> memref<Exi8>
Number.FloatVal.construct : arena * float -> memref<Exi8>
```

The `arena` operand shown is the **region-bounded form** of the constructor. The §3.1 lifetime lattice settles each construction site's class at saturation, and the site has exactly one form: the region-bounded form takes the covering arena; the scope-bounded and program-lifetime forms take no arena operand, because the block is a stack slot or a `memref.global` (§5.2). The signatures of this chapter show the region-bounded form throughout; the other forms are the same signature with the `arena` operand removed.

### 5.2 Constructor Semantics

A constructor:
1. Computes storage size for the case
2. Reserves the storage in the class the §3.1 lifetime lattice selected for the value (a stack slot, a region/arena, program-lifetime space, or the heap); an arena is threaded in only when the value classifies as region-bounded (§5.1)
3. Stores the tag value
4. Stores the payload with case-specific type: an inline value, a buffer view, an environment view, or an `index` link (§3.4)
5. Returns the value, a `memref<Exi8>` view of the block

When the block is placed in an arena, the arena-relative offset at which it is reserved is the value's `index` link within that arena. A recursive-case or collection payload stores that `index` (§3.4, §4.5): `Expr.Add.construct` takes the two child links as `index` operands, each carrying VC-LINK over the arena that holds the tree, and both children are placed in that same arena (§8.3).

### 5.3 Constructor Generation

The portable body reserves storage for the case and writes the tag and payload fields under the case layout. The reserved size is the case's computed `storage_size` (§2.2), not a fixed constant; for `IntVal` that is `1` tag byte plus a 4-byte `i32` payload, rounded to the case alignment:

```
func.func @Number.IntVal.construct(%arena: memref<?xi8>, %val: i32) -> memref<Exi8> {
    // Reserve storage sized for the IntVal case (tag + i32, aligned) in the arena
    %du = du.alloc %arena, case: IntVal : memref<Exi8>

    // Write the tag and payload fields under the IntVal layout
    du.store_tag   %du, 0 : i8            // tag = 0
    du.store_field %du, case: IntVal, field: 1, %val : i32

    return %du : memref<Exi8>
}
```

For a scope-bounded site `du.alloc` stands for a `memref.alloca` and the `%arena` operand is absent; for a program-lifetime site the block is a `memref.global` in the platform's declared program-lifetime space and the constructor initialises it once (§3.1). On the LLVM pathway the field writes lower to `llvm.insertvalue` into `!llvm.struct<(i8, i32)>` followed by an `llvm.store` through the `!llvm.ptr` the view became (the mirror of §10.2.1); the `insertvalue` form belongs to that pathway.

---

## 6. Pattern Matching Compilation

### 6.1 Match Expression Decomposition

A `match` expression over a DU compiles to:
1. Tag extraction
2. Tag dispatch (`scf.index_switch` over the literal case set)
3. Case eliminator calls in each branch

```fsharp
match number with
| IntVal i -> formatInt i
| FloatVal f -> formatFloat f
```

Compiles to the portable form, over `func`, `arith`, `scf`, and `memref` only:

```mlir
func.func @formatNumber(%du: memref<Exi8>) -> memref<?xi8> {
    %tag  = func.call @Number.tag(%du) : (memref<Exi8>) -> i8
    %case = arith.index_cast %tag : i8 to index
    %result = scf.index_switch %case -> memref<?xi8>
    case 0 {
        %i = func.call @Number.IntVal.eliminate(%du) : (memref<Exi8>) -> i32
        %r = func.call @formatInt(%i) : (i32) -> memref<?xi8>
        scf.yield %r : memref<?xi8>
    }
    default {
        // the last case of an exhaustive match occupies the default region scf.index_switch requires
        %f = func.call @Number.FloatVal.eliminate(%du) : (memref<Exi8>) -> f64
        %r = func.call @formatFloat(%f) : (f64) -> memref<?xi8>
        scf.yield %r : memref<?xi8>
    }
    return %result : memref<?xi8>
}
```

No block-based control flow (`cf.switch`, `cf.br`) appears above the witness boundary; the pathway's standard `scf` lowering produces the blocks ([Seq Representation §5.2](seq-representation.md)).

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
serialize_Number(writer, du):            // du : memref<Exi8>, the DU value (§3.1)
    tag = Number.tag(du)
    write_u8(writer, tag)
    switch tag:
        case 0:
            val = Number.IntVal.eliminate(du)
            write_i32(writer, val)
        case 1:
            val = Number.FloatVal.eliminate(du)
            write_f64(writer, val)
```

### 7.2 Deserialization

```
deserialize_Number(reader, arena) -> memref<Exi8>:    // the DU value; region-bounded form of §5.1
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
// is placed in the platform's declared immutable program-lifetime space
// (rodata on an ELF target, flash on an MCU), never freed; no arena involved
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

A closure-valued payload adds a constraint of its own. Constructing a DU value with a closure payload SHALL be an escape point for that closure ([Closure Representation §3.3](closure-representation.md)): the block stores the closure's environment view, not a copy of the environment (§3.4, §9.1), so the closure environment's lifetime class SHALL dominate the storage class of every DU block that holds a view of it. Escape analysis SHALL place the environment in a class whose lifetime covers each such block; a construction for which no covering class exists SHALL be rejected at compile time.

### 8.3 Recursive Structure Placement

For recursive DUs, all nodes in a tree share one arena, and so one storage class: the region that covers the tree when it is region-bounded, or a static-backed arena in the platform's declared program-lifetime space when the whole tree is program-lifetime ([Memory Regions](memory-regions.md), arena backing). Each node links its children by `index` into that shared arena (§3.4, §4.5), so VC-LINK for every link in the tree ranges over one extent.

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

The `Deferred` case stores the closure as the `(fn, env)` pair of [Closure Representation §6.3](closure-representation.md). The environment is the payload field; the code component is not a field, because a function value is never data in the interior ([Closure Representation §2.1](closure-representation.md)): it is fixed by the closure form at saturation ([Backend Lowering Architecture §4.2](backend-lowering-architecture.md)):

```
Deferred layout: { tag: i8, env: memref<Exi8> }      E = the environment extent, a literal at saturation
Deferred payload = (fn, env)
  fn:  func.constant @lifted : (memref<Exi8>) -> 'T   fixed by the closure form; no value at all when the callee is known at saturation
  env: the field above, a view of the environment, not a copy of it
```

The pair is never packed into one value and never stored as an address. The eliminator returns the pair, which the caller invokes as `func.call_indirect %fn(%env)`, or as a direct `func.call @lifted(%env)` where the callee is known at saturation. On the LLVM pathway `func.constant` becomes `llvm.mlir.addressof` through the standard lowering; that form belongs to the pathway.

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

The `Success` eliminator returns the `index` link of the nested `Result` block (§3.4); the caller views the arena at that offset (§4.5) and eliminates the nested value further.

---

## 10. MLIR Representation

### 10.1 DU Type in MLIR

At the portable middle-end level a DU value is a `memref<Exi8>` view of its storage block, `E` the type's `storage_size` of §2.2. A link from one block to an arena-resident block is a platform-sized `index` (the arena-relative offset of §3.4; the pointer-shaped NTU kind `NTUptr` maps to `index`, not to `!llvm.ptr`, [NTU Types §8.1](ntu-types.md)). The middle end commits to no target:

```mlir
// DU value as a view of its storage block (portable)
%du : memref<Exi8>

// link to an arena-resident block (portable)
%link : index
```

The `!llvm.ptr` form appears only after a target pathway (the LLVM/CPU/MCU pathway) has been selected; it is a lowering of the portable form, not what the middle end emits. See §10.2.1 for the LLVM-pathway realization.

### 10.2 Generated Functions

For each DU type `T` with cases `C₁, ..., Cₙ`, the middle end emits `func.func` declarations over portable types (the DU value as `memref<Exi8>`, a link as `index`, the arena as `memref<?xi8>`). The constructors are shown in their region-bounded form; the scope-bounded and program-lifetime forms omit the `%arena` operand (§5.1):

```mlir
// Tag accessor
func.func @T.tag(%du: memref<Exi8>) -> TagType

// Per-case eliminators
func.func @T.C₁.eliminate(%du: memref<Exi8>) -> PayloadType₁
func.func @T.C₂.eliminate(%du: memref<Exi8>) -> PayloadType₂
...

// Per-case constructors (region-bounded form)
func.func @T.C₁.construct(%arena: memref<?xi8>, %payload: PayloadType₁) -> memref<Exi8>
func.func @T.C₂.construct(%arena: memref<?xi8>, %payload: PayloadType₂) -> memref<Exi8>
...

// BAREWire serialization (if enabled)
func.func @T.serialize(%writer: memref<?xi8>, %du: memref<Exi8>) -> i32
func.func @T.deserialize(%reader: memref<?xi8>, %arena: memref<?xi8>) -> memref<Exi8>
```

A `PayloadTypeᵢ` is the payload's portable type: an inline primitive, a buffer view `memref<?xT>`, an environment view `memref<Exi8>`, or an `index` link (§3.4).

### 10.2.1 LLVM-Pathway Realization

After the LLVM target pathway is selected (CPU/MCU targets), the portable `memref<Exi8>` view lowers to `!llvm.ptr` and an `index` link to a machine address through the pathway's standard `memref` and `index` lowerings, and the eliminator body realizes the case-specific interpretation with LLVM ops:

```mlir
// LLVM pathway only, after target commitment
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
// FloatVal 3.14 construction, LLVM pathway
%f = arith.constant 3.14 : f64
%bits = llvm.bitcast %f : f64 to i64        // ← reinterpret into slot type
%result = llvm.insertvalue %bits, %struct[1] : !llvm.struct<(i8, i64)>
```

**Elimination:** Symmetric. Read the slot field, then reinterpret the raw bits as the payload type. On the LLVM pathway:

```mlir
// FloatVal elimination, LLVM pathway
%bits = llvm.extractvalue %struct[1] : i64   // ← raw slot bits
%f = llvm.bitcast %bits : i64 to f64        // ← interpret as float
 
```

**SSA Cost Implication:** The SSA count for `DUConstruct` depends on whether bitcast is needed - this is computed deterministically from the type comparison at SSA assignment time.

---

## 12. Normative Requirements

1. **View Representation**: A DU value SHALL be a `memref` view of its storage block, whose extent is settled at saturation (§3.1). The block SHALL be placed by the §3.1 lifetime lattice in the storage class whose lifetime covers the value (stack, region, program-lifetime space, or heap). A payload that is itself arena-resident (a nested DU, a recursive case, a collection) SHALL be linked by an `index` satisfying VC-LINK (§3.4), never by an address; no DU value and no link SHALL be null or tested for absence.

2. **Type-Safe Elimination**: Each case SHALL have a dedicated eliminator that returns the payload with its declared type.

3. **No Type Coercion in Eliminators**: Eliminators SHALL NOT perform type conversion; they interpret memory with the case-specific type.

4. **Tag Integrity**: The tag value SHALL be set by constructors and read by eliminators; direct tag manipulation is undefined behavior.

5. **Lattice-Driven Placement**: DU construction SHALL place the storage block in the storage class the §3.1 lifetime lattice selects for the value (stack when scope-bounded, a region when region-bounded, static storage when program-lifetime, the heap only when genuinely dynamic). An arena/region SHALL be threaded into construction only for a region-bounded value; it SHALL NOT be a precondition of construction for the other classes. On a target without a heap, a DU value that classifies as dynamic SHALL be a compile-time lifetime error, not a heap allocation.

6. **Lifetime Safety**: A DU value SHALL NOT outlive the storage class that holds it. A value whose lifetime is statically known to be the whole program is placed in static storage and MAY be returned and held; a value that escapes into a lifetime no covering class can satisfy SHALL be rejected at compile time.

7. **Monomorphization**: Generic DUs SHALL be fully monomorphized; no runtime type parameters.

8. **BAREWire Compatibility**: DU serialization SHALL follow the BAREWire union contract (§7.0). The wire tag SHALL be the compile-time case index, and endpoints SHALL derive read/write code from the same union type so that conformance holds by construction and non-conforming messages are rejected at the boundary. BAREWire SHALL NOT be conflated with the external BARE encoding it builds on.

9. **Recursive Support**: DUs MAY contain cases with payloads of the same or other DU types.

10. **Inlining Permitted**: Implementations MAY inline eliminators/constructors for optimization while preserving semantics.

11. **JSIR-Pathway Realization**: On the JSIR pathway a union SHALL be realized per §10.2.2. Requirements 1, 5, and 12 are requirements on layout-realizing pathways and bind pathways that realize memory layouts; the remaining requirements bind on every pathway, with placement discharged by the host garbage collector on this pathway.

12. **Closure Payloads**: A closure payload SHALL be stored as the `(fn, env)` pair of [Closure Representation §6.3](closure-representation.md): the environment view is the payload field and the code component is fixed by the closure form at saturation (§9.1). The pair SHALL NOT be packed into one value and SHALL NOT be stored as an address.

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
- `Pair` is recursive (contains two `index` links to `Value` blocks in the same arena)
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
