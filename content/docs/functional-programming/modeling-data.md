+++
draft = true
title = "3. Modeling data"
description = "Records instead of classes, custom types and when, Maybe and Result instead of None and exceptions, and modules instead of private fields."
weight = 3
+++

In Python, Go or C, most of the design of a program lives in its classes or
structs, and the rest in the code that checks them: `if x is None`,
`try`/`except`, `if err != nil`. In Geng, much more of it lives in the types.
A good type allows only the values that make sense, so the code that would
have checked for the others has nothing left to check.

This part covers the four tools for that: records, custom types, `Maybe`
and `Result`, and modules. Each gets a short treatment here;
[Geng's type system](@/docs/type-system.md) and
[Error handling](@/docs/error-handling.md) tell the full story.

## Records instead of classes

### Data and functions, kept apart

A Python class bundles data with the methods that work on it:

```python
@dataclass(frozen=True)
class Expense:
    date: str
    category: str
    cents: int
    note: str

    def is_large(self):
        return self.cents > 100_00

    def with_note(self, note):
        return replace(self, note=note)
```

A Geng record holds only the data. The methods become ordinary functions that
take the record, usually as the last argument, and live in the same module:

```geng
type alias Expense =
    { date : String
    , category : String
    , cents : Int
    , note : String
    }


isLarge : Expense -> Bool
isLarge expense =
    expense.cents > 10000


withNote : String -> Expense -> Expense
withNote note expense =
    { expense | note = note }
```

Instead of `lunch.is_large()`, you write `isLarge lunch`. Instead of a chain
of method calls, you write a pipeline:
`lunch |> withNote "with the team" |> isLarge`.

This is less different than it looks. A Python method is a function whose
first argument is `self`. Geng just doesn't attach it to the data.

### No inheritance

Records cannot extend one another. What inheritance is most often used for,
a function that works on anything with certain fields, Geng does with an
**extensible record**:

```geng
describe : { r | note : String, cents : Int } -> String
describe item =
    item.note ++ ": " ++ formatCents item.cents
```

`{ r | note : String, cents : Int }` means "any record with at least a `note`
and a `cents`". An `Expense` qualifies, and so would an `Invoice` with those
fields among others. It is Python's duck typing, checked by the compiler.

### No tuples

Where Python returns a tuple, Geng returns a small record:

```python
def bounds(expenses):
    return (min(...), max(...))
```

```geng
bounds : Array Expense -> { lowest : Int, highest : Int }
```

The fields have names, so `result.highest` says what it is, where
`result[1]` doesn't.

## Custom types: one of several things

### Replacing strings with a type

The expense report has a weakness from Part 2: a category is a `String`. A
file that says `fod` instead of `food` gets a category of its own, and every
function that looks at categories has to wonder what strings it might see.

Python's answer is an `Enum`:

```python
class Category(Enum):
    FOOD = "food"
    OFFICE = "office"
    TRAVEL = "travel"
```

Geng's is a **custom type**, which lists the values a type can have:

```geng
type Category
    = Food
    | Office
    | Travel
```

A `Category` is exactly one of these three. It is not a string or a number
underneath, and there is no fourth value.

### Cases that carry data

An `Enum` runs out when a case needs data of its own. Suppose that instead of
rejecting categories we don't know, we want to keep their names. In Python
you would need a separate field, or a class per case. In Geng, a case can
carry a value:

```geng
type Category
    = Food
    | Office
    | Travel
    | Other String
```

`Other "lodging"` is a `Category` too. The other three carry nothing, and
`Other` carries the name. There is no way to make a `Food` with a name, or an
`Other` without one.

### Taking one apart: `when`

To use a `Category`, you match on it with `when ... is`, which has one
branch for each case:

```geng
categoryName : Category -> String
categoryName category =
    when category is
        Food ->
            "food"

        Office ->
            "office"

        Travel ->
            "travel"

        Other name ->
            name
```

The `Other name` branch pulls out the string the case carries, and names it.

Going the other way, from the text in the file to a `Category`, matches on a
string. A `when` can match literal values too, and `_` matches anything
left:

```geng
categoryFromString : String -> Category
categoryFromString text =
    when text is
        "food" ->
            Food

        "office" ->
            Office

        "travel" ->
            Travel

        _ ->
            Other text
```

### The compiler checks every case

A `when` must cover every case. Leave out the `Travel` branch in
`categoryName`, and the program doesn't compile: the compiler says which case
is missing.

This pays off when the type changes. Add `| Lodging` to `Category`, and every
`when` on a `Category` that doesn't handle it becomes a compile error,
pointing at the line. The compiler gives you the list of places to update.
In Python, an `if`/`elif` chain over an `Enum` doesn't notice a new member,
and the missing case shows up when it runs, if at all.

> **If you know Rust:** this is `enum` and `match`, including exhaustiveness
> checking. Skim to the next section.
>
> **If you know Go:** the closest thing is an interface with a type switch,
> but Go doesn't check that the switch covers every type. For an `Enum`-like
> type with no data, Go uses `const` and `iota`, which allow any integer.
>
> **If you know C:** a custom type is a tagged union whose tag the compiler
> keeps in step with the data, and whose `switch` must cover every tag.

### Making impossible states impossible

Custom types are the main tool for ruling out values that make no sense. A
Python record with `loading: bool`, `data: bytes | None` and
`error: str | None` allows "loading, with an error, and data", which can't
happen but must still be handled. A custom type with the four states that
*can* happen leaves nothing to handle. See
[Making impossible states impossible](@/docs/type-system.md#making-impossible-states-impossible)
for the full example.

## `Maybe` instead of `None`

### A value that might not be there

In Python, any value might be `None`, and nothing in the type says so:

```python
def find_by_note(note, expenses):
    for e in expenses:
        if e.note == note:
            return e
    return None
```

A caller who forgets the `None` case gets an `AttributeError`, later and
somewhere else.

Geng has no `None`. A function that might have no answer returns a
**`Maybe`**, a custom type from the standard library:

```geng
type Maybe a
    = Just a
    | Nothing
```

`Maybe Expense` is either `Just` an expense or `Nothing`. The `a` is a type
parameter, like a generic in Go or a template in C++: `Maybe Int`,
`Maybe String` and `Maybe Expense` are all `Maybe`s.

You met one in Part 2: `Dict.get` returns a `Maybe`, because the key might not
be there. `Array.get` does too, and so does `String.toInt`, since the text
might not be a number.

### Using one

An `Expense` and a `Maybe Expense` are different types. You can't read `.note`
from a `Maybe Expense`, so you can't forget to check: the compiler makes you
take it apart first, with `when`:

```geng
noteOrBlank : Maybe Expense -> String
noteOrBlank found =
    when found is
        Just expense ->
            expense.note

        Nothing ->
            ""
```

For the common cases there are helpers. `Maybe.withDefault` gives a value to
use when there is none, as in Part 2, and `Maybe.map` changes the value if
there is one:

```geng
Maybe.withDefault 0 (Dict.get "food" totals)   -- the total, or 0

Maybe.map .note found                          -- Just the note, or Nothing
```

## `Result` instead of exceptions

### Failure you can see

Now the bad line in the example file. Here is how the Python version might
parse one line:

```python
def parse_line(line):
    date, category, amount, note = line.split(",")
    cents = int(amount.replace(".", ""))
    if cents < 0:
        raise ValueError("amount is negative")
    return Expense(date, Category.from_str(category), cents, note)
```

This raises an exception if the line has the wrong number of fields, if the
amount isn't a number, or if it is negative. Nothing in the signature says
so. A caller that forgets `try`/`except` works until the first bad file.

Geng has no exceptions. A function that can fail returns a **`Result`**:

```geng
type Result error value
    = Ok value
    | Err error
```

`Result String Int` is either `Ok` with an `Int`, or `Err` with a `String`
saying what went wrong. Like `Maybe`, it is a different type from `Int`, so a
caller must handle both before using the number.

### Parsing an amount

The amount in the file looks like `12.50`. To get whole cents, check it has
two digits after the point, and read the digits without it:

```geng
parseCents : String -> Result String Int
parseCents text =
    let
        problem =
            "\"" ++ text ++ "\" is not an amount"
    in
    when String.split "." text is
        [ dollars ] ->
            String.toInt dollars
                |> Maybe.map (\d -> d * 100)
                |> Result.fromMaybe problem

        [ dollars, cents ] ->
            if String.count cents == 2 then
                String.toInt (dollars ++ cents)
                    |> Result.fromMaybe problem

            else
                Err problem

        _ ->
            Err problem
```

`String.split "." text` gives an array of the pieces, and `when` matches on
how many there are: `[ dollars ]` is an array of exactly one, `[ dollars,
cents ]` of exactly two, and `_` is anything else. `Result.fromMaybe problem`
turns `Just n` into `Ok n`, and `Nothing` into `Err problem`.

So `parseCents "12.50"` is `Ok 1250`, `parseCents "48"` is `Ok 4800`, and
`parseCents "abc"` is `Err "\"abc\" is not an amount"`.

### Steps that can fail, one after another

Parsing a line has several steps that can fail. `Result.andThen` runs the
next step only if the last one succeeded, and the first error becomes the
result:

```geng
parseLine : String -> Result String Expense
parseLine line =
    when String.split "," line is
        [ date, category, amount, note ] ->
            parseCents amount
                |> Result.andThen notNegative
                |> Result.map
                    (\cents ->
                        { date = date
                        , category = categoryFromString category
                        , cents = cents
                        , note = note
                        }
                    )

        _ ->
            Err "expected 4 fields"


notNegative : Int -> Result String Int
notNegative cents =
    if cents < 0 then
        Err "the amount is negative"

    else
        Ok cents
```

Read the pipeline as: parse the amount; then, if that worked, check it is not
negative; then, if that worked, build the expense. `Result.map` changes the
value inside an `Ok` and leaves an `Err` alone.

Everything that can go wrong is now in the type: `parseLine` returns
`Result String Expense`, and a caller cannot get at the `Expense` without
deciding what to do with the `String`. In
[Part 4](@/docs/functional-programming/real-programs.md), the report does that
by listing each bad line instead of stopping.

(This simple parser doesn't handle notes with commas in them. A real CSV
reader would, and would return a `Result` in just the same way.)

> **If you know Go:** a `Result` is your `(value, err)` pair, except that you
> can't use the value without looking at the error. `Result.andThen` is the
> `if err != nil { return err }` that you no longer write by hand.
>
> **If you know Rust:** this is `Option` and `Result`. There is no `?`
> operator; chain steps with `andThen`, or match with `when`.

## Modules instead of private fields

Python marks private fields with a leading underscore, and trusts you not to
touch them. Geng enforces privacy through modules.

Suppose an amount must never be negative. Put the type in its own module, and
expose the type without its constructor:

```geng
module Amount exposing (Amount, fromCents, toCents)


type Amount
    = Amount Int


fromCents : Int -> Maybe Amount
fromCents cents =
    if cents >= 0 then
        Just (Amount cents)

    else
        Nothing


toCents : Amount -> Int
toCents (Amount cents) =
    cents
```

`exposing (Amount, ...)` makes the type visible, but not the constructor
`Amount` that builds one. (Writing `Amount(..)` would expose that too.) So
code in other modules can only get an `Amount` from `fromCents`, and every
`Amount` in the program is at least zero. Functions that take an `Amount`
never need to check.

This is a common pattern, sometimes summed up as "parse, don't validate":
check once, where the data comes in, and let the type carry the result.
[Part 5](@/docs/functional-programming/what-geng-adds.md) shows a way to state
rules like this one that the compiler itself understands, with no wrapper.

## The expense report so far

`Expense` now has a real category:

```geng
type alias Expense =
    { date : String
    , category : Category
    , cents : Int
    , note : String
    }
```

`parseLine` turns a line of the file into a `Result String Expense`.
`byCategory` from Part 2 now builds a `Dict Category Int`, since a custom
type can be a dictionary key, and `reportLines` uses `categoryName` to turn
each key into a label. The categories come out in the order the type lists
them, which here is alphabetical, followed by any `Other` ones.

All the pieces are pure functions, tested by passing in values and checking
what comes out. What's left is reading the file and printing the result,
which is [Part 4](@/docs/functional-programming/real-programs.md).

## Summary

| Python | In Geng |
|---|---|
| class with fields | record (`type alias`) |
| method | a function in the same module, taking the record last |
| inheritance, duck typing | an extensible record: `{ r \| name : String }` |
| tuple | a small record |
| `Enum` | custom type |
| a class per case, `isinstance` chain | custom type whose cases carry data, and `when` |
| `None` | `Maybe`: `Just x` or `Nothing` |
| `raise`, `try`/`except` | `Result`: `Ok x` or `Err e`, and `Result.andThen` |
| `_private` fields | a module that exposes a type without its constructor |
