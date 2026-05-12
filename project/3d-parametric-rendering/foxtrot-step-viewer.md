# foxtrot step viewer

[foxtrot](https://github.com/Formlabs/foxtrot) 是一个 rust 编写的 step 文件loader。它实现了step file的解析，三角化和显示。

> 这个项目我很久之前就star了，这个项目的[作者](https://github.com/mkeeter)是我最喜欢的程序员之一。

这个项目的大致流程是：

- 编写parser解析 express schema描述文件（express crate）(是的，你得先编写spec的spec的schema的parser！)
- 解析用express编写的 step ap214标准，生成step file的parser（step crate）
- 用step file parser parse step文件
  - 读取场景parent和transform信息，构造场景树
- 读取几何相关数据，实现三角化（trianglulate crate），导出stl或者渲染
  - 读取常见的曲面曲线类型，可能有实现/支持不完全的情况

step中描述3d曲面造型信息，基本都是通过n个参数曲面+曲面上n个闭合曲线的裁切范围构成的。

闭合曲线可能是分段的，每一段有不同的曲线类型。曲线是3d空间中定义的。foxtrot实现曲面裁切，或者说实现这样裁切的3d参数化曲面显示的流程是：

- 在世界（或者local）空间离散化闭合曲线，得到vertex的loop。（这个线是可以直接用来做3d曲面边界显示的）
- 将上述vertex 投影（foxtrot中称之为lowing，反之rasing）到曲面的参数空间
  - 因为离散化误差或者本身几何的容差（在曲面造型设计里，判断连接性都是靠容差判断的，他们数学上其实并不在一起，所谓的A级曲面设计只是容差控制非常严格（导致造型难度变高，可能需要更高次数的曲面来拟合来达成相关的连续性判断））。所以vertex其实并不在曲面上，所以其实是投影到参数空间中的最近点。
  - 一些简单曲面这样的lowing是可以简单计算/解析的计算的。对于nurbs曲面。fox的做法是对每一个nurbs曲面预计算8 x 8的离散点，然后找到最近的一个点开始用牛顿法来求解。。
- 投影到参数空间后，这些点集内部（其实fox的实现很粗糙，根本没有保证在border的内部，如果有问题直接删了重试罢了）插入一些点，这些点叫做steiner_point。很显然是要插入这些点的，不然这个曲面内部根本没有任何三角化。插入后在参数空间做带border限制的Delaunay三角化。然后在转化到世界空间的mesh
