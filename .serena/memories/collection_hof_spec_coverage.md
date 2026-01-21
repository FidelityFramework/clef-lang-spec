# Collection HOF Specification Coverage

**Date**: 2026-01-20
**Context**: Full audit and update of fsnative-spec for collection HOFs implemented in Baker

## Summary

Conducted a full audit of fsnative-spec regarding higher-order functions for collections (List, Map, Set, Option, Seq). Created missing specification documents to match the Baker implementations in FNCS.

## What Existed Before

1. **seq-representation.md** - Complete spec for `seq<'T>` type and state machine
2. **seq-operations-representation.md** - Complete spec for Seq HOFs (map, filter, fold, take, collect)
3. **native-type-universe.md § 5.1 Option** - Brief option type layout (no HOFs)
4. **native-type-universe.md § 5.3 List** - Brief list type layout (no HOFs)
5. **Map and Set** - NOT documented at all

## What Was Created/Updated

### New Spec Files

1. **list-operations-representation.md** - Complete List HOF spec
   - Primitive operations: empty, isEmpty, head, tail, cons
   - HOF decompositions: map, filter, fold, exists, forall, rev, append, length, collect, sumBy, contains, tryPick, minBy, max, forall2
   - SSA cost formulas
   - Comparison with Seq (eager vs lazy)

2. **map-representation.md** - Complete Map type + HOF spec
   - AVL tree memory layout
   - Primitive operations: empty, isEmpty, node, key, value, left, right, height
   - AVL balancing algorithms: rotations, rebalance
   - HOF decompositions: add, tryFind, containsKey, toList, keys, values, fold, forall
   - Structural sharing explanation

3. **set-representation.md** - Complete Set type + HOF spec
   - AVL tree memory layout (optimized vs Map)
   - Primitive operations: empty, isEmpty, node, value, left, right, height
   - HOF decompositions: add, contains, toList, fold, forall, exists, union, intersect, difference

4. **option-operations-representation.md** - Complete Option HOF spec
   - Primitive operations: None, Some, isSome, isNone, get
   - HOF decompositions: map, bind, defaultValue, defaultWith, orElse, orElseWith, filter, exists, forall, iter, toList, toArray, flatten, map2, map3
   - Comparison with Result type

### Updated Files

1. **native-type-universe.md** - Added sections 5.4 (Map) and 5.5 (Set) with:
   - Memory layout specifications
   - AVL tree structure
   - When to use / when not to use guidance
   - MLIR type mappings

2. **Catalog.json** - Added new spec files to the main body in appropriate order

## Architecture Pattern

All collection HOF specs follow the same structure:

1. **Overview** - High-level description and key insight
2. **Memory Layout** - Struct definition with field indices
3. **Operation Classification**
   - Primitive operations (Alex witnesses directly)
   - HOFs (Baker decomposes)
4. **Decomposition Specifications** - Detailed decomposition for each HOF
5. **SSA Cost Formulas** - Performance characteristics
6. **Normative Requirements** - SHALL/MUST statements
7. **References** - Links to implementation and related specs

## Relationship to Baker Implementation

The spec documents mirror the Baker recipe implementations:

| Spec File | Baker Implementation |
|-----------|---------------------|
| list-operations-representation.md | Baker/Recipes/ListRecipes.fs |
| map-representation.md | Baker/Recipes/MapRecipes.fs |
| set-representation.md | Baker/Recipes/SetRecipes.fs |
| option-operations-representation.md | Baker/Recipes/OptionRecipes.fs |
| seq-operations-representation.md | Baker/Recipes/SeqRecipes.fs |

## Key Design Decisions Documented

1. **List HOFs** - Decompose to primitive cons/head/tail operations via foldLeft/foldRight patterns
2. **Map/Set** - AVL tree with structural sharing; nodes arena-allocated
3. **Option** - Stack-allocated voption semantics; tag-based discrimination (not null pointer)
4. **Empty collections** - Null pointer (List, Map, Set) vs tag=0 struct (Option)
5. **Seq vs List** - Lazy wrapper creation (Seq) vs eager allocation (List)
