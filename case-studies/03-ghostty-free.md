# Adding ghostty_free for cross-runtime memory safety

## Context

On Windows, calling `free()` on memory allocated by libghostty crashes. When Zig builds a DLL with `link_libc = true`, it uses its own libc implementation backed by KERNEL32.dll, not the Windows C runtime (ucrtbase.dll). So even though `builtin.link_libc` is true and `c_allocator` is selected, Zig's malloc and MSVC's malloc are different implementations with separate heaps.

On Linux and macOS this is not a problem because Zig links the system libc and everyone shares the same heap.

The `format_alloc` docs said "the buffer can be freed with free()" but that is only true when the library and consumer share the same C runtime.

## What I did

I added `ghostty_free(allocator, ptr, len)` that frees through the same allocator that did the allocation. I updated the format_alloc docs and all 3 examples. I put the function inline in terminal/c/main.zig with no tests.

## What upstream did

Mitchell merged our PR but improved it before merging:

1. Added `ghostty_alloc()` alongside `ghostty_free()` -- completing the API pair
2. Created a dedicated `src/terminal/c/allocator.zig` file instead of putting it in main.zig
3. Added 6 tests covering normal use, null allocator, zero length, null pointer, and cross-allocator scenarios
4. Added a section-level "Alloc/Free Helpers" doc block in the header explaining the use cases
5. Used module naming: `alloc_alloc`, `alloc_free` internally, `ghostty_alloc`, `ghostty_free` exported

## Before and after

My approach (inline in main.zig, no tests):
```zig
// terminal/c/main.zig
pub fn free_alloc(
    alloc_: ?*const CAllocator,
    ptr: ?[*]u8,
    len: usize,
) callconv(.c) void {
    const mem = ptr orelse return;
    const alloc = lib_alloc.default(alloc_);
    alloc.free(mem[0..len]);
}
```

Upstream approach (dedicated file, with alloc + tests):
```zig
// terminal/c/allocator.zig
pub fn alloc(...) callconv(.c) ?[*]u8 { ... }
pub fn free(...) callconv(.c) void { ... }

test "alloc returns non-null" { ... }
test "alloc with null allocator" { ... }
test "free null pointer" { ... }
// ... 6 tests total
```

## What I learned

Complete the API pair. If someone needs free, they probably need alloc too.

New C API functions go in their own file, not inline in main.zig. Follow the existing pattern: cell.zig, formatter.zig, allocator.zig.

Always add tests. We had zero. Mitchell added six. Every exported function needs tests covering normal use and edge cases.

Add section-level docs in headers, not just per-function. Explain the group and when to use it.

See [api design](../patterns/api-design.md) and [cmake](../patterns/cmake.md).

PR 11785 on ghostty-org/ghostty (merged).
