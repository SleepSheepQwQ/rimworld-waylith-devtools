# Pinpoint — 环世界 1.7（Waylith 安卓移植版）开发者工具精确点击

[English](README.md) · **简体中文** · [Русский](README.ru.md)

移植版的触摸层回答的是"这一下点到了哪个**对象**"，而调试工具问的是"鼠标在哪个**格子**"。
Pinpoint 把丢掉的那一半补回去：手指所在的格子就是答案，吸附到对象是一个**必须被请求、有界、可解释**的例外。

> 曾用名 `AgentDevTools`，行为不变，只是名字如实描述了它做的事。

## 它解决的两个缺陷

**一、工具根本不触发。** 地图上的一次点击被触摸层转成合成选择调用（`PortUI.MobileSelectionController.Tap`），从不产生 `LudeonTK.DebugTool.DebugToolOnGUI` 需要的 `MouseDown`，于是已激活的工具什么都不做，点击被选择器截走。Pinpoint 在点击动作期间把 `UI.MouseCell()` / `UI.MouseMapPosition()` 钉在触点位置，工具就拿到了它等的输入。

**二、打在了错的东西上。** 就近吸附对玩法是对的、对工具是错的：清地形或放置物时，旁边躺着一件物品就会被吸过去。现在只有在触点落在物体自身占位格内，或距其投影 `DrawPos` 不超过 16 dp（且屏幕上绝不超过半格）时才接受候选——所以缩小视图也不会把吸附范围扩大到相邻格子。

## 模式

启动时读一次 `files/game/pinpoint.mode`：

| 模式 | 行为 |
|---|---|
| `off` | 完全不打补丁，与未安装一致 |
| `auto`（默认） | 只有点击真的落在物体上才吸附 |
| `always` | 旧行为：永远取移植版报告的第一个目标 |

每次决策都会记录依据，点错了可以事后解释清楚。

## 安装

把 `Pinpoint` 放进 `Mods/`，启用，重启。模组 zip 是扁平结构——`About/` 与 `Assemblies/` 在根目录，正是游戏内安装器要求的格式。

## 环境

环世界 1.7（Waylith 引擎），安卓移植版 `com.ports1de.waylith`。不改任何游戏文件，全部在运行时用 Harmony 施加。

设计依据：[`docs/PHILOSOPHY.md`](docs/PHILOSOPHY.md) · 实现说明：[`docs/MANUAL.md`](docs/MANUAL.md)
