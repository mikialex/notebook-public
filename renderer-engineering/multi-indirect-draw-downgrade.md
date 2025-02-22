# multidrawindirect 降级

在wgpu下完整的gpu driven multidraw需要用户设备支持multidrawindirectcount+nonezerofirstinstance

根据 wgpu之前接口调研的结果:

- 只有multidraw是不够的，因为只能画一个draw command
- multidrawindirect也是不够的，因为draw command本身的数量没法由gpu决定，需要multidrawindirectcount

## 为什么需要 nonezero-first-instance

在顶点着色器中pulling vertex的时候，你需要知道当前顶点属于哪个draw command，才能去draw command reference的range去取的顶点数据。

vulkan提供了draw id可以直接在顶点阶段访问此数据

<https://registry.khronos.org/VulkanSC/specs/1.0-extensions/man/html/DrawIndex.html>

wgsl标准中没有暴露此id，wgpu naga在native实现上也没有暴露此id。

一个目前采用广泛的workaround是利用nonefirstinstance，直接在command里的firstinstance填写drawid，这样在顶点着色器中可直接用vertex instance index拿到drawid。

nonefirstinstance也需要开feature并不能保证有，如果没有，那么问题会很麻烦。因为你缺少了关键的全局信息，因为multidraw下vertexid是局部（per draw command）的，如果没有支持，意味着整个连mutlidraw都是不能用的。

或许我应该开一个issue让wgpu expose drawid。。

### wgpu dx12 实现问题

wgpu dx12 backend，虽然feature 查询是支持 nonezero-first-instance，然而实际上没有实现。

<https://github.com/gfx-rs/wgpu/issues/2471>

<https://docs.google.com/document/d/1hjEFwJo-EU2noaG8_mV5D_e3fYB_bs8sy-RGs4oszro/edit?pli=1#heading=h.fshi85nj57x0>

dx12 并不是不支持 nonezero-first-instance，而是webgpu mutlidraw drawbuffer的布局对于dx12不能直接绑定shader使用。dawn采用了另一个compute shader来生成另一个buffer来适配，来将正确的数据提供给shader内的instanceid。

dx12行为上是支持nonezero-first-instance，只是其instanceid是相对的，即总是从0开始。

在wgpu修复相关问题之前，我们可以自行实现dawn的workaround，通过compute shader生成一个递增的instance vertex buffer，并声明一个instance vertex in，来得到此数据。

----

## 降级实现

考虑到平台很有可能支持会有问题，这部分讨论如果用一个indirect draw来做gpu driven。

gpu driven前stage（比如剔除）得到的结果是draw command，可以简单认为draw command描述了一个mesh buffer的range（offset+size）。

### 数据准备

#### 整体的draw size计算

draw indirect需要填写的draw的range的size是**所有draw command draw range的size的和**

#### vertex 寻址

要完成vertex数据访问，需要：

- vertex属于哪个draw command
- vertex在draw command的相对index
  - 可以通过vertex index 减去 draw command base index 得到

而我们仅有的只是vertex index，所以

我们需要能够从**vertex index 得到 draw command index，以及 base index**

### 具体做法

- 对per draw command的size用compute shader计算prefix sum
  - prefix sum的最后一项即 所有 **draw command draw range的size的和** sizeall
- allocate sizeall 大小的buffer
  - 用compute shader per draw command per thread的填写base index和draw index
  - base index和写的起始位置读取 上述prefix sum
  - draw index用global invocation id
  - 得到的buffer使得我们可以 **vertex index 得到 draw command index，以及 base index**

### 性能潜在问题，和改进方向

#### 准备数据copy的成本是否过高

如果我们认为gpu driven的前阶段，即剔除和lod，使得我们总是能生成合理的渲染workload，比如能够保障实际要绘制的三角面一般不会超过500w。那么这个成本是可以接受的。

`500w * 3（顶点）  * 8（byte） = 114mb`。写114mb的数据，对于gpu来说是完全没有问题的。即便要付出代价也远远比没有这个workaround 导致gpu driven根本无法work导致的代价要高。

#### copy数据的问题和改进方向

假设mesh size数据分布极不均匀，那么会出现只有几个thread在写数据，导致不同thread间的工作量极不均衡，导致性能问题。

这个情况在长期来看是不存在的，因为gpu driven的前阶段一定会优化成以meshlet为粒度进行处理，而不可能是mesh，而meshlet size基本是一致的。

假设非要解决这个问题，以支持某些降级情况。那么可以用command size prefix sum的结果来生成另一个copy task buffer ，通过copy task buffer来dispatch compute执行写。而这个新的copy task buffer 的workload是均衡的，一个过大的mesh会拆分成多个copy thread来copy。

生成copy task需要dispatch一个compute，这个compute的worksize是draw size all / copy粒度，可以考虑per thread 切分copy task，然后收集sub copy的结果。收集过程需要先disaptch计算 sub task count，并计算其前缀和，然后再执行切分写入。如果简单采用bumpallocator来收集copy task。会导致渲染order发生变化。渲染order即便无所谓一般也是需要保序的，因为可能会导致闪烁等问题

用compute shader来合并buffer，合并buffer的写入，类似做法在[storage-buffer-sparse-update](storage-buffer-sparse-update.md)一文中也有涉及。

### 直接对command list做二分查找

可能可以per vertex的对draw command size prefix sum做二分查找，来得到drawid信息。但是实际工程上我们没有做相关尝试，因为直觉上判断这种做法在性能上可能会有问题。per thread access一个很大的command table相比copy buffer要付出更多带宽成本，而优化相关的带宽实现是复杂的。
