# 减少Indirect渲染器的dispatch call数量

实现gpu driven/indirect渲染器的目的之一是为了AZDO（Approaching zero driver overhead），核心原理是indirect的渲染方式使得drawcall数量几乎不再和场景的基本绘制单元数量相关。但是naive实现的indirect渲染，在特定复杂的场景下可能dispatch call的数量非常高，以至于产生严重的性能问题。所以减少dispatch call相关的优化对于实现AZDO的目标是必要的。

## dispatch call数量多的原因

indirect dispatch数量多的一个核心原因是indirect draw的实际数量比理想情况要多

- 场景的实际pso variant可能有很多
  - 因为indirect draw也只能draw相同类型的pso
- 场景api总是有某种绘制顺序控制的需求
  - 这导致无法自由的合并相同pso的draw，被迫以更零碎的方式绘制

因为这是indirect draw api本身的限制，所以这一方面没有优化空间。

indirect dispatch数量多的另一个核心原因是 **drawlist compute处理的「乘法效应」**：

这里drawlist指的是一批可以通过一个indirect draw绘制的场景对象。

针对一个drawlist准备indirect draw需要涉及诸多compute dispatch。其中典型的有：

- drawlist的剔除（无论是优化相关还是业务逻辑相关），drawlist -> drawlist
- midc drawcommand list生成
- midc 到 single indirect draw的降级（可选）

这些处理操作可能需要per list 20～30次compute dispatch。当场景由于上述原因需要分为100个indirect draw来绘制时，为了绘制这些draw，就需要2000～3000次compute dispatch。

## 去除 drawlist compute处理的「乘法效应」

减少Indirect渲染器的dispatch call数量的核心做法就是避免出现这样的「乘法效应」

核心做法是改进上述per list的compute shader处理逻辑的实现，使其支持同时处理多个drawlist（list of list）。因为

- 虽然每个drawlist要使用的pso不同，但是他们的剔除算法，比如stream compaction的算法，都是一致的。不同pso的drawlist，应该在一个dispatch中处理剔除
- drawcommand list生成同理，但是需要注意的是，不同pso的drawlist，他们的drawcommand生成逻辑可能是相同，也可能不同的。所以需要额外的实现 list of list重新分组出新的list of list，生成command之后，然后再还原匹配的逻辑。
- 同理其他per drawlist的逻辑也是如此

为了支持dispatch处理list of list，drawlist的底层gpu资源需要进行调整：全局所有的drawlist，需要分配在一个gpu buffer上，或者采用bindless buffer。

具体算法的调整支持list of list的实现，没有什么通用的做法，需要针对具体问题具体分析。

其中比较通用的一点是这些算法都需要根据global id来获取当前处于哪个drawlist，这可以通过二分查找来drawlist count prefix sum来实现（这个和[midc降级的算法](./multi-indirect-draw-downgrade.md)一样）。因此实际上支持list of list会提高gpu成本。我考虑过其他的实现，但如果要综合高效的支持其他list算法，特别是list of list高效重新分组，似乎没有其他合理的做法。drawlist count prefix sum本身也需要compute shader来维护。
