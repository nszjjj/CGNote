# 进阶功能

## Tessellation

文件0中给出的一个参考网站里有这样一节：[songho.ca - tessellation](https://www.songho.ca/opengl/files/tessellation.zip)

GLU tessellation是利用OpenGL实用库（GLU）进行细分的操作，但是GLU本身已经过时，不推荐在新的项目中使用。 然而，理解GLU tessellation 的概念仍然对图形学编程很有帮助，因为它阐述了多边形细分的基本原理。

细分的目的是：

- 简化渲染：因为要处理的几何体情况多变，可能很复杂（如非凸多边形、孔洞），我们总希望简化成最简单的情形处理，即由三角面构成的凸多边形网格
- 曲面细分：使原有的多边形逼近曲面，获得更平滑、更逼真的效果，同时可以提高精度

GLU细分过时的关键在于它是在CPU端运行的，然后将结果传递给GPU进行渲染。而现代OpenGL依赖几何着色器（OpenGL 3.2+ Core Mode）和细分着色器（OpenGL 4.0+ Core Mode）来实现细分。

若要于现有图形管线结合，则顺序是顶点着色器->细分着色器->几何着色器->片元着色器
