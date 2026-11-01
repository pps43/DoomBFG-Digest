# 实体、地图对象与呈现更新

本页解释 `.map` 中的文本对象如何成为运行时 `idEntity`，以及实体模拟结果如何进入渲染、声音和网络可见性体系。调用环境见[游戏运行时](10-game-runtime.md)，动态创建机制见[事件与脚本](12-events-script.md)。

## 关键源码

- `neo/idlib/MapFile.cpp`：地图实体、key/value、brush/patch 的语法树。
- `neo/d3xp/Game_local.cpp`：地图加载、实体生成、编号注册和 active 列表。
- `neo/d3xp/Entity.cpp`：实体 spawn、think、dormant、renderEntity 生命周期。
- `neo/d3xp/WorldSpawn.cpp`：第 0 个地图实体的全局初始化。

## 从 `.map` 到 `idMapFile`

`idMapFile::Parse` 在未忽略 region 时先尝试同名 `.reg`，未加载才读取 `.map`；随后循环调用 `idMapEntity::Parse`，并计算 geometry CRC。[neo/idlib/MapFile.cpp:725-771](../../_src/DOOM-3-BFG/neo/idlib/MapFile.cpp#L725-L771) 这个 CRC 随后用于校验各 AAS 文件是否与地图几何匹配。[neo/d3xp/Game_local.cpp:983-986](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L983-L986)

每个 `idMapEntity` 同时容纳 `epairs` 和 primitive。解析器把普通 token 对写成 key/value，把嵌套块识别为 brush 或 patch；第 0 个实体被当作 worldspawn 解析，并为其 primitives 预分配空间。[neo/idlib/MapFile.cpp:543-589](../../_src/DOOM-3-BFG/neo/idlib/MapFile.cpp#L543-L589) [neo/idlib/MapFile.cpp:589-639](../../_src/DOOM-3-BFG/neo/idlib/MapFile.cpp#L589-L639)

`idGameLocal::LoadMap` 解析地图后先加载 collision map，再清空实体数组、spawn/active 链、时间和相机等地图状态。它预留 `[0, MAX_CLIENTS)` 给客户端，从 `MAX_CLIENTS` 开始分配普通实体，并初始化 clip、游戏 PVS 和全部 AAS。[neo/d3xp/Game_local.cpp:877-905](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L877-L905) [neo/d3xp/Game_local.cpp:907-976](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L907-L976)

## Worldspawn 先行

`SpawnMapEntities` 强制把地图第 0 项以 `ENTITYNUM_WORLD` 生成，并验证结果是 `idWorldspawn`；其余实体按顺序经过 `InhibitEntitySpawn`、媒体预缓存和 `SpawnEntityDef`。[neo/d3xp/Game_local.cpp:3443-3489](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L3443-L3489)

`idWorldspawn::Spawn` 设置全局 `world` 和 gravity，尝试编译与地图同名的 `.script` 并延迟启动 `main`，随后启动 worldspawn 中所有 `call*` 指定函数。[neo/d3xp/WorldSpawn.cpp:57-98](../../_src/DOOM-3-BFG/neo/d3xp/WorldSpawn.cpp#L57-L98) 因而地图脚本入口发生在一批普通实体生成之前，但其 `DelayedStart(0)` 的实际执行仍由线程/事件调度决定。

## Decl、C++ 类与脚本三种连接

`SpawnEntityDef` 先用地图 `classname` 查 `idDeclEntityDef`，把 decl dict 作为 spawnArgs 默认值，再读取扩展键：

| spawnArgs 键 | 行为 | 失败处理 |
| --- | --- | --- |
| `spawnclass` | 通过 `idClass::GetClass` 查类型、`CreateInstance`，再 `CallSpawn` | 类型或实例创建失败则 warning + false |
| `spawnfunc` | 在 `program` 中查脚本函数，创建 `idThread` 并 `DelayedStart(0)` | 函数不存在则 warning + false |
| 两者都无 | 不生成对象 | warning + false |

表依据：[Game_local.cpp:3271-3299](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L3271-L3299) [Game_local.cpp:3315-3354](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L3315-L3354)。地图值覆盖 decl 默认值，不是反向覆盖。

基类 `idEntity::Spawn` 先 `RegisterEntity`，再解析 renderEntity、health、默认物理、origin/axis、model、refSound 和 script object。地图启动阶段的 `target*`/`guiTarget*` 不立即解析，而是投递 `EV_FindTargets`，避免引用尚未生成的目标。[neo/d3xp/Entity.cpp:475-495](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L475-L495) [neo/d3xp/Entity.cpp:560-601](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L560-L601)

## 身份、列表与销毁

`RegisterEntity` 为对象分配 `entityNumber` 和单调递增的 `spawnId`，将对象放入 `entities[]` 与 `spawnedEntities`，并复制最终 spawnArgs。多人模式还区分 replicated 与 `net_skip_replication` 的空闲编号区间；编号耗尽或 spawn count 溢出是游戏错误。[neo/d3xp/Game_local.cpp:3157-3194](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L3157-L3194)

`UnregisterEntity` 从 spawned 链和数组清除指针、复位 spawnId，并下调相应空闲编号搜索起点。[neo/d3xp/Game_local.cpp:3201-3224](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L3201-L3224) 因此跨帧或网络引用不能仅凭对象地址推断身份；实体编号与 spawnId 承担不同作用，网络细节见[网络会话](16-network-session.md)。

## Think、dormant 与 Present

基类 `idEntity::Think` 的核心顺序只有 `RunPhysics()` 后 `Present()`；派生类可扩展 think，但应保留其负责的物理和呈现更新语义。[neo/d3xp/Entity.cpp:862-870](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L862-L870) 游戏层的 active 链遍历顺序见[游戏 tic](10-game-runtime.md)。

允许 dormant 的实体在与玩家区域不连通并超过延迟后可跳过工作；尚未唤醒的实体还会做更精确的玩家 PVS 检查。首次唤醒后，仅“不在 PVS”不再足以让它重新休眠，只有区域关闭才会。[neo/d3xp/Entity.cpp:874-909](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L874-L909) 默认 `fl.neverDormant = true`，所以 dormant 是实体显式启用的优化而非全局规则。[neo/d3xp/Entity.cpp:435-440](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L435-L440)

`Present` 只在新游戏帧且带 `TH_UPDATEVISUALS` 时执行，随后清除该 active flag。无模型或 hidden 实体提前返回；否则首次调用 `AddEntityDef`，之后调用 `UpdateEntityDef` 更新 renderer 持有的 handle。[neo/d3xp/Entity.cpp:1453-1485](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L1453-L1485) `UpdateVisuals` 本身只触发 `UpdateModel` 和 `UpdateSound`，并不直接 draw。[neo/d3xp/Entity.cpp:1284-1287](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L1284-L1287)

呈现数据还用于可见性：`UpdatePVSAreas` 从变换后的 renderEntity bounds 求 PVS areas；若面积集合超过上限，则退化为以 origin 为中心、扩展 64 的 bounds，避免超出 `MAX_PVS_AREAS`。[neo/d3xp/Entity.cpp:1294-1314](../../_src/DOOM-3-BFG/neo/d3xp/Entity.cpp#L1294-L1314)

## 修改约束

- 新实体类型通常同时需要 decl 默认键、`spawnclass` 注册和 `Spawn` 中的资源/物理设置；只声明 classname 不足以实例化 C++ 对象。
- 启动期跨实体引用应使用已有延迟事件或等价的二阶段解析，避免依赖地图顺序。
- 修改 origin/model/skin 等呈现状态后应走 `UpdateVisuals` 和 render def 更新路径，不能只改 `renderEntity` 字段。
- 地图几何变化会使 CRC 改变，需要同步验证 collision/AAS 等派生资源，而不是只重读 `.map` 文本。

返回[游戏运行时](10-game-runtime.md)或[总入口](index.md)。
