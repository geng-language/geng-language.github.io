+++
title = "Status and roadmap"
description = "What works today, what is planned, and what may change."
weight = 90
+++

## Current status

Geng is in development and has not been released. The language design is
settled. The compiler builds programs for JavaScript and for the Erlang
virtual machine (the BEAM), and the native backend is next. Nothing is
published yet: there is no installer and no first-party package you can
depend on.

Expect breaking changes until the first public release.

## What works

| Area | Status |
|---|---|
| Parser and type checker, including classes | Built |
| Typed intermediate representation (Core) shared by every backend | Built |
| JavaScript backend (browser and Node.js) | Built |
| BEAM backend, including OTP applications, supervision and releases | Built |
| Native backend (C, Linux first) | Next; an early prototype has run |
| WebAssembly (WasmGC) backend | Planned |
| JVM backend | Planned |
| [Refinement types](@/docs/type-system.md#refinement-types-planned) | Designed; `geng fmt` reads the syntax, the checker is not started |
| `geng fmt` | Built |
| `geng protobuf` | Built |
| Package manager (git-based, no registry) | Built; nothing published yet |
| `geng test` | Not yet; tests run as ordinary programs |
| REPL | None, by design |

Every backend is checked against the same conformance suite: a set of
programs whose expected output is written by hand from the specification.
Today every case in it runs on both JavaScript and the BEAM, and the few
places where the two differ are recorded with their reasons.

## Roadmap

In order:

1. **Native.** A C backend with the Boehm garbage collector, native
   executables on Linux, and shared libraries callable from C, Python and Go.
   macOS and Windows follow.
2. **Self-hosting.** The compiler's front end, today a fork of Gren's Haskell
   compiler, is rewritten in Geng.
3. **WebAssembly**, using WasmGC so the browser owns the heap.
4. **The JVM.**
5. **Before the public release**, a review of the `geng` command and of the
   standard library.

## Standard library

Geng has a small `core` library today, plus packages for Node.js, the BEAM
and a few other areas. Which packages make up the standard library has not
been decided. The goal is one large, consistent collection, as Go, Python and
Erlang/OTP have, rather than a small core and scattered packages. That review
comes before the public release, so there is no standard library
documentation yet.

## What may change

- **Structured concurrency.** A concurrency package, where every concurrent
  task runs inside a scope that waits for it, is designed but not built.
  Until it exists, `core` has an interim `Process.spawn`.
- **The command-line tool.** Command names and options are reviewed before
  release.
- **Class declarations.** Only `core` can declare classes and instances
  today. Opening this to all code is possible later.

## How to follow along

The [blog](@/blog/_index.md) has announcements.
