---
title: "Atomic Operations and Memory Ordering"
weight: 510
category: Semantics
status: normative
---

Clef provides atomic operations for concurrent programming. These operations map to hardware atomics with defined memory ordering semantics.

## Design Philosophy

Clef follows a **layered exposure model** for concurrency primitives:

| Layer | Audience | Semantics | Ordering |
|-------|----------|-----------|----------|
| Application | Most developers | Actors, channels | Sequential consistency (implicit) |
| Library | Framework implementers | `Atomic` module | Full ordering control |
| Runtime | Fidelity internals | Portable atomic encoding, realized per target pathway | Platform-specific |

Application developers use actors and channels. Library developers use the `Atomic` module when implementing lock-free data structures. The complexity of memory ordering is contained to library implementations.

## Memory Ordering Levels

Memory ordering specifies visibility guarantees across threads:

### Relaxed (Monotonic)

Atomic operation with no ordering constraints. Other memory operations may be reordered around it.

```fsharp
// Only guarantees atomicity, not visibility ordering
let counter = Atomic.Relaxed.load ptr
Atomic.Relaxed.store ptr (counter + 1)
```

Use case: Counters where exact order doesn't matter.

### Acquire

Prevents memory operations after this load from being reordered before it:

```fsharp
// All reads after this see writes before the paired release
let value = Atomic.Acquire.load ptr
```

Use case: Load side of a lock or flag.

### Release

Prevents memory operations before this store from being reordered after it:

```fsharp
// All writes before this are visible to threads that acquire
Atomic.Release.store ptr value
```

Use case: Store side of a lock or flag.

### Acquire-Release

Combines acquire and release semantics for read-modify-write operations:

```fsharp
// Both acquire and release semantics
let old = Atomic.AcquireRelease.exchange ptr newValue
```

Use case: Spinlocks, reference counting.

### Sequential Consistency

Total ordering across all sequentially consistent operations:

```fsharp
// Full fence, total order with all other SeqCst operations
let value = Atomic.SeqCst.load ptr
Atomic.SeqCst.store ptr newValue
```

Use case: Default choice when ordering requirements are unclear.

## The Atomic Module

### Atomic.SeqCst (Default)

Sequential consistency is the default and recommended ordering:

```fsharp
module Atomic.SeqCst =
    /// Atomically load a value
    val load<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> 'T

    /// Atomically store a value
    val store<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> unit

    /// Atomically exchange values, return old value
    val exchange<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T

    /// Compare and swap, return success and old value
    val compareExchange<'T when 'T : unmanaged> :
        cell:Ptr<'T, 'Region, ReadWrite> -> expected:'T -> desired:'T -> struct ('T * bool)

    /// Atomically add, return old value
    val fetchAdd<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T

    /// Atomically subtract, return old value
    val fetchSub<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T

    /// Atomically AND, return old value
    val fetchAnd<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T

    /// Atomically OR, return old value
    val fetchOr<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T

    /// Atomically XOR, return old value
    val fetchXor<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T
```

### Atomic.Relaxed

For operations where only atomicity matters:

```fsharp
module Atomic.Relaxed =
    val load<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> 'T
    val store<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> unit
    val exchange<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T
    val compareExchange<'T when 'T : unmanaged> :
        cell:Ptr<'T, 'Region, ReadWrite> -> expected:'T -> desired:'T -> struct ('T * bool)
    val fetchAdd<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T
    // ... other operations
 
```

### Atomic.Acquire / Atomic.Release

For synchronization patterns:

```fsharp
module Atomic.Acquire =
    val load<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> 'T
    val compareExchange<'T when 'T : unmanaged> :
        cell:Ptr<'T, 'Region, ReadWrite> -> expected:'T -> desired:'T -> struct ('T * bool)

module Atomic.Release =
    val store<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> unit
    val compareExchange<'T when 'T : unmanaged> :
        cell:Ptr<'T, 'Region, ReadWrite> -> expected:'T -> desired:'T -> struct ('T * bool)

module Atomic.AcquireRelease =
    val exchange<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T
    val compareExchange<'T when 'T : unmanaged> :
        cell:Ptr<'T, 'Region, ReadWrite> -> expected:'T -> desired:'T -> struct ('T * bool)
    val fetchAdd<'T when 'T : unmanaged> : cell:Ptr<'T, 'Region, ReadWrite> -> value:'T -> 'T
    // ... other operations
 
```

## Memory Fences

Explicit memory barriers when needed:

```fsharp
module Atomic =
    /// Acquire fence - prevent reordering of subsequent reads
    val fenceAcquire : unit -> unit

    /// Release fence - prevent reordering of prior writes
    val fenceRelease : unit -> unit

    /// Full fence - sequential consistency barrier
    val fenceSeqCst : unit -> unit
```

## Hardware Mapping

### x86-64

x86-64 has a strong memory model (TSO - Total Store Order):

| Clef | x86-64 |
|-----------|--------|
| Relaxed load | `mov` |
| Relaxed store | `mov` |
| Acquire load | `mov` (implicit acquire) |
| Release store | `mov` (implicit release) |
| SeqCst load | `mov` + implicit fence |
| SeqCst store | `xchg` or `mov` + `mfence` |
| compareExchange | `lock cmpxchg` |
| fetchAdd | `lock xadd` |

### ARM64

ARM64 has a weaker memory model:

| Clef | ARM64 |
|-----------|-------|
| Relaxed load | `ldr` |
| Relaxed store | `str` |
| Acquire load | `ldar` |
| Release store | `stlr` |
| SeqCst load | `ldar` |
| SeqCst store | `stlr` |
| compareExchange | `ldaxr`/`stlxr` loop |
| fetchAdd | `ldaddal` |

## Type Constraints

Atomic operations are constrained to types that can be atomically accessed by hardware:

```fsharp
// Valid atomic types (platform-dependent)
// x86-64: 1, 2, 4, 8 byte aligned types
// ARM64: 1, 2, 4, 8 byte aligned types

// Valid
Atomic.SeqCst.load<int32> cell
Atomic.SeqCst.load<int64> cell
Atomic.SeqCst.load<usize> cell

// Invalid (too large for hardware atomic)
Atomic.SeqCst.load<LargeStruct> cell  // Compile error
 
```

## Lock-Free Pattern Examples

### Spinlock

```fsharp
type Spinlock = { mutable locked: int32 }

// The atomic cell is a region-typed handle to the `locked` field.
// For a program-lifetime lock this handle points into static storage (Sram);
// `Ptr.field` projects the field handle from a handle to the record.
let acquire (lock: Ptr<Spinlock, 'Region, ReadWrite>) =
    let locked = Ptr.field lock (fun s -> s.locked)  // Ptr<int32, 'Region, ReadWrite>
    while Atomic.AcquireRelease.compareExchange locked 0 1
          |> snd |> not do
        // Spin
        ()

let release (lock: Ptr<Spinlock, 'Region, ReadWrite>) =
    let locked = Ptr.field lock (fun s -> s.locked)
    Atomic.Release.store locked 0
```

### Reference Counting

```fsharp
type RefCounted<'T> = {
    mutable refCount: int32
    value: 'T
}

let retain (obj: Ptr<RefCounted<'T>, 'Region, ReadWrite>) =
    let refCount = Ptr.field obj (fun o -> o.refCount)  // Ptr<int32, 'Region, ReadWrite>
    Atomic.Relaxed.fetchAdd refCount 1 |> ignore

let release (obj: Ptr<RefCounted<'T>, 'Region, ReadWrite>) =
    let refCount = Ptr.field obj (fun o -> o.refCount)
    let oldCount = Atomic.AcquireRelease.fetchSub refCount 1
    if oldCount = 1 then
        Atomic.fenceAcquire ()
        // Reclaim obj's storage
 
```

### Single-Producer Single-Consumer Queue

```fsharp
// `Capacity` is a compile-time bound; the buffer is a bounded array whose
// storage lives in the region 'Region (Stack for a scope-bounded queue,
// Sram for a program-lifetime one). No raw element pointer is exposed.
type SPSCQueue<'T, 'Capacity, 'Region> = {
    buffer: array<'T, 'Capacity, 'Region>
    mutable head: int  // Written by consumer, read by producer
    mutable tail: int  // Written by producer, read by consumer
}

let enqueue (q: Ptr<SPSCQueue<'T, 'Capacity, 'Region>, 'QRegion, ReadWrite>) (item: 'T) =
    let tailCell = Ptr.field q (fun s -> s.tail)  // Ptr<int, 'QRegion, ReadWrite>
    let headCell = Ptr.field q (fun s -> s.head)
    let tail = Atomic.Relaxed.load tailCell
    let head = Atomic.Acquire.load headCell
    if tail - head < capacity then
        let slot = Ptr.field q (fun s -> s.buffer.[tail % capacity])
        Ptr.write slot item
        Atomic.Release.store tailCell (tail + 1)
        true
    else
        false
```

## Relationship to Actor Model

The actor model eliminates most need for explicit atomics:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Concurrency Model Layers                             │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Actor Model (Application Level)                                   │ │
│  │  - Message passing                                                 │ │
│  │  - No shared mutable state                                         │ │
│  │  - Sequential consistency guaranteed                               │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                              │                                          │
│                              ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Channel Implementation (Library Level)                            │ │
│  │  - Uses Atomic module internally                                   │ │
│  │  - Lock-free where possible                                        │ │
│  │  - Memory ordering handled correctly                               │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                              │                                          │
│                              ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Atomic Module (Library Designer Level)                            │ │
│  │  - Explicit memory ordering                                        │ │
│  │  - For custom synchronization primitives                          │ │
│  │  - Expert use only                                                 │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

Most developers never use `Atomic` directly. The actor runtime uses it internally.

## Compiler Intrinsic Status

Atomic operations are CCS (Clef Compiler Service) intrinsics:

```fsharp
// In CCS Expressions/Intrinsics.fs
| "Atomic.SeqCst.load" ->
    NativeType.TFun(NativeType.TNativePtr tvar, tvar)

| "Atomic.SeqCst.store" ->
    NativeType.TFun(NativeType.TNativePtr tvar,
        NativeType.TFun(tvar, env.Globals.UnitType))

| "Atomic.SeqCst.compareExchange" ->
    NativeType.TFun(NativeType.TNativePtr tvar,
        NativeType.TFun(tvar,
            NativeType.TFun(tvar,
                NativeType.TStruct [tvar; env.Globals.BoolType])))
```

Alex emits a portable atomic encoding that preserves the memory-ordering annotation and commits to no target. Each target pathway then realizes it: the LLVM pathway maps to LLVM atomic instructions with the corresponding ordering, the thumbv8m pathway to `ldar`/`stlr` and the exclusive-monitor loop, and other pathways to their own atomic primitives.

## Grammar

```fsgrammar
atomic-ordering :=
    Relaxed
    Acquire
    Release
    AcquireRelease
    SeqCst

atomic-operation :=
    Atomic . atomic-ordering . operation-name

operation-name :=
    load
    store
    exchange
    compareExchange
    fetchAdd
    fetchSub
    fetchAnd
    fetchOr
    fetchXor
```
