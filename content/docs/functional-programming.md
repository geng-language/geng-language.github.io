+++
title = "Functional programming for imperative programmers"
description = "What pure functional programming is, for people who know Python, Go or C++."
weight = 20
+++

> [!NOTE]
> Outline only. Each section lists what it will cover.

## Who this is for

- Readers who know Python, Go or C++ and have never used a pure functional language.
- What they will get: enough of the mental model to read and write simple Geng.
- What this is not: a Geng tutorial. Link to the [cheat sheet](@/docs/cheat-sheet.md) for syntax lookups.

## What "pure" means

- A function's result depends only on its arguments, and calling it changes nothing else.
- Contrast with a Python function that appends to a global list, or a Go method that mutates its receiver.
- Why it matters: easy to test, easy to reason about, safe to run in parallel, and the compiler can check more.

## Values, not variables

- No reassignment. `let` names a value; it does not create a box you can change.
- "Changing" a record produces a new record: `{ user | name = "Ana" }`.
- Why this is not slow: structural sharing (brief, no deep dive).
- Side-by-side: incrementing a counter in Python vs Geng.

## Expressions, not statements

- Everything produces a value: `if`/`then`/`else` always has an `else`; `when ... is` always covers every case.
- No `return`, no early exit. The last expression is the result.
- Side-by-side: a Go function with early returns rewritten as one expression.

## Loops without loops

- Recursion as the basic tool (and tail calls, so it does not blow the stack).
- In practice you rarely write recursion: `Array.map`, `Array.filter`, `Array.foldl`.
- Side-by-side: a Python `for` loop that sums, filters and transforms, rewritten three ways.

## Functions are values

- Passing functions as arguments, returning them from functions.
- Lambdas: `\x -> x * 2` vs Python `lambda` and C++ lambdas.
- Currying and partial application: `add 1` is a function. Why every function "takes one argument".
- Pipelines with `|>`: reading data transformations top to bottom.

## Types that model your data

- Records vs structs / classes (no methods, no inheritance).
- Custom types (tagged unions) vs C++ `std::variant`, Go interfaces with a type switch, Python class hierarchies.
- Pattern matching with `when ... is`, and why the compiler insists you handle every case.
- Type inference: you can usually leave annotations out, but top-level annotations are good style.

## No null, no exceptions

- `Maybe` instead of `None`/`nil`/`nullptr`.
- `Result` instead of `raise`/`panic`/`throw` or Go's `(value, err)`.
- Short version here; link to [Error handling](@/docs/error-handling.md) for the full story.

## How does anything happen? Effects

- If functions cannot do I/O, how does a program print, read files or talk to the network?
- The idea: your program *describes* effects as `Task` values, and the runtime performs them. `main` is itself a `Task`.
- Analogy: building a list of instructions vs executing them.
- Link to [Common patterns](@/docs/patterns.md) for concrete I/O examples.

## What you give up, and what you get

- Give up: mutation, loops, `null`, exceptions, class hierarchies.
- Print debugging still works: `Debug.log` in development builds, and `inspect` anywhere.
- Get: no crashes, fearless refactoring, compiler as a pair programmer.
- Honest note on the learning curve.

## Where to go next

- [Cheat sheet](@/docs/cheat-sheet.md), [Error handling](@/docs/error-handling.md), [Common patterns](@/docs/patterns.md).
