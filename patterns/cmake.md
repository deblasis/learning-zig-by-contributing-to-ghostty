Last verified: 2026-03-25

# CMake

Patterns for building and consuming libghostty-vt via CMake on Windows.

## Patterns

**On Windows, DLLs go in bin/, import libs go in lib/.**
Zig places DLLs in zig-out/bin/ and import libraries (.lib) in zig-out/lib/. CMake's IMPORTED_LOCATION must point to the DLL in bin/, and IMPORTED_IMPLIB must point to the .lib in lib/. Getting this wrong causes link failures or runtime crashes.

**Ninja needs all outputs listed explicitly.**
If a custom command produces files as side effects (like the .lib import library alongside a DLL), Ninja will fail with "missing and no known rule to make it" unless those files are listed in the OUTPUT directive. The Visual Studio generator is more forgiving. Always list every produced file.

**Verify assumptions about the build graph before drawing conclusions.**
We initially assumed framegen only ran during dist packaging because upstream reverted the scandir fix shortly after applying it. We concluded our fix was unnecessary. Later, cross-platform testing proved framegen DOES run during normal builds (`zig build -Dapp-runtime=none test`) via `SharedDeps.framedata`. The revert had a different reason we hadn't identified. Lesson: verify build graph dependencies with actual test runs rather than inferring from git history alone.

**Zig's libc on Windows is not the system CRT.**
When Zig builds a DLL with link_libc = true on Windows, it uses its own libc implementation on top of KERNEL32.dll, not ucrtbase.dll (the Windows C runtime). This means the library and MSVC consumers have separate heaps. Calling free() on library-allocated memory crashes. The fix is to provide a library-owned free function (like ghostty_free) so the correct allocator is always used.

**For DLLs on MSVC, linkLibC() is not enough -- you need the static CRT bootstrap libs.**
Zig's `linkLibC()` links `msvcrt.lib` but not the companion `libvcruntime.lib` and `libucrt.lib` that contain CRT initialization code (`__vcrt_initialize`, `__acrt_initialize`). Static libs do not need these because symbols resolve at final link time. DLLs must resolve everything immediately. Link `libvcruntime` and `libucrt` explicitly in the shared library build step. This is a Zig bug (partially addressed in closed issues 5748, 5842 on ziglang/zig) that has not been fully fixed.

**The UCRT library path is not on Zig's default search path.**
Zig adds the Windows SDK `um\x64` directory to library search paths but not the `ucrt\x64` directory. These are sibling directories under the same SDK version. Use `std.zig.WindowsSdk.find()` to detect the SDK installation and construct the UCRT path. Do not hardcode paths -- SDK versions and install locations vary.

**`lib`-prefixed and unprefixed Windows libs are different things.**
On MSVC, `foo.lib` is the dynamic import library (forwarding stubs to foo.dll). `libfoo.lib` is the static library (actual code linked into your binary). When a symbol is missing from the dynamic version, check the static version. The CRT initialization symbols only exist in the static versions because they are bootstrap code that runs before the dynamic CRT is available.

## Where I learned this

- [03-ghostty-free](../case-studies/03-ghostty-free.md) -- heap mismatch, ghostty_free
- [05-windows-port-ghostling](../case-studies/05-windows-port-ghostling.md) -- Ninja OUTPUT fix proposed and merged upstream
- [09-dll-crt-linking](../case-studies/09-dll-crt-linking.md) -- DLL CRT bootstrap linking, UCRT path detection, Zig bug
