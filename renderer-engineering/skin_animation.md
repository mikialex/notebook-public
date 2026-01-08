# 蒙皮动画原理

蒙皮（Skin）：node sub tree构成的关节（Joint）树，每个node自身的space是关节坐标系。

蒙皮数据：per vertex数据，决定使用哪些关节（local joint indices），和这些关节的混合权重。

蒙皮数据的生成过程：

- 以合适绑定的形态（任务一般是T pose）完成物体建模。
- 设置关节hierachy，移动某个关节到合适的位置
- 设置（绘制）该关节的权重

计算蒙皮后的顶点位置的过程：

- 访问attribute蒙皮数据找到该顶点多个控制关节和其权重
- 对每一个关节，按权重线性混合该关节的控制结果

某个关节的控制结果（被蒙皮控制后的顶点的local space位置）的计算过程：

- **某个顶点被某个关节控制，意味着该顶点在运动/动画过程中（大致）相对于该关节的关节坐标系不变**
- 绑定蒙皮时决定mesh的vertex position属于哪个（些）关节，是在标记该vertex是应该应用哪个（些）关节的关节space（及其权重）。

- vertex buffer中取到的vertex position是绑定时的localspace，所以先需要转化到该关节的关节space的position
  - 实现这一步需要乘上**绑定时**root关节space到该关节space的transform mat，即**绑定时**joint node的world mat的inverse
    - 这个数据是和关节信息一起存储的。
    - 某个顶点被某个关节控制，意味着该顶点在运动/动画过程中相对于该关节的关节坐标系不变，所以也可以直接将该mat预乘到vertex buffer内，即vertex直接取到的就是关节space的position
- 下一步，从关节space的position再转换到实际渲染的local space。这一步就是乘上该关节space到root关节space的transform mat，即joint node的world mat
- 这两个矩阵一般在host端预先乘好，然后直接应用

> 个人觉得很多讲解蒙皮动画相关矩阵的文章教程写的很差，所以总结此文于此。这部分实现一般很少修改，甚至很少接触和阅读，此文亦帮助复习理解。
