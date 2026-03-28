# DX11 Review Cleanup

## Context

Self-review of PR 31 on deblasis/ghostty (DX11 renderer infrastructure). The PR had landed with pre-compiled .cso shader binaries committed to git, a `pipeline.zig` that collided with the generic `Pipeline.zig` contract on case-insensitive filesystems, implicit pointer coercion in a COM call, and emdashes in comments.

## What I did

Renamed `pipeline.zig` to `cell_pipeline.zig` to avoid the case-insensitive collision. Replaced the `@embedFile` references to .cso files with empty placeholders (shaders will come from HlslStep.zig at build time) and removed the .cso binaries from git. Added an explicit `@as(?*d3d11.ID3D11Buffer, self.constant_buffer)` cast in VSSetConstantBuffers. Cleaned up emdashes and method count comments across all COM interface files.

## Before and after

Before (implicit coercion, works but hides intent):
```zig
ctx.VSSetConstantBuffers(0, &.{self.constant_buffer});
```

After (explicit @as makes the optional visible):
```zig
ctx.VSSetConstantBuffers(0, &.{@as(?*d3d11.ID3D11Buffer, self.constant_buffer)});
```

Before (collides on Windows):
```
directx11/Pipeline.zig      - generic contract
directx11/pipeline.zig      - cell grid implementation
```

After (distinct names):
```
directx11/Pipeline.zig      - generic contract
directx11/cell_pipeline.zig - cell grid implementation
```

## What I learned

Case-insensitive filesystem collisions do not produce errors on Linux CI or even on Windows if only one of the two files is referenced at a time. They surface as confusing "wrong module" bugs when both are imported in the same compilation unit. Name concrete implementations after their purpose, not their abstraction.

Compiled shader bytecodes (.cso) are build artifacts, not source. Committing them creates merge noise, makes diffs unreadable (binary), and drifts from the HLSL source. The build system (HlslStep.zig) should be the single source of truth.

Related patterns: [com-vtable](../patterns/com-vtable.md), [code-style](../patterns/code-style.md)
