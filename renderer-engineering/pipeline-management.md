# Pipeline management

本文讨论一个典型的3d场景结构中，渲染器对(render/compute)pipeline 创建，销毁，获取等相关机制的最佳实践。本文的 render pipeline 指 [webgpu renderpipeline](https://developer.mozilla.org/en-US/docs/Web/API/GPURenderPipeline)。

gles api中的shader module和pipeline大致属于一类api，所以这里的pipeline管理对于gles等旧时代图形api中的shader module同样适用。

## Pipeline lifetime

### 渲染时按需创建，还是在渲染前全部预创建？

大部分游戏都采用预创建并缓存pipeline， 在游戏初次运行，或者升级显卡驱动后，都需要等待重新编译shader才能运行。预创建的优势是避免了运行时立即创建造成的卡顿。根据实际游戏的运行对比（某些引擎可以控制pipeline是按需创建还是预创建），采用按需创建的方式，会造成游戏在前1-5分钟的体验，或者在首次触发到某些特效时，造成非常严重的卡顿，严重影响用户体验。

场景中不同pipeline的组合方式可能不是完全自由的，但其组合的数目时非常大的，一般并不能枚举出所有pipeline组合的实例并预创建。而是需要挑选其中部分进行预创建，预创建一般有这样的实现方式：

- 由人工制作大量典型场景的相机用例，运行pipeline收集程序，来决定哪些pipeline需要创建
- 图形框架自动的排列组合所有shader变体组合，并由人工做裁剪（因为组合数量是不可控的）

如果运行时发现pipeline不存在，则当场按需创建。某些引擎会尽可能保证正确性的情况下，特别的采用一些降级的pipeline实现，保证基本的功能正确性，牺牲效果的准确性，以避免卡顿。

预创建pipeline的过程有可能非常耗时，甚至长达数分钟。某些游戏（比如战神，last of us）会允许用户直接进入游戏，牺牲游戏体验的情况下，在背景完成pipeline的预创建工作。这类游戏游玩时有个有趣的现象，就是如果触发了之前没触发过的视觉特效，就会明显卡顿一下。

### 是否需要销毁？

如果业务场景存在UGC shader的业务，比如节点图式材质编辑器，shader编辑器，则需要设计销毁清理机制。其他场景这样的机制不是必需的。这样的考虑主要有：

- 大量pipeline实例同时存在不会有问题
  - 图形API一般没有存活数量限制
  - pipeline实例的内存开销小
- 不考虑销毁机制能简化实现

考虑销毁推荐采用类LRU的方式简单实现，以平衡实现复杂度和有效性。

## ShaderHash：how to access cached pipeline

假设有（场景数据+渲染方式配置），类型为 `T`，`T` 的所有取值，可以生成n个不同的pipeline。即存在T对pipeline的多对一关系。即可以直接通过`Hashmap<T, pipeline>`来维护pipeline。

这么做显然具有非常不合理的开销。但是我们可以观察到：T中用以决定采用不同pipeline的实际数据，远远小于T全部的数据。所以可将`T`提取出决定pipeline变体的子数据类型`V`，用`V`来维护pipeline，即 `Hashmap<V, pipeline>`。`V` 称之为 shader variant key

这个优化的有效性来自于：`T` 提取`V`的成本充分低，`V`的数据量足够小。为了性能考虑，实际上我们采用u64的`V`的hash来代替`V`，该u64称之为shader hash。这么做的好处是V的内存大小是一致的，不需要堆上分配来存储V，以提升cache access的性能，带来的风险是hash冲突，但从实际工程实践上，在采用了较高质量的hash算法后，hash冲突的问题是可以不考虑的（Rust的typeid也采用了hash的方式来进行计算，同样权衡了风险和效能考虑）。

场景数据`T`，本身可能会由多个不同的`T1`，`T2`动态组合出来，比如Mesh， Material，Pass，都可作为组合的单元。ShaderHash也可以将逻辑拆分到不同的**着色逻辑单元**，使得着色逻辑内聚。组合这些着色单元中，如果其类型是**动态**的，比如不是通过静态的泛型来组合，而是通过dynamic trait，那么其**类型动态性**的hash可以通过hash 这些类型的TypeId来实现。

由此，给定场景数据`T`，我们可以高效的计算出shader hash，以此来访问缓存的pipeline。此做法支持了我们现在渲染流程中pipeline的按需创建：

```rust
let pipeline_cache: &mut HashMap<u64, Pipeline>;

for pass in rendergraph {
 for object in pass.object_list {
  let hash = shaderhash(pass.effect & object);
  let pipeline = pipeline_cache.get_or_create(hash, || create_method(pass & object));
  pass.set_pipeline(pipeline);
  ...
 }
}
```

其中 shader hash是由对象结构的组合实现，自然组合实现的：

``` txt
shaderhash(some draw) ==expand=>
shaderhash(pass.effect & object) ==expand=>
shaderhash(pass.effect & node & mesh & material) ==expand=>
shaderhash(pass.effect & node & mesh & material_part_a & material_effect_modifier) ..
```

## ShaderHash的改进方向

上述这套做法是简单有效的，但在极端情况下 存在若干问题比如：

- per draw 需要做hash，如果draw的数量非常多，其性能成本不可忽虑，profiler显示占据5-10%的cpu开销
- 因为hash的原因，正确性无法保证，存在理论上（实际极不可能）的可能性造成hash冲突，导致获取到错误的pipeilne。

### Reactive ShaderHash

shader hash逻辑应只能针对场景变更执行，而不应针对全量场景执行，可以考虑通过增量reactive框架改进shaderhash全量的计算成本。具体的伪代码并不易表达，但主要实现逻辑是：

1. 每个不同类型的着色逻辑单元，维护`ReactiveCollection<AllocId<Self>, ShaderVariantKey>`
2. 采用同样的基于关系投影的方式组合出scene上per draw的shader hash reactive collection
    - draw unit到hash本身的reactive 多对一关系已经可以实现出增量的场景根据不同pipeline进行渲染分组的能力，以直接实现全增量的gpu driven分组和pipeline排序优化。
    - 因为渲染方式具备高度动态性，所以渲染时，实际的hash，即hash(pass, scene draw unit)依然采用全量计算的方式（但是成本是极低的（通过额外一层per draw hash的materialize）），以平衡实现设计复杂度和性能开销。
3. 此时reactive容器内access的target是明确的variantKey类型，那么可以多check equal或者对variantKey做intern的方式来保证正确性。

### 预创建的工程实现方式

- 半自动录制方案的支持
  - 全局pipelinecache是map shaderhash 到 gpu pipeline，预录制即是将这个cache序列化掉，然后在实际运行时先反序列化。支持预先录制需要额外保存gpu pipeline的可序列化数据（当然这个也只有开启录制才需要付出此成本）。gpu pipeline的可序列化数据可直接采用shader variant key K，因为K的信息足够完全生成pipeline。或者直接依赖图形api的pipeline cache。
- 主动枚举所有可能的pipeline varient取值
  - 类型是值的集合，对任何类型T，都可以实现遍历其所有可能的取值
  - 场景`T`，可生成shader variant key `K`，考虑可能不需要为`K`的所有情况生成cache以避免组合数过多。可考虑让场景开发手工设计selected shader variant key，即`SK`类型，该类型
    - 由`T`提供， `From<T>`
    - 给定`ST`，可直接生成pipeline
    - `ST`能遍历其所有可能的取值
  - 针对场景的根`SK`类型，遍历所有的取值，并创建或修改（作为创建的优化）`T`，并查询和填充pipeline cache。该流程可单独实现，也可组合录制方案实现以优化执行开销。

## 图形API支持

vulkan pipeline cache早期实现质量存在的诸多问题（主要是应对驱动错误实现） <https://medium.com/@zeuxcg/creating-a-robust-pipeline-cache-with-vulkan-961d09416cda>

wgpu在22版本添加了vulkan pipeline cache，框架上的支持额外需要做的事，在现有的hash基础上，额外添加体现整个rust本身代码动态性的hash，比如程序版本号。以此来access和维护磁盘上的pipeline cache。
