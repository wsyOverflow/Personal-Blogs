---
typora-copy-images-to: ..\..\..\images\Rendering Blogs\GPGPU\${filename}.assets
typora-root-url: ..\..\..\
title: SlabHash
keywords: GPGPU, Coherent
categories:
- [Rendering Blogs, GPGPU, Scan]
mathjax: true
---

# 1 Introduction





# 2 2-Phase Parallel Scan

使用一种在并行计算中经常出现的算法模式：平衡树。其思想是，在输入数据上构建一棵平衡的二叉树，并通过自根向上和向下的遍历来计算前缀和。一棵有n*n*个叶子的二叉树有log⁡nlog*n*层，每一层d∈[0,n)*d*∈[0,*n*)有2d2*d*个节点。如果我们对每个节点执行一次加法操作，那么在一次树的遍历中，我们将执行O(n)*O*(*n*)次加法操作。

## 2.1 Reduce Phase (Up-Sweep Phase)



<img src="/images/Rendering Blogs/GPGPU/Parrallel Scan.assets/image-20240820110843745.png" alt="image-20240820110843745" style="zoom:67%;" />

## 2.2 Down-Sweep Phase







# Reference

1. Guy E. Blelloch. “Prefix Sums and Their Applications”. In John H. Reif (Ed.), Synthesis of Parallel Algorithms, Morgan Kaufmann, 1990.
   http://www.cs.cmu.edu/afs/cs.cmu.edu/project/scandal/public/papers/CMU-CS-90-190.html  
