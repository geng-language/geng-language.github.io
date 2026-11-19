+++
title = "Intro to refinement types"
description = "Conditions on values that the compiler proves: what they are, how to write them, how the compiler knows, and what they cost."
weight = 23
+++

This page follows [Geng's type system](@/docs/type-system.md). For the exact
rules, see [Refinement Types](@/reference/refinement-types.md).

## What they are

A refinement type is a type plus a condition that its values must meet. In
Geng, you give an argument or a result a name, then add the condition after
`where`:

```geng
(n : Int where n > 0)                  -- an Int greater than zero
(p : Int where 0 <= p && p <= 100)     -- a percentage
(a : Array x where Array.length a > 0) -- an array that is not empty
```

A value of the refined type is still an `Int` or an `Array`. There is no
wrapper and nothing to unwrap: you can pass a positive `Int` anywhere an `Int`
is expected. What changes is that the compiler checks, before the program runs,
that a value really meets the condition wherever the refined type is required.
`geng make` refuses a program where it cannot prove one. Once checked, the
condition is erased. A refined program runs exactly the same code as an
unrefined one, with no check added at run time, on every target.

Full rules: [Refinement types](@/reference/refinement-types.md).

## Writing one

Refinements go in a function's type signature. A condition may name any
argument of the same signature, before or after it:

```geng
-- The index must be inside the array that comes after it.
getProven : (i : Int where 0 <= i && i < Array.length a) -> (a : Array x) -> x

-- The result promises something about itself and the arguments.
pushLast : x -> (a : Array x) -> (r : Array x where Array.length r == Array.length a + 1)

-- A callback's argument: `indexedMap` promises its function an index in range.
indexedMap : ((i : Int where 0 <= i && i < Array.length array) -> a -> b) -> (array : Array a) -> (r : Array b where Array.length r == Array.length array)

-- An array that is never empty, so its first element always exists.
firstOf : (a : Array x where Array.length a > 0) -> x
```

`getProven` needs no `Maybe` and no bounds check, because a call that might read
outside the array does not compile. A name that no condition reads is an
error, since it would look as if it meant something.

A condition is deliberately small. It can contain:

- integer literals, and the names of the signature's arguments (or the
  result's own name, in the result's condition);
- a record's fields, to any depth: `m.index`, `o.move.index`;
- a name or a field that is a `Bool`, which is a condition of its own (see
  [below](#conditions-on-a-bool));
- a few *measures* applied to an argument or a field, such as
  `Array.length`, `Bytes.length`, `String.count`, `Dict.count`,
  `Set.count` and a builder's `Builder.length`: `Array.length m.items`;
- `Maybe.isJust` and `Result.isOk`, which say which constructor a value is;
- a [measure of your own](#measures-of-your-own), a function declared
  `@measure`: `isEmpty s`;
- `+`, `-`, and `*` of two values at most: `2 * n` or `rows * cols`, but
  not `a * b * c`;
- `<`, `<=`, `>`, `>=`, `==` and `/=` between numbers, and `==` and `/=`
  between two conditions: `Maybe.isJust r == (Array.length a > 0)`;
- `&&`, `||`, `not`, `True`, `False` and parentheses.

Nothing else: no calls to your own functions that are not measures, no records built in a
condition, and no floats. A condition outside this list is a
`BAD REFINEMENT` error. The arithmetic in a
condition is exact, because it is never run: `at + 4 <= Bytes.length b` means
the same as `at <= Bytes.length b - 4`.

Refinements go on the arguments and result of a top-level or `let` signature,
including callbacks. A `let`'s condition may also name the arguments of the
signatures around it. Annotations on expressions cannot be refined yet; a
record is refined through an alias, as the next sections show.

Full rules: [Where a refinement goes](@/reference/refinement-types.md#where-a-refinement-goes)
and [Conditions](@/reference/refinement-types.md#conditions).

## Conditions on a `Bool` {#conditions-on-a-bool}

A `Bool` argument is a condition, so one argument can decide what another
owes. Here `room` must be positive only when the row is kept:

```geng
push : (keep : Bool) -> (room : Int where not keep || 0 < room) -> Int
push keep room =
    if keep then
        needsRoom room      -- proved: keep is True here, so 0 < room

    else
        0
```

`push True 3` and `push False 0` compile; `push True 0` does not. A caller
that tested what it passes owes nothing more: inside `if not keep || room >
0 then push keep room`, the call is proved. A record's `Bool` field can
guard a rule the same way, `(n : Int where not cfg.on || n < cfg.limit)`,
and a result can say what the `Bool` it answers means:

```geng
positive : (n : Int) -> (r : Bool where r == (0 < n))
positive n =
    n > 0

-- inside `if positive n`, the compiler knows 0 < n
```

This is how a function stops testing at run time what its caller already
knows. puritty's terminal core had two such tests, one in its scrolling and
one in its printing; both are gone, each replaced by a `Bool` the caller
passes, and the compiler proves the rest.

## Naming a refinement

A type alias can name a refinement, so you write it once. `core` provides
`Nat`, an `Int` that is never negative, and you can write your own:

```geng
-- In core's Basics:
type alias Nat =
    (n : Int where 0 <= n)

-- Your own:
type alias Percent =
    (p : Int where 0 <= p && p <= 100)

setVolume : Percent -> Task Never {}

-- Named, so that another condition can read it, and with more added:
getProven : (i : Nat) -> (a : Array x where i < Array.length a) -> x
```

Unlike the opaque `Percent` of [Opaque
types](@/docs/type-system.md#opaque-types-rules-the-compiler-cannot-see),
this one is a plain `Int`. `setVolume 50` compiles with no `Maybe`, and
`setVolume 150` does not compile at all.

A refined alias goes only where a signature names a value, or in a field of
a record written there. `Array Nat`, `Maybe Percent` and a field of type
`Nat` in another type alias are errors, because nothing there is checked. An
alias's condition reads only its own value, so `Nat` cannot say "an index
into this array": write that one out, as `getProven` does.

Full rules: [Type aliases](@/reference/types.md#type-aliases).

## Records with a rule

A record often has a rule that ties its fields together: an index that stays
inside the array beside it, a cursor that stays inside the screen. Write the
rule once, on the field it is about:

```geng
type alias Move =
    { items : Array Item
    , index : Int where 0 <= index && index < Array.length items
    }

-- Every caller must pass a Move that keeps the rule, so the read needs no test.
apply : Move -> Item
apply move =
    Array.getProven move.index move.items

-- Every Move this returns must keep it too, on both branches.
next : Move -> Move
next move =
    if move.index + 1 < Array.length move.items then
        { move | index = move.index + 1 }

    else
        move
```

A field's condition may name any of the record's fields, written bare. Give
each field the facts about it, and the record's rule is all of them together:

```geng
type alias Span =
    { start : Int where 0 <= start
    , end : Int where start <= end
    }
```

You can also write the rule on the whole record, naming it. This `Move` is the
same type as the one above:

```geng
type alias Move =
    (m : { items : Array Item, index : Int } where 0 <= m.index && m.index < Array.length m.items)
```

A rule that needs another argument of a signature goes on the whole record,
since a field's condition reads only its record's fields:

```geng
pick : (n : Int) -> (r : { index : Int } where r.index < n) -> Int
```

The compiler reads records the way you do. A record literal is checked with
the values it was written with, and an update with the fields it changed and
the rest of the record it started from. A field tested and then used is the
same value, with or without an alias. A record chosen by an `if` is either
branch's, field by field, so `m = if reset then { move | index = 0 } else
move` is still a `Move`.

A `Move` can also be a field of a record that a signature takes or returns.
The rule then applies to that field:

```geng
split : Move -> { move : Move, rest : Int }
```

A `Move`, or a record with a condition on a field, can also be a variant's
whole argument: `type Path = Path { bytes : Bytes, sep : Int where 0 <= sep
&& sep <= Bytes.length bytes }`. The constructor then asks its argument's
rule wherever it is applied, and a `when` that matches it knows the rule.

A record alias with a rule goes anywhere a type does, and its rule goes
with it. That is the next section.

Full rules: [Type aliases](@/reference/types.md#type-aliases) and
[Where a refinement goes](@/reference/refinement-types.md#where-a-refinement-goes).

## Rules a type keeps everywhere

A record alias with a rule is a *type invariant*: the rule holds of every
record of that type, wherever it is. The compiler checks it each time such
a record is built or updated, and knows it each time one turns up: as an
argument, out of a `Maybe`, as a fold's accumulator, from a `Task`, or as a
field of another record.

```geng
type alias Keys =
    { stack : Array Int
    , at : Int where 0 <= at && at < Array.length stack
    }

type alias Term =
    { keys : Keys
    , depth : Int
    }

flags : Term -> Int
flags t =
    -- proved: every `Keys` keeps its rule
    Array.getProven t.keys.at t.keys.stack

later : Task Never Term -> Task Never Int
later task =
    Task.map (\t -> Array.getProven t.keys.at t.keys.stack) task
```

Where the alias is written, it is just its type: a function that takes a
`Keys` asks nothing of its caller beyond being given a `Keys`, and it can be
passed around like any other function. The checking happens where records
are made, so a program states its data's rules once, on the types, and every
function that reads the data gets them for free.

[puritty](https://github.com/gilramir/puritty), a terminal emulator written
in Geng, keeps its screen this way: one buffer of cells whose length the
screen's rule ties to its rows, columns and scrollback, so every read of a
screen row is proved in range, through the terminal's state and the
alternate screen kept in a `Maybe`. The [home page](@/_index.md) shows a
cut-down version.

Two rules on one record shape are refused, since the compiler could not
tell which one a record keeps. So is a function generic in its record
(`{ r | at : Int } -> { r | at : Int }`) that writes a field a rule reads:
nothing there says which record it is updating.

## Measures of your own

A condition can call a function of your own if you declare it `@measure`. A
measure says something about each constructor of a custom type, and the
compiler reads it wherever the type is matched:

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

`side (Circle 3)` compiles, and so does `side s` after a `when` or an
`if isEmpty s` has ruled out `Empty`. `side s` with nothing known of `s` is a
`CANNOT SHOW A REFINEMENT` error that says what it needs: `s is not Empty`. A
measure that answers an `Int`, such as the number of corners a shape has,
works the same way, and so does one that reads a record's field, such as
`b.used`.

A measure takes one argument of a type declared in the same module, answers
an `Int` or a `Bool`, and its body is a `when` over that argument whose
branches are conditions' terms, or one such term. It cannot call itself yet,
so a tree's size is not a measure for now. A rule broken is a `NOT A
MEASURE` error naming it.

Full rules: [Measures](@/reference/refinement-types.md#measures).

## Things that are almost refinement types

You may have met parts of this idea already:

- **Ada and Pascal range types.** `subtype Percent is Integer range 0 .. 100;`
  is a refinement of `Integer`. Ada checks most of these at run time and raises
  an error when a value falls outside the range.
- **Assertions and contracts.** `assert n > 0` in Python, C or C++ states the
  condition but only checks it when that line runs.
- **Validation libraries.** Pydantic's `conint(gt=0)` in Python describes a
  positive integer, and checks it when data is loaded.

All of these check at run time. A refinement type is checked by the compiler,
so a violation is a compile error rather than a crash or an exception.

## How the compiler knows

The compiler learns facts from your code, the same way you do when you read
it. Suppose `safeDivide` requires a divisor that is not zero:

```geng
safeDivide : Int -> (d : Int where d /= 0) -> Int
```

Calling it with a literal is fine: the compiler can see that `4` is not zero.
Calling it with an arbitrary `Int` is a compile error, because that `Int` might
be zero. Checking first makes it compile:

```geng
average : Int -> Int -> Int
average total count =
    if count /= 0 then
        safeDivide total count      -- here the compiler knows count /= 0

    else
        0
```

At each call, the compiler uses what it knows at that point:

- the tests on the way to the call, such as the `if` above (a `when` on a
  number works the same way);
- the patterns on the way to the call: which constructor a value is, and
  how long an array is;
- the refinements on the calling function's own arguments;
- the promises in the results of functions it has called;
- facts it works out for itself about loops.

The last one means that a simple loop needs nothing written for it:

```geng
sum : Array Int -> Int
sum a =
    let
        go i acc =
            if i < Array.length a then
                go (i + 1) (acc + Array.getProven i a)

            else
                acc
    in
    go 0 0
```

`Array.getProven i a` needs `0 <= i` and `i < Array.length a`. The `if` gives
the second. The first comes from the loop: `go` starts at 0 and only adds 1,
and the compiler finds that by itself. As a rule, you write refinements on the
functions a module exposes, which are their contract, and the compiler infers
them for private loops. It does not infer them for functions that call each
other, or a loop whose facts relate three values; those need a refinement
written on them.

The compiler follows a value chosen by an `if`, whether it is a number, an
array or a record, and understands `min`, `max` and `clamp`. A test that
mixes a `Bool` with a comparison keeps the comparison: inside
`if wrap && x < cols - 1`, it knows `x < cols - 1`. `f <| x` and `x |> f`
are calls like `f x`. What a function's result promises holds only where the
call was made: after `if b then below n else need n`, nothing `below`
promised is known in the `else` branch. It treats the same sum written twice
as two values, so test a sum once, give it a name with `let`, and use the
name.

Patterns count as tests. An array pattern says how long the array is, in
its own branch and, because it did not match, in the branches after it. A
constructor says which one the value is, and matching the same value twice
gives the same contents both times:

```geng
second : Array Int -> Int
second xs =
    when xs is
        [] ->
            0

        [ x ] ->
            x

        _ ->
            Array.getProven 1 xs    -- xs is neither empty nor one long
```

A value built with a constructor, or chosen by an `if`, is matched as it was
written, so a branch no value can reach owes nothing. When a call cannot be
proved, the error lists what the compiler knows, such as `m is Just`.

Full rules: [How a condition is proved](@/reference/refinement-types.md#how-a-condition-is-proved).

## Refinements in `core`

`core` uses refinements for its own reads, and has two functions that take a
refined index and return the element directly:

```geng
Array.getProven : (i : Int where 0 <= i && i < Array.length array) -> (array : Array a) -> a
Bytes.getProven : (index : Int where 0 <= index && index < Bytes.length bytes) -> (bytes : Bytes) -> UInt8
```

Use them where a test you write anyway keeps the index in range, as a loop
does. Elsewhere, `Array.get` and `Bytes.get` still return `Maybe`. Note that
`Array.getProven` does not count a negative index from the end, as `Array.get`
does.

Other functions in `Array` promise something about what they return, so a
read after them can be proved without any test:

- `map`, `indexedMap`, `reverse` and `set` return an array of the same length;
- `pushFirst` and `pushLast` return one that is one longer;
- `prepend` and `append` return one as long as both together;
- `indexedMap`, `indexedFoldl` and `indexedFoldr` give their function an index
  inside the array;
- `first`, `last`, `popFirst`, `popLast`, `minimum` and `maximum` return a
  `Just` exactly when the array is not empty, and `get` exactly when the index
  is inside it (counting a negative one from the end), as `Bytes.get` does
  for bytes:

```geng
Array.first : (array : Array a) -> (r : Maybe a where Maybe.isJust r == (Array.length array > 0))
```

So a `Just` from `Array.first a` says `a` is not empty, and when the compiler
already knows `a` is not empty, the `Nothing` branch owes nothing.

So this compiles, with no `Maybe` and no test:

```geng
mirrored : Array Int -> Array Int
mirrored a =
    let
        back =
            Array.reverse a
    in
    Array.indexedMap (\i x -> x * 100 + Array.getProven i back) a
```

A function that promises something asks nothing of its caller, so a program
that calls only `Array.map` and the like builds exactly as it would without
refinements. A function that asks something of its arguments, as `getProven`
does, cannot be passed around as a value, because the compiler could not see
its calls. Pass a lambda that calls it instead:
`Array.indexedMap (\i _ -> Array.getProven i b) a`.

## Overflow is checked too

`Int` is 32 bits and [wraps on overflow](@/reference/operators.md), and the
checker knows it. Where the program's own `+` or `-` feeds a refined
condition, the compiler must also show that the arithmetic does not wrap.
Arithmetic that feeds no refinement wraps as it always has.

This catches a classic bug. A test like `offset + n <= length` looks right,
but with `offset = 1` and `n = 2147483647` the sum wraps to a negative number
and the test passes when it should fail. The compiler rejects it and suggests
a form that cannot wrap:

| refused: may wrap | accepted: means the same |
|---|---|
| `offset + n <= length` | `n <= length - offset` |
| `(lo + hi) // 2` | `lo + (hi - lo) // 2` |

Geng's own code had exactly this bug in its byte decoder, and four sums in
`core`'s sort wrapped past 2^30 elements. Refinement types found all of them,
and they are fixed.

Full rules: [Conditions](@/reference/refinement-types.md#conditions).

## When a refinement does not hold

An error says what was required, points at the call, lists each fact the
compiler had and where it came from, and gives values that break the
condition. When there is a rewrite the compiler can prove means the same, it
suggests it. This one reads the previous element of an array, but forgets to
check that there is one:

```geng
changes : Array Float -> Array Float
changes temps =
    Array.indexedMap (\i t -> t - Array.getProven (i - 1) temps) temps
```

```text
-- CANNOT SHOW A REFINEMENT -------------------------------------- src/Main.geng

`getProven` requires this of its argument `i`:

    0 <= i && i < Array.length array

and I cannot show it here:

8|     Array.indexedMap (\i t -> t - Array.getProven (i - 1) temps) temps
                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Here, with this call's values for its names, that is:

    0 <= i - 1 && i - 1 < Array.length temps

What I know here:

    0 <= i && i < Array.length temps
        from the refinement on `indexedMap`'s callback's `i`

With `Array.length temps = 1` and `i = 0`, it is False.
```

Testing `i > 0` before the read makes it compile.

## Many errors, a slow check, or a missing fact

Adding refinements to code that already exists usually brings many errors
at once, and three settings help with them. Each is an environment
variable on `geng make` (they become options when the command-line tool is
reviewed before release):

- **`GENG_REFINE_SUMMARY=FILE`** groups the errors by what was asked (a
  function's argument, a type's rule, a sum that may wrap), the largest
  group first, each with its places. When puritty first refined every read
  of its screen rows, its ninety errors were five causes.
- **`GENG_REFINE_EXPLAIN=1`** makes each error that says "I know nothing
  here that bears on it" say why: which values in the condition no fact
  mentions, and where a fact about each was let go of, such as one learned
  inside an `if` and dropped where its branches meet.
- **`GENG_REFINE_PROFILE=FILE`** writes how much work the check did for
  each definition, and its costliest questions. The work is counted in
  solver steps, not seconds, so two builds compare exactly: if a change makes
  the build slower, this says which function to look at.

## In a package

A refinement on an exposed function is part of its type, so changing one is a
semver change. `geng package diff`, `bump` and `validate` work out how big a
change it is: asking for less of an argument, or promising more in a result,
is a minor change; asking for more, or promising less, is a major one.
Rewriting a condition so it means the same thing is a patch.

A package is held to its own refinements: `geng make` in the package and
`geng package validate` refuse it when a condition is not proved. An
application also checks every package it uses on every build, so a newer
compiler, or a different version of a dependency, can never leave a package's
refinements unchecked.

Full rules: [In a package](@/reference/refinement-types.md#refinements-in-a-package).

## What it costs

Nothing at run time. At compile time, a program with no refinements, or one
that only calls functions like `Array.map` that promise something, builds
exactly as before. One that calls `getProven` or writes its own refinements
takes a little longer: a small program calling `Array.getProven` builds warm
in 0.30 to 0.33 seconds, against 0.26 to 0.29 for one calling `Array.map`.

## What they will not do

- **Conditions are limited.** Comparisons, arithmetic and sizes are all a
  condition can talk about. It cannot decide whether a string is a valid email
  address, or whether a key is in a `Dict`. Opaque types stay the right tool
  for rules like that.
- **Run-time checks do not go away.** Data from a file, the network or the user
  is not known in advance. You check it once, with an `if` or a `when`, and
  from then on the compiler carries the result. That is the same "parse, don't
  validate" approach as today, with less wrapping.
- **A `Maybe` hides what was checked of a plain value.** A record with a
  rule keeps it inside a `Maybe`, as [above](#rules-a-type-keeps-everywhere),
  but an `Int` does not. The value inside a `Maybe Int` cannot be
  refined, so a helper that returns `Maybe Int` loses what it tested.
- **The compiler may give up.** When a call needs more cases than the checker
  tries, it says so rather than guessing. A test just before the call, or a
  small function with the condition as a refinement, makes it decidable.
- **They are optional.** Code that never uses a refinement type is ordinary
  Geng.
