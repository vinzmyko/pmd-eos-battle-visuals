# Move Effect Mechanics (Dungeon Mode)

## Summary

This document covers the move effect dispatch system in PMD:EOS dungeon mode.
It describes how `DoMoveX` wrappers invoke shared mechanic functions, how VFX
is dispatched from within those mechanic functions (not the wrappers), the stat
boost/drop system, taunt enforcement, and related move validation logic.

For status icons and the visual pipeline, see the companion documents
`status_icon_system.md` and `status_visual_pipeline.md`.

---

## DoMoveX Wrapper Pattern

Every move's effect is implemented as a `DoMoveX` function with a uniform signature:

```c
bool DoMoveX(entity *attacker, entity *defender, move *move, item_id item_id);
```

These are **thin wrappers**. They contain no VFX calls, no stat writes, no immunity checks.
They call one or more shared mechanic functions with move-specific parameters and return `true`.

### Examples

| Wrapper | Calls | Parameters |
|---------|-------|------------|
| `DoMoveSwordsDance` | `BoostOffensiveStat(atk, def, stat_idx, 2)` | stat_idx from `DAT_02329550`, 2 stages |
| `DoMoveNastyPlot` | `BoostOffensiveStat(atk, def, stat_idx, 2)` | stat_idx from `DAT_0232e730`, 2 stages |
| `DoMoveAmnesia` | `BoostDefensiveStat(atk, def, stat_idx, 2)` | stat_idx from `DAT_0232a074`, 2 stages |
| `DoMoveCalmMind` | `BoostOffensiveStat(...)` + `BoostDefensiveStat(...)` | stat_idx from `DAT_0232b924`, 1 stage each |
| `DoMoveScreech` | `ApplyDefensiveStatMultiplier(atk, def, stat_idx, 0x40, 1)` | stat_idx from `DAT_0232666c`, multiplier 0x40 |
| `DoMoveTaunt` | `TryInflictTauntStatus(atk, def, false)` | — |
| `DoMoveSetDamage` | `TryInflictSetDamageStatus(atk, def)` | — |

The `DAT_` constants are move-specific parameters (stat index, stage count, multiplier).
The stat index is 0 for physical, 1 for special.

**For undecompiled DoMoveX functions:** Identify what the move should do, find a decompiled
DoMoveX that does the same thing, and expect the same pattern — args loaded into r0–r3/stack
then a `bl` to the shared mechanic function. Match the branch target to a known function.

---

## Stat Boost/Drop System

### Stat Stages (BoostOffensiveStat / BoostDefensiveStat)

Stat stages are `short` values at monster info offsets:

| Offset | Stat |
|--------|------|
| 0x24 | Offensive physical stage |
| 0x26 | Offensive special stage |
| 0x28 | Defensive physical stage |
| 0x2A | Defensive special stage |
| 0x2C | Hit chance stage (accuracy?) |
| 0x2E | Hit chance stage (evasion?) |

Default value is **10**. Range is 0–20 (capped at 0x14). `ABILITY_SIMPLE` doubles the
stage change (`n_stages << 17 >> 16`, which is `n_stages * 2` with sign extension).

If the new stage equals the current stage (already at cap), a "can't go higher" message
is logged instead of applying the change.

`UpdateStatusIconFlags` is called after every stat change.

**Source:** Ghidra decompilation of `BoostOffensiveStat`, `BoostDefensiveStat`.

### Stat Multipliers (ApplyOffensiveStatMultiplier / ApplyDefensiveStatMultiplier)

Multipliers are `int` values at monster info offsets:

| Offset | Stat |
|--------|------|
| 0x34 | Offensive physical multiplier |
| 0x38 | Offensive special multiplier |
| 0x3C | Defensive physical multiplier |
| 0x40 | Defensive special multiplier |

Default value is **0x100** (256, i.e. 1.0× in fixed-point). Minimum is **2**.
Maximum is `DAT_023140d4`. The multiplier is applied via `MultiplyByFixedPoint`.

For drops, the function checks `IsProtectedFromStatDrops` before applying. This check
is skipped for buffs (multiplier >= 0x100).

`UpdateStatusIconFlags` is called after every multiplier change.

**Source:** Ghidra decompilation of `ApplyDefensiveStatMultiplier`.

### VFX Dispatch

VFX is called **inside the mechanic functions**, not in the DoMoveX wrappers.
The `stat_idx` parameter determines which effect.bin animation plays.

For stages:
```
BoostOffensiveStat  → PlayOffensiveStatUpEffect(target, stat_idx)
BoostDefensiveStat  → PlayDefensiveStatUpEffect(target, stat_idx)
```

For multipliers, the direction branches on `multiplier < 0x100`:
```c
if (multiplier < 0x100) {
    PlayDefensiveStatMultiplierDownEffect(target, stat_idx);  // debuff
} else {
    PlayDefensiveStatMultiplierUpEffect(target, stat_idx);    // buff
}
```

The `Play*Effect` functions are thin lookups: stat_idx indexes a two-entry table
(physical/special) and calls `PlayEffectAnimationEntity` with the result.

### Effect.bin Animation IDs (from status_visual_pipeline.md)

| Effect | Physical (stat_idx=0) | Special (stat_idx=1) |
|--------|----------------------|----------------------|
| Offensive Up | `0x1A9` (425) | `0x192` (402) |
| Offensive Down | `0x194` (404) | `0x193` (403) |
| Defensive Up | `0x18E` (398) | `0x190` (400) |
| Defensive Down | `0x18F` (399) | `0x191` (401) |
| Hit Chance Up | `0x18C` (396) | `0x0D` (13) |
| Hit Chance Down | `0x18D` (397) | `0x0E` (14) |

### No Persistent Icon for Stat Buffs

**Confirmed:** `UpdateStatusIconFlags` only checks for stats *below* default
(multiplier < 0x100 or stage < 10) to set bit 27 (`f_lowered_stat`, SMA anim 10 palette 3,
yellow down arrows). There is no corresponding check for stats above default.

Stat buffs produce only the one-shot effect.bin animation at application time —
no persistent overhead icon.

**Source:** `UpdateStatusIconFlags` decompilation at `022e3d28`–`022e3d80`.

---

## Taunt: Full Trace

### Infliction

`DoMoveTaunt` → `TryInflictTauntStatus(attacker, defender, false)`

**Immunity checks (in order):**
1. `SafeguardIsActive(user, target, true)`
2. `IsProtectedFromNegativeStatus(user, target, true)`
3. `ExclusiveItemEffectIsActiveWithLogging(user, target, true, ..., EXCLUSIVE_EFF_NO_MOVE_DISABLING)`

**Application:**
1. If `cringe_class_status` (offset 0xD0) is already 5 → log "already taunted" message
2. Otherwise: set `0xD0 = 5` (CRINGE_TAUNTED)
3. Duration: `CalcStatusDuration(target, turn_range, true) + 1` → written to offset 0xD1
4. `FUN_022e442c()` — confirmed no-op (`bx lr`)
5. Log infliction message
6. `TryActivateQuickFeet(user, target)`
7. `UpdateStatusIconFlags(target)` → sets bit 6 (`f_taunt`)

**Persistent icon:** SMA anim 7, palette 0 (fist icon)

**Application VFX:** None (no-op function)

**Source:** Ghidra decompilation of `TryInflictTauntStatus`.

### Enforcement (CanMonsterUseMove)

The taunt restriction is enforced in `CanMonsterUseMove`, which is the central move
validation function. It is NOT in `ExecuteMonsterAction` or `MonsterCannotAttack`.

```c
bool CanMonsterUseMove(entity *monster, move *move, bool extra_checks) {
    // ... (regular attack always returns true)
    // ... (disabled/sealed moves return false)
    if (extra_checks) {
        if (pp == 0) return false;
        if (cringe == 5 && IsAffectedByTaunt(move) == false) return false;
        if (cringe == 6) { /* encore enforcement */ }
        return true;
    }
    return true;
}
```

The `extra_checks` parameter gates the taunt/encore/PP checks. When called with
`extra_checks = true`, taunted monsters cannot use moves that fail `IsAffectedByTaunt`.

A near-identical duplicate exists at `FUN_02324be8`.

**Source:** Ghidra decompilation of `CanMonsterUseMove` and `FUN_02324be8`.

### IsAffectedByTaunt (waza_p.bin flag)

```c
bool IsAffectedByTaunt(move *move) {
    return *(bool *)(move_id * 0x1a + move_data_base + 0x14);
}
```

Reads a boolean at offset `0x14` within each move's `0x1A`-byte entry in `waza_p.bin`.

**Despite the name**, this returns `true` if the move **CAN** be used while taunted
(i.e. it is a damaging move). Returns `false` for status/non-damaging moves, which
are then blocked.

**Source:** Ghidra decompilation of `IsAffectedByTaunt`. Move data entry size is `0x1A` bytes.

### AI Consideration (AiConsiderMove)

The AI applies the same check at the top of `AiConsiderMove`:

```c
if (cringe == 5 && IsAffectedByTaunt(move) == false) {
    local_3c = 1;  // minimum weight, move not considered
}
```

The AI skips target evaluation entirely for moves blocked by taunt, assigning them
the minimum weight of 1. The `can_be_used` flag remains false.

**Source:** Ghidra decompilation of `AiConsiderMove`.

### Action Blocking

Taunt does **NOT** block actions via `MonsterCannotAttack`. The cringe values that
cause a hard action block are 1 (cringe), 3 (paused), and 7 (infatuated).
Taunted monsters (cringe = 5) can still act — they are only restricted in which
moves they can select.

---

## Encore: Enforcement (Same Code Path)

Encore (cringe_class_status = 6) is enforced in the same `CanMonsterUseMove` function,
immediately after the taunt check:

```c
if (cringe == 6) {
    if (move.id == 0x160) {
        if ((info[0x144] & 0x10) == 0) return false;
    } else {
        if ((move.flags & 0x10) == 0) return false;
    }
}
```

This restricts the monster to only using the encored move. The `0x10` flag on the move
struct likely indicates "this was the last move used" (`f_last_used`). Move ID `0x160`
appears to be a special case (possibly Struggle or the regular attack variant).

**Source:** Ghidra decompilation of `CanMonsterUseMove`.

---

## Cringe Class Status Transfer (FUN_02307198)

`FUN_02307198` transfers a cringe-class status from one monster to another. It:

1. Reads the source monster's `cringe_class_status` (offset 0xD0)
2. Attempts to inflict the same status on the target using `check_only = true`
3. If the check passes, copies both the status value and duration to the target
4. Clears the source's cringe status
5. Updates icons and speed stages for both monsters

The switch covers all cringe class values:

| Value | Status | Check Function |
|-------|--------|----------------|
| 1 | Cringe | `TryInflictCringeStatus` |
| 2 | Confused | `TryInflictConfusedStatus` |
| 3 | Paused | `TryInflictPausedStatus` |
| 4 | Cowering | `TryInflictCoweringStatus` |
| 5 | Taunted | `TryInflictTauntStatus` |
| 6 | Encore | `TryInflictEncoreStatus` |
| 7 | Infatuated | `TryInflictInfatuatedStatus` |
| 8 | (no handler) | — |

Likely used by a move or ability that transfers status conditions.

**Source:** Ghidra decompilation of `FUN_02307198`.

---

## ExecuteMonsterAction: Action ID Dispatch

`ExecuteMonsterAction` dispatches on `action.action_id` via a switch statement.
Some identified cases:

| Action ID | Handler | Purpose |
|-----------|---------|---------|
| 0x00 | (break → pass) | Nothing |
| 0x01 | (break → pass) | Pass turn |
| 0x02 | inline code | Walk |
| 0x08 | `FUN_022f4bf8` | **Item use (type A)** — calls `GetItemToUse` |
| 0x09 | `FUN_022f4350` | Unknown |
| 0x0A | `FUN_022f4dac` | **Item use (type B)** — calls `GetItemToUse` |
| 0x0B | (goto item throw) | Item throw |
| 0x0D, 0x0E | `UseSingleUseItemWrapper` | Single-use item |
| 0x10 | sets action to 1 | Force pass |
| 0x12 | `UseSingleUseItemWrapper` | Single-use item (alt) |
| 0x13 | `FUN_022f59c4` | Unknown (candidate for move execution) |
| 0x14 | `FUN_022f5f18` | Unknown (candidate for move execution) |
| 0x15 | `FUN_0231a8a0` | Unknown (candidate for move execution) |
| 0x17 | `FUN_022f6058` | Unknown (param 0x160) |
| 0x23 | `thunk_FUN_022f52bc` | Unknown |
| 0x24 | `UseThrowableItem` | Throw item |
| 0x25 | `TryTriggerTrap` | Trigger trap |
| 0x26 | stairs/floor transition | Use stairs |
| 0x27 | item throw path | Item throw (alt) |
| 0x31 | `FUN_0231a9f8` | Unknown |
| 0x32 | `FUN_022f6058` | Unknown (param 0x163) |
| 0x36 | `HandleHeldItemSwaps` | Swap held items |
| 0x3F | `TryNonLeaderItemPickUp` | Pick up item |
| 0x41 | item throw path | Item throw (with check) |
| 0x42 | `thunk_FUN_02343d30` | Unknown |

Cases 0x08 and 0x0A (`FUN_022f4bf8` / `FUN_022f4dac`) are confirmed as **item use handlers**,
not move execution. The move execution handler is likely one of cases 0x13, 0x14, or 0x15.

Note: `FUN_022f4dac` has incomplete decompilation in Ghidra — raw bytes after the `GetTile`
call at `022f4e84` indicate unanalyzed code. Re-disassembly starting at `022f4e90` is recommended.

**Source:** Ghidra decompilation of `ExecuteMonsterAction`.

---

## Functions Referenced

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `BoostOffensiveStat` | — | Stat stage increase (offensive) |
| `BoostDefensiveStat` | — | Stat stage increase (defensive) |
| `ApplyDefensiveStatMultiplier` | — | Stat multiplier change (defensive) |
| `PlayOffensiveStatUpEffect` | — | VFX dispatch for offensive stat up |
| `PlayDefensiveStatUpEffect` | — | VFX dispatch for defensive stat up |
| `PlayDefensiveStatMultiplierDownEffect` | — | VFX dispatch for defensive multiplier down |
| `PlayDefensiveStatMultiplierUpEffect` | — | VFX dispatch for defensive multiplier up |
| `TryInflictTauntStatus` | — | Taunt infliction with immunity checks |
| `IsAffectedByTaunt` | — | Reads waza_p.bin offset 0x14 per move |
| `CanMonsterUseMove` | — | Central move validation (taunt + encore enforcement) |
| `FUN_02324be8` | `0x02324be8` | Near-duplicate of `CanMonsterUseMove` |
| `AiConsiderMove` | — | AI move evaluation (includes taunt check) |
| `FUN_02307198` | `0x02307198` | Cringe-class status transfer between monsters |
| `TryInflictSetDamageStatus` | — | Set Damage infliction (Doom Desire / Future Sight) |
| `TryInflictFocusEnergyStatus` | — | Focus Energy infliction |
| `TryInflictStockpileStatus` | — | Stockpile level increment (caps at 3) |
| `TryInflictIngrainStatus` | — | Ingrain infliction (frees wrapped monsters) |
| `TryInflictLeechSeedStatus` | — | Leech Seed infliction (Grass immune, stores source) |
| `TryInflictHealBlockStatus` | — | Heal Block infliction |
| `TryInflictInfatuatedStatus` | — | Infatuation infliction (Oblivious blocks) |
| `TryInflictAquaRingStatus` | — | Aqua Ring infliction |
| `FUN_022f4bf8` | `0x022f4bf8` | Item use handler (action 0x08), NOT move execution |
| `FUN_022f4dac` | `0x022f4dac` | Item use handler (action 0x0A), NOT move execution |
| `ExecuteMonsterAction` | — | Top-level action dispatcher |
| `UpdateStatusIconFlags` | `0x022e3a58` | Recalculates status icon bitfield |

---

## Breadcrumbs

- **Move execution handler** — likely `FUN_022f59c4` (action 0x13), `FUN_022f5f18` (0x14),
  or `FUN_0231a8a0` (0x15). These are the unidentified action handlers that are candidates
  for the actual "use a move" dispatch.
- **waza_p.bin move data** — each entry is `0x1A` bytes. Offset `0x14` is the taunt flag
  (true = can be used while taunted). Other offsets contain move category, type, power, etc.
- **`FUN_022f4dac` incomplete decompilation** — Ghidra fails to decompile past `022f4e84`.
  Raw ARM instructions from `022f4e90` onward need manual re-disassembly. This is the
  item use handler for action 0x0A, continuing past the initial item validation checks.
- **Encore move ID `0x160`** — special-cased in `CanMonsterUseMove`. Needs identification
  (possibly Struggle or a regular attack variant).
- **`FUN_02307198` caller** — unknown. Likely a move effect or ability that transfers
  cringe-class statuses between monsters. XREF analysis needed.
