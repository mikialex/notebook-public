# EDSL Shader Framework

## 将shader视做为运行时可编程的数据结构

将shader视作和texture，mesh一样的静态资产（文本text）是错误的工程实践。

一方面这是因为着色器逻辑对于「现代」渲染器来说是高度动态的。另一方面这么做严重制约了着色器编写的编程体验和工程质量。

考虑动态性，其主要来源在于：

渲染框架的扩展性需求：扩展能力实现的核心思想在于用户可以自由的组合一些场景的“组件”或者效果的“组件”，并基于这样的组合的形态自动的生成合理的渲染能力（场景内容的渲染能力，效果的组合）。而这种方式事实上是一种**弱化的动态可编程能力**：用户动态的修改场景和效果描述来动态的决定渲染效果，意味着需要具备动态的创建不同的pipeline，及绑定资源的动态结构的能力。在[binding system](binding-management.md)中，我们讨论了这样顶层框架在资源方面的的动态组合能力，而对于pipeline及shader，这样的动态能力也是必须的。

另一方面，某些渲染能力或者外部IO格式的支持，要求完整的动态着色器能力，比如用户的可视化着色器编辑器，shadergraph。又比如现在高级的材质描述文件格式，不仅包含材质参数，还直接的包含了着色逻辑，比如 materialX。

即便场景和效果是完全静态，也完全不追求上述可组合的渲染架构。shader仍然具备一定的动态性需求。比如典型的兼容性要求：在不同客户端情况，不同的性能考量下采用不同的shader。比如某些材质效果参数出于性能等考虑不希望通过shader内的动态分支实现。比如某些环境级别的配置需要应用于所有shader。

在传统的渲染器中，上述的某些充分弱的动态性需求（比如第三点），一般是可以通过define和include等预编译机制简单实现的。随着项目的发展，以我实际历史上接触和维护的项目，阅读的第三方实现，以这种方式维护的动态性会造成严重的维护性问题。比如文档缺失不匹配，难以设计正确合理的组合方式，难以控制配置之间的相互影响。难以匹配动态的应用程序端资源结构和数据准备，以及最基本的：难以直观的阅读实际采用的着色逻辑。我在这方面的工作体验非常糟糕，以至于是我思考这一系列渲染器工程化改进的直接原因。

传统做法的核心问题，在于预处理机制是通过字符串以及模版操作来强行模拟和实现着色器的动态能力。通过字符串操作来实现编程语言的动态性是非常糟糕的。要实现真正的动态性要求我们正确处理着色逻辑的动态性。正确处理的方式，是将着色逻辑作为结构完备的可编程目标。

相对于完全无结构的字符串，应该采用某种AST形式的IR作为着色逻辑的数据结构。采用ast而不是更加lowlevel的其他表达，比如流图，是因为ast方便生成实际shader代码，也使得用户可以以一种简单直观的方式构造和修改shader逻辑。在后文的论述中，我们可以显而易见的看出这种做法相对于传统做法的工程优势。

## EDSL

这种推荐的构造shader ast的方式，称之为EDSL：用户通过一套host语言的api来直接构造ast ir，以编写shader逻辑。这里介绍一下api的核心使用方法，它非常简单。

在这样的api中，shader 运行时的右值通过`Node<T>`表达，`T` 是node的类型。node之间可以支持有各种运算符和方法。用户可以自由的为这些api操作整理为独立的函数，因为只是简单的rust的函数，所以会直接shader代码中是直接内联的。

```rust
fn add(a: Node<u32>, b: Node<u32>) -> c: Node<u32> {
  a + b
}

fn some_shader_logic() {
  let a = val(1);
  let b = val(2);
  let c = add(a + b) * val(4);
  let d = c.max(val(10));
}

```

在shader中，根据不同的变量的存储要求，有不同类型的左值，这些左值根据其存储的地址空间的差别有不同的类别，比如局部变量`LocalVarNode<T>`, uniform buffer: `UniformNode<T>`, storage buffer: `StorageNode<T>`, texture/sampler资源 `HandleNode<T>`，这些不同类型的左值一般都支持load和store方法。

```rust
let a = val(1).make_local_var();
a.store(a.load() + val(1));

```

如果是shader运行时的控制流则可以这样的api表达

```rust
let a = val(true);
let b = val(1).make_local_var();

if_by(a, ||{
  b.store(val(2))
});

let i = val(0).make_local_var();


loop_by(|cx|{
  if_by(i.equals(val(10)), ||{
    cx.do_break()
  });

  i.store(i.load() + val(1));
})

```

而凡是采用host语言编写的控制流，都作用于shader的编译期。所以可以采用这种方式动态的决定shader逻辑：

```rust
let should_use_a_effect = false;

if should_use_a_effect {
  // inject a effect logic
} else {
  // inject other effect logic
}

```

## 混合的动态性

edsl的引入让我们以一种类似multi stage programming的方式看待整个shader创建流程。

- 通过rust的const 代码，表达rust编译期要确定的事情。比如rust的array的长度（这个和shader也有关系，因为我们用rust类型来生成shader类型，rust的array是可以直接投影到shader的定长array的）。
- 通过普通的rust代码，表达shader编译期要确定的事情。比如弱动态的shader本身的变体生成，比如强动态性的根据运行时input完整生成shader。比如rust loop就是相当于做循环展开的优化。
- 通过edsl的`if_by`和`loop_by`等控制流构造器，来表达shader运行时要做的动态逻辑。

在阅读edsl shader代码时，这三种动态性是混合的。可以根据这些参与的值的具体类型（上述的哪一种），来了解实际上代码在动态性上做的保证。当你看到一个rust变量是Node类型，那么你就知道这是一shader运行时的值，当你看到if_by，那么这是一个shader运行时的branch。这种stage的重要区别，并不会因为混合编程就会被忽视和混淆，反而通过类型得到显式的区分。

## shader的静态检查

在edsl库中，可以将着色器代码的静态类型，实现为edsl库api的静态约束，通过host语言的静态类型系统，在编译期检查shader逻辑的类型错误。

因为在shader中，我们的值都是通过`Node<T>`这种handle表达的，所以这种检查的基本形式，就是在方法和运算符实现上对`T` 施加必要的约束。对于shader中的primitive类型，比如vec和mat，edsl会提供标准的host端数学库，这个host端的数学库上定义的若干trait bound，可以直接被「投影」到device版本。比如:

```rust
impl<T, U> Add<Node<U>> for Node<T>
where
  T: ShaderNodeType + Add<Output = X>,
  U: ShaderNodeType,
 .X: ShaderNodeType,
{
  type Output = Node<X>;

  fn add(self, other: Node<U>) -> Self::Output {
    // implementation
  }
}
```

这段实现主要定义了：`Node<T>` 和 `Node<U>` 可以被相加，当且仅当`T`和`U`可以被相加，并且如果`T`和`U`相加的结果类型为`X` ，那么 `Node<T>`和`Node<U>`的相加的结果类型是 `Node<X>`

如果项目的数学库直接采用的是用于镜像定义shader api计算规则的数学库，那么编写device版本的数学计算，和host版本的数学计算，具有基本完全一致的编程体验。甚至通过进一步应用某些泛型编程技巧，可以使得device和host的计算逻辑共用同一份实现。

## 强类型 IO

目前以上所展示的一些api，都只是一些基本的运算能力，但是计算input从哪里来？如何output？

io有两种。一种类型的io是固定，input是shader stage中固定会提供的builtin变量，output也是shader stage的固定约定。input比如vertex shader中的vertex index，或者compute shader中的global invocation id。output 比如vertex shader中的vertex position。

这种固定类型的io，取决于用户正在编程的shade stage，所以有必要为这些stage提供强类型的contex api，这样用户就可以从这些contex上访问这些固定的input，并将计算的结果设定为在固定的ouput上。

### 资源容器类型决定IO Node类型

另一种input是用户绑定的资源，比如buffer，texture，用户定义了数据的存在和具体读写方式。支持这类io的方式主要是让用户来定义这些资源在shader中的input。其通用的编程痛点是：在渲染时，要确保绑定正确的资源到合适位置。即用户的resource类型要和shader中定义类型匹配。

为了通用的解决这个问题，我们分别为resource和shader中的资源类型提供强类型的定义，然后通过trait的关联类型将这样的类型映射关系表达。然后在绑定流程中，binding的接口上采用上述trait，以自然的让resource生成shader类型。

```rust
pub trait ShaderBindingProvider {
  type Node: ShaderNodeType;
}
```

比如texture资源上，我们定义有`GPU2DTextureView`, 其shader中匹配的类型是`Node<ShaderTexture2D>`

```rust
macro_rules! map_shader_ty {
  ($ty: ty, $shader_ty: ty) => {
    impl ShaderBindingProvider for $ty {
      type Node = ShaderBinding<$shader_ty>;
    }
  };
}
map_shader_ty!(GPU1DTextureView, ShaderTexture1D);
map_shader_ty!(GPU2DTextureView, ShaderTexture2D);
map_shader_ty!(GPU2DArrayTextureView, ShaderTexture2DArray);
map_shader_ty!(GPUCubeTextureView, ShaderTextureCube);
map_shader_ty!(GPUCubeArrayTextureView, ShaderTextureCubeArray);
...
```

用户在得到texture node后，就可以调用其上定义的若干强类型强约束的texture各种采样方法，获取元数据的方法来完成业务需求。这些接口都是精心添加了合适的bound，并完全match底层标准。比如只能通过`Node<Vec3<f32>>`采样`Node<ShaderTextureCube>` ，比如`Node<ShaderDepthTexture2D>`只能通过`Node<ShaderCompareSampler>`采样。

### 强类型的Buffer接口

虽然texture比较确定，但是buffer的部分，是较为自由和复杂的。 为此我们进一步设计了一系列设施来改进和解决其中编程上的困难。图形api中buffer相关的一类典型的工程问题，在于

- 资源创建的接口只能创建和管理无类型的byte buffer
- render时绑定的也只是无类型的byte buffer

然而，从shader的角度，你在shader中定义的buffer，都是有具体数据类型的。

所以一般而言，用户需要手工的保证

- 在buffer数据更新和准备时，在byte buffer上的读写的位置需要符合shader中类型定义的layout
  - layout规则会很奇怪，比如std140
- 和上文一样，需要在渲染时，确保绑定正确类型的无类型buffer到合适位置

这样的手工保证是易错的，特别是随着项目的变更，可能就会出现不一致的情况。因为从图形api侧来看都是无类型的buffer，所以底层的实现方无法对资源绑定的正确性做完整的validation。这可能使得shader访问完全错误的数据但是却没有任何报错。buffer的定义和使用是非常基础的和广泛的，而排查，维护这样的正确性的工作量和心智负担非常高昂。

解决绑定错误的问题，同理需要对资源进行强类型封装，比如内部存储`MyStruct`类型的无类型`UniformBuffer`，现在会存储于`UniformBuffer<MyStruct>`。然后在构建shader时，和上述「资源容器类型决定IO Node类型」的设计一致：用户通过`UniformBuffer<MyStruct>`的实例或者显式的类型指定来自动的生成 `ShaderReadonlyPtrOf<MyStruct>`（具体细节见后文）. 由此自动的保证「绑定正确类型的无类型buffer到合适位置」。这样的保证是被host类型系统自动检查保证的。

解决layout匹配的问题，我们可以通过对`UniformBuffer<T>`的`<T>`添加和layout相关的静态约束实现。比如对于Uniform来说，就会有`Std140`这样的trait。这样就可以避免用户错误的将不符合layout规范的数据类型放入该容器中。这样的trait是unsafe的，除了我们内部会对基本数据类型做实现，我们一般不推荐用户实现这样的trait，而是

通过宏，让用户来对`MyStruct`的类型定义进行标注。比如下图中的 std140_layout 宏

```rust
#[repr(C)]
#[std140_layout]   <- this one
#[derive(Clone, Copy, ShaderStruct, Default)]
struct MyStruct {
  pub direction: Mat4<f32>,
  pub sample_count: u32,
  pub roughness: f32,
}
```

这个宏会做两件两件事情

- 注入隐藏的padding来保证public field的layout和device是匹配的，使得用户在host端构造出的此数据，其内存布局满足device访问的要求。
- 为该struct实现layout的`Std140` trait 来标记此数据可以被存储于`UniformBuffer<T>`容器中

由此，我们就彻底的，静态的，自动的保证了数据绑定，layout，出现错误的可能

### struct 如何实现 field access？

假设上文中的`UniformBuffer<MyStruct>` 正确映射进入shader成为 `UniformNode<MyStruct>` ，那用户该如何访问其field？这个问题是上述代码片段中的ShaderStruct宏解决的。这个宏会自动的生成

- 另一个struct定义，`MyStructShaderInstance`，用户也可以通过`ENode<MyStruct>`指代此struct
- `ENode<MyStruct>`和`Node<MyStruct>`的相互转化的实现，分别为expand和construct

`MyStructShaderInstance`的结构为

```rust
#[derive(Clone, Copy)]
struct MyStructInstance {
  pub direction: Node<Mat4<f32>>,
  pub sample_count: Node<u32>,
  pub roughness: Node<f32>,
}
```

用户可以通过构造`MyStructInstance`，然后compuse出 `Node<MyStruct>`。反之，用户如果已有 `Node<MyStruct>` 也可以expand出`MyStructInstance`。并在`MyStructInstance`直接通过field access来访问子field。如果struct的field是用户定义的其他struct，那么就自动的形成嵌套关系。

### 如何表达pointer semantics

当某buffer被绑定到pipeline后，其仅仅是暴露了其访问地址，用户需要通过某种方式将数据从地址中load出来或者store进去。这种load store方式，是需要用户显式控制的，因为和性能和正确性有很大关系。简单来说比如用户绑定了一个巨型的array，但是只会访问其中若干几个元素，那么load完整的数据是不现实的。又比如我们编写compute shader，其中对带宽，访问pattern，同步控制的细节实现，都要求我们直接表达。

展开而言，edsl是否支持pointer semantics，是其中非常重要的设计考量。如果不支持，那么其计算模型可以简单的规约为DAG，大幅简化了实现。但是在控制流支持和上述的io支持上会遇到非常高难度的设计问题。如果要维持纯粹的DAG设计，基本上需要应用函数式编程的思想。一个方向是直接引入phi节点（SSA的思想其实就是函数式编程的思想），另一个方向是纯粹的采用递归和continuation来表达计算。可以说在shader编程的角度看，这些方向在工程上，实现成本上基本上没有尝试的必要。这样的取舍，意味着edsl的形态就对齐到了ast，而不是图式的ir。

在我们edsl的设计中，为了支持pointer semantic。我们做严格的左右值区分：我们视`Node<T>`是右值，它是直接在计算中可用的。而代表某个地址的，或者持有实际存储地址信息的Node视为左值。左右的区别完全体现在类型系统上，在底层，左右值采用相同的representation。左值上暴露了load store方法来支持右值的修改。

对于shader而言，不同的左值有不同的地址空间，比如一个局部变量(private)，一个全局变量(global)，一个用户绑定的buffer (storage, uniform)，workspace变量。为了简化编程模型，在edsl层面我们静态的只对左值的readonly还是readwrite进行区分。因为我们需要静态的拒绝对uniform类型左值的write方法。

和上述`MyStructShaderInstance`一样，ShaderStruct宏还会生成 `MyStructShaderPtrInstance` 和 `MyStructShaderReadonlyPtrInstance`，来支持左值内部的结构化访问。并且和`ENode<MyStruct>`一样，通过`ShaderReadonlyPtrOf<MyStruct>`来映射`MyStructShaderReadonlyPtrInstance`，通过 `ShaderPtrOf<MyStruct>`来映射`MyStructShaderPtrInstance`。内部的结构化访问的具体上市，是通过`MyStructShaderPtrInstance` 和 `MyStructShaderReadonlyPtrInstance`上暴露的方法实现提供：

```rust
impl MyStructShaderPtrInstance {
  pub fn direction(&self) -> ShaderPtrOf<Mat4<f32>>;
  pub fn sample_count(&self) -> ShaderPtrOf<u32>,
  pub fn roughness(&self) -> ShaderPtrOf<f32>,
  pub fn load(&self) -> Node<MyStruct>;
  pub fn store(&self, v: Node<MyStruct>);
}

impl MyStructShaderReadonlyPtrInstance {
  pub fn direction(&self) -> ShaderReadonlyPtrOf<Mat4<f32>>,
  pub fn sample_count(&self) -> ShaderReadonlyPtrOf<u32>,
  pub fn roughness(&self) -> ShaderReadonlyPtrOf<f32>,
  pub fn load(&self) -> Node<MyStruct>;
}
```

其中对于`ShaderPtrOf<Mat4<f32>>`这种基本类型，其基本实现上直接提供了load和store方法。所以假设用户持有一个`ShaderPtrOf<MyStruct>` ptr，想要load其中roughness字段，那么就可以通过`ptr.roughness().load()`来实现。如果struct的field是用户定义的其他struct，那么就自动的形成嵌套关系。

框架除了会提供mat，vec等基础primtive的ptr实现，还会提供array和static size array的实现，并正确的将它们和一些常见的host绑定类型进行关联，以实现完整的结构化数据访问的需求。

### 数据绑定和访问的整体流程

比如用户需要绑定某MyStruct uniformbuffer，然后access其中roughness field，那么可以写成

```rust
let my_struct_instance: UniformBuffer<MyStruct>;
let roughness = binder.bind(my_struct_instance).roughness().load();
```

其中每一步的类型变化是

```rust
let my_struct_instance: UniformBuffer<MyStruct>;
let my_struct_shader_uniform_ptr: ShaderReaonlyPtrOf<MyStruct> = binder.bind(my_struct_instance);

let my_struct_shader_uniform_roughness_ptr: ShaderReaonlyPtrOf<f32> = my_struct_shader_uniform.roughness();
let roughness_field_data: Node<f32> = = my_struct_shader_uniform_loaded.load();
```

当然，用户也可以直接load一整个struct，然后通过expand来访问子field：

```rust
let my_struct_instance: UniformBuffer<MyStruct>;
let roughness = binder.bind(my_struct_instance).load().expand().roughness;
```

其中每一步的类型变化是

```rust
let my_struct_instance: UniformBuffer<MyStruct>;
let my_struct_shader_uniform: ShaderReaonlyPtrOf<MyStruct> = binder.bind(my_struct_instance);

let my_struct_shader_uniform_loaded: Node<MyStruct> = my_struct_shader_uniform.load();
let my_struct_shader_uniform_loaded_expaned: ENode<MyStruct> = my_struct_shader_uniform_loaded.expand();
let roughness_field_data = my_struct_shader_uniform_loaded_expaned.roughness;
```

## 避免用户表达和维护 binding layout

如果不采用任何框架级别的设计，在整个pipeline的声明创建和使用流程中，需要

- 在创建pipeline时，需要创建 pipeline resource的binding layout
- 在创建bindgroup时 同样需要 这样的binding layout
- 在编写shader时，需要声明这些binding

这三者实际上描述的是完全一件事情，但是仅仅是因为跨语言编程的原因，就需要重复编写三遍。这三者必须始终完全正确对应，在工程和编程体验上是非常糟糕的。在[binding system](binding-management.md) 中，我们的一个核心设计思想就是要让用户完全不考虑binding这一层的存在。所以在shader框架中，我们也不打算让用户感知到binding layout的存在，由此完全避免了上述工程上的维护成本。

最终shader framework和resource binding组合起来，使得表达渲染代码一般都具备这样的结构：

```rust
struct MyRenderer {
  some_render_resource: ResourceTyA,
  other_render_resource: ResourceTyB,
  ..
}

impl MyRenderer {
  fn build_shader(&self, shader_cx: &mut ShaderCtx) {
    let a_resource = shader_cx.bind_by(&self.some_render_resource);
    let b_resource = shader_cx.bind_by(&self.other_render_resource);

    // do some compute and output, for example:
    b_resource.store(a_resource.load())
    shader_cx.write_result_to(...)
  }

  fn bind_pass(&self, pass_cx: &mut PassCx) {
    pass_cx.bind_by(&self.some_render_resource);
    pass_cx.bind_by(&self.other_render_resource);
  }

}

```

其中 build_shader和bind_pass中关于资源绑定的代码是完全对称的。用户只要保证这样的对称性，就可以保证整个binding体系可以正确work。保证这样的正确性相对来说是容易的。如果这样的对称性没有正确保证，会有运行时错误提示哪里出现了mismatch。

更进一步的，理论上我们也能通过宏的方式来避免让用户手工提供这样的match保证，但是这一层的实现比较多变多样，再做如此工程上的建设对于认知成本的角度来说就得不偿失了。从最终的api形态上，我认为这个解法是令人满意的，它非常清晰的说明了它在做什么，资源和pipeline的两种对称的交互被显式的表达了出来，而其他所有的实现细节都被框架负责，其他大部分的正确性都被静态检查，用户只需要保证binding和building的对称性即可。

## 避免跨语言编程的其他工程优势

因为采用edsl相当于采用和host一样的编程语言，所以可以避免的跨语言开发的工程问题：

- 可以充分利用host语言的language server
  - 用户在编辑器中可以直接跳转浏览相关代码。函数调用，常量的引用，控制shader动态编译的变量，资源绑定，以及任何shader value的use也是如此，而采用预处理指令来做import，控制编译，只能搜索字符串，反复的在不同文件间手工的跳转检索。毫无编程体验可言。实际上这一点是工程上和编程体验上绝对的杀手级特性。
- 可以充分利用host语言的语言特性和抽象方式，构造高层次的上层建筑
- 可以充分利用host语言的包管理体系
  - cargo install 一段shader代码，然后直接import use。而不是从别的项目里抠出字符串复制进项目然后拼接进来。

可能有人会说如果我将shader language做的足够好，添加高级的language feature，添加完整的编辑器支持，甚至包管理，然后通过直接生成host接口的host代码来避免跨语言接口重复定义的问题，这么做也能解决工程和体验的问题。但是相比edsl，其本身开发和设计成本是及其巨大的，同时用户的学习成本也double了。采用edsl，用户的学习成本仅仅是framework的学习成本，而不是language和一整套开发环境。

## 顶层框架建设

「利用host语言的语言特性和抽象方式，构造高层次的上层建筑」在实际的工程中是非常powerful的

比如我们可以通过这种方式和rust标准库一样表达迭代器的抽象：

```rust
pub trait ShaderIterator {
  type Item;
  fn shader_next(&self) -> (Node<bool>, Self::Item);
}
```

这样使得一些代码可以以可读性更好的方式编写

```rust
impl<'a> GraphicsShaderProvider for AOComputer<'a> {
  fn build(&self, builder: &mut ShaderRenderPipelineBuilder) {
    builder.fragment(|builder, binding| {
      ...

      let occlusion_sum = samples
        .into_shader_iter()
        .clamp_by(parameter.sample_count)
        .map(|(_, sample): (_, UniformNode<Vec4<f32>>)| {
           ... 
        })
        .sum();

      ...
    })
  }
}

```

这样的例子有很多，比如甚至我们有shader版本的Future来表达gpu上的reactive计算。更上层，材质，光照的一整套抽象和扩展方式，完全也是基于通过Node来交换数据，定义接口的。可以说这套edsl的shader framework是整个图形框架的核心基础，是最重要的工程创新。

## 底层框架建设

shader edsl的底层后端实现也是被抽象的，edsl只是一套api来构造一个抽象的ast。这套api甚至都没有定义ast的具体数据类型，而只是一套抽象的ast 修改和构造的结构。

对于普通的跨平台shader代码生成，现在edsl的后端直接采用了naga。edsl api会直接操作和维护一个naga module，没有任何中间数据结构。用户可以自己为edsl实现其他后端支持。比如

- 支持其他新的shader语言和目标平台
- 做一些平台无关的编译优化
- 做一些transformation，比如自动求导，来实现一些高级功能
- 自动添加一些validation和错误处理，比如访问越界，增强调试能力
- 做直接的解释执行，或者编译到cpu实现。
