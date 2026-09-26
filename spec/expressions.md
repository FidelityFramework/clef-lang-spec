---
title: "Expressions"
weight: 200
category: Language
status: normative
---

The expression forms and related elements are as follows:

```fsgrammar
expr :=
    const                               -- a constant value
    ( expr )                            -- block expression
    begin expr end                      -- block expression
    long-ident-or-op                    -- lookup expression
    expr '.' long-ident-or-op           -- dot lookup expression
    expr expr                           -- application expression
    expr ( expr )                       -- high precedence application
    expr < types >                      -- type application expression
    expr infix-op expr                  -- infix application expression
    prefix-op expr                      -- prefix application expression
    expr .[ expr ]                      -- indexed lookup expression
    expr .[ slice-ranges ]              -- slice expression
    expr <- expr                        -- assignment expression
    expr , ... , expr                   -- tuple expression
    struct (expr , ... , expr)          -- struct tuple expression
    new type expr                       -- simple object expression
    { new base-call object-members interface-impls } -- object expression
    { field-initializers }              -- record expression
    { expr with field-initializers }    -- record cloning expression
    [ expr ; ... ; expr ]               -- list expression
    [| expr ; ... ; expr |]             -- array expression
    expr { comp-or-range-expr }         -- computation expression
    [ comp-or-range-expr ]              -- computed list expression
    [| comp-or-range-expr |]            -- computed array expression
    lazy expr                           -- delayed expression
    eager expr                          -- explicit shallow-demand expression
    expr : type                         -- type annotation
    expr :> type                        -- static upcast coercion
    expr :? type                        -- dynamic type test
    expr :?> type                       -- dynamic downcast coercion
    upcast expr                         -- static upcast expression
    downcast expr                       -- dynamic downcast expression
    let function-defn in expr           -- function definition expression
    let value-defn in expr              -- value definition expression
    let rec function-or-value-defns in expr -- recursive definition expression
    use ident = expr in expr            -- deterministic disposal expression
    fun argument-pats - > expr          -- function expression
    function rules                      -- matching function expression
    expr ; expr                         -- sequential execution expression
    match expr with rules               -- match expression
    try expr with rules                 -- try/with expression
    try expr finally expr               -- try/finally expression
    if expr then expr elif-branches? else-branch? -- conditional expression
    while expr do expr done             -- while loop
    for ident = expr to expr do expr done -- simple for loop
    for pat in expr - or-range-expr do expr done -- enumerable for loop
    assert expr                         -- assert expression
    <@ expr @>                          -- quoted expression
    <@@ expr @@>                        -- quoted expression

    %expr                              -- expression splice
    %%expr                              -- weakly typed expression splice

    (static-typars : (member-sig) expr) -- static member invocation
```

Expressions are defined in terms of patterns and other entities that are discussed later in this
specification. The following constructs are also used:

```fsgrammar
exprs := expr ',' ... ',' expr

expr-or-range-expr :=
    expr
    range-expr

elif-branches := elif-branch ... elif-branch

elif-branch := elif expr then expr

else-branch := else expr

function-or-value-defn :=
    function-defn
    value-defn

function-defn :=
    inline? access? ident-or-op typar-defns? argument-pats return-type? = expr

value-defn :=
    mutable? access? pat typar-defns? return-type? = expr

return-type :=
    : type

function-or-value-defns :=
    function-or-value-defn and ... and function-or-value-defn

argument-pats := atomic-pat ... atomic-pat

field-initializer :=
    long-ident = expr -- field initialization

field-initializers := field-initializer ; ... ; field-initializer

object-construction :=
    type expr -- construction expression
    type      -- interface construction expression

base-call :=
    object-construction          -- anonymous base construction
    object-construction as ident -- named base construction

interface-impls := interface-impl ... interface-impl

interface-impl :=
    interface type object-members? -- interface implementation

object-members := with member-defns end

member-defns := member-defn ... member-defn
```

Computation and range expressions are defined in terms of the following productions:

```fsgrammar
comp-or-range-expr :=
    comp-expr
    short-comp-expr
    range-expr

comp-expr :=
    let! pat = expr in comp-expr    -- binding computation
    let pat = expr in comp-expr
    do! expr in comp-expr           -- sequential computation
    do expr in comp-expr
    use! pat = expr in comp-expr    -- auto cleanup computation
    use pat = expr in comp-expr
    yield! expr                     -- yield computation
    yield expr                      -- yield result
    return! expr                    -- return computation
    return expr                     -- return result
    if expr then comp - expr        -- control flow or imperative action
    if expr then expr else comp-expr
    match! expr with pat -> comp-expr | ... | pat -> comp-expr
    match expr with pat -> comp-expr | ... | pat -> comp-expr
    try comp - expr with pat -> comp-expr | ... | pat -> comp-expr
    try comp - expr finally expr
    while expr do comp - expr done
    for ident = expr to expr do comp - expr done
    for pat in expr - or-range-expr do comp - expr done
    comp - expr ; comp - expr
    expr

short-comp-expr :=
    for pat in expr-or-range-expr -> expr -- yield result

range-expr :=
    expr .. expr                    -- range sequence
    expr .. expr .. expr            -- range sequence with skip

slice-ranges := slice-range , ... , slice-range

slice-range :=
    expr                            -- slice of one element of dimension
    expr ..                         -- slice from index to end
    .. expr                         -- slice from start to index
    expr .. expr                    -- slice from index to index
    '*'                             -- slice from start to end
```

## Some Checking and Inference Terminology

The rules applied to check individual expressions are described in the following subsections. Where
necessary, these sections reference specific inference procedures such as _Name Resolution_ ([§](inference-name-resolution.md#name-resolution))
and _Constraint Solving_ ([§](inference-constraint-solving.md#constraint-solving)).

All expressions are assigned a static type through type checking and inference. During type checking,
each expression is checked with respect to an _initial type_. The initial type establishes some of the
information available to resolve method overloading and other language constructs. We also use the
following terminology:

- The phrase “the type `ty1` is asserted to be equal to the type `ty2`” or simply “`ty1 = ty2` is asserted”
    indicates that the constraint “`ty1 = ty2`” is added to the current inference constraints.

- The phrase “`ty1` is asserted to be a subtype of `ty2`” or simply “`ty1 :> ty2` is asserted” indicates
    that the constraint `ty1 :> ty2` is added to the current inference constraints.
- The phrase “type `ty` is known to ...” indicates that the initial type satisfies the given property
    given the current inference constraints.
- The phrase “the expression `expr` has type `ty` ” means the initial type of the expression is asserted
    to be equal to `ty`.

Additionally:

- The addition of constraints to the type inference constraint set fails if it causes an inconsistent
    set of constraints ([§](inference-constraint-solving.md#constraint-solving)). In this case either an error is reported or, if we are only attempting to
    _assert_ the condition, the state of the inference procedure is left unchanged and the test fails.

## Elaboration and Elaborated Expressions

Checking an expression generates an _elaborated expression_ in a simpler, reduced language that
effectively contains a fully resolved and annotated form of the expression. The elaborated
expression provides more explicit information than the source form. For example, the elaborated
form of `Console.WriteLine("Hello")` indicates exactly which overloaded method definition
the call has resolved to.
Except for this extra resolution information, elaborated forms are syntactically a subset of syntactic
expressions, and in some cases (such as constants) the elaborated form is the same as the source
form. This specification uses the following elaborated forms:

- Constants
- Resolved value references: `path`
- Lambda expressions: `(fun ident -> expr)`
- Primitive object expressions
- Data expressions (tuples, union cases, array creation, record creation)
- Default initialization expressions
- Local definitions of values: `let ident = expr in expr`
- Local definitions of functions:
    `let rec ident = expr and ... and ident = expr in expr`
- Applications of methods and functions (with static overloading resolved)
- Dynamic type coercions: `expr :?> type`
- Dynamic type tests: `expr :? type`
- For-loops: `for ident in ident to ident do expr done`
- While-loops: `while expr do expr done`
- Sequencing: `expr; expr`
- Try-with: `try expr with expr`
- Try-finally: `try expr finally expr`
- The constructs required for the elaboration of pattern matching ([§](patterns.md#patterns)).
  - Null tests
  - Switches on integers and other types
  - Switches on union cases
  - Switches on the runtime types of objects

The following constructs are used in the elaborated forms of expressions that make direct
assignments to local variables and arrays and generate “byref” pointer values. The operations are
These are low-level operations for pointer manipulation.

- Assigning to a byref-pointer: `expr <-stobj expr`
- Generating a byref-pointer by taking the address of a mutable value: `&path`.
- Generating a byref-pointer by taking the address of a record field: `&(expr.field)`
- Generating a byref-pointer by taking the address of an array element: `&(expr.[expr])`

Elaborated expressions form the basis for evaluation (see [§](expressions.md#evaluation-of-elaborated-forms)) and for the expression trees that
_quoted expressions_ return (see [§](expressions.md#quoted-expressions)).

By convention, when describing the process of elaborating compound expressions, we omit the
process of recursively elaborating sub-expressions.

## Data Expressions

This section describes the following data expressions:

- Simple constant expressions
- Tuple expressions
- List expressions
- Array expressions
- Record expressions
- Copy-and-update record expressions
- Function expressions
- Object expressions
- Delayed expressions
- Computation expressions
- Sequence expressions
- Range expressions
- Lists via sequence expressions
- Arrays via sequence expressions
- Null expressions
- 'printf' formats

### Simple Constant Expressions

Simple constant expressions are numeric, string, Boolean and unit constants. For example:

```fsgrammar
3y              // sbyte
32uy            // byte
17s             // int16
18us            // uint16
86              // int/int32
99u             // uint32
99999999L       // int64
10328273UL      // uint64
1.              // float/double
1.01            // float/double
1.01e10         // float/double
1.0f            // float32/single
1.01f           // float32/single
1.01e10f        // float32/single
99999999n       // nativeint    (platform-sized signed integer)
10328273un      // unativeint   (platform-sized unsigned integer)
99999999I       // bigint       (arbitrary precision integer)
'a'             // char         (Unicode scalar value)
"3"             // string       (UTF-8 string)
"c:\\home"      // string       (UTF-8 string)
@"c:\home"      // string       (Verbatim, UTF-8 string)
"ASCII"B        // byte[]
()              // unit         (zero-sized type)
false           // bool         (8-bit boolean)
true            // bool         (8-bit boolean)
 
```

Simple constant expressions have the corresponding simple type and elaborate to the corresponding
simple constant value.

Integer literals with the suffixes `Q`, `R`, `Z`, `I`, `N`, `G` are processed using the following syntactic translation:

```fsgrammar
xxxx<suffix>
    For xxxx = 0                → NumericLiteral<suffix>.FromZero()
    For xxxx = 1                → NumericLiteral<suffix>.FromOne()
    For xxxx in the Int32 range → NumericLiteral<suffix>.FromInt32(xxxx)
    For xxxx in the Int64 range → NumericLiteral<suffix>.FromInt64(xxxx)
    For other numbers           → NumericLiteral<suffix>.FromString("xxxx")
```

For example, defining a module `NumericLiteralZ` as below enables the use of the literal form `32Z` to
generate a sequence of 32 ‘Z’ characters. No literal syntax is available for numbers outside the range
of 32-bit integers.

```fsharp
module NumericLiteralZ =
    let FromZero() = ""
    let FromOne() = "Z"
    let FromInt32 n = String.replicate n "Z"
```

F# compilers may optimize on the assumption that calls to numeric literal functions always
terminate, are idempotent, and do not have observable side effects.

### Tuple Expressions

An expression of the form `expr1 , ..., exprn` is a _tuple expression_. For example:

```fsharp
let three = (1,2,"3")
let blastoff = (10,9,8,7,6,5,4,3,2,1,0)
```

The expression has the type `S<ty1 * ... * tyn>` for fresh types `ty1 ... tyn` and fresh pseudo-type `S` that indicates the "structness" (i.e. reference tuple or struct tuple) of the tuple. Each individual
expression `expri` is checked using initial type `tyi`. The pseudo-type `S` participates in type checking similar to normal types until it is resolved to either reference or struct tuple, with a default of reference tuple.

An expression of the form `struct (expr1 , ..., exprn)` is a _struct tuple expression_. For example:

```fsharp
let pair = struct (1,2)
```

A _struct tuple expression_ is checked in the same way as a _tuple expression_, but the pseudo-type `S` is resolved to struct tuple.

Tuple types `ty1 * ... * tyn` are compiled to native product types with sequential field layout. There is no limit on tuple arity in Clef.

> **Clef Note**: In Clef, tuples are compiled directly to unboxed product types with deterministic memory layout. Fields are laid out sequentially with natural alignment. There is no distinction between "reference tuples" and "struct tuples" - both compile to value types with the same representation. The `struct` keyword on tuples is accepted for source compatibility with managed F#.

### List Expressions

An expression of the form `[expr1 ; ...; exprn]` is a _list expression_. The initial type of the expression is
asserted to be `FSharp.Collections.List<ty>` for a fresh type `ty`.

If `ty` is a named type, each expression `expri` is checked using a fresh type `ty'` as its initial type, with
the constraint `ty' :> ty`. Otherwise, each expression `expri` is checked using `ty` as its initial type.

List expressions elaborate to uses of `FSharp.Collections.List<_>` as
`op_Cons(expr1 ,(op_Cons(_expr2 ... op_Cons(exprn, op_Nil) ...)` where `op_Cons` and `op_Nil` are the
union cases with symbolic names `::` and `[]` respectively.

### Array Expressions

An expression of the form `[|expr1; ...; exprn|]` is an _array expression_. The initial type of the
expression is asserted to be `ty[]` for a fresh type `ty`.

If this assertion determines that `ty` is a named type, each expression `expri` is checked using a fresh
type `ty'` as its initial type, with the constraint `ty' :> ty`. Otherwise, each expression `expri` is
checked using `ty` as its initial type.

Array expressions are a primitive elaborated form.

Constant arrays are placed in read-only data sections of the binary and initialized
directly from the executable image.

### Record Expressions

An expression of the form `{field-initializer1; ... ; field-initializern}` is a _record
construction expression_. For example:

```fsharp
type Data = { Count : int; Name : string }
let data1 = { Count = 3; Name = "Hello"; }
let data2 = { Name = "Hello"; Count= 3 }
```

In the following example, `data4` uses a long identifier to indicate the relevant field:

```fsharp
module M =
    type Data = { Age : int; Name : string; Height : float }

let data3 = { M.Age = 17; M.Name = "John"; M.Height = 186.0 }
let data4 = { data3 with M.Name = "Bill"; M.Height = 176.0 }
```

Fields may also be referenced by using the name of the containing type:

```fsharp
module M2 =
    type Data = { Age : int; Name : string; Height : float }

let data5 = { M2.Data.Age = 17; M2.Data.Name = "John"; M2.Data.Height = 186.0 }
let data6 = { data5 with M2.Data.Name = "Bill"; M2.Data.Height=176.0 }

open M2
let data7 = { Data.Age = 17; Data.Name = "John"; Data.Height = 186.0 }
let data8 = { data5 with Data.Name = "Bill"; Data.Height=176.0 }
```

Each `field-initializeri` has the form `field-labeli = expri`. Each `field-labeli` is a `long-ident`,
which must resolve to a field `F` i in a unique record type `R` as follows:

- If `field-labeli` is a single identifier `fld` and the initial type is known to be a record type
    `R<_, ..., _>` that has field `Fi` with name `fld`, then the field label resolves to `Fi`.
- If `field-labeli` is not a single identifier or if the initial type is a variable type, then the field label
    is resolved by performing _Field Label Resolution_ (see [§](inference-name-resolution.md#name-resolution)) on `field-labeli`. This procedure
    results in a set of fields `FSeti`. Each element of this set has a corresponding record type, thus
    resulting in a set of record types `RSeti`. The intersection of all `RSeti` must yield a single record
    type `R`, and each field then resolves to the corresponding field in `R`.
    The set of fields must be complete. That is, each field in record type `R` must have exactly one
    field definition. Each referenced field must be accessible (see [§](namespaces-and-modules.md#accessibility-annotations)), as must the type `R`.

After all field labels are resolved, the overall record expression is asserted to be of type
`R<ty1, ..., tyN>` for fresh types `ty1, ..., tyN`. Each `expri` is then checked in turn. The initial type is
determined as follows:

1. Assume the type of the corresponding field `Fi` in `R<ty1, ..., tyN>` is `ftyi`
2. If the type of `Fi` prior to taking into account the instantiation `<ty1, ..., tyN>` is a named type, then
    the initial type is a fresh type inference variable `fty'i` with a constraint `fty'i :> ftyi`.
3. Otherwise the initial type is `ftyi`.

Primitive record constructions are an elaborated form in which the fields appear in the same order
as in the record type definition. Record expressions themselves elaborate to a form that may
introduce local value definitions to ensure that expressions are evaluated in the same order that the
field definitions appear in the original expression. For example:

```fsharp
type R = {b : int; a : int }
{ a = 1 + 1; b = 2 }
```

The expression on the last line elaborates to `let v = 1 + 1 in { b = 2; a = v }`.

Records expressions are also used for object initializations in additional object constructor
definitions ([§](type-definitions.md#additional-object-constructors-in-classes)). For example:

```fsharp
type C =
    val x : int
    val y : int
    new() = { x = 1; y = 2 }
```

### Copy-and-update Record Expressions

A _copy-and-update record expression_ has the following form:

```fsgrammar
{ expr with field-initializers }
```

where `field-initializers` is of the following form:

```fsgrammar
field-label1 = expr1; ...; field-labeln = exprn
```

Each `field-labeli` is a `long-ident`. In the following example, `data2` is defined by using such an
expression:

```fsharp
type Data = { Age : int; Name : string; Height : float }
let data1 = { Age = 17; Name = "John"; Height = 186.0 }
let data2 = { data1 with Name = "Bill"; Height = 176.0 }
```

The expression `expr` is first checked with the same initial type as the overall expression. Next, the
field definitions are resolved by using the same technique as for record expressions. Each field label
must resolve to a field `Fi` in a single record type `R` , all of whose fields are accessible. After all field
labels are resolved, the overall record expression is asserted to be of type `R<ty1, ..., tyN>` for fresh
types `ty1, ..., tyN`. Each `expri` is then checked in turn with initial type that results from the following
procedure:

1. Assume the type of the corresponding field `Fi` in `R<ty1, ..., tyN>` is `ftyi`.
2. If the type of `Fi` before considering the instantiation `<ty1, ..., tyN>` is a named type, then the
    initial type is a fresh type inference variable `fty'i` with a constraint `fty'i :> ftyi`.
3. Otherwise, the initial type is `ftyi`.

A copy-and-update record expression elaborates as if it were a record expression written as follows:

`let v = expr in { field-label1 = expr1; ...; field-labeln = exprn; F1 = v.F1; ...; FM = v.FM }`
where `F1 ... FM` are the fields of `R` that are not defined in `field-initializers` and `v` is a fresh
variable.

### Function Expressions

An expression of the form `fun pat1 ... patn -> expr` is a _function expression_. For example:

```fsharp
(fun x -> x + 1)
(fun x y -> x + y)
(fun [x] -> x) // note, incomplete match
(fun (x,y) (z,w) -> x + y + z + w)
```

Function expressions that involve only variable patterns are a primitive elaborated form. Function
expressions that involve non-variable patterns elaborate as if they had been written as follows:

```fsharp
fun v1 ... vn ->
    let pat1 = v 1
    ...
    let patn = vn
    expr
```

These parameter-pattern decisions belong to the completed declared application,
not partial application formation. For example, forming the following partial
application does not reach its failed pattern:

```fsharp
let f = fun [x] y -> y
let g = f [] // ok
 
```

If the result of the completed application `g 3` is demanded, its parameter
pattern fails. Execution then follows the always-active
[exhausted-match contract](patterns.md#exhausted-match-rules): termination with
a source diagnostic, not a catchable `MatchFailureException`. An ordinary
unused binding of that result remains deferred:

```fsharp
let z = g 3 // deferred until z is demanded
 
```

### Object Expressions

An expression of the following form is an _object expression_ :

```fsharp
{ new ty0 args-expr? object-members
  interface ty1 object-members1
  ...
  interface tyn object-membersn }
```

In the case of the interface declarations, the `object-members` are optional and are considered empty
if absent. Each set of `object-members` has the form:

```fsgrammar
with member-defns end?
```

Lexical filtering inserts simulated `$end` tokens when lightweight syntax is used.

Each member of an object expression members can use the keyword `member`, `override`, or `default`.
The keyword `member` can be used even when overriding a member or implementing an interface.

For example:

```fsharp
let obj1 =
    { new IComparer<int> with
        member x.Compare(a,b) = compare (a % 7) (b % 7) }

let obj2 =
    { new IDisposable with
        member x.Dispose() = Console.WriteLine "Disposed" }

let obj3 =
    { new IComparer<int> with
        member x.Compare(a,b) = compare (a % 7) (b % 7)
      interface IDisposable with
        member x.Dispose() = Console.WriteLine "Disposed" }
```

An object expression can specify additional interfaces beyond those required to fulfill the abstract
slots of the type being implemented. For example, `obj3` in the preceding examples has static type
`IComparer<int>` but the object additionally implements the interface `IDisposable`. The additional interfaces
are not part of the static type of the overall expression, but can be revealed through type tests.

> **Clef Note**: Object expressions in Clef MUST implement at least one interface type. The base type `obj` does not exist in native compilation, so object expressions of the form `{ new obj() with ... }` are not permitted. See [Native Type Mappings](native-type-mappings.md#the-universal-base-type-obj-is-not-available).

Object expressions are statically checked as follows.

1. First, `ty0` to `tyn` are checked to verify that they are named types. The overall type of the
expression is `ty0` and is asserted to be equal to the initial type of the expression.

2. The type `ty0` must be a class or interface type. The base construction argument `args-expr` must
    appear if and only if `ty0` is a class type. The type must have one or more accessible constructors;
    the call to these constructors is resolved and elaborated using _Method Application Resolution_
    (see [§](inference-application-resolution.md#method-application-resolution)). Except for `ty0`, each `tyi` must be an interface type.
3. The F# compiler attempts to associate each member with a unique _dispatch slot_ by using
    _dispatch slot inference_ ([§](inference-supplementary.md#dispatch-slot-inference)). If a unique matching dispatch slot is found, then the argument
    types and return type of the member are constrained to be precisely those of the dispatch slot.
4. The arguments, patterns, and expressions that constitute the bodies of all implementing
    members are next checked one by one to verify the following:
    - For each member, the “this” value for the member is in scope and has type `ty0`.
    - Each member of an object expression can initially access the protected members of `ty0`.
    - If the variable `base-ident` appears, it must be named `base`, and in each member a base
       variable with this name is in scope. Base variables can be used only in the member
       implementations of an object expression, and are subject to the same limitations as byref
       values described in [§](inference-supplementary.md#byref-safety-analysis).

The object must satisfy _dispatch slot checking_ ([§](inference-supplementary.md#dispatch-slot-checking)) which ensures that a one-to-one mapping
exists between dispatch slots and their implementations.

Object expressions elaborate to a primitive form. At execution, each object expression creates an
object whose runtime type is compatible with all of the `tyi` that have a dispatch map that is the
result of _dispatch slot checking_ ([§](inference-supplementary.md#dispatch-slot-checking)).

The following example shows how to implement an interface. The overall type of the expression is `INewIdentity`.

```fsharp
type public INewIdentity =
    abstract IsAnonymous : bool
    abstract Name : string

let anon =
    { new INewIdentity with
        member i.IsAnonymous = true
        member i.Name = "anonymous" }
```

### Delayed Expressions

An expression of the form `lazy expr` is a _delayed expression_. For example:

```fsharp
lazy (printfn "hello world")
```

is syntactic sugar for

```fsharp
Lazy.create (fun () -> expr)
```

The behavior of the `Lazy` type ensures that expression `expr` is evaluated on demand in
response to a `Lazy.force` operation on the lazy value.

> **Clef Note**: In Clef, `Lazy<'T>` is implemented as an extension of the flat closure architecture. A lazy value is the two-value pair `(thunk, env)`; the environment contains a computed flag, a value slot, and inlined captures, and no function address is stored in it. Forcing calls the thunk with the environment. See [Lazy Value Representation](lazy-representation.md) for complete specification.
>
> Key properties:
> - No garbage collector involvement
> - Captures computed at compile time via binding classification
> - Module-level bindings are referenced directly, not captured

### Eager Expressions

An expression of the form `eager expr` specifies an explicit, local demand
boundary. Its type is the type of `expr`; it is syntax, not a function call,
attribute, new wrapper type or whole-program evaluation mode. It has the same
operand extent as the prefix `lazy expr` form; parentheses can delimit the
operand when it appears inside an application.

`eager expr` demands `expr` to its **outer value** once at the applicable active
frontier. A scalar result is obtained; an aggregate's constructor/tag and identity
are established; a callable, lazy value or sequence is formed. It does not
recursively force ordinary fields, elements, captures or bodies. In particular,
`eager (lazy work())` constructs a lazy value without forcing its body;
`eager (Lazy.force value)` explicitly demands that force. Similarly, demanding a
sequence's outer value does not enumerate it.

The frontiers are:

| Context | Explicit demand and order |
|---|---|
| Reached ordinary binding `let x = eager expr` | Demand `expr` before continuing beyond the binding, even if `x` is never used. Bind its shared result; later uses do not replay it. |
| Activated application `f (eager expr)` | First resolve the demanded callee; then demand explicit eager actuals belonging to this declared application boundary in source order, before entering its body or returning its residual callable. Ordinary actuals remain deferred. |
| Activated aggregate constructor with a direct eager field, element or payload | Demand those explicitly marked components in source order as the constructor forms them. Ordinary sibling components remain deferred. |
| Other activated expression position | Demand the marked operand when control reaches that expression, preserving surrounding sequencing and selected control flow. |

Parentheses, `begin`/`end` grouping and type annotations around a direct eager
expression are transparent to these frontiers: `let x = (eager expr : T)` has
the same activation as `let x = eager expr`. An ordinary alias, helper function,
conditional or nested computation is not such a transparent wrapper.

There is **no recursive search or hoisting** for eager markers. If an enclosing
ordinary initializer or argument remains deferred, an eager marker inside its
nested application or branch is not reached. For example, an unused binding
`let x = if p then eager work() else value` does not force `p` or `work()`;
an unused argument in `ignore (g (eager work()))` does not activate `g`.
To activate that argument deliberately, write `ignore (eager (g (eager work())))`.
An unselected branch, uncalled function body, unforced lazy/cold computation or
unpulled sequence body likewise does not execute its contained eager marker.

Partial applications retain both completed eager values and ordinary deferred
identities. An eager actual is demanded when that partial formation is itself
activated, once for that dynamic formation; supplying remaining operands later
does not replay it. For a function-valued result, later actuals belong to the
subsequent callable's application boundary. They SHALL NOT be hoisted before
the earlier body has produced that callable. Bare aliases and pipelines retain
the same boundaries and the source order of explicit eager operands.

One dynamic marker instance demands its operand at most once. If the operand
already refers to a computed shared binding, demand reuses that result; `eager`
does not mean “recompute.” Repeated dynamic executions, such as distinct loop
iterations or factory calls, remain distinct instances unless source sharing or
a valid transformation proves otherwise.

`eager` may intentionally change whether and when effects or nontermination
occur compared with a deferred ordinary binding. The compiler SHALL retain that
demand, order, sharing and lifetime contract. It may optimize its representation
only with preservation evidence. A correct eager idiom is not automatically a
warning merely because its result is unused: effect timing may be its purpose.
Optional informational cost advice must distinguish proven demand/layout facts
from target/profile estimates and must not label a marker redundant without
proving preservation of its effects, termination and ordering. Violated admission
or correctness requirements retain their required diagnostics.

### Computation Expressions

The following expression forms are all _computation expressions_ :

```fsgrammar
expr { for ... }
expr { let ... }
expr { let! ... }
expr { use ... }
expr { while ... }
expr { yield ... }
expr { yield! ... }
expr { try ... }
expr { return ... }
expr { return! ... }
expr { match! ... }
```

More specifically, computation expressions have the following form:

```fsgrammar
builder-expr { cexpr }
```

where `cexpr` is, syntactically, the grammar of expressions with the additional constructs that are
defined in `comp-expr`. Computation expressions are used for sequences and other non-standard
interpretations of the F# expression syntax. For a fresh variable `b`, the expression

```fsgrammar
builder-expr { cexpr }
```

translates to

```fsgrammar
let b = builder-expr in {| cexpr |}C
```

The type of `b` must be a named type after the checking of builder-expr. The subscript indicates that
custom operations (`C`) are acceptable but are not required.

If the inferred type of `b` has one or more of the `Run`, `Delay`, or `Quote` methods when `builder-expr` is
checked, the translation involves those methods. For example, when all three methods exist, the
same expression translates to:

```fsgrammar
let b = builder-expr in b.Run (<@ b.Delay(fun () -> {| cexpr |}C) >@)
```

If a `Run` method does not exist on the inferred type of b, the call to `Run` is omitted. Likewise, if no
`Delay` method exists on the type of `b`, that call and the inner lambda are omitted, so the expression
translates to the following:

```fsgrammar
let b = builder-expr in b.Run (<@ {| cexpr |}C >@)
```

Similarly, if a `Quote` method exists on the inferred type of `b`, at-signs `<@ @>` are placed around `{| cexpr |}C`
or `b.Delay(fun () -> {| cexpr |}C)` if a `Delay` method also exists.

The translation `{| cexpr |}C` , which rewrites computation expressions to core language expressions,
is defined recursively according to the following rules:

`{| cexpr |}C = T (cexpr, [], fun v -> v, true)`

During the translation, we use the helper function {| cexpr |}0 to denote a translation that does not
involve custom operations:

`{| cexpr |}0 = T (cexpr, [], fun v -> v, false)`

```fsgrammar
T (e, V , C , q) where e : the computation expression being translated
                       V : a set of scoped variables
                       C : continuation (or context where “e” occurs,
                           up to a hole to be filled by the result of translating “e”)
                       q : Boolean that indicates whether a custom operator is allowed
```

Then, T is defined for each computation expression e:

**T** (let p = e in ce, **V** , **C** , q) = **T** (ce, **V**  `var` (p), v. **C** (let p = e in v), q)

**T** (let! p = e in ce, **V** , **C** , q) = **T** (ce, **V**  `var` (p), v. **C** (b.Bind( `src` (e),fun p -> v), q)

**T** (yield e, **V** , **C** , q) = **C** (b.Yield(e))

**T** (yield! e, **V** , **C** , q) = **C** (b.YieldFrom( `src` (e)))

**T** (return e, **V** , **C** , q) = **C** (b.Return(e))

**T** (return! e, **V** , **C** , q) = **C** (b.ReturnFrom( `src` (e)))

**T** (use p = e in ce, **V** , **C** , q) = **C** (b.Using(e, fun p -> {| `ce` |} 0 ))

**T** (use! p = e in ce, **V** , **C** , q) = **C** (b.Bind( `src` (e), fun p -> b.Using(p, fun p -> {| `ce` |} 0 ))

**T** (match e with pi - > cei, **V** , **C** , q) = **C** (match e with pi - > {| `ce` i |} 0 )

**T** (match! e with pi - > cei, **V** , **C** , q) = **C** (let! p = e in match p with pi - > {| `ce` i |} 0 )

**T** (while e do ce, **V** , **C** , q) = **T** (ce, **V** , v. **C** (b.While(fun () -> e, b.Delay(fun () -> v))), q)

**T** (try ce with pi - > cei, **V** , **C** , q) =
Assert(not q); **C** (b.TryWith(b.Delay(fun () -> {| `ce` |} 0 ), fun pi - > {| `ce` i |} 0 ))

**T** (try ce finally e, **V** , **C** , q) =
Assert(not q); **C** (b.TryFinally(b.Delay(fun () -> {| `ce` |} 0 ), fun () -> e))

**T** (if e then ce, **V** , **C** , q) = **T** (ce, **V** , v. **C** (if e then v else b.Zero()), q)

**T** (if e then ce1 else ce2 , **V** , **C** , q) = Assert(not q); **C** (if e then {| `ce` 1 |} 0 ) else {| `ce` 2 |} 0 )

**T** (for x = e1 to e2 do ce, **V** , **C** , q) = **T** (for x in e1 .. e2 do ce, **V** , **C** , q)

**T** (for p1 in e1 do joinOp p2 in e2 onWord (e3 `eop` e4 ) ce, **V** , **C** , q) =
Assert(q); **T** (for `pat` ( **V** ) in b.Join( `src` (e1 ), `src` (e2 ), p1 .e3 , p2 .e4 ,
p1. p2 .(p1 ,p2 )) do ce, **V** , **C** , q)

**T** (for p1 in e1 do groupJoinOp p2 in e2 onWord (e3 `eop` e4) into p3 ce, **V** , **C** , q) =
Assert(q); **T** (for `pat` ( **V** ) in b.GroupJoin( `src` (e1),
`src` (e2), p1.e3, p2.e4, p1. p3.(p1,p3)) do ce, **V** , **C** , q)

**T** (for x in e do ce, **V** , **C** , q) = **T** (ce, **V**  {x}, v. **C** (b.For( `src` (e), fun x -> v)), q)

**T** (do e in ce, **V** , **C** , q) = **T** (ce, **V** , v. **C** (e; v), q)

**T** (do! e in ce, **V** , **C** , q) = **T** (let! () = e in ce, **V** , **C** , q)

**T** (joinOp p2 in e2 on (e3 `eop` e4) ce, **V** , **C** , q) =
**T** (for `pat` ( **V** ) in **C** ({| yield `exp` ( **V** ) |}0) do join p2 in e2 onWord (e3 `eop` e4) ce, **V** , v.v, q)

**T** (groupJoinOp p2 in e2 onWord (e3 eop e4) into p3 ce, **V** , **C** , q) =
**T** (for `pat` ( **V** ) in **C** ({| yield `exp` ( **V** ) |}0) do groupJoin p2 in e2 on (e3 `eop` e4) into p3 ce,
**V** , v.v, q)

**T** ([<CustomOperator("Cop")>]cop arg, **V** , **C** , q) = Assert (q); [| cop arg, **C** (b.Yield `exp` ( **V** )) |] **V**

**T** ([<CustomOperator("Cop", MaintainsVarSpaceUsingBind=true)>]cop arg; e, **V** , **C** , q) =
Assert (q); **CL** (cop arg; e, **V** , **C** (b.Return `exp` ( **V** )), false)

**T** ([<CustomOperator("Cop")>]cop arg; e, **V** , **C** , q) =
Assert (q); **CL** (cop arg; e, **V** , **C** (b.Yield `exp` ( **V** )), false)

**T** (ce1; ce2, **V** , **C** , q) = **C** (b.Combine({| ce1 |}0, b.Delay(fun () -> {| ce2 |}0)))

**T** (do! e;, **V** , **C** , q) = **T** (let! () = `src` (e) in b.Return(), **V** , **C** , q)

**T** (e;, **V** , **C** , q) = **C** (e;b.Zero())

The following notes apply to the translations:

- The lambda expression (fun f x -> b) is represented by x.b.
- The auxiliary function var (p) denotes a set of variables that are introduced by a pattern p. For
    example:
    var(x) = {x}, var((x,y)) = {x,y} or var(S (x,y)) = {x,y}
    where S is a type constructor.
-  is an update operator for a set V to denote extended variable spaces. It updates the existing
    variables. For example, {x,y}  var((x,z)) becomes {x,y,z} where the second x replaces the
    first x.
- The auxiliary function pat ( **V** ) denotes a pattern tuple that represents a set of variables in **V**. For
    example, pat({x,y}) becomes (x,y), where x and y represent pattern expressions.
- The auxiliary function exp ( **V** ) denotes a tuple expression that represents a set of variables in **V**.
    For example, exp ({x,y}) becomes (x,y), where x and y represent variable expressions.

- The auxiliary function src (e) denotes b.Source(e) if the innermost ForEach is from the user
    code instead of generated by the translation, and a builder b contains a Source method.
    Otherwise, src (e) denotes e.
- Assert() checks whether a custom operator is allowed. If not, an error message is reported.
    Custom operators may not be used within try/with, try/finally, if/then/else, use, match, or
    sequential execution expressions such as (e1;e2). For example, you cannot use if/then/else in
    any computation expressions for which a builder defines any custom operators, even if the
    custom operators are not used.
- The operator eop denotes one of =, ?=, =? or ?=?.
- joinOp and onWord represent keywords for join-like operations that are declared in
    CustomOperationAttribute. For example, [<CustomOperator("join", IsLikeJoin=true,
    JoinConditionWord="on")>] declares “join” and “on”.
- Similarly, groupJoinOp represents a keyword for groupJoin-like operations, declared in
    CustomOperationAttribute. For example, [<CustomOperator("groupJoin",
    IsLikeGroupJoin=true, JoinConditionWord="on")>] declares “groupJoin” and “on”.
- The auxiliary translation **CL** is defined as follows:

```fsgrammar
CL (e1, V, e2, bind) where e1: the computation expression being translated
V : a set of scoped variables
e2 : the expression that will be translated after e1 is done
bind: indicator if it is for Bind (true) or iterator (false).
```

The following shows translations for the uses of CL in the preceding computation expressions:

```fsgrammar
CL (cop arg, V , e’, bind) = [| cop arg, e’ |] V
CL ([<MaintainsVariableSpaceUsingBind=true>]cop arg into p; e, V , e’, bind) =
T (let! p = e’ in e, [], v.v, true)
CL (cop arg into p; e, V , e’, bind) = T (for p in e’ do e, [], v.v, true)
CL ([<MaintainsVariableSpace=true>]cop arg; e, V , e’, bind) =
CL (e, V , [| cop arg, e’ |] V , true)
CL ([<MaintainsVariableSpaceUsingBind=true>]cop arg; e, V , e’, bind) =
CL (e, V , [| cop arg, e’ |] V , true)
CL (cop arg; e, V , e’, bind) = CL (e, [], [| cop arg, e’ |] V , false)
CL (e, V , e’, true) = T (let! pat ( V ) = e’ in e, V , v.v, true)
CL (e, V , e’, false) = T (for pat ( V ) in e’ do e, V , v.v, true)
```

- The auxiliary translation [| e1, e2 |]V is defined as follows:

[|[ e1, e2 |] **V** where e1: the custom operator available in a build
e2 : the context argument that will be passed to a custom operator
**V** : a list of bound variables

```fsgrammar
[|[<CustomOperator(" Cop")>] cop [<ProjectionParameter>] arg, e |] V =
b.Cop (e, fun pat ( V) - > arg)
[|[<CustomOperator("Cop")>] cop arg, e |] V = b.Cop (e, arg)
```

- The final two translation rules (for do! e; and do! e;) apply only for the final expression in the
    computation expression. The semicolon (;) can be omitted.

The following attributes specify custom operations:

- `CustomOperationAttribute` indicates that a member of a builder type implements a custom
    operation in a computation expression. The attribute has one parameter: the name of the
    custom operation. The operation can have the following properties:
  - `MaintainsVariableSpace` indicates that the custom operation maintains the variable space of
       a computation expression.
  - `MaintainsVariableSpaceUsingBind` indicates that the custom operation maintains the
       variable space of a computation expression through the use of a bind operation.
  - `AllowIntoPattern` indicates that the custom operation supports the use of ‘into’ immediately
       following the operation in a computation expression to consume the result of the operation.
  - `IsLikeJoin` indicates that the custom operation is similar to a join in a sequence
       computation, which supports two inputs and a correlation constraint.
  - `IsLikeGroupJoin` indicates that the custom operation is similar to a group join in a sequence
       computation, which support two inputs and a correlation constraint, and generates a group.
  - `JoinConditionWord` indicates the names used for the ‘on’ part of the custom operator for
       join-like operators.
- `ProjectionParameterAttribute` indicates that, when a custom operation is used in a
    computation expression, a parameter is automatically parameterized by the variable space of
    the computation expression.

The following examples show how the translation works. Assume the following simple sequence
builder:

```fsharp
type SimpleSequenceBuilder() =
    member __.For (source : seq<'a>, body : 'a -> seq<'b>) =
        seq { for v in source do yield! body v }
    member __.Yield (item:'a) : seq<'a> = seq { yield item }

let myseq = SimpleSequenceBuilder()
```

Then, the expression

```fsharp
myseq {
    for i in 1 .. 10 do
    yield i*i
    }
```

translates to

```fsharp
let b = myseq
b.For([1..10], fun i ->
    b.Yield(i*i))
```

`CustomOperationAttribute` allows us to define custom operations. For example, the simple sequence
builder can have a custom operator, “where”:

```fsharp
type SimpleSequenceBuilder() =
    member __.For (source : seq<'a>, body : 'a -> seq<'b>) =
        seq { for v in source do yield! body v }
    member __.Yield (item:'a) : seq<'a> = seq { yield item }
    [<CustomOperation("where")>]
    member __.Where (source : seq<'a>, f: 'a -> bool) : seq<'a> = Seq.filter f source

let myseq = SimpleSequenceBuilder()
```

Then, the expression

```fsharp
myseq {
    for i in 1 .. 10 do
    where (fun x -> x > 5)
    }
```

translates to

```fsharp
let b = myseq
    b.Where(
        b.For([1..10], fun i ->
            b.Yield (i)),
        fun x -> x > 5)
```

`ProjectionParameterAttribute` automatically adds a parameter from the variable space of the
computation expression. For example, `ProjectionParameterAttribute` can be attached to the second
argument of the `where` operator:

```fsharp
type SimpleSequenceBuilder() =
    member __.For (source : seq<'a>, body : 'a -> seq<'b>) =
        seq { for v in source do yield! body v }
    member __.Yield (item:'a) : seq<'a> = seq { yield item }
    [<CustomOperation("where")>]
    member __.Where (source: seq<'a>, [<ProjectionParameter>]f: 'a -> bool) : seq<'a> =
        Seq.filter f source

let myseq = SimpleSequenceBuilder()
```

Then, the expression

```fsharp
myseq {
    for i in 1 .. 10 do
    where (i > 5)
    }
```

translates to

```fsharp
let b = myseq
b.Where(
    b.For([1..10], fun i ->
        b.Yield (i)),
    fun i -> i > 5)
```

`ProjectionParameterAttribute` is useful when a let binding appears between `ForEach` and the
custom operators. For example, the expression

```fsharp
myseq {
    for i in 1 .. 10 do
    let j = i * i
    where (i > 5 && j < 49)
    }
```

translates to

```fsharp
let b = myseq
b.Where(
    b.For([1..10], fun i ->
        let j = i * i
        b.Yield (i,j)),
    fun (i,j) -> i > 5 && j < 49)
```

Without `ProjectionParameterAttribute`, a user would be required to write “`fun (i,j) ->`” explicitly.

Now, assume that we want to write the condition “`where (i > 5 && j < 49)`” in the following
syntax:

```fsharp
where (i > 5)
where (j < 49)
```

To support this style, the `where` custom operator should produce a computation that has the same
variable space as the input computation. That is, `j` should be available in the second `where`. The
following example uses the `MaintainsVariableSpace` property on the custom operator to specify this
behavior:

```fsharp
type SimpleSequenceBuilder() =
    member __.For (source : seq<'a>, body : 'a -> seq<'b>) =
        seq { for v in source do yield! body v }
    member __.Yield (item:'a) : seq<'a> = seq { yield item }
    [<CustomOperation("where", MaintainsVariableSpace=true)>]
    member __.Where (source: seq<'a>, [<ProjectionParameter>]f: 'a -> bool) : seq<'a> =
        Seq.filter f source

let myseq = SimpleSequenceBuilder()
```

Then, the expression

```fsharp
myseq {
    for i in 1 .. 10 do
    let j = i * i
    where (i > 5)
    where (j < 49)
    }
```

translates to

```fsharp
let b = myseq
b.Where(
    b.Where(
        b.For([1..10], fun i ->
            let j = i * i
            b.Yield (i,j)),
        fun (i,j) -> i > 5),
    fun (i,j) -> j < 49)
```

When we may not want to produce the variable space but rather want to explicitly express the chain
of the `where` operator, we can design this simple sequence builder in a slightly different way. For
example, we can express the same expression in the following way:

```fsharp
myseq {
    for i in 1 .. 10 do
    where (i > 5) into j
    where (j*j < 49)
    }
```

In this example, instead of having a let-binding (for `j` in the previous example) and passing variable
space (including `j`) down to the chain, we can introduce a special syntax that captures a value into a
pattern variable and passes only this variable down to the chain, which is arguably more readable.
For this case, `AllowIntoPattern` allows the custom operation to have an `into` syntax:

```fsharp
type SimpleSequenceBuilder() =
    member __.For (source : seq<'a>, body : 'a -> seq<'b>) =
        seq { for v in source do yield! body v }
    member __.Yield (item:'a) : seq<'a> = seq { yield item }

    [<CustomOperation("where", AllowIntoPattern=true)>]
    member __.Where (source: seq<'a>, [<ProjectionParameter>]f: 'a -> bool) : seq<'a> =
        Seq.filter f source

let myseq = SimpleSequenceBuilder()
```

Then, the expression

```fsharp
myseq {
    for i in 1 .. 10 do
    where (i > 5) into j
    where (j*j < 49)
    }
```

translates to

```fsharp
let b = myseq
b.Where(
    b.For(
        b.Where(
            b.For([1..10], fun i -> b.Yield (i))
            fun i -> i>5),
        fun j -> b.Yield (j)),
    fun j -> j*j < 49)
```

Note that the `into` keyword is not customizable, unlike `join` and `on`.

In addition to `MaintainsVariableSpace`, `MaintainsVariableSpaceUsingBind` is provided to pass
variable space down to the chain in a different way. For example:

```fsharp
type SimpleSequenceBuilder() =
    member __.For (source : seq<'a>, body : 'a -> seq<'b>) =
        seq { for v in source do yield! body v }
    member __.Return (item:'a) : seq<'a> = seq { yield item }
    member __.Bind (value , cont) = cont value
    [<CustomOperation("where", MaintainsVariableSpaceUsingBind=true, AllowIntoPattern=true)>]
    member __.Where (source: seq<'a>, [<ProjectionParameter>]f: 'a -> bool) : seq<'a> =
        Seq.filter f source

let myseq = SimpleSequenceBuilder()
```

The presence of `MaintainsVariableSpaceUsingBindAttribute` requires `Return` and `Bind` methods
during the translation.

Then, the expression

```fsharp
myseq {
    for i in 1 .. 10 do
    where (i > 5 && i*i < 49) into j
    return j
    }
```

translates to

```fsharp
let b = myseq
b.Bind(
    b.Where(B.For([1..10], fun i -> b.Return (i)),
        fun i -> i > 5 && i*i < 49),
    fun j -> b.Return (j))
```

where `Bind` is called to capture the pattern variable `j`. Note that `For` and `Yield` are called to capture
the pattern variable when `MaintainsVariableSpace` is used.

Certain properties on the `CustomOperationAttribute` introduce join-like operators. The following
example shows how to use the `IsLikeJoin` property.

```fsharp
type SimpleSequenceBuilder() =
    member __.For (source : seq<'a>, body : 'a -> seq<'b>) =
        seq { for v in source do yield! body v }
    member __.Yield (item:'a) : seq<'a> = seq { yield item }
    [<CustomOperation("merge", IsLikeJoin=true, JoinConditionWord="whenever")>]
    member __.Merge (src1:seq<'a>, src2:seq<'a>, ks1, ks2, ret) =
        seq { for a in src1 do
            for b in src2 do
            if ks1 a = ks2 b then yield((ret a ) b)
        }

let myseq = SimpleSequenceBuilder()
```

`IsLikeJoin` indicates that the custom operation is similar to a join in a sequence computation; that
is, it supports two inputs and a correlation constraint.

The expression

```fsharp
myseq {
    for i in 1 .. 10 do
    merge j in [5 .. 15] whenever (i = j)
    yield j
    }
```

translates to

```fsharp
let b = myseq
b.For(
    b.Merge([1..10], [5..15],
            fun i -> i, fun j -> j,
            fun i -> fun j -> (i,j)),
    fun j -> b.Yield (j))
```

This translation implicitly places type constraints on the expected form of the builder methods. For
example, for the `async` builder found in the `FSharp.Control` library, the translation phase
corresponds to implementing a builder of a type that has the following member signatures:

```fsharp
type AsyncBuilder with
    member For: seq<'T> * ('T -> Async<unit>) -> Async<unit>
    member Zero : unit -> Async<unit>
    member Combine : Async<unit> * Async<'T> -> Async<'T>
    member While : (unit -> bool) * Async<unit> -> Async<unit>
    member Return : 'T -> Async<'T>
    member Delay : (unit -> Async<'T>) -> Async<'T>
    member Using: 'T * ('T -> Async<'U>) -> Async<'U>
        when 'U :> IDisposable
    member Bind: Async<'T> * ('T -> Async<'U>) -> Async<'U>
    member TryFinally: Async<'T> * (unit -> unit) -> Async<'T>
    member TryWith: Async<'T> * (exn -> Async<'T>) -> Async<'T>
```

The following example shows a common approach to implementing a new computation expression
builder for a monad. The example uses computation expressions to define computations that can be
partially run by executing them step-by-step, for example, up to a time limit.

```fsharp
/// Computations that can cooperatively yield by returning a continuation
type Eventually<'T> =
    | Done of 'T
    | NotYetDone of (unit -> Eventually<'T>)

[<CompilationRepresentation(CompilationRepresentationFlags.ModuleSuffix)>]
module Eventually =

    /// The bind for the computations. Stitch 'k' on to the end of the computation.
    /// Note combinators like this are usually written in the reverse way,
    /// for example,
    /// e |> bind k
    let rec bind k e =
        match e with
        | Done x -> NotYetDone (fun () -> k x)
        | NotYetDone work -> NotYetDone (fun () -> bind k (work()))

    /// The return for the computations.
    let result x = Done x

    type OkOrException<'T> =
        | Ok of 'T
        | Exception of exn

    /// The catch for the computations. Stitch try/with throughout
    /// the computation and return the overall result as an OkOrException.
    let rec catch e =
        match e with
        | Done x -> result (Ok x)
        | NotYetDone work ->
            NotYetDone (fun () ->
                let res = try Ok(work()) with | e -> Exception e
                match res with
                | Ok cont -> catch cont // note, a tailcall
                | Exception e -> result (Exception e))

    /// The delay operator.
    let delay f = NotYetDone (fun () -> f())

    /// The stepping action for the computations.
    let step c =
        match c with
        | Done _ -> c
        | NotYetDone f -> f ()

    // The rest of the operations are boilerplate.

    /// The tryFinally operator.
    /// This is boilerplate in terms of "result", "catch" and "bind".
    let tryFinally e compensation =
        catch (e)
        |> bind (fun res ->
            compensation();
            match res with
            | Ok v -> result v
            | Exception e -> raise e)

    /// The tryWith operator.
    /// This is boilerplate in terms of "result", "catch" and "bind".
    let tryWith e handler =
        catch e
        |> bind (function Ok v -> result v | Exception e -> handler e)

    /// The whileLoop operator.
    /// This is boilerplate in terms of "result" and "bind".
    let rec whileLoop gd body =
        if gd() then body |> bind (fun v -> whileLoop gd body)
        else result ()

    /// The sequential composition operator
    /// This is boilerplate in terms of "result" and "bind".
    let combine e1 e2 =
        e1 |> bind (fun () -> e2)

    /// The using operator.
    let using (resource: #IDisposable) f =
        tryFinally (f resource) (fun () -> resource.Dispose())

    /// The forLoop operator.
    /// This is boilerplate in terms of "catch", "result" and "bind".
    let forLoop (e:seq<_>) f =
        let ie = e.GetEnumerator()
        tryFinally (whileLoop (fun () -> ie.MoveNext())
                              (delay (fun () -> let v = ie.Current in f v)))
                   (fun () -> ie.Dispose())

// Give the mapping for F# computation expressions.
type EventuallyBuilder() =
    member x.Bind(e,k) = Eventually.bind k e
    member x.Return(v) = Eventually.result v
    member x.ReturnFrom(v) = v
    member x.Combine(e1,e2) = Eventually.combine e1 e2
    member x.Delay(f) = Eventually.delay f
    member x.Zero() = Eventually.result ()
    member x.TryWith(e,handler) = Eventually.tryWith e handler
    member x.TryFinally(e,compensation) = Eventually.tryFinally e compensation
    member x.For(e:seq<_>,f) = Eventually.forLoop e f
    member x.Using(resource,e) = Eventually.using resource e

let eventually = new EventuallyBuilder()
```

After the computations are defined, they can be built by using eventually { ... }:

```fsharp
let comp =
    eventually {
        for x in 1 .. 2 do
            printfn " x = %d" x
        return 3 + 4 }
```

These computations can now be stepped. For example:

```fsharp
let step x = Eventually.step x
    comp |> step
// returns "NotYetDone <closure>"

comp |> step |> step
// prints "x = 1"
// returns "NotYetDone <closure>"

comp |> step |> step |> step |> step |> step |> step
// prints "x = 1"
// prints "x = 2"
// returns “NotYetDone <closure>”

comp |> step |> step |> step |> step |> step |> step |> step |> step
// prints "x = 1"
// prints "x = 2"
// returns "Done 7"
 
```

### Sequence Expressions

An expression in one of the following forms is a _sequence expression_ :

```fsgrammar
seq { comp-expr }
seq { short-comp-expr }
```

For example:

```fsharp
seq { for x in [ 1; 2; 3 ] do for y in [5; 6] do yield x + y }
seq { for x in [ 1; 2; 3 ] do yield x + x }
seq { for x in [ 1; 2; 3 ] -> x + x }
```

Logically speaking, sequence expressions can be thought of as computation expressions with a
builder of type `FSharp.Collections.SeqBuilder`. This type can be considered to be defined as
follows:

```fsharp
type SeqBuilder() =
    member x.Yield (v) = Seq.singleton v
    member x.YieldFrom (s:seq<_>) = s
    member x.Return (():unit) = Seq.empty
    member x.Combine (xs1,xs2) = Seq.append xs1 xs2
    member x.For (xs,g) = Seq.collect f xs
    member x.While (guard,body) = SequenceExpressionHelpers.EnumerateWhile guard body
    member x.TryFinally (xs,compensation) =
        SequenceExpressionHelpers.EnumerateThenFinally xs compensation
    member x.Using (resource,xs) = SequenceExpressionHelpers.EnumerateUsing resource xs
```

Sequence expressions are elaborated directly with behavior equivalent to this
builder; the builder itself is not a library type.

### Range Expressions

Expressions of the following forms are _range expressions_.

```fsgrammar
{ e1 .. e2 }
{ e1 .. e2 .. e3 }
seq { e1 .. e2 }
seq { e1 .. e2 .. e3 }
```

Range expressions generate sequences over a specified range. For example:

```fsgrammar
seq { 1 .. 10 } // 1; 2; 3; 4; 5; 6; 7; 8; 9; 10
seq { 1 .. 2 .. 10 } // 1; 3; 5; 7; 9
 
```

Range expressions involving `expr1 .. expr2` are translated to uses of the `(..)` operator, and those
involving `expr1 .. expr1 .. expr3` are translated to uses of the `(.. ..)` operator:

```fsgrammar
seq { e1 .. e2 } → ( .. ) e1 e2
seq { e1 .. e2 .. e3 } → ( .. .. ) e1 e2 e3
```

The default definition of these operators is in `FSharp.Core.Operators`. The ( `..` ) operator generates
an `IEnumerable<_>` for the range of values between the start (`expr1`) and finish (`expr2`) values, using
an increment of 1 (as defined by `FSharp.Core.LanguagePrimitives.GenericOne`). The `(.. ..)`
operator generates an `IEnumerable<_>` for the range of values between the start (`expr1`) and finish
(`expr3`) values, using an increment of `expr2`.

The `seq` keyword, which denotes the type of computation expression, can be omitted for simple
range expressions.

Range expressions also occur as part of the translated form of expressions, including the following:

- `[ expr1 .. expr2 ]`
- `[| expr1 .. expr2 |]`
- `for var in expr1 .. expr2 do expr3`

A sequence iteration expression of the form `for var in expr1 .. expr2 do expr3 done` is sometimes
elaborated as a simple for loop-expression ([§](expressions.md#simple-for-loop-expressions)).

### Lists via Sequence Expressions

A _list sequence expression_ is an expression in one of the following forms

```fsgrammar
[ comp-expr ]
[ short-comp-expr ]
[ range-expr ]
```

In all cases `[ cexpr ]` elaborates to `FSharp.Collections.Seq.toList(seq { cexpr })`.

For example:

```fsharp
let x2 = [ yield 1; yield 2 ]
```

```fsharp
let x3 = [ yield 1
           if Time.now().dayOfWeek = DayOfWeek.Monday then
               yield 2]
```

### Arrays Sequence Expressions

An expression in one of the following forms is an _array sequence expression_ :

```fsgrammar
[| comp-expr |]
[| short-comp-expr |]
[| range-expr |]
```

In all cases `[| cexpr |]` elaborates to `FSharp.Collections.Seq.toArray(seq { cexpr })`.

For example:

```fsharp
let x2 = [| yield 1; yield 2 |]
let x3 = [| yield 1
    if Time.now().dayOfWeek = DayOfWeek.Monday then
        yield 2 |]
```

### 'printf' Formats

Format strings are strings with `%` markers as format placeholders. Format strings are analyzed at
compile time and annotated with static and runtime type information as a result of that analysis.
They are typically used with one of the functions `printf`, `fprintf`, `sprintf`, or `bprintf` in the
`FSharp.Core.Printf` module. Format strings receive special treatment in order to type check uses of
these functions more precisely.

More concretely, a constant string is interpreted as a printf-style format string if it is expected to
have the type `FSharp.Core.PrintfFormat<'Printer,'State,'Residue,'Result,'Tuple>`. The string is
statically analyzed to resolve the generic parameters of the `PrintfFormat type`, of which `'Printer`
and `'Tuple` are the most interesting:

- `'Printer` is the function type that is generated by applying a printf-like function to the format
    string.
- `'Tuple` is the type of the tuple of values that are generated by treating the string as a generator
    (for example, when the format string is used with a function similar to `scanf` in other
    languages).

A format placeholder has the following shape:

`%[flags][width][.precision][type]`

where:

`flags` are 0 , -, +, and the space character. The # flag is invalid and results in a compile-time error.

`width` is an integer that specifies the minimum number of characters in the result.

`precision` is the number of digits to the right of the decimal point for a floating-point type..

`type` is as shown in the following table.

| Placeholder string | Type |
| --- | --- |
| `%b` | `bool` |
| `%s` | `string` |
| `%c` | `char` |
| `%d, %i` | One of the basic integer types.</br>A basic integer type is `byte`, `sbyte`, `int16`, `uint16`, `int32`, `uint32`, `int64`, `uint64`, `nativeint`, `unativeint`, or one of these types with a unit of measure |
| `%u` | Basic integer type formatted as an unsigned integer |
| `%x` | Basic integer type formatted as an unsigned hexadecimal integer with lowercase letters a through f. |
| `%X` | Basic integer type formatted as an unsigned hexadecimal integer with uppercase letters A through F. |
| `%o` | Basic integer type formatted as an unsigned octal integer. |
| `%e, %E, %f, %F, %g, %G` | `float` or `float32`, possibly with a unit of measure|
| `%M` | `decimal`, possibly with a unit of measure |
| `%O` | Any type with SRTP-resolved `ToString` member |
| `%A` | Any type with SRTP-resolved formatting |

> **Clef Note**: The `%O` and `%A` format specifiers use statically resolved type parameters rather than runtime type inspection. The type must have appropriate formatting members resolvable at compile time. See [Native Type Mappings](native-type-mappings.md#the-universal-base-type-obj-is-not-available).
| `%a` | Formatter of type `'State -> 'T -> 'Residue` for a fresh variable type `'T` |
| `%t` | Formatter of type `'State -> 'Residue` |

For example, the format string "`%s %d %s`" is given the type `PrintfFormat<(string -> int -> string -> 'd), 'b, 'c, 'd, (string * int * string)>` for fresh variable types `'b`, `'c`, `'d`. Applying `printf`
to it yields a function of type `string -> int -> string -> unit`.

## Application Expressions

### Basic Application Expressions

Application expressions involve variable names, dot-notation lookups, function applications, method
applications, type applications, and item lookups, as shown in the following table.

| Expression | Description |
| --- | --- |
| `long-ident-or-op` | Long-ident lookup expression |
| `expr '.' long-ident-or-op` | Dot lookup expression |
| `expr expr` | Function or member application expression |
| `expr(expr)` | High precedence function or member application expression |
| `expr<types>` | Type application expression |
| `expr< >` | Type application expression with an empty type list |
| `type expr` | Simple object expression |

The following are examples of application expressions:

```fsharp
Math.PI
Math.PI.ToString()
(3 + 4).ToString()
Environment.getVariable("PATH").Length
Console.WriteLine("Hello World")
```

Application expressions may start with object construction expressions that do not include the `new`
keyword:

```fsharp
MyRecord()
List<int>(10)
KeyValuePair(3,"Three")
MyType().GetProperty()
Map<int,int>.empty.[1]
```

If the `long-ident-or-op` starts with the special pseudo-identifier keyword `global`, F# resolves the
identifier with respect to the global namespace; that is, ignoring all `open` directives (see [§](inference-application-resolution.md#resolving-application-expressions)). For example:

```fsharp
global.Math.PI
```

is resolved to `Math.PI` ignoring all `open` directives.

The checking of application expressions is described in detail as an algorithm in [§](inference-application-resolution.md#resolving-application-expressions). To check an
application expression, the expression form is repeatedly decomposed into a _lead_ expression `expr`
and a list of projections `projs` through the use of _Unqualified Lookup_ ([§](inference-application-resolution.md#unqualified-lookup)). This in turn uses
procedures such as _Expression-Qualified Lookup_ and _Method Application Resolution_.

As described in [§](inference-application-resolution.md#resolving-application-expressions), checking an application expression results in an elaborated expression that
contains a series of lookups and method calls. The elaborated expression may include:

- Uses of named values
- Uses of union cases
- Record constructions
- Applications of functions
- Applications of methods (including methods that access properties)
- Applications of object constructors
- Uses of fields, both static and instance
- Uses of active pattern result elements

Additional constructs may be inserted when resolving method calls into simpler primitives:

- The use of a method or value as a first-class function may result in a function expression.

    For example, `Environment.getVariable` elaborates to:
    `(fun v -> Environment.getVariable(v))`
    for some fresh variable `v`.

- The use of post-hoc property setters results in the insertion of additional assignment and
    sequential execution expressions in the elaborated expression.

    For example, `new MyClass(Name="Text")` elaborates to
    `let v = new MyClass() in v.set_Name("Text"); v`
    for some fresh variable `v`.

- The use of optional arguments results in the insertion of `Some(_)` and `None` data constructions in
    the elaborated expression.

For uses of active pattern results (see [§](namespaces-and-modules.md#active-pattern-definitions-in-modules)), for result `i` in an active pattern that has `N` possible
results of types `types` , the elaborated expression form is a union case `ChoiceNOfi` of type
`FSharp.Core.Choice<types>`.

### Object Construction Expressions

An expression of the following form is an _object construction expression_:

```fsgrammar
new ty ( e1 ... en )
```

An object construction expression constructs a new instance of a type, usually by calling a
constructor method on the type. For example:

```fsharp
new MyClass()
new Queue<int>()
new ConfigOptions(Verbose=true)
new 'T()
```

The initial type of the expression is first asserted to be equal to `ty`. The type `ty` must not be an array,
record, union or tuple type. If `ty` is a named class or struct type:

- `ty` must not be abstract.
- If `ty` is a struct type, `n` is 0 , and `ty` does not have a constructor method that takes zero
    arguments, the expression elaborates to the default “zero-bit pattern” value for `ty`.
- Otherwise, the type must have one or more accessible constructors. The overloading between
    these potential constructors is resolved and elaborated by using _Method Application Resolution_
    (see [§](inference-application-resolution.md#method-application-resolution)).

If `ty` is a type variable:

- There must be no arguments (that is, `n = 0`).
- The type variable is constrained as follows:

    `ty : (new : unit -> ty )` -- default constructor constraint

- The expression elaborates to a call to the parameterless constructor of the concrete type that `ty` is instantiated with.

### Operator Expressions

Operator expressions are specified in terms of their shallow syntactic translation to other constructs.
The following translations are applied in order:

```fsgrammar
infix-or-prefix-op e1 → (~infix-or-prefix-op) e1
prefix-op e1 → (prefix-op) e1
e1 infix-op e2 → (infix-op) e1 e2
```

> Note: When an operator that may be used as either an infix or prefix operator is used in
prefix position, a tilde character ~ is added to the name of the operator during the
translation process.

These rules are applied after applying the rules for dynamic operators ([§](expressions.md#dynamic-operator-expressions)).

The parenthesized operator name is then treated as an identifier and the standard rules for
unqualified name resolution ([§](inference-name-resolution.md#name-resolution)) in expressions are applied. The expression may resolve to a
specific definition of a user-defined or library-defined operator. For example:

```fsharp
let (+++) a b = (a,b)
3 +++ 4
```

In some cases, the operator name resolves to a standard definition of an operator from the F#
library. For example, in the absence of an explicit definition of (+),

```fsharp
3 + 4
```

resolves to a use of the infix operator FSharp.Core.Operators.(+).

Some operators that are defined in the F# library receive special treatment in this specification. In
particular:

- The `&expr` address-of operator ([§](expressions.md#the-addressof-operator))
- The `expr && expr` and `expr || expr` shortcut control flow operators ([§](expressions.md#shortcut-operator-expressions))
- The `%expr` and `%%expr` expression splice operators in quotations ([§](expressions.md#expression-splices))
- The library-defined operators, such as `+`, `-`, `*`, `/`, `%`, `**`, `<<<`, `>>>`, `&&&`, `|||`, and `^^^`.

If the operator does not resolve to a user-defined or library-defined operator, the name resolution
rules ([§](inference-name-resolution.md#name-resolution)) ensure that the operator resolves to an expression that implicitly uses a static member
invocation expression (§ ?) that involves the types of the operands. This means that the effective
behavior of an operator that is not defined in the F# library is to require a static member that has the
same name as the operator, on the type of one of the operands of the operator. In the following
code, the otherwise undefined operator `-->` resolves to the static member on the `Receiver` type,
based on a type-directed resolution:

```fsharp
type Receiver(latestMessage:string) =
    static member (<--) (receiver:Receiver,message:string) =
        Receiver(message)

    static member (-->) (message,receiver:Receiver) =
        Receiver(message)

let r = Receiver "no message"

r <-- "Message One"

"Message Two" --> r
```

### Dynamic Operator Expressions

Expressions of the following forms are _dynamic operator expressions:_

```fsgrammar
expr1 ? expr2
expr1 ? expr2 <- expr3
```

These expressions are defined by their syntactic translation:

`expr ? ident` → `(?) expr "ident"`

`expr1 ? (expr2)` → `(?) expr1 expr2`

`expr1 ? ident <- expr2` → `(?<-) expr1 "ident" expr2`

`expr1 ? (expr2) <- expr3` → `(?<-) expr1 expr2 expr3`

Here `"ident"` is a string literal that contains the text of `ident`.

> Note: The F# core library `FSharp.Core.dll` does not define the `(?)` and `(?<-)` operators.
However, user code may define these operators. For example, it is common to define
the operators to perform a dynamic lookup on the properties of an object by using
reflection.

This syntactic translation applies regardless of the definition of the `(?)` and `(?<-)` operators.
However, it does not apply to uses of the parenthesized operator names, as in the following:

```fsharp
(?) x y
```

### The AddressOf Operator

Under default definitions, an expression of the following form is a _byref-address-of expression:_

```fsgrammar
& expr
```

Such an expression takes the address of a mutable local variable, byref-valued argument, field, array
element, or static mutable global variable.

For `&expr`, the initial type of the overall expression must be of the form `byref<ty>`, and the
expression `expr` is checked with initial type `ty`.

The overall expression is elaborated recursively by taking the address of the elaborated form of `expr`,
written `AddressOf(expr, DefinitelyMutates)`, defined in [§](expressions.md#taking-the-address-of-an-elaborated-expression).

In general, use of this operator is recommended only:

- To pass addresses where `byref` parameters are expected.
- To pass a `byref` parameter on to a subsequent function.

Direct uses of `byref` types require careful attention to memory safety. In particular, `byref` types may NOT be used within named types such as tuples or function types.

> Note: The rules in this section apply to the following prefix operator, which is defined
in the F# core library for use with one argument.
<br>`FSharp.Core.LanguagePrimitives.IntrinsicOperators.(~&)`
<br>Other uses of this operator are not permitted.

### Lookup Expressions

Lookup expressions are specified by syntactic translation:

`e1.[eargs]` → `e1.get_Item(eargs)`

`e1.[eargs] <- e3` → `e .set_Item(eargs, e3)`

In addition, for the purposes of resolving expressions of this form, array types of rank 1, 2, 3, and 4
are assumed to support a type extension that defines an `Item` property that has the following
signatures:

```fsharp
type 'T[] with
    member arr.Item : int -> 'T

type 'T[,] with
    member arr.Item : int * int -> 'T

type 'T[,,] with
    member arr.Item : int * int * int -> 'T

type 'T[,,,] with
    member arr.Item : int * int * int * int -> 'T
```

In addition, if type checking determines that the type of `e1` is a named type that supports the
`DefaultMember` attribute, then the member name identified by the `DefaultMember` attribute is used
instead of Item.

### Slice Expressions

Slice expressions are defined by syntactic translation:

`e1.[sliceArg1, ,,, sliceArgN]` → `e1.GetSlice(args1, ..., argsN)`

`e1.[sliceArg1, ,,, sliceArgN] <- expr` → `e1.SetSlice(args1, ...,argsN, expr)`

where each `sliceArgN` is one of the following and translated to `argsN` (giving one or two args) as
indicated

`*` → `None, None`

`e1..` → `Some e1, None`

`..e2` → `None, Some e2`

`e1..e2` → `Some e1, Some e2`

`idx` → `idx`

Because this is a shallow syntactic translation, the `GetSlice` and `SetSlice` name may be resolved by
any of the relevant _Name Resolution_ ([§](inference-name-resolution.md#name-resolution)) techniques, including defining the method as a type
extension for an existing type.

For example, if a matrix type has the appropriate overloads of the GetSlice method (see below), it is
possible to do the following:

```fsharp
matrix.[1..,*] // get rows 1.. from a matrix (returning a matrix)
matrix.[1..3,*] // get rows 1..3 from a matrix (returning a matrix)
matrix.[*,1..3] // get columns 1..3from a matrix (returning a matrix)
matrix.[1..3,1,.3] // get a 3x3 sub-matrix (returning a matrix)
matrix.[3,*] // get row 3 from a matrix as a vector
matrix.[*,3] // get column 3 from a matrix as a vector
 
```

In addition, CIL array types of rank 1 to 4 are assumed to support a type extension that defines a
method `GetSlice` that has the following signature:

```fsharp
type 'T[] with
    member arr.GetSlice : ?start1:int * ?end1:int -> 'T[]
type 'T[,] with
    member arr.GetSlice : ?start1:int * ?end1:int * ?start2:int * ?end2:int -> 'T[,]
    member arr.GetSlice : idx1:int * ?start2:int * ?end2:int -> 'T[]
    member arr.GetSlice : ?start1:int * ?end1:int * idx2:int - > 'T[]
type 'T[,,] with
    member arr.GetSlice : ?start1:int * ?end1:int * ?start2:int * ?end2:int *
                          ?start3:int * ?end3:int
                            -> 'T[,,]
type 'T[,,,] with
    member arr.GetSlice : ?start1:int * ?end1:int * ?start2:int * ?end2:int *
                          ?start3:int * ?end3:int * ?start4:int * ?end4:int
                            -> 'T[,,,]
```

In addition, CIL array types of rank 1 to 4 are assumed to support a type extension that defines a
method `SetSlice` that has the following signature:

```fsharp
type 'T[] with
    member arr.SetSlice : ?start1:int * ?end1:int * values:T[] -> unit

type 'T[,] with
    member arr.SetSlice : ?start1:int * ?end1:int * ?start2:int * ?end2:int *
                          values:T[,] -> unit
    member arr.SetSlice : idx1:int * ?start2:int * ?end2:int * values:T[] -> unit
    member arr.SetSlice : ?start1:int * ?end1:int * idx2:int * values:T[] -> unit

type 'T[,,] with
    member arr.SetSlice : ?start1:int * ?end1:int * ?start2:int * ?end2:int *
                          ?start3:int * ?end3:int *
                          values:T[,,] -> unit

type 'T[,,,] with
    member arr.SetSlice : ?start1:int * ?end1:int * ?start2:int * ?end2:int *
                          ?start3:int * ?end3:int * ?start4:int * ?end4:int *
                          values:T[,,,] -> unit
```

### Member Constraint Invocation Expressions

An expression of the following form is a member constraint invocation expression:

```fsgrammar
(static-typars : (member-sig) expr)
```

Type checking proceeds as follows:

1. The expression is checked with initial type `ty`.
2. A statically resolved member constraint is applied ([§](types-and-type-constraints.md#member-constraints)):
    <br>`static-typars: (member-sig)`
3. `ty` is asserted to be equal to the return type of the constraint.
4. `expr` is checked with an initial type that corresponds to the argument types of the constraint.

The elaborated form of the expression is a member invocation. For example:

```fsharp
let inline speak (a: ^a) =
    let x = (^a : (member Speak: unit -> string) (a))
    printfn "It said: %s" x
    let y = (^a : (member MakeNoise: unit -> string) (a))
    printfn "Then it went: %s" y

type Duck() =
    member x.Speak() = "I'm a duck"
    member x.MakeNoise() = "quack"
type Dog() =
    member x.Speak() = "I'm a dog"
    member x.MakeNoise() = "grrrr"

let x = new Duck()
let y = new Dog()
speak x
speak y
```

Outputs:

```fsother
It said: I'm a duck
Then it went: quack
It said: I'm a dog
Then it went: grrrr
```

### Assignment Expressions

An expression of the following form is an _assignment expression_ :

```fsharp
expr1 <- expr2
```

A modified version of _Unqualified Lookup_ ([§](inference-application-resolution.md#unqualified-lookup)) is applied to the expression `expr1` using a fresh
expected result type `ty` , thus producing an elaborate expression `expr1`. The last qualification for `expr1`
must resolve to one of the following constructs:

- An invocation of a property with a setter method. The property may be an indexer.

    Type checking incorporates `expr2` as the last argument in the method application resolution for
    the setter method. The overall elaborated expression is a method call to this setter property and
    includes the last argument.

- A mutable value `path` of type `ty`.

    Type checking of `expr2` uses the expected result type `ty` and generates an elaborated expression
    `expr2`. The overall elaborated expression is an assignment to a value reference `&path <-stobj expr2`.

- A reference to a value `path` of type `byref<ty>`.

    Type checking of `expr2` uses the expected result type `ty` and generates an elaborated expression
    `expr2`. The overall elaborated expression is an assignment to a value reference `path <-stobj expr2`.

- A reference to a mutable field `expr1a.field` with the actual result type `ty`.

    Type checking of `expr2` uses the expected result type `ty` and generates an elaborated expression
    `expr2`. The overall elaborated expression is an assignment to a field (see [§](expressions.md#taking-the-address-of-an-elaborated-expression)):

    `AddressOf(expr1a.field, DefinitelyMutates) <-stobj expr2`

- A array lookup `expr1a.[expr1b]` where `expr1a` has type `ty[]`.

    Type checking of expr2 uses the expected result type ty and generates thean elaborated
    expression expr2. The overall elaborated expression is an assignment to a field (see [§](expressions.md#taking-the-address-of-an-elaborated-expression)):

    `AddressOf(expr1a.[expr1b], DefinitelyMutates) <-stobj expr2`

    > Note: Because assignments have the preceding interpretations, local values must be
    mutable so that primitive field assignments and array lookups can mutate their
    immediate contents. In this context, “immediate” contents means the contents of a
    mutable value type. For example, given

    ```fsharp
    [<Struct>]
    type SA =
        new(v) = { x = v }
        val mutable x : int

    [<Struct>]
    type SB =
        new(v) = { sa = v }
        val mutable sa : SA

    let s1 = SA(0)
    let mutable s2 = SA(0)
    let s3 = SB(0)
    let mutable s4 = SB(0)
    ```

    > Then these are not permitted:

    ```fsharp
    s1.x <- 3
    s3.sa.x <- 3
    ```

    and these are:

    ```fsharp
    s2.x <- 3
    s4.sa.x <- 3
    s4.sa <- SA(2)
    ```

## Control Flow Expressions

### Parenthesized and Block Expressions

A _parenthesized expression_ has the following form:

```fsgrammar
(expr)
```

A _block expression_ has the following form:

```fsgrammar
begin expr end
```

The expression `expr` is checked with the same initial type as the overall expression.

The elaborated form of the expression is simply the elaborated form of `expr`.

### Sequential Execution Expressions

A _sequential execution expression_ has the following form:

```fsgrammar
expr1 ; expr2
```

For example:

```fsharp
printfn "Hello"; printfn "World"; 3
```

The `;` token is optional when both of the following are true:

- The expression `expr2` occurs on a subsequent line that starts in the same column as `expr1`.

- The current pre-parse context that results from the syntax analysis of the program text is a
    `SeqBlock` ([§](lexical-filtering.md#lexical-filtering)).

When the semicolon is optional, parsing inserts a `$sep` token automatically and applies an additional
syntax rule for lightweight syntax ([§](lexical-filtering.md#basic-lightweight-syntax-rules-by-example)). In practice, this means that code can omit the `;` token
for sequential execution expressions that implement functions or immediately follow tokens such as
`begin` and `(`.

The expression `expr1` is checked with an arbitrary initial type `ty`. After checking `expr1`, `ty` is asserted
to be equal to `unit`. If the assertion fails, a warning rather than an error is reported. The expression
`expr2` is then checked with the same initial type as the overall expression.

Sequential execution expressions are a primitive elaborated form.

### Conditional Expressions

A _conditional expression_ has the following forms

```fsgrammar
if expr1a then expr1b
elif expr3a then expr2b
...
elif exprna then exprnb
else exprlast
```

The `elif` and `else` branches may be omitted. For example:

```fsharp
if (1 + 1 = 2) then "ok" else "not ok"
if (1 + 1 = 2) then printfn "ok"
```

Conditional expressions are equivalent to pattern matching on Boolean values. For example, the
following expression forms are equivalent:

```fsgrammar
if expr1 then expr2 else expr3
match (expr1: bool) with true -> expr2 | false -> expr3
```

If the `else` branch is omitted, the expression is a _sequential conditional expression_ and is equivalent
to:

```fsgrammar
match (expr1: bool) with true -> expr2 | false -> ()
```

with the exception that the initial type of the overall expression is first asserted to be `unit`.

### Shortcut Operator Expressions

Under default definitions, expressions of the following form are respectively an _shortcut and expression_ and a _shortcut or expression_ :

```fsgrammar
expr && expr
expr || expr
```

These expressions are defined by their syntactic translation:

```fsgrammar
expr1 && expr2 → if expr1 then expr2 else false
expr1 || expr2 → if expr1 then true else expr2
```

These are the demand laws of the Clef intrinsic operators, including stored,
aliased and partially applied uses. Such uses SHALL retain the shared deferred
right operand and the same short-circuit behavior. Source lexical resolution
distinguishes a user-defined operator from these intrinsics; application shape
does not silently select a strict two-argument variant.

### Pattern-Matching Expressions and Functions

A _pattern-matching expression_ has the following form:

```fsgrammar
match expr with rules
```

Pattern matching is used to evaluate the given expression and select a rule ([§](patterns.md#patterns)). For example:

```fsharp
match (3, 2) with
| 1, j -> printfn "j = %d" j
| i, 2 - > printfn "i = %d" i
| _ - > printfn "no match"
```

A _pattern-matching function_ is an expression of the following form:

```fsgrammar
function rules
```

A pattern-matching function is syntactic sugar for a single-argument function expression that is
followed by immediate matches on the argument. For example:

```fsharp
function
| 1, j -> printfn "j = %d" j
| _ - > printfn "no match"
```

is syntactic sugar for the following, where x is a fresh variable:

```fsharp
fun x ->
    match x with
    | 1, j -> printfn "j = %d" j
    | _ - > printfn "no match"
```

### Sequence Iteration Expressions

An expression of the following form is a _sequence iteration expression_ :

```fsgrammar
for pat in expr1 do expr2 done
```

The done token is optional if `expr2` appears on a later line and is indented from the column position
of the for token. In this case, parsing inserts a `$done` token automatically and applies an additional
syntax rule for lightweight syntax ([§](lexical-filtering.md#basic-lightweight-syntax-rules-by-example)).

For example:

```fsharp
for x, y in [(1, 2); (3, 4)] do
    printfn "x = %d, y = %d" x y
```

The expression `expr1` is checked with a fresh initial type `tyexpr`, which is then asserted to be a subtype
of type `IEnumerable<ty>`, for a fresh type `ty`. If the assertion succeeds, the expression elaborates to
the following, where `v` is of type `IEnumerator<ty>` and `pat` is a pattern of type `ty` :

```fsharp
let v = expr1.GetEnumerator()
try
    while (v.MoveNext()) do
        match v.Current with
        | pat - > expr2
        | _ -> ()
finally
    match box(v) with
    | :? IDisposable as d -> d.Dispose()
    | _ -> ()
```

If the assertion fails, the type `tyexpr` may also be of any static type that satisfies the "collection
pattern" of the native library. If so, the _enumerable extraction_ process is used to enumerate the type. In
particular, `tyexpr` may be any type that has an accessible GetEnumerator method that accepts zero
arguments and returns a value that has accessible MoveNext and Current properties. The type of `pat`
is the same as the return type of the Current property on the enumerator value.

A sequence iteration of the form

```fsgrammar
for var in expr1 .. expr2 do expr3 done
```

where the type of `expr1` or `expr2` is equivalent to `int`, is elaborated as a simple for-loop expression
([§](expressions.md#simple-for-loop-expressions))

### Simple for-Loop Expressions

An expression of the following form is a _simple for loop expression_ :

```fsgrammar
for var = expr1 to expr2 do expr3 done
```

The `done` token is optional when `e2` appears on a later line and is indented from the column position
of the `for` token. In this case, a `$done` token is automatically inserted, and an additional syntax rule
for lightweight syntax applies ([§](lexical-filtering.md#basic-lightweight-syntax-rules-by-example)). For example:

```fsharp
for x = 1 to 30 do
    printfn "x = %d, x^2 = %d" x (x*x)
```

The bounds `expr1` and `expr2` are checked with initial type `int`. The overall type of the expression is
`unit`. A warning is reported if the body `expr3` of the `for` loop does not have static type `unit`.

The identifier introduced by an integer for-loop SHALL be an immutable source binding,
established afresh for each execution of the body with that iteration's value. Assignment
to it is a compile-time error. Closures retain that iteration's value under
[Closure Representation §2.2](closure-representation.md#22-capture-semantics); later
induction updates SHALL NOT change an earlier capture. An internal mutable counter
used in elaboration is distinct from the source binding and inaccessible through it.
Nested loops introduce distinct bindings even when their identifiers have the same
spelling. This rule also applies to integer range syntax elaborated as a simple
for-loop. Existing closure placement and lifetime obligations still apply.

The following shows the elaborated form of a simple for-loop expression for fresh variables `start`
and `finish`:

```fsharp
let start = expr1 in
let finish = expr2 in
for var = start to finish do expr3 done
```

For-loops over ranges that are specified by variables are a primitive elaborated form. When
executed, the iterated range includes both the starting and ending values in the range, with an
increment of 1.

An expression of the form

```fsgrammar
for var in expr1 .. expr2 do expr3 done
```

is always elaborated as a simple for-loop expression whenever the type of `expr1` or `expr2` is
equivalent to `int`.

### While Expressions

A _while loop expression_ has the following form:

```fsgrammar
while expr1 do expr2 done
```

The `done` token is optional when `expr2` appears on a subsequent line and is indented from the
column position of the `while`. In this case, a `$done` token is automatically inserted, and an additional
syntax rule for lightweight syntax applies ([§](lexical-filtering.md#basic-lightweight-syntax-rules-by-example)).

For example:

```fsharp
while Time.today().dayOfWeek = DayOfWeek.Monday do
    printfn "I don't like Mondays"
```

The overall type of the expression is `unit`. The expression `expr1` is checked with initial type `bool`. A
warning is reported if the body `expr2` of the while loop cannot be asserted to have type `unit`.

### Try-with Expressions

A _try-with expression_ has the following form:

```fsgrammar
try expr with rules
```

For example:

```fsharp
try "1" with _ -> "2"

try
    failwith "fail"
with
    | Failure msg -> "caught"
    | :? InvalidOperationException -> "unexpected"
```

Expression `expr` is checked with the same initial type as the overall expression. The pattern matching
clauses are then checked with the same initial type and with input type `exn`.

Try-with expressions are a primitive elaborated form.

### Reraise Expressions

A _reraise expression_ is an application of the `reraise` F# library function. This function must be
applied to an argument and can be used only on the immediate right-hand side of `rules` in a try-with
expression.

```fsharp
try
    failwith "fail"
with e -> printfn "Failing"; reraise()
```

> Note: The rules in this section apply to any use of the function
  `FSharp.Core.Operators.reraise`, which is defined in the F# core library.

When executed, `reraise()` continues exception processing with the original exception information.

### Try-finally Expressions

A _try-finally expression_ has the following form:

```fsgrammar
try expr1 finally expr2
```

For example:

```fsharp
try "1" finally printfn "Finally!"

try
    failwith "fail"
finally
    printfn "Finally block"
```

Expression `expr1` is checked with the initial type of the overall expression. Expression `expr2` is
checked with arbitrary initial type, and a warning occurs if this type cannot then be asserted to be
equal to `unit`.

Try-finally expressions are a primitive elaborated form.

### Assertion Expressions

An _assertion expression_ has the following form:

```fsgrammar
assert expr
```

The expression `assert expr` evaluates `expr` and triggers a runtime assertion failure if the result is `false`.

> **Clef Note**: In Clef, assertions are controlled by the `DEBUG` conditional compilation symbol. When disabled, assertion expressions are elided entirely. When enabled, a failed assertion causes program termination with diagnostic output.

## Definition Expressions

A _definition expression_ has one of the following forms:

```fsgrammar
let function-defn in expr
let value-defn in expr
let rec function-or-value-defns in expr
use ident = expr1 in expr
```

Such an expression establishes a local function or value definition within the lexical scope of `expr`
and has the same overall type as `expr`.

In each case, the `in` token is optional if `expr` appears on a subsequent line and is aligned with the
token `let`. In this case, a `$in` token is automatically inserted, and an additional syntax rule for
lightweight syntax applies ([§](lexical-filtering.md#basic-lightweight-syntax-rules-by-example))

For example:

```fsharp
let x = 1
x + x
```

and

```fsharp
let x, y = ("One", 1)
x.Length + y
```

and

```fsharp
let id x = x in (id 3, id "Three")
```

and

```fsharp
let swap (x, y) = (y,x)
List.map swap [ (1, 2); (3, 4) ]
```

and

```fsharp
let K x y = x in List.map (K 3) [ 1; 2; 3; 4 ]
```

Function and value definitions in expressions are similar to function and value definitions in class
definitions ([§](type-definitions.md#class-type-definitions)), modules ([§](namespaces-and-modules.md#function-and-value-definitions-in-modules)), and computation expressions ([§](expressions.md#computation-expressions)), with the following
exceptions:

- Function and value definitions in expressions may not define explicit generic parameters ([§](types-and-type-constraints.md#type-parameter-definitions)).
    For example, the following expression is rejected:
       <br>`let f<'T> (x:'T) = x in f 3`
- Function and value definitions in expressions are not public and are not subject to arity analysis
    ([§](inference-supplementary.md#arity-inference)).
- Any custom attributes that are specified on the declaration, parameters, and/or return
    arguments are ignored and result in a warning. As a result, function and value definitions in
    expressions may not have the `ThreadStatic` or `ContextStatic` attribute.

### Value Definition Expressions

A value definition expression has the following form:

```fsgrammar
let value-defn in expr
```

where _value-defn_ has the form:

```fsgrammar
mutable? access? pat typar-defns? return-type? = rhs-expr
```

Checking proceeds as follows:

1. Check the _value-defn_ ([§](inference-constraint-solving.md#checking-and-elaborating-function-value-and-member-definitions)), which defines a group of identifiers `identj` with inferred types `tyj`

2. Add the identifiers `identj` to the name resolution environment, each with corresponding type
    `tyj`.
3. Check the body `expr` against the initial type of the overall expression.

In this case, the following rules apply:

- If `pat` is a single value pattern `ident`, the resulting elaborated form of the entire expression is

    ```fsgrammar
    let ident1 <typars1> = expr1 in
    body-expr
    ```

    where ident1 , typars1 and expr1 are defined in [§](inference-constraint-solving.md#checking-and-elaborating-function-value-and-member-definitions).

- Otherwise, the resulting elaborated form of the entire expression is

    ```fsgrammar
    let tmp <typars1 ... typars n> = expr in
    let ident1 <typars1> = expr1 in
    ...
    let identn <typarsn> = exprn in
    body-expr
    ```

    where `tmp` is a fresh identifier and `identi`, `typarsi`, and `expri` all result from the compilation of
    the pattern `pat` ([§](patterns.md#patterns)) against the input `tmp`.

Value definitions in expressions may be marked as `mutable`. For example:

```fsharp
let mutable v = 0
while v < 10 do
    v <- v + 1
    printfn "v = %d" v
```

Such variables are implicitly dereferenced each time they are used.

### Function Definition Expressions

A function definition expression has the form:

```fsgrammar
let function-defn in expr
```

where `function-defn` has the form:

```fsgrammar
inline? access? ident-or-op typar-defns? pat1 ... patn return-type? = rhs-expr
```

Checking proceeds as follows:

1. Check the `function-defn` ([§](inference-constraint-solving.md#checking-and-elaborating-function-value-and-member-definitions)), which defines `ident1`, `ty1`, `typars1` and `expr1`
2. Add the identifier `ident1` to the name resolution environment, each with corresponding type `ty1`.
3. Check the body `expr` against the initial type of the overall expression.

The resulting elaborated form of the entire expression is

```fsgrammar
let ident1 < typars1 > = expr1 in
expr
```

where `ident1` , `typars1` and `expr1` are as defined in [§](inference-constraint-solving.md#checking-and-elaborating-function-value-and-member-definitions).

### Recursive Definition Expressions

An expression of the following form is a _recursive definition expression_:

```fsgrammar
let rec function-or-value-defns in expr
```

The defined functions and values are available for use within their own definitions; that is, they can be
used within any of the expressions on the right-hand side of `function-or-value-defns`. Multiple
functions or values may be defined by using `let rec ... and ...`. For example:

```fsharp
let test() =
    let rec twoForward count =
        printfn "at %d, taking two steps forward" count
        if count = 1000 then "got there!"
        else oneBack (count + 2)
    and oneBack count =
        printfn "at %d, taking one step back " count
        twoForward (count - 1)

    twoForward 1

test()
```

In the example, the expression defines a set of recursive functions. If one or more recursive values
are defined, the recursive expressions are analyzed for safety ([§](inference-constraint-solving.md#recursive-safety-analysis)). This may result in warnings
(including some reported as compile-time errors) and runtime checks.

#### Nested Recursive Functions and Captures

> **Clef Note**: When a recursive function is defined inside another function, it may capture variables from the enclosing scope. These captures must be tracked and propagated to code generation.

```fsharp
let sumTo (n: int) : int =
    let rec loop acc i =
        if i > n then acc    // 'n' is captured from enclosing scope
        else loop (acc + i) (i + 1)
    loop 0 1
```

In this example, the nested `loop` function captures `n` from `sumTo`. The capture analysis (see [Closure Representation §3.2](closure-representation.md#32-capture-analysis)) SHALL identify `n` as a captured variable for `loop`, even though `loop` is a named recursive binding rather than an anonymous lambda.

Contrast with:
```fsharp
let factorialTail (n: int) : int =
    let rec loop acc n =     // 'n' is a parameter, shadows outer 'n'
        if n <= 1 then acc
        else loop (acc * n) (n - 1)
    loop 1 n
```

Here, `loop` has its own parameter `n` that shadows the outer `n`, so no capture occurs.

**Calling Convention**: Nested named functions use **parameter-passing** for captures rather than the closure struct model (see [Closure Representation §8](closure-representation.md#8-nested-named-functions-vs-escaping-closures)). Captures are prepended as additional function parameters:

```
// Source: let rec loop acc i = if i > n then acc else ...
// Generated signature: loop(n: int, acc: int, i: int) -> int
//                           ↑ capture   ↑ explicit parameters
 
```

At call sites, the enclosing scope supplies capture values directly:
```fsharp
loop 0 1      // Source syntax
loop(n, 0, 1) // Generated call (n passed as first argument)
 
```

**Implementation Note**: Named function bindings that are nested (i.e., defined within another function) SHALL have capture analysis performed. Top-level function bindings never capture because there is no enclosing scope from which to capture.

### Deterministic Disposal Expressions

A _deterministic disposal expression_ has the form:

```fsgrammar
use ident = expr1 in expr2
```

For example:

```fsharp
use inStream = File.openText "input.txt"
let line1 = inStream.ReadLine()
let line2 = inStream.ReadLine()
(line1,line2)
```

The expression is first checked as an expression of form `let ident = expr1 in expr2` ([§](expressions.md#value-definition-expressions)), which results in an elaborated expression of the following form:

```fsgrammar
let ident1 : ty1 = expr1 in expr2.
```

Only one value may be defined by a deterministic disposal expression, and the definition is not
generalized ([§](inference-constraint-solving.md#generalization)). The type `ty1` , is then asserted to be a subtype of `IDisposable`. The `Dispose` method is called on the value when the value goes out of scope. Thus the overall expression elaborates to this:

```fsgrammar
let ident1 : ty1 = expr1
try expr2
finally (ident :> IDisposable).Dispose()
```

> **Clef Note**: In Clef, `use` bindings provide deterministic resource cleanup. Clef is null-free by construction, so disposal code requires no null checks.

## Type-related Expressions

### Type-Annotated Expressions

A _type-annotated expression_ has the following form, where `ty` indicates the static type of `expr`:

```fsgrammar
expr : ty
```

For example:

```fsharp
(1 : int)
let f x = (x : string) + x
```

When checked, the initial type of the overall expression is asserted to be equal to `ty`. Expression `expr`
is then checked with initial type `ty`. The expression elaborates to the elaborated form of `expr`. This
ensures that information from the annotation is used during the analysis of `expr` itself.

### Static Coercion Expressions

A _static coercion expression_, also called a flexible type constraint, has the following form:

```fsgrammar
expr :> ty
```

The expression `upcast expr` is equivalent to `expr :> _`, so the target type is the same as the initial
type of the overall expression. For example:

```fsharp
([1;2;3] :> seq<int>).GetEnumerator()
(upcast [1;2;3] : seq<int>)
```

> **Clef Note**: Static coercion is valid for upcasting to implemented interfaces or base class types. The expression `(x :> obj)` is not valid because `obj` does not exist in Clef. See [Native Type Mappings](native-type-mappings.md#the-universal-base-type-obj-is-not-available).

The initial type of the overall expression is `ty`. Expression `expr` is checked using a fresh initial type
`tye`, with constraint `tye :> ty`. Static coercions are a primitive elaborated form.

### Dynamic Type-Test Expressions

A type-test expression has the following form:

```fsgrammar
expr :? ty
```

The expression has type `bool`. The compiler must determine the relationship
between the static type of `expr` and `ty` at compile time. If that relationship
cannot be determined, the expression is a compile-time error
([§](types-and-type-constraints.md#type-conversions)). No runtime type inspection
is performed.

To distinguish cases of a discriminated union, use pattern matching:

```fsharp
type Shape = Circle of float | Rectangle of float * float

let isCircle (s: Shape) =
    match s with
    | Circle _ -> true
    | Rectangle _ -> false
```

### Dynamic Coercion Expressions

A coercion expression using `:?>` has the following form:

```fsgrammar
expr :?> ty
```

The expression `downcast expr` is equivalent to `expr :?> _`, so the target type is the same as the initial
type of the overall expression.

The expression has type `ty`. The compiler must verify the conversion from the
static type of `expr` to `ty` at compile time. A conversion that requires runtime
type information is a compile-time error
([§](types-and-type-constraints.md#type-conversions)).

Pattern matching extracts values from discriminated unions:

```fsharp
type Value = IntVal of int | StrVal of string

let extractInt (v: Value) : int option =
    match v with
    | IntVal i -> Some i
    | StrVal _ -> None
```

## Quoted Expressions

Clef has quoted expressions of two forms:

```fsgrammar
<@ expr @>

<@@ expr @@>
```

The former is a _strongly typed quoted expression_, the latter a _weakly typed quoted expression_. A quotation is an **intrinsic form of the language**, not a library facility: its type constructor `Expr<ty>` is provided by the compiler, and there is no quotations namespace, no `Expr` module, no `ReflectedDefinition` attribute and no run-time representation of an expression tree. Where a Clef program opens the F# core library's quotations namespace, the reference is not a Clef construct (CCS8080).

A quotation is a phase-distinct structure. The enclosed expression is type-checked in place and elaborated into the program graph as the quotation's body, in the same node forms every other expression elaborates to; the compiler reads it there **as data, at compile time**, and never evaluates it. Nothing about a quotation is observable in the running program: a quotation has no run-time value, and executed code cannot hold, pass or evaluate one. A reference to a quotation from executed code is diagnosed with CCS8066 at the reference; a quotation in expression position inside executed code is diagnosed with CCS8066 at the quotation. A quotation the compiler does not consult is dead by construction and emits nothing. A binding whose value is a quotation is a **declaration**: it is neither initialised by its module nor emitted as a definition, and reachability never enters it from the module; a reference to it from inside another quotation (a descriptor citing a descriptor) is compile-time structure and is legal.

What the compiler reads out of a quotation:

- **Declarations**, read structurally by type name and field name: the platform descriptor ([Platform Bindings](platform-bindings.md)), the binding descriptors a generator emits (`Expr<TypeDescriptor>`, `Expr<FunctionDescriptor>`), a peripheral descriptor ([Native Type Mappings](native-type-mappings.md), "Quotations as Semantic Carriers"). A quoted record is read exactly as a plain record value is. [Platform predicates](platform-predicates.md) additionally carry typed Boolean syntax: CCS decides the supported closed expression fragment at a required boundary, with no runtime quotation evaluation.
- **Laws**, evaluated at compile time in a total, terminating, closed-form sub-language: the range law of [Numeric Selection §4](numeric-selection.md), whose bound is an image computation in the outward-rounded interval domain, not a run-time evaluation.

Both readers belong to the compiler. No reader is a library, and no reader reflects over a tree apart from the program graph.

### Strongly Typed Quoted Expressions

```fsgrammar
<@ expr @>
```

For example:

```fsharp
<@ 1 + 1 @>

<@ (fun x -> x + 1) @>
```

The type of the first is `Expr<int>`, of the second `Expr<int -> int>`. When checked, the initial type of `<@ expr @>` is asserted to be of the form `Expr<ty>` for a fresh type `ty`, and `expr` is checked with initial type `ty`.

### Weakly Typed Quoted Expressions

```fsgrammar
<@@ expr @@>
```

A weakly typed quotation omits the type argument: `<@@ 1 + 1 @@>` has type `Expr<ty>` for a fresh `ty` constrained only by the body. In a type annotation the bare name `Expr` denotes `Expr<_>`.

### Expression Splices

The splice forms `%expr` and `%%expr` are **not Clef constructs** and are diagnosed with CCS8065. There is no run-time quotation value to splice, and a declaration is written whole; composition of declarations is ordinary record construction, one quotation citing the binding that holds another.

## Evaluation of Elaborated Forms

At runtime, execution evaluates expressions to values. The evaluation semantics of each expression
form are specified in the subsections that follow.

### Default Demand and Sharing

Clef is **lazy by default**, with call-by-need sharing. An ordinary value binding
or supplied function argument denotes a deferred computation whose identity
exists before its result. Merely binding, passing or capturing that computation
SHALL NOT force it, except at an explicit [eager frontier](#eager-expressions).
Its first demand evaluates the required computation; later
demands through the same binding or argument share the result rather than replay
it. Two separately created computations are not implicitly one shared instance.

This rule includes effects inside an ordinary deferred argument or initializer.
For example, `(fun _ -> 0) (trace 1; 42)` does not execute `trace 1`, while a
demanded `(fun _ -> 0) (eager (trace 1; 42))` does. In
`let x = (trace 1; 42) in x + x`, the demanded arithmetic result requires `x`,
and the shared initializer executes `trace 1` once. A discarded ordinary binding
does not become an effect root merely because its initializer contains effects.
These rules do not erase explicit effect roots: when a sequential expression
`expr1; expr2` is itself demanded, its sequencing contract requires `expr1`
before `expr2`. Entry, startup, resource, subscription and foreign-call contracts
must specify their activation and demanded operands separately.

Demand is operation-specific. A branch demands its condition and selected arm;
an arithmetic operation demands the operands needed for its result; a case test
demands a union's tag without thereby demanding every payload. Application syntax,
a type annotation, compiler traversal or a physical layout choice is not proof
that every written operand must execute. The operation's declared argument
boundary remains distinct from a later application of a function-valued result.

An implementation may evaluate earlier or eliminate a suspension only when it
proves preservation of required results, termination, observable effects, sharing,
storage identity and lifetime. In particular, earlier evaluation cannot execute
an otherwise undemanded effect or divergent computation. Flat closures and thunks
provide representations for deferred values; their layout alone is not that
proof. Source demand and effect relationships SHALL survive Baker elaboration,
admission, Alex witnessing and target lowering.

Explicit `Lazy<'T>` exposes a memoized force operation; `seq<'T>` exposes fresh
enumeration state; `Cold<'T>` exposes a start/force boundary; `Incremental<'T>`
adds tracked invalidation and cache validation. They refine the default and are
not interchangeable cache or scheduling policies. See [Lazy Representation](lazy-representation.md),
[Sequence Representation](seq-representation.md) and [Incremental Computation](incremental-computation.md).

### Values and Execution Context

The execution of demanded elaborated Clef expressions results in values. Values include:

- Primitive constant values
- Values for value types, containing a value for each field in the value type
- Function values with associated closure environments
- References to record, union, and class instances
- Pointers to mutable locations (including static mutable locations, mutable fields and array elements)

Evaluation assumes the following evaluation context:

- A global environment that maps module-qualified names to values
- A local environment mapping names of variables to values
- The scoped cleanup obligations established by resource-management constructs

Native recoverable failures are explicit `Result` values under
[Error Handling](error-handling.md#application-runtime-error-handling).
Compatibility syntax and exceptions used inside compiler/editor tooling do not
imply a native exception-handler stack. Always-active invariant failure,
including an [exhausted match](patterns.md#exhausted-match-rules), terminates
with a diagnostic. Scoped cleanup has its own ordering obligations; it is not
an implicit conversion of a failure into a recoverable result.

### Parallel Execution and Memory Model

In a concurrent environment, evaluation may involve both multiple active computations (multiple
concurrent and parallel threads of execution) and multiple pending computations (pending
callbacks, such as those activated in response to an I/O event).

If multiple active computations concurrently access mutable locations, atomicity guarantees depend on the size and alignment of the values:

- Reads and writes of sizes less than or equal to one machine word are atomic when properly aligned.
- Larger values require explicit synchronization for atomic access.

The `VolatileField` attribute marks a mutable location as volatile, ensuring memory ordering guarantees for that location. Volatile fields are essential for memory-mapped I/O and inter-thread communication.

### Zero Values

Some types have a _zero value_. The zero value is the "default" value for the type. The following types have the following zero values:

- For struct types, the value with all fields set to the zero value for the type of the field. The zero value is also computed by the F# library function `Unchecked.defaultof<ty>`.

> **Clef Note**: In Clef, reference types do not have a null zero value since Clef is null-free by construction. The `Unchecked.defaultof<ty>` function returns a zero bit pattern for struct types only; it is a compile-time error to use it with reference types.

### Taking the Address of an Elaborated Expression

When the F# compiler determines the elaborated forms of certain expressions, it must compute a
“reference” to an elaborated expression `expr` , written `AddressOf(expr, mutation)`. The `AddressOf`
operation is used internally within this specification to indicate the elaborated forms of address-of
expressions, assignment expressions, and method and property calls on objects of variable and value
types.

The `AddressOf` operation is computed as follows:

- If `expr` has form `path` where `path` is a reference to a value with type `byref<ty>`, the elaborated
    form is `&path`.
- If `expr` has form `expra.field` where `field` is a mutable, non-readonly field, the elaborated
    form is `&(AddressOf(expra).field)`.
- If `expr` has form expra.[exprb] where the operation is an array lookup, the elaborated form is
    `&(AddressOf(expra).[exprb])`.
- If `expr` has any other form, the elaborated form is `&v` ,where `v` is a fresh mutable local value that
    is initialized by adding `let v = expr` to the overall elaborated form for the entire assignment
    expression. This initialization is known as a _defensive copy_ of an immutable value. If `expr` is a
    struct, `expr` is copied each time the `AddressOf` operation is applied, which results in a different
    address each time. To keep the struct in place, the field that contains it should be marked as
    mutable.

The `AddressOf` operation is computed with respect to `mutation`, which indicates whether the
relevant elaborated form uses the resulting pointer to change the contents of memory. This
assumption changes the errors and warnings reported.

- If `mutation` is `DefinitelyMutates`, then an error is given if a defensive copy must be created.
- If `mutation` is `PossiblyMutates`, then a warning is given if a defensive copy arises.

A compiler may upgrade `PossiblyMutates` to `DefinitelyMutates` for calls to property
setters and methods named `MoveNext` and `GetNextArg`, which are common cases of struct-mutators.

### Evaluating Value References

At runtime, an elaborated value reference `v` refers to its binding in the local
environment. A demand for its result forces an unevaluated binding once and
shares that result with subsequent references. Merely forwarding the binding's
deferred identity does not force its contents. Mutable references retain their
specified storage identity; call-by-need does not turn successive explicit
mutable reads into one immutable snapshot.

### Evaluating Function Applications

When the result of an elaborated application `f e1 ... en` is demanded:

- Demand `f` sufficiently to identify the callable. Preserve its actual code and
  captured environment rather than reconstructing its initializer.
- Associate each supplied expression with one shared deferred argument in its
  lexical environment. Supplying an ordinary argument does not force it,
  including when the expression has effects. At this activated application
  boundary, demand direct explicit eager actuals in source order under
  [Eager Expressions](#eager-expressions). A body that never demands an ordinary
  argument leaves it unevaluated; repeated demands share its result.
- If the callable declares `m <= n` arguments, extend its environment with the
  first `m` argument bindings and demand its body as required by the application.
  Any remaining arguments apply to the resulting callable at the subsequent
  application boundary; they are not forced in advance.
- If fewer than the declared arguments are supplied, produce a residual callable
  retaining the already supplied deferred identities. Later completion neither
  replays their expressions nor forces an operand merely because the operation
  has become fully supplied.

Each primitive or intrinsic establishes which arguments it demands and their
required effect order. Proven strictness can remove thunks while preserving this
behavior. Source argument position alone does not establish strictness.

### Evaluating Method Applications

The elaborated form is `e0.M(e1, ..., en)` for an instance method or
`M(e1, ..., en)` for a static method. Clef-defined methods follow the same
demand and sharing rules as function applications. The receiver is demanded to
the extent required for the selected member; it does not make every supplied
argument strict. Member resolution follows the admitted dispatch contract
([§](inference-supplementary.md#dispatch-slot-checking)). Clef does not inherit a
CLR null-reference exception path for this operation.

A foreign or platform method may require materialized arguments at its declared
call boundary. That demand and its effect order must be established by the
boundary contract; it is not a default evaluation rule for ordinary Clef methods.

### Evaluating Union Cases

At runtime, an elaborated use of a union case `Case(e1 , ..., en)` for a union type `ty` is evaluated as
follows:

- The constructor establishes the union case label and shared deferred payload
  bindings for `e1, ..., en`, demanding direct explicit eager payloads at the
  constructor frontier. Demanding the tag does not demand ordinary unused payloads.
- Payload projections demand their corresponding bindings and share the results.
  Representation specialization may remove suspensions only under the default
  demand and sharing proof obligations above.
- Native union cases retain their admitted tag/payload representation; a case
  without a payload does not introduce a null inhabitant. Target-specific
  encodings must preserve [Null-Freedom](error-handling.md#null-freedom) and the
  [discriminated-union contract](discriminated-union-representation.md).

### Evaluating Field Lookups

At runtime, an elaborated lookup of an F# field is evaluated as follows:

- The elaborated form is `expr.F` for an instance field or `F` for a static field.
- The (optional) `expr` is evaluated.
- The value of the field is read from either the global field table or the local field table associated
    with the object.

> **Clef Note**: In Clef, there is no null reference check since all references are valid by construction.

### Evaluating Array Expressions

At runtime, an elaborated array expression `[| e1; ...; en |]ty` is evaluated as follows:

- The array has the declared element order and retains one shared initialization
  computation per element. Its activated constructor demands direct explicit
  eager elements in source order; demanding shape does not alone demand ordinary
  element computations.
- Element access, mutation and external materialization must preserve the
  admitted array storage and demand contract. A strict realization is valid only
  where it preserves the default demand and sharing rules, including effects.

### Evaluating Record Expressions

At runtime, an elaborated record construction `{ field1 = e1; ... ; fieldn = en }ty` is evaluated as
follows:

- The result has type `ty` and a shared deferred binding for each field expression;
  the activated constructor demands direct explicit eager fields in source order.
- A demanded field forces its binding; unused fields remain deferred. Storage
  specialization preserves the default demand and sharing obligations.

### Evaluating Function Expressions

At runtime, an elaborated function expression `(fun v1 ... vn -> expr)` is evaluated as follows:

- The expression produces the callable for `expr` and its required captures; it
  does not execute the body.
- Immutable captures preserve their binding's value or shared deferred identity
  without forcing an unevaluated initializer. Mutable captures preserve the
  original storage cell. Already established values and references are retained,
  not reconstructed by replaying their initializers. Physical representation
  follows [Closure Representation](closure-representation.md).

### Evaluating Object Expressions

At runtime, elaborated object expressions

```fsgrammar
{ new ty0 args-expr? object-members
      interface ty1 object-members1
      interface tyn object-membersn }
```

is evaluated as follows:

- The expression evaluates to an object whose runtime type is compatible with all of the `tyi` and
    which has the corresponding dispatch map ([§](inference-supplementary.md#dispatch-slot-checking)). If present, the base construction expression
    `ty0 (args-expr)` is executed as the first step in the construction of the object.
- The object's admitted capture environment retains the source identities
  required by its members, under [Closure Representation](closure-representation.md).
  Immutable captures preserve established values or shared deferred computations;
  mutable captures preserve their original cells. Capturing a binding does not
  force its initializer or snapshot a mutable cell's current contents.

### Evaluating Definition Expressions

An ordinary value definition `pat = expr` establishes one shared deferred
initializer and the bindings introduced by `pat` in the local environment. A
simple named pattern does not force an ordinary initializer. A reached direct
`eager` initializer instead uses the explicit binding frontier above, even when
the bound value is unused. Demanding an ordinary binding forces
only the initializer and pattern structure needed to obtain that binding;
bindings extracted from the same initializer share its evaluation. Pattern
selection and failure must retain their own demand and failure contracts rather
than using a blanket eager-initialization rule. Explicit sequential effects,
mutable storage operations and startup activation have separate contracts.

### Evaluating Integer For Loops

At runtime, an integer for loop `for var = expr1 to expr2 do expr3 done` is evaluated as follows:

- Expressions `expr1` and `expr2` are evaluated once, in that order, to values `v1` and `v2`.
- The expression `expr3` is evaluated repeatedly with a fresh immutable binding for `var`
    at each successive value in the inclusive range from `v1` to `v2`.
- If `v1` is greater than `v2` , then `expr3` is never evaluated.

### Evaluating While Loops

As runtime, while-loops `while expr1 do expr2 done` are evaluated as follows:

- Expression `expr1` is evaluated to a value `v1`.
- If `v1` is true, expression `expr2` is evaluated, and the expression `while expr1 do expr2 done` is
    evaluated again.
- If `v1` is `false`, the loop terminates and the result is the unit value `()`.

### Evaluating Static Coercion Expressions

At runtime, a static coercion expression `expr :> ty` evaluates `expr` to a value
`v` and returns that value through the statically verified target type `ty`.
The coercion does not box the value onto a heap.

### Evaluating Dynamic Type-Test Expressions

A type-test expression `expr :? ty` evaluates `expr` and returns the Boolean
result established at compile time
([§](#dynamic-type-test-expressions)). It does not inspect a runtime type.

### Evaluating Dynamic Coercion Expressions

A coercion expression `expr :?> ty` evaluates `expr` and applies the conversion
verified at compile time ([§](#dynamic-coercion-expressions)). It performs no
runtime type test, boxing, or unboxing.

### Evaluating Sequential Execution Expressions

At runtime, elaborated sequential expressions `expr1 ; expr2` are evaluated as follows:

- The expression `expr1` is evaluated for its side effects and the result is discarded.
- The expression `expr2` is evaluated to a value `v2` and the result of the overall expression is `v2`.

### Evaluating Try-with Expressions

The syntax `try expr1 with rules` is retained for tooling compatibility as
specified in [Try/With Syntax Compatibility](error-handling.md#trywith-syntax-compatibility).
Native recoverable handling uses explicit `Result` values and pattern matching;
exception-style source receives `CCS8300`. This compatibility form does not
authorize a hidden native throw/rethrow mechanism or interception of an
always-active requirement failure. A native elaboration must express its
actual success/error value and ordered handler selection in the settled graph.

### Evaluating Try-finally Expressions

The cleanup ordering of `try expr1 finally expr2` is independent of recoverable
failure representation. Once this computation is activated, `expr1` is
evaluated first. Before its value `v` leaves the protected scope, `expr2` is
evaluated exactly once for cleanup; successful cleanup preserves `v`. An
explicit `Error` is a value and does not bypass this cleanup. Other admitted
scope-exit paths must retain the cleanup obligations established for that
scope. If cleanup itself terminates execution, no normal result is produced.

These rules require explicit scoped cleanup in native elaboration. They do not
specify a managed exception stack, native throw/rethrow, or recovery from an
always-active requirement failure. Target termination behavior must not be
advertised as an exception-unwinding guarantee.

### Evaluating AddressOf Expressions

At runtime, an elaborated address-of expression is evaluated as follows. First, the expression has
one of the following forms:

- `&path` where `path` is a static field.
- `&(expr.field)`
- `&(expra.[exprb])`
- `&v` where `v` is a local mutable value.

The expression evaluates to the address of the referenced local mutable value, mutable field, or
mutable static field.

> **Clef Note**: In Clef, arrays are invariant and covariant array assignment is not supported. The type system statically prevents array type mismatches that would require runtime checks.

### Values with Underspecified Object Identity

Physical identity is not guaranteed for values of the following types:

- Function types
- Tuple types
- Immutable record types
- Union types

For two values of such types, the compiler may produce semantically equivalent
but physically distinct values. Equality and hashing are resolved through static
type constraints. Value comparison uses structural equality through the `=`
operator or `IEquatable<'T>` constraint. The `System.Object` operations
`ReferenceEquals`, `GetType()`, and `GetHashCode()` are not available.
