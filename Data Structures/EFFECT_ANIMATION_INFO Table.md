## Overview
The Effect Animation Info table defines the actual visual effects used by moves. Multiple moves can share the same effect animation (e.g., all "beam" type moves use similar visual effects).
## Memory Locations

| Region | Address      | Symbol Name             |
| ------ | ------------ | ----------------------- |
| EU     | `0x022CCE84` | `EFFECT_ANIMATION_INFO` |
| NA     | `0x022CC52C` | `EFFECT_ANIMATION_INFO` |
| JP     | `0x022CDC14` | `EFFECT_ANIMATION_INFO` |
## Table Properties
- **Entry Count:** 700 entries
- **Entry Size:** 0x1C (28 bytes)
- **Total Size:** 0x4F00 bytes (700 × 28)
- **Index:** Accessed by `effect_id` from `move_animation`->`effect_id`
## Structure Definition
```c
struct effect_animation {
    int field_0x0;             // Animation type/category (0-6)
    int file_index;            // ★ File index in effect.bin (Pack 3)
    int field_0x8;             // Unknown
    int animation_index;       // ★ Animation ID within the WAN file
    int se_id;                 // Sound effect ID to play
    int field_0x14;            // Unknown
    uint8_t field_0x18;        // Unknown - possibly flags
    int8_t field_0x19;         // Unknown - possibly signed parameter
    uint8_t is_non_blocking;   // If non-zero, animation plays delayed/async
    uint8_t unk_repeat;        // If non-zero, animation repeats
};
// Size: 0x1C (28 bytes)
```
## Key Fields
### `field_0x0` - Animation Type
**Determines rendering path and resource allocation.**

| Value | Type     | Description                 | Resource Handling                     |
| ----- | -------- | --------------------------- | ------------------------------------- |
| 0     | Standard | Basic effect animation      | Default path                          |
| 1     | ???      | Unknown type                | Special check in FUN_022be44c         |
| 2     | ???      | Unknown type                | Special check in FUN_022be44c         |
| 3     | Checked  | Requires duplicate check    | Memory size validation                |
| 4     | ???      | Unknown type                | Similar to type 3                     |
| 5     | Complex  | Needs special file handling | Uses +0x10C offset, extra validation  |
| 6     | ???      | Unknown type                | Similar resource allocation to type 5 |
**Reference:** See `FUN_022be44c` at line 2060 in formatted-ghidra-functions.md
### `file_index` (offset 0x4)
**Critical: Points to WAN file in effect.bin**
- Type: `int` (32-bit)
- Used as: Index into Pack Archive 3 (effect.bin)
- Special case (type 5): `file_index + 0x10C` is used
### `animation_index` (offset 0xC)
**Specifies which animation within the WAN file to play**
- Type: `int` (32-bit)
- Corresponds to: `wan_animation_group.sequences[animation_index]`
- Most WAN files contain multiple animations (idle, attack, hit, etc.)
### `se_id` (offset 0x10)
**Sound effect to play alongside animation**
- Type: `int` (32-bit)
- Value `0` = No sound effect
- Played via sound system during animation setup
### `is_non_blocking` (offset 0x1A)
**Controls animation blocking behavior**
- Type: `uint8_t`
- `0x00` = Blocking (wait for animation to finish)
- `!= 0x00` = Non-blocking (animation plays asynchronously)
### `unk_repeat` (offset 0x1B)
**Controls animation looping**
- Type: `uint8_t`
- `0x00` = Play once
- `!= 0x00` = Loop animation
## Accessor Function
```c
// Inline/macro in code
effect_animation* GetEffectAnimation(int effect_id) {
    return (effect_animation*)(effect_id * 0x1C + EFFECT_ANIMATION_INFO);
}
```
**Usage in Code:**
```c
// From PlayMoveAnimation
move_animation* move_anim = GetMoveAnimation(move_id);
effect_animation* effect = GetEffectAnimation(move_anim->effect_id);

// Load WAN file
SetAndLoadCurrentAttackAnimation(
    PACK_ARCHIVE_EFFECT,    // Pack 3
    effect->file_index       // Which file in effect.bin
);

// Play specific animation
PlayEffectAnimationEntity(
    target,
    effect->animation_index, // Which animation in the WAN
    true,                    // Blocking
    // ... other params
);

// Play sound
if (effect->se_id != 0) {
    PlaySoundEffect(effect->se_id);
}
```
## Memory Validation

### Type 5 Animations
```c
// From FUN_022bdd48, line 1979
if (effect->field_0x0 == 5) {
    uint32_t fileSize = GetFileLengthInPackWithPackNb(
        PACK_ARCHIVE_EFFECT,
        (effect->file_index + 0x10C) & 0xFFFF
    );
    
    if (availableMem < fileSize) {
        // Fallback to lower-memory mode
    }
}
```
### Type 3 Animations
```c
// From FUN_022bdd48, line 1994
if (effect->field_0x0 == 3) {
    bool foundSimilar = CheckForSimilarAnimations(effect, params);
    
    if (!foundSimilar) {
        uint32_t fileSize = GetFileLengthInPackWithPackNb(
            PACK_ARCHIVE_EFFECT,
            effect->file_index & 0xFFFF
        );
        
        if (availableMem < fileSize) {
            // Fallback to lower-memory mode
        }
    }
}
```
## Resource Allocation
Based on `field_0x0` (animation type), different resource allocation paths are taken:
```c
// From FUN_022be44c, line 2100
if (effectType == 5 || effectType == 6) {
    resourceId = AllocateAnimationResource(effectType, effect->file_index);
    
    if (effectType == 5) {
        FUN_022c01fc(resourceId, effectType, effect->file_index, param_3);
    } else {
        FUN_022c0280(resourceId, effectType, effect->file_index, param_3);
    }
} else {
    resourceId = AllocateAnimationResourceForType(effectType, effect->file_index);
    
    if (effectType == 3 || effectType == 4) {
        FUN_022c0114(resourceId, effectType, effect->file_index, ...);
    }
}
```
## Related Functions

| Function                           | Address/Line | Purpose                                     |
| ---------------------------------- | ------------ | ------------------------------------------- |
| `GetEffectAnimation`               | Inline       | Returns pointer to effect_animation entry   |
| `FUN_022bdd48`                     | Line 1953    | Memory validation and fallback handling     |
| `FUN_022be44c`                     | Line 2060    | Resource allocation based on animation type |
| `SetAndLoadCurrentAttackAnimation` | -            | Loads WAN file from effect.bin              |
| `PlayEffectAnimationEntity`        | -            | Plays the specified animation               |
## Pack File Structure

Effect animations are stored in:
- **Path:** `/EFFECT/effect.bin`
- **Pack Type:** Pack Archive (AT3PX format)
- **Pack ID:** `PACK_ARCHIVE_EFFECT` (3)
**Loading Process:**
```c
// 1. Open pack file
pack_file = OpenPackArchive(PACK_ARCHIVE_EFFECT);

// 2. Get specific file
wan_data = GetFileFromPack(pack_file, effect->file_index);

// 3. Parse WAN
wan_header = ParseWAN(wan_data);

// 4. Access animation
animation = wan_header->animations[effect->animation_index];
```
## Unknown Fields

- **field_0x8:** Unknown purpose
- **field_0x14:** Unknown purpose
- **field_0x18:** Possibly flags or parameters
- **field_0x19:** Possibly signed parameter (offset? priority?)
## Animation Type Investigation

**TODO:** Determine the exact purpose of each animation type (0-6)

Current observations:
- Types 3 & 5 have special memory checks
- Types 1 & 2 have system state requirements
- Types 5 & 6 use different resource allocation
- Types 3 & 4 require initialization calls