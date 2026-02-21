# Animation Timing

## Summary

- `AdvanceFrame` yields to the game loop for one frame (~1/60 second)
- `FUN_022ea370(n, ...)` advances exactly n frames
- Blocking animations loop on `AnimationHasMoreFrames` until complete
- Non-blocking animations fire-and-forget (no wait loop)
- Special monster animation types 98/99 use fixed 2-frame-per-direction timing
- Underlying system is a cooperative task scheduler

## AdvanceFrame

Single frame advance - yields to game loop and returns on next frame.

**Evidence:** `AdvanceFrame`
```c
void AdvanceFrame(undefined param_1)
{
    if (*(byte *)(DAT_022ea004 + 3) != 0) {
        FUN_022ea2a4((uint)param_1, (uint)*(byte *)(DAT_022ea004 + 3), ...);
    } else {
        FUN_022ea324((uint)param_1, 0, ...);
    }
}
```

**Usage:**
- Parameter is unused (debug marker character like `'0'`, `'['`, `']'`, `'('`)
- Called throughout animation code to wait one frame
- Returns after VBlank/frame completion

## Multi-Frame Delay

`FUN_022ea370` loops to advance multiple frames:

**Evidence:** `FUN_022ea370`
```c
undefined4 FUN_022ea370(int param_1, undefined4 param_2, undefined4 param_3, undefined4 param_4)
{
    while (param_1 > 0) {
        if (*(char *)(DAT_022ea3b0 + 3) == '\0') {
            FUN_022ea324(param_2, ...);
        } else {
            FUN_022ea2a4(param_2, ...);
        }
        param_1--;
    }
    return result;
}
```

**Key insight:** `param_1` is the frame count. `FUN_022ea370(2, 0x15, ...)` advances **exactly 2 frames**, not 2×21.

### Common Calls

| Call | Meaning | Context |
|------|---------|---------|
| `FUN_022ea370(1, 0x4a, ...)` | 1 frame delay | Pre-animation setup |
| `FUN_022ea370(2, 0x15, ...)` | 2 frame delay | Per-direction in spin animation (types 98/99) |
| `FUN_022ea370(4, 0x4a, ...)` | 4 frame delay | Pre-move effect |

## Blocking Animation Loop

Standard pattern for waiting on animations to complete:
```c
// Spawn effect
effect_handle = FUN_022bfc5c(&params);

// Advance one frame to start
AdvanceFrame('[');

// Attach to entity
FUN_022e6d68(effect_handle, target, 6);

// Wait for completion
while (AnimationHasMoreFrames((int)(short)effect_handle)) {
    AdvanceFrame('(');
}
```

**Evidence:** `PlayMoveAnimation`
```c
uVar11 = FUN_022bfc5c(&local_34);
AdvanceFrame('[');
FUN_022e6d68(uVar11, target, 6);
while (bVar5 = AnimationHasMoreFrames((int)(short)uVar11), bVar5 != '\0') {
    AdvanceFrame('(');
}
```

### Maximum Frame Limits

Some code has safety limits to prevent infinite loops:

**Evidence:** `PlayEffectAnimationPixelPos`
```c
if (play_now != '\0') {
    iVar3 = 0;
    while ((iVar3 < 100 && (bVar1 = AnimationHasMoreFrames((int)(short)iVar2), bVar1 != '\0'))) {
        AdvanceFrame('B');
        iVar3 = iVar3 + 1;
    }
    iVar2 = -1;
}
```

Maximum of 100 frames (~1.67 seconds at 60fps) before giving up.

## Non-Blocking Animations

When `effect_animation->is_non_blocking` is non-zero, the animation plays asynchronously:

- Effect is spawned and attached to entity
- No wait loop - execution continues immediately
- Animation plays in background via task scheduler
- Effect cleaned up when animation completes (or entity despawns)

## Screen Effect Pre-Wait

Before type 5 screen effects, waits for screen to be ready:

**Evidence:** `FUN_0234ba54`
```c
uint FUN_0234ba54(uint param_1, ...)
{
    WaitUntilAlertBoxTextIsLoaded(param_1);
    
    for (iVar4 = 0; iVar4 <= 0xEF; iVar4++) {  // Max 239 frames
        // Check game state
        uVar3 = *(short *)(iVar5 + 0xc90);
        if (uVar3 < 0xB4) {  // 180
            return uVar3;
        }
        
        // Check screen effect flags
        if ((*puVar2 & 3) == 3) {
            return 3;
        }
        
        uVar1 = puVar2[1];
        if ((uVar1 & 0xF0) != 0) {
            return uVar1;
        }
        
        AdvanceFrame(param_1);
    }
    return uVar3;
}
```

Called before screen effects via:
```c
if (FUN_022bf160(&local_34)) {  // Check if move has type 5 effect
    FUN_0234ba54(0x5d, ...);
    AdvanceFrame(']');
}
```

## Task Scheduler

The underlying frame advance mechanism is a cooperative task scheduler:

**Evidence:** `FUN_022ddef8`
```c
undefined8 FUN_022ddef8(param_1, param_2, param_3, param_4)
{
    // Save current execution context
    *DAT_022ddfd8 = &stack_context;
    
    // Get current task from task list (0x20 bytes per task)
    task = (ushort *)(DAT_022ddfe0 + task_index * 0x20);
    
    // State machine: 1 -> 2 transition
    if (*task == 1) {
        *task = 2;      // Mark as running
        task[1] = 0;
    }
    
    // Store context in task
    *(task + 6) = saved_context;
    *(task + 8) = some_value;
    
    // Find next runnable task
    while (task_index < max_tasks) {
        task_index++;
        if (task_state == 1 || task_state == 2) break;
    }
    
    // Update current task index
    *DAT_022ddfdc = task_index;
    
    // If no more tasks, return
    if (task_index >= max_tasks) {
        return **(result_ptr);
    }
    
    // Restore context for next task
    *DAT_022ddfd8 = *(task + 0x16);
    *DAT_022ddfe4 = *(task + 0x18);
    
    // Execute task if state != 2
    if (task_state != 2) {
        stack_context[0] = param_1;
        stack_context[1] = param_2;
        stack_context[2] = param_3;
        stack_context[3] = param_4;
        
        // Jump to task handler
        return (*(code *)*DAT_022ddfe4)();
    }
    
    return *(result_from_context);
}
```

### Task States

| State | Meaning |
|-------|---------|
| 1 | Ready to run |
| 2 | Currently running |

### How It Works

1. Animation code calls `AdvanceFrame`
2. `AdvanceFrame` calls `FUN_022ea324` or `FUN_022ea2a4`
3. These call `FUN_022ddef8` (task scheduler)
4. Scheduler saves current state, finds next task
5. Other game systems run (rendering, input, etc.)
6. Eventually returns to animation code
7. Next frame begins

## Animation Delay Helper

`AnimationDelayOrSomething` adds a standard delay:

**Evidence:** `FUN_023230fc`
```c
if ((ushort)param_2[2] == DAT_02323914) {
    AnimationDelayOrSomething('\x01');
}
```

Called after certain projectile animations and in move effect pipeline when flag bit 6 is set.

## Main Loop Integration

The dungeon main frame handler calls the effect tick system every frame.

**Evidence:** `FUN_0234c1d8`
```c
void FUN_0234c1d8(...)  // Dungeon main loop
{
    // ... other frame systems (input, physics, AI, etc.) ...
    
    if (*DAT_0234c2ec == 0) {
        FUN_022bf764((short *)0x0, ...);  // Tick effects with default camera
    }
    else {
        FUN_022bf764((short *)(*DAT_0234c2ec + 0x1a224), ...);  // Tick with dungeon camera
    }
    
    // ... rendering, UI updates, etc. ...
}
```

`FUN_022bf764` iterates all 32 effect slots and calls `FUN_022bf4f0` for each.

**Evidence:** `FUN_022bf764`
```c
void FUN_022bf764(short *param_1, ...)
{
    piVar3 = (int *)*DAT_022bf7cc;  // Effect pool base
    iVar2 = 0;
    do {
        uVar1 = FUN_022bf4f0(piVar3, param_1);  // Tick individual effect
        iVar2 = iVar2 + 1;
        piVar3 = piVar3 + 0x4f;  // 0x4f * 4 = 0x13C bytes per slot
    } while (iVar2 < 0x20);  // 32 iterations
    
    return uVar1;
}
```

## AnimationHasMoreFrames Behavior

This function is used in wait loops but does NOT check animation frames directly. Instead, it searches the effect pool by `instance_id` and returns based on the `is_non_blocking` flag.

**Evidence:** `AnimationHasMoreFrames`
```c
bool AnimationHasMoreFrames(int param_1)  // param_1 = instance_id
{
    if (param_1 == -1) {
        return false;  // Invalid handle
    }
    
    // Search for effect in pool by instance_id
    iVar1 = 0;
    iVar2 = *DAT_022bf960;
    while (true) {
        if (0x1f < iVar1) {
            return false;  // Not found (effect cleaned up)
        }
        if (*(int *)(iVar2 + 0xc) == param_1) break;  // Found matching instance_id
        iVar1 = iVar1 + 1;
        iVar2 = iVar2 + 0x13c;
    }
    
    // Return TRUE if effect is blocking (is_non_blocking == 0)
    return *(char *)(iVar2 + 0x60) == '\0';
}
```

### What This Means

`AnimationHasMoreFrames` returns:
- **TRUE** if effect exists in pool AND `is_non_blocking == 0` (blocking effect)
- **FALSE** if effect handle is -1, effect cleaned up, or `is_non_blocking != 0` (non-blocking)

It does NOT directly check animation frame count or completion. Instead, it relies on:
1. Effect lifecycle system setting `instance_id = -1` when cleanup occurs
2. `is_non_blocking` flag to determine if caller should wait

## Wait Loop Exit Conditions

| Condition | Trigger | AnimationHasMoreFrames Returns |
|-----------|---------|-------------------------------|
| Effect handle == -1 | Never allocated or explicitly stopped | FALSE (exits loop) |
| Effect not in pool | Cleaned up by tick function | FALSE (exits loop) |
| is_non_blocking != 0 | Non-blocking effect spawned | FALSE (exits immediately) |
| Frame counter >= 100 | Safety timeout | N/A (loop exits via counter) |

**Evidence:** Typical wait loop with timeout
```c
if (play_now != '\0') {
    iVar5 = 0;
    while ((iVar5 < 100 && 
           (bVar2 = AnimationHasMoreFrames((int)(short)iVar4), bVar2 != '\0'))) {
        AdvanceFrame('B');
        iVar5 = iVar5 + 1;
    }
    iVar4 = -1;  // Invalidate handle after wait
}
```

### The 100-Frame Timeout

**Purpose:** Safety fallback to prevent infinite loops if effect lifecycle fails.

**Normal behavior:**
1. Effect plays animation
2. Animation completes → bit 13 set in animation_control
3. Effect tick calls cleanup → `instance_id = -1`
4. `AnimationHasMoreFrames` returns FALSE → loop exits

**Timeout scenario:**
- Bug prevents cleanup from occurring
- Loop reaches 100 iterations (~1.67 seconds at 60fps)
- Loop exits via counter, not `AnimationHasMoreFrames`

**Critical:** The timeout is NOT the intended termination mechanism. Properly functioning effects should exit via cleanup long before 100 frames.

### Looping Effects and Wait Loops

For effects with `loop_flag == 1`, the 100-frame timeout IS the exit condition because:
- Animation never sets bit 13 (restarts instead)
- Effect tick never calls cleanup (loop flag prevents it)
- `AnimationHasMoreFrames` keeps returning TRUE
- Loop exits at 100 frames

**This is why looping effects should NOT use blocking wait loops.** Instead:
```c
// WRONG: Will always timeout at 100 frames
effect_handle = SpawnLoopingEffect(...);
while (AnimationHasMoreFrames(effect_handle)) {  // BAD
    AdvanceFrame();
}

// CORRECT: Spawn and manage separately
effect_handle = SpawnLoopingEffect(...);
// ... perform action ...
ExecuteMoveEffect(...);
// ... explicitly stop when done ...
FUN_022bde50(effect_handle);
```

> See `Systems/effect_lifecycle.md` for complete lifecycle management including looping effect handling.

## Cross-References

> See `Systems/effect_lifecycle.md` for effect tick system (FUN_022bf764, FUN_022bf4f0)

> See `Data Structures/effect_context.md` for effect slot structure and is_non_blocking field

> See `Data Structures/animation_control.md` for animation state bitfield (bits 12, 13, 15)

## Effect Cleanup

When animation completes or needs to be stopped:

**Evidence:** `FUN_022bde50` usage
```c
if (-1 < effect_handle) {
    FUN_022bde50((int)(short)effect_handle);
}
```

`FUN_022bde50` stops the effect and frees resources.

## Special Animation Types (98/99) Timing

Monster animation types 98 and 99 use fixed timing that differs from normal animations.

> See `Systems/move_effect_pipeline.md` section "Special Monster Animation Types" for complete details.

### Type 99 (Spin) - 16 Frames Total

Loops through all 8 directions sequentially:
```c
for (direction = 0; direction < 8; direction++) {
    ChangeMonsterAnimation(entity, '\0', current_direction);
    FUN_022ea370(2, 0x15, direction, param);  // 2 frames per direction
    current_direction = (current_direction + 1) & 7;
}
```

**Total: 8 directions × 2 frames = 16 frames (~0.27 seconds)**

### Type 98 (Multi-direction) - 18 Frames Total

Loops through 9 directions, incrementing by 2 each time:
```c
for (i = 0; i < 9; i++) {
    current_dir = direction & 7;
    ChangeMonsterAnimation(entity, '\0', current_dir);
    FUN_022ea370(2, 0x15, current_dir, param);  // 2 frames per direction
    direction = current_dir + 2;  // Skip one direction
}
```

**Total: 9 iterations × 2 frames = 18 frames (~0.3 seconds)**

### Key Differences from Normal Animations

- **Normal animations:** Variable timing based on WAN frame data, effect spawns on hit frame (flagged in WAN)
- **Types 98/99:** Fixed 2-frame timing, effect spawns once at start before direction loop
- **No synchronization:** Effect plays independently while sprite animation loops through directions

## Animation Frame Duration System

Frame duration is a **uint8_t** in raw game ticks, copied verbatim from WAN data with no transformation.

**Confirmed chain:**
1. `wan_animation_frame.duration` — uint8_t, max 255 ticks (~4.25s at 60fps). Value 0 = end-of-animation marker.
2. `LoadAnimationFrameAndIncrementInAnimationControl` copies directly: `anim_ctrl->anim_frame_duration = (ushort)anim_frame->duration`
3. `SwitchAnimationControlToNextFrame` decrements by `field2_0x4` each tick
4. `field2_0x4` is **hardcoded to 1** in `SetAnimationForAnimationControlInternal` — no speed multiplier exists

**Implication for Q1 (charge speed):** The ROM timing system is confirmed correct and identical for all effect types (including WanFile0/1 shared effects). If charge effects play too fast, the issue is in the scraper's duration extraction or the client's frame timing, not a ROM-side speed difference.

### Pre-Spawn Delay (FUN_02325d7c)

Before spawning layer 0 (charge) and layer 1 (secondary) effects, if the move has a type 5 screen effect, the system waits for screen readiness then adds a **5-frame delay** via `FUN_022ea428(5)`.

### Charge Fade-In (FUN_022bfdec)

Specific moves trigger a brightness fade-in during the charge phase:
- Move 0x65 (101 / Hyper Beam)
- Move 0x1bc (444 / Roar of Time)
- One additional move stored in `DAT_022bfe08` (unknown, needs data read)

## Timing Summary

| Action | Function | Frames |
|--------|----------|--------|
| Single frame wait | `AdvanceFrame` | 1 |
| Multi-frame wait | `FUN_022ea370(n, ...)` | n |
| Animation complete wait | `while (AnimationHasMoreFrames(...))` | Variable |
| Pre-screen effect wait | `FUN_0234ba54` | Up to 239 |
| Safety timeout | Loop counter | Usually 100 |
| Type 99 spin animation | Fixed loop | 16 (8 directions × 2) |
| Type 98 multi-direction | Fixed loop | 18 (9 iterations × 2) |

## Open Questions

- Full task scheduler structure and task list layout
- What param_2 in `FUN_022ea370` controls (passed to inner functions but purpose unclear)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `AdvanceFrame` | - | Wait one frame |
| `FUN_022ea370` | `0x022ea370` | Wait n frames |
| `FUN_022ea324` | `0x022ea324` | Frame advance path A |
| `FUN_022ea2a4` | `0x022ea2a4` | Frame advance path B |
| `FUN_022ddef8` | `0x022ddef8` | Task scheduler core |
| `AnimationHasMoreFrames` | - | Check if animation still playing |
| `FUN_022bde50` | `0x022bde50` | Stop/cleanup effect |
| `FUN_0234ba54` | `0x0234ba54` | Pre-screen effect wait |
| `AnimationDelayOrSomething` | - | Standard post-animation delay |
| `FUN_023250d4` | `0x023250d4` | Monster animation handler (types 98/99 logic) |
