# Intro

根据 RTR4 所说：布料往往具有不同于其他类型材质的微观几何结构，根据织物类型的不同，它可能还会具有高度重复的编织微观结构、从表面垂直突出的圆柱体（线），又或是二者都有。由于布料表面独特的特征外观，因此通常需要使用专门的着色模型来进行渲染，这些独特外观包括：各向异性的镜面高光，粗糙散射（光线在突出的、半透明的纤维中发生散射，从而导致边缘明亮的效果）， 甚至颜色也会随着观察方向的变化而变化（这是由织物中不同颜色的线所引起的）。

这意味着传统的 Microfacet BRDF 并不能准确表示织物的一些特点，确切的说，织物对光线的反射行为和普通微表面不一样。

# Implementation

首先传统 Microfacet BRDF 是只能处理 Specular 的，毕竟$F$项控制的是参与镜面反射的比例。然后现代引擎对织物的 BRDF 是基于微表面理论改的，那也就是说 Ashikhmin BRDF 或者 Charlie BRDF 都是只能处理非 diffuse 部分。

那第一步肯定是补一个 diffuse 项，然后因为布料的物理性质，还需要一个 sheen 项，其物理依据有两种：

- 逆向反射主导（如天鹅绒）：光线在布料表面垂直纤维方向的微观结构（如圆柱形纤维）上发生反射时，部分光线会沿入射方向返回，形成边缘亮光。
- 次表面散射主导（如棉布）：光线穿透纤维表层后，在极短距离内（≈纤维直径尺度）被散射并重新逸出表面，表现为柔和的边缘亮光。

实际的引擎解决方案会让 sheen 的能量来源源自 diffuse。所以整体比例是这样的：

$$
f_{\text{total}}(\omega_i, \omega_o) = \underbrace{(1 - F(\theta_d)) (1 - k_{\text{sheen}}) f_{\text{diffuse}}(\omega_i, \omega_o)}_{\text{修正的漫反射}} 
+ \underbrace{f_{\text{specular}}(\omega_i, \omega_o)}_{\text{镜面反射已含F}} 
+ \underbrace{(1 - F(\theta_d)) k_{\text{sheen}} f_{\text{sheen}}(\omega_i, \omega_o)}_{\text{纤维散射}}
$$


然后关于一些叫法：$G$项也被称作$V$项；NDF 也被成为$D$项

HDRP 的做法是这样的：$f_{fabric}=f_{diffuse}+f_{specular}+f_{sheen}$

## Modify Microfacet BRDF

前面已经说了 Ashikhmin BRDF 和 Charlie BRDF 都是对 Microfacet BRDF 的修改，使之更加适合表达织物的物理性质。

### Charlie BRDF

Charlie BRDF 是由 Sony Pictures Imageworks 提出的，旨在提供一种更直观且计算效率更高的布料着色方法。其主要工作是优化了 Microfacet BRDF 的 NDF，公式为：

$$
D=\frac{(2+\frac1\alpha)\sin^{\frac1\alpha}\theta}{2\pi}
\tag{1}
$$
这里的$\theta$表示微表面法线$m$与宏观法线$n$的夹角，即$\theta = \arccos(n·m)$，但是实际应用中并不想承担$\arccos$的计算开销，因此使用了三角恒等式$\sin\theta = \sqrt{1-{(n·m)}^2}$，这样得到：

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
    return (2.0 + invAlpha) * pow(sin2h, invR * 0.5) / (2.0 * PI);
}
```

有必要指出的是，`max(1.0 - cos2h, 0.0078125)`是为了数值稳定性设计的。因为在`sin2h`趋近 0 的时候，幂运算可能产生非数值（NaN）或浮点数下溢，所以选择了一个很小的正规数，在避免出现 0 的同时，不会明显产生渲染瑕疵（可能会略微增强高光亮度）。其中`fp16`也被称作半精度浮点数
### Ashikhmin BRDF


## Engine Implementation

讲完原理之后就要看现代引擎是如何实现的。Diffuse 部分采用Disney Principled BRDF 的 Diffuse 部分。

Specular 部分：
- UE4用的是 AshikhminV + CharlieD
- HDRP：CharlieV + CharlieD

# Reference

[Fabric Shader(织物，布料) - 知乎](https://zhuanlan.zhihu.com/p/595661286)
[[译]RealTimeRendering-基于物理的渲染(五)-布料BRDF - 知乎](https://zhuanlan.zhihu.com/p/351059112)
[真实角色渲染---衣服布料 - 知乎](https://zhuanlan.zhihu.com/p/63526854)
[【UE5】虚幻の布料材质渲染 - 知乎](https://zhuanlan.zhihu.com/p/641872297)
[【浅入浅出】Unity 布料渲染 - 知乎](https://zhuanlan.zhihu.com/p/178306591)
[布料着色 |Krzysztof Narkowicz](https://knarkowicz.wordpress.com/2018/01/04/cloth-shading/)
[Filament探索（N1）——Cloth specular BRDF 探索 - 知乎](https://zhuanlan.zhihu.com/p/554386277)