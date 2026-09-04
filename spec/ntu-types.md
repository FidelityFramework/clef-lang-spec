---
title: "NTU Type Nomenclature Specification"
weight: 140
category: Language
status: normative
---

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-02-11

## 1. Overview

This chapter specifies the NTU (Native Type Universe) nomenclature used internally by CCS (Clef Compiler Service) for platform-generic types. NTU types resolve via [quotation-based platform bindings](platform-bindings.md), following the F* pattern where type WIDTH is an erased assumption.

Width is a [first-class dimension](https://arxiv.org/abs/2603.16437) in NTU. Numeric types are parameterized by `NTUWidth`, which can be `Fixed` (known at all times) or `Resolved` (platform-dependent, resolved by Alex via `PlatformContext`). This replaces 16 discrete integer/float variants with 3 parameterized kinds.

### 1.1 Core Principle

**Platform awareness flows FROM THE TOP via quotation-based binding libraries, not from CCS type inference.**

CCS validates type **identity** (e.g. `NTUint (Resolved Register)` vs `NTUint (Fixed 64)`). Alex witnesses platform quotations to determine type **width** (32-bit vs 64-bit).

## 2. NTU Type Categories

### 2.1 Width Dimensions

Width is parameterized, not baked into variant names. The `WidthDimension` type names are NTU-native; they are NOT named after C types.

```fsharp
/// Platform-resolved width dimensions
type WidthDimension =
    | Pointer    // Address width (64-bit on x86_64, 32-bit on ARM32)
    | Register   // Machine register / natural word width

/// How the width of a numeric type is determined
type NTUWidth =
    | Fixed of bits: int              // Known at all times: 8, 16, 32, 64
    | Resolved of WidthDimension      // Platform-dependent, resolved by Alex
 
```

### 2.2 Parameterized Numeric Types

| NTUKind | Width | Description | MLIR Type |
|---------|-------|-------------|-----------|
| `NTUint (Fixed 8)` | 8-bit signed | `int8` / `sbyte` | `i8` |
| `NTUint (Fixed 16)` | 16-bit signed | `int16` | `i16` |
| `NTUint (Fixed 32)` | 32-bit signed | `int32` | `i32` |
| `NTUint (Fixed 64)` | 64-bit signed | `int64` | `i64` |
| `NTUint (Resolved Register)` | Platform word, signed | `int` | platform-dependent |
| `NTUint (Resolved Pointer)` | Pointer-sized, signed | `nativeint` | `index` |
| `NTUuint (Fixed 8)` | 8-bit unsigned | `uint8` / `byte` | `i8` |
| `NTUuint (Fixed 16)` | 16-bit unsigned | `uint16` | `i16` |
| `NTUuint (Fixed 32)` | 32-bit unsigned | `uint32` | `i32` |
| `NTUuint (Fixed 64)` | 64-bit unsigned | `uint64` | `i64` |
| `NTUuint (Resolved Register)` | Platform word, unsigned | `uint` | platform-dependent |
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

## 3. Type Identity and Type Width

### 3.1 Type Identity (CCS Responsibility)

CCS enforces type identity constraints. With parameterized width, type identity includes the width dimension:

```
NTUint(Resolved Register) ≠ NTUint(Fixed 32)    // Different types
NTUint(Resolved Register) ≠ NTUint(Fixed 64)    // Different types
NTUint(Resolved Register) = NTUint(Resolved Register)  // Same type

// Valid
let add (x: int) (y: int) : int = x + y   // Both NTUint(Resolved Register)

// Type Error
let invalid (x: int) (y: int64) = x + y   // NTUint(Resolved Register) ≠ NTUint(Fixed 64)
 
```

### 3.2 Type Width (Alex Responsibility)

Alex resolves type width via platform quotations and the `PlatformContext.Dimensions` map:

| Width Dimension | x86_64 | ARM32 | ARM64 |
|-----------------|--------|-------|-------|
| `Pointer` | 64 bits | 32 bits | 64 bits |
| `Register` | 64 bits | 32 bits | 64 bits |

Example resolutions:

| NTUKind | x86_64 | ARM32 | ARM64 |
|---------|--------|-------|-------|
| `NTUint (Resolved Register)` | 64 bits | 32 bits | 64 bits |
| `NTUint (Resolved Pointer)` | 64 bits | 32 bits | 64 bits |
| `NTUint (Fixed 32)` | 32 bits | 32 bits | 32 bits |
| `NTUfloat (Fixed 64)` | 64 bits | 64 bits | 64 bits |

### 3.3 Erased Width Assumptions

Following F*'s pattern, width is compile-time metadata only:

```fsharp
type NTULayout = {
    Kind: NTUKind
    /// Erased - not present at runtime
    AssumedSize: int option
    AssumedAlignment: int option
}
```

Width assumptions guide type checking but are **erased** before code generation. Alex makes final width decisions based on platform quotations.

## 4. Mapping to F# Source Types

### 4.1 Layered Type Abstraction

The architecture uses a three-tier exposure model:

| F# Source | CCS Internal (NTUKind) |
|-----------|-------------------------|
| `int` | `NTUint (Resolved Register)` |
| `uint` | `NTUuint (Resolved Register)` |
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
/// Platform-resolved width dimensions: NTU-native vocabulary.
[<RequireQualifiedAccess>]
type WidthDimension =
    | Pointer       // Address width (pointer-sized)
    | Register      // Machine register / natural word width

/// How the width of a numeric type is determined.
[<RequireQualifiedAccess>]
type NTUWidth =
    | Fixed of bits: int              // Known at all times: 8, 16, 32, 64
    | Resolved of WidthDimension      // Platform-dependent, resolved by Alex

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
        Dimensions = Map.ofList [ (WidthDimension.Pointer, 64); (WidthDimension.Register, 64) ]
        PointerAlign = 8
        PlatformLibraryPath = None
        Predicates = Map.ofList [ ... ]
        FreestandingStartup = None
    }
```

## 6. Unification Rules

### 6.1 NTU Types Unify Only with Themselves

```fsharp
// Valid unification: same kind and same width
unify(NTUint(Resolved Register), NTUint(Resolved Register)) = Success

// Invalid unification: same kind but different width
unify(NTUint(Resolved Register), NTUint(Fixed 64)) = Error(TypeMismatch)
unify(NTUint(Resolved Register), NTUint(Fixed 32)) = Error(TypeMismatch)

// Invalid unification: different kinds
unify(NTUint(Fixed 32), NTUuint(Fixed 32)) = Error(TypeMismatch)

// Pointer types: element types must unify
unify(NTUptr<int>, NTUptr<int>) = Success
unify(NTUptr<int>, NTUptr<float>) = Error(TypeMismatch)
```

### 6.2 No Implicit Width Coercion

NTU types never implicitly widen or narrow:

```fsharp
// These are TYPE ERRORS, not implicit conversions
let x: int64 = 42       // Error: int ≠ int64
let y: int = 42L        // Error: int64 ≠ int

// Explicit conversions required
let x: int64 = int64 42
let y: int = int 42L
```

## 7. Platform Quotation Resolution

### 7.1 Resolution Flow

```
F# Source (int)
    ↓
CCS: Maps to NTUint(Resolved Register)
    ↓
SemanticGraph: Carries NTUint(Resolved Register) annotation
    ↓
Alex: Resolves Register dimension via PlatformContext.Dimensions
    ↓
MLIR: i64 (on x86_64) or i32 (on ARM32)
```

### 7.2 Platform Quotation Structure

Width resolution is now dimension-based. Platform quotations provide the `Dimensions` map that `PlatformContext.resolveWidth` uses:

```fsharp
// From Fidelity.Platform library: dimension resolution map
type NTUResolutions = Map<WidthDimension, int>

let linux_x86_64: Expr<NTUResolutions> = <@
    Map.ofList [ (Pointer, 64); (Register, 64) ]
@>

let linux_arm32: Expr<NTUResolutions> = <@
    Map.ofList [ (Pointer, 32); (Register, 32) ]
@>
```

## 8. MLIR Mapping

### 8.1 NTU to MLIR Type Mapping

The middle end (Alex) emits only portable dialect types (`func`, `scf`, `arith`, `memref`, `index`) and commits to no target. Pointer-shaped NTU kinds map to the platform-sized `index` type, not to any target dialect: `NTUptr` maps to `index`. `NTUfnptr` is not pointer-shaped in the interior: a function value is a first-class `func` value (`func.constant`, consumed by `func.call_indirect`), and a closure is the two-value pair of [Backend Lowering Architecture §4](backend-lowering-architecture.md). Target realization is a backend-pathway concern: the LLVM pathway lowers `index` to `!llvm.ptr` where a raw address is needed and `func.constant` to `llvm.mlir.addressof` through its standard lowering; the CIRCT/FPGA pathway realizes each as a target-appropriate carrier. No cast carries a value across the tier boundary; `builtin.unrealized_conversion_cast` SHALL NOT appear in middle-end output.

| NTU Type | MLIR Type (x86_64) | MLIR Type (ARM32) |
|----------|-------------------|-------------------|
| `NTUint (Resolved Register)` | `i64` | `i32` |
| `NTUuint (Resolved Register)` | `i64` | `i32` |
| `NTUint (Resolved Pointer)` | `i64` | `i32` |
| `NTUuint (Resolved Pointer)` | `i64` | `i32` |
| `NTUint (Fixed 8)` | `i8` | `i8` |
| `NTUint (Fixed 16)` | `i16` | `i16` |
| `NTUint (Fixed 32)` | `i32` | `i32` |
| `NTUint (Fixed 64)` | `i64` | `i64` |
| `NTUfloat (Fixed 32)` | `f32` | `f32` |
| `NTUfloat (Fixed 64)` | `f64` | `f64` |
| `NTUptr` | `index` | `index` |
| `NTUfnptr` | `index` | `index` |
| `NTUsize` | `index` | `index` |
| `NTUdiff` | `index` | `index` |

The `index` type is platform-sized: it resolves to a 64-bit value on x86_64 and a 32-bit value on ARM32 (and on thumbv8m/M33), so a single portable mapping carries the correct pointer width per target without the NTU level naming any concrete byte count. On a target pathway that needs a raw address rather than an index (the LLVM CPU/MCU pathway), `index` is realized as `!llvm.ptr` during pathway lowering. That realization is lossy and belongs to the pathway, so it never appears in what the NTU or the middle end emits.

## 9. Conformance Requirements

### 9.1 CCS Requirements

1. **MUST** distinguish NTU type identity (`NTUint (Resolved Register)` vs `NTUint (Fixed 64)`)
2. **MUST NOT** assume platform-dependent type widths
3. **MUST** propagate NTU annotations through SemanticGraph
4. **MUST** reject operations between incompatible NTU types

### 9.2 Alex Requirements

1. **MUST** resolve NTU types using platform quotations
2. **MUST** generate platform-appropriate MLIR types
3. **MUST** use consistent resolution across all occurrences

## 10. Related Specifications

- [platform-predicates.md](platform-predicates.md) - Platform predicate semantics
- [native-type-mappings.md](native-type-mappings.md) - F# to native type mappings
- [native-type-universe.md](native-type-universe.md) - Complete type universe specification
