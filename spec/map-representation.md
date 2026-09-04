---
title: "Map Representation"
weight: 370
category: Representation
status: normative
---

> **Status**: Normative
> **Last Updated**: 2026-09-04
> **Depends On**: [Native Type Universe § 5.4 Map](native-type-universe.md#54-map)

## 1. Overview

Clef implements `Map<'K, 'V>` as a persistent immutable AVL tree. Map operations are decomposed by Baker into primitive tree operations, with Alex witnessing the primitives directly.

**Key Insight**: Maps are self-balancing binary search trees. All mutations return new maps with structural sharing: unchanged subtrees are shared between old and new versions.

## 2. Memory Layout

### 2.1 Map Node Structure

A map node is a flat aggregate with a settled layout. Storage is an MLIR `memref` view: the nodes of a tree are placed in one arena (region) chosen by the lifetime lattice of [Closure Representation §3.3](closure-representation.md), and a map value is the `index` of its root node in that arena's buffer. Which arena is a saturated annotation on the value, never runtime data: every slot load or store a recipe emits is issued against that arena's `memref`, `Map.empty` (index `0`) is valid in every arena, and a recipe is instantiated for the arena of the tree it operates on, so *E* is a literal at each site and no read selects between buffers.

```
Map<'K, 'V> node (flat aggregate, arena-placed)
┌─────────────────────────────────────────────────────────────────────────────────┐
│ key: 'K                    (sizeof<'K> bytes) - node key                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│ value: 'V                  (sizeof<'V> bytes) - associated value                │
├─────────────────────────────────────────────────────────────────────────────────┤
│ left: index (arena link)   (platform word) - left subtree (keys < this key)     │
├─────────────────────────────────────────────────────────────────────────────────┤
│ right: index (arena link)  (platform word) - right subtree (keys > this key)    │
├─────────────────────────────────────────────────────────────────────────────────┤
│ height: i8                 (1 byte) - subtree height for AVL balancing          │
└─────────────────────────────────────────────────────────────────────────────────┘

Field Indices:
  [0] = key
  [1] = value
  [2] = left
  [3] = right
  [4] = height
```

**Links**: `left` and `right` are arena-relative offsets into the buffer the tree lives in, not addresses. There is no null link and no raw pointer in the layout; a link load is one `memref.load` of an `index`. Every link word carries the obligation

```
VC-LINK: 0 <= i < extent(arena)        (i = 0 is the sentinel, §2.2)
```

which is quantifier-free over the graph's literals (QF_LIA) and is discharged at saturation before the node is witnessed.

### 2.2 Empty Map

The empty map is the zero-slot environment node placed statically. Its **sentinel image** for an instantiation `Map<'K, 'V>` is one program-lifetime, immutable node with the layout of §2.1:

```
sentinel image for Map<'K, 'V>   (one per instantiation; program lifetime; immutable)
  height = 0
  left   = 0        (its own index)
  right  = 0        (its own index)
  key, value = zero-initialised static data, read by no operation
```

**Residence (VC-RES)**: the image resides in the platform's declared immutable program-lifetime space, cited by name from the platform description through a `Resides` edge ([Program Hypergraph §6](program-hypergraph.md)): rodata on an ELF target, flash on an MCU, constant memory on a GPU, initialised BRAM on an FPGA. It is the program-lifetime point of the lifetime lattice ([Closure Representation §3.3](closure-representation.md)) and is never allocated.

**Offset 0**: every arena that hosts `Map<'K, 'V>` nodes carries the sentinel at offset 0, initialised from the image when the arena is created (a `memref.copy` of one node; a static initialiser for a static-backed arena). The image is `sizeof(node)` zero bytes (height 0, links 0, zero payload), as is the image of every other node type of this family (`Set<'T>`; `list<'T>`, whose `Empty` is case 0), so an arena that hosts several node types carries one zero block at offset 0 of the largest hosted node size, index 0 is the sentinel of each hosted type, and the floor is that size. A link is therefore always a plain arena-relative offset, `0` is the sentinel, and no read ever selects between buffers: there is no absence branch and no redirect at the witness. `Map.empty<'K, 'V>` is the index literal `0`; it allocates nothing. The copy occupies the arena's first `sizeof(node)` bytes, the arena's **floor**: a hosting arena's bump position begins at the floor and never returns below it (a reset of a hosting arena returns the position to the floor, not to 0), so every index `node` returns is ≥ `sizeof(node)` and the sentinel slot is never a placement target. The image is all-zero, so one slot at offset 0 serves every collection type the arena hosts; the slot's size is the arena's **floor**, below which `alloc` never returns and to which `reset` returns ([Memory Regions](memory-regions.md), Floor).

**Immutability (VC-RO)**: the sentinel slot `[0, sizeof(node))` of the arena is `ReadOnly` ([Access Kinds](access-kinds.md)); no store site targets it, and a store through it is the compile-time diagnostic CCS8020. `Map.add` on the sentinel places a fresh arena node and returns its index; the sentinel is never mutated in place.

**Property**: `Map.isEmpty` is the literal comparison `height = 0` (one `memref.load`, one compare), equivalently `index = 0`. It is never a null check, and two empty maps are equal by that test, not by identity across arenas.

### 2.3 Layout Obligations

Every obligation the layout generates is quantifier-free, in the discharge regime of [Closure Representation §11](closure-representation.md): at saturation, over the graph's literals, before witnessing. For a tree in an arena of extent *E*:

| VC | Obligation | Fragment | Discharge |
|---|---|---|---|
| VC-LINK | every link store site writes a value *i* with 0 ≤ *i* < *E*; *i* = 0 is the sentinel | QF_LIA | one conjunct per link store site in the recipe bodies, never one per link word: the stored operand is the literal `0`; or the index `node` returned, which is ≥ `sizeof(node)` by the floor of §2.2 and < *E* by the bump's capacity fact `Position + sizeof(node) ≤ Capacity` ([Memory Regions](memory-regions.md)), *E* being the hosting arena's capacity literal; or a value loaded from a link slot, which inherits the obligation discharged at the site that stored it. A link load carries no check of its own |
| VC-GUARD | every read of `key` or `value` of a node value *n* is guarded: it lies in the arm selected by `height n ≠ 0` of a match on the same SSA value *n* (§5), or under a balance-factor test that entails `height n ≥ 2` (§4.3) | none; graph for §5, QF_LIA over two loaded heights for §4 | per read site within one recipe body; a recursive call re-establishes the guard because the callee matches its own parameter, and no user function receives a node value |
| VC-RES | the sentinel image `Resides` in the platform's declared immutable program-lifetime space | none; declaration | cited by name from the platform description |
| VC-RO | no store site after the initialising copy targets the sentinel slot `[0, sizeof(node))` of the arena | none; structural | by construction, per store site: every store into a node slot is emitted by `node` at the index the bump returns, and a hosting arena's bump position begins at the floor `sizeof(node)` (§2.2) and never returns below it, so every store target is ≥ `sizeof(node)` without reasoning about any runtime index value; CCS8020 diagnoses the one remaining store form, a store through a `ReadOnly` view of the image |

Each decomposition in §5 reads `key` and `value` only under `isEmpty map = false`, which discharges VC-GUARD by shape. The AVL balance invariant (§4.1) is a schema lemma proven once per recipe shape (`add`, `remove`, and the four rotations), never a per-program fixpoint: each lemma assumes balanced inputs and concludes a balanced output, and its hypothesis holds for every `Map<'K, 'V>` value because `node` is internal, so every value is `empty` or the result of a recipe. A program that instantiates a recipe inherits the lemma and discharges only the per-site obligations above; none of those obligations depends on the lemma, so no layout obligation waits on an ordering or balance fact.

## 3. Operation Classification

### 3.1 Primitive Operations (Alex Witnesses Directly)

| Operation | Signature | Description |
|-----------|-----------|-------------|
| `Map.empty` | `unit -> Map<'K, 'V>` | Returns the sentinel node's index (no allocation) |
| `Map.isEmpty` | `Map<'K, 'V> -> bool` | Literal comparison `height = 0` (one `memref.load`, one `arith.cmpi`) |
| `Map.node` | Internal | Place a fresh node in the arena (bump + store fields); returns its index |
| `Map.setLeft` / `Map.setRight` | Internal | Path copy, never a store into an existing node: `setLeft n l = node (key n) (value n) l (right n) (1 + max (height l) (height (right n)))`, symmetrically `setRight`; the field stores inside `node` are the only store sites this chapter emits |
| `Map.key` | Internal | `memref.load` field 0 (VC-GUARD) |
| `Map.value` | Internal | `memref.load` field 1 (VC-GUARD) |
| `Map.left` | Internal | `memref.load` field 2, an `index` (VC-LINK) |
| `Map.right` | Internal | `memref.load` field 3, an `index` (VC-LINK) |
| `Map.height` | Internal | `memref.load` field 4 |

### 3.2 Higher-Order Functions (Baker Decomposes)

| Category | Operations | Description |
|----------|------------|-------------|
| **Lookup** | `tryFind`, `containsKey`, `find` | Tree traversal with comparison |
| **Modification** | `add`, `remove` | Tree traversal + rebalance |
| **Conversion** | `toList`, `toSeq`, `keys`, `values` | In-order traversal |
| **Iteration** | `fold`, `iter`, `forall`, `exists` | Tree traversal with accumulator |
| **Construction** | `ofList`, `ofSeq`, `ofArray` | Repeated add operations |

## 4. AVL Tree Algorithms

### 4.1 Balance Factor

The balance factor of a node is: `height(right) - height(left)`, where `height` is one `memref.load` of the `height` slot of the linked node. The sentinel node of §2.2 has `height = 0`, so an empty subtree contributes 0 without any test: there is no absent subtree, only the sentinel.

| Balance Factor | State |
|----------------|-------|
| -1, 0, +1 | Balanced (no rotation needed) |
| -2 | Left-heavy (right rotation needed) |
| +2 | Right-heavy (left rotation needed) |

### 4.2 Rotations

**Right Rotation** (for left-heavy trees):
```
      y                x
     / \              / \
    x   C    -->     A   y
   / \                  / \
  A   B                B   C
```

**Left Rotation** (for right-heavy trees):
```
    x                  y
   / \                / \
  A   y      -->     x   C
     / \            / \
    B   C          A   B
```

**Left-Right Rotation**: Left rotate left child, then right rotate root.

**Right-Left Rotation**: Right rotate right child, then left rotate root.

### 4.3 Rebalance Algorithm

```fsharp
let rebalance node =
    let bf = balanceFactor node
    if bf < -1 then
        // Left-heavy
        if balanceFactor (left node) > 0 then
            // Left-Right case
            rightRotate (setLeft node (leftRotate (left node)))
        else
            // Left-Left case
            rightRotate node
    elif bf > 1 then
        // Right-heavy
        if balanceFactor (right node) < 0 then
            // Right-Left case
            leftRotate (setRight node (rightRotate (right node)))
        else
            // Right-Right case
            leftRotate node
    else
        node  // Already balanced
 
```

`rebalance` reads `left node` and `right node` unconditionally. Both always index a valid node, and `balanceFactor` of the sentinel node is 0, so no case tests a link for absence. A rotation reads `key` and `value` of a child *x* of the rotated node *y* with no match on *x*, so its VC-GUARD (§2.3) discharges in QF_LIA over the two loaded heights rather than by shape: `rightRotate y` runs only under `height (right y) − height x < −1`, which with non-negative heights entails `height x ≥ 2` and hence *x* ≠ 0, and symmetrically for `leftRotate`. Heights are non-negative and below 127 at every store site (`node` stores the literal `1`, a copied height, or `1 + max` of two loaded heights, and the AVL height bound over at most *E* / `sizeof(node)` nodes keeps that sum in range), so the entailment is over literals; the same store-site fact makes `height = 0` and `index = 0` equivalent (§2.2). The AVL balance invariant is a schema lemma proven once for this recipe shape (§2.3); no layout obligation depends on it.

## 5. HOF Decomposition Specifications

In every decomposition below, `isEmpty map` is the literal test `height map = 0`: the recursion terminates at the sentinel node of §2.2. `left map` and `right map` are read unconditionally and always yield the index of a valid node; no decomposition has an absence branch, and `empty` denotes the sentinel's index. `setLeft m l` and `setRight m r` are path copies, `node (mapKey m) (mapValue m) l (right m) h` and `node (mapKey m) (mapValue m) (left m) r h` with `h` recomputed from the children's heights: each places a fresh node and stores into no existing one, so no store site targets the sentinel slot (VC-RO).

### 5.1 Map.add

```fsharp
Map.add : 'K -> 'V -> Map<'K, 'V> -> Map<'K, 'V>
```

**Decomposition**:
```fsharp
let rec add key value map =
    if isEmpty map then
        node key value empty empty 1
    else
        let cmp = compare key (mapKey map)
        if cmp < 0 then
            rebalance (setLeft map (add key value (left map)))
        elif cmp > 0 then
            rebalance (setRight map (add key value (right map)))
        else
            // Key exists: create node with new value
            node key value (left map) (right map) (height map)
```

**Complexity**: O(log n) comparisons, O(log n) allocations (path copying)

`empty` is the sentinel's index: a fresh leaf has `height = 1` and both links index the sentinel. When `map` is the sentinel, `node` places the new leaf in the arena and the sentinel is not written (§2.2).

### 5.2 Map.tryFind

```fsharp
Map.tryFind : 'K -> Map<'K, 'V> -> 'V option
```

**Decomposition**:
```fsharp
let rec tryFind key map =
    if isEmpty map then None
    else
        let cmp = compare key (mapKey map)
        if cmp < 0 then tryFind key (left map)
        elif cmp > 0 then tryFind key (right map)
        else Some (mapValue map)
```

**Complexity**: O(log n) comparisons, O(1) allocations

`mapKey map` and `mapValue map` are read only under `isEmpty map = false`, so `height ≠ 0` dominates both reads and VC-GUARD (§2.3) is discharged by the shape of the decomposition. The same holds for `containsKey` and `find`, each `tryFind` under a match on the resulting `option`. In `find` the `None` arm is the operation's precondition failure: a match on an `option` value (a DU tag test), never a test of any link and never a read of the sentinel; it is not a design-time obligation, since key presence is not a graph literal.

### 5.3 Map.containsKey

```fsharp
Map.containsKey : 'K -> Map<'K, 'V> -> bool
```

**Decomposition**:
```fsharp
let containsKey key map =
    match tryFind key map with
    | Some _ -> true
    | None -> false
```

### 5.4 Map.toList

```fsharp
Map.toList : Map<'K, 'V> -> ('K * 'V) list
```

**Decomposition** (in-order traversal):
```fsharp
let rec toList map =
    if isEmpty map then []
    else
        let leftList = toList (left map)
        let current = (mapKey map, mapValue map)
        let rightList = toList (right map)
        append leftList (current :: rightList)
```

**Note**: Returns keys in sorted order.

### 5.5 Map.keys / Map.values

```fsharp
Map.keys : Map<'K, 'V> -> seq<'K>
Map.values : Map<'K, 'V> -> seq<'V>
```

**Decomposition**: [Lazy in-order traversal](seq-representation.md) yielding only key or value component.

### 5.6 Map.fold

```fsharp
Map.fold : ('S -> 'K -> 'V -> 'S) -> 'S -> Map<'K, 'V> -> 'S
```

**Decomposition** (in-order traversal):
```fsharp
let rec fold folder state map =
    if isEmpty map then state
    else
        let state' = fold folder state (left map)
        let state'' = folder state' (mapKey map) (mapValue map)
        fold folder state'' (right map)
```

### 5.7 Map.forall

```fsharp
Map.forall : ('K -> 'V -> bool) -> Map<'K, 'V> -> bool
```

**Decomposition** (short-circuit):
```fsharp
let rec forall predicate map =
    if isEmpty map then true
    else
        predicate (mapKey map) (mapValue map) &&
        forall predicate (left map) &&
        forall predicate (right map)
```

### 5.8 Map.isEmpty

```fsharp
Map.isEmpty : Map<'K, 'V> -> bool
```

**Implementation**: Primitive, not decomposed. One `memref.load` of the `height` slot and one compare against the literal 0; equivalently, index equality with the sentinel node. Never a null check.

### 5.9 Map.remove

```fsharp
Map.remove : 'K -> Map<'K, 'V> -> Map<'K, 'V>
```

**Decomposition**:
```fsharp
let rec remove key map =
    if isEmpty map then map                                    // key absent: the sentinel's index, unchanged
    else
        let cmp = compare key (mapKey map)
        if cmp < 0 then rebalance (setLeft map (remove key (left map)))
        elif cmp > 0 then rebalance (setRight map (remove key (right map)))
        elif isEmpty (left map) then right map                 // at most one child: the other link's index; 0 when map was a leaf
        elif isEmpty (right map) then left map
        else
            let (k, v, right') = spliceMin (right map)         // in-order successor of map
            rebalance (node k v (left map) right' (height map))

// spliceMin t: the least key and value of t, and t without that node.
// Witnessed only under the test isEmpty t = false on the same index (the site above and the
// recursive site below), which is the dominance VC-GUARD (§2.3) requires for mapKey t / mapValue t.
and spliceMin t =
    if isEmpty (left t) then (mapKey t, mapValue t, right t)
    else
        let (k, v, left') = spliceMin (left t)
        (k, v, rebalance (setLeft t left'))
```

**Complexity**: O(log n) comparisons, O(log n) allocations (path copying)

Every result of `remove` is an index: the sentinel's index `0` when the last key is removed, a child's index when the removed node had at most one child, and a fresh node otherwise. No case yields an absent link, no case reads `key` or `value` of the sentinel, and the sentinel is never written: a removal that empties the tree returns `0`, it does not clear a node in place.

## 6. Structural Sharing

When modifying a map, only nodes along the path from root to the modified position are newly placed. All other nodes are shared:

```
Original:           After add "d" 4:
    b,2                 b,2 (NEW - right child changed)
   / \                 /   \
  a,1 c,3   -->     a,1    c,3 (NEW - right child added)
                            \
                            d,4 (NEW)
```

**Sharing**: `a,1` node is shared between both versions.

Sharing is index aliasing within one arena. The path copies `b,2` and `c,3` are placed in the arena that holds the nodes they link to, and both roots reach `a,1` by the same arena-relative index; VC-LINK ranges over that arena's extent. That is the placement rule at every `node` site: a node linking to a non-sentinel node is placed in that node's arena, and a node whose links are all `0` is placed in the arena the lifetime lattice selects for its value. The rule constrains classification, not only placement: a derived version reaches the nodes it shares, so the lifetime class of a tree's arena is the join, over the graph's enumerated derivation edges at saturation, of the classes of every version derived from it (a lattice-family fact, [Program Hypergraph §6](program-hypergraph.md)); the source is placed in the covering arena, and a join with no home on the target is the lifetime error of [Discriminated Union Representation §8.2](discriminated-union-representation.md), never a copy and never a cross-arena link. Every leaf link, including both links of `d,4`, indexes the sentinel at offset 0 of that arena (§2.2), which every `Map<'K, 'V>` hosted in the arena shares.

## 7. Normative Requirements

1. **AVL Property**: All Map operations SHALL maintain AVL balance (|height(left) - height(right)| ≤ 1)
2. **Ordering**: Map iteration SHALL yield keys in comparison order (smallest to largest)
3. **Immutability**: All modification operations SHALL return new maps without mutating originals
4. **Structural Sharing**: Unchanged subtrees SHALL be shared between versions as index aliasing within one arena (§6)
5. **Empty Map**: `Map.empty<'K, 'V>` SHALL be the index literal `0`: the sentinel at offset 0 of the hosting arena, initialised at arena creation from the one program-lifetime immutable sentinel image of the instantiation (§2.2), which SHALL reside in the platform's declared immutable program-lifetime space cited by a `Resides` edge (VC-RES, §2.3); `Map.isEmpty` SHALL be the literal test `height = 0`
6. **Sentinel Immutability**: The sentinel node SHALL reside `ReadOnly`; a store through it SHALL be diagnosed as CCS8020; `Map.add` on the sentinel SHALL place a fresh arena node
7. **Links**: `left` and `right` SHALL be arena-relative `index` values; every link store site SHALL discharge VC-LINK at saturation before witnessing and every link load SHALL inherit it; no operation SHALL test a link for absence
8. **Guarded Reads**: Every read of `key` or `value` SHALL be dominated by `height ≠ 0` (VC-GUARD)
9. **Comparison**: Key comparison SHALL use `compare` function from `'K : comparison` constraint
10. **[Arena Placement](memory-regions.md)**: Map nodes SHALL be placed in the arena chosen by the lifetime lattice ([Closure Representation §3.3](closure-representation.md)), not a GC heap

## 8. SSA Cost Formulas

| Operation | Complexity | SSA Operations |
|-----------|------------|----------------|
| `Map.add` | O(log n) | ~15 per level (compare + branch + possible rotate) |
| `Map.tryFind` | O(log n) | ~8 per level (compare + branch + recurse) |
| `Map.toList` | O(n) | ~10 per node (visit + cons) |
| `Map.fold` | O(n) | ~6 per node + folder cost |
| `Map.isEmpty` | O(1) | 2 (`memref.load` height, `arith.cmpi` against 0) |

A link traversal (`Map.left`, `Map.right`) is one `memref.load` of an `index`. No row counts an absence check: every traversal terminates through the sentinel node's `height` slot at the same cost as any other node's.

## 9. References

- [Native Type Universe § 5.4 Map](native-type-universe.md#54-map) - Map memory layout
- [List Operations Representation](list-operations-representation.md) - Comparison with List HOFs
- Baker MapRecipes.fs - Implementation reference
- Okasaki, "Purely Functional Data Structures" - AVL tree algorithms
- [Closure Representation §3.3](closure-representation.md) - lifetime lattice that places map nodes; the sentinel image is its program-lifetime point
- [Program Hypergraph §6](program-hypergraph.md) - `Resides` edge citing the platform description
- [Access Kinds](access-kinds.md) - `ReadOnly` residence of the sentinel node
