---
title: How CBVs data struct works
description: This is my first post on Docusaurus.
slug: d3d12-cbvs-data-struct
authors: perry
tags: [cg, d3d12]
image: https://i.imgur.com/mErPwqL.png
hide_table_of_contents: false
---

Direct3D 12 Constant Buffer Views (CBVs) have a strict rule: the starting address must be a multiple of 256 bytes.

<!-- truncate -->

```cpp
#include <DirectXMath.h>

struct GPUDataConstantBuffer
{
    // A 3D Vector, with coordinate is a float number, for example { 1.0f, 1.0f, 1.0f }.
    // Each float has a size of 4 bytes, total 12 bytes
    XMFLOAT3 lightSourcePosition; // 12 bytes
    // float is 4 bytes
    float padding0;
    // A 4D Vector, same with 3D, now with the w coordinate.
    // Total 16 bytes.
    XMFLOAT4 lightSourceColor; // 16 bytes
    XMFLOAT3 cameraPosition; // 12 bytes
    // Total: 40 bytes upto this point
    float padding[53]; // 53 * 4 = 212 bytes
    // Total: 256 bytes
};

GPUDataConstantBuffer m_data[n];
```

Imagine an array of data: GPUDataConstantBuffer m_data[10].

- **Without Padding**: The struct is only 40 bytes. `m_data0` is at byte 0, and `m_data1` is at byte 40. But the GPU hardware is physically incapable of "starting" a read at byte 40. It only knows how to jump to 0, 256, 512, and so on.
- **With Padding**: By adding float padding[53], it forces the struct to be exactly 256 bytes. Now, when the application execute `memcpy` to upload the whole array, `m_data1` lands exactly at byte 256. The GPU "jump" aligns perfectly with the data.

In additional, inside the shader (HLSL), data is also packed into 4-component (16-byte) slots. A single variable, like a `float4`, is not allowed to straddle the boundary between two slots.

Considering this struct, without any padding:

```cpp
#include <DirectXMath.h>

struct GPUDataConstantBuffer
{
    XMFLOAT3 lightSourcePosition; // 12 bytes (Bytes 0-11)
    // No padding here?
    XMFLOAT4 lightSourceColor; // 16 bytes (Bytes 12-27)
};
```

In C++, `lightSourceColor` starts at Byte 12. But HLSL sees that float4 and says, "I can't start a 4-component vector at Byte 12; I must skip to the next 16-byte boundary." It looks for the color at Byte 16.

Because the `memcpy` put the data at 12, but the GPU is looking at 16, the `lightSourceColor` will look like "garbage" or black. That tiny float `padding0` is what shifts the C++ data forward to match the HLSL's internal alignment.

Or much better, **prefer to use `XMFLOAT4` for almost everything** in constant buffers. Instead of `XMFLOAT3` and a float padding, just use an XMFLOAT4 and ignore the .w component in the shader. It makes the C++ code cleaner and significantly reduces the chance of a math error.

### References
Frank Luna. Introduction to 3D Game Programming with DirectX 12, 6.6 Constant Buffers, 224 - 235