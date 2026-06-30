---
title: "Map Representation"
weight: 370
category: Representation
status: normative
---

> **Status**: Normative
> **Last Updated**: 2026-01-20
> **Depends On**: [Native Type Universe § 5.4 Map](native-type-universe.md#54-map)

## 1. Overview

Clef implements `Map<'K, 'V>` as a persistent immutable AVL tree. Map operations are decomposed by Baker into primitive tree operations, with Alex witnessing the primitives directly.

**Key Insight**: Maps are self-balancing binary search trees. All mutations return new maps with structural sharing: unchanged subtrees are shared between old and new versions.

## 2. Memory Layout

### 2.1 Map Node Structure

```
Map<'K, 'V> node
┌─────────────────────────────────────────────────────────────────────────────────┐
│ key: 'K                 (sizeof<'K> bytes) - node key                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│ value: 'V               (sizeof<'V> bytes) - associated value                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│ left: ptr<Map<'K,'V>>   (8 bytes) - left subtree (keys < this key)              │
├─────────────────────────────────────────────────────────────────────────────────┤
│ right: ptr<Map<'K,'V>>  (8 bytes) - right subtree (keys > this key)             │
├─────────────────────────────────────────────────────────────────────────────────┤
│ height: i8              (1 byte) - subtree height for AVL balancing             │
└─────────────────────────────────────────────────────────────────────────────────┘

Field Indices:
  [0] = key
  [1] = value
  [2] = left
  [3] = right
  [4] = height
```

### 2.2 Empty Map

Empty map is represented as a null pointer:

```
Map.empty<'K, 'V> = null : ptr<Map<'K, 'V>>
```

**Property**: `Map.isEmpty` is a null pointer check.

## 3. Operation Classification

### 3.1 Primitive Operations (Alex Witnesses Directly)

| Operation | Signature | Description |
|-----------|-----------|-------------|
| `Map.empty` | `unit -> Map<'K, 'V>` | Returns null pointer |
| `Map.isEmpty` | `Map<'K, 'V> -> bool` | Null pointer check |
| `Map.node` | Internal | Create tree node (alloc + store fields) |
| `Map.key` | Internal | GEP field 0, load |
| `Map.value` | Internal | GEP field 1, load |
| `Map.left` | Internal | GEP field 2, load |
| `Map.right` | Internal | GEP field 3, load |
| `Map.height` | Internal | GEP field 4, load |

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

The balance factor of a node is: `height(right) - height(left)`

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

## 5. HOF Decomposition Specifications

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

**Implementation**: Direct null pointer check (primitive, not decomposed).

## 6. Structural Sharing

When modifying a map, only nodes along the path from root to the modified position are newly allocated. All other nodes are shared:

```
Original:           After add "d" 4:
    b,2                 b,2 (NEW - left child changed)
   / \                 /   \
  a,1 c,3   -->     a,1    c,3 (NEW - right child added)
                            \
                            d,4 (NEW)
```

**Sharing**: `a,1` node is shared between both versions.

## 7. Normative Requirements

1. **AVL Property**: All Map operations SHALL maintain AVL balance (|height(left) - height(right)| ≤ 1)
2. **Ordering**: Map iteration SHALL yield keys in comparison order (smallest to largest)
3. **Immutability**: All modification operations SHALL return new maps without mutating originals
4. **Structural Sharing**: Unchanged subtrees SHALL be shared between versions
5. **Empty Map**: `Map.empty` SHALL be represented as null pointer
6. **Comparison**: Key comparison SHALL use `compare` function from `'K : comparison` constraint
7. **[Arena Allocation](memory-regions.md)**: Map nodes SHALL be allocated in arenas, not GC heap

## 8. SSA Cost Formulas

| Operation | Complexity | SSA Operations |
|-----------|------------|----------------|
| `Map.add` | O(log n) | ~15 per level (compare + branch + possible rotate) |
| `Map.tryFind` | O(log n) | ~8 per level (compare + branch + recurse) |
| `Map.toList` | O(n) | ~10 per node (visit + cons) |
| `Map.fold` | O(n) | ~6 per node + folder cost |
| `Map.isEmpty` | O(1) | 2 (load ptr, icmp null) |

## 9. References

- [Native Type Universe § 5.4 Map](native-type-universe.md#54-map) - Map memory layout
- [List Operations Representation](list-operations-representation.md) - Comparison with List HOFs
- Baker MapRecipes.fs - Implementation reference
- Okasaki, "Purely Functional Data Structures" - AVL tree algorithms
