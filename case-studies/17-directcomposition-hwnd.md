# DirectComposition for HWND Surfaces

## Context

The DX11 renderer had three surface modes: HWND (CreateSwapChainForHwnd), XAML composition (CreateSwapChainForComposition), and shared texture. The HWND path used ALPHA_MODE_UNSPECIFIED, which means DWM ignores the alpha channel entirely. No per-pixel transparency possible.

Issue 93 on deblasis/ghostty proposed a progressive adaptive composition strategy. The first step was switching the HWND path to DirectComposition so the C# app gets transparency support from day one.

## What I learned

**Swap chains and composition are different things.** A swap chain is buffer infrastructure - back buffers, present semantics, flip models. Composition is how those buffers get attached to a window and displayed by DWM. They are independent concepts that work together.

CreateSwapChainForHwnd ties both together. DWM manages the swap chain presentation directly on the window. The alpha channel is ignored because DWM treats it as a simple window surface.

CreateSwapChainForComposition decouples them. You create a DirectComposition visual tree (dcomp device, target, visual) and attach the swap chain as content on the visual. DWM composites the visual tree, which means it respects the alpha channel. When content is fully opaque, DWM is smart enough to use independent flip (direct scanout, no extra DWM copy). When transparency is present, it falls back to composed flip (one frame of DWM latency).

This is how Windows Terminal does it. Always premultiplied alpha, always composition. DWM handles the optimization automatically.

**DirectComposition is available on Windows 8.1+.** Every supported Windows version has dcomp.dll. No fallback to CreateSwapChainForHwnd needed. The "auto-upgrade with fallback" approach would be dead code from day one.

**The embedder API did not change.** The embedder still passes an HWND. The renderer internally creates a DirectComposition visual tree on that HWND. This is an implementation detail, not a new surface mode. Three surface modes in, three surface modes out.

## Before and after

Before (device.zig HWND branch):
```zig
.Scaling = .NONE,
.AlphaMode = .UNSPECIFIED,
// ...
hr = factory.CreateSwapChainForHwnd(@ptrCast(dev), hwnd, &desc, null, null, &swap_chain);
```

After:
```zig
// Create dcomp visual tree
hr = dcomp.DCompositionCreateDevice(@ptrCast(dxgi_device), &dcomp.IDCompositionDevice.IID, &dcomp_device_opt);
// ...create target for hwnd, create visual...

.Scaling = .STRETCH,
.AlphaMode = .PREMULTIPLIED,
// ...
hr = factory.CreateSwapChainForComposition(@ptrCast(dev), &desc, null, &swap_chain);

// Attach swap chain to visual tree
visual.SetContent(@ptrCast(swap_chain.?));
target.SetRoot(visual);
dcomp_dev.Commit();
```

After Present(), flush the composition tree:
```zig
if (self.dcomp_device) |dcomp_dev| {
    _ = dcomp_dev.Commit();
}
```

## Design choices

We considered three approaches:
1. Minimal - change the HWND branch in Device, add dcomp bindings (chosen)
2. Strategy pattern - extract swap chain creation into a SwapChainStrategy
3. Unify HWND and XAML into one composition path, collapse the Surface enum

Chose approach 1. Approach 2 is over-engineering for two nearly identical paths. Approach 3 is the right end state but premature without a real reason to unify. This matches how the upstream project makes decisions: simplest thing that works, refactor when you have a reason.

## What the libghostty-dotnet work taught me

Building real .NET examples (Win32, WinForms, WPF, WinUI 3, shared texture) forced me to understand the rendering stack from both sides. The WinUI 3 example uses SwapChainPanel, which goes through CreateSwapChainForComposition with premultiplied alpha. The Win32 example used CreateSwapChainForHwnd. Seeing both paths side by side made the composition vs swap chain distinction concrete.

The shared texture example (DXGI shared handle) showed a third model where there is no swap chain at all - just a texture that gets flushed. That helped me see that "how you present" and "what you render to" are separate concerns.

Related patterns: [com-vtable](../patterns/com-vtable.md), [platform-abstraction](../patterns/platform-abstraction.md)

PR 97 on deblasis/ghostty.
