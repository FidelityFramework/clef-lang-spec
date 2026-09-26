---
title: "Interactive Development"
weight: 680
category: Compiler
status: normative
---

This chapter specifies native interactive Clef execution, script files, session
state, and integration with development tooling.

## Overview

Interactive development keeps the compiler available for repeated submissions,
inspection and native invocation. Clef source has the same type, effect,
lifetime and target requirements whether submitted interactively or compiled
ahead of time.

| Name | Role |
|---|---|
| `.clef` | Clef implementation source, compiled through CCS and Composer. |
| `.clefx` | Clef script file. |
| `clefx` | Clef interactive CLI. |

Clef has no separate signature-file extension. Module signatures are declared
inline; see [Namespace and Module Signatures](namespace-and-module-signatures.md).

## Semantic Requirements

1. **One semantic pipeline.** CCS and Baker perform source admission,
   construction, saturation and proof checking. Composer and Alex witness the
   resulting graph under the ordinary target and lowering contracts.
2. **Native execution.** Interactive Clef behavior comes from the admitted
   native code.
3. **Versioned observations.** Types, graphs, proofs, artifacts and execution
   results retain their source, compiler-generation and target identities.
4. **Shared authority.** Editors, agents and other clients consume the same
   compiler-owned facts and session contract.
5. **Comparable behavior.** For an admitted computation, interactive and AOT
   execution must agree under equivalent inputs and initialization.

## Clef Interactive (clefx)

### Invocation

The interactive CLI is named `clefx`.

### Session Model

A session identifies its source submissions, selected project and dependencies,
target, compiler generation, checked definitions, evidence and native execution
products. Session-visible results must distinguish checking from execution:
a checked declaration alone establishes neither completed initialization nor
the existence of a native value.

- A submission and any derived result retain the revision and origin needed to
  distinguish current state from stale work.
- Changed source, dependencies, target or compiler implementation invalidate the
  products that depend on them. Reuse requires valid provenance for every
  dependency.
- Required unresolved premises remain explicit at the computation's commitment
  boundary. Native invocation requires those premises to be established.
- Shared clients must select their session and target explicitly.

A session must define the behavior of binding and type redefinition, references
retained by earlier code, submission commitment, and recovery after partial
execution failure for each operation it admits. It must reject an operation
whose required session semantics it cannot provide.

### Execution Model

The native interactive pathway is:

```fsother
Clef source
  -> CCS and Baker
  -> versioned semantic graph and required evidence
  -> Alex witnesses
  -> admitted MLIR
  -> LLVM JIT
  -> native invocation
```

The [backend lowering contract](backend-lowering-architecture.md) applies at the
same boundaries as in AOT compilation. Interactive execution uses native code;
there is no separate interpreter or hybrid execution mode.

A .NET or FSI host may execute the compiler implementation and inspect its data.
The host's evaluation and value representations do not define Clef semantics.
The session contract does not require a particular actor or process topology.

## Script Files

### File Extension

Clef script files use the `.clefx` extension. A Clef script uses Clef source and
semantics through the native compilation pipeline.

### Script Structure

A script contains declarations and expressions. Script admission and execution
must preserve source identity, dependency resolution and initialization order.
Repeated loading and effect replay are subject to the same checking, lifetime
and invalidation requirements as other session operations.

### Script Directives

Script directives are subject to the ordinary compiler, project and lifetime
contracts. F# assembly-loading directives are not Clef dependency mechanisms.

### Shebang Support

The lexer treats a leading shebang as a comment, as described in
[Lexical Analysis](lexical-analysis.md#shebang).

## Memory Model in Interactive Mode

Native interactive values follow the ordinary
[lifetime](closure-representation.md) and
[memory-region](memory-regions.md) contracts, including ownership, region,
representation and target-admission requirements.

Retained values require an explicit storage lifetime and an admitted host/native
boundary. Reset, unloading or replacing code must account for live values,
closures and external references that depend on that code or storage. The
session must not expose invalidated native storage as a valid value.

## Platform Targeting

Types, widths, layouts, extern bindings and evidence come from the selected
target's declarations. The host's .NET types and ABI do not choose Clef layouts.
Native invocation requires a compatible, explicitly selected execution
environment and its admitted boundary contracts.

Changing target invalidates target-dependent compiler, proof and execution
products.

## Value Display Without Runtime Reflection

Native value presentation must use compiler-known type and layout information
and an admitted representation/lifetime boundary. Clef does not provide a
universal `obj` container or runtime reflection. The ordinary
[formatting contracts](native-type-mappings.md#the-universal-base-type-obj-is-not-available)
apply.

Formatting must not require evaluating the user's computation a second time.

## Tooling Architecture

Composer provides the compiler/execution service. CCS supplies semantic facts.
Lattice and other clients present those facts through versioned projections
with the source, compiler-generation and target identities of the underlying
observations.

## Package Management Integration

Interactive dependencies must participate in the ordinary project/dependency
identity, checking and invalidation contracts. A package-loading directive must
not bypass those contracts.

## Tooling Integration

Editor evaluation and shared client observations use the same session
authority. Clients must distinguish source diagnostics from execution results.

## Interoperability

Native interactive calls retain the ordinary
[FFI boundary](ffi-boundary.md) requirements, including source declarations,
target ABI, ownership and lifetime. JIT linking must preserve these requirements
and the evidence on which admission depends.

## Diagnostics

Interactive clients must distinguish source diagnostics, unresolved premises,
unsupported session operations, stale observations, target incompatibility and
execution failures.

## Grammar

Clef source within submissions follows the ordinary language grammar.
