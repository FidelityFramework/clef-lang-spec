---
title: "Normative References"
weight: 4
category: Process
status: normative
---

> **The external standards this specification binds to.** A conforming implementation must satisfy these references where this specification invokes them.

The following documents are referred to in the text in such a way that some or all of their content constitutes requirements of this specification. For dated references, only the cited edition applies; the edition is named because the requirement depends on it. For undated references, the latest edition of the referenced document applies.

These are **normative** references: where a chapter of this specification requires conformance to one of them, an implementation SHALL satisfy the referenced standard for that requirement. Academic works, prior-art citations, and lineage sources that *explain* a design without imposing a requirement are **not** listed here; they appear as informative References within the chapters that cite them.

## Numeric representation and arithmetic

- **IEEE 754-2019** — *IEEE Standard for Floating-Point Arithmetic.* Governs the IEEE-754 representations selectable by [Numeric Selection](numeric-selection.md), their error model, and the rounding-direction attributes required by [Rounding](rounding.md).
- **IEEE 1788-2015** — *IEEE Standard for Interval Arithmetic.* Governs the soundness obligation for interval enclosures (the outward-rounding requirement) referenced by [Rounding](rounding.md) and the real interval domain of [Numeric Selection §9.1](numeric-selection.md).
- **Standard for Posit Arithmetic (2022)** — Governs the full-gamut posit representations and their quire (the `n²/2` accumulator width and its exact-accumulation semantics) referenced by [Numeric Selection](numeric-selection.md) and [Rounding](rounding.md).
- **Jonnalagadda, Thotli & Gustafson (2026), arXiv:2603.01615, *Closing the Gap Between Float and Posit Hardware Efficiency*** — Governs the bounded-regime b-posit representation and its fixed 800-bit universal quire referenced by [Numeric Selection §10.2](numeric-selection.md) and [Rounding §4](rounding.md).

## Cryptographic operations

- **FIPS 180-4** — *Secure Hash Standard.* Governs the SHA-1 digest required of the `Cryptography.sha1` intrinsic ([Cryptography and Bits Intrinsic Modules](intrinsics-cryptography-bits.md)).
- **RFC 4648** — *The Base16, Base32, and Base64 Data Encodings.* Governs the Base64 encoding and decoding required of the `Cryptography.base64Encode` / `Cryptography.base64Decode` intrinsics.
- **RFC 6455** — *The WebSocket Protocol.* The protocol whose handshake motivates the SHA-1 and Base64 intrinsics and the byte-order operations of the Bits module.

## Text

- **The Unicode Standard** — Governs character classification, identifiers, and string semantics throughout [Lexical Analysis](lexical-analysis.md) and the text-handling chapters. The undated reference applies: the latest edition of The Unicode Standard governs unless a specific chapter cites a fixed version.

## Note on the relationship to this specification

Where this specification restates a figure or rule from a normative reference (for example, the posit quire width, or an IEEE-754 error bound), the restatement is for the reader's convenience and the normative reference governs in case of conflict. An implementation that satisfies this specification's requirements while violating a normative reference it invokes is **not** conforming.
