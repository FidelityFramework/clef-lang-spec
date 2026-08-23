---
title: "Program Hypergraph"
weight: 305
category: Representation
status: normative
---

> **Status (August 2026)**: Design-stage. Normative for the structure it defines; the PHG saturation engine is under active development.
>
> **Informative reference**: The design exposition, the domain analyses (geometric algebra, spatial dataflow, physics-aware computation), and the motivating measurements are developed in *The Program Hypergraph* (Haynes 2026, working draft, `arxiv-papers/program-hypergraph-paper.md`). This chapter carries the normative distillation of that design; the paper's application sections impose no requirement.

The Program Hypergraph (PHG) is the [Program Semantic Graph](program-semantic-graph.md) with its binary edges generalized to directed hyperedges of arbitrary source arity. The generalization exists for constraints that are genuinely multi-way: a co-location requirement over a set of operations mapped to spatial hardware, the join of several geometric elements, a boundary-sharing relation among mesh simplices. A set of pairwise edges asserts strictly less than the joint constraint: pairwise co-location of every pair does not entail joint co-location of the set, and a decomposed multi-way join introduces intermediate nodes with no semantic identity and no well-typed grade annotation.

## 1. Structure

A directed hyperedge is a triple $f = (S_f, t_f, \lambda_f)$: a source set $S_f \subseteq V$, a single target node $t_f \in V$, and a hyperedge annotation $\lambda_f$. The PHG is the tuple $(V, F, \alpha, \beta)$ where $V$ is the node set with its vertex annotation function $\alpha$ (type, dimension, coeffect, and lifetime annotations, unchanged from the PSG), $F$ is the hyperedge set, and $\beta$ assigns each hyperedge its relational annotation.

1. A binary PSG edge SHALL be represented as the degenerate hyperedge with $|S_f| = 1$.
2. Every valid PSG SHALL be a valid PHG under that embedding, and every PSG discipline of this specification (saturation, coeffect carriage, node activation, enrichment) SHALL apply without modification when restricted to binary hyperedges.
3. A hyperedge's source set SHALL be enumerated at elaboration: membership is settled before any constraint over the hyperedge is discharged, and no hyperedge has a source set of unresolved extent.

Requirement 3 is what keeps the PHG inside the decidable discipline: an obligation over a hyperedge quantifies over enumerated structure only, so it remains quantifier-free and discharges in the standing solver families ([Grade Discipline §4.1](grade-discipline.md)).

## 2. Hyperedge Annotations

The annotation $\lambda_f$ SHALL carry a relational kind and a target-reachability bitvector, and MAY carry kind-specific fields:

| Field | Requirement |
|---|---|
| Relational kind | Required. Identifies the multi-way relation: geometric product, join, meet, co-location, transfer, synchronization barrier, or a declared relational operator |
| Reachability bitvector | Required. One bit per configured target, indicating on which targets the hyperedge is active |
| Grade fields | For geometric kinds: the grades or blade supports of all sources and the derived annotation of the target, per the composition rules of [Grade Discipline §3.4](grade-discipline.md) |
| Co-location fields | For spatial kinds: the co-location requirement, route topology, and synchronization structure over $S_f$ |

A kind-specific field SHALL be interpreted only by the analyses and pathways that declare that kind; an unrecognized kind on a target where the reachability bit is clear imposes no obligation.

## 3. Saturation

Saturation runs the graph's inference rules to a fixpoint, monotonically enriching annotations until no rule adds information, over the lattice $\mathrm{Fresh} < \mathrm{Elaborated} < \mathrm{Saturated}$.

1. The inference rule for a hyperedge SHALL fire only when every node in $S_f$ is elaborated. A partial firing over a subset of sources SHALL NOT occur: the target's annotation is a joint function of all sources, and a partial observation is not a sound approximation of it.
2. Hyperedge saturation SHALL be monotone over the saturation lattice, and SHALL terminate; the fixpoint is bounded by $O(|V| + |F|)$ iterations.
3. A hyperedge whose constraint cannot be discharged (an unsatisfiable joint constraint, a co-location requirement exceeding a target's resources on a target where the hyperedge is active) SHALL be diagnosed under the diagnostic obligation of [Conformance §5](conformance.md), never silently dropped or weakened to its pairwise projection.

## 4. Constraint Discharge

Hyperedge constraints SHALL be discharged in the same solver families as the node-level annotation disciplines, and a hyperedge SHALL NOT mix families in a single query: a constraint whose fields span families decomposes into per-family projections before discharge, preserving the separate-discharge requirement of [Grade Discipline §4.1](grade-discipline.md). Grade and blade-support fields discharge in their group and lattice families; co-location and reachability fields discharge in the lattice family; an acyclicity obligation over a wait-for tuple discharges as a rank constraint in the group family ([Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md)).

## 5. Emission Transport

A hyperedge is an analysis-time structure. It participates in saturation and constraint discharge; it is not a query target for code generation.

1. A hyperedge's consequence SHALL reach emission in exactly one of two forms: as saturated annotations on the nodes it governs, read as codata during lowering ([Backend Lowering Architecture §4.5](backend-lowering-architecture.md)), or as a reified operation or attribute set that carries the annotation into the emitted IR itself.
2. The emission traversal SHALL NOT query the hyperedge set. The zipper elides only what the graph has already saturated ([Closure Representation §11](closure-representation.md)); its context is positional, and every semantic fact it consumes is node-local codata or reified structure.
3. A backend pass that consumes a multi-way constraint (a partitioner consuming co-location constraints, a tile-assignment pass) SHALL consume the reified annotation, not the graph hyperedge.
4. The stage separation of [Grade Discipline §3.3.1](grade-discipline.md) applies to every hyperedge annotation: present on the PHG, read during lowering, absent from emitted MLIR except where reified deliberately under item 1.

> **Clef Note**: The two forms of item 1 are one rule with two carriers. Node-local codata serves an annotation whose consequence is local (a settled support, an escape class). Reification serves an annotation whose consequence must survive into the IR as joint structure: a $k$-ary geometric operation in a dialect that carries grade as an operation attribute, or a co-location annotation on the operations a partitioner groups. In both carriers the traversal that emits code remains local, and the hypergraph remains behind the emission boundary.

## 6. Domain Instances

> *Informative.* The rows below locate the hyperedge kinds this specification and the design corpus currently use. Each is normative only in the chapter that owns it.

| Hyperedge kind | Discharge | Owning treatment |
|---|---|---|
| $k$-ary join, meet, geometric product | Lattice family (QF_BV) over blade masks | [Grade Discipline §3.4](grade-discipline.md) |
| Co-location, route, synchronization | Lattice family over resources and reachability | Target pathway lowerings ([Backend Lowering Architecture](backend-lowering-architecture.md)); partitioning is pathway work |
| Mesh boundary sharing | Saturation equality over shared boundary nodes | Design-stage; the PHG paper §5.3 |
| Wait-for tuples, dependency width | Rank constraints (QF_LIA) | [Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md); analyzer is design-stage |
| Accumulation exactness (quire targeting) | Derived from blade masks at design time | [Grade Discipline §5.3](grade-discipline.md), [Numeric Selection §10.2](numeric-selection.md) |

## 7. Normative Requirements

1. **Embedding**: Every valid PSG SHALL be a valid PHG under the $|S_f| = 1$ embedding, with all PSG disciplines applying unchanged to binary hyperedges.
2. **Enumerated sources**: A hyperedge's source set SHALL be enumerated at elaboration; obligations over hyperedges SHALL quantify over enumerated structure only.
3. **Joint firing**: A hyperedge's inference rule SHALL fire only when every source node is elaborated; partial firing SHALL NOT occur.
4. **Termination**: Hyperedge saturation SHALL be monotone and SHALL terminate.
5. **No pairwise weakening**: A multi-way constraint SHALL NOT be discharged, approximated, or diagnosed as the conjunction of its pairwise projections.
6. **Per-family discharge**: A hyperedge constraint spanning solver families SHALL decompose into per-family projections before discharge; no combined query SHALL be issued.
7. **Emission transport**: A hyperedge's consequence SHALL reach emission only as saturated node-local codata or as a reified operation or attribute set; the emission traversal SHALL NOT query the hyperedge set; a backend pass SHALL consume reified annotations only.
8. **Diagnosis**: An undischargeable hyperedge constraint SHALL be diagnosed at design time under [Conformance §5](conformance.md).

## 8. Related Chapters

- [Program Semantic Graph](program-semantic-graph.md), the binary structure this chapter generalizes
- [Grade Discipline](grade-discipline.md), the annotation disciplines whose multi-way composition rules ride hyperedges
- [Backend Lowering Architecture](backend-lowering-architecture.md), carrier realization and the target pathways
- [Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md), the wait-for relation whose tuples are hyperedge instances
- [Conformance](conformance.md), the diagnostic obligation hyperedge discharge inherits

## References

- Haynes, H. (2026). *The Program Hypergraph: Multi-Way Relational Structure for Geometric Algebra, Spatial Compute, and Physics-Aware Compilation.* Working draft.
- Haynes, H. (2026). *Dimensional Type Systems and Deterministic Memory Management.* (DTS+DMM.)
- Karypis, G., Kumar, V. (2000). Multilevel k-way hypergraph partitioning. *VLSI Design 11(3)*.
