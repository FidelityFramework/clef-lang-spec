# Sequence Expression Representation in F# Native

> **Normative specification for seq expression memory layout and state machine semantics in fsnative compilation.**

## 1. Overview

F# Native implements `seq { }` expressions as state machine closures that extend the flat closure architecture. Sequence expressions are resumable computations that yield values lazily. This chapter specifies the memory representation, state machine generation, and the critical **Sequential flattening** pattern required for correct pre/post-yield expression extraction.

## 2. Relationship to Closures and Lazy

Sequence expressions build on the flat closure representation specified in [Closure Representation](closure-representation.md) and extend the thunk pattern from [Lazy Representation](lazy-representation.md).

**Progressive Extension Pattern**:
```
PRD-11 (Closures)     → Flat closure: {code_ptr, cap₀, cap₁, ...}
         ↓ extends (adds state prefix)
PRD-14 (Lazy)         → Extended closure: {computed, value, code_ptr, cap₀, ...}
         ↓ extends (adds INTERNAL STATE suffix)
PRD-15 (SimpleSeq)    → State machine closure: {state, current, code_ptr, cap₀, ..., internalState₀, ...}
```

**Key Insight**: A sequence expression creates a struct containing both captured values from the enclosing scope AND internal mutable state declared within the seq body.

## 3. Primitive Sequence Values

### 3.1 Seq.empty

`Seq.empty<'T>` is the degenerate sequence containing no elements. It is a polymorphic value:

```fsharp
Seq.empty<'T> : seq<'T>
```

**Representation**: `Seq.empty` creates a minimal seq struct with:
- `state = -1` (already exhausted)
- `current = default<'T>` (never accessed)
- `code_ptr` pointing to a trivial MoveNext that returns `false`
- No captures, no internal state

```
Seq.empty<T>
┌─────────────────────────────────────────────────────────────────────────┐
│ state: i32 = -1       (4 bytes) - already done                          │
├─────────────────────────────────────────────────────────────────────────┤
│ current: T            (sizeof(T) bytes) - undefined (never read)        │
├─────────────────────────────────────────────────────────────────────────┤
│ code_ptr: ptr         (8 bytes) - trivial MoveNext (always false)       │
└─────────────────────────────────────────────────────────────────────────┘
```

**MoveNext for Seq.empty**:
```
func @seq_empty_movenext(%ptr: !llvm.ptr) -> i1 {
    return %false : i1
}
```

Alternatively, an implementation MAY optimize `Seq.empty` to immediately set `state = -1` without a code pointer, as the MoveNext is never meaningfully called.

**SSA Cost**: 3 (undef struct, insert state=-1, insert code_ptr)

### 3.2 Relationship to seq { }

`Seq.empty<'T>` is semantically equivalent to:

```fsharp
seq<'T> { }  // Empty seq expression
```

However, `Seq.empty` is a primitive that avoids state machine generation entirely. An implementation SHOULD recognize `seq { }` with no body and lower it to the same representation as `Seq.empty`.

## 4. Memory Layout Specification

### 4.1 Seq Structure

A seq value in F# Native is a struct containing:

```
Seq<T> with captures [c₁: T₁, ..., cₘ: Tₘ] and internal state [s₁: S₁, ..., sₖ: Sₖ]
┌─────────────────────────────────────────────────────────────────────────┐
│ state: i32              (4 bytes) - state machine position              │
├─────────────────────────────────────────────────────────────────────────┤
│ current: T              (sizeof(T) bytes) - current yielded value       │
├─────────────────────────────────────────────────────────────────────────┤
│ code_ptr: ptr           (8 bytes) - MoveNext function address           │
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
  [2] = code_ptr (MoveNext function)
  [3..m+2] = captured values from enclosing scope
  [m+3..m+k+2] = internal mutable state from seq body
```

### 3.2 Captures vs Internal State

| Category | Definition Location | Initialization Time | Access Pattern |
|----------|---------------------|---------------------|----------------|
| **Captures** | Enclosing scope | At seq struct creation | Read-only in MoveNext |
| **Internal State** | Inside seq body (`let mutable`) | At first MoveNext (state 0) | Read-modify-write between yields |

**Example**:
```fsharp
let multiplesOf factor count = seq {
    let mutable i = 1        // INTERNAL STATE → index [m+3]
    while i <= count do      // 'count' is CAPTURE → index [4]
        yield i * factor     // 'factor' is CAPTURE → index [3]
        i <- i + 1
}
// Struct: {state, current, code_ptr, factor, count, i}
//         [0]    [1]      [2]       [3]     [4]    [5]
```

### 3.3 State Values

| State | Meaning |
|-------|---------|
| 0 | Initial - not yet started |
| 1..N | After yield N - resumption point |
| -1 | Done - sequence exhausted |

## 5. MoveNext Calling Convention

### 5.1 Struct Pointer Passing

Following the lazy thunk pattern, MoveNext receives a pointer to its containing seq struct:

**MoveNext Signature**:
```
moveNext: (ptr<Seq<T>>) -> i1
```

Returns `true` if a value was yielded (available in `current`), `false` if exhausted.

### 5.2 State Machine Structure

For while-based seq expressions, the MoveNext function has this CFG:

```
entry:
    load state
    switch state: [0 → ^s0, 1 → ^s1, default → ^done]

^s0:  // Initial state
    initialize internal state variables
    br ^check

^s1:  // Resume after yield
    execute post-yield expressions
    br ^check

^check:
    evaluate while condition
    cond_br condition, ^yield, ^done

^yield:
    execute pre-yield expressions
    compute yield value
    store to current field
    set state = 1
    return true

^done:
    set state = -1
    return false
```

## 6. PSG Structure and Sequential Flattening

### 6.1 The Nested Sequential Problem

F# source code with statements before/after yield results in deeply nested `Sequential` nodes in the PSG:

```fsharp
while i <= count do
    sum <- sum + i    // pre-yield
    yield sum
    i <- i + 1        // post-yield
```

**PSG Structure** (simplified):
```
WhileBody = Sequential([
    Set(sum <- sum + i),        // [0] - obviously pre-yield
    Sequential([                 // [1] - contains yield AND post-yield
        Yield(sum),
        Set(i <- i + 1)
    ])
])
```

### 6.2 Naive Split Failure

A naive `splitAtYield` that only looks at top-level nodes fails:

```
splitAtYield([Set(sum), Sequential([Yield, Set(i)])], [])
  → Check Set(sum): no yields → pre = [Set(sum)]
  → Check Sequential([...]): has yields → return (pre=[Set(sum)], post=[])
                                                            ↑
                                                     WRONG! i <- i + 1 is INSIDE
```

The post-yield `Set(i <- i + 1)` is **inside** the nested Sequential, not after it in the outer list.

### 6.3 NORMATIVE: Sequential Flattening Requirement

**All nested Sequential nodes MUST be flattened before splitting at yield.**

The `flattenSequentials` function recursively expands nested Sequentials:

```fsharp
/// Flatten nested Sequentials into a single list of non-Sequential nodes
/// e.g., [A, Sequential([B, Sequential([C, D])])] → [A, B, C, D]
let rec flattenSequentials (graph: SemanticGraph) (nodeIds: NodeId list) : NodeId list =
    nodeIds
    |> List.collect (fun nodeId ->
        match isSequential graph nodeId with
        | Some innerNodes -> flattenSequentials graph innerNodes
        | None -> [nodeId])
```

After flattening:
```
flattenSequentials([Set(sum), Sequential([Yield, Set(i)])])
  → [Set(sum), Yield, Set(i)]
```

Now `splitAtYield` works correctly:
```
splitAtYield([Set(sum), Yield, Set(i)], [])
  → Check Set(sum): no yields → pre = [Set(sum)]
  → Check Yield: has yields → return (pre=[Set(sum)], post=[Set(i)])
                                                            ↑
                                                     CORRECT!
```

### 6.4 Complete Split Algorithm

```fsharp
let (preYield, postYield) =
    // CRITICAL: Flatten nested Sequentials FIRST
    let flattenedBody = flattenSequentials graph whileBodyNodes
    
    let rec splitAtYield (nodes: NodeId list) (pre: NodeId list) =
        match nodes with
        | [] -> (List.rev pre, [])
        | nodeId :: rest ->
            let nodeYields = collectYieldsInSubtree graph nodeId
            if not (List.isEmpty nodeYields) then
                // This node contains yield - rest is post-yield
                (List.rev pre, rest)
            else
                splitAtYield rest (nodeId :: pre)
    
    splitAtYield flattenedBody []
```

## 7. Post-Yield Expression Handling

### 7.1 Supported Expression Types

The `emitPostYield` function must handle these PSG node kinds:

| Kind | Example | Handling |
|------|---------|----------|
| `Set` | `i <- i + 1` | Load operands, compute, store |
| `Binding` (immutable) | `let temp = a + b` | Compute value, track in local map |
| `Sequential` | Multiple statements | Recursively process children |

### 7.2 Local Binding Tracking

For sequences like fibonacci:
```fsharp
yield a
let temp = a + b    // immutable local binding
a <- b
b <- temp           // references local binding
i <- i + 1
```

The `temp` binding is NOT in the struct - it's computed locally in MoveNext. Post-yield emission must:
1. Track local immutable bindings in a map
2. When evaluating VarRefs, check local map before struct fields

```fsharp
// Pseudocode for emitPostYield with local binding support
let mutable localBindings = Map.empty<string, SSA>

for expr in postYieldExprs do
    match expr.Kind with
    | Binding(name, valueExpr, isMutable=false) ->
        let (ops, ssa) = emitValue valueExpr localBindings
        localBindings <- Map.add name ssa localBindings
        ops  // No store - just track the SSA
    | Set(target, value) ->
        let (valueOps, valueSSA) = emitValue value localBindings
        valueOps @ storeToStruct target valueSSA
```

## 8. WhileBasedMoveNextInfo

### 8.1 Structure

```fsharp
type WhileBasedYieldInfo = {
    InitExprs: NodeId list       // let mutable declarations before while
    WhileNodeId: NodeId          // The WhileLoop node
    ConditionId: NodeId          // While condition expression
    PreYieldExprs: NodeId list   // Expressions BEFORE yield in while body
    YieldNodeId: NodeId          // The Yield node
    YieldValueId: NodeId         // Expression being yielded
    PostYieldExprs: NodeId list  // Expressions AFTER yield in while body
    ConditionalYield: ConditionalYieldInfo option  // If yield is inside an if
}
```

### 8.2 Population Requirements

1. `InitExprs`: All `let mutable` bindings between seq body start and while loop
2. `PreYieldExprs`: Non-yield nodes before yield in FLATTENED while body
3. `PostYieldExprs`: Non-yield nodes after yield in FLATTENED while body
4. `ConditionalYield`: Set if yield appears inside `if` within while body

## 9. SSA Cost Formula

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
| insert code_ptr | 1 |
| insert captures | N |
| internal state (const 0 + insert each) | 2 × M |

## 10. Normative Requirements

1. **Flat Representation**: Seq values SHALL use flat closure representation with captures AND internal state inlined
2. **Struct Layout**: Field order SHALL be: state, current, code_ptr, captures, internal_state
3. **Capture Indices**: Captures SHALL begin at index 3
4. **Seq.empty Representation**: `Seq.empty<'T>` SHALL be represented as a minimal seq struct with state=-1
5. **Internal State Indices**: Internal state SHALL begin at index 3 + capture_count
6. **Sequential Flattening**: Nested Sequentials in while body SHALL be flattened before pre/post yield splitting
7. **MoveNext Convention**: MoveNext SHALL receive pointer to containing seq struct
8. **State Machine**: State 0 = initial, positive = after yield N, -1 = done

## 11. Test Cases

### 11.1 triangularNumbers (Pre-yield + Post-yield)

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

**Struct**: `{state, current, code_ptr, count, sum, i}`

**MoveNext blocks**:
- `^s0`: sum=0, i=1, br check
- `^s1`: i=i+1, br check
- `^yield`: sum=sum+i, current=sum, state=1, return true

### 11.2 fibonacci (LetBinding in Post-yield)

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

**Struct**: `{state, current, code_ptr, count, a, b, i}`

**MoveNext ^s1**: Must compute `temp` locally, not load from struct.

## 12. Related Chapters

This chapter covers `seq { }` expressions (PRD-15). For **Seq module operations** (map, filter, take, fold, collect), see:

- [Seq Operations Representation](seq-operations-representation.md) - Wrapper structures, copy semantics, composition model

## References

- [Closure Representation](closure-representation.md) - Base flat closure architecture
- [Lazy Representation](lazy-representation.md) - Extended closure with memoization state
- [Seq Operations Representation](seq-operations-representation.md) - Seq module operations (PRD-16)
- PRD-15: SimpleSeq - Implementation requirements
- PRD-16: SeqOperations - Composed sequence operations
