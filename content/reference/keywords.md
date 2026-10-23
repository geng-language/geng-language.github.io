+++
title = "Keywords"
description = "Every reserved word and attribute in Geng, with examples."
weight = 20
+++

## Summary

| Keyword | Used for |
|---|---|
| `module`, `exposing` | declaring a module and what it makes public |
| `import`, `as`, `exposing` | using other modules |
| `type` | declaring a custom type |
| `type alias` | naming a type |
| `if`, `then`, `else` | conditional expressions |
| `when`, `is` | pattern matching |
| `let`, `in` | local definitions |
| `as` (in patterns) | naming a whole value while destructuring it |
| `class`, `instance`, `where` | declaring classes and instances (first-party packages only, for now) |
| `where` (in a type) | a [refinement](@/docs/type-system.md#refinement-types): `(n : Int where n > 0)` |

Attributes start with `@` and come before a declaration:

| Attribute | Used for |
|---|---|
| `@derive(...)` | asking for `Eq`, `Ord` or `Inspect` on an abstract type |
| `@extern(...)` | declaring a function implemented in the host language |
| `@externPure(...)` | the same, for a function with no effects |
| `@capability` | marking a value only the application may use |

## Modules

`module` names the file's module and lists what it exposes:

```geng
module Inventory exposing (Sku, Item, restock, summary)
```

To expose a custom type together with its constructors, write `Type(..)`.
Without `(..)`, the type is abstract: other modules can use it, but cannot
build or match on its values directly.

```geng
module Shape exposing (Shape(..), area)
```

`import` brings in another module. `as` gives it a shorter name, and
`exposing` lets you use some of its names unqualified:

```geng
import Dict exposing (Dict)
import Beam.Server as Server
import Task exposing (Task)
```

Geng forbids `exposing (..)` on a whole module, in both `module` and `import`
lines, so every name used unqualified is listed where it is imported. (This
rule is decided but the compiler does not enforce it yet.)

## Types

`type` declares a custom type. Each constructor takes at most one argument,
which is often a record:

```geng
type Shape
    = Circle { radius : Float }
    | Rectangle { width : Float, height : Float }
    | Point
```

`type alias` gives an existing type a name. It does not create a new type:
an `Item` below is interchangeable with any record of the same shape.

```geng
type alias Item =
    { name : String
    , priceCents : Int64
    }
```

## Conditionals

`if` is an expression, and every `if` has an `else`:

```geng
sign : Int -> String
sign n =
    if n < 0 then
        "negative"

    else if n == 0 then
        "zero"

    else
        "positive"
```

## Pattern matching

`when ... is` matches a value against patterns. The compiler checks that every
possible value is covered.

```geng
area : Shape -> Float
area shape =
    when shape is
        Circle { radius } ->
            Math.pi * radius * radius

        Rectangle { width, height } ->
            width * height

        Point ->
            0.0
```

Patterns can be literals, constructors, records, arrays and `_`, which matches
anything:

```geng
describe : Array String -> String
describe args =
    when args is
        [] ->
            "no arguments"

        [ one ] ->
            "one argument: " ++ one

        _ ->
            "several arguments"
```

`as` names a whole value while also destructuring it:

```geng
when shape is
    (Rectangle { width }) as rect ->
        ...
```

Elm uses `case ... of`. Gren and Geng use `when ... is`.

## Local definitions

`let ... in` defines names that are visible only in the expression after `in`.
Definitions can have type annotations:

```geng
summary : Array Item -> String
summary items =
    let
        cents : Int64
        cents =
            total (Array.map .priceCents items)
    in
    "value " ++ inspect cents
```

A name cannot shadow another name that is already in scope.

## Classes and instances

`class` declares a set of operations, and `instance` implements them for a
type. A constraint arrow `=>` says that a function needs an instance:

```geng
class Eq a where
    eq : a -> a -> Bool

member : Eq a => a -> Array a -> Bool
```

For now, only Geng's own first-party packages can declare classes and
instances; opening both to everyone is planned. Application code uses the
classes `core` provides: `Eq`, `Ord`, `Inspect`, `Num`, `Integral`,
`Fractional`, `Bits` and `Appendable`. `class` and `instance` are contextual
keywords, so they can still be used as ordinary names elsewhere.

## Attributes

`@derive` asks the compiler for structural `Eq`, `Ord` or `Inspect` on an
abstract type. A type whose constructors are exposed gets them automatically.

```geng
@derive(Eq, Ord, Inspect)
type Sku
    = Sku { vendor : String, code : UInt32 }
```

`@extern` declares a function whose implementation is in the host language,
with one line per language. `@externPure` is the same for a function with no
effects. See [Interop](@/docs/interop.md).

```geng
@extern(js, "Time", "now")
nowWith : (Int64 -> posix) -> Task x posix
```

`@capability` marks a value, usually a permission, that only the application
may use. A dependency that tries to use it gets a compile error, so a library
can only touch the file system or the network if the application hands it a
permission.

## Words that are no longer keywords

- `case` and `of` are Elm's; Geng uses `when` and `is`.
- `port` is an ordinary name. `port module` and `effect module` are refused,
  with a message pointing to `@extern` and `Source`.
- `infix` is not available outside `core`. See [Operators](@/reference/operators.md).

New keywords are only added in major versions, with a tool to migrate code
that used the word as a name.
