+++
draft = true
title = "4. Real programs"
description = "How a pure program does I/O with Task, the finished expense report, and a mental model for performance."
weight = 4
+++

[Part 1](@/docs/functional-programming/thinking-in-values.md) left a question
open. If every Geng function is pure, and cannot print, read a file or check
the clock, how does a Geng program do anything at all? This part answers it,
then puts the whole expense report together, and ends with what to expect
from Geng's performance.

## How anything happens: `Task`

### A recipe, not the cooking

In Python, `print("hello")` prints when that line runs. Calling it *does*
it.

In Geng, `Console.write "hello\n"` doesn't print anything. It is a value, of
type `Task x {}`, that *describes* printing "hello". You can name it, pass it
around, and put it in an array, and none of that prints a thing:

```geng
greet : Task x {}
greet =
    Console.write "hello\n"
```

The difference is the one between a recipe and the cooking. A recipe is a
description you can copy, store and combine with others. Nothing gets cooked
until someone follows it.

In a Geng program, the one who follows it is the runtime, and the recipe it
follows is `main`:

```geng
main : Task Never {}
main =
    greet
```

The program starts, the runtime carries out `main`, and when `main` is done,
the program ends. Every effect a program has, from printing to reading files
to talking to the network, is a `Task` that ends up as part of `main`.

This is how Geng keeps every function pure and still does I/O. A function
that returns a `Task` is still pure: called twice with the same arguments, it
returns the same description twice. Only the runtime ever carries one out.

### What a `Task`'s type says

`Task x a` is a task that, when carried out, fails with an `x` or succeeds
with an `a`.

- `Console.write "hello\n"` succeeds with `{}`, the empty record, since
  there is nothing to give back. It doesn't fail, so it fits any error type,
  which is why its type has a variable `x` there.
- A task that reads a file succeeds with the text, and fails with an error
  saying why it couldn't.
- `main`'s type is `Task Never {}`. `Never` is a type with no values, so a
  task whose error type is `Never` cannot fail. That means every error must be
  handled before it reaches `main`. A Geng program does not crash with an
  unhandled error, because there is no way to leave one unhandled.

### One step after another

`Task.andThen` makes a task that carries out one task, then uses its result to
decide on the next:

```geng
main : Task Never {}
main =
    Console.write "Working...\n"
        |> Task.andThen (\_ -> Console.write "Done.\n")
```

`\_ -> ...` is a lambda that ignores its argument, here the `{}` from the
first write. It is the same shape as `Result.andThen` in
[Part 3](@/docs/functional-programming/modeling-data.md): if the first task
fails, the second is never started, and the failure is the result.

A few more functions cover most programs:

| Function | Does |
|---|---|
| `Task.andThen f task` | carry out `task`, then the task `f` makes from its result |
| `Task.map f task` | change what `task` succeeds with, using a plain function |
| `Task.succeed x` | a task that does nothing and succeeds with `x` |
| `Task.onError f task` | if `task` fails, carry out the task `f` makes from the error |

### Isn't this just async?

It is close. A Python coroutine, `async def`, also returns a description that
something else has to run, and `await` is a lot like `Task.andThen`. The
difference is that in Python, ordinary functions can still do I/O directly;
`async` is one option among several. In Geng, a `Task` is the only way. So a
function whose type doesn't mention `Task` certainly doesn't do any I/O.

## The finished expense report

### A pure core and a thin shell

A good shape for a Geng program, and for programs in many languages, is a
**pure core** inside a **thin shell**. The core is everything from Parts 1
to 3: plain functions from values to values, which make up nearly all of
the program. The shell is the few lines of `Task` that read the input,
hand it to the core, and write out what the core returns.

For the expense report, the core can be a single function from the file's
text to the report's text:

```geng
report : String -> String
```

That makes the shell tiny, and the core easy to test: a test gives `report`
a string and checks the string it gets back. No files, no mocks, no fakes.

### The core

Here is `report`, using `parseLine` from Part 3 and `reportLines` from Part 2:

```geng
report : String -> String
report text =
    let
        rows =
            String.lines text
                |> Array.indexedMap (\i line -> { number = i + 1, line = line })
                |> Array.keepIf (\row -> row.number > 1 && String.trim row.line /= "")

        sorted =
            Array.foldl sortRow { good = [], skipped = [] } rows

        skippedText =
            if Array.length sorted.skipped == 0 then
                ""

            else
                "\n" ++ String.join "\n" sorted.skipped ++ "\n"
    in
    String.join "\n" (reportLines sorted.good) ++ "\n" ++ skippedText


sortRow :
    { number : Int, line : String }
    -> { good : Array Expense, skipped : Array String }
    -> { good : Array Expense, skipped : Array String }
sortRow row acc =
    when parseLine row.line is
        Ok expense ->
            { acc | good = Array.pushLast expense acc.good }

        Err problem ->
            { acc
                | skipped =
                    Array.pushLast
                        ("Skipped line " ++ String.fromInt row.number ++ ": " ++ problem)
                        acc.skipped
            }
```

Step by step:

1. **`rows`** splits the text into lines, numbers them from 1 with
   `Array.indexedMap` (which passes each item's index along with it), and
   keeps every line except the header and blank ones.
2. **`sorted`** is a fold, the record-accumulator pattern from Part 2. Each
   row is parsed, and lands in `good` if it parsed, or in `skipped` with a
   message saying why not.
3. The result is the report lines, then the skipped lines, if there were
   any, after a blank line.

Every line of the input is accounted for. A bad line can't crash the program,
because `parseLine` returns a `Result`, and `sortRow` has to handle the
`Err` before it can get at an `Expense`.

### The shell

The shell reads the file, runs `report` on its text, and writes the result.
If the file can't be read, it says so, and that handles the only error that
can reach `main`:

```geng
main : Task Never {}
main =
    readTextFile "expenses.csv"
        |> Task.map report
        |> Task.andThen Console.write
        |> Task.onError
            (\problem -> Console.writeErr ("Could not read expenses.csv: " ++ problem ++ "\n"))
```

Read it as a recipe: read the file; turn its text into the report; write the
report; and if anything failed, write a message to standard error instead.

A note on `readTextFile`. How a program reads a file depends on where it
runs: Node.js and native programs reach the file system through different
packages, and Geng's standard library has not been settled yet (see
[Status and roadmap](@/docs/status.md#standard-library)). So this example
assumes one function,

```geng
readTextFile : String -> Task String String
```

which succeeds with the file's text or fails with a message. In a real
program it is a line or two over the file API of the platform you build for.
Everything else on this page is the same on every platform.

### The same program in Python

For comparison, here is the Python, written the usual way:

```python
def main():
    try:
        with open("expenses.csv") as f:
            lines = f.read().splitlines()
    except OSError as e:
        print(f"Could not read expenses.csv: {e}", file=sys.stderr)
        return

    totals = {}
    skipped = []
    for number, line in enumerate(lines, start=1):
        if number == 1 or not line.strip():
            continue
        try:
            e = parse_line(line)
        except ValueError as err:
            skipped.append(f"Skipped line {number}: {err}")
            continue
        name = e.category.value
        totals[name] = totals.get(name, 0) + e.cents

    for category in sorted(totals):
        print(f"{category:<10}{format_cents(totals[category])}")
    print(f"{'total':<10}{format_cents(sum(totals.values()))}")
    if skipped:
        print()
        print("\n".join(skipped))
```

It is shorter, and it works. But reading, parsing, totalling and printing
are woven together in one function, so you can't test the totals without a
file, and nothing tells you which lines can raise an exception. The Geng
version separates them because it has to, and that is the version that is
easy to test and to change.

## A mental model for performance

Geng is designed to be fast, but its performance has a different shape from
Python's or Go's. Three things are worth knowing.

### Immutable values are shared, not copied

Since a value never changes, any number of places can hold the same one
safely. Passing a large array to a function copies nothing, and a record
update copies only the record's own fields, not what they point to. And when
the compiler can see that nothing else holds a value, it may update it in
place, as an optimization you don't have to think about.

### Building a large array: `Array.Builder`

`Array.pushLast` returns a new array. The compiler can often reuse the old
one's memory, but it doesn't promise to. When you are building a large array
one element at a time, in a hot loop, use an `Array.Builder` instead. It
writes into one buffer in place, like a Go slice you `append` to, and the
compiler checks that you never keep using an old copy of it. See
[Builders](@/docs/type-system.md#builders-values-used-once).

### Arithmetic never crashes

Integer overflow wraps, on every platform. Dividing an integer by zero gives
zero, rather than an exception or a crash. This keeps every function total,
and keeps Geng programs from crashing on arithmetic. It also means a result
of `0` from a division might be hiding a mistake, and
[Part 5](@/docs/functional-programming/what-geng-adds.md) shows how to rule
that out. See [Arithmetic is total](@/reference/types.md#arithmetic-is-total).

## Summary

| Python | In Geng |
|---|---|
| `print(...)` | `Console.write`, a `Task` that `main` carries out |
| `print(..., file=sys.stderr)` | `Console.writeErr` |
| `open(...).read()` | a `Task` from your platform's file API |
| `async def` and `await` | `Task` and `Task.andThen`, the only way to do I/O |
| `if __name__ == "__main__": main()` | `main : Task Never {}` |
| an exception that reaches the top | impossible: `main`'s error type is `Never` |
| `list.append` in a hot loop | `Array.Builder` |
