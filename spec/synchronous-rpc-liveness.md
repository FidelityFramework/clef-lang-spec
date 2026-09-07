---
title: "Synchronous RPC and Wait Classification"
weight: 520
category: Semantics
status: normative
---

Clef classifies every synchronous request-and-reply call between actors by its contribution to the program's wait-for graph. The classification is computed at compile time, governs whether deadlock freedom is statically guaranteed for that call, and is surfaced to the developer through diagnostics and an opt-in annotation. The design rationale and its relationship to the session-types literature are developed in the [Deadlock Freedom design note](https://clef-lang.com/docs/design/concurrency/deadlock-freedom-as-an-obligation/). This section specifies the normative behavior.

## Scope

This section governs synchronous inter-actor calls: a call that suspends the caller's continuation until the callee replies on a reply channel. Asynchronous sends (`Tell`) do not suspend the caller, contribute no edge to the wait-for graph, and are outside this section.

| Construct | Suspends caller | Wait-for edge | Classified |
|-----------|-----------------|---------------|------------|
| `actor <! msg` (Tell) | No | None | No |
| `actor.PostAndReply msg` | Yes | caller → callee | Yes |
| Parked `Actor.receive()` on a reply | Yes | caller → completer | Yes |

## The Wait-For Relation

Let \(W\) be a directed relation over actor behaviors. An edge `caller → callee` is in \(W\) whenever the caller issues a synchronous RPC to the callee and suspends until the reply. \(W\) is a may-wait over-approximation: a call site whose callee is selected from several candidates by control flow contributes an edge to each candidate.

Acyclicity of a sound may-wait relation is sufficient to exclude circular synchronous waits represented by that relation. A cycle in the over-approximation is a candidate: its edges may come from paths that cannot be taken together. A diagnostic asserting a feasible synchronous deadlock SHALL have evidence that the participating waits can be reached simultaneously. Without such evidence or a proved ordering that excludes the candidate, the static guarantee remains unresolved.

A finite sound summary can resolve the possible callees of a value-carried reference. Classification SHALL use the established dependency information rather than treating value carriage itself as a proof of undecidability. The analysis must account for all possible blocking dependencies within the region for which it claims the static guarantee.

The relation is carried on the joint-constraint axis of the [Program Semantic Graph](program-semantic-graph.md): each synchronous RPC node contributes a blocking-wait hyperedge whose source is the caller's suspended continuation and whose target is the callee's reply obligation.

## Wait Classification

Every synchronous RPC call site receives one classification:

```fsharp
type WaitClass =
    /// Possible callees covered by a sound summary, W acyclic at this site.
    /// No circular wait through the represented dependencies.
    /// No annotation, no diagnostic.
    | AcyclicStatic

    /// Connection cycle with a proved acyclic ordering of blocking actions.
    /// Priority inferred from checked dependencies; no annotation required.
    | OrderedCyclic of priority: int

    /// Static ordering remains unproved, or supervision was requested.
    /// Diagnostic emitted; supervised execution uses a timeout.
    | Unresolved of reason: WaitReason

and WaitReason =
    | UnresolvedRouting of RoutingKind
    | UnprovedWaitOrder
    | RequestedSupervision

and RoutingKind =
    | SelfReferencePassed     // Actor.self() sent for the callee to reply through
    | ContentRouted           // callee chosen by message content
    | DynamicHandle           // callee handle produced by runtime spawn
 
```

### AcyclicStatic

A sound summary covers the possible callees, and the strongly-connected-component analysis of \(W\) places this site in no cycle. This excludes circular synchronous waits through the represented dependencies. No annotation is written and no diagnostic is emitted.

### OrderedCyclic

The possible callees are covered by a sound summary and the site lies on a connection cycle, while the justified action-dependency graph admits a topological order. A connection cycle records references between actors. A blocking cycle records actions waiting on one another. A strictly increasing priority along every possible blocking chain excludes the latter. The priority is inferred from checked dependencies. No annotation is written.

A developer may propose a priority with `[<RpcPriority(n)>]` or select among valid orders. The annotation supplies an ordering obligation. The implementation SHALL check it against the possible blocking actions and the premises used to exclude candidate edges. Labeling actions with priorities cannot remove a blocking dependency. An explicit priority that contradicts a required ordering, or whose required justification cannot be established, is a compile error.

### Unresolved

The static guarantee is withheld when the possible routing lacks a sound summary or the required wait ordering remains unproved. `UnresolvedRouting` records the former. `UnprovedWaitOrder` includes a candidate cycle over known endpoints whose simultaneous feasibility remains unresolved. The compiler emits a diagnostic naming the missing justification, and the call lowers to supervised execution governed by a timeout. The program is accepted under that runtime contract.

An explicit request for supervised execution records `RequestedSupervision`. A cycle proved to permit simultaneous blocking SHALL be diagnosed as a feasible synchronous deadlock and rejected unless the relevant calls explicitly use the supervised contract. A candidate cycle without that feasibility evidence SHALL be described as a conservative finding, with the unproved sites classified `Unresolved`.

## Compile-Time Behavior

Classification runs over the resolved Program Semantic Graph using sound actor-reference summaries. A strongly-connected-component pass detects candidate cycles in \(W\), and a topological ordering constructs priorities for an acyclic action-dependency graph. These are graph algorithms and do not require an SMT query. Establishing the summaries or proving that candidate blocking edges cannot occur together can require additional evidence. A completed graph check can establish a cycle in the analyzed relation while leaving its runtime feasibility unresolved. Failure to compute or validate a rank discharges no static guarantee.

The ordering obligation is a Tier 2 obligation in the verification architecture. Where it depends on a library actor property, such as a pool never calling back into its caller, an applicable library lemma can supply that fact through a mode interface. The implementation SHALL establish the lemma's premises and retain the correspondence that makes its conclusion applicable to the wait relation.

Classification is sound and conservative. `AcyclicStatic` and `OrderedCyclic` exclude circular synchronous waits through the represented dependencies under their checked premises. An execution-safe site may remain `Unresolved` when the required summary or ordering is unavailable. Scheduler progress and any blocking outside this chapter's scope require their own realization contracts.

## Developer Override

The classification is visible and steerable at the call site. Three resolutions are available when a site is flagged:

```fsharp
// 1. Propose a priority whose ordering obligations the compiler checks.
[<RpcPriority(2)>]
let response = inventory.PostAndReply (Query item)

// 2. Refactor the call to an asynchronous send, removing the wait-for edge.
inventory <! Query (item, replyTo)

// 3. Opt the call out of the static guarantee into supervised execution.
[<SupervisedRpc(timeoutMs = 5000)>]
let response = router.PostAndReply (Route msg)
```

`[<RpcPriority(n)>]` and `[<SupervisedRpc(...)>]` are CCS attributes. A call carrying `[<SupervisedRpc(...)>]` is classified `Unresolved RequestedSupervision` regardless of its callee resolvability, which lets a developer accept the runtime discipline deliberately for a call that would otherwise be statically guaranteed.

> Attribute names are provisional. `RpcPriority` and `SupervisedRpc` are placeholders pending the final attribute vocabulary. The semantics specified here are stable.

## Compiler Intrinsic Status

CCS (Clef Compiler Service) SHALL perform wait classification during graph resolution and record it on the RPC node. The following annotation and diagnostic sketches specify the required association:

```fsharp
// In CCS graph annotation
type NodeAnnotation =
    // ...
    | RpcWait of WaitClass

// Diagnostic emission for the Unresolved case
| Unresolved reason ->
    diagnostic CCS8030 node.Span
        (sprintf "synchronous RPC uses supervised timeout: %s"
                 (describeWaitReason reason))
```

A cycle diagnostic SHALL report the candidate chain and whether simultaneous blocking has been established. CCS8031 reports an error for a feasible blocking cycle in the statically guaranteed path. For an unresolved candidate, it explains the conservative finding alongside CCS8030 and the supervised classification:

```fsharp
| WaitCycle (path, feasibility) ->
    reportWaitCycle CCS8031 node.Span feasibility
        (renderWaitPath path)   // e.g. "A.handleFoo → B.query → A.handleBar"
 
```

Alex SHALL preserve the checked wait semantics when lowering `AcyclicStatic` and `OrderedCyclic` calls against the callee's reply obligation. It SHALL lower `Unresolved` calls through the supervised path, which arms a timeout and routes expiry to the caller's supervisor. The target contract SHALL provide the scheduling, cancellation, and cleanup behavior needed for timeout recovery. The static ordering classification does not establish those runtime properties by itself.

## Diagnostics

| Code | Message |
|------|---------|
| CCS8030 | Synchronous RPC uses supervised timeout: unresolved routing, unproved wait order, or explicit supervision request |
| CCS8031 | Synchronous wait cycle: candidate chain with feasibility unresolved, or established simultaneous blocking |
| CCS8032 | Explicit `RpcPriority` ordering obligation is contradicted or remains unproved |
| CCS8033 | `SupervisedRpc` timeout must be a positive integer |

## Grammar

```fsgrammar
rpc-attribute :=
    [< RpcPriority ( int-literal ) >]
    [< SupervisedRpc ( timeoutMs = int-literal ) >]

wait-class :=
    AcyclicStatic
    OrderedCyclic
    Unresolved

wait-reason :=
    UnresolvedRouting of routing-kind
    UnprovedWaitOrder
    RequestedSupervision

routing-kind :=
    SelfReferencePassed
    ContentRouted
    DynamicHandle
```
