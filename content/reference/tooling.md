+++
title = "Tooling"
description = "The geng command, project files, modules and packages."
weight = 50
+++

> [!NOTE]
> Outline only.

## The `geng` command

- One subsection per subcommand: synopsis, options, examples.
- Likely: `init`, `build`, `run`, `test`, `format`, `package ...`, `docs`.
- Selecting a target (link to [Platforms](@/docs/platforms.md)).

## Project file

- File name and full format reference: name, version, source directories, dependencies, targets.
- Application vs package projects.

## Modules

- File layout and how module names map to paths.
- `exposing`, imports, aliases.
- Cyclic imports (not allowed) and how to break them.

## Packages

- Adding, removing and upgrading dependencies.
- Semantic versioning enforced by the compiler (if kept from Elm/Gren).
- Publishing a package.

## Formatter

- What it does, and that there are no options.

## Test runner

- Test file layout, running tests, running on multiple targets.

## Editor support

- Language server, syntax highlighting, supported editors.

### Open questions

- Command and subcommand names, project file name and format.
