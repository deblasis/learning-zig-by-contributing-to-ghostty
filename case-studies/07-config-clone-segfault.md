# Tracing config.Config.test.clone segfault to CommaSplitter

## Context

After all MSVC build fixes were merged, the Windows test suite showed 49/51 steps passing, 2631/2654 tests, 23 skipped. The only remaining failure was `config.Config.test.clone can then change conditional state` -- a segfault (exit code 3) with "attempt to use null values".

The test creates temporary theme files, gets their realpath, and passes them to the config system as `--theme=light:{path},dark:{path}`. On Linux and Mac it passes fine.

## What I did

Traced the data flow from the test through the config loading:

1. The test uses `td.dir.realpath()` to get absolute paths. On Windows, these contain backslashes and drive letter colons: `C:\Users\...\theme_light`.

2. The theme value goes through `Theme.parseCLI()` which checks for colons to detect light/dark pairs. A Windows drive letter (`C:`) would trigger this check incorrectly.

3. The value also goes through `CommaSplitter` which treats backslash as an escape character. A Windows path like `C:\Users\...` hits `\U` which is not a valid escape sequence (`n`, `r`, `t`, `\`, `'`, `"`, `x`, `u` are the only valid ones).

This looked familiar -- it was exactly what PR 11782 (backslash paths in config value parsing) fixes. Cherry-picked the CommaSplitter fix from that branch and the test passed.

## What I learned

When a test segfaults on one platform but passes on others, the first question should be "what is different about the data on this platform?" not "what is wrong with the code?". In this case, the code was fine -- the data (file paths) had platform-specific characters (backslashes, colons) that hit edge cases in parsing.

Recognizing that we had already seen this problem before (PR 11782) saved significant debugging time. Keeping a mental map of open PRs and what they fix helps connect seemingly unrelated failures.

The CommaSplitter treating backslash as escape is correct behavior for configuration values (where `\n` means newline), but it breaks when those values contain file paths on Windows. The fix in PR 11782 disables escape parsing outside of quoted strings on Windows via a comptime constant -- the same pattern documented in [platform abstraction](../patterns/platform-abstraction.md).

Related patterns: [platform abstraction](../patterns/platform-abstraction.md), [testing](../patterns/testing.md)
