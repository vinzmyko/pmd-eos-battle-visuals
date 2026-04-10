# Screen Effect System (Types 5/6)

## Summary

- Type 5/6 effects are BG overlay animations, NOT WAN sprites
- Files are SIR0-wrapped containing RLE-compressed tilemaps with blend/palette keyframes
- Rendered by writing tile data to an NDS BG layer each frame
- Alpha blending controlled via BLDALPHA shadow registers with keyframe interpolation
- Palette brightness fading via per-channel scaling on extended palette entries
- Command queue architecture: commands are queued then processed one-shot by main loop
- Maximum 2 groups × 2 simultaneous screen effects

## File Format

### Source Files

- Pack Archive ID: 3 (`PACK_ARCHIVE_EFFECT`)
- File Path: `/EFFECT/effect.bin`
- Type 5 file index: `effect_animation_info.file_index + 0x10C` (268)
- Type 6 file index: `effect_animation_info.file_index + 0x122` (290)
- Format: SIR0 container (confirmed by `HandleSir0TranslationVeneer` call in `FUN_02064a7c`)

### Loading

Types 5/6 bypass the WAN table loader entirely. Loaded via generic resource cache:

```
FUN_022c0450 (resource switch)
    → FUN_02063fc8(pack_id=3, file_index, 0xf) (generic pack load)
        → FUN_02064b0c (resource cache, 0x60-byte slots with ref counting)
            → AllocAndLoadFileInPack (raw data load)
                → HandleSir0TranslationVeneer (pointer translation)
```

**Evidence:** `FUN_022be44c` type 5 allocation
```c
if (iVar6 == 5) {
    uVar5 = FUN_022c0450(5, (short)peVar3->file_index);
    *(short *)(ptr + 0x39) = (short)uVar5;  // screen_effect_handle at +0xE4
    if (*(short *)(ptr + 0x39) == 0) return -1;
    FUN_022c01fc(*(short *)(ptr + 0x39), peVar3->field_0x0, peVar3->file_index, param_3);
}
```

**Evidence:** File index calculation in `FUN_022bfe18` / `FUN_022bfe28`
```c
// Type 5: file_index + 0x10C
// Type 6: file_index + 0x122
```

### SIR0 Internal Structure

After SIR0 pointer translation, the data contains:

```
SIR0 Translated Header:
  +0x08: uint32  group_count_minus_1    // Number of animation groups - 1
  +0x0C: ptr     group_array            // Pointer to array of group descriptors (each 0x20 bytes)
```

**Evidence:** `FUN_02064014`
```c
iVar1 = *(int *)(param_1 * 0x60 + *DAT_02064048 + 0xb0);  // SIR0 resource
FUN_020646f0(state, *(undefined4 **)(iVar1 + 0xc), param_2, *(uint *)(iVar1 + 8));
// iVar1 + 0x0C = group array pointer
// iVar1 + 0x08 = group_count - 1
```

### Animation Group Header

Each group is referenced via a structure with frame management data.

```c
struct screen_effect_group {
    int32_t frame_count;          // +0x00: Number of frames in this group
    ptr     frame_ptr_table;      // +0x04: Pointer to array of frame pointers (each 4 bytes)
    // ...
    struct {
        uint8_t is_looping;       // +0x08: If non-zero, animation loops
        uint8_t loop_start;       // +0x09: Frame to restart from when looping
        uint8_t loop_end;         // +0x0A: Frame that triggers loop restart
    } loop_control;
};
```

**Evidence:** `FUN_020642a8` frame bounds check
```c
if (*(char *)(iVar4 + 8) == '\0') {
    if (*param_2 <= (int)*param_3) {
        return 0;  // Past last frame, not looping → done
    }
} else if ((int)(uint)*(byte *)(iVar4 + 10) <= (int)*param_3) {
    *param_3 = (uint)*(byte *)(iVar4 + 9);  // Loop back to loop_start
}
```

**Evidence:** `FUN_02064c60` frame count at +0x0C, frame data offset at +0x08
```c
if (param_3 < *(uint *)(param_2 + 0xc)) {
    *param_4 = (int)*(short *)(param_3 * 0x2c + param_2 + *(int *)(param_2 + 8) + 8);
    return 1;
}
```

### Frame Entry

Each frame is 0x2C (44) bytes.

```c
struct screen_effect_frame {
    // +0x00: unknown (2 bytes)
    // +0x02: unknown (2 bytes)
    int16_t width;                // +0x04: Width in tiles
    int16_t height;               // +0x06: Height in tiles
    int16_t duration;             // +0x08: Duration in game frames (used as interpolation denominator)
    // +0x0A: unknown (2 bytes)
    int16_t param_a;              // +0x0C: Copied to playback state +0x16; also blend skip condition
    int16_t param_b;              // +0x0E: Copied to playback state +0x18
    int16_t param_c;              // +0x10: Copied to playback state +0x1A
    int16_t param_d;              // +0x12: Copied to playback state +0x1C
    // +0x14 to +0x1A: unknown (8 bytes)
    int32_t blend_value;          // +0x1C: 8.8 fixed point, target for BLDALPHA EVA/EVB interpolation
    // +0x20 to +0x22: unknown (4 bytes)
    uint16_t tile_data[];         // +0x24: RLE-compressed tile map entries
};
// Size: 0x2C (44 bytes) header + variable tile data
```

**Evidence:** `FUN_0206478c` reads frame fields
```c
iVar5 = *(int *)(param_2[1] + *param_3 * 4);  // Frame pointer from table
sVar1 = *(short *)(iVar5 + 6);   // height
sVar2 = *(short *)(iVar5 + 4);   // width
// ...
*(undefined2 *)(param_1 + 0x16) = *(undefined2 *)(iVar5 + 0xc);
*(undefined2 *)(param_1 + 0x18) = *(undefined2 *)(iVar5 + 0xe);
*(undefined2 *)(param_1 + 0x1a) = *(undefined2 *)(iVar5 + 0x10);
*(undefined2 *)(param_1 + 0x1c) = *(undefined2 *)(iVar5 + 0x12);
```

**Evidence:** `FUN_020642a8` blend interpolation reads `+0x1C`
```c
iVar3 = *(int *)(iVar5 + 0x1c);  // Current frame blend value
iVar5 = MultiplyByFixedPoint(x, *(int *)(iVar6 + 0x1c) - iVar3);  // Lerp to next frame
```

### Tile Data Encoding

RLE-compressed uint16 tile map entries starting at frame offset +0x24.

| Condition | Meaning |
|-----------|---------|
| `(entry & 0xF000) != 0` | Literal tile map entry — written directly to BG tilemap |
| `(entry & 0xF000) == 0` | Run of empty tiles — length = `entry & DAT_020648f8`, emit zeroes |

During a run, the run counter decrements per tile position. No new data is consumed until the run completes.

**Evidence:** `FUN_0206478c` tile loop
```c
if (uVar8 == 0) {
    puVar10 = puVar9 + 1;
    uVar3 = *puVar9;
    if ((uVar3 & 0xf000) == 0) {
        uVar8 = uVar3 & DAT_020648f8;  // Run length
        uVar3 = 0;                      // Write zero tile
    }
} else {
    uVar3 = 0;       // Still in run, write zero
    puVar10 = puVar9; // Don't advance data pointer
}
if (uVar8 != 0) {
    uVar8 = uVar8 - 1;
}
```

### Tile Map Entry Format

Standard NDS BG tile map entry (uint16):

| Bits | Purpose |
|------|---------|
| 0-9 | Tile index |
| 10 | Horizontal flip |
| 11 | Vertical flip |
| 12-15 | Palette row |

**Evidence:** `FUN_0200b3fc` writes raw uint16 to tilemap buffer with standard NDS BG layout:
- Row stride: 0x40 bytes (32 tiles × 2 bytes per entry)
- Screen block overflow at +0x7C0, +0x1000, +0x17C0 for 64×64 tile BGs
- Buffer base at `param_1 + 0x18`

## Resource Cache

Screen effect SIR0 files are stored in a resource cache pool.

| Property | Value |
|----------|-------|
| Slot size | 0x60 bytes |
| Base address | `*DAT_02064048` (heap-allocated) |
| Pool offset | +0xA8 from base (confirmed by `FUN_0206409c`) |
| Access | `slot_index * 0x60 + base + 0xA8` |

### Slot Layout (Partial)

```
Resource Cache Slot (0x60 bytes):
  +0x08: ptr     SIR0 translated data pointer (animation groups, frames)
  +0x0C: ...
  // Other fields: ref count, pack ID, file index, raw data pointer
```

## Command Queue

Screen effects use a fire-and-forget command queue between the effect system and the screen effect engine.

### Pool Layout

Allocated by `FUN_022bff30`, total size 0x84 bytes.

```
Command Queue Pool (0x84 bytes):
  +0x00: int32   context_param (set at init)
  +0x04: Slot[0][0] (0x20 bytes)
  +0x24: Slot[0][1] (0x20 bytes)
  +0x44: Slot[1][0] (0x20 bytes)
  +0x64: Slot[1][1] (0x20 bytes)
```

Maximum: 2 groups × 2 slots = 4 simultaneous commands.

**Evidence:** `FUN_022bffa4` reset loop
```c
do {
    iVar4 = 0;
    do {
        iVar2 = *piVar1 + 4 + iVar3 * 0x40;
        *(undefined4 *)(iVar2 + iVar4 * 0x20) = 0;
        // ...
        iVar4 = iVar4 + 1;
    } while (iVar4 < 2);
    iVar3 = iVar3 + 1;
} while (iVar3 < 2);
```

### Command Slot Layout

Populated by `FUN_022c0068`, consumed by `FUN_022c0304`.

```
Command Slot (0x20 bytes):
  +0x00: int32   (unused/clear flag — 0 = slot free)
  +0x04: int32   dispatch_type          // 5 or 6
  +0x08: int16   resource_handle        // SIR0 cache slot index
  +0x0C: int32   caller_context_param   // From effect spawner
  +0x10: int32   layer_mask             // Always 0x0E
  +0x14: int32   unused                 // Always 0
  +0x18: int32   screen_select          // 0 = main screen, 2 = sub screen
  +0x1C: int32   sentinel               // Always 0xFFFFFFFF
```

**Evidence:** `FUN_022c0068`
```c
if (param_1 == 2) {
    local_28 = 5;       // dispatch_type
    local_18 = 0xe;     // layer_mask
    local_14 = 0;
    local_c = 0xffffffff;
    local_20 = param_3;
    iVar1 = FUN_022bd744();
    if (iVar1 == 1) {
        local_10 = 0;   // main screen
    } else {
        local_10 = 2;   // sub screen
    }
}
```

### Screen Select

`FUN_022bd744` returns 1 when the dungeon is using screen layout 1 (likely top screen for gameplay), otherwise 0.

| FUN_022bd744 Return | screen_select | Meaning |
|---------------------|---------------|---------|
| 1 | 0 | Main screen (top) |
| Other | 2 | Sub screen (bottom) |

## Playback State

Playback state is stored in 0x20-byte entries at `*DAT_02064048 + param_3 * 0x20`, separate from the effect_context.

### Relationship to effect_context

The `short *param_1` passed to `FUN_020642a8` is `effect_context + 0xE8` (passed as `param_1 + 0x3A` in word offsets). However, `FUN_020642a8` accesses up to byte offset +0x3E from this base, meaning the screen effect state extends from `effect_context + 0xE8` through at least `effect_context + 0x126`.

### Control Structure Fields

Based on `FUN_020642a8` access patterns (offsets relative to `param_1` as `short*`):

```
Screen Effect Control (at effect_context + 0xE8):
  short[0]  (+0x00): int16   resource_handle (SIR0 cache index)
  short[2]  (+0x04): int32   complex_mode_ptr (non-zero = complex blend mode)
  short[4]  (+0x08): int32   simple_mode_ptr (non-zero = simple frame sequence mode)
  short[6]  (+0x0C): int32   bg_layer_handle
  short[8]  (+0x10): int32   frame_index (current frame, advances each duration expiry)
  short[10] (+0x14): int32   duration_countdown (decrements each tick)
  short[14] (+0x1C): uint8   palette_fade_active (at byte offset 0x1C from base)
  short[16] (+0x20): int32   brightness_current (fixed point, 0 to DAT_02064660)
  short[18] (+0x24): int32   brightness_step (signed, added to brightness each tick)
  short[30] (+0x3C): uint8   blend_complete_flag
  short[31] (+0x3E): uint8   cleanup_pending_flag
```

### Two Playback Modes

| Mode | Condition | Behavior |
|------|-----------|----------|
| Simple | `complex_mode_ptr == 0 && simple_mode_ptr != 0` | Frame sequence playback via `FUN_02063f78`/`FUN_02063f30` |
| Complex | `complex_mode_ptr != 0` | Tilemap rendering + keyframe blend interpolation + palette fading |

## Rendering

### Tilemap Writing

`FUN_0206478c` writes RLE tile data to a BG tilemap buffer each frame.

- Grid bounds: X = 0–31, Y = 0–23 (32×24 tile standard NDS BG)
- Before writing, `FUN_0206466c` clears the full tilemap (64×32 grid)
- After writing, `FUN_0200b330` flushes the buffer to VRAM

**Evidence:** `FUN_0206466c` clear loop
```c
do {
    iVar1 = 0;
    do {
        local_28 = iVar1;
        local_24 = iVar2;
        FUN_0200b3fc(*(undefined **)(param_1 + 8), &local_28, 0);  // Write zero
        iVar1 = iVar1 + 1;
    } while (iVar1 < 0x40);  // 64 columns
    iVar2 = iVar2 + 1;
} while (iVar2 < 0x20);  // 32 rows
```

### Alpha Blending

`FUN_020094c4(eva, evb, screen_id)` writes to BLDALPHA shadow registers.

- EVA and EVB stored in separate tables indexed by `screen_id * 10`
- Shadow registers flushed to hardware during VBlank
- Blend mode set via `FUN_02063bcc(mode, handle)`:

| Mode | Meaning |
|------|---------|
| 0 | Blend off |
| 2 | Alpha blend active |
| 3 | Special blend (used during palette fade) |

**Evidence:** `FUN_020094c4`
```c
*(undefined2 *)(DAT_020094e0 + param_3 * 10) = param_1;  // EVA
*(undefined2 *)(DAT_020094e4 + param_3 * 10) = param_2;  // EVB
```

### Blend Interpolation

For non-final frames with `param_a == 0` and `blend_value != 0xFF00`:

1. Compute interpolation factor: `t = (duration - remaining) / duration` (8.8 fixed point via `FUN_02001ab0`)
2. Lerp blend value: `blend = current_blend + t * (next_blend - current_blend)` (via `MultiplyByFixedPoint`)
3. Convert to EVA/EVB: `EVA = blend >> 9` (approx), `EVB = 0x80 - EVA`
4. On final frame: `EVA = blend >> 8`, `EVB = 0xFF - EVA`

When interpolated blend reaches 0xFF, blend mode is set to 0 (off) — fully opaque.

### Palette Brightness Fading

When `palette_fade_active` is set, each tick:

1. Read 16 palette entries starting at SIR0 data `+0x10` offset `+0x380` (extended palette area)
2. Scale each RGB component: `component = (original * brightness) >> 5` (brightness on 0–32 scale)
3. Write to palette slots starting at index 0xE0
4. Advance brightness: `brightness += brightness_step`
5. Clamp: 0 ≤ brightness ≤ `DAT_02064660`

**Evidence:** `FUN_0200c020`
```c
local_10 = (undefined)((uint)*param_2 * param_4 >> 5);       // R
local_f  = (undefined)((uint)param_2[1] * param_4 >> 5);     // G
local_e  = (undefined)((uint)param_2[2] * param_4 >> 5);     // B
```

### Brightness Step Calculation

Based on `local_22` (from frame data via `FUN_02063eb4`):

| local_22 Value | brightness_current | brightness_step | Behavior |
|----------------|--------------------|-----------------|----------|
| 0 | (unchanged) | (unchanged) | No palette fade |
| 99 | DAT_02064660 (max) | 0 | Full brightness, static |
| > 0 (not 99) | DAT_02064660 / value | same | Fade in (step = start) |
| < 0 | DAT_02064660 (max) | DAT_02064660 / value | Fade out (negative step) |

## Effect Lifecycle Integration

### Tick Path

```
Main Loop (FUN_0234c1d8)
    → FUN_022bf764 (tick all 32 effect slots)
        → FUN_022bf4f0 (per-effect tick)
            → if anim_type == 5 or 6:
                FUN_022bdf34 (screen effect wrapper)
                    → FUN_020642a8 (playback engine)
```

### Completion Detection

`FUN_022bdf34` calls `FUN_020642a8` each tick. When it returns 0 (playback complete):

1. `FUN_022bdec4(param_1, 1)` — marks effect slot free (`instance_id = -1`)
2. Clears `screen_effect_param` at `base + 0x279E + param_from_caller`
3. If `shared_wan_state == 0`: adds 13-frame delay via `FUN_022ea428(0xd)`
4. Cleanup resource via `FUN_022c07d0`

**Evidence:** `FUN_022bdf34`
```c
iVar2 = FUN_020642a8((short *)(param_1 + 0x3a));
if (iVar2 == 0) {
    FUN_022bdec4((int)param_1, 1);
    *(undefined *)(*DAT_022bdfbc + *param_1 + 0x279e) = 0;
    if (*(int *)(*piVar1 + 0x2784) == 0) {
        FUN_022ea428(0xd);  // 13 frame delay
    }
    puVar3 = (undefined4 *)FUN_022c07d0(2, *param_1);
    puVar3[1] = 0;
    *puVar3 = 0xffffffff;
}
```

### Completion Conditions in FUN_020642a8

| Mode | Returns 0 When |
|------|----------------|
| Simple | `simple_mode_ptr == 0` (no data) |
| Complex, non-looping | `duration_countdown == 0` AND frame advance fails (past last frame) |
| Complex, looping | Only via `cleanup_pending_flag` set by `FUN_0206423c` |

## Command Processing Chain

```
1. FUN_022c0068    Build command packet (dispatch_type, handle, screen_select)
2. FUN_022c0000    Queue into 0x84-byte pool (find free slot, copy 0x20 bytes)
3. FUN_022c039c    Process queue (called from main loop, iterates all slots)
4. FUN_022c0304    Dispatch command (switch on dispatch_type, one-shot)
5. FUN_02064014    Start playback (copy SIR0 group data into runtime state)
6. FUN_020642a8    Per-frame tick (called from effect lifecycle via FUN_022bdf34)
```

## Open Questions

- Complete SIR0 group descriptor structure (0x20 bytes) — only loop control fields at +0x08/+0x09/+0x0A confirmed
- Frame entry unknown fields: +0x00, +0x02, +0x0A, +0x14–0x1A, +0x20–0x22
- `DAT_020648f8` value (RLE run length mask — likely 0x0FFF)
- `DAT_02064660` value (max brightness constant — likely 0x2000 or similar fixed point)
- Type 6 (WBA) playback — case 6 in `FUN_022c0304` is empty/fallthrough, may use different init path
- Purpose of `FUN_022c0280` (type 6 init, called in `FUN_022be44c` but not yet decompiled)
- How `screen_effect_param` (from `effect_animation_info + 0x18`) affects rendering beyond storage at `base + 0x279E/0x279F`
- `FUN_02063f30` behavior (simple mode frame renderer)
- Where `FUN_022c039c` is called from in the main loop

### Where to Investigate

| Question | Function/Location |
|----------|-------------------|
| SIR0 group structure | Binary inspection of `effect.bin` files at index 268+ |
| Frame entry unknowns | `FUN_0206478c` (already decompiled), binary inspection |
| Type 6 (WBA) system | `FUN_022c0280` at `0x022c0280` |
| Simple mode rendering | `FUN_02063f30` |
| RLE mask value | `DAT_020648f8` in Ghidra (read data directly) |
| Max brightness | `DAT_02064660` in Ghidra (read data directly) |
| Command queue caller | Cross-refs to `FUN_022c039c` |

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_022bdf34` | `0x022bdf34` | Screen effect tick wrapper |
| `FUN_020642a8` | `0x020642a8` | Screen effect playback engine (main tick) |
| `FUN_022c01fc` | `0x022c01fc` | Type 5 init dispatcher |
| `FUN_022c0280` | `0x022c0280` | Type 6 init dispatcher (not yet decompiled) |
| `FUN_022c0068` | `0x022c0068` | Build command packet |
| `FUN_022c0000` | `0x022c0000` | Queue command into pool |
| `FUN_022c039c` | `0x022c039c` | Process command queue (main loop) |
| `FUN_022c0304` | `0x022c0304` | Dispatch individual command |
| `FUN_02064014` | `0x02064014` | Start playback (copy SIR0 data to state) |
| `FUN_020646f0` | `0x020646f0` | Bulk copy animation groups to runtime |
| `FUN_0206478c` | `0x0206478c` | Tilemap writer (RLE decompression + BG write) |
| `FUN_0206466c` | `0x0206466c` | Tilemap clear (zero full BG layer) |
| `FUN_02064a7c` | `0x02064a7c` | Post-load SIR0 translation |
| `FUN_02064c60` | `0x02064c60` | Frame duration lookup |
| `FUN_02063ee0` | `0x02063ee0` | Frame data loader (complex mode) |
| `FUN_02063f78` | `0x02063f78` | Frame advance (simple mode) |
| `FUN_02063f30` | `0x02063f30` | Frame renderer (simple mode) |
| `FUN_0206409c` | `0x0206409c` | Resource cache slot accessor |
| `FUN_022c07d0` | `0x022c07d0` | Screen effect resource cleanup |
| `FUN_022c0450` | `0x022c0450` | Resource loader switch (types 5/6) |
| `FUN_022bff30` | `0x022bff30` | Command queue pool allocator (0x84 bytes) |
| `FUN_022bffa4` | `0x022bffa4` | Command queue reset |
| `FUN_022bd744` | `0x022bd744` | Screen layout check (returns 1 for main screen) |
| `FUN_020094c4` | `0x020094c4` | BLDALPHA shadow register write |
| `FUN_0200c020` | `0x0200c020` | Palette brightness scaling |
| `FUN_0200b3fc` | `0x0200b3fc` | BG tilemap entry write |
| `FUN_0200b330` | `0x0200b330` | Tilemap VRAM flush |
| `FUN_02063bcc` | `0x02063bcc` | Blend mode set (0=off, 2=blend, 3=special) |
| `FUN_02063e68` | `0x02063e68` | Get screen ID for blend registers |
| `FUN_02063e7c` | `0x02063e7c` | Get palette write target |
| `FUN_0200a504` | `0x0200a504` | Palette VRAM flush |
| `FUN_0206423c` | `0x0206423c` | Looping effect termination handler |
| `HandleSir0TranslationVeneer` | - | SIR0 pointer translation |
| `FUN_02001ab0` | `0x02001ab0` | Fixed-point interpolation factor |
| `MultiplyByFixedPoint` | - | Fixed-point multiply for blend lerp |
