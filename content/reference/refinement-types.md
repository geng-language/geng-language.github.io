+++
title = "Refinement Types"
description = "Conditions on arguments and results, proved by the compiler: where a refinement goes, conditions, measures, how a condition is proved, and refinements in a package."
weight = 42
+++

An argument or a result of a signature can be named, `(a : Array x)`, and
refined with a condition after `where`, `(i : Int where 0 <= i)`. The
condition may name any argument of the same signature, before or after it:

```geng
getProven : (i : Int where 0 <= i && i < Array.length a) -> (a : Array x) -> x
```

The compiler proves every condition at compile time, then erases it: a
refined type is its underlying type at run time. See
[Refinement types](@/docs/type-system.md#refinement-types) for a guided
introduction.

## Where a refinement goes

- On the arguments and the result of a top-level or `let` signature,
  callbacks included. A `let` signature's condition may also name the
  arguments of the signatures around it.
- As the whole body of a type alias, including an alias over a record.
- On a record's field, after its type:
  `index : Int where index < Array.length items`. The condition reads the
  record's fields bare, and only them. A record with one is refined like a
  refined alias.
- A refined alias may be a field of a record written in a signature.
- A variant's whole argument may be a refined alias, or a record with a
  refined field: `type Decoder = Decoder Lead`. The constructor then asks
  it of its argument wherever it is applied, and a `when` knows it of what
  it binds.

A refinement cannot be written on a variant's argument itself
(`Decoder (lead : Int where ...)`), in a type's argument (`Maybe { index :
Int where ... }`; name the record with an alias instead), on an extensible
record or on an expression's annotation.

A function that refines its arguments, or a constructor whose argument is
refined, cannot be used as a value, since the compiler could not see its
calls. Pass a lambda that calls it instead:
`Array.indexedMap (\i _ -> Array.getProven i b) a`.

## Conditions

- A condition holds integer literals, the signature's names and their
  fields (`m.index`, `o.move.index`), the measures `Array.length`,
  `Bytes.length`, `Bytes.Owned.length`, `String.count`, `Dict.count`,
  `Set.count` and `Array.Builder.length` of a name or a field, the
  measures `Maybe.isJust` and `Result.isOk`, which are conditions, a
  [measure of your own](#measures) applied to a name or a field, `+`, `-`,
  `*` of two values at most, the six comparisons between numbers, `==`
  and `/=` between two conditions, and `&&`, `||`, `not`, `True` and
  `False`. A name or field read directly must be an `Int`, or a `Bool`,
  which is a condition: `(room : Int where not keep || 0 < room)`.
- Nothing else: no calls to functions that are not measures, no records
  built in a condition, and no floats. Anything else is a `BAD REFINEMENT`
  error, or `NOT A MEASURE` for a call of a function not declared one.
- A condition's arithmetic is exact. The program's own `Int` arithmetic still
  wraps at 32 bits, and where it feeds a refinement the compiler must show
  that it does not wrap. When it may wrap, the error suggests a form that
  cannot, such as `n <= length - offset` for `offset + n <= length`.
- A name that no condition reads is an error.

## Measures {#measures}

A function declared `@measure` may be called in a condition, and the
compiler reads what it says of each constructor:

```geng
type Shape
    = Circle Int
    | Square Int
    | Empty


@measure
isEmpty : Shape -> Bool
isEmpty s =
    when s is
        Empty ->
            True

        _ ->
            False


side : (s : Shape where not (isEmpty s)) -> Int
```

`side (Circle 3)` is proved, and so is `side s` after a `when` or an `if
isEmpty s` that ruled out `Empty`. A measure:

- takes one argument, named in its definition, of a custom type or record
  alias declared in the measure's own module;
- answers an `Int` or a `Bool`;
- has a body that is one `when` over its argument, each branch a condition's
  term over what its pattern binds, or such a term over the argument itself,
  such as `b.used`;
- does not call itself (recursive measures are coming).

Each rule broken is a `NOT A MEASURE` error naming it. Its own module calls
it bare, `isEmpty s`; another module as `Shape.isEmpty s`, or bare when the
import exposes it.

## How a condition is proved

Where a refined type is required, the compiler uses:

- the tests on the way there, from an `if` or a `when` on a number. A test
  that joins a `Bool` and a comparison with `&&` keeps the comparison;
- the patterns on the way there. A constructor pattern says which
  constructor the value is, and what its arguments' patterns say of them; an
  array pattern says the array's length (`[]` is 0, `[a, b]` is 2); a record
  pattern says what its fields' patterns say, to any depth. A branch knows
  that the patterns before it did not match. Matching one value twice gives
  the same arguments both times;
- the refinements on the enclosing function's arguments;
- the promises in the results of functions called on the way. A promise
  holds only where its call was made: after
  `if b then below n else need n`, nothing `below` promised is known in the
  `else` branch;
- invariants it infers for private loops. It does not infer them for
  functions that call each other, or for facts that relate three values;
  those need a refinement written on them.

It follows a value chosen by an `if`, whether a number, an array, a record
(field by field) or a custom type (its constructor, and its `Int`
arguments), and understands `min`, `max` and `clamp`.
`f <| x` and `x |> f` are calls like `f x`. A record literal is checked with
the values it was written with, and a record update with the fields it
changed and the rest of the record it started from.

The same expression written twice, such as a sum, is two separate values. To
test it once and use it, name it with `let`.

A condition the compiler cannot prove is a `CANNOT SHOW A REFINEMENT` error.
When a proof needs more cases than the checker tries, the error says so
rather than guessing.

## In a package {#refinements-in-a-package}

A refinement on an exposed function is part of its type. Asking less of an
argument, or promising more in a result, is a minor change; asking more, or
promising less, is a major one. Rewriting a condition so it means the same
is a patch. `geng package diff`, `bump` and `validate` work this out. A
condition that calls the package's own measure is compared by its text, and
a change to a measure's body is not yet seen.

`geng make` in a package and `geng package validate` refuse a package whose
conditions are not proved. An application checks every package it uses on
every build.
