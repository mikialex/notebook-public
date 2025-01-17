# GPU Raytracing 的上层框架

## 基本概念模型

### 场景 =维护=> raytracing加速结构

我们定义场景的「[基本绘制单元](../../editor-engineering/basic-scene-object-model.md)」为可独立控制材质，几何网格，transform的模型。一般来说，几乎任何场景api都有这样的基本绘制单元的概念。 定义「场景」为该基本绘制单元的集合。我们这里不考虑场景内部对绘制方式有额外假设的其他属性，比如通过图层等显式控制渲染顺序，而简单认为一个场景内，所有的绘制内容是顺序无关的，并且从raytracing的“全局照明”角度是可以相互影响的。

根据这样的设定，tlas实例显然应该由上述「场景」实例一一对应维护。tlas中的build source，显然就一一对应了上述的「基本绘制单元」。build source 引用 blas instance即 绘制单元 引用 mesh instance，build source transfrom配置 即 绘制单元的 transform配置。build source中的customid可采用绘制单元的id，并后续在rtx shader中通过其他scene buffer间接寻址到所需的其他数据引用。

因为“build source 引用 blas instance即 绘制单元 引用 mesh instance”，所以几何网格实例和blas实例一一对应维护。everything looks fine。

### RTX Feature =维护=> RTX pipeline及 SBT

我们定义一个RTX Feature为采用了以一个raytracing pipeline为核心实现的渲染功能。比如rtao，rt-shader，或者完整的pathtracing。

RTX Feature是独立的，独立于场景，独立于不同feature，使得工程上看，一个渲染系统的RTX Feature是可以干净扩展的，比如加载一个新的效果能力。

RTX Feature独立于场景具体是指，如果一个rtx feature对于场景内容或者能力有具体的依赖，那么就需要采用抽象的接口来表达这样的能力，并采用特别的编写场景能力的桥接实现。这样的能力比如获取hit 表面的bxdf参数，获取场景的光照信息。桥接实现中，维护和indirect rendering类似的scene gpu resource，并通过上述接口来暴露必要的feature所需的能力。这样的解耦除了工程上的好处，另外的目的就是希望scene的tlas实例能够在所有feature之间share，以化简维护成本。

RTX Feature独立于不同RTX Feature具体是指，不同的feature之间没有依赖，没有共享的资源。资源维护上，一个rtx featrue需要维护自己的 ShaderBindingTable(SBT) 和 RTX pipeline。维护pipeline是显然的，因为这就是feature表达核心效果实现的方式。同时也需要维护sbt的原因是，pipeline决定了ray stride（一共有几种ray类型）。而ray stride或者traceray的具体使用方式决定了sbt的具体结构设计。

当然，维护sbt仍然要保持和scene的具体实现解耦，解耦的方式就是采用上述的「基本绘制单元」的概念，即场景侧提供一个可观测的绘制单元mapping。场景侧可以以绘制单元的粒度来配置一个“标签”，RTX feature来负责将这个标签的mapping，根据内部traceray的具体实现要求，维护一个sbt实例。所谓的“标签”会转化为一类确定的hitgroup配置（比如使用用哪一个closest hit shader）。

## RTX hit group计算的设计缺陷

我认为上文中我对场景api的基本假设是普遍适用的，且对资源管理和解耦的设计是非常合理的，但是目前spec中对hit group的计算方式并不能很好的支持我上述设计，所以我个人认为hardware rtx spec 在hitgroup计算的标准上有设计问题。。当然不同人对场景渲染的“上层结构”有着不同的理解和习惯，所以可能标准的制定者并不觉得不妥。

根据vkrtx spec，hit group的计算方式为：

`ray_type_index + (max_ray_type_count * geometry_index) + tlas_sbt_offset`

其中 ray_type_index 一般又称之为ray offset，max_ray_type_count 即ray stride。geometry_index为hit到的blas中的第几个geometry（start from zero。tlas_sbt_offset为hit到的tlas内部unit配置的offset。

观察其中的乘法，可看出标准的实现者希望我们在线性的sbt中维护一个二维的结构，其中一个维度由ray type控制，这没什么问题，但是另一个维度是竟然是由geometry index决定的。我认为这是完全错误的设计。

SBT的目的，是为了让我们以某种粒度的去指定场景中不同的部分在哪一种ray下采用哪一种cloestest/anyhit shader，以控制渲染效果。「哪一种ray」这一配置，正确的反应在了hitgroup计算中，但是空间上的「某种粒度」的“粒度”却是以blas中的geometry为准，这非常的不可思议。我认为正确的设计是显然的，显然应该以上述的「绘制基本单元」为粒度，即以tlas的build source为粒度。「绘制基本单元」可以修改材质引用，也正好对应了修改hitgroup配置。

在我们上述的加速结构维护的设计中，完全没有使用到geometry index。任何功能也完全没有必要使用上geoemtry index。因为一个mesh实例内部几乎没有任何合理的需求以至于需要维护一个一对多的子结构（即便是mesh的group也并不需要，因为一个mesh的多个group直接也是完全独立的，即便有数据需要share，也完全不需要通过一个这么底层的api提供支持，并且还直接决定了hitgroup的计算逻辑）

我认为正确的api设计，应该删除geometryindex，hitgroup由 `ray_type_index + (max_ray_type_count * tlas_sbt_index)` 计算。

## hit group设计缺陷的workaround

标准对hitgroup的如此设计，使得为了有效合理的完成sbt的上述目的，就需要workaround：

一种做法是完全不使用tlas，将一个场景的所有数据装进一个blas，将geometry id 映射到「绘制单元」的id。这显然是完全不可接受的。

另一种做法是我现在实践的：

- 所有rtx feature共享一个充分大的ray stride 配置
- scene维护的tlas中的sbtoffset都乘该ray stride

因为geometry_index恒为0，所以hitgroup计算变成：`ray_type_index +  tlas_sbt_offset`，所以每个feature的raystride就完全不生效了， 所以为了重新模拟这样的二维access能力，就需要完全忽略每个feature的ray stride，而采用一个全局的充分大的ray stride 配置，并且将该stride预乘到sbtoffset上。

这需要我们放弃一部分RTX Feature之间完全独立的假设，同时付出一定内存代价（比如假设一个feature只采用了1种ray，但是全局ray stride是6，那么sbt要多耗费6倍存储）。 整体来说这样的代价还是可以接受的

## 顶层frame逻辑接入和hybrid支持

顶层frame逻辑的接入方式很简单，基本就是texture in， texture out。

hybrid 方式的渲染就是texture in 部分rasterization数据，比如gbuffer。rtx feature的实现加入特别的支持即可
