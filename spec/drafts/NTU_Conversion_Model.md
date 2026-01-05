# NTU Conversion Model

**Status**: Draft → Candidate for Chapter 4 (Type System Extensions)
**Authors**: Fidelity Team
**Date**: January 2026
**Related**: Chapter 3 (Native Type Universe), SRTPResolution.fs

## Abstract

This document specifies the Native Type Universe (NTU) conversion model - how numeric and string conversions work in Fidelity. The design preserves F#'s simple syntax (`float 42`, `string x`) while leveraging SRTP for principled, type-safe resolution.

The model represents a deliberate departure from the .NET BCL pattern of explicit overload catalogs, instead following the principled approaches of F*, Rust, and ML while maintaining F# ergonomics.

## 1. Design Philosophy

### 1.1 Core Principles

1. **F# Syntax Preserved**: Users write standard F# conversion expressions
2. **SRTP Resolution**: Compile-time dispatch based on source type
3. **No Runtime Library**: Conversions are compiler intrinsics
4. **Type Safety**: Invalid conversions fail at compile time
5. **Internal Verbosity, External Simplicity**: F* naming internally, F# syntax externally

### 1.2 The Central Insight

F# already has the right syntax for conversions:
```fsharp
let x = float 42       // int → float
let s = string 3.14    // float → string
let n = int "42"       // string → int (parsing)
```

The challenge is making this work in a native context without BCL infrastructure. The solution: SRTP resolution determines the operation at compile time based on the source type.

## 2. Comparison with Other Systems

### 2.1 .NET BCL / F# Standard Library

**Pattern**: Explicit overload catalog

```csharp
// BCL System.Convert - N×M explicit implementations
public static int ToInt32(string value) { ... }
public static int ToInt32(double value) { ... }
public static int ToInt32(decimal value) { ... }
public static string ToString(int value) { ... }
public static string ToString(double value) { ... }
// ... hundreds of explicit overloads
```

**Pros**:
- Each conversion explicitly implemented
- Can have different semantics per pair

**Cons**:
- Combinatorial explosion (N source types × M target types)
- Runtime dispatch via method resolution
- Requires runtime library
- Difficult to extend

**Why NTU differs**: NTU uses SRTP to resolve conversions at compile time without enumerating every pair. The compiler generates the appropriate MLIR operation based on the type pair.

### 2.2 Rust

**Pattern**: Trait-based with strict safety rules

```rust
// From trait - only for infallible, lossless conversions
pub trait From<T>: Sized {
    fn from(value: T) -> Self;
}

// Widening conversions implemented
impl From<i8> for i16 { ... }   // OK: lossless
impl From<u8> for u32 { ... }   // OK: lossless

// Narrowing conversions NOT implemented via From
// impl From<u32> for u16 { ... }  // Would NOT compile - lossy

// TryFrom for fallible conversions
pub trait TryFrom<T>: Sized {
    type Error;
    fn try_from(value: T) -> Result<Self, Self::Error>;
}
```

**Pros**:
- Type system prevents accidental lossy conversions
- Clear distinction between infallible/fallible
- Blanket implementations enable generic code

**Cons**:
- Requires `.into()` or explicit `From::from()` calls
- Trait machinery adds complexity
- `as` cast bypasses safety for primitives

**What NTU adopts**:
- Type safety for conversions
- Clear categorization (widening vs narrowing)

**What NTU differs**:
- F# syntax preserved (`float x` not `x.into()`)
- SRTP instead of trait bounds
- All conversions compile; narrowing semantics are explicit

### 2.3 F* (Proof Assistant)

**Pattern**: Explicit naming with refinement proofs

```fstar
module FStar.Int.Cast

// Widening with refinement proof
val uint8_to_uint64: a:U8.t -> Tot (b:U64.t{U64.v b = U8.v a})

// Narrowing with modulo semantics proven
val uint32_to_uint8: a:U32.t -> Tot (b:U8.t{U8.v b = U32.v a % pow2 8})

// Unsafe narrowing marked with deprecated attribute
[@@(deprecated "with care; in C the result is implementation-defined")]
val int32_to_int8: a:I32.t -> Tot (b:I8.t {I8.v b = (I32.v a @% pow2 8)})
```

**Pros**:
- Semantics proven via refinement types
- Clear naming (`uint8_to_uint64`)
- Unsafe operations marked explicitly

**Cons**:
- Verbose user-facing syntax
- Requires dependent types for proofs
- Overkill for practical programming

**What NTU adopts**:
- F* naming convention *internally* (`int_to_float`)
- Category-based thinking (widening, narrowing)
- Conservative defaults

**What NTU differs**:
- F# syntax externally (`float 42`)
- No dependent types or proofs
- Pragmatic rather than formally verified

### 2.4 Standard ML / OCaml

**Pattern**: Explicit, separate functions

```ocaml
(* ML - separate conversion functions *)
let x = float_of_int 42
let s = string_of_int 42
let n = int_of_string "42"

(* Each function explicitly named *)
float_of_int : int -> float
int_of_float : float -> int
string_of_int : int -> string
int_of_string : string -> int  (* raises Failure on invalid input *)
```

**Pros**:
- Explicit, no magic
- Clear direction (from_to naming)
- Simple to understand

**Cons**:
- Verbose (`float_of_int` vs `float`)
- No polymorphism - each pair explicit
- Exceptions for parsing failures

**What NTU adopts**:
- Internal naming follows ML pattern
- Each conversion maps to specific operation

**What NTU differs**:
- F# overloaded syntax (`float` works on any numeric)
- SRTP provides polymorphism
- Option/Result for parsing instead of exceptions

### 2.5 C/C++

**Pattern**: Implicit coercion with explicit casts

```cpp
// C - implicit widening, explicit narrowing
int x = 42;
double d = x;        // Implicit: int → double
int y = (int)d;      // Explicit cast: double → int (truncates)
int z = d;           // Warning: implicit narrowing

// C++ - more control but complex
auto a = static_cast<int>(d);           // Numeric conversion
auto b = reinterpret_cast<int*>(ptr);   // Bit reinterpret
auto c = const_cast<int*>(const_ptr);   // Remove const
```

**Pros**:
- Minimal syntax for common cases
- Full control when needed

**Cons**:
- Implicit conversions cause bugs
- Cast syntax doesn't indicate semantics
- Type punning via casts is UB in C++

**What NTU explicitly rejects**:
- NO implicit conversions
- NO pointer/reference casts masquerading as conversions
- ALL conversions explicit in source

### 2.6 Summary Table

| System | Syntax | Resolution | Safety | Extensible |
|--------|--------|------------|--------|------------|
| BCL/.NET | Overloaded | Runtime | Medium | Hard |
| Rust | Trait `.into()` | Compile-time | High | Via traits |
| F* | `uint8_to_uint64` | Compile-time | Proven | Limited |
| ML/OCaml | `float_of_int` | Compile-time | High | Limited |
| C/C++ | Implicit + casts | Compile-time | Low | Via overloads |
| **NTU** | `float x` | **SRTP** | **High** | **Via SRTP** |

## 3. NTU Conversion Categories

### 3.1 Category 1: Numeric Widening (Always Safe)

Conversions where no precision is lost:

| Source | Target | MLIR Operation | Notes |
|--------|--------|----------------|-------|
| int8 → int16/32/64 | int | `arith.extsi` | Sign-extend |
| uint8 → uint16/32/64 | uint | `arith.extui` | Zero-extend |
| float32 → float64 | float | `arith.extf` | Precision increase |
| int → float | float | `arith.sitofp` | May lose precision for >2^53 |
| uint → float | float | `arith.uitofp` | May lose precision for >2^53 |

### 3.2 Category 2: Numeric Narrowing (May Truncate)

Conversions where data may be lost:

| Source | Target | MLIR Operation | Semantics |
|--------|--------|----------------|-----------|
| int64 → int32 | int32 | `arith.trunci` | High bits dropped |
| float → int | int | `arith.fptosi` | Truncate toward zero |
| float64 → float32 | float32 | `arith.truncf` | Precision loss |

### 3.3 Category 3: String Formatting (Any → String)

| Source | Target | Operation | Notes |
|--------|--------|-----------|-------|
| int → string | string | Intrinsic | Decimal digits |
| float → string | string | Intrinsic | Scientific/fixed |
| bool → string | string | Intrinsic | "true"/"false" |

### 3.4 Category 4: String Parsing (String → Specific)

| Source | Target | Return Type | Notes |
|--------|--------|-------------|-------|
| string → int | int option | May fail | Invalid input returns None |
| string → float | float option | May fail | Invalid input returns None |

## 4. SRTP Resolution Mechanism

### 4.1 Resolution Flow

When the compiler sees `float someValue`:

1. **Recognize**: `float` identified as conversion function
2. **Type Check**: Determine `someValue` type (e.g., `int`)
3. **Lookup**: SRTP resolves `(int, float)` conversion
4. **Generate**: Create `Intrinsic(Conversion, "int_to_float")`
5. **Emit**: Alex generates `arith.sitofp`

### 4.2 WitnessResolution Structure

```fsharp
type WitnessResolution = {
    Operator: string           // "float"
    ArgType: NativeType        // int
    ResolvedMember: string     // "int_to_float"
    ImplementingModule: ModulePath
    Kind: WitnessKind
}

type ConversionCategory =
    | Widening      // Safe, no data loss
    | Narrowing     // May truncate
    | CrossFamily   // Different representations
    | Formatting    // To string
    | Parsing       // From string
```

## 5. MLIR Emission

### 5.1 Numeric Conversions

```mlir
// int → float (widening)
%result = arith.sitofp %source : i64 to f64

// float → int (narrowing)
%result = arith.fptosi %source : f64 to i64

// int32 → int64 (widening)
%result = arith.extsi %source : i32 to i64

// int64 → int32 (narrowing)
%result = arith.trunci %source : i64 to i32
```

### 5.2 String Conversions

```mlir
// int → string (formatting intrinsic)
%result = call @__fidelity_int_to_string(%source)
    : (i64) -> !llvm.struct<(ptr, i64)>

// string → int (parsing intrinsic, returns option)
%result = call @__fidelity_string_to_int(%source)
    : (!llvm.struct<(ptr, i64)>) -> !llvm.struct<(i1, i64)>
```

## 6. Implementation Status

### 6.1 Completed

- [x] `ConversionCategory` type in SRTPResolution.fs
- [x] `conversionFunctions` set of function names
- [x] `isConversionFunction` predicate
- [x] `categorizeConversion` based on source/target types
- [x] `getConversionMLIROp` for MLIR operation selection
- [x] `resolveConversion` returning WitnessResolution
- [x] Integration into `tryResolve` chain
- [x] Integration into `isSRTPOperator`

### 6.2 Pending

- [ ] Alex witnesses for Conversion intrinsics
- [ ] String formatting intrinsics implementation
- [ ] String parsing intrinsics implementation
- [ ] Sample 05 demonstrating interactive conversions

## 7. Design Decisions and Rationale

### 7.1 Why SRTP over Overloads?

**Decision**: Use SRTP resolution instead of explicit overload catalog.

**Rationale**:
- Avoids N×M combinatorial explosion
- Compile-time resolution - no runtime dispatch
- Extensible via SRTP witness table
- Aligns with F# language philosophy

### 7.2 Why Preserve F# Syntax?

**Decision**: Keep `float x` syntax rather than `float_of_int x`.

**Rationale**:
- F# developers expect this syntax
- Less cognitive overhead
- Internal naming provides clarity for compiler developers
- Best of both worlds: ergonomic externally, explicit internally

### 7.3 Why No Implicit Conversions?

**Decision**: All conversions require explicit syntax.

**Rationale**:
- Follows Rust/ML safety principles
- Prevents accidental precision loss
- Makes conversion sites visible in code
- Aligns with "explicit is better than implicit"

### 7.4 Why Options for Parsing?

**Decision**: String parsing returns `option` types.

**Rationale**:
- Parsing can fail at runtime
- Options are idiomatic F#
- No exceptions in native code
- Type signature documents fallibility

## 8. Future Considerations

### 8.1 Checked Narrowing

Could add checked narrowing that fails on overflow:
```fsharp
let x = int32.checked 999999999999L  // Returns int32 option
```

### 8.2 Custom Type Conversions

SRTP mechanism could be extended for user-defined types:
```fsharp
type MyType with
    static member op_Explicit(x: int) : MyType = ...
```

### 8.3 Locale-Aware Formatting

For internationalization, could add:
```fsharp
String.formatWith culture value
```

## 9. References

- F* Int.Cast module: `~/repos/fstar/ulib/FStar.Int.Cast.fst`
- Rust std::convert: `https://doc.rust-lang.org/std/convert/`
- MLIR arith dialect: `https://mlir.llvm.org/docs/Dialects/ArithOps/`
- fsnative-spec Chapter 3: Native Type Universe
- Serena memory: `ntu_conversion_architecture`
