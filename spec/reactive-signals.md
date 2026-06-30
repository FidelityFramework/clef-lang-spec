---
title: "Reactive Signals"
weight: 420
category: Semantics
status: normative
---

> **Status**: Revised
> **Normative**: This chapter normatively specifies the **surface API** and its **desugaring**. The underlying reactive semantics are normative in [Observable Computation](observable-computation.md) and [Incremental Computation](incremental-computation.md); this chapter does not restate them.
> **Last Updated**: 2026-06-14

## 1. Overview

Reactive Signals is a developer-facing surface API providing fine-grained reactivity in the style of SolidJS and TanStack Store. It is the **signal scaffolding** that connects a front end — a native renderer or a WebView/JavaScript front end — to the application core. Borrowing this broad, widely-understood convention is a deliberate adoption choice: developers arriving from the SolidJS/React ecosystem find a familiar `Signal`/`Memo`/`Effect` vocabulary.

Signals are a **thin surface layer**, not a separate reactive engine. Each construct desugars to the `Observable<'T>` and `Incremental<'T>` intrinsics:

- `Signal<'T>` is a settable reactive source — the graph's input leaf.
- `Memo<'T>` is a derived, cached, cutoff-bearing value — an `Incremental<'T>`.
- `Effect` is a side-effecting computation that re-runs on change — a demanded sink on the graph.

Two consequences follow from being a surface over the intrinsics, and they are the substance of this revision:

1. **Reactive callbacks are flat closures, not function pointers.** A `Memo` or `Effect` body is an ordinary closure that may capture signals and local state ([Closure Representation](closure-representation.md)). The set of signals it reads is its capture set, and the capture set *is* its dependency-edge set. This restores the closure ergonomics SolidJS depends on and removes the top-level-function restriction of the earlier formulation.
2. **The dependency graph is the [Program Semantic Graph](program-semantic-graph.md), not a runtime signal table.** Dependency tracking is the compile-time capture analysis already used by `Incremental<'T>`; there is no runtime slot table, no `CurrentTracking` global, and no manual subscription bookkeeping.

`FnPtr` (function pointers) is retained only as a **C FFI interop primitive** for platform callbacks (event loops, Wayland/GTK listeners); it is **not** the reactive callback mechanism. See [§9](#9-function-pointers-are-for-ffi-not-reactivity).

## 2. Mapping to the Intrinsics

| Signals surface | Desugars to | Semantics specified in |
|---|---|---|
| `Signal<'T>` (settable cell) | Settable source leaf: `set` emits invalidation (Observable side); `get` registers a dependency (Incremental side) | [Observable](observable-computation.md) / [Incremental](incremental-computation.md) |
| `Memo<'T>` | `Incremental<'T>` node; closure captures are dependency edges; cutoff from `'T : equality` | [Incremental Computation](incremental-computation.md) |
| `Effect` | Always-demanded `Incremental` sink (an observer that performs effects) | [Incremental §6.3](incremental-computation.md) |
| `Batch` | Stabilization-boundary control (coalesce to one stabilization) | [Incremental §6.2](incremental-computation.md) |
| `Store<'T>` | Per-field signals (sugar over `Signal`) | this chapter |
| Automatic dependency tracking | Flat-closure capture analysis (compile time) | [Closure Representation](closure-representation.md) |

Because both ends are intrinsic, a `Signal` driving a `Memo` lowers with the same fusion the intrinsics already specify ([Incremental §11.3](incremental-computation.md)); no library bridge or callback indirection is materialized.

## 3. Signal — Settable Reactive Source

A `Signal<'T>` is a settable cell at the root of a reactive graph. Reading it inside a reactive scope creates a dependency; writing it pushes an invalidation to dependents.

```fsharp
type Signal<'T>

val create : 'T -> Signal<'T>
val get    : Signal<'T> -> 'T            // within a reactive scope, registers a dependency
val set    : Signal<'T> -> 'T -> unit    // emits invalidation to dependents if changed
val update : Signal<'T> -> ('T -> 'T) -> unit
```

**Desugaring.** A `Signal<'T>` is a settable source leaf. `set` performs an `Observable`-style emission (invalidation) to the dependent `Incremental` nodes; when read within a `Memo`/`Effect` closure, the read is a capture and becomes a dependency edge in the PSG. Change detection on `set` uses the same `'T : equality` cutoff as `Incremental`.

```fsharp
let count = Signal.create 0
let name  = Signal.create "World"
```

## 4. Memo — Derived Cached Value

A `Memo<'T>` is a derived value that recomputes when its dependencies change and caches otherwise. It **is** an `Incremental<'T>`.

```fsharp
type Memo<'T>

val create : (unit -> 'T) -> Memo<'T>    // the thunk is a flat closure; its captures are the edges
val get    : Memo<'T> -> 'T               // demand
```

**Desugaring.** `Memo.create f` constructs an `Incremental<'T>` node whose recompute function is the flat closure `f`. The signals and memos that `f` reads are its captures, which the compiler records as dependency edges (applicative when unconditional, monadic when conditional — see [Incremental §4.1](incremental-computation.md)). Cutoff is structural equality on `'T`, or an `[<IncrementalCutoff>]` predicate.

```fsharp
let doubled = Memo.create (fun () -> Signal.get count * 2)   // captures `count`
```

## 5. Effect — Reactive Side Effect

An `Effect` runs when its tracked dependencies change. It is a demanded `Incremental` sink whose output is a side effect rather than a value.

```fsharp
type Effect

val create            : (unit -> unit) -> Effect
val createWithCleanup : (unit -> (unit -> unit)) -> Effect   // returns a cleanup closure
val dispose           : Effect -> unit
```

**Desugaring.** `Effect.create f` registers `f` (a flat closure) as an always-demanded sink. The signals/memos `f` reads are its dependencies. The effect runs once to establish dependencies, then re-runs when any dependency changes. `createWithCleanup` returns a cleanup closure that runs before each re-execution and on disposal.

**Lifetime.** An effect's lifetime is its enclosing actor or region. `dispose` is deterministic region/subscription release; there is no finalizer and no GC. In an actor context, Prospero retiring the actor disposes the effect and frees its captured state with the arena.

```fsharp
let logger =
    Effect.create (fun () ->
        let c = Signal.get count          // captures `count`
        Console.writeln (sprintf "Count: %d" c))
```

## 6. Batch — Stabilization Control

```fsharp
val run : (unit -> unit) -> unit
```

`Batch.run f` defers invalidation propagation until `f` completes, then performs a single stabilization. It maps directly to controlling the stabilization boundary specified in [Incremental §6.2](incremental-computation.md); multiple signal writes within the batch produce one stabilization wave, preventing intermediate inconsistency and redundant effect runs.

```fsharp
Batch.run (fun () ->
    Signal.set firstName "John"
    Signal.set lastName  "Doe")          // effects run once, after both writes
```

## 7. Store — Nested Reactive State (Optional Extension)

```fsharp
type Store<'T>

val create    : 'T -> Store<'T>
val state     : Store<'T> -> 'T
val setState  : Store<'T> -> ('T -> 'T) -> unit
val subscribe : Store<'T> -> (unit -> unit) -> (unit -> unit)   // returns an unsubscribe closure
```

A `Store<'T>` provides nested reactive objects in the TanStack Store style. Each reactive field desugars to a `Signal`; `subscribe` registers an effect and returns an unsubscribe closure whose invocation (or whose owning region's release) detaches it. `Store` is sugar; it introduces no mechanism beyond `Signal` and `Effect`.

## 8. Dependency Tracking by Capture

The earlier formulation maintained a runtime signal table (slots, dirty flags, subscriber lists) with a `CurrentTracking` global set during effect execution. **That runtime machinery is removed.** Dependency tracking is the flat-closure capture analysis:

- The signals and memos a `Memo`/`Effect` closure reads are its **captures**; the capture set is the node's **dependency-edge set** in the PSG.
- **Applicative tracking** (the closure reads a fixed set of signals unconditionally) yields static edges known at compile time.
- **Dynamic tracking** (the closure reads signals conditionally, so the dependency set varies per run) is the monadic case: the subgraph is rebuilt on demand from thunks and flat closures allocated in the enclosing region, exactly as specified for dynamic incremental subgraphs ([Incremental §4.1](incremental-computation.md)). No garbage collector is involved; lifetime is region/supervision-bounded.

This makes the reactive graph transparent to the compiler (it is the PSG), which is what permits fusion, cutoff proofs, and target-specific lowering — none of which a runtime signal table could provide.

## 9. Function Pointers Are for FFI, Not Reactivity

Reactive callbacks (`Memo`, `Effect`, `Batch`, `Store.subscribe`) take **flat closures**. They may capture signals and local state; that capture is what makes automatic dependency tracking work. They are never `FnPtr` values, and there is no top-level-function restriction.

`FnPtr<'F>` remains in the language as a **C FFI interop primitive** — the representation for passing a function address across a C boundary (platform event loops, Wayland/GTK listener structs). Its normative home is the [FFI Boundary](ffi-boundary.md) chapter, not this one. A platform callback (an `FnPtr`-level C shim, ideally Farscape-generated) typically does nothing more than `set` a `Signal`; from that point the reactive graph is closures and PSG nodes. `FnPtr` lives at the OS edge; the reactive layer above it is closures.

> **Migration note.** The prior version of this chapter specified `FnPtr.ofFunction` callbacks, a runtime signal table, and top-level-only effects. Those are superseded. `FnPtr`'s detailed specification lives in [FFI Boundary](ffi-boundary.md).

## 10. UI Scaffolding: Front End ↔ Core

Signals are the scaffolding between the application core and a front end, and the surface API is identical across targets — which is the point.

- **Native renderer.** Flat closures lower through LLVM to region-allocated closure records plus a function pointer. Signal writes drive re-render through the same stabilization model that governs any `Incremental` graph; a render function is an `Effect` demanded by the frame boundary.
- **WebView / JavaScript front end.** Through JSIR — Composer's JavaScript-as-MLIR backend — flat closures lower to native JavaScript closures (captured scope is exactly what a JS function object carries). The *same* `Signal`/`Memo`/`Effect` source therefore compiles to JavaScript, enabling interop with JS-side reactive libraries (e.g. SolidJS) and genuine frontend/backend consistency. This target-polymorphism is the reason the reactive primitive is a closure rather than an `FnPtr`: a closure rides JSIR's bidirectional MLIR↔JavaScript mapping idiomatically, whereas a raw function pointer has no natural JavaScript form.
- **Core ↔ front-end transport.** State crossing the native-core/WebView boundary is carried by BAREWire; a `Signal` on one side is mirrored to the other without a bespoke serialization layer.

## 11. Event-Loop and Platform Integration

Signals connect to external events at the platform boundary. The platform callback is a C-level shim (`FnPtr`); its body sets a `Signal`, after which propagation is entirely closures and PSG nodes.

```fsharp
// Platform shim (C FFI boundary): on readable fd, set a signal.
let onSocketReadable (fd: int) : unit =
    Signal.set socketData (Sockets.recv fd buffer 1024 0)

EventLoop.onReadable socketFd onSocketReadable   // FnPtr only at this OS edge
```

The same pattern applies to GLib/GTK signal connection and to a Wayland `wl_surface::frame` callback driving an animation `Signal`.

## 12. What Changes From the Prior Formulation

| Prior (FnPtr + runtime table) | Now (closures + PSG) |
|---|---|
| `FnPtr.ofFunction` for every callback | Flat closure capturing signals and local state |
| Effects restricted to top-level functions | Effects capture local state freely |
| Runtime signal table (slots / dirty / subscribers) | PSG dependency graph |
| `CurrentTracking` global for dependency tracking | Compile-time capture analysis |
| Manual `dispose`/unsubscribe bookkeeping | Region/actor-scoped deterministic release |
| `dlsym` symbol resolution for callbacks | Only at the C FFI boundary (Farscape-generated) |

## 13. Example: Counter Application

```fsharp
module Counter

// State
let count = Signal.create 0

// Derived value (closure captures `count`)
let doubled = Memo.create (fun () -> Signal.get count * 2)

// Side effect (closure captures `count` and `doubled`)
let logger =
    Effect.create (fun () ->
        Console.writeln (sprintf "Count: %d, Doubled: %d"
            (Signal.get count) (Memo.get doubled)))

// Actions
let increment () = Signal.update count ((+) 1)
let reset ()     = Signal.set count 0

[<EntryPoint>]
let main _ =
    increment ()   // logs "Count: 1, Doubled: 2"
    increment ()   // logs "Count: 2, Doubled: 4"
    reset ()       // logs "Count: 0, Doubled: 0"
    0
```

## 14. Comparison with SolidJS

| SolidJS | Clef Signals | Notes |
|---|---|---|
| `createSignal(v)` | `Signal.create v` | Same semantics |
| `signal()` (getter) | `Signal.get s` | Explicit call; registers dependency in a reactive scope |
| `setSignal(v)` | `Signal.set s v` | Same semantics |
| `createMemo(fn)` | `Memo.create fn` | `fn` is a closure; captures are dependencies |
| `createEffect(fn)` | `Effect.create fn` | `fn` is a closure; no top-level restriction |
| `batch(fn)` | `Batch.run fn` | Single stabilization boundary |

Unlike the earlier Clef formulation, **closures are used exactly as in SolidJS** — the prior departure to function pointers is removed. The difference from SolidJS is in the implementation, not the surface: dependency tracking is compile-time capture analysis over the PSG rather than a runtime tracking stack, and there is no GC.

## 15. Normative Requirements

1. **Surface, not engine**: `Signal`, `Memo`, `Effect`, `Batch`, and `Store` SHALL desugar to `Observable<'T>` / `Incremental<'T>`; their reactive semantics SHALL be those specified in the corresponding intrinsic chapters.
2. **Closures, not function pointers**: Reactive callbacks SHALL be flat closures. The compiler SHALL NOT require `FnPtr` for any reactive callback, and SHALL NOT restrict effects/memos to top-level functions.
3. **Capture-based tracking**: Dependency tracking SHALL be derived from flat-closure capture analysis. No runtime signal table SHALL be required.
4. **Memo is Incremental**: `Memo.create` SHALL construct an `Incremental<'T>` node, with cutoff from `'T : equality` or an `[<IncrementalCutoff>]` predicate.
5. **Signal set is invalidation**: `Signal.set` SHALL emit an invalidation to dependents per the Observable/Incremental model, subject to change detection.
6. **Batching**: `Batch.run` SHALL coalesce contained writes into a single stabilization boundary.
7. **Deterministic disposal**: Effect and subscription lifetimes SHALL be tied to the enclosing actor/region with deterministic release; no GC or finalizer SHALL be required to dispose.
8. **Target parity**: The `Signal`/`Memo`/`Effect` surface SHALL compile to both native (LLVM) and JavaScript (JSIR) targets from the same source, with reactive callbacks lowering to region-allocated closures and JavaScript closures respectively.
9. **FnPtr scope**: `FnPtr` SHALL be confined to C FFI interop; it SHALL NOT appear in the reactive API surface.

## References

- Solid contributors. *SolidJS — fine-grained reactivity.* https://www.solidjs.com
- TanStack. *Store — framework-agnostic reactive store.* https://tanstack.com/store
- Haaser, G., et al. *FSharp.Data.Adaptive* (`cval`/`aval`). https://github.com/fsprojects/FSharp.Data.Adaptive
