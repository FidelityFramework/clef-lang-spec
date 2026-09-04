---
title: "Sequence Expression Representation"
weight: 340
category: Representation
status: normative
---

> **Normative specification for seq expression memory layout and state machine semantics in Clef compilation.**

## 1. Overview

Clef implements `seq { }` expressions as state machine closures that extend the flat closure architecture. Sequence expressions are resumable computations that yield values lazily. This chapter specifies the memory representation, state machine generation, and the critical **Sequential flattening** pattern required for correct pre/post-yield expression extraction.

## 2. Relationship to Closures and Lazy

Sequence expressions build on the flat closure representation specified in [Closure Representation](closure-representation.md) and extend the thunk pattern from [Lazy Representation](lazy-representation.md).

**Progressive Extension Pattern**:
```
PRD-11 (Closures)     → Flat closure: (fn, {cap₀, cap₁, ...})
         ↓ extends (adds state prefix)
PRD-14 (Lazy)         → Extended closure: (thunk, {computed, value, cap₀, ...})
         ↓ extends (adds INTERNAL STATE suffix)
PRD-15 (SimpleSeq)    → State machine closure: (moveNext, {state, current, cap₀, ..., internalState₀, ...})
```

**Key Insight**: A sequence expression creates a struct containing both captured values from the enclosing scope AND internal mutable state declared within the seq body.

**Allocation and Lifetime**: The seq struct is a value like any other closure-family struct, so its storage is chosen by the four-point lifetime lattice specified in [Closure Representation §3.3](closure-representation.md). A seq whose lifetime is scope-bounded lives on the stack; a seq that escapes to program lifetime is placed in static storage (`memref.global`), constructed once and held to program end. On a no-heap target only the scope-bounded and program-lifetime classes exist; a seq value that classifies as genuinely-dynamic on such a target is a compile-time lifetime error, not a silent heap allocation.

## 3. Primitive Sequence Values

### 3.1 Seq.empty

`Seq.empty<'T>` is the degenerate sequence containing no elements. It is a polymorphic value:

```fsharp
Seq.empty<'T> : seq<'T>
```

**Representation**: `Seq.empty` creates a minimal seq struct with:
- `state = -1` (already exhausted)
- `current = default<'T>` (never accessed)
- `moveNext` (the function-value half) naming a trivial MoveNext that returns `false`, or elided (below)
- No captures, no internal state

```
Seq.empty<T>
┌─────────────────────────────────────────────────────────────────────────┐
│ state: i32 = -1       (4 bytes) - already done                          │
├─────────────────────────────────────────────────────────────────────────┤
│ current: T            (sizeof(T) bytes) - undefined (never read)        │
└─────────────────────────────────────────────────────────────────────────┘
```

**MoveNext for Seq.empty**:
```
func.func @seq_empty_movenext(%env: memref<?xi64>) -> i1 {
    func.return %false : i1
}
```

Alternatively, an implementation MAY optimize `Seq.empty` to immediately set `state = -1` and elide the MoveNext function value, as it is never meaningfully called.

**SSA Cost**: 2 (undef struct, insert state=-1), plus 1 for `func.constant` when the MoveNext value is materialized

### 3.2 Relationship to seq { }

`Seq.empty<'T>` is semantically equivalent to:

```fsharp
seq<'T> { }  // Empty seq expression
 
```

However, `Seq.empty` is a primitive that avoids state machine generation entirely. An implementation SHOULD recognize `seq { }` with no body and lower it to the same representation as `Seq.empty`.

## 4. Memory Layout Specification

### 4.1 Seq Structure

A seq value in Clef is the two-value pair `(moveNext, env)` of [Closure Representation §6.3](closure-representation.md): `moveNext` is a function value, and the environment is a flat struct containing:

```
Seq<T> with captures [c₁: T₁, ..., cₘ: Tₘ] and internal state [s₁: S₁, ..., sₖ: Sₖ]
┌─────────────────────────────────────────────────────────────────────────┐
│ state: i32              (4 bytes) - state machine position              │
├─────────────────────────────────────────────────────────────────────────┤
│ current: T              (sizeof(T) bytes) - current yielded value       │
├─────────────────────────────────────────────────────────────────────────┤
│ c₁: T₁                  (captured value from enclosing scope)           │
├─────────────────────────────────────────────────────────────────────────┤
│ ...                                                                      │
├─────────────────────────────────────────────────────────────────────────┤
│ cₘ: Tₘ                  (last captured value)                           │
├─────────────────────────────────────────────────────────────────────────┤
│ s₁: S₁                  (internal mutable state from seq body)          │
├─────────────────────────────────────────────────────────────────────────┤
│ ...                                                                      │
├─────────────────────────────────────────────────────────────────────────┤
│ sₖ: Sₖ                  (last internal state variable)                  │
└─────────────────────────────────────────────────────────────────────────┘

Field Indices:
  [0] = state (i32)
  [1] = current (T)
  [2..m+1] = captured values from enclosing scope
  [m+2..m+k+1] = internal mutable state from seq body
```

The `MoveNext` symbol is never stored in the environment as data: it is the function-value half of the pair, elided where the consumer knows it. An earlier revision placed a `code_ptr` word at `[2]`; that slot is retired with the cast that populated it ([Backend Lowering Architecture §4](backend-lowering-architecture.md)).

### 4.2 Captures vs Internal State

| Category | Definition Location | Initialization Time | Access Pattern |
|----------|---------------------|---------------------|----------------|
| **Captures** | Enclosing scope | At seq struct creation | Read-only in MoveNext |
| **Internal State** | Inside seq body (`let mutable`) | At first MoveNext (state 0) | Read-modify-write between yields |

**Example**:
```fsharp
let multiplesOf factor count = seq {
    let mutable i = 1        // INTERNAL STATE → index [m+2]
    while i <= count do      // 'count' is CAPTURE → index [3]
        yield i * factor     // 'factor' is CAPTURE → index [2]
        i <- i + 1
}
// Environment: {state, current, factor, count, i}
//              [0]    [1]      [2]     [3]    [4]
 
```

### 4.3 State Values

| State | Meaning |
|-------|---------|
| 0 | Initial - not yet started |
| 1..N | After yield N - resumption point |
| -1 | Done - sequence exhausted |

## 5. MoveNext Calling Convention

### 5.1 Struct Pointer Passing

Following the lazy thunk convention, `MoveNext` receives its environment — the struct of §4.1 — as its sole parameter; the seq value is the pair `(moveNext, env)`:

**MoveNext Signature**:
```
moveNext: (memref<Exi8>) -> i1      // E = the environment extent, a literal at saturation
```

Returns `true` if a value was yielded (available in `current`), `false` if exhausted.

### 5.2 State Machine Structure

`MoveNext` dispatches on the `state` discriminant with `scf.index_switch`. Each case is the segment that runs from that state to its next yield (or to completion), and every case ends by storing the next state and yielding whether a value was produced:

```mlir
func.func private @moveNext(%seq: memref<Exi8>) -> i1 {
  %c0 = arith.constant 0 : index
  %sv = memref.view %seq[%c0][] : memref<Exi8> to memref<1xindex>
  %s  = memref.load %sv[%c0] : memref<1xindex>
  %more = scf.index_switch %s -> i1
    case 0 { ... initialize internal state, then run the loop segment ... }
    case 1 { ... post-yield segment; evaluate the condition;
             true:  pre-yield segment, store current, store state 1, scf.yield %true
             false: store state 2, scf.yield %false ... }
    default { %f = arith.constant false ; scf.yield %f : i1 }
  return %more : i1
}
```

No block-based control flow (`cf.br`, `cf.cond_br`) appears above the witness boundary. `scf.index_switch` over the literal state set is the structured form of the same machine; the pathway's standard `scf` lowering produces the blocks.

## 6. PSG Structure: Segments at Yield

A `seq { }` body is elaborated by the suspension recipe of [Delimited Continuation Representation §2](dcont-representation.md), with `yield` as the cut and the caller's pull as the only resumption edge. The recipe, not a shape recognizer, produces the state machine:

- **Segments.** Fan-out splits the body at each `yield`. In a `while`-shaped body the code before the yield and the code after it are the two segments adjacent to the cut, whatever nesting of `Sequential` nodes the surface syntax produced. Segmentation follows the graph's evaluation order, so no flattening or splitting of `Sequential` nodes is specified or needed.
- **State count.** A body with *N* yields folds to a discriminant over *N*+2 values (§4.3 shows *N* = 1).
- **Slots.** Each cut's live-across set is enumerated at elaboration: `let mutable` bindings threaded across the yield become internal-state slots; an immutable `let` whose scope does not cross a yield is evaluated within its segment and occupies no slot. Offsets and the extent `E` are literals settled by interference colouring over segment liveness.
- **Conditional yield.** A `yield` under `if` is a cut on one branch; the other branch continues the segment. The discriminant records which cut was reached; no separate conditional-yield structure exists.

The saturated result is a frame node — the environment node of [Closure Representation §7](closure-representation.md) in its state-machine slot class — whose segments are the `scf.index_switch` cases of §5.2. The middle end witnesses that structure; it does not recognize shapes, split expressions, or track bindings.

## 7. SSA Cost Formula

For a seq expression with `N` captures and `M` internal state variables:

```
SSA cost = 5 + N + (2 × M)
```

| Component | SSAs |
|-----------|------|
| state constant (0) | 1 |
| undef struct | 1 |
| insert state | 1 |
| addressof MoveNext | 1 |
| `func.constant` for MoveNext (elided when the consumer knows it) | 1 |
| insert captures | N |
| internal state (const 0 + insert each) | 2 × M |

## 8. Normative Requirements

1. **Flat Representation**: Seq values SHALL use flat closure representation with captures AND internal state inlined
2. **Struct Layout**: Field order SHALL be: state, current, captures, internal_state; no code pointer SHALL be stored in the environment
3. **Capture Indices**: Captures SHALL begin at index 2
4. **Seq.empty Representation**: `Seq.empty<'T>` SHALL be represented as a minimal seq struct with state=-1
5. **Internal State Indices**: Internal state SHALL begin at index 2 + capture_count
6. **Segmentation**: The body SHALL be segmented at each `yield` by the suspension recipe (§6); segmentation SHALL follow the graph's evaluation order, and no flattening or splitting of `Sequential` nodes is specified
7. **MoveNext Convention**: MoveNext SHALL receive its environment as its sole parameter; a seq value SHALL be the two-value pair `(moveNext, env)` of [Closure Representation §6.3](closure-representation.md)
8. **State Machine**: State 0 = initial, positive = after yield N, -1 = done; MoveNext SHALL dispatch on the state with `scf.index_switch` (§5.2), and no `cf.*` operation SHALL appear above the witness boundary

## 9. Test Cases

### 9.1 triangularNumbers (Pre-yield + Post-yield)

```fsharp
let triangularNumbers count = seq {
    let mutable sum = 0
    let mutable i = 1
    while i <= count do
        sum <- sum + i    // PRE-YIELD
        yield sum
        i <- i + 1        // POST-YIELD
}
```

**Environment**: `{state, current, count, sum, i}`

**MoveNext blocks**:
- `^s0`: sum=0, i=1, br check
- `^s1`: i=i+1, br check
- `^yield`: sum=sum+i, current=sum, state=1, return true

### 9.2 fibonacci (LetBinding in Post-yield)

```fsharp
let fibonacci count = seq {
    let mutable a = 0
    let mutable b = 1
    let mutable i = 0
    while i < count do
        yield a
        let temp = a + b  // LOCAL BINDING (not in struct)
        a <- b
        b <- temp         // References local binding
        i <- i + 1
}
```

**Environment**: `{state, current, count, a, b, i}`

**MoveNext ^s1**: Must compute `temp` locally, not load from struct.

## 10. Related Chapters

This chapter covers `seq { }` expressions (PRD-15). For **Seq module operations** (map, filter, take, fold, collect), see:

- [Seq Operations Representation](seq-operations-representation.md) - Wrapper structures, copy semantics, composition model

## References

- PRD-15: SimpleSeq - Implementation requirements
- PRD-16: SeqOperations - Composed sequence operations
