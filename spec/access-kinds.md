---
title: "Access Kinds"
weight: 270
---

Access kinds define the permitted operations on pointers and memory regions. They are enforced at compile time and affect code generation.

## Overview

Every pointer in Clef carries an access kind that specifies what operations are legal:

```fsharp
type Ptr<'T, 'Region, 'Access>
//                     ^^^^^^^ Access kind
```

Access kinds prevent:
- Writing to read-only memory (flash, input registers)
- Reading from write-only memory (output registers)
- Accidental modification of constants

## Access Kind Types

| Kind | Read | Write | CMSIS Equivalent | Use Case |
|------|------|-------|------------------|----------|
| `ReadOnly` | Yes | No | `__I` | Input registers, flash, constants |
| `WriteOnly` | No | Yes | `__O` | Output registers, write-only buffers |
| `ReadWrite` | Yes | Yes | `__IO` | General mutable access |

### ReadOnly

Read-only pointers permit load operations only.

```fsharp
let flashData : Ptr<byte, Flash, ReadOnly> = ...
let value = Ptr.read flashData      // OK
Ptr.write flashData 0uy             // ERROR: Cannot write to ReadOnly
```

**Code Generation**: No store instructions generated for ReadOnly pointers.

### WriteOnly

Write-only pointers permit store operations only.

```fsharp
let outputReg : Ptr<uint32, Peripheral, WriteOnly> = ...
Ptr.write outputReg 0x1234u         // OK
let value = Ptr.read outputReg      // ERROR: Cannot read from WriteOnly
```

**Code Generation**: No load instructions generated for WriteOnly pointers.

### ReadWrite

Read-write pointers permit both load and store operations.

```fsharp
let gpioReg : Ptr<uint32, Peripheral, ReadWrite> = ...
let current = Ptr.read gpioReg      // OK
Ptr.write gpioReg (current ||| 0x1u) // OK
```

## Compile-Time Enforcement

Access kind violations are type errors:

```fsharp
// ERROR: Cannot write to ReadOnly pointer
let writeFlash (p: Ptr<byte, Flash, ReadOnly>) =
    Ptr.write p 0uy  // Compile error

// ERROR: Cannot read from WriteOnly pointer
let readOutput (p: Ptr<uint32, Peripheral, WriteOnly>) =
    Ptr.read p  // Compile error
```

## Access Kind Coercion

More restrictive access kinds can be coerced to less restrictive:

| From | To | Allowed |
|------|-----|---------|
| `ReadWrite` | `ReadOnly` | Yes (drop write) |
| `ReadWrite` | `WriteOnly` | Yes (drop read) |
| `ReadOnly` | `ReadWrite` | No |
| `WriteOnly` | `ReadWrite` | No |
| `ReadOnly` | `WriteOnly` | No |
| `WriteOnly` | `ReadOnly` | No |

```fsharp
let rw : Ptr<int, Stack, ReadWrite> = ...
let ro : Ptr<int, Stack, ReadOnly> = Ptr.asReadOnly rw  // OK
let wo : Ptr<int, Stack, WriteOnly> = Ptr.asWriteOnly rw // OK
```

## Hardware Register Patterns

Access kinds enable type-safe hardware register access:

```fsharp
[<PeripheralDescriptor("GPIO", 0x48000000UL)>]
type GPIO_TypeDef = {
    [<Register("MODER", 0x00u, "rw")>]
    MODER: Ptr<uint32, Peripheral, ReadWrite>
    
    [<Register("IDR", 0x10u, "r")>]
    IDR: Ptr<uint32, Peripheral, ReadOnly>
    
    [<Register("ODR", 0x14u, "rw")>]
    ODR: Ptr<uint32, Peripheral, ReadWrite>
    
    [<Register("BSRR", 0x18u, "w")>]
    BSRR: Ptr<uint32, Peripheral, WriteOnly>
}
```

## Grammar

```fsgrammar
access-kind :=
    ReadOnly
    WriteOnly
    ReadWrite

ptr-type := Ptr < type , region-type , access-kind >
```

## Diagnostics

| Code | Message |
|------|---------|
| CCS8020 | Cannot write to ReadOnly pointer |
| CCS8021 | Cannot read from WriteOnly pointer |
| CCS8022 | Access kind mismatch in assignment |
