---
title: "NTU Type Nomenclature Specification"
weight: 140
category: Language
status: normative
---

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-09-06
>
> **Normative Scope**: This chapter defines the canonical NTU naming and identity model. Project-level documentation outside this specification is informative only. The NTU rules in this chapter take precedence over any historical or ephemeral documentation that suggests other names, width semantics, or conversion rules.

## 1. Overview

This chapter specifies the NTU (Native Type Universe) nomenclature used internally by CCS (Clef Compiler Services) for numeric and pointer-shaped types. NTU types resolve against [quotation-based platform bindings](platform-bindings.md) in CCS, at saturation, per program-graph section.

A numeric type is its **kind**, integer or real, and its **dimension** ([Units of Measure](units-of-measure.md)). Its width and representation are not part of the type: they are coeffects derived from the value's analysed range and selected from the representations the platform declares ([Width Inference](width-inference.md), [Numeric Selection](numeric-selection.md)). There is one integer kind, `int`, and one real kind, `float`; there is no width-named numeric type, no width-bearing literal suffix, no seal and no conversion (decision D10, `Dimensional_Range_Design.md` in clef `docs/fidelity/phg`). "There is only int that happens to be 8 wide."

### 1.1 Core Principle

**Platform awareness flows FROM THE TOP via quotation-based binding libraries, and CCS applies it at saturation.**

CCS checks type identity (kind and dimension), propagates the range coeffect, selects each value's width or representation from the platform's declared set, and checks every boundary for coverage. Alex reads the selected width and representation from the node and emits the corresponding MLIR type; it computes no width, chooses no extension, inserts no conversion and consults no platform context ([NTU Dimensional Architecture §4.3, §5.2](ntu-dimensional-architecture.md)).

## 2. NTU Type Categories

### 2.1 Width Dimensions and the Width Coeffect

A `WidthDimension` is a name the platform description declares; the well-known CPU names are NTU-native and are NOT named after C types.

```fsharp
/// A width dimension is a name the platform description declares
/// (ntu-dimensional-architecture.md §7.1). CPU descriptions declare
/// Pointer and Register; a fabric binding declares its port widths.
type WidthDimension = WidthDimension of name: string

/// The width coeffect on a numeric node. Never written in source.
type NTUWidth =
    | Selected of bits: int           // from the analysed range, among the declared representations
    | Resolved of WidthDimension      // the platform description's declared width at a boundary its ABI governs
```

### 2.2 The Numeric Kinds

| Kind | Clef | Dimension | Width / representation | MLIR type |
|------|------|-----------|------------------------|-----------|
| integer | `int` | `int<m>` | the smallest declared integer representation covering the analysed range; exactly the range's width on fabric; signed only when the range is negative somewhere | `i<w>` from the node |
| real | `float` | `float<m>` | the argmin of [Numeric Selection §2](numeric-selection.md) over the declared real representations covering the range; `f64` for a bare real of unobservable range | `f32`, `f64`, a posit or fixed-point format, from the node |

The spellings `int8`, `int16`, `int32`, `int64`, `uint8`, `uint16`, `uint32`, `uint64`, `byte`, `sbyte`, `uint`, `nativeint`, `unativeint`, `float32`, `single`, `double`, `float64` and `Posit8`…`Posit64` are not Clef types; a program that writes one is CCS8706. The suffixes `L`, `u`, `uy`, `s`, `n` and `f` are CCS8018. A literal is `int` or `float` at the dimensionless measure `1` with the point range `[v, v]`.

### 2.3 Pointer and Semantic Types (Implicit Pointer Width)

| NTU Type | Semantic Meaning | Resolution |
|----------|------------------|------------|
| `NTUptr` | Native pointer to type T, held by a handle | Pointer dimension |
| `NTUfnptr` | Function pointer (no closures) | Pointer dimension |
| `NTUsize` | Size type (unsigned, pointer-width) | Pointer dimension |
| `NTUdiff` | Pointer difference (signed, pointer-width) | Pointer dimension |

An address is not an integer the program does arithmetic on ([FFI Boundary §1](ffi-boundary.md)): it lives in `Ptr<'T, Region, Access>`, `Mmio` or `CHandle<'T>`. An integer of the platform's pointer width crossing an ABI is `int` at a boundary whose declared representation is the description's `Pointer` width.

## 3. Type Identity, Range and Width

### 3.1 Type Identity (CCS)

Type identity is kind and dimension:

```
int<1>     = int<1>          // same kind, same dimension
float<m>   ≠ float<s>        // dimension mismatch, CCS8040
int<1>     ≠ float<1>        // kind mismatch, CCS8003
```

Two integers of different analysed ranges meeting at an operator are one kind; the result's range is computed and its width selected from it. Nothing is converted and nothing is rejected on width:

```fsharp
let add (x: int) (y: int) = x + y          // one kind; the result's width is the result range's
let a = 1 + 100000                         // range [100001, 100001]
```

### 3.2 The Range (CCS propagates; Alex reads)

CCS assigns every numeric node a range: a literal is a point, arithmetic propagates intervals, a comparison bounds its branch, `%` and `clamp` bound their results, a declaration bounds an input, loops close by widening, a function's parameter is the join over its call sites ([Width Inference §2](width-inference.md)). From the range CCS selects the width or representation among the platform's declared set ([Width Inference §3](width-inference.md), [Numeric Selection §2](numeric-selection.md)). An unobservable integer range is CCS8011; the language specification names no architecture and no default.

### 3.3 Width Is Never Erased by the Checker

The range, width and representation ride the PSG as coeffects from the pass that settles them, are read by every later pass without recomputation, and are dropped only at native emission, where they become debug metadata ([DTS/DMM §2.3](https://arxiv.org/abs/2603.16437)). No pass below the witness boundary decides a width.

## 4. Mapping to Clef Source Types

| Clef source | CCS internal (NTUKind) |
|-------------|-------------------------|
| `int`, `int<m>` | `NTUint`; width the range's |
| `float`, `float<m>` | `NTUfloat`; representation selected from the range |
| `Ptr<'T, Region, Access>`, `Mmio`, `CHandle<'T>` | `NTUptr` |
| `bool`, `char`, `string`, `unit`, `decimal` | the non-numeric primitives |

F#'s raw pointer type is not a Clef source type and never appears in user code; the compiler reaches `NTUptr` through its own internal plumbing.

### 4.1 Developer Experience

A developer writes the kind and, where the quantity has one, the dimension. Widths are never written. A byte buffer is `array<int>` whose element range is the buffer's declared cell range, `[0, 255]` for a platform buffer over octets; the cell's width is that range's.

```fsharp
let x = 42                                  // int, range [42, 42]
let arr = [| 1; 2; 3 |]                     // array<int>, element range [1, 3]
let write (fd: int) (buf: array<int>) (count: int) : int =
    Platform.Bindings.write fd buf count    // fd, count and the result at the boundary's declared widths
```

## 5. NTUKind Implementation

```fsharp
/// A width dimension is a name the platform description declares; the
/// well-known CPU names are Pointer and Register (ntu-dimensional-architecture.md §7.1).
type WidthDimension = WidthDimension of name: string

/// The width coeffect; never written in source.
[<RequireQualifiedAccess>]
type NTUWidth =
    | Selected of bits: int
    | Resolved of WidthDimension

/// NTU (Native Type Universe) kinds.
[<RequireQualifiedAccess>]
type NTUKind =
    | NTUint                  // the integer kind; width from the range
    | NTUfloat                // the real kind; representation from the range
    // Pointer types (width = Pointer, implicit)
    | NTUptr | NTUfnptr | NTUsize | NTUdiff
    // Special types
    | NTUstring | NTUbool | NTUchar | NTUunit | NTUdecimal
    | NTUlazy | NTUseq
    // Collection types
    | NTUarray | NTUlist | NTUmap | NTUset
    // Compound value types
    | NTUuuid | NTUdatetime | NTUtimespan
```

### 5.1 PlatformContext

```fsharp
type PlatformContext = {
    PlatformId: string
    /// The width dimensions the description declares, by declared name (bits)
    Dimensions: Map<string, int>
    /// The representations the description offers, by name: family, bits, exact range,
    /// capability (native | emulated | unavailable), boundary semantics (wrap | saturate | exact)
    Representations: Map<string, NumericRepresentation>
    PlatformLibraryPath: string option
    Predicates: Map<PlatformPredicate, bool>
    FreestandingStartup: FreestandingStartup option
}
```

Both maps are filled once, at saturation, from the platform description compiled into the graph (a quotation or a plain record, read structurally: [Platform Bindings](platform-bindings.md)); a dimension or representation the description does not declare is CCS8203 or CCS8204 at the site that needs it, never a default.

## 6. Unification Rules

### 6.1 Kind and Dimension Unify; Range Is Propagated

```
unify(int<1>, int<1>)                 = Success
unify(float<m s^-1>, float<'u>)       = Success, 'u := m s^-1
unify(float<m>, float<s>)             = Error(CCS8040)
unify(int<1>, float<1>)               = Error(CCS8003)
unify(NTUptr<int>, NTUptr<float>)     = Error(TypeMismatch)
```

The range is not an argument to `unify`: it is propagated forward to a least fixed point and checked by containment where a declaration claims a range ([Numeric Selection §3.4](numeric-selection.md)).

### 6.2 No Conversion

A value never changes representation, because it has none to change: it has a range, and a representation is selected for it wherever it is used. Intended loss is arithmetic ([Width Inference §7](width-inference.md)):

```fsharp
let lo = x % 256              // range [0, 255]: an 8-bit representation is selected
let s  = clamp 0 255 v        // range [0, 255], saturating by construction
let n  = truncate 3.7         // real to integer, range [3, 3]
let r  = float n              // integer to real, exact
```

A value meeting a declared boundary is checked for coverage of its range (CCS8012, a warning promoted under `--warnaserror`); it is never converted, truncated, wrapped or saturated by the compiler.

## 7. Platform Description Resolution

### 7.1 Resolution Flow

```
Clef source (int, float<m>, int<m>)
    ↓
CCS Elaboration: kind and dimension inferred and unified; literals seed point ranges
    ↓
CCS Saturation, per section: ranges propagated to a fixed point; widths and representations
    selected from PlatformContext.Dimensions / Representations; boundaries coverage-checked
    ↓
PSG: kind, dimension, range, width and representation on the node, as coeffects
    ↓
Alex: reads the width and representation the node carries, emits the MLIR type
    ↓
MLIR: i30 (a counter on fabric), i64 (a word-wide boundary on x86_64), i32 (on ARM32)
```

### 7.2 The Declaration

The description declares its widths by name and its representations with family, bits, exact range, capability and boundary semantics, inside a quotation or a plain record ([Platform Bindings, Platform Descriptor](platform-bindings.md)):

```fsharp
let platform: Expr<PlatformDescriptor> = <@
    { ...
      Widths = [ { Name = "Pointer"; Bits = 64 }; { Name = "Register"; Bits = 64 } ]
      Representations = [ (* each offered representation *) ] }
@>
```

## 8. MLIR Mapping

The middle end (Alex) emits only portable dialect types (`func`, `scf`, `arith`, `memref`, `index`) and commits to no target. A width in MLIR is a statement of what the computation requires, read from the node; it is not a claim about what any device provides.

| NTU form on the node | MLIR type |
|---|---|
| `NTUint`, width `w` selected from the range | `i<w>` |
| `NTUint`, `Resolved d` at a boundary | `i<Dimensions[d]>` of the section |
| `NTUfloat`, a selected representation | the format numeric selection recorded (`f32`, `f64`, a posit width, a fixed-point format) |
| `NTUptr`, `NTUfnptr`, `NTUsize`, `NTUdiff` | `index` |

The backend legs realise that intent differently and neither the annotation nor this chapter records the difference: the LLVM leg holds an `i<w>` in the register the selected representation occupies; the CIRCT leg realises exactly `w` flip-flops. The extension an operand needs when it meets a wider one is `extui` for a non-negative range and `extsi` otherwise, from the range's sign, never from a name. `index` is platform-sized on every leg.

## 9. Conformance Requirements

### 9.1 CCS Requirements

1. **MUST** keep width and representation out of type identity; identity is kind and dimension
2. **MUST** derive width and signedness from the analysed range and select among the representations the platform declares, never from a type name
3. **MUST** admit no width-named numeric type and no width-bearing literal suffix (CCS8706, CCS8018)
4. **MUST** check every boundary for coverage against the analysed range and report `CCS8012` when it fails, never converting
5. **MUST** report an unobservable integer range (`CCS8011`) and never default it
6. **MUST** propagate every coeffect through the SemanticGraph unchanged

### 9.2 Alex Requirements

1. **MUST** read the width and representation the node carries
2. **MUST NOT** compute a width, choose an extension, insert a conversion, or consult a platform context
3. **MUST** emit the MLIR type the node carries, consistently across all occurrences

## 10. Related Specifications

- [platform-predicates.md](platform-predicates.md) - Platform predicate semantics
- [native-type-mappings.md](native-type-mappings.md) - F# to native type mappings
- [native-type-universe.md](native-type-universe.md) - Complete type universe specification
- [width-inference.md](width-inference.md), [numeric-selection.md](numeric-selection.md), [rounding.md](rounding.md) - the numeric triptych
