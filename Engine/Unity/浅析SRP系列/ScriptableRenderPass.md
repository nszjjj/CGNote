对外提供的是`Excute`函数，根据 ScriptableRenderer 维护的 pass 的队列的顺序依次调用各 pass 的`Excute`。

虽然带个 Pass，其实这个东西和着色器的 Pass 不是一个概念。实际使用中会用`ScriptableRenderPass`封装一个独立的渲染任务或阶段，例如渲染透明物体、屏幕空间反射、后处理效果都可以对应一个`ScriptableRenderPass`。

---

<mark>一次性 Pass</mark>：

可以通过在管线预留好的回调（委托）中增加一个用于添加 Pass 的方式（见下面引用1），来实现即插即用的 Pass 添加，而不必如 RenderFeature 一般
# Reference
[Tech-Artist 学习笔记：URP 中比 RendererFeature 更灵活的自定义 Pass 插入小技巧](https://zhuanlan.zhihu.com/p/550948454)