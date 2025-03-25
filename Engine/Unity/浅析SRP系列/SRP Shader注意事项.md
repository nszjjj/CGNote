## CBUFFER

CBUFFER 是 Unity 的着色器能够参与 SRP Batcher 的必要条件。对于完全自定义 SRP，需要在着色器中手动声明这些 CBUFFER，Unity 不会主动提供这些数据，他们需要在自定义的 SRP 中自行提供。

为什么 URP 的着色器代码不需要手动填充？因为那些变量已经在`ShaderVariablesFunctions.hlsl`中定义，并且相关着色器包含了它。或者引用`Core.hlsl`，这样会间接引用前面那个。