# PMD: Explorers of Sky — Tileset Properties

## Overview

Each dungeon tileset in PMD:EoS has associated gameplay properties stored in two locations within the ROM. These properties affect weather, ambient effects, move behaviour, and secondary terrain type.

## Data Sources

### 1. Secondary Terrain Types — ARM9 Binary

**Location:** `SECONDARY_TERRAIN_TYPES` table in the ARM9 binary.

**Structure:** 1 byte per tileset (170 entries), indexed by tileset ID.

**Values:**

| Value | Type  | Description |
|-------|-------|-------------|
| 0     | Water | Standard water tiles. Water-type and Flying/Ghost Pokemon can traverse. |
| 1     | Lava  | Lava tiles. Fire-type and Flying/Ghost Pokemon can traverse. Non-fire types take damage. |
| 2     | Void  | Empty chasms. Only Flying and Ghost Pokemon can traverse. |

**Extraction via SkyTemple:**

```python
from skytemple_files.hardcoded.dungeons import HardcodedDungeons

terrains = HardcodedDungeons.get_secondary_terrains(arm9bin, config)
# Returns list[SecondaryTerrainTableEntry] indexed by tileset_id
# SecondaryTerrainTableEntry enum: WATER=0, LAVA=1, VOID=2
```

**ROM decompilation reference:**

```c
// FloorSecondaryTerrainIsChasm reads from a byte array where 0x02 = chasm
bool FloorSecondaryTerrainIsChasm(int16_t tileset_id) {
    if (*DAT_022e03a8 == '\0') {
        return *(char *)(DAT_022e03ac + tileset_id) == '\x02';
    }
    return '\x01';
}
```

**Known mappings (US ROM):**

- **Lava (5 tilesets):** 57, 58, 110, 112, 123
- **Void (11 tilesets):** 26, 27, 42, 43, 44, 54, 55, 56, 61, 73, 99
- **Water:** All remaining tilesets (0–143, excluding above)

### 2. Tileset Properties — Overlay 10

**Location:** `TILESET_PROPERTIES` table in overlay 10.

**Structure:** 12 bytes per tileset (`tileset_property` struct).

```c
struct tileset_property {
    int32_t  map_color;           // 0x00: Minimap colour (see TilesetMapColor)
    uint8_t  stirring_effect;     // 0x04: Ambient particle effect
    uint8_t  secret_power_effect; // 0x05: Effect of the move Secret Power
    uint16_t camouflage_type;     // 0x06: PokeType for the move Camouflage
    uint16_t nature_power_move;   // 0x08: Which move Nature Power becomes
    uint8_t  weather_effect;      // 0x0A: Ambient weather
    bool     is_water_tileset;    // 0x0B: Affects shadows, Dive, Drought Orb
};
```

**Extraction via SkyTemple:**

```python
from skytemple_files.hardcoded.dungeons import HardcodedDungeons

# Note: requires overlay10 binary, not arm9
ov10 = get_binary_from_rom(rom, config.bin_sections.overlay10)
properties = HardcodedDungeons.get_tileset_properties(ov10, config)
# Returns list[TilesetProperties]
```

#### Field Details

**Map Color** (`map_color`) — Minimap background colour.

| Value | Colour |
|-------|--------|
| 0     | White  |
| 1     | Black  |
| 2     | Red    |
| 3     | Blue   |
| 4     | Green  |
| 5     | Yellow |
| 6     | Orange |
| 7     | Purple |
| 8     | Pink   |

**Stirring Effect** (`stirring_effect`) — Ambient particle system displayed in the dungeon.

| Value | Effect      |
|-------|-------------|
| 0     | Leaves      |
| 1     | Snowflakes  |
| 2     | Flames      |
| 3     | Sand        |
| 4     | Bubbles     |
| 5     | Earthquake  |

**Secret Power Effect** (`secret_power_effect`) — Secondary effect when the move Secret Power is used.

| Value | Effect        |
|-------|---------------|
| 1     | Sleep         |
| 2     | Speed -1      |
| 3     | Attack -1     |
| 5     | Accuracy -1   |
| 7     | Cringe        |
| 8     | Freeze        |
| 9     | Paralysis     |

**Camouflage Type** (`camouflage_type`) — The Pokemon type assigned by the move Camouflage in this tileset. Uses the standard `PokeType` enum.

**Nature Power Move** (`nature_power_move`) — The move that Nature Power becomes in this tileset.

| Value | Move        |
|-------|-------------|
| 4     | Earthquake  |
| 7     | Rock Slide  |
| 9     | Tri Attack  |
| 10    | Hydro Pump  |
| 11    | Blizzard    |
| 12    | Ice Beam    |
| 13    | Seed Bomb   |
| 14    | Mud Bomb    |

**Weather Effect** (`weather_effect`) — Ambient weather displayed in the dungeon.

| Value | Effect  |
|-------|---------|
| 0     | Clear   |
| 1–6   | Foggy (levels 1–6) |

**Is Water Tileset** (`is_water_tileset`) — Boolean flag. When true:

- A different shadow type is displayed under monsters (water-specific shadow).
- Drought Orbs do not work.
- The move Dive can be used anywhere on the floor.

**ROM decompilation reference:**

```c
// Reads is_water_tileset from the tileset_property table
bool IsWaterTileset(void) {
    return *(bool *)(DAT_02337ebc + *(short *)(*DAT_02337eb8 + 0x40d4) * 0xc);
}
```

### 3. Per-Floor Secondary Terrain — Mappa

The mappa floor layout also specifies a `secondary_terrain` field and a `has_secondary_terrain` terrain setting per floor. This is technically a per-floor property, not per-tileset, though in practice the ROM consistently pairs tilesets with their expected secondary terrain type.

```python
floor_layout = mappa.floor_lists[mappa_idx][floor_idx].layout
floor_layout.secondary_terrain       # u8: 0=Water, 1=Lava, 2=Void
floor_layout.terrain_settings.has_secondary_terrain  # bool
```

## Research Sources

- **SkyTemple Files** — Python library for reading/writing PMD:EoS ROM data. Contains the parsers for all structures above.
  - Repository: [github.com/SkyTemple/skytemple-files](https://github.com/SkyTemple/skytemple-files)
  - Key file: `skytemple_files/hardcoded/dungeons.py`
  - Enum definitions: `skytemple_files/hardcoded/symbols/manual/enums.py`
  - Mappa floor layout: `skytemple_files/dungeon_data/mappa_bin/_python_impl/floor_layout.py`
- **DMA File Format** — SkyTemple documentation on autotiling rules.
  - Key file: `skytemple_files/graphics/dma/README.rst`
  - Describes the 10-bit bitfield indexing: 2 bits for tile type (Wall/Secondary/Floor), 8 bits for neighbour connectivity.
- **PMD ROM Decompilation** — C decompiled source from the ROM, referenced for struct layouts and function behaviour.
  - `tileset_property` struct (12 bytes)
  - `FloorSecondaryTerrainIsChasm()` — checks byte array for chasm (value 0x02)
  - `IsWaterTileset()` — reads `is_water_tileset` from overlay 10 table
  - `InitTilesetBuffer()` — tileset loading with index offsets (< 0xAA = tileset, >= 0xAA = background)
- **psy_commando Dropbox** — Original research documentation on dungeon data formats.
- **Project Pokemon / MegaMinerd** — DMA index mapping research.
