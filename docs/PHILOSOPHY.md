# 设计哲学 / Философия дизайна — AgentDevTools

> 一句话：**手指点在哪一格，就作用在哪一格；吸附是例外，必须被请求、有界、可解释。**
> Одной строкой: **палец указывает клетку — действует клетка; «прилипание» к объекту это исключение,
> которое нужно запросить, оно ограничено и объяснимо.**

这个模组只做一件事：把 Android 移植版的**触摸点**翻译成原版**鼠标语义**，让开发者工具（kill / destroy /
damage / spawn …）打在玩家真正指的那个格子上。它不是"触控增强包"，也不是"自动瞄准"。

---

## 1. 问题的本质不是坐标，而是语义丢失

移植版的触摸层（`PortUI.MobileSelectionController`）回答的是"这一下点到了哪个**对象**"，
而不是"这一下点在哪个**格子**"。对原版玩法这通常够用；对开发者工具就是致命的：

- `DebugTool` 的点击回调要的是 `UI.MouseMapPosition()` / `UI.MouseCell()`——**格级输入**
- 触摸层给的是"最近的实体"——**物级输入**
- 两者一旦混淆，工具会打在"最近的墙"上，或者因为找不到实体而什么都不做

所以我们不改工具的判定逻辑，只**把丢失的语义补回去**。

---

## 2. 六条公理（写代码时要能逐条对照）

### A1 · 格子优先 / Клетка прежде всего
`UI.MouseCell` 与 `UI.MouseMapPosition` 返回的必须是**手指所在**的格子与位置。
一切后续判定都建立在这个"锚点"上；锚点错了，后面再聪明都是错的。

### A2 · 吸附是例外，不是默认 / Прилипание — исключение
只有在**同时满足**下面三条时才吸附到对象：
1. PortUI 的候选列表**非空**（它确实在那里看见了东西）
2. 工具**需要**一个对象（区域类工具如铺地板/挖矿不需要）
3. 手指**落在该对象的占位范围内**（16 dp 的指尖容差，且不超过半格）

任一条不成立 → 用原始格子。**"点空地"必须真的是点空地**，不被"最近的实体"劫持。

### A3 · 只做那一毫米 / Только один миллиметр
本模组不碰渲染、不碰 UI 布局、不碰选择逻辑、不改游戏数值。
一旦发现需要改这些，那属于另一个模组（例如图标问题在 `AgentTouchFix` 里）。

### A4 · 测量与决策分离 / Измерение отдельно от решения
- **测量层**（Harmony）：手指落在哪、PortUI 回了什么、当前工具要不要对象
- **决策层**（`SnapPolicy`）：纯函数，无游戏类型、无全局状态

好处是决策表可以在 CI 上用 xunit 钉死（`tests/SnapPolicy`），不需要设备、不需要引擎。

### A5 · 静默可观测 / Тихая наблюдаемость
- 例行输出只写文件 `files/game/agentdevtools.log`
- **不弹窗**：`Verse.Log.Warning/Error` 在这个移植版会弹"there were errors"对话框
- 一次真实失败才允许进游戏日志，且每个进程最多一次（`ErrOnce`）
- 每一次改写都要能回答"为什么"：命中了哪个候选、距离多少、走了哪条规则

### A6 · 失败隔离 / Изоляция отказов
- 每个补丁独立 `try/catch`，一个目标找不到只记一行 `skip …`，不影响其它补丁与启动
- `mode=off` 时**一条补丁都不装**，模组在运行期等于不存在
- 补丁全部装完写一行汇总：`patches ok=N failed=M, snap=…, pick=…`

---

## 3. 决策表（`SnapPolicy.Accept`）

| 模式 | 候选列表非空 | 手指在占位内 | 结果 |
|---|---|---|---|
| `off` | 任意 | 任意 | 原始格子（不装补丁）|
| `auto` | 否 | — | **原始格子** |
| `auto` | 是 | 是 | 吸附到该对象 |
| `auto` | 是 | 否 | **原始格子** |
| `always` | 否 | — | 原始格子 |
| `always` | 是 | 任意 | 吸附到该对象（旧行为，供对照）|

容差：`16dp × 屏幕密度`，下限 6 px（缩得很小时不至于点不中），上限 64 px，且不超过半格宽。

---

## 4. 明确不做的四件事（反模式）

| 反模式 | 为什么不做 |
|---|---|
| **总是吸附最近的实体** | 违背 A1/A2：点空地会被劫持，玩家无法"点地板铺地板" |
| **改 `DebugTool.clickAction` 的参数** | 违背 A3：那是在改游戏逻辑；我们只修正它看到的输入 |
| **拿截图/像素做命中判定** | 违背 A4：不可测、不可解释、分辨率一变就失效 |
| **失败后自动重试 / 自动换目标** | 违背 A5/A6：行为变得不可预测，且掩盖真实缺目标 |

---

## 5. 演进规则

新增能力必须同时满足：

1. 能写成**纯函数**放进决策层，并有单测
2. 默认关闭或默认与桌面鼠标一致（不惊讶）
3. 有文件开关与一行说明日志
4. 不引入新的编译期依赖到游戏内部类型（引擎层只做反射与薄适配）

不满足其中任何一条 → 另开一个模组，别塞进来。

---

## 6. 与 `AgentTouchFix` 的边界

| 问题 | 归属 |
|---|---|
| 建造菜单图标空白 | `AgentTouchFix`（渲染路径）|
| 开发者工具点不准、点空地跑到别处 | **本模组** |
| 单指误触发缩放、指针卡住、IME 丢字 | 移植版输入层的其它问题，暂不改动 |

两个模组可以同时启用，互不引用；日志分别写 `agenttouchfix.log` 与 `agentdevtools.log`。

---

## 7. 外部负面经验（业界反复踩的四类坑）

调研了公开案例后，**所有失败都可归到四类**，每一类都对应本模组的一条硬约束：

| 类别 | 典型后果 | 来源（一手/社区） | 我们的对策 |
|---|---|---|---|
| **替你选（snapping）** | 点空地却打到最近目标；玩家被迫"先走近再点" | Genshin 自动锁定覆盖朝向 <https://screenrant.com/genshin-impact-auto-lock-target-combat-bad-fix>；Wild Rift 补 Target Lock Filtering <https://www.oneesports.gg/wild-rift/target-lock-filtering-wild-rift-patch> | A1/A2：格子是唯一锚点，候选为空即空地；吸附条件三条同时成立 |
| **状态不按 id 对账** | 按钮永久 pressed、光标卡住、幽灵点击 | Unity 双指卡按钮 <https://discussions.unity.com/t/ui-button-gets-stuck-in-the-pressed-state-while-playing-with-touch-screen/835916>；Android 拦截发 `ACTION_CANCEL` <https://codetheory.in/android/touch-gestures/understanding-android-input-touch-events> | A6′：输入状态机必须幂等收敛（见下）|
| **一个手势挂两个语义** | 长按想取消却弹菜单；菜单抬手即关 | tldraw iOS 长按 500ms 后 pointerup 被菜单判为外部点击 <https://github.com/tldraw/tldraw/issues/8740>；Apple 把长按定义为 continuous gesture <https://developer.apple.com/documentation/uikit/handling-long-press-gestures> | 本模组不新增长按语义：长按仍归移植层，我们只处理"点" |
| **用容差代替反馈** | 手指盖住目标，只能盲点；精度差被误认为"传感器问题" | Shift 论文：错误主因是遮挡与选择点歧义 <https://www.dgp.toronto.edu/papers/dvogel_CHI2007.pdf>；iOS 13 删 loupe / 15 加回 <https://9to5mac.com/2021/06/07/ios-15-brings-back-the-magnifying-glass-for-accurate-text-selection> | 容差只放大命中区、不放大格子；命中结果写进日志以便事后复现 |

### 三条被业界反复验证的数值基准

- **触摸目标 ≈ 9 mm**（≈48 dp / 44 pt）——但这是**可点击控件**的尺寸；地图格子是内容不是控件，
  所以我们的做法是"格子不变、命中区加一点点"（16 dp 指尖容差，且不超过半格）。来源：
  <https://support.google.com/accessibility/android/answer/7101858?hl=en>、<https://www.nngroup.com/articles/touch-target-size>
- **touch slop 取系统值**：Android 8 dp、Flutter 18 lp。自己拍数字是误触的主要来源。来源：
  <https://api.flutter.dev/flutter/gestures/kTouchSlop-constant.html>
- **放大命中区必须被邻居裁剪**：Fitts 定律的真正约束是相邻目标，不是目标本身。来源：
  <https://www.uxtigers.com/post/fitts-law>

---

## 8. 经外部经验修正后的公理（v1.1）

原六条保留，补强如下：

### A1′ · 坐标契约
判定函数的输入必须是 **(cell, layer)**，不是"世界坐标再去反查最近对象"。
只要数据流里还存在"最近"，A1 就形同虚设（Genshin 的教训）。

### A2′ · 吸附四条件
在原来三条之外补第四条：**一次手势只判定一次**。按下时确定候选集，抬手前只允许在候选内微调，
不允许 down/up 两套规则各算一遍（tldraw 的菜单被同一次抬手关掉，就是这个错误）。

### A4′ · 参数显式注入
容差、slop、长按时长、连点窗口全部作为**带单位的参数**传入决策层，禁止在纯函数里读全局 DPI 或常量。
否则同一份逻辑在不同密度设备上不可复现，测试也覆盖不到边界。

### A5′ · 日志要能分辨"点错格"与"吸错对象"
每次点击至少记录：`pointerId, phase, 原始落点, 判定出的 cell, 候选数, 选中项, 拒绝原因`。
真机上这两类 bug 外观完全相同，字段不够就无法定位。

### A6′ · 输入状态机幂等收敛
`cancel` / `ACTION_CANCEL` / 失焦 / 切后台 / 旋转重建，必须全部走**同一个** reset 到 idle 的路径，
且 reset **不发任何游戏命令**。这是"按钮卡住、幽灵点击"这一类 bug 的唯一通用解法。
