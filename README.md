# AgentDevTools — 开发者工具的精确点击 / Точные нажатия для инструментов разработчика

RimWorld **1.7 Android（Waylith 移植，`com.ports1de.waylith`）** 的运行时补丁模组：
让开发者工具（kill / destroy / damage / spawn …）作用在**手指真正点到的那个格子**上。

Run-time patch mod for the **RimWorld 1.7 Android port (Waylith)**: the developer tools act on the cell
your finger is actually on, instead of on whatever object the port's touch layer decided was nearest.

设计哲学见 [`docs/PHILOSOPHY.md`](docs/PHILOSOPHY.md)（六条公理、决策表、明确不做的反模式）。

---

## 1. 症状 / Симптомы

**中文**
- 打开开发者工具，点地图上一个格子，结果打在旁边的墙/床/小人身上，或者什么都不发生
- 想对"空地"使用工具（铺地板、挖矿、放置物）时，选中却跑到最近的实体
- 区域类工具（框选）与点选类工具混用后，指针位置与实际作用位置不一致

**Русский**
- Инструмент разработчика срабатывает не на той клетке: попадает в ближайшую стену/кровать/пешку,
  либо не срабатывает вовсе
- При нажатии на пустую клетку выбор «уезжает» к ближайшему объекту
- После работы с инструментами по площади позиция указателя перестаёт совпадать с точкой нажатия

---

## 2. 它怎么工作 / Как это работает

移植版的触摸层回答的是"这一下点到了哪个**对象**"，而开发者工具要的是"点在哪个**格子**"。
本模组把丢掉的格级语义补回去，并且只在真正点中对象时才吸附：

| 模式 | 行为 |
|---|---|
| `off` | 不安装任何补丁，模组等于不存在 |
| `auto`（默认） | 点中对象才吸附；点空地就是空地（16 dp 指尖容差，且不超过半格）|
| `always` | 只要 PortUI 报了对象就吸附（旧行为，供对照）|

开关文件：`files/game/agentdevtools.mode`，内容 `off` / `auto` / `always`（缺省 `auto`）。

---

## 3. 安装 / Установка

1. 下载 `AgentDevTools.zip`（Releases），解压得到 `AgentDevTools/{About,Assemblies}`
2. 整个文件夹放进游戏的 `Mods/`
3. 游戏内「Mod 配置」启用 → 重启

- `packageId` 是 **`sleepsheep.waylith.devtools`**，`ModsConfig.xml` 里的 `<li>` 必须与它完全一致，
  否则游戏会静默把这个模组移出 `activeMods`
- `About.xml` 必须位于 `About/About.xml`

该移植版 `debuggable=true` 时也可以直接写私有目录：

```bash
cat AgentDevTools/Assemblies/AgentDevTools.dll \
  | rish -c 'run-as com.ports1de.waylith sh -c "cat > files/game/Mods/agentdevtools/Assemblies/AgentDevTools.dll"'
cat AgentDevTools/About.xml \
  | rish -c 'run-as com.ports1de.waylith sh -c "cat > files/game/Mods/agentdevtools/About/About.xml"'
```

---

## 4. 验证 / Проверка

日志：`files/game/agentdevtools.log`（**不是** logcat）。

真实输出形如（v1.1.0）：

```
BUILD 1.1.0 / AgentDevTools 1.1.0.0 bootstrapping, mode=auto
patched RimWorld.Selector.HandleMapClicks -> HandleMapClicks() in Assembly-CSharp
patched PortUI.Candidates -> Candidates(Vector3) in PortUI
AgentDevTools ready: patches ok=5 failed=0, snap=auto, pick=port
tap ignored: 2 fingers down (gesture, not a tap)
debug tool fired at (123, 0, 45) (snapped: footprint)
debug tool fired at (120, 0, 40) (raw tap)
```

判据：

| 看到 | 含义 |
|---|---|
| `patches ok=5 failed=0` | 五个补丁全部命中（每行还带命中的程序集名）|
| `failed>0` + `skip …` | 某个目标在本次版本里不存在（降级运行，不崩）|
| `mode=off: no patch installed` | 处于关闭模式 |
| `tap ignored: N fingers down` | 多指手势被正确忽略（不会误开火）|
| 一条 `debug tool fired` 都没有 | 没有点击被处理，检查模组是否启用 |

开关（改完需重启）：

| 文件 | 取值 | 作用 |
|---|---|---|
| `files/game/agentdevtools.mode` | `off` / `auto`（默认）/ `always` | 总开关与吸附模式 |
| `files/game/agentdevtools.snap` | `off` / `auto` / `always` | 可选的细分覆盖（旧名 `agenttouchfix.snap` 仍可读）|
| `files/game/agentdevtools.pick` | `port` / `exact` | PortUI 候选列表策略（`exact` 会削弱吸附）|

| 一行 `click …` 都没有 | 没有点击被我们的 prefix 处理，检查是否启用了模组 |

---

## 5. 构建 / Сборка

引擎层直接使用移植版的 `Thing` / `IntVec3` / `Vector3` 等类型，**需要自备游戏引用程序集**（不随仓库分发）：

```bash
mkdir -p dlls
cp /path/to/Assembly-CSharp.dll /path/to/0Harmony.dll dlls/
dotnet build AgentDevTools.csproj -c Release
```

纯决策层（`src/Core/SnapPolicy.cs`）不依赖游戏，可以单独测：

```bash
dotnet test tests/SnapPolicy/SnapPolicy.csproj -c Release
```

---

## 6. 目录结构 / Структура

```
src/Core/SnapPolicy.cs     决策层：模式、容差、接受规则（纯函数，可单测）
src/Engine/ToolPatches.cs  点击路由与鼠标语义（Harmony）
src/Engine/SnapGeometry.cs 屏幕投影与占位命中判定
src/Engine/PickMode.cs     PortUI 候选列表的注入方式
src/Refl.cs                文件日志与反射helper
src/Bootstrap.cs           入口：读开关、装补丁、写汇总
tests/SnapPolicy/          决策层单测（CI 跑，无需游戏）
docs/PHILOSOPHY.md         设计哲学（六条公理 + 决策表 + 反模式）
```

---

## 7. 与 AgentTouchFix 的关系 / Связь с AgentTouchFix

- `AgentTouchFix` 解决**建造菜单图标空白**（渲染路径）
- 本模组解决**开发者工具点不准**（输入路径）
- 两者互不引用，可同时启用；日志分别写到 `agenttouchfix.log` 与 `agentdevtools.log`

---

## 8. 许可 / Лицензия

MIT。仓库不包含任何 RimWorld / 移植版二进制或美术资源。
