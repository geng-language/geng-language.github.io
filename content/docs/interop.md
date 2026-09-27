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

- There is one line per host language: `js`, `erlang`, and `c` (accepted, but
  there is no native backend yet).
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

## Checked before building

A package's `@extern` lines decide which targets it can serve. If an
application refers to a module whose externs have no implementation for the
application's target, the build stops before generating code and names the
declarations that are missing.

## Calling Geng from C, Python and Go

Designed, not yet built. A Geng package will compile to a shared library
with a generated C header:

- Every exported function returns a status code, and a failure inside Geng
  never aborts the host process.
- Numbers cross directly. Records, custom types and functions cross as
  opaque handles, so the host's garbage collector and Geng's never share
  memory.
- Only functions with concrete types can be exported. A generic
  `map : (a -> b) -> Array a -> Array b` cannot be called from C, so a
  library exports the specific versions it needs.
- The first version is single-threaded behind a lock, like Lua.
