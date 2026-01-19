# fsnative.com Site Migration Plan

> **Purpose**: Plan the migration of relevant SpeakEZ blog content to fsnative.com alongside the formal language specification.
> **Created**: January 2026

## 1. Site Architecture Vision

fsnative.com should serve as the authoritative home for F# Native, containing:

1. **Formal Specification** (`/spec/`) - Normative language specification
2. **Commentary** (`/commentary/`) - Accessible explanations of spec concepts
3. **Design Rationale** (`/rationale/`) - Historical design decisions and comparisons
4. **Tutorials** (`/learn/`) - Getting started guides
5. **Blog/News** (`/blog/`) - Ongoing development updates

This follows the **Standard ML model**: The Definition (formal spec) + Commentary (explanatory companion).

---

## 2. SpeakEZ Blog Entries - Migration Catalog

### 2.1 TIER 1: Core Language/Compiler Commentary (MIGRATE FIRST)

These directly explain spec concepts and should become the "Commentary on F# Native":

| Blog Entry | Spec Chapter Companion | Migration Priority |
|------------|----------------------|-------------------|
| **Gaining Closure.md** | closure-representation.md | ⭐⭐⭐ ESSENTIAL |
| **Why Lazy Is Hard.md** | lazy-representation.md | ⭐⭐⭐ ESSENTIAL |
| **Seqing Simplicity.md** | seq-representation.md, seq-operations-representation.md | ⭐⭐⭐ ESSENTIAL |
| **FSharp Native from IL to NTU.md** | native-type-universe.md, type-representation-architecture.md | ⭐⭐⭐ ESSENTIAL |
| **Absorbing Alloy.md** | ntu-types.md (types as intrinsics) | ⭐⭐⭐ ESSENTIAL |
| **Traits Versus Statically Resolved Type Parameters.md** | types-and-type-constraints.md (SRTP sections) | ⭐⭐⭐ ESSENTIAL |
| **ByRef Resolved.md** | (byref type handling) | ⭐⭐⭐ ESSENTIAL |
| **Inferring Memory Lifetimes.md** | memory-regions.md | ⭐⭐⭐ ESSENTIAL |
| **Coeffects And Codata In Firefly.md** | (coeffects, async) | ⭐⭐⭐ ESSENTIAL |
| **Delimited Continuations Fidelitys Turning Point.md** | (continuations, async) | ⭐⭐⭐ ESSENTIAL |

### 2.2 TIER 2: Architecture & Design Rationale (MIGRATE)

These explain architectural decisions and belong in `/rationale/`:

| Blog Entry | Topic Area | Migration Priority |
|------------|-----------|-------------------|
| **Baker A Key Ingredient to Firefly.md** | Type resolution layer | ⭐⭐ HIGH |
| **Building Firefly With Alloy.md** | Historical: library approach | ⭐⭐ HIGH |
| **Hello World Goes Native.md** | End-to-end compilation | ⭐⭐ HIGH |
| **FSharp Memory Management Goes Native.md** | Memory model design | ⭐⭐ HIGH |
| **Memory Management By Choice.md** | Memory model philosophy | ⭐⭐ HIGH |
| **Why FSHarp Is A Natural Fit for MLIR.md** | MLIR choice rationale | ⭐⭐ HIGH |
| **WREN Stack.md** | Desktop architecture vision | ⭐⭐ HIGH |
| **Standing Art FSharp Metaprogramming in Firefly.md** | Quotations/metaprogramming | ⭐⭐ HIGH |
| **Arity On The Side Of Caution.md** | Function calling conventions | ⭐⭐ HIGH |
| **Beyond Zero-Allocation.md** | Allocation strategy | ⭐⭐ HIGH |
| **Dimensional Type Safety.md** | Units of measure | ⭐⭐ HIGH |
| **DCont Inet Duality.md** | Continuation design | ⭐⭐ HIGH |

### 2.3 TIER 3: Fidelity Framework & Platform (MIGRATE)

These cover the broader Fidelity ecosystem:

| Blog Entry | Topic Area | Migration Priority |
|------------|-----------|-------------------|
| **Fidelity Framework a Primer.md** | Framework overview | ⭐ MEDIUM |
| **Fidelity UI Model.md** | UI architecture | ⭐ MEDIUM |
| **Fidelity as AI Refinery.md** | AI inference story | ⭐ MEDIUM |
| **AlloyRx Native Reactivity in Fidelity.md** | Reactive signals | ⭐ MEDIUM |
| **Library Binding in Fidelity Framework.md** | FFI/binding patterns | ⭐ MEDIUM |
| **FSharp On Metal - Fidelity Lowered to STM32.md** | Embedded targets | ⭐ MEDIUM |
| **FSharp On Metal Revisited.md** | Embedded updates | ⭐ MEDIUM |
| **From Dotnet To Fidelity Concurrency.md** | Concurrency model | ⭐ MEDIUM |
| **Scaling FidelityUI.md** | UI scaling | ⭐ MEDIUM |
| **Window Layout with Fidelity.md** | Desktop layout | ⭐ MEDIUM |
| **Leveraging Fabulous for Native UI.md** | MVU pattern | ⭐ MEDIUM |
| **RAII in Olivier And Prospero.md** | Resource management | ⭐ MEDIUM |

### 2.4 TIER 4: Ecosystem & Related Projects (CONSIDER)

These cover related projects (Farscape, BAREWire) - may belong on fsnative.com or stay on SpeakEZ:

| Blog Entry | Topic Area | Decision |
|------------|-----------|----------|
| **Binding Fs To Cpp in Farscape.md** | C++ interop | CONSIDER |
| **Farscape Modular Entry Points.md** | Distributed compute | CONSIDER |
| **The Farscape Bridge.md** | Distributed architecture | CONSIDER |
| **Getting The Signal With BAREWire.md** | Binary serialization | CONSIDER |
| **Bringing Posit Arithmetic to Fsharp.md** | Numeric types | CONSIDER |

### 2.5 TIER 5: Optimization & Hardware (CONSIDER)

Deep technical content that may fit fsnative.com:

| Blog Entry | Topic Area | Decision |
|------------|-----------|----------|
| **Cache Aware Compilation CPU.md** | CPU optimization | CONSIDER |
| **Cache Aware Compilation GPU.md** | GPU optimization | CONSIDER |
| **Context Aware Compilation.md** | Compilation strategy | CONSIDER |
| **Hardware Lessons From LISP.md** | Hardware co-design | CONSIDER |
| **RDNA and the Unified Memory Desktop.md** | Memory architecture | CONSIDER |
| **RDMA Accelerating Network Comms.md** | Network optimization | CONSIDER |
| **Next-Generation Memory Coherence.md** | Memory coherence | CONSIDER |
| **Speed And Safety With Graph Coloring.md** | Register allocation | CONSIDER |
| **Intelligent Tree-Shaking.md** | Dead code elimination | CONSIDER |
| **Proof-Aware Compilation.md** | Verification | CONSIDER |
| **MLIR Testing with Teeth.md** | Testing infrastructure | CONSIDER |

### 2.6 KEEP ON SPEAKEZ (Not F# Native specific)

These are broader SpeakEZ/company content:

- AI/ML vision articles (Beyond Transformers, Advent of Neuromorphic AI, etc.)
- Security articles (Quantum WireGuard, Zero Trust, etc.)
- General philosophy (How Our Innovations Express Our Values, etc.)
- .NET-specific tooling (FSharp Autocomplete Integration, Victor CLI)
- Other language comparisons (Musing on Mojo, Rust Revisited, Pondering Python)

---

## 3. Spec + Commentary Pairing

The goal is for each major spec chapter to have a companion commentary article:

| Spec Chapter | Commentary Article | Status |
|--------------|-------------------|--------|
| closure-representation.md | Gaining Closure | ✅ EXISTS |
| lazy-representation.md | Why Lazy Is Hard | ✅ EXISTS |
| seq-representation.md | Seq'ing Simplicity | ✅ EXISTS |
| seq-operations-representation.md | (part of Seq'ing Simplicity) | ✅ EXISTS |
| native-type-universe.md | FSharp Native from IL to NTU | ✅ EXISTS |
| type-representation-architecture.md | (needs pairing) | ❌ GAP |
| backend-lowering-architecture.md | Why F# Is A Natural Fit for MLIR | ✅ EXISTS |
| ntu-conversion-model.md | (needs pairing) | ❌ GAP |
| memory-regions.md | Inferring Memory Lifetimes | ✅ EXISTS |
| types-and-type-constraints.md (SRTP) | Traits vs SRTP | ✅ EXISTS |
| platform-bindings.md | (needs pairing) | ❌ GAP |

---

## 4. Content Transformation Notes

When migrating blog entries to fsnative.com commentary:

### 4.1 Keep As-Is
- Narrative explanatory style
- Code examples
- Diagrams (Mermaid)
- Academic references
- Comparisons to other languages

### 4.2 Remove/Adapt
- SpeakEZ-specific branding
- Hugo frontmatter (replace with fsnative.com format)
- Date-specific references ("In March 2025...")
- References to external SpeakEZ services

### 4.3 Add
- Cross-references to normative spec sections: "See §3.1 of closure-representation.md for the normative memory layout"
- "This article explains the design rationale for [spec section]"
- Navigation to related commentary

---

## 5. Site Structure Proposal

```
fsnative.com/
├── spec/                           # Formal specification
│   ├── index.md                    # Spec table of contents
│   ├── types-and-type-constraints.md
│   ├── closure-representation.md
│   ├── lazy-representation.md
│   └── ...
│
├── commentary/                     # "Commentary on F# Native"
│   ├── index.md                    # Commentary overview
│   ├── gaining-closure.md          # Companion to closure-representation
│   ├── why-lazy-is-hard.md         # Companion to lazy-representation
│   ├── seqing-simplicity.md        # Companion to seq chapters
│   └── ...
│
├── rationale/                      # Design decisions & history
│   ├── index.md
│   ├── absorbing-alloy.md          # Types as intrinsics decision
│   ├── srtp-vs-traits.md           # SRTP design philosophy
│   └── ...
│
├── learn/                          # Getting started
│   ├── hello-world.md
│   ├── memory-model.md
│   └── ...
│
└── blog/                           # Ongoing updates
    └── ...
```

---

## 6. Action Items

1. [ ] Finalize blog entry migration decisions (Tiers 1-3 confirmed, Tiers 4-5 reviewed)
2. [ ] Set up fsnative.com site infrastructure
3. [ ] Migrate Tier 1 commentary articles
4. [ ] Restructure spec chapters (normative-only)
5. [ ] Create rationale/ directory from extracted spec content
6. [ ] Cross-reference spec ↔ commentary
7. [ ] Update Firefly CLAUDE.md to reference fsnative.com

---

## 7. Key Insight

The SpeakEZ blog already contains an excellent "Commentary on F# Native" - it just needs to be:
1. **Consolidated** on fsnative.com
2. **Cross-referenced** with the formal spec
3. **Separated** from the normative spec text

This follows the Standard ML precedent: "The Definition" is terse and formal; "The Commentary" is accessible and explanatory. Both are valuable; they serve different audiences.
