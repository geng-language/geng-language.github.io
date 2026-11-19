+++
title = "Syntax"
description = "The grammar at a glance: what every form looks like, with links to what it means."
weight = 10
+++

What every form of Geng looks like, one example each. The pages linked from
each section say what the forms mean. Geng's syntax is Gren's, which is
close to Elm's.

## Comments

```geng
-- to the end of the line

{- a block,
   over lines -}

{-| documentation, before the declaration it describes -}
```

## Literals

```geng
42                  -- Int
-7
0xFF                -- hexadecimal
3.14                -- Float
1.5e3
'a'                 -- Char
"one\ttwo\n"        -- String
[ 1, 2, 3 ]         -- Array
{ x = 1.0, y = 2.0 } -- record
{}                  -- the empty record
```

A multi-line string starts and ends with `"""`. Each line starts at the
column of the opening quotes, and that indentation is not part of the
string:

```geng
help =
    """
    usage: tool FILE
    """
```

- **Escapes**: `\n`, `\r`, `\t`, `\"`, `\'`, `\\`, and `\u{1F600}`, four
  to six hex digits naming a codepoint that is not a surrogate.
- **Numbers**: no `_` separators, no binary or octal, and a float has a
  digit before its point (`0.5`, not `.5`).
- **Types**: an integer literal is an `Int` and one with a point or an
  exponent a `Float`, unless its context asks for another
  ([Types](@/reference/types.md#numbers)). `(42 : Int64)` gives one in
  place; a literal that does not fit its type is an error.

## Names

- Values, functions and fields start with a lowercase letter: `count`.
- Types, constructors and modules start with an uppercase letter: `Maybe`,
  `Just`, `Dict`. A module may be dotted, `Json.Decode`.
- A name from another module is qualified, `Dict.empty`, unless its import
  exposes it.
- Letters come from any script, by a fixed version of the Unicode database;
  module names are ASCII only.
- A name cannot shadow another in scope. [Keywords](@/reference/keywords.md)
  lists the reserved words.

## Layout

Indentation is significant. A top-level declaration starts in the first
column; everything that continues it is indented further; the branches of a
`when` and the definitions of a `let` line up with each other. `geng fmt`
writes the standard layout.

## Declarations

```geng
module Shape exposing (Shape(..), Sku, area)

import Dict exposing (Dict)
import Json.Decode as Decode
import Math


{-| A shape. -}
type Shape
    = Circle { radius : Float }
    | Rectangle { width : Float, height : Float }
    | Point


type alias Size =
    { width : Float, height : Float }


area : Shape -> Float
area shape =
    when shape is
        Circle { radius } ->
            Math.pi * radius * radius

        Rectangle { width, height } ->
            width * height

        Point ->
            0


@derive(Eq, Ord)
type Sku
    = Sku { code : UInt32 }
```

`exposing (Shape(..))` exposes a type with its constructors; without `(..)`,
as `Sku` here, it is abstract, and `@derive` asks for its `Eq` and `Ord`. A constructor takes at most one argument. An attribute
(`@derive`, `@extern`, `@measure`, ...) goes on the line before its
declaration ([Keywords](@/reference/keywords.md#attributes)).

## Expressions

```geng
f x y                       -- a call
\x y -> x + y               -- a function
if n < 0 then a else b      -- every `if` has an `else`
r.width                     -- a field
.width                      -- the function that reads it
{ r | width = 2, height = 3 } -- a record with fields replaced
(n : Int64)                 -- a type given in place
x |> f                      -- f x
f <| x                      -- f x
f >> g                      -- f, then g
g << f                      -- f, then g
(+)                         -- an operator as a function
```

```geng
when shape is
    Circle { radius } ->
        radius

    _ ->
        0

let
    double : Int -> Int
    double x =
        x * 2
in
double n
```

[Operators](@/reference/operators.md) has every operator and its
precedence; [Keywords](@/reference/keywords.md) has `if`, `when`, `let` and
`meant`.

## Patterns

In a `when`, a function's arguments, and a `let`'s left-hand side:

| Pattern | Matches |
|---|---|
| `_` | anything |
| `x` | anything, and names it |
| `0`, `'a'`, `"x"` | that literal |
| `Nothing`, `Just x`, `Just (Ok x)` | a constructor, and its argument |
| `{ x, y }` | a record, naming its fields |
| `{ x = 0, y }`, `{ rest = more }` | a field matched by a pattern, or named another way |
| `[]`, `[ x ]`, `[ x, y ]` | an array of exactly that length |
| `Just x as m` | the pattern, and the whole value as `m` |

A `when` must cover every value its subject can have.

## Types

```geng
Int
Maybe a                         -- a type with a type variable
Int -> String -> Bool           -- a function
{ x : Float, y : Float }        -- a record
{ r | x : Float }               -- any record with at least `x`
{}                              -- the empty record
Eq a => a -> Array a -> Bool    -- a constraint
```

A signature's argument or result may be named and refined, and an
argument may be `taken`:

```geng
(i : Int where 0 <= i && i < Array.length a)     -- a refined argument
(r : Array x where Array.length r > 0)           -- a refined result
Array (b : Int where 0 <= b && b < 256)          -- refined elements
Maybe (i : Int where 0 <= i)                     -- a refined payload
{ items : Array Item, index : Int where index < Array.length items } -- a field's condition
(taken b : Builder a)                            -- an argument the function uses up
```

[Types](@/reference/types.md), [Refinement
Types](@/reference/refinement-types.md) and [Affine
Types](@/reference/affine-types.md) say what each means and where it may go.
