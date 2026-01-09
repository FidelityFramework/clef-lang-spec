# The Native Library Alloy

Alloy is the native standard library for F# Native compilation. It provides BCL-sympathetic APIs that compile to native code with deterministic memory management.

> **Note**: This chapter specifies the Alloy library interface. For implementation details, see the [Alloy repository](https://github.com/speakez-codespace/Alloy).

## Overview

Where .NET F# programs reference `FSharp.Core.dll` and `mscorlib.dll`, F# Native programs reference **Alloy**. Alloy provides:

- Familiar API surface (Console, String, Array, etc.)
- Native type implementations (fat pointers, stack allocation)
- BCL-free implementation using FNCS intrinsics
- No garbage collector dependency

## Architectural Position

Alloy is **Layer 3** in the binding architecture - it is pure F# code that uses FNCS intrinsics:

```
Layer 1: FNCS Intrinsics (Sys.write, NativePtr.set, etc.)
    ↑
Layer 2: Binding Libraries (Farscape-generated)
    ↑
Layer 3: User Code (Alloy, applications)  ← Alloy is here
```

Alloy does NOT declare platform bindings. It uses `Sys.*` intrinsics directly.

> **See**: [Platform Bindings](platform-bindings.md) for the three-layer architecture.

## Automatically Opened Namespaces

The following namespaces are automatically opened for all F# Native code:

```fsharp
open Alloy
open Alloy.Core
open Alloy.Operators
open Alloy.Collections
```

## Basic Types

### Type Abbreviations

| Type Name | Native Representation | Notes |
|-----------|----------------------|-------|
| `unit` | Zero-sized | Identical to standard F# |
| `bool` | `i8` | 0 = false, 1 = true |
| `int` | Platform word | `nativeint` semantics |
| `int32` | `i32` | Fixed 32-bit |
| `int64` | `i64` | Fixed 64-bit |
| `float` | `f64` | IEEE 754 double |
| `float32` | `f32` | IEEE 754 single |
| `char` | `i32` | UTF-32 codepoint |
| `string` | `{ptr, len}` | UTF-8 fat pointer |

### Types with Unit of Measure Support

| Type Name | Description |
|-----------|-------------|
| `int<'u>` | Platform integer with unit of measure |
| `int32<'u>` | 32-bit integer with unit of measure |
| `int64<'u>` | 64-bit integer with unit of measure |
| `float<'u>` | Double with unit of measure |
| `float32<'u>` | Single with unit of measure |

## Core Modules

### Alloy.Console

Console I/O operations using FNCS intrinsics.

```fsharp
module Console =
    /// Write a string to stdout
    val Write : string -> unit

    /// Write a string followed by newline to stdout
    val WriteLine : string -> unit

    /// Read a line from stdin
    val ReadLine : unit -> string
```

Implementation uses `Sys.write` and `Sys.read` intrinsics directly.

### Alloy.String

String operations on UTF-8 fat pointers.

```fsharp
module String =
    val length : string -> int
    val isEmpty : string -> bool
    val concat : string -> string -> string
    val slice : int -> int -> string -> voption<string>
```

### Alloy.Array

Array operations on fat pointers.

```fsharp
module Array =
    val length : array<'T> -> int
    val isEmpty : array<'T> -> bool
    val tryItem : int -> array<'T> -> voption<'T>
    val map : ('T -> 'U) -> array<'T> -> array<'U>
```

### Alloy.Option

Option operations with `voption` semantics.

```fsharp
module Option =
    val isSome : voption<'T> -> bool
    val isNone : voption<'T> -> bool
    val defaultValue : 'T -> voption<'T> -> 'T
    val map : ('T -> 'U) -> voption<'T> -> voption<'U>
```

## Using FNCS Intrinsics

Alloy implements I/O using FNCS intrinsics directly:

```fsharp
// Alloy/Primitives.fs
module Primitives =
    let inline writeStr (fd: int) (s: string) : int =
        Sys.write fd s.Pointer s.Length

    let inline writeStrOut s = writeStr 1 s
    let inline writeStrErr s = writeStr 2 s
```

This pattern:
- Uses `Sys.write` (Layer 1 intrinsic)
- No stub declarations with BCL dependencies
- Pure F# implementation

## Operators

### Arithmetic Operators

Standard arithmetic operators are defined in `Alloy.Operators`:

| Operator | Description |
|----------|-------------|
| `+`, `-`, `*`, `/` | Arithmetic |
| `%` | Modulo |
| `**` | Power |

### Comparison Operators

| Operator | Description |
|----------|-------------|
| `=`, `<>` | Equality |
| `<`, `>`, `<=`, `>=` | Ordering |

### Pipe Operators

| Operator | Description |
|----------|-------------|
| `\|>` | Forward pipe |
| `<\|` | Backward pipe |
| `>>` | Forward composition |
| `<<` | Backward composition |

## Differences from FSharp.Core

| Aspect | FSharp.Core | Alloy |
|--------|-------------|-------|
| String encoding | UTF-16 | UTF-8 |
| Option type | Reference, nullable | `voption`, non-nullable |
| Array header | Object header | Fat pointer |
| Memory management | GC | Deterministic |
| Platform operations | P/Invoke | FNCS intrinsics |

## SRTP Resolution

Alloy provides witnesses for statically resolved type parameters. The compiler resolves SRTP constraints against Alloy's type definitions.

```fsharp
// SRTP constraint resolved against Alloy.Operators
let inline double x = x + x
```

> **See**: [SRTP Resolution](inference-procedures.md) for resolution rules.
