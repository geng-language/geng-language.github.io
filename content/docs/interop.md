+++
title = "Interop"
description = "Calling host-platform code from Geng, and Geng from the host."
weight = 70
+++

Geng code reaches its host platform through `@extern` declarations. The
declaration is ordinary Geng with a real type. The implementation is an
ordinary file in the host's language.

## Declaring an extern

```geng
@extern(js, "Time", "now")
@extern(erlang, "time", "now")
nowWith : (Int64 -> posix) -> Task x posix
```

- There is one line per host language: `js`, `erlang` and `c`.
- For `js` and `erlang`, the line names a module and a function. For `c` it
  names a symbol.
- The type is checked like any other Geng type.

**An extern returns a `Task`**, because host code can do anything. A function
that is known to be pure can be declared with `@externPure` instead, which
makes it easy to find every such promise in a codebase:

```geng
@externPure(js, "Hash", "sha256")
@externPure(erlang, "hash", "sha256")
sha256 : Bytes -> Bytes
```

An extern may also have an ordinary Geng body. A backend with an
implementation for its language calls that; any other backend compiles the
Geng body.

## Checking what comes back

On the BEAM, every value coming from Erlang is checked against the declared
type before Geng code sees it. A term that does not match is reported at the
boundary, naming where it came in and what the type allows.

## JavaScript

An implementation file is a plain script at `src/Ext/<Module>.js`. A task
extern is a function that receives its arguments, then `succeed` and `fail`
callbacks. It completes once, and may return a function that cancels it:

```js
// src/Ext/Time.js
function now(millisToPosix, succeed, fail) {
  succeed(millisToPosix(BigInt(Date.now())));
}
```

Only JavaScript's own types cross: numbers, `BigInt`, strings, booleans,
`Uint8Array`, arrays of these, opaque handles and functions. To return a
Geng record or custom type, the extern takes a Geng function that builds it,
as `millisToPosix` does above.

## The BEAM

Geng values have fixed Erlang shapes, so Erlang code can use them directly:

| Geng | Erlang |
|---|---|
| record | map with atom keys |
| custom type | tagged tuple |
| `Array` | tuple |
| `Dict`, `Set` | map |

Beyond externs, the `github.com/geng-language/beam` package covers OTP: a
Geng module can be a `gen_server` callback module started from an ordinary
Erlang child specification, and Geng code can call, cast to and monitor
Erlang processes.

## C

A `c` line names a C function, implemented in a file under the package's
`src/Ext/`, such as `src/Ext/hash.c`. The build compiles each file with a
prototype generated from the Geng type, so a function with the wrong number
of arguments or the wrong number types does not compile.

Below that, C is trusted. Every Geng value that is not a number crosses as
`void *`, so passing one kind of value where another was meant compiles,
and can corrupt memory. Write a `c` implementation as carefully as any C.

A package or native application names the C libraries it links in its
`geng.toml`, and the build asks `pkg-config` for their flags:

```toml
[native]
pkg-config = ["xcb", "xcb-render"]
```

## Checked before building

A package's `@extern` lines decide which targets it can serve. If an
application refers to a module whose externs have no implementation for the
application's target, the build stops before generating code and names the
declarations that are missing.

## Calling Geng from C, Python and Go

A native build with `runtime = "host"` and no `main` is a shared library
with a generated C header. The functions one module exposes are its exports,
and C, Go and Python programs can load it:

- Every exported function returns a status code, and a failure inside Geng
  never aborts the host process. A crash in Geng code comes back as a
  status, with the reason readable from the library.
- Numbers cross directly. Records, custom types and functions cross as
  opaque handles, so the host's garbage collector and Geng's never share
  memory.
- Only functions with concrete types can be exported. A generic
  `map : (a -> b) -> Array a -> Array b` cannot be called from C, so a
  library exports the specific versions it needs.
- No `Task` crosses the boundary: exports are pure functions.
- Calls from any number of threads are taken one at a time, behind a lock,
  as Lua does.
- To load on other machines, build the library with the system's C
  compiler, so that it links only the system's C library.
