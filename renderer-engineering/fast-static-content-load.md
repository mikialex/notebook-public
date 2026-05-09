# 支持动态场景的渲染器如何优化静态场景的加载性能？

渲染器的一个重要的技术定位/方向，是其是否支持动态的场景编辑？还是只支持静态的不可变场景。

我**不**认同静态场景比动态场景有更高的性能上限，因为任何静态的场景优化，如果其优化依赖的是运行时的算法，那么这个优化显然可以被动态场景采用。如果其优化依赖的是某种对数据的处理和组织方式，动态场景依然可以被采用（只是有更高的集成成本），即便我们不考虑细粒度的增量维护的能力，这种数据处理的本身可以动态的被执行来实现动态场景的优化。

只支持静态不可变场景所带来的技术优势，是场景的**加载**性能可以优化的非常彻底。因为静态场景不需要向用户提供支持符合编辑的业务逻辑模型的场景编辑API（这就是不支持动态性的体现）。所以加载器可以直接加载整个数据管线最后的数据（即gpu要用的数据）。假设我们不考虑某些平台因素的影响，场景加载性能的理论上限，就是直接一步加载（memory copy）gpu要使用的原始渲染数据。

一个支持编辑的渲染器从一个场景api到gpu所需的渲染数据的转化只能在运行时做，因为场景内容会在运行时变化/才能得知。进一步的，支持编辑的渲染器的核心实现是增量的实现这样的数据转化，以支持高性能编辑。

**一个以加载性能为卖点，并且不支持动态场景的渲染器，要完善的支持动态场景，其改造难度是非常高的。但是反过来，假设有一个完善支持动态场景的渲染器，我们其实可以 以低的成本，实现极致的静态场景加载优化。**

对于动态场景，我们有Scene，System，Renderer三个核心概念：

- Scene：一个可编辑的面向业务的场景API
- Renderer：一个渲染器实例，拥有绘制所需的一切资源
- System：实现Scene变更到renderer变更的增量计算系统，watch scene的变更，增量的维护renderer实例

整个应用的mental model大概是：

```rust
fn app() {
    let mut scene: Scene;
    let mut system: System;

    system.subscribe_changes(scene);

    for canvas in each_frame {
        user_bussiness_logic_modify_scene(&mut scene);

        let renderer: Renderer = system.incremental_maintain_and_create_renderer_instance();

        renderer.draw(canvas);
    }
}
```

要实现这样的静态场景加载优化，我们只需：

```rust
fn app() {
    let renderer: Renderer = load_from_path("my static scene");

    for canvas in each_frame {
        renderer.draw(canvas);
    }
}
```

所以实际只需要实现Renderer实例的高性能反序列化，即直接从磁盘/外部数据源 memory copy数据到gpu buffer上。反过来，要准备这样的静态优化数据，只需要实现Renderer实例的序列化。

我们一般会序列化反序列化场景，这个是常见的需求，这里我们**序列化和（高性能）反序列化renderer实例来实现高性能静态场景加载的支持**。

这种做法会导致静态场景的加载文件，耦合了具体的渲染器实现，但我们可以当作传统的兼容性问题（格式升级和降级兼容）来处理

## 混合模式

一个静态场景，其中某些部分也可能是动态的，至少camera会动吧。所以实际的技术方案是一种混合模式：renderer的实现是由不同的子renderer实现的（比如下面展示了rendiation的viewer renderer内部结构），这些子renderer有些可以是动态的，有些也可以是静态的。比如要解决camera的case，我们可以让camerarenderer采用上面的动态模式。

我们也可以同时支持动态scene和静态scene。做法是准备两个SceneRenderer，一个是从磁盘加载的，一个是动态维护的。后续的渲染逻辑，可以自由的去组装绘制逻辑。

```rust
pub struct ViewerRendererInstance {
  pub camera: CameraRenderer,
  pub background: SceneBackgroundRenderer,
  pub raster_scene_renderer: Box<dyn SceneRenderer>,
  pub batch_extractor: Box<dyn SceneBatchBasicExtractAbility>,
  pub mesh_lod_graph_renderer: Option<MeshLODGraphSceneRenderer>,
  pub clipping: ViewerClippingRenderer,
  ...
}
```

## 更新和渲染分离的架构

要实现上述设想，一个隐含的不太明显，但是重要的要求，是渲染器的更新和渲染必须是完全分离的，不能采用边渲染边更新的方式。在上面的语境中，就是renderer是在一个时刻明确的被创建出来，并且在后续渲染使用中，是immutable的。

更新和渲染是否采用分离模式，是一个非常重要的架构决策，这其实是一个值得展开讨论的问题，我可能会单独展开作为一篇内容。我早期接触和实现的渲染器采用的是边渲染边更新的模式，但我现在基本上可以确认更新和渲染完全分离才是正确的做法。其中一个原因就是采用这种做法能够支持上述这种优化。

除了这一优势，另外的主要原因是其更新过程可以被充分并行优化，因为渲染逻辑是串行的，所以边渲染边更新就很难做，几乎不能做并行更新。并且由于渲染时不需要通过脏标记检查确认是否需要更新，也避免了相关的overhead。
