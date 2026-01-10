# Move Effect Pipeline

## Summary

- `ExecuteMoveEffect` is the main entry point for move execution and animations
- `PlayMoveAnimation` handles standard single-target effect playback
- `FUN_023258ec` handles dual-target effects (flag bit 3)
- Four effect layers can play simultaneously (charge, secondary, primary, projectile)
- Special monster animations 98/99 trigger multi-directional attacks with unique spawning behavior
- Projectile spawning follows a specific call chain from move execution to motion handling

## Pipeline Overview
```
ExecuteMoveEffect
    │
    ├─► Check flag bit 3 (FUN_022bfd6c)
    │       │
    │       ├─► TRUE: FUN_023258ec (dual-target)
    │       │
    │       └─► FALSE: PlayMoveAnimation (single-target)
    │
    ├─► Layer handlers spawn effects
    │       ├─► Layer 0: FUN_022bfaa8 (charge/prep)
    │       ├─► Layer 1: FUN_022bed90 (secondary)
    │       ├─► Layer 2: FUN_022bfc5c (primary)
    │       └─► Layer 3: FUN_022be9e8 (projectile)
    │
    └─► Projectile motion: FUN_023230fc / FUN_0232393c
```

## ExecuteMoveEffect

Main entry point that orchestrates move execution and animation.

**Evidence:** `ExecuteMoveEffect` (partial)
```c
void ExecuteMoveEffect(target_list *targets, entity *attacker, move *move, ...)
{
    // ... target iteration and damage calculation ...
    
    bVar8 = ShouldDisplayEntityWrapper(entity);
    if (bVar8 != '\0') {
        // Pre-animation delay
        FUN_022ea370(4, 0x4a, pmVar22, iVar25);
        
        // Check flag bit 3 for dual-target path
        bVar7 = FUN_022bfd6c((uint)(ushort)move->id);
        if (bVar7) {
            // Dual-target animation (e.g., Drain Punch)
            FUN_023258ec((int *)attacker, (int *)entity, (int)move, iVar25);
        }
        else {
            // Special delay for move 0xAD (173)
            if (move->id == (move_id_16)0xad) {
                AnimationDelayOrSomething('\x01');
            }
            
            // Standard single-target animation
            if ((local_e0 == (entity *)0x0) ||
               (move->id != (move_id_16)0x1f4 && move->id != (move_id_16)0x50)) {
                PlayMoveAnimation(attacker, entity, move, (position *)0x0);
            }
            else {
                PlayMoveAnimation(attacker, local_e0, move, (position *)0x0);
            }
        }
    }
    
    // ... post-animation effects ...
}
```

## PlayMoveAnimation

Standard single-target move animation handler.

**Evidence:** `PlayMoveAnimation`
```c
void PlayMoveAnimation(entity *user, entity *target, move *move, position *position)
{
    // 1. Resolve animation ID (weather, charging state)
    bVar5 = ShouldMovePlayAlternativeAnimation(user, move);
    apparent_weather = GetApparentWeather(user);
    uVar7 = GetMoveAnimationId((move *)(uint)(ushort)move->id, apparent_weather, bVar5 != '\0');
    move_id = (move_id)uVar7;
    
    // 2. Get move animation data
    pmVar8 = GetMoveAnimation(move_id);
    sVar4 = pmVar8->field_0x4;  // Layer 2 effect_id
    
    // 3. Validate target
    bVar5 = EntityIsValid(target);
    if (bVar5 == '\0') {
        iVar9 = FUN_022e2ca0(&position->x);
        if (iVar9 == 0) return;
    }
    else {
        pvVar14 = target->info;
        bVar5 = ShouldDisplayEntityAdvanced(target);
        if (bVar5 == '\0') return;
    }
    
    // 4. Skip if no effect
    if (sVar4 == 0) return;
    
    // 5. Get attachment point from effect animation
    peVar10 = GetEffectAnimation((int)sVar4);
    bVar3 = peVar10->field_0x19;  // attachment_point
    
    // 6. Calculate spawn position
    if (bVar3 != 0xff && EntityIsValid(target)) {
        FUN_0201cf90((short *)&local_2c, &(target->anim_ctrl).some_bitfield, (uint)bVar3);
    }
    
    // 7. Set up position parameters
    if (pvVar14 == NULL) {
        // Position-based (no target entity)
        local_30 = (tile_x * 24 + 12);
        local_2e = (tile_y * 24 + 16);
    }
    else {
        // Entity-based
        local_30 = target->pixel_pos.x >> 8;
        local_2e = target->pixel_pos.y >> 8;
        
        // Special direction for Drain Punch / Leech Life
        if (move_id == MOVE_DRAIN_PUNCH || move_id == MOVE_LEECH_LIFE) {
            local_28 = *(byte *)(user->info + 0x4c) & 7;
        }
    }
    
    // 8. Check for type 5 screen effects
    bVar6 = FUN_022bf160(&local_34);
    if (bVar6) {
        FUN_0234ba54(0x5d, ...);
        AdvanceFrame(']');
    }
    
    // 9. Spawn Layer 2 effect
    uVar11 = FUN_022bfc5c(&local_34);
    AdvanceFrame('[');
    
    // 10. Attach to target
    FUN_022e6d68(uVar11, target, 6);
    
    // 11. Wait for completion
    while (AnimationHasMoreFrames((int)(short)uVar11)) {
        AdvanceFrame('(');
    }
}
```

## Dual-Target Handler (FUN_023258ec)

Handles moves that show effects on both attacker AND target (flag bit 3).

**Evidence:** `FUN_023258ec`
```c
void FUN_023258ec(int *attacker, int *target, int move, undefined4 param_4)
{
    // 1. Resolve animation
    bVar4 = ShouldMovePlayAlternativeAnimation((entity *)attacker, (move *)move);
    apparent_weather = GetApparentWeather((entity *)attacker);
    uVar6 = GetMoveAnimationId(...);
    pmVar7 = GetMoveAnimation((uint)uVar6);
    sVar1 = pmVar7->field_0x4;  // Layer 2 effect_id
    
    // 2. Validate both entities
    if (!EntityIsValid(target) && !EntityIsValid(attacker)) return;
    
    // 3. Get attachment point offsets for BOTH entities
    peVar9 = GetEffectAnimation((int)sVar1);
    if (peVar9->field_0x19 != 0xff) {
        uVar12 = (uint)(byte)peVar9->field_0x19;
        FUN_0201cf90((short *)local_30, (ushort *)(target + 0xb), uVar12);   // Target offset
        FUN_0201cf90((short *)local_44, (ushort *)(attacker + 0xb), uVar12); // Attacker offset
    }
    
    // 4. Set up position parameters for both
    local_34 = target_pixel_y;
    local_38 = CONCAT(target_id, move_id);
    local_48 = attacker_pixel_y;
    local_4c = CONCAT(attacker_id, move_id);
    
    // 5. Check for type 5 screen effects
    bVar5 = FUN_022bf160((ushort *)&local_38);
    if (bVar5) {
        FUN_0234ba54(0x5d, ...);
        AdvanceFrame(']');
    }
    
    // 6. Spawn effect on BOTH entities
    uVar11 = FUN_022bfc5c((ushort *)&local_38);  // Target effect
    uVar10 = FUN_022bfc5c((ushort *)&local_4c);  // Attacker effect
    
    AdvanceFrame('[');
    
    // 7. Attach both effects
    FUN_022e6d68(uVar11, target, 6);
    FUN_022e6d68(uVar10, attacker, 6);
    
    // 8. Wait for completion (only checks target effect)
    while (AnimationHasMoreFrames((int)(short)uVar11)) {
        AdvanceFrame('(');
    }
}
```

**Used by:** Drain Punch, Leech Life, and other HP-draining moves.

## Effect Layers

Moves can use up to 4 effect layers simultaneously.

### Layer Structure in move_animation_info

| Layer | Offset | Field | Handler Function | Purpose |
|-------|--------|-------|------------------|---------|
| 0 | 0x00 | field_0x0 | `FUN_022bfaa8` | Charge/preparation |
| 1 | 0x02 | field_0x2 | `FUN_022bed90` | Secondary/multi-hit |
| 2 | 0x04 | field_0x4 | `FUN_022bfc5c` | Primary visual |
| 3 | 0x06 | field_0x6 | `FUN_022be9e8` | Projectile |

### Layer 0: Charge/Preparation (FUN_022bfaa8)

**Evidence:** `FUN_022bfaa8`
```c
void FUN_022bfaa8(ushort *param_1)
{
    // Copy template from DAT_022bfb64
    memcpy(local_38, DAT_022bfb64, ...);
    
    // Get move animation
    pmVar3 = GetMoveAnimation((uint)*param_1);
    
    // Read Layer 0 effect_id
    local_38[0] = (uint)pmVar3->field_0x0;
    
    // Copy position data from params
    local_38[1] = *(uint *)(param_1 + 8);
    local_38[2] = *(uint *)(param_1 + 6);
    local_2c = param_1[2];  // move_id
    // ... more position fields ...
    
    // Get attachment point from effect animation
    pmVar3 = GetMoveAnimation((uint)*param_1);
    peVar4 = GetEffectAnimation((int)pmVar3->field_0x0);
    local_24[0] = peVar4->field_0x19;  // attachment_point
    
    // Spawn effect with dispatch type 5
    FUN_022be780(5, local_38, 0);
}
```

### Layer 1: Secondary Effects (FUN_022bed90)

Handles secondary effects and special cases like Razor Leaf.

**Evidence:** `FUN_022bed90` (Razor Leaf case)
```c
int FUN_022bed90(ushort *param_1, ...)
{
    if (*param_1 == 0x52) {  // Razor Leaf (move ID 82)
        // Create 8 separate effect instances
        for (iVar17 = 0; iVar17 < 8; iVar17++) {
            // Get Layer 1 effect_id
            pmVar5 = GetMoveAnimation((uint)*param_1);
            local_180[iVar17 * 0xb] = (int)pmVar5->field_0x2;
            
            // Offset each instance
            local_174[iVar17 * 0x16 + 1] += 0x40;  // Y offset +64 pixels
            local_174[iVar17 * 0x16 + 2] += sVar1;  // X offset from table
            
            // Get attachment point
            iVar7 = FUN_022bf088((int)(short)param_1[1], (uint)*param_1);
            local_16c[iVar15] = (char)iVar7;
            
            // Spawn with dispatch type 1
            iVar7 = FUN_022be780(1, local_180 + iVar17 * 0xb, 0);
            
            // Set flag at context + 0x136
            if (iVar15 != -1) {
                *(context + 0x136) = 6;
            }
            
            if (iVar17 == 0) iVar6 = iVar7;  // Remember first handle
        }
        return iVar6;
    }
    else {
        // Standard secondary effect
        pmVar5 = GetMoveAnimation((uint)*param_1);
        local_1cc[0] = (uint)pmVar5->field_0x2;
        // ... setup ...
        return FUN_022be780(1, local_1cc, 0);
    }
}
```

### Layer 2: Primary Visual (FUN_022bfc5c)

Main visual effect - always played if move has animation.

**Evidence:** `FUN_022bfc5c`
```c
void FUN_022bfc5c(ushort *param_1)
{
    // Copy template from DAT_022bfd18
    memcpy(local_38, DAT_022bfd18, ...);
    
    // Get move animation
    pmVar3 = GetMoveAnimation((uint)*param_1);
    
    // Read Layer 2 effect_id
    local_38[0] = (uint)pmVar3->field_0x4;
    
    // Copy position data
    local_38[1] = *(uint *)(param_1 + 8);
    local_38[2] = *(uint *)(param_1 + 6);
    // ...
    
    // Get attachment point
    pmVar3 = GetMoveAnimation((uint)*param_1);
    peVar4 = GetEffectAnimation((int)pmVar3->field_0x4);
    local_24[0] = peVar4->field_0x19;
    
    // Spawn effect with dispatch type 6
    FUN_022be780(6, local_38, 0);
}
```

### Layer 3: Projectile (FUN_022be9e8)

Projectile effects with trajectory data.

**Evidence:** `FUN_022be9e8`
```c
int FUN_022be9e8(ushort *param_1, undefined2 *param_2, ...)
{
    pmVar3 = GetMoveAnimation((uint)*param_1);
    
    // Read Layer 3 effect_id
    local_40[0] = (uint)pmVar3->field_0x6;
    
    // Get attachment point with per-Pokemon override
    iVar11 = FUN_022bf088((int)(short)param_1[1], (uint)*param_1);
    local_2c[0] = (undefined)iVar11;
    
    // Spawn effect with dispatch type 2
    iVar4 = FUN_022be780(2, local_40, 0);
    
    if (iVar4 != -1) {
        // Get effect context
        iVar11 = FUN_022be9a0((int)(short)iVar4);
        context = iVar11 * 0x13c + *DAT_022beb28;
        
        // Store projectile trajectory data
        *(context + 0x128) = source_x;
        *(context + 0x12a) = source_y;
        *(context + 0x12c) = dest_x;
        *(context + 0x12e) = dest_y;
        *(context + 0x130) = entity_id;
        *(context + 0x132) = velocity_x;
        *(context + 0x134) = velocity_y;
    }
    
    return iVar11;
}
```

## Projectile Spawn Call Chain

The complete call chain for projectile spawning:

```
FUN_02322374 (Move execution coordinator)
    │
    ├─► FUN_02322ddc (Returns wave pattern 0/1/2)
    │
    └─► FUN_023230fc (Main projectile handler)
            │
            ├─► FUN_02322f78 (Spawns projectile effect)
            │       │
            │       ├─► FUN_022bf01c (Get attachment point index)
            │       │
            │       ├─► FUN_0201cf90 (Calculate attachment offset - NOT USED for position)
            │       │
            │       └─► FUN_022be9e8 (Layer 3 projectile setup)
            │               │
            │               └─► FUN_022be780 (Effect dispatcher, type 2)
            │
            └─► Animation loop (FUN_022beb2c per frame)
```

### Key Functions in Chain

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_02322374` | `0x02322374` | Move execution coordinator, determines wave pattern |
| `FUN_02322ddc` | `0x02322ddc` | Returns wave pattern (0/1/2) based on move logic |
| `FUN_02322f78` | `0x02322f78` | Spawns projectile effect with position data |
| `FUN_023230fc` | `0x023230fc` | Main forward projectile motion handler |
| `FUN_0232393c` | `0x0232393c` | Reverse direction projectile handler |
| `FUN_022be9e8` | `0x022be9e8` | Layer 3 projectile effect setup |
| `FUN_022be780` | `0x022be780` | Main effect dispatcher |
| `FUN_022bf01c` | `0x022bf01c` | Get attachment point index with override |
| `FUN_022bf088` | `0x022bf088` | Get attachment point for species |
| `FUN_0201cf90` | `0x0201cf90` | Calculate attachment point offset from WAN |
| `FUN_022beb2c` | `0x022beb2c` | Update effect position during flight |

## Effect Dispatch Types

The first parameter to `FUN_022be780` indicates the handler type:

| Dispatch Type | Layer | Handler |
|---------------|-------|---------|
| 1 | 1 (Secondary) | Standard allocation |
| 2 | 3 (Projectile) | With trajectory setup |
| 5 | 0 (Charge) | Charge effects |
| 6 | 2 (Primary) | Main visual |

## Move Animation Flags

### Flag Bit Layout (offset 0x08)

| Bits | Mask | Accessor | Purpose |
|------|------|----------|---------|
| 0-2 | 0x07 | `FUN_022bfd58` | Animation category (0-7) |
| 3 | 0x08 | `FUN_022bfd6c` | Dual-target effect |
| 4 | 0x10 | `FUN_022bfd8c` | Skip fade-in effect |
| 5 | 0x20 | `FUN_022bfdac` | Unknown |
| 6 | 0x40 | `FUN_022bfdcc` | Add post-animation delay |
| 7 | 0x80 | - | Unknown/unused |

### Flag Bit 3: Dual-Target

When set, uses `FUN_023258ec` to play effects on both attacker and target.

**Evidence:** `ExecuteMoveEffect`
```c
bVar7 = FUN_022bfd6c((uint)(ushort)move->id);
if (bVar7) {
    FUN_023258ec((int *)attacker, (int *)entity, (int)move, iVar25);
}
```

Based on the Move Animation Info struct these are the only moves where flag bit 3 is true.

**Affected Moves:**
| Move Name | Move ID | Purpose |
|----------|---------|---------|
| Skill Swap | 137 | Exchange abilities |
| Guard Swap | 448 | Exchange Defense/Sp.Def stat changes |
| Switcheroo | 482 | Exchange held items |
| Heart Swap | 506 | Exchange all stat changes |
| Power Swap | 513 | Exchange Attack/Sp.Atk stat changes |

**Visual Purpose:** The simultaneous effect on both Pokemon represents the exchange/swap happening between them.

**Note:** Only the primary layer is duplicated. Charge effects still play on attacker only, and projectile effects still travel from attacker to target.

### Flag Bit 4: Skip Fade-In

**Evidence:** `FUN_02324e78`
```c
bVar6 = FUN_022bfd8c((uint)move_id);
if (!bVar6) {
    // Perform fade-in effect
    while (brightness < 0xFF) {
        brightness += 0x20;
        FUN_022ed0d4(brightness);
        AdvanceFrame('&');
    }
}
```

### Flag Bit 6: Post-Animation Delay

**Evidence:** `FUN_023250d4`
```c
bVar3 = FUN_022bfdcc((uint)*(ushort *)(param_2 + 4));
if (bVar3) {
    AnimationDelayOrSomething('\x01');
}
```

## Special Monster Animation Types (98/99)

Types 98 and 99 are special monster animation modes that use a simplified effect spawning pattern distinct from normal animations (types 0-12).

### Key Differences from Normal Animations

**Normal Animations (types 0-12):**
```c
ChangeMonsterAnimation(entity, anim_type, direction);

for (frame = 0; frame < 0x78; frame++) {
    AdvanceFrame('Y');
    flags = FUN_0201d1d4(entity + 0xb);  // Read WAN frame flags
    
    // HIT FRAME: bit 2 triggers effect spawning
    if ((flags & 2) != 0 && !effect_spawned) {
        FUN_02325644(&params, entity, move, ...);  // Spawn effect on hit frame
        effect_spawned = true;
    }
    
    // ANIMATION COMPLETE: bit 1
    if ((flags & 1) != 0) break;
}
```

**Type 99 (Spin) - Effect spawns once, animation loops:**
```c
FUN_02325644(&params, entity, move, ...);  // Effect spawns ONCE at start

for (i = 0; i < 8; i++) {
    direction = (direction + 1) & 7;  // Increment direction
    ChangeMonsterAnimation(entity, '\0', direction);
    FUN_022ea370(2, 0x15, ...);  // Fixed 2 frames per direction
}
// Total: 16 frames
```

**Type 98 (Multi-direction) - Effect spawns once, direction increments by 2:**
```c
FUN_02325644(&params, entity, move, ...);  // Effect spawns ONCE at start

for (i = 0; i < 9; i++) {
    current_dir = direction & 7;
    ChangeMonsterAnimation(entity, '\0', current_dir);
    FUN_022ea370(2, 0x15, ...);  // Fixed 2 frames per direction
    direction = current_dir + 2;  // Skip one direction (+2)
}
// Total: 18 frames
```

### Type 99: Spin Animation

Monster rotates through all 8 directions sequentially.

**Evidence:** `FUN_023250d4`
```c
if (iVar9 == 99) {
    dVar19 = (direction_id)*(byte *)(iVar17 + 0x4c);  // Current direction
    FUN_02325644(&local_3c, param_1, param_2, uVar7);  // Spawn secondary effect
    
    for (iVar9 = 0; iVar9 < 8; iVar9++) {
        dVar19 = (dVar19 + 1) & 7;  // Increment direction
        ChangeMonsterAnimation((entity *)param_1, '\0', dVar19);
        FUN_022ea370(2, 0x15, dVar19, uVar7);  // 2 frames per direction
    }
}
```

**Total duration:** 8 directions × 2 frames = 16 frames (~0.27 seconds)

### Type 98: Multi-Directional Attack

Monster attacks in 9 directions, incrementing by 2 (skipping every other direction).

**Evidence:** `FUN_023250d4`
```c
else if (iVar9 == 0x62) {  // 98 decimal
    uVar11 = (uint)*(byte *)(iVar17 + 0x4c);  // Current direction
    FUN_02325644(&local_3c, param_1, param_2, uVar7);  // Spawn secondary effect
    
    for (iVar9 = 0; iVar9 < 9; iVar9++) {
        dVar18 = uVar11 & 7;
        ChangeMonsterAnimation((entity *)param_1, '\0', dVar18);
        FUN_022ea370(2, 0x15, dVar19, uVar7);  // 2 frames per direction
        uVar11 = dVar18 + 2;  // Increment by 2 (skip directions)
    }
}
```

**Total duration:** 9 iterations × 2 frames = 18 frames (~0.3 seconds)

### Effect Spawning (FUN_02325644)

Spawns **layer 1** (secondary) effect only, before the direction loop begins.

**Evidence:** `FUN_02325644`
```c
void FUN_02325644(ushort *param_1, int *param_2, int param_3, int param_4)
{
    apparent_weather = GetApparentWeather((entity *)param_2);
    uVar1 = GetMoveAnimationId((move *)(uint)*(ushort *)(param_3 + 4), apparent_weather, (bool)param_4);
    pmVar2 = GetMoveAnimation((uint)uVar1);
    
    // Only spawns if layer 1 effect exists
    if (*param_1 == 0 || pmVar2->field_0x2 == 0) {
        return;
    }
    
    FUN_02325d7c(param_1, 1, param_4, iVar3);
    AdvanceFrame('Z');
    iVar3 = FUN_022bed90(param_1, ...);  // Layer 1 handler
    FUN_022e6d68(iVar3, param_2, 1);     // Attach to entity
}
```

**Key Points:**
- Effect spawns **once** at the start
- Uses **layer 1** (secondary effect) only
- Effect plays independently while sprite animation loops
- No frame flag synchronization needed

### Sound Effect Timing

Sound plays **once before the direction loop**, same as the effect:

**Evidence:** `FUN_023250d4`
```c
// Sound plays here, before type 98/99 check
uVar11 = FUN_022bf0f4((int)*(short *)(iVar17 + 4), (uint)uVar5);
if (uVar11 != 0x3f00) {
    PlaySeByIdIfNotSilence(uVar11 & 0xffff);
}

// Then type 98/99 handling
if (iVar9 == 99) {
    FUN_02325644(...);  // Effect
    // direction loop...
}
```

### Damage Application

Damage is applied **after** animation completes, in the caller context.

**Evidence:** `FUN_02322374` calling sequence
```c
// Animation plays (including types 98/99)
FUN_02322ddc((int *)param_1, (byte *)move, (int)local_140, (uint)ptVar19, param_3,
             iVar20 == 0);

// THEN damage is calculated and applied
ExecuteMoveEffect(&local_128, (entity *)param_1, move, ...);
```

**Key insight:** No synchronization needed between animation and damage for types 98/99. The effect spawns at the start, the sprite animation plays through all directions with fixed timing, then damage is applied after completion.

### Implementation Summary

For recreation of types 98/99:

1. **Sound:** Play sound effect once (if not 0x3F00)
2. **Effect:** Spawn layer 1 effect once via `FUN_022bed90` equivalent
3. **Animation Loop:**
   - Type 99: 8 iterations, direction += 1, 2 frames each (16 total)
   - Type 98: 9 iterations, direction += 2, 2 frames each (18 total)
4. **Completion:** Animation ends, caller handles damage separately

**Advantages:**
- Simpler than normal animations (no WAN frame flag parsing)
- Fixed timing (always 2 frames per direction)
- Effect management is straightforward (spawn once, plays independently)
- No complex synchronization between animation frames and effect spawning

## Animation ID Resolution

Before looking up move animation, the ID may be modified:
```c
bVar5 = ShouldMovePlayAlternativeAnimation(user, move);
apparent_weather = GetApparentWeather(user);
uVar7 = GetMoveAnimationId(move, apparent_weather, bVar5);
```

### Weather Ball

Animation changes based on weather:

- Clear: Normal type animation
- Rain: Water type animation
- Sun: Fire type animation
- etc.

### Two-Turn Moves

Charging turn uses alternative animation:

**Evidence:** `Is2TurnsMove`
```c
bool Is2TurnsMove(move_id move_id)
{
    if (move_id == MOVE_SOLARBEAM) return true;
    if (move_id == MOVE_SKY_ATTACK) return true;
    if (move_id == MOVE_RAZOR_WIND) return true;
    if (move_id == MOVE_FOCUS_PUNCH) return true;
    if (move_id == MOVE_SKULL_BASH) return true;
    if (move_id == MOVE_FLY) return true;
    if (move_id == MOVE_BOUNCE) return true;
    if (move_id == MOVE_DIVE) return true;
    if (move_id == MOVE_DIG) return true;
    // ... etc
    return false;
}
```

## Type 5 Screen Effect Check

Before playing animations, checks if move uses type 5 screen effects:

**Evidence:** `FUN_022bf160`
```c
bool FUN_022bf160(ushort *params)
{
    move_animation = GetMoveAnimation(params[0]);
    
    // Check all 4 layers for type 5 effects
    for (i = 0; i < 4; i++) {
        effect_id = move_animation->effect_layers[i];
        effect_anim = GetEffectAnimation(effect_id);
        if (effect_anim->anim_type == 5) {
            return true;
        }
    }
    return false;
}
```

If type 5 found, calls `FUN_0234ba54` to wait for screen readiness.

## Cross-References

> See `Systems/projectile_motion.md` for projectile spawn positions and motion handling

> See `Data Structures/effect_context.md` for trajectory field storage

## Open Questions

- Full purpose of animation category (flag bits 0-2)
- What flag bit 5 controls
- How layer 0 (charge) timing relates to two-turn moves

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `ExecuteMoveEffect` | - | Main move execution entry point |
| `PlayMoveAnimation` | - | Single-target animation handler |
| `FUN_023258ec` | `0x023258ec` | Dual-target animation handler |
| `FUN_022bfaa8` | `0x022bfaa8` | Layer 0 (charge) handler |
| `FUN_022bed90` | `0x022bed90` | Layer 1 (secondary) handler |
| `FUN_022bfc5c` | `0x022bfc5c` | Layer 2 (primary) handler |
| `FUN_022be9e8` | `0x022be9e8` | Layer 3 (projectile) handler |
| `FUN_022be780` | `0x022be780` | Main effect dispatcher |
| `FUN_022bf160` | `0x022bf160` | Check for type 5 effects |
| `FUN_022bfd58` | `0x022bfd58` | Read flag bits 0-2 |
| `FUN_022bfd6c` | `0x022bfd6c` | Read flag bit 3 |
| `FUN_022bfd8c` | `0x022bfd8c` | Read flag bit 4 |
| `FUN_022bfdac` | `0x022bfdac` | Read flag bit 5 |
| `FUN_022bfdcc` | `0x022bfdcc` | Read flag bit 6 |
| `FUN_023250d4` | `0x023250d4` | Monster animation handler (types 98/99 logic) |
| `FUN_02325644` | `0x02325644` | Spawn layer 1 effect (types 98/99) |
| `FUN_02322374` | `0x02322374` | Higher caller - handles damage after animation |
| `FUN_02322ddc` | `0x02322ddc` | Caller - sets up animation context |
| `FUN_0201d1d4` | `0x0201d1d4` | Read WAN frame flags (NOT used for 98/99) |
| `GetMoveAnimation` | - | Look up move_animation_info |
| `GetEffectAnimation` | - | Look up effect_animation_info |
| `GetMoveAnimationId` | - | Resolve animation ID with weather/charging |
| `ShouldMovePlayAlternativeAnimation` | - | Check for two-turn move charging |
| `GetApparentWeather` | - | Get current weather for Weather Ball |
