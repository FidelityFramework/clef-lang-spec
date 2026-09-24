---
title: "Reactive Signals"
weight: 420
category: Semantics
status: normative
---

> **Status**: Revised
> **Normative**: This chapter normatively specifies the **surface API** and its **desugaring**. The underlying reactive semantics are normative in [Observable Computation](observable-computation.md) and [Incremental Computation](incremental-computation.md); this chapter does not restate them.
> **Last Updated**: 2026-09-24

## 1. Overview

Reactive Signals is a developer-facing surface API for fine-grained reactivity. It is the **signal scaffolding** that connects a front end to a native renderer or a WebView/JavaScript front end, and to the application core. A familiar `Signal`/`Memo`/`Effect` vocabulary helps developers arriving from existing ecosystems navigate the API.

Signals are a **thin surface layer**, not a separate reactive engine. Each construct desugars to the `Observable<'T>` and `Incremental<'T>` intrinsics:

- `Signal<'T>` is a settable reactive source, the graph's input leaf.
- `Memo<'T>` is a derived, cached, cutoff-bearing value, an `Incremental<'T>`.
- `Effect` is a side-effecting computation that re-runs on change, a demanded sink on the graph.

Two consequences follow from being a surface over the intrinsics, and they are the substance of this revision:

1. **Reactive callbacks are flat closures.** A `Memo` or `Effect` body is an ordinary closure that may capture signals and local state ([Closure Representation](closure-representation.md)). Captures describe its environment; tracked reads determine its reactive dependencies. These sets are not generally identical. This preserves closure ergonomics without the top-level-function restriction of the earlier formulation.
2. **The reactive plan is compiler-visible in the [Program Semantic Graph](program-semantic-graph.md).** Read and effect analysis determines static dependencies and the operations that select dynamic dependencies. Target lowering specializes proven static structure and realizes the remaining state and dynamic graph instances. No mandatory global `CurrentTracking` mechanism or generic runtime signal table is prescribed.

## 2. Mapping to the Intrinsics

| | Signals surface | Desugars to | Semantics specified in |
|---|---|---|---|
| | `Signal<'T>` (settable cell) | Settable source leaf: `set` emits invalidation (Observable side); `get` registers a dependency (Incremental side) | [Observable](observable-computation.md) / [Incremental](incremental-computation.md) |
| | `Memo<'T>` | `Incremental<'T>` node; tracked reads establish dependencies; cutoff from `'T : equality` | [Incremental Computation](incremental-computation.md) |
| | `Effect` | Always-demanded `Incremental` sink (an observer that performs effects) | [Incremental §6.3](incremental-computation.md) |
| | `Batch` | Stabilization-boundary control (coalesce to one stabilization) | [Incremental §6.2](incremental-computation.md) |
| | `Store<'T>` | Per-field signals (sugar over `Signal`) | this chapter |
| | Automatic dependency tracking | Read/effect analysis over the typed graph, including closure environments and called computations | [§8](#8-dependency-tracking-by-capture) |

Because both ends are intrinsic, a `Signal` driving a `Memo` can lower directly through the shared reactive plan without a mandatory separate library bridge ([Incremental + Observable](incremental-computation.md#incremental--observable)). The selected target still realizes the dependency notifications and runtime state the program requires.

## 3. Signal: Settable Reactive Source

A `Signal<'T>` is a settable cell at the root of a reactive graph. Reading it inside a reactive scope creates a dependency; writing it pushes an invalidation to dependents.

```fsharp
type Signal<'T>

val create : 'T -> Signal<'T>
val get    : Signal<'T> -> 'T            // within a reactive scope, registers a dependency
val set    : Signal<'T> -> 'T -> unit    // emits invalidation to dependents if changed
val update : Signal<'T> -> ('T -> 'T) -> unit
```

**Desugaring.** A `Signal<'T>` is a settable source leaf. `set` performs an `Observable`-style emission (invalidation) to the dependent `Incremental` nodes; a tracked read within a `Memo`/`Effect` computation establishes a dependency represented by the PSG's reactive plan. Capturing a signal handle without reading it does not establish that dependency. Change detection on `set` uses the same `'T : equality` cutoff as `Incremental`.

**Read API.** The tracked read is expressed explicitly by the developer, either as a property access or a function call. A property access (e.g. `.Value` or `.current`) is zero-cost at runtime and is a common pattern in fine-grained reactive systems; a function call is semantically identical. Either form registers the dependency in a reactive scope. The spec uses a function call in its examples, but a property access is equally valid at the implementation level and avoids the allocation of a closure wrapper in some targets.

```fsharp
let count = Signal.create 0
let name  = Signal.create "World"
```

## 4. Memo: Derived Cached Value

A `Memo<'T>` is a derived value that recomputes when its dependencies change and caches otherwise. It **is** an `Incremental<'T>`.

```fsharp
type Memo<'T>

val create : (unit -> 'T) -> Memo<'T>    // flat closure; tracked reads establish dependencies
val get    : Memo<'T> -> 'T               // demand
  
```

**Desugaring.** `Memo.create f` constructs an `Incremental<'T>` node whose recompute function is the flat closure `f`. The compiler analyzes tracked reads, including those reached through helper calls and captured aggregates, to represent dependencies (applicative when fixed, monadic when selected by runtime values, see [Incremental §4.1](incremental-computation.md)). Lexical captures alone do not determine the active read set. Cutoff is structural equality on `'T`, or an `[<IncrementalCutoff>]` predicate.

A runtime may choose to represent a source and its derived computations with a single node type, distinguishing them only by mutability of the write handle. A source node accepts writes; a computed node does not. This keeps the combinator code monomorphic (no dispatch or hidden-class polymorphism per source/computed pair) and means that the observable surface area at runtime is smaller than the source-level vocabulary.

```fsharp
let doubled = Memo.create (fun () -> Signal.get count * 2)   // captures `count`
  
```

## 5. Effect: Reactive Side Effect

An `Effect` runs when its tracked dependencies change. It is a demanded `Incremental` sink whose output is a side effect rather than a value.

```fsharp
type Effect

val create            : (unit -> unit) -> Effect
val createWithCleanup : (unit -> (unit -> unit)) -> Effect   // returns a cleanup closure
val detach            : Effect -> unit    // detaches the effect from its reactive graph
```

**Desugaring.** `Effect.create f` registers `f` (a flat closure) as an always-demanded sink. Its tracked signal/memo reads determine dependencies. The effect runs initially, then re-runs when its dependencies change under the stabilization discipline. `createWithCleanup` returns a cleanup closure that runs before each re-execution and on detachment. An effect's `unit` result is not a value cutoff that permits suppressing its required side effects.

**Lifetime.** An effect's lifetime is bounded by its enclosing actor or region. In a scoped region, the runtime (or compiler for JSIR) automatically generates the teardown code that detaches every registered effect and runs its cleanup handlers. The developer does not call cleanup at every scope boundary. In an actor context, Prospero retiring the actor performs this teardown for all remaining effects. `detach` is available for the rare case where manual early teardown is needed (for example, canceling an effect before its enclosing scope ends), but the default path is automatic scoped cleanup. Native captured storage follows the region lifetime; the JavaScript target uses host-managed storage while preserving deterministic logical cleanup through compiler-generated teardown at scope boundaries. A separate `onCleanup` registration is available for cleanup callbacks that are not tied to a specific effect's re-execution.

> **[Not yet specified]** The detailed ownership and reclamation rules for repeatedly replaced dynamic subgraphs, ordering among multiple cleanups, cleanup failure, and pending asynchronous work remain open. An actor-lifetime arena alone does not establish early reclamation of a removed child scope. The lifetime ordering obligations in [Memory Regions](memory-regions.md#lifetime-constraints) must also be satisfied.

```fsharp
let logger =
    Effect.create (fun () ->
        let c = Signal.get count          // captures `count`
        Console.writeln (sprintf "Count: %d" c))
```

## 6. Batch: Stabilization Control

```fsharp
val run : (unit -> unit) -> unit
```

`Batch.run f` defers invalidation propagation until `f` completes, then performs a single stabilization. It maps directly to controlling the stabilization boundary specified in [Incremental §6.2](incremental-computation.md); multiple signal writes within the batch produce one stabilization wave, preventing intermediate inconsistency and redundant effect runs.

```fsharp
Batch.run (fun () ->
    Signal.set firstName "John"
    Signal.set lastName  "Doe")          // effects run once, after both writes
  
```

## 7. Store: Nested Reactive State (Optional Extension)

A `Store<'T>` is a single mutable record whose fields are independently observable. Updating one field triggers re-evaluation only of the selectors that read that field. This is the reactive alternative to field-level change notification in runtime managed environments: you define one record, and the framework provides per-field dependency tracking without boilerplate.

In runtime managed environments like the .NET CLR, field-level reactivity typically requires `INotifyPropertyChanged` (a manual change-notification protocol applied to every type) or eager evaluation of every dependent on any state change. `Store` avoids both patterns: each field desugars to a `Signal<'T>`, so a selector that reads `state.field` establishes a reactive dependency on just that field. Only those selectors re-run when that field changes.

```fsharp
type Store<'T>

val create    : 'T -> Store<'T>
val state     : Store<'T> -> 'T
val setState  : Store<'T> -> ('T -> 'T) -> unit
val subscribe : Store<'T> -> (unit -> unit) -> (unit -> unit)   // returns an unsubscribe closure
  
```

`create` builds the store and its per-field signal structure. `state` returns the current value. `setState` applies a transformation and emits invalidation to the dependent selectors of any changed field, with change detection via `'T : equality` cutoff. `subscribe` registers an effect and returns an unsubscribe closure; the closure's invocation (or whose owning region's release) detaches it.

A selector reads a single field through `state` and establishes a reactive dependency on that field alone. The selector desugars to an `Effect` backed by a `Signal` read. A record update via `setState` invalidates only the selectors that read the changed field, not all selectors on the store. This is sugar over `Signal` and `Effect`. It introduces no mechanism beyond them.

## 8. Dependency Tracking by Reads and Effects

Capture analysis supplies environment and lifetime information; dependency analysis additionally examines tracked reads and effects. The earlier formulation's generic runtime table and `CurrentTracking` global are not the required mechanism:

- A closure may capture a signal only to install a later callback, capture an aggregate containing several signals, or call a helper that performs tracked reads. Captures and active read dependencies SHALL NOT be equated without establishing that relationship.
- **Applicative tracking** of a proven fixed set of read dependencies yields static edges. Analysis includes the relevant behavior of called computations and access through captured values.
- **Dynamic tracking** represents dependencies selected by runtime values, including conditional reads and runtime-selected sources. The PSG carries the plan for selection and graph construction; its runtime instances carry the active dependencies, cached values and ownership state required by that plan ([Incremental §4.1](incremental-computation.md)). Static specialization can remove this state only where its absence preserves behavior.

The dependency plan SHALL account for every observation needed to justify memo reuse; capture layout alone establishes neither purity nor the immutability of referenced storage. A supported dynamic realization SHALL preserve the required active-dependency behavior. Unsupported cases SHALL be diagnosed rather than silently treated as fixed dependencies.

This retains compiler visibility for fusion, cutoff reasoning and target-specific lowering without identifying one compiler graph node with every runtime instance. It does not require a process-global tracking context, a managed native heap, or a separate library reactive engine.

**Dependency reconciliation.** During recomputation, reads are matched against the node's existing source list in order; only the diverged tail is reconciled. When a recomputation succeeds and the dependency set is unchanged, no edge is mutated. A stable graph is a zero-allocation path. A throwing evaluation must not touch the graph at all, so the tracking state is fully restored even on failure. This preserves edge stability across repeated successful evaluations.

> **[Not yet specified]** The admitted interprocedural read/effect analysis, representation of dynamic dependency selection and diagnostic boundary require further specification and conformance cases. This chapter does not claim that arbitrary closures can be resolved by lexical capture enumeration alone.

## 9. UI Scaffolding: Front End ↔ Core

Signals are the scaffolding between the application core and a front end, and the surface API is identical across targets. This is the point.

- **Native renderer.** Flat closures lower through the native target pathway (the LLVM pathway off the portable middle end, see [Closure Representation §6.3](closure-representation.md) and [Backend Lowering §4.2](backend-lowering-architecture.md)) to region-allocated closure records plus a function address. Signal writes drive re-render through the same stabilization model that governs any `Incremental` graph; a render function is an `Effect` demanded by the frame boundary.
- **WebView / JavaScript front end.** Through JSIR, Composer's JavaScript-as-MLIR backend, flat closures lower to native JavaScript closures (captured scope is exactly what a JS function object carries). The *same* `Signal`/`Memo`/`Effect` source therefore compiles to JavaScript, enabling interop with JS-side reactive libraries and genuine frontend/backend consistency. This target-polymorphism is the reason the reactive primitive is a closure rather than an `FnPtr`: a closure rides JSIR's bidirectional MLIR↔JavaScript mapping idiomatically, whereas a raw function pointer has no natural JavaScript form.
- **Core ↔ front-end transport.** State crossing the native-core/WebView boundary is carried by BAREWire and can update a local signal mirror. Encoding alone does not give the two sides one synchronous stabilization boundary. Ordering, resynchronization and failure behavior belong to the transport/application contract and are **[Not yet specified]** here.

## 10. Example: Counter Application

The example demonstrates the reactive signal flow through a pure domain transition. The model is the source of truth; signals carry changes to any front end.

```fsharp
type Model = { count: int; name: string }
type Msg = Increment | Decrement | SetName of string
```

A pure transition produces a new model:

```fsharp
let update (model: Model) (msg: Msg) =
    match msg with
    | Increment -> { model with count = model.count + 1 }
    | Decrement -> { model with count = model.count - 1 }
    | SetName n -> { model with name = n }
```

Signal updates follow the domain transition:

```fsharp
Signal.set countSignal resultModel.count
Signal.set nameSignal  resultModel.name
```

An effect reads signals and produces output, re-running on any change:

```fsharp
Effect.create (fun () ->
    let c = Signal.get countSignal
    let n = Signal.get nameSignal
    Console.writeLine (sprintf "Count: %d, Name: %s" c n))
```

The effect runs on each model update, but only the changed projections trigger front-end bindings. This is the core of the spec: domain transitions drive reactive signals, and signals drive updates to whatever front end (native, WebView, CLI, or TUI) subscribes to them.

## 11. Normative Requirements

1. **Surface, not engine**: `Signal`, `Memo`, `Effect`, `Batch`, and `Store` SHALL desugar to `Observable<'T>` / `Incremental<'T>`; their reactive semantics SHALL be those specified in the corresponding intrinsic chapters.
2. **Closures**: Reactive callbacks SHALL be flat closures. The compiler SHALL NOT require `FnPtr` for any reactive callback, and SHALL NOT restrict effects/memos to top-level functions.
3. **Read-based tracking**: The PSG reactive plan SHALL be derived from tracked reads and effect analysis, including relevant closure environments and called computations. Lexical capture identity alone SHALL NOT establish dependency completeness. Static specialization and supported dynamic realization SHALL preserve those dependencies; no mandatory global tracking context or generic runtime signal table is prescribed.
4. **Memo is Incremental**: `Memo.create` SHALL construct an `Incremental<'T>` node, with cutoff from `'T : equality` or an `[<IncrementalCutoff>]` predicate.
5. **Signal set is invalidation**: `Signal.set` SHALL emit an invalidation to dependents per the Observable/Incremental model, subject to change detection.
6. **Batching**: `Batch.run` SHALL coalesce contained writes into a single stabilization boundary. A practical propagation uses a three-level state per node: a source value change marks direct observers as *dirty*; downstream of a freshly-stale node, observers are marked *check*. Nodes at *check* that find an unchanged value do not propagate further. A batch increments a depth counter; writes within the batch mark nodes but defer the flush until the depth reaches zero. This is a push-staleness policy, not pull-on-read.
7. **Automatic scoped cleanup**: Effect and subscription lifetimes are managed by the enclosing actor or region. The compiler (for JSIR) and runtime generate the teardown code that detaches every registered effect and runs its cleanup handlers at scope exit. The developer does not call cleanup at every scope boundary. `detach` is available only when early manual teardown is needed. A practical implementation marks all nodes as invalidated before compacting each affected source's observer list in a single pass, so teardown is O(external observers) rather than O(nodes × external observers). No GC or finalizer is required.
8. **Target parity**: The `Signal`/`Memo`/`Effect` surface SHALL compile to both the native target (the LLVM target pathway) and the JavaScript target (the JSIR target pathway) from the same source, with reactive callbacks lowering to region-allocated closures and JavaScript closures respectively. LLVM and JSIR are peer target pathways off the portable middle end (as is CIRCT for FPGA); the middle end commits to no target, and each pathway realizes the flat closure through its own mechanism.
