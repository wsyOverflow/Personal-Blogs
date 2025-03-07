---
typora-copy-images-to: ..\..\..\images\Paper Notes\GI\${filename}.assets
typora-root-url: ..\..\..\
title: Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields
categories:
- [Paper Notes, GI, Light Probe]
mathjax: true
---

## 1 Summary

本文提出了一种基于light probe的实时动态diffuse GI(DDGI) 算法。该算法的probe在世界空间下摆放，其数据完全实时生成，不需要预计算，因此支持具有动态光源与动态几何的场景。

DDGI算法主要包含了probe数据更新与probe-based shading：

1. probe数据更新：为场景中的每个probe生成 $n$ 条probe-update光线，光线与表面相交记为surfel，最终每个probe可以得到存储求交信息的surfel buffer。使用probe-base shading方法进行surfel shading，包含了sufel上的直接光照与从surfel周围probes中得到的间接光照。将surfel shading以给定滞后系数blend到probe中。

2. probe-base shading：作者为surfel shading与最终着色提出了统一的probe-base shading，一共包含了两部分：直接光照pass生成的直接光照信息，注意作者为了简化面光源的模拟过程，使用一次反射的间接光作为面光源的直接光照，而其在直接光照过程中只会生成自发光信息；从probe数据中采样得到的间接光照信息。

本文提出的probe-based shading可以使用上一帧的光照信息与probe数据，实现了将多次反射的开销分摊到多帧。但这也会在visibility信息快速变化时，带来time-lag的异常现象。

## 2 DDGI Probe Overview

和 [[1]](#[1]) 类似，probe中同样存储球面分布的数据，包括几何信息和辐射度量数据，存储在两张oc-map中：

- 距最近几何的距离及其平方，存储在16x16 oc-map中，数据格式为`RG16F`
- diffuse irradiance存储在8x8 oc-map中，数据格式为`R11G11B10F`

不同的是，在 [[1]](#[1]) 中使用texture array打包所有probe的oc-map；而本文采用将所有probe的每一张oc-map打包进一张贴图集，并使用一个像素的边界隔开，同时增加额外的padding来确保4x4对齐。其次本文的probe数据是实时计算得到的，因此能够实现真正动态全局光照。在每一帧，除了插值probe深度信息以适应场景几何的变化外，还能够高效地将更新的光线追踪照明混合到probe地图集。

## 3 DDGI probe更新

每一帧都按照如下几步更新probe数据：

1. 为场景中的 $m$ 个active probe都生成并发出 $n$ 条primary rays，并将光线追踪得到的 $n\times m$ 求交信息存储到surfel buffer中
2. 使用统一的着色方法对surfel buffer计算直接光照与间接光照
3. 使用每个相交surfel得到的shading、距离及其平方结果以blending方式更新到 $m$ 个active probe的oc-map texel

与probe相交的surfel着色计算依赖于前一帧的光照与probe数据，这样做有两个目的：这种方式可以将多次反射的计算分摊到多帧；以blending方式更新probe数据可以使得几何上与辐射度量上在随着时间的尖锐过渡变得平滑。相反，这种方式会在visibility剧烈变化的情况下，由于间接光照更新的延迟，表现出明显的变化。

### 3.1 放置probe

probe在场景中的放置方式与 [[1]](#[1]) 相同，在轴向均匀的三维grid的每个顶点上放置一个probe，对于每轴需要不同空间离散化的场景，我们可以设置scale参数来独立地缩放网格在一个轴上的间距。空间中每一点都与一个包含该点的grid相关联，即对应了一个带有8个probe的cage，如下图所示。而着色过程则需要在cage的probe中进行查询。为了能够充分采样局部光照的变化，需要尽量满足每个点都能有cage的8个probe，避免某些probe位于墙内而无效。为此，作者建议使用grid分辨率以及缩放参数来使得每个类似房间的空间中产生至少一个完整的cage。文中给出，在人类尺度的场景中，1~2米的grid间隔效果最好。

<img src="/images/Paper Notes/GI/Dynamic%20Diffuse%20Global%20Illumination%20with%20Ray-Traced%20Irradiance%20Fields.assets/image-20230224010007700.png" alt="image-20230224010007700" style="zoom: 67%;" />

<center>cage的8个probe [1]</center>

### 3.2 生成与追踪Probe-Update光线 

为了捕获场景中几何与光照的变化，作者选择 $m$ 个active probes，对每个active probe均匀采样 $n$ 个球面方向，采样方式采用stochastically-rotated Fibonacci spiral pattern与 [[1]](#[1]) 类似。一个probe中心与这 $n$ 个方向可以生成 $n$ 条光线，作者将 $m$ 个probe的光线以thread-coherent方式排布，在一个batch内发出。经测试，这种组织方式要优于直接在Vulkan或DirectX的实时光追管线中的primary shader发出光线。

> 球面方向采样方式stochastically-rotated Fibonacci spiral pattern [[3]](#[3])以及光线组织方式需要看实现代码

此外，作者还测试了active probes选取场景中部分probe(相机周围给定半径的范围内)以及光线采样数量随相机距离增大而减少，但实验结果表明，虽然这些优化会带来性能收益，但优化参数依赖于场景，不同场景需要配置不同的参数，否则会带来artifact。在本文的实验中，作者统一采用了每帧更新probe、每个probe都为active以及为所有probe采样相同数量的光线。

### 3.3 Secondary Surfel Shading

对于probe更新和最终渲染中的间接光，作者采用统一的着色模型。具体来说，计算全局光照的两个阶段：更新 $m\times n$ Probe-Update光线采样的surfels着色；最终输出图像中相机下的像素着色。这两个阶段使用相同的着色例程，结合直接光照pass和使用probe数据的间接光照pass。也就是说，surfel的shading包含其直接光照与间接光照：

- surfel的直接光照：从直接光照pass生成直接光照数据中获取，当然surfel不一定可见
- surfel的间接光照：从surfel附近的probes中进行采样插值得到间接光照信息，即probe-based shading

而probe更新与最终渲染之间的差异在于shading queries，着色例程需要着色位置、法线以及观察方向作为输入。对于基于probe光追的surfel shading更新，需要光线求交得到的surfel位置与法线，以及probe中心到surfel的观察方向。

### 3.4 Probe Surfel Upates

在surfel shading之后，可以得到 $m\times n$ surfel points中的每一个surfel的更新着色值、probe中心到surfel的距离及其平方，这些值需要更新到probe数据中。作者给出了alpha-blending的更新方式，设置 $\alpha$ 为滞后参数，用来控制更新至覆盖前一帧值得速度，那么更新速度为 $1-\alpha$，文中采用 $\alpha\in[0.85,0.98]$，有以下更新过程，
$$
newIrradiance[texelDir]=lerp(oldIrradiance[texelDir], \\ 
\sum_{ProbeRays}(\max(0,texelDir\cdot rayDir)*rayRadiance),\alpha)
$$
上式中可以看出，filtered irradiance是采用moment-based filtered shadow query:confused: 直接计算得到的，避免了prefilter高分辨的入射radiance map。这种平滑的入射irradiance被用于计算diffuse间接光照，同时，也可以选择维护一个更高频的irradiance map用于glossy和specular的间接光。

- [ ]  技术细节：


## 4 Shading with DDGI Probes

### 4.1 直接光照

点光源与方向光的直接光照采用variance shadow map(VSM)的deferred render [[2]](#[2]) 来计算。

对于面积光源的直接光照，作者使用一次反射的间接光照管线计算。间接光照管线中，一次反射的间接光是计算场景中直接光的一次反射，多次反射的间接光则是基于前一次反射的间接光计算得到。而对于面积光源的一次反射间接光是基于区域的自发光，而不是面积光源下的直接光照，因此计算得到的是面积光源的直接光照。[图4.1](#Fig 4.1) 的最后一行给出了面积光源特殊处理的直观感受，直接光照结果只有面积光源区域的自发光，而没有照明场景中其他物体。

面积光源的特殊处理避免了对面积光源的直接光照的传统近似方式，例如在面积光源上采样多个点光源计算来近似面积光源的直接光照。除此之外，对面积光源的计算统一为本文提出的probe-base shading技术，可以得到平滑的面积光阴影与反射。

<a name="Fig 4.1"></a>

<img src="/images/Paper Notes/GI/Dynamic%20Diffuse%20Global%20Illumination%20with%20Ray-Traced%20Irradiance%20Fields.assets/image-20230226115136810.png" alt="image-20230226115136810" style="zoom:80%;" />
<img src="/images/Paper Notes/GI/Dynamic%20Diffuse%20Global%20Illumination%20with%20Ray-Traced%20Irradiance%20Fields.assets/image-20230226115205828.png" alt="image-20230226115205828" style="zoom:80%;" />

<center>图 4.1 第一列只有直接光照，第二列有完整的全局光照</center>

### 4.2 Diffuse间接光

为了弥补probe位置的入射irradiance没有考虑着色点周围的局部遮挡，作者使用SSAO得到的遮挡系数调制漫反射的出射radiance。但在更新probe过程中不会包含局部遮挡，间接光忽略该项的影响要远远小于直接可见表面的光照。

本文对间接漫反射的插值和采样技术结合了光线追踪与shadow map的想法，提高对动态几何和光源的鲁棒性，与 [[1]](#[1]) 中不同。具体来说，在得到包含着色点的cage上的8个probe的索引后，为每个irradiance probe根据其相对于着色点的位置与方向计算插值权重。如[图4.2](#Fig 4.2)中surfel X的着色过程的权重包括：使用表面法线 n 采样eight-probe cage的每个probe；使用X到P的方向 dir，为每个probe P施加back-face权重；P中存储的(进行过filter)平均距离 r；为了避免visibility边界采样问题，作者会将X朝着法线方向施加一个偏移量。

<a name="Fig 4.2"></a>

<img src="/images/Paper Notes/GI/Dynamic%20Diffuse%20Global%20Illumination%20with%20Ray-Traced%20Irradiance%20Fields.assets/image-20230226143129489.png" alt="image-20230226143129489" style="zoom:50%;" />

<center>图4.2 surfel X的着色。</center>

[图4.3](#Fig 4.3)说明了这些加权阶段对最终渲染结果的影响，从最初的传统probe渲染开始，逐步加入权重项，强调每个权重如何消除artifacts：

- 背面剔除位于着色点切平面下方的probe，设置一个soft threshold，当着色点法线与其到probe的方向之间的点乘小于该阈值时，则剔除。
- 人类视觉系统对于黑暗区域的低强度照明的敏感性，作者降低了少于5% irradiance范围的贡献
- VSM中描述的mean- and variance-biased Chebyshev interpolants :confused:
- 对着色点应用与着色点法线和其到probe方向成正比的bias偏移，避免visibility函数边界处不连续阴影的问题
- 使用上述权重与bias，根据着色点与probe中心的距离，执行三线性插值

<a name="Fig 4.3"></a>

<img src="/images/Paper Notes/GI/Dynamic%20Diffuse%20Global%20Illumination%20with%20Ray-Traced%20Irradiance%20Fields.assets/image-20230226145015593.png" alt="image-20230226145015593" style="zoom: 67%;" />

<center>图4.3 封闭房间场景下的irradiance插值与采样。d~f逐渐加入权重</center>

技术细节：之前的工作使用 $128\times 128 \times 6$ 的高精度cube map存储深度信息，而在应用本文的权重准则后，可以在不引入数值误差问题的情况下，降低至 $16 \times 16$ 的medium精度。

### 4.3 多次反射的全局光照

多次反射的计算是一个递归过程，每一次反射的间接光都是基于前一次反射的radiance buffer。本文的间接光计算是分销在多帧的，因此会带来time-lag artifact，这一点对于静态相机下的静态场景最明显，而对于视角、光源和几何是动态的场景，这种延迟在间接光反射是不易察觉的。



## Reference

<a name="[1]">[1]</a> Morgan McGuire, Mike Mara, Derek Nowrouzezahrai, and David Luebke. 2017. Real-time global illumination using precomputed light field probes. In Proceedings of the 21st ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D '17). Association for Computing Machinery, New York, NY, USA, Article 2, 1–11. https://doi.org/10.1145/3023368.3023378

<a name="[2]">[2]</a> Thaler, Jonathan. (2011). Deferred Rendering. URL: https://www.researchgate.net/profile/Jonathan_Thaler2/publication/323357208_Deferred_Rendering/links/5a8fce31aca272140560aaad/Deferred-Rendering.pdf

<a name="[3]">[3]</a> Ricardo Marques, Christian Bouville, Mickaël Ribardière, Luis Paulo Santos, Kadi Bouatouch. Spherical Fibonacci Point Sets for Illumination Integrals. Computer Graphics Forum, 2013, 32 (8), pp.134-143. ⟨10.1111/cgf.12190⟩. ⟨hal-01143347⟩
