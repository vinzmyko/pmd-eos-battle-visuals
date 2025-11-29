# Reverse Engineering Findings: Move Effect Animation System

## Overview

This document covers the complete pipeline from move execution to effect animation playback, including projectile velocity calculation, wave motion patterns, and the critical discovery of how animation indices are interpreted for effect sprites.

ROM Version: NA release (C2SE)

---

## Critical Discovery: Animation Index Interpretation

### The Core Finding

**For effect sprites (PROPS_UI / sprite_type 0), `animation_index` is a FLAT SEQUENCE INDEX into group 0, NOT a group index.**

The ROM ignores the `animation_group_id` parameter entirely for effect sprites and always uses:
```
animations[0].pnt[animation_index]
```

### Evidence: SetAnimationForAnimationControlInternal

Address: `0x0201C5C4` (approximate)

```c
if (*(char *)&wan_header->sprite_type == '\x01') {
    // PATH A: CHARA sprites (type 1) - character sprites
    // Uses BOTH group and sequence
    pwVar3 = pwVar8->animations[animation_group_id].pnt[animation_id];
    anim_ctrl->animations_or_group_len = pwVar8->nb_animation_groups;
}
else {
    // PATH B: PROPS_UI sprites (type 0) - effect sprites
    // IGNORES animation_group_id, uses sequence directly from group 0
    pwVar3 = pwVar8->animations->pnt[animation_id];  // animations[0].pnt[animation_id]
    anim_ctrl->animations_or_group_len = pwVar8->animations->len;
}
```

### Bounds Checking

The ROM clamps out-of-bounds animation indices to 0:
```c
uVar1 = pwVar8->animations[animation_group_id].len;
if ((int)(uint)uVar1 <= animation_id) {
    animation_id = 0;  // Clamp to zero if out of bounds
}
```

### Implications for ROM Scrapers

```python
# WRONG - treating animation_index as group index
group = wan.animations[animation_index]  # Fails when animation_index >= num_groups

# CORRECT - how the ROM actually does it
group = wan.animations[0]                 # Always group 0 for effects
sequence = group.pnt[animation_index]     # animation_index is sequence index
```

---

## Direction System: When Direction is Added to Animation Index

### The Bit Manipulation Check

In `FUN_022bdfc0`, direction is conditionally added to `animation_index`:

```c
*(int *)(param_1 + 0x50) = peVar3->animation_index;
iVar6 = *(int *)(param_1 + 0x1c);  // Direction (0-7 or -1)

if ((iVar6 != -1) &&
    (uVar1 = *(int *)(param_1 + 0x10) >> 0x1f,
    (*(int *)(param_1 + 0x10) * 0x20000000 + uVar1 >> 0x1d | uVar1 << 3) == uVar1)) {
    *(int *)(param_1 + 0x50) = *(int *)(param_1 + 0x50) + iVar6;
}
```

### Decoded Logic

The check at `param_1 + 0x10` evaluates:
- For a valid pointer (0x02xxxxxx): Check **FAILS** → direction NOT added
- For NULL (0x00000000): Check **PASSES** → direction IS added

### When Each Path is Taken

```c
// param_1 + 0x10 is set by FUN_0201da20 for types 1-4:
if (*(int *)(param_1 + 0x40) != 5 && *(int *)(param_1 + 0x40) != 6) {
    uVar4 = FUN_0201da20(*(short *)(param_1 + 100));  // Returns valid WAN pointer
    *(undefined4 *)(param_1 + 0x10) = uVar4;
}
// Types 5-6 skip this, leaving 0x10 as NULL
```

| Effect Type | Value at 0x10 | Direction Check | Direction Added? |
|-------------|---------------|-----------------|------------------|
| 1-4 (WAN) | Valid pointer | FAILS | **NO** |
| 5-6 (Screen) | NULL | PASSES | YES (but unused) |

### Conclusion

**For standard WAN-based effects (types 1-4), direction is NEVER added to animation_index.** Effects are non-directional in terms of sprite selection. Direction only affects projectile movement velocity.

---

## Move → Effect Pipeline

### Main Orchestrator: FUN_023250d4

Address: `0x023250d4`

This is the central function that coordinates all move animation playback.

```c
void FUN_023250d4(int *param_1, uint param_2, undefined4 param_3, uint param_4)
{
    // 1. Check for alternative animation (two-turn moves)
    should_play_alternative_animation = ShouldMovePlayAlternativeAnimation(entity, move);
    
    // 2. Get weather for Weather Ball
    weather = GetApparentWeather(entity);
    
    // 3. Resolve actual move animation ID
    move_anim_id = GetMoveAnimationId(move, weather, should_play_alternative);
    
    // 4. Get monster animation type (with per-Pokemon override)
    monster_anim = FUN_022bfa3c(sprite_id, move_anim_id);
    
    // 5. Get move animation data
    move_animation = GetMoveAnimation(move_anim_id);
    
    // 6. Set up position from entity
    local_30 = entity_direction;  // from entity + 0x4C
    local_38 = pixel_x >> 8;
    local_36 = pixel_y >> 8;
    
    // 7. Play sound effect
    se_id = FUN_022bf0f4(sprite_id, move_anim_id);
    if (se_id != 0x3F00) {
        PlaySeByIdIfNotSilence(se_id);
    }
    
    // 8. Handle special animation types (99 = spin, 0x62 = multi-direction)
    // 9. Change monster sprite animation
    // 10. Spawn effects via layer handlers
}
```

### Layer Handlers

| Layer | Offset | Handler Function | Purpose |
|-------|--------|------------------|---------|
| 0 | 0x00 | `FUN_022bfaa8` | Charge/preparation effects |
| 1 | 0x02 | `FUN_02325644` → `FUN_022bed90` | Secondary/multi-hit effects |
| 2 | 0x04 | `FUN_022bfc5c` | Primary visual effect |
| 3 | 0x06 | `FUN_022be9e8` | Projectile effects |

---

## Layer Handler Details

### FUN_022bfaa8 - Layer 0 Handler

Address: `0x022bfaa8`

```c
void FUN_022bfaa8(ushort *param_1)
{
    // Get move animation
    pmVar3 = GetMoveAnimation((uint)*param_1);
    
    // Read Layer 0 effect_id
    local_38[0] = (uint)pmVar3->field_0x0;  // Offset 0x00
    
    // Copy position data
    local_38[1] = *(uint *)(param_1 + 8);
    local_38[2] = *(uint *)(param_1 + 6);
    local_2c = param_1[2];  // move_id
    // ... more position fields ...
    
    // Get attachment point from effect animation
    peVar4 = GetEffectAnimation((int)pmVar3->field_0x0);
    local_24[0] = peVar4->field_0x19;  // attachment_point
    
    // Spawn effect
    FUN_022be780(5, local_38, 0);
}
```

### FUN_022bfc5c - Layer 2 Handler

Address: `0x022bfc5c`

```c
void FUN_022bfc5c(ushort *param_1)
{
    pmVar3 = GetMoveAnimation((uint)*param_1);
    
    // Read Layer 2 effect_id
    local_38[0] = (uint)pmVar3->field_0x4;  // Offset 0x04
    
    // ... same structure as Layer 0 ...
    
    FUN_022be780(6, local_38, 0);
}
```

### FUN_02325644 - Layer 1 Handler

Address: `0x02325644`

```c
void FUN_02325644(ushort *param_1, int *param_2, int param_3, int param_4)
{
    // Get move animation
    move_animation = GetMoveAnimation(move_anim_id);
    
    // Check if layer 1 has an effect
    if (*param_1 == 0 || move_animation->field_0x2 == 0) {
        return;  // No effect
    }
    
    // Setup screen effects if needed
    FUN_02325d7c(param_1, 1, param_4, iVar3);
    
    AdvanceFrame('Z');
    
    // Spawn the effect
    iVar3 = FUN_022bed90(param_1, ...);
    
    // Attach to entity
    FUN_022e6d68(iVar3, param_2, 1);
}
```

### FUN_022be9e8 - Layer 3 (Projectile) Handler

Address: `0x022be9e8`

```c
int FUN_022be9e8(ushort *param_1, undefined2 *param_2, ...)
{
    pmVar3 = GetMoveAnimation((uint)*param_1);
    
    // Read Layer 3 effect_id
    local_40[0] = (uint)pmVar3->field_0x6;  // Offset 0x06
    
    // Get attachment point with per-Pokemon override
    iVar11 = FUN_022bf088((int)(short)param_1[1], (uint)*param_1);
    local_2c[0] = (undefined)iVar11;
    
    // Spawn effect
    iVar4 = FUN_022be780(2, local_40, 0);
    
    if (iVar4 != -1) {
        // Get effect context
        iVar11 = FUN_022be9a0((int)(short)iVar4);
        iVar10 = iVar11 * 0x13c + *DAT_022beb28;
        
        // Store projectile trajectory data
        *(ushort *)(iVar10 + 0x128) = param_1[2];   // source_x
        *(ushort *)(iVar10 + 0x12a) = param_1[3];   // source_y
        *(undefined2 *)(iVar10 + 0x12c) = *param_2; // dest_x
        *(undefined2 *)(iVar10 + 0x12e) = param_2[1]; // dest_y
        *(ushort *)(iVar10 + 0x130) = param_1[1];   // entity_id
        *(undefined2 *)(iVar10 + 0x132) = *(undefined2 *)(iVar10 + 0x24); // velocity_x
        *(undefined2 *)(iVar10 + 0x134) = *(undefined2 *)(iVar10 + 0x26); // velocity_y
    }
    
    return iVar11;
}
```

---

## Effect Spawn Parameter Structure

The parameter block passed to `FUN_022be780`:

```c
struct effect_spawn_params {
    /* 0x00 */ uint32_t effect_id;          // From move_animation layer field
    /* 0x04 */ uint32_t param_from_caller;  // From param_1[8]
    /* 0x08 */ uint32_t direction;          // Entity direction (0-7) from entity+0x4C
    /* 0x0C */ uint16_t move_id;            // Current move ID
    /* 0x0E */ uint16_t unknown;
    /* 0x10 */ uint16_t source_x;           // Attacker pixel X >> 8
    /* 0x12 */ uint16_t source_y;           // Attacker pixel Y >> 8
    /* 0x14 */ uint16_t target_x;           // Target pixel X >> 8
    /* 0x16 */ uint16_t target_y;           // Target pixel Y >> 8
    /* 0x18 */ int8_t attachment_point;     // From effect_animation->field_0x19
    /* 0x19 */ uint8_t padding[3];
    /* 0x1C */ ... template data from DAT_022bfb64/etc ...
};
```

---

## Effect Allocation: FUN_022be730

Address: `0x022be730`

Wrapper that allocates effect context and initializes animation:

```c
int FUN_022be730(int param_1, int *param_2, int param_3)
{
    // Allocate effect context
    iVar1 = FUN_022be44c(param_1, param_2, param_3);
    
    if (iVar1 == -1) {
        return -1;
    }
    
    // Get context pointer
    iVar2 = FUN_022be9a0((int)(short)iVar1);
    
    // Initialize animation from effect_animation data
    FUN_022bdfc0(*DAT_022be77c + iVar2 * 0x13c, 
                 *(int *)(*DAT_022be77c + iVar2 * 0x13c));
    
    return (int)(short)iVar1;
}
```

---

## Projectile Velocity System

### Direction Lookup Table

The direction table is referenced via pointer:
- `DAT_023238f8` contains pointer to `DIRECTIONS_XY` at `0x0235171C`

Standard PMD 8-direction grid vectors:

| Direction | Index | (dx, dy) |
|-----------|-------|----------|
| Down | 0 | (0, +1) |
| Down-Right | 1 | (+1, +1) |
| Right | 2 | (+1, 0) |
| Up-Right | 3 | (+1, -1) |
| Up | 4 | (0, -1) |
| Up-Left | 5 | (-1, -1) |
| Left | 6 | (-1, 0) |
| Down-Left | 7 | (-1, +1) |

### Speed Mapping

From `FUN_023230fc`:

```c
iVar7 = GetMoveAnimationSpeed((uint)(ushort)param_2[2]);
if (iVar7 == 1) {
    iVar7 = 2;      // Slow
} else if (iVar7 == 2) {
    iVar7 = 3;      // Medium
} else {
    iVar7 = 6;      // Fast (default)
}
```

| Raw Speed | Mapped Multiplier | Frame Count (24/mult) |
|-----------|-------------------|----------------------|
| 1 | 2 | 12 frames |
| 2 | 3 | 8 frames |
| other | 6 | 4 frames |

### Velocity Calculation

```c
// Scale to fixed-point (8.8 format)
iVar7 = iVar7 * 0x100;  // speed * 256

// Get direction deltas
sVar3 = *(short *)(DAT_023238f8 + direction * 4);  // X delta
sVar4 = *(short *)(DAT_023238fc + direction * 4);  // Y delta

// Position update per frame
local_8c = local_8c + sVar3 * iVar7;  // X += delta_x * speed
iVar8 = iVar8 + sVar4 * iVar7;        // Y += delta_y * speed
```

### Frame Count Calculation

```c
uVar20 = _s32_div_f(0x18, iVar7);  // 24 / mapped_speed = frame count
```

---

## Wave Motion Patterns

### Trigonometry System

From `pmd-sky/src/main_020018D0.c`:

- **Angle format**: 12-bit (4096 units = 360°)
- **Sine table**: Only stores first quadrant, other quadrants derived
- **Functions**: `SinAbs4096(x)`, `CosAbs4096(x)`

```c
s32 SinAbs4096(s32 x)
{
    switch (x & 0xc00) {
        case 0x000: return SINE_VALUE_TABLE[x & 0x3ff];           // Q1
        case 0x400: return SINE_VALUE_TABLE[0x3ff - (x & 0x3ff)]; // Q2
        case 0x800: return -SINE_VALUE_TABLE[x & 0x3ff];          // Q3
        case 0xc00: return -SINE_VALUE_TABLE[0x3ff - (x & 0x3ff)];// Q4
        default: return 0;
    }
}
```

### Pattern 0: Straight Line

```c
iVar13 = 0;
local_98 = 0;
```

No wave offset applied. Projectile travels in straight line.

### Pattern 1: Simple Vertical Wave

```c
if (param_4 == 1) {
    local_98 = SinAbs4096(iVar14);      // sin(phase)
    local_98 = iVar17 * local_98;        // amplitude * sin(phase)
    iVar13 = 0;                          // No X offset
}
```

- Single sine wave oscillation perpendicular to travel
- Used for: wavy beams, undulating projectiles

### Pattern 2: Spiral/Circular Motion

```c
else if (param_4 == 2) {
    iVar13 = SinAbs4096(iVar14);                    // sin(phase)
    iVar13 = (iVar17 >> 1) * iVar13 >> 8;           // (amplitude/2) * sin / 256
    local_98 = SinAbs4096(x);                        // sin(secondary_angle)
    local_98 = iVar13 * local_98;                    // Modulated Y
    iVar9 = CosAbs4096(x);                           // cos(secondary_angle)
    iVar13 = iVar13 * iVar9;                         // Modulated X
}
```

- Circular or helical motion
- Uses two angles: main phase and secondary angle
- Secondary angle: `(direction_angle + 0xC00) & 0x1FFF`

### Amplitude Calculation

```c
if (param_3 < 2) {
    iVar17 = 0x20;  // 32 - Large wave (short range)
} else {
    iVar17 = 8;     // 8 - Small wave (long range)
}
```

### Phase Progression

```c
iVar19 = 0;                              // Starting phase (20-bit fixed point)
uVar21 = _s32_div_f(0x80000, 0);         // Phase step calculation

// Each frame:
iVar14 = iVar19 >> 8;                    // Convert to 12-bit angle
iVar19 = iVar19 + (int)uVar21;           // Advance phase
```

- `0x80000` = 524,288 = half cycle (180°)
- Projectile completes half sine wave over travel

---

## Curved Projectile Throw (Parabolic Arc)

From `HandleCurvedProjectileThrow` in `dungeon_projectile_throw.c`:

### Arc Parameters

```c
// Distance calculation
absX = abs(start_x - target_x);
absY = abs(start_y - target_y);

// Total steps scales with distance
totalSteps = (absX + absY) * 12;

// Arc height scales with distance, clamped to 64
arcHeight = totalSteps + 12;
if (arcHeight >= 64) {
    arcHeight = 64;
}
```

### Phase and Position

```c
// Phase step for half sine wave
sinePhaseStep = 0x80000 / totalSteps;

// Position deltas (linear movement)
deltaXFixed = (target_x_pixels - start_x_pixels) / totalSteps;
deltaYFixed = (target_y_pixels - start_y_pixels) / totalSteps;

// Each frame
for (i = 0; i < totalSteps - 3; i++) {
    // Elevation follows sine curve
    sinVal = SinAbs4096(sinePhase >> 8) * arcHeight;
    projectile->elevation = sinVal;
    
    // Linear position advance
    posXFixed += deltaXFixed;
    posYFixed += deltaYFixed;
    sinePhase += sinePhaseStep;
}
```

### Result

- X/Y movement: Linear (constant velocity toward target)
- Z/elevation: Sine curve (parabolic arc)
- Peak height at midpoint of throw

---

## Attachment Point Calculation: FUN_0201cf90

Address: `0x0201cf90`

```c
void FUN_0201cf90(short *param_1, ushort *param_2, uint param_3)
{
    // Initialize output to zero
    *param_1 = 0;
    param_1[1] = 0;
    
    // Check animation control flag
    if ((*param_2 & 0x8000) == 0) {
        return;
    }
    
    // Validate attachment index (0-3 only)
    if (3 < param_3) {
        return;
    }
    
    // Get current frame index
    iVar3 = (int)(short)param_2[0x1d];  // frame_id
    if (iVar3 == -1) {
        return;
    }
    
    // Get WAN offset table pointer
    iVar2 = *(int *)(param_2 + 0x28);  // wan_offsets
    if (iVar2 == 0) {
        return;
    }
    
    // Calculate offset address
    // Each frame has 16 bytes (4 attachment points × 4 bytes)
    iVar2 = iVar2 + iVar3 * 0x10;
    iVar3 = param_3 * 4;
    psVar1 = (short *)(iVar2 + iVar3);
    
    // Special case: (99, 99) means center
    if (*psVar1 == 99 && *(psVar1 + 1) == 99) {
        *param_1 = 0x63;      // 99
        param_1[1] = 0x63;    // 99
        return;
    }
    
    // Normal case: add sprite offset to attachment point
    *param_1 = param_2[0x10] + *psVar1;           // offset_x + attachment_x
    param_1[1] = param_2[0x11] + *(psVar1 + 1);   // offset_y + attachment_y
}
```

### Attachment Point Indices

| Index | Name | Description |
|-------|------|-------------|
| 0 | Head | Top of sprite |
| 1 | LeftHand | Left appendage |
| 2 | RightHand | Right appendage |
| 3 | Center | Body center |

### WAN Offset Structure

```c
struct wan_offset {
    int16_t head_x, head_y;         // Index 0
    int16_t left_hand_x, left_hand_y;   // Index 1
    int16_t right_hand_x, right_hand_y; // Index 2
    int16_t center_x, center_y;     // Index 3
};
// Size: 16 bytes per frame
```

---

## Reverse Direction Handler: FUN_0232393c

Address: `0x0232393c`

Used for effects that travel in reverse direction (boomerang-style):

```c
// Add 4 to direction (rotate 180°)
iVar4 = (*(byte *)(iVar2 + 0x4c) + 4 & 7) * 4;

// Use reversed direction tables
iVar11 = (int)*(short *)(DAT_02323c30 + iVar4);  // Reversed X delta
sVar1 = *(short *)(DAT_02323c34 + iVar4);        // Reversed Y delta
```

---

## Function Reference Summary

### Effect Pipeline Functions

| Function | Address | Purpose |
|----------|---------|---------|
| `FUN_023250d4` | `0x023250d4` | Main move animation orchestrator |
| `FUN_022bfaa8` | `0x022bfaa8` | Layer 0 effect handler |
| `FUN_02325644` | `0x02325644` | Layer 1 effect handler |
| `FUN_022bfc5c` | `0x022bfc5c` | Layer 2 effect handler |
| `FUN_022be9e8` | `0x022be9e8` | Layer 3 (projectile) handler |
| `FUN_022be780` | `0x022be780` | Main effect dispatcher |
| `FUN_022be730` | `0x022be730` | Effect allocation wrapper |
| `FUN_022be44c` | `0x022be44c` | Effect context allocation |
| `FUN_022bdfc0` | `0x022bdfc0` | Animation initialization from effect_animation |

### Animation Control Functions

| Function | Address | Purpose |
|----------|---------|---------|
| `SetAnimationForAnimationControl` | - | Outer wrapper, handles sprite type branching |
| `SetAnimationForAnimationControlInternal` | `0x0201C5C4` | Sets animation parameters, critical group/sequence logic |
| `FUN_0201da20` | `0x0201da20` | WAN header lookup from sprite ID |
| `FUN_0201cf90` | `0x0201cf90` | Attachment point offset calculation |

### Projectile Functions

| Function | Address | Purpose |
|----------|---------|---------|
| `FUN_023230fc` | `0x023230fc` | Main projectile movement with wave patterns |
| `FUN_0232393c` | `0x0232393c` | Reverse direction projectile handler |
| `FUN_022beb2c` | - | Update effect position during flight |
| `GetMoveAnimationSpeed` | - | Read speed from move animation table |

### Trigonometry Functions

| Function | Address | Purpose |
|----------|---------|---------|
| `SinAbs4096` | - | Sine with 4096-step angles |
| `CosAbs4096` | - | Cosine with 4096-step angles |

---

## Data References

| Symbol | Address | Description |
|--------|---------|-------------|
| `DIRECTIONS_XY` | `0x0235171C` | 8-direction velocity vectors |
| `DAT_023238f8` | `0x023238f8` | Pointer to X velocity table |
| `DAT_023238fc` | `0x023238fc` | Pointer to Y velocity table |
| `DAT_02323900` | `0x02323900` | Additional direction data |
| `DAT_02323908` | `0x02323908` | Z/layer priority data |
| `DAT_022bfb64` | `0x022bfb64` | Layer 0 parameter template |
| `DAT_022bfd18` | `0x022bfd18` | Layer 2 parameter template |
| `DAT_022beb20` | `0x022beb20` | Layer 3 parameter template |

---

## Key Insights for Implementation

1. **Effects are NON-DIRECTIONAL**: Direction only affects projectile velocity, not sprite selection

2. **Animation selection formula**: `wan.animations[0].pnt[animation_index]` (always group 0)

3. **Speed mapping**: Raw speed 1→2, 2→3, other→6

4. **Frame count**: `24 / mapped_speed` (4, 8, or 12 frames)

5. **Wave patterns**: 0=straight, 1=vertical sine, 2=spiral

6. **Attachment points**: Per-frame offsets stored in WAN file, indexed 0-3

7. **Direction check**: Only passes for NULL wan_header (types 5-6), fails for valid pointers (types 1-4)

---

## Credits

- Reverse engineering: Ghidra decompilation analysis
- pmdsky-debug: Symbol names, structure definitions, source code
- Project Pokemon: WAN format documentation
