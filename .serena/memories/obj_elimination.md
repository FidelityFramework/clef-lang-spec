# Elimination of `obj` from F# Native

## Core Decision

**`obj` (System.Object) is NOT available in F# Native.** The compiler SHALL reject any code that references `obj`.

This is as fundamental to F# Native as `voption` for optional values.

## Rationale

In managed F#, all types inherit from `System.Object` (aliased as `obj`). This enables:
- Boxing value types to heap-allocated objects
- Runtime type information and reflection
- Heterogeneous collections (`obj list`)
- Generic `%A`/`%O` formatting via runtime inspection

**None of these capabilities exist in native compilation:**

1. **No RTTI**: Native binaries don't carry type metadata
2. **No GC**: `obj` implies heap allocation with GC lifetime
3. **Full monomorphization**: Generics are specialized at each call site
4. **SRTP replaces runtime dispatch**: Compile-time method resolution

## Impact on F# Constructs

| Construct | Managed F# | F# Native |
|-----------|------------|-----------|
| `box x` | Wraps value in heap object | NOT AVAILABLE |
| `unbox x` | Extracts value from object | NOT AVAILABLE |
| `x :> obj` | Upcast to base type | NOT AVAILABLE |
| `x :?> T` | Downcast with runtime check | Use DU pattern matching |
| `%A` formatting | Runtime type inspection | SRTP-based `Showable $` pattern |
| `obj list` | Heterogeneous collection | Use discriminated union |
| `{ new obj() with ... }` | Anonymous object | Must implement interface |
| `obj.GetType()` | Runtime type token | NOT AVAILABLE |
| `obj.ReferenceEquals` | Identity comparison | Type-specific comparison |

## Object Expressions

Object expressions MUST implement at least one interface type:

```fsharp
// NOT AVAILABLE in F# Native
{ new obj() with member x.ToString() = "Hello" }

// VALID - implements interface
{ new IDisposable with member x.Dispose() = cleanup() }
```

## Polymorphic Operations

Use SRTP instead of `obj`-based dispatch:

```fsharp
// NOT AVAILABLE
let show (x: obj) = sprintf "%A" x

// VALID - SRTP pattern from Alloy
type Showable = Showable
    with static member inline ($) (Showable, x: int) = intToString x
         static member inline ($) (Showable, x: string) = x

let inline show x = Showable $ x
```

## Spec Location

Documented in `native-type-mappings.md` under "The Universal Base Type `obj` Is Not Available".

## Remediation Completed

The following spec files have been remediated to remove inappropriate `obj` references:

### expressions.md
- Object expression examples rewritten to use interface implementations only
- `%O` and `%A` format specifiers documented as SRTP-based
- Static coercion examples use interface upcasting instead of `obj`
- Dynamic type-test (`:?`) and coercion (`:?>`) sections note unavailability
- "Values with Underspecified Object Identity" section rewritten

### type-definitions.md
- Implicit `Equals` signatures changed from `obj -> bool` to `'T -> bool`
- Class inheritance section notes no default `obj` base
- Delegate examples use typed sender parameters
- Override method examples changed from `override obj.Method` to `override this.Method`
- Equality/comparison attribute table updated
- Generated `Equals` and `CompareTo` implementations use typed parameters

## Related Decisions

- `voption` instead of `option` (value semantics, no null)
- No reflection (`typeof<'T>`, `typedefof<'T>` not available)
- No dynamic dispatch (`:?>` not available for arbitrary types)
- All generics monomorphized (no type erasure)
