#URP
# Intro

> SRP 并不直接提供`ScriptableRenderer`，这是 URP 提供的。

按照解耦，`ScriptableRenderer`只关心与他对应的相机的渲染。它被管线实例操作完成数据准备和启动对相应相机的渲染。但是`ScriptableRenderer`仍旧只是宏观上组织了一个相机的渲染流程，具体的细节则是由继承自`ScriptableRenderer`的类补充的，例如 URP 中`ForwardRenderer`实现了前向渲染的具体逻辑。

派生类直接和各个 Pass 打交道，以及配置渲染所需的数据。

虽说`ScriptableRenderer`基类负责管理多个`ScriptableRenderPass`，但是在实际执行，也就是`ScriptableRenderer.Excute()`的时候，并不直接调用每个 Pass 的`Execute`方法，而是依靠`RenderPassBlock`管理，在相应的阶段执行某个 Block 下所拥有的 Pass，URP 中分成了 4 个 Block 如下所示：

- BeforeRendering
- MainRenderingOpaque
- MainRenderingTransparent
- AfterRendering

在实际渲染的时候是调用`ExecuteBlock`执行每个 Block 下包含的 Pass。然后获取到该 Block 含有的 Pass 的 Index 范围，并依次为每个 Index 执行`ExecuteRenderPass`。
# Detail

`ScriptableRenderer`的派生类管理了一个相机的完整渲染流程。而一个相机的渲染自然分多个阶段，而那则是`ScriptableRenderPass`做的事情。

`ScriptableRenderer`与相机二者相互关联：一个摄像机对应且仅对应一个`ScriptableRenderer`实例，可以在相机的 Inspector 面板找到使用的 Renderer。不同的摄像机可以都使用同一类`ScriptableRenderer`。`ScriptableRenderer`会通过`GetRenderer(camera)`获取与指定相机关联的 `ScriptableRenderer` 实例实现每个相机的渲染。

1. 根据相机类型调用`Clear(CameraRenderType cameraType)`去清空缓冲区
2. `SetupCullingParameters`：准备剔除参数
3. 根据剔除参数使用`context.Cull`获取剔除结果
4. 结合剔除结果和相机数据准备渲染数据`InitializeRenderingData`
5. `Setup()`：设置要进行哪些 Pass
6. `Execute()`：实际执行渲染逻辑，通过执行 Block 间接实现对激活的 Pass 的调用

具体来说，`ScriptableRenderer`负责收集场景中的可见物体（通过 `CullingResults`）并渲染该相机可看见的场景。由于渲染必然有存放输出的地方，所以这个阶段会配置好 RenderTarget。

此外它会利用引擎的数据、相机的参数配置和自身的参数配置完成一整个渲染流程所涉及的着色器输入的配置，以供后面启动每个 Pass 的渲染，例如：

- 相机与屏幕的内建变量的配置，例如`_WorldSpaceCameraPos`

有必要指出的是，`ScriptableRenderer`和 Renderer 组件并非同一个东西。后者是附加在要被渲染的物体上的，而`ScriptableRenderer` 则是 URP 独有的和摄像机一一对应的。

## Culling

渲染一个相机的物体是要经过剔除的。SRP 的剔除参数要通过相机获取，而 URP 选择直接把他放在了`ScriptableRenderer`里，要求它自行实现，大意如下：

```csharp
// typeof(a) == Camera
bool result = a.TryGetCullingParameters(out ScriptableCullingParameters cullingParameter);
if(!result) // 对空情况要处理，可以不进行后续渲染
// 实际的剔除通过 ScriptableRenderContext 进行
CullingResults cullingResult = context.Cull(ref cullingParameter);
```

但是经过剔除的元素仅仅表示能够被摄像机看见，但是不代表有资格进入某个 pass。而最终的渲染对象还需要结合`FilteringSettings`和`DrawingSettings`进一步过滤和配置。

## FilteringSettings

上面说了经过剔除的元素不意味着有资格执行某个 Pass，因此还要进行进一步的过滤。

`FilteringSettings`是专门给`ScriptableRenderContext.DrawRenderers`提供信息的结构体，他为进一步筛选那些物体可以参与渲染提供了要求。URP 在拿到 `CullingResults` 之后并未直接在`ScriptableRenderer`中做后续的处理，而是把相关的权限下放给了`ScriptableRenderPass`，由各个 Pass 自行决定要绘制哪些物体。

`FilteringSettings`主要针对渲染队列的值和`LayerMask`对剔除结果做进一步限制。

Tips：由于结构体的`Equals(object obj)`会产生装箱，`FilteringSettings`实现了`IEquatable<FilteringSettings>`，进而提供了更高效的比较。此外，通过实现`IEquatable<T>`还可以重新定义 class 的比较行为（原先是比较两个对象是否指向内存中的同一个地址）

## DrawingSettings

`DrawingSettings`的功能也很简洁：
1. 指定使用哪个 Pass 进行绘制(可以指定多个 Pass 作为备选)
2. 配置排序方式
3. 设置 GPU Instancing 选项
# ScriptableRendererData

`ScriptableRendererData`是`ScriptableRenderer`的配置类，用于配置：

- 使用的 `ScriptableRenderer` 类型（如 `ForwardRenderer`）
- 启用的 Renderer Features
- 渲染顺序和渲染目标设置

# Further Understanding

## Comparison between Base and Derived Class

> 前面对于功能的讨论总容易把基类`ScriptableRenderer`和派生类混淆，因此在这里讨论一下二者的功能划分

基类把实现`abstract void ScriptableRenderer.Setup(ScriptableRenderContext, ref RenderingData)`交由了派生类，也就是说在`ForwardRenderer`中能看到具体的实现。这部分逻辑因为 URP 支持不同的渲染路径，他们的 Setup 逻辑不一致。
# ForwardRenderer

