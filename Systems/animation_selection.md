# Animation Selection

## Summary

- Animation selection differs by sprite type (CHARA vs PROPS_UI)
- CHARA sprites use `animations[group_id][direction]` - 8 directions per animation group
- PROPS_UI sprites use `animations[0][animation_index]` - flat sequence index, direction ignored
- Effect sprites may have direction (0-7) added to base animation_index if sequence_count % 8 == 0
- `SetAnimationForAnimationControl` is the main entry point, routes to appropriate path

## Selection Flow

```
SetAnimationForAnimationControl(anim_ctrl, animation_key, direction, ...)
        │
        ├─► SpriteTypeInWanTable(sprite_id)
        │
        ├─► CHARA (type 1):
        │       │
        │       └─► SetAnimationForAnimationControlInternal(anim_ctrl, wan, animation_key, direction, ...)
        │               │
        │               └─► animations[animation_key][direction]
        │
        └─► PROPS_UI (type 0, 2, 3):
                │
                └─► SetAnimationForAnimationControlInternal(anim_ctrl, wan, 0, animation_key, ...)
                        │
                        └─► animations[0][animation_key]
```

## CHARA Sprite Selection (Type 1)

Character sprites (Pokémon, NPCs) use a two-level selection:

1. **Group ID** (`animation_key`): Selects animation type (walk, attack, idle, etc.)
2. **Direction** (`direction`): Selects variant within group (0-7 for 8 directions)

**Evidence:** `SetAnimationForAnimationControl` at `0x0201C4F4`
```c
wVar2 = SpriteTypeInWanTable(anim_ctrl->loaded_sprite_id);
// ...
else {
    // CHARA: Uses animation_key as GROUP, direction as SEQUENCE within group
    if (WanTableSpriteHasAnimationGroup(sprite_id, animation_key)) {
        SetAnimationForAnimationControlInternal(anim_ctrl, pwVar3, animation_key, direction, ...);
    }
}
```

**Evidence:** `SetAnimationForAnimationControlInternal` at `0x0201C5C4`
```c
if (*(char *)&wan_header->sprite_type == '\x01') {
    // CHARA: animations[group_id][animation_id]
    pwVar3 = pwVar8->animations[animation_group_id].pnt[animation_id];
}
```

### CHARA Animation Groups

Standard monster animation groups:

| Group | Name | Description |
|-------|------|-------------|
| 0 | Walk | Walking animation |
| 1 | Attack | Basic attack |
| 5 | Sleep | Sleeping |
| 6 | Hurt | Taking damage |
| 7 | Idle | Standing idle |
| 8 | Swing | Swing attack (Faint Attack, Thief) |
| 9 | Double | Clone effect (Double Team, Agility) |
| 10 | Hop | Jumping (Dig, Bounce) |
| 11 | Charge | Charging up (Glare, Detect) |
| 12 | Rotate | Throwing/rotating (Mud-Slap) |

Each group contains 8 sequences (directions 0-7).

## PROPS_UI Sprite Selection (Types 0, 2, 3)

Effect and UI sprites use flat indexing:

1. **Group**: Always 0 (ignored)
2. **Sequence Index** (`animation_key`): Direct index into animations[0]

**Evidence:** `SetAnimationForAnimationControl` at `0x0201C4F4`
```c
wVar2 = SpriteTypeInWanTable(anim_ctrl->loaded_sprite_id);
if ((wVar2 == WAN_SPRITE_PROPS_UI) || ((wVar2 + 0xfe & 0xff) < 2)) {
    // PROPS_UI: Uses animation_key directly, IGNORES direction parameter
    SetAnimationForAnimationControlInternal(anim_ctrl, pwVar3, 0, animation_key, ...);
}
```

**Evidence:** `SetAnimationForAnimationControlInternal` at `0x0201C5C4`
```c
else {
    // PROPS_UI/Effects: animations[0][animation_id] - IGNORES group
    pwVar3 = pwVar8->animations->pnt[animation_id];
}
```

### Direction Parameter Ignored

For PROPS_UI sprites, the `direction` parameter passed to `SetAnimationForAnimationControl` is **completely ignored**. All effect spawning code passes `DIR_DOWN` as a placeholder:

```c
SetAnimationForAnimationControl(
    (animation_control *)(param_1 + 0x68),
    animation_index,        // This becomes the sequence index
    DIR_DOWN,               // IGNORED for PROPS_UI
    ...
);
```

## Effect Direction Handling

Effects achieve directional appearance through a different mechanism: direction may be **added to the base animation_index** before selection.

### Direction Addition Logic

**Evidence:** `FUN_022bdfc0`
```c
// 1. Get base animation_index from effect_animation_info
*(int *)(param_1 + 0x50) = peVar2->animation_index;

// 2. Get sprite sequence count
if (anim_type != 5 && anim_type != 6) {
    uVar3 = FUN_0201da20(*(short *)(param_1 + 0x64));  // sprite_id
    *(undefined4 *)(param_1 + 0x10) = uVar3;  // Store sequence_count
}

// 3. Conditionally add direction to animation_index
iVar6 = *(int *)(param_1 + 0x1c);  // direction (0-7, or -1 if none)
if ((iVar6 != -1) && (sequence_count % 8 == 0)) {
    *(int *)(param_1 + 0x50) += iVar6;  // animation_index += direction
}
```

### The % 8 Test

Direction is added if and only if `sequence_count % 8 == 0`.

The ROM implements this via 32-bit overflow detection:
```c
// Condition: (sequence_count * 0x20000000) overflows to 0
// This happens when sequence_count is a multiple of 8
```

| sequence_count | % 8 | Direction Added? |
|----------------|-----|------------------|
| 1-7 | ≠ 0 | No |
| 8 | = 0 | Yes |
| 9-15 | ≠ 0 | No |
| 16 | = 0 | Yes |
| 50 | ≠ 0 | No |

### Directional vs Non-Directional Effects

**Directional Effects** (sequence_count % 8 == 0):
- WAN file has N×8 sequences (e.g., 8, 16, 24)
- Sequences are grouped in sets of 8 (one per direction)
- Base animation_index points to direction 0
- Final index = base + direction

**Non-Directional Effects** (sequence_count % 8 ≠ 0):
- WAN file has non-multiple-of-8 sequences (e.g., 50)
- Direction does NOT affect which sequence plays
- Direction only affects projectile motion (X/Y velocity)
- Same visual regardless of attacker facing

### Examples

**Water Gun (Effect 334):**
- Sequence count: 50
- 50 % 8 = 2 ≠ 0
- **Non-directional:** Same animation, direction only affects projectile travel

**Shadow Sneak (Effect 479):**
- Sequence count: 16
- 16 % 8 = 0
- **Directional:** Sequences 8-15 are 8 pre-rotated variants
- Base index 8 + direction (0-7) = final index 8-15

## Selection Summary Table

| Sprite Type | Group Selection | Sequence Selection | Direction Handling |
|-------------|-----------------|-------------------|-------------------|
| CHARA (1) | `animation_key` | `direction` param | 8 sequences per group |
| PROPS_UI (0) | Always 0 | `animation_key` | Ignored (or added to index if seq%8==0) |
| Type 2 | Always 0 | `animation_key` | Same as PROPS_UI |
| Type 3 | Always 0 | `animation_key` | Same as PROPS_UI |

## Open Questions

- Complete list of which effect WAN files have sequence counts divisible by 8
- Whether any effects use direction addition with base index > 0
- How screen effects (type 5) and WBA effects (type 6) handle direction

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `SetAnimationForAnimationControl` | `0x0201C4F4` | Main entry point, routes by sprite type |
| `SetAnimationForAnimationControlInternal` | `0x0201C5C4` | Actually selects animation sequence |
| `SpriteTypeInWanTable` | `0x0201DA98` | Get sprite type from WAN table |
| `WanTableSpriteHasAnimationGroup` | `0x0201DAA0` | Check if sprite has animation group |
| `FUN_0201da20` | `0x0201DA20` | Get sequence count from sprite |
| `FUN_022bdfc0` | `0x022BDFC0` | Effect initialization (direction addition logic) |
