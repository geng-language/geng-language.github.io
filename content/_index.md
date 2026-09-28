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

<div class="platforms">
<table>
<tr>
<!-- Only Linux works today. When macOS and Windows land, restore the cluster:
<td><span class="logo-tile os-cluster"><img src="/icons/linux.svg" alt="Linux"><img src="/icons/apple.svg" alt="macOS"><img src="/icons/windows.svg" alt="Windows"></span></td>
-->
<td><span class="logo-tile"><img src="/icons/linux.svg" alt="Linux" width="48" height="48"></span></td>
<td><span class="logo-tile"><img src="/icons/javascript.svg" alt="" width="48" height="48"></span></td>
<td><span class="logo-tile"><img src="/icons/wasm.svg" alt="" width="48" height="48"></span></td>
<td><span class="logo-tile"><img src="/icons/erlang.svg" alt="" width="48" height="48"></span></td>
<td><span class="logo-tile"><img src="/icons/java.svg" alt="" width="48" height="48"></span></td>
</tr>
<tr>
<td><strong>Native</strong></td>
<td><strong>JavaScript</strong></td>
<td><strong>WebAssembly</strong></td>
<td><strong>Erlang BEAM</strong></td>
<td><strong>JVM</strong></td>
</tr>
<tr>
<td>native executables and libraries, for speed</td>
<td>in the browser, or as a Node.js program</td>
<td>for the browser, or anywhere that accepts WASM</td>
<td>for extreme uptimes</td>
<td>for interoperability</td>
</tr>
</table>
</div>


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
            (f - 32.0) * 5.0 / 9.0

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
