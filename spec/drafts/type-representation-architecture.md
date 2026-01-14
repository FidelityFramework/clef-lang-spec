# Type Representation Architecture

> **Status**: Draft
> **Normative**: Yes
> **Last Updated**: 2026-01-06

## 1. Overview

This chapter specifies the internal representation of types in FNCS, particularly how composite types (records, unions) are represented and how type metadata is accessed. The design follows FCS's TyconRef.Deref pattern.

## 2. Core Principle: Single Representation with Deferred Lookup

**Type references should be REFERENCES, not copies. Metadata lives in one authoritative location (TypeDef) and is accessed via lookup when needed.**

This principle is borrowed from FCS where:
- `TType_app(tyconRef, typeInstantiation, nullness)` is the uniform representation
- Field information is accessed via `TyconRef.Deref`
- Type identity is determined by the tyconRef, not embedded data

## 3. TypeConRef Structure

### 3.1 Definition

```fsharp
type TypeConRef = {
    /// Type name (fully qualified within module)
    Name: string

    /// Module path
    Module: ModulePath

    /// Type parameters
    ParamKinds: TypeParamKind list

    /// Memory layout
    Layout: TypeLayout

    /// NTU kind for primitive types
    NTUKind: NTUKind option

    /// Field count for record types (0 = not a record)
    FieldCount: int
}
```

### 3.2 Why FieldCount Instead of Fields?

F# requires types to be defined before use. Since `NativeType` is a recursive type containing `TypeConRef`, embedding fields would create a forward reference:

```fsharp
// NOT POSSIBLE - forward reference
type TypeConRef = {
    Fields: (string * NativeType) list  // NativeType not yet defined!
}

and NativeType =
    | TApp of TypeConRef * NativeType list
```

Instead, `FieldCount` provides:
1. **Record detection**: `tycon.FieldCount > 0` indicates a record type
2. **Reachability signal**: Types with fields need their TypeDefs preserved
3. **Validation**: Expected field count for consistency checks

### 3.3 Field Lookup Pattern (FCS TyconRef.Deref)

Actual field information is stored in `TypeDef` nodes and accessed via lookup:

```fsharp
/// TypeDefKind for records
type TypeDefKind =
    | RecordDef of fields: (string * NativeType) list
    | UnionDef of cases: (string * (string option * NativeType) list) list
    | EnumDef of underlying: NativeType option
    | AbbrevDef of expanded: NativeType

/// Lookup fields from SemanticGraph
let tryGetRecordFields (typeName: string) (graph: SemanticGraph) =
    match recallType typeName graph with
    | Some nodeId ->
        match tryGetNode nodeId graph with
        | Some { Kind = SemanticKind.TypeDef(_, TypeDefKind.RecordDef fields, _) } ->
            Some fields
        | _ -> None
    | None -> None
```

## 4. Type Construction

### 4.1 Record Types

```fsharp
// Creating a record TypeConRef
let mkRecordTypeConRef name modulePath layout fieldCount =
    { Name = name
      Module = modulePath
      ParamKinds = []
      Layout = layout
      NTUKind = None
      FieldCount = fieldCount }

// Record types use TApp, NOT a separate TRecord constructor
let recordType = NativeType.TApp(mkRecordTypeConRef "Person" path layout 2, [])
```

### 4.2 Why TApp for Records?

Historical context: Early FNCS had both `TRecord` and `TApp`:

```fsharp
// DEPRECATED - causes type identity confusion
| TRecord of tycon: TypeConRef * fields: (string * NativeType) list
| TApp of tycon: TypeConRef * args: NativeType list
```

This dual representation caused:
- Unification failures ("expected TApp, got TRecord")
- Inconsistent field access patterns
- Code duplication for handling both cases

The unified approach uses `TApp` exclusively:
- Record fields accessed via `tryGetRecordFields`
- Union cases accessed via `tryGetUnionCases`
- Consistent type identity based on TypeConRef

## 5. Reachability and Type References

### 5.1 Type Reference Edges

When computing graph reachability, type references create edges:

```fsharp
let getTypeDefRefs (node: SemanticNode) (graph: SemanticGraph) =
    let rec getTypeNames ty =
        match ty with
        | NativeType.TApp(tycon, args) ->
            let tyconNames = if tycon.FieldCount > 0 then [tycon.Name] else []
            tyconNames @ (args |> List.collect getTypeNames)
        | NativeType.TFun(domain, range) ->
            getTypeNames domain @ getTypeNames range
        | NativeType.TTuple(elements, _) ->
            elements |> List.collect getTypeNames
        // ... other cases

    getTypeNames node.Type
    |> List.choose (fun name -> SemanticGraph.recallType name graph)
```

### 5.2 Why This Matters

If a reachable expression has type `Person` (a record type), the `Person` TypeDef must also be reachable. Without following type references:
- Hard-delete reachability would prune TypeDef nodes
- Field access lookups would fail
- Record construction/destruction would break

## 6. Pattern Matching and Constructor Types

### 6.1 Constructor Payload Types

DU constructors have function types encoding their payload:

```fsharp
// IntVal : int -> Number
// binding.Type = TFun(int, Number)

// Pair : int -> string -> PairType
// binding.Type = TFun(int, TFun(string, PairType))
```

### 6.2 Pattern Binding Type Resolution

When pattern matching a constructor, extract payload types from the constructor's function type:

```fsharp
// Match: | IntVal x -> ...
// To get type of 'x':
match tryLookupBinding "IntVal" env with
| Some binding ->
    match binding.Type with
    | NativeType.TFun(payloadType, _) ->
        // payloadType is 'int' for 'x'
```

### 6.3 Why Not Fresh Type Variables?

Creating fresh type variables for pattern bindings:
- Ignores known type information
- Creates unbound type variables that may not get resolved
- Breaks type conversions like `float x` in pattern match bodies

The principled approach uses constructor types directly.

## 7. Invariants

### 7.1 Single Representation Invariant

For any semantic concept (records, unions), use ONE type representation:
- Records: `TApp(tycon, [])` where `tycon.FieldCount > 0`
- Unions: `TApp(tycon, [])` with cases looked up via TypeDef
- Primitives: `TApp(tycon, [])` where `tycon.NTUKind` is `Some _`

### 7.2 Deferred Lookup Invariant

Type metadata (fields, cases) is NEVER embedded in type references:
- TypeConRef contains identity and layout hints
- TypeDef nodes contain full definitions
- Lookups connect references to definitions

### 7.3 Reachability Completeness Invariant

If a node is reachable and references a TypeDef (via its type), that TypeDef is reachable:
- Follows from semantic correctness
- Required for field/case access during code generation
- Enforced by `getTypeDefRefs` in reachability computation

## 8. Implementation Files

| File | Purpose |
|------|---------|
| `NativeTypes.fs` | TypeConRef, NativeType definitions |
| `SemanticGraph.fs` | tryGetRecordFields, Reachability.computeReachable |
| `Expressions/Patterns.fs` | Pattern binding type resolution |
| `MemoryWitness.fs` | Field access using tryGetRecordFields |
| `TypeMapping.fs` | NativeType → MLIRType conversion |

## Appendix A: Migration from TRecord

If existing code uses `TRecord`:

1. Replace `TRecord(tycon, fields)` with `TApp(mkRecordTypeConRef ... fieldCount, [])`
2. Replace direct field access with `tryGetRecordFields tycon.Name graph`
3. Ensure TypeDef nodes are created for all record types
4. Update pattern matching to use constructor type lookup
