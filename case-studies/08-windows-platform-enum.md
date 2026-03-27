# Adding Windows Platform to the Embedded Runtime

## Context

Ghostty's embedded apprt exposes a C API for GUI wrappers. The platform is modeled as a Zig tagged union (`Platform`) with variants for macOS and iOS. I needed to add a Windows variant so the upcoming C# WinUI 3 app can pass a window handle when creating surfaces.

The change touched three files: `ghostty.h` (C header), `embedded.zig` (Zig platform union), and -- as I discovered -- `Metal.zig` (Metal renderer).

## What I did

Added `GHOSTTY_PLATFORM_WINDOWS` to the C enum, a `ghostty_platform_windows_s` struct with a `void* hwnd` field, and the corresponding Zig `Platform.Windows` type. The Windows type uses the same comptime conditional pattern as macOS and iOS:

```zig
pub const Windows = if (builtin.target.os.tag == .windows) struct {
    hwnd: std.os.windows.HWND,
} else void;
```

On non-Windows builds, `Windows` is `void` -- zero-cost, no fields, no code generated.

I ran `zig build test` on Windows. Green. Ran on Linux. Green. Pushed confidently.

Then I tested on Mac. Build failed.

## What went wrong

`Metal.zig:99` has an exhaustive switch on the platform union:

```zig
.view = switch (opts.rt_surface.platform) {
    .macos => |v| v.nsview,
    .ios => |v| v.uiview,
},
```

Zig requires exhaustive switches -- every variant must be handled. My new `.windows` variant was not there. But I never saw the error on Windows or Linux because Metal.zig is only compiled when `renderer = .metal`, which only happens on Darwin targets.

## The fix

Added `.windows => unreachable` to the switch. Metal only runs on Apple platforms, so reaching the Windows arm is impossible at runtime. `unreachable` is the right tool here -- it tells both the compiler and the reader "this cannot happen".

```zig
.view = switch (opts.rt_surface.platform) {
    .macos => |v| v.nsview,
    .ios => |v| v.uiview,
    .windows => unreachable,
},
```

## What I learned

**Each platform compiles different code paths, so a change that builds on one platform can break another.** Zig's exhaustive switches are the safety net -- they force you to handle every variant. But the compiler only checks code that actually gets compiled for the current target. Metal.zig is dead code on Windows and Linux. The only way to catch the missing arm was to build on Mac.

This applies to more than just renderers. Font backends (CoreText vs FreeType vs DirectWrite), app runtimes (GTK vs embedded), and any platform-gated code all have this property. When you add a new variant to a tagged union, you have to test on every platform that compiles code switching on it.

See [platform abstraction](../patterns/platform-abstraction.md) for the general pattern.

PR 18 on deblasis/ghostty (draft).
