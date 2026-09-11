---
title: "Behavior Classification"
weight: 7
category: Process
status: normative
---

> **The classes of behavior this specification does not fully fix, and Clef's deliberate near-elimination of the most dangerous of them.**

## 1. The three classes, and why Clef's distribution among them is a design statement

A language specification cannot fix every behavior to a single outcome; some behavior is left to the implementation, to the target, or to the moment of execution. The discipline of a specification is in *which* behavior it leaves open and *how* it labels it. There are three classes:

- **Implementation-defined behavior** — behavior this specification permits to vary between implementations or targets, *and which the implementation SHALL document*. The variation is bounded and visible: a program can read the documentation and know what its implementation does.
- **Unspecified behavior** — behavior this specification permits to vary, where the implementation is *not* required to document the choice and the program may not rely on it. The result is one of a stated set of possibilities, but which one is not fixed.
- **Undefined behavior** — behavior for which this specification imposes *no* requirement at all: a program that reaches it has no defined meaning.

The conventional systems language (C is the archetype) carries a large catalogue of **undefined behavior** — the class that makes such languages fast and dangerous in the same stroke, because the compiler may assume it never happens and the program that triggers it has no meaning at all. **Clef's defining position is that this class is nearly empty by construction.** The conditions other languages leave undefined, Clef makes *diagnosed errors* (the diagnostic obligation of [Conformance §5](conformance.md)): an unobservable range, an uncovered representation, a wait-for cycle, a missing capability, a representation seam violation — each is a property a conforming implementation SHALL detect and report, not a silent door into meaninglessness. There is no `unsafe` escape hatch in the language: each foreign boundary is handled through a structured, checked contract rather than an unchecked region ([FFI Boundary](ffi-boundary.md) for C; [JavaScript Boundary Semantics](javascript-boundary.md) for the JavaScript host).

This is not an incidental virtue; it is the same discipline as everywhere else in the specification — *where a guarantee cannot be given, the condition is surfaced loudly rather than assumed away*. The behavior classification is where that discipline is stated as a property of the language as a whole.

## 2. Implementation-defined behavior

An aspect this specification marks **implementation-defined** SHALL be documented by every conforming implementation ([Conformance §3.3](conformance.md)). A program may rely on implementation-defined behavior; doing so makes the program portable across exactly those implementations that document the same choice.

The principal implementation-defined parameters of Clef are the **target's stated facts**, because the language is specified as a function of them rather than of a fixed machine:

- **The representation set a target offers.** Numeric selection chooses among the representations a target provides; *which* representations a given target provides (which posit widths, whether IEEE `f32`/`f64`, whether fixed-point, whether a quire) is implementation-defined and documented. The selection *function* is fully specified; its *codomain* is the documented per-target set ([Numeric Selection §8](numeric-selection.md)).
- **The capabilities a target reports.** Whether a target provides a capability natively, by emulation, or not at all (the three-valued capability of [Numeric Selection §7](numeric-selection.md) and [Rounding §7](rounding.md)) is an implementation-defined, documented fact. The *gate* over capabilities is specified; the per-target capability values are documented.
- **Platform facts** — endianness, word width and hardware features — are implementation-defined and belong to platform declarations. CCS reads the implemented declaration fields; additional capability consumers require explicit implementation ([Platform Predicates](platform-predicates.md)).
- **The `ulp_min` of each representation family**, where this specification has left the exact floor to be pinned per family ([Numeric Selection §14](numeric-selection.md)), is implementation-defined until fixed by an accepted change.

In every case the pattern is the same and is itself a normative property: **the rule is specified; the target-specific value is implementation-defined and documented.** This is what lets one source target many machines while remaining checkable — the machine-specific facts are parameters, read from documentation and from the platform mechanism, not assumptions baked into the program.

## 3. Unspecified behavior

An aspect this specification marks **unspecified** may vary between implementations, or between executions, without documentation, and a program SHALL NOT rely on it. Where this specification states that a result is one of a set of possibilities without fixing which, that selection is unspecified.

Unspecified behavior in Clef is deliberately narrow and is confined to places where fixing the outcome would over-constrain a sound implementation without benefit — for example, the order in which independent, side-effect-free subcomputations are evaluated, where every order yields the same observable result and the freedom permits the implementation to schedule them as the target prefers. Because the language's pure core is referentially transparent, evaluation-order freedom is unspecified *without* being a source of differing results: the latitude exists, but a conforming program cannot observe it.

Unspecified behavior SHALL NOT extend to the silent production of a *wrong* result where the specification requires a *diagnosed* one. The distinction is sharp: a permitted set of *equivalent* outcomes is unspecified behavior; a silently-wrong outcome in place of a required diagnostic is non-conformance ([Conformance §5](conformance.md)).

## 4. Undefined behavior

This specification reserves **undefined behavior** for conditions, if any, for which it imposes no requirement and a program reaching them has no defined meaning. **It is the design intent of Clef that this class be empty for any conforming program**, and that the conditions a conventional language would leave undefined be, in Clef, diagnosable errors instead.

Where a residual undefined behavior is unavoidable — for instance, behavior arising entirely within foreign code reached across a foreign boundary (foreign C across the [FFI Boundary](ffi-boundary.md); foreign JavaScript or the host runtime across the [JavaScript Boundary](javascript-boundary.md)), beyond the reach of the language's own guarantees — this specification SHALL name it explicitly at the point it arises and SHALL NOT leave it to inference. A behavior is undefined in Clef only where this specification says so in as many words. The absence of a general, unstated reservoir of undefined behavior is a normative property of the language, not a hopeful description: a conforming implementation that would otherwise reach an undefined condition within Clef's own semantics is instead required to diagnose the condition that leads to it.

## 5. The relationship to saturation and explicit edges

Two mechanisms the specification provides are the reason undefined behavior can be pushed almost entirely into the diagnosable class:

- **Saturation, not wraparound, at representation edges.** Where a value reaches the edge of its representation's envelope, the result is a loud, located condition — a diagnosed error or an *explicit, specified* saturation — never a silent precision collapse or a wraparound whose result has no defined meaning ([Numeric Selection §6, §7](numeric-selection.md); [Rounding §5](rounding.md)). Saturation is *defined* behavior; the point is that the edge is specified rather than left undefined.
- **Explicit, witnessed conversion, not reinterpretation.** A representation change is an explicit, fidelity-recorded conversion ([Rounding §6](rounding.md)), never an untyped reinterpretation of a value's bits. The bit-pattern reinterpretation that is a primary source of undefined and implementation-defined behavior in conventional languages is, by [the Cryptography and Bits chapter §3.3](intrinsics-cryptography-bits.md), *not provided* — binary serialization goes through the structured BAREWire path, which carries the value's type rather than punning it.

## 6. Behavior classification requirements

1. Every behavior this specification does not fully fix **SHALL** be classified as implementation-defined, unspecified, or undefined, at the point it arises.
2. Implementation-defined behavior **SHALL** be documented by every conforming implementation; the rule is specified and the target-specific value is the documented parameter.
3. Unspecified behavior **SHALL NOT** encompass the silent production of a wrong result where this specification requires a diagnosed one.
4. Undefined behavior **SHALL** be named explicitly wherever it is reserved; this specification reserves no general, unstated undefined behavior, and a condition a conventional language would leave undefined within Clef's own semantics **SHALL** instead be a diagnosable error ([Conformance §5](conformance.md)).
5. A representation edge **SHALL** be a diagnosed error or an explicitly specified saturation, never a silent wraparound or precision collapse; a representation change **SHALL** be an explicit witnessed conversion, never a bit reinterpretation.
