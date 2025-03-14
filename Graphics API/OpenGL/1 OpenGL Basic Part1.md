# 正式内容之前

## CLion配置OpenGL环境

1. 打开下载的GLFW源码，`设置=>构建、执行、部署=>CMake`在缓存变量一栏勾选`BUILD_SHARED_LIBS`，然后取消勾选`GLFW_BUILD_EXAMPLES`和`OFFGLFW_BUILD_TESTS`

2. 还是上面的页面中可以找到输出目录，在目录的`./src`中找到`glfw3.dll`、`libglfw3.a`和`libglfw3dll.a`，这些放在自己工程的`./lib`文件夹下

3. 在下载的源码中找到`include/GLFW`，把这个文件夹复制到自己项目的`./include`下

4. GLAD得到的两个文件夹`glad`和`KHR`复制到自己项目的`./include`下

5. GLAD得到的`glad.c`复制到自己项目的`./src`下

配置CMake文件如下，其中glfw1是项目的名字，可替换：

```cmake
cmake_minimum_required(VERSION 3.29)
project(glfw1)

set(CMAKE_CXX_STANDARD 20)

find_package(OpenGL REQUIRED)
include_directories(${OPENGL_INCLUDE_DIRS})

# 指定 include 文件夹
include_directories(
    ${PROJECT_SOURCE_DIR}/include
    ${PROJECT_SOURCE_DIR}/glfw3
)

add_executable(glfw1 src/main.cpp
        src/glad.c)

link_directories(${PROJECT_SOURCE_DIR}/lib)
target_link_libraries(glfw1 ${PROJECT_SOURCE_DIR}/lib/libglfw3.a)
```

当然，在项目资源列表对`glad.c`右键有选项直接加到CMake文件里。

>  题外话，其实好的做法是把GLFW作为子模块，独立于项目编译

## 前置概念速通

### 两种模式

核心模式：是 OpenGL 3.2 及更高版本引入的一种上下文配置方式，它与兼容性模式（Compatibility Profile）相对。因为要移除已弃用的功能、精简API和提高性能，所以不打算向以前兼容。

兼容模式：顾名思义的

### GLAD+GLFW的Init工作

1. GLFW初始化（比如说需要知道使用的是什么API的什么版本，毕竟GLFW也可以用于Vulkan）

2. 创建GLFW窗口

3. 初始化GLAD：加载 OpenGL 函数指针，通过由窗口系统提供给GLAD的函数来进行。

4. 创建OpenGL的视口，注册窗口改变时候的回调（通常情况下我们希望视口和窗口一样大）

关于3，例如SDL提供了`SDL_GL_GetProcAddress`，GLFW则是`glfwGetProcAddress`。

关于4，<mark>窗口≠视口</mark>。视口是OpenGL的概念，所以放在3后面，因为3进行完才能使用OpenGL的API。OpenGL 的视口坐标系是一个二维坐标系，其原点位于窗口的左下角。 x 轴向右，y 轴向上。

### Shader Program

OpenGL使用着色器的流程如下：

1. **从文本到着色器对象:**  提供文本形式的 GLSL 代码被 OpenGL 在 CPU 侧编译成一个着色器对象。 此过程中 OpenGL 会检查代码的语法和语义错误。  编译成功后，生成的着色器对象包含了编译后的着色器代码的二进制表示，以及其他一些元数据。

2. **创建着色器程序:**  一个着色器程序是一个容器，它包含了多个着色器对象（顶点着色器、片段着色器、几何着色器等等）。

3. **附加与链接:**  使用`glAttachShader` 函数将编译好的着色器对象（顶点着色器、片段着色器等）附加到指定的着色器程序对象中。使用 `glLinkProgram` 函数将附加到着色器程序中的多个着色器对象链接在一起。 这个链接过程会检查着色器对象之间的兼容性（例如，输入输出变量的类型和数量是否匹配），并生成一个可执行的着色器程序。

4. **使用着色器程序:**  `glUseProgram` 函数激活着色器程序。  一旦激活，OpenGL 就会使用这个着色器程序来处理后续的绘制命令。  这意味着，当 OpenGL 绘制几何体时，它会使用这个着色器程序来处理顶点数据并生成最终的像素颜色。

注意，使用着色器程序之后， 后续的绘制调用都会使用这个着色器程序来处理顶点数据和生成像素颜色，因此用完之后使用 `glDeleteProgram(programID)` 函数卸载着色器程序，并且它占用的 GPU 资源将被释放。

那么创建着色器程序会占用什么 GPU 资源？

1. 着色器代码（指令）

2. Uniform 变量

3. 纹理

额外地：着色器对象在编译为着色器程序之后可以直接删除（如果以后用不到的话）

### Pipeline

![](./img/pipeline.png)

再看回来，我发现我几乎忽略了图元装配这个阶段。

1. 光栅化是无法对单个顶点完成的，所以需要装配成可以处理的图元

2. 顶点着色器的输出已经是NDC了

3. OpenGL只处理NDC坐标的数据，也就是$x \in [-1,1],y \in [-1,1],z \in [-1,1]$

### ※ VAO、VBO、EBO与FBO

VBO：存储顶点数据的内存区域。将顶点数据（例如位置、颜色、纹理坐标等）以数组的形式存储在显存中，供 GPU 直接访问。<u>本质是字节流</u>。

VAO：需要按照固定的格式编码和解码VBO这个字节流，这就是VAO的作用。VAO定义了一个顶点含有的所有属性的总字节长度，以及按怎样的顺序解读（例如前$4*3$个字节是位置，之后$4*4$个字节是顶点色等）。并且由于一个顶点的所有属性长度是固定的，按照这个进行偏移可以读取字节流中第N个顶点的数据

EBO：就是一组顶点索引，表示绘制用到的顶点有哪些（及其顺序）。

如果没有 EBO，OpenGL 会根据 VBO 中顶点数据的顺序，按顺序将顶点组合成图元（通常是三角形）。这在处理简单几何体时可能没问题，但对于复杂几何体，这种方式可能有潜在的灵活性问题：

1. **难以表示共享顶点:** 复杂几何体通常包含许多共享顶点的面片。例如，一个立方体有8个顶点，但每个面都共享部分顶点。如果只用 VBO，每个面都需要重复存储共享顶点的坐标数据，导致<u>数据冗余</u>，浪费内存和带宽。使用 EBO，每个顶点只需要存储一次，通过索引就可以重复使用。

2. **难以控制绘制顺序:** EBO 允许精确控制绘制图元的顺序。这对于实现某些高级渲染技术（例如，控制渲染顺序以实现正确的深度测试或透明度混合）至关重要。 没有 EBO 则只能按照 VBO 中顶点数据的顺序绘制。

3. **难以构建复杂的拓扑结构:** 一些复杂的几何体具有非简单的拓扑结构，例如具有多个不相连部分的模型。 只有使用 EBO 才可以通过索引来定义这些不相连部分，并控制它们的绘制顺序。

4. **难以进行实例化渲染:** 实例化渲染是一种高效的渲染技术，可以重复使用相同的几何体数据来渲染多个实例。 使用 EBO 可以通过索引来访问相同的顶点数据，并为每个实例设置不同的变换矩阵，做到了灵活地控制顶点数据的重复使用。

FBO：是一个容器，管理多个RenderTarget， 一个 FBO 可以有多个颜色RenderTarget（绑定到 `GL_COLOR_ATTACHMENTi`），一个深度渲染目标（绑定到 `GL_DEPTH_ATTACHMENT`），以及一个模板渲染目标（绑定到 `GL_STENCIL_ATTACHMENT`）

RenderTarget 在 GPU 显存中体现为纹理或渲染缓冲区的内存块。纹理其实本质上也是缓冲区，但是在使用和对待上与传统的渲染缓冲区（renderbuffer）略有区别。

---

额外地，在 OpenGL 中创建 `Buffer Object` 本身并没有指定其用途（例如 VBO、IBO、UBO 等）。 `glGenBuffers` 只会生成一个空的缓冲区对象，它只是一个在 GPU 上分配的内存块，需要通过 `glBindBuffer` 将其绑定到特定的目标（target），才能赋予它具体的用途。

- 可以尝试将同一个缓冲区对象绑定到不同的目标，但是只有最后一个绑定有效。

- 一个目标可以被多个缓冲区对象绑定，只有最后一个绑定生效（之前的绑定会被自动解除）

这意味着通常需要尽量保证在一个VBO确实用完之后再载入下一个VBO使用，以避免额外地IO开销。

参考阅读：[zhihu - 深入理解顶点索引（Vertex Indexing）](https://zhuanlan.zhihu.com/p/18248843576)

### 从uniform变量讲变量类型

uniform变量的性质：

1. 全局性：uniform变量在一个绘制调用（draw call）期间是全局可见的，所有的顶点或片段着色器都可以访问它们。
2. 只读常量：在绘制前是可以设置，但是提交到着色器上是只读常量，不能在着色器内部被修改。它们的值由应用程序在绘制之前设置。

常用来存放：

- 变换矩阵：如模型矩阵、视图矩阵和投影矩阵，用于控制物体在场景中的位置、方向和缩放。
- 光照参数：如光源的位置、颜色和强度等，用于计算光照效果。
- 材质属性：如反射系数、环境光、漫反射和镜面反射系数等，用于控制物体的外观。
- 纹理：传递纹理单元的ID，以便在片段着色器中使用纹理。

值得指出的是，不是所有传递到着色器的变量都是uniform的，从下面的简化版着色器的代码可以看到：

```glsl
// 顶点着色器
#version 330 core
layout(location = 0) in vec3 aPos; // Attribute
layout(location = 1) in vec3 aNormal; // Attribute

uniform mat4 model; // Uniform
out vec3 FragPos; // Out variable
out vec3 Normal; // Out variable

void main() {
    FragPos = vec3(model * vec4(aPos, 1.0));
    Normal = aNormal;
    gl_Position = model * vec4(aPos, 1.0);
}

// 片段着色器
#version 330 core
in vec3 FragPos; // In variable
in vec3 Normal; // In variable
uniform vec3 lightPos; // Uniform
out vec4 FragColor;

void main() {
    // 计算光照
    // ...
    FragColor = vec4(1.0, 0.0, 0.0, 1.0); // 输出颜色
}
```

就修饰的类型关键字这个角度来说，还可以分为In Variables和Out Variables：顶点着色器接收`in`修饰的Attribute Variables、片元着色器使用 `in` 关键字来接收来自顶点着色器的插值数据（对于顶点着色器来说是Out Variables）

实际上，在顶点着色器和片段着色器之间传递数据也被称为Varying Variables (或 Interpolated Variables)：因为OpenGL会自动对这些变量进行<mark>插值</mark>。

此外，还有一些以`gl_`开头的内置变量，例如`gl_position`。它们都代表着由渲染管线提供的特定信息，而不是用户自定义的数据。它们的作用范围限定在特定的着色器阶段（顶点着色器、几何着色器、片元着色器等），并且其值通常由 OpenGL 管线自动提供或根据着色器中的计算结果而确定。

值得注意的是，`gl_position`是顶点着色器中唯一一个必须由用户写入的

类比来看，写Unity Shader用到的`SV_`系统值（System Value）和`gl_`极为相似。不过`SV_`可写入的似乎多一些。

### Texture in OpenGL

> 概念辨析

<mark>图像数据</mark>：图像从存储介质加载到内存是字节流，并非OpenGL直接能处理的纹理，此时只是二进制数据。

<mark>纹理对象</mark>：是`GLuint`类型，相当于对实际数据的引用。使用`glTexImage2D`将图像数据解析为 OpenGL 能处理的数据并让纹理对象引用这块数据，这块数据最终会存在 GPU 侧。OpenGL 依旧不会直接访问各种纹理对象，只会通过当前激活的纹理单元，按照给定的纹理目标访问。因此需要将纹理对象绑定到纹理目标上。

<mark>纹理目标</mark>：归属于纹理单元，每次取用某类型纹理目标的数据，就会从当前纹理单元的纹理目标拿到对应纹理对象的引用。如`GL_TEXTURE_2D`就是一种最常见的纹理目标。对纹理的操作只能通过纹理目标进行

<mark>纹理单元</mark>：使用`glActiveTexture(GL_TEXTUREx);`激活，一个纹理单元可以持有多个纹理目标，但是每类纹理目标不能持有多个，例如不能有2个以上的`GL_TEXTURE_2D`

总结与补充：

- 纹理对象被 CPU 视作句柄

- 可以使用`glGenerateMipmap`生成Mipmap

- 纹理目标相当于访问载点，或者比喻为 USB 接口。配置好从哪个 USB 接口读数据，至于上面插了什么U盘并不重要。这意味着`glBindTexture`为同一个纹理目标绑定多次，只会有一个（且是最后一个）生效。反过来说，同一时刻一个纹理目标只能被一个纹理对象绑定。

- 纹理单元典型的应用情况是一个着色器意图使用多个2DTexture的情况。默认情况下，`glBindTexture` 绑定到纹理单元 0 (`GL_TEXTURE0`)，无需显式调用函数指定活动的纹理单元为0（这也是为什么LearnOpenGL的教程在只使用一个纹理的时候，声明采样器<u>没有从cpp代码里赋值</u>的原因：因为默认值是0，恰恰对应`GL_TEXTURE0`）

如果需要使用多个纹理，就必须使用 `glActiveTexture` 来选择其他纹理单元，大致示例是这样的：

```cpp
GLuint texture1, texture2; // 假设已加载两个纹理，并获得了它们的ID

glActiveTexture(GL_TEXTURE0); // 选择纹理单元0
glBindTexture(GL_TEXTURE_2D, texture1);

glActiveTexture(GL_TEXTURE1); // 选择纹理单元1
glBindTexture(GL_TEXTURE_2D, texture2);
```

> 注意：显卡支持的纹理单元数量有限，可以通过查询 GL_MAX_TEXTURE_UNITS 来获取。OpenGL至少保证有16个纹理单元供开发者使用，即GL_TEXTURE0到GL_TEXTRUE15

---

接下来对 GPU 视角下的纹理做说明：

前文提到，纹理具体数据的填充要使用`glTexImage2D`（针对2D纹理），它会从字节流按指定类型读取并构建为OpenGL支持的纹理格式，结果会存放到GPU中（此举完成了纹理由内存到显存的输送）。需要注意的是，此过程实际上是异步的，详见[4 异步与同步.md](./4 异步与同步.md)一文。

GPU侧的纹理数据的第一个元素对应于纹理图像的左下角。后续元素从左向右穿过纹理图像最低行中的剩余纹素，然后依次穿过纹理图像的较高行。最后一个元素对应于纹理图像的右上角。

---

接下来对采样器作说明：

GPU 对纹理的读取/采样是硬件层面完成的，现代 GPU 包含专门的硬件单元，一般称为纹理单元 (Texture Units) 或纹理采样器 (Texture Samplers)，用于执行纹理读取和采样操作。

纹理单元通过读取采样参数来决定具体的采样行为。采样参数属于纹理目标，也就是说每个纹理目标会存储属于它相关的采样参数。

采样参数在CPU侧通过`glTexParameteri`配置，随后复制给GPU，主要包括：

- 纹理过滤 (Filtering)：指定如何处理纹理坐标落在像素之间的情况，因为像素是离散的，但纹理坐标是连续的。

- 纹理环绕 (Wrapping)：定义纹理坐标超出[0,1]范围时的行为

- Mipmapping：采样的纹理坐标可能因Mipmap级别导致不一样

- 各向异性过滤 (Anisotropic Filtering)：在视角倾斜时，提高纹理的清晰度，减少模糊。

对于着色器程序，采样器实际上是表示的含义是纹理单元x的索引，告诉着色器要处理的纹理数据（及采样参数）在`GL_TEXTUREx`中。这一点可以在CPU侧配置采样器值做印证：采样器变量在CPU侧被`glUniform1i`作为`GLuint`类型的 uniform 值传到着色器，由`sampler2D`（或类似类型）接收。

额外地，OpenGL 3.3 引入了 Sampler Objects，此举将采样参数从纹理对象中分离出来，作为对象绑定到不同的纹理单元。这意味着多个纹理单元可能共享同一个 Sampler Objects。

### 主序与左右乘

矩阵的主序可能不同，因此在谈及坐标乘矩阵的时候，可能会在坐标的左边或者右边乘，这是要具体情况具体分析的。

> 左行右列，即左侧矩阵的**行**和右侧矩阵的**列**进行点积

GLM使用的是列主序，这与 OpenGL 的默认矩阵约定一致。在其中的 vec 均被视作列向量处理。那自然要把坐标 vec 写在右侧，自然而然地，对于坐标$x$乘以 MVP 实际上是$P * (V * (M * x))$。由于矩阵乘法有结合律，所以等价地有$(P*V*M)*x$

至于矩阵本身的存储方式是列主序还是行主序存储，只是影响获取第$i$行$j$列元素的方式，并不影响实际计算。

换句话说：列主序的矩阵乘法是从右到左应用的，行主序的矩阵乘法是从左到右应用的。

## 3D操作速览

> glm已为你完成所需的大部分工作（快说谢谢）

### 相机/视点

1. 如果相机在坐标系原点且固定朝一个轴向，计算效果总是最易理解的和简单的

2. 变换坐标系并不会导致物体之间的空间关系错乱（比如距离变化）

基于上述两点，不难发现这就是乘MV矩阵的意义。不过既然常提的是MVP，就一块说了：

- 世界坐标 * M = 模型空间坐标
- 模型空间坐标 * V = 摄像机空间坐标
- 摄像机空间坐标 * P = 裁剪空间坐标

也就是说顶点坐标变换公式如下：

$$
V_{clip}=M_{projection}⋅M_{view}⋅M_{model}⋅V_{local}
$$

模型矩阵是对物体坐标系原点应用的，如果是硬编码的物体，通常在世界坐标中心，我们默认物体坐标系原点和世界坐标系原点，这种情况下需要在模型矩阵里将模型移动到场景对应位置

视图矩阵的话则是再次变换坐标系，进入相机坐标系（或者可以理解为移动场景使相机位于世界坐标系原点）。摄像机<mark>通常朝向负 Z 轴</mark>，至少在OpenGL里，视图坐标系正Z轴是从屏幕向外延伸，指向观察者。不过世界坐标系在应用视图矩阵之前也是正Z轴向屏幕外延伸。

就管线而言，在乘MVP得到裁剪空间坐标之后，会进行透视除法，转换为规范化设备坐标 (NDC)，最后映射到屏幕上的像素坐标。透视除法和视口变换通常在片元着色器之前由GPU硬件自动完成。

额外地，三维坐标乘MVP得到的是齐次坐标（其实乘M得到的已经是齐次坐标了）

---

此外一些注意点：

- 透视投影会把视锥体（一个锥台）映射到裁剪空间的长方体中，在这个过程中，透视矩阵将这个透视关系编码到 w 分量中。物体到相机的距离越远，w值越小。

- 裁剪空间的xyz构成立方体，其范围是 -w ≤ x, y, z ≤ w

- 裁剪空间经过透视除法变为规范化的立方体，范围为 -1 ≤ x, y, z ≤ 1

- 透视除法发生在裁剪空间坐标经过裁剪之后，在光栅化阶段之前，透视除法和裁剪是OpenGl自动完成的。

- 正交投影矩阵的构造方式保证算出来的裁剪空间的齐次坐标w是1，相当于透视除法没用（正交视图本身也不需要透视除法）

### FPS风格移动相机

基于上一小节，移动摄像机其实就是有针对性地修改V矩阵并应用于渲染。也就是说，我们先根据输入移动摄像机（世界坐标系下），然后再正常的乘MVP渲染就行。此举涉及到：输入同相机Transform的映射。

根据上一小节可知，相机的Forward向量总是朝向负Z轴。这意味着其实设置摄像机指向的时候，要要加符号，`cameraPos - cameraTarget`才是相机朝向物体（负Z朝向物体）。

描述一个相机的位置：

- 世界空间坐标

- 相机的XYZ方向（世界坐标系）

当然对于现代引擎来说，还会有更多的参数，例如：

- FOV

- 远裁面和近裁面

- 后处理堆栈

不过在LearnOpenGL里并不需要这么细致的Camera类，这些东西放到引擎板块去讲吧。

<mark>LookAt矩阵</mark>：在世界坐标系下，将相机坐标移动到指定位置，并且朝向指定物体。这个过程需要给定相机位置和朝向物体位置，以及相机的Up方向。因为相机朝向（Forward方向）已知，只需再给出Up方向就可以算出Right方向。有必要注意的是，使用glm库提供的`glm::lookAt`会导致将相机的负前向向量指向目标物体。

然后根据输入（一般是WASD）移动相机即可了，只需要Forward和Right就行，在他们的正负方向移动。不过为了平滑移动，需要记录帧时间计算deltaTime，乘以一个速度，使移动更加平滑。

视角的移动则根据鼠标的移动向量处理，这通过每帧记录鼠标的位置获取差值来得到，是一个二维向量。它要分解为水平和垂直分量，垂直分量影响Pitch，水平分量影响Yaw。大部分情况下是在相机坐标系。如果要操作欧拉角的话，Pitch和Yaw就够了，无需操作Roll。

与相机移动不同，视角移动无需乘速度，但是要乘一个sensitivity（灵敏度）。

```cpp
float xoffset = xpos - lastX;
float yoffset = lastY - ypos; // 注意这里是相反的，因为y坐标是从底部往顶部依次增大的
lastX = xpos;
lastY = ypos;

float sensitivity = 0.05f;
xoffset *= sensitivity;
yoffset *= sensitivity;

yaw += xoffset;
pitch += yoffset;
// 限制Pitch，但不限制Raw，水平方向是可以360°旋转视角的
if(pitch > 89.0f) pitch =  89.0f;
if(pitch < -89.0f) pitch = -89.0f;
```

这下可以得到一个移动的方向，然后就是要把摄像机的Forward移动到这个方向上，这依靠三角函数：

缩放的话则是通过修改FOV实现的，FOV变小产生放大效果，反之同理。

> 上述操作涉及旋转的，可以上层使用欧拉角操作，但建议底层统一使用四元数以避免万向节死锁。glm提供了四元数的类型与操作函数。

## 模型加载速览

> Assmip

这也没什么可说的，待会补一下CLion配置Assmip的方式。总之容易出现奇奇怪怪的问题。因为用CLion打开工程并编译Assmip的源码还是很奇怪的，会出很多问题。

# LearnOpenGL CN

> 这部分内容的定位其实就是对阅读完原文的补充总结，不能替代原文

## 前面几章

前面几章也没什么可提的，简单说一下他的代码结构与思路

### 顶点数据与属性

他教程里数据直接使用了`float[]`存贮，这个结构和常见的`.fbx`十分相似。这一部分主要为了明晰，如何配置相关设置指导GPU读取解析字节流为指定的数据。或者说，如何配置和使用VAO作为“模版”来读取数据。

```cpp
unsinged int VAO;
glGenVertexArrays(1, &VAO);
glBindVertexArray(VAO);
// 设置VBO等数据
// 设置VAO有哪些属性
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);    // 启用所需的属性
glBindVertexArray(0);    // 解绑VAO，该VAO配置完毕
```

或者说整个流程是这样的：

```
// 数据准备
Generate VAO
BindVAO
Generate VBO's
BindVBO's
Specify vertex attributes

// 绘制
Bind VAO
Draw 
```

### Texture Import and Apply

> 关于Texture的知识参阅前文"Texture in OpenGL"

`unsigned char`是8位，表示$[0,255]$的值，据说使用 `unsigned char` 是图像处理中的标准做法。但是OpenGL生成Texture对象的时候不一定只能处理8位数据，毕竟PNG也有16 位/通道的格式。

```cpp
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
```

典型格式是这样，那`GL_UNSIGNED_BYTE`就表明后面要处理 8 位无符号整数，类似地使用`GL_UNSIGNED_SHORT`就可以处理 16 位/通道的图像数据

从外部文件到OpenGL可处理的Texture的大致流程是：

1. 声明`GLuint`并通过`glGenTextures`和`glBindTexture`设置好上下文

2. 通过`glTexParameteri`指定相关参数

3. 加载图像到内存（可能要依赖第三方库去解析图像文件）

4. 通过`glTexImage2D`生成OpenGL的纹理

5. 释放资源（如果需要）

---

纹理的应用依靠顶点属性，需要为顶点设置纹理坐标（俗称UV），并调整着色器使之作为`vec2`类型变量被接收，通常情况下不需要在顶点着色器做处理，直接传递给片元着色器即可。

> 这段文本聚焦于采样器

### 简单光照与乘法

光照最终效果很会由多部分效果组合而成，在最简单的Phong模型中由三部分构成

- 环境光照(Ambient Lighting)：各种环境光自四面八方均匀地为物体铺色。

- 漫反射光照(Diffuse Lighting)：光源对物体的方向性影响，物体的某一部分越是正对着光源，它就会越亮。

- 镜面光照(Specular Lighting)：受物体表面反射特性决定，反射强度越强，其颜色上就越靠近光的颜色而非物体的颜色。

其实相当于是：

$$
Result = (I_a+I_d+I_s)*ObjectColor
$$

各个强度的构成如下：

$$
I_a = k_a \cdot I_{light}
$$

其中$k_a$是物体的环境反射系数

$$
I_d = k_d \cdot I_{light} \cdot \max(N \cdot L, 0) 
$$

其中

- $k_d$是物体的漫反射系数

- N是法线L是光源方向

$$
I_s = k_s \cdot I_{light} \cdot \max(R \cdot V, 0)^{\alpha}
$$

其中

- $k_s$：物体的镜面反射系数

- R：从表面点反射光线的方向的单位向量。

- V：从表面点指向观察者（摄像机）的单位向量。

- α：物体的光泽度（shininess）系数，控制镜面高光的锐度，值越大，高光越小且越集中

强度与物体的颜色是乘法，而不同光照成分的结果却是加法，要理解其中的意义：

- 加法：现实世界中光照的叠加效应（光源的叠加）

- 乘法：光照效果与物体的基本颜色结合（颜色的混合）

当然，谈及系数的时候，我们总能把它们放在贴图中进行采样，而不仅仅是作为一个参数硬编码或者由程序控制。

> Gouraud着色与Phong着色的区别，实际上应用公式阶段的不同（他们的公式是一样的），前者是逐顶点（在顶点阶段）应用Phong的公式，而Phong是逐片元着色的

### flat修饰符

官方教程中顶点着色器传递给片元着色器的，如下：

```glsl
FragPos = vec3(model * vec4(vertexPos, 1.0));
```

这个计算的是顶点的世界空间坐标，其实传递出去的和片元着色器接受的并不一致，光栅化的时候会对这个插值得到真正的片元坐标。

不过`fragPos` 本身并不是必须传递给片元着色器的，还可以由屏幕空间通过深度图重建世界坐标得到。但是一般只有在后处理时候会这样，因为效率和精度都低。正常流程中是直接从顶点着色器传更为妥当。

---

接着引申出插值问题，有些`out`变量会被插值：

<mark>flat</mark>修饰符：如果表示不应被插值的属性，可用`flat`修饰声明

注：`gl_Position`这种内置变量有单独的特殊处理方式，，不会被普通插值。

### 材质

材质的本质相当于是对一个光照模型的参数化，把一套着色模型（一组公式）中可调节的参数暴露出来由外部控制，可以视之为封装为一个函数。一个材质实例相当于一组参数预设。

对于给定的光照模型，教程使用结构体容纳相关描述一个材质的参数。

示例代码中的顶点着色器是这样的：

```glsl
FragPos = vec3(model * vec4(aPos, 1.0));
Normal = mat3(transpose(inverse(model))) * aNormal;  

gl_Position = projection * view * vec4(FragPos, 1.0);
```

---

额外地，据我观察ShaderLab似乎并非使用类似的方法将整体包装为一个结构体。

据查证，HLSL文档[Microsoft Learn - 在 Direct3D 9 中编写 HLSL 着色器 -  统一着色器输入](https://learn.microsoft.com/zh-cn/windows/win32/direct3dhlsl/dx-graphics-hlsl-writing-shaders-9#uniform-shader-inputs)指出，可以通过两种方法指定 uniform 数据：

1. 最常见的方法是声明全局变量并在着色器中使用它们， 在着色器中使用全局变量将导致将变量添加到该着色器所需的统一变量列表中

2. 第二种方法是将顶级着色器函数的输入参数标记为统一。 此标记指定应将给定变量添加到统一变量列表中。

另一方面Unity文档指出：Cg/HLSL 还可以接受 uniform 关键字，但该关键字并不是必需的。上述内容隐晦地表示了相关变量应当是uniform的。

### 光源类型

<mark>平行光（定向光）</mark>：与光源位置无关，一般假设光源处于**无限**远处。太阳是一个典型的平行光，不过要是渲染宇宙中的星系，可能此时太阳就不是平行光（因为不处于无限远）

<mark>点光源</mark>：特征是朝四面八方发光但光线会随着距离逐渐衰减。把衰减作为强度的系数，对光照强度进行控制，常见公式如下：

$$
F_{att}=\frac{1.0}{K_c+K_l * d + K_q * d^2}
$$

三个K都是常数，其中$K_c$大部分情况都是1.0（为了使分母不小于分子），在稍近的范围内看起来是线性衰减，但是超过某个阈值后二次项的变化更为显著，使得远距离衰减更为迅速，相关内容可以在[ORGE Wiki - Point Light Attenuation](https://wiki.ogre3d.org/tiki-index.php?page=-Point+Light+Attenuation)查得

<mark>聚光</mark>：像是一种限定了范围（锥体）的点光源。但是为了定义聚光，额外至少需要一个切光角$\phi$（Cutoff Angle）参数，在这种情况下与光源朝向夹角$\theta$若满足$\theta \le \phi$则可以被光照到。

接收到的光强度和片元到光源距离有关系，但是通常也和<u>光源方向和光源到片元方向之间的夹角</u>有关。如果考虑到这一点，实际上会划分为内锥角$\phi_i$和外锥角$\phi_o$，设shading point与光源朝向夹角为$\theta$，则：

- $\theta \le \phi_i$：光线强度保持不变或衰减较小

- $\phi_i < \theta \le \phi_o$：光线强度会逐渐衰减，直到光线完全消失

在三维空间上照射到平面，看上去相当于有一个环状的衰减区域。

这种情况下使用的公式如下，其中：

$$
I = \frac{\cos(\theta) - \cos(\phi_o)}{\cos(\phi_i) - \cos(\phi_o)}
$$

因为计算$\theta$实际上是`dot(fragDir2Light, normailze(-lightDir))`，获得的是夹角的预先，所以就直接在公式中都使用余弦值比较了，毕竟反余弦计算开销过大。

相当于最终强度是这样构成的：

```cpp
FinalIntensity = Attenuation * SpotlightFactor * LightIntensity
```

对于多光源情况，会结合UBO进行绘制，详见[OpenGL Basic Part2](./2 OpenGL Basic Part2.md)的UBO一节。

### 其他机制细节

> BufferObject、绘制指令与Shader的联系

分为两部分，一部分是配置数据，另一部分是使用

配置数据的大前提是先指定VAO，也就是有哪些属性（通过`glEnableVertexAttribArray`与`glDisableVertexAttribArray`启用\禁用某个属性）

然后就是`glBindBuffer`和`glBufferData`，绑定VBO或者EBO什么的然后填入数据，这两个操作一般是一块出现的。

配置完数据最好解绑一下，不然后续操作很有可能处理的是当前的数据。

打个比方，显存相当于是操作台，VAO是某个操作的数据清单，告诉OpenGL什么数据要去操作台的哪里取、VBO什么的相当于具体的数据，摆在显存里。然后在渲染循环里给出具体的指令去做哪些操作，让这些操作会涉及到哪些数据，就给他相应的VAO，相关指令就会自己去取。

绘制指令 (`glDrawArrays` 或 `glDrawElements`) 的直接依据是当前绑定的 VAO

1. shader中`(loaction = x)`是和VAO指定的顶点属性的索引一致的，设置几就会取哪个序号对应的数据

2. 解绑VAO不会删除显存中对应的数据，因为绑定本身相当于一种引用

---

1. UV是顶点属性配置的，作为`vec2`被着色器读取，通常要被vs输出到fs，在fs里使用
2. 使用`glEnable(GL_DEPTH_TEST);`开启深度测试，不过OpenGL并无帧的概念，所以需要在合适的时机（一般是帧开始是）通过`glClear`手动清理掉前一帧的深度信息

# Beyond LearnOpenGL

> 在LearnOpenGL之外，谈论一些他没提到的东西

## More of VAO

我确信看完LearnOpenGL之后关于VAO、VBO和EBO之间的关系仍不会很清晰，所以我翻了翻一些论坛中的讨论：[StackExchange - Understanding VAO and VBO](https://computergraphics.stackexchange.com/questions/10332/understanding-vao-and-vbo)

> 自 VAO 绑定的那时起任何后续的顶点属性调用都将存储在该 VAO 内

OpenGL的Object不存储调用，只存储状态，这可以理解为只Implement了获取数据的Interface，至于什么调用会通过哪些接口以及去做什么，Object是不知道的。

或者从另一角度理解，基于OpenGL是一个巨大的状态机这个事实，在绑定了VAO之后，VAO会把这个状态机里他关注的state的状态映射到自身，进而这可以理解为，VAO本质是OpenGL状态机中一些状态<mark>VAO解绑时那一时刻状态的“切片”</mark>（在一条时间轴上，OpenGL状态机的状态们总是在变化）。

此外，兼容模式下 0 号 VAO 会被当做默认的对象，但是核心模式下 0 号 VAO 不被视作对象，此时不应当调用任何修改 VAO 状态的函数。

当然，千言万语不如去看一眼官方文档：[OpenGL 对象](https://www.khronos.org/opengl/wiki/OpenGL_Object)

## More of VBO

> 这里我认为需要关注一下VBO的数据位置及传输时机

就时机而言，并不是绝对的，因为传输到GPU的过程是异步的，而这个异步过程会受到多个因素的影响（被推迟）。

进行绘制调用并不要求此绘制命令所依赖的任何缓冲区 DMA 当时都已完成。它仅要求这些 DMA 在 GPU <u>实际执行该绘制命令之前</u>发生。实现可以通过在调用中阻塞直到 DMA 完成来实现这一点。

OpenGL规范的唯一要求是相关实现的外在表现符合规范所述，这意味着OpenGL是一个巨大的黑盒模型

## VAO 与 VBO 的关系

> 参考：[khronos - Vertex_Specification](https://www.khronos.org/opengl/wiki/Vertex_Specification#Vertex_Array_Object)的 Vertex Buffer Object 小节

VAO 确实存有对于 VBO 的引用，但是需要了解更新时机和方式。

`GL_ARRAY_BUFFER`并非 VAO 存有的 State。绑定 VBO 到`GL_ARRAY_BUFFER`并不会引起 VAO 更新，只有在调用`glVertexAttribPointer`的时候才会读取全局变量`GL_ARRAY_BUFFER`上绑定的 VBO 并存储进 VAO。

这意味着 VAO 可以通过顶点属性间接存有多个 VBO 的引用，在一次绘制中绘制多个 VBO，或者说是顶点属性和 VBO 关联。

额外地，将场景中的静态物体的数据合成一个 VBO 上传，实际上就是静态合批。

对于那些因为一些因素不能合批的物体来说，通常一个物体会对应一个 VAO ，因为如果共用 VAO，在运行时对 VAO 进行绑定更新依旧存在不可忽略的开销。在实际的开发中，VAO 的数目可能成千上万。

VAO 数目增多的问题主要集中在每个 VAO 对内存的占用，以及状态切换的开销。

## Texture与PBO

参考：[songho - OpenGL 像素缓冲区对象 (PBO)](https://www.songho.ca/opengl/gl_pbo.html)

PBO（Pixel Buffer Object，像素缓冲对象）是OpenGL中的一种缓冲区对象，仅用于执行像素传输。PBO 通过允许异步 DMA 传输来减少 CPU 的负担，是一种性能优化。

> 官方文档称之为是一种改善应用程序和 OpenGL 之间异步行为的方法

像素传输分为两类：上传（像素解包）和下载（像素打包）。上传是指数据从属于 CPU 的存储空间到属于 GPU 的存储空间。

PBO 做像素传输的操作依赖于两个访问载点：对于上传的操作，会访问绑定到`GL_PIXEL_UNPACK_BUFFER`的缓冲对象，而对于下载则访问绑定到`GL_PIXEL_PACK_BUFFER`的缓冲对象。

就使用而言，含有 PBO 的代码主要是针对`glReadPixels`、`glTexImage2D`或`glTexSubImage2D`做补充。

在未含有 PBO 的流程中，使用`glTexImage2D`或`glTexSubImage2D`将像素数据从CPU上传到纹理。

> 还记得`glTexImage2D`吗？使用它将内存中的字节流构建为 OpenGL 可以识别的纹理对象，并配置好纹理属性，以及上传到 GPU。如果不指定字节数据，则只是分配纹理内存（显存）但是不发生上传

```cpp
// 假设 width 和 height 是纹理的尺寸
int width = 512;
int height = 512;

// 创建 PBO
GLuint pbo;
glGenBuffers(1, &pbo);
glBindBuffer(GL_PIXEL_UNPACK_BUFFER, pbo);
glBufferData(GL_PIXEL_UNPACK_BUFFER, width * height * 4, nullptr, GL_STREAM_DRAW);

// 填充像素数据
GLubyte* pixels = new GLubyte[width * height * 4]; // RGBA 格式
for (int i = 0; i < width * height * 4; i++) {
    pixels[i] = rand() % 256; // 随机生成像素数据
}

// 将像素数据上传到 PBO
glBindBuffer(GL_PIXEL_UNPACK_BUFFER, pbo);
glBufferSubData(GL_PIXEL_UNPACK_BUFFER, 0, width * height * 4, pixels);

// 创建纹理
GLuint texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);

// 从 PBO 上传数据到纹理
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, 0);

// 解绑与清理
glBindBuffer(GL_PIXEL_UNPACK_BUFFER, 0);
delete[] pixels;
```

使用 PBO 的代码并不会避开使用`glTexImage2D`，而是为该函数内部的 `data` 参数为 `0`，表示从当前绑定的 PBO 中读取数据。

这时候可能有人会好奇，上面不是说参数为 0 表示不上传数据只分配空间吗？在参数为 0 的情况下，会查看`GL_PIXEL_UNPACK_BUFFER`是否绑定有数据，如果它绑定了一个有效的PBO，并且PBO中有数据，OpenGL会从PBO中读取数据并上传到纹理，否则仅分配显存空间。

那么 PBO 究竟为客户端干了什么事情而提高效率呢？上面说了是异步 DMA。截取上述参考链接的一张图：

![pbo在纹理传输的作用](D:\CGNote\Graphics%20API\OpenGL\img\with%20or%20without%20pbo.png)

在没有 PBO 的时候数据流动是这样的：

1. 纹理数据字节流（内存）：源自于从磁盘等介质加载图像数据，可能还要经过一些数据处理变成 OpenGL 可识别的格式

2. OpenGL能识别的纹理数据（内存）：调用 `glTexImage2D` 或 `glTexSubImage2D` 等函数，OpenGL 驱动程序会将纹理数据从 CPU 内存复制到驱动程序管理的内存区域。

3. 纹理对象的区域（显存）：OpenGL 驱动程序会将纹理数据从驱动程序管理的内存区域复制到 GPU 的显存中。

在存在 PBO 的时候数据流动类似上面，但是第二步时 `glTexImage2D`并不会将数据复制到驱动程序管理的内存，直接在第三步异步 DMA 到显存。相当于少了一次内存拷贝（从CPU控制的内存拷贝到驱动程序控制的内存）

此外网上的资料表明 PBO 一定会使`glTexImage2D`立即返回（似乎进入命令缓冲的 OpenGL 指令并非所有都是立即返回的，与实现有关）

---

当然，`glTexImage2D`上传行为是异步的，这个放到[4 异步与同步](./4 异步与同步.md)中细说。

## Texture与Buffer Object

纹理和缓冲区对象是两种不同的数据存储和访问机制，二者的数据都位于显存，但它们的设计目的和使用方式有所不同。

|         | 纹理  | 缓冲对象 |
|:-------:|:---:|:----:|
| GPU访问方式 | 采样器 | 绑定点  |
| 格式      | 固定  | 任意   |
| 插值与过滤   | 支持  | 不支持  |

注：采样器是 GPU 访问纹理的方式，CPU 也可以通过`glGetTexImage`读取纹理进行访问。不过在大多数情况下 IO between CPU and GPU 是有不可忽视的开销的，性能较低。

---

纹理或者缓冲对象都可以存放 GPU 计算的结果，对于写入到纹理来说：

- 在计算着色器中通过 `imageStore` 将数据写入纹理

- 使用 FBO 将计算结果渲染到纹理

如果是存储到缓冲对象的话：

- 使用 SSBO 存放结果，在被传入的情况下着色器（包括但不限于计算着色器）可以直接访问，适合频繁读写或存储大量数据

注：帧缓冲是用于管理渲染目标的对象，它本身并不直接存储数据，而是通过附件（如纹理或渲染缓冲对象）来存储数据，所以帧缓冲不直接作为作为存储计算结果的“缓冲”

---

此外 Image Load/Store 与采样器的比较如下：

|        | 采样器访问        | Image Load/Store      |
|:------:|:------------:|:---------------------:|
| 访问方式   | 纹理坐标（uv 坐标）  | 像素坐标（整数坐标，`x`行`y`列这种） |
| 插值与过滤  | 支持           | 不支持                   |
| 读写操作   | 只读           | 读写                    |
| 性能优化   | 针对图形渲染优化     | 针对随机访问优化              |
| 使用场景   | 图形渲染（贴图、材质等） | 通用计算（GPGPU）           |
| 同步与一致性 | 自动管理         | 需要手动管理（内存屏障）          |

## Texture与TBO

TBO（Texture Buffer Object） 允许将缓冲区对象（Buffer Object）作为纹理使用，该机制结合了缓冲区对象和纹理的优点，适用于需要处理大量数据且需要随机访问的场景。

- TBO 有大小限制，可以使用``GL_MAX_TEXTURE_BUFFER_SIZE``查询。

- TBO 不支持 mipmapping 和纹理过滤，这意味着无法使用 `glTexParameteri` 来设置过滤参数

- TBO 可以被采样器采样，在着色器中声明`samplerBuffer`类型的采样器，并通过`texelFetch`按整数坐标访问，这意味着不能自动被插值，但是可以手动实现插值算法

- 在一些场合，更推荐使用 SSBO 替代 TBO

原本正常使用 texture 是使用`glBindTexture`给绑定到`GL_TEXTURE_2D`这样的目标上，但是使用 TBO 的话，是绑定到`GL_TEXTURE_BUFFER`

```cpp
// 创建和填充 TBO 内容
GLuint buffer;
glGenBuffers(1, &buffer);
glBindBuffer(GL_TEXTURE_BUFFER, buffer);
glBufferData(GL_TEXTURE_BUFFER, size, data, GL_STATIC_DRAW);
// 创建纹理缓冲区对象
GLuint texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_BUFFER, texture);
// 缓冲区对象与纹理对象关联，并指定内部格式（此处是GL_R32F）
glTexBuffer(GL_TEXTURE_BUFFER, GL_R32F, buffer);
```

`glBufferData`里传入的数据`data`是要和底下`glTexBuffer`指定的格式对应的。他必须是一个一维数组，这意味着对于二维数据需要自己手动展平，并在访问时自行计算偏移量。那假如指定的数据格式是`GL_RG32F`那传入的数据就得按照 RGRGRGRGRGRG... 的顺序排列

注意：`GL_TEXTURE_BUFFER` 既可以被 `glBindBuffer` 绑定，也可以被 `glBindTexture` 绑定，但这两种绑定方式的作用和目的是不同的。