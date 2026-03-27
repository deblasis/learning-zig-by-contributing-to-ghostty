# DLL CRT init -- code review fixes

## Context
PR 11856 on ghostty-org/ghostty added DllMain CRT initialization for the Windows DLL build. Mitchell reviewed and requested two changes.

## What I did
Used `b.graph.arena` as the allocator for `std.zig.WindowsSdk.find()` and `std.fmt.allocPrint()` in GhosttyLib.zig. Placed the C regression test in `windows/Ghostty.Tests/` following .NET test project conventions.

## What upstream did
Mitchell flagged `b.graph.arena` -- it should be `b.allocator`. Both work today because `b.allocator` wraps the arena, but `b.graph.arena` reaches into internal fields of `std.Build`. The public API is `b.allocator`.

He also asked for the test file to go in `test/windows/` with a short README. The `windows/` directory is for the Windows app itself. Test infrastructure belongs under `test/`, regardless of what platform it targets.

## What I learned
Two patterns:

1. **Use the public API, not the internals** -- `b.graph.arena` is how `b.allocator` is implemented today. That does not make it the right thing to use. If the Zig build system refactors its internals, code using `b.graph.arena` breaks while code using `b.allocator` keeps working. Same principle as using an interface instead of a concrete type. See [code-style](../patterns/code-style.md).

2. **Follow the project's directory structure, not your language's conventions** -- The `.NET` test project layout (`windows/Ghostty.Tests/`) does not apply to a C file in a Zig project. Ghostty has `test/` for test infrastructure. When in doubt, look at where existing tests live. See [testing](../patterns/testing.md).

Reference: PR 11856 on ghostty-org/ghostty.
