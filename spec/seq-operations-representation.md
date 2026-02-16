---
title: "Sequence Operations Representation in Clef"
weight: 220
---

> **Status**: Normative
> **Last Updated**: 2026-01-19
> **Depends On**: [Closure Representation](../closure-representation.md), [Seq Representation](../seq-representation.md)

## 1. Overview

Clef implements sequence operations (`Seq.map`, `Seq.filter`, `Seq.take`, `Seq.fold`, `Seq.collect`) as **wrapper sequences** that compose the flat closure architecture. This chapter specifies the memory representation, copy semantics, and composition model for sequence transformations.

**Key Insight**: Seq operations create wrapper sequences that contain both an inner sequence AND a transformation closure, both inlined (copied by value), following the flat closure model.

## 2. Relationship to Prior Chapters

Sequence operations build on:

- **[Closure Representation](../closure-representation.md)** - Mapper/predicate closures are flat closures
- **[Lazy Representation](../lazy-representation.md)** - Extended closure pattern with state prefix
- **[Seq Representation](../seq-representation.md)** - Base seq struct with MoveNext state machine

**Progressive Extension Pattern**:
```
PRD-11 (Closures)     → Flat closure: {code_ptr, cap₀, cap₁, ...}
         ↓ extends
PRD-14 (Lazy)         → Extended closure: {computed, value, code_ptr, cap₀...}
         ↓ extends
PRD-15 (SimpleSeq)    → State machine: {state, current, code_ptr, cap₀..., internal₀...}
         ↓ composes
PRD-16 (SeqOperations)→ Wrapper seq: {state, current, code_ptr, inner_seq, closure}
```

## 3. Operation Classification

### 3.1 Transformers vs Consumers

| Category | Operations | Behavior | Returns |
|----------|------------|----------|---------|
| **Transformer** | `Seq.map`, `Seq.filter`, `Seq.take` | Creates wrapper sequence | `seq<'T>` |
| **Consumer** | `Seq.fold` | Eagerly iterates, accumulates | `'S` |
| **Nested Transformer** | `Seq.collect` | Creates wrapper with nested iteration | `seq<'T>` |

**Transformers** create a new seq struct that wraps the input sequence. Iteration is lazy.

**Consumers** immediately iterate the input sequence and return a non-sequence value.

### 3.2 Closure Requirements

Each operation takes a function argument:

| Operation | Function Type | Role |
|-----------|---------------|------|
| `Seq.map` | `'a -> 'b` | Mapper - transforms each element |
| `Seq.filter` | `'a -> bool` | Predicate - selects matching elements |
| `Seq.take` | (none) | Count parameter, not a closure |
| `Seq.fold` | `'s -> 'a -> 's` | Folder - accumulates state |
| `Seq.collect` | `'a -> seq<'b>` | Mapper - produces inner sequences |

These function arguments are **flat closures** (per [Closure Representation](../closure-representation.md)) and may capture variables from their defining scope.

## 4. Memory Layout Specifications

### 4.1 MapSeq Structure

```
MapSeq<A, B> with inner: Seq<A>, mapper: Closure<A -> B>
┌─────────────────────────────────────────────────────────────────────────┐
│ state: i32              (4 bytes) - wrapper state (always 0 or 1)       │
├─────────────────────────────────────────────────────────────────────────┤
│ current: B              (sizeof(B) bytes) - current transformed value   │
├─────────────────────────────────────────────────────────────────────────┤
│ code_ptr: ptr           (8 bytes) - MapMoveNext function address        │
├─────────────────────────────────────────────────────────────────────────┤
│ inner_seq: Seq<A>       (sizeof(Seq<A>) bytes) - INLINED, copied        │
├─────────────────────────────────────────────────────────────────────────┤
│ mapper: Closure         (sizeof(Closure) bytes) - INLINED, copied       │
└─────────────────────────────────────────────────────────────────────────┘

Field Indices:
  [0] = state
  [1] = current
  [2] = code_ptr
  [3] = inner_seq (entire struct, not pointer)
  [4] = mapper (entire closure struct, not pointer)
```

**CRITICAL**: Both `inner_seq` and `mapper` are **inlined** (copied by value into the wrapper struct), not stored by pointer. This follows the flat closure principle of self-contained structs.

### 4.2 FilterSeq Structure

```
FilterSeq<A> with inner: Seq<A>, predicate: Closure<A -> bool>
┌─────────────────────────────────────────────────────────────────────────┐
│ state: i32              (4 bytes)                                        │
├─────────────────────────────────────────────────────────────────────────┤
│ current: A              (sizeof(A) bytes) - current matching value       │
├─────────────────────────────────────────────────────────────────────────┤
│ code_ptr: ptr           (8 bytes) - FilterMoveNext function address      │
├─────────────────────────────────────────────────────────────────────────┤
│ inner_seq: Seq<A>       (sizeof(Seq<A>) bytes) - INLINED                 │
├─────────────────────────────────────────────────────────────────────────┤
│ predicate: Closure      (sizeof(Closure) bytes) - INLINED                │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 TakeSeq Structure

```
TakeSeq<A> with inner: Seq<A>
┌─────────────────────────────────────────────────────────────────────────┐
│ state: i32              (4 bytes)                                        │
├─────────────────────────────────────────────────────────────────────────┤
│ current: A              (sizeof(A) bytes)                                │
├─────────────────────────────────────────────────────────────────────────┤
│ code_ptr: ptr           (8 bytes) - TakeMoveNext function address        │
├─────────────────────────────────────────────────────────────────────────┤
│ inner_seq: Seq<A>       (sizeof(Seq<A>) bytes) - INLINED                 │
├─────────────────────────────────────────────────────────────────────────┤
│ remaining: i32          (4 bytes) - count of elements still to take      │
└─────────────────────────────────────────────────────────────────────────┘
```

**Note**: `Seq.take` does not have a closure parameter. The `remaining` counter serves as the configuration.

### 4.4 CollectSeq Structure (flatMap)

```
CollectSeq<A, B> with outer: Seq<A>, mapper: Closure<A -> Seq<B>>
┌─────────────────────────────────────────────────────────────────────────┐
│ state: i32              (4 bytes) - 0=initial, 1=iterating_inner, -1=done│
├─────────────────────────────────────────────────────────────────────────┤
│ current: B              (sizeof(B) bytes)                                │
├─────────────────────────────────────────────────────────────────────────┤
│ code_ptr: ptr           (8 bytes) - CollectMoveNext function address     │
├─────────────────────────────────────────────────────────────────────────┤
│ outer_seq: Seq<A>       (sizeof(Seq<A>) bytes) - INLINED                 │
├─────────────────────────────────────────────────────────────────────────┤
│ mapper: Closure         (sizeof(Closure) bytes) - INLINED                │
├─────────────────────────────────────────────────────────────────────────┤
│ inner_seq: Seq<B>       (sizeof(Seq<B>) bytes) - current inner seq       │
└─────────────────────────────────────────────────────────────────────────┘
```

**Complexity Note**: `Seq.collect` requires storing the current inner sequence state. The `inner_seq` field is updated when advancing to a new outer element.

## 5. Copy Semantics (NORMATIVE)

### 5.1 Value Copy at Wrapper Creation

When creating a wrapper sequence:

```fsharp
let mapped = Seq.map mapper innerSeq
```

The wrapper creation **copies both `innerSeq` and `mapper` by value** into the wrapper struct:

```mlir
// Create MapSeq wrapper - COPIES both values
%undef = llvm.mlir.undef : !map_seq_type
%s0 = llvm.insertvalue %zero, %undef[0] : !map_seq_type           // state = 0
%s1 = llvm.insertvalue %mapMoveNext_ptr, %s0[2] : !map_seq_type   // code_ptr
%s2 = llvm.insertvalue %inner_seq_VALUE, %s1[3] : !map_seq_type   // COPY inner
%s3 = llvm.insertvalue %mapper_VALUE, %s2[4] : !map_seq_type      // COPY mapper
```

### 5.2 Why Copy Semantics?

**Independence**: Each wrapper owns its own copy of the inner seq's state. Multiple iterations of the same wrapper are independent.

**No Aliasing**: No shared mutable state between different wrappers created from the same source.

**Lifetime Simplicity**: The wrapper struct contains everything it needs. No dangling references.

**Example**:
```fsharp
let source = seq { yield 1; yield 2; yield 3 }
let doubled = Seq.map (fun x -> x * 2) source

// First iteration
for x in doubled do printfn "%d" x  // 2 4 6

// Second iteration - independent, works correctly
for x in doubled do printfn "%d" x  // 2 4 6 again
```

Both iterations work because each `for` expression copies `doubled` into its own iteration state.

### 5.3 Closure Invocation from Wrapper

When invoking the mapper/predicate, the closure is extracted from the wrapper and invoked per the flat closure convention:

```mlir
// In MapMoveNext:
// 1. Extract mapper closure (value copy in struct)
%mapper = llvm.extractvalue %wrapper[4] : !map_seq_type -> !closure_type

// 2. Extract code_ptr and captures from closure
%code_ptr = llvm.extractvalue %mapper[0] : !closure_type -> !llvm.ptr
%cap0 = llvm.extractvalue %mapper[1] : !closure_type -> ...  // if captures exist

// 3. Invoke: code_ptr(captures..., value)
%result = llvm.call %code_ptr(%cap0, %inner_val) : (...) -> !output_type
```

## 6. Composition Model

### 6.1 Nested Struct Pattern

When operations are composed:

```fsharp
let pipeline =
    source
    |> Seq.filter (fun x -> x % 2 = 0)
    |> Seq.map (fun x -> x * 2)
    |> Seq.take 5
```

The result is a **nested struct**:

```
TakeSeq {
    state, current, code_ptr,
    inner: MapSeq {
        state, current, code_ptr,
        inner: FilterSeq {
            state, current, code_ptr,
            inner: source_seq,
            predicate: {...}
        },
        mapper: {...}
    },
    remaining: 5
}
```

### 6.2 Struct Size Growth

Struct size is **compile-time known** and grows with composition depth:

```
size(TakeSeq(MapSeq(FilterSeq(source)))) = 
    base_TakeSeq + size(MapSeq(FilterSeq(source))) =
    base_TakeSeq + base_MapSeq + size(FilterSeq(source)) = ...
```

**Trade-off**: For typical pipeline depths (3-5 operations), struct sizes remain reasonable (hundreds of bytes). Very deep pipelines may benefit from alternative strategies (future optimization).

### 6.3 MoveNext Cascade

Calling MoveNext on the outermost wrapper cascades inward:

```
TakeMoveNext:
    if remaining > 0:
        if MapMoveNext(inner_map):    // <- calls into nested struct
            copy inner_map.current to current
            remaining--
            return true
    return false

MapMoveNext:
    if FilterMoveNext(inner_filter):  // <- calls into nested struct
        current = mapper(inner_filter.current)
        return true
    return false

FilterMoveNext:
    while SourceMoveNext(inner_source):
        if predicate(inner_source.current):
            current = inner_source.current
            return true
    return false
```

## 7. MoveNext Implementations

### 7.1 MapMoveNext Algorithm

```
MapMoveNext(self: ptr<MapSeq<A,B>>) -> bool:
    if inner.MoveNext():
        self.current = self.mapper(inner.current)
        return true
    return false
```

**State Values**: `0` = not started (same as `1`), `1` = active, `-1` = done (not used, delegated to inner)

### 7.2 FilterMoveNext Algorithm

```
FilterMoveNext(self: ptr<FilterSeq<A>>) -> bool:
    while inner.MoveNext():
        if self.predicate(inner.current):
            self.current = inner.current
            return true
    return false
```

**Note**: Filter loops internally until finding a match or exhausting the inner sequence.

### 7.3 TakeMoveNext Algorithm

```
TakeMoveNext(self: ptr<TakeSeq<A>>) -> bool:
    if self.remaining > 0:
        if inner.MoveNext():
            self.current = inner.current
            self.remaining--
            return true
    return false
```

**Note**: `remaining` is decremented on each successful iteration, providing count limiting.

### 7.4 CollectMoveNext Algorithm

```
CollectMoveNext(self: ptr<CollectSeq<A,B>>) -> bool:
    loop:
        // Try advancing current inner sequence
        if self.state == 1 and inner_seq.MoveNext():
            self.current = inner_seq.current
            return true
        
        // Inner exhausted or not started - advance outer
        if outer_seq.MoveNext():
            self.inner_seq = self.mapper(outer_seq.current)  // New inner seq
            self.state = 1
            continue loop
        
        // Outer exhausted
        self.state = -1
        return false
```

**Complexity**: `Seq.collect` maintains state for both outer and inner iteration.

## 8. Seq.fold: Eager Consumer

Unlike transformers, `Seq.fold` does **not** create a wrapper sequence. It immediately consumes the input:

```fsharp
let sum = Seq.fold (fun acc x -> acc + x) 0 source
```

**Implementation**:
```mlir
func @seq_fold(%folder: !closure, %initial: i64, %source: !seq_type) -> i64 {
    // Allocate source on stack for mutation
    %seq_alloca = llvm.alloca 1 x !seq_type : !llvm.ptr
    llvm.store %source, %seq_alloca
    
    // Accumulator
    %acc_alloca = llvm.alloca 1 x i64 : !llvm.ptr
    llvm.store %initial, %acc_alloca
    
    // Iteration loop
    scf.while : () -> () {
        %moveNext_ptr = llvm.getelementptr %seq_alloca[0, 2] : !llvm.ptr
        %moveNext = llvm.load %moveNext_ptr : !llvm.ptr
        %has_next = llvm.call %moveNext(%seq_alloca) : (!llvm.ptr) -> i1
        scf.condition(%has_next)
    } do {
        // Get current element
        %curr_ptr = llvm.getelementptr %seq_alloca[0, 1] : !llvm.ptr
        %elem = llvm.load %curr_ptr : i64
        
        // Apply folder
        %acc = llvm.load %acc_alloca : i64
        %folder_code = llvm.extractvalue %folder[0] : !closure -> !llvm.ptr
        %new_acc = llvm.call %folder_code(%acc, %elem) : (i64, i64) -> i64
        llvm.store %new_acc, %acc_alloca
        
        scf.yield
    }
    
    %result = llvm.load %acc_alloca : i64
    return %result : i64
}
```

## 9. SSA Cost Formulas

### 9.1 Wrapper Creation Costs

| Operation | Formula | Breakdown |
|-----------|---------|-----------|
| `Seq.map` | `5 + sizeof(inner) + sizeof(mapper)` | state(1) + code_ptr(1) + insertvalue×3 + inner fields + mapper fields |
| `Seq.filter` | `5 + sizeof(inner) + sizeof(predicate)` | Same structure |
| `Seq.take` | `6 + sizeof(inner)` | +1 for remaining counter |
| `Seq.collect` | `5 + sizeof(outer) + sizeof(mapper) + sizeof(inner_seq_type)` | Includes inner seq slot |

### 9.2 MoveNext SSA Costs

| Operation | Per-Invocation SSAs | Notes |
|-----------|---------------------|-------|
| MapMoveNext | `10 + N_mapper_caps` | GEP + load + call + extract + store |
| FilterMoveNext | `12 + N_pred_caps` | +loop overhead |
| TakeMoveNext | `14` | +remaining check and decrement |
| CollectMoveNext | `20 + N_mapper_caps` | +inner seq management |

## 10. Normative Requirements

1. **Flat Representation**: Seq operation wrappers SHALL use flat struct representation with inner seq and closure inlined
2. **Copy Semantics**: Wrapper creation SHALL copy inner seq and closure by value, not by pointer
3. **Field Order**: Wrapper struct fields SHALL be ordered: state, current, code_ptr, inner_seq, closure/config
4. **Closure Invocation**: Mapper/predicate invocation SHALL follow flat closure calling convention (extract code_ptr and captures, call with captures prepended)
5. **No Heap**: Wrapper sequences SHALL NOT be allocated on GC-managed heap
6. **Composition = Nesting**: Composed operations SHALL produce nested structs, not linked structures
7. **Eager Consumers**: `Seq.fold` SHALL consume immediately, not create wrapper

## 11. Test Cases

### 11.1 Seq.map with Captured Value

```fsharp
let scale factor xs = Seq.map (fun x -> x * factor) xs
let scaled = scale 3 (seq { yield 1; yield 2; yield 3 })
// Expected: 3 6 9
```

**Validates**: Mapper closure captures `factor` correctly.

### 11.2 Deep Composition

```fsharp
let pipeline =
    source
    |> Seq.filter (fun x -> x % 2 = 0)
    |> Seq.map (fun x -> x * 2)
    |> Seq.filter (fun x -> x > 10)
    |> Seq.take 5
```

**Validates**: 4-level nested struct works correctly.

### 11.3 Copy Semantics Independence

```fsharp
let doubled = Seq.map (fun x -> x * 2) source
let iter1 = doubled |> Seq.take 3 |> Seq.toList  // [2; 4; 6]
let iter2 = doubled |> Seq.take 3 |> Seq.toList  // [2; 4; 6] - same, independent
```

**Validates**: Each pipeline copies `doubled`, maintaining independence.

## 12. References

- [Closure Representation](../closure-representation.md) - Flat closure architecture
- [Seq Representation](../seq-representation.md) - Base seq state machine
- PRD-16: SeqOperations - Implementation requirements
- PRD-11: Closures - Flat closure foundation
