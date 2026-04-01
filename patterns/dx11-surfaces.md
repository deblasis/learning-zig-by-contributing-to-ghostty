Last verified: 2026-04-01

# DX11 Surface Modes and Composition

How swap chains, composition, and shared textures relate to each other in the DX11 renderer.

## Patterns

**Swap chain and composition are independent concepts.**
A swap chain is buffer infrastructure (back buffers, present semantics, flip model). Composition is how those buffers reach the screen via DWM. CreateSwapChainForHwnd ties them together. CreateSwapChainForComposition decouples them so you control the DWM visual tree.

**Use DirectComposition for HWND surfaces, not CreateSwapChainForHwnd.**
CreateSwapChainForHwnd ignores the alpha channel - DWM treats the window as opaque. DirectComposition lets DWM composite the swap chain properly, so per-pixel transparency works. When content is fully opaque, DWM uses independent flip automatically. This is the Windows Terminal approach.

**Always use premultiplied alpha for composition swap chains.**
Set DXGI_ALPHA_MODE_PREMULTIPLIED on both HWND (via dcomp) and XAML (via SwapChainPanel) paths. DWM decides independent flip vs composed flip based on actual pixel alpha values. No swap chain recreation needed when toggling transparency.

**The embedder picks a surface type, the renderer picks how to present it.**
Three surface types serve different embedding scenarios: HWND (window handle), SwapChainPanel (XAML), shared texture (external consumer). But composition vs swap chain is an internal renderer decision, not an embedder concern. Adding DirectComposition did not add a new surface type.

**Shared texture mode has no swap chain at all.**
When an embedder wants to sample the rendered texture on their own device, the renderer creates a D3D11 texture with MISC_SHARED, queries IDXGIResource::GetSharedHandle, and flushes the context instead of presenting. "How you present" and "what you render to" are separate concerns.

**DirectComposition requires Commit() after Present().**
After the swap chain presents, call IDCompositionDevice::Commit() to flush the composition tree to DWM. Without this, DWM may not pick up the new frame. This is a no-op for non-dcomp surfaces since dcomp_device is null.

**dcomp.dll is available on Windows 8.1+.**
Every supported Windows version (10+) has it. Do not write fallback paths to CreateSwapChainForHwnd. Dead code is not worth maintaining.

## Where I learned this

- [17-directcomposition-hwnd](../case-studies/17-directcomposition-hwnd.md) - replacing CreateSwapChainForHwnd with DirectComposition
