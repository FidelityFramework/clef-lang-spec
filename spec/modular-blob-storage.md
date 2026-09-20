---
title: "Modular Blob Storage"
weight: 465
category: Representation
status: normative
---

> **Status**: Draft (captured design; not yet ratified). This chapter defines **Modular Blob Storage (MBS)**, the abstraction. A target's instantiation — for the EK-RA6M5, the *Credential Store* — is documented with that target.
>
> **Profile**: This chapter belongs to the **Freestanding Substrate** profile ([Conformance §7](conformance.md)). Its requirements bind an implementation that claims that profile; an implementation that delegates persistence to a hosting environment does not claim the profile, and these requirements do not apply to it.

## 1. Overview

Modular Blob Storage persists values whose lifetime exceeds a single program run. It is the durable rung of the [lifetime lattice](closure-representation.md): an MBS value survives power cycles, at a known location, under an access policy.

MBS is a persistence substrate. It supplies durable storage, handle addressing, a small index, and sealing. A filesystem adds four things over such a substrate — an open namespace, a path hierarchy, run-time storage growth, and a mutable metadata tree — and a target that needs them builds them above MBS. A freestanding, no-heap target uses the substrate directly. MBS is thus the first step toward a filesystem, stopping at the layer a constrained target can afford.

The name records the model. **Modular**: a record may be composed of sub-parts stored and addressed as a unit. **Blob**: a record is an opaque sealed object, written and read whole. **Storage**: the medium is persistent, reached through an API.

MBS realizes a *durability coeffect* — that a value survive power loss, that its write be atomic, and that it be sealed at rest — carried on the [Program Semantic Graph](program-semantic-graph.md) beside the region, width, and representation coeffects and committed at target-binding (§8).

## 2. The Stored Unit

A single MBS instance holds records of one shape, written `MBS<'Record>`. An instance is provisioned for that shape, so its slot sizes and index attributes are known ahead of time; this is the extent of the shape's role, and no type discipline operates over stored objects at run time.

A record is written and read whole. The target must implement that operation with a durable commit protocol: a completed write makes the new record retrievable, while an interrupted write leaves the previous committed state retrievable. Copy-on-write slots or another target-supported atomic mechanism can satisfy this requirement. The API's record granularity alone does not make flash writes atomic. A reader reaches a slot only through a handle the store issues on a completed write.

A record may be composed of sub-parts — for the Credential Store, an identity record of a signing keypair, a KEM keypair, a certificate chain, and attributes. The index addresses whole records; storage granularity underneath may be per-sub-part.

## 3. Addressing

The address of a record is an opaque, stable **handle**.

```clef
type Handle<'Record>    // opaque; issued by put, consumed by get; not caller-computable
```

`put` issues a handle on a completed write; `get` resolves it to a record. A handle is a token the store defines, so the address space is closed by construction.

A record may carry a human-readable **label** among its index attributes (§4). A label is queried by `find`; it is not an address, need not be unique, and may be absent. The handle remains the sole address.

## 4. The Index

MBS keeps one **index entry** per record, holding the handle and a fixed set of non-secret attributes — for the Credential Store, an optional label, a usage class, and a creation epoch. The attribute set is fixed at provision time and carries no secret material. The index makes a record selectable without unsealing it.

```clef
Mbs.list : MBS<'R> -> Handle<'R> array                        // every present handle
Mbs.find : MBS<'R> -> (IndexEntry -> bool) -> Handle<'R> array // scan the index by attribute
```

`find` is a linear scan of the fixed index against a caller predicate — bounded by the slot count, reading no blob. It selects records by label, usage, or age. Richer directory and metadata query belong to a filesystem layer above MBS.

## 5. Sealing

Every crossing of the store boundary passes through a **seal**: a record is sealed on `put` and unsealed on `get`, under a key rooted in the target's device-bound custody. The persisted blob is ciphertext; the plaintext record exists only transiently in a secure region after an unseal.

A target may place sealed blobs in bulk storage without confidentiality protection from the medium itself. The seal must authenticate the record and its binding to the store. The target must separately state its freshness and rollback guarantees. Valid older ciphertext remains replayable unless trusted state distinguishes it from the current record. Encryption also leaves deletion and availability concerns to the storage design.

```clef
Mbs.put   : MBS<'R> -> 'R -> Handle<'R>       // seal, write whole, return the handle
Mbs.get   : MBS<'R> -> Handle<'R> -> 'R       // read whole, unseal into a secure region
Mbs.evict : MBS<'R> -> Handle<'R> -> unit     // remove the blob and its index entry
```

### 5.1 Symmetric authenticated encryption

The seal SHALL use authenticated encryption with a 256-bit symmetric key. AES-256 is the baseline algorithm direction. Its quantum-security assessment depends on current cryptanalysis and attack-resource assumptions, rather than an unconditional conversion from key length to security bits. [NIST's PQC FAQ](https://csrc.nist.gov/projects/post-quantum-cryptography/faqs) supports continued use of AES-256 under current understanding.

Fidelity.Cryptography's [MBS sealing design](../../Fidelity.Cryptography/docs/mbs-sealing.md) proposes AES-256-GCM, conditional on a durable nonce-allocation and recovery contract. The selected suite fixes nonce and tag lengths, associated-data encoding and usage limits. Plaintext is released to the caller only after authentication succeeds.

Post-quantum KEMs and signatures serve key establishment and credential authentication. They complement the symmetric seal even when an accelerator supports them. The [Credential Authority](credential-authority.md) owns their use in issuance and delegation.

### 5.2 Custody and providers

The root of sealing-key custody is device-bound and non-exportable. The selected provider may operate through a protected key handle, or the target may authorize a documented derivation route to a software-accessible working key. A software cipher cannot directly consume a hardware key that the CPU cannot read. Export and derivation permissions must therefore be stated separately from AES availability.

Fidelity.Cryptography owns the sealing operation contract and first-party software direction. Fidelity.Platform supplies hardware capabilities and bindings. Hardware acceleration is an admitted implementation choice with explicit device assumptions, conformance evidence and resource constraints. Software implementations carry their own arithmetic, secret-handling and lowering obligations. The target selects a provider satisfying the same suite and custody policy, as described in the [provider contract](../../Fidelity.Cryptography/docs/platform-providers.md).

MBS owns persistent nonce reservations and record commits. Provider substitution preserves the record format and does not reset nonce state. Unsupported custody or operations produce a capability failure rather than a silent key export or suite downgrade.

## 6. Allocation

An MBS instance is a fixed set of slots, sized at provision time. Its footprint is a program-lifetime fact, placed in static storage (the program-lifetime point of the [lifetime lattice](closure-representation.md#33-escape-analysis)); the working set of an unsealed record is placed in a bounded secure region. Neither the store nor the seal path allocates on a heap, which admits MBS on a freestanding target.

`put` into a store with no free slot of the record's class is a reported error, in the manner of an unobservable range under [width inference](width-inference.md).

## 7. The Durability Coeffect

MBS is the storage-facing form of a **durability coeffect** on the PSG, with three components:

- **survives-power-loss** — the storage outlives the process; the value is placed in non-volatile storage.
- **atomic-write** — the value is written whole; no partial-write state is observable.
- **sealed** — the value crosses the medium as authenticated ciphertext under the declared key-custody policy.

Like the region, width, and representation coeffects, the durability coeffect is analyzed and carried in the middle end and committed by the target pathway, which selects the non-volatile medium, the atomic-write primitive, and the sealing capability. A target lacking non-volatile storage or a sealing capability is a capability failure at binding.

## 8. Relationship to Other Features

- [Memory Regions](memory-regions.md) — the sealed medium, the index, and the secure working region are distinct regions with distinct access kinds; MBS storage is placed by region and lifetime.
- [Closure Representation](closure-representation.md) — the lifetime lattice's program-lifetime point places the store's own structure; MBS extends persistence one rung further, to lifetimes that outlive the process.
- [Credential Authority](credential-authority.md) — the layer that mints, derives, and delegates the credentials MBS persists.
- [Fidelity.Cryptography](../../Fidelity.Cryptography/README.md) supplies the planned sealing operations, provider contracts and verification requirements.
- [Namespace Storage](namespace-storage.md) — the layer above: names, history, and growth as an append-only ledger checkpointed into sealed segments, all stored as MBS records. Its §7 carries the server-scale generalization; MBS is the closed, fixed-slot floor beneath both.

## 9. Normative Requirements

1. **Handle addressing.** Records SHALL be addressed by opaque handles the store issues; an MBS instance SHALL NOT present a path namespace.
2. **Whole-record access.** A record SHALL be written and read whole; MBS SHALL NOT present partial (seek/append) access.
3. **Atomic write.** A `put` SHALL either complete and make the record retrievable by its handle or leave the slot unchanged; no partial-write state SHALL be observable through a handle.
4. **Sealed at rest.** A persisted record SHALL use authenticated encryption with a 256-bit symmetric key rooted in device-bound custody. Plaintext SHALL remain in authorized secure working regions and SHALL be exposed to an unseal caller only after authentication succeeds. The provider SHALL preserve the selected suite and custody policy.
5. **Fixed slots.** An MBS instance's slot set SHALL be fixed at provision time; the store SHALL NOT allocate on a heap; a full store SHALL be a reported error.
6. **Opaque handles.** A handle SHALL be a store-issued token, not a caller-computable name, path, or index.
7. **Secret-free index.** The index SHALL carry only non-secret attributes; scanning it SHALL require no unseal.
8. **Durability coeffect.** Durability, atomicity, and sealing SHALL be carried as a coeffect on the PSG and committed at target-binding; a target lacking non-volatile storage or a sealing capability SHALL be a capability failure.
9. **Nonce and recovery state.** The implementation SHALL preserve the selected suite's nonce requirements through retries, power loss and key rotation. Record publication SHALL use a target-backed durable commit protocol.
10. **Freshness.** Each target SHALL state its rollback and freshness guarantees, including the trusted state and recovery policy on which they depend. Authentication alone SHALL NOT be represented as rollback resistance.
