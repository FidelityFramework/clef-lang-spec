---
title: "Incremental Computation"
weight: 400
category: Semantics
status: normative
---

> **Normative specification for the `Incremental<'T>` intrinsic type, dependency-tracked change propagation, and target-specific stabilization lowering in Clef compilation.**

> **Acknowledgment**: The design of `Incremental<'T>` takes direct inspiration from the adaptive-computation model of **FSharp.Data.Adaptive** (the `aval`/`cval`/`aset` families and their change-propagation and cutoff semantics), as well as from Jane Street's `Incremental`. The framework's cold-first posture also draws on Jimmy Byrd's [IcedTasks](https://github.com/TheAngryByrd/IcedTasks): deferred, explicitly started work is a foundational influence, distinct from the caching and invalidation supplied by `Incremental`. Clef adopts demand-driven recomputation, cutoff and dependency tracking as compiler-known semantics that the [Program Semantic Graph](program-semantic-graph.md) preserves through lowering. Computation expressions and the [Reactive Signals](reactive-signals.md) surface express that model. Compiler visibility permits static specialization; it does not eliminate the state or dynamic instances required by the program.

## 1. Overview

Clef specifies `Incremental<'T>` as a compiler-known intrinsic type for dependency-tracked, demand-driven, change-minimizing computation. Its reactive plan is preserved in the Program Semantic Graph through lowering, enabling Composer to generate target-specific code for selective recomputation on CPU, GPU, and NPU hardware. Cached values, invalidation state and dynamically selected graph instances remain runtime facts, realized by the selected target rather than erased by intrinsic status.

`Incremental<'T>` occupies a specific position in a spectrum of evaluation strategies that the compiler understands natively:

| Property | Observable&lt;'T&gt; | Cold&lt;'T&gt; | Lazy&lt;'T&gt; | Incremental&lt;'T&gt; |
|---|---|---|---|---|
| Deferred evaluation | No (push) | Yes | Yes | Yes (demand-driven) |
| Cached result | No | No | Yes | Yes |
| Dependency tracking | No | No | No | Yes |
| Invalidation | No | No | No | Yes |
| Cutoff (change detection) | No | No | No | Yes |
| Propagation bound | Unbounded | N/A | N/A | Bounded by cutoff |

Each position to the right provides the compiler with more information during lowering. An `Observable<'T>` is opaque: the compiler must assume every emission matters. An `Incremental<'T>` exposes dependency and cutoff semantics that let the compiler reason about selective recomputation. Suppressing recomputation requires establishing that every relevant observation is unchanged and that skipping the body omits no required effect. Read/effect analysis supplies that obligation, including behavior reached through calls and captured references. A flat environment enumerates captures, not necessarily active reads; environment layout alone does not prove dependency completeness or purity ([Closure Representation §2.2](closure-representation.md#22-capture-semantics)).

### 1.1 Relationship to Lazy Values

`Incremental<'T>` extends the semantics of `Lazy<'T>` (specified in [Lazy Value Representation](lazy-representation.md)) with two additional properties: dependency tracking and invalidation. A `Lazy<'T>` computes once and caches indefinitely. An `Incremental<'T>` computes on demand, caches the result, tracks which inputs contributed to that result, and invalidates the cache when those inputs change. Recomputation occurs only when the node is both stale (inputs changed) and demanded (a downstream consumer requires the value).

### 1.2 Relationship to Actors

The following structural correspondence can inform integration with the Olivier/Prospero actor system. It is not a one-to-one runtime mapping:

| Incremental Concept | Actor Concept |
|---|---|
| Cached value | Actor state in arena memory |
| Recompute function | Computation within an actor turn |
| Dependency edges | Local data dependencies; BAREWire channels across admitted actor boundaries |
| Staleness flag | Invalidated actor-owned cached state |
| Cutoff predicate | Structural comparison on output |
| Demand registration | Local observation demand, integrated with Prospero for actor-owned nodes |
| Stabilization order | Dependency-ordered local work within actor dispatch |

An actor may own many incremental nodes that stabilize locally without a mailbox or actor per node. A node's lifetime is bounded by its owning actor or region; actor retirement disposes the remaining owned graph. Native cached storage follows that region's reclamation rules. Non-actor contexts are also admitted (§6.3).

Actor message ordering and admission do not by themselves establish consistent dependency-ordered recomputation. Local stabilization must preserve §6. A cross-actor or cross-process edge additionally needs a delivery and consistency contract; mailbox coalescing is not equivalent to a local batch. This chapter does not specify a distributed stabilization protocol.

### 1.3 Rationale for Intrinsic Status

Deferred computation, cached values, structural equality and tracked observations supply the functional ingredients for incremental computation. Clef's ordinary combinators and computation expressions expose their composition. Dependency discovery must account for actual reads and effects, including calls and captured references; `let!` syntax alone does not establish the complete dependency set.

Library composition alone does not establish compiler-visible dependency, cutoff or stabilization semantics. Intrinsic status gives the compiler a stable semantic contract under which it can:

1. Determine dependency graph structure statically for applicative subgraphs
2. Infer cutoff functions from type equality semantics
3. Lower to target-appropriate execution (CPU inline, GPU wavefront, NPU tile dispatch)
4. Fuse across intrinsic boundaries (e.g., `Observable<'T>` subscription into `Incremental<'T>` invalidation trigger)

This is the same justification that applies to `MailboxProcessor<'Msg>`: the underlying mechanics (delimited continuations, mutable state, concurrent queues) exist at a lower level, but the named type provides compositional identity that enables compiler-directed optimization.

## 2. Type Definition

### 2.1 Core Type

```fsharp
type Incremental<'T when 'T : equality> = intrinsic
```

The `'T : equality` constraint provides the default cutoff predicate. Structural equality on records and discriminated unions generates the cutoff automatically. A domain-specific cutoff is an ordinary typed function associated through `[<IncrementalCutoff>]` as specified in §5.2. The equality constraint and the cutoff's dimensional compatibility are checked at compile time.

### 2.2 Hardware-Targeted Type

For explicit hardware dispatch, a measure-annotated variant carries the target:

```fsharp
type Incremental<'T, [<Measure>] 'Target when 'T : equality> = intrinsic
```

Usage:

```fsharp
[<Measure>] type cpu
[<Measure>] type gpu_cu
[<Measure>] type npu_tile

let routingDecision : Incremental<RoutingVector> = ...             // CPU (default)
let visionFeatures  : Incremental<FeatureMap, npu_tile> = ...      // NPU
let languageEmbed   : Incremental<Embedding, gpu_cu> = ...         // GPU
 
```

When the target measure is absent, the compiler infers it from context or defaults to CPU. When present, it constrains lowering. This follows the general Fidelity pattern of inference by default, explicit annotation when needed.

### 2.3 NativeType Representation

```fsharp
type NativeType =
    // ...
    | TIncremental of
        elementType: NativeType *
        targetMeasure: MeasureType option
```

## 3. Node Structure

### 3.1 Logical Fields

Each `Incremental<'T>` construct in the PSG describes the following logical fields. The compiler plan identifies their meaning and lowering; runtime instances hold the values and active state needed by that plan. One construct may produce multiple instances. These fields do not prescribe one uniform runtime struct across targets.

| Field | Type | Semantics |
|-------|------|-----------|
| `value` | `'T` | Cached result of the most recent computation |
| `stale` | `bool` | Whether the cached value needs validation after possible input change |
| `height` | `int` | Topological depth in the dependency DAG; determines evaluation order |
| `dependencies` | `NodeId list` | PSG nodes this node reads from |
| `dependents` | `NodeId list` | PSG nodes that read from this node |
| `cutoff` | `'T -> 'T -> bool` | Equality predicate; defaults to structural equality |
| `recompute` | `unit -> 'T` | The deferred computation (thunk with captured environment) |

### 3.2 Arena Allocation

In a native actor context, the cached value is placed in storage owned by the enclosing actor or an admitted shorter-lived region. Its lifetime must cover use across stabilization cycles; it need not equal the whole actor lifetime. This connects directly to the lifetime inference model specified in [Memory Regions](memory-regions.md):

- **Level 1 (inferred):** The compiler determines that the cached value persists across stabilization cycles and chooses storage that covers its inferred lifetime, such as the actor's arena when that lifetime is appropriate.
- **Level 2 (bounded):** The developer marks a computation as incremental via the CE; the compiler infers arena placement.
- **Level 3 (explicit):** The developer specifies arena placement directly.

Reclamation of replaced dynamic subgraphs must satisfy the lifetime constraints of [Memory Regions](memory-regions.md#lifetime-constraints). A bump arena reclaimed only at actor retirement does not establish bounded storage for arbitrarily repeated replacement. The JavaScript pathway uses host-managed storage under [Memory Regions' target reachability rules](memory-regions.md#target-reachability), while preserving logical disposal.

### 3.3 Memory Layout on CPU Target

On CPU targets, an incremental node with element type `T` and `N` tracked dependencies can materialize with the fields below, with additional bookkeeping for dynamic dependencies and invalidation. Pointer fields are sized to the platform word: 4 bytes on thumbv8m/M33, 8 bytes on x86-64. The layout is target-parameterized, so the byte totals shown are the x86-64 case with the M33 word given alongside.

```
IncrementalNode<T> with dependencies [d₁: T₁, ..., dₙ: Tₙ]
┌──────────────────────────────────────────────────────────────────┐
│ stale: i1               (1 byte, padded to alignment)            │
├──────────────────────────────────────────────────────────────────┤
│ height: i32             (4 bytes)                                │
├──────────────────────────────────────────────────────────────────┤
│ value: T                (sizeof(T) bytes, aligned)               │
├──────────────────────────────────────────────────────────────────┤
│ recompute_ptr: ptr      (1 platform word: 4 bytes M33, 8 x86-64) │
├──────────────────────────────────────────────────────────────────┤
│ dep_count: i32          (4 bytes)                                │
├──────────────────────────────────────────────────────────────────┤
│ dep_ptrs: ptr[N]        (N platform words, pointers to dep nodes)│
└──────────────────────────────────────────────────────────────────┘
```

On GPU and NPU targets, the node does not materialize as a struct. The logical fields are distributed across hardware resources as specified in [§8](#8-target-specific-lowering).

## 4. Dependency Graph

### 4.1 Static vs. Dynamic Structure

The dependency graph for `Incremental<'T>` nodes forms a directed acyclic graph (DAG). The compiler distinguishes two structural categories based on applicative vs. monadic composition:

**Applicative subgraphs** have static structure known at compile time. All dependencies are declared unconditionally. Composer can compile these to fixed hardware configurations (static AIE overlays, pre-allocated GPU dispatch groups).

```fsharp
// Applicative: both dependencies are unconditional
incremental {
    let! sensorA = temperatureSensor
    let! sensorB = pressureSensor
    return fuseSensorData sensorA sensorB
}
```

**Monadic subgraphs** have dynamic structure that depends on runtime values. The set of active dependencies can change per stabilization cycle. These require dynamic orchestration (ERT ctrlcode on NPU, runtime dispatch on GPU).

```fsharp
// Monadic: which expert runs depends on routing decision
incremental {
    let! routing = bitnetRouter input
    let! result =
        match routing.selectedExpert with
        | Vision -> visionExpert input
        | Language -> languageExpert input
    return result
}
```

The compiler infers the category from the CE desugaring. `let!` followed by usage that does not influence subsequent `let!` bindings is applicative. `let!` whose result determines which subsequent `let!` executes is monadic.

The same classification applies to tracked reads expressed through the Signals surface; CE syntax is not a requirement for static analysis or parallel realization. The PSG carries a dependency plan, including dynamic selection operations. Runtime instances retain active edges and identity where those depend on values unavailable at compile time. Analysis SHALL account for relevant helper calls, aliases and aggregate access; lexical capture enumeration alone is insufficient. Unsupported dependency behavior SHALL be diagnosed rather than silently approximated as a fixed read set.

### 4.2 Height Assignment

Each node is assigned a height equal to the longest path from any leaf input to that node:

```
height(node) = 0                                        if node has no dependencies
height(node) = 1 + max(height(d) for d in dependencies) otherwise
```

Height determines evaluation order during stabilization. Nodes at height 0 are evaluated first. Nodes at height `h` are evaluated only after all nodes at height `h-1` have been processed. This corresponds directly to wave scheduling on hardware targets: tiles at height 0 fire in the first wave, tiles at height 1 in the second, and so on.

Height is computed at compile time for applicative subgraphs and tracked dynamically for monadic subgraphs.

## 5. Staleness and Cutoff

### 5.1 Staleness Propagation

When an input node's value is set or an external event invalidates it, staleness propagates forward through the dependency edges:

1. The modified node is marked stale.
2. All direct dependents are marked stale.
3. Propagation continues transitively through dependents.

Staleness propagation is conservative: a node marked stale may not actually need recomputation (the cutoff may determine its output is unchanged). The purpose is to identify the *candidate* recomputation set.

### 5.2 Cutoff Semantics

The cutoff predicate determines whether a recomputed value differs from the cached value. If the cutoff returns `true` (values are equal under that predicate), no output change propagates from that node. A dependent conservatively marked stale under §5.1 may still need validation because another dependency changed. Cutoff SHALL NOT erase an independent invalidation.

**Default cutoff:** Structural equality derived from the `'T : equality` constraint. For records and discriminated unions, the compiler generates field-by-field comparison. For primitive types, hardware-native equality instructions are used.

**Custom cutoff:** The developer provides a domain-specific equality predicate:

```fsharp
/// Cutoff: same routing decision (set equality on selected expert indices)
[<IncrementalCutoff>]
let routingEqual (a: RoutingVector) (b: RoutingVector) =
    Set.equals a.selectedExperts b.selectedExperts

/// Cutoff: value changed by less than epsilon
[<IncrementalCutoff>]
let epsilonEqual (a: float<celsius>) (b: float<celsius>) =
    abs (a - b) < 0.01<celsius>
```

The `[<IncrementalCutoff>]` attribute associates a cutoff function with a specific type for use in `Incremental<'T>` nodes.

### 5.3 Dimensional Type Constraints on Cutoff

The DTS (Dimensional Type System) constrains cutoff functions. Two values with incompatible units cannot be compared for equality without explicit conversion. This prevents a class of cutoff bugs: accidentally comparing values in different unit systems and obtaining spurious "unchanged" results.

```fsharp
// Compile-time error: cannot compare celsius and fahrenheit
let badCutoff (a: float<celsius>) (b: float<fahrenheit>) = abs(a - b) < 0.01  // TYPE ERROR
 
```

## 6. Stabilization

### 6.1 Algorithm

Stabilization brings the demanded incremental graph up to date in dependency order. Conservative staleness identifies candidates; it does not prove that every candidate's inputs changed. A conforming algorithm SHALL preserve the following discipline:

1. Collect the demanded stale fragment, including dependencies needed to validate it.
2. Process it in dependency order, using heights and maintaining that order when active dynamic dependencies change.
3. For a node with an existing cache, reuse that cache only when every relevant input has been validated unchanged and no other invalidation requires recomputation. Otherwise, recompute from current dependencies and compare the result with the cache using its cutoff predicate. A node without a cache requires initial computation.
4. If cutoff reports unchanged, retain the cached value and clear this node's stale state. Suppress output-change propagation from this node only; do not clear another node's unresolved invalidations.
5. If the output changed, update the cache, clear this node's stale state and preserve the affected demanded dependents for validation at their dependency order.

For example, let `A = X % 2`, `B = Y`, and `C = A + B`. A batch that changes `X` from 0 to 2 and `Y` from 0 to 1 leaves `A` unchanged but changes `B` and `C`. Cutoff at `A` must not remove `C` from the work required by `B`. A downstream subgraph may be skipped only when all relevant paths justify reuse, not merely because one incoming path reached cutoff.

Implementations may use input versions, invalidation causes or an equivalent discipline; this chapter does not prescribe that bookkeeping representation. Cutoff reduces work where outputs remain unchanged; it does not by itself provide a fixed bound on the size or execution time of an arbitrary graph.

### 6.2 Stabilization Scope

The compiler inserts stabilization at context-specific boundaries:

| Context | Stabilization Boundary |
|---|---|
| Actor message processing | After each message is received |
| Rendering pipeline | At frame boundaries |
| Batch data processing | At chunk boundaries |
| Stream processing | At element or micro-batch boundaries |

The developer does not call `stabilize()` manually. The compiler determines insertion points from the enclosing computation context.

Logical stabilization and frame presentation are distinct. An adapter to another reactive system must establish its read, write, batching, effect, suspension and failure behavior; the stabilization boundary alone does not establish behavioral equivalence.

### 6.3 Demand Registration

An `Incremental<'T>` node that no downstream consumer observes does not participate in stabilization, even if its inputs are stale. Prospero manages demand registration for actor-based nodes. For non-actor contexts, the compiler derives demand from the PSG plan, with runtime state where observation depends on dynamic instances or lifetimes.

## 7. Computation Expression

### 7.1 Builder Type

```fsharp
type IncrementalBuilder() =
    member _.Bind(node: Incremental<'T>, f: 'T -> Incremental<'U>) : Incremental<'U>
    member _.Return(value: 'T) : Incremental<'T>
    member _.ReturnFrom(node: Incremental<'T>) : Incremental<'T>
    member _.Combine(a: Incremental<'T>, b: Incremental<'T>) : Incremental<'T>
    member _.Zero() : Incremental<unit>

let incremental = IncrementalBuilder()
```

### 7.2 Desugaring to PSG Edges

Each `let!` binding in the CE desugars to a dependency edge in the PSG:

```fsharp
incremental {
    let! routing = bitnetRouter input       // edge: input → routing
    let! vision  = visionExpert routing     // edge: routing → vision
    let! lang    = languageExpert routing   // edge: routing → lang
    return fusionLayer vision lang          // edges: vision → fusion, lang → fusion
}
```

The compiler extracts the dependency DAG directly from the `Bind` chain. For applicative subgraphs (where `let!` bindings are independent), the compiler detects parallelism: `vision` and `lang` have the same height and can execute concurrently.

### 7.3 Applicative Optimization

When the compiler determines that a `Bind` does not introduce data dependency between sequential `let!` bindings, it lowers the subgraph as applicative:

```fsharp
// These two bindings are independent; compiler detects applicative structure
incremental {
    let! a = sensorA    // height 1
    let! b = sensorB    // height 1 (not height 2; no dependency on a)
    return combine a b  // height 2
}
```

This distinction determines whether the lowered code uses static dispatch (applicative) or dynamic dispatch (monadic) on hardware targets.

## 8. Target-Specific Lowering

### 8.1 CPU Target

On CPU, incremental nodes are lowered to inline stabilization with arena-allocated cached values. The `llvm.*` operations below are the CPU/MCU target pathway, not what the portable middle end emits. The middle end forms the stabilization control flow in portable dialects only (the staleness branch as `scf`/`cf`, the node struct and its fields as `memref` load and store), commits to no target, and hands that form to a target pathway. The LLVM pathway shown here is one such target commitment; the NPU pathway (§8.2) and GPU pathway (§8.3) realize the same portable form differently. Read the block below as the LLVM pathway's output after that commitment, not as middle-end output.

The following example shows the recompute/cutoff step for a node already selected for processing. Dependency validation follows §6.1.

```mlir
// Stabilization check for a single node
%is_stale = llvm.load %node_stale_ptr : !llvm.ptr -> i1
llvm.cond_br %is_stale, ^recompute, ^use_cached

^recompute:
    // Call recompute function (same convention as lazy thunks)
    %new_value = llvm.call %recompute_ptr(%node_ptr) : (!llvm.ptr) -> !result_type

    // Cutoff: compare with cached value
    %cached = llvm.load %node_value_ptr : !llvm.ptr -> !result_type
    %unchanged = llvm.call @structural_eq(%new_value, %cached) : (...) -> i1
    llvm.cond_br %unchanged, ^cutoff_hit, ^propagate

^cutoff_hit:
    // Clear stale flag, do NOT propagate to dependents
    %false = arith.constant 0 : i1
    llvm.store %false, %node_stale_ptr : i1, !llvm.ptr
    llvm.br ^use_cached

^propagate:
    // Update cached value, clear stale flag
    llvm.store %new_value, %node_value_ptr : !result_type, !llvm.ptr
    %false2 = arith.constant 0 : i1
    llvm.store %false2, %node_stale_ptr : i1, !llvm.ptr
    // Dependents remain stale; processed at their height
    llvm.br ^done(%new_value : !result_type)

^use_cached:
    %val = llvm.load %node_value_ptr : !llvm.ptr -> !result_type
    llvm.br ^done(%val : !result_type)

^done(%result: !result_type):
    // %result is the stabilized value
 
```

### 8.2 NPU Target (MLIR-AIE)

On XDNA 2 AIE tiles, incremental nodes map to tile configurations with selective DMA activation:

| Incremental Concept | AIE Lowering |
|---|---|
| Node | AIE tile (VLIW+SIMD processor with local SRAM) |
| Cached value | Tile-local SRAM contents |
| Recompute function | Tile ELF binary |
| Dependency edge | Stream switch route with DMA descriptor |
| Staleness flag | DMA activation gating (inactive = not stale) |
| Cutoff | Scalar processor comparison on output buffer |
| Height-based ordering | Wave-based DMA activation sequences |
| Demand | Column power gating (unused columns stay idle) |

For applicative subgraphs, the compiler generates a static overlay: tile placement, stream switch routing, and DMA descriptors are determined at compile time and configured once. Incremental recomputation means only activating DMA descriptors for tiles whose inputs changed, not reconfiguring the overlay.

For monadic subgraphs, the compiler generates ctrlcode that the ERT (Embedded Runtime) executes to dynamically select which tiles to activate based on runtime routing decisions.

The height-based stabilization order maps to MLIR-AIE's ObjectFIFO activation sequence:

```
Height 0 tiles: Input DMA activated (leaf nodes)
Height 1 tiles: Activated when height 0 outputs land in ObjectFIFO
Height 2 tiles: Activated when height 1 outputs land in ObjectFIFO
...
```

Cutoff at a tile suppresses its output update. A downstream tile still needs evaluation when another input changed; the realization must preserve the cached input or equivalent availability information for the unchanged path. Absence of a new ObjectFIFO write alone is not sufficient to decide that a join remains idle.

### 8.3 GPU Target (RDNA)

On RDNA 3.5 compute units, incremental nodes map to wavefront dispatch groups:

| Incremental Concept | GPU Lowering |
|---|---|
| Node | CU wavefront group |
| Cached value | LDS (Local Data Share) or VGPR contents |
| Dependency edge | Global memory handoff via BAREWire descriptor |
| Staleness flag | Dispatch predicate (skip CU if not stale) |
| Cutoff | Ballot/vote instruction across wavefront lanes |
| Height-based ordering | Sequential dispatch waves |
| Independent nodes at same height | Parallel CU dispatch within a wave |

The compiler generates HSA kernel dispatch packets with predicated execution: CUs whose input buffers are unchanged (determined by a BAREWire descriptor comparison) are not dispatched.

## 9. SemanticKind in the PSG

### 9.1 IncrementalExpr

```fsharp
type SemanticKind =
    // ...
    | IncrementalExpr of
        bodyNodeId: NodeId *
        dependencies: DependencyEdge list *
        cutoffNodeId: NodeId option *
        targetMeasure: MeasureType option *
        graphCategory: IncrementalGraphCategory

and DependencyEdge = {
    SourceNodeId: NodeId
    SinkNodeId: NodeId
    Height: int
}

and IncrementalGraphCategory =
    | Applicative    // Static graph structure, known at compile time
    | Monadic        // Dynamic graph structure, determined at runtime
    | Mixed          // Applicative skeleton with monadic subgraphs
 
```

### 9.2 IncrementalNode in PSG

The PSG node for an incremental computation contains:

- `bodyNodeId`: Reference to the recompute Lambda node (same convention as lazy thunks).
- `dependencies`: Pre-computed dependency edge list with height annotations.
- `cutoffNodeId`: Optional reference to a custom cutoff function. When `None`, structural equality is used.
- `targetMeasure`: Optional hardware target annotation. When `None`, inferred or defaulted to CPU.
- `graphCategory`: Compile-time determination of applicative vs. monadic structure.

## 10. Coeffect Model

### 10.1 IncrementalLayout Coeffect

SSA assignment computes `IncrementalLayout` for each incremental expression:

```fsharp
type IncrementalLayout = {
    NodeId: NodeId
    ElementType: MLIRType
    Height: int
    DependencyCount: int
    Dependencies: DependencySlot list
    CutoffFunctionSSA: SSA option
    TargetMeasure: MeasureType option
    GraphCategory: IncrementalGraphCategory

    // SSA identifiers for CPU target construction
    StaleConstSSA: SSA           // initial stale = true
    UndefNodeSSA: SSA            // undef node struct
    WithStaleSSA: SSA            // insertvalue stale at [0]
    WithHeightSSA: SSA           // insertvalue height at [1]
    RecomputeAddrSSA: SSA        // addressof recompute_ptr
    WithRecomputeSSA: SSA        // insertvalue recompute_ptr at [3]
    DepInsertSSAs: SSA list      // insertvalue for each dependency pointer
    NodeResultSSA: SSA           // final constructed node
}

and DependencySlot = {
    Name: string
    SourceNodeId: NodeId
    Height: int
    Type: MLIRType
}
```

### 10.2 Recompute Function Coeffects

The recompute function associated with an incremental node has the following coeffect requirements:

| Coeffect | Description |
|---|---|
| Read | Read capability on all dependency nodes |
| Write | Write capability on the cached value slot |
| Allocate | Allocation capability for temporaries (within arena scope) |
| Target | Hardware target constraint from measure annotation |

These are resolved by the standard coeffect resolution pipeline specified in [Inference Procedures](inference-procedures.md).

## 11. Interaction with Other Intrinsics

### 11.1 Incremental + Lifetime Inference

An `Incremental<'T>` node's cached value persists across stabilization cycles. Lifetime analysis therefore requires storage that outlives a single recompute call. In a native actor context this can be the owning actor's arena or an admitted shorter-lived region (§3.2); escaping the call frame alone does not establish that the cache must live until actor termination.

### 11.2 Incremental + UMX / DTS

Dimensional types on `Incremental<float<meters/seconds>>` constrain the cutoff function. The DTS verifies that comparison operands in the cutoff share compatible units. It also constrains cross-node connections: a dependency edge from `Incremental<float<celsius>>` to a node expecting `float<fahrenheit>` is a compile-time error.

<a id="113-incremental--observable-rx"></a>
<a id="incremental--observable"></a>

### 11.3 Incremental + Observable

An `Observable<'T>` feeding into an `Incremental<'T>` is a common pattern: an event source drives a cached derived computation. Because both are intrinsic, the compiler can fuse the observable subscription into the incremental node's invalidation trigger without a mandatory separate bridge allocation. Fusion must preserve required event delivery and effects; marking a cache stale does not itself authorize dropping or coalescing emissions. The `Memo<'T>` of the [Reactive Signals](reactive-signals.md) surface desugars to an `Incremental<'T>`; its dependencies may include settable sources and other incremental nodes, without requiring an observable bridge for every memo.

<a id="114-incremental--cold-frosty"></a>
<a id="incremental--cold"></a>

### 11.4 Incremental + Cold

A `Cold<Incremental<'T>>` represents a deferred incremental subgraph. The subgraph is not constructed (and no dependencies are registered) until the cold value is forced. On NPU targets, this means DMA descriptors are configured but not activated until demand arrives.

### 11.5 Incremental + Interaction Nets

When incremental nodes are modeled as interaction net agents, the net's reduction strategy implements stabilization. An interaction rule fires when two principal ports are connected; an incremental node recomputes when its dependencies are stale and its output is demanded. The interaction net formalism provides optimal sharing: two subgraphs computing the same value from the same dependencies are automatically shared, eliminating redundant incremental nodes.

### 11.6 Incremental + BAREWire

Dependency edges between incremental nodes on different hardware targets use BAREWire descriptors for zero-copy data movement. On HSA-unified memory (e.g., Strix Halo), CPU-to-GPU edges are pointer handoffs. CPU/GPU-to-NPU edges use DMA descriptors that BAREWire generates from the dependency edge's type information.

## 12. Inference Levels

Following the Fidelity convention for compiler inference, `Incremental<'T>` supports three levels of developer involvement:

### 12.1 Level 3: Explicit

The developer declares all incremental structure manually:

```fsharp
let router = IncrementalActor.create (arena: byref<Arena<'lifetime>>) {
    dependencies = [inputStream]
    cutoff = RoutingDecision.structuralEquality
    compute = fun input -> bitnetRoute input
}
```

### 12.2 Level 2: Bounded

The developer marks a computation as incremental; the compiler infers dependencies from `let!` bindings:

```fsharp
let router = incremental {
    let! input = inputStream
    return bitnetRoute input
}
```

### 12.3 Level 1: Inferred

The compiler recognizes that an actor's receive function is pure (or has only tracked effects) and automatically inserts dependency tracking and cutoff checks:

```fsharp
type Router() =
    inherit Actor<SensorInput>()
    override this.Receive input =
        bitnetRoute input  // Compiler infers: cacheable, structural cutoff
 
```

## 13. What Disappears

When `Incremental<'T>` is intrinsic, the following library-level operations are subsumed by compiler behavior:

| Library Operation | Compiler Equivalent |
|---|---|
| `Incremental.stabilize()` | Compiler inserts stabilization at context-appropriate boundaries |
| `Incremental.set_cutoff` | Inferred from type equality semantics or `[<IncrementalCutoff>]` attribute |
| `Incremental.bind` / `Incremental.map` | `let!` and `return` in incremental CE; desugars to PSG edges |
| `Incremental.Observer` | Prospero demand registration for actor nodes |
| Manual graph construction | Compiler derives the PSG reactive plan and realizes static or dynamic instances |
| Manual height computation | Compiler assigns heights statically for applicative subgraphs |

## 14. Implementation in the CCS/Composer Pipeline

### 14.1 CCS (Clef Compiler Service) Phase

1. **checkIncremental** in Coordinator.fs:
   - Checks the incremental body expression
   - Computes dependencies from `let!` bindings
   - Classifies graph category (Applicative, Monadic, Mixed)
   - Assigns heights to all nodes in applicative subgraphs
   - Creates `SemanticKind.IncrementalExpr` with dependency edges

2. **inferCutoff** in TypeChecker.fs:
   - Resolves the cutoff function for each node
   - Uses structural equality by default
   - Checks for `[<IncrementalCutoff>]` attribute on the element type
   - Verifies dimensional type compatibility of cutoff operands

### 14.2 Alex Preprocessing Phase

1. **SSAAssignment**:
   - Computes `IncrementalLayout` for IncrementalExpr nodes
   - Computes `ClosureLayout` for recompute Lambda nodes
   - Assigns SSAs for node construction (CPU target) or descriptor generation (NPU/GPU target)

2. **HeightAnalysis**:
   - Validates height assignments for applicative subgraphs
   - Generates dynamic height tracking code for monadic subgraphs

### 14.3 Witness Phase

1. **IncrementalWitness**:
   - Observes `IncrementalLayout` coeffect
   - Emits target-specific code:
     - CPU: arena-allocated node struct, inline stabilization loop
     - NPU: MLIR-AIE tile configuration, DMA descriptor generation, ObjectFIFO setup
     - GPU: HSA dispatch packet generation, predicated CU dispatch

2. **CutoffWitness**:
   - Generates equality comparison function for the element type
   - Applies structural equality derivation for records and DUs
   - Inlines the comparison into the stabilization check

## 15. Normative Requirements

1. **Intrinsic Status**: `Incremental<'T>` SHALL be a compiler-known intrinsic type, not a library type.
2. **Equality Constraint**: The element type `'T` SHALL satisfy the `equality` constraint. Types without equality SHALL be rejected at compile time.
3. **Native Storage**: On native targets, incremental nodes SHALL NOT be allocated on a GC-managed heap; cached values SHALL use storage covering their inferred region lifetime. The JavaScript pathway uses host-managed storage under the target rules of [Memory Regions](memory-regions.md#target-reachability).
4. **Cutoff Default**: When no custom cutoff is specified, structural equality on `'T` SHALL be the default cutoff predicate.
5. **Staleness Transitivity**: Staleness propagation SHALL be transitive through all dependency edges.
6. **Height Ordering**: Stabilization SHALL process nodes in ascending height order.
7. **Cutoff Termination**: When cutoff determines a value is unchanged, propagation attributable to that output change SHALL stop. Independent invalidations of shared dependents SHALL be preserved (§6.1).
8. **Demand Gating**: Nodes with no downstream observers SHALL NOT participate in stabilization.
9. **Deterministic Cleanup**: Node lifetime SHALL be bounded by its owning actor or region, with deterministic logical disposal. Native storage reclamation SHALL follow the owning region's lifetime; JavaScript storage reclamation remains host-managed.
10. **Applicative Detection**: The compiler SHALL detect applicative subgraph structure from CE desugaring and generate static dispatch configurations where applicable.
11. **Target Fidelity**: Hardware target annotations via measure types SHALL survive through the PSG to code generation without erasure.
12. **Dependency Completeness**: Reuse of a cached computation SHALL be justified by its relevant reads and effects, not by lexical capture identity alone. Static specialization and supported dynamic realization SHALL preserve that dependency plan; unsupported behavior SHALL be diagnosed.

## References

- Acar, U. A. (2005). *Self-Adjusting Computation*. PhD thesis, Carnegie Mellon University.
- Acar, U. A., Blelloch, G. E., & Harper, R. (2002). Adaptive Functional Programming. *POPL '02*.
- Hammer, M. A., Phang, K. Y., Hicks, M., & Foster, J. S. (2014). Adapton: Composable, Demand-Driven Incremental Computation. *PLDI '14*.
- Jane Street. *Incremental*. https://github.com/janestreet/incremental
- Haaser, G., et al. *FSharp.Data.Adaptive* — adaptive functional dependency graphs (`aval`/`cval`/`aset`). https://github.com/fsprojects/FSharp.Data.Adaptive
- Hunhoff, E., et al. (2025). Efficiency, Expressivity, and Extensibility in a Close-to-Metal NPU Programming Interface. *IEEE FCCM 2025*.
- Tofte, M., & Talpin, J.-P. (1997). Region-Based Memory Management. *Information and Computation*.
- Lafont, Y. (1990). Interaction Nets. *POPL '90*.
- Syme, D. (2006). Leveraging .NET Meta-programming Components from F#. *ML Workshop '06*.
