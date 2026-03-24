Last verified: 2026-03-24

# Code Style

Lessons from code review about matching abstraction level to project scope.

## Patterns

**Match the abstraction level to the project.**
Section banners (// --- Section ---) and organizational comments are fine in production code. In a single-file demo meant to be read top to bottom, they add visual noise. The structure of the code speaks for itself. Read the room.

**Prefer the simplest interface that works.**
A --flag parser with strcmp loops is overkill when argv[1] does the job. Demo code should show the minimum viable approach. Users reading the code should see the feature, not the argument parsing.

**Test your fix in isolation before proposing upstream.**
When you find a bug in a dependency, create a minimal branch in your fork that fixes just that issue. Point your project at the fork and verify with CI. This gives you a concrete proof that the fix works and makes the upstream PR easy to review -- one line, one purpose, tested.

## Where I learned this

- [05-windows-port-ghostling](../case-studies/05-windows-port-ghostling.md) -- Mitchell's review of the ghostling Windows port
