+++
title = "Geng"
description = "Avoid bugs. Write correct software. Run everywhere."
+++

## Why Geng?

There are two types of errors: those you expect, and those you never saw
coming. Geng is a pure functional programming language that makes all
errors **expected** and **handled**. Your code may still have logic bugs,
but it won't crash.

It is descended from Gren, which is descended from Elm, which is similar
in syntax to Haskell, but has a much reduced feature set, keeping your
programs **simple** to think about.

But **Geng** was designed to run your programs where you need them:

* As native Linux executables, for speed
* In Javascript, in the browser, or as a node.js program
* As WASM, for the browser, or anywhere that accepts WASM
* On the Erlang BEAM, for extreme uptimes
* On the JVM, for interoperability

The same program, on all these platforms.

## A taste of Geng

```geng
type Reading
    = Celsius Float
    | Fahrenheit Float

toCelsius : Reading -> Float
toCelsius reading =
    when reading is
        Celsius c ->
            c

        Fahrenheit f ->
            (f - 32) * 5 / 9

describe : Array Reading -> String
describe readings =
    when Array.first readings is
        Nothing ->
            "No readings yet"

        Just first ->
            "First reading: " ++ String.fromFloat (toCelsius first) ++ "°C"
```

There is no `null` here and nothing to throw: an empty array is a case
the compiler makes you handle.

Geng is not released yet. See [Status and roadmap](@/docs/status.md).
