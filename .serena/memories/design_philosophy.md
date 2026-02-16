# clef-lang-spec Design Philosophy

## Engineering-First Methodology

clef-lang-spec was **not** designed from academic first principles and then implemented. The specification **emerged from implementation**.

> "The specification documents what was discovered through implementation, not what was decreed in advance. This engineering-first methodology means that every normative statement in clef-lang-spec corresponds to a concrete implementation requirement that arose from making actual code compile and execute correctly."

## Why clef-lang-spec Exists

The F# Language Specification delegates extensively to the .NET runtime:

| What fslang-spec Delegates | Why Native Needs Explicit Definition |
|---------------------------|-------------------------------------|
| Primitive types → BCL types | No BCL exists; must define `int`, `string` etc. natively |
| Memory layout | CLR handles it; native must specify size, alignment, padding |
| Memory management | GC handles it; native needs explicit strategies |
| Platform abstraction | BCL handles it; native needs binding mechanisms |

**Native compilation fills these gaps explicitly.** There is no runtime to defer to.

## OCaml "Accidental Sympathy" Discovery

The relationship between Clef and OCaml was **discovered, not designed**.

Initial work on Alloy involved creating shadow types to intercept BCL type resolution:
- `NativeStr` was created because `System.String` couldn't exist runtime-free
- `voption` was adopted for null-freedom (already in FSharp.Core)

**After implementing several shadow types, a pattern emerged**: they bore striking resemblance to OCaml's native types.

| Aspect | BCL F# | OCaml | Clef |
|--------|--------|-------|-----------|
| String encoding | UTF-16 | UTF-8 | UTF-8 |
| Option type | Reference, nullable | Value | Value (voption) |
| Record default | Reference | Value possible | Struct |
| Integer type | Fixed 32-bit | Native width | Platform word |

> "This accidental sympathy revealed something fundamental about ML-family languages: when you remove the managed runtime, the natural type representations that emerge are value-oriented and memory-explicit."

## Influences and Adaptations

### F* HyperStack Memory Model

F*'s HyperStack concepts map to Clef's region system:
- Region identifiers via phantom type parameters
- Containment hierarchy with stack frames and heap regions
- Preorders for value evolution constraints
- Witnessed predicates for resource availability

### Rust RAII Guideposts

Rust pioneered compile-time ownership tracking. Clef reserves syntax for similar concepts, adapted to F# idioms:
- `Owned<'T>`, `Borrowed<'T>` for ownership
- `move`, `&`, `&mut` expressions
- Coeffect-based effect tracking rather than explicit lifetime annotations

The goal: have the compiler deal with ownership concerns without design-time "interference" like Rust's borrow checker.

### C/C++ Hardware Access

For low-level hardware access:
- CMSIS conventions (`__I`, `__O`, `__IO` volatile qualifiers)
- Structure layout control (`[<Struct>]`, packing, alignment)
- Pointer arithmetic (explicit, type-safe operations)

## Core Principles

1. **Same Syntax, Native Semantics**: Users write `string`, `option`, `int` - familiar F# syntax. clef-lang-spec defines what those types *mean* for native compilation.

2. **Absolute Null-Freedom**: Everything is non-nullable by construction. Use `voption<'T>` for optional values.

3. **Compile-Time Type Safety**: No runtime type discrimination. All types resolved statically.

4. **Memory Region Awareness**: Stack, Arena, Peripheral, SRAM, Flash - distinct regions with different semantics.

5. **Access Kind Enforcement**: ReadOnly, WriteOnly, ReadWrite - compile-time verification of hardware access patterns.

## What clef-lang-spec Is NOT

- **Not a subset**: Same syntax with different semantics and new capabilities
- **Not a replacement**: fslang-spec targets CLR; clef-lang-spec targets native
- **Not a formal semantics**: Prose specification, not operational/denotational semantics

## Source Document

Extracted from: `clef-lang-spec/docs/fidelity/beyond-fslang-spec.md` (now removed)
