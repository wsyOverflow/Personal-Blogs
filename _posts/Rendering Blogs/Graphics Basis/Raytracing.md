---
typora-copy-images-to: ..\..\..\images\Rendering Blogs\Graphics Basis\${filename}.assets
typora-root-url: ..\..\..\
title: Raytracing
keywords: Graphics, Raytracing
categories:
- [Rendering Blogs, Raytracing]
mathjax: true
---



# 2 Raycasting

raycasting 仅会投射两种光线：

- 从eye出发到场景的光线，得到最近交点，作为着色点
- 从着色点到光源投射光线，即shadow ray

# 3 Recursive Whitted Raytrace

引入了反射、折射、阴影光路，虽然加入了间接光，但只支持完美镜面反射、折射的递归追踪，其过程如下：

- 从eye出发到场景的光线，得到最近交点
- 当交点为镜面时，继续trace镜面反射光线
- 当交点为折射材质时，trace折射光线进入物体内部，再得到物体内部出射光线
- 向光源投射shadow ray

<img src="/images/Rendering Blogs/Graphics Basis/Raytracing.assets/image-20250405153338592.png" alt="image-20250405153338592" style="zoom: 50%;" />

## 3.1 光线可逆

对于现实世界而言，光线由光源发出经过反射等过程到达眼睛，形成颜色，这种称为 Forward Raytracing。这种过程对于计算机而言非常低效，因为会有很多光线无法到达眼睛。因此，更常见的是 Backward Raytracing，即从眼睛发出光线。

## 3.2 算法框架

```c++
Trace(ray) 
  object_point = Closest_intersection(ray)
  if object_point return Shade(object_point, ray)
  else return Background_Color
    
Shade(point, ray)
  radiance = black; /*初始化 */
  for each light source
    shadow_ray = calc_shadow_ray(point,light)
    if !in_shadow(shadow_ray,light)
        radiance += phong_illumination(point,ray,light)
    if material is specularly reflective
         radiance += spec_reflectance * Trace(reflected_ray(point,ray)))
    if material is specularly transmissive
        radiance += spec_transmittance * Trace(refracted_ray(point,ray)))
    return radiance
```

## 3.3 分布式光线追踪 (DRT)

分布式光线追踪的主要思想是，之前我们对每一个像素进行超采样可以进行求平均值的方式来进行抗锯齿的操作，那么也可以不光只在一开始eye Ray发出多个射线，也可以在每次反射折射时也发出多条射线，这能做到比whitted光线追踪的单条反射折射射线更多的事情和效果。并且引入了如下处理，提升真实感：

- 使用非均匀（抖动，jittered）采样
- 使用噪点noise代替图像失真的情况
- 提供了一些特效，如Glossy reflections，Soft shadow，Motion Blur，Time等。

![img](/images/Rendering Blogs/Graphics Basis/Raytracing.assets/v2-9a46e0df5fb57eb3109198f3869235e5_r.jpg)

# 4 Path Tracing

路径追踪是分布式光线追踪的一种变体，其中每个点只发出一条反射和折射光线。这样可以避免光线数量的激增，但太简单的实现会导致拥有非常多噪点的图像。为了弥补这一点，每个像素都对多条光线进行追踪。简而言之，分布式光线追踪在光线树中发出的光线最多，而路径追踪追踪在初始时发出的光线最多。

渲染方程有两种形式：

- 在半球面上对立体角的积分
  $$
  L_o(x, \boldsymbol{v})=L_e(x, \boldsymbol{v})+\int_{\Omega^+}L_i(\omega)f_r(\omega, \boldsymbol{v}) \cos\theta_l \space d\omega \label{render-equation-hemisphere}\tag{1}
  $$

- 对面积的积分，由 $d\omega=dA/r^2$，有
  $$
  L(x'\rightarrow x) = L_e(x'\rightarrow x) + \int_M L(x''\rightarrow x') f_r(x''\rightarrow x' \rightarrow x)G(x'' \leftrightarrow x')\space dA_{x''} \label{render-equation-area}\tag{2}
  $$

> 这两种积分在形式上有所不同，$\eqref{render-equation-hemisphere}$ 使用的是方向，$\eqref{render-equation-area}$ 使用路径描述。
>
> $\eqref{render-equation-area}$ 描述的是经过 $x'$ 到达着色点 $x$ 的irradiance，$r^2$ 隐藏在 $G$ 项中



## 4.1 算法框架

```c++
computeImage(eye)
  for each pixel
    radiance = 0;
    H = integral(h(p));
    for each sample // Np viewing rays
      pick sample point p within support of h;
      construct ray at eye, direction p-eye;
      radiance = radiance + rad(ray)*h(p);
  radiance = radiance/(#samples*H);

rad(ray)
  find closest intersection point x of ray with scene;
  return(Le(x,eye-x) + computeRadiance(x, eye-x));

computeRadiance(x, dir)
   estimatedRadiance += directIllumination(x, dir);
   estimatedRadiance += indirectIllumination(x, dir);
   return(estimatedRadiance);

directIllumination (x, theta)
  estimatedRadiance = 0;
  for all shadow rays // Nd shadow rays
    select light source k;
    sample point y on light source k;
    estimated radiance +=Le * BRDF * radianceTransfer(x,y)/(pdf(k)pdf(y|k));
  estimatedRadiance = estimatedRadiance / #paths;
  return(estimatedRadiance);

indirectIllumination (x, theta)
   estimatedRadiance = 0;
   if (no absorption) // Russian roulette
     for all indirect paths // Ni indirect rays
        sample direction psi on hemisphere;
        y = trace(x, psi);
        estimatedRadiance += compute_radiance(y, -psi) * BRDF *cos(Nx, psi)/pdf(psi);
     estimatedRadiance = estimatedRadiance / #paths;
   return(estimatedRadiance/(1-absorption));

radianceTransfer(x,y)
  transfer = G(x,y)*V(x,y);
  return(transfer);。
```

# 5 双向路径追踪 (Bidirectional Path Tracing)



# 6 Metropolis 光线追踪



https://zhuanlan.zhihu.com/p/72673165