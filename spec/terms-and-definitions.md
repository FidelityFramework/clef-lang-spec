---
title: "Terms and Definitions"
weight: 5
category: Process
status: normative
---

> **The defining vocabulary on which the normative requirements of this specification rest.** Each term is defined here and specified in full in the chapter cited.

For the purposes of this specification, the following terms and definitions apply. A term in **bold** at its point of definition carries the meaning given here throughout the document; where a later chapter is cited, that chapter is the full normative specification of the term and this entry is its capsule definition.

This clause is normative: a requirement stated elsewhere in terms of these words means exactly what is defined here. Where a term has a broader meaning in general usage, only the meaning given here applies within this specification.

## Process and conformance

**conforming implementation** — an implementation that satisfies all requirements of this specification applicable to it. Defined in [Conformance §3](conformance.md).

**conforming program** — a Clef program that violates no diagnosable rule of the language. Defined in [Conformance §4](conformance.md).

**normative** — imposing a requirement. **informative** — explanatory, imposing no requirement. The distinction is defined in [Conformance §2](conformance.md) and governs every clause and note.

**implementation-defined behavior**, **unspecified behavior**, **undefined behavior** — the three classes of behavior not fully fixed by this specification, defined and distinguished in [Behavior Classification](behavior-classification.md).

**[Design decision]**, **[Not yet specified]** — the maturity markers a requirement may carry, defined in [Change Process and Specification Management §2](change-process-management.md).

## Compiler architecture

**CCS (Clef Compiler Services)** — the front end that elaborates Clef source, builds and saturates the program graph, and discharges design-time obligations. Introduced in [Introduction](introduction.md).

**Composer** — the compiler that lowers the saturated graph through its middle end to target backends. Distinct from CCS (the front end) and from any one backend.

**Alex** — the Composer middle-end component that witnesses the saturated graph and lowers it to MLIR; the "Library of Alexandria" that holds the platform-resolving lowerings. Introduced in [Backend Lowering Architecture](backend-lowering-architecture.md).

**Baker** — the saturation engine that settles the program graph into its saturated form, expanding constructs that have no single machine instruction.

**nanopass** — an elaboration or lowering pass of small, single-purpose scope; the front end and middle end are composed of nanopasses.

**witness** — a component that consumes an enriched graph node and emits the lowered form (e.g. an `IntrinsicWitness`); witnesses pattern-match on graph structure, not on names.

## Program structure carriage

**Program Semantic Graph (PSG)** — the graph carrying a program's structure and its design-time facts (dimensions, ranges, grades, coeffects, escape classes) as annotations, preserved through lowering. Defined in [Program Semantic Graph](program-semantic-graph.md).

**Program Hypergraph (PHG)** — the PSG extended with hyperedges that bind multiple nodes at once, for constraints that are genuinely multi-way (e.g. co-location on a hardware tile, the join of several geometric elements) and that a set of pairwise edges would assert strictly more weakly.

**coeffect** — context a computation requires, carried as codata on the graph beside the value (as distinct from an effect a computation produces). Representation selection, memory residency, and the quire allocation are carried as coeffects.

**codata** — data attached to a graph node and preserved through passes that do not touch it; the carriage mechanism for coeffects and inference results.

## Types and dimensions

**dimensional type** — a type carrying a dimension drawn from a finitely generated free abelian group of base dimensions with integer exponents (the proposed fractional-type extension would admit rational exponents). Defined across [Units of Measure](units-of-measure.md) and [NTU Dimensional Architecture](ntu-dimensional-architecture.md).

**dimension** versus **range** — a value's *dimension* establishes its kind (e.g. *meters*); its *range* `[a, b]` establishes the concrete interval it occupies. Numeric selection takes the **range**, not the dimension, as its input. See [Numeric Selection §1](numeric-selection.md).

**grade** — the algebraic grade (scalar, vector, bivector, …) of a value in the geometric/Clifford algebra; a structural property the type system preserves.

**negative type**, **fractional type** *(proposed; non-normative)* — the additive-inverse and multiplicative-inverse (reciprocal) types that would carry, respectively, a reversal channel and a deferred-supply obligation, reaching the requirements into rational and real arithmetic. This is a proposed extension: the type universe as specified operates without reference to negative or fractional types, and the discipline is the subject of a companion treatment, not a normative chapter of this specification. No normative requirement in this specification depends on it.

**braid type** *(proposed; non-normative)* — a type that would decorate a tensor arrangement's crossing order, the non-abelian exchange of concurrent strands recorded as an element of the braid group. Its abelian projection, the writhe, is a single-generator quantity of the same kind the dimensional discipline already carries; the non-abelian remainder is the order-sensitive content the writhe discards. This is a proposed extension: the type universe as specified operates without reference to the braid word, and the discipline is the subject of a companion treatment, not a normative chapter of this specification. Only the writhe is available to normative text; no normative requirement in this specification depends on the non-abelian braid word.

## Numeric representation

**representation** — the machine form chosen for a real value: a posit, an IEEE-754 float, or a fixed-point number. Selected from the value's range, per target. Defined in [Numeric Selection](numeric-selection.md).

**numeric selection** — the compile-time function choosing a real value's representation from its analyzed dimensional range. The real-valued sibling of width inference.

**width inference** — the compile-time function sizing an integer from its value range. The integer sibling of numeric selection. Defined in [Width Inference](width-inference.md).

**regime** — a classification of a value's range into a representation family (e.g. near-unity-taper, wide-dynamic), the categorical output of selection prior to a concrete representation.

**posit**, **b-posit** — a tapered-precision real representation (Gustafson); b-posit is the [bounded-regime variant](https://arxiv.org/abs/2603.01615). Precision is maximal near magnitude 1.0 and tapers toward the extremes. (Posits do **not** carry extra precision near zero; the value of the quire is exactness of accumulation, not near-zero precision.)

**quire** — a wide fixed-point accumulator that holds a sum of products without per-step rounding, rounding once at final conversion; provides exact accumulation. Defined in [Numeric Selection §10.2](numeric-selection.md).

**coverage** — the requirement that a representation's dynamic range contain a value's range. A non-covering representation SHALL NOT be selected, and an empty coverage set is a hard error. See [Numeric Selection §2.1](numeric-selection.md).

## Memory and concurrency

**escape class** — the classification (e.g. stack-scoped, closure-capture, return-escape, by-ref-escape) of where an allocation's lifetime reaches, determining its placement. Defined in [Memory Regions](memory-regions.md) and [Access Kinds](access-kinds.md).

**arena** — a region whose entire contents are reclaimed as a unit when its owner (e.g. an actor) terminates.

**actor** — a unit of concurrency with private state, a single logical thread, and no shared mutable state; communication is by message only. The Clef actor execution is realized by **Olivier** (the actor runtime) under **Prospero** (the supervisor).

**wait-for edge** — an edge from a caller to a callee that a synchronous reply suspends on; the relation whose acyclicity is deadlock-freedom. Defined in [Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md).

## Verification

**tier** — a level of the decidable verification ladder. Tier 1 is the parametric "free" facts (dimensional and grade soundness); Tier 2 is quantifier-free linear obligations (integer `QF_LIA`, real `QF_LRA`) and bit-vector obligations (`QF_BV`); Tier 3 covers termination/probabilistic-bounded obligations; a relational/probabilistic concern beyond the decidable tiers is the separate top tier. Introduced across the verification chapters.

**seal** — an explicit developer commitment of a concrete representation at a site, turning the selection objective from a chooser into a coverage checker. Defined in [Numeric Selection §5](numeric-selection.md).
