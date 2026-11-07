+++
title = "5. What Geng adds"
description = "Checks moved from run time into the compiler, debugging and testing, and what you give up and get."
weight = 5
+++

Nearly everything in Parts 1 to 4 is shared with other pure functional
languages, such as Elm, Gren and Haskell: immutable values, pure functions,
folds, custom types, `Maybe`, `Result` and tasks. This last part covers what
Geng adds on top, the everyday tools for debugging and testing, and an honest
look at the trade.

## Letting the compiler check more

### Rules about values

Custom types and `Maybe` let the compiler check the *shape* of your data: a
`Category` is one of four things, and a `Maybe Expense` might be missing. But
many rules are about *values*. An index must be inside the array. A list
must not be empty. An amount must not be negative.

Most languages check rules like these while the program runs, if at all:

```python
def average_cents(expenses):
    assert len(expenses) > 0
    return sum(e.cents for e in expenses) // len(expenses)
```

The `assert` states the rule, but only checks it when that line runs. If
some caller passes an empty list, you find out in production.

Geng can state the rule in the type, with a **refinement**:

```geng
averageCents : (items : Array Expense where Array.length items > 0) -> Int
averageCents items =
    total items // Array.length items
```

`(items : Array Expense where Array.length items > 0)` reads "an array of
expenses, called `items`, whose length is more than zero". The compiler
checks every call to `averageCents`, and refuses to compile one where it
cannot prove the array is non-empty.

That matters more in Geng than in Python. Integer division by zero in Geng
gives `0` rather than crashing (as
[Part 4](@/docs/functional-programming/real-programs.md#arithmetic-never-crashes)
said), so without the refinement an empty array would quietly report an
average of `$0.00`. With it, that mistake can't compile.

### Checked once, where it belongs

The caller proves the rule the same way you would reason about it: by testing
first.

```geng
averageLine : Array Expense -> String
averageLine items =
    if Array.length items > 0 then
        "average   " ++ formatCents (averageCents items)

    else
        "no expenses"
```

Inside the `then` branch, the compiler knows the array is not empty, because
the `if` just checked it. Take out the `if`, and the program doesn't compile.
The check is still there, but there is exactly one, in the place that knows
what to do about an empty array, and `averageCents` doesn't need one of its
own.

Once checked, the refinement is erased. A refined program runs exactly the
same code as one without refinements, with no checks added at run time.

### Loops without bounds checks

Back to the loop from
[Part 2](@/docs/functional-programming/loops-without-loops.md#when-a-fold-doesn-t-fit-recursion).
It read each expense with `Array.get`, which returns a `Maybe` in case the
index is out of range. Geng's `core` has a second way to read an array,
`Array.getProven`, which returns the element directly, and whose index must
be *proved* to be in range:

```geng
firstOverBudget : Int -> Array Expense -> Maybe Int
firstOverBudget budget items =
    let
        go i spent =
            if i < Array.length items then
                let
                    e =
                        Array.getProven i items
                in
                if spent + e.cents > budget then
                    Just i

                else
                    go (i + 1) (spent + e.cents)

            else
                Nothing
    in
    go 0 0
```

There is no `Maybe` to unpack and no "not found" branch for a case that can't
happen. The `if` proves that `i` is below the length. That it is never
negative, the compiler works out for itself: `go` starts at 0 and only ever
adds 1. Nothing about the loop had to be written down for it.

### What it can and can't check

Refinements are deliberately small. A condition can compare whole numbers,
do simple arithmetic, and talk about sizes such as `Array.length` and
`String.count`. It can't decide whether a string is a valid email address,
or whether a key is in a `Dict`; the module pattern from
[Part 3](@/docs/functional-programming/modeling-data.md#modules-instead-of-private-fields)
is still the tool for those. And data from outside, such as the lines of a
file, still has to be checked once when it comes in. What goes away is
checking it again everywhere else.

They are also optional. Code that never uses one is ordinary Geng. The full
story, including how the compiler explains a refinement it cannot prove, is
in [Refinement types](@/docs/type-system.md#refinement-types).

## Debugging and testing

### Print debugging still works

You can't put a `print` in the middle of a pure function, but you can use
`Debug.log`. It takes a label and a value, prints both, and returns the value
unchanged, so it fits anywhere in a pipeline:

```geng
rows =
    String.lines text
        |> Debug.log "lines"
        |> Array.indexedMap (\i line -> { number = i + 1, line = line })
```

`Debug.log` is for development builds only, so one can't be left behind in a
release by accident.

To turn any value into text on purpose, for a log or an error message, use
`inspect`, which works on records, custom types, arrays and dictionaries, and
is allowed everywhere:

```geng
inspect { number = 4, line = "2026-11-03,food,abc,coffee" }
-- "{ line = \"2026-11-03,food,abc,coffee\", number = 4 }"
```

For code that isn't written yet, `Debug.todo "parse dates"` stands in for any
value. The program compiles, and stops with that message if it gets there.
An optimized build refuses it, so it can't ship.

### Testing pure functions

Most of a Geng program is pure functions, and testing a pure function needs
no setup: give it values, and check what it returns. There is no database to
start, no file to create, and no clock to fake. The tests for the expense
report check expressions like these:

```geng
parseCents "12.50" == Ok 1250

parseCents "12.5" == Err "\"12.5\" is not an amount"

categoryFromString "lodging" == Other "lodging"

report "date,category,amount,note\n2026-11-02,food,12.50,lunch\n"
    == "food      $12.50\ntotal     $12.50\n"
```

The last one tests the whole program except the two lines that read the file
and write the output. That is the payoff of the pure core and thin shell from
Part 4. `geng test` builds and runs a project's tests, on whichever platform
it targets.

## What you give up, and what you get

### What you give up

- **Mutation.** No changing a value in place, and no variables to reassign.
- **Loops.** No `for` or `while`; `map`, `keepIf`, `foldl` and recursion
  instead.
- **`null` and exceptions.** Every missing value and every failure is in a
  type, and must be handled.
- **Classes and inheritance.** Records, functions and modules instead.
- **Doing I/O anywhere.** Effects are `Task`s, and only `main` runs them.

### What you get

- **No crashes from the usual suspects.** No null dereferences, no unhandled
  exceptions, no missing `switch` cases, no arithmetic errors.
- **Functions you can trust from their type.** A function that doesn't
  mention `Task` doesn't do I/O. A function that returns an `Int` always
  returns one.
- **Refactoring with a checklist.** Change a type, and the compiler lists
  every place that needs to change with it.
- **Tests that are easy to write**, because most of the program is pure.
- **More checking before the program runs**, and less while it does.

That last one matters more as more code is written by tools, or by people
reading code they didn't write. The compiler holds every line to the same
standard, and its types say what each function can and can't do.

### The learning curve

The first weeks are slower. You will reach for a loop, or a variable to
update, and have to stop and think how to say it instead. Folds take the
longest to feel natural. That passes. Most programmers who stick with it
find that they think in values and types after a while, and that the
compiler, which felt strict at first, is the most useful reviewer they have.

## Translation table

The summaries from all five parts, in one place:

| Python | In Geng | Part |
|---|---|---|
| change a value in place | make a new one: `{ r \| field = x }` | 1 |
| a function reads or changes a global | pass it in, return the new value | 1 |
| `if` statement, early `return` | `if` expression with an `else`, or `when` | 1 |
| reassign a variable | a new name in a `let` | 1 |
| `[f(x) for x in xs]` | `Array.map f xs` | 2 |
| `[x for x in xs if p(x)]` | `Array.keepIf p xs` | 2 |
| a `for` loop that updates a total | `Array.foldl` | 2 |
| `while`, `break` | a `go` function that calls itself | 2 |
| `lambda x: ...` | `\x -> ...` | 2 |
| `functools.partial(f, a)` | `f a` | 2 |
| method chains | a pipeline with `\|>` | 2 |
| class, method | record, and a function that takes it | 3 |
| tuple | a small record | 3 |
| `Enum`, a class per case | custom type, and `when` | 3 |
| `None` | `Maybe` | 3 |
| `raise`, `try`/`except` | `Result`, and `Result.andThen` | 3 |
| `_private` | a module that doesn't expose the constructor | 3 |
| `print`, `open` | a `Task` | 4 |
| `async`/`await` | `Task.andThen` | 4 |
| `if __name__ == "__main__"` | `main : Task Never {}` | 4 |
| `assert`, validators | a refinement, checked by the compiler | 5 |
| `print` for debugging | `Debug.log` | 5 |
| `repr` | `inspect` | 5 |

## Where to go next

- [Geng's type system](@/docs/type-system.md): custom types, refinements and
  builders in depth.
- [Error handling](@/docs/error-handling.md): `Maybe`, `Result` and `Task` in
  practice, and what can still go wrong.
- [Geng for Elm and Gren programmers](@/docs/coming-from-elm-gren.md), if you
  want to know where Geng's ideas come from.
- [Types](@/reference/types.md), [Keywords](@/reference/keywords.md) and
  [Operators](@/reference/operators.md) in the Reference, for the exact rules.
