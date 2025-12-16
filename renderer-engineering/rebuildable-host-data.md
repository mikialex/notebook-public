# 表内数据的抽象访问机制

对于mesh/texture这类表内引用的外部数据，需要支持抽象的异步访问方法（比如uri）

## 几种使用场景

### 支持host数据卸载

用户的host端场景数据，如果没有在host端使用的需求，比如在渲染的资源准备（创建device上的buffer/texture）完成后，那么就应该释放以避免不必要的常驻内存消耗。

典型的场景数据中，贴图和mesh是主要的内存消耗，其中贴图基本上是没有任何host使用需求的，mesh如果采用gpu pick的技术（比如framebuffer pick或者gpu上的raytrace），也亦没有host端pick的使用需求。这些内容占据场景内存消耗的主要部分，释放这些内存是基本必要的优化。

然而要**合理**实现这个数据释放机制，远不是简单的在gpu数据生成之后删除数据这么简单（当然以某种方式hack的实现是可以的）。因为在web上，device丢失是必须要考虑的问题。device需要重建意味着device数据需要重新生成，这意味着host数据需要在device重建后，重新处于可用的状态，意味着host数据本身也需要支持重建。

### 支持流式加载

表内数据支持url/uri作为抽象的访问机制，意味着消费系统需要直接具备处理异步资源加载的能力，并正确实现某些资源没有加载完成时的降级行为。这意味着某些数据格式的外部资源引用可以直接表达。即外部数据的解析器可以不处理外部资源引用，直接填写db的uri数据，而直接具备场景渐近加载效果。这减轻了loader实现的难度，并直接在场景层面提供了流式加载的能力。

### 作为表内虚拟化的前置实现

我们定义db数据不变，通过调度所有抽象数据的实际显存/内存占用来实现消耗控制的虚拟化为表内虚拟化。（与之对比的是 通过直接控制表内容的表外虚拟化）。所以需要先实现抽象数据的支持，才能开始做表内数据虚拟化。

可以说实现抽象数据的支持是实现表内数据虚拟化的主要工作，在消费系统支持异步数据加载后，实现表内数据虚拟化只需要修改调度逻辑来决定哪些数据需要加载卸载即可。支持表内数据的抽象访问和异步数据加载，本身就包含了一个没有任何调度逻辑的表内虚拟化。

### 支持任意表外数据

- uri可以去指代任何数据源，包括我们在[持久化](../editor-engineering/database-incremental-persistency.md)相关讨论中提到的表外的mesh undo-redo system.
- uri的异步访问可以封装任意的异步数据处理方法，比如以额外参数的形式解压缩，mesh/texture处理。这部分实现可以封装进入访问机制的提供者抽象中，访问机制的提供者抽象本身可以做成可组合的抽象。

## 实现设计

异步可重建的数据可以通过uri/path来抽象的表达：

```rust
/// 描述抽象数据源，用来解释并加载某类uri/path，「访问机制的提供者」
pub trait UriDataSource<T> {
  /// 将同步数据转化为异步可重建数据
  fn create_for_direct_data(&mut self, data: T) -> &str;

  /// 加载异步数据的实现。消费系统只表达并发逻辑，实际数据源是抽象的
  fn request_uri_data_load(&mut self, uri: &str) -> impl Future<Output = Option<Arc<T>>> + 'static;

  /// 通知内部通过create_for_direct_data创建的uri的临时内存数据需要清理。见后文解释
  fn clear_init_direct_data(&mut self);
}
```

UriDataSource一般基于file/network io实现。也可以做一个In-memory source的伪实现。

原本在DB中直接存储的数据T，变更为这样的枚举，由用户决定是否采用可释放（重建）数据。

```rust
pub enum MaybeUriData<T> {
  Uri(Arc<String>), // 抽象可重建数据
  Living(T), // 直接数据
}
```

我们以这样的接口来进一步讨论实现细节：

## 需要提供将同步数据转化为异步可重建数据的方法

用户的场景数据有可能是算法直接生成的（比如cad模型三角化的结果），而不属于任意一种「异步可重建」的数据形态。所以基础实现需要提供基础设施，以让用户能够将任何直接数据转化为异步可重建的数据。整体概念类似于web上的createObjectURL方法。

- 对于用户本身就有磁盘上的数据：直接填写uri path，并配置基于磁盘的uridatasource
- 用户直接创建living（直接）数据，但是需要数据被消费后内存能被卸载：通过create_for_direct_data创建uri。UriDataSource会创建磁盘上的数据。

这部分实现有一定的性能要求和细节：在首次使用时不会导致额外的成本。用户转化为异步可重建的形态的操作需要保证转化性能，应buffer到worker线程中完成（但接口不一定是异步的，因为这里await结果对于用户没有太大意义）。因为数据原本就是可用的，所以在进行消费时，应该直接可用。整体而言，目标是用户创建大量直接场景并进行可重建数据生成，对比用户创建大量直接场景，不应该出现明显性能衰退。

## 为什么需要clear_init_direct_data？

消费系统首次调用request_uri_data_load时，UriDataSource因为并没有删除内存数据，所以会返回一个立刻resolve的future，避免任何延迟创建的现象。同时，因为UriDataSource不知道通过create_for_direct_data创建的数据会被request_uri_data_load使用几次，所以UriDataSource无法决定合理的时机释放create_for_direct_data创建数据的临时内存，所以UriDataSource上提供了clear_init_direct_data来让用户触发释放机制。合理的释放机制一般是每一次所有消费系统都运行完成之后。

## 用户虽然有磁盘上的数据，但是数据和db上的entity结构不匹配

比如用户有一磁盘上的glb。glb中同时包含了image和mesh buffer。但是显然我们不能直接将glb的path作为image/mesh buffer的uri。这种情况下有两种做法：

- 将glb中的image/mesh buffer通过create_for_direct_data重新单独写回磁盘，转化为上一个问题
  - 实际上需要再写一份数据到磁盘，有额外成本
  - 实现简单
- 构造特别的path，用glb path + sub resource信息来表达。由特别的UriDataSource来负责解析
  - 实现复杂，特别是同一glb path不同sub resource的加载future复用问题。
  - 用户需要保证glb数据是稳定的
  
采用哪一种做法需要根据具体情况决定

## 为什么不由一个外部系统负责数据加卸载？

具体做法是如果有uri数据需要写入db，我们只写入某个外部系统，由这个外部系统来负责实际的db修改，在uri加载后写入实际数据到db。在数据需要卸载时通过写db来卸载。这样原本现在基于db上现成数据的消费系统可以不做任何改动。（这个做法有别于提到的表外虚拟化）

这么做看似简单，但实际上会遇到一系列问题：

- 这个外部系统还需要负责数据卸载，需要修改数据消费系统以通知卸载数据
- 首次写入默认数据来达成uri未加载行为，需要修改消费系统实现，来适配这一缺省数据的降级行为
- 首次写入默认数据，和卸载数据，会造成额外两遍db change，而消费系统还要特别的在卸载数据的change时不做任何事情

即我们仍然需要大幅修改消费系统的实现，并且付出额外的性能代价。

## 消费系统的具体改造方式

db的uri改造实际上导致消费系统的input 从 `DualQuery<K, T>` 变更为 `DualQuery<K, MaybeUriData<T>>`。

所以具体的改造工作是：实现一个 `DualQuery<K, MaybeUriData<T>>` 到 `DataChanges<K, T>` 的转换器，并将所有对`DualQuery<K, T>`的依赖转化为对`DataChanges<K, T>`的依赖。

这个转换器内部会调用`UriDataSource`的数据源接口来做加载，同时还可以封装决定哪些数据加载和卸载的调度器逻辑（预留的虚拟化实现点）。进一步说转换器可能还可以暴露外部的注入weight的接口来提供支持调度重要性信息，weight可能是gpu管线异步生成的数据。

转化器能提供卸载能力，是因为output的datachange是具备remove message的

`DualQuery<K, MaybeUriData<T>>` 需要转化为 `DataChanges<K, T>` 而不是 `DualQuery<K, T>`，是因为 `DualQuery<K, T>` 包含 `Query<K, T>`，而这是不可能的，因为T的全集数据根本不可能存在（因为我们要实现数据卸载和虚拟化）。所以转化出来只能是增量数据。

采用`DataChanges<K, T>`而不是`DualQuery<K, ValueChange<T>>` 正是我们为什么要设计datachange的原因之一，即datachanges不包含previous message。转化器是不可能生成包含previous message的change信息，因为如果要生成，那么转化器内部就需要维持previous T，这同样和我们要支持数据卸载和虚拟化的要求是矛盾的。

texture和mesh的gpu数据维护系统本身就只依赖datachanges作为input，而不需要全量数据和previous数据。某些下游系统比如mesh bounding，本身不需要previous数据，但它的再下游系统依赖dualquery，那么我们可以直接使用datachange到dualquery的转化器。
