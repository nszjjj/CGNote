# BRDF LUT

Preintegral LUT 用于高效求解渲染方程中的镜面反射积分部分，这是一张二维查找表，参数分别是粗糙度（Roughness/α）和法线与视线夹角的余弦（$n·v$）。

结合渲染方程来看：

$$
L_o(v) = \int_\Omega L_i(l)·\frac{F·G·D}{4·(n·l)·(n·h)}·(n·l)\mathrm{d}l
$$
