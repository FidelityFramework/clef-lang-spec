---
title: "Credential Authority"
weight: 466
category: Representation
status: draft
---

> **Status**: Draft — scoped outline (captured design; not yet full normative prose). This chapter defines the **Credential Authority**, the on-device certificate-and-key authority layered over [Modular Blob Storage](modular-blob-storage.md). MBS persists credential material as sealed blobs; the authority mints, derives, holds, and delegates it. This outline fixes the structure and the decisions; the normative prose is deferred.

## 1. Purpose and Position

The Credential Authority is a device-resident root of trust. It mints and derives credential material on-device, holds the roots sequestered, and issues bounded derived credentials to trusted companion devices. It sits over [MBS](modular-blob-storage.md), which persists the material, and over the target's key custody, which holds the device-bound key.

The algorithms it invokes — ML-KEM, ML-DSA, SLH-DSA — are pure-functional Clef, and their constant-time, secret-independence, and allocation-free properties are checkable on the [Program Semantic Graph](program-semantic-graph.md). Each derivation and issuance is a composition of these verifiable operations.

## 2. The PKI Object Model *(outline)*

The authority operates over structured PKI objects, which MBS stores sealed:

- **Key objects** — a keypair or public key of a named algorithm (ML-KEM / ML-DSA / SLH-DSA), with a usage class (signing, key-encapsulation, identity) and a position in the derivation hierarchy (§3).
- **Certificate objects** — X.509 certificates and chains (root → intermediate → leaf), carrying the fields the delegation model reads (§4): subject binding, `notAfter`, key-usage and extended-key-usage.
- **Verifiable-credential objects** — W3C Verifiable Credentials and the Decentralized Identifiers (DIDs) they bind, for the self-sovereign lane (§4).
- **Requests** — CSRs and issuance requests the authority evaluates against its policy.

*(Deferred: the Clef type definitions, sealed layout, and index attributes for each object.)*

## 3. HUK-Rooted Derivation — The Refinement *(outline)*

Derivation is the core operation: the hardware-unique key (HUK), with the algorithms bound to a key, produces derived material that refines the root. What crosses any boundary is the derived material; the root secret stays in the sequester.

- **Root custody.** The HUK is held in the target's key custody and never exported. Root and intermediate secrets live only in the secure region; MBS holds them sealed at rest.
- **One-way hierarchy.** A derived key is refined from its parent — ultimately the HUK — by a derivation function a child cannot invert. The hierarchy runs root → intermediate → leaf, matching the PKI chain.
- **Verifiable derivation.** The derivation function is pure-functional Clef under the same proof obligations as the rest of the crypto, so the refinement is a checkable transformation.
- **Device separation.** The authority holding the roots is physically separate from the mobile or desktop it serves. Only derived material crosses to that device, under §4.

*(Deferred: the derivation-function specification, the hierarchy's representation, and how a derivation's proof obligations are discharged.)*

## 4. Bounded Delegation — Two Lanes by Relying Party *(outline)*

The authority issues derived credentials to companion devices it has established as trusted. Each carries a bounded lifetime and scope: it expires, it is scoped to a purpose, and it is bound to its target — a lease over the root, in the vocabulary the relying party speaks. The relying party determines the form.

- **Lane A — attenuated capability token** (Fidelity-native devices). The derived key travels in a token whose caveats — expiry, scope, target binding — are self-describing and offline-verifiable, in the macaroon/biscuit tradition: a holder may attenuate the token further but not broaden it. This lane serves disconnected and contested environments, where a relying party verifies without reaching the authority.
- **Lane B — short-lived X.509 certificate** (standard PKI relying parties). The derived key is issued as a certificate whose `notAfter`, key-usage/extended-key-usage, and subject fields carry the same bound. This lane serves existing PKI, directory services, and the classified-interop formats (NATO / STANAG / SKL) the console authority targets.

*(Deferred: the token caveat grammar, the attenuation semantics, the X.509 profile, and the trust-establishment handshake by which a device becomes a delegation target.)*

## 5. Lifecycle *(outline)*

The authority manages the credential lifecycle with signed provenance at each step:

entropy → generation (multi-person ceremony on the console authority) → derivation/issuance → expiration/rotation → destruction.

*(Deferred: the ceremony model, rotation policy, revocation, destruction verification, and the signed-provenance record format.)*

## 6. The Two-Device Realization *(outline)*

The framework is realized as a device pair, along the glass-box / sealed-box line:

- **Console authority** (touchscreen, air-gapped) — holds the roots, runs the multi-person ceremonies, and issues into the classified-interop lane.
- **Wallet / holder** (pocket device) — holds delegated credentials, presents them over its channels, and operates in the self-sovereign (DID / Verifiable Credential) lane.

The key service and the secure-chat client are the software that consumes the credentials the pair mints and delegates.

*(Deferred: the console/wallet protocol, the channel bindings, and the service-side integration surface.)*

## 7. Relationship to Other Features

- [Modular Blob Storage](modular-blob-storage.md) — the sealed, fixed-slot substrate the authority persists every object in.
- [Cryptographic Intrinsics](intrinsics-cryptography-bits.md) — the verifiable ML-KEM / ML-DSA / SLH-DSA implementations the authority's derivation and issuance compose from.
- [Memory Regions](memory-regions.md) — roots and unsealed material live only in the secure region; derived material is what crosses the device boundary.
