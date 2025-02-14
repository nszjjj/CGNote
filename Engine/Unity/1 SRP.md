# 基础概念

## 绘制机制

在没有其它干涉的情况下，Unity的GameObject通过相关绘制组件（例如 `MeshRenderer`、`SkinnedMeshRenderer` 等），与管线交互进行绘制。每个GameObject通常都附加了一个或多个渲染组件。这些组件包含了渲染所需的信息，例如材质、Mesh或者Transform。相关的渲染组件会将渲染过程中用到的信息传递给管线，该过程由引擎内部自动完成。

额外地，mono脚本可以修改渲染组件的绘制参数，从而影响 GameObject 的渲染结果。在`Update`或是`LateUpdate`，都属于GameLogic Stage，是先于Scene Rendering的，所以信息肯定可以更新过去。但是还有些阶段如`OnPostRender`肯定是更新不了会等到下一帧的。

## Core RP

SRP依赖Core RP Package，可以在包管理中找到并安装。由于URP和HDRP都是依赖于SRP构建的，因此在URP或者HDRP管线的工程中也会包含这个包。

通常，使用一个Pipeline Asset（一种ScriptableObject，它包含了渲染管线的配置和设置）来描述一个SRP

## Forward SRP

## Deferred SRP
