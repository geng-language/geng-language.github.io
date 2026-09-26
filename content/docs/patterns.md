+++
title = "Common patterns"
description = "Short recipes for everyday tasks: I/O, processes, pipelines, errors."
weight = 50
+++

> [!NOTE]
> Outline only. Each recipe will be a short problem statement, a complete
> example, and a few lines of explanation. Full programs live in the
> examples repository; recipes link to them where relevant.
> If this page gets long, it can become a `docs/patterns/` section with one page per group.

## Program structure

- The smallest complete program.
- Reading command-line arguments.
- Parsing arguments into a record (with errors for bad input).
- Reading environment variables.
- Exiting with a particular exit code.
- Printing to stdout vs stderr.

## Files and I/O

- Reading a whole file as a string.
- Writing a file.
- Reading a file line by line.
- Reading from stdin until EOF.
- Listing a directory.
- Handling "file not found" and permission errors.

## Subprocesses

- Running a command and capturing its output.
- Checking a command's exit code.
- Passing input to a command's stdin.
- Running several commands in sequence, stopping at the first failure.

## Sequencing effects

- Doing one task after another with `Task.andThen`.
- Running tasks and collecting their results.
- Turning a `Result` into a `Task`.
- Handling a task's error and continuing.

## Pipelines and composition

- When to use `|>` (data flowing through several steps).
- When to use `<|` (avoiding parentheses around the last argument).
- When *not* to use them (a single call; mixing both in one expression).
- `>>` and `<<` for building functions, and when a lambda is clearer.
- Argument order: why functions take the "data" argument last.

## Working with data

- Transforming an array: `map`, `filter`, `foldl`.
- Grouping items into a `Dict`.
- Removing duplicates with a `Set`.
- Updating a nested record field.
- Sorting by a field.
- Replacing tuples with small records.

## Custom types

- Modelling states so impossible states cannot be represented.
- Opaque types: hiding a constructor to enforce invariants.
- Phantom types (brief).

## Text

- Splitting, joining, trimming.
- Converting between strings and numbers (and handling failure).
- Building output with `String.join` instead of repeated `++`.

## JSON

- Decoding JSON into a record.
- Encoding a record as JSON.
- Decoding optional and nullable fields.

## Networking

- Making an HTTP GET request.
- Posting JSON.

## Time and randomness

- Getting the current time.
- Generating random numbers (and why it is an effect).

## Testing

- Writing a unit test.
- Testing a function that returns `Result`.

### Open questions

- Which of these depend on packages not yet chosen for the standard library? Those recipes can wait or name the package explicitly.
