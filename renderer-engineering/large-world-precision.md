# 改进大场景渲染精度

我近期在渲染器上做了大场景渲染精度改进的实现，此文主要讨论一些设计上的取舍。大场景的渲染精度是一个常见问题。这里所谓的大场景不是场景的数据量大，而是主要指场景对象距离坐标原点过远，导致采用单精度浮点数存储的position，丧失数值精度，进一步导致的渲染上的artifact问题。

进一步的我们限定这个问题主要是由于渲染器的可编辑单元具有不适合 f32类型存储的translation导致的精度问题，而不考虑当个可编辑单元的local space mesh尺寸过大或距离过远导致的精度问题。后者本身属于不合理的场景数据，可以考虑通过拆分mesh来转化为前一个问题解决。

## host侧的改进

在一个理想的情况下，可以统一采用f64双精度浮点来解决此问题。f64相比f32，只会在宇宙的尺度上遇到精度问题。无法采用f64的原因是

- gpu上f64的支持是少见的，兼容性会非常差
- 即使gpu上支持f64，在现在的消费级产品上，其性能是非常差的
- 采用软件实现的f64，是非常麻烦，性能也是非常差的
- 统一采用f64，内存和带宽消耗会double，具体影响取决于实际场景规模

因为这些约束，在gpu上使用f64是不现实的。

虽然device(gpu)上不能使用f64， 但是在host上，为了解决这个问题，使用f64是必要的。因为

- 在常见的64位cpu上，f64和f32计算性能没有差别
  - 实际上f32倾向于要稍微快一些，这主要是数据量导致的带宽上/cache使用的差别
- 你总是需要一个充分大的数据类型来存储充分大的world space的数据（world position），因为f64是硬件支持的，也是充分大的，那么采用f64来存储是唯一的理智选择

所以一个原则是，**在host端，凡是涉及world space position的数据，都必须通过f64存储。** 一般的，可以认为host端的world matrix需要f64类型。这方面可考虑一个特别的affine transform类型，其rotation scale部分是f32，position是f64。这一考虑属于实现成本上的取舍，我目前没有采用。

另一个问题是，场景API，比如node的 local transform 是否应该应该采用f64？考虑这样的场景：某个场景更新逻辑依赖world position（f64数据），然后需要设置某个场景对象跟随该world position。那么如果node 的local transform不具备f32的精度，那么在场景设置上就产生了精度不足的问题，导致在距离原点极远处出现跳跃的现象。这样的逻辑其实是常见的，而且用户总是会把local当world来使用。所以我认为是必要的。出于简单实现的考虑，local matrix可统一修改为f64。

## device 侧的改进

因为device上没有f64可用，而world space的数据要具备精度的表示依赖于f64，所以意味着**device上的计算绝对不能采用，以及touch到world space的数据**。

对于渲染而言，一定是离相机越近的内容越重要，越需要更高的精度。所以shader的几何和着色计算，应该采用跟随camera的space， local to camera的转换在host端通过高精度的类型实现，避免在device上touchworld space的数据。

一般的做法是采用camera space / view space作为shader的几何和着色计算空间来解决精度的问题。比如threejs就是这样做的。（有趣的是threejs，因为是js，所以其host的数据就是f64）。这样的做法虽然可行，但不是最佳的。其致命缺陷有

- host侧开销高
  - 需要host提供model view matrix，意味着 camera的change需要在host侧重新计算所有的可编辑单元的model view。对于海量可编辑单元的场景，其开销是不可接受的。又因为model view matrix是需要f64，那么通过计算着色器来加速也是不合适的
  - 对于 gpu driven的剔除，这样的change需要重新计算viewspace的box数据，其成本是不可接受的，并且因为旋转的存在，其效果会变差
  - camera的change导致device上的model view总是需要被大量重新同步。毕竟数据都变了。
- 实现成本高
  - 即便camera没有变化，那么对于多视图的情况，为了避免camera的change导致device上的model view总是需要被大量重新同步的问题，需要per camera的维护gpu的resource。一般在工程上我认为这种做法是糟糕的

所以另一个核心诉求出现了：**避免camera变化导致的任何场景级的数据变更**。即普通的场景数据不能和camera有关系。

所以理想的解决方案是：还是必须要想办法在shader内使用高精度的world space数据，来实现camera space的渲染。一种不依赖f64的唯一的合理做法是存在的，其核心做法是：

首先应该采用camera的 world translation only space作为 shader的几何和着色空间。这一做法的好处是：

- 跟随camera，满足上述精度表现要求
- 不需要shader内计算model view，避免了计算成本
- 实际上类似于“model view”的计算需求变成了两个（camera和model）高精度world position的减法操作
- 高精度world position本身是不需要和camera联动的，避免了上述问题

然后两个高精度position的减法，对于shader内是，不依赖f64，是可以高效实现的。比如

- 采用两个f32来表示一个高精度f64
  - 第一个f32直接由f64 clamp精度而来
  - 第二个f32表示被clamp的f64和f64的绝对差别。因为这个差别很小，所以能够充分利用极小数的浮点精度。
  - 当然两个f32的精度还是不如f64的，因为mantisa的位数要少
- 这样的f32 pair的高精度position实现减法是简单的，低成本的
  - 分别相减然后再加起来
  - 对比单纯的减法，多一个加法一个减法。所以这种做法，对于**原本的任意world space的数据，都是多一个加法一个减法，其计算上的额外开销是可以接受的。**

具体而言。对于所有worldspace的数据，剥离其中position的部分采用两个f32（high percision translation: hpt）传递到shader，非position的 roation scale 作为mat3 f32传递到shader。先通过none translation world matrix， 移动local space的geometry顶点到none translation world space，再计算camera的hpt减world的hpt，得到none translation world space到none translation camera space（即我们的渲染空间）的offset，matrix本身是translation rotation scale的组合。的顺序，所以加上这个offset即可。

以上的做法也同时需要考虑非node管理的world space position数据并特别处理，比如用以gpu剔除的world space bounding，lighing的数据等。

以上的做法需要关闭float reorder相关的优化。除了metal， dx12和vulkan默认是没有这样的优化的。

我目前没有看到其他更加合理的做法

- 其引入的开销是最小的必要的
- 没有特别的实现难度
- 不依赖特别的硬件平台能力。

同时我认为

- 大场景支持是非常必要的，是一个渲染器的基础能力。
  - 物体移动的较远就绘制质量出现问题是不可接受的
  - frame的输出精度进一步的影响下游比如后处理的效果呈现，某些精度导致的质量问题可能在光照/着色阶段不可见，但是可能其他特别的效果上是可见的
- 着色空间的改造的迁移成本很高

所以我认为 制作一个全局的开关来切换开启高精度大场景渲染是没有必要的。所以渲染器应该统一默认采用这种方式来进行场景绘制。

## 参考

- 印象中最早是看到pbrt这样做 <https://www.pbr-book.org/4ed/Cameras_and_Film/Camera_Interface>
- 目前主流游戏引擎比如unreal和unity都是worldspace着色，默认是不解决此问题，但是有一些optional solution
  - unreal的做法类似 <https://www.youtube.com/watch?v=HdIVSwdXQQM&list=PLLaly9x9rqjsXLW1tMFruyh_657sh8epk&index=7>
  - <https://dev.epicgames.com/documentation/en-us/unreal-engine/large-world-coordinates-rendering-in-unreal-engine-5>
  - <https://godotengine.org/article/emulating-double-precision-gpu-render-large-worlds/>
- webgpu 关闭fast math的issue <https://github.com/gpuweb/gpuweb/issues/2076>
