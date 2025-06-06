# id分配由caller决定

考虑有某系统需要用户控制和维护带有引用关系的数据结构，其中引用通过id来表达。

id分配可考虑这样设计：用户在创建对象时同时提供对象id，而不是由对象创建方（library）实现来负责生成id。对象创建方实现可以假定id满足正确性方面的约束，并考虑在debug模式下实现额外的可选的约束验证逻辑。这么做的好处是：

可能用户已经拥有现成的满足正确性约束要求的id， 可以直接使用

- id分配和管理是有成本的，避免重复的id分配提高性能
- 如果实现提供了额外的id，那么大概率会导致用户要实现两种id的remap，付出更多不必要的性能和消耗

即便用户没有现成id，可考虑提供独立的id生成模块给用户使用，用以适配实现。

---

这个原则我似乎早有耳闻（但出处已经不详），但直到我自己反复设计和实现一些资源对象管理系统时重新发现了这一做法。
