---
title: "Scope"
weight: 3
category: Process
status: normative
---

> **What this specification defines, what it does not, and the boundary between the language and the implementation that realizes it.**

## 1. Scope

This document specifies the **Clef programming language**: its lexical structure, grammar, type system, semantics, and the normative behavior a conforming implementation must provide. It defines Clef by specifying the requirements an implementation must satisfy; the first such requirement is that an implementation realize the language as described here.

This specification covers:

- the **surface language** a developer writes — lexical structure, grammar, types, expressions, patterns, modules, and the unit-of-measure and dimensional discipline (the *Language* chapters);
- the **semantics** of programs — error handling, the computation models (incremental, observable, reactive), numeric representation selection and rounding, and concurrency and memory ordering (the *Semantics* chapters);
- the **representation** of language constructs as they are carried and lowered — the Program Semantic Graph, closure and discriminated-union and collection representations, width inference, and memory regions (the *Representation* chapters);
- the **compiler-internal** structures and procedures that a conforming implementation's front end realizes — the typed expression representation, inference procedures, backend lowering architecture, and the foreign-function boundary (the *Compiler* chapters); and
- the **platform** surface — platform bindings, predicates, and intrinsic modules (the *Platform* chapters).

## 2. Out of scope

This specification does **not** define:

- **A specific implementation.** Clef Compiler Services (CCS) and the Composer toolchain are the reference implementation; their internal engineering, build system, performance characteristics, and source organization are documented separately and are not normative here. Where this specification describes compiler structures (the PSG, the inference procedures, Alex), it specifies the *normative shape and obligations* those structures must satisfy, not a particular code realization of them.
- **Target-specific behavior beyond what is marked implementation-defined.** Numeric representation selection, capability gating, and platform predicates are specified as functions of a target's stated facts; the concrete facts of any given target (its format set, its capabilities) are implementation-defined and documented by the implementation, per [Behavior Classification](behavior-classification.md).
- **The standard library in full.** The intrinsic modules required by the language are specified here; the broader standard library is a released interface governed by the compatibility discipline of [Change Process and Specification Management §6](change-process-management.md), and its complete surface is documented separately.
- **Tooling, packaging, and development environment**, except where a normative language requirement depends on them (e.g. the platform-quotation mechanism, or interactive development semantics).

## 3. Relationship to the implementation

The relationship between this specification and an implementation is the conformance relationship defined in [Conformance](conformance.md). In brief: this document states requirements; a *conforming implementation* satisfies them; a *conforming program* is one that violates none of the language's diagnosable rules. Where this specification says an implementation **SHALL** do something, that is a conformance requirement, not a description of how the reference implementation happens to do it.

## 4. The role of the design notes

Throughout this specification, **Lineage** notes, **rationale** passages, and worked examples are *informative* (see [Conformance §2](conformance.md) on the normative/informative distinction). They explain *why* a requirement takes the form it does and where it descends from, but they impose no requirement of their own. The intellectual lineage of Clef — its descent from the F# and ML family, and the specific F#-origin features it carries — is recorded in these informative notes and is never the source of a normative obligation.
