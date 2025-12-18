# WAN Animation Rendering System

> TODO: Refactor into file architecture.

## Summary

- WAN sprite type (offset 0x08) determines animation selection: CHARA (1) uses group+direction, PROPS_UI (0) ignores direction
- Flip flags are baked into WAN meta-frame piece data (bits 12-13 of piece[3]) - no runtime direction-based transformation
- Effect sprites always use `animations[0][animation_index]` regardless of direction parameter
- Direction may be added to animation_index for some sprites (controlled by `FUN_0201da20` return value)
- OAM override mask is initialized to zero - no external flip modification occurs during rendering

## Sprite Type Classification

### WAN Header Sprite Type (Offset 0x08)

The ROM distinguishes sprite handling based on the `sprite_type` field in the WAN header.

| Type | Name | Description |
|------|------|-------------|
| 0 | PROPS_UI | Effects, UI elements, props |
| 1 | CHARA | Character sprites (Pokémon, NPCs) |
| 3 | Unknown | Treated same as type 0 |

**Evidence:** `SetAnimationForAnimationControl` at `0x0201C4F4`
```c
wVar2 = SpriteTypeInWanTable(anim_ctrl->loaded_sprite_id);
if ((wVar2 == WAN_SPRITE_PROPS_UI) || ((wVar2 + 0xfe & 0xff) < 2)) {
    // PROPS_UI: Uses animation_key directly, IGNORES direction parameter
    SetAnimationForAnimationControlInternal(anim_ctrl, pwVar3, 0, animation_key, ...);
}
else {
    // CHARA: Uses animation_key as GROUP, direction as SEQUENCE within group
    if (WanTableSpriteHasAnimationGroup(sprite_id, animation_key)) {
        SetAnimationForAnimationControlInternal(anim_ctrl, pwVar3, animation_key, direction, ...);
    }
}
```

### Animation Selection Summary

| Sprite Type | Group Selection | Sequence Selection |
|-------------|-----------------|-------------------|
| CHARA (1) | `animation_group_id` parameter | `animation_id` parameter (direction) |
| PROPS_UI (0) | Always group 0 | `animation_id` parameter (flat index) |
| Type 3 | Always group 0 | `animation_id` parameter (flat index) |

## Animation Selection Logic

**Evidence:** `SetAnimationForAnimationControlInternal` at `0x0201C5C4`
```c
if (*(char *)&wan_header->sprite_type == '\x01') {
    // CHARA: animations[group_id][animation_id]
    pwVar3 = pwVar8->animations[animation_group_id].pnt[animation_id];
}
else {
    // PROPS_UI/Effects: animations[0][animation_id] - IGNORES group
    pwVar3 = pwVar8->animations->pnt[animation_id];
}
```

## Effect Context Direction Handling

### Direction Addition Logic

In effect initialization, direction (0-7) may be added to the base animation index.

**Evidence:** `FUN_022bdfc0`
```c
*(int *)(param_1 + 0x50) = peVar2->animation_index;  // Base from effect table

iVar6 = *(int *)(param_1 + 0x1c);  // Read direction
if ((iVar6 != -1) &&
   (uVar5 = *(int *)(param_1 + 0x10) >> 0x1f,
   (*(int *)(param_1 + 0x10) * 0x20000000 + uVar5 >> 0x1d | uVar5 << 3) == uVar5)) {
    // Condition: direction != -1 AND sprite_check_result == 0
    *(int *)(param_1 + 0x50) = *(int *)(param_1 + 0x50) + iVar6;  // Add direction!
}
```

The bit manipulation simplifies to checking if offset 0x10 equals 0. The function `FUN_0201da20` returns 0 for some sprites, enabling direction addition.

### Effect Context Structure (Partial)

```c
struct effect_context {
    // 0x00-0x0F: Unknown/reserved
    int32_t sprite_check_result;   // 0x10: Return from FUN_0201da20
    int32_t effect_id;             // 0x14: Effect animation ID from table
    int32_t timing;                // 0x18: Timing value
    int32_t direction;             // 0x1C: Direction (0-7), or -1 for non-directional
    int16_t position_x;            // 0x20: X position
    int16_t position_y;            // 0x22: Y position
    // ...
    int32_t effect_type;           // 0x40: Effect type (0-6)
    int32_t file_index;            // 0x44: WAN file index in effect.bin
    int32_t palette_slot;          // 0x48: Palette index
    int32_t unk_0x4c;              // 0x4C: Used for type 4
    int32_t animation_index;       // 0x50: Final animation index (may have direction added)
    int32_t render_param;          // 0x54: Some rendering parameter
    // ...
    uint16_t sprite_id;            // 0x64: Loaded sprite ID in WAN table
    animation_control anim_ctrl;   // 0x68: Embedded animation control (0x7C bytes)
    // ...
};
```

## Meta-Frame Rendering Pipeline

### Rendering Flow

```
animation_control->frame_id
        ↓
FUN_0201c5c4 (Meta-frame renderer)
        ↓
    Iterate meta-frame pieces (10 bytes each)
        ↓
FUN_0201b678 (Parse piece → OAM attributes)
        ↓
FUN_0201b6d4 (Build final OAM entry)
        ↓
DS OAM Hardware
```

### Meta-Frame Iteration

**Evidence:** `FUN_0201c5c4`
```c
while (true) {
    // Process piece...
    
    // Check is_last flag (bit 11 of piece[3])
    if (((int)(uint)puVar17[3] >> 0xb & 1U) != 0) break;
    
    puVar17 = puVar17 + 5;  // Next piece (10 bytes)
}
```

Key observations:
- Sprite type 3 (WAN_SPRITE_UNK_3): Special 3D rendering path
- Types 0-2: Standard meta-frame iteration
- Type 4: Different structure (WAT format)
- **No direction parameter** - function does not receive or use direction

## OAM Attribute Generation

### Meta-Frame Piece Parser

**Evidence:** `FUN_0201b678`
```c
void FUN_0201b678(undefined2 *output, undefined2 *piece) {
    uVar1 = DAT_0201b6d0;  // Mask constant
    
    output[0] = piece[0];
    output[1] = piece[1];
    output[2] = piece[2] & (ushort)uVar1;
    output[3] = piece[3] & ((ushort)uVar1 - 0xb00);  // Flip flags preserved here
    output[4] = piece[4];
    output[5] = /* combined bits from piece[2] and piece[3] */;
}
```

**Critical:** No direction parameter. Flip flags from piece[3] pass through unchanged.

### OAM Builder

**Evidence:** `FUN_0201b6d4`
```c
if (param_4 == NULL) {
    // Use values from meta-frame directly
    local_3a = local_2e;  // attr0
    local_38 = local_2c;  // attr1 (contains flip flags)
    local_36 = local_2a;  // attr2
}
else {
    // Apply optional mask/override (initialized to 0)
    local_3a = param_4[3] | local_2e & *param_4;
    local_38 = param_4[4] | local_2c & param_4[1];
    local_36 = param_4[5] | local_2a & param_4[2];
}
```

### DS OAM Attribute 1 (Flip Flags)

| Bit | Field |
|-----|-------|
| 0-8 | X coordinate |
| 12 | Horizontal Flip |
| 13 | Vertical Flip |
| 14-15 | Size |

## Meta-Frame Piece Structure

### Standard Meta-Frame Piece (10 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0x00 | 2 | piece[0] | Image/fragment reference |
| 0x02 | 2 | piece[1] | Y offset data |
| 0x04 | 2 | piece[2] | X offset + attribute bits |
| 0x06 | 2 | piece[3] | Flags: bit 11=is_last, bits 12-13=flip |
| 0x08 | 2 | piece[4] | Tile/palette data |

### Piece[3] Bit Layout

| Bits | Field | Values |
|------|-------|--------|
| 0-10 | Reserved/Size | |
| 11 | is_last | 0=more pieces, 1=last piece |
| 12 | h_flip | 0=normal, 1=flip horizontal |
| 13 | v_flip | 0=normal, 1=flip vertical |
| 14-15 | Size bits | |

## Direction Handling by Sprite Type

### CHARA Sprites (Type 1)

| Step | What Happens |
|------|--------------|
| 1 | `SetAnimationForAnimationControl` receives `animation_key` and `direction` |
| 2 | Calls `SetAnimationForAnimationControlInternal(anim_ctrl, wan, animation_key, direction, ...)` |
| 3 | ROM selects `animations[animation_key][direction]` |
| 4 | Each direction has pre-flipped meta-frame data |

### PROPS_UI Sprites (Type 0 - Effects)

| Step | What Happens |
|------|--------------|
| 1 | `SetAnimationForAnimationControl` receives `animation_key` and `direction` |
| 2 | Direction parameter **IGNORED** |
| 3 | Calls `SetAnimationForAnimationControlInternal(anim_ctrl, wan, 0, animation_key, ...)` |
| 4 | ROM selects `animations[0][animation_key]` |
| 5 | Flip flags are baked into WAN meta-frame data |

### Effect Types 1-4 Call Pattern

All effect types call with `DIR_DOWN` (direction is irrelevant for PROPS_UI):

```c
SetAnimationForAnimationControl(
    (animation_control *)(param_1 + 0x68),
    *(int *)(param_1 + 0x50),  // animation_index (may have direction added)
    DIR_DOWN,                   // IGNORED for PROPS_UI sprites
    ...
);
```

## How Directional Effects Work

For effects that visually change based on direction:

### Option A: Direction Added to animation_index

- If `FUN_0201da20(sprite_id)` returns 0
- AND direction != -1
- Then `animation_index += direction`
- Effect WAN file has 8 consecutive sequences (one per direction)

### Option B: Different Effect IDs

- Game logic selects different effect animation entries
- Each entry has different `animation_index` values
- Effect WAN file has pre-baked directional variants

### Option C: Non-Directional Effects

- Direction = -1 (no direction addition)
- Single sequence used for all directions
- Common for explosion/impact effects centered on target

## Open Questions

- What determines if `FUN_0201da20` returns 0 (enabling direction addition)? Function calls pointer on WAN header field at offset 0x30.
- For directional effects: Do they have exactly 8 sequences in group 0? Are sequences 0-7 visually rotated/flipped versions?
- What rendering pipeline state do effect types 1 vs 2 check at `*DAT_022be72c + 0x2784`?

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `SetAnimationForAnimationControl` | `0x0201C4F4` | Set animation, handles sprite type routing |
| `SetAnimationForAnimationControlInternal` | `0x0201C5C4` | Actually selects animation sequence |
| `FUN_0201c000` | `0x0201C000` | Initialize OAM attribute info (masks to 0) |
| `FUN_0201c5c4` | `0x0201C5C4` | Meta-frame renderer main loop |
| `FUN_0201b678` | `0x0201B678` | Parse meta-frame piece to OAM attributes |
| `FUN_0201b6d4` | `0x0201B6D4` | Build final OAM entry |
| `FUN_022be780` | `0x022BE780` | Main effect dispatcher |
| `FUN_022bdfc0` | `0x022BDFC0` | Initialize effect from effect_animation entry |
| `FUN_022beb2c` | `0x022BEB2C` | Update effect position during flight |
| `FUN_022be44c` | `0x022BE44C` | Allocate effect context |
| `FUN_0201da20` | `0x0201DA20` | Check sprite property (determines direction addition) |
| `WanTableSpriteHasAnimationGroup` | `0x0201DAA0` | Check if sprite has animation group |
| `SpriteTypeInWanTable` | `0x0201DA98` | Get sprite type from WAN table |
