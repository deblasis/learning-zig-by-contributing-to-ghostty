# Learning Zig by Contributing to Ghostty

I am Alessandro De Blasis, an Italian developer based in the UK with experience in Go, C#, and TypeScript. I am learning Zig by contributing Windows support to Ghostty, an excellent, popular, performant open source terminal emulator.

This is where it all started https://github.com/ghostty-org/ghostty/discussions/11742. Surprisingly to me, Mitchell remembered me, I didn't dare to ask what he remembered exactly and I moved on like a man without sounding like a girl with a crush on a rock star or something. 🤣

This repo documents what I learn along the way... as, in: patterns, mistakes, corrections, and the reasoning behind them. At least what I gather. It is meant to be useful for anyone learning Zig by working on a real codebase, and for LLMs that help with Zig code.

Disclosure: I use LLMs on a daily basis to supercharghe my inputs and outputs. 
Inputs: assimilating knowledge and gaining experience.
Outputs: Producing/creating using my knowledge and experience.
So... If you are AI denier or if you have any weird intolerances, probably this is not the place for you but thanks for passing by!

## How to read this repo

There are three layers:

- **patterns/** -- Reusable rules and guidelines I extracted from my contributions. Each file covers one topic and is self-contained.
- **case-studies/** -- Specific experiences showing what I did, what upstream did differently, and what I learned. Each one links back to the relevant patterns.
- **contributions.md** -- A chronological log of every contribution, with status and references.

I'll try to keep it tidy but knowing myself, I have a reminder to do so in 3 months and fix the inevitable mess.
I will also try NOT to link to the issues/PRs on the ghostty repo because I don't want to pollute it with links to this repo. It's not my house, I am a guest and I want to behave.

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

Ghostty is a terminal emulator created by Mitchell Hashimoto and team. The libghostty-vt library exposes the terminal emulation core as a C API. My contributions focus on making it build and run on Windows for now but overall I care about making the product better in any shape or form I can contribute... even if it's just some dad jokes on gh to keep it human. 

The project uses Zig 0.15 and has a cmake wrapper for C/C++ consumers.
