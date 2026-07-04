---
title: "Observable Computation"
weight: 410
category: Semantics
status: normative
---

> **Normative specification for the `Observable<'T>` intrinsic type, push-based reactive observation, and its fusion into demand-driven computation in Clef compilation.**

## 1. Overview

Clef implements `Observable<'T>` as a compiler-known intrinsic type for push-based, producer-driven reactive observation. Unlike a library reactive framework layered on a managed runtime (such as .NET's `IObservable<'T>`/`IObserver<'T>` with `Subject` plumbing), `Observable<'T>` in Fidelity is not a runtime abstraction. It is a compile-time annotation that the [Program Semantic Graph](program-semantic-graph.md) preserves through lowering, enabling the compiler to place observer state in arena memory and to fuse an observable directly into the demand-driven computation it feeds.

`Observable<'T>` occupies the push pole of the spectrum of evaluation strategies that the compiler understands natively (the same spectrum specified in [Incremental Computation §1](incremental-computation.md)):

| Property | Observable&lt;'T&gt; | Cold&lt;'T&gt; | Lazy&lt;'T&gt; | Incremental&lt;'T&gt; |
|---|---|---|---|---|
| Deferred evaluation | No (push) | Yes | Yes | Yes (demand-driven) |
| Cached result | No | No | Yes | Yes |
| Dependency tracking | No | No | No | Yes |
| Invalidation | No | No | No | Yes |
| Cutoff (change detection) | No | No | No | Yes |
| Propagation bound | Unbounded | N/A | N/A | Bounded by cutoff |

An `Observable<'T>` is **opaque**: the producer decides when values arrive, and the compiler must assume every emission matters. It carries none of the structure (cache, dependency graph, cutoff) that makes `Incremental<'T>` transparent. This opacity is definitional, not a deficiency: an event source — user input, a sensor, a network packet — genuinely produces on its own clock, and there is nothing for the compiler to memoize or elide.

### 1.1 Relationship to Incremental

`Observable<'T>` is the push counterpart to the demand-driven `Incremental<'T>`. The producer drives an observable (it invokes the consumer); the consumer drives an incremental (it forces the recompute). They are not interchangeable and they are not ordered on a single quality axis — they sit at opposite ends of the evaluation spectrum and **compose**:

```fsharp
// An event source (push) driving a cached derivation (demand)
let sensorEvents : Observable<Reading> = sensorStream
let fused        : Incremental<Decision> =
    incremental {
        let! r = sensorEvents          // subscription becomes an invalidation edge
        return decide r
    }
```

Because both are intrinsic, the compiler fuses the observable subscription directly into the incremental node's invalidation trigger, eliminating the intermediate allocation and callback indirection a library bridge would require (see [Incremental Computation §11.3](incremental-computation.md)). The observable supplies *change events*; the incremental supplies *bounded, cached recomputation* in response.

The developer-facing `Signal`/`Memo`/`Effect` surface (see [Reactive Signals](reactive-signals.md)) is built on this pair: a `Signal` is a settable source, a `Memo` is an `Incremental`, and an `Effect` is a demanded sink.

### 1.2 Relationship to Continuations

An observer is a **continuation**. Subscribing installs the consumer's continuation with the producer; emitting invokes it. In continuation-passing terms the producer holds the consumer's continuation and applies it on each emission — the inverse of demand-driven evaluation, where the consumer holds and forces the producer's suspended continuation. The registered observer is represented as a flat closure (see [Closure Representation](closure-representation.md)); its captured environment is the consumer state the continuation resumes into.

The delimited-continuation substrate underlying `MailboxProcessor<'Msg>` and `Incremental<'T>` is the same substrate beneath `Observable<'T>`: the named intrinsic provides compositional identity that enables compiler-directed optimization, while the capture/resume mechanics exist at a lower level.

### 1.3 Relationship to Actors

In the Olivier/Prospero actor system, an `Observable<'T>` corresponds structurally to a producing actor and its supervised subscribers:

| Observable Concept | Actor Concept |
|---|---|
| Producer / source | Actor emitting messages |
| Observer (registered continuation) | Subscribing actor's receive handler |
| Subscription | Supervised link (Prospero) |
| Emission | Message send (BAREWire channel) |
| Subscription lifetime | Supervised link lifetime |
| Unsubscribe / teardown | Arena release on link retirement |

An observer's lifetime is the lifetime of its subscription. When Prospero retires the subscribing link, the observer closure and its captured state are deterministically freed with the arena. No garbage collection is involved, and no `Dispose` call is required.

### 1.4 Rationale for Intrinsic Status

The ingredients for reactive observation exist at the library level: a callback is a function, a subscriber list is a collection, and `MailboxProcessor<'Msg>` provides message delivery. These could be composed without compiler knowledge.

However, library-level composition forces observer closures onto a managed heap and inserts callback indirection that the compiler cannot see through. When `Observable<'T>` is intrinsic, the compiler can:

1. Place observer closures in the subscriber's arena, with subscription-scoped lifetime.
2. Fuse an observable into an `Incremental<'T>` invalidation trigger, removing the bridge allocation.
3. Lower emission as a direct flat-closure dispatch rather than a virtual `IObserver` call.
4. Track subscription lifetime through region inference, so unsubscribe is deterministic rather than finalizer-driven.

## 2. Type Definition

### 2.1 Core Type

```fsharp
type Observable<'T> = intrinsic
```

Unlike `Incremental<'T>`, `Observable<'T>` imposes **no `equality` constraint**: an observable does not perform cutoff and never compares emissions. Suppression of duplicate values (the push-side analogue of cutoff) is an explicit operator, not an implicit property of the type (see [§5](#5-operators)).

### 2.2 Hardware Targeting

`Observable<'T>` does **not** carry a hardware-target measure the way `Incremental<'T>` does. Because an observable is opaque — the compiler cannot bound its emissions or statically schedule them — it has no standalone accelerator lowering. Reactive event sources are lowered at the CPU/event boundary; accelerator computation is reached by *fusing* an observable into an `Incremental<'T>`, which carries the target measure and the bounded schedule (see [§7](#7-target-specific-lowering)).

### 2.3 NativeType Representation

```fsharp
type NativeType =
    // ...
    | TObservable of elementType: NativeType
```

## 3. Node Structure

### 3.1 Logical Fields

Each `Observable<'T>` node in the PSG carries the following logical fields. As with incremental nodes, these are PSG annotations that lower differently per context, not fixed runtime struct fields.

| Field | Type | Semantics |
|-------|------|-----------|
| `source` | producer reference | The origin of emissions (event source, upstream operator, or actor) |
| `observers` | continuation list | Registered observer closures, invoked on each emission |

An observable node carries **no** `value`, `stale`, `height`, or `cutoff` field. The absence of these is the structural expression of its position on the spectrum (§1): there is nothing to cache, no dependency DAG, and no change detection.

### 3.2 Arena Allocation

An observer closure is allocated in the **subscriber's** arena, and its lifetime equals the subscription's lifetime. This follows the lifetime inference model of [Memory Regions](memory-regions.md):

- **Level 1 (inferred):** The compiler determines that an observer closure outlives the `subscribe` call (it is invoked on later emissions), therefore it resides in the subscription's arena, not the call frame.
- **Level 2 (bounded):** The developer establishes a subscription within an actor; the compiler infers arena placement and subscription-scoped lifetime.
- **Level 3 (explicit):** The developer specifies the subscription's arena directly.

### 3.3 Memory Layout on CPU Target

On CPU targets, an observable with `N` registered observers materializes as a source reference and a flat list of observer closures:

```
Observable<T>
┌──────────────────────────────────────────────────────────────────┐
│ source_ptr: ptr         (1 platform word)                       │
├──────────────────────────────────────────────────────────────────┤
│ observer_count: i32     (4 bytes)                               │
├──────────────────────────────────────────────────────────────────┤
│ observer_ptrs: ptr[N]   (N platform words; flat closures)       │
└──────────────────────────────────────────────────────────────────┘
```

A `ptr` here is the platform word: `sizeof(ptr)` is 4 bytes on thumbv8m/M33 and 8 bytes on x86-64. So a source reference plus `N` observer pointers occupy `(N + 1)` platform words plus the 4-byte count. Each observer pointer references a flat closure (function pointer plus captured environment) laid out per [Closure Representation](closure-representation.md).

## 4. Subscription and Emission

### 4.1 Subscription

`subscribe` installs an observer continuation with the producer and returns a subscription handle:

```fsharp
val subscribe : ('T -> unit) -> Observable<'T> -> Subscription
```

The observer (`'T -> unit`) is the consumer's continuation. The returned `Subscription` is a region-scoped handle; its release (on scope exit or actor retirement) unsubscribes the observer deterministically — no `IDisposable` ceremony.

### 4.2 Emission

On emission, the producer invokes each registered observer continuation with the emitted value, in subscription order:

1. The source produces a value `v : 'T`.
2. For each active observer `k`, the producer applies `k v`.
3. Delivery is **unbounded**: every active observer receives every emission. There is no implicit cutoff and no implicit coalescing.

Emission is synchronous by default — the producer's clock determines timing. Asynchronous and buffered delivery disciplines, where required, are properties of the producer or of an explicit operator (§5), not of the bare intrinsic.

## 5. Operators

Observables compose through combinators that transform emissions. Each operator composes the **coeffects** of its inputs (the same coeffect algebra used for async and incremental analysis; see [§9](#9-coeffect-model)):

```
source @ R₁ ⊢ Observable<'A>     f @ R₂ ⊢ 'A -> 'B
─────────────────────────────────────────────────
map f source @ R₁ ⊔ R₂ ⊢ Observable<'B>
```

The push-side analogue of incremental cutoff is an explicit operator, `distinctUntilChanged`, which suppresses an emission when it equals the previous one:

```fsharp
val distinctUntilChanged : Observable<'T> -> Observable<'T>   // requires 'T : equality
 
```

This mirror is exact: `Incremental<'T>` suppresses *recomputation* on an unchanged input; `distinctUntilChanged` suppresses *emission* on an unchanged output. The difference is that suppression is intrinsic to `Incremental<'T>` (it is what the type is for) and optional for `Observable<'T>` (the default observable delivers every emission).

> **Not yet specified.** The full operator surface (e.g. `filter`, `merge`, `scan`, scheduling/backpressure combinators) and whether a `reactive { ... }` computation-expression builder is provided are not yet normative. When a builder is specified it SHALL desugar to PSG observer edges by the same discipline the `incremental` CE uses for dependency edges ([Incremental Computation §7](incremental-computation.md)).

## 6. SemanticKind in the PSG

```fsharp
type SemanticKind =
    // ...
    | ObservableExpr of
        sourceNodeId: NodeId *
        observerEdges: ObserverEdge list

and ObserverEdge = {
    ObservableNodeId: NodeId      // the producer
    ObserverNodeId: NodeId        // the registered continuation (Lambda node)
}
```

The PSG node references the producer and the list of registered observer continuations. When an `ObservableExpr` feeds an `IncrementalExpr`, the corresponding `ObserverEdge` is rewritten during fusion (§7) into the incremental node's invalidation edge rather than materializing a standalone subscription.

## 7. Target-Specific Lowering

### 7.1 CPU Target

The CPU leg is one of several target legs the backend can select (alongside the CIRCT/FPGA and JS legs); it is the leg reached when the delivery context is a CPU or MCU. On it, emission lowers to a direct dispatch loop over the observer closures (no virtual dispatch, no managed callback). The loop structure and observer-array access are portable — the middle end expresses them in `scf`/`memref` and commits to no target:

```mlir
// Emit value %v to all registered observers (portable middle-end form)
%count = memref.load %observer_count[] : memref<i32>
%n = index.casts %count : i32 to index
// for i in 0 .. count-1: invoke observers[i](%v)
scf.for %i = %c0 to %n step %c1 {
    %obs = memref.load %observer_ptrs[%i] : memref<?x!closure>
    func.call_indirect %obs(%v) : (!result_type) -> ()   // flat-closure invocation
}
```

Only the flat-closure invocation carries a construct with no portable form (a function address applied as data). On the LLVM leg specifically, the indirect call and the raw environment pointer realize as `llvm.*`:

```mlir
// LLVM-leg realization of the flat-closure dispatch inside the loop body
%obs_ptr = llvm.getelementptr %observer_ptrs[%i] : (!llvm.ptr, i64) -> !llvm.ptr
%obs = llvm.load %obs_ptr : !llvm.ptr -> !llvm.ptr
llvm.call %obs(%v) : (!result_type) -> ()
```

Other legs realize the same indirect dispatch through their own lowering (a hardware-selected observer table on the CIRCT/FPGA leg, a closure object on the JS leg); the `scf`/`memref` loop above is shared across all of them.

### 7.2 Fusion into Incremental (Accelerator Path)

An `Observable<'T>` has no standalone NPU or GPU lowering, because its opacity gives the compiler nothing to schedule statically. When an observable feeds an `Incremental<'T>`, the compiler fuses the subscription into the incremental node's invalidation trigger: the `ObserverEdge` becomes a staleness-marking edge, and the incremental node's lowering (CPU inline, AIE tile activation, or HSA dispatch, per [Incremental Computation §8](incremental-computation.md)) carries the schedule. The observable contributes the *event*; the incremental contributes the *bounded, target-specific response*.

> **Not yet specified.** Standalone lowering of an observable whose consumer is itself an accelerator-resident actor (i.e., push delivery across a hardware boundary without an intervening `Incremental<'T>`) is open. The expected path is BAREWire-mediated event delivery (§8.3), but the scheduling discipline is not yet normative.

## 8. Interaction with Other Intrinsics

### 8.1 Observable + Incremental

The canonical composition (§1.1, §7.2): an event source drives a cached derived computation; subscription fuses into invalidation. See [Incremental Computation §11.3](incremental-computation.md).

### 8.2 Observable + Cold (Frosty)

A `Cold<Observable<'T>>` represents a deferred subscription: the producer is not connected and no observer is registered until the cold value is forced. This expresses a reactive source whose side effects (opening a device, joining a stream) are withheld until demand arrives.

### 8.3 Observable + BAREWire

Emissions delivered to an observer on a different hardware target or address space use BAREWire descriptors for zero-copy transport. The observer edge's element type drives descriptor generation, exactly as dependency edges do for cross-target incremental nodes.

### 8.4 Observable + Lifetime Inference

An observer closure outlives the `subscribe` call frame (it is invoked on later emissions). Level 1 lifetime inference detects this escape and allocates the closure in the subscription's arena, with deterministic release on unsubscribe.

## 9. Coeffect Model

Subscription and operators participate in the standard coeffect algebra. The recompute-free observer of an observable carries:

| Coeffect | Description |
|---|---|
| Read | Read capability on the producer/source |
| Capture | Capture capability on the observer's environment (subscription-scoped) |
| Thread | Thread/affinity constraint of the delivery context |

Operators compose coeffects as in §5. These are resolved by the standard coeffect resolution pipeline specified in [Inference Procedures](inference-procedures.md).

## 10. Inference Levels

Following the Fidelity convention, `Observable<'T>` supports three levels of developer involvement:

### 10.1 Level 3: Explicit

```fsharp
let sub = source |> subscribe (fun reading -> handle reading)
// `sub` release (scope exit / actor retirement) unsubscribes
 
```

### 10.2 Level 2: Bounded

The developer establishes a reactive derivation; the compiler infers subscription placement and lifetime (and fuses into any consuming `Incremental<'T>`).

### 10.3 Level 1: Inferred

The compiler recognizes that an actor emits on each receive and exposes its output as an `Observable<'T>`, inferring the subscription and delivery edges from the actor's structure.

## 11. What Disappears

When `Observable<'T>` is intrinsic, the following library-level constructs are subsumed by compiler behavior:

| Library Construct | Compiler Equivalent |
|---|---|
| `IObservable<'T>` / `IObserver<'T>` interfaces | Intrinsic type; emission is flat-closure dispatch |
| `Subject` / `BehaviorSubject` plumbing | Producer node in the PSG |
| `IDisposable` / manual `Dispose` on subscriptions | Region-scoped `Subscription`; deterministic unsubscribe |
| `ObserveOn` / `SubscribeOn` scheduler ceremony | Thread coeffect on the delivery context |
| Manual bridge from events to derived state | Fusion of `Observable<'T>` into `Incremental<'T>` |

## 12. Normative Requirements

1. **Intrinsic Status**: `Observable<'T>` SHALL be a compiler-known intrinsic type, not a library type.
2. **No Equality Constraint**: `Observable<'T>` SHALL NOT require `'T : equality`. Emission comparison SHALL occur only via an explicit operator (e.g. `distinctUntilChanged`).
3. **Unbounded Delivery**: Every active observer SHALL receive every emission. The compiler SHALL NOT assume emissions may be dropped or coalesced absent an explicit operator.
4. **No Heap Allocation**: Observer closures SHALL NOT be allocated on a GC-managed heap. They SHALL reside in the subscription's arena.
5. **Deterministic Unsubscribe**: A subscription's lifetime SHALL be tied to its enclosing scope or actor, with deterministic release; no finalizer or GC SHALL be required to unsubscribe.
6. **Fusion Availability**: An `Observable<'T>` feeding an `Incremental<'T>` SHALL be fusible into the incremental node's invalidation trigger, without an intermediate heap-allocated bridge.
7. **Opacity**: The compiler SHALL treat an observable as opaque for scheduling purposes and SHALL NOT elide or reorder emissions.

## References

- Elliott, C., & Hudak, P. (1997). Functional Reactive Animation. *ICFP '97*.
- Elliott, C. (2009). Push-Pull Functional Reactive Programming. *Haskell Symposium '09*.
- Meijer, E. (2012). Your Mouse Is a Database. *Communications of the ACM, 55(5)*.
- Hagino, T. (1987). A Categorical Programming Language. PhD thesis, University of Edinburgh. *(codata / final coalgebras)*
- Tofte, M., & Talpin, J.-P. (1997). Region-Based Memory Management. *Information and Computation*.
- Syme, D. (2006). Leveraging .NET Meta-programming Components from F#. *ML Workshop '06*.
