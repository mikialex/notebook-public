# host渲染数据释放机制

用户的host端场景数据，如果没有在host端使用的需求，在渲染的资源准备（创建device上的buffer/texture）完成后，那么就应该释放以避免不必要的host内存消耗。

典型的场景数据中，贴图和mesh是主要的内存消耗，其中贴图基本上是没有任何host使用需求的，mesh如果采用gpu pick的技术，也亦没有host端pick的使用需求。这些内容占据内存消耗的主要部分，释放这些内存是基本和必需的优化。

## host数据也需要支持重建

实现合理的host的数据释放机制，并不是简单的在gpu数据生成之后删除数据这么简单。

实现完备的渲染器，特别是需要支持web平台的渲染器，需要考虑device丢失重建的问题。device需要重建意味着device数据需要重新生成，这意味着host数据在重新生成时需要重新处于可用的状态，意味着host数据本身也需要支持重建。

从这个角度出发，可以认为用户需要表达的场景数据，并不应该是简单的host数据，而是应该支持「可重建」的host数据，或者说是数据的访问的接口，而不是数据本身。因为全量数据没有高度可压缩的假设，所以优化host的内存的方式，一般只能是存储在网络或者磁盘上，所以这样的数据重建接口是异步的。它返回一个获取全量数据的Future。

## 消费系统需要支持数据加载

因为host数据要支持重建，所以host数据就需要是抽象表达的，那么意味着用户场景数据的消费系统，必须同时负责数据的异步加载。这个改造是必需的，同时也是整个host数据释放机制的主要开发成本。

有的消费系统可能行为上做数据的异步加载并不合适，这种情况下应该检查并要求用户提供直接数据。比如cpu pick scene。有的消费系统行为上实际本来就应该实现数据的异步加载，比如渲染系统，应该做到边加载边渲染，以提升用户体验。

## 需要提供将同步数据转化为异步可重建数据的方法

用户的场景数据有可能是直接生成的，而不属于任意一种「异步可重建」的数据形态。基础实现应该提供必要的基础设施，以让用户能够将任何数据转化为异步可重建的数据。这样的工具函数需要让用户提供必要的序列化和反序列化细节（其中包括压缩和校对的细节），同时需要保证经过这样的转化之后，在首次使用不会导致额外的成本（因为数据原本就是可用的）。这类似于web上的createObjectURL方法（区别是真的创建磁盘上的数据）。

## 实现设计

简单的认为url或者path就是异步可重建的数据，可以设计这样的接口：

```rust
pub trait UriDataSource<T> {
  /// 将同步数据转化为异步可重建数据
  fn create_for_direct_data(&mut self, data: T) -> &str;
  /// 加载异步数据的实现。消费系统只表达并发逻辑，实际数据源是抽象的
  fn request_uri_data_load(&mut self, uri: &str) -> impl Future<Output = Option<Arc<T>>> + 'static;
  /// 通知内部通过create_for_direct_data创建的uri的内存数据需要清理。见后文解释
  fn clear_init_direct_data(&mut self);
}
```

UriDataSource可以做一个In-memory source的伪实现，也可以基于file/network io实现。

---

db层的数据表达，由原本的直接数据T，变更为这样的枚举，由用户决定是否采用可释放数据。

```rust
pub enum MaybeUriData<T> {
  Uri(Arc<String>),
  Living(T),
}
```

问题：为什么需要将uri存储入db，而不由一个外部系统负责数据加载？，具体做法是如果有uri数据需要写入db，我们只写入某个外部系统，由这个外部系统来负责实际的db修改，在uri加载后写入实际数据到db。这样原本现在基于db上现成数据的消费系统可以不做任何改动。

这么做看似简单，但实际上非常麻烦，这种做法的问题是：

- 这个外部系统还需要负责数据卸载，所以现有数据消费系统还需要通知它卸载数据，这其实还是需要修改消费系统的实现
- 首次写入默认数据来达成uri未加载行为，依然需要修改消费系统实现，来适配这一默认数据的行为
- 首次写入默认数据，和卸载数据，会造成额外两遍db change，而消费系统还要特别的在卸载数据的change时不做任何事情

### 具体使用案例和潜在问题

用户本身就有磁盘上的数据：

直接填写uri path，并配置基于磁盘的uridatasource交由消费系统。

用户直接创建living数据，但是需要数据被消费后内存能被卸载：

通过create_for_direct_data创建uri。UriDataSource会创建磁盘上的数据，同时依然暂时保留内存数据。消费系统首次调用request_uri_data_load时，UriDataSource因为并没有删除内存数据，所以会返回一个立刻resolve的future，避免任何延迟创建的现象。同时，因为UriDataSource不知道通过create_for_direct_data创建的数据会被request_uri_data_load使用几次，所以UriDataSource无法决定合理的释放create_for_direct_data创建数据的临时内存，所以UriDataSource上提供了clear_init_direct_data来让用户触发释放机制。合理的释放机制一般是每一次所有消费系统都运行完成之后。

用户有磁盘上的数据，但是数据和db上的entity结构不匹配：

比如用户有一磁盘上的glb。glb中包含了image和mesh buffer。但是显然我们不能直接将glb的path作为image/mesh buffer的uri。这种情况下有两种做法：

- 将glb中的image/mesh buffer通过create_for_direct_data重新单独写回磁盘，转化为上一个问题
  - 实际上需要再写一份数据到磁盘，有额外成本
- 构造特别的path，用glb path + sub resource信息来表达。由特别的UriDataSource来负责解析
  - 实现很复杂，特别是同一glb path不同sub resource的future复用问题。
  
一般认为方法1虽然要付出成本，但较为合理。因为

- 实现简单
- 加载之后其实并不能保证glb是一个稳定的数据源。如果重新制作新的磁盘备份反而能解决这个问题

## 作为其他高级实现的前置实现

- uri可以去指代任何数据源，包括我们在[持久化](../editor-engineering/database-incremental-persistency.md)相关讨论中提到的表外的mesh undoredo system.
- host数据可通过uri重建，才能确保表内可交换的host数据实际上不存在，而这是虚拟化相关实现的前置工作
