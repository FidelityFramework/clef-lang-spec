# Interactive Development

This chapter specifies the interactive development experience for F# Native, including the F# Native Interactive environment (fsni), script execution, and integration with development tooling.

## Overview

F# Native provides an interactive development experience comparable to F# Interactive (FSI) in managed F#. The goal is to give F# Native developers the same exploratory, REPL-driven workflow that .NET developers expect, while respecting the constraints of native compilation.

| Aspect | F# Interactive (FSI) | F# Native Interactive (fsni) |
|--------|---------------------|------------------------------|
| Execution | CLR JIT | Native interpretation or AOT |
| Memory | GC-managed | Deterministic (arena-based) |
| Script extension | `.fsx` | `.fsnx` |
| Entry command | `dotnet fsi` | `fsni` |
| Directive prefix | `#r`, `#load` | `#require`, `#load` |

## Design Principles

1. **Familiar Experience**: Developers moving from managed F# should find fsni familiar
2. **Native Semantics**: Interactive execution follows native memory and type semantics
3. **Tooling Integration**: fsni integrates with the same LSP infrastructure as the compiler
4. **Exploratory Development**: Support rapid prototyping and experimentation
5. **Seamless Transition**: Code developed interactively should compile without modification

## F# Native Interactive (fsni)

### Invocation

The F# Native Interactive environment is invoked via the `fsni` command:

```bash
# Start interactive session
fsni

# Execute a script file
fsni script.fsnx

# Evaluate an expression
fsni --eval "1 + 1"

# With specific platform target
fsni --target linux-x64
```

### Session Model

An fsni session maintains:

- A global environment of bound values and types
- An arena for interactive allocations
- A history of evaluated expressions
- Loaded modules and dependencies

```
F# Native Interactive (fsni) v1.0
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

fsni supports multiple execution strategies:

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

## Script Files

### File Extension

F# Native script files use the `.fsnx` extension:

```
script.fsnx      -- F# Native script
module.fs        -- F# Native implementation file
signature.fsi    -- F# Native signature file
```

> **Rationale**: Using a distinct extension (`.fsnx` rather than `.fsx`) clearly identifies scripts intended for native execution and avoids confusion with managed F# scripts.

### Script Structure

A script file contains a sequence of declarations and expressions:

```fsharp
// script.fsnx
#require "Alloy"
#load "helpers.fs"

open Alloy.Console

let data = [1; 2; 3; 4; 5]
let sum = List.fold (+) 0 data

WriteLine $"Sum: {sum}"
```

### Script Directives

| Directive | Description |
|-----------|-------------|
| `#require "name"` | Load a package dependency |
| `#load "file.fs"` | Load and compile an F# source file |
| `#load "file.fsnx"` | Load and execute another script |
| `#time "on"` \| `"off"` | Toggle timing display |
| `#mode interpret` \| `compile` \| `hybrid` | Set execution mode |
| `#arena size` | Set arena size (e.g., `#arena 128MB`) |
| `#target platform` | Set target platform |
| `#help` | Display help |
| `#quit` | Exit the session |

### Shebang Support

Script files may include a shebang for direct execution:

```fsharp
#!/usr/bin/env fsni
// script.fsnx

open Alloy.Console
WriteLine "Hello from F# Native!"
```

```bash
chmod +x script.fsnx
./script.fsnx
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

fsni can target different platforms:

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

## Tooling Integration

### LSP Integration

fsni connects to the F# Native Language Server for:

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

Editors supporting F# Native (via Ionide or similar) provide:

- Syntax highlighting for `.fsnx` files
- Inline evaluation (evaluate selection in fsni)
- Hover types
- Error underlining
- Send-to-REPL functionality

### Notebook Support

fsni supports notebook interfaces (Jupyter, Polyglot Notebooks):

```json
{
  "kernelspec": {
    "name": "fsnative",
    "display_name": "F# Native",
    "language": "fsharp"
  }
}
```

## Web Playground

### Online Interactive Environment

A web-based playground provides interactive F# Native execution:

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
| Alloy library | Full |
| File I/O | Sandboxed |
| Network | Restricted |
| Custom native code | Not available |
| Execution time | Limited (30 seconds) |
| Memory | Limited (256MB arena) |

## Interoperability

### Loading Compiled Modules

fsni can load pre-compiled F# Native modules:

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
| FS8500 | Error | Cannot execute cross-compiled code on host platform |
| FS8501 | Warning | Value invalidated by arena reset |
| FS8502 | Error | Arena size exceeded; use `#arena reset` or increase size |
| FS8503 | Warning | Interpretation mode may not reflect exact native behavior |
| FS8504 | Info | Expression compiled to native code |
| FS8505 | Error | Freestanding target has limited library support |

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

| Feature | fsni | utop (OCaml) | evcxr (Rust) | Swift REPL |
|---------|------|--------------|--------------|------------|
| Official | Yes | Community | Community | Yes |
| Execution | Hybrid | Bytecode | Compile | JIT |
| Memory model | Arena | GC | Ownership | ARC |
| Script files | `.fsnx` | `.ml` | N/A | `.swift` |
| Notebooks | Yes | Yes | Yes | Yes (Playgrounds) |
| Cross-compile | Yes | Limited | No | No |

## Areas Requiring Further Specification

1. **Interpretation semantics**: Exact behavior of interpreter vs compiled code
2. **Arena lifecycle**: Interaction between multiple scripts and arena management
3. **Debugging**: Breakpoints and stepping in interactive mode
4. **Profiling**: Performance analysis tools in fsni
5. **Package management**: Integration with a native package manager
6. **Caching**: Compilation caching for faster repeated execution
7. **State serialization**: Saving and restoring session state

## See Also

- [Program Structure and Execution](program-structure-and-execution.md) - Compiled program execution
- [Memory Regions](memory-regions.md) - Arena memory model
- [Error Handling](error-handling.md) - Tooling integration model
- [Platform Bindings](platform-bindings.md) - FFI in interactive mode
