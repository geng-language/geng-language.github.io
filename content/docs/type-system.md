+++
title = "Geng's type system"
description = "Custom types, pattern matching and planned refinement types, for people who know Python, Go, C or C++."
weight = 22
+++

Most of Geng's type system comes from Elm and Gren, and those languages got it
from the ML family: records, custom types and pattern matching. If you have
written Python, Go, C or C++, records will look familiar. Custom types will
look partly like enums, but they can do more. Geng also plans to add
refinement types, which none of those languages have.

This page explains the ideas. For the exact rules, see
[Types](@/reference/types.md).

## A type is a set of values

Read a type as the set of values it allows. `Bool` has two values. `UInt8` has
256. `String` has infinitely many.

Every type in this page is a way to build a new set from old ones:

- A **record** combines values: a `Point` is an `x` *and* a `y`.
- A **custom type** chooses between values: a `Shape` is a circle *or* a
  rectangle.
- A **refinement type** (planned) keeps only some values of another type: an
  `Int` that is greater than zero.

The aim is the same each time: make the type allow exactly the values that
make sense, so the compiler can reject the rest.

## Records: this *and* that

A record is a group of named fields, like a C `struct` or a Go `struct`:

```geng
type alias User =
    { name : String
    , age : Int
    }
```

Differences from what you may know:

- **No methods.** Functions that work on a `User` are ordinary functions that
  take a `User`.
- **No inheritance.** One record type cannot extend another. A function can
  still accept "any record with a `name` field", which covers most of what
  inheritance is used for. See [extensible records](@/reference/types.md#records).
- **No mutation.** `{ user | age = 37 }` makes a new record and leaves `user`
  unchanged.
- **No tuples.** Use a small record instead: `{ min = 0, max = 10 }`.

## Custom types: this *or* that

### Starting from enums

An enum in C, Python or Go lists the values a type can have:

```c
enum Direction { NORTH, EAST, SOUTH, WEST };
```

```python
class Direction(Enum):
    NORTH = 1
    EAST = 2
    SOUTH = 3
    WEST = 4
```

In Geng the same thing is a custom type:

```geng
type Direction
    = North
    | East
    | South
    | West
```

So far the only change is notation. Two things are already stricter, though:

- A `Direction` is not an integer underneath. You cannot pass `3` where a
  `Direction` is expected, or add two directions by mistake.
- There is no value outside the list. In C an `enum Direction` variable can
  hold `42`. In Python, nothing stops you from passing a string where a
  `Direction` is expected.

### Cases that carry data

Enums run out when each case needs different data. Say a shape is either a
circle with a radius or a rectangle with a width and a height.

In C you write a tagged union by hand:

```c
enum ShapeKind { CIRCLE, RECTANGLE };

struct Shape {
    enum ShapeKind kind;
    union {
        struct { double radius; } circle;
        struct { double width, height; } rectangle;
    };
};
```

Nothing stops code from reading `shape.rectangle.width` when `kind` is
`CIRCLE`. Nothing reminds you to update every `switch` when you add a
triangle.

In Python you might write a class per case, or one class with optional fields:

```python
@dataclass
class Shape:
    kind: str
    radius: float | None = None
    width: float | None = None
    height: float | None = None
```

This allows a "circle" with a width, or a shape with no fields set at all.

In Go you would use an interface and a type switch, which is safer, but the
compiler does not check that the switch covers every type.

In Geng, each case of a custom type can carry its own data:

```geng
type Shape
    = Circle { radius : Float }
    | Rectangle { width : Float, height : Float }
```

A `Shape` is exactly one of these. A circle has a radius and nothing else.
There is no way to build a circle with a width, and no `kind` field to get out
of step with the data.

The names `Circle` and `Rectangle` are called *constructors*. They are also
functions: `Circle { radius = 2.0 }` builds a `Shape`.

Other names for the same idea: tagged union, sum type, discriminated union,
variant. C++'s `std::variant` and Rust's `enum` are close relatives.

### Pattern matching

To use a `Shape`, you match on it with `when ... is`:

```geng
area : Shape -> Float
area shape =
    when shape is
        Circle { radius } ->
            3.14159 * radius * radius

        Rectangle { width, height } ->
            width * height
```

Each branch names a constructor and pulls its fields out. Inside the `Circle`
branch there is a `radius` and no `width`, so reading the wrong field cannot
happen.

The compiler checks that every `when` covers every case. Add a triangle:

```geng
type Shape
    = Circle { radius : Float }
    | Rectangle { width : Float, height : Float }
    | Triangle { base : Float, height : Float }
```

and every `when` on a `Shape` that does not handle `Triangle` is now a compile
error that points at it. In C, Python or Go, the missing case shows up at run
time, if at all. In Geng the compiler gives you a list of the places to
change.

### Making impossible states impossible

Custom types let you rule out combinations of data that make no sense. Here
is a common way to describe a download in Python:

```python
@dataclass
class Download:
    loading: bool
    data: bytes | None
    error: str | None
```

This allows eight combinations, and most of them are nonsense: loading with an
error, finished with both data and an error, or not loading with neither.
Every function that reads a `Download` has to decide what to do with them.

In Geng, list the states that can happen:

```geng
type Download
    = NotStarted
    | Loading
    | Failed String
    | Done Bytes
```

Now there are four states and each one carries only the data it has. No code
has to handle "loading with an error", because that value cannot exist.

When you design a type, ask which values it allows that should not exist, and
change the type until there are none.

### Type parameters

A custom type can take a type as a parameter, like generics in Go or Java, or
templates in C++. Two of the most used types in Geng are defined this way:

```geng
type Maybe a
    = Just a
    | Nothing

type Result error value
    = Ok value
    | Err error
```

`Maybe a` replaces `null`, `None` and `nil`. A `Maybe String` might hold a
string; a `String` always does. Because they are different types, you cannot
forget to check: the compiler makes you handle `Nothing` before you can use the
string.

`Result error value` replaces exceptions. A function that can fail returns a
`Result`, and the caller must handle the `Err` case.
[Error handling](@/docs/error-handling.md) covers both in detail.

### Recursive types

A custom type can refer to itself, which is how trees and similar structures
are built:

```geng
type Tree a
    = Leaf
    | Node { left : Tree a, value : a, right : Tree a }
```

There are no pointers and no null children. An empty subtree is `Leaf`, and
pattern matching makes you handle it.

## Opaque types: rules the compiler cannot see

Some rules are about values, not shapes. A percentage is an `Int` from 0 to
100. An email address is a `String` with an `@` in it. The types `Int` and
`String` allow far more than that.

Today, Geng enforces rules like these with an opaque type. The module wraps
the value in a custom type and does not expose the constructor:

```geng
module Percent exposing (Percent, fromInt, toInt)


type Percent
    = Percent Int


fromInt : Int -> Maybe Percent
fromInt n =
    if n >= 0 && n <= 100 then
        Just (Percent n)

    else
        Nothing


toInt : Percent -> Int
toInt (Percent n) =
    n
```

Other modules can only get a `Percent` from `fromInt`, so every `Percent` in
the program is between 0 and 100. Functions that take a `Percent` do not need
to check again. This is sometimes summed up as "parse, don't validate": check
once at the edge, and let the type carry the result.

This works, but it has costs:

- The check happens at run time, and the caller has to handle `Nothing` even
  when it is obvious that `50` is a valid percentage.
- The value is wrapped. You call `Percent.toInt` to do arithmetic with it, and
  the module has to provide every operation it wants to allow.
- The compiler does not know what the rule is. It knows that a `Percent` came
  from `fromInt`, not that it is at most 100.

Refinement types are meant to remove these costs for rules the compiler can
understand.

## Refinement types (planned)

> [!NOTE]
> Refinement types are designed but not yet checked. `geng fmt` already reads
> and formats the syntax below. `geng make` refuses any refined type with
> `REFINEMENTS NOT CHECKED YET`, so that nothing looks checked when it is
> not. Details may still change before the checker is built.

### What they are

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
Once checked, the condition is erased. A refined program runs exactly the same
code as an unrefined one, with no check added at run time.

### Writing one

Refinements go in a function's type signature. A condition may name any
argument of the same signature, before or after it:

```geng
-- The index must be inside the array that comes after it.
getAt : (i : Int where 0 <= i && i < Array.length a) -> (a : Array x) -> x

-- The result promises something about itself and the arguments.
pushLast : x -> (a : Array x) -> (r : Array x where Array.length r == Array.length a + 1)

-- A callback's argument: `initialize` promises its function an index in range.
initialize : (n : Int where 0 <= n) -> ((i : Int where 0 <= i && i < n) -> x) -> Array x

-- An array that is never empty, so its first element always exists.
firstOf : (a : Array x where Array.length a > 0) -> x
```

`getAt` needs no `Maybe` and no bounds check, because a call that might read
outside the array does not compile.

A condition is deliberately small. It can contain:

- integer literals, and the names of the signature's arguments (or the
  result's own name, in the result's condition);
- a few *measures* applied to an argument, such as `Array.length`,
  `Bytes.length`, `String.count`, `Dict.count` and `Set.count`;
- `+`, `-`, and `*` with a literal on one side;
- `<`, `<=`, `>`, `>=`, `==` and `/=`;
- `&&`, `||`, `not`, `True`, `False` and parentheses.

Nothing else: no calls to your own functions, no records and no floats. A
condition outside this list is a `BAD REFINEMENT` error. The arithmetic in a
condition is exact, because it is never run: `at + 4 <= Bytes.length b` means
the same as `at <= Bytes.length b - 4`.

In the first design, refinements go only on the arguments and result of a
top-level or `let` signature, including callbacks. Type aliases, record
fields and annotations on expressions cannot be refined yet.

### Things that are almost refinement types

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

### How the compiler knows

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

- the tests on the way to the call, such as the `if` above (`when` branches
  work the same way);
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
                go (i + 1) (acc + getAt i a)

            else
                acc
    in
    go 0 0
```

`getAt i a` needs `0 <= i` and `i < Array.length a`. The `if` gives the
second. The first comes from the loop: `go` starts at 0 and only adds 1, and
the compiler finds that by itself. As a rule, you write refinements on the
functions a module exposes, which are its contract, and the compiler infers
them for private loops.

### Overflow is checked too

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

Geng's own code had exactly this bug in its byte decoder. Designing
refinement types is what found it.

### When a refinement does not hold

An error says what was required, points at the call, lists each fact the
compiler had and where it came from, and gives values that break the
condition. When there is a rewrite the compiler can prove means the same, it
suggests it. For example, for a loop that counts down:

```text
-- CANNOT SHOW A REFINEMENT ---------------------------------- src/Compare.geng

`getAt` reads inside the array it is given, and here I cannot show
`0 <= level`:

52|     Array.getAt level levels
                    ^^^^^
What I know here:

    level < Array.length levels   inferred for the loop `compareLevels`

and nothing about `level` from below: the loop calls itself with `level - 1`,
and with `Array.length levels = 0` and `level = -1` it reads before the
start.
```

### In a package

A refinement on an exposed function is part of its type, so changing one is a
semver change. The package tool works out how big a change it is: asking for
less of an argument, or promising more in a result, is a minor change; asking
for more, or promising less, is a major one. Rewriting a condition so it means
the same thing is a patch.

### What this could mean for Geng

Some places where Geng today either returns a `Maybe` or picks a safe default
are places where a refinement type could state the rule instead:

- **Array and byte indexes.** `Array.get` returns a `Maybe` because the index
  might be out of range. `getAt` above takes an index the compiler knows is in
  range, and returns the element directly.
- **Non-empty arrays.** `Array.first` returns a `Maybe`. For an array known to
  be non-empty, it would not need to.
- **Division.** Geng defines `x // 0` as `0` so that arithmetic never crashes.
  A function that requires a non-zero divisor can say so in its type.
- **Number ranges.** Converting an `Int` to a `UInt8` saturates today. A value
  known to be between 0 and 255 could convert without any loss.

A survey of the code Geng builds found about 350 `Nothing` and `Err` branches
written only because a read might fail. The first design removes 122 of them.
These are examples of what refinement types can express, not a list of
planned API changes.

### What they will not do

- **Conditions are limited.** Comparisons, arithmetic and sizes are all a
  condition can talk about. It cannot decide whether a string is a valid email
  address. Opaque types stay the right tool for rules like that, and for
  ranges you want to name once, since type aliases cannot be refined yet.
- **Run-time checks do not go away.** Data from a file, the network or the user
  is not known in advance. You check it once, with an `if` or a `when`, and
  from then on the compiler carries the result. That is the same "parse, don't
  validate" approach as today, with less wrapping.
- **The compiler may give up.** When a call needs more cases than the checker
  tries, it says so rather than guessing. A test just before the call, or a
  small function with the condition as a refinement, makes it decidable.
- **They are optional.** Code that never uses a refinement type is ordinary
  Geng.

## Summary

| You know | In Geng |
|---|---|
| `struct`, dataclass, plain class | record |
| class methods | functions that take the record |
| inheritance | extensible records, or a custom type with one case per kind |
| `enum` | custom type with no data |
| tagged union, `std::variant`, interface with a type switch | custom type with data |
| `switch`, `match` | `when ... is`, checked for missing cases |
| `null`, `None`, `nil` | `Maybe` |
| exceptions, `(value, err)` | `Result` |
| generics, templates | type parameters |
| validated wrapper class | opaque type |
| range types, assertions, validators | refinement types (planned) |

## Where to go next

- [Types](@/reference/types.md): the full rules, including numbers, classes
  and type inference.
- [Error handling](@/docs/error-handling.md): `Maybe` and `Result` in practice.
- [Functional programming for imperative programmers](@/docs/functional-programming.md):
  the rest of the mental model.
