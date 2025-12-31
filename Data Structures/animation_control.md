# Animation Control Structure

## Summary

- Animation control manages playback state for a single sprite instance
- Contains current frame, animation index, timing, and WAN reference
- Embedded in effect contexts at offset 0x68 (size 0x7C bytes)
- Used by `SetAnimationForAnimationControl` to configure animation playback

## Structure Definition

```c
struct animation_control {
    uint16_t flags;              // 0x00: Control flags (bit 15 = active)
    uint16_t unk_0x02;           // 0x02: Unknown
    // ...
    int16_t sprite_offset_x;     // 0x20: X offset for rendering
    int16_t sprite_offset_y;     // 0x22: Y offset for rendering
    // ...
    int16_t frame_id;            // 0x3A: Current frame index (-1 = none)
    // ...
    int32_t offset_table_ptr;    // 0x50: Pointer to attachment point offset table
    // ...
    int16_t loaded_sprite_id;    // 0x64: Sprite ID in WAN table
    // ...
};
// Size: 0x7C (124 bytes)
```

## Key Fields

### flags (offset 0x00)

| Bit | Purpose |
|-----|---------|
| 15 | Animation active/valid flag |
| 0-14 | Various state flags |

**Evidence:** `FUN_0201cf90`
```c
// Check animation control flag
if ((*anim_ctrl & 0x8000) == 0) return;  // Bit 15 must be set
```

### frame_id (offset 0x3A)

Current frame index within the animation sequence.

| Value | Meaning |
|-------|---------|
| -1 | No frame / invalid |
| 0+ | Frame index into meta-frame table |

**Evidence:** `FUN_0201cf90`
```c
int frame_id = (int)(short)anim_ctrl[0x1d];  // offset 0x3A (0x1d * 2)
if (frame_id == -1) return;
```

### sprite_offset_x/y (offsets 0x20, 0x22)

Base sprite offsets added to attachment point calculations.

**Evidence:** `FUN_0201cf90`
```c
// Add sprite offset to attachment point
*output = anim_ctrl[0x10] + *point;       // 0x10 * 2 = 0x20 (sprite_offset_x)
output[1] = anim_ctrl[0x11] + *(point + 1); // 0x11 * 2 = 0x22 (sprite_offset_y)
```

### offset_table_ptr (offset 0x50)

Pointer to per-frame attachment point offset table.

**Evidence:** `FUN_0201cf90`
```c
int offset_table = *(int *)(anim_ctrl + 0x28);  // 0x28 * 2 = 0x50
if (offset_table == 0) return;

// Each frame has 16 bytes (4 points × 4 bytes)
short *point = (short *)(offset_table + frame_id * 0x10 + attachment_index * 4);
```

### loaded_sprite_id (offset 0x64)

Reference to sprite entry in WAN table.

**Evidence:** Effect initialization
```c
*(short *)(param_1 + 0x64) = sprite_id;  // Store sprite ID
```

## Embedding in Effect Context

Animation control is embedded in effect context structures at offset 0x68.

```c
struct effect_context {
    // ... other fields ...
    /* 0x64 */ uint16_t sprite_id;
    /* 0x68 */ animation_control anim_ctrl;  // 0x7C bytes
    // ... continues to 0x13C total ...
};
```

**Evidence:** `FUN_022bdfc0`
```c
SetAnimationForAnimationControl(
    (animation_control *)(param_1 + 0x68),  // anim_ctrl at offset 0x68
    animation_index,
    DIR_DOWN,
    (int)(short)*(undefined4 *)(param_1 + 0x54),
    *(uint *)(param_1 + 0x48) & 0xff,  // palette_index
    0,
    loop_flag,
    0
);
```

## SetAnimationForAnimationControl Parameters

```c
void SetAnimationForAnimationControl(
    animation_control* anim_ctrl,  // Target animation control
    int animation_key,             // Animation index or group ID
    direction_id direction,        // Direction (0-7), ignored for PROPS_UI
    int param_4,                   // Unknown
    uint low_palette_pos,          // Palette slot (0-14)
    int param_6,                   // Unknown
    uint8_t loop_flag,             // 0=play once, non-zero=loop
    int param_8                    // Unknown
);
```

**Behavior by sprite type:**
- CHARA: `animation_key` = group ID, `direction` = sequence within group
- PROPS_UI: `animation_key` = flat sequence index, `direction` ignored

## Animation State Machine

### Initialization

1. Effect context allocated
2. Sprite loaded into WAN table (if not already)
3. `sprite_id` stored at context + 0x64
4. `SetAnimationForAnimationControl` called to start animation

### Per-Frame Update

1. `AnimationHasMoreFrames` checks if animation still playing
2. Frame counter advances
3. Meta-frame renderer reads `frame_id` for current frame data

### Completion

1. `AnimationHasMoreFrames` returns false
2. If `loop_flag` set, animation restarts
3. Otherwise, animation stops

## AnimationHasMoreFrames

Checks if animation has remaining frames to play.

**Usage pattern:**
```c
while (AnimationHasMoreFrames((int)(short)effect_handle)) {
    AdvanceFrame('(');
}
```

Returns:
- Non-zero: Animation still playing
- Zero: Animation complete

## Open Questions

- Complete animation_control structure (many fields unknown)
- How timing/speed is controlled within animation_control
- Relationship between frame_id and actual frame timing
- What params 4, 6, 8 of SetAnimationForAnimationControl do

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `SetAnimationForAnimationControl` | `0x0201C4F4` | Configure animation playback |
| `SetAnimationForAnimationControlInternal` | `0x0201C5C4` | Inner animation setup |
| `AnimationHasMoreFrames` | - | Check if animation still playing |
| `FUN_0201cf90` | `0x0201CF90` | Calculate attachment point offset |
