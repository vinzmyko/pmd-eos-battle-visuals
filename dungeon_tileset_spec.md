# Dungeon Tileset Extraction Specification v1.0

## 1. Overview

This specification defines the export format for dungeon tilesets from Pokemon Mystery Dungeon: Explorers of Sky. The format is designed to be:

- **Visually faithful** to the original game
- **Engine-agnostic** but optimized for Godot 4.x
- **Easy to use** with standard image formats and JSON
- **Complete** with autotile rules for procedural dungeon generation

---

## 2. Output Structure

### 2.1 Directory Layout

```
dungeon_tilesets/
├── index.json                    # Master index of all tilesets
├── tileset_000/
│   ├── chunks.png                # Spritesheet with all chunks and animation frames
│   └── tileset.json              # Autotile rules, animation data, metadata
├── tileset_001/
│   ├── chunks.png
│   └── tileset.json
└── ... (up to 169 valid tilesets)
```

### 2.2 File Naming

- Tilesets use zero-padded 3-digit IDs: `tileset_000`, `tileset_001`, etc.
- Only valid, visually distinct tilesets are exported
- Placeholder or duplicate tilesets (e.g., tileset 170) are skipped

---

## 3. chunks.png Specification

### 3.1 Image Format

- **Format:** PNG (RGBA, 32-bit)
- **Color:** Fully baked RGB colors (not indexed)

### 3.2 Layout

The spritesheet contains all 400 chunks arranged in a 20×20 grid. Animation frames are stacked vertically.

```
┌─────────────────────────────────────┐
│  Frame 0: Chunks 0-399 (20×20 grid) │  480 × 480 px
├─────────────────────────────────────┤
│  Frame 1: Chunks 0-399 (20×20 grid) │  480 × 480 px
├─────────────────────────────────────┤
│  ...                                │
├─────────────────────────────────────┤
│  Frame N: Chunks 0-399 (20×20 grid) │  480 × 480 px
└─────────────────────────────────────┘

Total dimensions: 480 × (480 × frame_count) pixels
```

### 3.3 Chunk Addressing

To locate chunk `C` at animation frame `F`:

```python
CHUNK_SIZE = 24
GRID_WIDTH = 20
FRAME_HEIGHT = 480

x = (C % GRID_WIDTH) * CHUNK_SIZE
y = (F * FRAME_HEIGHT) + (C // GRID_WIDTH) * CHUNK_SIZE
```

### 3.4 Animation Notes

- Frame 0 is always the base frame
- If no palette animation exists, only 1 frame is exported
- Typical animated tilesets have 8-16 frames
- All chunks are present in all frames (even non-animated ones)

---

## 4. tileset.json Specification

### 4.1 Complete Schema

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

### 4.2 Field Descriptions

#### `tileset` Object

| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Original tileset index (0-169) |
| `name` | string | Identifier string `"dungeon_XXX"` |
| `dungeons` | string[] | Human-readable dungeon names that use this tileset |

#### `spritesheet` Object

| Field | Type | Description |
|-------|------|-------------|
| `file` | string | Filename of the PNG spritesheet |
| `chunk_size` | integer | Pixels per chunk side (always 24) |
| `tile_size` | integer | Pixels per tile side (always 8) |
| `tiles_per_chunk` | integer | Tiles per chunk side (always 3) |
| `grid_width` | integer | Chunks per row (always 20) |
| `grid_height` | integer | Chunks per column (always 20) |
| `total_chunks` | integer | Total chunks (always 400) |
| `frame_count` | integer | Animation frames (1 if not animated) |
| `frame_duration_ms` | integer | Milliseconds per frame |

#### `terrain_types` Object

Documents the terrain type values for reference:
- `wall` (0): Solid walls
- `secondary` (1): Water, lava, or void (depends on tileset appearance)
- `floor` (2): Walkable floor

#### `neighbor_bits` Object

Documents the 8-direction bitmask values. When building the neighbor bitfield, OR together the bits for each direction where the neighboring tile has the **same terrain type**.

#### `autotile_rules` Object

Each terrain type maps to an array of 256 entries (one per neighbor configuration). Each entry is an array of 3 chunk IDs (visual variations).

```json
"autotile_rules": {
  "wall": [
    [12, 45, 78],    // neighbor_bits = 0 (no same-type neighbors)
    [23, 56, 89],    // neighbor_bits = 1 (south neighbor is same type)
    [34, 67, 90],    // neighbor_bits = 2 (south-east neighbor is same type)
    ...              // ... up to index 255
  ],
  "secondary": [...],
  "floor": [...]
}
```

**Usage:** To get chunk options for a wall tile with neighbors to the north and east:
```python
neighbor_bits = NORTH | EAST  # = 16 | 4 = 20
chunk_options = tileset["autotile_rules"]["wall"][20]  # Returns [a, b, c]
chunk_id = random.choice(chunk_options)  # Pick one variation
```

#### `extra_tiles` Object

Additional tile variations not covered by the 8-direction autotiling:

| Key | Description |
|-----|-------------|
| `floor_variant` | Alternative floor appearance |
| `void` | Chasm/void tiles (first entry is typically the true void) |
| `floor_variant_2` | Additional floor variation |

Each is an array of chunk IDs.

---

## 5. index.json Specification

### 5.1 Schema

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
    },
    {
      "id": 1,
      "path": "tileset_001/",
      "dungeons": ["Drenched Bluff"],
      "animated": true,
      "frame_count": 8
    }
  ],
  
  "statistics": {
    "total_tilesets": 170,
    "exported_tilesets": 165,
    "skipped_tilesets": [170]
  }
}
```

### 5.2 Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `tilesets` | array | List of all exported tilesets |
| `tilesets[].id` | integer | Original tileset index |
| `tilesets[].path` | string | Relative path to tileset directory |
| `tilesets[].dungeons` | string[] | Dungeon names using this tileset |
| `tilesets[].animated` | boolean | Whether tileset has palette animation |
| `tilesets[].frame_count` | integer | Number of animation frames |
| `statistics` | object | Summary information |
| `statistics.skipped_tilesets` | integer[] | IDs of invalid/skipped tilesets |

---

## 6. Implementation Guide

### 6.1 Autotile Algorithm

When placing a tile in a procedural dungeon:

```python
def get_chunk_for_tile(tileset_json, terrain_type, x, y, dungeon_map):
    """
    terrain_type: "wall", "secondary", or "floor"
    dungeon_map: 2D array where each cell is a terrain type string
    """
    
    # 1. Get the terrain at this position
    current_terrain = dungeon_map[y][x]
    
    # 2. Check all 8 neighbors, build bitmask
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
            # Treat out-of-bounds as same type (wall-like behavior)
            neighbor_bits |= bit
    
    # 3. Look up chunk options
    rules = tileset_json["autotile_rules"][terrain_type]
    chunk_options = rules[neighbor_bits]
    
    # 4. Pick one variation (randomly for visual variety)
    return random.choice(chunk_options)
```

### 6.2 Rendering a Chunk

```python
def get_chunk_rect(chunk_id, frame, tileset_json):
    """Returns (x, y, width, height) for the chunk in the spritesheet."""
    
    ss = tileset_json["spritesheet"]
    chunk_size = ss["chunk_size"]
    grid_width = ss["grid_width"]
    frame_height = ss["grid_height"] * chunk_size
    
    x = (chunk_id % grid_width) * chunk_size
    y = (frame * frame_height) + (chunk_id // grid_width) * chunk_size
    
    return (x, y, chunk_size, chunk_size)
```

### 6.3 Godot 4.x Integration Outline

```gdscript
# Load tileset data
var tileset_json = JSON.parse_string(FileAccess.open("res://tilesets/tileset_000/tileset.json").get_as_text())
var chunks_texture = load("res://tilesets/tileset_000/chunks.png")

# Create TileSet
var tileset = TileSet.new()
tileset.tile_size = Vector2i(24, 24)

# Add atlas source
var source = TileSetAtlasSource.new()
source.texture = chunks_texture
source.texture_region_size = Vector2i(24, 24)

# For animated tiles, set up animation columns
# ... (Godot-specific animation setup)

# Create terrain sets for autotiling
# ... (Map autotile_rules to Godot's terrain system)
```

---

## 7. Edge Cases and Notes

### 7.1 Tileset 170 Redirect

Tileset 170 is invalid and redirects to tileset 1 in the game. It is not exported.

### 7.2 Unused Chunks

Some of the 400 chunks may be unused/blank. They are still exported for index consistency.

### 7.3 Variation Selection

The 3 variations per configuration exist for visual variety. Game engines should:
- **At map generation:** Pick randomly and store the choice
- **At runtime:** Use the stored choice (don't re-randomize each frame)

### 7.4 Out-of-Bounds Neighbors

When checking neighbors at map edges, treat out-of-bounds as:
- **Same as current tile** for wall-like edge behavior (recommended)
- **Or floor** for open-edge behavior

This matches the game's `treat_outside_as_wall` parameter.

### 7.5 Secondary Terrain Appearance

The `secondary` terrain type (internally "water") visually represents:
- **Water** - Blue, animated waves (most tilesets)
- **Lava** - Red/orange, animated flow (volcano tilesets)
- **Void/Chasm** - Dark, possibly animated (some tilesets)

The actual appearance is determined by the tileset's graphics, not by separate logic.

---

## 8. Example

### 8.1 Sample tileset.json (Abbreviated)

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
    "wall": [
      [1, 1, 1],
      [2, 2, 2],
      [3, 3, 3]
    ],
    "secondary": [
      [100, 100, 100],
      [101, 101, 101],
      [102, 102, 102]
    ],
    "floor": [
      [200, 200, 200],
      [201, 201, 201],
      [202, 202, 202]
    ]
  },
  
  "extra_tiles": {
    "floor_variant": [250, 251, 252],
    "void": [260, 261, 262],
    "floor_variant_2": [270, 271, 272]
  }
}
```

*Note: Actual chunk IDs would be the real values from the DMA data. The `...` would expand to all 256 entries per terrain type.*

---

## 9. Execution Plan

### Phase 1: Proof of Concept (Single Tileset)

**Goal:** Extract tileset 0 completely and verify the format works.

**Tasks:**
1. [ ] Create Python script using SkyTemple libraries
2. [ ] Load ROM and extract tileset 0 data (DMA, DPC, DPCI, DPL, DPLA)
3. [ ] Render all 400 chunks to a single frame PNG
4. [ ] Add palette animation frames (bake all frames)
5. [ ] Export DMA rules to JSON format
6. [ ] Map tileset 0 to dungeon names (lookup from MAPPA/dungeon data)
7. [ ] Manually verify output in image viewer
8. [ ] Test loading in Godot 4.x

**Success Criteria:**
- chunks.png displays correctly with all 400 chunks
- Animation frames render water/lava cycling
- JSON loads and parses without errors
- Can programmatically look up correct chunk for a given terrain + neighbor config

**Deliverables:**
```
output/
├── tileset_000/
│   ├── chunks.png
│   └── tileset.json
└── extraction_log.txt
```

**Resources Required:**
- Pokemon Mystery Dungeon: Explorers of Sky ROM
- Python 3.8+
- SkyTemple Files library (`pip install skytemple-files`)
- ndspy library (`pip install ndspy`)
- Pillow (`pip install pillow`)

**Reference Code:**
- `skytemple_files/export_maps.py` - ROM loading pattern
- `skytemple_files/graphics/dma/dma_drawer.py` - Chunk rendering
- `skytemple_files/graphics/dpc/_model.py` - `chunks_to_pil()` method

---

### Phase 2: Validation and Refinement

**Goal:** Verify the export works in practice and refine the format.

**Tasks:**
1. [ ] Create simple Godot test scene that loads the tileset
2. [ ] Implement autotile lookup using exported JSON
3. [ ] Render a test dungeon layout using the data
4. [ ] Compare visually against original game / emulator screenshot
5. [ ] Identify any issues with the format
6. [ ] Refine JSON structure if needed
7. [ ] Document any edge cases discovered

**Success Criteria:**
- Rendered test dungeon looks visually identical to original game
- Autotile transitions look correct at all neighbor configurations
- Animation plays at correct speed

**Potential Issues to Watch For:**
- Chunk ordering mismatch
- Wrong animation frame timing
- Missing or incorrect dungeon name mappings
- Edge cases in neighbor bitfield calculation

---

### Phase 3: Batch Export

**Goal:** Export all valid tilesets and create master index.

**Tasks:**
1. [ ] Modify script to iterate all tilesets (0-169)
2. [ ] Add validation to skip invalid tilesets (170, any others)
3. [ ] Implement dungeon name lookups for all tilesets
4. [ ] Generate `index.json` master file
5. [ ] Add progress logging and error handling
6. [ ] Run full extraction
7. [ ] Spot-check several tilesets across different dungeon types

**Success Criteria:**
- All valid tilesets exported without errors
- `index.json` contains complete listing
- File sizes are reasonable (estimate ~500KB-2MB per tileset with animation)
- Different dungeon types (cave, forest, volcano, etc.) all look correct

**Estimated Output:**
- ~165 tilesets × ~1MB average = ~165MB total
- Plus `index.json` (~50KB)

---

### Phase 4: Integration with Existing Scraper (Future)

**Goal:** Port the Python implementation to your Rust ROM scraper.

**Tasks:**
1. [ ] Analyze Python implementation for porting
2. [ ] Implement dungeon.bin parsing in Rust
3. [ ] Implement DMA/DPC/DPCI/DPL/DPLA handlers in Rust
4. [ ] Implement chunk rendering with palette animation
5. [ ] Output same JSON format
6. [ ] Verify output matches Python version

**Notes:**
- This phase is deferred until Python version is proven
- May reuse existing parsing code if your scraper already handles some formats

---

### Phase 5: Future Enhancements (Not in Scope)

These are documented for future consideration but not part of the current plan:

- [ ] DBG background extraction (boss fight rooms)
- [ ] Shadow sprite extraction (files 995-997)
- [ ] Godot import addon/tool
- [ ] Indexed palette mode for runtime color manipulation
- [ ] Collision data export
- [ ] Unity Rule Tile generator

---

## 10. Summary

| Aspect | Decision |
|--------|----------|
| **Image format** | Baked RGBA PNG |
| **Animation** | Pre-rendered frames stacked vertically |
| **Data format** | JSON with autotile rules |
| **Files per tileset** | 2 (chunks.png + tileset.json) |
| **Autotile system** | 8-direction bitmask (256 configurations × 3 variations) |
| **Terrain types** | wall, secondary (water/lava/void), floor |
| **Engine target** | Godot 4.x primary, engine-agnostic design |

