---
title: "Supplementary Inference Procedures"
weight: 650
category: Compiler
status: normative
---

This chapter covers additional inference procedures including dispatch slot inference, byref safety analysis, and arity inference. This is a component of the overall [Inference Procedures](inference-procedures.md).

## Dispatch Slot Inference

The F# compiler applies _Dispatch Slot Inference_ to object expressions and type definitions before it
processes their members. For both object expressions and type definitions, the following are input
to _Dispatch Slot Inference_:

- A type `ty0` that is being implemented.
- A set of members `override x.M(arg1 ... argN)`.
- A set of additional interface types `ty1 ... tyn`.
- A further set of members `override x.M(arg1 ... argN)` for each `tyi`.

Dispatch slot inference associates each member with a unique abstract member or interface
member that the collected types `tyi` define or inherit.

The types `ty0 ... tyn` together imply a collection of _required types_ R, each of which has a set of
_required dispatch slots_ SlotsR of the form `abstract M : aty1 ... atyN -> atyrty`. Each dispatch slot is
placed under the _most-specific_ `tyi` relevant to that dispatch slot. If there is no most-specific type for
a dispatch slot, an error occurs.

For example, assume the following definitions:

```fsharp
type IA = interface abstract P : int end
type IB = interface inherit IA end
type ID = interface inherit IB end
```

With these definitions, the following object expression is legal. Type `IB` is the most-specific
implemented type that encompasses `IA`, and therefore the implementation mapping for `P` must be
listed under `IB`:

```fsharp
let x = { new ID
          interface IB with
            member x.P = 2 }
```

But given:

```fsharp
type IA = interface abstract P : int end
type IB = interface inherit IA end
type IC = interface inherit IB end
type ID = interface inherit IB inherit IC end
```

then the following object expression causes an error, because both `IB` and `IC` include the interface
`IA`, and consequently the implementation mapping for `P` is ambiguous.

```fsharp
let x = { new ID
          interface IB with
            member x.P = 2
          interface IC with
            member x.P = 2 }
```

The ambiguity can be resolved by explicitly implementing interface `IA`.

After dispatch slots are assigned to types, the compiler tries to associate each member with a
dispatch slot based on name and number of arguments. This is called _dispatch slot inference,_ and it
proceeds as follows:

- For each `member x.M(arg1 ... argN)` in type `tyi`, attempt to find a single dispatch slot in the form

    ```fsgrammar
    abstract M : aty1 ... atyN -> rty
    ```

    with name `M`, argument count `N`, and most-specific implementing type `tyi`.

- To determine the argument counts, analyze the syntax of patterns and look specifically for
    tuple and unit patterns. Thus, the following members have argument count 1, even though
    the argument type is unit:

    ```fsgrammar
    member obj.ToString(() | ()) = ...
    member obj.ToString(():unit) = ...
    member obj.ToString(_:unit) = ...
    ```

- A member may have a return type, which is ignored when determining argument counts:

    ```fsother
    member obj.ToString() : string = ...
    ```

For example, given

```fsharp
let obj1 =
    { new IComparer<int> with
        member x.Compare(a,b) = compare (a % 7) (b % 7) }
```

the types of `a` and `b` are inferred by looking at the signature of the implemented dispatch slot, and
are hence both inferred to be `int`.

## Dispatch Slot Checking

_Dispatch Slot Checking_ is applied to object expressions and type definitions to check consistency
properties, such as ensuring that all abstract members are implemented.

After the compiler checks all bodies of all methods, it checks that a one-to-one mapping exists
between dispatch slots and implementing members based on exact signature matching.

The interface methods and abstract method slots of a type are collectively known as _dispatch slots_.
Each object expression and type definition results in an elaborated _dispatch map_. This map is keyed
by dispatch slots, which are qualified by the declaring type of the slot. This means that a type that
supports two interfaces `I` and `I2`, both of which contain the method m, may supply different
implementations for `I.m()` and `I2.m()`.

The construction of the dispatch map for any particular type is as follows:

- If the type definition or extension has an implementation of an interface, mappings are added
    for each member of the interface,
- If the type definition or extension has a `default` or `override` member, a mapping is added for the
    associated abstract member slot.

## Byref Safety Analysis

Byref arguments are pointers that can be stack-bound and are used to pass values by reference to
procedures, often to simulate multiple return values. Byref pointers are not often used in F#; more
typically, tuple values are used for multiple return values. However, a byref value can result from
calling or overriding a method that has a signature that involves one or more byref values.

To ensure the safety of byref arguments, the following checks are made:

- Byref types may not be used as generic arguments.
- Byref values may not be used in any of the following:
  - The argument types or body of function expressions `(fun ... -> ...)`.
  - The member implementations of object expressions.
  - The signature or body of let-bound functions in classes.
  - The signature or body of let-bound functions in expressions.

Note that function expressions occur in:

- The elaborated form of sequence expressions.
- The elaborated form of computation expressions.
- The elaborated form of partial applications of module-bound functions and members.

In addition:

- A generic type cannot be instantiated by a byref type.
- An object field cannot have a byref type.
- A static field or module-bound value cannot have a byref type.

As a result, a byref-typed expression can occur only in these situations:

- As an argument to a call to a module-defined function or class-defined function.

- On the right-hand-side of a value definition for a byref-typed local.

These restrictions also apply to uses of the prefix && operator for generating native pointer values.

## Promotion of Escaping Mutable Locals to Objects

Value definitions whose byref address would be subject to the restrictions on `byref<_>` listed in
[§](inference-supplementary.md#byref-safety-analysis) are treated as implicit declarations of reference cells. For example

```fsharp
let sumSquares n =
    let mutable total = 0
    [ 1 .. n ] |> Seq.iter (fun x -> total <- total + x*x)
    total
```

is considered equivalent to the following definition:

```fsharp
let sumSquares n =
    let total = ref 0
    [ 1 .. n ] |> Seq.iter
                    (fun x -> total.contents <- total.contents + x*x)
    total.contents
```

because the following would be subject to byref safety analysis:

```fsharp
let sumSquares n =
    let mutable total = 0
    &total
```

## Arity Inference

During checking, members within types and function definitions within modules are inferred to have
an _arity_. An arity includes both of the following:

- The number of iterated (curried) arguments `n`
- A tuple length for these arguments `[A1; ...; An]`. A tuple length of zero indicates that the
    corresponding argument is of type `unit`.

Arities are inferred as follows. A function definition of the following form is given arity `[A1; ...; An]`,
where each `Ai` is derived from the tuple length for the final inferred types of the patterns:

```fsother
let ident pat 1 ... patn = ...
```

For example, the following is given arity [1; 2]:

```fsharp
let f x (y,z) = x + y + z
```

Arities are also inferred from function expressions that appear on the immediate right of a value
definition. For example, the following has an arity of [1]:

```fsharp
let f = fun x -> x + 1
```

Similarly, the following has an arity of [1;1]:

```fsharp
let f x = fun y -> x + y
```

Arity inference is applied partly to help define the elaborated form of a function definition. This is
the form used in the compiled output. In particular:

- A function value `F` in a module that has arity `[A1 ; ...; An]` and the type
    `ty1,1 * ... * ty1,A1 -> ... -> tyn,1 * ... * tyn,An - > rty`
    elaborates to a static function with signature
    `rty F(ty1,1, ..., ty1,A1, ..., tyn,1 , ..., tyn,An)`.
- F# instance (respectively static) methods that have arity `[A1 ; ...; An]` and type
    `ty1,1 * ... * ty1,A1 -> ... -> tyn,1 * ... * tynAn -> rty`
    elaborate to an instance (respectively static) method definition with signature
    `rty F(ty1,1, ..., ty1,A1)`, subject to the syntactic restrictions that result from the patterns that
    define the member, as described later in this section.

For example, consider a function in a module with the following definition:

```fsharp
let AddThemUp x (y, z) = x + y + z
```

This function compiles to a static function with the following signature:

```fsharp
int AddThemUp(int x, int y, int z);
```

Arity inference applies differently to function and member definitions. Arity inference on function
definitions is fully type-directed. Arity inference on members is limited if parentheses or other
patterns are used to specify the member arguments. For example:

```fsharp
module Foo =
    // compiles as a static method taking 3 arguments
    let test1 (a1: int, a2: float, a3: string) = ()

    // compiles as a static method taking 3 arguments
    let test2 (aTuple : int * float * string) = ()

    // compiles as a static method taking 3 arguments
    let test3 ( (aTuple : int * float * string) ) = ()

    // compiles as a static method taking 3 arguments
    let test4 ( (a1: int, a2: float, a3: string) ) = ()

    // compiles as a static method taking 3 arguments
    let test5 (a1, a2, a3 : int * float * string) = ()

type Bar() =
    // compiles as a static method taking 3 arguments
    static member Test1 (a1: int, a2: float, a3: string) = ()
    
    // compiles as a static method taking 1 tupled argument
    static member Test2 (aTuple : int * float * string) = ()
    
    // compiles as a static method taking 1 tupled argument
    static member Test3 ( (aTuple : int * float * string) ) = ()
    
    // compiles as a static method taking 1 tupled argument
    static member Test4 ( (a1: int, a2: float, a3: string) ) = ()
    
    // compiles as a static method taking 1 tupled argument
    static member Test5 (a1, a2, a3 : int * float * string) = ()
```

## Additional Constraints on Common Methods

F# treats some methods and types specially, because they are common in F# programming and
cause extremely difficult-to-find bugs. For each use of the following constructs, the F# compiler
imposes additional _ad hoc_ constraints:

`x.Equals(yobj)` requires type `ty : equality` for the static type of `x`

`x.GetHashCode()` requires type `ty : equality` for the static type of `x`

`Map.empty<A,B>` requires `A : comparison`

`Set.empty<A>` requires `A : comparison`

> **Clef Note**: The native library defines collection types with appropriate constraints. `Map<'Key,'Value>` requires `'Key : comparison`, and `Set<'T>` requires `'T : comparison`. These constraints are enforced at compile time.
