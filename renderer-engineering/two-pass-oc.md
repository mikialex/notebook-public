# two-pass gpu 遮挡剔除的细节问题

做法简介： <https://medium.com/@mil_kru/two-pass-occlusion-culling-4100edcad501>

## depth pyramid为什么要做4个采样？

如果要测试的screenspace bounding在特定的合适位置，那么1个采样也是够的。

具体来说，目标是要在pyramid上采样尽可能少的depth texel，以及尽可能小（尽可能低level）的depth texel。采样多，执行遮挡测试的成本就高，采样的texel越大，则获得的结果就过于保守，测试的效果就变差。

想象最极端情况，假设你的物体虽然很小，但是在屏幕正中心，如果强求要一个texel，那么因为整个四叉树的最根节点的branch正好穿过你的物体，那么你只能用最高的level，即全屏幕的结果，即最差最保守的深度来做测试。如果总是采用4个texel，则能解决这个问题。连续的4个texel总是可以在四叉树上的最低level的覆盖任意位置的bounding。对于小于screen texel的bounding，计算出的depth texel四个采样位置会是一样的，此时我们就简单依赖硬件的cache优化采样性能。

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
