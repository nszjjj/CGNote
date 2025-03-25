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

小贴士：Unity 提供了`CommandBufferPool`以优化性能，但是不要将 `ProfilingScope` 与通过 `CommandBufferPool.Get("name")` 获取的命名 `CommandBuffer` 混合使用。命名 `CommandBuffer` 在执行时会自动关闭其自身的 Profiling Scope，这会导致 `ProfilingScope` 标记孤立。