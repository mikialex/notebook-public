# 3D parametric surface & curve rendering

- 整理和研究cad场景下，改进曲面曲线渲染的性能/消耗/效果的，技术方向和技术细节
- 记录相关原型实现和具体实践进展

cad类型的数据，比如发动机，载具，齿轮，复杂机械，工业品，建筑。这种类型的场景很模型都非常精致，其几何都是通过参数化曲面来定义的。其对于显示的性能和效果有很多技术上的探索空间。

## 直接支持矢量数据绘制

考虑实现一个3d版的[vello](https://github.com/linebender/vello)，以某种方式直接支持3d矢量内容绘制，以在cad场景中：

- 解决传统方法下因为离散精度不足，在放大曲面曲线细节后显示质量不佳的问题
- 实现合理的LOD，大幅改进渲染成本，在复杂大场景下相比传统方案具备显著性能优势

### 传统的cpu预计算mesh的价值

性能的角度：

1 对比直接支持矢量数据绘制的方案，只将优化应用于曲面/曲线的内部，对于复杂大场景是不够的。因为场景中这些矢量图元的数量可能有几百万上千万，以至于他们本身就是类似三角形级别的数据。

对于最简化的case，每一个矢量图元，只离散为一个quad，甚至一个point，也有非常大的开销。所以仍然需要某种形式的场景级别lod。既然谈到传统的lod，那么从可行性看，实际还是需要基于预计算的离散mesh（或者离散的其他无结构数据，比如点云和高斯）实现。

进一步的，在cad场景下，三角化应该考虑brep的topo信息，实现基于topo的水密三角化，具体参考 [Topology-First B-Rep Meshing](https://arxiv.org/html/2604.02141v1), 或者 meshStep（下面step viewer有提到）的实现。这么做的原因是，我们依赖面与面之间的连接性来大幅增加绘制和模型简化单元的处理粒度。以避免上述海量矢量图元本身的性能影响。具体来说：我们可以假设矢量图元的数量是非常多，但是body的数量比较可控。这可以说是一种基于bodytopo信息的场景HLOD方案。 另一方面处理粒度的增加，有利于提高模型简化，特别是极简化模型的质量。否则极简化模型会因为缺乏连接性而破碎。按照这个思路，场景层的曲面api也需要存储和处理拓扑相关的信息（可以考虑分两层处理，optional的处理包含拓扑的曲面转化到gpu绘制所需的曲面）。

2 从单个曲面角度进行实时三角化，可能会三角化出非常不适合渲染的结果，单个曲面因为曲面建模时的分面设计考虑，可能会产生非常细长的曲面条带，非常细长的条带会（特别是gpu上的三角化）生成出非常细长的三角形。实际的完整形态可能是多个细长的条带拼接出的，并不需要内部的曲面分面结构来指导三角化，如果考虑拓扑的三角化，再配合remesh则不会出现这样的情况。

## 基于topo的水密三角化的不适用之处

brep模型本身在数学上并不存在水密性，而是通过容差，并指定拓扑关系强行“缝合”的，对surface之间的容差的忠实显式展示，对于用户进行曲面建模质量调优是必要的，在这种情况下，基于topo的水密三角化反而实际上不能应用。只有当视角离远显示时，此时用户的关注点不在于曲面建模，而是回归到浏览性质，那么在brep body层面追求水密性的显示才有意义（性能和效果）。

<!-- ## 

采用gpu自适应离散的方法，可以在避免大量内存的情况下，大幅改进曲面内的显示效果问题。 -->

## step input

 [step](https://en.wikipedia.org/wiki/ISO_10303-21) step 是一个通用开放的的cad数据交换标准。实现上计划以step作为主要数据输入格式。涉及到曲面曲线造型的一些建模软件，一般都可以导出/保存这种文件。

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
- B 直接从曲面通过数值方法，生成表面信息，再生成Gbuffer
- C ray tracing （not fully researched， not promising）

[研究资料整理](./research.md)

[开发计划](./impl-plan.md)
