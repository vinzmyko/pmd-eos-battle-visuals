# Special Monster Animation Types (98/99)

> **TODO:** Refactor this into `Systems/move_effect_pipeline.md` and `Systems/animation_timing.md` once validated.

## Summary

- Types 98 and 99 are special monster animation modes that bypass the normal hit-frame system
- Effect spawns **once at start**, not per-direction
- Animation loops through directions with fixed 2-frame timing
- Damage handled by caller AFTER animation completes (no synchronization needed)
- Simpler to implement than normal animations (no WAN frame flag parsing required)

## Affected Moves

### Type 98 (Multi-direction: 9 attacks, +2 direction each)

| Move ID | Move Name |
|---------|-----------|
| 18 | Cut |
| 362 | UNKNOWN (possibly unused) |

### Type 99 (Spin: 8 directions sequentially)

| Move ID | Move Name |
|---------|-----------|
| 31 | Weather Ball |
| 166 | Egg Bomb |
| 213 | Mud Slap |
| 277 | Present |
| 285 | Bone Rush |
| 290 | Bonemerang |
| 309 | Mist Ball |
| 329 | Leech Seed |
| 408 | Item Toss |
| 486 | Seed Bomb |
| 501 | Mud Bomb |
| 504 | Worry Seed |
| 543-546 | UNKNOWN (likely system/dummy entries) |

## Key Difference from Normal Animations

### Normal Animations (types 0-12)
```c
ChangeMonsterAnimation(entity, anim_type, direction);

for (frame = 0; frame < 0x78; frame++) {
    AdvanceFrame('Y');
    flags = FUN_0201d1d4(entity + 0xb);  // Read WAN frame flags
    
    // HIT FRAME: bit 2 triggers effect spawning
    if ((flags & 2) != 0 && !effect_spawned) {
        FUN_02325644(&params, entity, move, ...);  // Spawn effect on hit frame
        effect_spawned = true;
    }
    
    // ANIMATION COMPLETE: bit 1
    if ((flags & 1) != 0) break;
}
```

### Type 99 (Spin)
```c
FUN_02325644(&params, entity, move, ...);  // Effect spawns ONCE at start

for (i = 0; i < 8; i++) {
    direction = (direction + 1) & 7;  // Increment direction
    ChangeMonsterAnimation(entity, '\0', direction);
    FUN_022ea370(2, 0x15, ...);  // Fixed 2 frames per direction
}
// Total: 16 frames
```

### Type 98 (Multi-direction)
```c
FUN_02325644(&params, entity, move, ...);  // Effect spawns ONCE at start

for (i = 0; i < 9; i++) {
    current_dir = direction & 7;
    ChangeMonsterAnimation(entity, '\0', current_dir);
    FUN_022ea370(2, 0x15, ...);  // Fixed 2 frames per direction
    direction = current_dir + 2;  // Skip one direction (+2)
}
// Total: 18 frames
```

## Effect Spawning (FUN_02325644)

Spawns **layer 1** (secondary) effect only.

**Evidence:** `FUN_02325644`
```c
void FUN_02325644(ushort *param_1, int *param_2, int param_3, int param_4)
{
    apparent_weather = GetApparentWeather((entity *)param_2);
    uVar1 = GetMoveAnimationId((move *)(uint)*(ushort *)(param_3 + 4), apparent_weather, (bool)param_4);
    pmVar2 = GetMoveAnimation((uint)uVar1);
    
    // Only spawns if layer 1 effect exists
    if (*param_1 == 0 || pmVar2->field_0x2 == 0) {
        return;
    }
    
    FUN_02325d7c(param_1, 1, param_4, iVar3);
    AdvanceFrame('Z');
    iVar3 = FUN_022bed90(param_1, ...);  // Layer 1 handler
    FUN_022e6d68(iVar3, param_2, 1);     // Attach to entity
}
```

## Caller Context (FUN_02322374)

Damage is applied AFTER animation completes:
```c
// Animation plays (including types 98/99)
FUN_02322ddc((int *)param_1, (byte *)move, ...);

// THEN damage is calculated and applied
ExecuteMoveEffect(&local_128, (entity *)param_1, move, ...);
```

**Key insight:** No synchronization needed between animation and damage for types 98/99.

## Sound Effect Timing

Sound plays **once before the loop**, same as effect:

**Evidence:** `FUN_023250d4`
```c
// Sound plays here, before type 98/99 check
uVar11 = FUN_022bf0f4((int)*(short *)(iVar17 + 4), (uint)uVar5);
if (uVar11 != 0x3f00) {
    PlaySeByIdIfNotSilence(uVar11 & 0xffff);
}

// Then type 98/99 handling
if (iVar9 == 99) {
    FUN_02325644(...);  // Effect
    // direction loop...
}
```

## Implementation Notes

For recreation:
1. Play sound effect once (if not 0x3F00)
2. Spawn layer 1 effect once via `FUN_022bed90` equivalent
3. Loop sprite direction changes:
   - Type 99: 8 iterations, direction += 1, 2 frames each (16 total)
   - Type 98: 9 iterations, direction += 2, 2 frames each (18 total)
4. Animation complete - caller handles damage separately

## Functions Referenced

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_023250d4` | `0x023250d4` | Monster animation handler (contains 98/99 logic) |
| `FUN_02325644` | `0x02325644` | Spawn layer 1 effect |
| `FUN_022bed90` | `0x022bed90` | Layer 1 effect handler |
| `FUN_022ea370` | `0x022ea370` | Wait n frames |
| `FUN_0201d1d4` | `0x0201d1d4` | Read WAN frame flags (NOT used for 98/99) |
| `FUN_02322ddc` | `0x02322ddc` | Caller - sets up animation context |
| `FUN_02322374` | `0x02322374` | Higher caller - handles damage after animation |
