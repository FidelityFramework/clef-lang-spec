---
title: "NTU Type Nomenclature Specification"
weight: 140
category: Language
status: normative
---

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-09-04

## 1. Overview

This chapter specifies the NTU (Native Type Universe) nomenclature used internally by CCS (Clef Compiler Services) for platform-generic types. NTU types resolve against [quotation-based platform bindings](platform-bindings.md) in CCS, at saturation, per program-graph section.

Width is not part of a numeric type's identity. A numeric type is its kind (integer or real) and its dimension ([Units of Measure](units-of-measure.md)); its width and representation are coeffects derived from the value's analysed range ([Width Inference](width-inference.md), [Numeric Selection](numeric-selection.md)). `NTUWidth` names the **seals** a value may carry: `Fixed n` is a written width (`int32`, `uint8`, `float32`), and `Resolved Pointer` or `Resolved Register` is a width the platform description supplies at a site its ABI governs (`nativeint`, an exported parameter, a foreign call). A seal is a Tier-3 range claim checked for coverage against the analysed range ([Numeric Selection §3.3, §5](numeric-selection.md)); it is never unified. The bare kinds `int` and `float` carry no seal.

### 1.1 Core Principle

**Platform awareness flows FROM THE TOP via quotation-based binding libraries, and CCS applies it at saturation.**

CCS checks type identity (kind and dimension), propagates the range coeffect, checks every seal for coverage, and resolves `Resolved` seals against the platform description of the section the value lives in. Alex reads the resolved width and representation from the node and emits the corresponding MLIR type; it computes no width and consults no platform context ([NTU Dimensional Architecture §4.3, §5.2](ntu-dimensional-architecture.md)).

## 2. NTU Type Categories

### 2.1 Width Dimensions

Width is parameterized, not baked into variant names. A `WidthDimension` is a name the platform description declares; the well-known CPU names are NTU-native and are NOT named after C types.

```fsharp
/// A width dimension is a name the platform description declares
/// (ntu-dimensional-architecture.md §7.1). CPU descriptions declare
/// Pointer and Register; a fabric binding declares its port widths.
type WidthDimension = WidthDimension of name: string

/// How the width of a numeric type is determined
type NTUWidth =
    | Fixed of bits: int              // Known at all times: 8, 16, 32, 64
    | Resolved of WidthDimension      // Platform-dependent, resolved by CCS at saturation against the platform description
 
```

### 2.2 Parameterized Numeric Types

| NTUKind | Width | Description | MLIR Type |
|---------|-------|-------------|-----------|
| `NTUint (Fixed 8)` | 8-bit signed | `int8` / `sbyte` | `i8` |
| `NTUint (Fixed 16)` | 16-bit signed | `int16` | `i16` |
| `NTUint (Fixed 32)` | 32-bit signed | `int32` | `i32` |
| `NTUint (Fixed 64)` | 64-bit signed | `int64` | `i64` |
| `NTUint`, no seal | bare integer kind, signed by range | `int` | `i<w>`, `w` the width the range requires; the word at an ABI site |
| `NTUint (Resolved Pointer)` | Pointer-sized, signed | `nativeint` | `index` |
| `NTUuint (Fixed 8)` | 8-bit unsigned | `uint8` / `byte` | `i8` |
| `NTUuint (Fixed 16)` | 16-bit unsigned | `uint16` | `i16` |
| `NTUuint (Fixed 32)` | 32-bit unsigned | `uint32` | `i32` |
| `NTUuint (Fixed 64)` | 64-bit unsigned | `uint64` | `i64` |
| `NTUuint`, no seal | bare integer kind, range claim `[0, ∞)` | `uint` | `i<w>`, as above |
| `NTUuint (Resolved Pointer)` | Pointer-sized, unsigned | `unativeint` | `index` |
| `NTUfloat (Fixed 32)` | 32-bit IEEE 754 | `float32` | `f32` |
| `NTUfloat (Fixed 64)` | 64-bit IEEE 754 | `float` | `f64` |

### 2.3 Pointer and Semantic Types (Implicit Pointer Width)

| NTU Type | Semantic Meaning | Resolution |
|----------|------------------|------------|
| `NTUptr` | Native pointer to type T | Pointer dimension |
| `NTUfnptr` | Function pointer (no closures) | Pointer dimension |
| `NTUsize` | Size type (unsigned, pointer-width) | Pointer dimension |
| `NTUdiff` | Pointer difference (signed, pointer-width) | Pointer dimension |

## 3. Type Identity and Width

### 3.1 Type Identity (CCS)

Type identity is kind and dimension. Seals are coeffects beside the type and do not enter unification:

```
int<1>     = int<1>          // same kind, same dimension
float<m>   ≠ float<s>        // dimension mismatch, CCS8040
int<1>     ≠ float<1>        // kind mismatch, CCS8000
```

Two values carrying different seals that meet at an operator or a binding form an explicit-conversion site (`CCS8013`), not a unification failure. A bare value meeting a sealed one adopts the seal when its analysed range is covered (`R₁ ⊆ dynrange(r)`, [Numeric Selection §3.4](numeric-selection.md)), else `CCS8012`:

```fsharp
let add (x: int) (y: int) = x + y          // bare: width follows the range
let a = 1 + 1L                             // accepted: the bare literal [1, 1] adopts int64
let bad (x: int32) (y: int64) = x + y      // CCS8013: int32 meets int64; convert explicitly
```

### 3.2 Width (CCS derives, Alex reads)

CCS derives each integer's width and signedness from its analysed range ([Width Inference §3](width-inference.md)) and each real's representation from its dimensional range ([Numeric Selection §2](numeric-selection.md)). A `Resolved` seal takes its value from `PlatformContext.Dimensions` of the section, a map the platform description declares; the language specification names no architecture and tabulates no widths. A CPU description declares `Pointer` and `Register`; a fabric binding declares the widths of the ports it governs. A bare `int` has no entry in that map: its width is its range's, rounded up to the native size on the CPU leg ([Width Inference §8](width-inference.md)) and exact on the FPGA leg. An unobservable range that carries no seal from any source is `CCS8011`; it is never defaulted to a word.

### 3.3 Width Is Never Erased by the Checker

The range, width, representation and seal ride the PSG as coeffects from the pass that settles them, are read by every later pass without recomputation, and are dropped only at native emission, where they become debug metadata ([DTS/DMM §2.3](https://arxiv.org/abs/2603.16437)). No pass below the witness boundary decides a width.

## 4. Mapping to F# Source Types

### 4.1 Layered Type Abstraction

The architecture uses a three-tier exposure model:

| F# Source | CCS Internal (NTUKind) |
|-----------|-------------------------|
| `int` | `NTUint`, no seal (bare integer kind; width from the range) |
| `uint` | `NTUuint`, no seal (bare, with the range claim `[0, ∞)`) |
| `int8` / `sbyte` | `NTUint (Fixed 8)` |
| `int16` | `NTUint (Fixed 16)` |
| `int32` | `NTUint (Fixed 32)` |
| `int64` | `NTUint (Fixed 64)` |
| `uint8` / `byte` | `NTUuint (Fixed 8)` |
| `uint16` | `NTUuint (Fixed 16)` |
| `uint32` | `NTUuint (Fixed 32)` |
| `uint64` | `NTUuint (Fixed 64)` |
| `nativeint` | `NTUint (Resolved Pointer)` |
| `unativeint` | `NTUuint (Resolved Pointer)` |
| `float32` | `NTUfloat (Fixed 32)` |
| `float` | `NTUfloat (Fixed 64)` |
| `Ptr<'T, Region, Access>` | `NTUptr` |

`NTUptr` is the internal kind for a pointer-shaped value. The only user-denotable source that maps to it is the opaque handle `Ptr<'T, Region, Access>` returned across a C binding, together with the register handle `Mmio` for a fixed-address peripheral. `nativeptr<'T>` is not a Clef source type and never appears in user code; the compiler reaches `NTUptr` through its own internal plumbing, not through a written `nativeptr<'T>` annotation.

### 4.2 Developer Experience

**Level 1 (Default)**: Developers use standard F# type names.
```fsharp
let x: int = 42  // CCS resolves NTUint
let arr: array<int> = [| 1; 2; 3 |]
```

**Level 2/3 (Explicit)**: Developers use semantic aliases for clarity. A byte buffer is a bounded stack array, not a raw pointer.
```fsharp
let write (fd: platformint) (buf: array<byte>) (count: platformsize) : platformint =
    Platform.Bindings.write fd buf count
```

## 5. NTUKind Implementation

### 5.1 Width and Kind Definitions

```fsharp
/// A width dimension is a name the platform description declares; the
/// well-known CPU names are Pointer and Register (ntu-dimensional-architecture.md §7.1).
type WidthDimension = WidthDimension of name: string

/// How the width of a numeric type is determined.
[<RequireQualifiedAccess>]
type NTUWidth =
    | Fixed of bits: int              // Known at all times: 8, 16, 32, 64
    | Resolved of WidthDimension      // Platform-dependent, resolved by CCS at saturation against the platform description

/// NTU (Native Type Universe) type kinds.
/// Numeric types parameterized by width. 3 kinds replace 16 discrete variants.
[<RequireQualifiedAccess>]
type NTUKind =
    // Parameterized numeric types (width as dimension)
    | NTUint of NTUWidth      // Signed integer of any width
    | NTUuint of NTUWidth     // Unsigned integer of any width
    | NTUfloat of NTUWidth    // IEEE float of any width
    // Pointer types (width = Pointer, implicit)
    | NTUptr                  // Native pointer
    | NTUfnptr                // Function pointer
    | NTUsize                 // Size type (unsigned, pointer-width)
    | NTUdiff                 // Pointer difference (signed, pointer-width)
    // Special types
    | NTUstring | NTUbool | NTUchar | NTUunit | NTUdecimal
    | NTUlazy | NTUseq
    // Collection types
    | NTUarray | NTUlist | NTUmap | NTUset
    // Compound value types
    | NTUuuid | NTUdatetime | NTUtimespan
```

### 5.2 PlatformContext (Width Resolution)

```fsharp
type PlatformContext = {
    PlatformId: string
    /// Width dimension resolutions (bits)
    Dimensions: Map<WidthDimension, int>  // Pointer → 64, Register → 64, etc.
    PointerAlign: int
    PlatformLibraryPath: string option
    Predicates: Map<PlatformPredicate, bool>
    FreestandingStartup: FreestandingStartup option
}

module PlatformContext =
    /// Resolve an NTUWidth to concrete bits
    let resolveWidth (ctx: PlatformContext) (width: NTUWidth) : int =
        match width with
        | NTUWidth.Fixed bits -> bits
        | NTUWidth.Resolved dim -> ctx.Dimensions.[dim]

    let defaultLinux_x86_64 = {
        PlatformId = "Linux_x86_64"
        Dimensions = Map.ofList [ (WidthDimension "Pointer", 64); (WidthDimension "Register", 64) ]
        PointerAlign = 8
        PlatformLibraryPath = None
        Predicates = Map.ofList [ ... ]
        FreestandingStartup = None
    }
```

## 6. Unification Rules

### 6.1 Kind and Dimension Unify; Seals Do Not

```
unify(int<1>, int<1>)                 = Success
unify(float<m s^-1>, float<'u>)       = Success, 'u := m s^-1
unify(float<m>, float<s>)             = Error(CCS8040)
unify(int<1>, float<1>)               = Error(CCS8000)
unify(NTUptr<int>, NTUptr<float>)     = Error(TypeMismatch)
```

Seals are not arguments to `unify`. They compose by precedence at the site where two values meet ([Numeric Selection §3.4](numeric-selection.md)): the seal's dynamic range is the binding claim, the analysed range becomes the containment obligation.

### 6.2 No Implicit Width Conversion

A value never changes representation without a written conversion. Deferring the representation is what removes conversions; it never introduces one:

```fsharp
let x: int64 = 42        // accepted: the literal is bare, [42, 42] ⊆ dynrange(int64)
let y: int = 42L         // accepted: `int` is the bare kind; the value keeps its int64 seal
let z = int32 x + y      // CCS8013: two seals meet; convert one explicitly
let w: int64 = int64 z   // an explicit conversion names its source and target seals
```

An explicit conversion is typed with a numeric source and a named target seal; a target that cannot hold the source's analysed range is `CCS8017` ([Width Inference §7](width-inference.md)). There is no polymorphic `'T -> Target` conversion.

## 7. Platform Quotation Resolution

### 7.1 Resolution Flow

```
Clef source (int, float<m>, nativeint, x: int32)
    ↓
CCS Elaboration: kind and dimension inferred and unified; seals read once at their sites
    ↓
CCS Saturation, per section: ranges propagated; seals coverage-checked; Resolved seals
    read from PlatformContext.Dimensions; representations selected
    ↓
PSG: kind, dimension, range, width, representation and seal on the node, as coeffects
    ↓
Alex: reads the width and representation the node carries, emits the MLIR type
    ↓
MLIR: i30 (a counter on fabric), i64 (a word-sealed boundary on x86_64), i32 (on ARM32)
```

### 7.2 Platform Quotation Structure

Width resolution is now dimension-based. Platform quotations provide the `Dimensions` map that `PlatformContext.resolveWidth` uses:

```fsharp
// From Fidelity.Platform library: dimension resolution map
type NTUResolutions = Map<WidthDimension, int>

let linux_x86_64: Expr<NTUResolutions> = <@
    Map.ofList [ (WidthDimension "Pointer", 64); (WidthDimension "Register", 64) ]
@>

let linux_arm32: Expr<NTUResolutions> = <@
    Map.ofList [ (WidthDimension "Pointer", 32); (WidthDimension "Register", 32) ]
@>
```

## 8. MLIR Mapping

### 8.1 NTU to MLIR Type Mapping

The middle end (Alex) emits only portable dialect types (`func`, `scf`, `arith`, `memref`, `index`) and commits to no target. A width in MLIR is a statement of what the computation requires, read from the node; it is not a claim about what any device provides. Pointer-shaped NTU kinds map to the platform-sized `index` type, never to a target dialect.

| NTU form on the node | MLIR type |
|---|---|
| `NTUint` / `NTUuint`, no seal, range-derived width `w` | `i<w>` |
| `NTUint` / `NTUuint`, seal `Fixed n` | `i<n>` |
| `NTUint` / `NTUuint`, seal `Resolved d` | `i<Dimensions[d]>` of the section |
| `NTUfloat`, seal `Fixed 32` / `Fixed 64` | `f32` / `f64` |
| a real with a selected representation | the format numeric selection recorded (`f32`, `f64`, a posit width, a fixed-point format) |
| `NTUptr`, `NTUfnptr`, `NTUsize`, `NTUdiff` | `index` |

The backend legs realise that intent differently and neither the annotation nor this chapter records the difference: the LLVM leg rounds `i<w>` up to a native integer size for arithmetic and keeps the analysed range for overflow checking; the CIRCT leg realises exactly `w` flip-flops. `index` is platform-sized on every leg, so one portable mapping carries the correct pointer width per target without the NTU level naming a byte count.

## 9. Conformance Requirements

### 9.1 CCS Requirements

1. **MUST** keep width, representation and seals out of type identity; identity is kind and dimension
2. **MUST** derive width and signedness from the analysed range, never from a type name
3. **MUST** check every seal for coverage against the analysed range and report `CCS8012` when it fails
4. **MUST** resolve `Resolved` seals against the platform description at saturation, per section
5. **MUST** report an unobservable integer range that carries no seal (`CCS8011`) and never default it
6. **MUST** propagate every coeffect through the SemanticGraph unchanged

### 9.2 Alex Requirements

1. **MUST** read the width and representation the node carries
2. **MUST NOT** compute a width, choose an extension, insert a conversion, or consult a platform context
3. **MUST** emit the MLIR type the node carries, consistently across all occurrences

## 10. Related Specifications

- [platform-predicates.md](platform-predicates.md) - Platform predicate semantics
- [native-type-mappings.md](native-type-mappings.md) - F# to native type mappings
- [native-type-universe.md](native-type-universe.md) - Complete type universe specification
