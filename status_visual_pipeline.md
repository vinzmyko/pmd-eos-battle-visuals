# Status Visual Pipeline (Dungeon Mode)

## Summary

This document covers the complete visual pipeline for status effects in PMD:EOS dungeon mode.
It describes what visual events occur when a status is applied, persists, ticks damage, and expires.
For the SMA icon format details and the full status→icon bit→SMA animation mapping, see the
companion document `status_icon_system.md`.

---

## Visual Pipeline Overview

Each status can trigger up to five visual stages:

```
1. APPLICATION   → One-shot effect.bin animation + sound + log message + icon appears
1b. STAT CHANGE  → One-shot effect.bin animation for stat stage/multiplier changes
1c. ACTION BLOCK → MonsterCannotAttack prevents action (no RNG, no VFX)
2. PERSISTENCE   → Overhead SMA icon cycles while status is active
3. PER-TICK      → Damage number + hurt animation + log message (each countdown cycle)
4. EXPIRY/CURE   → Log message + icon removed + recovery animation
```

Not all statuses use all stages. Short-duration statuses (cringe, paralysis) often skip
the persistent icon. Passive statuses (reflect, protect) skip per-tick damage.

---

## Stage 1: Application VFX

When a status is first inflicted, a one-shot visual effect plays on the pokemon.
These use `PlayEffectAnimationEntity` with an effect.bin animation index.

**Source:** Ghidra decompilation of each `TryInflict*` and `Play*Effect` function in overlay 29,
address range `022e4068`–`022e53f0`.

### Effect Animation VFX (effect.bin)

| Status | Function | Effect ID (hex) | Effect ID (dec) | Source Address |
|--------|----------|-----------------|-----------------|---------------|
| Burn | `FUN_022e4294` | `0x4C` | 76 | `022e42a0` |
| Paralysis | `PlayParalysisEffect` | `0x1A7` | 423 | `DAT_022e428c` |
| Petrified | `FUN_022e41f0` | `0x1A7` | 423 | `DAT_022e423c` |
| Poisoned | `FUN_022e4388` | `0x13A` | 314 | `DAT_022e43d4` |
| Badly Poisoned | `FUN_022e43d8` | `0x13A` | 314 | `DAT_022e4424` |
| Frozen | `FUN_022e4c4c` | `0x15` | 21 | `022e4c4c` (hardcoded, palette 3) |
| Flash Fire | `FUN_022e4338` | `0x1A9` | 425 | `DAT_022e4384` |
| Blinker | `FUN_022e486c` | `0x41` | 65 | `022e486c` (hardcoded) |
| Sure Shot | `FUN_022e456c` | `0x05` | 5 | `022e4578` (hardcoded) |
| Cringe/Flinch | `PlayExclamationPointEffect` | `0x143` | 323 | `DAT_022e3e70` / `DAT_022e3f1c` / `DAT_022e53e8` (3 identical copies) |
| HP Recovery | `FUN_022e4430` | `0x171` | 369 | `DAT_022e447c` |
| HP Restore | `FUN_022e4480` | `0x07` | 7 | `022e448c` (hardcoded) |
| Speed Up | `PlaySpeedUpEffect` | `0x18B` | 395 | `DAT_022e4518` |
| Speed Down | `PlaySpeedDownEffect` | `0x18A` | 394 | `DAT_022e4568` |
| Yawning tick → Sleep | `FUN_022e53f0` | `0x19` | 25 | `022e53f0` (hardcoded) |
| Sleep (infliction) | `FUN_022e3e74` | `0x25` | 37 | `022e3e74` (+ sound via `DAT_022e3ecc`) |
| Freeze (thaw) | `FUN_022e6798` | `0x18` | 24 | `022e6798` (only for case 1: frozen) |
| Damage (attacker side) | `FUN_022e45d0` part 1 | `0x2F` | 47 | `022e45dc` (hardcoded) |
| Damage (defender side) | `FUN_022e45d0` part 2 | `0x30` | 48 | `022e4618` (hardcoded) |

**Source:** Ghidra disassembly of each function listed. Effect IDs read from DAT_ constants
or hardcoded immediates in the ARM instructions.

### Heal Effect Dispatch (TryIncreaseHp / TryRestoreHp)

The two heal effects are dispatched through different code paths:

**Effect `0x07` (7) — HP Restore** is played by `TryRestoreHp` (`022e4480`).
This is the standard healing path used by the majority of heal sources.

Callers of `TryRestoreHp`: `TryIncreaseHp` (fallback when no max HP boost applies),
`DoMoveTag0x1A7` (move tag 0x1A7 — ends negative status + heals + restores PP).

**Effect `0x171` (369) — HP Recovery** is played by `TryIncreaseHp` directly,
but **only** when max HP is being boosted (the branch where `max_hp_boost > 0` and
either `hp == max_hp` or `hp_restoration == 0`). When `TryIncreaseHp` performs a
normal heal without max HP boost, it delegates to `TryRestoreHp` which plays effect 7.

Callers of `TryIncreaseHp`: `EndSleepClassStatus`, `ApplyDamage`, `FUN_0230fc24`
(per-turn tick handler), `ApplyAquaRingHealing`, `ApplyItemEffect`,
`DoMoveMorningSun`, `DoMoveDamageDrain`, `DoMoveSynthesis`, `DoMoveRecoverHp`,
`DoMoveAbsorb`, `DoMoveRecoverHpTeam`, `DoMoveMoonlight`, `DoMoveSwallow`,
`DoMovePresent`, `DoMoveDreamEater`, `DoMoveHealingWish`, `DoMoveHealOrder`,
`DoMoveRoost`, `DoMoveLunarDance`, `FUN_022fabc`, `FUN_02310c18`.

**Summary:** Effect 7 is the general-purpose heal VFX. Effect 369 is the max HP boost VFX.

**Source:** Ghidra XREF analysis of `FUN_022e4430` and `FUN_022e4480`.
`TryIncreaseHp` at `022e4d44`. `TryRestoreHp` at `022e4f98`.

### Sound-Only Effects (no visual animation)

| Status | Function | Sound Effect ID | Source Address |
|--------|----------|----------------|---------------|
| Confused | `FUN_022e41cc` | `0x310` (784) | `022e41d0` |
| Whiffer | `FUN_022e45b8` | `0x227` (551) | `DAT_022e45c8` |

These call `PlaySeByIdIfShouldDisplayEntity` instead of `PlayEffectAnimationEntity`.

**Source:** Ghidra disassembly at `022e41cc` and `022e45b8`.

### Animation Change (no effect.bin animation)

| Status | Function | Monster Anim ID | Anim Group | Source |
|--------|----------|-----------------|------------|--------|
| Cowering | `FUN_022e41dc` | 10 | 8 | `022e41e0`–`022e41e4` |

This calls `ChangeMonsterAnimation` to change the pokemon's sprite animation, not
`PlayEffectAnimationEntity`.

**Source:** Ghidra disassembly at `022e41dc`.

### No-Ops (no application VFX at all)

These functions are `bx lr` (immediate return) — the status applies silently with only
a log message and icon update.

| Status | Function | Source Address |
|--------|----------|---------------|
| Reflect | `FUN_022e4068` | `022e4068` |
| Protect | `FUN_022e40b8` | `022e40b8` |
| Mirror Coat | `FUN_022e40bc` | `022e40bc` |
| Encore | `FUN_022e4718` | `022e4718` |
| Yawning | `FUN_022e53ec` | `022e53ec` |
| Wrapped | `FUN_022e4290` | `022e4290` |
| Shadow Hold | `FUN_022e42e0` | `022e42e0` |
| Constriction | `FUN_022e42e4` | `022e42e4` |
| Paused | `FUN_022e4428` | `022e4428` |
| Taunted | `FUN_022e442c` | `022e442c` |
| Destiny Bond | `FUN_022e45cc` | `022e45cc` |
| Reveal Items | `FUN_022e465c` | `022e465c` |
| Reveal Stairs | `FUN_022e4660` | `022e4660` |
| Reveal Enemies | `FUN_022e4664` | `022e4664` |
| Leech Seed | `FUN_022e4668` | `022e4668` |
| Set Damage | `FUN_022e466c` | `022e466c` |

**Source:** Ghidra listing confirms `bx lr` at each address.

### Effect Reuse

Several statuses share the same effect.bin animation:

| Effect ID | Used By |
|-----------|---------|
| `0x1A7` (423) | Paralysis, Petrified |
| `0x13A` (314) | Poisoned, Badly Poisoned |
| `0x143` (323) | Cringe/Flinch (3 identical function copies at different addresses) |
| `0x1A9` (425) | Flash Fire, Offensive Stat Up (physical) |

---

## Stage 1b: Stat Change VFX

Stat changes play effect.bin animations that differ by stat category (physical vs special)
but NOT by whether it's a stage change or multiplier change — both use the same effect.

**Source:** Ghidra decompilation of `Play*Stat*Effect` functions. DAT_ values confirmed
at addresses listed below.

### Stat Stage & Multiplier Effect Table

| Effect | Physical (stat_index=0) | Special (stat_index=1) |
|--------|------------------------|------------------------|
| Offensive Up | `0x1A9` (425) | `0x192` (402) |
| Offensive Down | `0x194` (404) | `0x193` (403) |
| Defensive Up | `0x18E` (398) | `0x190` (400) |
| Defensive Down | `0x18F` (399) | `0x191` (401) |
| Hit Chance Up (accuracy/evasion) | `0x18C` (396) | `0x0D` (13) |
| Hit Chance Down (accuracy/evasion) | `0x18D` (397) | `0x0E` (14) |

**Source addresses for DAT_ constants:**

| Constant | Value | Address |
|----------|-------|---------|
| `DAT_022e4f14` | `0x1A9` | Offensive Stat Up, physical |
| `DAT_022e4dc8` | `0x193` | Offensive Stat Down, special |
| `DAT_022e4e6c` | `0x18F` | Defensive Stat Down, physical |
| `DAT_022e4e70` | `0x191` | Defensive Stat Down, special |
| `DAT_022e5060` | `0x1A9` | Offensive Multiplier Up, physical (same as stage) |
| `DAT_022e5108` | `0x193` | Offensive Multiplier Down, special (same as stage) |
| `DAT_022e5250` | `0x18F` | Defensive Multiplier Down, physical (same as stage) |
| `DAT_022e5254` | `0x191` | Defensive Multiplier Down, special (same as stage) |
| `DAT_022e5398` | `0x18D` | Hit Chance Down, accuracy |

The game uses **10 unique effect.bin animations** for all stat changes. Up/down pairs are
adjacent IDs, and physical/special get different visuals. The stat stage and multiplier
variants are visually identical.

---
## Stage 1c: Action Blocking (MonsterCannotAttack)

Some statuses completely prevent the monster from acting on its turn. This is checked by
`MonsterCannotAttack` at the top of `ExecuteMonsterAction`, before any action processing.
There is **no RNG roll** — if the status is active, the turn is unconditionally skipped.

**Source:** Ghidra decompilation of `MonsterCannotAttack`.

### Statuses That Block Action

| Field | Value(s) | Status | Notes |
|-------|----------|--------|-------|
| 0xBD (sleep class) | != 0, 2, 4 | Sleep (1), Nightmare (3), Napping (5) | Skipped if `skip_sleep` param is true |
| 0xC4 (freeze class) | 1 | Frozen | — |
| 0xC4 (freeze class) | 3 | Shadow Hold | — |
| 0xC4 (freeze class) | 4 | Wrap | — |
| 0xC4 (freeze class) | 6 | Petrified | — |
| 0xD0 (cringe class) | 1 | Cringe | — |
| 0xD0 (cringe class) | 3 | Paused | — |
| 0xD0 (cringe class) | 7 | Infatuated | — |
| 0xBF (burn class) | 4 | **Paralysis** | Hard block, no RNG, no damage |

### Statuses That Do NOT Block Action

| Field | Value(s) | Status |
|-------|----------|--------|
| 0xBF | 1 | Burn |
| 0xBF | 2 | Poison |
| 0xBF | 3 | Badly Poisoned |
| 0xBD | 2 | Sleepless |
| 0xBD | 4 | Yawning |

### Fallthrough: ShouldMonsterRunAway

If none of the above statuses block, `MonsterCannotAttack` calls `ShouldMonsterRunAway`.
This is NOT a status check — it returns true if:

- `info + 0x104` is set AND `0x105` counter is nonzero (temporary flee state)
- **Run Away** ability + HP below half max (wild enemies only, `info + 7 == 0`)
- **"Get Away From Here"** tactic is set
- **"Avoid Trouble"** tactic + HP at or below half max

### Paralysis: Complete Mechanic

PMD paralysis is a **double penalty** with no randomness:

1. **Speed reduction** — one stage down at infliction (fewer fractional turns per round)
2. **Hard action block** — `MonsterCannotAttack` returns true every turn while active
3. **Duration counter** — `TickStatusAndHealthRegen` decrements `0xC0` each fractional turn; at 0, `EndBurnClassStatus` cures it

No per-turn damage. No per-turn VFX. The monster sits idle on every turn it receives.

### Turn Execution Flow
```
RunFractionalTurn
  └─► per monster:
        TickStatusAndHealthRegen(entity)    ← decrements ALL status duration counters
        RunMonsterAi(entity)                ← decides action
        ExecuteMonsterAction(entity)
          ├─► MonsterCannotAttack(entity)   ← if true, turn skipped
          └─► action switch (walk, use move, use item, etc.)
              └─► FUN_0230fc24(entity)      ← per-turn damage ticks (burn, poison, etc.)
```

**Source:** `RunFractionalTurn` decompilation. `TickStatusAndHealthRegen` runs BEFORE
`ExecuteMonsterAction`, so duration counters tick even on blocked turns.

### Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `MonsterCannotAttack` | — | Checks if status prevents action (no RNG) |
| `ShouldMonsterRunAway` | — | Tactic/ability flee check (fallthrough) |
| `ExecuteMonsterAction` | — | Main action dispatcher, calls MonsterCannotAttack |
| `RunFractionalTurn` | — | Per-fractional-turn orchestrator |
| `TickStatusAndHealthRegen` | — | Decrements all status duration counters |

---

## Stage 2: Persistent Overhead Icons

While a status is active, an icon from `manpu_su.sma` renders above the monster's sprite.
If multiple status icons are active, they cycle through one at a time.

See `status_icon_system.md` for the complete mapping of status → icon bit → SMA animation
index + palette.

### Icon Cycling Timing

The renderer `FUN_022dc820` (Ghidra: `022dc820`–`022dd088`) manages icon cycling.

Each icon displays for **60 frames** (`0x3C`) before advancing to the next active icon.
At the NDS's ~60fps, this is approximately **1 second per icon**.

When the countdown reaches 0, the renderer scans forward through the `status_icon_flags`
bitfield to find the next set bit, wraps around if needed, and resets the countdown to 60.

The renderer is called twice per frame:
- `param_2 = 0` → cycling icons (scans all 32 bits of offset 0x218)
- `param_2 = 1` → persistent icons (only checks offset 0x21C, e.g. freeze)

Persistent icons (like the freeze ice block) are always displayed and do not participate
in the cycling rotation.

**Source:** `FUN_022dc820` decompilation. Timer value `0x3C` at Ghidra address
around `022dc970`. Decrement at `022dc840` region.

---

## Stage 3: Per-Tick Damage

The per-turn status tick handler is `FUN_0230fc24` (Ghidra: `0230fc24`).
It runs once per turn for each monster and processes all tick-based statuses.

### Tick Pipeline

Every tick follows the same pattern:

```
decrement burn_damage_countdown
  → if countdown reaches 0:
    → reset countdown to default value
    → DisplayActions(NULL)  // wait for pending animations
    → TryEndPetrifiedOrSleepStatus()
    → ApplyDamageAndEffectsWrapper(monster, damage, DAMAGE_MESSAGE_xxx, source)
      → ApplyDamage()
        → DisplayAnimatedNumbers(-damage, defender)  // damage number above sprite
        → ChangeMonsterAnimation(defender, 6, direction)  // hurt animation
        → FUN_022e5478(defender, damage_data)  // hit-stagger visual
        → FUN_022ea370(10, 0x18, ...)  // display pause
```

**No additional effect.bin animations play during ticks** (with one exception: constriction).

**Source:** Ghidra decompilation of `FUN_0230fc24` at `0230fc24`.

### Tick Status Details

| Status | Monster Field | Countdown Field | Countdown Reset | Damage | Damage Message | Damage Source | Notes |
|--------|-------------|-----------------|-----------------|--------|---------------|---------------|-------|
| Burn | 0xBF = 1 | 0xC1 | `DAT_02310aac` | `DAT_02310ab0` (fixed) | `DAMAGE_MESSAGE_BURN` (1) | `0x247` | Two-counter system (see below) |
| Poison | 0xBF = 2 | 0xC1 | `DAT_02310ac0` | `DAT_02310ac4` (fixed) | `DAMAGE_MESSAGE_POISON` (3) | `0x249` | Poison Heal ability reverses to HP recovery |
| Badly Poisoned | 0xBF = 3 | 0xC1 | `DAT_02310ac8` | table at `DAT_02310acc` | `DAMAGE_MESSAGE_POISON` (3) | `0x249` | Escalating damage indexed by tick count (0xC2, caps at 29). Poison Heal reverses |
| Constriction | 0xC4 = 7 | 0xCD | `DAT_02310ad0` | `DAT_02310ad4` (fixed) | `DAMAGE_MESSAGE_CONSTRICTION` (2) | `0x248` | **EXCEPTION:** plays `PlayEffectAnimationEntityStandard` with anim from offset 0xC8 BEFORE damage |
| Wrap | 0xC4 = 4 | 0xCD | `DAT_02310ad8` | `DAT_02310adc` (fixed) | `DAMAGE_MESSAGE_WRAP` (5) | `DAT_02310ae0` | — |
| Wrapped/Ingrain | 0xC4 = 5 | 0xCD | `DAT_02310ae4` | `DAT_02310ae8` | — | — | **Heals HP** instead of dealing damage |
| Curse | 0xD8 = 1 | 0xDC | `DAT_02310aec` | max_hp / 4 (min 1) | `DAMAGE_MESSAGE_CURSE` (7) | `0x24b` | Damage calculated as `(max_hp_stat + max_hp_boost) / 4` |
| Leech Seed | 0xE0 = 1 | 0xEA | `DAT_02310af0` | `DAT_02310af4` (fixed) | `DAMAGE_MESSAGE_LEECH_SEED` (9) | `0x24c` | Drains HP to source pokemon. Liquid Ooze reverses (deals `DAMAGE_MESSAGE_SLUDGE` to drainer) |
| Perish Song | 0x106 | counter | — | 999 (instant KO) | `DAMAGE_MESSAGE_PERISH_SONG` (11) | `0x24d` | If Protect (0xD5 = 7) is active, just logs a message instead |

**Source:** All offsets and DAT_ references from Ghidra decompilation of `FUN_0230fc24`.

### Freeze Lifecycle (0xC4 = 1)

**Infliction (`TryInflictFrozenStatus`):**
1. Immunity checks: Safeguard, `IsProtectedFromNegativeStatus`, exclusive item `EXCLUSIVE_EFF_NO_FREEZE`, Magma Armor ability, Ice type, lava terrain
2. If target was wrapping (0xC4 = 3 or 4), `FreeOtherWrappedMonsters` releases wrapped monsters
3. `FUN_022e4c4c` → `PlayEffectAnimationEntity(target, 0x15, palette=3)` — one-shot ice flash
4. `info + 0xC4 = 1`, `info + 0xCC = CalcStatusDuration() + 1`, `info + 0xCD = 0`
5. Log message, `UpdateStatusIconFlags` → SMA ice overlay (anim 4) starts rendering persistently
6. `TryActivateQuickFeet`, `ChangeShayminForme` (Shaymin reverts to Land forme)

**While frozen:**
- `MonsterCannotAttack` returns true → turn skipped every turn (hard block)
- No per-turn damage, no per-turn VFX, no entry in `FUN_0230fc24`
- No `ChangeMonsterAnimation` call at infliction — renderer likely forces pose via status field check each frame (see Animation Mystery below)
- SMA ice overlay at Centre attachment point is the only ongoing visual

**Thaw (`EndFrozenClassStatus`):**

| Case | Status | Visual | Releases Wrapped? |
|------|--------|--------|-------------------|
| 1 | Frozen | Log + `FUN_022e6798` → effect 0x18 (24) thaw effect | No |
| 2 | (unused?) | Log only | No |
| 3 | Shadow Hold | Log | Yes |
| 4 | Wrap | Log | Yes |
| 5 | Ingrain | Log only | No |
| 6 | Petrified | Log only | No |
| 7 | Constriction | Log only | No |

All cases clear `0xC4 = 0` and call `UpdateStatusIconFlags`. Only case 1 (frozen) plays a thaw VFX. `FUN_02304a48` (reset to idle) is NOT called at thaw.

**Source:** Ghidra decompilation of `TryInflictFrozenStatus`, `EndFrozenClassStatus`, `FUN_022e4c4c`, `FUN_022e6798`.

### Sleep Lifecycle (0xBD = 1)

**Infliction (`TryInflictSleepStatus`):**
1. Immunity check via `IsProtectedFromSleepClassStatus`
2. If already Sleepless (0xBD = 2) or Napping (5), log failure and return
3. `FUN_022e3e74` → `PlayEffectAnimationEntity(target, 0x25)` — effect 37 (sleep dust/sparkles) + sound via `DAT_022e3ecc`
4. `InflictSleepStatusSingle(target, turns)` — sets `0xBD = 1`, `0xBE = turns`
5. `FUN_02304a48(target, 8)` — resets animation to idle (NOT sleep group 5)
6. Log message, `TryActivateQuickFeet`

No `ChangeMonsterAnimation(target, 5, ...)` call anywhere in the chain — renderer likely forces sleep pose via status field check each frame (see Animation Mystery below).

**While sleeping:**
- `MonsterCannotAttack` returns true → turn skipped (hard block)
- SMA icon: Z's overhead (anim 16, palette 4)
- No per-turn damage for regular sleep (0xBD = 1)
- No per-turn VFX

**Wake-up (`EndSleepClassStatus`):**

| Case | Status | Visual | Damage/Heal |
|------|--------|--------|-------------|
| 1 | Sleep | Log + `FUN_02304a48(target, 8)` (idle reset) | None |
| 2 | Sleepless | Log only | None |
| 3 | Nightmare | Log + `FUN_02304a48(target, 8)` | Damage via `ApplyDamageAndEffectsWrapper(DAMAGE_MESSAGE_NIGHTMARE)` **at wake-up only** |
| 4 | Yawning | Transitions to real sleep via `TryInflictSleepStatus` (fresh duration) | None |
| 5 | Napping | Log + `FUN_02304a48(target, 8)` + `TryIncreaseHp` + `EndNegativeStatusCondition` | Heals HP + cures negative statuses **at wake-up only** |

All cases (except yawning transition) clear `0xBD = 0` and call `UpdateStatusIconFlags`.

**Key findings:**
- Nightmare damage is dealt **at wake-up, not per-turn**. There is no nightmare entry in `FUN_0230fc24`.
- Napping heals HP and cures negatives **at wake-up, not per-turn**. Same — no napping entry in `FUN_0230fc24`.
- Yawning (case 4) with `in_r2 != 0` (the path from `TickStatusAndHealthRegen` counter expiry) recursively calls `TryInflictSleepStatus` with a freshly calculated duration.

**Source:** Ghidra decompilation of `TryInflictSleepStatus`, `EndSleepClassStatus`, `FUN_022e3e74`.

### Animation Mystery: Freeze and Sleep

Neither `TryInflictFrozenStatus` nor `TryInflictSleepStatus` call `ChangeMonsterAnimation` to switch the sprite to an appropriate pose. Both set their status field and play a one-shot effect, but leave the sprite animation unchanged.

This strongly suggests the **per-frame rendering path** (`FUN_02303f18` or the animation update system) checks status fields each frame and forces the correct animation group:
- `0xBD` active (sleep/nightmare/napping) → sleep animation (group 5)
- `0xC4 == 1` (frozen) → idle or frozen pose

**Status: Needs ROM confirmation.** Verify by testing freeze and sleep in-game:
- Does the sprite visibly change pose?
- Is the frozen sprite static or still animating (idle loop)?
- Is the ice overlay opaque enough to cover the sprite?

**Breadcrumb:** `FUN_02303f18` — per-frame entity update function documented in `positioning_system.md`. Likely contains the status → animation group override logic.

### Burn Class: Two-Counter System

Burn, poison, and badly poisoned have **two independent counters**:

| Counter | Offset | Decremented By | Purpose |
|---------|--------|----------------|---------|
| **Duration** | 0xC0 | `TickStatusAndHealthRegen` (every fractional turn) | When 0 → `EndBurnClassStatus` → status cured |
| **Damage tick** | 0xC1 | `FUN_0230fc24` (per-turn status tick) | When 0 → reset from DAT + deal damage |

Duration counter (`0xC0`) controls how long the status lasts. Damage tick counter (`0xC1`)
controls how frequently damage is dealt. These tick independently — the damage tick resets
each time it fires, while the duration counter counts down to expiry.

Paralysis (`0xBF = 4`) only uses the duration counter (`0xC0`). It has no damage tick —
no entry in `FUN_0230fc24` deals damage for paralysis. Action is blocked by
`MonsterCannotAttack` instead.

**Source:** `TickStatusAndHealthRegen` for duration, `FUN_0230fc24` for damage ticks.

### Other Per-Turn Effects (same function)

| Effect | Trigger | Damage Message | Notes |
|--------|---------|---------------|-------|
| Hunger | belly reaches 0 | `DAMAGE_MESSAGE_HUNGER` (14) | 1 damage per turn |
| Hail weather | not Ice type / Snow Cloak / Ice Body | `DAMAGE_MESSAGE_BAD_WEATHER` (18) | |
| Sandstorm weather | not Ground / Rock / Steel / Sand Veil | `DAMAGE_MESSAGE_BAD_WEATHER` (18) | |
| Sunny + Solar Power | has Solar Power ability | `DAMAGE_MESSAGE_SOLAR_POWER` (25) | |
| Sunny + Dry Skin | has Dry Skin ability | `DAMAGE_MESSAGE_DRY_SKIN` (26) | |
| Yawning | 0xBD = 4 | — | Calls `FUN_022e53f0` (effect `0x19`), transitions to sleep |
| Bide | 0xD2 = 1 countdown expires | — | Executes stored move via `FUN_02322374` |

**Source:** `FUN_0230fc24` decompilation.

---

## Stage 4: Expiry / Cure

When a status expires or is cured, the typical sequence is:

1. Log message (varies by status, often from a `switch` on the status class enum value)
2. Status field set to 0 (e.g. `*(offset + 0xBF) = 0`)
3. `UpdateStatusIconFlags(target)` — recalculates icon bitfield, removes the icon
4. Recovery animation: `FUN_02304a48(target, 8)` — monster animation ID 8 (wake-up/recovery)

The `End*ClassStatus` functions (e.g. `EndBurnClassStatus`, `EndSleepClassStatus`,
`EndFrozenStatus`, `EndLeechSeedClassStatus`) all follow this pattern.

**Source:** Ghidra decompilation of `EndBurnClassStatus`, `EndSleepClassStatus`,
`EndFrozenStatus`, `EndLeechSeedClassStatus`.

---

## Damage Display System

### DisplayAnimatedNumbers

`DisplayAnimatedNumbers` (Ghidra: `022ea720`) renders floating numbers above a monster.

```c
void DisplayAnimatedNumbers(int amount, entity *entity, bool display_sign, number_color color);
```

- `amount`: Positive for healing, negative for damage. `9999` displays "MISS" text.
- `display_sign`: Whether to show +/- prefix.
- `number_color`: Explicit color, or `NUMBER_COLOR_AUTO` (-1) for automatic.

**Source:** Ghidra decompilation of `DisplayAnimatedNumbers`.

### Auto Color Logic

When `NUMBER_COLOR_AUTO` is passed, the color threshold is `DAT_022ea808 = 0xFFFFFC19` = **-999**:

| Condition | Color ID | Color Name | Meaning |
|-----------|----------|------------|---------|
| `amount >= 0` | 10 | Green | HP recovery / healing |
| `-999 <= amount < 0` | 3 | Yellow | Normal damage |
| `amount < -999` | 6 | Red | Extreme damage (edge case, max HP is 999) |

**Source:** `DAT_022ea808` at Ghidra address `022ea808`, value `0xFFFFFC19`.
Color logic in `DisplayAnimatedNumbers` at `022ea7c8`–`022ea7fc`.

### ApplyDamage Hurt Animation Sequence

`ApplyDamage` (Ghidra: `02309380`) handles the visual sequence when damage is dealt:

1. `DisplayAnimatedNumbers(-damage, defender, true, NUMBER_COLOR_AUTO)` — floating damage number
2. `GetDirectionTowardsPosition` → monster turns to face attacker
3. `ChangeMonsterAnimation(defender, 0x06, direction)` — plays the **hurt animation** (WAN anim ID 6)
4. `FUN_022e5478(defender, damage_data)` — hit-stagger/knockback visual
5. `FUN_022ea370(10, 0x18, ...)` — frame pause for visual readability
6. `UpdateStatusIconFlags(defender)` — update icons (e.g. low HP icon may appear)
7. If HP reaches 0: `defender->damage_visual_effect = DAMAGE_VISUAL_FLICKERING` (sprite flickers before faint)

**Source:** Ghidra decompilation of `ApplyDamage` at `02309380`.

### Key Monster WAN Animation IDs

These are sprite animation indices from the pokemon's WAN file, NOT effect.bin indices:

| Anim ID | Purpose | Where Used | Source |
|---------|---------|------------|--------|
| 6 | Hurt / flinch | `ApplyDamage`: `ChangeMonsterAnimation(defender, 6, ...)` | `02309680` region |
| 8 | Wake up / status clear | `FUN_02304a48(target, 8)` after sleep ends, after surviving damage | various `End*Status` functions |
| 10 | Cowering | `FUN_022e41dc`: `ChangeMonsterAnimation(entity, 10, 8)` | `022e41e0` |
| 11 | Revival | `ApplyDamage` revival path: `FUN_02304830(defender, 0xB)` | `02309c7c` region |

**Source:** Ghidra disassembly of `ApplyDamage` and `FUN_022e41dc`.

---

## Statuses With No Persistent Icon

These statuses have no overhead SMA icon (their lookup table entry is all zeros).
Visual feedback relies on other cues:

| Status | Application VFX | Ongoing Visual Cue |
|--------|----------------|-------------------|
| Paralysis | effect `0x1A7` + speed down effect `0x18A` | Monster moves slower (speed stage -1) AND cannot act at all (hard block via `MonsterCannotAttack`, no RNG) |
| Cringe/Flinch | effect `0x143` (exclamation) | Short duration — expires before next turn |
| Infatuated | — (no application VFX found) | No visual cue |
| Paused | — (no-op) | No visual cue |
| Yawning | — (no-op) | No visual cue (transitions to sleep with effect `0x19`) |
| Shadow Hold | — (no-op) | Monster cannot move |
| Wrap/Wrapped | — (no-op) | Monster cannot move |
| Constriction | — (no-op at application) | Per-tick plays stored constriction animation |
| Petrified | effect `0x1A7` | No ongoing cue |
| Ingrain | — | Per-tick heals HP |
| All Leech Seed class | — (no-op) | Per-tick damage/drain |
| All Long Toss class | — | Item throw behavior change |
| All Invisible class | — | Sprite transparency / not rendered |
| Dropeye (Blinker idx 4) | — | No visual cue |

**Source:** Lookup tables from `status_icon_system.md` (all-zero entries) cross-referenced
with application VFX functions above.

---

## Complete Per-Status Visual Reference

For quick reference, here is every status that has ANY visual component, with all stages:

| Status | Apply Effect | Apply Sound | Persistent Icon | Tick Damage | Expire Anim |
|--------|-------------|-------------|-----------------|-------------|-------------|
| Sleep | effect `0x25` (37) + sound | — | SMA 16:pal4 (Z's) | — (action blocked) | WAN anim 8 (idle reset) |
| Sleepless | — | — | SMA 1:pal0 (tiny eye) | — | — |
| Nightmare | effect `0x25` (37) + sound | — | SMA 16:pal4 (Z's) | damage at wake-up only (DAMAGE_MESSAGE_NIGHTMARE) | WAN anim 8 (idle reset) |
| Napping | effect `0x25` (37) + sound | — | SMA 16:pal4 (Z's) | heals HP + cures negatives at wake-up only | WAN anim 8 (idle reset) |
| Burn | effect `0x4C` | — | SMA 2:pal0 (flame) | `DAMAGE_MESSAGE_BURN` | — |
| Poisoned | effect `0x13A` | — | SMA 3:pal11 (white skull) | `DAMAGE_MESSAGE_POISON` | — |
| Badly Poisoned | effect `0x13A` | — | SMA 3:pal7 (purple skull) | `DAMAGE_MESSAGE_POISON` (escalating) | — |
| Paralysis | effect `0x1A7` + speed down `0x18A` | — | **none** | — (hard action block, no damage) | — |
| Frozen | effect `0x15` (palette 3) | — | SMA 4:pal0 (persistent ice block at Centre) | — (action blocked) | effect `0x18` (24) thaw effect |
| Cringe | effect `0x143` | — | **none** | — | — |
| Confused | — | SE `0x310` | SMA 5:pal0 (birds) | — | — |
| Cowering | WAN anim 10 | — | SMA 6:pal0 (swirls) | — | — |
| Taunted | — | — | SMA 7:pal0 (fist) | — | — |
| Encore | — | — | SMA 8:pal0 (vertical exclamation) | — | — |
| Reflect | — | — | SMA 9:pal0 (shield) | — | — |
| Safeguard | — | — | SMA 9:pal4 (shield) | — | — |
| Light Screen | — | — | SMA 9:pal3 (shield) | — | — |
| Protect | — | — | SMA 9:pal10 (shield) | — | — |
| Endure | — | — | SMA 9:pal5 (shield) | — | — |
| Curse | — | — | SMA 3:pal6 (skull) | `DAMAGE_MESSAGE_CURSE` (max_hp/4) | — |
| Embargo/Snatch/Gastro Acid | — | — | SMA 8:pal3 (vertical shape) | — | — |
| Heal Block | — | — | SMA 15:pal3 (cross/X) | — | — |
| Sure Shot | effect `0x05` | — | SMA 11:pal0 (sword) | — | — |
| Whiffer | — | SE `0x227` | SMA 6:pal10 (swirls) | — | — |
| Focus Energy | — | — | SMA 11:pal4 (sword) | — | — |
| Blinker | effect `0x41` | — | SMA 12:pal0 (tiny blinking 8×8) | — | — |
| Cross-Eyed | — | — | SMA 13:pal0 (question marks) | — | — |
| Eyedrops | — | — | SMA 14:pal0 (small blinking) | — | — |
| Muzzled | — | — | SMA 15:pal0 (cross/X) | — | — |
| Miracle Eye | — | — | SMA 15:pal4 (cross/X) | — | — |
| Magnet Rise | — | — | SMA 10:pal7 (arrows) | — | — |
| Constriction | — | — | **none** | `DAMAGE_MESSAGE_CONSTRICTION` + effect anim from offset 0xC8 | — |
| Wrap (attacker) | — | — | **none** | `DAMAGE_MESSAGE_WRAP` | — |
| Wrapped (defender) | — | — | **none** | heals HP (Ingrain-like) | — |
| Leech Seed | — | — | **none** | `DAMAGE_MESSAGE_LEECH_SEED` (drains to source) | — |
| Perish Song | — | — | **none** | 999 damage when counter reaches 0 | — |
| Speed Up | effect `0x18B` | — | (via lowered_stat icon if applicable) | — | — |
| Speed Down | effect `0x18A` | — | (via lowered_stat icon if applicable) | — | — |
| Low HP | — | — | SMA 8:pal0 (vertical exclamation) | — (hardcoded check: hp < max/4) | — |
| Lowered Stat | — | — | SMA 10:pal3 (down arrows) | — (hardcoded check: any stat below default) | — |
| Grudge | — | — | SMA 9:pal7 (shield) | — | — |
| Exposed | — | — | SMA 14:pal4 (small blinking) | — | — |

---

## Damage Source Reference

The `damage_source` values passed to `ApplyDamageAndEffectsWrapper` for status ticks:

| Source Value | Hex | Used By |
|-------------|-----|---------|
| 583 | `0x247` | Burn tick |
| 584 | `0x248` | Constriction tick |
| 585 | `0x249` | Poison / Badly Poisoned tick |
| 586 | `0x24A` | (Wrap tick via DAT) |
| 587 | `0x24B` | Curse tick |
| 588 | `0x24C` | Leech Seed tick |
| 589 | `0x24D` | Perish Song |
| 592 | `0x250` | Hunger |

These correspond to `enum damage_source_non_move` values in pmdsky-debug headers.

**Source:** Hardcoded values in `FUN_0230fc24` per-tick handler.

---

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
| `MonsterCannotAttack` | — | Checks if status prevents action (no RNG) |
| `ShouldMonsterRunAway` | — | Tactic/ability flee check (fallthrough) |
| `ExecuteMonsterAction` | — | Main action dispatcher, calls MonsterCannotAttack |
| `RunFractionalTurn` | — | Per-fractional-turn orchestrator |
| `TickStatusAndHealthRegen` | — | Decrements all status duration counters |
| `TryInflictFrozenStatus` | — | Freeze infliction with immunity checks |
| `EndFrozenClassStatus` | — | Thaw/cure all freeze class statuses |
| `FUN_022e4c4c` | `0x022e4c4c` | Freeze application VFX (effect 0x15, palette 3) |
| `FUN_022e6798` | `0x022e6798` | Thaw VFX (effect 0x18) |
| `TryInflictSleepStatus` | — | Sleep infliction with immunity checks |
| `EndSleepClassStatus` | — | Wake-up for all sleep class statuses |
| `FUN_022e3e74` | `0x022e3e74` | Sleep application VFX (effect 0x25 + sound) |
| `InflictSleepStatusSingle` | — | Inner sleep setter (writes 0xBD, 0xBE) |
| `FreeOtherWrappedMonsters` | — | Releases wrapped monsters when wrapper's freeze class changes |
