+++
title = "Platforms and targets"
description = "Building one Geng program for JavaScript, the BEAM, native code, WebAssembly and the JVM."
weight = 60
+++

Geng compiles one language to several platforms, and a program means the
same thing on each. `Int` is 32 bits everywhere, `sin 1.0` returns the same
bits everywhere, and the same conformance suite checks every backend.

## The platforms

| Platform | Status | Good for |
|---|---|---|
| JavaScript: browser and Node.js | built | web front ends, command-line tools, anything that runs where JavaScript runs |
| BEAM (the Erlang virtual machine) | built | long-running servers, many concurrent connections, supervision, uptime |
| Native: executables and shared libraries | built, Linux on x86-64 | speed; embedding Geng logic in C, Python or Go programs |
| WebAssembly (WasmGC) | planned | the browser, with the browser's own garbage collector |
| JVM | planned | interoperability with Java code |

Native code runs on Linux today. Windows comes next, then macOS.

## Target and runtime

A build is described by two settings in `geng.toml`:

- **`target`** is the code the compiler produces: `js`, `beam` or
  `native`, and later WebAssembly and JVM code.
- **`runtime`** is what the program can use when it runs: `browser`, `node`,
  `beam`, `os` for a native program, or `host` for a native library that
  another program loads.

They are separate because they vary independently. The browser and Node.js
share the JavaScript target but offer different capabilities.

Runtimes form a hierarchy. `any` covers everything, and native's `os` has
`linux`, `darwin` and `windows` beneath it for packages that need a
particular operating system:

```
any ─┬─ browser
     ├─ node
     ├─ beam
     ├─ os ───┬─ linux
     │        ├─ darwin
     │        └─ windows
     └─ host
```

A package declares the most general runtime it works on. An application can
use a package if the application's runtime is at or below the package's. A
package with `runtime = "any"` works everywhere.

## Building for each platform

### Node.js

```toml
[application]
geng = "0.6.0 <= v < 0.7.0"
runtime = "node"
```

`geng make Main` produces a JavaScript file; `geng run Main` builds it and
runs it with `node`.

### The BEAM

```toml
[application]
geng = "0.6.0 <= v < 0.7.0"
runtime = "beam"
target = "beam"
```

`geng make Main` produces an OTP application directory and an escript;
`geng run Main` runs the escript. `geng release` builds an OTP release,
including the Erlang runtime and the files needed to upgrade a running node.
A build with no `main` is an OTP application that Erlang code calls into.

The `github.com/geng-language/beam` package gives Geng code access to OTP:
`gen_server` callbacks written in Geng, processes, monitors, registered
names, timers and supervision trees.

### Native

```toml
[application]
geng = "0.6.0 <= v < 0.7.0"
runtime = "os"
target = "native"

[dependencies]
"github.com/geng-language/os" = "0.1.0 <= v < 0.2.0"
```

Geng writes C and compiles it with the C compiler on your `PATH`: clang by
default, or the one the `GENG_CC` environment variable names. `geng make
Main` produces an executable, and `geng run Main` builds it and runs it.
The `os` package gives a native program files, directories, processes, the
environment and paths. Every string the operating system hands back can also
be read as raw bytes, for systems whose text is not UTF-8.

`--debug-info` keeps the generated C and compiles it with `-g`, and every
build maps the C back to Geng source lines, so gdb, perf and gcov report
Geng files and lines.

Today, native means Linux on x86-64, and an executable runs on the kind of
machine that built it. The toolchain cannot build for another machine yet.

### A native library

With `runtime = "host"` and no `main`, `geng make` builds a shared library
and a C header instead of an executable. The functions that one module
exposes become the library's exports. See
[Calling Geng from C, Python and Go](@/docs/interop.md#calling-geng-from-c-python-and-go).

### WebAssembly and JVM

Not built yet. See [Status and roadmap](@/docs/status.md).

## What differs between platforms

Values behave identically, but how fast they are depends on the platform:

| | JavaScript | BEAM | Native |
|---|---|---|---|
| `Int` | fast | a little slower (masked to 32 bits) | machine integer |
| `Int64` | much slower (`BigInt`) | fast | machine integer |
| `Float` | fast | slower: infinity and NaN need a tagged value | machine double |
| `sin`, `log`, ... | 1× to 6× the host library | 16× to 78× the host library | 4× to 9× faster than on JavaScript |
| Deep tail calls between functions | local groups only | all | all |
| `Task.parallel` | one thread | all cores | one thread at first |

`Task.concurrent` interleaves tasks on one scheduler on every platform.
`Task.parallel` asks for real parallelism and gets it where the platform has
it, which today means the BEAM.

## Writing portable code

- Keep code that needs a specific platform in its own modules or packages,
  and keep the rest `runtime = "any"`.
- A package that uses `@extern` declarations only works on the targets it has
  implementations for. The compiler checks this before generating any code,
  and names the declarations that are missing.
