## Commit 1: Create `move_target_and_range.md`

# Move Target and Range

## Summary

- `target_range` is a packed 16-bit field in `waza_p.bin` move entries
- Contains three 4-bit values: target (0-3), range (4-7), AI condition (8-11)
- Each move has two such fields: one for player use, one for AI use
- `GetMoveRangeDistance` always reads the AI range, never the player range
- Only three AI range values produce a nonzero projectile distance; everything else returns 0

## Field Layout

Each move in `waza_p.bin` contains two 16-bit packed fields:

| Offset | Field | Use |
|--------|-------|-----|
| 0x04 | target_range | Player-use targeting |
| 0x06 | ai_target_range | AI-use targeting |

### Bit Packing

```
Bits 0-3   (mask 0x000F)  → target
Bits 4-7   (mask 0x00F0)  → range
Bits 8-11  (mask 0x0F00)  → ai_use_condition
Bits 12-15                → unused
```

## Target Values (bits 0-3)

| Value | Name | Description |
|-------|------|-------------|
| 0 | Enemies | Hostile monsters only |
| 1 | Party (including user) | All allies + user |
| 2 | All (including user) | Everyone in range |
| 3 | User | Self only |
| 4 | Enemies (after charging) | Two-turn moves targeting enemies |
| 5 | All (except user) | Allies + enemies, not user |
| 6 | Teammates (excluding user) | Allies only |
| 15 | Special | Move-specific (Spikes, Curse, etc.) |

## Range Values (bits 4-7)

| Value | Name | Adjacent Tiles | Notes |
|-------|------|---------------|-------|
| 0 | Front (1 tile) | 1 | Standard melee |
| 1 | Front_and_Sides (cuts corners) | 3 | Wide melee |
| 2 | Nearby (8 surrounding tiles) | 8 | All adjacent |
| 3 | Room | All in room | Room-wide |
| 4 | Front_2 (2 tiles, cuts corners, AI unaware) | 2 | See critical note below |
| 5 | Front_10 (10 tiles) | 10 | Long-range projectile |
| 6 | Floor | All on floor | Floor-wide |
| 7 | User (self-target) | 0 | Self or front after charging |
| 8 | Front (cuts corners) | 1 | Single-tile projectile |
| 9 | Front_2 (2 tiles, cuts corners, AI aware) | 2 | See critical note below |
| 15 | Special | varies | Move-specific |

## GetMoveRangeDistance Behavior

`GetMoveRangeDistance` returns 0, 1, 2, or 10 — the projectile travel distance used by `FUN_023230fc` as `param_3` (drives outer-loop iteration count and amplitude branch).

**Only three AI range values produce a nonzero result. Everything else returns 0:**

| AI Range Value | Mask Match | GetMoveRangeDistance Returns |
|----------------|------------|------------------------------|
| 5 | `& 0xF0 == 0x50` | 10 |
| 8 | `& 0xF0 == 0x80` | 1 |
| 9 | `& 0xF0 == 0x90` | 2 |
| All others | — | 0 |

### Critical: Front_2 Variants

There are two "Front_2" range values (4 and 9) that look similar but behave differently:

- **Range 4** ("Front_2, AI unaware") → `GetMoveRangeDistance` returns **0**. Despite covering 2 tiles spatially, this value is NOT in the function's check. Moves with range 4 do not run the projectile motion loop.
- **Range 9** ("Front_2, AI aware") → `GetMoveRangeDistance` returns **2**. This is the version that actually drives 2-tile projectile travel.

The "AI aware" / "AI unaware" distinction refers to whether the AI considers the move's full 2-tile reach when picking targets. Only the AI-aware variant is treated as a true ranged move by the projectile system.

### Two-Turn Move Adjustment

When `check_two_turn_moves = true`, `GetMoveRangeDistance` zeroes its return value for two-turn moves that aren't currently charging. This prevents the projectile system from running on the charging turn of moves like Sky Attack or Razor Wind.

**Sunny weather exception:** Move 0x97 (Solar Beam) skips the two-turn check when weather is sunny — it fires immediately as a single-turn move.

**Evidence:** `GetMoveRangeDistance`
```c
iVar4 = 0;
mVar2 = GetEntityMoveTargetAndRange(user, move, '\x01');
if (((ushort)mVar2 & 0xf0) == 0x50) iVar4 = 10;
if (((ushort)mVar2 & 0xf0) == 0x80 || ((ushort)mVar2 & 0xf0) == 0x90) {
    if (((ushort)mVar2 & 0xf0) == 0x90) iVar4 = 2; else iVar4 = 1;
    // Two-turn move zeroing logic
    if (((move->id != MOVE_SOLARBEAM) || (weather != WEATHER_SUNNY)) &&
        check_two_turn_moves != '\0' &&
        Is2TurnsMove(move->id) &&
        !IsChargingTwoTurnMove(user, move)) {
        iVar4 = 0;
    }
}
return iVar4;
```

### Always AI Range, Never Player Range

The third parameter to `GetEntityMoveTargetAndRange` is `is_ai`. `GetMoveRangeDistance` always passes `'\x01'` (true), so it always reads the AI range field. The player-use range field at offset 0x04 of the move entry is irrelevant to projectile distance calculation.

## AI Condition Values (bits 8-11)

| Value | Description |
|-------|-------------|
| 0 | Always eligible |
| 1 | Random chance (uses `ai_random_use_chance` %) |
| 2 | Target HP <= 25% |
| 3 | Target has negative status |
| 4 | Target is asleep/napping/nightmare |
| 5 | Target HP <= 25% or has negative status |
| 6 | Target is Ghost type and not exposed |

## Cross-References

> See `Systems/projectile_motion.md` for how `GetMoveRangeDistance` drives projectile loop count and amplitude

> See `Data Structures/move_animation_info.md` for animation flags that work alongside range

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `GetMoveRangeDistance` | - | Returns 0/1/2/10 projectile travel distance |
| `GetEntityMoveTargetAndRange` | - | Resolves target_range with player/AI selection |
| `GetMoveTargetAndRange` | - | Direct field accessor on waza_p.bin entry |
| `Is2TurnsMove` | - | Check if move is two-turn |
| `IsChargingTwoTurnMove` | - | Check if user is currently charging |
| `GetApparentWeather` | - | For Solar Beam sunny exception |
