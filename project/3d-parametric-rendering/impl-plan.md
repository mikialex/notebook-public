# 开发计划

- milestone 1 完成基本的曲面准备和测试环境
  - n阶nurbs曲面的数据类型定义
  - n阶有理贝塞尔曲面（以下简称B-data）的数据类型定义
  - n阶nurbs曲面到B-data的转换（节点插入法）
  - 其他step内常见的曲面到B-data的转化（optional）
  - B-data的cpu三角化实现（uniform）和rendiation viewer debug显示（用来做参照和基本fallback）
- milestone 2 gpu上的三角化实现mvp
  - 实现全局统一细分配置的 uniform 的三角化
    - 基本的dispatch 和draw流程
    - 迁移曲面求值逻辑到gpu
      - 评估高阶曲面的计算成本，目标设备的性能容量
- milestone 3 gpu上的自适应三角化实现mvp
  - TBD
    - 决定走eter方案（直接计算uniform细分等级），还是递归自适应方案，还是两个都做
    - mvp不做性能优化，不采用先进api，兼容性保证到webgpu标准

可以参考/利用的rust开源库（目前看是比较有限的）：

- <https://github.com/mattatz/curvo> 包含nurbs的定义，cpu三角化，一些建模操作的实现（lofting, sweeping, revolving）
  - <https://docs.rs/curvo/latest/curvo/all.html#structs>
- <https://docs.rs/neco-nurbs/latest/neco_nurbs/struct.NurbsSurface3D.html>

细分方面的成熟工业实现  <https://github.com/PixarAnimationStudios/OpenSubdiv>

- 参考文献<https://www.opensubdiv.org/docs/references.html>
- 只支持catmu和loop
