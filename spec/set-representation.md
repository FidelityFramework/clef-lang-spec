# Set Representation in Clef

> **Status**: Normative
> **Last Updated**: 2026-01-20
> **Depends On**: [Native Type Universe § 5.5 Set](native-type-universe.md#55-set)

## 1. Overview

Clef implements `Set<'T>` as a persistent immutable AVL tree. Set is structurally similar to `Map<'T, unit>` but with an optimized layout that omits the value field.

**Key Insight**: Sets share the same AVL balancing algorithms as Maps, but store only values (no key-value pairs), making them more memory-efficient for membership testing scenarios.

## 2. Memory Layout

### 2.1 Set Node Structure

```
Set<'T> node
┌─────────────────────────────────────────────────────────────────────────────────┐
│ value: 'T               (sizeof<'T> bytes) - node value                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│ left: ptr<Set<'T>>      (8 bytes) - left subtree (values < this value)          │
├─────────────────────────────────────────────────────────────────────────────────┤
│ right: ptr<Set<'T>>     (8 bytes) - right subtree (values > this value)         │
├─────────────────────────────────────────────────────────────────────────────────┤
│ height: i8              (1 byte) - subtree height for AVL balancing             │
└─────────────────────────────────────────────────────────────────────────────────┘

Field Indices:
  [0] = value
  [1] = left
  [2] = right
  [3] = height
```

### 2.2 Empty Set

Empty set is represented as a null pointer:

```
Set.empty<'T> = null : ptr<Set<'T>>
```

### 2.3 Comparison with Map Layout

| Field | Map<'K,'V> | Set<'T> |
|-------|------------|---------|
| key/value | `key: 'K` + `value: 'V` | `value: 'T` |
| left | `ptr<Map>` | `ptr<Set>` |
| right | `ptr<Map>` | `ptr<Set>` |
| height | `i8` | `i8` |
| **Total overhead** | 17 bytes + sizeof(K) + sizeof(V) | 17 bytes + sizeof(T) |

Set saves `sizeof(V)` per node compared to `Map<T, unit>`.

## 3. Operation Classification

### 3.1 Primitive Operations (Alex Witnesses Directly)

| Operation | Signature | Description |
|-----------|-----------|-------------|
| `Set.empty` | `unit -> Set<'T>` | Returns null pointer |
| `Set.isEmpty` | `Set<'T> -> bool` | Null pointer check |
| `Set.node` | Internal | Create tree node (alloc + store fields) |
| `Set.value` | Internal | GEP field 0, load |
| `Set.left` | Internal | GEP field 1, load |
| `Set.right` | Internal | GEP field 2, load |
| `Set.height` | Internal | GEP field 3, load |

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

## 5. HOF Decomposition Specifications

### 5.1 Set.add

```fsharp
Set.add : 'T -> Set<'T> -> Set<'T>
```

**Decomposition**:
```fsharp
let rec add value set =
    if isEmpty set then
        node value empty empty 1
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

**Implementation**: Direct null pointer check (primitive, not decomposed).

## 6. Normative Requirements

1. **AVL Property**: All Set operations SHALL maintain AVL balance
2. **Ordering**: Set iteration SHALL yield elements in comparison order
3. **Uniqueness**: Sets SHALL contain no duplicate elements (add of existing element is no-op)
4. **Immutability**: All modification operations SHALL return new sets without mutating originals
5. **Structural Sharing**: Unchanged subtrees SHALL be shared between versions
6. **Empty Set**: `Set.empty` SHALL be represented as null pointer
7. **Comparison**: Element comparison SHALL use `compare` function from `'T : comparison` constraint
8. **Arena Allocation**: Set nodes SHALL be allocated in arenas, not GC heap

## 7. SSA Cost Formulas

| Operation | Complexity | SSA Operations |
|-----------|------------|----------------|
| `Set.add` | O(log n) | ~12 per level (compare + branch + possible rotate) |
| `Set.contains` | O(log n) | ~6 per level (compare + branch + recurse) |
| `Set.toList` | O(n) | ~10 per node |
| `Set.fold` | O(n) | ~5 per node + folder cost |
| `Set.union` | O(m log(n+m)) | add cost × smaller set size |
| `Set.isEmpty` | O(1) | 2 (load ptr, icmp null) |

## 8. References

- [Native Type Universe § 5.5 Set](native-type-universe.md#55-set) - Set memory layout
- [Map Representation](map-representation.md) - AVL algorithms (shared)
- Baker SetRecipes.fs - Implementation reference
