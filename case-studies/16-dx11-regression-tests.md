# DX11 Regression Tests and Stale Base Branch

## Context

PR 75 fixed the padding background color (blend state + shader compositing). PR 76 fixed resize stretching (DXGI_SCALING_NONE + resize timer). During testing of PR 76, the PR 75 fix was completely missing - the padding bug was back.

The cause: `just sync` fetched upstream/main and rebased, but never pulled `origin/windows`. PR 75 had been squash-merged on GitHub, but the local `windows` branch was still at the old commit. Feature branch 038 was rebased on the stale windows, so it never got the PR 75 changes.

## What I did

**Fixed the sync recipe** by adding `git fetch origin windows` and `git merge --ff-only origin/windows` before the upstream rebase. This makes sure locally merged PRs are always present before rebasing on upstream.

**Added regression tests** so neither fix can silently disappear again:

1. Wired up two COM vtable methods that were `Reserved` - `ID3D11BlendState::GetDesc` and `IDXGISwapChain1::GetDesc1`. These let tests read back GPU object configuration.

2. `BlendState: premultiplied alpha config` - creates a blend state with the same parameters as Device.init, reads it back via GetDesc, and asserts SrcBlend=ONE, DestBlend=INV_SRC_ALPHA. This guards the PR 75 fix.

3. `SwapChain: HWND surface uses SCALING_NONE` - creates a hidden window using RegisterClassExW/CreateWindowExW (declared as extern "user32" inside the test), initializes a full Device, queries the swap chain via GetDesc1, and asserts Scaling=NONE. This guards the PR 76 fix.

4. Added D3D11_BLEND_DESC and D3D11_RENDER_TARGET_BLEND_DESC size checks to com_test.zig for ABI safety.

## Before and after

Wiring a COM method from Reserved to callable:

```zig
// Before: slot is a placeholder, can't call it
GetDesc: Reserved,

// After: typed function pointer + inline wrapper
GetDesc: *const fn (*ID3D11BlendState, *D3D11_BLEND_DESC) callconv(.winapi) void,

// ... and the wrapper:
pub inline fn GetDesc(self: *ID3D11BlendState, desc: *D3D11_BLEND_DESC) void {
    self.vtable.GetDesc(self, desc);
}
```

Creating a hidden window for testing (no real UI needed):

```zig
const WNDCLASSEXW = extern struct {
    cbSize: u32 = @sizeOf(@This()),
    style: u32 = 0,
    lpfnWndProc: *const fn (HWND, u32, usize, isize) callconv(.winapi) isize,
    // ... other fields with defaults
    lpszClassName: [*:0]const u16,
};

const class_name = std.unicode.utf8ToUtf16LeStringLiteral("GhosttyTestClass");
const wc = WNDCLASSEXW{ .lpfnWndProc = user32.defWindowProc, .lpszClassName = class_name };
_ = user32.RegisterClassExW(&wc);

const hwnd = user32.CreateWindowExW(
    0, class_name, null, 0,
    0, 0, 100, 100, null, null, null, null,
) orelse return; // skip if headless CI
defer _ = user32.DestroyWindow(hwnd);
```

## What I learned

- **COM GetDesc methods return void, not HRESULT.** Unlike Create methods which can fail, GetDesc just fills a struct. The function signature is `void (self, *Desc)` not `HRESULT (self, *Desc)`. GetDesc1 on IDXGISwapChain1 does return HRESULT though, so check each one.

- **You can declare Win32 extern functions locally in test scope.** No need to add them to the main bindings if they are only used in tests. Define the struct types and extern declarations in a `const user32 = struct { ... }` inside the test function.

- **Hidden windows are fine for swap chain testing.** A window that is never shown (no WS_VISIBLE, no ShowWindow) is enough for CreateSwapChainForHwnd. The HWND just needs to be valid.

- **Sync workflows must pull the fork remote too.** Fetching only upstream misses squash-merged PRs on the fork's own branch. The ff-only merge is safe because the local branch should always be a fast-forward of the remote after a squash merge.

Related patterns: [com-vtable](../patterns/com-vtable.md), [testing](../patterns/testing.md)

PR 75 and PR 76 on deblasis/ghostty.
