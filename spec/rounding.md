---
title: "Rounding and Directed Rounding"
weight: 450
category: Semantics
status: normative
---

> **Normative specification for how a rounding discipline is chosen, carried, and enforced once representation is not necessarily fixed by the platform. Rounding is the third panel of the numeric triptych: [Width Inference](width-inference.md) sizes an integer from its range, [Numeric Selection](numeric-selection.md) chooses a real's representation from its range, and this chapter specifies the rounding that representation applies — both when a value crosses a representation boundary and when an operation must commit a rounding direction for soundness.**

## 1. Overview

Rounding is a property of an operation or representation boundary. IEEE arithmetic already supports several directions; a selected format alone does not fix every instruction's rounding, intermediate precision, or exceptional behavior. This chapter specifies how the required discipline is established, carried, and preserved for IEEE, posit, fixed-point, and interval constructions.

Rounding enters the framework in **two distinct cases**, and the distinction is consequential because the two cases carry differently (§3):

- **Rounding at a representation boundary.** Accumulator finalization, fixed-point rescaling, or a cross-target transfer may discard information. The boundary SHALL establish exact representability or carry the permitted rounding and fidelity obligation. A transfer of an interval endpoint also has to preserve its enclosure.
- **Rounding within an operation.** An arithmetic operation may round even when its operands and result share one representation. Its rounding contributes to the numerical contract. An interval construction SHALL preserve an outward enclosure through every contributing operation (§4.3).

Neither location makes rounding inherently harmless. Capacity, error, exactness, and reproducibility remain separate obligations under [Numeric Selection §10.5](numeric-selection.md#105-capacity-error-and-decomposition-obligations). These requirements introduce no source conversion or seal syntax (§6).

### 1.1 Status discipline

As in [Numeric Selection §1.1](numeric-selection.md), requirements that follow from prior art or external standards are stated normatively without qualification; requirements this specification *adopts* as a design construction are marked **[Design decision]**; genuinely unresolved items are marked **[Not yet specified]**.

## 2. The Rounding Modes

The available rounding modes are a property of the representation family, and the families differ sharply in what they offer. This asymmetry is the reason rounding cannot be assumed.

- **IEEE-754** (per IEEE 754-2019) defines five rounding-direction attributes: *roundTiesToEven* (the default), *roundTiesToAway*, *roundTowardPositive* (toward `+∞`), *roundTowardNegative* (toward `−∞`), and *roundTowardZero* (truncation). The two directed modes (toward `±∞`) are what sound interval arithmetic requires.
- **Posit** (per the Posit Standard 2022) defines one format-specific rounding algorithm, including its encoding thresholds and extreme-value behavior, rather than selectable directed modes. A construction SHALL follow the complete declared rule; a nearest-even summary SHALL NOT replace that rule at tapered boundaries. Sound outward rounding over posit endpoints requires an additional construction (§4.2).
- **Fixed-point** uses a declared scale. Discarding fractional information requires a specified rule, such as toward `+∞`, toward `−∞`, toward zero, or nearest with a stated tie rule. Floor and truncation toward zero differ for negative values. Capacity, rescaling, and error are separate obligations (§6.1); neither wrapping nor saturation is an automatic fallback (§5).

> **[Not yet specified] — selection error floor.** The canonical floor in the selection metric remains open in [Numeric Selection §14](numeric-selection.md#14-genuinely-open-items). This does not make a declared format's spacing unknown: for binary IEEE precision `p` and minimum normal exponent `emin`, gradual-underflow spacing is `2^(emin-(p-1))`; binary fixed-point spacing is `2^(-f)`. A metric floor SHALL NOT substitute for the local spacing or error bound needed by an outward-rounding construction.

## 3. Carriage: Identity for Soundness, Coeffect for Loss

**[Design decision.]** Sound enclosure requirements constrain the admitted representation and construction; rounding and error evidence travel with the operation or boundary. These obligations can apply together at either location described in §1.

### 3.1 Sound-critical directed rounding is part of representation identity

When soundness requires an enclosure, the outward-rounding capability SHALL constrain the representation parameter. Its construction SHALL establish lower and upper bounds on the exact result through the operations and transfers that contribute to those endpoints (§4.2–§4.3). The capability requirement is part of representation identity and is checked with the applicable type constraints:

```
Interval<r>  is well-formed only if  r  supplies directed rounding
```

Here supplying directed rounding means an established outward-rounding realization, either native or synthesized (§4.2, §7). An `Interval<r>` SHALL NOT commit to a representation and construction that cannot preserve enclosure. The capability requirement follows the pending-obligation discipline of [Numeric Selection §10.5.2](numeric-selection.md#1052-automatic-analysis-and-commitment); an unknown target fact is not proof of unavailable capability.

### 3.2 Lossy-conversion rounding is a coeffect discipline

The required rounding discipline and its error evidence SHALL be carried as a **[coeffect on the Program Semantic Graph](program-semantic-graph.md)** beside the selected representation and dimension. The evidence SHALL identify the operation or boundary, reference quantity, rounding points, assumptions, and justified error bound where the contract requires one. It SHALL be preserved or re-established through lowering under [Conformance §6](conformance.md#6-the-preservation-obligation-through-lowering). An unspecified required discipline SHALL remain pending and be diagnosed before commitment, rather than receive a silent default.

### 3.3 The seam is the chapter's organizing principle

An interval endpoint can require both a sound enclosure and a bound on enclosure width. A quire finalization can require a format-defined result and, when feeding an interval, a separate outward bound. The carriage mechanisms SHALL preserve both requirements when applicable. Neither capacity nor a reproducibility claim SHALL discharge them.

## 4. The Quire's Single-Rounding Discipline

The quire defers accumulation rounding under the contract of [Numeric Selection §10.2](numeric-selection.md). Its finalization is an internal construction boundary, not a source conversion API.

An authorized exact accumulation SHALL perform no intermediate rounding. Its capacity SHALL cover every represented product, partial sum, and merge allowed by the construction. The standard quire has `16n` bits for posit width `n`; the bounded b-posit proposal specifies 800 bits for `n > 12`. Neither width implies unlimited accumulation. Finalization SHALL apply the selected output format's complete rounding rule exactly once. Product and result dimensions SHALL be preserved; products of newtons and meters accumulate as joules.

Exact accumulation preserves cancellation among the represented terms without intermediate rounding. It SHALL NOT be reported as recovery of input information or exactness of algebraic identities throughout a larger computation. Those claims require their own term-formation and error arguments ([Numeric Selection §10.5.1](numeric-selection.md#1051-obligations-by-arithmetic-family)).

### 4.1 The quire is a coeffect, not an identity

The scalar quire's rounding discipline is carried under §3.2. A construction producing an interval must additionally satisfy §3.1. If no permitted realization can provide required exact accumulation, the capability failure of [Numeric Selection §10.2](numeric-selection.md) applies; per-step rounding is not a silent fallback.

### 4.2 Interval over posit: synthesis, not native directed rounding

A sound interval over posit endpoints SHALL use an established outward-rounding construction and SHALL NOT be described as native directed posit arithmetic. A construction using neighbors of a rounded result SHALL establish that those neighbors enclose the exact operation result, using the format's local predecessor/successor rules and boundary behavior. Subtracting or adding one fixed minimum ULP SHALL NOT be assumed sufficient for nonuniform spacing, nor SHALL one-neighbor widening of a compound approximation be assumed to bound all earlier error.

If the exact result exceeds finite endpoint capacity, the construction SHALL provide a supported extended bound or diagnose the coverage/capability failure. NaR SHALL NOT be treated as an ordered infinity. Synthesized enclosure SHALL be distinguished from native directed rounding in diagnostics (§7).

### 4.3 Directed rounding threads through a multi-step operation

**[Design decision.]** A nonlinear interval operation does not compute each output endpoint from one fixed input endpoint the way addition does; it is a **sign-case analysis** whose output corner depends on the signs of the operands. Interval multiplication is the canonical case: each endpoint is a reduction over the four corner products,

```
lo = min(aLo·bLo, aLo·bHi, aHi·bLo, aHi·bHi)
hi = max(aLo·bLo, aLo·bHi, aHi·bLo, aHi·bHi)
```

and which product wins depends on whether each operand straddles zero. The sign-crossing reciprocal of [Numeric Selection §9.1](numeric-selection.md) is the same phenomenon in the case that produces *unbounded* pieces; multiplication is the bounded case of one general rule.

A construction using rounded corner products SHALL establish a downward bound for every product feeding `lo` and an upward bound for every product feeding `hi` before selecting the extrema. An alternative using exact corner products and outward rounding of the exact extrema MAY be used when its intermediate exactness and final enclosure are established. Selecting among nearest-rounded products and changing the rounding mode afterward SHALL NOT be treated as recovery of the discarded information.

> **Dependency note.** Because the corners are evaluated as if the operands were independent, an operation where a variable recurs over-estimates: `r·r` over `[a, b]` yields `[min(a², ab, b²), max(...)]`, which includes negatives when `[a, b]` straddles zero, where the true range of `r²` is `[0, max(a², b²)]`. This is the dependency problem named in [Numeric Selection §4](numeric-selection.md); a dedicated squaring operation, not generic multiplication, is the sound treatment for `r²`. The over-estimate is *sound* (it only ever widens the enclosure), but it bears on tightness.

## 5. Saturation and Wrap

**[Design decision.]** Overflow handling at a representation boundary is a separate axis from rounding direction and SHALL be specified independently of it. Two disciplines are defined:

- **Saturation** — a value exceeding the target's representable range is clamped to the nearest representable extreme.
- **Wrap** — an `n`-bit integer carrier uses reduction modulo `2^n`, with the resulting bits interpreted under its signed or unsigned encoding. Fixed-point interpretation additionally uses the declared scale.

Per [Width Inference §7](width-inference.md), failed coverage is a diagnostic, not permission to choose either discipline. A platform's saturating or wrapping operation MAY realize source `clamp` or modular arithmetic only when equivalence is established, including signedness and scale. Neither discipline is a default for dimensioned quantities. Defined machine behavior does not establish that the source calculation is correct, and saturating arithmetic SHALL NOT be presumed associative for parallel reduction.

## 6. No Conversion Form

There is no explicit conversion in Clef and no seal form ([Width Inference §7](width-inference.md), [Numeric Selection §3.3, §5](numeric-selection.md)). Representation, rounding, and deliberate loss are determined as follows:

1. the **target representation** is the boundary's declaration, or the representation selection made from the value's range;
2. the **rounding mode** at a boundary between two real representations is the mode the boundary's declared representation offers, read from the platform description (§7), and within an operation it is §3;
3. the **overflow discipline** is the platform's declared boundary semantics for the representation (§5), read to realise `%` and `clamp`, never chosen by the compiler and never written at a site.

An intended loss is written as arithmetic, `x % 2^n`, `clamp lo hi x`, `floor`, `ceiling`, `round`, `truncate`, whose result has an analysed range; the loss is visible at the site because the site is that function, and the ranges into and out of it are on the graph.

### 6.1 Fixed-point scale and error

For `x = X · 2^(-f)` and `y = Y · 2^(-g)`, an exact product uses carrier `X · Y` at scale `2^(-(f+g))`. The compiler SHALL establish product capacity before rescaling. A smaller final carrier SHALL NOT justify overflowing or discarding information in an earlier intermediate.

When rescaling to `2^(-h)`, with `d = f+g-h > 0`, exactness requires the product carrier to be divisible by `2^d`. Otherwise the construction SHALL use the rounding prescribed by its contract and record the resulting error. Under nearest rounding with an admitted tie rule and no clipping, one rescaling has absolute error at most `2^(-h)/2`; under directed rounding or truncation its magnitude is less than `2^(-h)`. These local bounds SHALL NOT be presented as bounds for a whole computation without accounting for subsequent propagation and amplification.

Same-scale additions MAY be regrouped as exact integer additions when every permitted partial sum and merge fits. Regrouping products, changing rescaling points, or inserting saturation requires a separate equivalence or permitted-error argument. The compiler SHALL generate these obligations without requiring an opt-in fixed-point safety wrapper ([Numeric Selection §10.5.2](numeric-selection.md#1052-automatic-analysis-and-commitment)).

## 7. Capability and Design-Time Surfacing

**[Design decision.]** A target's support for a required rounding mode is **three-valued** — *native*, *emulated*, or *unavailable* — the same capability gate [Numeric Selection §7](numeric-selection.md) applies to representations, applied here to rounding modes. The per-target support facts require their own declaration and consumer; the implemented static MMIO fragment of [Platform Predicates](platform-predicates.md) does not supply a rounding-capability resolver. The rule, not the per-target realization, is normative here:

- A required operation whose rounding is supported by the selected hardware instruction or circuit is *native*. Control-state setup can still have a cost.
- A required operation synthesized from other operations, such as the posit enclosure construction of §4.2, is *emulated*. Its cost MAY raise a diagnostic under the applicable emulation policy.
- A required rounding mode a target cannot realize soundly at all is *unavailable*, and a value whose soundness requires it SHALL raise a capability failure — never a silent substitution of a different mode.

Capability SHALL be determined for the actual operation, instruction or circuit, and target configuration. CPU implementations may use execution-context rounding state or instruction-specific controls. FPGA implementations may fix a direction in the datapath or provide selectable modes; required rounding logic still consumes resources. Neither a CPU label nor an FPGA label establishes capability or cost. State changes SHALL preserve the required environment across calls, suspension, and resumption; scheduling SHALL NOT change the arithmetic contract. Cost and support evidence are governed by [Numeric Selection §10.4](numeric-selection.md#104-target-eligibility-and-realization-cost).

The per-target realization of each rounding mode — how an FPGA bakes a direction into the datapath, how a CPU manages its mode register, how fixed-point rounding lowers, and the cost notes for each — is informative and is documented in the companion docs page, [Rounding on Real Hardware](/docs/design/types/rounding-on-real-hardware/), which this chapter forward-links rather than restates.

Numeric selection's diagnostic family (per [Numeric Selection §11](numeric-selection.md)) is extended with rounding codes: **rounding-unavailable** (a required mode the target cannot realize soundly), **rounding-emulated** (the chosen mode is realized at a cost worth surfacing), and **interval-over-non-directed-representation** (a sound enclosure synthesized by ULP widening, §4.2, rather than by native directed rounding).

## 8. Relationship to Other Features

- **Width Inference** — integer intermediate capacity supports fixed-point carriers; rounding and rescaling add the obligations of §6.1 to [Width Inference §7.1](width-inference.md#71-encoding-intermediate-capacity-and-modular-arithmetic).
- **Numeric Selection** — this chapter supplies operation and boundary rounding requirements for [Numeric Selection §10.5](numeric-selection.md#105-capacity-error-and-decomposition-obligations), alongside separate capacity, error, and decomposition evidence.
- **Native Type Universe** — the directed-rounding requirement on `Interval<r>` (§3.1) is a constraint on a representation parameter, formed in the type machinery of [Native Type Universe](native-type-universe.md); it is not a new kind of type, but an intrinsic constraint authored on an existing one.
- **Negative types and reversibility** *(non-normative)* — rounding is orthogonal to the typed reversibility contract: a reversibility type checks structurally and is representation-agnostic, while rounding governs the numerical residual the reversal accumulates. The quire's single rounding (§4) extends the residual horizon; it does not make a reversal exact.

## 9. The Two Cases, Restated

| | Boundary rounding (§1, §4, §5, §6) | Operation rounding (§1, §3.1) |
|---|---|---|
| When | a value changes representation | an operation produces a value |
| Required evidence | Exact transfer or permitted loss; enclosure preservation where applicable | Permitted rounding, error, and enclosure where applicable |
| Carried as | Rounding/error coeffects and applicable capability constraints (§3) | Rounding/error coeffects and applicable capability constraints (§3) |
| Example | Fixed-point rescaling, quire finalization, cross-target transfer | Rounded product, directed interval endpoint |
| Source syntax | No conversion or seal form (§6) | Existing operation and its contract |

## 10. Normative Requirements

1. **No conversion form.** There SHALL be no explicit conversion and no seal form; an intended loss SHALL be written as arithmetic with an analysed range ([Width Inference §7](width-inference.md)).
2. **No silent narrowing.** A value SHALL NOT be silently truncated, wrapped or saturated by the compiler; a range a boundary does not cover SHALL be a coverage diagnostic. A physical narrowing SHALL preserve the admitted values or realize loss permitted by the source arithmetic and boundary contract. Platform wrapping or saturation SHALL realize only the program's own modular or clamped arithmetic.
3. **Enclosure capability.** An interval's representation and construction SHALL supply an established outward-rounding realization (§3.1). Enclosure SHALL be preserved through contributing operations and boundaries; rounded-corner and exact-intermediate constructions SHALL meet the respective conditions of §4.3.
4. **Rounding evidence.** Rounding discipline and required error evidence SHALL be carried as coeffects on the PSG and preserved or re-established through lowering (§3.2), alongside any enclosure capability requirement.
5. **Quire single rounding.** An authorized exact quire accumulation SHALL establish product, partial-sum, and merge adequacy, and round only at finalization under the complete selected output rule (§4). No permitted exact realization SHALL be replaced silently by per-step rounding.
6. **Interval over non-directed representations.** A representation lacking native directed rounding SHALL use an established outward construction (§4.2). Fixed minimum-ULP widening SHALL NOT substitute for a format-aware enclosure proof; synthesized and native support SHALL be distinguished.
7. **Three-valued rounding capability.** A required rounding mode SHALL be gated as native, emulated, or unavailable per target; an unavailable mode whose soundness a value requires SHALL be a capability failure, never a silent substitution of a different mode.
8. **No default discipline.** The compiler SHALL NOT default to saturation or to wrap; a boundary a range does not cover is a diagnosed finding, and the program's own `clamp` or `%` is the only source of either behaviour.
9. **Fixed-point fidelity.** Scale alignment, product capacity, rescaling exactness or permitted rounding, and error propagation SHALL be established separately (§6.1). Integer capacity alone SHALL NOT discharge these obligations.
10. **Default checking and preservation.** Applicable rounding obligations SHALL be checked independently of build mode and without opt-in wrappers, under [Numeric Selection §10.5.2](numeric-selection.md#1052-automatic-analysis-and-commitment). Lowering SHALL preserve required rounding points, arithmetic modes, and execution-context state. Optimization permissions SHALL NOT supply their own justification.

## 11. Genuinely-Open Items

> **[Not yet specified].** None invented here as settled.

1. **Conversion and seal surface syntax (§6).** Closed 2026-09-04: there is none ([Width Inference §7](width-inference.md)).
2. **Selection error floor (§2).** The canonical metric-floor definition is shared with [Numeric Selection §14](numeric-selection.md#14-genuinely-open-items). It does not replace a declared format's spacing or a construction's outward-bound proof.
3. **Overflow-discipline default (§5).** Closed 2026-09-04: the compiler defaults to nothing; the program's arithmetic states its intent and the platform declares what its hardware does.
4. **Composed error analysis.** The algorithms and evidence encoding for error propagation across operations, rescaling, and transfers remain unspecified. The requirement to justify a claimed bound and preserve its assumptions is normative (§3.2; Numeric Selection §10.5).
5. **Quire-to-enclosure construction.** The concrete construction for producing outward interval endpoints from a quire remains unspecified (§4.2). It must satisfy enclosure independently of ordinary scalar finalization; this does not introduce a new rounding mode into the Posit Standard.

## References

- *IEEE Standard for Floating-Point Arithmetic*, IEEE 754-2019. (Five rounding-direction attributes; the directed modes toward `±∞`.)
- *IEEE Standard for Interval Arithmetic*, IEEE 1788-2015. (Sound enclosure requires the lower endpoint toward `−∞` and the upper toward `+∞` at every operation.)
- [*Standard for Posit Arithmetic* (2022)](https://posithub.org/docs/posit_standard-2.pdf). (Format-defined rounding; the standard `16n`-bit quire.)
- Jonnalagadda, A. A., Thotli, R., & Gustafson, J. L. (2026). *Closing the Gap Between Float and Posit Hardware Efficiency.* arXiv:2603.01615. (The bounded-regime b-posit and its fixed 800-bit quire.)
- Moore, R. E. (1966). *Interval Analysis.* Prentice-Hall. (Outward rounding / ULP widening for a sound enclosure when native directed rounding is unavailable.)
- Gustafson, J. L. (2017). *Posit Arithmetic.* (Tapered precision; the quire's single final rounding.)
- Intel® 64 and IA-32 Architectures Software Developer's Manual; Arm® Architecture Reference Manual. (Execution-context rounding controls and instruction-specific behavior — informative, for §7.)
- Petricek, T., Orchard, D., & Mycroft, A. (2014). Coeffects: A Calculus of Context-Dependent Computation. *ICFP '14.* (The coeffect carriage of §3.2.)
