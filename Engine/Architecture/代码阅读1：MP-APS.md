# 引

[Github - OpenGL-Renderer](https://github.com/htmlboss/OpenGL-Renderer)

作者介绍说是使用 C++17 的现代 OpenGL 渲染器，其理念是使用起来非常简单：没有奇怪的抽象层、不规则的变量名或大量的由 2000 多行代码组成的类。

特点：

- PBR

- 并行 AABB 视锥体剔除

- XML 引擎配置

整体结构上很多东西都堆在根目录下，没有分类。

- 根目录：

# Engine

## 概述

在这个类搞了的功能如下：

- 窗口管理：引擎管理输入并将其提交给窗口系统

- 帧时间计算

- 初始化：加载配置文件，初始化窗口和OpenGL渲染器，并设置GUI系统

- 场景管理：主要是管理活动场景

- mainloop：没啥说的

- 视锥剔除：在mainloop中每tick都要调用视锥剔除函数产生提交渲染的物体的list

- 资源管理和清理：主要就是在`shutdown`方法中对各个子系统`shoutdown`

他的定位其实就是围绕核心循环做了三件事：

1. 初始化和准备阶段：主要指的是核心循环运行前置依赖的东西

2. 主循环（游戏循环）阶段：使引擎运转的更新循环

3. 清理和卸载阶段：游戏退出时的资源卸载

## 优化

Engine 类的目的就是去启动主循环，而主循环内要与各个模块打交道，所以它应当能灵活地加载和卸载各个模块。但是这里最明显的问题就是不符合开放闭合原则，比如说各个系统的加载和卸载都是写死的。

一方面建议实现`Itickable`，通过优先队列维护所有能Tick的模块并在 mainloop 中 Tick

另一方面建议实现`IMoudle`，对于加载和卸载统一管理。按道理模块间不应当存有依赖，但是不同层之间可能有依赖，需要额外注意加载和卸载的顺序。

- 日志系统

- 事件系统：`IEventListener`

- 性能监控：

- 状态管理：菜单、游戏、暂停等

# RenderSystem

核心的 RenderSystem 相当的简洁，

对外的内容主要就是初始化的`Init`和每帧调用的`Update`，这个东西在渲染循环中不断调用。

不过问题也相当明显，就是他写死了只能用PBR材质

# 摄像机系统

# Mesh

Assimp 导入的模型肯定是无法被 OpenGL 直接使用，所以统一实现了 Mesh 结构体对导入的网格数据转换为 OpenGL 可以直接使用的格式，这样可以忽略底层不同 mesh 格式的差异，对引擎提供统一的操作接口。

相当于把Assimp导入的视为中间格式，对外支持多种模型格式的导入（通过 Assimp），对内提供统一的渲染接口。

他这里是 OpenGL 引擎，所以不用考虑多渲染后端的问题。如果意图支持多渲染后端，最好是把这部分的逻辑放到各自的后端实现中：

```cpp
class Mesh {
private:
    std::unique_ptr<IRenderResource> renderResource;

public:
    void CreateGPUResource(RenderAPI api) {
        switch(api) {
            case RenderAPI::OpenGL:
                renderResource = std::make_unique<OpenGLResource>();
                break;
            case RenderAPI::Vulkan:
                renderResource = std::make_unique<VulkanResource>();
                break;
        }
        renderResource->Upload(vertices, indices);
    }
};
```

# 材质

## PBR Material

这里没啥说的，他模型渲染写死了就是 PBR 材质。正常来说其实是应该实现材质基类，然后各自继承出来。

`PBRMaterial`支持从纹理或者纯数值描述 PBR 的各项参数。如果是纯数值的话他保存的是`glm::vec3`，如果是纹理的话保存的是`unsigned int`，直接对应的就是纹理对象的id（当然其实应该是`GLuint`）

然后在`./Data/Shaders`下面有 PBR 材质的实现，PBR 采用 Microfacet 模型。Microfacet 有很多实现，这里采用的是经典的 Cook-Torrance

结合一下微表面的 BRDF 公式来看：

$$
f_r(\omega_i,\omega_o)= \frac{D(h)·G(\omega_i,\omega_o)F(\omega,h)}{4·(n·\omega_i)·(n·\omega_o)}
$$

在这里：

- 法线分布函数$D$项使用的是GGX

- 几何项$G$使用的是GeometrySmith

- 菲涅尔项$F$使用的是Schlick近似

算是一种很经典的实现

顶点着色器里主要就对输入做了些处理，包括构建TBN矩阵、坐标变换等。片元着色器内才是主要计算PBR模型的地方。

有个疑问是他的`albedo`是在sRGB空间下做的后续计算：

```glsl
const vec3 albedo = pow(texture(albedoMap, fragData.TexCoords).rgb, vec3(2.2));
```

Bloom 也放在这里了，其实后处理效果应该单独放在一个stage

# 资源

资源部分由ResourceManager类统一管理。

# 其余技术细节

使用`std::string_view`传字符串参数

使用`std::shared_ptr`管理对象
