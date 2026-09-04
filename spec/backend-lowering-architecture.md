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
F# Source → CCS (Clef Compiler Service) → PSG → Alex → MLIR (portable dialects) → Target pathway → Target artifact
                                                          ↑                          ↑
                                                   commits to no target        commits to one target
```

The tier boundary is *portable-versus-target-specific*. While LLVM is the first target we built for, it is one target pathway among several: for instance, an LLVM serializer handles CPU, WASM and MCU targets, a CIRCT path handles FPGA targets, an MLIR-AIE path handles NPU tile-array targets, and a JSIR serializer handles the JavaScript target. CIRCT, MLIR-AIE, and JSIR are also MLIR, but they are *target-specific* MLIR, so they live in the backend for the same reason the LLVM serializer does. What separates the tiers is the commitment, not the technology. The artifact class is part of the pathway's commitment: a native binary from the LLVM pathway, a bitstream from the CIRCT pathway, an NPU binary from the MLIR-AIE pathway, a JavaScript module from the JSIR pathway.

A target commitment is lossy. Whatever a program's semantics carries that the chosen target's model cannot express is destroyed the moment the commitment is made, and no other target pathway can recover it. The middle end's job is therefore *information preservation*: it holds full semantic content in a form every pathway can still read, so each backend commits from complete information. Emitting an `llvm.*` operation in the middle end is a category error rather than a stylistic one, because it commits to LLVM and forecloses every other pathway. The same is true of a CIRCT hardware operation in the middle end.

### 2.1 Portable Dialects

The middle end emits only portable dialects:

| Dialect | Purpose | Operations |
|---------|---------|------------|
| `func` | Function structure | `func.func`, `func.call`, `func.return` |
| `scf` | Structured control | `scf.while`, `scf.for`, `scf.if`, `scf.index_switch` |
| `arith` | Arithmetic | `arith.addi`, `arith.constant`, `arith.cmpi` |
| `memref` | Memory and layout | `memref.alloca`, `memref.global`, `memref.load`, `memref.store` |
| `index` | Target-word integers | `index.constant`, `index.casts` |

These five dialects, and no other, are the witnessed vocabulary; they lower to any target pathway: LLVM, CIRCT, MLIR-AIE, JSIR, SPIR-V, WebAssembly. Block-based control flow (`cf.*`) is produced by the pathway's standard `scf` lowering and never appears above the witness boundary. Admitting a further dialect (for example `affine`) is a change to this table first, made step-wise and recorded here before any witness emits it.

### 2.2 Constructs Whose Realization Is a Target Commitment

Some constructs are expressed portably in the middle end and take a target-specific form only in the pathway. None requires a deferred cast: each has a portable carrier the pathway's standard lowerings already consume. An earlier revision of this section held that a function address as data and a raw environment pointer had no portable representation and carried each as a `builtin.unrealized_conversion_cast`; that premise is retired by §4, which shows a function value is never data in the interior.

| Category | Portable middle-end carrier | Committed by the target pathway |
|----------|-----------------------------|------------------------------|
| Function values | `func.constant` / `func.call_indirect`; a closure is the pair `(fn, env)` (§4.2) | `llvm.mlir.addressof` / `llvm.call` (LLVM pathway); function table index (SPIR-V, WebAssembly); host function value (JSIR pathway) |
| Struct manipulation | `memref` of the settled layout | ABI struct access (per pathway) |
| Volatile MMIO | typed `Mmio` op held to the serializer | `llvm.inttoptr` + `llvm.store volatile` (LLVM pathway) |

## 3. Where a Target Commitment Happens

### 3.1 The Middle End Emits Portable Dialects For

- Every operation with no representation dependency on the target
- Directly-called functions (`func.func`, called by name)
- Structural control flow (branches, loops, conditions)
- Arithmetic and comparison
- Function values and closures, as the two-value pair `(fn, env)` (§4.2)

### 3.2 The Target Pathway Commits For

- The memory form of a function address — at the extern boundary, where a C API receives a callback ([FFI Boundary §3.2](ffi-boundary.md))
- Manipulating heterogeneous structs (records, closures, unions) into the target ABI
- Operations with ABI implications (memory layout, alignment)
- Target intrinsics (syscalls, atomics, volatile MMIO)

None of these commitments appears in middle-end IR. Each is realized by the pathway's standard lowerings for the portable dialects (§4.3); no pathway requires a resolution pass or plugin.

## 4. Function Values in the Interior

The middle end represents a function value as two SSA values — a function symbol and an environment buffer — and passes them as two parameters. This is the multi-value form of [Closure Representation §6.3](closure-representation.md), and it is the whole of the closure calling convention above the witness boundary. Nothing about a function value is deferred to a target pathway.

### 4.1 Why No Commitment Is Deferred

An earlier revision of this chapter held that a portable dialect cannot express a function pointer in memory, and therefore encoded a closure as an `index` pair carried through `builtin.unrealized_conversion_cast` and resolved by a target-specific pass. That analysis was right about `memref` — a memref of function type is not expressible — and wrong in its conclusion, because a Clef closure is not that object. The environment is a byte buffer of captured *data*; the code is a `func.func` symbol referenced by name. `func.constant` is a first-class SSA value in the portable dialect, `func.call_indirect` consumes it, and the pair `(fn, env)` crosses any function boundary as two parameters. The things the earlier revision named as inexpressible — a function pointer in memory, an indirect call through it, its representation type — never need expressing. The interior forms are enumerated in [Closure Representation §7](closure-representation.md), all in `func` + `memref` + `arith`.

### 4.2 The Multi-Value Encoding

A closure value is `(%fn : (memref<Exi8>, args...) -> ret, %env : memref<Exi8>)`, where `E` is the environment extent settled at saturation. Creation is `func.constant @lifted` plus the allocation the lifetime lattice selects ([Closure Representation §3.3](closure-representation.md)) and a `memref.store` of each capture at its literal offset through a static `memref.view`. Application is `func.call_indirect %fn(%env, args...)`; where the callee is known at saturation the form degenerates to `func.call @lifted(%env, args...)` with no function value at all.

There is no cast. `builtin.unrealized_conversion_cast` SHALL NOT appear in the middle end's output for any construct, and no target pathway resolves a closure form: each pathway receives standard `func` and `memref` operations that its standard lowerings already handle. A closure that must reside in memory — an aggregate field, a container element — resides as its environment buffer, with the code component fixed by the closure form; the placement of a code component in memory is settled by that form at saturation, not by a cast at emission.

### 4.3 Realization on Each Pathway

Every pathway consumes the multi-value form with its standard lowerings, and none adds a pass:

- **CPU and MCU (LLVM).** `func.constant` → `llvm.mlir.addressof`; `func.call_indirect` → `llvm.call` through the function value; the environment `memref` → the pathway's memref lowering.
- **WebAssembly.** `func.constant` → a table index consumed by `call_indirect`; the environment lives in linear memory. The pair remains two values.
- **JSIR.** Under §4.5, the environment buffer is realized by the host closure and the pair collapses to the host function value.

A lazy thunk shows the interior at its simplest: `Lazy<int>` is an environment `{state: i1, value: i32}` with a known-callee body, so forcing is a direct `func.call @thunk(%env)` guarded by the state flag — no indirect call and no pointer anywhere. Escaping function values use `call_indirect`; the discriminating condition is the closure form, decided at saturation.

### 4.4 Function-Body Composition

Within a target pathway, MLIR allows portable and committed operations to coexist in one body, which is what makes the deferred-resolution pass a local rewrite rather than a whole-program retype:

1. **`func.call` inside `llvm.func`**: valid; an `llvm.func` can call a `func.func` using `func.call`.
2. **`llvm.call` target restriction**: `llvm.call` can only call functions defined as `llvm.func`.
3. **Portable ops in any function**: `arith.*`, `cf.*`, `scf.*`, `memref.*` operations remain valid in a committed `llvm.func` body after resolution.

### 4.5 Carrier Realization on Pathways Without Linear Memory

The portable carriers of this chapter presume nothing about the target's memory model. A `memref` of a storage block and an `index`-carried pointer are stand-ins. On a pathway whose target has linear memory (LLVM, CIRCT), they commit to byte layouts and machine pointers, and the layout figures of the representation chapters describe that commitment. On a pathway whose target has no linear memory, they commit to the pathway's own value model. The JSIR pathway is the current instance: its target manipulates host objects and function values, not bytes and addresses.

Two provisions make this realizable without weakening the middle end's target neutrality:

1. **Access is realized per pathway.** Structured storage is reached through the access forms the middle end emits: the generated functions of [Discriminated Union Representation §10.2](discriminated-union-representation.md), field access over the storage carrier for records and tuples, capture reads for closures. A pathway realizes each access form in its own model: a byte-offset load on the LLVM pathway, a field select on the CIRCT pathway, a property access or a host-closure capture on the JSIR pathway.

2. **A pathway MAY read the graph.** The [Program Semantic Graph](program-semantic-graph.md) carries the program's type structure (field names, case identities, capture sets) as codata beside the portable IR, under the same discipline that carries blade support and escape classification ([Grade Discipline §3.3.1](grade-discipline.md)): read during lowering, absent from what is emitted. A pathway whose value model needs structural identity reads it from the graph during realization; the JSIR pathway needs property names where the LLVM pathway needs byte offsets.

A realization SHALL preserve the structural requirements of the representation chapters: construction, elimination, initialization, capture modes, case identity, and every other observable the chapter states. The layout figures of those chapters (sizes, offsets, alignment, tag width) bind only pathways that realize memory layouts. Each representation chapter whose realization on the JSIR pathway requires a decision carries a JSIR-pathway realization section stating it ([Discriminated Union Representation §10.2.2](discriminated-union-representation.md), [Option Operations Representation](option-operations-representation.md), [Closure Representation §6.4](closure-representation.md)).

## 5. Entry Point Example

The entry point is where a pathway's target commitment is most visible, because it is inherently target-specific: how a program receives control and how it exits are properties of the target, not of the program. The middle end emits `main` as an ordinary `func.func`; the pathway supplies the entry glue.

A pathway operates in one of two modes. A **hosted** pathway targets an environment with an OS runtime beneath the program (an x86-64 Linux pathway, say), which supplies process startup, `syscall`, and stack-passed `argc`/`argv`. A **freestanding** pathway targets an environment with no such runtime: the program is self-contained and receives control directly. The mode governs only the presence of a host runtime; it does not by itself fix word size, whether an allocator or C library is linked, or the core count — those are properties of the specific target, not of being freestanding. (A unikernel is one freestanding target: a self-contained image with no host OS. Not every freestanding target is a unikernel, and this mode distinction does not turn on that term.)

In freestanding mode the pathway emits an `_start` in its committed function form (its address is taken by the linker) that calls the portable `main`. The glue below is written for a hosted x86-64 pathway; another pathway supplies its own. On the Cortex-M33 freestanding pathway there is no `syscall` and no stack-passed `argc`/`argv`: `_start` is the reset entry, arguments do not exist, and exit is a halt, so the glue is entirely different while `main` is unchanged.

```mlir
// Target pathway (x86-64 hosted) — committed dialect. NOT middle-end output.
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

The JSIR pathway has no `_start` analog. Its artifact is a module, and its entry commitment is the export glue for the host's module convention: the pathway emits exported bindings, and the host invokes them (the entry-point modes of [Program Structure and Execution](program-structure-and-execution.md)). Entry glue remains per pathway in every case: a syscall preamble on the hosted x86-64 pathway, a reset vector on the M33 pathway, module exports on the JSIR pathway.

## 6. Platform Configuration

The project file specifies the target platform. The values below are one target; the M33 freestanding pathway sets `target = "thumbv8m.main-none-eabi"` and `word_size = 32`, and the same fields drive its layout and word size.

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
4. The target pathway for the selected target commits and receives correctly-lowered IR

The JSIR pathway takes no `word_size`: it realizes no byte layouts (§4.5), and its integer realization is specified in [Width Inference §8](width-inference.md). Its platform configuration specifies a host environment in place of an architecture triple, described by the managed-substrate descriptor of [Platform Bindings](platform-bindings.md).

## 7. Normative Requirements

1. **The middle end SHALL emit only portable dialects**: Control flow, arithmetic, function values and calls, and memory use `func`, `scf`, `arith`, `memref`, and `index`, and no other dialect. The middle end SHALL NOT emit `llvm.*`, `cf.*`, `builtin.unrealized_conversion_cast`, or any target-specific operation; block-based control flow is produced by the pathway's standard `scf` lowering.
2. **A target commitment SHALL occur only in a target pathway**: The memory form of a function address, ABI struct layout, and target intrinsics are committed by the selected pathway's standard lowerings (e.g., `func.constant` → `llvm.mlir.addressof` on the LLVM pathway), never in middle-end IR.
3. **Struct manipulation SHALL be carried portably and committed per pathway**: Record, [union](discriminated-union-representation.md), and closure struct access is emitted over `memref` in the middle end and committed by the target pathway: to the target ABI on a pathway that realizes memory layouts, and to the pathway's own value model on a pathway that does not (§4.5).
4. **`llvm.call` target restriction**: within the LLVM pathway, `llvm.call` SHALL only call functions defined as `llvm.func`; to call a `func.func` from `llvm.func`, use `func.call`
5. **Function values are two SSA values**: the middle end SHALL represent a closure as `(fn, env)` per §4.2 and SHALL NOT convert a function address to or from data; `builtin.unrealized_conversion_cast` SHALL NOT appear in middle-end output, and no pathway SHALL require a cast-resolution pass or plugin to consume a closure
6. **Platform configuration flow**: `fidproj` platform settings, including `word_size`, SHALL inform all lowering decisions; pointer and word width SHALL be taken from the selected target's `word_size` and SHALL NOT be assumed to be 64-bit
7. **Carrier realization**: a target pathway SHALL realize the portable storage carriers and their access operations in its own value model, MAY read the Program Semantic Graph's type structure during that realization, and SHALL preserve every structural requirement the representation chapters state; the layout figures of the representation chapters SHALL bind only pathways that realize memory layouts (§4.5)
8. **Artifact class**: the artifact class a pathway produces is part of its commitment: a native binary on the LLVM pathway, a bitstream on the CIRCT pathway, an NPU binary on the MLIR-AIE pathway, a JavaScript module on the JSIR pathway
