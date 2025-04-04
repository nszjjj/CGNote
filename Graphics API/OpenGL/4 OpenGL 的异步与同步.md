# 异步与同步

## OpenGL 操作的异步性

[Khronos OpenGL Wiki - 同步](https://www.khronos.org/opengl/wiki/Synchronization)

OpenGL 是一个巨大的黑盒。其 API 在使用上是同步的，但是内部执行是异步的。因为 OpenGL 本身并没有像 Vulkan 或 DirectX 12 那样的原生命令缓冲区机制。 OpenGL 的渲染命令是直接提交给驱动程序的，驱动程序会立即（或尽可能快地）执行这些命令。

从概念上讲，GPU 有一个称为“命令队列”的东西。这是 OpenGL 驱动程序根据用户要求编写的命令列表。几乎每个 OpenGL 函数都会映射到一个或多个将添加到命令队列的命令。任何放置在命令队​​列中的命令都将被 GPU 读取并执行。

GPU 的命令队列源自于 CPU 侧的驱动程序从命令缓冲区（OpenGL 没有显式的命令缓冲区机制，但是行为表现上是有的）取出并提交到 GPU。大部分 OpenGL 指令都会先进入缓冲区等候驱动程序取出，驱动程序可能会根据缓冲区内的指令状态做一些诸如合并等操作。每个 OpenGL 上下文都有一个独立的命令缓冲区

OpenGL 开发者无法通过某些途径获取 GPU 命令队列的状态，例如剩余多少指令未被执行。因为 OpenGL 的设计目标是提供高层次的抽象，而 GPU 命令队列的状态属于底层驱动和硬件管理的细节。

注：NV 硬件有可能支持`NV_command_list`扩展，允许将 OpenGL 命令记录到命令列表中，然后一次性执行。

事实上官方文档直接说了：OpenGL 渲染命令被假定为异步执行。如果调用任何`glDraw*`函数来启动渲染，则根本不能保证在调用返回时渲染已经完成。

## 获知 GPU 工作状态的机制

有两种途径可以有限地获取 GPU 的一些工作状态，分别是：

- OpenGL API

- 扩展或特定驱动功能

例如：

- `glFinish`会阻塞调用线程，直到所有先前发出的 OpenGL 命令（包括那些尚未发送到 GPU 的命令）都执行完毕

- `glFlush`会阻塞调用线程，并强制 OpenGL 驱动程序将所有尚未发送到 OpenGL 服务器（通常是 GPU）的命令从命令缓冲区（概念上是有的命令缓冲区的）发送过去，且确保顺序和调用顺序一致

- `GL_NV_performance_monitor`或者`GL_AMD_performance_monitor`能够访问 GPU 的硬件性能计数器，例如纹理缓存命中率、算术运算吞吐量等。

注：`glFlush`针对的情况是，一些 GPU 驱动程序不会将发出的命令发送到硬件，除非已经积累了一定数量的命令。

## 交换缓冲

OpenGL 本身并不直接提供双缓冲机制，但是提供了双缓冲的概念。只是他本身并不管理窗口或缓冲区交换，而是依赖于窗口系统（如 GLFW、GLUT、SDL 等）来实现双缓冲。

例如 GLFW 在 Window (WGL) 下会调用`SwapBuffers`函数，而在 Linux (GLX) 下会调用`glXSwapBuffers`。

```cpp
while (!glfwWindowShouldClose(window))
{
    processInput(window);

    glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    glfwSwapBuffers(window);
    glfwPollEvents();
}
```

LearnOpenGL 教程最开始给出这样一段渲染循环，这不由得让人思考到一个问题：`glfwSwapBuffers`的时候所有绘制指令一定执行完成了吗？

实际上交换缓冲区的命令意味着<mark>将一个“交换缓冲区”的命令插入到 GPU 的命令队列中</mark>，还是得等到他执行到这个命令才会实际地交换缓冲区。

相当于缓冲区交换命令本身是一种隐式同步机制。它确保在交换缓冲区之前，所有先前的绘制命令都已经完成。这种同步是由 GPU 硬件和驱动程序自动处理的，开发者无需显式干预。

额外地，如果启用了垂直同步（VSync），缓冲区交换命令会进一步等待显示器的垂直回扫信号（V-Blank）。但是开发者无需担心其他细节，照常使用`glfwSwapBuffers`即可，GLFW 和 OpenGL 驱动会负责和 VSync 打交道。

## 状态管理

OpenGL很多设置是针对Context的，比如说背面剔除等操作，启用/禁用会影响全局。

那假如说有这样一种操作步骤：

1. 启用某个全局设置C

2. 提交物体A绘制调用

3. 禁用某个全局设置C

4. 提交物体B绘制调用

有些人可能会担心：如果在提交完绘制调用就禁用全局设置C，导致状态改变，那在实际绘制A的时候，拿到错误的状态怎么办？

实际上无需担心，即使调用的 OpenGL 命令是看起来是 CPU 侧执行的（例如启用/禁用某种设置），也会先被放入命令缓冲区，等待驱动程序取出并执行

注：存在某些指令是立即执行的，例如：

- `glGet*` 系列函数（如 `glGetIntegerv`）：这些函数会立即从 OpenGL 状态机中读取数据并返回给 CPU。
- `glReadPixels`：从帧缓冲区中读取像素数据到 CPU 内存。

---

关于 OpenGL 状态的存放位置：

OpenGL 状态主要存在于 OpenGL Context 中，它包含了 OpenGL 运行所需的所有状态信息。大部分 OpenGL 状态是由 OpenGL 驱动程序管理的，存储在驱动程序控制的内存中，这部分内存可能位于 CPU 的系统内存中，也可能位于 GPU 的显存中，或者两者都有。这和硬件架构也有关系。

## Barrier

在使用着色器写入 SSBO 后，需要使用`glMemoryBarrier`确保数据同步。

Barrier的主要作用是让所有参与线程在继续执行前都到达同一个点，确保数据一致性。具体来说是确保在调用`glMemoryBarrier`之前的所有指定类型的内存操作（如写入、读取等）在调用之后对其他操作可见。

对其他操作可见的意思是，某个操作（如写入数据到缓冲区或纹理）的结果能够被后续的操作（如读取数据或渲染）正确访问和使用。一个最典型的例子是这样的：

```cpp
// 计算着色器写入数据到SSBO
glDispatchCompute(1, 1, 1);
// 这一句执行完之后对SSBO的写入操作都应当执行完
glMemoryBarrier(GL_SHADER_STORAGE_BARRIER_BIT);
// 渲染着色器读取SSBO中的数据
glDrawArrays(GL_TRIANGLES, 0, 3);
```

因为`glDispatchCompute`与`glDrawArrays`都是异步的，如果没有 barrier 可能在`glDrawArrays`时用到 SSBO 的数据时 SSBO 尚未写入。

而且可能不仅仅是未写入的问题，如果存在多级缓存，计算着色器写入的数据可能暂时存储在缓存中，而没有立即写回主内存，要是不使用 barrier 同步，可能会从缓存中读取旧数据。

一般需要 barrier 的情况有：

- SSBO 或者纹理被写入，随后被其他地方（通常是着色器）读取

- 原子操作，如 Compute Shader 中使用`atomicAdd`

## 线程间同步

### Fence

[Khronos OpenGL Wiki - 同步对象](https://www.khronos.org/opengl/wiki/Sync_Object)

Sync Object（同步对象）就是 Fence Object（栅栏对象）具有相同的含义。

可以在 CPU 侧将 Fence Object 塞到命令流中，然后等 GPU 触发。

同步对象不遵循标准的 [OpenGL 对象模型](https://www.khronos.org/opengl/wiki/OpenGL_Object)，例如标准的 OpenGL 对象使用`GLuint`类型的名字，但是同步对象的名字类型是指向一个不透明对象的指针：

```cpp
typedef struct __GLsync* GLsync;
```

一个典型的同步对象使用范例是这样的：

```cpp
// 绘制操作
glClear(GL_COLOR_BUFFER_BIT);
DrawObject();

// 创建 Sync Object，同步条件是等待 GPU 完成所有之前的命令。
GLsync sync = glFenceSync(GL_SYNC_GPU_COMMANDS_COMPLETE, 0);

// 等待 Sync Object 被触发
GLenum waitReturn = glClientWaitSync(sync, 0, 1000000000); // 等待 1 秒
if (waitReturn == GL_CONDITION_SATISFIED || waitReturn == GL_ALREADY_SIGNALED) {
    // GPU 已完成绘制操作，可以安全读取像素数据
    glReadPixels(0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, pixels);
} else {
    // 处理超时或错误
}

// 删除 Sync Object
glDeleteSync(sync);
```

同步对象一经[glFenceSync](https://www.khronos.org/opengl/wiki/GLAPI/glFenceSync "GLAPI/glFenceSync")创建就会被添加到命令流中，该函数的第二个参数始终为0，保留的目的是未来存在扩展的可能。

同步对象是属于某个上下文的，因此线程间同步需要确保都属于同一个上下文。

---

该机制主要用于这样几方面：

1. 确保资源的可用性：在使用 GPU 运算的结果时应当确保 GPU 执行完毕（（如 `glReadPixels` 或 `glMapBuffer`）

2. 多线程渲染：确保一个线程的渲染操作在另一个线程开始之前完成

3. 避免竞争条件：确保某些操作按正确的顺序执行‘

### Atomic Counters

原子计数器由`GL_ARB_atomic_counters`扩展引入，并成为 OpenGL 4.2 的核心功能。该功能主要是在 GPU 上进行原子操作，但是 CPU 也可以读取其值。它可以实现简易的互斥锁或者信号量机制，但这并非它的全部功能，他也可以对一些渲染数据进行统计和可视化，例如：

[Geeks 3D - OpenGL 4.2 Atomic Counter Demo: Rendering Order of Fragments](https://www.geeks3d.com/20120309/opengl-4-2-atomic-counter-demo-rendering-order-of-fragments/)

## 资源传输中的异步

调用如`glBufferData`或`glTexImage2D`时，数据上传到GPU是异步的，相当于还是进入到驱动程序的命令队列。

此处似乎有一个需要关心的地方：如何确保控制资源上传指令后面用到相关资源的指令执行时，相关资源一定已经上传到显存的目标位置。

就结果而言肯定是不用担心的，OpenGL 规范要求每个后续命令的行为都好像所有先前的命令已经完成并且它们的内容是可见的。

就实现原理而言，OpenGL 驱动程序通常会隐式地处理同步。当一个命令依赖于前一个命令的结果时，实现会阻止这两个命令之间的任何执行重叠。相当于 OpenGL 保证用的时候一定是可用的。

当然在一些情况下仍需要显式地控制同步，以确保最佳性能或避免竞争条件，例如多线程。

# 引擎的异步渲染

主流游戏引擎的渲染通常是异步的。当然这实际上是一个非常庞大的话题，我不知道是否要写在这里。

# 参考

[zhihu - OpenGL的数据与同步](https://zhuanlan.zhihu.com/p/543748224)

[github pages - OpenGL驱动的异步性](https://rasterer.github.io/2014/10/OpenGL%E9%A9%B1%E5%8A%A8%E7%9A%84%E5%BC%82%E6%AD%A5%E6%80%A7/)

[WikiPedia - Multiple buffering](https://en.wikipedia.org/wiki/Multiple_buffering)

[zhihu - OpenGL API 介绍(七)：同步](https://zhuanlan.zhihu.com/p/609071975)
