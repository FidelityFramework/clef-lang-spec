# fsnative-spec Project Overview

## Purpose

Normative specification for F# Native compilation. Defines type semantics, memory layout, and platform bindings for native F# without .NET runtime.

## Repository Structure

```
fsnative-spec/
├── spec/                    # THE SPECIFICATION (authoritative)
│   ├── Catalog.json         # Chapter ordering
│   └── *.md                 # Spec chapters
├── docs/fidelity/           # TEMPORARY working context (will be removed)
│   └── *.md                 # Reference docs during spec development
├── releases/                # Versioned spec snapshots
└── README.md                # Project overview
```

## Spec Chapters (spec/)

### Stable (from fslang-spec, minimal changes needed)
- `lexical-analysis.md`
- `basic-grammar-elements.md`
- `patterns.md`
- `units-of-measure.md`
- `lexical-filtering.md`
- `features-for-ml-compatibility.md`

### Needs Revision (BCL assumptions to remove)
- `introduction.md` - Remove VS/fsc.exe/.NET references
- `types-and-type-constraints.md` - Native type semantics
- `program-structure-and-execution.md` - Native entry points
- `special-attributes-and-types.md` - Remove BCL attributes

### New fsnative-Specific Chapters
- `native-type-mappings.md` - F# syntax → native representation
- `memory-regions.md` - Stack/Arena/Peripheral/Sram/Flash
- `access-kinds.md` - ReadOnly/WriteOnly/ReadWrite
- `platform-bindings.md` - Platform.Bindings convention

### Rewritten
- `the-native-library-alloy.md` - Was FSharp.Core content, now Alloy

### Removed (BCL-dependent, not applicable)
- `provided-types.md` - Type providers require .NET
- `custom-attributes-and-reflection.md` - System.Reflection not available

## docs/fidelity/ (TEMPORARY)

These are working reference documents that will be removed once spec is complete:
- `native-type-universe.md` - Comprehensive type reference
- `fncs-specification.md` - FNCS compiler services
- `beyond-fslang-spec.md` - Historical context
- `fsharp-features-in-fidelity.md` - Feature matrix
- `README.md` - docs overview

**DO NOT** include docs/ content in spec/. They are context only.

## Key Principles

1. **Same syntax, native semantics** - Users write `string`, `option`, `array`; FNCS resolves to native types
2. **No BCL dependencies** - No System.*, no reflection, no GC
3. **Compiler-controlled memory** - Fidelity makes ALL layout decisions
4. **Platform bindings via module convention** - Not P/Invoke
