# Research Todos

## Critical - Directional Effects Not Working

**Problem:** All effects currently play the same animation regardless of attacker/target direction.

**Root Cause:** The Rust scraper hardcodes `is_directional = false` and only exports one animation sequence per effect.

**ROM Behavior (from reverse engineering):**
```
If (sequence_count % 8 == 0) → Effect IS directional
   - Direction (0-7) is ADDED to animation_index to select correct sequence
   - Example: Shadow Sneak has 16 sequences, base index 8 → direction 3 plays sequence 11
   
If (sequence_count % 8 != 0) → Effect is NOT directional
   - Direction only affects projectile velocity, same animation plays for all directions
   - Example: Water Gun has 50 sequences, 50%8≠0, always plays same sequence
```

**Fix Required (3 layers):**

### 1. Rust Scraper (`effect_sprite_extractor.rs`)
- [ ] Detect directional effects: `sequence_count % 8 == 0`
- [ ] For directional effects, export 8 separate sprite sheets: `{effect_id}_dir{0-7}.png`
- [ ] Add `base_animation_index` to JSON output
- [ ] Update `is_directional` and `direction_count` fields correctly

### 2. JSON Schema (`move_effects_index.rs`)
- [ ] Add `base_animation_index: u32` field to `SpriteEffect` struct

### 3. Client (Godot)
- [ ] `animation_sequencer.gd`: Pass attacker's `current_direction` to `setup_and_play()`
- [ ] `move_effect_player.gd`: Accept direction parameter, load correct sprite sheet (`_dir{N}.png`)
- [ ] `_spawn_projectile_effect()`: Calculate direction from attacker→target vector

---

## Known Issues - Deferred

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

### Entity Positioning
- [x] Tile-to-pixel formulas: Items (+4,+4), Monsters (+12,+16)
- [x] 8.8 fixed point format
- [x] Attachment point system (4 points per frame: head, lhand, rhand, center)
- [x] Special value (99,99) = use sprite center

### Projectile Motion
- [x] Speed mapping: raw values 1→2, 2→3, other→6
- [x] Frame counts: 24/speed (12, 8, or 4 frames)
- [x] Direction table (DIRECTIONS_XY) with 8 directions
- [x] Wave patterns: 0=straight, 1=vertical sine, 2=spiral
- [x] Source position = attacker pixel_pos, dest = target tile center
- [x] **Finding:** Attachment points looked up but NOT applied to trajectory

### Move Effect Pipeline
- [x] 4 effect layers: 0=charge, 1=secondary, 2=primary, 3=projectile
- [x] Flag bit 3 = dual-target (both attacker and target)
- [x] Special monster animation types 98/99:
  - Type 99: Spin through 8 directions, 2 frames each (16 total)
  - Type 98: Multi-direction, 9 iterations +2 direction each (18 total)
  - Both spawn effect ONCE at start, bypass hit-frame system

### Effect Lifecycle
- [x] Pool of 32 effects, 316 bytes each
- [x] Main loop: `FUN_022bf764` ticks all slots via `FUN_022bf4f0`
- [x] Auto-termination: Non-looping effects cleanup when `animation_control` bit 13 set
- [x] Looping effects (anim_type 3-5 with loop flag): Never auto-terminate, require explicit `FUN_022bde50` call
- [x] 100-frame timeout is safety fallback, not intended mechanism

### Data Structures
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

### Key File Paths (Godot Client)
- `client/game/battle/combat/animation_sequencer.gd` - Coordinates attack animations
- `client/game/battle/combat/move_effect_player.gd` - Plays individual effects
- `client/game/battle/combat/move_execution_context.gd` - Holds move data
- `client/pokemon/pokemon.gd` - Has `current_direction` and `ATLAS_DIR` mapping
