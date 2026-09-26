---
title: "Units of Measure"
weight: 160
category: Language
status: normative
---

A _unit of measure_ (_measure_) is a parameter of sort Measure that contributes
to a type's dimensional identity. A measure may occur in a numeric type, a
parameterized type or a quantified signature. Examples are `float<kg>`,
`Vector<m/s>` and `float<'U>`.

The numeric kinds are `int` and `float`, as specified by [NTU Types](ntu-types.md).
The implementation SHALL check measure compatibility using the equivalence and
inference rules in this chapter. It SHALL retain each established dimensional
judgment and its correspondence to lowered operations under
[Conformance §6](conformance.md#6-the-preservation-obligation-through-lowering).
[Width Inference](width-inference.md) and [Numeric Selection](numeric-selection.md)
specify the separate range and representation requirements.

The syntax of constants ([§](basic-grammar-elements.md#constants)) is extended to support numeric constants with units of measure. The syntax of types is extended with measure type annotations.

```fsgrammar
measure-literal-atom :=
    long-ident                                  -- named measure e.g. kg
    ( measure-literal-simp )                    -- parenthesized measure, such as (N m)

measure-literal-power :=
    measure-literal-atom
    measure-literal-atom ^ integer-exponent     -- power of measure, such as m^3

measure-literal-seq :=
    measure-literal-power
    measure-literal-power measure-literal-seq

measure-literal-simp :=
    measure-literal-seq                         -- implicit product, such as m s^- 2
    measure-literal-simp * measure-literal-simp -- product, such as m * s^3
    measure-literal-simp / measure-literal-simp -- quotient, such as m/s^2
    / measure-literal-simp                      -- reciprocal, such as /s
    1                                           -- dimensionless

measure-literal :=
    _                                           -- anonymous measure
    measure-literal-simp                        -- simple measure, such as N m

const :=
    ...
    integer-literal < measure-literal >         -- integer quantity
    real-literal < measure-literal >            -- real quantity

measure-atom :=
    typar                                       -- variable measure, such as 'U
    long-ident                                  -- named measure, such as kg
    ( measure-simp )                            -- parenthesized measure, such as (N m)

measure-power :=
    measure-atom
    measure-atom ^ integer-exponent             -- power of measure, such as m^3

measure-seq :=
    measure-power
    measure-power measure-seq

measure-simp :=
    measure-seq                                 -- implicit product, such as 'U 'V^3
    measure-simp * measure-simp                 -- product, such as 'U * 'V
    measure-simp / measure-simp                 -- quotient, such as 'U / 'V
    / measure-simp                              -- reciprocal, such as /'U
    1                                           -- dimensionless measure (no units)

measure :=
    _                                           -- anonymous measure
    measure-simp                                -- simple measure, such as 'U 'V
```

Measure definitions use the `Measure` attribute on type definitions. Measure parameters use the generic-parameter syntax described in [Measure Parameter Definitions](#measure-parameter-definitions). Both numeric kinds have dimensionless and measured forms: `int = int<1>` and `float = float<1>`. An `integer-exponent` is a signed decimal integer in the measure syntax.

Here is a simple example:

```fsharp
[<Measure>] type m          // base measure: meters
[<Measure>] type s          // base measure: seconds
[<Measure>] type sqm = m^2  // derived measure: square meters

let areaOfTriangle (baseLength:float<m>, height:float<m>) : float<sqm> =
    baseLength*height/2.0

let distanceTravelled (speed:float<m/s>, time:float<s>) : float<m> = speed*time
```

As with ordinary types, Clef can infer that functions are generic in their units. For example:

```fsharp
let sqr (x:float<_>) = x * x

let sumOfSquares x y = sqr x + sqr y
```

The inferred types are:

```fsother
val sqr : float<'u> -> float<'u ^ 2>

val sumOfSquares : float<'u> -> float<'u> -> float<'u ^ 2>
```

Measures are type-like annotations such as `kg` or `m/s` or `m^2`. Their special syntax includes the use of `*` and `/` for product and quotient of measures, juxtaposition as shorthand for product, and `^` for integer powers.

## Measures

Measures are built from:

- _Atomic measures_ from long identifiers such as `SI.kg` or `FreedomUnits.feet`.
- _Product measures_ , which are written `measure measure` (juxtaposition) or `measure * measure`.
- _Quotient measures_ , which are written `measure / measure`.
- _Integer powers of measures_ , which are written `measure ^ int`.
- _Dimensionless measures_ , which are written `1`.
- _Variable measures_, which are written `'u` or `'U`. Variable measures can include anonymous measures `_`, which indicates that the compiler can infer the measure from the context.

Dimensionless measures indicate “without units,” but are rarely needed, because non-parameterized types such as `float` are aliases for the parameterized type with `1` as parameter, that is, `float = float<1>`.

The precedence of operations involving measure is similar to that for floating-point expressions:

- Products and quotients (`*` and `/`) have the same precedence, and associate to the left, but juxtaposition has higher syntactic precedence than both `*` and `/`.
- Integer powers (`^`) have higher precedence than juxtaposition.
- The `/` symbol can also be used as a unary reciprocal operator.

## Constants Annotated by Measures

A numeric constant can be annotated with its measure in angle brackets following the constant. An integer literal remains `int`; a real literal remains `float`. Its representation SHALL be selected from its justified range and target declarations, separately from its measure identity.

Measure annotations on constants may not include measure variables.

Here are some examples of annotated constants:

```fsharp
let earthGravity = 9.81<m/s^2>
let atmosphere = 101325.0<N m^-2>
let zero: float<m> = 0.0<_>
```

Constants SHALL receive the indicated numeric kind and measure. Here `earthGravity` has type `float<m/s^2>`, `atmosphere` has type `float<N/m^2>`, and the anonymous measure of `zero` is inferred as `m` from its annotation. An anonymous measure SHALL participate in the same constraint solving as an explicit measure variable.

## Relations of Measures

After measures are parsed and checked, they are maintained in the following normalized form:

```fsgrammar
measure-int := 1 | long-ident | measure-par | measure-int measure-int | / measure-int
```

Powers of measures are expanded. For example, `kg^3` is equivalent to `kg` `kg` `kg`.

Two measures are indistinguishable if they can be made equivalent by repeated application of the following rules:

- _Commutativity_. `measure-int1 measure-int2` is equivalent to `measure-int2 measure-int1`.
- _Associativity_. It does not matter what grouping is used for juxtaposition (product) of measures, so parentheses are not required. For example, `kg m s` can be split as the product of `kg m` and `s`, or as the product of `kg` and `m s`.
- _Identity_. 1 `measure-int` is equivalent to `measure-int`.
- _Inverses_. `measure-int / measure-int` is equivalent to `1`.
- _Abbreviation_. `long-ident` is equivalent to `measure` if a measure abbreviation of the form `[<Measure>] type long-ident = measure` is currently in scope.

Note that these are the [laws of Abelian groups](https://arxiv.org/abs/2603.16437) together with expansion of abbreviations.

For example, `kg m / s^2` is the same as `m kg / s^2`.

For presentation purposes (for example, in error messages), measures are presented in the normalized form that appears at the beginning of this section, but with the following restrictions:

- Powers are positive and greater than 1. This splits the measure into positive powers and negative powers, separated by `/`.
- Atomic measures are ordered as follows: measure parameters first, ordered alphabetically, followed by measure identifiers, ordered alphabetically.

For example, the measure expression `m^1 kg s^-1` would be normalized to `kg m / s`.

This normalized form provides a convenient way to check the equality of measures: given two measure expressions `measure-int1` and `measure-int2` , reduce each to normalized form by using the rules of commutativity, associativity, identity, inverses, and abbreviation, and then compare the syntax.

To check the equality of two measures, abbreviations are expanded to compare their normalized forms. However, abbreviations are not expanded for presentation. For example, consider the following definitions:

```fsharp
[<Measure>] type a
[<Measure>] type b = a * a
let x = 1<b> / 1<a>
```

The inferred type is presented as `int<b/a>`, not `int<a>`. If a measure is equivalent to `1`, however, abbreviations are expanded to cancel each other and are presented without units:

```fsharp
let y = 1<b> / 1<a a> // val y : int = 1
 
```

### Constraint Solving

The mechanism described in [§](inference-constraint-solving.md#constraint-solving) is extended to support equational constraints between measure expressions. Such expressions arise from equations between parameterized types, that is, when `type<tyarg11 , ..., tyarg1n> = type<tyarg21, ..., tyarg2n>` is reduced to a series of constraints `tyarg1i = tyarg2i`. For the arguments that are measures, rather than types, the rules listed in [§](units-of-measure.md#relations-of-measures) are applied to obtain primitive equations of the form `'U = measure-int` where `'U` is a measure variable and `measure-int` is a measure expression in internal form. The variable `'U` is then replaced by `measure-int` wherever else it occurs. For example, the equation `float<m^2/s^2> = float<'U^2>` would be reduced to the `constraint m^2/s^2 = 'U^2`, which would be further reduced to the primitive equation `'U = m/s`.

If constraints cannot be solved, a type error occurs. For example, the following expression

```fsharp
fun (x : float<m^2>, y : float<s>) -> x + y
```

would eventually result in the constraint `m^2 = s`, which cannot be solved, indicating a type error.

### Generalization of Measure Variables

Analogous to the process of generalization of type variables described in [§](inference-constraint-solving.md#generalization), a generalization procedure produces measure variables over which a value, function, or member can be generalized.

Two measure schemes are equivalent when they admit the same instances under the
unit equations. Generalization SHALL respect this equivalence and SHALL preserve
constraints imposed by mutable state and the enclosing environment. For example,
`forall 'u. float<'u> -> float<1/'u>` and
`forall 'a 'b. float<'a*'b> -> float<1/('a*'b)>` admit the same instances.

Each use of a generalized scheme SHALL retain its actual measure substitution.
A callable's public signature, arguments, result, captures and body SHALL be
consistent under that substitution. Each instantiated occurrence SHALL retain
its dimensional identity when implementation code is shared. Instantiation SHALL
preserve the identity, evaluation multiplicity and sharing of retained
environments and deferred computations.

> NOTE — Kennedy, *Types for Units-of-Measure: Theory and Practice*, §§3.8–3.10,
> describes scheme equivalence and a scheme normal form based on Hermite
> normalization. Normalizing an entire quantified scheme involves relations
> among its variables as well as the normal form of each unit expression.

## Measure Definitions

Measure definitions define new named units of measure by using the same syntax as for type definitions, with the addition of the `Measure` attribute. For example:

```fsharp
[<Measure>] type kg
[<Measure>] type m
[<Measure>] type s
[<Measure>] type N = kg * m / s^2
```

A primitive measure definition introduces a fresh, named measure distinct from other measures. A measure abbreviation defines a new name for an existing measure. Repeatedly eliminating abbreviations must not result in an infinite measure expression. For example, the following definition is invalid:

```fsharp
[<Measure>] type X = X^2
```

Measure definitions and abbreviations may not have type or measure parameters.

## Measure Parameter Definitions

Measure parameter definitions can appear wherever ordinary type parameter definitions can (see [§](types-and-type-constraints.md#unmanaged-constraints)). If an explicit parameter definition is used, the parameter name is prefixed by the special `Measure` attribute. For example:

```fsharp
val sqr<[<Measure>] 'U> : float<'U> -> float<'U^2>

type Vector<[<Measure>] 'U> =
    { X: float<'U>;
      Y: float<'U>;
      Z: float<'U> }

type Sphere<[<Measure>] 'U> =
    { Center:Vector<'U>;
      Radius:float<'U> }

type Disc<[<Measure>] 'U> =
    { Center:Vector<'U>;
      Radius:float<'U>;
      Norm:Vector<1> }

type SceneObject<[<Measure>] 'U> =
    | Sphere of Sphere<'U>
    | Disc of Disc<'U>
```

The type checker distinguishes ordinary type parameters from measure parameters by their _sorts_ (Type or Measure). Actual arguments must have the declared sort. For example, `float<int>` is ill-formed because `int` is a type, and `seq<m/s>` is ill-formed because a sequence element requires a type. `seq<float<m/s>>` has a properly typed element.

## Measure Parameters Past the Checker

The program graph SHALL retain the measure of each value after checking.
Application resolution SHALL use the measured type and its actual scheme
instantiation. Lowering SHALL preserve those facts beside the selected
representation of the numeric kind ([Width Inference](width-inference.md),
[Numeric Selection](numeric-selection.md)).

Each lowering consumer SHALL have access to the measured types, instantiations
and proof premises its contract requires. A lowering step MAY release source
type structure after those consumers are satisfied, provided the resulting
operation retains the required correspondence and preservation evidence under
[Conformance §6](conformance.md#6-the-preservation-obligation-through-lowering).
Source associations required by emitted debug metadata SHALL also be retained.

A graph relation that supplies a numerical enclosure or recurrence certificate
SHALL validate the dimensional compatibility of its operands, state updates and
results independently of the numerical proof. The relation SHALL retain those
typed premises as dependencies. A change to a dimensional premise SHALL retract
the dependent proof until the relation has been revalidated.

## Numeric Operations with Measures

The following signatures define dimensional constraints on native operations.
`N` is consistently either `int` or `float` within an operation; `F` is `float`.

| Operation                                         | Measure Type                  |
| ------------------------------------------------- | ----------------------------- |
| Square root                                       | `F<'U^2> -> F<'U>`            |
| Two-argument arctangent                            | `F<'U> -> F<'U> -> F<1>`      |
| Addition, subtraction, remainder                   | `N<'U> -> N<'U> -> N<'U>`     |
| Multiplication                                    | `N<'U> -> N<'V> -> N<'U 'V>`  |
| Division                                          | `N<'U> -> N<'V> -> N<'U/'V>`  |
| Absolute value, unary negation, unary plus         | `N<'U> -> N<'U>`              |
| Sign                                              | `N<'U> -> int`                |

Addition, subtraction, remainder and comparison require matching measures.
Multiplication composes them by product; division composes them by quotient.
Assignment must preserve the target's measured type. Thus repeatedly multiplying
an `int<m>` state by a dimensionless `int` preserves its dimension, while
assigning an `int<m*s>` product back to that state is a type error.

Arithmetic SHALL also satisfy its domain, intermediate-capacity, rounding and
reassociation requirements under [Numeric Selection §10](numeric-selection.md#10-the-preservation-chain-and-arithmetic-construction)
and [Width Inference](width-inference.md).

## Restrictions

Measures can be used in range expressions but a properly measured step is required. For example, these are not allowed:

```fsharp
[<Measure>] type s
[1<s> .. 5<s>] // error: The type 'int<s>' does not match the type 'int'
[1<s> .. 1 .. 5<s>] // error: The type 'int<s>' does not match the type 'int'
 
```

However, the following range expression is valid:

```fsharp
[1<s> .. 1<s> .. 5<s>] // int<s> list = [1; 2; 3; 4; 5]
 
```
