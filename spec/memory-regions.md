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

```fsharp
Arena.using (fun arena ->
    let a = Arena.alloc<int> arena 100
    let b = Arena.alloc<float> arena 50
    // Both freed when arena scope exits
)
```

**Properties**:
- No individual deallocation
- Cache-friendly locality
- Scope-bounded lifetime

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
