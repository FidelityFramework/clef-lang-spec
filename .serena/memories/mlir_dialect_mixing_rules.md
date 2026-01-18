# MLIR Dialect Mixing Rules (January 2026)

## Key Findings from Regression Test Fixes

### 1. `llvm.mlir.addressof` Requires `llvm.func`

The `llvm.mlir.addressof` operation can ONLY reference:
- `llvm.func`
- `llvm.mlir.global`
- `llvm.mlir.alias`

It CANNOT reference `func.func`. This has implications for:

- **Closures with captures**: Already handled - use `llvm.func` via ClosureLayout
- **Lazy thunks WITHOUT captures**: Must ALSO use `llvm.func` because their address is always taken
- **Seq generators**: Already handled correctly - SeqWitness emits `llvm.func` directly

### 2. `func.call` Inside `llvm.func` Bodies

MLIR allows calling `func.func` functions from within `llvm.func` bodies using `func.call`. This is important for:

- **`_start` wrapper**: Defined as `llvm.func` (linker may reference it), but calls `main` which is `func.func`
- You CANNOT use `llvm.call` to call a `func.func` - it will fail with "does not reference a valid LLVM function"

### 3. LambdaContext Determines Function Dialect

The `LambdaContext` enum indicates whether a lambda's address will be taken:

| Context | Address Taken? | Function Dialect |
|---------|---------------|------------------|
| `RegularClosure` (no captures) | No | `func.func` |
| `RegularClosure` (with captures) | Yes | `llvm.func` |
| `LazyThunk` (any) | **Always Yes** | `llvm.func` |
| `SeqGenerator` | **Always Yes** | `llvm.func` |

### Fix Applied (LambdaWitness.fs)

```fsharp
let isLazyThunk =
    match node.Kind with
    | SemanticKind.Lambda (_, _, _, _, LambdaContext.LazyThunk) -> true
    | _ -> false
let isClosure = closureLayoutOpt.IsSome || isLazyThunk

// Later: use llvm.func when isClosure is true
```

### Spec Updates Made

1. **`lazy-representation.md`**: Added normative note after thunk example clarifying that lazy thunks ALWAYS use `llvm.func` regardless of captures.

2. **`backend-lowering-architecture.md`**: Added section 3.3 "Dialect Mixing Within Function Bodies" explaining `func.call` inside `llvm.func` and `_start` calling `main` pattern.
