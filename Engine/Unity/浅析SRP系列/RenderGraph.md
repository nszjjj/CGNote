#URP 
# Intro

## What is RenderGraph

RenderGraph（渲染图）是一种现代渲染架构，它将整个渲染流程表示为一张有向无环图(DAG)，其中节点表示渲染操作（如绘制、计算着色器等），边表示资源依赖关系（如纹理、缓冲区等）。

要认识这样一个机制是如何提出、是去解决什么问题的，先阅读这篇文章：[zhihu - Render Graph 全网最细介绍（一）](https://zhuanlan.zhihu.com/p/582098945)

简而言之就是，GPU 图形管线的一次完整流程叫做一个 Pass，Pass 产生的结果称之为 Resource，像 Buffer 或者 RenderTarget 这种占用显存的东西就是 Resource。这个东西不在用完后及时释放会造成显存利用率降低甚至显存泄露。

另一方面，渲染流程的多线程性和渲染系统的复杂性导致各个渲染环节不可避免的耦合，这对代码的维护与拓展、资源管理（手动管理资源很容易疏漏）和问题排查都造成了一定影响。而RenderGraph 就是管理渲染操作和资源（依赖、释放）的工具，其核心目标是通过以下内容更好的实现并行化和：

1. 解耦合
2. 自动的资源管理
3. 简化的多线程渲染
4. 良好的可扩展性
5. 容易debug

RenderGraph 相当于是从数据流动的视角来看渲染流程的，每个节点可能是一个或一组 Pass，就像一个函数抽象一样，我们只关注这组操作要获取什么，以及输出了什么。而输入输出，即 DAG
中的边，则表示的是数据流动（资源依赖）

为了构建这个 DAG，就需要三个阶段：

- Setup：各个 Pass 需进行注册，以便了解到依赖关系
- Compile：去除未使用的冗余，精简依赖结构，确定 Resource 生命周期
- Execute：各个 Pass 实际执行

## RenderGraph in URP

官方文档：[Render graph system in URP](https://docs.unity3d.com/6000.1/Documentation/Manual/urp/render-graph.html)

在Unity 中，RenderGraph 最初在 URP14 中得到引入，而在 URP17 中其地位进一步得到了确认，旧式的 RenderFeature 声明接口已经被标明废弃。官方文档称：RenderGraph 是一组用于创建 [[ScriptableRenderPass]] 的 API。

ScriptableRenderPass 是包含了 Pass 的实际执行逻辑的，但是需要额外告诉 RenderGraph 它需要的资源，因此才能判断依赖关系。包含了这两方面之后它才能被注册到 URP 中，受 RenderGraph 规划按顺序渲染。不过 RenderGraph 的参与和 RenderQueue 是两回事，RenderQueue 关注的是一个 Pass 要绘制的所有物体里，这些物体的绘制顺序。

# Using RenderGraph

## RecordRenderGraph

在 RenderGraph 存在之后，RenderFeature 的注册和以往没有太大差别，只是换成了依靠`RecordRenderGraph`将 Pass 注册

- RenderFeature 可以通过它添加一个或多个 Pass 到管线中

主要操作涉及：

```csharp
using (var builder = renderGraph.AddRasterRenderPass<PassData>(passName, out var passData))
{
	// 拿到所有可用的资源
	UniversalResourceData resourceData = frameData.Get<UniversalResourceData>();
	
	passData.copySourceTexture = resourceData.activeColorTexture;
	
	UniversalCameraData cameraData = frameData.Get<UniversalCameraData>();
	RenderTextureDescriptor desc = cameraData.cameraTargetDescriptor;
	desc.msaaSamples = 1;
	desc.depthBufferBits = 0;
	
	TextureHandle destination = UniversalRenderer.CreateRenderGraphTexture(renderGraph, desc, "CopyTexture", false);
	
	builder.UseTexture(passData.copySourceTexture);
	builder.SetRenderAttachment(destination, 0);
	builder.AllowPassCulling(false);
	
	builder.SetRenderFunc((PassData data, RasterGraphContext context) => ExecutePass(data, context));
}
```

这个来自官方的示例，可以看到主要
`passData`是一个传递渲染 Pass 所需数据的结构体
## Frame Data

官方文档参考：[Get data from the current frame in URP](https://docs.unity3d.com/6000.1/Documentation/Manual/urp/accessing-frame-data.html)

上文提到了 Resource 的概念，可以看到获取每一帧用到的 Resource 的方法是这样的：

```csharp
UniversalResourceData frameData = frameContext.Get<UniversalResourceData>();
```

不过需要注意的是有些数据可能还没计算出来

ScriptableRenderPass 可以指定自己需要的一些输入，Unity 会添加相关产生这些数据的 Pass 或者重新利用先前帧的数据，这些输入使用位枚举表示：

```csharp
[Flags]
public enum ScriptableRenderPassInput
```

然后通过`ConfigureInput`进行确认，完成对 Pass 所需要访问的渲染管线资源的声明
## IRasterRenderGraphBuilder

上面的 builder 变量是`IRasterRenderGraphBuilder`接口的一个实例，作为配置渲染过程相关信息的入口点