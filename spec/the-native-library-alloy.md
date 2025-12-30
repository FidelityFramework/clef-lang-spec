# The Native Library Alloy

Alloy is the native standard library for F# Native compilation. It provides BCL-sympathetic APIs that compile to native code with deterministic memory management.

> **Note**: This chapter specifies the Alloy library interface. For implementation details, see the [Alloy repository](https://github.com/speakez-codespace/Alloy).

## Overview

Where .NET F# programs reference `FSharp.Core.dll` and `mscorlib.dll`, F# Native programs reference **Alloy**. Alloy provides:

- Familiar API surface (Console, String, Array, etc.)
- Native type implementations (fat pointers, stack allocation)
- Platform bindings for system calls
- No garbage collector dependency

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

Console I/O operations using platform bindings.

```fsharp
module Console =
    val Write : string -> unit
    val WriteLine : string -> unit
    val ReadLine : unit -> string
```

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

## Platform Bindings

Alloy defines platform binding points that the compiler (Alex) implements for each target:

```fsharp
module Platform.Bindings =
    val writeBytes : int -> nativeptr<byte> -> int -> int
    val readBytes : int -> nativeptr<byte> -> int -> int
    val getCurrentTicks : unit -> int64
    val sleep : int -> unit
```

> **See**: [Platform Bindings](platform-bindings.md) for binding semantics.

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
| Platform bindings | P/Invoke | Compiler-provided |

## SRTP Resolution

Alloy provides witnesses for statically resolved type parameters. The compiler resolves SRTP constraints against Alloy's type definitions.

```fsharp
// SRTP constraint resolved against Alloy.Operators
let inline double x = x + x
```

> **See**: [SRTP Resolution](inference-procedures.md) for resolution rules.
