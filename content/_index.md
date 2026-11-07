+++
title = "Geng"
description = "Correct by construction. Reliable everywhere."
+++

Speed is easy to measure. Correctness is harder, and it matters more: a fast
program you can't trust is no use. Geng is a small, pure functional language
that moves as much checking as it can into the compiler, so a program is
checked before it runs, not while it does. That matters more now that LLMs
write much of our code. The compiler holds a model's work to the same
standard as anyone's.

## Correct by construction

There is no `null`, no exception to forget, and no case left unhandled. If a
Geng program compiles, those crashes cannot happen.
[What can still go wrong?](@/docs/error-handling.md#what-can-still-go-wrong)

## Checked before it runs

Many checks that other languages make while the program runs (is this index
in range? is this list empty? is this value valid?) can be done by the Geng
compiler. You write the rule in the type, check it once where the data comes
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

`latest` reads the last element directly: no `Maybe`, no bounds check. The
one check is in `report`, and the compiler sees it. Leave out the `if`, and
the program does not compile. See
[Refinement types](@/docs/type-system.md#refinement-types).

## Reliable everywhere

One program, the same behaviour on every target.

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
