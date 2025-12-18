# F# Native Language Specification

**The normative specification for native F# type semantics and memory management.**

---

## What is fsnative-spec?

fsnative-spec defines the complete language semantics for [fsnative](https://github.com/speakeztech/fsnative) (F# Native Compiler Services). Where the [standard F# specification](https://fsharp.org/specs/language-spec/) describes behavior in terms of the .NET runtime and BCL types, fsnative-spec provides explicit definitions for everything the CLR normally handles implicitly: type layouts, memory ownership, lifetime verification, and deterministic resource management.

This specification is necessarily **more comprehensive** than the standard F# specification. The F# spec can say "a string is `System.String`" and defer all questions of encoding, allocation, and cleanup to the CLR documentation. fsnative-spec cannot defer. Every aspect of type representation and memory behavior must be explicitly defined.

When fsnative's behavior is in question, this document is the authority.

## Why fsnative-spec Contains More Than the F# Specification

The standard F# specification makes extensive use of the .NET runtime as an implicit substrate. Consider what the F# spec does *not* need to define:

**Memory Allocation**: The F# spec never explains where objects live in memory, how allocation works, or when memory is reclaimed. It simply notes that values are created and trusts the CLR's garbage collector.

**Object Layout**: The F# spec doesn't define how a record's fields are arranged in memory, what padding exists between fields, or how discriminated union tags are represented. These are "implementation details" handled by the runtime.

**Reference Semantics**: The F# spec doesn't distinguish between a value that owns its memory and a value that borrows someone else's memory. In managed F#, all references are equivalent because the GC ensures memory remains valid.

**Resource Cleanup**: The F# spec has no drop semantics. Objects are allocated, used, and eventually collected. The programmer need not think about when cleanup occurs.

**Type Identity**: In managed F#, types are identified by their assembly metadata. `System.String` from one assembly is the same type as `System.String` from another because the runtime resolves these identities.

fsnative-spec must explicitly define all of these. Native compilation has no runtime to defer to. The specification must provide complete, unambiguous definitions for:

- Exact memory layout of every type
- Ownership semantics for every value
- Lifetime constraints for every reference
- Deterministic cleanup timing
- Type identity independent of assembly metadata

## Type System Departures

fsnative-spec redefines the primitive and built-in types of F# to have native semantics. These are not "mappings" or "translations" - they are the intrinsic types of the language when targeting native compilation.

### Primitive Types

| F# Syntax | Standard F# Type | fsnative Type | Notes |
|-----------|-----------------|---------------|-------|
| `int` | `System.Int32` | `i32` | 32-bit signed, two's complement |
| `int64` | `System.Int64` | `i64` | 64-bit signed, two's complement |
| `float` | `System.Double` | `f64` | IEEE 754 binary64 |
| `float32` | `System.Single` | `f32` | IEEE 754 binary32 |
| `byte` | `System.Byte` | `u8` | 8-bit unsigned |
| `bool` | `System.Boolean` | `i1` | Single bit, 0 = false, 1 = true |
| `unit` | `Microsoft.FSharp.Core.Unit` | Zero-sized | No runtime representation |
| `char` | `System.Char` | `u32` | Unicode scalar value (not UTF-16 code unit) |

**Key Departure: `char`**

In standard F#, `char` is a UTF-16 code unit (`System.Char`). This means a single `char` cannot represent all Unicode characters - surrogate pairs require two `char` values.

In fsnative, `char` is a Unicode scalar value (Unicode code point that is not a surrogate). This is a 21-bit value stored in 32 bits. Every `char` represents exactly one character. This aligns with Rust's `char` and modern Unicode handling.

### String Types

| F# Syntax | Standard F# Type | fsnative Type |
|-----------|-----------------|---------------|
| `"literal"` | `System.String` | `NativeStr` |
| `string` type | `System.String` | `NativeStr` |

**NativeStr Semantics**

`NativeStr` is UTF-8 encoded, null-terminated, with a known length. The specification defines:

```
NativeStr := {
    data: ptr<u8, region, read_only>  // Pointer to UTF-8 bytes
    length: usize                      // Byte count (not code point count)
}
```

**Encoding**: UTF-8. All string operations understand multi-byte sequences. Indexing by byte offset is O(1); indexing by character requires traversal.

**Null Termination**: The byte sequence is null-terminated for C interoperability. The length field stores the byte count *excluding* the null terminator.

**Immutability**: String literals are immutable. The `data` pointer has `read_only` access kind.

**Lifetime**: String literals have `'static` lifetime - they exist for the program's entire execution, typically in read-only memory (`.rodata` section).

**No Garbage Collection**: Dynamically created strings have explicit ownership. When the owning binding goes out of scope, the string's memory is freed.

### Option Types

| F# Syntax | Standard F# Type | fsnative Type |
|-----------|-----------------|---------------|
| `Some x` | `'a option` | `'a voption` |
| `None` | `'a option` | `'a voption` |

**Value Option Semantics**

In standard F#, `option` is a reference type. `Some x` allocates a heap object containing `x`. `None` is typically a null reference.

In fsnative, `voption` is a value type:

```
voption<'a> :=
    | None    // tag = 0, no payload
    | Some 'a // tag = 1, payload = 'a
```

**Layout**: The tag and payload are stored inline. For `Some 42`, the memory contains the tag byte followed by the integer value.

**Zero-Cost None**: `None` has no runtime cost beyond its tag. No allocation occurs.

**Nesting Optimization**: `voption<voption<'a>>` can distinguish `None`, `Some None`, and `Some (Some x)` without additional allocations.

**Size**: `sizeof(voption<'a>) = sizeof(tag) + sizeof('a) + padding` where tag is minimally sized based on variant count.

### Array Types

| F# Syntax | Standard F# Type | fsnative Type |
|-----------|-----------------|---------------|
| `[| 1; 2; 3 |]` | `'a[]` (System.Array) | `NativeArray<'a, N>` or `FatPtr<'a>` |

**NativeArray (Compile-Time Size)**

When the array size is known at compile time:

```
NativeArray<'a, N> := {
    elements: 'a[N]  // Inline storage for N elements
}
```

**Layout**: Elements are stored contiguously with no header. `sizeof(NativeArray<i32, 4>) = 16` (four 32-bit integers).

**Bounds Checking**: The size `N` is part of the type. Out-of-bounds access is a compile-time error when the index is constant, a runtime check when dynamic.

**Stack Allocation**: `NativeArray` lives wherever its binding lives - typically the stack for local variables.

**FatPtr (Runtime Size)**

When the size is determined at runtime:

```
FatPtr<'a, region, access> := {
    data: ptr<'a, region, access>  // Pointer to first element
    length: usize                   // Element count
}
```

**Layout**: A pointer-length pair. The actual elements are allocated in the specified region.

**Region Tracking**: The `region` parameter tracks where the data lives (stack, heap, arena, peripheral).

**Access Kind**: The `access` parameter constrains operations (read_only, write_only, read_write).

### Record Types

**Standard F#**: Records are reference types (unless marked `[<Struct>]`). Field access involves pointer indirection. The GC manages the record's lifetime.

**fsnative**: Records are value types by default.

```fsharp
type Point = { X: int; Y: int }
```

Becomes:

```
Point := {
    X: i32  // offset 0
    Y: i32  // offset 4
}
// sizeof(Point) = 8, align = 4
```

**Layout**: Fields are stored in declaration order with platform-appropriate padding for alignment.

**Ownership**: A `Point` value owns its data. When the binding goes out of scope, no cleanup is needed (all fields are scalars).

**Borrowing**: Functions can borrow records:

```fsharp
let distance (p: inref<Point>) = ...  // Immutable borrow
let move (p: byref<Point>) dx dy = ... // Mutable borrow
```

### Discriminated Unions

**Standard F#**: DUs are reference types. Each case may involve allocation. The runtime handles tag checking.

**fsnative**: DUs are tagged unions with explicit layout.

```fsharp
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Point
```

Becomes:

```
Shape := {
    tag: u8  // 0 = Circle, 1 = Rectangle, 2 = Point
    payload: union {
        Circle: { radius: f64 }
        Rectangle: { width: f64; height: f64 }
        Point: { } // zero-sized
    }
}
// sizeof(Shape) = 1 + 7 (padding) + 16 = 24
// align = 8 (due to f64)
```

**Tag Representation**: The tag is minimally sized. 2-4 cases use `u8`, 5-256 use `u8`, 257+ use `u16`, etc.

**Payload Union**: All cases share the same memory. The size is the maximum payload size plus padding for alignment.

**Exhaustiveness**: Pattern matching is verified exhaustive at compile time. No "incomplete match" exceptions at runtime.

### Function Types

**Standard F#**: Functions are represented as closure objects on the heap. Partial application creates new closure objects. The GC manages closure lifetime.

**fsnative**: Functions have multiple representations depending on usage.

**Non-Capturing Functions**

```fsharp
let add x y = x + y
```

Compiles to a plain function pointer with no closure. `add` has type `fn(i32, i32) -> i32`.

**Capturing Functions (Closures)**

```fsharp
let multiplier n =
    let factor = n * 2
    fun x -> x * factor
```

The inner function captures `factor`. The closure is:

```
Closure := {
    fn_ptr: fn(ptr<Env>, i32) -> i32  // Function taking environment + args
    env: Env                           // Captured variables
}

Env := {
    factor: i32
}
```

**Closure Lifetime**: The closure's environment is allocated where the closure is used. If the closure doesn't escape, the environment lives on the stack.

**Escape Analysis**: The compiler tracks whether closures escape their defining scope. Non-escaping closures avoid heap allocation entirely.

## Memory Management Model

This is the largest departure from standard F#. The F# specification has no memory model - it defers entirely to the CLR. fsnative-spec defines a complete ownership and borrowing system.

### Ownership

Every value in fsnative has exactly one owner. When the owner goes out of scope, the value is dropped (its resources are cleaned up).

```fsharp
let process () =
    let data = allocate 1024  // data owns the allocation
    use data                  // ... operations ...
    // data goes out of scope here; memory is freed
```

**Ownership Transfer (Move Semantics)**

Ownership can be transferred:

```fsharp
let a = createBuffer ()
let b = a  // Ownership moves from a to b
// a is no longer valid
```

After the move, `a` cannot be used. This is a compile-time error, not a runtime check.

**Copy Types**

Some types implement `Copy` - they are duplicated rather than moved:

```fsharp
let x = 42
let y = x  // x is copied to y
// Both x and y are valid
```

Primitive types (`int`, `float`, `bool`) and small value types are implicitly `Copy`.

### Borrowing

References can borrow values without taking ownership:

**Immutable Borrow (`inref<'a>`)**

```fsharp
let length (s: inref<NativeStr>) = s.length
```

Multiple immutable borrows can coexist. The borrowed value cannot be mutated through any borrow.

**Mutable Borrow (`byref<'a>`)**

```fsharp
let increment (x: byref<int>) = x <- x + 1
```

Only one mutable borrow can exist at a time. No other borrows (mutable or immutable) can coexist with a mutable borrow.

**Borrow Rules**

The compiler enforces at compile time:

1. A value can have either:
   - Any number of immutable borrows, OR
   - Exactly one mutable borrow
2. Borrows cannot outlive the borrowed value
3. A value cannot be moved while borrowed

### Lifetimes

Lifetimes are compile-time annotations that track how long references are valid.

**Implicit Lifetimes**

Most lifetimes are inferred:

```fsharp
let first (arr: inref<NativeArray<int, _>>) = arr.[0]
```

The compiler infers that the return value (an `int`, copied) has no lifetime constraint, while `arr` must be valid for the function call.

**Explicit Lifetimes**

When references are returned, lifetimes constrain the result:

```fsharp
let firstRef<'a> (arr: inref<NativeArray<'a, _>>) : inref<'a> = &arr.[0]
```

The returned reference has the same lifetime as `arr`. It cannot outlive the array.

**Lifetime Errors**

```fsharp
let dangling () =
    let local = [| 1; 2; 3 |]
    &local.[0]  // ERROR: returns reference to local that will be dropped
```

This is a compile-time error. No dangling references can exist.

### Memory Regions

Values exist in specific memory regions. The region affects allocation strategy, access patterns, and lifetime.

**Stack Region**

```fsharp
let local = { X = 1; Y = 2 }  // Allocated on stack
```

- Automatic allocation/deallocation (scope-based)
- Limited size (typically 1-8 MB)
- Fastest access
- Cannot outlive the creating function

**Heap Region**

```fsharp
let boxed = heap { X = 1; Y = 2 }  // Allocated on heap
```

- Manual or RAII-based lifetime
- Effectively unlimited size
- Slightly slower access
- Can outlive the creating function (with ownership transfer)

**Arena Region**

```fsharp
arena myArena {
    let a = myArena.alloc { X = 1; Y = 2 }
    let b = myArena.alloc { X = 3; Y = 4 }
    // ... many allocations ...
} // All arena allocations freed at once
```

- Bulk allocation with single deallocation
- No individual frees
- Excellent for temporary work
- Cannot escape the arena scope

**Peripheral Region**

```fsharp
let uart: ptr<UartRegisters, peripheral, read_write> = 0x4000_0000
```

- Memory-mapped I/O
- Special access semantics (volatile reads/writes)
- Never freed (hardware exists forever)
- Aligned access requirements

**Flash Region**

```fsharp
let lookup: ptr<LookupTable, flash, read_only> = flashAddress
```

- Read-only program memory
- May require aligned access
- Special read operations on some platforms
- Static lifetime

### Access Kinds

Pointers carry access kind information:

| Access Kind | Read | Write | Use Case |
|-------------|------|-------|----------|
| `read_only` | Yes | No | String literals, flash memory, shared data |
| `write_only` | No | Yes | DMA buffers, write-only registers |
| `read_write` | Yes | Yes | General mutable data |

**Access Kind Enforcement**

```fsharp
let s: ptr<u8, flash, read_only> = stringLiteral
s.[0] <- 65uy  // COMPILE ERROR: cannot write to read_only
```

### Deterministic Cleanup

When a value goes out of scope, its `drop` is called:

```fsharp
let process () =
    let file = openFile "data.txt"  // file owns the handle
    // ... use file ...
// file.drop() called here - closes the handle
```

**Drop Order**

Values are dropped in reverse declaration order:

```fsharp
let f () =
    let a = create ()
    let b = create ()
    let c = create ()
// Drop order: c, b, a
```

**Drop and Panics**

If a drop function panics, the program aborts. Drops must not fail.

## SRTP and Witness Resolution

Statically Resolved Type Parameters (SRTP) in standard F# resolve against .NET types and their members. In fsnative, SRTP resolves against the Alloy witness hierarchy.

**Standard F# SRTP**

```fsharp
let inline show x = (^a : (member ToString : unit -> string) x)
```

Resolves against whatever `ToString` method exists on the type at each call site.

**fsnative SRTP**

```fsharp
let inline add a b = a + b
```

Resolves against the Alloy `Add` witness:

```
trait Add<'a> {
    static member (+) : 'a -> 'a -> 'a
}

witness Add<i32> { (+) = prim_add_i32 }
witness Add<f64> { (+) = prim_add_f64 }
```

The compiler knows the complete witness hierarchy at compile time. SRTP resolution is always successful or fails with a clear error - there is no runtime dispatch.

## Effect Tracking (Coeffects)

fsnative tracks computational effects in the type system.

**Pure Functions**

```fsharp
let pure add x y = x + y  // No effects
```

Pure functions can be memoized, reordered, or eliminated.

**IO Effect**

```fsharp
let impure readLine () : io<string> = ...
```

Functions with IO effects cannot be reordered past other IO operations.

**Effect Polymorphism**

```fsharp
let inline map f xs = xs |> Array.map f
```

The effect of `map` is determined by the effect of `f`.

## Relationship to Standard F# Specification

fsnative-spec is a companion document that:

1. **Inherits** syntax, name resolution, type inference algorithms, and expression evaluation order from the F# spec
2. **Replaces** type definitions with native equivalents
3. **Extends** the type system with ownership, lifetimes, and regions
4. **Adds** memory management semantics not present in F#
5. **Defines** what the F# spec leaves implicit

Where fsnative-spec is silent, the standard F# specification applies. Where they conflict, fsnative-spec takes precedence for native compilation.

## The Fidelity Ecosystem

fsnative-spec is the normative foundation for the Fidelity native compilation framework:

```
fsnative-spec (this repo)     <- Defines the rules
        |
fsnative (FNCS)               <- Implements type resolution per spec
        |
Firefly                       <- Generates code per spec semantics
        |
Native Binary                 <- Runs with spec-defined behavior
```

**fsnative-spec** defines what the language means.

**[fsnative](https://github.com/speakeztech/fsnative)** implements the specification as a compiler frontend.

**[Firefly](https://github.com/speakeztech/Firefly)** generates native code that embodies these semantics.

**[Alloy](https://github.com/speakeztech/Alloy)** provides the standard library implementing spec-defined types.

## Specification Structure

The specification is organized as markdown files in the `spec/` folder:

| Chapter | Description |
|---------|-------------|
| `spec/01-introduction.md` | Scope, conformance, notation |
| `spec/02-lexical-structure.md` | Inherits from F# spec with notes on string literal encoding |
| `spec/03-types.md` | Native type definitions, layouts, alignment |
| `spec/04-ownership.md` | Ownership rules, move semantics, Copy trait |
| `spec/05-borrowing.md` | Borrow rules, lifetime constraints, borrow checking |
| `spec/06-memory-regions.md` | Stack, heap, arena, peripheral, flash |
| `spec/07-access-kinds.md` | Read-only, write-only, read-write enforcement |
| `spec/08-drop.md` | Deterministic cleanup, drop order, panic handling |
| `spec/09-srtp.md` | SRTP resolution against Alloy witnesses |
| `spec/10-coeffects.md` | Effect tracking, purity, IO |
| `spec/appendix-a-layouts.md` | Exact layouts for all built-in types |
| `spec/appendix-b-abi.md` | Calling conventions, FFI |

*Note: The specification is under active development. Not all chapters exist yet.*

## Status

| Chapter | Status |
|---------|--------|
| Introduction | Draft |
| Lexical Structure | Planning |
| Types | Active Development |
| Ownership | Active Development |
| Borrowing | Planning |
| Memory Regions | Planning |
| Access Kinds | Planning |
| Drop | Planning |
| SRTP | Planning |
| Coeffects | Future |

## Contributing

Contributions to the specification are welcome. Please open an issue to discuss significant changes before submitting a PR.

The specification follows these principles:

1. **Completeness** - Define everything the CLR would normally handle
2. **Precision** - Unambiguous definitions with exact semantics
3. **Examples** - Abstract rules accompanied by concrete illustrations
4. **Rationale** - Design decisions explain their reasoning
5. **Testability** - Specification points should be verifiable by tests

## License

This project is subject to the MIT License. See [LICENSE.txt](LICENSE.txt) for details.

## Contact

fsnative-spec is developed by [SpeakEZ Technologies](https://speakez.tech) as part of the Fidelity native compilation framework.

For questions about the specification, open an issue or reach out through the [Firefly repository](https://github.com/speakeztech/Firefly).

---

*The complete rules that make native F# safe, explicit, and deterministic.*
