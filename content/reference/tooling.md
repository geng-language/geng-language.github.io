+++
title = "Tooling"
description = "The geng command, project files, modules and packages."
weight = 50
+++

> [!NOTE]
> Command names and options will be reviewed before the first release.

## The `geng` command

One program does everything:

| Command | What it does |
|---|---|
| `geng make <Module>` | compile a program whose `main` is in `<Module>` |
| `geng run <Module>` | compile and run it |
| `geng test` | build and run the project's tests, on its target |
| `geng release` | build an OTP release (BEAM only) |
| `geng fmt` | format source files |
| `geng protobuf` | generate Geng modules from `.proto` files |
| `geng docs` | a package's documentation |
| `geng init` | create a project |
| `geng package ...` | install, uninstall and list outdated dependencies; validate a package; compare two versions' interfaces (`diff`) and choose the next version number (`bump`) |

`geng make` takes `--optimize` for a release build, which also refuses any
use of `Debug`. On the native target, `--debug-info` keeps the generated C
and compiles it for gdb, perf and gcov.

There is no REPL.

## Project file: `geng.toml`

An application:

```toml
[application]
geng = "0.6.0 <= v < 0.7.0"
runtime = "node"

[dependencies]
"github.com/geng-language/url" = "6.0.0 <= v < 7.0.0"
```

A package:

```toml
[package]
name = "github.com/geng-language/url"
version = "6.0.0"
summary = "Create and parse URLs."
license = "BSD-3-Clause"
geng = "0.6.0 <= v < 0.7.0"

[platform]
runtime = "any"

[modules]
exposed = ["Url", "Url.Builder", "Url.Parser", "Url.Parser.Query"]
```

`runtime` and `target` are described in
[Platforms and targets](@/docs/platforms.md). Source files end in `.geng` and
live under `src/`.

## Packages

- **A package is identified by a URL path**, such as
  `github.com/geng-language/url` or `git.example.com/team/pkg`.
- **There is no registry.** Packages are fetched directly from git, so
  private packages on an internal server work the same way as public ones.
- **Versions are resolved by minimal version selection**, as in Go: the
  build uses the oldest version that satisfies every requirement.
- **`geng.lock` pins exact commits.** A build with a lock file does not
  contact git, and a tag that has moved since it was locked is refused.
- **`core` comes with the compiler** and is not listed as a dependency.
- A `[sources]` table can point a package at a local path or another git
  location, for development.

## Formatter

`geng fmt` formats every file in the project, `geng fmt <paths>` formats
those files, and `geng fmt --check` changes nothing and fails if any file is
unformatted, for CI.

It has no options. Where a construct can be written on one line or across
several (a record, an array, a list of arguments), it keeps your choice, and
normalizes everything else. The compiler does not require formatted code.

## Protobuf

`geng protobuf` reads proto3 files and generates one Geng module per
`.proto` file, with a decoder, an encoder and a default value for each
message. Protobuf's integer types map directly to Geng's: `int32` is `Int`,
`int64` is `Int64`, `uint32` is `UInt32`, and `uint64` is `UInt64`. The naming
rules are part of the language specification, so every project generates the
same names.

## Editor support

Not yet available.
