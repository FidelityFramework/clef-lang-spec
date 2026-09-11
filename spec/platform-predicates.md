---
title: "Platform Predicates Specification"
weight: 540
category: Platform
status: normative
---

> **Scope**: Implemented static device-access predicate fragment; extensions are identified below.
> **Last Updated**: 2026-09-10

## 1. Predicate authority

A Clef predicate preserves a proposition, its declaration dependencies and its
standing until a concrete use requires a decision. CCS owns that decision.
Composer consumes the resulting evidence; it SHALL NOT reinterpret quotations
or infer hardware capabilities from architecture names.

This chapter supersedes the earlier Alex-time capability-resolution design.
The implemented consumer is static MMIO binding. General capability dispatch,
runtime mapping guards and program-wide relational proofs are not implemented
by this fragment.

## 2. Source declaration

Fidelity.Platform.Contracts declares:

```clef
type ClefPredicate = {
    Name: string
    Condition: Expr<bool>
    Source: string
}
```

`Condition` SHALL be a typed Clef quotation. A quotation is compile-time syntax
and has no runtime value ([quoted expressions](expressions.md)). `Source`
identifies the requirement or provenance; its text is not proof evidence.

A relation may refer to immutable declaration values:

```clef
let pwmTicks: ClefPredicate = {
    Name = "whole-pwm-clock-ticks"
    Condition = <@ pwmInterruptHz > 0 && applicationIclkHz % pwmInterruptHz = 0 @>
    Source = "Application clock and SysTick plan"
}
```

The identifiers in this example must be declared in scope. The condition checks
the relationship between those declarations. It does not measure a clock or
prove the correctness of the code that establishes it.

## 3. Implemented expression fragment

The evaluator supports integer and Boolean literals, immutable references,
fields of declaration-only records, arithmetic, comparisons, same-kind equality,
Boolean operators and conditional expressions. Integer arithmetic in this
fragment is mathematical, with no host-width truncation.

Explicit fixed-width arithmetic, conversions, general function calls, mutable
bindings and records accessible by runtime code are outside the fragment.
Division by zero is undecided. A source type error in a dependency SHALL prevent
a used condition from establishing access, even when the quotation is erased
from executable reachability.

The implementation SHALL retain the declaring node, expression node, dependency
nodes, source provenance and one of these states:

| State | Meaning at the declared dependencies |
| --- | --- |
| Established | The supported relation evaluates to true |
| Contradicted | The supported relation evaluates to false |
| Pending | Required information or supported semantics are missing |

Pending SHALL NOT be interpreted as either Boolean value. An unused declaration
may remain pending. A used binding that needs it SHALL be rejected until it can
be established; this implementation does not synthesize a runtime guard.

## 4. Device access consumer

Contracts separates `DeviceRegion`, `DeviceRegister`, `DeviceMapping`,
`DeviceGrant` and `DeviceAccessPlan`. Regions reference the actual BAREWire
`MemorySpace` declarations in the selected platform. Availability of a register
SHALL NOT imply a workload grant.

A selected plan closes raw-address MMIO construction for that workload.
`Mmio.bind8/16/32 grantName registerName` selects a register within a named grant.
The names SHALL be immutable static strings; they are erased from executable
storage. The existing read/write accessors consume the resulting opaque handle.

Before lowering, CCS SHALL establish:

- Region containment, mapping/address extent, alignment and granularity.
- Agreement between the accessor width and declared hardware transaction.
- Both register and workload permissions for the requested operation.
- Full unsigned range coverage for a written value.
- Supported byte order and ordering semantics.
- The mapping establishment and lifetime requirements of this implementation.
- Every additional predicate required by the grant.

An additional condition, including `<@ true @>`, SHALL NOT waive these checks.
Source integers and memory/transaction requirements remain distinct from the
concrete Pointer width in the platform. Current lowering supports 32-bit and
64-bit pointers with volatile 8/16/32-bit transactions.

Only nonzero static bases and image-lifetime mappings are currently lowered.
`reset-identity` requires equal region/mapping bases and address-space names.
`boot-contract` admits a CPU-visible placement supplied by an external boot
contract. A missing base, runtime establishment or scoped lifetime remains
pending when used. Unused declarations may retain these requirements or
transaction widths for which no accessor has yet been implemented.

## 5. Evidence and trust boundary

CCS stores per-site `MmioAccessEvidence` in graph codata, including the resolved
address, transaction, grant/declaration identities and predicate evidence.
Composer lowers this established information. With intermediates enabled, it
serializes the evidence to `device-access.json`.

Hardware documentation and boot mapping assertions SHALL be recorded separately
as external premises. Arithmetic does not prove physical wiring, page tables,
memory attributes, DMA visibility or lifetime of an external mapping. Grants
also do not themselves establish MPU/MMU/TrustZone isolation. Assembly and
foreign code remain explicit trust boundaries.

The current closed-expression checks are compiler evidence, not an SMT solver
certificate. Existing typed `ObligationBody` families and their proof dispatch
remain separate mechanisms. A future symbolic or runtime extension must connect
its evidence to the actual operation and its scope before claiming support.

## 6. Capability declarations and remaining design work

The legacy `PlatformPredicate` union and Boolean `PlatformContext.Predicates`
map are retained in CCS, but no general resolver/consumer populates that map.
Literal Ariel capability quotations do not by themselves implement automatic
pthread selection or conditional compilation.

Future capability consumers must distinguish instruction availability, numeric
representation, concurrency support and deployment permission. Neither a
64-bit word size nor an architecture name establishes vector extensions,
atomicity, cache behavior or access to a particular device.

Managed-substrate requirements such as shared-memory availability and the exact
integer envelope remain relevant to [width inference](width-inference.md) and
[behavior classification](behavior-classification.md). They need a concrete
host/platform declaration and consumer. This fragment does not enforce a
universal capability list or the old cross-architecture implication matrices.

## 7. Diagnostics

| Code | Meaning |
| --- | --- |
| CCS8209 | Malformed, inconsistent or ambiguous device-access declaration/selection |
| CCS8210 | A used MMIO operation lacks established access evidence, including a required false/pending predicate |

Ordinary source type errors retain their original diagnostics. CCS8210 may also
identify a source declaration error that would otherwise be hidden by erased
reachability. CCS8066 continues to reject quotations used as runtime values.
The diagnostic registry is [error handling](error-handling.md).

## 8. Implementation references

- [Contracts and acceptance scope](../../Fidelity.Platform/docs/MMIO_CONTRACTS.md).
- [Predicate evaluator](../../clef/src/Compiler/PSGSaturation/SemanticGraph/Predicates.fs).
- [Device-access settlement](../../clef/src/Compiler/PSGSaturation/SemanticGraph/DeviceAccess.fs).
- [Composer F# regression runner](../../Composer/tests/DeviceAccess/Program.fs).
- [Platform bindings](platform-bindings.md) and [numeric types](ntu-types.md).
