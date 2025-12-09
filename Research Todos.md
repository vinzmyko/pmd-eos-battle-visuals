# ROM Research TODO

## High Priority (Affects Visual Accuracy)

## Foundational (Research First)

### Pokemon Tile Positioning
- **Current**: Using `grid_to_world()` - unclear if matches ROM
- **Questions**:
  - Is Pokemon centered on tile or offset?
  - What's the sprite origin point? (feet? center?)
  - How does this affect attachment point calculations?
  - How does movement interpolation work?
- **Why it matters**: All effect positioning builds on this
- **To check**: Compare visual positioning with original game footage

### Animation 99/98 Spin Timing
- **Current**: Not implemented
- **Known from decompilation**:
  - Type 99: 8 iterations, one per direction
  - Type 98: 9 iterations, incrementing by 2
  - `FUN_022ea370(2, 0x15, ...)` called each iteration
- **Questions**:
  - What is `FUN_022ea370`? Delay function?
  - Is `0x15` (21) the frame count per direction?
  - Does the Pokemon play full animation or just pose?
- **Test**: 21 frames × 8 directions = 168 frames ≈ 2.8 sec total

### Effect Spawn Position
- **Current**: Primary on target center, others on attacker center
- **Questions**:
  - How does attachment_point affect spawn position relative to attacker vs target?
  - Does the ROM calculate position differently per layer?
  - What coordinate system is used? (pixel offset from entity origin?)
- **Functions to investigate**:
  - `FUN_0201cf90` - attachment point offset calculation
  - `FUN_022be780` - main effect dispatcher

### Looping Effect Termination
- **Current**: Timeout after 3 loops or 1 second
- **Questions**:
  - What triggers loop termination in ROM?
  - Is it tied to move execution state?
  - Is there an explicit "stop effect" call?
- **Functions to investigate**:
  - `FUN_022bde50` - stop/cleanup effect
  - `FUN_022bdec4` - force-stop effect
  - `AnimationHasMoreFrames` - checked in loops

### Centre Attachment Point Data
- **Current**: Returns Vector2.ZERO
- **Questions**:
  - Does WAN file contain center offset per frame?
  - Is it in `wan_offset.center` structure?
  - Should scraper extract this?
- **Check in scraper**: `wan/parser.rs` - does it parse center position?

## Medium Priority (Missing Features)

### Projectile Motion (Layer 3)
- **Current**: Spawns on attacker, doesn't move
- **Questions**:
  - How is velocity calculated from `projectile_speed`?
  - What are the wave patterns (0=straight, 1=sine, 2=spiral)?
  - How does direction affect trajectory?
- **Functions to investigate**:
  - `FUN_023230fc` - projectile movement with wave patterns
  - `FUN_0232393c` - reverse direction projectile
  - `GetMoveAnimationSpeed` - speed lookup
- **Data needed**:
  - Direction from attacker to target
  - `projectile_speed` from move_animation_info

### Layer Timing
- **Current**: All effects spawn at same time (on hit frame)
- **Questions**:
  - Does Layer 0 (Charge) play BEFORE attack animation?
  - Does Layer 1 (Secondary) have different timing?
  - Is there delay between layers?
- **Functions to investigate**:
  - `FUN_022bfaa8` - Layer 0 handler
  - `FUN_022bed90` - Layer 1 handler  
  - `FUN_022bfc5c` - Layer 2 handler
  - `FUN_022be9e8` - Layer 3 handler

### Non-Blocking Effect Await
- **Current**: Flag passed through but not used
- **Questions**:
  - How does ROM wait for blocking effects?
  - What happens during the wait?
  - Does this affect other game state?
- **Functions to investigate**:
  - `AnimationHasMoreFrames` - blocking check

## Lower Priority (Polish)

### Screen Effects (Type 5)
- **Current**: Skipped entirely
- **Questions**:
  - What are screen effects? (flashes, fades, overlays?)
  - How to render them?
  - File index uses `+0x10C` offset - what files are these?
- **Functions to investigate**:
  - `FUN_022c01fc` - type 5 setup

### Sound Effects
- **Current**: Not implemented
- **Data available**: `sfx_id` in effect_animation_info
- **Questions**:
  - How to map sfx_id to actual sound files?
  - Are sounds in a separate archive?

### Special Move Behaviors
- **Move 0x52 (Razor Leaf)**: Creates 8 animation instances
- **Animation type 99**: Spin through 8 directions
- **Animation type 98**: Multi-directional attack (9 directions)
- **Questions**:
  - How to detect these special cases?
  - Should scraper flag these specially?

## Data Format Questions

### Effect Animation Index Interpretation
- **Finding**: For effect sprites, `animation_index` is flat sequence index into group 0
- **Verify**: Is this always true or only for certain types?
- **Reference**: Move Effect System.md - "Critical Discovery" section

### Direction System
- **Finding**: Direction only affects projectile velocity, NOT sprite selection
- **Verify**: Test with directional effects in game
- **Reference**: EFFECT_ANIMATION_INFO Findings.md - Direction System section

## References
- EFFECT_ANIMATION_INFO Findings.md
- MOVE_ANIMATION_INFO Findings.md  
- Move Effect System.md
