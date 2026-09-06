---
title: "Numeric Selection"
weight: 440
category: Semantics
status: normative
---

> **Normative specification for compile-time selection of the numeric *representation* of real-valued quantities — posit, IEEE-754, or fixed-point — from a value's dimensional range, via the same coeffect machinery and [Program Semantic Graph](program-semantic-graph.md) carriage that width inference uses for integers.**

## 1. Overview

Numeric selection is the **real-valued counterpart** of [Width Inference](width-inference.md). Width inference sizes an *integer* from the bit count its value range requires; numeric selection chooses the *representation* of a *real* — whether a value is best carried as a posit, an IEEE-754 float, or a fixed-point number — from that value's dimensional range. The two are halves of one discipline: representation follows from analyzed range, never from a target type name. Unresolved facts remain pending during elaboration and are diagnosed when required for representation commitment, subject to the explicit bare-float policy in §6.

The objective is a single, deterministic, compile-time function: for a real value with range `[a, b]` on a target offering a set of representations `R`, select the representation that minimizes worst-case relative error over the range. This is the objective sketched in [Width Inference §4](width-inference.md), made sound here by two side-conditions the bare form requires; the present chapter specifies it in full, including those side-conditions, the tiered provenance model for the range input, the unobservable-range contract split on dimensionedness, the capability-coeffect treatment of performance, the concrete/parameterized representation scope split, and the quire pass that realizes exact accumulation.

The single observation that unifies the design: **the dimensional range, not the dimension, is the input to selection.** Knowing that a value carries dimension *meters* does not distinguish nanometers from astronomical units. The dimensional algebra (see [Units of Measure](units-of-measure.md) and [NTU Dimensional Architecture](ntu-dimensional-architecture.md)) establishes the *kind*; the concrete `[a, b]` establishes the *representation*. Authority over the range is therefore authority over the representation, because the objective is deterministic once the range is fixed.

> **Governing principle — design-time range selection provisions the runtime envelope.** A representation is selected whose dynamic range and precision profile *cover* the value's range with margin (§2.1), so that runtime — or training-time — behavior cannot out-run the representation's ability to preserve precision. For an evolving computation (e.g. an adaptive model whose distribution shifts during training), the *expected drift is part of the range*; the margin is sized to it. Preservation of representation is therefore a **design-time guarantee, not a runtime mechanism**: nothing re-selects at runtime. Intended loss is expressed in arithmetic whose range is analyzed ([Width Inference §7](width-inference.md#7-intended-loss-is-arithmetic)); a known failure of coverage is a hard error. A boundary declaration remains binding over inference (§3.3, §5), but its authority does not permit a value outside its declared range to pass the coverage check.

> **Lineage.** Kennedy's units-of-measure work supplies the dimensional algebra. Refinement and automated-verification work in F* and Dafny provides context for range obligations. Quotations and active patterns, developed in F#, inform the proposed library interface. Posit formats and quires follow their declared arithmetic specifications. These precedents explain the design without imposing their host representations on Clef.

### 1.1 Status discipline

This chapter distinguishes requirements that follow directly from prior art and standards from requirements that are this specification's own design construction. Where a requirement is a forced consequence of the objective plus the integer-twin infrastructure, it is stated as normative without qualification. Where a requirement is a design decision this specification *adopts* — implementable and recommended, but not compelled by external canon — it is marked **[Design decision]**. Genuinely unresolved items are marked **[Not yet specified]** rather than invented.

## 2. The Selection Objective

For a real value with range `[a, b]` on target `T` offering representation set `R(T)`:

\[r^* = \operatorname*{arg\,min}_{r \,\in\, R_{\mathrm{cov}}(T,\, [a, b])} \; \max_{x \,\in\, [a, b]} \; \mathrm{err}_r(x)\]

Three properties are load-bearing:

- **Accuracy-only objective.** The score is worst-case error over the range. There is no cost, latency, or area term. Performance is handled by *filtering the candidate set* (§6), never by perturbing the score.
- **Deterministic and compile-time computable** once `[a, b]` is fixed.
- **Per-target.** Both `R(T)` and the winner vary by target; the range does not.

The error models differ by representation family and decide the winner by *where the range sits*:

- **IEEE-754** distributes relative error approximately uniformly (≈ `2⁻ᵖ`) across its normal range — a representation that makes *no bet* on where values cluster.
- **Posits** taper: precision is maximal near magnitude `1.0` and degrades toward the regime extremes — a *bet on locality*.
- **Fixed-point** fixes a scale, trading dynamic range for uniform absolute spacing.

The bare argmin as written above is **ill-posed without two side-conditions**, specified in §2.1 and §2.2. The objective form shown already incorporates them: `R_cov` is the feasibility-filtered candidate set (§2.1), and `err_r(x)` is the ULP-floored error metric (§2.2).

### 2.1 The feasibility (coverage) constraint

**[Design decision.]** A candidate whose dynamic range does not cover `[a, b]` is *not* excluded by the raw argmin. For `x` beyond `r`'s maximum representable magnitude, `round_r(x)` saturates and the relative-error term approaches a finite value near `1.0` — so a non-covering representation produces a *bounded* score and can pathologically *win* when every candidate scores near `1.0`. A diagnostic bolted on after selection does not protect the objective. The candidate set is therefore filtered *before* the argmin:

\[R_{\mathrm{cov}}(T, [a, b]) = \{\, r \in R(T) \;:\; \mathrm{dynrange}(r) \supseteq [a, b] \,\}\]

with the explicit rule: **if `R_cov = ∅`, that is a hard coverage error (CCS8012)**, as required by [Conformance §5](conformance.md#5-the-diagnostic-obligation). No non-covering representation is selected. The diagnostic identifies the range, the target's offered representations, and the boundary declaration where applicable. Remedies are to establish a tighter valid range, express intended loss in arithmetic, or change the platform or boundary declaration. Warning policy cannot authorize a non-covering representation. The Tier-3 boundary check (§5) applies the same coverage check to a singleton `R`; no source seal or conversion is introduced. The coverage constraint is part of the objective, not a post-hoc warning.

A range or platform context that is still unresolved during elaboration is a pending obligation, not an empty coverage set. Selection waits for its required facts, with the diagnostic boundary specified in [Width Inference §6](width-inference.md#6-unobservable-ranges). Once a concrete range and offered set establish that coverage is impossible, deferred inference does not permit a fallback.

### 2.2 The zero-crossing error metric

**[Design decision.]** The relative-error term `|x − round_r(x)| / |x|` **diverges as `x → 0`**. Signed dimensioned ranges routinely straddle zero (e.g. a membrane potential range `[−80, +40] mV`). For any range containing a neighborhood of zero, the unmodified worst-case is unbounded for *every* candidate — IEEE included, since near zero IEEE precision is governed by the subnormal floor, not by `2⁻ᵖ` — and the argmin is undefined.

The error metric is therefore a **mixed absolute/relative form with an ULP floor**:

\[\mathrm{err}_r(x) = \frac{|x - \mathrm{round}_r(x)|}{\max(|x|,\, \mathrm{ulp}_{\min}(r))}\]

where `ulp_min(r)` is the magnitude of `r`'s smallest representable positive normal (IEEE) or smallest representable magnitude in the regime covering the range (posit / fixed-point). Below `ulp_min(r)` the metric becomes effectively absolute, which is the correct near-zero semantics: absolute spacing is what matters near zero, and the contest becomes "whose smallest representable magnitude best resolves the near-zero cluster." An equivalent formulation excludes an `[−δ, +δ]` neighborhood of zero from the relative-error worst-case and scores near-zero behavior by `ulp_min(r)` directly.

A consequence of correct posit modeling: **posits taper toward `1.0`, not toward zero.** Zero is a regime extreme where posits, like every representation, lose precision. Any claim that posits deliver high precision *near zero* is arithmetically false and SHALL NOT appear in selection diagnostics. The genuine near-zero argument for posits is the *exact accumulation* the quire provides (§8), not tapered near-zero precision.

> **Posit32/es2 anchors.** Dynamic range ≈ `[10⁻³⁶, 10³⁶]`; relative error ≈ `2⁻²⁷` near `1.0`, degrading toward ≈ `2⁻⁸` at regime extremes. These two figures are **illustrative endpoints of a continuous taper**, not a two-point error model. The worst-case over `[a, b]` depends on where `[a, b]` sits on the taper — which is the entire point of the objective. An implementation SHALL compute the taper at the actual range endpoints and SHALL NOT hard-code the endpoint figures.

## 3. The Tiered Authority Model

The range input `[a, b]` has exactly three provenances. These are **not** three selection algorithms — they are three provenances and bindingness levels of the *one* objective's input. All three feed the same selector and produce identical lowering: the pipeline treats every range claim identically regardless of provenance. The three tiers map onto the Level 1 / Level 2 / Level 3 inference hierarchy used throughout Clef (see [Incremental Computation §12](incremental-computation.md) and [Width Inference §5](width-inference.md)).

| Tier | Range source | Inference Level | Developer writes |
|---|---|---|---|
| **1 — Intrinsic** | interval analysis + profiling evidence | Level 1 (inferred) | nothing (rides the width-inference frame; see §3.1 caveat) |
| **2 — Library-assisted** | library constants / physical laws | Level 2 (bounded) | `open Fidelity.Physics.X` |
| **3 — Direct** | a declaration the compiler reads: a boundary's declared representation, or the one supplied hypothesis about input data | Level 3 (explicit) | a declaration (platform description, wire schema, binding descriptor, register width); never a type name in source |

### 3.1 Tier 1 — Intrinsic

The compiler obtains `[a, b]` from interval analysis over the PSG composed with dimensional inference — the same *kind* of analysis width inference runs for integers, but over a **real/dimensional interval domain that does not yet exist in the integer twin** (§9). Tier 1 succeeds *automatically* only for computations whose range is bounded by dataflow alone: closed-form expressions; division only by a quantity with a known non-zero lower bound; profiled values; or already-annotated values. For real physics with division by a quantity whose lower bound is not in dataflow (e.g. `r²` in a denominator), **Tier 1 cannot bound the range and falls through to Tier 2 or Tier 3, or to the §7 error.** This scope limit is honest and load-bearing: the headline gravitation example (§4) is a *Tier 2* success, not a garden-path success.

### 3.2 Tier 2 — Library-assisted

A domain library supplies the range that intrinsic analysis cannot derive. **Tier 2 has authority over the range and is advisory over the representation:** the library states that membrane potentials live in `[−80, +40] mV`; the compiler runs the argmin against the target's format set. The domain expert knows the physics range; the compiler knows the target's formats and their error profiles. The mechanism is specified as a design sketch in §4; the precise binding contract is **[Not yet specified]**.

### 3.3 Tier 3 — Direct

The third provenance is a declaration the compiler reads, not a type the developer writes. A boundary fixes a representation by declaration: a wire-schema field, an MMIO register width, a C ABI parameter carried by a binding descriptor, an endpoint contract, an exported entry point at the platform's declared word. Where a value meets one, the compiler runs selection *in reverse*: it checks that the declared representation's dynamic range covers the value's inferred range, the §2.1 coverage check on a singleton `R`, and emits a coverage diagnostic if not (§5). A declared representation wider than the range needs still compiles and is witnessed at design time. There is no seal form and no seal syntax: no width-named numeric type exists in the language ([NTU Types](ntu-types.md); [Width Inference §7](width-inference.md)). Where the compiler cannot infer a property of input data, the one supplied hypothesis is a number about that data in one construct, never a representation claim on a value.

### 3.4 Tier composition — precedence-override

**[Design decision.]** When more than one tier produces a range claim, composition is a **single, total, precedence-ordered refinement**, not a set intersection:

1. Each present tier yields a **range claim**: `R₁` (dataflow, if Tier-1 analysis terminates with a bound), `R₂` (library), `R₃` (a boundary declaration, which fixes a representation and supplies its dynamic range as a range claim).
2. **The binding range is the claim of the highest present tier** (`R₃` if present, else `R₂`, else `R₁`). Higher tiers *override*; they do not silently merge. This is the entire meaning of "precedence."
3. **Lower-tier claims become consistency obligations, not inputs to the selected range.** When a lower tier also produced a claim, the compiler checks containment `R_lower ⊆ R_binding` over sound bounds. Any declared measurement uncertainty or rounding allowance belongs in those bounds before the check; it cannot waive a known coverage failure (§14). If the *observed* dataflow range `R₁` is not contained in a higher tier's declared range, that is a **diagnostic** — the dataflow observed values the domain library or boundary claimed impossible (the real-valued analogue of integer overflow, a genuine bug signal). The diagnostic is a hard consistency error and does **not** change the binding range. A contradictory lower-tier observation prevents committing that selection to lowering; retaining the declaration as the binding claim does not authorize code outside its range ([Conformance §5](conformance.md#5-the-diagnostic-obligation)).
4. There is no empty-intersection state, because there is no intersection. Disagreement is always *binding-range-wins, lower-claim-flagged*, which makes composition total and decidable.

This mirrors the Level-1/2/3 hierarchy (inferred → bounded → explicit, each level authoritative over the one below) as **monotone override with disagreement-witnessing**, not as a lattice meet.

## 4. The `Fidelity.Physics` Mechanism (design sketch)

`Fidelity.Physics` is **planned, not built.** This section is a design sketch, not a settled specification; the binding contract is **[Not yet specified]** (§11).

The mechanism is **quotation-symbolic**: the library carries a range *law*, and the compiler *derives* the range by evaluating it. The derived capability — obtaining a range such as a force interval from the gravitation law — is the reason the law, rather than a precomputed range, is the carried object.

The native Clef quotation must retain the law's numeric kinds, dimensional parameters, and value-level premises. PSG elaboration checks that structure and evaluates the admitted law over justified input ranges. The result includes a sound enclosure and its provenance, which remain available when a representation is selected.

A hosted library can transport a quotation through an unmeasured `Expr<float -> ... -> float>` plus companion metadata where its host requires that encoding. This is a transport convention. Before using the law, Clef must reconstruct and validate the dimensional correspondence from the supplied metadata. Missing or contradictory dimensions are diagnostics. The hosted encoding does not define the type of a native Clef quotation or permit erasure of its source-level measure identity.

This is a design sketch of a planned, unbuilt mechanism; the input-range leaves are tabulated, and the export interface and details of the admitted bound-evaluation language remain open (§14).

> **Bound evaluation and selection.** The selected representation is a member of the target's finite candidate set. That finite result does not establish decidability or cost bounds for every upstream obligation. The compiler retains the sound input enclosure, the selected format's declared properties, and the justification for coverage. Product and partial-sum adequacy for a quire require the numerical obligations of §10.2.1. Any regime classifier used to reduce the search must preserve the covering candidates needed by the specified objective. A representation catalogue need not form a lattice under a single order.>
> Two consequences for the bound computation that feeds the classifier:
>
> - It must be **total and terminating**. The quotation range-law is restricted to a **total, terminating, closed-form sub-language**, and its bound `[a, b]` is obtained by **directly evaluating the expression in the outward-rounded interval domain of §9.1 — an image computation, not a satisfiability query handed to a decision procedure.** Algebraic operations (`+`, `−`, `×`, reciprocal, `sqrt`) evaluate to image intervals by the standard outward-rounded rules (including the sign-crossing reciprocal split); the `r²`-in-denominator case (§3.1) is bounded here because the Tier-2 contract supplies `r ≥ r_min > 0`, so `r²`'s interval excludes zero. Transcendentals (`exp`, `log`, `sin`) evaluate on **pre-split non-oscillatory segments** where each is monotone (for `sin`, split at the half-period extrema), so the image interval is read from the endpoint images. They are deliberately *not* admitted as a logical theory — the first-order theory of the reals with `sin` is undecidable, which is exactly why the design evaluates rather than decides over them. The bound computation forms no logical formula at all; it is an image computation in the interval domain. Each downstream obligation dispatched as a formula must identify its supported theory and encoding. Finite output categories alone do not establish that every source law translates to linear integer arithmetic or bitvectors.
> - The **method** used to compute the bound (interval arithmetic, affine forms, or any other sound image method) is an implementation detail of one terminating evaluation. Its safety obligation is **sound enclosure**: the interval must contain every result permitted by the stated premises. **Tightness** additionally bears on completeness and selection quality. An over-estimated bound remains sound, but an interval over-estimate where a variable recurs (e.g. `r` in `r·r`, the dependency problem) can push the bound across a regime boundary, change which lattice element wins, or manufacture a spurious `R_cov = ∅` error where a tighter method would have found a covering element. Refining that bound can resolve the error; accepting an under-covering representation cannot. No particular refinement algorithm is required here, and the tightness target remains open in §14.
>
> The remaining open detail is the precise segment-splitting convention for each admitted transcendental (§14).

The proposed library interface carries:

1. **Dimensional parameters.** The law's input and result dimensions, checked by the measure algebra.
2. **A typed quotation.** The range-law expression and its value-level premises, evaluated in the admitted bound domain.
3. **Range classification.** An optional classifier over the resulting enclosure, subject to the coverage and selection requirements.
4. **Registration and provenance.** An association between the law, its declaration, and its accepted justification, so the compiler can instantiate it at a relevant use site and check its premises.

The same quotation form is a candidate interface for parameterized lemma libraries. A quoted proposition is an obligation until its justification is accepted. A domain library can supply an established theorem for repeated automatic instantiation, while each use still requires its premises to hold. The general proof-library admission contract is outside this numeric-selection sketch.

The developer experience, under the honest scope of §3.1:

```fsharp
open Fidelity.Physics.OrbitalMechanics
let gravForce (m1: float<kg>) (m2: float<kg>) (r: float<m>) : float<N> =
    GravConst * m1 * m2 / (r * r)   // dimension inferred N (free, HM+ℤ unification);
                                    // range SUPPLIED BY Tier 2 — Tier 1 alone
                                    // cannot lower-bound r² in the denominator
 
```

The compiler infers `N` for free; the **range comes from Tier 2**, precisely because Tier 1 dataflow cannot lower-bound `r`. This is a Tier 2 success that depends on a library that is planned, not built.

> **ML activation tuning.** A `Fidelity.ML` companion library ships an asymmetric posit configuration (asymmetric es/regime, exponent bias shifted to center precision on the activation mode, narrow range ≈ `[10⁻¹⁴, 10¹]`) so an ML author obtains the tuned representation without hand-deriving it. **The asymmetry is supplied as input, not as a second objective.** Rather than replacing §2's symmetric worst-case argmin with a distribution-weighted (expected-error) objective in the general selector, `Fidelity.ML` feeds the existing selector a **range already skewed to the activation density** (the narrow, bias-shifted interval above): the symmetric argmin run over that skewed range naturally selects the asymmetric, bias-shifted configuration, because the range endpoints it scores already sit where the distribution's mass is. This keeps the general selector single-objective and exact (Requirement 4), and confines the distribution knowledge to the Tier-2 library that owns it. A genuinely distribution-*weighted* objective — integrating expected error against an activation density rather than scoring a skewed range's worst case — remains a possible future refinement scoped strictly to the ML domain library (§14), but is not required for the shipped behavior.

## 5. Boundaries and Reverse Selection

A boundary declaration fixes a representation at a site (§3.3). The compiler discharges exactly the §2.1 coverage check on the singleton candidate set:

- If the declared representation covers the value's range, the value takes it, the transfer is exact, and nothing is written in source. A covering-but-suboptimal declaration compiles and SHALL be witnessed at design time with the representation the open argmin would have chosen (CCS8014, information).
- If it does *not* cover the range, that is the §2.1 hard coverage error, CCS8012; the remedies are to bound the value in arithmetic (`%`, `clamp`, a guard) or to change the declaration. The compiler never inserts a conversion and never selects a wrap or saturate discipline ([Width Inference §7](width-inference.md)).

The parameterized `posit<n, es, rs, bias>` of §8 item 3 is a synthesis configuration for reconfigurable targets and never appears in source. Concrete representation names (`Posit32`, `f64`, `int` at 32 bits) are the codomain of selection and the vocabulary of platform declarations, never types in the language.

## 6. The Default and Unobservable Case

**[Design decision.]** Numeric selection distinguishes pending inference during elaboration from the boundary that commits a concrete representation ([Width Inference §6](width-inference.md#6-unobservable-ranges)). At that boundary, the unobservable-range contract is split on **dimensionedness**:

- **Dimensioned real with unobservable range → ERROR at representation commitment.** A `float<newtons>` whose required range remains unresolved is reported with its missing provenance. The dimension establishes the kind of quantity; it does not supply a numeric magnitude bound. That bound may be established by later source context or an applicable domain or platform declaration. The obligation need not be discharged by an annotation at the first dimensioned expression, and the compiler SHALL NOT fabricate a representation while it remains pending.
- **Bare `float` with unobservable range → IEEE `f64`, when offered and permitted.** This is the specified bare-float exception. It preserves the IEEE path without requiring a locality claim about where values cluster. It does not prove that an unbounded mathematical range fits a finite representation, authorize an unavailable format, or override a known coverage failure. The target's binding must offer `f64` and the capability policy of §7 must permit it; otherwise the required capability is diagnosed before commitment. Bare reals remain subject to range propagation, and known bounded ranges are checked under §2.1.

### 6.1 The bare/dimensioned seam

**[Design decision.]** Ranges are composition-dependent, so the bare/dimensioned boundary needs explicit handling. Consider:

```fsharp
let x : float = bareInput in
let y : float<newtons> = x * oneNewton
```

Here `x` is bare (no-error IEEE path), but `y` is dimensioned and would error on an unobservable range — and `y`'s range derives from `x`'s, which was never bounded. The resolution:

- **Bare floats carry interval analysis.** There is no second analysis to switch off; the same real interval domain (§9) runs on every real value. The bare-float exception permits the `f64` path under §6's capability conditions; it does not exempt bare floats from range propagation or coverage of known bounds.
- **The dimensioning site is where the range obligation originates.** When a bare value with an unobservable range flows into a dimensioned context, the obligation attaches to that site (here, `x * oneNewton`) and may remain pending while further context is established. If still unresolved at representation commitment, the error is located at this origin and *names the upstream bare source*: "`y : float<newtons>` requires a bounded range; its range derives from `x` (bare `float`, unresolved at <site>); establish a bound on the input or an applicable domain or boundary declaration." An editing environment may surface the pending obligation earlier without rejecting a consistent partial program ([Width Inference §6](width-inference.md#6-unobservable-ranges)).
- The pit-of-success boundary is therefore: **dimensioning is the trigger for the range *obligation*; the obligation is discharged by bounding the dimensioned value's range, which may require bounding the bare inputs that flow into it.** The seam does not leak, because the obligation attaches to the dimensioned boundary and the diagnostic traces provenance across it.

There is one source real kind, `float` ([NTU Types §2.2](ntu-types.md#22-the-numeric-kinds)). `f32` and `f64` are representation choices, never additional source types. The bare-float exception specifies one unobservable-range case; dimensioned and known ranged reals retain their selection obligations.

## 7. Performance as a Capability Gate

**[Design decision.]** Performance never enters the *score*. It enters as a **capability-coeffect filter on the candidate set**, with a three-valued capability per format: *native*, *emulated*, or *unavailable*.

```
R(T)      = { r : capability(T, r) ≠ unavailable }            -- the target's offered formats
R_cov     = { r in R(T)  : dynrange(r) ⊇ [a, b] }             -- §2.1 feasibility
R_eff(T)  = R_cov filtered by the per-build emulation policy:
  native-only         → { r in R_cov : capability = native }
  allow-emulated      → R_cov                                 (default)
  allow-emulated-warn → R_cov, with a perf diagnostic if r* is emulated
r* = argmin (over r in R_eff)  max (over x in [a, b])  err_r(x)   -- §2.2 metric
```

The relationship to the bare objective is explicit: `R(T)` is the *offered* set (capability not *unavailable*); `R_cov` and the emulation policy are filters layered above it. The argmin score is untouched.

Two reasons performance is a filter and not a cost term:

1. **Worst-case relative error is closed-form and target-microarchitecture-independent; cost is not.** Folding a `λ·cost` term into the score would smuggle delay-model guessing into a function that is otherwise exact.
2. **Binary, witnessed outcomes, not silent degradation.** A blended objective produces a silently degraded choice. A capability gate produces a witnessed outcome: a format is in `R_eff` or it is not, and an emulated accuracy-optimal choice yields a *diagnostic, not a silent swap*. This generalizes the quire contract — a target lacking quire support triggers a capability-coeffect failure (§8), never a fallback to lossy accumulation.

> **b-posit hardware.** b-posits close the float/posit hardware-efficiency gap by bounding the regime field to 6 bits, so one decoder and a five-way MUX serve 16/32/64-bit operands (the non-significand field is identical across precisions, which IEEE structurally cannot do). Jonnalagadda, Thotli, and Gustafson (arXiv:2603.01615) report a 32-bit b-posit decoder at 79% less power, 71% smaller area, and 60% less latency than a standard posit decoder, matching or beating IEEE-754 float, with the margin widening at 64-bit. On targets that carry b-posit hardware the capability gate reports `capability = native` and reduces to an accuracy-vs-range choice. The design stays robust to how each target actually ships: where hardware is absent the gate reports `capability = emulated` and filters under `native-only`, and the accuracy-vs-range conclusions do not change; only the size of the native candidate set does.

The `allow-emulated-warn` policy lets emulated-ness influence *which diagnostic fires* — cost re-enters as a *reporting* side channel only, never as a selection input. The pure-accuracy invariant is scoped to the *choice*, not to what is reported about it.

## 8. Concrete vs. Parameterized Representations

**[Design decision.]** The representation space lives on three layers, split across the CPU/FPGA fork so that surface syntax and synthesis configurations never collide:

1. **Surface: `float<dim>`.** A range is a coeffect on the node; no posit width appears in the source type. Tier 3 supplies a boundary declaration in the platform vocabulary (§5), never a source `Posit32` type or seal ([NTU Types §2.2](ntu-types.md#22-the-numeric-kinds)).
2. **Lowering codomain on fixed-ISA (CPU / SIMD / RISC-V): four concrete types `Posit8 / 16 / 32 / 64`.** Direct struct layout, clean SRTP dispatch, clean hardware mapping, clean error messages, clean quire pairing (a `Quire32.fma` takes `Posit32` by construction). Selection chooses *among these concrete types* plus IEEE and fixed-point. Concrete types are the **codomain of selection, not the surface syntax.**
3. **Parameterized `posit<n, es, rs, bias>`: FPGA / reconfigurable-only synthesis search.** The `(rs, es)` grid is enumerable (`rs ∈ [2, 6]`, `es ∈ [1, 5]`, ≤ 25 points); **bias and asymmetry are a bounded but continuous parameterization explored heuristically, not enumerated.** Calling the *full* space "enumerable" would be wrong. This is type-directed *hardware synthesis*, not a type the developer instantiates, and it is future work.

ML routing follows from this split: the Tier-2 library supplies the narrow range and the asymmetric-bias recommendation, which lower to a **parameterized b-posit config on FPGA** and to the **nearest concrete `Posit8` on fixed-ISA**. A precision-floor finding concerns accuracy; failure to cover the value's dynamic range is the hard coverage error of §2.1. The two findings SHALL NOT be conflated.

## 9. Carriage and the Real Interval Domain

Numeric selection rides the same Program Semantic Graph coeffect frame that width inference uses for integers. The pattern to replicate, with honest cost annotations:

1. **PSG coeffect computed pre-emission.** Interval analysis runs once per graph before transfer; the result is carried as a coeffect. Numeric selection adds a peer `RepresentationSelection` coeffect beside the width-inference coeffect, computed in the same coeffect pass: range propagation runs in Elaboration wherever no platform fact is needed and is closed, together with selection, at Saturation against the platform description of the section ([NTU Dimensional Architecture §4.3](ntu-dimensional-architecture.md)). *(Cheap: a new field and a new producer.)*
2. **Abstract sentinel at type-lowering.** Just as platform-word integers lower to an abstract width sentinel resolved at narrowing, reals lower to an abstract real/float sentinel resolved by a single `selectRepresentation` choke point into `posit<n, es, bias>`, IEEE `f32`/`f64`, or fixed-point. *(Moderate: a new sentinel and a new resolver; this hook does not exist for reals today.)*
3. **Single resolver choke point**, analogous to integer narrowing.
4. **Hard error on unobservability**, inherited and split on dimensionedness (§6).
5. **Check-time diagnostics**, emitted in the same shape as the existing width-inference and FPGA diagnostics (§10).
6. **Platform capability facts**, the natural seat for "does this target have native posit hardware, an extended posit instruction, or quire support?" (§7), and for the **boundary semantics** of each offered representation: what an operation does when its result leaves the representation's dynamic range (wrap on a two's-complement integer unit, saturation on a posit unit or a saturating DSP block, exact on fabric where the width is the range's). The range coeffect checks whether the operation's interval image fits the declared boundary. A possible crossing is the hard coverage error of §2.1, resolved by a tighter established bound, a covering boundary declaration, or arithmetic that explicitly expresses the intended loss. Hardware wrap or saturation behavior is used only to realize the program's declared arithmetic semantics; it never supplies an implicit conversion or licenses a non-covering representation.

### 9.1 The real interval domain is a new abstract domain, not a port

**[Design decision.]** The integer interval domain is the five-case lattice of `ValueRange` (empty, bounded, a half-line above or below, unbounded) with exact transfer functions that saturate an endpoint to its infinity, and a width derived from the range by [Width Inference §3](width-inference.md) on read. A real/dimensional interval domain reuses the PSG **traversal skeleton, the coeffect-carriage discipline, the sentinel/resolver pattern, and the diagnostic plumbing** — but it does **not** reuse the transfer functions, which must be written from scratch over a continuous interval domain. Minimally it requires:

- **Outward-rounded floating-point interval arithmetic** (sound interval endpoints round outward to remain a superset).
- **Reciprocal/division with sign-crossing**: `1/[lo, hi]` where the interval contains zero splits into two unbounded pieces — exactly the `r²`-in-denominator case that makes Tier 1 fall through (§3.1). This is one instance of the general rule that a nonlinear interval operation is a sign-case analysis (interval multiplication is the bounded case; see [Rounding §4.3](rounding.md)); reciprocal is the case whose pieces are unbounded.
- **Transcendentals** (`sqrt`, `log`, `exp`, `sin`) — the physics use cases need them.
- **A terminating widening over a continuous lattice** — the integer monotone-widening fixpoint does not transfer; real interval widening needs an explicit **widening-with-thresholds** operator (`∇`) to guarantee termination. The thresholds SHALL be the dynamic-range boundaries `±dynrange(r)` of the target's concrete format set: when an interval loop variable expands past a threshold, the widening snaps it to that format boundary, so the analysis terminates in a number of steps bounded by the (finite) count of format boundaries rather than ascending a continuous chain. For integers the thresholds are the declared representations' range boundaries of the sign-selected family, then the program's settled constants, then the infinity (clef `Dimensional_Range_Design.md` §1.2a, corrected 2026-09-05); for reals they are the dynamic-range boundaries `±dynrange(r)` of the target's concrete format set.

This is a new abstract interpreter of research-grade weight, materially harder than the integer one it rides. "Same traversal" means the same graph traversal and carriage, not the same transfer functions.

### 9.2 The third reading of one PSG traversal

Width inference is the *spatial* reading (bits per value); pipeline/combinational-depth inference is the *temporal* reading (chained operations per register boundary). Numeric selection is the *third* reading of the same PSG traversal: it consumes the dimensional range and selects a representation. As in §9.1, "same traversal" is the graph traversal and carriage, not the transfer functions.

## 10. The Preservation Chain and the Quire Pass

### 10.1 Preservation chain

Representation selection occupies the second arrow of the lowering preservation chain:

```
Dimension --(range)--> Representation --(width)--> Footprint --(escape)--> Allocation
   DTS        Tier 1/2/3     selection coeffect      quire pass        escape analysis
            authority        (argmin / boundary)
```

Each step consumes the preceding inference's output; the chain is verified during elaboration and surfaced at design time. These coeffects settle in the PSG when their required source and platform facts are available ([NTU Dimensional Architecture §4.3](ntu-dimensional-architecture.md#43-dimension-resolution-flow)). Deferral preserves pending obligations; it does not delegate semantic decisions to a witness that lacks their premises. The representation choice is recorded as **codata on the PSG beside the dimension, the grade, and the escape class** — a lowering pass that does not touch representation leaves the annotation as it found it. Certified proof-transformer passes preserve it by construction; uncertified passes receive a per-edge re-check ([Conformance §6](conformance.md#6-the-preservation-obligation-through-lowering)).

> **Cast fidelity.** A `posit32 → f64` widening is labeled **cast fidelity 1.0 (one-way widen)**: posit32's significand near `1.0` fits within f64's mantissa, so the widening is lossless. The round-trip `f64 → posit32 → f64` is **< 1.0**. The fidelity label applies to the one-way widening cast only and SHALL NOT be read as implying a posit32 value is f64-accurate.

> **Cross-target transfer fidelity.** A transfer edge that changes representation SHALL record whether every admitted source value is exactly representable at the destination. A fidelity annotation of **1.0** requires coverage and exact representability, including the precision needed across the admitted range. Dynamic-range coverage alone is insufficient. A lossy transfer SHALL carry a justified error bound and satisfy the declared boundary contract. The annotation is directional and derived from the source and destination specifications.

### 10.2 The quire pass

**[Design decision — placement.]** The quire pass is a Composer nanopass, **downstream of selection** (it depends on the selected posit width) and **before the target-lowering fork**. It recognizes accumulation, sizes the quire, writes a coeffect bundle, and defers lowering to target-binding — annotations, not instructions, the same coeffect discipline selection uses.

- **Recognition.** An active pattern over PSG nodes recognizes `fma`/`fold`/`reduce`-of-products over a selected posit type, e.g. `Array.fold (fun acc (a, b) -> acc + a*b) zero`, and fuses it into a quire MAC.
- **Sizing.** The quire width `Q` follows the selected format. A b-posit (arXiv:2603.01615) uses a fixed universal quire of 800 bits (25 32-bit integers) for any width `n > 12`, independent of `n`. A full-gamut posit (Posit Standard 2022) uses `16n` bits, precision-dependent — 512 bits for posit32.
- **Coeffect carriage**: four coeffects on one PSG node: *allocation* (100 B for a b-posit quire; 64 B for a full-gamut posit32 quire; stack or arena by escape class); *lifetime* (loop scope unless escaping, via the existing escape analysis); *capability* (is exact accumulation available on this target?); *dimension* (an `fma` of `newtons × meters` accumulates as `joules`, with a single rounding at `Quire.toPosit` and the dimension verified at output).
- **Per-capability lowering at the fork.** The quire is substrate-portable: it is placed on whichever processor the accumulation is advantaged, and each substrate carries it at its own cost. The b-posit's fixed 800-bit quire is the enabling property — one accumulator width for every precision, so the same recognition and coeffect emission lower to any of these without re-sizing per format.

| Target | b-posit 800-bit quire residency | Local cost character |
|---|---|---|
| x86_64 (AVX-512) | two 512-bit `zmm` registers (1024 bits, 224 bits slack); 25 32-bit lanes | possible vectorized shift-add, subject to register pressure, carry propagation, and the declared memory-coherence contract |
| Xilinx FPGA | a fabric accumulator (fixed-point shift-add tree); many quires bank in parallel across the LUT budget | fabric implementation with latency and throughput established by synthesis for the selected target |
| RISC-V + extended posit | an architectural quire register | latency and throughput supplied by the declared extension |
| Neuromorphic | not available | **capability failure** |

Where the quire sits is a placement decision, not a fixed target: the accumulation is a sequential, stateful reduction, so the design weighs each substrate's *local* accumulate cost against the *transport* cost of the boundary it would otherwise cross. A quire co-located with the ALUs that feed it moves one rounded result across a slow boundary; a quire on the far side of that boundary moves every operand. For a full-gamut posit the residency figures scale with `16n` (a full-gamut posit32 quire is 512 bits, compared with 800 bits for a b-posit quire); the placement reasoning is identical.

#### 10.2.1 The quire-adequacy invariant

The quire width `Q` is fixed by the selected format. Its allocation size is independent of the number of products. Its capacity is finite, so exact accumulation requires bounds on the products and on every partial sum in the chosen evaluation order.

> **Quire adequacy.** The allocated accumulator SHALL match the selected format's field layout. Each product SHALL be exactly representable, and every reachable partial sum SHALL remain within the finite quire range. The compiler SHALL establish both obligations before committing an exact-accumulation lowering.

An accumulation-length bound together with operand bounds can establish partial-sum coverage. A checked invariant can establish it without a fixed length bound, for example when every reachable partial sum stays in a bounded interval. The expression `k · bits-per-product ≤ Q` is not the capacity law for addition. Repeated positive products eventually exceed any fixed accumulator unless their number or accumulated magnitude is bounded.

A full-gamut posit quire has `16n` bits under the [Posit Standard (2022), §§3.1–3.4](https://posithub.org/docs/posit_standard-2.pdf). The b-posit design specifies an 800-bit quire for `n > 12` (arXiv:2603.01615). Neither fixed width establishes unlimited exact accumulation. Missing partial-sum evidence MAY remain a pending obligation during elaboration, but SHALL produce a located diagnostic if unresolved when committing the accumulation. Rounding intermediate blocks changes the exactness contract and SHALL NOT be introduced as a silent fallback.

#### 10.2.2 Why the quire matters

A quire preserves exact sums of products of its represented inputs while the adequacy conditions hold. This prevents rounding during accumulation, including cancellation between those products, until the final conversion. It does not undo errors already introduced into the inputs or guarantee exact algebraic identities throughout a larger computation. A dimensional or grade proof and a numerical error bound remain separate obligations.

## 11. Design-Time Surfacing and Accuracy Preservation

The selection objective is computable at design time, so the compiler can show — *before anything runs* — **how much relative accuracy each candidate representation preserves across the value's actual range.** This is the capability with the most direct leverage for users, and the clearest demonstration that the compiler is materially different from an IEEE-only framework: the **tapered-versus-uniform tradeoff is made visible and quantitative at authoring time**, not discovered empirically after a long run drifts. IEEE-754 spends precision uniformly because it makes no bet on where values cluster; a posit concentrates precision where the values are. Where a domain's values live near magnitude `1.0` — true after natural-unit normalization for most physics, and for normalized ML activations — a posit preserves materially more accuracy than IEEE-754 *at equal width*, and the developer sees exactly that, per target, at the point of writing the code. Surfacing that bet and its payoff at design time is the showcase capability of numeric selection.

Numeric selection emits **check-time diagnostics** in the same shape as the existing width-inference and FPGA diagnostics (severity, source range, related nodes, reachability), from a pass inside program checking, surfaced as squiggles and hovers. "Design time" here is continuous Lattice elaboration, distinct from ML "compile time." Canonical readouts (taper figures are illustrative continuous-taper anchors, not a fixed two-point model; see §2.2):

```
force: float<newtons>
  Dimensional range: [1e-11, 1e30] (from gravitational constant and stellar masses; Tier 2)
  ├── x86_64:  float64         (worst-case rel error: 1.11e-16, uniform)      [WideDynamic → IEEE]
  ├── xilinx:  posit<32, es=2> (~2.3e-8 at range extremes, ~1.5e-9 near 1.0)  [NearUnityTaper]
  └── Note: posit gives ~10x better precision in [0.01, 100] where most forces reside
```

```
Error CCS8012: the declared posit<32, es=2> does not cover
  the full range [1e-11, 1e72] of astronomicalDistance<meters>
  Change the boundary to an offered covering representation, such as IEEE f64,
  or establish a valid tighter range before this boundary.
```

A rescaling suggestion is valid only if the compiler establishes the transformed range and checks coverage again. Changing units to AU does not by itself make the illustrated range fit a posit. New diagnostic codes form a numeric-selection family (peers of the width-inference and FPGA codes) and SHALL include at least: **coverage-empty** (`R_cov = ∅`), **near-zero degeneracy** (a range straddling zero with no representation resolving it under the ULP floor), **bare-source-unbounded-at-the-dimensioning-seam** (§6.1), **tier disagreement** (`R_lower ⊄ R_binding`, §3.4), **suboptimal boundary representation** (§5), and **quire capability failure** (§10.2).

## 12. Relationship to Other Features

- **Width Inference** — numeric selection is the **real-valued counterpart**: width inference sizes integers from their value range; numeric selection chooses the representation of reals from their dimensional range, via the same coeffect frame. The unified objective is stated in both chapters; see [Width Inference §4](width-inference.md).
- **Dimensional Type System** — the dimensional range is the principal input; see [Units of Measure](units-of-measure.md) and [NTU Dimensional Architecture](ntu-dimensional-architecture.md). The dimension establishes the *kind*; the range establishes the *representation*.
- **Native Type Universe** — the source has one real kind, `float`; IEEE `f32` and `f64` are representation choices. The unobservable bare-float exception is specified in §6; see [NTU Types](ntu-types.md).
- **Incremental Computation** — the tiered authority model reuses the Level-1/2/3 inference hierarchy; see [Incremental Computation §12](incremental-computation.md).
- **Negative types and reversibility** *(non-normative)* — a typed reversibility contract (negative types) is **orthogonal** to representation selection: it checks *structurally* and is representation-agnostic, so IEEE and posit alike satisfy it. Representation selection makes no promise of numerical reversibility; it provisions the precision envelope within which a reversible computation's residual stays bounded, and the quire's exact accumulation extends that horizon. A symplectic integrator typed via negative types and lowered with the quire is a client of both disciplines; that composition lives in a `Fidelity.Numerics` library, not in this chapter.

## 13. Normative Requirements

1. **Range-driven representation.** For a real-valued quantity, the representation (IEEE-754 / posit / fixed-point) SHALL be selected per target as a function of the analyzed and dimensional range, not from a target type name.
2. **Feasibility constraint.** Selection SHALL be over the coverage-filtered candidate set `R_cov`; if it is empty, the implementation SHALL emit a hard coverage error (CCS8012) identifying the uncovered range and offered representations, and SHALL NOT select a non-covering representation.
3. **Zero-crossing soundness.** The error metric SHALL be the ULP-floored form of §2.2 (or its equivalent neighborhood-exclusion form); a range straddling zero SHALL NOT yield an undefined argmin.
4. **Accuracy-only objective.** The selection score SHALL be worst-case error over the range and SHALL NOT include a cost, latency, or area term; performance SHALL be expressed only as a candidate-set filter (§7).
5. **Tier composition.** When multiple tiers produce a range claim, the binding range SHALL be the claim of the highest present tier; lower-tier claims SHALL become consistency obligations whose violation is a hard error preventing the selection from being committed, and never a silent change to the binding range. There SHALL be no empty-intersection state.
6. **Unobservable range, dimensioned.** An unresolved range obligation MAY remain pending during elaboration. If a dimensioned real's required range is still unresolved at concrete representation commitment, the implementation SHALL emit a located error tracing the missing provenance and SHALL NOT assume a representation ([Width Inference §6](width-inference.md#6-unobservable-ranges)).
7. **Unobservable range, bare.** A bare `float` with an unobservable range SHALL use IEEE `f64` under §6's exception when the target offers it and capability policy permits it; otherwise the missing capability SHALL be diagnosed before commitment. This exception SHALL NOT waive coverage of a known range. Bare reals SHALL still carry range propagation.
8. **Dimensioning seam.** When an unresolved bare value flows into a dimensioned context, the range obligation SHALL attach to the dimensioning site. If still unresolved at representation commitment, its diagnostic SHALL identify that origin and the upstream bare source.
9. **Boundaries.** A declared boundary representation SHALL be honored when it covers the range; a covering-but-suboptimal declaration SHALL compile and SHOULD be witnessed at design time with the representation the open argmin would have selected; a non-covering one SHALL be the hard coverage error of requirement 2, never a conversion.
10. **Coeffect carriage.** The selected representation SHALL be recorded as a coeffect (codata) on the PSG beside the dimension and preserved through lowering without recomputation.
11. **Capability gating.** A representation unavailable on the target SHALL NOT be selected; an accuracy-optimal but emulated choice under an emulation-permitting policy SHALL be selected and MAY be accompanied by a performance diagnostic; a required exact-accumulation capability that the target lacks SHALL be a capability failure, never a silent fallback to lossy accumulation.
12. **Quire adequacy.** A recognized accumulation over a selected posit SHALL use the format's fixed quire width: 800 bits for a b-posit of width `n > 12` (arXiv:2603.01615), or `16n` bits for a full-gamut posit of width `n` (Posit Standard 2022). The compiler SHALL establish exact per-product representation and coverage of every reachable partial sum before committing exact accumulation (§10.2.1). Missing or inconsistent capacity evidence SHALL be diagnosed, never replaced by an unlimited-capacity assumption.

## 14. Genuinely-Open Items

> **Not yet specified.** The following are open and SHALL be resolved before the corresponding capability is claimed shippable. None is invented here as settled.

1. **Lossy-conversion discipline syntax.** Closed 2026-09-04: there is no conversion and no seal form, and an intended loss is arithmetic with an analysed range ([Width Inference §7](width-inference.md); `Dimensional_Range_Design.md`). Not open.
2. **`Fidelity.Physics` binding contract.** The export interface that composes a quotation-symbolic module (§4) into a Tier-2 range claim is unspecified, and the library is planned, not built. The terminating range-expression sub-language is now **resolved in §4**: it is a closed-form image computation in the interval domain (§9.1) that feeds the regime classifier, so the carried object is a categorical regime (a point in the bounded representation lattice), not a scalar, and downstream obligations are finite lattice operations over that categorical codomain. The **residual gaps are (a) the per-transcendental segment-splitting convention** (where exactly `sin` and other non-monotone functions are split into monotone pieces) and **(b) the tightness target for the bound method** — since interval over-estimation under the dependency problem (`r·r`) bears on *which* lattice element is selected (selection completeness), it must be pinned how tight the bound has to be, and whether affine/Taylor refinement is required when an over-estimate would straddle a regime boundary.
3. **Measurement and rounding uncertainty.** The provenance and per-domain specification of uncertainty allowances remain open. Any admitted allowance SHALL be reflected in sound bounds before consistency and coverage are checked; it SHALL NOT permit an observed value outside the committed representation's range ([Conformance §5](conformance.md#5-the-diagnostic-obligation)). No particular uncertainty model or inference algorithm is established here.
4. **Profiling-evidence provenance.** The third range source has no recording, versioning, or cross-build trust model; whether a profiling range is Tier 1 (with an evidence-provenance tag) or its own tier is unresolved.
5. **Asymmetric/biased ML objective.** **Resolved in §4 toward the shifted-range approach:** ML selection stays on §2's symmetric worst-case argmin, run over a range pre-skewed to the activation density by the `Fidelity.ML` library, so the general selector remains single-objective and exact (Requirement 4). What remains open is only the *optional* future refinement — a genuinely distribution-*weighted* (expected-error) objective that integrates error against an activation density. This is the one place an expectation term might legitimately re-enter; it is scoped strictly to the ML domain library, never to the general selector, and is not required for the shipped behavior.
6. **Quire sizing-pass obligation.** The fixed quire *width* per format is settled (800 bits for a b-posit; `16n` for a full-gamut posit); the product and partial-sum verification, allocation, and coeffect carriage (§10.2.1) is downstream and must be fully defined.
7. **Type-directed posit hardware synthesis.** The FPGA parameterized-config search (§8, layer 3) is future work; the bias/asymmetry portion of the search is bounded-but-continuous, not enumerable, and the synthesis pipeline is not built.
8. **Error precedence.** The ordering between `R_eff = ∅` (no native-policy-permitted format) and `R_cov = ∅` (no covering format) when a target both lacks native support and offers no covering format needs a stated precedence.
9. **ULP-floor definition.** The exact `ulp_min(r)` per representation family (IEEE subnormal floor vs. posit smallest-regime magnitude vs. fixed-point LSB), and whether the ULP-floor form or the `[−δ, +δ]` exclusion form is canonical, must be pinned. Normative once chosen.

## References

- Swamy, N., et al. *F\* — refinement-typed machine integers and verified low-level code.* https://www.fstar-lang.org
- Kennedy, A. (1996). *Programming Languages and Dimensions.* PhD thesis, University of Cambridge. (Units of measure; later realized in F#.)
- Syme, D. (2006). Leveraging .NET Meta-programming Components from F#. *ML Workshop '06.* (Quotations.)
- Syme, D., Neverov, G., & Margetson, J. (2007). Extensible Pattern Matching via a Lightweight Language Extension. *ICFP '07.* (Active patterns.)
- Gustafson, J. L. (2017). *Posit Arithmetic.* (Tapered-precision real representation.)
- *Standard for Posit Arithmetic* (2022). (Quire, `16n` accumulator width.)
- Jonnalagadda, A. A., Thotli, R., & Gustafson, J. L. (2026). *Closing the Gap Between Float and Posit Hardware Efficiency.* arXiv:2603.01615. (Bounded-regime b-posit; a 6-bit regime cap and five-way decode MUX shared across widths; fixed 800-bit quire; the source for this chapter's b-posit hardware figures and quire sizing.)
- Cousot, P., & Cousot, R. (1977). Abstract Interpretation: A Unified Lattice Model for Static Analysis of Programs by Construction or Approximation of Fixpoints. *POPL '77.* (Interval domains, widening.)
- Moore, R. E. (1966). *Interval Analysis.* Prentice-Hall. (Outward-rounded interval arithmetic.)
- Petricek, T., Orchard, D., & Mycroft, A. (2014). Coeffects: A Calculus of Context-Dependent Computation. *ICFP '14.*
- Baaij, C., et al. *CλaSH: structural descriptions of synchronous hardware (Haskell→FPGA).* (Sizing hardware to inferred representations.)
- IEEE Standard for Floating-Point Arithmetic, IEEE 754-2019.
