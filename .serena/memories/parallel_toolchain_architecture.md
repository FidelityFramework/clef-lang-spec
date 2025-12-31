# Parallel Toolchain Architecture

## Core Principle

F# Native uses a **parallel toolchain** rather than plugins/extensions to managed F# tooling. This is not a preference but a necessity driven by fundamental architectural differences.

## The Parallel Stack

| Layer | Managed F# | F# Native | Notes |
|-------|------------|-----------|-------|
| **Compiler Services** | FCS | FNCS | Different type resolution |
| **Language Server** | FSAC | FSNAC | Different value handling |
| **Package Manager** | NuGet | Fargo (fpm) | Binary vs source packages |
| **Registry** | nuget.org | frgo.dev | Different package format |
| **Project Format** | `.fsproj` | `.fidproj` | MSBuild vs TOML |
| **Package Format** | `.nupkg` | `.fidpkg` | Compiled vs source |
| **Interactive** | FSI | fsni | GC vs arena, reflection vs SRTP |
| **Script Files** | `.fsx` | `.fsnx` | Different execution model |

## Why Parallel is Necessary

### 1. Type Resolution Differs Fundamentally

```
FCS: "string" → System.String (BCL type, assembly metadata)
FNCS: "string" → NativeStr (intrinsic, UTF-8 fat pointer)
```

These cannot be reconciled. FCS has BCL types hard-coded internally.

### 2. No `obj` Escape Hatch

Managed tooling uses `obj` pervasively:
- Universal value container in type checker
- FSI stores evaluation results as `obj`
- `%A` formatting uses reflection on `obj`
- Hover info boxes values for display

F# Native has no `obj`. Every value must be typed statically.

### 3. SRTP Resolution Differs

```
FCS: SRTP resolves against BCL method tables (assembly metadata)
FNCS: SRTP resolves against Alloy intrinsic witnesses
```

The resolution sources are completely different.

### 4. Package Distribution Model

```
NuGet: Compiled assemblies (binary distribution)
Fargo: Source packages (whole-program optimization)
```

Source-based distribution enables cross-package inlining, dead code elimination, and platform-specific optimization.

## Coexistence Model

The parallel toolchains coexist in the same workspace via project-type routing:

```
Ionide (or equivalent IDE)
         │
    ┌────┴────┐
    │ Detect  │
    │ project │
    │  type   │
    └────┬────┘
         │
    ┌────┴────┐
    │         │
.fsproj    .fidproj
    │         │
    ▼         ▼
  FSAC      FSNAC
```

### Multi-Target Workspace Example

```
my-fidelity-app/
├── web-ui/                 # Fable → JavaScript
│   ├── App.fsproj         # FSAC
│   └── Components.fs
├── native-backend/         # F# Native → Native
│   ├── Server.fidproj     # FSNAC
│   └── Api.fs
└── shared/                 # Constrained pure F#
    ├── Shared.fsproj      # Can be consumed by both
    └── Domain.fs          # (with restrictions)
```

### Shared Code Constraints

Code shared between Fable and F# Native must be pure computation:
- ✅ Records, DUs (without custom ToString)
- ✅ Pure functions
- ❌ String manipulation (different types)
- ❌ Async/Task (different models)
- ❌ Reflection (not available in native)
- ❌ Printf %A with obj (different implementation)

## Ionide Integration Path

1. Fork Ionide projects for F# Native support
2. Implement project-type detection (`.fidproj`)
3. Route to FSNAC instead of FSAC
4. Upstream improvements for plugin/parallel-silo infrastructure
5. Enable same-workspace coexistence with Fable

## Key Documentation

- `spec/interactive-development.md` - fsni and tooling architecture
- `spec/native-type-mappings.md` - `obj` elimination rationale
- `docs/Fidelity_Tooling_Roadmap.md` (Firefly) - FSNAC design
- Blog: "Fargo: Native F# Source-Based Package Management"

## Related Decisions

- `obj` elimination (`obj_elimination` memory)
- `voption` for optional values
- SRTP-based polymorphism
- Source-based package distribution
