# Memory Regions

Memory region types define where memory lives and how it behaves. They are intrinsic to F# Native and guide code generation throughout the compilation pipeline.

## Overview

F# Native extends the type system with memory region information. Every pointer type carries region semantics that determine:

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
| `Sram` | General-purpose RAM | No | Yes |
| `Flash` | Read-only program memory | No | Yes |

### Stack

Stack-allocated values have automatic lifetime bounded by their lexical scope.

```fsharp
let example () =
    let buffer : Ptr<byte, Stack, ReadWrite> = stackalloc 1024
    // buffer is valid until function returns
    use buffer
```

**Properties**:
- Thread-local storage
- Automatic cleanup at scope exit
- Size limits enforced by platform

### Arena

Arena-allocated values are bulk-allocated and freed together.

> **Status (January 2026)**: Arena is implemented as an FNCS intrinsic type with compiler-provided operations.

**Type Definition**:
```fsharp
// Arena<[<Measure>] 'lifetime> - FNCS intrinsic type
// Layout: NTUCompound(3) = { Base: nativeint, Capacity: int, Position: int }
```

**Current Implementation (Level 3 - Explicit)**:
```fsharp
// Create arena from stack-allocated backing memory
let arenaMem = NativePtr.stackalloc<byte> 4096
let mutable arena = Arena.fromPointer (NativePtr.toNativeInt arenaMem) 4096

// Allocate from arena (note: byref parameter for mutation)
let buffer = Arena.alloc &arena 256  // Returns nativeint
let aligned = Arena.allocAligned &arena 64 16  // 64 bytes, 16-byte aligned

// Query and reset
let remaining = Arena.remaining arena
Arena.reset &arena  // Position back to 0
```

**Arena Operations** (FNCS Intrinsics):

| Operation | Type | Description |
|-----------|------|-------------|
| `fromPointer` | `nativeint -> int -> Arena<'lifetime>` | Create arena from backing memory |
| `alloc` | `Arena<'lifetime> byref -> int -> nativeint` | Bump allocate bytes |
| `allocAligned` | `Arena<'lifetime> byref -> int -> int -> nativeint` | Aligned allocation |
| `remaining` | `Arena<'lifetime> -> int` | Query remaining capacity |
| `reset` | `Arena<'lifetime> byref -> unit` | Reset position to 0 |

**Lifetime Parameter**: The `'lifetime` measure parameter enables future lifetime tracking. Currently documentation-level; compiler enforcement planned.

**Three Levels of Control** (Lifetime Inference Principle):
1. **Level 3 (Explicit)**: Full control via `Arena.fromPointer`, `Arena.alloc &arena` (implemented)
2. **Level 2 (Hints)**: `arena { }` computation expression (future)
3. **Level 1 (Inferred)**: Compiler escape analysis infers arena needs (future)

**Properties**:
- No individual deallocation
- O(1) bump allocation
- Cache-friendly locality
- Scope-bounded lifetime
- Backing memory can come from stack or heap

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

Pointers carry region and access information in their type:

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

## See Also

- [Access Kinds](access-kinds.md) - Read/Write permissions
- [Platform Bindings](platform-bindings.md) - Platform-specific memory access
