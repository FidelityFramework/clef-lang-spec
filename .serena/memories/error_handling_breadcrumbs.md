# Error Handling and Null Alternatives - Spec Breadcrumbs

> **Note**: A full draft specification has been created at `spec/error-handling.md`. This memory documents the reasoning and breadcrumbs that led to that specification.

## Context
During spec revision, we're removing CLI/BCL references. However, some concepts need *replacement with native equivalents*, not just deletion.

## Key Concepts Needing Native Specification

### 1. Null → voption
- CLI F# uses `null` for absent values in some contexts
- Clef uses `voption<'T>` (value option) with `ValueSome`/`ValueNone`
- **Spec needs**: How voption is the idiomatic way to express "may not have a value"
- **Guard rail**: Compiler should guide users toward voption, not null

### 2. Exceptions → Result<'T, 'E>
- CLI F# raises exceptions: `NullReferenceException`, `InvalidCastException`, `IndexOutOfBoundsException`
- Clef uses `Result<'T, 'E>` for explicit error handling
- **Spec needs**: Define standard error types, how Result flows through code
- **Guard rail**: Operations that can fail should return Result, not throw

### 3. Areas in expressions.md that need native equivalents (not just removal)

| Removed Concept | Native Equivalent | Spec Section Needed |
|-----------------|-------------------|---------------------|
| `null` expression | No equivalent - null-free by design | Explain null-freedom |
| NullReferenceException | Cannot occur - type system prevents | Explain why not needed |
| InvalidCastException | Cannot occur - static typing | Explain why not needed |
| IndexOutOfBoundsException | `Result<'T, IndexError>` or bounds-checked access | Define array/collection access semantics |
| Division by zero | `Result<'T, DivisionByZero>` | Define arithmetic error handling |
| Option using null internally | voption has no null representation | Define voption representation |

### 4. Evaluation semantics that need rewriting (not deletion)

- **Method application**: Instead of "if null, raise exception" → receiver is always valid
- **Field lookup**: Instead of "if null, raise exception" → instance is always valid  
- **Type tests/casts**: Static type system prevents invalid casts
- **While loop result**: Instead of "null (unit representation)" → unit is zero-sized, not null

### 5. Future spec sections to consider

- `error-handling.md` - Dedicated chapter on Result<'T,'E> patterns
- `special-attributes-and-types.md` already mentions Result - may need expansion
- How try/with/finally work with Result (or do they? maybe they're for effects only)

## Tooling Ecosystem Considerations

**Strategic Goal**: Keep the door open for re-integration with Ionide and F# ecosystem tooling.

### What This Means for Error Handling Design

1. **Syntax Compatibility**: Exception/error syntax should remain parseable by standard F# tooling
   - `try`/`with`/`finally` syntax is F# standard - keep it
   - The *semantics* may differ (effects-based vs exception-based) but syntax stays compatible

2. **Type Signatures**: Error-returning types should be expressible in standard F# type notation
   - `Result<'T, 'E>` is already standard F#
   - `voption<'T>` is standard F# (ValueOption)
   - Ionide can understand and display these types

3. **Diagnostics Format**: Compiler errors/warnings should follow F# conventions
   - Same error message format
   - Same source location reporting
   - Ionide integration depends on this

4. **What May Require Forks**:
   - Language server features that assume BCL types exist
   - Autocomplete for BCL namespaces
   - Go-to-definition for BCL symbols
   - These are *tooling* issues, not *language* issues

5. **Design Principle**: Semantic differences are acceptable; syntactic incompatibility is not
   - Clef code should parse as valid F#
   - Clef code should type-check with Clef type rules
   - The compiled behavior differs, but the source is "F# shaped"

### Implication for Spec

When specifying error handling semantics:
- Keep standard F# syntax (`try`/`with`, `raise`, etc.)
- Specify different runtime behavior where needed
- Use standard types (`Result`, `ValueOption`) where possible
- Don't invent new syntax that Ionide couldn't parse

## Revision Approach

When encountering null/exception patterns:
1. **Ask**: Does this concept have a native equivalent?
2. **If yes**: Mark for rewrite, don't just delete
3. **If no** (truly doesn't apply): Remove cleanly
4. **Leave TODO comments** in spec where rewrite is needed

## Example: Field Lookup

**Before (CLI):**
```
- If `expr` evaluates to `null`, a `NullReferenceException` is raised.
```

**After (native) - NOT just deletion:**
```
- The instance `expr` is evaluated. Since Clef is null-free, the instance is always valid.
```

This acknowledges the concept while explaining why the error case doesn't apply.
