# Rust 类型名过长问题（未有效解决）

rust中特别的适用「组合」的编程方式来分解问题实现需求。采用这种方式一般会编写出大量嵌套使用范型的框架代码。这种代码在debug编译时会生产巨大的调试文件。调试文件内主要的内容是巨大的符号名。这些符号巨大的原因是其中包含了非常复杂的范型定义。巨大的调试文件使得编译速度显著变慢，内存消耗显著上升，并且很可能会导致link失败，调试几乎也处于不可用的状态。

一种可行的workaround，是手工的引入box来进行workaround，因为box可以擦除类型。但是这样的workaround会导致性能变差，因为编译优化阶段缺少了更加具体的类型信息。进一步的，我们可以通过条件编译的方式，在release下采用具体类型，因为release不会生成符号名。在debug模式下采用box workaround来避免生成巨型的符号名。

然而这种workaround在实际实践中依然存在问题：我发现实际上release编译，虽然输出的产物不包含符号名，但是在编译过程似乎依然会计算实际的符号名，并且超大的符号名依然会导致link失败。比如在mac上的ld有`ld: Assertion failed: (name.size() <= maxLength), function makeSymbolStringInPlace, file SymbolString.cpp, line 74` 这使得上述的条件编译workaround失效。因为你不得不添加始终存在的box来避免编译失败。添加box的数量和位置都需要手工确认，添加的越多，性能越差，添加的不够，有可能导致编译失败。这显著影响了编程体验。

在windows msvc toolchain上虽然release编译虽然能够接受比ld更长的符号（也有上限），但是其生成的文件（rlib）大小比mac上要大2-3倍，同时pdb也非常巨大，超过1gb的pdb是常见的。因为过长符号的原因，此时甚至采用release编译，比debug编译还要快。

## 其他可能的workaround的方向

采用impl Trait来代替具体的type（失败）：因为从用户看来impl Trait是匿名的类型，所以该类型应该并不会具体化出一个具体的符号名。然而实际尝试下来，目前rust依然会给impl Trait生成具体的类型名，所以这种workaround并不可行。

根据： <https://doc.rust-lang.org/reference/attributes/limits.html#the-type_length_limit-attribute> 用户可以通过该attribute限制某个crate内部的type复杂度。那么一个工程上的workaround就是设置该上限，以此避免某些linker编译失败。简单尝试了一下这个配置似乎不可用

在debugbuild上关闭某些crate的debug symbol，需要断点调试时再开启。尝试了是可用的。但是效果很差，大概只能降低20%的rlib和pdb size。这是很奇怪的现象。

## 观点

我认为巨大的符号名导致编译上的风险对于rust来说是一个重要的问题需要思考和解决，因为这直接制约了对于rust对于最通用的编程和建模方式，使得其在编译上无法scale或者要付出动态性的开销。虽然我对rust的编译器内部细节不甚了解，但是我认为一些可能的做法包括：

- 在任何时候，即便是debug build生成巨大的符号名是没有必要的，对于复杂的符号，根据用户配置的长度阈值，只生成相对外层的名字（因为用户一般也只关心外层的结构），内层名字通过unique的不透明id进行区分。这样的名字对于用户来说是可以接受的。同时用户也可以通过控制阈值来控制名字长度的上限。
- 对于impl Trait的匿名类型，采用和closure一样的方式匿名处理，不生成具体的类型。因为用户采用impl Trait，本身也是希望去封装具体的类型，所以生成具体类型的符号是无用的。

## 相关问题链接

- <https://internals.rust-lang.org/t/exponentially-growing-impl-trait-type-names/7447/4>
- <https://github.com/rust-lang/rust/issues/72408>
- <https://github.com/rust-lang/rust/issues/75992>
- <https://github.com/rust-lang/rust/issues/130729>
- <https://github.com/rust-lang/rust/issues/135849>
