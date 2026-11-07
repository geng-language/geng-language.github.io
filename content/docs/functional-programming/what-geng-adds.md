+++
# Outline only: left out of the site until it is written.
draft = true
title = "5. What Geng adds"
description = "Checks moved from run time into the compiler, debugging and testing, and what you give up and get."
weight = 5
+++

> [!NOTE]
> Outline only. Each section lists what it will cover.

## Letting the compiler check more

- Everything so far is shared with other functional languages. Refinement
  types go further: rules about values, proved before the program runs.
- Running example: a loop over the expenses that reads by index with
  `Array.getProven`, with no `Maybe` and no bounds check at run time.
- Python's `assert` and Pydantic's `conint(gt=0)` check at run time; this is
  checked once, by the compiler.
- Short; link to [Refinement types](@/docs/type-system.md#refinement-types).

## Debugging and testing

- Print debugging still works: `Debug.log` in development builds, and
  `inspect` anywhere.
- Testing pure functions: no setup, no mocks, no fake clock. Test the
  expense report's core with plain values.

## What you give up, and what you get

- Give up: mutation, loops, `null`, exceptions, class hierarchies.
- Get: no crashes from `null` or unhandled errors, refactoring with the
  compiler listing every place to change, and functions whose type says
  everything they can do.
- An honest note on the learning curve: the first weeks are slower.

## Translation table

- One screen collecting the summary tables from all five parts.

## Where to go next

- [Geng's type system](@/docs/type-system.md),
  [Error handling](@/docs/error-handling.md),
  [Common patterns](@/docs/patterns.md),
  and the [Cheat sheet](@/docs/cheat-sheet.md).
