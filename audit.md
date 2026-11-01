# 覆盖与一致性审计

本报告记录 Wiki 的正向计划核对、从目录/入口/公共 API 反向盘点，以及停止补充正文后的独立对抗阶段。这里的“已覆盖”表示对应核心机制已有直接源码证据，不表示逐文件或逐函数穷尽。

- **源码快照：** `id-Software/DOOM-3-BFG`，`1caba1979589971b5ed44e315d9ead30b278d8b4`
- **审计日期：** 2026-10-31（Asia/Shanghai）
- **范围：** 第一方 Win32 引擎、DOOM 3 游戏层、Classic 宿主集成、核心资源/渲染/音频/网络路径与构建入口。
- **方法：** 同一上下文中先完成正文，再以源码和反向盘点为起点执行独立对抗阶段；未使用或声称使用独立代理审查。
- **结果：** 反向盘点 30 个关键对象，已覆盖 24、明确范围外 6、证据不足 0、遗漏 0；未发现未关闭的阻断问题或跨页中心行为冲突。

## 正向计划与页面树

`index.md` 中的 20 个主题页面均已存在并可从索引到达：构建与启动、idLib、资源声明、四个渲染页面、六个游戏机制页面、网络、GUI/SWF、音频、Classic 和术语表均有正文与就近证据。索引还链接本报告；没有以空章节、待写页面或孤立页面充当覆盖。

页面计划按实际运行关系组织，而不是按目录数量分摊篇幅。例如 `RenderWorld` 的场景定义与 `RenderSystem` 的命令消费分别解释，savegame、render demo 和 network snapshot 也没有被合并成一种“序列化”。入口与核心调用链见[总览](01-overview.md)、[帧循环](03-startup-frame.md)、[地图生命周期](10-game-runtime.md)和[Classic 集成](19-doom-classic.md)。

## 反向覆盖矩阵

以下对象来自根目录、`neo/` 与 `doomclassic/` 目录、解决方案项目、三个真实入口以及主要公共 API 的反向检查。数量是盘点对象状态，不是语义覆盖率。

| # | 反向盘点对象 | 状态 | Wiki 处置 |
| --- | --- | --- | --- |
| 1 | 根 README、发布数据与授权前提 | 已覆盖 | [总览](01-overview.md)、[构建](02-build-platform.md) |
| 2 | `doom3.sln`、七个项目及属性表 | 已覆盖 | [构建目标与配置继承](02-build-platform.md) |
| 3 | `base/renderprogs` 与 shader 资源路径 | 已覆盖 | [材质、图像与着色程序](08-materials-images.md) |
| 4 | `WinMain` 主进程入口 | 已覆盖 | [启动与帧循环](03-startup-frame.md) |
| 5 | `amplitude` 独立 `main` | 已覆盖 | [构建项目表](02-build-platform.md#解决方案与真实构建目标) |
| 6 | `idLib` 服务、内存、容器、SIMD、作业 API | 已覆盖 | [idLib 与并行作业](04-idlib-jobs.md) |
| 7 | `Common`、`EventLoop`、`common_frame` | 已覆盖 | [启动与帧循环](03-startup-frame.md) |
| 8 | `idFileSystem`、`.resources`、`idDeclManager` | 已覆盖 | [文件系统与声明](05-files-declarations.md) |
| 9 | `sys/win32` 核心适配及会话/存档平台边界 | 已覆盖 | [构建平台](02-build-platform.md)、[持久化](15-persistence.md)、[网络](16-network-session.md) |
| 10 | `idRenderSystem` / `idRenderWorld` 公共 API | 已覆盖 | [渲染架构](06-render-architecture.md)、[帧流水线](07-render-frame.md) |
| 11 | Material、Image、RenderProgs 管理器 | 已覆盖 | [材质、图像与着色程序](08-materials-images.md) |
| 12 | area/portal、light/entity interaction、前端 jobs | 已覆盖 | [可见性、灯光与交互](09-visibility-lighting.md) |
| 13 | `idGame`、`idSession`、`ExecuteMapChange` | 已覆盖 | [游戏运行时](10-game-runtime.md) |
| 14 | 地图实体生成、身份、Think/Present | 已覆盖 | [实体与地图](11-entities-map.md) |
| 15 | `idClass`、事件队列、脚本 program/VM/thread | 已覆盖 | [类系统、事件与脚本](12-events-script.md) |
| 16 | `idPhysics`、`idClip`、`idCollisionModelManager` | 已覆盖 | [物理、Clip 与碰撞](13-physics-collision.md) |
| 17 | Animator、Walk IK、动画 frame commands | 已覆盖 | [动画与 IK](14-animation-ai.md) |
| 18 | `idAAS`、AAS 文件/CRC、路由与无 AAS 分支 | 已覆盖 | [AI 导航](14-animation-ai.md#aas-文件与路由生命周期) |
| 19 | savegame、平台存储、render demo、`idSerializer` 边界 | 已覆盖 | [持久化](15-persistence.md) |
| 20 | usercmd、snapshot、direct lobby、LAN 查询、reliable | 已覆盖 | [网络与会话](16-network-session.md) |
| 21 | 原生 GUI、菜单桥接、SWF loader/VM/timeline | 已覆盖 | [GUI 与 SWF](17-ui-swf.md) |
| 22 | SoundShader、Sample、Emitter、XAudio2 voice/callback | 已覆盖 | [音频](18-audio.md) |
| 23 | `DoomLib` / `DoomInterface` 宿主桥接 | 已覆盖 | [Classic 集成](19-doom-classic.md) |
| 24 | `D_DoomMain`、WAD、Classic 声音与软件帧缓冲入口 | 已覆盖 | [Classic 主循环与适配](19-doom-classic.md#classic-主循环已被宿主取代) |
| 25 | JPEG、zlib、DXT、Timidity 等第三方算法内部 | 范围外 | 只分析主程序装配和资源边界，避免把依赖源码展开成第一方架构 |
| 26 | Steam、Bink、depth-fail 模板阴影商业实现 | 范围外 | 发布说明明确缺失；只保留公开缺口，不虚构实现 |
| 27 | 成就、排行、语音、完整个人资料等平台外围服务 | 范围外 | 非本次核心路径；不以接口存在推断完整服务行为 |
| 28 | 每一种 Decl 的完整字段与解析格式 | 范围外 | 覆盖管理器及材质/声音/实体代表路径，不声称逐类型穷尽 |
| 29 | 内置 UI 小游戏的完整玩法 | 范围外 | 覆盖宿主 GUI/SWF 机制，不展开独立游戏逻辑 |
| 30 | Classic 全玩法算法与条件网络实现 | 范围外 | 覆盖入口、宿主生命周期和共享服务；未把条件代码当作已验证联机能力 |

范围外项也已在[index 的范围与限制](index.md#范围与限制)中对读者披露。本次没有发现应覆盖却仍标为“证据不足”或“遗漏”的关键对象；运行期效果未实测另列于文末，不与源码覆盖状态混同。

## 关键结论台账

| ID | 规范化结论与适用条件 | 直接证据 | 页面位置 | 状态 |
| --- | --- | --- | --- | --- |
| K01 | 当前 Win32 项目把 Game-d3xp 与 Classic 静态链接进主程序；DLL 分支是条件编译的另一条路径 | [game-d3xp.vcxproj:32-43](../../_src/DOOM-3-BFG/neo/game-d3xp.vcxproj#L32-L43) [doomexe.vcxproj:520-545](../../_src/DOOM-3-BFG/neo/doomexe.vcxproj#L520-L545) | index、01、02、10、19 | 已验证 |
| K02 | 外层 frame 可运行零个或多个 tic；`RunGameAndDraw` 在启动新工作前锁存旧 `gameReturn_t` | [common_frame.cpp:86-178](../../_src/DOOM-3-BFG/neo/framework/common_frame.cpp#L86-L178) [common_frame.cpp:469-540](../../_src/DOOM-3-BFG/neo/framework/common_frame.cpp#L469-L540) | index、01、03、04、10 | 已验证 |
| K03 | SMP 后端读取已关闭的另一份 `idFrameData`；双缓冲不表示 GPU 无界异步 | [tr_local.h:527-544](../../_src/DOOM-3-BFG/neo/renderer/tr_local.h#L527-L544) [RenderSystem.cpp:598-620](../../_src/DOOM-3-BFG/neo/renderer/RenderSystem.cpp#L598-L620) [gl_backend.cpp:113-166](../../_src/DOOM-3-BFG/neo/renderer/OpenGL/gl_backend.cpp#L113-L166) | 01、03、06、07 | 已验证 |
| K04 | `OpenFileRead` 按 `fs_resourceLoadPriority` 改变目录与容器阶段顺序，两级查找均倒序 | [FileSystem.cpp:2780-2844](../../_src/DOOM-3-BFG/neo/framework/FileSystem.cpp#L2780-L2844) [FileSystem.cpp:2922-2937](../../_src/DOOM-3-BFG/neo/framework/FileSystem.cpp#L2922-L2937) | 05、19 | 已验证 |
| K05 | 未解析 Decl 在主线程惰性解析；无源才由管理器默认化，有源 `Parse` 的返回值不在此统一检查 | [DeclManager.cpp:1130-1164](../../_src/DOOM-3-BFG/neo/framework/DeclManager.cpp#L1130-L1164) [DeclManager.cpp:2365-2404](../../_src/DOOM-3-BFG/neo/framework/DeclManager.cpp#L2365-L2404) | 05、07、08 | 已修正 |
| K06 | savegame 只序列化已登记对象索引；未登记非空指针写成 null，而不是自动追加对象 | [Game_local.cpp:471-493](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L471-L493) [SaveGame.cpp:382-394](../../_src/DOOM-3-BFG/neo/d3xp/gamesys/SaveGame.cpp#L382-L394) | 15 | 已验证 |
| K07 | 恢复先创建完整对象图；`program.Restore` 失败返回 `false`，当前 `Common_load` 分支退出到标题 | [SaveGame.cpp:865-916](../../_src/DOOM-3-BFG/neo/d3xp/gamesys/SaveGame.cpp#L865-L916) [Game_local.cpp:1213-1250](../../_src/DOOM-3-BFG/neo/d3xp/Game_local.cpp#L1213-L1250) [Common_load.cpp:527-532](../../_src/DOOM-3-BFG/neo/framework/Common_load.cpp#L527-L532) | 15 | 已验证 |
| K08 | 当前实体 snapshot 的 `visMask` 为 `~0U`；不能由已计算的 PVS handle 推断复制已按 PVS 过滤 | [Game_network.cpp:343-414](../../_src/DOOM-3-BFG/neo/d3xp/Game_network.cpp#L343-L414) | 16 | 已验证 |
| K09 | prediction-key 接管实体的移动/`BecomeReplicated` 代码在 `#if 0` 中，当前启用分支只检查并继续 | [Game_network.cpp:736-783](../../_src/DOOM-3-BFG/neo/d3xp/Game_network.cpp#L736-L783) | 16 | 已验证 |
| K10 | `MoveToPosition` 在无有效 AAS/area 时只走 `NewWanderDir` 回退；该结论不外推为所有移动 API 的唯一行为 | [AI.cpp:1949-1989](../../_src/DOOM-3-BFG/neo/d3xp/ai/AI.cpp#L1949-L1989) | 14 | 已验证 |
| K11 | SWF 是带 VM 的资源子集；`RunTo` 跳过中间 `DoAction`，保留动作先入队后由 `RunActions` 执行 | [SWF_Sprites.cpp:96-167](../../_src/DOOM-3-BFG/neo/swf/SWF_Sprites.cpp#L96-L167) [SWF_SpriteInstance.cpp:350-405](../../_src/DOOM-3-BFG/neo/swf/SWF_SpriteInstance.cpp#L350-L405) [SWF_Render.cpp:109-112](../../_src/DOOM-3-BFG/neo/swf/SWF_Render.cpp#L109-L112) | index、17 | 已修正 |
| K12 | SWF 单次 Render 最多补 10 帧；时间在截断前推进，超额积压被丢弃 | [SWF_Render.cpp:76-131](../../_src/DOOM-3-BFG/neo/swf/SWF_Render.cpp#L76-L131) | 17 | 已修正 |
| K13 | 音频逻辑 channel 与硬件 voice 分离；Mute 可只释放 voice，FreeVoice 经 zombie 等异步停止 | [snd_emitter.cpp:154-165](../../_src/DOOM-3-BFG/neo/sound/snd_emitter.cpp#L154-L165) [snd_world.cpp:327-395](../../_src/DOOM-3-BFG/neo/sound/snd_world.cpp#L327-L395) [XA2_SoundHardware.cpp:422-465](../../_src/DOOM-3-BFG/neo/sound/XAudio2/XA2_SoundHardware.cpp#L422-L465) | 01、18 | 已验证 |
| K14 | Classic 在宿主帧边界切换，使用 35 Hz 和共享 usercmd；软件帧缓冲上传为 960×600 RGBA 图像 | [Common.cpp:1586-1650](../../_src/DOOM-3-BFG/neo/framework/Common.cpp#L1586-L1650) [common_frame.cpp:709-747](../../_src/DOOM-3-BFG/neo/framework/common_frame.cpp#L709-L747) | index、03、19 | 已验证 |
| K15 | Classic 保留自身逻辑但由宿主 `DoomInterface::Frame` 驱动；WAD 读取进入主 VFS，不是独立进程循环 | [doominterface.cpp:129-188](../../_src/DOOM-3-BFG/doomclassic/doom/doominterface.cpp#L129-L188) [w_wad.cpp:126-198](../../_src/DOOM-3-BFG/doomclassic/doom/w_wad.cpp#L126-L198) | 01、05、19 | 已验证 |
| K16 | `idThread` 是通过事件恢复的脚本协程，不是 OS 线程；事件参数按格式复制/重建 | [Script_Thread.cpp:544-555](../../_src/DOOM-3-BFG/neo/d3xp/script/Script_Thread.cpp#L544-L555) [Event.cpp:281-324](../../_src/DOOM-3-BFG/neo/d3xp/gamesys/Event.cpp#L281-L324) | 11、12、20 | 已验证 |
| K17 | GPL 快照缺少商业数据及 README 列出的三项功能；构建成功不等于完整可运行安装 | [README.txt:12-21](../../_src/DOOM-3-BFG/README.txt#L12-L21) [README.txt:35-52](../../_src/DOOM-3-BFG/README.txt#L35-L52) | index、01、02、19 | 已验证 |

## 对抗发现与修正

| 严重度 | 发现 | 处置 |
| --- | --- | --- |
| 重要 | Decl 段落曾把“有源类型解析失败”也概括为管理器统一默认化，与 `ParseLocal` 分支不符 | 已回到实现核对并改写[失败语义](05-files-declarations.md#findtype-是惰性解析边界)；K05 标记已修正 |
| 重要 | SWF 段落曾把 `DoAction` 简写为在 `RunTo` 内执行，且未说明 10 帧上限会丢弃超额积压 | 已按 `RunTo`、`DoAction`、`Render` 的真实顺序修正[K11/K12](17-ui-swf.md) |
| 一般 | 术语表曾有三处错误的脚本线程/存档路径与失效行号，总览曾有指向非目标实现的 RenderSystem 引用 | 已改为真实定义与调用位置，并重新执行路径、行号和内部链接检查 |
| 一般 | direct backend 的 fake 搜索结果容易被误读为没有真实服务器查询 | 已补充独立的 connectionless LAN 请求/响应路径，并明确未做跨主机实测 |

对抗阶段还专门检查了动态装配和替代路径：Decl allocator/folder 注册、`CLASS_DECLARATION` 类型图、`REGISTER_PARALLEL_JOB`、菜单 native functions、session/lobby 分发和 Classic 帧边界切换。对“所有、仅、自动、无锁、实时”等绝对化措辞进行了搜索，并优先复核其失败分支、条件编译和调用方；最终没有保留无来源的全局推断。

静态高风险结论 K01–K17 均已回源复核，没有未决的文档漂移或中心行为冲突。仍未验证的是运行期表现而非上述源码分支：旧工具链可构建性、商业数据下的完整启动、GPU/驱动差异、XAudio2 设备行为、旧存档兼容、跨主机发现与联机、Classic 条件网络和完整玩法。

## 机械检查与未执行命令

- 用 `git rev-parse HEAD` 确认上述快照；分析期间未修改源码仓库。
- 最终脚本检查全部 Markdown 源码链接的文件存在性、`#L` 行号范围与顺序，以及内部页面链接和标题锚点；结果均为 0 错误。
- 从 `index.md` 遍历内部页面，21 个内容页加本报告均可达，孤立页面为 0；代码围栏成对，15 个 Mermaid 块均保留图下或邻近证据。未安装 Mermaid 渲染器，因此没有把结构检查描述为渲染验证。
- 扫描占位章节和待办标记；正文保留的“占位”用于解释源码中的 `dummy` usercmd 字段，唯一 `FIXME` 匹配是持久化页对现有 smoke-particles 源码注释的引用，均不是 Wiki 待办。
- 未执行 `msbuild`、游戏、自动化测试或硬件/联机验证。当前 macOS 环境不具备该快照要求的 VS2010/Win32/DirectX SDK 组合，且没有商业游戏数据；[构建页](02-build-platform.md#可复现的构建步骤)中的命令来自解决方案与 README，未声称执行成功。

返回[总入口](index.md)。
