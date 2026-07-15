# Binding Moves

## Summary

- "Binding" is not one mechanism — it is three distinct `freeze_class_status` values (monster info offset `0xC4`)
- Shadow Hold (`0xC4 = 2`), Wrap/Wrapped (`0xC4 = 3/4`), and Constriction (`0xC4 = 7`) each have separate infliction functions
- Only Constriction deals per-tick damage and plays a per-tick animation
- The constriction animation is selected by move ID and stored at info `0xC8`, then replayed each tick
- Fire Spin, Bind, Sand Tomb, Magma Storm, and Clamp share one handler but use different animations

## Move → Status Class Mapping

| Move | Handler | Status Class (0xC4) | Tick Damage | Per-Tick Anim |
|------|---------|---------------------|-------------|---------------|
| Block | `DoMoveBlock` | 2 (Shadow Hold) | No | None |
| Mean Look | `DoMoveShadowHold` | 2 (Shadow Hold) | No | None |
| Spider Web | `DoMoveShadowHold` | 2 (Shadow Hold) | No | None |
| Wrap | `DoMoveWrap` | 3/4 (Wrap/Wrapped) | No (Wrapped side heals) | None |
| Whirlpool | `DoMoveWhirlpool` | 7 (Constriction) | Yes | `0x3b` |
| Clamp | `DoMoveDamageConstrict10` | 7 (Constriction) | Yes | move-ID dependent |
| Bind | `DoMoveDamageConstrict10` | 7 (Constriction) | Yes | move-ID dependent |
| Sand Tomb | `DoMoveDamageConstrict10` | 7 (Constriction) | Yes | move-ID dependent |
| Fire Spin | `DoMoveDamageConstrict10` | 7 (Constriction) | Yes | move-ID dependent |
| Magma Storm | `DoMoveDamageConstrict10` | 7 (Constriction) | Yes | move-ID dependent |

## Shadow Hold (0xC4 = 2)

Block, Mean Look, and Spider Web collapse to two wrappers, both calling `TryInflictShadowHoldStatus(attacker, defender, false)`.

**Evidence:** `DoMoveBlock` / `DoMoveShadowHold`
```c
bool DoMoveBlock(entity *attacker,entity *defender,move *move,item_id item_id)
{
  TryInflictShadowHoldStatus(attacker,defender,'\0');
  return '\x01';
}
```

No application VFX (`FUN_022e42e0` is a no-op), no persistent icon, no tick damage. The monster simply cannot move.

Non-move sources of Shadow Hold: `TryActivateNondamagingDefenderExclusiveItem`, `TryActivateNondamagingDefenderAbility`, and `ApplyGrimyFoodEffect`.

## Wrap / Wrapped (0xC4 = 3/4)

`DoMoveWrap` → `TryInflictWrappedStatus(attacker, defender)`. Gives the **user** the Wrap status (`0xC4 = 3`, wrapping the foe) and the **target** the Wrapped status (`0xC4 = 4`).

**Evidence:** `TryInflictWrappedStatus` (infliction body)
```c
*(undefined *)((int)pvVar6 + 0xc4) = 3;   // user: Wrap
*(undefined *)((int)pvVar6 + 0xcc) = 0x7f;
*(undefined *)((int)pvVar7 + 0xc4) = 4;   // target: Wrapped
```

Both entities store a shared reference at info `0xB4` (linking wrapper to wrapped). `FUN_022e4290` (application VFX) is a no-op. No persistent icon.

Per the tick handler, the Wrap side (`0xC4 = 4`) deals `DAMAGE_MESSAGE_WRAP`, while the Wrapped/Ingrain side (`0xC4 = 5`) **heals** HP.

> See `status_visual_pipeline.md` for the per-tick behavior of each freeze-class value

## Constriction (0xC4 = 7)

The only binding class that deals damage AND plays a recurring animation. Infliction stores an `animation_id` at info `0xC8`.

**Evidence:** `TryInflictConstrictionStatus`
```c
*(undefined *)((int)pvVar3 + 0xc4) = 7;
iVar2 = CalcStatusDuration(target,turn_range,'\x01');
*(char *)((int)pvVar3 + 0xcc) = (char)iVar2 + '\x01';
*(int *)((int)pvVar3 + 200) = animation_id;   // 200 = 0xC8
```

### Per-Tick Animation Replay

The tick handler `FUN_0230fc24` replays the stored animation each damage cycle, BEFORE dealing damage.

**Evidence:** `FUN_0230fc24` constriction arm (`0xC4 == 7`)
```c
*(char *)(uVar18 + 0xcd) = (char)*DAT_02310ad0;
TryEndPetrifiedOrSleepStatus((entity *)param_1,(entity *)param_1);
PlayEffectAnimationEntityStandard((entity *)param_1,*(int *)(uVar18 + 200));  // 0xC8
ApplyDamageAndEffectsWrapper((entity *)param_1,(int)*DAT_02310ad4,
                             DAMAGE_MESSAGE_CONSTRICTION,(damage_source)0x248);
```

This is the only status tick that plays an effect animation.

## Constriction Animation Selection

Fire Spin, Bind, Sand Tomb, Magma Storm, and Clamp share `DoMoveDamageConstrict10`, which picks the `animation_id` by move ID at runtime. Whirlpool uses a separate handler (`DoMoveWhirlpool`) with a hardcoded `0x3b`.

**Key result:** these moves do NOT all play the same animation. The shared handler branches on move ID.

**Evidence:** `DoMoveDamageConstrict10`
```c
uVar4 = (uint)(ushort)move->id;
if (uVar4 == DAT_0232b69c || uVar4 == 0x20c) {
    EndFrozenStatus(attacker,defender);   // fire binds thaw the target on contact
    animation_id = 0x13c;
}
else if (uVar4 == 0x45) {
    animation_id = 0x75;
}
else if (uVar4 == 0x7d) {
    animation_id = 0x7e;
}
else {
    animation_id = 0xf1;   // default
}
iVar2 = DealDamage(attacker,defender,move,0x100,item_id);
if (iVar2 != 0) {
    bVar1 = DungeonRandOutcomeUserTargetInteraction(attacker,defender,(int)*DAT_0232b6a0);
    if (bVar1 != '\0') {
        TryInflictConstrictionStatus(attacker,defender,animation_id,'\0');
    }
}
```

### Animation ID by Move

| move->id | animation_id | Notes |
|----------|--------------|-------|
| `DAT_0232b69c` or `0x20c` | `0x13c` (316) | Calls `EndFrozenStatus` first (fire-type binds thaw the target) |
| `0x45` (69) | `0x75` (117) | — |
| `0x7d` (125) | `0x7e` (126) | — |
| else (default) | `0xf1` (241) | — |
| (Whirlpool, separate handler) | `0x3b` (59) | Hardcoded in `DoMoveWhirlpool` |

The constriction proc chance comes from `DAT_0232b6a0` (Whirlpool uses `DAT_023267d4`).

## FreeOtherWrappedMonsters

Inflicting a new freeze-class status on a monster already wrapping another (`0xC4 == 3 or 4`) releases the wrapped monster via `FreeOtherWrappedMonsters(*(entity **)(info + 0xB4))`. Present in `TryInflictShadowHoldStatus`, `TryInflictConstrictionStatus`, and the `EndFrozenClassStatus` cure path.

## Open Questions

- Value of `DAT_0232b69c` — the move ID sharing the `0x13c` animation branch with `0x20c` (one of the fire-type binds)
- Move name resolution for IDs `0x45`, `0x7d`, `0x20c`, and `DAT_0232b69c` (map via waza_p.bin)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `DoMoveBlock` | `0x02329854` | Block → Shadow Hold |
| `DoMoveShadowHold` | — | Spider Web / Mean Look → Shadow Hold |
| `DoMoveWrap` | `0x0232b6b8` | Wrap → Wrapped |
| `DoMoveWhirlpool` | `0x02326750` | Whirlpool → Constriction (`0x3b`) |
| `DoMoveDamageConstrict10` | `0x0232b5e8` | Clamp/Bind/Sand Tomb/Fire Spin/Magma Storm → Constriction |
| `TryInflictShadowHoldStatus` | — | Inflict Shadow Hold (`0xC4 = 2`) |
| `TryInflictWrappedStatus` | — | Inflict Wrap/Wrapped (`0xC4 = 3/4`) |
| `TryInflictConstrictionStatus` | — | Inflict Constriction (`0xC4 = 7`), stores anim at `0xC8` |
| `FUN_0230fc24` | `0x0230fc24` | Per-turn tick; replays constriction anim from `0xC8` |
| `PlayEffectAnimationEntityStandard` | — | Plays the per-tick constriction animation |
| `FreeOtherWrappedMonsters` | — | Releases wrapped monster when wrapper's status changes |

## Cross-References

> See `Systems/move_dispatch.md` for how these handlers are reached

> See `status_visual_pipeline.md` for freeze-class tick behavior and application VFX

> See `status_icon_system.md` — none of the binding statuses have a persistent icon
