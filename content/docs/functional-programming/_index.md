+++
title = "Functional programming for imperative programmers"
description = "A walk from C, Go, Python or Rust to pure functional programming in Geng, in five parts."
sort_by = "weight"
weight = 20

[extra]
# Where the series sits among the Docs pages in the sidebar.
sidebar_before = "docs/type-system.md"
+++

This series is for programmers who know an imperative language, such as C,
Go, Python or Rust, and have never used a pure functional one. By the end you
will be able to read and write ordinary Geng, and you will know why it is
written the way it is.

It is a long read, in five parts. Each part builds on the one before, so read
them in order.

## How to read it

Most examples are in **Python**, since nearly everyone can read it. Many come
in three versions:

1. Python as most people write it.
2. The same Python, written in a functional style. Python can do almost
   everything in this series; it just doesn't insist.
3. The Geng version. Geng *does* insist, and that is what lets its compiler
   check so much.

The middle step is the important one. It shows that each idea is something
you can already do, not a new kind of magic.

Notes for Go, C and Rust programmers appear in short asides. Rust programmers
already know a good part of this, and the asides say what to skim.

## The running example

Across the series we build one small program: an **expense report**. It reads
a file like this:

```text
date,category,amount,note
2026-11-02,food,12.50,lunch with the team
2026-11-03,travel,48.00,train to Boston
2026-11-03,food,abc,coffee
2026-11-04,office,23.99,printer paper
```

and prints a total for each category, along with the lines it could not read:

```text
food      $12.50
office    $23.99
travel    $48.00
total     $84.49

Skipped line 4: "abc" is not an amount
```

Each part adds a piece: a record for one expense, totals, a type for
categories, parsing that reports bad lines instead of crashing, and finally
reading the file and printing the report.

## The parts
