# Meta-Frame Structure

## Summary

- Meta-frames define how sprite pieces combine to form a single animation frame
- Each meta-frame contains multiple pieces (10 bytes each)
- Pieces specify image fragment, position offset, and flip flags
- Flip flags are baked into WAN data at authoring time, not applied at runtime
- Last piece indicated by bit 11 of piece[3]

## Meta-Frame Piece Structure

Each piece within a meta-frame is 10 bytes.

```c
struct meta_frame_piece {
    uint16_t piece_0;    // 0x00: Image/fragment reference
    uint16_t piece_1;    // 0x02: Y offset data
    uint16_t piece_2;    // 0x04: X offset + attribute bits
    uint16_t piece_3;    // 0x06: Flags (is_last, flip_h, flip_v)
    uint16_t piece_4;    // 0x08: Tile/palette data
};
// Size: 10 bytes (0x0A)
```

## Piece Field Details

### piece[0] (offset 0x00)

Image or fragment reference index.

### piece[1] (offset 0x02)

Y offset data for positioning this piece relative to sprite origin.

### piece[2] (offset 0x04)

X offset and additional attribute bits.

### piece[3] (offset 0x06) - Flags

| Bits | Mask | Field | Description |
|------|------|-------|-------------|
| 0-10 | 0x07FF | Reserved | Size/shape bits |
| 11 | 0x0800 | is_last | 0=more pieces follow, 1=last piece in meta-frame |
| 12 | 0x1000 | h_flip | Horizontal flip (baked into WAN) |
| 13 | 0x2000 | v_flip | Vertical flip (baked into WAN) |
| 14-15 | 0xC000 | Size bits | OAM size attribute |

**Evidence:** Meta-frame iteration in `FUN_0201c5c4`
```c
while (true) {
    // Process piece...
    
    // Check is_last flag (bit 11 of piece[3])
    if (((int)(uint)puVar17[3] >> 0xb & 1U) != 0) break;
    
    puVar17 = puVar17 + 5;  // Next piece (10 bytes = 5 uint16_t)
}
```

### piece[4] (offset 0x08)

Tile index and palette information for OAM.

## Flip Flag Behavior

**Critical:** Flip flags are baked into the WAN file at authoring time. The ROM does NOT apply runtime direction-based flipping.

### Why Flips Are Pre-Baked

1. CHARA sprites have 8 direction variants - each direction's meta-frames have appropriate flips already applied
2. PROPS_UI sprites (effects) either:
   - Have multiple directional sequences (8 pre-rotated versions)
   - Are non-directional (same appearance regardless of direction)

### No Runtime Flip Modification

**Evidence:** `FUN_0201b678` (piece parser)
```c
void FUN_0201b678(undefined2 *output, undefined2 *piece) {
    uVar1 = DAT_0201b6d0;  // Mask constant
    
    output[0] = piece[0];
    output[1] = piece[1];
    output[2] = piece[2] & (ushort)uVar1;
    output[3] = piece[3] & ((ushort)uVar1 - 0xb00);  // Flip flags pass through
    output[4] = piece[4];
    // ...
}
```

No direction parameter is passed to this function. Flip flags from piece[3] are preserved unchanged.

## DS OAM Mapping

Meta-frame pieces are converted to DS OAM (Object Attribute Memory) entries.

### OAM Attribute 1 (contains flip flags)

| Bit | Field |
|-----|-------|
| 0-8 | X coordinate |
| 9-11 | Reserved |
| 12 | Horizontal Flip |
| 13 | Vertical Flip |
| 14-15 | Size (combined with Attr0) |

### OAM Builder

**Evidence:** `FUN_0201b6d4`
```c
if (param_4 == NULL) {
    // Use values from meta-frame directly
    local_3a = local_2e;  // attr0
    local_38 = local_2c;  // attr1 (contains flip flags from piece[3])
    local_36 = local_2a;  // attr2
}
else {
    // Apply optional mask/override (param_4 initialized to 0 by FUN_0201c000)
    local_3a = param_4[3] | local_2e & *param_4;
    local_38 = param_4[4] | local_2c & param_4[1];
    local_36 = param_4[5] | local_2a & param_4[2];
}
```

### OAM Override Mask

The OAM override mask is initialized to zero, meaning no external flip modification occurs.

**Evidence:** `FUN_0201c000` initializes OAM attribute info
```c
// Masks initialized to 0 - no override applied
```

## Meta-Frame Iteration

Meta-frames are processed piece by piece until the is_last flag is encountered.

```c
// Pseudocode for meta-frame rendering
piece_ptr = meta_frame_start;
while (true) {
    // 1. Parse piece to OAM attributes
    FUN_0201b678(output, piece_ptr);
    
    // 2. Build final OAM entry
    FUN_0201b6d4(...);
    
    // 3. Check is_last flag
    if (piece_ptr[3] & 0x0800) break;  // Bit 11 set = last piece
    
    // 4. Advance to next piece
    piece_ptr += 5;  // 10 bytes = 5 uint16_t
}
```

## Attachment Point Offsets

Each frame can have attachment point offsets stored separately (not in meta-frame pieces).

### Offset Table Structure

Per-frame, 4 attachment points × 4 bytes each = 16 bytes per frame.

```c
struct frame_offsets {
    int16_t head_x, head_y;        // Index 0: Head
    int16_t left_hand_x, left_hand_y;   // Index 1: Left hand
    int16_t right_hand_x, right_hand_y; // Index 2: Right hand
    int16_t center_x, center_y;    // Index 3: Center
};
// Size: 16 bytes (0x10) per frame
```

### Special Value

If offset is (99, 99), use sprite center instead of offset lookup.

**Evidence:** `FUN_0201cf90`
```c
if (*point == 99 && *(point + 1) == 99) {
    *output = 0x63;      // 99
    output[1] = 0x63;    // 99
    return;
}
```

## Open Questions

- Complete piece[0] format (image reference encoding)
- How piece[1] and piece[2] encode position (fixed point? direct pixel?)
- Relationship between piece[4] and VRAM tile allocation
- How palette selection works within piece data

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_0201c5c4` | `0x0201C5C4` | Meta-frame renderer main loop |
| `FUN_0201b678` | `0x0201B678` | Parse meta-frame piece to OAM attributes |
| `FUN_0201b6d4` | `0x0201B6D4` | Build final OAM entry |
| `FUN_0201c000` | `0x0201C000` | Initialize OAM attribute info (masks to 0) |
| `FUN_0201cf90` | `0x0201CF90` | Calculate attachment point offset |
