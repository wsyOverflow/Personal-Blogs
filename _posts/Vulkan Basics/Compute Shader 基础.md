---
typora-root-url: ..\..
title: Compute Shader 基础
categories:
- [Vulkan Basics]
---
# 基础概念

## 1. GPU 中的概念

### 1.1 处理器核心的层级概念

GPU 具有非常多的处理器核心，在 Nvidia 中称为 Streaming Multiprocessor(SM)，AMD 称为 Compute Unit(CU)，以下使用 SM。在 SM 内部包含最基本的处理单元 lane，lane 可以近似理解为一个线程。SM 进行调度执行任务时，最小单位并不是 lane，而是 Warp(Nvidia 中的称呼，AMD 称为 wavefront)。一个 Warp 包含一定数量的 lane，在 Nvidia 中是 32，AMD 是 64。不同的架构，一个 SM 中包含的 warp 数量不同，目前的 Nvidia GPU 中一个 SM 包含 4 个 warp，即 128 个 lane(shader core、cuda core等)。GPU 显卡型号不同，GPU 包含的 SM 数量也不同，3080Ti 包含 80 个 SM。

在同一个 Warp 中的所有 lane 执行相同的指令，但可以处理不同的数据，这样就构成了现代 GPU 的 SIMD 机制。但一个 Warp 中的所有 lane 并不总是同时处于运行状态，即 active lane；也并不总是同时处于非运行状态，即 inactive lane。例如：

- 下发的任务组包含的任务数量较少或者不是 warp 大小的整数倍，不足以填满整个 warp 的 lane，那么未分配任务的 lane 则处于 inactive 状态。下发的任务组在 vulkan 中称为  local workgroup(D3D 中的 thread group) 大小。

- Dynamic Branching 会导致 lane 的执行路径不止一条，即不止一套指令，这一定程度上破坏了 SIMD 机制。例如，代码中包含 if-else 语句，不满足 if 条件的 lane 由于 lockstep 运行方式需要等待其它所有满足 if 条件的 lane 执行 if 部分，等待中的 lane 则处于 inactive 状态。反之，满足 if 条件的 lane 也会等待不满足 if 条件的 lane 执行 else 部分。
  但 if-else 并不总是导致这种指令分歧，对于非常简单 if-else，GPU 能够使用同一套指令实现，例如如下代码

  ```glsl
  uint d = buff_array[...]; 
  uint c;
  if(d < some_value){
     c = 0;
  }else{
     c = 1; 
  }
  ```

  可能会被转换为下述伪代码描述的指令，注意 `cmp_and_choose`

  ```
  register_a = 0;
  register_b = 1; 
  register_d = load_from(buff_ptr_offset);
  register_some_value = ... ;
  register_c = cmp_and_choose(register_d, register_some_value, register_a, register_b);
  ```

因此，在执行之前，先将每个 local workgroup 中的线程划分为以 warp 为单位的子组，之后 GPU 以 warp 为单位进行调度执行。

### 1.2 存储类型与数据同步

以上描述的 SM、warp、lane 具有不同类型的存储，下面根据速度由快到慢依次说明，这里的参数数据以 3080Ti (Ampere-GA102架构)为例 [[1]](#[1])

- register：每个 SM 具有256KB 大小的寄存器区。

- shared memory/L1 cache：一个 SM 中的所有线程可以访问该 SM 具有有限大小的 shared memory。shared memory 本质上是一个较小的读写缓存区，GA102 架构中为 128KB，但其中只有一部分可被程序员使用，其他用作缓存或其他用途，例如 48KB 用作 shared memory。
- global memory(VRAM)：global memory 即 GPU 显存。如果寄存器存储或者shared memory 的使用大小溢出，则会将其中的数据写入显存中，这一点和 CPU 的高速缓存-内存机制类似。

这三种类型的存储除了存取速度差距大之外，还有工作机制上的区别：

- 首先，一个 SM 的 shared memory 对另一个 SM 上的线程是不可见的，即 shared memory 只用于同一 SM 的线程的数据同步与共享。
- 其次，shared memory 与 register 虽然都属于 SM，但只有 shared memory 中的数据可以在 SM 中的 lane 之间共享。register 一般用于暂存指令执行过程的输入输出，lane/线程执行中用到的 register 相当于是私有的，对于其他线程是不可见的。因此，一般情况下，同一 SM 中的线程共享数据需要额外花费 register 到缓存(shared memory) 的传输时间，以及占用了带宽。
- 最后，对于同一 SM 中正在执行的线程之间，GPU 提供了直接通过寄存器共享数据的机制。缓存与寄存器的存取速度相差很多，因此这会带来更高效的数据同步。各大图形 SDK 也对此实现了相应 API，如后续描述的 vulkan 中的 subgroup 概念。

## 2. Vulkan 中的概念

以上讲述了，一些硬件层面的概念，包括 SM(处理器核心)、Warp(处理器核心中同时执行的线程组)、Lane(相当于线程)。在图形 SDK 中，有相应的软件层面的概念。

- invocation：一个下发的并行任务，对应一个线程。

- local workgroup：在同一 SM 上执行的 invocation 组，面向一个 SM。声明在 compute shader 中，如

  ```glsl
  layout(local_size_x = X, local_size_y = Y, local_size_z = Z) in;
  ```

  上述声明表示，当前 compute shader 描述的 invocation 的 local workgroup 大小为 $X\times Y \times  Z$ 

- subgroup：local workgroup 中同时并行执行的 invocation 组，对应一个 Warp。如上述 Warp，subgroup 中的 invocation 同样有 active 和 inactive 两种状态，且具有相同的机制。

- global workgroup：一次下发并行任务的 API 调用生成的所有 invocation，面向整个 GPU。通过 `vkCmdDispatch` 指定 local workgroup 的维度与对应维度大小，如下
  ```c++
  void vkCmdDispatch(VkCommandBuffer cmdBuf, uint32_t groupCountX, uint32_t groupCountY, uint32_t groupCountZ);
  ```

  上述调用表面 global workgroup 中具有 (groupCountX x groupCountY x groupCountZ) 个 local workgroup，因此共有 (groupCountX x groupCountY x groupCountZ) x $X\times Y \times Z$ 个 invocation/线程。

Compute shader 的执行过程可以简要描述为：`vkCmdDispatch` 命令生成了一批 local workgroup，GPU 会将这批 local workgroup 划分到某些 SM 上执行，GPU 上的所有 SM 以并行形式执行。在 local workgroup 执行期间，一般不会离开其执行于的 SM。local workgroup 包含多个 invocation，GPU 的最小调度单位是 Warp，因此一个 local workgroup 会先被划分为一个或多个 subgroup(s)，每个 subgroup 的 invocation 可以并行执行。

global workgroup 中 local workgroup 的数量可以超过 GPU 中 SM 的数量，因此会有多个 local workgroup 被分配到同一个 SM 上执行。同一 SM 上的 subgroup 会在执行过程中发生切换，例如，在一个 subgroup 等待比较耗时内存访问操作完成时，为了隐藏延迟，会切换执行另一个 subgroup，这种切换也可能会发生在同一 SM 上的不同 local workgroup 的 subgroup 之间。假设 invocation 使用寄存器资源或 shared memory 过多，不足以满足切换的 subgroup，那么会有寄存器数据备份至缓存或 shared memory 备份至显存的操作，这样会带来性能上的影响。同样，local workgroup 的 subgroup 数量也可以超过一个 SM 中的 Warp 数量。

## 3. Compute Shader 中的内置变量

Compute Shader 没有 in/out 参数，只能使用 SSBO/Texture 资源进行读写。对于 Texture，compute shader 只能使用 image 类型，不支持采样且一个 image 类型参数只能使用一个 level。在访问 image 时，只能使用整数索引。

Compute Shader 经常用于处理高维数据，如贴图。有一些内置变量可以作为高维数据的索引，

- `uvec3 gl_NumWorkGroups`：传递给 dispatch 函数的 group 数量，即 (groupCountX, groupCountY, groupCountZ) 

- `uvec3 gl_WorkGroupID`：当前 invocation 所属的 local workgroup 的编号，范围 [0, gl_NumWorkGroups.XYZ)

- `uvec3 gl_LocalInvocationID`：当前 invocation 在其 local workgroup 内的编号，范围是
  [0, gl_WorkGroupSize.XYZ)，其中 gl_WorkGroupSize=(local_size_x, local_size_y, local_size_z)

- `uvec3 gl_GlobalInvocationID`：当前 invocation 在 global workgroup 中的编号，等于

  ```glsl
  gl_WorkGroupID * gl_WorkGroupSize + gl_LocalInvocationID
  ```

- `uint gl_LocalInvocationIndex`：相当于 gl_LocalInvocationID 的一维形式，等于

  ```glsl
  gl_LocalInvocationIndex =
            gl_LocalInvocationID.z * gl_WorkGroupSize.x * gl_WorkGroupSize.y +
            gl_LocalInvocationID.y * gl_WorkGroupSize.x + 
            gl_LocalInvocationID.x;
  ```

与 subgroup 相关的内置变量：

- `gl_NumSubgroups`：local workgroup 包含的 subgroup 的数量
- `gl_SubgroupID`：当前 subgroup 在 local workgroup 中的编号，范围是 [0, gl_NumSubgroups)
- `gl_SubgroupSize`：subgroup 的大小
- `gl_SubgroupInvocationID`：当前 invocation 在 subgroup 中 ID，范围是 [0, gl_SubgroupSize)
- `gl_SubgroupEqMask`, `gl_SubgroupGeMask`, `gl_SubgroupGtMask`, `gl_SubgroupLeMask`,  
  `gl_SubgroupLtMask`：subgroupBallot 相关

有关 gl_LocalInvocationID  与 gl_SubgroupInvocationID 的关系，没有找到有文档说明，但我相信应该与
gl_GlobalInvocationID、gl_LocalInvocationID之间的关系类似。因为 GPU 的最小调度单位是 subgroup/Warp 而不是 invocation/线程。在调度之前 local workgroup 就应该已经被划分为 subgroup。此时，subgroup 中的所有 invocation 都对于 GPU 而言是相同的指令序列。subgroup 内的分歧应该发生在实际执行过程中。因此，gl_LocalInvocationID 和 gl_SubgroupInvocationID 之间具有事先确定的关系应该不会对后续调度产生任何影响。但是如果没有确定的关系，那么subgroup 功能的使用将会受到很大的限制。经过在 RTX 3080ti 上测试，gl_LocalInvocationID  与
 gl_SubgroupInvocationID 有如下数值关系，

```glsl
gl_SubgroupID * gl_SubgroupSize + gl_SubgroupInvocationID == gl_LocalInvocationIndex 
```

或者

```glsl
gl_LocalInvocationIndex % gl_SubgroupSize == gl_SubgroupInvocationID 
```

### 内存模型术语

#### 1. Coherent or Incoherent Memory Access [[2]](#[2])

-  Coherent 内存访问

以最简单的例子——单处理器系统来说，对于同一内存区域同时只会有一个线程访问。因此，当一个处理元件对某一内存区域写，然后另一处理元件对同一内存区域读时，总能读到更新后的值。这是我们想要的结果，但得到这一正确结果并不只是多线程读写同步。因为，在硬件层有高速缓存-内存机制，处理器的读写往往不直接针对内存，而是针对缓存。缓存中的数据是内存中数据的拷贝或者是写操作得到的更新数据，因此需要确保写操作之后的读操作能够读到新数据，而非旧数据，这就是 **Coherent 内存访问**。单处理器系统的内存操作工作于同一套高速缓存-内存机制中，保证 Coherent 内存访问较容易。

- Incoherent 内存访问

但对于多处理器系统，每个处理器都有其本地缓存，当多个处理器同时访问同一内存区域时，同一内存区域的数据在这些处理器的缓存中都有一个备份。此时可能存在处理器读到的时本地缓存中的旧数据，而不是其他处理器写操作生成的新数据，即出现 **Incoherent 内存访问**。为了避免这种情况，多处理器系统的设计需要采用存储器一致性协议，粗略地说，当一个处理器对某一内存区域更新时，对该内存区域存在缓存备份的其他处理器也要对其本地缓存的相应位置进行更新。

#### 2. Visibility [[3]](#[3])

GPU 的核心数量远高于 CPU，在 GPU 的执行过程中，不同处理器同时读写同一内存区域时，也会出现上述 Incoherent 内存访问情况。而 **Visibility** 术语就是指一个 shader invocation 可以安全地读其他 shader invocation 的 Incoherent 写数据，也就是说，不会读取到缓存中旧的备份，或者其他 invocation 写数据对其是可见的。

对于多个 shader invocation 同时读写不同的区域，不会出现上述 incoherent 访问情况。此外，对于在同一处理器执行的 shader invocation，不会出现 incoherent 情况，因为使用的是同一套缓存-内存机制，例如 compute shader 中的同一 local workgroup 中的 invocation。特别注意，这里不会出现 incoherent 访问问题，并不是指读写同步问题，读写同步同样需要额外处理。当不同处理器上的 shader invocation 对同一内存区域进行读写时，则会出现 incoherent 内存访问。

[[5]](#[5]) 提到 incoherent 的内存访问操作有：

- 对 image 变量的 imageLoad/imageStore
- 对 buffer 变量的读写操作
- [Atomic Counters](https://www.khronos.org/opengl/wiki/Atomic_Counter)
- 对 compute shader 中的 shared 变量的读写操作。这个无法理解，shared 变量存储在缓存中，不应该会有多个备份的情况，为什么还是 incoherent？

这时需要考虑如何避免 incoherent 内存访问，即确保 visibility 性质。这里有两种情况：

##### 2.1 Internal Visibility：一个绘制指令执行内部，一部分写、而另一部分读。

- 读写次序控制、内存控制

想要使得一个 shader invocation 能够读取另一 shader invocation 写数据，要先保证写操作在读操作之前确实发生，也就是读写同步问题。例如 compute shader 中 `barrier` 函数，可以确保 local workgroup(执行在同一处理器) 中的所有 invocation 都执行到 barrier 同步点后，才开始执行之后的代码。

注意 barrier 函数只是对 local workgroup 中的 invocation 的执行过程进行了控制，这种控制只发生在同一个处理器上。对于 shared memory 缓存只会在同一个处理器上共享，因此 barrier 同时也能够做到 shared memory 的读写次序控制。在 compute shader 中 shared 变量即位于 shared memory 中，由于本身就是缓存，不会出现多个备份情况，因此 shared 变量是隐含的 coherent 访问。这一点在 [[6]](#[6]) 的 1.1.2 小节也有说明

> - Private GLSL issue #24: Clarify that **barrier**() by itself is enough to synchronize both control flow and memory accesses to **shared** variables and tessellation control output variables. For other memory accesses an additional memory barrier is still required.

- 内存访问控制 [[4]](#[4])

确保读写次序，相当于确保了在读操作之前，写操作一定已经发生。对于 coherent 内存访问而言，这已经确保了写数据的可见性；但对于 incoherent 内存访问，写操作发生不代表写数据对其他 invocation 可见，这会导致一个 invocation 的内存读写操作的相对次序对于另一个 invocation 而言是不确定的状态，换句话说，写操作的数据对其他 invocation 可见的次序不确定。例如一个 invocation 中执行两次写操作，而另一个 invocation 可能会先看到第二次写的数据，后看到第一次写的数据。

这时需要使用到 memory barrier 来控制读写操作，使得其他 invocation 看到写数据的次序与写操作执行的次序一致。先对可能被多个处理器访问到的 image 或 buffer 类型变量进行 `coherent` 修饰，声明该变量为 coherent 访问机制。该访问机制使得相应的 memoryBarrier 可以控制对被修饰变量的读写操作。调用 memoryBarrier 等函数的 invocation 会等待之前的所有读写操作完成，当该函数返回后，写数据对之后的访问处于可见状态。例如一个 invocation 执行两次写操作，每次写操作后都加上一个 memory barrier，那么其他 invocation 就不可能先看到第二次写的数据，而后看到第一次写的数据。memory barrier 有多种类型：

- `memoryBarrier`：控制所有类型变量的内存访问，render command 内作用于全局。
- `memoryBarrierAtomicCounter`：控制 atomic-counter 变量的访问，render command 内作用于全局。
- `memoryBarrierBuffer`：控制 buffer 变量的内存访问，render command 内作用于全局。
- `memoryBarrierImage`：控制 image 变量的内存访问，render command 内作用于全局。
- `memoryBarrierShared`：控制 shared 变量的内存访问，作用于同一 workgroup。
- `groupMemoryBarrier`：控制所有类型变量的内存访问，作用于同一 workgroup。

##### 2.2 External Visibility

一个 render command 内部的 visibility 是 shader invocation 之间的读写操作。对于 render command 之间的 visibility 使用 barrier 命令进行同步。例如 vulkan 中的 buffer barrier、image layout 等

# Reference

<a name="[1]">[1]</a> https://images.nvidia.com/aem-dam/en-zz/Solutions/geforce/ampere/pdf/NVIDIA-ampere-GA102-GPU-Architecture-Whitepaper-V1.pdf

<a name="[2]">[2]</a> https://en.wikipedia.org/wiki/Memory_coherence

<a name="[3]">[3]</a> https://www.khronos.org/opengl/wiki/Memory_Model#Ensuring_visibility

<a name="[4]">[4]</a>https://www.khronos.org/registry/OpenGL/specs/gl/GLSLangSpec.4.60.html#shader-memory-control-functions

<a name="[5]">[5]</a> https://www.khronos.org/opengl/wiki/Memory_Model#Incoherent_memory_access

<a name="[6]">[6]</a> https://registry.khronos.org/OpenGL/specs/gl/GLSLangSpec.4.60.html#changes
