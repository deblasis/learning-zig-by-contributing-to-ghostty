Last verified: 2026-03-24

# Platform Abstraction

How to write platform-specific code in Zig without scattering conditionals everywhere.

## Patterns

**Use struct-based comptime dispatch, not if-else branches inside functions.**
Scattering `if (comptime builtin.os.tag == .windows)` inside function bodies makes code hard to read and extend. Use a top-level comptime switch that selects between platform-specific structs instead.
Each platform's code lives in its own struct, fully isolated. Adding a third platform means adding one struct, not hunting through every function.

Example from Ghostty's page.zig:
```zig
const PageAlloc = switch (builtin.os.tag) {
    .windows => AllocWindows,
    else => AllocPosix,
};
```

**Use comptime constants for simple boolean platform differences.**
When the platform difference is just "this feature is on or off", a single constant at the top of the file is cleaner than branching in every test. Tests and logic both reference the constant.

Example from our CommaSplitter fix:
```zig
const escape_outside_quotes = builtin.os.tag != .windows;
```

**Fix the root cause before gating with platform checks.**
When tests fail on a platform, the first question should be "can I make this work?" not "can I skip it?". Gating with `if (comptime ...)` hides the problem and loses test coverage. Only skip when the test genuinely cannot run on that platform.

## Where I learned this

- [01-test-lib-vt-fix](../case-studies/01-test-lib-vt-fix.md) -- struct dispatch and root cause vs gating
- [04-config-path-parsing](../case-studies/04-config-path-parsing.md) -- comptime constants
