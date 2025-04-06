# Intro

根据 RTR4 所说：布料往往具有不同于其他类型材质的微观几何结构，根据织物类型的不同，它可能还会具有高度重复的编织微观结构、从表面垂直突出的圆柱体（线），又或是二者都有。由于布料表面独特的特征外观，因此通常需要使用专门的着色模型来进行渲染，这些独特外观包括：各向异性的镜面高光，粗糙散射（光线在突出的、半透明的纤维中发生散射，从而导致边缘明亮的效果）， 甚至颜色也会随着观察方向的变化而变化（这是由织物中不同颜色的线所引起的）。

这意味着传统的 Microfacet BRDF 并不能准确表示织物的一些特点，确切的说，织物对光线的反射行为和普通微表面不一样。
# Implementation

首先传统 Microfacet BRDF 是只能处理 Specular 的，毕竟$F$项控制的是参与镜面反射的比例。然后现代引擎对织物的 BRDF 是基于微表面理论改的，那也就是说 Ashikhmin BRDF 或者 Charlie BRDF 都是只能处理非 diffuse 部分。

那第一步肯定是补一个 diffuse 项，然后因为布料的物理性质，还需要一个 sheen 项，其物理依据有两种：

- 逆向反射主导（如天鹅绒）：光线在布料表面垂直纤维方向的微观结构（如圆柱形纤维）上发生反射时，部分光线会沿入射方向返回，形成边缘亮光。
- 次表面散射主导（如棉布）：光线穿透纤维表层后，在极短距离内（≈纤维直径尺度）被散射并重新逸出表面，表现为柔和的边缘亮光。

实际的引擎解决方案会让 sheen 的能量来源源自 diffuse。所以整体结构大概是这样的（有可能 sheen 不会单独拿出来而是体现在剩余那两项中）：

$$
f_{\text{total}}(\omega_i, \omega_o) = \underbrace{(1 - F(\theta_d)) (1 - k_{\text{sheen}}) f_{\text{diffuse}}(\omega_i, \omega_o)}_{\text{修正的漫反射}} 
+ \underbrace{f_{\text{specular}}(\omega_i, \omega_o)}_{\text{镜面反射已含F}} 
+ \underbrace{(1 - F(\theta_d)) k_{\text{sheen}} f_{\text{sheen}}(\omega_i, \omega_o)}_{\text{纤维散射}}
$$


然后关于一些叫法：$G$项也被称作$V$项；NDF 也被成为$D$项

## Microfacet Sheen BRDF

> 参阅：[虚幻5渲染编程(材质篇)[第三卷: Microfacet Sheen BRDF in UnrealEngine] - 知乎](https://zhuanlan.zhihu.com/p/396004742)

主要是因为是在 Microfacet BRDF 基础上修改的，主要涉及 Ashikhmin BRDF 和 Charlie BRDF，它们更适合表达织物的物理性质，并且增加了 Sheen 项。

不过当使用 Charlie 分布（CharlieD）或 Ashikhmin 分布（AshikhminD）时，是不需要除`4 * (NoL) * (NoV)`进行归一化的，直接$F_r = (D*V)*F$，另一方面会对$V$项进行简化，以简化计算。不进行归一化，一方面移动端不做这个是一种计算量的简化，可以提升性能（造成的渲染结果差异肉眼观测并不显著）。另一方面$V$项和$D$项隐式含有了补偿（大致上会能量守恒，但是经验成分更多）

大概是这样的意思：

```cpp
float D = D_Charlie(roughness, NoH);  // 这里已经归一化了
float V = V_Charlie(NoV, NoL);        // 简化的Visibility，也可以是Ashikhmin
vec3  F = F_Schlick(F0, LoH);         // Fresnel项
vec3 Fr = (D * V) * F;                // 无需归一化

vec3 Fd = CalDiffuse(); // 可以是 DisneyDiffuse 也可以是 Lambert
vec3 color = Fd + Fr; // 直接相加，然后去做光照
```

### Charlie BRDF

Charlie BRDF 是由 Sony Pictures Imageworks 提出的，旨在提供一种更直观且计算效率更高的布料着色方法。其主要工作是优化了 Microfacet BRDF 的 NDF，并引入了 Sheen 项。

原论文的 sheen 是在计算出其余内容得到的颜色之后再对其整体做的。

NDF 项的公式为：

$$
D=\frac{(2+\frac1\alpha)\sin^{\frac1\alpha}\theta}{2\pi}
\tag{1}
$$
这里的$\theta$表示微表面法线$m$与宏观法线$n$的夹角，即$\theta = \arccos(n·m)$，但是实际应用中并不想承担$\arccos$的计算开销，因此如果使用了三角恒等式$\sin\theta = \sqrt{1-{(n·m)}^2}$换元则会得到这样的式子：

$$
D=\frac{(2+\frac1\alpha)(1-(n·m)^2)^{\frac{1}{2\alpha}}}{2\pi}
\tag{2}
$$

由上述公式得到的着色器实现代码入下：

```cpp
// 公式1 实现
inline float CharlieD(float roughness, float nh) {
    float invR  = 1.0 / roughness;
    float cos2h = nh * nh;
    float sin2h = max(1.0 - cos2h, 0.0078125); // 2^(-14/2), so sin2h^2 > 0 in fp16
    return (2.0 + invR) * pow(sin2h, invR * 0.5) / (2.0 * PI);
}
```

有必要指出的是，`max(1.0 - cos2h, 0.0078125)`是为了数值稳定性设计的。因为在`sin2h`趋近 0 的时候，幂运算可能产生非数值（NaN）或浮点数下溢，所以选择了一个很小的正规数，在避免出现 0 的同时，不会明显产生渲染瑕疵（可能会略微增强高光亮度）。其中`fp16`也被称作半精度浮点数
### Ashikhmin BRDF

```cpp
float AshikhminD(float roughness, float ndoth)
{
	float m2    = roughness * roughness;
	float cos2h = ndoth * ndoth;
	float sin2h = 1. - cos2h;
	float sin4h = sin2h * sin2h;
	return (sin4h + 4. * exp(-cos2h / (sin2h * m2))) / (PI * (1. + 4. * m2) * sin4h);
}

float AshikhminV(float ndotv, float ndotl)
{
	return 1. / (4. * (ndotl + ndotv - ndotl * ndotv));
}
vec3 specular = lightColor * f * d * v * PI * ndotl;
```

你会看到这个`specular`中含有`PI`，这里提一下关于这个是否乘`PI`的问题：光源的辐照度（Irradiance）到辐射亮度（Radiance）转换时，需乘 $π$ 以抵消 BRDF 的 $1/π$。

如果 BRDF 根本没带$\frac1\pi$那最终结果就不用乘，如果带$\frac1\pi$了，那要么获取光源`lightColor`预乘$\pi$，要么最后统一补。
## Engine Implementation

讲完原理之后就要看现代引擎是如何实现的。Diffuse 部分采用Disney Principled BRDF 的 Diffuse 部分。

Specular 部分：
- UE4用的是 AshikhminV + CharlieD
- HDRP：CharlieV + CharlieD

Filament 的思路是：NDF 项对 BRDF 的贡献最大，$G$项对 velvet 材质不是必须的。

## URP



# Reference

[Fabric Shader(织物，布料) - 知乎](https://zhuanlan.zhihu.com/p/595661286)
[[译]RealTimeRendering-基于物理的渲染(五)-布料BRDF - 知乎](https://zhuanlan.zhihu.com/p/351059112)
[真实角色渲染---衣服布料 - 知乎](https://zhuanlan.zhihu.com/p/63526854)
[【UE5】虚幻の布料材质渲染 - 知乎](https://zhuanlan.zhihu.com/p/641872297)
[【浅入浅出】Unity 布料渲染 - 知乎](https://zhuanlan.zhihu.com/p/178306591)
[布料着色 |Krzysztof Narkowicz](https://knarkowicz.wordpress.com/2018/01/04/cloth-shading/)
[Cloth Shading](https://www.shadertoy.com/view/4tfBzn)上篇文章的 ShaderToy 的实现
[Filament探索（N1）——Cloth specular BRDF 探索 - 知乎](https://zhuanlan.zhihu.com/p/554386277)
