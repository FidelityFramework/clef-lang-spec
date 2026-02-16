# Spec Audit Against Native Type Universe Document

## Audit Date: 2024-12-30

## Reference Document
`Clef/docs/fidelity/native-type-universe.md` - The comprehensive native type specification (moved from clef-lang-spec)

## Summary

### Completed ("Needs Revision" Chapters) ✓

All four "Needs revision" chapters have been revised to align with native type universe principles:

| Chapter | File | Status | Alignment |
|---------|------|--------|-----------|
| Introduction | `introduction.md` | ✓ Revised | Good alignment |
| Types and Type Constraints | `types-and-type-constraints.md` | ✓ Revised | Good alignment |
| Program Structure and Execution | `program-structure-and-execution.md` | ✓ Revised | Good alignment |
| Special Attributes and Types | `special-attributes-and-types.md` | ✓ Revised | Good alignment |

### "New" Chapters (Native-Specific) ✓

All four new chapters exist and align well with native-type-universe.md (now at Clef/docs/fidelity/):

| Chapter | File | Alignment Notes |
|---------|------|-----------------|
| Native Type Mappings | `native-type-mappings.md` | ✓ Aligns with Parts 2-5 |
| Memory Regions | `memory-regions.md` | ✓ Aligns with Part 8 |
| Access Kinds | `access-kinds.md` | ✓ Aligns with Part 8.2 |
| Platform Bindings | `platform-bindings.md` | ✓ Aligns with CCS spec |

### "Review" Status Chapters - SIGNIFICANT BCL/CLI ISSUES

These chapters have extensive BCL/CLI dependencies that conflict with native type universe principles:

#### expressions.md - CRITICAL ISSUES

1. **Null Expression (line 32)**: Grammar includes `null` expression form
   - Native Type Universe: "No null anywhere in the type system"
   - CONFLICT: Section on null expressions (lines 1533-1535) directly contradicts null-freedom

2. **System.* Type Comments (lines 287-297)**:
   ```
   99999999n       // nativeint    (System.IntPtr)
   'a'             // char         (System.Char)
   "c:\\home"      // string       (System.String)
   ```
   - Should reference native representations, not BCL types

3. **System.Tuple/ValueTuple (lines 348-387)**: 
   - Extensive discussion of runtime type representation
   - Native Type Universe: Tuples are unboxed, no System.Tuple

4. **Runtime Semantics (lines 2999-3418)**:
   - Many references to "runtime type", null checks, System.Object
   - Native Type Universe: Compile-time types only

5. **Garbage Collection (lines 2682-2721)**:
   - References to GC, GCHandle, pinned pointers
   - Native Type Universe: No GC

6. **Exception Handling (lines 2400-2458)**:
   - System.Exception, System.Diagnostics.Debug.Assert
   - Should use Result-based error handling

#### type-definitions.md - CRITICAL ISSUES

1. **CLI Delegate Definition (line 95)**: Grammar includes CLI delegate
2. **System.Int32 (line 177)**: `type int = System.Int32`
3. **System.Collections Interfaces (lines 522-524, 636-638)**:
   - IStructuralEquatable, IStructuralComparable, IComparable
4. **CLIMutable (lines 548-577)**: Entire section about CLI serialization
5. **Compiled Form for CLI Languages (lines 680-704)**: Non-applicable section
6. **System.Object Inheritance (line 829)**: Default base class
7. **CLI Struct Rules (lines 1254-1310)**: CLI-specific struct constraints
8. **CLI Enumeration Type (lines 1314-1363)**: CLI enum compilation
9. **CLI Delegate Type (lines 1377-1398)**: P/Invoke delegates
10. **CLI C# Extension Members (lines 1537-1567)**: C# interop
11. **CLI-Compatible Optional Arguments (lines 2035-2092)**: C#/CLI interop
12. **CLI Events (lines 2333-2401)**: CLIEvent attributes
13. **Generated Comparison (lines 2920-3033)**: Many null checks

#### namespaces-and-modules.md - MODERATE ISSUES

1. **CLI Representations (lines 300-301)**: "compiled CLI representations"
2. **System.Type (line 368)**: `val typeof<'T> : System.Type`
3. **System.* References (lines 447-479)**: Windows.Forms examples
4. **Assembly Access (line 521)**: InternalsVisibleTo
5. **CLI Compiled Form (line 545)**: CLI access modifiers

### Alignment with Native Type Universe Parts

| Part | Topic | Covered In Spec | Status |
|------|-------|-----------------|--------|
| Part 1 | Foundational Principles | types-and-type-constraints.md | ✓ Revised |
| Part 2 | Primitive Types | native-type-mappings.md | ✓ Exists |
| Part 3 | Structural Types | types-and-type-constraints.md, native-type-mappings.md | ✓ Revised |
| Part 4 | Reference Types | types-and-type-constraints.md, native-type-mappings.md | ✓ Revised |
| Part 5 | Parameterized Types | types-and-type-constraints.md, native-type-mappings.md | ✓ Revised |
| Part 6 | Function Types | native-type-mappings.md | ✓ Exists |
| Part 7 | Mutable State | Need review in expressions.md | ⚠️ Review needed |
| Part 8 | Memory Region Types | memory-regions.md, access-kinds.md | ✓ Exists |
| Part 9 | Coeffects | Deferred (Part 9 is "TBD" in native-type-universe.md) | N/A |

## Clean Break Principle

> **Clef is multi-platform and multi-hardware by default.**

The spec should NOT illustrate from:
- Desktop-only perspectives
- Windows-only perspectives (Windows.Forms, WinUI)
- Any single-platform perspective

Examples should be:
- Platform-agnostic (console I/O, pure computation)
- Or explicitly using Platform.Bindings pattern (which is platform-neutral in syntax)

### What Doesn't Belong in a Language Spec

| Category | Example | Action |
|----------|---------|--------|
| Platform-specific UI | `System.Windows.Forms.*` | **REMOVE entirely** |
| Desktop-only patterns | GUI event handling | **REMOVE or abstract** |
| Windows-specific APIs | Registry, COM, WMI | **REMOVE entirely** |
| Runtime-dependent patterns | Reflection, dynamic loading | **REMOVE entirely** |
| BCL-specific types | `System.Type`, `System.Delegate` | **REMOVE or replace** |

### What Belongs (Platform-Neutral)

| Category | Example | Notes |
|----------|---------|-------|
| Console I/O | `Console.WriteLine`, `Console.ReadLine` | Via Platform.Bindings |
| Pure computation | Arithmetic, collections, pattern matching | Core language |
| Type definitions | Records, DUs, classes, interfaces | Core language |
| Memory regions | Stack, Arena, Peripheral | clef-lang-specific |
| Access kinds | ReadOnly, WriteOnly, ReadWrite | clef-lang-specific |

## Priority Remediation