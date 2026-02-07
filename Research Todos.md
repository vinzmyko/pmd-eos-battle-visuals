# Research Todos

## Next Steps - Effect Lifecycle Investigation

### Priority 1: Looping Effect Termination (Ghidra) ✓ DOCUMENTED

**Problem:** Our client uses arbitrary 3-loop / 100-frame timeout for looping effects. ROM has explicit termination.

**Documented findings (see `Systems/effect_lifecycle.md`):**
- Non-looping effects auto-terminate via tick when animation completes
- Looping effects (anim_type 3-5 with `loop_flag=1`) run until flag cleared
- `FUN_022bde50` is explicit stop function
- Projectiles self-terminate after motion completes
- Status effects: `EndReflectClassStatus` clears logical flag only, visual effect terminated via loop flag mechanism

**Implementation approach:**
- Track looping effects per entity
- Stop effects when status ends or entity despawns
- Projectiles call explicit stop after motion

### Priority 2: Pokemon Sprite Centering ✓ DOCUMENTED

**Problem:** Pokemon sprites don't align correctly with tile positions. The scraper centers cropped content in atlas cells, but ROM offset values are relative to entity's logical position (feet/ground point). This coordinate system mismatch causes positioning errors.

**Research Findings (Verified via Ghidra):**
- ROM formula: `Draw_Position = Entity_Position + SequenceFrame_Offset`
- Entity position = feet/ground point (NOT visual center)
- Shadow position = `Entity_Position + shadow_offset - 4` (hardcoded -4 Y adjustment)
- SequenceFrame offsets are small additive values (±1-2 pixels typically)

**Solution:** Anchor-based positioning
- Track where entity origin (0,0) is in each frame
- Position frames so origin is at consistent anchor pixel in all cells
- Output anchor point in JSON for client to use
- Dynamic cell sizing to accommodate all frame extents

**Status:** Implementation complete. Pending visual verification after:
- [x] Shadow sprite extraction
- [x] Dungeon tileset extraction

**See:** `Systems/positioning_system.md` for ROM coordinate system research

### Priority 3: Applying Pokemon Centering Logic to ROM scraper and game client ✓ APPLIED IN GAME CLIENT, NEED TO CHECK IF IT LOOKS GOOD WHEN DUNGEON TILES DONE

- Verify visual output matches ROM after shadow and tileset extraction
- Test with various Pokemon sizes (Pichu, Snorlax, Wailord)
- Confirm shadow alignment works with anchor system

### Priority 4: Dungeon Tileset Extraction ✓ COMPLETE

Python implementation complete. All valid tilesets (0-143) extracted with autotile rules, palette animation, and dungeon name mappings.

**See:** `dungeon_tileset_spec.md` (canonical reference, integrates implementation findings)

**Next:** Port to Rust scraper (Phase 4 in spec).

### Priority 5: Shadow Sprite Extraction ✓ COMPLETE

Python implementation complete. 6 shadow sprites extracted (3 land + 3 water, small/medium/large).

**Key findings vs original plan:**
- File 996 is water ripples, NOT shadow OAM metadata
- Palette is RGBX 4-byte format, not BGR555
- 3 shadow sizes (no XL), tile assignments fully verified
- File 995 has u32 tile count header (50 tiles)

**See:** `shadow_extraction_plan.md` (canonical reference, integrates implementation findings)

**Next:** Port to Rust scraper.

### Priority 6: Water Ripple Extraction ✓ COMPLETE

Python implementation complete. Enemy (32×8) and ally (32×16) ripples extracted, 3 animation frames each.

**Key findings:**
- File 996 is SIR0-wrapped, contains interleaved enemy/ally frames
- 8bpp format (unlike shadows which are 4bpp)
- Ally ripple has yellow circle baked into sprite data
- Shares palette with shadows (file 997)

**See:** `water_ripples_plan.md` (canonical reference, integrates implementation findings)

**Next:** Port to Rust scraper.

### Priority 7: Refactor the plans into the architecture
Might have information in the plans not in the architecture which could be helpful will need to specify between skytemple implementation and ghidra findings etc.

### Priority 8: Layer Spawn Timing

**Question:** Do all 4 effect layers spawn simultaneously or sequentially?

**Investigation:**
- Trace call order in `ExecuteMoveEffect` and `PlayMoveAnimation`
- Look for `AdvanceFrame` calls between layer handler invocations
- Document the exact timing relationship

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

### Attachment Point Implementation
- Data is extracted and available, not yet used in client
- `effect_animation_info.attachment_point` field indicates which point to use
- ROM looks up attachment points but does NOT apply them to projectile trajectory
- **Status:** Deferred, low priority
- **NOTE**: 
    - Might need to understand how the sprites get properly translated into the sprites and then check to see if how my game client and rom scraper work similar to the rom.
    - Figure out the end points of when the project animation gets played as well.

---

## Completed Research ✓

### Effect Termination ✓ (COMPLETED)

**See:** `Systems/effect_lifecycle.md`

**Key findings:**
- Two termination mechanisms: explicit (`FUN_022bde50`) and auto-terminate via tick
- Tick checks `loop_flag` at context offset `0x3C bit 0`
- Projectiles self-terminate after motion loop
- Status effects clear loop flag, tick auto-terminates
- Global cleanup via `FUN_022bdbc8` (iterates all 32 slots)

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

### Entity Positioning ✓
- [x] Tile-to-pixel formulas: Items (+4,+4), Monsters (+12,+16)
- [x] 8.8 fixed point format
- [x] Attachment point system (4 points per frame: head, lhand, rhand, center)
- [x] Special value (99,99) = use sprite center

### Projectile Motion ✓
- [x] Speed mapping: raw values 1→2, 2→3, other→6
- [x] Frame counts: 24/speed (12, 8, or 4 frames)
- [x] Direction table (DIRECTIONS_XY) with 8 directions
- [x] Wave patterns: 0=straight, 1=vertical sine, 2=spiral
- [x] Source position = attacker pixel_pos, dest = target tile center
- [x] **Finding:** Attachment points looked up but NOT applied to trajectory

### Move Effect Pipeline ✓
- [x] 4 effect layers: 0=charge, 1=secondary, 2=primary, 3=projectile
- [x] Flag bit 3 = dual-target (both attacker and target)
- [x] Special monster animation types 98/99:
  - Type 99: Spin through 8 directions, 2 frames each (16 total)
  - Type 98: Multi-direction, 9 iterations +2 direction each (18 total)
  - Both spawn effect ONCE at start, bypass hit-frame system

### Effect Lifecycle ✓
- [x] Pool of 32 effects, 316 bytes each
- [x] Main loop: `FUN_022bf764` ticks all slots via `FUN_022bf4f0`
- [x] Auto-termination: Non-looping effects cleanup when animation complete
- [x] Looping effects (anim_type 3-5 with loop flag): Require loop_flag clear or explicit stop
- [x] 100-frame timeout is safety fallback, not intended mechanism

### Data Structures ✓
- [x] `effect_animation_info` (28 bytes): All fields documented including directional logic
- [x] `move_animation_info` (24 bytes): All fields, per-Pokemon overrides
- [x] `animation_control`: Bitfield meanings (bits 12=loop, 13=complete, 15=active)
- [x] `effect_context` (316 bytes): Key offsets for trajectory, animation state

### Sprite Positioning System ✓
- [x] Core formula: `Draw = Entity + SequenceFrame_Offset`
- [x] Entity position = feet/ground point
- [x] Shadow Y adjustment: hardcoded -4
- [x] SequenceFrame offsets are small animation deltas (±1-2 pixels)
- [x] Meta-frame renderer combines base position with offsets

### Dungeon Tileset Extraction ✓
- [x] Python implementation complete (all tilesets 0-143)
- [x] 47-tile blob pattern documented
- [x] Per-color palette animation system documented
- [x] Debug tilesets 144-169 identified and skipped
- [x] Tileset 170 redirect handled
- [x] Autotile rules (256 configs × 3 variations) exported to JSON
- **See:** `dungeon_tileset_spec.md`

### Shadow Sprite Extraction ✓
- [x] Python implementation complete (6 sprites: 3 land + 3 water)
- [x] File 995: Raw 4bpp with u32 tile count header (50 tiles)
- [x] File 997: RGBX palette (shared with ripples)
- [x] 3 sizes (small/medium/large), no XL
- [x] Verified tile assignments for all 6 sprites
- **See:** `shadow_extraction_plan.md`

### Water Ripple Extraction ✓
- [x] Python implementation complete (enemy + ally, 3 frames each)
- [x] File 996: SIR0-wrapped, 8bpp, 2304 bytes content
- [x] Enemy: 32×8 cyan/white rings, Ally: 32×16 with yellow circle
- [x] Shares palette with shadows (file 997)
- **See:** `water_ripples_plan.md`

---

## Reference

### Key File Paths (Rust Scraper)
- `src/effect_sprite_extractor.rs` - Main extraction pipeline
- `src/move_effects_index.rs` - JSON schema definitions
- `src/graphics/wan/parser.rs` - WAN parsing (has `max_sequences_per_group`)
- `src/graphics/wan/renderer.rs` - Sprite sheet rendering
- `src/graphics/atlas/analyser.rs` - Frame analysis and reference point calculation
- `src/graphics/atlas/generator.rs` - Atlas layout and frame positioning
- `src/graphics/atlas/metadata.rs` - JSON metadata generation

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

### Shadow & Ripple System Reference
| Component | Location | Notes |
|-----------|----------|-------|
| Shadow texture | dungeon.bin[995] | Raw 4bpp, u32 header, 50 tiles |
| Water ripples | dungeon.bin[996] | SIR0 wrapped, 8bpp, 3 frames |
| Shared palette | dungeon.bin[997] | RGBX format, 16 colors × 2 palettes |
| Shadow size/monster | monster.md + 0x2E | Byte value within 0x44-byte entries |
| OAM attributes | DAT_02058c0c | 0x10 bytes per size |
| X offsets | DAT_02058c10 | 4 bytes per size |
| Water remap | DAT_02304c30 | land_size → water_size |

### Research Documentation Files
| File | Description |
|------|-------------|
| `Systems/effect_lifecycle.md` | How looping effects are stopped in ROM |
| `Systems/positioning_system.md` | ROM sprite coordinate system (verified functions) |
| `shadow_extraction_plan.md` | Shadow sprite extraction (verified, Python complete) |
| `water_ripples_plan.md` | Water ripple extraction (verified, Python complete) |
| `dungeon_tileset_spec.md` | Dungeon tileset extraction (verified, Python complete) |
