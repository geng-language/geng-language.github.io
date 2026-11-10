+++
title = "Status and roadmap"
description = "What works today, what is planned, and what may change."
weight = 90
+++

## Current status

Geng is in development and has not been released. The language design is
settled. The compiler builds programs for JavaScript, for the Erlang virtual
machine (the BEAM), and as native Linux executables and shared libraries.
Nothing is published yet: there is no installer and no first-party package
you can depend on.

The first release will be an alpha, published together with a large native
application written in Geng. Being an alpha, its standard library, command
names and options may still change.

## What works

| Area | Status |
|---|---|
| Parser and type checker, including classes | Built |
| Typed intermediate representation (Core) shared by every backend | Built |
| JavaScript backend (browser and Node.js) | Built |
| BEAM backend, including OTP applications, supervision and releases | Built |
| Native backend: Linux executables, and shared libraries called from C, Go and Python | Built, for Linux on x86-64 |
| Native on Windows and macOS | Planned, Windows first |
| WebAssembly (WasmGC) backend | Planned |
| JVM backend | Planned |
| [Refinement types](@/docs/type-system.md#refinement-types) | Built, on every target; `geng package` gives a changed refinement its semver magnitude |
| [Affine builders](@/docs/type-system.md#builders-values-used-once) | Built: checked on every build and every target; `geng docs` and `geng package` report what each function does with its arguments |
| `geng fmt` | Built |
| `geng test` | Built, on every target |
| `geng protobuf` | Built |
| Package manager (git-based, no registry) | Built; nothing published yet |
| REPL | None, by design |

Every backend is checked against the same conformance suite: a set of
programs whose expected output is written by hand from the specification.
Today every case in it runs on JavaScript, the BEAM and native code, and the
few places where they differ are recorded with their reasons.

## Roadmap

In order:

1. **Self-hosting and Windows.** The compiler's front end, today a fork of
   Gren's Haskell compiler, is rewritten in Geng, and native code gains a
   Windows target, so that the compiler itself runs on Windows.
2. **A second large application:** a web application with a Geng server on
   the BEAM and a Geng front end in the browser, to find what those two
   targets lack.
3. **WebAssembly**, using WasmGC so the browser owns the heap.
4. **The JVM.**
5. **Before the public release**, a review of the `geng` command and of the
   standard library, and release notes for each target.

The alpha does not wait for the end of this list. It comes when the first
application is ready to publish.

## Standard library

Geng has a small `core` library today, plus packages for Node.js, the BEAM,
the operating system (`os`, for native programs) and a few other areas.
Which packages make up the standard library has not been decided. The goal
is one large, consistent collection, as Go, Python and Erlang/OTP have,
rather than a small core and scattered packages. That review comes before
the public release, so there is no standard library documentation yet.

## What may change

- **Structured concurrency.** A concurrency package, where every concurrent
  task runs inside a scope that waits for it, is designed but not built.
  Until it exists, `core` has an interim `Process.spawn`.
- **The command-line tool.** Command names and options are reviewed before
  release.
- **Refinement types.** A record's rule is written on its fields or over the
  whole record, and is a type invariant that goes anywhere a type does; a
  refined type alias cannot take a value as a parameter. Only `Int`s,
  `Bool`s, a record's fields of those types and a few sizes can be refined. Patterns
  are facts, a condition can say whether a `Maybe` is a `Just`
  (`Maybe.isJust`), and a program can define measures of its own
  (`@measure`), but not yet recursive ones, such as a tree's size; those
  come next, then `modBy` and refinements inside type arguments. `geng
  package diff` does not yet see a change to a measure's body.
- **Affine types.** An `@affine` type cannot have `@derive(…)`, and whether
  `Array.map` should hand each builder on instead of lending it is still
  open.
- **Class declarations.** Only `core` can declare classes and instances
  today. Opening this to all code is possible later.
