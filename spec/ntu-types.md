# NTU Type Nomenclature Specification

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-01-04

## 1. Overview

This chapter specifies the NTU (Native Type Universe) nomenclature used internally by FNCS for platform-generic types. NTU types resolve via quotation-based platform bindings, following the F* pattern where type WIDTH is an erased assumption.

### 1.1 Core Principle

**Platform awareness flows FROM THE TOP via quotation-based binding libraries, not from FNCS type inference.**

FNCS validates type **identity** (NTUint vs NTUint64). Alex witnesses platform quotations to determine type **width** (32-bit vs 64-bit).

## 2. NTU Type Categories

### 2.1 Platform-Dependent Types (Quotation-Resolved)

| NTU Type | Semantic Meaning | Resolution |
|----------|------------------|------------|
| `NTUint` | Platform word, signed | Platform quotation |
| `NTUuint` | Platform word, unsigned | Platform quotation |
| `NTUnint` | Native int (pointer-sized, signed) | Platform quotation |
| `NTUunint` | Native uint (pointer-sized, unsigned) | Platform quotation |
| `NTUptr<'T>` | Native pointer to type T | Platform quotation |
| `NTUsize` | Size type (`size_t` equivalent) | Platform quotation |
| `NTUdiff` | Pointer difference (`ptrdiff_t` equivalent) | Platform quotation |

### 2.2 Fixed-Width Types (Platform-Independent)

| NTU Type | Bit Width | MLIR Type |
|----------|-----------|-----------|
| `NTUint8` | 8-bit signed | `i8` |
| `NTUint16` | 16-bit signed | `i16` |
| `NTUint32` | 32-bit signed | `i32` |
| `NTUint64` | 64-bit signed | `i64` |
| `NTUuint8` | 8-bit unsigned | `ui8` |
| `NTUuint16` | 16-bit unsigned | `ui16` |
| `NTUuint32` | 32-bit unsigned | `ui32` |
| `NTUuint64` | 64-bit unsigned | `ui64` |
| `NTUfloat32` | 32-bit float | `f32` |
| `NTUfloat64` | 64-bit float | `f64` |

## 3. Type Identity and Type Width

### 3.1 Type Identity (FNCS Responsibility)

FNCS enforces type identity constraints:

```
NTUint ≠ NTUint32    // Different types
NTUint ≠ NTUint64    // Different types
NTUint = NTUint      // Same type

// Valid
let add (x: NTUint) (y: NTUint) : NTUint = x + y

// Type Error
let invalid (x: NTUint) (y: NTUint64) = x + y
```

### 3.2 Type Width (Alex Responsibility)

Alex resolves type width via platform quotations:

| NTU Type | x86_64 Width | ARM32 Width | ARM64 Width |
|----------|--------------|-------------|-------------|
| NTUint | 64 bits | 32 bits | 64 bits |
| NTUuint | 64 bits | 32 bits | 64 bits |
| NTUptr<_> | 64 bits | 32 bits | 64 bits |
| NTUsize | 64 bits | 32 bits | 64 bits |
| NTUdiff | 64 bits | 32 bits | 64 bits |

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

### 4.1 Option B: NTU Internal with Semantic Aliases

The architecture uses a three-tier exposure model:

| F# Source | Semantic Alias | FNCS Internal |
|-----------|----------------|---------------|
| `int` | (implicit) | `NTUint` |
| `uint` | (implicit) | `NTUuint` |
| `platformint` | `platformint` | `NTUint` |
| `platformuint` | `platformuint` | `NTUuint` |
| `platformsize` | `platformsize` | `NTUsize` |
| `int32` | (implicit) | `NTUint32` |
| `int64` | (implicit) | `NTUint64` |
| `nativeint` | (implicit) | `NTUnint` |
| `nativeptr<'T>` | (implicit) | `NTUptr<'T>` |

### 4.2 Developer Experience

**Level 1 (Default)**: Developers use standard F# type names.
```fsharp
let x: int = 42  // FNCS sees NTUint
let arr: array<int> = [| 1; 2; 3 |]
```

**Level 2/3 (Explicit)**: Developers use semantic aliases for clarity.
```fsharp
let write (fd: platformint) (buf: nativeptr<byte>) (count: platformsize) : platformint =
    Platform.Bindings.write fd buf count
```

## 5. NTUKind Implementation

### 5.1 Discriminated Union Definition

```fsharp
/// NTU (Native Type Universe) type kinds
[<RequireQualifiedAccess>]
type NTUKind =
    // Platform-dependent (resolved via quotations)
    | NTUint      // Platform word, signed
    | NTUuint     // Platform word, unsigned
    | NTUnint     // Native int (pointer-sized signed)
    | NTUunint    // Native uint (pointer-sized unsigned)
    | NTUptr      // Pointer to type
    | NTUsize     // size_t equivalent
    | NTUdiff     // ptrdiff_t equivalent
    
    // Fixed width (platform-independent)
    | NTUint8
    | NTUint16
    | NTUint32
    | NTUint64
    | NTUuint8
    | NTUuint16
    | NTUuint32
    | NTUuint64
    | NTUfloat32
    | NTUfloat64
```

### 5.2 NTULayout Record

```fsharp
/// Platform-resolved type layout (erased at runtime)
type NTULayout = {
    /// The NTU kind
    Kind: NTUKind
    
    /// Assumed size in bytes (erased)
    AssumedSize: int option
    
    /// Assumed alignment in bytes (erased)
    AssumedAlignment: int option
}

module NTULayout =
    /// Create layout for platform-dependent type
    let platformDependent kind = { Kind = kind; AssumedSize = None; AssumedAlignment = None }
    
    /// Create layout for fixed-width type
    let fixed kind size align = { Kind = kind; AssumedSize = Some size; AssumedAlignment = Some align }
    
    /// Standard NTU layouts
    let ntuInt = platformDependent NTUKind.NTUint
    let ntuInt32 = fixed NTUKind.NTUint32 4 4
    let ntuInt64 = fixed NTUKind.NTUint64 8 8
```

## 6. Unification Rules

### 6.1 NTU Types Unify Only with Themselves

```fsharp
// Valid unification
unify(NTUint, NTUint) = Success

// Invalid unification - different NTU kinds
unify(NTUint, NTUint64) = Error(TypeMismatch)
unify(NTUint, NTUint32) = Error(TypeMismatch)

// Pointer types - element types must unify
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
FNCS: Maps to NTUint
    ↓
SemanticGraph: Carries NTUint annotation
    ↓
Alex: Reads platform quotation
    ↓
MLIR: i64 (on x86_64) or i32 (on ARM32)
```

### 7.2 Platform Quotation Structure

```fsharp
// From Fidelity.Platform library
type NTUResolutions = {
    NTUint: int    // Size in bytes
    NTUuint: int
    NTUptr: int
    NTUsize: int
    NTUdiff: int
}

let linux_x86_64: Expr<NTUResolutions> = <@
    { NTUint = 8; NTUuint = 8; NTUptr = 8; NTUsize = 8; NTUdiff = 8 }
@>

let linux_arm32: Expr<NTUResolutions> = <@
    { NTUint = 4; NTUuint = 4; NTUptr = 4; NTUsize = 4; NTUdiff = 4 }
@>
```

## 8. MLIR Mapping

### 8.1 NTU to MLIR Type Mapping

| NTU Type | MLIR Type (x86_64) | MLIR Type (ARM32) |
|----------|-------------------|-------------------|
| NTUint | `i64` | `i32` |
| NTUuint | `i64` | `i32` |
| NTUnint | `i64` | `i32` |
| NTUunint | `i64` | `i32` |
| NTUptr<_> | `!llvm.ptr` | `!llvm.ptr` |
| NTUsize | `i64` | `i32` |
| NTUdiff | `i64` | `i32` |
| NTUint32 | `i32` | `i32` |
| NTUint64 | `i64` | `i64` |
| NTUfloat32 | `f32` | `f32` |
| NTUfloat64 | `f64` | `f64` |

## 9. Conformance Requirements

### 9.1 FNCS Requirements

1. **MUST** distinguish NTU type identity (NTUint vs NTUint64)
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
