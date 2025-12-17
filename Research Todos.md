# Research Todos

## Status Overview

| Area | Status | Notes |
|------|--------|-------|
| Entity Positioning | Complete | Items +4,+4; Monsters +12,+16; Attachment points |
| Projectile Motion | Complete | Speed mapping, direction tables, straight line implemented |
| Projectile Spawn System | Complete | Source/dest positions, attachment points not used for trajectory |
| Animation Timing | Complete | AdvanceFrame, frame loops, task scheduler |
| Move Effect Pipeline | Complete | Layers, handlers, flag bits 3 |
| Special Monster Animations | Complete | Types 98/99 (spin, multi-direction) |
| effect_animation_info | Complete | All fields documented |
| move_animation_info | Complete | All fields documented |
| Looping Effect Termination | Not Started | Current implementation uses arbitrary timeout |
| Large/Vertical Effects | Not Started | rock_slide, thunder_shock positioning broken |
| Directional WAN Effects | Not Started | Effects don't respect attacker direction |
| Flag Bits 4 & 6 | Not Started | skip_fade_in, add_delay |
| Screen Effects (Type 5/6) | Not Started | Low priority |
| Sound System | Not Started | Low priority |

---

## Active Research Needed

### Looping Effect Termination (High Priority)

**Problem:** Current implementation uses arbitrary timeout (`total_duration * 3.0`), doesn't match game behavior.

**Questions:**
- What calls `FUN_022bde50` (effect stop) to terminate looping effects?
- How does `is_non_blocking` field affect effect lifetime?
- Is there a signal when parent attack completes?
- Does `loop_flag` have a max iteration count?

**Where to Look:**
- Trace what happens AFTER hit frame
- Effect dispatch functions (`FUN_022bfc5c`, `FUN_022bed90`)
- Effect cleanup/stop function (`FUN_022bde50`)

---

### Large/Vertical Effects (High Priority)

**Problem:** Effects like rock_slide, thunder_shock render too large or positioned wrong.

**Questions:**
- How are screen-space effects positioned?
- Does `animation_category` (flag bits 0-2) control coordinate system?
- Are some effects relative to screen center vs tile position?
- Is there a scale factor or different rendering path?

**Where to Look:**
- Effect positioning in `FUN_022beb2c`
- Check if `animation_category` values (0, 2 observed) affect positioning
- Compare positioning for small vs large effects

---

### Directional WAN Effects (Medium Priority - Separate Research)

**Problem:** Effects always show same direction regardless of attacker facing.

**Questions:**
- How does the game select WAN animation based on direction (0-7)?
- Is `is_pokemon_sprite_directional` field used for this?
- Where is direction passed to effect renderer?

**Note:** This involves WAN file format and likely scraper changes. Research separately from battle visual pipeline.

---

### Flag Bits 4 & 6 (Low Priority)

**flag_bit4 (skip_fade_in):**
- Where is mask `0x10` checked?
- What fade-in is being skipped?

**flag_bit6 (add_delay):**
- Where is mask `0x40` applied?
- What duration is the delay?
- When does delay occur (before/after effect)?

---

## Open Questions (Low Priority)

### Flag Bits
- [ ] Flag bits 0-2 (animation_category) - values 0-7 observed, purpose unknown
- [ ] Flag bit 5 - never observed set, purpose unknown
- [ ] Flag bit 7 - never observed set, purpose unknown

### Effect System
- [ ] Exact difference between dispatch type 1 and type 2 (both WAN, differ by state check)
- [ ] Complete effect_context structure (many offsets still unknown)
- [ ] Screen effect control structure at context + 0xE8

### Timing
- [ ] What param_2 in FUN_022ea370 controls (passed through but unclear)
- [ ] Layer 0 timing relationship to two-turn move charging

---

## Not Planned

### Screen Effects (Type 5/6)
- Rendering method unclear
- Files at file_index + 0x10C
- Not critical for basic move recreation

### Sound System
- sfx_id mapping to actual sound files
- Sound archive format
- Not visual, lower priority

### Wave Patterns (Sine/Spiral)
- Pattern 0 (straight line) implemented
- Patterns 1/2 determined at runtime by complex logic
- Can add later if specific moves look wrong

---

## Completed Research

### Entity Positioning
- Tile-to-pixel formula for items (+4, +4)
- Tile-to-pixel formula for monsters (+12, +16)
- 8.8 fixed point format confirmed
- Attachment point calculation (FUN_0201cf90)
- WAN offset structure (16 bytes per frame, 4 points)
- Special case (99, 99) = center

### Projectile Motion
- Speed mapping (1→2, 2→3, other→6)
- Frame count calculation (24 / mapped_speed)
- Direction table (DIRECTIONS_XY) confirmed
- Wave patterns documented (implementing pattern 0 only)
- Amplitude based on range
- Reverse direction (+4 & 7)

### Projectile Spawn System
- Source position = attacker pixel position (not attachment point)
- Destination position = target tile center (tile * 24 + 12/16)
- Attachment points looked up but NOT applied to trajectory
- Wave pattern determined at runtime, use 0 (straight) as default
- Effect context storage (offsets 0x128-0x134)

### Animation Timing
- AdvanceFrame yields one frame
- FUN_022ea370 loops n times
- Task scheduler structure (FUN_022ddef8)
- Blocking loop pattern (AnimationHasMoreFrames)
- Safety timeout (100 frames in some cases)

### Move Effect Pipeline
- ExecuteMoveEffect as entry point
- Flag bit 3 = dual-target (FUN_023258ec) - **Implemented**
- Layer 0 handler (FUN_022bfaa8) - charge/prep - **Implemented (spawns at attack start)**
- Layer 1 handler (FUN_022bed90) - secondary
- Layer 2 handler (FUN_022bfc5c) - primary
- Layer 3 handler (FUN_022be9e8) - projectile - **Implemented (straight line motion)**
- Dispatch types (1, 2, 5, 6)

### Special Monster Animations
- Type 98: Multi-direction (9 iterations, +2 direction, 2 frames each) - **Implemented**
- Type 99: Spin (8 directions, +1 each, 2 frames each) - **Implemented**
- Both bypass hit-frame system
- Secondary effect spawns at START
- Use walk animation frame (anim type 0), not attack
- Damage handled AFTER animation completes

### Data Structures
- effect_animation_info (28 bytes, all fields)
- move_animation_info (24 bytes, all fields)
- Per-Pokemon override system (special_monster_move_animation)
- Effect context structure (316 bytes, key offsets documented)
