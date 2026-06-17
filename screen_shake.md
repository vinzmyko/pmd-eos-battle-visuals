# Screen Shake (Dungeon Mode)

## Summary

- A self-contained dungeon-mode visual driven by three `display_data` fields in the dungeon struct
- Triggered by **exactly two moves** — Earthquake (118) and Magnitude (296). Nothing else in the dungeon engine shakes the screen (no traps, explosions, abilities, or weather)
- Each move sets the shake in **two places**: the shared move-animation executor (`FUN_023250d4`) and the move's own effect handler
- A per-frame consumer (`FUN_022e34c8`) maps the intensity counter to a render offset via a lookup table, decays the counter, and reloads it from a reset value
- Cleared on floor load by the display init (`FUN_022e2b68`)
- Client reproduction needs two table dumps: the intensity→offset curve and Magnitude's per-level intensities

## Memory Locations

| Symbol / Purpose | Address (NA) |
|------------------|--------------|
| `FUN_022e34c8` (per-frame consumer — unnamed) | `0x022E34C8` |
| `FUN_022e2b68` (display init / shake reset — unnamed) | `0x022E2B68` |
| `FUN_023250d4` (move-animation executor — unnamed) | `0x023250D4` |
| `DoMoveEarthquake` | `0x02328074` |
| `DoMoveMagnitude` | `0x0232B738` |
| `DAT_022e3530` (intensity → offset curve, `int32[32]`) | data |
| `DAT_02325610` (Magnitude per-level intensities, `int32[7]`) | data |
| `DAT_02325614` (stored Magnitude roll, runtime global) | data |
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

- `screen_shake_offset = DAT_022e3530[ min(intensity, 0x1F) ]` — `DAT_022e3530` is a 32-entry × 4-byte displacement curve.
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

`DoMoveMagnitude` (effect handler) references the same field on the damage side (reads the stored roll / ends the shake). The announced number (`roll + 4`) gives magnitudes 4–10.

## Reset (`FUN_022e2b68`)

The display-data init, called during floor setup in `RunDungeon`. Among many `display_data` fields, it zeroes all three shake fields, so a shake never carries across floors.

**Evidence:** `FUN_022e2b68`
```c
*(int *)(dungeon + 0x1a230) = 0;   // screen_shake_offset
*(int *)(dungeon + 0x1a234) = 0;   // screen_shake_intensity
*(int *)(dungeon + 0x1a238) = 0;   // screen_shake_intensity_reset
```

## Client / Scraper Notes

- Reproduce the shake as a per-frame offset on the camera/render origin: `offset = curve[min(intensity, 31)]`, decrement intensity each frame, reload from reset at 0.
- **Dump `DAT_022e3530`** (`int32[32]`) — the intensity→offset curve. This is the actual shake displacement per level.
- **Dump `DAT_02325610`** (`int32[7]`) — Magnitude's per-level intensities (indexed by the 0–6 roll).
- Both moves play SFX `0x214` (rumble); Earthquake's other audio comes from its `move_animation_info` sound, not this path.
- Trigger semantics: write `intensity` + `intensity_reset`; `reset = 0` for one-shot, `reset = intensity` for sustained-during-animation.

## Open Questions

- Contents of `DAT_022e3530` (offset curve) and `DAT_02325610` (Magnitude intensities) — to be dumped from the ROM.
- Whether `screen_shake_offset` (`0x1A230`) is a single-axis pixel offset or a packed X/Y value — confirm by inspecting the curve values and where the renderer reads `0x1A230`.
- Exact role of `DoMoveMagnitude`'s `0x1A234` reference (reads the stored roll vs. sets `reset = 0`) — body not yet pulled.
- Call site of `FUN_022e34c8` within the per-frame display/camera tick.

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_022e34c8` | `0x022E34C8` | Per-frame consumer: intensity → offset (table lookup), decay, reload |
| `FUN_022e2b68` | `0x022E2B68` | Display-data init; zeroes the shake fields on floor load |
| `FUN_023250d4` | `0x023250D4` | Move-animation executor; sets shake for moves 0x76 / 0x128 |
| `DoMoveEarthquake` | `0x02328074` | Earthquake effect; sets one-shot shake (reset 0) |
| `DoMoveMagnitude` | `0x0232B738` | Magnitude effect; uses the stored roll / ends shake |
| `DungeonRandInt` | — | Rolls Magnitude's 0–6 level |
| `PlaySeByIdIfShouldDisplayEntity` | `0x022E56A0` | Plays the rumble SFX (`0x214`) |

## Cross-References

> See `weather_visual_pipeline.md` for the `display_data` region context (the shake fields sit just before the camera/minimap display data at `dungeon + 0x1A21C`).

> See `Data Structures/move_animation_info.md` for `FUN_023250d4`'s role as the move-animation executor (monster anim types, flag-bit delays) — the shake branches are hardcoded by move ID, independent of the animation flags.
