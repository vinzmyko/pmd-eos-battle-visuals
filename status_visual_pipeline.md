# Status Visual Pipeline (Dungeon Mode)

## Summary

This document covers the complete visual pipeline for status effects in PMD:EOS dungeon mode.
It describes what visual events occur when a status is applied, persists, ticks damage, and expires.
For the SMA icon format details and the full status→icon bit→SMA animation mapping, see the
companion document `status_icon_system.md`.

---

## Visual Pipeline Overview

Each status can trigger up to four visual stages:

```
1. APPLICATION   → One-shot effect.bin animation + sound + log message + icon appears
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
| Damage (attacker side) | `FUN_022e45d0` part 1 | `0x2F` | 47 | `022e45dc` (hardcoded) |
| Damage (defender side) | `FUN_022e45d0` part 2 | `0x30` | 48 | `022e4618` (hardcoded) |

**Source:** Ghidra disassembly of each function listed. Effect IDs read from DAT_ constants
or hardcoded immediates in the ARM instructions.

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
| Sleep / Yawning | `FUN_022e53ec` | `022e53ec` |
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
| Burn | 0xBF = 1 | 0xC1 | `DAT_02310aac` | `DAT_02310ab0` (fixed) | `DAMAGE_MESSAGE_BURN` (1) | `0x247` | — |
| Poison | 0xBF = 2 | 0xC1 | `DAT_02310ac0` | `DAT_02310ac4` (fixed) | `DAMAGE_MESSAGE_POISON` (3) | `0x249` | Poison Heal ability reverses to HP recovery |
| Badly Poisoned | 0xBF = 3 | 0xC1 | `DAT_02310ac8` | table at `DAT_02310acc` | `DAMAGE_MESSAGE_POISON` (3) | `0x249` | Escalating damage indexed by tick count (0xC2, caps at 29). Poison Heal reverses |
| Constriction | 0xC4 = 7 | 0xCD | `DAT_02310ad0` | `DAT_02310ad4` (fixed) | `DAMAGE_MESSAGE_CONSTRICTION` (2) | `0x248` | **EXCEPTION:** plays `PlayEffectAnimationEntityStandard` with anim from offset 0xC8 BEFORE damage |
| Wrap | 0xC4 = 4 | 0xCD | `DAT_02310ad8` | `DAT_02310adc` (fixed) | `DAMAGE_MESSAGE_WRAP` (5) | `DAT_02310ae0` | — |
| Wrapped/Ingrain | 0xC4 = 5 | 0xCD | `DAT_02310ae4` | `DAT_02310ae8` | — | — | **Heals HP** instead of dealing damage |
| Curse | 0xD8 = 1 | 0xDC | `DAT_02310aec` | max_hp / 4 (min 1) | `DAMAGE_MESSAGE_CURSE` (7) | `0x24b` | Damage calculated as `(max_hp_stat + max_hp_boost) / 4` |
| Leech Seed | 0xE0 = 1 | 0xEA | `DAT_02310af0` | `DAT_02310af4` (fixed) | `DAMAGE_MESSAGE_LEECH_SEED` (9) | `0x24c` | Drains HP to source pokemon. Liquid Ooze reverses (deals `DAMAGE_MESSAGE_SLUDGE` to drainer) |
| Perish Song | 0x106 | counter | — | 999 (instant KO) | `DAMAGE_MESSAGE_PERISH_SONG` (11) | `0x24d` | If Protect (0xD5 = 7) is active, just logs a message instead |

**Source:** All offsets and DAT_ references from Ghidra decompilation of `FUN_0230fc24`.

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
| Paralysis | effect `0x1A7` + speed down effect `0x18A` | Monster moves slower (reduced speed stage) |
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
| Sleep | — | — | SMA 14:pal4 (red Z's) | — | WAN anim 8 |
| Sleepless | — | — | SMA 0 (null?) | — | — |
| Nightmare | — | — | SMA 14:pal4 (red Z's) | damage via EndSleepClass | WAN anim 8 |
| Napping | — | — | SMA 14:pal4 (red Z's) | heals HP | WAN anim 8 |
| Burn | effect `0x4C` | — | SMA 1:pal0 (flame) | `DAMAGE_MESSAGE_BURN` | — |
| Poisoned | effect `0x13A` | — | SMA 2:pal0 (white skull) | `DAMAGE_MESSAGE_POISON` | — |
| Badly Poisoned | effect `0x13A` | — | SMA 3:pal11 (purple skull) | `DAMAGE_MESSAGE_POISON` (escalating) | — |
| Paralysis | effect `0x1A7` | — | **none** | — | — |
| Frozen | effect `0x15` | — | SMA (persistent, ice block) | — | — |
| Cringe | effect `0x143` | — | **none** | — | — |
| Confused | — | SE `0x310` | SMA 3:pal7 (birds) | — | — |
| Cowering | WAN anim 10 | — | SMA 5:pal0 (green lines) | — | — |
| Taunted | — | — | SMA 6:pal0 (fist) | — | — |
| Encore | — | — | SMA 7:pal0 (blue !) | — | — |
| Reflect | — | — | SMA 8:pal0 (blue shield) | — | — |
| Safeguard | — | — | SMA 9:pal0 (pink shield) | — | — |
| Light Screen | — | — | SMA 9:pal4 (gold shield) | — | — |
| Protect | — | — | SMA 9:pal3 (green shield) | — | — |
| Endure | — | — | SMA 9:pal10 (blue+red shield) | — | — |
| Curse | — | — | SMA 8:pal0 | `DAMAGE_MESSAGE_CURSE` (max_hp/4) | — |
| Embargo/Snatch/Gastro Acid | — | — | SMA 3:pal6 (yellow !) | — | — |
| Heal Block | — | — | SMA 10:pal3 (green cross) | — | — |
| Sure Shot | effect `0x05` | — | SMA 8:pal3 (blue sword) | — | — |
| Whiffer | — | SE `0x227` | SMA 11:pal0 (green lines) | — | — |
| Focus Energy | — | — | SMA 11:pal5 (red sword) | — | — |
| Blinker | effect `0x41` | — | SMA 11:pal4 (eye X) | — | — |
| Cross-Eyed | — | — | SMA 12:pal0 (blue ?) | — | — |
| Eyedrops | — | — | SMA 13:pal0 (eye+wave) | — | — |
| Muzzled | — | — | SMA 14:pal0 (red cross) | — | — |
| Miracle Eye | — | — | SMA 15:pal3 (orange cross) | — | — |
| Magnet Rise | — | — | SMA 8:pal4 (purple arrow up) | — | — |
| Constriction | — | — | **none** | `DAMAGE_MESSAGE_CONSTRICTION` + effect anim from offset 0xC8 | — |
| Wrap (attacker) | — | — | **none** | `DAMAGE_MESSAGE_WRAP` | — |
| Wrapped (defender) | — | — | **none** | heals HP (Ingrain-like) | — |
| Leech Seed | — | — | **none** | `DAMAGE_MESSAGE_LEECH_SEED` (drains to source) | — |
| Perish Song | — | — | **none** | 999 damage when counter reaches 0 | — |
| Speed Up | effect `0x18B` | — | (via lowered_stat icon if applicable) | — | — |
| Speed Down | effect `0x18A` | — | (via lowered_stat icon if applicable) | — | — |
| Low HP | — | — | SMA 9:pal5 (blue !) | — (hardcoded check: hp < max/4) | — |
| Lowered Stat | — | — | SMA 16:pal4 (yellow arrow down) | — (hardcoded check: any stat below default) | — |
| Grudge | — | — | SMA 15:pal0 (purple shield) | — | — |
| Exposed | — | — | SMA 9:pal7 (eye+wave) | — | — |

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
