# 物理、Clip 与碰撞模型

本页解释[实体 Think](11-entities-map.md)中的物理求值、动态碰撞查询和地图碰撞数据。`idPhysics` 决定对象怎样运动，`idClip` 决定查询会碰到谁，`idCollisionModelManager` 执行底层几何测试；三者不是同一层。

## 关键源码

- `neo/d3xp/physics/Physics.*`、`Physics_*.*`：物理公共契约和不同运动策略。
- `neo/d3xp/Entity.cpp`：默认 clip model、team 物理求值和受阻处理。
- `neo/d3xp/physics/Clip.cpp`：动态 clip model 的空间链接与查询编排。
- `neo/cm/CollisionModel_*.cpp`：地图/模型加载及 translation、rotation、contact、contents 实现。

## 物理对象契约

`idPhysics` 统一暴露 clip model、contents/clip mask、bounds、`Evaluate`、网络插值、`UpdateTime`、冲量/力、rest 状态和 transform/velocity 接口。[neo/d3xp/physics/Physics.h:76-116](../../_src/DOOM-3-BFG/neo/d3xp/physics/Physics.h#L76-L116) [neo/d3xp/physics/Physics.h:119-145](../../_src/DOOM-3-BFG/neo/d3xp/physics/Physics.h#L119-L145) Static、Actor、Player、Parametric、RigidBody、AF 等实现共享此接口，但具体积分与约束属于各自实现，不能由基类推断为同一种求解器。

普通实体的 `GetPhysicsTimeStep` 返回 `gameLocal.time - previousTime`，而不是独立 wall-clock 计时；Player 可覆盖该函数。[neo/d3xp/Entity.cpp:2952-2959](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L2952-L2959) 因此物理时间与游戏 time group、补 tic 和帧率设置相关。

`InitDefaultPhysics` 按 spawnArgs 依次寻找显式 `clipmodel`、`mins/maxs` 或 `size` 生成的 box/cylinder/cone，最后尝试把视觉 model 当 collision model；`noclipmodel` 禁止后两类自动生成。无效 bounds/size 是游戏错误。[neo/d3xp/Entity.cpp:2488-2546](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L2488-L2546) 默认对象使用 `defaultPhysicsObj`；切换到自定义 `idPhysics` 时清 contacts、移除默认 clip、激活新对象、同步当前时间并设置 bind master。[neo/d3xp/Entity.cpp:2548-2575](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L2548-L2575)

## Team 求值与受阻边界

`idEntity::RunPhysics` 在未启用 `TH_PHYSICS` 时只更新动画控制器；team slave 不独立求值，由 master 处理整条 teamChain。[neo/d3xp/Entity.cpp:2603-2624](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L2603-L2624)

正常 team 流程为：

1. 保存每个部件的物理状态，并临时禁用 `!solidForTeam` 部件的 clip，避免 team 内部阻挡自身。
2. 逐部件调用 `Evaluate(timeStep, endTime)`；移动或强制更新时调用 `UpdateFromPhysics`，再更新动画控制器。
3. 重新启用 team clip。
4. 若某部件被阻挡，恢复它之前各部件的状态和视觉位置，对全 team 只做 `UpdateTime(endTime)`，恢复被推动者位置，并发出 team/part blocked 信号与事件。[neo/d3xp/Entity.cpp:2626-2674](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L2626-L2674) [neo/d3xp/Entity.cpp:2676-2709](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L2676-L2709)

这不是通用物理事务系统：它只实现当前 team 移动路径的保存、部分恢复和通知，具体碰撞响应仍由各 `idPhysics` 实现负责。

## Clip 的空间索引

地图加载后 `idClip::Init` 取得 `worldMap` 的 bounds，递归建立 clip sector 树，并创建一个默认 8 单位扩张 box trace model；shutdown 释放 sector 和缓存 trace model。[neo/d3xp/physics/Clip.cpp:769-816](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L769-L816)

动态 `idClipModel::Link` 先解除旧 sector 链接，根据 origin/axis 计算绝对 bounds，再为碰撞 epsilon 扩张后挂到所有相交 sector。[neo/d3xp/physics/Clip.cpp:630-660](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L630-L660) `SetPosition` 只解除旧链接并修改 origin/axis，本身不重新链接；调用方必须走后续 Link 更新路径，否则空间索引会停留在旧位置。[neo/d3xp/physics/Clip.cpp:529-535](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L529-L535)

`idClipModel::Handle` 可复用已有 collision handle，也可把缓存的 `idTraceModel` 转成 collision model；`renderModelHandle` 路径则使用单独的 render-model trace，而不是假设每个可视模型都有静态 CM handle。[neo/d3xp/physics/Clip.cpp:542-552](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L542-L552) [neo/d3xp/physics/Clip.cpp:1188-1195](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L1188-L1195)

## Translation 的两级查询

`idClip::Translation` 先对超大位移走专门检查，然后对 world 调用 `collisionModelManager->Translation`；若 world 在 fraction 0 立即阻挡则直接返回。否则以移动后的 trace bounds 查询 candidate clip models，对候选逐一执行 render-model trace 或 CM translation，并只保留 fraction 更小的结果，同时填 `entityNum` 与 clip id。[neo/d3xp/physics/Clip.cpp:1141-1169](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L1141-L1169) [neo/d3xp/physics/Clip.cpp:1171-1207](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L1171-L1207)

这使 `trace_t.fraction` 同时承担最近命中选择和提前终止条件。`passEntity`、content mask 和 clip mask 必须在同一查询语义下设置；忽略 world 的分支只在 `passEntity == ENTITYNUM_WORLD` 时使用。[neo/d3xp/physics/Clip.cpp:1154-1168](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L1154-L1168)

底层 manager 将加载地图、preload、load/setup model 与 Translation、Rotation、Contents、Contacts 分成独立 API，调用方不应把“找到 clip model”误当成“已经完成碰撞测试”。[neo/cm/CollisionModel.h:87-133](../../_src/DOOM-3-BFG/neo/cm/CollisionModel.h#L87-L133)

## 地图与模型缓存

`LoadMap` 在地图名和 file time 都未变时复用当前 collision map；否则 `FreeMap`、清空结构、建立共享 vertex/edge hash，并从 `idMapFile` `BuildModels`。[neo/cm/CollisionModel_load.cpp:3696-3744](../../_src/DOOM-3-BFG/neo/cm/CollisionModel_load.cpp#L3696-L3744) 这与[实体地图加载](11-entities-map.md)共用 `idMapFile` 时间信息，而非从渲染 `.proc` 反推碰撞。

`LoadModel` 先按名称复用 handle，再尝试 `generated/collision/<model>.bcm` 二进制缓存；缓存不可用时尝试文本 `.cm` 并写回二进制，最后尝试 ASE/LWO render model 转换。[neo/cm/CollisionModel_load.cpp:3865-3913](../../_src/DOOM-3-BFG/neo/cm/CollisionModel_load.cpp#L3865-L3913) preload manifest 中 `PRELOAD_COLLISION` 项会提前走同一 `LoadModel`。[neo/cm/CollisionModel_load.cpp:3663-3686](../../_src/DOOM-3-BFG/neo/cm/CollisionModel_load.cpp#L3663-L3686)

## 修改约束

- 新物理实现必须同时定义 `Evaluate`、时间同步、rest、transform、clip 和 save/restore 语义，不能只实现运动积分。
- 移动 clip model 后要同步 origin/axis、absBounds 与 sector link；单纯改字段会漏检或误检。
- 修改 map/collision 资源时验证时间戳、`.bcm` 缓存和 preload，不要把渲染模型与碰撞模型视作同一份数据。
- 新 team 运动逻辑需保留内部 clip 禁用/恢复、受阻通知和动画更新顺序，并测试 client prediction 分支。

返回[游戏运行时](10-game-runtime.md)或[总入口](index.md)。
