---
title: "Clef Type Universe Specification"
weight: 120
category: Language
status: normative
---

> **Status**: Draft
> **Phase**: A (Type Universe) - Part of Clef principled type system redesign
> **Last Updated**: 2024-12-30

## Overview

This document specifies the native type universe for Clef, the F# native compiler. The design follows ML/OCaml foundations while preserving familiar F# developer experience.

### Core Principles

1. **Familiar Design-Time Experience**: Use F# type names (`string`, `option`, `int`), not foreign alternatives
2. **Absolute Null-Freedom**: Everything is `voption<'T>` - no null values anywhere
3. **Null-Freedom Cascades Through APIs**: Magic return values (`-1`, null, a throw) become `voption` returns
4. **Leverage Existing F# Machinery**: Reuse FSharp.Core where possible
5. **Spec Before Scaffold**: Document before implementing

---

## Part 1: Foundational Principles

### 1.1 Type Universe Axioms

The native type universe is built on three primitive concepts from ML/OCaml:

1. **Products** (tuples, records) - Combine multiple values into one
2. **Sums** (discriminated unions) - Choose between alternatives
3. **Functions** - Transform values

Everything else is derived from these primitives.

### 1.2 Memory Guarantees

- **No implicit boxing**: Value types remain on stack unless explicitly requested
- **Deterministic layout**: Memory representation is predictable and specified
- **No null**: All types are non-nullable by construction

### 1.3 Type Equivalences

| User Writes | Underlying Implementation | Notes |
|-------------|--------------------------|-------|
| `option<'T>` | `voption<'T>` | Stack-allocated, non-null |
| `string` | UTF-8 `memref<?xi8>` | The buffer is the value; the byte length is the memref dimension |
| `int` | Platform word | `nativeint` semantics |

---

## Part 2: Primitive Types

> **CCS Resolution**: See the CCS specification for how CCS resolves these types at compile-time.
>
> **OCaml Provenance**: Primitives follow OCaml's value-oriented representation (unboxed by default) while eliminating GC-oriented overhead. See [Appendix E](#appendix-e-ocaml-provenance-and-fidelity-extensions) for detailed provenance analysis.

### 2.1 Unit

```fsharp
type unit = ()
```

| Property | Value | Notes |
|----------|-------|-------|
| **Memory** | Zero-sized (ZST) | No runtime representation |
| **Alignment** | 1 byte | Trivially aligned |
| **MLIR** | (elided) | Not materialized in generated code |

**OCaml Provenance**: Identical to OCaml's `unit` - the canonical "no information" type. In OCaml, `()` shares representation with `[]` (empty list) as the integer 0. In Clef, `unit` is truly zero-sized - not even allocated.

**Usage**:
```fsharp
let doSideEffect () : unit = Console.WriteLine "Hello"
let ignoredResult = ignore 42  // : unit
 
```

### 2.2 Boolean

```fsharp
type bool = true | false
```

| Property | Value | Notes |
|----------|-------|-------|
| **Memory** | 1 byte | Not 1 bit - for alignment |
| **Representation** | `i8` | `false` = 0, `true` = 1 |
| **Alignment** | 1 byte | Natural alignment |
| **MLIR** | `i1` or `i8` | Context-dependent |

**OCaml Provenance**: OCaml represents booleans as unboxed integers (`false` = 0, `true` = 1). Clef preserves this representation but uses a full byte for alignment efficiency in arrays and structs.

**Design Decision**: Booleans occupy 1 byte rather than 1 bit because:
1. Sub-byte addressing is inefficient on modern hardware
2. Array indexing requires byte-addressable elements
3. Cache line efficiency favors byte alignment

**No Boolean Coercion**:
```fsharp
// ILLEGAL in Clef - no implicit int-to-bool
let x = if 1 then "yes" else "no"  // Error: expected bool, got int

// LEGAL - explicit comparison
let x = if 1 <> 0 then "yes" else "no"
```

### 2.3 Integer Family

| Type | F# Name | Size | Signed | MLIR Type | OCaml Equivalent |
|------|---------|------|--------|-----------|------------------|
| Platform int | `int` | word | Yes | `index` | `int` (but 63-bit) |
| Platform uint | `uint` | word | No | `index` | N/A |
| 8-bit | `int8`, `uint8` | 1 byte | Yes/No | `i8` | N/A |
| 16-bit | `int16`, `uint16` | 2 bytes | Yes/No | `i16` | N/A |
| 32-bit | `int32`, `uint32` | 4 bytes | Yes/No | `i32` | `int32` |
| 64-bit | `int64`, `uint64` | 8 bytes | Yes/No | `i64` | `int64` |
| Native | `nativeint`, `unativeint` | word | Yes/No | `index` | `nativeint` |

**Design Decisions**:

1. **`int` = platform word**: Follows native compilation conventions (Rust, C), not F#'s 32-bit default. Array indexing and arena-relative `index` arithmetic use the platform word naturally. This aligns with Rust's `isize`/`usize` philosophy.

2. **No GC tagging overhead**: Unlike OCaml's 63-bit tagged integers (which reserve 1 bit for runtime GC discrimination), Clef integers use full precision. Compile-time type safety eliminates the need for runtime type tags.

   | System | `int` precision | Tag overhead |
   |--------|-----------------|--------------|
   | OCaml (64-bit) | 63 bits | 1 bit for GC |
   | F# (.NET) | 32 bits | None (boxed separately) |
   | **Clef** | **64 bits (full word)** | **None** |

3. **`int` and `nativeint` are synonyms**: Both map to MLIR `index` type. This differs from F# where `int` is always 32-bit.

4. **Fixed-width types for interop**: `int32`, `int64` provide explicit sizing for FFI and serialization. These match OCaml's `Int32.t` and `Int64.t`.

**Alignment and Cache Considerations**:

| Type | Alignment | Cache Line Fit (64 bytes) |
|------|-----------|---------------------------|
| `int8` | 1 byte | 64 values |
| `int16` | 2 bytes | 32 values |
| `int32` | 4 bytes | 16 values |
| `int64` | 8 bytes | 8 values |
| `int` (64-bit) | 8 bytes | 8 values |

For cache-critical code, arrays of smaller integer types pack more values per cache line.

**Overflow Behavior**:

| Context | Behavior |
|---------|----------|
| Default | Wrapping (two's complement) |
| `Checked` module | Returns `voption` on overflow |
| Debug builds | Optional overflow traps |

```fsharp
// Default: wrapping arithmetic
let wrapped = System.Int32.MaxValue + 1  // Wraps to MinValue

// Checked: explicit overflow handling
let checked = Checked.add System.Int32.MaxValue 1  // voption.None
 
```

> **JSIR pathway** (JavaScript Substrate profile): the host number model is IEEE-754 binary64 with exact integers to 2⁵³. Integer realization on this pathway, including the wide-integer mechanism for widths above 53 bits and the preservation of fixed-width wrapping semantics, is specified in [Width Inference §8](width-inference.md). The full-word figures in this section are properties of layout-realizing pathways.

### 2.4 Floating Point Family

| Type | F# Name | Size | MLIR Type | IEEE 754 |
|------|---------|------|-----------|----------|
| Single | `float32`, `single` | 4 bytes | `f32` | binary32 |
| Double | `float`, `double` | 8 bytes | `f64` | binary64 |

`float` and `float32` are the **bare, no-locality-claim** real types: a value with no analyzed range whose representation is not being selected. Their `f64`/`f32` MLIR mappings are IEEE-754 **lowering targets**, chosen for a wide or unobservable range on a target whose binding offers an FPU, not a universal declared representation. IEEE-754 is a legacy hardware fixture (the FPU of a CPU, GPU, or APU), so it is one lowering target among several, not the default a real is presumed to be.

For a **dimensioned or ranged** real, the representation — IEEE-754, posit, or fixed-point — is *selected* from the value's dimensional range by [Numeric Selection](numeric-selection.md#6-the-default-and-unobservable-case) (§6.1), never fixed by the type name. `float`/`float32` are what selection lowers to when the range is wide or unobservable; the `float = f64` / `float32 = f32` mappings remain correct as those lowering targets. Representation follows from analyzed range, not from a target type name.

**Design Decisions**:

1. **`float`/`float32` carry no representation claim**: they are the bare types that lower to `f64`/`f32` when a range is wide or unobservable. They are not "the default representation" for a real; a dimensioned or ranged real has its representation selected per [Numeric Selection](numeric-selection.md) (§6.1), which may select a posit or a fixed-point number instead. Explicit developer selection of a precision and range is sovereign over inference.

2. **IEEE 754 compliance when IEEE is the selected representation**: where the selected or bare representation is `f64`/`f32`, all floating-point operations follow IEEE 754 semantics including NaN propagation, infinities, and signed zeros.

3. **No implicit float-int conversion**:
   ```fsharp
   let x : float = 42    // Error: expected float, got int
   let x : float = 42.0  // OK
   let x : float = float 42  // OK - explicit conversion
 
   ```

**Alignment**:

| Type | Alignment | Notes |
|------|-----------|-------|
| `float32` | 4 bytes | Natural alignment |
| `float64` | 8 bytes | Natural alignment |

**Special Values**:

```fsharp
let inf = infinity        // Positive infinity
let ninf = -infinity      // Negative infinity
let nan = nan             // Not a Number
let isNan x = x <> x      // NaN property: NaN ≠ NaN
 
```

### 2.5 Character

```fsharp
type char = (* Unicode scalar value *)
```

| Property | Value | Notes |
|----------|-------|-------|
| **Memory** | 4 bytes | Full Unicode scalar value |
| **Representation** | `i32` | UTF-32 codepoint (U+0000 to U+10FFFF, excluding surrogates) |
| **Alignment** | 4 bytes | Natural alignment |
| **MLIR** | `i32` | Direct mapping |

**Design Decision (RESOLVED)**: Characters are UTF-32 codepoints (4 bytes), not UTF-16 code units.

**Rationale**:
1. **String encoding is UTF-8**: Clef strings are UTF-8 `memref<?xi8>` views (see Part 4.1)
2. **Iteration yields codepoints**: When iterating over a UTF-8 string, each `char` is a decoded Unicode scalar value
3. **No surrogate pairs**: Unlike UTF-16, a single `char` always represents a complete character
4. **Consistency with Rust**: Rust's `char` is also a 32-bit Unicode scalar value

**OCaml Divergence**: OCaml's `char` is a single byte (Latin-1 only, 0-255). Clef explicitly supports full Unicode.

**String-Character Interaction**:

```fsharp
let s = "Hello, 世界!"  // UTF-8 string

// Iteration yields Unicode codepoints
for c in String.chars s do
    printfn "U+%04X" (int c)  // U+0048, U+0065, ..., U+4E16, U+754C, U+0021

// Indexing by codepoint (not byte offset)
let c = String.charAt 7 s  // voption.Some '世' (U+4E16)

// Byte length vs character length
String.byteLength s   // 15 bytes (UTF-8 encoded)
String.charLength s   // 10 characters
 
```

**Invalid Codepoints**: Surrogate codepoints (U+D800 to U+DFFF) are not valid `char` values. Attempting to construct such values results in a compile-time or runtime error.

---

## Part 3: Structural Types

> **Foundation**: All composite types are built from products (tuples, records) and sums (discriminated unions). This follows ML/OCaml tradition.
>
> **OCaml Provenance**: OCaml's structural assembly semantics are preserved - products are contiguous, sums are tagged. However, OCaml's runtime block headers (for GC) are eliminated. See [Appendix E](#appendix-e-ocaml-provenance-and-fidelity-extensions).

### 3.1 Tuples (Anonymous Products)

```fsharp
let pair : int * string = (42, "hello")
let triple : int * string * float = (1, "x", 3.14)
```

**Memory Layout**:
```
Tuple: int * string (64-bit platform)
┌─────────────┬─────────────────────────────────────┐
│ int (word)  │ string (memref<?xi8> view: 2 words) │
└─────────────┴─────────────────────────────────────┘
     8 bytes                 16 bytes                 = 24 bytes total
```

| Property | Value |
|----------|-------|
| **Structural typing** | `int * string` ≡ `int * string` regardless of context |
| **Allocation** | Chosen by lifetime class: stack when scope-bounded; arena when region-bounded; static when program-lifetime (the platform's declared program-lifetime space, cited by name from the platform description: rodata/data on an ELF target, flash/SRAM on an MCU, constant memory on a GPU, initialised BRAM on an FPGA); heap only where one exists and the lifetime is genuinely dynamic. Escaping the defining scope does not by itself imply heap or arena; a program-lifetime escape goes to static storage. See [Closure Representation §3.3](closure-representation.md#33-escape-analysis). |
| **Alignment** | Natural alignment (largest field alignment) |
| **Nesting** | `(a * b) * c` ≠ `a * (b * c)` (different memory layouts) |
| **MLIR** | `tuple<index, memref<?xi8>>` |

**OCaml Comparison**:

| Aspect | OCaml | Clef |
|--------|-------|----------|
| **Header** | 8-byte block header (GC info) | **None** |
| **Fields** | Word-sized slots | Naturally aligned |
| **Boxing** | Floats often boxed | Never boxed |

**Design Decision**: Tuples are **unboxed** - no heap allocation, no indirection, no headers. Fields are laid out contiguously with natural alignment.

**Padding and Alignment Rules**:

```fsharp
// This tuple has internal padding for alignment
let t : int8 * int64 * int8 = (1y, 100L, 2y)
```

```
Memory Layout (with padding):
┌────────┬─────────┬──────────┬────────┬─────────┐
│ int8   │ padding │ int64    │ int8   │ padding │
└────────┴─────────┴──────────┴────────┴─────────┘
  1 byte   7 bytes   8 bytes    1 byte   7 bytes  = 24 bytes
```

**Cache Considerations**:

| Tuple Size | Cache Lines (64 bytes) | Notes |
|------------|------------------------|-------|
| ≤ 64 bytes | 1 | Single cache line - optimal |
| 65-128 bytes | 2 | May span two lines |
| > 128 bytes | 3+ | Consider record with explicit layout |

### 3.2 Records (Named Products)

```fsharp
type Person = { Name: string; Age: int }
type Point = { X: float; Y: float }
```

**Memory Layout**:
```
Record: Person (64-bit platform)
┌─────────────────────────────┬─────────────┐
│ Name: string (2 words)      │ Age: int    │
└─────────────────────────────┴─────────────┘
           16 bytes               8 bytes    = 24 bytes total
```

| Property | Value |
|----------|-------|
| **Nominal typing** | `Person` ≠ `{ Name: string; Age: int }` (different types) |
| **Field order** | Declaration order determines memory layout |
| **Allocation** | Chosen by lifetime class: stack when scope-bounded; arena when region-bounded; static when program-lifetime (the platform's declared program-lifetime space, cited by name from the platform description: rodata/data on an ELF target, flash/SRAM on an MCU, constant memory on a GPU, initialised BRAM on an FPGA); heap only where one exists and the lifetime is genuinely dynamic. Escaping the defining scope does not by itself imply heap or arena; a program-lifetime escape goes to static storage. See [Closure Representation §3.3](closure-representation.md#33-escape-analysis). |
| **Alignment** | Natural alignment per field, struct alignment = max field alignment |
| **MLIR** | `memref<Exi8>` — record layout settled at saturation, fields at literal offsets (`name` a `memref<?xi8>` view, `age` an `index`) |

**OCaml Comparison**:

| Aspect | OCaml | Clef |
|--------|-------|----------|
| **Header** | 8-byte block header | **None** |
| **Field access** | Offset from header | Direct offset |
| **Field order** | Declaration order | Declaration order |

**Mutable Fields**:

```fsharp
type MutablePoint = { mutable X: float; mutable Y: float }
```

| Property | Mutable | Immutable |
|----------|---------|-----------|
| Memory layout | Identical | Identical |
| Compile-time | Allows `<-` assignment | Disallows `<-` |
| Thread safety | Requires synchronization | Naturally thread-safe |

**Cache-Aware Record Design**:

For high-performance scenarios, consider field ordering for cache efficiency:

```fsharp
// HOT fields together - accessed frequently
type OptimizedActor = {
    // Hot path - first cache line (64 bytes)
    MessageCount: int64          // 8 bytes, offset 0
    LastProcessed: int64         // 8 bytes, offset 8
    State: ActorState            // 16 bytes, offset 16
    mutable Flags: uint32        // 4 bytes, offset 32
    // padding: 28 bytes to fill cache line
    
    // Cold path - second cache line
    Name: string                 // 16 bytes, offset 64
    CreatedAt: int64             // 8 bytes, offset 80
    Config: ActorConfig          // varies
}
```

**False Sharing Prevention**:

When mutable fields are accessed from multiple threads, place them on separate cache lines:

```fsharp
[<Struct>]
type ThreadCounters = {
    [<CacheLineAligned>]  // Attribute hint to compiler
    mutable Counter1: int64
    
    [<CacheLineAligned>]  // Separate cache line
    mutable Counter2: int64
}
```

### 3.3 Discriminated Unions (Named Sums)

```fsharp
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Point  // No payload
 
```

**Memory Layout**:
```
Union: Shape (64-bit platform)
┌──────────┬─────────┬────────────────────────────────────┐
│ Tag (i8) │ padding │ Payload (size of largest variant)  │
└──────────┴─────────┴────────────────────────────────────┘
   1 byte    7 bytes              16 bytes                 = 24 bytes total
```

| Property | Value |
|----------|-------|
| **Tag size** | `i8` for ≤256 variants, `i16` for ≤65536 |
| **Payload size** | Size of largest variant (all variants same size) |
| **Payload alignment** | Max alignment of any variant's fields |
| **Total alignment** | Max(tag alignment, payload alignment) |
| **MLIR** | `memref<Exi8>` — tag at `[0]`, payload at its literal offset ([Discriminated Union Representation](discriminated-union-representation.md)) |

**OCaml Comparison**:

| Aspect | OCaml | Clef |
|--------|-------|----------|
| **No-arg constructors** | Unboxed integer (0, 1, 2...) | Tag byte only |
| **With-arg constructors** | Block with tag byte in header | Tag + payload |
| **Tag range** | 0-245 (246-255 reserved) | 0-255 (full `i8`) |
| **Block header** | 8 bytes (includes tag) | **None** |

**Tag Assignment** (declaration order):

```fsharp
type Color = Red | Green | Blue
// Red = 0, Green = 1, Blue = 2

type Result<'T, 'E> = Ok of 'T | Error of 'E
// Ok = 0, Error = 1
 
```

**Variant-Specific Layouts**:

```fsharp
type Message =
    | Ping                              // Tag only: 1 byte + padding
    | Data of payload: array<byte>      // Tag + memref<?xi8> view: 24 bytes on x86-64 (1 tag + 7 pad + 2 platform words); 12 bytes on thumbv8m
    | Error of code: int * msg: string  // Tag + int + string: 1 + 7 + 8 + 16 = 32 bytes
 
```

```
Union: Message (largest variant determines size)
┌──────────┬─────────┬────────────────────────────────────┐
│ Tag (i8) │ padding │ 32 bytes (Error variant size)      │
└──────────┴─────────┴────────────────────────────────────┘
= 40 bytes total (all variants padded to this size)
```

**Single-Case Unions** (newtypes):

```fsharp
type UserId = UserId of int
type Email = Email of string
```

| Property | Value |
|----------|-------|
| **Optimization** | Tag elided - same representation as wrapped type |
| **Runtime size** | `sizeof(int)` for `UserId`, `sizeof(string)` for `Email` |
| **Type safety** | Compile-time distinction, zero runtime overhead |

```
UserId = int (no wrapping, no tag)
┌─────────────┐
│ int (word)  │
└─────────────┘
    8 bytes
```

**Recursive Unions**:

```fsharp
type List<'T> = Nil | Cons of head: 'T * tail: List<'T>
type Tree<'T> = Leaf of 'T | Node of left: Tree<'T> * value: 'T * right: Tree<'T>
```

| Property | Value |
|----------|-------|
| **Recursion** | The recursive field is an `index` (arena link): an arena-relative offset into the arena buffer the value lives in. Every link word carries VC-LINK, `0 <= i < extent(arena)` or `i` = the sentinel's index, quantifier-free (QF_LIA over literals), discharged at saturation before witnessing |
| **Nil (nullary case)** | The zero-slot sentinel: exactly one program-lifetime, immutable node per element type, tag `Nil`, links indexing itself, payload zero-initialised static data that no operation reads. It resides in the platform's declared immutable program-lifetime space, cited by name from the platform description (a `Resides` edge): rodata on an ELF target, flash on an MCU, constant memory on a GPU, initialised BRAM on an FPGA. Never a null; the emptiness test is the literal comparison `tag = Nil` |
| **Cons/Node/Leaf** | Ordinary tagged node, a flat aggregate with a settled layout, placed in an arena by the lifetime lattice of [Closure Representation §3.3](closure-representation.md#33-escape-analysis) |

```
Cons cell: List<int>
┌──────────┬─────────┬──────────┬──────────────────────────┐
│ Tag (i8) │ padding │ head: T  │ tail: index (arena link) │
└──────────┴─────────┴──────────┴──────────────────────────┘
   1 byte    7 bytes   8 bytes            8 bytes           = 24 bytes
```

> **Layout obligations**: VC-LINK on every link word, VC-GUARD (the discriminant test, `tag = Cons` or `height ≠ 0`, dominates every read of a payload slot on the saturated graph), and the sentinel's residence and immutability are quantifier-free at saturation over the graph's literals: QF_LIA plus a dominance check, no quantifiers. The AVL balance invariant of [§5.4](#54-map) and [§5.5](#55-set) and the list algebra of [§5.3](#53-list) are schema lemmas proven once per recipe shape, never a per-program fixpoint. Every link is always a valid node, so no algorithm over a recursive union has an absence branch: recursion terminates at the sentinel, and a link load is one `memref.load` of an `index`.

**Pattern Matching Compilation**:

Pattern matching on DUs compiles to efficient branch tables:

```fsharp
match shape with
| Circle r -> computeCircleArea r
| Rectangle (w, h) -> w * h
| Point -> 0.0
```

Compiles to:
```
switch (shape.tag) {
    case 0: goto circle_branch;   // Circle
    case 1: goto rect_branch;     // Rectangle
    case 2: goto point_branch;    // Point
}
```

**Exhaustiveness**: Pattern matching is verified exhaustive at compile time. Missing cases produce warnings/errors.

---

## Part 4: Reference Types

> **Principle**: Reference types are `memref` views: the buffer is the value and its length is the view's dimension. No null; empty is a view of dimension 0.
>
> **CCS Resolution**: See the CCS specification for compiler-level type resolution.

### 4.1 String

```fsharp
let greeting : string = "Hello, World!"
let empty : string = ""
```

**Memory Layout** (`memref<?xi8>`, UTF-8):
```
string = memref<?xi8>
┌────┬────┬────┬─────┬──────┐
│ b0 │ b1 │ b2 │  …  │ bn-1 │   the UTF-8 bytes: the buffer is the value
└────┴────┴────┴─────┴──────┘
  dimension n = byte length, carried by the view; there is no separate length field
```

> **Platform word**: Inside an aggregate (a tuple slot, a record field, a union payload) the view occupies two platform words, its buffer and its dimension: 16 bytes on x86-64 (8-byte word), 8 bytes on thumbv8m/M33 (4-byte word). The bytes themselves are placed by the lifetime lattice of [Closure Representation §3.3](closure-representation.md#33-escape-analysis); a string literal is program-lifetime and immutable, so it resides in the platform's declared immutable program-lifetime space (rodata on an ELF target, flash on an MCU).

| Property | Value |
|----------|-------|
| **Encoding** | UTF-8 (NOT UTF-16) |
| **Length semantics** | Byte count (the memref dimension), not character count |
| **Empty string** | A view of dimension 0 over a valid buffer; never a null. `String.isEmpty` is the literal comparison `memref.dim = 0` |
| **MLIR** | `memref<?xi8>` |

**Why UTF-8?**
1. **Native interop** - Rust, C, and most systems APIs use UTF-8
2. **Compact** - 1 byte per ASCII character
3. **Web/JSON native** - no transcoding overhead
4. **Embedded-friendly** - no UTF-16 surrogate handling

**Character Iteration** (codepoints, not bytes):
```fsharp
// Iterating over Unicode scalar values
for c in String.chars s do
    printfn "%c" c  // c : char (UTF-32 codepoint, 4 bytes)
 
```

**Zero-Copy Slicing**: A substring is a `memref.subview` of the same buffer with an adjusted offset and dimension; no bytes are copied.

> **JSIR pathway** (JavaScript Substrate profile): `string` is realized as a host string, whose internal encoding is UTF-16 code units. The observable semantics of this section bind unchanged: `String.byteLength` SHALL return the UTF-8 byte count, `String.chars` SHALL yield Unicode scalar values, and indexing SHALL be by codepoint. The `memref<?xi8>` layout and its cost figures are properties of layout-realizing pathways and do not bind on this pathway ([Backend Lowering Architecture §4.5](backend-lowering-architecture.md)).

> **See**: Appendix E for encoding comparison with OCaml (Latin-1) and .NET (UTF-16).

**API Changes (Null-Freedom Cascades)**:

| BCL Pattern | Clef Pattern | Rationale |
|-------------|------------------|-----------|
| `s.IndexOf(c)` → `-1` | `String.indexOf c s` → `voption<int>` | No magic return values |
| `s.Substring(i, len)` throws | `String.slice i len s` → `voption<string>` | No exceptions |
| `s.[i]` throws | `String.tryItem i s` → `voption<char>` | Bounds-safe |
| `s.Split(...)` → `string[]` | `String.split ... s` → `array<string>` | Native array |
| `String.IsNullOrEmpty(s)` | `String.isEmpty s` | No null possible |

### 4.2 Array

```fsharp
let numbers : array<int> = [| 1; 2; 3; 4; 5 |]
let empty : array<int> = [| |]
```

**Memory Layout** (`memref<?xT>`):
```
array<'T> = memref<?xT>
┌──────┬──────┬──────┬─────┬────────┐
│ e0   │ e1   │ e2   │  …  │ en-1   │   the elements: the buffer is the value, n * sizeof<'T> bytes, contiguous
└──────┴──────┴──────┴─────┴────────┘
  dimension n = length, carried by the view; there is no separate length field
```

> **Platform word**: Inside an aggregate the view occupies two platform words, its buffer and its dimension: 16 bytes on x86-64, 8 bytes on thumbv8m/M33 (4-byte word). The elements reside in the buffer, placed by the lifetime lattice of [Closure Representation §3.3](closure-representation.md#33-escape-analysis).

| Property | Value |
|----------|-------|
| **Element layout** | Contiguous, naturally aligned |
| **Bounds checking** | Always (no unsafe indexing by default): every access is guarded by `0 <= i < memref.dim` |
| **Empty array** | A view of dimension 0 over a valid buffer; never a null. `Array.isEmpty` is the literal comparison `memref.dim = 0` |
| **MLIR** | `memref<?xT>` |

**Monomorphized Layout**: Unlike uniform representations that box generic elements, Clef arrays are monomorphized - `array<int>` stores unboxed integers contiguously. Sequential access is cache-optimal (8 `int64` or 16 `int32` values per 64-byte cache line).

**Fixed Size After Creation**:
```fsharp
let arr = Array.create 10 0  // 10 elements, all 0
// arr.Length is immutable - no resizing
 
```

**API Changes (Null-Freedom Cascades)**:

| BCL Pattern | Clef Pattern |
|-------------|------------------|
| `arr.[i]` throws | `Array.tryItem i arr` → `voption<'T>` |
| `Array.find pred arr` throws | `Array.tryFind pred arr` → `voption<'T>` |
| `Array.head arr` throws | `Array.tryHead arr` → `voption<'T>` |

**Array Module Intrinsics (CCS Layer 1)**:

These operations are fundamental to the array type and are emitted directly by CCS. They cannot be expressed in pure F# because they require memory allocation and element size knowledge.

| Function | Signature | Description |
|----------|-----------|-------------|
| `Array.zeroCreate` | `int -> array<'T>` | Allocate n elements, zero-initialized |
| `Array.create` | `int -> 'T -> array<'T>` | Allocate n elements, all set to value |
| `Array.init` | `int -> (int -> 'T) -> array<'T>` | Allocate n elements, initialized by function |
| `Array.copy` | `array<'T> -> array<'T>` | Create a copy of the array |
| `Array.length` | `array<'T> -> int` | Return the length of the array (the memref dimension) |
| `Array.get` | `array<'T> -> int -> 'T` | Get element at index (bounds-checked) |
| `Array.set` | `array<'T> -> int -> 'T -> unit` | Set element at index (bounds-checked) |
| `Array.tryItem` | `int -> array<'T> -> voption<'T>` | Safe indexed access returning voption |
| `Array.isEmpty` | `array<'T> -> bool` | Return true if the dimension is zero |

**Allocation Semantics**: Arrays are allocated in the current memory region (stack arena or actor arena). The allocation strategy is determined by the memory region context, not by the Array function.

### 4.3 Span and ReadOnlySpan

```fsharp
let span : Span<int> = Span(arr, 2, 3)  // View into arr[2..4]
let roSpan : ReadOnlySpan<byte> = ReadOnlySpan(bytes)
```

**Memory Layout** (borrowed view):
```
Span<'T> = memref<?xT> view with a dynamic offset
┌──────────┬──────────┬─────┬────────────┐
│ e[off]   │ e[off+1] │  …  │ e[off+n-1] │   a window into the source buffer: nothing owned, nothing copied
└──────────┴──────────┴─────┴────────────┘
  offset off and dimension n carried by the view
```

| Property | Value |
|----------|-------|
| **Ownership** | Borrowed (does not own memory) |
| **Lifetime** | Must not outlive source |
| **Stack only** | Cannot be stored in heap structures |
| **MLIR** | `memref<?xT>` with a dynamic offset, a `memref.subview` of the source (no ownership) |

**Zero-Copy Views**: Spans provide borrowed views into arrays, strings, and memory regions without allocation. The stack-only constraint prevents lifetime escape (similar to Rust's slice borrowing). Maps directly to MLIR `memref` with dynamic offset.

> **Note**: Leverage `FSharp.Core` Span types - already stack-allocated and native-friendly.

---

## Part 5: Parameterized Types

> **Principle**: Parameterized types follow familiar F# syntax. CCS resolves native semantics at compile-time.
>
> **CCS Resolution**: See the CCS specification for option type resolution and null-free guarantees.

### 5.1 Option

```fsharp
let maybeValue : int option = Some 42
let nothing : string option = None
```

**Memory Layout** (stack-allocated tagged union):
```
option<'T>  (voption semantics)
┌──────────┬────────────────────┐
│ Tag (i8) │ Payload: 'T        │
└──────────┴────────────────────┘
   1 byte     sizeof<'T>         + padding
```

| Property | Value |
|----------|-------|
| **Tag values** | `None` = 0, `Some` = 1 |
| **Stack allocated** | Always (never heap) |
| **Null-freedom** | `None` is tag 0; never a null |
| **MLIR** | `memref<Exi8>` — `{tag, payload}`, stack-placed ([Option Operations](option-operations-representation.md)) |

**Stack-Only Guarantee**: Unlike heap-allocated options (cf. OCaml blocks, .NET reference types), Clef options are always stack-allocated with no GC involvement. This enables predictable memory layout for embedded targets and eliminates heap fragmentation from frequent option use.

> **JSIR pathway** (JavaScript Substrate profile): the stack layout above is a property of layout-realizing pathways. On the JSIR pathway, `option<'T>` is realized erased or reified under a proof discipline, specified in [Option Operations Representation §2.1](option-operations-representation.md).

> **See**: Appendix E for detailed OCaml/Rust comparison.

**CCS Resolution**:
```fsharp
// User writes familiar F# syntax:
let x : int option = Some 42
let y : int option = None

// CCS compiles with voption<int> semantics:
// - Stack allocated
// - Tag-based discrimination
// - No null anywhere
 
```

**API (Null-Freedom Cascades)**:

| BCL Pattern | Clef Pattern |
|-------------|------------------|
| `opt.Value` throws | `Option.get opt` (or pattern match) |
| `opt.IsSome` | `Option.isSome opt` |
| `Option.defaultValue v opt` | Same (works identically) |

> **Note**: CCS provides native `option` semantics directly - `option<'T>` maps to stack-allocated `voption` at compile time.

### 5.2 Result

```fsharp
let success : Result<int, string> = Ok 42
let failure : Result<int, string> = Error "not found"
```

**Memory Layout** (tagged union):
```
Result<'T, 'E>
┌──────────┬────────────────────────────────────┐
│ Tag (i8) │ Payload: max(sizeof<'T>, sizeof<'E>)│
└──────────┴────────────────────────────────────┘
```

| Property | Value |
|----------|-------|
| **Tag values** | `Ok` = 0, `Error` = 1 |
| **Stack allocated** | Always |
| **MLIR** | `memref<Exi8>` — `{tag, payload}` as a two-case union |

**Stack Allocation**: Like `option`, Result is always stack-allocated with zero heap overhead.

**Why Result Over Exceptions**: Result makes error handling explicit in type signatures, enables compile-time exhaustiveness checking, has zero runtime overhead, and can cross FFI boundaries via BAREWire. Exceptions are not supported for control flow in Clef.

**Preferred Error Handling**: Result is the idiomatic error handling pattern in Clef. Exceptions are not supported for control flow.

```fsharp
// Idiomatic error handling
let divide x y : Result<int, string> =
    if y = 0 then Error "division by zero"
    else Ok (x / y)

// Chaining with Result.map, Result.bind
divide 10 2
|> Result.map (fun x -> x * 2)
|> Result.defaultValue 0
```

### 5.3 List

```fsharp
let numbers : int list = [1; 2; 3; 4; 5]
let empty : int list = []
```

**Memory Layout** (cons cells):
```
list<'T>  (tagged node: Empty | Cons)
┌──────────┬─────────┬────────────────────┬──────────────────────────┐
│ tag: i8  │ padding │ head: 'T           │ tail: index (arena link) │
└──────────┴─────────┴────────────────────┴──────────────────────────┘
  1 byte   to align       sizeof<'T>             platform word
                                                (8 bytes x86-64,
                                                 4 bytes thumbv8m)
```

> **Platform word**: `tail` is an `index` (arena link), an arena-relative offset into the arena buffer the list lives in: one platform word, 8 bytes on x86-64 and 4 bytes on thumbv8m/M33. It carries VC-LINK, `0 <= i < extent(arena)`, with `i = 0` the sentinel at offset 0 of that arena, discharged at saturation before witnessing. A link load is one `memref.load` of an `index`.

| Property | Value |
|----------|-------|
| **Empty list** | Index `0`: the sentinel at offset 0 of every hosting arena, a copy of one program-lifetime, immutable sentinel image per element type, tag `Empty`, `tail` = 0, payload zero-initialised static data never read (VC-GUARD: `tag = Cons` dominates every read of `head` on the saturated graph). It resides in the platform's declared immutable program-lifetime space, cited by name from the platform description (a `Resides` edge): rodata on an ELF target, flash on an MCU, constant memory on a GPU, initialised BRAM on an FPGA. No allocation per use |
| **isEmpty** | Literal comparison `tag = Empty`, one load + one compare; never a null check |
| **Immutable** | Always; structural sharing between persistent values is index aliasing within one arena. The sentinel resides `ReadOnly`: a store through it is CCS8020 at compile time, and consing onto it produces a fresh arena node |
| **Allocation** | Arena or stack, or the platform's declared program-lifetime space for program-lifetime values (immutable or mutable, cited by name from the platform description as for the sentinel above); not GC heap |
| **MLIR** | `index` — link to an arena-placed cons cell ([List Operations](list-operations-representation.md)) |

**Arena Allocation**: List cons cells are allocated in arenas, or in the platform's declared program-lifetime space for program-lifetime values, cited by name from the platform description (rodata/data on an ELF target, flash/SRAM on an MCU, constant memory on a GPU, initialised BRAM on an FPGA; not GC heap), providing better cache locality and batch deallocation at scope end. The immutable structure enables structural sharing as in OCaml. On a heap-free target a cons cell whose lifetime classifies as genuinely dynamic is a compile-time lifetime error, not a silent heap allocation.

**When to Use**:
- Pattern matching on head/tail
- Recursive algorithms
- Functional transformations (map, filter, fold)

**When NOT to Use** (prefer array):
- Random access by index
- Performance-critical loops
- Large collections with mutations

### 5.4 Map

```fsharp
let lookup : Map<string, int> = Map.ofList [("a", 1); ("b", 2)]
let empty : Map<int, string> = Map.empty
```

**Memory Layout** (AVL tree nodes):
```
Map<'K, 'V>  (AVL tree node)
┌────────────────────┬────────────────────┬───────────────────────────┬───────────────────────────┬────────────┐
│ key: 'K            │ value: 'V          │ left: index (arena link)  │ right: index (arena link) │ height: i8 │
└────────────────────┴────────────────────┴───────────────────────────┴───────────────────────────┴────────────┘
     sizeof<'K>           sizeof<'V>             platform word               platform word            1 byte
```

> **Platform word**: `left` and `right` are each an `index` (arena link), an arena-relative offset into the arena buffer the tree lives in: one platform word each, 8 bytes on x86-64 and 4 bytes on thumbv8m/M33. Each carries VC-LINK, `0 <= i < extent(arena)`, with `i = 0` the sentinel at offset 0 of that arena, discharged at saturation before witnessing. A link load is one `memref.load` of an `index`.

| Property | Value |
|----------|-------|
| **Empty map** | Index `0`: the sentinel at offset 0 of every hosting arena, a copy of one program-lifetime, immutable sentinel image per instantiation `Map<'K, 'V>`, `height = 0`, `left` and `right` = 0, `key` and `value` zero-initialised static data that no operation reads (VC-GUARD: `height ≠ 0` dominates every read of `key`/`value` on the saturated graph). It resides in the platform's declared immutable program-lifetime space, cited by name from the platform description (a `Resides` edge): rodata on an ELF target, flash on an MCU, constant memory on a GPU, initialised BRAM on an FPGA. No allocation per use |
| **isEmpty** | Literal comparison `height = 0`, one load + one compare; never a null check |
| **Structure** | Self-balancing AVL tree. Every link is a valid node; insert, lookup, rotate and rebalance read `left`/`right` unconditionally and recursion terminates at the sentinel (`height = 0`), so no operation has an absence branch |
| **Immutable** | Always; structural sharing on update is index aliasing within one arena. The sentinel resides `ReadOnly`: a store through it is CCS8020 at compile time, and `Map.add` on the sentinel produces a fresh arena node |
| **Allocation** | Arena or stack, or the platform's declared program-lifetime space for program-lifetime values (immutable or mutable, cited by name from the platform description as for the sentinel above); not GC heap |
| **Key constraint** | `'K : comparison` |
| **MLIR** | `index` — link to an arena-placed node ([Map Representation](map-representation.md)) |

**AVL Balance Property**: Height difference between left and right subtrees is at most 1. Rebalancing occurs on `Map.add` when this property would be violated.

**When to Use**:
- Key-value associations with O(log n) lookup
- Ordered iteration by key
- Immutable dictionary semantics

**When NOT to Use** (prefer Dictionary or array):
- Frequent updates (mutable dictionary better)
- Small fixed key sets (array with enum index)
- Hash-based O(1) lookup needed

> **See**: [Map Representation](map-representation.md) for detailed AVL algorithms and HOF specifications.

### 5.5 Set

```fsharp
let numbers : Set<int> = Set.ofList [1; 2; 3]
let empty : Set<string> = Set.empty
```

**Memory Layout** (AVL tree nodes):
```
Set<'T>  (AVL tree node)
┌────────────────────┬───────────────────────────┬───────────────────────────┬────────────┐
│ value: 'T          │ left: index (arena link)  │ right: index (arena link) │ height: i8 │
└────────────────────┴───────────────────────────┴───────────────────────────┴────────────┘
     sizeof<'T>             platform word               platform word            1 byte
```

> **Platform word**: `left` and `right` are each an `index` (arena link), an arena-relative offset into the arena buffer the tree lives in: one platform word each, 8 bytes on x86-64 and 4 bytes on thumbv8m/M33. Each carries VC-LINK, `0 <= i < extent(arena)`, with `i = 0` the sentinel at offset 0 of that arena, discharged at saturation before witnessing. A link load is one `memref.load` of an `index`.

| Property | Value |
|----------|-------|
| **Empty set** | Index `0`: the sentinel at offset 0 of every hosting arena, a copy of one program-lifetime, immutable sentinel image per element type, `height = 0`, `left` and `right` = 0, `value` zero-initialised static data that no operation reads (VC-GUARD: `height ≠ 0` dominates every read of `value` on the saturated graph). It resides in the platform's declared immutable program-lifetime space, cited by name from the platform description (a `Resides` edge): rodata on an ELF target, flash on an MCU, constant memory on a GPU, initialised BRAM on an FPGA. No allocation per use |
| **isEmpty** | Literal comparison `height = 0`, one load + one compare; never a null check |
| **Structure** | Self-balancing AVL tree. Every link is a valid node; insert, lookup, rotate and rebalance read `left`/`right` unconditionally and recursion terminates at the sentinel (`height = 0`), so no operation has an absence branch |
| **Immutable** | Always; structural sharing on update is index aliasing within one arena. The sentinel resides `ReadOnly`: a store through it is CCS8020 at compile time, and `Set.add` on the sentinel produces a fresh arena node |
| **Allocation** | Arena or stack, or the platform's declared program-lifetime space for program-lifetime values (immutable or mutable, cited by name from the platform description as for the sentinel above); not GC heap |
| **Element constraint** | `'T : comparison` |
| **MLIR** | `index` — link to an arena-placed node ([Set Representation](set-representation.md)) |

**Relationship to Map**: `Set<'T>` is structurally equivalent to `Map<'T, unit>` but with optimized layout (no value field).

**When to Use**:
- Membership testing with O(log n) lookup
- Ordered unique elements
- Set operations (union, intersect, difference)

**When NOT to Use** (prefer HashSet or array):
- Very frequent membership tests (hash-based O(1) better)
- When order doesn't matter and hash is cheaper

> **See**: [Set Representation](set-representation.md) for detailed AVL algorithms and HOF specifications.

---

## Part 6: Function Types

> **Principle**: Functions are first-class values. Inline-by-default semantics (fsil) eliminates most closure overhead.

### 6.1 Pure Functions

```fsharp
let add : int -> int -> int = fun x y -> x + y
let apply : ('a -> 'b) -> 'a -> 'b = fun f x -> f x
```

**Representation Strategies**:

| Case | Representation |
|------|----------------|
| **Known call site** | Direct call (no indirection) |
| **Inline function** | Inlined at call site (fsil default) |
| **First-class value** | A function value (`func.constant`) or a closure pair `(fn, env)` |
| **Captures environment** | The closure pair `(fn, env)`: `fn` a function value, `env` a `memref<Exi8>` ([Closure Representation §6.3](closure-representation.md)) |

### 6.2 Closure Representation

When a function captures variables from its environment:

```fsharp
let makeAdder n =
    fun x -> x + n  // Captures 'n'
 
```

**Form** — two SSA values, never packed ([Closure Representation §6.3](closure-representation.md)):
```
Closure = (fn, env)
fn:  a function value (func.constant), not stored as data
env: ┌─────────────────────┐
     │ captured values     │   memref<Exi8>, E = sizeof<env>, literal at saturation
     └─────────────────────┘
```

| Property | Value |
|----------|-------|
| **Environment** | `memref<Exi8>` holding the captured values at literal offsets settled at saturation |
| **Invocation** | `func.call_indirect %fn(%env, args...)`; `fn` and `env` are two SSA values, never packed ([Closure Representation §6.1](closure-representation.md)) |
| **MLIR** | `(fn, env)`: a function value `(memref<Exi8>, args...) -> ret` and `memref<Exi8>` ([Closure Representation §6.3](closure-representation.md)) |

**Closure Allocation**: The environment is allocated on stack or in an arena, or in the platform's declared program-lifetime space for program-lifetime values, cited by name from the platform description (rodata/data on an ELF target, flash/SRAM on an MCU, constant memory on a GPU, initialised BRAM on an FPGA; not GC heap); `fn` is a `func.constant` and is never stored as data. Small closures (<64 bytes) fit within one cache line for efficient invocation. On a heap-free target a closure whose lifetime classifies as genuinely dynamic is a compile-time lifetime error, not a silent heap allocation.

### 6.3 Inline Semantics (fsil Absorption)

Most functions are inlined by default:

```fsharp
// These are inlined at call site - no closure allocation
let inline double x = x * 2
let inline (|>) x f = f x
```

| Function Type | Default Behavior |
|---------------|------------------|
| Simple arithmetic | Always inlined |
| Pipe operators | Always inlined |
| SRTP-constrained | Always inlined |
| Recursive | Not inlined (requires call) |
| Large body | Compiler decides |

### 6.4 Partial Application

```fsharp
let add x y = x + y
let add5 = add 5  // Partial application
 
```

**Representation**: Creates a closure capturing applied arguments:
```
add5 = (fn, env)
fn:  func.constant @add_impl
env: { x = 5 } : memref<Exi8>
```

**Currying Optimization** (fsil): Fully-applied curried calls compile to direct multi-argument calls (no intermediate closures). Partial application creates flat closures capturing applied arguments. Higher-order uses like `List.map f` are typically inlined at call sites.

> **See**: [ClefExpr § 5.2 Curried Call Flattening](clef-expr.md#52-curried-call-flattening) for the normative specification of how the SemanticGraph represents curried applications.

---

## Part 7: Mutable State

> **Principle**: Mutable state is explicit and controlled. All mutation is visible in the type system or syntax.

### 7.1 Ref Cells

```fsharp
let counter : int ref = ref 0
counter := !counter + 1  // Mutation
let value = !counter     // Dereference
 
```

**Memory Layout**:
```
ref<'T>
┌────────────────────┐
│ contents: 'T       │
└────────────────────┘
    sizeof<'T>
```

| Property | Value |
|----------|-------|
| **Representation** | Single-field mutable record |
| **Allocation** | Stack, arena, or the platform's declared mutable program-lifetime space for program-lifetime values, cited by name from the platform description (data on an ELF target, SRAM on an MCU; not GC) |
| **MLIR** | `memref<1xT>` |

**Stack/Arena Allocation**: Refs are allocated on stack or in arenas, or in the platform's declared mutable program-lifetime space for program-lifetime values (data on an ELF target, SRAM on an MCU; not GC heap), eliminating allocation pressure in loops and providing predictable memory behavior for embedded targets. On a heap-free target a ref whose lifetime classifies as genuinely dynamic is a compile-time lifetime error, not a silent heap allocation.

### 7.2 Mutable Bindings

```fsharp
let mutable x = 0
x <- x + 1  // Direct mutation
 
```

| Property | Value |
|----------|-------|
| **Scope** | Local to function |
| **Representation** | Stack slot |
| **Cannot escape** | Cannot be captured by closures |

**Escape Prevention**: Mutable bindings cannot be captured by closures (use `ref` instead). This restriction enables guaranteed stack allocation.

### 7.3 Mutable Record Fields

```fsharp
type Counter = { mutable Value: int }

let c = { Value = 0 }
c.Value <- c.Value + 1
```

| Property | Value |
|----------|-------|
| **Layout** | Same as immutable field |
| **Mutability** | Compile-time property |

**Cache Line Isolation**: For concurrent access, isolate frequently-written mutable fields using `[<CacheLinePadded>]` to prevent false sharing.

---

## Part 8: Memory Region Types (UMX Integration)

> **Core Fidelity Principle**: Memory region types are **intrinsic to Clef** - as fundamental as `voption`. They carry semantic meaning through the entire compilation pipeline, guiding every memory layout decision. **Fidelity makes ALL memory layout decisions - MLIR/LLVM never determine layout.**
>
> **Erasure at Last Lowering**: These types ARE erased - but at the **last possible lowering stage**, after Fidelity has made all memory layout decisions. By the time code reaches LLVM, "the type information that guided every transformation has done its job and compiled away to nothing." This is the entire point of "Fidelity" - preserving type fidelity through compilation so the F# compiler controls memory layout.
>
> **CCS Enforcement**: See the CCS specification for region constraint enforcement and diagnostic codes.

### 8.1 Memory Regions

Memory region types define where memory lives and how it behaves. They guide Fidelity's code generation decisions throughout the pipeline:

| Region | Use Case | Volatile | Cacheable | Code Generation Effect |
|--------|----------|----------|-----------|------------------------|
| `Stack` | Thread-local automatic | No | Yes | Stack allocation, automatic cleanup |
| `Arena` | Bulk allocation, batch free | No | Yes | Arena allocator calls, scope-bounded |
| `Peripheral` | Memory-mapped I/O | Yes | No | Volatile loads/stores, no reordering |
| `Sram` | General RAM | No | Yes | Standard memory access |
| `Flash` | Read-only program memory | No | Yes | Read-only access, link-time placement |

```fsharp
// These types guide memory layout decisions through compilation
type Stack      // Thread-local, automatic lifetime
type Arena      // Compiler-managed bulk allocation
type Peripheral // Memory-mapped I/O (volatile semantics)
type Sram       // General-purpose RAM
type Flash      // Read-only storage
 
```

**Why This Matters - The "Fidelity" in Fidelity Framework**:

The entire point of Fidelity is that **the F# compiler controls memory layout**, not MLIR, not LLVM. Memory region types:

1. **Carry semantic meaning** through the entire compilation pipeline
2. **Guide Alex's code generation** - `Peripheral` → volatile LLVM instructions
3. **Determine allocation strategy** - `Stack` vs `Arena` vs explicit
4. **Enable compile-time safety** - region mismatches are type errors
5. **Erase at final lowering** - only after ALL decisions are made by Fidelity

This is fundamentally different from letting MLIR/LLVM "figure out" memory layout. Fidelity dictates; LLVM implements.

### 8.2 Access Kinds

Access kinds are also intrinsic - they determine what operations are legal AND affect code generation:

| Kind | Read | Write | CMSIS | Code Generation Effect |
|------|------|-------|-------|------------------------|
| `ReadOnly` | Yes | No | `__I` | No store instructions generated |
| `WriteOnly` | No | Yes | `__O` | No load instructions generated |
| `ReadWrite` | Yes | Yes | `__IO` | Both permitted |

```fsharp
// Intrinsic access kind types
type ReadOnly   // Input registers, flash, const data
type WriteOnly  // Output registers, write-only buffers
type ReadWrite  // General mutable access
 
```

### 8.3 Region-Typed Pointers

Pointers carry region and access information as part of their type:

```fsharp
type Ptr<'T, 'Region, 'Access>

// Examples - region and access are part of the type, not annotations
let gpioReg : Ptr<uint32, Peripheral, ReadWrite> = ...
let flashData : Ptr<byte, Flash, ReadOnly> = ...
let stackBuffer : Ptr<int, Stack, ReadWrite> = ...
```

**Compile-Time Safety**:
```fsharp
// ERROR: Cannot write to readOnly pointer
let writeFlash (p: Ptr<byte, Flash, ReadOnly>) =
    Ptr.write p 0uy  // Compile error!

// OK: Can read from readOnly
let readFlash (p: Ptr<byte, Flash, ReadOnly>) =
    Ptr.read p  // OK
 
```

**Cache Behavior**: `Stack`, `Arena`, `Sram`, and `Flash` regions are cacheable with normal load/store semantics. `Peripheral` access bypasses cache and uses memory barriers - essential for hardware registers where timing and order matter.

### 8.4 Arena as CCS Intrinsic Type

> **Status (January 2026)**: Arena is implemented as an CCS intrinsic type.

**Type Definition**:
```fsharp
Arena<[<Measure>] 'lifetime>
```

**Memory Layout** (NTUCompound 3):
```
Arena<'lifetime>
┌─────────────────┬─────────────────┬─────────────────┐
│ Base: index     │ Capacity: index │ Position: index │
└─────────────────┴─────────────────┴─────────────────┘
     8 bytes           8 bytes           8 bytes       = 24 bytes (64-bit)
```

> **Platform word**: Each field is one platform word, so the struct is 3 words. The 24-byte total is the x86-64 instance; on thumbv8m/M33 (4-byte word) the struct is 12 bytes. `Base` here is an internal compiler field, never user-denotable.

| Property | Value |
|----------|-------|
| **CCS Type** | Intrinsic with NTUCompound(3) |
| **Lifetime param** | Measure type for tracking |
| **Allocation** | Stack-backed, static-backed (the platform's declared program-lifetime space: data/rodata on an ELF target, SRAM/flash on an MCU), or heap-backed where a heap exists |
| **MLIR** | Three-word struct with InsertValue/ExtractValue |

**CCS Intrinsic Operations**:

| Operation | Type Signature |
|-----------|----------------|
| `Arena.fromBuffer` | `array<byte> -> Arena<'lifetime>` |
| `Arena.alloc` | `Arena<'lifetime> byref -> int -> Ptr<byte, Arena, ReadWrite>` |
| `Arena.allocAligned` | `Arena<'lifetime> byref -> int -> int -> Ptr<byte, Arena, ReadWrite>` |
| `Arena.remaining` | `Arena<'lifetime> -> int` |
| `Arena.reset` | `Arena<'lifetime> byref -> unit` |

**Usage Pattern**:
```fsharp
// Arena backed by a bounded stack array
let arenaMem : array<byte> = Array.zeroCreate 4096
let mutable arena = Arena.fromBuffer arenaMem

// Allocate from arena
let buffer = Arena.alloc &arena 256

// Arena freed when stack frame exits
 
```

**Byref Parameter**: Operations that mutate arena state (alloc, reset) take `Arena<'lifetime> byref` to enable in-place position updates without copying the three-word struct (24 bytes on x86-64, 12 bytes on thumbv8m/M33).

**Lifetime Inference Principle**: Arena demonstrates the three-level approach:
- **Level 3 (Explicit)**: Current - full manual control
- **Level 2 (Hints)**: Future - `arena { }` computation expressions
- **Level 1 (Inferred)**: Future - compiler escape analysis

> **See**: [memory-regions.md](memory-regions.md#arena) for detailed Arena semantics.

### 8.5 Hardware Peripheral Descriptors

> **See**: Farscape documentation for peripheral binding generation.

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

---

## Part 9: Coeffects and Resource Tracking

> **DEFERRED TO PHASE B**: This section will be completed after QuantumCredential PoC.

### 9.1 Pure vs Effectful Classification

TBD - Continuation-based RAII model

### 9.2 Resource Coeffects

TBD - `ResourceCoeffect` type integration

---

## Part 10: Interactive Development (clefx)

> **Spec Reference**: See [`interactive-development.md`](../../spec/interactive-development.md) for full specification.

The type universe must account for interactive development scenarios where types are defined and evaluated incrementally.

### 10.1 Interactive Session Types

In clefx (Clef Interactive), types are resolved in an evolving environment:

```fsharp
> type Point = { x: int; y: int };;
type Point

> let origin = { x = 0; y = 0 };;
val origin : Point
```

**Type Resolution in Interactive Mode**:
- Types defined in the session are immediately available
- Types from `#require` directives are loaded into the type environment
- CCS resolves types against both session-local and loaded definitions

### 10.2 Arena Semantics in Interactive Mode

Interactive sessions use arena-based allocation:

| Aspect | Compiled Program | Interactive Session |
|--------|-----------------|---------------------|
| Arena Lifetime | Program-controlled | Session-controlled |
| Reset | Explicit | `#arena reset` directive |
| Persistence | N/A | `#persist` for cross-reset values |

**Type Implications**:
```fsharp
> let data = [1..1000000];;
val data : int list  // Allocated in session arena

> #arena reset;;
// data is no longer valid - type checker knows this
 
```

### 10.3 Interpretation vs Compilation Type Semantics

| Mode | Type Checking | Execution |
|------|---------------|-----------|
| Interpret | Full CCS | Interpreted |
| Compile | Full CCS | Native code |
| Hybrid | Full CCS | Mode-dependent |

All modes use identical type semantics. The difference is only in execution:

```fsharp
> #mode compile;;
> let rec fib n = if n < 2 then n else fib (n-1) + fib (n-2);;
val fib : int -> int  // Same type in all modes
 
```

### 10.4 Script File Type Semantics

Script files (`.clefx`) follow the same type semantics as compiled modules:

```fsharp
// script.clefx
let greeting : string = "Hello"  // string as a UTF-8 memref<?xi8> view
let maybe : int option = Some 42  // voption<int>, non-null
 
```

### 10.5 Cross-Compilation Type Considerations

When targeting a different platform:

```fsharp
> #target linux-arm64;;
Target: linux-arm64 (cross-compiling from linux-x64)

> sizeof<nativeint>;;
val it : int = 8  // Reflects target, not host
 
```

**Platform-Specific Types**:
- `nativeint`/`unativeint` size reflects target platform
- `index` width and the `Ptr<'T, 'Region, 'Access>` handle width reflect the target architecture
- Type layouts follow target ABI

---

## Appendix A: Type Mapping to MLIR

| F# Type | MLIR Type |
|---------|-----------|
| `unit` | (none - ZST) |
| `bool` | `i8` |
| `int` | `index` |
| `int32` | `i32` |
| `int64` | `i64` |
| `float` | `f64` |
| `float32` | `f32` |
| `string` | `memref<?xi8>` |
| `option<'T>` | `memref<Exi8>` — `{tag, payload}`, stack-placed ([Option Operations](option-operations-representation.md)) |
| `'a * 'b` | `tuple<A, B>` |
| Record | `memref<Exi8>` (settled layout) |
| DU | `memref<Exi8>` (`{tag, payload}`) |
| `'a -> 'b` | `(A) -> B` — a `func` value |

---

## Appendix B: OCaml Type System Reference

> **Source**: [OCaml Basic Data Types](https://ocaml.org/docs/basic-data-types), [Memory Representation](https://ocaml.org/docs/memory-representation), [Real World OCaml](https://dev.realworldocaml.org/runtime-memory-layout.html)

### B.1 OCaml Uniform Memory Representation

OCaml uses a "uniform memory representation" where every variable is a single word containing either an immediate integer or a pointer. The runtime distinguishes them using the lowest bit:

- **Bit = 1**: Integer value (unboxed)
- **Bit = 0**: Pointer to heap-allocated data

This tagging allows integers to remain "unboxed" (stored directly without heap allocation), making them fast and register-friendly. The tradeoff is **losing one bit of precision** in integer values.

### B.2 OCaml Primitive Types

| Type | OCaml Representation | Size | Notes |
|------|---------------------|------|-------|
| `int` | Tagged integer | 63-bit (64-bit platform) | One bit reserved for GC tag |
| `float` | Boxed IEEE 754 double | 8 bytes + header | No single-precision built-in |
| `bool` | Unboxed integer | 1 word | `true`=1, `false`=0 |
| `char` | Unboxed integer | 1 byte value | Latin-1 only (0-255) |
| `string` | Block with String_tag | Variable | Word-aligned, explicit length |
| `unit` | Unboxed integer 0 | 0 | Same representation as `[]` |
| `'a option` | Variant | 1 word or boxed | `None`=0, `Some x`=block |

### B.3 OCaml Composite Types

| Type | Representation | Tag |
|------|---------------|-----|
| Tuple | Block with fields | 0 |
| Record | Block with fields | 0 |
| Array | Block with variable fields | 0 |
| Float array | Unboxed contiguous | 254 (Double_array_tag) |
| Variant (no param) | Unboxed integer | N/A |
| Variant (with param) | Block with tag + fields | 0-245 |

### B.4 Key OCaml Design Decisions Clef Diverges From

| OCaml Choice | Clef Choice | Rationale |
|--------------|-----------------|-----------|
| 63-bit tagged int | Full word `nativeint` | No GC tag needed (compile-time safety) |
| Latin-1 char | UTF-32 codepoint | Modern Unicode support |
| Byte-sequence string | UTF-8 `memref<?xi8>` view | Text correctness + Rust interop |
| Boxed option (sometimes) | Always stack `voption` | Null-freedom guarantee |
| Runtime type tagging | Compile-time only | No runtime overhead |

---

## Appendix C: Comparison with OCaml/F#/Rust

| Aspect | OCaml | F# (.NET) | Clef | Rust |
|--------|-------|-----------|----------|------|
| String encoding | Byte sequence | UTF-16 | **UTF-8** | UTF-8 |
| Option | `'a option` (heap) | `'a option` (heap) | **`voption` (stack)** | `Option<T>` (stack) |
| Default int | 63-bit tagged | 32-bit | **Platform word** | Platform word |
| Null | No null | Nullable refs | **No null** | No null |
| Memory safety | Runtime GC | Runtime GC | **Compile-time** | Compile-time |
| Type representation | Runtime tagged | Runtime boxed | **Compile-time** | Compile-time |

---

## Appendix D: Migration from Shadow Types

> **Note**: CCS provides native type resolution directly. No shadow types or library workarounds are needed.

### FSharp.Core Types Leveraged

| Type | Location | Notes |
|------|----------|-------|
| `voption<'T>` | FSharp.Core | THE underlying option implementation |
| `nativeint` | FSharp.Core | Platform-sized integer |
| `Span<'T>` | FSharp.Core | Contiguous memory view |

### CCS Type Resolution

CCS (Clef Compiler Service) resolves types to native representations at the source:
- `string` → UTF-8 `memref<?xi8>`
- `option<'T>` → `voption` (stack-allocated)
- `array<'T>` → `memref<?xT>` with native element layout
- `int` → platform word

---

## Appendix E: OCaml Provenance and Fidelity Extensions

> **Design Philosophy**: Clef draws provenance from OCaml's direct memory layout idioms - concepts that F#/.NET lacks entirely because the CLR abstracts memory away. However, OCaml is desktop-centric and non-cache-aware. Fidelity extends OCaml's foundation with modern hardware realities while incorporating Rust's RAII principles.

### E.1 What OCaml Provides (KEEP)

OCaml contemplates direct memory layout in ways F#/.NET never does:

| OCaml Concept | Value for Clef | Notes |
|---------------|-------------------|-------|
| **Products/Sums/Functions as primitives** | Foundation | Type universe axioms (Part 1) |
| **Value-oriented structural assembly** | Core principle | Records/tuples laid out contiguously |
| **Deterministic tag layout for DUs** | Directly usable | Tag = 0,1,2... in declaration order |
| **Explicit-length string block** | Adapted | The length travels with the value: in OCaml's block header, in Clef as the dimension of the `memref<?xi8>` / `memref<?xT>` view |
| **No null philosophy** | Core principle | Everything representable without null or magic values |
| **Unboxed by default** | Core principle | No implicit heap allocation |

These concepts form the bedrock of Clef's type universe. The structural assembly semantics, deterministic tag assignment, and explicit-length model are adopted with minimal modification.

### E.2 What OCaml Lacks (SET ASIDE)

OCaml's desktop-centric, non-cache-aware limitations that Clef explicitly diverges from:

| OCaml Limitation | Fidelity Requirement | Gap |
|------------------|---------------------|-----|
| **63-bit tagged integers** | Full-width integers | GC tag overhead eliminated - compile-time safety replaces runtime discrimination |
| **No cache line awareness** | 64-byte alignment matters | Working set calculation, false sharing prevention |
| **No memory region types** | Stack/Arena/Peripheral/Sram/Flash | Compile-time placement decisions |
| **No access kinds** | ReadOnly/WriteOnly/ReadWrite | Hardware register semantics for embedded targets |
| **Desktop "sufficient RAM" assumption** | Embedded/constrained targets | Memory-mapped peripherals, limited stack |
| **No NUMA awareness** | Multi-socket topology | Actor placement strategies (Prospero) |
| **Runtime type discrimination** | Compile-time only | No runtime overhead |
| **No prefetch/cache bypass semantics** | Processor-specific strategies | Intel vs AMD vs ARM optimization |

OCaml assumes a desktop environment with ample RAM and a garbage collector managing memory. Clef targets the full spectrum from microcontrollers to GPU clusters.

### E.3 Rust RAII Guideposts (ADAPT)

Rust's ownership model provides valuable guideposts, adapted to F#'s functional idioms:

| Rust Concept | Fidelity Adaptation | Implementation |
|--------------|---------------------|----------------|
| **Ownership** | Memory region types + coeffects | `Ptr<'T, 'Region, 'Access>` carries placement semantics |
| **Borrow checker** | Type-guided analysis | Less invasive than Rust's explicit lifetimes |
| **Drop semantics** | Continuation-bounded resources | RAII via delimited continuation completion |
| **Deterministic cleanup** | Arena/scope-bounded lifetimes | Resources released at scope exit, not GC |
| **Move semantics** | Linear types (Phase B) | Ownership transfer without copying |

Rust's insight that ownership can be tracked statically is preserved, but expressed through F#'s type system rather than explicit lifetime annotations.

### E.4 Fidelity Extensions Beyond Both

Fidelity's cache-aware compilation doctrine extends beyond both OCaml and Rust:

#### Cache Hierarchy Awareness

| Concern | OCaml | Rust | Fidelity |
|---------|-------|------|----------|
| **Cache line alignment (64 bytes)** | No | Manual | First-class - `[<CacheLineAligned>]` |
| **False sharing prevention** | No | Manual | Automatic for mutable fields |
| **Working set calculation** | No | No | Compile-time via BAREWire layouts |
| **L1/L2/L3 tier placement** | No | No | Arena configuration strategies |

#### Memory Hierarchy Management

```fsharp
// Fidelity-specific: Cache-conscious arena configuration
let configureArena (profile: ActorProfile) =
    match profile.WorkingSetSize with
    | size when size <= 32<KB> -> L1Resident    // Fits in L1
    | size when size <= 256<KB> -> L2Optimized  // Fits in L2
    | size when size <= 8<MB> -> L3Optimized    // Fits in L3
    | _ -> StreamingMode                         // Bypass cache
 
```

#### Processor-Specific Optimization

| Feature | Purpose | OCaml/Rust |
|---------|---------|------------|
| **Prefetch distance calibration** | Intel vs AMD vs ARM strategies | No |
| **Non-temporal moves** | Bypass cache for streaming data | Manual only |
| **NUMA-aware placement** | Multi-socket actor placement | No |
| **TLB/huge page selection** | Based on access density | OS-level only |

#### Hardware Memory Regions

Clef introduces memory regions unknown to both OCaml and Rust:

| Region | Use Case | OCaml | Rust | Clef |
|--------|----------|-------|------|----------|
| `Stack` | Thread-local automatic | Implicit | Implicit | Explicit type |
| `Arena` | Bulk allocation, batch free | No | Manual | First-class |
| `Peripheral` | Memory-mapped I/O (volatile) | No | Manual unsafe | Type-safe |
| `Sram` | General RAM | N/A | N/A | Explicit |
| `Flash` | Read-only program memory | N/A | N/A | Explicit |

### E.5 The Synthesis

Clef's type universe represents a synthesis:

1. **From OCaml**: Products, sums, functions as primitives; value-oriented assembly; explicit-length views; no null
2. **Set Aside from OCaml**: GC tagging overhead; desktop assumptions; runtime discrimination
3. **From Rust**: Deterministic cleanup; ownership tracking (adapted to coeffects); RAII semantics
4. **Fidelity Original**: Memory regions; access kinds; cache hierarchy awareness; processor-specific optimization

This synthesis provides provenance (we didn't start from scratch) while forging a path suited to native compilation across the full hardware spectrum - from embedded microcontrollers to GPU clusters.

### E.6 References

- OCaml Memory Representation: https://ocaml.org/docs/memory-representation
- Real World OCaml, Runtime Memory Layout: https://dev.realworldocaml.org/runtime-memory-layout.html
- Rust Ownership: https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html
- Fidelity Cache-Aware Compilation: See SpeakEZ blog posts on cache-conscious memory management

---

## Changelog

- **2024-12-30**: Session 1 (Unit, Bool, Integers) - Comprehensive primitive type specification with OCaml provenance, cache alignment considerations, resolved char design decision (UTF-32 codepoints)
- **2024-12-30**: Added Appendix E (OCaml Provenance and Fidelity Extensions) documenting design heritage and extensions
- **2024-12-30**: Initial skeleton created from plan
