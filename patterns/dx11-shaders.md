Last verified: 2026-03-29

# DX11 Shader Patterns

Rules for writing HLSL shaders that consume data from Zig extern structs and integrating them into the Ghostty build pipeline.

## Patterns

**Always use packoffset in cbuffers that are written from C/Zig structs.**
HLSL cbuffer packing rules (16-byte rows, no straddling) differ from C struct alignment (field alignment with padding). Without packoffset, fields after the first padding gap read wrong bytes. Silent corruption, no error.

**Use Load() for pixel-exact texture sampling, Sample() for filtered sampling.**
Glyph atlases need exact pixel lookups (Load with integer coords). Background images and other filtered content use Sample/SampleLevel with normalized [0,1] UVs. Metal's coord::pixel maps to Load(), not Sample().

**Semantic names are validated, byte offsets are not.**
D3D11 CreateInputLayout checks that HLSL semantic names match the input element descriptions. But byte offsets (AlignedByteOffset) are trusted blindly. Wrong offsets give garbage rendering with no error. Use comptime assertions on @sizeOf and @offsetOf to catch drift.

**Full-screen triangle pipelines need no input layout.**
Pipelines that generate geometry from SV_VertexID (bg_color, cell_bg) don't need input elements or vertex buffers. Pass null for input_elements and stride 0. The vertex shader does all the work.

**One HLSL file, multiple entry points, separate fxc invocations.**
HlslStep compiles each entry point from the same source file with a different /E flag. Duplicate resource declarations at the same register (like t0) are fine because fxc only sees resources referenced by the entry point being compiled.

**Embed shader bytecode via @embedFile, not runtime compilation.**
Build-time compilation via fxc.exe + @embedFile means no runtime dependency on d3dcompiler DLL and no startup compilation cost. The build system (SharedDeps.zig) creates anonymous imports for each compiled .cso file.

**Guard @embedFile with builtin.os.tag for cross-platform builds.**
The anonymous imports only exist on Windows builds. Use a comptime if to provide empty slices on other platforms so the module compiles everywhere.

**Store stride on Pipeline, read it in RenderPass.**
Each pipeline knows its instance buffer stride at init time (from @sizeOf of the instance struct). RenderPass reads it when calling IASetVertexBuffers. This avoids hardcoding strides at the draw call site.

## Where I learned this

- [HLSL shaders and pipeline loading](../case-studies/14-hlsl-shaders-pipeline-loading.md) - first time writing HLSL for the DX11 renderer, the cbuffer packoffset lesson was the big one
