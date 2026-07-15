# Reflect-Class Moves (Counter / Magic Coat / Mirror Coat)

## Summary

- Counter, Magic Coat, Mirror Coat, and related moves all set the reflect-class status at info offset `0xD5`
- Application handlers are thin: they set `0xD5` and update the status icon (SMA anim 9)
- Application VFX is a no-op for all of them (`FUN_022e3f20` / `FUN_022e3fc8` return immediately)
- Magic Coat's bounce ENFORCEMENT lives in `ExecuteMoveEffect`'s per-target loop, not the handler
- Counter's damage-reflection enforcement is not yet traced

## reflect_class_status Values (info 0xD5)

| Value | Status | Icon (SMA:pal) |
|-------|--------|----------------|
| 4 | Counter | 9:0 (shield) |
| 5 | Magic Coat | 9:3 |
| (see status_icon_system.md for the full 18-entry table) | | |

> See `status_icon_system.md` for the complete reflect-class → icon mapping

## Counter (DoMoveCounter)

**Evidence:** `DoMoveCounter`
```c
bool DoMoveCounter(entity *attacker,entity *defender,move *move,item_id item_id)
{
  SetReflectStatus(attacker,defender,STATUS_REFLECT_COUNTER);
  return '\x01';
}
```

`SetReflectStatus` sets `0xD5 = 4`, sets the duration counter at `0xD6`, calls `FUN_022e3f20` (no-op), logs, and updates icons. Also used by Pursuit and Payback.

## Magic Coat (DoMoveMagicCoat)

**Evidence:** `TryInflictMagicCoatStatus`
```c
*(undefined *)((int)pvVar3 + 0xd5) = 5;
iVar2 = CalcStatusDuration(target,turn_range,'\0');
*(char *)((int)pvVar3 + 0xd6) = (char)iVar2 + '\x01';
FUN_022e3fc8();   // no-op
```

## Magic Coat Enforcement (ExecuteMoveEffect)

The bounce logic runs inside `ExecuteMoveEffect`'s per-target loop, before the hit-check. Two triggers: the target's reflect-class byte being Magic Coat, or the `EXCLUSIVE_EFF_MAY_BOUNCE_STATUS_MOVES` item effect.

**Evidence:** `ExecuteMoveEffect` bounce path
```c
if (*(char *)((int)&peVar26[1].elevation + 1) == '\x05') {   // reflect-class == Magic Coat
    bVar28 = true;
}
// ...
if (bVar28) {
    bVar6 = IsReflectedByMagicCoat(move_id);
    if ((bVar6 != '\0') && (CanHitWithRegularAttack(attacker,peVar13) != '\0')) {
        // log bounce, then re-dispatch the move at the original attacker
        FUN_022e3fcc((int *)peVar13);
        FUN_02333044((int *)peVar13,(int)attacker);
        entity = attacker;   // move now targets the original user
    }
}
```

`IsReflectedByMagicCoat(move_id)` gates which moves can be bounced. When it fires, `FUN_02333044` redirects the move back at the attacker and the loop re-targets `entity = attacker`.

The adjacent branch handles Mirror Move (`MirrorMoveIsActive`), which redirects similarly.

## Open Questions

- Counter's damage-reflection enforcement — `SetReflectStatus` sets `0xD5 = 4` (Counter), but the code path that reads `0xD5 == 4` during damage application and returns damage to the attacker is not yet located. Likely in `ApplyDamage` / `ApplyDamageAndEffects` / `PerformDamageSequence`.
- Purpose of `FUN_022e3fcc` and `FUN_02333044` internals (bounce redirect)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `DoMoveCounter` | `0x02326cb4` | Counter → reflect-class 4 |
| `DoMoveMagicCoat` | `0x0232b7c0` | Magic Coat → reflect-class 5 |
| `DoMoveMirrorCoat` | `0x0232b8d4` | Mirror Coat |
| `SetReflectStatus` | — | Sets `0xD5`, duration, icon (no-op VFX) |
| `TryInflictMagicCoatStatus` | — | Sets `0xD5 = 5` |
| `IsReflectedByMagicCoat` | — | Gates which moves Magic Coat bounces |
| `FUN_02333044` | `0x02333044` | Redirects bounced move at attacker |
| `FUN_022e3f20` | `0x022e3f20` | Counter application VFX (no-op) |
| `FUN_022e3fc8` | `0x022e3fc8` | Magic Coat application VFX (no-op) |

## Cross-References

> See `Systems/move_dispatch.md` for the per-target loop that runs the bounce logic

> See `status_icon_system.md` for reflect-class icon mapping (SMA anim 9)
