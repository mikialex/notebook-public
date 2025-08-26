# 编辑器场景实现基于database的增量保存, debug tracing 和 undo-redo

## 实现路径

实现tracing replay

- 记录所有entity的所有change，异步写到磁盘
- 主要是用来重放所有的db change，以调试和回归任何数据变更相关的逻辑和性能问题

实现 undo redo 和增量保存前置需求

- 实现entity scope的api，用来让用户标记哪些entity是需要在一起处理（增量）持久化和undoredo的。
  - 对于undoredo是必要的，因为场景中的gizmo/helper等entity显然是不应被undoredo的。
  - 对于增量保存是必要的，同理因为场景中的gizmo/helper等entity显然是不应被持久化的。
  - 这样的scope可以有多个，同一个entity只能属于一个或零个scope
- 需要添加checkpoint的API，让用户标记什么状态可以认为是undo的target。
  - 对于undoredo是必要的，因为很多连续的交互导致的场景变更过程，并不应该作为undo的 target，比如dragging一个物体。
  - 对于增量保存是应该的，因为记录这种不必要的中间change会导致不必要的性能消耗

实现增量保存

- 在entity scope的层面实现watch，而不是全局entity层面。
- 上述watch支持staging 能力，能buffer change，直到另外的事件来触发change flush。
- 改进tracing实现，接入上述实现，使得在checkpoint事件触发时才持久化scope内的change

实现undo redo

提取上述watch scope， stage change， flush event的实现，基于此另外实现undo stack和redo stack。stack item内容就是flushed changes。undo就是revert undo stack top item到redo stack。 redo就是revert redo stack top item到undo stack，任何新的checkpoint事件会flush change并push新的undo stack同时清空redo stack。

用户在编辑层实现完成上述api（scope和checkpoint设置）的集成工作，编辑器就自动具备了undo redo和增量保存的能力。

## 细节问题

### 如何避免异步的场景修改导致的问题？

比如有一个异步的场景加载task正在运行，而我正在移动场景中的一个物体，那么undo可能会导致revert场景加载。

这种情况应该通过支持多个watch/checkpoint scope来实现。多个watch/checkpoint scope的支持其实是必要的。这样用户可以在不同的edit context下，具备不同的相互隔离的undoredo stack。

更加一般的情况依赖用户来保证，用户需要避免在任何异步的场景修改任务完成之前设置checkpoint，这本身就是逻辑错误。

### 如何避免不同的edit feature导致的change在一个checkpoint里相互interleave？

这种情况依赖用户保证，属于逻辑问题。可能应该引入某种edit权限的锁机制来保证只有一个逻辑上的active editing logic。

checkpoint可以添加一些debug的信息，但是checkpoint本身不具备edit的semantic，不和任何一个特定的edit feature有关系。**checkpoint甚至和增量持久化和undoredo都没有关系，只是一个事件，来通知目前某个scope累计的change进入了一个对于上层业务来说有意义的整体状态。**

### 增量保存是否应该和undo redo共用同一个change scope（share同一个buffered change）？

如果持久化的entity scope 和undoredo的entity scope完全重合，那么是应该shared。

### undo redo的stack本身需要持久化吗？

一般是不需要的，除非

- editor做的非常完善，需要实现「即便崩溃了重新进来还要能undoredo」这种需求
- 支持超长undo历史，或者某些change本身非常消耗内存（比如mesh/位图编辑）那么交换到磁盘是必要的。

这种属于工程上的长期工作。

即便undoredo需要持久化，其持久化也不应该和数据本身的增量持久化复用同一个数据。因为增量持久化会渐近的持续转化为全量持久化。

### 增量持久化的细节

增量保存主要是：

- 为了防止应用崩溃造成编辑进度丢失
- 大幅提升应用的保存性能（因为基本就不需要全量保存了）

其实现上要具备这些考量

- 该功能不应该明显影响到应用的编辑性能和内存消耗
  - 写磁盘需要异步在其他线程进行，并尽早的完成写出并释放内存
- 因为持久化是异步的，可能需要提供上层的api，比如fence来在特殊情况下指示实际的持久化是否完成
  - 文件系统api是没有这样的功能的，估计实现上需要通过定期显式flush实现
- 因为应用可能会在任何时刻崩溃，所以实际文件写入可能中断在任何一个进度（有待确定是否能放松这样的假设），所以增量数据的反序列化实现及其格式选择要考虑到支持不完整的文件内容
- 增量持久化是总是基于一个全量持久化的结果进行的
- 增量持久化的文件也不应该持续无限增长，需要异步的后台定期合并/积分到全量持久化的结果中
- 存储层应该采用抽象实现，以后续支持持久化到任何一个存储后端
  - 比如fs，比如任何协议的网络存储，这方面可能 [opendal](https://opendal.apache.org/)是一个不错的选择

### entity scope的其他作用

entity scope可以作为某些场景片段的dropper，方便资源管理。

比如某个外部文件的load，这样不用给每一种loader都添加“这个loader load了哪些entity方便后面drop”这种feature
