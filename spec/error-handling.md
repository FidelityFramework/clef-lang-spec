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
src/Main.clef(12,5)-(12,15): error CCS8100: Cannot use 'null' in Clef; use 'ValueNone' for optional values
```

#### Error Codes

CCS uses error codes in the CCS8xxx range to distinguish native-specific diagnostics:

| Range | Category |
|-------|----------|
| CCS8000-CCS8099 | Type system (null-freedom, [access kinds](access-kinds.md)) |
| CCS8100-CCS8199 | Memory management (regions, lifetimes) |
| CCS8200-CCS8299 | Platform bindings |
| CCS8300-CCS8399 | Effect system |
| CCS8400-CCS8499 | Code generation |

#### LSP Compatibility

CCS implements the Language Server Protocol for editor integration. Key considerations:

1. **Diagnostic Publishing**: Errors are published via `textDocument/publishDiagnostics` in standard LSP format
2. **Code Actions**: Quick fixes (e.g., "Replace null with ValueNone") are provided via `textDocument/codeAction`
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

| Code | Severity | Message |
|------|----------|---------|
| CCS8100 | Error | Cannot use 'null' in Clef; use 'ValueNone' for optional values |
| CCS8101 | Error | Cannot use 'null' in Clef; all values must be initialized |
| CCS8102 | Warning | Exception-style error handling detected; consider Result-based pattern |
| CCS8103 | Error | Type does not support 'null' in Clef |
| CCS8104 | Warning | Unchecked.defaultof<'T> produces undefined behavior for reference types |

## Areas Requiring Further Specification

The following areas require additional design work:

1. **Effect System Integration**: How Result interacts with algebraic effects and delimited continuations
2. **Async/Concurrent Errors**: Error propagation in concurrent and asynchronous contexts
3. **Interop Boundaries**: Error translation at [FFI boundaries](ffi-boundary.md) with C libraries
4. **Panic vs. Error**: Distinction between recoverable errors (Result) and unrecoverable panics
5. **Stack Traces**: Diagnostic information for debugging without managed exception infrastructure
6. **Tooling PR Strategy**: Concrete changes needed for Lattice/FSAC to support CCS
