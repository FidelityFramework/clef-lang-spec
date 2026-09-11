---
title: "Scheduler Contract"
weight: 525
category: Semantics
status: normative
---

Clef actor execution is realized by **Olivier** (the actor runtime) under **Prospero** (the supervisor). Dispatch within that system is realized by **Ariel** (the scheduler). This file specifies the contract every conforming Ariel implementation SHALL satisfy. The contract exists so that obligations discharged elsewhere in this specification have a named premise: the acyclicity discharge of [Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md) rules out cyclic waiting, and the progress of the acyclic remainder rests on the clauses below.

> **Status (July 2026)**: This contract is design-stage. No conforming implementation exists. The simulated profile (§7) is the first implementation target, because it doubles as the conformance harness for the other profiles.

## 1. Scope

This file specifies the observable dispatch discipline of the scheduler: fairness, turn semantics, control-plane capacity, admission, determinism, and the per-implementation assumption manifest. It does not specify supervision policy, restart strategy, or actor placement, which belong to Prospero, and it does not specify actor semantics, mailbox ordering within a delivery, or message representation, which belong to Olivier and to [BAREWire](native-type-universe.md). It does not prescribe implementation mechanism. A conforming implementation MAY be a thread pool, an event loop, an interrupt binding, or a lowered continuation surface, provided the clauses hold.

**NORMATIVE**: User code SHALL NOT address the scheduler. The scheduler's clients are the supervisor and the compiler. No operation of this contract is exposed as a language-level API.

The dependency direction is part of the contract's purpose. Prospero's supervision guarantees are theorems whose premises are the clauses of this file, so the supervisor is specified against the scheduler, and the scheduler is specified against the execution substrate beneath it, never the reverse.

## 2. Definitions

**scheduling event** — a resume: the delivery of an awaited value to a suspended continuation. [DCont Representation](dcont-representation.md) establishes that a continuation runs only when resumed and that on bare-metal targets the interrupt-to-continuation binding is the scheduler's dispatch. This file adopts that definition unchanged.

**turn** — the execution of an actor from one scheduling event to its next suspension, completion, or fault. A turn is the unit of dispatch.

**ready** — the state of an actor whose mailbox holds a deliverable message or whose awaited value has arrived, and which is not presently executing a turn.

**data plane** — message delivery, mailbox traffic, and turn execution on behalf of user-defined actors.

**control plane** — supervision actions (restart, retirement, escalation), timer expiry, and actor-reference state maintenance.

**assumption manifest** — the published partition, per implementation, of this contract's clauses into those the implementation discharges itself and those it assumes from the substrate beneath it (§7).

**latency domain** — a set of processing elements over which a turn-granularity dispatch decision is meaningful: one core, one coherent multicore package, one board-level fabric. A network hop always crosses a latency domain.

> **Clef Note**: The word *sentinel* in this specification names the numeric width and representation placeholders of [Numeric Selection](numeric-selection.md) (the sentinel/resolver pattern). The runtime structure that carries an actor reference's liveness state is called the *actor-reference sentinel* in the design documentation. Where this file needs that concept it says **reference state**, and §10 gives the states.

## 3. Fairness

An actor that is ready SHALL be dispatched within a finite number of scheduling events. The bound MAY vary with load and with implementation profile, and it SHALL exist in every execution.

This clause supplies a dispatch premise for liveness results. The rank witness of [Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md) establishes that the wait-for relation is acyclic for the resolvable fragment. Acyclicity rules out cycles; fairness ensures dispatch of ready actors. A claim that an awaited reply eventually arrives SHALL additionally identify the required turn-progress, delivery, and external-completion premises. Fairness SHALL NOT be used to infer successful completion of an unavailable or failed external operation.

## 4. Turn Discipline

A turn SHALL run to completion. The scheduler SHALL NOT preempt a turn, migrate a turn between processing elements mid-execution, or interleave another turn of the same actor.

An implementation MAY attach a budget to a turn. Exhaustion of a turn budget is a fault. It SHALL be reported to the supervisor and SHALL NOT be handled as a scheduling event. The supervisor's remedies operate at actor-lifecycle granularity: retirement, restart with a fresh arena, escalation. Preemption in the Olivier/Prospero system therefore exists above the turn, as supervision policy, and never below it, as dispatch mechanism.

The actor definition in [Terms and Definitions](terms-and-definitions.md) relies on turn atomicity: one logical thread per actor, no shared mutable state, and consequently no intra-actor data race. An implementation whose substrate preempts its carrier threads (a hosted OS scheduler) SHALL preserve turn atomicity as observed by the program.

## 5. Control-Plane Immunity

Control-plane actions SHALL NOT contend with data-plane traffic for the same bounded capacity. When every data-plane resource an implementation bounds (mailbox slots, message envelopes, channel slots) is exhausted, the implementation SHALL still be able to: retire an actor, execute a supervisor restart, deliver a timer expiry, and update reference state.

On the native profile, the design realization is Prospero on an elevated thread with supervision executed as direct calls rather than as messages competing for data-plane capacity. On the freestanding single-core profile the separation is hardware exception priority rather than a thread: dispatch runs in the processor's thread mode, and supervision, timer expiry, and turn-budget enforcement execute at reserved handler priorities with capacities set at build time. A freestanding implementation SHALL arm the control-plane tier, its timer, its watchdog, and its vector entries, before dispatching the first turn. Other profiles satisfy the clause by equivalent separation.

## 6. Admission

Every implementation SHALL define its admission semantics and publish them in the assumption manifest. Admission at a mailbox either accepts the delivery or refuses it. A refusal SHALL be reported to the sender as a language-level result at the send site where the construct is synchronous with the sender, and counted where it is not.

An implementation SHALL NOT buffer on the sender's side of an inter-actor or inter-domain boundary to mask a receiver's exhaustion. A receiver's backpressure SHALL NOT become growth in the sender's memory.

A freestanding implementation SHALL bound every mailbox, envelope pool, and channel, with capacities taken from the build's capacity manifest. A hosted implementation MAY declare unbounded admission for a mailbox class, and in doing so assigns the memory consequence to the substrate and records that assignment in the assumption manifest.

## 7. Determinism Mode and the Assumption Manifest

The scheduler is the interpreter of suspension and resumption. A conforming implementation MAY substitute the sources it interprets: time, completion order, and message arrival. The substitution SHALL be total, meaning that identical sources and an identical program yield identical dispatch order.

A simulated implementation, whose sources are scripted, is a conforming implementation under this clause. It is the conformance harness for the other clauses, and it is the replay mechanism: a recorded admission sequence, replayed against the same program, reproduces the execution.

Each implementation SHALL publish an assumption manifest partitioning the clauses of this contract into those it discharges itself and those it assumes from its substrate. The following profile table is informative.

| Profile | Discharges locally | Assumes from substrate |
|---|---|---|
| Freestanding (unikernel, single core) | §3, §4, §5, §6 in full | hardware timer, interrupt delivery |
| MicroVM | §4, §5, §6 | vCPU progress and timer delivery against a quota named in the deployment specification |
| Container | §4, §5, §6 | carrier-thread progress against a budget that is observable rather than declared |
| Hosted (OS process, .NET fallback) | §4 as observed atomicity, §5, §6 per manifest | fairness of the substrate scheduler in full |
| Isolate (managed JavaScript substrate, JSIR pathway) | §4 actor-turn discipline through the adapter, §6 per manifest | Host callback serialization; §3 fairness and turn progress under explicitly named host assumptions; §5 per manifest, with control-plane actions sharing the host's event loop and execution budgets |
| Simulated | §3 through §6 by construction over scripted sources | nothing |

An isolate implementation SHALL NOT infer fairness or whole-handler atomicity solely from single-threaded JavaScript execution. Suspension ends a turn (§2); resumption SHALL preserve actor exclusivity and the validity conditions of facts used by that continuation. Host callback reentrancy and completion delivery SHALL follow the declared adapter contract. The assumption manifest SHALL distinguish host termination on budget exhaustion from a locally dispatched supervision action; a callback that does not yield cannot be assumed to admit another callback for supervision.

For a platform-supplied isolate execution model, the implementation MAY realize this contract through existing host facilities; this contract does not require a separate scheduler. The manifest SHALL distinguish host-provided dispatch from application-defined orchestration expressed through supported host APIs. In the Cloudflare realization, execution and dispatch remain platform-provided; generated bindings and orchestration do not introduce an independent Ariel dispatch implementation. A required property unsupported by the host SHALL be reported rather than attributed to an unimplemented scheduling mechanism.

> **Clef Note**: The determinism clause admits dispatch disciplines beyond mailbox order. A stabilization pass over the demanded, dirty fragment of a dependency graph, dispatched in dependency order, is a conforming implementation of this contract for the cold, pull-based side of the concurrency model. [Incremental Computation](incremental-computation.md) already places demand registration for actor-based nodes with the supervisor, and demand refines *ready*: an actor whose behavior is effect-free and whose outputs no consumer observes need not be dispatched at all.

On a single-core freestanding target, [DCont Representation](dcont-representation.md) already states that the suspend/resume semantics become the cooperative scheduler and that resume is the scheduling event. On that profile the conforming implementation is the lowered continuation surface together with the interrupt-to-continuation binding. There the contract names the discipline the reduction must preserve, and adds no mechanism of its own, so the smallest profile can be audited clause by clause.

The compiler is designed to emit the capacity manifest a freestanding profile provisions from, derived from escape classification and arena extents, so the numbers a boot image is provisioned with would be outputs of analysis rather than developer guesses.

## 8. Authority

An implementation occupies one of three authority positions, recorded in its manifest.

**Sovereign** — the only scheduler in its latency domain. The freestanding profile is sovereign by construction.

**Federated** — subordinate to a parent scheduler within one latency domain, as on a host package directing an accelerator fabric or a performance-core cluster directing an efficiency-core cluster. A parent SHALL NOT dispatch a subordinate's turns. It grants a budget and a region, and it SHALL hold the subordinate to this same contract. Coordination between schedulers is carried over BAREWire as ordinary structured messages on the type-informed fabric. No privileged shared-memory channel between schedulers is part of this contract.

**Replicated** — one sovereign scheduler per node across a distribution boundary, with no shared dispatch.

**NORMATIVE**: A turn-granularity dispatch decision SHALL NOT cross a latency domain. Cross-domain progress is a supervision concern, discharged with deadlines and escalation under Prospero, in the same posture as the `Unresolved` wait class of [Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md).

## 9. Control-Plane Observability

This contract separates watching-as-recording from watching-as-enforcement, because the two roles answer the watchers question differently. Enforcement escalates strictly outward: §5's control plane within an implementation, and beyond it a chain that SHALL terminate outside the implementation, at a hardware watchdog on the freestanding profile (armed before first dispatch, per §5) or at the substrate's own supervisor on hosted profiles. This contract provides no software watcher above the supervisor. Recording is the remainder of this section, and it requires no watcher because it holds no authority.

An implementation SHALL provide a bounded control-plane event record (a ring, in the natural realization) covering, at minimum: turn faults and budget exhaustions, restarts and retirements, escalations and quarantine decisions, admission-refusal counts, and the entry or reset cause where the substrate reports one. Writes to the record SHALL be wait-free and allocation-free, SHALL be permitted from the control plane's own execution context (Handler mode on the freestanding profile), and SHALL NOT enter the message fabric. The record is lossy by design: on overflow, the oldest entries are overwritten. No control-plane action SHALL depend on the acceptance of a record entry.

**NORMATIVE**: Dispatch SHALL NOT generate message-fabric traffic on its own behalf. The scheduler and the supervisor observe themselves through the record and never through messages, so that observation is never load.

An implementation MAY expose the record through a distinguished telemetry actor that drains it. The drain SHALL hold no supervision or dispatch authority. It SHALL be demand-driven: with no registered consumer it is not dispatched, in the sense of the §7 Clef Note. Its failure SHALL affect visibility only and never progress, which is why it is supervised as an ordinary actor and needs no watcher of its own.

A freestanding implementation SHOULD preserve a last-words region across reset, a no-init memory region read together with the reset cause, so that the record of the previous run is readable at the next boot, before the supervision tree is armed. Durable storage of fault records beyond that region is outside this contract's scope. Where a platform provides it, [Modular Blob Storage](modular-blob-storage.md) owns the durability commitments, and nothing in this section depends on them.

> **Clef Note**: The record and the determinism mode of §7 MAY share one mechanism. The recorded source sequence that makes replay possible and the telemetry stream are one object at two fidelities: total in a simulated implementation, a lossy prefix in production. Interpretation of user-level logging effects is Olivier's concern and out of scope here, except that the interpreter SHALL respect this section's invariants, with the turn as the natural batch boundary. The mechanism names in this section (ring, drain, last-words region) are illustrative. The invariants are the contract.

## 10. Dormant References and Hydration (Proposed)

> **Status (July 2026)**: This section is informative and proposed. Nothing in it is assumed by the normative clauses above.

The design documentation gives an actor reference four states: `Valid`, `ActorTerminated`, `ProcessUnavailable`, `Unknown`. This proposal adds a fifth, `Dormant`: the reference designates an actor whose identity is current and whose execution state is at rest.

Delivery to a `Dormant` reference is an admission decision at the actor's mailbox under §6, because the mailbox precedes the actor: addressability and admission capacity exist independently of whether execution state is hydrated. An accepted delivery raises a control-plane event to the resident supervisor. The supervisor decides hydration as policy, under a hydration budget symmetric with its restart budget, and the scheduler dispatches the initialization turn under §4. The message waits in the actor's own bounded mailbox for the duration. The supervisor holds the doorbell and never the mail: routing data-plane messages through the control plane would convert the supervisor into a queue, and §5 exists to keep those planes separate.

Hydration preserves the actor's identity, so a reference held across passivation and hydration reads `Valid` before and after, and the interval is a scheduling delay rather than a semantic event. Restart after a fault re-mints identity, so a stale reference reads `ActorTerminated` and the failure stays observable to every holder. A design in which the two are indistinguishable would surrender the supervision model's visibility, which is the reason this proposal keeps them distinct states rather than one transparent activation mechanism.

State at rest is a BAREWire layout. Passivation writes the actor's state through the same structural contract its messages use, and hydration maps that layout back. Both endpoints interpret the region by construction, so hydration never parses tagged bytes.

On the freestanding profile, hydration SHALL only activate capacity the build's capacity manifest provisioned. There is no novel allocation on that profile, only activation of what the compiler already sized.

The wire-crossing case, where a sender on one node holds a `Dormant` reference to an actor whose home is another node, composes the pieces above: admission at the remote mailbox, a control-plane event raised to the remote node's resident supervisor, hydration dispatched by that node's own scheduler. The sender neither observes nor orchestrates any of it. We are aware this reach of the design carries the most open questions, and we are taking some care to treat the referential-transparency and supervision-visibility consequences properly before any of it hardens into normative text.

## 11. Conformance Requirements

1. An implementation SHALL dispatch every ready actor within a finite number of scheduling events (§3).
2. An implementation SHALL run every turn to completion without scheduler preemption, and SHALL report turn-budget exhaustion to the supervisor as a fault (§4).
3. An implementation SHALL keep control-plane actions executable when all bounded data-plane capacity is exhausted (§5).
4. An implementation SHALL define admission semantics, report or count every refusal, and SHALL NOT buffer on the sender's side of a boundary to mask receiver exhaustion (§6).
5. An implementation MAY substitute time, completion, and arrival sources, and where it does, identical sources SHALL yield identical dispatch order (§7).
6. An implementation SHALL publish an assumption manifest naming its profile, its authority position, and the clauses it assumes from its substrate (§7, §8).
7. A federated implementation SHALL hold its subordinates to this contract and SHALL NOT dispatch their turns (§8).
8. No implementation SHALL expose a user-addressable scheduling API (§1).
9. An implementation SHALL provide the bounded control-plane event record, with wait-free, allocation-free, fabric-free writes, and no control-plane action SHALL depend on a record entry's acceptance (§9).
10. Dispatch SHALL NOT generate message-fabric traffic on its own behalf (§9).
11. A telemetry drain, where provided, SHALL be non-authoritative and demand-driven, and its failure SHALL affect visibility only (§9).
