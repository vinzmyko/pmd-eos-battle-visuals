# Weather Visual & Audio Pipeline (Dungeon Mode)

## Summary

- Covers the complete visual and audio pipeline for floor weather in PMD:EOS dungeon mode
- Every weather source (moves, abilities, orbs, natural floor weather) sets a turn counter, then funnels through `TryActivateWeather`
- When the dominant weather changes, `TryActivateWeather` calls one master apply function (`FUN_02334e70`) that performs all visual/audio work
- A weather produces up to four outputs: a per-weather effect.bin animation, an ambient sound, an animated colormap (palette) tint, and — for Fog/Sandstorm only — a scrolling 3D WTE texture
- Effect animations and sounds are driven by per-weather lookup tables; the colormap is a 256-entry RGBA table per weather; the 3D textures live in dungeon.bin
- A separate per-tileset path (`FUN_023389c4`, called from `RunDungeon`) drives an ambient mist/steam 3D overlay keyed by tileset rather than weather — see "Tileset 3D Effects" below
- This document is considered complete: the full picture (which weathers use which outputs, and the exact asset IDs) has been recovered

## Memory Locations

| Symbol / Purpose | Region | Address |
|------------------|--------|---------|
| `GetApparentWeather` | NA | `0x2334D08` |
| `TryActivateWeather` | NA | `0x2334Fxx` (overlay 29) |
| `FUN_02334e70` (apply weather visuals — unnamed) | NA | `0x02334E70` |
| `FUN_022e5fe8` (weather sound + effect anim — unnamed) | NA | `0x022E5FE8` |
| `FUN_02338a4c` (3D weather config — unnamed) | NA | `0x02338A4C` |
| `FUN_02335044` (weather message + form change — unnamed) | NA | `0x02335044` |
| `FUN_022de608` (colormap flush — unnamed) | NA | `0x022DE608` |
| `GetWeatherColorTable` | EU / NA / JP | `0x22DEF60` / `0x22DE620` / `0x22DFCC0` |
| `LoadWeather3DFiles` | EU / NA / JP | `0x2339480` / `0x23388B0` / `0x2339C74` |
| `RenderWeather3D` | EU / NA / JP | `0x2339694` / `0x2338AC4` / `0x2339E88` |
| `DUNGEON_COLORMAP_PTR` (render-side handle) | EU / NA | `0x21BA634` / `0x21B9CF4` |
| `WEATHER_MOVE_TURN_COUNT` (= 3000) | overlay 10 | data |

### Per-Weather Data Tables (NA)

These tables are **not named in pmdsky-debug**; addresses are the dereferenced pointer targets read inside the listed functions. All indexed by `weather_id` (0–7).

| Table | Pointer (NA) | Target (NA) | Layout | Read in |
|-------|--------------|-------------|--------|---------|
| Weather ambient sound | `DAT_022e60d8` | `0x023511DC` | `u16[8]`, index `× 2` (`0x3F00` = silence) | `FUN_022e5fe8` |
| Weather effect (announced change) | `DAT_022e60dc` | `0x0235122C` | `s32[8]`, index `× 4` (`-1` = none) | `FUN_022e5fe8` |
| Weather effect (floor-entry / boss) | `DAT_022e60e0` | `0x0235124C` | `s32[8]`, index `× 4` (`-1` = none) | `FUN_022e5fe8` |
| 3D weather config | `DAT_02338ab4` | `0x02353730` | stride 8 per weather; first byte = render mode | `FUN_02338a4c` |
| Per-weather colormap (target) | via `GetWeatherColorTable` | `*(*DAT_022de634 + 0x48)` | `rgba[256]` per weather, stride `0x400` | `FUN_02334e70` |
| Tileset 3D mode | `DAT_02338a3c` | `TILESET_PROPERTIES` field, stride `0xC` | byte per tileset → render mode | `FUN_023389c4` |
| Scroll-step (per mode) | `DAT_02338d54` | — | stride 8: `+0` X step, `+4` Y step | `FUN_02338d34` |

## Weather State

The weather struct lives inside the dungeon struct at `dungeon + 0xCD38` (the `weather` field).

```c
struct weather {
    enum weather_id weather;                // 0x0  (dungeon+0xCD38): current weather
    enum weather_id natural_weather;        // 0x1  (dungeon+0xCD39): floor default, reverted to
    u16 weather_turns[8];                   // 0x2  (dungeon+0xCD3A): turns left per weather type
    u16 artificial_permaweather_turns[8];   // 0x12: ability permaweather (Drought, Drizzle, etc.)
    u8  weather_damage_counter;             // 0x22: sandstorm/hail damage, counts 9→0
    u8  mud_sport_turns;                    // 0x23
    u8  water_sport_turns;                  // 0x24
    bool8 nullify_weather;                  // 0x25: Cloud Nine / Air Lock active
};
```

```c
enum weather_id {
    WEATHER_CLEAR = 0,  WEATHER_SUNNY = 1,  WEATHER_SANDSTORM = 2,  WEATHER_CLOUDY = 3,
    WEATHER_RAIN  = 4,  WEATHER_HAIL  = 5,  WEATHER_FOG       = 6,  WEATHER_SNOW   = 7,
    WEATHER_RANDOM = 8,
};
```

The active weather is the type with the highest `weather_turns` value (ties broken in enum order), recomputed by `TryActivateWeather`. `artificial_permaweather_turns` (any nonzero entry) overrides this for ability-set weather. `nullify_weather` forces `WEATHER_CLEAR` as the apparent weather.

### Weather Sources

All sources write a turn counter then call `TryActivateWeather('\x01', '\0')` (announce = true, force = false). They carry **no visual logic** — the visuals are entirely downstream in `FUN_02334e70`.

**Moves** set `weather_turns[type] = WEATHER_MOVE_TURN_COUNT` (3000):

| Move | Writes offset | `weather_turns[]` | weather_id |
|------|---------------|-------------------|------------|
| Sunny Day | `0xCD3C` | `[1]` | WEATHER_SUNNY |
| Sandstorm | `0xCD3E` | `[2]` | WEATHER_SANDSTORM |
| Rain Dance | `0xCD42` | `[4]` | WEATHER_RAIN |
| Hail | `0xCD44` | `[5]` | WEATHER_HAIL |

**Evidence:** `DoMoveRainDance`
```c
*(undefined2 *)(*DAT_02326124 + 0xcd42) = *DAT_02326120;  // weather_turns[4] = WEATHER_MOVE_TURN_COUNT
bVar1 = TryActivateWeather('\x01','\0');
```

**Abilities** (`TryActivateArtificialWeatherAbilities`) scan all monsters and set `artificial_permaweather_turns[]`: Drizzle→rain (`+0xCD52`), Sand Stream→sandstorm (`+0xCD4E`), Drought→sunny (`+0xCD4C`), Snow Warning→snow (`+0xCD54`). Air Lock / Cloud Nine set `nullify_weather` (`+0xCD5D`).

## Visual/Audio Pipeline Overview

```
<source> (move / ability / orb / natural floor weather)
   └─► sets weather_turns[] or artificial_permaweather_turns[]
   └─► TryActivateWeather(announce, force)
         ├─ recompute dominant weather → write dungeon+0xCD38
         └─ if (old != new) || force:
              FUN_02334e70(announce, ...)        ← master "apply weather visuals"
                 1. FUN_022e5fe8(new, announce)   → per-weather sound + effect.bin animation
                 2. colormap fade loop            → animate tint toward GetWeatherColorTable(new)
                 3. FUN_02338a4c(new)             → configure 3D scrolling texture (fog/sandstorm)
                 4. FUN_02335044(announce, ...)   → weather log message + Castform/Cherrim form change
```

**Evidence:** `TryActivateWeather` change gate
```c
wVar3 = GetApparentWeather((entity *)0x0);   // old
// ... recompute dominant weather, write dungeon+0xcd38 ...
wVar5 = GetApparentWeather((entity *)0x0);   // new
if (wVar3 != wVar5 || param_2 != '\0') {
    FUN_02334e70((uint)param_1, ...);        // param_1 = announce (log + animate)
}
```

## Stage 1: Per-Weather Effect Animation & Sound (`FUN_022e5fe8`)

Plays a per-weather ambient sound, then (if the weather has an effect) spawns an effect.bin animation on the leader (or the entity at `dungeon + 0x1A22C` if set). The effect ID indexes `effect_animation_info`; it is dispatched via `PlayEffectAnimationEntity` (type-7, entity-positioned — see `Data Structures/effect_animation_info.md`).

**Evidence:** `FUN_022e5fe8`
```c
entity = *(entity **)(*DAT_022e60d4 + 0x1a22c);
if (entity == 0) entity = GetLeader();

uVar2 = *(ushort *)(DAT_022e60d8 + param_1 * 2);          // sound table, index = weather_id
if (uVar2 != 0x3f00) PlaySeByIdIfShouldDisplayEntity(entity, uVar2);

iVar3 = *(int *)(DAT_022e60dc + param_1 * 4);             // effect table (announced), gate >= 0
if (-1 < iVar3) {
    if ((param_2 == 0) || IsCurrentFixedRoomBossFight())
        iVar3 = *(int *)(DAT_022e60e0 + param_1 * 4);     // effect table (floor-entry / boss)
    // else: keep DAT_022e60dc value (announced mid-floor change)
    uVar2 = GetEffectAnimationField0x19(iVar3);            // attachment_point
    PlayEffectAnimationEntity(entity, iVar3, '\0', uVar2 & 0xff, 2, '\0', -1, 0);
}
```

`DAT_022e60dc` is both the gate (`>= 0` means "this weather has an effect") and the effect ID used for an **announced mid-floor change**. `DAT_022e60e0` is the effect ID used on **floor entry** (`announce == 0`) or during a **fixed-room boss fight**.

### Decoded Effect Table

`effect_animation_info` IDs, decoded from the table dumps:

| weather_id | Weather | `DAT_022e60dc` (change) | `DAT_022e60e0` (entry / boss) |
|------------|---------|-------------------------|-------------------------------|
| 0 | Clear | `-1` (none) | `-1` |
| 1 | Sunny | `331` (0x14B) | `331` |
| 2 | Sandstorm | `239` (0xEF) | `239` |
| 3 | Cloudy | `-1` (none) | `-1` |
| 4 | Rain | `16` (0x10) | `440` (0x1B8) |
| 5 | Hail | `20` (0x14) | `20` |
| 6 | Fog | `-1` (none) | `-1` |
| 7 | Snow | `223` (0xDF) | `223` |

Rain is the only weather whose two variants differ (16 vs 440) — both must be scraped. Clear, Cloudy, and Fog spawn no entity effect.

### Decoded Sound Table

`DAT_022e60d8` (`0x023511DC`), `u16[8]`, `0x3F00` = silence:

| weather_id | Weather | Sound ID |
|------------|---------|----------|
| 3 | Cloudy | `0x0311` (785) |
| 6 | Fog | `0x0312` (786) |
| all others | — | `0x3F00` (silent) |

Only Cloudy and Fog have an ambient sound here — precisely the two weathers with no effect animation (Clear has neither). The other weathers' audio rides along with their effect.bin animation's own `sfx_id`, not this table.

## Stage 2: Colormap Tint (all weathers)

`FUN_02334e70` animates the live floor colormap toward the new weather's target table. This tints the entire dungeon (e.g. darker/bluer in rain) and applies to **every** weather, including those with no effect animation.

**Asset — dungeon.bin file 1034.** The weather colvec is `dungeon.bin[1034]`: a SIR0-wrapped file, **8 colormaps × `0x400` bytes** = 256 RGBX entries each (4th byte `0xFF`), one map per `weather_id` (0–7). Confirmed by scan (the only SIR0 file that is 8×`0x400` with a constant X byte) and by content (map 0 is a near-identity ramp). `dungeon.bin[1023]` is a sibling colvec of identical format used for **fade-to-black** (all 8 maps send every value to 0) — not weather. Scrape via `FileType.DBIN_SIR0_COLVEC` (`ColvecHandler`), which returns 256 RGB per map (X stripped).

**Apply — three per-channel transfer curves.** Each colormap is three independent intensity curves, not a palette swap. For a palette colour whose channel `c` holds value `v`, the output is `colormap[w][v*3 + c]`. For clear/sunny/fog the three curves are identical (pure luminance remap, no hue shift); for sandstorm/cloudy/rain/hail/snow they diverge, producing the colour cast:

| weather_id | weather | tint |
|---|---|---|
| 0 | clear | identity — no tint |
| 1 | sunny | bright neutral, lifted blacks |
| 2 | sandstorm | warm tan (R≈G > B) |
| 3 | cloudy | cool blue-gray (B > R=G) |
| 4 | rain | dark blue, crushed shadows |
| 5 | hail | cold cyan |
| 6 | fog | flat low-contrast gray, heavily lifted blacks |
| 7 | snow | cold cyan-blue |

**Runtime path:**

- Target table: `GetWeatherColorTable(weather)` → `*(*DAT_022de634 + 0x48) + weather_id * 0x400`. `DAT_022de634` (`0x02353530`) is the dungeon-struct pointer, so the loaded colvec sits at `dungeon + 0x48`; the `* 0x400` stride matches the 8×`0x400` file layout.
- Live working buffer: `dungeon + 0x1E0`, 256 RGBA (R/G/B at `+0/+1/+2` of each 4-byte entry).
- Transition is animated: per frame each channel steps `±10` toward the target, snapping when within 10; up to 64 frames, breaking early on convergence.
- Flushed each frame via `FUN_022de608`; `DUNGEON_COLORMAP_PTR` is the render-side handle.
- **Loader note:** the file index 1034 is established from content + the `0x400` apply stride; an immediate-search for `0x40A` in code came up empty (likely a literal-pool constant), so the exact load call is not yet pinned. Minor open item.

**Evidence:** `FUN_02334e70` colormap loop (R channel shown; G/B identical)
```c
prVar6 = GetWeatherColorTable(GetApparentWeather(0));   // target table
for (iVar9 = 0; iVar9 < 0x40; ...) {                     // up to 64 frames
    bVar3 = false;
    AdvanceFrame('%');
    for (iVar11 = 0; iVar11 < 0x100; iVar11++) {          // 256 colors
        bVar2  = prVar6[iVar11].r;                         // target
        iVar13 = *DAT_02335040;                            // dungeon base
        uVar12 = *(byte *)(iVar13 + iVar11 * 4 + 0x1e0);   // current R at dungeon+0x1E0
        if (abs(uVar12 - bVar2) < 10) {
            *(byte *)(iVar13 + iVar11 * 4 + 0x1e0) = bVar2;  // snap
        } else {
            bVar3 = true;                                    // still animating
            // step current ±10 toward target
        }
        // ... G at +0x1e1, B at +0x1e2 ...
    }
    FUN_022de608();                                          // flush colormap
    if (!bVar3) break;                                       // converged
}
```

## Stage 3: 3D Weather Textures — Fog / Sandstorm

Fog and sandstorm add a translucent **3D scrolling-texture overlay** on top of the 2D dungeon — the only two `weather_id`s that do (clear/sunny/cloudy/rain/hail/snow have no 3D layer). `FUN_02338a4c` configures it: it indexes the per-weather 3D config table `DAT_02338ab4` (→ `0x02353730`, stride 8) by `weather_id`; the first byte is the render mode, which is **3 (linear scroll) for sandstorm (2) and fog (6), 0 for all others**. The textures are `dungeon.bin` WTE files (1001 fog, 1005 sandstorm), 4bpp paletted 128×128, loaded once per dungeon by `LoadWeather3DFiles`.

The full renderer — two-slot model, scroll modes/directions, alpha/fade, WTE load + VRAM layout, and the per-frame scroll math — is documented in **`weather_3d_rendering.md`**. From the pipeline's view, all `TryActivateWeather` does here is set the weather slot's render mode for the new weather.

### 3D WTE Assets (dungeon.bin / `PACK_ARCHIVE_DUNGEON`)

Loaded by `LoadWeather3DFiles` (`LoadWteFromFileDirectory` + `ProcessWte`), rendered by `RenderWeather3D`.

| dungeon.bin index | Effect | VRAM tex offset | tex size | palette offset |
|-------------------|--------|-----------------|----------|----------------|
| 1001 | Fog (weather) | `0xF000` | `0x2000` | `0x1600` |
| 1005 | Sandstorm (weather) | `0xD000` | `0x2000` | `0x1500` |
| 1031 | Mist/Steam (tileset-paired visual) | `0xB000` | `0x2000` | `0x1400` |
| 1003 | Poison mist (Sky Peak boss only; alt of 1031) | `0xB000` | `0x2000` | `0x1400` |

- Files are **WTE** (SIR0-wrapped texture + palette). **WTU is not used** for weather — it is the UV-atlas companion used only for glyph sheets (e.g. damage numbers, files 1007–1011).
- Textures are **static**; motion is code-driven in `RenderWeather3D`: a 3×3 tiled grid scrolled continuously (scroll accumulators wrap at `0x8000`), with polygon alpha ramping between `0x4000` and `0xC000`. A render state of `9` switches the linear scroll to sine-wave modulation (`SinAbs4096`); otherwise a linear step is used. There is no keyframe animation to scrape — reproduce the scroll math in the client.

## Stage 4: Message & Form Change (`FUN_02335044`)

Not asset-relevant, but part of the same dispatch:

- If `announce != 0`: logs the weather message string, id = `0xA45 + weather_id`
- Iterates all 20 monster slots and calls `TryWeatherFormChange` (Castform / Cherrim weather forms)

**Evidence:** `FUN_02335044`
```c
if (param_1 != 0) {
    wVar3 = GetApparentWeather(0);
    LogMessageByIdWithPopupCheckUser(peVar4, wVar3 + 0xa45 & 0xffff);  // message = 0xA45 + weather_id
}
for (iVar5 = 0; iVar5 < 0x14; iVar5++) {                                // 20 monster slots
    peVar4 = *(entity **)(*piVar1 + iVar5 * 4 + 0x12b78);
    if (EntityIsValid(peVar4)) TryWeatherFormChange(peVar4);
}
```

## Tileset 3D Effects (Mist / Steam)

A second 3D overlay, keyed by **tileset** rather than `weather_id`, shares the same `RenderWeather3D` two-slot system. It is driven per-floor by `FUN_023389c4` (called from `RunDungeon` just before the weather config), which reads the tileset's **`weather_effect`** property (`TILESET_PROPERTIES + 0x0A`, confirmed via `DAT_02338a3c = 0x022C6326`) and passes it straight through as the mist slot's scroll mode: 0 = no mist, 1–6 = drift direction. Tileset **195** is the sole special case — it swaps in `dungeon.bin[1003]` (poison mist, `DAT_02338a48 = 0x3EB`) over the default mist texture (1031). This system is independent of the `weather_id` fog above.

See **`weather_3d_rendering.md`** for the slot mechanics and **`tileset-properties.md`** for the `weather_effect` field.

### Slot Mechanics

| Function | Purpose |
|----------|---------|
| `FUN_02338d94(slot, idx)` | Init slot: alpha start `0x4000` (+0x244), random scroll phase (`DungeonRandInt(0x400)` at +0x25C/+0x260), `+0x264=4`, `+0x268=1`, build the 3×3 quad grid |
| `FUN_02338d34(slot, mode)` | Write mode to `+0x240` (the byte `RenderWeather3D` reads) and load scroll-step X/Y from `DAT_02338d54[mode*8]` |
| `FUN_02338e50(elem, idx)` | Per-quad texture/UV binding (slot → VRAM texture + geometry) — *not yet pulled* |

`RenderWeather3D` reads `+0x240`: `0` = inactive, `3` = linear scroll (fog/sandstorm), `9` = sine-wave drift (phase params from `FUN_02338d94`, masked by `DAT_02338d2c`).

## Per-Weather Output Summary

| Weather | Colormap | Ambient Sound | Effect Animation | 3D Texture |
|---------|----------|---------------|------------------|------------|
| Clear | yes | — | — | — |
| Sunny | yes | — | 331 | — |
| Sandstorm | yes | — | 239 | file 1005 |
| Cloudy | yes | `0x311` | — | — |
| Rain | yes | — | 16 / 440 | — |
| Hail | yes | — | 20 | — |
| Fog | yes | `0x312` | — | file 1001 |
| Snow | yes | — | 223 | — |

## Scraper / Client Notes

- **Effect animations:** resolve IDs {331, 239, 16, 440, 20, 223} through the `effect_animation_info` → effect.bin pipeline (see `Data Structures/effect_animation_info.md`). These are the precipitation/sky visuals.
- **Colormap:** scrape `dungeon.bin[1034]` via `FileType.DBIN_SIR0_COLVEC` (`ColvecHandler`) — `Colvec.colormaps[weather_id]` is a 256-entry table of three per-channel transfer curves; replicate `apply_colormap` (`out_c = colormap[w][in_c*3 + c]`) as a fullscreen palette remap. The `±10`/frame fade is optional polish.
- **3D textures:** WTE files 1001 (fog) / 1005 (sandstorm) — plus 1031/1003 for tileset mist — are 4bpp paletted 128×128 and decode directly via the `Wte` handler (`to_pil`), so no manual A3I5/A5I3 decoder is needed. Reimplement the tiled scroll in the client; see `weather_3d_rendering.md` for the exact math.
- **Audio:** ambient sound table only for Cloudy/Fog; all other weather audio is the effect animation's own `sfx_id`.

## Open Questions

- Exact load call for the weather colvec (`dungeon.bin[1034]`): the index is established from content + the `0x400` apply stride, but an immediate-search for `0x40A` in code was empty (likely a literal-pool constant), so the loader/init that stores it at `dungeon + 0x48` is not yet identified.
- Whether `DUNGEON_COLORMAP_PTR` points to the `dungeon + 0x1E0` working buffer or a separate render buffer flushed by `FUN_022de608` (cosmetic, internal).
- Sandstorm uses both effect 239 **and** the 3D texture (both confirmed configured); the in-game layering order (sprite over haze) is assumed, not visually verified.

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `GetApparentWeather` | `0x2334D08` | Current apparent weather for an entity (or floor if null) |
| `TryActivateWeather` | overlay 29 | Recompute dominant weather; call `FUN_02334e70` if changed |
| `TryActivateArtificialWeatherAbilities` | overlay 29 | Set permaweather from Drought/Drizzle/etc. |
| `FUN_02334e70` | `0x02334E70` | Master apply: sound+effect, colormap fade, 3D config, message |
| `FUN_022e5fe8` | `0x022E5FE8` | Per-weather ambient sound + effect.bin animation |
| `FUN_02338a4c` | `0x02338A4C` | Configure 3D scrolling-texture grid (fog/sandstorm) |
| `FUN_02335044` | `0x02335044` | Weather log message + Castform/Cherrim form change |
| `FUN_022de608` | `0x022DE608` | Flush/upload the live colormap |
| `GetWeatherColorTable` | `0x22DE620` | Per-weather target colormap (256 RGBA, stride `0x400`) |
| `LoadWeather3DFiles` | `0x23388B0` | Load WTE files 1001/1005/1031 from dungeon.bin |
| `RenderWeather3D` | `0x2338AC4` | Render scrolling 3D weather quads (fog/sandstorm) |
| `PlayEffectAnimationEntity` | — | Spawn effect.bin animation on an entity (type-7 dispatch) |
| `PlaySeByIdIfShouldDisplayEntity` | `0x22E56A0` | Play a sound if the entity should be displayed |
| `GetEffectAnimationField0x19` | — | Read `attachment_point` from `effect_animation_info` |
| `TryWeatherFormChange` | overlay 29 | Castform/Cherrim weather form change |
| `IsCurrentFixedRoomBossFight` | — | Selects the entry/boss effect-ID table |
| `LogMessageByIdWithPopupCheckUser` | — | Weather message popup (id `0xA45 + weather_id`) |
| `LoadWteFromFileDirectory` / `ProcessWte` | — | WTE load + texture/palette VRAM upload |
| `RunDungeon` | overlay 29 | Calls `LoadWeather3DFiles` (once) and `FUN_023389c4`/`FUN_02338a4c` (per floor) |
| `FUN_023389c4` | `0x023389C4` | Tileset 3D activation: set mist slot mode; swap to 1003 for tileset 195 |
| `FUN_02338d94` | `0x02338D94` | Init a 3D render slot (alpha, scroll phase, 3×3 grid) |
| `FUN_02338d34` | `0x02338D34` | Set slot render mode + load per-mode scroll steps |
| `FUN_02338e50` | `0x02338E50` | Per-quad texture/UV binding (not yet pulled) |

## Cross-References

> See `Data Structures/effect_animation_info.md` for resolving the weather effect IDs (331/239/16/440/20/223) to WAN files and animation indices, and for the type-7 `PlayEffectAnimationEntity` dispatch.

> See `screen_effect_system.md` for the unrelated type-5/6 move screen effects (distinct from the weather 3D system).

> See `weather_3d_rendering.md` for the full `RenderWeather3D` system (two-slot model, scroll modes/directions, alpha/fade, WTE textures) shared by fog/sandstorm and the tileset mist.

> See `tileset-properties.md` for the per-tileset `weather_effect` field, which selects the 3D mist drift mode/direction (separate from the `weather_id` system here), and `stirring_effect` for ambient 2D particles.
