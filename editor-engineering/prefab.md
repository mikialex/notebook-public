# 场景prefab支持

## 什么是prefab

prefab 是指 pre-fabrication，这里指一种场景管理上的复用能力

[https://docs.unity3d.com/Manual/Prefabs.html](https://docs.unity3d.com/Manual/Prefabs.html)

业务上，特别是在设计软件中，prefab是非常基础的功能。即便是非参数化的设计软件，也会暴露类似能力使得用户能够复用和结构化内容组织，比如sketch的symbol，figma的component。

本文主要讨论prefab的实现方式和相关技术细节

API和术语定义：

- 用户可以用某个根结点node来定义prefab的原型：「prefab prototype」，该node的子树的内容就是prefab的内容
- 用户可以实例化某个prefab原型，挂载到某个node上。挂载的实例称之为「prefab ref」

## 以传统方式实现 prefab

prefab可以通过上层业务逻辑实现。即上层业务实现prefab场景到普通场景的增量展开逻辑：复制节点，并维护prefab的prototype到prefab clone instance的数据同步。

传统展开实现，如果用户场景的结构化程度很高，那么进行数据展开存在很高的内存和性能消耗。考虑有这样的场景：用户建模柱子的组件，用户组合10个柱子的组件成栏杆的组件，用户在一层楼内组合10个栏杆组件形成围栏组件，用户10层楼每层都有一个围栏组件。如果采用展开的方式实现prefab，那么需要创建`10 * 10 * 10`个node节点/模型实例，而对于prefab而言，实际定义的数据只需要`10 + 10 + 10`个节点。

这种在业务层展开的方式也可以完全移动到独立的展开系统中，而不由业务的上层视图框架驱动，这么做一般性能会更好。因为脱离于视图层框架，更有机会采用高度优化的并行实现。

## prefab tree

实现prefab的重要的一点是维护一个称之为prefab tree的数据结构。

prefab允许嵌套，即对于某个prefab prototype，其场景上是可能出现多个prefab ref的。对于每个prefab prototype，需要记录其下的所有prefab ref信息，形成parent-children的关系。对于每一个prefab ref，需要递归的根据上述关系，进行prefab实例化。实例化的prefab称之为prefab instance。prefab instance根据上述的父子关系构成一颗树，即prefab tree。

prefab tree可以认为是prefab层面的实例化展开。以传统方式实现的prefab。就是先完成prefab tree的展开，再以此指导完成实际场景数据的展开。

prefab prototype-ref的父子关系，除了记录有哪些prefab ref，还需要根据业务逻辑记录和维护相关的derive数据。

比如prefab ref 到 prefab prototype的transformation，（因为我们限定prefab prototype是根节点，即prefab ref的oworld matrix）。这样的信息同样会被展开到prefab树上，作为这对父子关系local matrix。prefab tree再以此维护prefab instance的world matrix。

这样的derive数据不一定是可以在prefab instance层面被维护的，这需要deriev计算满足结合律。

上述的计算以及prefab tree的维护，一般而言需要支持增量成本的实现。

## prefab的改进实现

对于传统的prefab实现方式，完整展开prefab到scene model的开销很高。

如果prefab的derive数据计算逻辑都满足结合律，那么我们就不用依赖完整的展开来计算scene model level的derive数据。所以这种情况下，就完全不需要展开prefab。在计算/渲染时，我们同时访问prefab instance + scene model，在访问时完成derive数据的计算。以付出访问开销的代价来换取不展开prefab到model级别的内存和同步成本。

在计算/渲染时同时访问prefab instance id + scene model id 并不意味着我们需要将这样的id pair展开到model level。（这样相当于还是需要在scene model层面做展开）。而是可以直接traverse prefab tree来得到这样的pair信息。

支持indirect渲染，即需要在device上per thread的access prefab instance id + scene model id，这种情况下也并不意味着我们需要将这样的id pair展开到model level。假设我们维护每个prefab prototype的model count，以及`[self_models, first_child_ref_models, second_child_ref_models, ...]`的model count prefix sum，并且我们要求perfab prototype下的所有model 连续线性的存储。那么我们就可以根据root 的model count dispatch compute， per invocation通过多层的二分查找来得到当前invocation的prefab instance id，然后通过访问model的线性存储来得到scene model id。多层的二分查找，每一层相当于在prefab tree上找到下一层的child instance（或者当前instance）。prefab场景嵌套深度越高，overhead也越高（O(n)）,prefab的同级children越多，overhead也越高（O(logn)），但prefab tree的深度嵌套实际是少见的。这样的id pair access也可以直接展开到storage buffer，以避免反复access的overhead。

还有一种indirect渲染的潜在支持方式是采用dymanic parallelism。这里暂时不展开讨论

## 其他问题

### model identification

支持prefab，需要在世界角度重新实现model的identification。

identification指：给定model的某种id信息就可以唯一的确定model的实例。用户可感知的模型是prefab完全展开后的实例，即交互都是针对展开后的实例进行。因为model的node能在同一prefab的不同实例之间共用，所以在支持prefab后，单纯的给定model的handle，是无法确定在全局视角下实际上是指哪个绝对的model实例。实现这样的id才能支持返回有意义的picking结果。有意义的picking结果才能实现场景的编辑交互。

可以通过prefab instance id + model id实现这样的 identification，用户可以通过查询prefab tree来获取更多信息，比如整个prefab instance parent chain

### 递归prefab和环检测

prefab是支持嵌套的，但是从实现的角度看显然是不能支持递归嵌套的（如果支持递归则场景结构变成了某种生成规则，使得用户可以表达无限的，或者子相似的场景内容）。即api上要支持环检测：prefab使用的子prefab不能以任何直接和间接的方式再引用到自己。

## prefab和视图层框架的联系

如果prefab视做生产和自动同步场景的**函数**，那么还可以允许函数具有参数。即可以同一类型的prefab（函数本体），每个不同的实例（函数调用）具有额外的不同属性。而大幅提升prefab的复用能力。一个典型的例子是可配置颜色的椅子prefab，场景中不同颜色的椅子，share同一个prefab。

如果prefab的参数可以采用另一个prefab。即高阶prefab。这种模式类似于react的高阶组件。一个3d中的应用例子比如有：一个桌子复合体的prefab，可以允许桌面物品prefab实例当作复合体prefab的构造参数。

这种更加通用的prefab支持，其实类似于现代ui框架的核心问题。prefab即ui component。抽象的概括：

- prefab prototype => function
- prefab ref => function call
- prefab tree => function call instance tree
- 支持高阶prefab => 支持函数作为函数参数

在如此general的case做优化在工程上是没有意义（不可能）的，可能还是需要回归到传统做法。传统做法总之也是两个流派：

- retain mode gui：展开并增量维护更低级的view视图
- immediate mode gui：全量运行时acsess当场计算，不维护任何低级视图
