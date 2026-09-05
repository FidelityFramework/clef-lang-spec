---
title: "Width Inference"
weight: 430
category: Semantics
status: normative
---

> **Normative specification for value-range analysis, minimal-width derivation, and representation selection in Clef compilation.**

## 1. Overview

Clef does not fix numeric representations by declaration and then coerce between them. It **infers** the representation a value needs — the bit width of an integer, or the posit / IEEE-float / fixed-point form of a real — from the value's **range**, traced through the program's dataflow. Width inference is the spatial half of numeric lowering: *how many bits does each value actually need?*

The discipline is: representation follows from analyzed range, precision loss is explicit and tracked, and an unanalyzable range is a reported error, never a silent default. Signedness is part of this — a value's range determines whether it is signed or unsigned (§3), so a representation is never selected by a target type name independent of the range the value actually occupies.

> **Lineage.** Width inference derives from F\*'s refinement-typed machine integers, where an integer carries its range as a refinement and operations are proven in-bounds. Clef takes the further step of *inferring* the range — and therefore the width — by interval analysis over the [Program Semantic Graph](program-semantic-graph.md), rather than requiring the developer to declare it. The position aligns with Clash (Haskell→FPGA), which sizes hardware to inferred widths rather than to machine-word defaults.

## 2. Value-Range Analysis

Width inference begins with interval analysis over the PSG dataflow. Each numeric node is assigned a value range `[a, b]`:

- **Literals** are point intervals: `124` has range `[124, 124]`.
- **Arithmetic** propagates intervals: `x + y` has range `[a_x + a_y, b_x + b_y]`; products, shifts, and so on follow standard interval arithmetic.
- **Comparisons seed ranges**: a value compared against a `threshold` is bounded by that threshold on the relevant branch, which in turn bounds any counter that must reach it.
- **Recurrences and counters** are bounded by their modulus or termination condition: a free-running counter `mod N` has range `[0, N-1]`.

For example, in a working FPGA design a wave-chase counter is free-running `mod (defaultPeriodMs * ticksPerMs * 2)` ≈ 10⁹, giving range `[0, ~10⁹]`.

## 3. Width Derivation (Integers)

From a range \([a, b]\), the minimal integer width is derived directly:

\[\mathrm{width}([a, b]) = \begin{cases} \lceil \log_2(b + 1) \rceil & a \ge 0 \quad (\text{unsigned}) \\ 1 + \lceil \log_2(\max(|a|,\, b + 1)) \rceil & a < 0 \quad (\text{signed}) \end{cases}\]

A value uses exactly the bits its range requires — no more. The following are inferred widths from a working FPGA design (Arty A7-100T):

| Value | Range | Inferred width |
|---|---|---|
| `Counter` | free-running mod ~10⁹, `[0, 10⁹ − 1]` | 30 bits, unsigned |
| `StepTick` | counts to ~781,250, `[0, 781 250]` | 20 bits, unsigned |
| `Phase` | cycles `0..511` | 9 bits, unsigned |
| `PeriodMs` | latched period, `[0, 4000]` | 12 bits, unsigned |

Each register uses exactly the bits its range requires; on the FPGA each `seq.compreg` flip-flop is narrowed accordingly, shortening carry chains. No width is declared by hand. A non-negative range spends no sign bit: signedness is a fact of the range (§1), and the zero-extension or sign-extension an operand needs when it meets a wider operand follows from that fact, never from a type name. An implementation that adds a sign bit to every range, or extends by a fixed rule, does not conform to this section.

## 4. Representation Selection (Reals)

For real-valued quantities, the analyzed range selects a *representation*, not merely a width. Following the dimensional-type model, representation selection is a deterministic compile-time function of the range and the target's available formats:

\[r^* = \operatorname*{arg\,min}_{r \,\in\, R(\text{target})} \; \max_{x \,\in\, [a, b]} \; \frac{|x - \mathrm{round}_r(x)|}{|x|}\]

— the representation minimizing worst-case relative error over the range. IEEE-754 distributes precision uniformly (≈ \(2^{-p}\)); posits taper precision toward 1.0; fixed-point fixes a scale. The choice is per target (e.g. IEEE-754 on CPU, posit on FPGA, fixed-point on a neuromorphic core) and is surfaced at design time. See [NTU Dimensional Architecture](ntu-dimensional-architecture.md) and [Units of Measure](units-of-measure.md); the dimensional range of a value is the primary input to this function.

The bare argmin above is the shape of the objective, not its sound form: for reals it is ill-posed without two side-conditions — a coverage constraint that excludes representations whose dynamic range does not cover `[a, b]`, and a zero-crossing floor on the error metric where the range straddles zero. The full, sound objective and both side-conditions are specified in [Numeric Selection §2](numeric-selection.md#2-the-selection-objective); this chapter states only the integer-width half of the discipline.

## 5. Width and Representation as a Coeffect

The inferred width/representation is a **coeffect**: a requirement settled during design-time analysis and carried on the PSG, read (not recomputed) by every later lowering pass. The width-inference coeffect holds the range; the lowering pass that fixes the integer representation reads it and writes the chosen width into the operations it produces. This is the same coeffect discipline that governs allocation and lifetime (see [Incremental Computation §10](incremental-computation.md) and the coeffect model), so width inference composes with dimensional resolution and target reachability in a single analysis rather than a separate pass.

## 6. Unobservable Ranges

When interval analysis cannot bound a value's range, the compiler does **not** guess a width or fall back to a machine-word default. It reports an error asking the source for an annotation. This preserves the decidable-by-construction property: every width is either inferred from an observed range or supplied explicitly; none is silently assumed.

## 7. Intended Loss Is Arithmetic

There is no explicit conversion in Clef, because there is nothing to convert between: a value is `int` or `float` at a dimension, its range is analysed, and its width or representation is selected from what the platform declares (§3, §4; [NTU Types](ntu-types.md)). No width-named type exists, no literal suffix names a width, and no operator changes a representation. Where a program intends to lose information it says so in arithmetic, and the result is a value with a known range like any other:

| Intent | Written as | Range |
|---|---|---|
| reduce modulo `2^n` | `x % 2^n` or `x &&& (2^n − 1)` | `[0, 2^n − 1]` |
| saturate to `[lo, hi]` | `clamp lo hi x` (`min hi (max lo x)`) | `[lo, hi]` |
| real to integer | `floor`, `ceiling`, `round`, `truncate` | the integer image of the range |
| integer to real | `float x` | `[a, b]`, exact where the selected real representation holds it |

Because the range of each of these is known, the width follows from it as it does everywhere, and no discipline is ever named at a site: the compiler never selects wrap or saturate, and never inserts a narrowing. A platform's declared boundary semantics for a representation (wrap on a two's-complement unit, saturate on a saturating block, exact on fabric: [Platform Bindings](platform-bindings.md)) are read only to realise `%` and `clamp` cheaply where the hardware does them natively, never to give a program its meaning.

Where a value meets a **boundary**, a representation a declaration fixed rather than the range (a wire-schema field, an MMIO register, a C ABI parameter, an endpoint contract, an exported entry point at the platform's word), the obligation is coverage: the analysed range is contained in the boundary's declared range, else CCS8012, a warning promoted under `--warnaserror`, whose remedies are to bound the value or to change the declaration ([Numeric Selection §5](numeric-selection.md)). Overflow is therefore never undefined behaviour and never a runtime discipline: on analysed ranges it does not occur, and at a boundary it is a design-time finding.

> **Settled.** The surface syntax this section once left open is closed by having none: the representation change and its consequence are visible at the site because the site is a `%`, a `clamp`, a rounding function or a boundary declaration, and the two ranges on the node are what a reader sees (`Dimensional_Range_Design.md`, clef `docs/fidelity/phg`).

## 8. Target Lowering

| Target | Effect of width inference |
|---|---|
| FPGA (e.g. Arty A7-100T) | Registers / `seq.compreg` flip-flops sized to exact inferred width; shorter carry chains, less area. Posit or fixed-point selected per range. |
| CPU | Inferred width rounded up to the nearest native integer size for arithmetic; the analyzed range still drives representation choice and overflow checking. |
| NPU / GPU | Inferred width/representation informs tile/lane packing and accumulator selection. |
| JavaScript (JSIR pathway) | Inferred widths at or below 32 bits realize as host numbers with integer semantics; widths in (32, 53] realize exactly as binary64-backed host numbers; widths above 53 require the pathway's documented wide-integer realization. Reals realize as binary64. |

The JavaScript row rests on the host number model: IEEE-754 binary64, in which every integer of magnitude at most 2⁵³ is exact. Three consequences bind an implementation claiming the **JavaScript Substrate** profile ([Conformance §7](conformance.md)). First, the wide-integer realization for widths above 53 bits (host `BigInt`, or a paired-word emulation) is implementation-defined and SHALL be documented ([Behavior Classification §2](behavior-classification.md)). Second, the no-narrowing clause of requirement 2 binds against that documented realization: a range that exceeds the exact envelope of the chosen realization SHALL be diagnosed under the coverage discipline of [Numeric Selection §2.1](numeric-selection.md), never silently realized in binary64. Third, the wrapping semantics of the fixed-width types ([Native Type Universe §2.3](native-type-universe.md)) SHALL be preserved observably, whatever realization carries them.

Width inference is the *spatial* dimension of hardware lowering. It is necessary but not sufficient: a design may meet its width budget yet still violate timing through combinational depth, which a companion *pipeline* (temporal) inference addresses. The two are distinct analyses; this chapter specifies only the spatial one.

## 9. Relationship to Other Features

- **Dimensional Type System** — the dimensional range of a value is the principal input to representation selection (§4); width inference is the value-level realization of the dimensional discipline.
- **[Native Type Universe](native-type-universe.md)** — inferred widths and representations are NTU representation choices carried through lowering.
- **Conversion** — width inference, together with explicit conversion (§7), specifies numeric conversion in full; the `Convert` and `NTU Conversion` chapters are not part of this specification.

## 10. Normative Requirements

1. **Range-driven width**: Integer representation width SHALL be derived from the value's analyzed range, not from a target type name.
2. **Exact width**: An integer value SHALL be representable in the minimal width its range admits; lowering MAY widen to a native size for arithmetic but SHALL NOT narrow below the range.
3. **Representation selection**: For real-valued quantities, representation (IEEE-754 / posit / fixed-point) SHALL be selected per target as a function of the analyzed and dimensional range.
4. **Coeffect carriage**: The inferred width/representation SHALL be recorded as a coeffect on the PSG and preserved through lowering without recomputation.
5. **No silent default**: An unanalyzable range SHALL be reported as an error requesting annotation; the compiler SHALL NOT assume a default width.
6. **No conversion, no width-named type**: There SHALL be no width-named numeric type, no width-bearing literal suffix and no conversion between representations; an intended loss SHALL be written as arithmetic (§7) whose range is analysed like any other value's.
7. **No undefined overflow**: Arithmetic on analysed ranges SHALL NOT overflow, its result's representation being selected to cover its range; a value whose range a boundary's declared representation does not cover SHALL be diagnosed (CCS8012, a warning promoted under `--warnaserror`) and SHALL NOT be silently truncated, wrapped or saturated.
8. **JavaScript realization**: On the JSIR pathway, integer realization SHALL follow the JavaScript row of §8; the wide-integer mechanism SHALL be documented; a range exceeding the documented realization's exact envelope SHALL be diagnosed, not silently realized in binary64; fixed-width wrapping semantics SHALL be preserved observably.

## References

- Swamy, N., et al. *F\* — refinement-typed machine integers and verified low-level code.* https://www.fstar-lang.org
- Baaij, C., et al. *CλaSH: structural descriptions of synchronous hardware (Haskell→FPGA).*
- Gustafson, J. (2017). *Posit Arithmetic*; *Standard for Posit Arithmetic* (2022).
- Petricek, T., Orchard, D., & Mycroft, A. (2014). Coeffects: A Calculus of Context-Dependent Computation. *ICFP '14*.
