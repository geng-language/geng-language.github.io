+++
# Hidden until Gilbert is satisfied with it.
draft = true
title = "Raising your checks with the checks report"
description = "How to read the checks report and turn the fallbacks it finds into rules the compiler proves."
weight = 24
+++

The [refinement types](@/docs/type-system.md#refinement-types) chapter shows
how to write a rule the compiler proves. This guide is about the other end:
you have a program, maybe one an LLM wrote, and you want to know where it
leans on a proof and where it quietly falls back to a default. The checks
report tells you, and this guide shows how to use it to move the program,
one fallback at a time, from the second kind to the first.

The examples are from [puritty](https://github.com/gilramir/puritty), a
terminal emulator written in Geng, whose terminal core went through five
rounds of this.

> [!NOTE]
> The report is built as a prototype, which is what produced every example
> here. It ships as `geng checks Main`, with `--json` and `--html DIR`, the
> way `geng profile` works; until then its output is what you see below.

## What the report says

The report goes through every function and puts what it finds in four
groups:

- **Checks hold.** The rules the compiler proved in that function: a read
  in range, a refined argument at a call, a sum that cannot wrap. The report
  says which, so "checks hold" never stands for "correct": it means these
  particular things were proved.
- **Fallbacks that never run.** A `Nothing ->` branch, an `Err _ ->` branch,
  a `_ ->` after `Just`, or the default of a `Maybe.withDefault`, that the
  compiler shows cannot be reached.
- **Fallbacks that can run**, with values that reach them. These are what
  to read.
- **Open.** A check the compiler could not prove: a sum that can wrap, with
  the values that make it wrap, or a question it could not decide.

Here is part of puritty's, from its fourth round:

```text
107 checks hold    14 fallbacks never run    180 can run    14 open

Vt.Charset
  checks hold  go          1 read in range · 21 contract checks · 10 sums cannot wrap · 1 fallback never runs
Vt.View
  never runs   cellAt:1092 the default of withDefault is never used
  open         cellAt:1089 row + x can wrap, when row = -2147483648, x = -1
Vt.Cursor
  can run      param:86    the Nothing branch runs when Array.length params = 0
```

A fallback that can run is not a bug. `Vt.Cursor.param`'s runs when an
escape sequence leaves out its parameter, which programs do, and the
default is what the standard says to use. The report's job is to show you
the values that reach each fallback, so you can tell in a second whether
that state happens and whether the default is right for it.

## The loop

1. Build the report and look at the fallbacks that can run.
2. For each, read the values that reach it and decide which kind it is
   (the next section).
3. Change the code so the compiler can prove the fallback away, or leave
   it, because it runs by design.
4. Build again. The fallback is gone, and the checks that hold have grown.
5. Keep what you won: hold the program to its list in CI (see
   [Keeping the count](#keeping-the-count)).

Most of the time, step 3 removes the fallback, not just proves it never
runs. A `Maybe.withDefault 0 (Array.get i a)` becomes `Array.getProven i a`,
and there is no `Maybe` left to ask about. That is the stronger result: the
read is checked by the build itself, and a later change that breaks it does
not compile.

## Reading a fallback: what kind is it?

When puritty's core had 202 fallbacks that could run or were undecided,
each was read and sorted by what would rule it out. The kinds, and how many
of puritty's were each:

| kind | puritty | what it means | what to do |
|---|---:|---|---|
| intended | 92 | it runs by design: input from outside, a missing parameter, a key the program does not know | leave it; check the default is right |
| a record's rule | 42 | an index is valid because of how a record's fields relate | write the rule on the record's type |
| restructure | 24 | an index read that an index-free form avoids | walk the arrays together, or carry the element |
| parameter | 17 | every caller passes a valid index | refine the parameter; the callers prove it |
| the checker | 11 | the fact is in the code, but the compiler did not use it | report it; it may be fixed in the compiler |
| a loop | 8 | a helper's bound is its caller's test | write the bound on the helper |

One more was a measure's, and seven the reading could not settle.

Almost half run by design, and that is normal for a program that reads
input. The rest are the ones worth an hour. The sections below show one of
each common kind, as puritty fixed it.

### A record's rule

The report said, of the read of the top of the keyboard-mode stack:

```text
can run   Vt.keyOptions:845   the default of withDefault is used when Array.length t.keys.stack = 0, t.keys.at = 0
```

An empty stack never happens: puritty makes the stack with eight slots and
keeps `at` inside it. But nothing told the compiler. The fix is to write
the rule on the type:

```geng
-- before
type alias Keys =
    { stack : Array Int, at : Int }

-- after
type alias Keys =
    { stack : Array Int
    , at : Int where 0 <= at && at < Array.length stack
    }
```

and read with `Array.getProven k.at k.stack` where it read
`Maybe.withDefault 0 (Array.get k.at k.stack)`. Now every place that builds
or updates a `Keys` must keep the rule. Two did not, as far as the compiler
could see: they moved `at` with `modBy 8 (k.at + 1)`, which is right only
while the stack has eight slots. The rule made them say so:

```geng
at =
    if k.at + 1 < Array.length k.stack then
        k.at + 1

    else
        0
```

That is the usual shape of this fix: the read gets simpler, and the writes
state what they relied on. A rule on a type travels with the type, so a
`Keys` inside the terminal's state, inside a `Maybe`, or handed to a `Task`
keeps it ([Rules a type keeps everywhere](@/docs/type-system.md#rules-a-type-keeps-everywhere)).

### Restructure

Two arrays that line up, one read at the other's index:

```geng
-- before: a fallback for the mark that might not be there
headMarks marks j row x found =
    Array.indexedMap
        (\k f ->
            if Maybe.withDefault -1 (Array.get k marks) == -2 - j then
                { row = row, x = x }

            else
                f
        )
        found

-- after: no index, no fallback
headMarks marks j row x found =
    Array.map2
        (\m f ->
            if m == -2 - j then
                { row = row, x = x }

            else
                f
        )
        marks
        found
```

Check the reason the old fallback was there before you take it away.
`Array.map2` stops at the shorter array, where the old code went on with -1
for a missing mark. In puritty `marks` and `found` always have the same
length, so the two are the same; in another program the fallback might
have been doing work.

### Parameter

When every caller already knows an index is in range, say so in the
helper's type, and the read inside it needs no test:

```geng
-- the callers prove 0 <= i; the read needs nothing more
dcsHeader : Builder UInt8 -> (i : Int where 0 <= i) -> Int -> Bool
```

Each caller now has to prove it, so the fallback moves to where the
knowledge is. If a caller cannot prove it, that caller had a bug, and the
report found it.

### A loop

A loop the compiler infers needs nothing written. A helper whose bound is
its caller's test does: write the bound on the helper's argument, as
puritty's `trimmed` takes `(n : Int where n <= Array.length cells)`, and
the read inside becomes `Array.getProven`.

## Open checks

An open sum is one the compiler could not show stays inside `Int`, with
values that make it wrap:

```text
open   cellAt:1089   row + x can wrap, when row = -2147483648, x = -1
```

Sometimes the values cannot happen and the fix is a rule that says so
(`0 <= row`); sometimes the sum can be rewritten so it cannot wrap, as
`n <= length - offset` for `offset + n <= length`. Open checks never stop a
build on their own: the compiler asks them only where a sum feeds a rule,
and the report asks them everywhere, so you can decide.

## Keeping the count

The report can be held to a list: every fallback that can run and every
open check, by module and function, without line numbers, so that moving
code does not move it. A run that finds something not on the list fails and
names it; something gone from the list passes, and the list is written
again. In CI, this catches the change that adds a silent
`Maybe.withDefault 0` where there was a proof, which is the kind of change
an LLM makes and a reviewer misses.

## puritty's five rounds

Each round used the report to find the next fallbacks to remove. Counts are
for puritty's terminal core:

| round | what changed | can run | never run | undecided | checks hold |
|---|---|---:|---:|---:|---:|
| 1 | nothing: the first report | 192 | 10 | 10 | 46 |
| 2 | the compiler learned `@measure` | 192 | 10 | 10 | 46 |
| 3 | 18 reads rewritten or refined | 174 | 10 | 10 | 101 |
| 4 | `core`'s `Maybe.map` promises more, and the solver decides more | 180 | 14 | 0 | 107 |
| 5 | rules on puritty's types | 177 | 14 | 0 | 112 |

Round 2 changed nothing, and that was the finding: puritty had no types a
measure helps, so the next round went to the reads themselves. Round 4's
"can run" went up because the nine fallbacks that had been undecided were
decided, and most of them run by design (escape sequences from a remote
program). Since round 5, every read of a screen row in puritty's core is
proved, by a rule on the screen's type that the [home page](@/_index.md)
shows a cut-down version of.

## What the report is not

- **Not a measure of correctness.** "Checks hold" says what was proved,
  never that a function does what you meant. There is no "percent proven".
- **Not a list of bugs.** Most fallbacks that can run are the program
  handling input. The values beside each are there so you can judge.
- **Not a replacement for tests.** It tells you where the compiler's
  guarantees end; tests tell you whether the code inside them is right.
