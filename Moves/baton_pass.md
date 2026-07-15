# Baton Pass

## Summary

- EoS Baton Pass is a POSITION SWAP, not a stat-buff transfer (unlike the main series)
- Move 252 dispatches to `DoMoveSwitchPositions` → `TrySwitchPlace`
- Shares its handler and mechanic with the Switcher Orb item effect
- Suction Cups (on user or target) blocks the swap; Mold Breaker bypasses the target's Suction Cups
- No stat stages, multipliers, or status are transferred — only tile/pixel position
- Visually silent: the effect hooks (`FUN_022e555c`) are no-ops

## Dispatch

Move ID 252 (`0xFC`) → jump table entry at `0x0232fca8` → trampoline at `0x023315d8` → `bl DoMoveSwitchPositions`.

> See `Systems/move_dispatch.md` for the jump-table decode

## DoMoveSwitchPositions

**Evidence:** `DoMoveSwitchPositions` (header confirms shared use)
```c
/* Move effect: Switches the user's position with positions of other monsters in the room.
   Relevant moves: Baton Pass, Switcher (item effect) */
bool DoMoveSwitchPositions(entity *attacker,entity *defender,move *move,item_id item_id)
{
  if (*(char *)((int)attacker->info + 0x108) == '\0') {
    *(undefined *)((int)attacker->info + 0x108) = 1;
  }
  TrySwitchPlace(attacker,defender);
  return '\x01';
}
```

## TrySwitchPlace

Performs the actual swap after ability checks.

**Ability gating:**
1. **User Suction Cups** → fails outright, logs a message, returns
2. **User Mold Breaker** → bypasses the target's Suction Cups check
3. **Target Suction Cups** (unless bypassed) → fails, logs a message

**The swap:**
```c
sVar1 = (user->pos).x;   sVar2 = (user->pos).y;
sVar3 = (target->pos).x; sVar4 = (target->pos).y;
FUN_022e555c();   // no-op (stubbed effect hook)
FUN_022e555c();   // no-op
MoveMonsterToPos(user,   (int)sVar3,(int)sVar4,'\x01');
MoveMonsterToPos(target, (int)sVar1,(int)sVar2,'\x01');
UpdateEntityPixelPos(user,(pixel_position *)0x0);
UpdateEntityPixelPos(target,(pixel_position *)0x0);
FUN_02321260(user);
FUN_02321260(target);
```

Only position fields are exchanged. There is no access to stat stages (`0x24`–`0x2E`), multipliers (`0x34`–`0x40`), or any status field. The main-series "pass boosts to a switch-in" behavior does not exist in EoS.

## Visuals

- `FUN_022e555c` is a no-op (`return;`) — a stubbed/removed effect hook. Both calls do nothing.
- `FUN_02321260` is a generic post-move animation/position settle (also called twice by `DoMoveSplash`), not Baton-Pass-specific.
- Net on-screen: the two monsters swap tiles with normal movement/redraw and no special animation.

## Open Questions

- What `FUN_022e555c` was intended to play before being stubbed
- Whether Baton Pass can target non-adjacent room members (header says "other monsters in the room")

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `DoMoveSwitchPositions` | `0x0232c538` | Baton Pass / Switcher → position swap |
| `TrySwitchPlace` | — | Ability checks + position exchange |
| `MoveMonsterToPos` | — | Move an entity to a tile |
| `UpdateEntityPixelPos` | — | Recompute pixel position after move |
| `FUN_022e555c` | `0x022e555c` | Stubbed effect hook (no-op) |
| `FUN_02321260` | `0x02321260` | Generic post-move settle (shared with Splash) |

## Cross-References

> See `Systems/move_dispatch.md` for dispatch and the jump-table decode of move 252
