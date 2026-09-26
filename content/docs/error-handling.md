+++
title = "Error handling"
description = "How Geng makes every error expected and handled."
weight = 40
+++

> [!NOTE]
> Outline only.

## The promise

- Two kinds of errors: the ones you expect and the ones you never saw coming. Geng turns the second kind into the first.
- What "won't crash" does and does not mean: logic bugs still happen; the program never dies with an unhandled error.

## Missing values: `Maybe`

- Defining and matching on `Maybe`.
- `Maybe.withDefault`, `Maybe.map`, `Maybe.andThen`.
- Where the standard library returns `Maybe` (e.g. `Array.get`, `Array.first`, `Dict.get`, `String.toInt`).

## Failures: `Result`

- Defining and matching on `Result`.
- `Result.map`, `Result.mapError`, `Result.andThen`, `Result.withDefault`.
- Designing your own error types as custom types, not strings.
- Converting between `Maybe` and `Result`.

## Chaining fallible steps

- Pipelines of `andThen` vs nested `when` expressions.
- Collecting many errors vs stopping at the first.

## Errors from the outside world

- I/O failures arrive as `Task` errors: `Task.onError`, `Task.mapError`.
- Decoding untrusted data (JSON, command-line input) with decoders.

## Things that crash in other languages

- For each: what Python / Go / C++ do, and what Geng does.
- Index out of range.
- Division by zero (integer and float).
- Integer overflow.
- Deep recursion / stack overflow.
- Out of memory.
- Unhandled "impossible" cases (Elm's `Debug.todo`).

## Reaching `main`

- How errors flow up to the top of a program.
- Reporting to the user and choosing an exit code.
- Link to the exit-code recipe in [Common patterns](@/docs/patterns.md).

### Open questions

- Exact behaviour for the items in "Things that crash in other languages" on each target.
- Is there any escape hatch (a panic equivalent), and if so how is it restricted?
