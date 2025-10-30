# 多viewport支持

## 需求

- 渲染器支持输出到多个surface。
  - 在共享同一份gpu资源的情况下（如果在图形api支持），支持在多个window handle或者web canvas上的绘制。
- 对于单个surface， 渲染支持输出到其中的一部分（支持绘制到其上的多个viewport， 这些viewport可以overlap）。
  - 一种争议是这种情况应用层应该还是只支持surface的全范围绘制，但考虑到应用层也可能负责一部分的自绘，那么绘制surface的一部分其实是合理的需求。如果支持绘制surface的一部分，那么支持绘制surface的多个部分也是合理的需求
  - 实现viewcube就依赖viewport支持overlap。

即便业务层没有多view的需求，一些有用的渲染层feature都依赖于多viewport完善支持

- viewcube
- camera editor(3d场景中一个viewport交互式的修改另一个viewport的相机参数)
- effect compare(两个viewport绑定同一个相机采用不同的渲染配置来对比效果)
- culling/lod debugger(一个viewport可视化另一个viewport的相机的剔除和lod效果)

多viewport支持看起来简单，但实际上有诸多细节需要考虑，如果在这方面缺乏必要的分析，那么容易造成非常hack和buggy的实现。

## 状态区分

需要区分配置/状态/资源是per view，还是per surface的， 还是global的。这样的状态区分是支持多viewport的最重要的技术决策。支持多viewport需要思考所有的，每一个配置/状态/资源，每一个feature，都需要确定是属于哪一的层级，并完成数据模型的修改和迁移。

suface的资源/状态可以在view之间share，global的资源/状态可以在surface之间share。比如一个典型的三层结构：

- global
  - surface A
    - viewport 1
    - viewport 2
  - surface B
    - viewport 1

具体状态区分的例子：

- global
  - scene本身相关的渲染资源，包括camera
  - shadowmap等和viewport/surface无关的效果资源
  - 选择集
- surface
  - swapchain相关的配置
- viewport
  - surface内的viewport
  - 任何和一个具体frame相关的状态和资源
    - taa
    - gpu pick

状态区分的决策并没有标准答案，具体需要取决于业务需求。如果我希望一个效果的开关总是同时作用于所有viewport，那么它就应该被共享。

在viewport api的决定上，viewport/output surface设置不应该存在于camera上，因为同一个camera可以输出到不同的surface/viewport。camera应该作为viewport/surface的配置。

配置/状态/资源也要清晰的确定是per camera还是per viewport的（因为camera是vieport的配置，所以我并没有将per camera纳入到上述的三层结构中讨论）。比如支持遮挡剔除的cache数据，是per camera的，而不是per viewport的，因为同一camera的不同viewport绘制实例是可以share这样的cache。比如taa应该是per viewport的，因为我们设计上允许不同viewport有不同的渲染效果，这些渲染效果假设是worldspace的，那么就可以被taa改进效果，那么不同的viewport就需要不同的taa历史，即不同的taa实例。

camera虽然是作为view的配置，在执行per view的更新逻辑时，有可能是需要考虑camera的复用关系，考虑这样的场景：假设用户的多视窗界面，左右两个viewport设置了同一个camera，如何合理正确的实现用户对其中任何一个viewport的相机操作自动的同步到另一个？合理的做法是camera控制器状态的实例应该设计成per camera的，而不是per viewport，应该per camera的以一组该camera的viewports上的操作来进行控制器的状态更新，需要提取这种per camera per viewports的accessor

### 自动share多viewport的状态

因为状态区分的决策并没有标准答案，所以sharing的需求是容易变化的。而[hooks](../general-practice/hooks-pattern.md)的动态性特别的适合表达这样动态share的逻辑。

实现上我们甚至可以完全per viewport的自由的表达renderer的资源/状态需求，这些需求采用hooks的share机制就可以自动的在合理的层级被share。采用这种做法，上述的思考状态应该设置/实现在哪个层级的问题就是不必要的了。

## 渲染实现

一种做法是：既然是viewport，那么就直接使用图形api的viewport api来实现。但实际上采用这种做法在工程上会引入非常多的复杂度

- clear会影响到viewport外的内容（以webgpu行为为准）
- 参与效果处理的中间attachment，比如taa的history，如果所有viewport并不cover全部，那么就有内存浪费，并且不支持viewport相互hoverlap的情况。如果单独以viewport为单独进行处理，那么进行地址转化会很麻烦
- 很难说图形api是否能高效优化小viewport在大canvas上的绘制性能和小canvas一致。因为viewport设置是pass内encode的信息，并不是pass的本身的结构信息，那么可能在移动端上有性能问题。

所以我认为合理的做法，是每个viewport执行独立的全attachment逻辑，最后自己在surface上做composition。这样原管线的逻辑基本可以不做调整，只是要多copy一次结果。

考虑到提交到surface上的内容要有操作系统等再做composition，对于上层应用来说，应该尽可能采用多surface来实现比如分屏/多视图的功能，而不是使用viewport。这样可以避免一次不必要的copy。

如果用户只设置了一个全屏幕的viewport，那么多余的copy应该避免。

这里暂不讨论vr的case。vr应实现特别的多viewport支持。

### Pass info

因为在fragment shader中是无法访问canvas size，viewport size信息的，所以这样的信息通过额外uniform传入。支持viewport意味着相关逻辑需要迁移到viewport size，而不是buffer size。

整个pipeline创建和管理中间attachement，也需要迁移到per viewport的单位上。

### 按需渲染

按需渲染的状态应该实现于per viewport，即便按需渲染是以any change这种最粗糙的实现的，因为不同窗口有不同的temporal效果，需要维持不同的渲染进度。比如左窗口是一个普通的渲染模式，右窗口是离线raytracing，那么即便任何change都没有，右窗口也要保持一致渲染的状态。

如果按需渲染能够获取camera only change信息，那么change信息要正确传播到viewport上再加以处理。

## 原有业务逻辑的调整

虽然本文主要讨论渲染层的支持。但考虑到很多内置于渲染层的图形调试工具，业务逻辑的调整也是必要的：

业务逻辑的调整，主要是mouse相关的状态的行为需要修改

- 需要考虑相对viewport的坐标，而不是window/surface
- 需要考虑当前mouse位置不处于任何一个viewport范围内的行为
  - 可能需要根据交互需求，保存上一个离开的viewport（同时上一个离开的viewport可能已经被销毁）
- 需要考虑完全没有设置任何viewport情况下的行为
- 需要考虑viewport可能重叠

需要考虑的业务组件，比如

- gizmo 拖动到viewport外部，以及进入另一viewport
- camera control鼠标交互到viewport外部

## view dependent transform

视角相关transform更新的问题是多viewport支持的另一大问题。这一能力也是editor场景常见的需求，比如和相机距离缩放无关的scale，比如billboard。通用来说就是收到相机相关因素影响的local matrix更新。

传统情况下，一般有上层业务逻辑根据camera来直接更新场景中对象的local matrix。为了支持这么做，viewport的渲染逻辑要支持让用户的更新逻辑能够注入到每一个view的渲染之前。（更加基本的问题：我们不能让用户控制自己完成viewport的渲染，因为渲染器本身要处理上述上层share结构包括按需渲染的细节，这本身就是内部实现。）

```txt
for each camera {
  do scene view related update logic
  do scene update
  do scene rendering
}
```

这么做的最大问题，是性能问题。

- 更新循环增加到per view
- 相关的gpu数据被不同的view更新反复刷新，即便它们没有任何变化。

我认为这个问题的合理解法，是让用户直接自行维护per camera/per view的 node local matrix override数据。这个override数据包含持久化的gpu数据。并且在per viewport渲染时绑定

- node local matrix override + node local matrix origin => 增量维护world matrix override
- world matrix override注入到渲染管线，原node逻辑查询，会多查询world matrix是否被override
- node local matrix override和world matrix override在数据存储上都是sparse的
- node local matrix override的更新完全是自由的
  - 并没有要求需要实现billboard等具体的行为，只要有override设置进来即可
  - 可以采用不同强度的实现

之所以不推荐采用shader实现，主要是因为

- 需要支持host pick
- 这类行为是多样的，采用shader内置支持，不如直接暴露override matrix。
