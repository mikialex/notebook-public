# Binding group management

本文中 bindgroup 指 webgpu [GPUBindGroup](https://developer.mozilla.org/en-US/docs/Web/API/GPUBindGroup)，webgpu设计上采用和vulkan类似的binding model，这里的binding相关讨论也适用于vulkan等类似图形api。本文主要探讨binding管理方式的最佳实践。

采用gles风格作为主要图形api的渲染器，在迁移到类vulkan binding model的API过程中，其中要面对的一个新的概念就是bindgroup。绘制，即录制renderpass commands的过程中，资源实例（指uniform、storage buffer，texture）不直接绑定到render pipeline实例，而是通过bindgroup实例进行绑定。相比之前的gles风格的api，在工程上形成了新的成本

## bindgroup的划分策略

所谓划分策略就是指：完成一个draw需要按顺序绑定若干资源，需要将这些资源先分组到bindgroup，划分策略就是指该分组策略。

划分策略的核心目标是

- minimize 录制pass时bindgroup的切换成本 （A）
- minimize bindgroup的维护成本（内存和更新性能） （B）
  - bindgroup的内存成本正比于bindgroup的size（逻辑上是显然的）

划分策略主要考虑的具体事宜是

- 划分的边界和粒度
- bindgroup的bind顺序

### 考虑边界和粒度

假设针对场景的每一个可绘制单元（即底层实现上依赖一个独立drawcall进行绘制， draw unit）的给定的每一种**渲染路径**都创建单独的1个bindgroup。假设场景里有n个单元，绘制需要m个不同的效果（effect），每个效果需要b个子pass （pass），然后场景有v个viewport （camera）。

所谓的**渲染路径**也存在于drawunit内部，通用的3d场景结构，drawunit是具备内部结构的：drawunit由node，model组合，model由material mesh 组合，material由texture和uniform组合，mesh由若干vertex buffer组合。

目标AB是矛盾的。完全追求目标A，不考虑B做法是per draw path + draw unit维护唯一的bindgroup：需要维护 `n * m * b * v`个bindgroup，绘制时需要`n * m * b * v`次bindgroup切换。

完全追求目标B，不考虑A的另一个做法是：为 draw unit，effect，pass，camera各自维护bindgroup，即需要维护` n + m + b + v `个bindgroup，绘制时需要`n * m * b * v`次bindgroup切换。场景一般实际情况是，n是非常大的，m，b，v都是较少的。

从维护成本和内存使用的角度看，**边界和粒度的决策逻辑是，判断构成渲染路径上的节点或者节点组合，是否数据上和其他大部分渲染路径是share的。如果是那么就有必要独立成一个bindgroup。**

重复的binding信息不需要在每一个不同的渲染路径上重复，所以可以节约内存。而维护成本下降是因为变更导致的bindgroup重建范围因为bindgroup切分而有效控制

### 考虑binding顺序

以vulkan后端为例

- 切换pipeline，bindgroup的绑定状态依然有效
- bindgroup设置会导致靠后的binding需要重新validation

这是为什么webgpu的binding perfer将低切换频率的binding优先设置的原因。

即：**渲染路径上share程度越高，那么binding的优先级也越高。** 当然，正确的考虑binding顺序的前提是正确的切分binding，所以正确的切分binding也包含了minimize binding 切换成本的目的。

## 上层框架的binding控制设计

根据上述分析，切分和排序binding的依据，实际取决于场景数据如何被组合和使用的（或者说share的程度），而和场景数据本身是什么关系不大。

比如为mesh，material实现setup pass逻辑时，是不能确定mesh或者material本身是否应该独立为一个binding。但是为camera，或者pass effect实现setup pass逻辑时，大概率是可以确定他们要独立为binding的，因为他们总是在渲染路径上被高效的share。

所以观点是：**渲染框架要将binding的划分（粒度）和排序（优先级）控制，作为一个独立的编程目标，而不能耦合在具体的场景组件的pass逻辑实现上。**

一般来说上层框架总是会有逻辑让用户编程描述渲染路径，所以在上层框架的设计上，可以在这个点来让用户注入binding控制逻辑：比如最简单的，用户提前知道场景特性，那么用户可以静态的决定binding。更加高级做法可能是，框架能够自动的根据运行时统计数据，分析渲染路径特征，自动的根据不同的场景特性，优化出最佳的binding设计。

## 隐式bindgroup 管理

工程上一个目标是尽可能让用户不用关心bindgroup的创建和更新细节。使得在调用api的表达资源绑定时，只关注于资源绑定到管线，让框架自动的处理bindgroup创建和维护的细节，达成这个目标是较容易的，但是要付出性能和内存代价。一般而言，隐式bindgroup带来的ergnomics提升我认为值得付出这样的代价。

### bindgroup cache管理

实现隐式管理，核心在于bindgroup cache的维护，其主要争议之处在于，如何清理失效的bindgroup缓存。

一种传统的，易于实现的，也是我实现的做法是：

1. bindgroup weak ref 所有被bind的资源
2. 被ref的资源 strong ref 被bind的bindgroup
3. 当任何资源drop之后，删除所有被ref的bindgroup
4. 当bindgroup显式drop时，清理被ref资源的bindgroup reference

这个做法不是最优的。因为

- 双向share结构一般性能上都不太好
- 内存问题
  - 引用关系是多余的（场景上也有多余的引用关系）
  - bindgroup的生命周期被延长了，假设某资源不仅被scene own，即便该scene再也不reference该资源，因为该资源总是不drop，所以ref的bindgroup的生命被延长到了scene drop时（即bindgroup cache 整体drop）

从性能和消耗上考虑的最优解，还是显式的增量reactive维护bindgroup缓存，直接mapping场景渲染路径到binding，使得

- 上述重复的reference关系复用了reactive框架内的引用缓存
- 显式的传播change，在最正确的时间更新和drop binding缓存
