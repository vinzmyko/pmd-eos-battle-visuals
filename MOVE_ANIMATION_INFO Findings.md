# Reverse Engineering Findings: MOVE_ANIMATION_INFO

## Overview

The `MOVE_ANIMATION_INFO` table defines visual animations, sound effects, and behavioural parameters for all moves in the game.

Table Properties:
- Entry Count: 563 moves
- Entry Size: 24 bytes (0x18)
- Total Size: 13,512 bytes (0x5328)

## Memory Locations

| Region | Address | Symbol Name |
|--------|---------|-------------|
| NA | `0x022C9064` | `MOVE_ANIMATION_INFO` |
| EU | `0x022C99BC` | `MOVE_ANIMATION_INFO` |
| JP | `0x022CA74C` | `MOVE_ANIMATION_INFO` |

---

## Structure Definition

```c
struct move_animation {
    int16_t effect_id_0;         // 0x00: Effect animation layer 0
    int16_t effect_id_1;         // 0x02: Effect animation layer 1
    int16_t effect_id_2;         // 0x04: Effect animation layer 2 (primary)
    int16_t effect_id_3;         // 0x06: Effect animation layer 3
    uint8_t behavior_flags;      // 0x08: Behavior control flags
    uint8_t unknown_0x9;         // 0x09: Unknown/unused
    uint8_t unknown_0xa;         // 0x0A: Unknown/unused
    uint8_t unknown_0xb;         // 0x0B: Unknown/unused
    int32_t animation_speed;     // 0x0C: Animation/projectile speed
    uint8_t monster_anim_type;   // 0x10: Monster sprite animation ID
    int8_t position_param;       // 0x11: Position offset lookup index
    uint16_t sound_effect_id;    // 0x12: Sound effect ID
    int16_t override_count;      // 0x14: Special override entry count
    uint16_t override_index;     // 0x16: Special override table index
};
```

---

## Field Documentation

### Offset 0x00-0x06: Effect Animation Layers

Type: Four `int16_t` values (effect IDs)

Purpose: References to the `EFFECT_ANIMATION_INFO` table. Multiple effect layers can be active simultaneously.

Evidence:
- `FUN_022bf160`: Iterates through all four fields, treating each as an effect_id and calling `GetEffectAnimation()` on each value to check if any layer uses animation type 5
- `FUN_022bf1fc`: Similar iteration pattern, validates specific effect layers by index
- `GetMoveAnimation`: Base accessor function that returns pointer to structure, confirming all four fields are read as effect IDs in various contexts
- `PlayMoveAnimation`: Reads `field_0x4` as the primary effect_id
- `FUN_022bed90`: Reads `field_0x2` to spawn secondary animations
- `FUN_022bfaa8`: Reads `field_0x0` for initial charge/preparation effects
- `FUN_022be9e8`: Reads `field_0x6` for projectile effects

Usage Pattern:
- Layer 0 (0x00): Often used for charge-up or preparation effects
- Layer 1 (0x02): Secondary impact or follow-up effects
- Layer 2 (0x04): Primary visual effect (always used if move has animation)
- Layer 3 (0x06): Projectile or traveling effects

Special Cases:
- Value `0x0000` indicates no effect for that layer
- Move ID `0x52` (Razor Leaf) creates 8 separate animation instances using layer data

---

### Offset 0x08: Behavior Flags

Type: `uint8_t` (bit field)

Purpose: Controls various animation behaviors through individual bit flags.

Evidence:
- `FUN_022bfd58`: Extracts bits 0-2 with mask `0x07`, returns animation category (0-7)
- `FUN_022bfd6c`: Checks bit 3 with mask `0x08`
- `FUN_022bfd8c`: Checks bit 4 with mask `0x10`, controls fade-in behavior
- `FUN_022bfdac`: Checks bit 5 with mask `0x20`
- `FUN_022bfdcc`: Checks bit 6 with mask `0x40`, triggers `AnimationDelayOrSomething()`
- `FUN_02324e78`: Uses bit 4 check to determine whether to perform screen fade-in effect

Bit Mapping:

| Bits | Mask | Purpose | Evidence Function |
|------|------|---------|-------------------|
| 0-2 | `0x07` | Animation category (0-7) | `FUN_022bfd58` |
| 3 | `0x08` | Unknown flag | `FUN_022bfd6c` |
| 4 | `0x10` | Skip fade-in effect (if set) | `FUN_022bfd8c`, `FUN_02324e78` |
| 5 | `0x20` | Unknown flag | `FUN_022bfdac` |
| 6 | `0x40` | Add post-animation delay (if set) | `FUN_022bfdcc`, `FUN_023250d4` |
| 7 | `0x80` | Unknown/unused | - |

---

### Offset 0x09-0x0B: Unknown Fields

Type: Three `uint8_t` values

Purpose: Unknown.

Evidence:
- Exhaustive search of decompiled functions found no reads or writes to these offsets
- Likely padding or reserved for future use
- Consistently zero in most move entries examined

---

### Offset 0x0C: Animation Speed

Type: `int32_t` (spans 0x0C-0x0F)

Purpose: Controls the speed of projectile animations and frame timing.

Evidence:
- `GetMoveAnimationSpeed`: Returns value at `BASE_ADDRESS + 0xC`, confirming it's a 4-byte integer starting at this offset
- `FUN_023230fc`: Uses return value to calculate frame count via division: `frame_count = 24 / speed`
- `FUN_0232393c`: Similar speed-based frame calculation logic

Speed Mapping:

| Raw Value | Mapped To | Frame Count | Description |
|-----------|-----------|-------------|-------------|
| 1 | 2 | 12 frames | Slow projectile |
| 2 | 3 | 8 frames | Medium projectile |
| Other | 6 | 4 frames | Fast projectile (default) |

Implementation:
```c
speed = GetMoveAnimationSpeed(move_id);
if (speed == 1) speed = 2;
else if (speed == 2) speed = 3;
else speed = 6;
frames = 24 / speed;
```

---

### Offset 0x10: Monster Animation Type

Type: `uint8_t`

Purpose: Specifies which sprite animation the attacking monster should perform during the move.

Evidence:
- `FUN_022bfa3c`: Reads this field and returns it (or override value), used to determine monster behavior
- `FUN_023250d4`: Checks for special values `99` (0x63) and `0x62` (98) which trigger unique animation sequences
- `ChangeMonsterAnimation`: Called with this value to set the monster's sprite animation

Special Values:

| Value | Animation | Behavior |
|-------|-----------|----------|
| 0 | Walk | Basic movement |
| 1 | Attack | Standard attack animation |
| 7 | Idle | Standing idle |
| 8 | Swing | Special attack (Faint Attack, Thief) |
| 9 | Double | Evasive moves (Double Team, Agility) |
| 10 | Hop | Jump moves (Dig, Bounce) |
| 11 | Charge | Stat boost/status moves |
| 12 | Rotate | Throwing/projectile moves |
| 99 | Special | Spin through all 8 directions |
| 98 (0x62) | Special | Attack in 9 directions sequentially |

Special Behavior (Value 99):
```
// From FUN_023250d4
Plays monster animation cycling through:
Direction → Direction+1 → Direction+2 → ... (8 iterations)
Used for moves like Mud-Slap
```

Special Behavior (Value 98):
```
// From FUN_023250d4
Plays monster animation in 9 different directions
Direction → Direction+2 → Direction+4 → ... (9 iterations)
Used for area-of-effect moves
```

---

### Offset 0x11: Position Offset Parameter

Type: `int8_t` (signed)

Purpose: Index (0-3) into a position offset lookup table. Controls where effects spawn relative to the monster sprite.

Evidence:
- `FUN_022bf01c`: Reads this field and uses it as a lookup index, can be overridden by special_monster_move_animation table
- `FUN_0201cf90`: Takes this parameter and uses it to calculate final position offsets:
  - Accesses lookup table at: `base_pointer + sprite_id  0x10 + (param  4)`
  - Returns two `int16_t` values (x_offset, y_offset)
  - Special case: If lookup returns `(99, 99)`, hardcodes to `(0x63, 0x63)`

Parameter Range: 0-3 (values outside this range may cause undefined behavior)

---

### Offset 0x12: Sound Effect ID

Type: `uint16_t`

Purpose: References the game's sound effect table. Plays when the move animation triggers.

Evidence:
- `FUN_022bf0f4`: Reads this field and returns it (or override value)
- `FUN_023250d4`: Checks if value equals `0x3F00` (silence constant), calls `PlaySeByIdIfNotSilence()` otherwise

Special Values:
- `0x3F00`: Represents silence/no sound effect
- `0x0000`: Also appears to mean no sound in some contexts
- Other values: Direct index into sound effect table

Implementation:
```c
sound_id = GetSoundEffectForMove(move_id);
if (sound_id != 0x3F00) {
    PlaySeByIdIfNotSilence(sound_id);
}
```

---

### Offset 0x14-0x16: Special Animation Override System

Type: `int16_t override_count`, `uint16_t override_index`

Purpose: Allows different Pokémon species to have custom animation parameters for the same move

Evidence:
- `GetSpecialMonsterMoveAnimation`: Returns pointer to override table entry: `base_address + (index  6)`
- `FUN_022bf01c`: Loops through `override_count` entries, matching against user's sprite ID
- `FUN_022bf0f4`: Similar loop for sound effect overrides
- `FUN_022bfa3c`: Similar loop for animation type overrides

Override Table Structure:
```c
struct special_monster_move_animation {
    int16_t sprite_id;      // 0x0: Monster sprite to match
    uint8_t anim_param;     // 0x2: Override for monster_anim_type
    uint8_t position_param; // 0x3: Override for position_param
    uint16_t sound_param;   // 0x4: Override for sound_effect_id
};
// Size: 6 bytes
```

Lookup Algorithm:
```
for (i = 0; i < override_count; i++) {
    entry = special_anim_table[override_index + i];
    if (entry.sprite_id == user_sprite_id) {
        return entry.override_value;  // Use override
    }
}
return default_value;  // Use base move_animation value
```

Use Cases:
- Same move appears different colors on different Pokémon
- Weather Ball changes based on current weather
- Signature moves have special effects for certain species

---

## Function Reference

### Core Accessor Functions

| Function | Purpose |
|----------|---------|
| `GetMoveAnimation` | Returns pointer to move_animation entry |
| `GetMoveAnimationSpeed` | Reads animation_speed field (offset 0xC) |
| `GetMoveAnimationId` | Resolves move ID with weather/alternative checks |

### Field Readers (Flags)

| Function | Reads | Returns |
|----------|-------|---------|
| `FUN_022bfd58` | `behavior_flags & 0x07` | Animation category |
| `FUN_022bfd6c` | `behavior_flags & 0x08` | Bit 3 flag |
| `FUN_022bfd8c` | `behavior_flags & 0x10` | Skip fade-in flag |
| `FUN_022bfdac` | `behavior_flags & 0x20` | Bit 5 flag |
| `FUN_022bfdcc` | `behavior_flags & 0x40` | Post-delay flag |

### Field Readers (Override System)

| Function | Reads | Returns |
|----------|-------|---------|
| `FUN_022bf01c` | `position_param` | Position parameter (or override) |
| `FUN_022bf0f4` | `sound_effect_id` | Sound effect ID (or override) |
| `FUN_022bfa3c` | `monster_anim_type` | Animation type (or override) |
| `GetSpecialMonsterMoveAnimation` | `override_index` | Pointer to override table entry |

### Effect Animation Handlers

| Function | Purpose |
|----------|---------|
| `FUN_022bf160` | Checks if any of 4 effect layers uses type 5 |
| `FUN_022bf1fc` | Validates specific effect layer by index |
| `FUN_022bfaa8` | Spawns effect from layer 0 |
| `FUN_022bed90` | Spawns effect from layer 1, handles Razor Leaf special case |
| `FUN_022bfc5c` | Spawns effect from layer 2 |
| `FUN_022be9e8` | Spawns effect from layer 3 |

### Animation Orchestrators

| Function | Purpose |
|----------|---------|
| `PlayMoveAnimation` | Main entry point, coordinates entire animation sequence |
| `FUN_023250d4` | Handles monster animation, special behaviors (99, 0x62) |
| `FUN_02325644` | Triggers secondary effects, checks if layer 1 is used |
| `FUN_023230fc` | Projectile movement logic, uses animation_speed |
| `FUN_02324e78` | Screen-wide effects, fade-in handling |
| `FUN_022e6d68` | Attaches animation instance to entity |

### Helper Functions

| Function | Purpose |
|----------|---------|
| `FUN_0201cf90` | Position offset calculation using position_param |
| `ShouldMovePlayAlternativeAnimation` | Checks for two-turn move charging state |
| `GetApparentWeather` | Gets current weather for Weather Ball logic |
| `AnimationDelayOrSomething` | Adds post-animation delay when flag bit 6 set |

---

## Special Move Behaviors

### Move 0x52 (Razor Leaf)

Evidence: `FUN_022bed90`

Creates 8 separate animation instances with staggered positions:
- Each instance offset by `+0x40` on Y-axis
- Each instance receives different X offset based on iteration
- All instances use the same effect_id from `field_0x2`
- First instance handle is returned for tracking

### Monster Animation Value 99 (Spin Attack)

Evidence: `FUN_023250d4`

Triggers special spin behavior:
1. Reads current direction from entity
2. Calls `FUN_02325644` to spawn effects
3. Loops 8 times:
   - Increments direction by 1 (mod 8)
   - Calls `ChangeMonsterAnimation()` with new direction
   - Calls `FUN_022ea370()` for frame timing

### Monster Animation Value 0x62 (Multi-Directional)

Evidence: `FUN_023250d4`

Triggers multi-directional attack:
1. Reads current direction from entity
2. Calls `FUN_02325644` to spawn effects
3. Loops 9 times:
   - Increments direction by 2 (wrapping at 7)
   - Calls `ChangeMonsterAnimation()` with new direction
   - Calls `FUN_022ea370()` for frame timing

---

## Projectile Motion System

Evidence: `FUN_023230fc`

The animation_speed field controls projectile behavior through mathematical motion calculations:

### Frame Count Calculation
```
base_frames = 24
actual_frames = base_frames / mapped_speed
```

### Motion Patterns (param_4)

Pattern 1: Simple Wave
- Single sine wave on vertical axis
- Creates gentle wobbling motion
- Formula: `y_offset = amplitude  sin(frame_angle)`

Pattern 2: Complex Spiral
- Dual sine/cosine calculation
- Creates circular or spiral motion
- Formulas:
  - `base = (amplitude / 2)  sin(frame_angle)`
  - `x_offset = base  sin(rotation)`
  - `y_offset = base  cos(rotation)`

### Position Update Per Frame
```
position_x += direction_x_delta  speed
position_y += direction_y_delta  speed
angle += angle_increment
```

---

## Data Validation Notes

### Confirmed Behaviors
- All four effect_id fields (0x00-0x06) are valid effect references
- Offset 0x0C is definitively a 4-byte integer (not single byte)
- Behavior flags use individual bits, not a single delay value
- Override system allows per-species customization

### Unconfirmed Elements
- Exact purpose of bits 3 and 5 in behavior_flags
- Purpose of bytes 0x09-0x0B (likely padding)
- Full range of animation_category values (bits 0-2 of flags)

### Common Pitfalls
- Do NOT treat offset 0x0C as a single byte
- Do NOT interpret offset 0x08 as a delay timer
- Remember offset 0x11 is an INDEX, not a direct offset value
- Effect layers do not have a strict "phase 1/2/3" system

---

## Reverse Engineering Methodology

Tools Used:
- Ghidra decompiler
- DeSmuME emulator with Lua scripting
- Memory inspection and pattern analysis

Approach:
1. Located base address via known move references
2. Identified accessor functions via cross-references
3. Traced data flow through animation system
4. Validated findings against multiple move examples
5. Confirmed structure size via array indexing patterns

Confidence Level: ~95%

Remaining Unknowns:
- Bytes 0x09-0x0B purpose
- Behavior flags bits 3 and 5
- Full mapping of animation categories (bits 0-2)

---

## Version Information

Game Versions: North America (NA) release
Last Updated: 2025

Credits:
- Reverse engineering: Analysis of Ghidra decompiled code
- Original game: Chunsoft / The Pokémon Company
