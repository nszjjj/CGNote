## 简介

> 一个渲染管线的实例就是继承了 RenderPipeline 的类的实例

可以将 RenderPipeline 类比为一种接口或抽象层，它使Unity的渲染后端能够访问并执行自定义的渲染逻辑。同时，`RenderPipeline`类解耦了渲染逻辑与底层实现。

不过继承自`RenderPipeline`的类并非构成摸个 SRP 运行的全部逻辑，例如还包括`RenderPipelineAsset`等。

这个类的主要功能有：

1. 定义渲染流程：老生常谈的`Render()`
2. 管理渲染资源：提供了 `Initialize` 和 `Dispose` 方法，用于初始化和释放渲染资源

引擎调用`RenderPipeline`类进行渲染的时候传入了用于操作图形后端的`ScriptableRenderContext`，以及要处理的内容（一组`Camera`）。这是它知道的全部内容，但是远不够用于渲染，因此它就得进行数据准备。所以它会在开头先配置一些宏观的特征，比如说是否启用哪些渲染技术（在`SetSupportedRenderingFeatures`中）

但是仍有一些渲染参数和具体的相机有关，所以会在`RenderSingleCamera`的时候调用`InitializeRenderingData`去为每个相机单独配置，主要涉及光源、阴影和后处理。

由于`Render()`调用一次就是一帧，所以在函数里开头的地方通过`SetupPerFrameShaderConstants`设置了每帧更新的常量。

然后还可以实现很多高级功能，例如针对特定平台优化，结合条件编译实现不同渲染逻辑的执行。

## 设计哲学与定位

很简单的问题，明明直接在`Render()`中就可以利用 [[CommandBuffer]] 写渲染逻辑，为什么不这么做呢？主要问题在于职责分离和模块化设计。

渲染就是要呈现看见的东西，能够看见自然是通过摄像机，所以`RenderPipeline`的职责就是负责协调和执行相机的渲染。

对于 SRP 特别是 URP 而言：

- `RenderPipeline`：渲染管线的入口，负责调用 `ScriptableRenderer` 执行渲染
- `ScriptableRenderer`：管理一个相机的渲染流程，调度多个 `ScriptableRenderPass`。
- `ScriptableRenderPass`：实现具体的渲染任务（如阴影渲染、后处理等）
- `RendererFeature`：向 `ScriptableRenderer` 中添加自定义的 `ScriptableRenderPass`

自己做简单的 SRP 就算不实现`ScriptableRenderer`，对具体摄像机的渲染逻辑也会抽出来，因为不同类型的相机的用途和渲染需要准备的数据和渲染流程可能有所差异。在底部引用的第一篇文章中，`Renderer()`就是准备好每个相机的数据，然后为每个相机调用了自己实现的`RenderContent(bool, Camera, CullingResults, ScriptableRenderContext)`，但是即便这样，也是把对于相机的渲染抽象了出来。

## SRP 中的细节

有一个小细节是，光照策略。渲染一个场景的时候会从`cullingResult`中获得参与计算的光源，此时光源先后顺序是有要求的，一般先是主光源，后是附加光源，最后粒子系统的光照。

主光源会从方向光源中产生，如果一个方向光源被指定是太阳光源，那他就是主光源。如果没有任何一个方向光源被指定为太阳光源，那光照强度最大的则是主光源。

---

在环境允许的情况下还额外提供了`ApplyAdaptivePerformance`，根据设备的性能状况动态调整渲染设置，以保持流畅的帧率

## ScriptableRenderContext

就好比在开发 OpenGL 应用的时候需要使用 OpenGL Context 一样。`ScriptableRenderContext`是 SRP 渲染环境的核心。 它管理 SRP 的状态、命令缓冲区、渲染 Pass 和其他资源。在自定义 SRP 流程时，需要结合`ScriptableRenderContext`实例才能完成。

`ScriptableRenderContext`实例是 Unity 提供的，但并非唯一的。它通常和一个相机的渲染流程绑定，负责该相机的渲染任务。实例的生命周期由 Unity 自动管理，无需手动创建或者释放。

具体配置`ScriptableRenderContext`实例内容的地方是`RenderPipeline.Render`函数。任何一个 SRP 实现都需要继承 `RenderPipeline` 类，并且在`Render()`方法中实现具体的渲染逻辑，这通常会在for循环内对每个相机分别处理。一次`Renderer()`就是一帧。

一如 OpenGL 存在命令缓冲区的概念（严格来说官方没说有，只是说存在一个类似的概念），在`Render()`方法中的逻辑实际上也是存放在缓冲区，直到`ScriptableRenderContext.Submit`才正式提交执行。其本质就是在向内部的命令队列中添加渲染命令，而`Submit`则是将所有已记录的渲染命令一次性提交给 Unity 的底层图形 API，然后 Unity 会将这些命令转换为底层的图形 API 调用，并将其发送给 GPU 执行。

## RenderPipelineAsset

配置在`GraphicsSettings.renderPipelineAsset`

1. 提供管线实例的创建
2. 管线的参数配置

一个典型的例子是这样：

```csharp
public class CustomRenderPipelineAsset : RenderPipelineAsset
{
    // 光照设置
    [Header("Lighting Settings")]
    public bool enableRealtimeShadows = true;
    public int shadowMapResolution = 1024;

    // 后处理设置
    [Header("Post-Processing Settings")]
    public bool enableBloom = true;
    [Range(0, 5)] public float bloomIntensity = 1.0f;

    // 自定义颜色
    [Header("Custom Settings")]
    public Color backgroundColor = Color.black;

    protected override RenderPipeline CreatePipeline()
    {
        // 创建自定义渲染管线实例，并传递当前 Asset 的配置
        return new CustomRenderPipeline(this);
    }
}
```

相当于把管线最终可调节的参数都放在了 Asset 里，这样能够集中管理、自定义参数、动态调整和持久化存储。

因为创建管线的时候已经把 Asset 的配置传进来了，所以管线是可以获取到这些参数的。

# Universal Render Logic

本节主要讨论一般来讲较为通用的渲染逻辑是怎样的

## Render()

其实这个`ScriptableRenderer`倒不一定非得存在，如果管线确实简单的话，直接`foreach (var camera in cameras)`执行各个 Pass 就可以了。不过这样的管线就显得较为固定，因为各个原子性的渲染阶段被固定好了。

## Error Detection

完整的管线也需要有错误检测，对于出于某些原因无法绘制的材质，应当使用专门的错误材质进行绘制（对，就是那个经典的品红色材质）
# Reference

[zhihu - Unity SRP 01](https://zhuanlan.zhihu.com/p/92686142)