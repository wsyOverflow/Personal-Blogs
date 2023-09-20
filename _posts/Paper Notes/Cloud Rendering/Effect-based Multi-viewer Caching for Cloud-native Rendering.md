---
typora-copy-images-to: ..\..\..\images\Paper Notes\Cloud Rendering\${filename}.assets
typora-root-url: ..\..\..\
title: Effect-based Multi-viewer Caching for Cloud-native Rendering
keywords: render pipeline, cloud rendering
categories:
- [Paper Notes, Cloud Rendering]
mathjax: true
---

# 1 Summary

本文基于object-space/texture-space shading管线，提出一种面向multi-viewers的云渲染的cache系统设计。对于effect中与视角无关的部分，通过转换为texture-space下的渲染任务，并且设计effect-based cache系统，使得不同viewers共享同一份effect中间数据。提出的effect-based cache系统共包含定义在物体表面的On-Surface Cache(OSC)与定义在grid上的World-Space Cache(WSC)两种策略，统一使用hash table的形式进行存储管理，但二者具有不同的hash key设计，包含实际数据的cache entry由具体的effect定义。作者为两种cache策略分别给出相应的具体effect应用，同时提出通过为不同的effect使用不同的bias配置来为低频effect选择低分辨率，减少内存开销。

# 2 Background



# 3 Pipeline设计

管线设计如图 [Fig-1](#Fig-1)，首先为每个viewer执行visibility pass来决定那些cache entries会被使用到；然后对所有viewers的cache request进行整合更新，这一步可以执行在不同的GPU上；之后viewers再通过采样cache来着色帧；最后是后处理，将生成的图像插入到output stream。



<a name="Fig-1"></a>
![image-20230703130901243](/images/Paper Notes/Cloud Rendering/Effect-based Multi-viewer Caching for Cloud-native Rendering.assets/image-20230703130901243.png)

<center>Fig-1 Cloud Native Rendering Pipeline</center>

想要被多个viewer复用，被cache的计算只能是视角无关的，同时作者提出基于effect的cache设计，可以为不同类型的渲染计算进行调整。经过对现代游戏引擎中的效果分析，作者提出两种类型的cache：on-surface cache，共享表面点上的计算；world-space cache，共享世界空间数据。

## 3.1 On-Surface Caches

On-Surface Cache(OSC)采用类似于[[1]](#[1])的管线。首先预处理阶段需要为mesh生成唯一表示的纹理空间。作者通过将一定数量的相连接的面片划分成island单位，每个island有独立的texture space，可以生成triangle lookup texture，即每个texel的triangle id。

对于cache texture的实现，尽管目前硬件支持virtual texture，但64KB的page大小对于本系统过大，因此作者采用软件实现的cache系统。本文采用的cache entity大小为8x8 texel block，texel的实际大小由具体effect定义。cache的访问采用hash形式，为此作者提出设计用于访问所有virtual textures的单个hash map。hash map中存储的是指向cache entity(实际数据)的hash entity，由如下部分组成：

- island index：32bit，用来标识island，island下是独立的texture space
- resolution：4bit，应该是mipmap level
- texel block coordinates：2x12bit，island下的8x8 texel block的纹理坐标。
  因此可以最多表示 32kx32k的纹理空间（$2^{12}*8=2^{15}=32k$）

OSC pipeline主要由如图[Fig-2](#[Fig-2])中的五个阶段组成：

1. Visibility：决定哪些cache entity会被用到以及去重cache request
2. Cache Update Queuing：填充着色任务队列
3. Triangle Lookup Rendering：
4. Effect Update：cache更新
5. Compositing：整合cache数据以及剩余的渲染计算生成最终渲染结果

<a name="Fig-2"></a>

![image-20230704143603816](/images/Paper Notes/Cloud Rendering/Effect-based Multi-viewer Caching for Cloud-native Rendering.assets/image-20230704143603816.png)

<center>Fig-2 OSC Pipeline</center>

### 3.1.1 Visibility

为每个viewer执行visibility pass生成所请求的cache entity，此外可以根据屏幕空间梯度以及每个effect的先验知识来调整cache entity的bias，提高目标mip level。例如空间低频的effect可以渲染的目标分辨率更低。

为每个texel block配置一个64bit mask，来标识block内哪些像素需要更新。对于每个surface point可能需要1、4、8个texel blocks，分别用于point、bilinear、trilinear采样。对于每个texel block都会查询hash table来判断是否已经map，如果没有map，则map一个空block。同时，每帧重置texel mask。通过这种方式，可以去除重复请求。

> 不是很清楚mip level如何工作的？似乎是降低的是texture space的分辨率，例如level 0的分辨率32kx32k，level 1的分辨率则为16kx16k。
>
> unbiased情况是通过屏幕空间梯度计算的mip level达到pixel与texel一一对应；biased情况是手动为某些低频effect增大mip level，多个pixel对应一个texel。

### 3.1.2 Cache Update Queuing

本阶段的更新以texel block为单位进行，为每个效果创建一个cache update queue，用于存储texel block的更新请求。

## 3.2 World-Space Caches



# Reference

<a name="[1]">[1]</a> K. E. Hillesland and J. C. Yang. 2016. Texel Shading. In Proceedings of the 37th Annual Conference of the European Association for Computer Graphics: Short Papers (Lisbon, Portugal) (EG ’16). Eurographics Association, Goslar, DEU, 73–76.  
