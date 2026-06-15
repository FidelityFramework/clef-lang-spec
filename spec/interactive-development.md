---
title: "Interactive Development"
weight: 350
---

This chapter specifies the interactive development experience for Clef, including the Clef Interactive environment (clefx), script execution, and integration with development tooling.

## Overview

Clef provides an interactive development experience comparable to F# Interactive (FSI) in managed F#. The goal is to give Clef developers the same exploratory, REPL-driven workflow that .NET developers expect, while respecting the constraints of native compilation.

| Aspect | F# Interactive (FSI) | Clef Interactive (clefx) |
|--------|---------------------|------------------------------|
| Execution | CLR JIT | Native (LLVM JIT / AOT) |
| Memory | GC-managed | Deterministic (arena-based) |
| Script extension | `.fsx` | `.clefx` |
| Entry command | `dotnet fsi` | `clefx` |
| Directive prefix | `#r`, `#load` | `#require`, `#load` |

## Design Principles

1. **Familiar Experience**: Developers moving from managed F# should find clefx familiar
2. **Native Semantics**: Interactive execution follows native memory and type semantics
3. **Tooling Integration**: clefx integrates with the same LSP infrastructure as the compiler
4. **Exploratory Development**: Support rapid prototyping and experimentation
5. **Seamless Transition**: Code developed interactively should compile without modification

## Clef Interactive (clefx)

### Invocation

The Clef Interactive environment is invoked via the `clefx` command:

```bash
# Start interactive session
clefx

# Execute a script file
clefx script.clefx

# Evaluate an expression
clefx --eval "1 + 1"

# With specific platform target
clefx --target linux-x64
```

### Session Model

An clefx session maintains:

- A global environment of bound values and types
- An arena for interactive allocations
- A history of evaluated expressions
- Loaded modules and dependencies

```
Clef Interactive (clefx) v1.0
Target: linux-x64
Arena: 64MB (expandable)

> let x = 42;;
val x : int = 42

> let greet name = $"Hello, {name}!";;
val greet : string -> string

> greet "World";;
val it : string = "Hello, World!"
```

### Execution Model

clefx supports multiple execution strategies:

#### Interpretation Mode (Default)

Expressions are interpreted without full native compilation. This provides:
- Fast feedback for simple expressions
- Lower latency than full compilation
- Suitable for exploration and prototyping

```
> #mode interpret;;
Execution mode: interpret

> [1..1000] |> List.map (fun x -> x * x);;
val it : int list = [1; 4; 9; 16; ...]
```

#### Compilation Mode

Expressions are compiled to native code and executed. This provides:
- Accurate performance characteristics
- Full optimization
- Behavior identical to compiled programs

```
> #mode compile;;
Execution mode: compile (target: linux-x64)

> let rec fib n = if n < 2 then n else fib (n-1) + fib (n-2);;
val fib : int -> int
-- Compiled to native code

> #time on;;
> fib 40;;
Real: 00:00:00.892
val it : int = 102334155
```

#### Hybrid Mode

The default for production use. Simple expressions are interpreted; complex definitions are compiled.

```
> #mode hybrid;;
Execution mode: hybrid

> let x = 1 + 1;;          // Interpreted
val x : int = 2

> let rec factorial n =    // Compiled (recursive)
      if n <= 1 then 1
      else n * factorial (n - 1);;
val factorial : int -> int
```

#### Self-Hosted Execution (target architecture)

> **Forward-looking**: This subsection describes the intended model once Clef is self-hosted; it is
> not yet normative.

When Clef is self-hosted, `clefx` runs as an **actor within the CLI tool environment** rather than as
a separate runtime process. The actor performs all compilation work on the CPU — lexing, dependency
analysis, type checking, and lowering — and drives an **LLVM JIT** that emits native machine code
into executable memory. Entered expressions and definitions are *exercised* by invoking that
JIT-resident code directly and reporting results back to the prompt. There is no managed runtime, no
reflection, and no bytecode interpreter on this path: the same compilation pipeline that produces
ahead-of-time binaries produces the JIT-resident code, so interactive behavior matches compiled
behavior by construction. This is the operational meaning of the `x` in `clefx` — Clef made directly
executable — and the substantive break from F# Interactive, whose evaluation bounced off the managed
runtime via JIT and reflection.

## Script Files

### File Extension

Clef interactive files use the `.clefx` extension:

```
script.clefx     -- Clef interactive (REPL/script) file
module.clef      -- Clef implementation file
```

> **Rationale**: The `.clefx` extension — shared with the `clefx` command — marks the file as Clef
> made *executable* in a REPL/script workflow. The `x` is a deliberate point of departure from F#'s
> `fsi`/`.fsx`: F# Interactive bounced expressions off the managed runtime, relying on JIT and
> reflection against the CLR. `clefx` has no managed runtime and no reflection; interactive code is
> compiled to native machine code and executed directly (see [§](#execution-model)). The `x`
> denotes that native-executable character, paralleling `.fsx` in role while breaking with its
> runtime model. Clef deliberately avoids a `.clefi`-style name: there is no Clef interface file, and
> an `i` suffix would invite exactly that misreading. Module signatures are declared inline within
> implementation files (see [Namespace and Module Signatures](namespace-and-module-signatures.md));
> there is no separate signature-file extension.

### Script Structure

A script file contains a sequence of declarations and expressions:

```fsharp
// script.clefx
#load "helpers.clef"

open Console

let data = [1; 2; 3; 4; 5]
let sum = List.fold (+) 0 data

writeln $"Sum: {sum}"
```

### Script Directives

| Directive | Description |
|-----------|-------------|
| `#require "name"` | Load a package dependency |
| `#load "file.clef"` | Load and compile a Clef source file |
| `#load "file.clefx"` | Load and execute another script |
| `#time "on"` \| `"off"` | Toggle timing display |
| `#mode interpret` \| `compile` \| `hybrid` | Set execution mode |
| `#arena size` | Set arena size (e.g., `#arena 128MB`) |
| `#target platform` | Set target platform |
| `#help` | Display help |
| `#quit` | Exit the session |

### Shebang Support

Script files may include a shebang for direct execution:

```fsharp
#!/usr/bin/env clefx
// script.clefx

open Console
writeln "Hello from Clef!"
```

```bash
chmod +x script.clefx
./script.clefx
```

## Memory Model in Interactive Mode

### Arena-Based Allocation

Interactive sessions use arena-based memory management:

```
> #arena 64MB;;
Arena size: 64MB

> let bigList = [1..1000000];;
val bigList : int list
-- Allocated in session arena

> #arena status;;
Arena: 12.4MB used of 64MB
```

### Arena Reset

The arena can be reset to reclaim memory:

```
> #arena reset;;
Arena reset. All interactive values invalidated.

> bigList;;
Error: Value 'bigList' is no longer valid after arena reset.
```

### Persistent Values

Values can be marked as persistent to survive arena resets:

```
> #persist let config = loadConfig();;
val config : Config  [persistent]

> #arena reset;;
Arena reset. Persistent values retained.

> config;;  // Still valid
val it : Config = { ... }
```

## Platform Targeting

### Target Selection

clefx can target different platforms:

```
> #target linux-x64;;
Target: linux-x64

> #target linux-arm64;;
Target: linux-arm64

> #target freestanding-arm-none-eabi;;
Target: freestanding-arm-none-eabi
-- Note: Limited library support in freestanding mode
```

### Cross-Compilation in Interactive Mode

When targeting a different platform than the host:

```
> #target linux-arm64;;
Target: linux-arm64 (cross-compiling from linux-x64)
Execution mode: compile-only (cannot execute on host)

> let add x y = x + y;;
val add : int -> int -> int
-- Compiled for linux-arm64, not executed

> add 1 2;;
Warning: Cannot execute arm64 code on x64 host.
Use #target linux-x64 to execute, or #emit to generate binary.

> #emit "add.o";;
Emitted: add.o (linux-arm64)
```

## Value Display Without Runtime Reflection

A fundamental difference between clefx and managed F# Interactive (FSI) is how values are formatted for display.

### The Managed F# Approach

In FSI, value display relies on `obj` and runtime reflection:

```
// Managed FSI internals (simplified)
let displayValue (value: obj) : string =
    sprintf "%A" value  // Uses reflection to inspect value
```

This approach is not available in Clef because:
- There is no universal base type `obj`
- There is no runtime type information or reflection
- Values cannot be "boxed" to a common representation

### SRTP-Based Value Formatting

Clef uses statically resolved type parameters (SRTP) to generate formatters at compile time:

```fsharp
// clefx generates specific formatters via SRTP
type Displayable = Displayable
    with static member inline ($) (Displayable, x: int) = 
             Text.Format.intToString x
         static member inline ($) (Displayable, x: string) = 
             "\"" + x + "\""
         static member inline ($) (Displayable, xs: 'T list) =
             "[" + (xs |> List.map (fun x -> Displayable $ x) 
                       |> String.concat "; ") + "]"
         // ... additional overloads for all displayable types

let inline display x = Displayable $ x
```

When you enter an expression in clefx:

```
> [1; 2; 3];;
```

The system:
1. Type-checks the expression (determines `int list`)
2. Generates a display function via SRTP resolution
3. Compiles both expression and display function
4. Executes and formats the result

### Implications for Custom Types

User-defined types require explicit display support:

```fsharp
> type Point = { X: float; Y: float };;
type Point = { X: float; Y: float }

> { X = 1.0; Y = 2.0 };;
val it : Point = { X = 1.0; Y = 2.0 }
-- Display generated from record field structure
```

For types requiring custom formatting:

```fsharp
> type Point = { X: float; Y: float }
      with static member ($) (Displayable, p: Point) =
               $"({p.X}, {p.Y})";;

> { X = 1.0; Y = 2.0 };;
val it : Point = (1.0, 2.0)
```

### Format Specifiers

The `%A` format specifier in Clef uses SRTP rather than reflection:

```fsharp
> printfn "%A" [1; 2; 3];;
[1; 2; 3]
-- SRTP resolves formatting at compile time
```

> **Clef Note**: Format specifiers `%A` and `%O` are resolved at compile time via SRTP. Types must have appropriate formatting members resolvable statically. See [Native Type Mappings](native-type-mappings.md#the-universal-base-type-obj-is-not-available).

## Tooling Architecture

### Parallel Toolchain Model

Clef uses a parallel toolchain rather than extending managed F# tooling:

| Component | Managed F# | Clef |
|-----------|------------|-----------|
| Compiler Services | FCS (F# Compiler Services) | CCS (Clef Compiler Service) |
| Language Server | FSAC (F# AutoComplete) | FSNAC (Clef AutoComplete) |
| Package Manager | NuGet | Fargo (fpm) |
| Project Format | `.fsproj` (MSBuild) | `.fidproj` (TOML) |
| Package Format | `.nupkg` (binary) | `.fidpkg` (source) |
| Interactive | FSI | clefx |
| Script Files | `.fsx` | `.clefx` |

### Why Parallel Rather Than Plugin

The toolchains are parallel rather than plugins because:

1. **Type resolution fundamentally differs**: CCS resolves `string` to native UTF-8 fat pointer semantics; FCS resolves to `System.String`. These cannot be reconciled at runtime.

2. **SRTP resolution differs**: CCS resolves SRTP against native type witnesses; FCS resolves against BCL method tables.

3. **No `obj` escape hatch**: Managed tooling uses `obj` as a universal container for values during type checking and display. Clef has no such type.

4. **Source-based packages**: Fargo distributes source code for whole-program optimization. NuGet distributes compiled binaries.

### Coexistence with Fable and Managed F#

Clef tooling is designed to coexist with other F# targets in a single workspace:

```
my-project/
├── web-ui/                 # Fable → JavaScript
│   ├── App.fsproj         # ← FSAC handles this
│   └── Components.fs
├── native-backend/         # Clef → Native binary
│   ├── Server.fidproj     # ← FSNAC handles this
│   └── Api.clef
└── shared/                 # Pure F# domain types
    ├── Shared.fsproj      # ← Both can consume (with constraints)
    └── Domain.fs
```

IDE integration (Lattice) routes to the appropriate language server based on project type:

- `.fsproj` → FSAC (managed F# or Fable)
- `.fidproj` → FSNAC (Clef)

### Shared Code Constraints

Code shared between Fable and Clef must avoid:

| Feature | Fable | Clef | Sharable? |
|---------|-------|-----------|-----------|
| Pure functions | ✅ | ✅ | ✅ |
| Records, DUs | ✅ | ✅ | ✅ |
| `string` operations | `System.String` | `NativeStr` | ❌ |
| `option` | Reference type | `voption` | ❌ |
| `printf "%A"` | Reflection | SRTP | ⚠️ |
| Async | `Async<'T>` | Native async | ❌ |
| Reflection | Available | Not available | ❌ |

Shared code should be restricted to pure domain modeling without IO or string manipulation.

## Package Management Integration

### Fargo Integration

clefx integrates with Fargo for package management:

```
> #require "robot-controller";;\n-- Resolving robot-controller from frgo.dev...\n-- Downloaded: robot-controller-1.2.0.fidpkg\n-- Source files: 12\n-- Compiling for current session...\nLoaded: RobotController (3 modules)

> open RobotController.Algorithms;;
> PIDController.create 1.0 0.1 0.05;;
val it : PIDController = { Kp = 1.0; Ki = 0.1; Kd = 0.05 }
```

### Source-Based Loading

Unlike NuGet which loads pre-compiled binaries, Fargo loads source:

```
> #require "crypto-algorithms";;\n-- Source package: crypto-algorithms-2.0.0\n-- Compiling with current target optimizations...\n-- Inlining enabled across package boundary\nLoaded: CryptoAlgorithms
```

This enables:
- Cross-package inlining
- Target-specific optimization
- Dead code elimination
- Consistent native semantics

### Local Package Development

For local package development:

```
> #require "path: ../my-local-package";;
-- Loading from: /home/dev/my-local-package
-- Watching for changes...
Loaded: MyLocalPackage

> // Edit my-local-package source files...

> #reload;;
-- Reloading changed packages...
-- MyLocalPackage: 2 files changed
Reloaded: MyLocalPackage
```

## Tooling Integration

### LSP Integration

clefx connects to the Clef Language Server (FSNAC) for:

- Autocompletion in the REPL
- Type information on hover
- Error diagnostics
- Go-to-definition for loaded modules

```
> List.ma<TAB>
  List.map        : ('a -> 'b) -> 'a list -> 'b list
  List.map2       : ('a -> 'b -> 'c) -> 'a list -> 'b list -> 'c list
  List.mapi       : (int -> 'a -> 'b) -> 'a list -> 'b list
```

### Editor Integration

Editors supporting Clef (via Lattice or similar) provide:

- Syntax highlighting for `.clefx` files
- Inline evaluation (evaluate selection in clefx)
- Hover types
- Error underlining
- Send-to-REPL functionality

### Notebook Support

clefx supports notebook interfaces (Jupyter, Polyglot Notebooks):

```json
{
  "kernelspec": {
    "name": "clef",
    "display_name": "Clef",
    "language": "fsharp"
  }
}
```

## Web Playground

### Online Interactive Environment

A web-based playground provides interactive Clef execution:

```
try.fidelity.dev
```

Features:
- Browser-based editor with syntax highlighting
- Server-side compilation and execution
- Shareable code snippets
- Example gallery
- Platform target selection

### Playground Limitations

The web playground operates with restrictions:

| Feature | Availability |
|---------|--------------|
| Core language | Full |
| Native library | Full |
| File I/O | Sandboxed |
| Network | Restricted |
| Custom native code | Not available |
| Execution time | Limited (30 seconds) |
| Memory | Limited (256MB arena) |

## Interoperability

### Loading Compiled Modules

clefx can load pre-compiled Clef modules:

```
> #load-native "mylib.fno";;
Loaded: MyLib (5 modules, 42 functions)

> MyLib.Utilities.process data;;
val it : Result<Data, Error> = Ok { ... }
```

### FFI in Interactive Mode

Foreign function interfaces work in compile mode:

```
> #mode compile;;

> [<PlatformBinding>]
  module Native =
      let puts : string -> int = native "puts";;

> Native.puts "Hello from C!";;
Hello from C!
val it : int = 14
```

## Diagnostics

| Code | Severity | Message |
|------|----------|---------|
| CCS8500 | Error | Cannot execute cross-compiled code on host platform |
| CCS8501 | Warning | Value invalidated by arena reset |
| CCS8502 | Error | Arena size exceeded; use `#arena reset` or increase size |
| CCS8503 | Warning | Interpretation mode may not reflect exact native behavior |
| CCS8504 | Info | Expression compiled to native code |
| CCS8505 | Error | Freestanding target has limited library support |

## Grammar

```fsgrammar
script-file :=
    shebang-line? script-directive* module-elems

shebang-line :=
    #! filepath newline

script-directive :=
    # require string
    # load string
    # time on-off
    # mode exec-mode
    # arena arena-spec
    # target platform-spec
    # persist let-binding
    # help
    # quit

exec-mode :=
    interpret
    compile
    hybrid

arena-spec :=
    size-literal
    reset
    status

on-off :=
    "on"
    "off"
```

## Comparison with Other Native REPLs

| Feature | clefx | utop (OCaml) | evcxr (Rust) | Swift REPL |
|---------|------|--------------|--------------|------------|
| Official | Yes | Community | Community | Yes |
| Execution | Hybrid | Bytecode | Compile | JIT |
| Memory model | Arena | GC | Ownership | ARC |
| Script files | `.clefx` | `.ml` | N/A | `.swift` |
| Notebooks | Yes | Yes | Yes | Yes (Playgrounds) |
| Cross-compile | Yes | Limited | No | No |

## Areas Requiring Further Specification

1. **Interpretation semantics**: Exact behavior of interpreter vs compiled code
2. **Arena lifecycle**: Interaction between multiple scripts and arena management
3. **Debugging**: Breakpoints and stepping in interactive mode
4. **Profiling**: Performance analysis tools in clefx
5. **Package management**: Integration with a native package manager
6. **Caching**: Compilation caching for faster repeated execution
7. **State serialization**: Saving and restoring session state

## See Also

- [Program Structure and Execution](program-structure-and-execution.md) - Compiled program execution
- [Memory Regions](memory-regions.md) - Arena memory model
- [Error Handling](error-handling.md) - Tooling integration model
- [Platform Bindings](platform-bindings.md) - FFI in interactive mode
