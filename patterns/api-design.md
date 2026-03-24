Last verified: 2026-03-24

# API Design

Patterns for the libghostty-vt C API, learned from code review and upstream improvements.

## Patterns

**Complete the API pair, not just one side.**
If someone needs to free through the library's allocator, they probably also need to allocate through it. We only added ghostty_free(). Mitchell added both ghostty_alloc() and ghostty_free() before merging. Think about the full lifecycle.

**One file per concern in terminal/c/.**
New C API functions go in their own file under src/terminal/c/, imported and reexported through main.zig. We put our function inline in main.zig. Mitchell created a dedicated allocator.zig. This matches the existing pattern: cell.zig, formatter.zig, color.zig, etc.

**Naming follows module grouping.**
Internal names use the pattern module_verb: alloc_alloc, alloc_free, formatter_format_buf. The export mapping in lib_vt.zig adds the ghostty_ prefix: ghostty_alloc, ghostty_free. Keep internal names consistent with their module.

**Section-level docs in headers, not just per-function.**
When adding related functions to a C header, add a section-level doc block explaining the group and when to use it. We documented only the function. Mitchell added a whole "Alloc/Free Helpers" section explaining the use cases.

**One switch arm beats guard + unreachable.**
When handling a new enum variant in a switch, add it as a direct arm (`.invalid => .invalid_value`) instead of adding a guard before the switch and marking the variant unreachable inside it. Same effect, half the code, all logic in one place.

## Where I learned this

- [02-c-api-crash-fix](../case-studies/02-c-api-crash-fix.md) -- switch arm pattern
- [03-ghostty-free](../case-studies/03-ghostty-free.md) -- complete the pair, dedicated files, naming, section docs, tests
