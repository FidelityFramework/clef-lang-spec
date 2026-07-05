---
title: "Modular Blob Storage"
weight: 465
category: Representation
status: draft
---

> **Status**: Draft (captured design; not yet ratified). This chapter defines **Modular Blob Storage (MBS)**, the abstraction. A target's instantiation — for the EK-RA6M5, the *Credential Store* — is documented with that target.

## 1. Overview

Modular Blob Storage persists values whose lifetime exceeds a single program run. It is the durable rung of the [lifetime lattice](closure-representation.md): an MBS value survives power cycles, at a known location, under an access policy.

MBS is a persistence substrate. It supplies durable storage, handle addressing, a small index, and sealing. A filesystem adds four things over such a substrate — an open namespace, a path hierarchy, run-time storage growth, and a mutable metadata tree — and a target that needs them builds them above MBS. A freestanding, no-heap unikernel uses the substrate directly. MBS is thus the first step toward a filesystem, stopping at the layer a constrained target can afford.

The name records the model. **Modular**: a record may be composed of sub-parts stored and addressed as a unit. **Blob**: a record is an opaque sealed object, written and read whole. **Storage**: the medium is persistent, reached through an API.

MBS realizes a *durability coeffect* — that a value survive power loss, that its write be atomic, and that it be sealed at rest — carried on the [Program Semantic Graph](program-semantic-graph.md) beside the region, width, and representation coeffects and committed at target-binding (§8).

## 2. The Stored Unit

A single MBS instance holds records of one shape, written `MBS<'Record>`. An instance is provisioned for that shape, so its slot sizes and index attributes are known ahead of time; this is the extent of the shape's role, and no type discipline operates over stored objects at run time.

A record is written and read whole. This gives crash-consistency without a journal: a slot write completes and the record becomes retrievable, or it does not and the slot retains its prior contents. A reader reaches a slot only through a handle the store issues on a completed write, so no partial-write state is observable.

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

The medium therefore need not be access-controlled. A sealed blob is inert without its key, and the key is held in the target's sequester, so a target may place sealed blobs in bulk storage that carries no security attribution of its own.

```clef
Mbs.put   : MBS<'R> -> 'R -> Handle<'R>       // seal, write whole, return the handle
Mbs.get   : MBS<'R> -> Handle<'R> -> 'R       // read whole, unseal into a secure region
Mbs.evict : MBS<'R> -> Handle<'R> -> unit     // remove the blob and its index entry
```

### 5.1 The seal is a symmetric primitive

The seal is data-at-rest encryption under a device-held key: a symmetric cipher. By primitive class it faces only Grover's quadratic search speedup, which halves the effective security parameter; a 256-bit key retains 128 effective bits, at the post-quantum floor. The seal is post-quantum secure by parameter, not by algorithm, and SHALL use a 256-bit-class cipher — a 128-bit key falls to 64 bits under Grover.

The post-quantum algorithms — lattice key-encapsulation and the signature schemes — live in the credentials the seal protects and in the delegation that issues them ([Credential Authority](credential-authority.md)), because key-exchange and signatures are the functions Shor's algorithm breaks. The seal and the credentials are distinct primitive classes with distinct quantum exposure.

### 5.2 The sealing capability is a per-target decision

The seal has two parts. **Key custody** is device-bound and non-extractable, which binds a sealed credential to one device; a target's crypto hardware may hold that key. The **sealing operation** is a symmetric cipher, expressible as verifiable code whose constant-time and secret-independence properties are checkable on the PSG, or delegated to a hardware crypto engine at the cost of moving it outside that verifiable boundary.

This chapter fixes only the seal's primitive class (256-bit symmetric) and its key custody (device-bound). Whether a target seals in verifiable code over a hardware-held key or in a hardware engine is a decision of record for that target, stated with its verifiability cost. Where the seal is verifiable code, its proof obligations ride the PSG through to the seal as they do for the rest of the program.

## 6. Allocation

An MBS instance is a fixed set of slots, sized at provision time. Its footprint is a program-lifetime fact, placed in static storage (the program-lifetime point of the [lifetime lattice](closure-representation.md#33-escape-analysis)); the working set of an unsealed record is placed in a bounded secure region. Neither the store nor the seal path allocates on a heap, which admits MBS on a freestanding target.

`put` into a store with no free slot of the record's class is a reported error, in the manner of an unobservable range under [width inference](width-inference.md).

## 7. The Durability Coeffect

MBS is the storage-facing form of a **durability coeffect** on the PSG, with three components:

- **survives-power-loss** — the storage outlives the process; the value is placed in non-volatile storage.
- **atomic-write** — the value is written whole; no partial-write state is observable.
- **sealed** — the value crosses the medium as ciphertext.

Like the region, width, and representation coeffects, the durability coeffect is analyzed and carried in the middle end and committed by the target's backend leg, which selects the non-volatile medium, the atomic-write primitive, and the sealing capability. A target lacking non-volatile storage or a sealing capability is a capability failure at binding.

## 8. Relationship to Other Features

- [Memory Regions](memory-regions.md) — the sealed medium, the index, and the secure working region are distinct regions with distinct access kinds; MBS storage is placed by region and lifetime.
- [Closure Representation](closure-representation.md) — the lifetime lattice's program-lifetime point places the store's own structure; MBS extends persistence one rung further, to lifetimes that outlive the process.
- [Credential Authority](credential-authority.md) — the layer that mints, derives, and delegates the credentials MBS persists.
- **Server-scale generalization** — an open-namespace document store is the same durability coeffect over an open set of objects, with the filesystem layers built up. It is future work; MBS is the closed, fixed-slot floor beneath it.

## 9. Normative Requirements

1. **Handle addressing.** Records SHALL be addressed by opaque handles the store issues; an MBS instance SHALL NOT present a path namespace.
2. **Whole-record access.** A record SHALL be written and read whole; MBS SHALL NOT present partial (seek/append) access.
3. **Atomic write.** A `put` SHALL either complete and make the record retrievable by its handle or leave the slot unchanged; no partial-write state SHALL be observable through a handle.
4. **Sealed at rest.** A persisted record SHALL be sealed under a 256-bit-class symmetric cipher keyed by device-bound custody; the plaintext SHALL exist only in a secure region after an unseal. A hardware sealing engine SHALL NOT be assumed; the sealing operation's realization is a per-target decision.
5. **Fixed slots.** An MBS instance's slot set SHALL be fixed at provision time; the store SHALL NOT allocate on a heap; a full store SHALL be a reported error.
6. **Opaque handles.** A handle SHALL be a store-issued token, not a caller-computable name, path, or index.
7. **Secret-free index.** The index SHALL carry only non-secret attributes; scanning it SHALL require no unseal.
8. **Durability coeffect.** Durability, atomicity, and sealing SHALL be carried as a coeffect on the PSG and committed at target-binding; a target lacking non-volatile storage or a sealing capability SHALL be a capability failure.
