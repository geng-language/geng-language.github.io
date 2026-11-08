+++
title = "FAQ and design rationale"
description = "Why Geng is the way it is."
weight = 80
+++

## About the project

### Why fork Gren instead of contributing to it?

Geng changes things that every Gren program depends on: `Int` becomes exactly
32 bits, `==` becomes a class method, `Cmd` and `Sub` are removed, and
strings stop exposing UTF-16. Those changes are what make one program mean
the same thing on JavaScript, the BEAM and native code, and they are not
compatible with existing Gren code.

### Why not Haskell, OCaml, F#, Gleam or Roc?

Each of them is a good language. None of them gives both of the things Geng
is for: a small language whose programs cannot crash on a null, an
unhandled error or a missing case, and one program that means the same
thing on JavaScript, the BEAM and native code.

- **Haskell** is pure and checks a great deal, but it is a large language:
  higher-kinded types, type-level programming and many extensions. Code can
  read like a different language per author. Lazy evaluation also makes
  memory use hard to predict. Haskell does not run on the BEAM.
- **OCaml** and **F#** have exceptions and mutable variables, and F# has
  `null` from .NET, so the guarantees are conventions, not checks. Neither
  runs on the BEAM, and F# runs where .NET runs.
- **Gleam** is the closest relative: a typed language for the BEAM and
  JavaScript. It has no native target, and its `Int` is arbitrary-precision
  on the BEAM but a JavaScript number in the browser, so the same program
  can give different answers on its two platforms. `panic` and
  `let assert` end a program from ordinary code.
- **Roc** compiles to native code and WebAssembly, but not to the BEAM or
  JavaScript.

Geng starts from Gren, and Gren from Elm, because they already had the
small language and the clear compiler errors. What Geng adds is the
specification that makes the platforms agree: fixed integer sizes, its own
float printing and math functions, and one conformance suite for every
backend.

### What does "Geng" mean?

It is the Thai word เก่ง, which means "skilled" or "capable".
The G is hard, as in "Go", not soft, as in "Java".

### Is it ready for production?

No. See [Status and roadmap](@/docs/status.md).

## Language design

### Why classes, but no `Functor` or `Monad`?

Classes were added for specific problems: custom types as `Dict` keys,
`==` on functions being a compile error instead of a crash, and arithmetic
over ten numeric types without ten copies of every function. Classes are
limited to one type parameter and there are no higher-kinded types, which
keeps type checking decidable and error messages understandable, and keeps
code from turning into a different language per author.

### Why no mutation, even locally?

Because a value that can never change can be shared with another task,
process or thread without a lock or a copy. On the BEAM this is how the
runtime already works; on other platforms it removes data races by
construction.

When building a large array, an `Array.Builder` writes into one buffer in
place, but nothing can see that happen: the compiler checks that a builder
is used only once, so no one else holds it. See
[Builders](@/docs/type-system.md#builders-values-used-once).

### Why no exceptions?

An exception is control flow that does not appear in any type. In Geng, a
function that can fail returns a `Maybe`, a `Result` or a failing `Task`, and
the compiler makes the caller handle it. See
[Error handling](@/docs/error-handling.md).

### Why is `Int` 32 bits?

Because it has to behave the same everywhere, and 32 bits is cheap on every
platform: JavaScript has fast 32-bit integer arithmetic, and native code and
the JVM have it built in. A 64-bit `Int` would force JavaScript onto `BigInt`,
which is much slower. `Int64`, `UInt32` and `UInt64` are available when you
need them, and they map directly to protobuf's integer types.

### Why no tuples?

Gren removed them, and Geng follows. A record with named fields says more
than a tuple with positions.

### Why can't I define my own operators?

Custom operators are a common source of code that only its author can read.
The fixed set is enough for arithmetic, comparison, pipelines and
composition. See [Operators](@/reference/operators.md).

### Why is there no `exposing (..)`?

With `import A exposing (..)`, the names in scope depend on the version of
`A`. If `A` adds a function in a minor release whose name you also import
from somewhere else, your code stops compiling, even though neither you nor
`A`'s author did anything wrong. Listing names explicitly prevents this, and
tells readers where each name comes from.

`exposing (Type(..))` is still allowed, because a type's constructors can
only change in a breaking release anyway.

## Platforms

### Is the same program really identical on every platform?

That is the goal, and it is tested. Every backend runs the same conformance
suite, whose expected outputs are written by hand from the specification.
Where the platform would give different answers, Geng specifies the answer
itself: integers have fixed sizes and wrapping, floats follow IEEE 754 even
on the BEAM, `sin` and float printing are implemented in Geng, and Unicode
tables come from a fixed Unicode version. What does differ between platforms
is speed.

### Why the BEAM?

It was built to run systems that must not go down: millions of lightweight
processes, isolation between them, supervisors that restart what fails, and
upgrades of running code. It is the right platform for the coordination part
of a distributed system. Erlang and Elixir are dynamically typed; Geng offers
the same platform with a strict type system, and the same code can run
elsewhere.

### Which platform should I use?

See [Platforms and targets](@/docs/platforms.md).

## Practical

### How do I debug without print statements?

`Debug.log` prints a value and returns it, in development builds. `inspect`
turns any value into a string and can be used anywhere, including in
production code.

### Is there a REPL?

No. A REPL would have to pick one backend to evaluate on, and Geng has
several.

### How fast is Geng?

Native code is faster than Node.js on most of the workloads measured: about
1.5× on a `Dict` updated in a loop, 2.4× on counting words, and 3× on the
compiler's own formatter and C backend. Math functions such as `sin` and
`log` are 4× to 9× faster natively than on JavaScript.

On JavaScript, some operations are slower than in Gren, such as 64-bit
integers and math functions, because they give the same answer on every
platform.

### Can I use Geng with my existing codebase?

See [Interop](@/docs/interop.md).

### Is there editor support?

Not yet.

### Where is the standard library documentation?

Which packages form the standard library has not been decided. See
[Status and roadmap](@/docs/status.md#standard-library).
