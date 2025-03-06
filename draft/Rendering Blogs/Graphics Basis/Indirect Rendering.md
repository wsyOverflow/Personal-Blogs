---
typora-copy-images-to: ..\..\..\images\Rendering Blogs\Graphics Basis\${filename}.assets
typora-root-url: ..\..\..\
title: Indirect Rendering
keywords: Graphics, Indirect Rendering
categories:
- [Rendering Blogs, Graphics, Indirect Rendering]
mathjax: true
---

# 1 概述

比较古早的渲染模式是按照primitive来访问 vertex buffer 或通过 index buffer 来访问 vertex buffer，可分为三大类：

- **non-indexed rendering**: 直接访问 vertex buffer 的顶点数据
  - [`glDrawArrays`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDrawArrays.xhtml)(GLenum mode, GLint first, GLsizei count)：直接访问 vertex buffer 的顶点数据，例如图元为 `GL_TRIANGLES`时，每三个为一组构成一个三角形。参数：mode 指定图元类型，first + count 指定了在vertex buffer中的绘制范围
- **indexed rendering**: 通过 index buffer 来访问 vertex buffer
  - [`glDrawElements`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDrawElements.xhtml)(GLenum mode, GLsizei count, GLenum type, const void * indices) : 通过 index buffer 中的顶点索引访问 vertex buffer。
    参数：count 本次绘制的索引数量，type 是索引的数据格式。当绑定了 `GL_ELEMENT_ARRAY_BUFFER` 时，indices 被 reinterpret_cast<uint64_t> 作为buffer的偏移量；否则，指向CPU的一段索引数据。
  - [`glDrawElementsBaseVertex`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDrawElementsBaseVertex.xhtml)(GLenum mode, GLsizei count, GLenum type, const void * indices, GLint basevertex) ：与 glDrawElements 基本相同，除了 basevertex 会为每个顶点索引增加一个偏移量。当 basevertex = 0 时，等价于 glDrawElements。
- **instanced rendering**: 一次性提交相同几何数据的多个 instance
  - `glDrawArraysInstanced`(GLenum mode, GLint first, GLsizei count, GLsizei instancecount)：绘制内容与glDrawArrays的指定方式相同。将指定的绘制内容作为instance，多次绘制 (instancecount)。每个instance都具有一个索引 gl_InstanceID。一般来说，会将instance数据单独组织到一个buffer里，使用 gl_InstanceID 来索引，例如每个 instance 的变换。
  - [`glDrawArraysInstancedBaseInstance`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDrawArraysInstancedBaseInstance.xhtml): 相比于 glDrawArraysInstanced 多了 baseinstance 参数，作为 instance 索引的偏移量
    baseinstance = 0 时，等价于 glDrawArraysInstanced。
  - [`glDrawElementsInstanced`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDrawElementsInstanced.xhtml)(GLenum mode,  GLsizei count,  const void * indices, GLsizei instancecount)：instance rendering，绘制内容与glDrawElements 的指定方式相同。将指定的绘制内容作为instance，多次绘制 (instancecount)。
    `glDrawElementsInstancedBaseVertex`：额外增加一个 basevertex 参数
  - `glDrawElementsInstancedBaseInstance`：相比于 glDrawElementsInstanced 增加了 baseinstance，作为 instance 索引的偏移量。
    baseinstance = 0 时，等价于 glDrawElementsInstanced。

这些渲染API全都是在参数中指明本次渲染的数据范围、instance 数量等等，也就是需要CPU端提交到GPU端。这就意味着渲染指令需要在CPU侧生成并提交，draw call数量很高会造成CPU侧的高性能开销。而 indirect draw 可以将渲染指令的参数打包到gpu indirect buffer中，一次性发出buffer中的所有draw call。此外，indirect buffer的内容可由GPU生成，减少了CPU与GPU之间的通信开销。

# 2 OpenGL

indirect rendering 中的 indirect draw 是指 draw call 参数不再直接使用API参数指定，而是存放在 indirect buffer 中。multi indirect draw 是对 indirect draw 的批处理命令，以 array 形式存放多个命令参数。indirect buffer 需要根据 indirect API 来填充不同数据结构的命令，而 indirect API 支持的命令里都带有 instance 参数，因此可分为 non-indexed 和 indexed 两大类。

## 2.1 Non-indexed rendering

[`glDrawArraysIndirect`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDrawArraysIndirect.xhtml)(GLenum mode, const void *indirect)：mode 是图元类型，indirect 参数在已绑定 `GL_DRAW_INDIRECT_BUFFER` 时，会被 reinterpret_cast<uint64_t> 作为buffer的偏移量；在未绑定时，作为指向命令的CPU地址，即 DrawArraysIndirectCommand\*。indirect buffer 中所记录命令的数据结构如下，实际上与 glDrawArraysInstancedBaseInstance 等价

```c++
typedef struct {
   GLuint  count;
   GLuint  instanceCount;
   GLuint  first;
   GLuint  baseInstance;
} DrawArraysIndirectCommand;

// 等价于
glDrawArraysInstancedBaseInstance(mode, cmd->first, cmd->count, cmd->instanceCount, cmd->baseInstance);
```

[`glMultiDrawArraysIndirect`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glMultiDrawArraysIndirect.xhtml)(GLenum mode, const void *indirect, GLsizei drawcount, GLsizei stride)：是对 glDrawArraysIndirect 的批处理。此时的 indirect buffer 存放的是 DrawArraysIndirectCommand 数组。参数 drawcount 是 buffer 中存放了多少个 glDrawArraysIndirect 命令，而 stride 是命令参数数据的访问步长。

OpenGL 4.6 引入一个新的扩展，允许使用绑定到 `GL_PARAMETER_BUFFER` 的buffer提供 drawcount 参数。

`glMultiDrawArraysIndirectCount`(GLenum mode, const void *indirect, GLintptr drawcount, GLsizei maxdrawcount, GLsizei stride)：除了 drawcount，maxdrawcount 参数，其余都和 glMultiDrawArraysIndirect 相同。其中 drawcount：`GL_PARAMETER_BUFFER`的偏移量；maxdrawcount 是 indrect buffer 中draw command的最大数量

## 2.2 Indexed rendering

[`glDrawElementsIndirect`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDrawElementsIndirect.xhtml)(GLenum mode, GLenum type, const void *indirect)：等价于 glDrawElementsInstancedBaseVertexBaseInstance，命令数据结构如下

```c++
typedef struct {
    GLuint  count;
    GLuint  instanceCount;
    GLuint  firstIndex;
    GLuint  baseVertex;
    GLuint  baseInstance;
} DrawElementsIndirectCommand;

// 等价于
glDrawElementsInstancedBaseVertexBaseInstance(mode, cmd->count, type, cmd->firstIndex * size-of-type, cmd->instanceCount, cmd->baseVertex, cmd->baseInstance);
```

[`glMultiDrawElementsIndirect`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glMultiDrawElementsIndirect.xhtml)(GLenum mode, GLenum type, const void *indirect, GLsizei drawcount, GLsizei stride)：是对 glDrawElementsIndirect 的批处理。drawcount 与 stride 如前所述

`glMultiDrawElementsIndirectCount`(GLenum mode, GLenum type, const void *indirect, GLintptr drawcount, GLsizei maxdrawcount, GLsizei stride)：除了 drawcount，maxdrawcount 参数，其余都和 glMultiDrawElementsIndirect 相同。

# 3 Vulkan

