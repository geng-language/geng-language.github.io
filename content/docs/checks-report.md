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

`geng checks` answers that question. This page follows nine rounds of
using it on puritty, over one week in November 2026. Each round starts
with what the report said, and ends with what changed in the program,
what it found, and what changed in Geng. The report lines are copied from
snapshots kept at each round.

If refinement types are new to you, start with the next section. It
explains the handful of ideas the rest of the page uses.

## The ideas you need

### A refinement: a type with a rule

A **refinement** is an ordinary type with a condition its values must
meet, written after `where`:

```geng
(n : Int where n > 0)                           -- an Int greater than zero
(i : Int where 0 <= i && i < Array.length rows) -- an index inside rows
```

A refined `Int` is still an `Int`, with nothing to wrap or unwrap. The
difference is that the compiler checks, while it compiles, that the
condition holds everywhere a value of that type is required. If it cannot
show that, the program does not build. Nothing is checked while the
program runs: the proof was done once, at compile time.

### The checker: how the compiler knows

The part of the compiler that does this is the **checker**. It does not
run the program. It reads it, and collects facts the way you would when
reading it:

- a test on the way to a line: inside `if i < Array.length rows then`, it
  knows `i < Array.length rows`;
- the rules on the arguments a function was given;
- what a function it calls promises about its result.

From those facts it tries to prove each rule. When it can't, it does not
guess: the build fails at that spot, and says which values break it.

### A contract: what a function promises

A function's signature can say what it promises about its result. That
promise is a **contract**, and the function's own body is checked to keep
it:

```geng
Array.pushLast : a -> (array : Array a) -> (r : Array a where Array.length r == Array.length array + 1)
```

The caller then knows the new array is one longer, without testing it.
Many of the changes on this page were contracts added to `core`, Geng's
standard library, so that code calling it knew more.

### A fallback: what runs when a value isn't there

Reading from an array can fail: the index may be past the end. So
`Array.get` returns a `Maybe`, and the code must say what happens when
there is nothing there:

```geng
when Array.get i rows is
    Just row ->
        row

    Nothing ->
        emptyRow          -- a fallback

Maybe.withDefault 0 (Array.get i counts)   -- 0 is a fallback too
```

Each `Nothing` branch, each `Err` branch, each `_` after `Just` or `Ok`,
and each default of `Maybe.withDefault` is a **fallback**. Some run by
design: a parameter a program left out of an escape sequence, a setting
nobody set. Others are there only because the compiler could not see that
the value is always present. Those hide mistakes: if the index ever is
wrong, the program carries on with `0` or an empty row, and nothing says
so.

When the compiler can prove an index is in range, the program can skip
the `Maybe` entirely:

```geng
Array.getProven : (i : Int where 0 <= i && i < Array.length array) -> (array : Array a) -> a
```

`getProven` asks for the proof in its signature and returns the element
itself. There is no fallback, because there is nothing to fall back
from.

### A sum that can wrap

An `Int` holds values from −2,147,483,648 to 2,147,483,647. Adding past
the top wraps around to a large negative number, in Geng as in C or Java.
A wrapped index or size is a bug that is hard to see. The report asks of
every sum and product whether it can wrap, and when it can, names values
that make it.

### A rule on a type

A refinement can also be written on a record type, as a rule between its
fields. That is a **type invariant**:

```geng
type alias Keys =
    { stack : Array Int
    , at : Int where 0 <= at && at < Array.length stack
    }
```

Every place that makes or changes a `Keys` must keep the rule, and the
compiler checks each one. In return, every place that reads a `Keys` can
rely on it, wherever the record has travelled.

The [type system chapter](@/docs/type-system.md#refinement-types) covers
all of this at length.

## What the report says

`geng checks` goes through every function and sorts what it finds into
these groups. Each is a column in the tables below.

- **checks hold**: rules the compiler proved in that function, such as a
  read in range, a refined argument at a call, or a sum that cannot wrap.
  The report names which ones. It means "these things were proved", not
  "this function is correct".
- **never run**: fallbacks the compiler showed can't be reached. The value
  is always there, and the fallback is dead code.
- **can run**: fallbacks the compiler showed can be reached, with values
  that reach them. These are the ones to read: each runs by design, or
  could be proved away, or hides a bug.
- **undecided**: fallbacks the compiler could neither rule in nor rule
  out. Only the prototype of the report, used in rounds 1 to 5, counted
  these apart.
- **open**: checks the compiler could not prove, mostly sums that may
  wrap, with the values that make them wrap. From round 6, undecided
  fallbacks count here too.
- **meant**: fallbacks that can run and that the program says are
  intended, with a reason (round 9 explains how).

## The counts at a glance

Rounds 1 to 6 counted only puritty's terminal core, `vt`. Rounds 7 to 9
counted the whole program. The two tables don't compare.

**The terminal core.** Rounds 1–5 used the prototype, whose "checks hold"
counted only checks other than fallbacks:

| round | what changed | can run | never run | undecided | checks hold |
|---|---|---:|---:|---:|---:|
| 1 | nothing: the first report | 192 | 10 | 10 | 46 |
| 2 | the compiler learned `@measure` | 192 | 10 | 10 | 46 |
| 3 | 18 reads rewritten or refined | 174 | 10 | 10 | 101 |
| 4 | `core` promises more, and the checker decides more | 180 | 14 | 0 | 107 |
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

The first report ran over the whole program, and the table above counts
the terminal core. Of its 212 fallbacks, 192 could run as far as the
compiler could tell. The prototype printed one line per question it
asked, like this one:

```text
Vt.Resize.countBlank 701:17 the Nothing branch on line 701 is unreachable: refuted, with Array.length rows = 0, i = 0, stop = 0
```

Read it as: in the module `Vt.Resize`, function `countBlank`, line 701,
the compiler tried to prove that a `Nothing` branch can never run, and
**refuted** it. It found values that reach the branch: an empty `rows`
array, with `i` and `stop` both 0. Whether `rows` can ever be empty
there is now a question a person can answer by reading the code. Before,
nothing pointed at it.

Some lines were about sums:

```text
Vt.Resize.lineStart 498:25 the difference count - k stays within Int: refuted, with count = 2147483647, k = -1
```

`count - k` with `k = -1` is `count + 1`, which wraps when `count` is the
largest `Int`. A normal build does not ask about sums like that one; the
report asks about all of them. It refuted 13, and every one was real.

Two things about the report's design came out of reading it. A "percent
proven" figure would oversell: one module's 14% "proven" was all sums.
So the report says "checks hold" and names which checks. And every
refined signature in the program was in `core`. puritty stated no rules
of its own yet.

## Round 2: `@measure`, and nothing moved

The compiler gained `@measure`, which lets the size of a custom type, such
as the depth of a tree, take part in proofs. The report came out byte for
byte the same: 1,640 lines, nothing moved.

That was the finding. To learn what would help, we read the 202 `vt`
fallbacks that could run or were undecided, by hand, and sorted them by
what would rule each one out:

| kind | how many | what it means |
|---|---:|---|
| intended | 92 | it runs by design: input from outside, a missing parameter |
| a record's rule, nested | 41 | an index is valid because of how a record's fields relate, inside another record |
| restructure | 24 | an index read that a form without indexes avoids |
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

The next round worked from that table, changing only puritty. Three kinds
of change covered the eighteen.

**Restructure: take the index away.** When a long line is rewrapped after
a resize, puritty follows some positions in it through the rewrap, such
as the cursor's. It keeps two arrays that line up: `marks`, one entry per
position followed, and `found`, where each one landed. The old code
walked `found` with an index `k` and looked up the matching mark at the
same index:

```geng
-- before
    Array.indexedMap
        (\k f ->
            if Maybe.withDefault -1 (Array.get k marks) == -2 - j then
```

`Array.get k marks` returns a `Maybe`, because nothing tells the compiler
that `marks` is as long as `found`. So the code needs a fallback, `-1`,
for a mark that is not there. If the two arrays ever did differ in
length, it would carry on with a made-up mark.

The new code walks both arrays together:

```geng
-- after
    Array.map2
        (\m f ->
            if m == -2 - j then
```

`Array.map2` hands the function one element of each array at a time.
There is no index, so there is no `Maybe` and nothing to prove. One
caution: `Array.map2` stops at the shorter array, where the old code went
on with -1. In puritty the arrays always have the same length, so the
two do the same thing. In another program, the fallback might have been
doing work. Check why a fallback was there before you take it away.

**A rule on a constructor.** puritty decodes EUC-KR, a Korean encoding in
which a character is two bytes. The first, the *lead* byte, is between
0x81 and 0xFE, and together with the second byte it gives a place in a
conversion table. When a lead byte arrives, the decoder keeps it until
the second one comes. That kept value used to be any `Int`:

```geng
-- before
type Decoder
    = Decoder Int
```

so when the second byte came, the compiler could not know that the table
index computed from the lead was inside the table. The read had to be
`Maybe.withDefault 0 (Array.get … index)`. A bad lead would have read 0,
which the decoder shows as the replacement character, �, with nothing to
say the decoder itself had gone wrong.

The new type says what a kept lead can be:

```geng
-- after
type Decoder
    = Decoder Lead

type alias Lead =
    (lead : Int where lead == 0 || (0x81 <= lead && lead <= 0xFE))
```

`Lead` is a refined `Int`: 0 when no lead is waiting, otherwise a valid
lead byte. Every place that makes a `Decoder` must now prove its value is
one of those. The decoder keeps a byte as the lead only after testing
that it is between 0x81 and 0xFE, so the proof is the test that was
already there. In
return, the read can use the rule. With the lead in range and the second
byte tested as before, the index is provably inside the table, and the
read needs no fallback:

```geng
-- before
    Maybe.withDefault 0 (Array.get ((lead - 0x81) * 190 + byte - 0x41) index)

-- after
    Array.getProven ((lead - 0x81) * 190 + byte - 0x41) index
```

**A parameter.** Some helpers read at an index their callers had already
checked. Writing the rule on the helper's parameter moves the proof to the
callers, where the knowledge is:

```geng
dcsHeader : Builder UInt8 -> (i : Int where 0 <= i) -> Int -> Bool
trimmed : (cells : Array Int64) -> (n : Int where n <= Array.length cells) -> Array Int64
```

Inside `trimmed`, `n` is known to be no more than the array's length, so
its reads need no test. Each caller must now prove `n` is in range when
it calls. If one could not, that caller had a bug, and the build would
have said so.

"Can run" went from 192 to 174, and checks that hold from 46 to 101.
"Never run" did not move, on purpose: each change took the fallback away
rather than proving it unreachable. A fallback proved unreachable is still
in the code. A read with `getProven` has nothing to fall back to, and
every later build checks it again.

**Found:** `defaultColor` read the palette from its end for a negative
index, because `Array.get` counts a negative index from the end. It
answers 0 now.

## Round 4: `core` promises more

Some fallbacks were there only because `core` promised less than it
could. `Maybe.map` applies a function to the value inside a `Just`, and
leaves a `Nothing` alone. So its result is a `Just` exactly when its input
is. Its signature now says so:

```geng
map : (a -> b) -> (maybe : Maybe a) -> (r : Maybe b where Maybe.isJust r == Maybe.isJust maybe)
```

Before, after `x |> Maybe.map f`, the compiler knew nothing about the
result, even when it knew `x` was a `Just`, so a `Maybe.withDefault` after
it always looked as though it could run. With the contract, the fact
carries through the `map`.

The checker also learned to rule cases out before splitting them, which
let it finish questions it had given up on. All ten undecided fallbacks
were decided, and four fallbacks in all were shown never to run, this
one among them:

```text
Vt.Keys.functionKey 873:17 the default of Maybe.withDefault on line 873 is unreachable: undecided
Vt.Keys.functionKey 873:17 the default of Maybe.withDefault on line 873 is unreachable: proved
```

The other nine undecided were shown to run. They handle input a remote
program sends, which the reading had already called intended. That is why
"can run" went up, to 180. That is progress: a fallback the compiler
can't decide tells you nothing, and one it shows can run comes with the
values that reach it.

Not everything could get a contract. `String.split` cannot promise a
non-empty result, because `String.split "" ""` is `[]`, and there is not
yet a way to write "when the separator is not empty" as a condition.

## Round 5: rules on types

A terminal keeps a stack of keyboard modes that programs push and pop.
puritty stores it as an array of eight slots and the index of the top:

```geng
type alias Keys =
    { stack : Array Int, at : Int }
```

Reading the current mode reads `stack` at `at`. The report said:

```text
Vt.keyOptions 845:20 the default of Maybe.withDefault on line 845 is unreachable: refuted, with Array.length t.keys.stack = 0, t.keys.at = 0
```

The compiler found a case where the default runs: an empty stack. That
never happens, since puritty always makes the stack with eight slots and
keeps `at` inside it. But nothing in the types said so, and the read was
written with a fallback for a case that cannot occur.

Geng gained **type invariants**, so the rule could be written on the
type:

```geng
type alias Keys =
    { stack : Array Int
    , at : Int where 0 <= at && at < Array.length stack
    }
```

Now a `Keys` whose `at` is outside its stack cannot be made. The compiler
checks every place that builds one or changes `at`. In return, every read
of `stack` at `at` is proved, wherever the `Keys` is, so `getProven`
replaced the fallback.

Two of the places that change `at` failed the check. They moved it with
`modBy 8 (k.at + 1)` and `modBy 8 (k.at - 1)`: wrap around within eight
slots. That is right only while the stack has exactly eight slots, which
nothing guaranteed. The rule made the code say what it relied on, by
testing the real length:

```geng
at =
    if k.at + 1 < Array.length k.stack then
        k.at + 1

    else
        0
```

Inside the `then`, the compiler knows `k.at + 1` is below the length; in
the `else`, 0 is in range because the stack is not empty. Both branches
keep the rule.

The palette got the same treatment (`named : Array Int where
Array.length named == 16`). The compiler checked the invariants at 234
places across the terminal's updates.

## Round 6: every screen row, then every cell

Most of the 41 nested-record sites were reads of a row of the screen,
through the screen inside the terminal's state. Finding a row among a
screen's cells multiplies the row's number by the screen's width, as in
`y * cols`. Proving that read in range needs the checker to reason about
a product of two values, which it could not do. It learned to. puritty then wrote invariants on its screen and
terminal types and refined 36 functions that make screens. On 2026-11-09
every read of a screen row in the core was proved in range.

**Found:** the scrollback keeps a count of rows that have scrolled away,
which only grows. At 32 bits it would have wrapped after 2³¹ rows, about
an hour of `yes`, and a selection's position, which is counted from that
number, would have jumped. The count is capped now, and the cap is a rule
the compiler checks:

```geng
addRows : (a : Int where 0 <= a && a <= 536870912) -> (b : Int where 0 <= b) -> (r : Int where 0 <= r && r <= 536870912)
addRows a b =
    if b > 536870912 - a then
        536870912

    else
        a + b
```

The signature says both inputs are at most 2²⁹, and so is the result. In
the `else`, the compiler knows `b <= 536870912 - a`, so `a + b` is at most
the cap and cannot wrap. Note how the test is written: `b > cap - a`
rather than `a + b > cap`. The second form would compute the sum that
might wrap in order to ask whether it wraps.

The same day, `geng checks` was built as a command, and `core`'s
`Array.Builder.get` gained a contract like `Maybe.map`'s. Then every
cell read the compiler could see was proved, which took the core's
fallbacks that never run from 14 to 41.

**Found:** inserting at column 0 read cell −1, which `Builder.get` counts
from the end, so it looked at the buffer's last cell. That was harmless
there. Two binary searches took their midpoint as `(lo + hi) // 2`, which
can wrap; they use `lo + (hi - lo) // 2` now.

**What proofs did not buy: speed.** Reading cells with `getProven` skips
the bounds test, and was slower anyway: 90 ms where the checked read took
78 ms. It was not kept. Sums compiled without their wrap checks, as a
proof would allow, saved at most a few percent, and that was not built
either. The speed came from a new primitive instead, `Bytes.spanWithin`,
which took the same case to 59 ms. Here, proofs bought correctness, not
speed.

## Round 7: the whole program, and a guard

`geng checks Main` ran over all of puritty for the first time: the
application, the window, and the tmux and ssh packages. From here on the
report is the command's own output: a line of totals, then one line per
finding.

```text
1662 checks hold    42 fallbacks never run    740 can run    51 open
open         inWord:661        the product y * cols can wrap, when cols = 46341, y = 46341
open         menuPress:1402          the sum box.col + box.width can wrap, when box.col = -1, box.width = -2147483648, the value at 1396:54 = 0
```

46,341 squared is just past the largest `Int`. Nothing stops a screen
that large, as far as the compiler can see, because nothing said how
large a screen can be.

Each open check was settled where its values are made, not where they are
used. The renderer gets the window's size once, so that is where the
bound went: rows and columns are between 1 and 1,000. A row's start is
then computed in one place:

```geng
rowStart : (y : Int where 0 <= y && y < 1000) -> (cols : Int where 1 <= cols && cols <= 1000) -> (r : Int where 0 <= r && r <= 999000)
rowStart y cols =
    clamp 0 999000 (y * cols)
```

With `y` below 1,000 and `cols` at most 1,000, `y * cols` is at most
999,000, far from wrapping. The result's own range is in the signature,
so every caller that adds a column to it can prove its sum safe too.
`clamp` states that range in a form the checker reads directly.

The same approach settled the rest. A tmux pane's size is a type
invariant, 0 to 16,384 on each axis. Menu and view boxes take the
window's size. The report ended at 0 open:

```text
1891 checks hold    46 fallbacks never run    736 can run    0 open
```

**Found:** the renderer accepted rows down to `0 - s.hist`, so a negative
row could reach a buffer where `Builder.get` counts from the end. The X11
event decoder could read past a short buffer coming from C; a short or
broken buffer now ends the events.

Then the fallbacks that could run were recorded as a **baseline**, a list
of them by module and function. Run with `--baseline`, `geng checks`
fails when it finds a fallback that can run and is not on the list, or
any open check. That makes the report a guard: new code that adds a
silent `Maybe.withDefault 0` is caught.

## Round 8: the guard had not run

Two days of features later (tabs, most of the tmux integration, sounds),
the guard was run again, and it failed:

```text
1937 checks hold    46 fallbacks never run    1115 can run    7 open
open         drawListing:6995         the sum (l.selected - listHeight) + 1 can wrap, when Array.length rows = 0, h = 0, l.previewMode = 0, l.selected = 2147483647, listHeight = 0, …
```

It had not been run since round 7. The guard is a command, not a build
step, and nobody ran it. That lesson is about process, not Geng.

tmux's buffer and client lists keep the selected row as an index,
`selected`. The open check above says the compiler knew nothing about
it: it could even be the largest `Int`. The fix was a rule on the
listing's type:

```geng
, selected : Int where 0 <= selected && (selected == 0 || selected < Array.length rows)
```

Read it as: `selected` is never negative, and it is a real row, except
that it may be 0 when there are no rows at all. Writing it that way, and
not simply `selected < Array.length rows`, matters: an empty list is
normal, and the type has to allow it.

**Found:** writing that rule found a bug. When tmux's answer emptied the
list, the selection became `clamp 0 (Array.length rows - 1) l.selected`.
With no rows, that is `clamp 0 -1`, which is −1. The compiler refused it,
since −1 breaks the rule. It is 0 now.

`core` gained contracts on the way: `keepIf` keeps no more elements than
it is given, and the sorts keep the length. `Array.findIndex` came next,
because `Array.findFirst` cannot promise anything about the index it
returns.

## Round 9: every fallback that can run, read

The baseline now held 1,114 fallbacks that could run. A count that large
stops meaning anything. Most ran by design, but nothing in the code said
which ones, or why.

Geng gained a way to say it at the fallback itself, with the keyword
`meant` and a reason. Here is the function that reads a parameter of an
escape sequence, the kind a program sends to move the cursor:

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

The signature uses refinements throughout: the default is a 16-bit
value, and so is the result, which is at least 1 whenever the default
is. The `Nothing` branch runs when a program leaves the parameter out,
which the terminal standards allow, and its value is the default. `meant
Reason.NotSet default` has exactly the value `default`. The mark changes
nothing about what runs; it records that this fallback is intended, and
why.

The reason is a constructor of a type puritty declares, and its doc
comment is the why:

```geng
type Reason
    = {-| what the program in the terminal or the keyboard sent was short, malformed or not what was asked -}
      FromOutside
    | {-| a setting or option is not set, so its default applies -}
      NotSet
```

The report shows the mark with its words, beside the values that reach
the fallback:

```text
meant        param:90             the Nothing branch runs when Array.length params = 0, get's result at 79:10.Just.0 = 0, k = 0, meant: Vt.Reason.NotSet (a setting or option is not set, so its default applies)
```

**A mark is a claim, and the compiler tests it.** It never turns a check
off. If a later change means the marked fallback can never run, `geng
checks` fails and calls the mark stale. It also fails when a reason is
used by no mark. A `meant` that stops being true can't stay in the code
unnoticed.

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
- **11 were bugs** the fallback was hiding. Most were in the ssh client.
  Two examples:

```geng
-- a Port that is not a number, or is out of range, became 22
port = String.toInt (one "port" "22") |> Maybe.withDefault 22
```

`String.toInt` returns `Nothing` for text that is not a number, and the
default turned that into 22. A typo in `~/.ssh/config` connected to the
wrong port without a word.

```geng
-- channel data that did not read was dropped, and the output lost
Err _ ->
    st
```

This is the branch for a message from the server that could not be
decoded. It returned the connection's state unchanged, as if the message
had never come, so whatever output it carried was lost.

Others: `ServerAliveInterval 1m` silently became 0, since only bare
seconds were read. A `Match host` with no argument was matched against an
empty string. With no `HOME`, `~/.ssh/config` was read from `/`.

All of them were fixed, each against OpenSSH. puritty now reads `Port`,
`ServerAliveInterval` and `ServerAliveCountMax` the way OpenSSH does.
When OpenSSH would refuse a line, puritty leaves it out and shows the
problem. Tests check every such case against OpenSSH's own `ssh -G`. A
channel message that does not read now ends the session, as an
unreadable confirmation already did. After the fixes, the report reads:

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
| 8 | the tmux list selection | became −1 when the list emptied |
| 9 | the ssh client and its caller | eleven, listed above; one was harmless |

## What came back to Geng

Each round asked something of the language, and the language changed:

- **The report itself** (rounds 1, 6): "checks hold" in place of a
  percentage, every sum asked about, and `geng checks` as a command with
  a baseline.
- **`core` contracts** (rounds 4, 6, 8): `Maybe.map`,
  `Array.Builder.get`, `keepIf`, the sorts, `setMany`, `takeFirst`,
  `remove`, and the new `Array.findIndex`.
- **The checker** (rounds 4, 6): ruling cases out before splitting them,
  products of two values, and facts that reach a branch's value.
- **Type invariants** (round 5): a rule on a record type that holds
  wherever the type goes.
- **`meant`, and doc comments on variants** (round 9).

Some questions are still open. `String.split` makes no promise yet, and
a dozen of the fallbacks still on the list wait on it. A function passed
to a fold is not followed into. A local function's result is opaque to
the checker.

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
