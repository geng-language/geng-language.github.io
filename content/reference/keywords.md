+++
title = "Keywords"
description = "Every reserved word in Geng, with examples."
weight = 20
+++

> [!NOTE]
> Outline only. The list follows Gren; mark each entry changed, added or removed once Geng's set is final.
> Each entry will have: syntax, a one-line description, a short example, and "see also" links.

## Summary table

- keyword | used for | example.

## Modules

- `module` ... `exposing`: declaring a module and its public names; `exposing (..)` and `Type(..)`.
- `import`, `as`, `exposing`: importing, aliasing, and bringing names into scope.
- `port` (if kept): port modules and port declarations.
- `effect`, `where` (if kept; core packages only in Elm/Gren).

## Types

- `type`: custom types, constructors, type parameters, record payloads.
- `type alias`: naming a record or other type.
- `alias` is only special after `type`.

## Expressions

- `if` / `then` / `else`: always has `else`; chained `else if`.
- `when` / `is`: pattern matching; patterns allowed (literals, constructors, records, arrays, wildcards, `as`).
- `let` / `in`: local definitions, with optional annotations and destructuring.
- `as` in patterns: naming a whole value while destructuring it.

## Operator declarations

- `infix` with `left` / `right` / `non`: how operators are declared (core packages only), precedence and associativity. Link to [Operators](@/reference/operators.md).

## Contextual words

- Words that are special only in certain positions (`left`, `right`, `non`, `alias`, `exposing`) and can otherwise be used as names.

## Reserved for future use

- Words the compiler rejects as names even though they do nothing yet.

### Open questions

- Final keyword list for Geng: which Gren keywords are dropped (`port`? `effect`?) and which are new (FFI? target selection?).
