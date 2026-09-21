---
title: "Program Hypergraph"
weight: 305
category: Representation
status: normative
---

> **Status (September 2026)**: Normative for the structure and checking boundaries it defines. Graph-resident joint relations have implemented instances; the general saturation machinery and additional domain instances remain under development. This chapter does not establish their implementation acceptance.
>
> **Informative reference**: The design exposition, the domain analyses (geometric algebra, spatial dataflow, physics-aware computation), and the motivating measurements are developed in *The Program Hypergraph* (Haynes 2026, working draft, `arxiv-papers/program-hypergraph-paper.md`). This chapter carries the normative distillation of that design; the paper's application sections impose no requirement.

The Program Hypergraph (PHG) is the [Program Semantic Graph](program-semantic-graph.md) with explicit directed relations of arbitrary source arity. A relation retains the participants of a joint requirement: buffers sharing a capacity budget, the ordered operands of a geometric operation, or mesh elements sharing a boundary. A binary graph can encode the same relation using a relation node and role-labelled links. Ordered hyperedges make that identity direct and available to the analyses that need it. Pairwise equality of locations already establishes a common location; checking whether all buffers fit there additionally requires their joint resource constraint.

The program's semantic nodes and structural/reference relationships remain its computational spine. A local fact may remain a node coeffect. When its justification depends on several participants, the PHG retains that incidence, the premises, and the applicable rule as a joint relation. Elaboration can construct operational nodes as well as analysis relations. These are different roles within one graph; an obligation edge is not an executable instruction. The lattice used by an analysis orders its information, not the graph's topology.

## 1. Structure

A directed hyperedge is a triple $f = (I_f, t_f, \lambda_f)$: an ordered sequence of source occurrences $I_f = (v_1,\ldots,v_k)$, a single target node $t_f \in V$, and a hyperedge annotation $\lambda_f$. The same node may occupy more than one input position. The PHG is the tuple $(V, F, \alpha, \beta)$ where $V$ is the node set with its vertex annotation function $\alpha$ (type, dimension, coeffect, and lifetime annotations), $F$ is the hyperedge set, and $\beta$ assigns each hyperedge its relational annotation. Write $S_f$ for the set of nodes occurring in $I_f$ when only dependency membership is needed.

1. A binary PSG edge SHALL be represented as the degenerate hyperedge with $|I_f| = 1$.
2. Every valid PSG SHALL be a valid PHG under that embedding, and every PSG discipline of this specification (saturation, coeffect carriage, node activation, enrichment) SHALL apply without modification when restricted to binary hyperedges.
3. A hyperedge's source occurrences and their roles SHALL be enumerated at elaboration before a constraint over that relation is discharged. This establishes a finite static incidence, not a bound on the number of runtime instances or on data reachable through a participant.
4. A relation SHALL preserve source order and multiplicity unless its declared interpretation justifies a permutation or identifies repeated occurrences. Dependency scheduling MAY use $S_f$ without replacing the operational input sequence $I_f$.

Finite incidence permits obligations about those participants to be generated explicitly. It does not by itself make their propositions quantifier-free, decidable, or cheap to check. Each admitted rule SHALL specify the meaning of its obligations and the inference or verification procedure that establishes them. The procedures and their composition remain subject to the owning domain contracts ([Grade Discipline §4.1](grade-discipline.md)).

## 2. Hyperedge Annotations

The annotation $\lambda_f$ SHALL carry a relational kind and a target-reachability bitvector, and MAY carry kind-specific fields:

| Field | Requirement |
|---|---|
| Relational kind | Required. Identifies the multi-way relation: geometric product, join, meet, co-location, transfer, synchronization barrier, or a declared relational operator |
| Participant roles and evidence | Identifies source positions, source provenance, required premises, and the obligation's current discharge status; a recorded relation alone is not a discharged proof |
| Reachability bitvector | Required. One bit per configured target, indicating on which targets the hyperedge is active |
| Grade fields | For geometric kinds: the grades or blade supports of all sources and the derived annotation of the target, per the composition rules of [Grade Discipline §3.4](grade-discipline.md) |
| Co-location fields | For spatial kinds: the co-location requirement, route topology, and synchronization structure over $S_f$ |

A kind-specific field SHALL be interpreted only by the analyses and pathways that declare that kind; an unrecognized kind on a target where the reachability bit is clear imposes no obligation.

## 3. Saturation

Saturation applies admitted elaboration and analysis rules until the facts required by a demanded compilation boundary are established. Each analysis has its own information domain and transfer rules. Lifecycle labels such as Fresh, Elaborated, and Saturated summarize progress; they do not supply the domain's convergence proof or a complexity bound.

1. An inference rule SHALL identify the premises required for each conclusion. It MAY derive a sound partial result while other facts remain pending, but SHALL NOT assume an unavailable premise or omit a participant required by that conclusion. Changed premises SHALL cause dependent conclusions to be reconsidered or invalidated.
2. Recursive analysis components SHALL use a worklist or an equivalent procedure with a sound convergence policy for their domains. Requiring every participant to be globally saturated before any rule can run SHALL NOT substitute for handling an inference cycle.
3. For a fixed finite collection of finite-height annotation domains, monotone inflationary rules with fair scheduling reach a fixed point after finitely many successful updates. An analysis with an infinite ascending chain SHALL provide a separate termination argument or sound acceleration policy, such as widening followed by an admitted narrowing procedure. Structural elaboration that generates further nodes or rules also requires its own termination argument. No $O(|V|+|F|)$ iteration bound follows from lifecycle labels alone.
4. A known contradiction SHALL be diagnosed under [Conformance §5](conformance.md). An unsupported, cancelled, timed-out, or otherwise unresolved required obligation SHALL remain pending during partial analysis and SHALL prevent the commitment that relies on it. Quiescence of an incomplete graph is not witness readiness.
5. A multi-way constraint SHALL NOT be replaced by weaker pairwise projections. A decomposition is permitted when its declared rule establishes the required preservation or sound implication and retains the original relation's provenance.

Constraint propagation may carry requirements from uses toward definitions as well as facts toward uses. This analysis-time direction does not reverse runtime effects or solve an arbitrary recursively defined value. Any admitted runtime cycle, delayed computation, or dynamic matching operation requires its own operational contract and realization.

## 4. Constraint Discharge

Hyperedge constraints SHALL use the admitted procedures of the node-level annotation disciplines. Under the separate-discharge contract of [Grade Discipline §4.1](grade-discipline.md), a relation spanning families SHALL produce per-family obligation projections rather than a combined query. The projections SHALL retain their common participants, premises, and dependencies. Separate variable names or separate solver calls do not establish semantic independence: any fact exchanged between domains SHALL have a sound transfer rule, and a changed premise SHALL invalidate its dependents.

The owning obligation determines its theory. Dimensional equations, blade-support calculations, resource inequalities, and synchronization predicates need not use the same procedure. A finite wait-for acyclicity check can use integer rank constraints ([Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md)); that certificate establishes program progress only under the wait-completeness and execution assumptions of its contract. A solver result SHALL justify the required proposition under its stated premises, not merely demonstrate a compatible assignment.

## 5. Emission Transport

A hyperedge is an analysis-time structure. It participates in saturation and constraint discharge; it is not a query target for code generation.

1. A hyperedge's consequence SHALL reach emission in exactly one of two forms: as saturated annotations on the nodes it governs, read as codata during lowering ([Backend Lowering Architecture §4.5](backend-lowering-architecture.md)), or as a reified annotation, an attribute set on emitted operations that a later pass consumes.
2. The emission traversal SHALL NOT query the hyperedge set. The zipper elides only what the graph has already saturated ([Closure Representation §11](closure-representation.md)); its context is positional, and every semantic fact it consumes is node-local codata or reified structure.
3. A backend pass that consumes a multi-way constraint (a partitioner consuming co-location constraints, a tile-assignment pass) SHALL consume the reified annotation, not the graph hyperedge.
4. The stage separation of [Grade Discipline §3.3.1](grade-discipline.md) applies to every hyperedge annotation: present on the PHG, read during lowering, absent from emitted MLIR except where reified deliberately under item 1.

> **Clef Note**: The two forms of item 1 are one rule with two carriers. Node-local codata serves an annotation whose consequence is consumed at emission (a settled support directing sparse generation, an escape class directing placement). Reification serves an annotation a backend pass must still consume after emission: the co-location annotation on the operations a partitioner groups is the standing instance. A $k$-ary geometric operation needs neither carrier to survive emission: its joint structure is consumed by generation, which emits the sparse arithmetic directly ([Grade Discipline §5.3](grade-discipline.md)). In both carriers the traversal that emits code remains local, and the hypergraph remains behind the emission boundary.

The witness SHALL retain or establish the correspondence between each consumed fact and the operations that realize it. A later transformation SHALL preserve that correspondence through an admitted rule or re-establish the affected property. Copying an attribute alone does not meet this requirement. Static discharge may justify an executable dynamic check or protocol; it does not establish the outcome of an unknown runtime input.

## 6. Domain Instances

> *Informative.* The rows below locate the hyperedge kinds this specification and the design corpus currently use. Each is normative only in the chapter that owns it.

| Hyperedge kind | Discharge | Owning treatment |
|---|---|---|
| $k$-ary join, meet, geometric product | Lattice family (QF_BV) over blade masks | [Grade Discipline §3.4](grade-discipline.md) |
| Co-location, route, synchronization | Lattice family over resources and reachability | Target pathway lowerings ([Backend Lowering Architecture](backend-lowering-architecture.md)); partitioning is pathway work |
| Mesh boundary sharing | Saturation equality over shared boundary nodes | Design-stage; the PHG paper §5.3 |
| Wait-for tuples, dependency width | Rank constraints (QF_LIA) | [Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md); analyzer is design-stage |
| Accumulation exactness (quire targeting) | Numeric representation, product and reachable partial-sum capacity obligations; blade masks can constrain the contributing terms | [Grade Discipline §5.3](grade-discipline.md), [Numeric Selection §10.2](numeric-selection.md) |
| Obligation residence — storage reservation, view containment, NUL sentinel, consecutive layout, capacity, input bounds | QF_LIA / QF_BV over graph literals at saturation; the same anchors re-derived from the artifact at build time and twin-paired | §5 of this chapter; [Conformance §6](conformance.md) |
| Platform residence — a node `Resides` in, or is `Constrained` by, a declared space or buffer | No solver: the declaration is cited by name as the obligation's authority | The platform description as declared authority ([Platform Bindings § Platform Descriptor](platform-bindings.md#platform-descriptor), whose `MemoryRegions` are the declared spaces; region kinds in [Memory Regions](memory-regions.md)) |
| Closure capture and continuation frame — ordered capture occurrences and site, or live-across participants and delimiter | VC-EXT/DIS/REG/REL/APP and VC-EXT/STATE/ACC/DOM/ONE under the owning representation contract; finite layout checks alone do not prove all usage or lifetime properties | [Closure Representation §11](closure-representation.md); [Delimited Continuation Representation §6](dcont-representation.md) |

## 7. Normative Requirements

1. **Embedding**: Every valid PSG SHALL be a valid PHG under the $|I_f| = 1$ embedding, with all PSG disciplines applying unchanged to binary hyperedges.
2. **Enumerated occurrences**: A hyperedge SHALL retain the enumerated positions, roles, order, and multiplicity required by its interpretation.
3. **Sound inference**: Every conclusion SHALL have its required premises; unresolved participants SHALL NOT become assumed facts. Sound partial inference is permitted.
4. **Convergence and readiness**: Each admitted analysis and structural elaboration SHALL provide the convergence or termination policy required by §3. Pending required facts SHALL prevent the dependent compilation commitment.
5. **No weakening**: Decomposition SHALL preserve the required joint conclusion and provenance under a justified rule.
6. **Per-family discharge**: A hyperedge spanning solver families SHALL retain coherent premises through separate per-family projections; exchanges between domains SHALL obey sound transfer rules.
7. **Emission transport**: A hyperedge's consequence SHALL reach emission only as saturated node-local codata or as a reified annotation (an attribute set on emitted operations); the emission traversal SHALL NOT query the hyperedge set; a backend pass SHALL consume reified annotations only.
8. **Diagnosis**: A contradiction or a required unresolved obligation at commitment SHALL be diagnosed under [Conformance §5](conformance.md). An admitted dynamic check SHALL retain its specified runtime success and failure behavior.

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
