# Research Todos

## Next Steps - Effect Lifecycle Investigation

### Priority 1: Looping Effect Termination (Ghidra) ✓ DOCUMENTED

**Problem:** Our client uses arbitrary 3-loop / 100-frame timeout for looping effects. ROM has explicit termination.

**Documented findings (see `effect_termination.md`):**
- Non-looping effects auto-terminate via tick when animation completes
- Looping effects (anim_type 3-5 with `loop_flag=1`) run until flag cleared
- `FUN_022bde50` is explicit stop function
- Projectiles self-terminate after motion completes
- Status effects: `EndReflectClassStatus` clears logical flag only, visual effect terminated via loop flag mechanism

**Implementation approach:**
- Track looping effects per entity
- Stop effects when status ends or entity despawns
- Projectiles call explicit stop after motion

### Priority 2: Pokemon Sprite Centering ✓ RESEARCH COMPLETE

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

**Implementation Status:** Ready for implementation

**See:** 
- `positioning_system.md` for ROM coordinate system research
- `sprite_centering_plan.md` for implementation plan

### Priority 3: Applying Pokemon Centering Logic to ROM scraper and game client

- Look at `sprite_centering_plan.md`

### Priority 4: Dungeon tileset extraction via python

- Look at `dungeon_tileset_spec.md`

### Priority 5: Shadow Sprite Extraction

**Goal:** Extract shadow sprites from ROM instead of providing static assets.

**Location:** `DUNGEON/dungeon.bin`
- File 995 = `DBIN_RAW_IMAGE_4BPP` (shadow texture, raw 4bpp tiles)
- File 996 = `SIR0` (unknown, possibly OAM metadata)
- File 997 = `DPL` (shadow palette, 16 colors)

**Shadow system (from Ghidra):**
- Shadow size per monster at `monster.md + 0x2E` (within 0x44-byte entries)
- Land vs water shadows (water uses remapped index via `DAT_02304c30`)
- OAM attributes at `DAT_02058c0c` (0x10 bytes per size, land+water variants)

**See:** `shadow_extraction_plan.md` for full implementation details

### Priority 6: Layer Spawn Timing

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

---

## Completed Research ✓

### Effect Termination ✓ (COMPLETED)

**See:** `effect_termination.md`

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

### Shadow System Reference
| Component | Location | Notes |
|-----------|----------|-------|
| Shadow texture | dungeon.bin[995] | Raw 4bpp tiles |
| Shadow metadata | dungeon.bin[996] | SIR0 wrapped, unknown structure |
| Shadow palette | dungeon.bin[997] | DPL format, 16 colors |
| Shadow size/monster | monster.md + 0x2E | Byte value per monster |
| OAM attributes | DAT_02058c0c | 0x10 bytes per size |
| X offsets | DAT_02058c10 | 4 bytes per size |
| Water remap | DAT_02304c30 | land_size → water_size |

### Research Documentation Files
| File | Description |
|------|-------------|
| `effect_termination.md` | How looping effects are stopped in ROM |
| `shadow_extraction_plan.md` | Plan for extracting shadow sprites from dungeon.bin |
| `sprite_centering_plan.md` | Plan for fixing Pokemon sprite positioning/centering |
| `positioning_system.md` | ROM sprite coordinate system research (verified functions) |
