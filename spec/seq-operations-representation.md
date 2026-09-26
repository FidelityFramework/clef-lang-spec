---
title: "Sequence Operations Representation"
weight: 350
category: Representation
status: normative
---

## 1. Overview

Sequence operations compose deferred producers and demand-driven consumers over Clef
`seq<'T>`. They preserve source NTU types, dimensions, evaluation order, capture
identity and admitted storage lifetime through the PSG and its proof obligations.

This chapter specifies operation behavior and the representation obligations of
composition. It extends [Sequence Representation](seq-representation.md) and
[Closure Representation](closure-representation.md); it does not define a second
wrapper-layout or callback system. [Lazy Representation](lazy-representation.md)
shares deferred formation, but sequence enumeration does not memoize a single
result for all consumers.

A producer retains its supplied configuration and values when formed. Each
subsequent enumeration owns independent progress. Referenced mutable storage can
remain shared across those enumerations. A consumer performs its required pulls
when the consumer application executes.

## 2. Source Contracts

The type parameters below are independently quantified native types. Equal type
parameters require agreement of the complete type, including dimensions; they do
not license object widening or a substitute physical carrier.

| Operation | Explicit parameter order | Source signature | Category |
|-----------|--------------------------|------------------|----------|
| `Seq.map` | `<'T,'U>` | `('T -> 'U) -> seq<'T> -> seq<'U>` | Deferred producer |
| `Seq.filter` | `<'T>` | `('T -> bool) -> seq<'T> -> seq<'T>` | Deferred producer |
| `Seq.append` | `<'T>` | `seq<'T> -> seq<'T> -> seq<'T>` | Deferred producer |
| `Seq.collect` | `<'T,'U>` | `('T -> seq<'U>) -> seq<'T> -> seq<'U>` | Deferred producer |
| `Seq.take` | `<'T>` | `int -> seq<'T> -> seq<'T>` | Deferred producer |
| `Seq.fold` | `<'S,'T>` | `('S -> 'T -> 'S) -> 'S -> seq<'T> -> 'S` | Demanded consumer |
| `Seq.iter` | `<'T>` | `('T -> unit) -> seq<'T> -> unit` | Demanded effect consumer |
| `Seq.exists` | `<'T>` | `('T -> bool) -> seq<'T> -> bool` | Demanded short-circuit consumer |
| `Seq.forall` | `<'T>` | `('T -> bool) -> seq<'T> -> bool` | Demanded short-circuit consumer |
| `Seq.tryHead` | `<'T>` | `seq<'T> -> option<'T>` | Demanded selection consumer |
| `Seq.tryPick` | `<'T,'U>` | `('T -> option<'U>) -> seq<'T> -> option<'U>` | Demanded selection consumer |

`fold` state and input element types are independent. For example, a folder may
consume `int<m>` elements and maintain an `int<s>` or `float<1/s>` state, provided
its argument and result types satisfy the declared scheme. The accumulator's
physical representation must follow its own range and target facts.

The `take` count is a dimensionless `int`; it does not select a machine word
size. Predicate results are `bool`, action results are `unit`, and a `collect`
mapper returns the corresponding sequence type. The compiler retains those
constraints at the operation's source application.

These contracts do not register other search or materialization operations by
analogy. `Seq.empty` follows [Sequence Representation §3](seq-representation.md#3-empty-and-zero-cut-sequences).
Other operations require their own source admission and behavioral contracts.

## 3. Formation and Application

Ordinary [application and pipeline rules](expressions.md#default-demand-and-sharing)
preserve each supplied operand's shared deferred identity. Supplying an operand
does not force it. A recipe must retain the actual value, suspension or storage
identity rather than replaying its expression at every pull or callback.

These are ordinary-operand defaults. Direct explicit
[`eager` operands](expressions.md#eager-expressions) are demanded at the activated
application or partial-formation frontier in source order. Their established
values remain shared through later completion and enumeration. An eager marker
inside an otherwise undemanded nested computation is not hoisted to this frontier.

Producer formation does not force callback/count/input computations solely
because they were supplied, execute the deferred body, pull an input or invoke a
callback. A demanded pull forces the operands required by that operation's
behavior. Input creation can have effects distinct from input enumeration; each
occurs only at its respective demand. A consumer likewise begins enumeration
when its result or iteration effect is demanded, not at syntactic application.

Partial application retains already supplied values or deferred identities at
its formation boundary. Supplying remaining operands neither forces unused
earlier operands nor recreates their suspensions or completed values.
A stored or bare operation value follows the same type and application rules;
these forms are not a distinct, weaker semantic API. For function-valued `fold`
state, the three declared operation operands remain the boundary before applying
the returned state function. Additional arguments remain deferred under their
subsequent application; selecting or returning a function does not invoke it.

An immutable captured binding contributes its established value or shared
deferred identity without forcing it. A value containing references preserves
their identities and sharing. A mutable binding
contributes its original storage cell, not a replacement cell initialized from a
scalar read. Re-enumeration does not rerun a producer operand initializer that
already completed, copy another iterator's progress or deep-copy shared captures.

Internal producer bindings establish their deferred computations when execution
reaches them during a pull and evaluate only when demanded. They are not
default-initialized or forced at producer formation. This distinction
also applies when a callback or child sequence is constructed inside a running
producer.

## 4. Deferred Producer Behavior

Each successful input pull makes one current value available for that iterator
and path. The following laws describe observations and demand; they are not
instructions for an Alex source-body emitter.

### 4.1 Map

`Seq.map mapper input` pulls the input when an output is demanded. On success it
retains current once and yields a shared deferred application of the mapper to
that value. Demanding that element demands the mapper result; reading only the
existence of an element does not force an unused mapped payload. Input exhaustion
completes the mapped enumeration without demanding the mapper. Input order is
preserved, and repeated demand for the same yielded value does not replay it.

A function-valued element is passed as a value; a function-valued mapper result
is yielded as a value. Neither is implicitly invoked by the operation.

### 4.2 Filter

`Seq.filter predicate input` pulls until a value is accepted or the input
exhausts. For each successful pull it reads current once and invokes the
predicate once. A true result yields that same selected value; a false result
continues pulling without an output element. Predicate and yield demands refer
to the same retained current snapshot.

An all-rejected finite input completes normally. Downstream demand may therefore
cause multiple upstream pulls for one accepted element, but a downstream stop
must prevent further pulls and predicate calls.

### 4.3 Append

`Seq.append first second` retains both supplied computations without forcing
them at formation. Enumeration demands and pulls the first input until it
exhausts, then demands the second. It does not force the second initializer or
pull its body after a downstream consumer has stopped within the first input.

Empty inputs contribute no elements. Their deferred exhaustion effects occur if
and when enumeration actually pulls them. A chain of empty inputs must preserve
those effects and proceed to the next input within the same outstanding demand.

### 4.4 Collect

`Seq.collect mapper outer` pulls one outer element, reads it once and invokes the
mapper once to obtain an inner sequence. It enumerates that inner result to
exhaustion before advancing the outer input. Inner values retain their order;
an empty inner result yields nothing and continues to the next outer element.

An outer suspension can retain an active inner iterator and its backing storage.
The mapper's returned sequence must have an admitted lifetime covering that use.
Storing its descriptor or identifying its code alone does not establish it.
When an inner iterator exhausts, its storage may be reused only under the
applicable lifetime and non-overlap premises.

### 4.5 Take

`Seq.take count input` yields at most `count` elements. A zero or negative count
causes neither input formation demand nor an input pull. Each attempted pull tests positive remaining demand first;
a successful pull yields current and consumes one unit of that demand.
Exhaustion before the requested count completes normally.

After the limit is reached there is no further upstream pull. In particular, the
operation must not pull first and discard an extra value, or run upstream
post-yield effects solely to discover that demand is already zero. Its count
updates require the same range and representation obligations as other integer
operations. No F#/CLR short-input exception behavior is imported into this law.

<a id="5-eager-consumer-behavior"></a>

## 5. Demanded Consumer Behavior

These laws apply when the consumer's result or explicit iteration effect is
demanded. Merely constructing or storing the application does not enumerate its
input. “Consume to exhaustion” describes required traversal once demanded, not
an eager default for argument initializers or intermediate accumulator values.

### 5.1 Fold

`Seq.fold folder initial input` starts with the shared deferred initial state.
For each successful input pull, it retains current once and forms the next shared
state computation `folder state current`. On exhaustion it returns the last
state. A demanded final result forces only the state computations and arguments
required by the folder; an implementation must not force an ignored initial or
intermediate state merely to place it in an accumulator slot. An empty input
returns the initial state without demanding the folder. A proven strict fold can
use an immediate accumulator while preserving these observations.

The accumulator preserves `'S` throughout initialization, reads, callback
application, writes and the result. It need not share the element's type,
dimensions, signedness or physical width. Stored integer/real values require
settled adaptation meets where their held representations differ; the witness
must not assume a universal accumulator width.

### 5.2 Iter

`Seq.iter action input` pulls to exhaustion, reading current and invoking the
action once per successful pull in input order. It returns unit. An empty input
invokes no action; an effectful empty body still performs its effects on the
pull that discovers exhaustion. Unit payload and result identities remain typed
through the graph, whether or not they need physical payload storage.

### 5.3 Exists and Forall

| Consumer | Decisive predicate result | Returned decision | Exhausted-input result |
|----------|--------------------------|-------------------|------------------------|
| `Seq.exists predicate input` | `true` | `true` | `false` |
| `Seq.forall predicate input` | `false` | `false` | `true` |

Each consumer invokes the predicate once for each successful pull it requires.
On the decisive result it returns immediately, with no further input pull or
predicate invocation. Otherwise it continues until exhaustion. The exhausted
result also applies to empty input, where no predicate is invoked. In particular,
`forall` must stop on false; its demand guard must retain that polarity.

These short-circuit laws preserve the complete input demand trace. Obtaining the
correct Boolean after unnecessary pulls or callbacks is not conforming behavior.

### 5.4 TryHead and TryPick

`Seq.tryHead input` pulls once. Exhaustion returns `None`; a successful pull
returns `Some current` and makes no further pull. An effectful empty input still
performs the effects needed to discover exhaustion.

`Seq.tryPick chooser input` invokes the chooser once per successful pull, in
input order. A `None` result continues the search. The first `Some value` is
returned unchanged, with no later pull or chooser invocation. If the input
exhausts, the result is `None`. The input is demanded for traversal and the chooser
only when a successful input requires a choice. Testing the returned result must
not reevaluate the chooser or force an unused `Some` payload.

The input type `'T` and result payload type `'U` are independent, including their
dimensions. These consumers return the native `option` algebra described in
[Option Operations](option-operations-representation.md). Payload admission and
placement follow the retained type and proof facts; the source contract does
not prescribe a runtime wrapper or a fixed allocation residence.

## 6. Callable Values and Captured Environments

Each producer retains the canonical `(moveNext, env)` sequence callable. Each
callback retains the canonical `(fn, env)` function callable, with the capture
semantics and lifetime requirements of [Closure Representation](closure-representation.md).
No function address is stored as data in either environment.

The compiler may elide a function half only where its exact implementation is
established for the use. That proof does not establish an environment instance.
Two callback formations with the same code can carry different immutable values,
mutable cells and allocation identities. An invocation must receive the actual
environment recalled at that use, including after an alias, capture or frame
read; it must not substitute the environment of another formation.

A function with no environment requires no invented empty address or null state.
A function whose callable half cannot be elided retains the full pair through
passing, return and control-flow joins under the closure contract. Persisting an
unknown callable across suspension requires its own admitted representation;
it is not resolved by storing a numeric address or relabeling a descriptor.

The flat environment requirement rules out an implicit linked chain of enclosing
activation environments. It does not require copying every captured sequence or
callback environment byte-for-byte into every producer. Captured typed views and
owned child regions must satisfy their explicit source, layout and lifetime
relations. A byte copy is sufficient only when representation compatibility,
reference validity, required sharing and code availability are established.

## 7. Placement, Residence and Composition

### 7.1 Logical roles and physical storage

Producer configurations, live input iterators, callback environments, remaining
demand and active inner iterators are logical storage requirements. They compose
the state/current/capture/live-value/child-region roles of
[Sequence Representation §4](seq-representation.md#4-memory-layout-specification).
They do not prescribe field ordinals or operation-specific native struct layouts.

Baker settles slot types, capture modes, ranges, offsets, extents and alignment
against the selected target. Values surviving a yield occupy admitted persistent
storage. Values needed only within one pull may use activation scratch. A retained
mutable cell can require persistent storage after its last scalar read. Current
needs physical storage only where an admitted successful yield can establish it.

Physical fields and regions follow the settled
requirements, not a capture-count formula or a source operation's name. Source
NTU types remain intact while target representation is selected.

### 7.2 Backing storage and independent instances

A view descriptor denotes storage; its own finite size does not establish the
backing allocation's lifetime or extent. A captured template or callback must
have backing storage covering every admitted use. Exact activation/region,
allocation, capture, iterator and use participants establish that relationship;
lexical nesting alone does not.

Caller-owned factory destinations and parent-owned child regions may provide
such storage where their premises hold. Distinct simultaneously retained
allocation occurrences require distinct storage or an explicit sharing proof.
Knowing a child layout owner does not identify which runtime instance holds a
particular iterator's state. A new enumeration must initialize fresh progress
from the retained template and captures, not copy a prior enumeration's progress.

Allocation follows the lifetime classes and target capabilities of
[Closure Representation §3.3](closure-representation.md#33-escape-analysis).
An escaping value does not automatically require a heap; a finite frame does not
automatically permit static residence. Undeclared allocation mechanisms or
unproved escaping storage remain compile-time residuals. A caller-owned result
frame does not extend the lifetime of a cell in a returning factory activation.
Sequence composition SHALL NOT introduce a garbage-collected heap allocation.

### 7.3 Composed demand

A filter/map/take pipeline combines the operation laws rather than imposing a
physical nesting formula. While remaining demand is positive, filtering may pull
and reject multiple source elements; mapping forms one shared deferred result
for each accepted input, and invokes the mapper only when that result is
demanded. Take counts the resulting outputs. Once take's limit is reached the entire
upstream demand chain stops.

Every stage retains its callback/configuration values and admitted environment
instances. Optimizing composition or sharing storage must preserve operand
formation, callback order, current identity, suspension ownership and observable
pulls, including effects on exhaustion. No fixed number of stages or capture
slots establishes a bound on total live program storage.

## 8. Graph and Witness Obligations

The common Baker ingredients and recipes establish ordered operand snapshots,
iterator creation, successful-pull guards, current snapshots, callback
applications and source results. Producer bodies use the same sequence owner,
generator and yield contracts as source sequence expressions; consumers establish
their ordinary loops and state updates through the graph.

The following premises remain distinct:

- **Type and application:** input, callback, state and result types satisfy the
  native scheme; logical argument boundaries and source ranges are retained.
- **Evaluation and current:** exact control occurrences preserve formation,
  short circuit and repeated loop evaluations. A current read requires a
  successful pull for that same iterator on its applicable path.
- **Range:** current-range evidence joins all applicable yielded payload sources
  under the current-read premise. Unknown alternatives cannot justify narrowing.
  Mutable accumulator ranges incorporate writes and call effects rather than
  reusing a prior observation's bounds.
- **Storage and lifetime:** definite assignment, live-across values, borrowed
  cells, callback environments and child instances have their required placed
  storage and complete-use residence premises.
- **Proof and provenance:** layout, discriminant, application and residence
  obligations retain their exact participants through recipe fan-out/fold-in and
  lowering. A layout proof or a recorded evidence edge does not discharge the
  other obligations.

Invalid source types retain their source diagnostics; target frame synthesis
must not manufacture additional settlement errors from that rejected premise.
Admitted source with unresolved control, representation or residence requirements
still receives the responsible settlement diagnostic before unsupported witnessing.

Alex pulls settled child regions through its positional Huet zipper interface
and composes standard operations. It does not choose a wrapper layout, recover a
callback's captures from its source body or rediscover source yields to construct
control. [Backend Lowering Architecture](backend-lowering-architecture.md) owns
standard target conversion. SSA identities and counts derive from witnessed
operations and settled representation meets, not fixed per-operation formulas.

Storage accounting includes actual persistent frames, activation scratch, owned
regions and the backing storage retained by views. Symbolic code elision does not
elide an environment's formation effects or lifetime requirements.

## 9. Normative Requirements

1. Operations SHALL preserve the native schemes in §2, including independent
   accumulator/element types and exact callback argument/result constraints.
2. Ordinary supplied expressions SHALL retain shared deferred identities at their
   actual application boundary. Formation SHALL NOT force their initializers;
   direct explicit eager operands SHALL follow the activated frontier and order
   in §3. Repeated demands SHALL reuse each established result rather than replay
   its initializer.
3. Producers SHALL defer their bodies and callbacks until demanded. Consumers
   SHALL execute their required pulls when their result or specified effect is
   demanded, following §§4–5.
4. Every enumeration SHALL have independent progress while retaining original
   mutable cells and immutable established values or shared deferred identities.
5. `take`, `exists` and `forall` SHALL stop upstream demand at their respective
   decisions. Empty input SHALL obey each operation's result and callback laws.
6. Current reads SHALL retain the successful-pull prerequisite for their exact
   iterator; empty or failed pulls SHALL NOT supply a default element.
7. Callback code elision SHALL require exact callable identity and SHALL preserve
   the actual environment instance. Unsupported full-pair or stored-callable
   requirements SHALL remain explicit residuals, not an alternative encoding.
8. Placement SHALL follow complete typed layout, lifetime and target premises.
   Mandatory whole-environment copying, fixed state widths, ordinal wrapper
   fields and fixed SSA costs SHALL NOT replace those premises.
9. Recipes SHALL preserve source/reference/proof participants through graph
   elaboration. Witnessing SHALL consume those facts without reconstructing
   missing operation semantics or storage contracts.
10. Optimizations SHALL preserve the complete demand/effect trace and sharing
    laws, not merely the final list of values or Boolean result.

## 10. Behavioral and Negative Conformance

Conforming implementations SHALL satisfy the following observations:

| Contract | Required observation |
|----------|----------------------|
| Captured mapping | Scaling `1,2,3` by a retained factor of three produces `3,6,9`; distinct formations preserve their own factors |
| Filter and take | Taking the first three doubled even values from `1..10` produces `4,8,12` and performs no demand beyond the third output |
| Count boundaries | Zero/negative take makes no pull; a shorter input completes without manufacturing elements |
| Independent enumeration | Two enumerations start with fresh progress while callbacks over the same external mutable binding retain that same cell |
| Fold | Summing `1..10` produces `55`; an empty input returns the initial state, including a state type different from the element type |
| Iter | Actions run once per required element in order; empty input runs no action |
| Exists/forall | Exists stops on true, forall on false; empty results are false/true respectively, with no predicate call |
| Collect | Each outer callback runs once; its inner sequence exhausts before the next outer pull, including effectful empty children |
| Formation and exhaustion | Ordinary operand effects remain deferred until demanded; direct eager operands follow their activated formation frontier; a reached no-yield body performs its exhaustion effects; stopped demand does not execute later input work |

Negative conformance includes wrong count kind/dimension, non-Boolean predicates,
non-unit actions, callback/element dimension mismatches and inconsistent fold
state/result types. The gate requires the responsible compiler diagnostic,
effective severity and source range, not any parse or compilation failure.
Missing current, callback-environment, lifetime, range or target-placement
premises are separate settlement failures. Unknown origins or factory-local
captured storage must not pass through an accidental native representation.

## 11. References

- [Sequence Representation](seq-representation.md): base callable, capture,
  suspension, storage and current-value requirements.
- [Closure Representation](closure-representation.md): flat environments,
  callable/environment identity, lifetime and transfer obligations.
- [Delimited Continuation Representation](dcont-representation.md): common
  owner/cut/resume evidence and proof boundaries.
- [Expressions](expressions.md): application, pipeline and evaluation rules.
- [Backend Lowering Architecture](backend-lowering-architecture.md): passive
  witnessing and standard target lowering.
