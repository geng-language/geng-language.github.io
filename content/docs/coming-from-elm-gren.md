+++
title = "Geng for Elm and Gren programmers"
description = "What is different in Geng if you already know Elm or Gren."
weight = 30
+++

Geng is a fork of Gren, and the syntax is still Gren's. If you know Gren, most
Geng code will look familiar. The changes are mostly underneath: numbers,
strings, equality and effects now mean the same thing on every platform Geng
runs on, instead of whatever JavaScript does.

If you know Elm but not Gren, read [From Elm](#from-elm) first.

## At a glance

| | Elm | Gren | Geng |
|---|---|---|---|
| Runs on | browser | browser, Node.js | browser, Node.js, BEAM, native; WasmGC and JVM planned |
| Collections | `List` | `Array` | `Array` |
| Tuples | yes | no | no |
| Pattern matching | `case ... of` | `when ... is` | `when ... is` |
| `Int` | JS number | JS number | exactly 32 bits, wrapping |
| Other number types | none | none | `Int64`, `UInt32`, `UInt64`, `Float32`, and 8/16-bit types |
| `number`, `comparable` | built-in supertypes | built-in supertypes | classes: `Num`, `Ord`, and others |
| `Dict` keys | `comparable` only | `comparable` only | any `Ord` type, including your own |
| `==` on functions | runtime crash | runtime crash | compile error |
| Debug printing | `Debug.toString` | `Debug.toString` | `inspect`, allowed everywhere |
| `main` | `Browser.*` program | `Node.defineProgram` etc. | `main : Task Never {}` |
| `Cmd`, `Sub`, ports | yes | yes | removed |
| Host interop | ports | ports, kernel code | typed `@extern` declarations |
| Manifest | `elm.json` | `gren.json` | `geng.toml` |
| Packages from | registry | registry | git, identified by URL |
| File extension | `.elm` | `.gren` | `.geng` |

Existing Gren packages do not compile unchanged. Compatibility ends at the
fork.

## New syntax

Everything else is Gren's syntax. The additions:

- **`class` and `instance`** declarations. For now only first-party packages
  may write them.
- **Constraints**: `Eq a =>`, `Num a =>` and so on, in type signatures.
- **Attributes**: `@derive`, `@extern`, `@externPure`, `@capability` and
  `@affine`.
- **Refinements** and **`moved`** in signatures: `(i : Int where 0 <= i)`
  and `(moved b : Builder a)`. See
  [Geng's type system](@/docs/type-system.md).
- **Expression annotations**: `(e : T)` gives an expression a type in place.
  This is how a literal gets a type other than its default: `(42 : Int64)`.

See [Keywords](@/reference/keywords.md).

## Numbers

In Gren, `Int` is documented as 32 bits but is a JavaScript double in
practice. In Geng, `Int` is **exactly 32 bits on every platform**, and
overflow wraps.

For other sizes there are `Int64`, `UInt32`, `UInt64` and `Float32`, each with
its own module, plus `Int8`, `UInt8`, `Int16` and `UInt16` for binary formats.

Literals have a default type: `42` is an `Int` and `1.5` is a `Float`. For
another type, annotate it, or let the type signature decide:

```geng
big : Int64
big =
    5000000000                  -- fine: the signature makes it an Int64

later =
    (5000000000 : Int64)        -- the same, written in place
```

An integer literal that does not fit its type is a compile error.

Arithmetic never crashes and never produces a value outside its type:

- `x // 0` is `0`, and `remainderBy 0 n` is `n`.
- Shift counts past the width are clamped.
- Lossy conversions saturate.
- Floats are IEEE 754 everywhere. `nan == nan` is `True` and NaN sorts last,
  so floats are safe to use as `Dict` keys.

`String.fromFloat` prints the same text on every platform, and whole numbers
keep a decimal point: `String.fromFloat 123` is `"123.0"`.

`Time.Posix` is built on `Int64` milliseconds, because milliseconds since 1970
do not fit in 32 bits.

## Classes replace `number`, `comparable` and `appendable`

Gren's special supertypes are gone. In their place are ordinary classes:

| Gren | Geng |
|---|---|
| `number` | `Num` (and `Integral`, `Fractional`, `Bits`) |
| `comparable` | `Ord` |
| `appendable` | `Appendable` |
| `==` for any type | `Eq` |

```geng
total : Num a => Array a -> a
total values =
    Array.foldl (+) 0 values
```

You rarely write instances. The compiler derives `Eq`, `Ord` and `Inspect`
structurally for every type whose constructors are exposed. An abstract type
asks for them with `@derive`:

```geng
@derive(Eq, Ord, Inspect)
type Sku
    = Sku { vendor : String, code : UInt32 }
```

What you get from this:

- **Custom types as `Dict` and `Set` keys.** `Dict Sku Int` works.
- **`==` on functions is a compile error**, not a runtime crash.

Geng deliberately stops there. Classes take one type parameter, and there are
no higher-kinded types, so there is no `Functor` or `Monad`.

One rule to know: a value without a type annotation does not stay
polymorphic over a numeric type. `zero = 0` takes the type of its first use;
write `zero : Num a => a` to keep it general.

## `inspect` replaces `Debug.toString`

`inspect : Inspect a => a -> String` turns any value into a string. Unlike
`Debug.toString`, it is allowed in packages and in optimized builds, and its
output is the same on every platform:

```geng
inspect { name = "Ana", age = 36 }
-- "{ age = 36, name = \"Ana\" }"      (fields in alphabetical order)

inspect (1250000 : Int64)
-- "(1250000 : Int64)"
```

`Debug.log` still exists, has the type `Inspect a => String -> a -> a`, and
is for development builds only.

## Strings and `Char`

- `String` is opaque. The `*Units` functions (`unitLength`, `sliceUnits` and
  so on) are gone, and every index is a codepoint index.
- `compare` on strings is codepoint order. For characters outside the Basic
  Multilingual Plane this differs from JavaScript's order.
- `Char` is a Unicode codepoint, not a one- or two-unit string.
  `Char.fromCode` returns a `Maybe Char`.
- Case mapping, whitespace and grapheme clusters come from Geng's own Unicode
  tables, so they give the same answer on every platform.
- `String.Regex` has left `core`.

## Programs and effects

This is the largest change for application code.

- **`main : Task Never {}`** is the only kind of `main`, on every platform.
  There is no `Node.defineProgram`, no `Init`, no `update` loop that the
  runtime drives. The program runs `main` and ends when it finishes.
- **`Cmd`, `Sub`, ports and effect managers are gone.** Everything effectful
  is a `Task`.
- **`Source`** replaces subscriptions. A `Source msg` is a mailbox: something
  outside the program puts messages in, and a `Task` loop takes them out with
  `Source.next`. Timers and listeners take the `Source` to deliver into:

  ```geng
  Time.every : Float -> Source msg -> (Posix -> msg) -> Task x Timer
  ```

- **`Console.write` and `Console.writeErr`** are in `core`.
- **Capabilities.** Tokens such as `FileSystem.Permission` can only be created
  by the application, so a dependency cannot touch the file system unless you
  pass it one.
- **New `Task` functions**: `Task.bracket` and `Task.finally` guarantee
  cleanup on success, failure and cancellation; `Task.race` runs tasks until
  the first finishes and cancels the rest; `Task.parallel` runs tasks on
  multiple cores where the platform has them.

The Elm Architecture is not built into the language. It is planned as an
ordinary library built on `Source`, for the browser package.

## Interop

Gren's kernel JavaScript and its ports are replaced by typed `@extern`
declarations. The declaration is Geng, with a real type; the implementation is
a plain file in the host language.

```geng
@extern(js, "Time", "now")
nowWith : (Int64 -> posix) -> Task x posix
```

See [Interop](@/docs/interop.md).

## Packages and tooling

- The manifest is `geng.toml`. Source files end in `.geng`.
- A package is identified by a URL path such as
  `github.com/geng-language/url`. There is no registry: packages are fetched
  from git, and `geng.lock` pins exact commits.
- `core` comes with the compiler and is not listed as a dependency.
- `geng fmt` is part of the compiler and has no options.
- There is no REPL.

See [Tooling](@/reference/tooling.md).

## From Elm

If you are coming straight from Elm, these Gren changes apply too:

- `List` is replaced by `Array`, and `[ 1, 2, 3 ]` is an array. There is no
  `::`.
- There are no tuples. Use a record: `{ x = 1, y = 2 }`.
- `case x of` is written `when x is`.
- A custom type's constructor takes at most one argument. Use a record for
  more: `Rectangle { width : Float, height : Float }`.
- Names cannot shadow other names in scope.
