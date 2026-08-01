---
title: "Grade Discipline"
weight: 170
category: Language
status: draft
---

> **Status**: Draft
> **Normative**: Prospective
> **Last Updated**: 2026-07-31
> **Companion Specs**: [algebra-declarations.md](algebra-declarations.md), [annotation-disciplines.md](annotation-disciplines.md), [units-of-measure.md](units-of-measure.md), [ntu-dimensional-architecture.md](ntu-dimensional-architecture.md), [width-inference.md](width-inference.md)

This chapter specifies the grade structure of geometric algebra as two annotation disciplines of different algebraic character, together with their inference, their interaction, and their erasure behaviour.

The discipline is optional. A compilation unit that declares no algebra (see [Algebra Declarations](algebra-declarations.md)) is unaffected by this chapter in every respect.

## 1. Why two disciplines and not one

The grade of a multivector is not a single quantity that composes under a single law. Three obstructions rule out a single integer-valued axis:

1. The geometric product of a grade-$p$ and a grade-$q$ blade occupies grades $|p-q|$, $|p-q|+2$, through $\min(p+q, n)$. The output is a set.
2. A general multivector is not homogeneous. A rotor is a sum of a grade-0 and a grade-2 part, and `v + B` is a legal expression with no single grade.
3. Grade admits no additive inverse in the value algebra, so the axis is a monoid and not a group.

Two quantities remain available as annotations.

**Parity** is $\mathbb{Z}_2$-valued, total on every multivector, and exact. It composes as a group under the geometric product. It is a measure generator and is specified in §2.

**Blade support** is a subset of the $2^n$ basis blades, total on every multivector, and a sound over-approximation. It composes as a bounded join semilattice under symmetric difference of the blade index sets. It is a coeffect and is specified in §3. Grade support, a subset of $\{0..n\}$, is its popcount image and is retained as a derived view.

A third quantity, the signature, is a property of the declared algebra and not of any value. It is specified in [Algebra Declarations](algebra-declarations.md).

## 2. Parity

### 2.1 Definition

Every element of a Clifford algebra decomposes uniquely into an even part and an odd part. The parity of a homogeneous element is its grade modulo 2. Parity is written as a measure generator supplied by the algebra declaration.

```fsharp
// Declared by the algebra; not user-defined
[<Measure>] type parity
```

A multivector type carries a parity annotation as a measure:

```fsharp
type Multivector<'A, [<Measure>] 'P>
```

where `'A` is an algebra reference and `'P` is `1` for even or `parity` for odd.

### 2.2 Composition

Parity composes under the geometric product by measure multiplication, with the torsion rule of [Units of Measure](units-of-measure.md#torsion-generators):

```fsgrammar
parity parity ≡ 1
```

The inference rules follow directly.

| Operation | Rule |
|---|---|
| `a * b` (geometric product) | $\pi_{\text{out}} = \pi_a \pi_b$ |
| `a ^ b` (outer product) | $\pi_{\text{out}} = \pi_a \pi_b$ |
| `a .| b` (inner product) | $\pi_{\text{out}} = \pi_a \pi_b$ |
| `a + b` | $\pi_a = \pi_b = \pi_{\text{out}}$ |
| `~a` (reversion) | $\pi_{\text{out}} = \pi_a$ |
| `-a` (grade involution) | $\pi_{\text{out}} = \pi_a$ |
| `grade<k> a` (projection) | $\pi_{\text{out}} = k \bmod 2$ |

Addition is the only operation that constrains its operands, and the constraint is an ordinary measure equation. No new solver machinery is required: the constraint `'Pa = 'Pb` is solved by the mechanism of [Constraint Solving](inference-constraint-solving.md#solving-equational-constraints) extended for torsion.

### 2.3 Parity is exact

The parity annotation is not an approximation. Where §3 gives a containment for blade support, §2 gives an equality. The compiler reports a parity constraint violation as a type error at the point of violation, diagnosed as `CLEF9620`, and does not defer it to elaboration.

In the majority of observed cases, the class of geometric-algebra error that produces a structurally meaningless result, as distinct from a numerically inaccurate one, is a parity error. The parity constraint is exact, and the check is one bit wide.

### 2.4 Parametricity

A term well-typed in the parity-extended measure algebra commutes with the grade involution, which is the algebra automorphism acting by $-1$ on the odd subspace. Parity instantiates the dimensional invariance property that [Units of Measure](units-of-measure.md) inherits from Kennedy's design, with the character group $\{\pm 1\}$ in place of $\mathbb{R}_{+}$.

> **Status.** Claimed, not proved. The interaction with the numeric representation layer, where the involution acts on stored coefficients and annotations erase before lowering, has not been discharged. The property is not relied upon by any normative requirement in this chapter.

### 2.5 Erasure

Parity is a measure and erases with the other measures, per [Measure Parameter Erasure](units-of-measure.md#measure-parameter-erasure). It is absent from code generation and does not affect layout.

## 3. Blade support

### 3.1 Basis indexing

Index the basis blades of an algebra of dimension $n$ by subsets of $\{1..n\}$: the blade $e_A$ for $A \subseteq \{1..n\}$, with $e_\emptyset = 1$ and $\mathrm{gr}(e_A) = |A|$. The basis product is

$$e_A e_B = \pm\, e_{A \oplus B}$$

where $\oplus$ is symmetric difference. The sign is fixed by a transposition count together with the squares of the generators in $A \cap B$, and the product is zero where a degenerate generator lies in $A \cap B$. Signs are tabulated at algebra declaration. See [Algebra Declarations](algebra-declarations.md).

### 3.2 Definition

The blade support of a multivector is the set of basis blades at which it may have a non-zero coefficient:

$$\beta(x) \;\supseteq\; \{\, A \subseteq \{1..n\} \;:\; \text{the } e_A \text{ coefficient of } x \text{ is non-zero} \,\}$$

The containment is the sound direction. The compiler computes an over-approximation. A component the analysis admits may be zero at runtime. A component the analysis excludes is zero by construction.

### 3.3 Representation

The support is a bitvector of width $2^n$, one bit per basis blade, indexed so that bit $i$ corresponds to the blade whose index set is the binary expansion of $i$. The width is fixed at algebra declaration.

| Algebra | $n$ | $\beta$ width |
|---|---|---|
| VGA `Cl(3,0,0)` | 3 | 8 bits |
| PGA `Cl(3,0,1)` | 4 | 16 bits |
| STA `Cl(1,3,0)` | 4 | 16 bits |
| CGA `Cl(4,1,0)` | 5 | 32 bits |

#### 3.3.1 Which stage this width belongs to

Four widths arise in connection with a multivector, at four nested stages. Conformance statements MUST distinguish them.

| Stage | Quantity | Governed by |
|---|---|---|
| Specification | $\beta$ is $2^n$ bits | This chapter |
| Host | Mask storage, Cayley sign cache, QF_BV query width | Implementation, §3.3.2 |
| MLIR | $\lvert\beta\rvert$ values at portable dialect types, carrying inferred widths | [Width Inference](width-inference.md), §5.2 |
| Backend leg | The physical realization of each width | The leg, §5.5 |

$\beta$ is present on the Program Semantic Graph, is read by CCS during inference and by Alex during lowering, and is absent from the emitted MLIR. The boundary holds because §3.7 makes an unresolvable support an error and not a runtime fallback, so every $\beta$ is statically determined and nothing carries it past emission.

A specification statement about $\beta$ SHALL NOT be expressed in terms of a host type, an MLIR type, or a device resource.

#### 3.3.2 Host representation

The mask, the Cayley sign data, and the constraint queries are compiler data structures on the machine running CCS, Composer, and Alex. Their cost is real and is an implementation concern, not a language property.

- **Per-node annotation.** $2^n$ bits on each node carrying a multivector type.
- **Cayley signs.** The sign for an ordered pair $(A,B)$ is one of $\{+1, -1, 0\}$ over $4^n$ pairs, computable in $O(n)$ from $A$ and $B$. A stored table is a memoization. An implementation MAY cache it fully, cache it partially, or recompute per query.
- **Alternative structure.** An implementation MAY represent the support as an explicit list of occupied blade indices at $\lvert\beta\rvert \cdot n$ bits against $2^n$. The list is smaller where $\lvert\beta\rvert < 2^n / n$ and requires a merge per operation where the bitvector requires a bitwise operation.
- **Solver width.** QF_BV queries carry $2^n$-bit bitvectors. The theory is decidable and the query cost grows with $n$.

Each choice above is unobservable in the language and in the generated program. This specification imposes no host word size and no packing convention, and states no bound on $n$: the practical bound is a compiler-resource figure that an implementation determines against its own graph sizes and solver budget.

The host row is contingent on two things that are scheduled to change: the machine CCS runs on, and the runtime CCS is written in. A concrete type written here would bind the compiler's representation to the bootstrap host. At self-hosting, the compiler's own blade masks become Clef values whose representation is selected by the discipline this chapter specifies.

### 3.4 Composition

| Operation | Rule |
|---|---|
| `a + b` | $\beta_{\text{out}} = \beta_a \cup \beta_b$ |
| `a * b` | $\beta_{\text{out}} = \{\, A \oplus B : A \in \beta_a,\ B \in \beta_b \,\}$, less pairs annihilated by a degenerate generator |
| `a ^ b` | as `a * b`, restricted to $A \cap B = \emptyset$ |
| <code>a .&#124; b</code> | as `a * b`, restricted to $A \subseteq B$ or $B \subseteq A$ |
| `grade<k> a` | $\beta_{\text{out}} = \{\, A \in \beta_a : |A| = k \,\}$ |
| `~a`, `-a` | $\beta_{\text{out}} = \beta_a$ |

On masks, symmetric difference is XOR. Addition is a bitwise OR. The product is an OR-reduction of $A \oplus B$ over the set bits of the two operand masks, bounded by $4^n$ bit operations and computable in $n\,2^n$ by an XOR-convolution. Every rule is a quantifier-free bitvector operation over a fixed-width bitvector and discharges in QF_BV. The bound is a compile-time cost and depends on no host or target word size.

No grade-indexed product table is required. Knowing that two operands are grade $p$ and grade $q$ leaves $|A \oplus B| = p + q - 2|A \cap B|$ undetermined, so a grade-indexed support would require a $(n{+}1)^2$ table of reachable grades to recover what the blade rule computes directly.

### 3.5 The grade view

$$\sigma(x) = \{\, |A| : A \in \beta(x) \,\}$$

is the popcount image of $\beta$. It is the vocabulary of the user-facing constraint syntax (`'T : grade S`) and of the diagnostics, and it is strictly coarser: for $x = e_{12} + e_{13}$ in `Cl(3,0)`, $\beta(x)$ has two elements and $\sigma(x) = \{2\}$ admits three. Layout SHALL be computed from $\beta$ and not from $\sigma$.

### 3.6 Support is a coeffect, not a type parameter

Blade support follows the discipline established by [Width Inference §5](width-inference.md#5-width-and-representation-as-a-coeffect). It is a requirement settled during design-time analysis, carried on the Program Semantic Graph, and read without recomputation by every later lowering pass.

Three consequences.

**It stays out of generalization.** A function polymorphic in blade support is annotated with the join of the masks at its call sites. The mask is a field on the PSG node, outside the unifier's input.

**It reports at elaboration.** The solver resolves support constraints by join and by containment check. Where a required support exceeds an inferred support, the compiler reports `CLEF9621` at the elaboration boundary, after constraint solving has completed.

**It is present in the middle end.** Alex computes the packed layout from the mask (component count $|\beta|$, one slot per set bit) and generates the sparse product from the operand masks, consulting the Cayley sign table for coefficients. Width receives the same treatment, for the same reason.

### 3.7 Unresolvable support

Where the analysis cannot bound a support, the compiler reports `CLEF9624` and requests an annotation. It does not fall back to the full support.

A fallback to the full support would type-check and would produce correct code. It would also discard the sparsity the representation argument rests on.

## 4. Interaction of the two disciplines

Parity is recoverable from a blade support whose members share a popcount parity, and is not recoverable otherwise. Carrying both is not redundant.

- $\pi$ is exact. $\beta$ is an over-approximation, and cancellation widens the gap between $\beta$ and the true occupied set.
- $\beta$ gives the component count. $\pi$ gives one bit.

Where the two disagree, replace $\beta$ with $\{\, A \in \beta : |A| \equiv \pi \bmod 2 \,\}$. The narrowing is not diagnosed.

This narrowing occurs during propagation inside the compiler. Neither annotation appears in the other's constraint system, and no joint query is issued. See §4.1.

### 4.1 Two solver calls

The annotation disciplines fall into two families. The group family (dimensional exponents, parity) discharges in QF_LIA. The lattice family (blade support, escape classification, capability, target reachability) discharges in QF_BV.

The two families range over disjoint sets of constraint variables. They are therefore discharged as two independent queries, and the composite cost is the sum of the two. An implementation SHALL NOT combine them into a single query: QF_BV is not stably infinite, so the classical Nelson-Oppen combination result does not license the combination, and no combination is needed because the variable sets do not meet.

## 5. Layout and lowering

Blade support fixes the number of coefficients a multivector carries. It does not fix their widths.

### 5.1 Component count

A multivector with blade support $\beta$ lowers to $|\beta|$ coefficients, one per set bit. It does not lower to a dense $2^n$ array, and it does not lower to a grade-padded layout allocating $\binom{n}{k}$ slots for each occupied grade $k$.

### 5.2 Component width at the MLIR level

The representation of each coefficient is selected independently by [Numeric Selection](numeric-selection.md) from that coefficient's analyzed range, under the discipline of [Width Inference](width-inference.md). Coefficients of one multivector are not required to share a representation, and an implementation SHALL NOT impose a common representation across a multivector's components in the absence of a range that justifies it.

Grouping components that share an analyzed range is a permitted coarsening. A grade band is a reasonable grouping heuristic and is not a rule.

Alex emits these as $\lvert\beta\rvert$ values at portable dialect types and commits to no target. A width at this level is a statement of what the computation requires, and not a claim about what any device provides.

### 5.3 Product generation

Generating the geometric product enumerates pairs $(A, B)$ with $A \in \beta_a$ and $B \in \beta_b$, accumulating into the coefficient for $A \oplus B$ with the sign from the Cayley table. The number of pairs contributing to one output coefficient is the length of that accumulation. Those accumulations are the ones targeted by the quire pass of [Numeric Selection §10.2](numeric-selection.md#102-the-quire-pass). Their lengths are determined from $\beta$ at design time.

### 5.4 Declared and derived support

Where a value is declared with a wider support than the analysis derives, the compiler emits `CLEF9625` as an advisory and packs to the derived support.

### 5.5 Leg realization

The backend leg realizes the widths of §5.2. The same emitted MLIR is realized differently per leg, and neither this chapter nor the annotation records the difference.

| Leg | Realization of a $w$-bit inferred coefficient |
|---|---|
| LLVM (CPU, MCU) | Rounded up to a native integer or float size for arithmetic. The analyzed range still governs representation choice and overflow checking. |
| CIRCT (FPGA) | Exactly $w$ flip-flops. A multivector's coefficients may carry as many distinct widths as they have distinct ranges, each narrowing its own carry chain. Posit or fixed-point per range. |

Arbitrary width is the intent in both cases, realized exactly on the CIRCT leg and rounded on the LLVM leg. Component count $\lvert\beta\rvert$ is invariant across legs. Only the per-coefficient realization varies.

## 6. Negative grade

Grade admits no additive inverse in the value algebra. During constraint solving, elimination may introduce a formal negative grade as an intermediate term. The constraint language is $\mathbb{Z}$ and admits it. The model is $\{0..n\}$ and does not.

A formal negative grade SHALL cancel before elaboration completes. A residual negative grade at the elaboration boundary is `CLEF9622`.

> **Design note.** The additive dual constructor carries the same cancellation obligation. This applies it to the grade axis. Whether negative grade admits a referent, and not only a discipline, is an open research question and is not part of this specification.

## 7. Normative Requirements

1. **Optionality.** A compilation unit that declares no algebra SHALL be unaffected by this chapter.
2. **Parity exactness.** Parity constraints SHALL be solved as equational constraints in the measure algebra and SHALL fail at the point of constraint violation.
3. **Support soundness.** An inferred blade support SHALL contain every basis blade at which the annotated value may have a non-zero coefficient.
4. **Support carriage.** Blade support SHALL be recorded as a coeffect on the PSG and preserved through lowering without recomputation.
5. **No silent default.** An unresolvable blade support SHALL be reported as an error requesting annotation. The compiler SHALL NOT default to the full support.
6. **Packed layout.** A multivector SHALL lower to a representation whose component count is the cardinality of its blade support.
7. **Grade view derivation.** Where a grade support is required, it SHALL be computed as the popcount image of the blade support. An implementation SHALL NOT carry a grade support as an independent annotation.
8. **Algebra identity.** Multivector types from distinct algebra declarations SHALL NOT unify.
9. **Negative-grade cancellation.** A formal negative grade SHALL NOT be present after elaboration.
10. **Parity precedence.** Where a parity annotation and a blade support are inconsistent, the support SHALL be narrowed to the parity-consistent subset and the narrowing SHALL NOT be diagnosed.
11. **Separate discharge.** The group-family and lattice-family constraints SHALL be discharged as independent queries. An implementation SHALL NOT issue a single combined query over both variable sets.
12. **Component width independence.** The representation of each coefficient of a multivector SHALL be selected from that coefficient's analyzed range. An implementation SHALL NOT impose a single representation across a multivector's components in the absence of a range that justifies it.
13. **Stage separation.** The blade support SHALL NOT appear in emitted MLIR or in the generated program. A conformance requirement SHALL NOT be stated in terms of a host type, an MLIR type, or a device resource, and a figure belonging to one of the four stages of §3.3.1 SHALL NOT be presented as governing another.
14. **Leg invariance.** The component count $\lvert\beta\rvert$ SHALL be identical across backend legs for a given program. Only the per-coefficient realization of §5.5 varies by leg.

## 8. Open items

- The parametricity claim of §2.4 requires discharge against the representation layer.
- The multivector-derivative convention is unsettled, so the parity behaviour of a derivative node is specified only as the support over-approximation. See the amendment note for the Decidable by Construction paper.
- The language server surface described in the PHG paper's §6.1 (grade resolution display, sparsity profile) is not specified here and is tracked separately.
- The interaction between blade support and the BAREWire schema discipline at process boundaries is unspecified. A packed multivector's wire layout depends on its support, and two peers must agree on it. [NTU Dimensional Architecture §7.3](ntu-dimensional-architecture.md#73-barewire-contract-verification) raises the same question for resolved widths, and the resolution is likely the same.

## References

- Kennedy, A. (2009). *Types for Units-of-Measure: Theory and Practice.* CEFP.
- Katsumata, S. (2014). Parametric effect monads and semantics of effect systems. *POPL*.
- Petricek, T., Orchard, D., and Mycroft, A. (2014). Coeffects: a calculus of context-dependent computation. *ICFP*.
- Dorst, L., and De Keninck, S. (2022). A guided tour to the plane-based geometric algebra PGA.
