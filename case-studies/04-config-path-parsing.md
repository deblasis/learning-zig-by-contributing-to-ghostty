# Fixing config path parsing for Windows

## Context

Ghostty's CommaSplitter treats backslash as an escape character. This breaks Windows paths like `C:\Users\foo` because `\U` is not a valid escape sequence. Also, the Theme parser mistakes the colon in drive letters (`C:\...`) for a light/dark theme pair separator.

## What I did

First attempt: I added `if (comptime native_os == .windows)` branches in every test that involved backslash escapes -- about 10 tests. Each test had a platform branch with different expectations. It worked but it was noisy and scattered.

After learning the comptime constant pattern from the superseded PRs, I reworked it:

1. Added a single constant at the top of CommaSplitter: `const escape_outside_quotes = builtin.os.tag != .windows;`
2. The `next()` function checks this constant -- one place where the behavior differs
3. Escape-specific tests skip on Windows with `SkipZigTest` instead of branching
4. Windows-specific tests are added separately at the end
5. The Theme parser skips colon detection at index 1 on Windows

## What upstream did

This is still open. The rework was self-correction based on patterns we learned from the superseded PRs, not upstream feedback.

## Before and after

First attempt (scattered branches in every test):
```zig
test "splitter 9" {
    var s: CommaSplitter = .init("\\x");
    if (comptime native_os == .windows) {
        try testing.expectEqualStrings("\\x", (try s.next()).?);
    } else {
        try testing.expectError(error.UnfinishedEscape, s.next());
    }
}
// ... repeated 10 times
```

Reworked (single constant, clean tests):
```zig
const escape_outside_quotes = builtin.os.tag != .windows;

// In next():
'\\' => {
    self.index += 1;
    if (comptime escape_outside_quotes) {
        last = .normal;
        continue :loop .escape;
    }
    continue :loop .normal;
},

// Escape tests skip cleanly:
test "splitter 9" {
    if (comptime !escape_outside_quotes) return error.SkipZigTest;
    // ... original test unchanged
}

// Windows tests are separate:
test "splitter: windows paths" {
    if (comptime escape_outside_quotes) return error.SkipZigTest;
    // ... windows-specific test
}
```

## What I learned

Platform behavior should live in a single constant, not scattered across every test. The constant pattern keeps the logic in one place and makes tests clean -- they either run or skip, no branching.

This was the first PR where I applied lessons from the superseded PRs before submitting. The first attempt used the old approach (scattered if-else). The rework used the new approach (comptime constant + SkipZigTest).

See [platform abstraction](../patterns/platform-abstraction.md).

PR 4 on deblasis/ghostty (open).
