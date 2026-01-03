# FSharpNativeExpr: FNCS Typed Expression Representation

> **Status**: Draft
> **Phase**: A (Core Representation) - Part of FNCS architecture
> **Last Updated**: 2026-01-03

## Overview

`FSharpNativeExpr` is FNCS's native typed expression representation. It provides an **expression-centric view** over the SemanticGraph, the core intermediate representation used by F# Native compilation.

### Why FSharpNativeExpr Exists

FNCS deliberately does NOT use FCS's `FSharpExpr` type. This is a principled architectural decision:

| Aspect | FSharpExpr (FCS) | FSharpNativeExpr (FNCS) |
|--------|------------------|-------------------------|
| **Type System** | CLR types (`System.Int32`, etc.) | Native types (`NativeType.I32`, etc.) |
| **SRTP Resolution** | .NET method tables | `WitnessResolution` with Alloy witnesses |
| **Memory Model** | GC-managed, runtime-determined | Arena/stack affinity, compile-time determined |
| **Dependencies** | BCL, assembly metadata | BCL-free, freestanding capable |
| **Runtime** | .NET CLR required | No runtime, standalone binaries |

FSharpNativeExpr is a **projection** over SemanticGraph - it materializes expression trees on demand from the underlying graph structure.

### Relationship to Other Documents

- **[introduction.md](introduction.md)**: FNCS architectural overview
- **[native-type-universe.md](native-type-universe.md)**: The `NativeType` system that FSharpNativeExpr uses
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

FSharpNativeExpr transforms this graph representation into a tree representation that's easier to:
- Pretty-print for debugging
- Serialize to JSON for intermediate inspection
- Navigate for IDE features (hover, go-to-definition)
- Reason about for optimization passes

### 1.2 Key Principle: Projection, Not Transformation

FSharpNativeExpr does **not** copy or transform data. It is a **view** that traverses the SemanticGraph and presents it in expression form. The underlying SemanticGraph remains the source of truth.

```
SemanticGraph (nodes + edges)
        ↓
  FSharpNativeExpr.fromNode
        ↓
FSharpNativeExpr (expression tree)
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
type FSharpNativeExpr =
    // Bindings
    | LetBinding of name: string * isMutable: bool * value: FSharpNativeExpr *
                    body: FSharpNativeExpr option * ty: NativeType
    | LetRecBindings of bindings: (string * FSharpNativeExpr) list *
                        body: FSharpNativeExpr option

    // Functions
    | Lambda of parameters: (string * NativeType) list * body: FSharpNativeExpr *
                returnType: NativeType * srtp: WitnessResolution option
    | Application of func: FSharpNativeExpr * args: FSharpNativeExpr list *
                    returnType: NativeType * srtp: WitnessResolution option

    // Values
    | Literal of value: LiteralValue * ty: NativeType
    | Variable of name: string * ty: NativeType * isMutable: bool *
                  definitionId: NodeId option

    // Control Flow
    | IfThenElse of guard: FSharpNativeExpr * thenBranch: FSharpNativeExpr *
                   elseBranch: FSharpNativeExpr option * ty: NativeType
    | Match of scrutinee: FSharpNativeExpr * cases: NativeMatchCase list * ty: NativeType
    | Sequential of exprs: FSharpNativeExpr list * ty: NativeType
    | WhileLoop of guard: FSharpNativeExpr * body: FSharpNativeExpr
    | ForLoop of var: string * start: FSharpNativeExpr * finish: FSharpNativeExpr *
                isUp: bool * body: FSharpNativeExpr

    // Data Structures
    | RecordExpr of fields: (string * FSharpNativeExpr) list *
                   copyFrom: FSharpNativeExpr option * ty: NativeType
    | UnionCase of caseName: string * payload: FSharpNativeExpr option * ty: NativeType
    | TupleExpr of elements: FSharpNativeExpr list * ty: NativeType

    // Platform Integration (Layer 1 & 2 bindings)
    | Intrinsic of name: string * args: FSharpNativeExpr list * ty: NativeType
    | FFICall of descriptor: FFIDescriptor * args: FSharpNativeExpr list * ty: NativeType

    // SRTP (Statically Resolved Type Parameters)
    | TraitCall of memberName: string * constrainedTypes: NativeType list *
                  arg: FSharpNativeExpr * resolution: WitnessResolution option * ty: NativeType

    // Structure
    | ModuleDef of name: string * members: FSharpNativeExpr list
    | TypeDef of name: string * kind: TypeDefKind * members: FSharpNativeExpr list

    // Error Recovery
    | Error of message: string * range: SourceRange
```

### 2.2 Supporting Types

```fsharp
/// Native match case for pattern matching
type NativeMatchCase = {
    Pattern: NativePattern
    Guard: FSharpNativeExpr option
    Body: FSharpNativeExpr
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

### 3.2 FNCS Intrinsics (Layer 1)

Intrinsics represent calls to FNCS-recognized native operations (syscalls, pointer operations, etc.):

```fsharp
| Intrinsic of name: string * args: FSharpNativeExpr list * ty: NativeType
```

- `name`: The intrinsic name (e.g., `"Sys.write"`, `"NativePtr.set"`)
- `args`: The argument expressions
- `ty`: The return type

FNCS recognizes these by module pattern (`Sys.*`, `NativePtr.*`) and Alex provides platform-specific implementations.

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
| FFICall of descriptor: FFIDescriptor * args: FSharpNativeExpr list * ty: NativeType
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
              arg: FSharpNativeExpr * resolution: WitnessResolution option * ty: NativeType
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
module FSharpNativeExpr =
    /// Get expression trees for all entry points
    let fromEntryPoints (graph: SemanticGraph) : FSharpNativeExpr list

    /// Get expression tree starting at a specific node
    let fromNode (graph: SemanticGraph) (nodeId: NodeId) : FSharpNativeExpr

    /// Find expression for a named binding
    let fromBinding (graph: SemanticGraph) (name: string) : FSharpNativeExpr option
```

### 4.2 Conversion Rules

Each `SemanticKind` maps to an `FSharpNativeExpr` case:

| SemanticKind | FSharpNativeExpr |
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

## Part 5: Debugging Output

### 5.1 Pretty-Print Format

FSharpNativeExpr provides a human-readable format for debugging:

```fsharp
let prettyPrint (indent: int) (expr: FSharpNativeExpr) : string
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

### 5.2 JSON Format

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

### 5.3 Intermediate Files

When compiling with `-k` (keep intermediates), FNCS emits:

| File | Description |
|------|-------------|
| `fncs_expr.json` | Full expression tree in JSON format |
| `fncs_expr.txt` | Pretty-printed text for human reading |
| `fncs_phase_*.json` | SemanticGraph at each phase |

---

## Part 6: Usage Patterns

### 6.1 Debugging Missing Inlines

When a function appears as an external reference in generated code:

1. Open `fncs_expr.txt` and find the function call
2. Check the `definitionId` on the Variable
3. If `null`, the definition wasn't captured in SemanticGraph
4. If present, trace to that node in `fncs_phase_5_final.json`

### 6.2 Tracing SRTP Resolution

For SRTP operators like `$` (the native string interpolation operator):

1. Find `TraitCall` nodes in `fncs_expr.json`
2. Check the `resolution` field
3. If `null`, SRTP resolution failed
4. If present, shows the resolved member and implementation

### 6.3 IDE Integration

FSharpNativeExpr enables IDE features:

- **Hover**: Traverse to node, show `ty` field
- **Go-to-Definition**: Follow `definitionId` to definition node
- **Find References**: Search for Variables with matching `definitionId`

---

## Appendix A: Complete Type Definition

The full type definition is in:
```
/home/hhh/repos/fsnative/src/Compiler/Checking.Native/FSharpNativeExpr.fs
```

Key modules:
- `FSharpNativeExpr` - The discriminated union type
- `NativePattern` - Pattern types for match cases
- `NativeMatchCase` - Match case with pattern, guard, body
- `FSharpNativeExpr` module - Conversion and pretty-printing functions

---

## Appendix B: Comparison with FCS FSharpExpr

| Feature | FSharpExpr | FSharpNativeExpr |
|---------|------------|------------------|
| Source | FCS typed tree | SemanticGraph projection |
| Types | `FSharpType` (CLR) | `NativeType` (native) |
| Null handling | Nullable references | Non-null by design |
| SRTP | Method table lookup | `WitnessResolution` |
| Memory | GC semantics | Arena/stack affinity |
| BCL dependency | Required | None |
| Freestanding | No | Yes |

FSharpNativeExpr is not a port or modification of FSharpExpr - it is a clean-room implementation with native-first semantics.

---

## Appendix C: Evolution Notes

### January 2026 - Initial Implementation

FSharpNativeExpr was introduced as part of the FNCS nanopass infrastructure to provide:
1. Inspectable intermediates for debugging
2. Expression-centric view for tooling
3. Clean break from FCS's BCL-dependent representation

The implementation follows the principle that SemanticGraph is the source of truth, and FSharpNativeExpr is a derived view for convenience.
