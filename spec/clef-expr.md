---
title: "ClefExpr: CCS Typed Expression Representation"
weight: 600
category: Compiler
status: normative
---

> **Status**: Draft
> **Phase**: A (Core Representation) - Part of CCS architecture
> **Last Updated**: 2026-01-12

## Overview

`ClefExpr` is the CCS (Clef Compiler Service) native typed expression representation. It provides an **expression-centric view** over [the SemanticGraph](program-semantic-graph.md), the core intermediate representation used by Clef compilation.

### Why ClefExpr Exists

CCS deliberately does NOT use FCS's `FSharpExpr` type. This is a principled architectural decision:

| Aspect | FSharpExpr (FCS) | ClefExpr (CCS) |
|--------|------------------|-------------------------|
| **Type System** | CLR types (`System.Int32`, etc.) | Native types (`NativeType.I32`, etc.) |
| **SRTP Resolution** | .NET method tables | `WitnessResolution` with native witnesses |
| **Memory Model** | GC-managed, runtime-determined | Arena/stack affinity, compile-time determined |
| **Dependencies** | BCL, assembly metadata | BCL-free, freestanding capable |
| **Runtime** | .NET CLR required | No runtime, standalone binaries |

ClefExpr is a **projection** over SemanticGraph - it materializes expression trees on demand from the underlying graph structure.

### Relationship to Other Documents

- **[introduction.md](introduction.md)**: CCS architectural overview
- **[native-type-universe.md](native-type-universe.md)**: The `NativeType` system that ClefExpr uses
- **[inference-procedures.md](inference-procedures.md)**: How types are inferred and attached

---

## Part 1: Core Design

### 1.1 Expression-Centric View

The SemanticGraph stores program structure as a graph of nodes connected by edges. Each node has:
- A unique `NodeId`
- A `SemanticKind` (what construct it represents)
- A `NativeType` (attached during construction, not post-hoc)
- Optional `WitnessResolution` (for SRTP calls)
- Reachability marks (`IsReachable` for soft-delete)

ClefExpr transforms this graph representation into a tree representation that's easier to:
- Pretty-print for debugging
- Serialize to JSON for intermediate inspection
- Navigate for IDE features (hover, go-to-definition)
- Reason about for optimization passes

### 1.2 Key Principle: Projection, Not Transformation

ClefExpr does **not** copy or transform data. It is a **view** that traverses the SemanticGraph and presents it in expression form. The underlying SemanticGraph remains the source of truth.

```
SemanticGraph (nodes + edges)
        ↓
  ClefExpr.fromNode
        ↓
ClefExpr (expression tree)
```

This design means:
- No data duplication
- Changes to SemanticGraph are immediately reflected
- Expression view can be regenerated at any point

---

## Part 2: Type Definition

### 2.1 Core Expression Type

```fsharp
[<RequireQualifiedAccess; NoComparison; NoEquality>]
type ClefExpr =
    // Bindings
    | LetBinding of name: string * isMutable: bool * value: ClefExpr *
                    body: ClefExpr option * ty: NativeType
    | LetRecBindings of bindings: (string * ClefExpr) list *
                        body: ClefExpr option

    // Functions
    | Lambda of parameters: (string * NativeType) list * body: ClefExpr *
                returnType: NativeType * srtp: WitnessResolution option
    | Application of func: ClefExpr * args: ClefExpr list *
                    returnType: NativeType * srtp: WitnessResolution option

    // Values
    | Literal of value: LiteralValue * ty: NativeType
    | Variable of name: string * ty: NativeType * isMutable: bool *
                  definitionId: NodeId option

    // Control Flow
    | IfThenElse of guard: ClefExpr * thenBranch: ClefExpr *
                   elseBranch: ClefExpr option * ty: NativeType
    | Match of scrutinee: ClefExpr * cases: NativeMatchCase list * ty: NativeType
    | Sequential of exprs: ClefExpr list * ty: NativeType
    | WhileLoop of guard: ClefExpr * body: ClefExpr
    | ForLoop of var: string * start: ClefExpr * finish: ClefExpr *
                isUp: bool * body: ClefExpr

    // Data Structures
    | RecordExpr of fields: (string * ClefExpr) list *
                   copyFrom: ClefExpr option * ty: NativeType
    | UnionCase of caseName: string * payload: ClefExpr option * ty: NativeType
    | TupleExpr of elements: ClefExpr list * ty: NativeType

    // Platform Integration (Layer 1 & 2 bindings)
    | Intrinsic of name: string * args: ClefExpr list * ty: NativeType
    | FFICall of descriptor: FFIDescriptor * args: ClefExpr list * ty: NativeType

    // SRTP (Statically Resolved Type Parameters)
    | TraitCall of memberName: string * constrainedTypes: NativeType list *
                  arg: ClefExpr * resolution: WitnessResolution option * ty: NativeType

    // Structure
    | ModuleDef of name: string * members: ClefExpr list
    | TypeDef of name: string * kind: TypeDefKind * members: ClefExpr list

    // Error Recovery
    | Error of message: string * range: SourceRange
```

### 2.2 Supporting Types

```fsharp
/// Native match case for pattern matching
type NativeMatchCase = {
    Pattern: NativePattern
    Guard: ClefExpr option
    Body: ClefExpr
}

/// Pattern for match cases
[<RequireQualifiedAccess>]
type NativePattern =
    | Wildcard
    | Named of name: string * ty: NativeType
    | Literal of value: LiteralValue
    | Constructor of caseName: string * args: NativePattern list
    | Tuple of elements: NativePattern list
    | Record of fields: (string * NativePattern) list
    | Or of left: NativePattern * right: NativePattern
    | And of left: NativePattern * right: NativePattern
    | As of pattern: NativePattern * name: string
    | Typed of pattern: NativePattern * ty: NativeType
    | Null
```

---

## Part 3: Key Expression Cases

### 3.1 Variable References

Variables carry their definition location, enabling navigation and inlining:

```fsharp
| Variable of name: string * ty: NativeType * isMutable: bool * definitionId: NodeId option
```

- `name`: The variable name as written in source
- `ty`: The resolved native type
- `isMutable`: Whether declared with `mutable`
- `definitionId`: Link to the defining node in SemanticGraph (for go-to-definition, inlining)

Example JSON output:
```json
{
  "kind": "Variable",
  "name": "Console.Write",
  "type": "TFun(string -> unit)",
  "isMutable": false,
  "definitionId": 12356
}
```

### 3.2 CCS Intrinsics (Layer 1)

Intrinsics represent calls to CCS-recognized native operations (syscalls, pointer operations, etc.):

```fsharp
| Intrinsic of name: string * args: ClefExpr list * ty: NativeType
```

- `name`: The intrinsic name (e.g., `"Sys.write"`, `"Mmio.read"`)
- `args`: The argument expressions
- `ty`: The return type

CCS recognizes these by module pattern (`Sys.*`, `Mmio.*`) and Alex provides platform-specific implementations.

Example JSON output:
```json
{
  "kind": "Intrinsic",
  "name": "Sys.write",
  "arguments": [
    {"kind": "Literal", "value": "Int32 1"},
    {"kind": "Variable", "name": "bufPtr"},
    {"kind": "Variable", "name": "len"}
  ],
  "type": "int"
}
```

### 3.3 FFI Calls (Layer 2)

FFI calls represent bindings to external libraries with quotation-carried metadata:

```fsharp
| FFICall of descriptor: FFIDescriptor * args: ClefExpr list * ty: NativeType
```

- `descriptor`: Metadata extracted from binding quotations (C name, calling convention, etc.)
- `args`: The argument expressions
- `ty`: The return type

These are recognized from binding libraries (Farscape-generated) and compiled with the correct ABI.

> **See**: [Platform Bindings](platform-bindings.md) for the three-layer binding architecture.

### 3.4 SRTP Trait Calls

SRTP (Statically Resolved Type Parameters) are fully resolved at compile time:

```fsharp
| TraitCall of memberName: string * constrainedTypes: NativeType list *
              arg: ClefExpr * resolution: WitnessResolution option * ty: NativeType
```

The `resolution` field contains:
```fsharp
type WitnessResolution = {
    Operator: string           // e.g., "$", "+"
    ArgType: NativeType        // The concrete type
    ResolvedMember: string     // Fully qualified member name
    Implementation: WitnessImplementation
}
```

When `resolution` is `Some`, the SRTP was successfully resolved. When `None`, resolution failed (a diagnostic is emitted).

---

## Part 4: Conversion from SemanticGraph

### 4.1 Entry Points

```fsharp
module ClefExpr =
    /// Get expression trees for all entry points
    let fromEntryPoints (graph: SemanticGraph) : ClefExpr list

    /// Get expression tree starting at a specific node
    let fromNode (graph: SemanticGraph) (nodeId: NodeId) : ClefExpr

    /// Find expression for a named binding
    let fromBinding (graph: SemanticGraph) (name: string) : ClefExpr option
```

### 4.2 Conversion Rules

Each `SemanticKind` maps to an `ClefExpr` case:

| SemanticKind | ClefExpr |
|--------------|------------------|
| `Binding(name, isMut, _, _)` | `LetBinding` with value from children |
| `Lambda(params, bodyId)` | `Lambda` with body from graph traversal |
| `Application(funcId, argIds)` | `Application` with func and args traversed |
| `VarRef(name, defId)` | `Variable` with mutability from definition |
| `Literal(value)` | `Literal` with attached type |
| `Sequential(nodeIds)` | `Sequential` with all nodes traversed |
| `Intrinsic(name)` | `Intrinsic` with children as args |
| `FFICall(descriptor)` | `FFICall` with descriptor and children as args |
| `TraitCall(member, types, argId)` | `TraitCall` with SRTP resolution attached |

---

## Part 5: Semantic Transformation Invariants

The SemanticGraph (and by extension, ClefExpr) maintains specific **construction invariants** that downstream stages can rely upon. These are enforced during graph construction in CCS - the graph is "correct by construction."

### 5.1 Lambda Desugaring

F# syntax allows multiple patterns in lambda expressions:

```fsharp
// Source syntax
fun x y z -> x + y + z
fun (a, b) (c, d) -> a + b + c + d
```

**Invariant**: Multi-parameter lambdas are unified to a **single canonical Lambda node** with a flat parameter list:

| Source Syntax | SemanticGraph Representation |
|--------------|------------------------------|
| `fun x -> e` | `Lambda([(x, tx)], body)` |
| `fun x y -> e` | `Lambda([(x, tx); (y, ty)], body)` |
| `fun x y z -> e` | `Lambda([(x, tx); (y, ty); (z, tz)], body)` |

**NOT** nested lambdas:
```
// WRONG - Not how CCS represents this
Lambda(x, Lambda(y, Lambda(z, body)))
```

This invariant means:
- All function arities are explicit in the Lambda node
- No need to "peel" nested lambdas during code generation
- Function types directly match parameter lists

**Pattern Lambda Desugaring**: Lambdas with patterns (not just variable names) elaborate to [pattern matching](patterns.md):

```fsharp
// Source
fun (a, b) -> a + b

// Elaborated representation
Lambda([($0, tuple)],
  Match($0, [TuplePattern(a, b) -> a + b]))
```

### 5.2 Curried Call Flattening

Fully-applied curried calls are represented as a **single Application node** with a flat argument list:

```fsharp
// Source
let greet prefix name = printfn "%s, %s!" prefix name
greet "Hello" "World"
```

**Invariant**: The call `greet "Hello" "World"` is represented as:
```
Application(Var(greet), ["Hello"; "World"])
```

**NOT** nested applications:
```
// WRONG - Not how CCS represents fully-applied calls
Application(Application(Var(greet), ["Hello"]), ["World"])
```

This invariant means:
- Fully-applied calls compile to direct multi-argument calls
- No intermediate closures are created for fully-applied curried functions
- Code generation receives all arguments at once

**Partial Application**: When a curried function is partially applied, the Application node contains only the supplied arguments, and the result type reflects the remaining curry:

```fsharp
let greetHello = greet "Hello"  // Partial application
// Represented as:
// Application(Var(greet), ["Hello"])
// with returnType = TFun(string, unit)
 
```

### 5.3 Pipe Operator Reduction

Pipe operators (`|>`, `<|`, `||>`, `<||`) are **fully reduced** during SemanticGraph construction:

```fsharp
// Source with pipe
name |> greet prefix
// Source without pipe (equivalent)
greet prefix name
```

**Invariant**: Pipe expressions are immediately transformed to direct application:
```
// Both source forms produce the same representation:
Application(Var(greet), [Var(prefix); Var(name)])
```

There is **no** `Pipe` node kind in the SemanticGraph. Pipes are syntactic sugar that is resolved during construction.

**Compound Example**:
```fsharp
// Source
Console.readln() |> greet prefix
```

Desugars to:
```
Application(Var(greet),
  [Var(prefix);
   Application(Intrinsic(Console.readln), [])])
```

### 5.4 Combined Transformation Example

The following demonstrates how multiple transformations combine:

```fsharp
// Source F#
let greet prefix name = Console.writeln $"{prefix}, {name}!"

let hello prefix =
    Console.write "Enter your name: "
    Console.readln() |> greet prefix

[<EntryPoint>]
let main argv =
    match argv with
    | [|prefix|] -> hello prefix
    | _ -> hello "Hello"
    0
```

After CCS construction, the `hello` function body appears as:
```
Lambda([(prefix, string)],
  Seq([
    Application(Intrinsic(Console.write), [Literal("Enter your name: ")]),
    Application(Var(greet), [Var(prefix), Application(Intrinsic(Console.readln), [])])
  ]))
```

Key observations:
1. `hello` is a single-parameter Lambda (not nested)
2. The pipe `|> greet prefix` is reduced to a flat `Application(greet, [prefix, readln()])`
3. The call `greet prefix name` is a single Application with two arguments
4. No intermediate `Pipe` or nested `Application` nodes exist

### 5.5 Recursive Binding NodeId Resolution

Recursive bindings (`let rec`) present a unique challenge: the function references itself before its definition is complete.

```fsharp
// Source
let rec factorial n =
    if n <= 1 then 1
    else n * factorial (n - 1)  // VarRef to 'factorial' - but we're defining it!
 
```

**The Problem**: When checking the body of `factorial`, we encounter a `VarRef` to `factorial`. But at that point, we haven't finished creating the Binding node, so we don't have a `NodeId` to link to.

**The Solution**: Pre-create Binding nodes to obtain NodeIds before checking bodies:

```fsharp
// Conceptual flow for let rec bindings:
// 1. Pre-create Binding nodes (get NodeIds)
// 2. Add bindings to environment WITH their NodeIds
// 3. Check body (VarRefs resolve via environment)
// 4. Connect body to Binding node via SetChildren
 
```

**Invariant**: All VarRefs, including self-references in recursive functions, have `defId = Some nodeId`:

```
// CORRECT - Self-reference has defId
VarRef ("factorial", Some (NodeId 526))

// WRONG - Missing defId
VarRef ("factorial", None)
```

This invariant applies to:
- **Simple recursion**: `let rec f x = ... f ...`
- **Nested recursion**: `let f x = let rec loop y = ... loop ... in loop x`
- **Mutual recursion**: `let rec f x = ... g ... and g y = ... f ...`

**Key Principle**: The NodeId must exist before we need to reference it. For recursive bindings:
1. Create the Binding node (get NodeId)
2. Add to environment with that NodeId
3. Check body (VarRefs resolve via environment)
4. Connect body to Binding node

**Downstream Reliance**: Alex and other consumers can safely assume every VarRef has a valid `defId` for name lookup. There are no "forward reference" special cases to handle.

### 5.6 Invariants Summary

| Transformation | Guarantee |
|---------------|-----------|
| Multi-param lambda | Single Lambda node with flat parameter list |
| Fully-applied curried call | Single Application node with flat argument list |
| Pipe operators | Reduced to direct Application |
| Pattern lambda | Elaborated to pattern matching on synthetic variable |
| Recursive bindings | All VarRefs (including self-references) have valid `defId` |

**Downstream Reliance**: These invariants allow Alex (and any other consumer) to treat the SemanticGraph as already normalized. Code generation does not need to perform additional flattening or reduction - the graph is correct by construction.

---

## Part 6: Debugging Output

### 6.1 Pretty-Print Format

ClefExpr provides a human-readable format for debugging:

```fsharp
let prettyPrint (indent: int) (expr: ClefExpr) : string
```

Example output:
```
=== Entry Point 0 ===
Module HelloWorldDirect:
  Let main =
    Lambda(argv) ->
      Seq:
        App(Var(Console.Write -> 12356), [Literal(String "Hello, World!")])
        Seq:
          App(Var(Console.WriteLine -> 12362), [Literal(String "")])
          Literal(Int32 0)
```

Key features:
- Indentation reflects nesting
- Variable references show definition IDs (`-> 12356`)
- Applications show function and argument list
- Sequences show child expressions

### 6.2 JSON Format

Structured JSON output for programmatic analysis:

```json
{
  "kind": "Application",
  "function": {
    "kind": "Variable",
    "name": "Console.Write",
    "definitionId": 12356
  },
  "arguments": [
    {
      "kind": "Literal",
      "value": "String \"Hello, World!\""
    }
  ],
  "returnType": "TFun(string -> unit)",
  "srtpResolution": null
}
```

### 6.3 Intermediate Files

When compiling with `-k` (keep intermediates), CCS emits:

| File | Description |
|------|-------------|
| `ccs_expr.json` | Full expression tree in JSON format |
| `ccs_expr.txt` | Pretty-printed text for human reading |
| `ccs_phase_*.json` | SemanticGraph at each phase |

---

## Part 7: Usage Patterns

### 7.1 Debugging Missing Inlines

When a function appears as an external reference in generated code:

1. Open `ccs_expr.txt` and find the function call
2. Check the `definitionId` on the Variable
3. If `null`, the definition wasn't captured in SemanticGraph
4. If present, trace to that node in `ccs_phase_5_final.json`

### 7.2 Tracing SRTP Resolution

For SRTP operators like `$` (the native string interpolation operator):

1. Find `TraitCall` nodes in `ccs_expr.json`
2. Check the `resolution` field
3. If `null`, SRTP resolution failed
4. If present, shows the resolved member and implementation

### 7.3 IDE Integration

ClefExpr enables IDE features:

- **Hover**: Traverse to node, show `ty` field
- **Go-to-Definition**: Follow `definitionId` to definition node
- **Find References**: Search for Variables with matching `definitionId`

---

## Appendix A: Complete Type Definition

The full type definition is in `ClefExpr.fs` within the CCS source tree (`Compiler/Checking.Native`).

Key modules:
- `ClefExpr` - The discriminated union type
- `NativePattern` - Pattern types for match cases
- `NativeMatchCase` - Match case with pattern, guard, body
- `ClefExpr` module - Conversion and pretty-printing functions

---

## Appendix B: Comparison with FCS FSharpExpr

| Feature | FSharpExpr | ClefExpr |
|---------|------------|------------------|
| Source | FCS typed tree | SemanticGraph projection |
| Types | `FSharpType` (CLR) | `NativeType` (native) |
| Null handling | Nullable references | Non-null by design |
| SRTP | Method table lookup | `WitnessResolution` |
| Memory | GC semantics | Arena/stack affinity |
| BCL dependency | Required | None |
| Freestanding | No | Yes |

ClefExpr is not a port or modification of FSharpExpr - it is a clean-room implementation with native-first semantics.

---

## Appendix C: Evolution Notes

### January 2026 - Initial Implementation

ClefExpr was introduced as part of the CCS nanopass infrastructure to provide:
1. Inspectable intermediates for debugging
2. Expression-centric view for tooling
3. Clean break from FCS's BCL-dependent representation

The implementation follows the principle that SemanticGraph is the source of truth, and ClefExpr is a derived view for convenience.
