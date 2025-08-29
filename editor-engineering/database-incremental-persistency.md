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

- 在db scope内创建的entity，必须在db scope内销毁
- db scope内创建的entity只能reference（外键指向） db scope内创建的entity
  - db scope外创建的entity可以reference db scope内创建的entity

scope api需要提供debug validation工具来检查上述db scope的隔离性

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

### 应用层需要确保 editing 修改是独占的

不同的edit featur同时运行会导致的不同的edit change汇总在一个checkpoint里，这显然不是合理的行为。应用层需要保证 editing对场景的修改是独占的。可以引入某种edit权限的锁机制来保证只有一个逻辑上的active editing logic。

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
