# DX11 Render Loop Wiring

## Context

The DX11 renderer had shaders loaded and pipelines ready, but the render loop was not connected end-to-end. Frame.renderPass() had a hack that called device.clearRenderTarget() directly, bypassing the attachment binding that RenderPass.begin() was supposed to do. Target was just width+height with no actual render target view (RTV). The pipeline path existed but nothing flowed through it.

The job was to wire the real RTV from Device through Target into RenderPass, so GenericRenderer's initTarget -> beginFrame -> renderPass -> step -> present path actually draws through the shader pipeline.

## What I did

### RTV ownership model

Device owns the swap chain's back buffer RTV. It creates it in init, releases and recreates it on resize. Target borrows the pointer without calling AddRef/Release.

This is safe because:
- GenericRenderer calls initTarget() each frame, so Target always gets the current RTV
- Target.deinit() nulls the pointer without Release
- If Target also Released, we'd get a double-free on resize

For future off-screen targets, Target will own its RTV and Release it in deinit. The comment in Target.zig explains this split.

### Moving attachment binding to RenderPass

The hack was in Frame.renderPass(): it reached into Device, extracted the clear color from the first attachment, and called device.clearRenderTarget(). This bypassed RenderPass entirely.

Metal's pattern is clear: RenderPass.begin() receives attachments, creates a descriptor, binds render targets. Frame is just a thin wrapper that forwards attachments.

I moved the viewport, RTV binding, and clear logic into RenderPass.begin(), and Frame.renderPass() became a passthrough.

### The for+break anti-pattern

My first version of RenderPass.begin() used a for loop over attachments with a break at the end:

```zig
for (opts.attachments) |att| {
    const rtv = switch (att.target) {
        .target => |t| t.rtv orelse continue,
        .texture => continue,
    };
    // ... bind and clear ...
    break;
}
```

This is confusing because it looks like a loop but always processes at most one element. The `continue` on texture means if the first attachment is a texture, we skip all remaining attachments too, which is not what I meant.

Self-review caught this. The fix was direct indexing:

```zig
if (opts.attachments.len == 0) return .{ .context = context, .device = device };
const att = opts.attachments[0];
```

With explicit early returns and log.warn for unsupported cases instead of silent skips.

### Switch capture reuse

The first version also had this:

```zig
const rtv = switch (att.target) {
    .target => |t| t.rtv orelse continue,
    .texture => continue,
};
const target = att.target.target;  // re-accessing the union!
```

The switch already captured `t` from the .target arm, but `t` went out of scope after the switch. Restructuring to capture both values from the switch arm avoids re-accessing the union:

```zig
const target, const rtv = switch (att.target) {
    .target => |t| .{ t, t.rtv orelse { ... } },
    .texture => { ... },
};
```

Zig's multi-value return from switch arms makes this clean.

## What I learned

- **RTV ownership matters.** The swap chain's back buffer RTV is owned by Device. Target borrows the pointer. Double-free would happen if Target also Released. Document ownership in comments, especially when pointers cross struct boundaries.
- **Attachment binding belongs in RenderPass, not Frame.** Metal's render pass descriptor binds attachments the same way. Putting it in Frame was a hack that ignored the attachments. Moving it to RenderPass means the same code path works for off-screen targets later.
- **DX11 immediate mode simplifies the frame lifecycle.** Metal needs command buffers and completion handlers. DX11 executes draw calls immediately on the device context, so Frame and RenderPass are thin wrappers with no deferred command recording.
- **for+break to process one element is misleading.** If you mean "process the first element", use direct indexing. The for loop implies iteration over multiple elements.
- **Use switch captures, don't re-access the union.** If a switch arm captures a value, use it. Re-accessing `union.field` after switching on the union is redundant and fragile.
- **Silent skips hide bugs.** When an unsupported case is hit (like a texture attachment that can't be bound yet), log a warning. Silent skips make debugging harder later.

PR 67 on deblasis/ghostty. Related: [code-style](../patterns/code-style.md), [com-vtable](../patterns/com-vtable.md)
