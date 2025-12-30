# Platform Bindings

Platform bindings define the interface between F# Native code and platform-specific operations. The compiler (Alex) provides implementations for each target platform.

## Overview

F# Native uses a **module convention** for platform bindings rather than P/Invoke or FFI attributes. This approach:

- Avoids BCL dependencies (`System.Runtime.InteropServices`)
- Enables compile-time platform specialization
- Provides type-safe syscall interfaces

## The Platform.Bindings Module

Alloy defines binding points in `Platform.Bindings`:

```fsharp
module Platform.Bindings =
    val writeBytes : int -> nativeptr<byte> -> int -> int
    val readBytes : int -> nativeptr<byte> -> int -> int
    val getCurrentTicks : unit -> int64
    val sleep : int -> unit
    val exit : int -> unit
```

These functions have **no F# implementation**. The compiler recognizes them and generates platform-specific code.

## Binding Recognition

The compiler identifies platform bindings by:

1. Module path: `Platform.Bindings.*`
2. Function signature matching
3. No implementation body (returns `Unchecked.defaultof<_>`)

```fsharp
// In Alloy - binding declaration
module Platform.Bindings =
    let writeBytes fd buffer count : int = 
        Unchecked.defaultof<int>  // Placeholder - compiler provides impl
```

## Platform-Specific Implementation

The compiler (Alex) provides implementations per target:

### Linux x86-64

| Binding | Implementation |
|---------|---------------|
| `writeBytes` | `syscall(1, fd, buffer, count)` (write) |
| `readBytes` | `syscall(0, fd, buffer, count)` (read) |
| `getCurrentTicks` | `syscall(228, ...)` (clock_gettime) |
| `sleep` | `syscall(35, ...)` (nanosleep) |
| `exit` | `syscall(60, code)` (exit) |

### Linux ARM64

| Binding | Implementation |
|---------|---------------|
| `writeBytes` | `svc #0` with x8=64 |
| `readBytes` | `svc #0` with x8=63 |
| `exit` | `svc #0` with x8=93 |

### Windows x86-64

| Binding | Implementation |
|---------|---------------|
| `writeBytes` | `WriteFile` via ntdll |
| `readBytes` | `ReadFile` via ntdll |
| `exit` | `NtTerminateProcess` |

### Freestanding

For bare-metal targets, bindings may:
- Map to hardware registers
- Generate inline assembly
- Require target-specific configuration

## Standard Bindings

### I/O Bindings

```fsharp
/// Write bytes to a file descriptor
/// Returns: Number of bytes written, or negative on error
val writeBytes : fd:int -> buffer:nativeptr<byte> -> count:int -> int

/// Read bytes from a file descriptor
/// Returns: Number of bytes read, or negative on error
val readBytes : fd:int -> buffer:nativeptr<byte> -> maxCount:int -> int
```

### Time Bindings

```fsharp
/// Get current time in ticks (platform-specific resolution)
val getCurrentTicks : unit -> int64

/// Sleep for specified milliseconds
val sleep : milliseconds:int -> unit
```

### Process Bindings

```fsharp
/// Exit the process with the specified code
val exit : code:int -> unit
```

### Memory Bindings

```fsharp
/// Allocate memory from the system
val allocateMemory : size:unativeint -> nativeptr<byte>

/// Free memory to the system
val freeMemory : ptr:nativeptr<byte> -> unit
```

## File Descriptors

Standard file descriptors:

| Descriptor | Value | Purpose |
|------------|-------|---------|
| `stdin` | 0 | Standard input |
| `stdout` | 1 | Standard output |
| `stderr` | 2 | Standard error |

## Adding Custom Bindings

Custom platform bindings follow the same pattern:

```fsharp
// In application code
module MyPlatform.Bindings =
    let customSyscall arg1 arg2 : int = 
        Unchecked.defaultof<int>
```

The compiler must be configured to recognize custom binding modules.

## Binding Constraints

Platform bindings have restrictions:

1. **No closures**: Bindings cannot capture environment
2. **Primitive types only**: Arguments and returns must be primitive or pointer types
3. **No exceptions**: Errors returned via return values
4. **No allocation**: Bindings do not allocate managed memory

## Diagnostics

| Code | Message |
|------|---------|
| FS8030 | Platform binding not available for target |
| FS8031 | Invalid platform binding signature |
| FS8032 | Platform binding requires primitive types |

## See Also

- [The Native Library Alloy](the-native-library-alloy.md) - Standard library using bindings
- [Memory Regions](memory-regions.md) - Pointer types for bindings
- [Access Kinds](access-kinds.md) - Pointer access semantics
