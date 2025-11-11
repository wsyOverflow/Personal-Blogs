---
typora-copy-images-to: ..\..\..\images\Render Engine\PBRT\${filename}.assets
typora-root-url: ..\..\..\
title: 13 Light Transport
categories:
- [Render Engine, PBRT]
mathjax: true
---

# 1 Light Transport Equation

LTE (Light Transport Equation) 的核心原理是能量守恒。

- 宏观尺度上：功率守恒。物体出射功率 $\Phi_o$ 与入射功率 $\Phi_i$ 的差值等于其发射功率 $\Phi_e$ 与吸收功率 $\Phi_a$ 的差值
  $$
  \Phi_o - \Phi_i = \Phi_e - \Phi_a
  $$

- 微观尺度上：表面上辐射亮度平衡。出射radiance $L_o$ 必须等于其自发光辐射亮度 $L_e$ 加上入射 radiance 经散射后的比例
  $$
  L_o(p,\omega_o)=L_e(p,\omega_o)+\int_{S^2}f(p,\omega_o,\omega_i)L_i(p,\omega_i)|\cos\theta_i|\space d\omega_i \label{int-incoming-radiance} \tag{1}
  $$

LTE 是光传输算法的数学框架。

## 1.2 LTE 两种积分形式

### 1.2.1 球面方向积分形式

假设无传播介质的场景，即radiance沿光线在场景中传播时保持恒定。定义 ray casting functiuon $t(p, \omega)$，计算从 $p$ 点沿方向 $\omega$ 发出光线与场景的最近交点 $p'$。因此可以将点 $p$ 处在 $\omega$ 方向上的入射radiance与 $p'$ 的出射radiance关联起来：
$$
L_i(p, \omega) = L_o(t(p,\omega),-\omega)
$$
将到达着色点的入射radiance转为其它表面上一点的出射radiance，LTE 可以重写为
$$
L_o(p,\omega_o)=L_e(p,\omega_o)+\int_{S^2}f(p,\omega_o,\omega_i)L(t(p,\omega_i),-\omega_i)|\cos\theta_i|\space d\omega_i \label{int-exitant-radiance} \tag{2}
$$

### 1.2.2 表面积分形式

$\eqref{int-exitant-radiance}$ 式的复杂在于场景中几何对象之间的关系被隐式封装在光线追踪函数 $t(p,\omega_i)$ 中。这里将 LTE 的球面方向积分域重写为表面上的积分形式，使用两点路径替代方向，$\omega$ 为 $(p - p')$ 方向，定义 $p'$ 到 $p$ 的出射radiance
$$
L(p'\rightarrow p) = L(p', \omega) \label{exitant-radiance} \tag{3}
$$
$p'$ 处的 BSDF 可以定义为（入射方向上的光在给定出射方向上的分布，$\omega_o$ 为 $(p-p')$ 方向，$\omega_i$ 为 $(p''-p')$ 方向）
$$
f(p'' \rightarrow p' \rightarrow p) = f(p', \omega_o, \omega_i) \label{3-points-BSDF} \tag{4}
$$
<img src="/images/Render Engine/PBRT/13 Light Transport.assets/image-20250405223426398.png" alt="image-20250405223426398" style="zoom:80%;" />

<center>Fig-1: BSDF的三点形式将LTE的积分域从球面方向转为表面点</center>

此外还要将立体角微元转为表面积分的微元面积 $dA$，关系为 $d\omega=dA/r^2$。将这个替换项 $1/r^2$ 与原LTE中的余弦项、可见性一起写入几何项
$$
G(p\leftrightarrow p') = V(p \leftrightarrow p')\frac{|\cos\theta||\cos\theta'|}{||p-p'||^2} \label{gemotric-term} \tag{5}
$$
将重写后的出射radiance $\eqref{exitant-radiance}$、BSDF $\eqref{3-points-BSDF}$、几何项 $\eqref{gemotric-term}$ 代入 LTE，得到 LTE 的3-points形式，其中 $A$ 是场景中所有表面
$$
L(p'\rightarrow p) = L_e(p'\rightarrow p) + \int_A f(p'' \rightarrow p' \rightarrow p)\cdot L(p''\rightarrow p')\cdot G(p''\leftrightarrow p')\space dA(p'') \label{LTE-surface} \tag{6}
$$
LTE 两种形式 $\eqref{int-exitant-radiance}$ 与 $\eqref{LTE-surface}$ 虽然是等价的，但它们代表了处理光线传输的两种不同的处理方法：

- 如果使用蒙特卡洛方法求解方程 $\eqref{int-exitant-radiance}$，需要在球面方向分布中采样方向并投射光线以计算被积函数
- 而对于方程 $\eqref{LTE-surface}$，则需要根据表面积分布采样表面点，计算这些点之间的几何项（通过光线追踪评估可见性 V）以计算被积函数
  这个更适合 radiosity patch 的方法。

> 关于二者是等价的，注意不要被符号混淆。$\eqref{int-exitant-radiance}$ 中计算点 $p$ 在 $\omega_o$ 方向上的出射radiance，$\eqref{LTE-surface}$ 计算的是点 $p'$ 在 $p' \rightarrow p$ 方向上的出射radiance。也就是说，在 $\eqref{int-exitant-radiance}$ 中使用 $p$ 表示着色点，而在 $\eqref{LTE-surface}$ 中使用 $p'$ 表示着色点。

## 1.3 Integral over Paths

通过 $\eqref{LTE-surface}$ 的表面积分形式，可以推导出光线传输的路径积分形式 (path integral)。这种显式形式为路径生成提供了极大的自由度——可以适用于任何路径采样技术，这也为双向光传输算法 (bidrectional light transport algorithms) 提供了基础。

### 1.3.1 LTE 光路展开

为了将表面积分转换为不同长度光路的路径积分之和，可以不断将 3-points-LTE 形式展开并替换右侧积分中的 $L(p''\rightarrow p')$ 项，每次展开即延长一段光路。为了直观理解，改为数字下标，$\eqref{LTE-surface}$ 变换为：
$$
L(\mathrm{p}_1\rightarrow \mathrm{p}_0) = L_e(\mathrm{p}_1\rightarrow \mathrm{p}_0) + \int_A f(\mathrm{p}_2 \rightarrow \mathrm{p}_1 \rightarrow \mathrm{p}_0)\cdot L(\mathrm{p}_2\rightarrow \mathrm{p}_1)\cdot G(\mathrm{p}_2\leftrightarrow \mathrm{p}_1)\space dA(\mathrm{p}_2)
$$
将 $L(\mathrm{p}_2\rightarrow \mathrm{p}_1)$ 展开为 3-points-LTE 形式有
$$
L(\mathrm{p}_2\rightarrow \mathrm{p}_1) = L_e(\mathrm{p}_2\rightarrow p_1) + \int_A f(\mathrm{p}_3 \rightarrow \mathrm{p}_2 \rightarrow \mathrm{p}_1)\cdot L(\mathrm{p}_3\rightarrow \mathrm{p}_2)\cdot G(\mathrm{p}_3\leftrightarrow \mathrm{p}_2)\space dA(\mathrm{p}_3)
$$
代入到 $L(\mathrm{p}_1\rightarrow \mathrm{p}_0)$ 有
$$
\begin{align}
L(\mathrm{p}_1\rightarrow \mathrm{p}_0) &= L_e(\mathrm{p}_1\rightarrow \mathrm{p}_0) \\
&+ \int_A f(\mathrm{p}_2 \rightarrow \mathrm{p}_1 \rightarrow \mathrm{p}_0)\cdot \left\{L_e(\mathrm{p}_2\rightarrow \mathrm{p}_1) + \int_A f(\mathrm{p}_3 \rightarrow \mathrm{p}_2 \rightarrow \mathrm{p} _1)\cdot L(\mathrm{p}_3\rightarrow \mathrm{p}_2)\cdot G(\mathrm{p}_3\leftrightarrow \mathrm{p}_2)\space dA(\mathrm{p}_3)\right\}\cdot G(\mathrm{p}_2\leftrightarrow \mathrm{p}_1)\space dA(\mathrm{p}_2)\\
&=L_e(\mathrm{p}_1\rightarrow \mathrm{p}_0) \\
&+\int_A f(\mathrm{p}_2 \rightarrow \mathrm{p}_1 \rightarrow \mathrm{p}_0)\cdot L_e(\mathrm{p}_2\rightarrow \mathrm{p}_1) \cdot G(\mathrm{p}_2\leftrightarrow \mathrm{p}_1) \space dA(\mathrm{p}_2)\\
&+\int_A \int_A f(p_3 \rightarrow p_2 \rightarrow \mathrm{p}_1)\cdot L(\mathrm{p}_3\rightarrow \mathrm{p}_2)\cdot G(\mathrm{p}_3\leftrightarrow \mathrm{p}_2)\cdot f(\mathrm{p}_2 \rightarrow \mathrm{p}_1 \rightarrow \mathrm{p}_0)\cdot G(\mathrm{p}_2\leftrightarrow \mathrm{p}_1)\space dA(\mathrm{p}_3)\space dA(\mathrm{p}_2)
\end{align}
$$
再继续展开 $L(\mathrm{p}_3\rightarrow \mathrm{p}_2)$，以此类推有：
$$
\begin{align}
L(\mathrm{p}_1 \rightarrow \mathrm{p}_0) &= L_e(\mathrm{p}_1 \rightarrow \mathrm{p}_0) \\
&+ \int_A L_e(\mathrm{p}_2\rightarrow \mathrm{p}_1)f(\mathrm{p}_2\rightarrow \mathrm{p}_1 \rightarrow \mathrm{p}_0)G(\mathrm{p}_2 \leftrightarrow \mathrm{p}_1) \space dA(\mathrm{p}_2)\\
&+ \int_A \int_A L_e(\mathrm{p}_3\rightarrow \mathrm{p}_2)f(\mathrm{p}_3\rightarrow \mathrm{p}_2 \rightarrow \mathrm{p}_1)G(\mathrm{p}_3 \leftrightarrow \mathrm{p}_2) \cdot f(\mathrm{p}_2 \rightarrow \mathrm{p}_1 \rightarrow \mathrm{p}_0)\cdot G(\mathrm{p}_2\leftrightarrow \mathrm{p}_1)\space dA(\mathrm{p}_3)dA(\mathrm{p}_2)\\
&+\cdots
\end{align}
\label{unfold-LTE} \tag{7}
$$
$\eqref{unfold-LTE}$ 式右侧每一项都是光路长度的递增，例如第三项对应四顶点路径，如下图，有三段光路连接的4个顶点。其中，$\mathrm{p}_0\mathrm{p}_1$ 是camera ray与场景交点的光路，但 $\mathrm{p}_2,\mathrm{p}_3$ 可以是场景表面上的任一点。对这样的 $\mathrm{p}_2,\mathrm{p}_3$ 的积分，可以得到所有到达相机长度为4的光路的贡献总和。

<img src="/images/Render Engine/PBRT/13 Light Transport.assets/pha13f03.png" alt="img" style="zoom: 50%;" />

### 1.3.2 LTE 无限求和形式

$\eqref{unfold-LTE}$ 式的无限求和可以简洁记为
$$
L(\mathrm{p}_1\rightarrow \mathrm{p}_0)=\sum_{n=1}^\infty P(\bar{\mathrm{p}}_n) \label{LTE-infinte-sum-compact} \tag{8}
$$
其中，$P(\bar{\mathrm{p}}_n)$ 表示沿着具有 $n+1$ 顶点的光路 $\bar{\mathrm{p}}_n$ 散射的radiance量，$\bar{\mathrm{p}}_n=\mathrm{p}_0,\mathrm{p}_1,\cdots,\mathrm{p}_n$，而 $\mathrm{p}_0$ 表示成像平面上的点，$\mathrm{p}_n$ 表示光源上的点。无限求和的具体形式如下，
$$
P(\bar{\mathrm{p}}_n)=\underbrace{\int_A\int_A\cdots\int_A}_{n-1} L_e(\mathrm{p}_n\rightarrow \mathrm{p}_{n-1})\cdot \left(\prod_{i=1}^{n-1}f(\mathrm{p}_{i+1}\rightarrow \mathrm{p}_i\rightarrow \mathrm{p}_{i-1})G(\mathrm{p}_{i+1}\leftrightarrow \mathrm{p}_i)\right)dA(\mathrm{p}_2)\cdots dA(\mathrm{p}_n) \label{LTE-infinte-sum} \tag{9}
$$

### 1.3.3 光路的 throughput

在整个光路中，BSDF与几何项的乘积称为路径的通量 (throughput)，它描述了光源radiance经过路径顶点间所有散射后到达相机的比例，可以表示：
$$
T(\bar{\mathrm{p}}_n)=\prod_{i=1}^{n-1}f(\mathrm{p}_{i+1}\rightarrow \mathrm{p}_i\rightarrow \mathrm{p}_{i-1})G(\mathrm{p}_{i+1}\leftrightarrow \mathrm{p}_i) \label{throughput}\tag{10}
$$
代入到 $\eqref{LTE-infinte-sum}$ 中，有
$$
P(\bar{\mathrm{p}}_n)=\underbrace{\int_A\int_A\cdots\int_A}_{n-1} L_e(\mathrm{p}_n\rightarrow \mathrm{p}_{n-1})\cdot T(\bar{\mathrm{p}}_n)\space dA(\mathrm{p}_2)\cdots dA(\mathrm{p}_n)
$$

### 1.3.4 蒙特卡洛估计

给定光路长度 $n$ 以及LTE无限求和形式 $\eqref{LTE-infinte-sun-compact}$ ，光路上顶点 $\bar{\mathrm{p}}_n \sim p$，到达 $\mathrm{p}_0$ 的radiance 有如下蒙特卡洛估计
$$
L(\mathrm{p}_1\rightarrow \mathrm{p}_0) \approx \sum_{n=1}^\infty \frac{P(\bar{\mathrm{p}}_n)}{p(\bar{\mathrm{p}}_n)} \label{LTE-sum-estimator}
$$
无论路径生成是何种方式，如从光源出发、两端同时构建还是从中间点开始，仅会影响路径概率密度 $p(\bar{\mathrm{p}}_n)$ 的计算。后续章节中将展示，这一数学框架如何演化为实际的光传输算法，如路径追踪、双向路径追踪、Metropolis光传输。

## 1.4 积分过程的处理

### 1.4.1 Delta 分布

在实际中，只有采样到光源上，$L_e$ 项才有效，而其它的采样点都属于无效采样。这种情况属于 delta 分布，在 0 处非零，其它都为0。Delta函数可能出现在光传输项中，例如特定类型的光源（点光源、方向光等），也可能来自有delta分布描述的BSDF组件。如果存在此类分布，随机采样表面点得到的出射方向难以命中光源，因此光传输算法必须显式处理，即选择从着色点到光源的方向。

例如，单个点光源的场景，直接光照项使用delta分布描述如下：
$$
\begin{align}
P(\bar{\mathrm{p}}_2)&=\int_A l_e(\mathrm{p_2} \rightarrow \mathrm{p_1})f(\mathrm{p_2}\rightarrow \mathrm{p_1}\rightarrow \mathrm{p_0})G(\mathrm{p_2}\leftrightarrow \mathrm{p_1})\space dA(\mathrm{p}_2) \\
&=\frac{\delta(\mathrm{p}_{light}-\mathrm{p_2})L_e(\mathrm{p}_{light}\rightarrow \mathrm{p}_1)}{p(\mathrm{p}_{light})}f(\mathrm{p_2}\rightarrow \mathrm{p_1}\rightarrow \mathrm{p_0})G(\mathrm{p_2}\leftrightarrow \mathrm{p_1})
\end{align}
$$
也就是说，$\mathrm{p}_2$ 必须是场景中光源的位置，其余采样位置都无需参与积分，从而消除了表面积分形式而变为单个蒙特卡洛积分采样项。类似地，路径通量 (throughput) 中包含delta分布的BSDF项也会消除对应的表面积分。

### 1.4.2 划分被积函数

为了开发能够无遗漏、无重复地计算所有散射模式的正确光传输算法，必须明确算法覆盖了LTE的哪些部分。一种有效方法是对LTE进行不同形式的划分。

#### 1.4.2.1 基于光路长度的划分

例如将对路径的求和展开，划分为直接光照路径与间接光照路径：
$$
L(\mathrm{p}_1\rightarrow \mathrm{p}_0)=P(\bar{\mathrm{p}}_1)+P(\bar{\mathrm{p}}_2)+\sum_{i=3}^{\infty}P(\bar{\mathrm{p}}_i)
$$
其中，第一项 $P(\bar{\mathrm{p}}_1)$ 计算自发射radiance；第二项 $P(\bar{\mathrm{p}}_2)$ 计算直接光照，即 $\mathrm{p}_2$ 自发光经由 $\mathrm{p}_1$ 到达 $\mathrm{p}_0$；剩余的求和项则可以使用更快但不太精确的方法计算。这样划分三项都可以独立求解。

> 再次重申，$\mathrm{p}_1$ 视为直接可见着色点，$\mathrm{p}_0$ 视为相机。第一项就是着色点的自发光，第二项是直接光照，第三项是之后的间接光。

#### 1.4.2.2 自发光项的划分

对单个积分项进行划分也很有用，例如可以将自发光项拆分为小光源 $L_{e,s}$、大光源 $L_{e,l}$，得到两个独立的积分估计：
$$
\begin{align}
P(\bar{\mathrm{p}}_n)&=\int_{A^{n-1}} \Big( L_{e,s}(\mathrm{p}_n\rightarrow \mathrm{p}_{n-1}) + L_{e,l}(\mathrm{p}_n\rightarrow \mathrm{p}_{n-1}) \Big)\cdot T(\bar{\mathrm{p}}_n)\space dA(\mathrm{p}_2)\cdots dA(\mathrm{p}_n)\\
&=\int_{A^{n-1}} L_{e,s}(\mathrm{p}_n\rightarrow \mathrm{p}_{n-1})\cdot T(\bar{\mathrm{p}}_n)\space dA(\mathrm{p}_2)\cdots dA(\mathrm{p}_n) 
+\int_{A^{n-1}} L_{e,l}(\mathrm{p}_n\rightarrow \mathrm{p}_{n-1})\cdot T(\bar{\mathrm{p}}_n)\space dA(\mathrm{p}_2)\cdots dA(\mathrm{p}_n)
\end{align}
$$
这两个积分可以独立计算，甚至采用完全不同的算法或采样数。只要估计 $L_{e,s}$ 时忽略掉来自 $L_{e,l}$ 的自发光，反之亦然；以及所有光源要被划分为小光源，要么为大光源；结果就是正确的。

#### 1.4.2.3 BSDF项的划分

BSDF 项同样可以划分。例如 $f_{\Delta}$ 表示BSDF中由delta分布描述的组件，以及 $f_{\neg\Delta}$ 表示BSDF的其它组件，那么可以拆分为
$$
\begin{align}
P(\bar{\mathrm{p}}_n)&=\int_{A^{n-1}} L_e(\mathrm{p}_n\rightarrow \mathrm{p}_{n-1})\cdot \left(\prod_{i=1}^{n-1}\big(f_{\Delta}(\mathrm{p}_{i+1}\rightarrow \mathrm{p}_i\rightarrow \mathrm{p}_{i-1})+f_{\neg\Delta}(\mathrm{p}_{i+1}\rightarrow \mathrm{p}_i\rightarrow \mathrm{p}_{i-1})\big)\cdot G(\mathrm{p}_{i+1}\leftrightarrow \mathrm{p}_i)\right)dA(\mathrm{p}_2)\cdots dA(\mathrm{p}_n)\\
\end{align}
$$
注意，路径通量的乘积中有 $n-1$ 个 BSDF 项，要将所有BSDF的delta与非delta组件包含进来。最终，积分内包含delta组件的都可以退化为单项蒙特卡洛估计。

# 2 Path Tracing

路径追踪是图形学中首个通用的无偏蒙特卡洛光传输算法，从相机出发逐步生成散射事件路径，直至场景的光源。

基于LTE path integral形式，可以估计由 $\mathrm{p}_0$ 发出的 camera ray，与场景最近交点 $\mathrm{p}_1$ 的出射radiance为：
$$
L(\mathrm{p}_1\rightarrow \mathrm{p}_0)=\sum_{i=1}^{\infty}P(\bar{\mathrm{p}}_i)
$$
对此有两个问题需要解决：

1. 无限项求和：如何以有限计算量来估计无穷项之和？
2. 路径生成与积分估计：如何生成光路的路径以计算多维积分的蒙特卡洛估计？

## 2.1 Roulette 截断

对于路径追踪，利用物理场景下路径贡献随顶点数衰减的特性（更长路径的整体贡献更低，特定路径例外）。基于此，可采取策略：

- 前几项精确估计
- 后续项采用俄罗斯轮盘赌 (Russian Roulette) 截断，以概率终止采样，避免偏差。

Russian Roulette 是一种通过跳过对最终结果贡献较小的样本评估来提高蒙特卡洛估计效率的技术。

- 首先选择一个终止概率 $p_{rr}$
- 以概率 $p_{rr}$ 跳过对当前样本的评估
- 以概率 $1-p_{rr}$ 正常评估估计量，并叠加权重 $\frac{1}{1-p_{rr}}$，以补偿跳过的样本

假设 $L(X)$ 作为样本估计量，蒙特卡洛积分估计量通常有如下形式
$$
L(X)=\frac{f(X)\cdot v(X)}{p(X)}
$$
在应用 RR 后，估计量为
$$
L_{rr}=
\begin{cases}
\frac{L(X)}{1-p_{rr}} \quad \epsilon > p_{rr} \\
0 \quad\quad \mathrm{otherwise}
\end{cases}
$$

由离散概率分布有新估计量的期望为
$$
E\left[L_{rr}\right]=(1-p_{rr})\cdot \frac{L(X)}{1-p_{rr}} + p_{rr}\cdot 0=L(X)
$$
由此可以看出应用俄罗斯轮盘赌不会改变估计量的无偏性，但也不会降低方差，而合理选择终止概率可以提升蒙特卡洛效率。每段光路，即path integral求和中的每一项都可以独立设置终止概率。

## 2.2 Path Sampling

在有了使用有限项来估计无限项之和的方法后，我们还需要评估特定项 $P(\bar{\mathrm{p}}_i)$ 的方法。$\bar{\mathrm{p}}_i$ 表示具有 $i+1$ 个顶点的路径，末顶点 $\mathrm{p}_i$ 位于光源，而 $\mathrm{p}_0$ 位于相机。

### 2.2.1 表面积均匀采样

采样顶点 $\mathrm{p}_i$ 最自然想到的是基于表面积的均匀采样。假设场景中 $n$ 个物体，每个具有表面积 $A_i$，那么采样到 $i$th 物体的概率为
$$
p_i = \frac{A_i}{\sum_iA_i}
$$
接下来，需要在 $i$th 物体表面均匀采样一点，那么pdf为 $\large \frac{1}{A_i}$。因此场景中采样一点 $\mathrm{p}_i$ 的pdf为
$$
p_A(\mathrm{p}_i)=\frac{A_i}{\sum_jA_j}\cdot \frac{1}{A_i}=\frac{1}{\sum_iA_i}
$$
也就是说，场景中每一点都具有相同的概率。

以此方式采样顶点集 $\mathrm{p}_0,\cdots,\mathrm{p}_{i-1}$，如果采样末顶点 $\mathrm{p}_i$ 采用同样的方式，会在没有采样到发光物体表面时，导致整个路径对最终结果无贡献。因此更好的策略是仅在发光物体表面采样末顶点。在给定完整路径后，即可逐项计算 $P(\bar{\mathrm{p}}_i)$。

虽然这种路径采样方法可以灵活设置采样概率，例如已知某几个物体的间接光主导场景照明，可为这些物体分配更高的采样概率。然而仍存在两个问题：

- 不可见顶点对：路径中存在两个不可见的相邻顶点，会导致整个路径无贡献，增大方差。
- delta函数失效：如果被积函数包含delta函数（如点光源、完美镜面BSDF），此采样无法选择delta分布非零的路径顶点，仍导致高方差。

### 2.2.2 逐步构建路径

一种同时解决上述两个问题的方法是从相机顶点 $\mathrm{p}_0$ 开始逐步构建路径，具体步骤：

1. 逐顶点采样方向：在每个顶点 $\mathrm{p}_i$ 处，根据 BSDF 采样新方向 $\omega_i$
2. 光线追踪确定下一顶点：从 $\mathrm{p}_i$ 沿 $\omega_i$ 反射光线，找到最近交点作为 $\mathrm{p}_{i+1}$

此方法通过逐级选择局部贡献显著的方向，尝试构建整体贡献较大的路径。尽管某些场景下可能失效，但通常是一种有效策略。由于此方法基于立体角采样BSDF，而路径积分 LTE 是表面积积分，概率密度需要执行如下转换：
$$
p_A(\mathrm{p}_i)=p_{\omega_{i-1}}\frac{|\cos\theta_i|}{||\mathrm{p}_{i-1}-\mathrm{p}_i||^2}
$$
路径末顶点 $\mathrm{p}_i$ 位于光源表面，采用光源表面积采样，称为 next event estimation (NEE)。

假设自发光物体的采样分布 $p_e$，一个路径的蒙特卡洛估计为
$$
P(\bar{\mathrm{p}}_i)=\frac{L_e(\mathrm{p}_i\rightarrow \mathrm{p}_{i-1})\cdot f(\mathrm{p}_{i}\rightarrow \mathrm{p}_{i-1}\rightarrow \mathrm{p}_{i-2})}{p_e(\mathrm{p}_i)}\cdot \left(\prod_{j=1}^{i-2}\frac{f(\mathrm{p}_{j+1}\rightarrow \mathrm{p}_j\rightarrow \mathrm{p}_{j-1})\cos\theta_j}{p_{\omega(\omega_j)}}\right)
$$

> 将末顶点改为光源表面积采样，将之前顶点改为基于立体角的采样

这种采样模式在构建第 $i$ 段光路时，复用了长度 $i-1$ 的路径。

# 3 Bidirectional Method

前面的描述均基于从相机出发构建路径，仅在路径末顶点连接光源。本章介绍双向路径算法，即从相机与光源同时采样路径，并在中间顶点连接。此类算法在复杂光照场景中能显著提升光路构建效率。

## 3.1 测量方程

测量方程描述了基于对光线radiance集合积分的抽象测量值的计算。例如，计算图像中像素 $j$ 时，我们想要以一定权重积分周围像素的光线，而权重由image reconstruction filter决定。忽略景深（即胶片平面每点对应单一相机出射方向），像素值可表示为胶片平面的权重函数与沿着camera ray的入射radiance乘积的积分：
$$
\begin{align}
I_j&=\int_{A_{film}}\int_{S^2}W_e^{(j)}(\mathrm{p}_{film},\omega)L_i(\mathrm{p}_{film},\omega)|\cos\theta|\space d\omega dA(\mathrm{p}_{film})\\
&=\int_{A_{film}}\int_{S^2}W_e^{(j)}(\mathrm{p}_0\rightarrow \mathrm{p}_1)L(\mathrm{p}_1\rightarrow \mathrm{p}_0)G(\mathrm{p}_0\leftrightarrow \mathrm{p}_1)\space dA(\mathrm{p}_1)dA(\mathrm{p}_0)
\end{align}
$$
其中，$I_j$ 是对 $j$th 像素的测量，$\mathrm{p}_0$ 是胶片平面上一点；$W_e^{(j)}(\mathrm{p}_0\rightarrow \mathrm{p}_1)$ 是该像素的filter函数 $f_j$ 与选择合适的camera ray方向的delta函数的乘积：
$$
W_e^{(j)}(\mathrm{p}_0\rightarrow \mathrm{p}_1)=f_j(\mathrm{p}_0)\cdot \delta\Big(t\big(\mathrm{p}_0,\omega_{camera}(\mathrm{p}_1)\big)-\mathrm{p}_1\Big)
$$

> $t$ 是光线投射函数，计算最近交点；$f_j$ 描述的是目标像素的邻居对其filter权重；

将 $I_j$ 中LTE求和项 $P(\bar{\mathrm{p}}_n)$ $\eqref{LTE-infinte-sum}$展开，有
$$
\begin{align}
I_j&=\int_{A_{film}}\int_{S^2}W_e^{(j)}(\mathrm{p}_0\rightarrow \mathrm{p}_1)L(\mathrm{p}_1\rightarrow \mathrm{p}_0)G(\mathrm{p}_0\leftrightarrow \mathrm{p}_1)\space dA(\mathrm{p}_1)dA(\mathrm{p}_0)\\
&=\sum_i \underbrace{\int_A\cdots\int_A}_{i+1} W_e^{(j)}(\mathrm{p}_0\rightarrow \mathrm{p}_1)\cdot T(\bar{\mathrm{p}}_i) L_e(\mathrm{p}_{i+1}\rightarrow \mathrm{p}_i)G(\mathrm{p}_0\leftrightarrow \mathrm{p}_1)\space dA(\mathrm{p}_{i+1})\cdots dA(\mathrm{p}_0)
\end{align}
$$
注意上述方程中的$L_e$ 与 $W_e^{(j)}$ 具有优美对称性：光源发射辐射亮度 $L_e$，量化光源发射特性；相机权重函数 $W_e^{(j)}$，量化像素 $j$ 的传感器灵敏度特性。二者在形式上地位完全平等，emission与measurement的概念在数学上是可互换的。这种对称性表明渲染过程可被两种视角理解：

- 传统视角：光从光源发射，经场景散射后抵达传感器，$W_e^{(j)}$ 描述其对测量的贡献
- 逆向视角：传感器发射一种虚拟量，当其到达光源时形成测量。

通过交换光源与相机的角色，可以得到 particle tracing 的方法：从光源发射射线，递归估计表面接收的入射重要性。虽然该方法本身并非高效渲染技术（大多无法进入相机），但它构成双向路径追踪与光子映射等算法的核心要素。

方程中 $W_e^{(j)}$ 项描述了场景中 $\mathrm{p}_0$ 到 $\mathrm{p}_1$ 光线的重要性。当测量方程用于计算像素值时，重要性常用 delta分布部分或完全描述，如前例中的相机光线选择。除图像生成外，其它类型的测量（如辐射场预计算）也可适当构建重要性函数描述。

### 3.1.1 Sampling Cameras

相机重要性函数计算从相机出发的光线对像素测量的贡献权重。

### 3.1.2 Sampling Light Rays

## 3.2 光子映射

https://zhuanlan.zhihu.com/p/208356944

### 3.2.1 Particle Tracing

particle tracing算法从光源出发，进行追踪，与表面相交时在交点放置能量，然后再进行反射。与之相对的是path tracing，从相机出发，为每个像素采样光线样本。particle tracing算法生成一批样本，表示一点 $p_j$ 收到来自方向 $\omega_j$ 的throughput weight $\beta_j$ 如下所示
$$
(p_j, \omega_j, \beta_j)
$$

> $\beta_j$ 中累积了 particle 路径中throughput与重要性采样pdf的比值，以及经过路径调制后的光源radiance。

基于tracing阶段得到的particle，我们可以计算任意测量的估计。测量量使用重要性函数 $W_e(p,\omega)$ 来描述，$W_e$应该满足测量量的定义与估计量之间的无偏，
$$
E\left[\frac{1}{N}\sum_{j=1}^N\beta_jW_e(p_k,\omega_j)\right]=\int_A\int_{S^2}W_e(p,\omega)L_i(p,\omega)|\cos\theta|\space \mathrm{d}\omega \mathrm{d}A \label{importance-func} \tag{3-1}
$$

其实就是将半球面上的积分再嵌套一层表面上点的积分，最外层嵌套 ($\mathrm{d}A$) 就是为了对所有粒子上的测量，而内层 ($\mathrm{d}\omega$) 是在着色点半球面内的测量，可以理解为粒子框架下的渲染方程。

#### 3.2.1.1 例子：测量墙的入射通量

以测量一面墙入射的通量为例，该通量定义如下
$$
\Phi = \int_{A_{wall}}\int_{H^2(\mathbf{n})}L_i(p,\omega)|\cos\theta|\space \mathrm{d}A\mathrm{d}\omega
$$
对于该测量可以定义如下重要性函数，用来选取位于墙上并且来自normal方向定义的半球区域的入射部分：
$$
W_e(p,\omega)=\begin{cases}\begin{align}
& 1 \quad \quad \text{p is on wall and } (\omega\cdot \mathbf{n}) > 0\\
& 0 \quad \quad otherwise
\end{align}\end{cases}
$$

如果 $\eqref{importance-func}$ 式满足，那么上述测量量就可以使用粒子加权和形式来估计。如果想要测量另一面墙 (或者墙的一部分) 的通量，我们可以重新定义 $W_e$，只需要重新计算粒子的加权和即可，粒子可以重复利用。

### 3.2.2 Photon Mapping

为了计算 $p$ 点在 $\omega_o$ 方向上反射的radiance，可以等价地表示为对场景表面的所有点的测量，其中Dirac delta分布仅选择精确在 $p$ 点处的粒子，如下式
$$
\int_{S^2}L_i(p,\omega_i)f(p,\omega_o,\omega_i)|\cos\theta_i|\space\mathrm{d}\omega_i = \int_A\int_{S^2}\delta(p-p')L_i(p',\omega_i)f(p',\omega_o,\omega_i)|\cos\theta_i|\space\mathrm{d}\omega_i\mathrm{d}A(p') \label{exit-radiance} \tag{3-2}
$$
左边是 $p$ 点处的渲染方程，右边是一种转为对表面点积分的等价表示。按照 $\eqref{importance-func}$ 式，可以得到描述测量量的重要性函数
$$
W_e(p',\omega)=\delta(p'-p)f(p,\omega_o,\omega_i) \label{pm-importance-func} \tag{3}
$$
光子映射引入的有偏近似：基于附近点的光照信息可以看作着色点的照明的合理近似的假设，提出在着色点周围的光子中进行插值来提供着色点接收的光照信息。因此，$\eqref{pm-importance-func}$ 式的delta分布可以替换为一种filter函数。

#### 3.2.2.1 光子映射算法

光子映射是一种particle tracing算法，其过程是2-pass算法：

1. light pass：构建一张光子图，存储从光源发射的所有光子的通量信息。光源发出光子，光子与表面相交的信息存储在kd-tree结构上（光子贴图）。存储的信息用于后面的着色，包括光子位置、方向以及throughput。
2. camera pass：由相机出发的光路，执行传统的path tracing。追踪到漫反射表面时，查找光路顶点在光子贴图中其周围的光子信息，进行加权着色。

光子映射的优点：光子可以被重用以及开销容易被分摊；着色点使用附近的光子提供光照，更容易解决无偏算法 (例如path tracing、BDPT等) 中无法基于增量路径构建的路径采样问题。例如如下场景，针孔相机与场景之间有一块能折射光线的玻璃板，以及场景中的点光源。没有能够采样到一个能够到达点光源的入射方向的采样策略。但光子映射可以通过追踪由光源出发后在diffuse表面沉淀的光子，着色点附近的光子可以用来很好地估算照明。

> 传统的path tracing选择与点光源直接相连，但中间光路会由于折射被改变，会导致这样采样得到的方向并不能到达光源。还要在求交后再进行调整，调整后再求交。

<img src="/images/Render Engine/PBRT/13 Light Transport.assets/image-20231217201735872.png" alt="image-20231217201735872" style="zoom:50%;" />

##### （1）Photon Map 的构建

光子图的构建规则就是：从光源发射光子，并让光子在场景中反复弹射，每次击中漫反射表面都进行一次光子的纪录，直到被某个漫反射表面彻底吸收掉为止。

<img src="/images/Render Engine/PBRT/13 Light Transport.assets/v2-3f4ae18cdefa2d6fbdb8a7b6edcde9f0_r.jpg" alt="img" style="zoom: 33%;" />

算法流程如下：

```c++
for (i = 0; i < nEmittedPhotons; i++)
{
    <基于光源采样，确定射线ray和初始功率power>
    while (true)
    {
        <ray在场景中迭代，找到交点>
        <生成交点处的反射分布BSDF>
        <基于该BSDF计算反射分布，并顺便记录反射方向nextDir+反射类型refType>
        <如果refType是漫反射，就记录到光子贴图>
        <使用俄罗斯轮盘赌，判断光子是否被吸收>
        <如果未被吸收，更新power（也就是下一次迭代的功率值）>
        <使用nextDir更新下一次迭代的新方向ray>
    }
}

<为光子贴图构建平衡kd树>
```

每个光子存储

```c++
struct Photon
{
    Vector3 position;
    Vector3 direction;
    Vector3 power;
};
```

##### （2）采样 Photon Map

```c++
<从相机发射射线ray>
while (depth < maxDepth)
{
    <寻找ray的最近交点>
    <如果射线击中光源，计算自发光radiance Le>
    <基于交点处材质，生成BSDF>
    <使用BSDF计算反射分布，并顺便记录反射方向nextDir+反射类型refType>
    if <refType是漫反射>
    {
        IsDiffuse = true;
        break;
    }
    else <refType是光滑反射/完美镜面反射/折射>
        <计算下一次迭代的吞吐量>
}

if (IsDiffuse)
{
    <使用kd树+kNN，寻找最近的N个光子>
    <使用Pass_2中的公式，基于所有邻近点对N个光子进行密度估计，实现光子映射>
    <计算最终结果并结束>
}
```



#### 3.2.2.2 Density Estimation

使用着色点附近的光子进行照明的一个统计方法density estimation：即先假设这些样本来自同一个总体分布，构造该组样本点的概率密度函数(PDF)。比如直方图这一简单例子，1D情况下可以划分等宽区间，然后计算每个区间的样本数量，最后进行归一化得到概率密度。

**kernel method**

kernel method可以得到更加平滑的PDF，给定核函数(kernel function) $k(x)$，有 $\int_{-\infin}^{+\infin} k(x)\mathrm{d}x = 1$。对于 $N$ 个样本，在$x_i$ 处的的核估计量(kernel estimator)为
$$
\hat{p}(x)=\frac{1}{Nh}\sum_{i=1}^N k\left(\frac{x-x_i}{h}\right) \label{kernel-estimator} \tag{4}
$$
其中，$h$ 是窗口大小(window width)（或者平滑参数、kernel bandwidth）。给定 $N$ 个样本，核估计就是在每个样本上应用核估计函数，然后将这些核函数值加起来形成整体的密度估计。下图就是1D下应用 Epanechnikov kernel 的示例，核函数为
$$
k(t) = 
\begin{cases}
0.75(1-2t^2)/\sqrt{5}, \quad t < \sqrt{5} \\ 
0,\quad  otherwise
\end{cases}
$$
图中每条圆弧形虚曲线都对应在某一点处的核估计，所有核估计形成实线形状的整体密度估计

<img src="/images/Render Engine/PBRT/13 Light Transport.assets/image-20231224143050504.png" alt="image-20231224143050504" style="zoom: 33%;" />

#### 3.2.2.3 nth nearest neighbor estimate

kernel method的关键问题是 $h$ 的选取，太宽会模糊具有许多样本的区域的相关细节；太窄会导致PDF在尾部的分布由于样本不够而凹凸不平。Nearest-neighbor 基于局部样本密度来自适应选择窗口大小，附近样本越多，窗口越小；反之，窗口越大。

nth nearest neighbor estimate 选择 $Nth$ nearest 样本到估计点 $x$ 的距离 $d_N(x)$ 作为窗口大小，代入到 $\eqref{kernel-estimator}$ 有
$$
\hat{p}(x)=\frac{1}{Nd_N(x)}\sum_{i=1}^Nk\left(\frac{x-x_i}{d_N(x)}\right)
$$
扩展到 $d$ 维，有
$$
\hat{p}(x)=\frac{1}{N(d_N(x))^d}\sum_{i=1}^Nk\left(\frac{x-x_i}{d_N(x)}\right) \label{density-estimate} \tag{5}
$$
将密度估计代入到 $p$ 点在出射方向 $\omega_o$ 上的出射radiance测量 $\eqref{exit-radiance}$ ，能够得到以下估计量
$$
L_o(p, \omega_o) \approx \frac{1}{N_p(d_N(p))^2}\sum_{j=1}^{N_p}k\left(\frac{p-p_j}{d_N(p)}\right)\beta_jf(p,\omega_o,\omega_j) \label{exit-radiance-pm} \tag{6}
$$
其中 $N_p$ 表示光子数量，重要性函数使用密度估计方程代替，同时对于超出 N nearest距离的采样点，核函数为0。这个估计过程是先在 $p$ 点的 tangent plane 上采样一点 $p_j$ (即内层积分的 $p'$)，因此这里的密度估计是2维。如前所述，$\beta_j$ 是 $p$ 在入射方向 $\omega_j$ 收到的 throughput与采样概率密度（渲染方程积分的采样）的比值。注意观察与标准渲染方程积分估计的区别：

- 渲染方程积分采样点永远是着色点，而这里近似为了着色点附近区域以某一密度估计下的采样点。这是一种着色点邻近区域的插值过程。
- 如果将着色点附近区域内的采样改为（着色点处为1，其余为0）的分布，上式恰好可以退化为渲染方程

$\eqref{exit-radiance-pm}$​ 在光子上的测量是一种插值过程，该过程引入的偏差难以估计，但通过提高光子密度往往可以得到更好的结果。如果直接光照使用传统方式，而对于更加低频的间接光通常问题不大。

>  $\eqref{exit-radiance-pm}$ 式过程的理解，$\eqref{exit-radiance-pm}$ 是对以下积分的近似：
>  $$
>  \int_{S^2}L_i(p,\omega_i)f(p,\omega_o,\omega_i)|\cos\theta_i|\space\mathrm{d}\omega_i = \int_A\int_{S^2}\hat{p}(p')L_i(p',\omega_i)f(p',\omega_o,\omega_i)|\cos\theta_i|\space\mathrm{d}\omega_i\mathrm{d}A(p')
>  $$
>  particle tracing与path tracing的不同，导致二者在求渲染方程积分上不同
>
>  - path tracing：从着色点出发，采样不同的光线，对光线交点着色得到光线方向上的入射radiance。光线相交于表面后，再采样下一级反射方向，递归执行。
>  - particle tracing：从光源出发，
>
>  每个光子携带了其从光源传播到当前位置整条光路累积后的 $f \cdot radiance / pdf$。光子映射本质上是采样着色点周围的光子，作为着色点的近似，是一种filter过程。因此渲染方程则近似为了对光子的加权平均，权重与光子的分布有关
>
>  从上式积分形式看，外层是对表面上（着色点附近）光子的积分，而内层积分则是对于光子 $p'$ 的 $\beta$ 积分。

[3.2.2.1 光子映射算法](#3.2.2.1 光子映射算法) 小节描述的最初形式的光子映射算法先进行光子生成，存放到光子贴图上；再进行测量过程。这会导致光子数量受存储空间限制，当存储耗尽时，质量就无法再进一步提高，限制了算法上限。

### 3.2.3 Progressive Photon Mapping (PPM)

PPM算法重构了最初形式的光子映射，避免光子存储带来的限制，同样也是2-pass算法：

- camera pass：追踪从相机出发的光路，直至第一个diffuse表面，记录下来交点(visible point)几何信息以及漫反射，同时累积光路经过的specular表面的BSDF。
- light pass：从光源发出光子光路(多级)，每一级光子与表面的相交时，都会向其周围的visible points贡献radiance估计

这种方法不需要存储光子，只需要存储camera pass得到的visible point，这部分要远小于记录场景的光子贴图。但对于高分辨率以及需要高SPP的情况下，存储开销依然较高。

### 3.2.4 Stochastic Progressive Photon Mapping (SPPM)

SPPM对PPM进一步改进，做到不受内存限制的影响。SSPM依然是从camera pass出发，但其限制了每像素的采样数（可以低至1SPP），即visible points的存储固定；之后light pass同样是从光源出发，使用光子对周围visible points贡献radiance。不同的是，SPPM将这个过程迭代执行，使用同样的visible points存储达到高SPP的结果。

SPPM 同样基于 $\eqref{exit-radiance-pm}$ 光子估计方程出发，但存在两个调整。首先它使用常数核函数，估计过程发生在visible point的tangent平面，$p$ 点在 $\omega_o$ 方向的出射radiance估计量如下，
$$
L_o(p,\omega_o)\approx \frac{1}{N_p\pi r^2}\sum_j^{N_p}\beta_jf(p,\omega_o,\omega_j) \label{constant-estimate} \tag{3-6}
$$
其中，$N_p$ 是光源发出的光子数量；$\pi r^2$ 是盘形 (disk-shaped) 核函数的表面积。

第二个调整是每次迭代逐步减小光子的搜索半径。这种做法的总体想法是，随着搜索半径内发现的光子数量增多，会有更多证据表明足够的光子密度可以很好估计入射分布；通过减小半径，可以更偏向于细节保留。而当减小半径时，意味着累积的光子来自不同的半径范围，那么同样需要对此进行调整。下面的更新规则描述了如何更新半径以及依赖的变量：
$$
\begin{align}
N_{i+1} &= N_i + \gamma M_i \\
r_{i+1} &= r_i\sqrt{\frac{N_{i+1}}{N_i+M_i}} \\
\tau_{i+1} &= (\tau_i + \Theta_i)\frac{r^2_{i+1}}{r^2_i}
\end{align}
$$
其中，$N_i$ 是 i-th 迭代之前的贡献给着色点的光子总数；$M_i$ 是当前迭代贡献的光子数量；$r_i$ 是i-th迭代的搜索半径；$\gamma$ 是用于调整收缩速度的超参，通常选择 2/3；$\tau$ 是BSDF的累积，应该是指当前着色点累积收到的光子通量；$\Theta_i$ 在i-th迭代的计算如下，即当前迭代光子的throughput之和，$\beta_j$ 是光子当前携带的功率，即throughput缩放的radiance。
$$
\Theta_i = \sum_j^{M_i}\beta_jf(p,\omega_o,\omega_j)
$$

## 3.3 BDPT

双向路径追踪算法的核心路径采样策略：从光源生成一条子路径，从视点生成第二条子路径，然后将它们连接起来。通过调整两端生成的顶点数量，我们获得适用于所有路径长度的一系列采样技术。每种采样技术在路径空间具有不同的概率分布，并考虑被积函数（即测量贡献函数）的不同因子子集。最后使用多重重要性采样将所有采样技术产生的样本进行组合。

https://zhuanlan.zhihu.com/p/384408596

## 3.4 Metropolis Light Transport

https://zhuanlan.zhihu.com/p/72673165

https://zhuanlan.zhihu.com/p/497375112

### 3.4.1 Metropolis 采样



