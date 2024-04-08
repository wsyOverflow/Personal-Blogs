---
typora-copy-images-to: ..\..\..\images\Rendering Blogs\Graphics Basis\${filename}.assets
typora-root-url: ..\..\..\
title: SlabHash
keywords: Graphics, Interpolation
categories:
- [Rendering Blogs, Graphics, Interpolation]
mathjax: true
---

# 1 Initialization



# 2 Acceleration Structure

Vulkan使用一个two-level的加速结构格式来表示具有不同变换、材质属性的物体实例：

- Top-level acceleration structure (TLAS) 是实例的加速结构，每个实例指向一个BLAS与实例数据。实例数据包括变换、24bit instance ID、32 bit shader offset index
- Bottom-level acceleration structure (BLAS)可以是三角形的加速结构或者AABB的加速结构。前者适用于mesh；后者表示procedural object，由对应的intersection shader定义。

<img src="/images/Rendering Blogs/Graphics Basis/Vulkan Ray Tracing.assets/image-20240204142756523.png" alt="image-20240204142756523" style="zoom:80%;" />

<center>TLAS：包含3个实例，其中前两个实例都指向BLAS0，具有各自的实例数据。最后一个实例指向BLAS1。</center>

<img src="/images/Rendering Blogs/Graphics Basis/Vulkan Ray Tracing.assets/image-20240204142709802.png" alt="image-20240204142709802" style="zoom: 80%;" />

<center>BLAS：左图使用BVH结构表示BLAS；右图使用tree来表示BLAS。</center>

这种两级加速结构较好地平衡了灵活性与遍历性能。例如，变形mesh的BLAS可以被重建，而不需要重建整个场景，改变实例的transform只需要重建TLAS。场景中几何可能随时间发生变化，加速结构通常需要更改。加速结构的更改有两种：

- rebuild：当几何图元的数量或者拓扑结构发生改变时，需要rebuild
- refit：当移动加速结构中的图元时，可以使用refit。refit保留了树形的层级结构，比rebuild更快，但有可能会导致光追变慢。

## 2.1 BLAS的构建

BLAS的构建包括以下6个步骤：

1. 获取存储几何数据的buffer的 host/device address。使用 `vkGetBufferDeviceAddress`获取`VkBuffer`对象的地址。
2. 描述实例的几何信息
   - `VkAccelerationStructureGeometryKHR`：对几何数据的描述，指引builder如何访问几何数据，如地址、格式、步长
   - `VkAccelerationStructureBuildRangeInfoKHR`：描述物体在几何数据中的范围信息，起始顶点、图元数量等
3. 确定加速结构的存储开销以及构建时所需的scratch storage
   - `VkAccelerationStructureBuildGeometryInfoKHR`：指定要构建加速结构的几何数据描述的数组以及构建配置，描述一个加速结构的build信息
   - `vkGetAccelerationStructureBuildSizesKHR()`：查询得到size信息 `VkAccelerationStructureBuildSizesInfoKHR`
4. 创建空白的加速结构以及其下层的 `VkBuffer`：
   - 步骤3已经得到加速结构的大小信息，创建一个buffer用于加速结构的存储
   - `VkAccelerationStructureCreateInfoKHR`指定加速结构大小、buffer存储，调用`vkCreateAccelerationStructureKHR()`创建一个空白的加速结构，同时该空白加速结构作为build info的一个字段
5. 申请scratch space的buffer存储
   - 步骤3已经得到scratch data的大小信息，申请一个buffer用于存储构建过程中的临时数据
   - 作为build info的`scratchData.deviceAddress`字段
6. 构建几何的加速结构。使用以下信息调用 `vkCmdBuildAccelerationStructuresKHR()`，用于提交加速结构的build，可一次build多个加速结构
   - `VkAccelerationStructureBuildGeometryInfoKHR* pInfos`数组：每个元素描述了一个加速结构的几何数据信息与构建配置，数组长度为要构建的加速结构的数量
   - `VkAccelerationStructureBuildRangeInfoKHR** ppBuildRaneInfos`指针数组：每个指针为一个加速结构所需几何数据range info的数组
     加速结构`i`：几何数据信息与构建配置存放在`pInfos[i]`，而`pInfos[i]`中每个几何的range info存放在`ppBuildRangeInfos[i]`中，数组`ppBuildRangeInfos[i]`的长度为`pInfo[i].geometryCount`

### 2.1.1 描述几何数据

几何数据都多种类型，这里介绍triangle类型的几何数据描述。使用`VkAccelerationStructureGeometryTrianglesDataKHR`结构体来描述，用来告诉builder去哪里寻找三角形数据(vertex、index)、格式与长度。字段`transformData`用来指定几何的仿射变换，该变换会在BLAS构建前应用到顶点上，可以视为模型变换。几何数据的描述创建如下

```c++
std::vector<Vertex> objVertices = ...;
std::vector<uint32_t> objIndices = ...;
VkDeviceAddress vertexBufferAddress = ...;
VkDeviceAddress indexBufferAddress = ...;

VkAccelerationStructureGeometryTrianglesDataKHR triangles{VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_TRIANGLES_DATA_KHR};
triangles.vertexFormat = VK_FORMAT_R32G32B32_SFLOAT; 		// 顶点格式
triangles.vertexData.deviceAddress = vertexBufferAddress; 	// 存储顶点数据的buffer地址
triangles.vertexStride = 3 * sizeof(float);					// 顶点数据访问步长
triangles.indexType = VK_INDEX_TYPE_UINT32;					// 索引格式
triangles.indexData.deviceAddress = indexBufferAddress;		// 存放索引数据的buffer地址
triangles.maxVertex = uint32_t(objVertices.size() - 1);		// 顶点索引的最大值
triangles.transformData = {0}; // No transform
```

此外，可以通过将一个三角形的每个顶点的x坐标设置为 NaN 来标记为 inactive，意味着对光线不可见。

上述几何描述信息结构体不能直接送给builder，因为几何类型不止有三角形。为了能够兼容多种图元类型，使用 C 的多态类型方式定义了 `VkAccelerationStructureGeometryKHR`，其一个字段用于指定图元类型，一个字段指向实际描述几何数据的结构体，如下

```c++
VkAccelerationStructureGeometryKHR geometry{VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_KHR};
geometry.geometryType = VK_GEOMETRY_TYPE_TRIANGLES_KHR; // 三角形几何
geometry.geometry.triangles = triangles;				// 三角形几何数据描述
geometry.flags = VK_GEOMETRY_OPAQUE_BIT_KHR;			// 不透明物体，不会触发any-hit
```

只有几何数据的描述是不够的，因为有可能buffer是共用的，会包含多个物体的数据，因此还需要指定几何数据的范围，使用`VkAccelerationStructureBuildRangeInfoKHR` 来描述一个几何的数据在buffer里的范围

```c++
VkAccelerationStructureBuildRangeInfoKHR rangeInfo;
rangeInfo.firstVertex = 0;
rangeInfo.primitiveCount = uint32_t(objIndices.size() / 3);
rangeInfo.primitiveOffset = 0;
rangeInfo.transformOffset = 0;
```

### 2.1.2 加速结构的存储开销

加速结构的存储开销通过`vkGetAccelerationStructureBuildSizesKHR()`查询得到，查询之前需要`VkAccelerationStructureBuildGeometryInfoKHR`的部分信息，描述了指向几何数据描述的数组以及构建设置，其它为空。

```c++
VkAccelerationStructureBuildGeometryInfoKHR buildInfo{
VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_BUILD_GEOMETRY_INFO_KHR};
buildInfo.flags = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_TRACE_BIT_KHR;
buildInfo.geometryCount = 1; 		// 几何数据的数量
buildInfo.pGeometries = &geometry; 	// 指向几何数据描述的数组
buildInfo.mode = VK_BUILD_ACCELERATION_STRUCTURE_MODE_BUILD_KHR; 	// build 操作
buildInfo.type = VK_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL_KHR; 	// blas
buildInfo.srcAccelerationStructure = VK_NULL_HANDLE;
// We will set dstAccelerationStructure and scratchData once we have created those objects.

VkAccelerationStructureBuildSizesInfoKHR sizeInfo{VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_BUILD_SIZES_INFO_KHR};
vkGetAccelerationStructureBuildSizesKHR(
	device, // The device
	VK_ACCELERATION_STRUCTURE_BUILD_TYPE_DEVICE_KHR, // Build on device instead of host
	&buildInfo, // Pointer to build info
	&rangeInfo.primitiveCount, // Array of number of primitives per geometry
	&sizeInfo // Output pointer to store sizes
);
```

每个`rangeInfo.primitiveCount`用来描述`buildInfo中`的每个geometry的`primitiveCount`。`VkAccelerationStructureBuildGeometryInfoKHR`描述了构建一个BLAS所需的信息。

### 2.1.3 加速结构的存储分配

在创建加速结构之前，需要先创建一个buffer用作加速结构的备用存储，加速结构会存储到这个buffer中，这里使用了 `nvvk::AllocatorDma`。

```c++
// Allocate a buffer for the acceleration structure.
bufferBLAS = allocator.createBuffer(
	sizeInfo.accelerationStructureSize,
	VK_BUFFER_USAGE_ACCELERATION_STRUCTURE_STORAGE_BIT_KHR | VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT | 
    VK_BUFFER_USAGE_STORAGE_BUFFER_BIT
);
```

接下来通过`VkAccelerationStructureCreateInfoKHR`指定存储大小信息、加速结构buffer来创建空白加速结构

```c++
// Create an empty acceleration structure object.
VkAccelerationStructureCreateInfoKHR createInfo{VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_CREATE_INFO_KHR};
createInfo.type = buildInfo.type;
createInfo.size = sizeInfo.accelerationStructureSize;
createInfo.buffer = bufferBLAS.buffer;
createInfo.offset = 0;
NVVK_CHECK(vkCreateAccelerationStructureKHR(device, &createInfo, nullptr, &blas));
buildInfo.dstAccelerationStructure = blas; // 空白加速结构作为build info的一部分
```

### 2.1.4 为scratch data申请buffer存储

加速结构构建过程中，需要存放一些临时数据，即scratch data。因此需要为其申请buffer存储，

```c++
nvvk::Buffer scratchBuffer = allocator.createBuffer(sizeInfo.buildScratchSize,
	VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT);
buildInfo.scratchData.deviceAddress = GetBufferDeviceAddress(device, scratchBuffer.buffer);
```

### 2.1.5 构建加速结构

```c++
VkAccelerationStructureBuildRangeInfoKHR* pRangeInfo = &rangeInfo;
vkCmdBuildAccelerationStructuresKHR(
	cmdBuffer, 	// The command buffer to record the command
	1, 			// Number of acceleration structures to build
	&buildInfo, // Array of ...BuildGeometryInfoKHR objects
	&pRangeInfo // Array of ...RangeInfoKHR objects
);
```

一个`VkAccelerationStructureBuildGeometryInfoKHR`对应一个BLAS，而一个BLAS可能包含多个geometry，记录在`buildInfo`的`VkAccelerationStructureGeometryKHR`数组中。而每个geometry的数据位于buffer中的范围使用一个`VkAccelerationStructureBuildRangeInfoKHR`来描述。

## 2.2 TLAS的构建

TLAS的构建过程与BLAS类似，不同的是TLAS构建于instances.

### 2.2.1 描述instance数据

BLAS中需要先将几何数据（顶点、索引等）上传到GPU，TLAS构建前也需要先将instance数据上传。每个instance包含指向一个BLAS的地址，因此我们需要先获取BLAS的地址

```c++
VkAccelerationStructureDeviceAddressInfoKHR addressInfo{VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_DEVICE_ADDRESS_INFO_KHR};
addressInfo.accelerationStructure = blas;
VkDeviceAddress blasAddress= vkGetAccelerationStructureDeviceAddressKHR(device, &addressInfo);
```

指定一个instance需要`VkAccelerationStructureInstanceKHR`，`transform`字段是仿射变换，用于以不同的变换实例化BLAS，其作用过程如下
$$
M
\begin{pmatrix}
x \\
y \\
z \\
1
\end{pmatrix}
=
\begin{pmatrix}
M_{00}x+M_{01}y+M_{02}Z+M_03 \\
M_{10}x+M_{11}y+M_{12}Z+M_13 \\
M_{20}x+M_{21}y+M_{22}Z+M_23
\end{pmatrix}
$$
`instanceCustomIndex`字段存储了一个shader中可以访问的24-bit整数；`mask`字段存储了8位整数，光线与instance求交前会先计算光线mask & instance mask，只有不为0时才会继续求交操作；`instanceShaderBindingTableRecordOffset`是instance shaders在 SBT(shader binding table)的偏移量；`flag`字段用于feature的启用，这里用于禁止backface culling；`accelerationStructureReference`字段包含的BLAS的地址。instance数据如下：

```c++
VkAccelerationStructureInstanceKHR instance{};
// Set the instance transform to a 135-degree rotation around the y-axis.
const float rcpSqrt2 = sqrtf(0.5f);
instance.transform.matrix[0][0] = -rcpSqrt2;
instance.transform.matrix[0][2] = rcpSqrt2;
instance.transform.matrix[1][1] = 1.0f;
instance.transform.matrix[2][0] = -rcpSqrt2;
instance.transform.matrix[2][2] = -rcpSqrt2;
instance.instanceCustomIndex = 0;
instance.mask = 0xFF;
instance.instanceShaderBindingTableRecordOffset = 0;
instance.flags = VK_GEOMETRY_INSTANCE_TRIANGLE_FACING_CULL_DISABLE_BIT_KHR;
instance.accelerationStructureReference = blasAddress;
```

创建存储instance的buffer，并将instance数组上传该buffer中。

```c++
nvvk::Buffer bufferInstances = allocator.createBuffer(cmdBuffer, sizeof(instance), &instance, VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT);
// Add a memory barrier to ensure that createBuffer's upload command finishes before starting the TLAS build.
VkMemoryBarrier barrier{VK_STRUCTURE_TYPE_MEMORY_BARRIER};
barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
barrier.dstAccessMask = VK_ACCESS_ACCELERATION_STRUCTURE_WRITE_BIT_KHR;
vkCmdPipelineBarrier(cmdBuffer,
	VK_PIPELINE_STAGE_TRANSFER_BIT,
	VK_PIPELINE_STAGE_ACCELERATION_STRUCTURE_BUILD_BIT_KHR ,
	0,
	1, &barrier,
	0, nullptr,
	0, nullptr
);
```

与BLAS过程中的几何数据描述类似，TLAS使用`VkAccelerationStructureGeometryInstancesDataKHR`结构体来描述instance数据，其中的地址则指向存放instance数据的buffer，同样封装到C多态类型`VkAccelerationStructureGeometryKHR`中。

```c++
VkAccelerationStructureGeometryInstancesDataKHR instancesVk{
    VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_INSTANCES_DATA_KHR};
instancesVk.arrayOfPointers = VK_FALSE;
instancesVk.data.deviceAddress = GetBufferDeviceAddress(device, bufferInstances.buffer);
// Like creating the BLAS, point to the geometry (in this case, the instances) in a polymorphic object.
VkAccelerationStructureGeometryKHR geometry{VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_KHR};
geometry.geometryType = VK_GEOMETRY_TYPE_INSTANCES_KHR;
geometry.geometry.instances = instancesVk;
```

与BLAS的每个几何包含一定范围的顶点类似，每个TLAS单位可以包括多个instance，这里使用`VkAccelerationStructureBuildRangeInfoKHR`描述

```c++
VkAccelerationStructureBuildRangeInfoKHR rangeInfo;
rangeInfo.primitiveOffset = 0;
rangeInfo.primitiveCount = 1; // Number of instances
rangeInfo.firstVertex = 0;
rangeInfo.transformOffset = 0;
```

### 2.2.2 后续步骤与BLAS一致

```c++
// Create the build info: in this case, pointing to only one geometry object.
VkAccelerationStructureBuildGeometryInfoKHR buildInfo{VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_BUILD_GEOMETRY_INFO_KHR};
buildInfo.flags = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_TRACE_BIT_KHR;
buildInfo.geometryCount = 1;
buildInfo.pGeometries = &geometry; // 这里的geometry实际指向instance数据
buildInfo.mode = VK_BUILD_ACCELERATION_STRUCTURE_MODE_BUILD_KHR;
buildInfo.type = VK_ACCELERATION_STRUCTURE_TYPE_TOP_LEVEL_KHR;
buildInfo.srcAccelerationStructure = VK_NULL_HANDLE;

// Query the worst-case AS size and scratch space size based on the number of instances (in this case, 1).
VkAccelerationStructureBuildSizesInfoKHR sizeInfo{VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_BUILD_SIZES_INFO_KHR};
vkGetAccelerationStructureBuildSizesKHR(
	device,
	VK_ACCELERATION_STRUCTURE_BUILD_TYPE_DEVICE_KHR,
	&buildInfo,
	&rangeInfo.primitiveCount,
	&sizeInfo
);

// Allocate a buffer for the acceleration structure.
bufferTLAS = allocator.createBuffer(
	sizeInfo.accelerationStructureSize,
	VK_BUFFER_USAGE_ACCELERATION_STRUCTURE_STORAGE_BIT_KHR
	| VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT
	| VK_BUFFER_USAGE_STORAGE_BUFFER_BIT
);

// Create the acceleration structure object. (Data has not yet been set.)
VkAccelerationStructureCreateInfoKHR createInfo {VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_CREATE_INFO_KHR};
createInfo.type = buildInfo.type;
createInfo.size = sizeInfo.accelerationStructureSize;
createInfo.buffer = bufferTLAS.buffer;
createInfo.offset = 0;
NVVK_CHECK(vkCreateAccelerationStructureKHR(device, &createInfo, nullptr, &tlas));

buildInfo.dstAccelerationStructure = tlas;

// Allocate the scratch buffer holding temporary build data.
nvvk::Buffer bufferScratch = allocator.createBuffer(
	sizeInfo.buildScratchSize,
	VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT
	| VK_BUFFER_USAGE_STORAGE_BUFFER_BIT
);
buildInfo.scratchData.deviceAddress = GetBufferDeviceAddress(device, bufferScratch.buffer);

// Create a one-element array of pointers to range info objects.
VkAccelerationStructureBuildRangeInfoKHR* pRangeInfo = &rangeInfo;

// Build the TLAS.
vkCmdBuildAccelerationStructuresKHR(cmdBuffer, 1, &buildInfo, &pRangeInfo);
```

## 2.3 Acceleration Structure Operations



# 3 Ray Tracing Pipelines





# 4 Shader Binding Table

## 4.1 定义

当光线与几何相交时，SBT指定了将执行哪些shader，以及在这些条件下使用哪些资源，即所有shader与材质数据都能够通过SBT访问。SBT本质上是一组shader records，这些records由shader handles以及定义的数据组成。SBT的shader records上传到一个`VkBuffer`中，并且具有同一个步长，因此步长需要满足最大的shader resource set。

对于给定的一个shader record，其中的shader handles决定了哪些shader会被执行，以及程序定义的数据会作为buffer对象提供给shader访问，通常包括资源索引，buffer device address以及常数。函数调用参数、shader管线以及加速结构共同确定要使用的shader record。

一共有三种shader record，ray generation records、hit group records与miss records：

- 一个ray generation record提供一个ray generation shader handle，作为ray tracer的入口，通常负责生成与追踪光线
- 一个hit group record可能包含三种不同的shader handles (因此叫hit group)，intersection、any-hit、closest-hit分别在不同的阶段调用。
- 一个miss record提供一个当光线没有与任何几何相交时调用的miss shader handle

此外ray tracer可以具有几种不同ray types，例如primary、shadow rays。每个ray type可能具有不同的hit group、miss records。例如，primary ray通常处理closest-hit，而shadow ray只需要确定有any hit发生。

下图是一个SBT的示例，一共包含4个shader records：一个ray generation shader；一个miss shader；两个hit groups，hit groups对应了场景中具有不同着色信息的物体，其由closest-hit shader与可选的intersection、any-hit shaders组成。

<img src="/images/Rendering Blogs/Graphics Basis/Vulkan Ray Tracing.assets/image-20240205104531529.png" alt="image-20240205104531529"  />

## 4.2 查找

如果ray与一个geometry相交，将会被调用的shader record有instance参数、trace ray call参数以及geometry在BLAS中顺序共同确定。有如下参数，

- $I_{offset}$ : TLAS 的每一个 Instance 可以声明其 hit group 在 hit group array 中的起始位置。每个 Instance 可以对应一个 hit group，该 Instance 的 BLAS 中的 Geometry 则在该 hit group 中索引其 hit group record。

- $G_{id}$ : Geometry ID，某一 Geometry 在 BLAS 的 Geometry 数组中的索引。

- $R_{offset}$ : ray type 的索引。当具有多个 ray type 时，需要为每个 ray type 定义 hit group。可以通过 ray type 来索引 ray type 所使用的 hit group。

- $R_{stride}$ : ray type 的数量。shader records 的存储方式是每个 Geometry 的不同 ray type 的 hit group 连续存放，ray type 的数量表示每个 Geometry 有多少个 hit group。

hit group record 的索引计算方式为
$$
HG_{index} = I_{offset}+R_{offset}+G_{id}*R_{stride}
$$
hit group records在buffer中连续存放，`&HG[0]`为hit group records的起始地址，$HG_{srtride}$为两个相邻hit group records的步长，因此hit group的起始地址为
$$
HG = \&HG[0] + HG_{stride} \times HG_{index}
$$
<img src="/images/Rendering Blogs/Graphics Basis/Vulkan Ray Tracing.assets/image-20240205113515529.png" alt="image-20240205113515529" style="zoom:80%;" />

<center>hit group索引的确定过程</center>

Miss Record 的索引可以直接由 traceRay call 的参数指定。