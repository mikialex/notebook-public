# 场景数据虚拟化

本文主要讨论通过场景数据交换系统的实现框架。

交换系统主要目的是通过 动态控制加载数据量，优先呈现对于用户更加重要的数据，来显示超过运行内存/显存的场景。

## 调度的基本框架

对于某种场景单元K，给定调度的输入：

- 完整场景的K的集合
- K 调度重要性 weight的集合
- 相关元数据集合。其中元数据指K加载完成后的retain数据量，和加载过程中的峰值数据量。

这些输入中，可以通过额外的增量变更信息以改进性能。其中元数据部分可以认为是不可变数据，可以简化实现。

调度器能够根据上述input，来决定retain set 和loading set。调度实现一般是给定limitation，根据weight来排序，结合某些防抖动策略来改进行为。防抖动策略的实现应该作为独立模块。

控制retain set 和loading set产生实际的控制动作，需要调度器能够操作调度的对象，或者说依赖一个被调度对象的控制器。该控制器能够执行实际的load(return Future)和unload操作。调度器本身不关心被调度对象的IO实现。

实现weight set的计算逻辑就是实现实际调度策略，因为权重直接影响了调度结果。

weight set以K寻址，但是场景调度的决定单元可能并不是K。比如一般来说对于场景我们会从scene model入手计算其重要性，即scenemodel weight。但是我们要调度的K，可能是texture。从 scene model 到texture 其中存在多层的多对一关系。所以scenemodel weight需要以某种方式transform到texture weight上。transformation可以考虑weight transformation实现为简单的多对一add逻辑。

调度实现可以针对不同的资源类型同时存在多个，在顶层配置每一块调度器的limitation。比如在整体上限确定的情况下，如果场景富几何但是贴图简单，那么应该提高几何部分的配给，反之亦然。

## device调度实现

weight set一般和相机强关联，所以weightset的计算一般是全量的。所以这样的计算应该考虑采用计算管线实现，所以，weight transformation也需要通过计算管线实现。调度器异步的从device读取调度结果执行调度动作。

细节实现分析：

- weight计算：per scene model per thread dispatch，计算sm的重要性。
- weight传播：per scene model  per thread dispatch，atomic add material weight buffer(s, 可能是多种material)的weight table。同理再per material per thread dispatch， atomic add texture weightbuffer
- 调度器中 weight 排序和已有set 查找是可以在device上高效实现的

基于图形管线本身的基本性质，实际上texture的调度并不会通过上述通过scene model方式进行，原因是通过图形管线我们可以精确的直接得到每一个texel的可见性，或者每一个texture/texture range的可见性，以此来支持传统的virtual texture。可以认为这个流程类似于计算管线的weight计算。同样的，因为不同屏幕像素可以绘制到同一个texture/texel，所以为了更好的调度结果，weight “求和”理论上也是需要的，而实际上weight和是没用的，因为texture的budget一般是足够的，但是去重是必要的，而求和其实就是强化版本的去重。

## LOD的调度

考虑某种传统的LOD方案：比如一个scene model的 mesh有不同的版本，根据相机距离和某些元数据来切换使用哪一个版本。这种情况，可以依然采用上述的weight控制框架实现。只需要修改weight计算部分的逻辑。调度单元是mesh的level，所以per  scene model per thread dispatch，计算距离和合适的level，直接访问level的weight table atomic add weight。

考虑类似hierarchy LOD的方案，比如mesh lod graph中的meshlet作为基本的调度单元。其实就是上述dispatch的复杂版本：某种level决定逻辑决定了某个draw instance的调度单元的可见性及进一步的重要性，然后atomic add求和weight到实际的调度单元上。

## 元数据的调度

元数据的调度，是指一种嵌套的调度。支持调度需要一定的元数据存在，当被调度的对象粒度细，绝对数量大，则元数据本身也具备一定内存消耗。对于支持真正的比如城市级别的超大规模场景，用于支持调度的元数据本身也可以认为是被调度的目标。由此，外层调度的结果决定了内层调度的空间。

## 地址转化

调度输入假设不是稀疏的，但是调度的结果一定是稀疏的。因为要最小化内存使用，所以实际retain的数据不可能是稀疏的，需要采用另一套方式寻址。资源总是需要以input的space进行寻址，用户认为原始input数据实际都存在，但是实际上物理它不一定存在，从虚拟地址input，到实际物理数据的需要做地址转化。

因为地址转化是高频操作，所以必须以最小代价完成。所以实现很简单一般是直接用虚拟地址随机的访问物理地址。没有就是没数据。

## 渲染LOD和数据LOD的关系

本文主要讨论的是数据lod，即避免数据量超过平台存储容量，以避免崩溃。
另一类技术是渲染lod，指避免实际的渲染workload超过平台性能容量，以保证渲染速度。

对于超大规模场景，需要前置数据LOD控制渲染LOD的元数据规模。渲染lod的实现依赖元数据的支持，特别是比较高级的例如mesh lod graph等技术，其元数据量是巨大的。这里可以利用上文「元数据的调度」的想法使得支持渲染lod的元数据本身被数据LOD机制所管理。

渲染LOD本身可以实现数据LOD（除了支持渲染LOD的元数据）。因为其本身的特性（post sort result）决定了其具备更好的调度效果，所以在有渲染LOD可用的情况下，应该尽可能的使用渲染LOD的结果指导数据LOD的调度，而并非自行实现比如基于假设的weight info调度逻辑。
