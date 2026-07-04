---
title: "Types and Type Constraints"
weight: 100
category: Language
status: normative
---

The notion of _type_ is central to the static checking of Clef programs. The word is used with three distinct but related meanings:

- **Type definitions**, such as the definitions of `string`, `option<_>`, or `Map<_,_>`. In Clef, all types have explicit memory representations defined at compile time.
- **Syntactic types**, such as the text `option<_>` that might occur in a program text. Syntactic types are converted to static types during the process of type checking and inference.
- **Static types**, which result from type checking and inference, either by the translation of syntactic types that appear in the source text, or by the application of constraints that are related to particular language constructs. For example, `option<int>` is the fully processed static type that is inferred for an expression `Some(1+1)`. Static types may contain `type variables` as described later in this section.

> **Clef Note**: Clef resolves all type information at compile time through CCS (Clef Compiler Service). Types are compile-time constructs that guide memory layout, code generation, and type-safe operations. Pattern matching type tests (`:?`, `:?>`) are resolved statically where possible, or generate compile-time errors when the type relationship cannot be determined.

The following describes the syntactic forms of types as they appear in programs:

```fsgrammar
type :=
    ( type )
    type - > type                  -- function type
    type * ... * type              -- tuple type
    struct (type * ... * type)     -- struct tuple type
    typar                          -- variable type
    long-ident                     -- named type, such as int
    long-ident <type-args>         -- named type, such as list<int>
    long-ident < >                 -- named type, such as IEnumerable< >
    type long-ident                -- named type, such as int list
    type [ , ... , ]               -- array type
    type typar-defns               -- type with constraints
    typar :> type                  -- variable type with subtype constraint
    # type                         -- anonymous type with subtype constraint

type-args := type-arg , ..., type-arg

type-arg :=
    type                           -- type argument
    measure                        -- unit of measure argument
    static-parameter               -- static parameter

atomic-type :=
    type : one of
            #type typar ( type ) long-ident long-ident <type-args>

typar :=
    _                              -- anonymous variable type
    ' ident                        -- type variable
    ^ ident                        -- static head-type type variable

constraint :=
    typar :> type                  -- coercion constraint
    typar : null                   -- nullness constraint (see note below)
    static-typars : ( member-sig ) -- member "trait" constraint
    typar : (new : unit -> 'T)     -- default constructor constraint
    typar : struct                 -- value type constraint
    typar : not struct             -- reference type constraint
    typar : enum< type >           -- enum decomposition constraint
    typar : unmanaged              -- unmanaged constraint
    typar : equality
    typar : comparison

typar-defn := attributes opt typar

typar-defns := < typar-defn, ..., typar-defn typar-constraints opt >

typar-constraints := when constraint and ... and constraint

static-typars :=
    ^ ident
    (^ ident or ... or ^ ident )

member-sig := <see Section 10>
```

In a type instantiation, the type name and the opening angle bracket must be syntactically adjacent
with no intervening whitespace, as determined by lexical filtering ([§](lexical-filtering.md#lexical-filtering)). Specifically:

```fsharp
array<int>
```

and not

```fsharp
array < int >
```

## Checking Syntactic Types

Syntactic types are checked and converted to _static types_ as they are encountered. Static types are a
specification device used to describe

- The process of type checking and inference.
- The connection between syntactic types and the execution of F# programs.

Every expression in an F# program is given a unique inferred static type, possibly involving one or
more explicit or implicit generic parameters.

For the remainder of this specification we use the same syntax to represent syntactic types and
static types. For example `int32 * int32` is used to represent the syntactic type that appears in
source code and the static type that is used during checking and type inference.

The conversion from syntactic types to static types happens in the context of a _name resolution
environment_ (see [§](inference-name-resolution.md#name-resolution)), a _floating type variable environment_, which is a mapping from names to type
variables, and a _type inference environment_ (see [§](inference-constraint-solving.md#constraint-solving)).

The phrase “fresh type” means a static type that is formed from a _fresh type inference variable_. Type
inference variables are either solved or generalized by _type inference_ ([§](inference-constraint-solving.md#constraint-solving)). During conversion and
throughout the checking of types, expressions, declarations, and entire files, a set of _current
inference constraints_ is maintained. That is, each static type is processed under input constraints `Χ` ,
and results in output constraints `Χ’`. Type inference variables and constraints are progressively
_simplified_ and _eliminated_ based on these equations through _constraint solving_ ([§](inference-constraint-solving.md#constraint-solving)).

### Unified Type Representation

CCS uses a unified type representation throughout type checking and inference. Type constructors from the [Native Type Universe (NTU)](native-type-universe.md) are recognized during construction, producing `NativeType` values directly with type variables preserved.

When CCS encounters a type expression such as `array<'T>`:

1. The type constructor `array` is recognized as part of the NTU
2. A `NativeType.TArray` value is produced
3. The type variable `'T` is preserved as `NativeType.TVar`
4. Hindley-Milner type inference proceeds with full polymorphism

Type variables participate fully in:
- Unification during type checking
- Constraint propagation
- Let-polymorphism (generalization at let-binding sites)
- SRTP constraint collection and resolution

Monomorphization, the instantiation of polymorphic types with concrete types, [occurs during PSG saturation](program-semantic-graph.md) when code generation requires concrete representations. This timing preserves optimization opportunities and maintains principal types throughout type inference.

### Named Types

_Named types_ have several forms, as listed in the following table.

| **Form**                   | **Description**                                                                                                                                                                                                      |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `long-ident <ty1, ..., tyn>` | Named type with one or more suffixed type arguments.                                                                                                                                                                 |
| `long-ident`               | Named type with no type arguments                                                                                                                                                                                    |
| `type long-ident`          | Named type with one type argument; processed the same as `long-ident<type>`                                                                                                                                          |
| `ty1 -> ty2`               | A function type, where: <br> ▪ ty1 is the domain of the function values associated with the type<br> ▪ ty2 is the range.<br>In Clef, function types compile to native closures (see [Closure Representation](closure-representation.md)). |

Named types are converted to static types as follows:

- _Name Resolution for Types_ (see [§](inference-name-resolution.md#name-resolution)) resolves `long-ident` to a type definition with formal generic
  parameters `<typar1, ..., typarn>` and formal constraints `C`. The number of type arguments `n` is
  used during the name resolution process to distinguish between similarly named types that take
  different numbers of type arguments.
- Fresh type inference variables `<ty'1, ..., ty'n>` are generated for each formal type parameter. The
  formal constraints `C` are added to the current inference constraints for the new type inference
  variables; and constraints `tyi = ty'i` are added to the current inference constraints.

### Variable Types

A type of the form `'ident` is a _variable_ type. For example, the following are all variable types:

```fsharp
'a
'T
'Key
```

During checking, _Name Resolution_ (see [§](inference-name-resolution.md#name-resolution)) is applied to the identifier.

- If name resolution succeeds, the result is a variable type that refers to an existing declared type
  parameter.
- If name resolution fails, the current _floating type variable environment_ is consulted, although
  only in the context of a syntactic type that is embedded in an expression or pattern. If the type
  variable name is assigned a type in that environment, F# uses that mapping. Otherwise, a fresh

type inference variable is created (see [§](inference-constraint-solving.md#constraint-solving)) and added to both the type inference environment
and the floating type variable environment.

A type of the form `_` is an _anonymous variable_ type. A fresh type inference variable is created and
added to the type inference environment (see [§](inference-constraint-solving.md#constraint-solving)) for such a type.

A type of the form `^ident` is a _statically resolved type variable_. A fresh type inference variable is
created and added to the type inference environment (see [§](inference-constraint-solving.md#constraint-solving)). This type variable is tagged with
an attribute that indicates that it can be generalized only at `inline` definitions (see [§](inference-constraint-solving.md#generalization)). The
same restriction on generalization applies to any type variables that are contained in any type that is
equated with the `^ident` type in a type inference equation.

> Note: This specification generally uses uppercase identifiers such as `'T` or `'Key` for user-declared generic type parameters,
 and uses lowercase identifiers such as `'a` or `'b` for compiler-inferred generic parameters.

### Tuple Types

A _tuple type_ has the following form:

```fsgrammar
ty 1 * ... * tyn
```

Tuple types in Clef represent anonymous product types with a direct, unboxed memory layout. Fields are laid out contiguously with natural alignment (see [§](expressions.md#tuple-expressions)).

> **Clef Note**: Tuples are value types with direct, unboxed memory layout. A tuple `int * string` is laid out as:
>
> ```
> ┌─────────────┬─────────────────────────────┐
> │ int (word)  │ string (ptr + len = 2 words)│
> └─────────────┴─────────────────────────────┘
> ```
>
> Tuples are stack-allocated by default, or arena-allocated when escaping their scope.

#### Struct Tuple Types

A _struct tuple type_ has the following form:

```fsgrammar
struct ( ty 1 * ... * tyn )
```

> **Clef Note**: In Clef, ALL tuples have value semantics with direct memory layout - there is no distinction between "reference tuples" and "struct tuples" at the representation level. The `struct` keyword is accepted for compatibility with managed F# code, but both forms compile to the same unboxed representation. There is no `System.Tuple` or `System.ValueTuple` - tuples are native anonymous product types.

The "structness" annotation of tuple expressions and tuple patterns is accepted for source compatibility but has no effect on memory representation in Clef - all tuples are laid out directly without heap allocation.

### Array Types

Array types have the following forms:

```fsgrammar
ty []
ty [ , ... , ]
```

A type of the form `ty []` is a _single-dimensional array_ type, and a type of the form `ty[ , ... , ]` is a _multidimensional array type_. For example, `int[,,]` is an array of integers of rank 3.

> **Clef Note**: Arrays in Clef use a fat pointer representation, NOT `System.Array`:
>
> ```
> array<'T>
> ┌─────────────────┬─────────────────┐
> │ ptr: *T         │ len: usize      │
> └─────────────────┴─────────────────┘
>      1 word            1 word        = 2 words (header)
>                                      + len * sizeof<'T> (elements)
> ```
>
> Elements are laid out contiguously with natural alignment. Bounds checking is always performed (no unsafe indexing by default). Empty arrays have `len = 0` with a valid pointer - arrays are never null.

> Note: The type `int[][,]` in F# is the same as the type `int[,][]` in C# although the dimensions are swapped. This ensures consistency with other postfix type names in F# such as `int list list`.

Clef supports multidimensional array types up to rank 4.

### Constrained Types

A _type with constraints_ has the following form:

```fsgrammar
type when constraints
```

During checking, `type` is first checked and converted to a static type, then `constraints` are checked
and added to the current inference constraints. The various forms of constraints are described
in (see [§](types-and-type-constraints.md#type-constraints))

A type of the form `typar :> type` is a _type variable with a subtype constraint_ and is equivalent to
`typar when typar :> type`.

A type of the form `#type` is an _anonymous type with a subtype constraint_ and is equivalent to `'a
when 'a :> type` , where `'a` is a fresh type inference variable.

## Type Constraints

A _type constraint_ limits the types that can be used to create an instance of a type parameter or type
variable. F# supports the following type constraints:

- Subtype constraints
- Nullness constraints
- Member constraints
- Default constructor constraints
- Value type constraints
- Reference type constraints
- Enumeration constraints
- Delegate constraints
- Unmanaged constraints
- Equality and comparison constraints

### Subtype Constraints

An _explicit subtype constraint_ has the following form:

```fsgrammar
typar :> type
```

During checking, `typar` is first checked as a variable type, `type` is checked as a type, and the
constraint is added to the current inference constraints. Subtype constraints affect type coercion as
specified in (see [§](types-and-type-constraints.md#subtyping-and-coercion))

Note that subtype constraints also result implicitly from:

- Expressions of the form `expr :> type`.
- Patterns of the form `pattern :> type`.
- The use of generic values, types, and members with constraints.
- The implicit use of subsumption when using values and members (see [§](inference-application-resolution.md#implicit-insertion-of-flexibility-for-uses-of-functions-and-members)).

A type variable cannot be constrained by two distinct instantiations of the same named type. If two
such constraints arise during constraint solving, the type instantiations are constrained to be equal.
For example, during type inference, if a type variable is constrained by both `IA<int>` and `IA<string>`,
an error occurs when the type instantiations are constrained to be equal. This limitation is
specifically necessary to simplify type inference, reduce the size of types shown to users, and help
ensure the reporting of useful error messages.

### Nullness Constraints

> **Clef Note**: Clef enforces **absolute null-freedom**. The nullness constraint `typar : null` is NOT SUPPORTED in Clef. All types are non-nullable by construction - there is no `null` literal and no null values at runtime.
>
> This is the most significant semantic difference from managed F#. See [§](#nullness) for the complete null-freedom model.

```fsgrammar
typar : null    -- NOT SUPPORTED in Clef
```

The nullness constraint syntax is accepted for source compatibility but produces a compile-time error (CCS8010) indicating that null is not permitted in Clef code.

Code that requires optional values MUST use `option<'T>` (which compiles to stack-allocated `voption<'T>` semantics):

```fsharp
// Managed F# pattern (NOT supported):
let maybeNull : string = null  // ERROR: null literal not permitted

// Clef pattern (correct):
let maybeValue : string option = None  // OK: explicit optionality
 
```

### Member Constraints

An _explicit member constraint_ has the following form:

```fsgrammar
( typar or ... or typar ) : ( member-sig )
```

For example, the F# library defines the + operator with the following signature:

```fsharp
val inline (+) : ^a -> ^b -> ^c
    when (^a or ^b) : (static member (+) : ^a * ^b -> ^c)
```

This definition indicates that each use of the `+` operator results in a constraint on the types that
correspond to parameters `^a`, `^b`, and `^c`. If these are named types, then either the named type for
`^a` or the named type for `^b` must support a static member called `+` that has the given signature.

In addition:

- Each `typar` must be a statically resolved type variable (see [§](types-and-type-constraints.md#variable-types)) in the form `^ident`. This ensures
  that the constraint is resolved at compile time against a corresponding named type. It also
  means that generic code cannot use this constraint unless that code is marked inline (see [§](inference-constraint-solving.md#generalization)).
- The `member-sig` cannot be generic; that is, it cannot include explicit type parameter definitions.
- The conditions that govern when a type satisfies a member constraint are specified in (see [§](inference-constraint-solving.md#solving-member-constraints)).

> Note: Member constraints are primarily used to define overloaded functions in the F# library and are used relatively rarely in F# code.<br>
Uses of overloaded operators do not result in generalized code unless definitions are marked as inline. For example, the function<br><br> `let f x = x + x`<br><br>
results in a function `f` that can be used only to add one type of value, such as `int` or `float`. The exact type is determined by later constraints.

A type variable may not be involved in the support set of more than one member constraint that has
the same name, staticness, argument arity, and support set (see [§](inference-constraint-solving.md#solving-member-constraints)). If it is, the argument and
return types in the two member constraints are themselves constrained to be equal. This limitation
is specifically necessary to simplify type inference, reduce the size of types shown to users, and
ensure the reporting of useful error messages.

### Default Constructor Constraints

An _explicit default constructor constraint_ has the following form:

```fsgrammar
typar : (new : unit -> 'T)
```

During constraint solving (see [§](inference-constraint-solving.md#constraint-solving)), the constraint `type : (new : unit -> 'T)` is met if `type` has a parameterless constructor.

> **Clef Note**: This constraint is supported for record and class types that have a default constructor. The compiler verifies that the type can be constructed with no arguments by examining the type definition.

### Value Type Constraints

An _explicit value type constraint_ has the following form:

```fsgrammar
typar : struct
```

During constraint solving (see [§](inference-constraint-solving.md#constraint-solving)), the constraint `type : struct` is met if `type` is a value type - that is, a type with direct (non-pointer) representation including:

- Primitive types (`int`, `float`, `bool`, etc.)
- Struct types (types marked with `[<Struct>]`)
- Enum types
- Single-case discriminated unions (which are optimized to their payload representation)

> **Clef Note**: There is no `System.Nullable<_>` in Clef. Optional values are represented by `option<'T>`, which in Clef compiles to stack-allocated `voption<'T>` semantics - a tagged value type, not a nullable reference. The `option` type can wrap any type, including other options, without the restrictions that apply to `System.Nullable` in managed code.

### Reference Type Constraints

An _explicit reference type constraint_ has the following form:

```fsgrammar
typar : not struct
```

During constraint solving (see [§](inference-constraint-solving.md#constraint-solving)), the constraint `type : not struct` is met if `type` is a reference type - that is, a type whose values are represented by pointers:

- Class types
- Interface types
- Records (unless marked `[<Struct>]`)
- [Discriminated unions](discriminated-union-representation.md) (unless marked `[<Struct>]` or single-case)
- Function types
- List types

> **Clef Note**: The distinction between "value types" and "reference types" in Clef refers to representation strategy, not heap allocation. Reference types use pointer indirection but may still be stack or arena allocated - there is no GC heap. The constraint `not struct` ensures the type uses pointer representation, which is relevant for certain generic patterns.

### Enumeration Constraints

An _explicit enumeration constraint_ has the following form:

```fsgrammar
typar : enum<underlying-type>
```

During constraint solving (see [§](inference-constraint-solving.md#constraint-solving)), the constraint `type : enum<underlying-type>` is met if `type` is an F# enumeration type that has constant literal values of type `underlying-type`.

> Note: This constraint form exists primarily to allow the definition of library functions such as `enum`. It is rarely used directly in F# programming. The `enum` constraint verifies that the type is an enumeration with the specified underlying integral type.

### Delegate Constraints

> **Clef Note**: The delegate constraint is NOT SUPPORTED in Clef. CLI delegates are a managed runtime concept that does not exist in native compilation.

```fsgrammar
typar : delegate< tupled-arg-type , return-type>    -- NOT SUPPORTED
```

Clef uses function types directly for callbacks and event handling. Where managed F# would use delegates, Clef uses first-class functions:

```fsharp
// Managed F# pattern (NOT supported):
let handler : EventHandler = new EventHandler(fun sender args -> ...)

// Clef pattern (correct):
let handler : obj -> EventArgs -> unit = fun sender args -> ...
```

The delegate constraint syntax is accepted for source compatibility but produces a compile-time error.

### Unmanaged Constraints

An _unmanaged constraint_ has the following form:

```fsgrammar
typar : unmanaged
```

During constraint solving (see [§](inference-constraint-solving.md#constraint-solving)), the constraint `type : unmanaged` is met if `type` is unmanaged as
specified below:

- Types sbyte, `byte`, `char`, `nativeint`, `unativeint`, `float32`, `float`, `int16`, `uint16`, `int32`, `uint32`,
  `int64`, `uint64`, `decimal` are unmanaged.
- Type `nativeptr<type>` is unmanaged.
- A non-generic struct type whose fields are all unmanaged types is unmanaged.

> **Clef Note**: `nativeptr<_>` is not user-denotable in Clef source. It survives only as internal `TNativePtr` compiler plumbing, so the clause above describes an internal representation rather than a type a program can write. Buffers use bounded stack arrays, registers use the `Mmio` width-typed handle, and a C binding's returned pointer uses an opaque handle (`Ptr<'T, Region, Access>`).

### Equality and Comparison Constraints

_Equality constraints_ and _comparison constraints_ have the following forms, respectively:

```fsgrammar
typar : equality
typar : comparison
```

During constraint solving (see [§](inference-constraint-solving.md#constraint-solving)), the constraint `type : equality` is met if both of the following conditions are true:

- The type is a named type, and the type definition does not have, and is not inferred to have, the `NoEquality` attribute.
- The type has `equality` dependencies `ty1,..., tyn`, each of which satisfies `tyi: equality`.

The constraint `type : comparison` is a `comparison constraint`. Such a constraint is met if all the following conditions hold:

- If the type is a named type, then the type definition does not have, and is not inferred to have, the `NoComparison` attribute, and the type supports ordering operations.
- If the type has `comparison dependencies` `ty1, ..., tyn`, then each of these must satisfy `tyi : comparison`.

> **Clef Note**: In Clef, equality and comparison are resolved through SRTP (Statically Resolved Type Parameters) against the native witness hierarchy. The compiler verifies that appropriate `(=)` and `compare` operations exist for the types at compile time. Comparison capability is a compile-time property verified by CCS.

An equality constraint is satisfied by:
- All primitive types (`int`, `float`, `bool`, `string`, etc.)
- Structural types (records, unions, tuples) where all component types satisfy equality
- Types not marked with `[<NoEquality>]`

A comparison constraint is satisfied by:
- Ordered primitive types (`int`, `float`, `string`, etc.)
- Structural types where all component types satisfy comparison
- Types not marked with `[<NoComparison>]`

## Type Parameter Definitions

Type parameter definitions can occur in the following locations:

- Value definitions in modules
- Member definitions
- Type definitions
- Corresponding specifications in signatures

For example, the following defines the type parameter ‘T in a function definition:

```fsharp
let id<'T> (x:'T) = x
```

Likewise, in a type definition:

```fsharp
type Funcs<'T1,'T2> =
    { Forward: 'T1 -> 'T2
      Backward : 'T2 -> 'T2 }
```

Likewise, in a signature file:

```fsharp
val id<'T> : 'T -> 'T
```

Explicit type parameter definitions can include `explicit constraint declarations`. For example:

```fsharp
let closeResources<'T when 'T :> ICloseable> (x: 'T, y: 'T) =
    x.Close()
    y.Close()
```

The constraint in this example requires that `'T` be a type that supports the `ICloseable` interface (or trait, resolved via SRTP in Clef).

However, in most circumstances, declarations that imply subtype constraints on arguments can be written more concisely:

```fsharp
let throw (x: exn) = raise x
```

Multiple explicit constraint declarations use `and`:

```fsharp
let processOrdered<'T when 'T : comparison and 'T :> ICloseable> (x: 'T, y: 'T) =
    if compare x y < 0 then x.Close() else y.Close()
```

> **Clef Note**: Interface constraints like `:> IDisposable` from managed F# are typically resolved through SRTP member constraints in Clef. The native library provides trait-like patterns for common capabilities.

Explicit type parameter definitions can declare custom attributes on type parameter definitions (see [§](special-attributes-and-types.md)).

## Logical Properties of Types

During type checking and elaboration, syntactic types and constraints are processed into a reduced
form composed of:

- Named types `op<types>`, where each `op` consists of a specific type definition, an operator to form
  function types, an operator to form array types of a specific rank, or an operator to form specific
  `n-tuple` types.
- Type variables `'ident`.

### Characteristics of Type Definitions

Type definitions include native types (such as `string`, `int`, `array`) and types that are defined in F# code (see [§](type-definitions.md#type-definitions)). The following terms are used to describe type definitions:

- Type definitions may be _generic_, with one or more type parameters; for example, `Map<'Key,'Value>`.
- The generic parameters of type definitions may have associated `formal type constraints`.
- Type definitions may have _custom attributes_ (see [§](special-attributes-and-types.md)), some of which are relevant to checking and inference.
- Type definitions may be _type abbreviations_ (see [§](type-definitions.md#type-abbreviations)). These are eliminated for the purposes of checking and inference (see [§](types-and-type-constraints.md#expanding-abbreviations-and-inference-equations)).

- Type definitions have a `kind` which is one of the following:
  - `Class`
  - `Interface`
  - `Struct`
  - `Record`
  - `Union`
  - `Enum`
  - `Measure`
  - `Abstract`

  > **Clef Note**: The `Delegate` kind from managed F# is not supported - Clef uses function types directly.

  The kind is determined at the point of declaration by Type Kind Inference (see [§](type-definitions.md#type-kind-inference)) if it is not specified explicitly as part of the type definition. The _kind_ of a type refers to the kind of its outermost named type definition, after expanding abbreviations. For example, a type is a _class_ type if it is a named type `C<types>` where `C` is of kind _class_.

- Type definitions may be _sealed_. Record, union, function, tuple, struct, enum, and array types are all sealed, as are class types that are marked with the `SealedAttribute` attribute.
- Type definitions may have zero or one _base type declarations_. Each base type declaration
  represents an additional type that is supported by any values that are formed using the type
  definition. Furthermore, some aspects of the base type are used to form the implementation of
  the type definition.
- Type definitions may have one or more _interface declarations_. These represent additional
  encapsulated types that are supported by values that are formed using the type.

Class, interface, function, tuple, record, and union types are all _reference_ type definitions (meaning they use pointer indirection in their representation). A type is a reference type if its outermost named type definition is a reference type, after expanding type definitions.

Struct types are _value types_ (meaning they have direct, non-pointer representation).

> **Clef Note**: The distinction between "reference types" and "value types" in Clef is about memory representation (pointer vs. direct), not about heap vs. stack allocation. Both can be stack-allocated or arena-allocated depending on escape analysis.

### Expanding Abbreviations and Inference Equations

Two static types are considered equivalent and indistinguishable if they are equivalent after taking into account both of the following:

- The inference equations that are inferred from the current inference constraints (see [§](inference-constraint-solving.md#constraint-solving)).
- The expansion of type abbreviations (see [§](type-definitions.md#type-abbreviations)).

> **Clef Note**: In Clef, type abbreviations like `int`, `string`, and `option` are resolved directly to their native representations by CCS (Clef Compiler Service). There is no mapping to BCL types like `System.Int32` or `System.String`.

For example, `int` is a native type abbreviation for the platform word-sized integer:

```fsharp
// int in Clef = platform word (64-bit on x86-64, 32-bit on thumbv8m/M33)
// NOT an abbreviation for System.Int32
 
```

The types `int` and `int32` are distinct in Clef:
- `int` = platform word (64-bit on x86-64, 32-bit on thumbv8m/M33)
- `int32` = fixed 32-bit integer

Likewise, consider the process of checking this function:

```fsharp
let checkString (x: string) y =
    (x = y), String.contains "Hello" y
```

During checking, fresh type inference variables are created for values `x` and `y`; let's call them `ty1` and `ty2`. Checking imposes the constraints `ty1 = string` and `ty1 = ty2`. The second constraint results from the use of the generic `=` operator. As a result of constraint solving, `ty2 = string` is inferred, and thus the type of `y` is `string`.

All relations on static types are considered after the elimination of all equational inference constraints and type abbreviations. For example, we say `int32` is a struct type because it has direct (non-pointer) representation.

> Note: Implementations should attempt to preserve type abbreviations when reporting types and errors to users. This typically means that type abbreviations should be preserved in the logical structure of types throughout the checking process.

### Type Variables and Definition Sites

Static types may be type variables. During type inference, static types may be _partial_, in that they
contain type inference variables that have not been solved or generalized. Type variables may also
refer to explicit type parameter definitions, in which case the type variable is said to be _rigid_ and
have a _definition site_.

For example, in the following, the definition site of the type parameter 'T is the type definition of C:

```fsharp
type C<'T> = 'T * 'T
```

Type variables that do not have a binding site are _inference variables_. If an expression is composed
of multiple sub-expressions, the resulting constraint set is normally the union of the constraints that
result from checking all the sub-expressions. However, for some constructs (notably function, value
and member definitions), the checking process applies _generalization_ (see [§](inference-constraint-solving.md#generalization)). Consequently, some
intermediate inference variables and constraints are factored out of the intermediate constraint sets
and new implicit definition site(s) are assigned for these variables.

For example, given the following declaration, the type inference variable that is associated with the
value `x` is generalized and has an implicit definition site at the definition of function `id`:

```fsharp
let id x = x
```

Occasionally in this specification we use a more fully annotated representation of inferred and
generalized type information. For example:

```fsharp
let id<'a> x'a = x'a
```

Here, `'a` represents a generic type parameter that is inferred by applying type inference and
generalization to the original source code (see [§](inference-constraint-solving.md#generalization)), and the annotation represents the definition site
of the type variable.

### Base Type of a Type

> **Clef Note**: The concept of "base type" in Clef differs from managed F#. There is no universal `System.Object` base type. Instead, types are organized by their _structural category_ which determines memory layout and available operations.

| **Static Type** | **Structural Category** | **Notes** |
|-----------------|------------------------|-----------|
| Abstract types  | Abstract | Must be inherited; no direct instantiation |
| All array types | Fat pointer | `{ptr, len}` representation |
| Class types     | Reference | Pointer to allocated data; may have declared base type |
| Enum types      | Integral | Same representation as underlying type |
| Exception types | Record-like | Structured error information |
| Interface types | Abstract | Compile-time contract only |
| Record types    | Product | Contiguous field layout |
| Struct types    | Value | Direct (non-pointer) representation |
| Union types     | Sum | Tagged payload layout |
| Variable types  | Polymorphic | Resolved at instantiation |

The inheritance hierarchy of class types works as in managed F#, with the declared base type determining inherited members. However, there is no universal base type that all types inherit from.

### Interface Types of a Type

The _interface types_ of a named type `C<type-inst>` are defined by the transitive closure of the interface declarations of `C` and the interface types of the base type of `C`, where formal generic parameters are substituted for the actual type instantiation `type-inst`.

> **Clef Note**: Interface implementation in Clef is verified at compile time through SRTP resolution against the native witness hierarchy. Arrays support iteration through the `seq<'T>` pattern.

### Type Equivalence

Two static types `ty1` and `ty2` are _definitely equivalent_ (with respect to a set of current inference
constraints) if either of the following is true:

- `ty1` has form `op<ty11, ..., ty1n>`, `ty2` has form `op<ty21, ..., ty2n>` and each `ty1i` is
  definitely equivalent to `ty2i` for all `1` <= `i` <= `n`.

**OR**

- `ty1` and `ty2` are both variable types, and they both refer to the same definition site or are the
  same type inference variable.

This means that the addition of new constraints may make types definitely equivalent where
previously they were not. For example, given `Χ = { 'a = int }`, we have `list<int>` = `list<'a>`.

Two static types `ty1` and `ty2` are _feasibly equivalent_ if `ty1` and `ty2` may become definitely equivalent if
further constraints are added to the current inference constraints. Thus `list<int>` and `list<'a>` are
feasibly equivalent for the empty constraint set.

### Subtyping and Coercion

A static type `ty2` _coerces to_ static type `ty1` (with respect to a set of current inference constraints X), if
`ty1` is in the transitive closure of the base types and interface types of `ty2`. Static coercion is written
with the `:>` symbol:

```fsharp
ty2 :> ty1
```

Variable types `'T` coerce to all types `ty` if the current inference constraints include a constraint of the
form `'T :> ty2`, and `ty` is in the inclusive transitive closure of the base and interface types of `ty2`.

A static type `ty2` _feasibly coerces_ to static type `ty1` if `ty2` _coerces_ to `ty1` may hold through the addition
of further constraints to the current inference constraints. The result of adding constraints is defined
in `Constraint Solving` (see [§](inference-constraint-solving.md#constraint-solving)).

### Nullness

> **Clef Note**: Clef enforces **absolute null-freedom**. This is the most significant semantic difference from managed F#. There is no `null` literal, no null values at runtime, and no nullness constraints.

**All types in Clef are non-nullable by construction.** There is exactly one category:

- **Types without `null`.** ALL types in Clef fall into this category. There is no `null` literal, no `AllowNullLiteral` attribute, and no way to construct or observe null values.

This null-freedom is enforced at multiple levels:

1. **Syntax**: The `null` keyword is not permitted in Clef source code (error CCS8010).
2. **Type system**: No type satisfies the nullness constraint; the constraint itself is not supported.
3. **Runtime**: All values have valid, non-null representations.

**Representing Optional Values**:

Where managed F# might use null to indicate absence, Clef uses `option<'T>`:

```fsharp
// Managed F# pattern (NOT supported in Clef):
let maybeString : string = null

// Clef pattern (correct):
let maybeString : string option = None
```

The `option<'T>` type in Clef compiles to stack-allocated `voption<'T>` semantics - a tagged value type with `None = 0` as a tag value, not a null pointer.

**API Implications (Null-Freedom Cascades)**:

Standard library APIs that use sentinel values in managed F# return `option` in Clef:

| Managed F# Pattern | Clef Pattern |
|-------------------|-------------------|
| `string.IndexOf(c)` returns `-1` | `String.indexOf c s` returns `voption<int>` |
| `dict.TryGetValue(k, &v)` | `Map.tryFind k m` returns `voption<'V>` |
| `Seq.head` throws on empty | `Seq.tryHead` returns `voption<'T>` |
| Nullable reference types | Not applicable - all types non-nullable |

**Nullness Constraint Not Supported**:

The nullness constraint `typar : null` is NOT SUPPORTED. Code using this constraint will produce a compile-time error. The `AllowNullLiteral` attribute has no effect in Clef.

### Default Initialization

Default initialization of values to _zero values_ is supported in Clef for types that have a well-defined zero representation.

> **Clef Note**: Default initialization is permitted only for types with explicit zero representations.

The following types permit _default initialization_:

- **Primitive value types**: `int`, `float`, `bool`, etc. (zero, 0.0, false)
- **Struct types**: Where all field types permit default initialization
- **Enum types**: Zero value of the underlying type
- **Arrays**: Initialized with default values for element type

The following types do NOT permit default initialization:

- **Records and unions**: Must be constructed with explicit values
- **Function types**: No default function value
- **Class types**: Must be constructed explicitly

The `Unchecked.defaultof<'T>` function is available but should be used with care - it produces zero-bit representations that may not be valid for all types. `Array.zeroCreate` creates arrays with default-initialized elements.

### Type Conversions

> **Clef Note**: Clef does not have "runtime types" in the managed F# sense. There is no boxing, no `System.Type`, and no runtime type discovery. All type conversions are verified at compile time.

**Static Type Coercion**:

Type coercion in Clef follows the static type hierarchy. A type `ty1` coerces to `ty2` (written `ty1 :> ty2`) if:

- `ty1` inherits from or implements `ty2`
- `ty1` and `ty2` are the same type

**Array Type Conversions**:

Arrays of compatible element types can be reinterpreted, verified at compile time:

- `int32[]` and `uint32[]` (reinterpret cast - same bit pattern)
- `int16[]` and `uint16[]`
- `int64[]` and `uint64[]`
- `nativeint[]` and `unativeint[]`
- `enum[]` and `underlying-type[]` (where enum has that underlying type)

These conversions are zero-cost reinterpret casts at the memory level.

**No Dynamic Type Tests**:

The `:?` and `:?>` operators for dynamic type testing are resolved statically in Clef:

```fsharp
// Statically resolvable - OK
let isString (x: obj) =
    match x with
    | :? string as s -> Some s  // Resolved at compile time for known type
    | _ -> None

// Not resolvable - compile error
let unknown (x: obj) = x :? SomeType  // Error if relationship unknown
 
```

Pattern matching type tests work when the type relationship can be determined at compile time from the type hierarchy.
