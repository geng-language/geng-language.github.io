+++
title = "Operators"
description = "Every operator in Geng, with precedence, associativity and examples."
weight = 30
+++

> [!NOTE]
> Outline only. The list follows Gren; mark differences once Geng's set is final.
> Each entry will have: type signature, the function it stands for, an example, and common mistakes.

## Precedence and associativity

- One table: precedence (9 to 0) | operators | associativity.
- Short explanation of how to read it, with a worked example of how `a |> f <| b` fails to parse (non-associative mixing).

## Function application and composition

- `|>` forward pipe.
- `<|` backward pipe.
- `>>` forward composition.
- `<<` backward composition.

## Arithmetic

- `+`, `-`, `*`: the `number` constraint (Int or Float).
- `/`: Float division.
- `//`: Int division (and what happens dividing by zero).
- `^`: exponentiation.
- Unary `-` (negation) and its spacing rules.

## Comparison

- `==`, `/=`: structural equality; what cannot be compared (functions).
- `<`, `>`, `<=`, `>=`: the `comparable` constraint.

## Boolean

- `&&`, `||`: short-circuiting.
- `not` is a function, not an operator.

## Appending

- `++`: strings and arrays (the `appendable` constraint).

## Operators as functions

- Wrapping in parentheses: `(+)`, `Array.foldl (+) 0`.

## Syntax that looks like an operator but is not

- `->` in function types, lambdas and `when` branches.
- `\` lambda.
- `:` type annotation.
- `=` definition.
- `|` in custom types and record update.
- `.field` accessor functions and `record.field` access.
- `..` in `exposing (..)`.

## Removed from Elm

- `::` (no linked lists), `,` tuples, and anything else Gren or Geng dropped.

## Defining operators

- `infix` declarations: restricted to core packages; why user-defined operators are not allowed.

### Open questions

- Does Geng keep Gren's operator set unchanged?
- Integer division by zero and overflow semantics for `//`, `+`, `*`.
