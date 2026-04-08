Last verified: 2026-04-08

# libghostty Embedder C API

Patterns for consuming libghostty's C API from a non-Swift, non-Zig embedder. The reference embedder upstream is the macOS Swift app, and a few things only show up when you write the embedder in a different language with a different UI framework.

## Patterns

**Action callback `target` and `action` arguments are passed by value but Win x64 hands them to you as hidden pointers.**
The C signature is `bool action_cb(ghostty_app_t, ghostty_target_s, ghostty_action_s)`. Both struct types are 16 bytes. On the Windows x64 calling convention, structs larger than 8 bytes are passed via a hidden pointer to a caller-allocated stack copy, not by value in registers. The C# delegate must therefore declare them as `IntPtr` and dereference. `ghostty_target_s` layout is `{ ghostty_target_tag_e tag at offset 0; void* surface at offset 8 }`. The union starts at offset 8 because of 8-byte alignment. Treating `targetPtr` directly as the surface handle silently misses every dispatch. The bell action passes because it does not need the target; title and close break because they do.

**Use per-surface userdata to route async callbacks back to the owning UI element.**
`close_surface_cb`, `read_clipboard_cb`, `write_clipboard_cb`, and `confirm_read_clipboard_cb` all echo back a `void*` userdata that was set on the `ghostty_surface_config_s` before `surface_new`. This is the only sane way to know which managed control owns a callback when the embedder hosts multiple surfaces. In .NET, allocate a normal (non-pinned) `GCHandle` to `this` in the control just before `surface_new`, set `surface_config.userdata = GCHandle.ToIntPtr(handle)`, and decode it in the host with `GCHandle.FromIntPtr(userdata).Target as MyControl`. Free the handle in the explicit close path AFTER `surface_free` returns, since libghostty may touch userdata during teardown.

**libghostty surface lifetime must be decoupled from UI-framework visual-tree events.**
Whenever the embedder UI framework reparents elements (WinUI 3, WPF, Avalonia, GTK, SwiftUI), it fires lifecycle events that are easy to mistake for "this control is being destroyed". Tying surface_new and surface_free to those events kills the running shell on every pane move. Concretely, WinUI 3 fires Unloaded then Loaded asynchronously when an element moves between parents, and the events can be delivered out of order. The right pattern: create the surface exactly once at first mount, free it only when the embedder's owner explicitly closes the leaf (or the window goes away). Mount/unmount events should be no-ops for the native resource.

**Hoist app and config ownership above the per-surface control.**
The first natural design is "one TerminalControl owns its app, config, and surface". This works for a single pane but breaks the moment you want multi-pane: `ghostty_app_t` is meant to host many surfaces, the action callback's `target` argument exists to disambiguate between them, and config reload and bell routing fragment if every surface has its own app. The right shape is one `Host` per window that owns the config and app, and N controls that each own only their surface. The macOS Swift code already does this; if you start otherwise, plan the refactor early.

**`wait-after-command` is a libghostty config option, not an embedder concern.**
The "process exited, press a key to close" behavior on shell exit is governed by `wait-after-command` in the user's ghostty config. The embedder does NOT enable it by hardcoding `surface_config.wait_after_command = true`. That would override user config silently. If a user wants the prompt, they set it in `~/.config/ghostty/config`. The embedder respects user config by leaving the field at its default.

**`ghostty.h` is internal, only the VT headers are the external public API.**
The `ghostty.h` header that the embedded apprt uses is shared between libghostty and its in-tree embedders (macOS, GTK, your app), but it is not a stable external API and changes between versions. Build against it from inside the ghostty repo, do not vendor it into a separate project as if it were a published header. The truly stable, externally consumable API is `libghostty-vt` and its headers under `include/`.

## Where I learned this

- [21-winui3-multipane-shell](../case-studies/21-winui3-multipane-shell.md) - building the WinUI 3 shell, hoisting config/app to a per-window host, decoupling surface lifetime from Loaded/Unloaded, and dereferencing the action target struct
