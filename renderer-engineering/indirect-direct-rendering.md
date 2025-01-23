# indirect和direct rendering的工程实现组织

## 术语

有两种渲染方式：direct和indirect。direct draw即gles模式的渲染，一个一个的提交draw call，一个drawcall一个drawcall的绑定所需资源。indirect draw即gpu driven模式的渲染，一次绘制所有（大批量的）内容，一次绑定所有所需的资源。

indirect是较为先进的高性能渲染方式，但是对平台支持有更高要求，比如依赖indirect draw等能力。direct是传统的，兼容性好的渲染方式，为了最佳兼容性，甚至不会使用storagebuffer，以实现gles300target （比如webgl2）的运行支持。

## 资源

direct rendering，一方面因为其本身设定的绑定方式，另一方面因为兼容性而采用的uniformbuffer，所以buffer最大尺寸有限制。一般认为无法将同类型的所有数据储存于一个uniform buffer内，所以direct资源需要per instance存储。而indirect会使用storagebuffer，几乎没有这样的限制，所以可以per type存储。

因为direct 是per instance的，所以binding也是per instance级别（instance指场景基本绘制单元），所以维护binding的成本和绑定binding的成本都远高于indirect。

## 寻址

因为direct draw的资源是per draw的，pre draw需要获取到这个draw特别所需的资源来进行绑定，所以需要在host端寻址。需要维护相关host的辅助数据（即场景的引用关系）以支持这样的寻址。

indirect draw虽然是合并绘制的，但是在device上依然需要获取当前draw的这个primitive的具体绘制信息，所以这样的一套寻址依然存在。indirect模式的资源为了支持device上的寻址，其辅助数据，即场景的引用关系需要维护在gpu resource内，而direct模式的资源并不需要。

寻址的入口，应该采用我在[basic-scene-object-model](../editor-engineering/basic-scene-object-model.md) 中提到的场景基本单元。基本单元会引用或间接的引用其他具体绘制信息。在host端，即通过这些reference或者handle去access 资源，形成handle的access chain。在device端，这些reference一般的被实现为u32 handle，这些资源以array access的方式在gpu上被寻址，形成`Node<u32>`的access chain。因为两种方式的绘制逻辑一致，场景数据结构一致，所以两种寻址方式是同构的。

对于indirect渲染，无论是采用draw indirect 还是 multi draw indirect，清晰的提供寻址的入口是必要的。比如 multi draw indirect 降级 draw indirect的关键就是复原其draw unit的区分能力，即复原其提供寻址入口的能力。对于indirect模式，用户也可以直接构造一个仅包含寻址入口信息的buffer来直接实现direct渲染。因此indirect模式中，提供寻址入口的实现在工程上是抽象的。

```rust
pub trait IndirectDrawProvider: ShaderHashProvider + ShaderPassBuilder {
  // indirect draw的入口的实现者肯定要提供这个draw的具体命令
  fn draw_command(&self) -> DrawCommand;
  // 并且当然要提供一个在shader ctx下访问当前primitive 寻址入口信息的方式
  fn create_indirect_invocation_source(
    &self,
    binding: &mut ShaderBindGroupBuilder,
  ) -> Box<dyn IndirectBatchInvocationSource>;
}

pub trait IndirectBatchInvocationSource {
  // 这里的 scene_model_id 即 root寻址入口
  fn current_invocation_scene_model_id(&self, builder: &ShaderVertexBuilder) -> Node<u32>;
}

```

multi indirect draw的commandbuffer，是根据draw unit id buffer生成的，生成过程需要在device上寻址mesh相关的数据。device上的剔除实现，剔除的目标draw unit id buffer，剔除过程中需要寻址剔除所需的数据。实现和维护device上的场景数据寻址能力，是indirect 渲染和资源管理的主要工程难点。

## 实现选择

实现选择指的是，给定场景entity，渲染器能够从已有注册实现中提取出渲染这个entity的具体实现。渲染框架能力的扩展性，一方面依赖于场景数据结构的扩展性，另一方面就依赖于渲染器渲染实现的注册机制。这一实现选择和上述的资源寻址比较类似，可以视为是对已注册实现的寻址。这一部分实现先于资源寻址，比资源寻址更加基本和重要，也比较简单。

用户通过实现寻址，来得到自己这种「场景类型」的渲染能力支持。然后在这种能力支持上，做具体渲染的资源寻址，来得到自己这种「场景类型」的具体「实例」的渲染能力支持，然后这个渲染能力支持用来直接表达图形api层面的渲染。

根据用户组装场景来组装渲染能力的设计思想，渲染实现的寻址是从场景结构本身的读取解析出发的。基本形式是访问查询某种动态的类虚表结构。这种虚表可能通过决定场景渲染的场景对象的typeid来访问，也可能简单的通过依次匹配实现。

对于direct draw 和indirect draw，显然这样的实现选择也是同构的。实现选择是一个host only的逻辑，indirect draw的实现选择也只能实现于host端（因为图形api（一般）不支持device切换pipeline），所以 indirect draw的drawlist input，不仅仅要在device上提供entity的迭代能力，还需要直接持有一个host端的entity handle，用来做实现选择。这个entity handle，可以实际上不属于drawlist，但是其选择的实现，必须能够正确的按照需求来渲染这个indirect draw list

## 在顶层统一渲染接口

因为direct和indirect这两种方式，所需要维护的资源类型不同，渲染方式不同，渲染中的资源寻址虽然同构但是实现完全不同。所以实际工程上，两种模式只能被分开实现，但是在顶层draw unit和draw scene的层面可以提供统一的抽象渲染结构。

```rust
// 描述了一个渲染器能够提供以基本单元为粒度的绘制能力
pub trait SceneModelRenderer {
  fn render_scene_model(
    &self,
    idx: EntityHandle<SceneModelEntity>,
    .. other necessary parameters
  );
}

// 描述了一个渲染器能够提供批量基本单元的绘制能力
pub trait SceneRenderer: SceneModelRenderer {
  fn render_scene_model_batch(
    &self, 
    batch: SceneModelBatch,  
   .. other necessary parameters
  );
}

// 批量基本单元，可以让用户提供device端数据或者host端数据
enum SceneModelBatch{
  Host(Vec<EntityHandle<SceneModelEntity>>), 
  // 后一个entity handle用于上述的实现选择
  Device(DeviceVec<EntityHandle<SceneModelEntity>>, EntityHandle<SceneModelEntity>)
}

```

对于direct模式的渲染器，实现SceneModelRenderer是直观的，实现 SceneRenderer可以通过foreach调用SceneModelRenderer来自动实现。

对于indirect模式的渲染器，实现SceneRenderer是直观的，实现SceneModelRenderer可以通过立刻构造一个目标draw unit id buffer提供寻址入口来自动实现。

如果direct渲染器在batch模式遇到了device版本的batch，可以拒绝实现或者同步读回转化为direct实现，这种情况一般是不会出现的，因为用户没有理由在一个不能使用indirect渲染的平台构造device版本的batch。如果indirect渲染器batch模式遇到了host版本的batch，那么可以立刻调用内部系统来转化为device版本batch。

虽然indirec和direct是两套实现，但是因为其资源和寻址的结构是对称的，所以在实现上，基本的目录结构应该对称，实现要对称，命名上通过后缀来区分，以利于长期维护。对于两者都支持的绘制内容，测试系统可以自动的回归对比。
