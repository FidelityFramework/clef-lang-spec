# Nested Function Capture Convention

## Date: January 2026 (PRD-13 Recursion)

## Key Distinction

F# Native distinguishes two categories of functions with captures:

### 1. Escaping Closures (Closure Struct Model)
- Anonymous lambdas: `fun x -> x + captured`
- PSG parent is Application, Sequential, etc.
- Can be returned, stored, passed to HOFs
- Representation: `{code_ptr, env}` struct with flat captures
- Calling convention: extract `env_ptr`, pass as first argument

### 2. Nested Named Functions (Parameter-Passing Model)
- Named definitions: `let rec loop acc i = ...`
- PSG parent is a Binding node
- Called only from defining scope, never escapes
- Representation: direct function with extended signature
- Calling convention: captures prepended as explicit parameters

## Classification Criteria

A Lambda is a nested named function iff:
1. `enclosingFunction = Some _` (nested inside another function)
2. Parent PSG node is a `Binding` (named function definition)

## Why Parameter-Passing for Nested Functions?

1. **Zero allocation**: No closure struct construction
2. **Better inlining**: Direct calls are simpler
3. **No indirection**: No extractvalue/getelementptr for env access
4. **Register-friendly**: All values can stay in registers

## Implementation (Firefly)

- **SSAAssignment.fs**: Checks `isNestedNamedFunction` to skip ClosureLayout
- **Witness.fs**: `isClosureCall` returns `(false, Some captures)` for nested functions
- **LambdaWitness.fs**: `preBindParams` adds captures as explicit parameters

## Spec References

- `closure-representation.md` §8: Full specification of the distinction
- `expressions.md` §Nested Recursive Functions: Calling convention details
