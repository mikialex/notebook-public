# 场景基本单元对象

此文主要讨论典型的三维场景API设计和其中的「基本单元对象」的概念。

因为场景本身的结构几乎是所有的上层建筑的基础。所以在考虑此基本的问题就需要谨慎思考决定。更进一步的，这样的思考并不应该建立与我们使用其他图形库的先入为主的经验（当然也不应是完全闭门造车），这里我们试图从原理层面推导出一些设计的必要性。

## 概念模型：场景编辑的层次性

对于一个可编辑场景，其编辑的粒度并不是灵活可变的，而是一个根据编辑需求构成的层次编辑模型。

比如在场景中，移动一个物体，和移动一个物体上的一个点来改变物体形状显然是两种粒度。设置一个物体的材质颜色，和设置物体材质上贴图的一点颜色，显然也是两种粒度。一「组」物体，有可能在编辑层面，其行为交互完全是一个物体，在更加粗的粒度进行编辑，而用户可能可以进入组的内部，编辑组的内部。

编辑行为本身就是以不同粒度存在的，以细粒度的方式难以编辑大量内容，以粗粒度的的方式无法编辑细节内容。

层次性的另一个存在理由是场景实现上的限制：一个场景的实现系统，无论是渲染还是状态维护，在维持基本的使用体验的情况下。其编辑对象的绝对数量是有限制的。比如受到内存的限制，更新性能的限制，drawcall数量的限制。即 实现上，如果场景的编辑需求非常自由多样，难以做到让用户同时编辑任意细粒度的场景层次。只能提供尽可能粗躁的编辑粒度，而细节的编辑粒度需要根据用户的编辑动作按需展开（当然不排除有非常高级的编辑体系，使得编辑粒度是虚拟化的，来尽可能避免层次切换的成本）。

因为用户有了层次性的假设，就可以自然的接受不同层次之间切换的成本。于用户而言，这样的成本是交互上的，比如切换编辑层次需要额外操作，甚至需要等待。所以这个层次性的概念模型有利于支持用户理解实现上的困难。

## 概念模型： 场景数据的复用

用户复用场景的部分内容是常见通俗需求，比如两个物体始终要保持形状一样，那么就应该复用其形状部分的数据。这样的复用能力是多样的，这种通用的复用能力的实现方式，使得场景数据成为一个复杂的动态的图结构，而不再是平坦的。

支持这样动态的复杂图结构，对于实现而言引入了新的困难。以至于引入一些限制是必要的。

完全的动态性是没有必要也无法支持的。因为讨论对象模型的目的就是要提供一套api，所谓api就是一套基本静态的协议，来解释用户应该构造出什么样合理的数据，以及实现方如何解释和处理这样合理的数据。支持复用，在api设计上要做的事情就是识别出场景中最符合需求的数据复用假设，固化出一套相对静态的引用关系模型。

## 基本单元对象

根据上文，场景的实现中，编辑对象的绝对数量是有限制的。更进一步的，能够以合理的性能开销进行编辑的「合理编辑对象」绝对数量是有限制的。另一种说法可以是，为了让绝对编辑对象的数量上限提升，基本的编辑动作需要尽可能的低成本。

所以我们需要一个基本的对象来对应这样的「合理编辑对象」，目的是用户可以通过控制该对象的数量在实现层面来控制和推理绝对性能表现，比如在此之上建设层次结构的编辑模型（基本对象一般是可以merge和split的）。

另根据上文，场景数据需要支持复用，一个自然的想法就是将引用关系平坦化到这样的基本对象上，用一个概念去承担多个正交的职责，用这样的基本对象来聚合场景资源数据的引用关系。

这个想法，和ECS架构有相似之处，这样的基本对象被称之为entity。但是是否应该采用纯粹的ECS？参考我之前对ecs的看法，我认为并不需要。我个人认为应该以关系型数据来建模场景，只不过这样的可编辑单元是最重要的entity（table）

## 基本单元对象有利于解耦

这样的基本单元，和任何一种entity一样，在实现上肯定是需要处理成全局线性寻址，通过一个stable的，linerlization后的id来指代。

确立这样的基本单元对象，另一个深层次的原因是为了解耦实现，从实现层面，也支持了明确这样的基本单元的必要性。

外部实现，可以对其要支持的抽象场景有这样的线性寻址的基本单元假设，使得其实现对具体的场景实现解耦。从渲染的角度看，这样的解耦体现在：

比如一个典型的two pass gpu driven的遮挡剔除系统，可以完全基于一个抽象的场景来实现。场景端的能力可以抽象的认为是将某上述id的内容绘制到某目标上。系统内最核心的比如要跨帧维护的gpu上的遮挡状态，也是通过这样的id来寻址读写，以及通过这样的抽象id来做遮挡测试。

比如一个典型的gpu picking实现，需要绘制一个object id buffer，此object id自然是可采用这样的id。

比如更加基础的，我们如何去抽象一整个scene renderer， 如何去抽象indirect和direct（gles）模式的绘制。如果这样的id，在图形侧能够体现direct 模式下draw的基本粒度，那么自然就可以通过这样的id来抽象场景的绘制接口。除了绘制，统一gpu和cpu上做场景数据批量处理的逻辑，比如实现上去统一gpu driven的过滤，排序，都是依赖这样的主entity存在。

## Rendiation的基本场景模型

在Rendiation的场景模型中，这样的基本单元对象称之为SceneModel。

- 引用Node，以决定Transformation。
- 可能引用Model，以决定传统模型式对象的内容。
- 可能引用其他entity，比如PointCloud，以决定其他特别的场景内容

对于传统模型式对象，Model：

- 引用不同类型的Material
- 引用不同类型的Mesh

Material Mesh可能引用不同类型的Buffer，Texture，或者自定义的Component

Rendiation的场景设计参考了常见的3d api的场景设计，比如Threejs 和 gltf。

Scenemodel在性能上主要假设了其对应一个gles 模式渲染的drawcall。另外其对应了主要的引用关系集合。从这个角度看，gltf中的(primitive)model比threejs的Mesh(mesh)要合理，因为它可以更好的对应一个传统drawcall（毕竟是为rendering设计的格式）。

采用额外一层Model的原因是扩展性，因为简单的假设所有的基本单元都是传统模型是不合适的。但是要求所有的编辑单元都可以被Node控制transformation是合理的。
