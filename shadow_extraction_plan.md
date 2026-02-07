# Shadow Sprite Extraction Plan v2.0

## Overview

Pokemon shadows in PMD:EoS are stored in `DUNGEON/dungeon.bin` and rendered using the DS OAM hardware sprite system. This document describes the verified extraction process.

**Status:** Python implementation complete. All open questions resolved.

---

## File Locations

| File Index | Type | Description |
|------------|------|-------------|
| 995 | Raw 4bpp with header | Shadow texture (50 tiles) |
| 996 | SIR0 | Water ripple animation (see `water_ripples_plan.md`) |
| 997 | RGBX palette | Shared palette (shadows + ripples) |

**Correction from v1.0:** File 996 is water ripple data, NOT shadow OAM metadata.

---

## Data Format

### File 995 — Shadow Texture

```
Offset  Size    Description
0x00    4       Tile count (u32 LE) = 50
0x04    1600    Tile data (50 tiles × 32 bytes per tile)
```

Total size: 1604 bytes

#### 4bpp Tile Format

Each tile is 8×8 pixels, 4 bits per pixel (32 bytes per tile):

```
For each byte:
  Low nibble  (bits 0-3) = left pixel palette index
  High nibble (bits 4-7) = right pixel palette index

Row layout: 4 bytes per row × 8 rows = 32 bytes
```

### File 997 — Palette

Format: **RGBX** (4 bytes per color, X is padding). NOT BGR555.

```
Offset  Size    Description
0x00    64      Palette 0 (16 colors × 4 bytes) — used for shadows
0x40    64      Palette 1 (16 colors × 4 bytes) — unused for shadows
```

Total: 128 bytes. Index 0 is transparent.

#### Palette 0 Colors (verified)

| Index | RGB | Usage |
|-------|-----|-------|
| 0 | (0, 127, 151) | Transparent |
| 1 | (0, 0, 0) | Land shadow (black) |
| 5 | (167, 111, 0) | Water shadow dark |
| 6 | (223, 183, 0) | Water shadow medium |
| 7 | (255, 247, 0) | Water shadow bright |

---

## Shadow Sprites

### Tile Assignments (verified)

| Shadow Type | Size (px) | Tiles (W×H) | Tile Indices |
|-------------|-----------|-------------|--------------|
| Land Small | 8×8 | 1×1 | [0] |
| Land Medium | 16×8 | 2×1 | [1, 2] |
| Land Large | 32×8 | 4×1 | [3, 4, 5, 6] |
| Water Small | 8×8 | 1×1 | [7] |
| Water Medium | 16×8 | 2×1 | [8, 9] |
| Water Large | 32×16 | 4×2 | [10, 11, 12, 13, 14, 15, 16, 17] |

**3 shadow sizes** (small, medium, large). No XL size exists.

Tiles are assembled left-to-right, top-to-bottom.

### Visual Differences
- **Land shadows**: Solid black ellipses (palette index 1)
- **Water shadows**: Yellow/gold gradient ellipses (palette indices 5, 6, 7)

---

## Shadow Size Per Monster

Each monster has a shadow size stored in `monster.md`:

```c
// Offset 0x2E within each monster's 0x44-byte entry
int GetShadowSize(monster_id) {
    return *(byte *)((short)monster_id * 0x44 + *DAT_02052854 + 0x2e);
}
```

Shadow size values range 0-35. The exact mapping to the 3 sprite sizes is: small values → small, medium values → medium, large values → large. Water vs land is determined by terrain type, not shadow_size.

### Shadow Types (from Ghidra `DetermineMonsterShadow`)

1. **Land Shadow** — Default on normal floor
2. **Water Shadow** — When tile type == 1 (floor) AND `IsWaterTileset()`, or tile type == 2 (secondary) AND secondary is water
3. **No Shadow** — When tile type == 3 (chasm)

Water shadow index lookup: `DAT_02304c30[land_shadow_size]`

---

## SkyTemple Access

```python
from skytemple_files.common.types.file_types import FileType

dungeon_bin = FileType.DUNGEON_BIN.deserialize(
    rom.getFileByName("DUNGEON/dungeon.bin"), config
)

raw_995 = dungeon_bin._files[995]  # Shadow texture
raw_997 = dungeon_bin._files[997]  # Palette
```

### Parsing Palette

```python
palette_rgba = []
for i in range(0, len(raw_997), 4):
    r, g, b, x = raw_997[i], raw_997[i+1], raw_997[i+2], raw_997[i+3]
    a = 0 if i == 0 else 255
    palette_rgba.append((r, g, b, a))
```

### Parsing Tiles

```python
tile_count = int.from_bytes(raw_995[0:4], 'little')  # = 50
tile_data = raw_995[4:]
tile_size = 32  # bytes per 4bpp 8x8 tile

def get_tile(tile_idx, palette_rgba):
    tile_img = Image.new("RGBA", (8, 8), (0, 0, 0, 0))
    tile_offset = tile_idx * tile_size
    tile_bytes = tile_data[tile_offset:tile_offset + tile_size]
    
    for byte_idx, byte in enumerate(tile_bytes):
        px = (byte_idx % 4) * 2
        py = byte_idx // 4
        tile_img.putpixel((px, py), palette_rgba[byte & 0x0F])
        tile_img.putpixel((px + 1, py), palette_rgba[(byte >> 4) & 0x0F])
    
    return tile_img
```

### Assembling Sprites

```python
def assemble_sprite(tile_indices, cols, rows, palette):
    width, height = cols * 8, rows * 8
    sprite = Image.new("RGBA", (width, height), (0, 0, 0, 0))
    
    for i, tile_idx in enumerate(tile_indices):
        tile_x = (i % cols) * 8
        tile_y = (i // cols) * 8
        sprite.paste(get_tile(tile_idx, palette), (tile_x, tile_y))
    
    return sprite
```

---

## Output Format

```
shadows/
├── shadow_sheet.png           # Combined sheet (64×64)
├── shadow_land_small.png      # 8×8
├── shadow_land_medium.png     # 16×8
├── shadow_land_large.png      # 32×8
├── shadow_water_small.png     # 8×8
├── shadow_water_medium.png    # 16×8
└── shadow_water_large.png     # 32×16
```

### Sprite Sheet Layout

```
64×64 pixel sheet:
Row 0 (y=0):    Land Small (8×8)
Row 1 (y=8):    Land Medium (16×8)
Row 2 (y=16):   Land Large (32×8)
Row 3 (y=24):   [empty]
Row 4 (y=32):   Water Small (8×8)
Row 5 (y=40):   Water Medium (16×8)
Row 6-7 (y=48): Water Large (32×16)
```

---

## Rust Implementation Notes

```rust
struct ShadowData {
    tile_count: u32,
    tiles: Vec<[u8; 32]>,
}

fn parse_shadow_texture(data: &[u8]) -> ShadowData {
    let tile_count = u32::from_le_bytes(data[0..4].try_into().unwrap());
    let mut tiles = Vec::new();
    for i in 0..tile_count as usize {
        let offset = 4 + i * 32;
        let mut tile = [0u8; 32];
        tile.copy_from_slice(&data[offset..offset + 32]);
        tiles.push(tile);
    }
    ShadowData { tile_count, tiles }
}

fn parse_palette(data: &[u8]) -> Vec<Color> {
    data.chunks(4)
        .enumerate()
        .map(|(i, c)| Color { r: c[0], g: c[1], b: c[2], a: if i == 0 { 0 } else { 255 } })
        .collect()
}
```

---

## ROM References

| Component | Location | Notes |
|-----------|----------|-------|
| Shadow texture | dungeon.bin[995] | Raw 4bpp with u32 tile count header |
| Shadow palette | dungeon.bin[997] | RGBX format, shared with ripples |
| Shadow size/monster | monster.md + 0x2E | Byte value within 0x44-byte entries |
| OAM attributes | DAT_02058c0c | 0x10 bytes per size |
| X offsets | DAT_02058c10 | 4 bytes per size |
| Water remap | DAT_02304c30 | land_size → water_size |

---

## Testing Checklist

- [x] Verify tile count header = 50
- [x] Land shadows render as solid black
- [x] Water shadows render with yellow gradient
- [x] Transparency works (index 0)
- [x] Large water shadow is 32×16 (not 32×8)
- [x] Tiles assemble left-to-right, top-to-bottom
- [x] Only 3 shadow sizes exist (no XL)
- [x] File 996 is ripples, not shadow OAM
- [x] Palette is RGBX 4-byte, not BGR555 2-byte
