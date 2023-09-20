---
typora-copy-images-to: ..\..\..\images\Paper Notes\Render Pipeline\${filename}.assets
typora-root-url: ..\..\..\
title: Texel Shading
keywords: render pipeline, texture space shading
categories:
- [Paper Notes, Render Pipeline]
mathjax: true
---

# 1 Summary

传统的光栅化管线着色（forward/deferred）是基于屏幕空间像素的着色过程，由于光栅化流水线作业的限制，shading rate完全由分辨率、几何复杂度、帧率控制。而本文提出的texel shading管线将着色转化为texture space的任务，并将着色结果cache在texture space，最终的着色通过采样cache完成。

基于texture space的着色将shading rate与分辨率、几何复杂度、帧率解耦，可以每像素调整shading rate，变得更加可控。同时texture space下对于时空复用类算法更加友好。概况地讲，texture space shading(TSS)的提出主要有如下动机：

1. 可控的表面像素采样率。对于远处高光的小物体，相机的运动会导致非常不稳定的效果。例如，从正对着表面运动到grazing视角，表面变得更加高频，而采样率不足。TSS的着色受相机视角影响较小。
2. 更加准确的帧间复用。不同的观察位置、角度下，相邻帧之间的表面像素的不匹配程度较高，例如可能处于不同的mip level。传统光栅化管线在时序上的复用难以考虑这一点，而TSS着色结果cache在texture space，时序连续性较高。
3. 更加准确的空间复用。屏幕空间的像素之间暴露出incoherent性质，这对于空间复用很不友好。而texture space下，相邻像素往往满足世界空间的coherent。

# 2 Pipeline设计

texel shading的前提是使用某种参数化方式将场景的mesh统一到唯一的texture space下，即每一个surface point都具有唯一的texture坐标。这种参数化方式在TSS技术中也是一大问题，本文未做这方面的详细调研。texel shading管线共包含三个阶段：

1. **Shade Queuing**：使用一个geometry pass来标记需要着色的texel tiles(8x8)，每个texel tile作为着色任务。再新增着色任务前会先查询cache中是否已存在。
2. **Shade Evaluation**：compute shader中提取着色任务需要的顶点属性，并生成着色结果到cache texture中。
3. **Shade Gathering**：再执行一次geometry pass，从cache texture中收集着色结果，得到最终屏幕着色。

## 2.1 预处理

为了能够在compute shader中得到几何信息，还需要在预处理阶段生成triangle index texture，即texture space下纹理坐标到triangle ID的映射。

**实现设计**：对整个texture space分辨率进行一次光栅化即可得到triangle index texture。

## 2.2 Shade Queuing

本阶段通过geometry pass为每个像素生成texel单位的shade任务，并选择合适的mip level。通过一张cache texture来缓存每个8x8 tile的状态，例如最近更新的帧数，可以控制tile的更新频率。8x8 tile的shade任务生成算法如下：

<img src="/images/Paper Notes/Render Pipeline/Texel Shading.assets/image-20230703182339279.png" alt="image-20230703182339279" style="zoom: 50%;" />

> 算法：geometry pass生成的texel任务(i, j, L)转为 8x8 tile任务(i/8. j/8, L-3)。一个L-3级别上的8x8 tile可以计算得到一个L的texel。

这里可以看出控制shading rate的方式有：

- tile的更新频率阈值（多少帧更新一次）
- texel的mip level越大，意味着越少数量的texel，一个texel可以覆盖多个像素，这适用于低频区域。

**实现设计**：

- geometry pass需要对带纹理坐标的几何进行光栅化，得到每个像素的纹理坐标、以及重心坐标。
- cache texture：从算法实现上看，cache texture应该是N-3层级的贴图，给定shade texel(i, j, L)，其在cache中的坐标为(i/8, j/8, L-3)。cache texture的texel是对应tile的状态信息：
  - 上一次更新的帧数
  - bias配置：提高mip level来减少shade texel
- 如果在本阶段的geometry pass中计算的mip level向上取整，一个8x8 tile的着色可以在Shade Gathering阶段做到三线性插值。

## 2.3 Shade Evaluation

本阶段执行于compute shader，每个thread group执行一个8x8 tile。通过texel的纹理坐标从triangle index texture得到当前triangle ID，并根据此ID得到triangle数据。使用texel的重心坐标进行插值得到texel的顶点数据。之后便可以按照传统渲染的方式对texel着色。

## 2.4 Shade Gathering

此时，cache中已经有了屏幕着色需要的所有数据。再执行一次geometry pass得到对应level的texel着色。