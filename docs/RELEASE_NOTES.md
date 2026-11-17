Pinpoint 2.0.0 — precise tapping for the RimWorld 1.7 (Waylith) developer tools.

Renamed from AgentDevTools. Behaviour is unchanged; the name now says what the mod does.

- Tools fire on the cell your finger is on: the tap is pinned to UI.MouseCell()/UI.MouseMapPosition()
  for the duration of the click action, so LudeonTK.DebugTool receives the input it waits for.
- Snapping is an exception, not the rule: a candidate is accepted only inside the thing's own
  footprint or within 16 dp of its projected DrawPos, never more than half a cell on screen, so
  zooming out cannot widen the snap into neighbouring cells.
- Modes: off / auto (default) / always, read once from files/game/pinpoint.mode.
- Every click logs the decision and the reason it took.

Note for existing users: the packageId changed from sleepsheep.waylith.devtools to
sleepsheep.waylith.pinpoint, and the mode file is now files/game/pinpoint.mode. Disable the old
folder before enabling this one.
