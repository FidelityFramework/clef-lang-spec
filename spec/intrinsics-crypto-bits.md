# Crypto and Bits Intrinsic Modules Specification

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-01-08

## 1. Overview

This chapter specifies two new intrinsic modules for CCS:

1. **Crypto** - Cryptographic operations (SHA-1, Base64 encoding/decoding)
2. **Bits** - Bit manipulation and byte order operations

These intrinsics support the WREN stack's WebSocket communication layer, which requires:
- SHA-1 hashing for WebSocket handshake (RFC 6455)
- Base64 encoding for WebSocket accept key generation
- Byte order conversion for network protocol handling
- Bit casting for binary serialization (BAREWire)

## 2. Crypto Module

### 2.1 Module Definition

```fsharp
module Crypto
```

The Crypto module provides cryptographic primitives. All operations are pure functions with no side effects.

### 2.2 SHA-1 Hash

```fsharp
val sha1 : byte[] -> byte[]
```

**Semantics:**
- Input: Arbitrary byte array
- Output: 20-byte (160-bit) SHA-1 digest
- Follows FIPS 180-4 specification
- Output array is always exactly 20 bytes

**Alex Witness Implementation:**
- IntrinsicWitness pattern matches `SemanticKind.Intrinsic(Crypto, "sha1")`
- Witness generates inline LLVM IR for SHA-1 algorithm
- Alternatively, witness emits external function declaration for platform crypto

**Example:**
```fsharp
let hash = Crypto.sha1 data  // hash.Length = 20
```

### 2.3 Base64 Encoding

```fsharp
val base64Encode : byte[] -> string
```

**Semantics:**
- Input: Arbitrary byte array
- Output: Base64-encoded string (RFC 4648)
- Uses standard alphabet (A-Z, a-z, 0-9, +, /)
- Includes padding (=) as required

**Output Length:**
- `ceil(inputLength / 3) * 4` characters

**Example:**
```fsharp
let encoded = Crypto.base64Encode [| 72uy; 101uy; 108uy; 108uy; 111uy |]
// encoded = "SGVsbG8="
```

### 2.4 Base64 Decoding

```fsharp
val base64Decode : string -> byte[]
```

**Semantics:**
- Input: Base64-encoded string (RFC 4648)
- Output: Decoded byte array
- Ignores whitespace in input
- Handles missing padding gracefully

**Error Behavior:**
- Invalid characters: Returns empty array (or runtime error in debug mode)

**Example:**
```fsharp
let decoded = Crypto.base64Decode "SGVsbG8="
// decoded = [| 72uy; 101uy; 108uy; 108uy; 111uy |]
```

## 3. Bits Module

### 3.1 Module Definition

```fsharp
module Bits
```

The Bits module provides bit manipulation and byte order operations. All operations are pure and map directly to LLVM intrinsics or inline operations.

### 3.2 Byte Order Conversion

Network protocols use big-endian (network byte order). These intrinsics convert between host and network byte order.

#### 3.2.1 Host to Network (16-bit)

```fsharp
val htons : uint16 -> uint16
```

**Semantics:**
- Converts 16-bit value from host byte order to network byte order (big-endian)
- On big-endian platforms: no-op
- On little-endian platforms: byte swap

**Alex Witness Implementation:**
- IntrinsicWitness pattern matches `SemanticKind.Intrinsic(Bits, "htons")`
- Witness queries platform quotation for byte order
- Little-endian platforms: emits `llvm.intr.bswap`
- Big-endian platforms: emits passthrough

```mlir
// Little-endian (x86_64, ARM64 LE):
%swapped = llvm.intr.bswap(%value) : i16
// Big-endian: passthrough
```

#### 3.2.2 Network to Host (16-bit)

```fsharp
val ntohs : uint16 -> uint16
```

**Semantics:**
- Converts 16-bit value from network byte order to host byte order
- Symmetric with `htons`

#### 3.2.3 Host to Network (32-bit)

```fsharp
val htonl : uint32 -> uint32
```

**Semantics:**
- Converts 32-bit value from host byte order to network byte order

#### 3.2.4 Network to Host (32-bit)

```fsharp
val ntohl : uint32 -> uint32
```

**Semantics:**
- Converts 32-bit value from network byte order to host byte order
- Symmetric with `htonl`

### 3.3 Bit Casting (Reinterpret)

Bit casting reinterprets the bit pattern of a value as a different type. No conversion occurs; the bits are preserved exactly.

#### 3.3.1 Float32 to Int32

```fsharp
val float32ToInt32Bits : float32 -> int32
```

**Semantics:**
- Reinterprets 32-bit float as 32-bit signed integer
- IEEE 754 representation preserved
- Example: `1.0f` -> `0x3F800000` (1065353216)

**Alex Witness Implementation:**
- IntrinsicWitness pattern matches `SemanticKind.Intrinsic(Bits, "float32ToInt32Bits")`
- Witness emits LLVM bitcast instruction

```mlir
%result = llvm.bitcast %value : f32 to i32
```

#### 3.3.2 Int32 to Float32

```fsharp
val int32BitsToFloat32 : int32 -> float32
```

**Semantics:**
- Reinterprets 32-bit signed integer as 32-bit float
- Inverse of `float32ToInt32Bits`

#### 3.3.3 Float64 to Int64

```fsharp
val float64ToInt64Bits : float -> int64
```

**Semantics:**
- Reinterprets 64-bit float as 64-bit signed integer
- IEEE 754 representation preserved

#### 3.3.4 Int64 to Float64

```fsharp
val int64BitsToFloat64 : int64 -> float
```

**Semantics:**
- Reinterprets 64-bit signed integer as 64-bit float
- Inverse of `float64ToInt64Bits`

## 4. IntrinsicModule Enumeration

Add the following variants to `IntrinsicModule`:

```fsharp
type IntrinsicModule =
    // ... existing variants ...
    | Crypto        // Cryptographic operations
    | Bits          // Bit manipulation and byte order
```

## 5. IntrinsicCategory Classification

| Intrinsic | Category |
|-----------|----------|
| `Crypto.sha1` | `Pure` |
| `Crypto.base64Encode` | `Pure` |
| `Crypto.base64Decode` | `Pure` |
| `Bits.htons` | `Pure` |
| `Bits.ntohs` | `Pure` |
| `Bits.htonl` | `Pure` |
| `Bits.ntohl` | `Pure` |
| `Bits.float32ToInt32Bits` | `Pure` |
| `Bits.int32BitsToFloat32` | `Pure` |
| `Bits.float64ToInt64Bits` | `Pure` |
| `Bits.int64BitsToFloat64` | `Pure` |

All operations are `Pure` category - no side effects, deterministic output.

## 6. Type Signatures Summary

| Intrinsic | Type Signature |
|-----------|----------------|
| `Crypto.sha1` | `byte[] -> byte[]` |
| `Crypto.base64Encode` | `byte[] -> string` |
| `Crypto.base64Decode` | `string -> byte[]` |
| `Bits.htons` | `uint16 -> uint16` |
| `Bits.ntohs` | `uint16 -> uint16` |
| `Bits.htonl` | `uint32 -> uint32` |
| `Bits.ntohl` | `uint32 -> uint32` |
| `Bits.float32ToInt32Bits` | `float32 -> int32` |
| `Bits.int32BitsToFloat32` | `int32 -> float32` |
| `Bits.float64ToInt64Bits` | `float -> int64` |
| `Bits.int64BitsToFloat64` | `int64 -> float` |

## 7. Platform Considerations

### 7.1 Byte Order Detection

CCS does not need to know the platform byte order. Alex resolves this via platform quotations:

```fsharp
// Platform.fs quotation
let byteOrder: Expr<Endianness> = <@ Endianness.LittleEndian @>
```

Alex generates appropriate code based on the platform:
- Little-endian: emit bswap instruction
- Big-endian: emit passthrough

### 7.2 Crypto Witness Implementation Options

The IntrinsicWitness for Crypto operations has two implementation strategies:

1. **Inline witness** (preferred for freestanding):
   - Witness generates pure LLVM IR implementing SHA-1/Base64 algorithms
   - No external dependencies
   - Larger binary size
   - Complete self-containment

2. **External witness** (optional for console/desktop):
   - Witness emits `llvm.func` declaration for platform crypto
   - Links against libcrypto (OpenSSL) or platform equivalent
   - Smaller binary
   - External dependency

The choice is made via `.fidproj` configuration and flows through platform quotations:

```toml
[compilation]
crypto_implementation = "inline"  # or "platform"
```

The witness queries this setting via platform context during MLIR generation.

## 8. Nanopass and Witness Flow

The pipeline for Crypto and Bits intrinsics follows the standard CCS→Alex flow:

```
F# Source: Crypto.sha1 data
    ↓
CCS Type Checking (Expressions/Intrinsics.fs, Expressions/Coordinator.fs)
    - Coordinator dispatches to Intrinsics module for intrinsic resolution
    - Recognizes "Crypto.sha1" pattern
    - Creates IntrinsicInfo { Module=Crypto, Operation="sha1", Category=Pure }
    - Assigns type: byte[] -> byte[]
    - Creates PSG node with SemanticKind.Intrinsic(info)
    ↓
PSG Construction (complete graph)
    ↓
Reachability Analysis (narrows graph)
    ↓
Enrichment Nanopasses (def-use edges, etc.)
    ↓
Alex/Zipper Traversal
    - Zipper provides "attention" at each node
    - XParsec pattern matches SemanticKind.Intrinsic
    ↓
IntrinsicWitness
    - Pattern matches on IntrinsicModule (Crypto, Bits)
    - Pattern matches on operation name
    - Generates appropriate MLIR based on platform context
    ↓
MLIR Builder accumulates emissions
    ↓
LLVM → Native Binary
```

**Key Architectural Points:**
1. CCS handles type checking and IntrinsicInfo creation
2. The PSG carries the intrinsic metadata through nanopasses
3. Alex witnesses consume the enriched PSG - no string matching on names
4. Platform decisions (byte order, crypto impl) flow via quotations

## 9. Relationship to Existing Intrinsics

These new modules complement existing intrinsics:

| Module | Purpose | Relationship |
|--------|---------|--------------|
| `Crypto` | Hash/encoding | Uses `byte[]` from `Array` module |
| `Bits` | Bit operations | Extends `Convert` module patterns |
| `NativePtr` | Memory access | `Crypto` may use for buffer access |
| `String` | String handling | `Crypto.base64Encode` produces strings |

## 10. Error Handling

All intrinsics in these modules follow the CCS error handling model:

- **No exceptions**: Operations return deterministic results
- **Invalid input**: Defined behavior (empty output, specific values)
- **Debug mode**: Additional runtime checks may be enabled

## 11. Normative Requirements

1. **CCS SHALL** add `Crypto` and `Bits` to `IntrinsicModule` enumeration
2. **CCS SHALL** type-check these intrinsics according to signatures in Section 6
3. **Alex SHALL** generate correct MLIR for all intrinsics
4. **Alex SHALL** respect platform byte order for `Bits.hton*`/`Bits.ntoh*`
5. **Crypto intrinsics SHALL** produce RFC-compliant output (SHA-1: FIPS 180-4, Base64: RFC 4648)
