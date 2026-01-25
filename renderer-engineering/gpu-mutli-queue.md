# GPU Multi Queue 的理解

我之前一直不理解multi gpu queue，async compute为何具有性能优势，因为我设想driver完全可以深度分析单个queue上提交的内容的依赖关系，来实际调度任务到设备的多个并行的硬件上完成计算。这个想法其实没有问题，比如webgpu是否实现multiqueue也是有争议的，一种论点就是应该由webgpu的实现者负责自动的multi queue优化。

对于vulkan，multi queue的存在是因为一般的driver scheduler实现不存在任何高级的调度逻辑。即driver只会将queue里的东西直接发送到设备上执行，不会实现上述优化。
queue的执行方式，朴素的认为就是来一个执行一个，然后在一定条件下一个没执行完就开始执行下一个，不存在深度执行/自动saturate所有硬件资源的能力。gpu task级别的并发的相关的优化需要用户的图形侧通过多queue来显式表达。

根据 [vulkan spec](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#devsandqueues-queues)，vulkan的queue，queue family api完全不对其提供的queue的执行方式提供任何保证，它仅仅保证说某个queue只支持哪些功能，其上的提交存在submission的order保证。不保证多个queue的会map到硬件的queue上，不保证它们的执行是并行的。唯一的性能方面的控制是queue可以设置它们的执行优先级。不同queue之间的workload可能按时间片调度，总之标准没有做出任何实现上的规定。开发者也无法从api上查询出有意义的利于实现选择的信息。

vulkan 令人感觉困惑的点是：其api大部分看起来设计的非常底层，即开发者可以假设驱动实现不会做fancy的优化，而是直接调用硬件的能力来执行计算，优化由我来负责。但是其中某些api，又实际上完全依赖vulkan的实现做优化，比如subpass，是非常高层的计算图描述。queue和queue family相关的api也有类似的问题，因为api的用户做的事情本质上就是表达计算图（只不过是隐式的），然而这个计算图并发执行方式，并行能力，调度细节，延迟吞吐量的控制，硬件的特征基本上是黑盒。

在多queue的使用上，一般是只使用一个grahics的主queue提交图形任务，然后手工的将能和graphics并行的纯compute，提交到另一个compute queue上，即所谓的async compute。即便api上没有暴露并行相关的能力信息，简单的认为存在一个可以和graphics并行的compute queue是合理的，因为driver肯定不会自己做多queue优化。就实际的测试来看，多个graphics queue/compute queue都是没有意义的。

总结来说：多queue优化的有效性在于

- 一个gpu任务，即便其宽度足够大，一般也只会在某个时刻满载gpu上某些硬件资源，而不是全部的硬件资源
- 某些gpu可以支持同时并行多个gpu任务，如果他们的资源消耗类型存在互补的关系，那么整体的吞吐量就可以提升
- 某些gpu可以支持同时并发多个gpu任务，并控制不同gpu任务执行的优先级/延迟。
- queue作为任务提交的端口，其在实现上一般是朴素的（设计上没有规定），不会自动的利用表达上述逻辑
- 图形api通过多queue让用户来表达上述逻辑

## 一些讨论

- <https://github.com/KhronosGroup/Vulkan-Docs/issues/569>
- webgpu multiqueue discuss
  - <https://kvark.github.io/webgpu-debate/MultiQueue.html>
  - <https://github.com/gpuweb/gpuweb/issues/1066>
- <https://gpuopen.com/learn/concurrent-execution-asynchronous-queues/>
  - GCN hardware contains a single geometry frontend, so no additional performance will be gained by creating multiple direct queues
  - While GCN hardware supports multiple compute engines we haven't seen significant performance benefits from using more than one compute queue in applications profiled so far
