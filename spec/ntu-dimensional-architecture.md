---
title: "NTU Dimensional Type Architecture"
weight: 150
category: Language
status: normative
---

> **Companion Specs**: [ntu-types.md](ntu-types.md), [platform-bindings.md](platform-bindings.md), [program-semantic-graph.md](program-semantic-graph.md)

## 1. Overview

The Fidelity Framework targets heterogeneous compilation: a single Clef application may
contain sections of its program graph that target CPU, GPU, FPGA, NPU, or other compute
architectures. Each section resolves to concrete types for its target, but the NTU must
provide the abstract dimensional substrate that makes cross-target type reasoning possible.

This chapter specifies the NTU as a multi-dimensional type substrate, where
multiple dimensions survive compilation, [flow through the Program Semantic Graph](program-semantic-graph.md), and
inform code generation for any target.

### 1.1 Core Principle

**The NTU is a [multi-dimensional type substrate](https://arxiv.org/abs/2603.16437).** Types carry dimensional metadata
that survives compilation and informs target-specific code generation. When a section
of the program graph takes a platform definition, that section becomes concretely typed
for its target, but the NTU machinery is not to enumerate those concrete types.
It is to have the dimensional underpinnings to accept and map from target to target.

---

## 2. Dimensional Axes

The NTU's type system carries orthogonal dimensional metadata through the program graph.

### 2.1 Width Dimension

The bit-width of numeric types. Can be fixed or platform-resolved.

```fsharp
/// A width dimension is a name the platform description declares.
/// CPU descriptions declare Pointer and Register; a fabric binding declares its port widths.
type WidthDimension = WidthDimension of name: string

/// The width coeffect on a numeric node: the width selected from the analysed
/// range (never written in source), or the platform's declared dimension at
/// a site its ABI governs.
type NTUWidth =
    | Selected of bits: int           // from the range, among the declared representations
    | Resolved of WidthDimension      // the platform description's declared width at a boundary
```

**Resolution**: `PlatformContext.Dimensions` (declared name → bits) and
`PlatformContext.Representations` provide the declared widths and representations per target.

On CPU and MCU targets, Pointer and Register describe independent hardware width axes.
Farscape carries C ABI widths (C `int`,
C `long`) in the binding descriptor quotation it emits, a boundary declaration the compiler
reads; the Clef signature beside it is `int`. There is no fixed-width NTU type.

**Declared, not enumerated**: `WidthDimension` is a name the platform description declares (§7.1). `Pointer` and `Register` are the CPU declarations; a fabric binding declares its port widths; the `Dimensions` map is keyed by the declared name. Fidelity.Platform, not Farscape, declares width dimensions.

### 2.2 Memory Space Dimension

The memory space a value inhabits. Critical for GPU targets where global, shared,
and private memory have fundamentally different performance and coherency characteristics.

Memory-space identity follows §7.2. Region placement and target reachability are
specified in [Memory Regions](memory-regions.md).

### 2.3 Access Pattern Dimension

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

### 2.4 Alignment Dimension

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

### 2.5 Temporal/Lifetime Dimension

Resource lifetimes are modeled through
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
│                 Clef Source Program                  │
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

The numeric kind and measure contribute to type identity in every section.
CCS SHALL select the width from the value's justified range and that section's
declared representations. A boundary governed by a declared width dimension
SHALL also satisfy that boundary's coverage and transfer requirements.

### 3.2 Section Compilation

Each section of the graph is compiled separately against its platform context. Dimensional types are a design-time property this specification requires an implementation to establish, and their preservation across these stages is governed by the [preservation obligation through lowering](conformance.md):

1. CCS SHALL elaborate each section's target assignment and dimensional types on the program graph.
2. CCS SHALL settle each section's required representations and layouts against its declared platform context.
3. CCS SHALL verify dimensional compatibility, coverage and transfer obligations at BAREWire boundaries between sections.
4. Alex SHALL compose Elements, Patterns and Witnesses through its Huet zipper from those settled graph facts.
5. Backend lowering SHALL realize the witnessed operations for the assigned target while preserving the established contracts.

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

## 4. PlatformContext

### 4.1 Width Context

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

### 4.2 Platform Authority

Fidelity.Platform declarations SHALL supply the target's dimensions, available
representations, storage spaces and boundary constraints. CCS SHALL derive each
section's platform context from the applicable declarations and SHALL retain
their provenance in dependent graph facts. Project configuration selects the
applicable declarations.

### 4.3 Dimension Resolution Flow

```
project selection → platform declarations → CCS saturation (per section)
                                                      ↓
                                          resolved widths, layouts and spaces
                                          as literal annotations on the PSG
                                                      ↓
                                          Alex reads them and witnesses MLIR
                                          with concrete widths, aligned layouts,
                                          space-appropriate ops
```

CCS SHALL resolve required target facts during saturation against each section's
platform declaration. Facts awaiting a declaration MAY remain pending during
elaboration. Concrete representation commitment SHALL require those facts to be
settled; an unresolved required fact SHALL produce a located diagnostic. Alex
SHALL consume the resulting annotations through the witness contract.

---

## 5. Relationship to Ecosystem Components

### 5.1 CCS (Clef Compiler Service)

CCS SHALL check native kind and dimensional identity and retain that information
through elaboration. Arithmetic typing SHALL follow the unit equations;
representation selection SHALL follow the justified operand, intermediate and
result ranges and the applicable declarations. Each value crossing a boundary
SHALL satisfy that boundary's coverage, dimensional and transfer requirements.

### 5.2 Composer / Alex (Code Generation)

Alex witnesses dimensional types that CCS has already resolved. The type mapping in Composer reads the
resolved width and layout annotations on each node and emits the corresponding concrete MLIR types; it does
not consult the platform context to decide a width, and a mapping that did would be computing a fact the
graph already carries ([Program Hypergraph §5](program-hypergraph.md)).

### 5.3 Fidelity.Platform

The platform descriptor layer constructs PlatformContext from fidproj TOML
configuration. Fidelity.Platform declares target-specific dimensions, including
dimensions for FPGA and NPU targets.

### 5.4 Farscape (C/C++ Binding Generator)

A generated Clef binding SHALL represent numeric values using `int` or `float`
with their applicable dimensions. Its binding descriptor SHALL record C ABI
widths, signedness and boundary behavior, including integer-valued `size_t` and
`intptr_t` boundaries. Address values SHALL use typed foreign references under
[FFI Boundary](ffi-boundary.md).

CCS SHALL check the value's justified range against the descriptor's required
representation and SHALL retain the applicable transfer obligations. The binding
descriptor and the evidence establishing the value's range SHALL remain distinct
participants in that check.

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

## 7. Dimensional Identity and Boundaries

### 7.1 WidthDimension Extensibility

A width dimension is a name the platform description declares, not a closed enumeration in the language. A CPU description declares `Pointer` and `Register`; a fabric binding declares the width of each port it governs; an NPU description declares its lane and accumulator widths. `Resolved d` is the platform description's declared width at a site its ABI governs ([NTU Types §1](ntu-types.md)); no width is written by a developer. `Dimensions: Map<WidthDimension, int>` is keyed by the name declared by Fidelity.Platform. Interior widths on any target come from the analysed range and need no dimension at all ([Width Inference](width-inference.md)).

### 7.2 Dimensional Type Identity

Identity is exact on every dimensional component: unit, memory space, region and access kind. No component is subtyped. Width and representation are not components: they are coeffects beside the type ([NTU Types §1](ntu-types.md), [Width Inference §5](width-inference.md)), selected from the analysed range; two integers of different selected widths meeting at an operator are one kind, and their result's width is selected from the result's range. There are no seals and no conversion sites.

- `int` at `Stack` and `int` at `Global` are different types; a value crosses memory spaces only through an explicit transfer.
- Access is invariant. A `ReadWrite` handle where `ReadOnly` is required passes only through `Ptr.asReadOnly` ([Access Kinds](access-kinds.md)), a node in the graph; a `ReadOnly` handle where `ReadWrite` is required is `CCS8022`.

The enumeration sort needs no ordering; every check is equality ([DTS/DMM §2.5](https://arxiv.org/abs/2603.16437)).

### 7.3 BAREWire Contract Verification

CCS SHALL verify that both sides of a BAREWire contract are dimensionally
consistent. The layout description SHALL be interpretable from both platform contexts.

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
