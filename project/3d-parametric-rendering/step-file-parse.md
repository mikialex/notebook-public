# step parser 调研

采用哪一个step parser？

<https://github.com/ricosjp/ruststep> or foxtrot express？

## 两种step parser的实现方式比较

ruststep：

- 采用nom 将step parse到一个动态的结构，因为是动态的结构，这一步不需要schema
- 用户可以动态的读取这一结构，手工的二次读取其关心的内容（类似二次parse）
  - 通过serde的trait来实现，deserialize input上述动态的parse结果。
- 提供了一个expr的compiler，可以根据schema，自动生成rust的上述二次parse的代码
  - 采用nom parse schema
  - 支持通过过程宏，直接内联exp语句来生成rust代码

foxtrot express：

- 采用nom parse schema，直接生成对应entity parse的 强类型rust代码
  - 生成的rust 代码中使用nom parse step file并直接生成数据的readview

总结对比：

- 理论上 foxtrot express会比ruststep快很多
  - 因为没有两阶段parse，尤其是没有第一阶段parse出动态的数据结构的性能损耗
- foxtrot express的生成的代码编译很慢（ap214，生成单crate 4w loc，release编译，5分钟）
  - ruststep理论上编译会快一点，但是没有测试
  - ruststep如果用户手工的二次读取其关心的内容，而不使用schema生成的parser，那么肯定会快很多
  - ruststep可以直接生成部分express语句的parser 代码
- ruststep的代码上看，更加“标准化”，特别的想处理好step方面的问题，scope也比foxtrot express要广
  - foxtrot express相比之下有点hack的感觉
