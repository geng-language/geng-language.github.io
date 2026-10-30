+++
title = "Types"
description = "Built-in types, records, custom types, classes and type inference."
weight = 40
+++

## Numbers

Geng's numeric types have the same size and behavior on every platform.

| Type | Size | Notes |
|---|---|---|
| `Int` | 32-bit signed | the default for integer literals |
| `Int64` | 64-bit signed | |
| `UInt32` | 32-bit unsigned | |
| `UInt64` | 64-bit unsigned | |
| `Float` | 64-bit IEEE 754 | the default for literals with a decimal point |
| `Float32` | 32-bit IEEE 754 | |
| `Int8`, `UInt8`, `Int16`, `UInt16` | 8 and 16 bits | for binary formats and interop |

### Literals

`42` is an `Int` and `1.5` is a `Float` unless the context says otherwise. A
type signature or an expression annotation picks another type:

```geng
timeout : Int64
timeout =
    30000

mask =
    (0xFF : UInt32)
```

A literal that does not fit its type is a compile error.

### Arithmetic is total

No arithmetic operation crashes:

- **Integer overflow wraps**, for every integer type.
- **Integer division by zero is `0`**: `x // 0 == 0`, and `remainderBy 0 n == n`.
- **Shifts** by more than the width are clamped.
- **Lossy conversions saturate**: converting a value too large for the target
  type gives the largest value the target can hold.
- **Floats** are IEEE 754 on every platform, including infinity and NaN.

### Float equality and ordering

Float `==` and `compare` are defined so every value equals itself: `nan ==
nan` is `True`, `-0.0 == 0.0` is `True`, and NaN sorts after every other
float. That makes floats usable as `Dict` keys.

### Transcendental functions

`sin`, `cos`, `log`, `exp`, `pow` and the rest are written in Geng, as a port
of fdlibm, the library behind Java's `StrictMath`. They give bit-for-bit the
same result on every platform. They are slower than the platform's own math
library, which matters in tight numeric loops.

## Other built-in types

| Type | Notes |
|---|---|
| `Bool` | `True` or `False` |
| `Char` | a Unicode codepoint |
| `String` | a sequence of codepoints; see below |
| `Array a` | a flat, immutable array |
| `Maybe a` | `Just a` or `Nothing` |
| `Result e a` | `Ok a` or `Err e` |
| `Dict k v`, `Set a` | keys may be any `Ord` type |
| `Bytes` | binary data |
| `Task x a` | an effect that fails with `x` or succeeds with `a` |
| `{}` | the empty record, used where there is no meaningful value |
| `Never` | a type with no values; `Task Never a` cannot fail |

### Strings

A `String` never contains invalid Unicode. Its API works in codepoints:
`String.length` counts codepoints, and indexes are codepoint indexes. String
`compare` is codepoint order. Grapheme clusters, case mapping and whitespace
use tables from a fixed Unicode version (17.0.0), so they behave the same
everywhere. `Bytes.fromString` always produces UTF-8.

## Records

```geng
type alias Point =
    { x : Float, y : Float }

origin : Point
origin =
    { x = 0.0, y = 0.0 }

moveRight : Float -> Point -> Point
moveRight dx point =
    { point | x = point.x + dx }
```

`{ point | x = ... }` produces a new record; the original is unchanged. `.x`
on its own is a function that reads the field: `Array.map .x points`.

Extensible record types describe any record that has at least some fields:

```geng
getName : { r | name : String } -> String
getName record =
    record.name
```

## Custom types

```geng
type Shape
    = Circle { radius : Float }
    | Rectangle { width : Float, height : Float }
    | Point
```

Each constructor takes zero or one argument; use a record when you need
several fields. Custom types can take type parameters (`type Tree a = ...`)
and can be recursive.

If a module exposes a type without its constructors (`exposing (Shape)`
rather than `exposing (Shape(..))`), other modules cannot build or inspect
its values directly. This is how a module enforces rules about what a valid
value is.

## Type aliases

`type alias` names a type. It does not create a new one, so a `Point` and a
`{ x : Float, y : Float }` are interchangeable.

An alias can also name a [refinement](#refinement-types). `core`'s `Basics`
has `Nat`:

```geng
type alias Nat =
    (n : Int where 0 <= n)
```

A refined alias goes only where a signature names a value: `Array Nat`,
`Maybe Nat` and a record field of type `Nat` are errors.

## Function types

`a -> b -> c` is a function that takes an `a` and a `b` and returns a `c`.
Functions are curried: giving it only an `a` returns a function `b -> c`.

## Refinement types

An argument or a result of a signature can be named, `(a : Array x)`, and
refined with a condition after `where`, `(i : Int where 0 <= i)`. The
condition may name any argument of the same signature, before or after it:

```geng
getAt : (i : Int where 0 <= i && i < Array.length a) -> (a : Array x) -> x
```

- A refinement goes on the arguments and the result of a top-level or `let`
  signature, callbacks included, and as the whole body of a type alias. It
  cannot go on a record field, a variant's argument or an expression's
  annotation.
- A condition holds integer literals, the signature's names, the measures
  `Array.length`, `Bytes.length`, `String.count`, `Dict.count` and
  `Set.count`, `+`, `-`, `*` with a literal on one side, the six
  comparisons, and `&&`, `||`, `not`, `True` and `False`. A name read
  directly must be an `Int`.
- A condition's arithmetic is exact. The program's own `Int` arithmetic still
  wraps at 32 bits, and where it feeds a refinement the compiler must show
  that it does not wrap.
- A name that no condition reads is an error.
- The compiler proves each condition at compile time, from the tests on the
  way to a call, the caller's refinements, the results of functions it
  called, and invariants it infers for loops. Then the condition is erased:
  a refined type is its underlying type at run time.

See [Refinement types](@/docs/type-system.md#refinement-types) for a guided
introduction.

## Affine types

An affine value can be *used* only once. Today that means one type,
`Array.Builder`, and any type that holds one. Nothing else in Geng is affine:
`Array`, `String`, `Dict`, records of them and so on are shared freely.

**Why.** Geng values never change, but a builder is the exception: it writes
into its buffer in place, so building a large array is fast. That is only safe
if nobody is still holding the old builder when it is written to. Go has
exactly this problem with `append`:

```go
a := make([]int, 0, 10)
b := append(a, 1)
c := append(a, 2) // writes into the same buffer as b
fmt.Println(b[0]) // prints 2, not 1
```

In C++ terms, a builder is like a `std::unique_ptr` or a moved-from
`std::vector`: once you hand it on, the old name is dead. The difference is
that C++ lets you use a moved-from object anyway, and Geng refuses to compile
it.

### Looking and using

You may **look at** a builder as often as you like, until you **use** it.

| Looking (any number of times) | Using (once) |
|---|---|
| `Builder.length b` | writing to it: `Builder.pushLast x b` |
| `Builder.get i b` | finishing it: `Builder.toArray b` |
| `Builder.slice from to b` | storing it in a record or custom type |
| passing it to a function that only looks | returning it |
| | passing it to a function that marks it `moved` |

A write gives back a builder, and that new one is what you use next. Think of
each name as a ticket that can be spent once.

### Allowed

Each write goes to the builder the previous step gave back:

```geng
let
    b0 = Builder.empty 3
    b1 = Builder.pushLast 1 b0
    b2 = Builder.pushLast 4 b1
    b3 = Builder.pushLast 9 b2
in
Builder.toArray b3
```

Look first, then use:

```geng
let
    n = Builder.length b
    first = Builder.get 0 b
in
Builder.toArray (Builder.pushLast (n + Maybe.withDefault 0 first) b)
```

One use in each branch is fine, since only one branch runs:

```geng
if Builder.length b == 0 then
    Builder.toArray (Builder.pushLast 1 b)

else
    Builder.toArray b
```

When you do want two, copy first. `Builder.copy` only looks at `b0`, and makes
a second builder with its own buffer:

```geng
let
    left = Builder.pushLast 2 (Builder.copy b0)
    right = Builder.pushLast 3 b0
in
Array.append (Builder.toArray left) (Builder.toArray right)
```

### Refused

Writing twice to the same builder. This is the Go `append` bug above:

```geng
let
    left = Builder.pushLast 2 b0
    right = Builder.pushLast 3 b0 -- error: b0 was already used
in
...
```

Looking after using. `toArray` took `b`, so its length is gone:

```geng
let
    arr = Builder.toArray b
in
Array.pushLast (Builder.length b) arr -- error: b was already used
```

Storing it twice. Two copies of it would share one buffer:

```geng
{ first = b, second = b } -- error: b is used twice
```

Each of these is an `OWNERSHIP` error that points at both places and suggests
a fix. See [When it is refused](@/docs/type-system.md#when-it-is-refused) for
more.

### Functions: `moved`

A function argument that is a builder is only looked at, unless the signature
marks it `moved`. `moved` tells the caller "I use this up, so you cannot use it
again":

```geng
addOne : (moved b : Builder Int) -> Builder Int
addOne b =
    Builder.pushLast 1 b


countOf : Builder Int -> Int
countOf b =
    Builder.length b
```

A caller can call `countOf b` and still use `b` afterwards, but not after
`addOne b`. The signature must be honest: leaving out `moved` on `addOne` is an
error, and so is writing it on `countOf`. This is the opposite of Rust, where
an unmarked argument is moved and a borrow is marked with `&`. For C++, think
of a plain argument as `const Builder&` and a `moved` one as `Builder&&`.

- `moved` goes on the arguments of a top-level or `let` signature. It cannot
  go on the result, on a callback's argument, or on a type that is not
  affine, including a type variable.
- A function with no signature has its `moved` arguments worked out from its
  body.

### Your own types

A record with a builder field is affine with no mark. Using the builder field
takes only that field; the other fields can still be read:

```geng
let
    arr = Builder.toArray doc.body
in
Array.pushLast (String.count doc.title) arr -- fine: title was not used
```

A custom type that holds a builder must say so with `@affine`:

```geng
@affine
type Document
    = Document { title : String, body : Builder Line }
```

`@affine` cannot go on a type alias, and an `@affine` type cannot also have
`@derive(…)`.

### Fine print

- "Once" means once on each path through the code, as with the `if` above.
  A record's fields, and a `let`'s bindings, are evaluated in the order they
  are written.
- Some code reads a builder more than once without using it, and that is
  allowed: reading it inside the function that a fold calls, and reading an
  element of a builder of builders without taking it out.
- A function given to `Task.andThen`, `Task.onError` or `Task.map` may use a
  builder it captured, since a task runs that function at most once. Any
  other function that captures a builder uses it when the function is made.
- A generic function that keeps a value twice cannot be given a builder.
- You never write any of the points in this list. The compiler works them
  out, and `geng docs` lists them for each function.

See [Builders](@/docs/type-system.md#builders-values-used-once) for a guided
introduction.

## Classes and constraints

A constraint says that a type variable must support some operations:

```geng
largest : Ord a => Array a -> Maybe a
```

The classes available are:

| Class | Provides | Instances |
|---|---|---|
| `Eq` | `==`, `/=` | derived for most types |
| `Ord` | `compare`, `<`, `>`, `<=`, `>=` | derived for most types |
| `Inspect` | `inspect` | derived for most types |
| `Num` | `+`, `-`, `*`, `^`, `negate` | every numeric type |
| `Integral` | `//`, `remainderBy`, `modBy` | integer types |
| `Fractional` | `/`, `isNaN`, `isInfinite` | `Float`, `Float32` |
| `Bits` | `and`, `or`, `xor`, shifts | every integer type |
| `Appendable` | `++` | `String`, `Array a` |

`Eq`, `Ord` and `Inspect` are derived structurally for every type whose
constructors are visible. An abstract type asks for them with
`@derive(Eq, Ord, Inspect)`, or its module writes an instance when the
structural answer would be wrong, as `Dict` and `Set` do. Functions have no
`Eq` instance, so comparing two functions is a compile error.

Classes take exactly one type parameter. There are no higher-kinded types, so
there is no `Functor` or `Monad`.

## Type inference

The compiler infers every type, so annotations are optional. Annotating
top-level definitions is still good practice: it documents the code and
gives better error messages.

Two rules affect numeric code:

- **Defaulting.** A numeric type variable that nothing determines becomes
  `Int`, or `Float` if it needs `Fractional`. Any other unresolved constraint
  is an error.
- **Unannotated values are not polymorphic over numbers.** `zero = 0` takes
  the type of its first use. Write `zero : Num a => a` to keep it general.

There is no polymorphic recursion and there are no higher-rank types. This is
what lets the compiler generate a specialized copy of every polymorphic
function for each type it is used at, which is how the native backend lays
out records as C structs.
