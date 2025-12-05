# 渲染器 wasm thread 支持问题

多线程场景更新对于渲染器是必要的。我近日通过 <https://docs.rs/wasm-bindgen-rayon/latest/wasm_bindgen_rayon/> 这个库在rendiation中做了一番集成和尝试（因为我现在用的就是rayon）。结论是wasm thread支持还有很多工作要做才能实现。

**最大的问题：**

<https://github.com/WebAssembly/threads/issues/106>

这个问题我几年前就注意到了，但是后来竟然忘记了这个问题，要是我还记得这个问题，那整个根本没有尝试的必要。

这导致主线程不能存在任何blocking操作。简单说，在主线程你根本不能调用锁。

这导致，应用多线程优化在工程上是非常困难的。可能的解决方案对现有架构需要做大范围的调整，比如：

- 将整个渲染器，或者说整个应用完全移动到worker线程池上。简单说，你渲染器暴露的所有的api都是异步的。
  - 具体来说，每一帧收到的交互事件在主线程buffer，序列化然后发到worker的wasm里消费，然后直接自绘offscreen渲染提交到canvas。如果用户不是自绘ui，或者有很多wasm io需要交互，那么会更加麻烦。
- 使用 [waitAsync](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/waitAsync) 来实现无rust锁的channel，通过这种channel实现同步。
  - 清理所有锁的使用，一般codebase里为了多线程环境的实现方便，可以只用refcell的地方都用了rwlock
  - 对于多线程更新部分的实现，采用上述channel完成通信

**次要但是麻烦的问题：**

在开启了atomics和bulk-memory的target feature之后，wgpu的 fragile-send-sync-atomic-wasm workaround就失效了，这的确是对的。因为这时候这些gpu对象真的就不send和sync了。所以需要修改整个消费流程的实现，保证所有对gpu资源的操作必须都在device所在的线程完成

**另外使用中发现wasm_bindgen_rayon本身的一些小问题：**

- 生成的workerHelper js里中，对主js的胶水代码的引用路径是错误的
- 从浏览器的profiler看，似乎会实际生成两倍数量的worker，比如我设置线程数量4，实际上profiler看到有8个worker实例
- 从浏览器的profiler看，某些worker会进入永久的postmessage状态，原因不详
- 这个库只支持rayon global pool

其他事项

thread需要shared array buffer，需要serve的file添加相关COEP header。我现在用的是rust的[static-web-server](https://static-web-server.net/) 通过额外配置文件可以实现。这个server很好用
