# Types and Type Constraints Revisions for clef-lang-spec

## Overview
The `spec/types-and-type-constraints.md` file was extensively revised to remove BCL/.NET assumptions and align with Clef semantics.

## Key Revisions Made

### 1. Opening Section
- Removed "Runtime types" as a fourth meaning of type (no System.Type, no GetType())
- Added Clef Note explaining compile-time-only type system

### 2. Constraint Grammar
- Removed CLI comments (CLI default constructor, CLI non-Nullable struct, CLI reference type)
- Removed delegate constraint from grammar (not supported)

### 3. Tuple Types
- Replaced System.Tuple references with native unboxed representation
- Added memory layout diagram showing fat pointer structure
- Explained stack/arena allocation

### 4. Struct Tuple Types
- Noted that struct vs reference tuple distinction doesn't apply in Clef
- Both compile to same unboxed representation

### 5. Array Types
- Replaced System.Array with fat pointer representation
- Added memory layout showing {ptr, len} structure
- Added bounds checking note

### 6. Nullness Constraints (CRITICAL)
- Complete rewrite for null-free semantics
- typar : null constraint NOT SUPPORTED
- Error FS8010 for null usage
- Explained option<'T> as the replacement pattern

### 7. Value Type Constraints
- Removed System.Nullable references
- Listed which types satisfy struct constraint
- Noted option<'T> uses voption semantics

### 8. Reference Type Constraints
- Explained pointer vs direct representation distinction
- Noted allocation strategy (stack/arena) is separate from type category

### 9. Delegate Constraints
- Marked as NOT SUPPORTED
- Explained function types as replacement

### 10. Equality/Comparison Constraints
- Removed System.IComparable interface references
- Explained SRTP resolution against Alloy witness hierarchy

### 11. Base Type of a Type
- Replaced System.Object/System.Array/etc. base types
- Created "Structural Category" table instead
- Noted no universal base type exists

### 12. Interface Types
- Removed System.Collections.Generic.IEnumerable references
- Explained compile-time SRTP resolution for interfaces

### 13. Nullness Section (CRITICAL)
- Complete rewrite for absolute null-freedom
- All types non-nullable by construction
- Explained option<'T> → voption<'T> semantics
- API changes table (sentinel values → voption returns)

### 14. Default Initialization
- Removed nullness constraint category
- Listed types that permit/don't permit default initialization

### 15. Dynamic Conversion (renamed to Type Conversions)
- Removed runtime type concepts (boxing, System.Type)
- All conversions verified at compile time
- Explained static type coercion

### 16. Type Definitions Characteristics
- Removed Delegate from type kinds
- Replaced System.Collections.Generic examples with native types

### 17. Expanding Abbreviations
- Removed System.Int32 examples
- Explained int = platform word (not 32-bit)
- Noted int ≠ int32 distinction

## Style Conventions Established
- Use `> **Clef Note**:` blocks for native-specific explanations
- Memory layout diagrams use ASCII box drawing
- Reference Clef/docs/fidelity/native-type-universe.md for detailed type specs
- Keep managed F# references only in explanatory "what Clef does NOT have" context
