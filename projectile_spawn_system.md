# Projectile Spawn System

> **TODO:** Refactor relevant sections into `Systems/projectile_motion.md` and `Systems/move_effect_pipeline.md` once validated.

## Summary

- Projectile source position is the attacker's pixel position (converted from 8.8 fixed point)
- Projectile destination is the target tile center (tile * 24 + offset)
- Attachment points are looked up but NOT applied to projectile start position
- Wave pattern (0/1/2) is determined at runtime by caller logic, not stored in data tables
- Direction comes from attacker's monster_info + 0x4C (0-7)
- Safe to use wave pattern 0 (straight line) for all moves as default

## Position Formulas

### Source Position (Projectile Start)

Projectile starts at **attacker's current pixel position**, converted from 8.8 fixed point to screen pixels.

| Component | Formula | Description |
|-----------|---------|-------------|
| X | `attacker->pixel_pos.x >> 8` | Screen X pixel |
| Y | `attacker->pixel_pos.y >> 8` | Screen Y pixel |

**Evidence:** `FUN_02322f78`
```c
local_28 = (undefined2)((uint)param_1[3] >> 8);  // attacker->pixel_pos.x >> 8
local_26 = (undefined2)((uint)param_1[4] >> 8);  // attacker->pixel_pos.y >> 8
```

These values are passed to `FUN_022be9e8` as `param_1[2]` and `param_1[3]`, then stored in the effect context:
```c
*(ushort *)(iVar10 + 0x128) = param_1[2];  // source_x
*(ushort *)(iVar10 + 0x12a) = param_1[3];  // source_y
```

### Destination Position (Projectile End)

Projectile ends at **target tile center**, using standard entity positioning offsets.

| Component | Formula | Offset | Description |
|-----------|---------|--------|-------------|
| X | `(tile_x * 24 + 12) * 256 >> 8` | +12 | Tile center X |
| Y | `(tile_y * 24 + 16) * 256 >> 8` | +16 | Below center Y (feet) |

**Evidence:** `FUN_02322f78`
```c
// param_2 is target tile position
local_30 = (undefined2)((uint)((*param_2 * 0x18 + 0xc) * 0x100) >> 8);   // tile_x * 24 + 12
local_2e = (undefined2)((uint)((param_2[1] * 0x18 + 0x10) * 0x100) >> 8); // tile_y * 24 + 16
```

These become `param_2[0]` and `param_2[1]` passed to `FUN_022be9e8`, stored as:
```c
*(undefined2 *)(iVar10 + 300) = *param_2;    // dest_x (offset 0x12C)
*(undefined2 *)(iVar10 + 0x12e) = param_2[1]; // dest_y
```

## Attachment Point Handling

Attachment points are looked up but **NOT added to projectile source position**.

### Lookup Process

**Evidence:** `FUN_02322f78`
```c
// Get attachment point index from move animation (with per-Pokemon override)
uVar2 = FUN_022bf01c((int)*(short *)(iVar5 + 4), (uint)uVar1);

if (uVar2 == 0xffffffff) {
    // No attachment point - use default offset (likely 0,0)
    sStack_24 = *DAT_023230f8;
    sStack_22 = DAT_023230f8[1];
}
else {
    // Calculate offset from WAN sprite data
    FUN_0201cf90(&sStack_24, (ushort *)(param_1 + 0xb), uVar2 & 0xff);
}
```

### Key Finding

The attachment point offset (`sStack_24`, `sStack_22`) is calculated but the **source position is still taken directly** from `param_1[3]` and `param_1[4]` (attacker pixel pos):

```c
// Source position - no attachment offset added
local_28 = (undefined2)((uint)param_1[3] >> 8);  // Direct from pixel_pos
local_26 = (undefined2)((uint)param_1[4] >> 8);  // Direct from pixel_pos
```

**Conclusion:** Attachment points may affect rendering/visual offset but do NOT modify the actual projectile trajectory source point.

## Wave Pattern System

Wave pattern is determined at **runtime by caller logic**, not stored in move/effect data tables.

### Pattern Values

| Pattern | Value | Description | Usage |
|---------|-------|-------------|-------|
| Straight | 0 | No wave, direct line | Default/most common |
| Vertical Sine | 1 | Up-down oscillation | Determined by `FUN_02322ddc` |
| Spiral | 2 | Circular motion | Determined by `FUN_02322ddc` |

### Source of Wave Pattern

**Evidence:** `FUN_02322374`
```c
// Wave pattern comes from FUN_02322ddc return value
bVar3 = FUN_02322ddc((int *)param_1, (byte *)move, (int)local_140, (uint)ptVar19, param_3,
                     iVar20 == 0);
uVar11 = (uint)bVar3;  // This becomes param_4 (wave pattern)

// Later passed to projectile motion
FUN_023230fc(param_1, (undefined2 *)move, (int)pmVar22, uVar11, param_3, param_5, bVar23);
//                                                      ^^^^^^ wave pattern
```

### Reverse Direction Projectiles

For boomerang/return effects (`FUN_0232393c`), wave pattern is always 0:

**Evidence:** `ExecuteMoveEffect`
```c
FUN_0232393c((int)attacker, (int *)entity, (int)auStack_74, uVar11, 0);  // Always 0 for wave
```

### Implementation Recommendation

**Use wave pattern 0 (straight line) for all moves** as a safe default. Wave patterns 1/2 are edge cases determined by complex game logic in `FUN_02322ddc` and can be added later if needed.

## Direction System

Direction is read from **attacker's monster_info at offset 0x4C**.

**Evidence:** `FUN_023230fc`
```c
uVar18 = param_1[0x2d];  // monster_info pointer
// ...
uVar18 = (uint)*(byte *)(uVar18 + 0x4c);  // direction byte (0-7)
iVar16 = *(int *)(DAT_02323900 + uVar18 * 4);
sVar3 = *(short *)(DAT_023238f8 + uVar18 * 4);  // X delta from direction table
sVar4 = *(short *)(DAT_023238fc + uVar18 * 4);  // Y delta from direction table
```

### Direction Values

| Index | Direction | X Delta | Y Delta |
|-------|-----------|---------|---------|
| 0 | Down | 0 | +1 |
| 1 | Down-Right | +1 | +1 |
| 2 | Right | +1 | 0 |
| 3 | Up-Right | +1 | -1 |
| 4 | Up | 0 | -1 |
| 5 | Up-Left | -1 | -1 |
| 6 | Left | -1 | 0 |
| 7 | Down-Left | -1 | +1 |

### Reverse Direction

For return projectiles, direction is rotated 180°:

**Evidence:** `FUN_0232393c`
```c
iVar4 = (*(byte *)(iVar2 + 0x4c) + 4 & 7) * 4;  // (direction + 4) & 7 = 180° rotation
iVar11 = (int)*(short *)(DAT_02323c30 + iVar4);  // Reversed X delta
sVar1 = *(short *)(DAT_02323c34 + iVar4);        // Reversed Y delta
```

## Speed System

Speed is read from `move_animation_info.projectile_speed` and mapped to actual values.

### Speed Mapping

| Raw Value | Mapped Speed | Frame Count | Description |
|-----------|--------------|-------------|-------------|
| 0 | 6 | 4 frames | Fast/instant |
| 1 | 2 | 12 frames | Slow |
| 2 | 3 | 8 frames | Medium |
| 3+ | 6 | 4 frames | Fast (default) |

**Evidence:** `FUN_023230fc`
```c
iVar7 = GetMoveAnimationSpeed((uint)(ushort)param_2[2]);
if (iVar7 == 1) {
    iVar7 = 2;
}
else if (iVar7 == 2) {
    iVar7 = 3;
}
else {
    iVar7 = 6;
}
```

### Frame Count Calculation

```c
frame_count = 24 / mapped_speed;  // 0x18 / iVar7
// Results: 24/6=4, 24/2=12, 24/3=8
```

### Velocity Calculation

```c
velocity = mapped_speed * 256;  // Fixed point
// Results: 1536, 512, or 768
```

## Effect Context Storage

Projectile data is stored in the effect context structure (316 bytes per effect).

| Offset | Size | Field | Source |
|--------|------|-------|--------|
| 0x128 | int16 | source_x | Attacker pixel_pos.x >> 8 |
| 0x12A | int16 | source_y | Attacker pixel_pos.y >> 8 |
| 0x12C | int16 | dest_x | Target tile center X |
| 0x12E | int16 | dest_y | Target tile center Y |
| 0x130 | uint16 | entity_id | Attacker entity reference |
| 0x132 | int16 | stored_velocity_x | From context + 0x24 |
| 0x134 | int16 | stored_velocity_y | From context + 0x26 |

**Evidence:** `FUN_022be9e8`
```c
iVar10 = iVar11 * 0x13c + *DAT_022beb28;  // Context base + index * 316
*(ushort *)(iVar10 + 0x128) = param_1[2];  // source_x
*(ushort *)(iVar10 + 0x12a) = param_1[3];  // source_y
*(undefined2 *)(iVar10 + 300) = *param_2;   // dest_x (0x12C)
*(undefined2 *)(iVar10 + 0x12e) = param_2[1]; // dest_y
*(ushort *)(iVar10 + 0x130) = param_1[1];   // entity_id
*(undefined2 *)(iVar10 + 0x132) = *(undefined2 *)(iVar10 + 0x24);
*(undefined2 *)(iVar10 + 0x134) = *(undefined2 *)(iVar10 + 0x26);
```

## Call Chain

```
FUN_02322374 (Move execution coordinator)
    │
    ├─► FUN_02322ddc (Returns wave pattern 0/1/2)
    │
    └─► FUN_023230fc (Main projectile handler)
            │
            ├─► FUN_02322f78 (Spawns projectile effect)
            │       │
            │       ├─► FUN_022bf01c (Get attachment point index)
            │       │
            │       ├─► FUN_0201cf90 (Calculate attachment offset - NOT USED for position)
            │       │
            │       └─► FUN_022be9e8 (Layer 3 projectile setup)
            │               │
            │               └─► FUN_022be780 (Effect dispatcher, type 2)
            │
            └─► Animation loop (FUN_022beb2c per frame)
```

## Implementation Summary

For accurate projectile recreation:

| Parameter | Value |
|-----------|-------|
| **Source X** | `attacker.pixel_pos.x >> 8` |
| **Source Y** | `attacker.pixel_pos.y >> 8` |
| **Dest X** | `target_tile.x * 24 + 12` |
| **Dest Y** | `target_tile.y * 24 + 16` |
| **Direction** | `attacker.monster_info[0x4C]` (0-7) |
| **Wave Pattern** | `0` (straight line - safe default) |
| **Speed** | From `move_animation_info.projectile_speed` mapped via table |

## Open Questions

- Exact logic in `FUN_02322ddc` that determines wave patterns 1/2
- Whether attachment point offsets affect visual rendering separately from trajectory
- Purpose of stored_velocity_x/y at context 0x132/0x134 (copied from 0x24/0x26)
- Complete list of moves that use wave patterns 1 or 2

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_02322374` | `0x02322374` | Move execution coordinator, determines wave pattern |
| `FUN_02322ddc` | `0x02322ddc` | Returns wave pattern (0/1/2) based on move logic |
| `FUN_02322f78` | `0x02322f78` | Spawns projectile effect with position data |
| `FUN_023230fc` | `0x023230fc` | Main forward projectile motion handler |
| `FUN_0232393c` | `0x0232393c` | Reverse direction projectile handler |
| `FUN_022be9e8` | `0x022be9e8` | Layer 3 projectile effect setup |
| `FUN_022be780` | `0x022be780` | Main effect dispatcher |
| `FUN_022bf01c` | `0x022bf01c` | Get attachment point index with override |
| `FUN_022bf088` | `0x022bf088` | Get attachment point for species |
| `FUN_0201cf90` | `0x0201cf90` | Calculate attachment point offset from WAN |
| `FUN_022beb2c` | `0x022beb2c` | Update effect position during flight |
| `FUN_022bde50` | `0x022bde50` | Stop/cleanup effect |
| `GetMoveAnimationSpeed` | - | Read projectile_speed from move_animation |
