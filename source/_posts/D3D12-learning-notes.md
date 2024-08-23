---
title: D3D12_learning_notes
date: 2023-10-18 11:47:10
tags:
mathjax : true
---



# Hello World
Describe the process of drawing a triangle, something similar to printf("Hello World!") in D3D.

## step 1: vertex and input layout

1. First thing is create a custom vertex structure.
```C++
struct Vertex
{
XMFLOAT3 Pos;
XMFLOAT3 Normal;
XMFLOAT2 Tex0;
XMFLOAT2 Tex1;
};
```

2. After that, we need to give D3D a description, to let D3D know what the vertex is doing with each components (ep. Pos, Normal), which is `D3D12_INPUT_LAYOUT_DESC`.

```c++
typedef struct D3D12_INPUT_LAYOUT_DESC
{
const D3D12_INPUT_ELEMENT_DESC *pInputElementDescs;
UINT NumElements;
} D3D12_INPUT_LAYOUT_DESC;
```

3. Each `D3D12_INPUT_ELEMENT_DESC` is defined as below 

```C++
typedef struct D3D12_INPUT_ELEMENT_DESC
{
LPCSTR SemanticName;
UINT SemanticIndex;
DXGI_FORMAT Format;
UINT InputSlot;
UINT AlignedByteOffset;
D3D12_INPUT_CLASSIFICATION InputSlotClass;
UINT InstanceDataStepRate;
} D3D12_INPUT_ELEMENT_DESC;
```
Below is each parameters' description:
```C++
struct Vertex
{
XMFLOAT3 Pos;
XMFLOAT3 Normal;
XMFLOAT2 Tex0;
XMFLOAT2 Tex1;
};
D3D11_INPUT_ELEMENT_DESC vertexDesc[] =
{
{"POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0,
D3D11_INPUT_PER_VERTEX_DATA, 0},
{"NORMAL", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 12,
D3D11_INPUT_PER_VERTEX_DATA, 0},
{"TEXCOORD", 0, DXGI_FORMAT_R32G32_FLOAT, 0, 24,
D3D11_INPUT_PER_VERTEX_DATA, 0},
{"TEXCOORD", 1, DXGI_FORMAT_R32G32_FLOAT, 0, 32,
D3D11_INPUT_PER_VERTEX_DATA, 0}
};
VertexOut VS(float3 iPos : POSITION,
float3 iNormal : NORMAL,
float2 iTex0 : TEXCOORD0,
float2 iTex1 : TEXCOORD1)
```
* SemanticName: Name of current component, which is used to map elements in vertex shader input signature. A example below: 
* SemanticIndex: Index of Semantic, ep Pos is become POSITION0.
* Format: The format of the component, because XMFLOAT3 is still not enough detailed.
* InputSlot: Totally 12 slot, leave it later to learn.
* AlignedByteOffset: Size of float: 4 bytes, so offset is 3 x 4 = 12.
* InputSlotClass: default right now : `InputSlotClass`.

## step 2: vertex buffer

We need to let GPU access the vertex array, they need to place into a GPU resource`ID3D12Resource` called `buffer`. Its a simpler structure to compare with texture.

```C++
static inline CD3DX12_RESOURCE_DESC Buffer(
UINT64 width,
D3D12_RESOURCE_FLAGS flags = D3D12_RESOURCE_FLAG_NONE,
UINT64 alignment = 0 )
{
return CD3DX12_RESOURCE_DESC( D3D12_RESOURCE_DIMENSION_BUFFER,
alignment, width, 1, 1, 1,
DXGI_FORMAT_UNKNOWN, 1, 0,
D3D12_TEXTURE_LAYOUT_ROW_MAJOR, flags );
}
```

For a buffer the `width`refers to the number of bytes in the buffer. If totally 64 floats, the width should be `64*sizeof(float)`


### let GPU access the data
$\textbf{If}$ Vertex buffer is static(house,road,etc..), we need to put it into default heap `D3D12_HEAP_TYPE_DEFAULT`. But default heap can only be accessed by GPU, so need a tools to let CPU push the data into GPU. Upload buffer `D3D12_HEAP_TYPE_UPLOAD` needed. right here. The step is  $\textbf{system memory}\rarr\textbf{upload heap}\rarr\textbf{default heap}$

$\textbf{Whole Default Heap Below}$ 
```C++
Microsoft::WRL::ComPtr<ID3D12Resource> d3dUtil::CreateDefaultBuffer(
ID3D12Device* device,
ID3D12GraphicsCommandList* cmdList,
const void* initData,
UINT64 byteSize,
Microsoft::WRL::ComPtr<ID3D12Resource>& uploadBuffer)
{
ComPtr<ID3D12Resource> defaultBuffer;
// Create the actual default buffer resource.
ThrowIfFailed(device->CreateCommittedResource(
&CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
D3D12_HEAP_FLAG_NONE,
&CD3DX12_RESOURCE_DESC::Buffer(byteSize),
D3D12_RESOURCE_STATE_COMMON,
nullptr,
IID_PPV_ARGS(defaultBuffer.GetAddressOf())));
// In order to copy CPU memory data into our default buffer, we need
// to create an intermediate upload heap.
ThrowIfFailed(device->CreateCommittedResource(
&CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
D3D12_HEAP_FLAG_NONE,
&CD3DX12_RESOURCE_DESC::Buffer(byteSize),
D3D12_RESOURCE_STATE_GENERIC_READ, nullptr,
IID_PPV_ARGS(uploadBuffer.GetAddressOf())));
// Describe the data we want to copy into the default buffer.
D3D12_SUBRESOURCE_DATA subResourceData = {};
subResourceData.pData = initData;
subResourceData.RowPitch = byteSize;
subResourceData.SlicePitch = subResourceData.RowPitch;
// Schedule to copy the data to the default buffer resource.
// At a high level, the helper function UpdateSubresources
// will copy the CPU memory into the intermediate upload heap.
// Then, using ID3D12CommandList::CopySubresourceRegion,
// the intermediate upload heap data will be copied to mBuffer.
cmdList->ResourceBarrier(1,
&CD3DX12_RESOURCE_BARRIER::Transition(defaultBuffer.Get(),
D3D12_RESOURCE_STATE_COMMON,
D3D12_RESOURCE_STATE_COPY_DEST));
UpdateSubresources<1>(cmdList,
defaultBuffer.Get(), uploadBuffer.Get(),
0, 0, 1, &subResourceData);
cmdList->ResourceBarrier(1,
&CD3DX12_RESOURCE_BARRIER::Transition(defaultBuffer.Get(),
D3D12_RESOURCE_STATE_COPY_DEST,
D3D12_RESOURCE_STATE_GENERIC_READ));
// Note: uploadBuffer has to be kept alive after the above function
// calls because the command list has not been executed yet that
// performs the actual copy.
// The caller can Release the uploadBuffer after it knows the copy
// has been executed.
return defaultBuffer;
}

```
```C++
typedef struct D3D12_SUBRESOURCE_DATA
{
const void *pData;
LONG_PTR RowPitch;
LONG_PTR SlicePitch;
} D3D12_SUBRESOURCE_DATA;
```
Above is how to manage upload heap -> default heap, then we just need to call the function above to store the vertex buffer, it makes the life easier. Example of cube below:

```C++
Vertex vertices[] =
{
{ XMFLOAT3(-1.0f, -1.0f, -1.0f), XMFLOAT4(Colors::White) },
{ XMFLOAT3(-1.0f, +1.0f, -1.0f), XMFLOAT4(Colors::Black) },
{ XMFLOAT3(+1.0f, +1.0f, -1.0f), XMFLOAT4(Colors::Red) },
{ XMFLOAT3(+1.0f, -1.0f, -1.0f), XMFLOAT4(Colors::Green) },
{ XMFLOAT3(-1.0f, -1.0f, +1.0f), XMFLOAT4(Colors::Blue) },
{ XMFLOAT3(-1.0f, +1.0f, +1.0f), XMFLOAT4(Colors::Yellow) },
{ XMFLOAT3(+1.0f, +1.0f, +1.0f), XMFLOAT4(Colors::Cyan) },
{ XMFLOAT3(+1.0f, -1.0f, +1.0f), XMFLOAT4(Colors::Magenta) }
};
const UINT64 vbByteSize = 8 * sizeof(Vertex);
ComPtr<ID3D12Resource> VertexBufferGPU = nullptr;
ComPtr<ID3D12Resource> VertexBufferUploader = nullptr;
VertexBufferGPU = d3dUtil::CreateDefaultBuffer(md3dDevice.Get(),
mCommandList.Get(), vertices, vbByteSize, VertexBufferUploader);
```


### bind the buffer to pipeline

1. We need a `vertex buffer view` to vertex buffer resource.
```C++
typedef struct D3D12_VERTEX_BUFFER_VIEW
{
D3D12_GPU_VIRTUAL_ADDRESS BufferLocation;
UINT SizeInBytes;
UINT StrideInBytes;
}
```
* BufferLocation: Virtual address (such as & in c) of the vertex buffer we wanna to view
* SizeInBytes: The number of bytes to view in the vertex buffer starting from `BufferLocation`.
* StrideInBytes: The size of each vertex in bytes.

2. After create vertex buffer and view, we need to combine them.




# graphics pipeline in DirectX 12.
![](D3D12-learning-notes/3.png)
1. Fixed-function stages (blue): cannot change how they process data, but can configure them using the DirectX 12 API. Such as imachines in a factory.
2. Programmable stages(green): can write a shadow program like HLSL to define exactly how data is processed. Such as a program a robot in a factory.

* Input-Assembler(IA) stage: read primitive data from user-defined vertex and index buffers and assemble that data into geometric primitives.

* Vertex Shader(VS) Stage
transform the vertex data from object-space into clip-space.

* Hull Shader(HS) Stage
It is responsible for determining how much an input control patch should be tessellated by the tesslation.

# basics of D3D12

## COM 
1. backward compatible.
2. language-independent.

Use WRL below to manager the lifetime of COM object, like smart pointer.
```C++
Microsoft::WRL::ComPtr
```

```C++
// Get
ComPtr<ID3D12RootSignature> mRootSignature;
...
// SetGraphicsRootSignature expects ID3D12RootSignature* argument.
mCommandList->SetGraphicsRootSignature(mRootSignature.Get());


// GetAddressOf
ComPtr<ID3D12CommandAllocator> mDirectCmdListAlloc;
...
ThrowIfFailed(md3dDevice->CreateCommandAllocator(
D3D12_COMMAND_LIST_TYPE_DIRECT,
mDirectCmdListAlloc.GetAddressOf()));


// Reset
ComPtr<ID3D11Device> device;
// first way
device.Reset();
// second way
device = nullptr;
```

## PSO

Pipeline State Object, in my understanding is analogous to the concept of draw state. Which means it can not change during the drawcall, a 'setup' before rendering process begin.

## Render Item
A lightweight structure for us to draw a shape, store based on different PSO.

```C++
// List of all the render items.
std::vector<std::unique_ptr<RenderItem>> mAllRitems;
// Render items divided by PSO.
std::vector<RenderItem*> mOpaqueRitems;
std::vector<RenderItem*> mTransparentRitems;
```


## useful functions

1. IDXGI Factory (DirectX Graphic Infrastructure)
Enum Adapters and Creating Swap Chain
```C++
	GRS_THROW_IF_FAILED(CreateDXGIFactory2(nDXGIFactoryFlags, IID_PPV_ARGS(&pIDXGIFactory5)));
```


## creating resources

### CreateCommittedResouce

implicit heap: the heap object can't be obtained by the application. Just call the heap and use it directly, do not need to build the heap manually. But hard to control the detail of the heap.


### CreatePlacedResource

### CreatReservedResource





## Heap 

```C++
typedef 
enum D3D12_HEAP_TYPE
{
    D3D12_HEAP_TYPE_DEFAULT            = 1, 
    D3D12_HEAP_TYPE_UPLOAD             = 2,
    D3D12_HEAP_TYPE_READBACK           = 3,
    D3D12_HEAP_TYPE_CUSTOM              = 4
} D3D12_HEAP_TYPE;
```

* DEFAULT: creating buffet when D3Dxx_USAGE = Default, only GPU  could access the data, CPU can not directly access the data. Which means it usually in $\textbf{video memory}$. Always insert some data hard to change in it, such as texture. 
* UPLOAD: GPU can not load the data, so upload heap is using to load the data in DEFAULT heap. For GPU "read only", For CPU "write only". For do not change.
* READBACK: the oppsite of UPLOAD


## Resource Barrier
Handle the parallelism problem between copy engine and graphic command engine. Ep. The texture is large enough and $\textbf{memcopy}$ need some time to copy. But the graphic command engine do not know that and already start $\textbf{Draw Call}$ the texture, which lead the unfinished texture to be rendered.

```C++
//send the command to the command heap of copy something from UPLOAD heap to DEFAULT heap
CD3DX12_TEXTURE_COPY_LOCATION Dst(pITexcute.Get(), 0);
CD3DX12_TEXTURE_COPY_LOCATION Src(pITextureUpload.Get(), stTxtLayouts);

// directly command a list's object.
pICommandList->CopyTextureRegion(&Dst, 0, 0, 0, &Src, nullptr);
 
// Resource Barrier
D3D12_RESOURCE_BARRIER stResBar = {};
stResBar.Type			= D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
stResBar.Flags			= D3D12_RESOURCE_BARRIER_FLAG_NONE;
stResBar.Transition.pResource	= pITexcute.Get();
stResBar.Transition.StateBefore = D3D12_RESOURCE_STATE_COPY_DEST;
stResBar.Transition.StateAfter	= D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;
stResBar.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
 
pICommandList->ResourceBarrier(1, &stResBar);
```

In my understanding, because command heap's excution on GPU is in serial order, which means rescource barrier is just like crossbars at supermarket checkout counters.


## Adapter

used to looking for a adapter(graphic card)
```C++
ComPtr<IDXGIAdapter4> GetAdapter(bool useWarp)
{
    ComPtr<IDXGIFactory4> dxgiFactory;
    UINT createFactoryFlags = 0;
#if defined(_DEBUG)
    createFactoryFlags = DXGI_CREATE_FACTORY_DEBUG;
#endif
 
    ThrowIfFailed(CreateDXGIFactory2(createFactoryFlags, IID_PPV_ARGS(&dxgiFactory)));
   ComPtr<IDXGIAdapter1> dxgiAdapter1;
    ComPtr<IDXGIAdapter4> dxgiAdapter4;

    if (useWarp)
    {
        ThrowIfFailed(dxgiFactory->EnumWarpAdapter(IID_PPV_ARGS(&dxgiAdapter1)));
        ThrowIfFailed(dxgiAdapter1.As(&dxgiAdapter4));
    }
    else
    {
        SIZE_T maxDedicatedVideoMemory = 0;
        for (UINT i = 0; dxgiFactory->EnumAdapters1(i, &dxgiAdapter1) != DXGI_ERROR_NOT_FOUND; ++i)
        {
            DXGI_ADAPTER_DESC1 dxgiAdapterDesc1;
            dxgiAdapter1->GetDesc1(&dxgiAdapterDesc1);
 
            // Check to see if the adapter can create a D3D12 device without actually 
            // creating it. The adapter with the largest dedicated video memory
            // is favored.
            if ((dxgiAdapterDesc1.Flags & DXGI_ADAPTER_FLAG_SOFTWARE) == 0 &&
                SUCCEEDED(D3D12CreateDevice(dxgiAdapter1.Get(), 
                    D3D_FEATURE_LEVEL_11_0, __uuidof(ID3D12Device), nullptr)) && 
                dxgiAdapterDesc1.DedicatedVideoMemory > maxDedicatedVideoMemory )
            {
                maxDedicatedVideoMemory = dxgiAdapterDesc1.DedicatedVideoMemory;
                ThrowIfFailed(dxgiAdapter1.As(&dxgiAdapter4));
            }
        }
    }
 
    return dxgiAdapter4;
}
```





## Command List, Command Allocator, Command Queue

```C++
// DirectX 12 Objects
ComPtr<ID3D12Device2> g_Device;
ComPtr<ID3D12CommandQueue> g_CommandQueue;
ComPtr<IDXGISwapChain4> g_SwapChain;
ComPtr<ID3D12Resource> g_BackBuffers[g_NumFrames];
ComPtr<ID3D12GraphicsCommandList> g_CommandList;
ComPtr<ID3D12CommandAllocator> g_CommandAllocators[g_NumFrames];
ComPtr<ID3D12DescriptorHeap> g_RTVDescriptorHeap;
UINT g_RTVDescriptorSize;
UINT g_CurrentBackBufferIndex;
```



* Comptr: it goes out of scope when COM object is no longer needed, helping to prevent memory leaks.
* CommandAllocator: create and manage the memory that backs(supports) command list. Every command list need a command allocator, and each command allocator can be used with one command list at a time.

* Command List: CPU records a list of commands to be executed by GPU. Such as state changes, resource barriers, drawing operations...
* Command Queue: An interface through which CPU submits the recorded command lists to the GPU for execution. The GPU start excute the command as soon as CPU put command list in it. 


## Fence
![](D3D12-learning-notes/6.png)
A marker let you know when GPU has finished doing its work and tell CPU, so they can be synchronised.
```C++
// Synchronization objects
// a pointer used to ensure the synchronization primitive that the CPU can use to determine the eprogress of the GPU's execution of command lists.
ComPtr<ID3D12Fence> g_Fence;
// the next fence value to signal the command queue
uint64_t g_FenceValue = 0;
// each frame could be 'in-flight' on the command queue, this is used to keep tracked to guarantee that any resources that are still being referenced by the command queue are not overwritten.
uint64_t g_FrameFenceValues[g_NumFrames] = {};
// used to hold on untill the fance has reached a specific value.
HANDLE g_FenceEvent;
```


## Swap Chain
![](D3D12-learning-notes/4.png)
```C++
const unit8_t g_NumFrames = 3;
```
must more than 2 if using flip ppresentation model.


## Transformation Pipeline
1. World Transform: change each 3D model's coordinates into world coordinates.
2. View Transform: $V = T \cdot R_z \cdot R_y \cdot R_z$
3. Projection Transform: 

![](D3D12-learning-notes/1.png)
![](D3D12-learning-notes/2.png)

4. Clip transform: ignore the part not in the camera.


## Render Target View(RTV):
The purpose of it is just tell GPU how to render at back buffer before swap. If without RTV, the GPU will not know where the rendered pixel should be sent.
![](D3D12-learning-notes/5.png)
# glossary of CG

* mipmap: a set of pictures, with different level of pixels. Becasue off-site viewing do not need that detailed.

* SRV(shader resource view): wrapping textures in a format that the shadow can access them. Read Only. For example : a single texture, individual arrays, planes, or colors from a mipmapped texture, 3D texture, 1D texture color gradinets, etc.

* UAV(unordered access view): same as SRV, but can read or write in any order, even could read/written simultaneously by multipl,e threads without generate memory conflicts.

* root signatures: link command to the resources the shaders require. It determines the type of data the shaders should expect, but does not define the actural memory or data. For graphics command list has both a graphics and compute root signature, for compute command list have one compute root signature. These root signatures are independent of each others.

* Resource: all the resource could be excuted by GPU is resource in D3D12. Which is 'ID3D12Resource', such as rendering targets(include back buffers), textures, vertex buffers, index buffers... 
* G-SYNC: refresh screen and graph card together.
* Window Advanced Rasterization Platform(WARP): If did not find a valiable GPU, the system will do the same step of D3D12 by CPU by WARP. It can instead all the rendering method such as rasterization, ray tracing...
