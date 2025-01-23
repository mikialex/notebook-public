# 符号计算的优化想法

看此文有感 <https://interplayoflight.wordpress.com/2023/12/29/low-level-thinking-in-high-level-shading-languages-2023/>

有一个数学表达式，用户编写代码进行数值计算。用户编写的代码性能上不是最优的，但是编译器可能不能做彻底的优化。因为

- 用户编写的数值计算代码并不是数学表达式，而是数值计算，在考虑浮点误差的情况下，就无法进行等价转换。
- 编译器要保证优化后的数值计算代码的结果，要和用户的输入程序完全一致。

那么，假设提供给编译器的是

- 符号表达的数学计算结构+输入输出精度范围
- （或者）用户的数值计算代码+输入输出精度范围

是否有可能放松精度生成更加高效的代码？
是否有可能放松性能生成精度更好的数值计算代码？

可能相关的link

- <https://en.wikipedia.org/wiki/Interval_arithmetic>
- <https://github.com/unageek/inari>
