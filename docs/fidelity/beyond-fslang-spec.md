# Beyond the F# Language Specification: Why fsnative-spec Exists

## Introduction

The F# Language Specification, maintained by the community through the F# Software Foundation, defines the syntax, semantics, and behavior of the F# programming language. It is a thorough document that has guided implementations and ensured compatibility across compilers and tools. However, the F# Language Specification was written with an implicit assumption: that F# programs execute on the Common Language Runtime (CLR) or an equivalent managed environment.

This assumption permeates the specification. Types are defined in terms of their BCL equivalents. Memory management is delegated to the garbage collector. The physical layout of data structures is an implementation detail, not a language concern. These delegations made sense when every F# program targeted .NET, but they create a gap when the target is native machine code without a runtime.

This document explains why fsnative-spec exists as a separate specification, how it relates to and differs from fslang-spec, and what new concerns it must address that the original specification could safely ignore.

A historical note is warranted. fsnative-spec was not designed from some level of academic remove and then implemented. The specification emerged from implementation. The initial work on Alloy involved "hand-jamming" shadow types to mask BCL types during Baker type resolution in Firefly. These shadow types were engineering expedients, not theoretical constructs. Only after observing that these expedient types consistently resembled OCaml's native types did the connection to ML-family semantics become clear. The specification documents what was discovered through implementation, not what was decreed in advance. This engineering-first methodology means that every normative statement in fsnative-spec corresponds to a concrete implementation requirement that arose from making actual code compile and execute correctly.

The sympathies to principled design are certainly part of the full picture, and those elements serve to futher inform how the framework will develop and extend as the requirements grow and opportunities to target new platforms emerge.

## Part I: History - What fslang-spec Delegates

### 1.1 The .NET BCL Type System

The original F# Language Specification defines primitive types in terms of BCL types:

> The type `int` is an abbreviation for `System.Int32`.
> The type `string` is an abbreviation for `System.String`.
> The type `float` is an abbreviation for `System.Double`.

This mapping is not merely convenient; it is primary to that system. The F# specification does not define what an `int` is in abstract terms. It defines `int` as `System.Int32`, and the BCL specification defines what `System.Int32` means.

This delegation extends to:

1. **Primitive operations**: Addition, comparison, and conversion follow BCL semantics
2. **String behavior**: Encoding, comparison, and operations are as `System.String` defines
3. **Array semantics**: Bounds checking, indexing, and iteration follow `System.Array`
4. **Numeric limits**: `System.Int32.MaxValue` defines what F# `int` can hold

### 1.2 Memory Layout - or lack thereof

The F# Language Specification does not specify memory layout. Consider a record:

```fsharp
type Point = { X: float; Y: float }
```

The specification defines the syntax and the fact that `Point` has fields `X` and `Y` of type `float`. It does not specify:

1. Whether `Point` is a value type or reference type (without `[<Struct>]`)
2. The size of a `Point` value in bytes
3. The alignment requirements of `Point`
4. The offset of `X` and `Y` within the structure
5. Whether `Point` might be padded

These details are implementation-defined, meaning the CLR determines them. The specification assumes a runtime that handles such details.

### 1.3 Memory Management - another delegated primitive

The F# Language Specification contains no discussion of memory allocation or deallocation. This is *not* an oversight; it reflects the reality that garbage collection makes these concerns invisible to the language:

```fsharp
let names = [ "Alice"; "Bob"; "Carol" ]
```

The specification defines that `names` has type `string list`. It does not specify:

1. Where the list nodes are allocated
2. When they will be deallocated
3. Whether the strings share storage
4. What happens to the memory when `names` goes out of scope

The garbage collector handles all of this in the .NET 'world'. The specification can ignore it without consequence.

### 1.4 Platform Abstraction

The original F# Language Specification abstracts over its underlying platform entirely. It does not mention:

1. System calls or operating system interfaces
2. Hardware registers or memory-mapped I/O
3. Calling conventions or binary interfaces
4. Architecture-specific behavior

By design, these concerns are handled by the .NET BCL's platform abstraction layers. F# code running on Windows and F# code running on Linux or MacOS produce identical behavior (within the bounds of the BCL's abstraction).

## Part II: What Native Compilation Requires

### 2.1 The Absence of the Runtime

When compiling to native machine code without a runtime, the delegations described above become ***substantial*** gaps. There is no CLR to define what `System.Int32` means; there is no garbage collector to manage memory; there is no platform abstraction layer to hide OS differences. In our case, for the purposes of `fsnative` this is exactly our *opportunity*.

A specification for native F# must fill these gaps. It must define what the standard F# specification delegated:

1. **What are the primitive types?** Without the BCL, we need native definitions
2. **How is memory laid out?** Without the CLR, we must specify structure layouts
3. **How is memory managed?** Without a GC, we need explicit strategies
4. **How do we access the platform?** Without the BCL, we need binding mechanisms

This is no small task. There are many considerations to account for - not simply for the presumed OS-based world of WindowsOS, MacOS and Linux. But there are memory mapping concerns around GPU, NPU and other accelerators that are just as much a target for the Fidelity framework. Simply considering the constrained environment of micro-controllers, there are specific patterns that are allowed anothers that would not work at all. These are all concerns that have to be sorted out in the Fidelity framework to varying degrees, and `fsnative` has to account for an allow them in order for the full range of options to be available to realize the platform's vision.

### 2.2 Native Type Definitions

fsnative-spec defines a native type universe that replaces BCL types in full:

| F# Keyword | fslang-spec Definition | fsnative-spec Definition |
|------------|------------------------|--------------------------|
| `int` | `System.Int32` | 32-bit signed integer, two's complement |
| `string` | `System.String` | UTF-8 fat pointer `{ptr, len}` |
| `float` | `System.Double` | 64-bit IEEE 754 binary floating-point |
| `bool` | `System.Boolean` | 8-bit value, 0 or 1 |
| `unit` | `Microsoft.FSharp.Core.Unit` | Zero-sized type |

fsnative-spec cannot reference BCL types because they do not exist in the target environment. Instead, it provides direct definitions of type behavior and representation.

### 2.3 Memory Layout Specification

fsnative-spec specifies what fslang-spec leaves implementation-defined:

```fsharp
[<Struct>]
type NativeStr = {
    Ptr: nativeptr<byte>
    Length: int
}
```

fsnative-spec defines:

1. **Size**: 16 bytes on 64-bit platforms (8-byte pointer + 8-byte length with padding)
2. **Alignment**: 8-byte alignment
3. **Field order**: `Ptr` at offset 0, `Length` at offset 8
4. **Encoding**: The bytes pointed to are valid UTF-8

This level of detail is absent from fslang-spec because the CLR handles it. fsnative-spec must be explicit. Arriving at a generalized pattern for memory layout that adheres to the goals of Fidelity framework while providing maximum degrees of freedom to target different processors is going to be a non-trivial challenge. We expect to start with some relatively straight-foward hard-coded patterns and develop a proper abstraction pattern later. Our sense is that a plug-in system will need to be developed that will have some coupling to project-level declaration of the targeted hardware, but that "story" has yet to develop at this early stage. We're willing to live with some brittle implementations to help us target early wins and avoid over-engineering in the abstract.

But we have enough of clarity to start that process in some initial formative stages, expecting that the story will develop over time. Those initial ideas are outlined below.

### 2.4 Memory Region Model

fsnative-spec introduces an entirely new concept that has no analog in fslang-spec: memory regions.

```fsharp
[<Measure>] type peripheral
[<Measure>] type sram
[<Measure>] type flash
[<Measure>] type arena
[<Measure>] type stack
```

These regions classify memory by its physical characteristics:

| Region | Volatility | Cacheability | Typical Use |
|--------|------------|--------------|-------------|
| `peripheral` | Volatile | No caching | Hardware registers |
| `sram` | Non-volatile | Normal caching | General RAM |
| `flash` | Non-volatile | Aggressive caching | Program storage |
| `arena` | Non-volatile | Scope-bounded | Temporary allocation |
| `stack` | Non-volatile | Core-local | Local variables |

fslang-spec has no need for these concepts. The BCL abstracts memory into a uniform addressable space. fsnative-spec must distinguish regions because:

1. Peripheral access requires volatile reads; optimizing them away is incorrect
2. Flash is read-only at runtime; writes are errors
3. Stack memory has automatic lifetime; arenas have scoped lifetime
4. Different regions may require different access patterns

### 2.5 Access Kind Enforcement

Closely related to memory regions is access kind enforcement:

```fsharp
[<Measure>] type readOnly
[<Measure>] type writeOnly
[<Measure>] type readWrite
```

fslang-spec does not distinguish read and write access at the type level. Any reference to a mutable value can be read or written. fsnative-spec must distinguish these because:

1. Some hardware registers are write-only; reading them is undefined behavior
2. Some memory regions are read-only; writing them is a fault
3. Distinguishing access enables static verification of hardware access patterns

### 2.6 Platform Binding Specification

fslang-spec relies on `DllImportAttribute` for platform interop:

```fsharp
[<DllImport("kernel32.dll")>]
extern bool WriteFile(...)
```

This mechanism depends on the BCL marshaling layer, which does not exist in native compilation. fsnative-spec defines an alternative: the `Platform.Bindings` module convention.

```fsharp
module Platform.Bindings =
    let writeBytes (fd: int) (buffer: nativeptr<byte>) (count: int) : int =
        Unchecked.defaultof<int>
```

This convention allows the source code to define binding signatures while the compiler backend provides platform-specific implementations. fsnative-spec defines:

1. The module naming convention
2. The signature format
3. The marker syntax (`Unchecked.defaultof<T>` and `()`)
4. How the compiler recognizes and replaces these markers

## Part III: New Concerns for Native Compilation

### 3.1 Lifetime and Ownership

fslang-spec does not discuss object lifetimes beyond the basic scoping rules for `use` bindings. The garbage collector ensures objects live as long as they are reachable; programmers need not think about it.

fsnative-spec must address lifetime explicitly:

1. **Stack lifetime**: Values on the stack live until the function returns
2. **Arena lifetime**: Values in an arena live until the arena is deallocated
3. **Static lifetime**: Values in static memory live for the program's duration
4. **Ownership transfer**: When is a value moved versus copied?

These concepts are reserved in the current specification but will require detailed treatment as the implementation matures.

### 3.2 Coeffect Tracking

fslang-spec does not track function effects. A function returning `int` might be pure, might perform I/O, or might allocate memory; the type does not reveal this.

fsnative-spec introduces coeffects:

```fsharp
// Reserved syntax
let readFile (path: NativeStr) : NativeArray<byte> -[IO.File]-> NativeArray<byte>
```

Coeffects make effect tracking explicit:

1. **Pure**: No observable side effects
2. **IO**: Performs input/output operations
3. **Alloc**: Allocates memory
4. **Unsafe**: Performs operations the type system cannot verify

This tracking enables the compiler to verify that pure functions do not call effectful ones, that unsafe operations are explicitly marked, and that resources are released appropriately.

### 3.3 Hardware Access Patterns

fslang-spec has no concept of hardware access. fsnative-spec must specify:

1. **Volatile semantics**: Reads and writes that cannot be reordered or eliminated
2. **Memory barriers**: Explicit ordering constraints
3. **Register access**: Type-safe peripheral descriptor patterns
4. **Interrupt handling**: How control flow interacts with hardware events

These concerns arise from fsnative's ability to target bare-metal embedded systems, a use case entirely outside fslang-spec's scope.

### 3.4 Verification Integration

fslang-spec does not address formal verification. fsnative-spec integrates with F* for design-time verification:

1. **Layout verification**: Proving size and alignment properties
2. **Region verification**: Proving pointer validity within regions
3. **Lifetime verification**: Proving that pointers do not outlive their referents
4. **Correctness verification**: Proving functional properties of algorithms

This integration allows proofs to guide compilation, enabling optimizations that would be unsafe without the proof.

## Part IV: Relationship to Influences

### 4.1 OCaml: Accidental Sympathy

The relationship between F# Native and OCaml was discovered, not designed. The initial work on the Alloy library involved creating shadow types to intercept BCL type resolution. A `NativeStr` type was needed because `System.String` could not exist in a runtime-free environment. And to give credit where it is due, FSharp.Core still contained some primitive types of its own. The `voption` type existed which is beyonds the standard `option` type's null representation ands assumed garbage collection. And similarly the NativeInterop nativeptr still is central to much of what we were able to accomplish before making the "full break" to create our own fsnative implementation.

After implementing several of these types, a pattern emerged: the shadow types bore a striking resemblance to OCaml's native types. UTF-8 strings, value-typed options, struct-based records; these were not copied from OCaml but converged toward the same design independently. This accidental sympathy revealed something fundamental about ML-family languages: when you remove the managed runtime, the natural type representations that emerge are value-oriented and memory-explicit.

| Aspect | BCL F# | OCaml | F# Native |
|--------|--------|-------|-----------|
| String encoding | UTF-16 | UTF-8 | UTF-8 |
| Option type | Reference, nullable | Value | Value |
| Record default | Reference | Value possible | Struct |
| Integer type | Fixed 32-bit | Native width | Fixed or native |

fsnative-spec documents these alignments with OCaml not as intentional design but as evidence that both languages, targeting native execution, arrive at similar solutions to the same underlying problems.

### 4.2 F*: Verification and Memory Models

F* provides the HyperStack memory model that again corresponds to what we arrived at for fsnative's region system. Key F* concepts that fsnative-spec respects and in certain cases adopts:

1. **Region identifiers**: Phantom type parameters encoding memory region membership
2. **Containment hierarchy**: Tree structure with stack frames and heap regions
3. **Preorders**: Constraints on how values in regions may evolve
4. **Witnessed predicates**: Tracking resource availability across code boundaries

fsnative-spec documents how these concepts map to F# Native's type system, particularly through units of measure as phantom type parameters. This will continue to develop as Fidelity's continuation patterns with actors and arenas/sentinels starts to become a more coherent part of the framework.

### 4.3 Rust influences: Considering Ownership and Borrowing

Rust pioneered compile-time ownership tracking for memory safety. fsnative-spec reserves syntax for similar concepts:

```fsharp
Owned<'T>      // Exclusive ownership
Borrowed<'T>   // Temporary borrow
move expr      // Ownership transfer
&expr          // Immutable borrow
```

The relationship to Rust is one of inspiration, not imitation. F# Native will adapt ownership concepts to F#'s idioms rather than adopting Rust's syntax directly. The point is to have the compiler deal with these concerns without design-time "interference" that Rust developers experience with having to deal with the borrow checker at every turn. We plan to provide options for managing this directly in design-time where it's performance-critical, but for now our emphasis is on keeping design-time concerns relatively consistent with an F# "idiomatic" experience.

### 4.4 C and C++: Hardware Access

One of the major areas of interest is how to expand a "native library system" for the Fidelity framework that can preserve all of the advantages of its operating mechanics. We found that many .NET libraries that "wrap" low-level C and C++ libraries offered some insight, so we're starting with a "clean" approach to that with our "Farscape" binding generator. With that, we need to have some "hooks" to integrate those F# wrappers into the library system of the Fidelty framework, and that means integrating those primitives into the pipeline in a way that native F# function wrappers can provide safe harbor for their integration - either as dynamic 'syscall' external references or as pipelined targets for static binding in the LLVM LTO layer of compilation. 

With that, we saw C and C++ as a means to provide the low-level hardware access patterns that systems programming requires. fsnative-spec incorporates:

1. **CMSIS conventions**: `__I`, `__O`, `__IO` volatile qualifiers
2. **Structure layout control**: `[<Struct>]`, packing, alignment
3. **Pointer arithmetic**: Explicit, type-safe pointer operations
4. **Inline assembly**: Reserved for platform-specific optimization

fsnative-spec documents these capabilities while maintaining F#'s type safety guarantees.

## Part V: Specification Structure

### 5.1 Normative versus Informative

fsnative-spec uses RFC 2119 keywords to distinguish requirements from recommendations:

- **SHALL/MUST**: Absolute requirements that implementations must satisfy
- **SHALL NOT/MUST NOT**: Absolute prohibitions
- **SHOULD**: Recommendations that may be deviated from with good reason
- **MAY**: Optional features

Example:

> NORMATIVE: String literals SHALL have type `string` with UTF-8 fat pointer semantics, not `System.String`.

This precision ensures that implementations can verify conformance.

### 5.2 Specification Parts

fsnative-spec is organized into parts that address distinct concerns:

| Part | Title | Scope |
|------|-------|-------|
| 1 | Native Type Universe | Primitive type definitions |
| 2 | Null-Free Semantics | Non-nullable type behavior |
| 3 | SRTP Resolution | Witness hierarchy for type constraints |
| 4 | Memory Semantics | Allocation strategies |
| 5 | Coeffects | Effect tracking |
| 6 | Platform Bindings | OS interface conventions |
| 7 | Compatibility | Syntax compatibility with F# |
| 8 | Diagnostics | Error messages |
| 9 | Memory Region Types | Region definitions |
| 10 | Access Kind Enforcement | Read/write permissions |
| 11 | Peripheral Descriptors | Hardware binding |
| 12 | Ownership and Coeffects | Future ownership model |

### 5.3 Appendices

Appendices provide reference material:

- **Appendix A**: Type mapping tables
- **Appendix B**: Grammar extensions (reserved syntax)
- **Appendix C**: Specification status
- **Appendix D**: Diagnostic message catalog

## Part VI: What fsnative-spec Is Not

### 6.1 Not a Subset of fslang-spec

fsnative-spec is sometimes described as defining a "subset" of F#. This characterization is misleading. fsnative-spec defines:

1. **Different semantics** for existing syntax (string literals become `NativeStr`)
2. **New concepts** with no fslang-spec analog (memory regions)
3. **New constraints** that fslang-spec does not impose (null prohibition)

A subset would accept less syntax. fsnative-spec accepts the same syntax with different meaning and adds new capabilities beyond what fslang-spec defines.

### 6.2 Not a Replacement for fslang-spec

fsnative-spec does not replace fslang-spec. They address different compilation targets:

- **fslang-spec**: F# targeting the Common Language Runtime
- **fsnative-spec**: F# targeting native machine code

Both specifications are valid. Code written for one may or may not work with the other, depending on which features it uses.

### 6.3 Not a Formal Semantics

fsnative-spec is a language specification, not a formal semantics in the mathematical sense. It defines behavior through prose descriptions and examples rather than through operational or denotational semantics.

Future work may develop a formal semantics for F# Native, particularly in conjunction with F* verification. Such a semantics would complement rather than replace the prose specification.

## Part VII: The Path Forward

### 7.1 Implementation Alignment

fsnative-spec and FNCS (F# Native Compiler Services) evolve together:

1. The specification defines intended behavior
2. FNCS implements that behavior
3. Firefly consumes FNCS output to produce binaries
4. Discrepancies between specification and implementation are bugs

This alignment requires ongoing coordination between specification authors and implementers.

### 7.2 Community Engagement

Unlike fslang-spec, which is maintained by the F# Software Foundation, fsnative-spec is maintained as part of the Fidelity framework. However, community input is welcome:

1. **Specification issues**: Report ambiguities or errors
2. **Implementation gaps**: Identify unspecified behavior
3. **Use case requirements**: Propose new capabilities

### 7.3 Future Parts

fsnative-spec reserves space for future capabilities:

1. **Ownership and borrowing**: Compile-time memory safety
2. **Lifetime parameters**: Explicit lifetime annotations
3. **Advanced coeffects**: Fine-grained effect tracking
4. **Distributed computation**: Actor and process specifications

These parts will be developed as the implementation matures and use cases emerge.

## Conclusion

fsnative-spec exists because native compilation requires decisions that fslang-spec delegated to the runtime. The Common Language Runtime defined type representations, managed memory, and abstracted platforms. Without that runtime, these concerns become language-level concerns that the specification must address.

This is not a limitation of fslang-spec. The original specification was appropriately scoped for its target environment. fsnative-spec extends that scope to environments where the runtime is absent, where memory layout matters, and where hardware access is a language feature.

For developers transitioning from F# on .NET to F# Native, fsnative-spec documents the semantic differences they will encounter. For implementers, it provides the normative requirements that FNCS and Firefly must satisfy. For language designers, it demonstrates how a specification can extend beyond a managed environment while preserving the essential character of the language it specifies.

The result is a specification that enables F# to target hardware directly, without a runtime intermediary, while retaining the type safety, expressiveness, and elegance that define the language.

A final reflection on methodology: specifications are often presented as if they preceded implementation, as if the normative requirements were derived from first principles and then realized in code. fsnative-spec does not pretend to this origin. The specification emerged from the practical work of making F# compile to native binaries. The native type universe was discovered by trying alternatives until something worked. The memory region model arose from the concrete need to distinguish peripheral registers from RAM. The OCaml correspondence was noticed after the fact, not planned in advance.

This engineering-driven approach has an important virtue: every requirement in this specification exists because implementation demanded it. There are no theoretical constructs that seemed elegant but proved impractical. There are no features specified for completeness that no one will use. What is written here is what was needed to compile real programs to real machine code. The universal nature of these findings, their convergence with OCaml, their alignment with F*'s memory models, suggests that they reflect something true about native compilation of ML-family languages, not merely the preferences of the implementers.
