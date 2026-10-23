+++
title = "Compiler messages"
description = "How to read Geng's error messages, and a catalog of common ones."
weight = 60
+++

> [!NOTE]
> Outline only.

## Reading a message

- Anatomy of a message: title, location, the offending code, explanation, hint.
- Errors vs warnings (if Geng has warnings).
- Machine-readable output for editors and CI.

## Catalog

- One entry per message: the message title, a minimal program that triggers it,
  why it happens, and how to fix it.
- Grouped by phase:
  - Parsing (indentation, missing `else`, unbalanced brackets).
  - Names and imports (unknown name, ambiguous import, module not found, import cycle).
  - Types (type mismatch, infinite type, missing annotation where required).
  - Classes (no instance, e.g. `==` on functions; ambiguous numeric type).
  - Pattern matching (missing cases, redundant cases).
  - Records (missing field, unknown field, update of a non-record).
  - Targets and runtimes (`EXTERNS DO NOT SERVE THE TARGET`, `INCOMPATIBLE RUNTIME`, `NO BACKEND FOR THE TARGET`).
  - Refinements: writing one (`BAD REFINEMENT`, `MISPLACED REFINEMENT`,
    `NONLINEAR REFINEMENT`, `UNKNOWN NAME IN REFINEMENT`, `UNUSED NAME`) and
    proving one (`CANNOT SHOW A REFINEMENT`, `CANNOT DECIDE A REFINEMENT`,
    `ARITHMETIC MAY WRAP`). The proving errors list each fact the compiler
    had and values that break the condition; see
    [When a refinement does not hold](@/docs/type-system.md#when-a-refinement-does-not-hold).
  - Capabilities (`CAPABILITY IN A DEPENDENCY`).
  - Refused Gren constructs (`port module`, `effect module`).
  - Packages (dependency conflicts, invalid project file, `CORE IS NOT A DEPENDENCY`).

## Coming from other languages

- Common first-week mistakes from Python, Go and C++ users, and the messages they produce.

### Open questions

- Does each message have a stable code (e.g. `G0123`) that entries can be keyed by?
