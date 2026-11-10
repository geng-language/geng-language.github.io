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

## Runtime asserts, moved to compile time

Other languages check many things with an assert while the program runs: is
this index in range? is this array empty? is this number between 0 and 100?
can this sum overflow? In Geng these rules go in a function's type, and the
compiler checks every call against them before the program is built. You
check once where the data comes in, and the compiler proves the rule holds
everywhere after that. Code past that point doesn't repeat the check, doesn't
slow down for it, and can't forget it.

A rule can sit on a type, too, and then every value of that type keeps it.
This is the screen of [puritty](https://github.com/gilramir/puritty), a
terminal emulator written in Geng, cut down to its layout: one flat buffer
of cells. The rule after `where` is written once. The compiler checks it
everywhere a `Screen` is built or changed, and knows it everywhere one is
read. From it, the compiler proves that `slot` lands inside the ring, that
`rowOffset` leaves room for a whole row (multiplying two values, without
overflowing), and that `cellAt` reads inside the buffer, with no test at run
time. Take `stride == cols + 1` out of the rule, and the program no longer
compiles. In puritty itself, every read of a screen row is proved this way.
See [Refinement types](@/docs/type-system.md#refinement-types).

```geng
type alias Screen =
    { cells : Builder Int64
    , cols : Int
    , rows : Int
    , stride : Int
    , base : Int
    , size : Int
        where 1 <= cols && 1 <= rows && rows <= size && stride == cols + 1
            && 0 <= base && base < size && Builder.length cells == size * stride
    }


-- Where row `y` is in the ring.
slot : (s : Screen) -> (y : Int where 0 <= y && y < s.rows) -> (r : Int where 0 <= r && r < s.size)
slot s y =
    if s.base >= s.size - y then
        s.base - (s.size - y)

    else
        s.base + y


-- Where row `y` starts in the buffer: a whole row always fits after it.
rowOffset : (s : Screen) -> (y : Int where 0 <= y && y < s.rows) -> (r : Int where 0 <= r && r + s.stride <= Builder.length s.cells)
rowOffset s y =
    slot s y * s.stride


-- So reading a cell needs no bounds check and no Maybe.
cellAt : (s : Screen) -> (y : Int where 0 <= y && y < s.rows) -> (x : Int where 0 <= x && x < s.cols) -> Int64
cellAt s y x =
    Builder.getProven (rowOffset s y + x) s.cells
```

Geng is not a proof assistant like [Lean](https://lean-lang.org/) or
[Rocq](https://rocq-prover.org/). You don't write proofs, and
the compiler doesn't claim your program does what you meant. Its rules are
refinement types, the approach of
[Liquid Haskell](https://ucsd-progsys.github.io/liquidhaskell/) and
[Flux](https://flux-rs.github.io/flux/) for Rust: a rule is a condition on a
value, and a solver checks it with no proof written by hand. Geng's solver
reasons about whole numbers (lengths, indexes, counts and ranges), and when it
can't show a rule holds, the program doesn't build.

## A report for you, and for your LLM

Code an LLM writes compiles, reads well, and can still hide a
`Maybe.withDefault 0` that has a code smell. The checks report (coming soon) lists
what the compiler proved, function by function. It also lists every fallback
in the program, a `Nothing` branch or a default value, and says which ones
can never run and which ones can. You can see where the program relies on a
proof and where it relies on a fallback, without reading every line. The same
report is something the LLM can act on: turn a reachable fallback into a rule
the compiler checks, build again, and watch the count go down.

From the report for [puritty](https://github.com/gilramir/puritty), a
terminal emulator written in Geng:

<div class="checks" role="figure" aria-label="An excerpt of the checks report for puritty">
<div class="checks-head"><span class="checks-prompt">$ geng checks</span><span class="checks-scope">puritty · vt, the terminal core</span></div>
<div class="checks-totals">
<span class="checks-total held"><b>107</b> checks hold</span>
<span class="checks-total never"><b>14</b> fallbacks never run</span>
<span class="checks-total reach"><b>180</b> can run</span>
<span class="checks-total open"><b>14</b> open</span>
</div>
<div class="checks-module">Vt.Charset</div>
<div class="checks-row"><span class="chip held">checks hold</span><code class="checks-def">go</code><span class="checks-line"></span><span class="checks-what">1 read in range · 21 contract checks · 10 sums cannot wrap · 1 fallback never runs</span></div>
<div class="checks-module">Vt.View</div>
<div class="checks-row"><span class="chip never">never runs</span><code class="checks-def">cellAt</code><span class="checks-line">:1092</span><span class="checks-what">the default of <code>withDefault</code> is never used</span></div>
<div class="checks-row"><span class="chip open">open</span><code class="checks-def">cellAt</code><span class="checks-line">:1089</span><span class="checks-what"><code>row + x</code> can wrap, when <code>row = -2147483648</code>, <code>x = -1</code></span></div>
<div class="checks-module">Vt.Cursor</div>
<div class="checks-row"><span class="chip reach">can run</span><code class="checks-def">param</code><span class="checks-line">:86</span><span class="checks-what">the <code>Nothing</code> branch runs when <code>Array.length params = 0</code></span></div>
</div>

The first is the EUC-KR decoder: its table read is proved in range, because
the decoder's state carries a rule the compiler checks at every step. The
last is a fallback that runs by design, when an escape sequence leaves out its
parameter, and the report shows the values that reach it so you can decide
that at a glance.

## Reliable everywhere

One program, the same behavior on every target.

<div class="platforms">
<table>
<tr>
<!-- Geng ships with Linux and Windows toolchains. When macOS lands, add it:
<td><span class="logo-tile os-cluster"><img src="/icons/linux.svg" alt="Linux"><img src="/icons/apple.svg" alt="macOS"><img src="/icons/windows.svg" alt="Windows"></span></td>
-->
<td><span class="logo-tile os-cluster os-pair"><img class="tux" src="/icons/linux-tight.svg" alt="Linux"><img src="/icons/windows.svg" alt="Windows"></span></td>
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
