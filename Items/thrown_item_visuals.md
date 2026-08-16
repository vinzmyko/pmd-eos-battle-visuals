# Thrown Item Visuals (Dungeon Mode)

## Summary

- Thrown items use a **dedicated scratch entity** embedded in the dungeon struct at `+0xCC`, not the item entity pool
- Two entirely separate motion systems, split on `item_category`:
  - `CATEGORY_THROWN_LINE` (0) → `FUN_02347518`, flat travel, 6 frames/tile, sprite rotates with direction
  - `CATEGORY_THROWN_ARC` (1) → `HandleCurvedProjectileThrow`, half-sine parabola, 12 frames/Manhattan-tile, no rotation
- Both draw via `FUN_023457c8` (item icon from `items.itm.img`) plus `FUN_022e9488` (ground shadow)
- Line-thrown spikes use **three sprite shapes × four flips** to cover 8 directions
- `entity + 0x1C` is a generic vertical render offset — resolved here as the arc height
- Gravity/arc is NOT the `FUN_022beb2c` move-projectile system; do not reuse those formulas

## Scratch Entity (`dungeon + 0xCC`)

Both throw handlers write to a single reused entity. Absolute offsets are `dungeon + N`;
relative offsets are from the entity base `0xCC`.

| Abs | Rel | Type | Field |
|-----|-----|------|-------|
| 0xCC | +0x00 | u32 | `type = 3` (thrown-item entity type; NOT `ENTITY_ITEM`) |
| 0xD0 | +0x04 | s16 | tile x |
| 0xD2 | +0x06 | s16 | tile y |
| 0xD8 | +0x0C | s32 | pixel x (8.8) |
| 0xDC | +0x10 | s32 | pixel y (8.8) |
| 0xE8 | +0x1C | s32 | **vertical render offset** (8.8) — arc height |
| 0xEC | +0x20 | u8 | `= 1` (visible) |
| 0xEE | +0x22 | u8 | `= 0` |
| 0xF0 | +0x24 | u8 | `= 0` |
| 0xF2 | +0x26 | u16 | `= 0` |
| 0x180 | +0xB4 | `item*` | the item being thrown; renderer reads `id` from here |

`entity + 0x1C` also appears in the monster renderer (`FUN_02303f18` draw-Y calc,
`(pixel_pos.y − entity[0x1C] − info[0x188]) >> 8`), confirming it as the generic
per-entity vertical offset. Larger = drawn higher.

## Renderer: `FUN_023457c8` (`0x023457c8`)

```c
undefined4 FUN_023457c8(entity *ent, int alt_sprite_mode, int param_3,
                        int direction_or_ff, bool use_z);
```

`param_3 != 0` takes an undocumented `GetTile` path (not used by either throw handler).

**Position:**
```c
screen_x = (ent->pixel_pos.x >> 8) - cam_x;
screen_y = dungeon[0x1A230] + (((ent->pixel_pos.y - ent[0x1C]) >> 8) - cam_y);
z        = (dungeon[0x1A230] + ((ent->pixel_pos.y >> 8) - cam_y) + 8) / 2;
```

**The z-sort uses the un-offset Y.** A lobbed item does not change draw layer as it
rises. Clients that sort on the arced Y will see items pop in front of entities at
the peak of the throw.

`use_z == 0` forces `z = 1`.

**Cull box:** `-0x20 ≤ x < 0x120`, `-0x20 ≤ y < 0xE0`. Returns 0 when culled, which is
what suppresses `AdvanceFrame` in both callers — off-screen throws resolve instantly
with no animation.

**Sprite selection:**
```c
sprite  = GetItemSpriteId(item->id);
palette = GetItemPaletteId(item->id);

if (direction_or_ff != 0xFF && (sprite == 0 || sprite == 0x3B)) {
    flip   = SPIKE_FLIP_TABLE[direction_or_ff];    // u8, bits 0-1 → OAM bit 12
    sprite = SPIKE_SPRITE_TABLE[direction_or_ff];  // u8
}
tile_index = sprite * 2 + 0x110;
```

`alt_sprite_mode` (from `dungeon[0x1A245]`) forces sprite `0x17`, palette `0xA`,
ignoring the item entirely. Purpose unknown.

Submits via `FUN_0201b9f8` — an OAM queue push (queue at `param_1 + 0x20`, count at
`+0x40`, cap 0x80 entries).

### Directional Spike Sprites

Two `u8[8]` tables, indexed directly by direction (0–7):

| Symbol | Address (NA) | Contents |
|--------|--------------|----------|
| `SPIKE_SPRITE_TABLE` | `0x023537B4` | `15 00 16 00 15 00 16 00` |
| `SPIKE_FLIP_TABLE` | `0x023537BC` | `02 03 01 01 00 00 00 02` |

Flip is `(value & 3)` shifted to OAM bit 12: bit 0 = H-flip, bit 1 = V-flip.

| dir | facing | sprite | flip |
|-----|--------|--------|------|
| 0 | Down | 0x15 (vertical) | V |
| 1 | Down-Right | 0x00 (diagonal) | H+V |
| 2 | Right | 0x16 (horizontal) | H |
| 3 | Up-Right | 0x00 (diagonal) | H |
| 4 | Up | 0x15 (vertical) | — |
| 5 | Up-Left | 0x00 (diagonal) | — |
| 6 | Left | 0x16 (horizontal) | — |
| 7 | Down-Left | 0x00 (diagonal) | V |

Diagonals reuse the item's own base shape (sprite 0) with four flips; orthogonals swap
to two dedicated shapes. Palette is never overridden, so all line-thrown items share
these three shapes and differ only by recolour.

**Suspected bug:** the override also fires for base sprite `0x3B` (Gold Thorn), but the
table is authored for base 0 — a diagonally-thrown Gold Thorn renders as a generic
spike. Unconfirmed in-game.

## Ground Shadow: `FUN_022e9488` (`0x022e9488`)

```c
FUN_022e9488(px + 8, py + 16, IsWaterTileset() ? 3 : 0);
```

`px`/`py` are the **un-arced** pixel position, so the shadow stays on the ground track
while the sprite rides the parabola. Third arg selects an OAM slot at stride `0xC`
(slot 3 on water tilesets — presumably a ripple rather than a shadow). Own, tighter
cull box: `-0x10 … 0x10F` / `-0x10 … 0xCF`.

## Line Throw: `FUN_02347518` (`0x02347518`)

```c
void FUN_02347518(entity *user, item *item, position *start,
                  uint direction, projectile_throw_info *info);
```

Start position:
```c
pixel_x = (start->x * 24 + 4) << 8;
pixel_y = (start->y * 24)     << 8;   // NOTE: no +4 on Y, unlike the arc path
```
`ent[0x1C]` is left at 0 — flat travel, no arc.

Travel loop:
```c
dx = DIRECTIONS_XY[dir].x << 10;   // 0x400 = 4 px in 8.8
dy = DIRECTIONS_XY[dir].y << 10;

for (;;) {                          // outer: one iteration per tile
    tile += DIRECTIONS_XY[dir];
    if (tile.x < 0 || tile.x >= 0x38 || tile.y < 0 || tile.y >= 0x20) {
        outcome = 2; break;
    }
    for (i = 0; i < 6; i++) {       // inner: 6 frames per tile
        IncrementEntityPixelPosXY(ent, dx, dy);
        shadow = FUN_022e9488(px + 8, py + 16, water ? 3 : 0);
        drawn  = FUN_023457c8(ent, alt, 0, dir & 0xFF, 1);
        if (drawn || shadow) AdvanceFrame(0x12);
    }
    // GetTile(tile) → wall / target collision (Ghidra-mangled, see Open Questions)
}
```

**6 frames × 4 px = 24 px per tile**, constant speed. Exactly 2× faster than the arc
path. `AdvanceFrame` arg is `0x12`.

Hits are accumulated during the walk into a stack array of `{entity*, bool hit}` pairs
(8 bytes/entry), then processed after the flight:

- hit → log `0xBE1`, `TryEndPetrifiedOrSleepStatus`, `ApplyItemEffect(1, lockon||pierce, woke, user, target, item)`
- miss → log `0xBE1 + 1` (pierce set) or `0xBE1 + 2`

Outcome dispatch:
```c
knockback = DIRECTIONS_XY[dir] * 8;
switch (outcome) {
  case 0: break;                                              // consumed on a target
  case 1: SpawnDroppedItem(user, ent, item, 1, &knockback, 0); // landed
  case 2: LogMessageByIdWithPopupCheckUser(user, 0xBE5);       // out of bounds
}
```

## Arc Throw: `HandleCurvedProjectileThrow` (`0x02347bc8`)

```c
void HandleCurvedProjectileThrow(entity *user, item *item, position *start,
                                 position *target, projectile_throw_info *info);
```

Start position: `((start->x * 24 + 4) << 8, (start->y * 24 + 4) << 8)` — the standard
item `+4,+4` offset on both axes.

```c
n          = (|dx| + |dy|) * 12;      // Manhattan distance → total frames
amp        = min(n + 12, 64);
phase_step = 0x80000 / n;             // half sine (0x800) over the whole flight

x = start->x * 0x1800;                // 0x1800 = 24 << 8
y = start->y * 0x1800;
step_x = (target->x * 0x1800 - x) / n;
step_y = (target->y * 0x1800 - y) / n;

for (i = 0; i < n - 3; i++) {         // NOTE: stops 3 frames early
    ent[0x1C] = amp * SinAbs4096(phase >> 8);
    phase += phase_step;
    SetEntityPixelPosXY(ent, x + 0x400, y + 0x400);
    shadow = FUN_022e9488((x >> 8) + 8, (y >> 8) + 16, water ? 3 : 0);
    drawn  = FUN_023457c8(ent, alt, 0, 0xFF, 1);
    if (drawn || shadow) AdvanceFrame(0x17);
    x += step_x;
    y += step_y;
}
```

Direction is passed as `0xFF`, so arc-thrown items **never rotate**.

`projectile_throw_info` is never read by this function — arc range is bounded by the
target position chosen upstream, not by `max_range`.

## `projectile_throw_info`

Built in `UseThrowableItem`, 4 bytes:

```c
struct projectile_throw_info {
    /* 0x0 */ u8  pierce;      // 1 = passes through targets
    /* 0x1 */ u8  curve;       // ItemIsActive(user, ITEM_CURVE_BAND)
    /* 0x2 */ u16 max_range;
};
```

From `statuses.long_toss` (`info + 0xEE`):

| long_toss | pierce | max_range |
|-----------|--------|-----------|
| 0 | 0 | 10 |
| 1 | 0 | 99 |
| 2 | 1 | 99 |
| 3 | *(uninitialised — struct left as stack garbage)* |

Then overridden: `EXCLUSIVE_EFF_LONG_TOSS` → `(0, 99)`; `ITEM_PIERCE_BAND` or
`IQ_PIERCE_HURLER` → `(1, 99)`.

Only the line path reads it.

## Throw Sequence (`UseThrowableItem`, `0x022f54bc`)

1. Embargo / sticky / run-away checks; item removed or decremented from bag or ground
2. `ITEM_NO_AIM_SCOPE` (0x30) → `direction = DungeonRandInt(8)`
3. If visible, the **wind-up spin**:
```c
   PlaySeById(0x103);
   for (i = 0; i < 8; i++) {
       dir = (dir - 1) & 7;            // counter-clockwise
       ChangeMonsterAnimation(user, 0, dir);
       FUN_022ea370(2, 0x15, ...);     // wait 2 units
   }
   monster->field_0x179 = 4;
```
4. Log message: `0xBC0` (line) or `0xBBF` (arc)
5. `FUN_022e5728(user, category)` → throw SFX
6. Build `projectile_throw_info`
7. Arc → `FUN_022e9a9c` (target selection) then `HandleCurvedProjectileThrow`;
   line → `FUN_02347518`
8. `FUN_02304a48`, `TryTriggerMonsterHouse`

`FUN_0234b4cc(1)` / `(0)` brackets the flight in both handlers — writes a single global
byte at `*(DAT_0234b4dc + 4) + 0xC8A`. Animation/input lock.

## Impact & Use VFX

`FUN_022e5a00` and `FUN_022e5ae4` are a mode pair over `FUN_022bec94`:

```c
anim_arg = (item->flags & 8) ? 0 : item->id;   // sticky items fall back to entry 0
FUN_022bec94(anim_arg, &pos, attach_offsets, MODE, z);
```

Inside `FUN_022bec94`:
```c
anim_id = (MODE == 0) ? GetItemAnimation2(id) : GetItemAnimation1(id);
```

| Caller | MODE | Table | Called from | Attachment |
|--------|------|-------|-------------|------------|
| `FUN_022e5a00` | 0 | `GetItemAnimation2` | thrown at a target | forced to 3 (Centre), offset at `param_3 + 0xC` |
| `FUN_022e5ae4` | 1 | `GetItemAnimation1` | item used / eaten | `effect_animation->field_0x19`, offset at `param_3 + idx*4` |

Dispatched to layer 4 via `FUN_022be780(4, ...)`.

### `ITEM_ANIMATION_INFO`

`0x022C7A84`, `struct { int16 anim1; int16 anim2; } [1400]`, indexed by item id.

| Item id range | anim1 | anim2 |
|---------------|-------|-------|
| 0–68 (thrown / held) | 0 | 8, 9, or 0x0A |
| 69 (0x45) onward | 0x197 | 0x197 |

Item `0x45` is exactly where `ShouldTryEatItem` begins, so every edible item shares
effect 407. The 8/9/0x0A split across ids 0–68 does **not** track the line/arc category
(Stick = 8, Iron Thorn = 0x0A, Gravelerock = 9) — it is a per-item flavour variant.

## Throw SFX: `FUN_022e5728` (`0x022e5728`)

```c
void FUN_022e5728(entity *user, item_category category) {
    if (category == CATEGORY_THROWN_LINE) PlaySeByIdIfNotSilence(DAT_022e5760);
    else if (category == CATEGORY_THROWN_ARC) PlaySeByIdIfNotSilence(DAT_022e5764);
    else PlaySeByIdIfNotSilence(DAT_022e5768);
}
```

## Hit Determination: `DoesProjectileHitTarget` (`0x02348020`)

`THROWN_ITEM_HIT_CHANCE` = **0x5A (90)**, stored at `0x022C46B0`.

```c
if (target->info[0x09] == 1) return false;
if (target->info[0xBC] == 7) return false;
hit = DungeonRandInt(100) < 90;
if (user->type == ENTITY_MONSTER) {
    if (ItemIsActive(user, 0x2F))                hit = false;   // early-out
    else if (ItemIsActive(user, ITEM_LOCKON_SPECS /*0x31*/)) hit = true;
}
if (target->type == ENTITY_MONSTER) {
    if (FUN_022fb9bc(target)
        || ItemIsActive(target, 0x2C)
        || ExclusiveItemEffectIsActive(target, 0x53)) hit = false;
}
```

Ordering matters: Lock-on Specs only applies when item `0x2F` is inactive, and
target-side effects override everything.

Note `info[0xBC]` sits one byte before `sleep_class_status` (`0xBD`) and is not yet
documented in `Data Structures/entity.md`.

## AI Arc Targeting: `FUN_022e9a9c` (`0x022e9a9c`)

Three tiers, first match wins:

1. **Statused** (`CheckVariousStatuses2` true) → `user.pos + DIRECTIONS_XY[dir] * 3`
2. **Explicit target** (`info[0x5A]` / `info[0x5C]` != -1) → those coordinates
3. **Auto-aim** → walk a per-direction scan pattern; fall back to
   `user.pos + DIRECTIONS_XY[dir] * 2`

Entity pool by context: `dungeon + 0x12B78` (20) if `dungeon[0x3E38]` set, else
`+0x12B28` (4, team) or `+0x12B38` (16) depending on `info[6]`.

### Scan Pattern Table

Base `0x0235179C`, stride 8, indexed by direction:

```c
struct scan_dir {
    /* 0x0 */ const int16_t *pattern;   // {dx,dy} pairs, terminated by 99
    /* 0x4 */ int16_t scale_x;
    /* 0x6 */ int16_t scale_y;
};
```

| dir | pattern | scale_x | scale_y |
|-----|---------|---------|---------|
| 0 Down | `0x02351974` | +1 | +1 |
| 1 Down-Right | `0x02351B00` | +1 | +1 |
| 2 Right | `0x02351C94` | +1 | +1 |
| 3 Up-Right | `0x02351B00` | +1 | −1 |
| 4 Up | `0x02351974` | −1 | −1 |
| 5 Up-Left | `0x02351B00` | −1 | −1 |
| 6 Left | `0x02351C94` | −1 | −1 |
| 7 Down-Left | `0x02351B00` | −1 | +1 |

Three distinct patterns — vertical (`0x02351974`), diagonal (`0x02351B00`), horizontal
(`0x02351C94`) — reflected into the other quadrants by sign, mirroring the sprite-table
trick. Target is `user.pos + (pattern[i].dx * scale_x, pattern[i].dy * scale_y)`.

Sizes: vertical 396 bytes (99 pairs), diagonal 404 bytes (101 pairs).

This is AI-side targeting only; a client driven by an authoritative server does not
need it.

## Line-Thrown Damage Path

`CATEGORY_THROWN_LINE` items fabricate a move and route through the normal move
damage system:

```c
InitMove(&m, PROJECTILE_MOVE_ID);   // 0x195
DealDamageProjectile(attacker, defender, &m, power, 0x100, ITEM_STICK);
```

`ApplyItemEffect` cases 1–6 and 9 cover Stick, Iron Thorn, Silver Spike, Gold Fang,
Cacnea Spike, Corsola Twig, Gold Thorn. Other thrown items (Gravelerock case 7, Geo
Pebble case 8, case 10) use `CalcDamageFixedNoCategory` instead.

`DealDamageProjectile` is damage only — the flight visual is `FUN_02347518`, not the
move-animation system. `PROJECTILE_MOVE_ID` has no `move_animation_info` role here.

## Implementation Summary

| Parameter | Line (`CATEGORY_THROWN_LINE`) | Arc (`CATEGORY_THROWN_ARC`) |
|-----------|-------------------------------|-----------------------------|
| Handler | `FUN_02347518` | `HandleCurvedProjectileThrow` |
| Start pixel | `(tile.x*24+4, tile.y*24)` | `(tile.x*24+4, tile.y*24+4)` |
| Frames | 6 per tile | `(|dx|+|dy|)*12`, loop runs `n-3` |
| Speed | 4 px/frame, constant | linear lerp over `n` |
| Vertical | none | `amp * sin(phase)`, `amp = min(n+12, 64)` |
| Phase step | — | `0x80000 / n` |
| `AdvanceFrame` arg | `0x12` | `0x17` |
| Sprite | direction tables (3 shapes × 4 flips) | `item_p` unchanged |
| Shadow | `(px+8, py+16)` | `(px+8, py+16)`, un-arced |
| Range | `projectile_throw_info.max_range` | bounded by chosen target |
| Impact VFX | `GetItemAnimation2(id)` | `GetItemAnimation2(id)` |

Common to both: 8-step counter-clockwise wind-up spin, SFX `0x103` on the spin then the
category SFX, z-sort on un-offset Y, instant resolution when off-screen.

## Open Questions

- **`dungeon[0x1A245]`** (`alt_sprite_mode`): forces sprite `0x17` / palette `0xA`,
  bypassing the item entirely. Set where, and for what? A write watchpoint would settle it.
- **`SpawnDroppedItem`** (`0x02345ad8`): landing / bounce / wall-hit resolution. Ghidra
  mis-flags `GetTile` as non-returning, truncating the whole body. Receives
  `DIRECTIONS_XY[dir] * 8` as a scatter vector.
- **`FUN_02347518` tile-collision block**: same `GetTile` truncation. The wall stop and
  per-tile enemy accumulation are unread.
- **`FUN_023457c8` `param_3 != 0` branch**: unreached by either throw handler.
- **Gold Thorn diagonal override**: sprite `0x3B` triggers the directional table but gets
  sprite 0. Confirm in-game whether this is a visible vanilla bug.
- **`long_toss == 3`**: leaves `projectile_throw_info` uninitialised. Reachable?
- **Scan pattern contents**: the three `int16` arrays at `0x02351974`, `0x02351B00`,
  `0x02351C94` are not yet dumped.
- **`entity + 0x20/0x22/0x24/0x26`** on the scratch entity (`dungeon + 0xEC/0xEE/0xF0/0xF2`):
  written to fixed values at setup, never read by the throw handlers.
- **`FUN_022be780` layer 4**: not catalogued alongside the layer 0/1/2/3 effect layers.

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `UseThrowableItem` | `0x022f54bc` | Top-level throw: bag bookkeeping, spin, dispatch |
| `FUN_02347518` | `0x02347518` | Line-throw flight handler |
| `HandleCurvedProjectileThrow` | `0x02347bc8` | Arc-throw flight handler |
| `FUN_023457c8` | `0x023457c8` | Thrown-item sprite renderer |
| `FUN_022e9488` | `0x022e9488` | Ground shadow / water ripple renderer |
| `FUN_0201b9f8` | `0x0201b9f8` | OAM queue push |
| `FUN_022e9a9c` | `0x022e9a9c` | AI arc target selection |
| `DoesProjectileHitTarget` | `0x02348020` | 90% hit roll + item/ability overrides |
| `FUN_022e5728` | `0x022e5728` | Throw SFX by category |
| `FUN_022e5a00` | `0x022e5a00` | Impact VFX (mode 0 → `GetItemAnimation2`) |
| `FUN_022e5ae4` | `0x022e5ae4` | Use/eat VFX (mode 1 → `GetItemAnimation1`) |
| `FUN_022bec94` | `0x022bec94` | Item effect-animation dispatcher (layer 4) |
| `FUN_0234b4cc` | `0x0234b4cc` | Animation/input lock (global byte at `+0xC8A`) |
| `DealDamageProjectile` | — | Damage for line-thrown items via `PROJECTILE_MOVE_ID` |
| `SpawnDroppedItem` | `0x02345ad8` | Landing / bounce resolution (undecompiled) |
| `ApplyItemEffect` | `0x0231b9xx` | Per-item effect switch |
| `GetItemSpriteId` / `GetItemPaletteId` | — | `item_p` appearance lookup |
| `GetItemAnimation1` / `GetItemAnimation2` | `0x022bfee8` / `0x022bff04` | `ITEM_ANIMATION_INFO` lookup |

## Cross-References

> See `Items/item_data_extraction.md` for `items.itm.img` sprite/palette extraction and
> the `(sprite_id, palette_id)` appearance key.

> See `Systems/entity_positioning.md` for the tile/pixel coordinate conventions and the
> `+4,+4` vs `+12,+16` spawn offsets.

> See `Systems/projectile_motion.md` for the **move**-projectile system, which is a
> separate mechanism — do not reuse its `FUN_022beb2c` gravity formulas for items.
