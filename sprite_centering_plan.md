# Pokemon Sprite Centering Plan

## Overview

Pokemon sprites need proper centering so that when rendered in the game client, entities align correctly with their tile positions. Currently, there's a coordinate system mismatch between how the ROM scraper outputs sprite atlases and how the game client needs to position them.

## Project Goal Context

The ROM scraper's purpose is to extract visual game assets in a format that makes client-side rendering straightforward. The client should be able to load assets and position entities on tiles with minimal complexity. This plan ensures sprite positioning "just works" for the client developer.

---

## The Current Problem

### What's Happening

1. **Scraper crops frames to content bounds** - Each frame is cropped to remove empty space
2. **Scraper centers cropped content in atlas cells** - Content is placed at the center of fixed-size cells
3. **ROM offset values are passed through unchanged** - `offset_x`, `offset_y` from WAN files go to JSON
4. **Client applies offsets to centered sprites** - But these offsets were designed for a different coordinate system

### Why This Fails

The ROM uses a coordinate system where:
- Entity has a logical position (where its feet/ground point is)
- Sprite offset is relative to that position: `draw_position = entity_position + sprite_offset`

The scraper breaks this by centering content, which:
- Loses the relationship between entity position and sprite content
- Makes the offset values meaningless in the new coordinate system
- Causes different-sized Pokemon to render incorrectly relative to tiles

### Visual Example

```
ROM System:                          Current Scraper:
┌─────────────┐                      ┌─────────────┐
│             │                      │   ┌───┐     │
│    ┌───┐    │                      │   │ P │     │
│    │ P │    │  offset=(0,-8)       │   └───┘     │
│    └───┘    │  means "draw 8px     │             │
│      ●      │  above feet"         │             │
│   (feet)    │                      │   (centered)│
└─────────────┘                      └─────────────┘
                                     
Entity at tile center,               Content centered in cell,
sprite positioned by offset          offset value has no meaning
```

### Code References

**Scraper - Content Centering** (`src/graphics/atlas/generator.rs`):
```rust
// Current: centers content in cell (LOSES POSITIONING INFO)
let final_pos_x = (frame_width as i32 - content_width as i32) / 2;
let final_pos_y = (frame_height as i32 - content_height as i32) / 2;
```

**Scraper - Offset Passthrough** (`src/graphics/atlas/metadata.rs`):
```rust
// Original WAN offsets passed through but meaningless after centering
offset_x: original_seq_frame.offset.0 as i32,
offset_y: original_seq_frame.offset.1 as i32,
```

**Client - Centered Sprite** (`pokemon.gd`):
```gdscript
sprite.centered = true  # Center of sprite at node position
# Applying offset to centered sprite doesn't match ROM behavior
```

---

## Research Findings (VERIFIED)

### ROM Coordinate System ✓

**Core Formula:**
```
Draw_Position = Entity_Position + SequenceFrame_Offset
```

This was verified by analyzing `FUN_0201c5c4` (meta-frame renderer):
```c
// animation_control offsets (ushort array indices)
short draw_x = param_1[0x0E] + param_1[0x10];  // base_x + offset_x
short draw_y = param_1[0x0F] + param_1[0x11];  // base_y + offset_y
```

### Entity Position = Feet/Ground Point ✓

The entity's `pixel_pos` represents where the monster "stands" - its feet touch this point.

**Evidence:**
1. Tile formula: `Monster_Pixel_Y = tile_y * 24 + 16` (offset down for feet, not +12 for center)
2. Shadow renders at `entity_pos - 4` (places shadow above feet)
3. SequenceFrame offsets are small (±1-2 px typically), not large positioning values

### Shadow Formula ✓

From `FUN_02058afc`:
```c
Shadow_X = base_x + offset_x + table_x_offset;
Shadow_Y = base_y + offset_y - 4;  // Hardcoded -4 adjustment
```

The `-4` places the shadow slightly above the feet position.

### SequenceFrame Offsets Are Additive ✓

The offset values create animation effects (bobbing, shaking, jumping) by shifting the sprite relative to entity position:

| Animation | Typical Offsets | Effect |
|-----------|-----------------|--------|
| Standing | (0, 0) | Neutral |
| Charge | (0, 0), (1, 0) alternating | Shake effect |
| Jump | (0, -2) → (0, -18) → (0, +2) | Arc motion |
| Attack | (0, 0) → (5, -2) → (0, 0) | Lunge forward |

**See:** `positioning_system.md` for full function analysis and verification details.

---

## Proposed Solution: Anchor-Based Positioning

### Concept

Instead of centering content, position frames so the entity's logical anchor point (ground/feet) is at a **consistent pixel location** in every atlas cell. Output this anchor position in the JSON so the client can align it with tile positions.

### Scraper Changes

**1. Track Entity Origin** (`src/graphics/atlas/analyser.rs`)

When extracting frames, calculate where the entity's logical origin (0,0) is relative to the cropped content:

```rust
pub struct AnalysedFrame {
    // ... existing fields ...
    
    /// Entity origin position relative to cropped content
    pub entity_origin_x: i32,
    pub entity_origin_y: i32,
}
```

**2. Unified Canvas with Anchor** (`src/graphics/atlas/generator.rs`)

Define a consistent anchor position in the atlas cell, then position content so entity origin aligns with anchor:

```rust
// Define anchor point (entity feet position in cell)
let anchor_x = frame_width as i32 / 2;
let anchor_y = (frame_height as f32 * ANCHOR_Y_RATIO) as i32;  // TBD from research

// Position content so entity origin aligns with anchor
let final_pos_x = anchor_x - analysed_frame.entity_origin_x;
let final_pos_y = anchor_y - analysed_frame.entity_origin_y;
```

**3. Output Anchor in JSON** (`src/graphics/atlas/metadata.rs`)

```rust
pub struct AtlasMetadata {
    // ... existing fields ...
    pub anchor_x: i32,  // Entity ground point X in cell
    pub anchor_y: i32,  // Entity ground point Y in cell
}
```

**4. Dynamic Cell Sizing**

Calculate cell size based on maximum extent from entity origin across all frames, ensuring no content is clipped:

```rust
fn calculate_required_cell_size(frames: &[AnalysedFrame]) -> (u32, u32) {
    let mut max_left = 0i32;
    let mut max_right = 0i32;
    let mut max_up = 0i32;
    let mut max_down = 0i32;
    
    for frame in frames {
        max_left = max_left.max(frame.entity_origin_x);
        max_right = max_right.max(frame.content_width as i32 - frame.entity_origin_x);
        max_up = max_up.max(frame.entity_origin_y);
        max_down = max_down.max(frame.content_height as i32 - frame.entity_origin_y);
    }
    
    let width = (max_left + max_right) as u32;
    let height = (max_up + max_down) as u32;
    
    // Round up to nice sizes for atlas packing
    (round_up_to_8(width), round_up_to_8(height))
}
```

### Client Changes

**1. Load Anchor Point** (`asset_converter.gd`)

```gdscript
pokemon_data.anchor_x = json_data.get("anchor_x", frame_width / 2)
pokemon_data.anchor_y = json_data.get("anchor_y", frame_height * 3 / 4)
```

**2. Apply Anchor Offset** (`pokemon.gd`)

```gdscript
func _setup_sprite():
    sprite.centered = false  # Use top-left positioning
    
    # Offset sprite so anchor point aligns with node position
    sprite.offset = Vector2(
        -pokemon_data.anchor_x,
        -pokemon_data.anchor_y
    )
```

Or alternatively, keep centered and adjust:

```gdscript
sprite.centered = true
sprite.offset = Vector2(
    pokemon_data.frame_width / 2.0 - pokemon_data.anchor_x,
    pokemon_data.frame_height / 2.0 - pokemon_data.anchor_y
)
```

**3. Shadow Positioning**

With the anchor system, shadows become trivial:

```gdscript
func get_shadow_position() -> Vector2:
    # Shadow is relative to entity origin, which is now at global_position
    var frame_data = get_current_frame_data()
    return global_position + Vector2(frame_data.shadow_offset_x, frame_data.shadow_offset_y)
```

### JSON Output Format Change

**Current format:**
```json
{
  "frame_width": 40,
  "frame_height": 40,
  "animations": { ... }
}
```

**New format:**
```json
{
  "frame_width": 48,
  "frame_height": 56,
  "anchor_x": 24,
  "anchor_y": 48,
  "animations": { ... }
}
```

The `anchor_x`, `anchor_y` tell the client: "In every frame, the Pokemon's feet/ground point is at this pixel coordinate."

---

## Implementation Steps

### Phase 1: Research ✓ COMPLETE

- [x] Find monster sprite rendering function (`FUN_0201c5c4`)
- [x] Document coordinate system: `Draw = Entity + Offset`
- [x] Verify entity position = feet/ground point
- [x] Confirm per-frame offsets are additive
- [x] Document shadow positioning (`entity + offset - 4`)

**See:** `positioning_system.md` for detailed function analysis

### Phase 2: Scraper Changes

- [ ] Modify `AnalysedFrame` to track entity origin
- [ ] Calculate entity origin from frame bounds and WAN offsets
- [ ] Implement dynamic cell sizing based on max extents
- [ ] Modify `prepare_frames()` to use anchor positioning
- [ ] Add `anchor_x`, `anchor_y` to JSON output

### Phase 3: Client Changes

- [ ] Update `asset_converter.gd` to parse anchor point
- [ ] Modify `pokemon.gd` to apply anchor offset
- [ ] Test with various Pokemon sizes (Pichu, Pikachu, Snorlax, Wailord)
- [ ] Verify shadow alignment works with new system

### Phase 4: Validation

- [ ] Compare visual output with ROM/emulator screenshots
- [ ] Test animations (walk, attack, etc.) maintain proper positioning
- [ ] Verify different-sized Pokemon align correctly on same tile

---

## Benefits of This Approach

1. **Client simplicity** - Just load anchor, set offset, done
2. **Automatic size handling** - Large Pokemon get larger cells, no special cases
3. **Shadow alignment** - Same anchor system works for shadows
4. **Animation consistency** - All frames use same reference point
5. **Future-proof** - Effect attachment points can use same system

## Files to Modify

### Scraper
- `src/graphics/atlas/analyser.rs` - Add entity origin tracking
- `src/graphics/atlas/generator.rs` - Anchor-based positioning, dynamic sizing
- `src/graphics/atlas/metadata.rs` - Add anchor to JSON output
- `src/pokemon_sprite_extractor.rs` - Pass through new metadata

### Client
- `client/autoloads/asset_converter.gd` - Parse anchor point
- `client/pokemon/pokemon.gd` - Apply anchor offset
- `client/resources/frame_data.gd` - (possibly) add anchor fields to resource

---

## Open Questions (Resolved)

1. **What is ANCHOR_Y_RATIO?**
   - **RESOLVED:** Entity position = feet. Anchor should be near bottom of cell.
   - Recommendation: Calculate dynamically based on frame content bounds, placing anchor where entity origin (0,0) would be.

2. **Per-frame vs per-Pokemon anchor?**
   - **RESOLVED:** Use single anchor per-Pokemon (per atlas). The entity origin is consistent across all frames - only the SequenceFrame offsets change per-frame to create animation effects.

3. **Attachment points** - Should these be recalculated relative to anchor?
   - **NO:** Attachment points are already relative to entity origin in the WAN data. Once anchor system is in place, they'll work correctly without modification.

---

## References

- WAN format: `src/graphics/wan/model.rs` - SequenceFrame, MetaFramePiece structures
- Current atlas generation: `src/graphics/atlas/` - analyser.rs, generator.rs, metadata.rs
- Entity positioning research: `Research_Todos_Updated.md` - Tile-to-pixel formulas
- Shadow system: `shadow_extraction_plan.md` - Shadow offset is -4 from entity Y
- **ROM positioning system: `positioning_system.md` - Verified coordinate system and function analysis**
