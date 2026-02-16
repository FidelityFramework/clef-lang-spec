---
title: "Name Resolution"
weight: 9060
draft: true
---

# Name Resolution

This chapter describes how Clef resolves names in various contexts during type inference. This is a component of the overall [Inference Procedures](inference-procedures.md).

The following sections describe how F# resolves names in various contexts.

### Name Environments

Each point in the interpretation of an F# program is subject to an environment. The environment
encompasses:

- All referenced source packages and native libraries.
- _ModulesAndNamespaces_ : a table that maps `long-ident`s to a list of signatures. Each signature is
    either a namespace declaration group signature or a module signature.

    For example, `Collections` may map to one namespace declaration group signature for
    each referenced package that contributes to the `Collections` namespace, and to a
    module signature, if a module called `Collections` is declared or in a referenced
    package.

    If the program references multiple packages, the packages are added to the name resolution
environment in the order in which the references appear in the project file. The order is
important only if ambiguities occur in referencing the contents of packages—for example, if
two packages define the type `MyNamespace.C`.

- _ExprItems_ : a table that maps names to the following items:
    - A value
    - A union case for use when constructing data
    - An active pattern result tag for use when returning results from active patterns
    - A type name for each class or struct type
- _FieldLabels_ : a table that maps names to sets of field references for record types
- _PatItems_ : a table that maps names to the following items:
    - A union case, for use when pattern matching on data
    - An active pattern case name, for use when specifying active patterns
    - A literal definition
- _Types_ : a table that maps names to type definitions. Two queries are supported on this table:
    - Find a type by name alone. This query may return multiple types. For example, a type name like `Map` may resolve to multiple generic types with different arities.

    - Find a type by name and generic arity `n`. This query returns at most one type. For example, the resolution of `Map` with `n = 2` returns a single type `Map<'Key, 'Value>`.
- _ExtensionsInScope_ : a table that maps type names to one or more member definitions

The dot notation is resolved during type checking by consulting these tables.

### Name Resolution in Module and Namespace Paths

Given an input `long-ident` and environment `env`, _Name Resolution in Module and Namespace Paths_
computes the result of interpreting `long-ident` as a module or namespace. The procedure returns a
list of modules and namespace declaration groups.

_Name Resolution in Module and Namespace Paths_ proceeds through the following steps:

1. Consult the _ModulesAndNamespaces_ table to resolve the `long-ident` prefix to a list of modules
    and namespace declaration group signatures.
2. If any identifiers remain unresolved, recursively consult the declared modules and sub-modules
    of these namespace declaration groups.
3. Concatenate all the results.

If the `long-ident` starts with the special pseudo-identifier keyword `global`, the identifier is resolved
by consulting the _ModulesAndNamespaces_ table and ignoring all `open` directives, including those
implied by `AutoOpen` attributes.

For example, if the environment contains two referenced packages, and each package has namespace
declaration groups for the namespaces `Fidelity`, `Fidelity.Collections`, and
`Fidelity.Collections.Concurrent`, _Name Resolution in Module and Namespace Paths_ for
`Fidelity.Collections` returns the two namespace declaration groups named `Fidelity.Collections`, one
from each package.

### Opening Modules and Namespace Declaration Groups

When a module or namespace declaration group `F` is opened, the compiler adds items to the name
environment as follows:

1. Add each exception label for each exception type definition ([§](type-definitions.md#exception-definitions)) in `F` to the _ExprItems_ and
    _PatItems_ tables in the original order of declaration in `F`.
2. Add each type definition in the original order of declaration in `F`. Adding a type definition
    involves the following procedure:
    - If the type is a class or struct type (or an abbreviation of such a type), add the type name to
    the _ExprItems_ table.
    - If the type definition is a record, add the record field labels to the _FieldLabels_ table, unless
    the type has the `RequireQualifiedAccess` attribute.
    - If the type is a union, add the union cases to the _ExprItems_ and _PatItems_ tables, unless the
    type has the `RequireQualifiedAccess` attribute.
    - Add the type to the _TypeNames_ table. For generic types such as `List<'T>`, add an entry under
    `List` that records the generic arity.

3. Add each value in the original order of declaration in `F` , as follows:

    - Add the value to the _ExprItems_ table.
    - If any value is an active pattern, add the tags of that active pattern to the _PatItems_ table
    according to the original order of declaration.
    - If the value is a literal, add it to the _PatItems_ table.
4. Add the member contents of each type extension in `Fi` to the _ExtensionsInScope_ table according
    to the original order of declaration in `Fi`.
5. Add each sub-module or sub-namespace declaration group in `Fi` to the _ModulesAndNamespaces_
    table according to the original order of declaration in `Fi`.
6. Open any sub-modules that are marked with the `FSharp.Core.AutoOpen` attribute.

### Name Resolution in Expressions

Given an input `long-ident` , environment `env` , and an optional count `n` of the number of subsequent
type arguments `<_, ..., _>`, _Name Resolution in Expressions_ computes a result that contains the
interpretation of the `long-ident` `<_, ..., _>` prefix as a value or other expression item, and a residue
path `rest`.

How Name Resolution in Expressions proceeds depends on whether `long-ident` is a single identifier
or is composed of more than one identifier.

If `long-ident` is a single identifier `ident`:

1. Look up `ident` in the _ExprItems_ table. Return the result and empty `rest`.
2. If `ident` does not appear in the _ExprItems_ table, look it up in the _Types_ table, with generic arity
    that matches `n` if available. Return this type and empty `rest`.
3. If `ident` does not appear in either the _ExprItems_ table or the _Types_ table, fail.

If `long-ident` is composed of more than one identifier `ident.rest`, _Name Resolution in Expressions_
proceeds as follows:

1. If `ident` exists as a value in the _ExprItems_ table, return the result, with `rest` as the residue.
2. If `ident` does not exist as a value in the _ExprItems_ table, perform a backtracking search as
    follows:

    - Consider each division of `long-ident` into `[namespace-or-module-path].ident[.rest]`, in
    which the `namespace-or-module-path` becomes successively longer.
    - For each such division, consider each module signature or namespace declaration group
    signature `F` in the list that is produced by resolving `namespace-or-module-path` by using Name
    _Resolution in Module and Namespace Paths_.
    - For each such `F` , attempt to resolve `ident[.rest]` in the following order. If any resolution
    succeeds, then terminate the search:
        - A value in `F`. Return this item and `rest`.
        - A union case in `F`. Return this item and `rest`.
        - An exception constructor in `F`. Return this item and `rest`.
        - A type in `F`. If `rest` is empty, then return this type; if not, resolve using _Name
        Resolution for Members_.
        - A [sub-]module in `F`. Recursively resolve `rest` against the contents of this module.
3. If steps 1 and 2 do not resolve `long-ident`, look up `ident` in the _Types_ table.

    - If the generic arity `n` is available, then look for a type that matches both `ident` and `n`.
    - If no generic arity `n` is available, and `rest` is not empty:
        - If the _Types_ table contains a type `ident` that does not have generic arguments,
        resolve to this type.
        - If the _Types_ table contains a unique type `ident` that has generic arguments, resolve
        to this type. However, if the overall result of the _Name Resolution in Expressions_
        operation is a member, and the generic arguments do not appear in either the
        return or argument types of the item, warn that the generic arguments cannot be
        inferred from the type of the item.
        - If neither of the preceding steps resolves the type, give an error.
    - If rest is empty, return the type, otherwise resolve using _Name Resolution for Members_.
4. If steps 1-3 do not resolve `long-ident`, look up `ident` in the _ExprItems_ table and return the result
    and residue `rest`.
5. Otherwise, if `ident` is a symbolic operator name, resolve to an item that indicates an implicitly
    resolved symbolic operator.
6. Otherwise, fail.

If the expression contains ambiguities, _Name Resolution in Expressions_ returns the first result that
the process generates. For example, consider the following cases:

```fsharp
module M =
    type C =
        | C of string
        | D of string
        member x.Prop1 = 3
    type Data =
        | C of string
        | E
        member x.Prop1 = 3
        member x.Prop2 = 3
    let C = 5
    open M
    let C = 4
    let D = 6

    let test1 = C               // resolves to the value C
    let test2 = C.ToString()    // resolves to the value C with residue ToString
    let test3 = M.C             // resolves to the value M.C
    let test4 = M.Data.C        // resolves to the union case M.Data.C
    let test5 = M.C.C           // error: first part resolves to the value M.C,
                                // and this contains no field or property "C"
    let test6 = C.Prop1         // error: the value C does not have a property Prop
    let test7 = M.E.Prop2       // resolves to M.E, and then a property lookup
```

The following example shows the resolution behavior for type lookups that are ambiguous by
generic arity:

```fsharp
module M =
    type C<'T>() =
        static member P = 1

    type C<'T,'U>() =
        static member P = 1

    let _ = new M.C() // gives an error
    let _ = new M.C<int>() // no error, resolves to C<'T>
    let _ = M.C() // gives an error
    let _ = M.C<int>() // no error, resolves to C<'T>
    let _ = M.C<int,int>() // no error, resolves to C<'T,'U>
    let _ = M.C<_>() // no error, resolves to C<'T>
    let _ = M.C<_,_>() // no error, resolves to C<'T,'U>
    let _ = M.C.P // gives an error
    let _ = M.C<_>.P // no error, resolves to C<'T>
    let _ = M.C<_,_>.P // no error, resolves to C<'T,'U>
```

The following example shows how the resolution behavior differs slightly if one of the types has no generic arguments.

```fsharp
module M =
    type C() =
        static member P = 1

    type C<'T>() =
        static member P = 1

    let _ = new M.C()       // no error, resolves to C
    let _ = new M.C<int>()  // no error, resolves to C<'T>
    let _ = M.C()           // no error, resolves to C
    let _ = M.C< >()        // no error, resolves to C
    let _ = M.C<int>()      // no error, resolves to C<'T>
    let _ = M.C< >()        // no error, resolves to C
    let _ = M.C<_>()        // no error, resolves to C<'T>
    let _ = M.C.P           // no error, resolves to C
let _ = M.C< >.P            // no error, resolves to C
    let _ = M.C<_>.P        // no error, resolves to C<'T>
```

In the following example, the procedure issues a warning for an incomplete type. In this case, the
type parameter `'T` cannot be inferred from the use `M.C.P`, because `'T` does not appear at all in the
type of the resolved element `M.C<'T>.P`.

```fsharp
module M =
    type C<'T>() =
        static member P = 1

    let _ = M.C.P // no error, resolves to C<'T>.P, warning given
```

The effect of these rules is to prefer value names over module names for single identifiers. For
example, consider this case:

```fsharp
let Foo = 1

module Foo =
    let ABC = 2
let x1 = Foo // evaluates to 1
```

The rules, however, prefer type names over value names for single identifiers, because type names
appear in the _ExprItems_ table. For example, consider this case:

```fsharp
let Foo = 1
type Foo() =
    static member ABC = 2
let x1 = Foo.ABC // evaluates to 2
let x2 = Foo() // evaluates to a new Foo()
```

### Name Resolution for Members

_Name Resolution for Members_ is a sub-procedure used to resolve `.member-ident[.rest]` to a
member, in the context of a particular type `type`.

_Name Resolution for Members_ proceeds through the following steps:

1. Search the hierarchy of the type from its root base type to `type`.

   > **Clef Note**: Clef classes do not inherit from `System.Object`. The type hierarchy is determined by explicit `inherit` declarations.

2. At each type, try to resolve `member-ident` to one of the following, in order:

    - A union case of `type`.
    - A property group of `type`.
    - A method group of `type`.
    - A field of `type`.
    - An event of `type`.
    - A property group of extension members of `type`, by consulting the _ExtensionsInScope_ table.
    - A method group of extension members of `type`, by consulting the _ExtensionsInScope_ table.
    - A nested type `type-nested` of `type`. Recursively resolve `.rest` if it is present, otherwise return
    `type-nested`.

3. At any type, the existence of a property, event, field, or union case named `member-ident` causes
    any methods or other entities of that same name from base types to be hidden.
4. Combine method groups with method groups from base types. For example:

```fsharp
type A() =
    member this.Foo(i : int) = 0

type B() =
    inherit A()
    member this.Foo(s : string) = 1

let b = new B()
b.Foo(1)        // resolves to method in A
b.Foo("abc")    // resolves to method in B
```

### Name Resolution in Patterns

_Name Resolution for Patterns_ is used to resolve `long-ident` in the context of pattern expressions.
The `long-ident` must resolve to a union case, exception label, literal value, or active pattern case
name. If it does not, the `long-ident` may represent a new variable definition in the pattern.

_Name Resolution for Patterns_ follows the same steps to resolve the `member-ident` as _Name
Resolution in Expressions_ ([§](inference-name-resolution.md#name-resolution-in-expressions)) except that it consults the _PatItems_ table instead of the _ExprItems_
table. As a result, values are not present in the namespace that is used to resolve identifiers in
patterns. For example:

```fsharp
let C = 3
match 4 with
| C -> sprintf "matched, C = %d" C
| _ -> sprintf "no match, C = %d" C
```

results in `"matched, C = 4"`, because `C` is _not_ present in the _PatItems_ table, and hence becomes a
value pattern. In contrast,

```fsharp
[<Literal>]
let C = 3

match 4 with
| C -> sprintf "matched, C = %d" C
| _ -> sprintf "no match, C = %d" C
```

results in `"no match, C = 3"`, because `C` is a literal and therefore _is_ present in the _PatItems_ table.

### Name Resolution for Types

_Name Resolution for Types_ is used to resolve `long-ident` in the context of a syntactic type. A generic
arity that matches `n` is always available. The result is a type definition and a possible residue `rest`.

_Name Resolution for Types_ proceeds through the following steps:

1. Given `ident[.rest]`, look up `ident` in the _Types_ table, with generic arity `n`. Return the result and
    residue `rest`.
2. If `ident` is not present in the _Types_ table:

    - Divide `long-ident` into `[namespace-or-module-path].ident[.rest]`, in which the `namespace-
    or-module-path` becomes successively longer.
    - For each such division, consider each module and namespace declaration group `F` in the list
    that results from resolving `namespace-or-module-path` by using _Name Resolution in Module
    and Namespace Paths_ ([§](inference-name-resolution.md#name-resolution-in-module-and-namespace-paths)).
    - For each such `F` , attempt to resolve `ident[.rest]` in the following order. Terminate the
    search when the expression is successfully resolved.
        1) A type in `F`. Return this type and residue `rest`.
        2) A [sub-]module in `F`. Recursively resolve `rest` against the contents of this module.

In the following example, the name `C` on the last line resolves to the named type `M.C<_,_>` because `C`
is applied to two type arguments:

```fsharp
module M =
    type C<'T, 'U> = 'T * 'T * 'U

module N =
    type C<'T> = 'T * 'T

open M
open N

let x : C<int, string> = (1, 1, "abc")
```

### Name Resolution for Type Variables

Whenever the F# compiler processes syntactic types and expressions, it assumes a context that
maps identifiers to inference type variables. This mapping ensures that multiple uses of the same
type variable name map to the same type inference variable. For example, consider the following
function:

```fsharp
let f x y = (x:'T), (y:'T)
```

In this case, the compiler assigns the identifiers `x` and `y` the same static type - that is, the same type
inference variable is associated with the name `'T`. The full inferred type of the function is:

```fsharp
val f<'T> : 'T -> 'T -> 'T * 'T
```

The map is used throughout the processing of expressions and types in a left-to-right order. It is
initially empty for any member or any other top-level construct that contains expressions and types.
Entries are eliminated from the map after they are generalized. As a result, the following code
checks correctly:

```fsharp
let f () =
    let g1 (x:'T) = x
    let g2 (y:'T) = (y:string)
    g1 3, g1 "3", g2 "4"
```

The compiler generalizes `g1`, which is applied to both integer and string types. The type variable `'T` in
`(y:'T)` on the third line refers to a different type inference variable, which is eventually constrained to
be type `string`.

### Field Label Resolution

_Field Label Resolution_ specifies how to resolve identifiers such as `field1` in `{field1 = expr; ... fieldN = expr}`.

> **Clef Extension**: Unlike standard F# where memory layout is delegated to the CLR runtime, Clef computes deterministic memory layouts at compile time. Field Label Resolution therefore encompasses both type resolution AND layout determination. See [§](native-type-universe.md#32-records-named-products) for memory layout principles.

_Field Label Resolution_ proceeds through the following steps:

#### Step 1: Candidate Set Construction

For each `field-label_i` in the record expression:

1. If `field-label_i` is a single identifier `fld` AND the initial type is known to be a record type `R<_, ..., _>` that has field `F_i` with name `fld`, then `field-label_i` resolves to `F_i` directly.

2. Otherwise, look up `field-label_i` in the _FieldLabels_ table. This yields a set of field references `FSet_i`, where each reference identifies a field in some record type. The corresponding set of record types is `RSet_i`.

#### Step 2: Type Resolution via Intersection

Compute the intersection of all candidate record type sets:

```
R_candidates = RSet_1 ∩ RSet_2 ∩ ... ∩ RSet_n
```

The resolution proceeds based on the cardinality of `R_candidates`:

| Cardinality | Result |
|-------------|--------|
| 0 | Error FS8704: "No single record type contains all specified fields" |
| 1 | Success: The unique record type `R` is identified |
| > 1 | Error FS8702: "Ambiguous record type. Could be: {types}. Use type annotation to disambiguate." |

#### Step 3: Completeness Verification

For the resolved record type `R`, verify that every field defined in `R` has exactly one corresponding `field-label_i` in the expression. Missing fields result in error FS8705.

#### Step 4: Layout Computation (Clef Extension)

> **Core Principle**: "Field order determines memory layout" ([§](native-type-universe.md#32-records-named-products)). The compiler controls layout—not MLIR, not LLVM.

For the resolved record type `R` with fields `f_1, f_2, ..., f_n` in declaration order:

1. Initialize `offset = 0`, `max_align = 1`
2. For each field `f_i` with type `T_i`:
   - Compute `(size_i, align_i) = layoutOf(T_i)`
   - Compute padding: `pad = (align_i - (offset mod align_i)) mod align_i`
   - Set `offset_i = offset + pad`
   - Update `offset = offset_i + size_i`
   - Update `max_align = max(max_align, align_i)`
3. Compute final padding for struct alignment: `final_pad = (max_align - (offset mod max_align)) mod max_align`
4. Total layout: `TypeLayout.Inline(offset + final_pad, max_align)`

The resolved record type carries this computed layout, ensuring deterministic memory representation throughout the compilation pipeline.

#### Step 5: Return Resolution

Return the resolved record type `R` with:
- Type constructor reference (including computed layout)
- Field types in declaration order
- Memory offsets for each field

#### Error Codes

| Code | Condition |
|------|-----------|
| FS8701 | Field name not found in any record type in scope |
| FS8702 | Multiple record types contain all specified fields (ambiguity) |
| FS8703 | Record type lookup failed (internal error) |
| FS8704 | No single record type contains all specified fields |
| FS8705 | Record expression is incomplete (missing required fields) |

