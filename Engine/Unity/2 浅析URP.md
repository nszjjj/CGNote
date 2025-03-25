URP 的工作流程相对来说较为精简，但是也分为两种，分别是前向渲染和延迟渲染。

## ForwardRenderer

1. 阴影渲染：调用 `ShadowCasterPass` 渲染阴影贴图。
2. 不透明物体渲染：调用 `DrawOpaqueObjectsPass` 渲染不透明物体。
3. 天空盒渲染：调用 `DrawSkyboxPass` 渲染天空盒。
4. 透明物体渲染：调用 `DrawTransparentObjectsPass` 渲染透明物体。
5. 后处理：调用 `PostProcessPass` 执行后处理效果。
