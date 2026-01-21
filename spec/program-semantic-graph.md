# Program Semantic Graph

> **Status**: Normative
> **Last Updated**: 2026-01-19 (Added Section 12: PSG Saturation)

## 1. Overview

The Program Semantic Graph (PSG) is the unified intermediate representation produced by FNCS. It carries semantic information from type checking through to code generation, preserving the meaning of F# programs in a form suitable for native compilation.

### 1.1 Architectural Heritage

The PSG draws on the **nanopass** tradition established in Standard ML and Scheme compiler research. Where traditional compilers make large, monolithic transformations between representations, nanopass architecture decomposes compilation into many small, single-purpose passes—each doing one thing well.

This heritage distinguishes F# Native from the F# Compiler Services (FCS) design. FCS was built for .NET integration, where semantic information flows to the CLR runtime. The PSG instead preserves semantic information through to native code generation, carrying proofs about types, memory, and execution that the CLR would normally handle implicitly.

**Key insight**: Traversal information is itself semantic information. The classification of what executes during module initialization versus what constitutes a definition versus what serves as an entry point—these are semantic facts about the program that flow through the pipeline as first-class data.

### 1.2 Relationship to FCS

The PSG consumes both the syntax tree (`SynExpr`) and typed tree (`FSharpExpr`) from FCS:

| FCS Output | PSG Usage |
|------------|-----------|
| `SynExpr` | Structural skeleton, source ranges |
| `FSharpExpr` | Resolved types, SRTP resolution, semantic edges |

The typed tree overlay via zipper captures information that exists only after type inference and constraint solving—particularly SRTP (Statically Resolved Type Parameter) resolution, which the syntax tree cannot express.

## 2. SemanticGraph Structure

The `SemanticGraph` record is the complete result of PSG construction:

```fsharp
type SemanticGraph = {
    Nodes: Map<NodeId, SemanticNode>
    EntryPoints: NodeId list
    Modules: Map<ModulePath, NodeId list>
    Types: Lazy<Map<string, NodeId>>
    Platform: PlatformContext option
    ModuleClassifications: Lazy<Map<NodeId, ModuleClassification>>
}
```

### 2.1 Field Specifications

| Field | Type | Description |
|-------|------|-------------|
| `Nodes` | `Map<NodeId, SemanticNode>` | All semantic nodes indexed by unique identifier |
| `EntryPoints` | `NodeId list` | Nodes marked as program entry points |
| `Modules` | `Map<ModulePath, NodeId list>` | Module contents indexed by path |
| `Types` | `Lazy<Map<string, NodeId>>` | Type definitions, lazily indexed |
| `Platform` | `PlatformContext option` | Target platform for NTU resolution |
| `ModuleClassifications` | `Lazy<Map<NodeId, ModuleClassification>>` | Module structure analysis, lazily computed |

### 2.2 Lazy Coeffects

The `Types` and `ModuleClassifications` fields use the **codata pattern**—they are computed on first observation rather than eagerly during construction.

This design serves two purposes:

1. **Efficiency**: Classification is only computed when Alex needs it
2. **Separation of concerns**: FNCS builds the graph; consumers decide what derived information they need

The lazy fields represent **coeffects**—observations about the graph that can be computed from its structure but are not part of the primary construction.

## 3. SemanticNode Structure

Each node in the PSG carries rich semantic information:

```fsharp
type SemanticNode = {
    Id: NodeId
    Kind: SemanticKind
    Range: SourceRange
    Type: NativeType
    SRTPResolution: WitnessResolution option
    ArenaAffinity: ArenaAffinity
    LayoutHint: TypeLayout option
    Children: NodeId list
    Parent: NodeId option
    Metadata: Map<string, MetadataValue>
    IsReachable: bool
    EmissionStrategy: EmissionStrategy
}
```

### 3.1 Field Specifications

| Field | Type | Description |
|-------|------|-------------|
| `Id` | `NodeId` | Unique identifier within the graph |
| `Kind` | `SemanticKind` | What construct this node represents |
| `Range` | `SourceRange` | Source location for diagnostics |
| `Type` | `NativeType` | Resolved type (attached during construction) |
| `SRTPResolution` | `WitnessResolution option` | Resolved SRTP constraint, if any |
| `ArenaAffinity` | `ArenaAffinity` | Memory region affinity |
| `LayoutHint` | `TypeLayout option` | Size/alignment hints |
| `Children` | `NodeId list` | Structural children |
| `Parent` | `NodeId option` | Parent for navigation |
| `Metadata` | `Map<string, MetadataValue>` | Typed extensible metadata |
| `IsReachable` | `bool` | Soft-delete marker for reachability |
| `EmissionStrategy` | `EmissionStrategy` | How to emit during code generation |

### 3.2 Type Attachment

**NORMATIVE**: Types SHALL be attached to nodes during PSG construction, not post-hoc.

This ensures that every node carries its resolved type from the moment of creation, enabling downstream passes to make decisions based on complete type information.

### 3.3 SRTP Resolution

The `SRTPResolution` field captures the compile-time resolution of statically resolved type parameters:

```fsharp
type WitnessResolution = {
    Operator: string              // e.g., "$", "+"
    ArgType: NativeType           // The type on which it's resolved
    ResolvedMember: string        // Fully qualified member name
    Implementation: WitnessImplementation
}
```

This information comes from the typed tree (`FSharpExpr.TraitCall`) and is not available in the syntax tree. The PSG captures it during the typed tree overlay phase.

## 4. SemanticKind

The `SemanticKind` discriminated union classifies each node. Key categories include:

### 4.1 Bindings and Control

| Kind | Description |
|------|-------------|
| `Binding` | Let binding with mutability and entry point flags |
| `Lambda` | Function with parameters, captures, and context |
| `PatternBinding` | Variable bound by pattern matching |

### 4.2 Core Expressions

| Kind | Description |
|------|-------------|
| `Literal` | Constant value |
| `VarRef` | Reference to a binding |
| `Application` | Function application |
| `TypeAnnotation` | Explicit type annotation |

### 4.3 Control Flow

| Kind | Description |
|------|-------------|
| `Match` | Pattern matching expression |
| `IfThenElse` | Conditional expression |
| `WhileLoop` | While loop |
| `ForLoop` | Numeric for loop |
| `ForEach` | Collection iteration |
| `Sequential` | Sequential composition |
| `TryWith` | Exception handling |
| `TryFinally` | Cleanup handler |

### 4.4 Data Structures

| Kind | Description |
|------|-------------|
| `TupleExpr` | Tuple construction |
| `ArrayExpr` | Array literal |
| `ListExpr` | List literal |
| `RecordExpr` | Record construction |
| `UnionCase` | Union case construction |

### 4.5 Compiler Features

| Kind | Description |
|------|-------------|
| `Intrinsic` | FNCS intrinsic operation |
| `TraitCall` | SRTP trait invocation |
| `PlatformBinding` | Platform-specific binding |
| `LazyExpr` | Lazy computation (PRD-14) |
| `SeqExpr` | Sequence expression (PRD-15) |

## 5. Emission Strategy

The `EmissionStrategy` field tells Alex how to emit each node:

```fsharp
type EmissionStrategy =
    | Inline
    | SeparateFunction of captureCount: int
    | MainPrologue
```

| Strategy | Description |
|----------|-------------|
| `Inline` | Emit inline at point of use |
| `SeparateFunction` | Emit as separate function (lambdas, seq expressions) |
| `MainPrologue` | Emit in main function prologue (module initialization) |

**NORMATIVE**: Emission strategy SHALL be determined by FNCS during construction. Alex SHALL observe this strategy without recomputing it.

This architectural decision moves traversal logic upstream to FNCS, where semantic context is fully available.

## 6. Module Classification

Module classification analyzes the structure of each module:

```fsharp
type ModuleClassification = {
    Name: string
    ModuleInit: NodeId list
    Definitions: NodeId list
    EntryPoint: NodeId option
}
```

| Field | Description |
|-------|-------------|
| `Name` | Module name |
| `ModuleInit` | Bindings that execute at module initialization |
| `Definitions` | Regular definitions (functions, types) |
| `EntryPoint` | The module's entry point, if any |

### 6.1 Classification Algorithm

Module classification is computed from `EmissionStrategy`:

1. Nodes with `MainPrologue` strategy → `ModuleInit`
2. Nodes with `Inline` or `SeparateFunction` strategy → `Definitions`
3. Bindings marked `isEntryPoint` → `EntryPoint`

This classification is semantic traversal information—it describes how the program executes, not just what it contains.

## 7. Intrinsic Metadata

Intrinsic operations carry structured metadata:

```fsharp
type IntrinsicInfo = {
    Module: IntrinsicModule
    Operation: string
    Category: IntrinsicCategory
    FullName: string
}
```

**NORMATIVE**: Intrinsic dispatch SHALL use structured `IntrinsicInfo`, not string matching on symbol names.

This ensures that Alex can dispatch on intrinsics without knowledge of namespace structure or symbol naming conventions.

### 7.1 Intrinsic Modules

| Module | Description | Provider |
|--------|-------------|----------|
| `Sys` | System calls | Compiler |
| `NativePtr` | Pointer operations | Compiler |
| `NativeStr` | String operations | Compiler |
| `Array` | Array operations | Compiler |
| `Math` | Mathematical functions | Compiler |
| `Lazy` | Lazy computation (PRD-14) | Compiler |
| `Seq` | Sequences (PRD-15) | Compiler |
| `Signal`, `Effect`, `Memo`, `Batch` | Reactive primitives | Library |

The distinction between compiler-provided and library-backed intrinsics is significant: compiler-provided intrinsics have implementations generated by Alex, while library-backed intrinsics dispatch to native library functions.

## 8. Reachability

### 8.1 Soft-Delete Model

**NORMATIVE**: Unreachable nodes SHALL be marked via `IsReachable = false`, not physically removed.

This preserves the graph structure for:
- Typed tree zipper navigation
- Diagnostic reporting
- Debugging and inspection

### 8.2 Reachability Computation

Reachability flows from entry points through structural and semantic edges:

1. Entry points are reachable
2. Children of reachable nodes are reachable
3. Definitions referenced by reachable `VarRef` nodes are reachable
4. Type definitions used by reachable nodes are reachable

## 9. Platform Context

The platform context carries target-specific information:

```fsharp
type PlatformContext = {
    PlatformId: string
    WordSize: int
    PointerSize: int
    PointerAlign: int
    PlatformLibraryPath: string option
    Predicates: Map<PlatformPredicate, bool>
}
```

This information flows from the project file through FNCS to the PSG, enabling platform-aware type resolution and code generation.

## 10. Invariants

The following invariants SHALL hold for any valid PSG:

1. **Node identity**: Each `NodeId` maps to exactly one `SemanticNode`
2. **Parent consistency**: If node A lists B as parent, B lists A as child
3. **Type completeness**: Every node has a resolved `NativeType`
4. **Reachability closure**: If A is reachable and references B, B is reachable
5. **Entry point marking**: Entry points are marked in both `EntryPoints` list and node `Kind`
6. **Emission strategy presence**: Every node has an `EmissionStrategy`

## 11. Traversal

The PSG supports multiple traversal patterns:

| Pattern | Use Case |
|---------|----------|
| `foldWithLambdaPreBind` | Follow semantic dependencies (definitions before uses) |
| `foldWithSCFRegions` | Respect SCF region boundaries for control flow |
| Pre-order structural | Visit parents before children |
| Post-order structural | Visit children before parents |

Alex uses these traversal patterns to generate MLIR in the correct order, respecting both structural and semantic dependencies.

## 12. PSG Saturation

### 12.1 The Saturation Principle

**NORMATIVE**: FNCS SHALL saturate the PSG with all semantic structure required for compilation, including synthetic constructs not directly expressed in source code.

The term **saturation** refers to making the PSG semantically complete—containing all information needed for downstream compilation without requiring structure synthesis during code generation.

### 12.2 Motivation

Certain F# constructs require runtime structures that don't appear directly in source code:

| Source Construct | Synthetic Structure | Location in PSG |
|------------------|---------------------|-----------------|
| `seq { }` | MoveNext state machine | SeqStateMachine nodes |
| `async { }` | Continuation state machine | (Future) AsyncStateMachine nodes |
| Pattern matching | Decision tree / switch | Match decomposition |

Without saturation, code generators must synthesize these structures during emission, leading to:
- Mutable state tracking during emission
- Structure-building logic in the wrong architectural layer
- Coupling between emission and semantic analysis
- SSA allocation during emission rather than in nanopasses

### 12.3 Architectural Separation

The saturation principle enforces clean layer separation:

| Layer | Responsibility | NOT Responsible For |
|-------|----------------|---------------------|
| **FNCS/PSGSaturation** | Build all semantic structure | Target-specific details |
| **PSGElaboration** | Add target-specific coeffects (SSA, platform bindings) | Semantic structure |
| **Alex/FNCSTransfer** | Witness and emit | Structure building, SSA allocation |

**Key insight**: Structure building is a semantic concern. FNCS knows F# semantics. Code generators should witness structure, not build it.

### 12.4 Saturation Pass Architecture

PSG construction flows through saturation phases:

```
Phase 1: Structural Construction    SynExpr → PSG with basic nodes
Phase 2: Symbol Correlation         + FSharpSymbol attachments
Phase 3: Reachability Analysis      + IsReachable marks
Phase 4: Typed Tree Overlay         + Types, SRTP resolution
Phase 5: PSG Saturation             + Synthetic structures (MoveNext, etc.)
```

The saturation phase (Phase 5) transforms high-level semantic constructs into explicit operational structure:

```
SeqExpr(body, captures)
    ↓ PSG Saturation
SeqStateMachine {
    StateVariables: [...]
    Captures: [...]
    InitBlock: NodeId
    CheckBlock: NodeId
    YieldBlock: NodeId
    PostYieldBlock: NodeId
    DoneBlock: NodeId
}
```

### 12.5 Sequence Expression Saturation

For `seq { }` expressions, saturation produces explicit state machine structure:

**Input (pre-saturation)**:
```fsharp
SeqExpr of body: NodeId * captures: CaptureInfo list
```

**Output (post-saturation)**:
```fsharp
SeqStateMachine of {
    BodyKind: SeqBodyKind           // Sequential or WhileBased
    StateVariables: StateVar list   // Internal mutable state
    Captures: CaptureInfo list      // Captured values
    InitBlock: BlockSpec            // State 0 initialization
    CheckBlock: BlockSpec           // While condition evaluation
    YieldBlock: BlockSpec           // Value production
    PostYieldBlock: BlockSpec       // Post-yield state transitions
    ConditionalYield: ConditionSpec option  // If-guarded yield
}
```

This explicit structure enables:
- SSAAssignment to walk block nodes and assign SSAs normally
- FNCSTransfer to witness structure without synthesis
- Clear separation between semantic analysis and code generation

### 12.6 Comparison to F# Compiler

F# Native's saturation phase parallels `LowerSequenceExpressions.fs` in the F# compiler:

| F# Compiler | F# Native | Purpose |
|-------------|-----------|---------|
| `CheckSequenceExpressions.fs` | PSG Builder | Initial semantic structure |
| `LowerSequenceExpressions.fs` | **PSG Saturation** | State machine elaboration |
| `IlxGen.fs` | FNCSTransfer | Target code emission |

The key insight from nanopass architecture: saturation happens at the **language level** (PSG), not in code generation.

### 12.7 Invariants for Saturated PSG

After saturation, the following additional invariants SHALL hold:

1. **Synthetic completeness**: All constructs requiring runtime structure have explicit nodes
2. **Block structure**: State machine blocks are explicit, traversable nodes
3. **No deferred synthesis**: Code generators need not build structure during emission
4. **SSA assignability**: All nodes can have SSAs assigned by walking structure

## See Also

- [Type Representation Architecture](type-representation-architecture.md) - NativeType structure and lookup
- [Backend Lowering Architecture](backend-lowering-architecture.md) - PSG to MLIR lowering
- [Closure Representation](closure-representation.md) - Lambda capture in PSG
- [Lazy Representation](lazy-representation.md) - LazyExpr semantics
- [Seq Representation](seq-representation.md) - SeqExpr semantics
