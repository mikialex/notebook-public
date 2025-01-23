# gpu driven editing

gpu driven editing 类比 gpu driven rendering。通过让gpu来决定和执行场景数据的修改和查询，使得支持超大规模的数据编辑能力。有一些已有的应用采用了类似的思想。这类技术一般是解决超大规模问题的决定性的技术（存在性直接决定了技术可行性）

## 定义

- 场景的数据完全持久化在gpu内
- 场景查询和修改采用compute shader
  - 查询比如基于几何的picking，基于条件的筛选
  - 能支持任何合理的方式进行修改，就像cpu的场景一样
    - 当然具体修改性能看实际的修改是否能有效并行，比如用户issue了大量不同类型的串行修改，显然性能无法好过同一类型的大量修改，甚至远远差于cpu的实现
  - 查询和修改都是异步的
- 支持cpu侧异步的增量持久化，可能可以是增量的持久化
  - 毕竟场景最后还是需要存储在文件系统里
  - optional的做异步备份，因为device可能会lost

## 应用

### 3d绘制和雕刻

现在已经有典型的gpu driven editing的成熟引用了：即3d绘制和雕刻。

3d绘制，指用户可以在屏幕上直接对mesh的texture内容进行绘制，可以自由的旋转，观察，并决定绘制位置，可以指定画笔的纹理。

一般来说是全程只使用gpu实现的：预生成物体texture space的object space 顶点位置纹理，绘制时以物体texture为target运行一个quad draw，per fragment采样前面提到的预生成纹理，转化到主相机screen space，和画笔mask texture和主相机screen space 深度做比较，决定是否画出（即更新物体texture）。3d雕刻的过程也是类似。

因为场景的数据量（mesh 顶点数目）可能非常大，所以如此修改逻辑用gpu实现才是合适的，而其数据存储，和修改方式，也符合我们对gpu driven editing 的定义。

### animation

gpu上的粒子系统，物理模拟，或者某些固定的大规模帧动画执行器，可以认为也是符合此类做法

### general gpu scene editing

想象一下场景中有100万的entity，然后你要实现实时的流畅框选，可能还需要选择到被遮挡的entity（这样传统的gpu picking就不能使用了），然后选中之后你还要拖动他们。那么你就非的把所有的逻辑和数据都实现在gpu上不可。

根据我们对场景数据存储于relational db的设想，更加通用的能力依赖于gpu上的数据库查询能力实现，这似乎是一个非常遥远的目标。
