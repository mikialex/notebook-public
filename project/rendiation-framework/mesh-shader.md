# Mesh Shader 集成细节备忘

本文主要讨论mesh shader 在rendiation的集成方法，主要难点和成本估计。

- <https://developer.nvidia.com/blog/introduction-turing-mesh-shaders/>
- <https://developer.nvidia.com/blog/using-mesh-shaders-for-professional-graphics/>
  - ref code： <https://github.com/nvpro-samples/gl_vk_meshlet_cadscene>
- wgpu mesh shader support： <https://github.com/gfx-rs/wgpu/blob/v30/docs/api-specs/mesh_shading.md>。 注意，这个文档已经过时了（其中limit部分，task和mesh的limit是分拆的，不是一起的）我另外还提了个问题 <https://github.com/gfx-rs/wgpu/issues/10020>（结果发现他们在最新版本修复了）
- <https://gpuopen.com/learn/mesh_shaders/mesh_shaders-optimization_and_best_practices/>
- <https://developer.nvidia.com/blog/advanced-api-performance-mesh-shaders/>
- <https://gpuopen.com/learn/mesh_shaders/mesh_shaders-meshlet_compression/>
- <https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdDrawMeshTasksIndirectCountEXT.html>

## scope

- 先只做 attribute mesh system层面的支持， 暂时先不考虑nanite-like的lod meshlet graph
  - 这么做并不是总体开发成本上升，只改进attribute mesh也有长期重要价值
  - 单独做现有系统的改进，控制scope，利于meshshader基础设施的调试和能力走通
- MVP只走通绘制流程，不做特别的优化（预期可以接受有性能衰退）

## 具体工作

- 改进现有的graphsics shader builder体系和edsl相关内容，支持新的shader stage
  - 需要能够自动将现在的vertex shader逻辑桥接到mesh shader上
    - 将现有的vertexshaderbuilder变成抽象的，其中注入varying，registry相关的逻辑变成trait方法
      - 提供到vertex或者mesh/task的downcast
  - graphicsshaderprovider需要在build shader一开始就可以指定使用mesh还是传统vertex，理论上我们可以边build 边切换,但是这么做意义不大，成本很高
- attribute mesh system的改进：
  - 根据index buffer做meshlet划分，做额外一层meshlet的数据和寻址
    - 使用自己之前port的meshopt的meshlet分割算法
  - lod每一层独立处理，将meshlet数据视作一层新的类型的index
- 类比现有的device drawcommand list机制，需要实现一种device task shader dispatch command list机制
  - 需要生成dispatch的workgroup count，和一个辅助buffer
  - 辅助buffer记录具体task级别的工作任务：（meshlet range，scenemodel id， mesh id）
- 类比现有的midc，在支持的平台，会使用 multi_draw_mesh_tasks_indirect_count， 否则使用draw_mesh_tasks_indirect（降级）
  - multi_draw_mesh_tasks_indirect_count没有任何机制可以在shader中得知自己属于第几个dispatch，所以按照传统做法无法寻址到mesh和scenemodel id
    - 这也是为什么我们需要记录辅助buffer，所以这个问题的解法是每个task workgroup bump deallocate来取实际的task，task信息内包含了具体的工作任务
      - 所以这也说明这种方式相比midc的绘制，是不保序的，所以在需要绘制顺序控制的地方，还是需要用midc，不能用taskshader。  
      - 也有可能是保序的，但是比较模糊，spec没有说multi的case，而且即便保序，taskshader的dispatch大概率是没有这样的约定的。
        - <https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_mesh_shader.html#_rasterization_order>
  - 降级需要和midc降级类似做法，采用一个巨型的 draw_mesh_tasks_indirect
    - 但是因为消费的时候是直接bump deallocate，所以不需要做count二分查找和prefix sum，可以直接atomic add的直接生成这个draw_mesh_tasks_indirect
    - metal（现在）只支持draw_mesh_tasks_indirect， 和不支持midc一样
    - 如果task shader的dispatch workgroup count 上限不高，就很麻烦，（这涉及到host端的继续拆分，这直接break gpu driven），暂时不考虑这个case
      - 这个上限实际决定了mesh的最大支持大小
        - 4060 vk 上是 单dimension是65535，总体 4000k
        - m2 pro（及以上）metal 没有限制
- 具体mesh系统集成时，amplification rate需要可配置，提供充分多的配置项来做性能调优
