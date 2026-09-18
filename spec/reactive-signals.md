---
title: "Reactive Signals"
weight: 420
category: Semantics
status: normative
---

> **Status**: Revised
> **Normative**: This chapter normatively specifies the **surface API** and its **desugaring**. The underlying reactive semantics are normative in [Observable Computation](observable-computation.md) and [Incremental Computation](incremental-computation.md); this chapter does not restate them.
> **Last Updated**: 2026-09-18

## 1. Overview

Reactive Signals is a developer-facing surface API providing fine-grained reactivity in the style of SolidJS and TanStack Store. It is the **signal scaffolding** that connects a front end — a native renderer or a WebView/JavaScript front end — to the application core. Borrowing this broad, widely-understood convention is a deliberate adoption choice: developers arriving from the SolidJS/React ecosystem find a familiar `Signal`/`Memo`/`Effect` vocabulary.

Signals are a **thin surface layer**, not a separate reactive engine. Each construct desugars to the `Observable<'T>` and `Incremental<'T>` intrinsics:

- `Signal<'T>` is a settable reactive source — the graph's input leaf.
- `Memo<'T>` is a derived, cached, cutoff-bearing value — an `Incremental<'T>`.
- `Effect` is a side-effecting computation that re-runs on change — a demanded sink on the graph.

Two consequences follow from being a surface over the intrinsics, and they are the substance of this revision:

1. **Reactive callbacks are flat closures, not function pointers.** A `Memo` or `Effect` body is an ordinary closure that may capture signals and local state ([Closure Representation](closure-representation.md)). Captures describe its environment; tracked reads determine its reactive dependencies. These sets are not generally identical. This preserves closure ergonomics without the top-level-function restriction of the earlier formulation.
2. **The reactive plan is compiler-visible in the [Program Semantic Graph](program-semantic-graph.md).** Read and effect analysis determines static dependencies and the operations that select dynamic dependencies. Target lowering specializes proven static structure and realizes the remaining state and dynamic graph instances. No mandatory global `CurrentTracking` mechanism or generic runtime signal table is prescribed.

`FnPtr` (function pointers) is retained only as a **C FFI interop primitive** for platform callbacks (event loops, Wayland/GTK listeners); it is **not** the reactive callback mechanism. See [§9](#9-function-pointers-are-for-ffi-not-reactivity).

## 2. Mapping to the Intrinsics

| Signals surface | Desugars to | Semantics specified in |
|---|---|---|
| `Signal<'T>` (settable cell) | Settable source leaf: `set` emits invalidation (Observable side); `get` registers a dependency (Incremental side) | [Observable](observable-computation.md) / [Incremental](incremental-computation.md) |
| `Memo<'T>` | `Incremental<'T>` node; tracked reads establish dependencies; cutoff from `'T : equality` | [Incremental Computation](incremental-computation.md) |
| `Effect` | Always-demanded `Incremental` sink (an observer that performs effects) | [Incremental §6.3](incremental-computation.md) |
| `Batch` | Stabilization-boundary control (coalesce to one stabilization) | [Incremental §6.2](incremental-computation.md) |
| `Store<'T>` | Per-field signals (sugar over `Signal`) | this chapter |
| Automatic dependency tracking | Read/effect analysis over the typed graph, including closure environments and called computations | [§8](#8-dependency-tracking-by-capture) |

Because both ends are intrinsic, a `Signal` driving a `Memo` can lower directly through the shared reactive plan without a mandatory separate library bridge ([Incremental + Observable](incremental-computation.md#incremental--observable)). The selected target still realizes the dependency notifications and runtime state the program requires.

## 3. Signal — Settable Reactive Source

A `Signal<'T>` is a settable cell at the root of a reactive graph. Reading it inside a reactive scope creates a dependency; writing it pushes an invalidation to dependents.

```fsharp
type Signal<'T>

val create : 'T -> Signal<'T>
val get    : Signal<'T> -> 'T            // within a reactive scope, registers a dependency
val set    : Signal<'T> -> 'T -> unit    // emits invalidation to dependents if changed
val update : Signal<'T> -> ('T -> 'T) -> unit
```

**Desugaring.** A `Signal<'T>` is a settable source leaf. `set` performs an `Observable`-style emission (invalidation) to the dependent `Incremental` nodes; a tracked read within a `Memo`/`Effect` computation establishes a dependency represented by the PSG's reactive plan. Capturing a signal handle without reading it does not establish that dependency. Change detection on `set` uses the same `'T : equality` cutoff as `Incremental`.

```fsharp
let count = Signal.create 0
let name  = Signal.create "World"
```

## 4. Memo — Derived Cached Value

A `Memo<'T>` is a derived value that recomputes when its dependencies change and caches otherwise. It **is** an `Incremental<'T>`.

```fsharp
type Memo<'T>

val create : (unit -> 'T) -> Memo<'T>    // flat closure; tracked reads establish dependencies
val get    : Memo<'T> -> 'T               // demand
 
```

**Desugaring.** `Memo.create f` constructs an `Incremental<'T>` node whose recompute function is the flat closure `f`. The compiler analyzes tracked reads, including those reached through helper calls and captured aggregates, to represent dependencies (applicative when fixed, monadic when selected by runtime values — see [Incremental §4.1](incremental-computation.md)). Lexical captures alone do not determine the active read set. Cutoff is structural equality on `'T`, or an `[<IncrementalCutoff>]` predicate.

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

**Desugaring.** `Effect.create f` registers `f` (a flat closure) as an always-demanded sink. Its tracked signal/memo reads determine dependencies. The effect runs initially, then re-runs when its dependencies change under the stabilization discipline. `createWithCleanup` returns a cleanup closure that runs before each re-execution and on disposal. An effect's `unit` result is not a value cutoff that permits suppressing its required side effects.

**Lifetime.** An effect's lifetime is bounded by its enclosing actor or region and may end earlier through `dispose`. Disposal detaches the effect and runs its cleanup; it does not depend on a finalizer. In an actor context, Prospero retiring the actor disposes its remaining effects. Native captured storage follows the region lifetime; the JavaScript target uses host-managed storage while preserving deterministic logical cleanup.

> **[Not yet specified]** The detailed ownership and reclamation rules for repeatedly replaced dynamic subgraphs, ordering among multiple cleanups, cleanup failure, and pending asynchronous work remain open. An actor-lifetime arena alone does not establish early reclamation of a removed child scope. The lifetime ordering obligations in [Memory Regions](memory-regions.md#lifetime-constraints) must also be satisfied.

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

<a id="8-dependency-tracking-by-capture"></a>

## Dependency Tracking by Reads and Effects

Capture analysis supplies environment and lifetime information; dependency analysis additionally examines tracked reads and effects. The earlier formulation's generic runtime table and `CurrentTracking` global are not the required mechanism:

- A closure may capture a signal only to install a later event handler, capture an aggregate containing several signals, or call a helper that performs tracked reads. Captures and active read dependencies SHALL NOT be equated without establishing that relationship.
- **Applicative tracking** of a proven fixed set of read dependencies yields static edges. Analysis includes the relevant behavior of called computations and access through captured values.
- **Dynamic tracking** represents dependencies selected by runtime values, including conditional reads and runtime-selected sources. The PSG carries the plan for selection and graph construction; its runtime instances carry the active dependencies, cached values and ownership state required by that plan ([Incremental §4.1](incremental-computation.md)). Static specialization can remove this state only where its absence preserves behavior.

The dependency plan SHALL account for every observation needed to justify memo reuse; capture layout alone establishes neither purity nor the immutability of referenced storage. A supported dynamic realization SHALL preserve the required active-dependency behavior. Unsupported cases SHALL be diagnosed rather than silently treated as fixed dependencies.

This retains compiler visibility for fusion, cutoff reasoning and target-specific lowering without identifying one compiler graph node with every runtime instance. It does not require a process-global tracking context, a managed native heap, or a separate library reactive engine.

> **[Not yet specified]** The admitted interprocedural read/effect analysis, representation of dynamic dependency selection and diagnostic boundary require further specification and conformance cases. This chapter does not claim that arbitrary closures can be resolved by lexical capture enumeration alone.

## 9. Function Pointers Are for FFI, Not Reactivity

Reactive callbacks (`Memo`, `Effect`, `Batch`, `Store.subscribe`) take **flat closures**. They may capture signals and local state; read/effect analysis uses that environment together with the callback's operations to establish dependencies. They are never `FnPtr` values, and there is no top-level-function restriction.

`FnPtr<'F>` remains in the language as a **C FFI interop primitive** — the representation for passing a function address across a C boundary (platform event loops, Wayland/GTK listener structs). Its normative home is the [FFI Boundary](ffi-boundary.md) chapter, not this one. A platform callback (an `FnPtr`-level C shim, ideally Farscape-generated) typically does nothing more than `set` a `Signal`; from that point the reactive graph is closures and PSG nodes. `FnPtr` lives at the OS edge; the reactive layer above it is closures.

> **Migration note.** The prior version of this chapter specified `FnPtr.ofFunction` callbacks, a runtime signal table, and top-level-only effects. Those are superseded. `FnPtr`'s detailed specification lives in [FFI Boundary](ffi-boundary.md).

## 10. UI Scaffolding: Front End ↔ Core

Signals are the scaffolding between the application core and a front end, and the surface API is identical across targets — which is the point.

- **Native renderer.** Flat closures lower through the native target pathway (the LLVM pathway off the portable middle end — see [Closure Representation §6.3](closure-representation.md) and [Backend Lowering §4.2](backend-lowering-architecture.md)) to region-allocated closure records plus a function address. Signal writes drive re-render through the same stabilization model that governs any `Incremental` graph; a render function is an `Effect` demanded by the frame boundary.
- **WebView / JavaScript front end.** Through JSIR — Composer's JavaScript-as-MLIR backend — flat closures lower to native JavaScript closures (captured scope is exactly what a JS function object carries). The *same* `Signal`/`Memo`/`Effect` source therefore compiles to JavaScript, enabling interop with JS-side reactive libraries (e.g. SolidJS) and genuine frontend/backend consistency. This target-polymorphism is the reason the reactive primitive is a closure rather than an `FnPtr`: a closure rides JSIR's bidirectional MLIR↔JavaScript mapping idiomatically, whereas a raw function pointer has no natural JavaScript form.
- **Core ↔ front-end transport.** State crossing the native-core/WebView boundary is carried by BAREWire and can update a local signal mirror. Encoding alone does not give the two sides one synchronous stabilization boundary. Ordering, resynchronization and failure behavior belong to the transport/application contract and are **[Not yet specified]** here.

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
| Required generic runtime signal table | PSG reactive plan with target-specific state and dynamic instances |
| Required `CurrentTracking` global for dependency tracking | Read/effect analysis and supported static or dynamic realization |
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
    increment ()
    increment ()
    reset ()
    0
```

The example shows source, derivation and effect construction. Exact logging order and whether intermediate values are observed depend on the enclosing stabilization boundary; the remaining timing contract is recorded in [Incremental §6.2](incremental-computation.md#62-stabilization-scope).

## 14. Comparison with SolidJS

| SolidJS | Clef Signals | Notes |
|---|---|---|
| `createSignal(v)` | `Signal.create v` | Settable reactive source |
| `signal()` (getter) | `Signal.get s` | Explicit call; registers dependency in a reactive scope |
| `setSignal(v)` | `Signal.set s v` | Change-filtered invalidation |
| `createMemo(fn)` | `Memo.create fn` | `fn` is a closure; tracked reads establish dependencies |
| `createEffect(fn)` | `Effect.create fn` | `fn` is a closure; no top-level restriction |
| `batch(fn)` | `Batch.run fn` | Single stabilization boundary |

Clef adopts SolidJS's familiar closure-based vocabulary; this table is not a claim of complete semantic equivalence. Clef preserves reactive intent in the PSG and realizes it per target. Native resource cleanup does not require GC; JavaScript storage remains host-managed. A Solid adapter must establish the admitted equality, dependency, timing and cleanup behavior rather than infer compatibility from API spelling. The unresolved timing questions are recorded in [Incremental §6.2](incremental-computation.md#62-stabilization-scope).

## 15. Normative Requirements

1. **Surface, not engine**: `Signal`, `Memo`, `Effect`, `Batch`, and `Store` SHALL desugar to `Observable<'T>` / `Incremental<'T>`; their reactive semantics SHALL be those specified in the corresponding intrinsic chapters.
2. **Closures, not function pointers**: Reactive callbacks SHALL be flat closures. The compiler SHALL NOT require `FnPtr` for any reactive callback, and SHALL NOT restrict effects/memos to top-level functions.
3. **Read-based tracking**: The PSG reactive plan SHALL be derived from tracked reads and effect analysis, including relevant closure environments and called computations. Lexical capture identity alone SHALL NOT establish dependency completeness. Static specialization and supported dynamic realization SHALL preserve those dependencies; no mandatory global tracking context or generic runtime signal table is prescribed.
4. **Memo is Incremental**: `Memo.create` SHALL construct an `Incremental<'T>` node, with cutoff from `'T : equality` or an `[<IncrementalCutoff>]` predicate.
5. **Signal set is invalidation**: `Signal.set` SHALL emit an invalidation to dependents per the Observable/Incremental model, subject to change detection.
6. **Batching**: `Batch.run` SHALL coalesce contained writes into a single stabilization boundary.
7. **Deterministic disposal**: Effect and subscription lifetimes SHALL be tied to the enclosing actor/region with deterministic release; no GC or finalizer SHALL be required to dispose.
8. **Target parity**: The `Signal`/`Memo`/`Effect` surface SHALL compile to both the native target (the LLVM target pathway) and the JavaScript target (the JSIR target pathway) from the same source, with reactive callbacks lowering to region-allocated closures and JavaScript closures respectively. LLVM and JSIR are peer target pathways off the portable middle end (as is CIRCT for FPGA); the middle end commits to no target, and each pathway realizes the flat closure through its own mechanism.
9. **FnPtr scope**: `FnPtr` SHALL be confined to C FFI interop; it SHALL NOT appear in the reactive API surface.

## References

- Solid contributors. *SolidJS — fine-grained reactivity.* https://www.solidjs.com
- TanStack. *Store — framework-agnostic reactive store.* https://tanstack.com/store
- Haaser, G., et al. *FSharp.Data.Adaptive* (`cval`/`aval`). https://github.com/fsprojects/FSharp.Data.Adaptive
