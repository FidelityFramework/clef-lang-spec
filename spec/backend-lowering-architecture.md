---
title: "Backend Lowering Architecture"
weight: 660
category: Compiler
status: normative
---

> **Status**: Normative
> **Last Updated**: 2026-01-19

## Informative References

> **Commentary**: For accessible explanation of the two-layer model and why certain constructs require backend-specific dialects, see [Why Clef Is A Natural Fit for MLIR](https://clef-lang.com/docs/design/compilation/why-clef-fits-mlir/) in the Clef design documentation.
>
> **For .NET Developers**: Guidance on transitioning from CLR concepts to native compilation is available in the rationale documentation.

---

## 1. Overview

This chapter specifies how Clef lowers high-level constructs to backend-specific representations using MLIR's multi-dialect architecture.

## 2. The Two-Layer Model

Clef uses a two-layer intermediate representation:

```
F# Source → CCS (Clef Compiler Service) → PSG → Alex → MLIR (mixed dialects) → Backend → Native Binary
                                      ↑
                               Portable + Backend-Specific
```

### 2.1 Portable Dialects

Operations that are semantically backend-independent use MLIR's portable dialects:

| Dialect | Purpose | Operations |
|---------|---------|------------|
| `func` | Function structure | `func.func`, `func.call`, `func.return` |
| `cf` | Control flow | `cf.br`, `cf.cond_br`, `cf.switch` |
| `scf` | Structured control | `scf.while`, `scf.for`, `scf.if` |
| `arith` | Arithmetic | `arith.addi`, `arith.constant`, `arith.cmpi` |

These dialects can lower to multiple backends: LLVM, SPIR-V, WebAssembly, custom hardware.

### 2.2 Backend-Specific Dialects

Operations that commit to a specific representation require backend-specific dialects. For LLVM targets:

| Category | Operations | Reason |
|----------|------------|--------|
| Function pointers | `llvm.mlir.addressof`, indirect `llvm.call` | Pointer representation varies by backend |
| Struct manipulation | `llvm.insertvalue`, `llvm.extractvalue`, `llvm.getelementptr` | No portable heterogeneous struct type |
| Memory operations | `llvm.load`, `llvm.store`, `llvm.alloca` | ABI-dependent alignment |

## 3. Dialect Selection Rules

### 3.1 Use Portable Dialects When

- The operation has no representation dependency on the target
- The function is called directly by name (not through a pointer)
- Control flow is structural (branches, loops, conditions)
- Operations are arithmetic (add, compare, etc.)

### 3.2 Use Backend-Specific Dialects When

- Taking a function's address for storage or indirect call
- Manipulating heterogeneous structs (records, closures, unions)
- Operations have ABI implications (memory layout, alignment)
- Platform-specific intrinsics (syscalls, atomics)

## 4. Flat Closure Pattern and Backend Dialects

Clef implements [closures](closure-representation.md), [lazy values](lazy-representation.md), and [sequences](seq-representation.md) using flat closures that store function pointers in structs. This pattern requires backend-specific code.

### 4.1 Why Backend-Specific

The flat closure pattern requires:
1. Taking a function's address: backend-specific operation
2. Storing address in struct: backend-specific struct manipulation
3. Calling through stored pointer: backend-specific indirect call

There is no portable MLIR representation for "pointer to function":

| Backend | Function Pointer Mechanism |
|---------|---------------------------|
| LLVM | `!llvm.ptr` + `llvm.mlir.addressof` |
| SPIR-V | Function tables, `OpFunctionPointer` |
| WebAssembly | Function indices, `call_indirect` |

### 4.2 The Middle-End Encoding and Deferred Resolution

The MiddleEnd (Alex) does not emit the backend-specific form of the previous section. It emits a **target-agnostic encoding** of the flat closure using only portable dialects (`func`, `memref`, `arith`), and defers the backend commitment to a later, per-target pass. A closure is encoded as a `(code_pointer, environment_pointer)` pair, with both pointers carried as `index` values in `memref` rather than as any backend's pointer type.

This is deliberate. Committing the closure representation to `!llvm.ptr` in the middle end would pre-commit every closure, and therefore every lazy value, sequence, and reactive callback that builds on it, to the LLVM backend. Keeping the encoding in portable dialects is what allows the same closure IR to reach LLVM, SPIR-V, WebAssembly, or a hardware backend. The middle end stays LLVM-free so that the choice of backend remains open past the middle end.

The obstacle is that storing a function's address **as data** has no standard MLIR lowering: there is no portable operation that turns a function into an integer-sized value and back. The MiddleEnd represents each such conversion as a `builtin.unrealized_conversion_cast`, which is MLIR's designated mechanism for a type conversion whose realization is deferred. Three net conversions arise:

| Middle-End Conversion | Meaning | Backend Realization (LLVM) |
|---|---|---|
| `func_type → index` | Store a function address as data | `llvm.ptrtoint` |
| `index → func_type` | Recover a function for an indirect call | `llvm.inttoptr` |
| `index → memref` | Reconstruct a captured-environment pointer as a `memref` for capture extraction | `llvm.inttoptr` + descriptor construction |

Each backend resolves these deferred casts into its own pointer representation. For the LLVM backend the resolution runs as a dedicated pass, positioned **after** the standard dialect conversions (`--convert-func-to-llvm`, `--finalize-memref-to-llvm`, and so on) and **before** `--reconcile-unrealized-casts`, because the standard conversions leave the closure casts with intermediate types that reconciliation cannot collapse on its own. The Fidelity `flat-closure-lowering` plugin provides this as `--resolve-closure-casts`. A backend targeting different hardware supplies its own resolution of the same three conversions; the middle-end IR it consumes is identical.

### 4.3 Correct Dialect Usage

```mlir
// Thunk function - LLVM dialect (address will be taken)
llvm.func private @lazy_thunk(%struct_ptr: !llvm.ptr) -> i64 {
    // Struct access - LLVM dialect
    %cap_ptr = llvm.getelementptr %struct_ptr[0, 3] : !llvm.ptr -> !llvm.ptr
    %cap = llvm.load %cap_ptr : !llvm.ptr -> i64

    // Arithmetic - portable dialect (valid in llvm.func body)
    %result = arith.addi %cap, %cap : i64

    llvm.return %result : i64
}

// Entry point - func dialect (called by name)
func.func @main() -> i32 {
    // ...
    func.return %ret : i32
}
```

### 4.4 Dialect Mixing Rules

MLIR allows mixing portable and backend-specific operations within function bodies:

1. **`func.call` inside `llvm.func`**: Valid. An `llvm.func` can call a `func.func` using `func.call`.
2. **`llvm.call` target restriction**: `llvm.call` can only call functions defined as `llvm.func`.
3. **Portable ops in any function**: `arith.*`, `cf.*`, `scf.*` operations work in both `func.func` and `llvm.func` bodies.

## 5. Entry Point Example

In freestanding mode, `_start` is an `llvm.func` (its address may be taken by the linker), but it calls `main` which is a `func.func`:

```mlir
llvm.func @_start() -> i32 {
    // Read argc/argv via inline asm (LLVM-specific)
    %argc = llvm.inline_asm "mov (%rsp), $0", "=r" : () -> i64
    %argv = llvm.inline_asm "lea 8(%rsp), $0", "=r" : () -> !llvm.ptr
    
    // Call main - uses func.call since main is func.func
    %result = func.call @main(%argc, %argv) : (i64, !llvm.ptr) -> i64
    
    // Exit syscall (LLVM-specific)
    llvm.inline_asm has_side_effects "syscall", "..." %result : ...
    llvm.unreachable
}
```

## 6. Platform Configuration

The project file specifies the target platform:

```toml
[compilation]
target = "x86_64-unknown-linux-gnu"

[platform]
word_size = 64
endianness = "little"
```

This configuration flows through:
1. `Fidelity.Platform` selects the appropriate `PlatformDescriptor`
2. CCS uses platform info for [type layouts](type-representation-architecture.md) and intrinsic typing
3. Alex selects appropriate backend dialect usage
4. Backend receives correctly-lowered IR

## 7. Normative Requirements

1. **Portable Operations SHALL use portable dialects**: Control flow, arithmetic, and directly-called functions use `func`, `cf`, `scf`, `arith` dialects
2. **Address-taken functions SHALL use backend dialects**: Functions whose address is taken use the backend's function definition (e.g., `llvm.func`)
3. **Struct operations SHALL use backend dialects**: Record, [union](discriminated-union-representation.md), and closure struct manipulation uses backend-specific operations
4. **`llvm.call` target restriction**: `llvm.call` SHALL only call functions defined as `llvm.func`; to call a `func.func` from `llvm.func`, use `func.call`
5. **LLVM-free middle end**: The MiddleEnd SHALL encode flat closures using portable dialects and SHALL NOT commit to a backend's pointer type; the conversion of a function address to and from data SHALL be represented as `builtin.unrealized_conversion_cast` and resolved per target
6. **Deferred cast resolution**: A backend SHALL resolve the `func_type ↔ index` and `index → memref` closure casts into its own pointer representation; for the LLVM backend this resolution SHALL run after standard dialect conversions and before `--reconcile-unrealized-casts`
7. **Platform configuration flow**: `fidproj` platform settings SHALL inform all lowering decisions
