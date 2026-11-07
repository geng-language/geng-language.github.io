+++
# Outline only: left out of the site until it is written.
draft = true
title = "2. Loops without loops"
description = "map, filter and fold in place of for loops, state carried through a loop, and functions as values."
weight = 2
+++

> [!NOTE]
> Outline only. Each section lists what it will cover.

## `map` and `filter`

- A Python list comprehension `[f(x) for x in xs if p(x)]` is `Array.filter`
  then `Array.map`. Show all three versions (loop, comprehension, Geng).
- Running example: the `cents` of every expense, and only the food expenses.

## `fold`: the loop with an accumulator

- The most important translation in the series: any `for` loop that updates
  a running value is a fold. Name the three parts: the starting value, the
  step, the array.
- Python: the `for` loop, then `functools.reduce`, then `Array.foldl`.
- `foldl` vs `foldr`, briefly.
- Running example: the total of all expenses.

## Carrying more than one value

- Several accumulators at once: a record as the accumulator
  (`{ total, count, largest }`).
- Totals by category with a `Dict` as the accumulator: `Dict.update`.
  Running example: the per-category totals of the report.

## When a fold doesn't fit: recursion

- `while` loops and loops that stop early (`break`) as a small recursive
  `go` function in a `let`.
- Tail calls: why a recursive loop does not run out of stack.
- In practice: reach for `map`, `filter` and `fold` first; write recursion
  when none of them fits.

## Functions are values

- Passing a function as an argument (already done with `map`), and returning
  one.
- Lambdas: `\x -> x * 2` vs Python's `lambda x: x * 2`.
- Why signatures have arrows between arguments: every function takes one
  argument and returns a function for the rest. `reimbursable 1000` is a
  function `Expense -> Int`. Compare `functools.partial`.
- Pipelines with `|>`: a chain of steps read top to bottom, in place of
  nested calls. Why data comes last in Geng's own functions.

## The expense report so far

- `total`, `byCategory`, and a pipeline that turns `Array Expense` into the
  lines of the report.

## Summary

- Table: `for`, comprehension, `reduce`, `while`, `break`, `lambda`,
  `partial`, method chaining → their Geng forms.
