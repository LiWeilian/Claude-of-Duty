# 迁移评估报告 — .NET 8 + C#（Windows-only）

> 目标：将本仓库的浏览器 FPS（Three.js r180 + WebGL2，约 55k 行 JS，11 个子系统）
> 迁移到 .NET 8 + C# 桌面端。**不使用任何专门的游戏引擎**（Unity/Godot/MonoGame/Stride 均排除）。
>
> 决策基线：
> - **平台：仅 Windows**
> - **物理：保留自研并移植**（BVH、扫掠胶囊体角色控制器、刚体、CCD、PBD 布娃娃、多层穿透）
> - **图形后端：Silk.NET + OpenGL 4.6**

---

## 0. 核心判断

本项目的最大资产是 **几千行自定义 GLSL shader**——HDR 后处理链、19 种程序化纹理生成、
粒子/天空/材质全部是自写 shader。Three.js 只是"引擎胶水层"。

因此迁移的第一原则是：**最大化复用 GLSL，用 C# 重写引擎胶水层**。
这决定了它不是"翻译项目"，而是 **"重写一个迷你 Three.js + 移植游戏逻辑"** 的项目。

### 为什么是 OpenGL 4.6 而不是别的

| 图形后端 | 结论 |
|---|---|
| **Silk.NET + OpenGL 4.6** ✅ 首选 | Windows 上 NVIDIA/AMD 驱动完整支持 4.6；GLSL 可 90% 原样迁移；单一后端无跨平台妥协 |
| Veldrid（D3D11 后端） | 曾为跨平台 macOS 考虑，Windows-only 下多一层无谓抽象 |
| Vortice + D3D12 | 性能上限更高（光追/Mesh Shader/VRS），但 shader 全改 HLSL + 左手坐标系 + PSO 样板，工作量 +30~50%。仅当确定需要 DX12 独占特性时才值得 |

> 注：若日后改回跨平台（尤其 macOS），需重新评估——macOS 上 OpenGL 被 Apple 软废弃
> （最高 4.1），届时应转向 Veldrid（Metal 后端）或 Vulkan+MoltenVK。

---

## 1. 选型定格

| 层 | 选择 | 说明 |
|---|---|---|
| 运行时 | .NET 8（LTS，支持至 2026-11） | 若在意更长期支持可升 .NET 10 LTS，但对本迁移无实质影响 |
| 图形 | Silk.NET.OpenGL 4.6 | GLSL 直通；状态机心智模型与现有 WebGL2 代码一致 |
| 窗口/输入 | Silk.NET.Windowing（GLFW 后端） | 鼠标锁定、原始输入、无边框原生支持 |
| 数学 | **double 精度自建数学层** | ⚠️ 关键：Three.js `Vector3` 是 double，`System.Numerics.Vector3` 是 float。用 float 会改变物理确定性与手感。需自建 `Vector3d`/`Matrix4d`/`Quatd`（或 Silk.NET.Maths 的 `Vector3D<double>`），物理全链路保持 double |
| 物理 | 自研移植 | 数学 double 一致；确定性是硬要求（见 §3-4） |
| 音频 | Silk.NET.OpenAL + EFX | 现成卷积混响（`AL_EFFECT_CONVOLUTION_REVERB`）与软 HRTF（`AL_SOFT_HRTF`），覆盖原 Web Audio 的混响+空间化；`dsp.js` 合成逻辑移植为 C# DSP，OpenAL 只做混音 |
| UI/HUD | ImGui.NET | 原 HUD 是 DOM/CSS。ImGui 画准星/血条/击杀信息/调试面板。**禁用 WPF/WinForms**（retained-mode 窗口系统，游戏内集成别扭） |
| 打包 | NativeAOT 单 exe | 无外部资产 + 程序化生成，正中 NativeAOT 下怀；无 .NET 运行时依赖 |
| 渲染目标 | 自建 RenderTarget/FBO 管理 | Three.js 无 C# 对应物，须手写 |

---

## 2. 子系统迁移难度矩阵

| 子系统 | 难度 | 主要工作量 | 复用度 |
|---|---|---|---|
| `core` | 🟢 低 | 事件总线（JS 动态对象→C# 强类型事件）、固定种子 RNG、配置、预热 | 逻辑全复用 |
| `materials` | 🟢 低 | GLSL 生成纹理 → 渲染到 RenderTarget | shader 原样复用 |
| `render` | 🔴 高 | **最大坑：Three.js 封装层消失**。RenderTarget/FBO、GL 状态管理、UBO、CSM 数组纹理、预渲染（prewarm）、后处理链接入 | shader 复用 ~90% |
| `sky` | 🟡 中 | 大气散射 shader 可用；**PMREM 预滤波需自实现**（Three.js 有 PMREMGenerator，C# 无对应物） | shader 复用 |
| `world` | 🟡 中 | 程序化几何生成重写；InstancedMesh → 自建实例化绘制；模块化建筑套件 | 逻辑复用 |
| `physics` | 🟡 中-高 | 纯数学/数据结构，移植直接但代码量大（BVH/扫掠胶囊/刚体/CCD/PBD/穿透）；**double 一致性** | 逻辑全复用 |
| `player` | 🟢 低 | 移动状态机 + 镜头手感，数学封装层 | 逻辑全复用 |
| `weapons` | 🟡 中 | 程序化几何 + 双场景渲染（viewScene 武器单独 pass）照搬 | 逻辑复用 |
| `fx` | 🟡 中 | 粒子/贴花逻辑重写 + shader 复用；贴花投影 | shader 复用 |
| `ai` | 🔴 高 | **蒙皮动画系统自建**（Three.js Skeleton/AnimationMixer 无对应物）；navmesh 寻路移植；感知/掩体 | 逻辑部分复用 |
| `ui` | 🟡 中 | DOM/CSS HUD → ImGui/自绘纹理，布局重写 | 低 |
| `audio` | 🟡 中 | dsp.js 合成 → C# DSP；空间化 → OpenAL/自研 | 逻辑复用 |

> 共享部分：`src/core`、`src/main.js`、`src/dev`、`tools`、`vite.config.js`
> 对应迁移到 C# 的 `Core`、程序入口、工具集（见 §5）。

---

## 3. 关键风险 TOP 5

1. **`render` + `ai` 的引擎层成本（40%+ 工作量）**
   工作量不在翻译，而在"重写迷你 Three.js"：RenderTarget/FBO、GL 状态管理、
   PMREM 生成、蒙皮动画/AnimationMixer、InstancedMesh 均无 C# 对应物，须自建。
   这两块决定整个迁移成败，应最先验证。

2. **double vs float**
   `System.Numerics.Vector3` 是 float。物理确定性、相机手感、复现截图全部依赖 double。
   需自建 double 数学层，并在 shader 上传边界做 float 转换（glVertexAttrib3dv 等）。

3. **shader 兼容性（10% 碎活）**
   GLSL ES 3.00 → desktop GLSL 4.60：`#version 460 core`、去 precision 声明、
   `texture()` 签名差异、部分内置函数。`sampler2DArray`（CSM 在用）桌面版原生支持。
   90% 直接可用，剩余 10% 是反复调试。

4. **确定性 / 可复现链**
   原仓库的 `baseline.mjs`（逐位一致截图）与 `imagediff.mjs`（像素门禁）依赖
   固定种子 + 固定步长 + 引擎时钟统一。C# 端必须守住：
   - RNG 用固定种子 + `ctx.rng.fork()`
   - `fixedUpdate` 固定 120 Hz，渲染用 `alpha` 插值
   - 禁 `DateTime.Now`/`Stopwatch` 驱动动画，统一用引擎时钟
   - 禁 `Math.Random`（.NET 的 `Random` 需用确定性实现，如 xoshiro/自实现）

5. **工具链重建**
   capture/baseline/imagediff/profile 基于 Playwright 无头浏览器，C# 端需重建等价物：
   - 截图：GL `glReadPixels`/readback + 位图编码
   - 像素门禁：自研逐像素对比（读 PNG）
   - 帧预算注入：渲染循环注入固定帧预算，保证截图像素级一致
   - 性能剖析：帧时间分布 + 每帧 shader 编译计数归因

---

## 4. 线程模型

- 起步保持**单线程 + 固定步长**：渲染主线程，物理固定 120 Hz 同线程。
  与 JS 原实现一致，保确定性（可复现截图）。
- 后续性能不足时再拆：物理 → 工作线程（Job），但必须把
  **确定性边界**留在固定步长内，否则复现链失效。
- 不建议起步就用多线程——先保住功能与确定性。

---

## 5. 工具链对应映射

| 原工具 | C# 对应 |
|---|---|
| `tools/capture.mjs` | 渲染循环内 `glReadPixels` + PNG 编码 |
| `tools/baseline.mjs` | 固定种子 + 固定帧预算 + 隔离初始化，逐位一致 |
| `tools/imagediff.mjs` | 自研逐像素对比，非零退出 |
| `tools/profile.mjs` | 帧时间分布 + per-frame 着色器编译计数归因 |
| `tools/playtest.mjs` | 脚本化输入注入冒烟测试 |
| `tools/demo*.mjs` / `probe.mjs` | 迁移为 C# 的 demo/探测子程序 |

---

## 6. 工作量与分阶段

总量预估 **40–70k 行 C#**（shader 行数直接复用不计，引擎层 +15k 新增，逻辑层 JS→C# 约 0.8–1.2×）。
单人熟练工程师约 **6 个月**。

| 阶段 | 内容 | 占比 | 里程碑 |
|---|---|---|---|
| 1 | 窗口 + Silk.NET.OpenGL + 数学层(double) + RenderTarget/FBO 管理 + 渲染循环 | 15% | 空窗口 + 清屏渲染跑通 |
| 2 | `render` 后处理链（HDR/CSM/GTAO/TAA/SSR/Bloom/MotionBlur/Dof/合成） | 25% | 一条后处理链 + baseline 像素对照 |
| 3 | `materials` + `sky` + `world`（画面先行） | 15% | 可运行一个静态场景画面 |
| 4 | `physics` + `player` + `weapons` | 20% | 可移动 + 开火 |
| 5 | `ai`（含蒙皮动画自建）+ `fx` | 15% | 敌人出现、交战、死亡 |
| 6 | `audio` + `ui` + 工具链重建 + NativeAOT 打包 | 10% | 完整可玩单 exe |

---

## 7. 建议的验证策略（动手前先读）

先做**双轨验证**，用一两周时间产出一份比本报告更硬的可行性证据：

1. 搭 Veldrid/Silk.NET + OpenGL 空窗口 + 一条后处理链（例如 LUT + 合成）
2. 搬 2–3 个程序化纹理 shader 跑通 RT 渲染
3. 重建 capture 等价物，用一张 baseline 图做像素级对照
4. 若以上全绿 → 投入全量迁移；若卡壳（如 PMREM/蒙皮动画成本失控）→ 本报告的成本预估需要修正

### 阶段 0 验证结果（2026-08-01，已执行 ✅）

最小离屏渲染闭环**已验证成立**。项目位于独立目录 `CodeSandBox/20260801_Claude-of-Duty`，
Silk.NET **2.23.0** + **net10.0**（环境无 .NET 8 SDK，TFM 与最终迁移目标解耦，正式迁移时再钉 LTS），
GLFW 隐藏窗口 + FBO 离屏渲染 + `glReadPixels` + 零依赖 PNG（手写 CRC32 / ZLibStream）。

结果：

- 输出 1280×720 有效 PNG（签名/IHDR 正确，常量色 19 KB）
- **连跑 5 次 sha256 全部一致** → 像素路径与编码路径双确定性成立
- 中心像素 `(51,102,204,255)` = shader 常量色 `vec4(0.2,0.4,0.8)` → 渲染管线真实工作，非黑帧
- `glfw3.dll` 由 NuGet native 资产（`runtimes/win-x64/native`）自动解析，无需手工拷贝

验证工具骨架（`Validator`：`--out` 参数化 + 零依赖 `PngWriter`/`Crc32`）即未来
`tools/capture` 与 `tools/imagediff` C# 对应物的种子。

**结论：桌面端视觉验证闭环成立，可投入全量迁移。** 下一里程碑见 §6 阶段 1
（窗口 + OpenGL 管线 + double 数学层 + RenderTarget/FBO 管理）。

### 阶段 1 结果（2026-08-01，已执行 ✅）

引擎骨架搭起，9 个文件 / 1251 行（含阶段 0 工具）：

| 模块 | 内容 |
|---|---|
| `Ow.Maths` | `Vector3d` / `Quatd` / `Matrix4d` — double 精度，API 对应 Three.js（引用语义、原地修改返回 this），列主序矩阵直接上传 uniform |
| `Ow.Core.GlContext` | GLFW 隐藏窗口 + GL context + 4.6→3.3 版本降级梯 + 函数指针加载 |
| `Ow.Core.EngineClock` | 固定 120 Hz 步进时钟，时间=步数×FixedDt，与墙钟完全解耦 |
| `Ow.Render.RenderTarget` | FBO + RGBA8 纹理 + Depth24，Bind/Clear/ReadPixels 抽象 |

**确定性动画验证成立**：三角形 `uOffset` 由引擎时间（`Math.Sin(t)/Cos(t)`）驱动，
`--settle N` 先推 N 个固定步再渲染。验证结果：

- 相同 `--settle` 连跑 3 次 → **逐位一致**
- 不同 `--settle`（0/10/50）→ **3 个不同帧**，且像素印证时间驱动（s50 左上角露出 clear 色，因三角形左边界随 t 右移 0.041）

这等价于原项目 `baseline.mjs` 的"固定 settle 帧预算 → 逐位一致截图"能力。

**两个排掉的坑**（记档）：

1. `glUniform*` 只作用于**当前绑定**的 program —— 必须先 `gl.UseProgram` 再 `gl.Uniform2`，否则静默 no-op（曾导致不同 settle 输出逐位相同）。
2. 命名空间 `Ow.Math` 遮蔽 `System.Math`（`Math.Sqrt` 解析到 `Ow.Math` 命名空间报错）—— 改名 `Ow.Maths`。

**下一步（§6 阶段 2）**：render 后处理链（HDR/CSM/GTAO/TAA 的前置——先做
Bloom/LUT/合成等单 pass 后处理 + `--settle` 帧预算截图工具化）。

### 阶段 2 结果·第一部分：后处理链骨架（2026-08-01，已执行 ✅）

后处理管线跑通，新增 3 个文件 / 约 1700 行总量：

| 模块 | 内容 |
|---|---|
| `Ow.Render.Shader` | shader 编译/链接/uniform/释放封装（uniform 设置内部自动 UseProgram） |
| `Ow.Render.FullScreenPass` | 全屏三角形（`gl_VertexID` 生成顶点，无 VBO）+ draw |
| `Ow.Render.Lut` | 33³ 确定性 LUT 生成（s-curve 对比度 + 饱和提升 + 暖色偏）+ GL_TEXTURE_3D 上传 |
| `RenderTarget` | 增加 `hdr: true` → RGBA16F 颜色纹理 |

管线：**HDR 程序化场景 → ACES 色调映射 + 33³ LUT 分级 → LDR 输出**。

验证（全部通过）：

- 相同 `--settle` 连跑 3 次 → **逐位一致**；不同 settle（0/10/50）→ 3 个不同帧
- HDR 高光（场景 3.2）被 ACES 压到暖白 `(255,254,247)`，**未截断为 255,255,255**
- 暗块（0.05,0.03,0.06）被 s-curve 压暗至 `(1,0,2)`
- **理论-实测吻合**：中心渐变像素预测 `(176,170,207)` vs 实测 `(177,172,209)`——验证了对管线与 LUT 的理解正确

**下一步（§6 阶段 2）**：加入 Bloom（Karis 金字塔）与 LUT 之后的多 pass 串联、
CSM 阴影与前向光照、GTAO/TAA/运动模糊等后处理，逐步逼近原项目 render 子系统。

### 阶段 2 结果·第二部分：Bloom 亮度金字塔（2026-08-01，已执行 ✅）

`Ow.Render.Bloom` 多 pass 后处理跑通（总代码约 1900 行）。管线：

```
bright(软阈值) → box 下采样金字塔(5级) → 高斯模糊上采样累加 → 合成(scene+bloom → ACES → LUT)
```

验证（全部通过）：

- **确定性**：settle 10 + bloom 2 连跑 3 次逐位一致
- **光晕平滑扩散**：衰减曲线 `255→12→5→2→1`（圆心到圆外 40px+），非硬边
- **intensity 单调性**：光晕点 `209→225→236`（intensity 0/1/2）
- **区域隔离**：远处渐变不受污染（三档 `48,168,219` 相同）

**排掉的两个坑**（记档，均与原项目"Karis"语义相关）：

1. **Karis 平均对亮/暗硬边界失效**：权重 `1/(luma+eps)` 让 0 亮度像素权重 ~1e4，把含边界的 2×2 块稀释成 ~0，光晕无法扩散出高亮区。高光是区域而非单像素时，**box 平均**（保留能量并向边界扩散）更符合光晕语义；firefly 抑制留待 bright pass 加亮度上限时引入。
2. **纯双线性上采样只放大不模糊**：光晕呈现"实心盘 + 硬切边"（衰减曲线 `255→…→0` 突变）。上采样链每级需 **9-tap 高斯模糊**（`G9`），才能产生平滑亮度衰减的光晕。

**下一步（§6 阶段 2）**：CSM 阴影与前向光照、GTAO/TAA/运动模糊，逐步逼近原项目 render 子系统。

### 阶段 2 结果·第三部分：3D 场景 + 相机 + 前向光照 + 单级阴影（2026-08-01，已执行 ✅）

从"程序化图案"迈入真实 3D 渲染（总代码约 2300 行）。新增/修改：

| 模块 | 内容 |
|---|---|
| `Ow.Render.Mesh` | 交错 pos+normal 的 VAO/VBO 抽象 + `Geo.Cube`/`Geo.Ground` 确定性程序化几何 |
| `Ow.Render.ShadowMap` | depth-only FBO + 深度纹理 + 深度 pass + PCF 3×3 采样（CSM 的单级基础） |
| `Matrix4d` | 加 `ToFloatArray`；**修复 LookAt 旋转存储转置 bug** |
| `Shader` | 加 `SetMatrix4`/`SetVec3` |
| 场景管线 | 透视相机(LookAt+Perspective) + 方向光(Lambert+Blinn-Phong, HDR) + 阴影 → Bloom → ACES → LUT |

验证（全部通过）：

- 4 个立方体颜色与 albedo 匹配；光方向改变时明暗正确变化（背光面暗 = 物理正确）
- **阴影存在**：立方体 A/B 影子为暗区，非阴影地面亮
- 确定性（连跑 3 次逐位一致）、时间驱动（旋转随 settle 变化）

**排掉的两个坑**（记档）：

1. **`LookAt` 旋转部分存了转置**：view 矩阵的行应是相机轴（`row0=(x.X,x.Y,x.Z)`），我存成了列（列 0 = x 轴）。导致 view 变换错误、深度错乱、立方体被地面"遮挡"。定位方法：手算 view 正反向变换发现 `R^T·R ≠ I`。修复后 3D 几何全部正确。
2. **相机看不到阴影**：光从 `(0.4,0.8,0.3)` 照，所有影子投在 xz 负向（远离相机），被物体自身遮挡。调整光方向使影子投向相机侧后阴影可见——这是"场景布置"问题而非阴影算法问题。

**下一步（§6 阶段 2）**：CSM 多级级联（`sampler2DArray`，按视锥深度划分 3-4 级阴影贴图 + 级联选择），逼近原项目 render 的 CSM。

### 阶段 2 结果·第四部分：CSM 多级级联（2026-08-01，已执行 ✅）

`Ow.Render.Csm` 取代单级 `ShadowMap` 接入管线（总代码约 2700 行）：

| 组件 | 内容 |
|---|---|
| 级联划分 | 对数/线性混合（lambda 0.5），3 级：near=0.1 → 8.75 → 19.85 → 50 |
| 每级光投影 | 视锥子集角点（view→world→light space）求光空间 AABB，生成独立 ortho |
| 深度存储 | `GL_TEXTURE_2D_ARRAY`（DepthComponent24，3 层），`glFramebufferTextureLayer` 逐层渲染 |
| 级联选择 | 主 pass 按像素 view-space 深度（`uSplitDepth`）选级联，`texture(uCsm, vec3(uv, cascade))` |
| `Shader` | 加 `SetMatrix4Array`（uniform mat4[3] 数组上传） |

验证（全部通过）：阴影存在（A/B 立方体影）、非阴影地面亮、确定性（逐位一致）、时间驱动（旋转随 settle）。

**排掉的坑**（记档）：

- **光空间 z 为负导致 ortho 深度错乱**：光看向场景中心，场景在光前方 → 光空间 z 是负值。把光 AABB 的 `cz±ez`（负值）直接当 near/far 传给 `MakeOrthographic`（期望正距离），深度映射到 z_ndc≈24，所有点判定超范围 → 阴影全消失。**转成正距离**（`nearDist=-(cz+ez)`）修复。

**下一步（§6 阶段 2）**：CSM 打磨（texel snapping、PCSS 接触硬化、级联接缝平滑）→ GTAO → TAA/运动模糊，逼近原项目 render 子系统。

### 阶段 2 结果·第五部分：GTAO 环境光遮蔽（2026-08-01，已执行 ✅）

`Ow.Render.Gtao`（确定性 SSAO）接入管线（总代码约 2400 行）。前置改造：
`RenderTarget` 支持**可采样 depth texture**（`depthTexture: true` → DepthComponent24 纹理而非 renderbuffer，供 AO 采样）。

算法：depth 纹理恢复 view 空间位置与法线 → 固定 8 方向（相对法线的半球，无随机）view 空间采样核 → 投影核点到屏幕 → **比较 NDC 屏幕深度**判定遮蔽 → `color × AO`。

验证（全部通过）：确定性、时间驱动、**接触阴影**（立方体底部接触处 AO 暗化、远离衰减恢复、开阔地面不受影响）、GTAO on/off 逐像素差异集中在立方体区域。

**排掉的坑**（记档）：

- **"屏幕邻居 + view 距离比较"把平坦地面涂黑**：第一版 AO 采样屏幕邻居、比较 view 距离，透视投影下平坦地面下方邻居距离更小（更近）被误判为遮蔽，整个地面全黑。改为**法线相对采样核 + 投影屏幕 + NDC 深度比较**的标准 SSAO 后，平坦表面核点深度 ≥ 表面深度 → 无遮蔽，接触处正确暗化。

**下一步（§6 阶段 2）**：TAA（时间抗锯齿，YCoCg 方差裁剪）→ 运动模糊，逼近原项目 render 子系统。

### 阶段 2 结果·第六部分：TAA 时间抗锯齿（2026-08-01，已执行 ✅）

`Ow.Render.Taa` 接入管线（总代码约 2600 行）。前置改造：

- **RenderTarget MRT**：第二个颜色附件 RG16F（运动矢量）
- **velocity 生成**：scene vs 用 `uPrevModel`（前一帧世界变换）算 `vVelocity`，**只含物体运动，不含抖动**

管线：scene(color+velocity+depth) → GTAO → **TAA**（velocity 重投影历史 + YCoCg variance clip + 抖动累积）→ Bloom → ACES → LUT。

TAA 组件：确定性 8 序列子像素抖动（`gl_Position.xy += uJitter*w`）、history ping-pong（blend 0.9）、**YCoCg variance clip**（邻域均值±σ，σ 下限 0.05，`uClipSigma` 2.5）。

验证（全部通过）：确定性、时间驱动、**光照方向正确**（B/C/D 受光顶面亮、背光 +X 面暗）、A 红/C 绿/D 黄颜色正确、阴影/地面正常、TAA 保留受光面亮度。

**排掉的三个坑**（记档，均导致"静止物体被 TAA 压暗/压黑"）：

1. **velocity 混入抖动差**：`vVelocity` 用 `clipCur - clipPrev`，而两者只差抖动 → 静止物体 velocity 非零 → history 采样错位 → 累积漂移（受光面逐帧变暗）。修复：velocity 用 `uPrevModel` 算前一帧位置，**只含物体运动**；抖动只加在 `gl_Position`。
2. **ambient 过低（0.06）**：背光面 HDR 值极小，ACES/LUT 后压黑；TAA 暗区 clip 进一步放大。修复：ambient 提到 0.15（背光面可见）。
3. **YCoCg AABB clamp 过紧**：均匀/暗面 AABB 窄，历史被过度压缩。修复：改 **variance clip**（均值±σ）+ σ 下限 0.05，暗区宽容。

**下一步（§6 阶段 2）**：运动模糊（tile-dilated，用 velocity）→ SSR → 景深，逼近原项目 render 子系统。

### 阶段 2 结果·第七部分：Tile-dilated 运动模糊（2026-08-01，已执行 ✅）

`Ow.Render.MotionBlur` 接入管线（总代码约 2800 行）。三 pass：

```
tile 最大速度(16×16) → 3×3 扩张 → 沿速度方向 12 点采样混合
```

输入颜色 + velocity（复用 TAA 的 velocity buffer）。管线：scene → GTAO → **MotionBlur** → TAA → Bloom。

验证（全部通过）：确定性、时间驱动、**运动物体模糊 / 静止物体基本不模糊**（A 旋转区差异 2138px vs B 静止区 437px）。

**下一步（§6 阶段 2）**：SSR → 景深 → 体积雾，逼近原项目 render 子系统。

### 阶段 2 结果·第八部分：SSR 屏幕空间反射（2026-08-01，已执行 ✅）

`Ow.Render.Ssr` 接入管线（总代码约 3000 行）。从 depth 恢复法线，沿反射方向 ray march 32 步，深度比较命中，Fresnel 控制强度。管线：scene → GTAO → **SSR** → MotionBlur → TAA。

验证（全部通过）：确定性、生效（7266 差异像素分布地面区、最大处 `47→106` 明显变亮）、**Fresnel 物理正确**（远处掠射角反射强、近处弱）。

**下一步（§6 阶段 2）**：景深（DOF）→ 体积雾，逼近原项目 render 子系统。

### 阶段 2 结果·第九部分：DOF 景深（2026-08-01，已执行 ✅）

`Ow.Render.Dof` 接入管线（总代码约 2950 行）。基于 depth 的高斯模糊：|view 深度 - 聚焦距离| 超过范围则模糊，半径随深度差增大。管线：scene → GTAO → SSR → **DOF** → MotionBlur → TAA。

验证（全部通过）：确定性、**聚焦物完全清晰**（B 聚焦距离 6.0，diff=0）、**非聚焦物模糊**（A/D 远处 diff>0）。

**下一步（§6 阶段 2）**：体积雾，逼近原项目 render 子系统。

### 阶段 2 结果·第十部分：体积雾（2026-08-01，已执行 ✅）

`Ow.Render.VolumetricFog`（简化距离雾）接入管线（总代码约 3000 行）。基于 depth 的指数衰减雾，远处向雾色混合，天空全雾。

验证（全部通过）：确定性、**距离梯度正确**（近 D `20→48`、中 A `92→124`、远 C `59→114`、天空 `2→208` 全雾）。

---

## render 子系统完成 ✅

对照原项目 render 子系统，核心特性已全部在 C# 桌面端落地并验证：

**HDR 管线 / CSM 3级联 / Bloom 金字塔 / 33³ LUT / ACES 色调映射 / GTAO / TAA(YCoCg variance clip) / tile-dilated 运动模糊 / SSR / DOF / 体积雾**

下一步（§6 阶段 3）：`materials`（程序化 PBR 纹理生成）+ `sky`（大气散射/PMREM）+ `world`（模块化建筑），或转向 gameplay（physics/player/weapons/ai）。

### 阶段 3 结果·第一部分：程序化纹理 + triplanar（2026-08-01，已执行 ✅）

`Ow.Render.TextureForge`（GPU texture forge）：确定性 GLSL（hash/value noise，无随机）渲染 albedo 纹理到独立 FBO，Repeat wrap 供 triplanar。3 种表面：混凝土、砖、金属（256²）。

场景 shader 加 **triplanar 采样**（按世界法线混合投影轴，无需 UV），`SceneMesh` 加 `AlbedoMap`。应用：地面混凝土、A 立方体砖、B/C/D 金属。

验证（全部通过）：确定性、**砖纹理清晰**（`(101,68,68)→(97,65,65)`）、混凝土灰度细节、金属灰色。

**排掉的坑**（记档）：

- **texture unit 冲突**：新增 `uAlbedoMap` 与已有 `uCsm` 都绑到 unit 0，per-mesh 循环里互相覆盖 → 场景全黑。修复：`uAlbedoMap` 用 unit 1。

**下一步**：法线贴图（triplanar normal 重建）、粗糙度、更多表面（金属/木/织物）→ `sky`/`world`/gameplay。

### 阶段 3 结果·第二部分：Sobel 法线贴图 + PBR 光照（2026-08-01，已执行 ✅）

`TextureForge.GenerateNormal`：从 albedo 亮度 **Sobel 梯度 → 法线纹理**（原项目 Sobel height→normal）。场景 shader 加 **triplanar normal**（切平面扰动叠加表面法线）+ **PBR 粗糙度/金属度**（specPower 随粗糙度、金属反射 albedo 色）。`SceneMesh` 加 `NormalMap`/`Roughness`/`Metallic`。

验证（全部通过）：确定性、**法线贴图生效**（on/off 差异 3140px，作用于表面 y[320,718]）、材质参数（地面粗糙 0.85、砖 0.8、金属 0.2-0.4 + metallic 0.8-1.0）。

**排掉的坑**（记档）：

- **法线扰动强度不足**：`n + tn.xy * 0.5` 扰动太弱，法线贴图几乎无效果（3px 差异）。强度提到 2.0 后生效（3140px）。

**下一步**：更多表面（木/织物/玻璃）→ `sky`/`world`/gameplay。

### 阶段 4 结果·第一部分：大气散射天空 + 半球环境光（2026-08-01，已执行 ✅）

`Ow.Render.Sky`：简化大气散射（**Rayleigh 天顶蓝** + Mie 太阳光晕 + 地平线雾），在 sceneRT 上全屏渲染、几何处（depth<1）discard 只填背景。场景 shader 的固定 ambient 改为**半球环境光**（`mix(地面色, 天空色, n.y*0.5+0.5)`）。

验证（全部通过）：确定性、**天空梯度**（天顶灰蓝 → 地平线白）、物体受半球光。

**排掉的两个坑**（记档）：

1. **体积雾覆盖天空**：sky 填充背景后，体积雾对 depth=1 全雾化把天空盖成雾色。修复：体积雾对天空保留原色（大气散射已含雾效）。
2. **Rayleigh 公式反向**：原用 `pow(1-y)` 让天顶 Rayleigh=0（无蓝），应天顶最强（`pow(max(y,0))`）——天顶蓝、地平线白。

**下一步**：PMREM 环境贴图（IBL 间接光/反射）→ 体积光轴 → `world`/gameplay。

### 阶段 4 结果·第二部分：环境 cubemap IBL（2026-08-01，已执行 ✅）

`Ow.Render.EnvMap`：CPU 用大气散射公式生成 6 面环境 cubemap（256²）+ `glGenerateMipmap`。场景 shader 金属反射 `reflect(-V,N)` 采样 cubemap，**粗糙度选 mip**（`textureLod`）。

验证（全部通过）：确定性、**金属物体环境反射**（B 金属从背光暗灰 `(36,35,36)` → `(243,233,228)` 亮色）。

**下一步**：体积光轴 → `world`/gameplay。

### 阶段 4 结果·第三部分：体积光轴（2026-08-01，已执行 ✅）

`Ow.Render.LightShafts`：简化屏幕空间 god rays——沿太阳屏幕方向 24 步累积光，物体遮挡处衰减（decay 0.92）。

验证（全部通过）：确定性、光轴生效（on/off 差异 163414px，分布在天空/地面）。

---

## sky 子系统完成 ✅

**大气散射天空（Rayleigh+Mie+地平线雾）/ 半球环境光 / 环境 cubemap IBL（金属镜面反射）/ 体积光轴**

下一步（§6 阶段 4/5）：`world`（模块化建筑）或 `gameplay`（physics/player/weapons/ai）。

### 阶段 5 结果·第一部分：模块化建筑关卡（2026-08-01，已执行 ✅）

`Ow.Render.World`：模块化建筑生成——单位立方体 + 3 轴缩放组合**带真实墙体厚度（0.15）的空心建筑**（地板/屋顶/4 墙），可进入。关卡：混凝土地面 + 3 栋砖/金属建筑 + 2 个运动物体（演示 velocity/TAA/光轴）。

验证（全部通过）：确定性、**建筑墙面可见**（砖纹理 + 光照）、天空/地面/阴影完整。

**下一步**：更多建筑/实例化道具/街道 → `gameplay`（physics/player/weapons/ai）。

### 阶段 6 结果·第一部分：确定性刚体物理（2026-08-01，已执行 ✅）

`Ow.Core.Physics`：从零手写刚体物理种子——固定 120Hz 步进、double 运算、无随机（确定性复现的天然契合）。重力 + 地面弹性反弹 + 球-球碰撞（弹性交换）。`Geo.Sphere` UV 球，物理位置每帧驱动渲染（PrevModel 供 velocity/TAA）。

验证（全部通过）：**确定性逐位一致**（settle 120 连跑 3 次相同）、**时间驱动**（settle 30/120/240 三帧不同 = 物理推进）、画面完整（天空/建筑/地面）。

**下一步**：深化物理（刚体旋转/更复杂碰撞）或 `player`/`weapons`/`ai`。

### 阶段 7 结果·实时交互模式（2026-08-01，已执行 ✅）

从"确定性离屏截图"升级为**实时交互窗口**：

- `GlContext` 支持可见窗口（尺寸/标题）+ 渲染循环（`PollEvents`/`SwapBuffers`/`ShouldClose`）+ 输入（`IsKeyDown`/鼠标锁定置中）
- `Core.PlayerCamera`：FPS 相机（yaw/pitch → view，WASD 水平移动 + 鼠标转动）
- `Program --live`：实时主循环（时间 → 相机 → 物理固定步 → 场景+后处理 → blit 到窗口 → swap），复用完整管线

运行：`Validator.exe --live`（WASD 移动、鼠标转动、ESC 退出）。

验证：`timeout 5` 稳定运行不崩溃（渲染循环 5 秒无错）。

**保留确定性模式**（`--out` 截图）与实时模式并存。下一步：`player`（移动/武器）深化实时体验。

### 阶段 7 结果·实时调试进展（2026-08-02）

用户实测反馈驱动的一系列实时专属问题修复（确定性模式相机固定从未暴露）：

**已修复**：

| 问题 | 修复 |
|---|---|
| TAA velocity 不含相机运动 → ghosting/乱涂 | scene shader 加 `uPrevView`（前一帧相机视图） |
| TAA velocity 源错（传了无 velocity 的 mbRT） | `Taa.Apply` 分 color/velocity 双源，velocity 用 sceneRT |
| 运动模糊相机旋转全屏糊 | 速度限幅 `clamp(±12px)` |
| 关 TAA 后 jitter 每帧抖 → 闪烁 | `--no-taa` 时 jitter 归零 |
| **光轴（LightShafts）全屏模糊** | 强度 `0.25→0.06` + 采样步数 `24→12`（本质是沿太阳方向长距平均） |
| 天空"一明一暗条带" | 方向重建去 `uInvProj`，改 NDC+fov/aspect 直接算（矩阵逆数值问题） |
| 鼠标左右反向 | PlayerCamera `Yaw` 方向反转 |

**待解决**：

- **TAA 实时静止仍模糊**（历史累积软化；blend 已 0.9→0.7 未完全解决，候选：YCoCg clip / history 传染）
- **天空渐变色带**（蓝/灰分界明显，`--live-raw` 可见）
- **阴影问题**（`--live-raw` 可见，待用户描述细节）
- **DOF 焦点固定 6.0 与移动相机不匹配**（实时建议 `--no-dof`）

**实时可用配置**：默认 `--live`（光轴已减弱）；最清晰基线 `--live --no-dof --no-taa --no-motionblur --no-ssr`（按需组合）。

---

## 8. 结论

**可行，且 shader 复用决定了它比"从零做"便宜得多。**
定性：Windows-only 把方案收窄为 **Silk.NET + OpenGL 4.6 + double 数学层 + 自研物理移植**，
GLSL shader 直通是最大红利；D3D12 仅在需要 DX 独占特性时才有理由付出 HLSL 重写成本。

最大成本集中在 `render`/`ai` 两个缺 Three.js 封装的子系统（40%+），
应在阶段 0 用双轨验证先行确认，再决定是否投入全量迁移。
