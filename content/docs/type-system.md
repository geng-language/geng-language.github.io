+++
title = "Geng's type system"
description = "Records, custom types, pattern matching, opaque types and builders, for people who know Python, Go, C or C++."
weight = 22
+++

Most of Geng's type system comes from Elm and Gren, and those languages got it
from the ML family: records, custom types and pattern matching. If you have
written Python, Go, C or C++, records will look familiar. Custom types will
look partly like enums, but they can do more. Geng also has
[refinement types](@/docs/intro-to-refinement-types.md), which none of those
languages have and which have a page of their own, and builders, which
are written in place but checked by the compiler so they stay safe.

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

## Builders: values used once

Geng's values never change, so adding to an `Array` gives you a new array.
That is simple and safe, but building a large array one element at a time
would copy it over and over. An `Array.Builder` avoids that: it writes into
one buffer in place, like a Go slice you `append` to or a C++ `std::vector`
you `push_back` to.

```geng
import Array.Builder as Builder exposing (Builder)

squares : Int -> Array Int
squares n =
    Array.range 0 (n - 1)
        |> Array.foldl (\i acc -> Builder.pushLast (i * i) acc) (Builder.empty n)
        |> Builder.toArray
```

### The catch

Writing in place is only safe if nobody is still holding the old builder.
Go shows what goes wrong when someone is:

```go
a := make([]int, 0, 10)
b := append(a, 1)
c := append(a, 2) // writes into the same buffer as b
fmt.Println(b[0]) // prints 2, not 1
```

`b` and `c` share a buffer, so writing `c` changed `b`. Go compiles this
without a word. Geng refuses to compile the same thing, by making a builder
**affine**: it can be *used* only once.

If you know C++, a builder is like a `std::unique_ptr`. Once you hand it on,
the old name is dead. The difference is that C++ still lets you touch a
moved-from object, and Geng does not.

### Looking and using

Think of a builder's name as a ticket you can spend once. Until you spend it,
you may *look* at the builder as often as you like.

- **Looking** is asking it something: `Builder.length b`, `Builder.get i b`,
  `Builder.slice from to b`, or passing it to a function that only looks.
- **Using** is spending it: writing to it (`Builder.pushLast x b`), finishing
  it (`Builder.toArray b`), storing it in a record, returning it, or passing it
  to a function that takes it.

A write gives back a builder, and that is the new ticket. So instead of
calling `v.push_back(x)` on the same `v` each time, you pass along what the
last write gave back:

```geng
let
    b0 = Builder.empty 3
    b1 = Builder.pushLast 1 b0
    b2 = Builder.pushLast 4 b1
    b3 = Builder.pushLast 9 b2
in
Builder.toArray b3
```

In practice you rarely name each step. A fold or a pipeline passes the builder
along for you, as `squares` does above, and most code already follows the
rule without trying.

Full rules: [Affine types](@/reference/affine-types.md) and
[Looking and using](@/reference/affine-types.md#looking-and-using).

### Compiles, and does not

| Refused | Compiles |
|---|---|
| `left = Builder.pushLast 2 b0`<br>`right = Builder.pushLast 3 b0`<br>Two writes to one builder: Go's `append` bug. | `left = Builder.pushLast 2 (Builder.copy b0)`<br>`right = Builder.pushLast 3 b0`<br>`Builder.copy` only looks at `b0`, and makes a second builder with its own buffer. |
| `arr = Builder.toArray b`<br>`n = Builder.length b`<br>`toArray` spent `b`, so it cannot be read after. | `n = Builder.length b`<br>`arr = Builder.toArray b`<br>Look first, then use. |
| `{ first = b, second = b }`<br>Two places would share one buffer. | `{ first = Builder.copy b, second = b }`<br>Copy first: see [Order matters](#order-matters). |

"Once" means once on each path the program can take, so one use in each
branch of an `if` or a `when` is fine:

```geng
if Builder.length b == 0 then
    Builder.toArray (Builder.pushLast 1 b)

else
    Builder.toArray b
```

Full rules: [Looking and using](@/reference/affine-types.md#looking-and-using).

### Order matters

You may look at a builder only *before* you use it, so the order your code
runs in decides whether it compiles. Geng's order is the one you read in:
left to right and top to bottom.

- **`let` bindings** run in the order they are written. Here `toArray` spends
  `b` on the first line, so the second line is refused, even though nothing
  else depends on it:

  ```geng
  let
      arr = Builder.toArray b
      n = Builder.length b -- refused: b was used on the line above
  in
  Array.pushLast n arr
  ```

  Swap the two lines and it compiles.

- **Record fields** are evaluated in the order they are written:
  `{ first = b, second = Builder.copy b }` is refused, and
  `{ first = Builder.copy b, second = b }` compiles.

- **Function arguments** are evaluated left to right. With
  `pair : Array Int -> Int -> Array Int`,

  ```geng
  pair (Builder.toArray b) (Builder.length b) -- refused
  ```

  is refused, because the first argument spends `b` before the second reads
  it. An argument is always computed before the call it goes to, so
  `Builder.pushLast (Builder.length b) b` is fine: the length is read, then
  the push spends `b`.

- **A pipeline** `x |> f y` is `f y x`, so the value on the left goes in
  *last*, even though you read it first. This compiles, because the length is
  read before the push:

  ```geng
  b
      |> Builder.pushLast (Builder.length b)
      |> Builder.toArray
  ```

When the compiler refuses code for its order, the error says "read again
after that" and points at both places. The fix is almost always to move the
look earlier, or to look at what the use gave back instead.

Full rules: [Order matters](@/reference/affine-types.md#order-matters).

### What you write

Almost nothing. There are two marks, and both are about saying what a
function or a type does with a builder.

- **`taken`** marks a function argument that the function uses up. Without
  it, the function may only look at the argument, and the caller can keep
  using the builder afterwards. With it, the caller cannot:

  ```geng
  addLines : Array Line -> (taken b : Builder Line) -> Builder Line
  addLines lines b =
      Array.foldl Builder.pushLast b lines


  lineCount : Builder Line -> Int
  lineCount b =
      Builder.length b
  ```

  In C++ terms, a plain argument is a `const Builder&` and a `taken` one is a
  `Builder&&`. The signature has to be honest: leaving `taken` off `addLines`
  is an error, and so is putting it on `lineCount`. A function without a
  signature has this worked out from its body. If you know Rust, note that
  this is the opposite way round: there an unmarked argument is moved, and a
  borrow is marked with `&`.

- **`@affine`** goes on a custom type that holds a builder, so readers can see
  that its values are used once too. A record that holds one needs no mark:

  ```geng
  @affine
  type Document
      = Document { title : String, body : Builder Line }
  ```

Everything else, such as `Array`, `Dict`, `String` and records of them, is
shared freely as before. Geng still writes those in place when it can prove
no one else holds them, but that is an optimization, not a rule you have to
follow.

Full rules: [Functions: `taken`](@/reference/affine-types.md#functions-taken) and
[Your own types](@/reference/affine-types.md#your-own-types).

### What the compiler works out for itself

Some code looks like it uses a builder many times, but only looks at it. That
is allowed, and nothing about it is written. Here the fold's function runs
once per index, but each time it only reads `b`:

```geng
total : Array Int -> Builder Int -> Int
total indexes b =
    Array.foldl (\i sum -> sum + Maybe.withDefault 0 (Builder.get i b)) 0 indexes
```

The same goes for reading an element of a builder of builders without taking
it out, and for a function given to `Task.andThen` that uses a builder it
captured, since a task runs that function at most once. Whether a function
keeps the callback you give it, and whether its result is a borrow of an
argument, are worked out from its body too.

Full rules: [Fine print](@/reference/affine-types.md#fine-print).

### When it is refused

A refused program gets an `OWNERSHIP` error that points at both places and
says what to write instead. This one is easy to write by accident:

```geng
step : Maybe Int -> (taken acc : Builder Int) -> Builder Int
step m acc =
    m
        |> Maybe.map (\x -> Builder.pushLast x acc)
        |> Maybe.withDefault (Builder.pushLast 0 acc)
```

It reads as "push `x`, or else push 0", but Geng computes
`withDefault`'s argument before calling it, so both pushes happen, to the same
builder:

```text
-- OWNERSHIP ----------------------------------------------------- src/Main.geng

`acc` is a Builder, which is written in place, so it can be used once. Here a
function captures it, and it is used again:

4|         |> Maybe.map (\x -> Builder.pushLast x acc)
                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

5|         |> Maybe.withDefault (Builder.pushLast 0 acc)
                                                    ^^^

A function takes what it captures when it is made, whether or not it is ever
called.

Hint: hand it to the function as an argument where the function is called.
```

A `when` runs only one branch, so it has one push on each path, and compiles:

```geng
step : Maybe Int -> (taken acc : Builder Int) -> Builder Int
step m acc =
    when m is
        Just x ->
            Builder.pushLast x acc

        Nothing ->
            Builder.pushLast 0 acc
```

Other common fixes:

| Refused | Instead |
|---|---|
| a `let` that writes the builder, used in only one branch of an `if` | move the `let` into that branch |
| one builder defined at the top level and used in many places | make a new one at each use |
| `Array.sortWith` or `Array.remove` on an `Array (Builder Int)` | keep the rows in a `Builder`, and move them with `Builder.rotate` |
| `get` a row of a builder of builders, change it, and `set` it back | `Builder.update`, which hands the row to your function and puts back what it returns |

Full rules: [Affine types](@/reference/affine-types.md).

### Reading a builder without a `Maybe`

`Builder.get` returns a `Maybe`, as `Array.get` does. Where the compiler can
prove the index is in range, `Builder.getProven` returns the element itself,
and `Builder.length` can be named in a [refinement](@/docs/intro-to-refinement-types.md):

```geng
tested : Int -> Builder Int -> Int
tested i b =
    if i >= Builder.length b || i < 0 then
        -1

    else
        Builder.getProven i b
```

`Builder`'s functions promise what they do to the length, so a read after
them needs no test: `empty` starts at 0, `pushLast` adds one, `append` adds
the array's length, `set`, `update` and `rotate` keep it, and the array
`toArray` returns is as long as the builder was.

```geng
pushed : Int
pushed =
    let
        b =
            Builder.empty 4
                |> Builder.pushLast 10
                |> Builder.pushLast 20
    in
    Builder.getProven 1 b
```

A record can state a rule about its builder field, like any other field:

```geng
type alias Screen =
    { cells : Builder Int
    , cols : Int where 1 <= cols && cols <= Builder.length cells
    }


lastCol : Screen -> Int
lastCol s =
    Builder.getProven (s.cols - 1) s.cells
```

Looking at the length is looking, not using, so it follows the same order
rules as any other look: read it before the builder is used.

Full rules: [A builder's length](@/reference/affine-types.md#a-builders-length).

### In a package

What a function does with its arguments is part of its interface, whether
you wrote it (`taken`, `@affine`) or the compiler worked it out. `geng docs`
lists these facts for each exposed function, and `geng package diff`
compares them between versions. Each one limits what a caller may do, so
gaining one is a major change and losing one is minor.

Full rules: [In a package](@/reference/affine-types.md#builders-in-a-package).

### What it costs

Nothing for code that does not use builders. For code that does, the check
is what makes them reliably fast. Without it, a builder that was
accidentally kept around would still give the right answer, but would
quietly copy itself on every write. With it, that code does not compile.
And since every program is checked, a write can be a plain store into the
buffer, with no bookkeeping: up to twice as fast on Geng's own terminal
renderer. (Today builders still keep that bookkeeping; dropping it is
next.)

On JavaScript, and natively, a builder is a buffer written in place. On the
BEAM it is a persistent tree, as before. The rules are the same everywhere:
a program refused on one target is refused on all of them.

Full rules: [Fine print](@/reference/affine-types.md#fine-print).

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
- [Types](@/reference/types.md): the full rules, including numbers, classes
  and type inference; [Refinement Types](@/reference/refinement-types.md)
  and [Affine Types](@/reference/affine-types.md) for refinements and
  builders.
- [Error handling](@/docs/error-handling.md): `Maybe` and `Result` in practice.
