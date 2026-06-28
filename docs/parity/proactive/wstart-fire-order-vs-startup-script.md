# World-start triggers fire AFTER the startup/init scripts instead of BEFORE them

**Confidence:** Medium-High (the original ordering is high-confidence and traced end-to-end;
the only soft spot is regression risk inside OpenGothic-specific code when reordering).

## Original function + address (prose)

In the retail engine a `zCTriggerWorldStart` does *not* react to touch/trigger/timer at all
(`zCTriggerWorldStart::OnTouch/OnUntouch/OnUntrigger` @0x0060c7c0/0x0060c7d0/0x0060c7e0 are stubs).
Its one and only firing path is `zCTriggerWorldStart::PostLoad` @0x0061a4e0, which the engine runs
during the per-vob post-load pass while a world is being brought up. PostLoad reads two byte fields:
field_0x134 = `fireOnceOnly` (the archived `fireOnlyFirstTime`, always serialized) and
field_0x135 = a runtime "has-fired" flag (serialized *only in savegames*, gated by the
`zCArchiver` IsSavegame query at vtable+0x100; see `Archive`/`Unarchive` @0x0061a530/0x0061a590).
The logic is: if `fireOnceOnly==0` fire every load; else if `hasFired==0` fire once. Firing calls
`zCTriggerWorldStart::OnTrigger` @0x0061a510, which forwards to `zCTriggerBase::OnTrigger`
@0x0060fd00 (send OnTrigger to every vob named `target`) and sets `hasFired=1`.

The decisive ordering fact: this PostLoad pass is part of *world load*, which completes before the
**startup script** runs. In `oCGame::LoadWorldStartup` @0x0061c9c10 (the new-game / first-visit
path, reached from `oCGame::LoadWorld` @0x006c90b0 only after the ZEN/vob load step), the vob
post-load/enable pass is invoked at the head of the function (GetGameWorld then world-vtable+0x10),
and only at the very tail does it call `CallScriptStartup` @0x006c1c70 (STARTUP_GLOBAL /
STARTUP_<world>) followed by `CallScriptInit` @0x006c1f60 (INIT_GLOBAL / INIT_<world>). There is no
code path in which a vob's PostLoad runs after CallScriptStartup. Therefore world-start triggers
fire **before** the startup script inserts NPCs and before init_<world> runs. (The savegame-resume
path `oCGame::LoadWorldStat` @0x006ca010 PostLoads the vobs too but never calls CallScriptStartup,
so ordering-vs-script is moot there.)

## OpenGothic file:line

- `game/game/gamesession.cpp:113-115` (new game / first world bring-up):
  `initScripts(true)` is called first, then `wrld->triggerOnStart(true)`.
- `game/game/gamesession.cpp:417-418` (level change / changeWorld):
  `initScripts(wss.isEmpty())` first, then `wrld->triggerOnStart(wss.isEmpty())`.
- (`game/game/gamesession.cpp:177`, savegame-resume ctor: `triggerOnStart(false)` standalone — no
  startup script runs on resume, so this call site needs no change.)

`World::triggerOnStart` -> `WorldObjects::triggerOnStart` (`game/world/worldobjects.cpp:527`) sends a
`T_Startup`/`T_StartupFirstTime` event to every `TriggerWorldStart`, whose `onTrigger`
(`game/world/triggers/triggerworldstart.cpp:13`) forwards to the target. `GameSession::initScripts`
(`game/game/gamesession.cpp:498`) runs startup_global / startup_<world> (NPC insertion) on the
firstTime branch and always runs init_global / init_<world>.

## Divergence

OpenGothic runs the startup/init scripts **first** and fires the world-start triggers **afterward**;
the original fires world-start triggers (vob PostLoad) **before** CallScriptStartup/CallScriptInit.
The order is observable whenever a world-start trigger and an init_<world> / startup_<world> script
both touch the same target's initial state (e.g. a mover/door/TriggerScript that the world-start
trigger toggles and that init code also configures, or a TriggerList whose members the script then
re-arms): in the original the script gets the final word over the world-start pulse, while in
OpenGothic the world-start pulse lands last and can clobber the scripted setup. Note targets are
always pre-existing static vobs (movers, script/list triggers) resolved by name, so firing before
the NPC-spawning startup script is safe — world-start triggers never target script-spawned NPCs.

## Proposed patch (reorder so world-start fires before the scripts)

`game/game/gamesession.cpp` (new-game bring-up, ~line 113):

OLD:
```cpp
  if(!testMode)
    initScripts(true);
  wrld->triggerOnStart(true);
```
NEW:
```cpp
  // NOTE: in original-game the world-start trigger fire is zCTriggerWorldStart::PostLoad @0x0061a4e0,
  // run during the vob post-load pass of world load; oCGame::LoadWorldStartup @0x0061c9c10 only calls
  // CallScriptStartup @0x006c1c70 / CallScriptInit @0x006c1f60 at its tail. So world-start triggers
  // fire BEFORE startup_<world>/init_<world>. Fire first, then run the scripts.
  wrld->triggerOnStart(true);
  if(!testMode)
    initScripts(true);
```

`game/game/gamesession.cpp` (changeWorld, ~line 417):

OLD:
```cpp
  initScripts(wss.isEmpty());
  wrld->triggerOnStart(wss.isEmpty());
```
NEW:
```cpp
  // NOTE: see zCTriggerWorldStart::PostLoad @0x0061a4e0 vs oCGame::LoadWorldStartup @0x0061c9c10 tail
  // (CallScriptStartup @0x006c1c70). World-start triggers fire before the world's startup/init scripts.
  wrld->triggerOnStart(wss.isEmpty());
  initScripts(wss.isEmpty());
```

Leave the savegame-resume call at line 177 unchanged (resume runs no startup script, matching
oCGame::LoadWorldStat @0x006ca010).
