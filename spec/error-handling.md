---
title: "Error Handling"
weight: 240
category: Semantics
status: normative
---

This chapter specifies error handling semantics in Clef, including the relationship between compile-time error propagation through tooling and runtime error handling in compiled applications.

## Overview

Clef takes a fundamentally different approach to error handling than managed F#:

| Aspect | Managed F# | Clef |
|--------|------------|-----------|
| Optional values | `option<'T>` with null representation for `None` | `voption<'T>` (ValueOption) with no null |
| Failure handling | Exceptions (`raise`, `try/with`) | `Result<'T, 'E>` with explicit propagation |
| Null values | Permitted for reference types | Not permitted; null-free by construction |
| Runtime type errors | `InvalidCastException`, `NullReferenceException` | Cannot occur; prevented by type system |

> **Design Principle**: Errors are values, not control flow. The type system encodes fallibility explicitly, making error handling visible and verifiable at compile time.

## The Dual Nature of Error Handling

Clef error handling operates at two distinct levels:

1. **Application Runtime**: How compiled Fidelity framework applications handle errors during execution
2. **Tooling Integration**: How Clef Compiler Service (CCS) propagates errors through the Language Server Protocol to editors like Lattice

These two domains have different requirements and constraints, but must remain coherent.

## Application Runtime Error Handling

### The Result Type

The [`Result<'T, 'E>` type](discriminated-union-representation.md) is the primary mechanism for representing operations that may fail:

```fsharp
type Result<'T, 'E> =
    | Ok of 'T
    | Error of 'E
```

Operations that can fail return `Result` values rather than raising exceptions:

```fsharp
// Managed F# style (NOT used in Clef applications)
let divide x y =
    if y = 0 then raise (DivideByZeroException())
    else x / y

// Clef style
let divide x y : Result<int, DivisionError> =
    if y = 0 then Error DivisionByZero
    else Ok (x / y)
```

### Standard Error Types

Clef defines standard error types for common failure modes:

```fsharp
type ArithmeticError =
    | DivisionByZero
    | Overflow
    | Underflow

type IndexError =
    | OutOfBounds of index: int * length: int

type ParseError =
    | InvalidFormat of input: string * expected: string
    | UnexpectedEnd

type IOError =
    | NotFound of path: string
    | PermissionDenied of path: string
    | DeviceError of code: int
```

> **Note**: The exact set of standard error types is subject to further specification as the standard library matures.

### The voption Type

For optional values where absence is not an error, `voption<'T>` (ValueOption) provides a null-free representation:

```fsharp
type voption<'T> =
    | ValueSome of 'T
    | ValueNone
```

The `voption<'T>` type has an explicit discriminator with no null representation.

```fsharp
// Looking up a value that may not exist
let tryFind key (map: Map<'K, 'V>) : voption<'V> =
    match Map.tryFind key map with
    | ValueSome v -> ValueSome v
    | ValueNone -> ValueNone
```

### Result Propagation

Clef provides computation expression syntax for Result propagation:

```fsharp
let result {
    let! x = tryParseInt "42"
    let! y = tryParseInt "17"
    return x + y
}
```

This is equivalent to explicit binding:

```fsharp
match tryParseInt "42" with
| Error e -> Error e
| Ok x ->
    match tryParseInt "17" with
    | Error e -> Error e
    | Ok y -> Ok (x + y)
```

### Try/With Syntax Compatibility

Clef preserves `try`/`with`/`finally` syntax for compatibility with standard F# tooling:

```fsharp
try
    riskyOperation()
with
| :? SomeException as e -> handleError e
```

However, the semantics differ:

- In Clef, `try`/`with` may be used for effect handling (delimited continuations) rather than exception catching
- The exact semantics depend on the effect system specification
- Code using `try`/`with` for exception handling must be migrated to Result-based patterns for native compilation

> **Tooling Note**: Lattice and other editors will parse `try`/`with` expressions normally. CCS may emit warnings when exception-style patterns are detected, guiding migration to Result-based alternatives.

### Null-Freedom

[Clef is null-free by construction](special-attributes-and-types.md). The following are compile-time errors:

```fsharp
let x : string = null           // ERROR: null literal not available
let y = Unchecked.defaultof<_>  // ERROR for reference types in most contexts
 
```

This eliminates entire classes of runtime errors:

| Managed F# Runtime Error | Clef |
|--------------------------|-----------|
| `NullReferenceException` | Cannot occur |
| `InvalidCastException` | Cannot occur (static typing) |
| `ArrayTypeMismatchException` | Cannot occur (no covariant arrays) |

## Tooling Integration

### CCS Error Propagation

Clef Compiler Service (CCS) must propagate errors through the tooling stack in a format compatible with existing F# tooling infrastructure.

#### Diagnostic Format

CCS diagnostics follow the F# compiler diagnostic format:

```
filepath(line,col)-(line,col): severity code: message
```

For example:

```
src/Main.clef(12,5)-(12,15): error CCS8010: 'null' is not permitted in Clef; use 'ValueNone' for an absent value
```

#### Error Codes

CCS uses error codes in the CCS8xxx range to distinguish native-specific diagnostics:

| Range | Category |
|-------|----------|
| CCS8000-CCS8099 | Type system: identity, measures, seals and ranges, null-freedom (`CCS8010`, [Types and Type Constraints](types-and-type-constraints.md)), [access kinds](access-kinds.md) |
| CCS8100-CCS8199 | Memory management (regions, lifetimes) |
| CCS8200-CCS8299 | Platform bindings |
| CCS8300-CCS8399 | Effect system |
| CCS8400-CCS8499 | Code generation |

#### The CCS code table

Decision D3 of the dimensional hardening (2026-09-04): every diagnostic the compiler service reports carries a code in the CCS series; the `FS` prefix is retired. Codes are allocated inside the blocks above and never reassigned. Inherited lexer and parser diagnostics keep their F# number under the CCS prefix (`FS0058` becomes `CCS0058`); the block `CCS0000`–`CCS0999` is reserved for that family.

| Code | Severity | Meaning |
|------|----------|---------|
| CCS8000 | Error | An operator's operand is not numeric ([Width Inference](width-inference.md), the numeric constraint) |
| CCS8001 | Error | The kind of an operator's operands cannot be determined at a binding that is not generalisable |
| CCS8002 | Error | A conversion's source is not numeric |
| CCS8003 | Error | Type mismatch |
| CCS8004 | Error | Type constructor arity mismatch |
| CCS8005 | Error | Infinite type (a type variable occurs in its own solution) |
| CCS8006 | Error | Tuple mismatch (length or struct kind) |
| CCS8007 | Error | Byref kind mismatch |
| CCS8008 | Error | The constructor is not defined |
| CCS8009 | Error | The value or constructor is not defined |
| CCS8010 | Error | The `null` keyword is not permitted ([Types and Type Constraints](types-and-type-constraints.md)) |
| CCS8011 | Error | An integer or dimensioned real whose range is unobservable; for a bare real flowing into a dimension, at the dimensioning seam naming the bare source ([Width Inference §6](width-inference.md), [Numeric Selection §6](numeric-selection.md)) |
| CCS8012 | Warning (error under `--warnaserror`) | A value's analysed range is not covered by the boundary's declared representation ([Numeric Selection §5](numeric-selection.md)) |
| CCS8013 | retired | "two seals meet": there are no seals ([NTU Types](ntu-types.md)) |
| CCS8014 | Info | A declared boundary representation wider than the range requires; the representation the open argmin would select is named |
| CCS8015 | retired | "sealed arithmetic may wrap": arithmetic on analysed ranges never overflows |
| CCS8016 | Warning (error under `--warnaserror`) | An analysed range exceeds a higher-provenance claim, a library law's range or a declaration ([Numeric Selection §3.4](numeric-selection.md)) |
| CCS8017 | retired | "a conversion cannot hold the range": there are no conversions |
| CCS8018 | Error | A literal suffix: every width suffix (`L`, `u`, `uy`, `s`, `n`, `f`), `I`, and any suffix the language does not have ([Width Inference §7](width-inference.md)) |
| CCS8020–CCS8022 | Error | Access kinds ([Access Kinds](access-kinds.md)) |
| CCS8030–CCS8033 | Error | Platform intrinsics ([Platform Bindings](platform-bindings.md)) |
| CCS8040–CCS8050 | Error | Units of measure ([Units of Measure](units-of-measure.md)): mismatch, no integer solution, not in scope, cyclic abbreviation, variable in a literal, sort mismatch, no dimension, unresolved at a non-generalisable binding, rational exponent, parameterised definition, arity |
| CCS8060 | Error | `obj` is not a Clef type |
| CCS8061 | Error | Boxing is not a Clef operation |
| CCS8062 | Error | Dynamic invocation is not a Clef operation |
| CCS8063 | Error | Quote expression patterns are not a Clef construct |
| CCS8064 | Error | Instance member patterns (object expressions) are not a Clef construct |
| CCS8065 | Error | Expression splices (`%e`, `%%e`) are not a Clef construct: a quotation is compile-time data read whole ([Expressions, Quoted Expressions](expressions.md)) |
| CCS8066 | Error | A quotation referenced from executed code: a quotation has no run-time value; reported at each reachable reference, or at the quotation when it stands in executed expression position |
| CCS8080 | Error | A BCL type or namespace is not available in Clef |
| CCS8081 | Error | The `System` namespace is not available in Clef |
| CCS8082 | Error | The `Microsoft` namespace is not available in Clef |
| CCS8083 | Error | `Unchecked.defaultof` is not available in Clef |
| CCS8090 | Error | Internal invariant violated in the compiler service (reported, never swallowed) |
| CCS8091 | Warning | A nullable annotation is ignored; native types are null-free by design |
| CCS8092 | Warning | Type arguments applied to a value that is not a type scheme |
| CCS8100 | Error | Region mismatch ([Memory Regions](memory-regions.md)) |
| CCS8101 | Error | Lifetime error |
| CCS8102 | Error | A reference escapes its region |
| CCS8200 | Error | Platform binding error |
| CCS8201 | Error | Unsupported platform operation |
| CCS8202 | Error | Platform binding undefined |
| CCS8203 | Error | A site needs a width dimension the platform description does not declare ([Platform Bindings](platform-bindings.md), [NTU Dimensional Architecture §7.1](ntu-dimensional-architecture.md)); never a default |
| CCS8204 | Error | A sealed value's representation is not offered by the platform description, absent or declared unavailable ([Numeric Selection §7](numeric-selection.md)) |
| CCS8205 | Info | A `[platform]` key the project file carries that the compiler does not read (`word_size`): width dimensions and representations come from the platform description |
| CCS8206 | Error | An element of the platform description the compiler cannot read (a field that is not a literal, an element that is not the record its list is declared over, a `Core` that is neither `Some core` nor `None`), reported at the declaration |
| CCS8207 | Error | An element of the platform description outside its vocabulary (a capability, family or boundary tag not in its closed set, a width or representation of no bits, a name declared twice, a Register width disagreeing with the word size), reported at the declaration |
| CCS8208 | Error | A second platform description of one form among the platform binding's sources; the first is read, each other is reported at its declaration |
| CCS8300 | Warning | Exception-style error handling detected; use the Result-based pattern |
| CCS8400 | Error | Code generation error |
| CCS8401 | Error | Unsupported construct in code generation |
| CCS8500–CCS8505 | see [Interactive Development](interactive-development.md) | Interactive session |
| CCS8701–CCS8705 | Error | Record field label resolution ([Name Resolution](inference-name-resolution.md)) |
| CCS8706 | Error | A type name in an annotation that resolves to nothing (no abbreviation, definition, primitive or built-in constructor), reported at the annotation; the error type it leaves unifies with anything, so this is the one report of the failure |
| CCS8710 | Error | Null constraint is not a Clef constraint |
| CCS8711 | Error | Unsupported constraint |

#### LSP Compatibility

CCS implements the Language Server Protocol for editor integration. Key considerations:

1. **Diagnostic Publishing**: Errors are published via `textDocument/publishDiagnostics` in standard LSP format
2. **Code Actions**: Quick fixes (e.g., "Insert the explicit conversion", "Add the measure annotation") are provided via `textDocument/codeAction`
3. **Hover Information**: Type information displays native types

### Lattice Integration Model

The multi-target editor model is well established in the F# ecosystem: Ionide routes a single editing experience across several F# compilation backends.

| Target | Integration Point |
|--------|-------------------|
| .NET | FSharp.Compiler.Service |
| Fable | Fable.Compiler (JavaScript output) |
| WebSharper | WebSharper.Compiler |

Lattice, the Clef editor tooling, follows this model:

```
Lattice ←→ LSP ←→ CCS ←→ Composer Compiler ←→ MLIR/LLVM
```

#### Extension Points

CCS provides extension points for Lattice integration:

1. **Project Recognition**: `.fidproj` files identify Clef projects
2. **Target Selection**: Lattice can route to CCS when native compilation is detected
3. **Shared Parsing**: Syntax parsing uses standard F# lexer/parser for compatibility
4. **Semantic Divergence**: Type checking and code generation use native semantics

#### Compatibility Considerations

To maintain compatibility with the broader F# ecosystem:

1. **Syntax Compatibility**: Clef code parses as valid F# syntax
2. **Type Notation**: Types are expressed using standard F# type notation
3. **Error Format**: Diagnostics follow F# compiler conventions
4. **Incremental Adoption**: Projects can mix managed and native targets during migration

### Editor Experience

The design-time experience for Clef should be consistent with managed F#:

| Feature | Behavior |
|---------|----------|
| Syntax highlighting | Standard F# highlighting |
| Error underlining | Red squiggles for errors, yellow for warnings |
| Hover types | Shows native type representations |
| Autocomplete | Suggests native library members |
| Go to definition | Navigates to native library source |
| Quick fixes | Offers native-appropriate fixes |

## Error Handling Patterns

### Railway-Oriented Programming

Clef encourages railway-oriented programming with Result:

```fsharp
let processOrder orderId =
    orderId
    |> validateOrderId
    |> Result.bind fetchOrder
    |> Result.bind validateInventory
    |> Result.bind processPayment
    |> Result.bind shipOrder
```

### Error Aggregation

For operations that may produce multiple errors:

```fsharp
type ValidationErrors = ValidationErrors of ValidationError list

let validateAll validators input =
    validators
    |> List.map (fun v -> v input)
    |> List.fold aggregateErrors (Ok input)
```

### Partial Success

For operations where partial results are meaningful:

```fsharp
type PartialResult<'T, 'E> =
    | Complete of 'T
    | Partial of 'T * 'E list
    | Failed of 'E list
```

## Grammar

```fsgrammar
result-type := Result < type , type >

voption-type := voption < type >

result-expr :=
    Ok expr
    Error expr

voption-expr :=
    ValueSome expr
    ValueNone

result-bind := let! pattern = expr in expr

result-return := return expr
```

## Diagnostics

Null-freedom is by construction and has exactly one diagnostic, `CCS8010` (the `null` keyword is not permitted, [Types and Type Constraints](types-and-type-constraints.md)). There is no second null diagnostic: no type "supports null", no value is uninitialised, and `Unchecked.defaultof` is BCL surface rejected as such. The codes this chapter contributes:

| Code | Severity | Message |
|------|----------|---------|
| CCS8100 | Error | Region mismatch: a handle of region '{r1}' where region '{r2}' is required ([Memory Regions](memory-regions.md)) |
| CCS8300 | Warning | Exception-style error handling detected; use the Result-based pattern |

A warning is promoted to an error under the `--warnaserror` policy, the rule every warning in the framework follows (the FPGA timing budget's `CCS0100` is the reference case).

## Areas Requiring Further Specification

The following areas require additional design work:

1. **Effect System Integration**: How Result interacts with algebraic effects and delimited continuations
2. **Async/Concurrent Errors**: Error propagation in concurrent and asynchronous contexts
3. **Interop Boundaries**: Error translation at [FFI boundaries](ffi-boundary.md) with C libraries
4. **Panic vs. Error**: Distinction between recoverable errors (Result) and unrecoverable panics
5. **Stack Traces**: Diagnostic information for debugging without managed exception infrastructure
6. **Tooling PR Strategy**: Concrete changes needed for Lattice/FSAC to support CCS
