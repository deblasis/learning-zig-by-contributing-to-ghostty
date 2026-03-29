# HLSL Shaders and Pipeline Loading

## Context

The DX11 renderer had all the infrastructure in place - device, swap chain, buffers, render pass, pipeline objects - but no actual shaders. Every pipeline was empty so RenderPass.step() would check for null shaders and skip every draw call. The renderer was going through all the motions but painting nothing.

The job was to write HLSL shaders for all 5 pipelines (bg_color, cell_bg, cell_text, image, bg_image), compile them at build time, and load them into real D3D11 objects at init.

## What I did

### The cbuffer layout trap

The hardest part was getting the Uniforms cbuffer to match the Zig extern struct byte-for-byte. GenericRenderer writes the Zig struct directly into the constant buffer, so the HLSL side has to read bytes at the exact same offsets.

The problem: HLSL cbuffer packing rules are not the same as C struct alignment. HLSL packs fields into 16-byte rows and won't let a field straddle a row boundary. C/Zig extern structs use field alignment with padding.

Concrete example that would silently break without packoffset:

```
Zig:
  grid_size: [2]u16 align(4)    -> offset 80, 4 bytes
  grid_padding: [4]f32 align(16) -> offset 96 (12 bytes padding!)

HLSL auto-pack:
  grid_size at byte 80 (row 5, slot x)
  next field at byte 84 (row 5, slot y)  <- WRONG, should be 96
```

The fix is using `packoffset` on every field in the cbuffer to force exact byte positions.

### Texture sampling

Metal uses `coord::pixel` for glyph atlas sampling, which takes raw pixel coordinates. In HLSL the equivalent is `Texture2D.Load()` which takes integer pixel coordinates. The more common `Sample()` expects normalized [0,1] UVs and applies filtering, which would give blurry or slightly offset glyphs.

For background images we use `SampleLevel()` with normalized UVs since we want filtering there.

### Shared vertex shaders

bg_color and cell_bg both use full-screen triangles (one oversized triangle clipped to the viewport, generated from SV_VertexID). They share the same vertex shader - BgColorVS. The pixel shader is where they differ: BgColorPS returns a flat color, CellBgPS does a grid lookup.

cell_text and image use instanced triangle strips (4 verts per instance) because each glyph or image is a positioned quad. These need per-instance data from the input layout.

### Build system

The existing HlslStep infrastructure handles compiling multiple entry points from one source file. Each gets its own fxc.exe invocation with a different /E flag and output name. The bytecode gets embedded via @embedFile so there's no runtime shader compilation and no files to ship.

### Input layout validation

D3D11 CreateInputLayout validates the input element descriptions against the vertex shader bytecode signature at creation time. The semantic names in HLSL (GLYPH_POS, BEARINGS, etc.) must match the strings in the D3D11_INPUT_ELEMENT_DESC exactly. A mismatch is a hard error at init, not a silent failure.

The byte offsets in the input elements, on the other hand, are not validated against anything. If you say color starts at byte 20 instead of 24, the GPU reads wrong bytes and you get garbage colors with no error. The comptime assertions on struct sizes and field offsets help catch this.

## What I learned

- HLSL cbuffer packing is not C struct packing. Always use packoffset when the buffer is written from a C/Zig struct. See [dx11-shaders pattern](../patterns/dx11-shaders.md).
- Load() is for pixel-exact sampling (glyph atlases). Sample() is for filtered/normalized sampling (images).
- Input layout semantic names are validated at creation. Byte offsets are not - use comptime assertions.
- Full-screen triangles don't need an input layout or instance buffer. The vertex shader generates geometry from SV_VertexID alone.
- Zig's @embedFile plus the build system's anonymous imports make offline shader compilation clean - compile once at build time, embed the bytecode, no runtime dependencies on d3dcompiler.

PR 66 on deblasis/ghostty. Related: [dx11-shaders](../patterns/dx11-shaders.md), [com-vtable](../patterns/com-vtable.md)
