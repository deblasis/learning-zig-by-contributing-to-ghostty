# Contributions

Chronological log of contributions to Ghostty and related projects.

## 2026-03-23

**test-lib-vt compilation fix** -- Superseded
Fixed test-lib-vt compilation on all platforms by gating refAllDecls and test imports behind artifact checks. Upstream fixed the root cause differently (made modules safe to import in all configurations).
PR 1 on deblasis/ghostty. Related: [testing](patterns/testing.md), [platform abstraction](patterns/platform-abstraction.md)

**C API invalid enum crash fix** -- Superseded
Fixed crash in C API get functions with .invalid enum and null pointer. Used early return + unreachable. Upstream used a single switch arm and caught one more file we missed.
PR 2 on deblasis/ghostty. Related: [api design](patterns/api-design.md), [testing](patterns/testing.md)

**Build tools Windows fix** -- Superseded
Replaced scandir with opendir/readdir in framegen, fixed helpgen stdout writer. Both files were rewritten upstream in the Zig 0.15 migration.
PR 3 on deblasis/ghostty. Related: [cmake](patterns/cmake.md)

**Config path parsing** -- Open
CommaSplitter backslash handling for Windows paths, drive letter colon detection in theme parsing. Reworked after learning the comptime constant pattern.
PR 4 on deblasis/ghostty. Related: [platform abstraction](patterns/platform-abstraction.md)

**expandHome test skip** -- Open
Skip expandHomeUnix test on Windows with SkipZigTest.
PR 5 on deblasis/ghostty. Related: [testing](patterns/testing.md)

**XDG test fixes** -- Open
Cross-platform cache test using path.join, skip XDG_DATA_DIRS tests on Windows.
PR 6 on deblasis/ghostty. Related: [testing](patterns/testing.md)

## 2026-03-24

**ghostty_free** -- Merged
Added ghostty_free() for cross-runtime memory safety on Windows. Mitchell improved it before merging: added ghostty_alloc(), dedicated allocator.zig file, 6 tests.
PR 11785 on ghostty-org/ghostty. Related: [api design](patterns/api-design.md), [cmake](patterns/cmake.md)

**cmake import-lib OUTPUT fix** -- Merged
One-line fix adding the .lib import library to the custom command OUTPUT directive. Without it Ninja fails on Windows. Tested in fork first, then proposed upstream.
PR 11794 on ghostty-org/ghostty. Related: [cmake](patterns/cmake.md), [code-style](patterns/code-style.md)

**ghostling Windows port** -- Open
ConPTY-based terminal emulation, threaded pipe reader, shell auto-detection. Single commit on top of upstream main.
PR 6 on ghostty-org/ghostling. Related: [code-style](patterns/code-style.md), [cmake](patterns/cmake.md)

**ssize_t typedef for MSVC** -- Draft
MSVC's sys/types.h does not define ssize_t. Added conditional typedef using SSIZE_T from BaseTsd.h. Fixes translate-c step.
PR 13 on deblasis/ghostty. Related: [platform abstraction](patterns/platform-abstraction.md)

**linkLibCpp + freetype enum signedness** -- Draft
Skip linkLibCpp on MSVC for dcimgui, spirv-cross, harfbuzz (same pattern as upstream glslang fix). Fix freetype C enum signedness with @intCast and @bitCast.
PR 14 on deblasis/ghostty. Related: [platform abstraction](patterns/platform-abstraction.md)

**comptime branch quota for ghostty.h enum checks** -- Draft
MSVC translate-c output has ~2173 declarations vs ~1502 on Linux/Mac. The 1M comptime branch quota was not enough for MouseShape (34 variants). Diagnosed with 7 isolated POC tests. Initial fix bumped to 10M. See 2026-03-25 for the refactor.
PR 15 on deblasis/ghostty. Related: [comptime](patterns/comptime.md)

## 2026-03-25

**freetype compilation fix** -- Merged
Fix freetype compilation on Windows with MSVC.
PR 11807 on ghostty-org/ghostty. Related: [platform abstraction](patterns/platform-abstraction.md)

**ssize_t typedef for MSVC** -- Merged
MSVC's sys/types.h does not define ssize_t. Added conditional typedef using SSIZE_T from BaseTsd.h.
PR 11810 on ghostty-org/ghostty. Related: [platform abstraction](patterns/platform-abstraction.md)

**linkLibCpp + freetype enum signedness** -- Merged
Skip linkLibCpp on MSVC for dcimgui, spirv-cross, harfbuzz. Fix freetype C enum signedness with @intCast and @bitCast.
PR 11812 on ghostty-org/ghostty. Related: [platform abstraction](patterns/platform-abstraction.md)

**checkGhosttyHEnum refactor to @hasDecl** -- Merged
Replaced O(N x M) nested `inline for` with O(N) `@hasDecl` lookups. Quota dropped from 10M to 100K. The original 10M fix treated the symptom; this fixed the algorithm. Comptime string construction required arrays not slices -- `++` does not work with slices at comptime.
PR 11813 on ghostty-org/ghostty. Related: [comptime](patterns/comptime.md)

**config.Config.test.clone segfault investigation** -- Diagnosed
Traced the only remaining Windows test failure (exit code 3) to CommaSplitter treating backslashes as escape characters. Windows paths like `C:\Users\...` hit `\U` which is an invalid escape. Cherry-picking the fix from PR 11782 resolved it. No new PR needed -- the fix is already open upstream.
Related: [testing](patterns/testing.md), [platform abstraction](patterns/platform-abstraction.md)

**Windows native architecture research** -- Complete
Researched Windows Terminal internals (AtlasEngine, DirectWrite, ConPTY, XAML Islands), evaluated native UI frameworks (WinUI 3, Win32+DirectX, WPF, Avalonia), investigated SwapChainPanel transparency limitations and workarounds. Conclusion: C# + WinUI 3 wrapping libghostty.dll, mirroring the macOS Swift + AppKit architecture. SwapChainPanel with premultiplied alpha for rendering, acrylic effects rendered in the shader pipeline. Updated fork README roadmap accordingly.

**Visual Studio reinstall** -- Complete
Reinstalled Visual Studio for C#/WinUI 3 development tooling.

**FPS counter feature branch** -- Complete
Simple pluggable stats overlay (FPS counter and potentially other metrics) for early development without needing the full inspector. Designed to be pulled in ad-hoc via a feature branch and replaceable once inspector support lands.

**CI full test suite for Windows** -- Draft
Added `test-windows` job to CI running `zig build -Dapp-runtime=none test` on windows-2025. Added to required checks. Ready to upstream once PR 11782 merges.
PR 16 on deblasis/ghostty. Related: [testing](patterns/testing.md)
