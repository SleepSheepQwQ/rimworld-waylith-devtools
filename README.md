# Pinpoint — precise tapping for RimWorld 1.7 dev tools (Waylith Android port)

**English** · [简体中文](README.zh-CN.md) · [Русский](README.ru.md)

The port's touch layer answers *"which object did this tap hit?"* — but the debug tools ask
*"which cell is the mouse on?"*. Pinpoint puts the missing half back: your finger's cell is the
answer, and snapping to an object is a named exception that must be asked for, bounded and logged.

> Formerly `AgentDevTools`. Same behaviour, a name that says what it does.

## The two defects it addresses

**1. The tool never fires.** A tap on the map is turned into a synthetic selection call by the touch
layer (`PortUI.MobileSelectionController.Tap`) and never produces the `MouseDown` that
`LudeonTK.DebugTool.DebugToolOnGUI` consumes, so an armed tool does nothing and the tap is stolen by
the selector. Pinpoint pins `UI.MouseCell()` / `UI.MouseMapPosition()` to the tap for the duration of
the click action, so the tool receives the input it was waiting for.

**2. It fires on the wrong thing.** Snapping to the nearest target is right for gameplay and wrong
for tools: emptying terrain or spawning a thing next to a dropped item used to jump onto that item.
A candidate is now accepted only when the tap lies inside the thing's own footprint, or within 16 dp
of its projected `DrawPos` — never more than half a cell on screen, so zooming out cannot widen the
snap.

## Modes

Read once at startup from `files/game/pinpoint.mode`:

| Mode | Behaviour |
|---|---|
| `off` | no patches at all; the port behaves exactly as shipped |
| `auto` (default) | snap only when the tap really lands on the object |
| `always` | legacy: always take the first target the port reports |

Every decision is logged with the reason it took, so a wrong click can be explained after the fact.

## Install

Drop `Pinpoint` into `Mods/`, enable, restart. The mod zip is flat — `About/` and `Assemblies/` sit at
its root, which is what the port's in-game installer expects.

## Requirements

RimWorld 1.7 (Waylith engine), Android port `com.ports1de.waylith`. No game file is modified;
everything is applied at runtime with Harmony.

Design rationale: [`docs/PHILOSOPHY.md`](docs/PHILOSOPHY.md) · implementation notes:
[`docs/MANUAL.md`](docs/MANUAL.md)
