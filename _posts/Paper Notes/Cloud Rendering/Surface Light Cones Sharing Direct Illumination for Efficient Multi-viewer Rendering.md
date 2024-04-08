---
typora-copy-images-to: ..\..\..\images\Paper Notes\Cloud Rendering\${filename}.assets
typora-root-url: ..\..\..\
title: Surface Light Cones: Sharing Direct Illumination for Efficient Multi-viewer Rendering
keywords: direct light, cloud rendering
categories:
- [Paper Notes, Cloud Rendering]
mathjax: true
---

# 1 Summary

直接光照与视角相关，无法直接缓存，但渲染方程中表面着色点的入射radiance与视角无关。基于此，这篇论文提出了一种基于cone的缓存表面入射radiance的方法——surface light cone。依靠这些cones来表示主要可见光源的投影，允许跨帧甚至同一场景内的多个相机之间复用入射radiance。

作者提出的 surface light cone 方法主要分为两部分：

- direct light 信息收集以及 cone 构建：使用光追对场景中光源进行随机采样，基于采样点到 surface point 的样本，不断更新其cone的radiance信息以及微调几何机构，以满足静态、动态场景下的DI收敛。
- shading阶段采样 cone cache，使用 gbuffer 进行着色。着色过程近似为对每个 cone 的迭代采样累积。



# 2 Cone-based Radiance Caching

一个cone使用一个方向 $D$ 与开口角度 $\alpha$，如下图所示。对于一个 surface point，使用多个 cones (通常为3个)来描述所有入射radiance信息。除了cone之外，作者还存储cone数量与累积数量用于radiance降噪。增大 cone 的最大数量可以提升对空间更精细地划分，从而提高这种抽象表示的精度。

<img src="/images/Paper Notes/Cloud Rendering/Surface Light Cones Sharing Direct Illumination for Efficient Multi-viewer Rendering.assets/image-20240408133612290.png" alt="image-20240408133612290" style="zoom:67%;" />

Surface Light Cone 方法实现在 [[2]](#[2]) 提出的 effect-cache 系统中，位于 cache update stage，如下图所示。一个统一的更新为每个surface point执行，以构建cones。在最终着色阶段，从cones中采样并进行着色，与其它effect组合得到最终着色结果。

![image-20240408144341361](/images/Paper Notes/Cloud Rendering/Surface Light Cones Sharing Direct Illumination for Efficient Multi-viewer Rendering.assets/image-20240408144341361.png)

## 2.1 DI Gathering and Cone Construction

DI 信息收集与 cone 表示构建过程是本文所提方法最重要的一部分，该过程位于 cache update stage。cache update是以一定周期为每个可见的surface point执行的。当一个surface point变为可见时，它的每个周期开始于对已存储的cone 进行unpack，除非它的 cone 表示为空（新出现的surface point）。之后确定是否在更高细节级别 (LOD) 上存在该 surface point 的cache entry，如果存在则重用这些信息。

该过程被划分为两步：先将radiance信息加入到cone表示中；对于光源的移动，仅仅加入radiance到cache中是不够的，还需要处理移动过程中膨胀的cone，第二步则收缩cone。

### 2.1.1 Incoming Radiance Updates

为了加入新的 radiance 到 cone 中，先随机采样场景光源，包括发光表面、解析面积光源以及environment map。采样数量与质量、效率要求有关，论文实验中选择 8 个样本，可以达到高帧率，同时由于时序累积，DI信息也可以保持准确。

对于每个样本，先通过光追确定surface point到采样点是否有直接可见（无遮挡）路径。之后只会将对surface point可见的采样点的radiance信息加入到cone结构中。实际中，surface point 可以长时间（相对于帧数而言）保持可见以及光源也会保持相对稳定，因此在没有动态改变被检测到时，光追样本数量可以逐渐减少，用以避免不必要的计算。

当得到一个新的入射radiance样本，将其加入到cone cache时，会面临四种不同的情况，对应四种不同的处理，如下图所示，

- A：新样本位于已存在cone之内，直接将样本radiance累积加入即可。否则，
- B：如果已有cone数量未达到最大限制，则为新样本创建一个新的cone。否则，
- C：如果新样本与距其最近的cone的夹角要小于cone之间的最小夹角，那么扩展该最近cone以将新样本包含其中。否则，
- D：合并两个最近的cone，并为新样本创建一个新的cone。

<img src="/images/Paper Notes/Cloud Rendering/Surface Light Cones Sharing Direct Illumination for Efficient Multi-viewer Rendering.assets/image-20240408150858211.png" alt="image-20240408150858211" style="zoom:80%;" />

可以看出，样本加入cone的过程包含了cone几何结构的更新以及radiance的累积。基于累积次数 $C$，累积更新过程如下
$$
L_{i,new} = L_{i,old} \cdot (1-1/C) + L_{i,sampled} * (1 / C)
$$
在每个周期根据确切的样本来增加 $C$，仅在静态场景中有效。为了捕获动态改变，可以通过重置 $C$，但会导致信息丢失。作者发现在实践中，cone structure中的改变预示着场景发生了改变，因此一个cone改变了方向意味着影响其surface point的光照环境发生了变换。这时可以通过降低 $C$ 以加快新信息的吸收。作者使用 $\beta$ 来表示cone前一个方向与当前方向的夹角，当 $\beta$ 与 $C$ 超过一个小的阈值时，执行自适应更新
$$
C = C / \min(\max(1 + \beta \cdot 0.05, 1.01), 3.0)
$$

### 2.1.2 Dynamic Scene Updates

在样本加入cone的过程中，cone总是执行扩展操作。对于动态改变，必须有方式能够自适应更新cone结构。以一个光源移动为例，如下图所示。在光源移动过程中，为了包含新的样本，cone可能会扩展的越来越大。因此，还需一个检测cone的哪些部分不再需要的方法。

<img src="/images/Paper Notes/Cloud Rendering/Surface Light Cones Sharing Direct Illumination for Efficient Multi-viewer Rendering.assets/image-20240408164550421.png" alt="image-20240408164550421" style="zoom: 80%;" />

作者通过在cone中采样一个偏离 opening angle 一定角度 $\varepsilon$ 的方向样本来检测是否有光源存在，基于opening angle的百分比来选择 $\varepsilon$ 。如果该样本击中一个光源，该样本被丢弃，cone在该方向上是准确的。如果没有光源被击中，则收缩cone。收缩过程如下图所示，首先确定有多少cone应该被收缩，
$$
\delta = \alpha - \arccos(dot(D,s))
$$
其中 $\alpha$ 是cone开口角度，$D$ 是cone方向，$s$ 是样本方向。最终 $\alpha$ 收缩了 $\delta$ ，而 cone 向远离 $s$ 的方向旋转了 $\delta/2$ 。

<img src="/images/Paper Notes/Cloud Rendering/Surface Light Cones Sharing Direct Illumination for Efficient Multi-viewer Rendering.assets/image-20240408165535845.png" alt="image-20240408165535845"  />

## 2.2 Shading

对于每个viewer，基于gbuffer中的材质信息与cache数据对像素进行着色。求解渲染方程则通过为每个cone采样 $nS$ 个样本，如下式
$$
L_o(x,\omega_o) = L_e(x, \omega_o) + \sum_c^{nC}\sum_s^{nS}f_r(x,\omega_{i,c,s},\omega_o)\frac{L_{i,DI,SLC}(c)}{nS}(\omega_{i,c,s}\cdot n)
$$
其中，$nC$ 与 $nS$ 分别是 cone 的数量与样本的数量，$L_{i,DL,SLC}$ 是存储在surface light cone内的 radiance，而 $\omega_{i,c,s}$ 是采样方向。

作者使用 16 个样本达到高质量着色，注意这里的样本不需要求解visibility，因为在cone cache更新过程中考虑了 visibility，即只会加入可见的光源采样点的radiance信息。

> 个人感觉 cone 还是一种 prefilter 思想。cone 中先累积了入射radiance，相当于把渲染方程被积函数 $L_i * f_r$ 的 $L_i$ 单独拿出来积分。

## 2.3 Cache Compression

cone cache 的数据是以紧凑的方式存储。每个cone包括以下部分，共 91 bits：

- 位于表面局部空间的方向（即TBN坐标系下），并转换为球坐标，使用 16-bit floats。球坐标应该是两个分量，共 32 bits
- 开口角度，使用 6-bit 定点数
- radiance RGB，每个channel使用 18 bits。共 54 bits

一个 surface point 会有多个 cone（文中为3个），累积计数 $C$ 使用 8 bits，cone数量使用 2 bits。因此一个 surface point 共需 $91\times 3+8+2=283$ bits。作者使用 RGB32 的贴图格式，使用 3 个texel，有 2 bits 多余。

此外，在论文实验中，作者使用 cache bias 1.0，3 个 cones ，以及最初 8 个primary样本用于cache更新，之后便降为 1 个样本，除非检测到动态改变。







# Reference

<a name="[1]">[1]</a> Pascal Stadlbauer, Alexander Weinrauch, Wolfgang Tatzgern, and Markus Steinberger. 2023. Surface Light Cones: Sharing Direct Illumination for Efficient Multi-viewer Rendering. In *High-Performance Graphics - Symposium Papers*, 2023. The Eurographics Association.

<a name="[2]">[2]</a> Alexander Weinrauch. 2023. Effect-based Multi-viewer Caching for Cloud-native Rendering. *siggraph* (2023).

