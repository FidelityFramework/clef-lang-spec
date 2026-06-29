---
title: "Option Operations Representation in Clef"
weight: 390
---

> **Status**: Normative
> **Last Updated**: 2026-01-22
> **Depends On**: [Native Type Universe § 5.1 Option](native-type-universe.md#51-option)

## 1. Overview

Clef implements option operations (`Option.map`, `Option.bind`, `Option.defaultValue`, etc.) as **Baker-decomposed pattern matches**. Options are stack-allocated tagged unions (`voption` semantics), and operations compile to simple conditional branches.

**Key Insight**: Option operations are structurally trivial: each is a single match expression with two branches (Some/None). Baker decomposes them to `isSome` checks and value extraction.

## 2. Memory Layout (Reference)

Options follow the layout specified in [Native Type Universe § 5.1](native-type-universe.md#51-option):

```
option<'T>  (voption semantics)
┌─────────────┬────────────────────┐
│ Tag (≥ i8)  │ Payload: 'T        │
└─────────────┴────────────────────┘
   ≥1 byte       sizeof<'T>        + padding
```

| Property | Value |
|----------|-------|
| **Tag values** | `None` = 0, `Some` = 1 |
| **Tag width** | Platform policy; minimum `i8` (see [DU Representation § 2.2.1](discriminated-union-representation.md#221-platform-aware-tag-width-policy)) |
| **Stack allocated** | Always (never heap) |
| **Null-freedom** | `None` is tag 0, NOT null pointer |

**Note**: Option has 2 cases, so the minimum tag is `i8` (1 byte). Platform policy MAY use larger tags for alignment efficiency on word-aligned architectures. The tag is NEVER `i1` (bit) because bits are not addressable and DU tags are case indices, not booleans.

## 3. Operation Classification

### 3.1 Primitive Operations (Alex Witnesses Directly)

| Operation | Signature | MLIR Generation |
|-----------|-----------|-----------------|
| `Option.None` | `unit -> 'T option` | Create struct with tag=0 |
| `Option.Some` | `'T -> 'T option` | Create struct with tag=1, value |
| `Option.isSome` | `'T option -> bool` | Extract tag, compare to 1 |
| `Option.isNone` | `'T option -> bool` | Extract tag, compare to 0 |
| `Option.get` | `'T option -> 'T` | Extract value field (unchecked) |

### 3.2 Higher-Order Functions (Baker Decomposes)

| Category | Operations | Description |
|----------|------------|-------------|
| **Transformers** | `map`, `map2`, `map3` | Apply function if Some |
| **Binders** | `bind`, `flatten` | Chain optional computations |
| **Defaults** | `defaultValue`, `defaultWith`, `orElse`, `orElseWith` | Provide fallback values |
| **Predicates** | `filter`, `exists`, `forall` | Conditional Some/None |
| **Conversion** | `toList`, `toArray`, `toNullable`, `ofNullable` | Type conversions |
| **Iteration** | `iter` | Side-effect if Some |

## 4. HOF Decomposition Specifications

### 4.1 Option.map

```fsharp
Option.map : ('a -> 'b) -> 'a option -> 'b option
```

**Decomposition**:
```fsharp
let map f opt =
    match opt with
    | Some x -> Some (f x)
    | None -> None
```

**Baker Recipe**:
```fsharp
recipe {
    let! isSome = isSome optId
    let! someCase = recipe {
        let! value = getValue optId elemType
        let! mapped = app1 mapperNodeId value outputType
        return! some mapped outputType
    }
    let! noneCase = none outputType
    return! ifThenElse isSome someCase noneCase optionOutputType
}
```

### 4.2 Option.bind

```fsharp
Option.bind : ('a -> 'b option) -> 'a option -> 'b option
```

**Decomposition**:
```fsharp
let bind f opt =
    match opt with
    | Some x -> f x
    | None -> None
```

**Baker Recipe**: Similar to map, but `f x` returns an option directly (no wrapping).

### 4.3 Option.defaultValue

```fsharp
Option.defaultValue : 'a -> 'a option -> 'a
```

**Decomposition**:
```fsharp
let defaultValue def opt =
    match opt with
    | Some x -> x
    | None -> def
```

**Baker Recipe**:
```fsharp
recipe {
    let! isSome = isSome optId
    let! someValue = getValue optId elemType
    return! ifThenElse isSome someValue defaultValueId elemType
}
```

### 4.4 Option.defaultWith

```fsharp
Option.defaultWith : (unit -> 'a) -> 'a option -> 'a
```

**Decomposition**:
```fsharp
let defaultWith defThunk opt =
    match opt with
    | Some x -> x
    | None -> defThunk ()
```

**Note**: The thunk is only evaluated if `opt` is `None` (lazy evaluation).

### 4.5 Option.orElse

```fsharp
Option.orElse : 'a option -> 'a option -> 'a option
```

**Decomposition**:
```fsharp
let orElse ifNone opt =
    match opt with
    | Some _ -> opt
    | None -> ifNone
```

### 4.6 Option.orElseWith

```fsharp
Option.orElseWith : (unit -> 'a option) -> 'a option -> 'a option
```

**Decomposition**:
```fsharp
let orElseWith ifNoneThunk opt =
    match opt with
    | Some _ -> opt
    | None -> ifNoneThunk ()
```

### 4.7 Option.filter

```fsharp
Option.filter : ('a -> bool) -> 'a option -> 'a option
```

**Decomposition**:
```fsharp
let filter predicate opt =
    match opt with
    | Some x -> if predicate x then Some x else None
    | None -> None
```

**Baker Recipe**:
```fsharp
recipe {
    let! isSome = isSome optId
    let! filteredSome = recipe {
        let! value = getValue optId elemType
        let! passes = app1 predicateId value boolType
        let! keepSome = some value elemType
        let! discardNone = none elemType
        return! ifThenElse passes keepSome discardNone optionType
    }
    let! noneCase = none elemType
    return! ifThenElse isSome filteredSome noneCase optionType
}
```

### 4.8 Option.exists

```fsharp
Option.exists : ('a -> bool) -> 'a option -> bool
```

**Decomposition**:
```fsharp
let exists predicate opt =
    match opt with
    | Some x -> predicate x
    | None -> false
```

### 4.9 Option.forall

```fsharp
Option.forall : ('a -> bool) -> 'a option -> bool
```

**Decomposition**:
```fsharp
let forall predicate opt =
    match opt with
    | Some x -> predicate x
    | None -> true  // Vacuously true
```

### 4.10 Option.iter

```fsharp
Option.iter : ('a -> unit) -> 'a option -> unit
```

**Decomposition**:
```fsharp
let iter action opt =
    match opt with
    | Some x -> action x
    | None -> ()
```

### 4.11 Option.toList

```fsharp
Option.toList : 'a option -> 'a list
```

**Decomposition**:
```fsharp
let toList opt =
    match opt with
    | Some x -> [x]
    | None -> []
```

### 4.12 Option.toArray

```fsharp
Option.toArray : 'a option -> 'a array
```

**Decomposition**:
```fsharp
let toArray opt =
    match opt with
    | Some x -> [| x |]
    | None -> [| |]
```

### 4.13 Option.flatten

```fsharp
Option.flatten : 'a option option -> 'a option
```

**Decomposition**:
```fsharp
let flatten opt =
    match opt with
    | Some inner -> inner
    | None -> None
```

### 4.14 Option.map2

```fsharp
Option.map2 : ('a -> 'b -> 'c) -> 'a option -> 'b option -> 'c option
```

**Decomposition**:
```fsharp
let map2 f opt1 opt2 =
    match opt1, opt2 with
    | Some x, Some y -> Some (f x y)
    | _ -> None
```

### 4.15 Option.map3

```fsharp
Option.map3 : ('a -> 'b -> 'c -> 'd) -> 'a option -> 'b option -> 'c option -> 'd option
```

**Decomposition**:
```fsharp
let map3 f opt1 opt2 opt3 =
    match opt1, opt2, opt3 with
    | Some x, Some y, Some z -> Some (f x y z)
    | _ -> None
```

## 5. SSA Cost Formulas

Option operations are extremely lightweight:

| Operation | SSA Operations |
|-----------|---------------|
| `None` | 2 (undef + insertvalue tag=0) |
| `Some v` | 3 (undef + insertvalue tag=1 + insertvalue value) |
| `isSome` | 2 (extractvalue + icmp) |
| `isNone` | 2 (extractvalue + icmp) |
| `getValue` | 1 (extractvalue) |
| `map` | 6 + C_mapper (isSome + branch + getValue + apply + wrap) |
| `bind` | 5 + C_binder (isSome + branch + getValue + apply) |
| `defaultValue` | 4 (isSome + branch + getValue or default) |
| `filter` | 8 + C_predicate (nested conditionals) |

## 6. Normative Requirements

1. **Stack Allocation**: Option values SHALL always be stack-allocated (voption semantics)
2. **Tag Encoding**: `None` = 0, `Some` = 1; tag width per platform policy (minimum `i8`)
3. **No Null**: `None` is NOT represented as null pointer; it is a valid struct with tag=0
4. **Decomposition**: Option HOFs SHALL be decomposed by Baker to primitive operations
5. **Lazy Defaults**: `defaultWith` and `orElseWith` SHALL only evaluate thunk when needed
6. **Vacuous Truth**: `Option.forall` on `None` SHALL return `true`
7. **Tag Is Not Boolean**: Tag MUST be at least `i8`, NEVER `i1`, because tags are case indices, not truth values

## 7. Relationship to Result

Option and Result are both sum types with similar operation patterns:

| Option | Result | Semantic Difference |
|--------|--------|---------------------|
| `Some x` | `Ok x` | Success case |
| `None` | `Error e` | Failure case (Option has no error info) |
| `Option.map` | `Result.map` | Transform success |
| `Option.bind` | `Result.bind` | Chain computations |
| `Option.defaultValue` | `Result.defaultValue` | Fallback on failure |

**When to use Option**: Absence of value (lookup miss, empty input).

**When to use Result**: Failure with explanation (validation error, IO failure).

## 8. References

- [Native Type Universe § 5.1 Option](native-type-universe.md#51-option) - Option memory layout
- Baker OptionRecipes.fs - Implementation reference
