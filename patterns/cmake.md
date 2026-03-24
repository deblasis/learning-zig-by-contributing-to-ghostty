Last verified: 2026-03-24

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

## Where I learned this

- [03-ghostty-free](../case-studies/03-ghostty-free.md) -- heap mismatch, ghostty_free
- [05-windows-port-ghostling](../case-studies/05-windows-port-ghostling.md) -- Ninja OUTPUT fix proposed and merged upstream
