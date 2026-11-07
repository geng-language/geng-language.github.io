+++
draft = true
title = "1. Thinking in values"
description = "Values that never change, functions with no hidden effects, and code made of expressions."
weight = 1
+++

Three ideas hold up everything else in a functional language. Values never
change. Functions have no hidden inputs or outputs. And code is made of
expressions that produce values, not statements that do things. None of them
is hard. What takes time is getting used to them all at once.

## Values never change

### A bug you have probably written

Here is a Python budget, and a plan based on it:

```python
budget = {"food": 400_00, "rent": 1500_00}

plan = budget
plan["food"] = 300_00

print(budget["food"])  # 30000
```

The plan was supposed to be a *change* to the budget, but `plan = budget`
does not copy anything. Both names point at the same dictionary, so cutting
the food in the plan cut it in the budget too.

This is a small example of a large problem. When a value can change, every
piece of code that holds it can be surprised by every other piece. A function
you pass a list to might sort it. A cache might hand two callers the same
object. To be sure a value is what you think it is, you have to know
everything that has ever touched it.

### The same thing, written in Python's functional style

Python can avoid this. Instead of changing the dictionary, make a new one:

```python
budget = {"food": 400_00, "rent": 1500_00}

plan = {**budget, "food": 300_00}

print(budget["food"])  # 40000
print(plan["food"])    # 30000
```

Nothing changed. There are two values now, and each one is what it was when
it was made.

Python lets you write this way, but it doesn't check that you do. Someone can
still write `plan["food"] = ...` later, or pass `budget` to a function that
changes it.

### In Geng

In Geng, there is no way to change a value at all, so the first version
cannot be written:

```geng
budget =
    { food = 40000, rent = 150000 }

plan =
    { budget | food = 30000 }
```

`{ budget | food = 30000 }` reads as "`budget`, but with `food` set to
30000". It makes a new record. `budget.food` is still `40000`, now and
forever, no matter what any other code does with `budget`.

This applies to everything: records, arrays, strings, dictionaries. A
function that "adds" to an array returns a new array. A function that "sorts"
one returns a sorted copy. When you hold a value, you know it won't change
under you.

> **If you know C:** think of every variable and every pointer target as
> `const`, enforced by the compiler, with no casts to get around it.
>
> **If you know Rust:** this is Rust with no `mut` anywhere, and no
> `RefCell`. You will not need the borrow checker for most code, because
> nothing can be changed through a borrow.

### Isn't copying slow?

Less than you might think, for two reasons.

First, a new value shares what it does not change. `plan` is a new record,
but it doesn't copy the `rent` field's data; the two records can point at the
same value, because neither can ever change it. The same goes for an array
of records: a new array can hold the same records as the old one.

Second, Geng writes in place when it can prove that nothing else holds the
old value. That is an optimization the compiler makes for you, and your code
does not change because of it.

For the cases where you do need to build up a large array quickly, Geng has
a tool made for it, which [Part 4](@/docs/functional-programming/real-programs.md)
covers.

## Functions with no hidden effects

### Hidden inputs and outputs

Look at what this Python function depends on, and what it changes:

```python
total = 0

def add_expense(expense):
    global total
    total += expense.cents
    print("added", expense.note)
```

Its signature says it takes an expense. In fact it also reads `total`,
changes `total`, and writes to the screen. None of that is in the signature.
To know what calling it does, you have to read its body, and the body of
everything it calls.

Here is another one, with a quieter hidden input:

```python
from datetime import date

def is_overdue(bill):
    return bill.due < date.today()
```

`is_overdue` gives a different answer today than tomorrow for the same bill.
To test it, you have to fake the clock.

### Pure functions

A **pure** function has no hidden inputs or outputs. Its result depends only
on its arguments, and calling it changes nothing. Both Python functions
become pure by moving the hidden parts into the open:

```python
def add_expense(total, expense):
    return total + expense.cents

def is_overdue(today, bill):
    return bill.due < today
```

Now the signature tells the whole story. `add_expense` takes a total and
returns a new one. `is_overdue` takes the date as an argument, so a test just
passes one in. Printing has moved out of the function, to whoever calls it.

In Geng, every function is pure:

```geng
addExpense : Int -> Expense -> Int
addExpense total expense =
    total + expense.cents


isOverdue : String -> Bill -> Bool
isOverdue today bill =
    bill.due < today
```

The first line of each is its type signature. `Int -> Expense -> Int` reads
"takes an `Int` and an `Expense`, and returns an `Int`". (Why it is written
with arrows between the arguments, rather than as a list, comes in
[Part 2](@/docs/functional-programming/loops-without-loops.md).)

Because Geng functions are pure, the signature is a promise. A function of
type `Int -> Expense -> Int` cannot read a global, cannot write a file,
cannot check the clock, and cannot print. If it could, its type would say
so.

That makes code easier to trust, whoever wrote it. You can read a function
on its own and know everything it can do, and so can the compiler. Much of
what Geng's compiler checks, it can check only because of this.

### But programs have to do things

A program that cannot print or read files would be useless, so Geng has a
way to do both. The short version is that a function can *return a
description* of something to do, such as "write this line to the console",
and the runtime carries it out. The description is a value like any other,
and the function that builds it is still pure.
[Part 4](@/docs/functional-programming/real-programs.md) covers how that works.
Until then, our examples compute values, and something else shows them.

## Expressions, not statements

### Everything has a value

Python has two kinds of code. *Statements* do something: `x = 1`, `if`,
`for`, `return`. *Expressions* produce a value: `1 + 2`, `f(x)`,
`"a" if c else "b"`.

Geng has only expressions. Every piece of code produces a value, and a
function's body is one expression whose value is the result. There is no
`return`, because there is nothing else the function could produce.

Python's conditional expression is the model for Geng's `if`:

```python
label = "large" if cents > 100_00 else "small"
```

```geng
label =
    if cents > 10000 then
        "large"

    else
        "small"
```

Since `if` produces a value, it must always have an `else`: otherwise, what
would the value be when the condition is false? For the same reason, both
branches must have the same type.

### No early returns

Early returns are common in imperative code. Here is a Python rule for how
much of an expense to pay back:

```python
def reimbursable(limit, expense):
    if expense.category == "personal":
        return 0
    if expense.cents > limit:
        return limit
    return expense.cents
```

Each `return` leaves the function, and the reader has to track which ones
have already been passed. Written as one expression, the three cases sit
side by side:

```python
def reimbursable(limit, expense):
    return (
        0 if expense.category == "personal"
        else limit if expense.cents > limit
        else expense.cents
    )
```

That is awkward in Python, which is why people rarely write it. In Geng it
is the normal way:

```geng
reimbursable : Int -> Expense -> Int
reimbursable limit expense =
    if expense.category == "personal" then
        0

    else if expense.cents > limit then
        limit

    else
        expense.cents
```

Every case is visible, every case produces an `Int`, and there is no path
through the function that forgets to return something.

> **If you know Go:** a function with several early returns becomes an
> `if`/`else if`/`else` chain, or a `when` (coming in
> [Part 3](@/docs/functional-programming/modeling-data.md)). Guard clauses
> that check for errors become a `Result`, also in Part 3.
>
> **If you know Rust:** Rust is already expression-oriented, and the last
> expression in a block is its value. Geng is the same, minus `return` and
> `;`.

### Naming things with `let`

Long expressions get hard to read, so you can name the pieces with `let`.
Here is a function that formats an amount in cents, such as `-1250` as
`-$12.50`. In Python:

```python
def format_cents(cents):
    sign = ""
    if cents < 0:
        sign = "-"
        cents = -cents
    dollars = cents // 100
    rest = cents % 100
    if rest < 10:
        rest_text = "0" + str(rest)
    else:
        rest_text = str(rest)
    return sign + "$" + str(dollars) + "." + rest_text
```

This changes `sign` and `cents` partway through, so to know what `cents`
means on a given line, you have to know which lines ran before it. In Geng,
each name is defined once:

```geng
formatCents : Int -> String
formatCents cents =
    let
        sign =
            if cents < 0 then
                "-"

            else
                ""

        size =
            if cents < 0 then
                negate cents

            else
                cents

        dollars =
            String.fromInt (size // 100)

        rest =
            remainderBy 100 size

        restText =
            if rest < 10 then
                "0" ++ String.fromInt rest

            else
                String.fromInt rest
    in
    sign ++ "$" ++ dollars ++ "." ++ restText
```

Each name in a `let` has one value, defined once, and the expression after
`in` is the result. Instead of changing `cents`, there is a new name, `size`,
for the new idea. You can read the definitions in any order and each one
means the same thing.

It is longer than the Python, mostly because Geng puts each branch of an `if`
on its own lines. It is also easier to check: there are no lines whose
meaning depends on which lines ran before.

## The expense report so far

Here is what we have for the running example: a record for one expense, and
two of the functions above.

```geng
type alias Expense =
    { date : String
    , category : String
    , cents : Int
    , note : String
    }


lunch : Expense
lunch =
    { date = "2026-11-02"
    , category = "food"
    , cents = 1250
    , note = "lunch with the team"
    }
```

`type alias Expense` gives a name to a record type, much like a frozen
Python dataclass:

```python
@dataclass(frozen=True)
class Expense:
    date: str
    category: str
    cents: int
    note: str
```

Amounts are kept as whole cents in an `Int`, not as a `Float`, so that
adding them up is exact. `formatCents lunch.cents` gives `"$12.50"`, and
`reimbursable 1000 lunch` gives `1000`.

So far we can handle one expense at a time. A report needs to work through
a whole list of them, and Geng has no `for` loop. That is
[Part 2](@/docs/functional-programming/loops-without-loops.md).

## Summary

| Python habit | In Geng |
|---|---|
| change a value in place | make a new value: `{ budget \| food = 30000 }` |
| a function reads or changes a global | pass it in as an argument, and return the new value |
| a function reads the clock or prints | it returns a description of the work instead (Part 4) |
| `if` statement | `if` expression, with an `else` |
| early `return` | `if` / `else if` / `else`, or `when` |
| reassign a variable | give the new idea a new name in a `let` |
