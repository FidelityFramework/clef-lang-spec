---
title: "Closure Representation"
weight: 310
category: Representation
status: normative
---

## 1. Overview

A closure is a function value that carries the variables it captured from the scope where it was defined. Clef represents every such value as a **flat closure**: a code pointer paired with a single flat environment holding the captures, placed in the storage whose lifetime covers it, stack, region, static, or heap, by the lifetime lattice of §3.3. It is *flat* in that the environment is one block, not a chain of enclosing environments to traverse. This chapter specifies that representation, its capture semantics, its calling convention, and the properties that follow from it.

The flat closure is the foundational representation on which much of the rest of the language rests. A [lazy value](lazy-representation.md) is a flat closure with fields added for memoization. A [sequence expression](seq-representation.md) is a flat closure that carries state-machine state. A [reactive callback](reactive-signals.md) and an [observer continuation](observable-computation.md) are flat closures whose capture sets are their dependency sets. A [discriminated union case](discriminated-union-representation.md) that holds a function holds a flat closure. Every function type in Clef ([types and type constraints](types-and-type-constraints.md)) compiles to one. Specifying the flat closure precisely, and once, is what lets those chapters extend it rather than restate it.

This chapter specifies the closure's layout (§2), capture and escape semantics (§3), field initialization (§4), representation properties (§5), calling convention (§6), and its relationship to the representations that extend it (§7). The design rationale is developed in [Null-Free by Construction](https://clef-lang.com/docs/design/language/null-free-by-construction/) and [Gaining Closure](https://clef-lang.com/docs/design/memory/gaining-closure/) in the Clef design documentation. This representation follows the MLKit-style flat closure; see Shao and Appel, *Space-Efficient Closure Representations* (LFP '94), and Tofte and Talpin, *Region-Based Memory Management* (Information and Computation, 1997).

## 2. Memory Layout Specification

### 2.1 Closure Structure

A closure is a struct containing a code pointer followed by captured values:

```
Closure with captures [c₁: T₁, ..., cₘ: Tₘ]
┌─────────────────────────────────────────────────────────┐
│ code_ptr: ptr                    (one platform word)    │
├─────────────────────────────────────────────────────────┤
│ c₁: T₁  (sizeof(T₁) bytes, aligned)                     │
├─────────────────────────────────────────────────────────┤
│ c₂: T₂  (sizeof(T₂) bytes, aligned)                     │
├─────────────────────────────────────────────────────────┤
│ ...                                                     │
├─────────────────────────────────────────────────────────┤
│ cₘ: Tₘ  (sizeof(Tₘ) bytes, aligned)                     │
└─────────────────────────────────────────────────────────┘

Field Indices:
  [0] = code_ptr (function address)
  [1..m] = captured values
```

This is the meaning of *flat*: a closure holds its captures in a single environment, so a capture is reached directly rather than by traversing a chain of enclosing environments. The alternative, the linked environment chain of Cardelli-style closures, is prohibited (§10); the reason is developed in §5. In the middle end this environment is encoded as a pointer alongside the code pointer, in portable dialects (§6.3); "flat" constrains the *shape* of the environment, a single block rather than a chain, not whether it is reached through a pointer.

### 2.2 Capture Semantics

Captures are classified by the mutability of the source binding:

| Variable Kind | Capture Mode | Entry Type | Semantics |
|---------------|--------------|------------|-----------|
| Immutable binding | By Value | `T` | Copy value into closure |
| Mutable binding | By Reference | `ptr<T>` | Store pointer to stack slot |
| Ref cell | By Value | `ref<T>` | Copy ref cell pointer |

An immutable binding is copied because nothing can change it; every holder of the closure observes the same value regardless. A mutable binding is captured by reference because all closures over it must observe the same changing storage; copying its value would break that contract. The mutability of each capture is tracked from type checking through emission ([access kinds](access-kinds.md) governs the underlying mutability classification).

### 2.3 Allocation Strategy

A closure's environment is placed in the storage whose lifetime covers it, chosen from four by its lifetime classification:

1. **[On the stack](memory-regions.md)**, when its lifetime is bounded by the enclosing scope.
2. **In a region**, when it escapes the enclosing scope but lives within a region's lifetime.
3. **In static storage**, the [`Sram`](memory-regions.md) region for a mutable environment or [`Flash`](memory-regions.md) for an immutable one, when its lifetime is the whole program (constructed once, held to program end, never freed).
4. **On the heap**, when its extent is genuinely dynamic.

The classification and placement are chosen at compile time by escape analysis (§3.3). A target without a heap admits only the stack and static placements; a closure that classifies as dynamic there is a lifetime error, not a silent heap allocation. Static placement uses the same program-lifetime storage that a fixed-address register (`Peripheral`) or a linker-carved buffer already occupies: a program-lifetime closure is a global, and it lives where the other globals live.

## 3. Capture, Escape, and the Type System

### 3.1 Function Type Representation

```
Type ::= ...
       | TFun(argTypes: Type list, retType: Type)
       | TClosure(argTypes: Type list, retType: Type, captures: CaptureInfo list)
```

At the native level, the Clef Compiler Service (CCS) distinguishes a direct function with no captures (`TFun`) from a closure with a captured environment (`TClosure`). The distinction is a representation choice, not a surface-syntax one: a lambda that captures nothing needs no environment and no closure value; it is a plain named function that callers reference directly. Only a lambda that captures builds a closure.

### 3.2 Capture Analysis

During type checking, CCS computes the capture set for each closure:

```fsharp
type CaptureInfo = {
    Name: string              // Variable name
    SourceNodeId: NodeId      // PSG node where defined
    Type: NativeType          // Type of captured variable
    IsMutable: bool           // Determines capture mode
}
```

The capture set determines the struct's layout: one field per captured variable, in a determined order, with the mode taken from `IsMutable`. Because the set is known at compile time, the layout is known at compile time (§5).

### 3.3 Escape Analysis

Capturing a mutable binding by reference creates a lifetime constraint: the captured storage must outlive every closure that references it. Escape analysis resolves this constraint at compile time by choosing where the closure's environment is allocated, so that the storage's lifetime covers the closure's.

**Invariant**: The environment holding a mutable capture SHALL have a lifetime that covers every closure that captures it by reference.

The analysis classifies each closure by *how long its environment must live*, and allocates in the storage whose lifetime covers it. Escaping the defining scope and requiring the heap are distinct questions: a closure can escape its scope and still have a statically known, program-long lifetime, in which case it belongs in static storage, not on a heap.

| Lifetime classification | Escapes scope? | Allocation |
|-------------------------|----------------|------------|
| Scope-bounded | No | Stack (`alloca`) |
| Region-bounded | Yes, within a region's lifetime | [Region](memory-regions.md) |
| **Program-lifetime** | Yes, for the whole program | Static storage (`.bss`-class, `memref.global`) |
| Dynamic | Yes, unknown extent | Heap |

A scope-bounded closure keeps its environment on the stack, reclaimed when the scope exits. A region-bounded closure has its environment in a region whose lifetime covers it, so a by-reference capture remains valid after the defining scope returns. A **program-lifetime** closure, one constructed once and held for the life of the program with no free (a capability record assembled at startup and held by the entry point is the canonical case), has a statically knowable lifetime equal to the program's, and its environment is placed in static storage rather than allocated: it is a global in the same sense a fixed-address register or a linker-carved ring buffer is a global. A dynamic closure of genuinely unknown extent uses the heap.

This four-point lattice matters because a target may have no heap. On a target with no allocator, the *only* lifetimes that have a home are scope-bounded (stack) and program-lifetime (static); a closure that classifies as dynamic on such a target is a lifetime error, not a silent heap allocation. Collapsing "escapes" to "heap" would make every long-lived closure, including a capability record, allocate on a heap that does not exist. Distinguishing program-lifetime from dynamic is what lets a returned-and-held closure live in `.bss` and satisfy a no-allocation discipline by construction. The escape classification is carried as a coeffect that the closure's witness reads when it emits the allocation; as a design-time property established here, it is subject to the [preservation obligation through lowering](conformance.md).

## 4. Initialization

Every field of a closure is assigned at construction. The code pointer is set to the closure's implementation function; each capture field is set to the captured value, or for a by-reference capture to the address of its storage.

No closure field admits a null value. This exclusion is stated explicitly because null-as-a-reference-state is a widespread convention that this representation deliberately does not adopt: a closure has no null code pointer and no null capture, and where a program models the possible absence of a value it uses `Option` ([option operations](option-operations-representation.md)) rather than a nullable field. The design rationale is developed in [Null-Free by Construction](https://clef-lang.com/docs/design/language/null-free-by-construction/).

## 5. Representation Properties

A closure has the following properties, which the representations in §7 rely on:

**Deterministic layout.** A closure's size and field offsets are fixed at compile time by its capture set, independent of run-time state.

**Static self-description.** A closure is interpretable from its static type alone, with no run-time header, descriptor, or tag.

**Position independence.** A closure's captures live in a single flat environment, so the closure is a self-contained value that can be copied between memory spaces as a block and remains valid at the destination.

These properties are preserved across backends: the closure is encoded in portable dialects and its pointers are materialized per target (§6.3, [Backend Lowering Architecture §4.2](backend-lowering-architecture.md)). The design treatment of substrate portability is developed in [Null-Free by Construction](https://clef-lang.com/docs/design/language/null-free-by-construction/).

## 6. Calling Convention

### 6.1 Closure Invocation

To invoke a closure, the caller extracts the code pointer from field `[0]`, then calls it with a pointer to the closure struct as the first argument, followed by the explicit arguments:

```
call_closure(closure, arg1, arg2):
    code_ptr = closure[0]
    call code_ptr(&closure, arg1, arg2)
```

### 6.2 Capture Access

The closure body receives a pointer to its own struct and reads captures by field index:

```
closure_body(self_ptr, arg1, arg2):
    cap1 = self_ptr[1]    // First capture
    cap2 = self_ptr[2]    // Second capture
    // use captures and arguments
```

Capture access is a load at a known offset.

### 6.3 Middle-End Encoding and Lowering

The layout in §2 is the conceptual representation. It is **not** encoded in a backend's pointer types in the middle end. The MiddleEnd (Alex) encodes a closure using only portable dialects (`func`, `memref`, `arith`), as a `(code_pointer, environment_pointer)` pair carried as `index` values, and defers the materialization of those pointers to a per-target pass. This is what keeps the representation portable across backends rather than committed to LLVM; the mechanism, including the `unrealized_conversion_cast` deferral and its per-backend resolution, is specified in [Backend Lowering Architecture §4.2](backend-lowering-architecture.md). On the LLVM backend the code pointer and captured-environment pointer materialize as `llvm.ptr` values through that resolution; on another backend they materialize into that backend's pointer mechanism from the same middle-end IR.

Because every field is assigned at construction (§4), the encoding carries a complete value into lowering on any backend.

### 6.4 JSIR-Pathway Realization

> This section binds implementations claiming the **JavaScript Substrate** profile ([Conformance §7](conformance.md)).

The JSIR pathway realizes the `(code_pointer, environment_pointer)` pair as a single host function value, under the carrier-realization rule of [Backend Lowering Architecture §4.5](backend-lowering-architecture.md): a JavaScript function carries its code and its captured environment together, so no separate environment pointer is materialized, and invocation is direct application of the function value.

The capture semantics of §2.2 SHALL be preserved observably. A mutable capture SHALL be realized so that every closure over the binding observes the same changing storage, which the host closure's captured binding provides directly; an immutable capture MAY be realized by sharing, since immutability makes a copy and a shared binding indistinguishable. The classification of §8 (escaping closures against nested named functions) applies unchanged: a nested named function that does not escape SHALL pass its captures as parameters on this pathway as on any other.

The layout requirements of this chapter (deterministic offsets, cache alignment, the §3.3 allocation lattice) are requirements on layout-realizing pathways and do not bind on this pathway: the host garbage collector owns placement, and the environment's shape is the host closure's captured scope. The structural requirements (full initialization, no null or undefined capture, capture modes, classification) bind unchanged.

## 7. The Representation Family

The flat closure is extended, not replaced, by several other representations. Each adds fields or structure to the base layout while keeping its properties.

| Representation | Extends the flat closure with | Specified in |
|----------------|-------------------------------|--------------|
| Lazy value | A computed flag and a memoized value slot | [Lazy Value Representation](lazy-representation.md) |
| Sequence expression | Resumable state-machine state | [Seq Representation](seq-representation.md) |
| Reactive callback | A capture set read as a dependency-edge set | [Reactive Signals](reactive-signals.md) |
| Observer continuation | The consumer state a continuation resumes into | [Observable Computation](observable-computation.md) |

Each of these is a flat closure first. A lazy value is null-free and position-independent for the same reason a plain closure is; a sequence's state machine is a settled struct for the same reason. The properties in §4 and §5 are inherited by the whole family because they are properties of the base representation, established once here.

An extending representation adds its fields to the flat environment as one or more **slot classes**, and each slot class carries a transition discipline: a finite monotone automaton whose transitions occur at statically identified program points. The environment's mutation obligations are the product of its slot classes' automata; because each automaton is finite and each transition point is statically identified, the obligations quantify over enumerated structure only and remain quantifier-free (§11).

| Slot class | Transition discipline | Specified in |
|------------|-----------------------|--------------|
| Memoization slots (`computed`, `value`) | Write-once at force | [Lazy Value Representation](lazy-representation.md) |
| State-machine slots (`state`, `current`, internal state) | Captures read-only in MoveNext; internal state read-modify-write between yields; `current` written at yield; `state` written at yield and at exhaustion | [Seq Representation](seq-representation.md) |
| Dependency references | Written only by the owning cell's emission | [Reactive Signals](reactive-signals.md), [Incremental Computation](incremental-computation.md) |
| DU tag | Set by constructors, read by eliminators | [Discriminated Union Representation](discriminated-union-representation.md) |

An extension chapter SHALL instantiate this schema for the slot classes it adds; it SHALL NOT restate the mutation rules the schema fixes.

## 8. Nested Named Functions vs Escaping Closures

Clef distinguishes two categories of functions that capture variables. Only one needs a closure struct.

### 8.1 Escaping Closures (Closure Struct Model)

[Anonymous lambdas and function values](expressions.md) that may escape their defining scope use the flat closure struct:

```fsharp
let makeAdder n =
    fun x -> x + n  // Anonymous lambda, may escape
```

The lambda is a first-class value that can be returned, stored, passed to a higher-order function, or carried as a discriminated-union payload; DU payload storage is an escape point for the closure ([Discriminated Union Representation §8.2](discriminated-union-representation.md)). CCS creates a closure struct `{ code_ptr, n }` for it.

### 8.2 Nested Named Functions (Parameter-Passing Model)

A named function defined inside another function that does **not** escape passes its captures as ordinary parameters instead of building a struct:

```fsharp
let sumTo n =
    let rec loop acc i =
        if i > n then acc     // 'n' captured from enclosing scope
        else loop (acc + i) (i + 1)
    loop 0 1
```

Here `loop` is called directly from `sumTo` and never escapes. Its capture becomes a leading parameter:

```
Generated signature:  loop: (n: int, acc: int, i: int) -> int
                              ↑ capture   ↑ explicit params
Call site:            loop(n, 0, 1)
```

Passing the capture as a parameter costs no struct and no allocation. This is the preferred representation whenever escape analysis proves the function cannot outlive the scope it captures from.

### 8.3 Classification

| Criterion | Escaping Closure | Nested Named Function |
|-----------|-----------------|----------------------|
| Definition form | `fun x -> ...` | `let [rec] name ...` |
| PSG parent | Application, Sequential, etc. | Binding node |
| Can escape scope | Yes | No |
| Representation | `{code_ptr, cap₁, ...}` struct | Direct function |
| Capture passing | Via struct extraction | As explicit parameters |
| Allocation | Stack, region, or static for the struct (§3.3) | None |

A Lambda is classified as a nested named function if and only if its enclosing function is present (it is nested) **and** its parent [PSG](program-semantic-graph.md) node is a `Binding` (it is named). Otherwise it is an escaping closure and uses the struct model.

## 9. Compilation Pipeline

The closure representation is produced across three phases, consistent with the coeffect model in which structure is computed before it is emitted.

**CCS** constructs the [PSG](program-semantic-graph.md) with complete lambda information: `SemanticKind.Lambda(parameters, body, captures)`, with captures computed during scope analysis and mutability tracked in `CaptureInfo.IsMutable`.

**Alex preprocessing** identifies lambda nodes that capture, tagging them with a `HasClosureCapture` coeffect, then computes each closure's concrete struct layout and assigns the SSA identifiers for its construction. Layout is settled here, before emission.

**Witnessing** observes the pre-computed coeffect. Where a closure coeffect is present, the witness emits the closure struct and the `(code_ptr, env_ptr)` pair. Where a lambda captures nothing, the witness emits no closure value at all: the lambda is a plain named function, and its callers reference it directly by name. A closure struct exists only when there is an environment to carry. The witness reads the layout; it does not compute it.

## 10. Normative Requirements

1. **Flat Representation**: A closure with captures SHALL use the flat closure representation: a code pointer and a single flat environment holding the captures, with no linked chain of enclosing environments.
2. **No Linked Environment Chain**: A closure's environment SHALL be a single flat block, not a chain of enclosing environments; a closure SHALL NOT traverse a linked chain of environment pointers to reach a capture.
3. **Full Initialization**: Every field of a closure SHALL be assigned at construction.
4. **No Null State**: A closure field SHALL NOT admit a null value; absence, where required, SHALL be modeled with `Option`.
5. **Deterministic Layout**: A closure's size and field offsets SHALL be determined at compile time from its capture set, independent of run-time state.
6. **No Runtime Metadata**: A closure SHALL be interpretable from its static type alone, without a run-time header, descriptor, or tag.
7. **Capture Mode**: A mutable binding SHALL be captured by reference; an immutable binding SHALL be captured by value.
8. **Lifetime-Driven Allocation**: A closure's environment SHALL be placed by escape analysis in the storage whose lifetime covers the closure, per the §3.3 lattice: the stack when scope-bounded, a region when region-bounded, static storage when its lifetime is the whole program, and the heap only when its extent is genuinely dynamic. The environment holding a by-reference capture SHALL be placed with a lifetime that covers every closure capturing it. On a target without a heap, a closure that classifies as dynamic SHALL be a compile-time lifetime error, not a heap allocation.
9. **Allocation Site**: A closure's environment SHALL be placed on the stack, in a region, in static storage, or on the heap, as determined by the §3.3 lifetime classification from escape analysis.
10. **Cache Alignment**: A small closure (≤64 bytes) SHOULD be aligned to a cache line.
11. **Nested Functions**: A named function defined within another function that does not escape SHALL pass its captures as parameters rather than building a closure struct.
12. **Classification**: A Lambda SHALL be classified as a nested named function if and only if its enclosing function is present AND its parent PSG node is a Binding.
13. **Pathway Scope**: Requirements 5, 8, 9, and 10 are requirements on layout-realizing pathways and bind pathways that realize memory layouts. On the JSIR pathway the closure SHALL be realized as a host function value per §6.4, with requirements 3, 4, 7, 11, and 12 binding unchanged.

## 11. Proof Extraction at Closure Sites

The flat representation is also a proof discipline. At every closure site the compiler extracts, without annotation, the judgments the representation makes finite. The capture set is enumerated (§3.2), so the closure's reachability frontier is exactly its field list. Its extent is a literal, fixed at compile time by the layout (§2.1, §5). Its release is a single site, chosen by the lifetime classification (§3.3). The memory-safety verification conditions a closure site generates therefore quantify over enumerated structure only: they are quantifier-free, and they discharge at the standing tiers of the decidable verification hierarchy ([terms and definitions](terms-and-definitions.md); [grade discipline](grade-discipline.md), where the escape classification discharges in the lattice family). An implementation SHALL derive these judgments from the representation itself; a closure site SHALL NOT require annotation to yield them.

The judgments survive lowering because the lowered form carries the same structure. The MLIR witnessed from the graph preserves the enumerated frontier: the zipper elides only what the graph has already saturated ([Program Semantic Graph](program-semantic-graph.md)), and every cast the closure representation requires ([Backend Lowering Architecture §4.2](backend-lowering-architecture.md)) corresponds to an obligation the graph records. Integrity is therefore substantiated through lowering, not re-derived after it; this is the [preservation obligation through lowering](conformance.md) in its closure-specific form. The proof shape is that of region soundness and safe-for-space closure conversion; see Tofte and Talpin, *Region-Based Memory Management* (Information and Computation, 1997), and Shao and Appel, *Space-Efficient Closure Representations* (LFP '94).

## 12. Related Chapters

- [Lazy Value Representation](lazy-representation.md), [Seq Representation](seq-representation.md) - representations that extend the flat closure
- [Reactive Signals](reactive-signals.md), [Observable Computation](observable-computation.md) - callbacks and continuations as flat closures
- [Discriminated Union Representation](discriminated-union-representation.md) - DU cases that hold closures
- [Memory Regions](memory-regions.md), [Access Kinds](access-kinds.md) - allocation and mutability that govern capture
- [Backend Lowering Architecture](backend-lowering-architecture.md) - how backends realize the representation
- [Native Type Universe](native-type-universe.md), [Types and Type Constraints](types-and-type-constraints.md) - where function types resolve to closures
