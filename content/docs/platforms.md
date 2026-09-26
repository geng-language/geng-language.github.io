+++
title = "Platforms and targets"
description = "Building one Geng program for native, JavaScript, WASM, BEAM and the JVM."
weight = 60
+++

> [!NOTE]
> Outline only.

## Overview

- The five targets and what each is good for (one line each).
- Same source, same behaviour: what Geng guarantees across targets.

## Choosing a target

- Table: target | startup time | throughput | deployment artifact | typical use.

## Building for each target

- Native Linux executable.
- JavaScript for the browser.
- JavaScript for Node.js.
- WASM (browser and standalone runtimes).
- Erlang BEAM.
- JVM.
- For each: the build command, the output, how to run it.

## What differs between targets

- Numbers (sizes, precision), if anything differs.
- Available platform APIs (no filesystem in the browser, etc.) and how the compiler reports using one on the wrong target.
- Concurrency model.
- Performance characteristics.

## Writing portable code

- Keeping platform-specific code at the edges.
- Structuring a project with shared core modules and per-target entry points.

## Building for several targets at once

- Project configuration for multiple targets.
- Testing on every target.

### Open questions

- How does a program declare which targets it supports?
- Is platform-specific code selected by module, by package, or by a build flag?
