---
typora-copy-images-to: ..\..\..\images\Render Engine\Large World Technique\${filename}.assets
typora-root-url: ..\..\..\
title: Virtual Texture
categories:
- [Render Engine, Large World]
mathjax: true
---

# 1 Summary



# 2 Virtual Texture 技术

## 2.1 Software Virtual Texture [[2]](#[2])

### 2.1.1 地址映射

如何把 virtual page 中的 virtual address 转换为 physical page 中的 physical address？

- virtual address 是描述 virtual texture space 的，比如将材质贴图放到 VT 空间，那么光栅化模型时，插值顶点中的texcoord属性就可以直接得到virtual address
- physical address 是数据实际的存放位置，一般指向位于显存中的一张texture上一个texel

virtual texture space 会划分为page单位，同样会以相同大小page来划分physical texture。地址映射过程就是要已知表面上一点的 virtual address，映射到physical texture上对应texel位置。

映射过程如下图所示，图中的 virtual, physical 分别表示采样点的 virtual address 与 physical address，应该都是 uv 空间下的归一化坐标。由于page都是等大小的，很自然可以想到采样点在 virtual page与physical page 内的局部坐标都是一样的。于是有如下恒等式 ，
$$
\mathrm{physical} = (\mathrm{virtual} - A) \times (C / D) + B
$$

> 对该等式的理解，可以先做移项变换，有
> $$
> (\mathrm{physical} - B) \times D = (\mathrm{virtual} - A) \times C
> $$
>
> - 等式左侧：(physical - B`“physical page offset”`) 为 physical page 内的局部uv坐标 `local_physcial_texel_uv`，乘上 D (physical texture size) ，得到当前采样点在page内的局部整数坐标 `local_texel_coord`
> - 等式右侧：(virtual - A`“virtual page offset”`) 为 virtual page 内的局部uv坐标 `local_virtual_texel_uv`，乘上 C (virtual mip size)，得到当前采样点在page内的局部整数坐标 `local_texel_coord`
>
> 注意：`local_physcial_texel_uv` 与 `local_virtual_texel_uv` 是不相等的，因为 physical texture space 大小与 virtual texture space 大小不相等。

再进行恒等变换，最终得到一个缩放平移变换 作为地址映射函数，而每个page entry只需要记录 scale与bias 即可。
$$
\mathrm{physical} = \mathrm{virtual} \times \mathrm{scale} + \mathrm{bias}
$$

<img src="/images/Render Engine/Large World Technique/Virtual Texture.assets/image-20250408220822366.png" alt="image-20250408220822366" style="zoom:50%;" />



### 2.1.1 page table

需要一个 page table 来负责管理每个 virtual page 信息。每个virtual page 在 page table 都对应一项，称为 page entry，其中记录了是否载入physical page，如果载入，具体的地址映射参数。page table 的管理自然想到 mipmap 这种四叉树结构，mipmap level 对应了 virtual texture space 的level。也就是说 level $i$ 的 virtual texture space 下的 virtual page 对应的 page entry 记录在 mipmap level $i$ 下。使用 virtual page coord 来索引 page table texture。

对于非方形的page，需要存储`vec2 scale`与 `vec2 bias`来完成地址转换。如果将scale、bias存储在address mipmap 中，存储量会随着virtual page的数量增加，开销较高。可以将寻址信息拆分为两个texture：
- mipmap(2-byte): one texel per virtual page，仅存储 virtual page 加载到的 physcial page坐标 (x,y)
- non-mipmap(f32x4): one texel per physical page，存储virtual address到physical address的变换信息，scale、bias

### 2.1.2 Texture Filtering

bilinear filtering: physical page 增加一个 texel 的border

anisotropic filtering:

trilinear filtering: 硬件实现则需要physical page增加 2 texel 的border，为physical texture生成mipmap，然后直接采样即可。软实现需要在shader中执行相邻两个层级的地址转换，再进行计算

### 2.1.3 Feedback Rendering

如何确定加载哪些page？

基于 Feedback Rendering 的策略。这个可以设计为一个单独的pass，或者在Pre-Z或者G-Buffer pass 中额外增加一个 render target，同时渲染生成 virtual page coord, mip level 和VT id (用来应对多虚纹理的情况）。

如下右图可视化的feedback信息，page的变化在屏幕空间相对较低，因此 feedback RT 可以比屏幕分辨率低。

<img src="/images/Render Engine/Large World Technique/Virtual Texture.assets/image-20250408230626300.png" alt="image-20250408230626300" style="zoom:50%;" />

在physical page不够用时，可以在生成feedback中增加 LOD bias

### 2.1.4 Texture Popping

如果一个page的目标LOD偏移了超过一个层级，那么当目标LOD突然可用时，会出现pop现象。直接的做法是，对于变换大的LOD，可以放缓切换过程。例如，LOD只向相邻LOD进行切换，而切换过程的处理可以有：

- 使用具有两次地址变换的tri-linear filtering
- 或者，physical page 以blend形式更新，即coarser 与 finer mip之间的某种blend方式

## 2.2 Adaptive Virtual Texture

以 mipmap 形式管理 page table 有一个缺陷。由于一个address mipmap(indirection texture)的texel 引用相应层级的virtual page。因此 indirect texture mip 0 的分辨率限制了virtual texture的最大分辨率。当大世界地形要求每米分辨率很高时，indirect texture会大到不能应用。

这种缺陷核心原因：由于屏幕分辨率有限，PVT中的mipmap有很大的浪费，其中有很大部分内容不会被用到。

AVT通过动态划分来更紧凑利用indirect texture，达到更高的virtual texture space大小上限。AVT 不再使用mipmap层级结构，每个texel依然是引用指定层级的virtual page，但对indirect texture实时动态划分为不同大小的块（address mipmap是一个个的像素）。page request会基于到相机的距离来申请不同 lod，对应indirect texture中不同大小的块。块越大，像素越多，每个像素引用一个virtual page，那么使用的virtual page越多，texel数据精度越高

### 2.2.1 Allocate Virtual Texture Atlas

下图是 virtual image 的申请过程

<img src="/images/Render Engine/Large World Technique/Virtual Texture.assets/image-20250408232141488.png" alt="image-20250408232141488" style="zoom: 45%;" /><img src="/images/Render Engine/Large World Technique/Virtual Texture.assets/image-20250408232207739.png" alt="image-20250408232207739" style="zoom:45%;" />

- (a): 为 sector A 插入一个 16x16 的 virtual image
- (b): 为 sector B 插入一个 64x64 的 virtual image
- (c): 为 sector C 插入一个 32x32 的 virtual image

### 2.2.2 实时调整 virtual image 大小

有如下过程

<img src="/images/Render Engine/Large World Technique/Virtual Texture.assets/image-20250408232703318.png" alt="image-20250408232703318" style="zoom:50%;" />

# 3 RVT in UE4 [[3]](#[3])

<img src="/images/Render Engine/Large World Technique/Virtual Texture.assets/virtual texture.png" alt="virtual texture"  />

术语

- Virtual Texture：虚拟纹理，以下简称 VT
- Runtime Virtual Texture：UE4 运行时虚拟纹理系统，以下简称 RVT
- VT feedback：存储当前屏幕像素对应的 VT Page 信息，用于加载 VT 数据
- VT Physical Texture：虚拟纹理对应的物理纹理资源
- PageTable：虚拟纹理页表，用来寻址 VT Physical Texture Page 数据。一个page table对应一个virtual texture space
- PageTable Texture：包含虚拟纹理页表数据的纹理资源，通过此纹理资源可查询 Physical Texture Page 信息。有些 VT 系统也叫 Indirection Texture，由于本文分析 UE4 VT 的内容，这里采用 UE4 术语
- PageTable Buffer：包含虚拟纹理页表数据内容的 GPU Buffer 资源

## 3.1 VT 总体流程

从原理上来说，VT 系统主要由 2 大阶段构成，VT 数据准备和 VT 运行时采样阶段。

1. VT 数据准备阶段：
   1. 生成 VT feedback 数据
   2. 生成 VT 纹理数据，并更新到指定 VT Physical Texture 对应的 Page
   3. 根据 feedback 数据生成并更新 PageTable 数据
2. VT 运行时采样阶段：
   1. 根据屏幕空间坐标以及相关信息生成 VT Physical Texture UV
   2. 对 VT Physical Texture 执行采样

UE4 的 RVT 基本上也是按照这个原理和流程来实现的，本文就按照这个顺序来详细讲解。在讲解之前，为了便于后续理解，先来了解下 UE4 RVT 的实现机制。

## 3.2 UE4 RVT

`FRuntimeVirtualTextureProducer` 实现了 `IVirtualTexture` 中的接口：

- RequestPageData：请求页面数据
- ProducePageData：产生页面数据

实际着色计算中，会有多种纹理数据，如 diffuse、normal、roughness 等，UE4 RVT 使用layer表示不同的数据。每个layer表示不同的physical texture，这些 Physical Texture 的寻址信息保存在同一个 VT 上的 PageTable Texture 的不同颜色通道中。

UE4 RVT 中所使用的 GPU 资源有以下 3 种：

- PageTable Buffer 用于在 CPU 端只写的 PageTable 数据。
- PageTable Texture 用于在采样 VT 时从 VT 到 PT 的地址变换，此资源数据是基于 PageTable Buffer 内容，通过 RHICommandList 在 GPU 上填充。
- VT Physical Texture 实际存储纹理数据的资源，通过 VT feedback 从磁盘或运行时动态生成纹理数据，并在 VT 数据准备阶段中更新到 Physical Texture 中。

### 3.2.1 VT 数据准备阶段

#### 3.2.1.1 生成 VT feedback 数据

与一般在 GBuffer pass 中生成 VT feedback Buffer 不同的是，UE4 中并没有单独的 VT feedback Pass，而是在材质 Shader 的最后调用 `FinalizeVirtualTextureFeedback` 将当前帧的 Request Page 信息 写入 feedback UAV Buffer，然后在 CPU 侧每帧的 `FVirtualTextureSystem::Update` 中读取这个 Buffer，根据 Buffer 中的 Request Page 信息读取 Page Texture 数据。

feedback 每项数据32位，内容如下：

<img src="/images/Render Engine/Large World Technique/Virtual Texture.assets/feedback.jpg" alt="feedback" style="zoom:50%;" />

1. TableID：FeedBack所在的像素块的纹理对应的PageTable的ID。因为我们在材质中采样这个贴图的时候，我们是知道我们这个 pageTable的ID的。（16个PageTable）
2. vPageX：所在PageTable中的X坐标，根据UV，Tile数量和PageTable 偏移求出来。
3. vPageY：所在PageTable中的Y坐标，根据UV，Tile数量和PageTable偏移 求出来。
4. vLevel：这里并不是简单的Mipmap，因此UE换了个名称叫做vLevel。做法是当前的Mipmap+一个随机值，这样的话在静止画面下，这个值实际上是一个区间，每帧我们的vLevel就是不同的，这样，我们就能根据TAA来做一些混合操作。



# References

<a name="[1]">[1]</a> [浅谈Virtual Texture](https://zhuanlan.zhihu.com/p/138484024)

<a name="[2]">[2]</a> Juraj Obert, J. M. P. van Waveren, and Graham Sellers. 2012. Virtual texturing in software and hardware. In *ACM SIGGRAPH 2012 Courses* (*SIGGRAPH ’12*), 2012. Association for Computing Machinery, New York, NY, USA.

<a name="[3]">[3]</a> [游戏引擎随笔 0x14：UE4 Runtime Virtual Texture 实现机制及源码解析](https://zhuanlan.zhihu.com/p/143709152)

<a name="[4]">[4]</a> [UE4 Virtual Texture](https://zhuanlan.zhihu.com/p/373969159)

