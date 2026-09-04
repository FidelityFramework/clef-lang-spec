---
title: "List Operations Representation"
weight: 360
category: Representation
status: normative
---

> **Status**: Normative
> **Last Updated**: 2026-09-04
> **Depends On**: [Native Type Universe § 5.3 List](native-type-universe.md#53-list), [Closure Representation §3.3](closure-representation.md#33-escape-analysis), [Program Hypergraph §6](program-hypergraph.md)

## 1. Overview

Clef implements list operations (`List.map`, `List.filter`, `List.fold`, `List.rev`, etc.) as **Baker-decomposed algorithms** that expand to primitive list operations at compile time. Unlike Seq operations (which create wrapper structures), List operations produce eager results via recursive traversal.

**Key Insight**: List HOFs are decomposed by Baker into compositions of primitive operations (`cons`, `head`, `tail`, `isEmpty`, `empty`). The decomposition happens at compile time, producing [a PSG](program-semantic-graph.md) that Alex witnesses directly.

## 2. Relationship to Prior Chapters

List operations build on:

- **[Native Type Universe § 5.3](native-type-universe.md#53-list)** - List node layout (cons cells)
- **[Discriminated Union Representation §3.3](discriminated-union-representation.md)** - Tag encoding for the two-case `Empty | Cons` node
- **[Closure Representation §3.3](closure-representation.md#33-escape-analysis)** - The lifetime lattice that places a node's arena; mapper/predicate closures are flat closures
- **[Program Hypergraph §6](program-hypergraph.md)** - The `Resides` edge that cites the sentinel image's placement from the platform description

**Architectural Model**:
```
Baker (Compile-Time)         Alex (Code Generation)
─────────────────────────    ─────────────────────────
List.map f xs                List primitives witnessed:
    ↓ decomposes to          - cons: arena node; store tag, head, tail
match xs with                - head: memref.load of the head slot
| [] -> []                   - tail: memref.load of the tail link (index)
| h::t -> (f h)::(map f t)   - isEmpty: tag = Empty (literal compare)
                             - empty: the sentinel's index (§5.2)
```

## 3. Operation Classification

### 3.1 Primitive Operations (Alex Witnesses Directly)

These operations are NOT decomposed; Alex generates MLIR directly:

| Operation | Signature | MLIR Generation |
|-----------|-----------|-----------------|
| `List.empty` | `unit -> 'T list` | Returns the sentinel's index (§5.2); no allocation |
| `List.isEmpty` | `'T list -> bool` | `memref.load` of the tag, compare with `Empty` (§5.3) |
| `List.head` | `'T list -> 'T` | `memref.load` of the head slot; witnessed only where the `tag = Cons` test on the argument already dominates the application site on the saturated graph (VC-GUARD, §5.4), otherwise diagnosed at design time. No test is synthesised and no failure arm is emitted |
| `List.tail` | `'T list -> 'T list` | `memref.load` of the tail link, an `index` (§5.1); never guarded: the sentinel's `tail` is its own index, so `tail` of the empty list is the empty list |
| `List.cons` | `'T -> 'T list -> 'T list` | Node placed in the arena hosting the tail argument (§5.5); `memref.store` tag = `Cons`, head, tail; result is the new node's index, never 0 |

No primitive tests for absence. Every link is a valid node index (VC-LINK, §5.4), and the sentinel is a valid node whose tag is `Empty`.

### 3.2 Higher-Order Functions (Baker Decomposes)

These operations are decomposed by Baker to primitive operations:

| Category | Operations | Decomposition Pattern |
|----------|------------|----------------------|
| **Transformers** | `map`, `filter`, `collect` | `foldRight` over list, building result |
| **Reducers** | `fold`, `foldBack`, `reduce` | Iterative/recursive accumulation |
| **Predicates** | `exists`, `forall`, `contains` | Short-circuit boolean fold |
| **Selectors** | `tryPick`, `tryFind`, `minBy`, `max` | Fold with early exit or comparison; `minBy`, `maxBy`, `min`, `max`, `reduce`, `last` require the `tag = Cons` test on the argument to dominate the application site (§4.13) |
| **Structural** | `rev`, `append`, `length` | Fold building list or counting |
| **Numeric** | `sum`, `sumBy`, `average` | Fold with arithmetic accumulation |

## 4. Decomposition Specifications

In every decomposition below, the `[]` arm is the `tag = Empty` test on the current node (§5.3) and the `h :: t` arm reads `head` and `tail` under that test (VC-GUARD, §5.4). No arm tests a link for absence: the sentinel is a valid node, and recursion terminates on it because its tag is `Empty`. A `[]` on the right-hand side is the sentinel's index (§5.2), not an allocation. The traversal recipes (`foldLeft`, `foldRight`, `boolFold`, `foldLeft2`) are the only bodies that read `head` or `tail`, each under its own tag test on the node it matches; the mapper, predicate, or folder they call receives element values and accumulators, never a node, so VC-GUARD (§5.4) is checked once per traversal recipe shape and inherited by every HOF built on it.

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
// xs is non-empty at the application site: the tag = Cons test on xs dominates it (VC-GUARD, §5.4)
let minBy f xs =
    fold (fun currentMin x ->
        if f x < f currentMin then x else currentMin) (head xs) (tail xs)
```

**Baker Recipe**: `foldLeft (head xs) (fun min h -> ifThenElse (lt (f h) (f min)) h min) (tail xs)`, witnessed only where the `tag = Cons` test on `xs` already dominates the application site on the saturated graph; Baker synthesises no test

**Key Property**: The read `head xs` is dominated by the `tag = Cons` test on `xs` (VC-GUARD, §5.4) only because the program supplies that test above the application site (a `match`, an `isEmpty` branch, or a literal construction that saturates the tag). The recipe synthesises no test and has no `[]` arm: a test whose other arm fails is an absence branch under another name, and it would make VC-GUARD vacuous. An application of `minBy`, `maxBy`, `min`, `max`, `reduce`, `last`, `average`, or `head` to a list whose tag is not so dominated is undischargeable and is diagnosed at design time ([Program Hypergraph §7](program-hypergraph.md), [Conformance §5](conformance.md)); it is never a runtime precondition failure.

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

## 5. Memory Layout

### 5.1 Node Layout

A list node is a flat aggregate with a settled layout. Its arena (region) is chosen by the lifetime lattice of [Closure Representation §3.3](closure-representation.md#33-escape-analysis). The node is the two-case union `Empty | Cons`; its tag width follows [Discriminated Union Representation §2.2.1](discriminated-union-representation.md) and its encoding [§3.3](discriminated-union-representation.md) (`Empty` is case 0, `Cons` is case 1). The node is the one specified in [Native Type Universe § 5.3](native-type-universe.md#53-list):

```
list<'T> node
┌─────────────────┬────────────────────┬──────────────────────────┐
│ tag: Empty|Cons │ head: 'T           │ tail: index (arena link) │
└─────────────────┴────────────────────┴──────────────────────────┘
   DU tag width        sizeof<'T>          one index word
```

The `tail` slot is an `index` into the arena buffer the list lives in (an arena-relative offset), not an address. Which arena is a saturated annotation on the value, never runtime data: every slot load or store a recipe emits is issued against that arena's `memref`, index `0` is valid in every arena, and a recipe is instantiated for the arena of the list it traverses, so the extent *E* of §5.4 is a literal at each site. Storage is an MLIR `memref` view of that buffer; a link load is one `memref.load` of an `index`. Every link word carries the range obligation VC-LINK (§5.4).

| Property | Value |
|----------|-------|
| **Empty list** | The sentinel at offset 0 of the arena (§5.2); its image is static, one per element type |
| **Cons cell** | Arena-placed node; tag = `Cons` |
| **Link** | `index` into the list's arena buffer |
| **Structural sharing** | Index aliasing within one arena (§5.5) |

### 5.2 Empty List

The empty list is the zero-slot environment node placed statically. Its **sentinel image** for an element type `'T` is one program-lifetime, immutable node with the layout of §5.1:

```
sentinel image for list<'T>   (one per element type; program lifetime; immutable)
┌─────────────────┬────────────────────┬──────────────────────────┐
│ tag: Empty      │ head: zero, unread │ tail: 0 (its own index)  │
└─────────────────┴────────────────────┴──────────────────────────┘
```

**Residence (VC-RES)**: the image resides in the platform's declared immutable program-lifetime space, cited by name from the platform description through a `Resides` edge ([Program Hypergraph §6](program-hypergraph.md)): rodata on an ELF target, flash on an MCU, constant memory on a GPU, initialised BRAM on an FPGA. It is the program-lifetime point of the lifetime lattice ([Closure Representation §3.3](closure-representation.md)) and is never allocated.

**Offset 0**: every arena that hosts `list<'T>` nodes carries the sentinel at offset 0, initialised from the image when the arena is created (a `memref.copy` of one node; a static initialiser for a static-backed arena). A link is therefore always a plain arena-relative offset, `0` is the sentinel, and no read ever selects between buffers: there is no absence branch and no redirect at the witness. `List.empty` is the index literal `0`; it allocates nothing. The copy occupies the arena's first `sizeof(node)` bytes, the arena's **floor**: a hosting arena's bump position begins at the floor and never returns below it (a reset of a hosting arena returns the position to the floor, not to 0), so every index `cons` returns is ≥ `sizeof(node)` and the sentinel slot is never a placement target. The image is all-zero, so one slot at offset 0 serves every collection type the arena hosts; the slot's size is the arena's **floor**, below which `alloc` never returns and to which `reset` returns ([Memory Regions](memory-regions.md), Floor).

**Immutability (VC-RO)**: the sentinel slot `[0, sizeof(node))` of the arena is `ReadOnly` ([Access Kinds](access-kinds.md)); no store site targets it, and a store through it is the compile-time diagnostic CCS8020. `cons` onto the sentinel produces a fresh arena node whose tail is 0; the sentinel is never mutated in place.

Two empty lists are equal by the tag test (§5.3), not by identity across arenas.

### 5.3 isEmpty

`List.isEmpty` is a literal comparison: one `memref.load` of the tag and one compare against `Empty`. Index equality with the sentinel is an equivalent test. It is never an absence check, because no link is ever absent.

### 5.4 Layout Obligations

There is no absence branch in any list algorithm. Every link is a valid node; `tail` is read unconditionally (the sentinel's `tail` is its own index, 0), `head` is read only where the `tag = Cons` test dominates, and recursion terminates at the sentinel because its tag is `Empty`. In a decomposition, the `[]` arm is the `tag = Empty` test on the current node, and the empty list as a result is the index literal 0.

Every obligation the layout generates is quantifier-free at saturation, over the graph's literals, before witnessing ([Closure Representation §11](closure-representation.md); [Program Hypergraph §6](program-hypergraph.md)). For a list arena of extent *E*:

| VC | Obligation | Fragment | Discharge |
|---|---|---|---|
| VC-LINK | every `tail` store site writes a value *i* with 0 ≤ *i* < *E*; *i* = 0 is the sentinel | QF_LIA | one conjunct per `tail` store site in the recipe bodies, never one per `tail` word: the stored operand is the literal `0`; or the index a `cons` returned, which is ≥ `sizeof(node)` by the floor of §5.2 and < *E* by the bump's capacity fact `Position + sizeof(node) ≤ Capacity` ([Memory Regions](memory-regions.md)), *E* being the hosting arena's capacity literal; or a list value passed through (`ret ys` in `append`, `t` in `filter`), which inherits the obligation discharged at the site that stored it. A `tail` load carries no check of its own |
| VC-GUARD | every read of `head` of a node value *n* lies in the `Cons` arm of a match on the tag of the same SSA value *n* | none; graph | per read site within one traversal recipe body (§4); a recursive call re-establishes the guard because the callee matches its own parameter, and the mapper, predicate, or folder receives element values, never a node |
| VC-RES | the sentinel image `Resides` in the platform's declared immutable program-lifetime space | none; declaration | cited by name from the platform description |
| VC-RO | no store site after the initialising copy targets the sentinel slot `[0, sizeof(node))` of the arena | none; structural | by construction, per store site: every store into a node slot is emitted by `cons` at the index the bump returns, and a hosting arena's bump position begins at the floor `sizeof(node)` (§5.2) and never returns below it, so every store target is ≥ `sizeof(node)` without reasoning about any runtime index value; CCS8020 diagnoses the one remaining store form, a store through a `ReadOnly` view of the image |

The list algebra the recipes of §4 rely on (the fold laws that make `map`, `filter`, `rev`, and `append` right) is a schema lemma proven once per recipe shape, never a per-program fixpoint. The layout obligations above, each stated per store site or per read site of the recipe bodies, and that lemma are the whole proof burden; neither quantifies over the nodes an arena holds or the lists a program builds. The lemma's hypothesis holds for every `'T list` value because `cons` and `empty` are its only constructors, and no layout obligation depends on the lemma.

### 5.5 Structural Sharing

Structural sharing between persistent lists is index aliasing within one arena: `cons h xs` places the new node in the arena that hosts `xs` (for `xs = []`, index 0 of the arena the lifetime lattice selects for the list) and stores the index of the first node of `xs` as the new node's tail, so both lists share every node from that index onward within that one arena; a `tail` never indexes another buffer. No node is copied, and no node is mutated (§5.2 for the sentinel; every other node is written only at construction). The placement rule at every `cons` site: `cons h xs` is placed in the arena of the first node of `xs`, or, when `xs` is the sentinel, in the arena the lifetime lattice selects for the result; `append xs ys` therefore places its copies of `xs` in the arena of `ys`. The rule constrains classification, not only placement: the lifetime class of a list's arena is the join, over the graph's derivation edges at saturation, of the classes of every list that shares its nodes; the shared nodes are placed in the covering arena, and a join with no home on the target is the lifetime error of [Discriminated Union Representation §8.2](discriminated-union-representation.md), never a cross-arena link.

## 6. SSA Cost Formulas

### 6.1 Primitive Operation Costs

| Operation | SSA Operations |
|-----------|---------------|
| `empty` | 1 (the index constant 0) |
| `isEmpty` | 2 (`memref.load` of the tag + `arith.cmpi`) |
| `head` | 1 (`memref.load` of the head slot) |
| `tail` | 1 (`memref.load` of the tail link, an `index`) |
| `cons` | 4 (arena bump + `memref.store` of tag, head, tail; the result is the bumped index) |

No row counts an absence check or an address computation: a link is an `index`, and slot access is one `memref.load` or `memref.store` at that index.

### 6.2 HOF Costs

A traversal step is `isEmpty` + `head` + `tail` = 4 operations from §6.1.

| Operation | Formula |
|-----------|---------|
| `map` | `N × (8 + C_mapper)` where N = list length (step + cons) |
| `filter` | `N × (4 + C_predicate + 0 or 4)` (conditional cons) |
| `fold` | `N × (4 + C_folder)` |
| `exists/forall` | `N × (4 + C_predicate)` (worst case) |
| `length` | `N × 4` (tag test + tail + add; head is not read) |
| `rev` | `N × 8` (step + cons) |

## 7. Normative Requirements

1. **Decomposition**: List HOFs SHALL be decomposed by Baker to primitive operations
2. **Primitive Witnesses**: Alex SHALL witness `empty`, `isEmpty`, `head`, `tail`, `cons` directly
3. **Arena Placement**: Cons cells SHALL be placed in the arena the lifetime lattice of [Closure Representation §3.3](closure-representation.md#33-escape-analysis) selects, not a GC heap
4. **Tail Recursion**: Left folds (`fold`, `rev`, `length`) SHALL compile to tail-recursive form
5. **Order Preservation**: `map` and `filter` SHALL preserve element order
6. **Short-Circuit**: `exists` and `forall` SHALL short-circuit on determining result
7. **Empty List**: The empty list SHALL be the index literal `0`: the sentinel at offset 0 of the hosting arena (§5.2), initialised at arena creation from the one program-lifetime, immutable sentinel image per element type, with tag `Empty` and tail `0`, which `Resides` in the platform's declared immutable program-lifetime space (VC-RES, §5.4). `List.empty` SHALL be the index literal `0` and SHALL NOT allocate
8. **Links**: Every `tail` word SHALL be an `index` into the list's arena buffer; every `tail` store site SHALL discharge VC-LINK at saturation and every `tail` load SHALL inherit it. No link SHALL be absent, and no operation SHALL test a link for absence
9. **Guarded Reads**: Every read of `head` SHALL be dominated by the `tag = Cons` test (VC-GUARD). The sentinel's `head` slot SHALL NOT be read; an application whose argument's tag is not so dominated SHALL be diagnosed at design time and SHALL NOT be guarded by a synthesised test with a failure arm
10. **Sentinel Immutability**: The sentinel SHALL reside `ReadOnly`; a store through it SHALL be diagnosed as CCS8020 ([Access Kinds](access-kinds.md)); `cons` onto the sentinel SHALL produce a fresh arena node
11. **Sharing**: Structural sharing between lists SHALL be index aliasing within one arena

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

- [Native Type Universe § 5.3 List](native-type-universe.md#53-list) - List node layout
- [Discriminated Union Representation](discriminated-union-representation.md) - Tag width (§2.2.1) and encoding (§3.3) for the `Empty | Cons` node
- [Closure Representation](closure-representation.md) - Flat closure architecture; the lifetime lattice (§3.3) and the quantifier-free discharge regime (§11)
- [Program Hypergraph](program-hypergraph.md) - The `Resides` edge and obligation residence (§6)
- [Access Kinds](access-kinds.md) - `ReadOnly` residence of the sentinel
- [Memory Regions](memory-regions.md) - Arena and static storage
- [Seq Operations Representation](seq-operations-representation.md) - Comparison with lazy Seq HOFs
- Baker ListRecipes.fs - Implementation reference

