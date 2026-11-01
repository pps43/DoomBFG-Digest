# 可见性、灯光与交互

本页深入[渲染前端](06-render-architecture.md)的场景筛选：哪些 area、实体和灯进入 view，以及 light/entity 组合如何产生光照与阴影几何。输出随后进入[后端分阶段绘制](07-render-frame.md)。

## 关键源码

- `neo/renderer/RenderWorld_defs.cpp`：实体/灯光派生数据与 area reference。
- `neo/renderer/RenderWorld_portals.cpp`：portal 洪泛、连通域、portal 状态和 view def 筛选。
- `neo/renderer/tr_frontend_addlights.cpp`、`tr_frontend_addmodels.cpp`：逐灯、逐模型构建 view 数据。
- `neo/renderer/Interaction.cpp`：静态 light/entity interaction 与缓存索引。

## 一个 view 的前端顺序

`R_RenderView` 先建立 view/projection/MVP 和朝外的 frustum planes，再调用 `FindViewLightsAndEntities`。之后依次等待上一批 shadow job、执行 `R_AddLights`、`R_AddModels`、加入世界 GUI、优化灯光列表、排序 ambient surfaces、递归生成 subview，最后才入队 draw-view 命令。[neo/renderer/tr_frontend_main.cpp:354-404](../../_src/DOOM-3-BFG/neo/renderer/tr_frontend_main.cpp#L354-L404) [neo/renderer/tr_frontend_main.cpp:406-423](../../_src/DOOM-3-BFG/neo/renderer/tr_frontend_main.cpp#L406-L423)

这意味着 mirror、远程相机和 GUI 3D view 不是后端的特殊补丁，而是同一套前端可见性与交互流程的额外 `viewDef`。

## Portal area 可见性

每个 view 先递增 `tr.viewCount` 使缓存的 `viewLight/viewEntity` 失效，清空两个 view 链和 area 屏幕矩形；随后用 `PointInArea` 找起始 area，建立用于“门后灯光”剔除的连通域，最后以 5 个 frustum planes 从 `vieworg` 洪泛 portal。[neo/renderer/RenderWorld_portals.cpp:660-690](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld_portals.cpp#L660-L690) [neo/renderer/RenderWorld_portals.cpp:710-715](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld_portals.cpp#L710-L715)

洪泛会沿可见 portal 累积 portal plane 与 screen rectangle；相机在世界外时改为把所有 area 加入 view。[neo/renderer/RenderWorld_portals.cpp:560-599](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld_portals.cpp#L560-L599) `r_singleArea` 和 `r_usePortals` 是调试分支，不应作为正常可见性规则。[neo/renderer/RenderWorld_portals.cpp:678-708](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld_portals.cpp#L678-L708)

portal 有 view、location、air 三类独立阻塞位；`SetPortalState` 在阻塞位变化时，从 portal 一侧重新洪泛相应的 connected-area 编号，并在录 demo 时记录状态变化。[neo/renderer/RenderWorld.h:260-270](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld.h#L260-L270) [neo/renderer/RenderWorld_portals.cpp:967-995](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld_portals.cpp#L967-L995) 因此门关闭不只隐藏一条 portal，也会改变后续 `AreasAreConnected` 的判断。

## 实体和灯如何进入 area

实体引用以模型 bounds 为基础；有回调而尚未完成昂贵动画/粒子计算时使用调用方给的保守 bounds，bounds 为空则不建立 area 引用。否则派生模型矩阵，并把模型 frustum 推入 BSP 树。[neo/renderer/RenderWorld_defs.cpp:162-195](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld_defs.cpp#L162-L195)

灯光先确定原点 area。只有带 prelight model、启用 `r_useLightPortalFlow` 且灯光投影会投阴影时，才通过 portal 洪泛限制引用；其他灯直接把 light frustum 推入 BSP 树。这个例外保留背向三角形的阴影投射可能性。[neo/renderer/RenderWorld_defs.cpp:540-568](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld_defs.cpp#L540-L568)

`R_AddSingleLight` 还会应用 view 抑制条件并求值 light shader registers。对普通非 fog/blend 灯，如果所有有效 stage 的 RGB 都不为正，就从本 view 移除，避免后续交互和阴影工作。[neo/renderer/tr_frontend_addlights.cpp:96-165](../../_src/DOOM-3-BFG/neo/renderer/tr_frontend_addlights.cpp#L96-L165)

## Interaction：灯光与表面的配对缓存

地图阶段可调用 `GenerateAllInteractions` 预建 `(lightDef, entityDef)` 表；若不调用，交互会按需动态生成。预建遍历每盏灯触及的 area 及其中实体，复用已有 interaction，否则 `AllocAndLink` 后调用 `CreateStaticInteraction`。[neo/renderer/RenderWorld.cpp:1524-1548](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld.cpp#L1524-L1548) [neo/renderer/RenderWorld.cpp:1550-1590](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld.cpp#L1550-L1590)

`AllocAndLink` 将 interaction 同时挂到 light 与 entity 的双向链，并以 `lightDef->index * interactionTableWidth + entityDef->index` 填表。[neo/renderer/Interaction.cpp:473-517](../../_src/DOOM-3-BFG/neo/renderer/Interaction.cpp#L473-L517) 空配对并不简单删除：`MakeEmpty` 把节点移到两链末尾，并在表中留下 `INTERACTION_EMPTY` 标记，供移动后的失效逻辑识别。[neo/renderer/Interaction.cpp:606-645](../../_src/DOOM-3-BFG/neo/renderer/Interaction.cpp#L606-L645)

静态构建先用模型 bounds 对 light frustum 粗剔除，再逐 surface 处理：受光材质生成裁剪后的 light triangle 索引；可投影且带 `silEdges` 的 surface 生成 shadow volume 索引，并写入 static vertex cache。未生成任何几何时转为 empty interaction。[neo/renderer/Interaction.cpp:663-707](../../_src/DOOM-3-BFG/neo/renderer/Interaction.cpp#L663-L707) [neo/renderer/Interaction.cpp:710-758](../../_src/DOOM-3-BFG/neo/renderer/Interaction.cpp#L710-L758)

这里有一个明确的失效约束：静态构建使用 `customSkin/customShader` 重映射材质，源码指出地图加载后改变它们而不 invalidate interaction 会得到错误结果。[neo/renderer/Interaction.cpp:696-703](../../_src/DOOM-3-BFG/neo/renderer/Interaction.cpp#L696-L703)

## 并行与最终裁剪

`R_AddLights` 和 `R_AddModels` 都可把单灯/单模型工作提交到 `frontEndJobList`，并在继续前 `Wait`；非并行配置执行相同函数的串行循环。[neo/renderer/tr_frontend_addlights.cpp:454-471](../../_src/DOOM-3-BFG/neo/renderer/tr_frontend_addlights.cpp#L454-L471) [neo/renderer/tr_frontend_addmodels.cpp:1003-1023](../../_src/DOOM-3-BFG/neo/renderer/tr_frontend_addmodels.cpp#L1003-L1023) 模型阶段生成静态/动态阴影后也等待任务完成，因为随后可能 unmap 阴影 index buffer。[neo/renderer/tr_frontend_addmodels.cpp:1025-1042](../../_src/DOOM-3-BFG/neo/renderer/tr_frontend_addmodels.cpp#L1025-L1042)

最后，若一盏 view light 没有 local、global 或 translucent interaction，其 local/global shadow 链会被清空；启用 shadow-surface scissor 时，再把阴影矩形限制到实际交互表面范围。[neo/renderer/tr_frontend_addlights.cpp:534-575](../../_src/DOOM-3-BFG/neo/renderer/tr_frontend_addlights.cpp#L534-L575)

## 修改时的检查点

- 改实体 bounds、灯投影或 transform 后，必须走对应 reference/derived-data 更新，不能只改游戏对象位置。
- 改 portal 阻塞语义时同时检查视图、声音/空气等使用方及 demo 记录；三类位不是同义词。
- 新动态材质或 skin 路径要触发 interaction 失效，尤其不能复用旧静态索引。
- 在 job 中新增的共享写入必须发生在 `Wait` 所保护的边界内，并考虑 shadow index buffer 的 unmap 时序。

返回[渲染架构](06-render-architecture.md)或[总入口](index.md)。
