# Backend Lowering Architecture

> **Status**: Normative
> **Last Updated**: 2026-01-19

## Informative References

> **Commentary**: For accessible explanation of the two-layer model and why certain constructs require backend-specific dialects, see [Why F# Is A Natural Fit for MLIR](https://speakez.com/blog/why-fsharp-is-a-natural-fit-for-mlir/) on the SpeakEZ blog.
>
> **For .NET Developers**: Guidance on transitioning from CLR concepts to native compilation is available in the rationale documentation.

---

## 1. Overview

This chapter specifies how Clef lowers high-level constructs to backend-specific representations using MLIR's multi-dialect architecture.

## 2. The Two-Layer Model

Clef uses a two-layer intermediate representation:

```
F# Source → CCS → PSG → Alex → MLIR (mixed dialects) → Backend → Native Binary
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

Clef implements closures, lazy values, and sequences using flat closures that store function pointers in structs. This pattern requires backend-specific code.

### 4.1 Why Backend-Specific

The flat closure pattern requires:
1. Taking a function's address — backend-specific operation
2. Storing address in struct — backend-specific struct manipulation
3. Calling through stored pointer — backend-specific indirect call

There is no portable MLIR representation for "pointer to function":

| Backend | Function Pointer Mechanism |
|---------|---------------------------|
| LLVM | `!llvm.ptr` + `llvm.mlir.addressof` |
| SPIR-V | Function tables, `OpFunctionPointer` |
| WebAssembly | Function indices, `call_indirect` |

### 4.2 Correct Dialect Usage

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

### 4.3 Dialect Mixing Rules

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
2. CCS uses platform info for type layouts and intrinsic typing
3. Alex selects appropriate backend dialect usage
4. Backend receives correctly-lowered IR

## 7. Normative Requirements

1. **Portable Operations SHALL use portable dialects**: Control flow, arithmetic, and directly-called functions use `func`, `cf`, `scf`, `arith` dialects
2. **Address-taken functions SHALL use backend dialects**: Functions whose address is taken use the backend's function definition (e.g., `llvm.func`)
3. **Struct operations SHALL use backend dialects**: Record, union, and closure struct manipulation uses backend-specific operations
4. **`llvm.call` target restriction**: `llvm.call` SHALL only call functions defined as `llvm.func`; to call a `func.func` from `llvm.func`, use `func.call`
5. **Platform configuration flow**: `fidproj` platform settings SHALL inform all lowering decisions

## See Also

- [Program Semantic Graph](program-semantic-graph.md) - PSG structure consumed by Alex
- [Type Representation Architecture](type-representation-architecture.md) - NativeType and TypeConRef
- [Closure Representation](closure-representation.md) - Flat closure memory layout
- [Lazy Representation](lazy-representation.md) - Lazy as extended closure
- [Platform Bindings](platform-bindings.md) - Platform descriptor and syscalls
- [Native Type Mappings](native-type-mappings.md) - Type-to-layout mapping
