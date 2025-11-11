---
typora-copy-images-to: ..\..\..\images\Paper Notes\Ray Tracing\${filename}.assets
typora-root-url: ..\..\..\
title: ReSTIR GI- Path Resampling for Real-Time Path Tracing
keywords: ray-tracing, ReSTIR
categories:
- [Paper Notes, Ray Tracing]
mathjax: true
---



# 1 Summary

本文在 path tracing 框架下，应用 ReSTIR 理论来提高 multi-bounce 全局光照质量。



NEE(next event estimation)

Next Event Estimation(NEE)：在path tracing计算全局光照时，对交点进行着色拆分为两部分，直接光照与间接光照。直接光照通过显式采样光源直接计算得到，即 light sampling。目的是解决传统路径追踪在直接光照计算上的低效问题，采样过程可结合多重重要性采样等采样技术来进一步提升其性能与质量。



本文基于 [[2]](#[2]) 的 ReSTIR 理论，在 path tracing 框架下提升 multi-bounce 全局光照质量。文中算法从像素出发，执行path tracing的NEE与MIS。在计算间接光时，采用基于BSDF的源分布，追踪得到采样点。文中算法仅维护屏幕空间大小的buffer：

- Initial sample buffer：维护一段采样点到可见点的光路，包括以下数据
  <img src="/images/Paper Notes/Ray Tracing/ReSTIR GI- Path Resampling for Real-Time Path Tracing.assets/image-20250326141236086.png" alt="image-20250326141236086" style="zoom: 60%;" />
- Temporal reservoir buffer：执行 temporal WRS
- Spatial reservoir buffer：执行 spatial WRS

<img src="/images/Paper Notes/Ray Tracing/ReSTIR GI- Path Resampling for Real-Time Path Tracing.assets/image-20250326141540223.png" alt="image-20250326141540223" style="zoom: 50%;" />

<img src="/images/Paper Notes/Ray Tracing/ReSTIR GI- Path Resampling for Real-Time Path Tracing.assets/image-20250326142124205.png" alt="image-20250326142124205" style="zoom:67%;" />

算法流程：

1. **初始采样**：从每个可见点（红点）随机方向发射光线，记录屏幕空间初始样本缓冲区中最接近的交点。记录交点的位置、法线和outgoing radiance，下一事件估计使用的随机数，以及像素的位置和法线。如果 outgoing radiance 用直接光照计算，算法得到的就是 1 bounce GI；如果用 n-bounce 的 PT w. NEE 计算，那么算法得到的就是 n+1-bounce GI。注意到这里 outgoing radiance 只保存了一个值而没有保存方向，因此后面重用采样点，连接到其他像素时忽略了方向。
2. **时间重用**：通过随机选择当前帧生成的新样本或缓冲区中已有的旧样本，利用初始样本缓冲区的样本更新时间蓄水池缓冲区。应用temporal reprojection 技术，插值上一帧中对应的 temporal reservoir。
3. **空间重用**：随机选择邻域像素的 spatial reservoir 来更新 spatial reservoir。为降低偏差，通过比较深度和法线与当前像素的相似性，选择具有相近几何特征的邻域像素。



ReSTIR GI 这套应用 ReSTIR 的过程，可以用到 world probe 的radiance上。





# References

<a name="[1]">[1]</a> Yaobin Ouyang, Shiqiu Liu, Markus Kettunen, Matt Pharr, and Jacopo Pantaleoni. 2021. ReSTIR GI: Path resampling for real-time path tracing. In *Computer Graphics Forum*, 2021. Wiley Online Library, 17–29.

<a name="[2]">[2]</a> Benedikt Bitterli, Chris Wyman, Matt Pharr, Peter Shirley, Aaron Lefohn, and Wojciech Jarosz. 2020. Spatiotemporal reservoir resampling for real-time ray tracing with dynamic direct lighting. <i>ACM Trans. Graph.</i> 39, 4, Article 148 (August 2020), 17 pages.