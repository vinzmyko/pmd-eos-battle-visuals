# Move Target Resolution

## Summary

- Moves store a packed `range_target` byte in waza_p.bin: high nibble = range (0-9, 15), low nibble = target (0-6, 15)
- `GetEntityMoveTargetAndRange` reads that byte and applies two per-move remaps before anyone else sees it
- `BuildMoveTargetList` applies one more remap for `TARGET_ENEMIES_AFTER_CHARGING` (target nibble = 4), then switches on the range nibble to collect candidate entities
- Each candidate passes through `FUN_023243b4` which validates it and filters by target nibble (ally/enemy/self/etc.)
- `FUN_02324c9c` runs after the list is built and moves the user to the END of the list (reorder, not filter)
- The 64-entry list is null-terminated; `ExecuteMoveEffect` loops until it hits a null

## Pipeline Overview

```
FUN_02322374 (per-strike dispatcher)
    │
    ├─► GetMoveRangeDistance → determines melee (< 2) vs projectile (≥ 2) path
    │
    └─► if melee:
            BuildMoveTargetList  ──►  GetEntityMoveTargetAndRange  (per-move remaps)
                │                      │
                │                      ├── Move 0xED (Curse) non-AI non-Ghost remap
                │                      └── Extend-Self-Effects exclusive item remap
                │
                ├── TARGET_ENEMIES_AFTER_CHARGING remap (charge-state-dependent)
                │
                ├── Switch on range nibble (0x00-0x90)
                │
                └─► FUN_023243b4 (per candidate)
                        │
                        ├── Per-move exclusions (self-skip for move 0xFC)
                        ├── Validity checks (alive, not offset 0xBC == 7)
                        ├── Target-nibble filter (ally/enemy/all/user/etc.)
                        ├── TwoTurnMoveForcedMiss veto
                        └── Write to output[index++] if valid
            FUN_02324c9c  (move user-entry to end)
            ExecuteMoveEffect  (iterate, dispatch per target)
```

## GetEntityMoveTargetAndRange — Per-Move Remaps

Wraps `GetMoveTargetAndRange` (which just reads the raw byte from waza_p.bin). Applies two overrides:

### Remap 1: Move 0xED (Curse), non-AI, non-Ghost user

Returns `0x73` unconditionally. Ghost-type users use the raw byte (which targets enemies). Non-Ghost users self-target to apply Curse's HP-cut buff.

```c
if (move->id == 0xed && !is_ai && !MonsterIsType(entity, TYPE_GHOST)) {
    return 0x73;
}
```

### Remap 2: `EXCLUSIVE_EFF_EXTEND_SELF_EFFECTS_TO_TEAM` + `IsMoveRangeString19`

Extends self-targeting moves to the whole team when the user carries the right exclusive item.

| Input `range_target` | Remapped |
|---|---|
| `MOVE_TARGET_AND_RANGE_SPECIAL_USER_HEALING` | `SPECIAL_USER_HEALING - 0x12` |
| `0x73` (range=USER, target=USER) | `0x61` (range=FLOOR, target=PARTY) |

`IsMoveRangeString19` is the "is this a self-effect move" flag, likely a waza_p.bin bit.

**Source:** Ghidra decompilation of `GetEntityMoveTargetAndRange`.

## BuildMoveTargetList — TARGET_ENEMIES_AFTER_CHARGING Remap

Target nibble `4` (`TARGET_ENEMIES_AFTER_CHARGING`) is always rewritten before the range switch:

```c
if ((range_target & 0xf) == 4) {
    is_charging = IsChargingTwoTurnMove(user, move);
    if (move->id == 0x97 && GetApparentWeather(user) == WEATHER_SUNNY) {
        is_charging = true;  // Solar Beam skips charge in sun
    }
    range_target = is_charging ? 0x00 : 0x73;
}
```

**Semantics:** byte `0x74` encodes "act like self-targeting during charge, enemy-front during release." Three cases:

| Move kind | `IsChargingTwoTurnMove` | Result |
|---|---|---|
| Non-charging move (Dragon Dance, Barrier, ...) | always false | `0x73` → self |
| Two-turn move during charge init (Solar Beam turn 1) | false | `0x73` → self (stores charge state) |
| Two-turn move on release (Solar Beam turn 2) | true | `0x00` → enemy front |
| Solar Beam in sun | false, but sun override | `0x00` → enemy front, single turn |

All the "User (self-target or front after charging)" moves in waza_p.bin have byte `0x74` and rely on this remap. The stored range nibble `7` on those entries is essentially a placeholder — the remap fully determines the effective range+target.

**Source:** `BuildMoveTargetList` decompilation.

## BuildMoveTargetList — Range Nibble Switch

Dispatches on `range_target & 0xf0`:

| Range | Mask | Candidate source |
|---|---|---|
| `RANGE_FRONT` | `0x00` | `FUN_022f87c0(user)` — entity in front tile |
| `RANGE_FRONT_AND_SIDES` | `0x10` | 3 positions: front + two corners, via direction table `DAT_023243ac` |
| `RANGE_NEARBY` | `0x20` | 8 surrounding tiles |
| `RANGE_ROOM` | `0x30` | All 20 entity slots, filtered by `FUN_022e28d4` (same-room check) |
| `RANGE_FRONT_2` | `0x40` | `FUN_022f8830` (tile 1); if null → `FUN_022f88c0` (tile 2) |
| `RANGE_FRONT_10` | `0x50` | Never reached — projectile path in caller |
| `RANGE_FLOOR` | `0x60` | All 20 entity slots, no room filter |
| `RANGE_USER` | `0x70` | `user` itself (single write) |
| `RANGE_FRONT_WITH_CORNER_CUTTING` | `0x80` | `FUN_022f8830` only, no fallback |
| `RANGE_FRONT_2_WITH_CORNER_CUTTING` | `0x90` | Never reached — projectile path |

Entity slot table at `*DAT_023243b0 + 0x12B78`, stride 4, 20 entries.

Direction offset tables: `DAT_023243ac` (xy pairs for 3-direction arc), global 8-direction table elsewhere.

`RANGE_SPECIAL=0xF0` has no branch — those moves presumably route around this function entirely.

## FUN_023243b4 — Candidate Validator + Writer

Signature inferred: `int TryAddEntityToTargetList(int index, entity **output, uint range_target, entity *user, entity *candidate, move *move, char attacker_flag)`

Returns the new index (incremented if candidate accepted, unchanged if rejected).

**Step 1 — Per-move self-skip:**
```c
if (move->id == 0xFC && user == candidate) return index;  // move 0xFC (252) never hits self
```

**Step 2 — Candidate validity:**
```c
if (candidate->info[0x9] != 1) return index;   // some "is targetable" flag
if (candidate->info[0xBC] == 7) return index;  // burn_class_status == 7? (something skippable)
```

**Step 3 — Target-nibble filter:** bypassed when `attacker_flag != 0` (confused attacker? → hits anyone).

| Target | Low nibble | Accepts candidate if |
|---|---|---|
| `TARGET_ENEMIES` | `0` | `GetTreatmentBetweenMonsters == TREAT_AS_ENEMY` |
| `TARGET_PARTY` | `1` | `GetTreatmentBetweenMonsters == TREAT_AS_ALLY` |
| `TARGET_ALL` | `2` | always |
| `TARGET_USER` | `3` | always (list should contain only the user by construction) |
| `TARGET_ENEMIES_AFTER_CHARGING` | `4` | same as `TARGET_ENEMIES` (shouldn't be reached post-remap, but handled) |
| `TARGET_ALL_EXCEPT_USER` | `5` | `candidate != user` |
| `TARGET_TEAMMATES` | `6` | `TREAT_AS_ALLY && candidate != user` |

**Step 4 — Two-turn invincibility veto:**
```c
if (TwoTurnMoveForcedMiss(candidate, move)) return index;  // Fly / Dig / Dive etc.
```

**Step 5 — Write & update side state:**
```c
if (accepted && index < 0x40) {
    output[index++] = candidate;
    // side effect: something to do with global offset 6 on candidate info, possibly last-targeted
}
```

The `param_1 < 0x40` guard enforces the 64-entry cap.

**Source:** Ghidra decompilation of `FUN_023243b4`.

## FUN_02324c9c — User-to-End Reorder

Two-pass reorder using a local scratch buffer (66 entries). Pass 1 copies all non-user entries into `local_108`. Pass 2 appends all user entries. Then writes scratch back over the input. Zeroes any trailing slots.

Effect: given `[A, user, B, C, null]`, produces `[A, B, C, user, null]`.

**Purpose:** probably ensures self-effects resolve after enemy-effects for multi-target moves that hit both (relevant for some stat-manipulation moves and room-range moves where the user is among the targets).

**Source:** Ghidra decompilation of `FUN_02324c9c`.

## Target List Structure

```c
struct target_list {
    entity *targets[0x40 + 1];  // 64 entries + null terminator
};
// Size: 0x104 (260 bytes)
```

Iterated in `ExecuteMoveEffect` with `for (local_9c = 0; local_9c < 0x40; local_9c++)` and early-exit on null.

## Metronome Dispatch Special Case

`DAT_02322d18 = 0x14A` (330) — Metronome's move ID. When a move with this ID reaches `FUN_02322374`:

```c
if (move->field_0x4 == 0x14A) {
    int idx = DungeonRandInt(0xA8);  // 168 possible moves
    InitMove(&local_138, *(u16*)(DAT_02322d20 + idx * 8));  // 8-byte stride lookup table
    // preserve original flags byte, re-dispatch with the rolled move
}
```

Also handled: `field_0x4 == 0x77` → Nature Power, uses `GetNaturePowerVariant()` instead of random.

Table `DAT_02322d20` has 168 entries × 8 bytes. First `u16` at each entry is the move ID; remaining bytes unknown.

## Cross-References

> See `Systems/move_effect_pipeline.md` for `ExecuteMoveEffect` details (consumes the target list)

> See `move_effect_mechanics.md` for `DoMoveX` wrappers (called per-target)

## Open Questions

- `FUN_023024e0` — returns the `attacker_flag` passed to `FUN_023243b4`. Confused status? IQ skill? Called twice with different second args (0 and 1).
- `FUN_022f87c0`, `FUN_022f8830`, `FUN_022f88c0` — tile/entity lookups for front-facing targets. Relationship between the three unclear.
- `FUN_022e28d4` — same-room check used by `RANGE_ROOM`. Needs confirmation.
- `IsMoveRangeString19` — waza_p.bin bit offset for "is self-effect" flag.
- Metronome table `DAT_02322d20` — confirm layout; probably `{u16 move_id, u16 ?, u16 ?, u16 ?}`.
- `FUN_023243b4` side effect at candidate `info[6]` (possibly "last targeted" tracking).

## Functions Used

| Function | Address (NA) | Purpose |
|---|---|---|
| `BuildMoveTargetList` | — | Main target resolver |
| `GetEntityMoveTargetAndRange` | — | Reads waza_p.bin byte + per-move remaps |
| `GetMoveTargetAndRange` | — | Raw waza_p.bin byte reader |
| `GetMoveRangeDistance` | — | Returns melee (0-1) vs projectile (2+) distance |
| `FUN_023243b4` | `0x023243b4` | Candidate validator + writer (`TryAddEntityToTargetList`) |
| `FUN_02324c9c` | `0x02324c9c` | Reorder: move user entries to end |
| `GetTreatmentBetweenMonsters` | — | ally/enemy/neutral classification |
| `TwoTurnMoveForcedMiss` | — | Dig/Fly/Dive invincibility check |
| `IsChargingTwoTurnMove` | — | True if user is in the charge state |
| `IsMoveRangeString19` | — | Self-effect flag from waza_p.bin |
| `ExclusiveItemEffectIsActive` | — | Exclusive item effect check |
| `MonsterIsType` | — | Type-match check |
| `GetApparentWeather` | — | Weather for Solar Beam sun bypass |
| `FUN_022f87c0` | `0x022f87c0` | Entity in front tile |
| `FUN_022f8830` | `0x022f8830` | Entity in front tile 1 (corner-cutting variant) |
| `FUN_022f88c0` | `0x022f88c0` | Entity in front tile 2 (range 2 fallback) |
| `FUN_022e28d4` | `0x022e28d4` | Same-room check |
| `FUN_023024e0` | `0x023024e0` | Attacker flag fetcher (possibly confused check) |

## Breadcrumbs

- Waza_p.bin offset for the range+target byte: find by looking at `GetMoveTargetAndRange` (not yet decompiled in your notes).
- `RANGE_SPECIAL=0xF0` moves: which moves use it, and where's their target resolution? Not in `BuildMoveTargetList`.
- `TARGET_SPECIAL=0xF` moves: same question.
- `MOVE_TARGET_AND_RANGE_SPECIAL_USER_HEALING` constant — exact value needed. Search pmdsky-debug headers.
- The `attacker_flag` bypass in `FUN_023243b4` is how confused monsters hit allies. Confirm by checking `FUN_023024e0` against confused status offset (0xD0 = 2).
- `DAT_02322d60` in `FUN_02322374` — another special-cased move ID in the melee path (sets user->info[0x170] = 1 before `ExecuteMoveEffect`). Worth identifying.
