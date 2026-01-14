# Program Structure and Execution

> **F# Native Note**: F# Native programs do not use CLI assemblies. Instead, programs are compiled directly to native binaries from source files, with dependencies resolved at compile time from source packages or pre-compiled native libraries.

F# Native programs are composed of an ordered sequence of signature (`.fsi`) files and implementation (`.fs`) files, plus any library dependencies specified in the project file (`.fidproj`). Script files (`.fsx`) are supported for development and tooling but are not part of native compilation.

```fsgrammar
implementation-file :=
    namespace-decl-group ... namespace-decl-group
    named-module
    anonynmous-module

script-file := implementation-file  -- script file, additional directives allowed

signature-file :=
    namespace-decl-group-signature ... namespace-decl-group-signature
    anonynmous-module-signature
    named-module-signature

named-module :=
    module long-ident module-elems

anonymous-module :=
    module-elems

named-module-signature :=
    module long-ident module-signature-elements

anonymous-module-signature :=
    module-signature-elements

script-fragment :=
    module-elems                    -- interactively entered code fragment
```

A sequence of implementation and signature files is checked as follows.

1. Form an initial environment `sig-env0` and `impl-env0` by adding all library dependencies to the environment in the order specified in the project file. This means the following procedure is applied for each dependency:
    - Add the top-level types, modules, and namespaces to the environment.
    - For each `AutoOpen` attribute in the library, find the types, modules, and namespaces that the attribute references and add these to the environment.
    
    > **F# Native Note**: The native standard library is automatically included and provides the core types (`string`, `option`, `int`, etc.) with native semantics. See [Native Type Mappings](native-type-mappings.md).

    The resulting environment becomes the active environment for the first file to be processed.
2. For each file:
    - If the `i`th file is a signature file `file.fsi`:

        a. Check it against the current signature environment `sig-envi1`, which generates the
        signature `Sigfile` for the current file.

        b. Add `Sigfile` to `sig-envi-1` to produce `sig-envi` to make it available for use in later
        signature files.

        The processing of the signature file has no effect on the implementation environment, so
`impl-envi` is identical to `impl-envi-1`.

    - If the file is an implementation file `file.fs`, check it against the environment `impl-envi-1`,
    which gives elaborated namespace declaration groups `Implfile`.

       a. If a corresponding signature `Sigfile` exists, check `Implfile` against `Sigfile` during this
          process ([§](namespace-and-module-signatures.md#signature-conformance)). Then add `Sigfile` to `impl-envi-1` to produce `impl-envi`. This step makes
          the signature-constrained view of the implementation file available for use in later
          implementation files. The processing of the implementation file has no effect on the
          signature environment, so `sig-envi` is identical to `sig-envi-1`.

       b. If the implementation file has no signature file, add `Implfile` to both `sig-envi-1` and `impl-envi-1`,
          to produce `sig-envi` and `impl-envi`. This makes the contents of the
          implementation available for use in both later signature and implementation files.

The signature file for a particular implementation must occur before the implementation file in the
compilation order. For every signature file, a corresponding implementation file must occur after the
file in the compilation order. Script files may not have signatures.

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

## Signature Files

Signature files specify the functionality that is implemented by a corresponding implementation file.
Each signature file contains a sequence of `namespace-decl-group-signature` elements. The inclusion
of a signature file in compilation implicitly applies that signature type to the contents of a
corresponding implementation file.

_Anonymous signature files_ do not have either a leading `module` or `namespace` declaration. Anonymous
signature files contain `module-elems` that are implicitly placed in a module. The name of the module
is generated from the name of the source file by capitalizing the first letter and removing the
filename extension. If the filename contains characters that are not valid in an F# identifier, the
resulting module name is unusable and a warning occurs.

Given an initial environment `env` , a signature file is checked as follows:

- Create a new constraint solving context.
- Check each `namespace-decl-group-signaturei` in `envi-1` and add the result to that environment to
    create a new environment `envi`.

The result of checking a signature file is a set of elaborated namespace declaration group types.

## Script Files

> **F# Native Note**: Script files (`.fsx`, `.fsscript`) are primarily used for development tooling and F# Interactive. They are not directly compiled to native binaries by the Firefly compiler. For native compilation, use implementation files (`.fs`) organized via a `.fidproj` project file.

Script files have the `.fsx` or `.fsscript` filename extension. They are processed for development scenarios with the following characteristics:

- Side effects from scripts are executed immediately in the interactive environment.
- Script files may add other signature, implementation, and script files to the list of sources by using the `#load` directive. Files are compiled in the same order that was passed to the compiler, except that each script is searched for `#load` directives and the loaded files are placed before the script, in the order they appear in the script. If a filename appears in more than one `#load` directive, the file is placed in the list only once, at the position it first appeared.
- Script files may have `#nowarn` directives, which disable a warning for the entire compilation.

The Firefly compiler defines the `FIDELITY` compilation symbol for native compilation. The `COMPILED` symbol is also defined for compatibility.

Script files may not have corresponding signature files.

## Compiler Directives

_Compiler directives_ are declarations in non-nested modules or namespace declaration groups in the following form:

```fsgrammar
# id string ... string
```

The lexical preprocessor directives `#if`, `#else`, `#endif` and `#indent "off"` are similar to compiler directives. For details on `#if`, `#else`, `#endif`, see [§](lexical-analysis.md#conditional-compilation). The `#indent "off"` directive is described in [§](features-for-ml-compatibility.md#file-extensions-and-lexical-matters).

The following directives are valid in all files:

| Directive | Example | Short Description |
| --- | --- | --- |
| `#nowarn` | `#nowarn "54"` | For signature (`.fsi`) files and implementation (`.fs`) files, turns off warnings within this lexical scope. For script (`.fsx` or `.fsscript`) files, turns off warnings globally. |

> **F# Native Note**: The `#r` directive for referencing assemblies is not applicable to native compilation. Dependencies are specified in the `.fidproj` project file. The following script directives are supported only in F# Interactive tooling, not in native compilation:

| Directive | Example | Short Description |
| --- | --- | --- |
| `#load` | `#load "core.fsi" "core.fs"` | Loads a set of signature and implementation files into the script execution engine. |
| `#time` | `#time "on"` | Enables or disables the display of performance information. |
| `#help` | `#help` | Asks the script execution environment for help. |
| `#quit` | `#quit` | Requests the script execution environment to halt execution and exit. |

## Program Execution

> **F# Native Note**: Execution of F# Native code occurs as a standalone native binary, not within a CLI runtime. There is no assembly loading, no JIT compilation, and no garbage collector. Memory management is deterministic and controlled by the compiler.

Execution of F# Native code begins when the native binary is loaded by the operating system. During execution, the program can use the functions, values, static members, and object constructors that the compiled modules define.

### Execution of Static Initializers

Each implementation file involves a _static initializer_. In F# Native, static initialization is deterministic and occurs at program startup:

- For executables with an explicit entry point function, the static initializers for all files are executed in compilation order before the entry point function is called.
- For executables with an implicit entry point, the static initializer for the last file is the body of the implicit entry point function.

> **F# Native Note**: Static initialization is eager and deterministic. All module-level bindings with observable initialization are evaluated at program startup, in compilation order. This provides predictable behavior essential for embedded and real-time systems.

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
  - A use of one of the unary or binary operators `=`, `<>`, `<`, `>`, `<=`, `>=`, `+`, `-`, `*`, `<<<`, `>>>`, `|||`, `&&&`, `^^^`, `~~~`, `enum<_>`, `not`, `compare`, prefix `–`, and prefix `+` on one or two arguments, respectively. The arguments themselves must be initialization constant expressions, but cannot be operations on decimals or strings.
  - A use of a `[<Literal>]` value.
  - A use of a case from an enumeration type.
  - A use of a value that is defined in the same compilation unit and does not have observable initialization.

If the execution environment supports concurrent execution of multiple threads, each static initializer runs as a mutual exclusion region. A static initializer runs only once, on the first thread that acquires entry to the mutual exclusion region.

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

The function becomes the entry point to the program. At startup, F# Native executes all static initializers in compilation order, then evaluates the body of the entry point function.

> **F# Native Note**: The entry point function's return value becomes the process exit code. A return value of 0 indicates success; non-zero values indicate errors. For freestanding (no-OS) targets, the return value may be ignored or handled by the runtime stub.

#### Entry Point ABI Boundary

Unlike .NET F#, where the runtime provides an already-marshalled `string[]` to the entry point, F# Native must handle the C ABI boundary explicitly. The entry point is a platform binding, just like syscalls or memory operations.

The F# semantic signature `array<string> -> int` maps to a platform-specific C ABI:

| Platform | C Signature | F# Semantic Type |
|----------|-------------|------------------|
| Linux/POSIX | `int main(int argc, char** argv)` | `array<string> -> int` |
| Windows (console) | `int main(int argc, char** argv)` | `array<string> -> int` |
| Windows (GUI) | `int WinMain(HINSTANCE, HINSTANCE, LPSTR, int)` | *platform-specific* |
| Freestanding | `void _start(void)` | `unit -> int` |

The platform descriptor (from `Fidelity.Platform`) defines the `EntryPointABI` via quotations, which specifies:
- The C function name and signature
- The parameter conversion strategy (e.g., `char**` to `array<string>`)
- The return type mapping

This binding is resolved at compile time through the platform binding resolution pass, ensuring the C boundary translation is type-safe and platform-appropriate. The F# code author writes the idiomatic F# signature; the compiler generates the ABI translation based on the platform quotation.

See [Platform Bindings](platform-bindings.md) for the quotation-based platform descriptor model.
