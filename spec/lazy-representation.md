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

A lazy value in Clef is the two-value pair `(thunk, env)` of [Closure Representation §6.3](closure-representation.md): the thunk is a function value (`func.constant @thunk`), and the environment is a flat struct containing:

```
Lazy<T> with captures [c₁: T₁, ..., cₘ: Tₘ]
┌─────────────────────────────────────────────────────────────────────────┐
│ computed: i1              (1 byte, padded to alignment)                  │
├─────────────────────────────────────────────────────────────────────────┤
│ value: T                  (sizeof(T) bytes, aligned)                     │
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
  [2..N+1] = captured values
```

The thunk symbol is never stored in the environment as data: it travels as the function-value half of the pair, and where the force site knows the thunk (a lazy value that does not escape) it is elided altogether and force is a direct call.

### 3.2 Field Semantics

| Field | Type | Initial Value | Purpose |
|-------|------|---------------|---------|
| `computed` | `i1` | `false` | Tracks whether thunk has been evaluated |
| `value` | `T` | `undef` | Stores result after first evaluation |
| `captures` | `T₁, ..., Tₘ` | captured values | Environment for thunk execution |

### 3.3 Size Formula

For a lazy value with element type `T` and `N` captures with types `T₁, ..., Tₙ`:

```
size(Lazy<T, [T₁...Tₙ]>) = align(1) + sizeof(T) + Σᵢ sizeof(Tᵢ)
                         = 1 + padding + sizeof(T) + Σᵢ sizeof(Tᵢ)
```

No platform word is spent on a code pointer; the thunk is the other half of the pair, not a field. The byte totals below are worked for two targets. Taking `int` as a 64-bit value for the x86-64 column:

- `lazy 42` (no captures):
  - x86-64: 1 + 7 + 8 = 16 bytes
  - thumbv8m (M33), with `int` as a 4-byte value: 1 + 3 + 4 = 8 bytes
- `lazy (a + b)` with `a, b: int`:
  - x86-64: 1 + 7 + 8 + 8 + 8 = 32 bytes
  - thumbv8m (M33): 1 + 3 + 4 + 4 + 4 = 16 bytes

## 4. Thunk Calling Convention

### 4.1 Struct Pointer Passing

Clef uses the **environment passing** convention for thunks. The thunk receives its environment — the lazy struct of §3.1 — as its sole environment parameter and extracts captures itself.

**Thunk Signature**:
```
thunk_fn: (memref<Exi8>) -> T      // E = the environment extent of §3.3, a literal at saturation
```

**Rationale**:
- Uniform signature for all thunks regardless of capture count
- Force is one call: `func.call_indirect %thunk(%env)`, or `func.call @thunk(%env)` where the force site knows the thunk
- Thunk extracts its own captures at known offsets

### 4.2 Thunk Implementation

The thunk body:
1. Receives the lazy struct's environment as `%arg0`
2. Extracts captures from indices `[2..N+1]`
3. Executes the deferred computation
4. Returns the result

The middle end emits the thunk as a portable `func.func` and reads captures through a `memref`, exactly as the flat closure does (see [Function Values in the Interior](backend-lowering-architecture.md#4-function-values-in-the-interior)). Nothing in the middle-end form names a target:

```mlir
// Middle end — portable dialects only.
func.func private @thunk_example(%env: memref<?xi64>) -> i64 {
    // Extract capture at index 2 — portable memref access, no ABI committed.
    %c2 = arith.constant 2 : index
    %a = memref.load %env[%c2] : memref<?xi64>

    // Extract capture at index 3.
    %c3 = arith.constant 3 : index
    %b = memref.load %env[%c3] : memref<?xi64>

    // Compute result.
    %result = arith.addi %a, %b : i64
    func.return %result : i64
}
```

A thunk with no captures (e.g., `lazy 42`) is still a `func.func`; its body reads nothing from the environment. The thunk is the function-value half of the lazy pair — `func.constant @thunk` — and is never stored as data; no cast exists in the middle end ([Backend Lowering Architecture §4](backend-lowering-architecture.md)).

The `llvm.func` / `llvm.mlir.addressof` form is one target pathway's realization of this thunk, not what the middle end emits. On the LLVM pathway the standard lowerings do all of it: `func.constant` becomes `llvm.mlir.addressof`, and the `memref` capture read becomes a `getelementptr` + `load` in the target ABI. No cast-resolution pass runs. A different pathway (CIRCT, SPIR-V, WebAssembly) realizes the same middle-end IR its own way.

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
    parameters = [("_unit", UnitType, <unitFormalId>)],
    bodyNodeId = <computation body>,
    captures = [...],
    enclosingFunction = ...,
    context = LambdaContext.LazyThunk
)
```

## 7. Coeffect Model

The [closure pipeline](closure-representation.md#9-compilation-pipeline)
governs lazy representation as well. CCS identifies captures and their source
types. Baker elaboration and saturation settle the storage, layout, initialization
and force relationships before Alex witnesses them. These semantic facts refer
to graph participants; they contain neither MLIR types nor preassigned SSA values.

### 7.1 Settled lazy layout

The settled reading identifies the lazy formation and thunk, the result type,
and the ordered fields for `computed`, `value` and captures. Each field retains
its selected representation, extent, alignment and source identity. Capture
slots also retain their actual formation initializer and value or shared-cell
mode. The environment has no code-pointer field.

The graph relates that layout to the backing allocation, its covering lifetime,
and the force sites. Moving or forwarding a lazy descriptor preserves the actual
memoization state and every captured storage reference. Equal thunk code does
not identify a unique lazy instance.

### 7.2 Witness operands

Alex pulls the settled reading through its actual Huet position. Its
Element/Pattern/Witness composition creates typed SSA operands as it emits the
admitted operations. The thunk and environment remain separate values; a known
force site may elide the thunk operand under §8's convention.

There is no language-level SSA cost formula. Emission bookkeeping, temporary
operands and selected physical operations do not establish layout or lifetime
facts. Missing semantic premises must be settled by their owning graph pass.

### 7.3 Thunk and force obligations

The `LazyThunk` context preserves the deferred computation boundary. The thunk
receives the actual environment instance, and capture access uses the settled
fields beginning at logical index 2. The result field is read only after its
initializing force has completed. The computed flag and cached value belong to
that same lazy instance.

The graph retains the participants needed to establish §11's memoization and
single-forcer rules. A finite field list bounds direct layout checks; it does
not by itself prove the lifetime of captured references or ownership across
threads and actor boundaries. Changes to a formation, capture, force site or
ownership premise invalidate the dependent conclusions before renewed witnessing.

## 8. MLIR Generation

The middle end emits lazy construction as the pair of a thunk function value and a `memref` of the lazy environment. The thunk is created with `func.constant`; environment field writes use `memref.store` at the fixed indices. Nothing here commits a target ABI.

### 8.1 Lazy Value Creation

```mlir
// lazy (a + b) where a, b are captured int values.
// Middle end — portable dialects only.

// Step 1: Allocate the lazy struct's storage per its lifetime class (§9).
//         Scope-bounded here, so memref.alloca; a program-lifetime lazy
//         would be a memref.global instead.
%lazy = memref.alloca() : memref<4xi64>

// Step 2: Write computed flag = false at [0].
%c0 = arith.constant 0 : index
%false = arith.constant 0 : i64
memref.store %false, %lazy[%c0] : memref<4xi64>

// Step 3: Write captures at [2], [3], ...
%c2 = arith.constant 2 : index
memref.store %a, %lazy[%c2] : memref<4xi64>
%c3 = arith.constant 3 : index
memref.store %b, %lazy[%c3] : memref<4xi64>

// Step 4: The thunk is the other half of the pair — a function value, not data.
%thunk = func.constant @thunk_lazyAdd_body : (memref<4xi64>) -> i64

// (%thunk, %lazy) is the complete lazy value.
```

The target pathway commits this to its ABI through its standard lowerings: on the LLVM pathway the `memref` becomes a pointer with `store` at the same indices and `func.constant` becomes `llvm.mlir.addressof`. That committed form is one pathway's realization, not middle-end output, and no cast is involved.

### 8.2 Force Operation

Force reads the `computed` flag, and either returns the cached value or calls the thunk — the function-value half of the pair — with the environment. The middle end emits this over `func`, `memref`, and `scf`. Nothing here commits a target ABI.

```mlir
// Lazy.force lazy_val, where the value is (%thunk, %lazy) and %lazy is memref<4xi64> (§8.1).
// Middle end — portable dialects only.

// Read computed flag at [0].
%c0 = arith.constant 0 : index
%flag = memref.load %lazy[%c0] : memref<4xi64>
%zero = arith.constant 0 : i64
%computed = arith.cmpi ne, %flag, %zero : i64

%value = scf.if %computed -> i64 {
    // Already computed: return the cached value at [1].
    %c1 = arith.constant 1 : index
    %cached = memref.load %lazy[%c1] : memref<4xi64>
    scf.yield %cached : i64
} else {
    // Environment-passing convention: call the thunk with its environment.
    // Where the force site knows the thunk this is func.call @thunk_lazyAdd_body(%lazy).
    %result = func.call_indirect %thunk(%lazy) : (memref<4xi64>) -> i64

    // Store the memoized result at [1] and set the computed flag at [0].
    %c1 = arith.constant 1 : index
    memref.store %result, %lazy[%c1] : memref<4xi64>
    %true = arith.constant 1 : i64
    memref.store %true, %lazy[%c0] : memref<4xi64>
    scf.yield %result : i64
}
// %value is the forced result.
```

The target pathway commits this through its standard lowerings: on the LLVM pathway the `memref` reads become `load`s and `func.call_indirect` becomes an `llvm.call` through the function value. A different pathway realizes the same middle-end IR its own way.

## 9. Memoization Strategy

### 9.1 Memoization Semantics

The first force evaluates the computation and stores its result. Subsequent
forces return the stored result without evaluating the computation again.

```fsharp
let expensive = lazy (printfn "Computing..."; 42)
Lazy.force expensive  // Prints, returns 42
Lazy.force expensive  // Returns the cached 42 without printing
 
```

### 9.2 Storage Placement

A lazy value is a flat closure, so its storage is placed by the same four-point lifetime lattice that governs closures (see [Closure Representation §2.3, §3.3](closure-representation.md#23-allocation-strategy)). Escape analysis classifies the lazy value by how long it must live and places it in the storage whose lifetime covers it:

1. **Scope-bounded**: on the stack (`memref.alloca`), reclaimed when the enclosing scope exits.
2. **Region-bounded**: in a [region](memory-regions.md) whose lifetime covers it, when it escapes the scope but lives within a region's lifetime.
3. **Program-lifetime**: in static storage, emitted as a `memref.global`, when it is constructed once and held for the life of the program with no free: the platform's declared mutable program-lifetime space for a mutable lazy value, or its declared immutable program-lifetime space for one that is never written (on an MCU the [`Sram`](memory-regions.md) and [`Flash`](memory-regions.md) regions; the data and rodata sections on an ELF target; constant memory on a GPU).
4. **Dynamic**: on the heap, when its extent is genuinely dynamic.

Escaping the defining scope does not imply the heap. A lazy value returned from a function and held for the program's life has a statically knowable, program-long lifetime and belongs in static storage, in the same sense a fixed-address register or a linker-carved buffer is a global. On a target with no allocator, only the scope-bounded and program-lifetime placements have a home; a lazy value that classifies as dynamic there is a compile-time lifetime error, not a silent heap allocation.

Memoization is a mutation-in-place property that interacts with this placement, because in-place update of the `computed` flag and the `value` slot requires a stable address:

- A **scope-bounded** or **program-lifetime** lazy value already has a stable address (its `alloca` slot or its `memref.global`), so memoization writes directly to `value` at index `[1]` and sets `computed` at `[0]`.
- Under concurrent access to a program-lifetime lazy value, the write-once obligation is discharged at compile time by the single-forcer discipline of §11: ownership establishes one semantic forcer, so the race is resolved in the proof before it is resolved in the code. Beneath that discharged proof, the target realization is a compare-and-swap on the `computed` flag (one force wins) and a memory barrier that makes the memoized `value` visible across threads.

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
TStruct [TInt I1; TInt I64]

// With captures [a: int; b: int]
TStruct [TInt I1; TInt I64; TInt I64; TInt I64]
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
2. **Struct Layout**: Field order SHALL be: computed, value, captures; no code pointer SHALL be stored in the environment
3. **Capture Indices**: Captures SHALL begin at index 2
4. **Module-Level Exclusion**: Module-level bindings SHALL NOT be captured
5. **Thunk Convention**: Thunks SHALL receive their environment as the sole environment parameter; a lazy value SHALL be the two-value pair `(thunk, env)` of [Closure Representation §6.3](closure-representation.md), with the thunk elided where the force site knows it
6. **Lifetime-Driven Placement**: A lazy value's storage SHALL be placed by escape analysis in the storage whose lifetime covers it, per the four-point lattice of [Closure Representation §3.3](closure-representation.md#33-escape-analysis): the stack when scope-bounded, a region when region-bounded, static storage (`memref.global`) when its lifetime is the whole program, and the heap only when its extent is genuinely dynamic. On a target without a heap, a lazy value that classifies as dynamic SHALL be a compile-time lifetime error, not a heap allocation. Lazy values SHALL NOT be placed on a GC-managed heap.
7. **Memoization**: The first force SHALL evaluate the computation and store its result. Subsequent forces SHALL return the stored result without evaluating the computation again.
8. **Single-Forcer Memoization**: A memoizing lazy value SHALL be forced under a single-forcer discipline: for each lazy value, exactly one semantic forcer performs the transition from unevaluated to computed. A lazy value whose force sites span threads or actor boundaries SHALL carry an ownership obligation discharged at compile time by establishing that single semantic forcer. This discipline keeps the write-once conditions on `computed` at `[0]` and `value` at `[1]` quantifier-free: each slot is written at one statically identified site, so the verification conditions quantify over enumerated structure only ([Closure Representation §11](closure-representation.md#11-proof-extraction-at-closure-sites)).

## 12. Implementation in CCS/Composer Pipeline

### 12.1 CCS Phase

CCS checks the body and result type, records exact lexical capture identities
and modes, and constructs the lazy expression and deferred thunk relationship.
The thunk's own formal parameter and admitted module-level references are
excluded from its lexical captures under §5. Parameters of an enclosing function,
such as `a` and `b` in `lazyAdd`, are captures when the thunk uses them. Their
established values or shared deferred identities are retained without forcing
them; mutable captures retain the original cells. Source checking does not
execute the deferred body.

### 12.2 Baker Elaboration and Saturation

Baker retains the actual formation initializers, thunk and environment
relationship, force sites and memoization protocol. Owning nanopasses settle
the target-dependent fields and storage together with initialization, lifetime
and single-forcer obligations. Recipes carry their participants and provenance
through fan-out/fold-in; a cached fact cannot substitute for missing or changed
premises. Result destinations, when required, participate in that settlement
without moving source effects across their formation or force boundaries.

### 12.3 Witness Phase

Alex observes the admitted lazy formation, thunk, fields and force protocol at
their actual graph occurrences. Patterns compose the corresponding Elements;
the emission accumulator tracks operands separately from the immutable zipper.
Bindings, calls and returned values preserve the settled thunk/environment
carrier and actual state instance. They do not reconstruct a layout from source
syntax or infer an environment from code identity.

Target realization lowers the emitted physical form and preserves or rechecks
its affected properties. Circuit scheduling and target memory realization
belong to that backend boundary, while §11's source semantics remain unchanged.

## References

- Tofte, M., & Talpin, J.-P. (1997). *Region-Based Memory Management*. Information and Computation.
- Elsman, M. (2021). *Programming with Regions in the MLKit*. IT University of Copenhagen.
- Appel, A. W. (1992). *Compiling with Continuations*. Cambridge University Press.
- Shao, Z., & Appel, A. W. (1994). *Space-Efficient Closure Representations*. LFP '94.
- Peyton Jones, S. L. (1987). *The Implementation of Functional Programming Languages*. Prentice Hall. (Chapter on lazy evaluation)
