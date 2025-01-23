# Storage buffer sparse update

假设storage buffer的更新者采用了高级的增量更新机制，使得buffer的change在非常细的粒度上被收集。又因为buffer更新调用的api或多或少有固定的调用overhead，那么在更新gpu资源时，直接将细粒度的buffer变更信息进行gpu更新可能并不是一个最优的选择。但如果进行完整更新，则又无法充分利用增量更新的性能优势，并且可能要额外持久化一份host数据。这里就需要在实现上思考取舍。

具体来说实现需要做buffer update合并。合并的取舍，困难之处在于

- 合并更新数据段的开销需要尽可能小，以避免成本超过收益
  - 将逻辑上应该合并的都合并（相邻连续）可能不是好的选择
- 相邻连续合并是不够的，比如大量的interleave change

## 采用分块合并更新

如果可以接受额外持久化一份host数据，那么可考虑对storagebuffer做分区，buffer更新以分区为单位/粒度，分区内自动合并。分区是相同size的，所以合并的开销只是记录change时除取余。

分区大小a，应该选择 buffer copy mb/buffer copy call overhead的比值，即每纯调用的开销可以等价于多少size的纯copy）。比值越大则分区越大/相比维护链表，排序来合并相邻change，分区的做法几乎没有change 合并成本，实现简单。

代价是不考虑合并成本，实际成本可能是最优解的两倍。考虑两个最差的极端情况：

- 假设每一个分区的数据都只变更了1byte。那么相比最优解（每个分区都写一个byte）相当于写size放大了a倍。而a的写时间相当于1次call的成本，所以整体成本比最优解多一倍
- 假设所有数据都修改了，那么相比最优解（完全一个写），相当于写的call的次数多了a倍，而相当于多了一倍的copy overhead，所以整体成本比最优解多一倍
  - 这个情况有可能遇到，但是合并成本显然非常高，所以也是净收益

分区大小可考虑在客户端进行测量（buffer copy速度和硬件和平台相关， call overhead和框架及api相关），也可以考虑大致估计。

## 采用compute shader更新

因为额外持久化一份host数据是非常大的浪费。所以解决此问题倾向于采用compute shader来执行sparse update。

将所有none overlap的更新请求，汇总在一起，形成copy command buffer+ data buffer。然后直接dispatch一个compute去copy数据。

none overlap需要保证因为，compute的thread之间没有执行顺序的依赖。这看起来也的确是一个很高的要求，但是对于大部分实际情况，比如采用storeagebuffer存储某AOS的数据，然后该数据以per-field的机制触发更新，实际上保证这样的none overlap成本和实现是trival的。

参考：

<https://www.reddit.com/r/vulkan/comments/wa9ss2/multiple_sparse_buffer_updates/>

该文评论提到vk 驱动会将大量[cmd data copy](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkBufferCopy.html)优化为compute。但个人认为自行实现也有性能上的价值，因为wgpu的接口调用不是free的，driver识别出这样的情况并转化实现也不是free的。
