# FNCS: Bidirectional Type Inference for DU Case Resolution

## Status: TO BE IMPLEMENTED

## Problem Statement

FNCS doesn't currently use bidirectional type inference to disambiguate DU cases based on expected type context. Standard F# does this - when the field type is `SlotState`, it constrains `Free` to be `SlotState.Free`.

## Example

```fsharp
type SlotState = | Free | Active
type EffectState = | Free | Idle | Running | Queued

type SignalSlot = { State: SlotState; ... }

// Standard F#: Works - infers SlotState.Free from field type
// FNCS: Fails - resolves Free → EffectState.Free (last registered wins)
let slot = { State = Free; ... }
```

## Current Behavior

FNCS uses `NameResolution.compose` with left-biased choice:
```fsharp
let compose (r1: Resolver) (r2: Resolver) : Resolver =
    fun name ->
        match r1 name with
        | Some b -> Some b
        | None -> r2 name
```

DU case constructors are registered via `addUnionCaseBinding`. Later definitions shadow earlier ones for unqualified names. Resolution is purely name-based, ignoring type context.

## Expected Behavior (Standard F#)

F# uses bidirectional type inference:
1. When checking a record expression `{ Field = value }`, the expected type for `value` is known from `Field`'s type
2. This expected type propagates through expression checking
3. When resolving an identifier like `Free`, only bindings compatible with the expected type are considered
4. If multiple compatible bindings exist, the most recently opened scope wins

## Proposed Solution

### Phase 1: Expected Type Threading

Modify `checkExpr` signature to accept optional expected type:
```fsharp
let rec checkExpr
    (env: TypeEnv)
    (expr: SynExpr)
    (expectedTy: NativeType option)  // NEW
    (range: SourceRange)
    : CheckedExpr * NativeType
```

### Phase 2: Record Construction

When checking record field values, pass field type as expected type:
```fsharp
| SynExpr.Record(_, _, recordFields, _) ->
    recordFields
    |> List.map (fun ((lid, _), synValueOpt, _) ->
        let fieldTy = lookupFieldType recordType lid.IdText
        match synValueOpt with
        | Some value -> checkExpr env value (Some fieldTy) range  // Pass expected
        | None -> ...
    )
```

### Phase 3: Identifier Resolution with Type Filter

Modify `tryLookupBinding` to optionally filter by expected type:
```fsharp
let tryLookupBindingWithType
    (name: string)
    (expectedTy: NativeType option)
    (env: TypeEnv)
    : ResolvedBinding option =
    match expectedTy with
    | None -> tryLookupBinding name env
    | Some expected ->
        // Find all bindings matching name
        // Filter by type compatibility with expected
        // Return the most recently scoped compatible binding
```

### Phase 4: DU Case Type Compatibility

For DU constructors, the binding type is a function type:
- `IntVal : int -> Number` has type `TFun(int, Number)`
- `Free : SlotState` has type `SlotState` (nullary constructor)

Type compatibility check:
```fsharp
let isTypeCompatible (expected: NativeType) (actual: NativeType) : bool =
    match expected, actual with
    // Nullary constructor: actual IS the expected type
    | expected, actual when expected = actual -> true
    // Unary+ constructor: actual returns expected type
    | expected, TFun(_, returnTy) -> isTypeCompatible expected returnTy
    | _ -> false
```

## Files to Modify

1. **CheckExpressions.fs**
   - Add `expectedTy` parameter to `checkExpr`
   - Update all call sites to pass expected type where known
   - Special handling for record field expressions

2. **NameResolution.fs**
   - Add `tryLookupBindingWithType` function
   - Implement type-filtered resolution

3. **Test Coverage**
   - Add tests for DU case disambiguation
   - Test shadowing + type filter interaction
   - Test nullary vs unary constructors

## Related Resources

- F# Language Spec: Type Inference and Subtype Resolution
- FCS source: `CheckExpressions.fs` bidirectional inference patterns
- See Serena memory: `fncs_du_resolution_limitation` in Firefly project

## Priority

Medium - workaround exists (unique case names), but proper F# behavior is expected.

## Discovery

January 2026 - Fidelity.Signal library had `SlotState.Free` and `EffectState.Free` colliding.
