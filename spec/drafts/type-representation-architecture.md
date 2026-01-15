# Type Representation Architecture

> **Status**: Normative
> **Last Updated**: 2026-01-15

## 1. Overview

This chapter specifies the internal representation of types in FNCS. The design follows the ML tradition established by OCaml, F#/FCS, and F*: **types are references to definitions, not embedded data**.

## 2. Core Principle: Reference + Deferred Lookup

**Type references are REFERENCES to type definitions. Metadata (fields, cases) lives in ONE authoritative location (TypeDef) and is accessed via lookup.**

This principle is universal in ML-family languages:

| Language | Type Reference | Lookup Pattern |
|----------|---------------|----------------|
| FCS | `TType_app(tyconRef, typeInst, nullness)` | `tyconRef.Deref` |
| F* | `Tm_fvar(fv)` + `Tm_app` | `lookup_lid` |
| OCaml | `Tconstr(path, args, abbrev)` | `Env.find_type` |
| **FNCS** | `TApp(tyconRef, args)` | `tryGetRecordFields` |

### 2.1 Why This Pattern?

1. **Single source of truth**: Field/case definitions exist in ONE place
2. **Type identity by reference**: Two types are equal iff their TypeConRefs are equal
3. **No duplication**: No risk of inconsistent copies of field lists
4. **Matches F# semantics**: Record types ARE nominal types, not structural

## 3. NativeType Definition

```fsharp
type NativeType =
    /// Polymorphic type: forall 'a. T
    | TForall of typars: TypeParam list * body: NativeType

    /// Type application: tycon<T1, T2, ...>
    /// Records, unions, primitives, and all named types use this form
    | TApp of tycon: TypeConRef * args: NativeType list

    /// Tuple type: T1 * T2 * ...
    | TTuple of elements: NativeType list * isStruct: bool

    /// Function type: T1 -> T2
    | TFun of domain: NativeType * range: NativeType

    /// Type variable: 'a
    | TVar of typar: TypeParam

    /// Unit of measure
    | TMeasure of measure: Measure

    /// Anonymous record: {| field1: T1; field2: T2 |}
    | TAnon of fields: (string * NativeType) list * isStruct: bool

    /// Discriminated union (inline case representation for pattern matching)
    | TUnion of tycon: TypeConRef * cases: UnionCase list

    /// Byref types
    | TByref of element: NativeType * kind: ByrefKind

    /// Native pointer
    | TNativePtr of element: NativeType

    /// Error recovery
    | TError of message: string
```

### 3.1 Record Types

**Records are `TApp(tyconRef, [])` where the TypeConRef identifies a record type.**

There is NO `TRecord` constructor. This matches FCS which has no `TType_record`.

To determine if a type is a record:
```fsharp
let isRecordType ty =
    match ty with
    | TApp(tycon, _) -> tycon.FieldCount > 0
    | _ -> false
```

To access fields:
```fsharp
let getRecordFields typeName graph =
    match SemanticGraph.recallType typeName graph with
    | Some nodeId ->
        match SemanticGraph.tryGetNode nodeId graph with
        | Some { Kind = SemanticKind.TypeDef(_, TypeDefKind.RecordDef fields, _) } ->
            Some fields
        | _ -> None
    | None -> None
```

### 3.2 Anonymous Records vs Named Records

| Aspect | Named Record | Anonymous Record |
|--------|-------------|------------------|
| Representation | `TApp(tyconRef, [])` | `TAnon(fields, isStruct)` |
| Identity | Nominal (by TypeConRef) | Structural (by field set) |
| Field access | Lookup via TypeDef | Inline in type |
| F# syntax | `{ Name = "x" }` | `{| Name = "x" |}` |

Anonymous records use `TAnon` because they have structural identity - two anonymous records with the same fields are the same type. Named records use `TApp` because they have nominal identity - two record types with the same fields are different types if they have different names.

### 3.3 Union Types

Union types use `TUnion` which carries inline case information. This differs from records because:
1. Pattern matching requires immediate access to case structure
2. Tag extraction needs case indices
3. Union case constructors need payload types

The TypeConRef in `TUnion` still provides nominal identity.

## 4. TypeConRef Structure

```fsharp
type TypeConRef = {
    /// Fully qualified type name
    Name: string

    /// Module path
    Module: ModulePath

    /// Type parameters
    ParamKinds: TypeParamKind list

    /// Memory layout (size, alignment)
    Layout: TypeLayout

    /// NTU kind for primitive types (None for composite types)
    NTUKind: NTUKind option

    /// Field count (> 0 indicates record type)
    FieldCount: int
}
```

### 4.1 Why FieldCount?

`FieldCount` serves as a **record indicator** without embedding field types:

1. **Record detection**: `tycon.FieldCount > 0` means "this is a record"
2. **Reachability**: Record types need their TypeDef nodes preserved
3. **Validation**: Consistency check for field count

Field TYPES are not in TypeConRef because:
- It would create circular dependency (TypeConRef contains NativeType contains TypeConRef)
- It would duplicate data that belongs in TypeDef
- It would break the single-source-of-truth principle

## 5. Type Definition Lookup

### 5.1 TypeDef Nodes

Type definitions are stored as SemanticGraph nodes:

```fsharp
type TypeDefKind =
    | RecordDef of fields: (string * NativeType) list
    | UnionDef of cases: UnionCase list
    | EnumDef of underlying: NativeType option
    | AbbrevDef of expanded: NativeType
```

### 5.2 Lookup Functions

```fsharp
/// Get record fields by type name
let tryGetRecordFields (typeName: string) (graph: SemanticGraph) : (string * NativeType) list option

/// Get union cases by type name
let tryGetUnionCases (typeName: string) (graph: SemanticGraph) : UnionCase list option
```

### 5.3 When to Use Lookup

| Operation | Uses Lookup? | Why |
|-----------|-------------|-----|
| Type equality | No | Compare TypeConRef identity |
| Type unification | No | Compare TypeConRef identity |
| Field access codegen | Yes | Need field types and offsets |
| Record construction | Yes | Need to verify all fields present |
| Pattern matching | No | TUnion has inline cases |

## 6. Reachability

### 6.1 Type Reference Edges

When a node has a record type, its TypeDef must be reachable:

```fsharp
let getTypeDefRefs (node: SemanticNode) =
    let rec collect ty =
        match ty with
        | TApp(tycon, args) when tycon.FieldCount > 0 ->
            tycon.Name :: (args |> List.collect collect)
        | TApp(_, args) ->
            args |> List.collect collect
        | TFun(d, r) -> collect d @ collect r
        | TTuple(elems, _) -> elems |> List.collect collect
        | TUnion(tycon, _) -> [tycon.Name]
        | _ -> []
    collect node.Type
```

### 6.2 Invariant

**If expression E is reachable and E has type T where T references TypeDef D, then D is reachable.**

This ensures field lookups succeed during code generation.

## 7. Unification

Record types unify by TypeConRef identity:

```fsharp
let rec unify t1 t2 =
    match t1, t2 with
    | TApp(tc1, args1), TApp(tc2, args2) ->
        if tc1.Name = tc2.Name && tc1.Module = tc2.Module then
            List.iter2 unify args1 args2
        else
            fail "Type mismatch"
    // ... other cases
```

There is no special case for records - they are just `TApp`.

## 8. Code Generation

### 8.1 Record Field Access

```fsharp
let emitFieldAccess recordExpr fieldName graph =
    match recordExpr.Type with
    | TApp(tycon, _) ->
        match tryGetRecordFields tycon.Name graph with
        | Some fields ->
            let fieldIndex = fields |> List.findIndex (fun (n, _) -> n = fieldName)
            let (_, fieldType) = fields.[fieldIndex]
            emitGEP recordExpr fieldIndex fieldType
        | None ->
            failwith $"Record type {tycon.Name} not found"
    | _ ->
        failwith "Expected record type"
```

### 8.2 Record Construction

```fsharp
let emitRecordConstruction typeName fieldExprs graph =
    match tryGetRecordFields typeName graph with
    | Some expectedFields ->
        // Verify all fields present and types match
        // Emit struct construction
    | None ->
        failwith $"Record type {typeName} not found"
```

## 9. Summary

| Concept | Representation | Metadata Access |
|---------|---------------|-----------------|
| Named record | `TApp(tyconRef, [])` | `tryGetRecordFields` |
| Anonymous record | `TAnon(fields, isStruct)` | Inline |
| Union | `TUnion(tyconRef, cases)` | Inline |
| Primitive | `TApp(tyconRef, [])` | `tyconRef.NTUKind` |
| Generic type | `TApp(tyconRef, args)` | Lookup + substitution |

The key insight: **Records are not special.** They are named types like any other, represented uniformly as `TApp`. The fact that they have fields is an implementation detail accessed via lookup, not embedded in the type representation.
