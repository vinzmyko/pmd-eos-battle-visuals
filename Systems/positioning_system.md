# ROM Sprite Positioning System

## Overview

This document describes how Pokemon Mystery Dungeon: Explorers of Sky positions sprites relative to entity coordinates. Understanding this system is essential for the ROM scraper to output assets that render correctly in the game client.

> Refactor into reverse engineering architecture later

---

## Core Formula

The ROM uses a simple additive coordinate system:

```
Screen_Draw_Position = Entity_Position + SequenceFrame_Offset + MetaFramePiece_Offset
```

For scraper/client purposes (since MetaFramePiece offsets are baked into rendered frames):

```
Draw_Position = Entity_Position + SequenceFrame_Offset
```

Where:
- **Entity_Position** = Logical position of the monster (feet/ground point)
- **SequenceFrame_Offset** = Per-frame offset from WAN animation data (small values like 0, 1, -1, -2)

---

## Entity Position Semantics

The entity's `pixel_pos` represents the **ground/feet point** of the monster, NOT the visual center.

**Evidence:**
1. Shadow renders at `entity_pos + shadow_offset - 4` (the -4 places shadow slightly above feet)
2. SequenceFrame offsets are small adjustments (bobbing, shaking) not large positioning values
3. The offset system wouldn't work if entity_pos were at sprite center

**Tile-to-Pixel Formula** (from previous research):
```
Monster_Pixel_X = tile_x * 24 + 12  (center of 24px tile)
Monster_Pixel_Y = tile_y * 24 + 16  (offset down for feet)
```

The +16 Y offset (vs +12 for center) confirms entity position is at feet level.

---

## Verified Functions

### Shadow Rendering: `FUN_02058afc`

```c
// Inputs
short base_x = param_2[0];   // Entity X position
short base_y = param_2[1];   // Entity Y position
short offset_x = *(short *)(param_3 + 0x24);  // From animation_control
short offset_y = *(short *)(param_3 + 0x26);  // From animation_control
int shadow_size = GetShadowSize(monster_id);
int table_x_offset = *(int *)(DAT_02058c10 + shadow_size * 4);

// OAM X position (bits 0-8)
Final_X = base_x + offset_x + table_x_offset;

// OAM Y position (multiplied by 16 for OAM encoding)
Final_Y = (base_y + offset_y - 4) * 16;
```

**Key insight:** The `-4` Y adjustment is hardcoded for shadows, placing them slightly above the entity's logical position.

### Main Sprite Rendering: `FUN_0201c5c4`

This is the meta-frame renderer that draws WAN sprites.

```c
// animation_control structure (param_1 is ushort*)
// Offsets are array indices, multiply by 2 for byte offset

ushort base_x = param_1[0x0E];      // Offset 0x1C: Entity X
ushort base_y = param_1[0x0F];      // Offset 0x1E: Entity Y
ushort sprite_offset_x = param_1[0x10];  // Offset 0x20: SequenceFrame offset X
ushort sprite_offset_y = param_1[0x11];  // Offset 0x22: SequenceFrame offset Y

// Combined position for this frame
short draw_x = base_x + sprite_offset_x;
short draw_y = base_y + sprite_offset_y;

// Per-piece offsets from WAN MetaFramePiece (added to above)
short piece_x = *(short *)&fragment->field_0x4;
short piece_y = *(short *)&fragment->field_0x6;

// Final piece position
local_46 = piece_x + base_x + sprite_offset_x;
local_44 = piece_y + base_y + sprite_offset_y;
```

### Monster Entity Update: `FUN_02303f18`

This function updates monster positions and writes to animation_control.

```c
// Convert 8.8 fixed point to screen coordinates
short screen_x = (*(int *)(entity + 0xc) >> 8) - camera_x;
short screen_y = (*(int *)(entity + 0x10) >> 8) - camera_y;

// Write to animation_control for rendering
*(short *)(entity + 0x48) = screen_x;  // Base X for renderer
*(short *)(entity + 0x4a) = screen_y;  // Base Y for renderer

// Also updates monster->pixel_pos for other systems
monster->pixel_pos.x = attachment_offset_x + (entity_pixel_x >> 8);
monster->pixel_pos.y = attachment_offset_y + (entity_pixel_y >> 8);
```

---

## Animation Control Structure

Relevant offsets for sprite positioning:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0x1C | i16 | base_x | Entity screen X (written before render) |
| 0x1E | i16 | base_y | Entity screen Y (written before render) |
| 0x20 | i16 | sprite_offset_x | SequenceFrame offset X |
| 0x22 | i16 | sprite_offset_y | SequenceFrame offset Y |
| 0x24 | i16 | shadow_offset_x | Shadow-specific offset X |
| 0x26 | i16 | shadow_offset_y | Shadow-specific offset Y |

---

## WAN Data Flow

### SequenceFrame Structure (from WAN file)

```rust
pub struct SequenceFrame {
    pub frame_index: u16,    // Which MetaFrame to display
    pub duration: u16,       // How long (in game ticks)
    pub flag: u8,            // Animation flags
    pub is_rush_point: bool, // Combat timing flag
    pub offset: (i16, i16),  // Sprite offset from entity center
    pub shadow: (i16, i16),  // Shadow offset from entity center
}
```

The `offset` field contains the values that become `sprite_offset_x/y` in the animation_control structure.

### Typical Offset Values

From `001_atlas.json` (Bulbasaur):

| Animation | Direction | Offset X | Offset Y | Notes |
|-----------|-----------|----------|----------|-------|
| Charge | 0 (South) | 0, 1 | 0 | Alternating X for shake effect |
| Charge | 1 (SE) | 0, 1 | -1, 0 | Diagonal has Y component |
| Charge | 3 (NE) | -1, 0 | -1, 0 | Negative values shift up/left |
| Special_1 | 0 | 0 | -2 to +18 | Jump animation: up then down |

These small values confirm they're positional adjustments, not absolute coordinates.

---

## Implications for Scraper

### The Problem

Current scraper:
1. Extracts frame → pieces have `x_offset`, `y_offset` relative to entity origin (0,0)
2. Crops to content bounds (loses where 0,0 was)
3. Centers content in cell (arbitrary positioning)
4. Outputs SequenceFrame offsets unchanged

Result: The relationship between entity origin and sprite content is lost.

### The Solution

Anchor-based positioning:
1. Track where entity origin (0,0) is in each extracted frame
2. Position frames so entity origin lands at consistent **anchor point** in all cells
3. Output anchor coordinates in JSON

Client usage:
```gdscript
# Anchor is where entity origin (feet) is in every frame
sprite.offset = -anchor + (cell_size / 2)  # If sprite.centered = true

# Per-frame offsets applied via animation tracks work correctly
# because they're relative to the same origin point
```

---

## Shadow Positioning

With the anchor system, shadow positioning becomes straightforward:

```gdscript
# Shadow position relative to entity
var shadow_pos = entity_position + Vector2(
    frame_data.shadow_offset_x,
    frame_data.shadow_offset_y - 4  # ROM's fixed -4 Y adjustment
)
```

The shadow offset values in JSON are already relative to entity origin, so they work directly once the anchor system is in place.

---

## Function Reference Table

| Function | Address | Purpose |
|----------|---------|---------|
| `FUN_02058afc` | 0x02058afc | Shadow sprite renderer |
| `FUN_0201b9f8` | 0x0201b9f8 | OAM writer wrapper |
| `FUN_0200b6f0` | 0x0200b6f0 | Low-level OAM writer |
| `FUN_0201c5c4` | 0x0201c5c4 | Main meta-frame renderer |
| `FUN_0201cf5c` | 0x0201cf5c | Animation render wrapper |
| `FUN_0201cf90` | 0x0201cf90 | Attachment point calculator |
| `FUN_02303f18` | 0x02303f18 | Monster entity update |
| `FUN_022bf4f0` | 0x022bf4f0 | Effect tick function |

---

## Summary

1. **Entity position = feet/ground point** (not visual center)
2. **Draw = Entity + Offset** (simple addition)
3. **Offsets are small adjustments** (typically -2 to +20 pixels)
4. **Shadow has -4 Y adjustment** (hardcoded in ROM)
5. **Anchor system preserves coordinate relationship** for correct client rendering
