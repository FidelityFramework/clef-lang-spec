---
title: "NTU Dimensional Type Architecture"
weight: 150
---

> **Status**: Design: Active
> **Normative**: Prospective (will become normative as sections are implemented)
> **Last Updated**: 2026-02-11
> **Companion Specs**: [ntu-types.md](ntu-types.md), [platform-bindings.md](platform-bindings.md), [program-semantic-graph.md](program-semantic-graph.md)

## 1. Motivation

The NTU width-as-dimension redesign (February 2026) replaced 16 discrete integer/float
variants with 3 parameterized kinds (`NTUint of NTUWidth`, `NTUuint of NTUWidth`,
`NTUfloat of NTUWidth`), where `NTUWidth = Fixed of int | Resolved of WidthDimension`.
This was a structural improvement, but it addressed only one axis of what the Fidelity
Framework's Dimensional Type System (DTS) requires.

The Fidelity Framework targets heterogeneous compilation: a single F# application may
contain sections of its program graph that target CPU, GPU, FPGA, NPU, or other compute
architectures. Each section resolves to concrete types for its target, but the NTU must
provide the abstract dimensional substrate that makes cross-target type reasoning possible.

This document specifies the architectural direction for the NTU as a multi-dimensional
type substrate, not merely "integers with variable width" but a type system where
multiple dimensions survive compilation, [flow through the Program Semantic Graph](program-semantic-graph.md), and
inform code generation for any target.

### 1.1 Design Provenance

The insight chain that motivates this architecture:

1. **Farscape exposed the C `int`/`long` problem**: C `int` and `long` have
   platform-dependent widths that don't map cleanly to `Pointer` or `Register`.
   This revealed that width resolution is a genuine design axis, not a detail.

2. **PlatformABI works for desktop but not heterogeneous compute**: Concrete
   per-platform width resolution (PInvokeTypeMapper) is correct for .NET P/Invoke
   targeting desktop/server platforms. But the same approach cannot extend to
   microcontrollers (8-bit registers, 16-bit pointers), FPGAs (synthesis-parameter
   datapaths), GPUs (warp-level execution, shared memory tiling), or NPUs (vector
   registers, cascade accumulators).

3. **Not every platform needs a binding generator**: FPGAs don't have C libraries to
   bind; the application IS the hardware design. NPU tile code is native, not foreign.
   The NTU needs to express target-native types directly, not only through FFI mappings.

4. **Width is one dimension among many**: The DTS vision (see "Doubling Down on DMM
   and DTS", SpeakEZ Technologies, January 2026) identifies dimensional types that
   don't erase after type checking; they flow through the PSG and inform code
   generation. Width, memory space, access pattern, alignment, tensor shape are all
   dimensions in this sense.

5. **BAREWire reconciles heterogeneous sections**: When a CPU component hands data
   to a GPU kernel, BAREWire defines the memory layout contract. Each side compiles
   against its own PlatformContext, but the BAREWire contract ensures the handoff is
   type-safe. "Good contracts make good boundaries."

### 1.2 Core Principle

**The NTU is a [multi-dimensional type substrate](https://arxiv.org/abs/2603.16437).** Types carry dimensional metadata
that survives compilation and informs target-specific code generation. When a section
of the program graph takes a platform definition, that section becomes concretely typed
for its target, but the NTU machinery is not to enumerate those concrete types.
It is to have the dimensional underpinnings to accept and map from target to target.

---

## 2. Dimensional Axes

The NTU's type system is organized around multiple orthogonal dimensional axes.
Width (the current `NTUWidth`) is the first implemented axis. This section catalogs
the full dimensional landscape.

### 2.1 Width Dimension (Implemented)

The bit-width of numeric types. Can be fixed or platform-resolved.

```fsharp
type WidthDimension =
    | Pointer    // Address width
    | Register   // Machine register / natural word width

type NTUWidth =
    | Fixed of bits: int
    | Resolved of WidthDimension
```

**Resolution**: `PlatformContext.Dimensions: Map<WidthDimension, int>` provides
concrete bit widths per target.

**Current scope**: Sufficient for CPU and MCU targets where Pointer and Register
capture the two independent hardware width axes. Farscape resolves C ABI-specific
widths (C `int`, C `long`) at code generation time using PlatformABI, emitting
Fixed-width NTU types. Only genuinely platform-abstract types (`size_t`, `intptr_t`,
F# `int`, `nativeint`) use `Resolved`.

**Future extension**: When FPGA or NPU targets require width dimensions beyond
Pointer and Register (e.g., synthesis-parameter datapath width, vector register width),
the `WidthDimension` type may need to become extensible. The `Dimensions` map
mechanism already supports arbitrary keys; the constraint is the closed DU.
This extension is deferred until a concrete target demands it. Fidelity.Platform
(not Farscape) would be the component declaring new width dimensions.

### 2.2 Memory Space Dimension (Design)

The memory space a value inhabits. Critical for GPU targets where global, shared,
and private memory have fundamentally different performance and coherency characteristics.

Candidate dimensional axis:

| Memory Space | Meaning | Primary Target |
|---|---|---|
| Stack | Thread-local stack allocation | All |
| Heap / Arena | Arena-managed allocation | CPU, MCU |
| Global | Device global memory | GPU |
| Shared | Workgroup-shared memory (programmer-controlled cache) | GPU |
| Private | Per-thread register file | GPU |
| Peripheral | Memory-mapped I/O | MCU (via BAREWire) |
| Constant | Read-only, broadcast-optimized | GPU |

**Interaction with type system**: Memory space annotations could be type-level
properties (analogous to Rust's lifetime annotations or CUDA's `__device__`,
`__shared__`, `__constant__` qualifiers) that flow through the PSG and constrain
code generation. The type system would prevent invalid cross-space access at
compile time rather than runtime.

**BAREWire relationship**: BAREWire already describes peripheral memory-mapped
access patterns (ReadOnly, WriteOnly, ReadWrite) and address layouts. The memory
space dimension would formalize what BAREWire expresses operationally.

### 2.3 Access Pattern Dimension (Design)

How a memory location may be accessed. Overlaps with memory space but is
orthogonal: a global memory location can be read-only or read-write.

| Access Pattern | Meaning | Enforcement |
|---|---|---|
| ReadOnly | Immutable access | Compile-time (default for let bindings) |
| WriteOnly | Write-only (e.g., output peripheral register) | Compile-time |
| ReadWrite | Mutable access | Compile-time (mutable bindings) |
| Volatile | Access must not be reordered or eliminated | Code generation |

**BAREWire relationship**: BAREWire's `PeripheralDescriptor` already carries
access qualifiers. This dimension formalizes them in the NTU.

### 2.4 Alignment Dimension (Design)

Alignment constraints that flow through the type system and inform layout decisions.

| Alignment | Meaning | Source |
|---|---|---|
| Natural | Type's natural alignment (size-dependent) | Default |
| CacheLine | Aligned to cache line boundary (64B x86, 128B Apple Silicon) | Platform-resolved |
| Packed | No padding between fields | Explicit annotation |
| Page | Page-aligned (4KB typical) | DMA, memory mapping |

**Cache-aware compilation**: As described in the SpeakEZ "Cache-Conscious Memory
Management" posts, cache line size varies by architecture and is a compilation
dimension. The type system can enforce cache-line alignment by construction,
preventing false sharing between actors without runtime discipline.

**Resolution**: Like width, alignment constraints can be Fixed or platform-Resolved.
Cache line size comes from PlatformContext; natural alignment comes from the type's
layout; explicit alignment comes from annotations.

### 2.5 Tensor Shape Dimension (Design, Future)

For ML/compute targets, tensor indices (batch, channel, height, width) are dimensional
metadata that could flow through the type system. This is relevant for NPU targeting
where tensor layout affects memory access patterns and computation scheduling.

### 2.6 Temporal/Lifetime Dimension (Existing, Coeffect System)

Resource lifetimes and ownership semantics are already partially modeled through
CCS's coeffect system and the Deterministic Memory Management (DMM) architecture.
These interact with memory space and access pattern dimensions: a value's lifetime
constrains which memory spaces it can inhabit, and ownership determines which access
patterns are valid.

---

## 3. Multi-Stack Compilation Model

### 3.1 Program Graph Sections

A Fidelity application targeting heterogeneous hardware has sections of its program
graph that inhabit different platforms:

```
┌─────────────────────────────────────────────────────┐
│                  F# Source Program                   │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ CPU      │  │ GPU      │  │ FPGA     │          │
│  │ Section  │  │ Section  │  │ Section  │          │
│  │          │  │          │  │          │          │
│  │ LP64     │  │ CUDA/    │  │ Dataflow │          │
│  │ Cache:64B│  │ SPIR-V   │  │ HLS      │          │
│  │ LLVM IR  │  │ Warp:32  │  │ Custom   │          │
│  │          │  │ Shared:  │  │ widths   │          │
│  │          │  │ 48KB/SM  │  │          │          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
│       │             │             │                  │
│       └──────┬──────┘             │                  │
│              │                    │                  │
│       ┌──────┴──────┐     ┌──────┴──────┐           │
│       │  BAREWire   │     │  BAREWire   │           │
│       │  Contract   │     │  Contract   │           │
│       │  (CPU↔GPU)  │     │  (CPU↔FPGA) │           │
│       └─────────────┘     └─────────────┘           │
└─────────────────────────────────────────────────────┘
```

Each section resolves its NTU types against **its own PlatformContext**:

- CPU section: `Register → 64`, `Pointer → 64`, `CacheLine → 64`
- GPU section: `Register → 32`, `Pointer → 64`, `Warp → 32`, `SharedMemory → 49152`
- FPGA section: `Register → 24` (synthesis parameter), `Pointer → 32`

The NTU type `NTUint(Resolved Register)` is the **same type** in all sections. Its
concrete width differs per platform context. Type identity is preserved; only
resolution varies.

### 3.2 Section Compilation

Each section of the graph is compiled separately against its platform context:

1. **CCS** elaborates the full program graph with dimensional types preserved
2. **Alex** partitions the graph into target sections (CPU, GPU, FPGA, etc.)
3. Each section is compiled with its own PlatformContext providing dimension resolutions
4. BAREWire contracts between sections are verified for structural compatibility
5. Each section emits target-specific code (LLVM IR, SPIR-V, HLS, etc.)

The NTU's dimensional machinery gives CCS the vocabulary to verify both sides of
every BAREWire contract against their respective platform contexts.

### 3.3 BAREWire as Reconciliation Layer

BAREWire serves three roles in the multi-stack model:

1. **Memory layout specification**: Deterministic, zero-copy layouts that map directly
   to hardware expectations. Position-based encoding enables hardware-enforced safety.

2. **Inter-section data contracts**: When CPU hands data to GPU, BAREWire describes
   the layout, alignment, and access semantics at the boundary. Each side interprets
   the contract through its own dimensional lens.

3. **Cross-target type reconciliation**: An AVX2 cache line (CPU) may need to be
   reconciled with a bit stream (FPGA) or a coalesced memory transaction (GPU).
   BAREWire's binary encoding provides the common ground; the dimensional type
   system on each side verifies that its interpretation is consistent.

---

## 4. PlatformContext Evolution

### 4.1 Current State (Width Only)

```fsharp
type PlatformContext = {
    PlatformId: string
    Dimensions: Map<WidthDimension, int>  // Pointer → 64, Register → 64
    PointerAlign: int
    PlatformLibraryPath: string option
    Predicates: Map<PlatformPredicate, bool>
    FreestandingStartup: FreestandingStartup option
}
```

### 4.2 Multi-Dimensional Evolution

As new dimensional axes are implemented, PlatformContext evolves to carry resolutions
for all axes. The existing `Dimensions` map may generalize:

```fsharp
// Conceptual direction: NOT a concrete implementation proposal
type PlatformContext = {
    PlatformId: string
    WidthDimensions: Map<WidthDimension, int>
    CacheLineSize: int                           // bytes
    MemorySpaces: Set<MemorySpace>               // what spaces exist on this target
    SimdWidth: int option                        // SIMD register width, if applicable
    ComputeModel: ComputeModel                   // Sequential | SIMT | Dataflow | ...
    Predicates: Map<PlatformPredicate, bool>
    // ...
}
```

The exact shape depends on which dimensional axes are implemented. The principle is:
**PlatformContext is the single source of truth for how abstract dimensions resolve
on a specific target.** Fidelity.Platform is the component that constructs
PlatformContext from platform descriptors (fidproj TOML).

### 4.3 Dimension Resolution Flow

```
fidproj TOML → Fidelity.Platform → PlatformContext → Alex (per section)
                                                      ↓
                                                 MLIR emission with
                                                 concrete widths,
                                                 aligned layouts,
                                                 space-appropriate ops
```

---

## 5. Relationship to Ecosystem Components

### 5.1 CCS (Clef Compiler Service)

CCS owns the NTU type definitions and the type checker. Dimensional types are
first-class in the type system; they don't erase after type checking. CCS validates
dimensional consistency (e.g., you cannot add a `Pointer`-width integer to a
`Fixed 32` integer without explicit conversion) without knowing the target platform.

### 5.2 Firefly / Alex (Code Generation)

Alex resolves dimensional types to concrete values using the PlatformContext for each
graph section. The `mapNativeTypeForArch` function in TypeMapping.fs is the resolution
point: it matches on NTUWidth dimensions and emits concrete MLIR types.

### 5.3 Fidelity.Platform

The platform descriptor layer. Constructs PlatformContext from fidproj TOML
configuration. As dimensional axes are added, Fidelity.Platform grows the set of
resolutions it provides. For novel targets (FPGA, NPU), Fidelity.Platform is the
component that would declare target-specific dimensions, not Farscape, not BAREWire.

### 5.4 Farscape (C/C++ Binding Generator)

A consumer of the NTU's dimensional types, not a driver. Farscape maps C/C++ header
declarations to NTU types:

- **Fidelity output**: Uses PlatformABI to resolve C-specific widths (C `int`,
  C `long`) at generation time, emitting Fixed-width NTU types. Only genuinely
  platform-abstract types (`size_t → unativeint`, `intptr_t → nativeint`) use
  Resolved dimensions.
- **P/Invoke output**: Uses PInvokeTypeMapper for CLR-concrete types. Entirely
  separate from the NTU.

Farscape's contribution to the dimensional architecture was diagnostic: it revealed
that the NTU's WidthDimension vocabulary was insufficient for C ABI diversity, which
catalyzed the broader rethinking documented here.

### 5.5 BAREWire

The memory layout and reconciliation layer. BAREWire provides:

- Deterministic memory layouts (zero-copy where possible)
- Peripheral/hardware register descriptions (MCU targeting)
- Inter-section data contracts (heterogeneous compilation)
- Binary encoding for cross-target data exchange

BAREWire is the operational mechanism; the NTU's dimensional type system is the
verification mechanism. Together they ensure that data flowing between heterogeneous
graph sections is structurally sound.

---

## 6. Implementation Roadmap

### Phase 1: Width Dimension (COMPLETE, February 2026)

- `NTUWidth = Fixed of int | Resolved of WidthDimension`
- `WidthDimension = Pointer | Register`
- `PlatformContext.Dimensions: Map<WidthDimension, int>`
- 3 parameterized kinds replace 16 discrete variants
- NTUother eliminated; no escape hatch

### Phase 2: Farscape Type System Separation (COMPLETE, February 2026)

- PInvokeTypeMapper.fs: CLR-concrete types per PlatformABI
- TypeMapper.fs: NTU-abstract types for Fidelity output
- PInvokeCodeGenerator uses PInvokeTypeMapper; FidelityCodeGenerator uses TypeMapper

### Phase 3: Farscape Fidelity Output PlatformABI Awareness (IN PROGRESS)

- TypeMapper gains PlatformABI parameter for C type resolution in Fidelity mode
- C `int`/`long` resolve to Fixed-width NTU types based on target platform
- Resolved dimensions reserved for genuinely platform-abstract types

### Phase N: Memory Space Dimension

- Memory space annotations as type-level properties
- GPU shared/global/private memory space tracking
- MCU peripheral memory via BAREWire integration
- Compile-time prevention of invalid cross-space access

### Phase N+1: Alignment Dimension

- Cache-line alignment as platform-resolved dimension
- False sharing prevention by construction
- SoA/AoS layout transformations guided by dimensional analysis

### Phase N+2: Multi-Stack Section Compilation

- Program graph partitioning into target sections
- Per-section PlatformContext resolution
- BAREWire contract verification between sections
- Target-specific code emission (LLVM, SPIR-V, HLS)

---

## 7. Open Questions

### 7.1 WidthDimension Extensibility

The current closed DU (`Pointer | Register`) is sufficient for CPU and MCU targets.
When FPGA or NPU targets require additional width dimensions (synthesis-parameter
widths, vector register widths), should WidthDimension become:

- **A. Extended closed DU**: Add variants as needed (compile-time safety, limited extensibility)
- **B. Open single-case DU**: `WidthDimension of string` with well-known constants (infinite extensibility, no compile-time exhaustiveness)
- **C. Keep closed, use Fixed**: Exotic targets use `Fixed` widths since their widths are known at binding time

Current leaning: **C** for the immediate term. FPGA widths are synthesis parameters
known at project configuration time, so `Fixed 24` (not `Resolved FpgaDatapath`) is
appropriate. `Resolved` is reserved for dimensions where the SAME source code genuinely
targets multiple platforms with different resolutions. Option B remains available if a
future target class requires it.

### 7.2 Dimensional Type Identity

When dimensions don't erase, they affect type identity. `NTUint(Fixed 32)` and
`NTUint(Resolved Register)` are already distinct types that don't unify. As more
dimensions are added, type identity becomes multi-dimensional:

- Is `NTUint(Fixed 32, Stack)` the same type as `NTUint(Fixed 32, Global)`?
- How do dimensional subtyping rules work? (Can a `ReadOnly` value be passed where `ReadWrite` is expected? No. Vice versa? Yes, covariant access.)

These questions become concrete as each dimensional axis is implemented.

### 7.3 BAREWire Contract Verification

How does CCS verify that both sides of a BAREWire contract are dimensionally
consistent? The layout description must be interpretable from both platform contexts.
This likely requires BAREWire layouts to be expressed in terms of Fixed dimensions
(concrete widths/alignments) rather than Resolved dimensions, since the two sides
may have different resolutions for the same dimension.

---

## 8. References

- "Doubling Down on DMM and DTS." SpeakEZ Technologies blog, January 2026.
- "Hyping Hypergraphs." SpeakEZ Technologies blog, August 2025.
- "Cache-Conscious Memory Management: CPU Edition." SpeakEZ Technologies blog, September 2025.
- "GPU Cache-Aware Compilation." SpeakEZ Technologies blog, September 2025.
- "Getting the Signal with BAREWire." SpeakEZ Technologies blog, December 2025.
- [ntu-types.md](ntu-types.md), NTU Type Nomenclature Specification.
- [platform-bindings.md](platform-bindings.md), Platform Bindings Specification.
- [program-semantic-graph.md](program-semantic-graph.md), PSG Architecture.
