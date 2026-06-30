---
title: "Application Resolution"
weight: 620
category: Compiler
status: normative
---

This chapter describes how Clef resolves application expressions, including function and method applications. This is a component of the overall [Inference Procedures](inference-procedures.md).

## Resolving Application Expressions

Application expressions that use dot notation - such as `x.Y<int>.Z(g).H.I.j` - are resolved
according to a set of rules that take into account the many possible shapes and forms of these
expressions and the ambiguities that may occur during their resolution. This section specifies the
exact algorithmic process that is used to resolve these expressions.

Resolution of application expressions proceeds as follows:

1. Repeatedly decompose the application expression into a leading expression `expr` and a list of
    projections `projs`. Each projection has the following form:
    - `.long-ident-or-op` is a dot lookup projection.
    - `expr` is an application projection.
    - `<types>` is a type application projection.

   For example:
    - `x.y.Z(g).H.I.j` decomposes into `x.y.Z` and projections `(g)`, `.H.I.j`.
    - `x.M<int>(g)` decomposes into `x.M` and projections `<int>`, `(g)`.
    - `f x` decomposes into `f` and projection `x`.

    > Note: In this specification we write sequences of projections by juxtaposition; for
    example, `(expr).long-ident<types>(expr)`. We also write ( `.rest` + `projs` ) to refer to
    adding a residue long identifier to the front of a list of projections, which results in `projs`
    if `rest` is empty and `.rest projs` otherwise.

2. After decomposition:
    - If `expr` is a long identifier expression `long-ident`, apply _Unqualified Lookup_ ([§](inference-application-resolution.md#unqualified-lookup)) on `long-ident` with projections `projs`.
    - If `expr` is not such an expression, check the expression against an arbitrary initial type `ty`, to
       generate an elaborated expression `expr`. Then process `expr`, `ty`, and `projs` by using
       _Expression-Qualified Lookup_ ([§](inference-application-resolution.md#expression-qualified-lookup))

### Unqualified Lookup

Given an input `long-ident` and projections `projs`, _Unqualified Lookup_ computes the result of
“looking up” `long-ident.projs` in an environment `env`. The first part of this process resolves a prefix
of the information in `long-ident.projs`, and recursive resolutions typically use _Expression-Qualified
Resolution_ to resolve the remainder.

For example, _Unqualified Lookup_ is used to resolve the vast majority of identifier references in F#
code, from simple identifiers such as `sin`, to complex accesses such as
`Array.init 10 id |> Array.length`.

_Unqualified Lookup_ proceeds through the following steps:

1. Resolve `long-ident` by using _Name Resolution in Expressions_ ([§](inference-name-resolution.md#name-resolution)). This returns a _name resolution item_ `item` and a _residue long identifier_ `rest`.

For example, the result of _Name Resolution in Expressions_ for `v.X.Y` may be a value reference `v`
along with a residue long identifier `X.Y`. Likewise, `N.X(args).Y` may resolve to an overloaded
method `N.X` and a residue long identifier `Y`.

_Name Resolution in Expressions_ also takes as input the presence and count of subsequent type
arguments in the first projection. If the first projection in `projs` is `<tyargs>`, _Unqualified Lookup_
invokes _Name Resolution in Expressions_ with a known number of type arguments. Otherwise, it
is invoked with an unknown number of type arguments.

2. Apply _Item-Qualified Lookup_ for `item` and (`rest` + `projs`).

### Item-Qualified Lookup

Given an input item `item` and projections `projs`, _Item-Qualified Lookup_ computes the projection
`item.projs`. This computation is often a recursive process: the first resolution uses a prefix of the
information in `item.projs`, and recursive resolutions resolve any remaining projections.

_Item-Qualified Lookup_ proceeds as follows:

1. If `item` is not one of the following, return an error:
    - A named value
    - A unique type
    - A union case
    - A group of named types
    - A group of methods
    - A group of indexer getter properties
    - A single non-indexer getter property
    - A static F# field
    - A static field
    - An implicitly resolved symbolic operator name
2. If the first projection is `<types>`, then we say the resolution has a type application `<types>` with
    remaining projections.
3. Otherwise, checking proceeds as follows.
    - If `item` is a value reference `v` then first
    Instantiate the type scheme of `v`, which results in a type `ty`. Apply these rules:

        - If the first projection is `<types>`, process the types and use the results as the
        arguments to instantiate the type scheme.
        - If the first projection is not `<types>`, the type scheme is _freshly instantiated_.
        - If the value has the `RequiresExplicitTypeArguments` attribute, the first
        projection must be `<types>`.
        - If the value has type `byref<ty2>`, add a byref dereference to the elaborated
        expression.
        - Insert implicit flexibility for the use of the value ([§](inference-application-resolution.md#implicit-insertion-of-flexibility-for-uses-of-functions-and-members)).

        Then Apply _Expression-Qualified Lookup_ for type `ty` and any remaining projections.

    - If `item` is a type name, where `projs` begins with `<types>.long-ident`

        - Process the types and use the results as the arguments to instantiate the named
        type reference, thus generating a type `ty`.
        - Apply _Name Resolution for Members_ to `ty` and `long-ident`, which generates a new
        `item`.
        - Apply _Item-Qualified Lookup_ to the new `item` and any remaining projections.

    - If `item` is a group of type names where `projs` begins with `<types>` or `expr` or `projs` is empty

        - Process the types and use the results as the arguments to instantiate the named
        type reference, thus generating a type `ty`.
        - Process the object construction `ty(expr)` as an object constructor call in the same
        way as `new ty(expr)`. If `projs` is empty then process the object construction `ty` as
        an object constructor call in the same way as `(fun arg -> new ty(arg))`, i.e.
        resolve the object constructor call with no arguments.
        - Apply _Expression-Qualified Lookup_ to `item` and any remaining projections.

    - If `item` is a group of method references

        - Apply _Method Application Resolution_ for the method group. _Method Application Resolution_
          accepts an optional set of type arguments and a syntactic expression
            argument. Determine the arguments based on what `projs` begins with:
            - `<types> expr`, then use `<types>` as the type arguments and `expr` as the
               expression argument.
            - `expr`, then use `expr` as the expression argument.
            - anything else, use no expression argument or type arguments.
        - If the result of _Method Application Resolution_ is labeled with the
            `RequiresExplicitTypeArguments` attribute, then explicit type arguments are
            required.
        - Let `fty` be the actual return type that results from _Method Application Resolution_.
            Apply _Expression-Qualified Lookup_ to `fty` and any remaining projections.

    - If `item` is a group of property indexer references

        - Apply _Method Application Resolution_, and use the underlying getter indexer
        methods for the method group.
        2. Determine the arguments to _Method Application Resolution_ as described for a
        group of methods.

    - If `item` is a static field reference
        - Check the field for accessibility and attributes.
        - Let `fty` be the actual type of the field, taking into account the type `ty` via which the
          field was accessed in the case where this is a field in a generic type.
        - Apply _Expression-Qualified Lookup_ to `fty` and `projs`.

    - If `item` is a [union case tag](discriminated-union-representation.md), exception tag, or active pattern result element tag
        - Check the tag for accessibility and attributes.
        - If `projs` begins with `expr`, use `expr` as the expression argument.
        - Otherwise, use no expression argument or type arguments. In this case, build a
          function expression for the union case.
        - Let `fty` be the actual type of the union case.
        - Apply _Expression-Qualified Lookup_ to `fty` and remaining `projs`.

    - If `item` is a unique type name `C` where `projs` is empty
        - Check the type for accessibility and attributes.
        - Process the types using a new instantiation for `C`, thus generating a type `ty`, and process the object construction `fun v -> new ty(v)` as an object constructor call.

    - If `item` is an event reference

        - Check the event for accessibility and attributes.
        - Let `fty` be the actual type of the event.
        - Apply _Expression-Qualified Lookup_ to `fty` and `projs`.

    - If `item` is an implicitly resolved symbolic operator name `op`

        - If `op` is a unary, binary or the ternary operator ?<-, resolve to the following
        expressions, respectively:

            ```fsharp
            (fun (x:^a) -> (^a : static member (op) : ^a -> ^b) x)
            (fun (x:^a) (y:^b) ->
                ((^a or ^b) : static member (op) : ^a * ^b -> ^c) (x,y))
            (fun (x:^a) (y:^b) (z:^c)
                -> ((^a or ^b or ^c) : static member (op) : ^a * ^b * ^c -> ^d) (x,y,z))
            ```

        - The resulting expressions are static member constraint invocation expressions ([§](expressions.md#member-constraint-invocation-expressions)),
        which enable the default interpretation of operators by using type-directed
        member resolution.
        - Recheck the entire expression with additional subsequent projections `.projs`.

### Expression-Qualified Lookup

Given an elaborated expression `expr` of type `ty`, and projections `projs`, _Expression-Qualified Lookup_
computes the “lookups or applications” for `expr.projs`.

Expression-Qualified Lookup proceeds through the following steps:

1. Inspect `projs` and process according to the following table.

    | `projs` | Action | Comments |
    | --- | --- | --- |
    | Empty | Assert that the type of the overall, original application expression is `ty`. | Checking is complete.|
    | Starts with `(expr2)` | Apply _Function Application Resolution_ ([§](inference-application-resolution.md#function-application-resolution)). | Checking is complete when _Function Application Resolution_ returns. |
    | Starts with `<types>` | Fail. | Type instantiations may not be applied to arbitrary expressions; they can apply only to generic types, generic methods, and generic values. |
    | Starts with `.long-ident` | Resolve `long-ident` using _Name Resolution for Members_ ([§](inference-name-resolution.md#name-resolution-in-expressions))_. Return a name resolution item `item` and a residue long identifier `rest`. Continue processing at step 2. | For example, for `ty = string` and `long-ident = Length`, _Name Resolution for Members_ returns a property reference to the `Length` property of the string type. |

2. If Step 1 returned an `item` and `rest`, report an error if `item` is not one of the following:
    - A group of methods.
    - A group of instance getter property indexers.
    - A single instance, non-indexer getter property.
    - A single instance F# field.
    - A single instance field.
3. Proceed based on `item` as follows:

    - If `item` is a group of methods
        - Apply _Method Application Resolution_ for the method group. _Method Application Resolution_ accepts an optional set of type arguments and a syntactic expression argument. If `projs` begins with:
            - `<types>(arg)`, then use `<types>` as the type arguments and `arg` as
            the expression argument.
            - `(arg)`, then use `arg` as the expression argument.
            - otherwise, use no expression argument or type arguments.

        - Let `fty` be the actual return type resulting from _Method Application Resolution_. Apply _Expression-Qualified Lookup_ to `fty` and any remaining projections.

    - If `item` is a group of indexer properties
        - Apply _Method Application Resolution_ and use the underlying getter indexer methods for the method group.
        - Determine the arguments to _Method Application Resolution_ as described for a group of methods.

    - If `item` is a non-indexer getter property
        - Apply _Method Application Resolution_ for the method group that contains
            only the getter method for the property, with no type arguments and one `()` argument.

    - If `item` is an instance intermediate language (IL) or F# field `F`

        - Check the field for accessibility and attributes.
        - Let `fty` be the actual type of the field (taking into account the type `ty`
          by which the field was accessed).
        - Assert that `ty` is a subtype of the actual containing type of the field.
        - Produce an elaborated form for `expr.F`. If `F` is a field in a value type
          then take the address of `expr` by using the _AddressOf_(`expr, NeverMutates`) operation [§](expressions.md#taking-the-address-of-an-elaborated-expression).
        - Apply _Expression-Qualified Lookup_ to `fty` and `projs`.

## Function Application Resolution

Given expressions `f` and `expr` where `f` has type `ty`, and given subsequent projections `projs`, _Function
Application Resolution_ does the following:

1. Asserts that `f` has type `ty1 -> ty2` for new inference variables `ty1` and `ty2`.
2. If the assertion succeeds:

    - Check `expr` with the initial type `ty1`.
    - Process `projs` using _Expression-Qualified Lookup_ against `ty2`.
3. If the assertion fails, and `expr` has the form `{ computation-expr }`:

    - Check the expression as the [computation expression](expressions.md) form `f { computation-expr }`, giving result type `ty1`.
    - Process `projs` using Expression-Qualified Lookup against `ty1`.

## Method Application Resolution

Given a method group `M`, optional type arguments `<ActualTypeArgs>`, an optional syntactic argument
`obj`, an optional syntactic argument `arg`, and overall initial type `ty`, _Method Application Resolution_
resolves the overloading based on the partial type information that is available. It also:

- Resolves optional and named arguments.
- Resolves “out” arguments.
- Resolves post-hoc property assignments.
- Applies method application resolution.
- Inserts _ad hoc_ conversions that are only applied for method calls.

If no syntactic argument is supplied, _Method Application Resolution_ tries to resolve the use of the
method as a first class value, such as the method call in the following example:

```fsharp
List.map String.length ["hello"; "world"]
```

_Method Application Resolution_ proceeds through the following steps:

1. Restrict the candidate method group `M` to those methods that are _accessible_ from the point of
    resolution.
2. If an argument `arg` is present, determine the sets of _unnamed_ and _named actual arguments_,
    `UnnamedActualArgs` and `NamedActualArgs`:

    - Decompose `arg` into a list of arguments:
        - If `arg` is a syntactic tuple `arg1, ..., argN`, use these arguments.
        - If `arg` is a syntactic unit value `()`, use a zero-length list of arguments.

    - For each argument:
        - If `arg` is a binary expression of the form `name=expr`, it is a named actual argument.
        - Otherwise, `arg` is an unnamed actual argument.

    If there are no named actual arguments, and `M` has only one candidate method, which accepts
    only one required argument, ignore the decomposition of `arg` to tuple form. Instead, `arg` itself is
    the only named actual argument.

    All named arguments must appear after all unnamed arguments.

    Examples:
    - `x.M(1, 2)` has two unnamed actual arguments.
    - `x.M(1, y = 2)` has one unnamed actual argument and one named actual argument.
    - `x.M(1, (y = 2))` has two unnamed actual arguments.
    - `x.M( printfn "hello"; ())` has one unnamed actual argument.
    - `x.M((a, b))` has one unnamed actual argument.
    - `x.M(())` has one unnamed actual argument.

3. Determine the named and unnamed _prospective actual argument types_, called `ActualArgTypes`.
    - If an argument `arg` is present, the prospective actual argument types are fresh type
       inference variables for each unnamed and named actual argument.
       - If the argument has the syntactic form of an address-of expression `&expr` after ignoring
          parentheses around the argument, equate this type with a type `byref<ty>` for a fresh
          type `ty`.
       - If the argument has the syntactic form of a function expression `fun pat1 ... patn -> expr`
          after ignoring parentheses around the argument, equate this type with a
          type `ty1 -> ... tyn -> rty` for fresh types `ty1 ... tyn`.
    - If no argument `arg` is present:

        - If the method group contains a single method, the prospective unnamed argument
        types are one fresh type inference variable for each required, non-“out” parameter that
        the method accepts.
        - If the method group contains more than one method, the expected overall type of the
        expression is asserted to be a function type `dty -> rty`.
            - If `dty` is a tuple type `(dty1 * ... * dtyN)`, the prospective argument types are `(dty1,
            ..., dtyN)`.

            - If `dty` is `unit`, then the prospective argument types are an empty list.

            - If `dty` is any other type, the prospective argument types are `dty` alone.

        - Subsequently:
            - The method application is considered to have one unnamed actual argument for
            each prospective unnamed actual argument type.

            - The method application is considered to have no named actual arguments.

4. For each candidate method in `M`, attempt to produce zero, one, or two _prospective method calls_
    `M~possible` as follows:

    - If the candidate method is generic and has been generalized, generate fresh type inference
    variables for its generic parameters. This results in the `FormalTypeArgs` for `M~possible`.
    - Determine the named and unnamed formal parameters , called `NamedFormalArgs` and
    `UnnamedFormalArgs` respectively, by splitting the formal parameters for `M` into parameters
    that have a matching argument in `NamedActualArgs` and parameters that do not.
    - If the number of `UnnamedFormalArgs` exceeds the number of `UnnamedActualArgs`, then modify
    `UnnamedFormalArgs` as follows:

        - Determine the suffix of `UnnamedFormalArgs` beyond the number of `UnnamedActualArgs`.
        - If all formal parameters in the suffix are `out` arguments with `byref` type, remove the
            suffix from `UnnamedFormalArgs` and call it `ImplicitlyReturnedFormalArgs`.
        - If all formal parameters in the suffix are optional arguments, remove the suffix from
            `UnnamedFormalArgs` and call it `ImplicitlySuppliedFormalArgs`.

    - If the last element of `UnnamedFormalArgs` has the `ParamArray` attribute and type `pty[]` for
    some `pty`, then modify `UnnamedActualArgs` as follows:

        - If the number of `UnnamedActualArgs` exceeds the number of `UnnamedFormalArgs - 1`,
            produce a prospective method call named `ParamArrayActualArgs` that has the excess of
            `UnnamedActualArgs` removed.
        - If the number of `UnnamedActualArgs` equals the number of `UnnamedFormalArgs - 1`, produce
            two prospective method calls:

            - One has an empty `ParamArrayActualArgs`.
            - One has no `ParamArrayActualArgs`.

        - If `ParamArrayActualArgs` has been produced, then `M~possible` is said to use _ParamArray conversion_ with type `pty`.

    - Associate each `name = arg` in `NamedActualArgs` with a target. A target is a _named formal
      parameter_, a _settable return property_, or a _settable return field_ as follows:

        - If one of the arguments in `NamedFormalArgs` has name `name`, that argument is the target.
        - If the return type of `M`, before the application of any type arguments `ActualTypeArgs`,
            contains a settable property `name`, then `name` is the target. The available properties
            include any property extension members of type, found by consulting the
            _ExtensionsInScope_ table.
        - If the return type of `M`, before the application of any type arguments `ActualTypeArgs`,
            contains a settable field `name`, then `name` is the target.

    - No prospective method call is generated if any of the following are true:

        - A named argument cannot be associated with a target.
        - The number of `UnnamedActualArgs` is less than the number of `UnnamedFormalArgs` after
            steps 4 a-e.
        - The number of `ActualTypeArgs`, if any actual type arguments are present, does not
            precisely equal the number of `FormalTypeArgs` for `M`.
        - The candidate method is static and the optional syntactic argument `obj` is present, or
            the candidate method is an instance method and `obj` is not present.

5. Attempt to apply initial types before argument checking. If only one prospective method call
`M~possible` exists, _assert_ `M~possible` by performing the following steps:

    - Verify that each `ActualTypeArgi` is equal to its corresponding `FormalTypeArgi`.
    - Verify that the type of `obj` is a subtype of the containing type of the method `M`.
    - For each `UnnamedActualArgi` and `UnnamedFormalArgi`, verify that the corresponding
    `ActualArgType` coerces to the type of the corresponding argument of `M`.
    - If `M~possible` uses ParamArray conversion with type `pty`, then for each `ParamArrayActualArgi`,
    verify that the corresponding `ActualArgType` coerces to `pty`.
    - For each `NamedActualArgi` that has an associated formal parameter target, verify that the
    corresponding `ActualArgType` coerces to the type of the corresponding argument of `M`.
    - For each `NamedActualArgi` that has an associated property or field setter target, verify that
    the corresponding `ActualArgType` coerces to the type of the property or field.
    - Verify that the prospective formal return type coerces to the expected actual return type. If
    the method `M` has return type `rty`, the formal return type is defined as follows:

        - If the prospective method call contains `ImplicitlyReturnedFormalArgs` with type `ty1, ..., tyN`,
          the formal return type is `rty * ty1 * ... * tyN`. If `rty` is `unit` then the formal
            return type is `ty1 * ... * tyN`.
        - Otherwise the formal return type is `rty`.

6. Check and elaborate argument expressions. If `arg` is present:
    - Check and elaborate each unnamed actual argument expression `argi`. Use the
    corresponding type in `ActualArgTypes` as the initial type.
    - Check and elaborate each named actual argument expression `argi`. Use the corresponding
    type in `ActualArgTypes` as the initial type.
7. Choose a unique `M~possible` according to the following rules:
    - For each `M~possible`, determine whether the method is _applicable_ by attempting to assert
    `M~possible` as described in step 4a). If the actions in step 4a detect an inconsistent constraint set
    ([§](inference-constraint-solving.md#constraint-solving)), the method is not applicable. Regardless, the overall constraint set is left unchanged
    as a result of determining the applicability of each `M~possible`.
    - If a unique applicable `M~possible` exists, choose that method. Otherwise, choose the unique _best_
    `M~possible` by applying the following criteria, in order:
        1) Prefer candidates whose use does not constrain the use of a user-introduced generic
        type annotation to be equal to another type.
        2) Prefer candidates that do not use ParamArray conversion. If two candidates both use
        ParamArray conversion with types `pty1` and `pty2`, and `pty1` feasibly subsumes `pty2`, prefer
        the second; that is, use the candidate that has the more precise type.
        3) Prefer candidates that do not have `ImplicitlyReturnedFormalArgs`.
        4) Prefer candidates that do not have `ImplicitlySuppliedFormalArgs`.
        5) If two candidates have unnamed actual argument types `ty11 ... ty1n` and `ty21 ... ty2n`, and
           each `ty1i` feasibly subsumes `ty2i`, then prefer the second candidate. That is, prefer any
           candidate that has the more specific actual argument types.
        6) Prefer candidates that are not extension members over candidates that are.
        7) To choose between two extension members, prefer the one that results from the most
        recent use of open.
        8) Prefer candidates that are not generic over candidates that are generic - that is, prefer
        candidates that have empty `ActualArgTypes`.

    Report an error if steps 1) through 8) do not result in the selection of a unique better method.

8. Once a unique best `M~possible` is chosen, commit that method.
9. Apply attribute checks.
10. Build the resulting elaborated expression by following these steps:
    - If the type of `obj` is a variable type or a value type, take the address of `obj` by using the
    _AddressOf_`(obj , PossiblyMutates)` operation ([§](expressions.md#taking-the-address-of-an-elaborated-expression)).
    - Build the argument list by:
        - Passing each argument corresponding to an `UnamedFormalArgs` where the argument is an
            optional argument as a `Some` value.
        - Passing a `None` value for each argument that corresponds to an `ImplicitlySuppliedFormalArgs`.
        - Applying coercion to arguments.

    - Bind `ImplicitlyReturnedFormalArgs` arguments by introducing mutable temporaries for each
    argument, passing them as `byref` parameters, and building a tuple from these mutable
    temporaries and any method return value as the overall result.
    - For each `NamedActualArgs` whose target is a settable property or field, assign the value into
    the property.
    - If `arg` is not present, return a function expression that represents a first class function value.

Two additional rules apply when checking arguments (see [§](type-definitions.md#type-directed-conversions-at-member-invocations) for examples):

- If a formal parameter has delegate type `D`, an actual argument `farg` has known type
      `ty1 -> ... -> tyn -> rty`, and the number of arguments of the Invoke method of delegate type
      `D` is precisely `n`, interpret the formal parameter in the same way as the following:
         `new D (fun arg1 ... argn -> farg arg1 ... argn)`.

    For more information on the conversions that are automatically applied to arguments, see
  [§](type-definitions.md#optional-arguments-to-method-members).
  
- If a formal parameter is an `out` parameter of type `byref<ty>`, and an actual argument type is
      not a byref type, interpret the actual parameter in the same way as type `ref<ty>`. That is, an F#
      reference cell can be passed where a `byref<ty>` is expected.

One effect of these additional rules is that a method that is used as a first class function value can
resolve even if a method is overloaded and no further information is available. For example:

```fsharp
let builder = StringBuilder()
let append = builder.Append;;
```

_Method Application Resolution_ results in the following, despite the fact that `StringBuilder.Append` may be overloaded:

```fsharp
val append : string -> StringBuilder
```

The reason is that if the initial type contains no information about the expected number of
arguments, the F# compiler assumes that the method has one argument.

### Additional Propagation of Known Type Information in F# 3.1

In the above descreiption of F# overload resolution, the argument expressions of a call to an
overloaded set of methods

```fsgrammar
callerObjArgTy.Method(callerArgExpr1 , ... callerArgExprN)
```

calling

```fsgrammar
calledObjArgTy.Method(calledArgTy1, ... calledArgTyN)
```

In F# 3.1 and subsequently, immediately prior to checking argument expressions, each argument
position of the unnamed caller arguments for the method call is analysed to propagate type
information extracted from method overloads to the expected types of lambda expressions. The
new rule is applied when

- the candidates are overloaded
- the caller argument at the given unnamed argument position is a syntactic lambda, possible
    parenthesized
- all the corresponding formal called arguments have `calledArgTy` either of

  - function type `calledArgDomainTy1 -> ... -> calledArgDomainTyN -> calledArgRangeTy`
        (after taking into account “function to delegate” adjustments), or
  - some other type which would cause an overload to be discarded

- at least one overload has enough curried lambda arguments for it corresponding expected
    function type

In this case, for each unnamed argument position, then for each overload:

- Attempt to solve `callerObjArgTy = calledObjArgTy` for the overload, if the overload is for an
    instance member. When making this application, only solve type inference variables present in
    the `calledObjArgTy`. If any of these conversions fail, then skip the overload for the purposes of
    this rule
- Attempt to solve `callerArgTy = (calledArgDomainTy1_ -> ... -> calledArgDomainTyN_ -> ?)`. If
    this fails, then skip the overload for the purposes of this rule

### Conditional Compilation of Member Calls

If a member definition has the `Conditional` attribute, then any application of
the member is adjusted as follows:

- The `Conditional("symbol")` attribute may apply to methods only.
- Methods that have the `Conditional` attribute must have return type `unit`. The return type may
    be checked either on use of the method or definition of the method.
- If `symbol` is not in the current set of conditional compilation symbols, the compiler eliminates
    application expressions that resolve to calls to members that have the `Conditional` attribute and
    ensures that arguments are not evaluated. Elimination of such expressions proceeds first with
    static members and then with instance members, as follows:
  - Static members: `Type.M(args)` => `()`
  - Instance members: `expr.M(args)` => `()`

### Implicit Insertion of Flexibility for Uses of Functions and Members

At each use of a data constructor, named function, or member that forms an expression, flexibility is
implicitly added to the expression. This flexibility is associated with the use of the function or
member, according to the inferred type of the expression. The added flexibility allows the item to
accept arguments that are statically known to be subtypes of argument types to a function without
requiring explicit upcasts

The flexibility is added by adjusting each expression `expr` which represents a use of a function or
member as follows:

- The type of the function or member is decomposed to the following form:

    ```fsgrammar
    ty11 * ... * ty1n -> ... -> tym1 * ... * tymn -> rty
    ```

- If the type does not decompose to this form, no flexibility is added.
- The positions `tyij` are called the “parameter positions” for the type. For each parameter position
    where `tyij` is not a sealed type, and is not a variable type, the type is replaced by a fresh type
    variable `ty'ij` with a coercion constraint `ty'ij :> tyij`.
- After the addition of flexibility, the expression elaborates to an expression of type

    ```fsgrammar
    ty'11 * ... * ty'1n -> ... -> ty'm1 * ... * ty'mn -> rty
    ```

    but otherwise is semantically equivalent to `expr` by creating an anonymous function expression
    and inserting appropariate coercions on arguments where necessary.

This means that F# functions whose inferred type includes an unsealed type in argument position
may be passed subtypes when called, without the need for explicit upcasts. For example:

```fsharp
type Base() =
    member b.X = 1

type Derived(i : int) =
    inherit Base()
    member d.Y = i

let d = new Derived(7)

let f (b : Base) = b.X

// Call f: Base -> int with an instance of type Derived
let res = f d

// Use f as a first-class function value of type : Derived -> int
let res2 = (f : Derived -> int)
```

The F# compiler determines whether to insert flexibility after explicit instantiation, but before any
arguments are checked. For example, given the following:

```fsharp
let M<'b>(c :'b, d :'b) = 1
let obj = new obj()
let str = ""
```

these expressions pass type-checking:

```fsharp
M<obj>(obj, str)
M<obj>(str, obj)
M<obj>(obj, obj)
M<obj>(str, str)
M(obj, obj)
M(str, str)
```

These expressions do not, because the target type is a variable type:

```fsharp
M(obj, str)
M(str, obj)
```

