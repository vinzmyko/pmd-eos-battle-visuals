# Projectile Motion

## Summary

- Projectile speed has 3 levels: slow (12 frames), medium (8 frames), fast (4 frames)
- Direction uses 8-direction lookup table (DIRECTIONS_XY)
- Three wave patterns: straight (0), vertical sine (1), spiral (2)
- Position updated per frame in 8.8 fixed point
- Reverse direction adds 4 to direction index (180° rotation)

## Speed System

### Raw to Mapped Speed

The `projectile_speed` field in move_animation_info is mapped to actual values:

| Raw Value | Mapped Value | Frame Count | Description |
|-----------|--------------|-------------|-------------|
| 1 | 2 | 12 frames | Slow projectile |
| 2 | 3 | 8 frames | Medium projectile |
| Other (0, 3+) | 6 | 4 frames | Fast projectile |

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
// Results: 24/2=12, 24/3=8, 24/6=4
```

**Evidence:** `FUN_023230fc`
```c
uVar20 = _s32_div_f(0x18, iVar7);
```

### Velocity Calculation

Velocity is speed multiplied by fixed point scale:
```c
velocity = mapped_speed * 256;  // iVar7 * 0x100
// Results: 512, 768, or 1536 in fixed point
```

**Evidence:** `FUN_023230fc`
```c
iVar7 = iVar7 * 0x100;
```

## Direction System

### Direction Table

8-direction vectors stored in `DIRECTIONS_XY`:

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

**Evidence:** `pmd-sky/src/dungeon_util.c`
```c
const struct position DIRECTIONS_XY[] = {
    {0, 1},    // 0: Down
    {1, 1},    // 1: Down-Right
    {1, 0},    // 2: Right
    {1, -1},   // 3: Up-Right
    {0, -1},   // 4: Up
    {-1, -1},  // 5: Up-Left
    {-1, 0},   // 6: Left
    {-1, 1}    // 7: Down-Left
};
```

### Direction Lookup

Direction is read from monster_info and used as table index:

**Evidence:** `FUN_023230fc`
```c
uVar18 = (uint)*(byte *)(uVar18 + 0x4c);  // Read direction from monster_info
iVar16 = *(int *)(DAT_02323900 + uVar18 * 4);
sVar3 = *(short *)(DAT_023238f8 + uVar18 * 4);  // X delta
sVar4 = *(short *)(DAT_023238fc + uVar18 * 4);  // Y delta
```

### Memory Addresses

| Symbol | Address (NA) | Contents |
|--------|--------------|----------|
| DAT_023238f8 | 0x023238f8 | Pointer to X deltas |
| DAT_023238fc | 0x023238fc | Pointer to Y deltas |
| DAT_02323900 | 0x02323900 | Additional direction data |
| DAT_02323908 | 0x02323908 | Z priority data |
| DIRECTIONS_XY | 0x0235171C | Direction vector table |

### Reverse Direction

For boomerang/return effects, direction is rotated 180°:
```c
reversed_direction = (original_direction + 4) & 7;
```

**Evidence:** `FUN_0232393c`
```c
iVar4 = (*(byte *)(iVar2 + 0x4c) + 4 & 7) * 4;
iVar11 = (int)*(short *)(DAT_02323c30 + iVar4);  // Reversed X delta
sVar1 = *(short *)(DAT_02323c34 + iVar4);        // Reversed Y delta
```

## Wave Patterns

### Pattern Types

| Pattern | param_4 Value | Description |
|---------|---------------|-------------|
| Straight | 0 | No wave, direct line |
| Vertical Sine | 1 | Up-down oscillation perpendicular to travel |
| Spiral | 2 | Circular/helical motion |

### Amplitude Calculation

Amplitude depends on projectile range:
```c
if (param_3 < 2) {
    amplitude = 0x20;  // 32 - short range, big wave
} else {
    amplitude = 8;     // 8 - long range, small wave
}
```

**Evidence:** `FUN_023230fc`
```c
if (param_3 < 2) {
    iVar17 = 0x20;
}
else {
    iVar17 = 8;
}
```

### Phase Progression

Phase advances to complete half a sine wave over the projectile's travel:
```c
phase = 0;                          // Starting phase (20-bit fixed point)
phase_step = 0x80000 / frame_count; // Half cycle over travel

// Per frame:
angle = phase >> 8;                 // Convert to 12-bit for trig functions
phase += phase_step;
```

**Evidence:** `FUN_023230fc`
```c
iVar19 = 0;
uVar21 = _s32_div_f(0x80000, 0);  // Note: divisor seems to be frame_count in context
// ...
iVar14 = iVar19 >> 8;
// ...
iVar19 = iVar19 + (int)uVar21;
```

### Pattern 0: Straight Line

No wave offset applied:
```c
wave_x = 0;
wave_y = 0;
```

**Evidence:** `FUN_023230fc`
```c
else {
    iVar13 = 0;
    local_98 = 0;
}
```

### Pattern 1: Vertical Sine Wave

Single sine wave perpendicular to travel:
```c
wave_y = amplitude * SinAbs4096(angle);
wave_x = 0;
```

**Evidence:** `FUN_023230fc`
```c
if (param_4 == 1) {
    local_98 = SinAbs4096(iVar14);
    local_98 = iVar17 * local_98;
    iVar13 = 0;
}
```

### Pattern 2: Spiral/Circular Motion

Uses two angles for circular motion:
```c
// Primary angle from phase
radius = (amplitude / 2) * SinAbs4096(angle) >> 8;

// Secondary angle based on direction
secondary_angle = (direction_angle + 0xC00) & 0x1FFF;

wave_y = radius * SinAbs4096(secondary_angle);
wave_x = radius * CosAbs4096(secondary_angle);
```

**Evidence:** `FUN_023230fc`
```c
else if (param_4 == 2) {
    iVar13 = SinAbs4096(iVar14);
    iVar13 = (iVar17 >> 1) * iVar13 >> 8;
    local_98 = SinAbs4096(x);
    local_98 = iVar13 * local_98;
    iVar9 = CosAbs4096(x);
    iVar13 = iVar13 * iVar9;
}
```

### Secondary Angle Setup
```c
uVar15 = direction_angle + 0xC00;   // Add 3/4 turn (270°)
x = uVar15 & DAT_02323904;          // Mask to valid range (0x1FFF)
```

## Position Update Loop

### Per-Frame Update
```c
// Starting position (8.8 fixed point)
pos_x = (tile_x * 24 + 12) * 256;
pos_y = (tile_y * 24 + 16) * 256;

for (frame = 0; frame < frame_count; frame++) {
    // Calculate wave offset
    angle = phase >> 8;
    // ... wave pattern calculation ...
    
    // Apply wave for rendering
    render_x = (pos_x + wave_x) >> 8;
    render_y = (pos_y - wave_y) >> 8;  // Y is SUBTRACTED
    
    // Update effect position
    FUN_022beb2c(effect_handle, &render_pos, z_priority);
    
    // Advance frame
    AdvanceFrame('0');
    
    // Move projectile
    pos_x += delta_x * velocity;
    pos_y += delta_y * velocity;
    
    // Advance phase
    phase += phase_step;
}
```

**Evidence:** `FUN_023230fc`
```c
for (local_70 = 0; local_70 < (int)uVar20; local_70 = local_70 + 1) {
    // ... wave calculations ...
    
    local_4c = (undefined2)((uint)(local_8c + iVar13) >> 8);
    local_4a = (undefined2)((uint)(iVar8 - local_98) >> 8);
    FUN_022beb2c((int)sVar1, &local_4c, z_priority);
    
    AdvanceFrame('0');
    
    local_8c = local_8c + sVar3 * iVar7;
    iVar8 = iVar8 + sVar4 * iVar7;
    
    iVar19 = iVar19 + (int)uVar21;
}
```

### Z Priority Calculation

Z priority based on Y position relative to camera:
```c
z_priority = base_z + (render_y - camera_y) / 2;
```

**Evidence:** `FUN_023230fc`
```c
FUN_022beb2c((int)sVar1, &local_4c,
             iVar16 + ((iVar8 >> 8) - (int)*(short *)(*DAT_0232390c + DAT_02323910)) / 2);
```

## Effect Cleanup

After projectile completes, effect is stopped:
```c
if (effect_handle >= 0) {
    FUN_022bde50((int)(short)effect_handle);
}
```

**Evidence:** `FUN_023230fc`
```c
if (-1 < local_38) {
    FUN_022bde50((int)(short)local_38);
}
```

## Dual Projectile Support

`FUN_023230fc` can handle two simultaneous projectiles (e.g., attacker and target effects):

- `local_38` / `sVar1`: Primary effect handle
- `local_34` / `iVar11` / `sVar2`: Secondary effect handle

Both are updated and cleaned up independently.

## Trigonometry

### Angle Format

- 12-bit angles: 4096 units = 360°
- 0x000 = 0°, 0x400 = 90°, 0x800 = 180°, 0xC00 = 270°

### Functions

- `SinAbs4096(angle)`: Sine lookup, returns signed value
- `CosAbs4096(angle)`: Cosine lookup, returns signed value

Both use a quarter-wave table and derive other quadrants.

## Open Questions

- Exact contents of DAT_02323900 (additional direction data)
- How param_3 (range) is determined by caller
- Purpose of FUN_0234b4cc calls (enable/disable something)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_023230fc` | `0x023230fc` | Main projectile motion handler |
| `FUN_0232393c` | `0x0232393c` | Reverse direction projectile handler |
| `FUN_022beb2c` | `0x022beb2c` | Update effect position during flight |
| `FUN_022bde50` | `0x022bde50` | Stop/cleanup effect |
| `GetMoveAnimationSpeed` | - | Read speed from move animation table |
| `SinAbs4096` | - | Sine with 4096-step angles |
| `CosAbs4096` | - | Cosine with 4096-step angles |
| `AdvanceFrame` | - | Wait one frame |
| `FUN_0234b4cc` | `0x0234b4cc` | Enable/disable something (called with 1/0) |
