---
typora-copy-images-to: ..\..\..\images\Rendering Blogs\Graphics GPU\${filename}.assets
typora-root-url: ..\..\..\
title: Mesh Shader
keywords: mesh shader, Graphics
categories:
- [Rendering Blogs, Graphics GPU]
mathjax: true
---



# 1 传统的 Geometry Stage

## 1.1 Geometry Pipeline

在图元送入rasterizer之前，会经过下图所示几何管线，其中包括4种着色器：

- vertex shader：变换vertex buffer中的输入数据为适合光栅化的形式，每个 vertex shader 仅能处理单个顶点
- hull shader：作用于高阶控制面片 (control patch) 的控制顶点
- domain shader：处理一维或二维域中的采样点，与hullshader共同构成硬件曲面细分 (tessellation) 的可编程阶段
- geometry shader：处理点、线或三角面片图元，并能生成一个或多个新图元

<img src="/images/Rendering Blogs/Graphics GPU/Mesh Shader.assets/image-20250328101543275.png" alt="image-20250328101543275"  />

以 index buffer/vertex buffer 组成的几何数据为例，vertex shader 需要为 IB 中引用的每个顶点执行，这样就导致同一个顶点被重复执行多次 vertex shader。在Next Generation Geometry (NGG) 技术中，引入了顶点重用机制。

## 1.2 NGG & Primitive Subgroup

Vertex Shader 的发起不是对每个顶点执行，而是以 primitive subgroup 为单位，geometry engine会扫描其中三角形的顶点索引，并生成唯一顶点索引列表 (Unique Vertices)，最后再对唯一列表中的顶点执行vertex shader。在 primitive subgroup 中的同一顶点仅会发出一次vertex shader。

<img src="/images/Rendering Blogs/Graphics GPU/Mesh Shader.assets/image-20250327235134084.png" alt="image-20250327235134084" style="zoom:50%;" />

但对于不同 primitive subgroup 内的顶点处理过程是独立的，因此为了最大化顶点重用率，理想情况下希望所有引用同一顶点的图元都位于同一个图元子组中。也就是说，每个顶点仅会存在于一个图元子组。顶点缓存优化则是为了能最大化geometry engine对顶点的重用，[zeux’ meshoptimizer](https://github.com/zeux/meshoptimizer) 等库试图近似模拟 GPU 的顶点缓存行为，并据此对index buffer进行重新排序。共享同一顶点的图元会被聚类在一起，从而提高了在顶点缓存中找到该顶点、或将这些图元全部打包到同一图元子组的概率。这种预处理可以且应该静态应用于任何三角形网格。

<img src="/images/Rendering Blogs/Graphics GPU/Mesh Shader.assets/image-20250328000759681.png" alt="image-20250328000759681" style="zoom:50%;" />



https://gpuopen.com/learn/mesh_shaders/mesh_shaders-index/