Last verified: 2026-03-27

# COM Vtable Bindings in Zig

How to write correct COM interface bindings in Zig for DirectX 11 and DXGI.

## Patterns

**Every inherited method slot must be present in the vtable, even if unused.**
COM vtables are flat arrays of function pointers. If you skip one inherited slot, every subsequent slot shifts by one and calls the wrong function. This causes subtle runtime crashes, not compile errors. Use a `Reserved` stub type for slots you do not call.

**Use a shared `Reserved` type defined once.**
All unused vtable slots should use the same type: `pub const Reserved = *const fn () callconv(.winapi) void;`. Define it in a common module (com.zig) and import everywhere. Duplicating the type per file invites drift.

**Document slot numbers in comments.**
Number every vtable slot and cross-reference with the Windows SDK header (d3d11.h, dxgi.h). This is the only reliable way to verify you have not missed a slot. For large interfaces like ID3D11DeviceContext (115 methods), list the full slot mapping in a block comment above the vtable.

**Provide inline wrappers for every method you actually call.**
Raw `self.vtable.Foo(self, ...)` calls work but are inconsistent if some interfaces have wrappers and others do not. Wrap every method you use. This also gives you a place to convert Zig idioms (slices to ptr+len) at the boundary.

**Each COM type is an extern struct with a single vtable pointer field.**
COM interfaces in Zig are `extern struct { vtable: *const VTable }`. This matches the C++ ABI -- a COM object is one pointer to a vtable. Verify with `comptime { std.debug.assert(@sizeOf(MyInterface) == @sizeOf(*anyopaque)); }` or test it.

**Use `errdefer` chains for COM resource cleanup.**
COM resources are reference-counted. Every successful `QueryInterface`, `CreateDevice`, or factory call returns an object that must be `Release()`d on error. Zig's `errdefer` is perfect for this -- acquire, errdefer release, repeat. Release in reverse acquisition order.

**Use `@extern` with `library_name` for DLL function imports.**
For functions imported directly from system DLLs (like `D3D11CreateDevice` from d3d11.dll), use `@extern` with `.library_name`. This tells the linker to pull the symbol from the named library. COM interface methods do not need this -- they go through vtable pointers obtained at runtime via QueryInterface.

**Verify struct sizes at comptime or in tests.**
Extern structs that cross the COM boundary (DXGI_SWAP_CHAIN_DESC1, D3D11_BUFFER_DESC, etc.) must match the C ABI exactly. Size mismatches cause runtime crashes with no useful error message. Write tests like `expectEqual(@sizeOf(MyStruct), 48)` and include a comment explaining the expected value.

**Use `callconv(.winapi)` for all COM function pointers.**
COM uses `__stdcall` on 32-bit and the default calling convention on 64-bit. Zig's `.winapi` handles both cases. Never use `.c` for COM.

## Where I learned this

- [11-dx11-renderer-infrastructure](../case-studies/11-dx11-renderer-infrastructure.md) -- building DX11/DXGI bindings from scratch
