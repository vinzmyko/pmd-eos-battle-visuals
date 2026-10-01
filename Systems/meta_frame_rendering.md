# Meta-Frame Rendering

## Summary

- Meta-frames are rendered by iterating through pieces until is_last flag
- Each piece is parsed to OAM attributes, then inserted into a 320-entry draw order bucket list via `FUN_0200b6f0`
- DS OAM priority (attr2 bits 10-11) comes from WAN `piece[4]` and passes through the renderer unchanged; every scanned WAN uses priority 3
- Within-layer ordering is driven by **draw order**: a per-sprite value (`animation_control + 0x38`) plus a signed per-piece `draw_order_offset` (high byte of `piece[1]`). Higher draw order draws in front
- Pokémon draw order = feet screen Y / 2. Effects bound to a Pokémon use that Pokémon's draw order ±1. Some effects use large offsets (e.g. −127 for Night Shade) to sit behind or in front of everything
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
    │  FUN_0200b6f0 (bucket insert)     │
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

**Render parameters:** for each piece, `FUN_0201c5c4` builds a parameter block (`local_6c`, passed to `FUN_0201b6d4` as `param_3`) from the animation control:

| `param_3[n]` | Source (`animation_control` offset) | Meaning |
|---|---|---|
| [0] | `+0x1C` + `+0x20` | Screen X (base + sequence frame offset) |
| [1] | `+0x1E` + `+0x22` | Screen Y (base + sequence frame offset) |
| [2] | `+0x32` | Tile base |
| [3] | `+0x38` | **draw_order** |
| [4] | byte `+0x41` | Palette |
| [5] | byte `+0x42` | Palette add |
| [6] | table `DAT_0201cf50` | Colour mode |

`FUN_0201c5c4` does not read `effect_context + 0x2C` itself. The effect tick `FUN_022bf4f0` copies it into the effect's `animation_control + 0x38` before calling the renderer. See "Draw Order Sources" below.

`param_4` (OAM override) is `animation_control + 0x10` when bit `0x20` of `animation_control + 0x2` is set, otherwise NULL.

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
    output[5] = ((piece[2] & 0x3FF) << 4) | ((piece[3] & 0xE00) >> 9);  // 10-bit Y offset << 4 | attr1 bits 9-11
}
```

**Key observation:** No direction parameter. The function simply masks and copies piece data. Flip flags in piece[3] bits 12-13 pass through unchanged. `piece[1]` is copied unmodified.

### Piece Word Layout (10 bytes)

| Word | Byte | Content | Community name |
|---|---|---|---|
| `piece[0]` | 0x0 | Fragment image index (int16, −1 = reuse previous) | pmd_wan `fragment_bytes_index` |
| `piece[1]` | 0x2 | Low byte: unknown (0x00 in effect 330). High byte: **`draw_order_offset`** (int8) | pmd_wan `unk1`; pmdsky-debug `wan_fragment.unk1` / `unk2` |
| `piece[2]` | 0x4 | attr0: Y offset (renderer reads low 10 bits, centred at 0x200), mosaic bit 12, colour mode bit 13, shape bits 14-15 | |
| `piece[3]` | 0x6 | attr1: X offset bits 0-8, is_last bit 11, H/V flip bits 12-13, size bits 14-15 | |
| `piece[4]` | 0x8 | attr2: tile/alloc bits 0-9, **OAM priority bits 10-11**, palette bits 12-15 | pmd_wan `alloc_and_palette` |

The names `draw_order` and `draw_order_offset` are ours. Neither SkyTemple (`pmd_wan`) nor pmdsky-debug names these fields.

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

**The mask is `0xFC00`, not `0xF000`.** Bits 10-11 (the 2-bit OAM priority field) are preserved from `local_36`, which is loaded from `local_2a` = parsed `output[4]` = **`piece[4]`** (attr2), copied unmodified by `FUN_0201b678`. Earlier versions of this doc said `piece[3]`; that was wrong (`piece[3]` is attr1, whose bit 11 is is_last). OAM priority is **baked into WAN piece data**. The only thing that can change it is a non-NULL OAM override (`local_36 = param_4[5] | local_2a & param_4[2]`).

After attribute construction, `FUN_0201b6d4` calls `FUN_0200b6f0` at `0x0201b990` to insert the entry into the draw order bucket list.

## Draw Order Bucket List (FUN_0200b6f0)

The OAM staging buffer is a **320-bucket linked list** indexed by fragment draw order (see "Bucket Index" below).

The render context is the global `DAT_020afc4c`, initialised by `FUN_0201bb3c` (called once from `MainLoop`). It holds four sub-lists of `0x70` bytes each (`+0x0`, `+0x70`, `+0xE0`, `+0x150`), selected per sprite by `animation_control` byte `+0x7A` via the table `DAT_0201cf4c`. Each sub-list has its own bucket buffer at `+0x20`.

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

Each new entry is head-inserted into its bucket, so each bucket's list runs newest-first. Whether the drain emits that order as-is (newest on top) is not yet traced.

### Bucket Index = Fragment Draw Order

The `bucket_idx` parameter is `FUN_0201b6d4`'s `iVar12`. It is the sprite's `draw_order` (`param_3[3]`, from `animation_control + 0x38`) plus the piece's signed `draw_order_offset`: `local_2f`, the high byte of parsed `output[1]` = `piece[1]`.

```
ldrh  r7,[r6,#0x6]        ; r7 = param_3[3] = draw_order
...
ldrsb r0,[sp,#local_2f]   ; signed high byte of piece[1] = draw_order_offset
adds  r7,r7,r0            ; fragment draw order
movmi r7,#0x0             ; clamp low to 0
cmp   r7,#0x140
ldrge r7,[DAT_0201b9ac]   ; clamp high to 0x13F (319)
```

The piece's screen Y is computed separately, as `(output[5] >> 4) + param_3[1] - 0x200`, and does **not** affect the bucket. Earlier versions of this doc said bucket = base screen Y + piece Y offset; that was wrong on both counts.

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
| 10-11 | Priority | **Preserved from WAN piece[4]** — renderer mask is `0xFC00` |

## Layering Model

### DS OAM Priority (attr2 bits 10-11) — "Layer"

Baked into WAN meta-frame piece data at authoring time. The renderer does not compute or modify this. Use cases:
- Floor / dungeon tile layer
- Entity layer
- Status icon / UI overlay

In practice, every fragment in every scanned WAN uses priority 3 (scraper scan: 664 effect sequences, 50 Pokémon). SkyTemple's `pmd_wan` writer also hardcodes `0x0C00` (priority 3) into attr2. Multi-layer effects (part behind the entity, part in front) are built with `draw_order_offset`, not OAM priority. Pokémon priority may still be changed at runtime by the OAM override; see Open Questions.

### Draw Order — "Within-Layer Ordering"

Within a given OAM priority level, the final draw order is determined by:
1. Fragment draw order bucket (0-319): **higher draws in front**
2. Within a bucket: insertion order (LIFO list; which end draws on top depends on the drain)

**The direction is inferred, not traced.** Higher = front is the only direction consistent with all three of these: (a) Pokémon lower on screen draw in front; (b) primary hit effects at target draw order +1 draw in front of the target; (c) Night Shade (offset −127, clamped to bucket 0) draws behind everything in ROM observation.

### Draw Order Sources

`animation_control + 0x38` is written by each object's per-frame update before it calls the renderer.

| Object | draw_order | Written by |
|---|---|---|
| Pokémon | `((pixel_y >> 8) − camera_y) / 2`, −1 if no shadow (feet Y; ignores elevation and hop) | `FUN_02303f18` → `entity + 0x64` |
| Bound effect, bind_type 6 (primary) | entity draw order + 1 | `FUN_022bfb6c` → `effect_context + 0x2C` |
| Bound effect, other bind types, directional | entity draw order + `DIRECTION_DRAW_ORDER_TABLE[dir]` (±1) | `FUN_022bfb6c` → `effect_context + 0x2C` |
| Bound effect, other bind types, non-directional | entity draw order + 1 | `FUN_022bfb6c` → `effect_context + 0x2C` |
| Unbound effect (`+0x2C == DAT_022bf75c`) | `(current_y − camera_y) / 2` + 3 (−3 if directional and facing 3/4/5) | `FUN_022bf4f0` |

For effects, `FUN_022bf4f0` copies the result to `effect_context + 0xA0` every tick. The effect's animation control sits at `+0x68`, so `+0xA0` is its `animation_control + 0x38`. On screen, Pokémon draw order normally spans about 0–96.

**Evidence:** `FUN_02303f18` (Pokémon)
```c
iVar16 = ((*(int *)(param_1 + 0x10) >> 8) - (int)camera_y) / 2;   // feet screen Y / 2
if (monster->display_shadow == '\0') iVar16 = iVar16 + -1;
FUN_022e6e80(param_1, iVar16);                // forwarded as the base for bound effects
*(short *)(param_1 + 100) = (short)iVar16;    // entity + 0x64 = animation_control (+0x2C) + 0x38
```

**Evidence:** `FUN_022bf4f0` (effects)
```c
iVar11 = param_1[0xb];                              // effect_context + 0x2C
if (iVar11 == DAT_022bf75c) {                       // DRAW_ORDER_FROM_Y sentinel
    iVar11 = iVar9 + (current_y - camera_y) / 2;    // iVar9 = -3 if directional && dir in {3,4,5}, else 3
}
*(short *)(param_1 + 0x28) = (short)iVar11;         // effect_context + 0xA0 = animation_control + 0x38
FUN_0201cf5c((ushort *)(param_1 + 0x1a), ...);      // render
```

### Draw Order Offset Data

Source: a scraper scan (histogram of `piece[1] >> 8`) over the base sequence of every effect.bin WAN effect (direction 0 for directional effects; Screen/type 5 effects excluded), plus 50 Pokémon (monster.md 1–50, all animations).

- **Pokémon:** every fragment is 0, so a Pokémon sorts as a single unit.
- **Effects:** 664 sequences scanned. 529 are all 0, 25 use one non-zero offset throughout, and 110 mix offsets. None use more than three distinct values.

**Uniform non-zero (25)**: the whole effect shifts by one offset.

| Offset | Effect IDs | Meaning |
|---|---|---|
| −127 | 330 (Night Shade) | Behind everything |
| −24 | 478, 693, 694, 695 | Behind |
| −10 | 643 | Behind |
| −5 | 63, 121, 173, 390, 453 | Behind |
| +2 | 65 | In front |
| +5 | 27, 496, 500, 529, 536, 562, 578 | In front |
| +12 | 26, 135, 284 | In front |
| +18 | 157 | In front |
| +127 | 351, 363 | In front of everything |

**Mixed (110)**: different fragments of one effect use different offsets.

| Offsets | Effect IDs |
|---|---|
| {−5, 0} (92) | 2, 7, 13, 14, 17, 59, 66, 75, 115, 126, 129, 130, 131, 132, 134, 147, 148, 198, 201, 238, 241, 244, 245, 247, 261, 264, 272, 285, 286, 306, 310, 316, 342, 344, 348, 386, 388, 391, 394–404, 425, 426, 427, 432, 433, 435, 436, 444, 456, 457, 460, 474, 488, 492, 506, 520, 524, 527, 528, 542, 546, 547, 565, 576, 577, 588, 592, 604, 606, 611, 618, 654, 672, 674, 680–687, 691 |
| {−16, 0} | 625, 626, 627, 628, 629, 646, 664 |
| {−127, 0} | 31, 70, 389 |
| {−5, 32} | 12, 387 |
| {−5, 4} | 243 |
| {−5, 5} | 634 |
| {0, 5} | 484 |
| {0, 127} | 250 |
| {−5, 0, 32} | 248 |
| {−5, 0, 64} | 249 |

The {−5, 0} pattern is how the ROM wraps an effect around a Pokémon: the −5 fragments sort about 4 behind the target and the 0 fragments 1 in front. All stat up/down effects (394–404) use it, as does Solar Beam's charge (247).

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

To reproduce ROM layering:

1. Give every Pokémon a `draw_order` of feet screen Y / 2 (−1 if it has no shadow). Use the screen Y after any board flip.
2. Give every effect a `draw_order` per "Draw Order Sources": bound effects take their entity's draw order (+1 for primary; the direction table for directional non-primary; +1 otherwise). Unbound effects use their own screen Y / 2 ± 3.
3. Add each fragment's `draw_order_offset`, clamp to 0–319, and draw higher values in front.
4. Treat OAM priority as uniform (3) unless the Pokémon OAM override question resolves otherwise.

The scraper flattens all fragments of a frame into one image, so an effect whose fragments use different offsets cannot be reproduced with a single sprite.

> **Client status (deferred):** Not implemented. The client currently approximates layering in `MoveEffectPlayer._update_z_index` (charge layer `z_index − 1`, all other layers `+1`). Godot's `z_index` overrides Y-sort, so this also draws every non-charge effect above every Pokémon, not just its target. Planned in two phases:
> - **Phase 1 (client only):** a per-sprite draw order as above, with a single offset per effect: the uniform value for the 25 uniform non-zero effects, 0 for mixed effects. Fixes Night Shade (330) and the over-everything bug.
> - **Phase 2 (scraper + client):** export one image per distinct offset for the 110 mixed effects and spawn one layer per image, so effects wrap around Pokémon (−5 behind, 0 in front) as in the ROM.

- Complete sprite type 3 rendering path
- **Bucket drain:** where the bucket list is walked into OAM (`0x07000000`, or a shadow copy plus DMA). Needed only to confirm that a higher bucket draws in front (currently inferred) and which end of a bucket's LIFO list draws on top. Candidates: the functions containing the `DAT_020afc4c` references at `0x0201BCD4`–`0x0201BFBC`.
- Value of `DAT_022bf75c` (the `DRAW_ORDER_FROM_Y` sentinel) and where `effect_context + 0x2C` is set to it at spawn
- **Pokémon OAM override:** `FUN_02303f18` sets bit `0x20` at `animation_control + 0x2` and passes a 6-word block to `FUN_0201d110`. In that block, the attr2 OR value is `(dungeon + 0x1A23C) << 10` and the attr2 mask is `(short)DAT_023046e0`. If the mask clears bits 10-11, Pokémon OAM priority comes from `dungeon + 0x1A23C`, not from WAN data. ROM observation (effects drawing both in front of and behind Pokémon) is consistent with equal priority, but neither value is confirmed. The same block ORs `0x400` into attr0 (semi-transparent OBJ mode) under an invisibility condition.
- Whether effects also use an OAM override: `FUN_022bf4f0` calls `FUN_0201d110(animation_control, effect_context + 0x30)`; it's unknown whether bit `0x20` is set for effects
- The 320-bucket count: on-screen draw order spans about 0–96, so the extra range likely leaves room for large positive offsets (+127) rather than off-screen scrolling

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_0201c5c4` | `0x0201C5C4` | Meta-frame renderer main loop |
| `FUN_0201b678` | `0x0201B678` | Parse meta-frame piece to OAM attributes |
| `FUN_0201b6d4` | `0x0201B6D4` | Build final OAM entry and submit to draw order bucket |
| `FUN_0200b6f0` | `0x0200B6F0` | Draw order bucket insertion |
| `FUN_0201c000` | `0x0201C000` | Initialize OAM attribute info (masks to 0) |
| `FUN_0201bb3c` | `0x0201BB3C` | Render context init (`DAT_020afc4c`), called once from `MainLoop` |
| `FUN_0201cf5c` | `0x0201CF5C` | Render wrapper: `FUN_0201c5c4`, then advance frame |
| `FUN_022bf4f0` | `0x022BF4F0` | Effect tick; copies `effect_context + 0x2C` to `animation_control + 0x38` |
| `FUN_02303f18` | `0x02303F18` | Pokémon per-frame update; writes Pokémon draw order |
| `FUN_0201d110` | `0x0201D110` | Appears to copy an OAM override block into `animation_control + 0x10` |
