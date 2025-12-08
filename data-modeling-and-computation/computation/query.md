# 基于抽象query的数据消费框架

根据[统一数据模型到database的思想](../data-modeling/database-is-fundamental.md)，在数据结构完成normalization之后，我们有必要探索一种数据处理逻辑的normalization（统一模型）。考虑到存储之于消费/计算的关系，如同database之于query，我们将这一统一计算模型称之为query。

## Query的抽象

 `Query<K, V>`描述了一个**抽象的kv map**，其中

- K （一般）需要实现 Eq + Hash + Clone
- V （一般）需要实现 Clone + PartialEq
- 可从&K 以 O(1) 获取 V
- 可获取 K, V 的迭代器

rust表达为这样的trait

```rust
pub trait Query {
  type Key: Eq + Hash + Clone;
  type Value: Clone + PartialEq;

  fn iter_key_value(&self) -> impl Iterator<Item = (Self::Key, Self::Value)> + '_;

  fn access(&self, key: &Self::Key) -> Option<Self::Value>;
}
```

一些常见的数据容器比如hashmap，hashset，vec 都可以实现这样的query。database的component会实现这样的query，以抽象的暴露其中存储的数据

这一抽象query的意义在于我们可以和迭代器一样，对query进行transform。比如mapping，filtering。transform的query。query作为抽象的dataview，transformation在access和迭代时当场计算，而不是转化原始数据。

和迭代器一样，不同的抽象query可以被组合，比如

- zip：垂直拼接两个query `A: Query<K, V1>`，`B: Query<K, V2>`，要求A B的key的值域和类型完全一致。组合新的query `Query<K, (V1, V2)>`
- select: 横向拼接两个query `A: Query<K, V>`，`B: Query<K, V>`, 要求A B的key value 类型完全一致，但key的值域完全不重合，组成新的query `Query<K, V>`
- join，intersect等常见的集合运算

通过这些query，用户可以在db，或者任何通用数据容器上组合和构造复合query，以实现数据的全量抽象访问。

### materialization的概念

query的抽象的，上面可能有任意的封装的access时计算逻辑。将一个抽象view 访问遍历一遍，并将结果存储于另一个容器的过程为 materialization。

materialization的目的是将抽象view上的计算应用并保存下来，以避免后续每次访问view都要触发on the fly的计算逻辑。

materialization后的新容器本身也直接实现query，所以query内部可以嵌套隐形的materialization。

## 增量计算的实现

### 数据变更的表达

database中某个entity的某个component的变化，通过这样的枚举表达：

```rust
pub enum ValueChange<V> {
  // new_v, pre_v
  Delta(V, Option<V>),
  // pre_v
  Remove(V),
}
```

ValueChange不仅记录变更的新数据，还记录变更前的数据。

- 用户对某个数据V的修改历史，可形成valuechange的序列。
- 两个value change可以被合并。即
  - mutation+mutaion = mutation
  - add+remove = nothing
  - add+mutation = mutaion
  - remove+add = mutation
  - 其他情况是不可能的，属于非法的value change组合
- value change的历史序列可以被合并，合并满足结合率，不满足交换律

### DeltaQuery

定义DeltaQuery为 value是valuechange的Query，即： `Query<K, ValueChange<V>>`

database的生产消费流程是：

- 用户修改db数据（生产变更）
  - 在修改时，触发同步的回调函数
  - 回调函数将data change写入到某收集change的容器中，缓存起来
- 用户触发消费逻辑
  - 被buffer的data changes被take出来，作为变更集在消费管线中被处理
- 这样的生产消费反复循环运行

上述「被buffer的data changes」的变更集，即实现了DeltaQuery。在变更过程中，因为value change可以merge，所以相同key的change只会保留被后的结果，这不仅满足了变更集实现DeltaQuery的要求，同时也防止了收集变更历史导致临时内存无限增长的问题。

因为 DeltaQuery 其实就是 Value为ValueChange的Query，所以Query的抽象transform和组合的实现同时适用于DeltaQuery。

### DualQuery

假设考虑对DeltaQuery执行mapping，filter之类的transform，是否可以完成数据集的增量消费？

答案是这仅限于单个query的情况，如果我们考虑多个delta query需要结合的进行增量消费，单纯依赖delta query是不够的（而实际逻辑中大部分都设计到复杂的query合并逻辑）。举例来说，假设有一个zip的query结构，要获得zip后的query的增量change，单纯依赖zip前的两边的delta query是不够的，直接对detla query进行zip在逻辑上是没有意义的。

正确处理多query组合子的增量消费，不仅依赖delta query这个增量表，还同时依赖原始query的全量表。还是以zip举例：zip的上游两边假设分别有以k为key的a1，b1的取值，假设经修改后，左边的k取值变为 a2，那么zip的增量change里会是(a2, b1)。位于zip的右侧上游，即便没有任何change，要生产正确的(a2, b1)下游change，也必须依赖对b1的访问，即必须访问右侧的全量表。

用户某段时间内持续修改 `Query<K, V>`, 形成了 `DeltaQuery<K, V>`的数据。我们将这样匹配的一对 `Query<K, V>` `DeltaQuery<K, V>`组合起来，称之为`DualQuery<K, V>`。

因为要支持普遍的多query组合消费，而多query组合消费依赖全量表，所以Dualquery是增量计算框架的核心抽象和核心实现。和Query一样。Dualquery同样支持transform和combinator的组合子。Dualquery map Dualquery， Dualquery zip Dualquery ..

这些组合子的实现内部也经常会使用到普通query的组合子。Dualquery组合子的实现是比较复杂的，不过其复杂度被Dualquery这个结构完美封装了。用户实现增量消费的过程，就是去组装dualquery计算，然后直接访问output的增量表即可获得transform后的change。因为dualquery组合子的实现保证了消费成本完全正比于增量表大小，无论Dualquery是由多么复杂的实现组装出来，都可以保证最终消费成本完全正比于增量表。

## 并行计算和reactive支持

用户组装的Dualquery算子，形成一个计算图。 在实际的计算框架中，这个计算图是在一个线程池上被并行消费的。考虑到编译性能和编译本身的风险（dualquery会生成非常非常复杂的符号名导致编译时间长），并行消费仅限于算子间，除非用户采用特殊的实现，算子内是不采用并行计算的。

用户会根据实际change的存在情况来组装Dualquery算子，而不是每次都生成最大的计算图。

用户组装的Dualquery算子时，需要传入Context。在实际change 发生后，负责生成delta query的channel实现会wake，以此实现reactive能力的支持。用户可以直接根据wake信息，在最上层或者任意层次结构来直接得知是否存在增量change需要消费，以避免任何check消费的固定成本，或根据change信息实现任何额外逻辑。
