# OVERWATCH — 引擎契约

**每个代理在写代码之前必须通读本文件。它是唯一的协调机制。**

目标：一款浏览器 FPS，其*视觉和手感质量*足以比肩现代《使命召唤》。
WebGL2 + Three.js r180，无外部美术资产——所有纹理、网格、动画和音频
都在加载时程序化生成。

## 硬性规则

1. **你拥有自己的目录。永远不要编辑目录之外的文件。** 其他每个目录都由别的
   代理拥有，你的编辑要么被覆盖，要么破坏他们的工作。
2. **绝不 import 其他子系统的模块。** 运行时获取：
   `const fx = ctx.get('fx')`。这是并行工作之所以安全的原因。
3. **不新增任何 npm 依赖。** 只用 `three`。无 CDN 拉取、无外部图片/HDR/
   模型/音频文件——游戏必须完全离线可运行。
4. **游戏逻辑和视觉效果中禁用 `Math.random()`。** 使用 `ctx.rng`
   （见 `src/core/rng.js`），或使用你保留的 `ctx.rng.fork()`。
   截图可复现性依赖于此。
5. **每帧零分配。** 在 `init()` 中预分配向量、矩阵和数组并复用。
   `update()` 里出现 `new THREE.Vector3()` 就是 bug。
6. **用完即释放。** 几何体、材质、纹理和渲染目标都要在 `dispose()` 中释放。
7. `npm run build` 必须通过，且你的改动后 `node tools/capture.mjs` 必须能
   产出一帧。如果你弄坏了启动流程，所有人都没法干活了。

## 子系统接口

```js
export class MySystem {
  static id = 'mysystem';       // 唯一；他人通过它找到你
  static deps = ['render'];     // 必须在你之前初始化的 id

  async init(ctx) {}            // 构建资源；可以 await
  fixedUpdate(h, ctx) {}        // 可选，120 Hz，确定性游戏逻辑
  update(dt, ctx) {}            // 可选，每帧一次
  lateUpdate(dt, ctx) {}        // 可选，在所有 update() 之后
  resize(w, h, ctx) {}          // 可选
  dispose() {}                  // 可选
}
```

`ctx` 提供：`scene`、`camera`、`viewScene`、`viewCamera`、`canvas`、
`config`、`events`、`input`、`time`、`rng`、`get(id)`、`peek(id)`、`has(id)`。

- `scene` / `camera` — 世界。`viewScene` / `viewCamera` — 第一人称武器，
  单独绘制，因此永远不会穿墙。
- `time` — `{ elapsed, raw, dt, fixed, alpha, scale, frame }`。使用 `alpha`
  在物理步进之间插值渲染变换。
- `config.q` — 当前生效的质量预设（见 `src/core/config.js`）。遵守
  `q.taa`、`q.gtao`、`q.ssr`、`q.volumetrics`、`q.shadowMapSize`、
  `q.particleBudget`、`q.decalBudget`。永远不要超出预算。

## 归属地图

| id | 目录 | 拥有 |
|---|---|---|
| `render` | `src/render/` | WebGLRenderer、HDR 管线、全部后处理、CSM 阴影、最终合成 |
| `materials` | `src/materials/` | 程序化 PBR 纹理生成、共享材质库、三平面/细节映射 |
| `sky` | `src/sky/` | 物理天空、日/月、昼夜、IBL/环境贴图生成、体积雾与光轴 |
| `world` | `src/world/` | 关卡几何、模块化建筑套件、道具、场景布景、静态碰撞网格 |
| `physics` | `src/physics/` | 宽相位、射线检测、角色控制器碰撞、刚体、布娃娃、穿透 |
| `player` | `src/player/` | 移动状态机、镜头手感、冲刺/滑铲/翻越/侧身、生命值 |
| `weapons` | `src/weapons/` | 武器网格、第一人称模型骨架、ADS、后坐力、晃动、摆动、换弹与检视动画、弹道 |
| `fx` | `src/fx/` | GPU 粒子、枪口闪光、曳光弹、命中特效、贴花、烟雾、血迹、弹壳 |
| `ai` | `src/ai/` | 敌方角色、导航、感知、掩体选择、战斗行为 |
| `ui` | `src/ui/` | HUD、准星、命中标记、伤害指示、弹药、击杀信息、菜单 |
| `audio` | `src/audio/` | 合成武器/动作音效、空间化、混响、遮挡、混音 |

共享部分，归负责人所有（请勿编辑）：`src/core/`、`src/main.js`、
`src/dev/`、`tools/`、`vite.config.js`。

## 跨子系统事件

通过 `ctx.events` 发出与监听。载荷是普通对象。标准事件集：

| 事件 | 载荷 | 发出方 |
|---|---|---|
| `weapon:fire` | `{ weapon, origin: Vector3, dir: Vector3, seed }` | weapons |
| `weapon:reload` | `{ weapon, phase: 'start'\|'magout'\|'magin'\|'end' }` | weapons |
| `weapon:shell` | `{ position, velocity }` | weapons |
| `bullet:impact` | `{ point, normal, surface, incident, damage }` | physics |
| `bullet:tracer` | `{ from, to, speed }` | weapons |
| `damage:dealt` | `{ target, amount, headshot, killed, point }` | ai / physics |
| ↳ | 意为 *对 `target` 造成的伤害*。`target` 是本地玩家时（`'player'`、player 系统，或任何 `isPlayer === true` 的对象）——画命中标记前要过滤掉它。伤害由其目标的监听器应用，发出方绝不重复应用。 | |
| `damage:taken` | `{ amount, from: Vector3, health }` | player |
| `actor:death` | `{ actor, point, impulse }` | ai |
| `player:land` | `{ velocity, surface }` | player |
| `player:footstep` | `{ position, surface, running }` | player |
| `player:state` | `{ stance, sprinting, sliding, ads }` | player |
| `explosion` | `{ position, radius, damage }` | 任意 |
| `resize` | `{ width, height }` | engine |

如果你需要一个未列出的事件，在同一个提交里在这里加一行。

## 表面类型

命中特效、贴花、音频和脚步声的共享词汇表。物理为每个碰撞体打上以下标签之一：
`concrete`（混凝土）、`metal`（金属）、`wood`（木材）、`dirt`（泥土）、
`sand`（沙地）、`glass`（玻璃）、`water`（水）、`foliage`（植被）、
`fabric`（织物）、`flesh`（血肉）、`rubber`（橡胶）、`plaster`（灰泥）。

## 渲染集成

`render` 向其他子系统暴露：

```js
const r = ctx.get('render');
r.renderer            // THREE.WebGLRenderer —— 不要在帧外修改它的状态
r.registerPass(pass)  // 插入自定义后处理 pass
r.addLight(light)     // 注册一个点光源，使其参与裁剪/预算
r.requestEnvMap()     // 当前使用的 PMREM 环境贴图
r.screenSize          // { width, height } 内部渲染目标的尺寸
r.depthTexture        // 线性深度，用于柔和粒子 / SSR
r.velocityTexture     // 运动矢量，用于 TAA / 运动模糊
```

任何绘制进 `viewScene` 的内容都会在世界之后、清空深度缓冲的状态下合成。

每对象退出开关，`render._collect` 每帧都会尊重：

```js
mesh.userData.owNoPrepass = true  // 不进入深度/法线/速度预通道
mesh.userData.owNoShadow  = true  // 不投射进 CSM 级联
```

`owNoShadow` 是**唯一的**阴影投射开关：级联用 `scene.overrideMaterial`
绘制，绝不参考 `mesh.castShadow`。`src/ai` 依赖它做屏幕外角色的 LOD。

### 点光源数量是着色器排列的关键

`r.addLight()` 会把灯光置于距离裁剪之下，裁剪会在淡出归零时将
`light.visible = false`。Three 会把**可见**点光源的数量烘焙进每个材质的
程序缓存键中，因此一盏灯越过自己的半径就会触发场景中所有已点亮材质的重编译
——实测为 +33 到 +36 个程序、那一帧 640–900 ms，在 900 帧里发生了 5 次。
任何注册了距离裁剪点光源的东西都必须保持可见数量恒定。两种做法，均像素精确：

- 把 `intensity` 驱动到 0 而保持 `visible` 为 true（`src/fx/lights.js` 的做法），或
- 停放零强度「镇流」灯，并在每个 `lateUpdate` 把数量补足到固定槽位预算
  （`src/world` 对其 17 盏实用灯的做法——见 `_stabiliseLightCount`，
  它镜像了渲染器自己的淡出测试，因为裁剪在 `lateUpdate` *之后*运行）。

一盏颜色 × 强度恰好为 0 的灯只会向辐照度累加器加一个浮点 `0.0`，
所以多余的照亮槽位不可能移动任何像素。

### 预热

`src/core/prewarm.js` 在第一帧之前运行，并为实现了 `prewarmMaterials(ctx)`
的每个子系统（`render`、`world`、`ai`）调用它。契约：**构建并编译子系统能
产生的每一种材质，但不得生成游戏对象、不得绘制游戏帧、不得触碰时钟/RNG。**
仅 `renderer.compileAsync(scene, camera)` 只能覆盖前向光照变体——无法覆盖
CSM 深度通道、MRT 预通道或后处理链。两个陷阱：

- 编译时必须绑定渲染目标。`outputColorSpace` 和 `toneMapping` 属于缓存键
  的一部分，读取自*当前绑定*的目标，因此绑定画布编译会预热错误的变体。
- `fx` 被排除在外，在第二帧自我预热：它的键依赖可见灯光数量，
  而后者只在第一帧渲染内部才确定下来。

## 质量标准

每个视觉子系统都由对抗性评审对照真实 CoD 帧审阅。底线：

- **没有平/无纹理的表面。** 每个材质都需要反照率变化、法线贴图、粗糙度变化，
  以及 0.5 m 处可见的细节层。
- **没有均匀光照。** 接触阴影、反弹、环境光遮蔽，以及清晰的主光/补光/轮廓光分离。
- **物理上合理的取值。** 反照率 0.02–0.9，金属是 0 或 1，
  真实世界的光强度，由曝光驱动而非乘数驱动。
- **没有任何东西是完全笔直、干净或重复的。** 边缘磨损、缝隙污垢、
  细微翘曲、变化的实例旋转/缩放。
- **每个动作都有重量。** 后坐力、镜头震动、屏幕空间冲击、音频瞬态，
  以及每次命中都有视觉特效。
