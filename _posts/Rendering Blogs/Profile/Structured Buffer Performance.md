# 1 对齐问题 [[1]](#[1])

structured buffer 为在 GPU 上表示除了颜色或4分量之外的数据结构提供了一个非常便利的解决方案，非常适用于如 tile-based deferred shading 等技术。

structured buffer 的内存是紧密排列的（与C++的默认对齐似乎是一样的），例如以下代码会生成步幅20字节的缓冲区

```c++
struct Foo
{
    float4 Position;
    float Radius;
};
StructuredBuffer<Foo> FooBuf;
```

这存在一个问题：由于结构体的大小没有对齐到 128-bit，`Position`可能会跨cache line，这会导致数据读取更加昂贵。虽然一次非高效的读取可能不会对性能造成太大影响，但随着代码的复杂性增加，它可能迅速放大。例如，着色器对复杂光源列表进行迭代，每个光源包含超过 100 字节的数据，这可能成为一个严重的性能瓶颈。事实上，我们最近发现了一段预发布代码，仅因为这种差异，就导致了全帧性能下降超过 5%。

避免这些问题的简单做法：

- 设计大小为 128-bit 整数倍的结构体
- 注意内部对齐，是向量类型能够自然对齐

调整后如下所示，尽管这样会浪费一些内存，但相比于性能损失，代价更小。

```c++
struct Foo
{
    float4 Position;
    float  Radius;
    float pad0;
    float pad1;
    float pad2;
};
StructuredBuffer<Foo> FooBuf;
```

# 2 冗余使用与延迟 [[2]](#[2])

structured buffer 的处理有冗余会给性能带来影响。如下代码中，这种写法导致每个线程都在重复获取相同的数据。对于包含大量光源的列表，这种冗余工作可能会非常多。

```c++
StructuredBuffer<Light> LightBuf;

for (int i = 0; i < num_lights; i++) {
    Light cur_light = LightBuf[i];

    // Do Lighting
}
```

**结构化缓冲区的实现机制**是针对分散访问设计的，具备较好的性能。但这种设计的一个特点是：structured buffer 每次获取数据都可能有相当的延迟。当所有线程完全一致地访问时，缓存命中率会非常高，但延迟问题仍然存在。在这种情况下，将数据批量存储到共享内存中可能是一个更好的选择。

```c++
StructuredBuffer<Light> LightBuf;
groupshared Light[MAX_LIGHTS] LocalLights;

LocalLights[ThreadIndex] = LightBuf[ThreadIndex];
GroupSharedMemoryBarrierWithGroupSync();
for (int i = 0; i < num_lights; i++) {
    Light cur_light = LocalLights[i];

    // Do Lighting
}
```

显然，这种优化增加了代码复杂性，同时可能因为共享内存压力或额外的barrier指令而无法提升性能。此外，结构体的大小也会影响这一方法的效率。例如，对于大小为 1024 字节的结构体，由于线程之间的步幅较大，效率可能会受到影响。在某些情况下，使用更简单的结构体设计，比如将数据展平为 `float` 或 `float4` 数组，并手动计算索引偏移，可能会更高效。尽管代码会显得复杂一些，但这是一个内部循环，**消除冗余**可能值得为此付出代价。

# 3 Constant Buffers [[3]](#[3])

本章节将讨论如何使用常量缓冲区（Constant Buffers）来避免结构化缓冲区（Structured Buffers）可能遇到的陷阱。constant buffer 即GL中的uniform buffer。

作为开发者，应该考虑为何使用 structured buffer。如果数据大小可以满足 constant buffer 限制的 64 KB 空间，那么 constant buffer 可能是一个更好的选择。这在代码使用一致访问模式时尤为适用（如上一章节描述的那样，连续访存）。**constant buffer 的一致性访问的效率很高，延迟比 structured buffer 低。而 structured buffer 的分散访问性能更好**，如果代码的访问模式预计会有大量不一致性，那么 structured buffer 才是真正的优选。

来自一个实际的示例：使用结构化缓冲区、128-bit 步幅、一致的访问模式，并且数据量小于 64 KB。将该结构化缓冲区转换为常量缓冲区后，单个着色器的执行时间减少了 25-33%，整体帧率提升了 10%。这仅通过更改一个资源的类型就能获得如此显著的性能提升。以下是代码示例及所需的更改：

```c++
struct Light
{
    float3 Position;
    float Radius;
    float4 Color;
    float3 AttenuationParams;
    uint Type;
};

// Original code
StructuredBuffer<Light> LightBuf;

for (int i = 0; i < num_lights; i++)
{
    Light cur_light = LightBuf[i];

    // Do Lighting
}

// Revised code
cbuffer LightCBuf
{
    Light LightBuf[MAX_LIGHTS];
};
 
for (int i = 0; i < num_lights; i++)
{
    Light cur_light = LightBuf[i];

    // Do Lighting
}
```

## 3.1 实例：Tiled Deferred Lighting

**在实际应用中，结构化缓冲区的一个常见用途是支持分块延迟光照（Tiled Deferred Lighting）。**
这种情况下，使用常量缓冲区会遇到两个限制：

1. 数据量可能超出常量缓冲区的容量。
2. 虽然着色阶段通常是一致访问，但 culling 阶段通常完全是分散的访问。（应该是指为tile生成光源列表）

幸运的是，这两个问题都可以通过简单的解决方案解决。

### 3.1.1 拆分结构体

如果需要 256 字节的结构体来表示光源，并支持超过 256 个光源，那么数据大小超过了 constant buffer 的限制。以下代码展示了两个解决方案：

1. 将结构体拆分为多个小结构体。
2. 保留 structured buffer，同时创建一个并行的 constant buffer，只包含最常用的数据。

```c++
struct Light // 144 bytes, only 455 lights possible w/ CB
{
    float3 Position;
    float Radius;
    float4 Color;
    float3 AttenuationParams;
    uint Type;
    float4 SpotDirectionAndAngle;
    float4 ShadowRect;
    float4x4 ShadowMatrix;
};

// Original structured buffer version
StructuredBuffer<Light> LightBuf;
```



```c++
/*
 * Two constant buffers, with lesser-used shadow data in second
 */
struct LightBase // 64 bytes, 1024 lights possible
{
    float3 Position;
    float Radius;
    float4 Color;
    float3 AttenuationParams;
    uint Type;
    float4 SpotDirectionAndAngle;
};

struct LightShadow // 80 bytes, 819 lights possible
{
    float4 ShadowRect;
    float4x4 ShadowMatrix;
};

// MAX_LIGHTS restricted to min of two structures
#define MAX_LIGHTS 819

cbuffer LightCBuf
{
    LightBase LightBufBase[MAX_LIGHTS];
};

cbuffer LightCBufShadow
{
    LightShadow LightBufShadow[MAX_LIGHTS];
};


/*
 * One constant buffer for core parameters with structured buffer
 * for infrequently used parameters and divergent access
 */

// MAX_LIGHTS restricted by only the one structure
#define MAX_LIGHTS 1024

cbuffer LightCBuf
{
    LightBase LightBufBase[MAX_LIGHTS];
};

StructuredBuffer<Light> LightBuf;
```

### 3.1.2 处理分散访问

处理分散访问则更加简单。如上代码注释所述，只需在 culling 阶段绑定数据的结构化缓冲区副本即可解决问题。这需要为数据创建冗余副本，但额外的 64-256 KB 数据代价很小，换来的性能提升可能高达帧率的 10%。

# Reference

<a name="[1]">[1]</a> https://developer.nvidia.com/content/understanding-structured-buffer-performance

<a name="[2]">[2]</a> https://developer.nvidia.com/content/redundancy-and-latency-structured-buffer-use

<a name="[3]">[3]</a> https://developer.nvidia.com/content/how-about-constant-buffers