+++
title = "Error handling"
description = "How Geng makes errors expected and handled."
weight = 40
+++

In Geng, a function that can fail says so in its type, and the caller must
deal with both outcomes before the code compiles. There is no `null`, no
exception, and no `throw`, so there is no invisible control flow.

This does not make programs correct: logic bugs are still possible. It means
that the ways a function can fail are written down, and the compiler checks
that someone handles them.

## Missing values: `Maybe`

`Maybe a` is either `Just a` or `Nothing`. It replaces `None`, `nil` and
`nullptr`.

```geng
type Maybe a
    = Just a
    | Nothing
```

Standard functions that might have no answer return a `Maybe`:

```geng
Array.get : Int -> Array a -> Maybe a
Array.first : Array a -> Maybe a
Dict.get : Ord k => k -> Dict k v -> Maybe v
String.toInt : String -> Maybe Int
Char.fromCode : Int -> Maybe Char
```

To use the value, match on it:

```geng
greeting : Maybe String -> String
greeting maybeName =
    when maybeName is
        Just name ->
            "Hello, " ++ name

        Nothing ->
            "Hello, stranger"
```

or use one of the helpers:

| Function | Does |
|---|---|
| `Maybe.withDefault 0 m` | the value, or `0` if there is none |
| `Maybe.map f m` | apply `f` to the value, if there is one |
| `Maybe.andThen f m` | apply `f`, which itself returns a `Maybe` |

A `Nothing` branch or a default is a choice about what the program does
when the value is missing. When it is the intended answer, and not a
place you hoped would never be reached, mark it with
[`meant`](@/reference/keywords.md#fallbacks-you-mean) and a reason:

```geng
port : Maybe Int -> Int
port given =
    Maybe.withDefault (meant NotSet 22) given
```

The reason is a constructor of your own whose doc comment says why.
`geng checks` lists every fallback the compiler cannot show never runs.
It shows a marked one with its reason, and fails if a later change means
the marked fallback can never run.

## Failures: `Result`

`Result e a` is either `Ok a` or `Err e`. It replaces exceptions and Go's
`(value, err)` pairs, and the error can carry information about what went
wrong.

```geng
type Result error value
    = Ok value
    | Err error
```

Define your own error type as a custom type, not a string, so callers can
match on it:

```geng
type ConfigError
    = MissingField String
    | BadPort String


parsePort : String -> Result ConfigError Int
parsePort text =
    when String.toInt text is
        Just port ->
            Ok port

        Nothing ->
            Err (BadPort text)
```

Helpers:

| Function | Does |
|---|---|
| `Result.map f r` | transform the `Ok` value |
| `Result.mapError f r` | transform the `Err` value |
| `Result.andThen f r` | run another step that can fail |
| `Result.withDefault x r` | the `Ok` value, or `x` |
| `Result.toMaybe r` | drop the error |
| `Result.fromMaybe err m` | turn `Nothing` into `Err err` |

## Chaining steps that can fail

`andThen` runs the next step only if the previous one succeeded. The first
error stops the chain and becomes the result:

```geng
type alias Config =
    { port : Int }


loadConfig : Dict String String -> Result ConfigError Config
loadConfig fields =
    Dict.get "port" fields
        |> Result.fromMaybe (MissingField "port")
        |> Result.andThen parsePort
        |> Result.map (\port -> { port = port })
```

## Errors from effects

A `Task x a` is an effect that either succeeds with an `a` or fails with an
`x`. Reading a file, calling a server and similar work return a `Task`, and
the same patterns apply:

| Function | Does |
|---|---|
| `Task.andThen` | run the next task if this one succeeded |
| `Task.mapError` | change the error type |
| `Task.onError` | recover from an error by running another task |
| `Task.bracket` | acquire a resource, use it, and always release it |
| `Task.finally` | run cleanup whether the task succeeded, failed or was cancelled |

## Reaching `main`

A program's `main` has the type `Task Never {}`. `Never` is a type with no
values, so a task whose error type is `Never` cannot fail. That means every
error has to be handled before it reaches `main`, typically by reporting it:

```geng
main : Task Never {}
main =
    run
        |> Task.onError (\err -> Console.writeErr (errorToString err))
```

## Things that crash in other languages

| Situation | Python / Go / C++ | Geng |
|---|---|---|
| Missing value | `None`, `nil` or null dereference | `Maybe`, checked by the compiler |
| Index out of range | exception, panic, or undefined behavior | `Array.get` returns `Nothing` |
| Missing `Dict` key | `KeyError`, zero value, or inserts a default | `Dict.get` returns `Nothing` |
| Parsing a number | exception, or an error to check | `String.toInt` returns a `Maybe` |
| Unhandled enum case | silent fall-through | compile error: `when` must cover every case |
| Integer overflow | bignum, wrap, or undefined behavior | wraps, the same on every platform |
| Integer division by zero | exception, panic, or undefined behavior | `x // 0` is `0` |
| Float division by zero | exception in Python, infinity elsewhere | infinity, as IEEE 754 specifies |
| Comparing functions with `==` | allowed | compile error |
| Forgetting to close a file | leak | `Task.bracket` releases on every path |

## What can still go wrong

A few things still end a program. Each one ends it with a message saying
what happened, and exit status 1:

- **Running out of stack.** Very deep recursion that is not a tail call can
  exhaust the stack. Tail calls, including calls between a group of local
  functions, run in constant stack on every backend, so this mostly affects
  recursion over deep trees. A native program has a 1 GiB stack, enough for
  ten million frames.
- **Running out of memory.** Allocating is treated as something that always
  succeeds, as in most languages, so a program that runs out of memory ends.
- **A size past a fixed limit.** Sizes are `Int`s, so a `String`, an
  `Array` or `Bytes` cannot be longer than 2³¹ − 1. A function that would
  build one longer, such as `Bytes.flatten`, ends the program and names the
  limit. Where that can happen, a `…Checked` form, such as
  `Bytes.flattenChecked`, returns a `Maybe` instead.
- **`Debug.todo`**, which you write yourself for code that is not finished.
  It names its module and line, and an `--optimize` build refuses it.

A crash is never how a function reports an ordinary failure. That is what
`Maybe`, `Result` and `Task` are for.
