+++
title = "Interop"
description = "Calling host-platform code from Geng, and Geng from the host."
weight = 70
+++

> [!NOTE]
> Outline only.

## Why interop needs care

- Foreign code can crash, mutate, or return anything. How Geng keeps its guarantees at the boundary.

## The general model

- Declaring a foreign function in Geng.
- Converting values across the boundary (and validating them).
- How foreign failures become `Result`/`Task` errors.

## Per target

- Native: calling C libraries.
- JavaScript: calling JS in the browser and Node.
- WASM: imports and exports.
- BEAM: calling Erlang/Elixir; being called from them.
- JVM: calling Java libraries; being called from Java/Kotlin.

## Calling Geng from other languages

- Exposing Geng functions as a library on each target.

## Writing a portable package with foreign code

- One Geng API with per-target implementations.

### Open questions

- Is this FFI, ports, or both?
- Can application code use FFI, or only packages (like Elm's kernel restriction)?
