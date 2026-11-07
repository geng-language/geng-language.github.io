+++
draft = true
title = "2. Loops without loops"
description = "map, filter and fold in place of for loops, state carried through a loop, and functions as values."
weight = 2
+++

Geng has no `for` and no `while`. That sounds like a big loss until you
look at what loops are usually for. Most loops do one of three things:
transform each item, keep some of the items, or combine all of them into one
result. Each of those has a function, and once you know the three, you will
rarely miss the loop.

The examples use the `Expense` record from
[Part 1](@/docs/functional-programming/thinking-in-values.md):

```geng
type alias Expense =
    { date : String
    , category : String
    , cents : Int
    , note : String
    }
```

and, in the short examples, a value `expenses : Array Expense` to work on.
Functions that take an array of expenses call their argument `items`, since
a Geng name cannot shadow another one in scope.

## Transforming each item: `map`

A loop that builds a new list from an old one, one item at a time:

```python
amounts = []
for e in expenses:
    amounts.append(e.cents)
```

Python already has a functional way to say this, the list comprehension:

```python
amounts = [e.cents for e in expenses]
```

In Geng it is `Array.map`, which takes a function and an array, and applies
the function to every item:

```geng
amounts =
    Array.map (\e -> e.cents) expenses
```

`\e -> e.cents` is a lambda: a function with no name, like Python's
`lambda e: e.cents`. Reading a field is common enough to have a shorthand,
`.cents`, which is a function that takes a record and returns its `cents`:

```geng
amounts =
    Array.map .cents expenses
```

Any function will do, not just a lambda. With `formatCents` from Part 1:

```geng
labels =
    Array.map formatCents amounts
```

The result is a new array the same length as the old one. `expenses` is
untouched, as always.

## Keeping some items: `keepIf`

A loop that keeps the items that pass a test:

```python
food = []
for e in expenses:
    if e.category == "food":
        food.append(e)
```

```python
food = [e for e in expenses if e.category == "food"]
```

```geng
food =
    Array.keepIf (\e -> e.category == "food") expenses
```

The function returns a `Bool`, and `keepIf` keeps the items it returns
`True` for. (In many languages this is called `filter`. Gren, which Geng
comes from, chose a name that says which way round the test goes.)

## Combining everything: `fold`

This is the most important section in this part. A great many loops look like
this:

```python
total = 0
for e in expenses:
    total = total + e.cents
```

There is a value that starts somewhere (`0`), and each time round the loop,
it is replaced by a new value computed from the old one and the current item.
That shape is called a **fold**, and in Geng it is `Array.foldl`:

```geng
total : Array Expense -> Int
total items =
    Array.foldl (\e sum -> sum + e.cents) 0 items
```

`foldl` takes three things, and each one is a piece of the Python loop:

| Piece | Python loop | `Array.foldl` |
|---|---|---|
| the starting value | `total = 0` before the loop | `0` |
| the step | the loop body: new total from old total and `e` | `\e sum -> sum + e.cents` |
| the items | `for e in expenses` | `items` |

The step is a function that takes the current item and the value so far, and
returns the new value. `foldl` calls it once per item, from left to right,
and the last value it returns is the result.

Python has this too, as `functools.reduce`, though few people reach for it:

```python
from functools import reduce

total = reduce(lambda sum, e: sum + e.cents, expenses, 0)
```

Note that Python's step takes the value so far first and the item second.
Geng's takes the item first.

**This is a recipe you can use on any loop.** Find the variables the loop
changes. Together, they are the value the fold carries. Their values before
the loop are the starting value. The loop body, rewritten to return the new
values instead of assigning them, is the step.

There is also `Array.foldr`, which goes from right to left. For a sum the
direction doesn't matter. It matters when the order of the result does, and
then `foldr` is occasionally handier. Most code uses `foldl`.

## Carrying more than one value

### A record as the accumulator

A loop that keeps track of several things at once:

```python
count = 0
spent = 0
largest = 0
for e in expenses:
    count += 1
    spent += e.cents
    largest = max(largest, e.cents)
```

By the recipe above, the fold carries all three. Since Geng has no tuples, it
carries them in a record:

```geng
stats =
    Array.foldl
        (\e acc ->
            { count = acc.count + 1
            , spent = acc.spent + e.cents
            , largest = max acc.largest e.cents
            }
        )
        { count = 0, spent = 0, largest = 0 }
        expenses
```

`stats.count`, `stats.spent` and `stats.largest` are the three results. The
record replaces the three variables, and each step builds the next record
from the last one.

### A dictionary as the accumulator

The expense report needs a total for each category. In Python:

```python
totals = {}
for e in expenses:
    totals[e.category] = totals.get(e.category, 0) + e.cents
```

The thing the loop changes is the dictionary, so that is what the fold
carries. It starts empty, and each step returns a new dictionary with one
total updated:

```geng
byCategory : Array Expense -> Dict String Int
byCategory items =
    Array.foldl addToCategory Dict.empty items


addToCategory : Expense -> Dict String Int -> Dict String Int
addToCategory e totals =
    let
        before =
            Dict.get e.category totals
                |> Maybe.withDefault 0
    in
    Dict.set e.category (before + e.cents) totals
```

`Dict.get` might not find the key, so it returns a `Maybe Int` rather than an
`Int`, and `Maybe.withDefault 0` turns "not found" into `0`. This plays the
part of Python's `totals.get(e.category, 0)`. `Maybe` is covered properly in
[Part 3](@/docs/functional-programming/modeling-data.md).

`Dict.set` doesn't change `totals`. It returns a new dictionary with the
key set, which the fold passes to the next step.

## When a fold doesn't fit: recursion

A fold always visits every item. Sometimes a loop should stop early. This one
finds the first expense that takes spending over a budget:

```python
def first_over_budget(budget, expenses):
    spent = 0
    i = 0
    while i < len(expenses):
        spent += expenses[i].cents
        if spent > budget:
            return i
        i += 1
    return None
```

When none of `map`, `keepIf` or `foldl` fits, write the loop as a function
that calls itself. By convention it is called `go`, and it lives in a `let`:

```geng
firstOverBudget : Int -> Array Expense -> Maybe Int
firstOverBudget budget items =
    let
        go i spent =
            when Array.get i items is
                Nothing ->
                    Nothing

                Just e ->
                    if spent + e.cents > budget then
                        Just i

                    else
                        go (i + 1) (spent + e.cents)
    in
    go 0 0
```

Compare it with the Python line by line:

- The loop's variables, `i` and `spent`, are `go`'s arguments.
- Their starting values are the first call, `go 0 0`.
- "Go round again" is a call to `go` with the new values.
- Leaving the loop is returning something other than a call to `go`.

`Array.get` returns `Nothing` past the end of the array, which stops the
loop. The result is a `Maybe Int`, Geng's version of "an index, or `None`".

### Doesn't that use up the stack?

In most languages, a function that calls itself a million times needs a
million stack frames, and runs out. Geng avoids this when the call to `go`
is the very last thing the function does, as it is here. That is called a
**tail call**, and Geng runs it as a jump back to the top of the function, not
as a new frame. A loop written this way runs in constant stack, on every
platform.

### Which one to use

Reach for `map`, `keepIf` and `foldl` first. They say what the loop is for in
one word, and there is no counter to get wrong. Write a `go` function when
none of them fits: stopping early, or walking two arrays at once, or anything
with an unusual step.

[Part 5](@/docs/functional-programming/what-geng-adds.md) comes back to this
loop, and shows how Geng can read `items` by index without a `Maybe` at
all.

## Functions are values

You have been passing functions around since the start of this part:
`Array.map` takes one, and so do `keepIf` and `foldl`. In Geng, a function is
a value like a number or a string. You can name it, pass it, store it in a
record, and return it from another function.

### Partial application

Here is `reimbursable` from Part 1, and its type:

```geng
reimbursable : Int -> Expense -> Int
```

The arrows are not just notation. `Int -> Expense -> Int` means
`Int -> (Expense -> Int)`: a function that takes an `Int` and *returns a
function* that takes an `Expense`. So you can give it just the first
argument:

```geng
underLimit : Expense -> Int
underLimit =
    reimbursable 1000
```

`underLimit` is a new function, with the limit already filled in. That is
**partial application**, and it is how most Geng code passes a function that
needs some setup:

```geng
payBack =
    Array.map (reimbursable 1000) expenses
```

Python can do the same with `functools.partial(reimbursable, 1000)`, or a
lambda. In Geng it needs neither, because every function already works this
way.

### Why the data comes last

This is why Geng's own functions take the thing they work on as the *last*
argument: `Array.map f array`, `Dict.set key value dict`,
`String.split separator text`. The arguments that set up the job come first,
so you can fill them in and get back a function that is waiting for the data.

Your own functions should follow the same rule, as `reimbursable` does: the
limit first, the expense last.

### Pipelines

Put several steps together and you get nesting:

```python
food_total = sum(e.cents for e in expenses if e.category == "food")
```

```geng
foodTotal =
    Array.foldl (+) 0 (Array.map .cents (Array.keepIf (\e -> e.category == "food") expenses))
```

That Geng reads inside out. The pipe operator, `|>`, lets you write the same
thing top to bottom, in the order it happens:

```geng
foodTotal =
    expenses
        |> Array.keepIf (\e -> e.category == "food")
        |> Array.map .cents
        |> Array.foldl (+) 0
```

`x |> f` means `f x`: the value on the left becomes the last argument of the
function on the right. Because the data is always the last argument, every
step in a pipeline lines up. `(+)` is the `+` operator used as a function, so
`Array.foldl (+) 0` adds up an array of numbers.

> **If you know Rust:** this is an iterator chain,
> `expenses.iter().filter(...).map(...).sum()`, with `|>` in place of the
> dots. Unlike Rust's iterators, each step here makes a whole new array; the
> difference rarely matters for code like this.
>
> **If you know Go:** Go writes these as loops. Each step in a pipeline
> corresponds to one loop, with a slice built up in each.

## The expense report so far

We can now compute the whole report from an array of expenses. Each line is
a label padded to ten characters, then the amount:

```geng
reportLines : Array Expense -> Array String
reportLines items =
    let
        line label cents =
            String.padRight 10 ' ' label ++ formatCents cents

        categoryLines =
            byCategory items
                |> Dict.foldl
                    (\category cents lines -> Array.pushLast (line category cents) lines)
                    []
    in
    Array.pushLast (line "total" (total items)) categoryLines
```

`line` is a small function defined in the `let`, used twice. `Dict.foldl` is
a fold over a dictionary, in order of its keys, so the categories come out
sorted. Its step takes each key and value along with the value so far, here
the array of lines built up so far, starting from the empty array `[]`.

For the four expenses in the example file (leaving out the bad line for
now), `reportLines` gives:

```text
food      $12.50
office    $23.99
travel    $48.00
total     $84.49
```

There is still a weakness. `category` is a `String`, so a typo such as
`"fod"` becomes a category of its own, and nothing catches it. And we have
assumed every line of the file is a valid expense. Fixing both is
[Part 3](@/docs/functional-programming/modeling-data.md).

## Summary

| Python | In Geng |
|---|---|
| `for` loop that builds a list | `Array.map` |
| `[f(x) for x in xs]` | `Array.map f xs` |
| `[x for x in xs if p(x)]` | `Array.keepIf p xs` |
| `for` loop that updates a total | `Array.foldl` |
| `functools.reduce` | `Array.foldl`, with the step's arguments the other way round |
| several variables updated in a loop | a fold that carries a record |
| `while`, `break`, early `return` from a loop | a `go` function that calls itself |
| `lambda x: ...` | `\x -> ...` |
| `functools.partial(f, a)` | `f a` |
| nested calls, method chains | a pipeline with `\|>` |
