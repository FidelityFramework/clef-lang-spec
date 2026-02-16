---
title: "Front Matter"
weight: 1
---

This is the normative language specification for **Clef**, a natively compiled language in the ML family derived from F#.

Clef preserves F# syntax and type-checking behavior while defining explicit native semantics for type layouts, memory ownership, lifetime verification, and deterministic resource management. This specification defines those native semantics.

The starting point for this specification was the [F# 4.1 Language Specification](https://fsharp.org/specs/language-spec/4.1/FSharpSpec-4.1-latest.pdf). Sections that assumed .NET runtime behavior have been revised to define explicit native semantics. Content on .NET interop, reflection, and runtime type discovery has been removed. New chapters cover ownership, borrowing, memory regions, access kinds, and lifetime constraints.

## Notices

_The original F# spec was authored by Don Syme, with assistance from Anar Alimov, Keith Battocchi, Jomo Fisher, Michael Hale, Jack Hu, Luke Hoban, Tao Liu, Dmitry Lomov, James Margetson, Brian McNamara, Joe Pamer, Penny Orwick, Daniel Quirk, Kevin Ransom, Chris Smith, Matteo Taveggia, Donna Malayeri, Wonseok Chae, Uladzimir Matsveyeu, Lincoln Atkinson, and others._

_The Clef language specification is developed by [SpeakEZ Technologies](https://speakez.tech)._

_© 2024-2026 SpeakEZ Technologies. Made available under the [MIT License](../LICENSE)._
