+++
title = "Affine Types"
description = "Array.Builder and the types that hold one: values used once, written in place, and what a function does with them."
weight = 44
+++

An affine value can be *used* only once. Today that means one type,
`Array.Builder`, and any type that holds one. A builder writes into its
buffer in place, and this rule is what makes that safe. Nothing else in Geng
is affine: `Array`, `String`, `Dict`, records of them and so on are shared
freely. (Geng may still write those in place when it can prove nothing else
holds them, but that is an optimization, not a rule.)

See [Builders](@/docs/type-system.md#builders-values-used-once) for a guided
introduction.

## Looking and using

You may **look at** a builder as often as you like, until you **use** it.

| Looking (any number of times) | Using (once) |
|---|---|
| `Builder.length b` | writing to it: `Builder.pushLast x b` |
| `Builder.get i b` | finishing it: `Builder.toArray b` |
| `Builder.slice from to b` | storing it in a record or custom type |
| passing it to a function that only looks | returning it |
| | passing it to a function that marks it `taken` |

A write gives back a new builder, and the next use must be of that one.
`Builder.copy` only looks, and gives back a second builder with its own
buffer.

| Refused | Compiles |
|---|---|
| `Builder.pushLast 2 b0` and `Builder.pushLast 3 b0` | `Builder.pushLast 2 (Builder.copy b0)` and `Builder.pushLast 3 b0` |
| `Builder.length b` after `Builder.toArray b` | `Builder.length b` before `Builder.toArray b` |
| `{ first = b, second = b }` | `{ first = Builder.copy b, second = b }` |
| | one use of `b` in each branch of an `if` or a `when` |

Each refused case is an `OWNERSHIP` error that points at both places and
suggests a fix. See [When it is refused](@/docs/type-system.md#when-it-is-refused)
for common cases.

## Order matters

Looking is only allowed *before* using, so order decides whether code
compiles. Geng evaluates left to right and top to bottom:

| Where | Order | Refused | Compiles |
|---|---|---|---|
| `let` | bindings in the order written | `arr = Builder.toArray b`<br>`n = Builder.length b` | `n = Builder.length b`<br>`arr = Builder.toArray b` |
| record | fields in the order written | `{ first = b, second = Builder.copy b }` | `{ first = Builder.copy b, second = b }` |
| call | arguments left to right, then the call | `pair (Builder.toArray b) (Builder.length b)` | `Builder.pushLast (Builder.length b) b` |
| `\|>` | `x \|> f y` is `f y x`: `x` goes in last | | `b \|> Builder.pushLast (Builder.length b)` |

## Functions: `taken`

A function argument that is a builder is only looked at, unless the signature
marks it `taken`. `taken` tells the caller "I use this up, so you cannot use it
again":

```geng
addOne : (taken b : Builder Int) -> Builder Int
addOne b =
    Builder.pushLast 1 b


countOf : Builder Int -> Int
countOf b =
    Builder.length b
```

A caller can call `countOf b` and still use `b` afterwards, but not after
`addOne b`. The signature must be honest: leaving out `taken` on `addOne` is an
error, and so is writing it on `countOf`.

- `taken` goes on the arguments of a top-level or `let` signature. It cannot
  go on the result, on a callback's argument, or on a type that is not
  affine, including a type variable.
- A function with no signature has its `taken` arguments worked out from its
  body.
- Whether a function keeps a callback it is given, and whether its result
  borrows from an argument, are always worked out from its body.

## Your own types

A record with a builder field is affine with no mark. Using the builder field
takes only that field; the other fields can still be read:

```geng
let
    arr = Builder.toArray doc.body
in
Array.pushLast (String.count doc.title) arr -- fine: title was not used
```

A custom type that holds a builder must say so with `@affine`:

```geng
@affine
type Document
    = Document { title : String, body : Builder Line }
```

`@affine` cannot go on a type alias, and an `@affine` type cannot also have
`@derive(…)`.

## A builder's length {#a-builders-length}

`Builder.length` is a measure, so a condition may name it, and
`Builder.getProven` reads an element with no `Maybe`:

```geng
getProven : (i : Int where 0 <= i && i < Array.Builder.length builder) -> (builder : Builder a) -> a
```

Each function in `Array.Builder` promises the length of the builder it
returns:

| Function | Length of the result |
|---|---|
| `empty n` | 0 (`n` is only the room set aside) |
| `repeat n x` | `n`, or 0 when `n <= 0` |
| `fromArray a` | `Array.length a` |
| `pushLast x b` | one more than `b` |
| `append a b` | `b`'s plus `Array.length a` |
| `set`, `update`, `fill`, `copyWithin`, `moveWithin`, `rotate` | the same as `b` |
| `swap` | the same as `b`, in the result's `builder` field |
| `toArray b` | the array's `Array.length` is `b`'s |

Reading the length is looking, so it must come before the builder is used:
to state the length of what `toArray` returned, name `Builder.length b`
before the call.

## Fine print

- "Once" means once on each path through the code, as with the `if` above.
- Some code reads a builder more than once without using it, and that is
  allowed: reading it inside the function that a fold calls, and reading an
  element of a builder of builders without taking it out.
- A function given to `Task.andThen`, `Task.onError` or `Task.map` may use a
  builder it captured, since a task runs that function at most once. Any
  other function that captures a builder uses it when the function is made.
- A generic function that keeps a value twice cannot be given a builder.
- You never write any of the points in this list. The compiler works them
  out, and `geng docs` lists them for each function.
- The rules are the same on every target: a program refused on one is
  refused on all. On JavaScript and native targets a builder is a buffer
  written in place; on the BEAM it is a persistent tree.

## In a package {#builders-in-a-package}

What a function does with its arguments, whether written (`taken`,
`@affine`) or worked out by the compiler, is part of its interface.
`geng docs` lists it for each exposed function, and `geng package diff`
compares it between versions. Each fact limits what a caller may do, so
gaining one is a major change and losing one is minor.
