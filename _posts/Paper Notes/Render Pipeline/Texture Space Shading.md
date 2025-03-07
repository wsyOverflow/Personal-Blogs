---
typora-copy-images-to: ..\..\..\images\Paper Notes\Render Pipeline\${filename}.assets
typora-root-url: ..\..\..\
title: Texture Space Shading
keywords: render pipeline, texture space shading
categories:
- [Paper Notes, Render Pipeline]
mathjax: true
---

# 1 Summary

传统的光栅化管线（forward或者deferred）执行于屏幕空间，表面采样率或者像素密度与相机距离、视角相关。同时，GPU光栅化管线的流水线作业使得shading rate完全由分辨率、几何复杂度、帧率控制。而texture space shading(TSS)提出在texture space着色，使得shading rate变得更加可控。同时texture space下的texel着色具有更优的spatial coherent以及temporal coherent，这对于时空复用类算法更加友好。

概况地讲，TSS的提出主要有如下动机：

1. 可控的表面像素采样率。对于远处高光的小物体，相机的运动会导致非常不稳定的效果。例如，从正对着表面运动到grazing视角，表面变得更加高频，而采样率不足。TSS的着色受相机视角影响较小。
2. 更加准确的帧间复用。不同的观察位置、角度下，相邻帧之间的表面像素的不匹配程度较高，例如可能处于不同的mip level。传统光栅化管线在时序上的复用难以考虑这一点，而TSS着色结果cache在texture space，时序连续性较高。
3. 更加准确的空间复用。屏幕空间的像素之间暴露出incoherent性质，这对于空间复用很不友好。而texture space下，相邻像素往往满足世界空间的coherent。

# 2 Category

本文按照 [[1]](#[1]) 给出的分类方法描述TSS的相关工作，以及设想中的大致实现思路。

<img src="/images/Paper Notes/Render Pipeline/Texture Space Shading.assets/image-20230628191733254.png" alt="image-20230628191733254" style="zoom:80%;" />







# Reference

<a name="[1]">[1]</a> Neff, T., Mueller, J.H., Steinberger, M. and Schmalstieg, D. (2022), Meshlets and How to Shade Them: A Study on Texture-Space Shading. Computer Graphics Forum, 41: 277-287. https://doi.org/10.1111/cgf.14474