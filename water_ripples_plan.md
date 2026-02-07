# Water Ripple Extraction Plan v1.0

## Overview

Water ripples are animated overlays displayed beneath characters when standing on water terrain in PMD:EoS. There are separate variants for enemy and ally characters. The ally variant includes a yellow circle border baked into the sprite data.

**Status:** Python implementation complete.

---

## File Locations

| File Index | Type | Description |
|------------|------|-------------|
| 996 | SIR0 | Water ripple animation data |
| 997 | RGBX palette | Shared palette (also used by shadows) |

---

## Data Format

### File 996 — Ripple Animation (SIR0)

After SIR0 unwrapping, content is **2304 bytes (0x900)**.

```
For each of 3 animation frames:
  0x100 bytes = Enemy ripple frame (4 tiles, 8bpp)
  0x200 bytes = Ally ripple frame (8 tiles, 8bpp)

Total: (0x100 + 0x200) × 3 = 0x900
```

#### Frame Layout

```
Offset    Size    Description
0x000     0x100   Frame 0 — Enemy
0x100     0x200   Frame 0 — Ally
0x300     0x100   Frame 1 — Enemy
0x400     0x200   Frame 1 — Ally
0x600     0x100   Frame 2 — Enemy
0x700     0x200   Frame 2 — Ally
```

### Tile Format (8bpp)

Unlike shadows (4bpp), ripples use **8 bits per pixel** (1 byte = 1 pixel, 64 bytes per 8×8 tile).

Only 4 unique byte values exist in the data: `0x00`, `0xB7`, `0xBA`, `0xBF`

Palette index decoding:
```python
idx = 0 if byte == 0 else (byte & 0x0F)
```

### File 997 — Palette

Same file as shadows. RGBX format, 4 bytes per color. Only Palette 0 used.

#### Relevant Colors

| Index | RGB | Usage |
|-------|-----|-------|
| 0 | (0, 127, 151) | Transparent |
| 7 | (255, 247, 0) | Yellow (ally circle) |
| 10 | (159, 231, 255) | Light cyan (ripple) |
| 15 | (255, 255, 255) | White (ripple highlight) |

---

## Ripple Sprites

### Specifications

| Type | Size (px) | Tiles (W×H) | Bytes/Frame |
|------|-----------|-------------|-------------|
| Enemy | 32×8 | 4×1 | 256 (0x100) |
| Ally | 32×16 | 4×2 | 512 (0x200) |

### Visual Differences
- **Enemy ripple**: Cyan/white concentric rings only (32×8)
- **Ally ripple**: Same rings + yellow circle border (32×16, taller)

The yellow circle is baked into the ally sprite data, not a palette effect.

### Animation
- 3 frames per type
- Each frame shows ripple rings at slightly different positions (rippling outward effect)
- Typical playback: ~150ms per frame

---

## SkyTemple Access

```python
from skytemple_files.common.types.file_types import FileType

dungeon_bin = FileType.DUNGEON_BIN.deserialize(
    rom.getFileByName("DUNGEON/dungeon.bin"), config
)

raw_996 = dungeon_bin._files[996]
raw_997 = dungeon_bin._files[997]

# Unwrap SIR0
sir0 = FileType.SIR0.deserialize(raw_996)
content = sir0.content  # 2304 bytes
```

### Parsing Palette

```python
dpl_997 = FileType.DPL.deserialize(raw_997)

def dpl_to_rgba(pal_data):
    colors = []
    for c in range(len(pal_data) // 3):
        r, g, b = pal_data[c*3], pal_data[c*3+1], pal_data[c*3+2]
        a = 0 if c == 0 else 255
        colors.append((r, g, b, a))
    while len(colors) < 16:
        colors.append((0, 0, 0, 255))
    return colors

palette = dpl_to_rgba(dpl_997.palettes[0])
```

### Rendering

```python
TILE_SIZE = 64  # 8x8 pixels, 1 byte per pixel

def render_ripple(data, tiles_w, tiles_h, palette):
    width, height = tiles_w * 8, tiles_h * 8
    img = Image.new("RGBA", (width, height), (0, 0, 0, 0))
    
    for tile_idx in range(tiles_w * tiles_h):
        tile_x = (tile_idx % tiles_w) * 8
        tile_y = (tile_idx // tiles_w) * 8
        tile_start = tile_idx * TILE_SIZE
        tile_data = data[tile_start:tile_start + TILE_SIZE]
        
        for pixel_idx, byte in enumerate(tile_data):
            px, py = pixel_idx % 8, pixel_idx // 8
            idx = 0 if byte == 0 else (byte & 0x0F)
            img.putpixel((tile_x + px, tile_y + py), palette[idx])
    
    return img
```

### Extracting All Frames

```python
ENEMY_SIZE = 0x100
ALLY_SIZE = 0x200
NUM_FRAMES = 3

for frame_idx in range(NUM_FRAMES):
    frame_start = frame_idx * (ENEMY_SIZE + ALLY_SIZE)
    enemy_data = content[frame_start:frame_start + ENEMY_SIZE]
    ally_data = content[frame_start + ENEMY_SIZE:frame_start + ENEMY_SIZE + ALLY_SIZE]
    
    enemy_img = render_ripple(enemy_data, 4, 1, palette)  # 32×8
    ally_img = render_ripple(ally_data, 4, 2, palette)     # 32×16
```

---

## Output Format

```
ripples/
├── sheet_enemy_ripple.png    # 96×8 (3 frames horizontal)
├── sheet_ally_ripple.png     # 96×16 (3 frames horizontal)
├── sheet_water_ripple.png    # 96×24 (combined)
├── enemy_ripple.gif          # Animated preview
├── ally_ripple.gif           # Animated preview
└── frames/
    ├── enemy_frame0.png
    ├── enemy_frame1.png
    ├── enemy_frame2.png
    ├── ally_frame0.png
    ├── ally_frame1.png
    └── ally_frame2.png
```

### Sprite Sheet Layout

```
Enemy sheet (96×8):
  [Frame0 32×8][Frame1 32×8][Frame2 32×8]

Ally sheet (96×16):
  [Frame0 32×16][Frame1 32×16][Frame2 32×16]

Combined sheet (96×24):
  [Enemy frames]  ← y=0, height=8
  [Ally frames]   ← y=8, height=16
```

---

## Rust Implementation Notes

### SIR0 Unwrapping

```rust
fn unwrap_sir0(data: &[u8]) -> &[u8] {
    let content_ptr = u32::from_le_bytes(data[4..8].try_into().unwrap()) as usize;
    let pointer_list_ptr = u32::from_le_bytes(data[8..12].try_into().unwrap()) as usize;
    &data[content_ptr..pointer_list_ptr]
}
```

### Frame Extraction

```rust
const ENEMY_SIZE: usize = 0x100;
const ALLY_SIZE: usize = 0x200;
const NUM_FRAMES: usize = 3;

fn extract_frames(content: &[u8]) -> (Vec<&[u8]>, Vec<&[u8]>) {
    let mut enemy = Vec::new();
    let mut ally = Vec::new();
    for frame in 0..NUM_FRAMES {
        let offset = frame * (ENEMY_SIZE + ALLY_SIZE);
        enemy.push(&content[offset..offset + ENEMY_SIZE]);
        ally.push(&content[offset + ENEMY_SIZE..offset + ENEMY_SIZE + ALLY_SIZE]);
    }
    (enemy, ally)
}
```

### Pixel Decoding

```rust
fn decode_pixel(byte: u8) -> u8 {
    if byte == 0 { 0 } else { byte & 0x0F }
}
```

---

## Key Differences from Shadows

| Aspect | Shadows (995) | Ripples (996) |
|--------|---------------|---------------|
| Bit depth | 4bpp | 8bpp |
| Bytes per tile | 32 | 64 |
| Animation | Static | 3 frames |
| Container | Raw with u32 header | SIR0 wrapped |
| Pixel encoding | Direct nibble split | `byte & 0x0F` |

---

## Relationship to Other Systems

| System | Relationship |
|--------|--------------|
| Shadows (995) | Shares palette file 997 |
| Dungeon terrain | Ripples only show on water tiles |
| Character position | Rendered at character feet |
| Ally/enemy status | Determines which ripple variant |

---

## Testing Checklist

- [x] Verify SIR0 content size = 2304 bytes (0x900)
- [x] Enemy ripple renders as 32×8 cyan/white rings
- [x] Ally ripple renders as 32×16 with yellow circle
- [x] All 3 frames are visually distinct (rings move)
- [x] Transparency works (byte 0x00 → invisible)
- [x] Only 4 byte values exist: 0x00, 0xB7, 0xBA, 0xBF
- [x] Animation loops smoothly
