---
title: "Program Structure and Execution"
weight: 500
category: Semantics
status: normative
---

> **Clef Note**: Clef programs do not use CLI assemblies. Programs are compiled directly to native binaries from source files, with dependencies resolved at compile time from source packages or pre-compiled native libraries.

A Clef program comprises a set of implementation files (`.clef`), library dependencies specified in the project file (`.fidproj`), and optionally interactive files (`.clefx`) for the Clef Interactive REPL. Implementation files MAY be presented to the compiler in any order; CCS computes the compilation order from a syntactic dependency analysis (see [§](program-structure.md#compilation-pipeline)). There are no separate signature files; module signatures, when present, are declared inline (see [§](namespace-and-module-signatures.md)).

```fsgrammar
implementation-file :=
    namespace-decl-group ... namespace-decl-group
    named-module
    anonymous-module

script-file := implementation-file       -- interactive (.clefx) file, additional directives allowed

named-module :=
    module long-ident module-elems

anonymous-module :=
    module-elems

script-fragment :=
    module-elems                          -- interactively entered code fragment
```

Module signatures may appear inline using the form:

```fsgrammar
signature-decl :=
    signature long-ident? = module-signature-body
    -- inline module signature (placed before the module it describes)
```

The set of implementation files is checked as follows.

1. **Initial environment**. Form an initial environment `env0` by adding all library dependencies in the order specified in the project file. For each dependency, add the top-level types, modules, and namespaces to the environment.

   > **NORMATIVE**: Library imports SHALL be additive only. Clef does not honour the `[<AutoOpen>]` attribute on imported modules; library content becomes available in scope only via explicit `open` declarations in the consuming file.

   > **Clef Note**: The native standard library is automatically included and provides the core types (`string`, `option`, `int`, etc.) with native semantics. See [Native Type Mappings](native-type-mappings.md).

2. **Dependency analysis**. CCS computes the topological order of all implementation files by syntactic dependency analysis (see [§](program-structure.md#compilation-pipeline)). The result is a sequence of compilation units, where each compilation unit is either a single file or a strongly-connected component (a set of files with mutually-recursive references, processed as a single dependency group).

3. **Type checking**. For each compilation unit, in topological order:
    - Check the unit against the current environment `env_{i-1}`, including any inline signatures, to produce the elaborated namespace declaration groups `Impl_unit` and inferred signature `Sig_unit`.
    - If the unit declares an inline `signature ... end` block for a module, check the implementation against the signature ([§](namespace-and-module-signatures.md#signature-conformance)).
    - Add `Impl_unit` (constrained by `Sig_unit` if present) to `env_{i-1}` to produce `env_i`, which becomes the environment for the next compilation unit.

> **NORMATIVE**: There is no concept of "compilation order" that the developer SHALL declare. CCS computes the order from the syntactic dependency graph. Mutually-recursive groups are processed as a single unit; CCS exposes the strongly-connected component structure for inspection and policy enforcement.

## Implementation Files

Implementation files consist of one or more namespace declaration groups. For example:

```fsharp
namespace MyCompany.MyOtherLibrary

    type MyType() =
        let x = 1
        member v.P = x + 2

    module MyInnerModule =
        let myValue = 1

namespace MyCompany. MyOtherLibrary.Collections

    type MyCollection(x : int) =
        member v.P = x
```

An implementation file that begins with a `module` declaration defines a single namespace declaration
group with one module. For example:

```fsharp
module MyCompany.MyLibrary.MyModule

let x = 1
```

is equivalent to:

```fsharp
namespace MyCompany.MyLibrary

module MyModule =
    let x = 1
```

The final identifier in the `long-ident` that follows the `module` keyword is interpreted as the module
name, and the preceding identifiers are interpreted as the namespace.

_Anonymous implementation files_ do not have either a leading `module` or `namespace` declaration. Only
the scripts and the last file within an implementation group for an executable image (.exe) may be
anonymous. An anonymous implementation file contains module definitions that are implicitly
placed in a module. The name of the module is generated from the name of the source file by
capitalizing the first letter and removing the filename extensionIf the filename contains characters
that are not valid in an F# identifier, the resulting module name is unusable and a warning occurs.

Given an initial environment `env0`, an implementation file is checked as follows:

- Create a new constraint solving context.
- Check the namespace declaration groups in the file against the existing environment `envi-1` and
    incrementally add them to the environment ([§](namespaces-and-modules.md#namespace-declaration-groups)) to create a new environment `envi`.
- Apply default solutions to any remaining type inference variables that include `default`
    constraints. The defaults are applied in the order that the type variables appear in the type-
    annotated text of the checked namespace declaration groups.
- Check the inferred signature of the implementation file against any required signature by using
    _Signature Conformance_ ([§](namespace-and-module-signatures.md#signature-conformance)). The resulting signature of an implementation file is the required
    signature, if it is present; otherwise it is the inferred signature.
- Report a “value restriction” error if the resulting signature of any item that is not a member,
    constructor, function, or type function contains any free inference type variables.
- Choose solutions for any remaining type inference variables in the elaborated form of an
    expression. Process any remaining type variables in the elaborated form from left-to-right to
    find a minimal type solution that is consistent with constraints on the type variable. If no unique
    minimal solution exists for a type variable, report an error.

The result of checking an implementation file is a set of elaborated namespace declaration groups.

## Inline Module Signatures

Clef has no separate signature files (see [§](namespace-and-module-signatures.md)). A module's public
surface, when constrained, is given by an inline `signature ... end` block placed immediately before
the `module` declaration it describes. The block belongs to the same implementation file and is
checked together with that file; its grammar is given in
[§](namespace-and-module-signatures.md#signature-elements).

When a compilation unit contains an inline signature for a module, the implementation is checked
against it by _Signature Conformance_
([§](namespace-and-module-signatures.md#signature-conformance)): the resulting signature of the module
is the declared signature when one is present, and otherwise the signature inferred from the
implementation. A type exposed by name only in the signature is abstract to other modules; a type
exposed with its representation reveals that representation (see
[§](namespace-and-module-signatures.md#abstract-type-signatures)).

## Script Files

Script files have the `.clefx` filename extension. They are used by the Clef Interactive REPL for development scenarios and are not directly compiled to native binaries by the Composer compiler. For native compilation, use implementation files (`.clef`) organized via a `.fidproj` project file.

Script files have the following characteristics:

- Side effects in the script are executed immediately in the interactive environment.
- Script files may add other implementation files and script files to the list of sources by using the `#load` directive. Files are processed in the dependency order computed by CCS (see [§](program-structure.md#compilation-pipeline)); the textual position of `#load` directives does not determine compilation order. If a filename appears in more than one `#load` directive, the file is loaded only once.
- Script files may have `#nowarn` directives, which disable a warning for the entire compilation.

The Composer compiler defines the `FIDELITY` compilation symbol for native compilation. The `COMPILED` symbol is also defined for compatibility.

## Compiler Directives

_Compiler directives_ are declarations in non-nested modules or namespace declaration groups in the following form:

```fsgrammar
# id string ... string
```

The lexical preprocessor directives `#if`, `#else`, `#endif` are similar to compiler directives. For details, see [§](lexical-analysis.md#conditional-compilation).

The following directives are valid in all files:

| Directive | Example | Short Description |
| --- | --- | --- |
| `#nowarn` | `#nowarn "54"` | For implementation (`.clef`) files, turns off warnings within this lexical scope. For interactive (`.clefx`) files, turns off warnings globally. |

> **Clef Note**: The `#r` directive for referencing assemblies is not applicable to native compilation. Dependencies are specified in the `.fidproj` project file. The following directives are supported only in the Clef Interactive REPL, not in native compilation:

| Directive | Example | Short Description |
| --- | --- | --- |
| `#load` | `#load "core.clef"` | Loads one or more implementation files into the interactive execution engine. |
| `#time` | `#time "on"` | Enables or disables the display of performance information. |
| `#help` | `#help` | Asks the script execution environment for help. |
| `#quit` | `#quit` | Requests the script execution environment to halt execution and exit. |

## Program Execution

> **Clef Note**: On the native target pathways, execution of Clef code occurs as a standalone native binary, not within a CLI runtime. There is no assembly loading, no JIT compilation, and no garbage collector. Memory management is deterministic and controlled by the compiler: each value's storage is chosen at compile time by its lifetime class under the four-point lifetime lattice (scope-bounded to the stack, region-bounded to a region, program-lifetime to static storage, genuinely-dynamic to the heap), specified in [Closure Representation §3.3](closure-representation.md) and given its region-storage form in [Memory Regions](memory-regions.md). On a target without a heap, only the scope-bounded and program-lifetime classes have a home, and a value that classifies as dynamic is a compile-time lifetime error rather than a silent allocation.
>
> On the JSIR pathway ([Backend Lowering Architecture](backend-lowering-architecture.md)), the artifact is a JavaScript module executed by a host runtime that supplies JIT compilation and garbage collection. The lifetime lattice still classifies every value at design time; the host collector realizes reclamation. The clauses of this chapter that presume a native binary are scoped to the native pathways; the JSIR-pathway entry contract is the module-export mode below.

Execution of Clef code begins when the native binary is loaded by the operating system. During execution, the program can use the functions, values, static members, and object constructors that the compiled modules define.

### Execution of Static Initializers

Each implementation file involves a _static initializer_. In Clef, static initialization is deterministic and occurs at program startup:

- For executables with an explicit entry point function, the static initializers for all files are executed in compilation order before the entry point function is called.
- For executables with an implicit entry point, the static initializer for the last file is the body of the implicit entry point function.

> **Clef Note**: Static initialization is eager and deterministic. All module-level bindings with observable initialization are evaluated at program startup, in compilation order. This provides predictable behavior essential for embedded and real-time systems.

At startup, the static initializer evaluates, in order, the definitions in each file that have observable initialization. Definitions with observable initialization in nested modules and types are included in the static initializer for the overall file.

All definitions have observable initialization except for the following definitions in modules:

- Function definitions
- Type function definitions
- Literal definitions
- Value definitions that are generalized to have one or more type variables
- Non-mutable values that are bound to an _initialization constant expression_, which is an expression whose elaborated form is one of the following:
  - A simple constant expression.
  - A use of the `sizeof<_>` operator or the `defaultof<_>` operator from `Unchecked`.
  - A let expression where the constituent expressions are initialization constant expressions.
  - A match expression where the input is an initialization constant expression, each case is a test against a constant, and each target is an initialization constant expression.
  - A use of one of the unary or binary operators `=`, `<>`, `<`, `>`, `<=`, `>=`, `+`, `-`, `*`, `<<<`, `>>>`, `|||`, `&&&`, `^^^`, `~~~`, `enum<_>`, `not`, `compare`, prefix `-`, and prefix `+` on one or two arguments, respectively. The arguments themselves must be initialization constant expressions, but cannot be operations on decimals or strings.
  - A use of a `[<Literal>]` value.
  - A use of a case from an enumeration type.
  - A use of a value that is defined in the same compilation unit and does not have observable initialization.

If the execution environment supports concurrent execution of multiple threads, each static initializer runs as a mutual exclusion region. A static initializer runs only once, on the first thread that acquires entry to the mutual exclusion region.

> **Clef Note (JSIR pathway)**: On the JSIR pathway the static initializer runs at module evaluation, once per host instantiation of the module. An isolate host MAY instantiate the module many times over a deployment's life and MAY restrict what module-evaluation code is permitted to do. "Once at program startup" therefore reads as "once per instantiation" on this pathway, in compilation order as on every other pathway, and a program SHALL NOT rely on the static initializer executing exactly once per logical deployment.

For example, if the program accesses `data` in this example, the static initializer runs and the program prints "hello":

```fsharp
module LibraryModule
printfn "hello"
let data = Map.empty<int, int>
```

All of the following represent definitions that do not have observable initialization because they are initialization constant expressions:

```fsharp
let x = DayOfWeek.Friday
let x = 1.0
let x = "two"
let x = enum<DayOfWeek>(0)
let x = 1 + 1
let x : int list = []
let x : int option = None
let x = compare 1 1
let x = match true with true -> 1 | false -> 2
let x = true && true
let x = 42 >>> 2
let x = Unchecked.defaultof<int>
let x = Unchecked.defaultof<string>
let x = sizeof<int>
```

### Explicit Entry Point

The last file that is specified in the compilation order for an executable file may contain an explicit entry point. The entry point is indicated by annotating a function in a module with `EntryPoint` attribute:

- The `EntryPoint` attribute applies only to a "let"-bound function in a module. The function cannot be a member.
- This attribute can apply to only one function, and the function must be the last declaration in the last file processed. The function may be in a nested module.
- The function is asserted to have type `array<string> -> int` before type checking. If the assertion fails, an error occurs.
- At startup, the entry point is passed one argument: an array that contains the command-line arguments passed to the program (excluding the program name).

The function becomes the entry point to the program. At startup, Clef executes all static initializers in compilation order, then evaluates the body of the entry point function.

> **Clef Note**: The entry point function's return value becomes the process exit code. A return value of 0 indicates success; non-zero values indicate errors. For freestanding (no-OS) targets, the return value may be ignored or handled by the runtime stub.

#### Entry Point Modes

Clef supports multiple entry point modes, specified via `output_kind` in the project file. The entry convention is target-specific: the symbol the platform hands control to, and the way the program terminates, are fixed by the target's ABI, not by a single universal freestanding form.

| Mode | Entry Symbol | libc Required | Description |
|------|-------------|---------------|-------------|
| **Console** | `main` | Yes | Standard mode; libc provides `_start` which calls user's `main` |
| **Freestanding (hosted ELF)** | `_start` | No | Hosted OS (Linux/ELF) without libc; compiler generates a `_start` wrapper that terminates via the exit syscall |
| **Freestanding (bare-metal)** | reset vector | No | Bare metal (for example a thumbv8m/M33 target); the platform enters at the reset-vector entry, there is no `_start` and no exit syscall, and termination is a halt |
| **Module export (JSIR pathway)** | exported bindings | No | The artifact is a JavaScript module; the host invokes exported handler functions and instantiates exported classes; there is no process entry and no exit code |
| **Library** | None | Optional | Shared library with exported symbols |

#### Freestanding Entry Point Generation

Freestanding builds (`output_kind = "freestanding"`) run without libc, so the compiler supplies the entry itself. What that entry looks like is fixed by the target: a hosted OS that loads an ELF image and services syscalls hands control to a `_start` symbol and expects an exit syscall to terminate; a bare-metal target has neither. The two cases below are the ends of that range.

**Hosted ELF (Linux/ELF freestanding).** On a hosted OS that loads an ELF image without libc, the compiler generates a `_start` wrapper function that:

1. Creates an empty `string array` (F# convention; full argc/argv conversion is future work)
2. Calls the user's `main` function with this argument
3. Calls `Sys.exit` with the return value

The generated `_start` has F# type `unit -> unit`:
- Takes no parameters (startup context comes from the platform)
- Returns unit (because `Sys.exit` never returns; the syscall terminates the process)

```
_start : unit -> unit
    argv ← Sys.emptyStringArray()     // Empty string array
    result ← main(argv)               // Call F# main
    Sys.exit(result)                  // Terminate with exit code (never returns)
 
```

> **Implementation Note**: `Sys.exit` has type `int -> unit`. Although the syscall never returns, the binding honors the type contract by emitting an unreachable unit return value. This maintains type consistency throughout the compilation pipeline.

**Bare-metal (reset-vector entry).** On a bare-metal target such as a thumbv8m/M33 there is no loader, no `_start`, and no exit syscall. The processor enters the image at the reset-vector entry recorded in the vector table, which the compiler emits together with the initial stack pointer. There is no argc/argv and no command line, so no empty `string array` is constructed. When the entry function returns, there is no process to terminate and no exit code to report to; termination is a halt (an idle loop, a `wfi`-class wait, or a target-defined reset), not a syscall.

```
reset : unit -> unit                  // reset-vector entry; address recorded in the vector table
    result ← main()                   // Call F# main (no argv on bare metal)
    halt(result)                      // No exit syscall: park the core (idle/wfi) or reset
 
```

> **Implementation Note**: `main`'s return value is the process exit code only where the platform has a process to exit. On a bare-metal target the return value has no OS recipient; a target descriptor MAY map it to a status register, an indicator, or nothing, and the entry does not return to any caller.

#### Module-Export Entry (JSIR Pathway)

> This subsection binds implementations claiming the **JavaScript Substrate** profile ([Conformance §7](conformance.md)).

The JSIR pathway's artifact is a module, and its entry contract is the export convention of the host environment named by the managed-substrate descriptor ([Platform Bindings](platform-bindings.md)). The pathway SHALL emit the export glue, the analog of `_start` on this pathway: exported handler functions and exported classes derived from the program's declared entry surface. Control arrives when the host invokes an export, and each invocation is an entry; there is no single entry function, no `argc`/`argv`, and no exit code, since termination belongs to the host.

The `EntryPoint` attribute's `array<string> -> int` shape does not apply on this pathway. A program compiled for this pathway declares its entry surface through the platform bindings of its host environment ([JavaScript Boundary Semantics §7](javascript-boundary.md)); values crossing inward at an entry invocation are foreign and are narrowed as at any boundary.

> **Not yet specified.** The declaration form by which a program names its exported entry surface.

#### Console Mode (libc)

For console builds (`output_kind = "console"`), the platform's libc provides `_start`, which:
1. Initializes the C runtime
2. Converts argc/argv from the stack
3. Calls the user's `main` symbol

The compiler generates `main` with a signature compatible with the platform's C ABI. The F# type `array<string> -> int` maps to the platform-appropriate C signature.

See [Platform Bindings](platform-bindings.md) for platform descriptor conventions.
