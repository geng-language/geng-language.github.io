+++
# Outline only: left out of the site until it is written.
draft = true
title = "3. Modeling data"
description = "Records instead of classes, custom types and when, Maybe and Result instead of None and exceptions, and modules instead of private fields."
weight = 3
+++

> [!NOTE]
> Outline only. Each section lists what it will cover. Keep each idea short
> and link to [Geng's type system](@/docs/type-system.md) and
> [Error handling](@/docs/error-handling.md) for the full story.

## Records instead of classes

- A record is a frozen dataclass with no methods. Methods become plain
  functions that take the record, kept in the same module.
- No inheritance. Extensible records (`{ r | name : String }`) cover the
  duck-typing cases.
- No tuples: use a small record.

## Custom types and `when`

- Replacing a Python `Enum`, and an `isinstance` chain over several classes.
- Cases that carry their own data.
- `when ... is` must cover every case; adding a case gives a list of places
  to update.
- Making impossible states impossible (short; link to the type system guide).
- Running example: `type Category = Food | Travel | Office | Other String`
  replaces the `category : String` field from Part 1.
- Aside for Rust: this is `enum` and `match`; skim to the next section.

## `Maybe` instead of `None`

- A `Maybe String` might hold a string; a `String` always does.
- `when` on a `Maybe`, then `Maybe.withDefault` and `Maybe.map` for the
  common cases.

## `Result` instead of exceptions

- Python `try`/`except` and Go's `if err != nil` compared with a function
  that returns `Result`.
- Chaining steps that can fail with `Result.andThen`.
- Running example: parsing one CSV line into `Result String Expense`, so a
  bad line becomes an error message instead of a crash.
- Aside for Rust: `Option` and `Result`, without `?`.

## Modules instead of private fields

- Exposing a type without its constructors, so other modules can't build an
  invalid value.
- Running example: an `Amount` that can only be made by the parser.
- A note that refinement types (Part 5) can replace some of these.

## The expense report so far

- `Category`, `parseLine : Int -> String -> Result String Expense`, and the
  totals from Part 2 keyed by category.

## Summary

- Table: class, method, inheritance, `Enum`, `isinstance`, `None`, `raise`,
  `try`/`except`, `_private` → their Geng forms.
