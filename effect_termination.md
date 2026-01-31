# Effect Termination

## Summary

- Effects terminate via two mechanisms: explicit stop (`FUN_022bde50`) or auto-terminate via tick
- Per-frame tick (`FUN_022bf4f0`) checks animation completion and loop flag to decide cleanup
- Loop flag at context offset `0x3C bit 0` controls whether effect continues after animation ends
- Looping effects (anim_type 3-5 with loop flag) run indefinitely until flag is cleared
- Projectiles self-terminate by calling `FUN_022bde50` after motion completes
- Status effect visuals terminate when status logic clears the loop flag, NOT via explicit stop
- Global cleanup (`FUN_022bdbc8`) kills all 32 effects during dungeon transitions

## Termination Mechanisms

### Mechanism 1: Explicit Stop (`FUN_022bde50`)

Direct termination by instance_id. Finds effect in pool and triggers cleanup.

**Evidence:** `FUN_022bde50`
```c
void FUN_022bde50(int param_1)  // param_1 = instance_id
{
    int iVar1 = FUN_022be9a0(param_1);  // Find slot index by instance_id
    if (iVar1 == -1) return;            // Not found
    
    int iVar2 = *DAT_022bdeb0;          // Pool base address
    iVar1 = iVar1 * 0x13c + iVar2;      // Calculate context address
    
    if (*(int *)(iVar1 + 0xc) == -1) return;  // Already inactive
    
    // Route to cleanup based on anim_type
    if (*(int *)(iVar1 + 8) - 1U < 2) {       // anim_type 1 or 2
        FUN_022bdcbc(param_1, 0, 0xffffffff, iVar2);
    }
    else {                                     // anim_type 3-6
        FUN_022bdcbc(param_1, 1, 0xffffffff, iVar2);
    }
}
```

**Used by:**
- Projectile motion completion
- Trap effect cleanup
- Two-stage effect transitions (charge → fire)
- Manual effect cancellation
- Global cleanup routines

### Mechanism 2: Auto-Terminate via Tick (`FUN_022bf4f0`)

Per-frame tick checks animation state and loop flag. Triggers cleanup automatically when conditions met.

**Evidence:** `FUN_022bf4f0` (relevant section)
```c
// Check if animation still playing
bVar6 = FUN_0201d1b0((ushort *)(param_1 + 0x1a));  // animation_control at +0x68

if (bVar6) {
    // Animation still playing - advance frame, update position
    SwitchAnimationControlToNextFrame((animation_control *)(param_1 + 0x1a));
    // ... position/rendering updates ...
}
else {
    // Animation completed - check if should cleanup
    if (param_1[2] - 3U < 3) {           // anim_type 3, 4, or 5
        if ((param_1[0xf] & 1U) == 0) {  // Loop flag at offset 0x3C, bit 0
            FUN_022bdcbc(...);           // NOT looping → cleanup
        }
        // If loop flag SET → effect continues, no cleanup
    }
    else {                               // anim_type 1, 2, or 6
        FUN_022bdcbc(...);               // Always cleanup
    }
}
```

### Termination Decision Matrix

| anim_type | Animation Complete | Loop Flag | Result |
|-----------|-------------------|-----------|--------|
| 1, 2 | Yes | N/A | Always cleanup |
| 3, 4, 5 | Yes | 0 (clear) | Cleanup |
| 3, 4, 5 | Yes | 1 (set) | Continue looping |
| 6 | Yes | N/A | Always cleanup |

## Loop Flag Locations

Two loop-related fields exist in effect context:

| Offset | Size | Set By | Read By | Purpose |
|--------|------|--------|---------|---------|
| `0x61` | byte | `FUN_022bdfc0` (from table) | `SetAnimationForAnimationControl` | Animation system loop control |
| `0x3C bit 0` | bit | Unknown | `FUN_022bf4f0` (tick) | Runtime termination control |

### Initialization (`0x61`)

Copied from `effect_animation_info.loop_flag` during effect setup.

**Evidence:** `FUN_022bdfc0`
```c
*(uint8_t *)(param_1 + 0x61) = peVar3->unk_repeat;  // Copy from table
// ...
SetAnimationForAnimationControl(
    (animation_control *)(param_1 + 0x68),
    animation_index,
    DIR_DOWN,
    (int)(short)*(undefined4 *)(param_1 + 0x54),
    *(uint *)(param_1 + 0x48) & 0xff,
    0,
    (uint)*(byte *)(param_1 + 0x61),  // Loop flag passed here
    0
);
```

### Runtime Check (`0x3C bit 0`)

Checked by tick to determine if looping effect should terminate.

**Evidence:** `FUN_022bf4f0`
```c
if ((param_1[0xf] & 1U) == 0) {  // param_1[0xf] = offset 0x3C (0xf * 4)
    FUN_022bdcbc(...);           // Bit 0 clear → terminate
}
```

### Open Question: Flag Synchronization

The relationship between `0x61` and `0x3C bit 0` is unclear. Possibilities:
1. `0x3C` is copied from `0x61` during initialization (not shown in `FUN_022bdfc0`)
2. `0x3C` serves a different purpose (runtime override)
3. Another function synchronizes them

## Projectile Self-Termination

Projectiles manage their own lifecycle. After motion loop completes, they explicitly stop themselves.

**Evidence:** `FUN_023230fc`
```c
// Motion loop
for (local_70 = 0; local_70 < (int)uVar20; local_70 = local_70 + 1) {
    // Update position, wave calculations...
    FUN_022beb2c((int)sVar1, &local_4c, z_priority);
    AdvanceFrame('0');
    // Advance position...
}

// Motion complete → explicit termination
if (-1 < local_38) {
    FUN_022bde50((int)(short)local_38);  // Stop primary projectile
}
if (-1 < iVar11) {
    FUN_022bde50((int)sVar2);            // Stop secondary projectile
}
```

**Key insight:** Projectiles do NOT rely on auto-terminate. They call `FUN_022bde50` directly after reaching destination.

## Trap Effect Cleanup

Traps maintain a list of active effects. When trap logic completes, all effects are force-stopped.

**Evidence:** `FUN_022e6ce0`
```c
void FUN_022e6ce0(void)
{
    iVar1 = DAT_022e6d64;
    if (*(int *)(DAT_022e6d64 + 4) == 0) return;
    
    // Iterate trap's effect list
    for (iVar4 = 0; iVar3 = *(int *)(iVar1 + 4), iVar4 < *(int *)(iVar3 + 8); iVar4 = iVar4 + 1) {
        // Check if effect still playing
        bVar2 = AnimationHasMoreFrames((int)(short)*(undefined4 *)(iVar3 + iVar4 * 4 + 0xc));
        if (bVar2 != '\0') {
            // Force stop any remaining effects
            FUN_022bde50((int)(short)*(undefined4 *)(*(int *)(iVar1 + 4) + iVar4 * 4 + 0xc));
        }
    }
    
    FUN_022bdc68();  // Additional cleanup
    MemFree(*(void **)(DAT_022e6d64 + 4));
    *(undefined4 *)(DAT_022e6d64 + 4) = 0;
}
```

**Call chain:** Trap Handler (`FUN_022ec9a4`) → Wrapper (`FUN_022e5dbc`) → Cleanup (`FUN_022e6ce0`)

## Two-Stage Effect Termination

Charge effects (Layer 0) are explicitly stopped before firing the main effect.

**Evidence:** `FUN_02324e78`
```c
// Spawn charge effect
uVar14 = FUN_022bfaa8(&local_38);
FUN_022e6d68(uVar14, param_1, 5);

// ... fade-in, waiting ...

// Wait for charge animation
while (true) {
    bVar3 = AnimationHasMoreFrames((int)(short)uVar14);
    if (bVar3 == '\0' || bVar6 == false) break;
    // ... fade handling ...
    AdvanceFrame('&');
}

// Note: Charge effect auto-terminates here via tick when animation completes
// Then caller spawns main effect (Layer 2)
```

## Global Effect Cleanup

### Kill All Active Effects (`FUN_022bdbc8`)

Iterates all 32 effect slots and stops each active one.

**Evidence:** `FUN_022bdbc8`
```c
void FUN_022bdbc8(void)
{
    int iVar2 = 0;
    int iVar1 = *DAT_022bdc08;  // Pool base
    
    do {
        if (*(int *)(iVar1 + 0xc) != -1) {           // If slot active
            FUN_022bde50((int)(short)*(int *)(iVar1 + 0xc));  // Stop it
        }
        iVar2 = iVar2 + 1;
        iVar1 = iVar1 + 0x13c;                       // Next slot
    } while (iVar2 < 0x20);                          // 32 slots
}
```

**Used for:** Dungeon transitions, cutscenes, hard resets.

### Full Pool Reset (`FUN_022bdc68`)

Calls `FUN_022bdbc8` then forcibly marks all slots inactive.

**Evidence:** `FUN_022bdc68`
```c
void FUN_022bdc68(void)
{
    FUN_022bdbc8();  // Stop all effects
    
    int iVar2 = 0;
    int iVar1 = *DAT_022bdca0;
    
    do {
        iVar2 = iVar2 + 1;
        *(undefined4 *)(iVar1 + 0xc) = 0xffffffff;  // Force instance_id = -1
        iVar1 = iVar1 + 0x13c;
    } while (iVar2 < 0x20);
    
    FUN_022bffa4();  // Additional resource cleanup
    FUN_022c0588();  // Additional resource cleanup
}
```

## Status Effect Termination

### The Mystery: No Direct Effect Stop

Status ender functions (e.g., `EndReflectClassStatus`) do NOT call `FUN_022bde50`.

**Evidence:** `EndReflectClassStatus`
```c
void EndReflectClassStatus(entity *user, entity *target)
{
    bVar1 = EntityIsValid(target);
    if (bVar1 == '\0') return;
    
    pvVar3 = target->info;
    
    // Print appropriate message based on status type
    switch(*(undefined *)((int)pvVar3 + 0xd5)) {
        case 1: LogMessageByIdWithPopupCheckUser(target, DAT_0230669c); break;
        case 2: LogMessageByIdWithPopupCheckUser(target, DAT_023066a0); break;
        // ... more cases ...
    }
    
    // ONLY clears the logical status flag
    *(undefined *)((int)pvVar3 + 0xd5) = 0;
    UpdateStatusIconFlags(target);
    return;
    
    // NOTE: No FUN_022bde50 call anywhere!
}
```

### Hypothesis: Loop Flag Clearing

Since the tick system (`FUN_022bf4f0`) auto-terminates when `loop_flag` is clear, the status system likely:

1. Stores effect-to-entity attachment in table (`DAT_022e6dcc`)
2. When status ends, finds attached effect via table
3. Clears `loop_flag` (offset `0x3C bit 0`) in effect context
4. Next tick detects `loop_flag == 0` → auto-terminate

### Attachment Table

`FUN_022e6d68` (Attach Effect) writes to attachment table at `DAT_022e6dcc`.

**Evidence:** `FUN_022e6d68` is called after spawning effects:
```c
// In PlayMoveAnimation:
uVar11 = FUN_022bfc5c(&local_34);  // Spawn effect
AdvanceFrame('[');
FUN_022e6d68(uVar11, target, 6);   // Attach to entity
```

**Related functions:**
- `FUN_022e6d68` - Attach effect to entity (writes to table)
- `FUN_022befd8` - Get slot from table (reads from table)

### Status Tick Integration

`TickStatusAndHealthRegen` calls status enders when counters expire.

**Evidence:** `TickStatusAndHealthRegen` (Reflect section)
```c
if (*(char *)((int)pvVar8 + 0xd5) != '\0') {
    TickStatusTurnCounter((uint8_t *)((int)pvVar8 + 0xd6));
    
    // Aqua Ring special case
    if ((*(char *)((int)pvVar8 + 0xd5) == '\x10') &&
       (TickStatusTurnCounter((uint8_t *)((int)pvVar8 + 0xd7)),
       *(char *)((int)pvVar8 + 0xd7) == '\0')) {
        ApplyAquaRingHealing(entity);
    }
    
    // Counter expired → end status
    if (*(char *)((int)pvVar8 + 0xd6) == '\0') {
        EndReflectClassStatus(entity, entity);  // Only clears logical flag
    }
}
```

**Key insight:** The visual effect must be terminated by a separate mechanism, likely loop flag clearing triggered by status change.

## Cleanup Coordinator (`FUN_022bdcbc`)

Central cleanup function called by both explicit stop and auto-terminate paths.

**Evidence:** `FUN_022bdcbc`
```c
void FUN_022bdcbc(int param_1, int param_2, undefined4 param_3, undefined4 param_4)
{
    int iVar2 = FUN_022be9a0(param_1);  // Find slot
    if (iVar2 == -1) return;
    
    int iVar5 = *DAT_022bde44;
    int *piVar6 = (int *)(iVar2 * 0x13c + iVar5);
    
    // Handle screen effects (type 5, 6)
    if (piVar6[0x10] - 5U < 2) {
        iVar2 = FUN_022bdca4(0);
        FUN_02063e44(iVar2, ...);
        *(undefined *)(*DAT_022bde44 + 0x279e) = 0;
        if (*(int *)(*piVar4 + 0x2784) == 0) {
            FUN_022ea428(0);
        }
        FUN_0206423c((int)(piVar6 + 0x3a));
    }
    
    // Handle WAN effects by type
    int iVar2 = piVar6[0x10];  // anim_type
    if (iVar2 == 3) {
        // Type 3: Decrement reference counter
        puVar3 = (undefined4 *)FUN_022c07d0(0, *piVar6);
        if (puVar3[1] == 0) {
            DebugPrint0(DAT_022bde48);
        }
        else {
            puVar3[1] = puVar3[1] - 1;
            if (puVar3[1] == 0) {
                *puVar3 = 0xffffffff;
            }
        }
    }
    else if (iVar2 == 4) {
        // Type 4: Similar reference handling
        // ...
    }
    // ... type 6, default cases ...
    
    // Final cleanup - marks slot inactive
    FUN_022bdec4((int)piVar6, param_2);
}
```

## Effect Lifecycle Summary

```
SPAWN
  │
  ▼
FUN_022be44c (Allocate slot)
  │
  ▼
FUN_022bdfc0 (Initialize)
  ├─► Set loop_flag at 0x61 from table
  ├─► Initialize animation_control
  └─► Set is_non_blocking at 0x60
  │
  ▼
FUN_022e6d68 (Attach to entity) [Optional]
  │
  ▼
╔═══════════════════════════════════════╗
║           ACTIVE LIFETIME              ║
║                                        ║
║  Per-frame: FUN_022bf4f0 (Tick)       ║
║    ├─► Advance animation              ║
║    ├─► Update position                ║
║    └─► Check termination conditions   ║
╚═══════════════════════════════════════╝
  │
  ▼
TERMINATION (one of):
  │
  ├─► [Explicit] FUN_022bde50(instance_id)
  │     • Projectile reaches target
  │     • Trap cleanup
  │     • Two-stage transition
  │     • Global reset
  │
  └─► [Auto] FUN_022bf4f0 detects:
        • Animation complete (bit 13 set)
        • Loop flag clear (for types 3-5)
        │
        ▼
      FUN_022bdcbc (Cleanup coordinator)
        │
        ▼
      FUN_022bdec4 (Final cleanup)
        └─► instance_id = -1 (slot free)
```

## Implementation Guidance

### For Non-Looping Effects
```
1. Spawn effect
2. Enter wait loop: while (AnimationHasMoreFrames(handle))
3. Effect auto-terminates when animation completes
4. Wait loop exits when effect removed from pool
```

### For Looping Effects (Status Visuals)
```
1. Spawn effect, store handle with entity reference
2. Do NOT wait - effect plays indefinitely
3. When status ends:
   a. Find effect by entity attachment
   b. Either: Clear loop_flag (ROM-accurate)
      Or: Call stop function directly (simpler)
4. Effect terminates on next tick (or immediately if explicit)
```

### For Projectiles
```
1. Spawn projectile effect
2. Run motion loop (position updates per frame)
3. On motion complete: explicitly stop effect
4. Do NOT rely on auto-terminate
```

## Open Questions

- Exact mechanism linking status end to loop flag clear
- Full structure of attachment table (`DAT_022e6dcc`)
- Whether `0x3C` and `0x61` are synchronized or independent
- Which function clears `0x3C bit 0` for status effects

## Cross-References

> See `Data Structures/effect_context.md` for full context structure

> See `Data Structures/animation_control.md` for animation state flags

> See `Systems/effect_lifecycle.md` for allocation and initialization

> See `Systems/projectile_motion.md` for projectile motion loop details

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_022bde50` | `0x022bde50` | Explicit effect stop by instance_id |
| `FUN_022bdbc8` | `0x022bdbc8` | Kill all active effects (global) |
| `FUN_022bdc68` | `0x022bdc68` | Full pool reset |
| `FUN_022bdcbc` | `0x022bdcbc` | Cleanup coordinator |
| `FUN_022bdec4` | `0x022bdec4` | Final cleanup (sets instance_id = -1) |
| `FUN_022bf4f0` | `0x022bf4f0` | Per-effect tick (auto-terminate logic) |
| `FUN_022be9a0` | `0x022be9a0` | Find effect slot by instance_id |
| `FUN_022e6d68` | `0x022e6d68` | Attach effect to entity |
| `FUN_022befd8` | `0x022befd8` | Get slot from attachment table |
| `FUN_022e6ce0` | `0x022e6ce0` | Trap effect cleanup |
| `FUN_023230fc` | `0x023230fc` | Projectile motion (self-terminates) |
| `FUN_02324e78` | `0x02324e78` | Two-stage effect handler |
| `FUN_0230d7d4` | `0x0230d7d4` | Revive animation handler |
| `EndReflectClassStatus` | - | Status ender (no effect stop) |
| `TickStatusAndHealthRegen` | - | Status tick (calls enders) |
| `AnimationHasMoreFrames` | - | Check if effect still active |
