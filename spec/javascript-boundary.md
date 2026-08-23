---
title: "JavaScript Boundary Semantics"
weight: 680
category: Platform
status: normative
---

> **Status (August 2026)**: Design-stage. No conforming implementation exists.
>
> **Profile**: This chapter is assigned to the **JavaScript Substrate** profile ([Conformance §7](conformance.md)). Its requirements bind an implementation when, and only when, the implementation claims that profile by providing the JSIR target pathway ([Backend Lowering Architecture](backend-lowering-architecture.md)). An implementation with no JavaScript pathway is unaffected by this chapter in every respect.

This chapter defines the boundary between Clef code and the JavaScript host: the values that cross it, the two foreign types that carry what the crossing cannot convert, the narrowing discipline that is the sole path inward, the coeffect that makes foreign contact visible in signatures, the treatment of absence, the interception of host exceptions, and the normative requirements for binding generation tools like Xantham.

It is the JavaScript counterpart of [FFI Boundary Semantics](ffi-boundary.md). The two chapters instantiate one family invariant, stated in §1.1: each foreign boundary confines its own absence sentinels to its boundary conversions and admits values inward only through declared conversions. The C boundary's sentinel is `NULL` and its opaque carrier is `CHandle<'T>`; the JavaScript boundary's absence alphabet has three states (§5) and its carriers are the foreign pair of §2.

## 1. The Boundary Model

### 1.1 Core Invariant

> **A JavaScript value enters Clef only through a declared boundary function, is carried inland only by a foreign type, and leaves a foreign type only by narrowing. Within Clef code, no value is `null` or `undefined`.**

This is the JavaScript instance of the family invariant shared with [FFI Boundary Semantics §1.1](ffi-boundary.md): a foreign boundary's sentinels exist only at that boundary. Interior Clef has no `null` (the Nullness rules of [Types and Type Constraints](types-and-type-constraints.md)) and no `undefined`; absence is structure, an `Option` case or a union case. In the emitted JavaScript artifact, `null` and `undefined` occur only at the sites §8 enumerates.

### 1.2 The Two Worlds

```
┌─────────────────────────────────────────────────────────┐
│  Clef World                                             │
│                                                         │
│  T (declared types)     - established by narrowing      │
│  JsValue                - foreign, shape undetermined   │
│  JsRef<'T>              - foreign, opaque reference     │
│  Option<'T>             - absence as structure          │
│  Result<'T, 'E>         - failure as structure          │
└─────────────────────────────────────────────────────────┘
                  ↕ JavaScript Boundary
┌─────────────────────────────────────────────────────────┐
│  JavaScript World                                       │
│                                                         │
│  any value              - shape unknown until read      │
│  null / undefined       - two distinct absence values   │
│  absent property        - a third absence state         │
│  thrown values          - any value, at any call        │
└─────────────────────────────────────────────────────────┘
```

### 1.3 Rationale

The C boundary of [FFI Boundary Semantics](ffi-boundary.md) converts values whose shapes both sides fix at compile time; its hazard is the null sentinel, and `Option` marshalling retires it. The JavaScript boundary carries a second hazard the C boundary does not: many of the values that cross it originate outside the compiled program (a request body, a database row, a model response, an argument delivered to a callback), and the host erases declared types at runtime, so no declaration constrains what actually arrives. A property this specification requires an implementation to establish at design time cannot be established for such a value by analysis, because the value does not exist at design time.

The [preservation obligation](conformance.md) names the discharge for this situation: a property that cannot be preserved through a step is re-checked at the step that could perturb it. The boundary is that step. The premises a declared type states about an inbound value are discharged by a compiler-constructed check at the boundary, and the check's failure is a typed exit, a `Result` error naming the failed premise; a value of the wrong shape is never admitted inland. The mechanism, narrowing, is specified in §3.

Behavior arising entirely within foreign JavaScript or within the host runtime is beyond the reach of the language's guarantees and is a named residual of [Behavior Classification §4](behavior-classification.md).

## 2. The Foreign Pair

Two foreign types carry what the boundary cannot convert. They are the only types this chapter adds to the language surface.

### 2.1 JsValue

`JsValue` denotes a JavaScript value whose shape is undetermined at compile time. It corresponds to the positions the host's own declarations leave dynamic: `any` and `unknown` parameters and results, error channels, property bags read from foreign objects.

| Property | Rule |
|---|---|
| Subtyping | Participates in no subtyping relationship: not a top type, not a bottom type; no type coerces to or from it |
| Injection | Produced only at a declared boundary position (§3.1); no conversion in interior code yields a `JsValue` |
| Elimination | Narrowing (§3.2) is its only elimination |
| Operations | None: no property access, no invocation, no arithmetic, no comparison |

### 2.2 JsRef

`JsRef<'T>` is an opaque reference to a host object: held, stored, and passed back across the boundary, never opened. It is the JavaScript analog of `CHandle<'T>` ([FFI Boundary Semantics §2.1](ffi-boundary.md)). The parameter `'T` records the host type the reference designates; it gives distinct handles distinct types, and it grants no operations.

| Property | Rule |
|---|---|
| Subtyping | Participates in no subtyping relationship, including between instantiations |
| Injection | Produced only by a declared boundary function |
| Elimination | None in Clef code; its only use is as an argument to a declared boundary function |
| Operations | None |

### 2.3 Roles and Dispositions

The pair, together with the binding generator's own constructs, covers the dynamic positions a JavaScript host surface presents. No universal type is required and none is introduced: [Native Type Mappings](native-type-mappings.md) continues to reject `obj` unconditionally wherever this chapter is in force. The hazard of a universal type is subsumption, the unmarked conversion that lets any value become the universal type silently; the foreign pair admits no subsumption edge, so every contact is marked in the source by a boundary declaration and in the types by the grade of §4.

| Host surface position | Clef disposition |
|---|---|
| `any` / `unknown` parameter or result | `JsValue`, eliminated by narrowing |
| Opaque handle, host object held for later calls | `JsRef<'T>` |
| Union of concrete types | Erased union owned by the binding layer (§7.1) |
| Options bag | Generated nominal record with `Option` fields (§7.3) |
| Error channel, property bag | `JsValue`, narrowed to a declared shape |

## 3. Injection and Elimination

### 3.1 Boundary Functions

A **boundary function** is a function a binding declares whose signature carries a foreign type or performs a foreign conversion. Boundary functions are the only injection sites: interior code cannot produce a `JsValue` or a `JsRef<'T>` except by calling one.

Generated bindings (§7) declare the boundary functions for a host surface. The set of boundary functions in a program is therefore closed and enumerable, which is what makes the grade of §4 computable.

### 3.2 Narrowing

**Narrowing** is the sole elimination of `JsValue`: a generated, schema-directed check that converts a foreign value to a declared Clef type.

1. Narrowing SHALL be total over its input: every JavaScript value, including `null`, `undefined`, and values of unexpected shape, SHALL produce a defined result.
2. Narrowing SHALL return `Result<'T, 'E>` whose error case identifies the failed premise: the path into the value at which the check failed, and the shape the declaration expected there.
3. Narrowing SHALL be generated from the declared shape of the target type. The generated check reads the host's retained runtime evidence (property presence, `typeof` classification, array and null tests) to establish exactly the premises the declared type states.
4. A failed narrowing SHALL NOT admit a partially converted value.

```fsharp
// Declared target shape
type User = { Id: int; Name: string; Email: Option<string> }

// Generated narrowing (signature; the body is generated per §3.2.3)
val narrowUser : JsValue -> Result<User, NarrowingError>
```

> **Not yet specified.** The concrete shape of the narrowing error type (its path and expected-shape representation).

> **Clef Note**: Narrowing is the boundary instance of the [preservation obligation](conformance.md): premises the closed world discharges by analysis are discharged at the open boundary by a constructed check, with a typed exit on failure. The failure is a value, not a diagnostic. A malformed inbound payload is a runtime condition of the program, handled through `Result` like any other failure ([Error Handling](error-handling.md)).

### 3.3 Higher-Order Crossings

A callback that crosses from Clef into the host is later invoked by foreign code and receives foreign values. Its declared parameter types SHALL be established by narrowing at entry, under the rules of §3.2, before the callback body executes. A callback result that crosses outward is converted under the same outbound rules as any boundary result (§5.3).

## 4. The Boundary Grade

Contact with the foreign pair is a capability coeffect, the **boundary grade**, carried in the signature of every function that touches `JsValue` or `JsRef<'T>`.

1. An implementation claiming this profile SHALL compute the boundary grade of every function in a compilation unit.
2. A function whose boundary grade is zero is thereby proven free of foreign contact, transitively through everything it calls.
3. A project SHALL be able to require grade zero outside a designated interop layer, and a violation SHALL be diagnosable at the declaration that introduces the contact.

The grade is what makes the pair's confinement structural. An escape hatch policed by convention is invisible in a signature; the boundary grade is computed by the compiler and visible in every type it touches, so the extent of the interop layer is a checked property of the program and a dependency's foreign contact is visible to its consumers.

> **Clef Note**: The word *grade* in this section names the capability coeffect of foreign contact. It is unrelated to the algebraic grade of [Grade Discipline](grade-discipline.md); the two share only a solver family (the QF_BV lattice family of [Grade Discipline §4.1](grade-discipline.md), which already carries capability constraints).

## 5. Absence

### 5.1 The Three-State Alphabet

C has one absence sentinel. A JavaScript position has three states, and host APIs assign meaning to the differences:

| State | Example meanings on host surfaces |
|---|---|
| Absent (property or argument not present) | "keep the current value" in merge-patch conventions; an omitted optional argument |
| Present holding `undefined` | A lookup miss in some host APIs; an explicitly supplied "no value" |
| Present holding `null` | A lookup miss in other host APIs; SQL NULL carried through a driver; the only absence value JSON admits |

The differences are declared where the host surface is typed: TypeScript distinguishes an optional member (`?`, possibly absent or `undefined`) from a member typed `| null`. The binding generator reads that distinction (§7.1).

### 5.2 Inbound Conversion

1. For a declared `Option<'T>` position, the inbound conversion SHALL by default map all three absence states to `None`, and a present value that is neither `null` nor `undefined` through narrowing to `Some`.
2. Where the binding declares that a position distinguishes absence states (the declared distinction between optionality and `| null`, with JSON merge-patch as the canonical case: `null` meaning clear, absence meaning keep), the generated binding SHALL present a three-case union distinguishing *keep* (absent), *clear* (`null`), and *set with a value*, in place of `Option<'T>`.

> **Not yet specified.** The concrete naming of the generated three-case union.

### 5.3 Outbound Conversion

`None` has no single JavaScript lowering. For each outbound optional position, the binding SHALL select one representation (the key omitted, `null`, or `undefined`) from the declared surface of the host API at that position, and the generated conversion SHALL emit exactly that representation. The selection is made per position at generation time and SHALL NOT vary at runtime.

### 5.4 Interior Absence

Away from the boundary, absence remains structure. The realization of `Option` inside the emitted artifact (erased or reified) is a pathway decision specified in [Option Operations Representation](option-operations-representation.md); it is a representation choice, not a boundary conversion, and is governed there.

## 6. Host Exceptions

Clef has no exception control flow ([Error Handling](error-handling.md)). The host throws: synchronously from API calls, and asynchronously as rejected promises.

1. A boundary function whose host operation can throw SHALL intercept the thrown value at the boundary. The interception SHALL be generated into the binding; user code SHALL NOT be required to guard a boundary call.
2. An intercepted thrown value SHALL be surfaced as the error case of a `Result`, carried as `JsValue`, or narrowed to a declared error shape where the binding declares one.
3. A rejection delivered at a suspension point that a boundary operation awaited SHALL be intercepted under the same rules as a synchronous throw.
4. No host exception SHALL propagate into Clef code as nonlocal control flow.

> **Clef Note**: This is the JavaScript instance of the discipline the C boundary applies to error returns: the foreign failure convention is converted at the boundary into the language's own failure structure, once, by generated code ([FFI Boundary Semantics §5](ffi-boundary.md)).

## 7. Binding Generation Contract

This section defines normative requirements for Xantham and other JavaScript binding generation tools, parallel to the Farscape contract of [FFI Boundary Semantics §5](ffi-boundary.md).

> *Informative.* Xantham ingests TypeScript declarations through the TypeScript compiler API and emits a target-neutral structural analysis that the binding generator consumes. The tooling pipeline is a build-time concern; this section binds the mapping, not the tool's architecture.

### 7.1 Declared-Surface Mapping

A binding generator SHALL map the host's declared surface as follows:

| TypeScript declaration state | Clef surface |
|---|---|
| `T` (required; no null or undefined admitted) | `T` |
| `prop?: T` (optional member or parameter) | `Option<T>` |
| `T \| null` | `Option<T>` |
| `T \| undefined` | `Option<T>` |
| `prop?: T \| null` at a declared-distinction site (§5.2) | The generated three-case union |
| `any`, `unknown` | `JsValue` |
| A class or interface bound as an opaque handle | `JsRef<'T>` |
| A union of concrete types | An erased union owned by the binding layer |
| An overloaded function | Distinct declared functions, one per overload shape |
| An options-bag parameter | A generated nominal record with `Option` fields |

An **erased union** is a binding-layer construct: its introduction and elimination forms are generated with the binding, and it SHALL NOT enter the native type universe as a type ([Native Type Universe](native-type-universe.md)). The binding layer owns its representation on the wire side of the boundary and its typed elimination on the Clef side.

### 7.2 Defaults and Overrides

1. Where the host surface is declared (a typed SDK), the declaration as written SHALL be the default authority for the mapping of §7.1.
2. A generator SHOULD support a per-position override mechanism for sites where a host API's behavior is known to diverge from its declaration, and SHALL document each override it applies.
3. Where a generator detects a divergence between a declaration and observed behavior, it SHOULD surface the disagreement at generation time; it SHALL NOT silently prefer the observed behavior over the declaration.

### 7.3 Generated Binding Structure

1. Generated bindings SHALL declare the boundary functions of §3.1, and foreign types SHALL appear only at the positions the mapping of §7.1 assigns them.
2. Each inbound position with a declared shape SHALL receive a generated narrowing check (§3.2); each outbound optional position SHALL receive its selected representation (§5.3); each throwing operation SHALL receive interception (§6).
3. Options bags SHALL be generated as nominal records; erased unions SHALL be confined to the binding layer.

### 7.4 Regeneration

Bindings SHOULD be regenerated against each release of the host surface they bind. A generated binding is made current by regeneration; a hand-maintained binding for a moving surface accumulates silent divergence from it.

## 8. Emitted-Artifact Confinement

Within the JavaScript artifact the JSIR pathway emits, `null` and `undefined` SHALL occur only:

1. at outbound boundary positions, as the representation §5.3 selected;
2. at inbound boundary positions, as values a narrowing or absence check reads;
3. at `Option` erasure sites the pathway selected under the proof discipline of [Option Operations Representation](option-operations-representation.md); and
4. within foreign code itself, which this specification does not govern.

No other emitted site SHALL produce or test `null` or `undefined`. This is the artifact-level form of the §1.1 invariant: interior Clef code compiles to JavaScript that neither manufactures nor inspects the host's absence values outside the sites above.

## 9. Normative Requirements

1. **Family invariant**: A JavaScript value SHALL enter Clef only through a declared boundary function; `null` and `undefined` SHALL NOT be representable in interior Clef code.
2. **No subsumption**: `JsValue` and `JsRef<'T>` SHALL NOT participate in any subtyping relationship; no implicit conversion SHALL produce or consume either type.
3. **Explicit injection**: A foreign type SHALL be introduced only at positions declared by a binding (§3.1).
4. **Narrowing-only elimination**: The only elimination of `JsValue` SHALL be narrowing; narrowing SHALL be total, SHALL be generated from the declared shape, and SHALL return `Result` with the failed premise identified.
5. **No foreign operations**: Neither foreign type SHALL admit property access, invocation, arithmetic, or comparison.
6. **Callback narrowing**: A callback invoked by foreign code SHALL have its declared parameter types established by narrowing before its body executes.
7. **Boundary grade**: An implementation SHALL compute the boundary grade of every function; a project SHALL be able to require grade zero outside a designated interop layer, with violations diagnosable.
8. **Inbound absence**: The default inbound conversion for `Option<'T>` SHALL map absent, `undefined`, and `null` to `None`; a declared-distinction site SHALL receive the generated three-case union.
9. **Outbound absence**: Each outbound optional position SHALL have one representation (omitted, `null`, or `undefined`) selected at generation time from the declared surface.
10. **Exception interception**: Host throws and rejections SHALL be intercepted at the boundary and surfaced as `Result` errors; no host exception SHALL propagate into Clef code.
11. **Declared-surface mapping**: A binding generator SHALL apply the mapping of §7.1, with the declaration as the default authority and documented overrides only.
12. **Erased-union confinement**: An erased union SHALL NOT enter the native type universe; its introduction and elimination SHALL be generated binding constructs.
13. **Artifact confinement**: `null` and `undefined` SHALL occur in the emitted artifact only at the sites §8 enumerates.
14. **Profile scope**: The requirements of this chapter bind only implementations claiming the JavaScript Substrate profile ([Conformance §7](conformance.md)).

## 10. References

> *Informative.* The design lineage of the foreign pair, per [Conformance §2](conformance.md), imposes no requirement.

- Abadi, M., Cardelli, L., Pierce, B., Plotkin, G. (1991). Dynamic typing in a statically typed language. *TOPLAS 13(2)*. The founding form of a dynamic type without subsumption: explicit injection, `typecase` elimination.
- Matthews, J., Findler, R. B. (2007). Operational semantics for multi-language programs. *POPL*. The lump embedding (`JsRef<'T>`) and the natural embedding (narrowing) as the two disciplined crossings.
- Findler, R. B., Felleisen, M. (2002). Contracts for higher-order functions. *ICFP*; Wadler, P., Findler, R. B. (2009). Well-typed programs can't be blamed. *ESOP*. The contract discipline governing higher-order crossings (§3.3).

## 11. Related Chapters

- [FFI Boundary Semantics](ffi-boundary.md), the C counterpart of this chapter
- [Backend Lowering Architecture](backend-lowering-architecture.md), the JSIR pathway and carrier realization
- [Option Operations Representation](option-operations-representation.md), interior `Option` realization on the JSIR pathway
- [Discriminated Union Representation](discriminated-union-representation.md), the JSIR-pathway union realization and the wire contract
- [Native Type Mappings](native-type-mappings.md), the exclusion of `obj`
- [Conformance](conformance.md), profiles and the preservation obligation
