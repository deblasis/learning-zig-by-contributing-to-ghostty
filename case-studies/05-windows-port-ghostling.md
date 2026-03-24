# Windows Port for Ghostling

## Context

Ghostling is a single-file C demo that uses libghostty-vt and Raylib to build a minimal terminal emulator. It worked on Linux and macOS. I added Windows support using ConPTY.

The port had two parts: the C code (ConPTY, shell detection, threaded pipe reader) and the build system (CMake needed to know about the .lib import library on Windows).

## What I did

For the build: upstream ghostty PR 11756 added IMPORTED_IMPLIB but did not list the .lib as a custom command OUTPUT. Ninja failed. I first wrote a string(REPLACE) workaround that patched upstream's CMakeLists.txt at configure time. It worked but was ugly.

Then I realized I could fix it upstream. I created a branch in my ghostty fork with the one-line fix, pointed ghostling at the fork, and verified with Windows CI. Once confirmed, I opened PR 11794 on ghostty-org/ghostty. Mitchell merged it the same day.

For the C code: I added ConPTY support, a --shell flag with strcmp parsing, section banner comments for organization, and the full Windows platform code in main.c.

## What upstream did

Mitchell's review was two lines:

1. "We don't need this at the top." -- referring to the // Platform-independent headers banner
2. "let's just change this to argv[1] (no parsing)." -- referring to the --shell flag parser

His overall comment: "Not bad. I agree the ifdefs are pretty ugly. I'll have to think about it."

## What I learned

This is a toy project -- a tech demo. The right level of abstraction is different here than in production code. Section banners and flag parsers are fine in real apps but overshoot for a single-file demo meant to be read top to bottom.

The fork-first approach for the CMake fix worked perfectly. Instead of carrying a workaround, I proved the fix worked and proposed it upstream. It went from PR to merged in hours.

Key patterns:
- [code-style](../patterns/code-style.md) -- match abstraction to project scope
- [cmake](../patterns/cmake.md) -- Ninja needs all outputs listed explicitly

Reference: PR 11794 on ghostty-org/ghostty (cmake fix). PR 6 on ghostty-org/ghostling (Windows port).
