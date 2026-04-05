# COM ABI Struct Passing

## Context

After the DX12 pivot, the renderer initialized successfully - device, swap chain, descriptor heaps, everything. But the moment it tried to create a shader resource view for the glyph atlas texture, the GPU reported DEVICE_REMOVED. No useful error message, just "the device is gone."

This happened even though all the COM calls returned S_OK. The device looked healthy right up until it wasn't.

## What I did

Traced it to two separate COM ABI issues, both related to how structs cross the Zig-to-COM boundary.

### Problem 1: Struct return methods (PR #142)

`GetCPUDescriptorHandleForHeapStart` and `GetGPUDescriptorHandleForHeapStart` return a `D3D12_CPU_DESCRIPTOR_HANDLE` struct by value. In the COM binary ABI, "return struct by value" actually means "caller passes a hidden pointer, callee writes the result there."

I had declared the vtable slot as:
```zig
GetCPUDescriptorHandleForHeapStart: *const fn (*Self) callconv(.winapi) D3D12_CPU_DESCRIPTOR_HANDLE,
```

But the COM binary ABI expects:
```zig
GetCPUDescriptorHandleForHeapStart: *const fn (*Self, *D3D12_CPU_DESCRIPTOR_HANDLE) callconv(.winapi) *D3D12_CPU_DESCRIPTOR_HANDLE,
```

The result: all three descriptor heaps returned identical garbage handle values. The inline wrapper hides the ugly output-pointer convention from callers.

### Problem 2: Struct-by-value parameters (PR #145)

`CreateShaderResourceView` takes a `D3D12_CPU_DESCRIPTOR_HANDLE` by value. On MSVC x64, structs that fit in a register (1, 2, 4, or 8 bytes) are passed as plain integers. The vtable slot must use `usize`, not the struct type.

I had:
```zig
CreateShaderResourceView: *const fn (*Self, ..., D3D12_CPU_DESCRIPTOR_HANDLE) callconv(.winapi) void,
```

Fixed to:
```zig
CreateShaderResourceView: *const fn (*Self, ..., usize) callconv(.winapi) void,
```

With an inline wrapper that takes the struct and passes `.ptr` (the inner usize).

### Problem 3: Union sizing (also PR #145)

`D3D12_SHADER_RESOURCE_VIEW_DESC` contains a union. I only declared the `Texture2D` variant (16 bytes). But MSVC sizes the union to its largest member, which is `D3D12_BUFFER_SRV` (24 bytes). My struct was 28 bytes, MSVC's was 36. The D3D12 runtime read past my struct and got garbage, triggering DEVICE_REMOVED.

Fix: add `D3D12_BUFFER_SRV` to the union even though I never use buffer SRVs.

## What I learned

COM ABI bugs are brutal because they compile fine, the calls "succeed" (return S_OK), and then the GPU dies later with no clear connection to the cause. The symptoms show up far from the bug.

The pattern I follow now:
1. Check the Windows SDK C header for every COM method I bind. Not the C++ version - the C version shows the actual calling convention with hidden parameters
2. Write comptime or test assertions for every struct size
3. When a union appears in a struct, include ALL variants even if I only use one
4. For methods taking or returning small structs, check if MSVC would pass them as integers

Reference: PRs 142 and 145 on deblasis/ghostty. See [com-vtable](../patterns/com-vtable.md).
