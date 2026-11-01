# 术语与缩写速查

本表统一 Wiki 中容易混用的引擎术语。定义只覆盖本快照中的用法；详细条件、失败路径和精确源码引用以链接页面为准，不把同名通用技术概念直接套入 DOOM 3 BFG。

## 帧、模拟与并发

| 术语 | 本 Wiki 中的含义 | 不要混淆为 |
| --- | --- | --- |
| `idCommonLocal` / Common | 进程组合根，负责初始化、外层帧、绘制和关闭 | 某个具体游戏对象或独立通用库 |
| Session | 菜单、 lobby、加载、进入/退出游戏等会话状态编排 | `idGameLocal` 内的一张地图或一次比赛 |
| `idGameLocal` | DOOM 3 权威游戏状态与每 tic 模拟的实现 | 渲染系统或 SWF 菜单 |
| 外层 frame | `idCommonLocal::Frame` 的一次执行，`idLib::frameNumber` 随之递增 | 一个固定时长游戏 tic |
| game tic | 一次游戏模拟步进；一次外层 frame 可运行零个或多个 | GPU 帧或网络快照 |
| `usercmd` | 带客户端/服务端时间的输入命令，供游戏和 Classic 消费 | 已经完成的网络快照 |
| `gameReturn_t` | Game/Draw 工作单元向主线程返回的会话命令和呈现相关结果 | 可随时读写的无锁消息队列 |
| SMP | 本代码中 Game/Draw 与渲染后端等工作的重叠执行模式 | 所有子系统都线程安全或全部并行 |

依据：外层帧计数见 [common_frame.cpp:367-373](../../_src/DOOM-3-BFG/neo/framework/common_frame.cpp#L367-L373)，tic/Draw 与工作提交见 [common_frame.cpp:86-178](../../_src/DOOM-3-BFG/neo/framework/common_frame.cpp#L86-L178)，返回值构造见 [Game_local.cpp:2473-2488](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L2473-L2488)。详见[启动帧循环](03-startup-frame.md)与[游戏运行时](10-game-runtime.md)。

## 资源与渲染

| 术语 | 本 Wiki 中的含义 |
| --- | --- |
| VFS / `idFileSystem` | 统一 loose file、`.resources` 和生成缓存等资源查找的虚拟文件系统 |
| Decl / `idDeclManager` | 材质、实体、声音等命名声明的注册、惰性解析和复用层 |
| `.resources` | 带索引的资源容器，不等于操作系统目录或 Classic WAD |
| `idRenderWorld` | 游戏侧更新实体、灯、area 和可见性所用的场景定义接口 |
| `idRenderSystem` | 初始化渲染管理器、消费 command buffers 并驱动 GPU 的接口 |
| `viewDef` | 一次 view 的前端参数及相机/场景上下文 |
| `renderEntityDef` / `renderLightDef` | RenderWorld 中实体与灯的呈现状态，不是 gameplay entity 本身 |
| command buffer | Draw/前端生成的渲染命令流，后端在受控边界消费 |
| area / portal | 地图空间分区及其连接，用于可见性、交互和声音路径 |
| PVS | Potentially Visible Set；在相关路径中表示潜在可见集合，不等于最终 draw 列表 |
| Interaction | 灯光与实体表面的配对及缓存数据，供后续光照绘制使用 |

资源证据见 [FileSystem.cpp:2780-2844](../../_src/DOOM-3-BFG/neo/framework/FileSystem.cpp#L2780-L2844) 与 [DeclManager.cpp:1130-1164](../../_src/DOOM-3-BFG/neo/framework/DeclManager.cpp#L1130-L1164) [DeclManager.cpp:2365-2404](../../_src/DOOM-3-BFG/neo/framework/DeclManager.cpp#L2365-L2404)；渲染接口与命令缓冲见 [RenderWorld.h:273-335](../../_src/DOOM-3-BFG/neo/renderer/RenderWorld.h#L273-L335) 和 [RenderSystem.h:265-283](../../_src/DOOM-3-BFG/neo/renderer/RenderSystem.h#L265-L283)。详见[文件声明](05-files-declarations.md)、[渲染架构](06-render-architecture.md)与[可见性光照](09-visibility-lighting.md)。

## 实体、脚本与运动

| 术语 | 本 Wiki 中的含义 |
| --- | --- |
| `entityDef` / `idDict` | 从声明或 map entity 取得的键值参数集合，可继续选择 C++ 类和脚本对象 |
| `idEntity` | DOOM 3 游戏实体基类，连接 think、事件、物理、动画和 Present |
| dormant | 降低非活动实体模拟/呈现工作的状态机制，不等于销毁 |
| `idThread` | 脚本 VM 上协作式执行的线程对象，不是操作系统线程 |
| `idClip` / `idClipModel` | 游戏层空间索引与碰撞查询对象 |
| collision model / CM | `neo/cm` 管理的碰撞几何与查询，不等同于可视 render model |
| team | 主实体及绑定部件的组合运动单位，有专门的内部 clip 禁用和受阻恢复流程 |
| animator channel | 动画混合通道；本实现有 5 个 channel，每个最多混合 3 项 |
| `jointMods` | 对动画骨架 joint 的修改/覆盖数据，Walk IK 等路径会写入 |
| AAS | Area Awareness System，供 AI area 与路由查询使用的导航数据和运行时 |

脚本协作线程见 [Script_Thread.cpp:544-555](../../_src/DOOM-3-BFG/neo/d3xp/script/Script_Thread.cpp#L544-L555) [Script_Thread.cpp:659-688](../../_src/DOOM-3-BFG/neo/d3xp/script/Script_Thread.cpp#L659-L688)，Clip 链接与查询见 [Clip.cpp:630-660](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L630-L660) [Clip.cpp:1141-1207](../../_src/DOOM-3-BFG/neo/d3xp/physics/Clip.cpp#L1141-L1207)，动画与 AAS 的精确条件见[动画、IK 与 AI 导航](14-animation-ai.md)。实体生命周期见[实体与地图](11-entities-map.md)，脚本 VM 见[事件与脚本](12-events-script.md)。

## 持久化与网络

| 术语 | 本 Wiki 中的含义 |
| --- | --- |
| savegame | 通过对象登记和序列化恢复游戏对象图的持久化格式 |
| `idSerializer` | 位级读写辅助层，可被网络等路径复用；不自动提供 savegame 对象恢复 |
| render demo | 记录并重放 render/sound 命令的演示，不是完整游戏状态快照 |
| snapshot | host 向 client 发送的游戏状态视图，可按已确认 base 做 delta |
| base / ack | 某 peer 已确认的快照基线及确认信息，用于后续 delta 和重发管理 |
| lobby | 管理 peers、可靠消息和快照传输的会话传输/成员关系层 |
| prediction key | client 用来识别预测生成实体的键；本快照匹配后的完整接管代码未启用 |
| PVS handle / `visMask` | 网络路径中的可见性句柄与实体掩码字段；当前实体掩码均为 `~0U`，不能据此声称已按 PVS 过滤复制 |

对象图恢复见 [gamesys/SaveGame.cpp:162-193](../../_src/DOOM-3-BFG/neo/d3xp/gamesys/SaveGame.cpp#L162-L193)，render demo 边界见[持久化页](15-persistence.md#render-demo-记录什么)，快照和预测限制见[网络页](16-network-session.md)及其就近源码引用。

## UI、音频与 Classic

| 术语 | 本 Wiki 中的含义 |
| --- | --- |
| `windowDef` | 原生 `.gui` 的顶层窗口定义，由 `idWindow` 解析 |
| SWF | 本引擎实现的 Flash 资源/时间线/ActionScript **子集**运行时，不是完整 Flash 兼容层 |
| BSWF | `generated/` 下的 SWF 二进制缓存，带格式版本和源时间校验 |
| native function | 注册给 SWF 脚本调用的 C++ 函数，例如 `updateMenuDisplay` |
| sound shader | 描述样本列表、循环、空间化和音量等参数的声明 |
| emitter | sound world 中持有一个或多个逻辑 channel 的发声器 |
| 逻辑 channel | 记录 shader、样本和起止时间的播放状态；失去硬件 voice 后仍可存在 |
| hardware voice | 实际提交 XAudio2 buffers 的 voice；其停止和回收具有异步边界 |
| `.idwav` | 含格式、amplitude 和音频 buffers 的生成样本缓存 |
| `Globals` / `::g` | Classic 每个玩家的经典 DOOM 全局状态，由 `DoomLib::SetPlayer` 选择 |
| `DoomInterface` | 宿主驱动 Classic Startup/Poll/Tic/Frame/Shutdown 的桥接对象 |

GUI 与 SWF 证据见 [UserInterface.cpp:316-391](../../_src/DOOM-3-BFG/neo/ui/UserInterface.cpp#L316-L391) 和 [SWF_Load.cpp:130-144](../../_src/DOOM-3-BFG/neo/swf/SWF_Load.cpp#L130-L144)；逻辑/硬件声音分离见 [snd_emitter.cpp:154-165](../../_src/DOOM-3-BFG/neo/sound/snd_emitter.cpp#L154-L165) [snd_emitter.cpp:758-813](../../_src/DOOM-3-BFG/neo/sound/snd_emitter.cpp#L758-L813)；Classic 玩家状态与桥接见 [doomlib.cpp:140-154](../../_src/DOOM-3-BFG/doomclassic/doom/doomlib.cpp#L140-L154) [doominterface.cpp:68-127](../../_src/DOOM-3-BFG/doomclassic/doom/doominterface.cpp#L68-L127)。详见[GUI/SWF](17-ui-swf.md)、[音频](18-audio.md)和[DOOM Classic](19-doom-classic.md)。

返回[总目录](index.md)或[架构总览](01-overview.md)。
