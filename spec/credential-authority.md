---
title: "Credential Authority"
weight: 466
category: Representation
status: normative
---

This chapter defines the **Credential Authority**, the on-device certificate-and-key authority layered over [Modular Blob Storage](modular-blob-storage.md). MBS persists credential material as sealed blobs; the authority mints, derives, holds, and delegates it.

> **Profile**: This chapter belongs to the **Freestanding Substrate** profile ([Conformance §7](conformance.md)); its requirements bind an implementation that claims that profile.

## 1. Purpose and Position

The Credential Authority is a device-resident root of trust. It mints and derives credential material on-device, holds the roots sequestered, and issues bounded derived credentials to trusted companion devices. It sits over [MBS](modular-blob-storage.md), which persists the material, and over the target's key custody, which holds the device-bound key.

The authority consumes [Fidelity.Cryptography](../../Fidelity.Cryptography/README.md) for algorithm and key-operation contracts. Hardware providers bind through Fidelity.Platform. Algorithm conformance, secret independence and resource behavior require evidence for the selected implementation and target.

## 2. The PKI Object Model

The authority operates over structured PKI objects, which MBS stores sealed:

- **Key objects** — a keypair or public key with an explicit algorithm, parameter set and encoding revision, with a usage class (signing, key-encapsulation, identity) and an authorized custody policy (§3).
- **Certificate objects** — X.509 certificates and chains (root → intermediate → leaf), carrying the fields the delegation model reads (§4): subject binding, `notAfter`, key-usage and extended-key-usage.
- **Verifiable-credential objects** — W3C Verifiable Credentials and the Decentralized Identifiers (DIDs) they bind, for the self-sovereign lane (§4).
- **Requests** — CSRs and issuance requests the authority evaluates against its policy.

## 3. HUK-Rooted Derivation

The hardware-unique key (HUK) remains in device custody. Its supported operations and the application's policy determine whether a derived working key may enter software memory or remain represented by an opaque handle. A certificate hierarchy establishes issuance authority. Secret-key derivation requires a separately specified KDF and purpose context, and each algorithm retains its own key-generation procedure.

- **Root custody.** The HUK remains in non-exportable device custody. Other private keys use the declared protected-handle or secure-region policy, with MBS sealing where persistence is required.
- **Purpose-bound derivation.** An approved derivation profile binds child material to its parent and intended use. The issuance hierarchy authenticates subordinate credentials independently of whether their private keys were derived or generated separately.
- **Derivation evidence.** First-party Clef derivation and a hardware derivation service have separate evidence requirements. The selected path must preserve the root's custody policy and document any software-accessible child material.
- **Device separation.** The authority holding the roots is physically separate from the mobile or desktop it serves. Only derived material crosses to that device, under §4.

## 4. Bounded Delegation — Two Lanes by Relying Party

The authority issues derived credentials to companion devices it has established as trusted. Each carries a bounded lifetime and scope: it expires, it is scoped to a purpose, and it is bound to its target — a lease over the root, in the vocabulary the relying party speaks. The relying party determines the form.

- **Lane A — attenuated capability token** (Fidelity-native devices). The derived key travels in a token whose caveats — expiry, scope, target binding — are self-describing and offline-verifiable, in the macaroon/biscuit tradition: a holder may attenuate the token further but not broaden it. This lane serves disconnected and contested environments, where a relying party verifies without reaching the authority.
- **Lane B — short-lived X.509 certificate** (standard PKI relying parties). The derived key is issued as a certificate whose `notAfter`, key-usage/extended-key-usage, and subject fields carry the same bound. This lane serves existing PKI, directory services, and the classified-interop formats (NATO / STANAG / SKL) the console authority targets.

## 5. Lifecycle

The authority manages the credential lifecycle with signed provenance at each step:

entropy → generation (multi-person ceremony on the console authority) → derivation/issuance → expiration/rotation → destruction.

## 6. The Two-Device Realization

The framework is realized as a device pair, along the glass-box / sealed-box line:

- **Console authority** (touchscreen) — holds the roots, runs the multi-person ceremonies, and issues into the classified-interop lane. An air-gapped custody profile uses a separate gateway for digital ceremony transport. A console that directly terminates an ephemeral network is temporarily connected and uses the corresponding networked threat model.
- **Wallet / holder** (pocket device) — holds delegated credentials, presents them over its channels, and operates in the self-sovereign (DID / Verifiable Credential) lane.

The key service and the secure-chat client are the software that consumes the credentials the pair mints and delegates.

## 7. Relationship to Other Features

- [Modular Blob Storage](modular-blob-storage.md) — the sealed, fixed-slot substrate the authority persists every object in.
- [Fidelity.Cryptography algorithms](../../Fidelity.Cryptography/docs/algorithms.md) and [verification requirements](../../Fidelity.Cryptography/docs/verification.md) define the algorithm contracts and required evidence.
- [WireGuard ceremony channels](../../Fidelity.Cryptography/docs/wireguard-hybrid.md) define short-lived delegation transport and persistent overlay responsibilities.
- [Merkle Tree Certificates](../../Fidelity.Cryptography/docs/merkle-tree-certificates.md) define the library boundary for proof operations and offline trust updates.
- [Memory Regions](memory-regions.md) — roots and unsealed material live only in the secure region; derived material is what crosses the device boundary.
