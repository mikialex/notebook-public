# 数据容器的内存消耗控制

本文主要讨论场景数据容器尺寸控制的策略。

数据容器指代渲染器运行时被持久化的数据，主要有以下几种类型：

- db 中的场景数据
- db query 消费过程中的 materialization cache 容器
- gpu buffer/texture

尺寸控制有这些需求：

- 实际容器内存消耗和场景规模相关
  - 加载大量内容，然后卸载，内存消耗能够**自动**降低到合理水平
- 支持 shrink to fit， 比如用户加载了某静态场景，加载完成后调用此过程以最小化不必要的内存消耗
- 控制极端的内存增长
  - 方案1：直接设置内存使用上限，这样任何激进的扩容行为都是安全的
  - 方案2: 在内存消耗已经很高的情况下，避免成倍的内存扩容行为
  - 这两种方案都需要用户提供额外配置信息以指示触发额外的处理逻辑的临界内存消耗量

## linear容器的size控制方案

容器根据是否具备随机线性寻址能力，可区分为linear和非linear两类。前者的存储采用类似vec的容器实现，后者一般是hashmap。具备linear性质的容器没有办法高效的shrink。

在[Generational arena shrink 的实现改进](./generational-container-shrink.md) 一文中我们讨论了linear arena的shrink实现。总之判断一个容器是否可以shrink，是compute/memory intensive的。主要原因是大部分容器的内部分配状态都都是跟随db容器的内部分配状态，即需要O(n)的check，或者维护O(n)大小的empty slot信息（这被认为空间成本，是因为消费环节的容器，不需要支持主动allocate操作，所以这样的信息是不需要的）。

根据这样的特性：凡是这种跟随db arena分配状态的线性容器，其shrink和grow，应该直接跟随db arena。以避免上述计算和存储的开销。即合理的做法是，db层需要暴露grow和shrink事件，以让下游直接在消费前同步arena的size变化。

在实际工程上，这一size同步机制目前是难以实现在已有的query框架内的。因为query框架不仅没有暴露max allocation key的信息，也不会对query本身key的linear性质有任何要求。对query框架本身进行改进是可行的，但是其改动和维护另一套特化的抽象，其成本相对于收益过于高昂。所以对于query框架维护的派生linear容器，目前的做法是需要用户手工的和上游linear容器（比如最上游的db）进行关联。

## 非linear容器的size控制方案

query框架驱动的非linear容器size控制，可直接由query接口提供的iter size_hint实现。

- 确保能正确实现size hint的query都实现size hint
- 在容器侧，debug warning没有合理信息的size hint
- 对于没有size hint实现的query，可考虑实现维护size的dual query算子

## 调试能力建设

在容器size控制优化实施之前，应先行完成调试能力相关的技术建设。需要能够实时显示目前所有容器的绝对内存消耗，和利用率。和更新cycle中触发的resize事件。这有助于

- 暴露不合理的容器初始化size配置
- 暴露不合理的场景容量变化模式
- 暴露忽略控制的容器

### shrink是否需要引入时间方面的策略？

比如某一帧大量内容删除，但是下一帧又大量内容增加，可考虑通过记录需要shrink的时间，来debounce实际的shrink。

但实际上不需要，其实现的复杂度不足以支持其带来的益处。 某一帧大量的内容增加本身就是costly的，优化其扩容或者反复扩容的成本意义不大。
