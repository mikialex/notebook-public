# 分析调试内存问题

以下调试经验主要适用于 native 平台上采用 rust 语言编写的应用程序。

## 内存错误

### 避免问题

- 尽可能不编写unsafe代码
- unsafe代码需要注明安全边界
  - 并编写额外的*可通过编译配置关闭*的运行时检查来验证安全边界
  - 安全假设不能依赖用户输入
- unsafe代码需要编写覆盖率完善的miri的单元测试
- 采用[randomize layout](https://github.com/rust-lang/compiler-team/issues/457)编译配置尽早暴露transmute unsound问题
  - transmute是容易写错的，建议在api上强制写清楚类型。即便在开发时编写正确，后续的长期维护很可能导致类型变动，然后不知不觉transmute错误的类型。（后续：clippy已经默认强制要求写明类型）

### 排查和修复问题

- 仔细review unsafe code，完善上述测试和边界条件的运行时验证
- 采用目标平台上包含额外内存问题检查的工具，如[WinDbg](http://www.windbg.org/)
- 采用目标平台或者编译器提供的sanitizer工具
  - 比如rust编译内置了[sanitizer](https://doc.rust-lang.org/beta/unstable-book/compiler-flags/sanitizer.html)(不过windows上msvc toolchain似乎没有支持？)
- 更换具备一定内存保护能力的allocator来验证问题（但是不能作为修复）

## 内存泄漏

### 避免此问题

- 尽可能不编写unsafe代码
- 尽可能不使用引用计数
  - 即便要使用，尽可能的使用weak ref。（当然还需要正确考虑清楚谁来hold强引用，不然数据直接没了）

### 修复此问题

修复问题，即排查泄漏位置，同样的和排查内存用量过多问题一致，见下文

## 过多内存用量问题

目前并没有完善的工具能用来高效的排查内存用量问题。用量有两类，某一段时间内的峰值用量，和应用进入稳定状态的常驻用量。一些用过的工具和经验如下有：

dhat有rust[实现](https://github.com/nnethercote/dhat-rs)，通过替换全局allocator来集成。但使用体验不佳，报告格式可读性极差，看不明白。

linux only 的 [heaptrack](https://github.com/KDE/heaptrack)比较好用，这是[具体用法](https://gist.github.com/HenningTimm/ab1e5c05867e9c528b38599693d70b35), 我试图寻找类似的跨平台替代品，但是没有。

RA的[这篇文章](https://rust-analyzer.github.io/blog/2020/12/04/measuring-memory-usage-in-rust.html)比较有帮助, 其中介绍到了一种阿基米德方法（我在公司里一般口头戏称之为反向排水法），即将排查的代码数据结构及维护逻辑double一份，通过观测内存上涨来估计之前这部分逻辑的绝对内存消耗。这个方法在项目中排查中采用过，具备很高的实践价值。
