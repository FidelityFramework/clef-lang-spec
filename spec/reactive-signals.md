---
title: "Reactive Signals Module Specification"
weight: 390
---

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-01-08

## 1. Overview

This chapter specifies native reactive signals for CCS (Clef Compiler Service), inspired by SolidJS's fine-grained reactivity and TanStack Store's framework-agnostic approach.

**Design Goals:**
- Mirror SolidJS/TanStack signal semantics for frontend/backend consistency
- Zero-runtime-overhead abstractions where possible
- Function pointer support for callbacks (no closures)
- Integration with native event loops (epoll, GLib)

## 2. Function Pointer Type

### 2.1 Type Definition

CCS introduces a function pointer type for callbacks:

```fsharp
type FnPtr<'T, 'R>  // Function pointer from 'T to 'R
```

**NTU Kind:** `NTUfnptr` (new NTU kind, pointer-sized)

**Syntax:**
```fsharp
// Function pointer to a function taking int and returning string
let callback: FnPtr<int, string> = ...
```

### 2.2 Creating Function Pointers

Function pointers can only be created from **top-level functions** (no closures):

```fsharp
// Top-level function
let handleEvent (x: int) : unit =
    Console.writeln (sprintf "Event: %d" x)

// Create function pointer
let handler: FnPtr<int, unit> = FnPtr.ofFunction handleEvent
```

**MLIR Generation:**
```mlir
// Function pointer is the address of the function
%fn_ptr = llvm.mlir.addressof @handleEvent : !llvm.ptr<func<void (i32)>>
```

### 2.3 Invoking Function Pointers

```fsharp
FnPtr.invoke handler 42  // Calls handleEvent(42)
```

**MLIR Generation:**
```mlir
llvm.call %fn_ptr(%arg) : !llvm.ptr<func<void (i32)>>, (i32) -> ()
```

### 2.4 Null Function Pointers

```fsharp
FnPtr.isNull handler     // Check if null
FnPtr.null<int, unit>()  // Create null pointer
```

## 3. Signal Module

### 3.1 Module Definition

```fsharp
module Signal
```

The Signal module provides reactive primitives. Unlike SolidJS which uses closures, CCS signals use explicit subscription with function pointers.

### 3.2 Signal Type

```fsharp
type Signal<'T>  // Opaque handle to a reactive value
```

**Internal Representation:**
- Platform word slot index into runtime signal table
- NTU Kind: None (opaque handle type, layout determined by TypeLayout.PlatformWord)

### 3.3 Creating Signals

```fsharp
val create : 'T -> Signal<'T>
```

**Semantics:**
- Allocates a slot in the signal table
- Stores initial value
- Returns opaque handle

**Example:**
```fsharp
let count = Signal.create 0
let name = Signal.create "World"
```

### 3.4 Reading Signals

```fsharp
val get : Signal<'T> -> 'T
```

**Semantics:**
- Returns current value
- If called within an effect, registers dependency

**Example:**
```fsharp
let currentCount = Signal.get count  // Returns 0
```

### 3.5 Writing Signals

```fsharp
val set : Signal<'T> -> 'T -> unit
val update : Signal<'T> -> ('T -> 'T) -> unit
```

**Semantics:**
- `set`: Replaces value, notifies subscribers if changed
- `update`: Applies function to current value, notifies if changed

**Example:**
```fsharp
Signal.set count 5           // Set to 5
Signal.update count ((+) 1)  // Increment by 1
```

### 3.6 Comparison Function

```fsharp
val setWithCompare : Signal<'T> -> 'T -> FnPtr<'T * 'T, bool> -> unit
```

**Semantics:**
- Updates only if comparison function returns false (values are different)
- Allows custom equality for complex types

## 4. Effect Module

### 4.1 Module Definition

```fsharp
module Effect
```

Effects are computations that run when their dependencies (signals) change.

### 4.2 Effect Type

```fsharp
type Effect  // Opaque handle to an effect
```

### 4.3 Creating Effects

```fsharp
val create : FnPtr<unit, unit> -> Effect
```

**Semantics:**
- Registers the function pointer as an effect
- Runs the effect immediately to establish dependencies
- Re-runs whenever dependencies change

**Example:**
```fsharp
// Top-level effect function
let logCount () : unit =
    let c = Signal.get count
    Console.writeln (sprintf "Count: %d" c)

// Create effect
let countLogger = Effect.create (FnPtr.ofFunction logCount)
```

### 4.4 Creating Effects with Cleanup

```fsharp
val createWithCleanup : FnPtr<unit, FnPtr<unit, unit>> -> Effect
```

**Semantics:**
- Effect function returns a cleanup function pointer
- Cleanup runs before each re-execution and on disposal

### 4.5 Disposing Effects

```fsharp
val dispose : Effect -> unit
```

**Semantics:**
- Removes effect from subscription graph
- Runs cleanup if present
- Effect will no longer re-run

## 5. Memo Module

### 5.1 Module Definition

```fsharp
module Memo
```

Memos are derived values that cache computation results.

### 5.2 Memo Type

```fsharp
type Memo<'T>  // Opaque handle to a memoized value
```

### 5.3 Creating Memos

```fsharp
val create : FnPtr<unit, 'T> -> Memo<'T>
```

**Semantics:**
- Computation runs when dependencies change
- Result is cached until dependencies change
- Reading a memo is like reading a signal

**Example:**
```fsharp
// Top-level computation
let doubleCount () : int =
    Signal.get count * 2

// Create memo
let doubled = Memo.create (FnPtr.ofFunction doubleCount)

// Read memo (cached)
let value = Memo.get doubled
```

### 5.4 Reading Memos

```fsharp
val get : Memo<'T> -> 'T
```

**Semantics:**
- Returns cached value
- Recomputes if dependencies changed
- If called within an effect, registers dependency

## 6. Batch Module

### 6.1 Module Definition

```fsharp
module Batch
```

Batching groups multiple signal updates to avoid redundant effect runs.

### 6.2 Batching Updates

```fsharp
val run : FnPtr<unit, unit> -> unit
```

**Semantics:**
- All signal updates within the batch are deferred
- Effects run once after batch completes
- Prevents intermediate state inconsistencies

**Example:**
```fsharp
let updateBoth () : unit =
    Signal.set firstName "John"
    Signal.set lastName "Doe"

// Effects only run once after both updates
Batch.run (FnPtr.ofFunction updateBoth)
```

## 7. Store Module (Optional Extension)

### 7.1 Module Definition

```fsharp
module Store
```

Stores provide nested reactive objects, similar to TanStack Store.

### 7.2 Store Type

```fsharp
type Store<'T>  // Opaque handle to a reactive store
```

### 7.3 Creating Stores

```fsharp
val create : 'T -> Store<'T>
```

**Semantics:**
- Creates a reactive store from an initial value
- Nested properties become reactive

### 7.4 Reading Store State

```fsharp
val state : Store<'T> -> 'T
```

### 7.5 Updating Store State

```fsharp
val setState : Store<'T> -> ('T -> 'T) -> unit
```

### 7.6 Subscribing to Store Changes

```fsharp
val subscribe : Store<'T> -> FnPtr<unit, unit> -> FnPtr<unit, unit>
```

**Returns:** Unsubscribe function pointer

## 8. Runtime Implementation

### 8.1 Signal Table

The runtime maintains a signal table:

```
┌─────────────────────────────────────────────────────┐
│ Signal Table (arena-allocated)                      │
├──────┬──────────┬────────────┬─────────────────────┤
│ Slot │ Value    │ Dirty Flag │ Subscriber List     │
├──────┼──────────┼────────────┼─────────────────────┤
│ 0    │ 42       │ false      │ [Effect 0, Memo 1]  │
│ 1    │ "Hello"  │ true       │ [Effect 2]          │
│ ...  │ ...      │ ...        │ ...                 │
└──────┴──────────┴────────────┴─────────────────────┘
```

### 8.2 Dependency Tracking

During effect/memo execution:
1. Set `CurrentTracking` to effect/memo handle
2. Execute function
3. Each `Signal.get` registers dependency
4. Clear `CurrentTracking`

### 8.3 Notification Flow

On `Signal.set`:
1. Compare new value with old (skip if equal)
2. Update value in table
3. If batching: mark dirty, defer notification
4. If not batching: notify subscribers immediately

### 8.4 Subscription Management

Effects maintain their dependency list:
- On re-run: clear old dependencies, establish new ones
- On dispose: remove from all signal subscriber lists

## 9. Event Loop Integration

### 9.1 Integration with epoll

Signals can be connected to external events:

```fsharp
// Top-level handler for socket events
let onSocketReadable (fd: int) : unit =
    let data = Sockets.recv fd buffer 1024 0
    Signal.set socketData data

// Register with epoll
EventLoop.onReadable socketFd (FnPtr.ofFunction onSocketReadable)
```

### 9.2 Integration with GTK/GLib

For GTK applications:

```fsharp
// Top-level handler for window destroy
let onWindowDestroy () : unit =
    Signal.set appState AppState.Closing

// Connect GTK signal (via platform bindings)
GTK.signalConnect window "destroy" (FnPtr.ofFunction onWindowDestroy)
```

## 10. NTU Types Summary

| Type | NTU Kind | Layout | Notes |
|------|----------|--------|-------|
| `FnPtr<'T,'R>` | `NTUfnptr` | PlatformWord | Function pointer |
| `Signal<'T>` | None | PlatformWord | Opaque slot index |
| `Effect` | None | PlatformWord | Opaque handle |
| `Memo<'T>` | None | PlatformWord | Opaque handle |
| `Store<'T>` | None | PlatformWord | Opaque handle |

## 11. IntrinsicModule Classification

Add the following to `IntrinsicModule`:

```fsharp
type IntrinsicModule =
    // ... existing variants ...
    | FnPtr      // Function pointer operations
    | Signal     // Reactive signals
    | Effect     // Side effects
    | Memo       // Memoized computations
    | Batch      // Update batching
    | Store      // Reactive stores (optional)
```

## 12. Type Signatures Summary

### FnPtr Module

| Intrinsic | Type Signature |
|-----------|----------------|
| `FnPtr.ofFunction` | `('T -> 'R) -> FnPtr<'T, 'R>` |
| `FnPtr.invoke` | `FnPtr<'T, 'R> -> 'T -> 'R` |
| `FnPtr.isNull` | `FnPtr<'T, 'R> -> bool` |
| `FnPtr.null` | `unit -> FnPtr<'T, 'R>` |

### Signal Module

| Intrinsic | Type Signature |
|-----------|----------------|
| `Signal.create` | `'T -> Signal<'T>` |
| `Signal.get` | `Signal<'T> -> 'T` |
| `Signal.set` | `Signal<'T> -> 'T -> unit` |
| `Signal.update` | `Signal<'T> -> ('T -> 'T) -> unit` |

### Effect Module

| Intrinsic | Type Signature |
|-----------|----------------|
| `Effect.create` | `FnPtr<unit, unit> -> Effect` |
| `Effect.createWithCleanup` | `FnPtr<unit, FnPtr<unit, unit>> -> Effect` |
| `Effect.dispose` | `Effect -> unit` |

### Memo Module

| Intrinsic | Type Signature |
|-----------|----------------|
| `Memo.create` | `FnPtr<unit, 'T> -> Memo<'T>` |
| `Memo.get` | `Memo<'T> -> 'T` |

### Batch Module

| Intrinsic | Type Signature |
|-----------|----------------|
| `Batch.run` | `FnPtr<unit, unit> -> unit` |

## 13. Comparison with SolidJS

| SolidJS | CCS Native | Notes |
|---------|-------------|-------|
| `createSignal(v)` | `Signal.create v` | Same semantics |
| `signal()` (getter) | `Signal.get signal` | Explicit call |
| `setSignal(v)` | `Signal.set signal v` | Same semantics |
| `createEffect(fn)` | `Effect.create (FnPtr.ofFunction fn)` | Requires top-level function |
| `createMemo(fn)` | `Memo.create (FnPtr.ofFunction fn)` | Requires top-level function |
| `batch(fn)` | `Batch.run (FnPtr.ofFunction fn)` | Same semantics |

**Key Difference:** CCS requires function pointers from top-level functions instead of closures. This enables native compilation without a closure runtime.

## 14. Example: Counter Application

```fsharp
module Counter

// State
let count = Signal.create 0

// Derived value
let countDoubled () = Signal.get count * 2
let doubled = Memo.create (FnPtr.ofFunction countDoubled)

// Side effect
let logChanges () =
    let c = Signal.get count
    let d = Memo.get doubled
    Console.writeln (sprintf "Count: %d, Doubled: %d" c d)

let logger = Effect.create (FnPtr.ofFunction logChanges)

// Actions
let increment () = Signal.update count ((+) 1)
let decrement () = Signal.update count (fun x -> x - 1)
let reset () = Signal.set count 0

// Entry point
[<EntryPoint>]
let main _ =
    increment ()  // Logs: "Count: 1, Doubled: 2"
    increment ()  // Logs: "Count: 2, Doubled: 4"
    reset ()      // Logs: "Count: 0, Doubled: 0"
    0
```

## 15. Normative Requirements

1. **CCS SHALL** add `NTUfnptr` to the NTU kind enumeration
2. **CCS SHALL** add `FnPtr`, `Signal`, `Effect`, `Memo`, `Batch` to `IntrinsicModule`
3. **CCS SHALL** type-check these intrinsics according to signatures in Section 12
4. **Alex SHALL** generate correct MLIR for function pointer operations
5. **Alex SHALL** generate runtime calls for signal/effect/memo operations
6. **The runtime SHALL** implement dependency tracking as specified in Section 8
7. **Function pointers SHALL** only be created from top-level functions (no closures)
8. **Signal notifications SHALL** respect batching boundaries
