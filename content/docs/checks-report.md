+++
title = "Case study: puritty's checks, round by round"
description = "How puritty, a terminal emulator written in Geng, used the checks report in nine rounds over one week: fallbacks turned into proofs, bugs found, and what the language learned."
weight = 24
+++

[puritty](https://github.com/gilramir/puritty) is a terminal emulator
written in Geng, and much of its code was written by an LLM. Code like that
compiles and reads well, and reviewing all of it line by line does not
scale. The question we wanted answered was smaller: **where does the program
rely on something the compiler proved, and where does it quietly fall back
to a default?**

`geng checks` answers that. It goes through every function and reports:

- **checks that hold**: what the compiler proved there, such as a read in
  range, a refined argument at a call, or a sum that cannot wrap;
- **fallbacks that never run**: a `Nothing ->` or `Err _ ->` branch, a `_`
  after `Just`, or a `Maybe.withDefault` default that the compiler shows
  cannot be reached;
- **fallbacks that can run**, with values that reach them;
- **open checks**: what it could not prove, such as a sum that can wrap,
  with the values that make it wrap.

This page follows nine rounds of using that report on puritty, over one
week in November 2026. Each round starts with what the report said and
ends with what changed in the program, what it found, and what changed in
Geng. The report lines are copied from snapshots kept at each round.

## The counts at a glance

Rounds 1 to 6 counted only puritty's terminal core, `vt`. Rounds 7 to 9
counted the whole program. The two tables don't compare.

**The terminal core.** Rounds 1–5 used a prototype of the report, whose
"checks hold" counted only checks other than fallbacks:

| round | what changed | can run | never run | undecided | checks hold |
|---|---|---:|---:|---:|---:|
| 1 | nothing: the first report | 192 | 10 | 10 | 46 |
| 2 | the compiler learned `@measure` | 192 | 10 | 10 | 46 |
| 3 | 18 reads rewritten or refined | 174 | 10 | 10 | 101 |
| 4 | `core` promises more, and the solver decides more | 180 | 14 | 0 | 107 |
| 5 | rules on puritty's types | 177 | 14 | 0 | 112 |

Round 6 was the first run of `geng checks` itself, which counts every
check. On the same code it counted 1,218 that hold where the prototype had
counted 112. By the end of round 6, `vt` had 1,597 checks that hold, 151
fallbacks that can run and 41 that never run.

**The whole program** (`geng checks Main`):

| round | what changed | checks hold | never run | can run | open | meant |
|---|---|---:|---:|---:|---:|---:|
| 7, before | the first whole-program report | 1,662 | 42 | 740 | 51 | |
| 7, after | every open check settled | 1,891 | 46 | 736 | 0 | |
| 8, before | two days of new features, unchecked | 1,937 | 46 | 1,115 | 7 | |
| 8, after | open checks settled again | 1,948 | 47 | 1,114 | 0 | |
| 9, after | every fallback that can run, read | 1,985 | 47 | 133 | 0 | 981 |
| 9, fixed | the bugs it found, fixed | 1,985 | 47 | 125 | 0 | 991 |

Between rounds 7 and 8 the program grew by tabs, most of its tmux
integration, sounds and new settings pages, so most of the jump in "can
run" is new code.

## Round 1: the first report

The first report ran over the whole program, and the table above counts the
terminal core. Of its 212 fallbacks, 192 could run as far as the compiler
could tell. Some lines were plain:

```text
Vt.Resize.countBlank 701:17 the Nothing branch on line 701 is unreachable: refuted, with Array.length rows = 0, i = 0, stop = 0
```

Others were open sums, and every one of the 13 it refuted was real:

```text
Vt.Resize.lineStart 498:25 the difference count - k stays within Int: refuted, with count = 2147483647, k = -1
```

A normal build does not ask about sums like that one; the report asks about
all of them. Two things about the report's design came out of reading it.
A "percent proven" figure would oversell: one module's 14% "proven" was
all sums. So the report says "checks hold" and names which checks. And
every refined signature in the program was in `core`, the standard
library: puritty stated no rules of its own yet.

## Round 2: `@measure`, and nothing moved

The compiler gained `@measure`, which lets a custom type's size take part
in proofs. The report came out byte for byte the same: 1,640 lines,
nothing moved.

That was the finding. We read the 202 `vt` fallbacks that could run or
were undecided by hand and sorted them by what would rule each one out:

| kind | how many | what it means |
|---|---:|---|
| intended | 92 | it runs by design: input from outside, a missing parameter |
| a record's rule, nested | 41 | an index is valid because of how a record's fields relate, inside another record |
| restructure | 24 | an index read that an index-free form avoids |
| parameter | 17 | every caller passes a valid index |
| the checker | 11 | the fact is in the code, but the compiler did not use it |
| a loop | 8 | a helper's bound is its caller's test |
| unsure | 7 | the reading could not settle it |
| a record's rule | 1 | as above, not nested |
| a measure | 1 | what `@measure` is for |

puritty declares three custom types, none holding an array or a count, so
only one site in 202 needed a measure. Almost half the fallbacks ran by
design. The 41 nested-record sites were the largest block, and the
compiler could not yet handle them.

## Round 3: eighteen reads, rewritten or refined

The next round worked from that table, in puritty alone. Three kinds of
change covered the eighteen.

**Restructure.** Two arrays that line up, one read at the other's index:

```geng
-- before: a fallback for the mark that might not be there
    Array.indexedMap
        (\k f ->
            if Maybe.withDefault -1 (Array.get k marks) == -2 - j then

-- after: no index, no fallback
    Array.map2
        (\m f ->
            if m == -2 - j then
```

`Array.map2` stops at the shorter array, where the old code went on with
-1. In puritty the two arrays always have the same length, so the change
is safe; in another program the fallback might have been doing work.

**A rule on a constructor.** The EUC-KR decoder's state was any `Int`. Now
it carries the rule the table read needs, and the read is proved:

```geng
type Decoder
    = Decoder Lead

type alias Lead =
    (lead : Int where lead == 0 || (0x81 <= lead && lead <= 0xFE))

-- the read, before and after
    Maybe.withDefault 0 (Array.get ((lead - 0x81) * 190 + byte - 0x41) index)
    Array.getProven ((lead - 0x81) * 190 + byte - 0x41) index
```

**A parameter.** When every caller already knows an index is in range,
the helper's type says so, and the callers prove it:

```geng
dcsHeader : Builder UInt8 -> (i : Int where 0 <= i) -> Int -> Bool
trimmed : (cells : Array Int64) -> (n : Int where n <= Array.length cells) -> Array Int64
```

"Can run" went from 192 to 174, and checks that hold from 46 to 101.
"Never run" did not move, on purpose: each change took the fallback away
rather than proving it unreachable, so the read is now checked by every
build. **Found:** `defaultColor` read the palette from its end for a
negative index, since `Array.get` counts a negative index from the end.

## Round 4: `core` promises more

Some fallbacks were there because `core` promised less than it does.
`Maybe.map` now says it answers a `Just` exactly when it is given one:

```geng
map : (a -> b) -> (maybe : Maybe a) -> (r : Maybe b where Maybe.isJust r == Maybe.isJust maybe)
```

and the solver learned to rule out cases before splitting them. All ten
undecided fallbacks were decided, and four fallbacks in all were shown
never to run, this one among them:

```text
Vt.Keys.functionKey 873:17 the default of Maybe.withDefault on line 873 is unreachable: undecided
Vt.Keys.functionKey 873:17 the default of Maybe.withDefault on line 873 is unreachable: proved
```

The other nine undecided were shown to run. They handled input a remote
program sends, which the reading had already called intended. That is why "can
run" went up, to 180: a fallback the compiler can't decide is worse than
one it shows can run.

Not everything could get a contract. `String.split` cannot promise a
non-empty answer, because `String.split "" ""` is `[]`, and a predicate
cannot yet say the separator is not empty.

## Round 5: rules on types

The keyboard-mode stack's read was the clearest case of a record's rule:

```text
Vt.keyOptions 845:20 the default of Maybe.withDefault on line 845 is unreachable: refuted, with Array.length t.keys.stack = 0, t.keys.at = 0
```

An empty stack never happens: puritty makes it with eight slots and keeps
`at` inside it. Nothing told the compiler. Geng gained **type
invariants**, a rule written on a record type that holds wherever records
of that type go:

```geng
type alias Keys =
    { stack : Array Int
    , at : Int where 0 <= at && at < Array.length stack
    }
```

Every place that builds or changes a `Keys` must now keep the rule. Two
did not, as far as the compiler could see: they moved `at` with
`modBy 8 (k.at + 1)` and `modBy 8 (k.at - 1)`, which is right only while
the stack has eight slots. The rule made them say so:

```geng
at =
    if k.at + 1 < Array.length k.stack then
        k.at + 1

    else
        0
```

The palette got the same treatment (`named : Array Int where
Array.length named == 16`). The compiler checked the invariants at 234
places across the terminal's updates.

## Round 6: every screen row, then every cell

Most of the 41 nested-record sites were reads of a screen row through the
screen inside the terminal. Geng's checker learned products of two values
(`y * cols`), and puritty wrote invariants on its screen and terminal
types and refined 36 functions that make screens. On 2026-11-09 every
read of a screen row in the core was proved in range.

**Found:** the scrollback's count of rows gone only grew. At 32 bits it
would have wrapped after 2³¹ rows, about an hour of `yes`, taking a
selection's place with it. It is capped now:

```geng
addRows : (a : Int where 0 <= a && a <= 536870912) -> (b : Int where 0 <= b) -> (r : Int where 0 <= r && r <= 536870912)
addRows a b =
    if b > 536870912 - a then
        536870912

    else
        a + b
```

The same day `geng checks` was built as a command, and `core`'s
`Array.Builder.get` gained a contract. Then every cell read the compiler
could see was proved. That took the core's fallbacks that never run from
14 to 41.

**Found:** inserting at column 0 read cell -1, which `Builder.get` counts
from the end, so it looked at the buffer's last cell. That was harmless
there. Two binary searches took their midpoint as `(lo + hi) // 2`, which
can wrap; they use `lo + (hi - lo) // 2` now.

**What proofs did not buy: speed.** Reading cells with `getProven`, which
skips the bounds test, was proved and slower: 90 ms where the checked read
took 78 ms. It was not kept. Sums compiled without their wrap checks, as
a proof would allow, saved at most a few percent, and that was not built
either. The speed came from a new primitive instead, `Bytes.spanWithin`,
which took the same case to 59 ms. Here, proofs bought correctness, not
speed.

## Round 7: the whole program, and a guard

`geng checks Main` ran over all of puritty for the first time: the
application, the window, the tmux and ssh packages.

```text
1662 checks hold    42 fallbacks never run    740 can run    51 open
open         inWord:661        the product y * cols can wrap, when cols = 46341, y = 46341
open         menuPress:1402          the sum box.col + box.width can wrap, when box.col = -1, box.width = -2147483648, the value at 1396:54 = 0
```

Each open check was settled where its values are made, not where they are
used. A row's start in the renderer is bounded once:

```geng
rowStart : (y : Int where 0 <= y && y < 1000) -> (cols : Int where 1 <= cols && cols <= 1000) -> (r : Int where 0 <= r && r <= 999000)
rowStart y cols =
    clamp 0 999000 (y * cols)
```

A tmux pane's size is a type invariant, 0 to 16,384 on each axis. Menu
and view boxes take the window's size, 1 to 1,000. The report ended at
0 open:

```text
1891 checks hold    46 fallbacks never run    736 can run    0 open
```

**Found:** the renderer accepted rows down to `0 - s.hist`, so a negative
row could reach a buffer where `Builder.get` counts from the end. The X11
event decoder could read past a short buffer from C; a short or broken
buffer now ends the events.

Then the fallbacks that could run were recorded as a **baseline**. From then on, a new fallback that can run, or any open
check, fails `geng checks --baseline`.

## Round 8: the guard had not run

Two days of features later (tabs, most of the tmux integration, sounds),
the guard was run again, and it failed:

```text
1937 checks hold    46 fallbacks never run    1115 can run    7 open
open         drawListing:6995         the sum (l.selected - listHeight) + 1 can wrap, when Array.length rows = 0, h = 0, l.previewMode = 0, l.selected = 2147483647, listHeight = 0, …
```

It had not been run since round 7. The guard is a command, not a build
step, and nobody ran it. That lesson is about process, not Geng.

The selection in tmux's buffer and client lists became a rule on the
listing's type:

```geng
, selected : Int where 0 <= selected && (selected == 0 || selected < Array.length rows)
```

**Found:** writing that rule found a bug. When tmux's answer emptied the
list, the selection became `clamp 0 (Array.length rows - 1) l.selected`,
which is -1. It is 0 now.

`core` gained contracts on the way: `keepIf` keeps no more than it is
given, and the sorts keep the length. `Array.findIndex` came next, because
`Array.findFirst` cannot promise anything about the index it returns.

## Round 9: every fallback that can run, read

The baseline now held 1,114 fallbacks that could run. A count that large
stops meaning anything. Most of them ran by design, but nothing in the code
said which ones, or why.

Geng gained a way to say it at the fallback, with `meant` and a reason:

```geng
param : Int -> (default : Int where 0 <= default && default <= 65535) -> Array Int -> (r : Int where 0 <= r && r <= 65535 && (default < 1 || 1 <= r))
param k default params =
    when Array.get k params is
        Just 0 ->
            default

        Just v ->
            clamp 1 65535 v

        Nothing ->
            meant Reason.NotSet default
```

The reason is a constructor of a type of puritty's own, and its doc
comment says why. The report shows it:

```text
meant        param:90             the Nothing branch runs when Array.length params = 0, get's result at 79:10.Just.0 = 0, k = 0, meant: Vt.Reason.NotSet (a setting or option is not set, so its default applies)
```

A mark is a claim the compiler tests. If a later change means the marked
fallback can never run, `geng checks` fails and says the mark is stale.
It also fails when a reason is used by no mark.

To mark a fallback, someone has to read it and give a reason. All 1,114
were read. Nine reasons covered the program:

| reason | words |
|---|---|
| `FromOutside` | what came from outside was short, missing or not what was asked |
| `NotSet` | a setting or option is not set, so its default applies |
| `NotOpen` | a part of the model is empty because its menu, search, dialog, prompt or mode is not open |
| `NotKnown` | a name, key or id is not in the program's own tables |
| `OutOfRange` | an index or position is past an edge, and this is the edge's answer |
| `NoMore` | a scan, list or queue ended, and this is its end |
| `NotYet` | it is not known yet; a later event fills it in |
| `Gone` | a pane, tab, window or session closed before an event about it was handled |
| `Refused` | a call to the OS, a file or the network failed |

981 were marked. The other 133 were not meant, and marking them would
have been a false claim:

- **122 can never run**, for reasons the compiler cannot see. A dozen are
  the first or last piece of a `String.split`, which is never empty.
  Others are an index found in one array and read from a parallel one, or
  a field matched just after a test that it is set.
- **11 were bugs** the fallback was hiding. Most were in the ssh client:

```geng
-- a Port that is not a number, or is out of range, became 22
port = String.toInt (one "port" "22") |> Maybe.withDefault 22

-- channel data that did not read was dropped, and the output lost
Err _ ->
    st
```

Others: `ServerAliveInterval 1m` silently became 0, since only bare
seconds were read. A `Match host` with no argument was matched against
`""`. With no `HOME`, `~/.ssh/config` was read from `/`.

All of them were fixed, each against OpenSSH. puritty now reads `Port`,
`ServerAliveInterval` and `ServerAliveCountMax` the way OpenSSH does.
When OpenSSH refuses a line, puritty leaves it out and shows the problem;
tests check every such case against `ssh -G`. A channel message that does
not read ends the session, as an unreadable confirmation already did.
After the fixes, the report reads:

```text
1985 checks hold    47 fallbacks never run    125 can run    0 open    991 meant
```

## What it found

| round | bug | what it did |
|---|---|---|
| 1 | `lineStart`'s `count - k` | could wrap; a normal build did not ask |
| 3 | `defaultColor` | read the palette from its end for a negative index |
| 6 | the scrollback's rows-gone count | would wrap after about an hour of output |
| 6 | an insert at column 0 | read the buffer's last cell; harmless |
| 6 | two binary searches' midpoints | could wrap |
| 7 | the renderer's row range | let a negative row through |
| 7 | the X11 event decoder | could read past a short buffer |
| 8 | the tmux list selection | became -1 when the list emptied |
| 9 | the ssh client and its caller | eleven, listed above; one was harmless |

## What came back to Geng

Each round asked something of the language, and the language changed:

- **The report itself** (rounds 1, 6): "checks hold" in place of a
  percentage, every sum asked about, and `geng checks` as a command with
  a baseline.
- **`core` contracts** (rounds 4, 6, 8): `Maybe.map`,
  `Array.Builder.get`, `keepIf`, the sorts, `setMany`, `takeFirst`,
  `remove`, and the new `Array.findIndex`.
- **The checker** (rounds 4, 6): a solver that rules cases out before
  splitting them, products of two values, and facts that reach a
  branch's value.
- **Type invariants** (round 5): a rule on a record type that holds
  wherever the type goes.
- **`meant`, and doc comments on variants** (round 9).

And some it has not answered yet. `String.split` still makes no
promise, which a dozen of the fallbacks still on the list wait on. A function passed to a fold is
not followed. A local function's result is opaque.

## What did not work

- `@measure` moved nothing in puritty (round 2). It is for types puritty
  doesn't have.
- Proofs did not make puritty faster (round 6). The one proved read
  measured was slower, and unchecked sums would have saved at most a few
  percent.
- The guard only works when someone runs it (round 8).
- 125 fallbacks remain on the list. As far as reading them shows, none
  of them can run, and each waits on the compiler to see why.

## What the report is not

- **Not a measure of correctness.** "Checks hold" says what was proved,
  never that a function does what you meant. There is no "percent
  proven".
- **Not a list of bugs.** Most fallbacks that can run are the program
  handling input. The values beside each are there so you can judge,
  and `meant` is how you record the judgment.
- **Not a replacement for tests.** It tells you where the compiler's
  guarantees end. Tests tell you whether the code inside them is right:
  in the same week, an `Array.append` that put two halves of the tab
  order in the wrong order kept every rule and was caught by a test.
