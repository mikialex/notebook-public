# step file viewer

我一直考虑做一个 [step](https://en.wikipedia.org/wiki/ISO_10303-21) file的viewer。 step file是一个通用开放的的cad数据交换标准。我本科读的是产品设计，使用过的涉及到曲面曲线造型的一些建模软件，可以导出/保存这种文件。我想做这个文件支持的另一个原因是有大量的cad数据，比如发动机，载具，齿轮，复杂机械，工业品都是采用这种格式，这种类型的场景很模型都非常精致，因为几何都是通过参数化曲面来定义的。

step是一个非常[复杂](https://www.steptools.com/stds/stp_expg/arm.html)的文件格式。格式本身通过一个称之为express的schema语言来描述，express是个比较复杂的schema描述语言。

## foxtrot step viewer

[foxtrot](https://github.com/Formlabs/foxtrot) 是一个 rust 编写的 step 文件loader。它实现了step file的解析，三角化和显示。这个项目我很久之前就star了，这个项目的[作者](https://github.com/mkeeter)是我最喜欢的程序员之一。

我把这个项目的代码里外看了一遍。对整个流程有个大致框架性的理解：

大致流程是：

- 编写parser解析 express schema描述文件（express crate）(是的，你得先编写schema的schema的parser！)
- 解析用express编写的 step ap214标准，生成step file的parser（step crate）
- 用step file parser parse step文件
  - 读取场景parent和transform信息，构造场景树
- 读取几何相关数据，实现三角化（trianglulate crate），导出stl或者渲染
  - 读取常见的曲面曲线类型，可能有实现/支持不完全的情况

step中描述3d曲面造型信息，基本都是通过n个参数曲面+曲面上n个闭合曲线的裁切范围构成的。

闭合曲线可能是分段的，每一段有不同的曲线类型。曲线是3d空间中定义的。foxtrot实现曲面裁切，或者说实现这样裁切的3d参数化曲面显示的流程是：

- 在世界（或者local）空间离散化闭合曲线，得到vertex的loop。（这个线是可以直接用来做3d曲面边界显示的）
- 将上述vertex 投影（foxtrot中称之为lowing，反之rasing）到曲面的参数空间
  - 因为离散化误差或者本身几何的容差（是的，在曲面造型设计里，判断连接性都是靠容差判断的，他们数学上其实并不在一起，所谓的A级曲面设计只是容差控制非常严格（导致造型难度变高，可能需要更高次数的曲面来拟合来达成相关的连续性判断））。所以vertex其实并不在曲面上，所以其实是投影到参数空间中的最近点。
  - 一些简单曲面这样的lowing是可以简单计算/解析的计算的。对于nurbs曲面。fox的做法是对每一个nurbs曲面预计算8 x 8的离散点，然后找到最近的一个点开始用牛顿法来求解。。
- 投影到参数空间后，这些点集内部（其实fox的实现很粗糙，根本没有保证在border的内部，如果有问题直接删了重试罢了）插入一些点，这些点叫做steiner_point。很显然是要插入这些点的，不然这个曲面内部根本没有任何三角化。插入后在参数空间做带border限制的Delaunay三角化。然后在转化到世界空间的mesh

## 3d版vello的可行性

> 注：仅是初步的不完整研究，其可行性也是不确定的，内容会随时更新调整。

和foxtrot一样简单的转化为三角形模型肯定是可以的，但是我一直幻想做一个3d版的[vello](https://github.com/linebender/vello)，来解决cad场景：

- 能够无限放大曲面曲线细节而不会出现传统渲染方法离散精度不足的问题
- 大幅改进渲染性能，特别是极其复杂的cad场景

### 可能的做法和技术路线

#### 直接raytracing 参数化曲面

- [https://www.mattkeeter.com/projects/mrep/](https://www.mattkeeter.com/projects/mrep/)
  - 结论：比三角化+bvh 再tracing性能慢很多
- [https://dl.acm.org/doi/pdf/10.1145/800064.801287](https://dl.acm.org/doi/pdf/10.1145/800064.801287)

#### 实时三角化（目前倾向的技术方向）

- 根据视角自适应三角化密度
  - 在保证性能的基础上，是否能进一步改进到曲率上
- 采用mesh shader

### 流程梳理

#### input

- 某类curve的所有instance buffer
- boundary -> curve one-to-many buffer（ordered）
  - curve: (curve-type-id, curve-id)
- boundary -> surface-patch-id buffer
- surface-patch -> boundray-id + surface-id + bounding buffer
  - bounding可能并不方便精确计算，可采用较为保守的估计
- 某类surface的所有instance 及boundary reference
  - surface 不采用bounding进行剔除，因为border curve的剔除已经能够实现这一能力

#### output

- triangle mesh buffer

#### 流程

- 所有surface-patch利用bounding信息，进入其他gpu 剔除管线进行预剔除得到visible surface-patch
- 计算visible curves，fanout（range二分查找）
- per visible curve 离散化
- per surface patch lowing curve离散化后的点到surface 参数空间
- per surface patch 三角化 参数空间的boundary
- 参数空间的mesh，自适应细分并输出到世界空间

缺失的待调研具体算法需求：

- gpu上的polygon(无自交，可能有洞，非凸，多个)三角化并行实现（有效的per vertex的并行） !blocking!
- 三角化细分的并行实现  !blocking!

性能风险

- 上述三角化相关算法的性能
- 高次曲面 lowing和raising 非常昂贵。每一帧全量生成很有可能顶不住
  - 是否可以在local部分采用简单的细分算法（正确性问题，接缝问题）
  - 可能要设计cache和渐近行为来避免全量生成（所以实际上整个体系是退化成gpu driven的渐近曲面mesh显示优化）
- curve和triangle继续细分的条件不明确，目前设想是差值点raising到world然后对比该点到旁边点线段/面垂直距离是否小于给定阈值
  - 可能要根据具体曲面类型来决定上述尝试的差值点个数和位置，比如大于两次的曲线，采用一个点判断很可能是错的。
- cad 类型的模型，内部结构是非常非常丰富的，所以以完整曲面进行剔除估计效果很差。而又很难找到一个曲面的子结构来做剔除。所以潜在要做的三角化可能会很多。

### 参考资料及WIP搜索实现

- triangulation
  - foxtrot 是自己写的Delaunay
    - <https://en.wikipedia.org/wiki/Delaunay_triangulation> O(n log n)
    - <https://en.wikipedia.org/wiki/Constrained_Delaunay_triangulation>
  - 假设monotone的boundary O(n) 实现
    - <https://en.wikipedia.org/wiki/Polygon_triangulation>
  - refinement
    - <https://en.wikipedia.org/wiki/Delaunay_refinement>
  - gpu实现
    - <https://www.comp.nus.edu.sg/~tants/gdel3d_files/gDel3D.pdf>
    - <https://www.comp.nus.edu.sg/~tants/flipflop_files/flipflop.pdf> 「reading WIP」
      - <https://www.comp.nus.edu.sg/~tants/flipflop_files/flipflop-poster.pdf>
    - 先rasterize算voronoi图， 感觉不太好
      - [Computing 2D Constrained Delaunay Triangulation Using Graphics Hardware](https://www.comp.nus.edu.sg/~tants/cdt_files/TRB3-11-report.pdf)
      - [Computing 2D Delaunay Triangulation using GPU](https://www.comp.nus.edu.sg/~tants/delaunay2DDownload_files/cao_hyp_2009.pdf)
      - [Parallel Delaunay Triangulation](https://conniefan.github.io/DelaunayTriangulation/Parallel_Delaunay_Triangulation_on_CUDA.pdf)
    - <https://github.com/chenzhenghai/gDP2d>
      - 这个只是refinement过程是gpu的，初始的triangulation用的还是上面的做法
    - [2D Triangulation of Polygons on CUDA](https://ieeexplore.ieee.org/document/6641459)
      - 只是将per invocation 做独立的earcut， 没有参考价值
    - [A GPU Implementation of Dynamic Programming for the Optimal
Polygon Triangulation](https://www.cs.hiroshima-u.ac.jp/cs/_media/triangulation_ieice.pdf)
      - convex, 而且追求optimal，不太符合要求

[A three-dimensional parametric mesher with surface boundary-layer capability](https://www.sciencedirect.com/science/article/pii/S0021999114002447)

## 大幅改进复杂场景的渲染性能和显示效果的其他方案

另外一种做法，就是生成足够高精度的mesh，然后全部预处理成mesh lod graph，做好虚拟化。

## 其他step的有用资源

[grabcad](https://grabcad.com/library?page=1&time=all_time&sort=most_liked)是一个cad file share的网站，上面有很多step格式的文件可以用来测试。

[step-tool](https://www.steptools.com/)是一个该文件格式相关的非官方网站，上面有step spec的细节（官方的iso文档几乎下载不到，购买的话价格非常贵）

另一个step and express rust parser[ruststep](https://github.com/ricosjp/ruststep)
