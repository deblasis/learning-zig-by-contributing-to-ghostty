# DX12 Pivot

## Context

I spent about a week building a complete DX11 renderer for the Ghostty Windows port (March 26 to April 1, 2026). It worked: 3 surface modes, 5 HLSL pipelines, cell rendering wired end-to-end, 6 .NET examples proving the architecture. The DX11 branch was solid.

Then the upstream Ghostty team made it clear they are not interested in shipping a DX11 backend. The reasoning makes sense: DX11 is frozen at Microsoft, following Windows 10 EOL in October 2025. No new features, no investment, just maintenance mode. Upstream does not want to own a renderer built on a dead API.

## What I did

Pivoted to DX12-only, no fallback. Clean break.

Analyzed what could carry forward vs what was sunk cost:
- **Carried forward:** DXGI bindings (adapters, factories, swap chains), DirectComposition bindings, COM helpers, test infrastructure, HLSL shaders (recompiled for SM 6.0 DXIL), the GenericRenderer contract pattern from Metal.zig
- **Sunk cost:** D3D11 COM interfaces, DX11 device lifecycle, DX11-specific resource management

Backed up the DX11 work to `windows_dx11` branch. Restructured as a 15-PR stacked branch plan. Each PR builds on the previous one, from directory rename through to GPU integration tests.

The reusable pieces meant the pivot was not starting from zero. DXGI is DXGI whether you use DX11 or DX12. The HLSL shaders needed recompilation but not rewriting. COM patterns are the same. About 40% of the infrastructure carried over directly.

## What upstream did

Upstream was straightforward about it. DX11 is not something they want to maintain long-term. DX12 is the only path that has a chance of being accepted. No ambiguity, which made the decision easy even if the sunk cost stung.

## What I learned

Building on a frozen API wastes effort even when that API is simpler and more familiar. DX11 was easier to work with than DX12, but "easier" does not matter if the result cannot ship.

The saving grace was that I built the DX11 renderer with clean abstractions: COM wrappers, DXGI as a separate layer, HLSL as a separate compilation step. If I had baked D3D11 calls directly into the renderer with no separation, the carry-forward would have been close to zero.

The lesson is two-fold:
1. Check the vendor's API lifecycle before committing to an API. If it is frozen or deprecated, it is not a foundation - see [api lifecycle](../patterns/api-lifecycle.md)
2. Clean abstraction boundaries pay off when you need to pivot. The DXGI/HLSL/COM separation was not planned for a pivot, but it made the pivot survivable

## Before and after

The DX11 renderer directory structure:
```
src/renderer/directx11/
  com.zig, d3d11.zig, dxgi.zig, dcomp.zig
  DirectX11.zig, Device.zig, Target.zig, ...
  shaders/ (fxc compiled .cso)
```

After pivot:
```
src/renderer/directx12/
  com.zig, dxgi.zig, dcomp.zig          <- carried forward
  d3d12.zig                              <- new, replaces d3d11.zig
  DirectX12.zig, Device.zig, Target.zig  <- rewritten for DX12
  shaders/ (dxc compiled .dxil)          <- same HLSL, new compiler
```

Reference: PRs 107-110 and 113-124 on deblasis/ghostty. See [api lifecycle](../patterns/api-lifecycle.md).
