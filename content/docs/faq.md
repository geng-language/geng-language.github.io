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

*To be written.*

### What does "Geng" mean?

*To be written.*

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

On the BEAM, Geng is within 10% of Gleam on the benchmarks measured so far.
On JavaScript, some operations are slower than in Gren, such as 64-bit
integers and transcendental functions, because they give the same answer on
every platform.

### Can I use Geng with my existing codebase?

See [Interop](@/docs/interop.md).

### Is there editor support?

Not yet.

### Where is the standard library documentation?

Which packages form the standard library has not been decided. See
[Status and roadmap](@/docs/status.md#standard-library).
