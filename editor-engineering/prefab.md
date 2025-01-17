
# 场景prefab支持

## 什么是prefab

prefab 是指 pre-fabrication，这里指一种场景管理上的复用能力

[https://docs.unity3d.com/Manual/Prefabs.html](https://docs.unity3d.com/Manual/Prefabs.html)

业务上，特别是在设计软件中，prefab是非常基础的功能。即便是非参数化的设计软件，也会暴露类似能力使得用户能够复用和结构化内容组织，比如sketch的symbol，figma的component。

## prefab的实现方式

prefab可以通过业务逻辑实现，业务逻辑上可以通过复制节点的方式实现prefab，编写业务逻辑维护prefab 的prototype到prefab clone instance的数据同步。prefab也可以被渲染器场景层直接支持，假设场景层能直接支持prefab，则意味着内存使用可以下降（不需要clone 节点），业务逻辑更新性能会有提升（不需要同步prototype到所有instance的修改），业务实现也会简单。

在一些业务场景，prefab如果由渲染层实现，上述内存和性能的节约是可观的。考虑有这样的场景：用户有柱子的组件，用户组合10个柱子的组件成栏杆的组件，用户在一层楼内组合10个栏杆组件形成围栏组件，用户10层楼每层都有一个围栏组件。如果没有prefab支持，那么不考虑其他优化的情况下，实现场景需要创建10 *10* 10个node节点，而prefab仅需要10+10+10个节点。

场景层直接支持prefab，对场景层实现来说，是非常巨大的改动。基本上所有涉及到场景层的代码都需要修改。我目前经手设计实现的场景都没有prefab支持，本文主要讨论prefab在场景层直接支持的实现方式和细节。

## 在场景层实现prefab

### prefab tree 和 node identification

支持prefab，最重要的是在「世界」角度重新实现node的identification。因为node能在同一prefab的不同实例之间共用，所以单纯给定node的引用，是无法指定在world下到底是哪个绝对的node实例。node的identification定义了场景绝对节点的寻址方式。实现新的identification，可以通过添加一层prefab instance tree来实现，新的id，可以采用prefab instance id + node id实现。整个场景由prefab tree和各个prefab的子场景构成。世界本身也是一个prefab。

prefab id是用户需要感知的，应该直接提供显式prefab节点，以方便用户管理和使用prefab。并给用户prefab tree的查询能力以实现一些用户侧的上层需求。 prefabid需要由真正的“世界”提供，因为只有真正的世界才能将prefabid绝对化。

### derive数据及其更新

场景数据需要逐个节点的对应一份derive数据，derive数据一般由自己的对应原节点和parent的derive节点计算维护。比如worldmatrix就是localmatrix的derive数据，net visible就是visible的derive数据。灵活的场景api一般允许用户自定义节点数据类型，以及derive计算方法。场景层通过一套高级的增量变更传播和标记系统来增量的维护derive数据。

derive数据维护，会利用新的node identification来维护两层 derive trees： prefab tree + 各个prefab的node tree。所以tree更新增量实现并不需要升级成复杂的DAG版本。用户对场景的编辑，增量修改 prefab tree和prefab node tree。prefab tree自顶向下的增量更新，先增量更新 prefab node tree的derive，再用derive的增量change，增量更新 prefab tree的 tree node。最后再根据prefab tree的derive delta+各个prefab原型的derive delta 增量更新真实的derive数据，以实现id pair ⇒ derive的增量维护。

derive数据是否需要clone？这里有tradeoff。如果不需要clone，则要求derive数据的计算方法满足结合率。同时，任何access derive数据都需要on the fly计算：应用节点所在prefab实例的根derive和节点在prefab内的derive。如果需要clone，则以上述栏杆的例子则会有10 *10* 10个derive数据，但是不需要on the fly计算，也不需要derive计算满足结合率。

### 递归prefab和环检测

prefab是支持嵌套的，但是从实现的角度看显然是不能支持递归嵌套的（如果支持递归则场景结构变成了某种生成规则，使得用户可以表达无限的，或者子相似的场景内容）。即api上要支持环检测：prefab使用的子prefab不能以任何直接和间接的方式再引用到自己。

### 其他内部系统的支持

任何之前依赖node id作为 node的寻址的系统，都需要变更为id pair。比如返回picking的结果，场景优化的数据输入，gpu driven shader里对node数据的寻址。node的寻址方式比较fundamental，如果框架上没有抽象过，存在较高改动成本。

相比gpu driven的渲染方式，关于merge和instance的场景优化，和prefab系统是存在一定冲突的的。假设我们将此类优化限制在prefab内部的非prefab节点集合，则对于高度结构化的场景，则优化的效果大打折扣。可考虑将prefab的子prefab递归的flatten到自己再应用此类优化，关键在于定义flatten的规则来平衡内存消耗和drawcall数。

## 高级prefab特性

如果prefab视做生产和自动同步场景的**函数**，那么还可以允许函数具有参数。即可以同一类型的prefab（函数本体），每个不同的实例（函数调用）具有额外的不同属性。而大幅提升prefab的复用能力。一个典型的例子是可配置颜色的椅子prefab，场景中不同颜色的椅子，share同一个prefab。

如果prefab的参数可以采用另一个prefab。即高阶prefab。这种模式类似于react的高阶组件。虽然我还没有思考具体的实现方式，但是一个3d中的应用例子比如有：一个桌子复合体的prefab，可以允许桌面物品prefab实例当作复合体prefab的构造参数。

如果将prefab视作现代ui框架的component的概念，那么对prefab有完善支持的场景，也是一个ui框架的实现。我们前面讨论的各种优化，面向数据，增量，reactive，是否是一个新的视图层框架实现的研究方向，将是一个有趣的问题。
