Last verified: 2026-03-25

# Comptime

How comptime evaluation interacts with code generation, and what goes wrong when you hit limits.

## Patterns

**Comptime branch quota exhaustion can silently break runtime behavior.**
When an `inline for` loop uses `comptime` string comparisons to decide whether to emit runtime code (like `set.remove` or a function call), exceeding the branch quota does not always produce a compile error. Instead, the matching branches are silently not generated. The test compiles and runs, but the runtime behavior is wrong -- it looks like missing data, not a budget problem. Always check the branch quota when comptime loops over large declaration sets produce unexpected runtime results.

**MSVC translate-c output is much larger than Linux/Mac.**
On MSVC, `<sys/types.h>` and other system headers pull in Windows SDK declarations. A typical translate-c output for ghostty.h is ~360KB with ~2173 declarations on MSVC vs ~112KB with ~1502 on Linux/Mac. Any code that iterates over `@typeInfo(c).@"struct".decls` with `inline for` needs a higher branch quota on MSVC. The default 1000 is never enough. Even 1M can be too low for nested loops (enum fields x declarations x string comparison length).

**Prefer @hasDecl over iterating declarations when you know the name.**
When you know the exact declaration name you are looking for, use `@hasDecl` for O(1) lookup instead of `inline for` over `@typeInfo(module).@"struct".decls`. The nested loop approach generates comptime branches proportional to the total number of declarations -- which varies wildly across platforms. Direct lookup is platform-independent and eliminates the branch quota problem entirely.

**Comptime string construction needs arrays, not slices.**
You cannot use `++` to concatenate slices at comptime -- it only works on arrays. To build a comptime string from parts, use a fixed-size buffer (`[N]u8`), fill it with `@memcpy` and byte-by-byte loops, and return a pointer to the array (`*const [N]u8`). The pointer coerces to `[]const u8` where needed. Also, `std.ascii.upperString` returns a slice so you cannot use it directly for concatenation -- use `std.ascii.toUpper` per byte instead.

**Debug comptime issues with isolated POC tests.**
When a comptime-dependent test fails in a way that does not make sense (the data is there but the code does not see it), write small tests that exercise each step of the failing function in isolation. Check module access, declaration count, `@hasDecl`, direct `@field` access, and `inline for` iteration separately. The step that fails (or hits a branch quota error) pinpoints the real cause. In our case, 7 POC tests narrowed the problem from "all 34 constants missing" to "comptime branch quota exceeded in inline for".

## Where I learned this

- [06-comptime-branch-quota](../case-studies/06-comptime-branch-quota.md) -- MouseShape test failure on MSVC
