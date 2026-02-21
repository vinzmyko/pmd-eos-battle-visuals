# Move Animation Info Structure

## Summary

- 563 entries, 24 bytes each
- Maps move IDs to visual effects, sounds, and animation parameters
- Contains 4 effect layers that can play simultaneously
- Supports per-Pokemon animation overrides via special_monster_move_animation table

## Memory Locations

| Region | Address | Symbol Name |
|--------|---------|-------------|
| NA | `0x022C9064` | `MOVE_ANIMATION_INFO` |
| EU | `0x022C99BC` | `MOVE_ANIMATION_INFO` |
| JP | `0x022CA74C` | `MOVE_ANIMATION_INFO` |

## Table Properties

| Property | Value |
|----------|-------|
| Entry Count | 563 |
| Entry Size | 24 bytes (0x18) |
| Total Size | 13,512 bytes (0x34F8) |

## Structure Definition
```c
struct move_animation {
    int16_t effect_id_layer0;    // 0x00: Charge/preparation effect
    int16_t effect_id_layer1;    // 0x02: Secondary effect
    int16_t effect_id_layer2;    // 0x04: Primary visual effect
    int16_t effect_id_layer3;    // 0x06: Projectile effect
    uint8_t flags;               // 0x08: Behavior flags (see below)
    uint8_t _padding[3];         // 0x09-0x0B: Alignment padding (always 0)
    int32_t projectile_speed;    // 0x0C: Speed value (0-2, other = fast)
    uint8_t monster_anim_type;   // 0x10: Monster sprite animation (0-12, 98, 99)
    int8_t  attachment_point_idx;// 0x11: Position offset index (-1 to 3) - SIGNED
    uint16_t sound_effect_id;    // 0x12: Sound effect (0x3F00 = silence)
    int16_t override_count;      // 0x14: Number of per-Pokemon overrides
    uint16_t override_table_idx; // 0x16: Index into special_monster_move_animation
};
// Size: 0x18 (24 bytes)
```

## Field Documentation

### Effect Layers (offsets 0x00-0x06)

Four effect layers can play simultaneously. Value 0 = no effect for that layer.

| Layer | Offset | Field | Handler | Purpose |
|-------|--------|-------|---------|---------|
| 0 | 0x00 | effect_id_layer0 | `FUN_022bfaa8` | Charge/preparation |
| 1 | 0x02 | effect_id_layer1 | `FUN_022bed90` | Secondary/multi-hit |
| 2 | 0x04 | effect_id_layer2 | `FUN_022bfc5c` | Primary visual |
| 3 | 0x06 | effect_id_layer3 | `FUN_022be9e8` | Projectile |

**Evidence:** `FUN_022bf160` iterates all 4 layers
```c
for (i = 0; i < 4; i++) {
    effect_id = move_animation->effect_layers[i];
    effect_anim = GetEffectAnimation(effect_id);
    if (effect_anim->anim_type == 5) {
        return true;  // Has type 5 effect
    }
}
```

**Evidence:** `PlayMoveAnimation` uses layer 2
```c
pmVar8 = GetMoveAnimation(move_id);
sVar4 = pmVar8->field_0x4;  // Layer 2 effect_id
```

**Evidence:** `FUN_022bed90` uses layer 1
```c
pmVar5 = GetMoveAnimation((uint)*param_1);
local_1cc[0] = (uint)pmVar5->field_0x2;  // Layer 1 effect_id
```

### flags (offset 0x08)

Bit field controlling animation behavior.

| Bits | Mask | Accessor | Purpose |
|------|------|----------|---------|
| 0-2 | 0x07 | `FUN_022bfd58` | Projectile wave pattern (0=straight, 1=vertical sine, 2=spiral) |
| 3 | 0x08 | `FUN_022bfd6c` | Dual-target effect (both attacker and target) |
| 4 | 0x10 | `FUN_022bfd8c` | Skip fade-in effect |
| 5 | 0x20 | `FUN_022bfdac` | Face direction + pre-animation delay |
| 6 | 0x40 | `FUN_022bfdcc` | Add post-animation delay |
| 7 | 0x80 | Unknown/unused |

**Bits 0-2: Projectile Wave Pattern**

Value is read by `FUN_022bfd58`, returned through the charge handler (`FUN_02324e78`) → `FUN_02322ddc` → `FUN_02322374`, and passed as `param_4` to `FUN_023230fc` (projectile motion handler).

| Value | Pattern | Description |
|-------|---------|-------------|
| 0 | Straight | No wave offset, direct line |
| 1 | Vertical Sine | Up-down oscillation perpendicular to travel |
| 2 | Spiral | Circular/helical motion using two angles |

Previously thought to be determined at runtime — confirmed stored directly in move flags.

**Bit 5: Face Direction + Pre-Animation Delay**

When set, `FUN_02304a48` is called to reset the attacker's facing direction, followed by `AnimationDelayOrSomething` to add a short pause before the main monster animation begins.

**Evidence:** `FUN_02322ddc`
```c
bVar7 = FUN_022bfdac((uint)*(ushort *)(param_2 + 4));
if (bVar7) {
    FUN_02304a48(param_1, (uint)*(byte *)(param_1[0x2d] + 0x4c));
    AnimationDelayOrSomething('\x01');
}
```

**Evidence:** Flag bit 3 check in `ExecuteMoveEffect`
```c
bVar7 = FUN_022bfd6c((uint)(ushort)move->id);
if (bVar7) {
    FUN_023258ec((int *)attacker, (int *)entity, (int)move, iVar25);  // Dual-target
} else {
    PlayMoveAnimation(attacker, entity, move, (position *)0x0);  // Single-target
}
```

**Evidence:** Flag bit 6 check in `FUN_023250d4`
```c
bVar3 = FUN_022bfdcc((uint)*(ushort *)(param_2 + 4));
if (bVar3) {
    AnimationDelayOrSomething('\x01');
}
```

### _padding (offsets 0x09-0x0B)

Three bytes of alignment padding. Always zero.

**Evidence:** Memory alignment for the following int32_t field.

### projectile_speed (offset 0x0C)

Controls projectile travel speed. Stored as int32_t (4 bytes).

| Raw Value | Mapped Speed | Frame Count | Description |
|-----------|--------------|-------------|-------------|
| 0 | 6 | 4 frames | Instant/fast |
| 1 | 2 | 12 frames | Slow |
| 2 | 3 | 8 frames | Medium |
| Other | 6 | 4 frames | Fast (default) |

**Evidence:** `FUN_023230fc`
```c
iVar7 = GetMoveAnimationSpeed((uint)(ushort)param_2[2]);
if (iVar7 == 1) {
    iVar7 = 2;
} else if (iVar7 == 2) {
    iVar7 = 3;
} else {
    iVar7 = 6;
}
uVar20 = _s32_div_f(0x18, iVar7);  // frame_count = 24 / speed
```

### monster_anim_type (offset 0x10)

Specifies which sprite animation the attacking monster performs.

**Standard Values:**

| Value | Animation | Example Moves |
|-------|-----------|---------------|
| 0 | Walk | - |
| 1 | Attack | Most attacks |
| 5 | Sleep | - |
| 6 | Hurt | - |
| 7 | Idle | - |
| 8 | Swing | Faint Attack, Thief, Flame Wheel |
| 9 | Double | Double Team, Agility |
| 10 | Hop | Dig, Bounce |
| 11 | Charge | Glare, Detect, Bulk Up |
| 12 | Rotate | Mud-Slap, throwing moves |

**Special Values:**

| Value | Behavior | Description |
|-------|----------|-------------|
| 98 (0x62) | Multi-direction | 9 attacks, direction += 2 each |
| 99 (0x63) | Spin | 8 directions sequentially |

**Type 98 Moves (Multi-direction):**

9 iterations, direction increments by 2 each time, 2 frames per direction (18 frames total).

| Move ID | Move Name |
|---------|-----------|
| 18 | Cut |
| 362 | UNKNOWN (possibly unused) |

**Type 99 Moves (Spin):**

8 iterations through all directions sequentially, 2 frames per direction (16 frames total).

| Move ID | Move Name |
|---------|-----------|
| 31 | Weather Ball |
| 166 | Egg Bomb |
| 213 | Mud Slap |
| 277 | Present |
| 285 | Bone Rush |
| 290 | Bonemerang |
| 309 | Mist Ball |
| 329 | Leech Seed |
| 408 | Item Toss |
| 486 | Seed Bomb |
| 501 | Mud Bomb |
| 504 | Worry Seed |
| 543-546 | UNKNOWN (likely system/dummy entries) |

> See `Systems/move_effect_pipeline.md` section "Special Monster Animation Types" for implementation details on types 98/99.

**Evidence:** Spin animation (99) in `FUN_023250d4`
```c
if (iVar9 == 99) {
    dVar19 = (direction_id)*(byte *)(iVar17 + 0x4c);
    FUN_02325644(&local_3c, param_1, param_2, uVar7);
    iVar9 = 0;
    do {
        dVar19 = dVar19 + DIR_NONE & 7;  // Increment direction
        ChangeMonsterAnimation((entity *)param_1, '\0', dVar19);
        FUN_022ea370(2, 0x15, dVar18, uVar7);  // 2 frames per direction
        iVar9 = iVar9 + 1;
    } while (iVar9 < 8);
}
```

**Evidence:** Multi-direction (98) in `FUN_023250d4`
```c
else if (iVar9 == 0x62) {
    uVar11 = (uint)*(byte *)(iVar17 + 0x4c);
    FUN_02325644(&local_3c, param_1, param_2, uVar7);
    iVar9 = 0;
    do {
        dVar18 = uVar11 & 7;
        ChangeMonsterAnimation((entity *)param_1, '\0', dVar18);
        FUN_022ea370(2, 0x15, dVar19, uVar7);
        iVar9 = iVar9 + 1;
        uVar11 = dVar18 + DIR_DOWN_RIGHT;  // Increment by 2
    } while (iVar9 < 9);
}
```

### attachment_point_idx (offset 0x11)

**SIGNED int8_t.** Index for position offset lookup.

| Value | Name | Description |
|-------|------|-------------|
| -1 | Default | No lookup, use sprite center |
| 0 | Head | Offset to head position |
| 1 | LeftHand | Offset to left hand |
| 2 | RightHand | Offset to right hand |
| 3 | Centre | Offset to body center |

**Evidence:** `FUN_022bf01c` reads and validates
```c
psVar2 = GetSpecialMonsterMoveAnimation((uint)pmVar1->field_0x16);
// ... loop to find override ...
return (int)pmVar1->field_0x11;  // Default attachment point
```

**Evidence:** `FUN_0201cf90` validates range
```c
if (3 < param_3) {
    return;  // Out of range
}
```

### sound_effect_id (offset 0x12)

Sound effect to play when animation triggers.

| Value | Meaning |
|-------|---------|
| 0x0000 | No sound |
| 0x3F00 | Silence constant |
| Other | Sound effect table index |

**Evidence:** `FUN_023250d4`
```c
uVar11 = FUN_022bf0f4((int)*(short *)(iVar17 + 4), (uint)uVar5);
if (uVar11 != 0x3f00) {
    PlaySeByIdIfNotSilence(uVar11 & 0xffff);
}
```

### override_count (offset 0x14)

Number of per-Pokemon overrides in the special_monster_move_animation table.

### override_table_idx (offset 0x16)

Starting index into special_monster_move_animation table.

## Per-Pokemon Override System

Allows different Pokemon species to have custom animation parameters for the same move.

### Override Table Structure
```c
struct special_monster_move_animation {
    int16_t sprite_id;      // 0x00: Monster sprite to match
    uint8_t anim_param;     // 0x02: Override for monster_anim_type
    uint8_t palette_param;  // 0x03: Override for attachment_point_idx
    uint16_t se_param;      // 0x04: Override for sound_effect_id
};
// Size: 6 bytes
```

**Evidence:** `GetSpecialMonsterMoveAnimation`
```c
special_monster_move_animation* GetSpecialMonsterMoveAnimation(int index) {
    return (special_monster_move_animation*)(index * 6 + DAT_022bfed8);
}
```

### Override Lookup

**Evidence:** `FUN_022bf01c` (attachment point override)
```c
psVar2 = GetSpecialMonsterMoveAnimation((uint)pmVar1->field_0x16);
iVar3 = 0;
while (true) {
    if (pmVar1->field_0x14 <= iVar3) {
        return (int)pmVar1->field_0x11;  // No override found, use default
    }
    if (psVar2->field_0x0 == (short)(uVar4 >> 0x20)) break;  // Match found
    iVar3 = (iVar3 + 1) * 0x10000 >> 0x10;
    psVar2 = psVar2 + 1;
}
return (int)psVar2->field_0x3;  // Return override value
```

Similar patterns exist for:
- `FUN_022bf0f4`: Sound effect override (returns `psVar2->field_0x4`)
- `FUN_022bfa3c`: Animation type override (returns `psVar2->field_0x2`)

## Accessor Function
```c
move_animation* GetMoveAnimation(move_id move_id) {
    return (move_animation*)(move_id * 0x18 + MOVE_ANIMATION_INFO);
}
```

## Animation ID Resolution

Before table lookup, move ID may be modified for weather or charging state.
```c
bVar5 = ShouldMovePlayAlternativeAnimation(user, move);
apparent_weather = GetApparentWeather(user);
final_id = GetMoveAnimationId(move, apparent_weather, bVar5);
move_anim = GetMoveAnimation(final_id);
```

### Weather Ball

Animation changes based on current weather (handled by `GetMoveAnimationId`).

### Two-Turn Moves

Charging turn uses alternative animation.

**Evidence:** `Is2TurnsMove`
```c
bool Is2TurnsMove(move_id move_id) {
    if (move_id == MOVE_SOLARBEAM) return true;
    if (move_id == MOVE_SKY_ATTACK) return true;
    if (move_id == MOVE_RAZOR_WIND) return true;
    if (move_id == MOVE_FOCUS_PUNCH) return true;
    if (move_id == MOVE_SKULL_BASH) return true;
    if (move_id == MOVE_FLY) return true;
    if (move_id == MOVE_BOUNCE) return true;
    if (move_id == MOVE_DIVE) return true;
    if (move_id == MOVE_DIG) return true;
    // ...
    return false;
}
```

## Special Move Cases

### Razor Leaf (Move ID 0x52 / 82)

Creates 8 separate effect instances in a spread pattern.

**Evidence:** `FUN_022bed90`
```c
if (*param_1 == 0x52) {
    for (iVar17 = 0; iVar17 < 8; iVar17++) {
        local_174[iVar17 * 0x16 + 1] += 0x40;  // Y offset +64
        local_174[iVar17 * 0x16 + 2] += sVar1;  // X offset from table
        iVar7 = FUN_022be780(1, local_180 + iVar17 * 0xb, 0);
    }
}
```

### Drain Punch / Leech Life

Uses attacker direction for effect orientation.

**Evidence:** `PlayMoveAnimation`
```c
if (move_id == MOVE_DRAIN_PUNCH || move_id == MOVE_LEECH_LIFE) {
    local_28 = *(byte *)((int)user->info + 0x4c) & 7;
}
```

## Open Questions

- What flag bit 7 (0x80) controls (if anything)
- Complete list of moves using flag bit 3 (dual-target)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `GetMoveAnimation` | - | Look up move_animation by ID |
| `GetMoveAnimationSpeed` | - | Read projectile_speed field |
| `GetMoveAnimationId` | - | Resolve ID with weather/charging |
| `GetSpecialMonsterMoveAnimation` | - | Look up override table entry |
| `FUN_022bfd58` | `0x022bfd58` | Read flag bits 0-2 |
| `FUN_022bfd6c` | `0x022bfd6c` | Read flag bit 3 |
| `FUN_022bfd8c` | `0x022bfd8c` | Read flag bit 4 |
| `FUN_022bfdac` | `0x022bfdac` | Read flag bit 5 |
| `FUN_022bfdcc` | `0x022bfdcc` | Read flag bit 6 |
| `FUN_022bf01c` | `0x022bf01c` | Get attachment point (with override) |
| `FUN_022bf0f4` | `0x022bf0f4` | Get sound effect (with override) |
| `FUN_022bfa3c` | `0x022bfa3c` | Get animation type (with override) |
| `ShouldMovePlayAlternativeAnimation` | - | Check two-turn move charging |
| `GetApparentWeather` | - | Get weather for Weather Ball |
| `Is2TurnsMove` | - | Check if move is two-turn |
