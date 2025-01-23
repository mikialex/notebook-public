# gpu并行计算框架

图形api暴露了一套和cuda差不多的通用计算模型，compute shader。这套模型足够通用，但是比较底层。在实际工程中，直接采用这套接口来编写计算逻辑，虽然没有什么问题。但是其中有很大的潜力做一些高层的抽象和工程建设，以改进编程效率和性能表现。

改进编程效率，主要方式是屏蔽大部分重复的实现细节，提供一套高层次接口。在计算着色器的使用上，有非常多可以被复用的基本操作，比如prefix scan，reduction，histogram，sort，或者简单的mapping。  这些操作应该需要提取出复用和组合的能力，以直接改进编程效率。这些操作的目的和实现是可以分离的，其实现应该能够和其要表达的计算目标解耦。比如prefix scan本身有很多算法，算法本身也有诸多可以调优的参数，这些算法在不同的workload，不同的平台，有不同的适用范围。

假设我们直接采用gpu的compute shader，代码编写会非常冗余，充斥着大量的偶然复杂度。compute shader在没有高级工具的辅助下，排查和调试问题成本高昂。我们需要设计一个框架级别的解决方案，来隐藏实现细节是必要的。精简实现量，以在开发过程中减少低级问题出现的几率，减少代码编写的调试的实现。

假设我们直接采用gpu的compute shader，复用和组合这些并行计算的基本的结构在工程是困难的。我们需要设计一个框架级别的解决方案，来提取分离其实现目标和实现细节。使得独立的或者全局的，替换和改进实现，调优参数，来改进性能。

## 基本结构

基于edsl shader的基础设施的强力支持，使得我们可以编写出形态上类似rayon的gpu版本的并行计算框架。比如

```rust
let middle_result = my_input_data
  .map(|item|item_convert())
  .prefix_sum();

let final_result = my_input_data_2
   .zip(middle_result)
   .map(|(a, b)|do_sth(a, b))


```

为了实现上述形态的组合和复用能力，实现上具体设计了三层接口，这三层结构通过[组合生成组合](../../general-practice/composition-create-composition.md)的方式连接起来。

最底层是shader层面的 [edsl](../../renderer-engineering/edsl-shader.md) 接口。这个接口描述了一个并行计算的dispatch的invocation逻辑。实现需要在`invocation_logic`提供当前invocation的计算结果`T`和计算是否有效的 ，这样的计算要求是无副作用的。计算可能是无效的，比如`logic_global_id` 超过了这个dispatch的工作集范围。该接口可以被自由组合，实现内部可以调用其他`DeviceInvocation<T>`的实现

```rust
pub trait DeviceInvocation<T> {
  fn invocation_logic(&self, logic_global_id: Node<u32>) -> (T, Node<bool>);
  fn invocation_size(&self) -> Node<u32>;
}
```

中层的抽象是 `DeviceInvocationComponent<T>` ，该实现主要定义上述执行并行计算的dispatch的host端细节。这些细节包含：

- `work_size`：表达工作集大小，如果工作集的大小只能在device上获得，那么返回None。
- `ShaderHashProvider` ：通过shader hash来访问缓存的compute pipeline，该机制和[pipeline-management](../../renderer-engineering/pipeline-management.md)中的render pipeline access机制相同。
- `build_shader` 创建上述`DeviceInvocation<T>`，并在shader ctx下声明相关资源绑定
- `bind_input` 提供相关资源绑定
- `requested_workgroup_size` 表达这个dispatch所需要的workgroup size，如果不提供，那么runtime会采用某个全局默认值

`build_shader` 和 `bind_input` 这一对组合的实现是对称的，这和[edsl](../../renderer-engineering/edsl-shader.md) 提到的render部分的对称性完全一致，其底层也采用同一套binding的cache机制。

该接口可以被自由组合，实现内部可以调用其他`DeviceInvocationComponent<T>`的实现。来生成组合版本的`DeviceInvocation<T>`，其中，work_size可能是需要对齐的，workgroup size无法支持相互冲突的dispatch。`DeviceInvocationComponent<T>`的组合可以认为是两块并行计算逻辑在一个dispatch内的组合。

该接口会自动的提供 `dispatch_compute` 来执行该dispatch，以及 `compute_work_size` 来计算该compute 的worksize，并将该worksize自动的转化为适用于indirect dispatch的 size buffer

```rust
pub trait DeviceInvocationComponent<T>: ShaderHashProvider {
  fn work_size(&self) -> Option<u32>;

  fn build_shader(
    &self,
    builder: &mut ShaderComputePipelineBuilder,
  ) -> Box<dyn DeviceInvocation<T>>;

  fn bind_input(&self, builder: &mut BindingBuilder);

  fn requested_workgroup_size(&self) -> Option<u32>;

  fn dispatch_compute(
  &self,
  cx: &mut DeviceParallelComputeCtx,
  ) -> Option<StorageBufferReadOnlyDataView<Vec4<u32>>> {
    // some default implementation
  }

  fn compute_work_size(
    &self,
    cx: &mut DeviceParallelComputeCtx,
  ) -> (
    StorageBufferDataView<DispatchIndirectArgsStorage>,
    StorageBufferDataView<Vec4<u32>>,
  ) {
    // some default implementation
  }
}
```

高层抽象是`DeviceParallelCompute<T>`, 最上方的sample 代码，用户操作和组合的结构，就是该抽象的实现。

实现者需要提供 `execute_and_expose` 来表达完成该并行计算任务的所有逻辑，并将结果以一个
`DeviceInvocationComponent<T>` 的方式暴露出来。如果完成某计算需求，只需要一个 `DeviceInvocationComponent<T>` 那么可以直接返回这个component，这样组合实现就可以尽可能的直接组合这一上游的component，避免materialization的成本。

这层抽象存在的核心原因就是某些并行计算需求无法在一个dispatch中完成，需要多个。materialization只在不得不执行的时候执行，避免了任何带宽上的浪费，同时实现了最佳的组合能力。materialization还承担了获取最终结果的功能，由框架提供默认实现，某些算子，需要定制化的materialization实现，也可以实现于此。用户还需要一个result size来表达工作集的保守大小，以便于materialization的target准备合适的容量来承接输出。

```rust
pub trait DeviceParallelCompute<T>: DynClone {
  fn execute_and_expose(
    &self,
    cx: &mut DeviceParallelComputeCtx,
  ) -> Box<dyn DeviceInvocationComponent<T>>;

  
  fn result_size(&self) -> u32;

  fn materialize_storage_buffer(
    &self,
    cx: &mut DeviceParallelComputeCtx,
  ) -> DeviceMaterializeResult<T>
    where
      T: Std430 + ShaderSizedValueNodeType,
  {
    // some default impelementation
  }

```
