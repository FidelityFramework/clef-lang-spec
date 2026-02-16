# Enrichment and Coeffect Analysis Specification

> **Spec location:** `spec/program-semantic-graph.md` §13-14

## Summary

Added two new sections to the PSG specification (January 2026):

### §13 Enrichment

**Enrichment** is the parent concept for compiler-synthesized PSG structure:
- **Elaboration** (PLT term): Making implicit structure explicit
- **Saturation** (Fidelity term): Filling PSG for lowering

Two categories:
| Category | `Elaboration.Kind` | Purpose |
|----------|-------------------|---------|
| Intrinsic Elaboration | `"Intrinsic"` | Implement intrinsic semantics |
| Baker Saturation | `"Baker"` | Decompose language features |

Metadata keys:
- `Elaboration.Kind` - "Intrinsic" or "Baker"
- `Elaboration.For` - What triggered enrichment
- `Elaboration.Id` - Links related nodes

Source-based nodes have NO elaboration metadata.

### §14 Coeffect Analysis

Computes metadata WITHOUT creating nodes:
- SSA Assignment
- Mutability Analysis
- Yield State Analysis
- Pattern Binding Analysis

Key distinction:
- Enrichment CREATES nodes
- Coeffect analysis ANALYZES existing nodes

Pipeline: `Enrichment → Coeffect Analysis → Lowering`

Enables the control-flow ↔ dataflow pivot for lowering decisions.

## Related

- Implementation: `Clef/src/Compiler/PSGSaturation/SemanticGraph/Elaboration.fs`
- Firefly docs: `docs/PSG_Enrichment_Architecture.md`, `docs/Coeffect_Analysis_Architecture.md`
