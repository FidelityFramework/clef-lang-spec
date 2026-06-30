---
title: "Program Structure"
weight: 20
category: Language
status: normative
---

## Compilation Inputs

The inputs to the Clef Compiler Service (CCS) and the `clef` driver consist of:

- **Implementation files**, with extension `.clef`. These conform to grammar element `implementation-file` in [§](program-structure-and-execution.md#implementation-files).
- **Interactive files**, with extension `.clefx`. These are used by the Clef Interactive REPL (`clefx`) and conform to grammar element `script-file` in [§](program-structure-and-execution.md#script-files). The `.clefx` extension marks Clef made *executable* in a REPL/script workflow — the `x` parallels F#'s `.fsx` in role while departing from its runtime-bound model (no reflection, no managed runtime). Clef has no separate interface-file concept, so no `.clefi`-style extension is reserved.
- **Script fragments** for the Clef Interactive environment, conforming to grammar element `script-fragment` and separated by `;;` tokens at the prompt.
- **Library dependencies** specified in the project file (`.fidproj`) and resolved at compile time from source packages or pre-compiled native libraries.
- **Compiler directives** such as `#nowarn`.

> **NORMATIVE**: Clef does not recognize separate signature files. Module signatures, when present, are declared inline within the implementation file using `signature ... end` blocks (see [§](namespace-and-module-signatures.md)).

The `FIDELITY` compilation symbol is defined for input that CCS has processed. The `INTERACTIVE` symbol is defined within the Clef Interactive environment.

## Compilation Pipeline

CCS processes source files through the following pipeline:

1. **Decoding**. Each file is decoded into a stream of UTF-8 Unicode characters.
2. **Tokenization**. The character stream is broken into a token stream by the lexical analysis described in [§](lexical-analysis.md#lexical-analysis).
3. **Lexical Filtering**. The token stream is filtered by the offside-rule state machine described in [§](lexical-filtering.md#lexical-filtering). Indentation determines structure.
4. **Parsing**. The augmented token stream is parsed according to the grammar specification in this document.
5. **Dependency Analysis**. Before type checking, CCS performs a syntactic dependency-analysis pass over all parsed files: it builds an export map of declared names per file, resolves cross-file identifier references against that map, and computes strongly-connected components of the file-level reference graph. The result is a topological order of compilation units, where each unit is either a single file or a mutually-recursive group of files.

   > **NORMATIVE**: CCS SHALL compute compilation order by syntactic dependency analysis. Source files MAY be presented to the compiler in any order; the developer SHALL NOT need to declare file order. This aligns Clef with established practice in mature ML-family ecosystems (Haskell's GHC, OCaml's `dune`-driven module system, Roc) and modern statically-typed languages where the compiler reconciles ordering and the developer expresses domain logic.

6. **Importing**. Library dependencies are resolved from source packages or pre-compiled native libraries and added to the initial name resolution environment ([§](inference-name-resolution.md#name-resolution)).

   > **Clef Note**: Clef does not use CLI assemblies. Type providers are not available in native compilation.
7. **Checking**. The dependency-ordered compilation units are type-checked. Checking involves Name Resolution, Constraint Solving ([§](inference-constraint-solving.md#generalization)), and Generalization, as well as the application of other rules described in this specification. After processing of all compilation units is complete, all type-inference variables have been either generalized or resolved.
8. **Elaboration**. One result of checking is an elaborated program graph that contains elaborated declarations, expressions, and types. The elaborated graph is the Program Semantic Graph (PSG, [§](program-semantic-graph.md)) handed to downstream stages of the Composer pipeline for native code generation.

   > **Clef Note**: Clef does not support runtime reflection. Elaborated forms are used for native code generation, not CLI metadata emission.
9. **Execution**. Elaborated program fragments produce a native binary whose static initializers run at startup in topological dependency order ([§](program-structure-and-execution.md#program-execution)).
