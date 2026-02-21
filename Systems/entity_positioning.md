# Entity Positioning

## Summary

- Tile size is 24×24 pixels
- Pixel positions stored in 8.8 fixed point format (multiply by 256)
- Items spawn at tile corner + 4 pixels (X and Y)
- Monsters/entities spawn at tile center X (+12), lower Y (+16) for feet positioning
- Projectiles start at attacker pixel position, end at target tile center
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

## Projectile Positioning

Projectiles use a specific source/destination model.

### Projectile Source Position

Projectile starts at **attacker's current pixel position**, NOT at tile center.

| Component | Formula | Description |
|-----------|---------|-------------|
| X | `attacker->pixel_pos.x >> 8` | Screen X pixel |
| Y | `attacker->pixel_pos.y >> 8` | Screen Y pixel |

**Evidence:** `FUN_02322f78`
```c
local_28 = (undefined2)((uint)param_1[3] >> 8);  // attacker->pixel_pos.x >> 8
local_26 = (undefined2)((uint)param_1[4] >> 8);  // attacker->pixel_pos.y >> 8
```

### Projectile Destination Position

Projectile ends at **target tile center**, using standard entity positioning offsets.

| Component | Formula | Offset | Description |
|-----------|---------|--------|-------------|
| X | `(tile_x * 24 + 12) * 256 >> 8` | +12 | Tile center X |
| Y | `(tile_y * 24 + 16) * 256 >> 8` | +16 | Below center Y (feet) |

**Evidence:** `FUN_02322f78`
```c
local_30 = (undefined2)((uint)((*param_2 * 0x18 + 0xc) * 0x100) >> 8);   // tile_x * 24 + 12
local_2e = (undefined2)((uint)((param_2[1] * 0x18 + 0x10) * 0x100) >> 8); // tile_y * 24 + 16
```

> See `Systems/projectile_motion.md` for complete projectile motion details.

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

### Attachment Points and Projectiles

**Important:** Attachment points are looked up for projectiles but **NOT applied to the trajectory source position**. The projectile always starts at the attacker's pixel position regardless of attachment point.

**Evidence:** `FUN_02322f78`
```c
// Attachment point calculated
FUN_0201cf90(&sStack_24, (ushort *)(param_1 + 0xb), uVar2 & 0xff);

// But source position taken directly from pixel_pos
local_28 = (undefined2)((uint)param_1[3] >> 8);  // No attachment offset added
local_26 = (undefined2)((uint)param_1[4] >> 8);
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

## Position Summary Table

| Entity Type | X Formula | Y Formula |
|-------------|-----------|-----------|
| Item | `(tile * 24 + 4) * 256` | `(tile * 24 + 4) * 256` |
| Monster/Entity | `(tile * 24 + 12) * 256` | `(tile * 24 + 16) * 256` |
| Projectile Source | `attacker.pixel_pos.x` | `attacker.pixel_pos.y` |
| Projectile Dest | `(tile * 24 + 12) * 256` | `(tile * 24 + 16) * 256` |


## Effect-Entity Binding System

Effects track target entities via a 3-slot binding table at `*DAT_022e6dcc + 0x618`. Maximum 3 effects can track entities simultaneously.

### Binding Slot Structure (0x10 bytes each)

| Offset | Type | Field | Description |
|--------|------|-------|-------------|
| +0x00 | int32 | effect_instance_id | -1 = free slot |
| +0x04 | int32 | bind_type | From FUN_022e6d68 param_3 (1, 5, or 6) |
| +0x08 | byte | attachment_point_index | 0-3 or 0xFF (none), read from effect_context + 0x28 |
| +0x0C | int32 | entity_pointer | Target entity to track |

### Bind Types

| Value | Layer | Usage |
|-------|-------|-------|
| 1 | Secondary (layer 1) | `FUN_02325644` |
| 5 | Charge (layer 0) | `FUN_02324e78` |
| 6 | Primary (layer 2) | `PlayMoveAnimation`, `FUN_023258ec` |

### Registration (FUN_022e6d68)

Called after spawning an effect to bind it to an entity. Finds a free slot and stores the effect handle, entity reference, bind type, and reads the attachment_point_index from the effect context via `FUN_022befd8`.

**Evidence:** `FUN_022e6d68`
```c
*puVar2 = param_1;       // effect_instance_id
puVar2[3] = param_2;     // entity_pointer
puVar2[1] = param_3;     // bind_type
iVar1 = FUN_022befd8((int)(short)*puVar2);  // reads effect_context + 0x28
*(char *)(puVar2 + 2) = (char)iVar1;        // attachment_point_index
```

### Per-Frame Position Sync (FUN_022e6e80)

Called per entity each frame. Iterates all 3 binding slots, updates any effect bound to that entity.

**Steps:**
1. Read entity pixel position: `entity + 0x0C` (x) and `entity + 0x10` (y), shifted >> 8
2. If attachment_point_index != 0xFF, call `FUN_0201cf90` with **entity's** anim_ctrl at `entity + 0x2C` to get per-frame attachment offset
3. Call `FUN_022bfb6c` to write position and offset into effect context

**Evidence:** `FUN_022e6e80`
```c
local_28 = (ushort)((uint)*(undefined4 *)(param_1 + 0xc) >> 8);   // entity pixel_x
local_26 = (undefined2)((uint)*(undefined4 *)(param_1 + 0x10) >> 8); // entity pixel_y
if (*(byte *)(puVar4 + 2) != 0xff) {
    FUN_0201cf90(&local_2c, (ushort *)(param_1 + 0x2c), (uint)*(byte *)(puVar4 + 2));
}
FUN_022bfb6c((int)(short)*puVar4, &local_28, &local_2c, param_2, (uint)*(byte *)(iVar5 + 0x4c));
```

**Critical for Q3:** The attachment offset is read from the **target Pokemon's** animation_control, using the Pokemon's current frame_id. For attachment_point=3 (Centre), this gives the Y offset from feet to body centre. Without this offset, effects render at feet.

### Position Write (FUN_022bfb6c)

Writes entity position and attachment offset into the effect context.

| Effect Context Offset | Value | Condition |
|---|---|---|
| +0x20 (current_x) | entity pixel_x >> 8 | Only if +0x136 == 0 AND +0x138 == 0 |
| +0x22 (current_y) | entity pixel_y >> 8 | Same condition |
| +0x24 | attachment_offset_x | If attachment_point != -1 |
| +0x26 | attachment_offset_y | If attachment_point != -1 |
| +0x2C (z_priority) | Varies by bind_type and directionality | See below |

**Position override:** If effect_context + 0x136 is nonzero, position comes from +0x138 instead of entity. This is how projectiles detach from entity tracking after launch.

**Z-priority logic:**
- bind_type 6 (primary): `cam_z + 1`
- Directional effect (sequence_count % 8 == 0): `cam_z + DIRECTION_Z_TABLE[entity_direction]` (8-entry table at `DAT_022bfc58`)
- Non-directional: `cam_z + 1`

### Cleanup (FUN_022e6dd0)

Iterates all 3 slots. If `AnimationHasMoreFrames` returns false for a slot's effect, clears the slot (sets instance_id to -1).

### Complete Per-Frame Pipeline
```
FUN_022e6e80 (per entity)
  │  entity->pixel_pos >> 8
  │  FUN_0201cf90(entity->anim_ctrl, attachment_index)
  └──► FUN_022bfb6c
         writes effect_context + 0x20/0x22 (base position)
         writes effect_context + 0x24/0x26 (attachment offset)
              │
              ▼
FUN_022bf4f0 (per effect, each tick)
  │  reads +0x24/0x26 as attachment offset
  │  checks for (99, 99) special case
  │  render_pos = (base_pos - camera) + attachment_offset
  └──► FUN_0201cf5c → FUN_0201c5c4 (render)
```

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
| `FUN_02322f78` | `0x02322f78` | Spawns projectile (demonstrates source/dest positions) |
