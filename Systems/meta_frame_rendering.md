# Meta-Frame Rendering

## Summary

- Meta-frames are rendered by iterating through pieces until is_last flag
- Each piece is parsed to OAM attributes, then inserted into a Y-bucketed linked list via `FUN_0200b6f0`
- DS OAM priority (attr2 bits 10-11) is baked into WAN piece data and passes through the renderer unchanged
- Within-layer sprite ordering is driven by Y coordinate, not by any per-effect z value
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
    │  FUN_0200b6f0 (Y-bucket insert)   │
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
    // Handle different sprite types
    if (sprite_type == 3) {
        // WAN_SPRITE_UNK_3: Special 3D rendering path
        // ...
    }
    else {
        // Types 0, 1, 2: Standard meta-frame iteration
        puVar17 = meta_frame_start;
        
        while (true) {
            // Build OAM entry and insert into Y bucket
            FUN_0201b6d4(*DAT_0201cf54 + slot_offset, puVar17, &local_6c, local_8c);
            
            // Check is_last flag (bit 11 of piece[3])
            if (((int)(uint)puVar17[3] >> 0xb & 1U) != 0) break;
            
            // Advance to next piece (10 bytes)
            puVar17 = puVar17 + 5;
        }
    }
}
```

**Critical observation:** `FUN_0201c5c4` never reads `effect_context + 0x2C`. The z_priority value computed and stored by `FUN_022bfb6c` is not threaded into rendering. See `Data Structures/effect_context.md` "z_priority Semantics" for details.

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

Creates final DS OAM entry from parsed piece data and submits it via `FUN_0200b6f0`.

**Evidence:** `FUN_0201b6d4` at `0x0201b7e8-0x0201b82c` (attr2 construction):
```
ldrh  r1, [sp, #local_36]       ; current attr2
mov   r2, #0x400
rsb   r2, r2, #0x0              ; r2 = 0xFFFFFC00
and   r1, r1, r2                ; attr2 &= 0xFC00 (clear low 10 bits)
strh  r1, [sp, #local_36]
...
orr   r1, r0, r1                ; attr2 |= (tile_index & 0x3FF)
strh  r1, [sp, #local_36]
```

**The mask is `0xFC00`, not `0xF000`.** Bits 10-11 (the 2-bit OAM priority field) are preserved from whatever `local_36` came in with, which is `piece[3]` (via `FUN_0201b678`). This confirms OAM priority is **baked into WAN piece data** and untouched by the renderer.

After attribute construction, `FUN_0201b6d4` calls `FUN_0200b6f0` at `0x0201b990` to insert the entry into the Y-bucketed linked list.

## Y-Bucket Linked List (FUN_0200b6f0)

The OAM staging buffer uses a **256-bucket linked list** indexed by sprite Y coordinate. This is the standard DS technique for high sprite counts with per-scanline priority.

### Buffer Structure

`FUN_0201b6d4` passes `param_1 + 0x20` as the buffer pointer. The buffer is a struct:

| Offset | Field | Description |
|--------|-------|-------------|
| +0 | max_slots | Capacity (128 for hardware OAM) |
| +4 | num_buckets | Number of Y buckets (320 = 0x140) |
| +8 | slot_count | Current number of used slots |
| +12 | head_array | Pointer to int16 head-of-list array, one per bucket |
| +16 | entry_array | Pointer to 8-byte OAM entries |

### Entry Format (8 bytes)

| Offset | Field |
|--------|-------|
| +0 | attr0 |
| +2 | attr1 |
| +4 | attr2 |
| +6 | next (int16 slot index; previous head of this bucket) |

### Insertion

**Evidence:** `FUN_0200b6f0`
```c
void FUN_0200b6f0(int *buffer, undefined2 *oam_entry, int bucket_idx) {
    slot = buffer[2];                     // current count
    if (slot < buffer[0]) {                // within max slots
        if (bucket_idx < 0) bucket_idx = 0;
        else if (buffer[1] <= bucket_idx) bucket_idx = buffer[1] - 1;
        
        entries = buffer[4];
        *(u16*)(entries + slot*8 + 0) = oam_entry[0];  // attr0
        *(u16*)(entries + slot*8 + 2) = oam_entry[1];  // attr1
        *(u16*)(entries + slot*8 + 4) = oam_entry[2];  // attr2
        *(u16*)(entries + slot*8 + 6) = *(u16*)(buffer[3] + bucket_idx*2);  // next = current head
        
        buffer[2] = slot + 1;
        *(s16*)(buffer[3] + bucket_idx*2) = slot;  // update head
    }
}
```

Each new entry is head-inserted into its bucket. **Entries within the same bucket emit in reverse insertion order** (newest first).

### Bucket Index = Y Coordinate

The `bucket_idx` parameter comes from `FUN_0201b6d4`'s `iVar12`, computed from `piece[3]` (the Y field of the piece). It is clamped to `[0, 0x13F]` = `[0, 319]` at `0x0201b754-0x0201b758`:

```
cmp r7, #0x140            ; 320
ldrge r7, [DAT_0201b9ac]  ; clamp to 0x13F (319)
```

The clamp range of 0-319 accommodates sprites positioned above or below the 192px DS screen during scrolling.

## OAM Override Mask

The OAM override structure can modify final OAM attributes but is initialized to zero (passthrough).

### Override Structure
```c
struct oam_override {
    uint16_t mask_0;     // 0x00: AND mask for attr0
    uint16_t mask_1;     // 0x02: AND mask for attr1
    uint16_t mask_2;     // 0x04: AND mask for attr2
    uint16_t or_0;       // 0x06: OR value for attr0
    uint16_t or_1;       // 0x08: OR value for attr1
    uint16_t or_2;       // 0x0A: OR value for attr2
};
```

### Initialization (FUN_0201c000)

All masks and OR values initialized to 0, resulting in passthrough when applied.

When `oam_override == NULL` (the common case for effect rendering), `FUN_0201b6d4` uses direct passthrough of parsed piece values.

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

### Attribute 1 (attr1) — Contains Flip Flags

| Bits | Field |
|------|-------|
| 0-8 | X coordinate |
| 9-11 | Unused / Rotation |
| 12 | Horizontal Flip |
| 13 | Vertical Flip |
| 14-15 | Size |

### Attribute 2 (attr2)

| Bits | Field | Source |
|------|-------|--------|
| 0-9 | Tile index | Set by `FUN_0201b6d4` (`& 0x3FF`) |
| 10-11 | Priority | **Preserved from WAN piece[3]** — renderer mask is `0xFC00` |
| 12-15 | Palette | Computed from palette_index + offset |

## Layering Model

### DS OAM Priority (attr2 bits 10-11) — "Layer"

Baked into WAN meta-frame piece data at authoring time. The renderer does not compute or modify this. Use cases:
- Floor / dungeon tile layer
- Entity layer
- Status icon / UI overlay

A single effect WAN may contain multiple pieces with different priority values for multi-layer effects (e.g., a piece that renders behind the entity and another in front).

### Y-Bucket Sort — "Within-Layer Ordering"

Within a given OAM priority level, final draw order is determined by:
1. Y coordinate bucket (0-319)
2. Within bucket: reverse insertion order (LIFO)

The DS OAM hardware then emits sprites in OAM slot order, with earlier slots drawing on top. Because `FUN_0200b6f0` head-inserts into buckets, the LAST sprite inserted into a given Y bucket draws on top of earlier sprites in the same bucket.

**Practical effect:** sprites at the same Y overlap by tick processing order, not by any explicit z value. Sprites at different Y separate naturally through bucket ordering.

### effect_context + 0x2C is NOT Used Here

The `z_priority` field written by `FUN_022bfb6c` (including the directional ±1 from `DIRECTION_Z_TABLE` at `0x022C7890`) is not read by this rendering pipeline. Its consumer is elsewhere — see `Data Structures/effect_context.md` open questions.

## Why No Runtime Flipping

The ROM does not perform direction-based sprite flipping at runtime because:

1. **CHARA sprites** have 8 pre-made direction variants. Each direction's meta-frames already have correct flip flags baked in.

2. **PROPS_UI sprites** (effects) either:
   - Have 8 directional sequences (already rotated/flipped)
   - Are non-directional (symmetric, same appearance for all directions)

3. **Performance**: Baking flips at authoring time avoids per-frame direction checks and conditional flip logic.

## Sprite Type 3 Rendering

Sprite type 3 (WAN_SPRITE_UNK_3) uses a different rendering path, possibly for 3D effects. Not fully documented.

## Performance Considerations

- Meta-frame pieces are processed sequentially (no parallelism)
- Piece count affects render time (more pieces = slower)
- is_last flag allows early termination
- OAM hardware has 128 slots on DS (`buffer[0] = 128`)
- Y-bucket insertion is O(1) per sprite

## Implementation Notes for Client Recreation

For a pixel-perfect recreation of the layering system:

1. **Read piece priority (attr2 bits 10-11) from WAN data** — determines render layer (BG / entity / UI separation)
2. **Within each layer, sort sprites by Y coordinate ascending** — lower Y draws on top
3. **For ties at the same Y**, preserve tick insertion order with later inserts drawing on top (matches ROM LIFO bucket behaviour)
4. **Ignore effect_context + 0x2C for rendering** — it's not in the render path

The directional ±1 from `DIRECTION_Z_TABLE` affects +0x2C but not render order, so for rendering purposes it can be ignored.

## Open Questions

- Complete sprite type 3 rendering path
- Where the Y-bucket buffer is drained to actual DS OAM hardware (`0x07000000`). Likely a function triggered per VBlank that walks each bucket's linked list and writes to `OAMDATA`.
- Whether the 320-bucket count (vs. 192 screen height) is purely for off-screen scroll coverage or encodes something else
- What code path reads effect_context + 0x2C (not this one)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_0201c5c4` | `0x0201C5C4` | Meta-frame renderer main loop |
| `FUN_0201b678` | `0x0201B678` | Parse meta-frame piece to OAM attributes |
| `FUN_0201b6d4` | `0x0201B6D4` | Build final OAM entry and submit to Y-bucket |
| `FUN_0200b6f0` | `0x0200B6F0` | Y-bucket linked-list insertion |
| `FUN_0201c000` | `0x0201C000` | Initialize OAM attribute info (masks to 0) |
