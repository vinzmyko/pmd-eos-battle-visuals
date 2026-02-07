# Dungeon Tileset Extraction Specification v2.0

## 1. Overview

This specification defines the export format for dungeon tilesets from Pokemon Mystery Dungeon: Explorers of Sky. The format is designed to be:

- **Visually faithful** to the original game
- **Engine-agnostic** but optimized for Godot 4.x
- **Easy to use** with standard image formats and JSON
- **Complete** with autotile rules for procedural dungeon generation

**Status:** Python implementation complete. This document is the canonical reference, superseding earlier research notes.

---

## 2. File Locations in ROM

| File Path | Description |
|-----------|-------------|
| `DUNGEON/dungeon.bin` | Container for all dungeon tileset data |
| `BALANCE/mappa_s.bin` | Floor layouts, maps dungeons to tilesets |
| `ARM9` binary | Contains hardcoded dungeon list |

### 2.1 SkyTemple Access Pattern

```python
from skytemple_files.common.types.file_types import FileType

dungeon_bin = FileType.DUNGEON_BIN.deserialize(
    rom.getFileByName("DUNGEON/dungeon.bin"), config
)

# Access per-tileset files
dma  = dungeon_bin.get(f"dungeon{tileset_id}.dma")
dpc  = dungeon_bin.get(f"dungeon{tileset_id}.dpc")
dpci = dungeon_bin.get(f"dungeon{tileset_id}.dpci")
dpl  = dungeon_bin.get(f"dungeon{tileset_id}.dpl")
dpla = dungeon_bin.get(f"dungeon{tileset_id}.dpla")
```

---

## 3. Graphics Data Classes

### 3.1 DMA (Dungeon Map Assembly)

Maps neighbor configurations to chunk IDs. This is the autotile logic.

```python
from skytemple_files.graphics.dma.protocol import DmaType

DmaType.WALL  = 0    # Wall terrain
DmaType.WATER = 1    # Water/lava (secondary terrain)
DmaType.FLOOR = 2    # Floor terrain

# Get chunk IDs for a terrain type + neighbor configuration
chunks = dma.get(DmaType.WALL, neighbor_bits)  # Returns list of 3 variations
chunk_id = chunks[0]  # Use first variation
```

### 3.2 DPC (Dungeon Palette Chunks)

Contains the actual chunk graphics (24×24 pixel tiles).

```python
# Render all chunks as a single image (20 columns wide)
base_chunks = dpc.chunks_to_pil(dpci, dpl.palettes, width_in_mtiles=20)
```

**Important:** Returns a `PIL.Image` in mode `'P'` (indexed/paletted), not RGBA. This preserves palette indices for shader-based animation.

### 3.3 DPCI (Dungeon Palette Chunk Index)

Index data for DPC. Always passed alongside DPC.

### 3.4 DPL (Dungeon Palette List)

Contains the base palettes (12 palettes × 16 colors each).

```python
num_palettes = len(dpl.palettes)  # Usually 12
palette_10 = dpl.palettes[10]     # List of RGB values [R,G,B,R,G,B,...]
# Each palette has 16 colors = 48 bytes
r, g, b = palette[0], palette[1], palette[2]  # First color
```

### 3.5 DPLA (Dungeon Palette List Animation)

Contains palette animation data. See §6 for full details.

```python
dpla.colors                        # List of 32 slots (16 for pal10, 16 for pal11)
dpla.durations_per_frame_for_colors  # List of 32 durations

# Get animated palette for a specific frame
palette = dpla.get_palette_for_frame(0, frame_index)  # Palette 10
palette = dpla.get_palette_for_frame(1, frame_index)  # Palette 11
```

---

## 4. Dungeon Metadata

### 4.1 Dungeon List (ARM9)

```python
from skytemple_files.hardcoded.dungeons import HardcodedDungeons

dungeons = HardcodedDungeons.get_dungeon_list(
    get_binary_from_rom(rom, config.bin_sections.arm9), config
)

# Each dungeon has:
dungeon.mappa_index   # Index into mappa floor lists
dungeon.start_after   # Starting floor offset
dungeon.number_floors # Number of floors
```

### 4.2 MAPPA (Floor Layouts)

```python
mappa = FileType.MAPPA_BIN.deserialize(rom.getFileByName("BALANCE/mappa_s.bin"))

# Get tileset ID for a dungeon's first floor
floor = mappa.floor_lists[dungeon.mappa_index][dungeon.start_after]
tileset_id = floor.layout.tileset_id
```

---

## 5. Neighbor Bit System

### 5.1 Bit Layout

The DMA uses an 8-bit value where each bit represents a neighbor:

```
    NW(32)  N(16)  NE(8)
    W(64)    X     E(4)
    SW(128) S(1)   SE(2)

Binary: SW W NW N NE E SE S
        7  6  5 4  3 2  1 0
```

Examples:
- `0b11111111` (255) = All 8 neighbors = fully surrounded
- `0b00000000` (0) = No neighbors = isolated tile
- `0b00010001` (17) = N + S = vertical corridor

### 5.2 The 47-Tile Blob Pattern

The game uses 256 possible neighbor configurations but only ~47 visually distinct tiles. This is because **diagonals only matter when both adjacent cardinals are present**.

Example — these all look the same (single south connection):
- `0b00000001` (S only)
- `0b00000011` (S + SE)
- `0b10000001` (S + SW)
- `0b10000011` (S + SE + SW)

---

## 6. Palette Animation System

### 6.1 Per-Color Animation

Unlike typical palette cycling, PMD uses **per-color animation** where:
- Each of the 16 colors in palette 10 has its own animation sequence
- Each color has its own duration (speed)
- This creates complex effects like shimmering water + moving highlights

### 6.2 Data Structure

```
DPLA.colors[0-15]   → Animation frames for palette 10, colors 0-15
DPLA.colors[16-31]  → Animation frames for palette 11, colors 0-15 (rarely used)

Each color slot contains: [R,G,B, R,G,B, R,G,B, ...]
                          frame0  frame1  frame2  ...
```

### 6.3 Duration System

```python
dpla.durations_per_frame_for_colors[0]   # Duration for color 0 (in 60fps frames)
# Convert to milliseconds
duration_ms = round(1000 / 60 * duration_frames)
# Example: duration=4 → 67ms, duration=18 → 300ms
```

### 6.4 Animation Effects

The combination of different speeds creates:
1. **Slow shimmer** (colors 1-4, duration=18): Gradual RGB shifts in water body
2. **Fast highlights** (colors 13-15, duration=4): Bright spots that appear to move

### 6.5 Which Palettes Animate

| Palette Index | Pixel Index Range | Used For | Actually Animates |
|---------------|-------------------|----------|-------------------|
| 10 | 160-175 | Water/Lava | Yes (always check) |
| 11 | 176-191 | Secondary | Rarely (often empty) |

### 6.6 Static vs Animated Detection

Having DPLA data doesn't mean animation exists. Must check if colors actually change:

```python
def check_animation(dpla):
    if not dpla.colors[0]:
        return False
    for color_idx in range(16):
        color_data = dpla.colors[color_idx]
        if color_data and len(color_data) >= 6:
            if color_data[0:3] != color_data[3:6]:
                return True
    return False
```

---

## 7. Output Structure

### 7.1 Directory Layout

```
dungeon_tilesets/
├── layout.json              # Universal tile positions
├── index.json               # Tileset metadata + animation info
├── 001_beach_cave.png       # Indexed PNG (576×144)
├── 001_beach_cave.pal.png   # Palette texture (16×N) - only if animated
├── ...
└── shader/
    ├── palette_animation.gdshader
    └── usage_example.gd
```

### 7.2 Chunk Layout

Chunks are stored in a 20-column grid within DPC data:

```python
chunk_id = 45
col = chunk_id % 20   # = 5
row = chunk_id // 20  # = 2
x = col * 24  # Pixel position
y = row * 24
```

Organized output layout (8×6 grid per terrain, 47 tiles + 1 empty):

```
┌─────────────────────────────────────────────────────────────────┐
│  WALL (8×6)        │  SECONDARY (8×6)   │  FLOOR (8×6)         │
│  192×144 px        │  192×144 px        │  192×144 px          │
└─────────────────────────────────────────────────────────────────┘
Total: 576×144 pixels
```

### 7.3 Palette Texture Layout

For shader-based animation:

```
Row 0-11:  Base palettes (static lookup)
Row 12+:   Palette 10 animation frames
Width: 16 pixels (one per color)
Height: num_base_palettes + num_animation_frames
```

---

## 8. index.json Specification

### 8.1 Schema

```json
{
  "format_version": "1.0",
  "generator": "pmd-eos-tileset-extractor",
  "game": "Pokemon Mystery Dungeon: Explorers of Sky",
  "extraction_date": "2025-01-31",
  
  "tilesets": [
    {
      "id": 0,
      "path": "tileset_000/",
      "dungeons": ["Beach Cave", "Beach Cave Pit"],
      "animated": true,
      "frame_count": 12
    }
  ],
  
  "statistics": {
    "total_tilesets": 170,
    "exported_tilesets": 144,
    "skipped_tilesets": [170]
  }
}
```

---

## 9. tileset.json Specification

### 9.1 Complete Schema

```json
{
  "format_version": "1.0",
  "generator": "pmd-eos-tileset-extractor",
  
  "tileset": {
    "id": 0,
    "name": "dungeon_000",
    "dungeons": ["Beach Cave", "Beach Cave Pit"]
  },
  
  "spritesheet": {
    "file": "chunks.png",
    "chunk_size": 24,
    "tile_size": 8,
    "tiles_per_chunk": 3,
    "grid_width": 20,
    "grid_height": 20,
    "total_chunks": 400,
    "frame_count": 12,
    "frame_duration_ms": 100
  },
  
  "terrain_types": {
    "wall": 0,
    "secondary": 1,
    "floor": 2
  },
  
  "neighbor_bits": {
    "south": 1,
    "south_east": 2,
    "east": 4,
    "north_east": 8,
    "north": 16,
    "north_west": 32,
    "west": 64,
    "south_west": 128
  },
  
  "autotile_rules": {
    "wall": [],
    "secondary": [],
    "floor": []
  },
  
  "extra_tiles": {
    "floor_variant": [],
    "void": [],
    "floor_variant_2": []
  }
}
```

### 9.2 Autotile Rules

Each terrain type maps to an array of 256 entries (one per neighbor configuration). Each entry is an array of 3 chunk IDs (visual variations).

**Usage:**
```python
neighbor_bits = NORTH | EAST  # = 16 | 4 = 20
chunk_options = tileset["autotile_rules"]["wall"][20]  # Returns [a, b, c]
chunk_id = random.choice(chunk_options)
```

### 9.3 Variation Selection

The 3 variations per configuration exist for visual variety. Game engines should:
- **At map generation:** Pick randomly and store the choice
- **At runtime:** Use the stored choice (don't re-randomize each frame)

---

## 10. Implementation Guide

### 10.1 Autotile Algorithm

```python
def get_chunk_for_tile(tileset_json, terrain_type, x, y, dungeon_map):
    current_terrain = dungeon_map[y][x]
    neighbor_bits = 0
    
    directions = [
        ("south", 0, 1, 1),
        ("south_east", 1, 1, 2),
        ("east", 1, 0, 4),
        ("north_east", 1, -1, 8),
        ("north", 0, -1, 16),
        ("north_west", -1, -1, 32),
        ("west", -1, 0, 64),
        ("south_west", -1, 1, 128)
    ]
    
    for name, dx, dy, bit in directions:
        nx, ny = x + dx, y + dy
        if is_in_bounds(nx, ny, dungeon_map):
            if dungeon_map[ny][nx] == current_terrain:
                neighbor_bits |= bit
        else:
            neighbor_bits |= bit  # Out-of-bounds = same type
    
    rules = tileset_json["autotile_rules"][terrain_type]
    chunk_options = rules[neighbor_bits]
    return random.choice(chunk_options)
```

### 10.2 Rendering a Chunk

```python
def get_chunk_rect(chunk_id, frame, tileset_json):
    ss = tileset_json["spritesheet"]
    chunk_size = ss["chunk_size"]
    grid_width = ss["grid_width"]
    frame_height = ss["grid_height"] * chunk_size
    
    x = (chunk_id % grid_width) * chunk_size
    y = (frame * frame_height) + (chunk_id // grid_width) * chunk_size
    return (x, y, chunk_size, chunk_size)
```

### 10.3 Godot 4.x Integration Outline

```gdscript
var tileset_json = JSON.parse_string(
    FileAccess.open("res://tilesets/tileset_000/tileset.json").get_as_text()
)
var chunks_texture = load("res://tilesets/tileset_000/chunks.png")

var tileset = TileSet.new()
tileset.tile_size = Vector2i(24, 24)

var source = TileSetAtlasSource.new()
source.texture = chunks_texture
source.texture_region_size = Vector2i(24, 24)
```

---

## 11. Edge Cases

### 11.1 Tileset ID Remapping

```python
if tileset_id == 170:
    tileset_id = 1  # Maps to Beach Cave
```

Tileset 170 is invalid and not exported.

### 11.2 Debug Tilesets

Tilesets 144-169 are debug/test tiles — skip these in export.

### 11.3 Out-of-Bounds Neighbors

When checking neighbors at map edges, treat out-of-bounds as same type (wall-like edge behavior). This matches the game's `treat_outside_as_wall` parameter.

### 11.4 Secondary Terrain Appearance

The `secondary` terrain type (internally "water") visually represents water, lava, or void depending on the tileset's graphics.

### 11.5 Unused Chunks

Some of the 400 chunks may be unused/blank. They are still exported for index consistency.

---

## 12. Rust Implementation Notes

### 12.1 Recommended Crates
- `image` — PNG reading/writing with indexed mode support
- `serde` / `serde_json` — JSON serialization
- NDS ROM reading — need to implement or port from ndspy

### 12.2 Key Differences from Python
1. PIL's indexed image mode → Use `image::GrayImage` or custom indexed format
2. SkyTemple's file handlers → Port the binary parsing logic
3. Palette handling → Preserve indices, don't convert to RGBA prematurely

### 12.3 SkyTemple Source Reference
- `skytemple_files/graphics/dma/_model.py` — DMA parsing
- `skytemple_files/graphics/dpc/_model.py` — DPC parsing
- `skytemple_files/graphics/dpci/_model.py` — DPCI parsing
- `skytemple_files/graphics/dpl/_model.py` — DPL parsing
- `skytemple_files/graphics/dpla/_model.py` — DPLA parsing
- `skytemple_files/dungeon_data/mappa_bin/_model.py` — MAPPA parsing

---

## 13. Execution Plan

### Phase 1: Proof of Concept (Single Tileset) ✅ COMPLETE

Python implementation verified with SkyTemple libraries. Tileset 0 (Beach Cave) extracted and validated.

### Phase 2: Validation and Refinement ✅ COMPLETE

Format verified, autotile lookup works, animation renders correctly.

### Phase 3: Batch Export ✅ COMPLETE

All valid tilesets (0-143) exported. Debug tilesets (144-169) and invalid tileset 170 skipped.

### Phase 4: Port to Rust Scraper — PENDING

Port proven Python implementation to Rust ROM scraper. Output same JSON format, verify output matches Python version.

### Phase 5: Future Enhancements — NOT IN SCOPE

- DBG background extraction (boss fight rooms)
- Godot import addon/tool
- Indexed palette mode for runtime color manipulation
- Collision data export
- Unity Rule Tile generator

---

## 14. Testing Checklist

- [x] Verify indexed PNG preserves palette indices (not converted to RGBA)
- [x] Test tileset 1 (Beach Cave) — has animation
- [x] Test tileset 3 (Mt. Bristle) — static, no animation
- [x] Test tileset 20 (Quicksand Pit) — has animation
- [x] Verify palette 10 indices (160-175) are used in water areas
- [x] Confirm palette 11 (176-191) is not used in tested tilesets
- [ ] Check shader produces correct animation in Godot
