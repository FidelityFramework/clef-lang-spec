---
title: "Namespace Storage"
weight: 467
category: Representation
status: normative
---

> **Status**: Draft (captured design; not yet ratified). This chapter defines **Namespace Storage (NSS)**, the layer that gives [Modular Blob Storage](modular-blob-storage.md) names, history, and growth — the first rungs of the filesystem ladder MBS §1 describes. A target's instantiation is documented with that target.
>
> **Profile**: This chapter belongs to the **Freestanding Substrate** profile ([Conformance §7](conformance.md)). Its requirements bind an implementation that claims that profile; an implementation that delegates namespace persistence to a hosting environment does not claim the profile, and these requirements do not apply to it.

## 1. Overview

MBS is deliberately nameless: records are addressed by opaque handles, and MBS §1 enumerates the four things a filesystem adds above such a substrate — an open namespace, a path hierarchy, run-time storage growth, and a mutable metadata tree. This chapter defines the first rungs of that ladder in a form a constrained target can afford, and in a form that scales without changing shape.

The central decision is that **the namespace has no mutable tree**. Namespace state is the fold of an append-only, hash-linked **ledger** of change entries, checkpointed into compressed, sealed **segments** that are stored as ordinary MBS records. Mutation is appending; history is primary; the current tree is derived. A constrained target holds only a bounded hot set in RAM; everything cold is a sealed blob like any other.

Two consequences follow. First, the filesystem's own metadata needs no second storage substrate: cold namespace state rounds back into MBS, so the fixed-slot floor and any larger target share one persistence mechanism. Second, crash consistency, wear behavior, point-in-time recovery, and replication all reduce to properties of one structure — the ledger — rather than to separate machinery.

*Prior art (informative).* The two-tier separation of nameless blob storage from a namespace layer follows the Haystack lineage. The demotion of cold namespace metadata into compressed chunks stored as ordinary blob data is the move SeaweedFS ships as sealed directories, at cluster scale; NSS adopts it as the *default* representation rather than an optimization, because a constrained target cannot afford the mutable-tree representation in the first place. The ledger-first, replay-derived posture is the event-sourcing discipline, and the framework's duality proposal gives it a typed justification (§6). Log-structured wear behavior at the flash layer is familiar from embedded filesystems such as littlefs; NSS obtains it at the namespace layer by construction.

## 2. The Ledger

The ledger is an append-only sequence of **change entries**. Each entry is written whole as an MBS record and records one namespace transition as an *(old, new)* pair:

```clef
type ChangeEntry = {
    Prev    : Digest              // hash of the preceding entry; the chain
    Old     : NameBinding option  // None for a create
    New     : NameBinding option  // None for a remove
    Epoch   : Epoch               // creation epoch, monotone
}

type NameBinding = {
    Parent  : NodeId              // the containing directory node
    Name    : Name                // the binding's name within it
    Target  : Handle<Blob>        // the MBS handle the name resolves to, or a directory node
    Attrs   : Attrs               // fixed, non-secret attributes
}
```

The *(old, new)* pair derives every transition kind without an operation vocabulary: a create carries only `New`, a remove only `Old`, an update both, and a rename both with differing `Parent`/`Name`. A consumer that folds entries in order reconstructs the namespace at any prefix; a consumer that folds to the end holds the current namespace.

Each entry carries the digest of its predecessor. The chain makes the ledger tamper-evident and gives replay a verifiable spine: a fold either consumes an unbroken chain or reports the break's position. Where a deployment signs entries or checkpoints, the signing policy is a deployment configuration in the manner of the framework's version records; the chain is the structural floor beneath any such policy.

Ledger entries are MBS records: written whole, atomic per MBS requirement 3, sealed at rest per MBS requirement 4. The ledger inherits crash consistency from the substrate — an append either becomes retrievable or the ledger's tail is unchanged — and inherits wear-friendliness from its own shape, since nothing is ever rewritten in place.

## 3. Segments

Unbounded replay is not affordable on a constrained target, so the ledger is **checkpointed**. A **segment** is the fold of a ledger prefix over one subtree, serialized, compressed, sealed, and written as an ordinary MBS record — namespace metadata demoted to data. A segment carries a small **segment index** mapping names to offsets within it, so resolving one name reads one segment and decompresses one frame, not a tree.

```clef
Nss.checkpoint : Nss -> SubtreeId -> Handle<Segment>   // fold, compress, seal, store
Nss.resolve    : Nss -> Path -> NameBinding option     // hot set, then segments, then ledger tail
```

The **root record** binds the whole: it names the current segment set and the ledger position they checkpoint. Advancing a checkpoint writes the new segment, then swaps the root record — one whole-record MBS write, atomic per the substrate — after which the checkpointed ledger prefix is dead and its slots may be evicted. A crash before the swap leaves the prior root intact with a longer replay; a crash after leaves the new root with a shorter one. No state between the two is observable.

Compression precedes sealing, and both are per-target decisions of record in the manner of MBS §5.2: a codec realized as verifiable code carries its proof obligations on the PSG; a hardware engine states its verifiability cost. The segment index is secret-free in the manner of MBS's index, so segment selection requires no unseal.

## 4. Resolution and the Hot Set

A resolving target holds a bounded **hot set**: the root record, open bindings, and recently resolved entries, in a RAM region sized at provision time. Resolution consults the hot set, then the segment indexes, then replays the ledger tail past the checkpoint. Every stage is bounded: the hot set by provision, a segment read by one record, the tail by compaction policy. There is no unbounded traversal and no heap; a namespace that outgrows its provisioned segment slots is a reported error in the manner of MBS §6.

Growth is therefore run-time in extent but provisioned in envelope: the namespace grows by appending entries and adding segments within the slot budget the target declared. The open namespace and path hierarchy of MBS §1 are supplied; unbounded growth is deliberately not, on this profile.

## 5. The Durability Coeffect, Extended

NSS extends the durability coeffect of MBS §7 with two components carried the same way on the [Program Semantic Graph](program-semantic-graph.md):

- **chained** — the value's history is hash-linked; replay is verifiable, and truncation is detectable.
- **replayable** — the current state is the deterministic fold of the ledger; state at any checkpoint boundary is recoverable by construction.

Like the substrate's components, these are analyzed in the middle end and committed at target binding, which selects the digest primitive, the codec, and the checkpoint policy. Point-in-time recovery, replication, and audit all consume the same two components rather than adding mechanisms.

## 6. Recorded and Computed Reversibility (informative)

The framework's duality proposal distinguishes computed reversal — an inverse re-run live against an η/ε pairing carried in the graph — from recorded reversal, a durable log replayed. The boundary is decidable from the types: an effect whose inverse depends on state outside the program is log work. A write to persistent media is the canonical such effect, which places the storage layer's reversibility on the recorded side *by type discipline*, not by convention. The ledger is that record, in the minimal form the discipline requires: (old, new) pairs, chained, sealed.

On targets past the constrained floor, the two mechanisms compose: in-graph structures reverse computationally where the pairing certifies completeness, and the ledger holds exactly the sites whose inverses cross the durability boundary. The fractional discipline sketches the sharing account — read-shares of sealed segments as `1/T` obligations, with checkpoint compaction demanding the unified whole — and is a research direction of the duality proposal, not a requirement of this chapter.

## 7. Server-Scale Continuity (informative)

Nothing in §§2–5 names a scale. At cluster scale the same structures reappear: the ledger becomes the metadata event stream peers subscribe to and replay from a position; segments become compressed metadata chunks resident in bulk blob storage; the root record becomes the store's superblock; custody generalizes from the device sequester to a metadata-store keyring over media that may run anywhere. An S3-compatible object service over this design — resolution, sharding, and sealing in one sealed image, in the tradition this framework's unikernel direction describes — is a target instantiation, documented with that target when specified.

## 8. Relationship to Other Features

- [Modular Blob Storage](modular-blob-storage.md) — the substrate. Ledger entries, segments, and the root record are MBS records; NSS adds no second persistence mechanism.
- [Memory Regions](memory-regions.md) — the hot set is a bounded RAM region with its own access kind; sealed media carry none.
- [Closure Representation](closure-representation.md) — the lifetime lattice places the hot set at program lifetime; the ledger and segments extend persistence past the process as MBS does.
- [Credential Authority](credential-authority.md) — where entries or checkpoints are signed, the credentials and their custody are that chapter's subject.

## 9. Normative Requirements

1. **Ledger-first.** Namespace state SHALL be the fold of an append-only ledger of change entries; an NSS instance SHALL NOT maintain a mutable metadata tree as the persistent representation.
2. **Whole entries as records.** Each change entry SHALL be written whole as an MBS record, inheriting the substrate's atomicity and sealing.
3. **(Old, new) form.** Each entry SHALL carry the prior and new bindings as an *(old, new)* pair from which the transition kind is derived; NSS SHALL NOT persist an operation vocabulary in place of the pair.
4. **Hash chain.** Each entry SHALL carry the digest of its predecessor; a fold SHALL verify the chain and SHALL report a break's position.
5. **Segments are records.** A checkpoint SHALL serialize, compress, then seal a subtree fold into one or more MBS records carrying a secret-free index; resolving one name SHALL read at most one segment past the hot set and ledger tail.
6. **Atomic root.** The current segment set and checkpoint position SHALL be bound by a root record whose replacement is a single whole-record write; no intermediate checkpoint state SHALL be observable.
7. **Bounded resolution.** The hot set SHALL be a RAM region of provisioned size; resolution SHALL be bounded by the hot set, the segment indexes, and a compaction-bounded ledger tail; NSS SHALL NOT allocate on a heap.
8. **Provisioned envelope.** Segment and ledger slots SHALL be fixed at provision time on this profile; exhaustion SHALL be a reported error.
9. **Coeffect carriage.** The chained and replayable components SHALL be carried on the PSG with the durability coeffect and committed at target binding; a target lacking a digest primitive or a sealing capability SHALL be a capability failure.
10. **Verifiable transforms.** Where the codec or digest is realized as code, its proof obligations SHALL ride the PSG as for the rest of the program; a hardware realization SHALL be a decision of record with its verifiability cost stated.
