# F# Native Language Specification (fsnative-spec)

## Purpose

fsnative-spec defines the normative semantics for native F# compilation in the Fidelity framework. Where the standard F# specification assumes .NET runtime behavior, fsnative-spec explicitly defines:

- Native type layouts and representations
- Memory ownership and borrowing semantics
- Lifetime verification rules
- Deterministic resource management
- Memory region classifications
- Access kind enforcement

## Core Principle: Compiler-Controlled Memory Layout

> **The entire point of "Fidelity" is that the F# compiler controls memory layout - not MLIR, not LLVM.**

Semantic types (memory regions, access kinds, voption, etc.):
1. **Carry meaning through the entire compilation pipeline**
2. **Guide every memory layout decision** made by Fidelity
3. **ARE erased - but at the LAST possible lowering stage**
4. By the time code reaches LLVM: "the type information that guided every transformation has done its job and compiled away to nothing"

**Fidelity dictates; LLVM implements.**

## Familiar Names, Native Semantics

**Users write familiar F# type names. FNCS resolves native semantics at compile-time.**

| F# Syntax | Standard F# (BCL) | F# Native (FNCS) |
|-----------|-------------------|------------------|
| `"Hello"` | `System.String` | UTF-8 fat pointer `{ptr, len}` |
| `Some 42` | `int option` (heap, nullable) | `voption<int>` (stack, non-null) |
| `[| 1; 2; 3 |]` | `System.Int32[]` | `array<int>` fat pointer |
| `int` | `System.Int32` (32-bit) | Platform word (no GC tagging) |

> **CRITICAL**: Alloy shadow types (e.g., `type option<'T> = voption<'T>`) are **TEMPORARY WORKAROUNDS** that will be REMOVED once FNCS is complete. FNCS handles type resolution at the compiler level.

## Specification Documents

| Document | Purpose | Status |
|----------|---------|--------|
| `Native_Type_Universe.md` | Complete native type specification | **COMPREHENSIVE** |
| `FNCS_Specification.md` | F# Native Compiler Services spec | Cross-referenced |
| `Beyond_FSlang_Spec.md` | Historical context, extensions beyond fslang-spec | Updated |
| `FSharp_Features_In_Fidelity.md` | Feature coverage matrix | Reference |

### Native_Type_Universe.md Coverage (Phase A Complete)

| Part | Content | Status |
|------|---------|--------|
| Part 1 | Foundational Principles (products, sums, functions) | **Complete** |
| Part 2 | Primitive Types (unit, bool, integers, floats, char) | **Complete** - Session 1 enhanced with OCaml provenance, cache alignment, char resolved as UTF-32 |
| Part 3 | Structural Types (tuples, records, DUs) with memory layouts | **Complete** |
| Part 4 | Reference Types (string UTF-8, array, span) | **Complete** |
| Part 5 | Parameterized Types (option/voption, result, list) | **Complete** |
| Part 6 | Function Types (closures, inline, partial application) | **Complete** |
| Part 7 | Mutable State (ref cells, mutable bindings) | **Complete** |
| Part 8 | Memory Region Types (UMX absorption) | **Complete** |
| Part 9 | Coeffects | Deferred to Phase B |
| Appendix A | Type Mapping to MLIR | **Complete** |
| Appendix B | OCaml Type System Reference | **Complete** |
| Appendix C | OCaml/F#/Rust Comparison | **Complete** |
| Appendix D | Migration from Shadow Types | **Complete** |
| Appendix E | OCaml Provenance and Fidelity Extensions | **Complete** |

## Specification Parts

| Part | Title | Status |
|------|-------|--------|
| Part 1 | Native Type Universe | Specified |
| Part 2 | Null-Free Semantics | Specified |
| Part 3 | SRTP Resolution | Specified |
| Part 4 | Memory Semantics | Draft |
| Part 5 | Coeffects | Deferred (Phase B) |
| Part 6 | Platform Bindings | Specified |
| Part 7 | Compatibility | Specified |
| Part 8 | Diagnostics | Specified |
| Part 9 | Memory Region Types | Specified |
| Part 10 | Access Kind Enforcement | Specified |
| Part 11 | Peripheral Descriptors | Specified |
| Part 12 | Ownership/Coeffects | Reserved (Phase B) |

## OCaml Provenance (Appendix E)

fsnative draws provenance from OCaml's direct memory layout idioms - concepts that F#/.NET lacks because the CLR abstracts memory away.

**From OCaml (KEEP)**:
- Products/sums/functions as primitives
- Value-oriented structural assembly
- Deterministic tag layout for DUs
- Fat pointer concept
- No null philosophy

**From OCaml (SET ASIDE)**:
- 63-bit tagged integers (GC overhead)
- Desktop-centric assumptions
- Runtime type discrimination

**From Rust (ADAPT)**:
- Ownership → Memory regions + coeffects
- Drop semantics → Continuation-bounded resources
- Deterministic cleanup → Arena/scope-bounded lifetimes

**Fidelity Extensions Beyond Both**:
- Memory region types (Stack, Arena, Peripheral, Sram, Flash)
- Access kinds (ReadOnly, WriteOnly, ReadWrite)
- Cache line alignment as first-class concern
- Working set calculation at compile time
- Processor-specific optimization strategies

## Key Semantic Additions

### Absolute Null-Freedom

THE material difference from standard F#:
- No `null` values anywhere in the type system
- `option<'T>` has `voption` semantics (stack-allocated, non-null)
- APIs return `voption` instead of sentinel values (`IndexOf` returns `voption<int>`, not -1)
- FNCS emits errors on null assignment (FS8010+)

### Memory Regions

| Region | Use Case | Volatile | Cacheable |
|--------|----------|----------|-----------|
| `Stack` | Thread-local, automatic | No | Yes |
| `Arena` | Bulk allocation | No | Yes |
| `Peripheral` | Memory-mapped I/O | Yes | No |
| `Flash` | Read-only program memory | No | Yes |

### Access Kinds

| Kind | Read | Write | CMSIS |
|------|------|-------|-------|
| `ReadOnly` | Yes | No | `__I` |
| `WriteOnly` | No | Yes | `__O` |
| `ReadWrite` | Yes | Yes | `__IO` |

## Directory Structure

```
fsnative-spec/
├── spec/                    # Core specification chapters
│   ├── Catalog.json        # Chapter ordering
│   └── ...                 # Various spec chapters
├── docs/fidelity/          # Fidelity-specific extensions
│   ├── Native_Type_Universe.md  # Complete type spec
│   ├── FNCS_Specification.md    # Compiler services spec
│   ├── Beyond_FSlang_Spec.md    # Historical context
│   └── FSharp_Features_In_Fidelity.md
├── releases/               # Versioned spec releases
└── README.md
```

## Relationship to Other Projects

| Project | Relationship |
|---------|--------------|
| **FNCS** | Implements this specification |
| **FSNAC** | IDE services using FNCS |
| **Firefly** | Compiles according to spec |
| **Alloy** | Native library (temporary shadow types to be removed) |

## Normative Language

Uses RFC 2119 keywords:
- **SHALL/MUST**: Absolute requirement
- **SHALL NOT/MUST NOT**: Absolute prohibition
- **SHOULD**: Recommended
- **MAY**: Optional

## Phase Boundaries

- **Phase A (NOW)**: Type Universe - fsnative types, **UMX memory regions**, **fsil inline semantics**, FNCS, Ionide toolchain
- **Phase B (AFTER QC PoC)**: Delimited continuations, interaction net decomposition
