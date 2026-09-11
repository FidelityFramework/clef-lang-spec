---
title: "Rounding and Directed Rounding"
weight: 450
category: Semantics
status: normative
---

> **Normative specification for how a rounding discipline is chosen, carried, and enforced once representation is not necessarily fixed by the platform. Rounding is the third panel of the numeric triptych: [Width Inference](width-inference.md) sizes an integer from its range, [Numeric Selection](numeric-selection.md) chooses a real's representation from its range, and this chapter specifies the rounding that representation applies — both when a value crosses a representation boundary and when an operation must commit a rounding direction for soundness.**

## 1. Overview

Rounding is invisible in conventional practice because IEEE 754 made it universal and implicit: round-to-nearest-ties-to-even, the same on every mainstream processor, baked into the hardware, so a program never names it. The framework's own thesis removes that assumption. Once representation follows from analyzed range rather than from a platform default — posit on one target, fixed-point on another, IEEE on a third, an interval enclosure on a fourth — the rounding discipline is no longer carried for the program by the platform, because the representations do not share one. Rounding therefore becomes an explicit property, selected and carried the way width and representation already are, and this chapter specifies it.

Rounding enters the framework in **two distinct cases**, and the distinction is load-bearing because the two cases carry differently (§3):

- **Rounding at a representation boundary.** A value changes representation — a narrowing conversion, a Tier-3 seal's lossy cast, a `quire → posit` conversion, a cross-target transfer. Precision is lost, and the *discipline* (which way to round, whether to saturate) governs how much and in which direction. Here a wrong choice is an **inaccuracy**: the result is less precise than it could be, but it is still a number.
- **Rounding within an operation under a fixed representation.** An operation produces a value and must commit a rounding direction. For most representations this is round-to-nearest and unremarkable. For a **sound interval enclosure** it is not: the lower endpoint must round toward `−∞` and the upper toward `+∞` at *every* operation, or the enclosure ceases to contain the true value. Here a wrong choice is a **falsehood**: the value claims to bound a result it does not bound.

This chapter closes the rounding-and-saturation items left open in [Width Inference §7](width-inference.md) and [Numeric Selection §5, §14](numeric-selection.md); those chapters name the requirement that a lossy conversion specify a discipline and defer the discipline itself to here.

### 1.1 Status discipline

As in [Numeric Selection §1.1](numeric-selection.md), requirements that follow from prior art or external standards are stated normatively without qualification; requirements this specification *adopts* as a design construction are marked **[Design decision]**; genuinely unresolved items are marked **[Not yet specified]**.

## 2. The Rounding Modes

The available rounding modes are a property of the representation family, and the families differ sharply in what they offer. This asymmetry is the reason rounding cannot be assumed.

- **IEEE-754** (per IEEE 754-2019) defines five rounding-direction attributes: *roundTiesToEven* (the default), *roundTiesToAway*, *roundTowardPositive* (toward `+∞`), *roundTowardNegative* (toward `−∞`), and *roundTowardZero* (truncation). The two directed modes (toward `±∞`) are what sound interval arithmetic requires.
- **Posit** (per the Posit Standard 2022) defines **exactly one** rounding mode: round-to-nearest-ties-to-even. **It defines no directed rounding.** This is a normative fact about the representation, not an omission to be filled, and it has a direct consequence (§4.2): a sound interval *over posits* cannot be produced by the posit's own arithmetic and must be synthesized by outward ULP widening.
- **Fixed-point** rounds at the least-significant retained bit, with the standard disciplines — toward `+∞` (ceiling), toward `−∞` (floor/truncate), round-to-nearest, and convergent (round-to-nearest-even) — and, on overflow, either **saturates** (clamps to the representable extreme) or **wraps** (modular). Saturation and wrap are a separate axis from rounding direction and SHALL be specified independently (§5).

> **[Not yet specified] — `ulp_min(r)`.** The exact smallest-magnitude floor per representation family (the IEEE subnormal floor `2^emin`, the posit smallest-regime magnitude, the fixed-point LSB `2^(−frac_bits)`) is the same open item as [Numeric Selection §14.10](numeric-selection.md). It is named here because it is the quantity directed rounding and the zero-crossing metric both depend on; it is normative once pinned.

## 3. Carriage: Identity for Soundness, Coeffect for Loss

**[Design decision.]** The two cases of §1 are carried by two different mechanisms, split on whether a wrong rounding produces a falsehood or an inaccuracy.

### 3.1 Sound-critical directed rounding is part of representation identity

When the *soundness* of a value depends on its rounding direction — the interval endpoint case — the rounding direction SHALL be **part of the representation's type identity**, not a fact layered beside it. An interval enclosure is sound only if its lower endpoint is computed toward `−∞` and its upper toward `+∞` at every operation (per IEEE 1788-2015); a value carrying the wrong direction is not a less-accurate enclosure, it is a non-enclosure. The type system SHALL therefore treat the directed-rounding capability as a requirement on the representation parameter, checked at unification:

```
Interval<r>  is well-formed only if  r  supplies directed rounding
```

so that an `Interval<r>` SHALL NOT be well-formed over a representation that cannot round outward, discharging the obligation where the type is formed rather than deferring it to runtime. This is the type-level enforcement that an `invalidArg` runtime check (the conventional approach) pushes to execution; the ill-formed enclosure is rejected at unification rather than at a runtime guard.

### 3.2 Lossy-conversion rounding is a coeffect discipline

When a wrong rounding produces only an *inaccuracy* — the representation-boundary case — the rounding discipline SHALL be carried as a **[coeffect on the Program Semantic Graph](program-semantic-graph.md)**, codata beside the representation, dimension, grade, and escape class, in the same frame [Numeric Selection §9–§10](numeric-selection.md) uses for representation itself. The discipline is recorded at the conversion site, preserved through lowering without recomputation (certified passes preserve it by construction; uncertified passes receive a per-edge re-check), and surfaced at design time as part of the conversion's fidelity (§6). A lossy conversion whose discipline is unspecified is the open-syntax item of §6, not a silent default.

### 3.3 The seam is the chapter's organizing principle

The split is total: every rounding obligation in the framework is either soundness-critical (identity, §3.1) or accuracy-affecting (coeffect, §3.2). The quire (§4) sits cleanly on the coeffect side — its single final rounding is a precision discipline, not an enclosure obligation — and the interval enclosure sits on the identity side. No obligation is both, and none is neither.

## 4. The Quire's Single-Rounding Discipline

The quire is the framework's canonical example of rounding *deferred* rather than *directed*, and its discipline is already normative in [Numeric Selection §10.2](numeric-selection.md); this section states the rounding semantics that section's `Quire.toPosit` depends on.

An accumulation lowered to a quire performs **no rounding at any intermediate step**. The fixed-width accumulator (800 bits for a b-posit of width `n > 12`, per arXiv:2603.01615; `n²/2` bits for a full-gamut posit of width `n`, per the Posit Standard 2022) holds every partial product exactly, and rounding occurs **exactly once**, at the final `quire → posit` conversion. The single rounding at conversion SHALL use the posit's only defined mode, round-to-nearest-ties-to-even (§2); no directed mode applies, because none exists for posits. The dimension is preserved through the accumulation and verified at the conversion output (an `fma` of `newtons × meters` accumulates as `joules`, rounded once to a `Posit32<joules>`).

This single-rounding discipline is the mechanism behind the quire's two stated benefits (per [Numeric Selection §10.2.2](numeric-selection.md), and the dts-dmm and decidable-by-construction pre-prints): exact accumulation keeps the structural zeros of Clifford/Cayley algebra structurally zero, and it defeats catastrophic cancellation in a difference or sum of nearly-equal quantities by deferring all rounding past the cancellation. Both follow from *one rounding instead of `k`*, and neither is a claim about near-zero precision, which posits do not have.

### 4.1 The quire is a coeffect, not an identity

The quire's rounding rides §3.2: it is a precision discipline carried as the existing quire capability coeffect, not a soundness obligation on a type parameter. A target that cannot accumulate exactly triggers the capability failure of [Numeric Selection §10.2](numeric-selection.md), never a silent fallback to per-step rounding.

### 4.2 Interval over posit: synthesis, not native directed rounding

Because posits define no directed rounding (§2), a sound `Interval<Posit32>` SHALL NOT be claimed to round outward natively. Its endpoints SHALL be produced by the outward ULP-widening construction (Moore's method): compute each endpoint round-to-nearest, then widen the lower endpoint downward and the upper endpoint upward by one ULP at `ulp_min` (§2). The result is a sound but slightly loose enclosure. This is the honest discharge of the §3.1 identity requirement on a representation whose hardware cannot round outward, and it SHALL be distinguished in diagnostics from a representation that rounds outward natively (§7).

### 4.3 Directed rounding threads through a multi-step operation

**[Design decision.]** A nonlinear interval operation does not compute each output endpoint from one fixed input endpoint the way addition does; it is a **sign-case analysis** whose output corner depends on the signs of the operands. Interval multiplication is the canonical case: each endpoint is a reduction over the four corner products,

```
lo = min(aLo·bLo, aLo·bHi, aHi·bLo, aHi·bHi)
hi = max(aLo·bLo, aLo·bHi, aHi·bLo, aHi·bHi)
```

and which product wins depends on whether each operand straddles zero. The sign-crossing reciprocal of [Numeric Selection §9.1](numeric-selection.md) is the same phenomenon in the case that produces *unbounded* pieces; multiplication is the bounded case of one general rule.

The directed-rounding obligation of §3.1 therefore SHALL thread through each intermediate, **rounded before the reduction, in the output endpoint's direction**: every corner product feeding `lo` SHALL be computed toward `−∞`, and every corner product feeding `hi` toward `+∞`, *before* the `min`/`max`. Taking the reduction first and rounding the result afterward is unsound — it may round the winning product the wrong way by one ULP and break the enclosure. The ordering, not merely the final direction, is the requirement.

> **Dependency note.** Because the corners are evaluated as if the operands were independent, an operation where a variable recurs over-estimates: `r·r` over `[a, b]` yields `[min(a², ab, b²), max(...)]`, which includes negatives when `[a, b]` straddles zero, where the true range of `r²` is `[0, max(a², b²)]`. This is the dependency problem named in [Numeric Selection §4](numeric-selection.md); a dedicated squaring operation, not generic multiplication, is the sound treatment for `r²`. The over-estimate is *sound* (it only ever widens the enclosure), but it bears on tightness.

## 5. Saturation and Wrap

**[Design decision.]** Overflow handling at a representation boundary is a separate axis from rounding direction and SHALL be specified independently of it. Two disciplines are defined:

- **Saturation** — a value exceeding the target's representable range is clamped to the nearest representable extreme.
- **Wrap** — a value exceeding the range is reduced modulo the range.

Per [Width Inference §7](width-inference.md), a value whose analysed range a boundary's declared representation does not cover is a coverage diagnostic, not a candidate for either discipline, and arithmetic on analysed ranges never leaves its selected representation. Saturation and wrap are therefore facts the platform description declares about each of its representations ([Platform Bindings](platform-bindings.md)), read by the compiler to realise `clamp` and `%` natively where the hardware does them; they are never named at a site, and the compiler never chooses one. Saturation is the recommended default for dimensioned quantities, because a clamped physical value is a bounded error whereas a wrapped one is an unbounded one; the default is **[Design decision]** and stated as a SHOULD in §10.

## 6. No Conversion Form

There is no explicit conversion in Clef and no seal form ([Width Inference §7](width-inference.md), [Numeric Selection §3.3, §5](numeric-selection.md)). The three things §6 once required a conversion to carry are carried elsewhere:

1. the **target representation** is the boundary's declaration, or the representation selection made from the value's range;
2. the **rounding mode** at a boundary between two real representations is the mode the boundary's declared representation offers, read from the platform description (§7), and within an operation it is §3;
3. the **overflow discipline** is the platform's declared boundary semantics for the representation (§5), read to realise `%` and `clamp`, never chosen by the compiler and never written at a site.

An intended loss is written as arithmetic, `x % 2^n`, `clamp lo hi x`, `floor`, `ceiling`, `round`, `truncate`, whose result has an analysed range; the loss is visible at the site because the site is that function, and the ranges into and out of it are on the graph.

## 7. Capability and Design-Time Surfacing

**[Design decision.]** A target's support for a required rounding mode is **three-valued** — *native*, *emulated*, or *unavailable* — the same capability gate [Numeric Selection §7](numeric-selection.md) applies to representations, applied here to rounding modes. The per-target support facts require their own declaration and consumer; the implemented static MMIO fragment of [Platform Predicates](platform-predicates.md) does not supply a rounding-capability resolver. The rule, not the per-target realization, is normative here:

- A rounding mode a target supports in hardware is *native*.
- A rounding mode a target can realize only by additional work (a CPU switching its global rounding-mode register between operations; a posit interval synthesized by ULP widening, §4.2) is *emulated*, and its cost MAY raise a design-time diagnostic under an emulation-warning policy.
- A required rounding mode a target cannot realize soundly at all is *unavailable*, and a value whose soundness requires it SHALL raise a capability failure — never a silent substitution of a different mode.

The asymmetry that makes this gate three-valued rather than a boolean is real and target-structural: on a CPU, rounding direction is a **global dynamic mode** (the x86 `MXCSR` and ARM `FPCR` rounding-control fields) that affects all subsequent operations until changed, so interval arithmetic with interleaved endpoints must either reorder all lower-bound computations ahead of all upper-bound computations or pay a substantial mode-switching penalty — directed rounding on a CPU is *emulated* in the cost sense even though the modes exist. On an FPGA, rounding direction is **combinational datapath logic fixed at synthesis**, so a chosen direction costs nothing at runtime — directed rounding is *native* and free. The same `Interval<r>` is therefore a comfortable native construct on fabric and an expensive emulated one on a host, and the gate records which without changing the source.

The per-target realization of each rounding mode — how an FPGA bakes a direction into the datapath, how a CPU manages its mode register, how fixed-point rounding lowers, and the cost notes for each — is informative and is documented in the companion docs page, [Rounding on Real Hardware](/docs/design/types/rounding-on-real-hardware/), which this chapter forward-links rather than restates.

Numeric selection's diagnostic family (per [Numeric Selection §11](numeric-selection.md)) is extended with rounding codes: **rounding-unavailable** (a required mode the target cannot realize soundly), **rounding-emulated** (the chosen mode is realized at a cost worth surfacing), and **interval-over-non-directed-representation** (a sound enclosure synthesized by ULP widening, §4.2, rather than by native directed rounding).

## 8. Relationship to Other Features

- **Width Inference** — this chapter supplies the rounding and saturation discipline that [Width Inference §7](width-inference.md) requires of an explicit conversion and leaves open.
- **Numeric Selection** — this chapter specifies the rounding semantics that [Numeric Selection §5 (seal), §10.2 (`Quire.toPosit`)](numeric-selection.md) depend on, and shares its capability-gate and design-time-surfacing machinery.
- **Native Type Universe** — the directed-rounding requirement on `Interval<r>` (§3.1) is a constraint on a representation parameter, formed in the type machinery of [Native Type Universe](native-type-universe.md); it is not a new kind of type, but an intrinsic constraint authored on an existing one.
- **Negative types and reversibility** *(non-normative)* — rounding is orthogonal to the typed reversibility contract: a reversibility type checks structurally and is representation-agnostic, while rounding governs the numerical residual the reversal accumulates. The quire's single rounding (§4) extends the residual horizon; it does not make a reversal exact.

## 9. The Two Cases, Restated

| | Boundary rounding (§1, §4, §5, §6) | Operation rounding (§1, §3.1) |
|---|---|---|
| When | a value changes representation | an operation produces a value |
| Wrong choice is | an **inaccuracy** | a **falsehood** (for enclosures) or unremarkable (otherwise) |
| Carried as | a **coeffect** (§3.2) | **representation identity** (§3.1) for sound-critical cases |
| Example | narrowing cast, seal, `quire → posit`, transfer | interval endpoints toward `∓∞` |
| Open syntax | the conversion/seal form (§6) | — |

## 10. Normative Requirements

1. **No conversion form.** There SHALL be no explicit conversion and no seal form; an intended loss SHALL be written as arithmetic with an analysed range ([Width Inference §7](width-inference.md)).
2. **No silent narrowing.** A value SHALL NOT be truncated, wrapped or saturated by the compiler; a range a boundary does not cover SHALL be a coverage diagnostic, and the platform's declared boundary semantics SHALL be read only to realise the program's own `%` and `clamp`.
3. **Directed rounding is representation identity.** Where a value's soundness depends on rounding direction (an interval enclosure), the directed-rounding capability SHALL be a requirement on the representation parameter, checked at unification; an enclosure type over a representation that cannot round outward SHALL NOT be well-formed. In a nonlinear interval operation (§4.3), each intermediate SHALL be rounded in the output endpoint's direction *before* the `min`/`max` reduction; rounding the reduced result afterward is unsound.
4. **Lossy rounding is a coeffect.** Where a rounding choice affects only accuracy, the discipline SHALL be carried as a coeffect on the PSG and preserved through lowering without recomputation.
5. **Quire single rounding.** A quire accumulation SHALL round exactly once, at the final `quire → posit` conversion, using round-to-nearest-ties-to-even; it SHALL NOT round at any intermediate step. A target lacking exact accumulation SHALL raise a capability failure, never a silent per-step-rounding fallback.
6. **Interval over non-directed representations.** A sound enclosure over a representation that defines no directed rounding (posits) SHALL be produced by outward ULP widening, SHALL be sound, and SHALL be distinguished in diagnostics from native directed rounding.
7. **Three-valued rounding capability.** A required rounding mode SHALL be gated as native, emulated, or unavailable per target; an unavailable mode whose soundness a value requires SHALL be a capability failure, never a silent substitution of a different mode.
8. **No default discipline.** The compiler SHALL NOT default to saturation or to wrap; a boundary a range does not cover is a diagnosed finding, and the program's own `clamp` or `%` is the only source of either behaviour.

## 11. Genuinely-Open Items

> **[Not yet specified].** None invented here as settled.

1. **Conversion and seal surface syntax (§6).** Closed 2026-09-04: there is none ([Width Inference §7](width-inference.md)).
2. **`ulp_min(r)` per representation family (§2).** The IEEE subnormal floor, the posit smallest-regime magnitude, and the fixed-point LSB; whether the ULP-floor form or the `[−δ, +δ]` exclusion form is canonical. Shared with [Numeric Selection §14.10](numeric-selection.md); normative once chosen.
3. **Overflow-discipline default (§5).** Closed 2026-09-04: the compiler defaults to nothing; the program's arithmetic states its intent and the platform declares what its hardware does.
4. **Cascade rounding.** Error tracking across a chain of conversions (e.g. `posit32 → f64 → fixed`) — whether the carried fidelity coeffect composes the per-stage losses or records them separately — is unspecified.
5. **Selectable quire conversion mode.** Whether the final `quire → posit` rounding is fixed at round-to-nearest or may be directed for a quire feeding an interval endpoint (§4.2) is open.

## References

- *IEEE Standard for Floating-Point Arithmetic*, IEEE 754-2019. (Five rounding-direction attributes; the directed modes toward `±∞`.)
- *IEEE Standard for Interval Arithmetic*, IEEE 1788-2015. (Sound enclosure requires the lower endpoint toward `−∞` and the upper toward `+∞` at every operation.)
- *Standard for Posit Arithmetic* (2022). (Round-to-nearest-ties-to-even as the sole posit rounding mode; the full-gamut quire and its `n²/2` width.)
- Jonnalagadda, A. A., Thotli, R., & Gustafson, J. L. (2026). *Closing the Gap Between Float and Posit Hardware Efficiency.* arXiv:2603.01615. (The bounded-regime b-posit and its fixed 800-bit quire.)
- Moore, R. E. (1966). *Interval Analysis.* Prentice-Hall. (Outward rounding / ULP widening for a sound enclosure when native directed rounding is unavailable.)
- Gustafson, J. L. (2017). *Posit Arithmetic.* (Tapered precision; the quire's single final rounding.)
- Intel® 64 and IA-32 Architectures Software Developer's Manual; Arm® Architecture Reference Manual. (The CPU rounding-mode control registers `MXCSR` and `FPCR` as global dynamic state — informative, for the §7 asymmetry.)
- Petricek, T., Orchard, D., & Mycroft, A. (2014). Coeffects: A Calculus of Context-Dependent Computation. *ICFP '14.* (The coeffect carriage of §3.2.)
