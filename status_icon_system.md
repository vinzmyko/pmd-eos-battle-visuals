# Status Icon System (manpu_su.sma)

## Summary

- Status condition icons are stored in `/SYSTEM/manpu_su.sma`, not in effect.bin
- SMA is a distinct format from WAN, with its own image data, palette, and animation blocks
- 18 animation entries confirmed in the US ROM (index 0 is a null/padding entry)
- SkyTemple provides read+export support only — no write support or UI integration
- The mapping from status condition ID → SMA animation index is NOT yet documented
- The display system (how/where icons are rendered in dungeon mode) is NOT yet documented

## File Location

| File | Path | Purpose |
|------|------|---------|
| Status icons | `SYSTEM/manpu_su.sma` | Animated status condition icons |
| Unknown SMA | `SYSTEM/manpu_ma.sma` | Unknown — no image data, structure/use undocumented |

## Confirmed Animation Inventory (US ROM)

| Index | Width (blocks) | Height (blocks) | Frames | Byte Offset |
|-------|---------------|-----------------|--------|-------------|
| 0 | 0 | 0 | 0 | 0 (null entry) |
| 1 | 1 | 1 | 14 | 0 |
| 2 | 2 | 2 | 7 | 448 |
| 3 | 2 | 2 | 16 | 1344 |
| 4 | 4 | 4 | 6 | 3392 |
| 5 | 4 | 2 | 4 | 6464 |
| 6 | 2 | 2 | 9 | 7488 |
| 7 | 2 | 2 | 8 | 8640 |
| 8 | 1 | 2 | 8 | 9664 |
| 9 | 2 | 2 | 13 | 10176 |
| 10 | 2 | 2 | 10 | 11840 |
| 11 | 2 | 2 | 13 | 13120 |
| 12 | 1 | 1 | 14 | 14784 |
| 13 | 2 | 2 | 10 | 15232 |
| 14 | 1 | 1 | 14 | 16512 |
| 15 | 2 | 2 | 8 | 16960 |
| 16 | 2 | 2 | 10 | 17984 |
| 17 | 2 | 2 | 13 | 19264 |

1 block = 8 pixels. Palette index 0-15 are valid; each index references one of 16 palettes
stored in the Color Data block.

## SMA Format Summary

Image data is stored as a flat nibble array (4bpp), read sequentially.
Frames within an animation share the same block dimensions.
Palette data contains 16 palettes of 16 colors each (4 bytes per color: R, B, G, padding).

**Note:** Color byte order is R, B, G (NOT R, G, B) — verified in `sma/model.py`.

## SkyTemple Access
```python
from ndspy.rom import NintendoDSRom
from skytemple_files.graphics.sma.handler import SmaHandler

rom = NintendoDSRom.fromFile("pmd_eos_us.nds")
sma = SmaHandler.deserialize(rom.getFileByName("SYSTEM/manpu_su.sma"))

# Export all animations for all palettes
for palette_idx in range(16):
    SmaHandler.export_sheets(f"out/palette_{palette_idx}", sma, palette_idx)
```

`SmaHandler.serialize()` raises `NotImplementedError` — write support does not exist.

## Open Questions

- Which palette index is the "correct" one used by the game at runtime?
- What maps status condition enum values to SMA animation indices?
  - Likely hardcoded in `overlay_0029.bin` (dungeon status logic) or `overlay_0010.bin`
  - `"Debug Strings (Status)"` string block (NA indices 15971–16051) may name the status IDs
- How are icons rendered in dungeon mode?
  - Possibly a separate UI rendering path, NOT the 32-slot effect pool (`FUN_022bf764`)
  - Or a looping effect spawned per entity with `loop_flag=1` when status is applied
- What is `manpu_ma.sma`? No image data, purpose unknown.
- Do buff/debuff stat changes use this same SMA file or a different source?

## Breadcrumbs

- `overlay_0010.bin` — dungeon HUD/display overlay, likely candidate for icon rendering
- `overlay_0029.bin` — status condition application logic, likely candidate for ID→index mapping
- pmdsky-debug `symbols/na/overlay10.yml` — search for any "status", "icon", or "manpu" symbols
- `"Status Names and Descriptions"` string block (NA 13554–13760) — ~103 status conditions;
  count vs 17 non-null SMA entries suggests many statuses share icons or have no icon
