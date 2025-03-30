# Intro

根据 RTR4 所说：布料往往具有不同于其他类型材质的微观几何结构，根据织物类型的不同，它可能还会具有高度重复的编织微观结构、从表面垂直突出的圆柱体（线），又或是二者都有。由于布料表面独特的特征外观，因此通常需要使用专门的着色模型来进行渲染，这些独特外观包括：各向异性的镜面高光，粗糙散射（光线在突出的、半透明的纤维中发生散射，从而导致边缘明亮的效果）， 甚至颜色也会随着观察方向的变化而变化（这是由织物中不同颜色的线所引起的）。

这意味着传统的 Microfacet BRDF 并不能准确表示织物的一些特点，确切的说，织物对光线的反射行为和普通微表面不一样。

# Implementation

首先传统 Microfacet BRDF 是只能处理 Specular 的，$F$项控制了参与镜面反射的比例。然后现代引擎对织物的 BRDF 是基于微表面理论改的，那也就是说 Ashikhmin BRDF 或者 Charlie BRDF 都是只能处理非 diffuse 部分。

传统的 Microfacet BRDF 的形式是$F·G·D/c$，其中$c$是归一化常数，在 UE 中$G$项也被称作$V$项

HDRP 的做法是这样的：$f_{fabric}=f_{diffuse}+f_{specular}+f_{sheen}$
# Reference

[Fabric Shader(织物，布料) - 知乎](https://zhuanlan.zhihu.com/p/595661286)
[[译]RealTimeRendering-基于物理的渲染(五)-布料BRDF - 知乎](https://zhuanlan.zhihu.com/p/351059112)
[真实角色渲染---衣服布料 - 知乎](https://zhuanlan.zhihu.com/p/63526854)
[【UE5】虚幻の布料材质渲染 - 知乎](https://zhuanlan.zhihu.com/p/641872297)
[【浅入浅出】Unity 布料渲染 - 知乎](https://zhuanlan.zhihu.com/p/178306591)
[布料着色 |Krzysztof Narkowicz](https://knarkowicz.wordpress.com/2018/01/04/cloth-shading/)
[Filament探索（N1）——Cloth specular BRDF 探索 - 知乎](https://zhuanlan.zhihu.com/p/554386277)