# Constraint Solving and Definition Checking

This chapter describes constraint solving and the checking/elaborating of function, value, and member definitions. This is a component of the overall [Inference Procedures](inference-procedures.md).

## Constraint Solving

Constraint solving involves processing (“solving”) non-primitive constraints to reduce them to
primitive, normalized constraints on type variables. The F# compiler invokes constraint solving every
time it adds a constraint to the set of current inference constraints at any point during type
checking.

Given a type inference environment, the _normalized form_ of constraints is a list of the following
primitive constraints where `typar` is a type inference variable:

```fsother
typar :> type
typar : null
( type or ... or type ) : ( member-sig )
typar : (new : unit -> 'T)
typar : struct
typar : unmanaged
typar : comparison
typar : equality
typar : not struct
typar : enum< type >
typar : delegate< type, type >
```

Each newly introduced constraint is solved as described in the following sections.

### Solving Equational Constraints

New equational constraints in the form `typar = type` or `type = typar` , where `typar` is a type
inference variable, cause `type` to replace `typar` in the constraint problem; `typar` is eliminated. Other
constraints that are associated with `typar` are then no longer primitive and are solved again.

New equational constraints of the form `type<tyarg11,..., tyarg1n> = type<tyarg21, ..., tyarg2n>`
are reduced to a series of constraints `tyarg1i = tyarg2i` on identical named types and solved again.

### Solving Subtype Constraints

Primitive constraints in the form `typar :> obj` are discarded.

New constraints in the form `type1 :> type2`, where `type2` is a sealed type, are reduced to the
constraint `type1` = `type2` and solved again.

New constraints in either of these two forms are reduced to the constraints
`tyarg11 = tyarg 21 ... tyarg1n = tyarg2n` and solved again:

```fsother
type<tyarg11, ..., tyarg1n> :> type<tyarg21, ..., tyarg2n>
type<tyarg11, ..., tyarg1n> = type<tyarg21, ..., tyarg2n>
```

> Note: F# generic types do not support covariance or contravariance. F# treats array types
as invariant during constraint solving. Likewise, F# considers delegate types as
invariant and ignores any variance type annotations on generic interface types and
generic delegate types.

New constraints of the form `type1<tyarg11, ..., tyarg1n> :> type2<tyarg21, ..., tyarg2n>` where
`type1` and `type2` are hierarchically related, are reduced to an equational constraint on two
instantiations of `type2` according to the subtype relation between `type1` and `type2`, and solved again.

For example, if `MySubClass<'T>` is derived from `MyBaseClass<list<'T>>`, then the constraint

```fsother
MySubClass<'T> :> MyBaseClass<int>
```

is reduced to the constraint

```fsother
MyBaseClass<list<'T>> :> MyBaseClass<list<int>>
```

and solved again, so that the constraint `'T = int` will eventually be derived.

> Note : Subtype constraints on single-dimensional array types `ty[] :> ty` are reduced to
residual constraints, because these types are considered to implement `IEnumerable<'T>`.
Multidimensional array types `ty[,...,]` are also subtypes of the base array type.
<br>A type may support multiple instantiations of the same interface type, such as
`C : I<int>, I<string>`. Consequently, it is more difficult to solve a constraint such as
`C :> I<'T>`. Such constraints are rarely used in practice in F# coding. To solve this
constraint, the compiler reduces it to a constraint `C :> I<'T>`, where `I<'T>` is the first
interface type that occurs in the tree of supported interface types, when the tree is
ordered from most derived to least derived, and iterated left-to-right in the order of the
interface declarations.

> **Clef Note**: Clef does not inherit array subtyping from a managed runtime. Array types implement collection interfaces as defined by the native library.

New constraints of the form `type :> 'b` are solved again as `type = 'b`.

> Note : Such constraints typically occur in generic code where a method accepts a parameter
of a "naked" variable type—for example, a function with a signature such as
`T Choose<'T>(T x, T y)`.

### Solving Nullness, Struct, and Other Simple Constraints

New constraints in any of the following forms, where `type` is not a variable type, are reduced to
further constraints:

```fsother
type : null
type : (new : unit -> 'T)
type : struct
type : not struct
type : enum< type >
type : delegate< type, type >
type : unmanaged
```

The compiler then resolves them according to the requirements for each kind of constraint listed in
[§](types-and-type-constraints.md#type-constraints) and [§](types-and-type-constraints.md#nullness).

### Solving Member Constraints

New constraints in the following form are solved as _member constraints_ ([§](types-and-type-constraints.md#member-constraints)):

```fsother
(type1 or ... or typen) : (member-sig)
```

A member constraint is satisfied if one of the types in the _support set_ `type1 ... typen` satisfies the
member constraint. A static type `type` satisfies a member constraint in the form
`(static~opt member ident : arg-type1 * ... * arg-typen -> ret-type)`
if all of the following are true:

- `type` is a named type whose type definition contains the following member, which takes `n`
    arguments:
    `static~opt member ident : formal-arg-type1 * ... * formal-arg-typen -> ret-type`
- The `type` and the constraint are both marked `static` or neither is marked `static`.
- The assertion of type inference constraints on the arguments and return types does not result in
    a type inference error.

As mentioned in [§](types-and-type-constraints.md#member-constraints), a type variable may not be involved in the support set of more than one
member constraint that has the same name, staticness, argument arity, and support set. If a type
variable is in the support set of more than one such constraint, the argument and return types are
themselves constrained to be equal.

#### Simulation of Solutions for Member Constraints

Certain types are assumed to implicitly define static members even though no explicit member
definition exists in the type declaration. This mechanism is used to implement the extensible
conversion and math functions of the F# library including `sin`, `cos`, `int`, `float`, `(+)`, and `(-)`. The
following table shows the static members that are implicitly defined for various types.

| Type | Implicitly defined static members |
| --- | --- |
| Integral types:<br> `byte`, `sbyte`, `int16`, `uint16`, `int32`, `uint32`, `int64`, `uint64`, `nativeint`, `unativeint` | `op_BitwiseAnd`, `op_BitwiseOr`, `op_ExclusiveOr`, `op_LeftShift`, `op_RightShift`, `op_UnaryPlus`, `op_UnaryNegation`, `op_Increment`, `op_Decrement`, `op_LogicalNot`, `op_OnesComplement`, `op_Addition`, `op_Subtraction`, `op_Multiply`, `op_Division`, `op_Modulus`, `op_UnaryPlus`<br>`op_Explicit`: takes the type as an argument and returns `byte`, `sbyte`, `int16`, `uint16`, `int32`, `uint32`, `int64`, `uint64`, `float32`, `float`, `decimal`, `nativeint`, or `unativeint` |
| Signed integral types:<br> `sbyte`, `int16`, `int32`, `int64` and `nativeint` | `op_UnaryNegation`, `Sign`, `Abs` |
| Floating-point types:<br>`float32` and `float` | `Sin`, `Cos`, `Tan`, `Sinh`, `Cosh`, `Tanh`, `Atan`, `Acos`, `Asin`, `Exp`, `Ceiling`, `Floor`, `Round`, `Log10`, `Log`, `Sqrt`, `Atan2`, `Pow`, `op_Addition`, `op_Subtraction`, `op_Multiply`, `op_Division`, `op_Modulus`, `op_UnaryPlus`, `op_UnaryNegation`, `Sign`, `Abs` <br> `op_Explicit`: takes the type as an argument and returns `byte`, `sbyte`, `int16`, `uint16`, `int32`, `uint32`, `int64`, `uint64`, `float32`, `float`, `decimal`, `nativeint`, or `unativeint` |
| decimal type <br>**Note** : The decimal type is included only for the Sign static member. | `Sign` |
| String type `string` | `op_Addition` <br> `op_Explicit`: takes the type as an argument and return `byte`, `sbyte`, `int16`, `uint16`, `int32`, `uint32`, `int64`, `uint64`, `float32`, `float` or `decimal`. |

### Over-constrained User Type Annotations

An implementation of F# must give a warning if a type inference variable that results from a user
type annotation is constrained to be a type other than another type inference variable. For example,
the following results in a warning because `'T` has been constrained to be precisely `string`:

```fsharp
let f (x:'T) = (x:string)
```

During the resolution of overloaded methods, resolutions that do not give such a warning are
preferred over resolutions that do give such a warning.

## Checking and Elaborating Function, Value, and Member Definitions

This section describes how function, value, and member definitions are checked, generalized, and
elaborated. These definitions occur in the following contexts:

- Module declarations
- Class type declarations
- Expressions
- Computation expressions

Recursive definitions can also occur in each of these locations. In addition, member definitions in a
mutually recursive group of type declarations are implicitly recursive.

Each definition is one of the following:

- A function definition :

    ```fsgrammar
    inline~opt ident1 pat1 ... patn :~opt return-type~opt = rhs-expr
    ```

- A value definition, which defines one or more values by matching a pattern against an expression:

    ```fsgrammar
    mutable~opt pat :~opt type~opt = rhs-expr
    ```

- A member definition:

    ```fsgrammar
    static~opt member ident~opt ident pat1 ... patn = expr
    ```

For a function, value, or member definition in a class:

1. If the definition is an instance function, value or member, checking uses an environment to
    which both of the following have been added:
    - The instance variable for the class, if one is present.
    - All previous function and value definitions for the type, whether static or instance.
2. If the definition is static (that is, a static function, value or member defeinition), checking uses an
    environment to which all previous static function, value, and member definitions for the type
    have been added.

### Ambiguities in Function and Value Definitions

In one case, an ambiguity exists between the syntax for function and value definitions. In particular,
`ident pat = expr` can be interpreted as either a function or value definition. For example, consider
the following:

```fsharp
type OneInteger = Id of int

let Id x = x
```

In this case, the ambiguity is whether `Id x` is a pattern that matches values of type `OneInteger` or is
the function name and argument list of a function called `Id`. In F# this ambiguity is always resolved
as a function definition. In this case, to make a value definition, use the following syntax in which the
ambiguous pattern is enclosed in parentheses:

```fsharp
let v = if 3 = 4 then Id "yes" else Id "no"
let (Id answer) = v
```

### Mutable Value Definitions

Value definitions may be marked as mutable. For example:

```fsharp
let mutable v = 0
while v < 10 do
    v <- v + 1
    printfn "v = %d" v
```

These variables are implicitly dereferenced when used.

### Processing Value Definitions

A value definition `pat = rhs-expr` with optional pattern type `type` is processed as follows:

1. The pattern `pat` is checked against a fresh initial type `ty` (or `type` if such a type is present). This
    check results in zero or more identifiers `ident1 ... identm`, each of type `ty1` ... `tym`.
2. The expression `rhs-expr` is checked against initial type `ty`, resulting in an elaborated form `expr`.
3. Each `identi` (of type `tyi`) is then generalized ([§](inference-constraint-solving.md#generalization)) and yields generic parameters `<typarsj>`.
4. The following rules are checked:
    - All `identj` must be distinct.
    - Value definitions may not be `inline`.
5. If `pat` is a single value pattern, the resulting elaborated definition is:

    ```fsgrammar
    ident<typars1> = expr
    body-expr
    ```

6. Otherwise, the resulting elaborated definitions are the following, where `tmp` is a fresh identifier
    and each `expri` results from the compilation of the pattern `pat` ([§](patterns.md#patterns)) against input `tmp`.

    ```fsgrammar
    tmp<typars1 ... typarsn> = expr
    ident1<typars1> = expr1
    ...
    identn<typarsn> = exprn
    ```

### Processing Function Definitions

A function definition `ident1 pat1 ... patn = rhs-expr` is processed as follows:

1. If `ident1` is an active pattern identifier then active pattern result tags are added to the
    environment ([§](namespaces-and-modules.md#active-pattern-definitions-in-modules)).
2. The expression `(fun pat1 ... patn : return-type -> rhs-expr)` is checked against a fresh initial
    type `ty1` and reduced to an elaborated form `expr1`. The return type is omitted if the definition
    does not specify it.
3. The `ident1` (of type `ty1`) is then generalized ([§](inference-constraint-solving.md#generalization)) and yields generic parameters `<typars1>`.
4. The following rules are checked:
    - Function definitions may not be `mutable`. Mutable function values should be written as follows:

    ```fsother
    let mutable f = (fun args -> ...)`
    ```

    - The patterns of functions may not include optional arguments ([§](type-definitions.md#optional-arguments-to-method-members)).
5. The resulting elaborated definition is:

    ```fsgrammar
    ident1<typars1> = expr1
    ```

### Processing Recursive Groups of Definitions

A group of functions and values may be declared recursive through the use of `let rec`. Groups of
members in a recursive set of type definitions are also implicitly recursive. In this case, the defined
values are available for use within their own definitions—that is, within all the expressions on the
right-hand side of the definitions.

For example:

```fsharp
let rec twoForward count =
    printfn "at %d, taking two steps forward" count
    if count = 1000 then "got there!"
    else oneBack (count + 2)
and oneBack count =
    printfn "at %d, taking one step back " count
    twoForward (count – 1)
```

When one or more definitions specifies a value, the recursive expressions are analyzed for safety
([§](inference-constraint-solving.md#recursive-safety-analysis)). This analysis may result in warnings—including some reported at compile time—and
runtime checks.

Within recursive groups, each definition in the group is checked ([§](inference-constraint-solving.md#generalization)) and then the definitions
are generalized incrementally. In addition, any use of an ungeneralized recursive definition results in
immediate constraints on the recursively defined construct. For example, consider the following
declaration:

```fsharp
let rec countDown count x =
    if count > 0 then
        let a = countDown (count - 1) 1 // constrains "x" to be of type int
        let b = countDown (count – 1) "Hello" // constrains "x" to be of type string
        a + b
    else
        1
```

In this example, the definition is not valid because the recursive uses of `f` result in inconsistent
constraints on `x`.

If a definition has a full signature, early generalization applies and recursive calls at different types
are permitted ([§](inference-constraint-solving.md#generalization)). For example:

```fsharp
module M =
    let rec f<'T> (x:'T) : 'T =
        let a = f 1
        let b = f "Hello"
        x
```

In this example, the definition is valid because `f` is subject to early generalization, and so the
recursive uses of `f` do not result in inconsistent constraints on `x`.

### Recursive Safety Analysis

A set of recursive definitions may include value definitions. For example:

```fsharp
type Reactor = React of (int -> React) * int

let rec zero = React((fun c -> zero), 0)

let const n =
    let rec r = React((fun c -> r), n)
    r
```

Recursive value definitions may result in invalid recursive cycles, such as the following:

```fsharp
let rec x = x + 1
```

The _Recursive Safety Analysis_ process partially checks the safety of these definitions and convert
thems to a form that uses lazy initialization, where runtime checks are inserted to check
initialization.

A right-hand side expression is _safe_ if it is any of the following:

- A function expression, including those whose bodies include references to variables that are
    defined recursively.
- An object expression that implements an interface, including interfaces whose member bodies
    include references to variables that are being defined recursively.
- A `lazy` delayed expression.
- A record, tuple, list, or data construction expression whose field initialization expressions are all
    safe.
- A value that is not being recursively bound.
- A value that is being recursively bound and appears in one of the following positions:
  - As a field initializer for a field of a record type where the field is marked `mutable`.
  - As a field initializer for an immutable field of a record type that is defined in the current
       assembly.
       If record fields contain recursive references to values being bound, the record fields must be
       initialized in the same order as their declared type, as described later in this section.

- Any expression that refers only to earlier variables defined by the sequence of recursive
    definitions.

Other right-hand side expressions are elaborated by adding a new definition. If the original definition
is

```fsharp
u = expr
```

then a fresh value (say v) is generated with the definition:

```fsharp
v = lazy expr
```

and occurrences of the original variable `u` on the right-hand side are replaced by `Lazy.force v`. The
following definition is then added at the end of the definition list:

```fsharp
u = v .Force()
```

> Note: This specification implies that recursive value definitions are executed as an
initialization graph of delayed computations. Some recursive references may be checked
at runtime because the computations that are involved in evaluating the definitions
might actually execute the delayed computations. The F# compiler gives a warning for
recursive value definitions that might involve a runtime check. If runtime self-reference
does occur then an exception will be raised.
<br>Recursive value definitions that involve computation are useful when defining objects
such as forms, controls, and services that respond to various inputs. For example, GUI
elements that store and retrieve the state of the GUI elements as part of their
specification typically involve recursive value definitions. A simple example is the
following menu item, which prints out part of its state when invoked:

```fsharp
let rec node : TreeNode =
    { Value = 42
      OnVisit = fun () -> printfn "Visiting node with value %d" node.Value
      Children = [] }
```

> This code results in a compiler warning because, in theory, the
record construction might evaluate the `OnVisit` callback as part of initialization.
In practice, function values in records are not evaluated during construction, so the
warning can be suppressed or ignored by using compiler options.

The F# compiler performs a simple approximate static analysis to determine whether immediate
cyclic dependencies are certain to occur during the evaluation of a set of recursive value definitions.
The compiler creates a graph of definite references and reports an error if such a dependency cycle
exists. All references within function expressions, object expressions, or delayed expressions are
assumed to be indefinite, which makes the analysis an under-approximation. As a result, this check
catches naive and direct immediate recursion dependencies, such as the following:

```fsharp
let rec A = B + 1
and B = A + 1
```

Here, a compile-time error is reported. This check is necessarily approximate because dependencies
under function expressions are assumed to be delayed, and in this case the use of a lazy initialization
means that runtime checks and forces are inserted.

> Note: In F# 3 .1 this check does not apply to value definitions that are generic through
generalization because a generic value definition is not executed immediately, but is
instead represented as a generic method. For example, the following value definitions
are generic because each right-hand-side is generalizable:

```fsharp
let rec a = b
and b = a
```

> In compiled code they are represented as a pair of generic methods, as if the code had
been written as follows:

```fsharp
let rec a<'T>() = b<'T>()
and b<'T>() = a<'T>()
```

> As a result, the definitions are not executed immediately unless the functions are called.
Such definitions indicate a programmer error, because executing such generic,
immediately recursive definitions results in an infinite loop or an exception. In practice
these definitions only occur in pathological examples, because value definitions are
generalizable only when the right-hand-side is very simple, such as a single value. Where
this issue is a concern, type annotations can be added to existing value definitions to
ensure they are not generic. For example:

```fsharp
let rec a : int = b
and b : int = a
```

> In this case, the definitions are not generic. The compiler performs immediate
dependency analysis and reports an error. In addition, record fields in recursive data
expressions must be initialized in the order they are declared. For example:

```fsharp
type Foo = {
    x: int
    y: int
    parent: Foo option
    children: Foo list
}

let rec parent = { x = 0; y = 0; parent = None; children = children }
and children = [{ x = 1; y = 1; parent = Some parent; children = [] }]

printf "%A" parent
```

> Here, if the order of the fields x and y is swapped, a type-checking error occurs.

### Generalization

Generalization is the process of inferring a generic type for a definition where possible, thereby
making the construct reusable with multiple different types. Generalization is applied by default at
all function, value, and member definitions, except where listed later in this section. Generalization
also applies to member definitions that implement generic virtual methods in object expressions.

Generalization is applied incrementally to items in a recursive group after each item is checked.

Generalization takes a set of ungeneralized but type-checked definitions _checked-defns_ that form
part of a recursive group, plus a set of unchecked definitions _unchecked-defns_ that have not yet been
checked in the recursive group, and an environment _env_. Generalization involves the following steps:

1. Choose a subset `generalizable-defns` of `checked-defns` to generalize.

    A definition can be generalized if its inferred type is closed with respect to any inference
    variables that are present in the types of the `unchecked-defns` that are in the recursive group and
    that are not yet checked or which, in turn, cannot be generalized. A greatest-fixed-point
    computation repeatedly removes definitions from the set of `checked-defns` until a stable set of
    generalizable definitions remains.
2. Generalize all type inference variables that are not otherwise ungeneralizable and for which any
    of the following is true:
    - The variable is present in the inferred types of one or more of `generalizable-defns`.
    - The variable is a type parameter copied from the enclosing type definition (for members and
       “let” definitions in classes).
    - The variable is explicitly declared as a generic parameter on an item.

The following type inference variables cannot be generalized:

- A type inference variable `^typar` that is part of the inferred or declared type of a definition,
    unless the definition is marked `inline`.
- A type inference variable in an inferred type in the _ExprItems_ or _PatItems_ tables of `env` , or in
    an inferred type of a module in the _ModulesAndNamespaces_ table in `env`.
- A type inference variable that is part of the inferred or declared type of a definition in which
    the elaborated right-hand side of the definition is not a generalizable expression, as
    described later in this section.
- A type inference variable that appears in a constraint that itself refers to an ungeneralizable
    type variable.

Generalizable type variables are computed by a greatest-fixed-point computation, as follows:

1. Start with all variables that are candidates for generalization.

2. Determine a set of variables `U` that cannot be generalized because they are free in the
environment or present in ungeneralizable definitions.

3. Remove the variables in _U_ from consideration.

4. Add to `U` any inference variables that have a constraint that involves a variable in `U`.

5. Repeat steps 2 through 4.

Informally, generalizable expressions represent a subset of expressions that can be freely copied and
instantiated at multiple types without affecting the typical semantics of an F# program. The
following expressions are generalizable:

- A function expression
- An object expression that implements an interface
- A delegate expression
- A “let” definition expression in which both the right-hand side of the definition and the body of
    the expression are generalizable
- A “let rec” definition expression in which the right-hand sides of all the definitions and the body
    of the expression are generalizable
- A tuple expression, all of whose elements are generalizable
- A record expression, all of whose elements are generalizable, where the record contains no
    mutable fields
- A union case expression, all of whose arguments are generalizable
- An exception expression, all of whose arguments are generalizable
- An empty array expression
- A simple constant expression
- An application of a type function that has the `GeneralizableValue` attribute.

Explicit type parameter definitions on value and member definitions can affect the process of type
inference and generalization. In particular, a declaration that includes explicit generic parameters
will not be generalized beyond those generic parameters. For example, consider this function:

```fsharp
let f<'T> (x : 'T) y = x
```

During type inference, this will result in a function of the following type, where `'_b` is a type
inference variable that is yet to be resolved.

```fsharp
f<'T> : 'T -> '_b -> '_b
```

To permit generalization at these definitions, either remove the explicit generic parameters (if they
can be inferred), or use the required number of parameters, as the following example shows:

```fsharp
let throw<'T,'U> (x:'T) (y:'U) = x
```

### Condensation of Generalized Types

After a function or member definition is generalized, its type is condensed by removing generic type
parameters that apply subtype constraints to argument positions. (The removed flexibility is
implicitly reintroduced at each use of the defined function; see [§](inference-application-resolution.md#implicit-insertion-of-flexibility-for-uses-of-functions-and-members)).

Condensation decomposes the type of a value or member to the following form:

```fsgrammar
ty11 * ... * ty1n -> ... -> tym1 * ... * tymn -> rty
```

The positions `tyij` are called the parameter positions for the type.

Condensation applies to a type parameter `'a` if all of the following are true:

- `'a` is not an explicit type parameter.
- `'a` occurs at exactly one `tyij` parameter position.
- `'a` has a single coercion constraint `'a :> ty` and no other constraints. However, one additional
    nullness constraint is permitted if `ty` satisfies the nullness constraint.
- `'a` does not occur in any other `tyij`, nor in `rty`.
- `'a` does not occur in the constraints of any condensed `typar`.

Condensation is a greatest-fixed-point computation that initially assumes all generalized type
parameters are condensed, and then progressively removes type parameters until a minimal set
remains that satisfies the above rules.

The compiler removes all condensed type parameters and replaces them with their subtype
constraint `ty`. For example:

```fsharp
let F x = (x :> IComparable).CompareTo(x)
```

After generalization, the function is inferred to have the following type:

```fsharp
F : 'a -> int when 'a :> IComparable
```

In this case, the actual inferred, generalized type for `F` is condensed to:

```fsharp
F : IComparable -> R
```

Condensation does not apply to arguments of unconstrained variable type. For example:

```fsharp
let ignore x = ()
```

with type

```fsharp
ignore: 'a -> unit
```

In particular, this is not condensed to

```fsharp
ignore: obj -> unit
```

In rare cases, condensation affects the points at which value types are boxed. In the following
example, the value `3` is now boxed at uses of the function:

```fsharp
F 3
```

If a function is not generalized, condensation is not applied. For example, consider the following:

```fsharp
let test1 =
    let ff = Seq.map id >> Seq.length
    (ff [1], ff [| 1 |]) // error here
```

In this example, `ff` is not generalized, because it is not defined by using a generalizable expression—
computed functions such as `Seq.map id >> Seq.length` are not generalizable. This means that its
inferred type, after processing the definition, is

```fsharp
F : '_a -> int when '_a :> seq<'_b>
```

where the type variables are not generalized and are unsolved inference variables. The application
of `ff` to `[1]` equates `'a` with `int list`, making the following the type of `F`:

```fsharp
F : int list -> int
```

The application of `ff` to an array type then causes an error. This is similar to the error returned by
the following:

```fsharp
let test1 =
    let ff = Seq.map id >> Seq.length
    (ff [1], ff ["one"]) // error here
```

Again, `ff` is not generalized, and its use with arguments of type `int list` and `string list` is not
permitted.

