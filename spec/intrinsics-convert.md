# Convert Intrinsic Module Specification

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-01-08

## 1. Overview

The Convert module provides type conversion intrinsics for transforming values between numeric types. These are the F# standard conversion functions (`int`, `float`, `byte`, etc.) with native semantics.

All Convert intrinsics are **pure functions** with no side effects.

## 2. Module Definition

```fsharp
module Convert
```

Convert intrinsics are accessed via standard F# conversion function syntax:

```fsharp
let x = int 3.14      // float -> int
let y = float 42      // int -> float
let z = byte 255      // int -> byte
```

## 3. Conversion Operations

### 3.1 Integer Conversions

#### toInt (int, int32)

```fsharp
val int : 'T -> int
val int32 : 'T -> int32
```

**Semantics:**
- From float/float32: Truncates toward zero (`arith.fptosi`)
- From int64: Truncates to 32 bits (`arith.trunci`)
- From int8/int16: Sign-extends (`arith.extsi`)
- From int32: Identity (no operation)

**MLIR Generation:**
```mlir
// float64 -> int32
%result = arith.fptosi %input : f64 to i32

// int64 -> int32
%result = arith.trunci %input : i64 to i32

// int16 -> int32
%result = arith.extsi %input : i16 to i32
```

#### toInt64 (int64)

```fsharp
val int64 : 'T -> int64
```

**Semantics:**
- From float/float32: Truncates toward zero (`arith.fptosi`)
- From int8/int16/int32: Sign-extends (`arith.extsi`)
- From int64: Identity

#### toInt16 (int16)

```fsharp
val int16 : 'T -> int16
```

**Semantics:**
- From float/float32: Truncates toward zero (`arith.fptosi`)
- From int32/int64: Truncates (`arith.trunci`)
- From int8: Sign-extends (`arith.extsi`)
- From int16: Identity

#### toSByte (sbyte, int8)

```fsharp
val sbyte : 'T -> sbyte
val int8 : 'T -> int8
```

**Semantics:**
- From float/float32: Truncates toward zero (`arith.fptosi`)
- From int16/int32/int64: Truncates (`arith.trunci`)
- From int8: Identity

### 3.2 Unsigned Integer Conversions

#### toUInt32 (uint32)

```fsharp
val uint32 : 'T -> uint32
```

**Semantics:**
- From float/float32: Truncates toward zero, unsigned (`arith.fptoui`)
- From uint64/int64: Truncates (`arith.trunci`)
- From uint8/uint16/int8/int16: Zero-extends (`arith.extui`)
- From uint32/int32: Identity (same bit width)

#### toUInt64 (uint64)

```fsharp
val uint64 : 'T -> uint64
```

**Semantics:**
- From float/float32: Truncates toward zero, unsigned (`arith.fptoui`)
- From uint8/uint16/uint32/int8/int16/int32: Zero-extends (`arith.extui`)
- From uint64/int64: Identity

#### toUInt16 (uint16)

```fsharp
val uint16 : 'T -> uint16
```

**Semantics:**
- From float/float32: Truncates toward zero, unsigned (`arith.fptoui`)
- From uint32/uint64/int32/int64: Truncates (`arith.trunci`)
- From uint8/int8: Zero-extends (`arith.extui`)
- From uint16/int16: Identity

#### toByte (byte, uint8)

```fsharp
val byte : 'T -> byte
val uint8 : 'T -> uint8
```

**Semantics:**
- From float/float32: Truncates toward zero, unsigned (`arith.fptoui`)
- From uint16/uint32/uint64/int16/int32/int64: Truncates (`arith.trunci`)
- From uint8/int8: Identity

### 3.3 Floating-Point Conversions

#### toFloat (float, float64, double)

```fsharp
val float : 'T -> float
val float64 : 'T -> float64
val double : 'T -> float
```

**Semantics:**
- From int8/int16/int32/int64: Signed int to float (`arith.sitofp`)
- From uint8/uint16/uint32/uint64: Unsigned int to float (`arith.uitofp`)
- From float32: Extends precision (`arith.extf`)
- From float64: Identity

**MLIR Generation:**
```mlir
// int32 -> float64
%result = arith.sitofp %input : i32 to f64

// float32 -> float64
%result = arith.extf %input : f32 to f64
```

#### toFloat32 (float32, single)

```fsharp
val float32 : 'T -> float32
val single : 'T -> float32
```

**Semantics:**
- From int8/int16/int32/int64: Signed int to float (`arith.sitofp`)
- From uint8/uint16/uint32/uint64: Unsigned int to float (`arith.uitofp`)
- From float64: Truncates precision (`arith.truncf`)
- From float32: Identity

### 3.4 Character Conversion

#### toChar (char)

```fsharp
val char : 'T -> char
```

**Semantics:**
- Target type is `i32` (Unicode code point)
- From int8/int16/uint8/uint16: Zero-extends (`arith.extui`) - char is unsigned
- From int64/uint64: Truncates (`arith.trunci`)
- From int32/uint32: Identity

## 4. IntrinsicModule Classification

All conversion operations are classified as:

```fsharp
IntrinsicModule.Convert
IntrinsicCategory.Conversion
```

## 5. Type Signatures Summary

| Function | Type Signature | FNCS Operation |
|----------|----------------|----------------|
| `int`, `int32` | `'T -> int` | `toInt` |
| `int64` | `'T -> int64` | `toInt64` |
| `int16` | `'T -> int16` | `toInt16` |
| `sbyte`, `int8` | `'T -> sbyte` | `toSByte` |
| `byte`, `uint8` | `'T -> byte` | `toByte` |
| `uint16` | `'T -> uint16` | `toUInt16` |
| `uint32` | `'T -> uint32` | `toUInt32` |
| `uint64` | `'T -> uint64` | `toUInt64` |
| `float`, `float64`, `double` | `'T -> float` | `toFloat` |
| `float32`, `single` | `'T -> float32` | `toFloat32` |
| `char` | `'T -> char` | `toChar` |

## 6. Identity Conversions

When source and target types are the same bit width, no MLIR operation is emitted. The input value passes through unchanged.

**Important**: MLIR does not allow identity conversions like `arith.extsi %x : i32 to i32`. Alex witnesses must detect identity cases and return the input SSA value directly.

## 7. Overflow Behavior

Truncating conversions (e.g., `int64 -> int32`, `float -> int`) may lose information:

- **Integer truncation**: High bits are discarded
- **Float to int**: Value is truncated toward zero; overflow produces undefined results
- **Float precision loss**: `float64 -> float32` may lose precision

FNCS does not provide checked conversions. For bounds checking, use explicit comparison before conversion.

## 8. Relationship to Other Intrinsics

| Module | Relationship |
|--------|--------------|
| `Bits` | Bit casting (reinterpret bits, no conversion): `float32ToInt32Bits` |
| `Convert` | Type conversion (change representation): `int 3.14f` |

**Distinction**: `Bits.float32ToInt32Bits 1.0f` returns `0x3F800000` (IEEE 754 bits). `int 1.0f` returns `1` (converted value).

## 9. Normative Requirements

1. **FNCS SHALL** classify all standard F# conversion functions as `Convert` intrinsics
2. **FNCS SHALL** type conversion functions as `'T -> TargetType` (polymorphic input)
3. **Alex SHALL** generate appropriate MLIR conversion operations based on source type
4. **Alex SHALL** handle identity conversions by passing through the input value
5. **Alex SHALL** use signed operations (`extsi`, `fptosi`, `sitofp`) for signed types
6. **Alex SHALL** use unsigned operations (`extui`, `fptoui`, `uitofp`) for unsigned types
