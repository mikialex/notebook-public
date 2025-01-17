# 场景过滤和剔除

## 剔除和过滤的抽象和实现

剔除和过滤的工作是类似的，剔除更多的指避免不必要的绘制以提升性能，过滤更多的指根据绘制的业务要求来筛选（不）需要绘制的物体。

基于 [basic-scene-object-model](../editor-engineering/basic-scene-object-model.md) 中的场景模型，所谓的物体我们采用scene-entity来指代。并进一步合理的认为剔除系统可以通过scene-entity在host和device上访问到该entity所有剔除所需要的数据。

整个剔除和过滤系统可以认为是一个数据流图。剔除的input，output，可以认为是这样entity的有序集合。图中流动的数据即这样有序集合。

这样的集合是抽象的，比如host应该通过entity的迭代器表达，device应该通过device-handle的 gpu迭代器表达。这个抽象是必要性，同迭代器这个设计的存在的必要性一致。避免逐步的materialization来提升性能。

流图的root input，即上游数据，是由场景提供的entity-list。比如一个scene中的所有entity。这样的维护可能是更细致粒度的，某一类list。流图的output，即具体绘制所需的数据终点。由业务需求定义。

## 场景提供增量实现

上文提到场景需要为剔除系统提供初始的list数据。这些list有哪些？

显然全部的entity应该要可以提供，因为这支持让外部剔除做任意自由的实现。

一种想法是，这样的list还需要包含特殊的，用以支持场景正确绘制的集合单元。比如一个场景内部定义了类似图层的结构，而渲染逻辑上假设约定图层应该按照一定顺序绘制。那么scene本身就需要以某种方式暴露出这些图层为单位的list。但是这个说法并不成立，因为外部的剔除系统也可以访问相关数据进行分类。

或者进一步的，让用户自由的决定哪些list由场景原始提供。比如只包含半透明的或者不透明，只包含某种flag的list。那么问题是，到底什么样的list应该由场景直接提供，而不是在剔除系统里计算得到？

这里的关键在于「直接提供」的定义，我对「直接提供」，的定义是：这些数据是被增量维护的。

由此，哪些list应该由场景直接维护的问题就是哪些list应该被增量维护的问题，那么就是什么情况下应该选择全量计算，什么时候应该采用增量计算的问题，这由实际场景决定。

对于每一帧都显然大概率要变动的数据，比如视锥剔除，那么全量计算是更合理的选择。对于相对静态的数据，增量计算是合理的。当然权衡到增量计算的内存开销，也有必要采用全量模式。

更进一步的，因为增量系统一般也是流图实现的，所以我们可以认为整体剔除体系包含了场景内侧提供的增量部分。整体流图上游部分是增量的，下游是全量的。

|             | 全量计算                                | 增量计算                                   |
| ----------- | ----------------------------------- | -------------------------------------- |
| host(cpu)   | 「host-list-graph」传统的host list过滤     | 「reactive-list-graph」reactive系统维护的list |
| device(gpu) | 「device-list-graph」传统的device list过滤 | 未实现，但理论上可以实现                           |

### Device版本全量计算的组合表达

host版本的全量剔除计算，可以简单认为是可以通过`fn(impl Iterator<Item = Entity>) -> impl Iterator<Item = Entity>` 来表达，剔除体系的组合能力，可以通过这种调用的组装来表达。

但是 device版本的全量剔除计算，并不适合采用类似的结构表达组装能力。而是需要显式的去组装一个culler的抽象。culler本身表达entity是否通过剔除。通过组装culler，来表达剔除的组装逻辑。最后显式的应用culler。

这样设计的原因是：

在device版本的materialize的实现中，list因为性能(比如减少overdraw)或者正确性(比如半透明)的原因需要保序，所以materialize的实现需要采用stream-compaction。device上虽然剔除都是并行执行的，但是stream-compaction虽然也是并行的，但是开销要高的多。所以在device版本的实现上，要尽可能少做materialize。

为了尽可能少做materialization，那么就要尽可能的合并materialization。然而如果采用`fn(impl Iterator<Item = Entity>) -> impl Iterator<Item = Entity>` 的device版本表达，内部实现是完全屏蔽的，是否有materialization，乃至具体上是无法得知的。

所以只能显式的让用户来控制materialization的调用，让用户来决定何时materialization。剔除本身只是filter，并不涉及更加自由的迭代器封装行为，所以就应该让用户转而去组合filter本身的行为，而不是迭代器。将filter组合的结果对list做显式应用。

用户应该尽可能保证只在graph出口节点做materialization，非出口节点的materialize，其必要性取决于对数据本身剔除表现的判断。对于一个流图的高级runtime，根据运行时信息自动控制materialization的策略理论上是可以实现的。

## 工程上统一device和host的剔除

为了避免剔除行为在不同渲染方式（direct和indirect）下的偶然错误区别，应该在工程上统一实现。具体将两种list合二为一，并实现同样的culler组装和materialization接口即可。

```rust
pub enum SceneModelRenderBatch {
  Device(DeviceSceneModelRenderBatch),
  Host(Box<dyn HostRenderBatch>),
}

pub trait HostRenderBatch: DynClone {
  fn iter_scene_models(&self) -> Box<dyn Iterator<Item = EntityHandle<SceneModelEntity>> + '_>;
}

pub struct DeviceSceneModelRenderBatch {
  /// each sub batch could be and would be drawn by a multi-indirect-draw.
  pub sub_batches: Vec<DeviceSceneModelRenderSubBatch>,
  pub stash_culler: Option<Box<dyn AbstractCullerProvider>>,
}

pub struct DeviceSceneModelRenderSubBatch {
  pub scene_models: Box<dyn DeviceParallelComputeIO<u32>>,
  ..
}
```
