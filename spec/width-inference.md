---
title: "Width Inference"
weight: 430
category: Semantics
status: normative
---

> **Normative specification for value-range analysis, minimal-width derivation, and representation selection in Clef compilation.**

## 1. Overview

Clef infers numeric representation from the value's range, traced through the program's dataflow. For an integer, this determines the required bit width. For a real, the range and target declarations determine the posit, IEEE float, or fixed-point representation ([Numeric Selection](numeric-selection.md)).

The compiler tracks explicit arithmetic loss and derives signedness from the range (§3). Where required facts remain unresolved at representation commitment, it emits a located diagnostic (§6).

> **Lineage.** F\*'s refinement-typed machine integers informed the separation between value ranges and representations. Clef infers ranges over the [Program Semantic Graph](program-semantic-graph.md), including the relationships established by branch guards. Clash's hardware-width inference provides a further precedent for sizing hardware from program constraints.

## 2. Value-Range Analysis

Width inference combines interval analysis with the constraints over the PSG dataflow. An established range is recorded as `[a, b]`:

- **Literals** are point intervals: `124` has range `[124, 124]`.
- **Arithmetic** propagates intervals: `x + y` has range `[a_x + a_y, b_x + b_y]`; products, shifts, and so on follow standard interval arithmetic.
- **Comparisons seed ranges**: a value compared against a `threshold` is bounded by that threshold on the relevant branch, which in turn bounds any counter that must reach it.
- **Recurrences and counters** are bounded by their modulus or termination condition: a free-running counter `mod N` has range `[0, N-1]`.

For example, in a working FPGA design a wave-chase counter is free-running `mod (defaultPeriodMs * ticksPerMs * 2)` ≈ 10⁹, giving range `[0, ~10⁹]`.

### 2.1 Relational guards and immutable values

A branch guard establishes a relationship between values. For dimensionally compatible integer expressions, `count <= length - offset` establishes `offset + count <= length` on the true branch. Replacing that relationship with independent bounds for `offset` and `count` can lose the bound the program established. The implementation SHALL preserve such integer-affine guard constraints through aliases, operand reordering, and substitution through a pure Boolean helper. Integer-affine expressions here are sums of integer constants and integer-valued terms multiplied by constant integer coefficients. Further constraint fragments remain governed by the tiered verification discipline.

A carried constraint SHALL identify its originating guard and polarity, the value identities it relates, and the dependencies on mutable storage, if any. An interval consequence used for representation selection SHALL retain this derivation provenance on the PSG. Naming the sum before testing it and recomputing the same sum from unchanged operands on the guarded branch SHALL yield the same usable constraint. A strict integer guard `total < limit` entails `total <= limit - 1`. Each branch SHALL use the constraint corresponding to its comparison and polarity.

An immutable binding preserves the identity of its value. Rebinding another name or mutating storage reachable through a different value does not invalidate a fact about that immutable scalar. For an immutable reference to mutable storage, this stability applies to the reference value. Facts about the storage's contents require separate validity tracking. A write that can change a constraint's dependencies SHALL invalidate that constraint for later reads unless the analysis establishes its preservation. This applies to writes through aliases and calls whose effects can reach that storage.

### 2.2 Deferred demand and capture

A computation's dimensional type is established independently of when its result is demanded. Deferral SHALL preserve that type and its pending constraints. At a closure, lazy value, or sequence boundary, range facts over immutable captured values remain available: Clef's [flat closures](closure-representation.md#22-capture-semantics) capture immutable bindings by value. Mutable captures share storage by reference. A guard checked before construction establishes a fact about that storage at the check, while a later read requires the fact to remain valid. Storage-dependent constraints SHALL be established for the relevant demand or invocation, or remain unresolved. [Memoization](lazy-representation.md) preserves the result already computed. The assumptions used to compute it SHALL hold at the relevant reads.

These rules constrain the compiler's joint analysis and every target realization, including native flat environments and host closures on the JSIR pathway. Eager traversal of a compiler graph SHALL NOT be treated as evidence that a deferred source computation has executed. A lowering may change the representation of a capture, but SHALL preserve the capture mode, dimensional identity, and the validity conditions of any range fact it uses ([Conformance §6](conformance.md#6-the-preservation-obligation-through-lowering)).

## 3. Width Derivation (Integers)

From a range \([a, b]\), the minimal integer width is derived directly:

\[\mathrm{width}([a, b]) = \begin{cases} \lceil \log_2(b + 1) \rceil & a \ge 0 \quad (\text{unsigned}) \\ 1 + \lceil \log_2(\max(|a|,\, b + 1)) \rceil & a < 0 \quad (\text{signed}) \end{cases}\]

The inferred width is the minimum required by the range. The following are inferred widths from a working FPGA design (Arty A7-100T):

| Value | Range | Inferred width |
|---|---|---|
| `Counter` | free-running modulo `800 000 000`, `[0, 799 999 999]` | 30 bits, unsigned |
| `StepTick` | reset by a comparison against a threshold of at most `390 625`, `[0, 390 624]` | 19 bits, unsigned |
| `Phase` | cycles modulo `1024`, `[0, 1023]` | 10 bits, unsigned |
| `PeriodMs` | the latched period, the join of four button values and the initial `4000`, `[500, 4000]` | 12 bits, unsigned |

On the FPGA, each `seq.compreg` register is sized to the inferred width, with corresponding changes to its carry chain. No width is declared by hand. A non-negative range spends no sign bit: signedness is a fact of the range (§1), and the zero-extension or sign-extension an operand needs when it meets a wider operand follows from that fact, never from a type name. An implementation that adds a sign bit to every range, or extends by a fixed rule, does not conform to this section.

## 4. Representation Selection (Reals)

For real-valued quantities, the analyzed range selects a *representation*, not merely a width. Following the dimensional-type model, representation selection is a deterministic compile-time function of the range and the target's available formats:

\[r^* = \operatorname*{arg\,min}_{r \,\in\, R(\text{target})} \; \max_{x \,\in\, [a, b]} \; \frac{|x - \mathrm{round}_r(x)|}{|x|}\]

The objective minimizes worst-case relative error over the range. IEEE-754 has approximately uniform relative precision over its normal range (≈ \(2^{-p}\)). Posits taper precision toward magnitude 1.0, and fixed-point uses a fixed scale. The choice is per target (e.g. IEEE-754 on CPU, posit on FPGA, fixed-point on a neuromorphic core) and is surfaced at design time. See [NTU Dimensional Architecture](ntu-dimensional-architecture.md) and [Units of Measure](units-of-measure.md); the dimensional range of a value is the primary input to this function.

The objective requires a coverage constraint excluding representations whose dynamic range does not cover `[a, b]`, and a floor on the error metric where the range approaches zero. The full, sound objective and both side-conditions are specified in [Numeric Selection §2](numeric-selection.md#2-the-selection-objective); the integer-width rules are specified here.

## 5. Width and Representation as a Coeffect

The compiler settles width and representation as **coeffects** on the PSG, together with the range and declaration provenance used in selection. Later lowering passes read the settled choice and realize it in target operations. Allocation and lifetime use the same coeffect discipline ([Incremental Computation §10](incremental-computation.md)), with their constraints resolved jointly with dimensions and target reachability.

Preservation concerns the property and its evidence across each lowering edge. The implementation may carry this correspondence in compiler metadata or a checked witness without introducing runtime type inspection. When a representation of the source type has served its structural purpose, releasing it SHALL preserve or re-check the obligations still required below that edge ([Conformance §6](conformance.md#6-the-preservation-obligation-through-lowering)).

## 6. Unobservable Ranges

An unresolved range is a pending inference obligation while the graph is being elaborated and the platform context is incomplete. An implementation MAY expose that obligation in the editing environment without rejecting an otherwise consistent partial program. A missing observation, an unreachable computation, an inconsistent constraint, and a known range with no covering representation are distinct states. An implementation SHALL NOT substitute one for another.

At the boundary that commits a reachable integer value to a concrete representation, the required range and platform facts SHALL be established. If the range remains unobservable, the compiler SHALL report a located error tracing the missing provenance. It SHALL NOT guess a width or fall back to a machine-word default. The facts may come from arithmetic and guards, a checked domain law, or a platform or boundary declaration. The obligation need not be discharged by an annotation on the first line of source. If a known range has no covering representation, the hard coverage error of [Conformance §5](conformance.md#5-the-diagnostic-obligation) prevents that selection from being committed. The distinct real-valued unobservable cases, including the explicitly specified bare-real default, remain governed by [Numeric Selection §6](numeric-selection.md#6-the-default-and-unobservable-case).

## 7. Intended Loss Is Arithmetic

There is no explicit conversion in Clef, because there is nothing to convert between: a value is `int` or `float` at a dimension, its range is analysed, and its width or representation is selected from what the platform declares (§3, §4; [NTU Types](ntu-types.md)). No width-named type exists, no literal suffix names a width, and no operator changes a representation. Where a program intends to lose information it says so in arithmetic, and the result is a value with a known range like any other:

| Intent | Written as | Range |
|---|---|---|
| reduce modulo `2^n` | `x % 2^n` or `x &&& (2^n − 1)` | `[0, 2^n − 1]` |
| saturate to `[lo, hi]` | `clamp lo hi x` (`min hi (max lo x)`) | `[lo, hi]` |
| real to integer | `floor`, `ceiling`, `round`, `truncate` | the integer image of the range |
| integer to real | `float x` | `[a, b]`, exact where the selected real representation holds it |

Because the range of each of these is known, the width follows from it as it does everywhere, and no discipline is ever named at a site: the compiler never selects wrap or saturate, and never inserts a narrowing. A platform's declared boundary semantics for a representation (wrap on a two's-complement unit, saturate on a saturating block, exact on fabric: [Platform Bindings](platform-bindings.md)) are read only to realise `%` and `clamp` cheaply where the hardware does them natively, never to give a program its meaning.

Where a value meets a **boundary**, a representation a declaration fixed rather than the range (a wire-schema field, an MMIO register, a C ABI parameter, an endpoint contract, an exported entry point at the platform's word), the obligation is coverage: the analysed range is contained in the boundary's declared range, else CCS8012, a hard error, whose remedies are to bound the value or to change the declaration ([Numeric Selection §5](numeric-selection.md)). Overflow is therefore never undefined behaviour and never a runtime discipline: on analysed ranges it does not occur, and at a boundary it is a design-time finding.

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

- **Dimensional Type System**: the dimensional range of a value is the principal input to representation selection (§4); width inference is the value-level realization of the dimensional discipline.
- **[Native Type Universe](native-type-universe.md)**: inferred widths and representations are NTU representation choices carried through lowering.
- **Intended loss and boundaries**: range analysis of arithmetic and declared boundaries (§7) specifies representation changes; the `Convert` and `NTU Conversion` chapters are not part of this specification.

## 10. Normative Requirements

1. **Range-driven width**: Integer representation width SHALL be derived from the value's analyzed range, not from a target type name.
2. **Exact width**: An integer value SHALL be representable in the minimal width its range admits; lowering MAY widen to a native size for arithmetic but SHALL NOT narrow below the range.
3. **Representation selection**: For real-valued quantities, representation (IEEE-754 / posit / fixed-point) SHALL be selected per target as a function of the analyzed and dimensional range.
4. **Coeffect carriage**: The inferred width/representation SHALL be recorded as a coeffect on the PSG and preserved through lowering without recomputation.
5. **No silent default**: An integer range still unobservable at representation commitment SHALL be reported as an error tracing missing provenance (§6); the compiler SHALL NOT assume a default width.
6. **No conversion, no width-named type**: There SHALL be no width-named numeric type, no width-bearing literal suffix and no conversion between representations; an intended loss SHALL be written as arithmetic (§7) whose range is analysed like any other value's.
7. **No undefined overflow**: Arithmetic on analysed ranges SHALL NOT overflow, its result's representation being selected to cover its range; a value whose range a boundary's declared representation does not cover SHALL be diagnosed (CCS8012, a hard error) and SHALL NOT be silently truncated, wrapped or saturated.
8. **JavaScript realization**: On the JSIR pathway, integer realization SHALL follow the JavaScript row of §8; the wide-integer mechanism SHALL be documented; a range exceeding the documented realization's exact envelope SHALL be diagnosed, not silently realized in binary64; the declared arithmetic boundary semantics SHALL be preserved observably.
9. **Guard validity**: Integer-affine range constraints SHALL preserve their value identities, guard polarity, and derivation provenance; facts depending on mutable storage SHALL be re-established or invalidated when those dependencies may change (§2.1).
10. **Demand boundaries**: Deferred computation SHALL preserve dimensional identity and pending obligations; immutable capture facts SHALL remain available, and a mutable capture SHALL NOT be treated as a frozen value (§2.2).
11. **Deferred inference**: An unresolved obligation during elaboration SHALL NOT be silently replaced by a bound or representation; a required unresolved fact SHALL be diagnosed before the concrete representation is committed (§6).

## References

- Swamy, N., et al. *F\* — refinement-typed machine integers and verified low-level code.* https://www.fstar-lang.org
- Baaij, C., et al. *CλaSH: structural descriptions of synchronous hardware (Haskell→FPGA).*
- Gustafson, J. (2017). *Posit Arithmetic*; *Standard for Posit Arithmetic* (2022).
- Petricek, T., Orchard, D., & Mycroft, A. (2014). Coeffects: A Calculus of Context-Dependent Computation. *ICFP '14*.
