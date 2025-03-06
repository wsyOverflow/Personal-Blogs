---
typora-copy-images-to: ..\..\..\images\Paper Notes\Cloud Rendering\${filename}.assets
typora-root-url: ..\..\..\
title: Radiance Caching with On-Surface Caches for Real-Time Global Illumination
keywords: radiance caching, cloud rendering
categories:
- [Paper Notes, Cloud Rendering]
mathjax: true
---

# 1 Summary

本文提出了在 On-Surface Caches 管线下的两种类型radiance cache，以及基于这两种radiance cache的GI方法。定义在texture space下的radiance cache相较于probe或者hash grid等方案，有更好的时序一致性、直接位于表面之上的避免漏光等优点，以及可以在multi-viewer下共享radiance信息。

两种类型的radiance cache缓存了表面区域的半球8x8分辨率的radiance信息：

- primary cache: cache entry分配自每个viewer的visibility buffer，即直接可见表面。存储了到达cache位置的 incoming radiance。可用于查询像素的incoming radiance或者通过SH查询收到的irradiance。

  cache entry 的更新：对于 diffuse 表面，使用 ray guiding 策略；对于 glossy 表面，使用 BRDF 重要性采样。

- secondary cache: cache entry分配在primary cache弹射后交到的表面位置，即secondary hit。存储了cache位置的 outgoing radiance，可以用于查询某光线方向的radiance信息

  cache entry 的更新：基于 ReSTIR 的直接光照着色以及在secondary cache中采样。secondary cache之间的采样可以实现无限反弹。

基于这两种类型的radiance cache，作者提出GI的计算：

- 像素的irradiance：在相邻两层mip level的primary cache之间执行trilinear插值
- 像素的反射：
  - 对于roughness较高的glossy，使用BRDF重要性采样primary cache的incoming radiance。由于 primary cache 是视角无关的radiance信息，尽管采样primary cache时使用BRDF重要性权重，仍然会受到限制。因此仅能应用于较粗糙的glossy表面。
  - 对于roughness较低(低于0.2)的光滑表面：光滑表面的反射与视角相关性更强，对此每个视角执行BRDF重要性采样，trace视角相关的光线。反射光线仍然采样交点处的secondary cache。



# 2 Background





# 3 Method

[[2]](#[2]) 提出的 On-Surface Caches (OSC)，构建位于场景模型表面之上的texture-based cache，用于缓存渲染的中间计算。这篇论文基于此cache策略，提供一种 visibility-based radiance caching 方法，同类方法有 [[3]](#[3]) [[4]](#[4])。不同于将最终radiance值存储在屏幕空间的screen-space方法，帧与帧之间不需要重投影，因为表面上的数据直接存放在时序一致的texture space。

## 3.1 Radiance Cache

本文的方法利用两种类型的radiance cache，两种cache存储半球面上的radiance，如 [Fig-3.1](#Fig-3.1):

- primary hit cache: 用于存储到达表面（当前视角）命中点位置的incoming radiance
- secondary hit cache: 存放outgoing radiance，本身也会被primary hit cache用于查询入射radiance。以及secondary hit cache直接可以互相采样，随着时间得到无限次弹射

[[4]](#[4]) 中的hash grid cells不采样彼此，并且只缓存了hash grid内的直接光照，因此限制了屏幕外的多次弹射。

> incoming radiance 与 outgoing radiance。下面渲染方程计算P点在 $\boldsymbol{v}$ 方向上的 outgoing radiance，计算过程则是对半球面上的incoming radiance以及BRDF积分。
> $$
> L_o(P, \boldsymbol{v})=\int_{\Omega^+}L_i(\boldsymbol{l})f(\boldsymbol{l}, \boldsymbol{v}) \cos\theta_l \space d\boldsymbol{l}
> $$
> irradiance。下面 diffuse 材质在 $\omega_o$ 方向上的outgoing radiance计算。由于diffuse brdf是常量，可以提出积分，因此得到先对incoming radiance的积分，该积分结果就是irradiance，(辐照度) 描述的是单位面积上接收到的总入射辐射能量
> $$
> L_o(\omega_o)=\int_{\Omega^+} \mathcal{C}\cdot L_i(\omega_i)\cdot \cos\theta_i\space d\omega_i=\mathcal{C}\cdot \int_{\Omega^+} L_i(\omega_i)\cdot \cos\theta_i\space d\omega_i
> $$

这两种radiance cache存储不同类型的radiance是有实际意义的。如果在 secondary hit cache 中存放 incoming radiance，那么当从 secondary hit cache 采样指定方向的outgoing radiance时，由渲染方程可知，需要执行一次积分，每次采样一次积分是不现实的；而在 primary hit cache 中存放 outgoing radiance，也就是直接查询着色点到视点方向的 outgoing radiance，这样cache在表面满足不了视口分辨率的精度。

<a name="Fig-3.1"></a>

![image-20241207155709420](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241207155709420.png)

<center>Fig-3.1 左图：两种类型cache之间交互以及与环境、光照的交互；右图：cache的spatial filter，只能在位于同一island的caches之间执行</center>

### 3.1.1 Primary Cache

<a name="Fig-3.2"></a>

![image-20241207164153230](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241207164153230.png)

<center>Fig-3.2 Primary radiance cache 的可视化。(a) 每个方形是一个 8x8 texels 的cache entry，存储incoming radiance；(b) 展示了hit distance，用于 detail-preserving filtering。</center>

primary cache entries 放置在由视点观察到的primary hits位置（即直接可见表面），存放了半球面上的incoming radiance数据，如图 [Fig-3.2](#Fig-3.2) 所示。半球分辨率为 8x8，每个 texel 4通道，rgb 存放 radiance，a 存放hit distance。相比于 grid 形式的probe放置，定义在texture space的cache entry能够保留更多几何表面细节。

OSC 的mip bias用于管理一个cache entry的覆盖表面的区域大小，作者给出最优范围 1.5~3.5。在更新cache entry时，会在其中抖动采样一个点，得到 triangle ID与重心坐标，这可以确定其世界坐标。triangle ID 的精度不足会导致图 [Fig-3.4](#Fig-3.4) 的问题。为此，作者将GI cache与triangle ID cache(visibility)的分辨率进行解耦，即可以使用更高分辨率的triangle ID，实验下来 triangle ID 的mip bias取-2最优，如图 [Fig-3.3](#Fig-3.3)。

primary cache entry 会为每个VB样本执行，确保覆盖所有可见表面。如果相邻像素位于不同的island，那么为会每个像素分配一个cache entry，这回导致cache entry数量以及计算负载的增加。对于这一点，未来探索的一个潜在方向是优化分配策略，特别是针对非常小的表面。在分配新的cache entry时，我们尝试使用相邻的mip level的信息来初始化它。在发生mip level切换时，已存在的cache entry数据会被用于新出现的相邻mip level，从而确保平滑的mip level过渡。

对于 diffuse 表面，利用缓存的radiance信息执行 ray guiding 策略 [[3]](#[3])[[4]](#[4]) 。对于单个viewer的 glossy 反射，使用 BRDF 重要性采样。在更新 cache entry 的 incoming radiance 时，会将rays样本加载到shared memory中以atomic操作进行累积。

对于光线交于发光表面或者环境光，获取直接光照。对于小的发光表面，它们可能根本无法被 ray guiding 采样到（尤其是考虑到 8 × 8 的有限分辨率），因此可能导致时序不稳定性。

<a name="Fig-3.3"></a>

![image-20241207171925197](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241207171925197.png)

<center>Fig-3.3 (a) 场景几何的实际triangle ID；(b) 方形表示了表面上的一个cache entry；(c) 2x2 下的triangle ID cache entry的三角形编号；(d) 4x4 下更精细的编号</center>

<a name="Fig-3.4"></a>

![image-20241207171132639](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241207171132639.png)

<center>Fig-3.4 (a) Black artifacts，由triangle ID对于cache表面的覆盖不充足，导致从cache到world的转换得到光线起点的过程匹配到相邻世界坐标；(b) 更精细的triangle ID可以明显缓解该问题</center>

### 3.1.2 Secondary Cache

irradiance probe 以及 [[4]](#[4]) hash grid 的radiance caching方法会有潜在的漏光问题，主要是由于其放置于空间或者接近表面的位置，而不是表面上。因此需要一些启发式策略避免漏光。而本文提出的secondary cache直接放置于表面。

secondary cache 缓存了半球面的outgoing radiance，能够采样任意方向，并且由于cache entry之间可以彼此采样，可以达到多次弹射的全局光照，如图 [Fig-3.6](#[Fig-3.6])。hemisphere采用 8x8 tile的分辨率。直接光照采用时序重采样的 ReSTIR [[5]](#[5]) 。对于多次弹射光照，投射了 32 条余弦加权的光线，以使secondary cache entries 能够互相采样，也可以使用更复杂的采样策略。outgoing radiance则是为每个cached direction对incoming radiance(ray radiance)应用BRDF，即每个光线会执行64次计算。

与primary cache不同，secondary cache以固定的表面大小进行放置，大概 20cm X 20cm 到 30cm X 30cm。对于大场景，可以在距离所有视角远的区域降低cache分辨率。secondary cache entry的分配发生于primary cache的光线交于一个表面时。在cache区域采样时，triangle ID的改进与primary cache相同。

<a name="Fig-3.5"></a>

![image-20241207180126279](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241207180126279.png)

<center>Fig-3.5 secondary radiance cache 的可视化。每个方形表示一个8x8 texels的cache entry，其中存放了outgoing radiance。secondary cache是紧密附着在表面上的，可以避免漏光。(a) 没有temporal filtering，比较噪；(b) filter后更平滑</center>

<a name="Fig-3.6"></a>

![image-20241208194928613](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241208194928613.png)

<center>Fig-3.6 (a) 单次弹射的全局光照，在比较暗的区域缺失了大量细节；(b) secondary cache采样彼此实现的无限反弹可以捕获的多弹射全局光照，明显增强了较暗角落的光照</center>

## 3.2 Cache Data Structure

primary/secondary cache 都具有 2x(8x8) 半精度 vec4，存放 radiance 和光线距离，应该是有一份历史信息；以及全精度的表面法线与位置 (是指采样点？)， 2 个vec3，法线与位置在初次访问cache时写入。primary cache 还需要 54 bytes SH系数，以及 4个64-bit 整数来存放相邻cache entries的地址。secondary cache 需要额外的 28 bytes 用于 reservoir。

因此，primary cache entry需要 1134 bytes，secondary cache entry 需要 1076 bytes。

- radiance与光线距离：2x8x8x4x2=1024 bytes；法线与位置：2x3x4=24bytes；SH系数：54 bytes；相邻地址：4x8=32 bytes；共 1024+24+54+32=1134bytes
- radiance与光线距离：1024 bytes；法线与位置：2x3x4=24bytes；reservoir：28 bytes；共 1024+24+28=1056 bytes

## 3.3 Temporal Filtering

OSC下temporal filtering不需要重投影，作者采用基于时间的 exponential moving average，blending factor为样本数的倒数，设置样本数范围 4~8，应对不同的响应速度。来自低样本数的噪声可以通过screen-space denoiser来减轻。对于来自 primary hit cache 的 glossy 反射，降低样本数至4以下。

## 3.4 Spatial Filtering

相邻的cache texel位于同一平面上，如图 [Fig-3.1](#Fig-3.1)，cache space下的filter没有必要进行depth/normal检查以及启发式策略。作者仅采用一个额外的blur，半径为3，权重为 1/(step+1) 的filter。

同一表面上也可能出现light bleeds，例如，Sponza场景中窗帘下方的光线溢出到地板上。作者使用 [[3]](#[3]) 提出的angle error detection 方法解决该问题。此外，在选择filter邻居时，引入视差矫正方向，避免过度模糊，如图 [Fig-3.7](#Fig-3.7)。

island 边界问题，位于不同的island的邻居无法被访问，这会导致一条明显的缝。这在定义不佳的island上尤为明显，例如，当一个平坦表面由多个islands组成时。这种在没有表面纹理的场景中非常显眼，但在许多应用了表面纹理的场景中，接缝的可见性会大大降低，如图 [Fig-3.8](#Fig-3.8) 所示。

<a name="Fig-3.7"></a>

![image-20241209110506190](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241209110506190.png)

<center>Fig-3.7 Sponza 场景窗帘被间接光照照亮。(a) filter过程中，为相邻cache entriy使用相同方向，导致细节丢失以及过渡平坦。(b) 使用视差矫正方向保留了光照特征。</center>

<a name="Fig-3.8"></a>

![image-20241209111702863](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241209111702863.png)

<center>Fig-3.8 由于spatial filter无法跨island边界，导致明显的缝隙。相接、平坦或者曲率很低的表面应该位于一个island上，来避免island边界的缝隙问题。最左边两张图中，被划分了多个不同的island，导致明显的缝隙；有时这些边界可能被surface texture隐藏，例如最右边两张图中</center>

## 3.5 Irradiance Evaluation

在相邻两个mip层级的primary cache之间执行trilinear插值得到像素的irradiance。primary cache可以转为SH提供diffuse illumination；或者应用半球上incoming radiance的 BRDF重要性采样，支持更细节的surface shading以及glossy reflections，如图 [Fig-3.9](#Fig-3.9)。trilinear 插值来自SH的irradiance是比较高效的，但重要性采样的trilinear插值相对较慢。对此，作者选择在三线性插值样本中进行基于插值权重的随机采样。这种方法将每像素的cache访问次数从 8 次减少到 1 次。然而，它会引入少量额外噪声，这可以通过temporal filter减轻。

<a name="Fig-3.9"></a>

![image-20241209113157530](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241209113157530.png)

<center>Fig-3.9 除窗帘外，roughness 设置为 0.4。(a)~(c) 是没有TAA或者其它filter，对primary hit cache的不同spp下的BRDF重要性采样。时间与每个VB像素采样primary cache相关；(d) 8spp下以及简单temporal filter的TAA；(e) path-traced reference，五次bounce累积40K帧。本文的方法成功捕捉了glossy反射。然而，与参考图像相比，某些高光未能完全重现，作者认为这与filter过程以及cache分辨率可能不足有关。</center>

## 3.6 Reflections

与AMD-GI1.1[[6]](#[6]) 相似，本文提供了两种处理反射的方式，多个视角可以在secondary hit时共享radiance信息并提供多次反射光照：

- glossy reflection 通过对primary hit hemisphere进行 BRDF 重要性采样来实现，而不是依赖球谐函数（如图 [Fig-3.10](#Fig-3.10)）。由于半球已经捕获了incoming radiance，因此无需进一步进行光线追踪。为了提供更好的反射质量，incoming radiance应通过 BRDF 重要性采样追踪，而不是使用ray guiding方案。然而，需要注意的是，**这种方法在缺少单一视角的multi-viewer场景中会遇到限制**。虽然ray guiding仍然是一个选项，但这会以降低反射质量为代价。为了确保反射的响应性，重要的是减少主要缓存的时间滤波强度，如第 3.3 节所述。
- sharp reflections 是通过使用专门的反射光线来实现的，如图 [Fig-3.11](#Fig-3.11) 所示。这些光线的radiance从secondary cache中获取，其中已缓存了outgoing radiance。secondary cache支持multi-viewer场景，因为每个viewer可以发射独立的反射光线，同时共享存储在secondary cache中radiance信息。这是因为outgoing radiance是按半球存储的。实际实现中根据材质属性动态决定**是否发射反射光线**，特别是在粗糙度低于特定阈值（本文 0.2）时。

> 这两种方法应该是适应不同的粗糙度水平。
>
> - 对于粗糙度较高的glossy表面，反射光线的radiance从primary cache中采样，由于primary cache中的incoming radiance已经包含了secondary cache中的outgoing radiance，因此有无限反弹的信息。注意，primary cache是视角无关的，虽然反射采样incoming radiance是可以使用BRDF重要性权重，但视角相关的反射仍然会缺失信息。
> - 对于粗糙度低的光滑表面，需要额外trace反射光线，反射光线的radiance来自于secondary cache的outgoing radiance。光滑表面的反射与视角更加强相关，这里要每个view单独trace反射光线。似乎这也就是说，这里要为反射光线生成secondary cache entry

通过在两种方法中都依赖次级缓存中的入射辐射，我们的系统通过反射提供了多次反射光照，如图 [Fig-3.12](#Fig-3.12) 所示。然而，需要注意的是，这两种方法仍然会出现一些噪声，因此需要进行screen-space filer。

<a name="Fig-3.10"></a>

![image-20241209132939923](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241209132939923.png)

<center>Fig-3.10 重要性采样primary cache、path-traced reference与AMD-GI1.1之间的对比。尽管secondary cache entry覆盖的区域要大得多，但它们为primary cache生成了令人满意的光滑反射，接近路径追踪的参考结果。对primary cache进行重要性采样更适合处理较大物体的反射，并且粗糙度的最低值大约为 0.2。
</center>

<a name="Fig-3.11"></a>

![image-20241209133723897](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241209133723897.png)

<center>Fig-3.11 在VB的ray-traced反射中，secondary cache为反射光线提供radiance，消除了进一步光线投射的昂贵开销。左图，具有近似镜面反射的地板；secondary cache覆盖范围广，偶尔会出现在反射中。由于缓存分辨率较低，表面细节（如黄色便签上的画）受到影响。右图，粗糙度为 0.1，场景显得更平滑，使得secondary cache变得更难以辨认。</center>

<a name="Fig-3.12"></a>

![image-20241209134052245](/images/Paper Notes/Cloud Rendering/Radiance Caching with On-Surface Caches for Real-Time Global Illumination.assets/image-20241209134052245.png)

<center>Fig-3.12 secondary cache 能够为 ray-traced reflections 提供无限反弹。左图没有无限反弹；中间使用secondary cache达到的无限反弹；右图是AMD-GI1.1的效果，无法提供屏幕外的无限反弹</center>

# Reference

<a name="[1]">[1]</a> Wolfgang Tatzgern, Alexander Weinrauch, Pascal Stadlbauer, Joerg H. Mueller, Martin Winter, and Markus Steinberger. 2024. Radiance Caching with On-Surface Caches for Real-Time Global Illumination. Proc. ACM Comput. Graph. Interact. Tech. 7, 3, Article 38 (August 2024), 17 pages.

<a name="[2]">[2]</a> Alexander Weinrauch. 2023. Effect-based Multi-viewer Caching for Cloud-native Rendering. *siggraph* (2023).

<a name="[3]">[3]</a> Daniel Wright, Krzysztof Narkowicz, and Patrick Kelly. 2022. Lumen: Real-time Global Illumination in Unreal Engine 5. https://advances.realtimerendering.com/s2022/index.html

<a name="[4]">[4]</a> Guillaume Boissé, Sylvain Meunier, Heloise de Dinechin, Pieterjan Bartels, Alexander Veselov, Kenta Eto, and Takahiro Harada. 2023. GI-1.0: A Fast and Scalable Two-level Radiance Caching Scheme for Real-time Global Illumination. 

<a name="[5]">[5]</a> Benedikt Bitterli, Chris Wyman, Matt Pharr, Peter Shirley, Aaron Lefohn, and Wojciech Jarosz. 2020. Spatiotemporal reservoir resampling for real-time ray tracing with dynamic direct lighting. ACM Transactions on Graphics (TOG) 39, 4 (2020), 148–1  

<a name="[6]">[6]</a> Kenta Eto, Sylvain Meunier, Takahiro Harada, and Guillaume Boissé. 2023. Real-Time Rendering of Glossy Reflections Using Ray Tracing and Two-Level Radiance Caching. In SIGGRAPH Asia 2023 Technical Communications  
