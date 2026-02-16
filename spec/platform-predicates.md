# Platform Predicates Specification

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-01-04

## 1. Overview

This chapter specifies the platform predicate system for Fidelity. Platform predicates are abstract propositions that enable conditional compilation without runtime checks, following the F* pattern of erased type-level assumptions.

## 2. Design Principles

### 2.1 F* Inspiration

F* uses predicates like `fits_u32` and `fits_u64` as abstract propositions:

```fstar
(* F* pattern *)
assume val fits_u32 : bool
assume val fits_u64 : bool

(* Width is erased - only affects type checking *)
let platform_word = if fits_u64 then U64 else U32
```

Fidelity adopts this pattern using F# quotations as the carrier mechanism.

### 2.2 Abstract During Checking, Resolved During Generation

| Phase | Predicate Treatment |
|-------|---------------------|
| CCS Type Checking | Abstract - values unknown |
| SemanticGraph | Carried as quotations |
| Alex Code Generation | Resolved from platform library |
| MLIR Output | Concrete - dead code eliminated |

## 3. Core Platform Predicates

### 3.1 Word Size Predicates

| Predicate | Type | Meaning |
|-----------|------|---------|
| `fits_u32` | `Expr<bool>` | Platform supports 32-bit word operations |
| `fits_u64` | `Expr<bool>` | Platform supports 64-bit word operations |

**Implication**: `fits_u64 ==> fits_u32`

A 64-bit platform always supports 32-bit operations.

### 3.2 Vector Extension Predicates

| Predicate | Type | Meaning |
|-----------|------|---------|
| `has_avx512` | `Expr<bool>` | Platform has AVX-512 vector support |
| `has_avx2` | `Expr<bool>` | Platform has AVX2 vector support |
| `has_sse42` | `Expr<bool>` | Platform has SSE4.2 support |
| `has_neon` | `Expr<bool>` | Platform has NEON vector support (ARM) |
| `has_sve` | `Expr<bool>` | Platform has SVE vector support (ARM) |
| `vector_width_max` | `Expr<int>` | Maximum vector width in bits |

**Implications**:
- `has_avx512 ==> has_avx2`
- `has_avx2 ==> has_sse42`

### 3.3 Atomics Predicates

| Predicate | Type | Meaning |
|-----------|------|---------|
| `has_atomics_32` | `Expr<bool>` | Platform has 32-bit atomic operations |
| `has_atomics_64` | `Expr<bool>` | Platform has 64-bit atomic operations |
| `has_atomics_128` | `Expr<bool>` | Platform has 128-bit CAS operations |

### 3.4 Memory Model Predicates

| Predicate | Type | Meaning |
|-----------|------|---------|
| `has_cache_coherent` | `Expr<bool>` | Platform has cache coherency |
| `has_mmio` | `Expr<bool>` | Platform supports memory-mapped I/O |
| `cache_line_size` | `Expr<int>` | Cache line size in bytes |

## 4. Platform Predicate Matrices

### 4.1 Desktop/Server Platforms

| Predicate | Linux_x86_64 | Linux_ARM64 | Windows_x86_64 | MacOS_ARM64 |
|-----------|--------------|-------------|----------------|-------------|
| fits_u32 | true | true | true | true |
| fits_u64 | true | true | true | true |
| has_avx512 | CPU-dep | false | CPU-dep | false |
| has_avx2 | true | false | true | false |
| has_neon | false | true | false | true |
| has_atomics_64 | true | true | true | true |
| has_atomics_128 | true | true | true | true |

### 4.2 Embedded Platforms

| Predicate | BareMetal_ARM32 | BareMetal_RISCV32 | BareMetal_Cortex_M0 |
|-----------|-----------------|-------------------|---------------------|
| fits_u32 | true | true | true |
| fits_u64 | false | false | false |
| has_neon | varies | false | false |
| has_atomics_32 | varies | varies | false |
| has_atomics_64 | false | false | false |
| has_mmio | true | true | true |

## 5. Predicate Declaration in Platform Libraries

### 5.1 Capabilities.fs Structure

```fsharp
// Fidelity.Platform/Linux_x86_64/Capabilities.fs
namespace Fidelity.Platform.Linux_x86_64

open FSharp.Quotations

module Capabilities =
    // Word size predicates
    let fits_u32: Expr<bool> = <@ true @>
    let fits_u64: Expr<bool> = <@ true @>
    
    // Vector extension predicates
    let has_avx512: Expr<bool> = <@ false @>  // CPU-dependent
    let has_avx2: Expr<bool> = <@ true @>     // Baseline for x86_64
    let has_sse42: Expr<bool> = <@ true @>
    let has_neon: Expr<bool> = <@ false @>    // ARM only
    let vector_width_max: Expr<int> = <@ 256 @>
    
    // Atomics predicates
    let has_atomics_32: Expr<bool> = <@ true @>
    let has_atomics_64: Expr<bool> = <@ true @>
    let has_atomics_128: Expr<bool> = <@ true @>
    
    // Memory model predicates
    let has_cache_coherent: Expr<bool> = <@ true @>
    let has_mmio: Expr<bool> = <@ true @>
    let cache_line_size: Expr<int> = <@ 64 @>
```

### 5.2 Dynamic CPU Detection (Optional)

For predicates that depend on runtime CPU features:

```fsharp
// Compile-time: Predicate is abstract
let has_avx512: Expr<bool> = <@ Runtime.cpuSupportsAVX512() @>

// Code generation: Alex may generate runtime check or
// use compile-time target selection
```

## 6. Using Predicates in Application Code

### 6.1 Conditional Compilation Pattern

```fsharp
// Application or library code
let vectorAdd (a: array<float>) (b: array<float>) : array<float> =
    if Platform.has_avx512 then
        vectorAdd_avx512 a b
    elif Platform.has_avx2 then
        vectorAdd_avx2 a b
    elif Platform.has_neon then
        vectorAdd_neon a b
    else
        vectorAdd_scalar a b
```

### 6.2 Dead Code Elimination

After Alex resolves predicates, unreachable branches are eliminated:

**Before (source):**
```fsharp
if Platform.has_avx512 then avx512_impl()
elif Platform.has_avx2 then avx2_impl()
else scalar_impl()
```

**After (on x86_64 without AVX-512):**
```mlir
// Only avx2_impl remains
call @vectorAdd_avx2(%a, %b) : (!llvm.ptr, !llvm.ptr) -> !llvm.ptr
```

## 7. Predicate Implications

### 7.1 Implication Rules

The following implications are normatively defined:

```
fits_u64 ==> fits_u32

has_avx512 ==> has_avx2
has_avx2 ==> has_sse42
has_sse42 ==> has_sse2

has_sve2 ==> has_sve
has_sve ==> has_neon

has_atomics_128 ==> has_atomics_64
has_atomics_64 ==> has_atomics_32
```

### 7.2 Implication Usage

CCS may use implications to simplify predicate checking:

```fsharp
// If fits_u64 is known true, fits_u32 need not be checked
if checkPredicate "fits_u64" ctx then
    // fits_u32 is implicitly true
```

## 8. CCS Handling of Predicates

### 8.1 Abstract Predicate Values

During type checking, predicates are abstract:

```fsharp
// CCS does NOT know the value of Platform.fits_u64
// It carries the predicate through unchanged
type PredicateContext = {
    Predicates: Map<string, Expr<bool>>
    Implications: (string * string) list
}
```

### 8.2 Predicate-Dependent Type Checking

Some type validity depends on predicates:

```fsharp
// Using int64 requires fits_u64
let x: int64 = 100L  // Valid only if fits_u64
```

CCS may emit warnings/errors for predicate-dependent types on constrained platforms.

## 9. Alex Resolution of Predicates

### 9.1 Quotation Evaluation

Alex evaluates predicate quotations to constant values:

```fsharp
let resolvePredicate (name: string) (ctx: PlatformContext) : bool =
    match ctx.Predicates.TryFind name with
    | Some expr -> evaluateConstBool expr
    | None -> 
        // Check implications
        resolveViaImplications name ctx
```

### 9.2 Branch Elimination

Resolved predicates enable branch elimination:

```mlir
// Before predicate resolution
scf.if %has_avx512 {
    call @avx512_impl()
} else {
    scf.if %has_avx2 {
        call @avx2_impl()
    } else {
        call @scalar_impl()
    }
}

// After resolution (has_avx512=false, has_avx2=true)
call @avx2_impl()
```

## 10. Conformance Requirements

### 10.1 Platform Library Requirements

1. **MUST** define all core predicates listed in Section 3
2. **MUST** use F# quotations as the carrier mechanism
3. **MUST** honor implication rules from Section 7
4. **SHOULD** document CPU-dependent predicates

### 10.2 CCS Requirements

1. **MUST** carry predicates through SemanticGraph unchanged
2. **MUST NOT** assume predicate values during type checking
3. **MAY** use implications for constraint simplification
4. **MAY** emit warnings for predicate-dependent constructs

### 10.3 Alex Requirements

1. **MUST** resolve all predicates before MLIR generation
2. **MUST** eliminate dead code based on resolved predicates
3. **MUST** use consistent resolution across compilation unit

## 11. Error Handling

### 11.1 Unknown Predicate

If a predicate is referenced but not defined:

```
Error CCS4001: Unknown platform predicate 'has_avx1024'
```

### 11.2 Predicate Conflict

If predicate values violate implications:

```
Error CCS4002: Platform defines has_avx512=true but has_avx2=false
               (has_avx512 implies has_avx2)
```

### 11.3 Missing Platform Definition

If platform library lacks required predicates:

```
Error CCS4003: Platform library missing required predicate 'fits_u32'
```

## 12. Related Specifications

- [ntu-types.md](ntu-types.md) - NTU type nomenclature
- [platform-bindings.md](platform-bindings.md) - Platform binding architecture
- [native-type-universe.md](native-type-universe.md) - Complete type universe
