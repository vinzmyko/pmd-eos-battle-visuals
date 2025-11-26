# Reverse Engineering Findings: EFFECT_ANIMATION_INFO

## Overview

The `EFFECT_ANIMATION_INFO` table defines visual effect animations used by moves, items, traps, and other game events. Each entry specifies which WAN file to load, which animation to play, sound effects, and rendering parameters.

Table Properties:
- Entry Count: 700 entries
- Entry Size: 28 bytes (0x1C)
- Total Size: 19,600 bytes (0x4CB0)

## Memory Locations

| Region | Address | Symbol Name |
|--------|---------|-------------|
| NA | `0x022CC52C` | `EFFECT_ANIMATION_INFO` |
| EU | `0x022CCE84` | `EFFECT_ANIMATION_INFO` |
| JP | `0x022CDC14` | `EFFECT_ANIMATION_INFO` |

---

## Structure Definition

```c
struct effect_animation {
    int32_t anim_type;           // 0x00: Animation type/rendering category (0-6)
    int32_t file_index;          // 0x04: Index into effect.bin pack file
    int32_t palette_index;       // 0x08: Palette slot for rendering (0-14)
    int32_t animation_index;     // 0x0C: Base animation group within WAN file
    int32_t sfx_id;              // 0x10: Sound effect ID (-1 = none)
    int32_t timing_offset;       // 0x14: Added to timing calculations
    uint8_t screen_effect_param; // 0x18: Parameter for type 5 screen effects
    int8_t  attachment_point;    // 0x19: Position offset lookup (-1 to 3) - SIGNED
    uint8_t is_non_blocking;     // 0x1A: If non-zero, animation plays asynchronously
    uint8_t loop_flag;           // 0x1B: If non-zero, animation loops
};
// Size: 0x1C (28 bytes)
```

---

## Field Documentation

### Offset 0x00: Animation Type

Type: `int32_t`

Purpose: Determines the rendering path, resource allocation strategy, and system state requirements for the effect animation.

Evidence:
- `GetEffectAnimation`: Returns pointer to effect_animation entry, accessed as `peVar3->field_0x0`
- `FUN_022be44c`: Main effect allocation function, uses type to determine allocation path
- `FUN_022bdfc0`: Effect setup function, stores type at context offset 0x40 and uses switch statement for initialization

Type Values and Behaviors:

| Value | Name | File Index Handling | System Requirements | Resource Allocation |
|-------|------|---------------------|---------------------|---------------------|
| 0 | Invalid | - | - | Returns -1 (no effect) |
| 1 | WanFile0 | Direct | `*(DAT+0x2784) == 0` | Shared resource at `DAT+0x2788` |
| 2 | WanFile1 | Direct | `*(DAT+0x2784) == 1` | Shared resource at `DAT+0x2788` |
| 3 | WanOther | Direct | None | Standard allocation, clears conflicting type-3 |
| 4 | Wat | Direct | None | Standard allocation, clears conflicting type-4 |
| 5 | Screen | `file_index + 0x10C` | Check at struct offset 0x8 | Special via `FUN_022c01fc` |
| 6 | Wba | Direct | Check at struct offset 0xC | Special via `FUN_022c0280` |

Type 1 and 2 System State Check:
```c
// From FUN_022be44c
if ((iVar6 == 2) && (*(int *)(*DAT_022be72c + 0x2784) != 1)) {
    return -1;  // Type 2 requires state == 1
}
if ((iVar6 == 1) && (*(int *)(*DAT_022be72c + 0x2784) != 0)) {
    return -1;  // Type 1 requires state == 0
}
```

Type 3 Conflict Resolution:
```c
// From FUN_022be44c - clears other type-3 effects with different file_index
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

Type 4 Conflict Resolution:
```c
// From FUN_022be44c - clears other type-4 effects with different effect_id
else if (iVar6 == 4) {
    for (i = 0; i < 0x20; i++) {
        if ((entry->id != -1) && (entry->type == 4) &&
            (entry->effect_id != current_effect_id)) {
            FUN_022bdec4(entry, 1);  // Clear conflicting effect
        }
    }
}
```

Type 5 File Index Offset:
```c
// From FUN_022c01fc - type 5 uses file_index + 0x10C (268 decimal)
if (param_2 != 5) return;
iVar1 = FUN_022c067c(5, param_3, param_4);  // param_3 = file_index
// Actual file loaded is effect.bin[file_index + 268]
```

Animation Control Initialization by Type:
```c
// From FUN_022bdfc0 second switch statement
switch (anim_type) {
case 1:
case 2:
case 3:
    InitAnimationControlWithSet((animation_control *)(context + 0x68));
    SetSpriteIdForAnimationControl(...);
    SetAnimationForAnimationControl(..., animation_index, DIR_DOWN, ...);
    // Sets rendering flags at context + 0x6a
    break;
case 4:
    // Similar to 1-3 but with different palette handling
    FUN_0201d0f8(context + 0x68, palette_value);
    break;
case 5:
case 6:
    // Screen effect initialization
    FUN_020640bc(context + 0xe8);
    FUN_020640cc(...);
    FUN_020640dc(...);
    break;
}
```

---

### Offset 0x04: File Index

Type: `int32_t`

Purpose: Index into the effect.bin pack archive file. Specifies which WAN/WAT sprite file to load for the effect animation.

Evidence:
- `FUN_022bdfc0`: Stores at context offset 0x44: `*(int *)(param_1 + 0x44) = peVar3->file_index`
- `FUN_022be44c`: Used with pack file loading functions
- `LoadWanTableEntryFromPack`: Takes file_index as parameter for `PACK_ARCHIVE_EFFECT` (pack ID 3)

Special Handling for Type 5:
```c
// Type 5 effects add 0x10C (268) to the file index
// From FUN_022c01fc
if (param_2 == 5) {
    // file_index + 0x10C is used for actual file lookup
    FUN_022c067c(5, file_index, param_4);
}
```

Pack File Reference:
- Pack Archive ID: 3 (`PACK_ARCHIVE_EFFECT`)
- File Path: `/EFFECT/effect.bin`
- Format: AT4PX compressed pack archive
- Contains: WAN sprites (types 1-4), screen effect data (types 5-6)

---

### Offset 0x08: Palette Index

Type: `int32_t`

Purpose: Specifies which palette slot to use when rendering the effect sprite. Passed to animation control setup functions.

Observed Range: 0-14 in extracted data

Evidence:
- `FUN_022bdfc0`: Stores at context offset 0x48, then passed to `SetAnimationForAnimationControl`:
  ```c
  *(int *)(param_1 + 0x48) = peVar3->field_0x8;
  // ...
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
- `SetAnimationForAnimationControl`: Fifth parameter is `low_palette_pos` according to pmdsky-debug documentation

Usage Pattern:
- Values 0-12 typically reference static palettes loaded at game start
- Values 13-14 may reference dynamic palettes loaded per-effect
- The value is masked to 8 bits (`& 0xff`) before use

---

### Offset 0x0C: Animation Index

Type: `int32_t`

Purpose: Base animation group index within the WAN file. For directional effects, the entity's facing direction (0-7) is added to this value at runtime.

Evidence:
- `FUN_022bdfc0`: Stores base value, then conditionally adds direction offset:
  ```c
  // From FUN_022bdfc0
	*(int *)(param_1 + 0x50) = peVar3->animation_index;
	iVar6 = *(int *)(param_1 + 0x1c);  // Direction from effect params
	
	// Complex sprite type check - exact meaning not fully understood
	// Appears to verify WAN pointer validity and sprite type compatibility
	if ((iVar6 != -1) &&
	    (uVar1 = *(int *)(param_1 + 0x10) >> 0x1f,
	    (*(int *)(param_1 + 0x10) * 0x20000000 + uVar1 >> 0x1d | uVar1 << 3) == uVar1)) {
	    *(int *)(param_1 + 0x50) = *(int *)(param_1 + 0x50) + iVar6;
	}
  ```
- `SetAnimationForAnimationControl`: Receives final animation_index as `animation_key` parameter

Direction Offset System:
```c
// Final animation = base_animation_index + direction
// Direction values from direction_id enum:
// DIR_DOWN = 0, DIR_DOWN_RIGHT = 1, DIR_RIGHT = 2, DIR_UP_RIGHT = 3,
// DIR_UP = 4, DIR_UP_LEFT = 5, DIR_LEFT = 6, DIR_DOWN_LEFT = 7

// For directional effects, WAN file must contain 8 consecutive animation groups:
// animation_index + 0 = Down
// animation_index + 1 = Down-Right
// animation_index + 2 = Right
// animation_index + 3 = Up-Right
// animation_index + 4 = Up
// animation_index + 5 = Up-Left
// animation_index + 6 = Left
// animation_index + 7 = Down-Left
```

Non-Directional Effects:
- Direction parameter is -1
- animation_index is used directly without offset
- WAN file contains single animation group at animation_index

---

### Offset 0x10: Sound Effect ID

Type: `int32_t` (signed)

Purpose: Sound effect to play when the effect animation starts. Value -1 indicates no sound.

Evidence:
- `FUN_022bdfc0`: Stores at context offset 0x58: `*(int *)(param_1 + 0x58) = peVar3->se_id`
- Sound playback occurs during effect initialization

Special Values:
- `-1` (0xFFFFFFFF): No sound effect
- `0x3F00` (16128): Silence constant (same as move animations)
- Other values: Direct index into sound effect table

---

### Offset 0x14: Timing Offset

Type: `int32_t`

Purpose: Added to timing calculations for effect synchronization.

Evidence:
- `FUN_022bdfc0`: Added to value from context offset 0x18:
  ```c
  *(int *)(param_1 + 0x5c) = *(int *)(param_1 + 0x18) + peVar3->field_0x14;
  ```

Observed Values: Always 0 in extracted data samples, suggesting this field may be reserved or rarely used.

---

### Offset 0x18: Screen Effect Parameter

Type: `uint8_t`

Purpose: Parameter specific to type 5 (screen) effects. Stored in global state during effect setup.

Evidence:
- `FUN_022bdfc0`: For type 5 effects, stores this value in global memory:
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

Usage: Controls screen-wide visual effects like flashes, fades, or overlays.

---

### Offset 0x19: Attachment Point

Type: `int8_t` (signed)

Purpose: Index for position offset lookup. Determines where the effect spawns relative to the target entity's sprite attachment points.

Range: -1 to 3 (or 255/0xFF for none)

Evidence:
- `FUN_0201cf90`: Position offset calculation function, validates range:
  ```c
  void FUN_0201cf90(short *output_offset, ushort *anim_ctrl, uint attachment_index) {
      *output_offset = 0;
      output_offset[1] = 0;
      
      if ((*anim_ctrl & 0x8000) == 0) return;  // Check flag
      if (3 < attachment_index) return;         // Range check (0-3 only)
      
      // Calculate offset from sprite's wan_offset table
      base = *(int *)(anim_ctrl + 0x28);
      frame_index = (int)(short)anim_ctrl[0x1d];
      offset_ptr = base + frame_index * 0x10 + attachment_index * 4;
      
      // Special case: (99, 99) means use center
      if (*offset_ptr == 99 && *(offset_ptr + 2) == 99) {
          *output_offset = 0x63;
          output_offset[1] = 0x63;
          return;
      }
      
      // Calculate final offset
      *output_offset = anim_ctrl[0x10] + *offset_ptr;
      output_offset[1] = anim_ctrl[0x11] + *(offset_ptr + 2);
  }
  ```
- `FUN_023258ec`: Uses attachment_point to position effects on entities:
  ```c
  if (peVar9->field_0x19 != 0xff) {
      uVar12 = (uint)(byte)peVar9->field_0x19;
      FUN_0201cf90((short *)local_30, (ushort *)(target + 0xb), uVar12);
      FUN_0201cf90((short *)local_44, (ushort *)(attacker + 0xb), uVar12);
  }
  ```

Attachment Point Values:

| Value | Name | Description |
|-------|------|-------------|
| -1 / 255 | None | No lookup, effect spawns at default position |
| 0 | Head | Offset to head position |
| 1 | LeftHand | Offset to left hand/appendage |
| 2 | RightHand | Offset to right hand/appendage |
| 3 | Centre | Offset to body center |

WAN Offset Structure Reference:
```c
// From wan.h
struct wan_offset {
    struct uvec2_16 head;        // Attachment point 0
    struct uvec2_16 hand_left;   // Attachment point 1
    struct uvec2_16 hand_right;  // Attachment point 2
    struct uvec2_16 center;      // Attachment point 3
};
// Size: 16 bytes (4 points × 4 bytes each)
```

---

### Offset 0x1A: Is Non-Blocking

Type: `uint8_t` (boolean)

Purpose: Controls whether the game waits for this effect to complete before continuing.

Evidence:
- `FUN_022bdfc0`: Stores at context offset 0x60: `*(uint8_t *)(param_1 + 0x60) = peVar3->is_non_blocking`
- `AnimationHasMoreFrames`: Called in loops to wait for blocking animations

Values:
- `0x00`: Blocking - game waits for animation to finish
- Non-zero: Non-blocking - animation plays asynchronously

---

### Offset 0x1B: Loop Flag

Type: `uint8_t` (boolean)

Purpose: Controls whether the animation loops continuously or plays once.

Evidence:
- `FUN_022bdfc0`: Stores at context offset 0x61: `*(uint8_t *)(param_1 + 0x61) = peVar3->unk_repeat`
- `SetAnimationForAnimationControl`: Receives as 7th parameter (unk3)

Values:
- `0x00`: Play once
- Non-zero: Loop continuously until explicitly stopped

---

## Effect Context Structure

The game allocates a 0x13C (316) byte structure for each active effect. Up to 32 effects can be active simultaneously.

Evidence:
- `FUN_022be44c`: Allocates context from pool: `ptr = ptr + 0x4f` (0x4F × 4 = 0x13C bytes per entry)
- Pool size check: `if (0x1f < (int)uVar7)` confirms 32 entry limit

```c
struct effect_context {
    /* 0x000 */ int32_t param_from_caller;    // param_3 in FUN_022be44c
    /* 0x004 */ int32_t caller_context;       // param_1 in FUN_022be44c
    /* 0x008 */ int32_t anim_type;            // Copied from effect_animation
    /* 0x00C */ int32_t instance_id;          // Unique ID for this effect instance
    /* 0x010 */ int32_t wan_ptr;              // WAN data pointer
    
    /* 0x014 */ int32_t effect_id;            // Effect animation ID
    /* 0x018 */ int32_t unknown_0x18;
    /* 0x01C */ int32_t direction;            // Direction 0-7, or -1 for none
    /* 0x020 */ int16_t current_x;            // Current render position X
    /* 0x022 */ int16_t current_y;            // Current render position Y
    /* 0x024 */ int16_t velocity_x;           // Movement per frame X
    /* 0x026 */ int16_t velocity_y;           // Movement per frame Y
    /* 0x028 */ int32_t unknown_0x28;
    /* 0x02C */ int32_t z_priority;           // Rendering layer priority
    
    /* 0x040 */ int32_t stored_anim_type;     // Copy of anim_type
    /* 0x044 */ int32_t file_index;           // effect.bin file index
    /* 0x048 */ int32_t palette_index;        // Palette slot
    /* 0x04C */ int32_t unknown_0x4c;
    /* 0x050 */ int32_t animation_index;      // Current animation (base + direction)
    /* 0x054 */ int32_t unknown_0x54;
    /* 0x058 */ int32_t sfx_id;               // Sound effect
    /* 0x05C */ int32_t timing_value;         // Timing offset result
    /* 0x060 */ uint8_t is_non_blocking;
    /* 0x061 */ uint8_t loop_flag;
    /* 0x062 */ uint8_t padding[2];
    
    /* 0x064 */ uint16_t sprite_id;           // WAN table sprite ID
    /* 0x066 */ uint16_t unknown_0x66;
    /* 0x068 */ animation_control anim_ctrl;  // Animation state (~0x42 bytes)
    
    /* 0x0E4 */ uint16_t screen_effect_id;    // For type 5/6
    /* 0x0E8 */ uint8_t screen_ctrl[0x1C];    // Screen effect control data
    /* 0x104 */ uint8_t screen_flag;
    
    /* 0x128 */ int16_t source_x;             // Projectile source X
    /* 0x12A */ int16_t source_y;             // Projectile source Y
    /* 0x12C */ int16_t dest_x;               // Projectile destination X
    /* 0x12E */ int16_t dest_y;               // Projectile destination Y
    /* 0x130 */ uint16_t entity_id;
    /* 0x132 */ int16_t stored_velocity_x;
    /* 0x134 */ int16_t stored_velocity_y;
    
    /* 0x13A */ uint8_t field_0x13a;
    /* 0x13B */ uint8_t padding_end;
};
// Size: 0x13C (316 bytes)
```

---

## Direction System

Direction for effect animations is determined at runtime based on entity facing direction, NOT stored in the effect table.

### Direction Enum

From `direction.h` in pmdsky-debug:
```c
enum direction_id {
    DIR_NONE = -1,
    DIR_DOWN = 0,
    DIR_DOWN_RIGHT = 1,
    DIR_RIGHT = 2,
    DIR_UP_RIGHT = 3,
    DIR_UP = 4,
    DIR_UP_LEFT = 5,
    DIR_LEFT = 6,
    DIR_DOWN_LEFT = 7,
    DIR_CURRENT = 8,
};

#define DIRECTION_MASK 7
#define NUM_DIRECTIONS 8
```

### Direction Calculation

From `position_util.c`:
```c
const s32 POSITION_DISPLACEMENT_TO_DIRECTION[3][3] = {
    {DIR_UP_LEFT, DIR_UP, DIR_UP_RIGHT},
    {DIR_LEFT, DIR_DOWN, DIR_RIGHT},
    {DIR_DOWN_LEFT, DIR_DOWN, DIR_DOWN_RIGHT}
};

s32 GetDirectionTowardsPosition(struct position *origin, struct position *target) {
    s32 x_diff = target->x - origin->x;
    s32 y_diff = target->y - origin->y;
    
    if (x_diff == 0 && y_diff == 0)
        return DIR_DOWN;
    
    // Clamp to -1, 0, or 1
    if (x_diff >= 1) x_diff = 1;
    if (y_diff >= 1) y_diff = 1;
    if (x_diff <= -1) x_diff = -1;
    if (y_diff <= -1) y_diff = -1;
    
    return POSITION_DISPLACEMENT_TO_DIRECTION[++y_diff][++x_diff];
}
```

### Direction Source

Entity facing direction is stored at entity info offset 0x4C:
```c
// From FUN_023230fc and FUN_02322f78
uint direction = *(byte *)(entity->info + 0x4c);  // 0-7

// Direction is passed through effect creation functions:
// FUN_02322f78:
local_20 = (uint)*(byte *)(entity_info + 0x4c);  // Read entity direction
iVar5 = FUN_022be9e8(&local_2c, &local_30, 0, local_20);  // param_4 = direction

// FUN_022be9e8:
uStack_14 = param_4;  // Direction stored in effect params template
// This eventually ends up at effect context offset 0x1C
```

### Direction Application

From `FUN_022bdfc0`:
```c
// Store base animation index
*(int *)(context + 0x50) = effect_animation->animation_index;

// Get direction from effect params
direction = *(int *)(context + 0x1c);

// Add direction to animation index if valid and sprite supports it
if ((direction != -1) && (sprite_supports_directions)) {
    *(int *)(context + 0x50) += direction;
}
```

### Directional vs Non-Directional Effects

Directional Effects:
- WAN file contains 8 consecutive animation groups
- Runtime adds direction (0-7) to base animation_index
- Used for projectiles, beams, directional attacks

Non-Directional Effects:
- WAN file contains single animation group
- Direction parameter is -1 or ignored
- Used for explosions, auras, screen effects

---

## WAN Table Management

The game maintains a global table of loaded WAN sprites to avoid redundant file loading.

### WAN Table Structure

```c
struct wan_table {
    // Header fields
    void* at_decompress_scratch_space;  // Scratch buffer for AT decompression
    int16_t next_alloc_start_pos;       // Hint for next allocation search
    int16_t total_nb_of_entries;        // Always 0x60 (96)
    
    // Entries array
    wan_table_entry sprites[96];        // 0x38 bytes each
};

struct wan_table_entry {
    wan_source_type_8 source_type;      // 0=null, 1=file, 2=pack
    int16_t pack_id;                    // Pack archive ID
    uint16_t file_index;                // File index within pack
    void* iov_base;                     // Raw data pointer
    uint32_t iov_len;                   // Data length
    int16_t reference_counter;          // Reference count
    char file_externally_allocated;     // If true, don't free on delete
    char path[...];                     // Path string (for source_type=1)
    wan_header* sprite_start;           // Parsed WAN header (at offset 0x30)
};
// Entry size: 0x38 (56 bytes)
```

### Source Type Enum

```c
enum wan_source_type {
    WAN_SOURCE_NULL = 0,  // Empty slot
    WAN_SOURCE_FILE = 1,  // Loaded from filesystem path
    WAN_SOURCE_PACK = 2,  // Loaded from pack archive
};
```

### Key Functions

| Function | Purpose |
|----------|---------|
| `InitWanTable` | Initialize table with 0x60 entries |
| `AllocateWanTableEntry` | Find free slot, zero it, return index |
| `DeleteWanTableEntry` | Decrement refcount, free if zero |
| `FindWanTableEntry` | Search by path string (source_type=1) |
| `GetLoadedWanTableEntry` | Search by pack_id + file_index (source_type=2) |
| `LoadWanTableEntry` | Load from filesystem, reuse if exists |
| `LoadWanTableEntryFromPack` | Load from pack archive, reuse if exists |
| `GetWanForAnimationControl` | Get WAN header for animation control |
| `WanHasAnimationGroup` | Check if WAN has specific animation group |

### Reference Counting

```c
// From LoadWanTableEntryFromPack
iVar2 = GetLoadedWanTableEntry(wan_table, pack_id, file_index);
if (iVar2 != -1) {
    // Already loaded - increment reference
    wan_table->sprites[iVar2].reference_counter++;
    return iVar2;
}
// Not loaded - allocate new entry and load file
```

```c
// From DeleteWanTableEntry
if (entry->file_externally_allocated) {
    MemZero(entry, 0x38);  // Always delete if external
    return;
}
if (entry->iov_base != NULL && entry->reference_counter != 0) {
    entry->reference_counter--;
    if (entry->reference_counter == 0) {
        MemFree(entry->iov_base);
        MemZero(entry, 0x38);
    }
}
```

---

## WAN File Structure

WAN files store animated sprites with meta-frames, animation sequences, and image data.

### WAN Header

```c
struct wan_header {
    wan_animation_header* anim_header;   // Pointer to animation data
    wan_image_header* image_header;      // Pointer to image data
    wan_sprite_type_16 sprite_type;      // 0=Props/UI, 1=Chara, 2=?, 3=3D engine
};
// Size: 10 bytes
```

### Sprite Types

```c
enum wan_sprite_type {
    WAN_SPRITE_PROPS_UI = 0,  // Object and UI sprites
    WAN_SPRITE_CHARA = 1,     // Monster/character sprites
    WAN_SPRITE_UNK_2 = 2,     // Unknown
    WAN_SPRITE_UNK_3 = 3,     // Uses 3D engine for rendering
};
```

### Animation Header

```c
struct wan_animation_header {
    wan_fragment** frames;              // Meta-frame groups
    wan_offset* frame_offsets;          // Per-frame attachment point offsets
    wan_animation_group* animations;    // Animation groups array
    uint16_t nb_animation_groups;       // Number of animation groups
    uint16_t allocation_for_max_frame;
    // ... additional fields
    uint16_t is_256_color_alt;
};
```

### Animation Group

```c
struct wan_animation_group {
    wan_animation_frame** pnt;  // Array of frame sequences
    uint16_t len;               // Number of sequences (1 or 8 for directional)
    uint16_t loop_start;        // Frame to restart from when looping
};
// Size: 8 bytes
```

### Animation Frame

```c
struct wan_animation_frame {
    uint8_t duration;           // Frame display time in game ticks
    uint8_t flag;               // Frame flags
    uint16_t frame_id;          // Meta-frame index to display
    vec2_16 offset;             // Position offset from sprite center
    vec2_16 shadow_offset;      // Shadow position offset
};
// Size: 12 bytes
```

### Attachment Point Offsets

```c
struct wan_offset {
    uvec2_16 head;        // Offset 0x0: Head attachment point
    uvec2_16 hand_left;   // Offset 0x4: Left hand attachment point
    uvec2_16 hand_right;  // Offset 0x8: Right hand attachment point
    uvec2_16 center;      // Offset 0xC: Center attachment point
};
// Size: 16 bytes
```

---

## Effect Meta-Frame Format (effect.bin specific)

Effect WAN files use a different meta-frame format than character WAN files.

From Project Pokemon documentation:

```
Meta-Frame Structure (10 bytes):
Offset 0-1: Always 0xFFFF
Offset 2:   Always 0x00
Offset 3:   0x00 or 0xFB (0xFB = draw behind character)
Offset 4:   Y offset (signed)
Offset 5:   Dimension flags (binary)
            Bits 0-1: Unknown (00 in most, 10 in some)
            Bit 2: Unknown (always 1)
            Bit 3: Mosaic mode (always 0)
            Bits 4-5: Unknown (always 00)
            Bits 6-7: Upper bits for Y offset
Offset 6:   X offset (signed)
Offset 7:   Dimension/flip flags (binary)
            Bits 0-1: Size (00=8x8, 01=16x16, 10=32x32, 11=64x64)
            Bit 2: Flip vertical
            Bit 3: Flip horizontal
            Bit 4: Is last meta-frame in group
            Bits 5-6: Unknown (always 10)
            Bit 7: Upper bit for X offset
Offset 8:   Image offset by blocks (1 LSB = 2 8x8 tiles)
Offset 9:   Palette index (0-C=static, D-E=custom)
            Lower nibble: palette index
            Upper nibble: always 0xC
```

---

## Function Reference

### Core Accessor Functions

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `GetEffectAnimation` | `0x022BFE9C` | Returns pointer: `effect_id * 0x1C + EFFECT_ANIMATION_INFO` |
| `GetMoveAnimation` | - | Returns move animation entry for effect_id lookup |
| `GetSpecialMonsterMoveAnimation` | - | Returns per-Pokemon override entry |

### Effect Allocation and Setup

| Function | Purpose |
|----------|---------|
| `FUN_022be44c` | Allocate effect context, load WAN, initialize state |
| `FUN_022be730` | Wrapper: allocate + start playback |
| `FUN_022bdfc0` | Initialize animation control from effect_animation data |
| `FUN_022be780` | Main effect dispatcher (called by move handlers) |
| `FUN_022be9a0` | Convert effect handle to context index |
| `FUN_022be9e8` | Create projectile effect with direction calculation |

### Animation Control Functions

| Function | Purpose |
|----------|---------|
| `InitAnimationControlWithSet` | Initialize animation_control structure |
| `SetSpriteIdForAnimationControl` | Set WAN table sprite reference |
| `SetAnimationForAnimationControl` | Configure animation parameters |
| `SetAnimationForAnimationControlInternal` | Internal implementation |
| `AnimationHasMoreFrames` | Check if animation still playing |

### WAN Table Functions

| Function | Purpose |
|----------|---------|
| `InitWanTable` | Initialize with 96 entries |
| `AllocateWanTableEntry` | Find and allocate free slot |
| `DeleteWanTableEntry` | Decrement refcount, optionally free |
| `FindWanTableEntry` | Search by path string |
| `GetLoadedWanTableEntry` | Search by pack_id + file_index |
| `LoadWanTableEntry` | Load from filesystem |
| `LoadWanTableEntryFromPack` | Load from pack archive |
| `LoadWanTableEntryFromPackUseProvidedMemory` | Load into caller's buffer |
| `ReplaceWanFromBinFile` | Replace existing entry's data |
| `GetWanForAnimationControl` | Get WAN header pointer |
| `SpriteTypeInWanTable` | Get sprite type for entry |
| `WanHasAnimationGroup` | Check if animation group exists |
| `WanTableSpriteHasAnimationGroup` | Check via table lookup |

### Type-Specific Handlers

| Function | Type | Purpose |
|----------|------|---------|
| `FUN_022c01fc` | 5 | Screen effect setup |
| `FUN_022c0280` | 6 | WBA effect setup |
| `FUN_022c03f4` | 3,4 | Standard effect resource allocation |
| `FUN_022c0450` | 5,6 | Special effect resource allocation |
| `FUN_022c067c` | 5,6 | Effect resource lookup |

### Position and Direction

| Function                      | Purpose                                                                                                                                                                                                                                           |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `FUN_0201cf90`                | Calculate position offset from attachment point index (0-3)                                                                                                                                                                                       |
| `GetDirectionTowardsPosition` | Calculate direction (0-7) from position delta                                                                                                                                                                                                     |
| `FUN_022bf088`                | Look up attachment point for specific Pokemon species. Searches special_monster_move_animation table for species-specific override; returns default attachment_point_idx if no override found. Note: This returns attachment point, NOT direction |
### Effect Playback

| Function       | Purpose                                             |
| -------------- | --------------------------------------------------- |
| `FUN_022bde50` | Stop/cleanup effect                                 |
| `FUN_022bdec4` | Force-stop effect                                   |
| `FUN_022beb2c` | Update effect position during projectile motion     |
| `FUN_022bfaa8` | Play effect layer 0 (charge/preparation)            |
| `FUN_022bfc5c` | Play effect layer 2 (primary)                       |
| `FUN_022bed90` | Play effect layer 1 (secondary), handles Razor Leaf |
| `FUN_022be9e8` | Play effect layer 3 (projectile) with direction     |

---

## Effect Layer System

Moves can use up to 4 effect layers simultaneously. Each layer is played by a different handler function.

| Layer | Offset | Handler | Typical Use |
|-------|--------|---------|-------------|
| 0 | 0x00 | `FUN_022bfaa8` | Charge-up, preparation effects |
| 1 | 0x02 | `FUN_022bed90` | Secondary impacts, multi-hit |
| 2 | 0x04 | `FUN_022bfc5c` | Primary visual effect |
| 3 | 0x06 | `FUN_022be9e8` | Projectile, additional effects |
Layer Usage Notes:
- Layers are NOT strictly ordered - any layer can be zero (unused)
- Multiple layers can reference the same effect_id
- Layer 3 (offset 0x06) is used for projectile effects via FUN_022be9e8 which calculates direction
- Layer 0 and 2 use FUN_022bfaa8 and FUN_022bfc5c respectively, which call FUN_022be780 with different type parameters

Layer Check Example:
```c
// From FUN_022bf160 - checks if any layer uses type 5
for (i = 0; i < 4; i++) {
    effect_id = move_animation->effect_layers[i];
    effect_anim = GetEffectAnimation(effect_id);
    if (effect_anim->anim_type == 5) {
        return true;  // Has type 5 effect
    }
}
```

---

## Confirmed vs Uncertain Fields

### Fully Confirmed
- `anim_type` (0x00): Values 0-6 with distinct behaviors, verified via switch statements in FUN_022bdfc0 and FUN_022be44c
- `file_index` (0x04): Direct pack index, +0x10C for type 5 (verified in FUN_022c01fc)
- `palette_index` (0x08): Range 0-14, passed to SetAnimationForAnimationControl as `low_palette_pos` parameter
- `animation_index` (0x0C): Base index, direction (0-7) added at runtime for directional effects
- `sfx_id` (0x10): Sound effect ID, -1 = none (stored at context+0x58)
- `attachment_point` (0x19): Signed int8, range -1 to 3, used by FUN_0201cf90 for position offset lookup
- `is_non_blocking` (0x1A): Boolean, controls whether game waits for animation completion
- `loop_flag` (0x1B): Boolean, passed to SetAnimationForAnimationControl as 7th parameter

### Partially Confirmed
- `timing_offset` (0x14): Added to value from context+0x18, stored at context+0x5c. Always 0 in extracted samples - may be reserved or rarely used
- `screen_effect_param` (0x18): Used for type 5 screen effects, stored at global offset 0x279e or 0x279f depending on param_2 value

### Unknown/Needs Investigation
- Exact purpose of types 1 vs 2 (both use shared resource at 0x2788, differ by system state check at 0x2784)
- Exact purpose of types 3 vs 4 (both standard allocation, differ in conflict resolution logic)
- The bit manipulation check for direction application:
```c
  (*(int *)(param_1 + 0x10) * 0x20000000 + uVar1 >> 0x1d | uVar1 << 3) == uVar1
```
  Appears to validate WAN pointer or sprite type, but exact meaning unclear
- Complete screen effect control structure at context+0xE8 (initialized by FUN_020640bc/cc/dc)
- What context offset 0x18 contains (used in timing_offset calculation)
- Full structure of the resource returned by FUN_0206409c (used in type 5/6 checks)

---

## Pack File Information

Effect animations are stored in:
- Path: `/EFFECT/effect.bin`
- Pack Type: AT4PX compressed pack archive
- Pack ID: 3 (`PACK_ARCHIVE_EFFECT`)
- File Count: Varies by region (270+ files)

Type 5 effects use files at index `file_index + 0x10C` (268), suggesting the pack is split:
- Indices 0-267: Standard WAN/WAT effect sprites
- Indices 268+: Screen effect data files

---

## Version Information

ROM Version: NA release (C2SE)
Analysis Date: 2025

Related Tables:
- `MOVE_ANIMATION_INFO`: References EFFECT_ANIMATION_INFO via effect_id fields
- `TRAP_ANIMATION_TABLE`: 2-byte entries with effect_id
- `ITEM_ANIMATION_TABLE`: 4-byte entries with two effect_ids

Credits:
- Reverse engineering: Ghidra decompilation analysis
- pmdsky-debug: Symbol names and structure definitions
- Project Pokemon: WAN format documentation