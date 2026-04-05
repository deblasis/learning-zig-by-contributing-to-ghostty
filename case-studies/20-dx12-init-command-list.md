# DX12 Init Command List and Staging Buffer Lifetime

## Context

After fixing the COM ABI issues (PRs 142 and 145), the DX12 renderer could create textures without crashing. But the terminal window showed white squares instead of text. The squares were positioned correctly - text layout worked, background color worked, but every glyph was a solid white rectangle.

## What I did

Traced the rendering pipeline from texture creation through to GPU execution. Found three linked problems:

### Problem 1: No command list during init

Ghostty's GenericRenderer creates placeholder atlas textures during init, before any frame is rendered. The DX12 Texture uses CopyTextureRegion to upload pixel data, which needs a command list. But per-frame command lists only exist after beginFrame, which runs in the render loop.

During init, `pending_command_list` was null. The texture stored null as its cached command list. When atlas data arrived, `uploadRegion` checked for null, logged an error, and silently returned. No data ever reached the GPU.

Fix: create a one-shot command allocator + command list in DirectX12.init(), set it as `pending_command_list` so initAtlasTexture picks it up through the existing code path, then flush (close, execute, wait, release) after SwapChain.init finishes.

### Problem 2: Atlas sync before beginFrame

Even after the init fix, the per-frame atlas sync was broken. In generic.zig, the atlas texture sync happened BEFORE beginFrame:

```
syncAtlasTexture(...)   // needs a command list
beginFrame(...)         // creates the command list
```

For Metal, this doesn't matter - `replaceRegion` is an immediate CPU-to-GPU copy that doesn't need a command encoder. For DX12, the texture uploads were recorded on a null command list and silently lost.

Fix: move beginFrame before atlas sync. Metal and OpenGL don't care about the order, so it's safe for all backends.

### Problem 3: Staging buffer use-after-free

After fixing the ordering, glyphs were STILL white squares. This one was subtle.

DX12 texture uploads work like this:
1. Create a staging buffer (UPLOAD heap)
2. Map it, copy pixel data in
3. Record CopyTextureRegion on the command list (staging -> texture)
4. The command list executes later in drawFrameEnd

The old code released the staging buffer right after step 3. But the GPU hadn't executed the copy yet - it was just recorded. D3D12 does NOT extend resource lifetimes for recorded commands (unlike D3D11). The GPU read freed memory, which happened to be all 0xFF, making every texel look like full glyph coverage - solid white rectangles.

Fix: store the staging buffer in a `pending_staging` field on the Texture. Release it at the start of the NEXT replaceRegion call (when the fence guarantees the GPU is done with the previous copy) or in deinit.

### Bonus: Command list staleness

Added `updateTextureCommandList` to refresh the texture's cached command list pointer before each upload. DX12 rotates command lists across 3 frame slots. A texture updated in frame slot 0 holds a pointer to slot 0's command list. In frame slot 1, that pointer is stale. Without refreshing it, uploads go to the wrong (possibly in-use) command list.

## Before and after

Before: grey window with no text (null CL), then white squares (staging buffer UAF).

After: actual glyphs rendered correctly. The terminal is usable.

## What I learned

DX12's explicit model means you own every resource lifetime. Three things that Metal/OpenGL handle automatically required explicit management:

1. **Init-time GPU work** needs its own command list because per-frame ones don't exist yet
2. **Staging buffers** must outlive command list execution, not just command list recording
3. **Command list pointers** go stale across triple-buffered frame rotation

The trickiest part was the staging buffer UAF. The symptoms (white squares) didn't obviously point to a freed staging buffer. The connection: freed GPU memory contained 0xFF, which the shader interpreted as full glyph coverage, producing solid white rectangles the exact size of each character cell.

Cross-backend generic code can hide API-specific ordering requirements. The atlas sync ordering didn't matter for Metal or OpenGL, so nobody noticed it was wrong for DX12 until DX12 actually tried to use it.

Reference: PR 149 on deblasis/ghostty, fixing issue 144. See [dx12-resource-lifecycle](../patterns/dx12-resource-lifecycle.md), [com-vtable](../patterns/com-vtable.md).
