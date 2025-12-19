# F# Native Language Specification

**Toward a normative specification for native F# type semantics and memory management.**

---

## What is fsnative-spec?

fsnative-spec aims to define the complete language semantics for [fsnative](https://github.com/speakeztech/fsnative) (F# Native Compiler Services). Where the [standard F# specification](https://fsharp.org/specs/language-spec/) describes behavior in terms of the .NET runtime and BCL types, we plan for fsnative-spec to provide explicit definitions for everything the CLR normally handles implicitly: type layouts, memory ownership, lifetime verification, and deterministic resource management.

**The F# you write stays the same.** You write `string`, `option`, `int`, `array` - the familiar F# types. fsnative-spec defines what those types *mean* when targeting native compilation. The specification is about semantics, not new syntax.

When complete, this document will serve as the authority for fsnative's behavior.

## Why fsnative-spec Will Contain More Than the F# Specification

The standard F# specification makes extensive use of the .NET runtime as an implicit substrate. Consider what the F# spec does *not* need to define:

**Memory Allocation**: The F# spec never explains where objects live in memory, how allocation works, or when memory is reclaimed. It simply notes that values are created and trusts the CLR's garbage collector.

**Object Layout**: The F# spec doesn't define how a record's fields are arranged in memory, what padding exists between fields, or how discriminated union tags are represented. These are "implementation details" handled by the runtime.

**Reference Semantics**: The F# spec doesn't distinguish between a value that owns its memory and a value that borrows someone else's memory. In managed F#, all references are equivalent because the GC ensures memory remains valid.

**Resource Cleanup**: The F# spec has no drop semantics. Objects are allocated, used, and eventually collected. The programmer need not think about when cleanup occurs.

**Type Identity**: In managed F#, types are identified by their assembly metadata. `System.String` from one assembly is the same type as `System.String` from another because the runtime resolves these identities.

We plan for fsnative-spec to explicitly define all of these. Native compilation has no runtime to defer to. The specification will need to provide complete, unambiguous definitions for:

- Exact memory layout of every type
- Ownership semantics for every value
- Lifetime constraints for every reference
- Deterministic cleanup timing
- Type identity independent of assembly metadata

## The Core Principle: Same Types, Native Semantics

When you write this F# code:

```fsharp
let greeting = "Hello, World!"
let maybeValue = Some 42
let numbers = [| 1; 2; 3 |]
```

You're using `string`, `option`, and `array` - exactly as you would in any F# program. The specification defines what these types mean for native compilation:

- **`string`**: UTF-8 encoded, null-terminated, deterministic lifetime
- **`option`**: Value type, zero-cost `None`, no heap allocation
- **`array`**: Contiguous memory, compile-time or runtime size tracking
- **`int`**, **`float`**, **`bool`**: Native machine representations

The specification describes the semantics behind these familiar types. It does not introduce new type names that users need to learn.

## Planned Semantic Additions

Beyond redefining what existing types mean, we're considering semantic concepts that have no equivalent in the F# specification:

### Ownership and Borrowing

We're designing rules for tracking who owns memory and who borrows it:

- Every value has exactly one owner
- References can borrow values without taking ownership
- The compiler would verify borrows don't outlive their owners
- Ownership can transfer (move semantics) or values can be copied

### Memory Regions

We plan to define where values can live:

- **Stack**: Automatic lifetime, fastest access, limited size
- **Heap**: Manual or RAII lifetime, can outlive creating scope
- **Arena**: Bulk allocation with single deallocation point
- **Peripheral**: Memory-mapped I/O with volatile semantics
- **Flash**: Read-only program memory

### Access Kinds

Pointers would carry access restrictions:

- **Read-only**: Can read but not write (string literals, flash memory)
- **Write-only**: Can write but not read (certain DMA buffers)
- **Read-write**: Full access

### Deterministic Cleanup

When values go out of scope, resources are freed immediately:

- Drop order is reverse declaration order
- No garbage collector decides when cleanup happens
- The programmer can reason about exactly when resources are released

## The Absorption Model

In standard F#, types like `int` and `string` are defined in external assemblies. The compiler discovers their operations by reading assembly metadata.

We plan for fsnative to define these types **intrinsically** - built into the compiler itself. When you write `int`, the compiler knows its representation, operations, and semantics because that knowledge is part of fsnative, not discovered from external sources.

This means:
- Type resolution requires no external assemblies
- SRTP constraints resolve against built-in definitions
- The compiler contains the complete type system

Alloy provides library *functions* that operate on these intrinsic types. The types themselves are defined by fsnative per this specification.

## The Fidelity Ecosystem

fsnative-spec is intended to be the normative foundation for the Fidelity native compilation framework:

```
fsnative-spec (this repo)     <- Will define the semantic rules
        |
fsnative (FNCS)               <- Will implement type resolution per spec
        |
Firefly                       <- Will generate code per spec semantics
        |
Native Binary                 <- Will run with spec-defined behavior
```

## Planned Specification Structure

We're planning to organize the specification as markdown files in the `spec/` folder:

| Chapter | Description |
|---------|-------------|
| `spec/01-introduction.md` | Scope, conformance, notation |
| `spec/02-types.md` | Native semantics for F# types |
| `spec/03-ownership.md` | Ownership rules, move semantics |
| `spec/04-borrowing.md` | Borrow rules, lifetime constraints |
| `spec/05-memory-regions.md` | Stack, heap, arena, peripheral, flash |
| `spec/06-drop.md` | Deterministic cleanup semantics |
| `spec/07-srtp.md` | SRTP resolution against intrinsic types |

*Note: These chapters do not yet exist.*

## Status

The specification is not yet started. This README describes our intent and design direction.

## Contributing

Contributions will be welcome once initial drafts are available. Please open an issue to discuss ideas.

## License

MIT License. See [LICENSE.txt](LICENSE.txt) for details.

## Contact

fsnative-spec is being developed by [SpeakEZ Technologies](https://speakez.tech) as part of the Fidelity native compilation framework.

---

*Toward the rules that will make native F# safe and deterministic.*
