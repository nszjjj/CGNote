## DSA

DSA (Direct State Access) 是从 OpenGL 4.5 版本开始引入的一套 API，它改变了 OpenGL 管理对象状态的方式，旨在提高性能和简化代码。 在 DSA 之前，OpenGL 使用的是一种基于状态机的 API，需要频繁地绑定和解绑对象才能修改其状态。 DSA 使用命名对象 (named objects) 来表示 OpenGL 对象，例如纹理、缓冲区、帧缓冲区等。 这些对象由一个唯一的整数 ID 来标识。 通过 DSA 可以直接使用这些 ID 来操作对象的状态，而无需绑定它们。

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