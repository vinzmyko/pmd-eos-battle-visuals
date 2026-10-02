# Attachment Point Pipeline

## Summary

- Bound layers read the attachment point from the entity they are bound to (usually the target); projectiles read the **attacker** once, at launch
- `effect_animation_info.field_0x19` selects which point index (0-3) or disables attachment (0xFF)
- Layer 2 (primary) effects track the target's attachment point every frame via `FUN_022e6e80`
- The (99, 99) sentinel is only handled in `PlayEffectAnimationEntity`, not in the `PlayMoveAnimation` → Layer 2 path
- Per-species overrides exist via `FUN_022bf01c` / `FUN_022bf088` but only for Layer 1/3 (secondary/projectile)

## Attachment Point Index Selection

The attachment point index comes from `effect_animation_info.field_0x19` (offset 0x19 in the effect animation data).

| Value | Meaning |
|-------|---------|
| 0 | Head |
| 1 | Left Hand |
| 2 | Right Hand |
| 3 | Centre |
| 0xFF | No attachment — use entity position directly |

## Resolution by Layer

### Layer 2: Primary Effect (PlayMoveAnimation → FUN_022bfc5c)

The main visual effect path. Attachment offset is read from the **target**.

**Evidence:** `PlayMoveAnimation`
```c
// Defaults loaded from global (DAT_023258e8)
uVar1 = *DAT_023258e8;       // default offset X
uVar2 = DAT_023258e8[1];     // default offset Y

peVar10 = GetEffectAnimation((int)sVar4);
bVar3 = peVar10->field_0x19;  // attachment_point index

if ((bVar3 != 0xff) && (EntityIsValid(target))) {
    // Read attachment offset from TARGET's current animation frame
    FUN_0201cf90((short *)&local_2c, &(target->anim_ctrl).some_bitfield, (uint)bVar3);
    uVar1 = local_2c;
    uVar2 = local_2a;
}
// Passed directly into spawn params — no (99,99) check
```

**Key points:**
- Reads from `target->anim_ctrl`, not `user->anim_ctrl`
- If index is 0xFF, keeps global default offset
- If target is invalid, keeps global default offset
- Does NOT check for (99, 99) return value — passes raw values through

### Per-Frame Tracking (FUN_022e6e80)

After initial spawn, effects bound to a target are updated every frame.

**Evidence:** `FUN_022e6e80`
```c
// Iterates 3-slot binding table at base + 0x618
if (*(byte *)(puVar4 + 2) != 0xff) {
    // Re-read attachment offset from target's CURRENT frame
    FUN_0201cf90(&local_2c, (ushort *)(param_1 + 0x2c), (uint)*(byte *)(puVar4 + 2));
}
// Passes result to FUN_022bfb6c
```

The stored attachment point index (slot offset +2) is re-evaluated against the target's current animation frame each tick, so the effect tracks the attachment point as the target animates.

### Storage in Effect Context (FUN_022bfb6c)

**Evidence:** `FUN_022bfb6c`
```c
if (*(char *)(iVar2 + 0x28) == -1) {
    // No attachment point (0xFF) → zero offset
    *(undefined2 *)(iVar2 + 0x24) = 0;
    *(undefined2 *)(iVar2 + 0x26) = 0;
} else {
    // Store raw offset from FUN_0201cf90
    *(undefined2 *)(iVar2 + 0x24) = *param_3;  // offset X
    *(undefined2 *)(iVar2 + 0x26) = param_3[1]; // offset Y
}
```

| Context Offset | Size | Field |
|----------------|------|-------|
| 0x24 | i16 | Resolved attachment offset X |
| 0x26 | i16 | Resolved attachment offset Y |
| 0x28 | i8 | Attachment point index (0-3 or -1/0xFF) |

### Layers 1 & 3: Secondary / Projectile

These layers use `FUN_022bf01c` → `FUN_022bf088` which supports **per-species overrides** via the `special_animations` table. The base attachment point still comes from the effect animation data, but can be overridden per-Pokemon.

**Evidence:** `FUN_02322f78` (projectile spawner)
```c
uVar2 = FUN_022bf01c((int)*(short *)(iVar5 + 4), (uint)uVar1);
if (uVar2 == 0xffffffff) {
    // Override returned "no point" → use global default
    sStack_24 = *DAT_023230f8;
    sStack_22 = DAT_023230f8[1];
} else {
    FUN_0201cf90(&sStack_24, (ushort *)(param_1 + 0xb), uVar2 & 0xff);
}
```

Note: Projectile attachment reads from the **attacker** (`param_1 + 0xb`), once, at launch. The index is the move's `attachment_point_idx`, not the effect's `field_0x19`. The offset is written to `ctx + 0x24/0x26` and decays toward (0, −9) every frame; it is never re-read from the attacker or applied to the target. See `Systems/projectile_motion.md`.

## The (99, 99) Sentinel

### Where It IS Handled

`PlayEffectAnimationEntity` explicitly checks both components:

```c
FUN_0201cf90(&local_48, &(entity->anim_ctrl).some_bitfield, param_4);
iVar4 = (int)local_48;

if (iVar4 != 99 && param_4 != 99) {
    // Normal: entity_pos + offset * 256
    uVar8 = (entity->pixel_pos.x + iVar4 * 0x100) >> 8;
    uVar9 = (entity->pixel_pos.y + param_4 * 0x100 - height_offset) >> 8;
} else {
    // Sentinel: use entity position directly, ignore offset
    uVar8 = entity->pixel_pos.x >> 8;
    uVar9 = (entity->pixel_pos.y - height_offset) >> 8;
}
```

Semantics: (99, 99) means "use entity position directly, don't add any attachment offset." Each component is checked independently.

### Render Tick

`FUN_022bf4f0` checks each component of `ctx + 0x24/0x26`: if either is 99, the effect is **not drawn** that tick and its velocity is not applied. This differs from `PlayEffectAnimationEntity`, which falls back to the entity position.

`PlayMoveAnimation`, `FUN_022e6e80` and `FUN_022bfb6c` pass the raw value through, so a bound effect whose entity frame returns (99, 99) disappears for that frame.

For projectiles, the decay turns a (99, 99) launch offset into about (82, 81) after one step, so the sentinel survives only for its undrawn first frame and the projectile is then drawn far down-right. Probable ROM bug.

## Summary Table: Who Reads What

| Path | Reads From | Override? | (99,99) Check? |
|------|-----------|-----------|----------------|
| Layer 2 initial spawn | Target | No | No |
| Layer 2 per-frame track | Target | No | No |
| Layer 1 (secondary) | Target | Yes (per-species) | Caller-dependent |
| Layer 3 (projectile) | **Attacker**, once at launch (attack loop's break frame) | Yes (per-species) | Tick skips draw; decay mangles it |
| PlayEffectAnimationEntity | Entity param | No | **Yes** |

## Implementation Guidance

For the game client:

1. **Layer 2 effects:** Read attachment point from target's current frame, add to target position. No sentinel handling needed.
2. **Projectiles:** Read the move's attachment point from the attacker once at launch; −1 means (0, 0) and no visible offset. Decay it each frame per `projectile_motion.md`. Never apply the target's attachment.
3. **PlayEffectAnimationEntity path:** Check for (99, 99) and fall back to entity position with no offset.
4. **Per-frame updates:** Re-read the target's attachment point each frame — don't cache the initial value.

## Cross-References

> See `Data Structures/meta_frame.md` for offset table structure and (99, 99) in `FUN_0201cf90`

> See `Systems/move_effect_pipeline.md` for layer spawning and `FUN_022e6d68` binding

> See `Systems/positioning_system.md` for entity position semantics and coordinate systems

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `PlayMoveAnimation` | - | Layer 2 spawn, reads target attachment |
| `FUN_022e6e80` | `0x022e6e80` | Per-frame attachment tracking (3-slot table) |
| `FUN_022bfb6c` | `0x022bfb6c` | Store resolved offset in effect context |
| `FUN_0201cf90` | `0x0201cf90` | Read attachment offset from animation frame |
| `FUN_022bf01c` | `0x022bf01c` | Get attachment index with per-species override |
| `FUN_022bf088` | `0x022bf088` | Species-specific attachment override lookup |
| `PlayEffectAnimationEntity` | - | Alternative spawn path with (99,99) handling |
| `FUN_022bfc5c` | `0x022bfc5c` | Layer 2 effect dispatcher |
| `FUN_022be9e8` | `0x022be9e8` | Layer 3 projectile dispatcher |

## Open Questions

- What are the global default offset values at `DAT_023258e8`? Likely (0, 0) but unconfirmed.
- Does `FUN_022bfb6c`'s direction-based offset table (`DAT_022bfc58`) interact with attachment points?
- Complete effect context layout (offsets 0x20-0x2C and their rendering usage)
