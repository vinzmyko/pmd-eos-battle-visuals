# Screen Shake (Dungeon Mode)

## Summary

- A self-contained dungeon-mode visual driven by three `display_data` fields in the dungeon struct
- Triggered by **exactly two moves** — Earthquake (118) and Magnitude (296). Nothing else in the dungeon engine shakes the screen (no traps, explosions, abilities, or weather)
- Each move sets the shake in **two places**: the shared move-animation executor (`FUN_023250d4`) and the move's own effect handler
- A per-frame consumer (`FUN_022e34c8`) maps the intensity counter to a render offset via a lookup table, decays the counter, and reloads it from a reset value
- Cleared on floor load by the display init (`FUN_022e2b68`)
- Both shake tables are now dumped (see **Tables**): the intensity→offset curve is a single-axis signed scalar, and Magnitude's per-level intensities are `[6, 12, 18, 20, 24, 30, 30]`

| Symbol / Purpose | Address (NA) |
|------------------|--------------|
| `FUN_022e34c8` (per-frame consumer — unnamed) | `0x022E34C8` |
| `UpdateCamera` (per-frame caller; calls consumer at `0x022E32D0`) | — |
| `FUN_022e2b68` (display init / shake reset — unnamed) | `0x022E2B68` |
| `FUN_023250d4` (move-animation executor — unnamed) | `0x023250D4` |
| `DoMoveEarthquake` | `0x02328074` |
| `DoMoveMagnitude` | `0x0232B738` |
| `DUNGEON_PTR` (global holding the dungeon base) | `0x02353538` |
| intensity → offset curve (`int32[32]`) — via ptr `DAT_022e3530` (`0x022E3530`) | `0x0235110C` |
| Magnitude per-level intensities (`int32[7]`) — via ptr `DAT_02325610` (`0x02325610`) | `0x02352B1C` |
| Magnitude per-level power (`int16[7]`, damage) — via ptr `DAT_0232b7bc` (`0x0232B7BC`) | `0x022C4924` |
| stored Magnitude roll (runtime global) — via ptr `DAT_02325614` (`0x02325614`) | `0x0237CA84` |
| `DAT_02325618` (Magnitude message string id) | data |

## State Fields

All three live in `struct display_data`, which begins at `dungeon + 0x1A21C`.

```c
// within struct display_data (base dungeon+0x1A21C):
uint32_t screen_shake_offset;          // +0x14  (dungeon+0x1A230): per-frame render offset, written by the consumer
uint32_t screen_shake_intensity;       // +0x18  (dungeon+0x1A234): countdown; nonzero = shaking; -1 per frame
uint32_t screen_shake_intensity_reset; // +0x1C  (dungeon+0x1A238): value reloaded into intensity at 0 (0 = stop, nonzero = sustained)
```

- A **trigger** starts a shake by writing a nonzero `screen_shake_intensity` (and a `screen_shake_intensity_reset`).
- The **consumer** turns the current intensity into `screen_shake_offset` each frame and decays intensity.
- `screen_shake_intensity_reset` controls duration: `0` = one decay to zero then stop; nonzero = re-arm to that value (sustained loop).

## Per-Frame Consumer (`FUN_022e34c8`)

Runs once per frame as part of the dungeon display/camera tick. This is the entire shake "animation" — there is no formula, the offset is a table lookup.

**Evidence:** `FUN_022e34c8`
```c
intensity = *(int *)(dungeon + 0x1a234);
if (intensity != 0) {
    if (0x1e < intensity) intensity = 0x1f;                       // clamp index to 31
    *(int *)(dungeon + 0x1a230) = *(int *)(DAT_022e3530 + intensity * 4);  // offset = curve[intensity]
    *(int *)(dungeon + 0x1a234) += -1;                            // decay
    if (*(int *)(dungeon + 0x1a234) == 0)
        *(int *)(dungeon + 0x1a234) = *(int *)(dungeon + 0x1a238); // reload from reset
}
```

- `screen_shake_offset = curve[ min(intensity, 0x1F) ]`, where `curve` is the `int32[32]` table at `0x0235110C`. `DAT_022e3530` (at `0x022E3530`) is a **pointer** to that table, so the access is pointer-load-then-index, not a direct array. Values are signed single-axis pixel offsets — see **Tables**.
- Intensity decrements by 1 each frame. At 0 it reloads from `screen_shake_intensity_reset`; if that is 0, the shake stops.
- The renderer applies `screen_shake_offset` to the view origin (consumer of `0x1A230`).

## Triggers

Screen shake is hardcoded by move ID, not driven by any `move_animation_info` flag. Both moves set it in their animation (`FUN_023250d4`) and again in their effect handler. Because the pipeline plays the animation **before** applying the effect, the shake runs sustained during the animation, then the effect handler re-arms it with `reset = 0` so it tapers off once and stops.

### Earthquake (move 118 / 0x76)

**Evidence:** `FUN_023250d4` (animation, move 0x76)
```c
else if (*(short *)(move + 4) == 0x76) {
    *(int *)(dungeon + 0x1a234) = 0xc;                            // intensity = 12
    *(int *)(dungeon + 0x1a238) = *(int *)(dungeon + 0x1a234);    // reset = 12 → sustained during animation
    PlaySeByIdIfShouldDisplayEntity(entity, 0x214);              // rumble SFX
}
```

**Evidence:** `DoMoveEarthquake` (effect handler)
```c
*(int *)(dungeon + 0x1a234) = 0xc;   // intensity = 12
*(int *)(dungeon + 0x1a238) = 0;     // reset = 0 → taper to zero and stop after the animation
// ... Dig-state double-damage check, then DealDamage ...
```

Net: sustained shake while the animation plays, then a final ~12-frame decay and stop.

### Magnitude (move 296 / 0x128)

**Evidence:** `FUN_023250d4` (animation, move 0x128)
```c
if (*(short *)(move + 4) == 0x128) {
    roll = DungeonRandInt(7);                                     // 0..6
    *DAT_02325614 = roll;                                        // store rolled level (read by DoMoveMagnitude)
    *(int *)(dungeon + 0x1a234) = *(int *)(DAT_02325610 + roll * 4);  // intensity = per-level table
    *(int *)(dungeon + 0x1a238) = *(int *)(dungeon + 0x1a234);    // reset = intensity → sustained
    SetMessageLogPreprocessorArgsNumberVal('\0', *DAT_02325614 + 4);
    LogMessageByIdWithPopupCheckUser(entity, DAT_02325618);       // "Magnitude N!"
    PlaySeByIdIfShouldDisplayEntity(entity, 0x214);              // rumble SFX
}
```

`DoMoveMagnitude` (effect handler) resolved: it re-reads `intensities[stored_roll]` (`0x02352B1C`) into `screen_shake_intensity` (`0x1A234`) and sets `screen_shake_intensity_reset` (`0x1A238`) to `0` — the same one-shot taper as Earthquake, recomputed from the stored roll. On the damage side it indexes the power table (`0x022C4924`) by the same roll, doubling when the target is underground, then deals damage. Announced number (`roll + 4`) gives magnitudes 4–10.

**Evidence:** `DoMoveMagnitude`
```c
roll = *DAT_0232b7b0;                                            // stored roll (global 0x0237CA84)
*(int *)(dungeon + 0x1a234) = *(int *)(DAT_0232b7b8 + roll * 4); // intensity = intensities[roll]
*(int *)(dungeon + 0x1a238) = 0;                                // reset = 0 → one-shot taper
power = *(short *)(DAT_0232b7bc + roll * 2);                     // power table 0x022C4924
if (defender->info[0xd2] == 0xa) power <<= 1;                   // underground → double damage
// ... deal damage with power ...
```

## Reset (`FUN_022e2b68`)

The display-data init, called during floor setup in `RunDungeon`. Among many `display_data` fields, it zeroes all three shake fields, so a shake never carries across floors.

**Evidence:** `FUN_022e2b68`
```c
*(int *)(dungeon + 0x1a230) = 0;   // screen_shake_offset
*(int *)(dungeon + 0x1a234) = 0;   // screen_shake_intensity
*(int *)(dungeon + 0x1a238) = 0;   // screen_shake_intensity_reset
```

## Tables

The `DAT_` symbols are **pointers**; the arrays live at the pointee addresses.

### Intensity → offset curve (`int32[32]` @ `0x0235110C`, via `DAT_022e3530`)
```
idx:  0   1   2   3   4   5   6   7    8   9  10  11  12  13  14  15

val:  0  -1   1  -1   1  -1   1  -2    1  -2   2  -2   2  -2   2  -3

idx: 16  17  18  19  20  21  22  23   24  25  26  27  28  29  30  31

val:  3  -3   3  -3   4  -4   4  -4    4  -5   4  -5   4  -5   4  -5
```

- **Single-axis signed scalar, not packed X/Y.** Each entry is a small signed int sign-extended into the high half (negatives `0xFFFFFFFF…`); a packed X/Y would carry a nonzero second axis there, which these don't.
- The curve carries both oscillation and envelope: adjacent entries alternate sign, amplitude grows with index. The consumer decrements intensity by 1/frame, so the index walks down the table — sign flips frame-to-frame (the shake) while amplitude tapers to 0 (the decay). No sine/formula.
- Earthquake's fixed intensity 12 peaks at `curve[12] = 2`. Magnitude's max intensity 30 stays under the `0x1F` clamp.

### Magnitude per-level intensities (`int32[7]` @ `0x02352B1C`, via `DAT_02325610`)

| roll | announced (roll+4) | intensity |
|------|--------------------|-----------|
| 0 | 4 | 6 |
| 1 | 5 | 12 |
| 2 | 6 | 18 |
| 3 | 7 | 20 |
| 4 | 8 | 24 |
| 5 | 9 | 30 |
| 6 | 10 | 30 |

### Magnitude per-level power (`int16[7]` @ `0x022C4924`, via `DAT_0232b7bc`)

Damage power, **not** shake — read by `DoMoveMagnitude`, indexed by the same roll, doubled when the target is underground (`info + 0xD2 == 0x0A`), mirroring Earthquake's double-damage check. Newly identified; only `roll → 5` is visible in the current dump, full contents still to transcribe.

## Client / Scraper Notes

- Reproduce the shake as a per-frame **single-axis signed** offset on the camera/render origin: `offset = curve[min(intensity, 31)]`, decrement intensity each frame, reload from reset at 0.
- Curve (`int32[32]` @ `0x0235110C`) and Magnitude intensities (`int32[7]` @ `0x02352B1C`) are dumped — see **Tables**. `DAT_022e3530` / `DAT_02325610` are pointers; the arrays are at the pointees.
- Camera center is computed each frame in `UpdateCamera`: `cam_x = (entity.pixel_pos.x >> 8) - 0x80` (`0x1A224`), `cam_y = (entity.pixel_pos.y >> 8) - 0x6c` (`0x1A226`); the shake offset is added downstream at draw time. Which axis/axes it lands on is the lone remaining unknown — approximate as both (rumble) until the draw reader is pulled.
- Both moves play SFX `0x214` (rumble); Earthquake's other audio comes from its `move_animation_info` sound, not this path.
- Trigger semantics: write `intensity` + `intensity_reset`; `reset = 0` for one-shot, `reset = intensity` for sustained-during-animation.

## Open Questions

- **Axis of application** — `screen_shake_offset` (`0x1A230`) is confirmed a single signed scalar (not packed X/Y), but whether draw code adds it to X, Y, or both is still open. `UpdateCamera` only triggers the write; the reader that combines it with the camera center (`0x1A224`/`0x1A226`) sits elsewhere, not adjacent to the writer. To find it: define a struct with a field at `+0x230`, retype the `DUNGEON_PTR` global (`0x02353538`) to a pointer to it, then xref the field.
- Full contents of the Magnitude **power** table (`int16[7]` @ `0x022C4924`) — newly identified, only one entry visible so far.

Resolved since last revision: both shake tables dumped; offset confirmed single-axis scalar; `DoMoveMagnitude`'s `0x1A234` role (re-reads `intensities[roll]`, sets `reset = 0`); consumer call site (`UpdateCamera`).

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_022e34c8` | `0x022E34C8` | Per-frame consumer: intensity → offset (table lookup), decay, reload. Sole caller: `UpdateCamera` (call at `0x022E32D0`) |
| `UpdateCamera` | — | Per-frame camera/minimap/blinker tick; computes camera center (`0x1A224`/`0x1A226`) and calls the consumer at the end |
| `FUN_022e2b68` | `0x022E2B68` | Display-data init; zeroes the shake fields on floor load |
| `FUN_023250d4` | `0x023250D4` | Move-animation executor; sets shake for moves 0x76 / 0x128 |
| `DoMoveEarthquake` | `0x02328074` | Earthquake effect; sets one-shot shake (reset 0) |
| `DoMoveMagnitude` | `0x0232B738` | Magnitude effect; re-reads `intensities[roll]`→`0x1A234`, sets `reset = 0` (one-shot); indexes power table `0x022C4924` for damage (×2 if target underground) |
| `DungeonRandInt` | — | Rolls Magnitude's 0–6 level |
| `PlaySeByIdIfShouldDisplayEntity` | `0x022E56A0` | Plays the rumble SFX (`0x214`) |

## Cross-References

> See `weather_visual_pipeline.md` for the `display_data` region context (the shake fields sit just before the camera/minimap display data at `dungeon + 0x1A21C`).

> See `Data Structures/move_animation_info.md` for `FUN_023250d4`'s role as the move-animation executor (monster anim types, flag-bit delays) — the shake branches are hardcoded by move ID, independent of the animation flags.
