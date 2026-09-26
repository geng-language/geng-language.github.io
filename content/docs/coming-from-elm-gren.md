+++
title = "Geng for Elm and Gren programmers"
description = "What is different in Geng if you already know Elm or Gren."
weight = 30
+++

> [!NOTE]
> Outline only. Many sections depend on design decisions still open;
> those are listed under "Open questions" at the end.

## Who this is for

- Readers fluent in Elm or Gren. Assumes the Elm Architecture, custom types and `Maybe`/`Result` are familiar.
- Structure: a summary table first, then one section per difference.

## At a glance

- Table: feature | Elm | Gren | Geng.
- Rows: collections, tuples, pattern matching syntax, record payloads, targets, entry point, effects/ports, FFI, package manager, operators.

## Where Geng runs

- Elm: browser. Gren: browser and Node. Geng: native, JS (browser and Node), WASM, BEAM, JVM.
- What this changes for the language: numbers, concurrency, platform APIs.
- Link to [Platforms and targets](@/docs/platforms.md).

## Syntax differences

- Differences from Gren (the direct parent).
- Differences from Elm, for Elm users who skipped Gren: `when ... is` instead of `case ... of`, `Array` instead of `List`, no tuples, no `::`.
- Link to [Keywords](@/reference/keywords.md) and [Operators](@/reference/operators.md).

## Numbers

- Elm's `Int` is a JS number underneath. What `Int` and `Float` mean in Geng on every target.
- Overflow and division by zero behaviour.

## Errors that cannot happen

- Things that can crash or misbehave in Elm/Gren and how Geng makes them expected: `Debug.todo`, stack overflow on deep recursion, comparing functions with `==`, out-of-memory, etc.
- Link to [Error handling](@/docs/error-handling.md).

## Programs, effects and I/O

- Gren's `Node.defineProgram` / `Browser` programs vs Geng's entry point.
- `Task`, `Cmd`, `Sub` in Geng.
- Command-line programs: arguments, environment, exit codes.

## Interop

- Elm ports / Gren ports and kernel code vs Geng's FFI.
- Link to [Interop](@/docs/interop.md).

## Tooling and packages

- Compiler CLI, project file, package registry compared with `elm`/`gren` tools.
- Link to [Tooling](@/reference/tooling.md).

## Porting code

- Checklist for porting a Gren package or program to Geng.
- Checklist for porting from Elm (larger: lists, tuples, `case`).

### Open questions

- What syntax changes (if any) does Geng make relative to Gren?
- Numeric types: fixed-width? arbitrary precision? same on all targets?
- Is `Debug` kept? Is `Debug.todo` removed?
- Are ports kept alongside FFI, or replaced?
- Does Geng use Gren's package format and registry, or its own?
