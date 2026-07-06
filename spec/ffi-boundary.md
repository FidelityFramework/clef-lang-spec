---
title: "FFI Boundary Semantics"
weight: 670
category: Platform
status: normative
---

This chapter defines the Foreign Function Interface (FFI) boundary between Clef code and external C libraries on a target that links a host C runtime (Lane 1, hosted). It establishes the null-safety contract, the opaque-handle representation of a C binding's pointer, and the normative requirements for binding generation tools like Farscape. The FFI leg exists whenever a C runtime is linked, and its presence is independent of whether the target is freestanding:

- **Hosted** — libc is linked dynamically; the FFI boundary of this chapter applies as written.
- **Freestanding with static libc** — libc is linked statically and its resources are accessed directly, with no dynamic linker in the image. The FFI leg still exists and this chapter still applies; static coupling removes the dynamic-binding step (a supply-chain consideration developed in the platform documentation, out of scope here).
- **Bare (no C runtime)** — no libc is linked at all, so there is no FFI leg and nothing in this chapter applies. Interior memory follows the lifetime lattice defined in `closure-representation.md` §3.3.

Interior Clef has no raw pointer type. `nativeptr<'T>`, `voidptr`, and `nativeint`-as-pointer are not denotable in Clef source, at the FFI boundary or anywhere else. A C binding that returns a pointer marshals that pointer through an opaque handle, `CHandle<'T>`: the handle is non-arithmetic and non-dereferenceable in Clef source, and its only use is to be passed back across the boundary to another C binding. The interior pointer mechanism is the flat closure; a register is the width-typed `Mmio` handle.

## 1. Null Safety Principle

### 1.1 Core Invariant

> **Null exists ONLY at the FFI boundary. Within Clef code, `CHandle<'T>` and `FnPtr<'F>` are NEVER null.**

This invariant is fundamental to Clef's memory safety guarantees. Unlike C where any pointer may be null, Clef enforces non-nullability at the type level. A C binding hands back an opaque handle rather than a raw pointer, so interior code never holds a dereferenceable address.

### 1.2 Rationale

Null pointer dereferences are a leading cause of crashes and security vulnerabilities in native code. By eliminating null from the type system's interior, Clef provides:

1. **Compile-time safety**: The type checker ensures handles handed back from C are always valid
2. **Explicit optionality**: `Option<CHandle<'T>>` makes nullability visible in the type signature
3. **Clean FFI boundary**: Null handling is isolated to the interface with C code
4. **No runtime null checks**: Interior code needs no defensive null checks

### 1.3 The FFI Boundary

The FFI boundary is the interface between Clef code and external C functions. At this boundary:

- **Outgoing** (F# → C): `Option<CHandle<'T>>` converts to nullable C pointer
  - `None` → `NULL`
  - `Some handle` → the pointer the handle carries

- **Incoming** (C → F#): Nullable C pointer converts to `Option<CHandle<'T>>`
  - `NULL` → `None`
  - Non-null → `Some handle`

```
┌─────────────────────────────────────────────────────────┐
│  Clef World                                        │
│                                                         │
│  CHandle<'T>            - NEVER null, opaque           │
│  FnPtr<'F>              - NEVER null                   │
│  Option<CHandle<'T>>    - explicit nullability         │
│  Option<FnPtr<'F>>      - explicit nullability         │
└─────────────────────────────────────────────────────────┘
                         ↕ FFI Boundary
┌─────────────────────────────────────────────────────────┐
│  C World                                                │
│                                                         │
│  T*                    - may be NULL                   │
│  void (*f)(...)        - may be NULL                   │
└─────────────────────────────────────────────────────────┘
```

## 2. Pointer Types at FFI Boundary

### 2.1 Non-Nullable Handles

| Clef Type | C Equivalent | Semantics |
|---------------|--------------|-----------|
| `CHandle<'T>` | `T*` (non-null) | Opaque typed handle, guaranteed valid; non-arithmetic, non-dereferenceable in Clef source |
| `CHandle<unit>` | `void*` (non-null) | Opaque untyped handle, guaranteed valid |
| `FnPtr<'F>` | Function pointer (non-null) | Function pointer, guaranteed valid |

A `CHandle<'T>` carries a C pointer across the boundary but exposes no pointer operations in Clef source: no arithmetic, no dereference, no conversion to an integer. Its only role is to be handed back to another C binding. These types have no null representation, and attempting to construct a null value is a compile-time error.

### 2.2 Nullable Handles (FFI Only)

| Clef Type | C Equivalent | Semantics |
|---------------|--------------|-----------|
| `Option<CHandle<'T>>` | `T*` (nullable) | May be null, explicit handling required |
| `Option<CHandle<unit>>` | `void*` (nullable) | Untyped nullable handle |
| `Option<FnPtr<'F>>` | Function pointer (nullable) | May be null callback |

`Option` wrapping is used ONLY at FFI boundaries where C semantics require nullable pointers.

### 2.3 Memory Layout

`Option<CHandle<'T>>` has the same memory layout as `CHandle<'T>` (a single platform word). The compiler uses the null pointer optimization:

- `None` is represented as the bit pattern `0` (null)
- `Some handle` is represented as the carried pointer value itself

This ensures zero overhead for Option-wrapped handles at the FFI boundary.

## 3. FnPtr Intrinsics

### 3.1 FnPtr Type

`FnPtr<'F>` is a function pointer type where `'F` is the full function signature:

```fsharp
FnPtr<unit -> unit>                              // void (*)(void)
FnPtr<int -> int>                                // int (*)(int)
FnPtr<CHandle<byte> -> int -> int>               // int (*)(char*, int)
FnPtr<Option<CHandle<int>> -> unit>              // void (*)(int*)  -- nullable param
 
```

The type parameter `'F` MUST be a function type (`'a -> 'b`). Using a non-function type is a compile-time error.

### 3.2 FnPtr.fromSymbol

Declares an external symbol to be resolved by the linker.

**Signature:**
```fsharp
FnPtr.fromSymbol<'F> : string -> FnPtr<'F>
```

**Semantics:**
- The string argument MUST be a compile-time constant (string literal)
- Returns a non-null function pointer (linker guarantees symbol exists)
- Symbol resolution occurs at link time, not runtime

**Example:**
```fsharp
// Declare external C functions
let private strlen_ptr = FnPtr.fromSymbol<CHandle<byte> -> int> "strlen"
let private gtk_init_ptr =
    FnPtr.fromSymbol<Option<CHandle<int>> -> Option<CHandle<CHandle<byte>>> -> unit> "gtk_init"
```

**Code Generation:**
The middle end emits portable dialects only: an external `func.func` declaration for the symbol, and the symbol address carried as a `builtin.unrealized_conversion_cast` (a function address is data with no portable form). Each backend leg realizes that cast: the LLVM leg (CPU/MCU) lowers the declaration to an `llvm.func` and the address to an `llvm.mlir.addressof`; other legs realize it in their own terms.

### 3.3 FnPtr.invoke

Calls a function through a function pointer.

**Signature:**
```fsharp
FnPtr.invoke : FnPtr<'F> -> 'F
```

**Semantics:**
- Invokes the function with the provided arguments
- Arguments matching `Option<CHandle<'T>>` are marshalled (None → NULL)
- Return values matching `Option<CHandle<'T>>` are marshalled (NULL → None)

**Example:**
```fsharp
// Call external function
let len = FnPtr.invoke strlen_ptr myStringPtr

// Call with nullable arguments (None → NULL)
FnPtr.invoke gtk_init_ptr None None
```

### 3.4 FnPtr.ofFunction

Converts a top-level F# function to a function pointer (for callbacks).

**Signature:**
```fsharp
FnPtr.ofFunction : 'F -> FnPtr<'F>
```

**Constraints:**
- The argument MUST be a reference to a module-level `let` binding
- Lambdas and closures are REJECTED at compile time
- The function must not capture any environment

**Rationale:** C callbacks expect stable function addresses. Closures capture environment with unpredictable lifetime. Compile-time enforcement prevents subtle bugs.

**Example:**
```fsharp
// OK - top-level function
let myCallback (x: int) : int = x + 1
let callbackPtr = FnPtr.ofFunction myCallback

// ERROR - lambda (even without captures)
let ptr = FnPtr.ofFunction (fun x -> x + 1)  // Compile error

// ERROR - closure with captures
let multiplier = 2
let ptr = FnPtr.ofFunction (fun x -> x * multiplier)  // Compile error
 
```

### 3.5 Removed Intrinsics

The following intrinsics are NOT available in Clef:

- ~~`FnPtr.null`~~: Use `Option<FnPtr<'F>>` with `None` instead
- ~~`FnPtr.isNull`~~: Use pattern matching on `Option<FnPtr<'F>>` instead

**Migration:**
```fsharp
// Old (NOT SUPPORTED):
let maybeCallback = FnPtr.null<int -> unit> ()
if not (FnPtr.isNull maybeCallback) then
    FnPtr.invoke maybeCallback 42

// New (CORRECT):
let maybeCallback : Option<FnPtr<int -> unit>> = None
match maybeCallback with
| Some cb -> FnPtr.invoke cb 42
| None -> ()
```

## 4. Option↔NULL Marshalling

### 4.1 Parameter Marshalling (F# → C)

When a function parameter has type `Option<CHandle<'T>>` or `Option<FnPtr<'F>>`:

| F# Value | C Value |
|----------|---------|
| `None` | `NULL` (0) |
| `Some handle` | the pointer the handle carries |

**Optimization:** When the argument is a compile-time `None` literal, the compiler directly emits null without runtime checks.

### 4.2 Return Value Marshalling (C → F#)

When a function return type is `Option<CHandle<'T>>` or `Option<FnPtr<'F>>`:

| C Value | F# Value |
|---------|----------|
| `NULL` (0) | `None` |
| Non-null | `Some handle` |

**Code Generation (portable dialects):**
The compare-against-null and the select are expressed in portable ops. The returned pointer is a platform word, typed `index`; the null check is `arith.cmpi`; the choice between `None` and `Some` is `scf.select` (or `arith.select`).
```mlir
// C function returns nullable pointer (platform word, carried as index)
%result = func.call @may_return_null() : () -> index

// Marshal to Option
%zero = arith.constant 0 : index
%is_null = arith.cmpi eq, %result, %zero : index
%option = arith.select %is_null, %none_value, %some_result : ...
```

**LLVM-leg lowering example:** on the CPU/MCU leg the same marshalling lowers to `llvm.call ... -> !llvm.ptr`, `llvm.icmp "eq"`, and `llvm.select`. That form is one target leg, not what the middle end emits.

### 4.3 Non-Marshalled Types

Types NOT wrapped in `Option` are passed directly without marshalling:

- `CHandle<'T>`: passed as-is (must be non-null)
- `FnPtr<'F>`: passed as-is (must be non-null)
- `int`, `float`, etc.: passed as-is (value types)

## 5. Farscape Binding Generation Contract

This section defines normative requirements for Farscape and other binding generation tools.

> *Informative.* The [C++ Binding via Farscape](https://clef-lang.com/docs/internals/farscape/binding-cpp-to-clef-in-farscape/) guide describes how these requirements are applied in practice.

### 5.1 C Nullability Annotation Mapping

Farscape MUST interpret C nullability annotations as follows:

| C Annotation | Platform | Clef Output |
|-------------|----------|------------------|
| `_Nonnull` | Clang/Apple | `CHandle<'T>` |
| `_Nullable` | Clang/Apple | `Option<CHandle<'T>>` |
| `_Null_unspecified` | Clang/Apple | See default policy |
| `__attribute__((nonnull))` | GCC | `CHandle<'T>` |
| `_In_` | Windows SAL | `CHandle<'T>` |
| `_In_opt_` | Windows SAL | `Option<CHandle<'T>>` |
| `_Out_` | Windows SAL | `CHandle<'T>` |
| `_Out_opt_` | Windows SAL | `Option<CHandle<'T>>` |

### 5.2 Default Policy (Unannotated Pointers)

When C code lacks nullability annotations, Farscape MUST apply these defaults:

**Function Parameters:**
- Default: `CHandle<'T>` (assume non-null)
- Override to `Option<CHandle<'T>>` when documentation indicates nullable

**Function Return Values:**
- Default: `CHandle<'T>` (assume non-null)
- Override to `Option<CHandle<'T>>` for functions documented to return NULL on error

**Rationale:** Most C APIs expect non-null parameters and return non-null on success. Defaulting to non-null reduces Option ceremony while the override mechanism handles exceptions.

### 5.3 Generated Binding Structure

Farscape-generated bindings MUST follow this pattern:

```fsharp
module LibraryName.Bindings

// External function declarations (private)
let private function_name_ptr =
    FnPtr.fromSymbol<param_types -> return_type> "c_function_name"

// High-level F# API (public)
let functionName (param1: type1) (param2: type2) : returnType =
    FnPtr.invoke function_name_ptr param1 param2
```

### 5.4 Ambiguity Handling

When nullability is ambiguous, Farscape SHOULD:

1. Emit a warning indicating the assumption made
2. Support a hints file for manual override
3. Document the default in generated binding comments

**Hints File Format (example):**
```toml
[gtk_window_new]
return = "nonnull"  # Override: gtk_window_new never returns NULL

[g_object_get_data]
return = "nullable"  # Override: may return NULL if key not found
 
```

### 5.5 Callback Function Types

For C functions accepting callbacks, Farscape MUST:

1. Generate `FnPtr<'F>` for non-null callback parameters
2. Generate `Option<FnPtr<'F>>` for nullable callback parameters
3. Document that callbacks must be top-level functions (no closures)

## 6. Examples

### 6.1 GTK Bindings

```fsharp
module Platform.GTK

// External declarations
let private gtk_init_ptr =
    FnPtr.fromSymbol<Option<CHandle<int>> -> Option<CHandle<CHandle<byte>>> -> unit> "gtk_init"

let private gtk_window_new_ptr =
    FnPtr.fromSymbol<int -> CHandle<GtkWindow>> "gtk_window_new"

let private gtk_widget_show_all_ptr =
    FnPtr.fromSymbol<CHandle<GtkWidget> -> unit> "gtk_widget_show_all"

let private gtk_main_ptr =
    FnPtr.fromSymbol<unit -> unit> "gtk_main"

// High-level API
let gtkInit () =
    FnPtr.invoke gtk_init_ptr None None

let gtkWindowNew (windowType: GtkWindowType) : CHandle<GtkWindow> =
    FnPtr.invoke gtk_window_new_ptr (int windowType)

let gtkWidgetShowAll (widget: CHandle<GtkWidget>) : unit =
    FnPtr.invoke gtk_widget_show_all_ptr widget

let gtkMain () =
    FnPtr.invoke gtk_main_ptr ()
```

### 6.2 libc Bindings

This example is hosted-libc (Lane 1). `malloc`/`free` exist only where a C runtime with a heap is linked. On a no-heap target these particular bindings do not exist, and interior memory follows the lifetime lattice of `closure-representation.md` §3.3: a value that classifies as genuinely-dynamic on a no-heap target is a compile-time lifetime error rather than a call to `malloc`. A bare target with no C runtime at all has no FFI leg of any kind.

```fsharp
module Platform.Libc

// strlen: never returns null, never accepts null
let private strlen_ptr =
    FnPtr.fromSymbol<CHandle<byte> -> int> "strlen"

// malloc: may return NULL on failure
let private malloc_ptr =
    FnPtr.fromSymbol<int -> Option<CHandle<unit>>> "malloc"

// free: accepts NULL (no-op)
let private free_ptr =
    FnPtr.fromSymbol<Option<CHandle<unit>> -> unit> "free"

// High-level API
let strlen (s: CHandle<byte>) : int =
    FnPtr.invoke strlen_ptr s

let malloc (size: int) : Option<CHandle<unit>> =
    FnPtr.invoke malloc_ptr size

let free (handle: Option<CHandle<unit>>) : unit =
    FnPtr.invoke free_ptr handle
```

### 6.3 Callback Pattern

```fsharp
module Platform.GLib

// Type alias for GLib callback
type GSourceFunc = int -> int  // gboolean (*)(gpointer) simplified

// g_idle_add accepts non-null callback; user_data is a nullable gpointer (void*)
let private g_idle_add_ptr =
    FnPtr.fromSymbol<FnPtr<GSourceFunc> -> Option<CHandle<unit>> -> uint32> "g_idle_add"

// User's callback (must be top-level)
let myIdleCallback (userData: int) : int =
    // Do work...
    0  // Return FALSE to remove source

// Register callback (no user_data -> None -> NULL)
let sourceId =
    let callbackPtr = FnPtr.ofFunction myIdleCallback
    FnPtr.invoke g_idle_add_ptr callbackPtr None
```

## 7. Normative Summary

1. A C binding's pointer marshals through the opaque `CHandle<'T>`; interior Clef has no raw pointer type, and `nativeptr<'T>`/`voidptr`/`nativeint`-as-pointer are not denotable anywhere. `CHandle<'T>` and `FnPtr<'F>` are NEVER null within Clef code
2. `Option<CHandle<'T>>` and `Option<FnPtr<'F>>` represent nullable pointers at the FFI boundary
3. `FnPtr.fromSymbol` declares linker-resolved external symbols
4. `FnPtr.invoke` calls through function pointers with automatic Option↔NULL marshalling
5. `FnPtr.ofFunction` converts top-level functions only (no closures)
6. Farscape MUST follow the nullability annotation mapping and default policies defined herein
7. This chapter applies wherever a host C runtime is linked, whether dynamically (hosted) or statically (freestanding with static libc); a bare target with no C runtime has no FFI leg, and its interior memory follows the lifetime lattice of `closure-representation.md` §3.3
