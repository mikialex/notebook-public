# 3d parametric rendering

实现一个3d版的[vello](https://github.com/linebender/vello)，以某种方式直接支持3d矢量内容绘制，以在cad场景中：

- 解决传统方法下因为离散精度不足，在放大曲面曲线细节后显示质量不佳的问题
- 实现合理的LOD，大幅改进渲染成本，在复杂大场景下相比传统方案具备显著性能优势

### 传统的cpu预计算mesh是否还有价值？

只将优化应用于曲面/曲线的内部对于复杂大场景可能是不够的，因为场景中这些矢量图元的数量可能有几百万上千万，以至于他们本身就是类似三角形级别的数据。

对于最简化的case，每一个矢量图元，只离散为quad，甚至point，也有非常大的开销。所以某种方式的场景级别lod必须存在，那么从可行性看，这种场景lod实际还是需要基于预计算的离散mesh实现。

三角化应该考虑brep 拓扑，具体参考 [Topology-First B-Rep Meshing](https://arxiv.org/html/2604.02141v1), 或者 meshStep（下面step viewer有提到）的实现

## step input

计划以step作为主要数据输入格式

多年以来我一直考虑做一个 [step](https://en.wikipedia.org/wiki/ISO_10303-21) file的viewer。 step file是一个通用开放的的cad数据交换标准。我本科读的是产品设计，使用过的涉及到曲面曲线造型的一些建模软件，可以导出/保存这种文件。我想做这个文件支持的另一个原因是有大量的cad数据，比如发动机，载具，齿轮，复杂机械，工业品都是采用这种格式，这种类型的场景很模型都非常精致，因为几何都是通过参数化曲面来定义的。

step是一个非常[复杂](https://www.steptools.com/stds/stp_expg/arm.html)的文件格式。格式本身通过一个称之为express的schema语言来描述，express是个比较复杂的schema描述语言。

[step parser的实现调研](./step-file-parse.md)

## step viewer 参考实现

[foxtrot step viewer](./foxtrot-step-viewer.md)

[truck](./truck.md)

### meshStep

主要是基于拓扑的三角化实现

- <https://github.com/CNCKitchen/meshStep> AGPL
  - readme里有一些其他step三角化的对比
- <https://cnckitchen.github.io/meshStep/> web example
- <https://x.com/CNC_Kitchen/status/2077136207889227868> twitter post

## 技术路线

- A 实时自适应三角化
  - A 在gpu上per fragment处理曲面裁减
    - A ETER-like 在细分前完全决定细分参数，uniform采用
    - B 自适应细分曲面
  - B 在gpu上三角化时直接考虑曲面裁减（和foxtrot 类似）
- B 直接从曲面通过数值方法生成表面信息生成Gbuffer
- C ray tracing （not fully researched， not promising）

[研究资料整理](./research.md)

[开发计划](./impl-plan.md)
