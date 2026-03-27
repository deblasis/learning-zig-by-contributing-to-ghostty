# DLL CRT Linking on Windows MSVC

## Context

Building ghostty.dll (the full libghostty shared library) on Windows with `zig build -Dapp-runtime=none -Demit-exe=false` failed with 11 undefined symbol errors:

```
lld-link: undefined symbol: __vcrt_initialize
  referenced by msvcrt.lib(utility.obj)
lld-link: undefined symbol: __acrt_initialize
  referenced by msvcrt.lib(utility.obj)
...
```

These symbols are CRT initialization functions. They are called during DLL loading to set up the C runtime. The static library built fine because static libs do not resolve symbols at link time -- the consumer resolves them at final link.

## What I did

### First attempt: link the dynamic import libs

Added `linkSystemLibrary("vcruntime")` and `linkSystemLibrary("ucrt")` in SharedDeps.zig. Two problems:

1. `ucrt.lib` was not found. Zig's library search paths include the MSVC lib dir and the Windows SDK `um\x64` dir, but UCRT lives in `ucrt\x64`. Different subdirectory.
2. Even after fixing the path, the same 11 errors persisted. The dynamic import libraries (`vcruntime.lib`, `ucrt.lib`) export runtime functions like memset and printf. They do NOT contain `__vcrt_initialize` -- that is internal bootstrap code.

### Second attempt: link the static CRT libs

Switched to `libvcruntime` and `libucrt` (the `lib`-prefixed versions are the static CRT libraries). These contain the initialization routines. This is actually how MSVC works too -- even when using the dynamic CRT (`/MD` flag), the DLL startup code is statically linked from these libraries. Every Windows DLL does this.

### UCRT path detection

Used `std.zig.WindowsSdk.find()` to detect the Windows 10 SDK installation at build time. Constructed the UCRT library path from the detected SDK path and version, with arch awareness for x86/x64/ARM64.

The fix went in `GhosttyLib.zig`'s `initShared` function (not SharedDeps.zig) since this is specifically a shared library linking issue. Static lib builds are unaffected.

### Why this is a Zig bug

Zig's `linkLibC()` on MSVC links `msvcrt.lib` (the dynamic CRT import lib) but does not add the companion static bootstrap libraries that `msvcrt.lib` depends on. For executables this works because the compiler internally handles the CRT entry point. For DLLs, the `_DllMainCRTStartup` entry point in `msvcrt.lib` calls `__vcrt_initialize` and `__acrt_initialize`, which must come from `libvcruntime.lib` and `libucrt.lib`.

There are several closed issues on the Zig repo about this (5748, 5842) and one unmerged PR (5870). The problem was partially fixed years ago but this specific case slipped through. No open issue tracks it.

## What upstream did

N/A -- this has not been submitted yet. The fix lives in my local branch for now.

## Before and after

Before (linker errors):
```
error: lld-link: undefined symbol: __vcrt_initialize
error: lld-link: undefined symbol: __acrt_initialize
error: lld-link: undefined symbol: __vcrt_uninitialize
...11 total errors
```

After (GhosttyLib.zig initShared, added after deps.add):
```zig
if (deps.config.target.result.os.tag == .windows and
    deps.config.target.result.abi == .msvc)
{
    lib.linkSystemLibrary("libvcruntime");

    const arch = deps.config.target.result.cpu.arch;
    const sdk = std.zig.WindowsSdk.find(b.allocator, arch) catch null;
    if (sdk) |s| {
        if (s.windows10sdk) |w10| {
            const arch_str: []const u8 = switch (arch) {
                .x86_64 => "x64",
                .x86 => "x86",
                .aarch64 => "arm64",
                else => "x64",
            };
            const ucrt_lib_path = std.fmt.allocPrint(
                b.allocator,
                "{s}\\Lib\\{s}\\ucrt\\{s}",
                .{ w10.path, w10.version, arch_str },
            ) catch null;
            if (ucrt_lib_path) |path| {
                lib.addLibraryPath(.{ .cwd_relative = path });
            }
        }
    }
    lib.linkSystemLibrary("libucrt");
}
```

Result: ghostty.dll builds successfully (60MB, all exports present).

## What I learned

**Static CRT init linking is not overhead -- it is architecture.** Every Windows DLL does this. The `lib`-prefixed versions are not "the full static CRT" -- they are just the bootstrap code that sets up the connection to the dynamic CRT DLLs at load time. About 50-100KB in a 60MB binary.

**Zig's library search paths for Windows SDK are incomplete.** `linkLibC()` adds the `um\x64` subdirectory but not the `ucrt\x64` subdirectory. These are siblings under the same SDK version directory. The UCRT was split out of the Windows SDK starting with VS 2015 / Windows 10 SDK.

**Use `std.zig.WindowsSdk` for SDK detection, not hardcoded paths.** Zig has a built-in Windows SDK detector that reads the registry. It handles different SDK versions and installation paths. Use it.

**When a dynamic import lib does not have the symbol you need, check the static version.** The naming convention is: `foo.lib` (dynamic import) vs `libfoo.lib` (static). The static versions contain different symbols -- internal implementation, not just DLL forwarding stubs.

**This is a real Zig bug worth reporting.** Closed issues 5748 and 5842 on ziglang/zig partially addressed this. An unmerged PR 5870 tried to fix it completely. The specific gap with `libvcruntime` and `libucrt` for DLL targets is not tracked. Zig's issue tracker is now on Codeberg.

Related patterns: [cmake](../patterns/cmake.md)
