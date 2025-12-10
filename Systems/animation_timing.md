# Animation Timing

## Summary

- `AdvanceFrame` yields to the game loop for one frame (~1/60 second)
- `FUN_022ea370(n, ...)` advances exactly n frames
- Blocking animations loop on `AnimationHasMoreFrames` until complete
- Non-blocking animations fire-and-forget (no wait loop)
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
| `FUN_022ea370(2, 0x15, ...)` | 2 frame delay | Per-direction in spin animation |
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

## Effect Cleanup

When animation completes or needs to be stopped:

**Evidence:** `FUN_022bde50` usage
```c
if (-1 < effect_handle) {
    FUN_022bde50((int)(short)effect_handle);
}
```

`FUN_022bde50` stops the effect and frees resources.

## Timing Summary

| Action | Function | Frames |
|--------|----------|--------|
| Single frame wait | `AdvanceFrame` | 1 |
| Multi-frame wait | `FUN_022ea370(n, ...)` | n |
| Animation complete wait | `while (AnimationHasMoreFrames(...))` | Variable |
| Pre-screen effect wait | `FUN_0234ba54` | Up to 239 |
| Safety timeout | Loop counter | Usually 100 |

## Spin Animation Timing (Type 99)

For monster animation type 99 (spin through 8 directions):
```c
for (direction = 0; direction < 8; direction++) {
    ChangeMonsterAnimation(entity, '\0', current_direction);
    FUN_022ea370(2, 0x15, direction, param);  // 2 frames per direction
    current_direction = (current_direction + 1) & 7;
}
```

Total: 8 directions × 2 frames = 16 frames (~0.27 seconds)

## Open Questions

- Full task scheduler structure and task list layout
- What param_2 in `FUN_022ea370` controls (passed to inner functions but purpose unclear)
- How looping effects (`unk_repeat` flag) are terminated

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
