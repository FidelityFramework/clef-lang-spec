# NTU Conversion Model

> **Status**: Normative
> **Last Updated**: 2026-01-19

## Informative References

> **Design Rationale**: For comparison with other type conversion systems (Rust traits, F* refinements, ML explicit functions, C/C++ casts) and the design philosophy behind NTU conversions, see the rationale documentation.
>
> **Related Spec**: This chapter complements [Convert Intrinsic Module](intrinsics-convert.md) which specifies the intrinsic function signatures.

---

## 1. Overview

This chapter specifies the Native Type Universe (NTU) conversion model—how numeric and string conversions are resolved and emitted in Clef compilation.

Clef preserves standard F# conversion syntax (`float 42`, `string x`, `int "42"`) while using SRTP (Statically Resolved Type Parameters) for compile-time resolution to native operations.

## 2. Conversion Categories

### 2.1 Category 1: Numeric Widening

Conversions where no precision is lost:

| Source | Target | MLIR Operation | Notes |
|--------|--------|----------------|-------|
| int8 → int16/32/64 | Signed integer | `arith.extsi` | Sign-extend |
| uint8 → uint16/32/64 | Unsigned integer | `arith.extui` | Zero-extend |
| float32 → float64 | Float | `arith.extf` | Precision increase |
| int → float | Float | `arith.sitofp` | Signed to float |
| uint → float | Float | `arith.uitofp` | Unsigned to float |

> **Note**: Integer to float conversions may lose precision for values exceeding 2^53.

### 2.2 Category 2: Numeric Narrowing

Conversions where data may be lost:

| Source | Target | MLIR Operation | Semantics |
|--------|--------|----------------|-----------|
| int64 → int32/16/8 | Smaller integer | `arith.trunci` | High bits dropped |
| float → int | Integer | `arith.fptosi` / `arith.fptoui` | Truncate toward zero |
| float64 → float32 | float32 | `arith.truncf` | Precision loss |

### 2.3 Category 3: String Formatting

| Source | Target | Operation | Notes |
|--------|--------|-----------|-------|
| int → string | string | `@__fidelity_int_to_string` | Decimal representation |
| float → string | string | `@__fidelity_float_to_string` | Scientific/fixed |
| bool → string | string | `@__fidelity_bool_to_string` | "true" / "false" |

### 2.4 Category 4: String Parsing

| Source | Target | Return Type | Notes |
|--------|--------|-------------|-------|
| string → int | `int option` | May fail | Invalid input returns `None` |
| string → float | `float option` | May fail | Invalid input returns `None` |

## 3. SRTP Resolution Mechanism

### 3.1 Resolution Flow

When the compiler encounters a conversion expression like `float someValue`:

1. **Recognition**: `float` identified as conversion function
2. **Type Inference**: Determine type of `someValue` (e.g., `int`)
3. **SRTP Lookup**: Resolve `(int, float)` conversion pair
4. **Intrinsic Generation**: Create `Intrinsic(Conversion, "int_to_float")`
5. **MLIR Emission**: Generate appropriate `arith.*` operation

### 3.2 Resolution Types

```fsharp
type ConversionCategory =
    | Widening      // Safe, no data loss possible
    | Narrowing     // May truncate or lose precision
    | CrossFamily   // Different type families (int ↔ float)
    | Formatting    // To string representation
    | Parsing       // From string representation

type WitnessResolution = {
    Operator: string           // e.g., "float"
    ArgType: NativeType        // Source type
    ResultType: NativeType     // Target type
    Category: ConversionCategory
    ResolvedMember: string     // e.g., "int_to_float"
}
```

### 3.3 Conversion Function Recognition

The following identifiers are recognized as conversion functions:

```
int, int8, int16, int32, int64
uint, uint8, uint16, uint32, uint64, byte, sbyte
float, float32, float64, single, double
char, string
```

## 4. MLIR Emission

### 4.1 Numeric Conversions

```mlir
// int → float (widening)
%result = arith.sitofp %source : i64 to f64

// float → int (narrowing)
%result = arith.fptosi %source : f64 to i64

// int32 → int64 (widening)
%result = arith.extsi %source : i32 to i64

// int64 → int32 (narrowing)
%result = arith.trunci %source : i64 to i32

// float64 → float32 (narrowing)
%result = arith.truncf %source : f64 to f32
```

### 4.2 String Conversions

```mlir
// int → string (formatting intrinsic)
%result = call @__fidelity_int_to_string(%source)
    : (i64) -> !llvm.struct<(ptr, i64)>

// string → int (parsing intrinsic, returns option)
%result = call @__fidelity_string_to_int(%source)
    : (!llvm.struct<(ptr, i64)>) -> !llvm.struct<(i1, i64)>
```

### 4.3 Identity Conversions

When source and target types have the same bit width, no MLIR operation is emitted. The input value passes through unchanged.

> **Note**: MLIR does not permit identity conversions like `arith.extsi %x : i32 to i32`. Alex witnesses SHALL detect identity cases and return the input SSA value directly.

## 5. Overflow and Precision Behavior

### 5.1 Integer Truncation

Narrowing integer conversions drop high-order bits without error:

```fsharp
int8 256    // Result: 0 (high bits dropped)
int8 -129   // Result: 127 (two's complement truncation)
```

### 5.2 Float to Integer

Float-to-integer conversion truncates toward zero:

```fsharp
int 3.7     // Result: 3
int -3.7    // Result: -3
```

Overflow (value outside integer range) produces undefined results.

### 5.3 Float Precision

Float64 to Float32 conversion may lose precision:

```fsharp
float32 3.141592653589793   // Result: 3.1415927 (precision loss)
```

## 6. Normative Requirements

1. **SRTP Resolution**: All standard F# conversion functions SHALL be resolved via SRTP at compile time
2. **No Runtime Dispatch**: Conversions SHALL NOT require runtime type inspection or method dispatch
3. **Category Preservation**: The conversion category (widening, narrowing, etc.) SHALL be determined by source and target types
4. **Identity Optimization**: Identity conversions (same bit width) SHALL emit no MLIR operation
5. **Signed/Unsigned Distinction**: Signed types SHALL use `extsi`/`fptosi`/`sitofp`; unsigned types SHALL use `extui`/`fptoui`/`uitofp`
6. **String Parsing Fallibility**: String-to-numeric conversions SHALL return `option` types to represent parse failure
7. **No Implicit Conversions**: All type conversions SHALL require explicit conversion syntax in source code

## See Also

- [Convert Intrinsic Module](intrinsics-convert.md) - Intrinsic function signatures
- [Native Type Universe](native-type-universe.md) - NTU type system
- [Native Type Mappings](native-type-mappings.md) - Type-to-layout mapping
