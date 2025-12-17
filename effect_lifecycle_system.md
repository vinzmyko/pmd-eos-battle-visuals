# Effect Lifecycle System

> **TODO:** Refactor relevant sections into `Systems/animation_timing.md` and `Data Structures/effect_animation_info.md` once validated.

## Summary

- Effects are managed in a pool of 32 slots, each 316 bytes (0x13C)
- Main loop calls `FUN_022bf764` every frame to tick all active effects
- `FUN_022bf4f0` is the per-effect tick function that advances animation and handles completion
- Non-looping effects auto-terminate when animation completes (bit 13 set in animation_control)
- Looping effects (`loop_flag = 1`) run indefinitely until explicitly stopped via `FUN_022bde50`
- 100-frame timeout in wait loops is a safety fallback, not the intended termination mechanism

## Effect Pool Structure

Effects are stored in a contiguous pool with 32 slots.

| Property | Value |
|----------|-------|
| Pool Size | 32 effects |
| Entry Size | 316 bytes (0x13C) |
| Total Size | 10,112 bytes (0x2780) |
| Base Address | `*DAT_022be72c` (NA) |

**Evidence:** `FUN_022bf764` iterates pool
```c
piVar3 = (int *)*DAT_022bf7cc;
iVar2 = 0;
do {
    uVar1 = FUN_022bf4f0(piVar3, param_1);
    iVar2 = iVar2 + 1;
    piVar3 = piVar3 + 0x4f;  // 0x4f * 4 = 0x13C bytes
} while (iVar2 < 0x20);  // 32 iterations
```

## Effect Context Structure (Expanded)

```c
struct effect_context {
    /* 0x00 */ int32_t param_from_caller;
    /* 0x04 */ int32_t caller_context;
    /* 0x08 */ int32_t anim_type;          // 1-6
    /* 0x0C */ int32_t instance_id;        // -1 = inactive/free slot
    /* 0x10 */ int32_t wan_ptr;
    /* 0x14 */ int32_t effect_id;
    /* 0x18 */ int32_t delay_counter;      // Countdown before tick processes
    /* 0x1C */ int32_t direction;
    /* 0x20 */ int16_t current_x;
    /* 0x22 */ int16_t current_y;
    /* 0x24 */ int16_t velocity_x;
    /* 0x26 */ int16_t velocity_y;
    /* 0x2C */ int32_t z_priority;
    /* ... */
    /* 0x3C */ uint32_t flags;             // Bit 0 = loop flag (from context, not effect_animation)
    /* 0x40 */ int32_t stored_anim_type;
    /* 0x44 */ int32_t file_index;
    /* 0x48 */ int32_t palette_index;
    /* 0x4C */ int32_t unknown_4c;
    /* 0x50 */ int32_t animation_index;
    /* 0x54 */ int32_t unknown_54;
    /* 0x58 */ int32_t sfx_id;
    /* 0x5C */ int32_t timing_value;
    /* 0x60 */ uint8_t is_non_blocking;    // Controls AnimationHasMoreFrames return
    /* 0x61 */ uint8_t loop_flag;          // From effect_animation_info
    /* 0x64 */ int16_t wan_table_entry;    // For cleanup
    /* 0x68 */ animation_control anim_ctrl; // WAN animation state (~0x44 bytes)
    /* ... */
    /* 0xE4 */ int16_t screen_effect_handle; // For type 5/6
    /* 0xE8 */ uint8_t screen_effect_ctrl[0x1C]; // Screen effect control structure
    /* ... */
    /* 0x128 */ int16_t source_x;
    /* 0x12A */ int16_t source_y;
    /* 0x12C */ int16_t dest_x;
    /* 0x12E */ int16_t dest_y;
    /* 0x130 */ uint16_t entity_id;
    /* 0x132 */ int16_t stored_velocity_x;
    /* 0x134 */ int16_t stored_velocity_y;
    /* 0x136 */ int16_t position_offset;   // Added to X each frame
    /* 0x13A */ uint8_t unknown_13a;
};
// Size: 0x13C (316 bytes)
```

## Main Loop Integration

`FUN_0234c1d8` is the dungeon main frame handler that calls the effect tick system.

**Evidence:** `FUN_0234c1d8`
```c
void DungeonMainLoop(...) {
    // ... other frame systems ...
    
    if (*DAT_0234c2ec == 0) {
        FUN_022bf764((short *)0x0, ...);  // Tick effects with default camera
    }
    else {
        FUN_022bf764((short *)(*DAT_0234c2ec + 0x1a224), ...);  // Tick with dungeon camera
    }
    
    // ... rendering, input, etc ...
}
```

## Effect Tick Function (FUN_022bf4f0)

The core per-effect update function. Called once per frame for each active effect.

**Evidence:** `FUN_022bf4f0`
```c
undefined4 FUN_022bf4f0(int *param_1, short *param_2)
{
    iVar9 = param_1[3];  // instance_id at offset 0x0C
    if (iVar9 != -1) {   // Effect is active
        
        if (param_1[6] < 1) {  // delay_counter at offset 0x18
            iVar7 = param_1[0x10];  // stored_anim_type at offset 0x40
            
            if (iVar7 == 5) {
                FUN_022bdf34(param_1);  // Screen effect tick
            }
            else if (iVar7 == 6) {
                FUN_022bdf34(param_1);  // Screen effect tick
            }
            else {
                // WAN animation tick
                bVar6 = FUN_0201d1b0((ushort *)(param_1 + 0x1a));  // Check animation_control
                
                if (bVar6) {
                    // Animation still playing
                    SwitchAnimationControlToNextFrame((animation_control *)(param_1 + 0x1a));
                    // ... position/rendering updates ...
                }
                else {
                    // Animation completed - check if should cleanup
                    if (param_1[2] - 3U < 3) {  // anim_type 3, 4, or 5
                        if ((param_1[0xf] & 1U) == 0) {  // Loop flag NOT set
                            FUN_022bdcbc((int)(short)param_1[3], 1, iVar7, iVar9);  // Cleanup
                        }
                        // If loop flag set, don't cleanup - effect continues
                    }
                    else {  // anim_type 1, 2, or 6
                        FUN_022bdcbc((int)(short)param_1[3], 0, iVar7, iVar9);  // Always cleanup
                    }
                }
            }
        }
        
        // Decrement delay counter
        if (0 < param_1[6]) {
            param_1[6] = param_1[6] + -1;
        }
        
        // ... sound effect handling ...
        
        // Return based on is_non_blocking
        if (*(char *)(param_1 + 0x18) == '\0') {  // offset 0x60
            return 1;  // Blocking effect, still active
        }
    }
    return 0;  // Inactive or non-blocking
}
```

## Animation State Check (FUN_0201d1b0)

Checks if WAN animation is still playing by examining the animation_control bitfield.

**Evidence:** `FUN_0201d1b0`
```c
bool FUN_0201d1b0(ushort *param_1)  // param_1 = &animation_control.some_bitfield
{
    ushort bitfield = *param_1;
    
    if ((bitfield & 0x2000) != 0)    // Bit 13 = COMPLETED/STOPPED
        return false;
    
    return (bitfield & 0x8000) != 0;  // Bit 15 = ACTIVE
}
```

### Animation Control Bitfield

| Bit | Mask | Meaning |
|-----|------|---------|
| 15 | 0x8000 | Animation active/initialized |
| 13 | 0x2000 | Animation completed/stopped |
| 12 | 0x1000 | Looping enabled |

**Evidence:** `SetAnimationForAnimationControlInternal`
```c
anim_ctrl->some_bitfield = 0;
// ...
uVar5 = anim_ctrl->some_bitfield | 0x8000;  // Set active bit
anim_ctrl->some_bitfield = uVar5;

if ((char)loop_flag != '\0') {
    anim_ctrl->some_bitfield = uVar5 | 0x1000;  // Set loop bit
}
```

## AnimationHasMoreFrames Behavior

This function is used in wait loops but does NOT check animation frames directly.

**Evidence:** `AnimationHasMoreFrames`
```c
bool AnimationHasMoreFrames(int param_1)
{
    if (param_1 == -1) {
        return false;  // Invalid handle
    }
    
    // Search for effect in pool by instance_id
    iVar1 = 0;
    iVar2 = *DAT_022bf960;
    while (true) {
        if (0x1f < iVar1) {
            return false;  // Not found
        }
        if (*(int *)(iVar2 + 0xc) == param_1) break;  // Found
        iVar1 = iVar1 + 1;
        iVar2 = iVar2 + 0x13c;
    }
    
    return *(char *)(iVar2 + 0x60) == '\0';  // Return TRUE if is_non_blocking == 0
}
```

### Wait Loop Exit Conditions

| Condition | Trigger | Result |
|-----------|---------|--------|
| Effect handle == -1 | Explicit stop or never allocated | Exit loop |
| Effect not in pool | Cleaned up by tick function | Exit loop |
| is_non_blocking != 0 | Non-blocking effect | Exit immediately |
| Frame counter >= 100 | Safety timeout | Exit loop (fallback) |

**Evidence:** `PlayEffectAnimationEntity`
```c
if (play_now != '\0') {
    iVar5 = 0;
    while ((iVar5 < 100 && (bVar2 = AnimationHasMoreFrames((int)(short)iVar4), bVar2 != '\0'))) {
        AdvanceFrame('B');
        iVar5 = iVar5 + 1;
    }
    iVar4 = -1;
}
```

## Effect Termination Paths

### Path 1: Auto-Termination (Non-Looping)

For effects with `loop_flag == 0`:

```
Frame N: Animation plays last frame
    ↓
Frame N+1: WAN system sets bit 13 (0x2000) in animation_control.some_bitfield
    ↓
FUN_022bf4f0 tick:
    FUN_0201d1b0() returns FALSE (animation complete)
    ↓
    FUN_022bdcbc() called (cleanup)
    ↓
Effect removed from pool
```

### Path 2: Explicit Stop

For any effect, caller can force termination:

```
Caller decides to stop effect
    ↓
FUN_022bde50(instance_id)
    ↓
FUN_022bdcbc() called
    ↓
FUN_022bdec4() sets instance_id = -1
    ↓
Effect slot now free
```

**Evidence:** `FUN_022bde50`
```c
void FUN_022bde50(int param_1)
{
    iVar1 = FUN_022be9a0(param_1);  // Find slot index
    if (iVar1 == -1) return;
    
    iVar2 = *DAT_022bdeb0;
    iVar1 = iVar1 * 0x13c + iVar2;
    
    if (*(int *)(iVar1 + 0xc) == -1) return;  // Already stopped
    
    if (*(int *)(iVar1 + 8) - 1U < 2) {  // anim_type 1 or 2
        FUN_022bdcbc(param_1, 0, 0xffffffff, iVar2);
    }
    else {  // anim_type 3-6
        FUN_022bdcbc(param_1, 1, 0xffffffff, iVar2);
    }
}
```

### Path 3: Type Conflict Resolution

When spawning type 3 or 4 effects, conflicting effects are force-stopped.

**Evidence:** `FUN_022be44c`
```c
if (iVar6 == 3) {
    iVar6 = 0;
    iVar10 = *DAT_022be72c;
    do {
        if (((*(int *)(iVar10 + 0xc) != -1) &&          // Active
             (*(int *)(iVar10 + 8) == 3)) &&            // Same type
            ((*(int *)(iVar10 + 0x14) != *param_2 &&    // Different effect_id
             (peVar4 = GetEffectAnimation(*(int *)(iVar10 + 0x14)),
              peVar4->file_index != peVar3->file_index)))) {  // Different file
            FUN_022bdec4(iVar10, 1);  // Force stop conflicting effect
        }
        iVar6 = iVar6 + 1;
        iVar10 = iVar10 + 0x13c;
    } while (iVar6 < 0x20);
}
```

### Path 4: Looping Effects

For effects with `loop_flag == 1`:

```
FUN_022bf4f0 tick:
    FUN_0201d1b0() returns FALSE (animation "complete" but loops)
    ↓
    Check: (param_1[0xf] & 1U) == 0?  // Loop flag check
    ↓
    Loop flag SET → Skip cleanup, effect continues
    ↓
Effect plays indefinitely until Path 2 (explicit stop)
```

## Looping Effect Behavior by anim_type

| anim_type | Loop Check | Behavior |
|-----------|------------|----------|
| 1, 2 | No | Always auto-terminate |
| 3, 4, 5 | Yes | Check loop flag at context + 0x3C |
| 6 | No | Always auto-terminate |

**Evidence:** `FUN_022bf4f0`
```c
if (param_1[2] - 3U < 3) {  // anim_type 3, 4, or 5
    if ((param_1[0xf] & 1U) == 0) {  // Loop flag NOT set
        FUN_022bdcbc(...);  // Cleanup
    }
    // If loop flag set, don't cleanup
}
else {  // anim_type 1, 2, or 6
    FUN_022bdcbc(...);  // Always cleanup
}
```

## Effect Cleanup (FUN_022bdec4)

Marks effect as inactive and frees resources.

**Evidence:** `FUN_022bdec4`
```c
void FUN_022bdec4(int param_1, int param_2)
{
    *(undefined4 *)(param_1 + 0xc) = 0xffffffff;  // instance_id = -1
    *(undefined *)(param_1 + 0x60) = 0;           // Clear is_non_blocking
    
    if (1 < *(int *)(param_1 + 8) - 5U) {  // anim_type 1-4
        if (*(short *)(param_1 + 100) == 0) return;
        if (param_2 != 0) {
            DeleteWanTableEntryVeneer(..., *(short *)(param_1 + 100));
        }
        *(undefined2 *)(param_1 + 100) = 0;
    }
    else {  // anim_type 5-6
        if (param_2 == 0) return;
        thunk_FUN_02063ff4(*(short *)(param_1 + 0xe4));  // Free screen effect
    }
}
```

## Complete Effect Lifecycle

```
1. ALLOCATION (FUN_022be44c)
   - Find free slot (instance_id == -1)
   - Clear slot memory
   - Load WAN/screen resources
   - Assign unique instance_id
   
2. INITIALIZATION (FUN_022bdfc0)
   - Copy from effect_animation_info:
     - is_non_blocking → context + 0x60
     - loop_flag → context + 0x61
     - animation_index, palette, etc.
   - Initialize animation_control at context + 0x68
   - Set some_bitfield |= 0x8000 (active)
   - If looping: some_bitfield |= 0x1000
   
3. PER-FRAME TICK (FUN_022bf4f0)
   - Called from main loop via FUN_022bf764
   - Decrement delay counter if > 0
   - Check FUN_0201d1b0 for animation state
   - If playing: advance frame, update position
   - If complete: check loop flag, cleanup if needed
   
4. TERMINATION
   - Auto: Tick detects completion, calls FUN_022bdcbc
   - Explicit: Caller calls FUN_022bde50
   - Conflict: New effect force-stops old via FUN_022bdec4
   - Loop effects: Only terminate via explicit stop
```

## Implementation Guidance

### For Non-Looping Effects
```
- Spawn effect
- Enter wait loop with AnimationHasMoreFrames()
- Effect auto-terminates when animation completes
- Wait loop exits when effect cleaned up from pool
```

### For Looping Effects
```
- Spawn effect, store handle
- Do NOT wait in blocking loop (will timeout at 100 frames)
- Track parent action state (move execution, entity status)
- Explicitly call FUN_022bde50 equivalent when:
  - Move damage is applied
  - Parent animation completes
  - Entity dies/despawns
  - Action is interrupted
- Use 100-frame (~1.67s at 60fps) timeout as safety fallback
```

## Open Questions

- How does `delay_counter` (offset 0x18) get set initially?
- What determines the value at `context + 0x3C` (runtime loop flag) vs `context + 0x61` (from table)?
- Complete screen effect tick logic in `FUN_022bdf34`
- How are looping effects intended to be stopped in original game logic?

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_0234c1d8` | `0x0234c1d8` | Dungeon main frame handler |
| `FUN_022bf764` | `0x022bf764` | Effect pool tick (iterates 32 slots) |
| `FUN_022bf7e0` | `0x022bf7e0` | Filtered effect tick (checks anim_type) |
| `FUN_022bf864` | `0x022bf864` | Filtered effect tick (alternate filter) |
| `FUN_022bf4f0` | `0x022bf4f0` | Per-effect tick function |
| `FUN_0201d1b0` | `0x0201d1b0` | Check if animation still playing |
| `AnimationHasMoreFrames` | - | Check if effect still active (for wait loops) |
| `FUN_022be44c` | `0x022be44c` | Effect allocation |
| `FUN_022bdfc0` | `0x022bdfc0` | Effect initialization |
| `FUN_022bde50` | `0x022bde50` | Explicit effect stop |
| `FUN_022bdcbc` | `0x022bdcbc` | Effect cleanup coordinator |
| `FUN_022bdec4` | `0x022bdec4` | Effect cleanup (sets instance_id = -1) |
| `FUN_022bdf34` | `0x022bdf34` | Screen effect (type 5/6) tick |
| `FUN_022be9a0` | `0x022be9a0` | Find effect slot by instance_id |
| `SwitchAnimationControlToNextFrame` | - | Advance WAN animation |
| `SetAnimationForAnimationControlInternal` | `0x0201C5C4` | Initialize animation_control |
