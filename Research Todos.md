# Research Todos

## Status Overview

| Area | Status | Notes |
|------|--------|-------|
| Entity Positioning | Complete | Items +4,+4; Monsters +12,+16; Attachment points |
| Projectile Motion | Complete | Speed mapping, direction tables, wave patterns |
| Animation Timing | Complete | AdvanceFrame, frame loops, task scheduler |
| Move Effect Pipeline | Complete | Layers, handlers, flag bits |
| effect_animation_info | Complete | All fields documented |
| move_animation_info | Complete | All fields documented |
| Entity Structure | Partial | Basic structure known, monster_info incomplete |
| Screen Effects (Type 5/6) | Not Started | Low priority |
| Sound System | Not Started | Low priority |

---

## Completed Research

### Entity Positioning
- [x] Tile-to-pixel formula for items (+4, +4)
- [x] Tile-to-pixel formula for monsters (+12, +16)
- [x] 8.8 fixed point format confirmed
- [x] Attachment point calculation (FUN_0201cf90)
- [x] WAN offset structure (16 bytes per frame, 4 points)
- [x] Special case (99, 99) = center

### Projectile Motion
- [x] Speed mapping (1→2, 2→3, other→6)
- [x] Frame count calculation (24 / mapped_speed)
- [x] Direction table (DIRECTIONS_XY) confirmed
- [x] Wave pattern 0: Straight line
- [x] Wave pattern 1: Vertical sine
- [x] Wave pattern 2: Spiral/circular
- [x] Amplitude based on range
- [x] Phase progression (half sine over travel)
- [x] Reverse direction (+4 & 7)
- [x] Z priority calculation

### Animation Timing
- [x] AdvanceFrame yields one frame
- [x] FUN_022ea370 loops n times (not n × param)
- [x] Task scheduler structure (FUN_022ddef8)
- [x] Blocking loop pattern (AnimationHasMoreFrames)
- [x] Safety timeout (100 frames in some cases)
- [x] Pre-screen effect wait (FUN_0234ba54, up to 239 frames)

### Move Effect Pipeline
- [x] ExecuteMoveEffect as entry point
- [x] Flag bit 3 = dual-target (FUN_023258ec)
- [x] Flag bit 4 = skip fade-in
- [x] Flag bit 6 = post-animation delay
- [x] Layer 0 handler (FUN_022bfaa8) - charge/prep
- [x] Layer 1 handler (FUN_022bed90) - secondary
- [x] Layer 2 handler (FUN_022bfc5c) - primary
- [x] Layer 3 handler (FUN_022be9e8) - projectile
- [x] Dispatch types (1, 2, 5, 6)
- [x] Razor Leaf special case (8 instances)
- [x] Monster animation 98 (multi-direction)
- [x] Monster animation 99 (spin)
- [x] Type 5 effect check (FUN_022bf160)

### Data Structures
- [x] effect_animation_info (28 bytes, all fields)
- [x] move_animation_info (24 bytes, all fields)
- [x] Animation index interpretation for PROPS_UI sprites
- [x] Direction NOT added to animation_index for effects
- [x] Per-Pokemon override system (special_monster_move_animation)
- [x] Effect context structure (316 bytes, key offsets)

---

## Open Questions (Low Priority)

### Flag Bits
- [ ] Flag bits 0-2 (animation category) - values 0-7 observed, purpose unknown
- [ ] Flag bit 5 - never observed set, purpose unknown
- [ ] Flag bit 7 - never observed set, purpose unknown

### Effect System
- [ ] Exact difference between type 1 and type 2 effects (both WAN, differ by state check)
- [ ] How looping effects (`loop_flag`) are terminated naturally
- [ ] Complete effect_context structure (many offsets still unknown)
- [ ] Screen effect control structure at context + 0xE8

### Entity System
- [ ] Complete monster_info structure (many offsets unknown)
- [ ] Animation control structure layout
- [ ] How elevation affects Y rendering
- [ ] Entity struct total size

### Timing
- [ ] What param_2 in FUN_022ea370 controls (passed through but unclear)
- [ ] Layer 0 timing relationship to two-turn move charging

---

## Not Planned (Will do much later after we confirm most moves are working)

### Screen Effects (Type 5/6)
- Rendering method unclear
- Files at file_index + 0x10C
- Would need significant investigation
- Not critical for basic move recreation

### Sound System
- sfx_id mapping to actual sound files
- Sound archive format
- Not visual, lower priority

### Complete ROM Documentation
- Full move effect switch statement
- All ability interactions
- Damage calculation
- Not related to animation system

---

## Potential Future Research

If deeper accuracy is needed:

### Layer Timing Investigation
**Question:** Does Layer 0 (charge) play BEFORE attack animation starts for two-turn moves?

**Approach:**
1. Find a two-turn move with Layer 0 effect
2. Trace execution in FUN_023250d4
3. Check timing relative to ChangeMonsterAnimation call

### Loop Termination Investigation
**Question:** What triggers natural termination of looping effects?

**Approach:**
1. Find an effect with loop_flag = 1
2. Trace what calls FUN_022bde50 for that effect
3. Check for move completion signals

### Animation Category Investigation
**Question:** What do flag bits 0-2 (animation category 0-7) control?

**Approach:**
1. Extract all moves with different category values
2. Compare behavior in-game
3. Trace FUN_022bfd58 callers
