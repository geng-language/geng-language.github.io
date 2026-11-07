+++
# Outline only: left out of the site until it is written.
draft = true
title = "Cheat sheet: Python, Go, C++ to Geng"
description = "Side-by-side equivalents of common constructs."
weight = 25
+++

> [!NOTE]
> Outline only. Each section will be a table with columns Python | Go | C++ | Geng.
> Where a construct has no Geng equivalent, the Geng cell says what to do instead.

## Basics

- Comments, line and block.
- Naming a value (`x = 1`, `x := 1`, `auto x = 1;`, `x = 1` in a `let`).
- Function definition and call.
- Type annotations.
- String concatenation and interpolation (Geng has no interpolation: `++` and `String.fromInt`).

## Control flow

- `if`/`else` (and Geng's mandatory `else`).
- `switch`/`match` → `when ... is`.
- `for` loop over a collection → `Array.map` / `Array.foldl`.
- `while` loop → recursion.
- `break` / `continue` / early `return` → no equivalent; restructure.

## Data

- Arrays / lists / slices / vectors → `Array`.
- Appending to a list in a loop, `append` to a slice, `push_back`, `StringBuilder` → `Array.Builder`, written in place and used once.
- Dicts / maps → `Dict`.
- Sets → `Set`.
- Structs / classes → records (no methods, no inheritance).
- Interfaces / traits → no user-defined classes yet; pass functions or records of functions.
- Enums and tagged unions → custom types.
- Tuples → records (Gren and Geng have no tuples).
- Updating a field.
- Range types, `assert`, validators (`conint(gt=0)`) → [refinement types](@/docs/type-system.md#refinement-types), checked at compile time.
- Bounds-checked indexing (`a[i]`, `a.at(i)`) → `Array.get` (returns a `Maybe`), or `Array.getProven` when the compiler can prove the index is in range.

## Missing values and errors

- `None` / `nil` / `nullptr` → `Maybe`.
- `raise` / `return err` / `throw` → `Result`.
- `try`/`except`, `if err != nil`, `try`/`catch` → `when ... is` on a `Result`, or `Result.andThen`.

## Functions

- Lambdas.
- Higher-order functions.
- Methods → plain functions taking the value as the last argument (so `|>` works).
- Default and keyword arguments → a record argument.

## Program structure

- Modules, packages, imports.
- Entry point (`if __name__ == "__main__"`, `func main()`, `int main()`) → `main : Task Never {}`.
- Visibility (`_name`, lowercase, `private`) → `exposing`.

## Things with no equivalent

- Mutation, global variables, inheritance, exceptions, `goto`, pointers.
- One line each on why, and what to use instead.
