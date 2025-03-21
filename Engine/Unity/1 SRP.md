# SRP

## Definition of Pipeline

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

具体配置`ScriptableRenderContext`实例内容的地方是`RenderPipeline.Render`函数。任何一个 SRP 实现都需要继承 `RenderPipeline` 类，并且在 `Render()` 方法中实现具体的渲染逻辑，这通常会在for循环内对每个相机分别处理。

一如 OpenGL 存在命令缓冲区的概念（严格来说官方没说有，只是说存在一个类似的概念），在`Render()`方法中的逻辑实际上也是存放在缓冲区，直到`ScriptableRenderContext.Submit`才正式提交执行。其本质就是在向内部的命令队列中添加渲染命令，而`Submit`则是将所有已记录的渲染命令一次性提交给 Unity 的底层图形 API，然后 Unity 会将这些命令转换为底层的图形 API 调用，并将其发送给 GPU 执行。

## RenderPipelineAsset

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

## RenderPipeline 类

可以将 RenderPipeline 类比为一种接口或抽象层，它使Unity的渲染后端能够访问并执行自定义的渲染逻辑。同时，RenderPipeline 类解耦了渲染逻辑与底层实现

> 一个渲染管线的实例就是继承了 RenderPipeline 的类的实例

不过继承 RenderPipeline 的类并非构成摸个 SRP 运行的全部逻辑，例如还包括`RenderPipelineAsset`等。

这个类的主要功能有：

1. 定义渲染流程：老生常谈的`Render()`

2. 支持多相机渲染：会传入相机数组

3. 管理渲染资源：提供了 `Initialize` 和 `Dispose` 方法，用于初始化和释放渲染资源

然后就可以实现很多高级功能，例如针对特定平台优化，结合条件编译实现不同渲染逻辑的执行。

## CommandBuffer

上文提到在一个`RenderPipeline.Render`函数内部实现具体的渲染逻辑，在其中可能会使用一个到多个 CommandBuffer。CommandBuffer 在不显式指定 RenderTarget 的时候是默认使用当前绑定的 RenderTarget。

`CommandBuffer` 是一个轻量级的容器，用于记录一系列渲染命令（如清除屏幕、绘制几何体、设置渲染状态等）。

一个典型的应用流程分为四步：

```csharp
// 创建 CommandBuffer
var cmd = new CommandBuffer();
// 记录渲染命令
cmd.ClearRenderTarget(true, true, Color.clear);
cmd.DrawMesh(mesh, matrix, material);
// 执行 CommandBuffer
context.ExecuteCommandBuffer(cmd);
// 释放 CommandBuffer
cmd.Release();
```

然而不一定非要在`Render()`中使用 CommandBuffer，实际上在`Render`函数里面的调用链上的任何位置都可以使用CommandBuffer对渲染流程进行配置，例如在 `ScriptableRenderer.Excute` 中利用它对渲染相关的内建变量做了配置（类似于你写一个OpenGL渲染器要先配置各种 uniform）

典型的时机包括：

- `ScriptableRenderPipeline.Render`：执行一些全局的渲染设置或操作

- `ScriptableRenderer.Excute` \ `ScriptableRenderer.Excute`：

- `ScriptableRenderPass.Configure`：配置渲染目标 (Render Target) 和深度缓冲区

- `ScriptableRenderPass.Execute`：执行实际的渲染指令，如绘制网格 (Mesh)、执行 Compute Shader、Blit 纹理等



## Renderer

Renderer 就是绘制组件，它是一个基类，常见的派生类有： `MeshRenderer`、`SkinnedMeshRenderer` 、`LineRenderer`等。

在没有其它干涉的情况下，Unity的 GameObject 通过相关绘制组件与管线交互进行绘制。每个 GameObject 通常都附加了一个或多个渲染组件。这些组件管理渲染所需的信息，例如材质、Mesh 或者 Transform。

Renderer 会将渲染过程中用到的信息传递给管线，该过程由引擎内部自动完成。在没有其他干涉的情况下，Unity引擎会自动处理 `Renderer` 与渲染管线之间的交互。包括：

1. 收集渲染数据

2. 剔除：根据摄像机的视锥体（Frustum）和遮挡关系，剔除不可见的 Renderer。这两个过程发生在 CPU 侧，

3. 生成渲染指令：将可见的 `Renderer` 数据转换为渲染指令，并提交给渲染管线

4. 执行渲染：管线根据渲染指令实际调用 GPU 进行绘制

额外地，mono 脚本可以修改渲染组件的绘制参数，从而影响 GameObject 的渲染结果。在`Update`或是`LateUpdate`，都属于 GameLogic Stage，是先于 Scene Rendering 的，所以信息肯定可以更新过去。但是还有些阶段如`OnPostRender`肯定是更新不了会等到下一帧的。

---

这个 renderer 和`CameraData.renderer`不是一个东西，后者是`ScriptableRenderer`

# RenderPipeline调用链

从自顶向下的顺序描述一下，首先每帧渲染肯定是核心循环发起的，他告诉 SRP 实例要进行渲染，然后把要渲染的相机传给实例。

<mark>RenderPipeline</mark>：

SRP 的实例就是继承自`RenderPipeline`的类，他配置在`GraphicsSettings.renderPipelineAsset`中。他只知道要渲染哪些相机，以及去操纵渲染的上下文，那他就得去获取很多数据和做一些配置才能开始真正的渲染。所以它会在开头先配置一些宏观的特征，比如说是否启用哪些渲染技术（在`SetSupportedRenderingFeatures``中）

但是仍有一些渲染参数和具体的相机有关，所以会在`RenderSingleCamera`的时候调用`InitializeRenderingData`去为每个相机单独配置，主要涉及光源、阴影和后处理。

一个着色器运行的时候会获取引擎的内建变量，具体有哪些可以在官方文档$^{[6]}$查到。这些内建变量并不是在核心循环发起渲染时就能确定的，例如有些光源相关的变量在剔除完之后才能确定，有些关于相机的变量则要等到具体渲染某个相机时才能确定。所以在整个调用链的不同位置都有可能去设置这些内建变量的值。

由于`Render()`调用一次就是一帧，所以在函数里开头的地方通过`SetupPerFrameShaderConstants`设置了每帧更新的常量。

<mark>ScriptableRenderer</mark>：

然后在这个阶段就已经知道要渲染哪个相机了，为了渲染相机看到的内容，管线实例会使用当前正在处理的相机的 Renderer 去准备数据和渲染。

所以要做的事情就是根据相机的内容结合其他数据准备要渲染的数据，设置好各个 Pass 的顺序。当然 URP 把`Setup()`做成了抽象函数，具体的 Renderer 比如说`ForwardRenderer`才会去实现，因为 URP 支持不同的渲染路径，他们的 Setup 逻辑不一致。

因为渲染相机一定要有输出的地方，所以这个阶段会配置好

最终在`Excute()`中，完成了对各个 Pass 的执行。

## RenderPipeline

小贴士：Unity 提供了`CommandBufferPool`以优化性能，但是不要将 `ProfilingScope` 与通过 `CommandBufferPool.Get("name")` 获取的命名 `CommandBuffer` 混合使用。命名 `CommandBuffer` 在执行时会自动关闭其自身的 Profiling Scope，这会导致 `ProfilingScope` 标记孤立。

---

有一个小细节是，光照策略。渲染一个场景的时候会从`cullingResult`中获得参与计算的光源，此时光源先后顺序是有要求的，一般先是主光源，后是附加光源，最后粒子系统的光照。

主光源会从方向光源中产生，如果一个方向光源被指定是太阳光源，那他就是主光源。如果没有任何一个方向光源被指定为太阳光源，那光照强度最大的则是主光源。

---

在环境允许的情况下还额外提供了`ApplyAdaptivePerformance`，根据设备的性能状况动态调整渲染设置，以保持流畅的帧率

## ScriptableRenderer

`ScriptableRenderer`和上面提到的 Renderer 并非同一个东西，`ScriptableRenderer` 负责管理多个 `ScriptableRenderPass`，并调用它们的 `Execute` 方法。

相机的渲染是通过`ScriptableRenderer`实现的，这二者相互关联，一个摄像机对应且仅对应一个`ScriptableRenderer`实例，可以在相机的 Inspector 面板找到使用的 Renderer。不同的摄像机可以都使用同一类`ScriptableRenderer`。`ScriptableRenderer`会通过`GetRenderer(camera)`获取与指定相机关联的 `ScriptableRenderer` 实例实现每个相机的渲染。

1. 根据相机类型调用`Clear(CameraRenderType cameraType)`去清空缓冲区

2. `SetupCullingParameters`：准备剔除参数

3. 根据剔除参数使用`context.Cull`获取剔除结果

4. 结合剔除结果和相机数据准备渲染数据`InitializeRenderingData`（上面提到了）

5. `Setup()`：设置要进行哪些 Pass

6. `Execute()`：按顺序渲染每个 Pass

具体来说`ScriptableRenderer` 负责收集场景中的可见物体（通过 `CullingResults`）并渲染该相机可看见的场景。它会利用相机的参数配置和自身的参数配置完成一整个渲染流程所涉及的 Pass 的提交执行。

当然这也是分阶段的，URP里面叫做 Block，定义在`RenderPassBlock`中，分别有：

- BeforeRendering

- MainRenderingOpaque

- MainRenderingTransparent

- AfterRendering

就定位来说，一个`ScriptableRenderer`直接管理了<mark>一个相机的完整渲染流程</mark>。而一个渲染流程自然分多个阶段，这就是`ScriptableRenderPass`做的事情。

---

当然`ScriptableRenderer`本身也有一个`Excute`，其定位是完成已入队的各个 Pass 的调用执行。但是各个 Pass 执行肯定是需要有环境的，所以这里做了这些事情：

- 完成着色器用到的相机与屏幕的内建变量的配置，例如`_WorldSpaceCameraPos`

## ScriptableRenderPass

对外提供的是`Excute`函数，根据 ScriptableRenderer 维护的 pass 的队列的顺序依次调用各 pass 的`Excute`。

虽然带个 Pass，其实这个东西和着色器的 Pass 不是一个概念。实际使用中会用`ScriptableRenderPass`封装一个独立的渲染任务或阶段，例如渲染透明物体、屏幕空间反射、后处理效果都可以对应一个`ScriptableRenderPass`。

## RenderFeature

`RendererFeature`是用于向 `ScriptableRenderer` 中添加自定义 `ScriptableRenderPass` 的工具。它通过 `ScriptableRendererFeature.AddRenderPasses` 方法将 `ScriptableRenderPass` 添加到 `ScriptableRenderer` 中。

---

<mark>一次性 Pass</mark>：

可以通过在管线预留好的回调（委托）中增加一个用于添加 Pass 的方式$^{[5]}$，来实现即插即用的 Pass 添加，而不必如 RenderFeature 一般

## ScriptableRendererData

`ScriptableRendererData` 是 `ScriptableRenderer` 的配置类，用于配置：

- 使用的 `ScriptableRenderer` 类型（如 `ForwardRenderer`）。

- 启用的 Renderer Features。

- 渲染顺序和渲染目标设置。

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

# 关键机制

标识一个物体可渲染其实也算一个机制，就是上面的 Renderer 里提到的。

## 数据准备

## 管线选择物体

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

额外提一句：SubShader 或者 Pass 都可以有 Tags，但`LightMode` 只能用于 Pass 的 Tags。但是`Render Queue` 只能用于`SubShader`的 Tags，其作用是定义整个 `SubShader` 的渲染顺序。SubShader 上的设置会影响到其中的 Pass，这意味着会间接影响具有同一种 `LightMode`的 pass 中物体的渲染顺序。Unity 在获取到使用这些 Pass 的物体后会依据 Render Queue 的值排序再渲染。

针对这个再额外提一句：一个 ShaderLab 文件中的多个 SubShader 通常是为了解决相同问题的不同实现方法，这些实现方法可能针对不同的硬件平台、性能需求或渲染管线进行优化。每个 SubShader 可以看作是同一渲染目标的不同版本。那自然存在多个 SubShade r都含有相同 LightMode 的 Pass，Unity 会去选择最为匹配的那个

---

当然，渲染物体的方式不止这一种。

## SRP Batcher

SRP Batcher 也是 SRP 中不得不品的一个机制，其核心目标是通过减少CPU与GPU之间的通信开销，提升渲染性能。其中的关键因素在于状态切换为 CPU 和 GPU 双侧带来的开销。本质上不会减少绘制调用，但是为了易于开发人员理解所以编辑器上显示减少

有关开销内容可以参考 [5 Depth in OpenGL.md](../../Graphics API/OpenGL/5 Depth in OpenGL.md) 中关于状态切换开销的文字。

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

[5] [Tech-Artist 学习笔记：URP 中比 RendererFeature 更灵活的自定义 Pass 插入小技巧](https://zhuanlan.zhihu.com/p/550948454)

[6] [Unity Documentation - Built-in shader variables reference](https://docs.unity3d.com/Manual/SL-UnityShaderVariables.html)
