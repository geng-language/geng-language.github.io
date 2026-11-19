+++
title = "Refinement Types"
description = "Conditions on values, proved by the compiler: every place a refinement can be written, what a condition can hold, measures, how a condition is proved, and refinements in a package."
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
[Refinement types](@/docs/intro-to-refinement-types.md) for a guided
introduction.

## Where a refinement goes

| Place | Example |
|---|---|
| A function's argument | `(i : Int where 0 <= i)` |
| A function's result, named | `(r : Array x where Array.length r == Array.length a + 1)` |
| A callback's arguments and result | `((i : Int where 0 <= i && i < n) -> x)` |
| A `let` signature | as a top-level one, and it may also name the arguments of the signatures around it |
| A type alias, as its whole body | `type alias Percent = (p : Int where 0 <= p && p <= 100)` |
| A record alias, as a rule over the whole record | `(m : { items : Array Item, index : Int } where m.index < Array.length m.items)` |
| A record's field, after its type | `index : Int where index < Array.length items` |
| A record written in a signature | its fields' conditions, or a refined alias as a field's type |
| A variant's whole argument | `type Decoder = Decoder Lead`, where `Lead` is a refined alias or a record with refined fields |
| An array's or a builder's elements | `Array (x : Int where 0 <= x && x < 256)`, `Builder (c : Int where c < 16)`, `Array Nat` |
| A `Maybe`'s payload | `Maybe (x : Int where 0 <= x && x < 10)`, `Maybe Nat` |
| A `Set`'s elements | `Set (d : Int where 0 <= d && d < 16)` |
| A `Dict`'s keys and values | `Dict (k : Int where 0 <= k && k < 16) (v : Int where v > 0)`, `Dict String Nat` |
| The elements of a record alias's field | `cells : Array (c : Int where 0 <= c && c < 256)` |

What each place means:

- **A signature's names.** An argument, the result or a callback's argument
  can be named and refined. A refined argument is asked of every caller and
  known inside the function; a refined result is asked of the function and
  known by every caller. A callback is the other way round: its arguments
  are promised to it, and its result is asked of it.
- **A refined type alias** names a condition to write it once. `core` has
  `Nat`, an `Int` that is never negative. Where the alias is written, its
  condition is that argument's or result's. Its condition can read only its
  own value, so it cannot say "an index into this array": write that one
  out.
- **A record alias with a rule** is a *type invariant*. The rule holds of
  every record of the type, so the alias goes anywhere a type does,
  `Maybe Move` and `Array Move` included. The compiler checks the rule
  wherever such a record is built or updated, and knows it wherever one
  is read. The rule may be written on the whole record or on its fields;
  a field's condition reads the record's other fields bare, and only them.
  Two refined aliases of one record shape are an error, and so is a function
  generic in its record (`{ r | index : Int } -> ...`) that writes a field a
  rule reads.
- **A variant's argument** that is a refined alias, or a record with
  refined fields, is the constructor's contract: the constructor asks it of
  its argument wherever it is applied, and a `when` knows it of what it
  binds.
- **An element** of an `Array` or an `Array.Builder`, or the payload of a
  `Maybe`, holds of each element: known wherever one is read, and asked
  wherever an array is given. `core`'s array functions say what they do
  with elements, so `Array.set`, `pushLast`, a literal and what `map`'s or
  `initialize`'s callback makes are each checked. A lambda given to
  `map`, `indexedMap`, `keepIf`, `any`, `all`, `findFirst`, `update`,
  `sortBy`, `sortWith` or `Maybe.map` knows each element it is handed:
  `Array.map (\n -> needs n) sizes` is proved when `sizes`' elements are
  what `needs` asks. An element's condition can read only the element and
  what its signature's position may read.
- **A `Set`'s element, or a `Dict`'s key or value**, holds of each in the
  same way. `get`, `first`, `keys`, `values` and a fold read them, a lambda
  given to `map`, `keepIf`, `any`, `all`, `findFirst` or `partition` knows
  the key and value it is handed, and what `set`, `singleton`, `union`,
  `remove`, `update` and `map` make is checked.

A refinement cannot be written on a variant's argument directly
(`Decoder (lead : Int where ...)`; use an alias) or on an extensible record
(`{ r | index : Int where ... }`). Each is a `MISPLACED REFINEMENT` error.

A function that refines its arguments, or a constructor whose argument is
refined, cannot be used as a value, since the compiler could not see its
calls. Pass a lambda that calls it instead:
`Array.indexedMap (\i _ -> Array.getProven i b) a`.

## What a condition holds {#conditions}

| Kind | What |
|---|---|
| Numbers | integer literals, decimal or hex, and `^` between literals, `a < 2 ^ 128` |
| Names | the signature's arguments, the result's own name in the result's condition, a callback's arguments in the callback's conditions, a `let` signature's enclosing arguments |
| Fields | a name's record fields, to any depth: `m.index`, `o.move.index` |
| Value types | a name or field read directly is an integer, `Int`, `Int8`, `UInt8`, `Int16`, `UInt16`, `UInt32`, `Int64`, `UInt64` or `BigInt`, or a `Bool`, which is a condition of its own: `(room : Int where not keep \|\| 0 < room)` |
| Sizes | `Array.length`, `Array.Builder.length`, `Bytes.length`, `Bytes.Owned.length`, `String.count`, `Dict.count` and `Set.count`, of a name or a field |
| Which constructor | `Maybe.isJust` and `Result.isOk`, which are conditions |
| Your own measures | a function declared `@measure`, applied to a name or a field: `isEmpty s` ([below](#measures)) |
| Arithmetic | `+`, `-`, unary `-`, and `*` of two values at most (`2 * n`, `rows * cols`) |
| Powers | `b ^ k` with a literal base of at least 2 and an `Int` exponent a value names: `r < 2 ^ k` |
| Division | `//`, and `modBy`, `divFloor` and `remainderBy` of two numbers, divisor first: `tail == modBy 8 (head + size)` |
| Bits | `Bitwise`'s methods, qualified as code writes them, by a constant or between two values: `wire == Bitwise.and 7 key`; and `Bitwise.popCount` |
| `BigInt`'s functions | its exact ones, qualified: `add`, `mul`, `subBy`, `negate`, `fromInt`, `fromInt64`; the tests `isZero`, `isNegative`, `isEven`, `isOdd`, `fitsSigned`, `fitsUnsigned`; and the bits `and`, `or`, `xor`, `complement`, `shiftLeftBy`, `shiftRightBy`, `maskTo`, `toSigned`: `BigInt.fitsUnsigned 64 x`, `r == BigInt.fromInt n` |
| Comparisons | `<`, `<=`, `>`, `>=`, `==` and `/=` between two numbers; `==` and `/=` between two conditions |
| Logic | `&&`, `\|\|`, `not`, `True`, `False` and parentheses |

Nothing else:

- a call of a function that is not a measure is `NOT A MEASURE`;
- a float, a string or a record built in a condition is `BAD REFINEMENT`;
- a product of three values, `a * b * c`, is `NONLINEAR REFINEMENT`;
- `^` with a base that is not a literal, a negative exponent or one past
  4,096 is `EXPONENT IN A REFINEMENT`;
- `BigInt.abs`, `min` and `max`, which choose between two values, are `NOT A
  MEASURE`: write the comparisons, `-10 < x && x < 10` for `abs x < 10`.

**Types.** A condition is typed as code is. A bare literal takes the type of
what it is compared with (`b == 255` at `UInt8`), a literal outside that
type is refused, and comparing two types, an `Int` with a `UInt8`, is a type
mismatch. A name that no condition reads is an error.

**Arithmetic.** A condition's arithmetic is exact: it is never run, so
`at + 4 <= Bytes.length b` means what it says over the integers, and so does
`a < 2 ^ 128`. The program's own arithmetic still wraps at its type's width,
and where it feeds a refinement the compiler must show that it does not
wrap. When it may wrap, the error suggests a form that cannot, such as
`n <= length - offset` for `offset + n <= length`. `BigInt` has no width,
so its arithmetic owes nothing.

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
- the refinements on the enclosing function's arguments, the invariants of
  the records it is given, and what an element or a payload it reads is,
  including one a callback is handed;
- the signature of a `let`'s constant, `sizes : Array (n : Int where n <
  4096)`, wherever the constant is read, as for a top-level one;
- the promises in the results of functions called on the way. A promise
  holds only where its call was made: after
  `if b then below n else need n`, nothing `below` promised is known in the
  `else` branch;
- invariants it infers for private loops, `core`'s folds among them. It
  does not infer them for functions that call each other, or for facts that
  relate three values; those need a refinement written on them.

It reads the program's arithmetic as `Basics` computes it:

- `+`, `-` and `*` at each integer type, held to that type's range;
- `modBy`, `divFloor` and `remainderBy` exactly by a literal, and by any
  other divisor only what holds for each sign of it;
- `Bitwise` by a constant exactly, at every integer type, and between two
  values bit by bit; `popCount` from 0 to its type's width;
- `BigInt`'s own arithmetic, tests and bits by a constant exactly, and its
  shifts, masks and fits by a width a value names through `2 ^ k`;
- a product of two values with what holds of every product: two values at
  least 0 make one at least 0, and a row's cells end before the next row's
  begin, `y < rows` giving `y * cols + cols <= rows * cols`.

It follows a value chosen by an `if`, whether a number, an array, a record
(field by field) or a custom type (its constructor, and its `Int`
arguments), and understands `min`, `max` and `clamp`. `==` on records
compares field by field. A small helper of the same module, and a `let`'s
function with no signature, is walked at each call with that call's
arguments. Two calls of one function with the same arguments give the same
answer. `f <| x` and `x |> f` are calls like `f x`. A record literal is
checked with the values it was written with, and a record update with the
fields it changed and the rest of the record it started from.

The same expression written twice, such as a sum, is two separate values. To
test it once and use it, name it with `let`.

A condition the compiler cannot prove is a `CANNOT SHOW A REFINEMENT` error,
and a sum, product or power that may wrap is `ARITHMETIC MAY WRAP`, each
with values that break it. When a proof needs more cases than the checker tries, the error
says so rather than guessing.

## Asking without requiring

`@candidate` before an annotated value turns its refinements into
questions: each is checked and reported, and one the compiler cannot show
refuses nothing. A candidate must refine something, may not be exposed, and
nothing may name it, so no caller relies on what it could not show.

`GENG_REFINE_JSON=<file>` has `geng make` write every condition it checked,
proved or not, as one JSON document, for an editor, a CI annotation or a
tool that shows an error in its own terms.

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
