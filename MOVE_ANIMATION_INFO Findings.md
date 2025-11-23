# Reverse Engineering Findings: MOVE_ANIMATION_INFO

## Overview

The `MOVE_ANIMATION_INFO` table defines visual animations, sound effects, and behavioral parameters for all moves in the game.

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
    int16_t effect_id_0;         // 0x0:  Effect animation layer 0
    int16_t effect_id_1;         // 0x2:  Effect animation layer 1 (secondary)
    int16_t effect_id_2;         // 0x4:  Effect animation layer 2 (main/primary)
    int16_t effect_id_3;         // 0x6:  Effect animation layer 3
    uint8_t flags;               // 0x8:  Behavior flags (see Flags section)
    uint8_t padding[3];          // 0x9:  UNUSED - Memory alignment padding
    int32_t animation_speed;     // 0xC:  Projectile/animation speed (1, 2, or other)
    uint8_t anim_type;           // 0x10: Monster animation type (maps to Animation IDs)
    int8_t palette_param;        // 0x11: Position offset index (0-3)
    uint16_t se_id;              // 0x12: Sound effect ID (0x3F00 = silence)
    int16_t spc_anim_count;      // 0x14: Number of sprite-specific overrides
    uint16_t spc_anim_index;     // 0x16: Index into special_monster_move_animation table
};
// Size: 0x18 (24 bytes)
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
- `FUN_022bfc5c`: Reads `field_0x6` for tertiary effects

Usage Pattern:
- Layer 0 (0x00): Often used for charge-up or preparation effects
- Layer 1 (0x02): Secondary impact or follow-up effects, checked by `FUN_02325644` for non-zero value
- Layer 2 (0x04): Primary visual effect (always used if move has animation)
- Layer 3 (0x06): Additional effects or projectile visuals

Special Cases:
- Value `0x0000` indicates no effect for that layer
- Move ID `0x52` (Razor Leaf, decimal 82) creates 8 separate animation instances using layer 1 data

Layer Check Example:
```c
// From FUN_023250d4 and FUN_02325644
if (pmVar10->field_0x2 == 0) {
    // No secondary animation
} else {
    // Spawn secondary effect from layer 1
    FUN_02325644(&local_3c, param_1, param_2, uVar7);
}
```

---

### Offset 0x08: Behavior Flags

Type: `uint8_t` (bit field)

Purpose: Controls various animation behaviors through individual bit flags.

Evidence:
- `FUN_022bfd58`: Extracts bits 0-2 with mask `0x07`, returns animation category (0-7)
- `FUN_022bfd6c`: Checks bit 3 with mask `0x08`
- `FUN_022bfd8c`: Checks bit 4 with mask `0x10`, controls fade-in behavior in `FUN_02324e78`
- `FUN_022bfdac`: Checks bit 5 with mask `0x20`
- `FUN_022bfdcc`: Checks bit 6 with mask `0x40`, triggers `AnimationDelayOrSomething()` in `FUN_023250d4`
- `FUN_02324e78`: Uses bit 4 check result to determine whether to perform screen fade-in effect

Bit Mapping:

| Bits | Mask | Purpose | Evidence Function |
|------|------|---------|-------------------|
| 0-2 | `0x07` | Animation category (0-7) | `FUN_022bfd58` |
| 3 | `0x08` | Unknown flag | `FUN_022bfd6c` |
| 4 | `0x10` | Skip fade-in effect (if set) | `FUN_022bfd8c`, `FUN_02324e78` |
| 5 | `0x20` | Unknown flag | `FUN_022bfdac` |
| 6 | `0x40` | Add post-animation delay (if set) | `FUN_022bfdcc`, `FUN_023250d4` |
| 7 | `0x80` | Unknown/unused | - |

Bit 6 Implementation Example:
```c
// From FUN_023250d4
bVar3 = FUN_022bfdcc((uint)*(ushort *)(param_2 + 4));
if (bVar3) {
    AnimationDelayOrSomething('\x01');
}
```

Bit 4 Implementation Example:
```c
// From FUN_02324e78
bVar6 = FUN_022bfd8c((uint)move_id);  // Check bit 4
if (!bVar6) {
    // Perform fade-in effect
    while (brightness < 0xFF) {
        brightness += 0x20;
        FUN_022ed0d4(brightness);
        AdvanceFrame('&');
    }
}
```

---

### Offset 0x09-0x0B: Padding Bytes

Type: Three `uint8_t` values

Purpose: Memory alignment padding (unused)

Evidence:
- Memory inspection shows these bytes are consistently zero across all move entries
- No decompiled functions read or write to offsets `0x022C906D`, `0x022C906E`, or `0x022C906F`
- Ghidra scalar search for these addresses returns zero results
- Standard C compiler behavior adds 3 bytes of padding to align the following 4-byte integer (`animation_speed`) to a 4-byte boundary

Memory Layout Rationale:
```
0x8:  uint8_t (1 byte)
0x9:  padding (3 bytes) ← Ensures next field starts at 4-byte boundary
0xC:  int32_t (4 bytes) ← Must be 4-byte aligned
```

---

### Offset 0x0C: Animation Speed

Type: `int32_t` (spans 0x0C-0x0F)

Purpose: Controls the speed of projectile animations and frame timing.

Evidence:
- `GetMoveAnimationSpeed`: Reads from address `0x022C9070` (confirmed via Ghidra: `DAT_022bff2c = 0x022C9070`), which equals `MOVE_ANIMATION_INFO + 0xC`
- Returns a 4-byte integer, not a single byte
- `FUN_023230fc`: Uses return value to calculate frame count via division: `uVar20 = _s32_div_f(0x18, iVar7)` (24 / speed)
- `FUN_0232393c`: Similar speed-based frame calculation: `uVar19 = _s32_div_f(0x18, iVar3)`

Speed Mapping:

| Raw Value | Mapped To | Frame Count | Description |
|-----------|-----------|-------------|-------------|
| 1 | 2 | 12 frames | Slow projectile |
| 2 | 3 | 8 frames | Medium projectile |
| Other | 6 | 4 frames | Fast projectile (default) |

Implementation:
```c
// From FUN_023230fc
iVar7 = GetMoveAnimationSpeed((uint)(ushort)param_2[2]);
if (iVar7 == 1) {
    iVar7 = 2;
} else if (iVar7 == 2) {
    iVar7 = 3;
} else {
    iVar7 = 6;
}
uVar20 = _s32_div_f(0x18, iVar7);  // Calculate frame count
```

---

### Offset 0x10: Monster Animation Type

Type: `uint8_t`

Purpose: Specifies which sprite animation the attacking monster should perform during the move.

Evidence:
- `FUN_022bfa3c`: Reads this field via `pmVar1->field_0x10` and returns it (or override value from special animation table)
- `FUN_023250d4`: Stores result of `FUN_022bfa3c` in `iVar9`, checks for special values `99` (0x63) and `0x62` (98) which trigger unique animation sequences
- `ChangeMonsterAnimation`: Called with this value to set the monster's sprite animation: `ChangeMonsterAnimation((entity *)param_1, (int8_t)iVar9, direction)`

Standard Values (Maps to PMD Sprite Repository Animation IDs):

| Value | Animation | Behavior |
|-------|-----------|----------|
| 0 | Walk | Basic movement |
| 1 | Attack | Standard attack animation |
| 5 | Sleep | Sleep status |
| 6 | Hurt | Taking damage |
| 7 | Idle | Standing idle |
| 8 | Swing | Special attack (Faint Attack, Thief, Flame Wheel) |
| 9 | Double | Evasive moves (Double Team, Agility, Evasion Orb) |
| 10 | Hop | Jump moves (Dig, Bounce) |
| 11 | Charge | Stat boost/status moves (Glare, Detect, Bulk Up) |
| 12 | Rotate | Throwing/projectile moves (Mud-Slap, items) |

Special Values:

| Value | Behavior |
|-------|----------|
| 99 (0x63) | Spin through all 8 directions sequentially |
| 98 (0x62) | Attack in 9 directions (incrementing by 2 each time) |

Special Behavior (Value 99):
```c
// From FUN_023250d4
if (iVar9 == 99) {
    dVar19 = (direction_id)*(byte *)(iVar17 + 0x4c);
    FUN_02325644(&local_3c, param_1, param_2, uVar7);
    iVar9 = 0;
    do {
        dVar19 = dVar19 + DIR_NONE & 7;  // Increment direction
        dVar18 = dVar19;
        ChangeMonsterAnimation((entity *)param_1, '\0', dVar19);
        FUN_022ea370(2, 0x15, dVar18, uVar7);
        iVar9 = iVar9 + 1;
    } while (iVar9 < 8);  // Loop 8 times
}
```

Special Behavior (Value 98/0x62):
```c
// From FUN_023250d4
else if (iVar9 == 0x62) {
    uVar11 = (uint)*(byte *)(iVar17 + 0x4c);
    FUN_02325644(&local_3c, param_1, param_2, uVar7);
    iVar9 = 0;
    do {
        dVar18 = uVar11 & 7;
        dVar19 = dVar18;
        ChangeMonsterAnimation((entity *)param_1, '\0', dVar18);
        FUN_022ea370(2, 0x15, dVar19, uVar7);
        iVar9 = iVar9 + 1;
        uVar11 = dVar18 + DIR_DOWN_RIGHT;  // Increment by 2
    } while (iVar9 < 9);  // Loop 9 times
}
```

---

### Offset 0x11: Position Offset Parameter

Type: `int8_t` (signed)

Purpose: Index (0-3) into a position offset lookup table. Controls where effects spawn relative to the monster sprite.

Evidence:
- `FUN_022bf01c`: Reads `pmVar1->field_0x11` and uses return value with `FUN_0201cf90`, can be overridden by special_monster_move_animation table
- `FUN_0201cf90`: Takes this parameter as `param_3`, validates range (returns early if `param_3 > 3`), then uses it to calculate position:
  ```c
  iVar2 = iVar2 + iVar3 * 0x10;  // Base offset for sprite
  iVar3 = param_3 * 4;           // Convert index to byte offset
  psVar1 = (short *)(iVar2 + iVar3);  // Get offset pair
  ```
- Returns two `int16_t` values (x_offset, y_offset) to `param_1`
- Special case: If lookup returns `(99, 99)`, function hardcodes to `(0x63, 0x63)` instead

Parameter Range: 0-3 (values outside this range cause early return with zeros)

Usage Pattern:
```c
// From FUN_023250d4
uVar11 = FUN_022bf01c((int)*(short *)(iVar13 + 4), (uint)uVar6);
if (uVar11 == 0xffffffff) {
    // Use default position
    sStack_34 = *(short *)(DAT_02325608 + 4);
    uStack_32 = *(undefined2 *)(DAT_02325608 + 6);
} else {
    // Calculate position using parameter
    uVar11 = uVar11 & 0xff;
    FUN_0201cf90(&sStack_34, (ushort *)(param_1 + 0xb), uVar11);
}
```

---

### Offset 0x12: Sound Effect ID

Type: `uint16_t`

Purpose: References the game's sound effect table. Plays when the move animation triggers.

Evidence:
- `FUN_022bf0f4`: Reads `pmVar1->field_0x12` and returns it as `(uint)pmVar1->field_0x12` (or override value from special animation table)
- `FUN_023250d4`: Stores result in `uVar11`, checks if value equals `0x3F00` (silence constant), calls `PlaySeByIdIfNotSilence(uVar11 & 0xffff)` if not silence

Special Values:
- `0x3F00`: Represents silence/no sound effect (checked explicitly before playing)
- `0x0000`: May also indicate no sound in some contexts
- Other values: Direct index into sound effect table

Implementation:
```c
// From FUN_023250d4
uVar11 = FUN_022bf0f4((int)*(short *)(iVar17 + 4), (uint)uVar5);
if (uVar11 != 0x3f00) {
    PlaySeByIdIfNotSilence(uVar11 & 0xffff);
}
```

---

### Offset 0x14-0x16: Special Animation Override System

Type: `int16_t spc_anim_count`, `uint16_t spc_anim_index`

Purpose: Allows different Pokémon species to have custom animation parameters for the same move.

Evidence:
- `GetSpecialMonsterMoveAnimation`: Returns pointer calculated as `(ent_id * 6 + DAT_022bfed8)`, confirming 6-byte structure
- `FUN_022bf01c`: Reads `pmVar1->field_0x16` to get index, then `pmVar1->field_0x14` for loop count:
  ```c
  psVar2 = GetSpecialMonsterMoveAnimation((uint)pmVar1->field_0x16);
  iVar3 = 0;
  while (true) {
      if (pmVar1->field_0x14 <= iVar3) {
          return (int)pmVar1->field_0x11;  // No override found
      }
      if (psVar2->field_0x0 == (short)(uVar4 >> 0x20)) break;  // Match found
      iVar3 = (iVar3 + 1) * 0x10000 >> 0x10;
      psVar2 = psVar2 + 1;
  }
  return (int)psVar2->field_0x3;  // Return override value
  ```
- `FUN_022bf0f4`: Similar loop structure for sound effect overrides, returns `psVar2->field_0x4`
- `FUN_022bfa3c`: Similar loop structure for animation type overrides, returns `psVar2->field_0x2`

Override Table Structure:
```c
struct special_monster_move_animation {
    int16_t sprite_id;      // 0x0: Monster sprite to match (compared via division result)
    uint8_t anim_param;     // 0x2: Override for anim_type (field_0x10)
    uint8_t palette_param;  // 0x3: Override for palette_param (field_0x11)
    uint16_t se_param;      // 0x4: Override for se_id (field_0x12)
};
// Size: 6 bytes (confirmed by GetSpecialMonsterMoveAnimation calculation)
```

Lookup Algorithm:
```c
// Generalized from FUN_022bf01c, FUN_022bf0f4, FUN_022bfa3c
psVar2 = GetSpecialMonsterMoveAnimation((uint)pmVar1->field_0x16);
uVar4 = _s32_div_f(user_sprite_param, 600);  // Normalize sprite ID

for (iVar3 = 0; iVar3 < pmVar1->field_0x14; iVar3++) {
    if (psVar2[iVar3].sprite_id == (short)(uVar4 >> 0x20)) {
        return psVar2[iVar3].override_value;  // Use override
    }
}
return default_value;  // Use base move_animation value
```

Use Cases:
- Same move appears different colors on different Pokémon (via palette_param override)
- Different sound effects for the same move based on user species
- Alternative animation types for signature moves

Example: Move 0x128 (296 decimal) in `FUN_023250d4` uses this system to randomize effects.

---

## Function Reference

### Core Accessor Functions

| Function | Purpose |
|----------|---------|
| `GetMoveAnimation` | Returns pointer to move_animation entry at `base + (move_id * 0x18)` |
| `GetMoveAnimationSpeed` | Reads animation_speed field at offset 0xC |
| `GetMoveAnimationId` | Resolves move ID with weather/alternative animation checks |
| `GetSpecialMonsterMoveAnimation` | Returns pointer to override table entry at `base + (index * 6)` |

### Field Readers (Flags)

| Function | Reads | Returns |
|----------|-------|---------|
| `FUN_022bfd58` | `flags & 0x07` | Animation category (0-7) |
| `FUN_022bfd6c` | `flags & 0x08` | Bit 3 flag |
| `FUN_022bfd8c` | `flags & 0x10` | Skip fade-in flag |
| `FUN_022bfdac` | `flags & 0x20` | Bit 5 flag |
| `FUN_022bfdcc` | `flags & 0x40` | Post-delay flag |

### Field Readers (Override System)

| Function | Reads | Returns |
|----------|-------|---------|
| `FUN_022bf01c` | `palette_param` (0x11) | Position parameter (or override from table offset 0x3) |
| `FUN_022bf0f4` | `se_id` (0x12) | Sound effect ID (or override from table offset 0x4) |
| `FUN_022bfa3c` | `anim_type` (0x10) | Animation type (or override from table offset 0x2) |

### Effect Animation Handlers

| Function | Purpose | Layer(s) Used |
|----------|---------|---------------|
| `FUN_022bf160` | Checks if any of 4 effect layers uses type 5 | All (0x0, 0x2, 0x4, 0x6) |
| `FUN_022bf1fc` | Validates specific effect layer by param_2 index | Indexed access |
| `FUN_022bfaa8` | Spawns effect animation | Layer 0 (0x0) |
| `FUN_022bed90` | Spawns secondary effect, handles Razor Leaf case | Layer 1 (0x2) |
| `FUN_022bfc5c` | Spawns tertiary effect | Layer 2 (0x4) |
| `FUN_022be9e8` | Spawns projectile/additional effect | Layer 3 (0x6) |

### Animation Orchestrators

| Function | Purpose |
|----------|---------|
| `PlayMoveAnimation` | Main entry point, coordinates entire animation sequence |
| `FUN_023250d4` | Handles monster animation, special behaviors (99, 0x62), secondary effects |
| `FUN_02325644` | Triggers secondary effects when field_0x2 != 0, calls FUN_022bed90 |
| `FUN_02325d7c` | Checks for type 5 effects, sets up screen effects |
| `FUN_023230fc` | Projectile movement logic with wave patterns, uses animation_speed |
| `FUN_0232393c` | Alternate projectile handler, also uses animation_speed |
| `FUN_02324e78` | Screen-wide effects, fade-in handling based on bit 4 flag |
| `FUN_022e6d68` | Attaches animation instance to entity for tracking |

### Helper Functions

| Function | Purpose |
|----------|---------|
| `FUN_0201cf90` | Position offset calculation, takes palette_param (0-3) as index |
| `ShouldMovePlayAlternativeAnimation` | Checks for two-turn move charging state |
| `GetApparentWeather` | Gets current weather for Weather Ball logic |
| `AnimationDelayOrSomething` | Adds post-animation delay when flag bit 6 set |
| `ChangeMonsterAnimation` | Sets monster sprite animation based on anim_type |

---

## Special Move Behaviors

### Move 0x52 (Razor Leaf, decimal 82)

Evidence: `FUN_022bed90`

Creates 8 separate animation instances in a spread pattern:

```c
if (*param_1 == 0x52) {
    // ... initialization ...
    for (iVar17 = 0; iVar17 < 8; iVar17++) {
        // Set up position with offsets
        local_174[iVar17 * 0x16 + 1] = param_1[3];
        local_174[iVar17 * 0x16 + 1] = local_174[iVar17 * 0x16 + 1] + 0x40;  // Y offset
        local_174[iVar17 * 0x16 + 2] = param_1[4];
        local_174[iVar17 * 0x16 + 2] = local_174[iVar17 * 0x16 + 2] + sVar1;  // X offset
        
        // Create animation instance
        iVar7 = FUN_022be780(1, local_180 + iVar17 * 0xb, 0);
        
        if (iVar17 == 0) iVar6 = iVar7;  // Remember first handle
    }
    return iVar6;
}
```

Behavior:
- Uses `field_0x2` (effect_id_1) for all 8 instances
- Each instance offset by `+0x40` pixels on Y-axis
- Each instance receives calculated X offset from position table
- First instance handle returned for animation tracking

### Monster Animation Value 99 (Spin Attack)

Evidence: `FUN_023250d4`

Triggers spinning behavior where monster rotates through all 8 directions:

```c
if (iVar9 == 99) {
    bVar4 = ShouldDisplayEntityAdvanced(peVar15);
    if (bVar4 != '\0') {
        dVar19 = (direction_id)*(byte *)(iVar17 + 0x4c);
        FUN_02325644(&local_3c, param_1, param_2, uVar7);
        iVar9 = 0;
        do {
            dVar19 = dVar19 + DIR_NONE & 7;  // Increment, wrap at 8
            dVar18 = dVar19;
            ChangeMonsterAnimation((entity *)param_1, '\0', dVar19);
            FUN_022ea370(2, 0x15, dVar18, uVar7);
            iVar9 = iVar9 + 1;
        } while (iVar9 < 8);
    }
}
```

Used by moves that involve spinning in place (e.g., Rapid Spin).

### Monster Animation Value 0x62 (Multi-Directional Attack)

Evidence: `FUN_023250d4`

Triggers 9 sequential attacks in different directions:

```c
else if (iVar9 == 0x62) {
    bVar4 = ShouldDisplayEntityAdvanced(peVar15);
    if (bVar4 != '\0') {
        uVar11 = (uint)*(byte *)(iVar17 + 0x4c);
        FUN_02325644(&local_3c, param_1, param_2, uVar7);
        iVar9 = 0;
        do {
            dVar18 = uVar11 & 7;
            dVar19 = dVar18;
            ChangeMonsterAnimation((entity *)param_1, '\0', dVar18);
            FUN_022ea370(2, 0x15, dVar19, uVar7);
            iVar9 = iVar9 + 1;
            uVar11 = dVar18 + DIR_DOWN_RIGHT;  // Add 2 (diagonal increment)
        } while (iVar9 < 9);
    }
}
```

Used by area-of-effect moves that hit in multiple directions (possibly Explosion or Discharge).

### Move 0x128 (296 decimal) Special Handling

Evidence: `FUN_023250d4`

Uses randomization and special global state:

```c
if (*(short *)(param_2 + 4) == 0x128) {
    uVar12 = DungeonRandInt(7);  // Random 0-6
    puVar2 = DAT_02325614;
    piVar1 = DAT_0232560c;
    iVar13 = *DAT_0232560c;
    uVar14 = *(undefined4 *)(DAT_02325610 + uVar12 * 4);
    *DAT_02325614 = uVar12;
    *(undefined4 *)(iVar13 + 0x1a234) = uVar14;
    *(undefined4 *)(*piVar1 + 0x1a238) = *(undefined4 *)(*piVar1 + 0x1a234);
    SetMessageLogPreprocessorArgsNumberVal('\0', *puVar2 + 4);
    LogMessageByIdWithPopupCheckUser(peVar15, DAT_02325618);
    PlaySeByIdIfShouldDisplayEntity(peVar15, 0x214);
}
```

Randomly selects from 7 possible variants and displays a message.

### Move 0x76 (118 decimal) Special Handling

Evidence: `FUN_023250d4`

Sets specific global state value:

```c
else if (*(short *)(param_2 + 4) == 0x76) {
    *(undefined4 *)(*DAT_0232560c + 0x1a234) = 0xc;
    *(undefined4 *)(*piVar1 + 0x1a238) = *(undefined4 *)(*piVar1 + 0x1a234);
    PlaySeByIdIfShouldDisplayEntity(peVar15, 0x214);
}
```

---

## Projectile Motion System

Evidence: `FUN_023230fc`, `FUN_0232393c`

The animation_speed field controls projectile behavior through mathematical motion calculations.

### Frame Count Calculation
```c
// From FUN_023230fc
iVar7 = GetMoveAnimationSpeed((uint)(ushort)param_2[2]);
if (iVar7 == 1) iVar7 = 2;
else if (iVar7 == 2) iVar7 = 3;
else iVar7 = 6;

uVar20 = _s32_div_f(0x18, iVar7);  // 24 / speed = frame count
```

### Motion Patterns (param_4)

Pattern 1 (param_4 == 1): Simple Vertical Wave
```c
if (param_4 == 1) {
    local_98 = SinAbs4096(iVar14);
    local_98 = iVar17 * local_98;  // amplitude * sin(angle)
    iVar13 = 0;
}
```
Creates gentle up-down wobbling motion.

Pattern 2 (param_4 == 2): Complex Circular/Spiral
```c
else if (param_4 == 2) {
    iVar13 = SinAbs4096(iVar14);
    iVar13 = (iVar17 >> 1) * iVar13 >> 8;
    local_98 = SinAbs4096(x);
    local_98 = iVar13 * local_98;
    iVar9 = CosAbs4096(x);
    iVar13 = iVar13 * iVar9;
}
```
Creates circular or spiral motion using dual sine/cosine calculation.

### Position Update Per Frame
```c
// Direction-based movement
sVar3 = *(short *)(DAT_023238f8 + direction * 4);  // X delta
sVar4 = *(short *)(DAT_023238fc + direction * 4);  // Y delta

// Apply speed multiplier
local_8c = local_8c + sVar3 * iVar7;
iVar8 = iVar8 + sVar4 * iVar7;

// Increment angle for wave calculation
iVar19 = iVar19 + (int)uVar21;
```

### Amplitude Calculation
```c
// From FUN_023230fc
if (param_3 < 2) {
    iVar17 = 0x20;  // Normal amplitude
} else {
    iVar17 = 8;     // Reduced amplitude
}
```

---

## Data Validation Notes

### Confirmed Behaviors
- All four effect_id fields (0x00-0x06) are valid references to EFFECT_ANIMATION_INFO table
- Offset 0x0C is a 4-byte integer (int32_t), not a single byte
- Behavior flags use individual bits for different purposes, not a combined value
- Override system allows per-species customization of sounds, positions, and animations
- Padding bytes 0x09-0x0B are unused (all zeros in memory)
- Special animation table has 6-byte entries

### Unconfirmed Elements
- Exact purpose of flag bits 3 and 5 (masks 0x08 and 0x20)
- Full mapping of animation_category values (bits 0-2, range 0-7)
- Complete list of moves using multi-layer effects
- Exact meaning of special moves 0x128 and 0x76

---

## Version Information

ROM Version: NA release  
Analysis Date: 2025  

Credits:
- Reverse engineering: Analysis of Ghidra decompiled code
- Memory inspection: Manual verification of structure layout
- Documentation: Comprehensive function tracing and evidence compilation
