# Fixing C API crash on invalid enum

## Context

All `get` functions in the C API (build_info, cell, render, row, terminal) crashed when called with `.invalid` and a null output pointer. The functions use `inline else` on an enum switch to dispatch to a typed helper. When `.invalid` is used, `@ptrCast(@alignCast(out))` casts null to a non-optional pointer -- safety panic.

## What I did

I added an early return before the switch for `.invalid`, then marked it `unreachable` inside the switch:

```zig
if (data == .invalid) return .invalid_value;
return switch (data) {
    .invalid => unreachable,
    inline else => |comptime_data| getTyped(...),
};
```

I fixed this in 5 files.

## What upstream did

Mitchell added `.invalid` as a single switch arm:

```zig
return switch (data) {
    .invalid => .invalid_value,
    inline else => |comptime_data| getTyped(...),
};
```

One line instead of two statements. All dispatch logic in one place. He also fixed 6 files -- I missed osc.zig which had the same pattern.

## Before and after

My approach (2 statements, 5 files):
```zig
if (data == .invalid) return .invalid_value;
return switch (data) {
    .invalid => unreachable,
    inline else => ...
};
```

Upstream approach (1 arm, 6 files):
```zig
return switch (data) {
    .invalid => .invalid_value,
    inline else => ...
};
```

## What I learned

When handling a new enum variant in a switch, add it as a direct arm. Do not guard before the switch and mark unreachable inside it. Same effect, less code, one place to look.

When fixing a repeated pattern across files, grep the whole directory before declaring the fix complete. I missed one file out of six.

See [api design](../patterns/api-design.md) and [testing](../patterns/testing.md).

PR 2 on deblasis/ghostty (superseded). Upstream commit 58283528c on ghostty-org/ghostty.
