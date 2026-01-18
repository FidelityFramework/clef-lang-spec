# Backend Lowering Architecture

> **Status**: Informative Draft
> **Last Updated**: 2026-01-18

## 1. Overview

This chapter specifies how F# Native lowers high-level constructs to backend-specific representations. Understanding this architecture is essential for:

- **.NET developers** transitioning to native compilation, where there is no runtime to hide ABI details
- **Backend implementers** wiring up new platform targets via `fidproj` configuration
- **Language feature implementers** understanding when portable vs backend-specific representations are required

## 2. The Two-Layer Model

F# Native uses a two-layer intermediate representation during compilation:

```
F# Source → FNCS → SemanticGraph → Alex → MLIR (mixed dialects) → Backend → Native Binary
                                          ↑
                                   Portable + Backend-Specific
```

### 2.1 MLIR Portable Dialects

Operations that are **semantically backend-independent** use MLIR's portable dialects:

| Dialect | Purpose | Operations |
|---------|---------|------------|
| `func` | Function structure | `func.func`, `func.call`, `func.return` |
| `cf` | Control flow | `cf.br`, `cf.cond_br`, `cf.switch` |
| `scf` | Structured control | `scf.while`, `scf.for`, `scf.if` |
| `arith` | Arithmetic | `arith.addi`, `arith.constant`, `arith.cmpi` |

These dialects can lower to multiple backends:
- LLVM (current primary target)
- SPIR-V (GPU compute)
- WebAssembly (browser/embedded)
- Custom hardware

### 2.2 Backend-Specific Dialects

Operations that **commit to a specific representation** require backend-specific dialects. For LLVM targets:

| Category | Operations | Why Backend-Specific |
|----------|------------|---------------------|
| Function pointers | `llvm.mlir.addressof`, indirect `llvm.call` | Pointer representation varies |
| Struct manipulation | `llvm.insertvalue`, `llvm.extractvalue`, `llvm.getelementptr` | No portable struct type |
| Memory operations | `llvm.load`, `llvm.store`, `llvm.alloca` | ABI-dependent alignment |

## 3. The Flat Closure Pattern

F# Native implements closures, lazy values, and sequences using **MLKit-style flat closures**. This pattern stores function pointers in structs:

```
Closure: { code_ptr, cap₀, cap₁, ... }
Lazy:    { computed, value, code_ptr, cap₀, cap₁, ... }
Seq:     { state, current, code_ptr, cap₀, cap₁, ... }
```

### 3.1 Why Flat Closures Require Backend-Specific Code

The flat closure pattern requires:

1. **Taking a function's address** - Backend-specific operation
2. **Storing address in struct** - Backend-specific struct manipulation
3. **Calling through stored pointer** - Backend-specific indirect call

There is no portable MLIR representation for "pointer to function". Each backend has its own function pointer model:

| Backend | Function Pointer |
|---------|-----------------|
| LLVM | `!llvm.ptr` + `llvm.mlir.addressof` |
| SPIR-V | Function tables, `OpFunctionPointer` |
| WebAssembly | Function indices, `call_indirect` |

### 3.2 Correct Dialect Usage for Flat Closures

```mlir
// Thunk function - LLVM dialect (address will be taken)
llvm.func private @lazy_thunk(%struct_ptr: !llvm.ptr) -> i64 {
    // Struct access - LLVM (backend-specific)
    %cap_ptr = llvm.getelementptr %struct_ptr[0, 3] : !llvm.ptr -> !llvm.ptr
    %cap = llvm.load %cap_ptr : !llvm.ptr -> i64

    // Arithmetic - portable
    %result = arith.addi %cap, %cap : i64

    // Return - matches function dialect
    llvm.return %result : i64
}

// Entry point - func dialect (called by name, portable)
func.func @main() -> i32 {
    // ...
    func.return %ret : i32
}
```

## 4. Classification Guide

### 4.1 Use Portable Dialects When

- The operation has **no representation dependency** on the target
- The function is **called directly by name** (not through a pointer)
- Control flow is **structural** (branches, loops, conditions)
- Operations are **arithmetic** (add, compare, etc.)

### 4.2 Use Backend-Specific Dialects When

- Taking a **function's address** for storage or indirect call
- Manipulating **heterogeneous structs** (records, closures, unions)
- Operations have **ABI implications** (memory layout, alignment)
- Platform-specific **intrinsics** (syscalls, atomics)

## 5. For .NET Developers

In .NET F#, the CLR runtime handles many details invisibly:

| Concept | .NET F# | F# Native |
|---------|---------|-----------|
| Function values | Delegate objects, GC-managed | Flat closure structs |
| Method dispatch | Virtual tables, JIT | Direct calls or function pointers |
| Memory layout | Runtime decides | Compiler computes, ABI-constrained |
| Closures | Heap-allocated objects | Stack/arena structs with captures |

F# Native makes these representations **explicit and controllable**. The flat closure pattern gives:
- **Predictable memory usage** - No hidden allocations
- **Cache-friendly layout** - Captures inline, not indirected
- **Deterministic lifetimes** - Region-based, not GC

The tradeoff is that function pointer operations commit to a specific backend representation.

## 6. For Backend Implementers

When adding a new backend target:

### 6.1 Platform Descriptor

Create a `PlatformDescriptor` quotation in `Fidelity.Platform` that defines:
- Architecture and word size
- Type layouts and alignments
- Syscall conventions
- Entry point ABI

### 6.2 Dialect Conversion

Implement conversions from MLIR portable dialects to your backend:
- `func.func` → your function representation
- `cf.*` → your control flow
- `arith.*` → your arithmetic ops

### 6.3 Flat Closure Backend

Implement the flat closure pattern for your backend:
- Function address operation
- Struct manipulation operations
- Indirect call mechanism

The **portable parts** (control flow, arithmetic) use standard MLIR conversions. The **backend-intrinsic parts** require custom implementation.

## 7. fidproj Platform Configuration

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
2. FNCS uses platform info for type layouts and intrinsic typing
3. Alex selects the appropriate backend dialect usage
4. LLVM/backend receives correctly-lowered IR

## 8. Normative Requirements

1. **Portable Operations SHALL use portable dialects**: Control flow, arithmetic, and directly-called functions use `func`, `cf`, `scf`, `arith` dialects
2. **Flat closure functions SHALL use backend dialects**: Functions whose address is taken use the backend's function definition
3. **Struct operations SHALL use backend dialects**: Record, union, and closure struct manipulation uses backend-specific operations
4. **Platform configuration SHALL flow through compilation**: `fidproj` platform settings inform all lowering decisions

## 9. See Also

- [Closure Representation](../closure-representation.md) - MLKit-style flat closures
- [Lazy Representation](../lazy-representation.md) - Lazy as extended closure
- [Platform Bindings](../platform-bindings.md) - Platform descriptor and syscalls
- [Native Type Mappings](../native-type-mappings.md) - Type-to-layout mapping
