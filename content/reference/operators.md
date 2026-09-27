+++
title = "Operators"
description = "Every operator in Geng, with precedence, associativity and examples."
weight = 30
+++

Geng has a fixed set of operators. The compiler knows each one's precedence and
associativity, and each is shorthand for a named function. Programs and
packages cannot define new operators.

## Precedence and associativity

Higher numbers bind tighter. Function application (`f x`) binds tighter than
any operator.

| Precedence | Operators | Associativity |
|---|---|---|
| 9 | `<<` | left |
| 9 | `>>` | right |
| 8 | `^` | right |
| 7 | `*` `/` `//` | left |
| 6 | `+` `-` | left |
| 5 | `++` | right |
| 4 | `==` `/=` `<` `>` `<=` `>=` | none |
| 3 | `&&` | right |
| 2 | `\|\|` | right |
| 0 | `\|>` | left |
| 0 | `<\|` | right |

How to read the table:

```geng
1 + 2 * 3           -- 7: * (7) binds tighter than + (6)
2 ^ 3 ^ 2           -- 512: ^ is right-associative, so 2 ^ (3 ^ 2)
10 - 4 - 3          -- 3: - is left-associative, so (10 - 4) - 3
a < b && b < c      -- comparisons (4) bind tighter than && (3)
```

The comparison operators have no associativity, so they cannot be chained.
`a < b < c` is a compile error; write `a < b && b < c`.

`|>` and `<|` share precedence 0 but associate in opposite directions, so
they cannot be mixed in one expression without parentheses.

## Pipelines

| Operator | Type | Stands for |
|---|---|---|
| `\|>` | `a -> (a -> b) -> b` | `Basics.apR` |
| `<\|` | `(a -> b) -> a -> b` | `Basics.apL` |

`x |> f` is `f x`. A pipeline reads top to bottom, in the order the steps run:

```geng
lowItems threshold stock =
    stock
        |> Dict.keepIf (\_ count -> count < threshold)
        |> Dict.keys
```

`f <| x` is also `f x`. It is mostly used to avoid parentheses around a last
argument that is long:

```geng
Console.write <| "Total: " ++ String.fromInt total ++ "\n"
-- same as
Console.write ("Total: " ++ String.fromInt total ++ "\n")
```

Pipelines work because functions take the value they operate on as their last
argument: `Dict.keys dict`, `Array.map f array`, `Task.andThen f task`.

## Composition

| Operator | Type | Stands for |
|---|---|---|
| `>>` | `(a -> b) -> (b -> c) -> (a -> c)` | `Basics.composeR` |
| `<<` | `(b -> c) -> (a -> b) -> (a -> c)` | `Basics.composeL` |

`f >> g` is a function that applies `f` and then `g`. `g << f` is the same
function, written in the other direction.

```geng
lengthOfTrimmed : String -> Int
lengthOfTrimmed =
    String.trim >> String.length
```

## Arithmetic

| Operator | Type | Stands for |
|---|---|---|
| `+` | `Num a => a -> a -> a` | `Num.add` |
| `-` | `Num a => a -> a -> a` | `Num.sub` |
| `*` | `Num a => a -> a -> a` | `Num.mul` |
| `^` | `Num a => a -> a -> a` | `Num.pow` |
| `/` | `Fractional a => a -> a -> a` | `Fractional.fdiv` |
| `//` | `Integral a => a -> a -> a` | `Integral.idiv` |

`Num` covers every numeric type: `Int`, `Int64`, `UInt32`, `UInt64`,
`Float`, `Float32` and the 8- and 16-bit integers. `/` works on the
floating-point types and `//` on the integer types. Both operands must be the same type; convert explicitly with functions
such as `toFloat`.

```geng
7 // 2              -- 3: integer division truncates toward zero
7 / 2               -- 3.5
7 // 0              -- 0: integer division by zero is defined, not a crash
1.0 / 0.0           -- infinity, as IEEE 754 specifies
```

Arithmetic never crashes:

- Integer overflow wraps. `Int` is exactly 32 bits on every platform.
- `x // 0` is `0`.
- Floats follow IEEE 754 on every platform, including infinity and NaN.

For the full rules, see [Types](@/reference/types.md#numbers).

Unary minus is written `-x` and stands for `Num.negate`.

## Comparison

| Operator | Type | Stands for |
|---|---|---|
| `==` | `Eq a => a -> a -> Bool` | `Eq.eq` |
| `/=` | `Eq a => a -> a -> Bool` | not `Eq.eq` |
| `<` `>` `<=` `>=` | `Ord a => a -> a -> Bool` | `Ord.compare` |

Equality and ordering are structural, and the compiler provides them for
records, custom types and arrays. Comparing two functions with `==` is a
compile error, because functions have no `Eq` instance.

`/=` means "not equal". There is no `!=`.

Float equality is defined so that every value equals itself: `nan == nan` is
`True` and `-0.0 == 0.0` is `True`, and NaN sorts after every other float.
This is what makes a `Float` usable as a `Dict` key.

## Boolean

| Operator | Type | Stands for |
|---|---|---|
| `&&` | `Bool -> Bool -> Bool` | `Basics.and` |
| `\|\|` | `Bool -> Bool -> Bool` | `Basics.or` |

Both short-circuit: the right operand is evaluated only if it is needed.
Negation is the function `not`, not an operator.

## Append

| Operator | Type | Stands for |
|---|---|---|
| `++` | `Appendable a => a -> a -> a` | `Appendable.append` |

`++` joins two strings or two arrays:

```geng
"butter" ++ "fly"       -- "butterfly"
[ 1, 2 ] ++ [ 3 ]       -- [ 1, 2, 3 ]
```

## Operators as functions

Wrap an operator in parentheses to use it as an ordinary function:

```geng
Array.foldl (+) 0 [ 1, 2, 3 ]   -- 6
```

## Symbols that are not operators

These appear between values but are part of the syntax, not functions:

| Symbol | Meaning |
|---|---|
| `=` | definition |
| `:` | type annotation, as in `x : Int` or `(e : Int64)` |
| `->` | function type, lambda body, `when` branch |
| `=>` | ends a constraint, as in `Eq a => a -> Bool` |
| `\` | starts a lambda: `\x -> x + 1` |
| `\|` | separates custom type variants; record update `{ r \| x = 1 }` |
| `.` | field access `r.x`; accessor function `.x`; qualified name `Dict.get` |
| `..` | `exposing (Type(..))` |

## Compared with Elm and Gren

- The operator set is Gren's. Elm's list cons `::` does not exist, because
  Gren and Geng have arrays instead of linked lists.
- In Elm and Gren, `==` on functions compiles and crashes at run time. In
  Geng it is a compile error.
- In Gren, `+` and friends are limited to the built-in `number` type. In Geng
  they belong to the `Num` class, which covers every numeric type.
- `infix` declarations are not available to programs or packages.
