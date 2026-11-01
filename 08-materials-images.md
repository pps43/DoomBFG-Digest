# 材质、图像与着色程序

本页解释[声明文本](05-files-declarations.md)如何变成可提交的纹理、材质阶段和 GLSL program；它们在[后端绘制顺序](07-render-frame.md)中被消费。

## 关键源码

- `neo/renderer/Material.cpp`：材质与 stage 解析、coverage 推导、寄存器求值。
- `neo/renderer/Image_load.cpp`、`BinaryImage.cpp`、`OpenGL/gl_Image.cpp`：源图读取、二进制缓存、GPU 上传和绑定。
- `neo/renderer/RenderProgs*.cpp`：内建着色器注册、GLSL 转换/链接、uniform 提交。

## 材质不是单张纹理

一个 `shaderStage_t` 同时保存 lighting 类别、draw state、颜色/alpha test、纹理阶段、vertex color 行为和可选的新式 shader stage；`newShaderStage_t` 则记录 vertex/fragment program、参数和最多 8 张 fragment 图像。[neo/renderer/Material.h:184-212](../../_src/DOOM-3-BFG/neo/renderer/Material.h#L184-L212)

`idMaterial::Parse` 从声明文本建立 lexer，经 `ParseMaterial` 解析后统计 ambient stage、识别 subview，再补全 coverage、content flag、editor alpha 和 sort 等派生属性。[neo/renderer/Material.cpp:2196-2213](../../_src/DOOM-3-BFG/neo/renderer/Material.cpp#L2196-L2213) [neo/renderer/Material.cpp:2221-2240](../../_src/DOOM-3-BFG/neo/renderer/Material.cpp#L2221-L2240)

coverage 未显式指定时：无 stage 的材质归为 translucent；存在非 ambient（光照交互）stage 时归为 opaque；否则根据首 stage 的目标混合状态决定 translucent 或 opaque。translucent 会自动带 `MF_NOSHADOWS`，其他 coverage 会添加 `CONTENTS_OPAQUE`。[neo/renderer/Material.cpp:2242-2272](../../_src/DOOM-3-BFG/neo/renderer/Material.cpp#L2242-L2272) 因而修改 blend 或 stage lighting 不只影响 draw call，也可能改变阴影与场景内容分类。

## 动态参数与 stage 生命周期

`ParseStage` 使用材质解析期临时数据 `pd->parseStages`，识别 `blend`、`map`、图像选项等 token；超过 `MAX_SHADER_STAGES` 或解析失败会将材质标记为 defaulted，而不是继续构造不完整阶段。[neo/renderer/Material.cpp:1134-1179](../../_src/DOOM-3-BFG/neo/renderer/Material.cpp#L1134-L1179) [neo/renderer/Material.cpp:1188-1196](../../_src/DOOM-3-BFG/neo/renderer/Material.cpp#L1188-L1196)

每帧动态材质值由 `EvaluateRegisters` 写入调用方提供的寄存器数组：先复制常量，再写入 time、实体局部 shader parms 与全局 parms，之后按表达式操作求值。[neo/renderer/Material.cpp:2467-2500](../../_src/DOOM-3-BFG/neo/renderer/Material.cpp#L2467-L2500) 这些寄存器是材质表达式与后端 uniform 数据之间的中间表示，不等同于实体脚本变量。

## 图像加载与 `.bimage` 缓存

`idImage::ActuallyLoadImage` 的正常路径是：

1. 渲染系统尚未初始化时直接返回；有 `generatorFunction` 时只调用生成器。
2. 非 production 模式读取源图或 cubemap 并推导格式；随后以 usage/cube 类型生成缓存名并尝试读取 `.bimage`。
3. 缓存满足 production 条件，或源时间戳有效且 texture type/format/color format 匹配时，采用缓存头；否则重新读取源图并生成缓存。
4. 分配 GL texture，逐级、逐 cube face 上传缓存中的像素。[neo/renderer/Image_load.cpp:266-303](../../_src/DOOM-3-BFG/neo/renderer/Image_load.cpp#L266-L303) [neo/renderer/Image_load.cpp:341-358](../../_src/DOOM-3-BFG/neo/renderer/Image_load.cpp#L341-L358) [neo/renderer/Image_load.cpp:408-426](../../_src/DOOM-3-BFG/neo/renderer/Image_load.cpp#L408-L426)

缓存路径为 `generated/images/<生成名>.bimage`，生成名中的括号和逗号会规范化为目录分隔。[neo/renderer/BinaryImage.cpp:442-446](../../_src/DOOM-3-BFG/neo/renderer/BinaryImage.cpp#L442-L446) 读取时校验 magic，并在非 production 模式拒绝源时间戳不匹配的缓存；cubemap 的图像块数为 mip 层数乘 6。[neo/renderer/BinaryImage.cpp:349-359](../../_src/DOOM-3-BFG/neo/renderer/BinaryImage.cpp#L349-L359) [neo/renderer/BinaryImage.cpp:369-395](../../_src/DOOM-3-BFG/neo/renderer/BinaryImage.cpp#L369-L395)

加载失败不是无限重试：普通 2D 图像读取失败会建立 8×8、单层的清空纹理后返回。[neo/renderer/Image_load.cpp:383-405](../../_src/DOOM-3-BFG/neo/renderer/Image_load.cpp#L383-L405)

`Bind` 对未加载图像会按需调用 `ActuallyLoadImage(true)`，源码明确标注此路径“not SMP safe”；正常游戏模式应避免在后端分离线程首次触发加载。[neo/renderer/Image_load.cpp:436-444](../../_src/DOOM-3-BFG/neo/renderer/Image_load.cpp#L436-L444) `PurgeImage` 删除 GL texture，并清除所有 texture unit 的绑定缓存，使下次 bind 真正重新绑定。[neo/renderer/OpenGL/gl_Image.cpp:458-467](../../_src/DOOM-3-BFG/neo/renderer/OpenGL/gl_Image.cpp#L458-L467)

## GLSL 的生成、链接与绑定

初始化时 `idRenderProgManager::Init` 将内建名称映射到 builtin index，并加载对应 vertex/fragment shader，包括 GUI、texture、interaction、environment 等路径。[neo/renderer/RenderProgs.cpp:69-95](../../_src/DOOM-3-BFG/neo/renderer/RenderProgs.cpp#L69-L95)

`LoadGLSLShader` 以 `renderprogs/*.vertex|pixel` 为输入，以 `renderprogs/glsl/*_{vertex|fragment}.glsl` 为输出。GLSL 缺失或 HLSL 更新时间更新时，执行 dead-code stripping 和 `ConvertCG2GLSL`，并把生成文件写入 `fs_basepath`；否则读取现有 GLSL。[neo/renderer/RenderProgs_GLSL.cpp:911-947](../../_src/DOOM-3-BFG/neo/renderer/RenderProgs_GLSL.cpp#L911-L947) [neo/renderer/RenderProgs_GLSL.cpp:953-975](../../_src/DOOM-3-BFG/neo/renderer/RenderProgs_GLSL.cpp#L953-L975) 这意味着开发期改 shader 可能写资源目录，移植时不能只准备只读 pak。

program 链接前绑定 vertex attribute location；链接失败会删除 program 并触发 `idLib::Error`。成功后按配置获取统一 uniform array 或逐参数 location，并将 `matrices_ubo` 绑定到 0 号 block binding。[neo/renderer/RenderProgs_GLSL.cpp:1163-1192](../../_src/DOOM-3-BFG/neo/renderer/RenderProgs_GLSL.cpp#L1163-L1192) [neo/renderer/RenderProgs_GLSL.cpp:1213-1265](../../_src/DOOM-3-BFG/neo/renderer/RenderProgs_GLSL.cpp#L1213-L1265)

`BindShader` 在 vertex/fragment 组合未变化时跳过重复绑定，否则对有效 GLSL program 调用 `qglUseProgram`。[neo/renderer/RenderProgs.cpp:366-377](../../_src/DOOM-3-BFG/neo/renderer/RenderProgs.cpp#L366-L377) `CommitUniforms` 则按配置一次上传 vertex/fragment uniform array，或逐个提交保存的 location。[neo/renderer/RenderProgs_GLSL.cpp:1119-1150](../../_src/DOOM-3-BFG/neo/renderer/RenderProgs_GLSL.cpp#L1119-L1150)

## 修改约束

- 新 stage 关键字需要同时考虑解析、派生 coverage/sort、后端 draw state 和资源引用，不能只让 lexer 接受 token。
- 修改源图或格式推导后，要验证 `.bimage` 时间戳和格式匹配路径；production 对缓存有效性的判断不同。
- 新增 GPU 资源时明确加载线程；`Bind` 的惰性加载是兼容路径，不应成为 SMP 正常帧的资源管理方案。
- 新 uniform 必须接入参数命名、GLSL 生成和 `CommitUniforms`，仅在 shader 源码中声明不会自动获得引擎值。

返回[渲染架构](06-render-architecture.md)或[总入口](index.md)。
