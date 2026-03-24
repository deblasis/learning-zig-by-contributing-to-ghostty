Last verified: 2026-03-24

# Testing

How to write and organize tests in the Ghostty codebase.

## Patterns

**Always add tests for C API functions.**
Every function exported through the C API needs tests in its module. Cover: normal use, null inputs, edge cases (zero-length allocations), and the null-allocator-falls-back-to-default path. Mitchell added 6 tests to our ghostty_free before merging. We had zero.

**Use error.SkipZigTest, never silent returns.**
When a test cannot run on a platform, return `error.SkipZigTest` so the test runner reports it as skipped. Never use a bare `return` -- that makes the test silently pass, which hides the fact that you are not testing anything.

Good: `if (builtin.os.tag == .windows) return error.SkipZigTest;`
Bad: `if (builtin.os.tag == .windows) return;`

**Grep wider when fixing a repeated pattern.**
When you find a bug pattern that appears in multiple files, search the whole directory (or codebase) before declaring the fix complete. We fixed a crash pattern in 5 files but missed osc.zig which had the same pattern. Upstream caught all 6.

**Fix the root cause, not the symptom.**
We gated failing imports behind artifact checks to make tests pass. Mitchell fixed the underlying modules so they work in all build configurations. Our approach lost test coverage. His preserved it. Always ask "why does this fail?" before asking "how do I skip it?"

## Where I learned this

- [01-test-lib-vt-fix](../case-studies/01-test-lib-vt-fix.md) -- root cause vs gating, SkipZigTest
- [02-c-api-crash-fix](../case-studies/02-c-api-crash-fix.md) -- grep wider
- [03-ghostty-free](../case-studies/03-ghostty-free.md) -- always test C API functions
