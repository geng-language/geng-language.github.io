+++
title = "Syntax"
description = "Lexical structure: comments, literals, identifiers and layout."
weight = 10
+++

Geng's syntax is Gren's, which is close to Elm's. This page covers the
lexical basics; [Keywords](@/reference/keywords.md) and
[Operators](@/reference/operators.md) cover the rest.

## Comments

```geng
-- a line comment

{- a block comment,
   which can span lines -}

{-| A documentation comment, placed before a declaration. -}
```

## Literals

```geng
42              -- Int
0xFF            -- Int, in hexadecimal
3.14            -- Float
'a'             -- Char, a single Unicode codepoint
"hello\n"       -- String

"""
A multi-line string.
"""

[ 1, 2, 3 ]                 -- Array Int
{ x = 1.0, y = 2.0 }        -- a record
```

An integer literal is an `Int` and a literal with a decimal point is a
`Float`, unless its context asks for another numeric type. To give a literal
a type in place, annotate it:

```geng
(42 : Int64)
(0.5 : Float32)
```

A literal that does not fit its type, such as `(300 : UInt8)`, is a compile
error.

## Identifiers

- Values, functions and record fields start with a lowercase letter:
  `count`, `parseConfig`.
- Types, constructors and modules start with an uppercase letter: `Maybe`,
  `Just`, `Dict`.
- Letters can come from any script. Which characters count as letters is
  decided by a fixed version of the Unicode database, so the same identifier
  is valid everywhere. Module names are ASCII only.
- A name cannot shadow another name already in scope.

## Layout

Indentation is significant, as in Elm:

- Top-level declarations start in the first column.
- Everything that continues a declaration is indented further.
- The branches of a `when` and the definitions in a `let` line up with each
  other.

`geng fmt` produces the standard layout, so in practice you rarely think about
this.
