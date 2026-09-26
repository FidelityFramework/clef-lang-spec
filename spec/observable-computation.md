---
title: "Observable Computation"
weight: 410
category: Semantics
status: normative
---

> **Normative specification for the `Observable<'T>` intrinsic type, push-based reactive observation, and its fusion into demand-driven computation in Clef compilation.**

## 1. Overview

Clef specifies `Observable<'T>` as a compiler-known intrinsic type for push-based, producer-driven reactive observation. Its subscription and delivery plan is preserved in the [Program Semantic Graph](program-semantic-graph.md) through lowering. Active subscriptions, observer closures and dynamically created sources remain runtime instances with owned state. The compiler can specialize their representation and fuse an observable into the demand-driven computation it feeds; intrinsic status does not eliminate that runtime state or require a separate reactive package.

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

Because both are intrinsic, the compiler can fuse the observable subscription directly into the incremental node's invalidation trigger, avoiding a separate bridge representation where the semantics permit (see [Incremental + Observable](incremental-computation.md#incremental--observable)). The observable supplies *change events*; the incremental supplies *bounded, cached recomputation* in response. Fusion must preserve required event delivery and effects; merely marking a cache stale does not establish that multiple emissions may be discarded.

The developer-facing `Signal`/`Memo`/`Effect` surface (see [Reactive Signals](reactive-signals.md)) is built on this pair: a `Signal` is a settable source, a `Memo` is an `Incremental`, and an `Effect` is a demanded sink.

### 1.2 Relationship to Continuations

An observer is a **continuation**. Subscribing installs the consumer's continuation with the producer; emitting invokes it. In continuation-passing terms the producer holds the consumer's continuation and applies it on each emission — the inverse of demand-driven evaluation, where the consumer holds and forces the producer's suspended continuation. The registered observer is represented as a flat closure (see [Closure Representation](closure-representation.md)); its captured environment is the consumer state the continuation resumes into.

The delimited-continuation substrate underlying `MailboxProcessor<'Msg>` and `Incremental<'T>` is the same substrate beneath `Observable<'T>`: the named intrinsic provides compositional identity that enables compiler-directed optimization, while the capture/resume mechanics exist at a lower level.

### 1.3 Relationship to Actors

The following structural correspondence can inform integration with the Olivier/Prospero actor system. It is not a one-to-one runtime mapping:

| Observable Concept | Actor Concept |
|---|---|
| Producer / source | Event source within an actor, or an actor emitting messages |
| Observer (registered continuation) | Local continuation; receive handler when delivered across actors |
| Subscription | Local registration; supervised link where actor integration requires it |
| Emission | Local continuation dispatch; message send across admitted actor boundaries |
| Subscription lifetime | Bounded by its owner; may end before that owner retires |
| Unsubscribe / teardown | Logical registration removal, followed by safe resource reclamation |

An owner may contain many sources and subscriptions without an actor or mailbox per observer. Non-actor ownership is also admitted. Owner retirement releases remaining subscriptions, but an individual subscription may end earlier. Logical unsubscribe is distinct from reclaiming closure storage or shared captured resources; native regions and JavaScript-managed storage realize that lifetime differently (§3.2).

Local delivery follows §4.2. Actor message admission adds a delivery boundary; it does not itself establish a shared synchronous clock or distributed incremental stabilization. Cross-actor ordering and failure handling require an explicit protocol.

### 1.4 Rationale for Intrinsic Status

The ingredients for reactive observation exist at the library level: a callback is a function, a subscriber list is a collection, and `MailboxProcessor<'Msg>` provides message delivery. These could be composed without compiler knowledge.

Library composition alone does not establish compiler-visible subscription, lifetime or fusion semantics. It does not inherently require a managed heap. Intrinsic status gives the compiler a stable semantic contract under which it can:

1. Place native observer storage in an inferred or explicit owning region.
2. Fuse an observable into an `Incremental<'T>` invalidation trigger when event and effect semantics are preserved.
3. Lower emission as flat-closure dispatch on native targets, or target-appropriate closure dispatch elsewhere.
4. Track subscription lifetime so logical unsubscribe does not depend on finalization or garbage collection.

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

Each `Observable<'T>` node in the PSG describes the following logical roles. These are a compile-time plan, not a complete enumeration of every active runtime registration or a fixed runtime struct. Dynamic source instances and subscription changes require corresponding runtime state, represented by the selected target.

| Field | Type | Semantics |
|-------|------|-----------|
| `source` | producer reference | The origin of emissions (event source, upstream operator, or actor) |
| `observers` | continuation list | Registered observer closures, invoked on each emission |

An observable node carries **no** `value`, `stale`, `height`, or `cutoff` field. The absence of these is the structural expression of its position on the spectrum (§1): there is nothing to cache, no dependency DAG, and no change detection.

### 3.2 Arena Allocation

An observer's logical lifetime is bounded by its subscription owner. Native closure storage follows the lifetime inference model of [Memory Regions](memory-regions.md); it must outlive every permitted invocation and cannot remain registered after release:

- **Level 1 (inferred):** An observer that escapes the `subscribe` call requires storage beyond that call frame; inference determines an admissible owning region.
- **Level 2 (bounded):** The developer establishes a subscription within an actor or other owner; the compiler infers placement within that lifetime bound.
- **Level 3 (explicit):** The developer specifies the native subscription's region directly.

On JavaScript targets, closures may reside in host-managed storage. Logical unsubscribe and owned resource cleanup remain deterministic and do not wait for host garbage collection. On native targets, removing a subscription does not imply that an individual allocation can immediately be reclaimed from a longer-lived bump arena.

Physical reclamation must account for pending invocations and shared captured resources, including when subscription changes or retirement occur during an active emission.

### 3.3 Memory Layout on CPU Target

The following CPU layout illustrates the source reference and a flat list of `N` observer closures. Dynamic registration, safe removal and active delivery require additional bookkeeping:

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

The observer (`'T -> unit`) is the consumer's continuation. The returned `Subscription` is an owner-scoped handle; its release unsubscribes the observer deterministically, with remaining handles released when the owner retires. This language-level lifetime contract does not require a .NET `IDisposable` interface. Physical reclamation follows §3.2.

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

`distinctUntilChanged` suppresses *emission* when its output is unchanged. Incremental cutoff suppresses change propagation along the unchanged node's path; a dependent with another changed input may still need recomputation. Cutoff is intrinsic to `Incremental<'T>`; duplicate suppression is optional for `Observable<'T>` (the default observable delivers every emission).

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

The PSG node references the producer and observer-registration plan. Repeated or conditional execution can realize multiple instances or different active registrations. When an `ObservableExpr` feeds an `IncrementalExpr`, fusion (§7) can realize the corresponding `ObserverEdge` as an incremental invalidation edge where delivery and effect semantics are preserved.

## 7. Target-Specific Lowering

### 7.1 CPU Target

The CPU pathway is one of several target pathways the backend can select (alongside the CIRCT/FPGA and JS pathways); it is the pathway reached when the delivery context is a CPU or MCU. On it, emission lowers to a direct dispatch loop over the observer closures (no virtual dispatch, no managed callback). The loop structure and observer-array access are portable — the middle end expresses them in `scf`/`memref` and commits to no target:

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

Only the flat-closure invocation carries a construct with no portable form (a function address applied as data). On the LLVM pathway specifically, the indirect call and the raw environment pointer realize as `llvm.*`:

```mlir
// LLVM-pathway realization of the flat-closure dispatch inside the loop body
%obs_ptr = llvm.getelementptr %observer_ptrs[%i] : (!llvm.ptr, i64) -> !llvm.ptr
%obs = llvm.load %obs_ptr : !llvm.ptr -> !llvm.ptr
llvm.call %obs(%v) : (!result_type) -> ()
```

Other pathways realize the same indirect dispatch through their own lowering (a hardware-selected observer table on the CIRCT/FPGA pathway, a closure object on the JS pathway); the `scf`/`memref` loop above is shared across all of them.

### 7.2 Fusion into Incremental (Accelerator Path)

An `Observable<'T>` has no standalone NPU or GPU lowering, because its opacity gives the compiler nothing to schedule statically. When an observable feeds an `Incremental<'T>`, fusion can realize the subscription as an invalidation trigger, subject to preservation of event delivery and effects. The incremental node's lowering (CPU inline, AIE tile activation, or HSA dispatch, per [Incremental Computation §8](incremental-computation.md)) then carries the computation schedule. The observable contributes the *event*; the incremental contributes the *bounded, target-specific response*.

## 8. Interaction with Other Intrinsics

### 8.1 Observable + Incremental

The canonical composition (§1.1, §7.2): an event source drives a cached derived computation; subscription can fuse into invalidation while preserving required event delivery. See [Incremental + Observable](incremental-computation.md#incremental--observable).

<a id="82-observable--cold-frosty"></a>
<a id="observable--cold"></a>

### 8.2 Observable + Cold

A `Cold<Observable<'T>>` represents a deferred subscription: the producer is not connected and no observer is registered until the cold value is forced. This expresses a reactive source whose side effects (opening a device, joining a stream) are withheld until demand arrives.

### 8.3 Observable + BAREWire

Emissions delivered to an observer on a different hardware target or address space use BAREWire descriptors for zero-copy transport. The observer edge's element type drives descriptor generation, exactly as dependency edges do for cross-target incremental nodes.

### 8.4 Observable + Lifetime Inference

An observer closure can outlive the `subscribe` call frame because it is invoked on later emissions. Lifetime inference must assign storage and ownership that cover those invocations. Logical unsubscribe is deterministic; target-specific physical reclamation follows §3.2.

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

The developer establishes a reactive derivation; the compiler infers subscription placement and lifetime, and can fuse into a consuming `Incremental<'T>` where event and effect semantics permit.

### 10.3 Level 1: Inferred

The compiler recognizes that an actor emits on each receive and exposes its output as an `Observable<'T>`, inferring the subscription and delivery edges from the actor's structure.

## 11. What Disappears

When `Observable<'T>` is intrinsic, the following library-level constructs are subsumed by compiler behavior:

| Library Construct | Compiler Equivalent |
|---|---|
| `IObservable<'T>` / `IObserver<'T>` interfaces | Intrinsic type; emission is flat-closure dispatch |
| `Subject` producer plumbing | Intrinsic source and subscription plan; retained current state belongs to the Signal/Incremental surface |
| `IDisposable` / manual `Dispose` on subscriptions | Region-scoped `Subscription`; deterministic unsubscribe |
| `ObserveOn` / `SubscribeOn` scheduler ceremony | Thread coeffect on the delivery context |
| Manual bridge from events to derived state | Fusion of `Observable<'T>` into `Incremental<'T>` |

## 12. Normative Requirements

1. **Intrinsic Status**: `Observable<'T>` SHALL be a compiler-known intrinsic type, not a library type.
2. **No Equality Constraint**: `Observable<'T>` SHALL NOT require `'T : equality`. Emission comparison SHALL occur only via an explicit operator (e.g. `distinctUntilChanged`).
3. **Unbounded Delivery**: Every active observer SHALL receive every emission. The compiler SHALL NOT assume emissions may be dropped or coalesced absent an explicit operator.
4. **Target-Appropriate Storage**: Native observer closures SHALL reside in inferred or explicit owning regions, without requiring a GC-managed heap. JavaScript lowering MAY use host-managed closure storage; logical lifetime requirements SHALL remain the same.
5. **Deterministic Unsubscribe**: A subscription's lifetime SHALL be bounded by its owner, with deterministic release; no finalizer or GC SHALL be required to unsubscribe. Physical reclamation SHALL account for permitted pending invocations and resource ownership.
6. **Fusion Availability**: An `Observable<'T>` feeding an `Incremental<'T>` SHALL admit fusion into the incremental node's invalidation trigger without a mandatory separately allocated bridge. Fusion SHALL preserve required event delivery and effects; it SHALL NOT authorize implicit event dropping or coalescing.
7. **Opacity**: The compiler SHALL treat an observable as opaque for scheduling purposes and SHALL NOT elide or reorder emissions.

## References

- Elliott, C., & Hudak, P. (1997). Functional Reactive Animation. *ICFP '97*.
- Elliott, C. (2009). Push-Pull Functional Reactive Programming. *Haskell Symposium '09*.
- Meijer, E. (2012). Your Mouse Is a Database. *Communications of the ACM, 55(5)*.
- Hagino, T. (1987). A Categorical Programming Language. PhD thesis, University of Edinburgh. *(codata / final coalgebras)*
- Tofte, M., & Talpin, J.-P. (1997). Region-Based Memory Management. *Information and Computation*.
- Syme, D. (2006). Leveraging .NET Meta-programming Components from F#. *ML Workshop '06*.
