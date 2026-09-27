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

Very deep recursion that is not a tail call can run out of stack. Tail calls,
including calls between a group of local functions, run in constant stack on
every backend, so this mostly affects recursion over deep trees.

If a program does crash, it exits with status 1.
