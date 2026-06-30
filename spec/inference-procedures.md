---
title: "Inference Procedures"
weight: 610
category: Compiler
status: normative
---

This chapter provides an overview of the inference procedures used during Clef type checking. The procedures are organized into focused sub-chapters for clarity.

## Overview

Type inference in Clef follows standard Hindley-Milner inference extended with:
- Overload resolution for methods and functions
- Statically Resolved Type Parameters (SRTP)
- [Member constraints](types-and-type-constraints.md)
- Constraint propagation

The inference process involves:
1. **Name Resolution** - Resolving identifiers to their definitions
2. **Application Resolution** - Resolving function and method applications
3. **Constraint Solving** - Solving type constraints to determine concrete types
4. **Definition Checking** - Verifying and elaborating function/value/member definitions

## Sub-Chapters

### [Name Resolution](inference-name-resolution.md)

Covers how Clef resolves names in various contexts:
- Name environments and tables
- Module and namespace path resolution
- Expression, pattern, and type name resolution
- Type variable resolution
- Field label resolution

### [Application Resolution](inference-application-resolution.md)

Covers resolution of application expressions:
- Dot-notation expressions
- Unqualified, item-qualified, and expression-qualified lookup
- Function application resolution
- Method application resolution
- Implicit flexibility insertion

### [Constraint Solving and Definition Checking](inference-constraint-solving.md)

Covers constraint solving and definition elaboration:
- Equational and subtype constraint solving
- Member constraint solving
- Function, value, and member definition checking
- Recursive definition processing
- Generalization and condensation

### [Supplementary Procedures](inference-supplementary.md)

Covers additional inference procedures:
- Dispatch slot inference and checking
- Byref safety analysis
- Mutable local promotion
- Arity inference
- Common method constraints
