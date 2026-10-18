+++
title = "Geng"
description = "Correct by construction. Reliable everywhere."
+++

## Why Geng?

Geng has two goals, and the tagline names them: programs that are
**correct by construction**, and that run **reliably everywhere**.

### Correct by construction

There are two kinds of errors: those you expect, and those you never saw
coming. Geng is a pure functional language that turns the second kind into
the first.

In Geng, your types describe your problem: what a value can be, what a
function returns, and how it can fail. The compiler checks all of it. A
missing case, a value that might not be there, an error nobody handles:
each one is a compile error, not a crash in production. A Geng program that
compiles has been checked far more thoroughly than one in Python, Go or C++.
Your code can still have logic bugs, but it won't crash.

That should not take a specialist. Richer type systems and formal
verification tools can prove more, but they ask more of the programmer than
most projects can afford. Geng comes from Gren and Elm, which keep the
language small, so programs stay **simple** to read and write. Planned
[refinement types](@/docs/type-system.md#refinement-types-planned) will let
the compiler check rules about values too, such as "this number is never
zero", in the same plain style.

### Reliable everywhere

The same Geng program runs where you need it:

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


No platform is the main one with the others as ports. Every backend is
tested against the same conformance suite, and numbers, strings and equality
behave the same on all of them. Geng will always support every one of these
platforms. See [Platforms and targets](@/docs/platforms.md).

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
