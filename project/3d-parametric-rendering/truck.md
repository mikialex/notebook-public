# Truck CAD Kernel

<https://github.com/ricosjp/truck>

- 和 ruststep 是同一作者
  - ruststep用来支持其中的step读写。在 truck-stepio 这个crate
    - 虽然ruststep内置的step cad相关的spec的生成代码，但是奇怪的是truck-stepio还是手写的二次parse逻辑（具体ruststep的调研[参见](./step-file-parse.md)）

<!-- ---

阅读代码的几个目标

- 学习了解brep的基本概念
- 和foxtrot viewer对比三角化实现 -->

---

truck-stepio下从step转换三角形的example：[truck-stepio/examples/step-to-mesh.rs](https://github.com/ricosjp/truck/blob/4fab838a911356ab4b332d7f46916095ab72dbaf/truck-stepio/examples/step-to-mesh.rs)

truck-stepio 读取step，二次parse生成 [Table](https://github.com/ricosjp/truck/blob/cb164bb369b5b0ba665b95c8a36cd0e1a5b61a22/truck-stepio/src/in/mod.rs#L26) 这个结构，包含了step内几乎所有的几何信息。
