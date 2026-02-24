# Damage Visual Pipeline

## Summary

- Damage visuals are handled inside `ApplyDamage`, completely separate from the move effect system
- The hurt animation (group 6) and a type-matchup-based hit reaction effect play after HP is subtracted
- Effect ID is selected by `damage_data->type_matchup` (not a fixed generic effect)
- Target is in idle during `PlayMoveAnimation` — no freeze/pause mechanism exists
- A 10-frame hold follows the hurt reaction, then the entity resets to idle (or faints)
- Hunger damage (`DAT_02309fdc`) skips the hurt animation entirely

## Call Chain

```
ExecuteMoveEffect
  │
  ├─► PlayMoveAnimation / FUN_023258ec  (visual effects only, no damage)
  │
  └─► switch(move_id)
        └─► DoMoveDamage
              └─► DealDamage
                    └─► PerformDamageSequence
                          └─► ApplyDamageAndEffects
                                └─► ApplyDamage  ← hurt visuals live here
```

## ApplyDamage Hurt Sequence

After HP subtraction, if the defender should display and the damage source is not hunger:

```c
// 1. Face attacker
dVar20 = GetDirectionTowardsPosition(&defender->pos, &attacker->pos);
*(byte *)((int)pvVar33 + 0x4c) = (byte)dVar20 & 7;

// 2. Play hurt animation (CHARA group 6)
ChangeMonsterAnimation(defender, '\x06', direction);

// 3. Play type-matchup hit reaction effect (blocking)
FUN_022e5478((int *)defender, (int)damage_data);

// 4. 10-frame hold
FUN_022ea370(10, 0x18, ...);

// 5a. If survived:
FUN_02304a48((int *)defender, 8);  // Reset to idle, keep facing

// 5b. If fainted (hp == 0):
defender->damage_visual_effect = (damage_visual_8)0x1;
FUN_022ea370(0x1e, 0x18, ...);  // 30-frame hold
// → HandleFaint(...)
```

### Target State During Move Effects

The target is in idle during `PlayMoveAnimation`. There is no freeze, pause, or dedicated flinch state. The hurt animation only begins after the move effect pipeline completes and damage is applied inside the switch statement.

## Hit Reaction Effect (FUN_022e5478)

Selects an effect ID based on `damage_data->type_matchup` (offset 0x08 in damage_data).

**Evidence:** `FUN_022e5478`
```c
void FUN_022e5478(int *param_1, int param_2)
{
    int iVar1 = *(int *)(param_2 + 8);  // damage_data->type_matchup

    // IQ skill or ability may hide effectiveness
    if (*(char *)(*DAT_022e5508 + 0x1c5) != '\0') {
        iVar1 = FUN_0230d618(iVar1);
    }

    switch(iVar1) {
    case 0: default: iVar1 = 8;  break;  // Neutral
    case 1:          iVar1 = 9;  break;  // Not very effective
    case 2:          iVar1 = 10; break;  // Super effective
    case 3:          iVar1 = 11; break;  // Immune
    }

    PlayEffectAnimationEntity(
        (entity *)param_1,
        iVar1,       // effect_id
        '\x01',      // play_now (blocking, 100-frame timeout)
        3,           // unknown
        0,           // unknown
        '\x01',      // unknown
        -1,          // unknown
        (undefined2 *)0x0
    );
}
```

### Effect ID Table

| type_matchup | Value | Effect ID | Expected Visual |
|--------------|-------|-----------|-----------------|
| MATCHUP_NEUTRAL | 0 | 8 | Neutral hit |
| MATCHUP_NOT_VERY_EFFECTIVE | 1 | 9 | Resisted hit |
| MATCHUP_SUPER_EFFECTIVE | 2 | 10 | Super effective hit |
| MATCHUP_IMMUNE | 3 | 11 | Immune hit |

**Note:** Effects 8-11 need visual verification in extracted sprite data (`asset_index.json`).

**Note:** `FUN_0230d618` overrides the matchup value when dungeon flag `+0x1C5` is set, likely the Type-Bulldozer IQ skill or similar mechanic that hides type effectiveness from the player.

### Playback Behavior

`PlayEffectAnimationEntity` is called with `play_now = '\x01'`, which triggers the blocking 100-frame timeout wait loop. The hurt reaction effect plays to completion (or timeout) before the 10-frame hold begins.

## Reset to Idle (FUN_02304a48)

Called after hurt animation with `param_2 = 8`. Resets animation to idle, does not change direction.

**Evidence:** `FUN_02304a48`
```c
void FUN_02304a48(int *param_1, int param_2)
{
    if (*param_1 != 1) return;  // Must be ENTITY_MONSTER

    uint8_t uVar1 = GetIdleAnimationId((entity *)param_1);
    *(uint8_t *)((int)param_1 + 0xae) = uVar1;  // Set animation group to idle
    *(undefined *)(param_1 + 10) = 0;            // Clear animation state

    if (param_2 < 0) return;
    if (param_2 < 8) {
        *(char *)(param_1 + 0x2c) = (char)param_2;  // Set direction
    }
    // param_2 >= 8: direction unchanged
}
```

**When called with param_2 = 8:** Direction branch is skipped (8 is not < 8). The defender keeps facing the attacker (set earlier by `GetDirectionTowardsPosition`).

## Miss/Block Sound (FUN_022e576c)

Plays a sound effect on miss or block. Not a visual — used for Sturdy, frozen, Magic Guard cases.

**Evidence:** `FUN_022e576c`
```c
void FUN_022e576c(undefined4 param_1, int param_2)
{
    if (*(char *)(*(int *)(param_2 + 0xb4) + 6) != '\0') {
        PlaySeByIdIfNotSilence(DAT_022e5798);  // Team leader miss sound
    } else {
        PlaySeByIdIfNotSilence(DAT_022e579c);  // Normal miss sound
    }
}
```

Called in `ApplyDamage` for:
- Magic Guard blocking non-move damage
- Sturdy preventing OHKO
- Frozen status blocking damage
- Zero damage dealt

Also called in `PerformDamageSequence` when `MoveHitCheck` fails (move misses).

## Faint Sequence

When `monster->hp == 0` after damage:

```c
// Special case: if damage_source is hunger, play hurt anim now
// (was skipped earlier for hunger)
if (damage_source.move == DAT_02309fdc) {
    ChangeMonsterAnimation(defender, '\x06', direction);
    FUN_022e5478((int *)defender, (int)damage_data);
    FUN_022ea370(10, 0x18, ...);
}

// End invisibility if active
if (monster->statuses.invisible == 0x2) {
    EndInvisibleClassStatus(attacker, defender, '\0');
}

// Faint visual
defender->damage_visual_effect = (damage_visual_8)0x1;
FUN_022ea370(0x1e, 0x18, ...);  // 30-frame hold (~0.5s)

// → HandleFaint(defender, damage_source, attacker)
```

### damage_visual_effect Values

| Value | Meaning | Context |
|-------|---------|---------|
| 0x0 | None | Normal state / cleared after revive |
| 0x1 | Faint | Set before HandleFaint |
| 0x2 | Revive tile | Set when fainting on a revive tile |

## Timing Summary

| Phase | Frames | Duration (60fps) |
|-------|--------|-------------------|
| Hurt animation (group 6) starts | — | Immediate |
| Hit reaction effect (blocking) | Variable | Up to 100 frames max |
| Post-hurt hold | 10 | ~0.167s |
| Reset to idle | — | Immediate |
| Faint hold (if hp == 0) | 30 | ~0.5s |

## Hunger Damage Special Case

`DAT_02309fdc` is the hunger damage source. When damage comes from hunger:
- Hurt animation is **skipped** during the normal sequence
- Animated damage numbers are **skipped**
- If the Pokémon faints from hunger, hurt anim plays at that point instead
- Direction facing toward attacker is skipped (no attacker for hunger)

## Cross-References

> See `Systems/move_effect_pipeline.md` for the move animation system that precedes damage

> See `Data Structures/animation_control.md` for CHARA animation group 6 (Hurt)

> See `Systems/animation_timing.md` for `FUN_022ea370` frame delay mechanics

> See `effect_termination.md` for effect lifecycle and `PlayEffectAnimationEntity` blocking behavior

## Rename: is_hit_frame → is_effect_trigger_frame

The WAN frame flag `(flags & 2)` in `FUN_023250d4` triggers `FUN_02325644`, which spawns charge + secondary effects. It does not trigger damage or the hurt animation. Damage is applied in a completely separate code path via the move effect switch statement inside `ExecuteMoveEffect`.

**Affected code:**
- `FUN_023250d4`: reads flag via `FUN_0201d1d4`, uses bit 1 to call `FUN_02325644`
- Client: `animation_sequencer.gd` callback naming
- Client: `pokemon.gd` signal naming
- Context handoff document

## Open Questions

- Visual verification of effects 8-11 in extracted data
- What `FUN_0230d618` maps matchup values to when effectiveness is hidden
- Whether `PlayEffectAnimationEntity` params 3-7 affect rendering (position, z-order, etc.)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `ApplyDamage` | — | Main damage application, contains hurt visual sequence |
| `ApplyDamageAndEffects` | — | Wrapper adding counter damage and contact abilities |
| `PerformDamageSequence` | — | Hit check + ApplyDamageAndEffects + Illuminate |
| `DealDamage` | — | Calc damage + PerformDamageSequence |
| `DoMoveDamage` | — | DealDamage with multiplier 1 |
| `FUN_022e5478` | `0x022e5478` | Hit reaction effect (matchup-based effect ID) |
| `FUN_02304a48` | `0x02304a48` | Reset to idle animation |
| `FUN_022e576c` | `0x022e576c` | Miss/block sound effect |
| `ChangeMonsterAnimation` | — | Set animation group + direction |
| `GetDirectionTowardsPosition` | — | Calculate facing direction |
| `PlayEffectAnimationEntity` | — | Spawn + block on effect animation |
| `HandleFaint` | — | Faint processing, removal, rewards |
| `FUN_0230d618` | `0x0230d618` | Override matchup for hidden effectiveness |
