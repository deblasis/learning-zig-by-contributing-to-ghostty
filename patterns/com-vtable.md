Last verified: 2026-04-05

# COM Vtable Bindings in Zig

How to write correct COM interface bindings in Zig for DirectX and DXGI.

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

**Use explicit `@as` for COM optional pointer coercion.**
When passing a `*T` where `?*T` is expected (common in COM method wrappers that take optional arrays), use `@as(?*T, value)` to make the coercion visible. Implicit coercion works but hides the intent, and reviewers cannot tell if the optionality was considered.

**Wire up GetDesc methods for regression testing.**
COM objects store their creation parameters internally. Calling GetDesc/GetDesc1 reads them back, which lets you verify the config in tests. This is how you guard against someone changing blend factors or swap chain scaling. Note: some GetDesc methods return void (ID3D11BlendState), others return HRESULT (IDXGISwapChain1). Check the SDK header for each one.

**You can declare Win32 extern functions locally in test scope.**
If a Win32 API (RegisterClassExW, CreateWindowExW, etc.) is only needed in tests, declare the extern and struct types inside a `const user32 = struct { ... }` in the test function. No need to add them to the main bindings. Hidden windows (never shown) are valid HWND targets for swap chain creation.

**DirectComposition interfaces follow the same vtable pattern as D3D11/DXGI.**
dcomp.dll exposes IDCompositionDevice, IDCompositionTarget, and IDCompositionVisual. They inherit IUnknown and use the same flat vtable layout. IDCompositionVisual has overloaded methods (SetOffsetX float vs animation) that each occupy their own slot - count carefully. SetContent is at slot 15, not where you'd guess from the method list.

**Avoid case-insensitive filename collisions in module naming.**
Windows filesystems are case-insensitive. If a directory has a generic contract type `Pipeline.zig` and a concrete implementation `pipeline.zig`, they collide. Name the concrete type after its purpose, like `cell_pipeline.zig`, so the names are distinct on any filesystem.

**COM methods returning structs use a hidden output pointer, not register returns.**
In the COM binary ABI, methods that return a struct by value actually take a hidden pointer as the first parameter (after `self`) and write the result there. Zig's default calling convention treats them as register returns, which gives garbage. For example, `GetCPUDescriptorHandleForHeapStart` returns a `D3D12_CPU_DESCRIPTOR_HANDLE` (8 bytes). The vtable slot must be declared as `fn(self, *Handle) callconv(.winapi) *Handle`, not `fn(self) callconv(.winapi) Handle`. The inline wrapper hides this by taking the pointer internally and returning the value.

**Small structs passed by value in COM vtables may need to be declared as raw scalars.**
The MSVC x64 ABI passes structs of 1, 2, 4, or 8 bytes in a register as if they were integers. COM vtable slots follow this rule. A `D3D12_CPU_DESCRIPTOR_HANDLE` (just a `usize` wrapper) must be declared as `usize` in the vtable, not as the struct type. The type-safe inline wrapper converts between the scalar and the struct. If you use the struct type directly, the calling convention may pass it differently and you get wrong values.

**COM union types must include all variants to match MSVC layout, even unused ones.**
A union's size is determined by its largest member. If you skip a member you don't use, the union may be smaller than what MSVC produces. `D3D12_SHADER_RESOURCE_VIEW_DESC` has a union with `D3D12_BUFFER_SRV` (24 bytes) as its largest member. If you only declare `D3D12_TEX2D_SRV` (16 bytes), the struct is 28 bytes instead of 36. The D3D12 runtime reads 36 bytes and goes past the end, causing DEVICE_REMOVED. Always include enough union variants to match the MSVC size, even if you never fill them.

## Where I learned this

- [11-dx11-renderer-infrastructure](../case-studies/11-dx11-renderer-infrastructure.md) - building DX11/DXGI bindings from scratch
- [13-dx11-review-cleanup](../case-studies/13-dx11-review-cleanup.md) - @as coercion, filename collisions, build artifacts
- [16-dx11-regression-tests](../case-studies/16-dx11-regression-tests.md) - GetDesc wiring, hidden HWND for testing, regression guards
- [17-directcomposition-hwnd](../case-studies/17-directcomposition-hwnd.md) - DirectComposition bindings, overloaded method slots, composition vs swap chain
- [19-com-abi-struct-passing](../case-studies/19-com-abi-struct-passing.md) - struct return ABI, struct-by-value parameters, union sizing for D3D12
