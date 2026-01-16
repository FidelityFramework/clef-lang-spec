# Closure Representation in F# Native

> **Normative specification for closure memory layout and capture semantics in fsnative compilation.**

## 1. Overview

F# Native uses **MLKit-style flat closures** for function values that capture variables from their enclosing scope. This chapter specifies the memory representation, capture semantics, and type system integration.

## 2. Academic Foundation

### 2.1 MLKit and Region-Based Memory Management

The closure representation in F# Native draws from the MLKit compiler for Standard ML, developed by Mads Tofte and others at the University of Copenhagen.

**Key Papers**:
- Tofte, M., & Talpin, J.-P. (1994). *Implementation of the Typed Call-by-Value λ-calculus using a Stack of Regions*
- Tofte, M., Birkedal, L., Elsman, M., & Hallenberg, N. (2004). *A Retrospective on Region-Based Memory Management*
- Elsman, M. (2003). *Garbage Collection Safety for Region-Based Memory Management*

**Core Insight**: All runtime values, including function closures, are allocated in regions. Regions are stack-allocated with bump-pointer allocation and wholesale deallocation. This eliminates GC for deterministic memory management.

### 2.2 Flat vs Linked Closure Representations

The seminal work on closure representations is:
- Shao, Z., & Appel, A. W. (1994). *Space-Efficient Closure Representations* (ACM SIGPLAN LISP and Functional Programming)

**Flat Closures**:
```
┌─────────────────────┬─────────────────────────────────┐
│ code_ptr (8 bytes)  │ captured values (inline)        │
└─────────────────────┴─────────────────────────────────┘
```
- All free variables copied directly into closure
- O(1) access to any captured variable
- Safe for space (no chains preventing GC)
- Higher creation cost if many captures

**Linked Closures**:
```
┌─────────────────────┬─────────────────────────────────┐
│ code_ptr (8 bytes)  │ env_ptr → outer environment     │
└─────────────────────┴─────────────────────────────────┘
```
- Only store pointer to enclosing environment
- O(depth) access via chain traversal
- NOT safe for space (keeps outer scopes alive)
- Lower creation cost (just copy one pointer)

**Space Safety** (Appel, 1992): A closure representation is *safe for space* if garbage collection can reclaim memory proportional to what a reference-counting collector would reclaim. Linked closures fail this property because they keep entire enclosing environments alive even when only some variables are used.

### 2.3 Why F# Native Chooses Flat Closures

1. **No GC Runtime**: Space safety becomes structural rather than runtime property
2. **Region Allocation**: Closures live in regions with deterministic lifetime
3. **Cache Efficiency**: Contiguous memory access for all captures
4. **Simplicity**: Single indirection for invocation

## 3. Memory Layout Specification

### 3.1 Closure Structure

A closure in F# Native is a struct containing:

```
Closure<(A₁, ..., Aₙ) -> R, Env>
┌─────────────────────────────────────────────────────────┐
│ code_ptr: ptr<fn(Env*, A₁, ..., Aₙ) -> R>  (8 bytes)   │
├─────────────────────────────────────────────────────────┤
│ env: Env                                    (variable)  │
└─────────────────────────────────────────────────────────┘
```

Where `Env` is a struct containing the captured variables:

```
Env (for captures [c₁: T₁, ..., cₘ: Tₘ])
┌─────────────────────────────────────────────────────────┐
│ c₁: T₁  (sizeof(T₁) bytes, aligned)                     │
├─────────────────────────────────────────────────────────┤
│ c₂: T₂  (sizeof(T₂) bytes, aligned)                     │
├─────────────────────────────────────────────────────────┤
│ ...                                                     │
├─────────────────────────────────────────────────────────┤
│ cₘ: Tₘ  (sizeof(Tₘ) bytes, aligned)                     │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Capture Semantics

Captures are classified by mutability:

| Variable Kind | Capture Mode | Env Entry Type | Semantics |
|---------------|--------------|----------------|-----------|
| Immutable binding | By Value | `T` | Copy value into env |
| Mutable binding | By Reference | `ptr<T>` | Store pointer to stack slot |
| Ref cell | By Value | `ref<T>` | Copy ref cell pointer |

**Example**:
```fsharp
let makeCounter (start: int) =
    let mutable count = start    // Mutable - captured by reference
    fun () ->
        count <- count + 1
        count
```

The closure captures `count` by reference:
```
Env = { count_ptr: ptr<int> }
```

The mutable `count` remains on the stack of `makeCounter`'s activation frame. The closure stores a pointer to this location.

### 3.3 Allocation Strategy

Closures are allocated:

1. **On Stack**: When lifetime is bounded to enclosing scope
2. **In Region**: When escaping scope but within region lifetime
3. **Never Heap**: No GC-managed heap allocation

Small closures (≤64 bytes total) fit in a single cache line for efficient invocation.

## 4. Type System Integration

### 4.1 Function Type Representation

```
Type ::= ...
       | TFun(argTypes: Type list, retType: Type)
       | TClosure(argTypes: Type list, retType: Type, envType: Type)
```

At the syntax level, users write function types:
```fsharp
let f: int -> int -> int = fun x y -> x + y
```

At the native level, FNCS resolves this to either:
- `TFun([int; int], int)` - direct function, no captures
- `TClosure([int; int], int, Env)` - closure with environment

### 4.2 Capture Analysis in FNCS

During type checking, FNCS performs capture analysis:

```fsharp
type CaptureInfo = {
    Name: string              // Variable name
    SourceNodeId: NodeId      // PSG node where variable is defined
    Type: NativeType          // Type of captured variable
    IsMutable: bool           // Determines capture mode
}

type LambdaInfo = {
    Parameters: (string * Type) list
    Body: NodeId
    Captures: CaptureInfo list  // Computed during scope analysis
}
```

The `Captures` list is part of `SemanticKind.Lambda` in the PSG.

### 4.3 Escape Analysis

A mutable binding that is captured creates a lifetime constraint:

```fsharp
let makeCounter start =
    let mutable count = start  // count's lifetime must exceed closure's
    fun () -> count <- count + 1; count
```

FNCS enforces: **The stack frame containing `count` must outlive all uses of the closure.**

This is verified via coeffect analysis. If the closure escapes to a longer-lived scope, a compile-time error is raised.

## 5. MLIR Representation

### 5.1 Closure Type

```mlir
!fidelity.closure<(i32, i32) -> i32, !llvm.struct<(ptr<i32>)>>
```

Breaking this down:
- `(i32, i32) -> i32` - function signature
- `!llvm.struct<(ptr<i32>)>` - environment type (captures by ref)

### 5.2 Closure Creation

```mlir
// Allocate environment
%env = llvm.alloca 1 x !llvm.struct<(ptr<i32>)>

// Store captures
%slot = llvm.getelementptr %env[0, 0] : ... -> !llvm.ptr
llvm.store %count_ptr, %slot : !llvm.ptr

// Build closure struct
%closure.1 = llvm.insertvalue %undef[0], %code_ptr : !fidelity.closure<...>
%closure = llvm.insertvalue %closure.1[1], %env : !fidelity.closure<...>
```

### 5.3 Closure Invocation

```mlir
// Extract components
%code_ptr = llvm.extractvalue %closure[0] : !fidelity.closure<...> -> !llvm.ptr
%env_ptr = llvm.extractvalue %closure[1] : !fidelity.closure<...> -> !llvm.ptr

// Call with environment as first argument
%result = llvm.call %code_ptr(%env_ptr, %arg1, %arg2) : ...
```

## 6. Optimization Opportunities

### 6.1 Closure Inlining

When a closure is immediately applied and not stored, inline the body:
```fsharp
(fun x -> x + 1) 5  // Becomes: 5 + 1
```

### 6.2 Known-Function Optimization

When the closure's code pointer is statically known, use direct call:
```fsharp
let f = fun x -> x + 1
f 5  // Direct call to lambda body, no indirection
```

### 6.3 Environment Flattening

When multiple closures share captures, consider:
- Shared environment struct (with appropriate lifetime)
- Copy-on-capture if lifetimes differ

## 7. Relation to Standard ML Implementations

### 7.1 MLKit

MLKit pioneered region-based memory management for Standard ML. F# Native follows MLKit's principles:
- All values (including closures) allocated in regions
- Flat closure representation
- Compile-time lifetime analysis

### 7.2 SML/NJ

Standard ML of New Jersey (Appel's implementation) explored multiple closure strategies:
- Flat closures (default)
- Linked closures (for deeply nested functions)
- Hybrid strategies

F# Native uses exclusively flat closures for simplicity and predictability.

### 7.3 OCaml

OCaml uses a uniform representation where closures are tagged blocks. F# Native diverges:
- No uniform tagging (types known statically)
- No GC heap (regions instead)
- Struct-based layout (no boxing)

## 8. Implementation in FNCS/Firefly Pipeline

### 8.1 FNCS Phase

FNCS constructs the PSG with complete lambda information:
- `SemanticKind.Lambda(parameters, body, captures)`
- Captures computed during scope analysis
- Mutability tracked in `CaptureInfo.IsMutable`

### 8.2 Alex Preprocessing Phase

Two closure-related nanopasses run in sequence:

1. **CaptureIdentification** (before SSAAssignment):
   - Identifies all Lambda nodes with captures
   - Tags nodes with `HasClosureCapture` coeffect
   - Records capture list per Lambda

2. **ClosureLayout** (after SSAAssignment):
   - Computes environment struct layout
   - Assigns SSAs for env allocation, stores
   - Records `ClosureCoeffect` with full layout info

### 8.3 Witness Phase

`LambdaWitness` observes:
- If `ClosureCoeffect` present: emit flat closure struct
- If no captures: emit simple function pointer

No computation in witnesses - all layout pre-computed.

## 9. Normative Requirements

1. **Flat Representation**: All closures SHALL use flat environment representation
2. **Capture Mode**: Mutable bindings SHALL be captured by reference; immutable by value
3. **Escape Safety**: Closures capturing mutable bindings SHALL NOT escape the binding's scope
4. **No Heap**: Closures SHALL NOT be allocated on GC-managed heap
5. **Cache Alignment**: Small closures (≤64 bytes) SHOULD be aligned to cache lines

## References

- Appel, A. W. (1992). *Compiling with Continuations*. Cambridge University Press.
- Shao, Z., & Appel, A. W. (1994). *Space-Efficient Closure Representations*. LFP '94.
- Tofte, M., & Talpin, J.-P. (1997). *Region-Based Memory Management*. Information and Computation.
- Elsman, M. (2003). *Garbage Collection Safety for Region-Based Memory Management*. TLDI '03.
- Perconti, J. T., & Ahmed, A. (2019). *Closure Conversion Is Safe for Space*. ICFP '19.
