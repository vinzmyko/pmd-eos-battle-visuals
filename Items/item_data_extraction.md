# Item Data & Icon Extraction (item_p / item_s_p / items.itm.img)

## Summary

- Covers item stats, exclusive-item metadata, item names/descriptions, and item + trap icon graphics in PMD:EOS
- Item stats live in `BALANCE/item_p.bin` (SIR0-wrapped, 16 bytes/entry, 1400 entries); exclusive-item metadata in `BALANCE/item_s_p.bin` (4 bytes/entry, 956 entries)
- Item icons are **not** WAN sprites — they live in `dungeon.bin` as `items.itm.img`, a paletted chunk container handled by `ImgItm`
- Icons are 16×16 with 64 sprite shapes × 16 palettes; the item's appearance is the `(sprite_id, palette_id)` pair, so many items share a shape and differ only by recolour
- Trap icons use the identical `ImgItm` format at `dungeon.bin` `traps.trp.img`, but are **24×24 with 35 sprites × 2 palettes**
- `SYSTEM/dw_icon.bin` + `dw_plt.bin` are a red herring (512 bytes / 1 palette, overlay 8, Download Play icon) — not item graphics
- This document is considered complete for the extraction path; open items are noted at the end

## File Locations in ROM

| File Path | Description |
|-----------|-------------|
| `BALANCE/item_p.bin` | Per-item stats table, SIR0-wrapped |
| `BALANCE/item_s_p.bin` | Exclusive-item type/parameter table, SIR0-wrapped |
| `MESSAGE/text_e.str` | Item names, short descriptions, long descriptions |
| `DUNGEON/dungeon.bin` → `items.itm.img` (index 1022) | Item icon graphics + palettes |
| `DUNGEON/dungeon.bin` → `traps.trp.img` (index 1033) | Trap icon graphics + palettes |
| `TABLEDAT/item00.dat` … `item15.dat` | 16 files, one per item category; loaded by overlays 3/4. Small (18–402 bytes), `u16` pairs. Purpose not established — likely per-category spawn/sort tables. Not graphics. |

### Path strings in code

Confirmed by scanning arm9 + all overlays for `.bin`/`.dat`/`.wan` literals:

| Path | Found in |
|------|----------|
| `/BALANCE/item_p.bin` | arm9 (`ITEM_P_BIN_FILE_PATH`) |
| `/BALANCE/item_s_p.bin` | arm9 (`ITEM_S_P_BIN_FILE_PATH`) |
| `/BALANCE/st_i2n_j.bin` | arm9 (`ITEM_ST_I2N_BIN_FORMAT`; EU variant is a `%s` format string over `ITEM_LANG_FILE_ARRAY`) |
| `TABLEDAT/item00.dat` | overlay 3, overlay 4 |
| `system/dw_icon.bin`, `system/dw_plt.bin` | overlay 8 (Download Play — unrelated) |

Notably, **no path string exists for the item icons** — they are inside `dungeon.bin`, loaded by index, which is why a string scan cannot find them.

## Data Structures

### `item_p.bin` — Item Data (16 bytes/entry, 1400 entries)

The pret/pmd-sky struct is authoritative; psy_commando's 2015 wiki notes label several fields "Unknown" that are now identified.

```c
struct item_data {
    /* 0x0 */ u16 buy_price;
    /* 0x2 */ u16 sell_price;
    /* 0x4 */ enum item_category category;   // u8
    /* 0x5 */ u8  sprite_id;                 // → items.itm.img sprite index
    u8 fill5[0x8 - 0x6];                     // 0x6-0x7: item_id (redundant with index)
    /* 0x8 */ u16 move_id;                   // TMs/HMs/orbs; role unclear otherwise
    /* 0xA */ u8  quantity_limit[2];         // thrown items: min/max stack on pickup
    /* 0xC */ u8  palette_id;                // → items.itm.img palette index
    /* 0xD */ u8  action_name;               // menu verb (Use/Eat/Ingest/Equip…)
    /* 0xE */ u8  flags;
};
```

**Field naming reconciliation** — SkyTemple vs psy_commando wiki vs pret:

| Offset | pret / SkyTemple | psy_commando wiki (obsolete) |
|--------|------------------|------------------------------|
| 0x08 | `move_id` | Item Parameter #1 |
| 0x0A | `quantity_limit[0]` / `range_min` | Item Parameter #2 |
| 0x0B | `quantity_limit[1]` / `range_max` | Item Parameter #3 |
| 0x0C | `palette_id` | Unknown#1 |
| 0x0D | `action_name` | Unknown#2 |
| 0x0E | `flags` | Unknown#3 |

```c
enum item_data_flag {
    ITEM_DATA_FLAG_VALID              = 1 << 0,
    ITEM_DATA_IN_TIME_DARKNESS        = 1 << 1,
    ITEM_DATA_FLAG_THROWABLE_AT_ENEMY = 1 << 5,
    ITEM_DATA_FLAG_THROWABLE_AT_ALLY  = 1 << 6,
    ITEM_DATA_FLAG_CONSUMABLE         = 1 << 7,
};
```

SkyTemple exposes these as `is_valid`, `is_in_td`, `ai_flag_1`, `ai_flag_2`, `ai_flag_3`.

**Validity gate.** `EnsureValidItem` silently substitutes `ITEM_PLAIN_SEED` (85) for any id that is `<= 0`, `>= NUM_ITEM_IDS`, or has `ITEM_DATA_FLAG_VALID` clear. Every accessor (`GetItemCategory`, `GetItemBuyPrice`, `GetItemSpriteId`, …) runs through it, so invalid entries never render. **Scrapers must filter on `is_valid`** — 1311 of 1400 entries are valid in the US ROM.

### Categories

```c
enum item_category {
    CATEGORY_THROWN_LINE = 0, CATEGORY_THROWN_ARC = 1,
    CATEGORY_BERRIES_SEEDS_VITAMINS = 2, CATEGORY_FOOD_GUMMIES = 3,
    CATEGORY_HELD_ITEMS = 4, CATEGORY_TMS_HMS = 5, CATEGORY_POKE = 6,
    CATEGORY_UNK_7 = 7, CATEGORY_OTHER = 8, CATEGORY_ORBS = 9,
    CATEGORY_LINK_BOX = 10, CATEGORY_USED_TM = 11,
    CATEGORY_TREASURE_BOXES_1 = 12, CATEGORY_TREASURE_BOXES_2 = 13,
    CATEGORY_TREASURE_BOXES_3 = 14, CATEGORY_EXCLUSIVE_ITEMS = 15,
    CATEGORY_DUMMY = 16,
};
```

`quantity_limit[]` is only meaningful when `category <= CATEGORY_THROWN_ARC` (`IsThrownItem`). For other categories it reads 0.

### `item_s_p.bin` — Exclusive Item Data (4 bytes/entry, 956 entries)

**Not a separate icon source.** Exclusive items are ordinary item ids (444–1351) with ordinary `item_p` entries and ordinary `sprite_id`/`palette_id`. This file only adds rarity/target metadata.

```c
struct item_exclusive_data {
    u8  type;    // 0x0 (u16 in the file; SkyTemple decodes to ItemSPType)
    u16 unk2;    // 0x2 — the parameter
};
```

**Indexing is offset, not direct.** `GetExclusiveItemOffsetEnsureValid` maps item id → table index:

```c
static s16 GetExclusiveItemOffsetEnsureValid(s16 item_id) {
    if (item_id < ITEM_PRISM_RUFF || item_id >= NUM_ITEM_IDS) return ITEM_PLAIN_SEED;
    if (IsItemValid(item_id)) return (s16)(item_id - ITEM_PRISM_RUFF);
    return ITEM_PLAIN_SEED;
}
```

So `item_s_p.item_list[0]` is `ITEM_PRISM_RUFF` (444). Scrapers must apply `index = item_id - 444`.

| `type` | Rarity | Slot | Exclusive To | Extra trait |
|--------|--------|------|--------------|-------------|
| 0x0 | – | – | – | – |
| 0x1 | ★ | 1 | Type | – |
| 0x2 | ★ | 2 | Type | – |
| 0x3 | ★★ | – | Type | – |
| 0x4 | ★★★ | – | Type | – |
| 0x5 | ★ | 1 | Pokémon | – |
| 0x6 | ★ | 2 | Pokémon | – |
| 0x7 | ★★ | – | Pokémon | – |
| 0x8 | ★★★ | – | Pokémon | – |
| 0x9 | ★★★ | – | Pokémon | May hatch holding the item |
| 0xA | ★★★ | – | Pokémon | Unknown (only Eeveelutions + Tyrogue line) |

`parameter` is the National Dex number when exclusive to a Pokémon, or the type id when exclusive to a type.

### Names & Descriptions (`MESSAGE/text_e.str`)

Relevant `string_index_data.string_blocks` entries:

- `Item Names` — 1400 entries, 1:1 with item ids
- `Item Short Descriptions`
- `Item Long Descriptions`
- (also `Exclusive Items Strings`, `Dungeon Item Action Menu Strings`, etc.)

**In-code offset:** `GetItemName` reads `StringFromId(valid_item_id + ITEM_NAME_OFFSET)` where the offset is region-dependent — `0x1A76` (NA), `0x1A78` (EU), `0x0D93` (JP). Via SkyTemple this is handled by the string block bounds; the constant only matters for a from-scratch reimplementation.

**Markup tags.** Names contain presentation markup, e.g. `[FT:1]T[FT:0]-Stone` for the alphabet stones. `[FT:n]` is a font/colour switch. Strip with `\[[A-Z]{2}(?::[^\]]*)?\]` for display names; keep the raw string if you intend to reproduce in-game rendering. Related formatting constants in arm9: `ITEM_NAME_FORMAT_YELLOW` / `_INDIGO` / `_PLAIN` / `_CREAM`, selected per-category by `GetItemNameFormatted` (exclusive items → indigo, treasure boxes → yellow, else cream/plain).

## Icon Graphics (`ImgItm`)

### Container

`items.itm.img` is SIR0-wrapped (`53 49 52 30` magic), 9280 bytes. SkyTemple parses it to `skytemple_files.graphics.img_itm.model.ImgItm`:

```python
img = dungeon_bin.get("items.itm.img")
img.sprites        # list of chunk blobs
img.palettes       # list of 16-colour palettes, flat [R,G,B, R,G,B, ...] (48 bytes)
img.to_pil(sprite_index, palette_index)  # -> PIL Image, mode 'P'
img.from_pil(...)
img.sir0_serialize_parts()
```

Internally `to_pil` builds a dummy tilemap of `CHUNK_DIM * CHUNK_DIM` `TilemapEntry`s and calls the shared chunk renderer, so cell size = `TILE_DIM * CHUNK_DIM` and comes from the module, **not** from anything caller-supplied.

### Dimensions (US ROM)

| Container | Cell | Sprites | Palettes |
|-----------|------|---------|----------|
| `items.itm.img` | 16×16 | 64 | 16 |
| `traps.trp.img` | 24×24 | 35 | 2 |

The 24×24 trap cell matches the dungeon chunk size; the 16×16 item cell does **not** — item icons are drawn centred in a 24×24 floor tile when lying on the ground. Client should offset by (4, 4) if compositing onto a dungeon grid.

### Sprite/palette usage

`(sprite_id, palette_id)` from `item_p` is the full appearance key. In the US ROM: **56 distinct sprite ids used (max 62, of 64 available), 210 distinct pairs across 1311 valid items.**

Worked examples:

| Item | id | sprite | palette |
|------|----|--------|---------|
| Stick | 1 | 0 | 1 |
| Iron Thorn | 2 | 0 | 2 |
| Silver Spike | 3 | 0 | 11 |
| Cacnea Spike | 5 | 0 | 10 |
| Corsola Twig | 6 | 0 | 4 |
| Gravelerock | 7 | 1 | 1 |
| Geo Pebble | 8 | 1 | 11 |
| Gold Thorn | 9 | 59 | 3 |
| Plain Seed | 85 | 14 | 13 |
| Gravelyrock | 137 | 1 | 1 |
| Prism Ruff | 444 | 37 | 14 |

Sprite 0 is the generic thrown-spike shape recoloured five ways; sprite 1 the rock shape. Note Gravelerock (7) and Gravelyrock (137) are byte-identical in appearance despite being unrelated items — **appearance is many-to-one with item id**, so an asset map must be keyed by item, not by cell.

Max `palette_id` referenced by any valid item is 15, matching `len(img.palettes) == 16`. All 16 palettes are reachable.

### Transparency

Colour index 0 is transparent. `to_pil(i, 0)` returns mode `'P'` with palette 0 applied, so **index 0 renders as palette 0's colour 0 (a blue) rather than as alpha** — this is a display artefact of the paletted export, not corrupt data. Flatten manually to RGBA, mapping index 0 → `(0,0,0,0)`.

### Non-icon neighbours in dungeon.bin

Files adjacent to `items.itm.img` that were checked and ruled out:

| Index | Name | Size | Verdict |
|-------|------|------|---------|
| 1023 | `1023.sir0` | 8224 | Fade-to-black colvec (see `weather_visual_pipeline.md`), not item palettes |
| 1024/1025 | `.wte` | 1120 / 3184 | Unrelated textures |
| 1028 | `1028.wan.bin` | — | Candidate for ground-object WAN (in-world item/object sprites) — **not investigated** |
| 1030 | `1030.dpl` | 896 | Palette list, not used by `ImgItm` (which carries its own) |
| 1034 | `colormap.colvec` | — | Weather colormaps |

`ImgItm` holds both graphics and palettes, so no companion file is needed.

## Extraction Recipe

```python
rom = NintendoDSRom.fromFile(ROM_PATH)
config = get_ppmdu_config_for_rom(rom)

item_p   = FileType.ITEM_P.deserialize(rom.getFileByName("BALANCE/item_p.bin"))
item_s_p = FileType.ITEM_SP.deserialize(rom.getFileByName("BALANCE/item_s_p.bin"))  # note: ITEM_SP, not ITEM_S_P
dungeon_bin = FileType.DUNGEON_BIN.deserialize(
    rom.getFileByName("DUNGEON/dungeon.bin"), config)

items_img = dungeon_bin.get("items.itm.img")
traps_img = dungeon_bin.get("traps.trp.img")

str_file = FileType.STR.deserialize(rom.getFileByName("MESSAGE/text_e.str"))
b = config.string_index_data.string_blocks
names = str_file.strings[b["Item Names"].begin:b["Item Names"].end]
```

### API gotchas encountered

| Symptom | Cause / fix |
|---------|-------------|
| `FileType.ITEM_S_P` AttributeError | Handler is `FileType.ITEM_SP` |
| `rom.filenames` raises "Folders are not iterable" | It's an ndspy `Folder`; recurse `.files` / `.folders` |
| `config.dungeon_bin_files` AttributeError | Not on `Pmd2Data`; enumerate via `DungeonBinPack` |
| `dungeon_bin.get_filenames()` AttributeError | Singular `get_filename(index)` |
| `dungeon_bin.get_raw(1022)` KeyError | `get_raw` takes the **filename**, not the index |

### Atlas layout

Export one atlas per container: row = sprite id, column = palette id, cell size probed from `to_pil(0,0).width`. Cell origin is `(palette * cell, sprite * cell)`. Items → 256×1024; traps → 48×840. Emit palettes as hex alongside so the client can recolour a shape into a new palette without new art.

## Open Questions

- **Trap names.** `traps.trp.img` has 35 sprites but no `item_p`-equivalent name table was located; no `text_e.str` block matching "trap" was found. Canonical ordering is `enum trap_id` in pret/pmd-sky (`include/trap.h`); the last few entries are likely stairs/marker graphics rather than traps proper. Currently exported as `trap_00`…`trap_34` placeholders.
- **`TABLEDAT/itemNN.dat`.** 16 files, one per category, loaded by overlays 3/4 via a single format string. Contents are `u16` pairs; purpose unconfirmed.
- **`dungeon.bin[1028]` (`1028.wan.bin`).** Likely the in-world ground-object sprite bank (the `DUNGEON/sub_obj.wan` analogue). Not yet opened — relevant if the client needs items as they appear lying on the floor rather than as menu icons.
- **8 unused sprite ids** in `items.itm.img` (56 of 64 referenced, max used 62). Free-standing art with no canon name.
- **Trap palette 1.** Two palettes exist; the second is presumed the revealed/highlighted variant but this is not confirmed against the renderer.
- **`item_p` offsets 0x6–0x7.** pret marks this `fill5` but it holds the item id, redundant with the array index. Whether any code reads it is unverified.

## Cross-References

> See `Data Structures/dungeon_tileset_spec.md` for the `dungeon.bin` container access pattern and the 24×24 chunk convention that trap icons follow.

> See `weather_visual_pipeline.md` for `dungeon.bin` indices 1001/1003/1005/1023/1031/1034, which sit in the same non-tileset index range as `items.itm.img` (1022) and `traps.trp.img` (1033).
