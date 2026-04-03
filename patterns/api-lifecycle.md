Last verified: 2026-04-03

# API Lifecycle

Check the vendor's API lifecycle before building on it.

## Patterns

**Do not build on frozen or deprecated APIs, even if they are simpler.**
A frozen API means no new investment from the vendor. If the platform owner has moved on, your work has an expiry date. DX11 was easier than DX12 but upstream would not accept it because Microsoft stopped investing in DX11 after Windows 10 EOL.

**Separate reusable infrastructure from API-specific code.**
DXGI bindings, HLSL shaders, COM patterns, and composition code are not DX11-specific or DX12-specific. Keeping them in their own modules meant about 40% of the DX11 work carried forward to DX12. If the renderer had been one monolithic file mixing DXGI and D3D11 calls, the pivot would have been a full rewrite.

**When you need to pivot, do a clean break.**
No dual-renderer fallback, no compatibility layers, no "we will support both for now". That just doubles the maintenance surface. Pick the target API and commit to it. The old work is preserved on a branch if you ever need to reference it.

**Check upstream's appetite before investing.**
A week of DX11 work (March 26 to April 1, 2026) became sunk cost because upstream was not interested. A conversation earlier would have saved time. This is not just about API lifecycle - it is about checking that your chosen approach aligns with the project you are contributing to.

## Where I learned this

- [18-dx12-pivot](../case-studies/18-dx12-pivot.md) - 3 months of DX11 work pivoted to DX12 after upstream feedback
