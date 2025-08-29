# Notebook

## About

本项目主要存储笔者[mikialex](https://github.com/mikialex)的公开写作。内容主要偏向工程实践和技术研究，亦有非技术类的个人思考。这些条目主要从笔者的其他私有文档中萃取而来，定期手工或半自动同步，并简单校对差漏调整行文。仅代表个人观点，不保证正确性，不包含算法生成内容。勿商业使用。转发和引用需注明来源。

暂时没有继续在 <https://mikialex.github.io/> 继续更新。这样考虑有：

- 不再考虑美学，形式上的细节，进一步的回归内容。
- 试图维护一些成体系的内容，注重自洽和时效

## Table of Contents

- 基础数据和计算框架
  - 数据建模和状态管理
    - [采用关系型数据库作为富状态应用的数据管理方案](./data-modeling-and-computation/data-modeling/database-is-fundamental.md)
    - [数据模型的所有权](./data-modeling-and-computation/data-modeling/database-ownership-model.md)
  - 计算表达
    - [gpu并行计算框架](./data-modeling-and-computation/computation/parallel-compute-framework.md)
    - [gpu上的动态workload计算](./data-modeling-and-computation/computation/gpu-dynamic-workload.md)
- 编辑器和场景层工程实现研究
  - [场景数据和能力的扩展](./editor-engineering/scene-extension.md)
  - [场景基本单元对象](./editor-engineering/basic-scene-object-model.md)
  - [场景prefab支持](./editor-engineering/prefab.md)
  - [基于database的持久化和undo-redo实现](./editor-engineering/database-incremental-persistency.md)
  - [gpu driven editing](./editor-engineering/gpu-driven-editing.md)
- 渲染器工程实现研究
  - [EDSL Shader Framework](./renderer-engineering/edsl-shader.md)
  - [Binding group management](./renderer-engineering/binding-management.md)
  - [Pipeline management](./renderer-engineering/pipeline-management.md)
  - [如何解决Buffer binding count 的平台限制](./renderer-engineering/how-to-workaround-binding-count-limitation.md)
  - [indirect和direct rendering的工程实现组织](./renderer-engineering/indirect-direct-rendering.md)
  - [改进大场景渲染精度](./renderer-engineering/large-world-precision.md)
  - [场景过滤和剔除](./renderer-engineering/culling-system.md)
  - [如何排查数组访问越界导致的Device lost](./renderer-engineering/debug-device-lost.md)
  - [multi draw indirect 降级](./renderer-engineering/multi-indirect-draw-downgrade.md)
  - [two-pass gpu 遮挡剔除的细节问题](./renderer-engineering/two-pass-oc.md)
  - [Compute shader 错误处理的改进方向](./renderer-engineering/compute-shader-error-handling.md)
  - [Storage buffer sparse update](./renderer-engineering/storage-buffer-sparse-update.md)
  - [Reverse-Z 相关问题](./renderer-engineering/reverse-z.md)
  - [渲染器性能优化的迷思](./renderer-engineering/performance-optimization-think.md)
  - [渲染器工程研究思考](./renderer-engineering/rendering-engineering-think.md)
  - [host渲染数据释放机制](./renderer-engineering/dispose-host-data.md)
  - [场景数据虚拟化](./renderer-engineering/scene-virtualization.md)
  - ray-tracing
    - [GPU ray-tracing 的上层框架](./renderer-engineering/ray-tracing/scene-integration.md)
- 通用工程实践
  - [id分配由caller决定](./general-practice/caller-provide-id.md)
  - [组合创建组合](./general-practice/composition-create-composition.md)
  - [RAII is bad(in some case)](./general-practice/raii-is-bad.md)
  - [hooks pattern](./general-practice/hooks-pattern.md)
  - [分析调试内存问题](./general-practice/debug-memory-issue.md)
- open question
  - [符号计算的优化想法](./misc/symbolic-compute.md)
- 其他
  - [远程工作的观点](./working/remote-work.md)
  - [或许你不应该将渲染器称之为渲染引擎](./working/language-corrpution.md)
  - [两种软件类型](./misc/two-kind-of-software.md)
  - [Rust 类型名过长问题](./misc/rust-huge-debug-symbol.md)
  - [制度化加班](./working/overtime-work.md)
