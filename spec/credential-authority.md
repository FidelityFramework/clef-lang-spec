---
title: "Credential Authority"
weight: 466
category: Representation
status: draft
---

> **Status**: Draft — scoped outline (captured design space; not yet full normative prose). This chapter defines the **Credential Authority**, the on-device certificate-and-key authority layered *over* [Modular Blob Storage](modular-blob-storage.md). MBS is the sealed storage substrate; the Credential Authority is the capability that mints, derives, holds, and delegates credential material. The two are separate: MBS does not know about PKI, and the authority uses MBS for persistence. This outline fixes the structure and the decisions; the normative prose is deferred.

## 1. Purpose and Position

The device is not merely a store of keys. It is a **root-of-trust that mints and derives credential material on-device, holds the roots sequestered, and issues bounded derived credentials to trusted companion devices.** The Credential Authority is that capability. It sits above MBS (which persists the material as sealed blobs) and above the target's sealing/derivation engine (which holds the hardware-unique key and performs the cryptographic operations).

The founding constraint on this chapter, and on the whole framework, is that the cryptographic algorithms it invokes — ML-KEM, ML-DSA, and SLH-DSA — are implemented in pure functional Clef and are **verifiable**: the authority's operations rest on crypto whose constant-time, secret-independence, and allocation-free properties are checkable properties of the source carried on the [Program Semantic Graph](program-semantic-graph.md), not trust in an opaque binary. The authority does not weaken that; a derivation or an issuance is a composition of verifiable operations.

## 2. The PKI Object Model *(outline)*

The authority operates over **typed PKI objects**, not opaque blobs. MBS stores them sealed; the authority understands their structure.

- **Key objects** — a keypair (or a public key alone) of a named algorithm (ML-KEM / ML-DSA / SLH-DSA), with a usage class (signing, key-encapsulation, identity) and a position in a derivation hierarchy (§3).
- **Certificate objects** — X.509 certificates and chains (root → intermediate → leaf), with the standard fields the delegation model (§4) reads: subject binding, `notAfter` expiry, key-usage and extended-key-usage extensions.
- **Verifiable-credential objects** — W3C Verifiable Credentials and the Decentralized Identifiers (DIDs) they bind to, for the self-sovereign lane (§4).
- **Requests** — CSRs and issuance requests the authority evaluates against its policy.

*(Deferred: the exact Clef type definitions for each object, their sealed layout, and the catalog attributes each contributes.)*

## 3. HUK-Rooted Derivation — The Refinement *(outline)*

The core operation is **derivation**: the hardware-unique key (HUK) plus the algorithms bound to a key produce derived material that is a *refinement* of the root. The root secret never leaves the sequester; what is produced and delivered is derived, constrained material.

- **The root stays sequestered.** The HUK resides in the target's crypto engine and is never exported. Root and intermediate secrets live only inside the secure region; MBS holds them sealed at rest.
- **Derivation is one-way and hierarchical.** A derived key is refined from a parent (ultimately the HUK) by a derivation function; a child cannot reconstruct its parent. The hierarchy is root → intermediate → leaf, mirroring the PKI chain.
- **Derivation is a verifiable operation.** The derivation function is pure-functional Clef with the same proof obligations as the rest of the crypto; the "refinement" is a checkable transformation, not an opaque HSM call.
- **Device separation.** The authority (holding the roots) is physically separate from the mobile/desktop it serves. Roots never transit; only derived material crosses the boundary, under §4.

*(Deferred: the derivation-function specification, the hierarchy's representation, and how a derivation's proof obligations are stated and discharged.)*

## 4. Bounded Delegation — Two Lanes by Relying Party *(outline)*

The authority issues **derived credentials with a bounded lifetime and scope** to companion devices it has established as trusted. A derived credential is *not* a root and is *not* unbounded: it expires, it is scoped to a purpose, and it is bound to its target. The device issues whichever of two forms the relying party needs.

- **Lane A — attenuated capability token** (for Fidelity-native trusted devices). The derived key travels inside a capability token carrying its own caveats — expiry, scope/purpose, target-device binding — in the macaroon/biscuit tradition: the holder may only *attenuate* the token further, never broaden it, and the bounds are self-describing and offline-verifiable. This lane suits disconnected or contested environments where a relying party must verify without contacting the authority.
- **Lane B — short-lived X.509 certificate** (for standard PKI relying parties). The derived key is issued as a standard certificate with `notAfter` (expiry), key-usage/extended-key-usage extensions (scope), and subject binding (target device). This lane suits interoperation with existing PKI, directory services, and the classified-interop formats (NATO / STANAG / SKL) the console authority targets.

Both lanes express the same underlying bound — a lease, not the root — in the vocabulary the relying party speaks.

*(Deferred: the token caveat grammar, the attenuation semantics, the X.509 profile, and the trust-establishment handshake by which a companion device becomes a delegation target.)*

## 5. Lifecycle *(outline)*

The authority manages the full credential lifecycle, with signed provenance at each step:

entropy → generation (multi-person ceremony, on the console authority) → derivation/issuance → expiration/rotation → destruction.

*(Deferred: the ceremony model, rotation policy, revocation, destruction verification, and the signed-provenance/audit record format.)*

## 6. The Two-Device Realization *(outline)*

The framework is realized as a **device pair**, mapping onto the glass-box / sealed-box duality:

- **Console authority** (touchscreen, air-gapped) — holds the roots, runs the multi-person key ceremonies, issues into the classified-interop lane. The ceremony and root custody live here.
- **Wallet / holder** (pocket device) — holds delegated credentials, presents them to relying parties over its channels, operates in the self-sovereign (DID / Verifiable Credential) lane. The delegation target and everyday presentation live here.

The service side (the key service, and the secure-chat client that consumes credentials) are the software that relies on the credentials the pair mints and delegates.

*(Deferred: the console/wallet protocol, the channel bindings, and the service-side integration surface.)*

## 7. Relationship to Other Features

- [Modular Blob Storage](modular-blob-storage.md) — the sealed, fixed-slot substrate the authority persists every object in; the authority is a layer over it, not a mode of it.
- [Cryptographic Intrinsics](intrinsics-cryptography-bits.md) — the verifiable pure-functional ML-KEM / ML-DSA / SLH-DSA implementations the authority's derivation and issuance compose from; the authority adds no cryptography of its own, it orchestrates verifiable primitives.
- [Memory Regions](memory-regions.md) — roots and unsealed working material live only in the secure region; delegated material is what crosses the device boundary.
