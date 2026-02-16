# clef-lang-spec Project Overview

## Purpose

Normative specification for Clef compilation. Defines type semantics, memory layout, and platform bindings for native F# without .NET runtime.

## Repository Structure

```
clef-lang-spec/
├── spec/                    # THE SPECIFICATION (authoritative)
│   ├── Catalog.json         # Chapter ordering
│   └── *.md                 # Spec chapters
├── docs/                    # Empty (docs/fidelity moved to Clef)
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

### New Clef-Specific Chapters
- `native-type-mappings.md` - F# syntax → native representation
- `memory-regions.md` - Stack/Arena/Peripheral/Sram/Flash
- `access-kinds.md` - ReadOnly/WriteOnly/ReadWrite
- `platform-bindings.md` - Platform.Bindings convention

### Rewritten
- `the-native-library-alloy.md` - Was FSharp.Core content, now Alloy

### Removed (BCL-dependent, not applicable)
- `provided-types.md` - Type providers require .NET
- `custom-attributes-and-reflection.md` - System.Reflection not available

## docs/fidelity/ (REMOVED)

The docs/fidelity folder has been removed. Content was migrated:
- `native-type-universe.md` → Moved to `Clef/docs/fidelity/`
- `ccs-specification.md` → Moved to `Clef/docs/fidelity/`
- `beyond-fslang-spec.md` → Extracted to Serena memory `design_philosophy`
- `fsharp-features-in-fidelity.md` → Extracted to Firefly memory `fsharp_metaprogramming_patterns`
- `README.md` → Deleted (navigation no longer needed)

**DO NOT** include docs/ content in spec/. They are context only.

## Structural Audit Status: COMPLETE (Dec 2025)

All structural changes verified:
- README.md updated with ToC, numbered chapter table, status key
- Catalog.json synchronized with 21 chapters
- New chapters created: memory-regions, access-kinds, native-type-mappings, platform-bindings
- Removed chapters deleted: provided-types, custom-attributes-and-reflection
- Alloy chapter (the-native-library-alloy.md) completely rewritten

## Key Principles

1. **Same syntax, native semantics** - Users write `string`, `option`, `array`; CCS resolves to native types
2. **No BCL dependencies** - No System.*, no reflection, no GC
3. **Compiler-controlled memory** - Fidelity makes ALL layout decisions
4. **Platform bindings via module convention** - Not P/Invoke
