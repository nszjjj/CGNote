# 渲染管线

## Definition

引擎的 pipeline 和图形API所说的 pipeline 是两个截然不同的东西。渲染管线是引擎实现的一套完整流程，用于将3D场景转换为2D图像。它包括多个阶段，如几何处理、光栅化、着色、后处理等。而图形 API 的 Pipeline 则是一组固定的硬件操作步骤，用于处理顶点和片元数据。

对于一个渲染引擎来说，渲染管线的任务是这样的：

1. 数据收集：从场景中收集几何、材质、光照、相机等信息。

2. 任务制定：根据渲染需求（如阴影、反射、后处理等）生成具体的渲染任务。

3. 任务排序与提交：优化任务顺序（如按材质、深度排序以减少状态切换），提交给图形API执行。

> Unity 官方是解释是：渲染管线执行一系列操作来获取场景的内容，并将这些内容显示在屏幕上。

渲染管线属于渲染模块的一部分，并且依赖于渲染模块的其他部分（如资源管理、场景管理）来获取数据和执行任务。

管线的功能是可以自行定制的，这意味着必须获取某些信息和发布一些指令同 Unity 的渲染后端交互，以完成渲染流程。Unity 的渲染后端提供了 [ScriptableRenderContext](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.html) 使开发者拥有对渲染后端的一定控制能力（渲染后端是一个黑盒，开发者实际上不知道其运行方式，至少不知道大部分）。

## ScriptableRenderContext

就好比在开发 OpenGL 应用的时候需要使用 OpenGL Context 一样。`ScriptableRenderContext`是 SRP 渲染环境的核心。 它管理 SRP 的状态、命令缓冲区、渲染 Pass 和其他资源。在自定义 SRP 流程时，需要结合`ScriptableRenderContext`实例才能完成。

`ScriptableRenderContext`实例是 Unity 提供的，但并非唯一的。它通常和一个相机的渲染流程绑定，负责该相机的渲染任务。实例的生命周期由 Unity 自动管理，无需手动创建或者释放。

具体配置`ScriptableRenderContext`实例内容的地方是`RenderPipeline.Render`函数。任何一个 SRP 都需要继承 `RenderPipeline` 类，并且在 `Render()` 方法中实现具体的渲染逻辑。

一如 OpenGL 存在命令缓冲区的概念（严格来说官方没说有，只是说存在一个类似的概念），在`Render()`方法中的逻辑实际上也是存放在缓冲区，直到`ScriptableRenderContext.Submit`才正式提交执行。其本质就是在向内部的命令队列中添加渲染命令，而`Submit`则是将所有已记录的渲染命令一次性提交给 Unity 的底层图形 API，然后 Unity 会将这些命令转换为底层的图形 API 调用，并将其发送给 GPU 执行。

## RenderPipeline

可以将 RenderPipeline 类比为一种接口或抽象层，它使Unity的渲染后端能够访问并执行自定义的渲染逻辑。但是继承 RenderPipeline 的类并非构成摸个 SRP 运行的全部逻辑，例如还包括`RenderPipelineAsset`等。

> 一个渲染管线的实例就是继承了 RenderPipeline 的类的实例



## 绘制机制

在没有其它干涉的情况下，Unity的GameObject通过相关绘制组件（例如 `MeshRenderer`、`SkinnedMeshRenderer` 等），与管线交互进行绘制。每个GameObject通常都附加了一个或多个渲染组件。这些组件包含了渲染所需的信息，例如材质、Mesh或者Transform。相关的渲染组件会将渲染过程中用到的信息传递给管线，该过程由引擎内部自动完成。

额外地，mono脚本可以修改渲染组件的绘制参数，从而影响 GameObject 的渲染结果。在`Update`或是`LateUpdate`，都属于GameLogic Stage，是先于Scene Rendering的，所以信息肯定可以更新过去。但是还有些阶段如`OnPostRender`肯定是更新不了会等到下一帧的。

## Core RP

SRP依赖Core RP Package，可以在包管理中找到并安装。由于URP和HDRP都是依赖于SRP构建的，因此在URP或者HDRP管线的工程中也会包含这个包。

通常，使用一个Pipeline Asset（一种ScriptableObject，它包含了渲染管线的配置和设置）来描述一个SRP

## Forward SRP

## Deferred SRP

# 参考

[zhihu - 【Unity】SRP底层渲染流程及原理](https://zhuanlan.zhihu.com/p/378781638)
