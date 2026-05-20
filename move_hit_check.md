# MoveHitCheck (Accuracy Roll)

## Summary

- Returns `true` if a move hits, `false` if it misses
- **Called twice per target** — once with `use_second_accuracy = false`, then with `= true`
- Each call performs an independent `DungeonRandInt(100)` roll
- Both rolls must pass for the move to land
- Auto-hit shortcuts bypass the roll entirely
- Final formula combines move accuracy with gender-indexed stage tables

## Signature

```c
bool MoveHitCheck(entity *attacker, entity *defender, move *move,
                  bool use_second_accuracy, bool never_miss_self);
```

- `use_second_accuracy`: reads `accuracy2` (true) or `accuracy1` (false) via `GetMoveAccuracyOrAiChance(move, use_second_accuracy)`
- `never_miss_self`: if true AND `attacker == defender`, auto-hit

## Dual-Roll System

The function comment in the ROM states it is called twice per target. The first call (in `ExecuteMoveEffect`'s loop, with `use_second_accuracy='\0'`) checks `accuracy1`. The second call (inside `PerformDamageSequence`, with `use_second_accuracy='\x01'`) checks `accuracy2`. **Both must succeed** for the move to hit.

This means PMD moves carry two accuracy values in `waza_p.bin`. Examples:
- `accuracy1=100, accuracy2=100` → never misses
- `accuracy1=100, accuracy2=85` → rarely grazes (passes accuracy1, sometimes fails accuracy2)
- `accuracy1=70, accuracy2=100` → unreliable but consistent damage when it lands

Each roll is independent (`DungeonRandInt(100)` per call).

## Auto-Hit / Auto-Miss Shortcuts

Checked in order. First match wins, no further modifiers applied.

| Order | Condition | Result |
|---|---|---|
| 1 | `never_miss_self && attacker == defender` | Hit |
| 2 | `move->id == DAT_02324010` AND `IqSkillIsEnabled(attacker, IQ_SURE_HIT_ATTACKER)` | Hit |
| 3 | `attacker->info[0xEC] == 1` (Lock-On / Mind Reader stored state) | Hit |
| 4 | `attacker->info[0xEC] == 2` (forced miss debuff) | Miss |
| 5 | `accuracy >= 0x65` (101+) | Hit |
| 6 | `move->id == 0x40` (Thunder) AND `GetApparentWeather == WEATHER_RAIN` | Hit |
| 7 | `move->id == DAT_0232401c` (Blizzard) AND `GetApparentWeather == WEATHER_HAIL` | Hit |
| 8 | `ABILITY_NO_GUARD` on attacker OR defender | Both stages forced to 10 (neutral), roll continues |

## Accuracy Modifiers

Applied to `accuracy` value (`uVar12`) before the roll:

| Modifier | Source | Adjustment |
|---|---|---|
| Detect Band | `HasHeldItem(defender, ITEM_DETECT_BAND)` (and not Klutz) | `accuracy -= DAT_02324014` |
| Quick Dodger | `IqSkillIsEnabled(defender, IQ_QUICK_DODGER)` | `accuracy -= DAT_02324018` |

## Stage Modifiers (Accuracy Side)

Applied to attacker's accuracy stage (`info[0x2C]`):

| Modifier | Source | Adjustment |
|---|---|---|
| Compound Eyes | `ABILITY_COMPOUNDEYES` on attacker | `+2` |
| Thunder in sun | `move->id == 0x40` AND weather sunny | `-2` |
| Concentrator (attacker) | `IqSkillIsEnabled(attacker, IQ_CONCENTRATOR)` | `+1` |

## Stage Modifiers (Evasion Side)

Applied to defender's evasion stage (`info[0x2E]`, or 10 if `info[0xFE] != 0`):

| Modifier | Source | Adjustment |
|---|---|---|
| Sand Veil | `DefenderAbilityIsActive(SAND_VEIL)` AND weather sandstorm | `+2` |
| Snow Cloak | `DefenderAbilityIsActive(SNOW_CLOAK)` AND weather hail/snow | `+2` |
| Hustle | `AbilityIsActiveVeneer(attacker, HUSTLE)` AND `!MoveIsNotPhysical(move)` | `+2` |
| Clutch Performer | `IqSkillIsEnabled(defender, IQ_CLUTCH_PERFORMER)` AND defender HP ≤ max/4 | `+2` |
| Tangled Feet | `DefenderAbilityIsActive(TANGLED_FEET)` AND target confused (`info[0xD0]==2`) or cringing (`info[0xF1]==2`) | `+3` |
| Concentrator (defender) | `IqSkillIsEnabled(defender, IQ_CONCENTRATOR)` | `-1` |
| Weather exclusive item | `*(DAT_02324024 + weather) != 0` AND `ExclusiveItemEffectIsActive(defender, ...)` | `+1` |

## Stage Clamping

Both stages clamped to `[0, 20]` (0x14) before lookup.

## Gender-Indexed Stage Tables

After clamping, stages index into per-gender tables:

```c
local_2c = (attacker_gender == FEMALE) ? 1 : 0;
local_30 = (defender_gender == FEMALE) ? 1 : 0;

accuracy_multiplier = *(int *)(local_2c * 0xA8 + DAT_02324028 + accuracy_stage * 4);
evasion_multiplier  = *(int *)(local_30 * 0xA8 + DAT_0232402c + evasion_stage * 4);
```

Each table is 0xA8 bytes (21 stages × 4 bytes × 2 genders). Multipliers clamped to `[0, 0x6400]`.

## Final Formula

```c
bool hit = (roll < ((accuracy * accuracy_multiplier) >> 8) * evasion_multiplier >> 8);
```

Where `roll = DungeonRandInt(100)`.

## Call Sites

| Caller | `use_second_accuracy` | Context |
|---|---|---|
| `ExecuteMoveEffect` per-target loop | `false` | First accuracy check (`accuracy1`) |
| `PerformDamageSequence` | `true` | Second accuracy check (`accuracy2`) |

## Cross-References

> See `Systems/move_effect_pipeline.md` for `ExecuteMoveEffect` first call site

> See `Systems/damage_visual_pipeline.md` for `PerformDamageSequence` second call site

> See `move_target_resolution.md` for `TwoTurnMoveForcedMiss` (a separate invincibility check, not part of accuracy)

## Open Questions

- Exact move ID at `DAT_02324010` (Sure-Hit IQ skill target — likely a specific move)
- Exact move ID at `DAT_0232401c` (Blizzard candidate)
- Values at `DAT_02324014` (Detect Band penalty) and `DAT_02324018` (Quick Dodger penalty)
- Full contents of `DAT_02324024` (weather → exclusive item mapping table)
- Full contents of `DAT_02324028` / `DAT_0232402c` (gendered stage multiplier tables, 0xA8 bytes each)

## Functions Used

| Function | Address (NA) | Purpose |
|---|---|---|
| `MoveHitCheck` | — | Accuracy roll (this function) |
| `GetMoveAccuracyOrAiChance` | — | Reads accuracy1 or accuracy2 by flag |
| `DungeonRandInt` | — | RNG roll 0..99 |
| `GetMonsterGenderVeneer` | — | Gender lookup for table index |
| `AbilityIsActiveVeneer` | — | Ability check (attacker side) |
| `DefenderAbilityIsActive` | — | Ability check (defender side, suppression-aware) |
| `IqSkillIsEnabled` | — | IQ skill check |
| `HasHeldItem` | — | Item check (Detect Band) |
| `GetApparentWeather` | — | Weather lookup |
| `ExclusiveItemEffectIsActive` | — | Exclusive item effect check |
| `MoveIsNotPhysical` | — | Physical/special category check |
