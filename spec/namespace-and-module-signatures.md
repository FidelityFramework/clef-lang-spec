---
title: "Namespace and Module Signatures"
weight: 230
---

A _module signature_ specifies the functionality exposed by a module: the public values, types, and
nested modules. Signatures hide implementation details.

> **NORMATIVE**: Clef has no separate signature files. There is no `.fsi` analog; CCS does not
> recognize a distinct interface artifact, and no file extension is reserved for one. A module's
> public surface is described **in the same source file as its implementation**, using an inline
> `signature ... end` block placed immediately before the `module` declaration it constrains.

Because Clef is closely related to F#, this position is stated explicitly rather than left to
inference. F# inherited separate signature files (`.fsi`) from the ML tradition, in which a
separately compiled, separately reified interface is a load-bearing part of the compilation model.
Clef's model removes both preconditions that justify a separate interface artifact:

- **The source is always available.** CCS computes compilation order from a whole-program syntactic
  dependency analysis (see [§](program-structure.md#compilation-pipeline)); it never checks a client
  against an interface in the absence of the corresponding implementation. The "compilation firewall"
  that a C header or an ML `.mli` provides is not needed.
- **Nothing is separately reified.** Clef compiles directly to native binaries with no assembly
  metadata (see [§](program-structure-and-execution.md)). There is no metadata artifact for an
  interface file to shape, so a separate interface would merely restate information the compiler
  already holds in full.

What is genuinely valuable in the ML signature tradition — *information hiding* (exposing a type
abstractly while concealing its representation) and a *readable, reviewable statement of a module's
public surface* — is retained by the inline `signature ... end` form, without a second file. A
separate per-module artifact would impose maintenance overhead disproportionate to any guarantee it
adds: every change to a module's surface would require editing two locations that the compiler can
already reconcile from one. Where Clef does require an explicit cross-boundary contract that the
compiler cannot infer — for instance a BAREWire descriptor for cross-processor memory layout — that
contract is written once, at the boundary where it is load-bearing, rather than mirrored across every
module interface.

> **Note**: This position was settled during the 2026 specification audit informed by F#'s
> `--file-order-auto` work (the Costanich contribution). That work established that signature/
> implementation pairs add non-trivial cost to dependency analysis: the export map must coalesce each
> pair and redirect references from the signature to the implementation. Languages without separate
> signature files avoid both steps. Clef aligns with mature ML-family practice that pairs
> order-independent compilation with a single source of truth per module. The grammar of inline
> signatures and the abstraction discipline they encode are given below; see also
> [§](#design-discipline-and-prior-art).

```fsgrammar
signature-decl :=
    signature long-ident~opt = signature-body   -- inline module signature, placed
                                                 -- immediately before the module it constrains

signature-body :=
    begin module-signature-element ... module-signature-element end

module-signature-element :=
    val mutable~opt curried-sig     -- value signature
    val value-defn                  -- literal value signature
    type type-signatures            -- type signature(s) revealing representation
    type type-name                  -- abstract (opaque) type signature; see below
    exception exception-signature   -- exception signature
    signature-decl                  -- nested submodule signature
    module-abbrev                   -- local alias for a module
    import-decl                     -- locally import contents of a module

type-signature :=
    abbrev-type-signature
    record-type-signature
    union-type-signature
    anon-type-signature
    class-type-signature
    struct-type-signature
    interface-type-signature
    enum-type-signature
    delegate-type-signature
    type-extension-signature

type-signatures := type-signature ... and ... type-signature

type-signature-element :=
    attributesopt access~opt new : uncurried-sig        -- constructor signature
    attributesopt member acces~opt member-sig           -- member signature
    attributesopt abstract access~opt member-sig        -- member signature
    attributesopt override member-sig                   -- member signature
    attributesopt default member-sig                    -- member signature
    attributes~opt static member access~opt member-sig  -- static member signature
    interface type                                      -- interface signature

abbrev-type-signature := type-name '=' type

union-type-signature := type-name '=' union-type-cases type-extension-elements-
    signature~opt

record-type-signature := type-name '=' '{' record-fields '}' type-extension-
    elements-signature~opt

anon-type-signature := type-name '=' begin type-elements-signature end

class-type-signature := type-name '=' class type-elements-signature end

struct-type-signature := type-name '=' struct type-elements-signature end

interface-type-signature := type-name '=' interface type-elements-signature end

enum-type-signature := type-name ' =' enum-type-cases

delegate-type-signature := type-name ' =' delegate-sig

type-extension-signature := type-name type-extension-elements-signature

type-extension-elements-signature := with type-elements-signature end
```

The `begin` and `end` tokens are optional when lightweight syntax is used.

### Abstract Type Signatures

A type signature that gives a representation — `type T = { ... }`, `type T = union-cases`, or
`type T = ty` — reveals that representation to clients. A type signature that gives **only the type
name**, with no `=` and no representation, exposes the type _abstractly_:

```fsharp
signature
    type Handle                       -- abstract: name visible, representation hidden
    val create : unit -> Handle
    val close  : Handle -> unit
end
module Resource =
    type Handle = { fd: int; generation: int }
    let create () = ...
    let close h = ...
```

Clients of `Resource` may name `Handle`, hold values of it, and pass them to `create` and `close`,
but cannot observe or depend on its representation (`fd`, `generation`). This is the single most
important capability the signature mechanism provides, and it is fully expressible inline: opacity is
a property of *what a type signature reveals*, not of whether the signature lives in a separate file.

The all-or-none rules in [§](#signature-conformance) govern partial revelation: a record or union
type is either fully abstract (name only) or fully revealed (all fields or cases, in declaration
order); it cannot expose a subset.

Signature declarations, like module declarations, are processed in dependency order computed by CCS
(see [§](program-structure.md#compilation-pipeline)). The following inline signature is well-formed
regardless of the textual order of the two namespaces:

```fsharp
namespace Utilities.Part1

module Module1 =
    val x : Utilities.Part2.StorageCache

namespace Utilities.Part2

type StorageCache =
    new : unit -> unit
```

## Signature Elements

A namespace or module signature declares one or more _value signatures_ and one or more _type
definition signatures_. A type definition signature may include one or more _member signatures_ , in
addition to other elements of type definitions that are specified in the signature grammar at the
start of this chapter.

### Value Signatures

A value signature indicates that a value exists in the implementation. For example, in the signature
of a module, the following declares two value signatures:

```fsharp
module MyMap =
    val mapForward : index1: int * index2: int -> string
    val mapBackward : name: string -> (int * int)
```

The corresponding implementation file might contain the following implementation:

```fsharp
module MyMap =
    let mapForward (index1:int, index2:int) = string index1 + "," + string index2
    let mapBackward (name:string) = (0, 0)
```

### Type Definition and Member Signatures

A type definition signature indicates that a corresponding type definition appears in the
implementation. For example, in an interface type, the following declares a type definition signature
for `Forward` and `Backward`:

```fsharp
type IMap =
    interface
        abstract Forward : index1: int * index2: int -> string
        abstract Backward : name: string -> (int * int)
    end
```

A member signature indicates that a corresponding member appears on the corresponding type
definition in the implementation. Member specifications must specify argument and return types,
and can optionally specify names and attributes for parameters.

For example, the following declares a type definition signature for a type with one constructor
member, one property member `Kind` and one method member `Purr`:

```fsharp
type Cat =
    new : kind:string -> Cat
    member Kind : string
    member Purr : unit -> Cat
```

The corresponding implementation file might contain the following implementation:

```fsharp
type Cat(kind: string) =
    member x.Meow() = printfn "meow"
    member x.Purr() = printfn "purr"
    member x.Kind = kind
```

## Signature Conformance

Values, types, and members that are present in implementations can be omitted in signatures, with
the following exceptions:

- Type abbreviations may not be hidden by signatures. That is, a type abbreviation `type T = ty` in
    an implementation does not match type `T` (without an abbreviation) in a signature.
- Any type that is represented as a record or union must reveal either all or none of its fields or
    cases, in the same order as that specified in the implementation. Types that are represented as
    classes may reveal some, all, or none of their fields in a signature.
- Any type that is revealed to be an interface, or a type that is a class or struct with one or more
    constructors may not hide its `inherit` declaration, abstract dispatch slot declarations, or abstract
    interface declarations.

> Note: This section does not yet document all checks made by the F# 3.1 language
implementation.

### Signature Conformance for Functions and Values

If both a signature and an implementation contain a function or value definition with a given name,
the signature and implementation must conform as follows:

- The declared accessibilities, `inline`, and `mutable` modifiers must be identical in both the
    signature and the implementation.
- If either the signature or the implementation has the `[<Literal>]` attribute, both must have this
    attribute. Furthermore, the declared literal values must be identical.
- The number of generic parameters, both inferred and explicit, must be identical.
- The types and type constraints must be identical up to renaming of inferred and/or explicit
    generic parameters. For example, assume a signature is written `val head : seq<'T> -> 'T` and
    the compiler could infer the type `val head : seq<'a> -> 'a` from the implementation. These
    are considered identical up to renaming the generic parameters.
- The arities must match, as described in the next section.

#### Arity Conformance for Functions and Values

Arities of functions and values must conform between implementation and signature. Arities of
values are implicit in module signatures. A signature that contains the following results in the arity
`[A1 ... An]` for `F`:

```fsgrammar
val F : ty11 * ... * ty1A1 - > ... -> tyn1 * ... * tynAn - > rty
```

Arities in a signature must be equal to or shorter than the corresponding arities in an
implementation, and the prefix must match. This means that F# makes a deliberate distinction
between the following two signatures:

```fsharp
val F: int -> int
```

and

```fsharp
val F: (int -> int)
```

The parentheses indicate a top-level function, which might be a first-class computed expression that
computes to a function value, rather than a compile-time function value.

The first signature can be satisfied only by a true function; that is, the implementation must be a
lambda value as in the following:

```fsharp
let F x = x + 1
```

> Note: Because arity inference also permits right-hand-side function expressions, the
implementation may currently also be:
<br>`let F = fun x -> x + 1`

The second signature

```fsharp
val F: (int -> int)
```

can be satisfied by any value of the appropriate type. For example:

```fsharp
let f =
    let mutable cache = Map.empty<int,int>
    fun x ->
        match Map.tryFind x cache with
        | Some v -> v
        | None ->
            let res = x * x
            cache <- Map.add x res cache
            res
```

or

```fsharp
let f = fun x -> x + 1
```

or

```fsharp
// throw an exception as soon as the module initialization is triggered
let f : int -> int = failwith "failure"
```

For both the first and second signatures, you can still use the functions as first-class function values
from client code; the parentheses simply act as a constraint on the implementation of the value.

The reason for this interpretation of types in value and member signatures is that function arity affects compilation. F# functions with known arity compile to direct function calls, while function values require closure allocation. Signatures must contain enough information to reveal the desired arity for efficient native code generation.

> **Clef Note**: In native compilation, arity information enables direct function calls without closure overhead. Parenthesized signatures indicate that the implementation must be callable with that arity, enabling the compiler to generate efficient native calling conventions.

#### Signature Conformance for Type Functions

If a value is a type function, then its corresponding value signature must have explicit type
arguments. For example, the implementation

```fsharp
let empty<'T> : list<'T> = printfn "hello"; []
```

conforms to this signature:

```fsharp
val empty<'T> : list<'T>
```

but not to this signature:

```fsharp
val empty : list<'T>
```

The reason for this rule is that the second signature indicates that the value is, by default,
generalizable ([§](inference-constraint-solving.md#generalization)).

### Signature Conformance for Members

If both a signature and an implementation contain a member with a given name, the signature and
implementation must conform as follows:

- If one is an extension member, both must be extension members.
- If one is a constructor, then both must be constructors.
- If one is a property, then both must be properties.

- The types must be identical up to renaming of inferred or explicit type parameters (as for
    functions and values).
- The `static`, `abstract`, and `override` qualifiers must match precisely.
- Abstract members must be present in the signature if a representation is given for a type.

> Note: This section does not yet document all checks made by the F# 3 .1 language
implementation.

## Design Discipline and Prior Art

Clef's inline-signature design is not a novelty; it recombines well-established positions from the
languages in its lineage, keeping the capabilities that earn their place and declining the artifact
whose justification Clef's compilation model removes.

- **OCaml (`.mli`) and the ML signature tradition.** The defining contribution of ML signatures is
  the _abstract type_: a signature may expose `type t` while concealing its representation, so that
  clients depend only on the operations, never the layout. Clef adopts this capability directly (see
  [§](#abstract-type-signatures)) while declining the separate file. The abstraction is the asset;
  the sidecar artifact is incidental to it.
- **Standard ML signatures and opaque ascription.** SML distinguishes transparent ascription
  (`structure M : S`) from opaque ascription (`structure M :> S`), where opacity is a property of how
  a signature is *applied* to a structure — not of a separate compilation unit. Clef's inline
  `signature ... end` is precisely an ascription on the module that follows it. Opacity is expressed
  per type (reveal the name only) rather than per module, which is finer-grained than whole-module
  sealing and does not require ML's functor machinery to be useful.
- **Modula-2 and Mesa definition modules.** The separate `DEFINITION`/`IMPLEMENTATION` (Modula-2) and
  `DEFINITIONS`/`PROGRAM` (Mesa) split is the historical origin of the two-file interface model. Its
  motivation was separate compilation with cross-module type checking at a time when that was a novel
  capability — the same compilation-model necessity a C header serves, and the same one that Clef's
  whole-program dependency analysis makes unnecessary. The split solved a problem Clef does not have.
- **Scheme (R6RS libraries).** A Scheme `library` form declares its public surface with an in-form
  `export` clause; there is no separate interface artifact. This is direct prior art for the position
  that a module's exported surface belongs *in the module's source*, declared explicitly, rather than
  in a parallel file.

The synthesis: Clef keeps **abstraction** (OCaml/SML opaque types) and **in-source declaration of the
public surface** (Scheme `export`), and declines the **separate interface artifact** (Modula/Mesa,
ML `.mli`, F# `.fsi`) whose rationale — a compilation firewall across separately reified units — does
not apply to a whole-program, directly-native compiler.
