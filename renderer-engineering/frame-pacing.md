# Frame pacing 的理解

## 一些反直觉的现象

### 某一（些）帧的渲染耗时降低了（这应该是好事），为什么用户反而会体验到卡顿？

这种情况一般出现在绘制帧率低于设备显示帧率的情况。比如渲染帧率30，屏幕显示帧率60。

某一帧的渲染耗时降低，意味着这一帧的present的时间提前。如果这一帧绘制时间小到一定程度，present调用提前到一定程度，那么就有可能这一帧“跟”上设备实际帧率的显示频率，导致上一帧的状态被提早刷掉（因为这一帧结果已经可用了）。但是因为用户是固定每隔间隔开始每一帧的渲染，而下一帧的渲染耗时恢复正常（变慢），那么就会导致下一帧会多显示一次。形成帧率抖动的现象。用户不会因为变快的那一帧而感到体验变好，而是会因为变慢的那一帧感受到体验变差。

如果用户的帧时间处于一个临界的情况，那么帧率抖动就会非常频繁。在120hz屏幕上，假设渲染能比较稳定的实现60hz的性能，那么有可能造成 30hz 120hz 30hz 120hz的抖动现象，而对于用户来说，实际体验就会退化到30hz，而不是60hz。 即便这种情况偶然出现，用户依然能感觉到卡顿。

### 为什么实际显示以恒定速度刷新有效（新）内容，用户还是能感受到卡顿？

这种情况是因为驱动物理/动画模拟的deltatime出现了抖动（正常来说应该是完全恒定的显示器实际刷新率）。这导致直接的视觉变化上空间的不连续（或者说不平滑的连续过程）。

在没有 VK_EXT_present_timing这种api的情况下，开发者一般会在eventloop上每一帧获取时间戳来计算deltatime，然而这个做法其实是错误的。

整个vsync的swapchain是个生产消费的模型，显示器消费端，是用户真正体验的刷新率，这个刷新率和你作为生产者被eventpool驱动的提交率完全不是一个东西。

- 即便显示器实际上有效的以16ms的间隔更新画面，应用测量的deltatime不会是精确的16ms，eventloop本身调用是不稳定的
- 即便显示器实际上有效的以16ms的间隔更新画面，因为提交过快产生了背压，get next texutre被block，导致出现了你测量出22ms的帧耗时（然后下一帧又是10ms的帧耗时）

此时如果用户要体验到流畅的画面，用以物理和动画计算的deltatime必须保持精确不变的16ms。

## 解决方法

有两种实现方式：

### 延迟帧计算逻辑，主动控制和稳定帧率

在事件循环中，动态延迟每一帧处理逻辑（处理用户事件，和提交渲染命令）开始的时间

- 推迟渲染present调用的时间，避免短耗时帧，造成的实际画面更新帧率不稳定的问题
- 采用刷新率倍数的帧时间估计代替eventloop采样的帧时间，避免动画/物理计算抖动

<!-- ### 为什么推迟处理帧内新发生用户事件的处理时间，可以改进交互延迟？

很简单，因为只有等待了一段时间，才能有可能收集到这一帧内的一这部分用户交互事件，这部分收集到的交互，就可以少一帧的延迟。
- 推迟处理帧内新发生用户事件的处理时间，造成的用户交互延迟的问题 -->

#### 具体应该延迟多久？

延迟的不够，那么效果不好，如果延迟的过多，没有留下足够的headspace，那么就会直接造成卡顿。一般需要根据性能数据进行统计。

从这个角度看，渲染的平均帧率其实没有任何意义，要保证用户体验，需要保证worstcase，或者前99%的的worstcase。这也同时说明了某些会造成不确定耗时成本的技术，比如采用垃圾回收的语言，是不适合应用在渲染器开发上

一个简单实现比如有： <https://github.com/aevyrie/bevy_framepace>

#### 推迟处理帧内新发生用户事件的开始处理时间，还可以改进交互延迟

因为只有等待了一段时间，才能有可能收集到最新这一帧内的一这部分用户交互事件，这部分收集到的交互，就可以少一帧的延迟

### 依赖api彻底解决问题  VK_EXT_present_timing VK_GOOGLE_display_timing

- <https://www.khronos.org/blog/vk-ext-present-timing-the-journey-to-state-of-the-art-frame-pacing-in-vulkan>
  - 可以让用户查询某个present call的实际present timestamp和实际帧率，直接拿到真实的presenttime，解决上述deltatime的获取问题
  - 可以让用户指定某个present不得早于某个时间present，这样相当于从源头避免了短帧问题，即我们不再需要通过推迟计算来解决短帧造成的问题（相当于让消费者控制消费进度来避免问题），但推迟计算来改进延迟还是有意义的
- wgpu 改进swapchain api的方向：<https://github.com/gfx-rs/wgpu/issues/2869>

## 参考

- <https://developer.android.com/games/sdk/frame-pacing>
- <https://raphlinus.github.io/ui/graphics/gpu/2021/10/22/swapchain-frame-pacing.html>
  - 我不太理解其中 Faster hardware → latency regression一节的内容，因为我理解事件处理肯定是在渲染之前的，但是它这里是反过来的。只有反过来才有可能得到这样的结论
- [Youtube: Myths and Misconceptions of Frame Pacing / Alen Ladavac, Croteam / Reboot Develop Blue 2019](https://www.youtube.com/watch?v=n0zT8YSSFzw)
- <https://blog.unity.com/technology/fixing-time-deltatime-in-unity-2020-2-for-smoother-gameplay-what-did-it-take>
