#SRP 
# Intro

## Get and Release

在 SRP 中进行渲染操作经常需要 RenderTexture 机械能辅助。SRP 提供了`CommandBuffer.GetTemporaryRT()`方法对临时渲染纹理进行获取。该函数有多重重载，如下所示：

```csharp
// 1. 使用 RenderTextureDescriptor
public void GetTemporaryRT (int nameID, RenderTextureDescriptor desc, FilterMode filter = FilterMode.Point);

// 2. 指定宽度、高度和格式
public void GetTemporaryRT (int nameID, int width, int height, int depthBuffer, FilterMode filter, RenderTextureFormat format, RenderTextureReadWrite readWrite, int antiAliasing, bool enableRandomWrite, RenderTextureMemoryless memorylessMode, bool useDynamicScale);

// 3. 简化版本，指定宽度和高度
public void GetTemporaryRT (int nameID, int width, int height);
```

一般`nameID`由`Shader.PropertyToID(string)`得到，它相当于为一个字符串计算一个哈希值，而字符串一般就设为这张贴图的名字就行了。值得注意的是这个字符串名称和具体的 Shader 无关，即便当前使用的 Shader 可能没有名称和这个字符串一样的 Property，但是依旧可以根据这个字符串得到一个 hash 值。

在获取到 TemporaryRT 之后应当使用它的`nameID`的值进行后续的访问操作，例如：

```csharp
int tempRT = Shader.PropertyToID("_MyTempRT");
CommandBuffer cmd = new CommandBuffer();

// 在 `CommandBuffer` 中设置渲染目标
cmd.SetRenderTarget(tempRT, RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store);

// 通过 `Material.SetTexture` 或 `CommandBuffer.SetGlobalTexture` 传递给Shader
cmd.SetGlobalTexture("_MyTempTexture", tempRT); // 在Shader中可通过 `_MyTempTexture` 访问

// 使用 `CommandBuffer.Blit` 复制或处理临时RT
cmd.Blit(sourceTexture, tempRT); // 渲染到临时RT
cmd.Blit(tempRT, destinationTexture); // 从临时RT读取
```

用完之后调用`CommandBuffer.ReleaseTemporaryRT()`释放。

不过 Unity SRP 有自动回收机制，会在帧结束时自动释放所有未显式管理的临时RT（通过内部对象池回收）

## RT for GBuffer

GBuffer 的 RT 是持久化保存的吗？

取决于具体的渲染需求和性能优化策略。Unity 内置的延迟渲染管线默认每帧重建 GBuffer。不过有些图形效果需要用到上一帧的 GBuffer，例如 TAA、运动模糊，这种情况可以使用持久化 RT。

Unity 实现的 GBuffer 是类似于相机的`"_CameraColorTexture"`这种全局纹理吗？不是。

对于每帧重建的 GBuffer，URP 在`ForwardRednerer.cs`中的`EnqueueDeferred()`里调用了`DeferredLights.Setup()`并传入了准备好的 GBuffer 数组。这个数组是在`ForwardRenderer(ForwardRendererData)`构造函数中准备的，类型是`RenderTargetHandle[]`

当然`RenderTargetHandle`只是配置了引用，具体实际的 RT 的声明不在这（我还没找到）
# RenderTargetHandle

> 本质是 RenderTexture 的引用

`RenderTargetHandle`是一个轻量级的 RenderTexture 标识符，主要用于在 SRP 的渲染流程中安全地引用和管理临时 RenderTarget。它的核心作用是解耦具体的 RT 资源与渲染逻辑，避免直接操作 RenderTexture 或 RTHandle 对象，从而简化跨 Pass 的 RT 共享和生命周期管理。

`RenderTargetHandle`作为抽象标识符，不直接持有渲染纹理，而是通过字符串名称间接引用。当然本质上字符串名称还是经过了`Shader.PropertyToID`转化回`int`。也就是说，必须在 RT 实际存在的情况下才能正确引用，一般包括这三类：

1. Unity内置RT（如相机纹理`"_CameraColorTexture"`）
2. 手动创建的RT：需提前通过`RenderTexture`或`RTHandle`分配
3. 临时RT：需通过`CommandBuffer.GetTemporaryRT`分配，并在释放前完成使用

`RenderTargetHandle`可以引用已有的渲染纹理，包括包括Unity内置的RT（如相机颜色/深度纹理）或手动创建的 `RenderTexture`/`RTHandle`。代码示例如下：

```csharp
// 获取全局纹理“相机颜色纹理”的引用
RenderTargetHandle renderTargetHandle;
renderTargetHandle.Init("_CameraColorTexture");

// 在ScriptableRenderPass的Configure()中配置渲染目标
ConfigureTarget(renderTargetHandle.Identifier());

// 从临时RT读取
cmd.Blit(renderTargetHandle.Identifier(), destination);
```