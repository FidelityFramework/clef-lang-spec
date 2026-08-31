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

The lattice extends one rung past the process on targets that claim the Freestanding Substrate profile: [Modular Blob Storage](modular-blob-storage.md) persists values that outlive a program run, and [Namespace Storage](namespace-storage.md) layers names and history above it. Both are placed through the regions of this chapter — sealed records in non-volatile storage, working sets in bounded RAM — and both carry their durability as a coeffect committed at target binding.

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

The `{ Base, Capacity, Position }` layout states the allocation discipline as checkable facts: every allocation advances `Position` by the requested (aligned) size, and `Position` never exceeds `Capacity`. An implementation's allocation emission is subject to the [preservation and diagnostic obligations](conformance.md) over exactly these facts.

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

Lifetime verification in Clef is coeffect discipline carried on the Program Semantic Graph, with no ownership or borrowing annotations in the source language. The normative floor is in place: every value is classified against the four-point lifetime lattice by escape analysis ([Closure Representation §3.3](closure-representation.md)), placement follows the classification, a classification with no home on the selected target is diagnosed at compile time, and the classification is subject to the preservation and introduction obligations of [Conformance §6](conformance.md).

The rules that remain to be specified are the **lifetime orderings**: that a region outlives every value placed in it, and that every use of a value falls within its region's extent. These are orderings over the lattice, decidable facts stated and discharged as proof obligations that ride the PSG with the escape coeffect ([Program Semantic Graph §14.3](program-semantic-graph.md)). A future revision of this chapter SHALL state the ordering rules and their obligation forms; the mechanism is fixed as coeffect-and-obligation discipline, and no ownership or borrowing vocabulary is planned.

## Target Reachability

The region types of this chapter name storage a target provides, and the platform descriptor lists the regions each target has ([Platform Bindings](platform-bindings.md)). A construct whose region the selected target does not provide SHALL be diagnosed at compile time when compiled for that target; reachability is carried as a lattice-family coeffect alongside escape classification ([Grade Discipline §4.1](grade-discipline.md)).

On the JSIR pathway, no region of this chapter exists: the host garbage collector owns placement, every lifetime class of the [lifetime lattice](closure-representation.md) has a home, and `Ptr`, `Arena`, `Mmio`, and every region-typed construct SHALL be diagnosed as unavailable for the target (the CCS8030 pattern of [Platform Bindings](platform-bindings.md)). The lifetime lattice itself still classifies every value at design time on that pathway; only the storage classes of this chapter are absent.

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
