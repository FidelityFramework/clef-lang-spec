# F# Native Language Specification

## Overview

This repository contains the normative specification for **F# Native**, a dialect of F# designed for native compilation without runtime dependencies. F# Native extends standard F# semantics with native type resolution, memory region types, and hardware access primitives.

**F# Native is NOT a new language.** It is F# with:
- Native types instead of BCL types
- SRTP resolution against Alloy witnesses instead of .NET method tables
- Memory region and access kind enforcement
- Zero-cost peripheral access patterns

## Relationship to Standard F#

F# Native is a **superset of F# syntax** with **modified type semantics**:

```fsharp
// This code is valid in both F# and F# Native
let greeting = "Hello, World!"
let numbers = [| 1; 2; 3 |]
let maybe = Some 42

// In standard F#:
//   greeting : System.String
//   numbers  : System.Int32[]
//   maybe    : int option (reference type, None = null)

// In F# Native:
//   greeting : NativeStr (fat pointer to UTF-8)
//   numbers  : NativeArray<int> (fat pointer)
//   maybe    : int voption (value type, non-nullable)
```

## Relationship to Fidelity Ecosystem

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

## Document Organization

### Core Specification

| Document | Contents |
|----------|----------|
| [FNCS_Specification.md](FNCS_Specification.md) | Complete normative specification |

### Specification Parts

| Part | Title | Status |
|------|-------|--------|
| Part 1 | Native Type Universe | Specified |
| Part 2 | Null-Free Semantics | Specified |
| Part 3 | SRTP Resolution | Specified |
| Part 4 | Memory Semantics | Draft |
| Part 5 | Coeffects | Draft |
| Part 6 | Platform Bindings | Specified |
| Part 7 | Compatibility | Specified |
| Part 8 | Diagnostics | Specified |
| Part 9 | Memory Region Types | Specified |
| Part 10 | Access Kind Enforcement | Specified |
| Part 11 | Peripheral Descriptors | Specified |
| Part 12 | Ownership/Coeffects | Reserved (Future) |

### Appendices

| Appendix | Contents |
|----------|----------|
| A | Type Mapping Reference |
| B | Grammar Extensions (Future) |
| C | Specification Status |
| D | Native-Specific Diagnostics (FS8xxx) |

## Normative vs Informative

The specification uses RFC 2119 keywords:

- **SHALL/MUST**: Absolute requirement
- **SHALL NOT/MUST NOT**: Absolute prohibition
- **SHOULD**: Recommended but not required
- **MAY**: Optional

Example:
> NORMATIVE: String literals SHALL have type `NativeStr`, not `System.String`.

## Key Concepts

### Native Type Universe

F# Native replaces BCL types with native equivalents:

| F# Syntax | F# Native Type | Why |
|-----------|----------------|-----|
| `string` | `NativeStr` | UTF-8, fat pointer, no GC |
| `option<'T>` | `voption<'T>` | Value type, non-nullable |
| `'T[]` | `NativeArray<'T>` | Fat pointer, explicit lifetime |

### Memory Regions

Hardware memory has different characteristics:

| Region | Use Case | Volatile | Cacheable |
|--------|----------|----------|-----------|
| `Peripheral` | Memory-mapped I/O | Yes | No |
| `SRAM` | General RAM | No | Yes |
| `Flash` | Read-only storage | No | Yes |
| `Stack` | Thread-local | No | Yes |

### Access Kinds

Pointers carry access permissions:

| Kind | Read | Write | CMSIS |
|------|------|-------|-------|
| `ReadOnly` | Yes | No | `__I` |
| `WriteOnly` | No | Yes | `__O` |
| `ReadWrite` | Yes | Yes | `__IO` |

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

## Related Resources

| Resource | Description |
|----------|-------------|
| [fsnative](https://github.com/user/fsnative) | FNCS implementation |
| [Firefly](https://github.com/user/Firefly) | Native compilation pipeline |
| [Alloy](https://github.com/user/Alloy) | Native F# standard library |
| [BAREWire](https://github.com/user/BAREWire) | Memory-efficient IPC |
| [Farscape](https://github.com/user/Farscape) | C/C++ binding generator |
| [fslang-spec](https://github.com/fsharp/fslang-spec) | Standard F# specification |

## Contributing

Specification changes require:
1. Discussion of semantic implications
2. Coordination with FNCS implementation
3. Validation via Firefly compilation

Changes to normative sections must include:
- Rationale for the change
- Impact analysis on existing code
- Implementation plan for FNCS
