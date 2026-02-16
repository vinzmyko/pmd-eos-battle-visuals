# Dungeon Bin — Shadows, Ripples & Shared Palette

## Overview

Files 995–997 in `DUNGEON/dungeon.bin` contain shadow sprites, water ripple animations, and their shared palette. These are global assets (not per-tileset).

---

## File Index

| Index | Container | Description |
|-------|-----------|-------------|
| 995 | Raw (u32 header + data) | Shadow texture — 50 tiles, 4bpp |
| 996 | SIR0 | Water ripple animation — 3 frames, 8bpp |
| 997 | Raw | Shared RGBX palette — 2 × 16 colors |

---

## File 997 — Shared Palette

Format: RGBX (4 bytes per color, 4th byte is padding).
```
[0x00] Palette 0: 16 colors × 4 bytes = 64 bytes
[0x40] Palette 1: 16 colors × 4 bytes = 64 bytes
```

Total: 128 bytes. Only Palette 0 is used by shadows and ripples. Index 0 is transparent.

### Palette 0 — Used Colors

| Index | RGB | Used By |
|-------|-----|---------|
| 0 | (0, 127, 151) | Transparent (alpha = 0) |
| 1 | (0, 0, 0) | Enemy shadow (black) |
| 5 | (167, 111, 0) | Ally shadow dark |
| 6 | (223, 183, 0) | Ally shadow medium |
| 7 | (255, 247, 0) | Ally shadow bright / ripple yellow circle |
| 10 | (159, 231, 255) | Ripple cyan |
| 15 | (255, 255, 255) | Ripple white highlight |

---

## File 995 — Shadow Texture

### Structure
```
[0x00] u32 LE  tile_count = 50
[0x04] data    tile_count × 32 bytes (4bpp tiles)
```

Total: 1604 bytes. No container wrapping.

### 4bpp Tile Format

Each tile is 8×8 pixels, 32 bytes:
```
for each byte in tile:
    left_pixel  = byte & 0x0F        // low nibble
    right_pixel = (byte >> 4) & 0x0F  // high nibble

Layout: 4 bytes per row × 8 rows = 32 bytes
```

### Shadow Sprites

| Shadow | Size (px) | Tiles (W×H) | Tile Indices |
|--------|-----------|-------------|--------------|
| Enemy Small | 8×8 | 1×1 | [0] |
| Enemy Medium | 16×8 | 2×1 | [1, 2] |
| Enemy Large | 32×8 | 4×1 | [3, 4, 5, 6] |
| Ally Small | 8×8 | 1×1 | [7] |
| Ally Medium | 16×8 | 2×1 | [8, 9] |
| Ally Large | 32×16 | 4×2 | [10–17] |

3 sizes only (no XL). Tiles assemble left-to-right, top-to-bottom. Enemy shadows are solid black ellipses (index 1). Ally shadows are yellow gradient ellipses (indices 5, 6, 7).

### Shadow Size Per Monster (Ghidra)

Each monster's shadow size is stored in `monster.md` at offset `0x2E` within each `0x44`-byte entry:
```
shadow_size = read_byte(monster_id * 0x44 + monster_table_base + 0x2E)
```

Values range 0–35, mapping to the 3 sprite sizes (small/medium/large).

### Shadow Type Selection (Ghidra: `DetermineMonsterShadow`)

| Condition | Shadow Type |
|-----------|-------------|
| Normal floor tile | Enemy shadow |
| Floor tile + `IsWaterTileset()`, or secondary terrain = water | Ally shadow |
| Tile type 3 (chasm) | No shadow |

Water shadow index lookup: `DAT_02304c30[land_shadow_size]`

### Shadow Rendering (Ghidra: `FUN_02058afc`)

See `Systems/positioning_system.md` for full details. Key points:

- Shadow position = entity position + animation offset + table X offset
- Hardcoded Y adjustment of −4 pixels
- OAM attributes at `DAT_02058c0c` (0x10 bytes per size)
- X offset table at `DAT_02058c10` (4 bytes per size)

---

## File 996 — Water Ripple Animation

### Structure

SIR0-wrapped. After unwrapping, content is **2304 bytes (0x900)**.

Interleaved frames — each frame contains enemy then ally data:
```
for each of 3 frames:
    0x100 bytes = enemy ripple (4 tiles, 32×8 px)
    0x200 bytes = ally ripple  (8 tiles, 32×16 px)

Total: (0x100 + 0x200) × 3 = 0x900
```

### Frame Offsets

| Offset | Size | Content |
|--------|------|---------|
| 0x000 | 0x100 | Frame 0 — Enemy |
| 0x100 | 0x200 | Frame 0 — Ally |
| 0x300 | 0x100 | Frame 1 — Enemy |
| 0x400 | 0x200 | Frame 1 — Ally |
| 0x600 | 0x100 | Frame 2 — Enemy |
| 0x700 | 0x200 | Frame 2 — Ally |

### 8bpp Tile Format

Each tile is 8×8 pixels, 64 bytes (1 byte per pixel):
```
for each byte in tile:
    palette_index = (byte == 0) ? 0 : (byte & 0x0F)
```

Only 4 unique byte values exist: `0x00`, `0xB7`, `0xBA`, `0xBF`

### Ripple Sprites

| Type | Size (px) | Tiles (W×H) | Bytes/Frame |
|------|-----------|-------------|-------------|
| Enemy | 32×8 | 4×1 | 256 |
| Ally | 32×16 | 4×2 | 512 |

Enemy ripple: cyan/white concentric rings. Ally ripple: same rings plus yellow circle border (baked into sprite data, not a palette effect).

Animation: 3 frames showing rings rippling outward, ~150ms per frame.

---

## Key Format Differences

| Aspect | Shadows (995) | Ripples (996) |
|--------|---------------|---------------|
| Bit depth | 4bpp | 8bpp |
| Bytes per tile | 32 | 64 |
| Animation | Static | 3 frames |
| Container | Raw + u32 header | SIR0 |
| Pixel decoding | Split byte into nibbles | `byte & 0x0F` |

---

## ROM References

| Component | Location | Notes |
|-----------|----------|-------|
| Shadow texture | dungeon.bin[995] | Raw 4bpp + u32 tile count header |
| Ripple animation | dungeon.bin[996] | SIR0 wrapped, 8bpp |
| Shared palette | dungeon.bin[997] | RGBX, 2 × 16 colors |
| Monster shadow size | monster.md + 0x2E | Per-monster, 0x44-byte stride |
| Shadow OAM attrs | DAT_02058c0c | 0x10 bytes per size |
| Shadow X offsets | DAT_02058c10 | 4 bytes per size |
| Water shadow remap | DAT_02304c30 | land_size → water_size |
| Shadow renderer | FUN_02058afc | See positioning_system.md |
| Shadow type logic | DetermineMonsterShadow | Terrain-based selection |
| Monster shadow lookup | GetShadowSize | Reads monster.md offset 0x2E |
