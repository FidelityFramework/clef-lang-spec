---
title: "Backend Lowering Architecture"
weight: 660
category: Compiler
status: normative
---

> **Status**: Normative
> **Last Updated**: 2026-01-19

## Informative References

> **Commentary**: For accessible explanation of the portable middle end and why a target commitment is deferred to the backend, see [Why Clef Is A Natural Fit for MLIR](https://clef-lang.com/docs/design/compilation/why-clef-fits-mlir/) in the Clef design documentation.
>
> **Artifact Class**: The deployment-facing rationale for the sealed-image artifact class that the freestanding mode of §5 enables is developed in [Getting to the Heart of Unikernels](https://clef-lang.com/blog/getting-to-the-heart-of-unikernels/) on the Clef blog.
>
> **For .NET Developers**: Guidance on transitioning from CLR concepts to native compilation is available in the rationale documentation.

---

## 1. Overview

This chapter specifies how Clef lowers high-level constructs through a portable middle end to a target-specific backend, using MLIR's multi-dialect architecture. The organizing distinction is between representation that is *portable* (independent of any target) and representation that *commits to a target*. The middle end holds the first; the backend performs the second.

## 2. Portable Middle End, Target-Committing Backend

Clef separates its intermediate representation by whether a construct has committed to a target:

```
F# Source → CCS (Clef Compiler Service) → PSG → Alex → MLIR (portable dialects) → Backend leg → Native Binary
                                                          ↑                          ↑
                                                   commits to no target        commits to one target
```

The tier boundary is *portable-versus-target-specific*, not *MLIR-versus-LLVM*. LLVM is one backend leg among several: an LLVM serializer handles CPU and MCU targets, and a CIRCT path handles FPGA targets. CIRCT is also MLIR, but it is *target-specific* MLIR, so it lives in the backend for the same reason the LLVM serializer does. What separates the tiers is the commitment, not the technology.

A target commitment is lossy. Whatever a program's semantics carries that the chosen target's model cannot express is destroyed the moment the commitment is made, and no other backend leg can recover it. The middle end's job is therefore *information preservation*: it holds full semantic content in a form every leg can still read, so each backend commits from complete information. Emitting an `llvm.*` operation in the middle end is a category error rather than a stylistic one, because it commits to LLVM and forecloses every other leg. The same is true of a CIRCT hardware operation in the middle end.

### 2.1 Portable Dialects

The middle end emits only portable dialects:

| Dialect | Purpose | Operations |
|---------|---------|------------|
| `func` | Function structure | `func.func`, `func.call`, `func.return` |
| `cf` | Control flow | `cf.br`, `cf.cond_br`, `cf.switch` |
| `scf` | Structured control | `scf.while`, `scf.for`, `scf.if` |
| `arith` | Arithmetic | `arith.addi`, `arith.constant`, `arith.cmpi` |
| `memref` | Memory and layout | `memref.alloca`, `memref.global`, `memref.load`, `memref.store` |
| `index` | Target-word integers | `index.constant`, `index.casts` |

These dialects lower to any backend leg: LLVM, CIRCT, SPIR-V, WebAssembly.

### 2.2 Constructs With No Portable Representation

Some constructs have no portable operation that expresses them, because their realization depends on a target the middle end has not chosen. The two the closure family relies on are the *function address as data* and the *raw environment pointer*. The middle end does not resolve these into any backend's form; it represents each as a `builtin.unrealized_conversion_cast`, MLIR's designated mechanism for a type conversion whose realization is deferred, and the backend leg for the chosen target performs the commitment. §4.2 develops this for the flat closure.

| Category | Portable middle-end carrier | Committed by the backend leg |
|----------|-----------------------------|------------------------------|
| Function pointers | `func_type → index` cast | `llvm.ptrtoint` / `llvm.inttoptr` (LLVM leg); function table index (SPIR-V) |
| Struct manipulation | `memref` of the closure layout | ABI struct access (per leg) |
| Volatile MMIO | typed `Mmio` op held to the serializer | `llvm.inttoptr` + `llvm.store volatile` (LLVM leg) |

## 3. Where a Target Commitment Happens

### 3.1 The Middle End Emits Portable Dialects For

- Every operation with no representation dependency on the target
- Directly-called functions (`func.func`, called by name)
- Structural control flow (branches, loops, conditions)
- Arithmetic and comparison
- The closure `(code_pointer, environment_pointer)` encoding, carried as `index`-in-`memref` (§4.2)

### 3.2 The Backend Leg Commits For

- Taking a function's address for storage or indirect call
- Manipulating heterogeneous structs (records, closures, unions) into the target ABI
- Operations with ABI implications (memory layout, alignment)
- Target intrinsics (syscalls, atomics, volatile MMIO)

None of these commitments appears in middle-end IR. Each is realized by the serializer or a resolution plugin for the specific target (§4.2).

## 4. Flat Closure Pattern and Deferred Resolution

Clef implements [closures](closure-representation.md), [lazy values](lazy-representation.md), and [sequences](seq-representation.md) using flat closures that store function pointers in structs. The commitment this pattern needs is deferred to the backend; the middle-end encoding stays portable.

### 4.1 Why the Commitment Is Deferred

The flat closure pattern needs three things a portable dialect cannot express directly:
1. Taking a function's address as data
2. Storing that address in a struct alongside captures
3. Calling through the stored pointer

There is no portable MLIR representation for "pointer to function"; each target realizes it differently:

| Backend leg | Function Pointer Mechanism |
|-------------|---------------------------|
| LLVM (CPU/MCU) | `!llvm.ptr` + `llvm.inttoptr` |
| CIRCT (FPGA) | placed hardware, no runtime pointer |
| SPIR-V | Function tables, `OpFunctionPointer` |
| WebAssembly | Function indices, `call_indirect` |

Because the realization differs per leg, the middle end must not pick one. It carries the closure in a form every leg can still commit from, and each leg performs its own commitment (§4.2).

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

### 4.3 Portable Encoding and Its Backend Realization

The middle end emits the thunk as a `func.func` and reads captures through a `memref`, with the deferred casts standing in for the address conversions. Nothing here names a target:

```mlir
// Middle end — portable dialects only.
func.func private @lazy_thunk(%env: memref<?xi64>) -> i64 {
    // Capture read — portable memref access, no ABI committed.
    %cap = memref.load %env[%c3] : memref<?xi64>
    %result = arith.addi %cap, %cap : i64
    func.return %result : i64
}
```

The LLVM leg resolves this into its pointer representation during the closure-cast pass (§4.2); the `func_type → index` and `index → memref` casts become `llvm.ptrtoint` / `llvm.inttoptr`, and the `memref` capture access becomes a `getelementptr` + `load` in the target ABI. A different leg resolves the same middle-end IR its own way. The realization is the backend's, not the middle end's.

### 4.4 Function-Body Composition

Within a backend leg, MLIR allows portable and committed operations to coexist in one body, which is what makes the deferred-resolution pass a local rewrite rather than a whole-program retype:

1. **`func.call` inside `llvm.func`**: valid; an `llvm.func` can call a `func.func` using `func.call`.
2. **`llvm.call` target restriction**: `llvm.call` can only call functions defined as `llvm.func`.
3. **Portable ops in any function**: `arith.*`, `cf.*`, `scf.*`, `memref.*` operations remain valid in a committed `llvm.func` body after resolution.

## 5. Entry Point Example

The entry point is where a backend leg's target commitment is most visible, because it is inherently target-specific: how a program receives control and how it exits are properties of the target, not of the program. The middle end emits `main` as an ordinary `func.func`; the leg supplies the entry glue.

A leg operates in one of two modes. A **hosted** leg targets an environment with an OS runtime beneath the program (an x86-64 Linux leg, say), which supplies process startup, `syscall`, and stack-passed `argc`/`argv`. A **freestanding** leg targets an environment with no such runtime: the program is self-contained and receives control directly. The mode governs only the presence of a host runtime; it does not by itself fix word size, whether an allocator or C library is linked, or the core count — those are properties of the specific target, not of being freestanding. (A unikernel is one freestanding target: a self-contained image with no host OS. Not every freestanding target is a unikernel, and this mode distinction does not turn on that term.)

In freestanding mode the leg emits an `_start` in its committed function form (its address is taken by the linker) that calls the portable `main`. The glue below is written for a hosted x86-64 leg; another leg supplies its own. On the Cortex-M33 freestanding leg there is no `syscall` and no stack-passed `argc`/`argv`: `_start` is the reset entry, arguments do not exist, and exit is a halt, so the glue is entirely different while `main` is unchanged.

```mlir
// Backend leg (x86-64 hosted) — committed dialect. NOT middle-end output.
llvm.func @_start() -> i32 {
    // Read argc/argv via inline asm — target-specific glue.
    %argc = llvm.inline_asm "mov (%rsp), $0", "=r" : () -> i64
    %argv = llvm.inline_asm "lea 8(%rsp), $0", "=r" : () -> !llvm.ptr

    // Call main — func.call, since main is the portable func.func.
    %result = func.call @main(%argc, %argv) : (i64, !llvm.ptr) -> i32

    // Exit syscall — target-specific glue.
    llvm.inline_asm has_side_effects "syscall", "..." %result : ...
    llvm.unreachable
}
```

## 6. Platform Configuration

The project file specifies the target platform. The values below are one target; the M33 freestanding leg sets `target = "thumbv8m.main-none-eabi"` and `word_size = 32`, and the same fields drive its layout and word size.

```toml
[compilation]
target = "x86_64-unknown-linux-gnu"

[platform]
word_size = 64
endianness = "little"
```

The word size is a platform property resolved from this configuration, not a constant baked into the middle end. Pointer and word width in every layout ([type layouts](type-representation-architecture.md)) come from `word_size` for the selected target, so the same IR yields a four-byte word on the M33 and an eight-byte word on x86-64. This configuration flows through:
1. `Fidelity.Platform` selects the appropriate `PlatformDescriptor`
2. CCS uses platform info for [type layouts](type-representation-architecture.md) and intrinsic typing
3. Alex emits portable IR parameterized by the platform word
4. The backend leg for the selected target commits and receives correctly-lowered IR

## 7. Normative Requirements

1. **The middle end SHALL emit only portable dialects**: Control flow, arithmetic, directly-called functions, memory, and the flat-closure encoding use `func`, `cf`, `scf`, `arith`, `memref`, `index`, and `builtin`. The middle end SHALL NOT emit `llvm.*` or any other target-specific operation.
2. **A target commitment SHALL occur only in a backend leg**: Taking a function's address, ABI struct layout, and target intrinsics are committed by the serializer or a resolution plugin for the selected target (e.g., an `llvm.func` on the LLVM leg), never in middle-end IR.
3. **Struct manipulation SHALL be carried portably and committed per leg**: Record, [union](discriminated-union-representation.md), and closure struct access is emitted over `memref` in the middle end and committed to the target ABI by the backend leg.
4. **`llvm.call` target restriction**: within the LLVM leg, `llvm.call` SHALL only call functions defined as `llvm.func`; to call a `func.func` from `llvm.func`, use `func.call`
5. **Target-free middle end**: the MiddleEnd SHALL encode flat closures using portable dialects and SHALL NOT commit to any leg's pointer type; the conversion of a function address to and from data SHALL be represented as `builtin.unrealized_conversion_cast` and resolved per target
6. **Deferred cast resolution**: a backend leg SHALL resolve the `func_type ↔ index` and `index → memref` closure casts into its own pointer representation; for the LLVM leg this resolution SHALL run after standard dialect conversions and before `--reconcile-unrealized-casts`
7. **Platform configuration flow**: `fidproj` platform settings, including `word_size`, SHALL inform all lowering decisions; pointer and word width SHALL be taken from the selected target's `word_size` and SHALL NOT be assumed to be 64-bit
