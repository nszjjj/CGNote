# Unclassify

官方标准手册：[The OpenGL® Graphics System : A Specification(Version 4.6 (Core Profile) - May 5, 2022)](https://registry.khronos.org/OpenGL/specs/gl/glspec46.core.pdf)

现代硬件的运行方式与经典的 OpenGL 模型相差甚远，了解更加现代的 OpenGL 的功能是本文关注的重点。

## OpenGL Extension Prefix

| 扩展前缀      | 定义                                     | 功能描述                                          |
| --------- | -------------------------------------- | --------------------------------------------- |
| GL_ARB_   | 由OpenGL Architecture Review Board提出的扩展 | 代表正式的标准化扩展，经过审查，常被纳入未来核心OpenGL版本，提供功能增强和性能优化。 |
| GL_EXT_   | 由多个OpenGL实现供应商共同开发的扩展                  | 提供不在核心规范中的功能，有多个供应商实现，适用于广泛的硬件。               |
| GL_NV_    | 由NVIDIA公司提出的扩展                         | 针对NVIDIA硬件优化的特定功能，允许访问NVIDIA GPU的独特特性。        |
| GL_APPLE_ | 由Apple公司提出的扩展                          | 针对Apple平台的特定需求，提供与Mac和iOS优化相关的功能。             |
| GL_SGIS_  | 由Silicon Graphics, Inc.提出的扩展           | 关注特定图形处理功能的扩展，例如生成mipmap等。                    |
| GL_INTEL_ | 由英特尔公司提出的扩展                            | 针对英特尔硬件的特定优化和增强功能。                            |
| GL_KHR_   | 由Khronos Group提出的扩展                    | 反映Khronos对OpenGL的维护和新功能，通常是更新和改进的扩展。          |
| GL_SUN    | 由Sun Microsystems提出的扩展                 | 主要用于Solaris平台的特定功能和优化。                        |
| GL_PGI    | 由Portland Group Inc.提出的扩展              | 针对高性能计算和图形的特定功能。                              |
| GL_OML    | 由Khronos Group提出的OpenML扩展              | 用于移动平台的图形功能，提供与OpenML相关的功能。                   |
| GL_OVR    | 由Oculus VR提出的扩展                        | 针对虚拟现实应用中使用的特定图形功能。                           |
| GL_OES    | 由Khronos Group提出的OpenGL ES扩展           | 用于嵌入式系统和移动设备的特定功能，提供与OpenGL ES相关的功能。          |
| GL_WIN    | 由Microsoft提出的扩展                        | 针对Windows平台的图形功能和特性，例如WGL（Windows OpenGL）。    |

出于很多考量都可能会选择使用扩展，例如：

- 最新功能的引入（最新功能通常未能及时纳入核心规范）

- 针对特定硬件优化

- 向后兼容

- 特定应用需求（尤指游戏，例如MRT）

- 实验性功能

## Tessellation

文件0中给出的一个参考网站里有这样一节：[songho.ca - tessellation](https://www.songho.ca/opengl/files/tessellation.zip)

GLU tessellation是利用OpenGL实用库（GLU）进行细分的操作，但是GLU本身已经过时，不推荐在新的项目中使用。 然而，理解GLU tessellation 的概念仍然对图形学编程很有帮助，因为它阐述了多边形细分的基本原理。

细分的目的是：

- 简化渲染：因为要处理的几何体情况多变，可能很复杂（如非凸多边形、孔洞），我们总希望简化成最简单的情形处理，即由三角面构成的凸多边形网格
- 曲面细分：使原有的多边形逼近曲面，获得更平滑、更逼真的效果，同时可以提高精度

GLU细分过时的关键在于它是在CPU端运行的，然后将结果传递给GPU进行渲染。而现代OpenGL依赖几何着色器（OpenGL 3.2+ Core Mode）和细分着色器（OpenGL 4.0+ Core Mode）来实现细分。

若要于现有图形管线结合，则顺序是顶点着色器->细分着色器->几何着色器->片元着色器

## 管道程序

在高版本（4.5以上）OpenGL中，支持使用管道程序 (Pipeline Program)代替着色器程序 (Shader Program)参与渲染流程。使用`glBindProgramPipeline` 选择整个管道，`glUseProgramStages` 指定各阶段程序，因此管道程序可以视为多个着色器程序的集合（含有先后顺序）。

额外地，对于一个着色器程序，同时含有顶点着色器和片元着色器不是必须的。不过虽然一个着色器程序可以包含多个不同类型的着色器阶段（例如，顶点着色器、几何着色器、片元着色器），但对于同一种类型的着色器阶段，例如顶点着色器，只能存在一个。

不过对与只存在顶点着色器的程序，如何知道其计算结果呢？其中一中方法是使用Transform Feedback，不过写回CPU侧意味着容易触及IO瓶颈。

## 带宽

当我们谈论节约带宽，我们在谈论什么？

## 缓冲区视图

OpenGL 4.3 引入了纹理和缓冲区视图，允许创建现有纹理或缓冲区的子集视图，典型的用途是将纹理的某一部分作为新纹理使用

```cpp
GLuint textureView;
glGenTextures(1, &textureView);
glTextureView(textureView, GL_TEXTURE_2D, originalTexture, GL_RGBA8, 0, 1, 0, 1);
```

## SSO

OpenGL 4.1 引入了 SSO，即 Separate Shader Objects。允许将顶点、片段、几何等着色器单独编译和链接，而不是必须链接成一个完整的程序。

## Shader Reflection

即便这是一个在2.0就引入的机制，但是现代的图形程序为了灵活性和拓展性，通常不能忽略它。

OpenGL Shader Reflection 用于在运行时获取着色器程序中 uniform 变量、attribute 变量以及其他接口块的信息，而无需预先知道这些变量的名称或类型。

写引擎时为了灵活性，着色器要传入的信息肯定不能硬编码。在这种情况下，需要着色器的输入进行反射，动态获取输入并在 CPU 侧匹配赋值。也就是说需要约定一些值，通过反射拿到之后比对名称，如果一致的话则把相应的值设置上去。

# DSA

参考：[khronos - EXT_direct_state_access](https://registry.khronos.org/OpenGL/extensions/EXT/EXT_direct_state_access.txt)

## 发展背景

DSA (Direct State Access) 是从 OpenGL 4.5 版本开始引入的一套 API，它改变了 OpenGL 管理对象状态的方式，旨在提高性能和简化代码。

在 DSA 之前，OpenGL 使用的是一种基于状态机的 API，需要频繁地绑定和解绑对象才能修改其状态。 DSA 使用命名对象 (named objects) 来表示 OpenGL 对象，例如纹理、缓冲区、帧缓冲区等。 这些对象由一个唯一的整数 ID 来标识。 通过 DSA 可以直接使用这些 ID 来操作对象的状态，而无需绑定它们。

DSA 的改动原因在于现代图形硬件的变化：硬件变的越来越并行，大量的计算单元在一段时间内以不可变状态工作。

并行性能的发挥需要尽量<mark>减少状态切换和同步开销</mark>。因为在并行计算中，状态切换会导致计算单元之间的同步问题，降低性能。这意味着传统 OpenGL 的状态绑定机制在现代硬件上显得过于低效。另一方面，这使得一些可变的东西变成不可变的，OpenGL 服务器处理不可变的对象通常更快。

现代图形 API 专注于增加每帧的绘制调用（将绘制命令插入命令队列），而不是去修改状态来间接影响 GPU 的绘制流程。

# 