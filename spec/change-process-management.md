---
title: "Change Process and Specification Management"
weight: 2
category: Process
status: normative
---

> **How the Clef language and this specification are developed, proposed, reviewed, and amended.** This chapter is *process-normative*: it governs the procedure by which the normative content of every other chapter changes. It does not define language semantics; it defines the discipline under which those semantics evolve.

This chapter specifies how changes to the Clef language and this specification are proposed, evaluated, and recorded.

## 1. Governing principle

A language specification is trustworthy only if the *way it changes* is itself specified. Three commitments govern every change to Clef and to this document.

1. **No silent normative change.** A requirement in a released specification is added, altered, or removed only through a recorded process step. The provenance of each such requirement is recoverable from that record.
2. **Requirements are stated directly.** Normative chapters state adopted requirements. Proposals, unresolved design questions, and implementation progress belong in separate change records (§2).
3. **Implementation and specification advance together.** A feature is not "done" because it is described, nor because it compiles. A change is complete only when the specification text, the compiler implementation, and the conformance expectation agree. The process is built around keeping these three in step.

## 2. Specification content

Normative text SHALL state the requirements of the language independently of
implementation progress. A requirement has the same normative force whether it
originates in an external standard or a Clef design decision.

Proposals, unresolved design questions, implementation milestones, and change
history SHALL be recorded in suggestions, RFCs, RFDs, or implementation records.
They SHALL NOT appear as qualifications on normative requirements. Informative
notes and examples MAY explain a requirement without changing its force.

## 3. Change lifecycle

A change proceeds through suggestion, approval in principle, formal design,
preview, and release. RFC acceptance requires a final-review window and a
recorded disposition. Suggestions and RFDs retain unresolved design questions
until a concrete change can be specified in an RFC.

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

An accepted RFC is **implemented behind a preview gate** before it becomes a released, stable part of the language. Preview is where specification text meets running code: the feature is available for use under an explicit opt-in, its specification is exercised against a real implementation, and discrepancies between the two are reconciled by amending whichever is wrong. A feature is promoted from **preview** to **released** only when its specification, its implementation, and its conformance expectation agree and have been validated in preview. Any subsequent change to a released feature requires a fresh RFC.

## 5. The two committed-design tracks

### 5.1 RFC (Request for Comments)

An RFC proposes a **concrete change** to the Clef language or specification. It is the appropriate track when the shape of the change is clear enough to write down. An RFC is required for a change to a released specification when:

- a new language feature is proposed;
- an existing feature's semantics need revision;
- a normative section of the specification requires amendment; or
- an unresolved design question has a concrete proposed resolution.

**Required content.** Each RFC SHALL include:

1. **Summary** — a one-paragraph statement of the change.
2. **Motivation** — the problem solved and why the status quo is insufficient.
3. **Detailed design** — the precise normative text proposed, including each new or altered requirement.
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
| **Accepted** | Approved at the close of final review. The proposed normative text is scheduled for implementation behind the preview gate (§4.4); the specification incorporates the adopted requirements on release. |
| **Rejected** | Declined; the RFC is retained as a record of the decision and its reasoning. |
| **Postponed** | Neither accepted nor rejected; the question is real but not yet ripe, and the RFC remains a recoverable anchor for the open item. |
| **Superseded** | Replaced by a later RFC; the record points forward to its replacement. |

An accepted-and-released RFC is the mechanism by which requirements in a released specification are added, altered, or removed.

### 5.2 RFD (Request for Discussion)

An RFD opens a **broader design discussion** without committing to a specific proposal. It is the appropriate track when:

- a problem area is identified but the solution space is not yet clear;
- multiple competing approaches exist and review would inform the direction; or
- a cross-cutting concern affects several areas of the language or toolchain at once.

**Required content.** An RFD SHALL include the problem statement, the constraints any solution must satisfy, and — where known — the candidate approaches and their tradeoffs. An RFD is *not* required to recommend a single answer; its task is to map the design space honestly, including admitting where the boundary of the tractable solution is itself unknown.

**Outcome.** An RFD concludes, with the conclusion recorded, in one of three ways:

- **Spawns RFC(s)** — the discussion converges on one or more concrete proposals, which proceed on the RFC track.
- **No change needed** — the status quo is found correct; the RFD is retained as the record of *why*.
- **Deferred** — the question is real but not yet ripe; the RFD retains the unresolved question in the change records.

### 5.3 Choosing the track

The distinction is **concreteness**, not size. A small but specific change (renaming a module, amending a coverage rule) is an RFC. A large but open question (how to reach a whole class of targets) is an RFD until it converges, then becomes one or more RFCs. When in doubt, the safer entry is the earlier stage: it costs little to discover an idea was already settled, and it is cheaper to refine a proposal that began as exploration than to relitigate one committed too early.

## 6. Compatibility and the released surface

Once a feature is **released**, the expectation a developer may rely on is that conforming Clef code continues to compile and to mean the same thing across compiler versions, except where a subsequent RFC explicitly and with a stated migration path changes it. The standard library that ships with the language is held to the same expectation: a released library surface is a normative interface, and a breaking change to it is an RFC-governed event, not an implementation detail. This is the compatibility discipline the F# lineage observes for its core library, carried into Clef as a normative commitment rather than a convention.

## 7. Change records

Each change record SHALL identify the affected requirements, the proposed
normative text, and the decision reached. Implementation progress and unresolved
design questions SHALL remain in those records. The specification SHALL contain
the resulting requirements without drafting notes or implementation-status
qualifications.

## 8. Process-normative requirements

1. **Normative content.** Requirements SHALL be stated directly as specified in §2; proposals and implementation status SHALL be maintained in separate change records.
2. **Funnel.** A change to a released language feature or specification requirement SHALL pass the approval-in-principle gate (§4.2) before entering committed design (RFC or RFD), and an accepted RFC SHALL be delivered through the preview gate (§4.4) before its requirements are treated as released.
3. **Recorded change.** A change to a requirement in a released specification SHALL proceed through an accepted-and-released RFC. Proposals MAY originate from a suggestion or an RFD.
4. **Recorded disposition.** An RFC SHALL NOT be accepted except at the close of a final-review window with a recorded disposition (§5.1).
5. **Retention.** Declined suggestions and rejected, postponed, deferred, and superseded proposals SHALL be retained as records, not deleted, so that resolved questions are not silently reopened.
6. **No silent normative change.** A requirement in a released specification SHALL NOT be added, altered, or removed by an unannounced edit.
7. **Released-surface compatibility.** A breaking change to a released language or standard-library surface SHALL be governed by an RFC with a stated migration path (§6).
