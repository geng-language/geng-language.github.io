+++
title = "Geng's type system"
description = "Custom types, pattern matching, refinement types and builders, for people who know Python, Go, C or C++."
weight = 22
+++

Most of Geng's type system comes from Elm and Gren, and those languages got it
from the ML family: records, custom types and pattern matching. If you have
written Python, Go, C or C++, records will look familiar. Custom types will
look partly like enums, but they can do more. Geng also has
refinement types, which none of those languages have, and builders, which
are written in place but checked by the compiler so they stay safe.

This page explains the ideas. For the exact rules, see
[Types](@/reference/types.md).

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

Refinement types remove these costs for rules the compiler can understand.

## Refinement types

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
`geng make` refuses a program where it cannot prove one. Once checked, the
condition is erased. A refined program runs exactly the same code as an
unrefined one, with no check added at run time, on every target.

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
outside the array does not compile. A name that no condition reads is an
error, since it would look as if it meant something.

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

Refinements go on the arguments and result of a top-level or `let` signature,
including callbacks. A `let`'s condition may also name the arguments of the
signatures around it. Record fields and annotations on expressions cannot be
refined yet.

### Naming a refinement

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
getAt : (i : Nat) -> (a : Array x where i < Array.length a) -> x
```

Unlike the opaque `Percent` above, this one is a plain `Int`. `setVolume 50`
compiles with no `Maybe`, and `setVolume 150` does not compile at all.

A refined alias goes only where a signature names a value. `Array Nat`,
`Maybe Percent` and a record field of type `Nat` are errors, because nothing
there is checked. An alias's condition reads only its own value, so `Nat`
cannot say "an index into this array": write that one out, as `getAt` does.

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

- the tests on the way to the call, such as the `if` above (a `when` on a
  number works the same way);
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
                go (i + 1) (acc + Array.getAt i a)

            else
                acc
    in
    go 0 0
```

`Array.getAt i a` needs `0 <= i` and `i < Array.length a`. The `if` gives the
second. The first comes from the loop: `go` starts at 0 and only adds 1, and
the compiler finds that by itself. As a rule, you write refinements on the
functions a module exposes, which are their contract, and the compiler infers
them for private loops. It does not infer them for functions that call each
other, or a loop whose facts relate three values; those need a refinement
written on them.

The compiler follows a value chosen by an `if`, and understands `min`, `max`
and `clamp`. It treats the same sum written twice as two values, so test a
sum once, give it a name with `let`, and use the name.

### Refinements in `core`

`core` uses refinements for its own reads, and has two functions that take a
refined index and return the element directly:

```geng
Array.getAt : (i : Int where 0 <= i && i < Array.length array) -> (array : Array a) -> a
Bytes.getAt : (index : Int where 0 <= index && index < Bytes.length bytes) -> (bytes : Bytes) -> UInt8
```

Use them where a test you write anyway keeps the index in range, as a loop
does. Elsewhere, `Array.get` and `Bytes.get` still return `Maybe`. Note that
`Array.getAt` does not count a negative index from the end, as `Array.get`
does.

Other functions in `Array` promise something about what they return, so a
read after them can be proved without any test:

- `map`, `indexedMap`, `reverse` and `set` return an array of the same length;
- `pushFirst` and `pushLast` return one that is one longer;
- `prepend` and `append` return one as long as both together;
- `indexedMap`, `indexedFoldl` and `indexedFoldr` give their function an index
  inside the array.

So this compiles, with no `Maybe` and no test:

```geng
mirrored : Array Int -> Array Int
mirrored a =
    let
        back =
            Array.reverse a
    in
    Array.indexedMap (\i x -> x * 100 + Array.getAt i back) a
```

A function that promises something asks nothing of its caller, so a program
that calls only `Array.map` and the like builds exactly as it would without
refinements. A function that asks something of its arguments, as `getAt`
does, cannot be passed around as a value, because the compiler could not see
its calls. Pass a lambda that calls it instead:
`Array.indexedMap (\i _ -> Array.getAt i b) a`.

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

Geng's own code had exactly this bug in its byte decoder, and four sums in
`core`'s sort wrapped past 2^30 elements. Refinement types found all of them,
and they are fixed.

### When a refinement does not hold

An error says what was required, points at the call, lists each fact the
compiler had and where it came from, and gives values that break the
condition. When there is a rewrite the compiler can prove means the same, it
suggests it. This one reads the previous element of an array, but forgets to
check that there is one:

```geng
changes : Array Float -> Array Float
changes temps =
    Array.indexedMap (\i t -> t - Array.getAt (i - 1) temps) temps
```

```text
-- CANNOT SHOW A REFINEMENT -------------------------------------- src/Main.geng

`getAt` requires this of its argument `i`:

    0 <= i && i < Array.length array

and I cannot show it here:

8|     Array.indexedMap (\i t -> t - Array.getAt (i - 1) temps) temps
                                     ^^^^^^^^^^^^^^^^^^^^^^^^^

Here, with this call's values for its names, that is:

    0 <= i - 1 && i - 1 < Array.length temps

What I know here:

    0 <= i && i < Array.length temps
        from the refinement on `indexedMap`'s callback's `i`

With `Array.length temps = 1` and `i = 0`, it is False.
```

Testing `i > 0` before the read makes it compile.

### In a package

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

### What it costs

Nothing at run time. At compile time, a program with no refinements, or one
that only calls functions like `Array.map` that promise something, builds
exactly as before. One that calls `getAt` or writes its own refinements takes
a little longer: a small program calling `Array.getAt` builds warm in 0.30
to 0.33 seconds, against 0.26 to 0.29 for one calling `Array.map`.

### What they will not do

- **Conditions are limited.** Comparisons, arithmetic and sizes are all a
  condition can talk about. It cannot decide whether a string is a valid email
  address, or whether a key is in a `Dict`. Opaque types stay the right tool
  for rules like that.
- **Run-time checks do not go away.** Data from a file, the network or the user
  is not known in advance. You check it once, with an `if` or a `when`, and
  from then on the compiler carries the result. That is the same "parse, don't
  validate" approach as today, with less wrapping.
- **A `Maybe` hides what was checked.** The value inside a `Maybe` cannot be
  refined, so a helper that returns `Maybe Int` loses what it tested.
- **The compiler may give up.** When a call needs more cases than the checker
  tries, it says so rather than guessing. A test just before the call, or a
  small function with the condition as a refinement, makes it decidable.
- **They are optional.** Code that never uses a refinement type is ordinary
  Geng.

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

### What you write

Almost nothing. There are two marks, and both are about saying what a
function or a type does with a builder.

- **`moved`** marks a function argument that the function uses up. Without
  it, the function may only look at the argument, and the caller can keep
  using the builder afterwards. With it, the caller cannot:

  ```geng
  addLines : Array Line -> (moved b : Builder Line) -> Builder Line
  addLines lines b =
      Array.foldl Builder.pushLast b lines


  lineCount : Builder Line -> Int
  lineCount b =
      Builder.length b
  ```

  In C++ terms, a plain argument is a `const Builder&` and a `moved` one is a
  `Builder&&`. The signature has to be honest: leaving `moved` off `addLines`
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

### When it is refused

A refused program gets an `OWNERSHIP` error that points at both places and
says what to write instead. This one is easy to write by accident:

```geng
step : Maybe Int -> (moved acc : Builder Int) -> Builder Int
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
step : Maybe Int -> (moved acc : Builder Int) -> Builder Int
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

The full rules are in [Affine types](@/reference/types.md#affine-types).

### In a package

What a function does with its arguments is part of its interface, whether
you wrote it (`moved`, `@affine`) or the compiler worked it out. `geng docs`
lists these facts for each exposed function, and `geng package diff`
compares them between versions. Each one limits what a caller may do, so
gaining one is a major change and losing one is minor.

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

- [Types](@/reference/types.md): the full rules, including numbers, classes
  and type inference.
- [Error handling](@/docs/error-handling.md): `Maybe` and `Result` in practice.
- [Functional programming for imperative programmers](@/docs/functional-programming.md):
  the rest of the mental model.
