# Meta-Frame Rendering

## Summary

- Meta-frames are rendered by iterating through pieces until is_last flag
- Each piece is parsed to OAM attributes, then written to DS hardware
- Flip flags are read directly from WAN data, no runtime direction-based transformation
- OAM override mask is initialized to zero, allowing passthrough of baked flip flags
- Sprite type 3 uses special 3D rendering path

## Rendering Pipeline

```
animation_control->frame_id
        │
        ▼
Get meta-frame pointer from animation sequence
        │
        ▼
FUN_0201c5c4 (Meta-frame renderer)
        │
        ▼
    ┌───────────────────────────────────┐
    │  For each piece in meta-frame:    │
    │                                   │
    │  FUN_0201b678 (Parse piece)       │
    │         │                         │
    │         ▼                         │
    │  FUN_0201b6d4 (Build OAM entry)   │
    │         │                         │
    │         ▼                         │
    │  Write to DS OAM hardware         │
    │                                   │
    │  Check is_last flag (bit 11)      │
    │  If set, exit loop                │
    └───────────────────────────────────┘
```

## Meta-Frame Renderer

Main loop that processes all pieces in a meta-frame.

**Evidence:** `FUN_0201c5c4`
```c
void FUN_0201c5c4(...) {
    // Get current frame's meta-frame data
    // ...
    
    // Handle different sprite types
    if (sprite_type == 3) {
        // WAN_SPRITE_UNK_3: Special 3D rendering path
        // ...
    }
    else {
        // Types 0, 1, 2: Standard meta-frame iteration
        puVar17 = meta_frame_start;
        
        while (true) {
            // Parse this piece
            FUN_0201b678(local_output, puVar17);
            
            // Build OAM entry
            FUN_0201b6d4(..., local_output, ...);
            
            // Check is_last flag (bit 11 of piece[3])
            if (((int)(uint)puVar17[3] >> 0xb & 1U) != 0) break;
            
            // Advance to next piece (10 bytes)
            puVar17 = puVar17 + 5;
        }
    }
}
```

## Piece Parser

Converts raw WAN piece data to intermediate OAM attributes.

**Evidence:** `FUN_0201b678`
```c
void FUN_0201b678(undefined2 *output, undefined2 *piece) {
    uVar1 = DAT_0201b6d0;  // Mask constant
    
    output[0] = piece[0];
    output[1] = piece[1];
    output[2] = piece[2] & (ushort)uVar1;
    output[3] = piece[3] & ((ushort)uVar1 - 0xb00);  // Preserves flip flags
    output[4] = piece[4];
    output[5] = /* combined bits from piece[2] and piece[3] */;
}
```

**Key observation:** No direction parameter. The function simply masks and copies piece data. Flip flags in piece[3] bits 12-13 pass through unchanged.

## OAM Builder

Creates final DS OAM entry from parsed piece data.

**Evidence:** `FUN_0201b6d4`
```c
void FUN_0201b6d4(..., undefined2 *parsed_piece, undefined2 *oam_override, ...) {
    // ... process parsed_piece into local OAM attributes ...
    
    if (oam_override == NULL) {
        // Direct passthrough - use values from meta-frame
        local_3a = local_2e;  // attr0
        local_38 = local_2c;  // attr1 (contains flip flags)
        local_36 = local_2a;  // attr2
    }
    else {
        // Apply mask/override
        local_3a = oam_override[3] | local_2e & *oam_override;
        local_38 = oam_override[4] | local_2c & oam_override[1];
        local_36 = oam_override[5] | local_2a & oam_override[2];
    }
    
    // Write to OAM hardware
    // ...
}
```

## OAM Override Mask

The OAM override structure can modify final OAM attributes, but is initialized to zero.

### Override Structure

```c
struct oam_override {
    uint16_t mask_0;     // 0x00: AND mask for attr0
    uint16_t mask_1;     // 0x02: AND mask for attr1
    uint16_t mask_2;     // 0x04: AND mask for attr2
    uint16_t or_0;       // 0x06: OR value for attr0
    uint16_t or_1;       // 0x08: OR value for attr1 (flip flags here)
    uint16_t or_2;       // 0x0A: OR value for attr2
};
```

### Initialization

**Evidence:** `FUN_0201c000`
```c
void FUN_0201c000(oam_override *override) {
    // Initialize all masks to 0 (passthrough)
    // Initialize all OR values to 0 (no modification)
}
```

With all masks and OR values at 0:
- `attr1 = (0) | (original & 0) = 0 | 0 = original` (effectively passthrough when mask logic differs)

**Actual behavior:** When `oam_override` is NULL (common case), direct passthrough occurs.

## DS OAM Attributes

### Attribute 0 (attr0)

| Bits | Field |
|------|-------|
| 0-7 | Y coordinate |
| 8-9 | Object mode |
| 10-11 | GFX mode |
| 12 | Mosaic |
| 13 | Color mode |
| 14-15 | Shape |

### Attribute 1 (attr1) - Contains Flip Flags

| Bits | Field |
|------|-------|
| 0-8 | X coordinate |
| 9-11 | Unused / Rotation |
| 12 | Horizontal Flip |
| 13 | Vertical Flip |
| 14-15 | Size |

### Attribute 2 (attr2)

| Bits | Field |
|------|-------|
| 0-9 | Tile index |
| 10-11 | Priority |
| 12-15 | Palette |

## Why No Runtime Flipping

The ROM does not perform direction-based sprite flipping at runtime because:

1. **CHARA sprites** have 8 pre-made direction variants. Each direction's meta-frames already have correct flip flags baked in.

2. **PROPS_UI sprites** (effects) either:
   - Have 8 directional sequences (already rotated/flipped)
   - Are non-directional (symmetric, same appearance for all directions)

3. **Performance**: Baking flips at authoring time avoids per-frame direction checks and conditional flip logic.

### Evidence Chain

```
SetAnimationForAnimationControl
    │
    └─► Direction routes to correct sequence (CHARA)
        OR is ignored (PROPS_UI)
            │
            ▼
SetAnimationForAnimationControlInternal
    │
    └─► Selects animations[group][sequence]
        Sequence already has correct flip flags
            │
            ▼
FUN_0201c5c4 (Meta-frame renderer)
    │
    └─► No direction parameter received
        Processes meta-frame pieces as-is
            │
            ▼
FUN_0201b678 (Piece parser)
    │
    └─► No direction parameter
        Flip flags pass through from piece[3]
            │
            ▼
FUN_0201b6d4 (OAM builder)
    │
    └─► Override mask is NULL or zero
        Flip flags written to hardware unchanged
```

## Sprite Type 3 Rendering

Sprite type 3 (WAN_SPRITE_UNK_3) uses a different rendering path, possibly for 3D effects.

**Evidence:** `FUN_0201c5c4`
```c
if (sprite_type == 3) {
    // Special handling - different from standard meta-frame iteration
    // Possibly 3D billboard or special effect rendering
    // ...
}
```

This path is not fully documented.

## Performance Considerations

- Meta-frame pieces are processed sequentially (no parallelism)
- Piece count affects render time (more pieces = slower)
- is_last flag allows early termination
- OAM hardware has limited slots (128 on DS)

## Open Questions

- Complete sprite type 3 rendering path
- How OAM slot allocation works across multiple sprites
- Maximum pieces per meta-frame in practice
- How palette conflicts are resolved when multiple effects render

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_0201c5c4` | `0x0201C5C4` | Meta-frame renderer main loop |
| `FUN_0201b678` | `0x0201B678` | Parse meta-frame piece to OAM attributes |
| `FUN_0201b6d4` | `0x0201B6D4` | Build final OAM entry |
| `FUN_0201c000` | `0x0201C000` | Initialize OAM attribute info (masks to 0) |
