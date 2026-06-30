---
title: "Change Process and Specification Management"
weight: 2
category: Process
status: normative
---

> **How the Clef language and this specification are developed, proposed, reviewed, and amended.** This chapter is *process-normative*: it governs the procedure by which the normative content of every other chapter changes. It does not define language semantics; it defines the discipline under which those semantics evolve.

This specification is under active internal development by SpeakEZ Technologies. As the Clef language and its compiler toolchain mature, the normative content of this document is expected to undergo substantial revision: chapters will be added, reorganized, and rewritten as implementation experience reveals better formulations. This chapter specifies the process by which those revisions are proposed, evaluated, and recorded, so that change is deliberate, traceable, and reviewable rather than ad hoc.

## 1. Governing principle

A language specification is trustworthy only if the *way it changes* is itself specified. Three commitments govern every change to Clef and to this document.

1. **No silent normative change.** Once the process is active (§7), a normative requirement is added, altered, or removed only through a recorded process step. The provenance of every load-bearing requirement is recoverable from that record.
2. **Status is always marked.** Every requirement carries an explicit maturity marker (§2). A reader can tell at a glance whether a statement is settled, an adopted design decision, or a genuinely open question. Presenting an open item as settled is itself a process violation.
3. **Implementation and specification advance together.** A feature is not "done" because it is described, nor because it compiles. A change is complete only when the specification text, the compiler implementation, and the conformance expectation agree. The process is built around keeping these three in step.

## 2. Status discipline (the maturity vocabulary)

Every normative chapter marks the maturity of its requirements with a fixed vocabulary, and the same markers are the unit a process step acts on.

- **Normative (unqualified).** A requirement that follows from prior art, an external standard, or as a forced consequence of an already-settled mechanism. Stated without qualification.
- **[Design decision].** A requirement this specification *adopts* — implementable and recommended, but not compelled by external canon. A genuine choice the project made, which an RFC may revisit, harden, or reverse.
- **[Not yet specified].** A genuinely unresolved item, named rather than invented. Each `[Not yet specified]` item is a standing seed for a suggestion, an RFD, or an RFC.

A process step is, in effect, a transition between these markers: promoting a `[Not yet specified]` item to a `[Design decision]`, hardening a `[Design decision]` into unqualified normative text, or reopening settled text for reconsideration. The set of `[Not yet specified]` items across the specification constitutes the project's open design backlog at any moment.

## 3. Lineage and the chosen model

Clef's development model is adapted from the **F# language design process**, which Clef's surface and type system already descend from. F# separates *idea capture* from *committed design*: open-ended suggestions are gathered, discussed, and voted in one venue; only ideas "approved in principle" are promoted to a formal RFC with a concrete design and a tracked implementation, and an accepted RFC then lands first behind a *preview* gate before becoming a released, stable feature. This staged funnel — broad intake, a deliberate approval gate, RFC formalization, preview, release — is the spine of the process specified here. F# operates this process effectively even though it is not, at the time of writing, formalized within the F# specification itself; Clef adopts the same staged structure and specifies it normatively.

The lifecycle rigor of the RFC stage — an explicit final review window with a recorded disposition before a proposal is accepted — follows the discipline established by the Rust RFC process. The two influences compose cleanly: the F# *funnel* decides what becomes an RFC; the Rust-style *final-comment discipline* decides when an RFC is accepted.

## 4. The development funnel

A change to Clef proceeds through four stages of increasing commitment. Not every idea reaches the end; the funnel exists precisely so that scrutiny increases as commitment increases.

### 4.1 Suggestion (idea intake)

A **suggestion** is an open, low-ceremony proposal: "Clef should be able to do X," or "feature Y is awkward and should change." Suggestions are the intake stage. They are discussed and prioritized, and the great majority of design conversation happens here, before any commitment is made. A suggestion requires only a clear statement of the desired capability or the problem observed; it does not require a design.

A suggestion has three possible outcomes: it is **approved in principle** and promoted toward an RFC or RFD; it is **declined**, with the reasoning recorded so the same idea is not silently re-raised; or it remains **open** for further discussion.

### 4.2 Approval in principle (the gate)

**Approval in principle** is the deliberate gate between open discussion and committed design. It is a decision by the Clef language design authority that an idea is worth the cost of a formal design, *without yet committing to its specific form*. Approval in principle does not guarantee the feature will ship; it commits only to designing it properly and evaluating that design on its merits. Promotion through this gate produces either an RFC (when the shape of the change is clear enough to specify) or an RFD (when the solution space must be mapped first).

### 4.3 RFC and RFD (committed design)

The two committed-design tracks are specified in §5. An RFC carries a concrete proposal to a recorded accept/reject decision; an RFD maps an open design space and typically spawns one or more RFCs.

### 4.4 Preview and release (delivery)

An accepted RFC is **implemented behind a preview gate** before it becomes a released, stable part of the language. Preview is where specification text meets running code: the feature is available for use under an explicit opt-in, its specification is exercised against a real implementation, and discrepancies between the two are reconciled by amending whichever is wrong. A feature is promoted from **preview** to **released** only when its specification, its implementation, and its conformance expectation agree and have been validated in preview. A released feature's normative text carries no maturity qualifier; its stability is the default expectation, and any subsequent change to it requires a fresh RFC.

## 5. The two committed-design tracks

### 5.1 RFC (Request for Comments)

An RFC proposes a **concrete change** to the Clef language or specification. It is the appropriate track when the shape of the change is clear enough to write down. An RFC is required when:

- a new language feature is proposed;
- an existing feature's semantics need revision;
- a normative section of the specification requires amendment;
- a `[Design decision]` is to be hardened, revisited, or reversed; or
- a `[Not yet specified]` item has a specific proposed resolution.

**Required content.** Each RFC SHALL include:

1. **Summary** — a one-paragraph statement of the change.
2. **Motivation** — the problem solved and why the status quo is insufficient.
3. **Detailed design** — the precise normative text proposed, including each new or altered requirement and the §2 marker it will carry.
4. **Drawbacks** — the costs and risks the change introduces.
5. **Alternatives** — the design options weighed and why the proposed one was chosen. (An RFC descended from an RFD inherits this from the RFD's exploration.)
6. **Impact and migration** — the effect on existing Clef code, on other specification chapters, and on the toolchain; whether the change is backward-compatible, and the migration path if it is not.
7. **Unresolved questions** — what the RFC deliberately leaves open, to be settled in implementation or a follow-up RFC.

**Lifecycle.** An RFC proceeds through explicit stages:

| Stage | Meaning |
|---|---|
| **Draft** | Authored and open to revision by its author; not yet under formal review. |
| **Review** | Under formal review by the language design authority; feedback gathered, design refined. |
| **Final review** | A bounded review window with a proposed **disposition** — *accept*, *reject*, or *postpone* — opened only when the tradeoffs have been sufficiently discussed for a decision. Entering final review requires the design authority to have reviewed the RFC in full. The window is advertised so that any final objection can be lodged before a decision is reached; a substantial new objection cancels final review and returns the RFC to **Review**. |
| **Accepted** | Approved at the close of final review. The proposed normative text is scheduled for implementation behind the preview gate (§4.4); affected requirements take their stated markers once released. |
| **Rejected** | Declined; the RFC is retained as a record of the decision and its reasoning. |
| **Postponed** | Neither accepted nor rejected; the question is real but not yet ripe, and the RFC remains a recoverable anchor for the open item. |
| **Superseded** | Replaced by a later RFC; the record points forward to its replacement. |

An accepted-and-released RFC is the only mechanism by which unqualified normative text is added or altered once the process is active (§7).

### 5.2 RFD (Request for Discussion)

An RFD opens a **broader design discussion** without committing to a specific proposal. It is the appropriate track when:

- a problem area is identified but the solution space is not yet clear;
- multiple competing approaches exist and review would inform the direction; or
- a cross-cutting concern affects several areas of the language or toolchain at once.

**Required content.** An RFD SHALL include the problem statement, the constraints any solution must satisfy, and — where known — the candidate approaches and their tradeoffs. An RFD is *not* required to recommend a single answer; its task is to map the design space honestly, including admitting where the boundary of the tractable solution is itself unknown.

**Outcome.** An RFD concludes, with the conclusion recorded, in one of three ways:

- **Spawns RFC(s)** — the discussion converges on one or more concrete proposals, which proceed on the RFC track.
- **No change needed** — the status quo is found correct; the RFD is retained as the record of *why*.
- **Deferred** — the question is real but not yet ripe; the RFD remains open as a marked, recoverable `[Not yet specified]` anchor.

### 5.3 Choosing the track

The distinction is **concreteness**, not size. A small but specific change (renaming a module, amending a coverage rule) is an RFC. A large but open question (how to reach a whole class of targets) is an RFD until it converges, then becomes one or more RFCs. When in doubt, the safer entry is the earlier stage: it costs little to discover an idea was already settled, and it is cheaper to refine a proposal that began as exploration than to relitigate one committed too early.

## 6. Compatibility and the released surface

Once a feature is **released**, the expectation a developer may rely on is that conforming Clef code continues to compile and to mean the same thing across compiler versions, except where a subsequent RFC explicitly and with a stated migration path changes it. The standard library that ships with the language is held to the same expectation: a released library surface is a normative interface, and a breaking change to it is an RFC-governed event, not an implementation detail. This is the compatibility discipline the F# lineage observes for its core library, carried into Clef as a normative commitment rather than a convention.

## 7. Current status

The process described here is **not yet active.** It will launch when the initial language design and specification have reached a stable baseline. Until then:

- changes to this specification are made directly by the core development team;
- the **status markers of §2 are the operative discipline** — every requirement carries its maturity marker, and the `[Not yet specified]` items are the standing record of what remains open; and
- those open items will form the initial backlog when the process activates.

On activation, direct core-team edits give way to the funnel and the two-track process for all normative change, and the §8 requirements take full effect.

## 8. Process-normative requirements

1. **Marked status.** Every normative requirement SHALL carry one of the §2 maturity markers; an unmarked requirement defaults to unqualified normative and SHALL NOT be `[Not yet specified]` content presented as settled.
2. **Funnel.** A change SHALL pass the approval-in-principle gate (§4.2) before entering committed design (RFC or RFD), and an accepted RFC SHALL be delivered through the preview gate (§4.4) before its requirements are treated as released.
3. **Recorded change (once active).** Once the process is active, a change to unqualified normative text SHALL proceed through an accepted-and-released RFC; `[Not yet specified]` and `[Design decision]` items MAY additionally originate from a suggestion or an RFD.
4. **Recorded disposition.** An RFC SHALL NOT be accepted except at the close of a final-review window with a recorded disposition (§5.1).
5. **Retention.** Declined suggestions and rejected, postponed, deferred, and superseded proposals SHALL be retained as records, not deleted, so that resolved questions are not silently reopened.
6. **No silent normative change.** Once the process is active, a normative requirement SHALL NOT be added, altered, or removed by an unannounced edit.
7. **Released-surface compatibility.** A breaking change to a released language or standard-library surface SHALL be governed by an RFC with a stated migration path (§6).
