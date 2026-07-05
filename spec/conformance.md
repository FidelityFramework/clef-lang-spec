---
title: "Conformance"
weight: 6
category: Process
status: normative
---

> **What it means for an implementation to conform to this specification, what a conforming program is, and the diagnostic obligation that distinguishes a standard from a description.**

## 1. The role of this clause

The other chapters of this specification state requirements. This clause defines what those requirements *bind*: the relationship between this document, an implementation that realizes Clef, and a program written in Clef. A requirement is meaningful only against a stated notion of conformance; this clause supplies it.

## 2. Normative and informative content

Every part of this specification is either **normative** or **informative**.

- **Normative** content imposes requirements. The clauses, the **SHALL** / **SHALL NOT** / **SHOULD** / **MAY** statements, the type rules, the grammar productions, and the per-chapter Normative Requirements sections are normative.
- **Informative** content explains, motivates, or illustrates, and imposes no requirement. Notes, examples, **Lineage** passages, rationale, and any chapter or annex explicitly marked informative are informative. The intellectual lineage of a feature — its descent from prior art or the F# and ML family — is always informative and is never the source of an obligation.

Where the two appear together, the normative content governs. An example that appears to permit behavior the normative text forbids does not permit it; the example is in error and the normative text stands.

The interpretation of the requirement keywords is as follows. **SHALL** and **SHALL NOT** state an absolute requirement or prohibition. **SHOULD** and **SHOULD NOT** state a strong recommendation: there may exist valid reasons to deviate, but the full implications must be understood and weighed. **MAY** states that an item is permitted but not required. A conforming implementation that exercises a **MAY** permission, or deviates from a **SHOULD** with reason, remains conforming; one that violates a **SHALL** does not.

## 3. Conforming implementation

A **conforming implementation** of Clef SHALL:

1. **Accept and correctly realize every conforming program** (§4), giving it the behavior this specification defines, within the bounds of the behavior classification of [Behavior Classification](behavior-classification.md).
2. **Diagnose every violation of a diagnosable rule** (§5). For any program that violates a diagnosable rule, a conforming implementation SHALL issue at least one diagnostic message identifying the violation, and SHALL NOT silently accept the program as though no violation occurred.
3. **Document its implementation-defined behavior.** For each aspect this specification marks implementation-defined, a conforming implementation SHALL document its choice ([Behavior Classification §2](behavior-classification.md)). The set of representations a target offers, the capabilities a target reports, and the platform facts the platform-predicate mechanism reads are the principal implementation-defined parameters.
4. **Satisfy the normative references it invokes** ([Normative References](normative-references.md)) for the requirements that invoke them.

A conforming implementation MAY provide extensions — features, intrinsics, or relaxations beyond this specification — provided that no extension alters the behavior of a conforming program that uses only the specified language. An implementation SHOULD provide a means by which a program can be checked against this specification alone, with extensions disabled, so that portability can be established.

## 4. Conforming program

A **conforming program** is a Clef program that contains no violation of any diagnosable rule of the language. A program that violates a diagnosable rule is **non-conforming**; a conforming implementation rejects it with at least one diagnostic (§3.2).

A program may rely only on:

- behavior this specification defines; and
- implementation-defined behavior, which it may rely on but which makes the program's behavior portable only across implementations that document the same choice.

A **strictly conforming program** additionally does not depend on any implementation-defined, unspecified, or extension behavior; its behavior is fixed by this specification alone and is therefore identical across all conforming implementations, modulo the behavior the specification itself leaves to the target ([Behavior Classification](behavior-classification.md)).

## 5. The diagnostic obligation

The diagnostic obligation is the heart of conformance for Clef, and it follows directly from the language's design discipline: **where this specification cannot guarantee a property, it requires the implementation to say so, loudly and at a located point, rather than to proceed on a silent assumption.**

A **diagnosable rule** is any rule of this specification whose violation a conforming implementation is required to detect. Unless a specific rule is designated otherwise, the rules of this specification are diagnosable at design time (during program checking) or at build time (during lowering). For every violation of a diagnosable rule, a conforming implementation SHALL issue at least one diagnostic.

This obligation is not a generality but a pervasive, concrete contract that recurs throughout the specification. In particular, a conforming implementation SHALL:

- **Report an unobservable range rather than fabricate one.** Where a dimensioned real value's range cannot be bounded by any provenance, the implementation SHALL emit the unobservable-range diagnostic and SHALL NOT silently assume a representation ([Numeric Selection §6](numeric-selection.md)).
- **Report an empty coverage set as an error.** Where no available representation on a target covers a value's range, that is a hard error to be diagnosed, never a silent selection of a non-covering representation ([Numeric Selection §2.1](numeric-selection.md)).
- **Report a wait-for cycle.** Where the synchronous wait-for relation over a region has no rank, the implementation SHALL diagnose the cycle, naming the chain of calls that wait on one another ([Synchronous RPC and Wait Classification](synchronous-rpc-liveness.md)).
- **Report a capability failure rather than silently degrade.** Where a value requires a target capability the target lacks (for example, exact accumulation on a target without quire support), that is a capability failure to be diagnosed, never a silent fallback to a weaker mechanism ([Numeric Selection §7, §10.2](numeric-selection.md)).
- **Report a representation seam violation at its source.** Where an unbounded bare value flows into a dimensioned context, the diagnostic SHALL fire at the dimensioning boundary and name the upstream source ([Numeric Selection §6.1](numeric-selection.md)).

A conforming implementation SHALL NOT discharge any of these by silently substituting a default, a softening term, a fabricated bound, or a weaker operation. Silence in place of a required diagnostic is non-conformance.

## 6. The preservation obligation through lowering

A design-time property this specification requires an implementation to establish SHALL be preserved through lowering to the target, or re-checked at the lowering steps that could perturb it. An implementation SHALL NOT establish a property at the source and then emit target code that violates it without diagnosis. Where a lowering pass is certified to preserve a carried property by construction, the property rides through; where it is not, the property SHALL be re-checked at the affected lowering edge. A property silently lost in lowering is a conformance violation, detected by the re-check.

## 7. Profiles

Some requirements of this specification bind an implementation only when it serves a class of target for which those requirements are relevant. A **profile** is a named body of normative content that binds an implementation when, and only when, the implementation claims that profile. An implementation claims a profile by stating that it does and by meeting every requirement the profile carries; the requirements of an unclaimed profile place no obligation on it.

A profile has a **conformance designator** — a name by which an implementation's conformance to the profile is claimed and reported, distinct from conformance to the core. An implementation conforms to the **core** of this specification (Clauses 1 through 13 and every chapter not assigned to a profile) unconditionally. It conforms to a profile additionally and separately, and a conformance claim states which profiles it includes.

A profile is used where a requirement is fully normative for the targets that need it and absent for those that do not, so that neither the implementation-defined channel (§3.3) nor a **SHOULD** captures the obligation correctly. The implementation-defined channel records a target's *choice* among options; a profile states the *requirements* that bind once a target is of the profile's class. A value a target reports (its representations, its capabilities) is implementation-defined; a body of requirements that binds a freestanding target and does not exist for a hosted one is a profile.

A chapter or a section assigned to a profile states that assignment. Its requirements are normative for an implementation that claims the profile and do not apply to one that does not. The **Freestanding Substrate** profile carries the requirements a target must meet when it provides the services a hosting environment would otherwise supply — persistence ([Modular Blob Storage](modular-blob-storage.md)), the storage regions that back it ([Memory Regions](memory-regions.md)), and the scheduling and memory management the target owns in the absence of an operating system. A hosted implementation that delegates these to its environment does not claim the profile, and the profile's requirements do not bind it.

## 8. Limits of conformance

This specification does not require that a conforming implementation:

- target any particular hardware, or any minimum set of targets;
- realize the *internal structure* of the reference implementation (the specific organization of CCS, Alex, or its nanopasses), only the normative obligations those structures carry ([Scope §3](scope.md));
- achieve any particular performance, except where a normative requirement is itself stated in terms of a cost bound; or
- diagnose a non-diagnosable rule, where this specification has explicitly designated a rule as non-diagnosable.

Where this specification leaves a behavior implementation-defined, unspecified, or (in the rare cases it arises) undefined, conformance is governed by [Behavior Classification](behavior-classification.md), not by this clause.

## 9. Conformance requirements

1. A conforming implementation **SHALL** accept and correctly realize every conforming program, within the behavior classification of [Behavior Classification](behavior-classification.md).
2. A conforming implementation **SHALL** issue at least one diagnostic for every violation of a diagnosable rule, and **SHALL NOT** silently accept a non-conforming program.
3. A conforming implementation **SHALL** document each aspect this specification marks implementation-defined.
4. A conforming implementation **SHALL** satisfy each normative reference for the requirements that invoke it.
5. A conforming implementation **SHALL NOT** discharge a required diagnostic by silent substitution, fallback, or fabrication.
6. A conforming implementation **SHALL** preserve through lowering, or re-check at lowering, every design-time property this specification requires it to establish.
7. A conforming implementation **SHALL** conform to the core unconditionally; it **SHALL** meet every requirement of each profile it claims, and a profile's requirements **SHALL NOT** bind an implementation that does not claim the profile.
8. An extension **SHALL NOT** alter the behavior of a conforming program that uses only the specified language; an implementation **SHOULD** provide a specification-only checking mode.
