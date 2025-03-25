#SRP #URP
# Definition of Pipeline

引擎的 pipeline 和图形API所说的 pipeline 是两个截然不同的东西。渲染管线是引擎实现的一套完整流程，用于将3D场景转换为2D图像。它包括多个阶段，如几何处理、光栅化、着色、后处理等。而图形 API 的 Pipeline 则是一组固定的硬件操作步骤，用于处理顶点和片元数据。

对于一个渲染引擎来说，渲染管线的任务是这样的：

1. 数据收集：从场景中收集几何、材质、光照、相机等信息。
2. 任务制定：根据渲染需求（如阴影、反射、后处理等）生成具体的渲染任务。
3. 任务排序与提交：优化任务顺序（如按材质、深度排序以减少状态切换），提交给图形API执行。

> Unity 官方是解释是：渲染管线执行一系列操作来获取场景的内容，并将这些内容显示在屏幕上。

渲染管线属于渲染模块的一部分，并且依赖于渲染模块的其他部分（如资源管理、场景管理）来获取数据和执行任务。

管线的功能是可以自行定制的，这意味着必须获取某些信息和发布一些指令同 Unity 的渲染后端交互，以完成渲染流程。Unity 的渲染后端提供了 [ScriptableRenderContext](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.html) 使开发者拥有对渲染后端的一定控制能力（渲染后端是一个黑盒，开发者实际上不知道其运行方式，至少不知道大部分）。
# SRP 内容

这一部分主要关注 SRP 为构建自定义的管线准备了什么东西。
# URP 调用链

## 简述

省流：`RenderPipeline->ScriptableRenderer->ScriptableRendererPass`

从自顶向下的顺序来看，首先每帧渲染是由核心循环发起的，他告诉 URP 实例要进行渲染，然后把要渲染的相机传给实例。而 URP 的实例则是继承自[[RenderPipeline]]的类，它是由`RenderPipelineAsset`提供的。

URP 认为 RenderPipeline 核心的工作就是启动各个相机的渲染，因此在`UniversalRenderPipeline.Render()`中主要工作就是排序相机和启动单个相机的渲染。

启用单个相机的渲染最终依靠与相机对应的`ScriptableRenderer`执行，[[ScriptableRenderer]]承载了对于相机的渲染的具体逻辑。该类具体定义了渲染一个相机的流程。从解耦层面来说，它只知道要渲染哪个相机，并依据相关数据完成对该相机的渲染。

对与一个相机的渲染显然是分多个阶段的，例如对于不透明的物体肯定先于透明物体的渲染，因此将不同的渲染阶段封装为不同的 Pass。但是在 Pass 之上，URP 还有一个名为`RenderPassBlock`的概念，这主要是确立了更为宏观的渲染阶段，并提供了一些事件的插入时机。

关于具体各个 Pass 的渲染，`ScriptableRenderer`依照相关渲染设置维护一个容纳了待渲染的 Pass 的队列（其实就是个 List），实际执行的时候是通过`ExecuteBlock`渲染单个 Block，在其中会获取到属于这个 Block 的 Pass 的 index，并从队列中拿出来渲染。

当然，贯穿上述整个调用链，各种用于监测性能的`ProfilingScope`也穿插在其中。

一个着色器运行的时候会获取引擎的内建变量，具体有哪些可以在官方文档[Unity Documentation - Built-in shader variables reference](https://docs.unity3d.com/Manual/SL-UnityShaderVariables.html)查到。这些内建变量并不是在核心循环发起渲染时就能确定的，例如有些光源相关的变量在剔除完之后才能确定，有些关于相机的变量则要等到具体渲染某个相机时才能确定。所以在整个调用链的不同位置都有可能去设置这些内建变量的值。
## Q&A

### 堆叠摄像机

一个摄像机上会有摄像机堆栈，摄像机堆栈上的相机称之为堆叠相机（Overlay Camera），堆叠摄像机身不会作为独立的、需要渲染的摄像机直接传递给整个渲染管线，而是作为 Base Camera 的一部分被处理。
### GameCamera

Pipeline 的`Render`函数会这样判断：

```csharp
if(IsGameCamera(camera)){
    RenderCameraStack(renderContext, camera);
}
else{
    // 一些ProfilingScope统计
    UpdateVolumeFramework(camera, null);
    RenderSingleCamera(renderContext, camera);
    // 一些ProfilingScope统计
}
```

GameCamera 是游戏摄像机，直接和游戏画面关联的。但是 Unity 也存在其他类型的摄像机，例如：

- 阴影摄像机 (Shadow Camera)：渲染 shadow map 的
- 反射摄像机 (Reflection Camera)：生成反射纹理实现反射效果的
- UI 摄像机

当然`RenderCameraStack`里面还是会为 Base Camera 调用`RenderSingleCamera`，只是在这之后会额外地再处理堆叠相机
### SRP 与 UGUI

你可能关心 SRP 与 UGUI 的渲染是如何协调的，相关问题见[[UGUI 渲染机制]]。

UGUI 的渲染不走 SRP，而是由 Unity 的 Canvas 渲染系统处理的。Canvas Renderer 的更新频率主要取决于 Canvas 的重建频率，这意味着每次核心循环并不一定更新渲染 UI 的内容。

Canvas 的更新依靠`CanvasWillRenderCanvases`事件，
# 关键机制
## Data Collection

## Renderer

`Renderer`就是绘制组件，它是所有绘制组件的基类（当然它继承自`Component`），常见的派生类有： `MeshRenderer`、`SkinnedMeshRenderer` 、`LineRenderer`等。

在没有其它干涉的情况下，Unity的 GameObject 通过相关绘制组件与管线交互进行绘制，换句话说：Renderer 标识了一个 GameObject 可渲染。每个 GameObject 通常都附加了一个或多个渲染组件。这些组件管理渲染所需的信息，例如材质、Mesh 或者 Transform。

Renderer 会将渲染过程中用到的信息传递给管线，该过程由引擎内部自动完成。在没有其他干涉的情况下，Unity 引擎会自动处理 `Renderer` 与渲染管线之间的交互。包括：

1. 收集渲染数据
2. 剔除：根据摄像机的视锥体（Frustum）和遮挡关系，剔除不可见的 Renderer。这两个过程发生在 CPU 侧，
3. 生成渲染指令：将可见的 `Renderer` 数据转换为渲染指令，并提交给渲染管线
4. 执行渲染：管线根据渲染指令实际调用 GPU 进行绘制

额外地，mono 脚本可以修改渲染组件的绘制参数，从而影响 GameObject 的渲染结果。在`Update`或是`LateUpdate`，都属于 GameLogic Stage，是先于 Scene Rendering 的，所以信息肯定可以更新过去。但是还有些阶段如`OnPostRender`肯定是更新不了会等到下一帧的。

注：这个 renderer 和`CameraData.renderer`不是一个东西，后者是`ScriptableRenderer`
## Select Renderable Object

ShaderLab 的 Tags 中的`"LightMode"`标识了一个着色器的`ShaderTagId`，虽然看着他字面含义是光照模式，但是实际上它标示了所属 Pass 的用途，如处理前向渲染、延迟渲染、阴影等。自然，这个`"LoghtMode"`可以自定义。

在`Render()`函数中，SRP 在不同的着色阶段依据 ShaderTagId 来选取着色器的某个 pass，并对含有这个 pass 的物体进行渲染。相关代码的示例如下：

```csharp
// 创建 ShaderTagId
ShaderTagId forwardBaseTag = new ShaderTagId("ForwardBase");
// 创建 SortingSettings（排序设置）
SortingSettings sortingSettings = new SortingSettings(camera) { criteria = SortingCriteria.CommonOpaque };
// 创建 DrawingSettings
DrawingSettings forwardSettings = new DrawingSettings(forwardBaseTag, sortingSettings);
// 渲染所有包含 "ForwardBase" Pass 的物体
context.DrawRenderers(renderingData.cullResults, ref forwardSettings, ref filteringSettings);
```

额外提一句：SubShader 或者 Pass 都可以有 Tags，但`LightMode` 只能用于 Pass 的 Tags。而`Render Queue` 只能用于`SubShader`的 Tags，其作用是定义整个 `SubShader` 的渲染顺序。SubShader 上的设置会影响到其中的 Pass，这意味着会间接影响具有同一种`LightMode`的 pass 中物体的渲染顺序。Unity 在获取到使用这些 Pass 的物体后会依据 Render Queue 的值排序再渲染。

针对这个再额外提一句：一个 ShaderLab 文件中的多个 SubShader 通常是为了解决相同问题的不同实现方法，这些实现方法可能针对不同的硬件平台、性能需求或渲染管线进行优化。每个 SubShader 可以看作是同一渲染目标的不同版本。那自然存在多个 SubShade r都含有相同 LightMode 的 Pass，Unity 会去选择最为匹配的那个

---

当然，渲染物体的方式不止这一种。
## SRP Batcher

SRP Batcher 也是 SRP 中不得不品的一个机制，其核心目标是通过减少CPU与GPU之间的通信开销，提升渲染性能。其中的关键因素在于状态切换为 CPU 和 GPU 双侧带来的开销。本质上不会减少绘制调用，但是为了易于开发人员理解所以编辑器上显示减少

有关开销内容可以参考 [[5 Depth in OpenGL]] 中关于状态切换开销的文字。

所以思路也很明确了：既然你说切换着色器开销大，那我就先把同一个着色器的能绘制的都绘制了。 那同一个着色器不同材质的情况也有很多，那就为每个材质实例存储一份常量缓冲区（Constant Buffer）数据在 GPU 内存中。当切换同个着色器的其它材质实例时就在显存中读取即可。

具体来说，着色器用到的属性分为两种：内置引擎属性和材质属性。为了能兼容 SRP 必须将他们都标记出来以便正确识别和处理。

- 内置引擎属性不需要开发者手动处理，Unity 引擎会自动将它们放在名为 `UnityPerDraw` 的 CBUFFER 中。
- 材质属性需要开发者手动将它们放在名为`UnityPerMaterial`的 CBUFFER（Constant Buffer）中

类似使用方式如下所示：

```glsl
Pass
{
    // 各种声明
    // ...
    CBUFFER_START(UnityPerMaterial)
        fixed4 _Color;
        sampler2D _MainTex;
        float4 _MainTex_ST;
        half _Glossiness;
    CBUFFER_END
    // 着色器代码
}
```

额外地，内置引擎属性也分两类，分别是 Per-Object 属性和全局属性。Per-Object 属性是每个 GameObject 独有的；全局属性是整个场景或渲染管线共享的，它们在不同的材质实例之间通常不会发生变化。

再额外地，`CBUFFER`并非只有`UnityPerMaterial`，还有`UnityPerDraw`或`UnityPerFrame`等，可以在 URP 的包里的`./ShaderLibrary/UnityInput.hlsl`里找到不同的 CBUFFER 及其内容。材质 Inspector 面板中设置的属性通常都属于`UnityPerMaterial`。

Unity 对于不同类型的 CBUFFER 的对待方式是不一样的，他们的更新频率和时机、在显存中的位置都不尽相同。例如`UnityPerDraw`在绘制物体变化的时候就会更新，但是`UnityPerMaterial`在材质属性变化的时候才会更新

材质属性可以不使用`CBUFFER_START(UnityPerMaterial)`，那样的话会作为独立的 Uniform 变量传递给着色器。    

# SRP Package

## Core RP

SRP 依赖 Core RP Package，可以在包管理中找到并安装。由于 URP 和HDRP都是依赖于 SRP 构建的，因此在 URP 或者 HDRP 管线的工程中也会包含这个包。

通常，使用一个 Pipeline Asset（一种 ScriptableObject，它包含了渲染管线的配置和设置）来描述一个 SRP

## Forward SRP

## Deferred SRP

# 参考

[zhihu - 【Unity】SRP底层渲染流程及原理](https://zhuanlan.zhihu.com/p/378781638)
[2] [zhihu - Unity的URP HDRP等SRP管线详解（包含源码分析）](https://zhuanlan.zhihu.com/p/654056866)
[zhihu - 关于静态批处理/动态批处理/GPU Instancing /SRP Batcher的详细剖析](https://zhuanlan.zhihu.com/p/98642798)
[zhihu - Unity性能优化最佳实践(2)-渲染优化: SrpBatcher解析与优化](https://zhuanlan.zhihu.com/p/693145681)
[zhihu - Unity SRP 01](https://zhuanlan.zhihu.com/p/92686142)