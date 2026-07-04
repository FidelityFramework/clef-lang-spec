---
title: "Memory Regions"
weight: 460
category: Representation
status: normative
---

Memory region types define where memory lives and how it behaves. They are intrinsic to Clef and guide code generation throughout the compilation pipeline.

## Overview

Clef extends the type system with memory region information. Every pointer type carries region semantics that determine:

- Allocation strategy
- Access patterns (volatile, cached)
- Lifetime constraints
- Code generation decisions

## Region Types

The following memory regions are defined:

| Region | Use Case | Volatile | Cacheable |
|--------|----------|----------|-----------|
| `Stack` | Thread-local automatic storage | No | Yes |
| `Arena` | Bulk allocation with batch deallocation | No | Yes |
| `Peripheral` | Memory-mapped I/O | Yes | No |
| `Sram` | General-purpose RAM; static storage for mutable program-lifetime values | No | Yes |
| `Flash` | Read-only program memory; static storage for immutable program-lifetime values | No | Yes |

`Sram` and `Flash` are also the *static storage* of the [lifetime lattice](closure-representation.md): a value whose lifetime is the whole program (constructed once, held to program end, never freed) is placed in `Sram` when mutable or `Flash` when immutable, as a program-lifetime global rather than a heap allocation. A fixed-address `Peripheral` register is already a program-lifetime global of this kind; a program-lifetime closure, list, or record is the same, and lives in the same static storage. This is what lets a target without a heap still hold a value that outlives its constructing scope.

### Stack

Stack-allocated values have automatic lifetime bounded by their lexical scope.

```fsharp
let example () =
    // Fixed-size stack array; storage is allocated in the Stack region
    let buffer : array<byte, 1024, Stack> = [| 0uy; ... |]
    // A region-typed handle to the buffer's storage, valid until the function returns
    let handle : Ptr<byte, Stack, ReadWrite> = Ptr.ofArray buffer
    use handle
```

**Properties**:
- Thread-local storage
- Automatic cleanup at scope exit
- Size limits enforced by platform

### Arena

Arena-allocated values are bulk-allocated and freed together.

> **Status (January 2026)**: Arena is implemented as a CCS (Clef Compiler Service) intrinsic type with compiler-provided operations.

**Type Definition**:
```fsharp
// Arena<[<Measure>] 'lifetime> - CCS intrinsic type
// Layout: NTUCompound(3) = { Base: nativeint, Capacity: int, Position: int }
 
```

**Current Implementation (Level 3 - Explicit)**:
```fsharp
// Create arena from a bounded stack array as backing memory
let arenaMem : array<byte, 4096, Stack> = [| 0uy; ... |]
let mutable arena = Arena.fromArray arenaMem  // capacity taken from the array bound

// Allocate from arena (note: byref parameter for mutation)
let buffer = Arena.alloc &arena 256  // Returns a Ptr<byte, Arena, ReadWrite> handle
let aligned = Arena.allocAligned &arena 64 16  // 64 bytes, 16-byte aligned

// Query and reset
let remaining = Arena.remaining arena
Arena.reset &arena  // Position back to 0
 
```

**Arena Operations** (CCS Intrinsics):

| Operation | Type | Description |
|-----------|------|-------------|
| `fromArray` | `array<byte, 'n, Stack> -> Arena<'lifetime>` | Create arena from a bounded stack-array backing |
| `alloc` | `Arena<'lifetime> byref -> int -> Ptr<byte, Arena, ReadWrite>` | Bump allocate bytes |
| `allocAligned` | `Arena<'lifetime> byref -> int -> int -> Ptr<byte, Arena, ReadWrite>` | Aligned allocation |
| `remaining` | `Arena<'lifetime> -> int` | Query remaining capacity |
| `reset` | `Arena<'lifetime> byref -> unit` | Reset position to 0 |

**Lifetime Parameter**: The `'lifetime` measure parameter enables future lifetime tracking. Currently documentation-level; compiler enforcement planned.

**Three Levels of Control** (Lifetime Inference Principle):
1. **Level 3 (Explicit)**: Full control via `Arena.fromArray`, `Arena.alloc &arena` (implemented)
2. **Level 2 (Hints)**: `arena { }` computation expression (future)
3. **Level 1 (Inferred)**: Escape analysis via `inline` expansion - see [Inline Functions and Escape Analysis](special-attributes-and-types.md#inline-functions-and-escape-analysis) (implemented)

**Properties**:
- No individual deallocation
- O(1) bump allocation
- Cache-friendly locality
- Scope-bounded lifetime
- Backing memory comes from the stack, from static storage (`Sram`/`Flash`), or, where the target has one, from the heap. A target without a heap backs arenas with stack or static storage only.

### Peripheral

Peripheral regions represent memory-mapped I/O with volatile semantics.

```fsharp
let gpioReg : Ptr<uint32, Peripheral, ReadWrite> = Ptr.ofAddress 0x48000000UL
```

**Properties**:
- Volatile loads and stores
- No reordering by compiler or CPU
- Memory barriers as required
- Not cacheable

### Sram

General-purpose RAM with standard memory semantics.

**Properties**:
- Cacheable
- Standard load/store semantics
- May be reordered by optimizer

### Flash

Read-only program memory, typically for embedded systems.

```fsharp
let lookupTable : Ptr<int, Flash, ReadOnly> = ...
```

**Properties**:
- Read-only access enforced
- Link-time placement
- Cacheable

## Region-Typed Pointers

Pointers carry region and [access information](access-kinds.md) in their type:

```fsharp
type Ptr<'T, 'Region, 'Access>
```

**Examples**:
```fsharp
Ptr<uint32, Peripheral, ReadWrite>  // GPIO register
Ptr<byte, Flash, ReadOnly>          // Constant data
Ptr<int, Stack, ReadWrite>          // Stack buffer
Ptr<float, Arena, ReadWrite>        // Arena-allocated array
 
```

## Compile-Time Enforcement

Region mismatches are type errors:

```fsharp
// ERROR: Cannot assign Peripheral pointer to Stack pointer
let wrong : Ptr<int, Stack, ReadWrite> = peripheralPtr
```

## Code Generation Effects

| Region | Generated Code |
|--------|---------------|
| `Stack` | Stack pointer arithmetic |
| `Arena` | Arena allocator calls |
| `Peripheral` | Volatile loads/stores, memory barriers |
| `Sram` | Standard memory access |
| `Flash` | Read-only access patterns |

## Lifetime Constraints

> **DEFERRED**: Detailed lifetime verification rules will be specified in a future revision covering ownership and borrowing semantics.

## Grammar

```fsgrammar
region-type :=
    Stack
    Arena
    Peripheral
    Sram
    Flash

region-typed-ptr := Ptr < type , region-type , access-kind >
```
