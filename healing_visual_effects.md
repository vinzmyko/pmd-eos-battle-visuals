# Healing Visual Effects

## Summary

- HP restoration triggers two visual elements: floating numbers and an effect animation
- Effect animation ID **7** is hardcoded for normal HP restoration
- Max HP boost uses a separate effect ID stored at `DAT_022e447c`
- Both effects play via `PlayEffectAnimationEntity` with `play_now = true` (blocking, 100-frame timeout)

## Call Chain (Drain Moves)

```
DoMoveAbsorb / DoMoveDamageDrain
  └─► TryIncreaseHp(attacker, attacker, hp_amount, 0, true)
        ├─► TryRestoreHp (normal HP path)
        │     └─► FUN_022e4480 → PlayEffectAnimationEntity(entity, 7, ...)
        │
        ├─► Max HP boost path (when HP is full and max_hp_boost > 0)
        │     └─► FUN_022e4430 → PlayEffectAnimationEntity(entity, DAT_022e447c, ...)
        │
        └─► DisplayAnimatedNumbers(amount, target, true, NUMBER_COLOR_AUTO)
              (green floating "+N" numbers)
```

## TryIncreaseHp Visual Breakdown

### Floating Numbers

`DisplayAnimatedNumbers(amount, target, '\x01', NUMBER_COLOR_AUTO)` is called when:
- `val_00 != 0` (HP restored) OR `val != 0` (max HP increased)
- `ShouldDisplayEntityWrapper(target)` returns true

With `NUMBER_COLOR_AUTO` and a positive amount, the auto-color logic selects palette index **10** (green).

### Effect Animation: Normal HP Restore (FUN_022e4480)

```c
void FUN_022e4480(int *param_1)
{
  uint uVar1;
  uVar1 = GetEffectAnimationField0x19(7);
  PlayEffectAnimationEntity((entity *)param_1, 7, '\x01', uVar1 & 0xff, 2, '\0', -1, (undefined2 *)0x0);
}
```

- **Effect ID:** 7 (hardcoded literal)
- **play_now:** true (blocks up to 100 frames)
- **Lookup:** `EFFECT_ANIMATION_INFO[7]` at NA address `0x022CC52C + 7 * 0x1C = 0x022CC5F0`
- **Called from:** `TryRestoreHp` path inside `TryIncreaseHp`

### Effect Animation: Max HP Boost (FUN_022e4430)

```c
void FUN_022e4430(int *param_1)
{
  uint uVar1;
  uVar1 = GetEffectAnimationField0x19(DAT_022e447c);
  PlayEffectAnimationEntity((entity *)param_1, DAT_022e447c, '\x01', uVar1 & 0xff, 2, '\0', -1, (undefined2 *)0x0);
}
```

- **Effect ID:** Constant at ROM address `0x022e447c` (needs ROM read)
- **play_now:** true (blocks up to 100 frames)
- **Called from:** Max HP boost path inside `TryIncreaseHp` (when current HP == effective max HP and max_hp_boost > 0)

## When Each Path Triggers

### Normal HP Restore Path
Triggers when `hp_restoration > 0` and current HP is not already at max. Calls `TryRestoreHp` which plays effect 7.

### Max HP Boost Path
Triggers when either:
- Current HP equals effective max HP AND `max_hp_boost > 0`
- `hp_restoration == 0` AND `max_hp_boost > 0`

Increases base max HP, then plays the max HP boost effect via `FUN_022e4430`.

### For Drain Moves Specifically
`TryIncreaseHp` is called with `max_hp_boost = 0`, so drain moves **always** take the normal HP restore path → effect 7.

## IQ Skill: Wise Healer

Before applying healing, `TryIncreaseHp` checks for `IQ_WISE_HEALER`:
```c
bVar2 = IqSkillIsEnabled(target, IQ_WISE_HEALER);
if (bVar2 != '\0') {
    // hp_restoration increased by DAT_023155f0 percent
}
```

This affects the amount healed but not which visual effect plays.

## Exclusive Item: HP Drain Recovery Boost

In the drain move functions (not in TryIncreaseHp), the recovery amount is doubled before being passed to TryIncreaseHp:
```c
bVar2 = ExclusiveItemEffectFlagTest(..., EXCLUSIVE_EFF_HP_DRAIN_RECOVERY_BOOST);
if (bVar2 != '\0') {
    iVar3 = iVar3 << 1;  // Double recovery
}
```

Gated by `attacker->info + 6 == '\0'` (must not have some condition active).

## Log Messages

`TryIncreaseHp` logs different messages based on outcome:

| Condition | Message ID Source |
|-----------|-------------------|
| No change, max HP boosted | `DAT_023155f8` |
| No change, HP full | `DAT_023155fc` |
| Max HP increased but HP unchanged | `DAT_02315600` |
| Max HP increased | `DAT_02315604` |
| HP restored to full | `DAT_02315608` |
| HP restored (not full) | `DAT_0231560c` |

## Status: Heal Block

If `monster_info + 0xD8 == 0x05` (heal blocked status), `TryIncreaseHp` prints a failure message (`DAT_023155ec`) and returns false immediately — no healing or visual effects.

## Liquid Ooze Path (No Healing)

When Liquid Ooze is active, drain moves skip `TryIncreaseHp` entirely and instead call:
```c
ApplyDamageAndEffectsWrapper(attacker, iVar3, DAMAGE_MESSAGE_SLUDGE, (damage_source)0x239);
```
This deals damage to the attacker with `defender_response = false` (no counter effects).

## Cross-References

- `EFFECT_ANIMATION_INFO` table: see `effect_animation_info.md`
- `PlayEffectAnimationEntity`: see `effect_lifecycle.md`, `animation_timing.md`
- `DisplayAnimatedNumbers`: number color auto-logic selects palette by sign of amount
- Drain move functions: `DoMoveAbsorb`, `DoMoveDamageDrain`

## Next Steps

- Read ROM at `0x022e447c` to get the max HP boost effect ID
- Look up `EFFECT_ANIMATION_INFO[7]` to find which WAN file/animation sequence the heal effect uses
- Verify effect 7 visually in extracted effect.bin sprites

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `TryIncreaseHp` | — | Main HP restoration with visuals |
| `TryRestoreHp` | — | Raw HP write + effect 7 |
| `FUN_022e4480` | `0x022e4480` | Plays effect animation 7 (HP restore) |
| `FUN_022e4430` | `0x022e4430` | Plays max HP boost effect animation |
| `DisplayAnimatedNumbers` | — | Floating green numbers |
| `UpdateStatusIconFlags` | — | Refreshes status icons after HP change |
| `PlayEffectAnimationEntity` | — | Spawns + blocks on effect animation |
| `GetEffectAnimationField0x19` | — | Reads attachment_point from effect table |
