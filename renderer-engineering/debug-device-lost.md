# 如何排查 数组访问越界导致的 Device lost

数组越界是导致device lost最常见的原因。如果没有好的思路，排查成本极高。

- 难以定位是哪一个shader哪一个dispatch，哪一个invocation越界，以及导致越界的rootcause
- 即便越界，也大概率不一定会device lost，这是最麻烦的。

## 早期采用的外部工具

目前尝试过的有 Nvidia aftermath monitor，一般会随驱动相关软件一同安装。

可以界面配置全局开启。但实际使用上有一些问题：

- 对于不一定复现的device lost，能够增加device lost复现的几率，但是不能保证复现。
- lost后，只会显示lost的shader，其shader spriv binary的md5？hash。
  - 实际上没有找到官方文档提到这一点，这一点是我同事神奇猜测出来的。
  - 要确定是哪一个shader，渲染器运行时创建shader的过程要存储计算hash来在lost之后重新查询，来找到是哪一个shader。

除了采用monitor界面，还有一种方式是直接集成sdk。有现成的rust crate来做这个事情。 <https://crates.io/crates/nvidia-aftermath-rs>

这个crate接入方式需要添加于wgpu vulkan后端实现。实测是有用的，以这种方式集成，能够在越界时保证触发lost

## 实际上最佳的工程实践

因为目前rendiation所有的shader，全部都是从自己ir编译到目标平台的shader代码。所以显然的在编译数组索引访问的代码时，添加越界检查即可。越界检查失败，立刻触发一个死循环，以实现在越界时保证触发lost。这个方法实际上在工程上非常有效。很多深藏的bug都立刻暴露了出来。

另外，在工程上，如果触发了越界，可以自动的二分的只给其中一半的索引访问添加越界检查，反复重试就可以定位到具体的越界点。

另外值得一提的是，死循环不一定会导致越界，实际测试metal不会，nv和amd会。开启所有越界检查对于一个典型的workload，整体shader的执行速度大概速度有30%左右的下降。
