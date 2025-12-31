# Alloy BCL-Equivalent Library Gaps

## Purpose
This memory tracks F# language constructs and common BCL patterns that require Alloy implementations but don't yet exist.

## Known Gaps

### Language Construct Support

| F# Construct | Required Alloy Type | Status | Notes |
|--------------|---------------------|--------|-------|
| `lazy expr` | `Lazy<'T>` | **MISSING** | Language keyword requires backing type. Should be struct-based, single-threaded default. |
| `lock expr` | Synchronization primitives | **MISSING** | May need platform-specific implementation via Platform.Bindings |
| `async { }` | `Async<'T>` builder | **PARTIAL** | Need to verify completeness |
| `task { }` | `Task<'T>` equivalent | **MISSING** | Lower priority for embedded targets |

### Common BCL Patterns Needing Native Equivalents

| BCL Type/Pattern | Alloy Equivalent | Status | Notes |
|------------------|------------------|--------|-------|
| `System.Lazy<'T>` | `Alloy.Lazy<'T>` | **MISSING** | See above |
| `System.IDisposable` | `IDisposable` | Exists? | Verify in Alloy |
| `System.IComparable<'T>` | `IComparable<'T>` | Exists? | Verify in Alloy |
| `System.Collections.Generic.*` | `Alloy.Collections.*` | **PARTIAL** | Map, Set exist; verify List, Queue, Stack |
| `System.Text.StringBuilder` | `Alloy.Text.StringBuilder` | **UNKNOWN** | Check if needed |
| `System.Random` | `Alloy.Random` | **UNKNOWN** | Platform PRNG via bindings |
| `System.Guid` | `Alloy.Guid` or UUID | **UNKNOWN** | May need for distributed systems |
| `System.DateTime` | `Alloy.DateTime` | **UNKNOWN** | Time representation for native |
| `System.TimeSpan` | `Alloy.TimeSpan` | **UNKNOWN** | Duration type |
| `System.Numerics.BigInteger` | `Alloy.Numerics.BigInt` | **UNKNOWN** | For `I` suffix literals |

### Lazy<'T> Design Notes

When implementing `Alloy.Lazy<'T>`:

1. **Struct-based**: Avoid allocation for the wrapper itself
2. **Single-threaded default**: Predictable for embedded/real-time
3. **Optional thread-safe variant**: `LazyThreadSafe<'T>` if needed
4. **Closure allocation**: Thunk needs arena or explicit lifetime

```fsharp
// Proposed API
module Alloy.Lazy

[<Struct>]
type Lazy<'T> = ...

module Lazy =
    val create : (unit -> 'T) -> Lazy<'T>
    val force : Lazy<'T> -> 'T
    val isValueCreated : Lazy<'T> -> bool
```

## Action Items

1. **Audit Alloy** for existing implementations of these types
2. **Prioritize** based on language construct requirements (Lazy is high priority)
3. **Add to Alloy project Serena memory** when activating that project
4. **Update fsnative-spec** to document what Alloy MUST provide

## Cross-References

- fsnative-spec: `spec/the-native-library-alloy.md` should enumerate required types
- Firefly: Compiler expects certain types to exist for language constructs
