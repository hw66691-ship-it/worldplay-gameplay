---
name: worldplay-gameplay
author: hewei
description: >-
  在抖音造世界（Worldplay/造世界编辑器）中新构建世界、关卡、场景、物品、实例物品或 UI，或添加、修改玩法交互与效果时使用。按交付类型制作并验证 GLB、MIX 或 WPREFAB、可修改源码 ZIP 和导入说明；不用于其他游戏引擎或泛指“创造一个世界”的写作。
---

# 造世界玩法构建 SOP

## 适用与定位

- 用户说“在造世界中新构建……”“在造世界中添加……玩法”“给造世界的物体加交互/效果”，或明确指向造世界编辑器资源时使用。先弄清用户要交付的是地形、地图摆放、Module、UI、单物体交互，还是跨资源玩法；同一任务可涉及数项。
- 把用户指定的项目目录作为工作区。若未指定，先在当前上下文寻找实际项目和 `ai-sessions/workspace_*`；不要把编辑器的 `resources/addon` 当成游戏项目。找不到实际项目时，完成可独立完成的设计和模板准备，再询问项目位置。需要创建独立新项目时，遵守用户的工作区偏好。
- addon 路径不固定，不要写死盘符或应用名。先在当前编辑器进程/应用包、工作区和 `ai-sessions/workspace_*` 的父级目录中查找候选 addon 根目录；Windows 常见形态是 `<安装目录>\resources\addon`，macOS 常见形态是 `<应用>.app/Contents/Resources/addon`，Linux 常见形态是 `<安装目录>/resources/addon`。以同时包含 `agent-runtime/`、`lua-support/` 和 `node_modules/@ai-world-creator/worldplay-toolkit-core/` 的目录作为 `ADDON_ROOT`；找到多个候选时优先选择与当前编辑器版本和正在使用的工作区匹配者。找不到时记录未找到并停止依赖 addon 的步骤，不要猜测路径。这里的 `ADDON_ROOT/agent-runtime/templates` 是底稿，`ADDON_ROOT/agent-runtime/tools` 是编辑器工具实现，`ADDON_ROOT/lua-support` 是运行库与类型提示。游戏改动写入实际 `workspace_*`，不直接改 addon 来实现某个玩法。
- 先检查工作区现有文件、资源 ID、模型节点、`meta.json`、脚本和既有交互。延续既有命名及运行约定。必要时阅读当前 addon 的相关工具说明和模板；某些附带文档可能是加密占位内容，不要把占位文本当作完整规范。

## 交付合同

在动手前按用户最终要导入的对象确定交付类型。同一需求可能同时产出多类资产；每一类分别满足下表。地形、场景或关卡若要作为可玩的完整世界导入，按“玩法”交付；仅制作可复用单个物体按“物品”交付。不要用文件改名代替格式转换。

| 交付类型 | 可直接导入的完整资产 | 同时交付 |
| --- | --- | --- |
| 物品（Module）、运行时 UI | 各自的 `.glb`，含该资源实际需要的模型/界面、元数据、绑定和运行脚本 | `<名称>-source.zip`（可修改的作者侧源码及必要素材）、`导入与使用说明.md` |
| 玩法、完整世界 | `.mix`，包含根（及所需的 Runtime Host 脚本）、实际使用的 Module/UI GLB parts 和布局 | `<名称>-source.zip`（Map、Module、UI、Lua、素材及生成/打包所需文件）、`导入与使用说明.md` |
| 实例物品（编辑器 Prefab） | `.wprefab`，保存可复用的实例层级、变换、组件/绑定与资源引用 | `<名称>-source.zip`（可修改的实例/Prefab 作者数据、脚本和依赖素材）、`导入与使用说明.md` |

- 源码 ZIP 应可解压、定位入口并重建对应资产。纳入必要的编辑器工作区源文件、生成代码、Lua、HTML/CSS、模型/贴图源文件或其可重建脚本、依赖清单；不要只把最终二进制重新压缩。排除缓存、无关项目、密钥和用户私有文件。
- 导入说明写明适用编辑器版本、每个文件的用途、实际验证过的导入入口与步骤、依赖和放置条件、交互操作/参数、源码修改后重新打包的方法，以及已完成和未完成的验证。不要凭猜测写菜单名。
- `.mix` 必须由当前编辑器的导出链路或 `@ai-world-creator/worldplay-toolkit-core` 的 `WorldplayMixPackageWriter` 生成：它是 ZIP 容器，含 `manifest.json` 和 manifest 引用的 `parts/*.glb`。`archive(target=mix)` 当前只产出 `package/mix-archive.json` 及 GLB parts；这一步本身**不是** `.mix` 成品。按索引中的根、实例顺序、类型、位姿与资源引用组装，再回读校验。
- `.wprefab` 使用编辑器实际的 Prefab 导出/保存链路。当前本机样本是含 `SerializePcgData.json` 和 `preview.png` 的 ZIP 容器；以目标编辑器版本的真实导出结果为准。不能把 GLB、场景 JSON 或 ZIP 任意改后缀冒充 WPREFAB。

## 文件路由

| 需求 | 作者侧位置或入口 | 关键区别 |
| --- | --- | --- |
| 自然地形与撒点 | Map Chat 的 `pcg_generate_reference_images` / `pcg_generate_terrain` / `pcg_generate_sprinkle` | PCG 按参考图 → `Terrain3d` → 撒点的 artifact ID 链路运行；不等同于 Placement 工作区源码。 |
| 场景中放置 Module | `assets/map/map-<id>/source/src/main.js` | Three.js 代理用于预览和记录摆放；`placement-runtime.js` 是固定运行辅助，不是玩法脚本。 |
| 物体外观与节点 | `assets/module/module-<id>/static/preview-module.js` 或 `topology/model.glb`，加 `meta.json`、必要的 anchor | Geometry 与 Topology 来源不同；先检查模型实际节点再写目标路径。 |
| 单物体交互 | `assets/module/module-<id>/scripts/**/pi-*.lua` | 进入范围、聚焦、按钮、拾取、座位、命中和局部反馈等运行逻辑。脚本目录对应模型节点 `targetPath`。 |
| 人形/生物运动 | 同一 Module 的 `scripts/**/ms-*.lua` | 骨骼适配、运动状态、贴地和动画切换。 |
| 多物体规则 | 工作区根 `scripts/runtime-host.lua`，配合各资源脚本 | 用注册、查询和事件分发连接资源；不要把跨资源状态散落在多个不协调的脚本中。 |
| 运行时界面 | `assets/ui/ui-<id>/html/1920x1080.html`、`images/`、`scripts/ui.lua` | HTML/SVG/PNG 管外观，Lua 管运行时交互；需要竖屏时增加 `html/1170x2532.html`。 |
| UI 动画与物体效果 | `official.ui.animation`、相关 UI Lua 或 Module Lua | 区分视觉反馈和玩法状态；Three.js Placement 预览动画不等于 Unity 运行时效果。 |

## 实施流程

1. **定义最小可验证闭环。** 写清触发条件、执行者、状态变化、玩家可见反馈、多人同步要求、离开/销毁时清理。例：“本地玩家走近宝箱 → 出现打开按钮 → 点击后只结算一次 → 播放发光并更新 UI”。把未知的引擎能力标出来，先查现有模板或 API 声明再实现。
2. **定位资源和依赖。** 确定资源 ID：`module-*`、`map-*`、`ui-*`、最终 `mix-*`。检查模型节点路径、锚点、Collider、trigger、已有脚本和 UI 节点。优先在现有资源内增量修改；新资源可使用 addon 的模板应用脚本，但这些脚本通常要求从 `ai-sessions/workspace_*` 根目录运行，且拒绝覆盖已有目标。
3. **做模型与元数据。** 在 `meta.json.attributes` 给正确 `targetPath` 配置需要的碰撞体、trigger、interaction 等；让模型节点名、锚点路径、脚本目录和元数据一致。不要手写未核实的元数据字段值：参照同类可用资源、当前模板与归档校验结果。
4. **写运行逻辑。** 单物体玩法写 `pi-*.lua`；持续运动写 `ms-*.lua`。它们是独立 DouyinScript，不是 `return { ... }` 的 Lua 模块。常用回调可参照现有模板：`Start` 初始化，`OnActorTriggerEnter/Exit` 控制可交互范围，`OnFocus/OnLostFocus` 管按钮，`OnInteractorButtonClick` 执行一次动作，`OnDisable/OnDestroy` 释放按钮、监听和临时对象。交互中检查本地玩家；影响共享物体的状态参照已有所有权/同步做法。
5. **接入效果与 UI。** 在状态结算成功后触发声音、粒子、材质、位移、后坐力或 UI 动画；用可确认的 API 和资源节点。需要跨物体联动时，由资源向 Runtime Host 分发具名事件，目标资源注册处理器；保留清晰的事件名、payload 和重复触发规则。UI 脚本统一从 `assets/ui/ui-<id>/scripts/ui.lua` 进入。
6. **在编辑器里试玩。** 检查初次进入、重复点击、离开范围、重新进入、对象销毁、多个实例，以及有要求时的多玩家视角。核对效果是否只在正确时机出现，状态是否保持一致，按钮/监听是否清理。地图只负责摆放的任务也要检查预览与最终实例位置。
7. **归档与打包。** 用编辑器暴露的 `archive` 工具逐个归档改过的 `module-*`、`ui-*`、`map-*`（`target=resource`）；组成完整世界时，在所有资源成功后归档 `mix-*`（`target=mix`）。只有返回明确的 `completed` 才算该归档通过。按“交付合同”导出最终 GLB/MIX/WPREFAB，并制作源码 ZIP 和说明。归档报错时修作者侧源码再重试，不改 `package/**`、Map `review/**` 或生成的 GLB 来掩盖问题。
8. **验证并交付。** 执行下方验收门槛；最终回答给出三件套的实际文件路径、玩法操作方式、测试结果及限制。缺少任一交付物或关键验证未通过时，不宣称该资产已完成；说明具体阻碍和可复现的剩余步骤。

## 制作后的验收门槛

1. **设计与规则。** 对照需求逐项检查外观比例、放置环境、控制方式、效果时序、反馈、可达性及重复操作行为。检查本机模板/API、资源命名、资源 ID、节点/锚点、碰撞体、脚本绑定与事件约定；不调用未经证实的接口。需要多人同步时验证状态归属和不同玩家视角。
2. **源码与归档。** 检查 Lua 语法和角色脚本约束、UI 入口与引用、模型节点路径、Map 摆放和依赖完整性；相关 `archive` 均须明确 `completed`。用实测交互覆盖进入/离开范围、角度或其他参数边界、重复触发、销毁/重进、多个实例，以及成功和失败路径。效果须由真实状态驱动，不仅在预览中看起来正确。
3. **成品格式。** 核对三个交付文件均存在且非空；解压并回读源码 ZIP。GLB 校验 `glTF`/GLB 2.0 结构、节点与运行 payload；MIX 校验 ZIP、`manifest.json`、`schemaVersion`、`resourceType`、每个 `entryPath`、part 类型/位姿及 GLB 内容与归档索引一致；WPREFAB 校验 ZIP、编辑器要求的场景数据/预览、唯一根节点、父子引用、实例变换、组件/脚本绑定及外部资源可解析。按目标编辑器的当前格式核验，不沿用过期样本字段。
4. **编辑器回验。** 用交付的最终资产文件在造世界编辑器中重新导入，在可用场景里摆放、预览/运行并核对完整交互；确认源码 ZIP 可按说明修改并重建。记录编辑器版本、测试步骤和结果。若当前环境无法访问导入或运行功能，仍完成可执行的静态/格式/归档检查，并明确标记“编辑器导入或试玩未验证”；不能把静态通过表述成全面验证。

## 常见约束与定位线索

- Module 归档只接受非空 `pi-*.lua` 与 `ms-*.lua` 角色脚本；子目录形成 binding `targetPath`。交互脚本调用触发/按钮 API 时，同一目标路径需要相应 `meta.attributes[<targetPath>].trigger`。可查 addon 的 `agent-runtime/tools/archive/module/model-script-bundle/worldplay-hamlet-script-validation.ts` 与 `agent-runtime/tools/archive/module/model-script-bundle/worldplay-hamlet-script-discovery.ts`。
- `RuntimeScope.from(self)`、`scope:FindTransform(path)`、`scope:GetScript(scriptId)` 可用于资源内节点和脚本查找；跨资源 Host 用 `scope:GetRuntimeHostScript("runtime-host.lua")`。具体写法参照 addon 的 `lua-support/Library/official/core/runtime_scope.lua` 和 `agent-runtime/templates/runtime-host/assets/runtime-host.lua`。
- `official.interaction.trigger` 可查找 BoxCollider 并调整范围；`official.ui.animation` 提供常用 UI tween。先确认目标确实有对应 Collider/UI 对象。
- Runtime UI 横屏入口是 `html/1920x1080.html`。存在 helper Lua 时必须有非空 `scripts/ui.lua`；完整 Mix 中的 UI 需要运行脚本。归档还会校验 HTML 节点、图片路径和布局。
- Placement Map 的 `main.js` 写 Module 代理与 `THREE.Matrix4` 位姿；placement 矩阵保持单位缩放。Map 单资源归档构建预览和 manifest，最终 Mix 归档再结合真实 Module GLB 求统一缩放。不要把这个链路误作 PCG 地形生成。
- 可复用的例子在 addon 的 `agent-runtime/templates/module-interaction/assets/`：`four-wheel-vehicle-single-file-base.lua`（触发区、按钮、座位和驾驶）、`pickup-raycast-rifle/`（拾取、所有权、射线和枪口反馈）、`humanoid/motion/` 与 `creature/motion/`（运动状态）。按实际玩法裁剪，不整份复制无关逻辑。
