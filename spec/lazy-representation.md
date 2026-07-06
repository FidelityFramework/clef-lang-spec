---
title: "Lazy Value Representation"
weight: 320
category: Representation
status: normative
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
│ code_ptr: ptr             (1 platform word)                             │
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
                         = 1 + padding + sizeof(T) + sizeof(ptr) + Σᵢ sizeof(Tᵢ)
```

`sizeof(ptr)` is the platform word: 4 bytes on `thumbv8m`/Cortex-M33, 8 bytes on x86-64. The byte totals below are worked for both. Taking `int` as a 64-bit value for the x86-64 column:

- `lazy 42` (no captures):
  - x86-64: 1 + 7 + 8 + 8 = 24 bytes
  - thumbv8m (M33), with `int` as a 4-byte value: 1 + 3 + 4 + 4 = 12 bytes
- `lazy (a + b)` with `a, b: int`:
  - x86-64: 1 + 7 + 8 + 8 + 8 + 8 = 40 bytes
  - thumbv8m (M33): 1 + 3 + 4 + 4 + 4 + 4 = 20 bytes

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
1. Receives the lazy struct's environment as `%arg0`
2. Extracts captures from indices `[3..N+2]`
3. Executes the deferred computation
4. Returns the result

The middle end emits the thunk as a portable `func.func` and reads captures through a `memref`, exactly as the flat closure does (see [Flat Closure Pattern and Deferred Resolution](backend-lowering-architecture.md#4-flat-closure-pattern-and-deferred-resolution)). Nothing in the middle-end form names a target:

```mlir
// Middle end — portable dialects only.
func.func private @thunk_example(%env: memref<?xi64>) -> i64 {
    // Extract capture at index 3 — portable memref access, no ABI committed.
    %c3 = arith.constant 3 : index
    %a = memref.load %env[%c3] : memref<?xi64>

    // Extract capture at index 4.
    %c4 = arith.constant 4 : index
    %b = memref.load %env[%c4] : memref<?xi64>

    // Compute result.
    %result = arith.addi %a, %b : i64
    func.return %result : i64
}
```

A thunk with no captures (e.g., `lazy 42`) is still a `func.func`; its body reads nothing from the environment. The thunk's address is stored in the lazy struct as *data*, which has no portable operation. The middle end carries that conversion as a `builtin.unrealized_conversion_cast` (`func_type → index`) and defers the commitment to the backend leg, the same mechanism the flat closure uses for a function address (backend §4.2).

The `llvm.func` / `llvm.mlir.addressof` form is one backend leg's realization of this thunk, not what the middle end emits. On the LLVM leg the closure-cast pass resolves the deferred casts into the target's pointer representation: the `func_type → index` cast becomes `llvm.ptrtoint`, and the `memref` capture read becomes a `getelementptr` + `load` in the target ABI. A different leg (CIRCT, SPIR-V, WebAssembly) resolves the same middle-end IR its own way.

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

The [PSG node](program-semantic-graph.md) contains:
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

The middle end emits lazy construction over `memref`, with the thunk address carried as a deferred `func_type → index` cast. The struct is a `memref` of the lazy layout; field writes are `memref.store` at the fixed indices. Nothing here commits a target ABI.

### 8.1 Lazy Value Creation

```mlir
// lazy (a + b) where a, b are captured int values.
// Middle end — portable dialects only.

// Step 1: Allocate the lazy struct's storage per its lifetime class (§9).
//         Scope-bounded here, so memref.alloca; a program-lifetime lazy
//         would be a memref.global instead.
%lazy = memref.alloca() : memref<5xi64>

// Step 2: Write computed flag = false at [0].
%c0 = arith.constant 0 : index
%false = arith.constant 0 : i64
memref.store %false, %lazy[%c0] : memref<5xi64>

// Step 3: Write code pointer at [2].
//         The thunk address is data, so it is carried as a deferred cast.
%code = builtin.unrealized_conversion_cast @thunk_lazyAdd_body
      : (memref<?xi64>) -> i64 to i64
%c2 = arith.constant 2 : index
memref.store %code, %lazy[%c2] : memref<5xi64>

// Step 4: Write captures at [3], [4], ...
%c3 = arith.constant 3 : index
memref.store %a, %lazy[%c3] : memref<5xi64>
%c4 = arith.constant 4 : index
memref.store %b, %lazy[%c4] : memref<5xi64>

// %lazy is the complete lazy value.
```

The backend leg commits this to its ABI: on the LLVM leg the `memref` becomes an `!llvm.struct` with `insertvalue`/`store` at the same indices, and the deferred `func_type → index` cast becomes `llvm.ptrtoint` of the thunk's `llvm.func` address. That committed form is one leg's realization, not middle-end output.

### 8.2 Force Operation

Force reads the `computed` flag, and either returns the cached value or calls the thunk through its stored code pointer. The middle end emits this over `memref` and `scf`, with the code-pointer call carried as a deferred `index → func_type` cast. Nothing here commits a target ABI.

```mlir
// Lazy.force lazy_val, where %lazy is memref<5xi64> (§8.1).
// Middle end — portable dialects only.

// Read computed flag at [0].
%c0 = arith.constant 0 : index
%flag = memref.load %lazy[%c0] : memref<5xi64>
%zero = arith.constant 0 : i64
%computed = arith.cmpi ne, %flag, %zero : i64

%value = scf.if %computed -> i64 {
    // Already computed: return the cached value at [1].
    %c1 = arith.constant 1 : index
    %cached = memref.load %lazy[%c1] : memref<5xi64>
    scf.yield %cached : i64
} else {
    // Read the code pointer at [2] and realize it as callable.
    %c2 = arith.constant 2 : index
    %code = memref.load %lazy[%c2] : memref<5xi64>
    %fn = builtin.unrealized_conversion_cast %code
        : i64 to (memref<?xi64>) -> i64

    // Struct-pointer-passing convention: pass the environment to the thunk.
    %env = memref.cast %lazy : memref<5xi64> to memref<?xi64>
    %result = func.call_indirect %fn(%env) : (memref<?xi64>) -> i64

    // For memoizing: store %result at [1] and set the flag at [0]
    // (§9; current implementation uses pure thunk semantics, no memoization).
    scf.yield %result : i64
}
// %value is the forced result.
```

The backend leg commits this: on the LLVM leg the `memref` reads become `extractvalue`/`load`, the `index → func_type` cast becomes `llvm.inttoptr`, and `func.call_indirect` becomes an indirect `llvm.call`. A different leg realizes the same middle-end IR its own way.

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

### 9.2 Storage Placement

A lazy value is a flat closure, so its storage is placed by the same four-point lifetime lattice that governs closures (see [Closure Representation §2.3, §3.3](closure-representation.md#23-allocation-strategy)). Escape analysis classifies the lazy value by how long it must live and places it in the storage whose lifetime covers it:

1. **Scope-bounded**: on the stack (`memref.alloca`), reclaimed when the enclosing scope exits.
2. **Region-bounded**: in a [region](memory-regions.md) whose lifetime covers it, when it escapes the scope but lives within a region's lifetime.
3. **Program-lifetime**: in static storage (the [`Sram`](memory-regions.md) region for a mutable lazy value or [`Flash`](memory-regions.md) for an immutable one), emitted as a `memref.global`, when it is constructed once and held for the life of the program with no free.
4. **Dynamic**: on the heap, when its extent is genuinely dynamic.

Escaping the defining scope does not imply the heap. A lazy value returned from a function and held for the program's life has a statically knowable, program-long lifetime and belongs in static storage, in the same sense a fixed-address register or a linker-carved buffer is a global. On a target with no allocator, only the scope-bounded and program-lifetime placements have a home; a lazy value that classifies as dynamic there is a compile-time lifetime error, not a silent heap allocation.

Memoization is a mutation-in-place property that interacts with this placement, because in-place update of the `computed` flag and the `value` slot requires a stable address:

- A **scope-bounded** or **program-lifetime** lazy value already has a stable address (its `alloca` slot or its `memref.global`), so memoization writes directly to `value` at index `[1]` and sets `computed` at `[0]`.
- Under concurrent access to a program-lifetime lazy value, a compare-and-swap on the `computed` flag resolves the race (one force wins), and a memory barrier makes the memoized `value` visible across threads.

The current implementation (§9.1) uses pure thunk semantics and updates no state, so it is insensitive to placement. Memoizing semantics are placement-sensitive precisely because they mutate the struct in place.

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
6. **Lifetime-Driven Placement**: A lazy value's storage SHALL be placed by escape analysis in the storage whose lifetime covers it, per the four-point lattice of [Closure Representation §3.3](closure-representation.md#33-escape-analysis): the stack when scope-bounded, a region when region-bounded, static storage (`memref.global`) when its lifetime is the whole program, and the heap only when its extent is genuinely dynamic. On a target without a heap, a lazy value that classifies as dynamic SHALL be a compile-time lifetime error, not a heap allocation. Lazy values SHALL NOT be placed on a GC-managed heap.
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
