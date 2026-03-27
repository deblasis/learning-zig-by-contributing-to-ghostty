# CRLF Line Endings Break Comptime Parsing on Windows CI

## Context

PR 11839 added a `test-windows` CI job running `zig build -Dapp-runtime=none test` on `windows-2025`. The job failed with a compile error:

```
src\font\sprite\draw\symbols_for_legacy_computing_supplement.zig:116:56: error: no field named '...'
```

The error message looked broken -- the field name was invisible because it was `\r` (carriage return), which doesn't render in log output.

Tests passed locally on Windows. Classic "works on my machine".

## What I did

### Investigation

The file uses `@embedFile("octants.txt")` at comptime and splits by `'\n'`. Each line like `BLOCK OCTANT-1235` is parsed by iterating characters after `-` and using `@field(current, &.{c})` to set struct fields named `"1"` through `"8"`.

On the CI runner, `actions/checkout` defaults to `core.autocrlf = true`, which converts LF to CRLF in text files. The split-by-`'\n'` leaves `'\r'` at the end of each line. When the parser hits `'\r'`, it tries `@field(current, &.{'\r'})` -- a field that doesn't exist.

Locally, my git config has `core.autocrlf = input` (LF on commit, no conversion on checkout), so `octants.txt` stays LF and parsing works.

### Root cause vs symptom

I considered three fixes:

1. **CI-only fix:** Set `core.autocrlf: false` in the checkout step. Quick but fragile -- every new Windows CI job would need the same setting.
2. **Code fix:** Trim `'\r'` from parsed lines. Defensive, follows existing precedent in `x11_color.zig` which already does this for `rgb.txt`.
3. **Repo-level fix:** Add `* text=auto eol=lf` to `.gitattributes`. Makes the repo self-describing -- no developer or CI runner needs special git config.

Applied all three: code fix for defense-in-depth, `.gitattributes` as the root cause fix. The CI config became unnecessary but is implied by `.gitattributes`.

### Checking for collateral damage

Scanned the entire repo for committed CRLF files. Found exactly 2: `dist/windows/ghostty.manifest` and `dist/windows/ghostty.rc`. Both are Windows resource files. Tested that they work fine with LF (the Windows resource compiler and XML parser don't care about line endings). Ran `zig build -Dapp-runtime=none test` after normalization -- all tests passed.

Also searched for other `@embedFile` + line-split patterns. Found `x11_color.zig` already has the CRLF trim (good precedent), and `list_themes.zig` uses `splitAny` which is inherently safe.

## What upstream did

N/A -- this PR is still open.

## Before and after

Before (comptime crash on CI):
```zig
const data = @embedFile("octants.txt");
var it = std.mem.splitScalar(u8, data, '\n');
while (it.next()) |line| {
    if (line.len == 0 or line[0] == '#') continue;
    // line ends with '\r' on Windows CI -> @field crashes
```

After:
```zig
while (it.next()) |raw_line| {
    // Trim \r so this works with both LF and CRLF line endings,
    // since git may convert octants.txt to CRLF on Windows checkouts.
    const line = if (raw_line.len > 0 and raw_line[raw_line.len - 1] == '\r')
        raw_line[0 .. raw_line.len - 1]
    else
        raw_line;
    if (line.len == 0 or line[0] == '#') continue;
```

`.gitattributes` addition:
```
* text=auto eol=lf
```

## What I learned

"Works on my machine" applies even when you ARE testing on the target platform. My local `core.autocrlf=input` saved me from the bug. CI's default `autocrlf=true` exposed it. The fix is to make the repo self-describing with `.gitattributes` so no machine's config matters.

When `@embedFile` is used with line splitting at comptime, always handle CRLF. The embedded file's line endings depend on git config at checkout time, not on what was committed.

Related patterns: [cmake](../patterns/cmake.md), [testing](../patterns/testing.md)
