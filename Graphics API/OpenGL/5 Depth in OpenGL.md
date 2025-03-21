# 着色器变量

## 输入类型

一般情况下着色器有两种输入，我直接从网上复制一段着色器代码来做示例：

```glsl
#version 440 core

layout (location = 0) in vec3 position;
layout (location = 1) in vec2 texCoords;
layout (location = 2) in vec3 normal;
layout (location = 3) in vec3 tangent;

layout (std140, binding = 0) uniform Matrices {
  mat4 projection;
  mat4 view;
};
uniform mat4 modelMatrix;
uniform mat4 lightSpaceMatrix;

out VertexData {
    out vec2 TexCoords;
    out vec3 FragPos;
    out mat3 TBN;
  out vec4 FragPosLightSpace;
} vertexData;

void main() {
    vertexData.TexCoords = texCoords;
    vertexData.FragPos = vec3(modelMatrix * vec4(position, 1.0));

    // Construct TBN matrix
    vec3 T = normalize(vec3(modelMatrix * vec4(tangent, 0.0)));
    const vec3 N = normalize(vec3(modelMatrix * vec4(normal, 0.0)));

    // Gram-schmidt process (produces higher-quality normal mapping on large meshes)
    // Re-orthogonalize T with respect to N
    T = normalize(T - dot(T, N) * N);
    // Then calculate Bitangent
    const vec3 B = cross(N, T);

    vertexData.TBN = mat3(T, B, N);

    vertexData.FragPosLightSpace = lightSpaceMatrix * vec4(vertexData.FragPos, 1.0);

    gl_Position = projection * view * vec4(vertexData.FragPos, 1.0);
}
```

可以看到有两种输入，一种是顶点属性，一种是 uniform。

对于顶点属性，`layout (location = 0)`指明了属性的 loaction，他们的值和顶点属性指针相对应，通过`glVertexAttribPointer`设置。

对于 uniform，亦分为两种情况。分别是 uniform 变量和 uniform 块。使用 uniform 块的时候其声明类似于结构体，并且不同于 uniform 变量，uniform 块需要使用绑定点管理，所以在其声明中需要显式指定`binding = 0`或者其他值。

对于其余正常声明的 uniform 变量，看似没有`location`，实则会隐式分配一个（当然确实也可以显式手动指定）。uniform 变量的 loaction 是 OpenGL 在链接着色器程序时分配的，由于它和顶点属性属于不同的资源，所以 loaction 可以重复。

## 生命周期

顶点属性基本上不考虑生命周期，因为数据源自 VBO，VBO 只要还在就能读到，而 VBO 又归 VAO 管理，所以它的生命周期和 VAO 一致。

每个着色器程序对象都有自己的 uniform 变量集合，在`glUseProgram`激活 Program 对象后，前一个 Program 对象的uniform 变量将不再有效。

这意味着在使用一个新的着色器程序的时候不需要单独设置顶点属性（因为设置了 VAO），但是 uniform 变量还是需要重新设置的。

# 设计哲学

[CSDN - Vulkan与OpenGL对比——Vulkan的全新渲染架构](https://blog.csdn.net/u011686167/article/details/122914604)

OpenGL 的架构分为三层：应用层、驱动层、GPU层。实际上先前的文章中已经隐约探寻到了这个架构的雏形，因为总是在关注用户在 CPU 上做了什么，驱动程序做了什么和 GPU 做了什么。

# 开销

参考链接：来自 NV 的 Cass Everitt 与 John McDonald 在 2014 年的 Steam Dev Days 上的讲座，其中谈论了 OpenGL 的一些开销问题

[YouTube - Beyond Porting: How Modern OpenGL Can Radically Reduce Driver Overhead (Steam Dev Days 2014)](https://www.youtube.com/watch?v=-bCeNzgiJ8I)

---

<img src="file:///D:/CGNote/Graphics%20API/OpenGL/img/cost_of_state_changes.jpg" title="" alt="状态切换开销" data-align="center">

节选自上述视频，展示了一些切换的开销。关于开销需要了解到的一些共识包括：

1. 状态信息需要从 CPU 传递给 GPU。

2. 状态切换要等待流水线所有正在进行计算的单位都得工作完（同步开销）

着色器切换会导致重新读取 uniform变量、纹理或buffer绑定

一般来说，render target 和 shader program的切换开销最大，uniform 变量的更新开销最小，vertex buffer 和纹理的切换开销居中
