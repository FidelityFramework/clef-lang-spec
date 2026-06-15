---
title: "Introduction"
weight: 10
---

Clef is a concurrent, natively compiled language in the ML family. It produces standalone executables for CPUs, GPUs, NPUs, FPGAs, and other accelerators without runtime dependencies. Clef uses ML-family syntax rooted in F# and shares common constructs with [OCaml](https://ocaml.org/), while incorporating influences from F\* (proof-carrying compilation), Scheme (nanopass compilation architecture), and hardware-oriented concurrency models.

Clef's type system resolves types to native representations at compile time rather than to .NET Base Class Library (BCL) types. Concurrency is a first-class language concern: the actor model, `Incremental<'T>`, delimited continuations, and interaction nets are foundational primitives. This specification defines those native and concurrent semantics.

## Clef Compiler Service (CCS)

The Clef Compiler Service (CCS) is the compiler frontend that implements this specification. CCS originated from the F# Compiler Services (FCS) codebase but has diverged substantially, with native type semantics, a nanopass compilation architecture, and Program Semantic Graph (PSG) construction that has no FCS counterpart.

### What CCS Provides

CCS constitutes the front end of the Composer compiler pipeline. It performs parsing, type inference, constraint resolution, and Program Semantic Graph (PSG) construction for Clef programs:

| Capability | Description |
|------------|-------------|
| **Parsing** | Clef syntax analysis producing syntax trees |
| **Type Checking** | Type inference with native type resolution |
| **SRTP Resolution** | Statically resolved type parameters against native witnesses |
| **PSG Construction** | Program Semantic Graph with fully resolved types and concurrency annotations |

CCS outputs a PSG with native types attached. This graph flows to downstream stages of the Composer pipeline for code generation targeting CPUs, GPUs, and other accelerators.

### Distinction from FCS

CCS is not an extension or plugin to FCS. It is a separate compiler frontend with fundamentally different type semantics:

| Aspect | FCS (Standard F#) | CCS (Clef) |
|--------|-------------------|------------------|
| **Type Universe** | BCL types (`System.String`, `System.Int32`) | Native representations (same syntax, native semantics) |
| **String Literals** | `System.String` (UTF-16, GC-managed) | `string` with native semantics (UTF-8, fat pointer) |
| **Option Types** | Reference type, nullable | `option<'T>` with value semantics, stack-allocated, non-nullable |
| **SRTP Resolution** | .NET method tables | Native witness hierarchy |
| **Base Type** | `System.Object` (`obj`) | None - no universal base type |
| **Output** | IL generation | Program Semantic Graph (PSG) for Composer pipeline |

### Architectural Principles

CCS adheres to these principles:

**Native Types Are Intrinsic**: Primitive types (`int`, `string`, `bool`, etc.) are defined within CCS itself, not discovered from external assemblies. When a program uses `string`, CCS knows its representation, operations, and memory semantics because that knowledge is built into the compiler.

**No BCL Dependencies**: The type checking path SHALL NOT reference BCL types. Types resolve to native representations as defined in [Native Type Mappings](native-type-mappings.md).

**No Universal Base Type**: There is no `obj` type. All types are concrete. Polymorphism is achieved through generics and SRTP, not runtime type erasure. See [Native Type Mappings § The Universal Base Type `obj` Is Not Available](native-type-mappings.md#the-universal-base-type-obj-is-not-available).

**SRTP Against Native Witnesses**: Statically resolved type parameters resolve against the native witness hierarchy. This enables compile-time polymorphism without runtime overhead.

**PSG Fidelity**: CCS produces a Program Semantic Graph that preserves full type information, constraint resolutions, SRTP witness selections, and concurrency annotations. Downstream stages of the Composer pipeline consume this graph directly. The typed representation is defined by [`ClefExpr`](clef-expr.md), CCS's native expression type that replaces FCS's `FSharpExpr`.

### Layer Separation

CCS has a focused responsibility within the Composer pipeline:

| Component | Responsibility |
|-----------|---------------|
| **CCS** | Type universe, literal typing, type inference, SRTP resolution, PSG construction, editor services |
| **Composer** | PSG consumption, nanopass transformations, platform-aware MLIR generation, native code output |

CCS produces a PSG with native types attached and full symbol information preserved for design-time tooling. Composer consumes the PSG as "correct by construction" and applies a series of nanopass transformations before generating platform-specific code.

### Normative Requirements

NORMATIVE: CCS SHALL resolve `string` to native semantics (UTF-8 fat pointer), not `System.String`.

NORMATIVE: CCS SHALL resolve `option<'T>` to value semantics (stack-allocated, non-nullable), not reference semantics.

NORMATIVE: CCS SHALL reject any code that references `obj`, `System.Object`, or performs boxing/unboxing operations.

NORMATIVE: CCS SHALL resolve SRTP constraints against the native witness hierarchy.

NORMATIVE: The typed tree output by CCS SHALL include resolved SRTP witnesses, enabling downstream stages to generate direct calls without runtime dispatch.

## A First Program

Over the next few sections, we will look at some small F# programs, describing some important aspects of F# along the way. As an introduction to F#, consider the following program:

```fsharp
let numbers = [ 1 .. 10 ]
let square x = x * x
let squares = List.map square numbers
Console.WriteLine $"N^2 = {squares}"
```

To compile this program with Clef:

- Use the Firefly compiler to produce a native executable.
- The resulting binary runs without any runtime dependencies.

### Lightweight Syntax

The F# language uses simplified, indentation-aware syntactic constructs known as lightweight syntax. The lines of the sample program in the previous section form a sequence of declarations and are aligned on the same column. For example, the two lines in the following code are two separate declarations:

```fsharp
let squares = List.map square numbers
Console.WriteLine $"N^2 = {squares}"
```

Lightweight syntax applies to all the major constructs of the F# syntax. In the next example, the code is incorrectly aligned. The declaration starts in the first line and continues to the second and subsequent lines, so those lines must be indented to the same column under the first line:

```fsharp
let computeDerivative f x =
    let p1 = f (x - 0.05)
    let p2 = f (x + 0.05)
        (p2 - p1) / 0.1
```

The following shows the correct alignment:

```fsharp
let computeDerivative f x =
    let p1 = f (x - 0.05)
    let p2 = f (x + 0.05)
    (p2 - p1) / 0.1
```

The use of lightweight syntax is the default for all Clef code in files with the extension `.clef` or `.clefx`.

### Making Data Simple

The first line in our sample simply declares a list of numbers from one through ten.

```fsharp
let numbers = [1 .. 10]
```

An F# list is an immutable linked list, which is a type of data used extensively in functional programming. Some operators that are related to lists include `::` to add an item to the front of a list and `@` to concatenate two lists:

```fsharp
let vowels = ['e'; 'i'; 'o'; 'u']
let withA = ['a'] @ vowels        // ['a'; 'e'; 'i'; 'o'; 'u']
let withY = vowels @ ['y']        // ['e'; 'i'; 'o'; 'u'; 'y']
```

F# supports several other highly effective techniques to simplify the process of modeling and manipulating data such as tuples, options, records, unions, and sequence expressions. A tuple is an ordered collection of values that is treated as an atomic unit. In many languages, if you want to pass around a group of related values as a single entity, you need to create a named type, such as a class or record, to store these values. A tuple allows you to keep things organized by grouping related values together, without introducing a new type.

To define a tuple, you separate the individual components with commas:

```fsharp
let tuple = (1, false, "text")            // int * bool * string
let getNumberInfo (x : int) = (x, x * x)  // int -> int * int
let info = getNumberInfo 42               // (42, 1764)
```

A key concept in F# is immutability. Tuples and lists are some of the many types in F# that are immutable, and indeed most things in F# are immutable by default. Immutability means that once a value is created and given a name, the value associated with the name cannot be changed. Immutability has several benefits. Most notably, it prevents many classes of bugs, and immutable data is inherently thread-safe, which makes the process of parallelizing code simpler.

### Making Types Simple

The next line of the sample program defines a function called `square`, which squares its input.

```fsharp
let square x = x * x
```

Most statically-typed languages require that you specify type information for a function declaration. However, F# typically infers this type information for you. This process is referred to as *type inference*.

From the function signature, F# knows that `square` takes a single parameter named `x` and that the function returns `x * x`. The last thing evaluated in an F# function body is the return value; hence there is no "return" keyword here. Many primitive types support the multiplication (*) operator (such as `int8`, `int64`, and `float`); however, for arithmetic operations, F# infers the type `int` by default.

> **Clef Note**: In Clef, `int` is the platform word size (64 bits on 64-bit platforms), not a fixed 32-bit integer. See [Native Type Mappings](native-type-mappings.md) for details.

Although F# can typically infer types on your behalf, occasionally you must provide explicit type annotations in F# code. For example, the following code uses a type annotation for one of the parameters to tell the compiler the type of the input.

```fsharp
let concat (x : string) y = x + y  // string -> string -> string
```

Because `x` is stated to be of type `string`, and the only version of the `+` operator that accepts a left-hand argument of type `string` also takes a `string` as the right-hand argument, the F# compiler infers that the parameter `y` must also be a string. Thus, the result of `x + y` is the concatenation of the strings. Without the type annotation, the F# compiler would not have known which version of the `+` operator was intended and would have assumed `int` data by default.

The process of type inference also applies *automatic generalization* to declarations. This automatically makes code *generic* when possible, which means the code can be used on many types of data. For example, the following code defines a function that returns a new tuple in which the two values are swapped:

```fsharp
let swap (x, y) = (y, x)       // 'a * 'b -> 'b * 'a
let swapped1 = swap (1, 2)     // (2, 1)
let swapped2 = swap ("you", true)  // (true, "you")
```

Here the function `swap` is generic, and `'a` and `'b` represent type variables, which are placeholders for types in generic code. Type inference and automatic generalization greatly simplify the process of writing reusable code fragments.

### Functional Programming

Continuing with the sample, we have a list of integers named `numbers`, and the `square` function, and we want to create a new list in which each item is the result of a call to our function. This is called *mapping* our function over each item in the list. The F# library function `List.map` does just that:

```fsharp
let squares = List.map square numbers
```

Consider another example:

```fsharp
let isEven = List.map (fun x -> x % 2 = 0) [1 .. 5]
// [false; true; false; true; false]
```

The code `(fun x -> x % 2 = 0)` defines an anonymous function, called a *function expression*, that takes a single parameter `x` and returns the result `x % 2 = 0`, which is a Boolean value that indicates whether `x` is even. The `->` symbol separates the argument list (`x`) from the function body (`x % 2 = 0`).

Both of these examples pass a function as a parameter to another function: the first parameter to `List.map` is itself another function. Using functions as *function values* is a hallmark of functional programming.

Another tool for data transformation and analysis is *pattern matching*. This powerful switch construct allows you to branch control flow and to bind new values. For example, we can match an F# list against a sequence of list elements.

```fsharp
let checkList alist =
    match alist with
    | [] -> 0
    | [a] -> 1
    | [a; b] -> 2
    | [a; b; c] -> 3
    | _ -> failwith "List is too big!"
```

In this example, `alist` is compared with each potentially matching pattern of elements. When `alist` matches a pattern, the result expression is evaluated and is returned as the value of the match expression. Here, the `->` operator separates a pattern from the result that a match returns.

Pattern matching can also be used as a control construct. In Clef, type-based dispatch uses discriminated unions rather than runtime type tests:

```fsharp
type Value =
    | StrVal of string
    | IntVal of int
    | Other

let describeValue (x : Value) =
    match x with
    | StrVal _ -> "x is a string"
    | IntVal _ -> "x is an int"
    | Other -> "x is something else"
```

This approach provides exhaustive pattern matching verified at compile time.

> **Clef Note**: The `:?` type test operator and `obj` type are not available in Clef. Use discriminated unions for type-safe variant handling. See [Native Type Mappings § The Universal Base Type `obj` Is Not Available](native-type-mappings.md#the-universal-base-type-obj-is-not-available).

Function values can also be combined with the *pipeline operator*, `|>`. For example, given these functions:

```fsharp
let square x = x * x
let toStr (x : int) = String.ofInt x
let reverse (x : string) = String.reverse x
```

We can use the functions as values in a pipeline:

```fsharp
let result = 32 |> square |> toStr |> reverse  // "4201"
```

Pipelining demonstrates one way in which F# supports *compositionality*, a key concept in functional programming. The pipeline operator simplifies the process of writing compositional code where the result of one function is passed into the next.

### Imperative Programming

The next line of the sample program prints text in the console window.

```fsharp
Console.WriteLine $"N^2 = {squares}"
```

The `Console.WriteLine` function is a simple and type-safe way to print text in the console. Consider this example, which prints an integer, a floating-point number, and a string:

```fsharp
Console.WriteLine $"{5} * {0.75} = {5.0 * 0.75}"
// 5 * 0.75 = 3.75
```

String interpolation with `$"..."` syntax provides type-safe formatting. The `%A` format can be used to print arbitrary data types (including lists) in `sprintf` and similar functions.

The `Console.WriteLine` function is an example of *imperative programming*, which means calling functions for their side effects. Other commonly used imperative programming techniques include arrays and dictionaries. F# programs typically use a mixture of functional and imperative techniques.

### Native Compilation and System Intrinsics

Clef compiles to standalone native executables. Platform-specific operations use **CCS intrinsics** - operations that are intrinsic to the native type universe:

```fsharp
// Sys intrinsics are recognized by CCS during type checking
// and compiled to platform-specific code by Alex
module Sys =
    val write : int -> nativeptr<byte> -> int -> int  // syscall on Unix
    val read  : int -> nativeptr<byte> -> int -> int  // syscall on Unix
    val exit  : int -> 'T                             // never returns
```

CCS recognizes these by module pattern (`Sys.*`, `NativePtr.*`) and the Firefly compiler (Alex component) provides implementations for each target platform (Linux, macOS, Windows, embedded, etc.).

> **See**: [Platform Bindings](platform-bindings.md) for the three-layer binding architecture including Sys intrinsics and quotation-based bindings for external libraries.

### Parallel and Asynchronous Programming

F# is both a *parallel* and a *reactive* language. During execution, F# programs can have multiple parallel active evaluations and multiple pending reactions.

One way to write parallel F# programs is to use F# *async* expressions. For example, the code below computes the Fibonacci function and schedules the computation of the numbers in parallel:

```fsharp
let rec fib x = if x < 2 then 1 else fib(x-1) + fib(x-2)

let fibs =
    Async.Parallel [ for i in 0..40 -> async { return fib(i) } ]
    |> Async.RunSynchronously

Console.WriteLine $"The Fibonacci numbers are {fibs}"
```

The preceding code sample shows multiple, parallel, CPU-bound computations.

### Strong Typing for Numerical Code

F# applies type checking and type inference to numerically-intensive domains through *units of measure inference and checking*. This feature allows you to type-check programs that manipulate numerical values that represent physical and abstract quantities in a stronger way than other typed languages, without losing any performance in your compiled code. You can think of this feature as providing a type system for numerical code.

Consider the following example:

```fsharp
[<Measure>] type kg
[<Measure>] type m
[<Measure>] type s

let gravityOnEarth = 9.81<m/s^2>
let heightOfTowerOfPisa = 55.86<m>
let speedOfImpact = sqrt(2.0 * gravityOnEarth * heightOfTowerOfPisa)
```

The `Measure` attribute tells F# that `kg`, `s`, and `m` are not really types in the usual sense of the word, but are used to build units of measure. Here `speedOfImpact` is inferred to have type `float<m/s>`.

> **Clef Extension**: Units of measure are also used for memory regions and access kinds. See [Memory Regions](memory-regions.md) and [Access Kinds](access-kinds.md).

### Object-Oriented Programming and Code Organization

The sample program shown at the start of this chapter is a *script*. Although scripts are excellent for rapid prototyping, they are not suitable for larger software components. F# supports the transition from scripting to structured code through several techniques.

The most important of these is *object-oriented programming* through the use of *class type definitions*, *interface type definitions*, and *object expressions*. Object-oriented programming is a primary application programming interface (API) design technique for controlling the complexity of large software projects. For example, here is a class definition for an encoder/decoder object.

```fsharp
/// Build an encoder/decoder object that maps characters to an
/// encoding and back. The encoding is specified by a sequence
/// of character pairs, for example, [('a','Z'); ('Z','a')]
type CharMapEncoder(symbols: seq<char*char>) =
    let swap (x, y) = (y, x)

    /// An immutable tree map for the encoding
    let fwd = symbols |> Map.ofSeq

    /// An immutable tree map for the decoding
    let bwd = symbols |> Seq.map swap |> Map.ofSeq

    let encode (s:string) =
        String.map (fun c -> Map.tryFind c fwd |> Option.defaultValue c) s

    let decode (s:string) =
        String.map (fun c -> Map.tryFind c bwd |> Option.defaultValue c) s

    /// Encode the input string
    member x.Encode(s) = encode s

    /// Decode the given string
    member x.Decode(s) = decode s
```

You can instantiate an object of this type as follows:

```fsharp
let rot13 (c:char) =
    char(int 'a' + ((int c - int 'a' + 13) % 26))
let encoder =
    CharMapEncoder( [for c in 'a'..'z' -> (c, rot13 c)])
```

And use the object as follows:

```fsharp
let encoded = "F# is fun!" |> encoder.Encode   // "F# vf sha!"
let decoded = encoded |> encoder.Decode        // "F# is fun!"
```

An interface type can encapsulate a family of object types:

```fsharp
type IEncoding =
    abstract Encode : string -> string
    abstract Decode : string -> string
```

In this example, `IEncoding` is an interface type that includes both `Encode` and `Decode` object types.

Both object expressions and type definitions can implement interface types. For example, here is an object expression that implements the `IEncoding` interface type:

```fsharp
let nullEncoder =
    { new IEncoding with
        member x.Encode(s) = s
        member x.Decode(s) = s }
```

*Modules* are a simple way to encapsulate code during rapid prototyping when you do not want to spend the time to design a strict object-oriented type hierarchy. In the following example, we place a portion of our original script in a module.

```fsharp
module ApplicationLogic =
    let numbers n = [1 .. n]
    let square x = x * x
    let squares n = numbers n |> List.map square

Console.WriteLine $"Squares up to 5 = {ApplicationLogic.squares 5}"
Console.WriteLine $"Squares up to 10 = {ApplicationLogic.squares 10}"
```

Modules are also used in the F# library design to associate extra functionality with types. For example, `List.map` is a function in a module.

Other mechanisms aimed at supporting software engineering include *signatures*, which can be used to give explicit types to components, and namespaces, which serve as a way of organizing the name hierarchies for larger APIs.

### Native Type Semantics

Clef uses the same syntax as standard F#, but types have native semantics:

| F# Syntax | Standard F# | Clef |
|-----------|-------------|-----------|
| `string` | `System.String` (UTF-16) | UTF-8 fat pointer |
| `option<'T>` | Reference type, nullable | `voption<'T>`, stack-allocated, non-nullable |
| `array<'T>` | `System.Array` | Fat pointer `{ptr, len}` |
| `int` | Fixed 32-bit | Platform word size |

> **See**: [Native Type Mappings](native-type-mappings.md) for the complete type mapping specification.

### Null-Free Semantics

Clef enforces null-free semantics for all types. There are no null values:

```fsharp
// COMPILE ERROR in Clef
let s: string = null         // Error: Cannot assign null

// Use option for optional values
let maybeValue: int option = None   // Stack-allocated, NOT null
```

> **See**: [Native Type Mappings](native-type-mappings.md#option) for option type semantics.

### Memory Regions and Access Kinds

Clef extends the type system with memory region types and access kinds for embedded and systems programming:

```fsharp
// Memory-mapped I/O with type-safe access
let gpioReg : Ptr<uint32, Peripheral, ReadWrite> = Ptr.ofAddress 0x48000000UL

// Read-only flash memory
let flashData : Ptr<byte, Flash, ReadOnly> = ...
```

> **See**: [Memory Regions](memory-regions.md) and [Access Kinds](access-kinds.md) for complete specifications.

## Notational Conventions in This Specification

This specification describes the Clef language by using a mixture of informal and semiformal techniques. All examples in this specification use lightweight syntax, unless otherwise specified.

Regular expressions are given in the usual notation, as shown in the table:

| Notation          | Meaning                                  |
| ----------------- | ---------------------------------------- |
| regexp+           | One or more occurrences                  |
| regexp*           | Zero or more occurrences                 |
| regexp?           | Zero or one occurrences                  |
| [ char - char ]   | Range of ASCII characters                |
| [ ^ char - char ] | Any characters except those in the range |

Unicode character classes are referred to by their abbreviation; for example, `\Lu` refers to any uppercase letter. The following characters are referred to using the indicated notation:

| Character | Name      | Notation                          |
| --------- | --------- | --------------------------------- |
| \b        | backspace | ASCII/UTF-8/UTF-32 code 08        |
| \n        | newline   | ASCII/UTF-8/UTF-32 code 10        |
| \r        | return    | ASCII/UTF-8/UTF-32 code 13        |
| \t        | tab       | ASCII/UTF-8/UTF-32 code 09        |

> **Clef Note**: The specification uses UTF-8 and UTF-32, not UTF-16. See [Native Type Mappings](native-type-mappings.md#string) for string encoding details.

Strings of characters that are clearly not a regular expression are written verbatim. Therefore, the following string

```fsgrammar
abstract
```

matches precisely the characters `abstract`.

Where appropriate, apostrophes and quotation marks enclose symbols that are used in the specification of the grammar itself, such as `'<'` and `'|'`. For example, the following regular expression matches `(+)` or `(-)`:

```fsgrammar
'(' (+|-) ')'
```

This regular expression matches precisely the characters `#if`:

```fsgrammar
"#if"
```

Regular expressions are typically used to specify tokens.

```fsgrammar
token token-name = regexp
```

In the grammar rules, the notation `element-name?` indicates an optional element. The notation `...` indicates repetition of the preceding non-terminal construct and the separator token. For example, `expr ',' ... ',' expr` means a sequence of one or more `expr` elements separated by commas.
