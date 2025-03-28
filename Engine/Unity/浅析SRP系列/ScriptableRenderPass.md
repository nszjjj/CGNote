# Intro

## Basic Concepts

对外提供的是`Excute`函数，`ScriptableRenderer`会根据 Pass 的排序值依次调用各 pass 的`Excute`。

虽然带个 Pass，其实这个东西和着色器的 Pass 不是一个概念。

---

<mark>一次性 Pass</mark>：

可以通过在管线预留好的回调（委托）中增加一个用于添加 Pass 的方式（见下面引用1），来实现即插即用的 Pass 添加，而不必如 RenderFeature 一般

## Independent shading steps

> `ScriptableRenderPass`与 Shader Pass 并非同一个概念，虽然它也带个 Pass 的字样

`ScriptableRenderPass`是管线的一个阶段，定义了一个绘制步骤，这与着色器的 Pass 并非同一概念。例如 URP 准备了`RenderObjectsPass`、`PostProcessPass`等阶段作为独立的着色步骤，URP 中一些常见的如下表所示：

| `ScriptableRenderPass` 类型   | 职责                               |
| --------------------------- | -------------------------------- |
| `RenderObjectsPass`         | 绘制特定条件的几何体（如覆盖层、透明物体）            |
| `PostProcessPass`           | 应用全屏后处理效果（Bloom、Color Grading 等） |
| `DrawSkyboxPass`            | 渲染天空盒                            |
| `MainLightShadowCasterPass` | 处理主光源的阴影投射                       |
| `ScreenSpaceShadowPass`     | 计算屏幕空间阴影                         |
显然，一个绘制步骤可能会用到多个 Shader，这显然也表明一个绘制步骤可能由多个 Shader Pass 构成。相比之下，Shader Pass 仅仅完成了一次渲染过程（由顶点数据到最终写入渲染目标的过程）。

## Why Need ScriptableRenderPass

上一小节说了这是一个独立的着色步骤，因为不希望`ScriptableRenderer`直接拿着 Shader 去操作各种细节，那太乱了。对于一个绘制步骤，可能涉及一个到多个 Shader，还有各种临时申请资源，还有各种绘制设置的细节，如果这些东西都堆在`ScriptableRenderer`则显得十分臃肿。

因此考虑到很多步骤是一个共性行为，就会把他们抽象出来单独管理：

1. 配置自己的渲染状态（如混合模式、深度测试等）
2. 设置专属的 `DrawingSettings` 和 `FilteringSettings`
3. 执行完整的绘制逻辑（可能包含多个 `CommandBuffer` 操作）
4. 管理临时渲染目标（如后处理需要的 RT）

以后处理为例，URP SSAO 实际上就会用到多个 RenderTarget 存放中间数据，而这都需要使用`CommandBuffer.GetTemporaryRT`创建临时纹理，并且有创建就要有释放，这都需要由`ScriptableRenderPass`自行管理。

所以自己做 SRP 的时候，例如渲染透明物体、屏幕空间反射、后处理效果都可以对应一个`ScriptableRenderPass`。

## How to Match a Pass

本小节探讨一下在`ScriptableRenderPass`如何匹配要使用的 Shader Pass。因为在某些地方会发现使用的是 index 进行索引找到 Pass，而有的地方却使用 Name。

在`DrawingSettings`中，使用 Shader Pass 的名字进行索引

```csharp
var drawingSettings = new DrawingSettings(
    new ShaderTagId("UniversalForward"), // 指定 Pass 名称
    sortingSettings
);
```
# Reference
[Tech-Artist 学习笔记：URP 中比 RendererFeature 更灵活的自定义 Pass 插入小技巧](https://zhuanlan.zhihu.com/p/550948454)