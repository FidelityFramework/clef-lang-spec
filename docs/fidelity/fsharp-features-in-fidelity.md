# F# Features in the Fidelity Compilation Pipeline

> **Purpose**: This document provides a high-level summary of how three key F# features enable the Fidelity compilation architecture. Each section includes cross-references to detailed documentation.

## Overview

Three F# features form the architectural backbone of the Fidelity compilation pipeline:

| Feature | Role in Fidelity | Key Application |
|---------|------------------|-----------------|
| **Quotations** | Encode constraints as data | Memory descriptors, peripheral bindings |
| **Active Patterns** | Compositional recognition | PSG node classification, pattern matching |
| **Computation Expressions** | Structured control flow | Continuation-based compilation, builders |

These features work together in a unified architecture where:
- Quotations carry **semantic information** through the pipeline
- Active patterns enable **structural recognition** in PSG traversal
- Computation expressions provide the **control flow abstraction** for compilation phases

---

## Quotations (`Expr<'T>`)

### Role in Fidelity

Quotations encode memory constraints and peripheral descriptors as **first-class data** that flows through the PSG. Rather than relying on attributes or string-based annotations, quotations carry type-safe semantic information that the nanopass pipeline can inspect and transform.

### Key Application: Memory Descriptors

Farscape generates quotations that describe peripheral memory layouts:

```fsharp
let gpioQuotation: Expr<PeripheralDescriptor> = <@
    { Name = "GPIO"
      Instances = Map.ofList [("GPIOA", 0x48000000un); ("GPIOB", 0x48000400un)]
      Layout = gpioLayout
      MemoryRegion = Peripheral }
@>
```

### Why Quotations

1. **Type-safe**: The F# compiler verifies quotation structure
2. **Inspectable**: PSG nanopasses can examine quotation content
3. **Transformable**: Quotations can be manipulated programmatically
4. **BCL-free**: No runtime reflection required

### Detailed Documentation

| Document | Location | Contents |
|----------|----------|----------|
| Quotation-Based Memory Architecture | `Firefly/docs/Quotation_Based_Memory_Architecture.md` | Full architecture, MemoryModel record |
| Memory Interlock Requirements | `Firefly/docs/Memory_Interlock_Requirements.md` | Farscape → BAREWire integration |
| Hardware Descriptors | `BAREWire/docs/08 Hardware Descriptors.md` | PeripheralDescriptor types |

---

## Active Patterns

### Role in Fidelity

Active patterns provide **compositional recognition** for PSG nodes. They enable the typed tree zipper and Alex traversal to classify nodes structurally rather than through string matching or type discrimination.

### Key Application: PSG Node Recognition

```fsharp
// Recognize peripheral memory access
let (|PeripheralAccess|_|) (node: PSGNode) =
    match node with
    | CallToExtern name args when isPeripheralBinding name ->
        Some (extractPeripheralInfo args)
    | _ -> None

// Recognize SRTP dispatch
let (|SRTPDispatch|_|) (node: PSGNode) =
    match node.TypeCorrelation with
    | Some { SRTPResolution = Some srtp } -> Some srtp
    | _ -> None

// Use in traversal
match currentNode with
| PeripheralAccess info -> emitVolatileAccess info
| SRTPDispatch srtp -> emitResolvedCall srtp
| _ -> emitDefault node
```

### Why Active Patterns

1. **Compositional**: Patterns combine with `&` and `|`
2. **Type-safe**: Return type determines extracted information
3. **Encapsulated**: Recognition logic is localized
4. **Testable**: Patterns can be unit tested independently

### Categories in Fidelity

| Category | Purpose | Example |
|----------|---------|---------|
| **Memory patterns** | Recognize memory operations | `PeripheralAccess`, `StackAlloc` |
| **Type patterns** | Classify types | `NativeString`, `FatPointer` |
| **Operation patterns** | Identify operations | `SRTPDispatch`, `PlatformBinding` |
| **Structure patterns** | Match PSG structure | `FunctionCall`, `LetBinding` |

### Detailed Documentation

| Document | Location | Contents |
|----------|----------|----------|
| TypedTree Zipper Design | `Firefly/docs/TypedTree_Zipper_Design.md` | Zipper correlation patterns |
| XParsec PSG Architecture | `Firefly/docs/XParsec_PSG_Architecture.md` | Pattern combinators |
| Baker Architecture | `Firefly/docs/Baker_Architecture.md` | SRTP resolution patterns |

---

## Computation Expressions

### Role in Fidelity

Computation expressions (CEs) serve two distinct roles:

1. **User-facing**: The sugared form for async, result handling, and effects in application code
2. **Compiler-internal**: The structured abstraction for compilation phases and builders

### Key Insight: Continuations in Disguise

Every `let!` in a computation expression is syntactic sugar for continuation capture:

```fsharp
// CE syntax
maybe {
    let! x = someOption    // Bind(someOption, fun x -> ...)
    let! y = otherOption   // Bind(otherOption, fun y -> ...)
    return x + y
}

// Desugars to
builder.Bind(someOption, fun x ->
    builder.Bind(otherOption, fun y ->
        builder.Return(x + y)))
```

This is why CEs compile naturally to the DCont dialect - they already express delimited continuations.

### DCont/Inet Duality

The Fidelity compilation strategy depends on computation pattern:

| Pattern | Dialect | Compilation Strategy |
|---------|---------|---------------------|
| **Sequential effects** (async, state) | DCont | Preserve continuations |
| **Parallel pure** (validated, reader) | Inet | Compile to data flow |
| **Mixed** | Both | Analyze and split |

```fsharp
// Async CE → DCont dialect (sequential, effectful)
async {
    let! data = readAsync()    // Suspension point → dcont.shift
    return process data
}

// Validated CE → Inet dialect (parallel, pure)
validated {
    let! a = validateA input
    and! b = validateB input   // Independent, can parallelize
    return combine a b
}
```

### Key Application: MLIR Builder

The MLIR builder is itself a computation expression:

```fsharp
let emitFunction (node: PSGNode) : MLIR<Val> = mlir {
    let! funcType = deriveType node
    let! entry = createBlock "entry"
    do! setInsertionPoint entry
    let! result = emitBody node.Body
    do! emitReturn result
    return result
}
```

### Detailed Documentation

| Document | Location | Contents |
|----------|----------|----------|
| DCont Pipeline Roadmap | `Firefly/docs/DCont_Pipeline_Roadmap.md` | DCont/Inet compilation |
| Architecture Canonical | `Firefly/docs/Architecture_Canonical.md` | MLIR builder patterns |
| Coeffects And Codata | `SpeakEZ/blog/Coeffects And Codata In Firefly.md` | Coeffect tracking |
| DCont Inet Duality | `SpeakEZ/blog/DCont Inet Duality.md` | CE decomposition theory |

---

## The Unified Architecture

These three features compose into a coherent pipeline:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Feature Integration Flow                          │
│                                                                      │
│  Farscape Output                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ Expr<PeripheralDescriptor>   ← Quotations encode constraints   │ │
│  │ (|PeripheralAccess|_|)       ← Active patterns for recognition │ │
│  │ MemoryModel record           ← Integration surface             │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              ↓                                       │
│  PSG Construction (Baker)                                            │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ TypedTreeZipper uses active patterns to correlate trees        │ │
│  │ Quotation content attached to PSG nodes                        │ │
│  │ SRTP patterns recognize trait calls                            │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              ↓                                       │
│  Alex Traversal                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ Zipper + XParsec patterns match PSG structure                  │ │
│  │ mlir { } CE accumulates MLIR output                            │ │
│  │ Quotation data guides code generation                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              ↓                                       │
│  DCont/Inet Compilation                                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ CE patterns → DCont (sequential) or Inet (parallel)            │ │
│  │ Coeffects determine compilation strategy                       │ │
│  │ Continuations preserved or compiled to state machines          │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Pure F# Conventions

The Fidelity ecosystem uses pure F# patterns, avoiding BCL dependencies:

| Use | Avoid |
|-----|-------|
| Record types | `interface` declarations |
| Discriminated unions | Abstract classes |
| Active patterns | Reflection-based matching |
| Computation expressions | Callback-based APIs |
| Module functions | Static methods |
| `voption` | Nullable types |

This enables the quotation, active pattern, and CE features to work without BCL contamination.

---

## Cross-Reference Index

### By Feature

| Feature | Primary Documentation |
|---------|----------------------|
| Quotations | `Firefly/docs/Quotation_Based_Memory_Architecture.md` |
| Active Patterns | `Firefly/docs/TypedTree_Zipper_Design.md`, `XParsec_PSG_Architecture.md` |
| Computation Expressions | `Firefly/docs/DCont_Pipeline_Roadmap.md` |

### By Component

| Component | F# Features Used |
|-----------|------------------|
| Farscape | Quotations (output), Active Patterns (parsing) |
| BAREWire | Quotations (descriptors) |
| Firefly/Baker | Active Patterns (correlation), CEs (builders) |
| Firefly/Alex | Active Patterns (traversal), CEs (MLIR builder) |

### By Concept

| Concept | Documentation |
|---------|---------------|
| Memory constraints | `Quotation_Based_Memory_Architecture.md` |
| PSG traversal | `XParsec_PSG_Architecture.md` |
| SRTP resolution | `Baker_Architecture.md` |
| Continuation compilation | `DCont_Pipeline_Roadmap.md` |
| Coeffects | `SpeakEZ/blog/Coeffects And Codata In Firefly.md` |
