+++
title = "Types"
description = "Built-in types, records, custom types, refinement types, builders (affine types), classes and type inference."
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

A refined alias goes only where a signature names a value, or in a field of
a record written there (`{ count : Nat } -> Int`): `Array Nat`, `Maybe Nat`
and a field of type `Nat` in another type alias are errors.

An alias over a record states a rule about its fields, on the fields or on
the whole record. These two are the same type:

```geng
type alias Move =
    { items : Array Item
    , index : Int where 0 <= index && index < Array.length items
    }

type alias Move =
    (m : { items : Array Item, index : Int } where 0 <= m.index && m.index < Array.length m.items)
```

## Function types

`a -> b -> c` is a function that takes an `a` and a `b` and returns a `c`.
Functions are curried: giving it only an `a` returns a function `b -> c`.

## Refinement types

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

### Where a refinement goes

- On the arguments and the result of a top-level or `let` signature,
  callbacks included. A `let` signature's condition may also name the
  arguments of the signatures around it.
- As the whole body of a type alias, including an alias over a record.
- On a record's field, after its type:
  `index : Int where index < Array.length items`. The condition reads the
  record's fields bare, and only them. A record with one is refined like a
  refined alias.
- A refined alias may be a field of a record written in a signature.

A refinement cannot go on a variant's argument, a field of another type
alias, an extensible record or an expression's annotation.

A function that refines its arguments cannot be used as a value, since the
compiler could not see its calls. Pass a lambda that calls it instead:
`Array.indexedMap (\i _ -> Array.getProven i b) a`.

### Conditions

- A condition holds integer literals, the signature's names and their
  fields (`m.index`, `o.move.index`), the measures `Array.length`,
  `Bytes.length`, `Bytes.Owned.length`, `String.count`, `Dict.count`,
  `Set.count` and `Array.Builder.length` of a name or a field, `+`, `-`, `*` with a literal on one side, the six comparisons, and
  `&&`, `||`, `not`, `True` and `False`. A name or field read directly must
  be an `Int`.
- Nothing else: no calls to other functions, no records built in a
  condition, and no floats. Anything else is a `BAD REFINEMENT` error.
- A condition's arithmetic is exact. The program's own `Int` arithmetic still
  wraps at 32 bits, and where it feeds a refinement the compiler must show
  that it does not wrap. When it may wrap, the error suggests a form that
  cannot, such as `n <= length - offset` for `offset + n <= length`.
- A name that no condition reads is an error.

### How a condition is proved

Where a refined type is required, the compiler uses:

- the tests on the way there, from an `if` or a `when` on a number. A test
  that joins a `Bool` and a comparison with `&&` keeps the comparison;
- the refinements on the enclosing function's arguments;
- the promises in the results of functions called on the way. A promise
  holds only where its call was made: after
  `if b then below n else need n`, nothing `below` promised is known in the
  `else` branch;
- invariants it infers for private loops. It does not infer them for
  functions that call each other, or for facts that relate three values;
  those need a refinement written on them.

It follows a value chosen by an `if`, whether a number, an array or a
record (field by field), and understands `min`, `max` and `clamp`.
`f <| x` and `x |> f` are calls like `f x`. A record literal is checked with
the values it was written with, and a record update with the fields it
changed and the rest of the record it started from.

The same expression written twice, such as a sum, is two separate values. To
test it once and use it, name it with `let`.

A condition the compiler cannot prove is a `CANNOT SHOW A REFINEMENT` error.
When a proof needs more cases than the checker tries, the error says so
rather than guessing.

### In a package {#refinements-in-a-package}

A refinement on an exposed function is part of its type. Asking less of an
argument, or promising more in a result, is a minor change; asking more, or
promising less, is a major one. Rewriting a condition so it means the same
is a patch. `geng package diff`, `bump` and `validate` work this out.

`geng make` in a package and `geng package validate` refuse a package whose
conditions are not proved. An application checks every package it uses on
every build.

## Affine types: `Array.Builder` {#affine-types}

An affine value can be *used* only once. Today that means one type,
`Array.Builder`, and any type that holds one. A builder writes into its
buffer in place, and this rule is what makes that safe. Nothing else in Geng
is affine: `Array`, `String`, `Dict`, records of them and so on are shared
freely. (Geng may still write those in place when it can prove nothing else
holds them, but that is an optimization, not a rule.)

See [Builders](@/docs/type-system.md#builders-values-used-once) for a guided
introduction.

### Looking and using

You may **look at** a builder as often as you like, until you **use** it.

| Looking (any number of times) | Using (once) |
|---|---|
| `Builder.length b` | writing to it: `Builder.pushLast x b` |
| `Builder.get i b` | finishing it: `Builder.toArray b` |
| `Builder.slice from to b` | storing it in a record or custom type |
| passing it to a function that only looks | returning it |
| | passing it to a function that marks it `moved` |

A write gives back a new builder, and the next use must be of that one.
`Builder.copy` only looks, and gives back a second builder with its own
buffer.

| Refused | Compiles |
|---|---|
| `Builder.pushLast 2 b0` and `Builder.pushLast 3 b0` | `Builder.pushLast 2 (Builder.copy b0)` and `Builder.pushLast 3 b0` |
| `Builder.length b` after `Builder.toArray b` | `Builder.length b` before `Builder.toArray b` |
| `{ first = b, second = b }` | `{ first = Builder.copy b, second = b }` |
| | one use of `b` in each branch of an `if` or a `when` |

Each refused case is an `OWNERSHIP` error that points at both places and
suggests a fix. See [When it is refused](@/docs/type-system.md#when-it-is-refused)
for common cases.

### Order matters

Looking is only allowed *before* using, so order decides whether code
compiles. Geng evaluates left to right and top to bottom:

| Where | Order | Refused | Compiles |
|---|---|---|---|
| `let` | bindings in the order written | `arr = Builder.toArray b`<br>`n = Builder.length b` | `n = Builder.length b`<br>`arr = Builder.toArray b` |
| record | fields in the order written | `{ first = b, second = Builder.copy b }` | `{ first = Builder.copy b, second = b }` |
| call | arguments left to right, then the call | `pair (Builder.toArray b) (Builder.length b)` | `Builder.pushLast (Builder.length b) b` |
| `\|>` | `x \|> f y` is `f y x`: `x` goes in last | | `b \|> Builder.pushLast (Builder.length b)` |

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
error, and so is writing it on `countOf`.

- `moved` goes on the arguments of a top-level or `let` signature. It cannot
  go on the result, on a callback's argument, or on a type that is not
  affine, including a type variable.
- A function with no signature has its `moved` arguments worked out from its
  body.
- Whether a function keeps a callback it is given, and whether its result
  borrows from an argument, are always worked out from its body.

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

### A builder's length {#a-builders-length}

`Builder.length` is a measure, so a condition may name it, and
`Builder.getProven` reads an element with no `Maybe`:

```geng
getProven : (i : Int where 0 <= i && i < Array.Builder.length builder) -> (builder : Builder a) -> a
```

Each function in `Array.Builder` promises the length of the builder it
returns:

| Function | Length of the result |
|---|---|
| `empty n` | 0 (`n` is only the room set aside) |
| `repeat n x` | `n`, or 0 when `n <= 0` |
| `fromArray a` | `Array.length a` |
| `pushLast x b` | one more than `b` |
| `append a b` | `b`'s plus `Array.length a` |
| `set`, `update`, `fill`, `copyWithin`, `moveWithin`, `rotate` | the same as `b` |
| `swap` | the same as `b`, in the result's `builder` field |
| `toArray b` | the array's `Array.length` is `b`'s |

Reading the length is looking, so it must come before the builder is used:
to state the length of what `toArray` returned, name `Builder.length b`
before the call.

### Fine print

- "Once" means once on each path through the code, as with the `if` above.
- Some code reads a builder more than once without using it, and that is
  allowed: reading it inside the function that a fold calls, and reading an
  element of a builder of builders without taking it out.
- A function given to `Task.andThen`, `Task.onError` or `Task.map` may use a
  builder it captured, since a task runs that function at most once. Any
  other function that captures a builder uses it when the function is made.
- A generic function that keeps a value twice cannot be given a builder.
- You never write any of the points in this list. The compiler works them
  out, and `geng docs` lists them for each function.
- The rules are the same on every target: a program refused on one is
  refused on all. On JavaScript and native targets a builder is a buffer
  written in place; on the BEAM it is a persistent tree.

### In a package {#builders-in-a-package}

What a function does with its arguments, whether written (`moved`,
`@affine`) or worked out by the compiler, is part of its interface.
`geng docs` lists it for each exposed function, and `geng package diff`
compares it between versions. Each fact limits what a caller may do, so
gaining one is a major change and losing one is minor.

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
