# DOOM 3 BFG 技术 Wiki

这是 `id-Software/DOOM-3-BFG` 的中文源码导航，面向需要修改、调试或移植引擎的技术人员。重点解释实际调用链、状态与数据所有权、关键失败路径和模块边界，而不是逐目录复述文件名称。

- **源码快照：** `1caba1979589971b5ed44e315d9ead30b278d8b4`
- **主要范围：** 第一方 Win32 引擎、DOOM 3 游戏层、Classic 集成、资源/渲染/音频/网络与构建路径。
- **证据形式：** 就近的仓库相对源码链接，均带 1 起始行号；图表和关键流程附实现依据。
- **质量记录：** [覆盖与一致性审计](audit.md)包含正反向盘点、关键结论台账、对抗检查及未验证事项。

## 从这里开始

1. [架构总览](01-overview.md)：先建立进程、资源、游戏、渲染、声音和 Classic 的整体地图。
2. [构建、产物与 Win32 平台边界](02-build-platform.md)：确认 VS2010、DirectX SDK、静态库和商业数据前提。
3. [启动、帧循环与线程交接](03-startup-frame.md)：理解 Common 如何把外层帧拆成 tic、Draw 和后端渲染。
4. 按修改目标进入下面的子系统页面；跨页名词可在[术语表](20-glossary.md)速查。

## 页面树

- [01 · DOOM 3 BFG 架构总览](01-overview.md)
  - [02 · 构建、产物与 Win32 平台边界](02-build-platform.md)
  - [03 · 启动、帧循环与线程交接](03-startup-frame.md)
  - [04 · idLib 基础层与并行作业](04-idlib-jobs.md)
  - [05 · 虚拟文件系统、资源容器与声明](05-files-declarations.md)
  - [06 · 渲染架构：场景定义与前后端边界](06-render-architecture.md)
    - [07 · 渲染帧流水线：命令缓冲、SMP 与后端阶段](07-render-frame.md)
    - [08 · 材质、图像与着色程序](08-materials-images.md)
    - [09 · 可见性、灯光与交互](09-visibility-lighting.md)
  - [10 · 游戏运行时：Session、Game 与地图生命周期](10-game-runtime.md)
    - [11 · 实体、地图对象与呈现更新](11-entities-map.md)
    - [12 · 类系统、事件与脚本 VM](12-events-script.md)
    - [13 · 物理、Clip 与碰撞模型](13-physics-collision.md)
    - [14 · 动画、IK 与 AI 导航](14-animation-ai.md)
    - [15 · 存档、恢复与 Render Demo](15-persistence.md)
    - [16 · 网络、快照与会话](16-network-session.md)
  - [17 · 原生 GUI、菜单与 SWF 运行时](17-ui-swf.md)
  - [18 · 音频：Sound Shader、Emitter 与 XAudio2](18-audio.md)
  - [19 · DOOM Classic 的嵌入式第二运行时](19-doom-classic.md)
- [20 · 术语与缩写速查](20-glossary.md)
- [审计 · 覆盖、关键结论与验证边界](audit.md)

## 按任务选择路径

| 任务 | 推荐阅读顺序 |
| --- | --- |
| 搭建或恢复旧构建环境 | [02 构建](02-build-platform.md) → [03 启动](03-startup-frame.md) → [05 文件声明](05-files-declarations.md) |
| 修改渲染或材质 | [06 渲染架构](06-render-architecture.md) → [07 帧流水线](07-render-frame.md) / [08 材质图像](08-materials-images.md) / [09 可见性](09-visibility-lighting.md) |
| 修改玩法或角色 | [10 游戏运行时](10-game-runtime.md) → [11 实体地图](11-entities-map.md) → [12 事件脚本](12-events-script.md) → [13 物理](13-physics-collision.md) / [14 动画 AI](14-animation-ai.md) |
| 修改存档或 demo | [10 游戏运行时](10-game-runtime.md) → [15 持久化](15-persistence.md) → 对照[18 音频](18-audio.md)的声音恢复边界 |
| 修改多人协议 | [03 启动帧循环](03-startup-frame.md) → [10 游戏运行时](10-game-runtime.md) → [16 网络会话](16-network-session.md) |
| 修改菜单或 HUD | [17 GUI/SWF](17-ui-swf.md) → [03 启动帧循环](03-startup-frame.md) → [06 渲染架构](06-render-architecture.md) |
| 修改 Classic | [19 Classic](19-doom-classic.md) → [03 启动帧循环](03-startup-frame.md) → [05 文件声明](05-files-declarations.md) / [18 音频](18-audio.md) |

## 关键架构结论

- 当前 Win32 配置把 DOOM 3 游戏层与 Classic 静态链接进主程序；不要默认游戏 DLL 或 Classic 外部进程。[项目证据](02-build-platform.md#解决方案与真实构建目标)
- 外层引擎 frame 与游戏 tic 不是同一计数；SMP 下 `RunGameAndDraw` 的返回值属于上一轮工作。[时序证据](03-startup-frame.md)
- RenderWorld 的场景状态、前端交互和 RenderSystem 后端命令是不同层；实体 Present 不是直接 GPU draw。[边界证据](06-render-architecture.md)
- savegame 的对象图恢复、render demo 的命令重放和网络 snapshot 是三种不同机制。[持久化](15-persistence.md) / [网络](16-network-session.md)
- SWF 是资源与 ActionScript 子集运行时；Classic 则是独立游戏逻辑通过桥接嵌入同一宿主帧循环。[UI](17-ui-swf.md) / [Classic](19-doom-classic.md)

## 范围与限制

根 README 明确说明发布不包含游戏数据；需要另行取得并遵守原 EULA。Steam 集成、Bink 视频和 depth-fail 模板阴影后端功能也不在发布内，因此本 Wiki 不会把相关调用点描述为完整商业实现。[README.txt:12-21](https://github.com/id-Software/DOOM-3-BFG/blob/1caba1979589971b5ed44e315d9ead30b278d8b4/README.txt#L12-L21) [README.txt:35-52](https://github.com/id-Software/DOOM-3-BFG/blob/1caba1979589971b5ed44e315d9ead30b278d8b4/README.txt#L35-L52)

第三方库只分析其与主程序的装配边界，不逐行讲解 JPEG、zlib、DXT、Timidity 等算法；不分析外部移植 fork，也不假设未随仓库发布的 Steam/Bink 实现。成就、排行、语音和完整个人资料等平台外围服务，以及逐种 Decl、内置 UI 小游戏和 Classic 条件网络代码不作独立展开。大型仓库采用核心路径优先的有边界覆盖，未决项和排除原因见[审计报告](audit.md)。

本次仅做静态源码分析：未执行 Windows 构建，未启动游戏，未运行自动化测试，未实测 GPU、XAudio2、存档兼容或多人联机。文档中的命令来自项目文件与 README，不表示已在当前 macOS 环境执行成功。
