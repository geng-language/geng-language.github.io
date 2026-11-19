+++
title = "Geng's type system"
description = "Records, custom types, pattern matching and opaque types, for people who know Python, Go, C or C++."
weight = 22
+++

Most of Geng's type system comes from Elm and Gren, and those languages got it
from the ML family: records, custom types and pattern matching. If you have
written Python, Go, C or C++, records will look familiar. Custom types will
look partly like enums, but they can do more. Geng also has
[refinement types](@/docs/intro-to-refinement-types.md), which none of those
languages have, and [builders](@/docs/builders.md), which are written in
place but checked by the compiler so they stay safe; each has a page of its
own.

This page explains the ideas. For the exact rules, see
[Types](@/reference/types.md), [Refinement Types](@/reference/refinement-types.md)
and [Affine Types](@/reference/affine-types.md).

## A type is a set of values

Read a type as the set of values it allows. `Bool` has two values. `UInt8` has
256. `String` has infinitely many.

Every type in this page is a way to build a new set from old ones:

- A **record** combines values: a `Point` is an `x` *and* a `y`.
- A **custom type** chooses between values: a `Shape` is a circle *or* a
  rectangle.
- A **refinement type** keeps only some values of another type: an
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

Full rules: [Records](@/reference/types.md#records).

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

Full rules: [Pattern matching](@/reference/keywords.md#pattern-matching).

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

Full rules: [Custom types](@/reference/types.md#custom-types).

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

[Refinement types](@/docs/intro-to-refinement-types.md) remove these costs for rules the compiler can understand.

Full rules: [Custom types](@/reference/types.md#custom-types), on exposing a
type without its constructors.

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
| range types, assertions, validators | refinement types |
| a slice or vector you append to, `StringBuilder` | `Array.Builder`, used once |

## Where to go next

- [Intro to refinement types](@/docs/intro-to-refinement-types.md): the
  conditions on values the compiler proves.
- [Builders](@/docs/builders.md): arrays written in place, used once.
- [Types](@/reference/types.md): the full rules, including numbers, classes
  and type inference; [Refinement Types](@/reference/refinement-types.md)
  and [Affine Types](@/reference/affine-types.md) for refinements and
  builders.
- [Error handling](@/docs/error-handling.md): `Maybe` and `Result` in practice.
