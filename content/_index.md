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

### Fields that must agree

A rule can sit on a type, too. Every place that builds or changes a value of
that type must prove the new value keeps the rule, and every place that reads
one can rely on it. Here a game character's health may never be more than
its maximum:

```geng
type alias Player =
    { name : String
    , health : Int
    , maxHealth : Int
        where 0 <= health && health <= maxHealth && 1 <= maxHealth && maxHealth <= 999
    }
```

<details class="more">
<summary>How every change to a <code>Player</code> is held to its rule</summary>

```geng
-- A potion heals 50, but never past the maximum.
drinkPotion : Player -> Player
drinkPotion p =
    { p | health = min p.maxHealth (p.health + 50) }


-- A curse lowers the maximum by 100, and health has to come down with it.
curse : Player -> Player
curse p =
    let
        newMax =
            max 1 (p.maxHealth - 100)
    in
    { p | maxHealth = newMax, health = min p.health newMax }
```

`drinkPotion` changes one field, so the compiler asks whether the new health
still fits under the maximum, and `min p.maxHealth ...` guarantees it. Leave
the `min` out and the program doesn't build. The compiler shows a player
whose maximum is 1, healed to 50:

```
-- CANNOT SHOW A REFINEMENT -------------------------------------- src/Main.geng

Every `Player` keeps this, so a record of its shape built or updated here must:

    0 <= player.health && player.health <= player.maxHealth && 1 <= player.maxHe
alth && player.maxHealth <= 999

and I cannot show it here:

17|     { p | health = p.health + 50 }
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Here, with this call's values for its names, that is:

    0 <= p.health + 50 && p.health + 50 <= p.maxHealth && 0 < p.maxHealth && p.m
axHealth <= 999

What I know here:

    0 <= p.health && p.health <= p.maxHealth && 0 < p.maxHealth
        && p.maxHealth <= 999
        from the invariant of `Player`

With `p.health = 0` and `p.maxHealth = 1`, it is False.
```

`curse` is subtler. It lowers `maxHealth`, but the rule ties the two fields
together, so health may now be too high. Forget `health = min p.health newMax`
and the compiler refuses, showing a player at full health, 2 of 2, whose
maximum drops to 1. You changed one field, and the rule made you deal with
the other:

```
-- CANNOT SHOW A REFINEMENT -------------------------------------- src/Main.geng

Every `Player` keeps this, so a record of its shape built or updated here must:

    0 <= player.health && player.health <= player.maxHealth && 1 <= player.maxHe
alth && player.maxHealth <= 999

and I cannot show it here:

27|     { p | maxHealth = newMax }
        ^^^^^^^^^^^^^^^^^^^^^^^^^^

Here, with this call's values for its names, that is:

    0 <= p.health && p.health <= newMax && 0 < newMax && newMax <= 999

What I know here:

    0 <= p.health && p.health <= p.maxHealth && 0 < p.maxHealth
        && p.maxHealth <= 999
        from the invariant of `Player`
    (p.maxHealth - 100 <= 0 && newMax == 1)
        || (0 < p.maxHealth - 100 && p.maxHealth - 100 == newMax)
        from `max` on line 25

With `newMax = 1`, `p.health = 2` and `p.maxHealth = 2`, it is False.
```

The rule also caps `maxHealth` at 999, which is how the compiler knows
`p.health + 50` can't overflow. Everywhere else in the program, any
`Player` can be read knowing its health is between 0 and its maximum, with
no check at run time. In [puritty](https://github.com/gilramir/puritty), a
terminal emulator written in Geng, a rule like this on the screen's layout
is what lets every read of a screen row be proved inside the buffer. See
[Refinement types](@/docs/intro-to-refinement-types.md).

</details>

### Overflow bugs

Binary search is one of the first algorithms anyone learns, and for decades
the textbook version had a bug. Its middle index is `(low + high) / 2`, and
on an array of about a billion elements or more, `low + high` overflows. In
2006 Joshua Bloch, who wrote the binary search in Java's standard library,
[reported](https://research.google/blog/extra-extra-read-all-about-it-nearly-all-binary-searches-and-mergesorts-are-broken/)
that his own version had it, "lying in wait for nine years or so" until it
broke someone's program. Geng's compiler finds it, and proves the fixed
version:

```geng
searchBetween : (items : Array Int) -> Int -> (low : Int where 0 <= low) -> (high : Int where high < Array.length items) -> Maybe Int
searchBetween items target low high =
    if low > high then
        Nothing

    else
        let
            -- this was fixed so it cannot overflow
            mid =
                low + (high - low) // 2

        ...
```

<details class="more">
<summary>The bug, the compiler's error, and the proof</summary>

Geng's `Int` is 32-bit, as Java's is, so the same line overflows here too.
The difference is that the compiler finds it. Write the textbook version and
the program doesn't build:

```
-- ARITHMETIC MAY WRAP ------------------------------------------- src/Main.geng

I cannot show that `low + high` stays inside `Int` here:

19|                 (low + high) // 2
                    ^^^^^^^^^^^^

What I know here:

    0 <= low                     from the refinement on `low`
    high < Array.length items    from the refinement on `high`
    low <= Array.length items    inferred for the loop `searchBetween`
    low <= high                  from the test on line 13

With `Array.length items = 2147483647`, `high = 2147483646` and `low = 2`, `low +
high` is 2147483648, past the largest `Int`, 2147483647.

`low + (high - low) // 2` is the same value and cannot wrap.
```

It names values that break it, and the fix. With that fix, the compiler
proves the whole search: no sum overflows, every recursive call keeps `low`
and `high` in their ranges, and `Array.getProven` reads inside the array, so
the read needs no `Maybe` and no bounds check at run time. Nobody wrote the
loop's invariant (`low <= Array.length items`); the compiler worked it out.

```geng
search : Array Int -> Int -> Maybe Int
search items target =
    searchBetween items target 0 (Array.length items - 1)


searchBetween : (items : Array Int) -> Int -> (low : Int where 0 <= low) -> (high : Int where high < Array.length items) -> Maybe Int
searchBetween items target low high =
    if low > high then
        Nothing

    else
        let
            -- this was fixed so it cannot overflow
            mid =
                low + (high - low) // 2

            -- proved inside the array: no Maybe, no check at run time
            value =
                Array.getProven mid items
        in
        if value == target then
            Just mid

        else if value < target then
            searchBetween items target (mid + 1) high

        else
            searchBetween items target low (mid - 1)
```

</details>

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
[Case study: puritty's checks, round by round](@/docs/checks-report.md)
follows how puritty used it over nine rounds.

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
