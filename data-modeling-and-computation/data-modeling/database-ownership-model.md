# 数据模型的所有权

这里讨论的所有权，指的是entity之间存在own的关系，使得

- 自动的回收资源，避免leak：某entity被删除后，被其own的entity，在一些情况下（比如没有任何其他entity own，且进入不可达的状态）自动的被删除
- 保证数据的引用完整性：避免删除还被引用的entity，造成bad handle(entity指向不存在的已经删除的entity)

用户可以完全自行手工保证上述这样的要求，除此之外，还可以通过自动/半自动的方式实现这样的能力：

## 引用计数

实现的前提：

- 需要用户显式的定义外键的ownership方向。比如A B两种entity，有多个A对应一个B的关系，通过设置A指向B的外键实现。此时可以决定是 A own B，还是B own A。系统总是默认的认为外键的指向即ownership方向。但是某些情况下，反向的定义是有用的。比如我们希望scene在删除时能够删除所有的其下的scene model，但是外键的方向是从scene model指向scene的，那么此时就需要反向配置。
- 整体的ownership关系，形成一个图的结构，这个结构不能出现环。

实现的风格：

- 采用同步模式：在任何一次数据entity的修改都立刻执行所有级联的计数修改和资源清理
  - 可以尽早释放资源
  - 但是性能会非常差。[raii-is-bad](../../general-practice/raii-is-bad.md)
- 采用异步模式：在必要时执行批量的级联计数修改和资源清理，比如定时或者存储需要扩容时。
  - 资源释放是延迟的，并且需要外部在合理的时机主动调用
  - 性能是最好的

## tracing GC

用户需要定义哪些entity处于存活状态（GC root），系统假设外键具有双向的ownership（或者说可达性），然后对不可达的entity执行清理。

对比引用计数实现的前提，gc并没有这样的约束。然而正确配置entity的存活状态是困难的，所以这里我们不做展开，只是补充说明tracing GC也是一种可能的实现方向。

## 数据模型和所有权模型的解耦

采用任何自动实现，都会引入不同形式的运行时性能开销，在entity的lifetime都是静态，且数据量巨大的情况下，手工管理才是最佳方案。所以数据模型层内置一个所有权实现并不合理。
根据上文，数据的所有权实现可以在不同的场景下根据不同的假设，使用不同的设计和实现，甚至混合采用不同的实现。所以所有权的设计和实现并不应该由数据模型层决定和提供。

由此决策产生的问题是，引用完整性无法由数据模型层本身实现和保证。所以上层应用总是要能够处理由于错误的编程逻辑导致的bad handle问题和内存泄漏问题。为了缓解此类问题的实际影响，有必要开发利于排查和定位这类问题的基础设施。比如entity的create remove backtrace（以便于分析entity的创建和销毁位置），debug模式的同步完整性检查（以便于在出现bad handle的第一现场触发调试）。
