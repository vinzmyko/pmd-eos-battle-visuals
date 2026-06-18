# Weather & Tileset 3D Rendering (`RenderWeather3D`)

## Summary

- Covers the NDS 3D-engine overlay that draws **fog**, **sandstorm**, and the per-tileset **mist/steam** as a scrolling translucent texture on top of the 2D dungeon.
- Not a particle system: each effect is a single static WTE texture (128×128, 4bpp) tiled in a 3×3 grid and slid a fraction of a pixel per frame. Hardware texture-repeat makes the drift seamless.
- Two render "slots" share one loop in `RenderWeather3D`: one for the tileset mist, one for the weather (fog/sandstorm).
- Scroll direction/speed come from a 10-entry mode table: mode 0 = off, modes 1–8 = eight compass directions at a fixed step, mode 9 = (inert) sine drift.
- Alpha is a steady ~25% with a short fade-in when the overlay (re)appears.
- Textures are `dungeon.bin` WTE files (1001 fog / 1005 sandstorm / 1031 mist / 1003 poison-mist), loaded once per dungeon and copied raw into texture VRAM.
- This is one of four weather outputs; the precipitation sprites, ambient sound, and colormap tint live in `weather_visual_pipeline.md`.

## Context: the four weather outputs

A floor weather can produce up to four independent visual/audio outputs. This document covers only the 3D scrolling overlay; the others are documented in `weather_visual_pipeline.md`:

1. **Precipitation sprites** — effect.bin WAN animations on the leader (rain 16/440, hail 20, snow 223, sandstorm 239, sunny 331). 2D sprites.
2. **Ambient sound** — a per-weather table where only **Cloudy (`0x311`)** and **Fog (`0x312`)** have a dedicated ambient loop; every other weather's audio rides its precipitation animation's own `sfx_id`.
3. **Colormap tint** — a full-screen palette remap from `dungeon.bin[1034]` (see `weather_visual_pipeline.md`, Stage 2).
4. **3D scrolling overlay** — *this document*.

Among the weathers, only **fog** and **sandstorm** use the 3D overlay. **Fog has no precipitation sprite at all**, so for fog the overlay + tint is the entire visual. **Sandstorm** layers the overlay beneath its sprite (239). The tileset **mist/steam** is a separate use of the same renderer, keyed by tileset rather than weather.

## Two-slot model

`RenderWeather3D` (NA `0x02338AC4`) iterates **two slots** from a contiguous array at `DAT_02338d28`, stride `0x274`, every frame. They run the identical loop and differ only in configured mode and bound texture:

| Slot | Purpose | Driven by | Init binding idx |
|------|---------|-----------|------------------|
| Tileset | Mist / steam | `FUN_023389c4` ← tileset `weather_effect` | 0 |
| Weather | Fog / sandstorm | `FUN_02338a4c` ← `weather_id` | 1 |

## Slot layout

Each slot is `0x274` bytes. The first `0x240` bytes are the 3×3 quad grid; the tail holds animation state.

| Offset | Type | Field |
|--------|------|-------|
| 0x000 | 9 × `render_3d_element_64` | 3×3 quad grid (3 rows × `0xC0`, each row 3 quads × `0x40`) |
| 0x240 | u8 | render mode (0 = inactive, 1–8 = linear scroll direction, 9 = sine) |
| 0x244 | int | alpha accumulator (`0x4000`–`0xC000`; rendered as `>>8`) |
| 0x248 | int | alpha step (init **0** → no oscillation) |
| 0x24C | int | scroll X accumulator |
| 0x250 | int | scroll Y accumulator |
| 0x254 | int | scroll X step (from the mode table) |
| 0x258 | int | scroll Y step |
| 0x25C | int | sine phase X (mode 9) |
| 0x260 | int | sine phase Y (mode 9) |
| 0x264 | int | sine phase X increment (= 4) |
| 0x268 | int | sine phase Y increment (= 1) |
| 0x26C | u8 | alpha ramp direction flag |
| 0x26D | u8 | enabled/visible flag (render gate) |
| 0x26E | u8 | fade-in active flag |
| 0x270 | int | bound texture/UV descriptor index |

Initialised by `FUN_02338d94` (NA `0x02338D94`): alpha `0x4000`, random sine phases via `DungeonRandInt(0x400)`, and it builds the 3×3 grid.

## Render modes & scroll-step table

`FUN_02338d34(slot, mode)` (NA `0x02338D34`) writes the mode to `+0x240` and loads the X/Y step from the mode table `DAT_02338d54` (→ `0x02352F7C`, stride 8: X at `+0`, Y at `+4`):

| Mode | X step | Y step | Drift |
|------|--------|--------|-------|
| 0 | 0 | 0 | off |
| 1 | 0 | +0x60 | South |
| 2 | +0x60 | +0x60 | South-East |
| 3 | +0x60 | 0 | **East** — fog / sandstorm |
| 4 | +0x60 | −0x60 | North-East |
| 5 | 0 | −0x60 | North |
| 6 | −0x60 | −0x60 | North-West |
| 7 | −0x60 | 0 | West |
| 8 | −0x60 | +0x60 | South-West |
| 9 | 0 | 0 | sine (zero amplitude → no motion) |

Modes 1–8 are the eight compass directions at a fixed magnitude `0x60`. Mode 9 selects the sine path but its step is `(0,0)` in shipped data, so it produces no translation; whether any tileset selects it is unconfirmed.

## Per-frame update (`RenderWeather3D`)

For each of the two slots, when `+0x240` (mode) is non-zero:

1. **Visibility gate.** Call `FUN_022e3580()` (a per-frame suppression check). If it returns 0, call `FUN_02338d6c(slot, 1)` — enables the slot and, if it had been disabled, starts a fade-in. Otherwise set `+0x26D = 0` (disable). `FUN_02338d58` then returns 0 if mode == 0 or `+0x26D == 0`, skipping the slot.

2. **Draw the grid.** With `ox = scrollX >> 8`, `oy = scrollY >> 8`:
   - quad (row, col) top-left = `(col*0x80 + ox, row*0x80 + oy)`, size `0x80`×`0x80` (128 px)
   - cull any quad whose top edge `>= 0xC0` (192 = screen height)
   - per-quad alpha = `(+0x244) >> 8`
   - submit via `Render3dElement64`

3. **Advance scroll.**
   - **Linear (modes 1–8):** `scrollX += step.x`, `scrollY += step.y`.
   - **Sine (mode 9):** `scrollX += MultiplyByFixedPoint(step.x*10, SinAbs4096(phaseX))` (same for Y); `phaseX += 4`, `phaseY += 1`, then both `&= DAT_02338d2c` (`0x0FFF`).

4. **Wrap accumulators** to one tile: if `accum >= 1` subtract `0x8000`; if `accum < -0x8000` add `0x8000`. `0x8000 >> 8 = 0x80` = one 128 px tile, so the visible offset cycles seamlessly.

5. **Alpha.** Fade-in branch (`+0x26E != 0`): `alpha += 0x400`/frame until it reaches `0x4000`, then clear `+0x26E`. Steady branch otherwise: `alpha ±= +0x248` clamped to `[0x4000, 0xC000]` — but `+0x248` is 0, so alpha holds at `0x4000` (rendered `0x40`, ~25%).

**Speed:** step `0x60` = `0x60/256` ≈ 0.375 px/frame; one full 128 px tile-cycle ≈ `0x8000/0x60` ≈ 341 frames — a slow ambient drift.

**Evidence:** `RenderWeather3D` — scroll advance, wrap, and fade-in (condensed; linear branch shown):
```c
*(int *)(slot + 0x24c) += *(int *)(slot + 0x254);   // scrollX += stepX
*(int *)(slot + 0x250) += *(int *)(slot + 0x258);   // scrollY += stepY

if (scrollX >= 1)            scrollX -= 0x8000;      // wrap to one-tile range
else if (scrollX < -0x8000)  scrollX += 0x8000;

if (slot[0x26e]) {                                   // fading in
    alpha += 0x400;
    if (alpha >= 0x4000) { alpha = 0x4000; slot[0x26e] = 0; }
}
```

## WTE textures

The overlay textures are `dungeon.bin` WTE files (SIR0-wrapped, **16-colour 4bpp, 128×128**), decodable directly by SkyTemple's `Wte` handler (`to_pil`) — no manual A3I5/A5I3 decoder needed.

| dungeon.bin index | Use | Tex VRAM | Palette VRAM |
|-------------------|-----|----------|--------------|
| 1001 | Fog (weather) | `0xF000` | `0x1600` |
| 1005 | Sandstorm (weather) | `0xD000` | `0x1500` |
| 1031 | Mist/steam (tileset) | `0xB000` | `0x1400` |
| 1003 | Poison mist (tileset 195 only) | `0xB000` | `0x1400` |

`LoadWeather3DFiles` (NA `0x023388B0`) loads the first three once per dungeon; 1003 is swapped in over the mist VRAM only for tileset 195, by `FUN_023389c4`. WTU files are **not** used here (the UV-atlas companion is only for glyph sheets like damage numbers, files 1007–1011).

### Load descriptor table

`LoadWeather3DFiles` reads a stride-`0x14` descriptor array (`DAT_02352f54`):

| Field | Meaning |
|-------|---------|
| +0x00 | u16 dungeon.bin file index |
| +0x04 | texture VRAM offset |
| +0x08 | palette VRAM offset ÷ `0x100` |
| +0x0C, +0x10 | polygon params consumed by `FUN_02338e50` |

Decoded entries: `1005 → 0xD000 / 0x15`, `1001 → 0xF000 / 0x16`, `1031 → 0xB000 / 0x14`.

## Loading & VRAM upload

`LoadWteFromFileDirectory` pulls the SIR0 file from `PACK_ARCHIVE_DUNGEON` and resolves its header (`HandleSir0Translation`). `ProcessWte` then queues the texture and palette into VRAM:

- WTE in-memory header: `+0x00` magic `"WTE\0"`, `+0x04` texture ptr, `+0x08` texture size, `+0x18` palette ptr, `+0x1C` palette colour count.
- The texture is **copied raw** into texture VRAM through a deferred (VBlank) copy queue; the palette is converted (`Rgb8ToRgb5`) into palette VRAM. The pixel **format is not interpreted at upload** — it is set per-polygon in the NDS texture registers at draw time, which is why the WTE `image_type` (here `COLOR_4BPP`) matters only to the decoder, not to `ProcessWte`.

## Per-quad binding (`FUN_02338e50`)

`FUN_02338e50(quad, index)` (NA `0x02338E50`) binds each of the 9 quads to the texture and a fixed UV rect:

- UV origin `(0,0)`, size `0x80`×`0x80` — the full 128×128 texture, relying on hardware texture-repeat across the grid.
- Texture VRAM address and palette base come from a stride-`0x14` descriptor (`DAT_02338f08`) indexed by the binding index.
- Polygon attribute bits at quad `+0x14` set the standard textured-quad mode.

The binding index is the slot's `+0x270` value: `0` for the tileset slot, `1` for the weather slot at init, then overridden by `FUN_02338a4c` to the weather config's index field.

## Triggers

Both slots are configured per-floor from `RunDungeon`, tileset first:

```c
FUN_023389c4(tileset_id);   // tileset mist slot
FUN_02338a4c(weather_id);   // weather slot
```

**Weather slot — `FUN_02338a4c` (NA `0x02338A4C`).** Indexes the per-weather 3D config table `DAT_02338ab4` (→ `0x02353730`, stride 8) by `weather_id`. Byte `+0x00` = render mode (**3 for sandstorm (2) and fog (6), 0 for all other weathers**); int `+0x04` = the `FUN_02338e50` binding index. So only fog and sandstorm activate the weather slot, both drifting East (mode 3).

**Tileset slot — `FUN_023389c4` (NA `0x023389C4`).** Reads the tileset's `weather_effect` byte (`DAT_02338a3c = 0x022C6326` = `TILESET_PROPERTIES + 0x0A`) and passes it straight through as the mist slot's render mode (0 = none, 1–6 = drift direction). For tileset **195** only, it re-loads `dungeon.bin[1003]` (`DAT_02338a48 = 0x3EB`) over the mist VRAM (`0xB000`, pal `0x1400`). This path is independent of the `weather_id` system.

## Client reimplementation notes

- Draw one 128×128 texture, repeat-tiled, across a grid covering the screen; offset the whole grid by an accumulator that advances `step` per frame and wraps over a 128 px tile. Fog/sandstorm = horizontal (East) drift; tileset mist = the `weather_effect` direction.
- Alpha ≈ 25% (`0x40`/`0xFF`), with a ~16-frame fade-in when the overlay appears. The breathing and sine paths are inert in shipped data — a linear scroll + constant alpha reproduces the game exactly.
- Textures decode as ordinary indexed PNGs via SkyTemple; export 1001/1005/1031 (and 1003) the same way as tilesets.
- Composite the overlay translucently over the 2D dungeon; for sandstorm it sits under the precipitation sprite (239).

## Open Questions

- `DAT_02338f08` (the `FUN_02338e50` binding-descriptor table) was not dumped. Fog and sandstorm both pass binding index 0 yet use different VRAM textures, so this table is likely distinct from the `DAT_02352f54` load descriptor — to be confirmed.
- Whether any shipped tileset selects mode 9 (sine). If none do, the sine path is fully dead code in practice.
- Exact behaviour of `FUN_022e3580` (the per-frame suppression check — an indirect call to `0x022BF8E8`).

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `RenderWeather3D` | `0x02338AC4` | Per-frame render of both 3D slots |
| `FUN_02338a4c` | `0x02338A4C` | Configure weather slot from `weather_id` |
| `FUN_023389c4` | `0x023389C4` | Configure tileset mist slot from `weather_effect`; 1003 swap |
| `FUN_02338d34` | `0x02338D34` | Set slot mode + load X/Y scroll step |
| `FUN_02338d58` | `0x02338D58` | Render gate (mode != 0 && enabled) |
| `FUN_02338d6c` | `0x02338D6C` | Enable slot / trigger fade-in |
| `FUN_02338d94` | `0x02338D94` | Init slot (alpha, sine phase, 3×3 grid) |
| `FUN_02338e50` | `0x02338E50` | Per-quad texture/UV binding |
| `LoadWeather3DFiles` | `0x023388B0` | Load WTE 1001/1005/1031 into VRAM |
| `LoadWteFromFileDirectory` | — | Pull + SIR0-resolve a WTE from a pack |
| `ProcessWte` | — | Queue WTE texture + palette into VRAM |
| `FUN_022e3580` | `0x022E3580` | Per-frame visibility/suppression check |
| `Render3dElement64` | — | Submit a textured quad |
| `SinAbs4096` | — | Sine LUT (mode 9) |

## Memory Locations & Data Tables (NA)

| Symbol | Address / Target | Notes |
|--------|------------------|-------|
| Slot array base | `DAT_02338d28` | 2 slots, stride `0x274` |
| Scroll-step table | `DAT_02338d54` → `0x02352F7C` | 10 entries, stride 8 |
| Sine phase mask | `DAT_02338d2c` | `0x0FFF` |
| Per-weather 3D config | `DAT_02338ab4` → `0x02353730` | stride 8; byte = mode, int = binding idx |
| WTE load descriptor | `DAT_02352f54` | stride `0x14` |
| Tileset mist field ptr | `DAT_02338a3c` | `0x022C6326` = `TILESET_PROPERTIES + 0x0A` (`weather_effect`) |
| Tileset 195 swap file | `DAT_02338a48` | `0x3EB` = 1003 |

## Cross-References

> See `weather_visual_pipeline.md` for the overall weather pipeline — sources, the colormap tint via `dungeon.bin[1034]`, the precipitation sprites, and the ambient sound table (Cloudy `0x311` / Fog `0x312`).

> See `tileset-properties.md` for the `weather_effect` field that selects the tileset mist drift direction.

> See `Data Structures/effect_animation_info.md` for the precipitation-sprite effect IDs (note: fog and the tileset mist have no sprite).
