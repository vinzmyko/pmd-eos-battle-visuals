# Effect Context Structure

## Summary

- Effect contexts manage individual active effects (visual animations, sounds, etc.)
- Pool of 32 slots, each 316 bytes (0x13C)
- Contains animation state, positioning, resources, and lifecycle flags
- Embedded animation_control at offset 0x68 for WAN sprite playback
- Screen effects (types 5/6) use different control structure at offset 0xE8
- Projectile effects store trajectory data at offsets 0x128-0x134

## Effect Pool

Effects are stored in a contiguous pool with 32 slots.

| Property | Value |
|----------|-------|
| Pool Size | 32 effects |
| Entry Size | 316 bytes (0x13C) |
| Total Size | 10,112 bytes (0x2780) |
| Base Address | `*DAT_022be72c` (NA) |

**Evidence:** `FUN_022bf764` iterates pool
```c
piVar3 = (int *)*DAT_022bf7cc;
iVar2 = 0;
do {
    uVar1 = FUN_022bf4f0(piVar3, param_1);
    iVar2 = iVar2 + 1;
    piVar3 = piVar3 + 0x4f;  // 0x4f * 4 = 0x13C bytes
} while (iVar2 < 0x20);  // 32 iterations
```

## Global Effect State

The global state area sits immediately after the 32-slot pool (32 × 0x13C = 0x2780 bytes).

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x2780 | int32 | instance_counter | Monotonically increasing, assigned to new effects |
| +0x2784 | int32 | shared_wan_state | 0 or 1; controls which shared WAN file is active |
| +0x2788 | int16 | shared_sprite_id | WAN table entry for pre-loaded file 0 or file 1 |
| +0x278C | int32 | unknown_278c | Written at init, used for param_1 + 0x54 in types 1-3 |
| +0x2790 | int16 | palette_variant | Palette param, only written in state 1 path |
| +0x2794 | int32 | unknown_2794 | Used for type 3 sprite setup |
| +0x279C | int16 | unknown_279c | Read in types 1 and 4 |
| +0x279E | uint8 | screen_param_0 | Type 5 screen_effect_param (param_2 == 0) |
| +0x279F | uint8 | screen_param_1 | Type 5 screen_effect_param (param_2 == 1) |
| +0x27A0 | uint8 | memory_pressure | Set to 1 when effect memory allocation fails |

**Base address:** `*DAT_022be72c` = `0x022DC1C0` (heap-allocated)

**Evidence:** `FUN_022be44c` increments instance_counter
```c
iVar6 = *(int *)(*piVar1 + 0x2780);
*(int *)(*piVar1 + 0x2780) = iVar6 + 1;
ptr[3] = iVar6;  // Assign to new effect's instance_id
```

> See `Data Structures/effect_animation_info.md` section "WAN File 0/1 Shared Resource System" for how +0x2788 is initialised

## Structure Definition
```c
struct effect_context {
    /* 0x00 */ int32_t param_from_caller;
    /* 0x04 */ int32_t caller_context;
    /* 0x08 */ int32_t anim_type;          // 1-6 (0=invalid)
    /* 0x0C */ int32_t instance_id;        // -1 = inactive/free slot
    /* 0x10 */ int32_t wan_ptr;
    /* 0x14 */ int32_t effect_id;          // Index into effect_animation_info
    /* 0x18 */ int32_t delay_counter;      // Countdown before tick processes
    /* 0x1C */ int32_t direction;          // 0-7 or -1
    /* 0x20 */ int16_t current_x;          // Base position (entity pixel_x >> 8, or projectile pos)
    /* 0x22 */ int16_t current_y;          // Base position (entity pixel_y >> 8, or projectile pos)
    /* 0x24 */ int16_t offset_x;           // Dual-purpose: attachment offset OR projectile arc offset
    /* 0x26 */ int16_t offset_y;           // Dual-purpose: attachment offset OR projectile arc offset
    /* 0x28 */ int8_t  attachment_point;   // Copied from effect_animation_info, read by binding system
    /* 0x2C */ int32_t z_priority;
    /* ... */
    /* 0x3C */ uint32_t flags;             // Bit 0 = runtime loop flag
    /* 0x40 */ int32_t stored_anim_type;
    /* 0x44 */ int32_t file_index;
    /* 0x48 */ int32_t palette_index;
    /* 0x4C */ int32_t unknown_4c;
    /* 0x50 */ int32_t animation_index;
    /* 0x54 */ int32_t unknown_54;
    /* 0x58 */ int32_t sfx_id;
    /* 0x5C */ int32_t timing_value;
    /* 0x60 */ uint8_t is_non_blocking;    // Controls AnimationHasMoreFrames return
    /* 0x61 */ uint8_t loop_flag;          // From effect_animation_info
    /* 0x64 */ int16_t wan_table_entry;    // For cleanup
    /* 0x68 */ animation_control anim_ctrl; // WAN animation state (~0x7C bytes)
    /* ... */
    /* 0xE4 */ int16_t screen_effect_handle; // For type 5/6
    /* 0xE8 */ uint8_t screen_effect_ctrl[0x1C]; // Screen effect control structure
    /* ... */
    /* 0x128 */ int16_t source_x;          // Projectile start X (screen pixels)
    /* 0x12A */ int16_t source_y;          // Projectile start Y (screen pixels)
    /* 0x12C */ int16_t dest_x;            // Projectile end X (screen pixels)
    /* 0x12E */ int16_t dest_y;            // Projectile end Y (screen pixels)
    /* 0x130 */ uint16_t entity_id;        // Attacker entity reference
    /* 0x132 */ int16_t stored_velocity_x; // Copied from offset 0x24
    /* 0x134 */ int16_t stored_velocity_y; // Copied from offset 0x26
    /* 0x136 */ int16_t position_offset;   // Added to X each frame
    /* 0x13A */ uint8_t unknown_13a;
};
// Size: 0x13C (316 bytes)
```

## Key Fields

### instance_id (offset 0x0C)

Unique identifier for this effect instance. Used to find effects in the pool.

| Value | Meaning |
|-------|---------|
| -1 | Slot is free/inactive |
| 0+ | Active effect with unique ID |

**Evidence:** Pool allocation
```c
do {
    if (*(int *)(iVar10 + 0xc) == -1) {  // Found free slot
        // ... initialize effect ...
        break;
    }
    iVar6 = iVar6 + 1;
    iVar10 = iVar10 + 0x13c;
} while (iVar6 < 0x20);
```

### delay_counter (offset 0x18)

Frames to wait before processing effect. Decrements each frame until 0.

**Evidence:** `FUN_022bf4f0`
```c
if (param_1[6] < 1) {  // delay_counter at offset 0x18
    // Process effect
}
// ...
if (0 < param_1[6]) {
    param_1[6] = param_1[6] + -1;  // Decrement
}
```

### is_non_blocking (offset 0x60)

Controls whether `AnimationHasMoreFrames` returns true (blocking) or false (non-blocking).

| Value | Behavior |
|-------|----------|
| 0x00 | Blocking - wait loops continue |
| Non-zero | Non-blocking - wait loops exit immediately |

**Evidence:** `AnimationHasMoreFrames`
```c
return *(char *)(iVar2 + 0x60) == '\0';  // Return TRUE if is_non_blocking == 0
```

### loop_flag (offset 0x61)

Copied from `effect_animation_info->loop_flag` during initialization.

**Evidence:** `FUN_022bdfc0`
```c
*(uint8_t *)(param_1 + 0x61) = peVar3->unk_repeat;
```

### flags (offset 0x3C)

Runtime flags. Bit 0 controls loop behavior.

**Evidence:** `FUN_022bf4f0`
```c
if ((param_1[0xf] & 1U) == 0) {  // offset 0x3C, bit 0
    FUN_022bdcbc(...);  // Cleanup non-looping effect
}
```

### wan_table_entry (offset 0x64)

Reference to loaded sprite in WAN table. Used for cleanup.

**Evidence:** `FUN_022bdec4`
```c
if (*(short *)(param_1 + 100) == 0) return;  // offset 0x64
if (param_2 != 0) {
    DeleteWanTableEntryVeneer(..., *(short *)(param_1 + 100));
}
```

## Projectile Trajectory Fields (0x128-0x134)

These fields are populated by `FUN_022be9e8` (Layer 3 projectile handler) and used by `FUN_023230fc` for projectile motion.

### source_x / source_y (offsets 0x128, 0x12A)

Projectile starting position in screen pixels. Set from attacker's pixel position.

**Evidence:** `FUN_022be9e8`
```c
*(ushort *)(iVar10 + 0x128) = param_1[2];  // source_x from attacker
*(ushort *)(iVar10 + 0x12a) = param_1[3];  // source_y from attacker
```

**Source calculation:** `attacker->pixel_pos.x >> 8` (8.8 fixed point to screen pixels)

### dest_x / dest_y (offsets 0x12C, 0x12E)

Projectile destination position in screen pixels. Set from target tile center.

**Evidence:** `FUN_022be9e8`
```c
*(undefined2 *)(iVar10 + 300) = *param_2;    // dest_x (0x12C)
*(undefined2 *)(iVar10 + 0x12e) = param_2[1]; // dest_y
```

**Destination calculation:** `(tile * 24 + offset) * 256 >> 8` where offset is +12 for X, +16 for Y

### entity_id (offset 0x130)

Reference to the attacker entity. Used for tracking which entity spawned the projectile.

**Evidence:** `FUN_022be9e8`
```c
*(ushort *)(iVar10 + 0x130) = param_1[1];  // entity_id
```

### stored_velocity_x / stored_velocity_y (offsets 0x132, 0x134)

Copied from velocity fields at offsets 0x24/0x26 during projectile setup.

**Evidence:** `FUN_022be9e8`
```c
*(undefined2 *)(iVar10 + 0x132) = *(undefined2 *)(iVar10 + 0x24);
*(undefined2 *)(iVar10 + 0x134) = *(undefined2 *)(iVar10 + 0x26);
```

## Embedded Structures

### animation_control (offset 0x68)

Standard animation control structure for WAN sprite playback. Size ~0x7C bytes.

> See `Data Structures/animation_control.md` for complete structure definition.

### screen_effect_ctrl (offset 0xE8)

Special control structure for type 5/6 screen effects. Size 0x1C bytes (28 bytes).

Used by `FUN_022bdf34` for screen effect updates.

## Pool Management

### Finding Free Slot

**Evidence:** `FUN_022be44c`
```c
iVar6 = 0;
iVar10 = *DAT_022be72c;
do {
    if (*(int *)(iVar10 + 0xc) == -1) {  // instance_id == -1
        // Initialize this slot
        break;
    }
    iVar6 = iVar6 + 1;
    iVar10 = iVar10 + 0x13c;
} while (iVar6 < 0x20);  // Max 32 slots

if (0x1f < (int)uVar7) {
    return -1;  // Pool full
}
```

### Finding Effect by instance_id

**Evidence:** `FUN_022be9a0`
```c
int FUN_022be9a0(int param_1)  // param_1 = instance_id
{
    iVar1 = 0;
    iVar2 = *DAT_022bea00;
    while (true) {
        if (0x1f < iVar1) {
            return -1;  // Not found
        }
        if (*(int *)(iVar2 + 0xc) == param_1) break;  // Found
        iVar1 = iVar1 + 1;
        iVar2 = iVar2 + 0x13c;
    }
    return iVar1;  // Return slot index
}
```

## Initialization

**Evidence:** `FUN_022bdfc0`
```c
void FUN_022bdfc0(int param_1, ...)
{
    // Copy fields from effect_animation_info
    *(int *)(param_1 + 0x40) = peVar3->anim_type;        // stored_anim_type
    *(int *)(param_1 + 0x44) = peVar3->file_index;
    *(int *)(param_1 + 0x48) = peVar3->palette_index;
    *(int *)(param_1 + 0x50) = peVar2->animation_index;
    *(int *)(param_1 + 0x58) = peVar3->se_id;
    *(uint8_t *)(param_1 + 0x60) = peVar3->is_non_blocking;
    *(uint8_t *)(param_1 + 0x61) = peVar3->unk_repeat;
    
    // Initialize animation control
    SetAnimationForAnimationControl(
        (animation_control *)(param_1 + 0x68),
        animation_index,
        DIR_DOWN,
        ...
    );
}
```

## Cross-References

> See `Systems/effect_lifecycle.md` for lifecycle management (allocation, ticking, termination)

> See `Data Structures/animation_control.md` for animation_control bitfield details

> See `Systems/animation_timing.md` for AnimationHasMoreFrames and wait loop patterns

> See `Systems/projectile_motion.md` for how trajectory fields are used during flight

## Open Questions

- Purpose of effect_context + 0x136 (position override flag) and + 0x138 (override position source)
- Complete layout between offsets 0x2C-0x3C
- Complete layout between offsets 0xAC-0xE4
- Screen effect control structure details (offset 0xE8)
- Purpose of flags bits 1-31 (only bit 0 is known)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_022be44c` | `0x022be44c` | Effect allocation (find free slot) |
| `FUN_022bdfc0` | `0x022bdfc0` | Effect initialization |
| `FUN_022be9a0` | `0x022be9a0` | Find effect by instance_id |
| `FUN_022be9e8` | `0x022be9e8` | Layer 3 projectile setup (sets trajectory fields) |
| `FUN_022bf764` | `0x022bf764` | Effect pool tick (iterates 32 slots) |
| `FUN_022bf4f0` | `0x022bf4f0` | Per-effect tick function |
| `AnimationHasMoreFrames` | - | Check if effect still active (for wait loops) |
