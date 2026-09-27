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

## Function types

`a -> b -> c` is a function that takes an `a` and a `b` and returns a `c`.
Functions are curried: giving it only an `a` returns a function `b -> c`.

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
