Last verified: 2026-03-29

# Code Style

Lessons from code review about matching abstraction level to project scope.

## Patterns

**Match the abstraction level to the project.**
Section banners (// --- Section ---) and organizational comments are fine in production code. In a single-file demo meant to be read top to bottom, they add visual noise. The structure of the code speaks for itself. Read the room.

**Prefer the simplest interface that works.**
A --flag parser with strcmp loops is overkill when argv[1] does the job. Demo code should show the minimum viable approach. Users reading the code should see the feature, not the argument parsing.

**Test your fix in isolation before proposing upstream.**
When you find a bug in a dependency, create a minimal branch in your fork that fixes just that issue. Point your project at the fork and verify with CI. This gives you a concrete proof that the fix works and makes the upstream PR easy to review -- one line, one purpose, tested.

**Mirror the existing module structure even when your code is incomplete.**
If the codebase has an established pattern for how modules are organized (Metal.zig is the public surface, metal/ contains the implementation), follow it from day one. A one-line stub that only re-exports one type sends the wrong signal about how the module will grow. The right structure makes architecture obvious to reviewers and establishes the integration path.

**Provide inline wrappers consistently -- all or nothing.**
If some COM interfaces have inline wrappers for their methods and others use raw vtable calls, the inconsistency suggests some were forgotten. Wrap every method you use. This is especially important in a codebase where the reviewer expects uniformity.

**Use `b.allocator`, not `b.graph.arena`.**
In Zig's build system, `b.allocator` is the public API for getting an allocator from `std.Build`. `b.graph.arena` is the internal field it wraps. Using internal fields is brittle -- the implementation can change across Zig versions. Same principle as preferring an interface over a concrete type in any language.

**Don't use `for` + `break` when you mean "process the first element".**
A for loop implies iteration. If you always break after one iteration, use direct indexing (`arr[0]`) with a length check. The loop form is especially misleading when it has `continue` branches that silently skip remaining elements.

**Use switch captures instead of re-accessing union fields.**
If a switch arm captures a value (`|t|`), use the captured binding. Re-accessing `union.field` after the switch is redundant and fragile. Zig's multi-value return from switch arms (`const a, const b = switch ...`) lets you extract multiple values cleanly.

**Log warnings for unsupported paths, don't silently skip.**
When hitting a code path that can't do useful work yet (like an unimplemented texture attachment), emit a log.warn. Silent skips make bugs invisible. The warning also serves as documentation that the path exists but is not wired yet.

## Where I learned this

- [05-windows-port-ghostling](../case-studies/05-windows-port-ghostling.md) -- Mitchell's review of the ghostling Windows port
- [11-dx11-renderer-infrastructure](../case-studies/11-dx11-renderer-infrastructure.md) -- module structure and COM wrapper consistency
- [12-dll-crt-review](../case-studies/12-dll-crt-review.md) -- b.allocator vs b.graph.arena
- [15-dx11-render-loop-wiring](../case-studies/15-dx11-render-loop-wiring.md) -- for+break anti-pattern, switch captures, silent skips
