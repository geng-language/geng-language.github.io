+++
title = "Types"
description = "Built-in types, records, custom types, and type inference."
weight = 40
+++

> [!NOTE]
> Outline only.

## Built-in types

- `Int`, `Float`: sizes and behaviour (overflow, NaN, infinity).
- `Bool`, `Char`, `String`.
- `Array a`, `Maybe a`, `Result e a`.
- `Dict k v`, `Set a` (if in core).
- Unit / empty record.

## Records

- Record types, construction, field access, update syntax.
- Extensible records `{ a | name : String }`.
- Records as named arguments.

## Custom types

- Constructors with and without payloads; record payloads.
- Type parameters.
- Recursive types.
- Opaque types via `exposing`.

## Type aliases

- What an alias is (and is not: no new type).
- Record aliases double as constructor functions.

## Function types

- `a -> b -> c` and currying.

## Type variables and constraints

- Ordinary type variables.
- Constrained variables: `number`, `comparable`, `appendable`, `compappend`.
- Why there are no user-defined constraints (typeclasses).

## Type inference

- What the compiler infers, when annotations are required, and why top-level annotations are recommended.

## Equality and comparison

- What `==` and `<` work on; what they reject.

### Open questions

- Numeric types: any additions (e.g. fixed-width integers, `Bytes`)?
- Are the constrained variables the same as Gren's?
