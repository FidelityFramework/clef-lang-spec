---
title: "Modular Blob Storage"
weight: 465
category: Representation
status: draft
---

> **Status**: Draft (captured design; not yet ratified). This chapter defines **Modular Blob Storage (MBS)**, the abstraction; a target's concrete instantiation (for the EK-RA6M5, the *Credential Store*) is documented with that target.

## 1. Overview

Modular Blob Storage is the persistence model for values whose lifetime is longer than a single program run. It is the durable rung of the [lifetime lattice](closure-representation.md): a value in MBS survives power cycles, in a known location, under an access policy. MBS is **not a filesystem** and does not present one. It has no path namespace, no directory hierarchy, no byte-stream I/O, and no mutable-metadata tree. It presents a fixed set of typed slots holding sealed blobs, addressed by opaque handles, organized by a queryable typed catalog.

The name states the model. **Modular**: the store holds records composed of a fixed schema of slots, not an open set of arbitrary objects. **Blob**: the stored unit is an opaque sealed object, written whole or not at all. **Storage**: the medium is persistent, and its organization is an API, not a namespace.

MBS realizes the *durability coeffect* — the requirement that a value survive power loss, that a write be atomic, and that a read come from sealed storage — carried on the [Program Semantic Graph](program-semantic-graph.md) beside the region, width, and representation coeffects, and committed at target-binding like them.

## 2. What MBS Is Not

MBS is defined as much by its exclusions as its inclusions, because the exclusions are properties, not gaps.

- **No path namespace.** There is no `/certs/alice/signing.key`. Items are addressed by an opaque handle (§4). The absence of a hierarchical, mutable path namespace is a security and bounded-ness property: a closed address space cannot be traversed, enumerated by guessing, or grown without bound.
- **No byte-stream I/O.** There is no `open`, `read`, `write`, `seek`, `append`, or `stat` over a file descriptor. The unit is a whole record (§3), sealed and written atomically.
- **No hierarchy or traversal.** There are no directories, no `readdir`, no `..`. Organization is a flat typed catalog queried by attribute (§5), not a tree walked by path.
- **No mutable metadata tree.** The catalog carries a fixed schema of non-secret attributes; it is not an extensible attribute filesystem.

The prior art MBS resembles is the sealed-object keystore — PKCS#11 (Cryptoki), a TPM's sealed non-volatile index set, a JWK set — an organized, API-driven store of opaque sealed objects that is not, and does not claim to be, a filesystem. MBS is not a general-purpose object store either: the slot set is fixed and typed at provision time.

## 3. The Stored Unit

The stored unit is a **record** of a fixed type, not a byte buffer. An MBS instance is parameterized by its record type, written `MBS<'Record>`. A record is written whole and read whole; there is no partial access. This yields crash-consistency without a journal: a slot write either completes and the record is present, or it does not and the slot holds its prior contents. There is no torn-write intermediate state a reader can observe, because a reader addresses a slot only through a handle the store issues on a completed write.

A record is **sealed** before it enters storage (§6): the persisted form is ciphertext, and the record type describes the *plaintext* the store never itself holds.

Records compose. An MBS record may itself be a bundle of typed sub-slots (for the EK Credential Store, an identity record composed of a signing keypair, a KEM keypair, a certificate chain, and attributes). The index addresses records; storage granularity may be per-sub-slot. This is the modular structure the name refers to.

## 4. Addressing: Handles and Labels

The address of a stored record is an opaque, stable **handle**.

```clef
type Handle<'Record>    // opaque; issued by put, consumed by get; not forgeable, not arithmetic
```

`put` issues a handle on a completed write; `get` resolves a handle to a record. A handle is not a name, not a path, and not an integer the caller may compute — it is an opaque token the store defines, so the address space is closed by construction.

A record MAY carry a human-readable **label** among its non-secret catalog attributes (§5). A label is a catalog *attribute* used for `find`, not an *address*: two records may share a label, a label may be absent, and resolving a label yields zero or more handles, never storage directly. This keeps the handle the sole address while allowing enumeration and display to be legible.

## 5. Organization: The Typed Catalog

Organization in MBS lives in a **catalog** of non-secret attributes, one entry per stored record, queryable without unsealing any blob. The catalog is the *attributed tier*: it holds what may be known about a record without holding the record. Its schema is fixed for a given `MBS<'Record>` — typically a type tag, an optional label, a usage class, and a creation epoch — and carries no secret material.

```clef
Mbs.list  : MBS<'R> -> Handle<'R> array                          // every present handle
Mbs.find  : MBS<'R> -> (CatalogEntry -> bool) -> Handle<'R> array // query by non-secret attribute
```

Querying the catalog is how a caller organizes and selects records — by type, by label, by usage, by age — with no blob read and no unseal. The organization is real and useful; it is simply not a directory tree.

## 6. Sealing and the Access Policy

The read/write boundary of MBS is a **seal**. A record is sealed on the way in and unsealed on the way out, by a target-provided sealing capability (for the EK, the SCE9 crypto engine under the hardware-unique key). The persisted blob is ciphertext; the plaintext record exists only transiently in a secure working region after an unseal, never in the storage medium.

This means the storage medium need not itself be access-controlled: its security rests on the seal, not on the medium's attribution. A target may place sealed blobs in bulk storage that is not itself secure-attributed, because a sealed blob is meaningless without the sealing key, and the key resides in the target's sequester.

```clef
Mbs.put : MBS<'R> -> 'R -> Handle<'R>                 // seals, then writes whole; returns the handle
Mbs.get : MBS<'R> -> Handle<'R> -> 'R                 // reads whole, then unseals into a secure region
Mbs.evict : MBS<'R> -> Handle<'R> -> unit             // removes the sealed blob and its catalog entry
```

`put` and `get` cross the seal; the plaintext `'R` is present to the caller only inside the secure region the unseal targets. Constant-time and secret-independence obligations on the sealing path ride the PSG through to the seal, so the sealing itself is subject to the same proof discipline as the rest of the program.

## 7. Allocation and the No-Heap Discipline

An MBS instance is a **fixed set of typed slots**, sized at build or provision time. It does not grow dynamically and does not allocate on a heap. On a target without a heap (a freestanding unikernel), MBS is the only durable store available, and its fixed-slot structure is what makes it admissible there: the slot count and slot sizes are program-lifetime facts, so the store's own footprint is placed in static storage (the program-lifetime point of the [lifetime lattice](closure-representation.md#33-escape-analysis)), and the working set of an unsealed record is placed in a bounded secure region.

A full store is a reported condition, not a silent failure or an unbounded grow: `put` into a store with no free slot of the record's class is an error the caller handles, in the same spirit as an unobservable range being a reported error rather than a silent default.

## 8. The Durability Coeffect

MBS is the storage-facing face of a **durability coeffect** carried on the PSG:

- **survives-power-loss** — the value's storage outlives the process; it is placed in non-volatile storage, not volatile working memory.
- **atomic-write** — the value is written whole; no partial-write intermediate is observable.
- **sealed** — the value crosses the medium as ciphertext; plaintext exists only inside the sequester.

Like the region, width, and representation coeffects, the durability coeffect is analyzed and carried on the graph and committed at target-binding: the middle end records *that* a value is durable, atomic, and sealed, and the target's backend leg commits *how* (which non-volatile medium, which atomic-write primitive, which sealing capability). A target without a sealing capability or without non-volatile storage cannot host an MBS instance, and that is a capability failure at binding, never a silent fallback to volatile or unsealed storage.

## 9. Relationship to Other Features

- [Memory Regions](memory-regions.md) — MBS storage is placed by region and lifetime; the sealed medium and the on-chip catalog and the secure working region are distinct regions with distinct access kinds.
- [Closure Representation](closure-representation.md) — the lifetime lattice's program-lifetime (static) point places the store's own fixed structure; MBS extends persistence one rung beyond, to lifetimes that outlive the process.
- The **server-scale generalization** — an open-namespace, dynamically-sized document store is the same durability coeffect scaled to an open set of untyped objects. MBS is deliberately the closed, typed, fixed-slot case; the open case is future work and is where a filesystem-shaped model would belong. MBS does not become that by growing; a different model would.

## 10. Normative Requirements

1. **No path namespace.** An MBS instance SHALL NOT present a hierarchical path namespace; records SHALL be addressed by opaque handles.
2. **Whole-record access.** A record SHALL be written whole and read whole; MBS SHALL NOT present partial (seek/append) access to a record.
3. **Atomic write.** A `put` SHALL either complete and make the record retrievable by its issued handle, or leave the target slot unchanged; no partial-write state SHALL be observable through a handle.
4. **Sealed at rest.** A persisted record SHALL be sealed; the plaintext record SHALL exist only inside a secure working region after an unseal, never in the storage medium.
5. **Fixed slots.** An MBS instance SHALL have a slot set fixed at build or provision time and SHALL NOT allocate on a heap; a full store SHALL be a reported error.
6. **Handles are opaque.** A handle SHALL be an opaque token issued by the store, not a caller-computable name, path, or index.
7. **Catalog carries no secrets.** The queryable catalog SHALL carry only non-secret attributes; querying it SHALL require no unseal.
8. **Durability is a coeffect.** The durability, atomicity, and sealing of a stored value SHALL be carried as a coeffect on the PSG and committed at target-binding; a target lacking non-volatile storage or a sealing capability SHALL be a capability failure, never a silent fallback.
