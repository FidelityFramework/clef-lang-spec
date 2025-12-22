# F# Native Language Specification (fsnative-spec)

## Purpose

fsnative-spec defines the normative semantics for native F# compilation in the Fidelity framework. Where the standard F# specification assumes .NET runtime behavior, fsnative-spec explicitly defines:

- Native type layouts and representations
- Memory ownership and borrowing semantics
- Lifetime verification rules
- Deterministic resource management
- Memory region classifications
- Access kind enforcement

## Core Principle

**Same syntax, native semantics.** Developers write standard F# (`string`, `option`, `array`), but the specification defines what these types mean for native compilation:

| F# Syntax | Standard F# | F# Native |
|-----------|-------------|-----------|
| `"Hello"` | `System.String` | `NativeStr` (UTF-8, fat pointer) |
| `Some 42` | `int option` (reference) | `int voption` (value type) |
| `[| 1; 2; 3 |]` | `System.Int32[]` | `NativeArray<int>` (fat pointer) |

## Specification Parts

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
| Part 12 | Ownership/Coeffects | Reserved |

## Key Semantic Additions

### Memory Regions

| Region | Use Case | Volatile | Cacheable |
|--------|----------|----------|-----------|
| `Stack` | Thread-local, automatic | No | Yes |
| `Heap` | Manual or RAII lifetime | No | Yes |
| `Arena` | Bulk allocation | No | Yes |
| `Peripheral` | Memory-mapped I/O | Yes | No |
| `Flash` | Read-only program memory | No | Yes |

### Access Kinds

| Kind | Read | Write | CMSIS |
|------|------|-------|-------|
| `ReadOnly` | Yes | No | `__I` |
| `WriteOnly` | No | Yes | `__O` |
| `ReadWrite` | Yes | Yes | `__IO` |

### Ownership Model

- Every value has exactly one owner
- References can borrow without taking ownership
- Compiler verifies borrows don't outlive owners
- Move semantics or explicit copy

## Directory Structure

```
fsnative-spec/
├── spec/                    # Core specification chapters
│   ├── Catalog.json        # Chapter ordering
│   ├── introduction.md
│   ├── types-and-type-constraints.md
│   ├── the-native-library-alloy.md  # Native library spec
│   └── ...
├── docs/fidelity/          # Fidelity-specific extensions
│   ├── FNCS_Specification.md
│   ├── Beyond_FSlang_Spec.md
│   └── FSharp_Features_In_Fidelity.md
├── releases/               # Versioned spec releases
│   └── FSharpNative-Spec-*.md
└── README.md
```

## Relationship to Other Projects

| Project | Relationship |
|---------|--------------|
| **FNCS** | Implements this specification |
| **FSNAC** | IDE services using FNCS |
| **Firefly** | Compiles according to spec |
| **Alloy** | Native library per spec |

## Normative Language

Uses RFC 2119 keywords:
- **SHALL/MUST**: Absolute requirement
- **SHALL NOT/MUST NOT**: Absolute prohibition
- **SHOULD**: Recommended
- **MAY**: Optional
