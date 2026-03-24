# Fixing test-lib-vt compilation

## Context

The `zig build test-lib-vt` target was broken on all three platforms (macOS, Linux, Windows). On macOS it produced 33 compilation errors from transitive imports. On Linux it crashed the compiler. On Windows it had 25 compilation errors. The problem was that test blocks in the lib-vt build were pulling in modules that only exist in the full Ghostty build -- things like ghostty.h, oniguruma, freetype, harfbuzz.

## What I did

I gated `refAllDecls` and test imports behind `artifact == .ghostty` checks so they only run in the full build. I also removed the `lib/main.zig` import from lib_vt.zig and replaced `refAllDecls(input)` with `_ = input`. For mouse.zig and osc.zig, I used `if (comptime build_options.artifact != .ghostty) return;` to skip tests.

For page.zig (the Windows VirtualAlloc fix), I added `backingAlloc`/`backingFree` standalone functions with `if (comptime builtin.os.tag == .windows)` branches inside.

## What upstream did

Mitchell fixed the root cause instead of hiding it. He made the modules themselves safe to import in all build configurations, so no gating was needed. All imports stayed unconditional. Test coverage was preserved in the lib-vt build.

For tests that genuinely cannot run (like ghostty.h validation), he used `return error.SkipZigTest` so they show up as "skipped" in the test runner output.

For page.zig, he used a struct-based comptime dispatch pattern:

## Before and after

My approach (gating):
```zig
// lib_vt.zig
if (comptime terminal_options.artifact == .ghostty) {
    _ = key_event;
    _ = key_encode;
}
```

Upstream approach (fix the modules):
```zig
// All imports unconditional, modules fixed to not pull in heavy deps
_ = key_event;
_ = key_encode;
```

My page.zig approach (if-else inside functions):
```zig
fn backingAlloc() {
    if (comptime builtin.os.tag == .windows) {
        // VirtualAlloc path
    } else {
        // mmap path
    }
}
```

Upstream page.zig approach (struct dispatch):
```zig
const PageAlloc = switch (builtin.os.tag) {
    .windows => AllocWindows,
    else => AllocPosix,
};
```

## What I learned

Fix the root cause, do not hide the symptom. Gating tests behind artifact checks is a last resort, not a first move. When tests fail because of transitive dependencies, the right question is "can I make the import work?" not "can I skip it?"

For platform-specific code, use struct-based comptime dispatch instead of if-else branches inside functions. It is more idiomatic Zig and easier to extend.

Use `error.SkipZigTest` when a test genuinely cannot run. Never use a bare return.

See [platform abstraction](../patterns/platform-abstraction.md) and [testing](../patterns/testing.md).

PR 1 on deblasis/ghostty (superseded).
