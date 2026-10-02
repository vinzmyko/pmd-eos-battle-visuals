# Effect Context Structure

## Summary

- Effect contexts manage individual active effects (visual animations, sounds, etc.)
- Pool of 32 slots, each 316 bytes (0x13C)
- Contains animation state, positioning, resources, and lifecycle flags
- Embedded animation_control at offset 0x68 for WAN sprite playback
- Screen effects (types 5/6) use different control structure at offset 0xE8
- 0x14-0x3F is a spawn block copied verbatim from the dispatch struct (effect_id, delay, direction, position, offset, attachment index, draw order, flags)
- Projectile effects store trajectory data at 0x128-0x134; 0x136/0x138 are per-tick velocities

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
    /* 0x00 */ int32_t param_from_caller;   // param_3 of FUN_022be780; passed to FUN_022bdfc0 as param_2
    /* 0x04 */ int32_t dispatch_type;       // param_1 of FUN_022be780 (1 secondary, 2 projectile, 5 charge, 6 primary, 7 generic)
    /* 0x08 */ int32_t anim_type;           // 1-6 (0=invalid)
    /* 0x0C */ int32_t instance_id;         // -1 = inactive/free slot
    /* 0x10 */ int32_t sequence_count;      // FUN_0201da20(wan_table_entry); % 8 == 0 → directional
    // --- 0x14-0x3F: spawn block, 11 words copied verbatim from the dispatch struct ---
    /* 0x14 */ int32_t effect_id;           // word [0]
    /* 0x18 */ int32_t delay_counter;       // word [1]
    /* 0x1C */ int32_t direction;           // word [2]; 0-7 or -1
    /* 0x20 */ int16_t current_x;           // base position, world px (pixel_pos >> 8)
    /* 0x22 */ int16_t current_y;
    /* 0x24 */ int16_t offset_x;            // bound: attachment offset. projectile: launch offset, decays per frame
    /* 0x26 */ int16_t offset_y;
    /* 0x28 */ int8_t  attachment_point;    // -1 → tick ignores 0x24/0x26
    /* 0x2C */ int32_t draw_order;          // 0xFFFF = unbound sentinel
    /* 0x30 */ uint8_t spawn_tail[0x0C];    // template data, passed to FUN_0201d110 each drawn tick
    /* 0x3C */ uint32_t flags;              // word [10]; bit 0 = runtime loop flag
    // --- end spawn block ---
    /* 0x40 */ int32_t stored_anim_type;
    /* 0x44 */ int32_t file_index;
    /* 0x48 */ int32_t palette_index;
    /* 0x4C */ int32_t unknown_4c;
    /* 0x50 */ int32_t animation_index;     // + direction when directional
    /* 0x54 */ int32_t unknown_54;
    /* 0x58 */ int32_t sfx_id;
    /* 0x5C */ int32_t timing_value;        // delay_counter + effect_animation.field_0x14
    /* 0x60 */ uint8_t is_non_blocking;
    /* 0x61 */ uint8_t loop_flag;           // from effect_animation_info
    /* 0x64 */ int16_t wan_table_entry;
    /* 0x68 */ animation_control anim_ctrl; // 0x84/0x86 = screen x/y, 0xA0 = draw order (anim_ctrl + 0x38)
    /* ... */
    /* 0xE4 */ int16_t screen_effect_handle; // type 5/6
    /* 0xE8 */ uint8_t screen_effect_ctrl[0x1C];
    /* ... */
    /* 0x128 */ int16_t source_x;           // attacker pixel_pos.x >> 8
    /* 0x12A */ int16_t source_y;
    /* 0x12C */ int16_t dest_x;             // end tile * 24 + 12
    /* 0x12E */ int16_t dest_y;             // end tile * 24 + 16
    /* 0x130 */ int16_t species_id;         // attacker monster id (not an entity reference)
    /* 0x132 */ int16_t launch_offset_x;    // copy of 0x24 at spawn, before any decay
    /* 0x134 */ int16_t launch_offset_y;
    /* 0x136 */ int16_t velocity_x;         // added to 0x20 after each drawn tick
    /* 0x138 */ int16_t velocity_y;         // added to 0x22 after each drawn tick
    /* 0x13A */ uint8_t unknown_13a;        // cleared by FUN_022bdfc0
};
// Size: 0x13C (316 bytes)
```

### Spawn Block

`FUN_022be780` → `FUN_022be730` → `FUN_022be44c` (allocate) then `FUN_022bdfc0` (init). The allocator copies 11 words of the dispatch struct into `ctx + 0x14`; init never touches 0x14-0x3F.

**Evidence:** `FUN_022be44c`
```c
ptr[2] = peVar3->field_0x0;   // anim_type
ptr[1] = param_1;             // dispatch_type
piVar1 = ptr + 5;             // ctx + 0x14
// 2 × 4 words + 3 words = 11 words copied from param_2
```

**Evidence:** `FUN_022be730`
```c
iVar1 = FUN_022be44c(param_1, param_2, param_3);
iVar2 = FUN_022be9a0((int)(short)iVar1);
FUN_022bdfc0(base + iVar2 * 0x13c, *(int *)(base + iVar2 * 0x13c));  // param_2 = ctx[0x00]
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

Frames to wait before processing effect. Decrements each frame until 0. Set from word [1] of the dispatch struct; projectiles pass 0.

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

### draw_order (offset 0x2C)

Read by `FUN_022bf4f0` on every drawn tick and written to `anim_ctrl + 0x38` (`ctx + 0xA0`), where it becomes the base of the OAM bucket index. The earlier claim that this field is never read was wrong.

| Value | Source | Used as |
|-------|--------|---------|
| `0xFFFF` (`DAT_022bf75c`) | template, unbound effects | tick computes `bias + (current_y − cam_y) / 2`; bias = −3 if directional and direction ∈ {3,4,5}, else +3 |
| anything else | `FUN_022bfb6c` (bound), `FUN_022beb2c` (projectile) | used as-is |

### Render Tick (`FUN_022bf4f0`)

```c
off = (ctx[0x28] != -1) ? *(s16x2 *)(ctx + 0x24) : *(s16x2 *)0x022C787C;  // default (0, 0)
if (off.x == 99 || off.y == 99) skip;           // not drawn, velocity not applied
sx = off.x + (ctx[0x20] - cam_x);
sy = off.y + (ctx[0x22] - cam_y);
z  = (ctx[0x2C] == 0xFFFF) ? bias + (ctx[0x22] - cam_y) / 2 : ctx[0x2C];
ctx[0x20] += ctx[0x136];  ctx[0x22] += ctx[0x138];
if (-0x40 < sx < 0x13F && -0x40 < sy < 0x100) {
    anim_ctrl.screen = (sx, sy);  anim_ctrl[0x38] = z;  draw;
}
```

A 99 component means the frame is not drawn. This differs from `PlayEffectAnimationEntity`, which falls back to the entity position.

### velocity_x / velocity_y (offsets 0x136, 0x138)

Per-tick position deltas, added after each drawn tick. Razor Leaf sets `velocity_x = 6`. `FUN_022bfb6c` skips its position write while either is nonzero, so an effect with its own velocity is not dragged back to its bound entity.

## Projectile Trajectory Fields (0x128-0x134)

Written by `FUN_022be9e8` immediately after spawn. Only `source`/`dest` are read: by `FUN_022beb2c`, for the decay divisor. See `Systems/projectile_motion.md`.

**Evidence:** `FUN_022be9e8`
```c
*(ushort *)(ctx + 0x128) = param_1[2];   // attacker pixel_pos.x >> 8
*(ushort *)(ctx + 0x12a) = param_1[3];   // attacker pixel_pos.y >> 8
*(short  *)(ctx + 0x12c) = param_2[0];   // end tile * 24 + 12
*(short  *)(ctx + 0x12e) = param_2[1];   // end tile * 24 + 16
*(ushort *)(ctx + 0x130) = param_1[1];   // species id
*(short  *)(ctx + 0x132) = *(short *)(ctx + 0x24);   // launch offset snapshot
*(short  *)(ctx + 0x134) = *(short *)(ctx + 0x26);
```

`0x132/0x134` were previously named `stored_velocity`. They are a snapshot of the launch offset; no reader has been found.

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

- Layout of the spawn tail (0x30-0x3B) and what `FUN_0201d110` does with it
- Complete layout between offsets 0xAC-0xE4
- Screen effect control structure details (offset 0xE8)
- Purpose of flags bits 1-31 (only bit 0 is known)
- Whether anything reads 0x132/0x134

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
| `FUN_022be780` | `0x022be780` | Effect dispatcher (writes dispatch type to 0x04) |
| `FUN_022be730` | `0x022be730` | Allocate + init wrapper |
| `FUN_022beb2c` | `0x022beb2c` | Projectile per-frame position + offset decay |
