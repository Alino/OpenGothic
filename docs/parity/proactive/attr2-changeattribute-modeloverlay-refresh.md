# ChangeAttribute: missing per-change wounded-overlay refresh (CheckModelOverlays)

**Confidence:** High that the divergence exists; **NO surgical fix** (DEFERRED — non-trivial feature, one unresolved gate).

## Scope checked
This pass audited the *engine-side math* of `oCNpc::ChangeAttribute` against
`Npc::changeAttribute`, focusing on the prompted hypotheses. Result: the math is
**faithful**, so those hypotheses are NOT divergences. The only real divergence is a
missing side-effect call, documented below.

## Original fn + address
`oCNpc::ChangeAttribute` @ `Gothic2.exe 0x0072ff60`. Its body, after the godmode /
val==0 / immortal(+0x1b4 bit1, val!=-999) guards, does exactly:

1. `attribute[a] += val`.
2. Negative floor: `if(attribute[a] < 0) attribute[a] = 0` — applied to **every**
   attribute index.
3. Current-to-max clamp, only when the *current* pool was changed:
   - `a == ATR_MANA(2)`: `if(MANAMAX < MANA) MANA = MANAMAX;` then call
     `CheckModelOverlays()` and **return**.
   - `a == ATR_HITPOINTS(0)`: `if(HITPOINTSMAX < HITPOINTS) HITPOINTS = HITPOINTSMAX;`
     then fall through.
4. `oCNpc::CheckModelOverlays()` @ `0x007301d0` is called **on every accepted attribute
   change** (HP, HPMAX, MANA, MANAMAX, STR, DEX, …).

`CheckModelOverlays` (verified by decompiling 0x007301d0) applies/removes the low-HP
"wounded" model overlay (`GetOverlay` builds a `_WOUNDED`-suffixed MDS the same way
OpenGothic's `_TORCH.MDS` overlay is built): it early-returns when `HITPOINTS < 1`,
**applies** the overlay when `HITPOINTS <= 2` (and a vtable-gated human check passes and
`oCAniCtrl_Human::GetWaterLevel <= 1`), and **removes** it once `HITPOINTS > 2` (or the
gate/water condition flips), calling `InitAnimations` on transition and tracking state in
flag `+0x75c & 0x100`.

## Hypotheses explicitly DISPROVEN (faithful in OpenGothic)
- **Raising HITPOINTSMAX/MANAMAX does NOT add the max-delta to current HP/mana.** The
  original has no such code; `a == HITPOINTSMAX(1)`/`MANAMAX(3)` skip both clamp branches.
  OpenGothic matches.
- **Negative floor** (`<0 -> 0`, all attributes): present and matching — `npc.cpp:1309`.
- **Current-to-max clamp** on HP/MANA change: present and matching — `npc.cpp:1311-1314`.
- **Guards** (godmode rejects any negative delta for the player; val==0 no-op; immortal
  blocks all HP change except -999): present and matching — `npc.cpp:1289-1306`.
- **No re-clamp on max change**: lowering HITPOINTSMAX below current HP does NOT clamp
  current in the original; OpenGothic matches.

## Divergence
**OG file:line:** `game/world/objects/npc.cpp:1288-1324` (`Npc::changeAttribute`).
OpenGothic never invokes a `CheckModelOverlays` equivalent. A `grep` over `game/` for
`wounded`/`_WOUNDED.MDS`/`CheckModelOverlays` finds only death-animation transitions in
`graphics/mesh/animationsolver.cpp` — **no low-HP wounded model overlay is ever applied**.
Consequently, in the original an NPC/player whose HP drops to 1-2 gets the limping/wounded
animation overlay (and loses it on recovery above 2) immediately as part of the attribute
change; OpenGothic shows no such overlay.

## Proposed patch
**DEFERRED — not surgical.** A faithful fix would mirror the existing torch-overlay path
(`Npc::setTorch`/`addOverlay`/`delOverlay`, `npc.cpp:846-869`) with a `_WOUNDED`-suffixed
`string_frm`, driven from `changeAttribute` after the HP clamp with the
`HITPOINTS<=2 && !inWater` apply / `HITPOINTS>2` remove transition and an `InitAnimations`
refresh. Two reasons this is held back rather than landed surgically:
1. The apply gate at vtable `+0x100` (a human/animation-capability predicate) and the exact
   `GetWaterLevel <= 1` mapping are not yet pinned to OpenGothic equivalents, so the apply
   condition cannot be reproduced 1:1 with high confidence.
2. It is a new visual feature (overlay add/remove + skeleton re-init on transition), not a
   one-line math correction — it needs its own focused parity pass and in-game verification.

```
// NOTE: in original-game oCNpc::ChangeAttribute (Gothic2.exe 0x0072ff60) every accepted
// attribute change ends with oCNpc::CheckModelOverlays (0x007301d0), which applies the
// "_WOUNDED" model overlay while HITPOINTS<=2 (human, not in deep water) and removes it
// once HITPOINTS>2. OpenGothic's changeAttribute omits this refresh, so the low-HP wounded
// animation overlay is never shown. (DEFERRED: needs the +0x100 human gate + water mapping.)
```
