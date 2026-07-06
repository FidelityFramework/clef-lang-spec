---
title: "Delimited Continuation Representation"
weight: 350
category: Representation
status: normative
---

> **Normative specification for the delimited-continuation operation surface, its target
> lowerings, and the cooperative scheduling that suspend/resume realizes on a
> single-core target in Clef compilation.**

## 1. Overview

Delimited continuations are the substrate under `async { }`, actor `receive`, and every synchronous suspension point. This chapter specifies them the way a dialect specifies an abstraction: as a **transportable operation surface** — `cont.new`, `cont.suspend`, `cont.resume`, and their memory ops — that is target-neutral, together with the lowerings that carry that surface to concrete backends. The point of the surface is precisely that it does not commit to a lowering: the same continuation ops travel unchanged while the backend is chosen per target.

The surface has **two destinations, and both reach the native-CPU backend leg through LLVM** — they are not rival architectures but two shapes of the one native lowering. LLVM here is the CPU/MCU backend leg specifically ([Backend Lowering Architecture](backend-lowering-architecture.md) §2), not a universal backend: continuation lowering commits to LLVM coroutine intrinsics because that leg targets a native call stack, whereas backend legs without a continuation ABI (a hardware target such as CIRCT/FPGA) are not continuation destinations. The two destinations are:

- **LLVM coroutines** (§5.1) — `cont.suspend`/`resume` lower to `llvm.coro.*` intrinsics, which the LLVM coroutine passes split into a resumable frame. This is the general destination on the native-CPU leg.
- **Stack switching** (§5.2) — on a target exposing first-class suspend/resume as a primitive, the ops lower to it directly. This was always the same LLVM destination reached through the coroutine ABI; the stack-switching primitive is how the coroutine's suspend/resume is realized where the target provides it natively.

Under the §5.1 lowering (the freestanding leg, [Backend Lowering Architecture](backend-lowering-architecture.md) §5), the coroutine frame is the resumable state that a `seq` expression already uses ([Sequence Expression Representation](seq-representation.md)) — the same struct, generalized to capture the **delimited remainder** rather than only loop-local state (§4). No separate scheduler runtime is introduced. Whether the resume semantics also constitute the entire scheduler is a separate, target-cardinality question, addressed in §7: on a **single-core** target the suspend/resume semantics **are** the cooperative scheduler, because there is one thread of control to hand back and forth. That reduction turns on the core count, not on the leg being freestanding, and the same continuation surface lowers unchanged to hosted and multi-core targets.

The choice among destinations is a per-call-site lowering decision, not a change to the operation surface or its semantics. The transportability is the design: a front end emits the continuation ops once, and each target supplies a lowering pass — the same discipline the framework applies to platform bindings and device sources elsewhere.

## 2. The Continuation Operation Surface

The normative object of this chapter is the operation surface, not any one lowering. A delimited continuation is expressed by these target-neutral operations (naming after the standard coroutine-intrinsic ABI; the surface is what a front end emits and what every target lowering consumes):

| Operation | Role |
|-----------|------|
| `cont.new @f : !cont<τ>` | create a continuation from a function; `τ` is the suspend/resume value type |
| `cont.suspend (%v) : (τ) -> ()` | suspend, passing `%v` to whoever resumed the continuation |
| `cont.resume %k, %v` | resume `%k`, delivering `%v`; a handler receives control if `%k` suspends again |
| `cont.alloc` / `cont.store` / `cont.load` | reserve a slot for a continuation, stash a suspended one, reload it |
| `cont.is_done %k` | whether `%k` has run to its delimiter |

Clef developers do not write these operations; they write `async { }`, actor behaviors, and computation expressions whose `let!` desugars to continuation capture (see [Native Type Mappings](native-type-mappings.md), the DCont/Inet routing table). An `async` `let!`, an actor `receive`, and a synchronous RPC ([synchronous-rpc-liveness](synchronous-rpc-liveness.md)) each lower the front-end syntax to one `cont.suspend`; their resume trigger differs (I/O completion, message arrival, reply delivery) but the operation surface is identical. The surface is transportable: it fixes *what* a continuation is (create, suspend, resume, and the memory that carries a suspended one), and each target supplies *how* by a lowering pass (§5).

## 3. Relationship to Sequence Expressions and Closures

The continuation surface joins a family of resumable-computation representations, each extending the flat-closure architecture of [Closure Representation](closure-representation.md):

| Form | Suspends to | Captured state | Resumed by |
|------|-------------|----------------|------------|
| `seq { }` ([seq](seq-representation.md)) | yield a value outward | loop-local internal state | caller pulling `MoveNext` |
| **delimited continuation** (this chapter) | await a value inward | the delimited remainder | `cont.resume` with a value |
| `Observable` / `Incremental` | (opaque, push-driven) | subscriber closure | emission / staleness |

The relationship is a lowering fact, not a definitional one: on the freestanding target the coroutine frame a continuation lowers to (§5.1) is the same resumable-computation struct `seq` already uses. What it adds is the **capture set** — the values live across a `cont.suspend`, i.e. the delimited remainder up to the continuation's origin, not only loop-local state. This is the `seq` "captures vs internal state" split ([seq §3.2](seq-representation.md)) with the capture set computed at each suspension point. The surface of §2 does not name a struct; §5.1 does, as one destination.

## 4. Suspended-Continuation State (target-neutral)

Independent of lowering, a suspended continuation carries the state the surface of §2 implies: which suspension point it paused at, the value in flight, and everything live across the suspension. A conforming lowering SHALL realize this state; §5 gives the two realizations.

- **Suspension index** — which `cont.suspend` the continuation paused at (`0` = not yet started; `1..N` = paused at the Nth suspension; done = ran to its origin).
- **In-flight value** — the value delivered by the last `cont.resume` (read inward) or passed by the last `cont.suspend` (read outward by the resumer).
- **Capture set** — the bindings live on the path from a `cont.suspend` to the continuation's origin, unioned across all suspension points. This is the delimited remainder; it is what distinguishes a continuation from a `seq` (whose captured state is only loop-local).
- **Internal state** — `let mutable` bindings threaded across suspensions.

The capture-set computation is the one genuinely new analysis relative to `seq`: `seq` captures only loop-local state, a continuation captures the delimited remainder. It is computed per suspension point during elaboration and consumed (not recomputed) by whichever lowering §5 selects.

## 5. Target Lowerings

The surface of §2 transports to two destinations. Both reach the native-CPU backend leg through LLVM ([Backend Lowering Architecture](backend-lowering-architecture.md) §2 — LLVM is the CPU/MCU leg, one of several); they differ only in whether the target realizes suspend/resume as a coroutine frame the compiler builds or as a primitive the target provides. The selection is per call site (a lowering decision), and neither changes the operation surface or the state of §4.

### 5.1 LLVM coroutines (the freestanding state machine)

On the general native path, `cont.suspend`/`resume` lower through the LLVM coroutine intrinsics (`llvm.coro.*`), whose split pass realizes the §4 state as a resumable frame. On a **freestanding target** the frame is the `seq` state-machine struct ([seq §4.1](seq-representation.md)) — byte-compatible, with `seq`'s `current` reinterpreted as the in-flight value — so the allocation, escape-classification, and arena-placement rules of [Memory Regions](memory-regions.md) apply unchanged. When the continuation's scope is bounded (the common case when a freestanding target drives an event loop) the frame is stack- or arena-allocated with no heap involvement, and resume is the `seq` CFG of [seq §5.2](seq-representation.md): an `entry` block loads the suspension index and `switch`es to the per-suspension resume blocks, each of which delivers the in-flight value, advances the index, and returns; the origin block marks done. On this target it reduces to a single `llvm.switch %index, %resume_blocks` — no coroutine runtime library, no heap frame, no scheduler.

Resume on this path has the `MoveNext` shape of [seq §5.1](seq-representation.md):

```
resume: (ptr<Cont<T>>, T) -> i1   // true if it suspended again, false at the origin
```

### 5.2 Stack switching

On a target that exposes suspend/resume as a first-class primitive, the surface lowers to that primitive directly rather than to a compiler-built frame. This is the same LLVM coroutine destination reached natively: the coroutine's suspend and resume are realized by the target's stack-switch, so the continuation is preserved as a first-class value rather than reified into a state-machine struct. The operation surface of §2 is unchanged; only the realization differs, which is the point of specifying the surface separately from the lowering.

## 6. PSG Structure

A continuation region saturates to a `ContStateMachine` node, the analogue of the `SeqStateMachine` node of [Program Semantic Graph §12.5](program-semantic-graph.md):

```
ContStateMachine {
    Delimiter:      NodeId              // the reset boundary
    SuspendPoints:  (NodeId * int) list // each shift and its state index
    CaptureSet:     NodeId list         // bindings live across suspensions
    InternalState:  NodeId list         // let mutable across suspensions
    ResumeBlocks:   NodeId list         // per-suspension resume code
}
```

The suspension-point state indices are assigned by a coeffect analogous to the Yield State Analysis of [Program Semantic Graph §14.3.3](program-semantic-graph.md), computed during preprocessing and consumed (not recomputed) during witnessing. The saturation principle is unchanged: CCS builds the `ContStateMachine`; code generators only witness it.

## 7. NORMATIVE: Single-Core Cooperative Scheduling

Delimited continuations are pervasive in Clef and carry no scheduling role of their own; the operation surface of §2 is what `async`, actor `receive`, and every suspension point lower to, on every target. On a **single-core** small-form-factor target ([Backend Lowering Architecture](backend-lowering-architecture.md) §5 defines the freestanding leg), one further fact holds: the suspend/resume semantics of that surface **become** the cooperative scheduler. This is a forcing function of the braid, not a property of continuations. With one thread of control and no room for a separate scheduler runtime, a continuation that suspends is the only thing that can hand the core to another, so the braid presses the continuation's own suspend/resume into the scheduler's role — no task queue or preemption mechanism is introduced because none can be. The operative constraint is the single core, not the freestanding host: a freestanding leg is not inherently single-core, and where the target has more cores the braid coordinates them by other means, out of this section's scope. This section states the single-core discipline normatively so it can be relied upon; it holds regardless of which §5 lowering realizes the surface.

A conforming single-core implementation SHALL observe:

1. **Continuations are tasks.** A suspended continuation is a runnable unit. There is no task object distinct from the continuation itself.

2. **Resume is the scheduling event.** A continuation runs only when resumed (`cont.resume`) by delivery of its awaited value. On a bare-metal freestanding target the delivery source is a peripheral interrupt (or a completion the interrupt records); the interrupt-to-continuation binding is the scheduler's dispatch.

3. **Run-to-completion until the next `cont.suspend` (non-preemption).** Once resumed, a continuation runs until it reaches its next `cont.suspend` or its origin. It is never preempted mid-remainder. This is the cooperative guarantee: a continuation yields the core only at a suspension point, never involuntarily.

4. **Quiescence to low power.** When no continuation is runnable — every continuation is suspended awaiting a value — the implementation SHALL return the core to a wait state (e.g. `WFI` on Arm) until the next interrupt. The interrupt resumes the awaiting continuation, and the cycle repeats.

The interrupt handler that delivers a value SHALL be a captureless top-level handler bound directly to the vector-table slot (hardware vectors carry no environment pointer); it reaches continuation state through the resume entry point, not through a captured closure. The progress guarantee under this discipline is that every continuation whose awaited value has been delivered is eventually resumed; on a single core with one interrupt source per suspension class this reduces to "the handler resumes the awaiting continuation," and no fairness policy beyond interrupt priority is required.

> This is the sense in which the concurrency model reduces to an event loop *on a single core*:
> the braided crossing (a `cont.suspend` that spawns an awaited computation
> and threads its result back through the remainder) is held by the continuation, and where the
> target has one core the continuation's own suspend/resume is the whole of the scheduling. The
> reduction is a property of the core count, not of the freestanding host — the same continuation
> semantics carry to a multi-core target, where the scheduling is more than one event loop.

## 8. SSA Cost

On the §5.1 freestanding lowering the resume state machine has the `seq` SSA cost profile ([seq §9](seq-representation.md)): one `switch` at entry, plus per-suspension the value-store and index-store, plus the capture-set loads on each resume. No allocation-site cost is added for a scope-bounded continuation (stack/arena), matching the zero-heap requirement of a freestanding target.

## 9. Normative Requirements

1. **Operation surface.** A delimited continuation SHALL be expressed by the target-neutral operation surface of §2 (`cont.new` / `cont.suspend` / `cont.resume` and the continuation memory ops), which every target lowering consumes.
2. **Suspended state.** A lowering SHALL realize the suspended-continuation state of §4: the suspension index, the in-flight value, the capture set, and internal state.
3. **Capture set.** The capture set SHALL be the bindings live on the path from each `cont.suspend` to the continuation's origin, unioned across suspension points — the delimited remainder.
4. **Freestanding lowering.** On the freestanding native path the surface SHALL lower per §5.1: the `seq` state-machine struct as the coroutine frame, resume with the `seq` CFG and calling convention, and no continuation-runtime, coroutine-library, or scheduler-runtime dependency.
5. **Transportability.** The operation surface SHALL be independent of the lowering; a different target (e.g. stack switching, §5.2) SHALL be a different lowering pass over the same surface, not a change to it.
6. **Cooperative discipline.** A single-core implementation SHALL observe the scheduling discipline of §7 (continuations-as-tasks, resume-as-dispatch, run-to-completion, quiescence-to-wait).

## 10. Related Chapters

- [Sequence Expression Representation](seq-representation.md) — the state-machine struct and state-index discipline the §5.1 freestanding lowering reuses.
- [Closure Representation](closure-representation.md) — the flat-closure architecture both extend; the observer continuation as a resumed consumer.
- [Program Semantic Graph](program-semantic-graph.md) — `SeqStateMachine` (§12.5) and Yield State Analysis (§14.3.3), the analogues of `ContStateMachine` and its state-index coeffect.
- [Synchronous RPC and Liveness](synchronous-rpc-liveness.md) — one specified suspension point (the actor reply wait) that lowers to a `cont.suspend` and whose wait-for edge is rank-checked.
- [Native Type Mappings](native-type-mappings.md) — the DCont/Inet routing table: which regions lower through delimited continuations versus the parallel targets.

## References

- Danvy, O., Filinski, A. *Abstracting Control* (1990) — shift/reset, the source-level control operators the `cont.*` surface represents.
- Dybvig, R. K., Peyton Jones, S., Sabry, A. *A Monadic Framework for Delimited Continuations* (JFP 2007).
- Kang, B., Desai, H., Jia, L., Lucia, B. *WAMI: Compilation to WebAssembly through MLIR without Losing Abstraction* (2025), arXiv:2506.16048 — the `DCont`/coroutine-intrinsic dialect and its `CoroToLLVM` / `CoroToWAMI` lowerings that this surface-and-transport structure follows.
- Appel, A. W. *SSA is Functional Programming* (1998) — the state-machine/CFG equivalence the resume lowering rests on.
