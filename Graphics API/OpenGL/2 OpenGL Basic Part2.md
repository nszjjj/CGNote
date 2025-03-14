# 高级OpenGL

> 其实也不高级，但是 LearnOpenGL 是这么叫的

当然我写这个的目的是对 LearnOpenGL 进行补充，尽量避免变成简单的笔记

## 深度

在没有绑定FBO（或者解绑其他FBO之后）任何操作都是在默认FBO上进行的，默认FBO只包含颜色缓冲区、深度缓冲区和模板缓冲区。通常情况下默认FBO的颜色缓冲区映射到屏幕输出。

深度在写入的时候会写入当前FBO的深度缓冲区，写入的条件如下：

- 启用了深度测试`glEnable(GL_DEPTH_TEST)`（默认是禁用的）

- 允许写入深度缓冲区`glDepthMask(GL_TRUE)`（一般默认为true）

启用深度测试的针对该上下文全局的，但是如果绑定的FBO没有深度缓冲区，则依旧会忽略深度测试。

如果单纯只是想渲染深度，则禁用其余缓冲区写入：

```cpp
glDrawBuffer(GL_NONE); // 重要：禁用颜色缓冲区写入
glReadBuffer(GL_NONE); // 重要：禁用颜色缓冲区读取
```

并且使用的着色器十分简单，在顶点着色器中将正确的顶点位置传递给`gl_Position`即可，片元着色器空着就行。

经典流程下，深度的写入在片元着色器之后，也就是拿到片元的颜色后，再判断是否能写入。相当于在启用的时候，写入片元颜色必然更新深度值。但是在使用了一些优化技术的情况下，例如Early-Z，深度值会远早于片元颜色的写入。

---

深度缓冲区的本质是把近裁面和远裁面之间的值映射到$[0,1]$存储，公式如下：

$$
F_{depth} = \frac{z-near}{far - near}
$$

但是实际上深度缓冲区的值是非线性的，和$1/z$成正比，公式如下：

$$
F_{depth} = \frac{1/z-1/near}{1/far - 1/near}
$$

关于实际使用的深度缓冲区是非线性的，有两方面说明：

- 近处的东西需要更高的精度，远处的东西细节通常不重要

- 在数学原理上，非线性源自于齐次坐标系下的透视除法。在视投影变换后得到一个齐次坐标，转换回三维坐标时会除w，此时就非线性了。

快速线性化深度值的近似方法，相关代码如下：

```glsl
float ndc = depth * 2.0 - 1.0;
float linearDepth = (2.0 * near * far) / (far + near - ndc * (far - near));
```

首先把$[0,1]$的深度转换到NDC的$[-1,1]$范围，然后套用[OpenGL Projection Matrix](https://www.songho.ca/opengl/gl_projectionmatrix.html)的公式可知。

## 模版

模版测试在片元着色器之后，深度测试之前。通过模版测试的片元会保留参与后续的流程，并且在模板操作 (Stencil Operation)合适的情况下会更新模版缓冲区。

在使用模版测试的流程中，首先就是这三样：

```cpp
glEnable(GL_STENCIL_TEST);    // 启用模版测试
// 清空模版缓冲区
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);
// 设置模版掩码（一般是这两种情况）
glStencilMask(0xFF); // 每一位写入模板缓冲时都保持原样
glStencilMask(0x00); // 每一位在写入模板缓冲时都会变成0（禁用写入）
```

在设置好上下文之后即可具体指定模版操作，主要涉及两方面：

- 使用`glStencilFunc(func, ref, mask)`指定<u>模版测试的逻辑</u>。

- 使用`glStencilOp(sfail, dpfail, dppass)`指定模版缓冲区<u>更新的逻辑</u>。

有必要指出的是

- 这俩都是针对当前绑定的FBO而言的，切换FBO需要重新设置。

- `glStencilMask` 和 `glStencilFunc` 都设置了掩码参数，但是前者针对的是更新缓冲区的时候，后者针对的是模版测试比较的时候。

> 额外地：`glStencilMask(0x00)`会导致`glClear(stencil_buffer)`无效

---

`glStencilFunc`为模版测试提供了测试逻辑，测试逻辑是二元运算，第一个值（位于比较符号左侧）源自模版缓冲区，第二个值源自该函数的第二个参数。第三个参数则控制哪些位参与比较，在具体比较之前会把第三个参数和模板缓冲区的值和参考值进行按位与操作。

例如在八位的模版缓冲区情况下，`0xFF`所有位都为1，所以是8位都会参与比较。类似地，`0x0F`则只有模板值的低 4 位会参与比较，高 4 位会被忽略。

```cpp
glStencilFunc(GL_EQUAL, 1, 0xFF)
```

以上述代码为例，表示这个片段的模版值（从模版缓冲区获取）若等于(`GL_EQUAL`)参考值1，片段将会通过测试。

`glStencilOp(sfail, dpfail, dppass)` 函数的三个参数分别规定了在三种不同情况下如何更新模板缓冲区的值

- sfail (Stencil Fail)：模板测试失败时的操作。

- dpfail (Depth Fail)：模板测试通过但深度测试失败时的操作。

- dppass (Depth Pass)：模板测试和深度测试都通过时的操作。

每个参数都可以设置为以下值之一：

| 操作 (Operation) | 模板缓冲区更新行为                                         | 说明                          |
| -------------- | ------------------------------------------------- | --------------------------- |
| `GL_KEEP`      | 保持 `stencil` 不变                                   | 不修改模板缓冲区的值                  |
| `GL_ZERO`      | `stencil = 0`                                     | 将模板缓冲区的值设置为 0               |
| `GL_REPLACE`   | `stencil = ref`                                   | 将模板缓冲区的值替换为参考值 `ref`        |
| `GL_INCR`      | `stencil = min(stencil + 1, 最大值)`                 | 将模板缓冲区的值加 1，如果溢出则保持最大值      |
| `GL_INCR_WRAP` | `stencil = (stencil + 1) % (最大值 + 1)`             | 将模板缓冲区的值加 1，如果溢出则环绕到最小值 (0) |
| `GL_DECR`      | `stencil = max(stencil - 1, 最小值)`                 | 将模板缓冲区的值减 1，如果溢出则保持最小值      |
| `GL_DECR_WRAP` | `stencil = (stencil - 1 + (最大值 + 1)) % (最大值 + 1)` | 将模板缓冲区的值减 1，如果溢出则环绕到最大值     |

对此需要说明的是：

- `最大值` 和 `最小值` 取决于模板缓冲区的位数。通常情况下模板缓冲区是 8 位的，即 `最大值` 为 255，`最小值` 为 0。

- 参考值`ref`源自于上面`glStencilFunc`指定的

---

如果一个场景多个物体有不同的模版操作该怎么办？

### 模版的高级应用

除了物体轮廓之外，模板测试还有很多用途，比如在一个后视镜中绘制纹理，让它能够绘制到镜子形状中，或者使用一个叫做阴影体积(Shadow Volume)的模板缓冲技术渲染实时阴影。

## 混合

谈及混合的语境是含有半透明/透明图像的渲染。混合发生在模版测试和深度测试之后，写入帧缓冲区之前，即位于光栅化的最后阶段。

> 只有在`glEnable(GL_BLEND)`的情况下会执行混合

混合的方程也是老生常谈的

$$
Result = C_{source}*F_{source} + C_{destiny}*F_{destiny}
$$

- $C_{source}$是要混合的颜色，通常来自片元着色器

- $C_{destiny}$是目标颜色，就是帧缓冲区里已有的

由于颜色是已知从哪获取的，描述一个混合操作通常只需要给出两个颜色的Factor即可，或者说给出两个Factor可以唯一低确定一种混合方式。

ShaderLab也有类似的指令控制混合方式，比如`Blend One One`

在OpenGL中通过`glBlendFunc`来指定混合操作，例如`glBlendFunc(GL_ONE, GL_ONE)`。

需要注意的是：

1. 混合不具有交换律

2. 渲染不透明物体前要排序

3. 透明物体的渲染要在所有不透明物体渲染完之后

针对2，排序是针对物体而言的。要么一个物体全在前面，要么一个物体完全在后面（先被渲染）。但是这无法应对物体之间部分交叠的情况：即有一部分在前面，有一部分在后面。这种情况称之为“交错透明度” (Interleaved Transparency)问题。这种问题的一种解决方案是OIT技术。

## 剔除

剔除的前提是知道面朝向，面朝向通过构成面的顶点的绕序确定，顶点绕序由下图所示（图源自LearnOpenGL）：

<img src="file:///D:/CGNote/OpenGL/img/faceculling_windingorder.png" title="" alt="顶点绕序示意图" data-align="center">

通过构成面的点绕序判断，不同图形API是不同的。OpenGL 默认情况下，逆时针 (Counter-Clockwise, CCW) 绕序的三角形被认为是正面。 DirectX 默认情况下，顺时针 (Clockwise, CW) 绕序的三角形被认为是正面。

1. 判断方式可以通过API修改

2. 启用/禁用该设置是全局的

3. 模型面的绕序通常在DCC软件中就处理好了，所有面的绕序都遵循相同规则

顶点绕序可以确定一个三角形的法线方向的本质是计算两个边的叉积。对于给定顶点顺序ABC计算AB和AC的叉积（这同时意味着计算开销）。逆时针绕序，得到的结果朝向摄像机是正面，顺时针类似同理。

设置剔除的代码长这样：

```cpp
glEnable(GL_CULL_FACE);    // 启用
glCullFace(GL_BACK);    // 背面剔除
glFrontFace(GL_CW);    // 设置顺时针绕序为正面
```

---

绕序是一种硬件层面高效的剔除方法，通常由图形API在光栅化阶段自动完成。

顶点法线则是一种软件层面的灵活方法，可以处理各种复杂的几何形状，但是比前者更有开销。

## 帧缓冲

帧缓冲是一个容纳多种缓冲的集合，缓冲的本质是纹理，一般情况下是Texture2D，每种缓冲作为附件归属于一个帧缓冲对象 (Framebuffer Object, FBO)，例如：

- 颜色附件 (Color Attachment)

- 深度附件 (Depth Attachment)

- 模板附件 (Stencil Attachment)

其中颜色缓冲可以有多个。

同其他Object一样，FBO也是用一个GL_uint唯一标识，使用`glBindFramebuffer`指定缓冲区的功能：

```cpp
unsigned int fbo;
glGenFramebuffers(1, &fbo)
glBindFramebuffer(GL_FRAMEBUFFER, fbo);;
```

不过绑定到`GL_FRAMEBUFFE`是读/写，`GL_READ_FRAMEBUFFER`与`GL_DRAW_FRAMEBUFFER`是只读与只写。可以通过绑定的渲染目标控制读写。

在配置完FBO最好调用函数`glCheckFramebufferStatus`检测一下FBO是否完整，完整的FBO是这样的：

1. 至少要有一个颜色附件

2. 所有附件必须完整：必须有分配的内存，并且其大小和格式必须有效

3. 所有附件必须具有相同的样本数：就是MSAA那个样本，每个像素的采样点数目。

4. 不能混合使用纹理和渲染缓冲区：附件要么全是纹理，要么全是缓冲区（OpenGL规范要求的）

渲染到纹理的本质就是绑定一个附加了纹理作为颜色附件的FBO，取代默认FBO。

帧缓冲的首要优势就是实现后处理，后处理是在着色器中进行的，但是在管线的前面，每个片元都是并行处理的，所以在着色器内无法访问片元近邻的其它片元的结果。但是在后处理阶段已经把之前计算好的结果（甚至本来要显示在屏幕上）给存到Texture里。这时候相当于一种隐式的同步操作（等所有片元都处理完）

教程中介绍了几种经典的Filter Kernel，包括Sharpen、Blur、Edge-detection。并指出了Filter时纹理环绕方式对结果的影响，为了更好的效果`GL_CLAMP_TO_EDGE`会比`GL_REPEAT`更好。

## CubeMap

### Basic Concepts

CubeMap 可以实现多种效果，例如天空盒、IBL等。

CubeMap的数据载体通常是6张普通的正方形图像。其读取方式与普通图像无异：使用`glTexImage2D`，但是要调用6次（针对Cube的每个面各一次），并且要指明某张图对应哪个面。为了连续性，通常还要对warp mode 做额外配置，因为在立方体边缘处可能某些时候纹理坐标略微超出$[0, 1]$ 范围，为了不让他采样到另一个面（会导致明显的接缝）会令其`GL_CLAMP_TO_EDGE`由于纹理是正方体，所以会使用`GL_TEXTURE_WRAP_S`、`GL_TEXTURE_WRAP_T` 和 `GL_TEXTURE_WRAP_R`这三个参数对应x、y和z轴方向的采样。这与二维不同，二维只有`GL_TEXTURE_WRAP_S`和`GL_TEXTURE_WRAP_T`。

> 注：据官方文档$^{[1]}$所述，现代硬件能够跨立方体面边界进行插值

不同于二维纹理的采样参数是`vec2`类型的uv，CubeMap 的采样参数是`vec3`类型的方向向量。OpenGL 规范中定义了`GL_TEXTURE_CUBE_MAP_POSITIVE_X`是对应x轴正方向的纹理，其余方向以此类推。

在绘制 SkyBox 的时候，要求场景中必须实际拥有一个Box，用于将 CubeMap 的内容映射上去。Box 的大小通常是任意的，因为会把 Box 的中心置于相机的位置，对 Box 的采样是通过方向向量（相机到Box上某点的方向）进行的，因而 Box 的尺寸（及其世界坐标）通常是无关紧要的。不过要保证尺寸在近裁面和远裁面之间，以确保不会被裁剪丢弃。

在 LearnOpenGL 中，由于天空盒并不是摄像机的子级，所以他会随着摄像机的移动而发生变化。对此采用的做法是在 view 矩阵传入之前重置其中的位移参数，让移动不会影响天空盒的位置向量。

### SkyBox

天空盒的绘制时机有两种：在管线不透明物体绘制阶段最前和管线不透明物体绘制阶段最后。

在不透明物体绘制阶段最前渲染天空盒主要的优势是禁用深度测试和深度写入简化shader，劣势是overdraw。

在管线不透明物体绘制阶段最后渲染主要的优势是减少overdraw

主流引擎似乎都是在管线不透明物体绘制阶段最后（或者说是不透明物体绘制完）再对天空盒进行绘制。

在 LearnOpenGL 中使用这样一个技巧：通过对观察空间的顶点位置`pos`执行`gl_Position = pos.xyww`，此举使得透视除法时Z永远为1，因而必然会处于最远的深度。

### Environment Mapping

CubeMap可以参与环境映射，这主要包括反射和折射，其本质都是找到 view point 到shading point的方向，结合 shading point 的法向找到光路改变的方向。

在 GLSL 中使用内置的`reflect`函数和`refract`函数实现对反射/折射后光线方向`R`的求解。随后沿此方向对 CubeMap 采样即可。不过通常需要注意的是，`R`可能被其他物体遮挡，此时不应当直接对CubeMap采样。

动态环境贴图是一种解决方案，不过开销甚大，基本上不会考虑。

光线追踪毫无疑问地也可以，而且想当准确，但是开销也不低。

其他方法还包括反射探针、PRT和SSR，这部分内容将会在 Games202 里做补充说明，届时本小节将附上指向链接。

### 额外补充

据评论区说：

OpenGL 大部分纹理是左下角逐行向上扫描编码为一维数据，但是 CubeMap 数据是从左上角逐行向下扫描。这是因为采用了名为 RenderMan 的行业标准。该标准由皮克斯动画工作室开发，定义了一系列约定和最佳实践，旨在确保渲染结果的一致性、高效和高质量。

不过我并未在RenderMan

## 高级数据

### 缓冲区的本质与操作拓展

OpenGL 的缓冲只是管理特定显存块的对象，只有在使用`glBindBuffer`将之绑定到一个缓冲目标(Buffer Target)时才有具体的意义。例如绑定到`GL_ARRAY_BUFFER`上则缓存被认为是VBO

缓冲目标依旧是一种访问载点（还记得之前的比喻吗？缓冲目标是USB接口，数据是插在其上的U盘，程序只知道从某个USB接口读数据，不知道上面插的是装有什么数据的U盘）

- 对于任何缓冲，从缓冲目标解绑不意味着数据消失，除非被`glDeleteBuffers`

- 可以在缓冲申请的时候将Data传null，此时只会分配内存，但不进行填充

- 填充缓冲的某一部分数据需要使用`glBufferSubData`

---

此外，`glMapBuffer`可以把 GPU 缓冲区（通常位于显存中）的数据映射到 CPU 的地址空间，随后依据访问标志（读、写、读&写）可能会将CPU对内存相应区块的修改回传至 GPU 缓冲区。CPU 的修改不是实时同步的，一般在调用``glUnmapBuffer``之后才会同步回 GPU。

`glMapBuffer`的本质是内存映射。其具体行为什不仅和 OpenGL 的实现有关，也和操作系统内存管理有关，例如分页机制。通常情况下的宏观表现是按需传输，延迟加载的。相当于把一块显存空间当做内存空间使用，但是实际在发生IO时，和真正内存的IO在底层的操作是不一样的。

（相关参阅：`mmap`系统调用）

- 为什么说通常位于显存，是因为在统一内存架构（UMA）的系统中（例如，某些集成显卡），CPU和GPU共享同一块物理内存

- 内存映射的另一个好处是，从磁盘可以直接读入 GPU 缓冲区而无需额外的临时内存分配

- 显然，回传修改的数据仅仅涉及有改动的部分，这避免了回传整个缓冲区的IO开销。

### 顶点属性的布局结构

VBO的内容组织有两种：交错（Interleave）与分批（Batched）

如果一个顶点具有三个属性，那么四个顶点的数据在交错的情况下是123123123123，在分批的情况下是111122223333。前者相当于一个顶点的数据都放在一起，后者相当于一类数据都放在一起。

分配存放的情况下，很自然地使用`glBufferSubData`来填充数据，当然相应地也需要`glVertexAttribPointer`更新顶点属性指针

### 复制缓冲

使用`glCopyBufferSubData`将 GPU 上的数据从一个缓冲区拷贝到另一个缓冲区，此举无需 CPU 参与。不过只能指定从一个缓冲目标复制到另一个缓冲目标，对于意图从同一类用途的缓冲之间进行复制，例如从一个VBO复制到另一个VBO，显然不能同时把他们绑定到`VERTEX_ARRAY_BUFFER`上，对此提供了`GL_COPY_READ_BUFFER`和`GL_COPY_WRITE_BUFFER`两个缓冲目标。

注：一个缓冲可以绑定到多个缓冲目标上。

## 高级GLSL

### 内建变量

官方文档$^{[2]}$提供了GLSL所有内建变量。

比如

- 可以使用`gl_PointSize`设置点的大小，不过仅对``GL_POINTS``图元才生效，并且各个硬件平台对于最大大小的支持程度不一样。

- `gl_VertexID`储存了正在绘制顶点的当前ID。这个和实例化相关

- `gl_FragCoord`在片元着色器提供了一个`vec3`，其xy坐标是片段的窗口空间坐标，左下角为原点，z是片段的深度值，范围在$[0,1]$

- 在不启用`gl_FrontFacing`的情况下可以通过`gl_FrontFacing`判断片元是不是属于正面（比如说根据这个给正反面渲染不一样的东西）

---

关于`gl_FragCoord.z`与`gl_FragDepth`：

- 如果片段着色器没有写入`gl_FragDepth`，那么`gl_FragCoord.z`的值会被用作片段的深度值，并写入深度缓冲区 。

- 如果片段着色器写入了`gl_FragDepth`，那么写入的值将覆盖`gl_FragCoord.z`，并用作片段的深度值，并写入深度缓冲区。

自行通过`gl_FragDepth`写入深度糊导致 OpenGL 禁用所有的提前深度测试。因为那时测试出的数据可能是不准确的。也就是说由于自行写入深度禁用了提前深度测试，会造成一定的性能影响）。

在4.2+的OpenGL中，可以使用深度条件重新声明

```glsl
layout (depth_<condition>) out float gl_FragDepth;
```

启用了这个意味着，可以在启用提前深度测试的同时，在片元着色器修改其值。例如：

```glsl
layout (depth_greater) out float gl_FragDepth;
```

上述语句在欲写入的z值比原本的z值大的情况下写入，写入不满足条件的值会被忽略。

### 接口块

写个 ShaderLab 的人都知道。

```glsl
out VS_OUT
{
    vec2 TexCoords;
} vs_out;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);    
    vs_out.TexCoords = aTexCoords;
}  
```

在后面着色器接收的时候，块名（上例是`VS_OUT`）要一致，具体变量名无所谓

## UBO

### 对齐规范

OpenGL 支持多种对齐规范，包括但不限于 `std140` 和 `std430`。不同的对齐规范在内存对齐和填充方面有不同的规则，std430比std140更加紧凑，且适合SIMD。

因为对齐的问题，存在一些情况使 uniform 块的总大小大于其元素大小的总和，例如在 std140 的情况下，`vec3`会实际占用16字节（有4个字节的填充）。在这种情况下，使用`glBufferData`填充 UBO 数据的时候，需要考虑到填充字节的存在，指定正确的块大小。

### 绑定点

对于渲染流程中使用多个 uniform 块的情况，必须有机制能够使 OpenGL 知道 CPU 侧的某个 Uniform 缓冲对应的是着色器侧哪个 Uniform 块。

- 着色器中的 uniform 块有自己的索引

- 通过 uniform 块的索引可以将着色器 uniform 块绑定到指定的绑定点

- 一个绑定点可以被多个着色器的多个 uniform 块绑定

- 绑定 uniform 缓冲区对象到绑定点，确保绑定点有实际内容

- 多个着色器的 uniform 块绑定到相同的绑定点意味着这个绑定点的数据被多个着色器共享

示意代码：

```csharp
// 获取指定着色器程序上的 uniform 块索引
GLuint blockIndex = glGetUniformBlockIndex(program, "MyUniformBlock");
// 将 uniform 块绑定到指定的绑定点
glUniformBlockBinding(program, blockIndex, bindingPoint);
// 绑定 uniform 缓冲区对象到绑定点
glBindBufferBase(GL_UNIFORM_BUFFER, bindingPoint, uniformBuffer);

//自 OpenGL 4.2+ 可以有简化声明
// 在着色器中为 uniform 块显式指定绑定的访问点
layout(std140, binding = 2) uniform Lights { ... };
// 在 C# 侧依旧需要绑定 uniform 缓冲区对象到绑定点
glBindBufferBase(GL_UNIFORM_BUFFER, bindingPoint, uniformBuffer);
```

`glBindBufferBase`是绑定指定的缓冲目标到指定的访问点。需要指定缓冲目标意味着肯定不止 UBO 拥有访问点，实际上Shader Storage Buffer Objects (SSBOs)、Atomic Counter Buffer Objects等也都有访问点，这些访问点在不同类型的缓冲目标之间不通用。

使用`glBindBufferRange`可以绑定Uniform缓冲的特定一部分到绑定点，这样的话可以更加灵活地使用UBO：

- 在多线程中分别访问 UBO 的其中某部分

- 把一类 uniform 都放在一个 UBO 内方便管理，并且各个着色器只用自己需要的部分，不需要把整个 UBO 都传过去

- 可以只更新 UBO 的某一部分

<mark>着色器程序与 uniform 块</mark>：

顶点着色器和片元着色器可以访问同一个 uniform 块，不过他们的名称和结构必须一致（同一个声明代码复制到另一个着色器）。因为 uniform 块在编译链接着色器程序的时候会被着色器程序持有，而非被着色器持有。此时两个着色器声明的 uniform 块会被合并成一个，并具有唯一的索引。

### 多光源与UBO

类似于画画，一般情况下先拿到片元对应的diffuse，作为最基础的output。然后对于每一个光源，它对片元的贡献颜色将会加到片段的输出颜色向量上（类似于在基础色上面画亮部和暗部）

一般定向光就1个，但是点光源可能有多个，需要foreach操作逐光源计算。更高级一点，可以先在空间上判断是否在受到点光源影响的范围内再决定是否进行点光源的计算。foreach操作一般在shader中进行，需要将所有参与计算的光源的相关信息传入，例如方向、光强等。当然如果做过games202的作业的话，其实他的

光源会有一些基本的参数，例如位置或者方向。一般使用uniform存储，而不是属性。一般情况下，uniform针对一次DC或者一个场景，绘制时相关uniform的具体值不会频繁变化。而属性则是存那些针对每个顶点或者片元不同的。

对于不同类型的光源依旧可以抽象出很多相同的运算操作，可以考虑

为了方便管理不同的光源的uniform变量，会使用UBO。

### UBO与SSBO

SSBO 是在 UBO 之后提出的，与 UBO 具有一定的相似性：

- 通过访问点实现 CPU 到 GPU 的数据传递，并和指定的着色器关联

- 都是缓冲区对象

- 都可以跨着色器阶段共享

但 SSBO 的提出主要解决了 UBO 在灵活性和容量上的限制，因为 UBO 大小有限（通常为16KB或64KB）且在着色器中只读，不支持随机访问和原子操作。

SSBO 允许着色器以读写模式访问数据，并且大小可以达到128MB或更多（具体取决于硬件实现），支持随机访问和原子操作。甚至可以使用更复杂的数据结构，如不定长数组。

遇到大规模或需要频繁更新的数据（如粒子系统、计算着色器中的中间结果）使用 SSBO 是更好的选择。

由于 SSBO 在 CPU 和 GPU 双侧都可以更新，因此有可能会导致数据竞争或未定义行为，所以需要同步机制。

就大多数现代图形应用中，SSBO 通常的使用方式是在 CPU 侧初始化数据，然后在 GPU 侧更新数据，供后续的 GPU 计算或渲染使用，以实现 GPU-Driven。

创建 SSBO 的代码示意：

```csharp
// 创建SSBO
GLuint ssbo;
glGenBuffers(1, &ssbo);
glBindBuffer(GL_SHADER_STORAGE_BUFFER, ssbo);

// 数据定义
std::vector<glm::vec4> data = { glm::vec4(1.0f), glm::vec4(2.0f), glm::vec4(3.0f) };

// 将数据传输到SSBO
glBufferData(GL_SHADER_STORAGE_BUFFER, data.size() * sizeof(glm::vec4), data.data(), GL_DYNAMIC_DRAW);

// 绑定SSBO
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 0, ssbo);
```

这样就可以在 GPU 侧使用了，如下所示，不过通常对于不定长的数组，需要额外传个数组长度以确保在访问的时候不会越界：

```glsl
layout(std430, binding = 0) buffer MySSBO {
    vec4 data[];
};
```

注：数据上传的顺序需要与GLSL中声明的顺序严格对应

## Geometry Shader

### Basic Concepts

输入：通常由一个完整的图元的顶点组成，不过可以附加额外信息

输出：一个或多个图元（输入的图元类型与CPU侧的绘制调用是一致的，比如说`glDrawElements(GL_TRIANGLES, ...)`那几何着色器接收的肯定就是三角形作为图元）

时机：顶点着色器之后，光栅化之前

代码示例：

```glsl
#version 330 core
layout (triangles) in;          // 输入图元类型：三角形
layout (points, max_vertices = 3) out; // 输出图元类型：点，最大顶点数为 3

void main() {
    for (int i = 0; i < 3; i++) {
        gl_Position = gl_in[i].gl_Position; // 获取输入顶点的位置
        EmitVertex();                      // 发射一个顶点
    }
    EndPrimitive();                        // 结束图元
}
```

每次调用`EmitVertex`都会把当前的`gl_Position`作为一个顶点添加到当前正在构建的图元中，也就是说需要先设置好`gl_Position`才可以。调用`EndPrimitive`会把当然正在构成图元的顶点组装成指定的图元。

如果在调用 `EndPrimitive` 时，通过 `EmitVertex` 添加了比当前输出图元类型所需更多的顶点，OpenGL 会忽略多余的顶点，并仅使用前 N 个顶点来构建图元，其中 N 是当前输出图元类型所需的顶点数。

每次调用`EndPrimitive`后 之前添加的所有顶点都会被清空

### ★着色器间的数据流动与匹配

官方文档参考：[接口匹配规则](https://www.khronos.org/opengl/wiki/Shader_Compilation#Interface_matching)

大意就是分为两种情况：松散变量和接口块

对于松散变量来说，需要满足以下条件：

- 这两个变量代表同一个接口，这是通过按顺序检查以下内容来确定的（1比2优先，通过任意一个即可）
  
  - 两个变量`layout`位置一致
  
  - 两个变量名称一致

- 两个变量的类型匹配。如果它们是数组，则元素数目必须相同

- 具有匹配的类型限定符

对于接口块，则需要满足以下条件：

- 接口块名称匹配

- 输出块具有与输入块相同的成员，这意味着：
  
  - 按相同顺序声明
  
  - 有相同的名字
  
  - 具有完全匹配的类型。如果它们是数组，则元素数量必须匹配
  
  - 具有匹配的类型限定符

就具体使用而言：

着色器 out 的接口块只能被紧随其后的着色器接收，不能被其后续的后续着色器接收。这意味着如果顶点着色器out了一个接口块，在几何着色器存在的情况下，片元着色器无法接收，想要接收必须在几何着色器中转一下out出来。

系统提供的内建变量不遵循上述规则，只要其内容有效，在任何着色器阶段访问均可。例如`gl_position`在顶点着色器之后的任何着色器阶段都可以访问

out 的松散变量可以具有（并且推荐有）location，以为后面的着色器输入进行清晰的匹配

### 实际应用：法向量可视化

两个pass，第一个pass正常绘制物体，第二个pass绘制法向量指示线。

绘制顶点法线会比较简单，只需在顶点着色器传递出来法线方向即可。

此时顶点着色器的`gl_position`给的是乘 mv 之后的值，而不是乘 mvp 之后的值。要让顶点着色器拿到观察空间（相机空间）的顶点坐标，因为在乘 MVP 之后已经是裁剪空间的值了，裁剪空间的坐标比例已经被拉伸了，因此无法表示实际世界的情况。

当然在世界空间或者相机空间都可以做这件事：拿到顶点的位置，生成一个`line_strip`类型的图元。线段的第一个点是顶点本身的位置，第二个点是顶点沿法线方向移动的一段距离的位置

所以几何着色器拿到什么阶段的信息无所谓，主要看它和后续阶段需要什么。如果后续存在片元着色器还需要正确的坐标去做计算，那可能还需要在几何着色器传入投影矩阵，然后及时更新正确的`gl_position`。并不是说必须让顶点着色器输出裁剪空间的坐标，然后后面的几何着色器只能接受裁剪空间的顶点坐标。而且各个坐标要一致，比如说位置是裁剪空间，那传入的法线也得是裁剪空间。

---

当然，如果是要绘制面法线，则需要计算

1. 面的中心点（作为法线起点），计算方式是构成面的顶点坐标取平均值

2. 求面法线：构成面的所有顶点的法线的平均值

其余运算同上。

不必担心具有平滑组的模型的情况。在需要同时实现平滑着色和硬边效果的情况下，顶点使用拆分法线，一个法线会对应一个区域。有些软件的实现方式不是采用存储多个法线，而是额外复制顶点。

<img src="file:///D:/CGNote/Graphics%20API/OpenGL/img/split_normal.png" title="" alt="拆分法线示意图" data-align="center">

在拆分法线的情况下，如果一个顶点属于某平滑组，那么他这部分的拆分法线和顶点法线一致。在他构成的面都属于某个平滑组的时候，整个拆分法线只有一条且和顶点法线一致。如果一个顶点不属于任何平滑组，则拆分法线有多条且分别和构成的面的法线一致。

## 合批与实例化

> 从VAO和物体的对应的角度来切入

最基础的情况下，一个物体对应一个 VAO。但是这样过于低效，因为有很多冗余存在。

有时候我们考虑把静态物体合批，让他们的VBO合成一个。这样确实减少了很多 VAO 和绘制调用，但是在一些情况下 VBO 仍旧存在数据冗余：当复用一些物体的时候，通常模型本身是每变化的，只有位置、旋转或者缩放在变。那么模型的数据完全没必要传多份。

对此，简单的做法是通过 uniform 变量把修改位置的矩阵传进去，这意味着又回到了每个物体一个绘制调用，而且在要更新的变量数量级大的情况下其开销仍旧不可忽略，且无法充分利用 GPU 的并行计算的能力。

于是引入了另一种技术：实例化。实例化能够运行一次 DC 绘制多个物体，并且减少了 CPU 到 GPU 数据 IO 的开销。

模型本身的数据通过`layout (position = x)`传入给顶点着色器，但是对于实例化的情况，还需额外传入实例的信息（如每个实例的位置、旋转、缩放等），从而将“模版”模型变换到对应的位置。

实例化模型的默认位置就是模版实例所在的位置，因为任何变换都是基于上传的 VBO 获取到的`gl_position`。根据要做的变换方式不同，可以传递不同的实例信息。例如对于屏幕空间的移动，可以仅仅传递一个`vector2`，而对于世界空间的移动甚至是旋转缩放，就需要传递一个矩阵。

最终使用`glDrawArraysInstanced`进行实例化。不过关键之处仍是在着色器中利用`gl_InstanceID`访问每个实例的信息。

就传递实例信息的手段而言，固然可以在uniform里传数组，然后每个实例的`gl_InstanceID`会随着实例化的进行而变化，不过uniform有上限，对于数据量过大的情况，会把实例化相关的数据做到顶点属性里，这种技术称之为实例化数组（Instanced Array），这种手段下实例化数据的获取和顶点属性的获取无异。

使用实例化数组则无需手动使用`gl_InstanceID`更新数据，而是通过`glVertexAttribDivisor(属性index, 更新频率)`指定更新频率并自动更新。当然依旧可以访问到`gl_InstanceID`以进行一些差异化的操作。此种方法在访问实例数据的方式上和普通的输入变量无异。这部分实例数据一般作为单独的 VBO 而非和常规的顶点数据放在一起。

不过顶点属性只允许最大到`vec4`，当意图传入`mat4`的时候，实际上应当传入4个`vec4`。

额外地，实例信息可以放在纹理中供实例化时查找，SSBO 亦可以实现对实例化信息的更新。这两种方式适合大规模的实例化，特别是需要动态更新的场合。（实例化数组适合中小规模）

## 抗锯齿

### SSAA

超级采样抗锯齿（Super Sample Anti-aliasing）的原理是以高于显示分辨率的倍数（如2x、4x）渲染场景，生成更多像素。对每个输出像素，从它对应的高分辨率区域中取多个采样点。然后将多个采样点的颜色值平均，得到最终像素颜色。

这种方式质量最好，不过开销也不容忽略。

### MSAA

多重采样抗锯齿（Multisample Anti-aliasing）是借鉴了SSAA理念但更为高效的现代抗锯齿手段。核心思想是通过对每个像素的子采样点进行覆盖测试和颜色混合，来确定当前像素最终会以多少比例混合到帧缓冲区中。

采样是不可避免的，但是可以不渲染出高倍分辨率再采样，而是把采样区域就限制在1x分辨率的像素内。

像素是有面积的，在原本渲染的时候，如果像素的中心在一个面片覆盖的区域内，才会把这个像素渲染为该面片。借用 LearnOpenGL 的图是这样：

<img src="file:///D:/CGNote/Graphics%20API/OpenGL/img/anti_aliasing_sample_points.png" title="" alt="多重采样的示意" data-align="center">

原先这个像素的中心点在面片外面，所以直接被丢弃了（不会运行片元着色器）。但是现在在这个像素的区域内，进行了多次采样。如果采样点都属于一个物体，则只会运行一次片元着色器。当其中一个到多个采样点被某物体覆盖，就会为该物体运行一遍片元着色器，并把结果写到该采样点的颜色缓冲中。这意味着边缘位置会运行多次片元着色器，并且最终会把该像素对应的所有颜色缓冲求平均值作为该像素的缓冲。

在这个流程中，几何内部只运行一次片元着色器，几何边缘可能运行多次片元着色器，这显著减少了计算次数。

MSAA 相比于 SSAA 减少了计算开销，但是占据的空间（如颜色缓冲或者深度缓冲）仍旧没减少，因为需要多倍的空间存储每一个采样点的信息。

### 在 OpenGL 中使用多重采样

所幸 MSAA 已经被 OpenGL 所支持，只需要

- 为窗口系统开启多重采样缓冲

- 为 OpenGL 开启`glEnable(GL_MULTISAMPLE)`

<mark>离屏 MSAA</mark>：

使用`glTexImage2DMultisample`生成支持存储多个采样点的纹理

使用`glFramebufferTexture2D`将多重采样纹理附加到帧缓冲

在开启多重采样的时候，深度和模版也得是多重缓冲，所以如果要在离屏渲染的时候，FBO 的相关附件需符合此要求。使用`glRenderbufferStorageMultisample`为之分配多重采样的 RBO。

使用Blit（位块传输）将 FBO 的结果复制到指定的缓冲里。`glBlitFramebuffer`函数会读取`GL_READ_FRAMEBUFFER`和`GL_DRAW_FRAMEBUFFER`渲染缓冲，以决定复制的源和目标。

```cpp
glBindFramebuffer(GL_READ_FRAMEBUFFER, multisampledFBO);
glBindFramebuffer(GL_DRAW_FRAMEBUFFER, 0);
glBlitFramebuffer(0, 0, width, height, 0, 0, width, height, GL_COLOR_BUFFER_BIT, GL_NEAREST);
```

`glBlitFramebuff`前四个参数指定读取帧缓冲中的源矩形的左下角坐标和右上角坐标，紧接着四个参数指定绘制帧缓冲中的目标矩形的左下角坐标和右上角坐标。这里相当于整个读取缓冲的内容复制到整个绘制缓冲。

该方法会从`GL_READ_FRAMEBUFFER`读取并将数据写入到`GL_DRAW_FRAMEBUFFER`。对于这两种缓冲目标，如果未绑定，则其默认值是窗口系统提供的帧缓冲。

`GL_COLOR_BUFFER_BIT`指明复制的缓冲类型，这里只复制颜色缓冲

最后一个参数是滤波方法。

额外地，`glCopyBufferSubData`和`glBlitFramebuffer`的区别在于前者针对原始字节数据，后者处理的是像素数据，这会考虑到转换、缩放、滤波等操作。

<mark>在着色器中访问多重采样的结果</mark>：

使用`sampler2DMS`而非`sampler2D`

```glsl
vec4 colorSample = texelFetch(screenTextureMS, TexCoords, 3);  // 第4个子样本
```

## PBR

> 考虑到本节是一个巨大的主题，本主体的内容单独更新到一个文档中

# Beyond LearnOpenGL

## 多次渲染与多通道渲染

多次渲染 (Multi-pass Rendering)和多通道渲染 (Multi-channel Rendering)是两个概念。

## AlphaTest与AlphaBlend

## Order-Independent Transparency

上文在“混合”一节种提到了交错透明度的问题，这是一种源于对物体进行排序然后按顺序渲染的透明度渲染方法产生的问题。实际上这种方法既有性能开销（因为要排序）而且极依赖于排序结果，排序不正确效果就不正确。

OIT，即顺序无关透明度，是一种在光栅化计算机图形学中用于渲染3D场景中透明效果的技术，它不需要按照几何体的排序顺序进行alpha混合。

OIT 的实现也有很多种方式，对于实时渲染来说 Weighted Blended OIT 是一种常见的方式。加权 OIT 的思路是由片元到相机的距离决定权值。

## RBO 与 FBO

渲染缓冲对象（Renderbuffer Object, RBO） 是 OpenGL 中用于离屏渲染的一种缓冲区，通常与 FBO 结合使用。RBO 主要用于存储没有直接纹理格式的渲染数据，如深度、模板等。没有直接纹理格式的意思是某些数据（如深度、模板等）虽然可以存储在缓冲区中，但不能直接作为纹理使用。

```cpp
// 生成并绑定 RBO
GLuint rbo;
glGenRenderbuffers(1, &rbo);
glBindRenderbuffer(GL_RENDERBUFFER, rbo);

// 分配存储空间
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, 800, 600);

// 生成并绑定 FBO
GLuint fbo;
glGenFramebuffers(1, &fbo);
glBindFramebuffer(GL_FRAMEBUFFER, fbo);

// 附加 RBO 到 FBO
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);

// 检查完整性
if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE) {
    std::cerr << "Framebuffer is not complete!" << std::endl;
}

// 解绑
glBindFramebuffer(GL_FRAMEBUFFER, 0);
glBindRenderbuffer(GL_RENDERBUFFER, 0);
```

当然 FBO 的深度和模版附件本身可以绑纹理，不一定是 RBO。绑纹理的优势在于可以在着色器访问

## Context 概念

OpenGL 本身有 Context 的概念，不过在 LearnOpenGL 的系列文章中并没有体现。因为在使用 GLFW 的时候，该库在管理窗口的同时会自行创建并管理一个与窗口相关联的 Context，因此无需手动管理。此时窗口为 OpenGL 提供了渲染目标。

OpenGL 支持多个Context 切换。特别是在多个窗口渲染的情况下，每个窗口对应一个 Context。不过每个 Context 并非一定要与窗口关联，例如离屏渲染。

此外 Context 的切换具有一定的性能开销，所以尽量避免频繁地切换。离屏渲染时也可以通过 Context 实现，不过就开销而言。通常会选择 FBO 替代。

OpenGL 的各种数据类型中只有一部分类型能够在 Context 之间共享，能够共享的东西需要显式通过代码进行设置。原则上来说 OpenGL 能够共享的东西是纯粹的数据对象，例如 VBO 这种，而[容器对象](https://www.khronos.org/opengl/wiki/OpenGL_Object#Container_objects)（能够将其他对象附加到其上的）则不可以。

Context 的共享，在不同平台的实现是不一样的。例如在 Windows 平台需要使用 WGL (Windows Graphics Library) 来处理与 Windows 窗口系统交互的部分。当然，在使用 GLFW 后，只需要关注 OpenGL 渲染代码本身，而无需关心底层平台相关的窗口管理和上下文创建细节。

## Query Object

> 官方文档：[Khronos - Query_Object](https://www.khronos.org/opengl/wiki/Query_Object)

Query  Object 主要用于性能分析，是一种用于查询 GPU 执行状态或性能数据的机制。

单个对象可以复用，在一次查询中作为一种类似“句柄”的作用，可以在查询完后经由`glGetQueryObjectuiv`查询到某个查询对象的结果。

例如：

- 遮挡查询，其结果表示查询期间有多少片元通过深度测试

- 图元查询，其结果表示在渲染管线中，特别是几何着色器和变换反馈阶段，生成的或写入的图元数量。

- 耗时查询，其结果表示查询范围内所有指令执行完毕所用的时间

注：图元查询，有`GL_PRIMITIVES_GENERATED`和`GL_TRANSFORM_FEEDBACK_PRIMITIVES_WRITTEN`两种

# Reference

[1] [khronos - 立方体贴图纹理](https://www.khronos.org/opengl/wiki/Cubemap_Texture) 

[2] [Built-in Variable (GLSL)](https://www.khronos.org/opengl/wiki/Built-in_Variable_(GLSL))
