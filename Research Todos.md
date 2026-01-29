# Research Todos

## Next Steps - Client Implementation

**Task:** Update Godot client to use directional effect sprites.

### Files to Modify

1. **`move_effect_player.gd`** - Load correct sprite sheet based on direction
   ```gdscript
   func _load_sprite_sheet(effect_data: Dictionary, direction: int) -> Texture2D:
       if effect_data.get("is_directional", false):
           # Directional: append _dir{N}.png to base path
           var base_path = effect_data["sprite_sheet"]
           var sheet_path = "%s_dir%d.png" % [base_path, direction]
           return load(sheet_path)
       else:
           # Non-directional: use path directly
           return load(effect_data["sprite_sheet"])
   ```

2. **`animation_sequencer.gd`** - Pass attacker's `current_direction` to effect player

3. **`_spawn_projectile_effect()`** - Use attacker direction for projectile sprites

### Direction Mapping (ROM → Godot)
| ROM Dir | Name | Rotation |
|---------|------|----------|
| 0 | South (Down) | 0° |
| 1 | Southwest | Counter-clockwise from South |
| 2 | West (Left) | 90° CCW |
| 3 | Northwest | ... |
| 4 | North (Up) | 180° |
| 5 | Northeast | ... |
| 6 | East (Right) | 270° CCW |
| 7 | Southeast | ... |

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

**Problem Solved:** Effects now correctly render directional variants.

**Implementation:**
- Scraper detects directionality via `sequence_count % 8 == 0`
- Directional effects output 8 files: `{effect_id}_dir{0-7}.png`
- Non-directional effects output 1 file: `{effect_id}.png`
- JSON includes `is_directional`, `direction_count`, `base_animation_index`

**ROM Behavior (from FUN_022bdfc0):**
```
If (sequence_count % 8 == 0) → Directional effect
   - Direction (0-7) is ADDED to base animation_index
   - Example: Effect 333 (Water Gun projectile) base_index=48, direction 3 → sequence 51
   
If (sequence_count % 8 != 0) → Non-directional effect
   - Same animation regardless of direction
   - Direction only affects projectile velocity
```

**Verified Example - Water Gun (Move 308):**
| Layer | Effect ID | Type | File Index | Anim Index | Notes |
|-------|-----------|------|------------|------------|-------|
| Secondary | 21 | WanFile0 | 0 | 17 | Reuse (attacker anim) |
| Primary | 334 | WanFile0 | 0 | 45 | Reuse (attacker anim) |
| **Projectile** | **333** | **WanOther** | **13** | **48** | **Actual projectile - DIRECTIONAL** |

**Files Modified:**
- `src/effect_sprite_extractor.rs` - Added `check_directional_effect()`, renders 8 sheets for directional
- `src/move_effects_index.rs` - Added `base_animation_index` field to `SpriteEffect`

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
- [x] Auto-termination: Non-looping effects cleanup when `animation_control` bit 13 set
- [x] Looping effects (anim_type 3-5 with loop flag): Never auto-terminate, require explicit `FUN_022bde50` call
- [x] 100-frame timeout is safety fallback, not intended mechanism

### Data Structures ✓
- [x] `effect_animation_info` (28 bytes): All fields documented including directional logic
- [x] `move_animation_info` (24 bytes): All fields, per-Pokemon overrides
- [x] `animation_control`: Bitfield meanings (bits 12=loop, 13=complete, 15=active)
- [x] `effect_context` (316 bytes): Key offsets for trajectory, animation state

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
