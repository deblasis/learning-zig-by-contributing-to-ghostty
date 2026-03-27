Last verified: 2026-03-26

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

**When a test segfaults on one platform only, check the data first.**
Cross-platform test failures are often caused by platform-specific data (file paths, environment variables), not by broken code. On Windows, file paths contain backslashes and drive letter colons that can break parsers expecting Unix-style paths. Before debugging the logic, ask "what does the input look like on this platform?"

**"Works on my machine" applies even when testing on the target platform.**
Your local git config, env vars, or OS settings can mask bugs that CI exposes. The CRLF bug passed locally (autocrlf=input) but failed on CI (autocrlf=true). The fix is to make the repo self-describing (e.g. `.gitattributes * text=auto eol=lf`) so no machine's config matters. When `@embedFile` is used with line splitting at comptime, always trim `'\r'` -- the embedded file's line endings depend on git config at checkout time.

**Keep a mental map of open PRs and what they fix.**
When investigating a test failure, check if any open PR already addresses the root cause. Recognizing that a segfault in config.Config.test.clone was the same CommaSplitter backslash issue from an open PR saved hours of debugging. Cherry-pick and verify before duplicating work.

**Follow the project's directory structure, not your language's conventions.**
A C test file does not belong in `windows/Ghostty.Tests/` just because that is where .NET tests go. Ghostty has `test/` for test infrastructure. When placing a new test file, look at where existing tests live in the project, not where your IDE or language ecosystem would put them.

## Where I learned this

- [01-test-lib-vt-fix](../case-studies/01-test-lib-vt-fix.md) -- root cause vs gating, SkipZigTest
- [02-c-api-crash-fix](../case-studies/02-c-api-crash-fix.md) -- grep wider
- [03-ghostty-free](../case-studies/03-ghostty-free.md) -- always test C API functions
- [07-config-clone-segfault](../case-studies/07-config-clone-segfault.md) -- platform-specific data, connecting to open PRs
- [10-crlf-ci-failure](../case-studies/10-crlf-ci-failure.md) -- "works on my machine" with git autocrlf, @embedFile CRLF handling
- [12-dll-crt-review](../case-studies/12-dll-crt-review.md) -- test file placement follows project structure
