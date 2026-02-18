# Effect Lifecycle

## Summary

- Effects are allocated from a 32-slot pool, initialized, ticked per frame, then terminated
- Main loop calls `FUN_022bf764` every frame to tick all active effects
- `FUN_022bf4f0` is the per-effect tick function that advances animation and handles completion
- Non-looping effects auto-terminate when animation completes
- Looping effects run indefinitely until explicitly stopped
- Four termination paths: auto-termination, explicit stop, type conflict, and looping behavior

## Complete Effect Lifecycle
```
1. ALLOCATION (FUN_022be44c)
   - Find free slot (instance_id == -1)
   - Clear slot memory
   - Load resources (type-dependent):
     - Types 1/2: Reuse shared sprite at base + 0x2788 (no file loading)
     - Types 3/4: Load per-effect WAN file via FUN_022c03f4
     - Types 5/6: Allocate screen effect resource via FUN_022c0450
   - Assign unique instance_id from base + 0x2780 counter
   
2. INITIALIZATION (FUN_022bdfc0)
   - Copy from effect_animation_info:
     - is_non_blocking → context + 0x60
     - loop_flag → context + 0x61
     - animation_index, palette, etc.
   - Initialize animation_control at context + 0x68
   - Set animation_control.some_bitfield |= 0x8000 (active)
   - If looping: animation_control.some_bitfield |= 0x1000
   
3. PER-FRAME TICK (FUN_022bf4f0)
   - Called from main loop via FUN_022bf764
   - Decrement delay counter if > 0
   - Check animation state via FUN_0201d1b0
   - If playing: advance frame, update position
   - If complete: check loop flag, cleanup if needed
   
4. TERMINATION
   - Auto: Tick detects completion, calls FUN_022bdcbc
   - Explicit: Caller calls FUN_022bde50
   - Conflict: New effect force-stops old via FUN_022bdec4
   - Loop effects: Only terminate via explicit stop
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
    Check anim_type and loop flag
    ↓
    FUN_022bdcbc() called (cleanup)
    ↓
Effect removed from pool
```

**Evidence:** `FUN_022bf4f0`
```c
bVar6 = FUN_0201d1b0((ushort *)(param_1 + 0x1a));  // Check animation state

if (bVar6) {
    // Animation still playing
    SwitchAnimationControlToNextFrame(...);
}
else {
    // Animation completed
    if (param_1[2] - 3U < 3) {  // anim_type 3, 4, or 5
        if ((param_1[0xf] & 1U) == 0) {  // Loop flag NOT set
            FUN_022bdcbc(...);  // Cleanup
        }
    }
    else {  // anim_type 1, 2, or 6
        FUN_022bdcbc(...);  // Always cleanup
    }
}
```

### Path 2: Explicit Stop

For any effect, caller can force termination:
```
Caller decides to stop effect
    ↓
FUN_022bde50(instance_id)
    ↓
Find slot by instance_id
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
    // Check all active effects
    for (i = 0; i < 0x20; i++) {
        if ((entry->instance_id != -1) &&          // Active
            (entry->anim_type == 3) &&             // Same type
            (entry->effect_id != current_effect_id) &&    // Different effect
            (GetEffectAnimation(entry->effect_id)->file_index != 
             GetEffectAnimation(current_effect_id)->file_index)) {  // Different file
            
            FUN_022bdec4(entry, 1);  // Force stop conflicting effect
        }
    }
}
```

**Type 4 follows similar pattern:**
```c
if (iVar6 == 4) {
    // Force stop any existing type 4 effects
    for (i = 0; i < 0x20; i++) {
        if (entry->instance_id != -1 && entry->anim_type == 4) {
            FUN_022bdec4(entry, 1);
        }
    }
}
```

### Path 4: Looping Effects

For effects with `loop_flag == 1` (and anim_type 3, 4, or 5):
```
FUN_022bf4f0 tick:
    FUN_0201d1b0() returns FALSE (animation "complete" but looping)
    ↓
    Check: (context[0xf] & 1U) == 0?  // Loop flag check at offset 0x3C
    ↓
    Loop flag SET → Skip cleanup, effect continues
    ↓
Animation restarts via WAN system (bit 12 enabled in animation_control)
    ↓
Effect plays indefinitely until Path 2 (explicit stop)
```

## Looping Behavior by anim_type

| anim_type | Loop Check | Behavior |
|-----------|------------|----------|
| 1 | No | Always auto-terminate when animation completes |
| 2 | No | Always auto-terminate when animation completes |
| 3 | Yes | Check loop flag; if set, continue indefinitely |
| 4 | Yes | Check loop flag; if set, continue indefinitely |
| 5 | Yes | Check loop flag; if set, continue indefinitely |
| 6 | No | Always auto-terminate when animation completes |

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
        *(undefined2 *)(param_1 + 100) = 0;  // Clear wan_table_entry
    }
    else {  // anim_type 5-6
        if (param_2 == 0) return;
        thunk_FUN_02063ff4(*(short *)(param_1 + 0xe4));  // Free screen effect
    }
}
```

## Implementation Guidance

### For Non-Looping Effects
```
1. Spawn effect via FUN_022be780 (returns instance_id)
2. Enter wait loop with AnimationHasMoreFrames(instance_id)
3. Effect auto-terminates when animation completes
4. Wait loop exits when effect cleaned up from pool
```

**Example:**
```c
effect_handle = FUN_022bfc5c(&params);
AdvanceFrame('[');
FUN_022e6d68(effect_handle, target, 6);

while (AnimationHasMoreFrames((int)(short)effect_handle)) {
    AdvanceFrame('(');
}
// Effect is now cleaned up
```

### For Looping Effects
```
1. Spawn effect, store handle
2. Do NOT wait in blocking loop (will timeout at 100 frames)
3. Track parent action state (move execution, entity status)
4. Explicitly call FUN_022bde50(handle) when:
   - Move damage is applied
   - Parent animation completes
   - Entity dies/despawns
   - Action is interrupted
5. Use 100-frame timeout as safety fallback only
```

**Example:**
```c
effect_handle = FUN_022be780(...);  // Looping effect

// Perform action (move execution, etc.)
ExecuteMoveEffect(...);

// Stop effect when action completes
if (effect_handle >= 0) {
    FUN_022bde50((int)(short)effect_handle);
}
```

## Cross-References

> See `Data Structures/effect_context.md` for structure layout and pool management

> See `Systems/animation_timing.md` for AnimationHasMoreFrames and wait loop patterns

> See `Data Structures/animation_control.md` for animation state bitfield meanings (bits 12, 13, 15)

## Open Questions

- How does `delay_counter` (offset 0x18) get set initially?
- What determines the value at `context + 0x3C` (runtime loop flag) vs `context + 0x61` (from table)?
- Complete screen effect tick logic in `FUN_022bdf34`
- Are there other context flag bits beyond bit 0?

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_022bf764` | `0x022bf764` | Effect pool tick (iterates 32 slots) |
| `FUN_022bf4f0` | `0x022bf4f0` | Per-effect tick function |
| `FUN_0201d1b0` | `0x0201d1b0` | Check if animation still playing |
| `FUN_022be44c` | `0x022be44c` | Effect allocation |
| `FUN_022bdfc0` | `0x022bdfc0` | Effect initialization |
| `FUN_022bde50` | `0x022bde50` | Explicit effect stop |
| `FUN_022bdcbc` | `0x022bdcbc` | Effect cleanup coordinator |
| `FUN_022bdec4` | `0x022bdec4` | Effect cleanup (sets instance_id = -1) |
| `FUN_022bdf34` | `0x022bdf34` | Screen effect (type 5/6) tick |
| `FUN_022be9a0` | `0x022be9a0` | Find effect slot by instance_id |
| `SwitchAnimationControlToNextFrame` | - | Advance WAN animation |
