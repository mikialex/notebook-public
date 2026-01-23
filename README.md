# Notebook

## About

本项目主要存储笔者[mikialex](https://github.com/mikialex)的公开写作。内容主要偏向工程实践和技术研究，亦有非技术类的个人思考。这些条目主要从笔者的其他私有文档中萃取而来，定期手工或半自动同步，并简单校对差漏调整行文。仅代表个人观点，不保证正确性，不包含算法生成内容。禁止商业使用, 转发和引用需注明来源。

暂时没有继续在 <https://mikialex.github.io/> 继续更新。这样考虑有：

- 不再考虑美学，形式上的细节，进一步的回归内容。
- 试图维护一些成体系的内容，注重自洽和时效

本项目不保证文件和路径名的稳定，为了保证链接始终有效，请使用包含git hash的url。

## Table of Contents

- 数据和计算框架
  - 数据建模和状态管理
    - [统一数据模型和数据框架](./data-modeling-and-computation/data-modeling/database-is-fundamental.md)
    - [数据模型的所有权](./data-modeling-and-computation/data-modeling/database-ownership-model.md)
  - 计算表达
    - [增量计算的基本原理](./data-modeling-and-computation/computation/incremental-compute-principal.md)
    - [基于抽象query的数据消费框架](./data-modeling-and-computation/computation/query.md)
    - [query框架的另一种变更数据抽象：DataChanges](./data-modeling-and-computation/computation/data-changes.md)
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
  - [关于Framegraph架构的观点](./renderer-engineering/framegraph.md)
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
  - [表内数据的抽象访问机制](./renderer-engineering/rebuildable-host-data.md)
  - [场景数据虚拟化](./renderer-engineering/scene-virtualization.md)
  - [场景虚拟化实现的细节问题](./renderer-engineering/scheduler-design.md)
  - [渲染器 wasm thread 支持问题](./renderer-engineering/renderer-wasm-thread.md)
  - [多viewport支持](./renderer-engineering/multi-viewport.md)
  - [蒙皮动画原理](./renderer-engineering/skin_animation.md)
  - ray-tracing
    - [GPU ray-tracing 的上层框架](./renderer-engineering/ray-tracing/scene-integration.md)
- 通用工程实践
  - [id分配由caller决定](./general-practice/caller-provide-id.md)
  - [组合创建组合](./general-practice/composition-create-composition.md)
  - [RAII is bad(in some case)](./general-practice/raii-is-bad.md)
  - [hooks pattern](./general-practice/hooks-pattern.md)
  - [Generational arena shrink 的实现改进](./misc/generational-container-shrink.md)
  - [数据容器的内存消耗控制](./misc/resize_strategy.md)
  - [IO组件实现的质量标准](./general-practice/better-io-impl.md)
  - [分析调试内存问题](./general-practice/debug-memory-issue.md)
  - [性能优化的方向](./general-practice/performance-direction.md)
  - [Semantic log level 规范](./general-practice/semantic-log-level.md)
- open question
  - [符号计算的优化想法](./misc/symbolic-compute.md)
- 其他
  - [step-viewer](./misc/step-viewer.md)
  - [Rendiation 六年开发回顾](./misc/rendiation-six-years.md)
  - [远程工作的观点](./working/remote-work.md)
  - [或许你不应该将渲染器称之为渲染引擎](./working/language-corrpution.md)
  - [两种软件类型](./misc/two-kind-of-software.md)
  - [Rust 类型名过长问题](./misc/rust-huge-debug-symbol.md)
  - [制度化加班](./working/overtime-work.md)
  - [面向绩效（利益）工作](./working/result-oriented-working.md)
