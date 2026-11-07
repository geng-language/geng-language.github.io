+++
# Outline only: left out of the site until it is written.
draft = true
title = "4. Real programs"
description = "How a pure program does I/O with Task, the finished expense report, and a mental model for performance."
weight = 4
+++

> [!NOTE]
> Outline only. Each section lists what it will cover.

## How anything happens: `Task`

- The question from Part 1: if every function is pure, how does a program
  print or read a file?
- A `Task` is a description of work, a recipe rather than the cooking. The
  runtime carries out `main`, which is itself a `Task`.
- Running steps in order with `Task.andThen`; turning a value into a task
  with `Task.succeed`.
- What `Task Never {}` in `main`'s type means.
- Link to [Common patterns](@/docs/patterns.md) for files, processes and the
  network.

## The finished expense report

- Read the file, split it into lines, parse each line, total the good ones,
  and print the report and the skipped lines.
- The structure: a pure core (everything from Parts 1–3) with a thin layer
  of `Task`s around it. Why this is easy to test: the core needs no files.
- The whole program, about one screen of Geng, and the Python it replaces.

## A mental model for performance

- Immutable values are shared, not copied, and Geng writes in place when
  nothing else holds a value.
- When to reach for `Array.Builder`: building a large array one element at a
  time. Short, and link to
  [Builders](@/docs/type-system.md#builders-values-used-once).
- Arithmetic never crashes: overflow wraps, division by zero is 0. Link to
  [Types](@/reference/types.md#arithmetic-is-total).

## Summary

- Table: `print`, `open`, `input`, `main()`, `StringBuilder` / `append` →
  their Geng forms.
