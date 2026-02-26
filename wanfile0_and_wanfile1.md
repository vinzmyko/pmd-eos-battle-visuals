# Confirmed Results (Python/skytemple_files Testing)

## WanFile0 (Type 1 Effects)

**Structure:** 8bpp (`is256Color=1`), no image data of its own. `imgData.imageLists` references tiles that only exist in file 292.

**Extraction method — hybrid construction:**
1. Parse file 292 → provides `imgData`, `customPalette` (16 rows), `imgType`, `is256Color`, `paletteOffset`
2. Parse file 0 → provides `frameData` (max base `paletteIndex` = 11), `animData`
3. Combine: imgData/palette from 292, frameData/animData from file 0
4. Apply palette shift: `piece.paletteIndex += palette_index` per piece (deep copy frameData first!)
5. Valid palette range: 0–4 (16 rows - max base 11 - 1 = 4)

**Critical implementation note:** `frameData` must be deep copied per palette variant. Pieces are mutated in-place — without copying, `paletteIndex` compounds across iterations.

**8bpp tile addressing (`GeneratePiece`):** `blockOffset` is NOT a direct index into `imageLists`. The code walks all `imageLists` entries, accumulating flattened byte lengths (rounded up to 128-byte boundaries), until the byte position matches `blockOffset * 128`. This is how the ROM's VRAM tile addressing works — tiles are in a contiguous block, not individually addressed.

## WanFile1 (Type 2 Effects)

**Structure:** 8bpp, fully self-contained. Has its own `imgData` (86 image lists), own `customPalette` (3 rows), own `frameData` and `animData`.

**Extraction method — raw export, no hybridization:**
1. Parse file 1 → export directly via `FileType.WAN.EFFECT.export_sheets()`
2. No palette shifting — only 3 palette rows with max base index 2, leaving 0 room for shifting
3. File 292's palette is NOT compatible — using it causes `None` imgPx lookups because palette entries at file 1's expected indices are transparent

**Why hybrid fails for file 1:**
- File 1's `imgData.imageLists` has 86 entries with its own tile data
- File 292's `imageLists` has 114 entries with different tile data
- Swapping imgData sources causes tile/palette mismatches → `GeneratePiece` returns `None` for all pieces → zero-size bounding boxes

## File 292 (Bootstrap)

Only needed for WanFile0. Loaded at dungeon init by `FUN_022bd82c` to populate VRAM with tile data and palette, then discarded. File 1 doesn't need it because it ships with its own tiles.

## Summary Table

| Property | File 0 (Type 1) | File 1 (Type 2) |
|----------|-----------------|-----------------|
| Has own imgData | No | Yes (86 lists) |
| Needs file 292 | Yes (imgData + palette) | No |
| Palette rows | 16 (from file 292) | 3 (own) |
| Max base paletteIndex | 11 | 2 |
| Palette shift range | 0–4 | None (0 only) |
| Export method | Hybrid construction | Raw export |

## Porting to Rust

The Rust scraper needs two code paths:
- **Type 1 effects:** Replicate the hybrid construction. Load file 292 for imgData/palette, load file 0 for frameData/animData, apply palette offset per `effect_animation_info.palette_index`.
- **Type 2 effects:** Load file 1 directly. Use its own imgData and palette as-is. Ignore `palette_index` from `effect_animation_info` (or verify ROM always passes 0 for type 2).

For 8bpp tile addressing, replicate skytemple's `GeneratePiece` logic: walk `imageLists` accumulating flattened lengths with 128-byte alignment to find the correct tile block for a given `blockOffset`.
