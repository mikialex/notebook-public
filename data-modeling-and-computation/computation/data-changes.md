# query框架的另一种变更数据抽象：DataChanges

## 为什么delta query的valuechange需要包含previous message？

包含previous 看起来是多余和麻烦的，因为：

- 所有算子都需要在消费环节处理previous，这实际上几乎导致了两倍的性能消耗
- 某些重计算，因为希望避免重新对previous执行计算，只能缓存计算的结果，这需要付出额外内存使用的代价
- 正确处理和生成previous，导致某些算子实现复杂，难以推理实现的正确性

但实际上previous是有用的，也很难说其在计算和内存使用上相比没有previous的版本孰优孰劣。在query框架的早期研发阶段，我曾经整体迁移尝试过使用无preivous版本的变更数据，遇到了很多实现上的问题。

- 业务逻辑可能真的需要previous。（这种情况下，业务侧也可以通过存储previous来实现previous获取，同样的付出内存代价）
- 没有previous，类filter的算子实现成本会很高。
  - 因为previous可能已经是被过滤掉的，所以对于一个新的change，是否应该emit change还是什么都不做，需要我们知道previous数据。所以就需要cache previous数据，付出内存的代价。然而类filter算子是非常常见的，cache previous数据的内存和性能代价是无法接受的。
  - 假设我们允许remove message可以重复key。filter算子的实现在大部分都被过滤的情况下（这很常见）会输出大量的无效的remove。导致整个下游消费成本剧增。可以考虑人工指定哪些filter 算子需要过滤，但这对于工程上是难以接受的
- 很多算子的实现，如果要保证消费性能，同样要求整个变更protocol允许「remove message可以重复key」，并且还要允许「重复key的相同new message」。然而这和filter算子一样，很可能出现消费性能问题。

## 为什么一个不包含previous message的变更接口是必要的？

- 包含previous数据，会导致数据的生命周期被延长到previous数据释放之后。在实际应用上，变更数据可能以`Arc<HugeData>`的方式包含场景中的大量非结构化数据，如果previous数据总是被消费，意味着内存中总是以某种方式存储着变更之前的数据，这对于实现场景数据虚拟化的内存控制来说是不可接受的。
- 包含previous数据的变更信息，在驱动gpu数据更新等简单实现上，是overkill的。
  - previous数据实际上没有使用，但是记录存储传播都付出了成本
  - 实现复杂，导致binary和编译时间增长

## DataChanges

DataChanges是特地为解决这类问题设计的变更接口：

```rust
pub trait DataChanges{
  type Key;
  type Value;
  fn iter_removed(&self) -> impl Iterator<Item = Self::Key> + '_;
  fn iter_update_or_insert(&self) -> impl Iterator<Item = (Self::Key, Self::Value)> + '_;
}
```

该接口提供的承诺是，用户先调用iter_remove移除数据（可能会移除本来就不存在的数据）。用户再调用iter_update_or_insert设置新的数据（可能会override已有的数据，并且后面的会override前面迭代的），就可以完成数据变更的同步。因为这个接口可以简单的用来同步数据，所以取名为data changes

除了设计和实现简单，另一个重要特点是，该接口要求用户先调用iter remove。这样做的目的是优先同步所有的删除，以最小化峰值内存使用。这样有利于虚拟化内存控制的实现。

DataChanges可以通过cache current数据来生成delta query的change。delta query可以直接构造访问的view来实现datachange接口。以此实现两种数据接口的转化。
