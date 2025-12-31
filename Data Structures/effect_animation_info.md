# Effect Animation Info Structure

## Summary

- 700 entries, 28 bytes each
- Defines visual effects used by moves, items, traps
- Specifies WAN file, animation index, sound, and rendering parameters
- Animation type (0-6) determines resource allocation and rendering path

## Memory Locations

| Region | Address | Symbol Name |
|--------|---------|-------------|
| NA | `0x022CC52C` | `EFFECT_ANIMATION_INFO` |
| EU | `0x022CCE84` | `EFFECT_ANIMATION_INFO` |
| JP | `0x022CDC14` | `EFFECT_ANIMATION_INFO` |

## Table Properties

| Property | Value |
|----------|-------|
| Entry Count | 700 |
| Entry Size | 28 bytes (0x1C) |
| Total Size | 19,600 bytes (0x4CB0) |

## Structure Definition
```c
struct effect_animation {
    int32_t anim_type;           // 0x00: Animation type/rendering category (0-6)
    int32_t file_index;          // 0x04: Index into effect.bin pack file
    int32_t palette_index;       // 0x08: Palette slot for rendering (0-14)
    int32_t animation_index;     // 0x0C: Sequence index within WAN file (NOT group index)
    int32_t sfx_id;              // 0x10: Sound effect ID (-1 = none)
    int32_t timing_offset;       // 0x14: Added to timing calculations
    uint8_t screen_effect_param; // 0x18: Parameter for type 5 screen effects
    int8_t  attachment_point;    // 0x19: Position offset lookup (-1 to 3) - SIGNED
    uint8_t is_non_blocking;     // 0x1A: If non-zero, animation plays asynchronously
    uint8_t loop_flag;           // 0x1B: If non-zero, animation loops
};
// Size: 0x1C (28 bytes)
```

## Field Documentation

### anim_type (offset 0x00)

Determines rendering path, resource allocation, and system state requirements.

| Value | Name | Description | Resource Handling |
|-------|------|-------------|-------------------|
| 0 | Invalid | No effect | Returns -1 |
| 1 | WanFile0 | Standard WAN | Shared resource, requires state == 0 |
| 2 | WanFile1 | Standard WAN | Shared resource, requires state == 1 |
| 3 | WanOther | Standard WAN | Clears conflicting type-3 with different file_index |
| 4 | Wat | WAT format | Clears conflicting type-4 effects |
| 5 | Screen | Screen effect | Uses file_index + 0x10C, special allocation |
| 6 | Wba | WBA format | Special allocation |

**Evidence:** `FUN_022be44c`
```c
if ((iVar6 == 2) && (*(int *)(*DAT_022be72c + 0x2784) != 1)) {
    return -1;  // Type 2 requires state == 1
}
if ((iVar6 == 1) && (*(int *)(*DAT_022be72c + 0x2784) != 0)) {
    return -1;  // Type 1 requires state == 0
}
```

**Evidence:** Type 3 conflict resolution in `FUN_022be44c`
```c
if (iVar6 == 3) {
    for (i = 0; i < 0x20; i++) {
        if ((entry->id != -1) && (entry->type == 3) &&
            (entry->effect_id != current_effect_id) &&
            (GetEffectAnimation(entry->effect_id)->file_index != peVar3->file_index)) {
            FUN_022bdec4(entry, 1);  // Clear conflicting effect
        }
    }
}
```

### file_index (offset 0x04)

Index into effect.bin pack archive. For type 5 effects, actual index is `file_index + 0x10C` (268).

**Evidence:** `FUN_022c01fc`
```c
if (param_2 == 5) {
    // file_index + 0x10C is used for actual file lookup
    FUN_022c067c(5, file_index, param_4);
}
```

**Pack File Info:**
- Pack Archive ID: 3 (`PACK_ARCHIVE_EFFECT`)
- File Path: `/EFFECT/effect.bin`
- Format: AT4PX compressed pack archive

### palette_index (offset 0x08)

Palette slot for sprite rendering. Range: 0-14.

**Evidence:** `FUN_022bdfc0`
```c
*(int *)(param_1 + 0x48) = peVar3->field_0x8;
// Later passed to SetAnimationForAnimationControl as 5th parameter (low_palette_pos)
SetAnimationForAnimationControl(
    (animation_control *)(param_1 + 0x68),
    animation_index,
    DIR_DOWN,
    (int)(short)*(undefined4 *)(param_1 + 0x54),
    *(uint *)(param_1 + 0x48) & 0xff,  // palette_index masked to byte
    0,
    loop_flag,
    0
);
```

### animation_index (offset 0x0C)

**CRITICAL:** For effect sprites (PROPS_UI / type 0), this is a FLAT SEQUENCE INDEX into animation group 0, NOT a group index.

The ROM ignores the animation_group_id parameter for effect sprites:

**Evidence:** `SetAnimationForAnimationControlInternal`
```c
if (*(char *)&wan_header->sprite_type == '\x01') {
    // CHARA sprites (type 1): uses group and sequence
    pwVar3 = pwVar8->animations[animation_group_id].pnt[animation_id];
}
else {
    // PROPS_UI sprites (type 0): IGNORES group, uses sequence from group 0
    pwVar3 = pwVar8->animations->pnt[animation_id];  // animations[0].pnt[animation_id]
}
```

**For directional effects:** Direction (0-7) is NOT added to animation_index for effect sprites. Direction only affects projectile velocity.

## Animation Sequence Resolution

> See `Systems/animation_selection.md` for sprite type routing.

**CRITICAL:** The `animation_index` field may have direction added to it at runtime based on sprite sequence count.

### Directional Effect Logic

The ROM determines whether an effect uses 8 pre-rotated animation sequences or a single sequence based on the sprite's sequence count.

**Rule:** Direction (0-7) is added to `animation_index` if and only if `sequence_count % 8 == 0`.

**Evidence:** `FUN_022bdfc0`
```c
// 1. Get base animation_index from effect_animation_info
*(int *)(param_1 + 0x50) = peVar2->animation_index;

// 2. Get sprite sequence count
if (anim_type != 5 && anim_type != 6) {  // Skip for screen effects
    uVar3 = FUN_0201da20(*(short *)(param_1 + 0x64));  // sprite_id
    *(undefined4 *)(param_1 + 0x10) = uVar3;  // Store sequence_count
}

// 3. Conditionally add direction to animation_index
iVar6 = *(int *)(param_1 + 0x1c);  // direction (0-7, or -1 if none)
if ((iVar6 != -1) && (sequence_count % 8 == 0)) {
    *(int *)(param_1 + 0x50) += iVar6;  // animation_index += direction
}
```

#### WAN_TABLE Entry Structure

```c
struct wan_table_entry {
    char path[32];                   // 0x00: Null-terminated path
    bool8 file_externally_allocated; // 0x20
    uint8_t source_type;             // 0x21: 1=direct file, 2=pack file
    int16_t pack_id;                 // 0x22
    int16_t file_index;              // 0x24
    uint8_t field5_0x26;
    uint8_t field6_0x27;
    uint32_t iov_len;                // 0x28
    int16_t reference_counter;       // 0x2C
    uint8_t field9_0x2e;
    uint8_t field10_0x2f;
    wan_header* sprite_start;        // 0x30: Pointer to WAN header
    void* iov_base;                  // 0x34: Points to SIR0 container
};
// Total size: 0x38 (56 bytes)
```

### Sequence Count Extraction from WAN Header

**Evidence:** `LAB_0201da00`
```c
short LAB_0201da00(wan_header* header) {
    uint8_t sprite_type = *(uint8_t*)(header + 0x8);
    void* ptr_anim_info = *(void**)(header + 0x0);
    
    if (sprite_type == 0 || sprite_type == 2) {
        // Effect sprites (PROPS_UI): return sequence count in group 0
        void* anim_group_table = *(void**)(ptr_anim_info + 0x8);
        return *(int16_t*)(anim_group_table + 0x4);  // anim_length
    } else {
        // Character sprites (CHARA): return number of animation groups
        return *(int16_t*)(ptr_anim_info + 0xC);  // nb_anim_groups
    }
}
```

### The % 8 Test

The condition `sequence_count % 8 == 0` is implemented via bit manipulation:

**Evidence:** Original decompiled code
```c
if ((iVar6 != -1) &&
   (uVar5 = *(int *)(param_1 + 0x10) >> 0x1f,
   (*(int *)(param_1 + 0x10) * 0x20000000 + uVar5 >> 0x1d | uVar5 << 3) == uVar5))
```

This tests for 32-bit overflow when multiplying by 0x20000000:

| sequence_count | × 0x20000000 | Overflows to 0? | Result |
|----------------|--------------|-----------------|---------|
| 0 | 0 | N/A | TRUE (but no sequences exist) |
| 1-7 | Non-zero | No | FALSE |
| **8** | **0x100000000** | **Yes → 0** | **TRUE** |
| 9-15 | Non-zero | No | FALSE |
| **16** | **0x200000000** | **Yes → 0** | **TRUE** |
| 50 | Wraps to non-zero | No | FALSE |

### Practical Examples

#### Water Gun (Effect 334)
- File index: 0
- Base animation_index: 45
- Sequence count in file 0: 50
- **50 % 8 = 2 ≠ 0**
- **Result:** NOT directional
- Direction only affects projectile translation (X/Y movement)
- Single animation sequence used regardless of attacker direction

#### Shadow Sneak (Effect 479)
- File index: 184
- Base animation_index: 8
- Sequence count in file 184: 16
- **16 % 8 = 0**
- **Result:** DIRECTIONAL
- Final animation_index = 8 + direction (0-7)
- Sequences 8-15 are 8 pre-rotated variants
- Used for effects that must visually rotate

### Summary: Directional vs Non-Directional

**Directional Effects** (sequence_count % 8 == 0):
- ROM adds direction (0-7) to base animation_index
- WAN file contains 8 consecutive sequences with identical frame counts
- Each sequence is a pre-rotated version of the same effect
- Used for effects that must visually rotate (shadows, directional beams)

**Non-Directional Effects** (sequence_count % 8 != 0):
- ROM uses base animation_index directly (no direction added)
- Single animation sequence regardless of attacker direction
- Direction only affects projectile translation via velocity vectors
- Used for symmetric effects (particles, explosions, splashes)

### sfx_id (offset 0x10)

Sound effect to play when animation starts.

| Value | Meaning |
|-------|---------|
| -1 (0xFFFFFFFF) | No sound effect |
| 0x3F00 (16128) | Silence constant |
| Other | Sound effect table index |

**Evidence:** `FUN_022bdfc0`
```c
*(int *)(param_1 + 0x58) = peVar3->se_id;
```

### timing_offset (offset 0x14)

Added to timing calculations. Always 0 in observed data - may be reserved.

**Evidence:** `FUN_022bdfc0`
```c
*(int *)(param_1 + 0x5c) = *(int *)(param_1 + 0x18) + peVar3->field_0x14;
```

### screen_effect_param (offset 0x18)

Parameter for type 5 screen effects. Stored in global state during setup.

**Evidence:** `FUN_022bdfc0`
```c
case 5:
    if (param_2 == 0) {
        *(uint8_t *)(*piVar2 + 0x279e) = peVar3->field_0x18;
    }
    else if (param_2 == 1) {
        *(uint8_t *)(*piVar2 + 0x279f) = peVar3->field_0x18;
    }
    break;
```

### attachment_point (offset 0x19)

**SIGNED int8_t.** Index for position offset lookup relative to target sprite.

| Value | Name | Description |
|-------|------|-------------|
| -1 / 0xFF | None | No lookup, use default position |
| 0 | Head | Offset to head position |
| 1 | LeftHand | Offset to left hand/appendage |
| 2 | RightHand | Offset to right hand/appendage |
| 3 | Centre | Offset to body center |

**Evidence:** `FUN_023258ec`
```c
if (peVar9->field_0x19 != 0xff) {
    uVar12 = (uint)(byte)peVar9->field_0x19;
    FUN_0201cf90((short *)local_30, (ushort *)(target + 0xb), uVar12);
    FUN_0201cf90((short *)local_44, (ushort *)(attacker + 0xb), uVar12);
}
```

**Evidence:** `FUN_0201cf90` validates range
```c
if (3 < param_3) {
    return;  // Invalid index, output zeroes
}
```

### is_non_blocking (offset 0x1A)

Controls whether game waits for animation to complete.

| Value | Behavior |
|-------|----------|
| 0x00 | Blocking - game waits |
| Non-zero | Non-blocking - plays asynchronously |

**Evidence:** `FUN_022bdfc0`
```c
*(uint8_t *)(param_1 + 0x60) = peVar3->is_non_blocking;
```

### loop_flag (offset 0x1B)

Controls animation looping.

| Value | Behavior |
|-------|----------|
| 0x00 | Play once |
| Non-zero | Loop continuously |

**Evidence:** `FUN_022bdfc0`
```c
*(uint8_t *)(param_1 + 0x61) = peVar3->unk_repeat;
```

Passed to `SetAnimationForAnimationControl` as 7th parameter.

## Accessor Function
```c
effect_animation* GetEffectAnimation(int effect_id) {
    return (effect_animation*)(effect_id * 0x1C + EFFECT_ANIMATION_INFO);
}
```

**Evidence:** `GetEffectAnimation` at `0x022BFE9C`

## Effect Context Structure

The game allocates a 316-byte (0x13C) context for each active effect. Maximum 32 simultaneous effects.
```c
struct effect_context {
    /* 0x00 */ int32_t param_from_caller;
    /* 0x04 */ int32_t caller_context;
    /* 0x08 */ int32_t anim_type;
    /* 0x0C */ int32_t instance_id;
    /* 0x10 */ int32_t wan_ptr;
    /* 0x14 */ int32_t effect_id;
    /* 0x18 */ int32_t unknown_0x18;
    /* 0x1C */ int32_t direction;
    /* 0x20 */ int16_t current_x;
    /* 0x22 */ int16_t current_y;
    /* 0x24 */ int16_t velocity_x;
    /* 0x26 */ int16_t velocity_y;
    /* ... */
    /* 0x40 */ int32_t stored_anim_type;
    /* 0x44 */ int32_t file_index;
    /* 0x48 */ int32_t palette_index;
    /* 0x50 */ int32_t animation_index;
    /* 0x58 */ int32_t sfx_id;
    /* 0x5C */ int32_t timing_value;
    /* 0x60 */ uint8_t is_non_blocking;
    /* 0x61 */ uint8_t loop_flag;
    /* ... */
    /* 0x128 */ int16_t source_x;
    /* 0x12A */ int16_t source_y;
    /* 0x12C */ int16_t dest_x;
    /* 0x12E */ int16_t dest_y;
    /* 0x130 */ uint16_t entity_id;
    /* 0x132 */ int16_t stored_velocity_x;
    /* 0x134 */ int16_t stored_velocity_y;
};
// Size: 0x13C (316 bytes)
```

**Evidence:** `FUN_022be44c`
```c
// Entry size: 0x4F × 4 = 0x13C bytes
ptr = ptr + 0x4f;

// Pool limit: 32 entries
if (0x1f < (int)uVar7) { ... }
```

## Open Questions

- Exact purpose of types 1 vs 2 (both use shared resource, differ by state check)
- Complete screen effect control structure at context + 0xE8
- How looping effects are terminated

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `GetEffectAnimation` | `0x022BFE9C` | Look up effect_animation by ID |
| `FUN_022be44c` | `0x022be44c` | Allocate effect context |
| `FUN_022bdfc0` | `0x022bdfc0` | Initialize animation from effect_animation |
| `FUN_022be780` | `0x022be780` | Main effect dispatcher |
| `FUN_022bde50` | `0x022bde50` | Stop/cleanup effect |
| `FUN_022bdec4` | `0x022bdec4` | Force-stop effect |
| `FUN_0201cf90` | `0x0201cf90` | Calculate attachment point offset |
| `FUN_0201da20` | `0x0201da20` | Sprite sequence count lookup from WAN_TABLE |
| `LAB_0201da00` | `0x0201da00` | Extract sequence count from WAN header |
| `SetAnimationForAnimationControl` | - | Configure animation parameters |
| `SetAnimationForAnimationControlInternal` | `0x0201C5C4` | Inner animation setup (critical group/sequence logic) |
