+++
title = "Getting started"
description = "Install Geng and write your first program."
weight = 10
+++

> [!NOTE]
> Geng has not been released, and there is no installer yet. This page
> shows what a first program looks like with the compiler as it stands.
> See [Status and roadmap](@/docs/status.md).

## Installation

Not yet available.

## Your first program

A Geng application is a directory with a `geng.toml` file and a `src/`
directory. The manifest says which runtime the program is built for:

```toml
[application]
geng = "0.6.0 <= v < 0.7.0"
runtime = "node"
```

Create `src/Main.geng`:

```geng
module Main exposing (main)

import Console
import Task exposing (Task)


main : Task Never {}
main =
    Console.write "Hello, world!\n"
```

Then build and run it:

```sh
geng run Main
```

`main` is a `Task`: a description of work for the runtime to do. `Never` as
its error type means every error must be handled before it reaches `main`,
and `{}` means it produces no value. The program ends when `main` finishes.

## Running on the BEAM

The same program runs on the Erlang virtual machine. Change the manifest:

```toml
[application]
geng = "0.6.0 <= v < 0.7.0"
runtime = "beam"
target = "beam"
```

and run `geng run Main` again. On the BEAM, `geng make` builds an OTP
application and an escript.

## Next steps

- New to functional programming? Read
  [Functional programming for imperative programmers](@/docs/functional-programming.md).
- Coming from Elm or Gren? Read
  [Geng for Elm and Gren programmers](@/docs/coming-from-elm-gren.md).
