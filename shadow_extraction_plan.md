# Shadow Sprite Extraction Plan

## Overview

Pokemon shadows in PMD:EoS are stored in `DUNGEON/dungeon.bin` and rendered using the DS OAM (Object Attribute Memory) hardware sprite system. This document outlines the plan for extracting shadow sprites from the ROM.

## File Locations

From `pmd2dungeondata.xml`:

| File Index | Type | Description |
|------------|------|-------------|
| 995 | `DBIN_RAW_IMAGE_4BPP` | Shadow texture (raw 4bpp tiles) |
| 996 | `SIR0` | Unknown (possibly OAM metadata or tile arrangement) |
| 997 | `DPL` | Shadow palette (16 colors) |

## Shadow System Analysis (from Ghidra)

### Shadow Size Per Monster

Each monster has a shadow size stored in `monster.md`:

```c
int GetShadowSize(monster_id monster_id)
{
    // Offset 0x2E within each monster's 0x44-byte entry
    return (uint)*(byte *)((short)monster_id * 0x44 + *DAT_02052854 + 0x2e);
}
```

**Monster entry size:** 0x44 bytes (68 bytes)
**Shadow size offset:** 0x2E within entry

### Shadow Types

From `DetermineMonsterShadow`:

1. **Land Shadow** - Default shadow on normal floor tiles
2. **Water Shadow** - Used when:
   - Tile type == 1 (floor) AND `IsWaterTileset()` returns true
   - Tile type == 2 (secondary terrain) AND secondary terrain is water
3. **No Shadow** - When tile type == 3 (chasm)

Water shadow index is looked up via: `DAT_02304c30[land_shadow_size]`

### OAM Rendering

From `FUN_02058afc`:

```c
// OAM attribute table base
puVar8 = (ushort *)(DAT_02058c0c + iVar6 * 0x10 + (uint)(param_6 != '\0') * 8);
```

**Structure:**
- `DAT_02058c0c` = Base of shadow OAM table
- Each shadow size = 0x10 bytes (16 bytes)
- Land variant at offset +0x00
- Water variant at offset +0x08
- 4 OAM attributes × 2 bytes each = 8 bytes per variant

**X Offset per size:**
```c
// DAT_02058c10 = X offset table, 4 bytes per shadow size
(short)*(undefined4 *)(DAT_02058c10 + iVar6 * 4)
```

**Y Offset:**
```c
// Fixed -4 adjustment for all shadows
puVar8[3] = puVar8[3] | (sVar3 + sVar4 + -4) * 0x10;
```

## Extraction Pipeline

### Step 1: Parse dungeon.bin (BinPack)

```python
# BinPack format:
# Offset 0x00: u32 = 0 (always)
# Offset 0x04: u32 = number of files
# Offset 0x08: TOC entries (8 bytes each)
#   - u32 pointer (file offset)
#   - u32 length (file size)

def parse_binpack(data: bytes) -> list[bytes]:
    num_files = int.from_bytes(data[4:8], 'little')
    files = []
    for i in range(num_files):
        toc_offset = 8 + i * 8
        pointer = int.from_bytes(data[toc_offset:toc_offset+4], 'little')
        length = int.from_bytes(data[toc_offset+4:toc_offset+8], 'little')
        files.append(data[pointer:pointer+length])
    return files
```

### Step 2: Extract Raw Files

```python
dungeon_bin = open("DUNGEON/dungeon.bin", "rb").read()
files = parse_binpack(dungeon_bin)

shadow_texture = files[995]  # Raw 4bpp image
shadow_metadata = files[996]  # SIR0 (investigate)
shadow_palette = files[997]   # DPL palette
```

### Step 3: Parse DPL Palette (File 997)

DPL is a simple palette format:

```python
def parse_dpl(data: bytes) -> list[tuple[int, int, int, int]]:
    """Parse DPL palette to list of RGBA tuples."""
    colors = []
    for i in range(0, len(data), 2):
        color16 = int.from_bytes(data[i:i+2], 'little')
        # DS uses BGR555 format
        r = (color16 & 0x1F) << 3
        g = ((color16 >> 5) & 0x1F) << 3
        b = ((color16 >> 10) & 0x1F) << 3
        a = 255 if i > 0 else 0  # First color is transparent
        colors.append((r, g, b, a))
    return colors
```

### Step 4: Parse 4bpp Image (File 995)

Raw 4bpp tiles in DS format (8x8 pixels per tile):

```python
def parse_4bpp_tiles(data: bytes) -> list[list[int]]:
    """Parse 4bpp tile data to list of tiles (each tile = 64 pixel indices)."""
    tiles = []
    tile_size = 32  # 8x8 pixels, 4bpp = 32 bytes per tile
    
    for tile_offset in range(0, len(data), tile_size):
        tile_data = data[tile_offset:tile_offset + tile_size]
        pixels = []
        for byte in tile_data:
            # Each byte contains 2 pixels (low nibble first)
            pixels.append(byte & 0x0F)
            pixels.append((byte >> 4) & 0x0F)
        tiles.append(pixels)
    
    return tiles
```

### Step 5: Determine Shadow Layout

**Unknown - needs investigation:**

The OAM metadata in file 996 (SIR0) likely contains information about:
- How many shadow sizes exist
- Tile arrangement for each shadow size
- Dimensions of each shadow variant

**Hypothesis based on DS OAM sizes:**

| Shadow Size | Likely Dimensions | Tiles |
|-------------|------------------|-------|
| 0 (Small) | 16x8 | 2 tiles |
| 1 (Medium) | 24x16 or 32x8 | 4-6 tiles |
| 2 (Large) | 32x16 | 8 tiles |
| 3 (XL) | 48x24 or 64x32 | 12-16 tiles |

### Step 6: Render Shadow Sprites

```python
from PIL import Image

def render_shadow(tiles: list[list[int]], 
                  palette: list[tuple], 
                  width_tiles: int, 
                  height_tiles: int,
                  tile_indices: list[int]) -> Image:
    """Render a shadow sprite from tiles."""
    width = width_tiles * 8
    height = height_tiles * 8
    
    img = Image.new('RGBA', (width, height), (0, 0, 0, 0))
    
    for ty in range(height_tiles):
        for tx in range(width_tiles):
            tile_idx = tile_indices[ty * width_tiles + tx]
            tile = tiles[tile_idx]
            
            for py in range(8):
                for px in range(8):
                    pixel_idx = tile[py * 8 + px]
                    color = palette[pixel_idx]
                    img.putpixel((tx * 8 + px, ty * 8 + py), color)
    
    return img
```

## Investigation Tasks

### Task 1: Analyze File 996 (SIR0 Metadata)

1. Unwrap the SIR0 container
2. Dump the content and analyze structure
3. Look for:
   - Number of shadow variants
   - Tile indices per variant
   - Dimensions or OAM attribute data

### Task 2: Dump and Visualize File 995

1. Extract raw bytes
2. Calculate number of tiles: `len(data) / 32`
3. Render all tiles in a grid to visualize
4. Identify shadow shapes

### Task 3: Cross-Reference with ROM OAM Table

From Ghidra, dump the actual OAM data at `DAT_02058c0c`:
- Read 64+ bytes (4 shadow sizes × 16 bytes)
- Decode OAM attributes to understand tile indices

### Task 4: Verify Shadow Sizes in monster.md

Check shadow size values at offset 0x2E for various monsters:
- Small pokemon (Pichu, etc.)
- Medium pokemon (Pikachu, etc.)
- Large pokemon (Snorlax, etc.)

## Output Format

For the scraper, output shadow sprites as:

```
shadows/
├── shadow_0.png      # Small land shadow
├── shadow_1.png      # Medium land shadow  
├── shadow_2.png      # Large land shadow
├── shadow_3.png      # XL land shadow (if exists)
├── shadow_water_0.png  # Small water shadow
├── shadow_water_1.png  # Medium water shadow
├── shadow_water_2.png  # Large water shadow
├── shadow_water_3.png  # XL water shadow (if exists)
└── shadow_metadata.json
```

**shadow_metadata.json:**
```json
{
  "shadow_sizes": [
    {
      "id": 0,
      "name": "small",
      "width": 16,
      "height": 8,
      "x_offset": -8,
      "land_file": "shadow_0.png",
      "water_file": "shadow_water_0.png"
    },
    ...
  ]
}
```

## Client Usage

```gdscript
# In pokemon_entity.gd or shadow_renderer.gd

var shadow_sprites: Dictionary = {}  # size_id -> {land: Texture2D, water: Texture2D}
var shadow_metadata: Dictionary = {}

func _ready():
    _load_shadow_sprites()

func _load_shadow_sprites():
    var json = JSON.parse_string(FileAccess.open("res://assets/shadows/shadow_metadata.json", FileAccess.READ).get_as_text())
    shadow_metadata = json
    
    for size_data in json["shadow_sizes"]:
        var size_id = size_data["id"]
        shadow_sprites[size_id] = {
            "land": load("res://assets/shadows/" + size_data["land_file"]),
            "water": load("res://assets/shadows/" + size_data["water_file"]),
            "x_offset": size_data["x_offset"],
            "width": size_data["width"],
            "height": size_data["height"]
        }

func get_shadow_for_monster(monster_id: int, is_water_tile: bool) -> Dictionary:
    var shadow_size = _get_shadow_size_from_monster_data(monster_id)
    var shadow_data = shadow_sprites[shadow_size]
    
    return {
        "texture": shadow_data["water"] if is_water_tile else shadow_data["land"],
        "offset": Vector2(shadow_data["x_offset"], -4)
    }
```

## Related Files

- `DUNGEON/dungeon.bin` - Contains shadow assets (files 995-997)
- `BALANCE/monster.md` - Contains shadow size per monster (offset 0x2E)
- ROM addresses:
  - `DAT_02058c0c` - Shadow OAM attribute table
  - `DAT_02058c10` - Shadow X offset table
  - `DAT_02304c30` - Land-to-water shadow mapping

## Open Questions

1. **Exact number of shadow sizes?** (Likely 4, but needs verification)
2. **Water shadow differences?** (Different palette? Different shape? Both?)
3. **File 996 structure?** (OAM data? Tile arrangement? Both?)
4. **Are land/water shadows in same texture or separate regions?**

## References

- SkyTemple DPL handler: `skytemple_files/graphics/dpl/`
- SkyTemple BinPack handler: `skytemple_files/container/bin_pack/`
- Ghidra functions: `GetShadowSize`, `DetermineMonsterShadow`, `FUN_02058afc`
- pmd2dungeondata.xml for file type definitions
