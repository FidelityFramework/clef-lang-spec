---
title: "List Operations Representation"
weight: 360
category: Representation
status: normative
---

> **Status**: Normative
> **Last Updated**: 2026-01-20
> **Depends On**: [Native Type Universe § 5.3 List](native-type-universe.md#53-list)

## 1. Overview

Clef implements list operations (`List.map`, `List.filter`, `List.fold`, `List.rev`, etc.) as **Baker-decomposed algorithms** that expand to primitive list operations at compile time. Unlike Seq operations (which create wrapper structures), List operations produce eager results via recursive traversal.

**Key Insight**: List HOFs are decomposed by Baker into compositions of primitive operations (`cons`, `head`, `tail`, `isEmpty`, `empty`). The decomposition happens at compile time, producing [a PSG](program-semantic-graph.md) that Alex witnesses directly.

## 2. Relationship to Prior Chapters

List operations build on:

- **[Native Type Universe § 5.3](native-type-universe.md#53-list)** - List memory layout (cons cells)
- **[Closure Representation](closure-representation.md)** - Mapper/predicate closures are flat closures

**Architectural Model**:
```
Baker (Compile-Time)         Alex (Code Generation)
─────────────────────────    ─────────────────────────
List.map f xs                List primitives witnessed:
    ↓ decomposes to          - cons: arena alloc + store
match xs with                - head: GEP field 0, load
| [] -> []                   - tail: GEP field 1, load
| h::t -> (f h)::(map f t)   - isEmpty: null check
                             - empty: null pointer
```

## 3. Operation Classification

### 3.1 Primitive Operations (Alex Witnesses Directly)

These operations are NOT decomposed; Alex generates MLIR directly:

| Operation | Signature | MLIR Generation |
|-----------|-----------|-----------------|
| `List.empty` | `unit -> 'T list` | Returns null pointer |
| `List.isEmpty` | `'T list -> bool` | Null pointer check |
| `List.head` | `'T list -> 'T` | GEP to field 0, load |
| `List.tail` | `'T list -> 'T list` | GEP to field 1, load |
| `List.cons` | `'T -> 'T list -> 'T list` | Arena alloc, store head + tail |

### 3.2 Higher-Order Functions (Baker Decomposes)

These operations are decomposed by Baker to primitive operations:

| Category | Operations | Decomposition Pattern |
|----------|------------|----------------------|
| **Transformers** | `map`, `filter`, `collect` | `foldRight` over list, building result |
| **Reducers** | `fold`, `foldBack`, `reduce` | Iterative/recursive accumulation |
| **Predicates** | `exists`, `forall`, `contains` | Short-circuit boolean fold |
| **Selectors** | `tryPick`, `tryFind`, `minBy`, `max` | Fold with early exit or comparison |
| **Structural** | `rev`, `append`, `length` | Fold building list or counting |
| **Numeric** | `sum`, `sumBy`, `average` | Fold with arithmetic accumulation |

## 4. Decomposition Specifications

### 4.1 List.map

```fsharp
List.map : ('a -> 'b) -> 'a list -> 'b list
```

**Decomposition** (right fold for correct order):
```fsharp
let rec map f xs =
    match xs with
    | [] -> []
    | h :: t -> (f h) :: (map f t)
```

**Baker Recipe**: `foldRight (emptyList outputType) (fun h acc -> cons (f h) acc) xs`

**SSA Cost**: O(n) cons allocations, O(n) function applications

### 4.2 List.filter

```fsharp
List.filter : ('a -> bool) -> 'a list -> 'a list
```

**Decomposition**:
```fsharp
let rec filter p xs =
    match xs with
    | [] -> []
    | h :: t -> 
        if p h then h :: (filter p t)
        else filter p t
```

**Baker Recipe**: `foldRight (emptyList elemType) (fun h acc -> guardCons (p h) h acc) xs`

### 4.3 List.fold

```fsharp
List.fold : ('s -> 'a -> 's) -> 's -> 'a list -> 's
```

**Decomposition** (iterative, tail-recursive):
```fsharp
let rec fold f state xs =
    match xs with
    | [] -> state
    | h :: t -> fold f (f state h) t
```

**Baker Recipe**: `foldLeft state (fun acc h -> f acc h) xs`

**SSA Cost**: O(n) function applications, O(1) space (tail-recursive)

### 4.4 List.exists

```fsharp
List.exists : ('a -> bool) -> 'a list -> bool
```

**Decomposition** (short-circuit):
```fsharp
let rec exists p xs =
    match xs with
    | [] -> false
    | h :: t -> p h || exists p t
```

**Baker Recipe**: `boolFold false true (fun h -> p h) xs`

**Key Property**: Short-circuits on first `true`

### 4.5 List.forall

```fsharp
List.forall : ('a -> bool) -> 'a list -> bool
```

**Decomposition** (short-circuit):
```fsharp
let rec forall p xs =
    match xs with
    | [] -> true
    | h :: t -> p h && forall p t
```

**Baker Recipe**: `boolFold true false (fun h -> p h) xs`

**Key Property**: Short-circuits on first `false`

### 4.6 List.rev

```fsharp
List.rev : 'a list -> 'a list
```

**Decomposition** (left fold builds reversed list):
```fsharp
let rev xs = fold (fun acc h -> h :: acc) [] xs
```

**Baker Recipe**: `foldLeft (emptyList elemType) (fun acc h -> cons h acc) xs`

### 4.7 List.append

```fsharp
List.append : 'a list -> 'a list -> 'a list
```

**Decomposition**:
```fsharp
let append xs ys = foldBack (fun h acc -> h :: acc) xs ys
```

**Baker Recipe**: `foldRight (ret ys) (fun h acc -> cons h acc) xs`

### 4.8 List.length

```fsharp
List.length : 'a list -> int
```

**Decomposition**:
```fsharp
let length xs = fold (fun acc _ -> acc + 1) 0 xs
```

**Baker Recipe**: `foldLeft (intLit 0) (fun acc _ -> add acc 1) xs`

### 4.9 List.collect (flatMap)

```fsharp
List.collect : ('a -> 'b list) -> 'a list -> 'b list
```

**Decomposition**:
```fsharp
let collect f xs = fold (fun acc h -> append acc (f h)) [] xs
// Or more efficiently with foldRight:
let collect f xs = foldBack (fun h acc -> append (f h) acc) xs []
```

**Baker Recipe**: `foldRight (emptyList outType) (fun h acc -> append (f h) acc) xs`

### 4.10 List.sumBy

```fsharp
List.sumBy : ('a -> int) -> 'a list -> int
```

**Decomposition**:
```fsharp
let sumBy f xs = fold (fun acc x -> acc + f x) 0 xs
```

**Baker Recipe**: `foldLeft (intLit 0) (fun acc h -> add acc (f h)) xs`

### 4.11 List.contains

```fsharp
List.contains : 'a -> 'a list -> bool  // when 'a : equality
 
```

**Decomposition**:
```fsharp
let contains value xs = exists (fun x -> x = value) xs
```

**Baker Recipe**: `boolFold false true (fun h -> eq h value) xs`

### 4.12 List.tryPick

```fsharp
List.tryPick : ('a -> 'b option) -> 'a list -> 'b option
```

**Decomposition**:
```fsharp
let rec tryPick f xs =
    match xs with
    | [] -> None
    | h :: t ->
        match f h with
        | Some v -> Some v
        | None -> tryPick f t
```

**Baker Recipe**: Recursive function with option match short-circuit

### 4.13 List.minBy / List.maxBy

```fsharp
List.minBy : ('a -> 'b) -> 'a list -> 'a  // when 'b : comparison
List.maxBy : ('a -> 'b) -> 'a list -> 'a
```

**Decomposition**:
```fsharp
let minBy f xs =
    let first = head xs
    let rest = tail xs
    fold (fun currentMin x -> 
        if f x < f currentMin then x else currentMin) first rest
```

**Baker Recipe**: `foldLeft (head xs) (fun min h -> ifThenElse (lt (f h) (f min)) h min) (tail xs)`

### 4.14 List.forall2

```fsharp
List.forall2 : ('a -> 'b -> bool) -> 'a list -> 'b list -> bool
```

**Decomposition** (parallel iteration):
```fsharp
let rec forall2 f xs ys =
    match xs, ys with
    | [], [] -> true
    | h1::t1, h2::t2 -> f h1 h2 && forall2 f t1 t2
    | _ -> false  // Length mismatch
 
```

**Baker Recipe**: `foldLeft2 (fun h1 h2 -> f h1 h2) xs ys`

## 5. Memory Layout (Reference)

List cons cells follow the layout specified in [Native Type Universe § 5.3](native-type-universe.md#53-list):

```
list<'T>
┌────────────────────┬─────────────────────┐
│ head: 'T           │ tail: ptr<list<'T>> │
└────────────────────┴─────────────────────┘
     sizeof<'T>            8 bytes (pointer)
```

| Property | Value |
|----------|-------|
| **Empty list** | Null pointer (no allocation) |
| **Cons cell** | Arena-allocated struct |
| **Structural sharing** | Enabled (immutable cells) |

## 6. SSA Cost Formulas

### 6.1 Primitive Operation Costs

| Operation | SSA Operations |
|-----------|---------------|
| `empty` | 1 (null constant) |
| `isEmpty` | 2 (load ptr, icmp null) |
| `head` | 2 (GEP + load) |
| `tail` | 2 (GEP + load) |
| `cons` | 6 (alloc + 2 GEP + 2 store + return) |

### 6.2 HOF Costs

| Operation | Formula |
|-----------|---------|
| `map` | `N × (6 + C_mapper)` where N = list length |
| `filter` | `N × (2 + C_predicate + 0-6)` (conditional cons) |
| `fold` | `N × C_folder` |
| `exists/forall` | `N × C_predicate` (worst case) |
| `length` | `N × 3` (add + iteration) |
| `rev` | `N × 6` (N cons operations) |

## 7. Normative Requirements

1. **Decomposition**: List HOFs SHALL be decomposed by Baker to primitive operations
2. **Primitive Witnesses**: Alex SHALL witness `empty`, `isEmpty`, `head`, `tail`, `cons` directly
3. **Arena Allocation**: Cons cells SHALL be allocated in arenas, not GC heap
4. **Tail Recursion**: Left folds (`fold`, `rev`, `length`) SHALL compile to tail-recursive form
5. **Order Preservation**: `map` and `filter` SHALL preserve element order
6. **Short-Circuit**: `exists` and `forall` SHALL short-circuit on determining result
7. **Empty List**: Empty list SHALL be represented as null pointer (no allocation)

## 8. Relationship to Seq Operations

| Aspect | List Operations | Seq Operations |
|--------|-----------------|----------------|
| **Evaluation** | Eager (immediate) | Lazy (on-demand) |
| **Result** | New list | Wrapper sequence |
| **Memory** | Allocates all cells | Allocates wrapper only |
| **Iteration** | Complete traversal | Partial possible |
| **Fusion** | Not applicable | Wrapper composition |

**When to use List**: Pattern matching, recursive algorithms, small-to-medium collections, when full result needed.

**When to use Seq**: Large/infinite streams, when partial iteration likely, when fusion benefits outweigh overhead.

## 9. References

- [Native Type Universe § 5.3 List](native-type-universe.md#53-list) - List memory layout
- [Closure Representation](closure-representation.md) - Flat closure architecture
- [Seq Operations Representation](seq-operations-representation.md) - Comparison with lazy Seq HOFs
- Baker ListRecipes.fs - Implementation reference
