---
title: "Front Matter"
weight: 1
---

This is the normative language specification for **Clef**, a concurrent, natively compiled language.

Clef uses ML-family syntax to express operations across CPUs, GPUs, NPUs (Neural Processing Units), FPGAs (Field-Programmable Gate Arrays), and other compute accelerators. The language is built around concurrency as a first-class concern: the actor model, [incremental computation](incremental-computation.md) via `Incremental<'T>`, delimited continuations, and interaction nets are central to both the language's semantics and its compilation model. Where F# treated its `MailboxProcessor` as a library convenience within the .NET ecosystem, Clef elevates message-passing concurrency to a foundational language primitive.

Clef's principal syntax is rooted in F#, but with significant divergences. Constructs that existed solely for interoperation with the .NET Common Language Runtime (CLR), the Base Class Library (BCL), and the managed runtime have been removed. The type system draws on F\* (FStar) for proof-carrying compilation and design-time verification. The [Native Type Universe (NTU)](native-type-universe.md) was motivated by F\*'s distinction between type width and type identity. The compilation architecture follows the nanopass methodology pioneered in Scheme compiler research, structuring the compiler as a sequence of small, composable transformations. These influences (F#, F\*, Scheme, and hardware-oriented concurrency) are fused into a single coherent language design.

This specification defines Clef's native semantics: type layouts, memory ownership, lifetime verification, deterministic resource management, and concurrency primitives. The [F# Language Specification](https://github.com/fsharp/fslang-spec) served as a starting point. Sections that assumed .NET runtime behavior have been rewritten. Content on .NET interop, reflection, and runtime type discovery has been removed. New chapters cover the actor model, incremental computation, [memory regions](memory-regions.md), [access kinds](access-kinds.md), [platform bindings](platform-bindings.md), and proof-aware compilation.

## Notices

_The original F# spec was authored by Don Syme, with assistance from Anar Alimov, Keith Battocchi, Jomo Fisher, Michael Hale, Jack Hu, Luke Hoban, Tao Liu, Dmitry Lomov, James Margetson, Brian McNamara, Joe Pamer, Penny Orwick, Daniel Quirk, Kevin Ransom, Chris Smith, Matteo Taveggia, Donna Malayeri, Wonseok Chae, Uladzimir Matsveyeu, Lincoln Atkinson, and others._

_The Clef language specification is developed by [SpeakEZ Technologies](https://speakez.tech)._

_© 2024-2026 SpeakEZ Technologies. Made available under the [MIT License](../LICENSE)._
