---
title: "Platform Bindings and System Intrinsics"
weight: 530
category: Platform
status: normative
---

Platform bindings define the interface between Clef code and platform-specific operations. This chapter specifies the three-layer binding architecture used by CCS (Clef Compiler Service) and Firefly.

## Overview

Clef uses a **three-layer architecture** for platform operations:

| Layer | Purpose | Examples |
|-------|---------|----------|
| **Layer 1: CCS Intrinsics** | Native type universe operations | `Sys.write`, `Sys.exit` |
| **Layer 2: Binding Libraries** | External library bindings | GTK, CMSIS, OpenGL |
| **Layer 3: User Code** | Applications and libraries | User programs |

This approach:
- Avoids BCL dependencies (`System.Runtime.InteropServices`)
- Enables compile-time platform specialization
- Provides type-safe syscall interfaces
- Separates intrinsic operations from external library bindings

---

## Layer 1: CCS Intrinsics

CCS recognizes certain operations as **intrinsic** to the native type universe. These are recognized by module pattern - no declaration in user code is needed.

### The Sys Module

The `Sys` module provides direct system call primitives:

```fsharp
module Sys =
    /// Write bytes to a file descriptor
    /// fd: file descriptor (0=stdin, 1=stdout, 2=stderr)
    /// buffer: bounded stack array holding the data
    /// count: number of bytes to write
    /// Returns: number of bytes written, or negative on error
    val write : fd:int -> buffer:array<byte, 'n, Stack> -> count:int -> int

    /// Read bytes from a file descriptor
    /// fd: file descriptor
    /// buffer: bounded stack array to receive the data
    /// maxCount: maximum bytes to read
    /// Returns: number of bytes read, or negative on error
    val read : fd:int -> buffer:array<byte, 'n, Stack> -> maxCount:int -> int

    /// Exit the process with the specified code
    /// This function never returns
    val exit : code:int -> 'T
```

The buffer parameter carries the bounded stack array whose bound the compiler knows, not a raw pointer. There is no `nativeptr<byte>` surface for an intrinsic argument. The interior mechanism that carries the buffer's address to the backend leg is not user-denotable; source code names only the array.

### Buffer and Register Surfaces

Layer 1 exposes no raw-pointer module. A buffer is a bounded stack array (`array<byte, 'n, Stack>`), whose bound the compiler carries so an intrinsic reads and writes within it without a denotable pointer. A memory-mapped register is the width-typed [`Mmio`](ffi-boundary.md) handle. A pointer returned by a C binding is the opaque [`CHandle<'T>`](ffi-boundary.md), which is non-arithmetic and non-dereferenceable and exists only to be handed back across the boundary. The interior pointer mechanism is the flat closure. None of `nativeptr<'T>`, `voidptr`, `nativeint`-as-pointer, or a `NativePtr.*` operation is denotable in Clef source, at the Layer 1 boundary or anywhere else.

### Intrinsic Recognition

CCS recognizes intrinsics by module path pattern during type checking. When code calls `Sys.write`, CCS:

1. Matches the module path `Sys`
2. Matches the member name `write`
3. Returns the intrinsic's native type signature
4. Marks the call as `SemanticKind.Intrinsic` in the [SemanticGraph](program-semantic-graph.md)

The compiler (Alex) then provides platform-specific implementations during code generation.

---

## Platform-Specific Implementation

Alex provides implementations of Sys intrinsics for each target platform.

### Linux x86-64

| Intrinsic | Implementation |
|-----------|---------------|
| `Sys.write` | `syscall(1, fd, buffer, count)` |
| `Sys.read` | `syscall(0, fd, buffer, count)` |
| `Sys.exit` | `syscall(60, code)` |

Syscall convention: `rax`=syscall number, `rdi`=arg1, `rsi`=arg2, `rdx`=arg3

### Linux ARM64

| Intrinsic | Implementation |
|-----------|---------------|
| `Sys.write` | `svc #0` with `x8=64` |
| `Sys.read` | `svc #0` with `x8=63` |
| `Sys.exit` | `svc #0` with `x8=93` |

### macOS x86-64

| Intrinsic | Implementation |
|-----------|---------------|
| `Sys.write` | `syscall(0x2000004, fd, buffer, count)` |
| `Sys.read` | `syscall(0x2000003, fd, buffer, count)` |
| `Sys.exit` | `syscall(0x2000001, code)` |

Note: macOS x86-64 uses BSD syscall numbers with `0x2000000` offset.

### macOS ARM64

| Intrinsic | Implementation |
|-----------|---------------|
| `Sys.write` | `svc #0x80` with `x16=4` |
| `Sys.read` | `svc #0x80` with `x16=3` |
| `Sys.exit` | `svc #0x80` with `x16=1` |

### Windows x86-64

| Intrinsic | Implementation |
|-----------|---------------|
| `Sys.write` | `WriteFile` via ntdll |
| `Sys.read` | `ReadFile` via ntdll |
| `Sys.exit` | `NtTerminateProcess` |

### Freestanding

For bare-metal targets, intrinsics may:
- Map to hardware registers
- Generate inline assembly
- Require target-specific configuration

---

## Layer 2: Binding Libraries

External library bindings (GTK, CMSIS, OpenGL, etc.) require **rich semantic metadata** that CCS cannot know intrinsically:

- Memory layouts and alignment
- Ownership semantics (managed, unmanaged, refcounted)
- Volatile access requirements
- Callback calling conventions
- Register mappings (for hardware peripherals)
- FFI calling conventions

### The Quotation Solution

F# quotations (`<@ ... @>`) are **compile-time inspectable data structures**. Unlike regular code which compiles to instructions, quotations compile to expression trees that can be examined during compilation.

This makes quotations ideal for carrying binding metadata:
- Generated by Farscape from C/C++ headers
- Compiled as regular F# code
- Inspected at compile time by CCS
- Never executed at runtime

### How Quotation Binding Works

#### Step 1: Farscape Generates Binding Library

Farscape parses C/C++ headers and generates F# binding libraries:

```fsharp
// Generated by Farscape from gtk.h
module Gtk.Bindings

open BAREWire.Descriptors
open Memory

/// Type descriptor - quotation carries layout and semantics
let gtkWindowDescriptor: Expr<TypeDescriptor> = <@
    { TypeName = "GtkWindow"
      CName = "GtkWindow"
      Layout = { Size = 24un; Alignment = 8un }
      Ownership = Unmanaged
      RefCounted = true
      Destructor = Some "gtk_widget_destroy" }
@>

/// Function descriptor - quotation carries calling convention
let gtkWindowNewDescriptor: Expr<FunctionDescriptor> = <@
    { CName = "gtk_window_new"
      Parameters = [
          { Name = "type"; Type = I32; PassBy = Value }
      ]
      ReturnType = Handle gtkWindowDescriptor
      CallingConvention = CDecl
      OwnershipTransfer = CallerOwns }
@>

/// The callable function - references the descriptor
let windowNew (windowType: int) : CHandle<GtkWindow> =
    // Body references descriptor, enabling CCS to find metadata
    failwith "Binding placeholder"
```

#### Step 2: CCS Inspects Quotations at Compile Time

When CCS encounters a call to a binding function, it:

1. **Recognizes the binding pattern** by module structure
2. **Finds associated quotations** by naming convention (`*Descriptor`)
3. **Inspects quotation structure** to extract metadata
4. **Attaches metadata to SemanticGraph nodes**

```
User code: let window = Gtk.windowNew 0
                ↓
CCS: "This calls Gtk.Bindings.windowNew"
                ↓
CCS: "Find associated descriptor quotation"
                ↓
CCS: Inspects <@ { CName = "gtk_window_new"; ... } @>
                ↓
SemanticGraph node gets:
  - FFI.CName = "gtk_window_new"
  - FFI.CallingConvention = CDecl
  - FFI.OwnershipTransfer = CallerOwns
  - MemoryRegion = Unmanaged
```

#### Step 3: Alex Uses Metadata for Code Generation

The metadata flows from SemanticGraph to Alex:

```
SemanticGraph node (with FFI metadata)
                ↓
Alex sees: "FFI call to gtk_window_new, CDecl, returns owned pointer"
                ↓
Alex emits the portable call node; a backend leg emits the target-specific call
with the correct ABI and ownership tracking (LLVM being one such leg)
```

### Active Patterns for Recognition

Binding libraries also provide active patterns for PSG traversal:

```fsharp
/// Active pattern for matching GTK window creation
let (|GtkWindowCreate|_|) (node: SemanticNode) =
    match node.Kind with
    | Application(funcNode, args) when
        funcNode.Symbol = Some "Gtk.Bindings.windowNew" ->
        Some { WindowType = extractArg args 0 }
    | _ -> None

/// Active pattern for matching GTK signal connection
let (|GtkSignalConnect|_|) (node: SemanticNode) =
    match node.Kind with
    | Application(funcNode, args) when
        funcNode.Symbol = Some "Gtk.Bindings.signalConnect" ->
        let widget = extractArg args 0
        let signal = extractArg args 1
        let callback = extractArg args 2
        Some { Widget = widget; Signal = signal; Callback = callback }
    | _ -> None
```

These patterns enable Alex to recognize and handle specific binding patterns during code generation.

### Quotation Structure Requirements

Binding quotations must follow specific structure for CCS inspection:

```fsharp
/// Type descriptor quotation
type TypeDescriptor = {
    TypeName: string          // F# type name
    CName: string             // C/C++ type name
    Layout: LayoutInfo        // Size, alignment
    Ownership: OwnershipKind  // Managed | Unmanaged | RefCounted
    RefCounted: bool          // Uses reference counting
    Destructor: string option // Cleanup function name
}

/// Function descriptor quotation
type FunctionDescriptor = {
    CName: string                 // C function name
    Parameters: ParameterInfo[]   // Parameter types and passing
    ReturnType: TypeRef           // Return type reference
    CallingConvention: CallConv   // CDecl | StdCall | FastCall
    OwnershipTransfer: Transfer   // CallerOwns | CalleeOwns | Borrowed
}

/// Hardware register descriptor (for embedded)
type RegisterDescriptor = {
    Name: string              // Register name
    Address: usize            // Memory-mapped address (platform-word integer, as the Mmio handle carries it)
    AccessKind: AccessKind    // ReadOnly | WriteOnly | ReadWrite | Volatile
    ResetValue: uint32        // Value after reset
    Fields: FieldInfo[]       // Bit field definitions
}
```

### Why Quotations, Not Attributes?

| Approach | Limitation |
|----------|------------|
| **Attributes** | Limited to simple values (strings, numbers) |
| **Interfaces** | Require runtime dispatch |
| **Reflection** | Requires runtime, BCL dependency |
| **Quotations** | Full F# expressions, compile-time inspectable, BCL-free |

Quotations can express:
- Nested structures (layouts containing fields)
- References to other types (pointer to TypeDescriptor)
- Complex expressions (computed offsets, conditional layouts)
- All without runtime overhead

---

## Layer 3: User Code

User code uses intrinsics and binding libraries. It does NOT declare platform bindings.

### Correct Usage Pattern

```fsharp
// Console module - uses CCS intrinsics directly
module Console

let inline write (s: string) : unit =
    // s.Bytes is the string's bounded backing array; no raw pointer is named
    Sys.write 1 s.Bytes s.Length |> ignore

let inline writeln (s: string) : unit =
    write s
    let nl : array<byte, 1, Stack> = [| 0x0Auy |]
    Sys.write 1 nl 1 |> ignore
```

### Incorrect Pattern (Deprecated)

The following pattern is **deprecated** and should not be used:

```fsharp
// WRONG - Do not declare platform bindings with BCL stubs
module Platform.Bindings =
    let writeBytes fd buffer count : int =
        Unchecked.defaultof<int>  // BCL dependency!
 
```

This pattern was used historically but creates BCL dependencies and requires special handling in the compiler.

---

## Standard File Descriptors

Standard file descriptors on Unix-like systems:

| Descriptor | Value | Purpose |
|------------|-------|---------|
| `stdin` | 0 | Standard input |
| `stdout` | 1 | Standard output |
| `stderr` | 2 | Standard error |

---

## Intrinsic Constraints

CCS intrinsics have restrictions:

1. **No closures**: Intrinsics cannot capture environment
2. **Primitive or sanctioned-handle types only**: Arguments and returns must be primitive types, a bounded stack array, the `Mmio` register handle, or a `CHandle<'T>`; there is no raw-pointer argument type
3. **No exceptions**: Errors returned via return values
4. **No dynamic allocation**: Intrinsics do not allocate on a heap. A buffer they operate on is a scope-bounded stack array or program-lifetime static storage, per the [lifetime lattice](closure-representation.md); an intrinsic never introduces a genuinely-dynamic allocation, so it remains usable on a no-heap freestanding target
5. **No currying**: Intrinsics must be called with all arguments

---

## Diagnostics

| Code | Message |
|------|---------|
| CCS8030 | Platform intrinsic not available for target |
| CCS8031 | Invalid intrinsic signature |
| CCS8032 | Intrinsic requires primitive types |
| CCS8033 | Intrinsic called with partial application |

---

## Platform Descriptor

The **Platform Descriptor** is a quotation-based structure that defines all platform-specific characteristics. It flows from `Fidelity.Platform` through CCS to Alex, enabling compile-time platform specialization.

### Structure

```fsharp
type PlatformDescriptor = {
    Architecture: Architecture          // X86_64, ARM64, RISCV64, etc.
    OperatingSystem: OperatingSystem    // Linux, Windows, MacOS, BareMetal
    Dimensions: Map<WidthDimension, int> // Pointer → 64, Register → 64, etc.
    Endianness: Endianness              // Little or Big
    TypeLayouts: Map<string, TypeLayout>  // Type sizes and alignments
    SyscallConvention: SyscallConvention  // Syscall ABI
    MemoryRegions: MemoryRegion list      // Per-target; e.g. Stack, Text, Data, and Heap only where the target has an allocator
    FreestandingStartup: FreestandingStartup option  // Entry point for freestanding mode
}
```

`MemoryRegions` is per-target. A hosted target lists `Stack`, `Heap`, `Text`, `Data`. A freestanding target with no allocator has no `Heap`: it lists `Stack` for scope-bounded values and static storage (`.bss`-class `Sram`/`Flash`) for program-lifetime values. A value that would classify as genuinely dynamic on such a target is a compile-time lifetime error, not a silent heap allocation. See the four-point lifetime lattice in [closure-representation.md §3.3](closure-representation.md).

### Freestanding Startup

For freestanding builds (no libc), the platform descriptor includes startup information:

```fsharp
type FreestandingStartup =
    /// Hosted freestanding ELF (Linux, no libc): a named entry symbol the linker
    /// points at, and a numbered exit syscall. "_start"/60 is this convention, not
    /// a universal freestanding one.
    | HostedElf of EntrySymbol: string * ExitSyscall: int64
    /// Bare-metal M33 unikernel: control arrives at the reset vector, there is no
    /// _start symbol, no linker entry flag, no argv, and no syscall. Exit is a halt.
    | ResetVector of ResetHandler: string
```

The startup shape is per-target. `_start` plus a numbered exit syscall is the hosted-ELF freestanding convention; a bare-metal target has neither.

When `output_kind = "freestanding"` is specified in the project file for a hosted-ELF target, the compiler:

1. Generates a `_start` wrapper function
2. `_start` creates an empty string array, calls the F# `main`, and calls `Sys.exit`
3. Links with `-Wl,-e,_start` to set the entry point

For a bare-metal M33 unikernel target the compiler instead:

1. Emits the reset handler as the reset-vector entry (no `_start` symbol, no linker entry flag)
2. The reset handler calls the F# `main` with no `argv` (arguments do not exist)
3. Return from `main` is a halt (a `wfi`/spin), not an exit syscall

### Console Mode

For console builds (with libc), no special entry point handling is needed:
- libc provides `_start` which initializes the runtime and calls `main`
- The F# `main` function is emitted with C-compatible signature
- The F# type `array<string> -> int` maps to the platform C ABI

### Linux x86-64 Example (hosted ELF)

The `SyscallConvention` and `FreestandingStartup` below are the hosted-ELF freestanding shape: SysV argument registers, a `_start` entry symbol, and a numbered exit syscall. These belong to this target, not to freestanding mode in general (the M33 example that follows has none of them).

```fsharp
let platform: Expr<PlatformDescriptor> = <@
    { Architecture = X86_64
      OperatingSystem = Linux
      Dimensions = Map.ofList [ (Pointer, 64); (Register, 64) ]
      Endianness = Little
      TypeLayouts = (* ... *)
      SyscallConvention =
        { CallingConvention = SysV_AMD64
          ArgRegisters = [| RDI; RSI; RDX; R10; R8; R9 |]
          ReturnRegister = RAX
          SyscallNumberRegister = RAX
          SyscallInstruction = Syscall }
      MemoryRegions = (* ... *)
      FreestandingStartup = Some (HostedElf ("_start", 60L))
    }
@>
```

### Bare-Metal M33 Example (reset-vector entry)

The Cortex-M33 unikernel target has no syscall ABI and no `_start`. `Pointer` is the 4-byte platform word, there is no `Heap` region, and startup is the reset vector.

```fsharp
let platform: Expr<PlatformDescriptor> = <@
    { Architecture = ARM_Thumbv8m
      OperatingSystem = BareMetal
      Dimensions = Map.ofList [ (Pointer, 32); (Register, 32) ]
      Endianness = Little
      TypeLayouts = (* ... *)
      SyscallConvention = NoSyscalls
      MemoryRegions = (* Stack + static Flash/Sram; no Heap *)
      FreestandingStartup = Some (ResetVector "Reset_Handler")
    }
@>
```

### Compile-Time Resolution

The platform descriptor is inspected at compile time:

1. **CCS** reads the platform descriptor from `Fidelity.Platform`
2. For freestanding mode, **Intrinsic Elaboration** generates the `_start` wrapper
3. The wrapper uses `Sys.emptyStringArray` and `Sys.exit` intrinsics
4. Alex emits portable dialects (`func`, `cf`, `scf`, `arith`, `memref`, `index`, `builtin`), committing to no target; a backend leg lowers them and supplies the entry glue. On the hosted x86-64 leg that glue is an `_start` symbol the linker points at via `-Wl,-e,_start`; the M33 unikernel leg has no linker entry flag and no `_start` (control arrives at the reset vector).

The F# code author writes idiomatic F# (`main: string[] -> int`); the compiler handles entry point generation based on the platform and output mode

---
