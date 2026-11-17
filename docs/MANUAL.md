# Pinpoint — 技术说明书

| 项 | 值 |
|---|---|
| 组件 | `Pinpoint`（原 `AgentDevTools`） |
| 目标运行时 | RimWorld 1.7 / Waylith Engine / Android（`com.ports1de.waylith`） |
| 注入方式 | Harmony（运行时；不改文件、不重打包） |
| 补丁面 | 触摸→鼠标语义转换；`LudeonTK.DebugTool` 输入；`UI.MouseCell/MouseMapPosition` |
| 配置 | `files/game/pinpoint.mode`（`off` / `auto` / `always`） |

---

## 1. 问题域：不是坐标错，是语义丢失

移植版触摸层（`PortUI.MobileSelectionController`）的返回值语义是**对象级**：

```
tap(x, y) → Tap() → 合成选择调用 → "最近的实体"
```

而原版开发者工具消费的是**格级**输入：

```
LudeonTK.DebugTool.DebugToolOnGUI → UI.MouseCell() / UI.MouseMapPosition()
```

两者存在两个独立缺陷：

| 编号 | 缺陷 | 机制 | 用户可见表现 |
|---|---|---|---|
| A | 工具不触发 | 触摸层不产生 `MouseDown`，`DebugTool` 永远收不到事件 | 工具已激活，点地图毫无反应，且点击被选择器吃掉 |
| B | 打错目标 | 移植版按 28.8 px 半径做"就近吸附"，对选择合理、对工具越界 | 点空地却在旁边物品上生效；清地形/放置物落到邻居格 |

---

## 2. 分层结构

```
src/Core/SnapPolicy.cs        纯函数：给定(触点, 候选, 模式) → 结论 + 理由      无游戏类型，可单测
src/Engine/SnapGeometry.cs    测量：投影、footprint、16 dp 上界               只读游戏状态
src/Engine/ToolPatches.cs     Harmony：钉住 MouseCell/MouseMapPosition、放行点击  唯一的副作用层
```

设计约束：**判定逻辑与副作用分离**。`SnapPolicy` 在 CI 上可无引擎运行（`tests/SnapPolicy`），
`ToolPatches` 只做三件事——测量、询问策略、施加结果。

---

## 3. 判定规则

对每个候选（`MobileSelectionController.Candidates(tapPos)` 的首项）：

```
accept  ⇔  tap ∈ footprint(thing)                        // 触点落在物体自身占位格内
        ∨  dist(tap, project(DrawPos(thing))) ≤ 16 dp     // 或距其投影不超过 16 dp
           且 dist ≤ 0.5 × cellSizeOnScreen             // 并且屏幕上不超过半格
```

拒绝时使用**原始触点**对应的格子与位置（不做四舍五入损失）。第三条约束的作用是让**缩放无关**：
视图拉远时屏幕半格变小，吸附范围随之收紧，不会把相邻格吞进来。

| 模式 | 语义 |
|---|---|
| `off` | 不打任何补丁 |
| `auto`（默认） | 上述判定 |
| `always` | 直接取候选首项（v5 旧行为，保留用于对照/回退） |

---

## 4. 补丁时序

```
DebugTool 激活 → 用户点击
  ├─ Prefix: 记录触点；解析目标（SnapPolicy）
  ├─ 把 UI.MouseCell()/MouseMapPosition() 钉到解析结果（仅在 clickAction 期间）
  ├─ 合成 MouseDown，使 DebugToolOnGUI 走正常分支
  └─ Postfix: 还原钉住状态，写一行决策日志
```

只在 `clickAction` 期间钉住，避免污染其它帧的鼠标语义；失败路径一律 fail-open（异常即放弃补丁、保留原行为）。

---

## 5. 可观测性

每次点击记录一行，字段固定：

```
pinpoint: mode=auto tap=(x,y) cell=(i,j) candidate=<thing> decision=accept|reject reason=footprint|within-16dp|off-footprint|off-threshold dist=<dp>
```

判定为 `reject` 时同时记录 `dist` 与阈值，便于事后回答"为什么这一下没吸过去"。

---

## 6. 构建与发布（与 GlyphFix 的差异）

引擎层引用游戏真实参考程序集，**这些程序集不再分发**，因此本仓库的 CI 默认只跑纯逻辑单测：

```xml
<Reference Include="Assembly-CSharp"><HintPath>..\dlls\Assembly-CSharp.dll</HintPath></Reference>
```

`dlls/` 位于仓库外的同级目录。要在 CI 里产出 DLL，需要二者之一：

1. 在仓库添加 secret `DLLS_TOKEN`（对私有仓库 `rimworld-waylith-re` 有读权限），`build.yml` 会用它取 `dlls/` 后编译并打包扁平 zip；
2. 或在具备 `dlls/` 的环境本地构建。

打包格式（游戏内安装器要求根级目录结构）：

```
Pinpoint.zip
├── About/About.xml
└── Assemblies/Pinpoint.dll
```

手动安装则相反：解压出的 `Pinpoint/` 文件夹放进 `Mods/`。

---

## 7. 验收标准

1. 激活任意开发者工具，点地图上的空地 → 工具在**该格**生效
2. 点在一个物体上 → 工具作用在该物体（吸附生效）
3. 紧邻物品的空格上使用"清地形/放置"类工具 → 不跳到相邻物品
4. 缩放比例改变后，第 3 条仍成立（吸附范围随缩放收紧）
5. `files/game/pinpoint.mode` 写 `off` → 行为与未安装一致
