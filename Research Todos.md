# Research Todos

## Next Steps - Effect Lifecycle Investigation

### Priority 1: Looping Effect Termination (Ghidra)

**Problem:** Our client uses arbitrary 3-loop / 100-frame timeout for looping effects. ROM has explicit termination.

**What we know:**
- Looping effects (anim_type 3-5 with `loop_flag=1`) never auto-terminate
- They require explicit `FUN_022bde50(instance_id)` call
- Tick function `FUN_022bf4f0` skips cleanup when loop flag is set

**Investigation needed:**
1. Find all XREFs to `FUN_022bde50` (0x022bde50) - which callers are NOT user-triggered?
2. In `ExecuteMoveEffect` - after damage application, is there effect cleanup code?
3. Look for patterns: `if (effect_handle >= 0) { FUN_022bde50(effect_handle); }`

**Hypothesis:** Looping effects are stopped when:
- Move execution completes (damage applied)
- Entity dies/despawns
- New action starts on same entity
- Move is interrupted

### Priority 2: Layer Spawn Timing

**Question:** Do all 4 effect layers spawn simultaneously or sequentially?

**Investigation:**
- Trace call order in `ExecuteMoveEffect` and `PlayMoveAnimation`
- Look for `AdvanceFrame` calls between layer handler invocations
- Document the exact timing relationship

### Priority 3: Attachment Point Implementation (Deferred)

**Status:** Data is extracted and available, not yet used in client.

**Current state:**
- Scraper extracts attachment points per frame (head, lhand, rhand, center)
- `effect_animation_info.attachment_point` field indicates which point to use
- ROM looks up attachment points but does NOT apply them to projectile trajectory

**To implement:**
- Effect spawns at entity position + attachment point offset
- Need to pass attachment point index through to effect player
- Get offset from current animation frame's attachment data

---

## Known Issues - Deferred

### Identical Directional Sprites
Some effects detected as directional (sequence_count % 8 == 0) have 8 identical sprite files.
- **Likely cause:** Intentionally symmetric effects (explosions, circles) where artist duplicated frames
- **Impact:** Minor - wastes disk space but renders correctly
- **Future optimization:** Could add deduplication to detect and collapse identical _dir files

### Large/Vertical Effects (rock_slide, thunder_shock)
- Render too large or positioned incorrectly
- **Likely causes:**
  - WAN sprite scale/dimensions not being applied
  - Type 5/6 screen effects use different rendering
  - Unusual attachment point values
- **Status:** Deferred pending WAN format research

### Type 5/6 Screen Effects
- Type 5 uses `file_index + 268 (0x10C)` for actual file lookup
- Need to understand screen overlay rendering system
- **Status:** Not implemented, returns placeholder `ScreenEffect`

---

## Investigation Notes (Ghidra)

### FUN_022bde50 - Effect Stop Function
- **Address (NA):** 0x022bde50
- **Purpose:** Explicitly stops an effect by instance_id
- **Called by:** Need to find all XREFs

### Key Questions to Answer
1. What triggers looping effect cleanup in normal move execution?
2. Is `delay_counter` (context + 0x18) ever set to non-zero? Where?
3. Are there entity cleanup routines that iterate active effects?

### Addresses to Investigate
| Function | Address | Purpose |
|----------|---------|---------|
| `FUN_022bde50` | 0x022bde50 | Stop effect by instance_id |
| `FUN_022bf4f0` | 0x022bf4f0 | Per-effect tick |
| `FUN_022bdfc0` | 0x022bdfc0 | Effect initialization |
| `ExecuteMoveEffect` | ? | Main move execution |

---

## WAN Format Gaps

### Effect WAN vs Character WAN
- [ ] Effect WAN metaframe format (10-byte structure, different from CHARA)
- [ ] Palette offset handling (`Unk#5` field in PaletteInfo)
- [ ] 8bpp vs 4bpp image data organization
- [ ] Sequence organization in effect files

### From SkyTemple/Project Pokemon Docs
```
Effect Metaframe Structure (10 bytes):
- Section 0-2: Flags (FFFF, 00, draw-behind)
- Section 3-4: Y offset + dimensions
- Section 5-6: X offset + dimensions (size, flip, is_last)
- Section 7: Image offset by blocks
- Section 8: Palette index (0-C external, D-E custom)
- Section 9: Always 0xC
```

---

## Completed Research ✓

### Directional Effects ✓ (COMPLETED)

**Problem Solved:** Effects now correctly render directional variants with consistent frame sizes.

**Scraper Implementation:**
- Detects directionality via `sequence_count % 8 == 0`
- **Two-pass rendering for directional effects:**
  1. First pass: `calculate_unified_canvas_box()` finds max bounds across all 8 directions
  2. Second pass: Render all 8 directions using unified dimensions
- Directional effects output 8 files: `{effect_id}_dir{0-7}.png` (all same dimensions)
- Non-directional effects output 1 file: `{effect_id}.png`
- JSON includes `is_directional`, `direction_count`, `base_animation_index`

**Client Implementation:**
- `move_resolver.gd` - Passes `is_directional` flag, builds base path without .png for directional
- `animation_sequencer.gd` - Gets attacker direction via `attacker.get_current_direction()`, passes to effect player
- `move_effect_player.gd` - `_get_sprite_sheet_path()` appends `_dir{N}.png` for directional effects
- `pokemon_entity.gd` - `get_current_direction()` exposes ROM-format direction (0-7)

**Files Modified (Scraper):**
- `src/effect_sprite_extractor.rs` - Added `calculate_unified_canvas_box()`, two-pass `process_directional_effect()`
- `src/graphics/wan/renderer.rs` - Added `get_effect_animation_canvas_box()`, `render_effect_animation_sheet_with_canvas()`

**Files Modified (Client):**
- `client/autoloads/move_resolver.gd`
- `client/game/battle/combat/animation_sequencer.gd`
- `client/game/battle/combat/move_effect_player.gd`
- `client/game/battle/common/pokemon_entity.gd`

---

## Reference

### Key File Paths (Rust Scraper)
- `src/effect_sprite_extractor.rs` - Main extraction pipeline (UPDATED)
- `src/move_effects_index.rs` - JSON schema definitions (UPDATED)
- `src/graphics/wan/parser.rs` - WAN parsing (has `max_sequences_per_group`)
- `src/graphics/wan/renderer.rs` - Sprite sheet rendering

### Key File Paths (Godot Client)
- `client/game/battle/combat/animation_sequencer.gd` - Coordinates attack animations
- `client/game/battle/combat/move_effect_player.gd` - Plays individual effects
- `client/game/battle/combat/move_execution_context.gd` - Holds move data
- `client/pokemon/pokemon.gd` - Has `current_direction` and `ATLAS_DIR` mapping

### Effect Type Reference
| anim_type | Name | Handling |
|-----------|------|----------|
| 0 | Invalid | Skip |
| 1 | WanFile0 | Reuse attacker animation (requires state == 0) |
| 2 | WanFile1 | Reuse attacker animation (requires state == 1) |
| 3 | WanOther | **Standard effect sprite** - extracted by scraper |
| 4 | Wat | WAT format - not implemented |
| 5 | Screen | Screen effect (file_index + 0x10C) - placeholder |
| 6 | Wba | WBA format - not implemented |
