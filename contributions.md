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

**libghostty shared lib install fix for Windows** -- Draft
Install as ghostty.dll + ghostty-static.lib on Windows instead of libghostty.so + libghostty.a. Guard ubsan_rt bundling in GhosttyLib.zig for MSVC.
PR 17 on deblasis/ghostty. Related: [platform abstraction](patterns/platform-abstraction.md)

**Windows platform in embedded runtime** -- Draft
Added GHOSTTY_PLATFORM_WINDOWS to C API, Zig PlatformTag, and Platform union. Cross-platform testing caught a missing exhaustive switch arm in Metal.zig on Mac.
PR 18 on deblasis/ghostty. Related: [platform abstraction](patterns/platform-abstraction.md)

**CI full test suite for Windows** -- Open
Added `test-windows` job to CI running `zig build -Dapp-runtime=none test` on windows-2025. Added to required checks. First run exposed a CRLF comptime parsing bug -- fixed with code trim and .gitattributes normalization.
PR 11839 on ghostty-org/ghostty. Related: [testing](patterns/testing.md)

## 2026-03-26

**CRLF comptime parsing fix** -- Open (part of PR 11839)
actions/checkout on Windows CI converts LF to CRLF. The octants.txt comptime parser split by '\n' but the trailing '\r' became a struct field lookup, crashing the build. Fixed by trimming '\r' (same pattern as x11_color.zig) and adding `* text=auto eol=lf` to .gitattributes so the repo enforces LF everywhere. Scanned for other vulnerable @embedFile patterns -- only octants.txt needed fixing.
PR 11839 on ghostty-org/ghostty. Related: [testing](patterns/testing.md), [cmake](patterns/cmake.md)

**WinUI 3 C# project scaffold** -- Draft
Added windows/Ghostty/ with WinUI 3 project, XAML navigation, DPI awareness, publish profiles for x86/x64/ARM64. Mirrors the macOS/Swift project structure.
PR 19 on deblasis/ghostty.

**DLL CRT linking fix** -- Draft
Fixed ghostty.dll build on Windows MSVC by linking `libvcruntime` + `libucrt` (static CRT bootstrap libs) and detecting the UCRT path via `std.zig.WindowsSdk.find()`. This is a workaround for a Zig bug where `linkLibC()` does not set up the full CRT chain for DLL targets. Related Zig issues: 5748, 5842 (closed), PR 5870 (closed, not merged). Plan to file an issue and potentially a fix on Codeberg (Zig's new tracker) once I can do it confidently by hand.
PR 20 on deblasis/ghostty. Related: [cmake](patterns/cmake.md)

**P/Invoke bindings for libghostty** -- Draft
C# interop layer: GhosttyTypes.cs (struct/enum mappings), NativeMethods.cs (LibraryImport P/Invoke), GhosttyApp.cs (managed wrapper for init-config-app lifecycle). Uses source-generated P/Invoke with DisableRuntimeMarshalling. Stub callbacks for wakeup, action, clipboard, close_surface. Post-build step copies ghostty.dll to output.
PR 20 on deblasis/ghostty. Related: [cmake](patterns/cmake.md)

**DllMain CRT init workaround** -- Draft
Zig's _DllMainCRTStartup does not initialize MSVC C runtime for DLL targets. Declared DllMain in main_c.zig that calls __vcrt_initialize/__acrt_initialize on DLL_PROCESS_ATTACH via @extern. Comptime-guarded to Windows MSVC. This is the runtime counterpart to the linking fix in PR 20.
PR 21 on deblasis/ghostty. Related: [cmake](patterns/cmake.md), [platform abstraction](patterns/platform-abstraction.md)

**SwapChainPanel spike** -- Complete
C# WinUI 3 app with SwapChainPanel rendering DX11 scenes via libghostty.dll. Proved the architecture works: C# GUI wrapping Zig/DX11 rendering, matching macOS Swift + Metal. Animated demo scenes, bitmap font splash, resize support. Found IDXGIFactory2 vtable bug (missing CreateSoftwareAdapter slot).
PR 22 on deblasis/ghostty.

**DX11 renderer infrastructure** -- Draft
Extracted clean DX11 infrastructure from spike: COM bindings (com.zig, dxgi.zig, d3d11.zig), device lifecycle, instanced cell grid pipeline, pre-compiled HLSL shaders. DirectX11.zig expanded to mirror Metal.zig module pattern. Backend enum with directx11 variant, d3d11/dxgi system library linking. Tests verify struct sizes, GUID constants, vtable pointer layout. Cross-platform: 2604 Windows, 2655 Linux, 2655 Mac tests passing.
PR 23 on deblasis/ghostty. Related: [com-vtable](patterns/com-vtable.md), [code-style](patterns/code-style.md), [testing](patterns/testing.md)

## 2026-03-27

**DX11 code review fixes** -- Draft (part of PR 23)
Self-review applying Mitchell's patterns: expanded DirectX11.zig to mirror Metal.zig module structure, added QueryInterface inline wrapper to ID3D11Device for consistency, added HLSL shader README with fxc recompilation instructions.
PR 23 on deblasis/ghostty. Related: [com-vtable](patterns/com-vtable.md), [code-style](patterns/code-style.md)

**DLL CRT init review fixes** -- In progress
Mitchell's review of PR 11856: use `b.allocator` instead of `b.graph.arena` (public API over internals), move C test from `windows/Ghostty.Tests/` to `test/windows/` (follow project directory conventions, not .NET conventions).
PR 11856 on ghostty-org/ghostty. Related: [code-style](patterns/code-style.md), [testing](patterns/testing.md)
