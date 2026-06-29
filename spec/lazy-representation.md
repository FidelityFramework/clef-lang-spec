---
title: "Lazy Value Representation in Clef"
weight: 320
---

> **Normative specification for lazy value memory layout and thunk semantics in Clef compilation.**

## 1. Overview

Clef implements `Lazy<'T>` as an extension of the flat closure architecture. Lazy values are thunks that defer computation until forced. This chapter specifies the memory representation, capture semantics, thunk calling convention, and memoization behavior.

## 2. Relationship to Closures

Lazy values build directly on the flat closure representation specified in [Closure Representation](closure-representation.md). A lazy value is a flat closure with additional fields for memoization state.

**Key Insight**: The `lazy` keyword creates a thunk, which is a nullary closure (a function of unit) with embedded state tracking whether evaluation has occurred.

## 3. Memory Layout Specification

### 3.1 Lazy Structure

A lazy value in Clef is a struct containing:

```
Lazy<T> with captures [c₁: T₁, ..., cₘ: Tₘ]
┌─────────────────────────────────────────────────────────────────────────┐
│ computed: i1              (1 byte, padded to alignment)                  │
├─────────────────────────────────────────────────────────────────────────┤
│ value: T                  (sizeof(T) bytes, aligned)                     │
├─────────────────────────────────────────────────────────────────────────┤
│ code_ptr: ptr             (8 bytes on 64-bit)                            │
├─────────────────────────────────────────────────────────────────────────┤
│ c₁: T₁                    (sizeof(T₁) bytes, aligned)                    │
├─────────────────────────────────────────────────────────────────────────┤
│ ...                                                                      │
├─────────────────────────────────────────────────────────────────────────┤
│ cₘ: Tₘ                    (sizeof(Tₘ) bytes, aligned)                    │
└─────────────────────────────────────────────────────────────────────────┘

Field Indices:
  [0] = computed flag
  [1] = memoized value
  [2] = code pointer (thunk function)
  [3..N+2] = captured values
```

### 3.2 Field Semantics

| Field | Type | Initial Value | Purpose |
|-------|------|---------------|---------|
| `computed` | `i1` | `false` | Tracks whether thunk has been evaluated |
| `value` | `T` | `undef` | Stores result after first evaluation |
| `code_ptr` | `ptr` | `&thunk_fn` | Address of thunk implementation |
| `captures` | `T₁, ..., Tₘ` | captured values | Environment for thunk execution |

### 3.3 Size Formula

For a lazy value with element type `T` and `N` captures with types `T₁, ..., Tₙ`:

```
size(Lazy<T, [T₁...Tₙ]>) = align(1) + sizeof(T) + sizeof(ptr) + Σᵢ sizeof(Tᵢ)
                         = 1 + padding + sizeof(T) + 8 + Σᵢ sizeof(Tᵢ)
```

Typical sizes on 64-bit platforms:
- `lazy 42` (no captures): 1 + 7 + 8 + 8 = 24 bytes
- `lazy (a + b)` with `a, b: int`: 1 + 7 + 8 + 8 + 8 + 8 = 40 bytes

## 4. Thunk Calling Convention

### 4.1 Struct Pointer Passing

Clef uses the **struct pointer passing** convention for thunks. The thunk receives a pointer to its containing lazy struct and extracts captures itself.

**Thunk Signature**:
```
thunk_fn: (ptr<Lazy<T>>) -> T
```

**Rationale**:
- Uniform signature for all thunks regardless of capture count
- Force implementation is simple: extract code pointer, call with struct pointer
- Thunk extracts its own captures at known offsets

### 4.2 Thunk Implementation

The thunk body:
1. Receives pointer to lazy struct as `%arg0`
2. Extracts captures from indices `[3..N+2]`
3. Executes the deferred computation
4. Returns the result

```mlir
llvm.func @thunk_example(%lazy_ptr: !llvm.ptr) -> i64 {
    // Extract capture at index 3
    %cap0 = llvm.getelementptr %lazy_ptr[0, 3] : ... -> !llvm.ptr
    %a = llvm.load %cap0 : !llvm.ptr -> i64
    
    // Extract capture at index 4
    %cap1 = llvm.getelementptr %lazy_ptr[0, 4] : ... -> !llvm.ptr
    %b = llvm.load %cap1 : !llvm.ptr -> i64
    
    // Compute result
    %result = arith.addi %a, %b : i64
    llvm.return %result : i64
}
```

> **NORMATIVE**: Lazy thunks SHALL use `llvm.func` regardless of capture count. Even a lazy thunk with no captures (e.g., `lazy 42`) must be defined as `llvm.func` because its address is taken via `llvm.mlir.addressof` and stored in the lazy struct. The `llvm.mlir.addressof` operation requires the target to be `llvm.func`, `llvm.mlir.global`, or `llvm.mlir.alias` - it cannot reference a `func.func`.

### 4.3 Alternative Considered: Parameter Passing

An alternative convention passes captures as function parameters:

```
thunk_fn: (cap₁, cap₂, ..., capₙ) -> T
```

This was rejected because:
- Force implementation must know capture types at each call site
- Different thunks have different signatures
- Type complexity at forcing increases with capture count

## 5. Capture Analysis

### 5.1 Binding Classification

Capture analysis for lazy values must distinguish:

| Binding Kind | Location | Capture? | Reference Mode |
|--------------|----------|----------|----------------|
| Function parameter | Enclosing function | Yes | By value (immutable) or by reference (mutable) |
| Local let binding | Enclosing scope | Yes | By value or by reference |
| Module-level binding | Global | **No** | Direct address reference |
| Intrinsic | Compiler | **No** | Inline expansion |

**Critical Rule**: Module-level bindings are NOT captured. They have stable addresses and are referenced directly in the generated code.

### 5.2 IsModuleLevel Tracking

CCS (Clef Compiler Service) tracks binding scope in `ResolvedBinding`:

```fsharp
type ResolvedBinding = {
    QualifiedName: string
    Type: NativeType
    IsMutable: bool
    NodeId: NodeId option
    IsModuleLevel: bool  // True if defined at module scope
    // ...
}
```

During capture analysis:

```fsharp
let computeCaptures env bodyNodeId excludeNames =
    bodyVarRefs
    |> List.choose (fun name ->
        match tryLookupBinding name env with
        | Some binding ->
            if binding.IsModuleLevel then
                None  // Not captured - reference by address
            else
                Some { Name = name; Type = binding.Type; ... }
        | None -> None)
```

### 5.3 Example: Correct vs Incorrect Capture

```fsharp
// Module level
let sideEffect msg = Console.writeln msg

// Function with lazy returning body
let lazyAdd a b = lazy (sideEffect "Computing..."; a + b)
```

**Correct captures for `lazyAdd`'s lazy body**: `[a, b]`

**Incorrect (bug)**: `[a, b, sideEffect]`

The `sideEffect` function is module-level; it should be called via its global address, not captured.

## 6. LazyExpr in the PSG

### 6.1 SemanticKind.LazyExpr

```fsharp
type SemanticKind =
    // ...
    | LazyExpr of bodyNodeId: NodeId * captures: CaptureInfo list
```

The PSG node contains:
- `bodyNodeId`: Reference to the thunk Lambda node
- `captures`: Pre-computed capture list (same as thunk's captures)

### 6.2 Lambda with LazyThunk Context

The thunk is represented as a Lambda with special context:

```fsharp
type LambdaContext =
    | RegularClosure
    | LazyThunk
    // ...

SemanticKind.Lambda(
    parameters = [("_unit", UnitType, NodeId -1)],
    bodyNodeId = <computation body>,
    captures = [...],
    enclosingFunction = ...,
    context = LambdaContext.LazyThunk
)
```

## 7. Coeffect Model

### 7.1 LazyLayout Coeffect

SSA assignment computes `LazyLayout` for each lazy expression:

```fsharp
type LazyLayout = {
    LazyNodeId: NodeId              // The LazyExpr node
    CaptureCount: int               // Number of captures
    Captures: CaptureSlot list      // Reuses CaptureSlot from closures
    LazyStructType: MLIRType        // { i1, T, ptr, cap₀, cap₁, ... }
    ElementType: MLIRType           // T (for force operations)
    
    // SSA identifiers for construction
    FalseConstSSA: SSA              // computed flag = false
    UndefSSA: SSA                   // undef lazy struct
    WithComputedSSA: SSA            // insertvalue computed at [0]
    CodeAddrSSA: SSA                // addressof code_ptr
    WithCodePtrSSA: SSA             // insertvalue code_ptr at [2]
    CaptureInsertSSAs: SSA list     // insertvalue for each capture at [3..N+2]
    LazyResultSSA: SSA              // final result
}
```

### 7.2 SSA Cost Formula

For a lazy expression with `N` captures: `5 + N` SSAs

| Operation | SSA Count |
|-----------|-----------|
| `false` constant | 1 |
| `undef` struct | 1 |
| insert computed flag | 1 |
| addressof code_ptr | 1 |
| insert code_ptr | 1 |
| insert captures | N |

### 7.3 ClosureLayout for Thunk

The thunk Lambda uses `ClosureLayout` with `LazyThunk` context:

```fsharp
type ClosureLayout = {
    // ...
    Context: LambdaContext  // LazyThunk for lazy thunks
    LazyStructType: MLIRType option  // Full lazy struct type for capture extraction
}
```

When `Context = LazyThunk`:
- `closureExtractionBaseIndex` returns `3` (captures start at index 3)
- `closureLoadStructType` returns the full lazy struct type

## 8. MLIR Generation

### 8.1 Lazy Value Creation

```mlir
// lazy (a + b) where a, b are captured int values

// Step 1: Constants and undef
%false = arith.constant 0 : i1
%undef = llvm.mlir.undef : !llvm.struct<(i1, i64, !llvm.ptr, i64, i64)>

// Step 2: Insert computed flag at [0]
%v1 = llvm.insertvalue %false, %undef[0] : !llvm.struct<...>

// Step 3: Insert code pointer at [2]
%code_addr = llvm.mlir.addressof @thunk_lazyAdd_body : !llvm.ptr
%v2 = llvm.insertvalue %code_addr, %v1[2] : !llvm.struct<...>

// Step 4: Insert captures at [3], [4], ...
%v3 = llvm.insertvalue %a, %v2[3] : !llvm.struct<...>
%v4 = llvm.insertvalue %b, %v3[4] : !llvm.struct<...>

// %v4 is the complete lazy value
```

### 8.2 Force Operation

```mlir
// Lazy.force lazy_val

// Extract computed flag
%computed = llvm.extractvalue %lazy_val[0] : !llvm.struct<...> -> i1

// Branch based on computed
llvm.cond_br %computed, ^already_computed, ^need_compute

^need_compute:
    // Extract code pointer
    %code_ptr = llvm.extractvalue %lazy_val[2] : !llvm.struct<...> -> !llvm.ptr
    
    // Get pointer to lazy struct (struct pointer passing convention)
    %lazy_ptr = llvm.alloca 1 x !llvm.struct<...> : ... -> !llvm.ptr
    llvm.store %lazy_val, %lazy_ptr : ...
    
    // Call thunk
    %result = llvm.call %code_ptr(%lazy_ptr) : (!llvm.ptr) -> i64
    
    // For memoizing: store result and set computed = true
    // (Currently: pure thunk semantics, no memoization)
    
    llvm.br ^done(%result : i64)

^already_computed:
    %cached = llvm.extractvalue %lazy_val[1] : !llvm.struct<...> -> i64
    llvm.br ^done(%cached : i64)

^done(%value: i64):
    // %value is the forced result
```

## 9. Memoization Strategy

### 9.1 Current Implementation: Pure Thunk Semantics

The initial Clef implementation provides **pure thunk semantics**:
- Forcing always executes the computation
- No memoization state is updated
- Correct for pure computations (same result each time)

```fsharp
let expensive = lazy (printfn "Computing..."; 42)
Lazy.force expensive  // Prints, returns 42
Lazy.force expensive  // Prints again, returns 42
```

### 9.2 Future: Memoizing Semantics

True memoization requires:
1. Mutation of the `computed` flag
2. Storage of result in `value` slot
3. Thread synchronization (for concurrent access)

This will be implemented with arena-based memory:
- Lazy struct allocated in arena (stable address)
- Compare-and-swap for thread-safe memoization
- Memory barrier for cross-thread visibility

### 9.3 Thread Safety Considerations

For memoizing lazy values with concurrent access:

| Scenario | Behavior |
|----------|----------|
| Single force, single thread | Simple memoization |
| Multiple forces, single thread | Return cached value |
| Concurrent forces, multiple threads | CAS on computed flag, one wins |
| Concurrent reads after memo | Memory barrier ensures visibility |

## 10. Type System Integration

### 10.1 Lazy Type Constructor

```fsharp
type NativeType =
    // ...
    | TLazy of elementType: NativeType
```

At the source level:
```fsharp
let x: Lazy<int> = lazy 42
```

At the native level, the actual struct type includes captures:
```fsharp
// No captures
TStruct [TInt I1; TInt I64; TPtr]

// With captures [a: int; b: int]
TStruct [TInt I1; TInt I64; TPtr; TInt I64; TInt I64]
```

### 10.2 Lazy.create and Lazy.force

The `Lazy` module provides two intrinsic operations:

```fsharp
module Lazy =
    /// Create a lazy value from a thunk
    val create: (unit -> 'T) -> Lazy<'T>
    
    /// Force evaluation of a lazy value
    val force: Lazy<'T> -> 'T
```

The `lazy expr` syntax desugars to:
```fsharp
Lazy.create (fun () -> expr)
```

## 11. Normative Requirements

1. **Flat Representation**: Lazy values SHALL use flat closure representation with captures inlined
2. **Struct Layout**: Field order SHALL be: computed, value, code_ptr, captures
3. **Capture Indices**: Captures SHALL begin at index 3
4. **Module-Level Exclusion**: Module-level bindings SHALL NOT be captured
5. **Thunk Convention**: Thunks SHALL receive pointer to containing lazy struct
6. **No Heap**: Lazy values SHALL NOT be allocated on GC-managed heap
7. **Pure Thunks Initially**: Initial implementation SHALL use pure thunk semantics (no memoization)

## 12. Implementation in CCS/Firefly Pipeline

### 12.1 CCS Phase

1. **checkLazy** in Coordinator.fs:
   - Checks the lazy body expression
   - Computes captures via `computeCaptures`
   - Creates `SemanticKind.Lambda` with `LambdaContext.LazyThunk`
   - Creates `SemanticKind.LazyExpr` with captures list

2. **computeCaptures** in Applications.fs:
   - Collects VarRefs in body
   - Excludes parameter names
   - Excludes module-level bindings (`IsModuleLevel = true`)
   - Returns `CaptureInfo list`

### 12.2 Alex Preprocessing Phase

1. **SSAAssignment**:
   - Computes `LazyLayout` for LazyExpr nodes
   - Computes `ClosureLayout` for thunk Lambda nodes
   - Assigns SSAs for all construction operations

### 12.3 Witness Phase

1. **LazyWitness**:
   - Observes `LazyLayout` coeffect
   - Emits struct construction operations
   - Emits thunk function definition

2. **BindingWitness**:
   - Handles functions returning lazy values
   - Uses `getActualFunctionReturnType` for correct lazy struct type

## References

- Tofte, M., & Talpin, J.-P. (1997). *Region-Based Memory Management*. Information and Computation.
- Elsman, M. (2021). *Programming with Regions in the MLKit*. IT University of Copenhagen.
- Appel, A. W. (1992). *Compiling with Continuations*. Cambridge University Press.
- Shao, Z., & Appel, A. W. (1994). *Space-Efficient Closure Representations*. LFP '94.
- Peyton Jones, S. L. (1987). *The Implementation of Functional Programming Languages*. Prentice Hall. (Chapter on lazy evaluation)
