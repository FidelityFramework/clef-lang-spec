---
title: "Cryptography and Bits Intrinsic Modules Specification"
weight: 470
category: Platform
status: normative
---

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-01-08

## 1. Overview

This chapter specifies two new intrinsic modules for CCS (Clef Compiler Service):

1. **Cryptography** - Cryptographic operations (SHA-1, Base64 encoding/decoding)
2. **Bits** - Bit manipulation and byte order operations

These intrinsics support the WREN stack's WebSocket communication layer, which requires:
- SHA-1 hashing for WebSocket handshake (RFC 6455)
- Base64 encoding for WebSocket accept key generation
- Byte order conversion for network protocol handling

> **Note on binary serialization.** Reinterpreting a typed value as raw bits — the
> C `reinterpret_cast` / "bit cast" idiom for reaching the IEEE-754 representation of
> a float — is **deliberately absent** from these modules. It is a category error in
> an ML-family language: it discards the value's type, its dimension, and the
> representation provenance the [numeric selection](numeric-selection.md) discipline
> depends on, and it makes the bit layout a property of the source code rather than a
> property the compiler controls. Binary serialization (the BAREWire use case) is
> handled by the **structured, deterministic-layout** path of [BAREWire](ffi-boundary.md),
> not by bit-punning a typed value into an integer. See §3.3.

## 2. Cryptography Module

### 2.1 Module Definition

```fsharp
module Cryptography
```

The Cryptography module provides cryptographic primitives. All operations are pure functions with no side effects.

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
- IntrinsicWitness pattern matches `SemanticKind.Intrinsic(Cryptography, "sha1")`
- Witness emits the SHA-1 algorithm in portable dialects (`arith`, `memref`, `func`, `scf`, `index`) at the middle end; no target commitment
- A target pathway realizes the portable form for its target: the LLVM dialect for a CPU/MCU pathway, CIRCT for an FPGA pathway, MLIR-AIE for an NPU pathway
- Alternatively, witness emits a portable `func.func` external declaration (`private`) for platform cryptography, which the CPU/MCU pathway lowers to an `llvm.func` extern and links against the platform library

**Example:**
```fsharp
let hash = Cryptography.sha1 data  // hash.Length = 20
 
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
let encoded = Cryptography.base64Encode [| 72uy; 101uy; 108uy; 108uy; 111uy |]
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
let decoded = Cryptography.base64Decode "SGVsbG8="
// decoded = [| 72uy; 101uy; 108uy; 108uy; 111uy |]
 
```

## 3. Bits Module

### 3.1 Module Definition

```fsharp
module Bits
```

The Bits module provides byte order operations for network protocol handling. All operations are pure and lower to portable `arith` operations at the middle end; a target pathway may realize them as its target's byte-order intrinsic (the LLVM dialect's `llvm.intr.bswap` on a CPU/MCU pathway).

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
- Little-endian platforms: emits the byte swap in the portable `arith` form (shift, mask, or the two-byte gather), with no target commitment
- Big-endian platforms: emits passthrough
- A target pathway realizes the portable byte swap for its target: the LLVM dialect's `llvm.intr.bswap` for a CPU/MCU pathway, an equivalent CIRCT construct for an FPGA pathway

```mlir
// Middle end (portable arith), little-endian; swaps the two bytes of an i16:
%hi   = arith.shli %value, %c8 : i16   // high byte to low position
%lo   = arith.shrui %value, %c8 : i16  // low byte to high position
%swapped = arith.ori %hi, %lo : i16
// Big-endian: passthrough (no ops)
```

```mlir
// CPU/MCU target pathway realizes the portable swap as the LLVM intrinsic:
%swapped = llvm.intr.bswap(%value) : i16
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

### 3.3 No Bit Casting (Reinterpret) — Use BAREWire Instead

**[Design decision.]** This specification **does not** provide a bit-cast /
reinterpret-cast facility (no `floatToIntBits`, no `intBitsToFloat`, no
type-punning of a typed value into its raw representation). The C and C++ idiom of
reinterpreting an IEEE-754 float as an integer to inspect or transmit its bit
pattern is a deliberate **non-feature** in Clef, for three reasons:

1. **It discards the type.** A `float<newtons>` reinterpreted as `int32` has lost its
   dimension, its unit, and its place in the dimensional algebra. Nothing downstream
   can recover that the integer "was" a force. The whole point of the type system is
   that this information is carried, not punned away.
2. **It discards the representation provenance.** [Numeric selection](numeric-selection.md)
   chooses a real value's representation (posit / IEEE / fixed-point) from its
   analyzed range, per target. A bit cast assumes the bits *are* IEEE-754, hard-coding
   a representation the compiler was supposed to choose — and silently producing
   garbage on a target where the value was lowered to a posit or a fixed-point format.
3. **It moves the bit layout into the source.** Bit-punning makes the byte layout a
   fact about the program text rather than a property the compiler controls and can
   re-check through lowering. That is the opposite of the framework's
   information-preservation discipline.

The legitimate need that motivates bit casting in C — **binary serialization**, e.g.
writing a value onto the wire for [BAREWire](ffi-boundary.md) — is handled by
BAREWire's **structured, deterministic-layout** path, not by reinterpreting a typed
value as an integer. BAREWire serializes a value *as the typed value it is*, with the
byte layout determined by the contract both endpoints were built to read; the layout
is the compiler's to fix and the contract's to enforce, and the dimensional and
representation metadata travel with it. A value crosses the wire as itself, and the
type is checked at the fabric — never reconstructed from a guessed bit pattern.

Where a developer genuinely needs an explicit representation change (e.g. a lossy
narrowing conversion), that is the *explicit, fidelity-recorded conversion* discipline
of [Rounding §6](rounding.md) and [Numeric Selection §5](numeric-selection.md), which
is a typed, witnessed conversion — not an untyped bit reinterpretation.

## 4. IntrinsicModule Enumeration

Add the following variants to `IntrinsicModule`:

```fsharp
type IntrinsicModule =
    // ... existing variants ...
    | Cryptography  // Cryptographic operations
    | Bits          // Byte order operations
 
```

## 5. IntrinsicCategory Classification

| Intrinsic | Category |
|-----------|----------|
| `Cryptography.sha1` | `Pure` |
| `Cryptography.base64Encode` | `Pure` |
| `Cryptography.base64Decode` | `Pure` |
| `Bits.htons` | `Pure` |
| `Bits.ntohs` | `Pure` |
| `Bits.htonl` | `Pure` |
| `Bits.ntohl` | `Pure` |

All operations are `Pure` category - no side effects, deterministic output.

## 6. Type Signatures Summary

| Intrinsic | Type Signature |
|-----------|----------------|
| `Cryptography.sha1` | `byte[] -> byte[]` |
| `Cryptography.base64Encode` | `byte[] -> string` |
| `Cryptography.base64Decode` | `string -> byte[]` |
| `Bits.htons` | `uint16 -> uint16` |
| `Bits.ntohs` | `uint16 -> uint16` |
| `Bits.htonl` | `uint32 -> uint32` |
| `Bits.ntohl` | `uint32 -> uint32` |

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

### 7.2 Cryptography Witness Implementation Options

The IntrinsicWitness for Cryptography operations has two implementation strategies:

1. **Inline witness** (preferred for freestanding):
   - Witness emits the SHA-1/Base64 algorithms in portable dialects (`arith`, `memref`, `func`, `scf`, `index`); the target pathway realizes the portable form for its target
   - No external dependencies
   - Larger binary size
   - Complete self-containment

2. **External witness** (optional for console/desktop):
   - Witness emits a portable `func.func` external declaration (`private`) for platform cryptography; the CPU/MCU pathway lowers it to an `llvm.func` extern
   - Links against libcrypto (OpenSSL) or platform equivalent
   - Smaller binary
   - External dependency (available only on a pathway with an FFI boundary; a freestanding no-heap target has none)

The choice is made via `.fidproj` configuration and flows through platform quotations:

```toml
[compilation]
cryptography_implementation = "inline"  # or "platform"
 
```

The witness queries this setting via [platform context](platform-bindings.md) during MLIR generation.

## 8. Nanopass and Witness Flow

The pipeline for Cryptography and Bits intrinsics follows the standard CCS→Alex flow:

```
Clef Source: Cryptography.sha1 data
    ↓
CCS Type Checking (Expressions/Intrinsics.fs, Expressions/Coordinator.fs)
    - Coordinator dispatches to Intrinsics module for intrinsic resolution
    - Recognizes "Cryptography.sha1" pattern
    - Creates IntrinsicInfo { Module=Cryptography, Operation="sha1", Category=Pure }
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
    - Pattern matches on IntrinsicModule (Cryptography, Bits)
    - Pattern matches on operation name
    - Emits portable dialects (arith/memref/func/scf/index) based on platform context; no target commitment
    ↓
MLIR Builder accumulates portable emissions
    ↓
Target pathway (target commitment): LLVM dialect → Native Binary (CPU/MCU),
                                 or CIRCT (FPGA), or MLIR-AIE (NPU)
```

**Key Architectural Points:**
1. CCS handles type checking and IntrinsicInfo creation
2. [The PSG carries](program-semantic-graph.md) the intrinsic metadata through nanopasses
3. Alex witnesses consume the enriched PSG - no string matching on names
4. Platform decisions (byte order, cryptography impl) flow via quotations

## 9. Relationship to Existing Intrinsics

These new modules complement existing intrinsics:

| Module | Purpose | Relationship |
|--------|---------|--------------|
| `Cryptography` | Hash/encoding | Uses `byte[]` from `Array` module |
| `Bits` | Byte order | Network protocol byte order conversion |
| `String` | String handling | `Cryptography.base64Encode` produces strings |

Binary serialization is **not** in this set — it is handled by the structured
[BAREWire](ffi-boundary.md) path (see §3.3), which carries the value's type and layout
rather than reinterpreting its bits.

## 10. Error Handling

All intrinsics in these modules follow the CCS error handling model:

- **No exceptions**: Operations return deterministic results
- **Invalid input**: Defined behavior (empty output, specific values)
- **Debug mode**: Additional runtime checks may be enabled

## 11. Normative Requirements

1. **CCS SHALL** add `Cryptography` and `Bits` to `IntrinsicModule` enumeration
2. **CCS SHALL** type-check these intrinsics according to signatures in Section 6
3. **Alex SHALL** generate correct MLIR for all intrinsics
4. **Alex SHALL** respect platform byte order for `Bits.hton*`/`Bits.ntoh*`
5. **Cryptography intrinsics SHALL** produce RFC-compliant output (SHA-1: FIPS 180-4, Base64: RFC 4648)
6. **This specification SHALL NOT** provide a bit-cast / reinterpret-cast facility; binary serialization SHALL be handled by the structured [BAREWire](ffi-boundary.md) path (§3.3).
