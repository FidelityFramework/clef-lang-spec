# Clef Language Specification

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

<p align="center">
🚧 <strong>Under Active Development</strong> 🚧<br>
<em>This project is in early development and not intended for production use.</em>
</p>

**Toward a normative specification for Clef type semantics and memory management.**

`main` is the canonical development branch. The
[clef-lang-site](https://forge.spkez.dev/FidelityFramework/clef-lang-site)
repository imports its `spec/` directory through Hugo Modules and publishes the
[working draft](https://clef-lang.com/spec/draft/). The former `fidelity` branch
has been consolidated into `main`; Git history and existing tags are retained.
The inherited MkDocs/GitHub Pages publishing path has been retired.

## Table of Contents

- [Overview](#overview)
- [The Fidelity Framework](#the-fidelity-framework)
- [Specification Flow](#specification-flow)
- [Why clef-lang-spec Contains More Than the F# Specification](#why-clef-lang-spec-contains-more-than-the-f-specification)
- [The Core Principle: Same Types, Native Semantics](#the-core-principle-same-types-native-semantics)
- [Key Semantic Additions](#key-semantic-additions)
- [The Absorption Model](#the-absorption-model)
- [Document Organization](#document-organization)
- [Normative Language](#normative-language)
- [Relationship to the F# Language Specification](#relationship-to-the-f-language-specification)
- [Contributing](#contributing)

## Overview

clef-lang-spec aims to define the complete language semantics for [Clef](https://github.com/speakeztech/clef) (Clef Compiler Service). Where the [standard F# specification](https://fsharp.org/specs/language-spec/) describes behavior in terms of the .NET runtime and BCL types, clef-lang-spec provides explicit definitions for everything the CLR normally handles implicitly: type layouts, memory ownership, lifetime verification, and deterministic resource management.

**The F# you write stays the same.** You write `string`, `option`, `int`, `array` - the familiar F# types. clef-lang-spec defines what those types *mean* when targeting native compilation. The specification is about semantics, not new syntax.

## The Fidelity Framework

clef-lang-spec is part of the **Fidelity** native compilation ecosystem:

| Project | Role |
|---------|------|
| **[Clef](https://github.com/speakeztech/clef)** | Clef Compiler Service (CCS): parsing, type checking, PSG construction |
| **[Firefly](https://github.com/speakeztech/firefly)** | AOT compiler: consumes PSG → MLIR → Native binary |
| **[BAREWire](https://github.com/speakeztech/barewire)** | Binary encoding, memory mapping, zero-copy IPC |
| **[Farscape](https://github.com/speakeztech/farscape)** | C/C++ header parsing for native library bindings |
| **[XParsec](https://github.com/speakeztech/xparsec)** | Parser combinators powering PSG traversal and header parsing |
| **clef-lang-spec** | Clef language specification (this repository) |

The name "Fidelity" reflects the framework's core mission: **preserving type and memory safety** from source code through compilation to native execution.

## Specification Flow

```
┌────────────────────────────────────────────────────────────────┐
│                      Specification Flow                         │
│                                                                 │
│   clef-lang-spec              Clef                Firefly      │
│   ┌───────────┐            ┌───────────┐        ┌───────────┐  │
│   │           │  specifies │           │  uses  │           │  │
│   │ NORMATIVE │───────────▶│   CCS     │───────▶│   Alex    │  │
│   │   RULES   │            │           │        │           │  │
│   └───────────┘            └───────────┘        └───────────┘  │
│        │                         │                    │         │
│        │                         │                    │         │
│   "Strings SHALL             Implements            Generates   │
│    be NativeStr"           type resolution      native code    │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

- **clef-lang-spec** (this repository): Defines WHAT Clef means
- **Clef**: Implements HOW Clef works (CCS compiler service)
- **Firefly**: Consumes typed trees to produce native binaries

## Why clef-lang-spec Contains More Than the F# Specification

The standard F# specification makes extensive use of the .NET runtime as an implicit substrate. Consider what the F# spec does *not* need to define:

- **Memory Allocation**: The F# spec never explains where objects live in memory, how allocation works, or when memory is reclaimed. It simply notes that values are created and trusts the CLR's garbage collector.
- **Object Layout**: The F# spec doesn't define how a record's fields are arranged in memory, what padding exists between fields, or how discriminated union tags are represented.
- **Reference Semantics**: The F# spec doesn't distinguish between a value that owns its memory and a value that borrows someone else's memory.
- **Resource Cleanup**: The F# spec has no drop semantics. Objects are allocated, used, and eventually collected.
- **Type Identity**: In managed F#, types are identified by their assembly metadata.

clef-lang-spec explicitly defines all of these. Native compilation has no runtime to defer to.

## The Core Principle: Same Types, Native Semantics

When you write this F# code:

```fsharp
let greeting = "Hello, World!"
let maybeValue = Some 42
let numbers = [| 1; 2; 3 |]
```

You're using `string`, `option`, and `array` - exactly as you would in any F# program. The specification defines what these types mean for native compilation:

| F# Syntax | Standard F# | Clef | Why |
|-----------|-------------|------|-----|
| `"Hello"` | `System.String` | `NativeStr` | UTF-8, fat pointer, no GC |
| `Some 42` | `int option` (reference) | `int voption` | Value type, non-nullable |
| `[| 1; 2; 3 |]` | `System.Int32[]` | `NativeArray<int>` | Fat pointer, explicit lifetime |

## Key Semantic Additions

Beyond redefining what existing types mean, clef-lang-spec covers concepts that have no equivalent in the F# specification:

### Ownership and Borrowing

- Every value has exactly one owner
- References can borrow values without taking ownership
- The compiler verifies borrows don't outlive their owners
- Ownership can transfer (move semantics) or values can be copied

### Memory Regions

| Region | Use Case | Volatile | Cacheable |
|--------|----------|----------|-----------|
| `Stack` | Thread-local, automatic lifetime | No | Yes |
| `Arena` | Bulk allocation, batch deallocation | No | Yes |
| `Peripheral` | Memory-mapped I/O | Yes | No |
| `Sram` | General-purpose RAM | No | Yes |
| `Flash` | Read-only program memory | No | Yes |

### Access Kinds

Pointers carry access permissions:

| Kind | Read | Write | CMSIS |
|------|------|-------|-------|
| `ReadOnly` | Yes | No | `__I` |
| `WriteOnly` | No | Yes | `__O` |
| `ReadWrite` | Yes | Yes | `__IO` |

### Deterministic Cleanup

When values go out of scope, resources are freed immediately:
- Drop order is reverse declaration order
- No garbage collector decides when cleanup happens
- The programmer can reason about exactly when resources are released

### Peripheral Descriptors

Farscape generates type-safe hardware bindings:

```fsharp
[<PeripheralDescriptor("GPIO", 0x48000000UL)>]
type GPIO_TypeDef = {
    [<Register("MODER", 0x00u, "rw")>]
    MODER: Ptr<uint32, peripheral, readWrite>

    [<Register("IDR", 0x10u, "r")>]
    IDR: Ptr<uint32, peripheral, readOnly>
}
```

## The Absorption Model

In standard F#, types like `int` and `string` are defined in external assemblies. The compiler discovers their operations by reading assembly metadata.

Clef defines these types **intrinsically** - built into the compiler itself. When you write `int`, the compiler knows its representation, operations, and semantics because that knowledge is part of CCS, not discovered from external sources.

This means:
- Type resolution requires no external assemblies
- SRTP constraints resolve against built-in definitions
- The compiler contains the complete type system

The native library provides functions that operate on these intrinsic types. The types themselves are defined by CCS per this specification.

## Document Organization

The specification lives in the `spec/` directory. Chapter ordering is defined in `spec/Catalog.json`.

### Front Matter

| Document | File | Description |
|----------|------|-------------|
| Front Matter | `front-matter.md` | Title, copyright, version |
| RFC Status | `rfc-status.md` | RFC 2119 compliance notes |

### Specification Chapters

| # | Chapter | File | Status |
|---|---------|------|--------|
| 1 | Introduction | `introduction.md` | Revised |
| 2 | Program Structure | `program-structure.md` | Revised |
| 3 | Lexical Analysis | `lexical-analysis.md` | **Needs revision** |
| 4 | Basic Grammar Elements | `basic-grammar-elements.md` | **Needs revision** |
| 5 | Types and Type Constraints | `types-and-type-constraints.md` | Revised |
| 6 | **Type Representation Architecture** | `type-representation-architecture.md` | **New** |
| 7 | **Native Type Mappings** | `native-type-mappings.md` | **New** |
| 8 | **Native Type Universe** | `native-type-universe.md` | **New** |
| 9 | **NTU Types** | `ntu-types.md` | **New** |
| 10 | **NTU Dimensional Architecture** | `ntu-dimensional-architecture.md` | **New** |
| 11 | Expressions | `expressions.md` | Revised |
| 12 | Patterns | `patterns.md` | **Needs revision** |
| 13 | Type Definitions | `type-definitions.md` | Revised |
| 14 | Units of Measure | `units-of-measure.md` | **Needs revision** |
| 15 | Namespaces and Modules | `namespaces-and-modules.md` | Revised |
| 16 | Namespace and Module Signatures | `namespace-and-module-signatures.md` | Revised |
| 17 | Program Structure and Execution | `program-structure-and-execution.md` | Revised |
| 18 | **Program Semantic Graph** | `program-semantic-graph.md` | **New** |
| 19 | **Memory Regions** | `memory-regions.md` | **New** |
| 20 | **Closure Representation** | `closure-representation.md` | **New** |
| 21 | **Lazy Representation** | `lazy-representation.md` | **New** |
| 22 | **Discriminated Union Representation** | `discriminated-union-representation.md` | **New** |
| 23 | **Atomic Operations** | `atomic-operations.md` | **New** |
| 24 | **Incremental Computation** | `incremental-computation.md` | **New** |
| 25 | **Observable Computation** | `observable-computation.md` | **New** |
| 26 | **Seq Representation** | `seq-representation.md` | **New** |
| 27 | **Seq Operations Representation** | `seq-operations-representation.md` | **New** |
| 28 | **List Operations Representation** | `list-operations-representation.md` | **New** |
| 29 | **Map Representation** | `map-representation.md` | **New** |
| 30 | **Set Representation** | `set-representation.md` | **New** |
| 31 | **Option Operations Representation** | `option-operations-representation.md` | **New** |
| 32 | **Access Kinds** | `access-kinds.md` | **New** |
| 33 | **Platform Bindings** | `platform-bindings.md` | **New** |
| 34 | **FFI Boundary** | `ffi-boundary.md` | **New** |
| 35 | **Platform Predicates** | `platform-predicates.md` | **New** |
| 36 | **Backend Lowering Architecture** | `backend-lowering-architecture.md` | **New** |
| 37 | Inference Procedures | `inference-procedures.md` | Revised |
| 38 | Inference: Name Resolution | `inference-name-resolution.md` | **Draft** |
| 39 | Inference: Application Resolution | `inference-application-resolution.md` | **Draft** |
| 40 | Inference: Constraint Solving | `inference-constraint-solving.md` | **Draft** |
| 41 | Inference: Supplementary | `inference-supplementary.md` | **Draft** |
| 42 | **Clef Expressions** | `clef-expr.md` | **New** |
| 43 | Lexical Filtering | `lexical-filtering.md` | **Needs revision** |
| 44 | Special Attributes and Types | `special-attributes-and-types.md` | Revised |
| 45 | **Error Handling** | `error-handling.md` | **New** |
| 46 | **Interactive Development** | `interactive-development.md` | **New** |
| 47 | **Width Inference** | `width-inference.md` | **New** |
| 48 | **Numeric Selection** | `numeric-selection.md` | **New** |
| 49 | **Intrinsics: Crypto/Bits** | `intrinsics-crypto-bits.md` | **New** |
| 50 | **Reactive Signals** | `reactive-signals.md` | Revised |

### Removed Chapters

The following chapters from the standard F# specification are **not applicable** to Clef and have been removed:

| Chapter | Reason |
|---------|--------|
| Provided Types | Type providers require .NET runtime |
| Custom Attributes and Reflection | System.Reflection not available |
| Features for ML Compatibility | Clef has no OCaml-compatibility heritage; F# tokens (`(*IF-FSHARP*)`, `(*IF-OCAML*)`), the `--mlcompatibility` flag, the `#indent "off"` directive, and the `.ml`/`.mli` extensions do not apply |

### Chapter Status Key

| Status | Meaning |
|--------|---------|
| **Revised** | Adapted from fslang-spec; further revision may be needed |
| **Needs revision** | Known BCL or F#-specific dependencies to remove |
| **Draft** | Inherited from fslang-spec; voice and content not yet adapted |
| **New** | Clef-specific chapter (not in fslang-spec) |

## Normative Language

The specification uses RFC 2119 keywords:

- **SHALL/MUST**: Absolute requirement
- **SHALL NOT/MUST NOT**: Absolute prohibition
- **SHOULD**: Recommended but not required
- **MAY**: Optional

Example:
> NORMATIVE: String literals SHALL have type `NativeStr`, not `System.String`.

## Relationship to the F# Language Specification

The [F# Language Specification](https://fsharp.org/specs/language-spec/) is an extensive document covering syntax, type system, name resolution, evaluation semantics, and more. clef-lang-spec takes this specification as its starting point:

- **Revisions**: Sections that assume .NET runtime behavior are revised to define explicit native semantics
- **Removals**: Content on .NET interop, reflection, and runtime type discovery is omitted
- **Additions**: New chapters cover ownership, borrowing, memory regions, access kinds, and lifetime constraints
- **Lock-Step Evolution**: Clef (the compiler) and clef-lang-spec evolve together

## Contributing

Specification changes require:
1. Discussion of semantic implications
2. Coordination with CCS implementation
3. Validation via Firefly compilation

Changes to normative sections must include:
- Rationale for the change
- Impact analysis on existing code
- Implementation plan for CCS

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contact

clef-lang-spec is being developed by [SpeakEZ Technologies](https://speakez.tech) as part of the Fidelity native compilation framework.

## Acknowledgments

- **[F# Language Specification](https://fsharp.org/specs/language-spec/)**: The foundation this specification extends
- **Don Syme and F# Contributors**: For creating an elegant functional language
- **Firefly Team**: For the native compilation infrastructure

---

*Toward the rules that will make Clef safe and deterministic.*
