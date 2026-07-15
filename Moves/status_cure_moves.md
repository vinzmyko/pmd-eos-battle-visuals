# Status Cure Moves

## Summary

- Refresh, Heal Bell, and Aromatherapy all share one handler: `DoMoveHealStatus`
- There is no dedicated `DoMoveRefresh` — the three moves point at the same function
- The handler calls `EndNegativeStatusConditionWrapper` — team-wide vs self-only comes from waza_p target range, not the code
- Cure moves are visually silent except for the universal cure sparkle (`FUN_022e543c`)
- The `EndNegativeStatusCondition` family is also driven by abilities, berries, and terrain — not just moves

## DoMoveHealStatus

**Evidence:** `DoMoveHealStatus` (header confirms all three moves)
```c
/* Move effect: Heal the team's status conditions
   Relevant moves: Aromatherapy, Heal Bell, Refresh */
bool DoMoveHealStatus(entity *attacker,entity *defender,move *move,item_id item_id)
{
  EndNegativeStatusConditionWrapper(attacker,defender,'\x01','\0');
  return '\x01';
}
```

The wrapper is passed `animation = 1`, `fail_message = 0`. It cures whichever entity it is handed; the move's target-and-range in waza_p decides whether that's the user alone (Refresh) or the whole team (Heal Bell / Aromatherapy). `BuildMoveTargetList` expands the team range before the handler runs.

## EndNegativeStatusCondition

The cure-all. Iterates every negative status class and calls the matching `End*ClassStatus` for any that are active.

Signature:
```c
bool EndNegativeStatusCondition(entity *user, entity *target,
                                bool animation, bool fail_message, bool remove_wrapping);
```

`EndNegativeStatusConditionWrapper` calls it with `remove_wrapping = false`.

**Statuses cured (in order):** sleep class → burn class → freeze class (skipped for Wrap `0xC4 == 3` when `remove_wrapping` is false) → cringe class → curse class → leech seed → sure shot → blinker → muzzled → miracle eye. Also clears the `0x106` and `0xFE` flags, resets speed-related counters at `0x119`, and recalculates speed stage.

### Cure Sparkle (FUN_022e543c)

The only VFX in the cure path. Gated on the `animation` parameter.

**Evidence:** `EndNegativeStatusCondition`
```c
if (bVar2) {
    if (animation != '\0') {
        FUN_022e543c((int *)target);   // cure sparkle
    }
    // ...
}
```

`FUN_022e543c` is the universal status-cure animation. No cure move or `End*ClassStatus` function plays any effect.bin animation beyond this — cures are otherwise silent. (The one exception across the codebase is `EndFrozenClassStatus` case 1, which fires the thaw effect `0x18` — documented under freeze.)

## Non-Move Cure Sources

The `EndNegativeStatusCondition` family is reached from many non-move paths. Complete caller map:

| Caller | Type | Cures |
|--------|------|-------|
| `DoMoveHealStatus` | Move | Refresh / Heal Bell / Aromatherapy |
| `DoMoveHealingWish` | Move | Healing Wish (heal + cure) |
| `DoMoveLunarDance` | Move | Lunar Dance (heal + PP + cure) |
| `DoMoveTag0x1A7` | Move | Tag 0x1A7 (cure + heal + PP restore) |
| `ApplyItemEffect` case `0x45` | Item | Cure-all berry/seed |
| `ApplyItemEffect` case `0x5c` | Item | Cure-all variant |
| `FUN_0230fc24` (Shed Skin, Hydration, watery terrain) | Tick | Passive cures |
| `FUN_0234a980` | Cutscene | Story-driven cure |

Individual `End*ClassStatus` functions have their own additional callers:

| Function | Non-move callers |
|----------|------------------|
| `EndBurnClassStatus` | `DoMoveSmellingSalt`, `ApplyCheriBerryEffect`, `TryEndStatusWithAbility` (Limber/Water Veil/Immunity), `FUN_022f9c74` (lava terrain), per-turn duration tick |
| `EndCringeClassStatus` | `TryEndStatusWithAbility` (Own Tempo / Oblivious) |
| `EndFrozenClassStatus` | `TryEndStatusWithAbility` (Magma Armor / Run Away) |
| `EndSleepClassStatus` | `TryEndStatusWithAbility` (Insomnia / Vital Spirit) |

## TryEndStatusWithAbility

Called after an ability change (Skill Swap, Role Play, Trace) to strip statuses the new ability should prevent. Checks Limber (paralysis), Own Tempo (confusion), Water Veil (burn), Oblivious (infatuation), Insomnia/Vital Spirit (sleep), Magma Armor (freeze), Immunity (poison), Forecast, Run Away.

## Open Questions

- Whether Refresh and the team cures differ in any way beyond waza_p target range (appears not)
- Exact effect ID played by `FUN_022e543c`

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `DoMoveHealStatus` | `0x02326188` | Refresh / Heal Bell / Aromatherapy |
| `EndNegativeStatusCondition` | — | Cure-all (iterates every negative status class) |
| `EndNegativeStatusConditionWrapper` | — | Calls the above with `remove_wrapping = false` |
| `FUN_022e543c` | `0x022e543c` | Universal status-cure sparkle |
| `TryEndStatusWithAbility` | — | Ability-triggered cures after ability change |
| `EndBurnClassStatus` | — | Cure burn/poison/badly poisoned/paralysis (`0xBF`) |

## Cross-References

> See `Systems/move_dispatch.md` for how `DoMoveHealStatus` is reached

> See `status_visual_pipeline.md` for the cure sparkle in the Stage 4 (expiry/cure) pipeline

> See `effect_termination.md` for the `End*ClassStatus` cleanup pattern
