---
title: "Closure Representation in Clef"
weight: 310
---

> **Status**: Normative
> **Last Updated**: 2026-01-19

## Informative References

> **Commentary**: For accessible explanation of the design rationale, including comparison with other closure representations and the MLKit heritage, see [Gaining Closure](https://clef-lang.com/docs/design/memory/gaining-closure/) in the Clef design documentation.
>
> **Academic Foundation**: This representation follows MLKit-style flat closures. Key references:
> - Shao, Z., & Appel, A. W. (1994). *Space-Efficient Closure Representations*. LFP '94.
> - Tofte, M., & Talpin, J.-P. (1997). *Region-Based Memory Management*. Information and Computation.

---

## 1. Overview

Clef uses **flat closures** for function values that capture variables from their enclosing scope. This chapter specifies the memory representation, capture semantics, and calling conventions.

## 2. Memory Layout Specification

### 2.1 Closure Structure

A closure in Clef is a struct containing a code pointer followed by captured values:

```
Closure with captures [c₁: T₁, ..., cₘ: Tₘ]
┌─────────────────────────────────────────────────────────┐
│ code_ptr: ptr                           (8 bytes)       │
├─────────────────────────────────────────────────────────┤
│ c₁: T₁  (sizeof(T₁) bytes, aligned)                     │
├─────────────────────────────────────────────────────────┤
│ c₂: T₂  (sizeof(T₂) bytes, aligned)                     │
├─────────────────────────────────────────────────────────┤
│ ...                                                     │
├─────────────────────────────────────────────────────────┤
│ cₘ: Tₘ  (sizeof(Tₘ) bytes, aligned)                     │
└─────────────────────────────────────────────────────────┘

Field Indices:
  [0] = code_ptr (function address)
  [1..m] = captured values
```

### 2.2 Capture Semantics

Captures are classified by the mutability of the source binding:

| Variable Kind | Capture Mode | Entry Type | Semantics |
|---------------|--------------|------------|-----------|
| Immutable binding | By Value | `T` | Copy value into closure |
| Mutable binding | By Reference | `ptr<T>` | Store pointer to stack slot |
| Ref cell | By Value | `ref<T>` | Copy ref cell pointer |

### 2.3 Allocation Strategy

Closures are allocated:

1. **On Stack**: When lifetime is bounded to enclosing scope
2. **In Region**: When escaping scope but within region lifetime
3. **Never Heap**: No GC-managed heap allocation

## 3. Calling Convention

### 3.1 Closure Invocation

When invoking a closure, the caller:

1. Extracts `code_ptr` from field [0]
2. Obtains pointer to the closure struct
3. Calls `code_ptr` with closure pointer as first argument, followed by explicit arguments

```
call_closure(closure, arg1, arg2):
    code_ptr = closure[0]
    call code_ptr(&closure, arg1, arg2)
```

### 3.2 Capture Access

The closure body receives a pointer to its containing struct and extracts captures by field index:

```
closure_body(self_ptr, arg1, arg2):
    cap1 = self_ptr[1]    // First capture
    cap2 = self_ptr[2]    // Second capture
    // ... use captures and arguments
```

## 4. Type System Integration

### 4.1 Function Type Representation

```
Type ::= ...
       | TFun(argTypes: Type list, retType: Type)
       | TClosure(argTypes: Type list, retType: Type, captures: CaptureInfo list)
```

At the native level, CCS (Clef Compiler Service) distinguishes:
- `TFun` - Direct function, no captures
- `TClosure` - Closure with captured environment

### 4.2 Capture Analysis

During type checking, CCS computes capture information:

```fsharp
type CaptureInfo = {
    Name: string              // Variable name
    SourceNodeId: NodeId      // PSG node where defined
    Type: NativeType          // Type of captured variable
    IsMutable: bool           // Determines capture mode
}
```

### 4.3 Escape Analysis

A mutable binding that is captured creates a lifetime constraint:

**Invariant**: The stack frame containing a mutable binding SHALL outlive all closures that capture it by reference.

Violation of this invariant produces a compile-time error.

## 5. MLIR Representation

### 5.1 Closure Type

```mlir
!llvm.struct<(ptr, T1, T2, ...)>
```

Where `ptr` is the code pointer and `T1, T2, ...` are capture types.

### 5.2 Closure Creation

```mlir
// Build closure struct with captures
%closure.1 = llvm.insertvalue %undef[0], %code_ptr : !llvm.struct<...>
%closure.2 = llvm.insertvalue %closure.1[1], %cap1 : !llvm.struct<...>
%closure = llvm.insertvalue %closure.2[2], %cap2 : !llvm.struct<...>
```

### 5.3 Closure Invocation

```mlir
// Extract code pointer
%code_ptr = llvm.extractvalue %closure[0] : !llvm.struct<...> -> !llvm.ptr

// Get closure address
%closure_ptr = llvm.alloca ... // or addressof if already allocated

// Indirect call with closure pointer as first argument
%result = llvm.call %code_ptr(%closure_ptr, %arg1, %arg2) : ...
```

## 6. Nested Named Functions vs Escaping Closures

Clef distinguishes two categories of functions that capture variables.

### 6.1 Escaping Closures (Closure Struct Model)

Anonymous lambdas and function values that may escape their defining scope use the flat closure struct:

```fsharp
let makeAdder n =
    fun x -> x + n  // Anonymous lambda, may escape
```

The lambda is a first-class value that can be returned, stored, or passed to higher-order functions. CCS creates a closure struct: `{ code_ptr, n }`.

### 6.2 Nested Named Functions (Parameter-Passing Model)

Named functions defined inside another function that are NOT escaping use parameter-passing:

```fsharp
let sumTo n =
    let rec loop acc i =
        if i > n then acc     // 'n' captured from enclosing scope
        else loop (acc + i) (i + 1)
    loop 0 1
```

Here `loop` is called directly from `sumTo` and never escapes. Captures become additional parameters:

**Generated Signature**:
```
loop: (n: int, acc: int, i: int) -> int
       ↑ capture   ↑ explicit params
```

**Call Site**: `loop(n, 0, 1)`, where capture `n` is passed as first argument.

### 6.3 Classification Criteria

| Criterion | Escaping Closure | Nested Named Function |
|-----------|-----------------|----------------------|
| Definition form | `fun x -> ...` | `let [rec] name ...` |
| PSG parent | Application, Sequential, etc. | Binding node |
| Can escape scope | Yes | No |
| Representation | `{code_ptr, cap₁, ...}` struct | Direct function |
| Capture passing | Via struct extraction | As explicit parameters |
| Allocation | Stack/region for struct | None |

### 6.4 Classification Rule

A Lambda SHALL be classified as a nested named function if and only if:
1. Its `enclosingFunction` is `Some _` (nested)
2. Its parent PSG node is a `Binding` (named definition)

## 7. Implementation in CCS/Firefly Pipeline

### 7.1 CCS Phase

CCS constructs the PSG with complete lambda information:
- `SemanticKind.Lambda(parameters, body, captures)`
- Captures computed during scope analysis
- Mutability tracked in `CaptureInfo.IsMutable`

### 7.2 Alex Preprocessing Phase

1. **CaptureIdentification**: Identifies Lambda nodes with captures, tags with `HasClosureCapture` coeffect
2. **ClosureLayout**: Computes struct layout, assigns SSAs for construction

### 7.3 Witness Phase

`LambdaWitness` observes:
- If `ClosureCoeffect` present: emit flat closure struct
- If no captures: emit simple function pointer

## 8. Normative Requirements

1. **Flat Representation**: Escaping closures SHALL use flat closure representation with captures stored directly in the struct
2. **No Environment Pointer**: Closures SHALL NOT use linked environment chains or `env_ptr` fields
3. **Capture Mode**: Mutable bindings SHALL be captured by reference; immutable by value
4. **Escape Safety**: Closures capturing mutable bindings SHALL NOT escape the binding's scope
5. **No Heap**: Closures SHALL NOT be allocated on GC-managed heap
6. **Cache Alignment**: Small closures (≤64 bytes) SHOULD be aligned to cache lines
7. **Nested Functions**: Named functions defined within another function that do not escape SHALL use parameter-passing for captures
8. **Classification**: A Lambda SHALL be classified as a nested named function iff its enclosing function is present AND its parent PSG node is a Binding
