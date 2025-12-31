# F# Native Language Specification

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

<p align="center">
🚧 <strong>Under Active Development</strong> 🚧<br>
<em>This project is in early development and not intended for production use.</em>
</p>

**Toward a normative specification for native F# type semantics and memory management.**

## Table of Contents

- [Overview](#overview)
- [The Fidelity Framework](#the-fidelity-framework)
- [Specification Flow](#specification-flow)
- [Why fsnative-spec Contains More Than the F# Specification](#why-fsnative-spec-contains-more-than-the-f-specification)
- [The Core Principle: Same Types, Native Semantics](#the-core-principle-same-types-native-semantics)
- [Key Semantic Additions](#key-semantic-additions)
- [The Absorption Model](#the-absorption-model)
- [Document Organization](#document-organization)
- [Normative Language](#normative-language)
- [Relationship to the F# Language Specification](#relationship-to-the-f-language-specification)
- [Contributing](#contributing)

## Overview

fsnative-spec aims to define the complete language semantics for [fsnative](https://github.com/speakeztech/fsnative) (F# Native Compiler Services). Where the [standard F# specification](https://fsharp.org/specs/language-spec/) describes behavior in terms of the .NET runtime and BCL types, fsnative-spec provides explicit definitions for everything the CLR normally handles implicitly: type layouts, memory ownership, lifetime verification, and deterministic resource management.

**The F# you write stays the same.** You write `string`, `option`, `int`, `array` - the familiar F# types. fsnative-spec defines what those types *mean* when targeting native compilation. The specification is about semantics, not new syntax.

## The Fidelity Framework

fsnative-spec is part of the **Fidelity** native F# compilation ecosystem:

| Project | Role |
|---------|------|
| **[Firefly](https://github.com/speakeztech/firefly)** | AOT compiler: F# → PSG → MLIR → Native binary |
| **[Alloy](https://github.com/speakeztech/alloy)** | Native standard library with platform bindings |
| **[BAREWire](https://github.com/speakeztech/barewire)** | Binary encoding, memory mapping, zero-copy IPC |
| **[Farscape](https://github.com/speakeztech/farscape)** | C/C++ header parsing for native library bindings |
| **[XParsec](https://github.com/speakeztech/xparsec)** | Parser combinators powering PSG traversal and header parsing |
| **[fsnative](https://github.com/speakeztech/fsnative)** | F# Native Compiler Services (FNCS) |
| **fsnative-spec** | F# Native language specification (this repository) |

The name "Fidelity" reflects the framework's core mission: **preserving type and memory safety** from source code through compilation to native execution.

## Specification Flow

```
┌────────────────────────────────────────────────────────────────┐
│                      Specification Flow                         │
│                                                                 │
│   fsnative-spec              fsnative              Firefly      │
│   ┌───────────┐            ┌───────────┐        ┌───────────┐  │
│   │           │  specifies │           │  uses  │           │  │
│   │ NORMATIVE │───────────▶│   FNCS    │───────▶│   Alex    │  │
│   │   RULES   │            │           │        │           │  │
│   └───────────┘            └───────────┘        └───────────┘  │
│        │                         │                    │         │
│        │                         │                    │         │
│   "Strings SHALL             Implements            Generates   │
│    be NativeStr"           type resolution      native code    │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

- **fsnative-spec** (this repository): Defines WHAT F# Native means
- **fsnative**: Implements HOW F# Native works (FNCS compiler services)
- **Firefly**: Consumes typed trees to produce native binaries

## Why fsnative-spec Contains More Than the F# Specification

The standard F# specification makes extensive use of the .NET runtime as an implicit substrate. Consider what the F# spec does *not* need to define:

- **Memory Allocation**: The F# spec never explains where objects live in memory, how allocation works, or when memory is reclaimed. It simply notes that values are created and trusts the CLR's garbage collector.
- **Object Layout**: The F# spec doesn't define how a record's fields are arranged in memory, what padding exists between fields, or how discriminated union tags are represented.
- **Reference Semantics**: The F# spec doesn't distinguish between a value that owns its memory and a value that borrows someone else's memory.
- **Resource Cleanup**: The F# spec has no drop semantics. Objects are allocated, used, and eventually collected.
- **Type Identity**: In managed F#, types are identified by their assembly metadata.

fsnative-spec explicitly defines all of these. Native compilation has no runtime to defer to.

## The Core Principle: Same Types, Native Semantics

When you write this F# code:

```fsharp
let greeting = "Hello, World!"
let maybeValue = Some 42
let numbers = [| 1; 2; 3 |]
```

You're using `string`, `option`, and `array` - exactly as you would in any F# program. The specification defines what these types mean for native compilation:

| F# Syntax | Standard F# | F# Native | Why |
|-----------|-------------|-----------|-----|
| `"Hello"` | `System.String` | `NativeStr` | UTF-8, fat pointer, no GC |
| `Some 42` | `int option` (reference) | `int voption` | Value type, non-nullable |
| `[| 1; 2; 3 |]` | `System.Int32[]` | `NativeArray<int>` | Fat pointer, explicit lifetime |

## Key Semantic Additions

Beyond redefining what existing types mean, fsnative-spec covers concepts that have no equivalent in the F# specification:

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

fsnative defines these types **intrinsically** - built into the compiler itself. When you write `int`, the compiler knows its representation, operations, and semantics because that knowledge is part of fsnative, not discovered from external sources.

This means:
- Type resolution requires no external assemblies
- SRTP constraints resolve against built-in definitions
- The compiler contains the complete type system

Alloy provides library *functions* that operate on these intrinsic types. The types themselves are defined by fsnative per this specification.

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
| 3 | Lexical Analysis | `lexical-analysis.md` | Stable |
| 4 | Basic Grammar Elements | `basic-grammar-elements.md` | Stable |
| 5 | Types and Type Constraints | `types-and-type-constraints.md` | Revised |
| 6 | **Native Type Mappings** | `native-type-mappings.md` | **New** |
| 7 | Expressions | `expressions.md` | Review |
| 8 | Patterns | `patterns.md` | Stable |
| 9 | Type Definitions | `type-definitions.md` | Review |
| 10 | Units of Measure | `units-of-measure.md` | Stable |
| 11 | Namespaces and Modules | `namespaces-and-modules.md` | Revised |
| 12 | Namespace and Module Signatures | `namespace-and-module-signatures.md` | Revised |
| 13 | Program Structure and Execution | `program-structure-and-execution.md` | Revised |
| 14 | **Memory Regions** | `memory-regions.md` | **New** |
| 15 | **Access Kinds** | `access-kinds.md` | **New** |
| 16 | **Platform Bindings** | `platform-bindings.md` | **New** |
| 17 | Inference Procedures | `inference-procedures.md` | Revised |
| 18 | Lexical Filtering | `lexical-filtering.md` | Stable |
| 19 | Special Attributes and Types | `special-attributes-and-types.md` | Revised |
| 20 | **Error Handling** | `error-handling.md` | **New** |
| 21 | **Interactive Development** | `interactive-development.md` | **New** |
| 22 | The Native Library Alloy | `the-native-library-alloy.md` | **Rewritten** |
| 23 | Features for ML Compatibility | `features-for-ml-compatibility.md` | Stable |

### Removed Chapters

The following chapters from the standard F# specification are **not applicable** to F# Native and have been removed:

| Chapter | Reason |
|---------|--------|
| Provided Types | Type providers require .NET runtime |
| Custom Attributes and Reflection | System.Reflection not available |

### Chapter Status Key

| Status | Meaning |
|--------|---------|
| **Stable** | Minimal changes needed from fslang-spec |
| **Review** | Needs review for BCL assumptions |
| **Needs revision** | Known BCL dependencies to remove |
| **New** | fsnative-specific chapter (not in fslang-spec) |
| **Rewritten** | Completely rewritten for fsnative |

## Normative Language

The specification uses RFC 2119 keywords:

- **SHALL/MUST**: Absolute requirement
- **SHALL NOT/MUST NOT**: Absolute prohibition
- **SHOULD**: Recommended but not required
- **MAY**: Optional

Example:
> NORMATIVE: String literals SHALL have type `NativeStr`, not `System.String`.

## Relationship to the F# Language Specification

The [F# Language Specification](https://fsharp.org/specs/language-spec/) is an extensive document covering syntax, type system, name resolution, evaluation semantics, and more. fsnative-spec takes this specification as its starting point:

- **Revisions**: Sections that assume .NET runtime behavior are revised to define explicit native semantics
- **Removals**: Content on .NET interop, reflection, and runtime type discovery is omitted
- **Additions**: New chapters cover ownership, borrowing, memory regions, access kinds, and lifetime constraints
- **Lock-Step Evolution**: fsnative (the compiler) and fsnative-spec evolve together

## Contributing

Specification changes require:
1. Discussion of semantic implications
2. Coordination with FNCS implementation
3. Validation via Firefly compilation

Changes to normative sections must include:
- Rationale for the change
- Impact analysis on existing code
- Implementation plan for FNCS

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contact

fsnative-spec is being developed by [SpeakEZ Technologies](https://speakez.tech) as part of the Fidelity native compilation framework.

## Acknowledgments

- **[F# Language Specification](https://fsharp.org/specs/language-spec/)**: The foundation this specification extends
- **Don Syme and F# Contributors**: For creating an elegant functional language
- **Firefly Team**: For the native compilation infrastructure

---

*Toward the rules that will make native F# safe and deterministic.*