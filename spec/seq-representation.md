---
title: "Sequence Expression Representation"
weight: 340
category: Representation
status: normative
---

> **Normative specification for sequence values, storage and suspension in Clef.**
> Implementation coverage and final native acceptance are recorded separately in
> [Composer C-06](../../Composer/docs/PRDs/C-06-SimpleSeq.md) and its linked
> waypoints. A required representation contract is not a claim that every source
> composition or target path already implements it.

## 1. Overview

A Clef `seq { }` expression creates a deferred computation that produces elements
on demand. A successful pull yields one element and records where execution must
resume. Exhaustion returns false. The PSG retains source types, evaluation order,
ownership, storage and proof premises before the middle end witnesses operations.

This chapter specifies the sequence callable, persistent frame, current-value
admission and suspension contract. Sequential syntax is not an instruction for
Alex to flatten or split expressions. Baker composes the graph's evaluation
relations and constructs the resumed computation through its recipes.

## 2. Relationship to Closures and Lazy

Sequences use the callable/environment model of
[Closure Representation](closure-representation.md). Like
[Lazy Representation](lazy-representation.md), construction forms captures without
executing the deferred body. Unlike a lazy value, a sequence does not memoize one
result for every consumer: each enumeration has its own iteration state.

The canonical sequence value is the pair `(moveNext, env)`. The callable half is
separate from the environment; no function address is stored as an environment
data field. Where the exact callable is established for a use, compilation may
elide that half and call the known function with its environment. This
specialization does not replace the general pair contract. A consumer without
that premise must retain both halves through an admitted typed representation.

Storage follows the admitted lifetime classes in
[Closure Representation §3.3](closure-representation.md). Stack residence needs a
bounded lifetime; program-lifetime storage needs an admitted program-lifetime
site. Repeated dynamic construction is not automatically a single static object.
An unsupported lifetime or target allocation class is a compile-time residual,
not a reason to return storage that dies with the generator activation or to
silently introduce heap allocation on a target without it.

## 3. Empty and Zero-Cut Sequences

### 3.1 Seq.empty

`Seq.empty<'T> : seq<'T>` contains no elements and every pull returns false.
Its element type remains meaningful for type checking and composition, but no
physical current slot or default value of `'T` is required. A compiler may retain
a minimal exhausted frame and a trivial MoveNext, or elide storage/calls when
that transformation preserves the complete observable behavior.

There is no fixed SSA cost, field width or mandatory allocation for this
primitive. Eliding a callable value requires its identity to be established as
in §2. Reading current from the empty iterator is never admitted.

### 3.2 A body with no yields can still execute

A zero-cut generator is not necessarily the pure empty primitive:

```fsharp
let mutable visits = 0
let effects = seq {
    visits <- visits + 1
}
```

Creating `effects` captures its required storage without running the assignment.
Its first pull executes the assignment and then exhausts; later pulls do not
repeat the completed body. A new enumeration starts again and may update the
same captured external cell.

The generator may retain a logical current identity and element type without
allocating a current field. A certified consumer can omit an unattainable
successful-current branch, but must preserve the pull and its effects. No
uninitialized field is read, and no placeholder element is synthesized.

## 4. Memory Layout Specification

### 4.1 Logical frame and settled physical layout

The environment has the following logical roles:

| Role | Required storage behavior |
|------|---------------------------|
| State | Initial entry, reached cut's resumption, or completion |
| Current | Most recently yielded element; physical slot only when needed |
| Captures | Creation-time immutable values or retained original mutable-cell views |
| Live-across values | Values and storage needed after a suspension |
| Owned child regions | Separately placed backing storage whose lifetime is the owning enumeration |

These roles do not prescribe ordinal field indices, fixed offsets or a universal
`i32` state/current representation. Baker settles each field's source identity,
NTU type, representation, extent, offset and alignment against the target
contract. The source state has a finite discriminant range; placement selects a
carrier that represents it. A yielded integer retains its dimensional type and
range obligations even when its physical storage is narrower than a source
read's held representation. Explicit settled meets adapt those representations.

The total environment extent is a literal before witnessing. Field placement
must satisfy alignment, containment and non-overlap or an explicitly justified
sharing contract. Interference coloring may reduce storage after the necessary
liveness proof, but coloring is not mandatory and is not implied by the existence
of a settled frame. The environment contains no function address field.

### 4.2 Captures, internal values and activation scratch

Immutable captures copy their values when the sequence is formed. Mutable
captures retain the original cell identity: reads and writes from an enumeration
observe that cell, and separate enumerations do not clone it. Internal bindings
are initialized when evaluation reaches them in the generator, not when the
sequence is created or by default stores before the first pull.

Both immutable and mutable bindings may be live across a suspension. A borrowed
cell can need persistent storage even after its enclosing computation's last
scalar read. Those requirements are established by the graph's liveness and
residence contracts.

Values needed only during one MoveNext activation may use distinct scratch
storage, including explicit stores between generated control regions. Such
scratch does not belong to the persistent resume frame. An explicit zero scratch
extent is valid when no scratch slots exist; it admits no slot access. A local
value does not become persistent merely because its source syntax is inside a
sequence.

### 4.3 State and current

| State | Meaning |
|-------|---------|
| 0 | Initial entry, not yet started |
| Positive settled label | Resume after the corresponding reached yield |
| -1 | Completed |

These values describe the persisted resume discriminant. A generator may also
have a local dispatch position for several control occurrences within one pull;
that local position is not an additional suspension state.

Each yield evaluates its payload, stores current and the corresponding resume
state, then returns true. Completion records the completed state and returns
false. A current read requires the successful-pull premise for the exact iterator
and the applicable control path. Completion, construction and empty enumeration
do not establish that premise. The compiler must reject an unadmitted read.

### 4.4 Typed views and owned storage

A typed storage view can retain a descriptor containing physical address,
offset, extent and stride. This is an unboxed storage value. It is not a runtime
type object, tag, boxed scalar or dynamic type conversion. Descriptor fields and
sizes are governed by the target representation; a particular MLIR rank-one
carrier's fields do not define a universal Clef type-size formula. No optimization
of those fields is assumed as an admission premise.

A mutable-cell capture stores its original cell view. Reading/writing the captured
scalar goes through that view; it does not copy the scalar into a replacement
cell. Borrowing an inline scalar slot exposes the exact placed cell view.
A buffer value does not acquire mutable-cell semantics without an explicit
admitted access contract.

A factory may initialize caller-owned frame storage when a settled destination
and residence contract establish its lifetime. A child frame constructed during
MoveNext may reside in an explicitly placed region of its parent's persistent
frame. Each simultaneously retained child allocation occurrence requires its
own admitted extent and identity. A descriptor alone does not supply backing
storage or extend its lifetime.

A sequence may capture a surrounding sequence template when the source
allocation's activation covers every use of the capturing sequence and its
iterators. Lexical containment alone is insufficient. Current Baker evidence
retains the allocation, covering activation, captured declaration, capturing
generator and constructor identities in `SequenceTemplateBorrow`. Nested and
repeated uses can preserve that covering lifetime; returned, stored, opaque or
ambiguously owned uses require additional settlement. Capturing the template
preserves its mutable capture identities while each enumeration gets fresh
iteration state.

Current native support uses explicit per-occurrence destination/origin rows and
finite parent/child region coordinates. It does not establish arbitrary escaping
factory results, recursive region growth, mixed callable origins or generic
aggregate transport. These cases still require the canonical callable and
lifetime contracts; they are not redefined by that implementation scope.

## 5. MoveNext Calling Convention

### 5.1 Environment argument

MoveNext receives its typed environment and returns a Boolean:

```text
moveNext : memref<E x i8> -> i1
```

Here `E` is the settled literal extent, not runtime type information. The function
half identifies this operation; the environment descriptor identifies a specific
allocation instance. Calls with a statically known origin may use the direct
function symbol. Unknown origins may not be reconstructed from symbol spelling
or a type-only guessed extent.

### 5.2 Structured machine and standard lowering

Baker constructs the Boolean generator body from ordinary typed graph operations.
The persisted state selects a resume entry; local structured control executes
until a yield or completion. A `ContinuationDispatch` supplies its selector,
case labels and child regions. Alex pulls these settled children through its
positional Huet zipper witness interface and composes `scf.index_switch`.

The switch selector has MLIR `index` type. A settled integer state carrier is
adapted with the appropriate existing typed index operation; this requirement
does not change its stored width to `index`. Frame accesses use typed
`memref.view`, load and store operations at settled literal offsets. Ordinary
branches and loops retain their structured control operations.

Sequence witnessing uses standard `func`, `memref`, `arith`, `index` and `scf`.
The backend lowers structured control to `cf` and then its target form, with
memory, index, function and arithmetic conversion governed by that target's
pipeline. No private continuation dialect, raw address reconstruction or
imperative source-body emitter is required for this sequence machine.

## 6. PSG Ownership, Evaluation, Cuts and Evidence

The suspension recipe of [Delimited Continuation Representation §2](dcont-representation.md)
owns elaboration. It retains the following distinct contracts:

- **Ownership:** Each retained suspension site has exactly one sequence delimiter
  and that owner's generator. Nested sequences own their suspension sites;
  ordinary function, lambda and lazy bodies do not inherit an outer delimiter.
  Ownership alone establishes neither branch feasibility nor evaluation order.
- **Delegation:** Reaching `yield! input` evaluates input once, initializes its
  iterator and pulls it locally. Each successful pull reads current once and
  yields through the delegating owner. Resumption continues that iteration;
  exhaustion proceeds after the delegation. Empty delegation produces no outer
  yield. Original source identity/range and operand provenance remain when Baker
  replaces the source form with the explicit loop and yield.
- **Evaluation:** Bindings, operands, selected branches, loop backedges and
  deferred formation preserve source order. Formation does not execute a nested
  deferred body. A definition reference does not rerun its initializer; graph
  traversal deduplication does not suppress a loop's required evaluations.
  Local demand/entry/completion relations precede their composition into control.
- **Control and cuts:** Composed control gives exact occurrences, successors,
  uses and definitions. A yield cuts one path and resumes at its settled
  successor. Untaken conditional paths do not create a yield. The persisted
  state labels come from this plan, not a source-order scan by Alex.
- **Availability and storage:** Definite assignment and live-across equations
  establish which values must survive each cut. Placement retains their exact
  types, ranges and storage requirements. Values used only within a pull can
  occupy activation scratch; borrowed storage retains its own lifetime premise.
- **Machine and evidence:** Recipe fan-out/fold-in creates typed state/frame
  accesses, dispatch and Boolean completion, retaining reference participants
  and provenance. Resident relations connect owner/generator, cut, payload,
  resume successor, live values, generated bodies and slots. Discriminant and
  layout obligations retain their premises and discharge boundary.

A valid layout obligation is not proof of every lifetime or continuation law.
Likewise, a recorded liveness relation is not a substitute for checking its
control participants and equations. Unsettled prerequisites remain diagnostic
at their owning layer. Alex consumes settled facts and representation meets; it
does not infer missing semantic control, choose residence or discharge proofs.

## 7. SSA and Storage Accounting

No fixed `5 + captures + 2 * locals` formula is specified. Required operations
and SSA results depend on settled field representations, adaptations, branch
results, scratch requirements, capture descriptors, destinations and callable
elision. Internal values are initialized only when source evaluation reaches
their definitions. Current is written only by a successful yield.

Target resource analysis must account for the actual admitted persistent frame,
activation scratch and owned child regions. A descriptor cost does not stand in
for the backing storage it denotes. Alex's deterministic SSA identities name
witnessed operations; they do not constitute the frame or resource proof.

## 8. Normative Requirements

1. A sequence SHALL retain its NTU element constraints through source checking,
   graph construction, proof obligations and physical representation settlement.
2. The canonical callable SHALL be `(moveNext, env)`, with no function address
   data field in the environment. Elision SHALL require an established callable
   identity for the use.
3. Each enumeration SHALL have independent iteration state while preserving
   original mutable capture identity and immutable capture values.
4. Baker SHALL establish suspension ownership, evaluation order, composed control,
   value availability and live-across storage before Alex witnesses a machine.
5. Every field and allocation SHALL have its required settled representation,
   placement and residence. Source types SHALL NOT be replaced by a blanket fixed
   integer width or a guessed MLIR descriptor layout.
6. A yield SHALL establish current and resume state before returning true. Current
   reads SHALL require the exact successful-pull premise. Empty and zero-cut
   sequences SHALL NOT fabricate an element or require a default of its type.
7. Delegation SHALL preserve input evaluation, element order, exhaustion,
   resumption and the separate ownership of the supplied sequence's sites.
8. Graph elaboration SHALL preserve source/reference participants and provenance;
   obligation presence SHALL NOT be represented as proof of unrelated properties.
9. Alex SHALL pull settled child regions at their Huet positions and compose
   standard operations. It SHALL NOT rediscover source yields, build semantic
   control in an emitter or guess missing frame/origin facts.
10. Allocation elision and caller/parent-owned storage SHALL preserve runtime
    allocation identity and admitted lifetime. Unsupported contracts SHALL remain
    explicit compile-time residuals.

## 9. Behavioral Test Contracts

These examples state required behavior, not completed test results.

### 9.1 Pre-yield and post-yield effects

```fsharp
let triangularNumbers count = seq {
    let mutable sum = 0
    let mutable i = 1
    while i <= count do
        sum <- sum + i
        yield sum
        i <- i + 1
}
```

The pull yielding a value updates `sum` first. Resuming increments `i` before
rechecking the guard. State and live values survive the cut; scratch evaluation
must not repeat or skip either effect.

### 9.2 Local value after a yield

```fsharp
let fibonacci count = seq {
    let mutable a = 0
    let mutable b = 1
    let mutable i = 0
    while i < count do
        yield a
        let temp = a + b
        a <- b
        b <- temp
        i <- i + 1
}
```

`temp` is computed at its actual resumed definition. It does not need persistent
storage if its lifetime ends before the next cut, but it must remain available
across any generated local control regions that use it.

### 9.3 Independent iteration and shared captures

Two enumerations start at the initial resume state and must not copy each other's
current or internal progress. A mutable cell captured outside both remains the
same cell. A no-yield effectful body still executes once per enumeration; a
nested empty delegation continues the outer body within the current pull.

## 10. Related Chapters

- [Closure Representation](closure-representation.md): callable values, capture
  identity, target layout and residence.
- [Lazy Representation](lazy-representation.md): deferred formation and memoization.
- [Delimited Continuation Representation](dcont-representation.md): shared
  suspension recipe and proof obligations.
- [Seq Operations Representation](seq-operations-representation.md): operations
  that compose the same sequence contracts.
- [Composer C-06](../../Composer/docs/PRDs/C-06-SimpleSeq.md): implementation scope
  and final source, graph, MLIR, native and tooling gates.
