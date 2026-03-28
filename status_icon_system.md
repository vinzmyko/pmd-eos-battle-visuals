# Status Icon System (manpu_su.sma)

## Summary

- Status condition icons are stored in `/SYSTEM/manpu_su.sma`, not in effect.bin
- SMA is a distinct format from WAN, with its own image data, palette, and animation blocks
- 18 animation entries confirmed in the US ROM (index 0 is a null/padding entry)
- SkyTemple provides read+export support only — no write support or UI integration
- The complete mapping chain is now documented:
  **game status → `UpdateStatusIconFlags()` → `status_icon_flags` bitfield → renderer → SMA animation index + palette**
- The display system renders icons above monster sprites in dungeon mode via `FUN_022dc820`

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
| 4 | 4 | 4 | 6 | 6464 |
| 5 | 4 | 2 | 4 | 7488 |
| 6 | 2 | 2 | 9 | 8640 |
| 7 | 2 | 2 | 8 | 9664 |
| 8 | 1 | 2 | 8 | 10176 |
| 9 | 2 | 2 | 13 | 11840 |
| 10 | 2 | 2 | 10 | 13120 |
| 11 | 2 | 2 | 13 | 14784 |
| 12 | 1 | 1 | 14 | 15232 |
| 13 | 2 | 2 | 10 | 16512 |
| 14 | 1 | 1 | 14 | 16960 |
| 15 | 2 | 2 | 8 | 17984 |
| 16 | 2 | 2 | 10 | 19264 |
| 17 | 2 | 2 | 13 | — |

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

---

## Status → Icon Mapping Chain (FULLY DOCUMENTED)

The mapping chain has three stages:

```
1. Monster status fields  →  UpdateStatusIconFlags()  →  status_icon_flags bitfield (offset 0x218/0x21C)
2. status_icon_flags bits →  Renderer (FUN_022dc820)  →  SMA animation index + palette index
3. SMA animation index    →  manpu_su.sma frame data  →  rendered icon above monster sprite
```

### Stage 1: Status → Icon Bit (via lookup tables)

`UpdateStatusIconFlags` (Ghidra: `022e3a58`–`022e3d90` region) reads each status class
field from the monster struct, uses it as an index into an 8-byte-per-entry lookup table,
and ORs the results into `status_icon_flags` at monster struct offset `0x218` (cycling)
and `0x21C` (persistent/always-shown).

**Source:** Ghidra decompilation of `UpdateStatusIconFlags`. Table pointer addresses
confirmed at `022e3d94`–`022e3dcc`. Table data confirmed at `023511bc`–`02351583`.

#### Lookup Table Addresses

| Monster Offset | Status Field | Table Pointer Addr | Table Data Addr | Entries |
|----------------|-------------|--------------------|-----------------|----|
| 0xBD | `sleep_class_status.sleep` | `022e3d94` | `0235130c` | 6 |
| 0xBF | `burn_class_status.burn` | `022e3d98` | `02351294` | 5 |
| 0xC4 | `frozen_class_status.freeze` | `022e3d9c` | `02351374` | 8 |
| 0xD0 | `cringe_class_status.cringe` | `022e3da0` | `023513b4` | 8 |
| 0xD2 | `bide_class_status.bide` | `022e3da4` | `023513f4` | 14 |
| 0xD5 | `reflect_class_status.reflect` | `022e3da8` | `023514f4` | 18 |
| 0xD8 | `curse_class_status.curse` | `022e3dac` | `0235133c` | 7 |
| 0xE0 | `leech_seed_class_status.leech_seed` | `022e3db0` | `023511fc` | 3 |
| 0xEC | `sure_shot_class_status.sure_shot` | `022e3db4` | `023512bc` | 5 |
| 0xEE | `long_toss_class_status.status` | `022e3db8` | `02351214` | 3 |
| 0xEF | `invisible_class_status.status` | `022e3dbc` | `023512e4` | 5 |
| 0xF1 | `blinker_class_status.blinded` | `022e3dc0` | `0235126c` | 5 |
| 0xF3 | `muzzled` | `022e3dc4` | `023511bc` | 2 |
| 0xF5 | `miracle_eye` | `022e3dc8` | `023511ec` | 2 |
| 0xF7 | `magnet_rise` | `022e3dcc` | `023511cc` | 2 |

#### Table Contents: Sleep (0x0235130c, 6 entries)

| Index | Enum | Cycling (0x218) | Persistent (0x21C) | Icon Bit |
|-------|------|-----------------|--------------------|----|
| 0 | SLEEP_NONE | `0x00000000` | `0x00000000` | — |
| 1 | SLEEP_SLEEP | `0x04000000` | `0x00000000` | bit 26: f_sleep |
| 2 | SLEEP_SLEEPLESS | `0x00000001` | `0x00000000` | bit 0: f_sleepless |
| 3 | SLEEP_NIGHTMARE | `0x04000000` | `0x00000000` | bit 26: f_sleep |
| 4 | SLEEP_YAWNING | `0x00000000` | `0x00000000` | — |
| 5 | SLEEP_NAPPING | `0x04000000` | `0x00000000` | bit 26: f_sleep |

**Source:** ROM data at `0235130c`–`0235133b`

#### Table Contents: Burn (0x02351294, 5 entries)

| Index | Enum | Cycling (0x218) | Persistent (0x21C) | Icon Bit |
|-------|------|-----------------|--------------------|----|
| 0 | BURN_NONE | `0x00000000` | `0x00000000` | — |
| 1 | BURN_BURN | `0x00000002` | `0x00000000` | bit 1: f_burn |
| 2 | BURN_POISONED | `0x00000004` | `0x00000000` | bit 2: f_poison |
| 3 | BURN_BADLY_POISONED | `0x00000008` | `0x00000000` | bit 3: f_toxic |
| 4 | BURN_PARALYSIS | `0x00000000` | `0x00000000` | — (no icon) |

**Source:** ROM data at `02351294`–`023512bb`

#### Table Contents: Freeze (0x02351374, 8 entries)

| Index | Enum | Cycling (0x218) | Persistent (0x21C) | Icon Bit |
|-------|------|-----------------|--------------------|----|
| 0 | FROZEN_NONE | `0x00000000` | `0x00000000` | — |
| 1 | FROZEN_FROZEN | `0x00000000` | `0x00000001` | persistent bit 0: f_freeze |
| 2 | FROZEN_SHADOW_HOLD | `0x00000000` | `0x00000000` | — |
| 3 | FROZEN_WRAP | `0x00000000` | `0x00000000` | — |
| 4 | FROZEN_WRAPPED | `0x00000000` | `0x00000000` | — |
| 5 | FROZEN_INGRAIN | `0x00000000` | `0x00000000` | — |
| 6 | FROZEN_PETRIFIED | `0x00000000` | `0x00000000` | — |
| 7 | FROZEN_CONSTRICTION | `0x00000000` | `0x00000000` | — |

**Source:** ROM data at `02351374`–`023513b3`

#### Table Contents: Cringe (0x023513b4, 8 entries)

| Index | Enum | Cycling (0x218) | Persistent (0x21C) | Icon Bit |
|-------|------|-----------------|--------------------|----|
| 0 | CRINGE_NONE | `0x00000000` | `0x00000000` | — |
| 1 | CRINGE_CRINGE | `0x00000000` | `0x00000000` | — (no icon) |
| 2 | CRINGE_CONFUSED | `0x00000010` | `0x00000000` | bit 4: f_confused |
| 3 | CRINGE_PAUSED | `0x00000000` | `0x00000000` | — (no icon) |
| 4 | CRINGE_COWERING | `0x00000020` | `0x00000000` | bit 5: f_cowering |
| 5 | CRINGE_TAUNTED | `0x00000040` | `0x00000000` | bit 6: f_taunt |
| 6 | CRINGE_ENCORE | `0x00000080` | `0x00000000` | bit 7: f_encore |
| 7 | CRINGE_INFATUATED | `0x00000000` | `0x00000000` | — (no icon) |

**Source:** ROM data at `023513b4`–`023513f3`

#### Table Contents: Bide / Two-Turn (0x023513f4, 14 entries)

All 14 entries are `0x00000000` / `0x00000000`. **No two-turn move statuses have icons.**

**Source:** ROM data at `023513f4`–`02351463`

#### Table Contents: Reflect (0x023514f4, 18 entries)

| Index | Enum | Cycling (0x218) | Icon Bit |
|-------|------|-----------------|-----|
| 0 | REFLECT_NONE | `0x00000000` | — |
| 1 | REFLECT_REFLECT | `0x00000100` | bit 8: f_reflect |
| 2 | REFLECT_SAFEGUARD | `0x00000200` | bit 9: f_safeguard |
| 3 | REFLECT_LIGHT_SCREEN | `0x00000400` | bit 10: f_light_screen |
| 4 | REFLECT_COUNTER | `0x00000100` | bit 8: f_reflect |
| 5 | REFLECT_MAGIC_COAT | `0x00000400` | bit 10: f_light_screen |
| 6 | REFLECT_WISH | `0x00000000` | — |
| 7 | REFLECT_PROTECT | `0x00000800` | bit 11: f_protect |
| 8 | REFLECT_MIRROR_COAT | `0x00000200` | bit 9: f_safeguard |
| 9 | REFLECT_ENDURING | `0x00001000` | bit 12: f_endure |
| 10 | REFLECT_MINI_COUNTER | `0x00000100` | bit 8: f_reflect |
| 11 | REFLECT_MIRROR_MOVE | `0x00000800` | bit 11: f_protect |
| 12 | REFLECT_CONVERSION2 | `0x00000000` | — |
| 13 | REFLECT_VITAL_THROW | `0x00000800` | bit 11: f_protect |
| 14 | REFLECT_MIST | `0x00000100` | bit 8: f_reflect |
| 15 | REFLECT_METAL_BURST | `0x00000100` | bit 8: f_reflect |
| 16 | REFLECT_AQUA_RING | `0x00000100` | bit 8: f_reflect |
| 17 | REFLECT_LUCKY_CHANT | `0x00000100` | bit 8: f_reflect |

All persistent bits are zero.

**Source:** ROM data at `023514f4`–`02351583`

#### Table Contents: Curse (0x0235133c, 7 entries)

| Index | Enum | Cycling (0x218) | Icon Bit |
|-------|------|-----------------|-----|
| 0 | CURSE_NONE | `0x00000000` | — |
| 1 | CURSE_CURSED | `0x00004000` | bit 14: f_curse |
| 2 | CURSE_DECOY | `0x00000000` | — |
| 3 | CURSE_SNATCH | `0x00008000` | bit 15: f_embargo |
| 4 | CURSE_GASTRO_ACID | `0x00008000` | bit 15: f_embargo |
| 5 | CURSE_HEAL_BLOCK | `0x10000000` | bit 28: f_heal_block |
| 6 | CURSE_EMBARGO | `0x00008000` | bit 15: f_embargo |

All persistent bits are zero.

**Source:** ROM data at `0235133c`–`02351373`

#### Table Contents: Leech Seed (0x023511fc, 3 entries)

All 3 entries are `0x00000000` / `0x00000000`. **No leech seed class statuses have icons.**

**Source:** ROM data at `023511fc`–`02351213`

#### Table Contents: Sure Shot (0x023512bc, 5 entries)

| Index | Enum | Cycling (0x218) | Icon Bit |
|-------|------|-----------------|-----|
| 0 | SURE_SHOT_NONE | `0x00000000` | — |
| 1 | SURE_SHOT_SURE_SHOT | `0x00010000` | bit 16: f_sure_shot |
| 2 | SURE_SHOT_WHIFFER | `0x00020000` | bit 17: f_whiffer |
| 3 | SURE_SHOT_SET_DAMAGE | `0x00040000` | bit 18: f_set_damage |
| 4 | SURE_SHOT_FOCUS_ENERGY | `0x00080000` | bit 19: f_focus_energy |

All persistent bits are zero.

**Source:** ROM data at `023512bc`–`023512e3`

#### Table Contents: Long Toss (0x02351214, 3 entries)

All 3 entries are `0x00000000` / `0x00000000`. **No long toss class statuses have icons.**

**Source:** ROM data at `02351214`–`0235122b`

#### Table Contents: Invisible (0x023512e4, 5 entries)

All 5 entries are `0x00000000` / `0x00000000`. **No invisible class statuses have icons.**

**Source:** ROM data at `023512e4`–`0235130b`

#### Table Contents: Blinker (0x0235126c, 5 entries)

| Index | Enum | Cycling (0x218) | Icon Bit |
|-------|------|-----------------|-----|
| 0 | BLINKER_NONE | `0x00000000` | — |
| 1 | BLINKER_BLINKER | `0x00100000` | bit 20: f_blinded |
| 2 | BLINKER_CROSS_EYED | `0x00200000` | bit 21: f_cross_eyed |
| 3 | BLINKER_EYEDROPS | `0x00400000` | bit 22: f_eyedrops |
| 4 | BLINKER_DROPEYE | `0x00000000` | — (no icon) |

All persistent bits are zero.

**Source:** ROM data at `0235126c`–`02351293`

#### Table Contents: Muzzled (0x023511bc, 2 entries)

| Index | Enum | Cycling (0x218) | Icon Bit |
|-------|------|-----------------|-----|
| 0 | (false) | `0x00000000` | — |
| 1 | (true) | `0x00800000` | bit 23: f_muzzled |

**Source:** ROM data at `023511bc`–`023511cb`

#### Table Contents: Miracle Eye (0x023511ec, 2 entries)

| Index | Enum | Cycling (0x218) | Icon Bit |
|-------|------|-----------------|-----|
| 0 | (false) | `0x00000000` | — |
| 1 | (true) | `0x20000000` | bit 29: f_miracle_eye |

**Source:** ROM data at `023511ec`–`023511fb`

#### Table Contents: Magnet Rise (0x023511cc, 2 entries)

| Index | Enum | Cycling (0x218) | Icon Bit |
|-------|------|-----------------|-----|
| 0 | (false) | `0x00000000` | — |
| 1 | (true) | `0x80000000` | bit 31: f_magnet_rise |

**Source:** ROM data at `023511cc`–`023511db`

#### Hardcoded Bits (not from tables)

These are set by explicit checks in `UpdateStatusIconFlags`, not via the lookup tables:

| Bit | Flag | Condition | Source |
|-----|------|-----------|--------|
| 13 | f_low_hp | `hp < max_hp / 4` (team members only) | Ghidra: `022e3d00`–`022e3d18` |
| 13 | f_low_hp | `identify_orb_flag && has_held_item` | Ghidra: `022e3d18`–`022e3d28` |
| 24 | f_grudge | `monster.grudge != 0` (offset 0xFD) | Ghidra: `022e3cc0`–`022e3ccc` |
| 25 | f_exposed | `monster.exposed != 0` (offset 0xFE) | Ghidra: `022e3ccc`–`022e3cd8` |
| 27 | f_lowered_stat | Any stat multiplier < 0x100 OR any stat stage < 10 | Ghidra: `022e3d28`–`022e3d80` |

---

### Stage 2: Icon Bit → SMA Animation Index + Palette

The renderer `FUN_022dc820` (Ghidra: `022dc820`–`022dd088`) iterates set bits in
`status_icon_flags`, and for each set bit calls `FUN_022dc7e4` which returns
`bit_position + 1`. This value is used as an index into a table at `02350f8c`.
Each entry is 8 bytes: `[u32 sma_animation_index, u32 palette_index]`.
Entry 0 in the table is unused padding — bit N reads table[N+1].

The SMA animation data is accessed via the structure pointer at `DAT_02353518`, offset `+0x6f4`.
Each animation entry in the SMA header is 12 bytes (`0xC`), indexed by the animation index value.

**Source:** ROM data at `02350f8c`–`0235109b`. Renderer disassembly at `022dcb34`–`022dcb60`.

#### Icon Bit → SMA Rendering Table (32 cycling entries)

| Bit | Table Index | Flag | SMA Anim | Palette | Visual Description |
|-----|------|----------|---------|-------------------|
| 0 | 1 | f_sleepless | 1 | 0 | Tiny eye (8×8) |
| 1 | 2 | f_burn | 2 | 0 | Red flame |
| 2 | 3 | f_poison | 3 | 11 | White skull |
| 3 | 4 | f_toxic | 3 | 7 | Purple skull |
| 4 | 5 | f_confused | 5 | 0 | Yellow birds |
| 5 | 6 | f_cowering | 6 | 0 | Circular swirls |
| 6 | 7 | f_taunt | 7 | 0 | Fist icon |
| 7 | 8 | f_encore | 8 | 0 | Vertical exclamation shape |
| 8 | 9 | f_reflect | 9 | 0 | Shield |
| 9 | 10 | f_safeguard | 9 | 4 | Shield (palette 4) |
| 10 | 11 | f_light_screen | 9 | 3 | Shield (palette 3) |
| 11 | 12 | f_protect | 9 | 10 | Shield (palette 10) |
| 12 | 13 | f_endure | 9 | 5 | Shield (palette 5) |
| 13 | 14 | f_low_hp | 8 | 0 | Vertical exclamation shape (same as encore) |
| 14 | 15 | f_curse | 3 | 6 | Skull (palette 6) |
| 15 | 16 | f_embargo | 8 | 3 | Vertical shape (palette 3) |
| 16 | 17 | f_sure_shot | 11 | 0 | Sword |
| 17 | 18 | f_whiffer | 6 | 10 | Circular swirls (palette 10) |
| 18 | 19 | f_set_damage | 11 | 5 | Sword (palette 5) |
| 19 | 20 | f_focus_energy | 11 | 4 | Sword (palette 4) |
| 20 | 21 | f_blinded | 12 | 0 | Tiny blinking (8×8) |
| 21 | 22 | f_cross_eyed | 13 | 0 | Question marks |
| 22 | 23 | f_eyedrops | 14 | 0 | Small blinking (8×8) |
| 23 | 24 | f_muzzled | 15 | 0 | Cross/X shapes |
| 24 | 25 | f_grudge | 9 | 7 | Shield (palette 7) |
| 25 | 26 | f_exposed | 14 | 4 | Small blinking (palette 4) |
| 26 | 27 | f_sleep | 16 | 4 | Z's |
| 27 | 28 | f_lowered_stat | 10 | 3 | Down arrows (palette 3) |
| 28 | 29 | f_heal_block | 15 | 3 | Cross/X shapes (palette 3) |
| 29 | 30 | f_miracle_eye | 15 | 4 | Cross/X shapes (palette 4) |
| 30 | 31 | f_red_exclamation | 8 | 4 | Vertical shape (palette 4) — probably unused |
| 31 | 32 | f_magnet_rise | 10 | 7 | Down arrows (palette 7) |

**Source:** ROM data at `02350f8c`–`0235108b` (32 entries × 8 bytes)

#### Additional Entries (persistent icons, after the 32 cycling entries)

| Table Index | Offset | SMA Anim | Palette | Purpose |
|-------------|--------|----------|---------|---------|
| 32 | `0235108c` | 10 | 7 | Unknown — down arrows palette 7 |
| 33 | `02351094` | 4 | 0 | **Freeze ice block (confirmed)** — 4×4 (32×32px), 6 frames, rendered persistently over frozen monster |

**Source:** ROM data at `0235108c`–`0235109b`. Freeze confirmed visually from extracted sprite sheet.

#### SMA Animation Reuse Summary

Many icons share the same SMA animation shape but use different palettes for color variation:

| SMA Anim | Shape | Used By (bit: palette) |
|----------|-------|----------------------|
| 3 | 2×2, 16fr skull shape | poison(2:11), toxic(3:7), curse(14:6) |
| 6 | 2×2, 9fr swirl shape | cowering(5:0), whiffer(17:10) |
| 8 | 1×2, 8fr vertical shape | encore(7:0), low_hp(13:0), embargo(15:3), red_exclamation(30:4) |
| 9 | 2×2, 13fr shield shape | reflect(8:0), safeguard(9:4), light_screen(10:3), protect(11:10), endure(12:5), grudge(24:7) |
| 10 | 2×2, 10fr arrow shape | lowered_stat(27:3), magnet_rise(31:7) |
| 11 | 2×2, 13fr sword shape | sure_shot(16:0), set_damage(18:5), focus_energy(19:4) |
| 14 | 1×1, 14fr small blinking | eyedrops(22:0), exposed(25:4) |
| 15 | 2×2, 8fr cross/X shape | muzzled(23:0), heal_block(28:3), miracle_eye(29:4) |
| 16 | 2×2, 10fr Z shape | sleep(26:4) |

---

### Stage 3: SMA Loader

`FUN_022dd5b4` (Ghidra: `022dd5b4`) loads `SYSTEM/manpu_su.sma` into memory.

- String reference: `"rom0:SYSTEM/manpu_su.sma"` at `0235109c`
- Data structure pointer: `DAT_02353518`
- SMA data stored at offset `+0x6f4` within the structure
- 16 palettes loaded in a loop (indices 0–15)
- `FUN_022dd518` called for animation indices 1–16 (sets up each animation's rendering data)

**Source:** Ghidra decompilation of `FUN_022dd5b4`. String XREF at `022dd5ec`.

---
### Stage 4: Rendering & Positioning

#### Icon Rendering State Struct (0x50 bytes per monster)

22 (0x16) slots are allocated at `*DAT_022dda50 + 4`, one per visible dungeon entity.
Each slot is 0x50 bytes. The base pointer is also accessible via `DAT_022dd08c` and `DAT_022dc7e0`.

`FUN_022dc78c` looks up a slot by matching `unique_id` at offset +0x08.

| Offset | Size | Writer | Content |
|--------|------|--------|---------|
| +0x00 | 1 | — | Active flag (nonzero = slot in use) |
| +0x08 | 2 | FUN_022dd7d8 | `apparent_id` (species sprite index) |
| +0x0C | 4 | FUN_022dd7d8 | `status_icon_flags` low (cycling bitfield, from monster +0x218) |
| +0x10 | 4 | FUN_022dd7d8 | `status_icon_flags` high (persistent bitfield, from monster +0x21C) |
| +0x14 | 1 | FUN_022dd7d8 | Visibility flag |
| +0x15 | 1 | FUN_022ddb98 | Display mode flag |
| +0x16 | 2 | FUN_022ddb98 | Entity pixel_x >> 8 |
| +0x18 | 2 | FUN_022ddb98 | Entity pixel_y >> 8 (elevation-adjusted) |
| +0x1A | 2 | FUN_022ddb98 | Head attachment X (sprite-relative, per-frame) |
| +0x1C | 2 | FUN_022ddb98 | Head attachment Y (sprite-relative, per-frame) |
| +0x1E | 2 | FUN_022ddb98 | Centre attachment X (sprite-relative, per-frame) |
| +0x20 | 2 | FUN_022ddb98 | Centre attachment Y (sprite-relative, per-frame) |
| +0x2C | 12 | FUN_022dc820 | Cycling icon state: {bits_lo(4), bits_hi(4), countdown_timer(4)} |
| +0x38 | 12 | FUN_022dc820 | Persistent icon state: same layout, param_2=1 |
| +0x44 | 1 | FUN_022dd7d8 / FUN_022dd8b4 | Dirty flag (cleared after render) |
| +0x46 | 2 | FUN_02303f18 | Screen offset X |
| +0x48 | 2 | FUN_02303f18 | Screen offset Y |
| +0x4A | 2 | FUN_02303f18 | Z-priority related |

#### Per-Frame Data Flow
```
FUN_02303f18 (per entity, each frame)
  │
  ├── Computes pixel_pos, screen offsets, animation state
  │
  ├── FUN_0201d034(asStack_30, 4, anim_ctrl)
  │     Reads all 4 WAN attachment points for current frame:
  │       [0] = Head.x/y    [1] = LeftHand.x/y
  │       [2] = RightHand.x/y   [3] = Centre.x/y
  │
  ├── FUN_022ddb98(unique_id, &pixel_pos, asStack_30, flag)
  │     Writes pixel position + Head + Centre into icon struct
  │
  ├── FUN_022e3a40(&local_48, entity)
  │     Copies monster->info+0x218/0x21C into local vars
  │
  └── FUN_022dd7d8(unique_id, apparent_id, flags_lo, flags_hi, visible)
        Writes status flags + visibility into icon struct
```
```
FUN_022dd8b4 (renderer, each frame — called via function pointer)
  │
  ├── FUN_022dd518 × 4: SMA palette animation tick (4 anims per quarter-frame)
  │
  └── For each of 22 slots:
        FUN_022dc820(slot, 0, camera)   → cycling icons
        FUN_022dc820(slot, 1, camera)   → persistent icons
```

#### Icon Positioning Formula

The position is computed in `FUN_022dd0a4`, called from within `FUN_022dc820`.

**Cycling icons** (all statuses except freeze) use the **Head** attachment point:
```
screen_x = pixel_x + head_x - (sma_width_blocks * 4) - camera_x
screen_y = pixel_y + head_y - (sma_height_blocks * 4) - camera_y - 16
```

**Freeze icon** (persistent, SMA anim 4) uses the **Centre** attachment point:
```
screen_x = pixel_x + centre_x - (sma_width_blocks * 4) - camera_x
screen_y = pixel_y + centre_y - (sma_height_blocks * 4) - camera_y
```

Where:
- `pixel_x/y` = entity pixel position >> 8, with Y adjusted for elevation
- `head_x/y` and `centre_x/y` = WAN attachment point offsets for the current animation frame (includes sprite origin offset from `anim_ctrl[0x10]/[0x11]`)
- `sma_width_blocks` / `sma_height_blocks` = from the SMA animation header (1 block = 8 pixels)
- The `- blocks * 4` centers the icon on the attachment point
- The `-16` pushes cycling icons above the head

**Bounds clipping:** Icons are culled if `screen_x < -32` or `screen_x >= 255` or the combined Y is out of range `[0, 192)` (NDS screen bounds with margin).

#### Icon Cycling Behavior

**Per-monster, not synchronized.** Each monster has independent cycling state in its 0x50-byte slot at offsets +0x2C (cycling) and +0x38 (persistent).

**Timer:** Each active icon displays for **60 frames** (0x3C). At ~60fps this is approximately **1 second per icon**. When the timer reaches 0, the renderer scans forward through the `status_icon_flags` bitfield to find the next set bit, wrapping around if needed, and resets the timer to 60.

**Bit rotation:** The current icon is stored as a single-bit mask (bits_lo/bits_hi). On each cycle, it shifts left by 1. If it overflows past the highest valid bit, it wraps to bit 0 (cycling) or bit 0 of the persistent field. The first set bit found after rotation becomes the next displayed icon.

**If all icon bits are cleared** (status cured), both bits_lo and bits_hi are set to 0 and the timer is reset to 0. The icon disappears immediately.

**Persistent icons** (freeze) do not participate in cycling rotation. They are always displayed via the `param_2=1` path.

#### SMA Animation Header Access

The SMA data is at `*(DAT_022dd08c + 0x6F4)`. Each animation entry is **12 bytes** (0xC), accessed as:
```
entry = *(sma_data + 4) + anim_index * 0xC
  byte  [0]   = width_blocks
  byte  [1]   = height_blocks
  int32 [4]   = image data offset
  int16 [8]   = frame count (used for palette animation division)
  int16 [10]  = palette base index
```

#### Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_022dd8b4` | `0x022dd8b4` | Top-level icon renderer (called via function pointer) |
| `FUN_022dc820` | `0x022dc820` | Per-slot icon tick + render |
| `FUN_022dd0a4` | `0x022dd0a4` | Icon OAM write (computes final screen position) |
| `FUN_022dc7e4` | `0x022dc7e4` | Bit position finder (returns highest set bit index + 1) |
| `FUN_022dc78c` | `0x022dc78c` | Slot lookup by unique_id |
| `FUN_022ddb98` | `0x022ddb98` | Writes pixel position + attachment points into icon struct |
| `FUN_022dd7d8` | `0x022dd7d8` | Writes status flags + visibility into icon struct |
| `FUN_022e3a40` | `0x022e3a40` | Copies status_icon_flags from monster struct |
| `FUN_0201d034` | `0x0201d034` | Reads all 4 WAN attachment points for current frame |
| `FUN_022dd518` | `0x022dd518` | SMA palette animation setup per animation index |
| `FUN_02303f18` | `0x02303f18` | Entity per-frame update (animation, position, icon data) |


---
## status_icon_flags Bitfield Reference

From `headers/types/dungeon_mode/dungeon_mode.h` in pmdsky-debug:

```c
struct status_icon_flags {
    // Cycling icons (offset 0x218, first 4 bytes)
    bool8 f_sleepless : 1;         // bit 0  - Blue eye blinking yellow
    bool8 f_burn : 1;              // bit 1  - Red flame
    bool8 f_poison : 1;            // bit 2  - White skull
    bool8 f_toxic : 1;             // bit 3  - Purple skull
    bool8 f_confused : 1;          // bit 4  - Yellow birds
    bool8 f_cowering : 1;          // bit 5  - 2 green lines in circle
    bool8 f_taunt : 1;             // bit 6  - Fist icon
    bool8 f_encore : 1;            // bit 7  - Blue exclamation mark
    bool8 f_reflect : 1;           // bit 8  - Blue shield with white sparks
    bool8 f_safeguard : 1;         // bit 9  - Pink shield
    bool8 f_light_screen : 1;      // bit 10 - Golden shield
    bool8 f_protect : 1;           // bit 11 - Green shield
    bool8 f_endure : 1;            // bit 12 - Blue shield with red sparks
    bool8 f_low_hp : 1;            // bit 13 - Blue exclamation mark
    bool8 f_curse : 1;             // bit 14 - Red skull
    bool8 f_embargo : 1;           // bit 15 - Yellow exclamation mark
    bool8 f_sure_shot : 1;         // bit 16 - Blue sword blinking yellow
    bool8 f_whiffer : 1;           // bit 17 - 2 green lines in circle
    bool8 f_set_damage : 1;        // bit 18 - Blue sword blinking red
    bool8 f_focus_energy : 1;      // bit 19 - Red sword blinking yellow
    bool8 f_blinded : 1;           // bit 20 - Blue eye with an X
    bool8 f_cross_eyed : 1;        // bit 21 - Blue question mark
    bool8 f_eyedrops : 1;          // bit 22 - Blue eye with circular wave
    bool8 f_muzzled : 1;           // bit 23 - Blinking red cross
    bool8 f_grudge : 1;            // bit 24 - Purple shield
    bool8 f_exposed : 1;           // bit 25 - Blue eye blinking red with wave
    bool8 f_sleep : 1;             // bit 26 - Red Z's
    bool8 f_lowered_stat : 1;      // bit 27 - Yellow arrow pointing down
    bool8 f_heal_block : 1;        // bit 28 - Blinking green cross
    bool8 f_miracle_eye : 1;       // bit 29 - Blinking orange cross
    bool8 f_red_exclamation : 1;   // bit 30 - Probably unused
    bool8 f_magnet_rise : 1;       // bit 31 - Purple arrow pointing up

    // Persistent icons (offset 0x21C, second 4 bytes)
    bool8 f_freeze : 1;            // bit 0  - Ice block (always shown, not cycling)
    // remaining bits appear unused
};
```

**Source:** pmdsky-debug `headers/types/dungeon_mode/dungeon_mode.h`, struct `status_icon_flags`

---

## Open Questions

- What is `manpu_ma.sma`? No image data, purpose unknown.
- The persistent freeze icon rendering path (`022dcfac` region) may use a slightly different
  lookup mechanism — needs confirmation of which SMA anim/palette the freeze icon uses at runtime.
- Confirmed: No raised-stat icon exists. UpdateStatusIconFlags only checks for stats below default (multiplier < 0x100 or stage < 10). There is no corresponding check for stats above default. Stat buffs (Swords Dance, Calm Mind, etc.) produce only a one-shot effect.bin animation with no persistent overhead icon. Verified from UpdateStatusIconFlags decompilation at 022e3d28–022e3d80.
- The `f_sleepless` icon (bit 0) maps to SMA anim 1 (tiny 8×8 eye) via table[1]. Previously
  thought to be null — resolved by the bit+1 indexing discovery.

## Breadcrumbs

- `FUN_022dc820` — main status icon renderer, contains the cycling icon draw loop
- `FUN_022dd5b4` — SMA file loader
- `FUN_022dd518` — per-animation setup function called during load
- `DAT_02353518` — pointer to status icon data structure at runtime
- `02350f8c` — icon bit → SMA animation/palette lookup table (32×8 bytes)
- `023511bc`–`02351583` — status → icon bit lookup tables (15 tables)
- `overlay_0029.bin` — all of the above code lives in overlay 29
- Two-turn move animations — separate system via `bide_class_status` + `two_turn_move_invincible`
  flag at monster offset 0x10B (graphical flag, not icon-based). Documented for future research.
