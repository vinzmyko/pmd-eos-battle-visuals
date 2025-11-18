## Overview
The Move Animation Info table maps move IDs to their visual effects. Each move has an entry that specifies which effect animation to use, along with additional parameters for playback.

## Memory Locations

| Region | Address | Symbol Name |
|--------|---------|-------------|
| EU     | `0x022C99BC` | `MOVE_ANIMATION_INFO` |
| NA     | `0x022C9064` | `MOVE_ANIMATION_INFO` |
| JP     | `0x022CA74C` | `MOVE_ANIMATION_INFO` |

## Table Properties
- **Entry Count:** 563 entries
- **Entry Size:** 0x18 (24 bytes)
- **Total Size:** 0x5328 bytes (563 × 24)
- **Index:** Accessed directly by `move_id`

## Structure Definition
```c
struct move_animation {
int16_t field_0x0;         // Unknown - possibly animation variant/flags
int16_t field_0x2;         // Unknown
int16_t effect_id;         // ★ Index into EFFECT_ANIMATION_INFO
int16_t field_0x6;         // Unknown
uint8_t field_0x8;         // Unknown - possibly timing/delay
uint8_t field_0x9;         // Unknown
uint8_t field_0xa;         // Unknown
uint8_t field_0xb;         // Unknown
uint8_t field_0xc;         // Unknown
int8_t field_0x11;         // Unknown - possibly signed offset
uint16_t field_0x12;       // Unknown
int16_t field_0x14;        // Unknown
uint16_t field_0x16;       // Unknown
};
// Size: 0x18 (24 bytes)
```
## Key Fields

### `effect_id` (offset 0x4)
**Critical field that links moves to effect animations.**
- Type: `int16_t`
- Range: `0x0000` - `0x02BC` (0-700)
- Value `0x0000` = No animation
- Used as index into `EFFECT_ANIMATION_INFO` table
## Accessor Function
```c
// Address: Varies by region
move_animation* GetMoveAnimation(move_id move_id) {
    return (move_animation*)(move_id * 0x18 + MOVE_ANIMATION_INFO);
}
```

**Usage in Code:**
```c
// From PlayMoveAnimation
move_animation* anim = GetMoveAnimation(move->id);
int16_t effect_id = anim->effect_id;  // offset 0x4

if (effect_id == 0) return;  // No animation

effect_animation* effect = GetEffectAnimation(effect_id);
```
## Special Cases

### Weather Ball (Move ID 0x1AF)
- Has dynamic `effect_id` selection based on current weather
- Handled by `GetMoveAnimationId()` before table lookup
### Two-Turn Moves
- Alternative animations triggered via `ShouldMovePlayAlternativeAnimation()`
- Examples: Solar Beam (charging), Fly (ascending), Dig (underground)
### Moves with No Animation
- `effect_id = 0x0000`
- Function returns early without effect lookup
## Unknown Fields
The following fields have been identified but their purpose is unclear:
- **field_0x0, field_0x2:** Possibly animation variants or flags
- **field_0x6:** Unknown purpose
- **field_0x8 - field_0x11:** Possibly timing, positioning, or behaviour flags
- **field_0x12 - field_0x16:** Unknown purpose