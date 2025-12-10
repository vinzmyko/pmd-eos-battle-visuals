# Entity Positioning

## Summary

- Tile size is 24×24 pixels
- Pixel positions stored in 8.8 fixed point format (multiply by 256)
- Items spawn at tile corner + 4 pixels (X and Y)
- Monsters/entities spawn at tile center X (+12), lower Y (+16) for feet positioning
- Attachment points provide per-frame sprite offsets for effect spawning

## Coordinate Systems

### Tile Space
- Integer coordinates (0, 0) to (map_width-1, map_height-1)
- Stored in `entity->pos.x` and `entity->pos.y`
- Map bounds: X < 0x38 (56), Y < 0x20 (32)

### Pixel Space (8.8 Fixed Point)
- Internal format: upper 8 bits = pixel, lower 8 bits = sub-pixel
- Stored in `entity->pixel_pos.x` and `entity->pixel_pos.y`
- To get screen pixels: `pixel_pos >> 8`

## Position Formulas

### Item Positioning

Items spawn near the top-left corner of tiles.

| Axis | Formula | Offset |
|------|---------|--------|
| X | `(tile_x * 24 + 4) * 256` | +4 pixels |
| Y | `(tile_y * 24 + 4) * 256` | +4 pixels |

**Evidence:** `SpawnItemEntity`
```c
(entity->pixel_pos).x = (position->x * 0x18 + 4) * 0x100;
(entity->pixel_pos).y = (position->y * 0x18 + 4) * 0x100;
```

### Monster/Entity Positioning

Monsters spawn at tile center horizontally, with Y offset for feet anchoring.

| Axis | Formula | Offset | Meaning |
|------|---------|--------|---------|
| X | `(tile_x * 24 + 12) * 256` | +12 pixels | Tile center |
| Y | `(tile_y * 24 + 16) * 256` | +16 pixels | Below center (feet) |

**Evidence:** `UpdateEntityPixelPos`
```c
if (pixel_pos == NULL) {
    (entity->pixel_pos).x = ((entity->pos).x * 0x18 + 0xc) * 0x100;
    (entity->pixel_pos).y = ((entity->pos).y * 0x18 + 0x10) * 0x100;
}
```

### Position-Based Effects

When spawning effects at a tile position (no target entity), uses monster formula.

**Evidence:** `PlayMoveAnimation` (when target is NULL)
```c
local_30 = (undefined2)((uint)((position->x * 0x18 + 0xc) * 0x100) >> 8);
local_2e = (undefined2)((uint)((position->y * 0x18 + 0x10) * 0x100) >> 8);
```

## Attachment Points

Effects can spawn relative to sprite attachment points rather than entity center.

### Attachment Point Indices

| Index | Name | Description |
|-------|------|-------------|
| -1 / 0xFF | None | No lookup, use default position |
| 0 | Head | Top of sprite |
| 1 | LeftHand | Left appendage |
| 2 | RightHand | Right appendage |
| 3 | Centre | Body center |

### Offset Calculation

Attachment point offsets are stored per-frame in WAN files. The function looks up the offset for the current animation frame.

**Evidence:** `FUN_0201cf90`
```c
void FUN_0201cf90(short *output, ushort *anim_ctrl, uint attachment_index)
{
    *output = 0;
    output[1] = 0;
    
    // Check animation control flag
    if ((*anim_ctrl & 0x8000) == 0) return;
    
    // Validate index (0-3 only)
    if (3 < attachment_index) return;
    
    // Get current frame
    int frame_id = (int)(short)anim_ctrl[0x1d];
    if (frame_id == -1) return;
    
    // Get WAN offset table
    int offset_table = *(int *)(anim_ctrl + 0x28);
    if (offset_table == 0) return;
    
    // Each frame has 16 bytes (4 points × 4 bytes)
    short *point = (short *)(offset_table + frame_id * 0x10 + attachment_index * 4);
    
    // Special case: (99, 99) means use center
    if (*point == 99 && *(point + 1) == 99) {
        *output = 0x63;      // 99
        output[1] = 0x63;    // 99
        return;
    }
    
    // Add sprite offset to attachment point
    *output = anim_ctrl[0x10] + *point;
    output[1] = anim_ctrl[0x11] + *(point + 1);
}
```

### WAN Offset Structure

Each animation frame stores 4 attachment points:
```c
struct wan_offset {
    int16_t head_x, head_y;        // Index 0
    int16_t left_hand_x, left_hand_y;   // Index 1  
    int16_t right_hand_x, right_hand_y; // Index 2
    int16_t center_x, center_y;    // Index 3
};
// Size: 16 bytes per frame
```

## Helper Functions

**Evidence:** `SetEntityPixelPosXY`
```c
void SetEntityPixelPosXY(entity *entity, uint32_t x, uint32_t y)
{
    (entity->pixel_pos).x = x;
    (entity->pixel_pos).y = y;
}
```

**Evidence:** `IncrementEntityPixelPosXY`
```c
void IncrementEntityPixelPosXY(entity *entity, uint32_t x, uint32_t y)
{
    (entity->pixel_pos).x = (entity->pixel_pos).x + x;
    (entity->pixel_pos).y = (entity->pixel_pos).y + y;
}
```

## Constants

| Constant | Value | Hex |
|----------|-------|-----|
| TILE_SIZE | 24 | 0x18 |
| FIXED_POINT_SHIFT | 8 | - |
| FIXED_POINT_MULTIPLIER | 256 | 0x100 |
| ITEM_OFFSET_X | 4 | 0x04 |
| ITEM_OFFSET_Y | 4 | 0x04 |
| ENTITY_OFFSET_X | 12 | 0x0C |
| ENTITY_OFFSET_Y | 16 | 0x10 |

## Open Questions

- Exact offset values for all sprites' attachment points (stored in WAN files, need scraper to extract)
- How elevation (`entity->elevation`) affects Y position rendering

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `SpawnItemEntity` | - | Creates item entity with +4,+4 offset |
| `UpdateEntityPixelPos` | - | Sets entity pixel pos with +12,+16 offset |
| `SetEntityPixelPosXY` | - | Direct pixel position setter |
| `IncrementEntityPixelPosXY` | - | Adds to pixel position |
| `FUN_0201cf90` | `0x0201cf90` | Calculates attachment point offset |
