Last verified: 2026-04-05

# DX12 Resource Lifecycle

DX12's explicit resource model means you own every buffer, barrier, and command list lifetime. Getting it wrong produces silent corruption, not errors.

## Patterns

**D3D12 does not extend resource lifetimes for recorded commands.**
Unlike D3D11, releasing a resource after recording a command that reads from it is a use-after-free. The GPU hasn't executed the command yet, but the memory is already gone. This is the biggest difference from D3D11. Keep staging buffers alive until the GPU finishes the copy - store them on the texture and release at the next safe point (next upload call or deinit).

**Init-time textures need a dedicated one-shot command list.**
DX12 texture uploads require a command list for CopyTextureRegion. Per-frame command lists only exist after beginFrame, but textures are created during renderer init (before any frame). Create a temporary command allocator + command list, record the init barriers, execute, wait for the GPU, then release. This is the standard DX12 pattern for one-shot initialization work.

**Atlas sync must happen after beginFrame in a cross-backend renderer.**
Metal's replaceRegion is immediate - it copies CPU data to the GPU texture without a command encoder. DX12's CopyTextureRegion must be recorded on a command list. If generic renderer code syncs atlas textures before beginFrame, DX12 has no command list to record on. The fix is to move beginFrame before atlas sync. Metal and OpenGL don't care about the order, so this is safe for all backends.

**Triple-buffered textures hold stale command list pointers.**
DX12 rotates command lists across frame slots (typically 3). A texture created or last updated in frame slot 0 caches that slot's command list pointer. When the texture is used again in frame slot 1, the cached pointer is stale (it points to slot 0's command list, which may be in use by the GPU). Refresh the texture's command list pointer from the current frame before every upload.

**Use @hasDecl for optional backend-specific hooks in generic code.**
When cross-backend generic code needs to call DX12-specific functions (like flushInitCommands or updateTextureCommandList), use `@hasDecl(GraphicsAPI, "methodName")` to check at comptime. Metal and OpenGL don't declare these methods, so the call compiles away. This is the established pattern in Ghostty's generic renderer.

**Resource state tracking must stay in sync with actual GPU state.**
Every DX12 texture tracks its current resource state (COPY_DEST, PIXEL_SHADER_RESOURCE, etc.) in a field. Transitions record barriers on the command list and update the tracked state. If a barrier is recorded on a command list that never executes (null or stale CL), the tracked state diverges from the real GPU state. Future barriers will use wrong "before" states, causing GPU validation errors or silent corruption.

## Where I learned this

- [20-dx12-init-command-list](../case-studies/20-dx12-init-command-list.md) - init command list, staging buffer UAF, beginFrame ordering
