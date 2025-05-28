# two-pass gpu 遮挡剔除的细节问题

做法简介： <https://medium.com/@mil_kru/two-pass-occlusion-culling-4100edcad501>

## depth pyramid为什么要做4个采样？

如果要测试的screenspace bounding在特定的合适位置，那么1个采样也是够的。

具体来说，目标是要在pyramid上采样尽可能少的depth texel，以及尽可能小（尽可能低level）的depth texel。采样多，执行遮挡测试的成本就高，采样的texel越大，则获得的结果就过于保守，测试的效果就变差。

想象最极端情况，假设你的物体虽然很小，但是在屏幕正中心，如果强求要一个texel，那么因为整个四叉树的最根节点的branch正好穿过你的物体，那么你只能用最高的level，即全屏幕的结果，即最差最保守的深度来做测试。如果总是采用4个texel，则能解决这个问题。连续的4个texel总是可以在四叉树上的最低level的覆盖任意位置的bounding。对于小于screen texel的bounding，计算出的depth texel四个采样位置会是一样的，此时我们就简单依赖硬件的cache优化采样性能。

## depth 需要向上取 POT

用户屏幕渲染几乎不可能是pot的input，在生成hierarchy depth前，需要向上（保守）取pot，否则会导致在降采样过程中丢失深度数据，导致剔除错误。

问题的核心原因是texture的mipmap size的大小是`full_size >> level_index`计算的，所以每一个非偶数边长的level，到下一层level的过程中都会丢弃一个像素（列）的采样。比如变成为3的level，其下一层level边长为1，而不是2。

### depth 可能可以不需要向上取POT

向上取pot是解决这个问题的一个做法，这个做法简单，但是成本高。

如果降采样算法能够支持3 to 1这种的case。那么在边缘部分按需处理也可以实现。但是实现成本较高。

另一种做法是将用户的非pot frame depth的mipmap数据采用另一个独立的pot texture容器存储。这样做，生产mipmap的实现会简单，成本上升要少一些，但是后面所有使用的地方会变得麻烦，需要封装好。这个做法还有一个好处是，生成mipmap的spd compute shader，需要采用storage image来写mipmap数据，然而用户的frame texture是depth32或者stencildepth24格式，这种格式的mipmap数据是无法以storage image的方式进行绑定的。所以你只能为mipmap数据准备另一个独立的texture 容器。总之可以说因为这些细节，整个实现无比的麻烦。

## 渲染流程中的clear depth

如果渲染流程中掺杂depth clear。那么遮挡剔除流程也需要因为depth clear而被拆分为多个。

如果切分后的物体很少，那么可考虑直接跳过剔除

## 特殊物体的处理

| 物体类别              | 是否能够作为遮挡体   | 是否需要参与遮挡测试  |
| ----------------- | ----------- | ----------- |
| 一般的三角面模型          | 是           | 是           |
| 线和点               | 否           | 是           |
| 关闭了depth write的物体 | 是（ and其他条件） | 否           |
| 关闭了depth test的物体  | 否           | 是（ and其他条件） |

## 如果last frame visible 出现 false positive

后果是如果这个物体如果实际被遮挡，亦然会被认为是遮挡体，即成为不必要的遮挡体。造成当前帧绘制开销上升 不会造成正确性问题。

考虑false positive是因为绘制的物体可能下一帧因为其他剔除原因消失，但是下下一帧又重新出现。正面解决此问题需要对整个visible buffer做重置，但是因为max unit count可能会很高，所以重置的带宽开销很高。又因为正确性如何都不会影响绘制正确性，并且正确性会随着流程运行立刻被纠正。所以last frame visible可以不重置状态。

## 如果last frame visible 出现 false negative

后果是如果这个物体会如果实际是作为遮挡体，但是不会被认为是遮挡体，即少绘制了一个遮挡体，即h-depth数据不够浅，即造成当前帧的遮挡剔除效果变差，以及导致下一帧实际不应该作为遮挡体的物体成为遮挡体，即上述false positive情况。总之，造成当前帧以及下一帧绘制开销上升，不会造成正确性问题。

考虑 false negative是因为整个遮挡剔除刚开始运行，默认都是visible false，所以此时就是完全false negative的状态。
