# Reverse-Z 相关问题

#### 为什么必须要做在projection matrix内，而不是对结果 clip position直接reverse z？

这种集成方式看似很简单，但是是错误的

- 如果采用普通的projection matrix，拿到的clip position的精度已经是损失过的。
- 如果实现于projection内，渲染器中reprojection实现不需要做任何修改

#### 如何集成projection的修改？

目前为了支持不同api不同的ndc，目前在工程上的设计有

- 底层数学库的projection统一要求输出为opengl标准ndc
- 渲染器层面，projection matrix 统一的左乘opengl ndc到目标设备ndc的矩阵

可以利用此设计，将reverse添加到该修正矩阵中

#### 是否有必要做成开关？

有必要，因为

- 目前高跨平台要求的应用需要支持gles300（虽然在这种情况下extension一般是可以work的）
- 可以对比验证问题
- 可以改进项目的质量（在某个层面统一的对depth的全局读写测试，ndc的配置做控制）

#### 是否有必要支持其他改进depth精度分布的技术

比如log depth： [three log depth exampel](https://threejs.org/examples/#webgl_camera_logarithmicdepthbuffer)

个人认为log depth目前还没有实际使用的价值。因为

- [据说](https://www.reddit.com/r/GraphicsProgramming/comments/1cqt0yi/logarithmic_depth/) 32bit reverse z 效果好于log depth
- log depth disable early z，要付出性能代价
- log depth 需要渲染器真正的将depth写入视作一个custom的encoder和decoder，并实现相关的扩展机制，引入了显著的复杂度。而reversez可以继续依赖projection matrix来做encoder和decoder

比如分层 depth渲染。

这个做法，基本上是靠渲染成本翻倍提升，来获取精度的翻倍提升。只在特殊场景有用，实现较为复杂。
