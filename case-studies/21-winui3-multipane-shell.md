# WinUI 3 Multi-Pane Shell

## Context

The Windows shell scaffold landed in PRs 156 to 159 with a single TerminalControl per window. Mitchell-style splits (Ctrl+Shift+D for vertical, Ctrl+Shift+E for horizontal, Alt+Arrows for directional focus) were the next obvious feature. Two PRs were needed: a prerequisite refactor that hoisted libghostty's config and app handles up to a per-window host, then the actual splits feature on top.

This is the story of the embedder-side gotchas. The pure WinUI 3 quirks (routed event ordering, KeyboardAccelerator dispatch timing, Border being sealed, etc.) are not really Ghostty learnings and live in the project memory instead. What is documented here is the part that any non-Swift embedder will hit.

## What I did

### PR 161: hoist app and config ownership

The original TerminalControl created its own GhosttyConfig, GhosttyApp, and GhosttySurface in OnLoaded, with a TODO comment saying "when we add tabs/splits the app handle will move up". Splits forced the refactor.

I created a `GhosttyHost` per-window class that owns the config and app and the six runtime callback delegates (wakeup, action, read clipboard, confirm read clipboard, write clipboard, close surface). TerminalControl became a thin surface host that takes a `Host` property and only creates its own surface.

The action callback needed to know which control to dispatch to. The first attempt kept a `Dictionary<IntPtr, TerminalControl>` keyed on `GhosttySurface.Handle` and tried to look up `targetPtr` directly. The bell test passed (it does not need the target). Title and close silently dropped every dispatch.

The reason: `ghostty_target_s` and `ghostty_action_s` are 16-byte structs passed by value in the C signature, but on Windows x64 any struct larger than 8 bytes is passed via a hidden pointer. The C# delegate already had `IntPtr targetPtr` for that reason, but I had been treating that pointer as the surface handle when it was actually the address of an ephemeral stack copy of the target struct. The fix was to dereference: read the tag at offset 0 (must be `GHOSTTY_TARGET_SURFACE`), then read the surface pointer at offset 8 (the union starts at 8 because of alignment), THEN look up the dictionary.

For per-surface callbacks (close_surface, clipboard) the surface handle is not what gets echoed back. They take the userdata that was set on the surface config. The pattern there is to allocate a normal `GCHandle` to the control just before `surface_new`, set `surfaceConfig.Userdata = GCHandle.ToIntPtr(handle)`, and decode it in the host with `GCHandle.FromIntPtr(userdata).Target as TerminalControl`. The handle is freed in the explicit close path AFTER `surface_free` returns, because libghostty may touch userdata during teardown.

### PR 163: multi-pane splits

Once the host refactor was in, splits could be built on top. The model is a binary tree of `LeafPane` and `SplitPane` with pure visitor and mutation helpers in `PaneTree.cs`. A `PaneHost` UserControl owns the tree and renders it as nested 2-cell Grids with a hand-rolled 1px draggable Splitter between cells. Bindings live in a `KeyBindings` registry that both MainWindow (for the keyboard accelerators) and TerminalControl (for the chord short-circuit) read from.

The biggest non-WinUI lesson came from surface lifetime. The original TerminalControl created the surface in `OnLoaded` and freed it in `OnUnloaded`. WinUI 3 fires Unloaded then Loaded asynchronously when an element moves between parents during a tree rebuild, and the events can be delivered out of order for the same element. The result: every pane split killed and restarted every existing pane's shell process, and some leaves ended up permanently in an Unloaded state with no surface and no recovery path. After typing `pre` in the original pane and splitting, the original pane came back as a fresh shell with no `pre` in scrollback.

The fix was to decouple surface lifetime from the visual-tree events entirely. The surface is created exactly once at first Loaded (guarded by `_surfaceCreated`) and freed only when PaneHost explicitly calls `DisposeSurface()` on the leaf actually being closed. OnUnloaded became a no-op for the native resource. This is the right shape for any embedder where the UI framework reparents elements.

## What upstream did

There is no upstream comparison here since the Windows shell is independent fork work that I am not pushing back to ghostty-org. The macOS Swift shell already does both of these things correctly: the Ghostty app handle lives on a per-window controller, surfaces are owned by the controller and outlive SwiftUI view lifecycle, and the action callback target is read from the union. I just had to learn it the hard way before realizing the macOS code already had the answer.

## What I learned

The action callback's struct-by-value parameters are the same Win x64 ABI rule that bit me in the DX12 COM bindings (case study 19), but in a different context. The COM bindings case was Zig writing vtable types for D3D12. The libghostty embedder case was C# writing a delegate for a libghostty C callback. Same underlying ABI rule, different surface, both bugs compile fine and pass shallow tests.

The surface-lifetime lesson is one I would not have hit on macOS because SwiftUI's view identity model handles reparenting differently. Whenever you write the embedder against a UI framework that reparents elements through its lifecycle events (WinUI 3, WPF, Avalonia, GTK, even React Native if anyone tried), you have to assume those events do NOT mean "destroy this". Native state owned by the control has to be explicitly disposed by the owner, not by the framework.

Reference: PRs 161 and 163 on deblasis/ghostty. See [libghostty-embedder](../patterns/libghostty-embedder.md) and [com-vtable](../patterns/com-vtable.md) for the underlying Win x64 ABI rule.
