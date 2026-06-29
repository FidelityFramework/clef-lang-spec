---
title: "Synchronous RPC and Wait Classification"
weight: 520
draft: true
---

Clef classifies every synchronous request-and-reply call between actors by its contribution to the program's wait-for graph. The classification is computed at compile time, governs whether deadlock freedom is statically guaranteed for that call, and is surfaced to the developer through diagnostics and an opt-in annotation. The design rationale and its relationship to the session-types literature are developed in the [Deadlock Freedom design note](https://clef-lang.com/docs/design/deadlock-freedom/); this section specifies the normative behavior.

## Scope

This section governs synchronous inter-actor calls: a call that suspends the caller's continuation until the callee replies on a reply channel. Asynchronous sends (`Tell`) do not suspend the caller, contribute no edge to the wait-for graph, and are outside this section.

| Construct | Suspends caller | Wait-for edge | Classified |
|-----------|-----------------|---------------|------------|
| `actor <! msg` (Tell) | No | None | No |
| `actor.PostAndReply msg` | Yes | caller → callee | Yes |
| Parked `Actor.receive()` on a reply | Yes | caller → completer | Yes |

## The Wait-For Relation

Let \(W\) be a directed relation over actor behaviors. An edge `caller → callee` is in \(W\) whenever the caller issues a synchronous RPC to the callee and suspends until the reply. \(W\) is a may-wait over-approximation: a call site whose callee is selected from several candidates by control flow contributes an edge to each candidate.

A program has a synchronous deadlock when \(W\) contains a cycle whose actors can be simultaneously in their blocked-on-reply state. For the fragment in which every callee is a statically resolvable actor reference, \(W\) is finite and deadlock freedom is equivalent to its acyclicity.

The relation is carried on the joint-constraint axis of the [Program Semantic Graph](program-semantic-graph.md): each synchronous RPC node contributes a blocking-wait hyperedge whose source is the caller's suspended continuation and whose target is the callee's reply obligation.

## Wait Classification

Every synchronous RPC call site receives one classification:

```fsharp
type WaitClass =
    /// Callee statically resolvable, W acyclic at this site.
    /// Deadlock-free by construction. No annotation, no diagnostic.
    | AcyclicStatic

    /// Callee statically resolvable, participates in a connection cycle
    /// that admits a consistent priority order. Deadlock-free by
    /// construction. Priority inferred; no annotation required.
    | OrderedCyclic of priority: int

    /// Callee identity is value-carried (passed reference, content-based
    /// routing, runtime-spawned handle). Acyclicity is not statically
    /// decidable. Static guarantee is withdrawn; the call falls back to
    /// supervised execution with a timeout. Diagnostic emitted.
    | Unresolved of routing: RoutingKind

and RoutingKind =
    | SelfReferencePassed     // Actor.self() sent for the callee to reply through
    | ContentRouted           // callee chosen by message content
    | DynamicHandle           // callee handle produced by runtime spawn
```

### AcyclicStatic

The callee is a statically resolvable reference and the strongly-connected-component analysis of \(W\) places this site in no cycle. The call is deadlock-free by construction. No annotation is written and no diagnostic is emitted.

### OrderedCyclic

The callee is statically resolvable and the site lies on a connection cycle, but the action-dependency graph admits a topological order. A consistent priority assignment exists, and a strictly increasing priority along every blocking chain forbids a wait cycle. The priority is inferred from the wait-for edges. No annotation is written.

A developer may state the priority explicitly with `[<RpcPriority(n)>]` when inference cannot resolve a consistent order, or to fix an order the inference would otherwise leave to its default. An explicit priority that does not yield an acyclic order is a compile error.

### Unresolved

The callee identity is value-carried, so the wait-for edge cannot be resolved at compile time and acyclicity over the relevant routing is not statically decidable. The static guarantee is withdrawn for this site. The compiler emits a diagnostic naming the routing kind, and the call lowers to supervised execution governed by a timeout. The program is accepted.

## Compile-Time Behavior

Classification runs over the resolved Program Semantic Graph after actor references are bound. The decision procedure is a strongly-connected-component pass over \(W\) for cycle detection and a topological ordering of the action-dependency graph for priority inference. Both are graph algorithms over structure the graph already carries; neither requires an SMT query.

The obligation discharged is a Tier 2 obligation in the verification architecture. Where a site's acyclicity depends on a property of a library actor (for example, that a supervised pool never calls back into its caller), the obligation lifts by a mode shift to the tier at which a library lemma supplies that property, then projects the verified ordering back.

Classification is sound and conservative. A site classified `AcyclicStatic` or `OrderedCyclic` admits no execution that deadlocks through it. A site that no execution deadlocks through may still be classified `Unresolved` when its callee is value-carried.

## Developer Override

The classification is visible and steerable at the call site. Three resolutions are available when a site is flagged:

```fsharp
// 1. State an explicit priority to break a cycle the inference leaves open.
[<RpcPriority(2)>]
let response = inventory.PostAndReply (Query item)

// 2. Refactor the leg to an asynchronous send, removing the wait-for edge.
inventory <! Query (item, replyTo)

// 3. Opt the call out of the static guarantee into supervised execution.
[<SupervisedRpc(timeoutMs = 5000)>]
let response = router.PostAndReply (Route msg)
```

`[<RpcPriority(n)>]` and `[<SupervisedRpc(...)>]` are CCS attributes. A call carrying `[<SupervisedRpc(...)>]` is classified `Unresolved` regardless of its callee resolvability, which lets a developer accept the runtime discipline deliberately for a call that would otherwise be statically guaranteed.

> Attribute names are provisional. `RpcPriority` and `SupervisedRpc` are placeholders pending the final attribute vocabulary; the semantics specified here are stable.

## Compiler Intrinsic Status

Wait classification is performed by CCS (Clef Compiler Service) during graph resolution. The classification is recorded as an annotation on the RPC node:

```fsharp
// In CCS graph annotation
type NodeAnnotation =
    // ...
    | RpcWait of WaitClass

// Diagnostic emission for the Unresolved case
| Unresolved routing ->
    diagnostic CCS8030 node.Span
        (sprintf "synchronous RPC callee is %s; static deadlock-freedom \
                  guarantee withdrawn, call lowered to supervised timeout"
                 (describeRouting routing))
```

A flagged cycle in the statically resolvable fragment reports the wait-for path:

```fsharp
| StaticCycle path ->
    diagnostic CCS8031 node.Span
        (sprintf "synchronous wait cycle: %s"
                 (renderWaitPath path))   // e.g. "A.handleFoo → B.query → A.handleBar"
```

Alex lowers `AcyclicStatic` and `OrderedCyclic` calls to direct continuation suspend-and-resume against the callee's reply obligation. It lowers `Unresolved` calls through the supervised path, which arms a timeout and routes expiry to the caller's supervisor.

## Diagnostics

| Code | Message |
|------|---------|
| CCS8030 | Synchronous RPC callee is value-carried; static deadlock-freedom guarantee withdrawn, call lowered to supervised timeout |
| CCS8031 | Synchronous wait cycle detected among statically resolvable actors |
| CCS8032 | Explicit `RpcPriority` does not yield an acyclic order |
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

routing-kind :=
    SelfReferencePassed
    ContentRouted
    DynamicHandle
```
