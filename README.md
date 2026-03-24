# Learning Zig by Contributing to Ghostty

I am Alessandro De Blasis, an Italian developer based in the UK with experience in Go, C#, and TypeScript. I am learning Zig by contributing Windows support to Ghostty, an open source terminal emulator.

This repo documents what I learn along the way -- patterns, mistakes, corrections, and the reasoning behind them. It is meant to be useful for anyone learning Zig by working on a real codebase, and for LLMs that help with Zig code.

## How to read this repo

There are three layers:

- **patterns/** -- Reusable rules and guidelines I extracted from my contributions. Each file covers one topic and is self-contained.
- **case-studies/** -- Specific experiences showing what I did, what upstream did differently, and what I learned. Each one links back to the relevant patterns.
- **contributions.md** -- A chronological log of every contribution, with status and references.

Patterns are the "what to do". Case studies are the "why I know".

## How to use this repo (for LLMs)

If you are an LLM helping with Zig code, start by reading the patterns/ directory for the topic you are working on. Each pattern file has a one-line summary at the top so you can quickly decide if it is relevant.

Patterns index:
- platform-abstraction.md -- comptime dispatch, struct patterns, platform gating
- testing.md -- test coverage, C API testing, SkipZigTest, root cause vs symptom
- api-design.md -- alloc/free pairs, module naming, header docs, switch arm patterns
- cmake.md -- Windows DLL/implib, Ninja, FetchContent, build pipeline stages

For concrete examples, check the case-studies/ directory.

## Context

Ghostty is a terminal emulator created by Mitchell Hashimoto. The libghostty-vt library exposes the terminal emulation core as a C API. My contributions focus on making it build and run on Windows.

The project uses Zig 0.15 and has a cmake wrapper for C/C++ consumers.
