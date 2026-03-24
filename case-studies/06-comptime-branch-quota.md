# Comptime branch quota breaks MouseShape test on MSVC

## Context

After fixing all compilation errors on Windows MSVC (ssize_t, linkLibCpp, freetype enum signedness), the full test suite ran for the first time: 2630/2654 tests passed, 1 failed. The failing test was `ghostty.h MouseShape` in src/terminal/mouse.zig.

The test calls `checkGhosttyHEnum(Shape, "GHOSTTY_MOUSE_SHAPE_")` which iterates over all declarations in the translate-c output of ghostty.h to verify that every Zig enum variant has a matching C constant with the correct value.

The error message said all 34 GHOSTTY_MOUSE_SHAPE_* constants were "missing" -- but they were clearly present in the translate-c .zig output file. The constants had the right names, the right type (c_int), and the right values. Other enum checks (like GHOSTTY_ACTION_*, GHOSTTY_SPLIT_DIRECTION_*) passed fine.

## What I did

Listed 5 hypotheses:
1. `@typeInfo` declaration truncation
2. Comptime branch quota exceeded
3. String comparison bug on MSVC target
4. Declaration ordering + early termination
5. `@field` access failure

Then wrote 7 POC tests in enum.zig, each isolating one step of checkGhosttyHEnum:

- POC1-2: Module access and declaration count -- passed
- POC3: `@hasDecl(c, "GHOSTTY_MOUSE_SHAPE_DEFAULT")` -- passed
- POC4: Direct value read via `c.GHOSTTY_MOUSE_SHAPE_DEFAULT` -- passed
- POC5: `inline for` over decls with `comptime std.mem.eql` -- **compile error: "evaluation exceeded 1000 backwards branches"**
- POC6: Same with `@setEvalBranchQuota(10_000_000)` -- passed
- POC7: Full checkGhosttyHEnum clone with 10M quota -- passed, all 34 constants found

POC5 was the breakthrough. The default branch quota (1000) immediately hit the limit. The existing `@setEvalBranchQuota(1000000)` in checkGhosttyHEnum was enough for Linux/Mac but not for MSVC.

## The math

The nested loop in checkGhosttyHEnum does: enum_fields x c_decls x string_comparison_length.

On Linux/Mac: 34 fields x 1502 decls x ~20 chars = ~1M branches. Just within the 1M quota.
On MSVC: 34 fields x 2173 decls x ~20 chars = ~1.5M branches. Over the 1M quota.

The MSVC translate-c output is ~3x larger because `<sys/types.h>` pulls in Windows SDK headers (BaseTsd.h, etc.), adding ~670 extra declarations.

## The fix

One line: change `@setEvalBranchQuota(1000000)` to `@setEvalBranchQuota(10_000_000)` in src/lib/enum.zig.

## What I learned

The scariest part was that the branch quota exhaustion did not produce a compile error. The test compiled, linked, and ran. But the comptime loop silently stopped emitting matching branches. The `set.remove` calls were never generated, so at runtime the set reported all entries as missing.

This happens because the `inline for` with `comptime` conditions decides at compile time whether to emit each loop body. When the quota runs out mid-iteration, the remaining iterations produce no code. There is no error -- the compiler just stops evaluating comptime branches.

The takeaway: if a comptime-dependent test produces impossible runtime results (the data exists but the code cannot see it), check the branch quota first. And when targeting MSVC, always assume translate-c modules will be larger than on other platforms.

The POC test approach -- isolating each step of a complex function into its own test -- was the key debugging technique. Without it, we could have spent hours staring at the translate-c output wondering why perfectly good constants were invisible.

Related patterns: [comptime](../patterns/comptime.md)
