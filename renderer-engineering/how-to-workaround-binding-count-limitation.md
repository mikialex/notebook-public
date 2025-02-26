# 如何解决Buffer binding count 的平台限制

## binding个数限制的问题

buffer binding count的平台限制问题，在webgpu的ctx下具体是指：

- maxUniformBuffersPerShaderStage：在一个shader内最多可绑定的uniform buffer个数
- maxStorageBuffersPerShaderStage：在一个shader内最多可绑定的storage buffer个数
- maxVertexBuffers：在vertex shader内所有绑定的buffer的上限个数

标准参考：<https://gpuweb.github.io/gpuweb/#limits>

可访问 <https://webgpureport.org> 来查看当前浏览器的支持情况。在我的设备（MacBook pro M2pro）上，可查询到这些限制是

- maxUniformBuffersPerShaderStage： 12
- maxStorageBuffersPerShaderStage： 10
- maxVertexBuffers：8

这些数字都很小，特别是storagebuffer的支持数量很少，这对一系列需要使用很多storage buffer的feature影响很大。

这些限制有时并不直接是硬件设备的限制，而是浏览器出于某些原因强加的。如果在上述设备上采用wgpu native版本运行，实际上maxStorageBuffersPerShaderStage的限制是31。在我的其他测试机器，如4090上，storage buffer的shader支持数量超过1百万，但是浏览器依然仅提供 10 的支持数量。

解决这些buffer binding count的问题是必要的，因为我们无法假设浏览器后续会放松这些限制，这些限制可能出于安全或者资源控制的角度是必要的。并且在某些低端设备上，binding的数量确实是受限的。更进一步，即便是M2pro这种并不算低端的设备，只有31个storagebuffer binding支持对于实现某些feature也是不足的。

## 限制造成的实际问题

对于一个完全gpu driven的渲染器，其中一个必要实现是需要能够在一个shader invocation内，访问到所有的场景数据。实现这一点是通过将场景数据维护在storagebuffer内然后暴露给shader达成。场景结构越复杂/支持能力越丰富，意味着场景中数据的类型越多，意味着需要更多的storagebuffer实例来维护per type的场景数据，意味着maxStorageBuffersPerShaderStage直接制约了gpu driven渲染能够支持的场景复杂度。从实际情况来看，10个一般是不够的。

除了gpu driven，某些特殊的feature或者runtime实现，也需要大量的storage binding count支持。比如实现gpu上的taskgraph，如果不做任何改进，那么storage binding count直接决定了max的task type count。

这一问题可以workaround，比如gpu driven rendering，的确是有可能手工的将数据通过引用fanout到其他类型上来减少类型数量，但对于更加通用的情况，就很难通过调整数据结构来解决问题。

通过将数据进行复制来减少数据类型以减少storagebuffer个数的workaround方式，即便在某些场景下可以操作，当也并不是合理的解决方案。因为这会引入显著的运行成本，比如host端的数据复制的device端的更高带宽消耗。此外实现也会变得复杂，这些大量偶然复杂度的存在仅仅是为了让feature能够在某几个受限平台正确运行，从工程上来说是糟糕的。为了让不受限平台不必付出这样的降级成本，甚至需要引入更多的复杂度，比如动态的降级控制能力。

## gpu 编程模型的缺陷

另一类workaround的方案，是将多个buffer直接合并，即直接allocate在一个buffer内。这么做避免了上述的运行成本，实现成本和实现形态也非常可控。但是一个致命问题是shader内就没有办法直接写出这样的合并类型：一个binding可以使用一个runtime sized的类型（比如runtime size array，或者tail是runtime size array的struct，但是一个binding无法使用多个runtime sized的类型。从shader访问逻辑上，你还是需要多个runtime sized的类型。

所以采用这种workaround，这个binding只能使用`[u32]`这种无类型buffer作为绑定类型。这么做，意味着shader的structral access，整个类型系统就完全不可用了，需要用户自己在这个u32 heap的正确的位置手工loadstore数据来模拟出shader的类型系统。这么做会导致无法接受的实现复杂度，对于长期需求多变项目，成本角度实际上是不可行的。而如果试图实现动态切换实现的要求，就意味着必须要维护两份实现，这两份实现的统一是非常困难的。

这种`[u32]`手工实现数据读写的做法，比如[vello](https://github.com/linebender/vello)就是采用了这种方案。

我认为这个问题表现出gpu编程模型上的一个典型缺陷：shader内的ptr类型是无法直接cast为另一类型的ptr。假设这样的ptr cast是存在的，那么就可以`[u32]`的一部分slice出来，然后transmute到给定的runtime sized type直接进行读写访问。因为缺少这种机制，所以用户不得不自行模拟类型系统提供的功能。

这个缺陷的存在可以说也让bindless buffer变得毫无意义。因为声明bindless buffer array，也只能指明唯一的类型。而如果我有大量同一类型的变长buffer数据需要绑定，我完全不必使用bindless buffer，只要把它们allocate在一个容器里然后管理好offset和size即可，这甚至给予更多存储行为控制的能力。

## 基于 shader edsl 的解法

假设shader逻辑都采用了[edsl](./edsl-shader.md)框架，那么上述的`[u32]`手工实现数据读写的工程成本就会魔法一般的完全消失，以至于能够完美支持一份shader实现，运行时动态切换决定是否采用这样的降级实现。

其核心原因是在shader edsl框架中，每一个不同类型的ptr node，并且其底层的ptr实现是完全抽象。这个abstract ptr的默认实现，就是原生的shader中的原生ptr。对于u32 heap而言，我们可以通过`(ShaderPtrOf<[u32]>, Node<u32>, ShaderTypeInfo)`来实现出一个「软件」版本的abstract ptr。其中array ptr就是整个buffer的ptr，u32 node就是当前这个虚拟左值的地址offset，同时还保留类型信息，即记录了这个array的这个offset，存储了ty类型数据。

```rust
pub trait AbstractShaderPtr: DynClone {
  fn field_index(&self, field_index: usize) -> BoxedShaderPtr;
  fn field_array_index(&self, index: Node<u32>) -> BoxedShaderPtr;
  fn array_length(&self) -> Node<u32>;
  fn load(&self) -> ShaderNodeRawHandle;
  fn store(&self, value: ShaderNodeRawHandle);
  fn downcast_self_as_atomic_ptr(&self) -> ShaderNodeRawHandle;
}
```

这个软件版本的shader ptr可以完美实现abtract shader ptr的所有接口：field ptr/array ptr的access的实现是计算地址偏移，以生成另一个偏移后地址+子field类型的ptr，而load store的实现实际上就是数据在shader内的序列化和反序列化。因为ptr始终持有类型信息，所以可以直接通过运行时反射的方式遍历描述类型本身的数据结构，自动化的生成所有索引计算和序列化的shader代码。

我近期在我的side project里完整验证了这一想法的可行性。

在实现过程中，有一些细节比较有趣：

### Binding buffer

必须有一个buffer来记录绑定的bufferid，即记录binding_index到buffer_allocation_index。这个buffer在用户构造binding的过程中构造，并且需要一同绑定到shader中。我这里将其称之为Binding buffer。

binding buffer其实就是一个软件版本的bindgroup。这个东西是必需的，但是其必要性很微妙，以至于我在编写实现的后期才意识到它的必要性。即：shader对资源的reference，并不能建立在实例的引用之上，而是应该建立在binding的顺序index。

假设你的allocator内allocate了ab两个子buffer。这两个buffer的类型是一样T。假设有某一个shader，需要依次绑定两个T的实例，它在第一次运行的时候，假设绑定顺序是a，b，后一次运行，绑定顺序是b，a。假设shader依赖了buffer实例的信息，那么相当于a，b的实例信息（比如buffer的alloation index）被直接硬编码到了shader中，必然会导致第二次运行依然采用第一次的绑定顺序/数据。

所以解决这个问题就只能自己模拟bindinggroup，然后在bind时，根据bind index在shader运行时去查询binding buffer来找到当前实际绑定的buffer实例是哪一个来重建初始指针的offset。

### Buffer header

因为buffer index的访问是shader runtime的，所以初始offset的信息也是需要在shader runtime进行访问，些信息并不需要在独立的buffer中记录，而是记录在合并的buffer的头部header即可。支持runtime sized array类型的shader ptr，意味着需要支持实现arraylength的api，意味着每一个sub buffer的size在shader runtime也是需要访问的。这些信息并不需要在独立的buffer中记录，而是记录在合并的buffer的头部header即可。

上述的binding buffer并不能合并在header内。因为其生命周期类似bindgroup，并且和bindgroup一样，用户可以持久化来避免重建开销，也意味着它可以同时存在多个实例。

### Atomic buffer

对于webgpu/wgsl而言，atomic操作只能在atomic类型上进行，atomic是类型化的。这导致atomic的数据比如atomic u32array， single atomic u32 无法存储在`[u32]`，所以这些数据不能和普通数据进行合并，而是需要采用特别的合并器，内部采用`[atomic_u32]`作为heap。同样的因为atomici32无法存储于`[atomic_u32]`，所以i32类型的atomic也是通过特别的合并器独立区分出来。atomici32和u32的loadstore有可能可以通过bitcast进行patch，但是对于一些atomic操作，这样的patch可能是不可行的。

### Aliasing

采用这套实现，即便virtual binding是mutable alias的，也可以通过validation layer的 binding alias检查，因为自始至终都只有一个mutable的buffer binding。所以在virtual binding的角度，用户可以将同一个buffer实例，bind两个mutable的binding。虽然我没有做检查，但是这么做是危险的，比如在意想不到之处造成同步问题。如果要通过开关切换buffer merge是否动态开启的情况，alias检查又会动态开启，造成validation失败。

### Layout and alignment

ptr的abstract实现依赖于类型反射，而具体的offset的读写，需要明确type的memory layout。我实现了了140，430和packed 三种layout。
实现140和430的原因是因为，用户可以自由的将子buffer slice出来在一个正常的非合并shader中直接通过原生ptr访问数据，同时方便相关的host读写。packed layout的内存占用最小，因为其中不包含任何padding。

支持sub buffer独立bind的另一个重要细节是sub buffer的起始位置受到uniform和storage 的global alignment限制，比如256byte，所以allocator需要处理好其内存分配的aliagment。

### Vectorize load store

据说，采用`[vec4<f32>]`相比`[u32]`作为heap能够改善load store性能。因为一次可以load4个值。这个优化我目前没有实现。而且采用这个优化，意味着用户就不应该使用packed layout，因为此时就任何数据都需要4个f32的alignment。

## 结语

通用的无限制buffer绑定能力应该是支持渲染器动态扩展能力的核心技术。用户的场景定义的复杂度是unbound的，其资源结构的复杂度是unbound的，以目前图形api的binding支持能力，考虑到不同设备，binding的limitation会是场景/渲染的扩展能力核心制约因素。edsl abstract ptr+buffer合并的技术方案可以说完美解锁了相关限制。其可以做到完全的运行时动态能力：运行的合并动态个数不同类型的buffer，同时隔离一切，无感保持各个effect子组件的静态shader内的检查能力。
