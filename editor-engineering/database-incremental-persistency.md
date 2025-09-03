# 基于database的持久化和undo-redo实现

## 只有一种保存（持久化）模式

根本不应该有全量保存这个概念，只有增量保存。**编辑数据的持久化必须是增量实现**

以下做法都是错误的：

- 实现中有「序列化整个文档/场景数据到文件」的所谓“全量保存”的做法。
- 设计了两种不同的文件格式，分别作为增量和全量的实现。

增量保存就是记录历史，最终编辑结果就是用编辑历史来表达的。历史编辑的反复只会造成最终编辑结果表达的效率降低，不会影响正确性。所以从所谓增量到“全量”的转换，只是编辑历史的压缩优化。“全量”保存只是增量保存的特例：是只包含一个transaction的增量保存。

所以没有必要设计全量的持久化格式，只需要设计增量历史的存储格式，和transaction合并的实现即可。是否合并，何时合并，只是存储消耗和重建性能，对保存性能的取舍。

从上层产品的交互，从交互的角度看，我们需要什么样的持久化行为？

- 任何编辑变更，都应该自动的保存。用户不用担心的频繁显式触发保存动作，来确保不会因为应用崩溃而导致编辑进展丢失。整个持久化行为应该是毫无存在感的。
- 显式保存只有一个触发场景：在关闭应用时，用户来确保任何在途的持久化工作都已正确完成。以确保所有的编辑都完成了同步（因为有些平台比如web，页面关闭是立即的，没有机会做更多事情）
- 保存的性能/消耗应该正比于修改量，而不是场景/文档本身的规模
- 持久化可以以磁盘为目标，也可以以网络存储为目标。用户以项目为单位进行工作，自动的在不同的存储目标上同步修改。无论是local first还是纯云端的工作流程都可以完美支持

从我们追求的上层用户体验，来反推技术上的底层实现：任何持久化实现都必须是增量的。

简略起见，下文的持久化和保存都是指增量的实现

## undo-redo和持久化的关系

undo-redo和持久化，在行为和实现上都有重要的相关性。

### checkpoint 和编辑的 transaction

工具的编辑，一般会对场景的多个数据进行修改。这些修改是不可分割的，只有被一起应用，才是完整的。否则会导致数据模型进入对于上层应用来说的非法状态。我们定义一系列不可分割的数据修改，为编辑的transcation。

根据工程上唯一的[合理做法](../data-modeling-and-computation/data-modeling/database-is-fundamental.md), undo-redo,和持久化需要实现在底部的数据模型db层，但是编辑的transaction是上层的业务信息。所以我们需要db层能够提供这样的支持：以某种方式，通知db，目前的数据修改，构成一个edit transaction，或者说 **目前的change进入了一个对于上层业务来说有意义的整体状态**。

这样的通知机制，我们称之为checkpoint事件。checkpoint事件：

- 对于undoredo，决定了undo redo的target state，group出state变更的数据变更
- 对于持久化，group出需要保证原子性的数据变更

### 交互式编辑的问题

抽象的工具编辑流程是：

- 根据场景的状态（一般是选择集的状态）来激活某些工具的可用性
- 用户选择一个被激活的工具，开始编辑，有两种编辑方式
  - 对于重计算的编辑，设定编辑参数，触发计算，修改场景数据，完成编辑
  - 对于轻计算的编辑，为了更好的用户体验，一般是通过连续的交互（比如拖拽），来实时的在每一帧都设定编辑参数，触发计算，修改场景数据，使得用户能够所见即所得的看到编辑的效果，最后决定使用哪一个编辑参数。

这种所见即所得的交互式编辑

- 对于持久化，会产生大量被覆盖的临时修改，浪费存储并产生不必要的计算消耗。
- 对于undo redo，其中间的临时编辑，是用户舍弃的编辑，并不应该作为undo的targe。

因为交互式编辑是上层的行为，所以解决其带来的问题，只需要上层应用改进checkpoint api的集成实现。即对于需要丢弃中间结果的交互式编辑，不要触发checkpoint即可。

### DB scope

持久化和undoredo，都需要记录change。这样的change记录，不应该以全局的entity来。而是用户关心的entity set。这样的scope based watch一般是必要的，因为

- 用以支持编辑的数据，一般是和用户的场景数据是存储在一个db的，因为某些系统是以全局为单位进行处理的（比如希望利用到全局线性id的性质以简化实现和改进访问性能（比如gpu方面的系统））。比如3d编辑器中的gizmo，light-helper等场景物体，这些物体的变更不应该被undo redo和持久化
- 用户很可能会在一个db内同时维持两个编辑ctx，比如同时打开和编辑两个项目，希望他们具备隔离的持久化和undoredo行为。

#### db scope的隔离性

因为要保证scope change中的外键（引用）信息一定是合法的。scope内的change是self contain的。所以db scope内的场景修改需要应用层保证这样的隔离性：

- 在db scope内创建的entity，必须在db scope内销毁和修改 (A)
- db scope内创建的entity只能reference（外键指向） db scope内创建的entity
  - db scope外创建的entity可以reference db scope内创建的entity

scope api需要提供debug validation工具来检查上述db scope的隔离性。

(A): 对于采用hooks api来说，这样的隔离保证是容易实现的。因为其资源/数据有明确的scope的范围，只要保证该范围在db scope内即可。

#### db scope 的其他作用

db scope可以作为某些场景片段的dropper，方便资源管理。

比如有个gltf loader，简单实现是把数据加载到scene里生成一些新的entity，要支持卸载就需要记录哪些entity是来自于这此加载，那么加载的时候套一个scope，就有这个信息了。这种feature对于loader来说是常见的，所以可以复用db scope的能力，就不用给每一种loader都添加“这个loader load了哪些entity方便后面drop”这种实现

#### 持久化可能和undo redo共用同一个db scope

从产品的角度看，持久化的 db scope 和undoredo的db scope一般是相同的。而db scope本身有维护和内存成本，所以持久化和undo-redo要在实现层面上能够复用一个db scope

### 核心实现

根据上述的推理，我们认为一个被undoredo和持久化共享的底层实现是必要的，这个底层实现是：一个scope的api，用户在每个scope内的场景编辑，能够被分别记录，在scope内用户可以触发checkpoint事件，来flush目前记录的db数据变更。该数据变更集：

- 对于持久化，会异步序列化并持久化到外部目标上
- 对于undoredo。
  - 实现undo stack和redo stack。stack item内容就是该数据变更集。undo就是revert undo stack top item到redo stack。 redo就是revert redo stack top item到undo stack，任何新的checkpoint事件会flush change并push新的undo stack同时清空redo stack。

## entity handle 的持久化映射

在db中，entity通过 raw_entity_handle 来寻址，entity handle由`(allocation_index, allocation_generation)` 构成。raw_entity_handle实现了entity的在**当前db内存实例的历史**中的identification(uniqueness)，同时支持random access。

持久化id对identification有更高的要求，其要求应该至少是**当前project实例的历史**，这和**当前db内存实例的历史**完全不一样。所以我们需要一种新的id，来在持久化的序列化和反序列化实现中mapping 持久化的entity handle 到实际db的entity的entity handle(raw_entity_handle)。分析以下几种选择：

- 沿用raw_entity_handle作为持久化的entity handle
  - 如果只支持reload是可以的，但是reload之后就无法编辑，因为raw_entity_handle的生成依赖于db的执行实例的内部状态。而上一次db已经完全不存在了。
- 采用uuid，完全彻底的unique id。
  - 这一般是持久化/分布式系统的默认做法。是可行的。因为其提供了最强的保证，保证了任何历史的uniqueness。
    - 因为这样的保证，所以我们几乎甚至可以通过直接连接两个持久化文件来完成两个场景的合并，因为其中的enitty id不会有冲突
  - 预计生成和存储的开销较高，如果对持久化系统有debug需求，或者tracing debug需求，编辑，复制，查看冗长的id比较麻烦。
- 采用一个per project的计数器来作为id 生成器。
  - 能够避免上述uuid的问题，但是无法实现快速项目/场景合并（等于一个新的project，所以需要重新remapping）
  - current id并不需要持久化，因为加载时会重放change，重放change会自动计算出当前最新的自增id

id mapping转换的实现细节：

- id 转化是有成本的工作，应该在持久化的背景线程中完成。
- id转化也要处理外键的change。~~如果某些component虽然不是外键类型，但是实际上有entity引用信息，也需要处理转换工作（比如node的parent）。~~ 因为component中保证不包含外键类型，所以可以忽略component的转换。

## Hydration

Hydration这个问题比较抽象，我不确定有什么更好的名字，采用这个名字是因为这个问题和React的hydration是相似。

### 一个具体的例子

假设我要持久化一个scene以及下面若干scene model，并且出于方便起见我使用代码自动的创建这个scene，只让用户来在上面attach scene model。

根据上文scope的隔离性要求（db scope内创建的entity只能reference db scope内创建的entity），那么这些scene model的scene reference必须指向scope内创建的scene，所以scene必须在scope内创建。所以相当于这个scope内有个内存中一直存在的scene变量，后面创建的scenemodel都attach这个scene。

在首次运行时，因为之前没有任何持久化结果，那么就应该从db内直接创建新的scene。而在后续的重新打开应用的情况下，这个scene entity会从之前持久化结果内自动的复活在db内。那么这时候，就应该采用这个复活的scene。

那么问题是：我怎么找到这个复活的scene的handle是哪一个？如果我之前创建了多个scene，重新加载后读回的可能是多个handle，那么哪个重建的handle是哪个代码内之前create的scene？

### 可能的解法

（注： 这里的做法的可行性和内容有待原型实现结果予以补充）

如果被持久化的db数据，其中的一些entity，被运行时的变量引用，那么需要保证每次程序加载，这个变量都引用的是一个entity数据。持久化结果重新加载后，将这样的变量，重新正确绑定到由持久化重建的handle的过程，称之为 Hydration。

如果没有这样的程序变量，那么就不需要这个过程。如果避免这个需求，那么可以不用考虑这样的复杂度。但是一般来说，比如上述的scene的case，表达数据的挂载点还是常见的，所以必要的支持是需要考虑的。

实现hydration，需要持久化时一同保存变量的identity对handle的mapping。hydration时查询此信息来完成重建。这个mapping和db数据一样，采用增量的方式进行持久化。

变量的identity简单处理可以通过设置label来进行标记。更加完善的做法可和hooks一样采用scoping的track_caller的callsite信息。但这种做法可能不适合，因为持久化的文件是要长期存储考虑兼容性的，那么call site是非常不稳定的，采用callsite，虽然简化了记录变量id的做法，但是无法实现长期的格式兼容性。

react的 hydration没有这样单独的mapping信息，是因为dom本身就是和state tree匹配的结构化的数据，不仅包含了普通数据，还隐含了这样的mapping。所以可以直接通过tree diff的方式来找到映射。而db的数据缺少这样的业务逻辑的state tree结构，所以需要存储和使用必要的额外信息。

## 其他细节问题

### 持久化和undoredo只应用于纯业务的数据

以camera为例，为了改进3d编辑体验，用户的相机操作和视角切换可能是具有动画平滑效果。实现这样的效果，会持续的修改相机参数，而这么做会导致破坏整个持久化和undoredo的行为，因为相机参数的连续变化总是被interleave进其他场景编辑中。

解决这个问题，我们需要分离用于动画渲染的camera和用于持久化的业务camera，动画的camera采用现有的连续交互逻辑，同时在必要时同步到用于持久化的camera触发checkpoint。

同理，支持 animation system，比如gltf的animation，具体来说比如参数驱动的node transform动画，可以添加一个新的render node component，用以综合origin node component和来自aniamtion系统的override。这个render node component取代了origin node component来作为实际的渲染input，但是并不会执行持久化。

database作为通用的数据容器，其中并不一定要只存储渲染相关的场景数据，也可以直接存储较为上层的业务数据，渲染的场景数据甚至可能只是上层业务数据的derive，总之原则就是分离动态交互的次生数据和真正的原始业务数据，只对原始业务数据做持久化来避免动态交互的影响。

### 两种不同的change记录方法

比如某个工具，其作用总是将x的坐标position+2， 假设position原本位于1，那么这个工具运行后，position变为3.

- 记录new value，可以将change，记录为x's position change to 3 （现有逻辑）
- 记录value delta，也可将change，记录为x's position increase 2

哪一种记录change的做法更好？

我认为记录new value是正确的做法，因为

记录value delta意味着**试图在mutation中记录初值无关的mutation算法**，对于上述position来说，这个算法就是加减法，但是对于通用的情况，其算法本身可以是任意的，而通用实现是必要的，因为加减法是trivral的。这么做会引入非常多的复杂度和可能性。mutation如果encode的不是change结果，而是实现change的算法，意味着序列化格式其实得是executable的，其实就是一种script。记录new value的做法其实只是记录value delta的特例，因为相当于mutation的算法是设置value的绝对值，不包含实际的计算，即实际上没有任何算法被显式或者隐式的被encode。

这两者的区别就是我们是否允许（需要）记录mutation的算法，而不是记录mutation本身？

采用delta value的优势是，delta value记录的是change的transformation本身的信息，可以避免依赖初始状态， 即相对值和绝对值的区别。使得这些change可以被回放到任何初始状态的对象上。

我认为从tools/editing的角度看，有三个问题需要考虑。

- 用户是否需要从editing tools来生成自定义宏/脚本的功能？
- 用户是否总是希望生成的宏/脚本能够使用在多种input下？
  - 我可以还是只记录绝对change，并且生成宏让用户使用（只处理绝对的批量化的工作）
  - 我可以还是只记录绝对change，生成绝对change的脚本，让用户修改脚本来支持相对的case
- 现有的editing tools是否大多数能够生成有意义的mutation的算法，而不是mutation本身？
  - 即现有的工具能够生成相对change（change算法）的脚本/宏

我一般认为同时满足，特别是第三点，是非常少见的。同时，要支持这样的能力，其开发成本是极其高昂的，所以记录value delta/算法是不合理的做法。

补充：对于undoredo来说，value delta的算法还需要是可逆的，如果某些算法做不到或者无法以合理的成本做到这一点，那么就只能记录算法output的绝对change。

### 业务数据本身可以是“可编程”的

一个场景是，假设我有个图片，需要做高斯模糊，那么持久化和undo redo应该怎么做？

一个想法是，假设有上文的delta value的change 记录，那么我们可以将高斯模糊及其参数打包成一个change的desc。这样我根本不用记录change后的图。且不论delta value这个做法本身不应该实现的问题，支持undo该怎么做？用一个反卷积的计算来表达吗？

对于这种情况，应该认为原始图片和高斯模糊运算都是场景的业务数据，而模糊的结果是场景业务数据的derive数据，就和渲染的结果是一个性质。这样是否开启高斯模糊就变成添加/删除一个effect entity的问题。

业务数据本身可以是一个可编程的结构，比如一个2d的场景上配置各种嵌套的effect。应该对「计算表达式」本身进行持久化/undoredo，而不是计算过程或者计算结果来做。

### undo redo的stack本身需要持久化吗？

一般是不需要的，除非

- editor做的非常完善，需要实现「即便崩溃了重新进来还要能undoredo」这种需求
- 支持超长undo历史，或者某些change本身非常消耗内存（比如mesh/位图编辑）那么交换到磁盘是必要的。

这种属于工程上的长期工作。

即便undoredo需要持久化，其持久化也不应该复用场景本身的持久化。因为场景本身的持久化会渐近的持续的执行压缩优化，而undo redo的transaction信息是要独立保留的。

### debug tracing

debug tracing是指对db的数据变更进行录制回放，用以调试和回归应用的逻辑和性能问题，是非常重要的测试基础设施。tracing可以直接采用持久化来实现，具有和持久化一样的业务性质（比如scoping）。区别是要关闭编辑历史异步压缩（显然压缩就直接删除了编辑的过程）。如果追求db修改的完整过程，那么transcation内的change合并应该也同时关闭。

### 持久化的实现细节要求

- 该功能不应该明显影响到应用的编辑性能和内存消耗
  - 写磁盘需要异步在其他线程进行，并尽早的完成写出并释放内存
- 因为持久化是异步的，可以令触发checkpoint事件的API返回是否实际完成持久化的futures，以便上层业务得知该信息。
  - 文件系统api是没有这样的功能的，估计实现上需要通过定期显式flush实现
- 因为应用可能会在任何时刻崩溃，所以实际文件写入可能中断在任何一个进度（有待确定是否能放松这样的假设），所以增量数据的反序列化实现及其格式选择要考虑到支持不完整的文件内容
- 持久化的文件也不应该持续无限增长，需要异步的在后台合并压缩
- 存储层应该采用抽象实现，以后续支持持久化到任何一个存储后端
  - 比如fs，比如任何协议的网络存储，这方面可能 [opendal](https://opendal.apache.org/) 是一个不错的选择

### 非结构化数据的持久化和undo redo支持

db内存储的倾向于是一种结构化的数据：具有多种entity，用外键来复用数据和表达关联，比如表达场景，和某种设计文档。

有些“非结构化”的数据并不适合在db中直接存储，比如编辑器场景的文字，mesh， 点云，位图等。这里所谓非结构化并不是说它内部完全不具备结构，比如富文本和mesh内部结构是极丰富的，但是他们并不适合在现有的db模型上进行建模。

不适合存储的原因，既有实现的问题也有设计上的问题

- per entity db都会添加generation标记并访问时check，如果point cloud的point都是entity，那么显然开销是无法忽视的
  - 实现上我们可以配置某类entity不采用generation check。牺牲一些逻辑上的安全性。
- db需要支持嵌套表，或者支持range的insert/delete。即：批量的插入和删除entity，但是保证他们的allocation(entity id)是连续的
  - 还需要考虑好用的上层api，比如封装好的range entity的概念，甚至2d range，或者单独为某种业务结构服务的api
  - range的insert/delete方案其实并不好，因为比如让不同点云entity的点enitty保证在一个地址空间意义不是很大
  - 支持嵌套表会有很多设计上的不明确的地方，需要做很多的设计和实现来解决小部分的问题
  - 如果今天支持嵌套表，但是明天有一个需求，使得嵌套「表」是不够的，需要嵌套其他类型的数据结构，那么db是否本身应该允许嵌套复合结构本身就是一个疑问

这类数据在db中，应该通过url/uri的方式进行存储。这个问题的类似版本在[另一文](../renderer-engineering/dispose-host-data.md)中有所讨论

采用uri可以认为是一种表外存储，对于db来说，其需要满足的性质有

- 可以异步的从某个外部系统获取实际的数据，实际数据的存储，形态结构，都是抽象的
  - 对于db来说，没有额外的任何存储消耗
  - 对于db来说，完全是不透明的，不关心内部实现
- 保证immutable，相同的uri，在任何时候access的数据都是相同的
  - 这样db的watch系统（以及所有基于watch的feature，包括持久化和undoredo）可以直接work，不用考虑内部可变性
    - 表外存储需要自行处理内部的持久化和undoredo，这种数据的表外实现，可能具有独立的编辑环境，在checkpoint事件时会生成新的uri来表达当前版本。

总之，表外数据用url，实现封闭，url change就是内容change，实际数据能异步access

#### 以一个mesh编辑器（或者广义的处理mesh数据的表外系统）为例

原3d场景的mesh数据，不在表内存储实际buffer信息，而是采用某个uri（uuid/path/url）。

在3d渲染创建gpu mesh资源或者picking时，按需的从mesh系统，用uri access实际数据。access是异步的，因为数据不一定在内存内。caller可以决定block_on，也可以决定做降级处理。比如降级显示和pick精度到bbox。

考虑编辑场景：mesh的编辑应该分为两种，精细编辑和整体编辑。

- 精细编辑指的是手工编辑mesh细节
- 批量编辑指的是简化和remesh，细分这种算法性质的整体改造

这两种编辑模式是需要区分的，或者说批量编辑应该被独立处理，因为批量编辑相当于全量change，这种情况下，存储增量信息是非常浪费的，反而应该存储算法本身的change（参考上文高斯模糊的例子）。这种编辑模式也符合一般常用的dcc软件工作流：用户细节编辑和调整一个低模，然后通过开关细分预览来查看效果

对于精细编辑：

- mesh系统激活某个mesh uri进入精细编辑模式
- 用户选择具体的mesh edit tool，比如翻转边，切割边，补洞等进行编辑
  - tools记录精细的增量变更信息，交给mesh系统作为transaction，mesh系统生成一个uri来代表这个编辑结果
  - 新的uri被设置在db上
  - 所有下游系统自动的更新和使用新的mesh

对于批量编辑：

mesh系统实现一个组合mesh算法的数据结构，比如堆栈：用户可以组装mesh的处理算法，来表达参数化的编辑。这类编辑的change就是算法结构的change。更加自由一点的可以采用一个计算图，或者blender的geometry node架构。

mesh系统对于某个手工编辑的base mesh的uri，组合当前算法堆栈的信息，形成新的uri。

某个算法的应用结果，可以bake回到base mesh，以支持用户来在算法的结果上做手工编辑。

这个bake 操作需要全量的mesh数据作为变更集，因为算法很可能是destrcutive的

除了mesh 编辑器，点云，位图的编辑器也具有类似的特征，所以可以采用相同的思想来解决，主要是要分离手工和算法编辑，使得两种编辑都能够有效的处理增量变更。我们也可以看到，这两种编辑变更信息虽然都是增量的，但是是截然不同的类型。
