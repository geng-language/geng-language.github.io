+++
title = "Geng"
description = "Correct by construction. Reliable everywhere."
+++

> “A craftsman who wants to do his work well must first sharpen his tools.”
>
> <span lang="zh-Hans">工欲善其事，必先利其器。</span>
>
> — Confucius <span lang="zh-Hans">孔子</span>, *Analects* <span lang="zh-Hans">论语</span>

Speed is easy to measure. Correctness is harder, and it matters more: a fast
program you can't trust is no use. Geng is a small, pure functional language
that moves many of the checks other languages make at run time into the
compiler, so more of a program is checked *before* it runs, not *while* it
runs. That matters even more now that
LLMs write much of our code.

## Correct by construction

There is no `NULL` pointer, no exception to forget, and no case left unhandled. If a
Geng program compiles, those crashes cannot happen.
[What can still go wrong?](@/docs/error-handling.md#what-can-still-go-wrong)

## Checked before it runs

Many checks that other languages make while the program runs (is this index
in range? is this list empty? is this value valid?) can be done by the Geng
compiler **at compile time**. You write the rule in the type, check it once where the data comes
in, and the compiler proves it holds everywhere else. Code past that check
doesn't repeat it, doesn't slow down for it, and can't forget it.

```geng
-- The type says there is always at least one reading.
latest : (readings : Array Float where Array.length readings > 0) -> Float
latest readings =
    Array.getProven (Array.length readings - 1) readings

report : Array Float -> String
report readings =
    if Array.length readings > 0 then
        "Now " ++ String.fromFloat (latest readings) ++ "°C"

    else
        "No readings yet"
```

The `latest` function takes an `Array Float`, but not just any `Array Float`:
only one that the compiler can prove has a length greater than zero. So
`latest` reads the last element directly: no `Maybe`, no bounds check. The
one check is in `report`, and the compiler sees it. Leave out the `if`, and
the program does not compile. Yes, you can assert this at run time in any
language, but Geng enforces it at **compile time**. See
[Refinement types](@/docs/type-system.md#refinement-types).

## Runtime asserts, moved to compile time

Geng is not a proof assistant like Lean or Coq. You don't write proofs, and
the compiler doesn't claim your program does what you meant. Its job is
narrower, and it does it without being asked: the conditions other languages
check with an assert while the program runs become part of a function's type,
and every call is checked against them before the program is built. Is this
index in range? Is this array empty? Is this number between 0 and 100? Can this sum
overflow? It reasons about whole numbers (lengths, indexes, counts and
ranges), and when it can't show a rule holds, the program doesn't build. The
error names the rule, the facts the compiler had, and values that would break
it.

## A report for you, and for your LLM

Code an LLM writes compiles, reads well, and can still hide a
`Maybe.withDefault 0` that has a code smell. The checks report (coming soon) lists
what the compiler proved, function by function. It also lists every fallback
in the program, a `Nothing` branch or a default value, and says which ones
can never run and which ones can. You can see where the program relies on a
proof and where it relies on a fallback, without reading every line. The same
report is something the LLM can act on: turn a reachable fallback into a rule
the compiler checks, build again, and watch the count go down.

## Reliable everywhere

One program, the same behavior on every target.

<div class="platforms">
<table>
<tr>
<!-- Only Linux works today. When macOS and Windows land, restore the cluster:
<td><span class="logo-tile os-cluster"><img src="/icons/linux.svg" alt="Linux"><img src="/icons/apple.svg" alt="macOS"><img src="/icons/windows.svg" alt="Windows"></span></td>
-->
<td><span class="logo-tile"><img src="/icons/linux.svg" alt="Linux" width="48" height="48"></span></td>
<td><span class="logo-tile"><img src="/icons/erlang.svg" alt="" width="48" height="48"></span></td>
<td><span class="logo-tile"><img src="/icons/javascript.svg" alt="" width="48" height="48"></span></td>
<td class="soon"><span class="logo-tile"><img src="/icons/wasm.svg" alt="" width="48" height="48"></span></td>
<td class="soon"><span class="logo-tile"><img src="/icons/java.svg" alt="" width="48" height="48"></span></td>
</tr>
<tr>
<td><strong>Native</strong></td>
<td><strong>Erlang BEAM</strong></td>
<td><strong>JavaScript</strong></td>
<td class="soon"><strong>WebAssembly</strong><span class="soon-badge">Coming soon</span></td>
<td class="soon"><strong>JVM</strong><span class="soon-badge">Coming soon</span></td>
</tr>
<tr>
<td>for speed</td>
<td>for extreme uptime</td>
<td>browser and Node.js</td>
<td class="soon">anywhere WASM runs</td>
<td class="soon">for interop</td>
</tr>
</table>
</div>

See [Platforms and targets](@/docs/platforms.md).

---

[Get started](@/docs/getting-started.md) · Geng is not released yet; see
[Status and roadmap](@/docs/status.md).

Geng started as a fork of [Gren](https://gren-lang.org), which is itself a
fork of [Elm](https://elm-lang.org). We thank everyone who has contributed
to those languages: without them, we would not have had such a strong
foundation.
