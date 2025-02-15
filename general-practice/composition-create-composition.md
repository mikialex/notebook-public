# 组合创建组合

组合是我最为常用的设计模式（虽然我并不认可设计模式这套东西）。

这种做法的步骤是：为一类行为定义接口，为若干基本的类型实现该接口，实现若干单纯负责组合行为的组合子或组合容器，为这些组合子实现该接口，最后用组合子去组合基本实现来实现需求。

在多年的进一步实践中，基于这套做法，又演化出了新的做法，这里我称之为「组合创建组合」

这种做法很简单：如果某些类型要抽象的行为或能力，无法简单的在接口上一步完成得到最终结果。而是需要做一些必要工作来得到另一类类型，那么就有必要让该接口，返回另一接口的抽象实现。因为该接口本身是为了表达组合实现，返回的另一接口实际上的实现也是组合模式。整体上就是定义了一套组合结构，然后这套组合结构返回了相同形态的另一套不同目的的组合结构。

如果一组行为要组合实现，每个行为有两个子步骤，而其中第二个步骤，需要等待这一组行为的所有第一个步骤完成才能开始，那么这种模式就会自然出现。A接口创建B接口，即A的结果形成了B，B的实例保存了A组合实现的执行结果，然后计算再以B的目标重入来完成剩余逻辑。

在实际项目中，因为上述的这种纵向的流程结构是多见的，如果在整个架构上充分应用组合，那么组合创建组合，甚至组合创建组合再连续的创建下一层组合是多见的。我在实际项目中设计和实现过4层组合，虽然整个抽象结构宏伟，但每一层接口都行为分工明确，只要对基本流程和以及对这种组合创建组合的模式有正确理解，整体的工程结构是非常整洁和自洽的。
