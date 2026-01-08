# Effect Context Structure

## Summary

- Effect contexts manage individual active effects (visual animations, sounds, etc.)
- Pool of 32 slots, each 316 bytes (0x13C)
- Contains animation state, positioning, resources, and lifecycle flags
- Embedded animation_control at offset 0x68 for WAN sprite playback
- Screen effects (types 5/6) use different control structure at offset 0xE8

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
    /* 0x20 */ int16_t current_x;
    /* 0x22 */ int16_t current_y;
    /* 0x24 */ int16_t velocity_x;
    /* 0x26 */ int16_t velocity_y;
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
    /* 0x128 */ int16_t source_x;
    /* 0x12A */ int16_t source_y;
    /* 0x12C */ int16_t dest_x;
    /* 0x12E */ int16_t dest_y;
    /* 0x130 */ uint16_t entity_id;
    /* 0x132 */ int16_t stored_velocity_x;
    /* 0x134 */ int16_t stored_velocity_y;
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

## Open Questions

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
| `FUN_022bf764` | `0x022bf764` | Effect pool tick (iterates 32 slots) |
| `FUN_022bf4f0` | `0x022bf4f0` | Per-effect tick function |
| `AnimationHasMoreFrames` | - | Check if effect still active (for wait loops) |
