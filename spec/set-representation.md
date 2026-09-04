---
title: "Set Representation"
weight: 380
category: Representation
status: normative
---

> **Status**: Normative
> **Last Updated**: 2026-09-04
> **Depends On**: [Native Type Universe § 5.5 Set](native-type-universe.md#55-set), [Closure Representation §3.3](closure-representation.md#33-escape-analysis), [Access Kinds](access-kinds.md)

## 1. Overview

Clef implements `Set<'T>` as a persistent immutable AVL tree. Set is structurally similar to `Map<'T, unit>` but with an optimized layout that omits the value field.

**Key Insight**: Sets share the same AVL balancing algorithms as Maps, but store only values (no key-value pairs), making them more memory-efficient for membership testing scenarios.

Nodes are flat aggregates placed in an arena; links between nodes are arena-relative `index` values; the empty set is index 0, the copy at offset 0 of every hosting arena of one static sentinel image per element type (§2.2). No representation state is null and no algorithm branches on absence: interior Clef has no null ([Types and Type Constraints](types-and-type-constraints.md)).

## 2. Memory Layout

### 2.1 Set Node Structure

A set node is a flat aggregate with a settled layout. Its storage is a `memref` view into the arena buffer the tree resides in; the arena is chosen by the lifetime lattice of [Closure Representation §3.3](closure-representation.md#33-escape-analysis) (a region when region-bounded, static storage when program-lifetime; never a GC heap). Each link is an `index` into that arena buffer, an arena-relative offset. Which arena is a saturated annotation on the value, never runtime data: every slot load or store a recipe emits is issued against that arena's `memref`, index `0` is valid in every arena, and a recipe is instantiated for the arena of the tree it operates on, so the extent *E* of §2.4 is a literal at each site. A link is never an address and never a null: interior Clef has no null and no raw pointer ([Types and Type Constraints](types-and-type-constraints.md)).

```
Set<'T> node
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ value: 'T                 (sizeof<'T> bytes)  - node value                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ left: index (arena link)  (1 platform word)   - left subtree (values < this value)  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ right: index (arena link) (1 platform word)   - right subtree (values > this value) │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ height: i8                (1 byte)            - subtree height for AVL balancing    │
└─────────────────────────────────────────────────────────────────────────────────────┘

Field Indices:
  [0] = value
  [1] = left
  [2] = right
  [3] = height
```

> **Platform word**: `index` is one platform word, 8 bytes on x86-64 and 4 bytes on thumbv8m/M33.

Every link word carries the range obligation VC-LINK of §2.4: its value is within the extent of the arena, and index 0 is the sentinel (§2.2). A leaf is a node whose `left` and `right` both index the sentinel (§2.2); there is no other leaf form.

### 2.2 Empty Set

The empty set is the zero-slot environment node of [Closure Representation §2.1](closure-representation.md) placed statically. Its **sentinel image** for an element type `'T` is one program-lifetime, immutable node with the layout of §2.1:

```
sentinel image for Set<'T>   (one per element type; program lifetime; immutable)
  height = 0
  left   = 0        (its own index)
  right  = 0        (its own index)
  value  = zero-initialised static data, never read
```

**Residence (VC-RES)**: the image resides in the platform's declared immutable program-lifetime space, cited by name from the platform description through a `Resides` edge ([Program Hypergraph §6](program-hypergraph.md)): rodata on an ELF target, flash on an MCU, constant memory on a GPU, initialised BRAM on an FPGA. It is the program-lifetime point of the lifetime lattice ([Closure Representation §3.3](closure-representation.md)) and is never allocated.

**Offset 0**: every arena that hosts `Set<'T>` nodes carries the sentinel at offset 0, initialised from the image when the arena is created (a `memref.copy` of one node; a static initialiser for a static-backed arena). A link is therefore always a plain arena-relative offset, `0` is the sentinel, and no read ever selects between buffers: there is no absence branch and no redirect at the witness. `Set.empty<'T>` is the index literal `0`; it allocates nothing. The copy occupies the arena's first `sizeof(node)` bytes, the arena's **floor**: a hosting arena's bump position begins at the floor and never returns below it (a reset of a hosting arena returns the position to the floor, not to 0), so every index `node` returns is ≥ `sizeof(node)` and the sentinel slot is never a placement target. The image is all-zero, so one slot at offset 0 serves every collection type the arena hosts; the slot's size is the arena's **floor**, below which `alloc` never returns and to which `reset` returns ([Memory Regions](memory-regions.md), Floor).

**Immutability (VC-RO)**: the sentinel slot `[0, sizeof(node))` of the arena is `ReadOnly` ([Access Kinds](access-kinds.md)); no store site targets it, and a store through it is the compile-time diagnostic CCS8020. `Set.add` on the sentinel places a fresh arena node whose links are 0; the sentinel is never mutated in place.

**isEmpty**: `Set.isEmpty` is a literal comparison: one `memref.load` of `height` and one compare with 0, equivalently `index = 0`. It is never a null check, and two empty sets are equal by that test, not by identity across arenas.

**No absence branch**: Every link is a valid node. No algorithm of §4 or §5 tests a link for absence; `left` and `right` are read unconditionally, and recursion terminates at the sentinel. The sentinel's `value` slot is never read: VC-GUARD (§2.4) requires the test `height ≠ 0` to dominate every read of `value` on the saturated graph.

### 2.3 Comparison with Map Layout

| Field | Map<'K,'V> | Set<'T> |
|-------|------------|---------|
| key/value | `key: 'K` + `value: 'V` | `value: 'T` |
| left | `index` (arena link) | `index` (arena link) |
| right | `index` (arena link) | `index` (arena link) |
| height | `i8` | `i8` |
| empty | one sentinel image per instantiation `Map<'K, 'V>`, at offset 0 of each hosting arena; height 0, links 0 | one sentinel image per element type `'T`, at offset 0 of each hosting arena; height 0, links 0 |
| **Total overhead** | 2 words + 1 byte + sizeof(K) + sizeof(V) | 2 words + 1 byte + sizeof(T) |

Set saves `sizeof(V)` per node compared to `Map<T, unit>`. On x86-64 the fixed overhead is 17 bytes per node; on thumbv8m/M33 it is 9. The sentinel, the link form, and the obligations of §2.4 are the same for both; only the slot set differs.

### 2.4 Layout Obligations

Every obligation the layout generates is quantifier-free, in the discharge regime of [Closure Representation §11](closure-representation.md#11-proof-extraction-at-closure-sites): at saturation, over the graph's literals, before witnessing. For a tree in an arena of extent *E*:

| VC | Obligation | Fragment | Discharge |
|---|---|---|---|
| VC-LINK | every link store site writes a value *i* with 0 ≤ *i* < *E*; *i* = 0 is the sentinel | QF_LIA | one conjunct per link store site in the recipe bodies, never one per link word: the stored operand is the literal `0`; or the index `node` returned, which is ≥ `sizeof(node)` by the floor of §2.2 and < *E* by the bump's capacity fact `Position + sizeof(node) ≤ Capacity` ([Memory Regions](memory-regions.md)), *E* being the hosting arena's capacity literal; or a value loaded from a link slot, which inherits the obligation discharged at the site that stored it. A link load carries no check of its own |
| VC-GUARD | every read of `value` of a node value *n* is guarded: it lies in the arm selected by `height n ≠ 0` of a match on the same SSA value *n* (§5), or under a balance-factor test that entails `height n ≥ 2` (the rotations of §4, [Map Representation §4.3](map-representation.md#43-rebalance-algorithm)) | none; graph for §5, QF_LIA over two loaded heights for §4 | per read site within one recipe body; a recursive call re-establishes the guard because the callee matches its own parameter, and no user function receives a node value |
| VC-RES | the sentinel image `Resides` in the platform's declared immutable program-lifetime space | none; declaration | cited by name from the platform description |
| VC-RO | no store site after the initialising copy targets the sentinel slot `[0, sizeof(node))` of the arena | none; structural | by construction, per store site: every store into a node slot is emitted by `node` at the index the bump returns, and a hosting arena's bump position begins at the floor `sizeof(node)` (§2.2) and never returns below it, so every store target is ≥ `sizeof(node)` without reasoning about any runtime index value; CCS8020 diagnoses the one remaining store form, a store through a `ReadOnly` view of the image |

The AVL balance invariant is a schema lemma, proven once per recipe shape (the `add` recipe of §5.1 and the rotation and rebalance recipes of §4; `remove` has the same shape), never a per-program fixpoint. A program that instantiates a recipe inherits the lemma and discharges only the literal obligations above.

## 3. Operation Classification

### 3.1 Primitive Operations (Alex Witnesses Directly)

| Operation | Signature | Description |
|-----------|-----------|-------------|
| `Set.empty` | `unit -> Set<'T>` | The sentinel's `index` (§2.2); no allocation |
| `Set.isEmpty` | `Set<'T> -> bool` | `memref.load` of `height`, compare with 0 |
| `Set.node` | Internal | Place a node in the arena (bump allocate, store fields); yields its `index` |
| `Set.setLeft` / `Set.setRight` | Internal | Path copy, never a store into an existing node: `setLeft n l = node (value n) l (right n) (1 + max (height l) (height (right n)))`, symmetrically `setRight`; the field stores inside `node` are the only store sites this chapter emits |
| `Set.value` | Internal | `memref.load` of field 0 (under VC-GUARD, §2.4) |
| `Set.left` | Internal | `memref.load` of field 1: an `index` |
| `Set.right` | Internal | `memref.load` of field 2: an `index` |
| `Set.height` | Internal | `memref.load` of field 3 |

### 3.2 Higher-Order Functions (Baker Decomposes)

| Category | Operations | Description |
|----------|------------|-------------|
| **Membership** | `contains`, `isSubset`, `isSuperset` | Tree traversal with comparison |
| **Modification** | `add`, `remove` | Tree traversal + rebalance |
| **Set Operations** | `union`, `intersect`, `difference` | Merge algorithms |
| **Conversion** | `toList`, `toSeq`, `toArray` | In-order traversal |
| **Iteration** | `fold`, `iter`, `forall`, `exists` | Tree traversal |
| **Construction** | `ofList`, `ofSeq`, `ofArray`, `singleton` | Repeated add operations |

## 4. AVL Tree Algorithms

Set uses identical AVL balancing algorithms to Map. See [Map Representation § 4](map-representation.md#4-avl-tree-algorithms) for:

- Balance factor calculation
- Left and right rotations
- Left-right and right-left double rotations
- Rebalance algorithm

The sentinel has height 0, so `balanceFactor` and the rotations read `left` and `right` unconditionally: a leaf's links index the sentinel, and its balance factor is computed from two height-0 children. No rotation or rebalance step tests a link for absence; every link is a valid node (§2.2).

## 5. HOF Decomposition Specifications

In every decomposition below, `isEmpty` is the height test of §2.2 and `empty` is the sentinel's index. The branch `isEmpty` guards is the recursion's base case at the sentinel, not an absence test: `left` and `right` are always valid nodes and are read unconditionally, and `setValue` is read only on the arm where `height ≠ 0`, which is the dominance VC-GUARD (§2.4) requires. `setLeft s l` and `setRight s r` are path copies, `node (setValue s) l (right s) h` and `node (setValue s) (left s) r h` with `h` recomputed from the children's heights: each places a fresh node and stores into no existing one, so no store site targets the sentinel slot (VC-RO).

### 5.1 Set.add

```fsharp
Set.add : 'T -> Set<'T> -> Set<'T>
```

**Decomposition**:
```fsharp
let rec add value set =
    if isEmpty set then
        node value empty empty 1  // at the sentinel: a fresh arena node, both links indexing the sentinel
    else
        let cmp = compare value (setValue set)
        if cmp < 0 then
            rebalance (setLeft set (add value (left set)))
        elif cmp > 0 then
            rebalance (setRight set (add value (right set)))
        else
            set  // Value already exists, return unchanged
 
```

**Complexity**: O(log n)

### 5.2 Set.contains

```fsharp
Set.contains : 'T -> Set<'T> -> bool
```

**Decomposition**:
```fsharp
let rec contains value set =
    if isEmpty set then false
    else
        let cmp = compare value (setValue set)
        if cmp < 0 then contains value (left set)
        elif cmp > 0 then contains value (right set)
        else true
```

**Complexity**: O(log n)

### 5.3 Set.toList

```fsharp
Set.toList : Set<'T> -> 'T list
```

**Decomposition** (in-order traversal):
```fsharp
let rec toList set =
    if isEmpty set then []
    else
        let leftList = toList (left set)
        let current = setValue set
        let rightList = toList (right set)
        append leftList (current :: rightList)
```

**Note**: Returns elements in sorted order.

### 5.4 Set.fold

```fsharp
Set.fold : ('S -> 'T -> 'S) -> 'S -> Set<'T> -> 'S
```

**Decomposition**:
```fsharp
let rec fold folder state set =
    if isEmpty set then state
    else
        let state' = fold folder state (left set)
        let state'' = folder state' (setValue set)
        fold folder state'' (right set)
```

### 5.5 Set.forall

```fsharp
Set.forall : ('T -> bool) -> Set<'T> -> bool
```

**Decomposition** (short-circuit):
```fsharp
let rec forall predicate set =
    if isEmpty set then true
    else
        predicate (setValue set) &&
        forall predicate (left set) &&
        forall predicate (right set)
```

### 5.6 Set.exists

```fsharp
Set.exists : ('T -> bool) -> Set<'T> -> bool
```

**Decomposition** (short-circuit):
```fsharp
let rec exists predicate set =
    if isEmpty set then false
    else
        predicate (setValue set) ||
        exists predicate (left set) ||
        exists predicate (right set)
```

### 5.7 Set.union

```fsharp
Set.union : Set<'T> -> Set<'T> -> Set<'T>
```

**Decomposition** (fold-based):
```fsharp
let union set1 set2 = fold (fun acc x -> add x acc) set1 set2
```

**Complexity**: O(m log(n+m)) where m is smaller set size

### 5.8 Set.intersect

```fsharp
Set.intersect : Set<'T> -> Set<'T> -> Set<'T>
```

**Decomposition**:
```fsharp
let intersect set1 set2 =
    fold (fun acc x -> if contains x set2 then add x acc else acc) empty set1
```

### 5.9 Set.difference

```fsharp
Set.difference : Set<'T> -> Set<'T> -> Set<'T>
```

**Decomposition**:
```fsharp
let difference set1 set2 =
    fold (fun acc x -> if not (contains x set2) then add x acc else acc) empty set1
```

### 5.10 Set.isEmpty

```fsharp
Set.isEmpty : Set<'T> -> bool
```

**Implementation**: One `memref.load` of `height` and one compare with the literal 0 (primitive, not decomposed; §2.2). It is a literal comparison, never a null check.

### 5.11 Set.remove

```fsharp
Set.remove : 'T -> Set<'T> -> Set<'T>
```

**Decomposition**:
```fsharp
let rec remove value set =
    if isEmpty set then set                                    // value absent: the sentinel's index, unchanged
    else
        let cmp = compare value (setValue set)
        if cmp < 0 then rebalance (setLeft set (remove value (left set)))
        elif cmp > 0 then rebalance (setRight set (remove value (right set)))
        elif isEmpty (left set) then right set                 // at most one child: the other link's index; 0 when set was a leaf
        elif isEmpty (right set) then left set
        else
            let (v, right') = spliceMin (right set)            // in-order successor of set
            rebalance (node v (left set) right' (height set))

// spliceMin t: the least value of t, and t without that node.
// Witnessed only under the test isEmpty t = false on the same index (the site above and the
// recursive site below), which is the dominance VC-GUARD (§2.4) requires for setValue t.
and spliceMin t =
    if isEmpty (left t) then (setValue t, right t)
    else
        let (v, left') = spliceMin (left t)
        (v, rebalance (setLeft t left'))
```

**Complexity**: O(log n)

Every result of `remove` is an index: `0` when the last element is removed, a child's index when the removed node had at most one child, a fresh node otherwise. No case yields an absent link, none reads the sentinel's `value`, and the sentinel is never written.

## 6. Normative Requirements

1. **AVL Property**: All Set operations SHALL maintain AVL balance
2. **Ordering**: Set iteration SHALL yield elements in comparison order
3. **Uniqueness**: Sets SHALL contain no duplicate elements (add of existing element is no-op)
4. **Immutability**: All modification operations SHALL return new sets without mutating originals
5. **Structural Sharing**: Unchanged subtrees SHALL be shared between versions. Sharing is index aliasing within one arena: a version derived from a non-empty set resides in that set's arena; a tree grown from `Set.empty` is placed in the arena the lifetime lattice selects at its first `add`, where index 0 is already the sentinel (a version derived only from `Set.empty` resides in the arena the lifetime lattice selects for it), and the lifetime class of that arena SHALL be the join, over the graph's derivation edges at saturation, of the classes of every version derived from it; a join with no home on the target SHALL be the lifetime error of [Discriminated Union Representation §8.2](discriminated-union-representation.md), never a cross-arena link
6. **Empty Set**: `Set.empty<'T>` SHALL be the index literal `0`: the sentinel at offset 0 of the hosting arena (§2.2), initialised at arena creation from the one program-lifetime, immutable sentinel image per element type, with height 0 and both links `0`. It SHALL NOT be a null value, an absent link, or a distinguished address
7. **Sentinel Residence**: The sentinel image SHALL reside in the platform's declared immutable program-lifetime space, cited by name from the platform description as a `Resides` edge (VC-RES, §2.4), and SHALL NOT be allocated; every arena that hosts `Set<'T>` nodes SHALL carry a copy of the image at offset 0, initialised when the arena is created (§2.2)
8. **Sentinel Immutability**: The sentinel SHALL reside ReadOnly; a store through it SHALL be diagnosed at compile time (CCS8020, [Access Kinds](access-kinds.md)). `Set.add` on the sentinel SHALL place a fresh arena node
9. **Links**: Every `left` and `right` word SHALL be an `index` into the arena the tree resides in and SHALL satisfy VC-LINK (§2.4). No algorithm SHALL test a link for absence
10. **Guarded Reads**: Every read of `value` SHALL be dominated by the test `height ≠ 0` (VC-GUARD, §2.4); the sentinel's `value` slot SHALL NOT be read
11. **Comparison**: Element comparison SHALL use `compare` function from `'T : comparison` constraint
12. **[Arena Placement](memory-regions.md)**: Set nodes SHALL be placed in the arena the lifetime lattice of [Closure Representation §3.3](closure-representation.md#33-escape-analysis) selects for the tree, not a GC heap
13. **Discharge**: The obligations of §2.4 SHALL be discharged at saturation, over the graph's literals, before witnessing

## 7. SSA Cost Formulas

| Operation | Complexity | SSA Operations |
|-----------|------------|----------------|
| `Set.add` | O(log n) | ~12 per level (compare + branch + possible rotate) |
| `Set.contains` | O(log n) | ~6 per level (compare + branch + recurse) |
| `Set.toList` | O(n) | ~10 per node |
| `Set.fold` | O(n) | ~5 per node + folder cost |
| `Set.union` | O(m log(n+m)) | add cost × smaller set size |
| `Set.isEmpty` | O(1) | 2 (`memref.load` of `height`, `arith.cmpi` with 0) |

A link read (`Set.left`, `Set.right`) is one `memref.load` of an `index`. No row counts a null check or a pointer operation: no algorithm of this chapter has an absence branch (§2.2).

## 8. References

- [Native Type Universe § 5.5 Set](native-type-universe.md#55-set) - Set memory layout
- [Map Representation](map-representation.md) - AVL algorithms (shared)
- [Closure Representation §3.3](closure-representation.md#33-escape-analysis) - Lifetime lattice; arena placement of nodes and static placement of the sentinel
- [Access Kinds](access-kinds.md) - ReadOnly residence of the sentinel; CCS8020
- [Program Hypergraph §6](program-hypergraph.md) - `Resides` edge; the platform description as declared authority
- Baker SetRecipes.fs - Implementation reference
