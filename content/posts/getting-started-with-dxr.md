+++
title = "Getting Started With DirectX Raytracing"
date = "2022-07-04T22:08:12+02:00"
author = "Seppe Dekeyser"
authorTwitter = "SeppahBaws" #do not include @
cover = "/images/getting-started-with-dxr/DXR_Banner.png"
tags = ["rendering", "raytracing", "DirectX 12", "DirectX Raytracing"]
keywords = ["", ""]
description = "A tutorial/guide on getting started with DirectX Raytracing. It is basically everything I wished I had when getting started with DXR for the first time."
showFullContent = false
readingTime = true
hideComments = false
toc = true
+++

In [my previous post]({{< ref "/posts/ramblings-on-games-rendering-and-real-time-raytracing.md" >}} "my previous post") I talked about what raytracing is and what it does. This one is more of a tutorial/guide on getting started with DirectX Raytracing.

This post is based off the documentation I wrote during my internship last year at [DAE Research](https://digitalartsandentertainment.be/page/133/Research), slightly edited in places where some things were missing or needed some clarification. My aim for this is to be the one-stop place for anyone who's starting with DirectX Raytracing. It is basically what I would've personally wanted to have when I was getting started.

It might not be *all* the source code that you'll need, since I was working with [Microsoft's MiniEngine](https://github.com/microsoft/DirectX-Graphics-Samples/tree/master/MiniEngine) during my internship, but it should still provide all the necessary details.
All the source code of my internship project is available on [GitHub](https://github.com/SeppahBaws/DirectX-Raytracing/tree/main/MiniEngine/RayBinning), if you're curious. The main code is in the `RaytracingTest.cpp` file.


# Setup

## DXR-enabled device
The only additional step we have to do for MiniEngine DXR is to retrieve an `ID3D12Device5*` from the device. This is the earliest device version that supports ray-tracing. There currently are newer versions available, but this version should be all we need for this post.
```cpp
ComPtr<ID3D12Device5> pRTDevice;
HRESULT hr = g_Device->QueryInterface(IID_PPV_ARGS(&pRTDevice));
// If not successful, we can assume DXR is not supported.
```

Because this project is rather limited in scope, the only functionality we are going to need from the `ID3D12Device5` are the [`CreateStateObject()`](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device5-createstateobject) and the [`GetRaytracingAccelerationStructurePrebuildInfo()`](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device5-getraytracingaccelerationstructureprebuildinfo) functions. The full [Microsoft DirectX 12 documentation](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nn-d3d12-id3d12device5) shows what other functionality the `ID3D12Device5` interface provides.


# Root Signatures
A root signature defines what "root parameters" a shader has, what their type is, and in which register they are bound. In DirectX Raytracing we have two types of root signatures: global root signatures and local root signatures.

Creating a root signature with the `RootSignature` class in the MiniEngine is really simple:
```cpp
// SamplerDesc wraps a D3D12_SAMPLER_DESC and provides default values
SamplerDesc sampler;

RootSignature exampleSignature{};
// The Reset function takes in two parameters:
// UINT NumRootParams : the amount of root parameters we want to pass.
// UINT NumStaticSamplers : the amout of static samplers we want to pass.
exampleSignature.Reset(2, 1);
exampleSignature.InitStaticSampler(0, sampler); // Pass in the sampler on register 0
// We initialize the first entry as a buffer SRV, bound to register 0.
exampleSignature[0].InitAsBufferSRV(0);
// The 2nd entry gets initialized as a descriptor range.
// First we pass in the descriptor range type,
// then the register at which the descriptor range starts,
// and lastly the amount of descriptors that are in this range.
exampleSignature[1].InitAsDescriptorRange(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 1, 3);
// When we added all the entries, we can create the root signature.
// The string we pass in is the debug name that will show up in graphics debuggers
exampleSignature.Finalize(L"My Example Root Signature");
```


## Global Root Signature
A global root signature defines root parameters that are accessible across all DXR shaders in that pipeline. Every shader in the pipeline will have access to the root parameters defined in the global root signature.

Good candidates for parameters in the global root signature are: the raytracing output buffer, acceleration structure, mesh info...  
In general: parameters that are needed across all shader stages.

## Local Root Signature
Unlike a global root signature, a local root signature is only visible to one shader, specified upon creating the pipeline. Arguments are provided by the shader table.

Some logical use cases: bind an environment texture to the miss shader, bind the mesh texture to the hit shader...

## Local vs global root signatures
In general, you want to use global root signatures for data that has to be available to all shaders, and local root signatures for data that is specific to one shader step. One important thing to note when using local and global root signatures together, is that the registers of the local root signature cannot overlap with those defined in the global root signature.

Local root signatures also have a larger limit on the amount of shader records they can hold.


# DXR Shaders
Although DirectX Raytracing shaders are very similar to normal HLSL shaders, they do have some extra features to facilitate raytracing.

The most important thing to note is that raytracing is only supported in Shader Model 6.3 and above. In MiniEngine, shaders are built with Visual Studio, you just have to add them to the solution. In the file options, make sure to set the item type to "HLSL Compiler"
![](/images/getting-started-with-dxr/vs-shader-type.png)

To compile a raytracing shader in Visual Studio, make sure to add it to the solution, remove the Entrypoint Name, set the Shader Type to "Library" and set the Shader Model to "Shader Model 6.3". Higher Shader Model versions should also work, but that may depend on the compiler and Windows SDK version you use.
![](/images/getting-started-with-dxr/vs-shader-compile-options.png)

A raytracing shader also needs to have an "attribute" on the shader function. An attribute looks as follows: `[shader("shadertype")]`, where you replace `shadertype` with the type of the shader (the exact attributes will be shown in each section below).


## Ray generation shader
To declare a ray generation shader, assign the following attribute to your shader function: `[shader("raygeneration")]`

A ray generation shader in its essence will look something like the following:
```c
[shader("raygeneration")]
void RayGen()
{
    // Do some stuff...
    RayDesc ray = { /* ... */ };
    MyPayload payload = { /* ... */ };
    TraceRay( /* ... */, ray, payload);
}
```


The primary function of the ray generation shader is to call `TraceRay` to generate the rays that will be shot out, based on a `RayDesc` structure that is filled in and passed to the `TraceRay` function. The `RayDesc` structure is filled in as follows:
```c
RayDesc ray = {};
ray.Origin = /* ... */;
ray.TMin = /* ... */;
ray.Direction = /* ... */;
ray.TMax = /* ... */;
```

With this `RayDesc` structure now filled in, we can make a call to the `TraceRay` function:
```c
TraceRay(
    // Here we pass in the acceleration structure
    AccelerationStructure,

    // Flags to specify the behavior when a ray hits a surface. A good default
    // is RAY_FLAG_CULL_BACK_FACING_TRIANGLES
    RayFlags,

    // This mask can be used to mask out some geometries.
    // We pass in ~0 or 0xFF, indicating that no geometries will be masked out.
    InstanceOcclusionMask,

    // Sometimes an object can have multiple hit groups attached to it.
    // (e.g. one for diffuse shading, and one for shadow rays)
    // so we can use this parameter to index to the correct hit group
    // Since we only have one hit group in this project, we can default it to 0
    RayContributionToHitMask,

    // According to the documentation:
    // This specifies the stride to multiply by GeometryContributionToHitGroupIndex,
    // which is just the 0 based index the geometry was supplied by the app into the
    // bottom-level acceleration structure.
    // If you're not doing anything fancy with this, you can just set it to 1.
    MultiplierForGeometryContributionToHitGroupIndex,

    // In case we are using multiple miss shaders, we can use this parameter to
    // index to the correct shader that we want to use. If you only have one
    // miss shader, you can just pass in 0.
    MissShaderIndex,

    // Here we pass the RayDesc structure that we filled in the previous step
    RayDesc,

    // The payload that we associate with this ray. This is used to communicate
    // information between the raygen and hit/miss shaders.
    Payload
);
```

Important to note: the `TraceRay` function can be called from the Ray Generation shader, Closest Hit shader and the Miss shader. This is especially useful if you want to e.g. render reflections, as you can just call another `TraceRay` in the Closest hit shader.

### Ray Payload
The ray payload is a user-defined structure that gets passed along with the `TraceRay` function, and is then passed to the any hit, closest hit, and miss shaders as an inout parameter. Important to note is that the shaders using the payload must use the same structure as the one that was provided to the `TraceRay` function.

### TraceRay vs TraceRayInline
Besides the `TraceRay` function, there is also the `TraceRayInline` function. The inline version offers the same functionality as the normal `TraceRay` function, except that it doesn't make use of separate shaders for hit and miss etc. The shader that calls `TraceRayInline` has to control what the raytracer does. A more in-depth explanation can be found here: [DirectX Raytracing (DXR) Functional Spec](https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html#inline-raytracing).


## Miss shader
This shader is invoked when the ray does not hit anything at all. To identify a shader as a miss shader, use the `[shader("miss")]` attribute. A common use for this shader is to sample from an environment map.

An example:
```c
[shader("miss")]
void Miss(inout MyPayload payload)
{
    // Possibly sample from environment map...
    // Calls to TraceRay and CallShader can also be done here if desired
}
```

## Hit shaders
The hit shaders are executed when a ray intersects with a triangle in the acceleration structure. There are two different types of hit shaders:

### Closest Hit
The Closest Hit shader can only get invoked once per ray, at the closest intersection with an object. Most of the shading work should be done in this shader. The attribute used for this shader is `[shader("closesthit")]`.

A closest hit shader may look like this:
```c
[shader("closesthit")]
void ClosestHit(inout MyPayload payload, in MyAttributes attr)
{
    // Your logic here...
    // Possibly even additional calls to TraceRay with a reflected ray...
}
```

### Any Hit
The Any Hit shader is called every time a ray intersects with a triangle. They are very useful to calculate transparency in objects, as they can tell the API to ignore the current hit and continue searching for other hits. Any Hit shaders are defined by the attribute `[shader("anyhit")]`.

To prevent heavy performance impacts, it is good practice to keep the Any Hit shaders as trivial as possible, because they can get called many times per `TraceRay()` call.

An example of an any hit shader:
```c
[shader("anyhit")]
void AnyHit(inout MyPayload payload, in MyAttributes attr)
{
    // Typically some alpha-testing logic here...

    // Call to `AcceptHitAndEndSearch(...)` if we're ok with this current intersection.
    // Call `IgnoreHit(...)` if we want to discard this intersection and search for more.
}
```

## Intersection shader
An intersection shader is used in case you want to implement custom intersection primitives. If you have procedural geometry in your acceleration structure, you can write a custom intersection shader to test each ray for collision against this procedural object. (e.g. you can pass a sphere as a point and a radius, and then write a custom intersection shader to define these collisions, instead of making a triangle mesh for the sphere)

An intersection shader uses the `[shader("intersection")]` attribute:
```c
[shader("intersection")]
void Intersection()
{
    // Intersection checks
    // Call `ReportHit(...)` 
}
```

If you do not provide an intersection shader, DXR will use a default ray-triangle intersection shader. For most use cases, you shouldn't have to write an intersection shader yourself.

## Callable shader
Callable shaders are shaders that can be invoked from another shader, by using the `CallShader(...)` function. A callable shader can be used to group common behavior together, and reduce duplicated code across the shaders. For my simple example, I didn't find a use case to use a callable shader.

In order to declare a shader as a callable shader, you have to use the `[shader("callable")]` attribute on the shader function.
```c
[shader("callable")]
void Callable(inout MyParams params)
{
    // Do some shader magic.
    // Perhaps another call to CallShader
}
```



# Acceleration Structure
The acceleration structure is key to real-time raytracing. It is a Bounding Volume Hierarchy (BVH for short) which can be efficiently traversed to calculate ray-object intersections. In DXR this BVH exists of two levels: a Bottom-Level Acceleration Structure (BLAS) and a Top-Level Acceleration Structure (TLAS). The BLASes hold mesh data, along with a transform matrix. Each TLAS then holds an instance of a BLAS, along with a transform matrix.
![](/images/getting-started-with-dxr/nvidia-accel-overview.png)
Image credit: NVIDIA


## Scratch Buffer
To create the acceleration structure in DXR, we need to allocate a scratch buffer that will be used to store temporary calculations while building the acceleration structure on the GPU. Before we can do that, we first need to query the minimum size that we're gonna need:

```cpp
// Get the TLAS prebuild info, so that we know how much scratch buffer size we need.
D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO tlasPrebuildInfo;
D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC tlasDesc = {};

// Here we specify how many BLASes we need, and other parameters.
D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS& tlasInputs = tlasDesc.Inputs;
tlasInputs.Type = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_TOP_LEVEL;
tlasInputs.NumDescs = numBottomLevels;
tlasInputs.Flags = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_TRACE;
tlasInputs.pGeometryDescs = nullptr;
tlasInputs.DescsLayout = D3D12_ELEMENTS_LAYOUT_ARRAY;
// Query how much we need.
rtDevice->GetRaytracingAccelerationStructurePrebuildInfo(&tlasInputs, &tlasPrebuildInfo);

// We'll update this when we create the BLASes.
UINT64 scratchBufferSizeNeeded = tlasPrebuildInfo.ScratchDataSizeInBytes;
```

We'll come back to creating the actual scratch buffer later, when we know how big our scratch buffer needs to be.


## Bottom-Level Acceleration Structure
To create the BLASes, we need to first describe the geometry that it will take in.
In my case, I decided to have one BLAS for each model in my scene, and merge all the meshes in a model into the same BLAS, but your approach could be different.

```cpp
std::vector<D3D12_RAYTRACING_GEOMETRY_DESC> geometryDescs(numMeshes);

// Set up the descriptor for the mesh
for (UINT i = 0; i < numMeshes; i++)
{
    Model::Mesh& mesh = pModel->m_pMesh[i];

    D3D12_RAYTRACING_GEOMETRY_DESC& desc = geometryDescs[i];
    desc.Type = D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES;
    desc.Flags = D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE;

    // Specify some properties of the mesh data
    D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC& trianglesDesc = desc.Triangles;
    trianglesDesc.VertexFormat = DXGI_FORMAT_R32G32B32_FLOAT;
    trianglesDesc.VertexCount = mesh.vertexCount;
    trianglesDesc.VertexBuffer.StartAddress = pModel->m_VertexBuffer.GetGpuVirtualAddress() + (mesh.vertexDataByteOffset + mesh.attrib[Model::attrib_position].offset);
    trianglesDesc.VertexBuffer.StrideInBytes = mesh.vertexStride;
    trianglesDesc.IndexBuffer = pModel->m_IndexBuffer.GetGpuVirtualAddress() + mesh.indexDataByteOffset;
    trianglesDesc.IndexCount = mesh.indexCount;
    trianglesDesc.IndexFormat = DXGI_FORMAT_R16_UINT;
    trianglesDesc.Transform3x4 = 0;
}
```

Now that we have the geometry descriptors, we can create the BLAS create structs:
```cpp
// Prepare the BLAS create structs
std::vector<UINT64> blasSize(numBottomLevels);
std::vector<D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC> blasDescs(numBottomLevels);
for (UINT i = 0; i < numBottomLevels; i++)
{
    D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC& blasDesc = blasDescs[i];
    D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS& blasInputs = blasDesc.Inputs;
    blasInputs.Type = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL;
    blasInputs.NumDescs = numMeshes;
    blasInputs.pGeometryDescs = &geometryDescs[i];
    blasInputs.Flags = buildFlags;
    blasInputs.DescsLayout = D3D12_ELEMENTS_LAYOUT_ARRAY;

    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO blasPrebuildInfo;
    rtDevice->GetRaytracingAccelerationStructurePrebuildInfo(&blasInputs, &blasPrebuildInfo);

    blasSize[i] = blasPrebuildInfo.ResultDataMaxSizeInBytes;
    // Here we'll make sure to increase the scratch buffer size, if we need it.
    scratchBufferSizeNeeded = std::max(blasPrebuildInfo.ScratchDataSizeInBytes, scratchBufferSizeNeeded);
}

// Now that we know the size, we can finally create the scratch buffer.
scratchBuffer.Create(L"Acceleration Structure Scratch Buffer", static_cast<UINT>(scratchBufferSizeNeeded), 1);
```

With our scratch buffer created and our BLAS descriptors set up, we can create the BLASes:
```cpp
std::vector<D3D12_RAYTRACING_INSTANCE_DESC> instanceDescs(numBottomLevels);
blases.resize(numBottomLevels);
for (UINT i = 0; i < blasDescs.size(); i++)
{
    auto& blas = blases[i];

    // Create the BLAS
    auto bottomLevelDesc = CD3DX12_RESOURCE_DESC::Buffer(blasSize[i], D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS);
    g_Device->CreateCommittedResource(
        &defaultHeapDesc,
        D3D12_HEAP_FLAG_NONE,
        &bottomLevelDesc,
        D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE,
        nullptr,
        IID_PPV_ARGS(&blas));

    blasDescs[i].DestAccelerationStructureData = blas->GetGPUVirtualAddress();
    blasDescs[i].ScratchAccelerationStructureData = scratchBuffer.GetGpuVirtualAddress();

    D3D12_RAYTRACING_INSTANCE_DESC& instanceDesc = instanceDescs[i];
    UINT descriptorIndex = descriptorHeap->AllocateBufferUav(*blas.Get());

    // Identity matrix
    ZeroMemory(instanceDesc.Transform, sizeof(instanceDesc.Transform));
    instanceDesc.Transform[0][0] = 1.0f;
    instanceDesc.Transform[1][1] = 1.0f;
    instanceDesc.Transform[2][2] = 1.0f;

    instanceDesc.AccelerationStructure = blases[i]->GetGPUVirtualAddress();
    instanceDesc.Flags = 0;
    instanceDesc.InstanceID = 0;
    instanceDesc.InstanceMask = 1;
    instanceDesc.InstanceContributionToHitGroupIndex = i;
}

// We create a buffer to hold all of our BLAS instances.
instanceDataBuffer.Create(L"Instance Data Buffer", numBottomLevels, sizeof(D3D12_RAYTRACING_INSTANCE_DESC), instanceDescs.data());
```

## Top-Level Acceleration Structure
The top-level acceleration structure could be seen as an acceleration structure of acceleration structures. It holds instances of BLASes, each with their own transform matrix, so that it can correctly placed it in the world.

Now that we have our BLASes created, we can create the TLAS.
```cpp
// Specify where the instance data buffer is located.
tlasInputs.InstanceDescs = instanceDataBuffer.GetGpuVirtualAddress();
tlasInputs.DescsLayout = D3D12_ELEMENTS_LAYOUT_ARRAY;

// With all the necessary buffers set up and structures filled in,
// we can finally tell the GPU to build our acceleration structure

// Create the BLASes
for (UINT i = 0; i < blasDescs.size(); i++)
{
    pRaytracingCommandList->BuildRaytracingAccelerationStructure(&blasDescs[i], 0, nullptr);
}
// Create the TLAS
pRaytracingCommandList->BuildRaytracingAccelerationStructure(&tlasDesc, 0, nullptr);
```

The full source code of this example can be found [here](https://github.com/SeppahBaws/DirectX-Raytracing/blob/main/MiniEngine/AdaptiveSampling/RaytracingHelpers/AccelerationStructureBuilder.cpp). I mostly put everything into this post, but it might be more useful to see the full source code.

## Acceleration Structure Refitting
If we want to animate our scenes now, we would need to completely rebuild the acceleration structure from scratch. As you can guess, this would cause a huge performance impact. Luckily we can avoid this by "refitting" the TLAS, which is much faster than a complete rebuild.

Since the TLAS simply stores BLASes along with a transformation matrix, we can simply update the transformation matrices for the BLASes that we want to animate, and refit the acceleration structure.

Since refitting was out of scope for my internship and this post, I kindly refer you to the [NVIDIA DXR tutorial on refitting](https://developer.nvidia.com/rtx/raytracing/dxr/dx12-raytracing-tutorial/extra/dxr_tutorial_extra_refit).


# Descriptor Heap
A descriptor heap is a collection of resource views. Its purpose is to group the majority of memory allocations for the resource views together. We can then create resource views for the shaders from this heap.

It is important to note that not all resource views can be created from the same descriptor heap: SRVs, UAVs and CBVs can be created from the same heap, but RTVs and Sampler views each need their own separate heap.

```cpp
// Create our descriptor heap:
D3D12_DESCRIPTOR_HEAP_DESC descriptorHeapDesc = {};
descriptorHeapDesc.NumDescriptors = 10; // How many you need
descriptorHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
descriptorHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;

g_Device->CreateDescriptorHeap(&descriptorHeapDesc, IID_PPV_ARGS(&m_DescriptorHeap));

// Get the handle so we can write to it on the CPU side
D3D12_CPU_DESCRIPTOR_HANDLE descHandle = m_DescriptorHeap->GetCPUDescriptorHandleForHeapStart();

// Now we can describe what our descriptor view should look like:

// Add TLAS as SRV
descHandle.ptr += g_Device->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);

// In this case we're creating our acceleration structure resource view:
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc;
srvDesc.Format = DXGI_FORMAT_UNKNOWN;
srvDesc.ViewDimension = D3D12_SRV_DIMENSION_RAYTRACING_ACCELERATION_STRUCTURE;
srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.RaytracingAccelerationStructure.Location = scene.m_InstanceDataBuffer->GetGPUVirtualAddress(); // GPU address to the acceleration instance data buffer.
g_Device->CreateShaderResourceView(nullptr, &srvDesc, descHandle);
```


# Shader (Binding) Tables
In a rasterized pipeline, we always know what part of the scene we're rendering at any one point. However, we don't have that luxury in raytracing, since two rays might get bounced around, hitting two totally different objects (e.g. one just a normal mesh, the other might hit a transparent object, etc...).

This is why we need a Shader Table: it holds all the shaders we might possibly need during raytracing. While raytracing, the API then indexes into the shader tables and uses the specified shader, depending on the current context (Did the ray hit anything? If yes, *what* did we hit? Was it a normal triangle? Was it something which requires a custom intersection shader? Which hit shader should I even run? etc...)


In the simple case of a hello-dxr, we only need 3 byte address buffers to store our shader tables:
```cpp
ByteAddressBuffer m_RayGenShaderTable;
ByteAddressBuffer m_MissShaderTable;
ByteAddressBuffer m_HitShaderTable;
```

We can now start creating our shader tables. We'll need some extra helpers to get started first though:
```cpp
const UINT shaderTableSize = D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES;

ID3D12StateObjectProperties* stateObjectProperties = nullptr;
ASSERT_SUCCEEDED(pPSO->QueryInterface(IID_PPV_ARGS(&stateObjectProperties)));
```

We can now create the ray generation shader table - the miss shader table will be the exact same, except for a different shader identifier and a different ByteAddressBuffer:
```cpp
const UINT alignment = 16; // We need to align it to 16 bytes.
// A vector of bytes to hold our aligned table.
// We add alignment - 1 to the initial size, so that we have room to pad our bytes.
std::vector<BYTE> alignedShaderTableData(shaderTableSize + alignment - 1);

// Now we can get an aligned pointer into the bytes, where we will then write our shader table
BYTE* pAlignedShaderTableData = alignedShaderTableData.data() + ((UINT64)alignedShaderTableData.data() % alignment);

// For the ray generation table and miss table, we'll only need the shader identifier
// The `rayGenExportName` parameter here is the export name of the shader. This has to be the same as the one that you passed into the PSO
void* pRayGenShaderData = stateObjectProperties->GetShaderIdentifier(rayGenExportName);

// Copy our shader data into the aligned portion of the vector.
memcpy(pAlignedShaderTableData, pRayGenShaderData, shaderTableSize);

// We can now create the ByteAddressBuffer with MiniEngine's helpers:
m_RayGenShaderTable.Create(
    L"Ray Gen Shader Table",       // The name of the buffer
    1,                             // The amount of elements
    shaderTableSize,               // The size of each element
    alignedShaderTableData.data()  // The initial data
);
```

Creating the hit shader table is a bit more complicated, and will vary depending on what data you need access to in the hit shader.
A simple pass where you output the mesh texture color might look something like this:
```cpp
// The size of the shader identifier
const UINT shaderIdentifierSize = D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES;
// The offset in the blob to the descriptor handle
const UINT offsetToDescriptorHandle = ALIGN(sizeof(D3D12_GPU_DESCRIPTOR_HANDLE), shaderIdentifierSize);
// THe offset in the blob to the material constants
const UINT offsetToMaterialConstants = ALIGN(sizeof(UINT32), offsetToDescriptorHandle + sizeof(D3D12_GPU_DESCRIPTOR_HANDLE));
// The size of one shader record
const UINT shaderRecordSizeInBytes = ALIGN(D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT, offsetToMaterialConstants + sizeof(MeshRootConstant));


// This is a temporary buffer where we will write the shader record table to.
// We will have as many shader records as we have meshes.
// In my case, a "scene" only consisted of a single model with one or more meshes.
std::vector<byte> pHitShaderTable(shaderRecordSizeInBytes * model.m_Header.meshCount);

// Get the shader identifier:
void* pHitGroupIdentifierData = stateObjectProperties->GetShaderIdentifier(DEFAULT_HIT_GROUP_NAME);

for (UINT i = 0; i < model.m_Header.meshCount; i++)
{
    // First entry in the record: the hit group for which this entry is
    byte* pShaderRecord = i * shaderRecordSizeInBytes + pHitShaderTable.data();
    memcpy(pShaderRecord, pHitGroupIdentifierData, shaderIdentifierSize);

    // Second entry: shader descriptors (textures etc)
    UINT materialIndex = model.m_pMesh[i].materialIndex;
    memcpy(pShaderRecord + offsetToDescriptorHandle,
        &scene.m_ModelDescriptors[materialIndex].ptr,
        sizeof(scene.m_ModelDescriptors[materialIndex].ptr)
    );

    // Third entry: mesh id (used to query mesh data like UVs in the shader)
    MeshRootConstant meshConst;
    meshConst.meshId = i;
    memcpy(pShaderRecord + offsetToMaterialConstants,
        &meshConst,
        sizeof(meshConst)
    );
}

// Now we can create the ByteAddressBuffer with `pHitGroupIdentifierData`,
// exactly the same way we did for the ray gen table and miss table
```


# Raytracing pipeline
The raytracing pipeline groups all the objects together that are required to kick off the raytracing. These are: the shaders, hit groups, shader associations and the global root signature.
The DirectX 12 helpers library (available at: [microsoft/DirectX-Headers](https://github.com/microsoft/DirectX-Headers/blob/main/include/directx/d3dx12.h)) is a very useful tool to quickly set this up, because it abstracts away a lot of boilerplate code.

To create a pipeline, we start off by creating the pipeline descriptor:
```cpp
CD3DX12_STATE_OBJECT_DESC raytracingPipeline{ D3D12_STATE_OBJECT_TYPE_RAYTRACING_PIPELINE };
```

Then, we add our shaders to the pipeline: (repeat this for each shader)
```cpp
auto shaderLib = raytracingPipeline.CreateSubobject<CD3DX12_DXIL_LIBRARY_SUBOBJECT>();
D3D12_SHADER_BYTECODE shaderDxil = CD3DX12_SHADER_BYTECODE((void*)g_pShader, ARRAYSIZE(g_pShader));
shaderLib->SetDXILLibrary(&shaderDxil);
shaderLib->DefineExport(L"ShaderMainFunction");
```
`g_pShader` is the bytecode array which is output by the shader compiler in Visual Studio, and `L"ShaderMainFunction"` is the name of the main function in your shader. This is the one with the shader identifier attribute.

After that, we add the hit group(s): (again, repeat for each group)
```cpp
auto hitGroup = raytracingPipeline.CreateSubobject<CD3DX12_HIT_GROUP_SUBOBJECT>();
hitGroup->SetClosestHitShaderImport(L"..."); // Closest hit shader main function name.
hitGroup->SetHitGroupExport(L"..."); // Name to identify this hit group as.
hitGroup->SetHitGroupType(D3D12_HIT_GROUP_TYPE_TRIANGLES); // Can also be D3D12_HIT_GROUP_TYPE_PROCEDURAL_PRIMITIVE.
```

Next, we add the shader config:
```cpp
auto shaderConfig = raytracingPipeline.CreateSubobject<CD3DX12_RAYTRACING_SHADER_CONFIG_SUBOBJECT>();
UINT payloadSize = 1 * sizeof(float); // float rayHitT
UINT attributeSize = 2 * sizeof(float); // float2 barycentrics
shaderConfig->Config(payloadSize, attributeSize);
```
In this example, the ray payload only has 1 float for the distance, and we're using the built-in attributes (`BuiltInTriangleIntersectionAttributes`) which only has a `float2 barycentrics` as data. ([spec](https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html#intersection-attributes-structure))

Next, we add the local root signatures: (repeat for each shader)
```cpp
auto shaderLocalRootSig = raytracingPipeline.CreateSubobject<CD3DX12_LOCAL_ROOT_SIGNATURE_SUBOBJECT>();
shaderLocalRootSig->SetRootSignature(m_ShaderLocalRootSig.GetSignature());
```
`m_ShaderLocalRootSig` is of type `RootSignature` which is provided by the MiniEngine, and filled in later in the code

Then, we move on to adding the shader associations: (repeat for each shader)
```cpp
auto shaderAssoc = raytracingPipeline.CreateSubobject<CD3DX12_SUBOBJECT_TO_EXPORTS_ASSOCIATION_SUBOBJECT>();
shaderAssoc->SetSubobjectToAssociate(*shaderLocalRootSig); // Root signature subobject from previous step.
shaderAssoc->AddExport(L"..."); // Shader export name.
```

Next, we bind the global root signature:
```cpp
auto globalRootSig = raytracingPipeline.CreateSubobject<CD3DX12_GLOBAL_ROOT_SIGNATURE_SUBOBJECT>();
globalRootSig->SetRootSignature(m_RTGlobalRootSig.GetSignature());
```
Similar as the local root signatures, the global root signature is of type `RootSignature`, which is provided by the MiniEngine, and initialized earlier in the code.

Finally, we bind the pipeline configuration:
```cpp
auto pipelineConfig = raytracingPipeline.CreateSubobject<CD3DX12_RAYTRACING_PIPELINE_CONFIG_SUBOBJECT>();
UINT maxRecursionDepth = 1;
pipelineConfig->Config(maxRecursionDepth);
```

The `maxRecursionDepth` defines how much recursion we can have (some effects like reflections need to trace secondary rays from a ray hit point). In my example I only used primary rays, so a maxRecursionDepth of 1 worked just fine.

Now that everything is bound to the pipeline descriptor, we can finally build the pipeline:
```cpp
HRESULT hr = m_pRTDevice->CreateStateObject(raytracingPipeline, IID_PPV_ARGS(&m_RaytracingPSO));
// Check hr for errors
```
This can fail, so make sure you check the HRESULT for errors!


# Binding data
Binding data to the global root signature is pretty easy; right before calling `DispatchRays()`, all you have to do is bind the root signature, and then bind all the global root signature parameters:
```cpp
// Bind the root signature.
commandList->SetComputeRootSignature(m_GlobalRTRootSignature);
// Bind the global root parameters
commandList->SetComputeRootDescriptorTable(0, m_RTOutputUAV);
commandList->SetComputeRootShaderResourceView(1, m_TLAS->GetGPUVirtualAddress());
// etc...
```


Passing data to a local root signature is a bit more difficult, because you have to bind the data to a shader table, which can hold any arbitrary data. You are expected to set the memory directly, but it's not too difficult with some pointer offset tricks.

My shader table looks like this:

| Shader record name | Size |
| --- | --- |
| hit group identifier | D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES (defined as 32) |
| material textures | sizeof(D3D12_GPU_DESCRIPTOR_HANDLE) |
| mesh id | sizeof(UINT32) |

Now that we know what our entries are and how big they are, we can calculate the offset of a shader record by just adding all the sizes of the previous shader records together.

Important to note is the first shader record: the `hit group identifier`. It is a mandatory field, and DXR expects us to set it, so that DXR knows to which hit group this shader stage belongs to.


# Invoking the raytracing
To render everything correctly, we first have to make sure to bind our output buffer as our render target:
```cpp
// Transition the output buffer to a UAV
gfxContext.TransitionResource(m_RaytracingOutput, D3D12_RESOURCE_STATE_UNORDERED_ACCESS);
gfxContext.FlushResourceBarriers(); // Wait until m_RaytracingOutput has finished transitioning
gfxContext.SetRenderTarget(m_RaytracingOutput.GetRTV());
```

As you can see, we also need to transition the output buffer to be a UAV, so that we can properly access it and write to it from the shaders.

Next up we bind the descriptor heap:
```cpp
ID3D12DescriptorHeap* pDescriptorHeaps[] = { &m_pRTDescHeap->GetDescriptorHeap() };
rtCommandList->SetDescriptorHeaps(ARRAYSIZE(pDescriptorHeaps), pDescriptorHeaps);
```

After that, we bind the global root signature and its parameters
```cpp
commandList->SetComputeRootSignature(m_RTGlobalRootSig.GetSignature());
// Bind the root parameters by using the SetComputeRoot* functions
// on the command list, e.g.:
commandList->SetComputeRootDescriptorTable(0, m_RTOutputUAV);
commandList->SetComputeRootShaderResourceView(1, m_TLAS->GetGPUVirtualAddress());
// etc...
```

Then we can bind our raytracing pipeline:
```cpp
rtCommandList->SetPipelineState1(m_RaytracingPSO.Get());
```

Now that we have everything we need, we can finally tell DXR to kick off the raytracing. In order to do this, we have to fill in the `D3D12_DISPATCH_RAYS_DESC` struct, which will hold our shader tables and thread grid dimensions (similar to compute shaders).
```cpp
D3D12_DISPATCH_RAYS_DESC dispatchRaysDesc = {};

// Ray generation shader record
dispatchRaysDesc.RayGenerationShaderRecord.StartAddress = m_RayGenShaderTable.GetGpuVirtualAddress();
dispatchRaysDesc.RayGenerationShaderRecord.SizeInBytes = m_RayGenShaderTable.GetBufferSize();

// Hit group table
dispatchRaysDesc.HitGroupTable.StartAddress = m_HitShaderTable.GetGpuVirtualAddress();
dispatchRaysDesc.HitGroupTable.SizeInBytes = m_HitShaderTable.GetBufferSize();
dispatchRaysDesc.HitGroupTable.StrideInBytes = m_HitGroupStride;

// Miss table
dispatchRaysDesc.MissShaderTable.StartAddress = m_MissShaderTable.GetGpuVirtualAddress();
dispatchRaysDesc.MissShaderTable.SizeInBytes = m_MissShaderTable.GetBufferSize();
dispatchRaysDesc.MissShaderTable.StrideInBytes = dispatchRaysDesc.MissShaderTable.SizeInBytes; // Only one entry

// The dimensions to dispatch the rays over
dispatchRaysDesc.Width = m_RaytracingOutput.GetWidth();
dispatchRaysDesc.Height = m_RaytracingOutput.GetHeight();
dispatchRaysDesc.Depth = 1;
```

Once we have that filled in, we can finally call `DispatchRays`:
```cpp
rtCommandList->DispatchRays(&dispatchRaysDesc);
```


When we're done with raytracing, we can now use our raytracing output buffer however we like. In my example, I simply copied it to MiniEngine's "backbuffer":
```cpp
// Make sure the resources are in the correct state
gfxContext.TransitionResource(m_RaytracingOutput, D3D12_RESOURCE_STATE_COPY_SOURCE);
gfxContext.TransitionResource(g_SceneColorBuffer, D3D12_RESOURCE_STATE_COPY_DEST);

// Wait for them to finish transitioning
gfxContext.FlushResourceBarriers();

// Issue the copy command
gfxContext.CopyBuffer(g_SceneColorBuffer, m_RaytracingOutput);

// Transition the back buffer back to a render target
gfxContext.TransitionResource(g_SceneColorBuffer, D3D12_RESOURCE_STATE_RENDER_TARGET);
// We transition m_RaytracingOutput at the beginning of the raytracing pass,
// so no need to transition it here *again*.
```



# Conclusion

And that's it! If everything went well, you should now have your scene being raytraced.
In case you got stuck on something, I recommend going through the references linked below. They were the major ones I used when setting up DXR for the first time.
I also point you to [my own repository](https://github.com/SeppahBaws/DirectX-Raytracing) where you can find all the source code I wrote during my internship. Everything in there should be working fine(tm).

If you haven't yet, I also recommend giving [my previous post]({{< ref "/posts/ramblings-on-games-rendering-and-real-time-raytracing.md" >}}) a read, where I ramble about raytracing in games and stuff. If this post was a bit too much over your head, I think you'll enjoy that one. :)


# References

These are my most-used documentation pages and tutorials that I used while getting my feet wet with DXR.
I highly recommend going through these, because they contain a ton of information.
- [DirectX Raytracing (DXR) Functional Spec](https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html)
- [DirectX-Specs](https://microsoft.github.io/DirectX-Specs/d3d/ResourceBinding.html#descriptor-heaps)
- [DX12 Raytracing tutorial - Part 1](https://developer.nvidia.com/rtx/raytracing/dxr/dx12-raytracing-tutorial-part-1#accelerationstructure)
- [Learning DirectX 12](https://www.3dgep.com/learning-directx-12-1/#Create_a_Descriptor_Heap)
- [ID3D12Device5 - Win32 apps](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nn-d3d12-id3d12device5)
- [Root Signatures Overview - Win32 apps](https://docs.microsoft.com/en-us/windows/win32/direct3d12/root-signatures-overview)
- [Direct3D 12 Raytracing HLSL Shaders - Win32 apps](https://docs.microsoft.com/en-us/windows/win32/direct3d12/direct3d-12-raytracing-hlsl-shaders)
- [D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC (d3d12.h) - Win32 apps](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_build_raytracing_acceleration_structure_desc)
- [D3D12_RAYTRACING_GEOMETRY_DESC (d3d12.h) - Win32 apps](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_raytracing_geometry_desc)
- [Descriptor Heaps Overview - Win32 apps](https://docs.microsoft.com/en-us/windows/win32/direct3d12/descriptor-heaps-overview)
