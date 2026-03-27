# DX11 Renderer Infrastructure

## Context

After proving the SwapChainPanel approach worked in the spike (PR 22 on deblasis/ghostty -- a C# WinUI 3 app rendering animated DX11 scenes via libghostty.dll), the next step was extracting the DX11 infrastructure into clean, reviewable code that could eventually integrate with Ghostty's renderer framework.

Ghostty has a `GenericRenderer(GraphicsAPI)` pattern where Metal.zig and OpenGL.zig each provide the platform's graphics API as a self-contained module. The DX11 renderer needs to follow the same structure.

## What I did

### COM interface bindings

Created Zig bindings for DX11 and DXGI COM interfaces from scratch:

- `com.zig` -- GUID, HRESULT helpers, IUnknown base, shared Reserved type
- `dxgi.zig` -- IDXGIDevice, IDXGIAdapter, IDXGIFactory2, IDXGISwapChain1/2, ISwapChainPanelNative
- `d3d11.zig` -- ID3D11Device (43 methods), ID3D11DeviceContext (115 methods), textures, buffers, shaders, render target views

Each vtable is laid out slot by slot with comments referencing the Windows SDK headers. The spike taught me that missing one inherited slot (IDXGIFactory2's CreateSoftwareAdapter) shifts every subsequent call -- a bug that only manifests at runtime as an access violation.

### Device lifecycle

`device.zig` handles D3D11 device creation, swap chain for XAML composition (CreateSwapChainForComposition with PREMULTIPLIED alpha and FLIP_SEQUENTIAL -- same approach as Windows Terminal), render target view management, resize, present, and cleanup. Uses `errdefer` chains so if step 4 of 6 fails, steps 1-3 are cleaned up automatically.

### Instanced cell grid renderer

`pipeline.zig` loads pre-compiled HLSL shaders via `@embedFile`, creates the input layout, and manages a constant buffer for per-frame uniforms. `cell_grid.zig` manages a CPU-side cell array and a dynamic GPU instance buffer. The shader generates quads from SV_VertexID + SV_InstanceID -- no index buffer needed.

### Module structure

Initially `DirectX11.zig` was a one-liner that only re-exported `Device`. During code review, I expanded it to mirror Metal.zig's pattern: module doc comment, self-reference (`pub const DirectX11 = @This()`), re-exports for all sub-modules and renderer components, and a test block.

### Build system integration

Added `directx11` to the renderer Backend enum with Windows as the default. Linked d3d11 and dxgi system libraries in `GhosttyLib.initShared`. Static builds do not need explicit linking because static archives do not resolve symbols -- the consumer does.

The `GenericRenderer` integration is placeholder (`GenericRenderer(OpenGL)`) because implementing the full GraphicsAPI contract (Target, Frame, RenderPass, Buffer, Texture, Sampler, shaders) is follow-up work.

### DllMain CRT workaround

Zig's `_DllMainCRTStartup` does not initialize the MSVC C runtime when building a DLL. This means any C library function that depends on CRT state (setlocale, malloc from C deps like glslang and oniguruma) crashes. Declared a `DllMain` in `main_c.zig` that calls `__vcrt_initialize` and `__acrt_initialize` on DLL_PROCESS_ATTACH. Uses `@extern` to get the function pointers without pulling in CRT objects that would conflict with Zig's own startup symbol. Comptime-guarded to Windows MSVC only.

## What upstream did

N/A -- this has not been submitted upstream yet. It stacks on two open PRs (CI test suite and DLL CRT linking fix).

## Before and after

Before (DirectX11.zig -- one line):
```zig
pub const Device = @import("directx11/device.zig").Device;
```

After (mirrors Metal.zig module pattern):
```zig
pub const DirectX11 = @This();

pub const com = @import("directx11/com.zig");
pub const d3d11 = @import("directx11/d3d11.zig");
pub const dxgi = @import("directx11/dxgi.zig");

pub const Device = @import("directx11/device.zig").Device;
pub const Pipeline = @import("directx11/pipeline.zig").Pipeline;
pub const Constants = @import("directx11/pipeline.zig").Constants;
pub const CellGrid = @import("directx11/cell_grid.zig").CellGrid;
pub const CellInstance = @import("directx11/cell_grid.zig").CellInstance;
```

Before (raw vtable call in device.zig):
```zig
hr = dev.vtable.QueryInterface(dev, &dxgi.IDXGIDevice.IID, &dxgi_device_opt);
```

After (consistent inline wrapper):
```zig
hr = dev.QueryInterface(&dxgi.IDXGIDevice.IID, &dxgi_device_opt);
```

## What I learned

**COM vtable slot counting is not optional -- it is the entire correctness story.** There is no type system or compiler check that catches a missing inherited slot. The only defense is meticulous comments cross-referencing SDK headers. I found the CreateSoftwareAdapter bug in the spike only because IDXGIFactory2::CreateSwapChainForComposition was calling the wrong function.

**Mirror the existing module structure even when your code is incomplete.** Ghostty has a clear pattern: Metal.zig is the public surface, metal/ contains the implementation. DirectX11.zig should follow the same layout from day one, even if most of the GenericRenderer contract is not implemented yet. This makes the architecture obvious to reviewers and establishes the right integration path.

**Static archives do not need linkSystemLibrary for their dependencies.** Only shared libraries (DLLs) resolve all symbols at link time. A `.a` archive just packages object files -- symbol resolution happens when the consumer links the final binary. This is why the DX11 library linking only appears in `initShared`, not `initStatic`.

**`@extern` with `library_name` replaces explicit DLL imports.** Instead of using LoadLibrary/GetProcAddress or extern function declarations, Zig's `@extern` with `.library_name` tells the linker to pull the symbol from the named library. This is clean and works for both static and dynamic linking.

**`errdefer` chains are the Zig answer to COM cleanup.** C++ COM code uses RAII wrappers (ComPtr). C code uses goto chains. Zig's `errdefer` gives you the same correctness with less ceremony -- each acquisition line is followed by its cleanup, and the compiler guarantees they run in reverse order on error.

**Pre-compile HLSL shaders and embed them.** Runtime shader compilation requires the D3DCompiler DLL and adds startup latency. Pre-compiling with `fxc.exe` and embedding via `@embedFile` is the same approach Windows Terminal uses. Document the recompilation command somewhere.

Related patterns: [com-vtable](../patterns/com-vtable.md), [code-style](../patterns/code-style.md), [cmake](../patterns/cmake.md), [platform-abstraction](../patterns/platform-abstraction.md), [testing](../patterns/testing.md)
