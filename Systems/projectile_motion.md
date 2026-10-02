# Projectile Motion

## Summary

- `FUN_023230fc` runs one flight per strike, only when `GetMoveRangeDistance` (R) is nonzero
- The endpoint is found first: walk from the attacker's tile along its facing for up to R tiles, stopping on the first wall or monster tile (inclusive). T = tiles walked
- Speed is 2/3/6 px per frame; each tile takes `fc = 24 / speed` frames (12/8/4). Flight = `T × fc` frames
- The ground track runs from the attacker's tile centre (+12, +16) along the facing direction
- Wave patterns: 0 none, 1 screen-vertical bump, 2 sideways bulge to the right of travel. Both are a half sine over the whole flight
- The attacker's attachment offset (the move's `attachment_point_idx`) seeds `ctx + 0x24/0x26` and decays toward (0, −9) each frame with divisor `n = 6T`. This is the "gravity arc"
- Draw order = ground Y / 2 + `{1,1,1,0,0,0,1,1}[dir]`
- `GetBodySize ≥ 4` with R == 1 spawns no effect; the frames still run
- Wall/monster checks and the hit roll happen per tile inside the flight; hits resolve after it

## Endpoint and Spawn

### Endpoint Pre-Walk (`FUN_023230fc` @ 0x02323198)

```c
tile = attacker->pos; T = 0; hit = NULL;
for (i = 0; i < R; i++) {
    if (tile.x < 0 || tile.y < 0 || tile.x >= 56 || tile.y >= 32) break;
    tile += DIRECTIONS_XY[dir];
    T++;                                         // local_64
    t = GetTile(tile);
    if ((t->terrain & 3) == 0) break;            // wall
    if (t->monster && t->monster->type == ENTITY_MONSTER) { hit = t->monster; break; }
}
```

**Evidence:** hand-decoded. Ghidra marks `GetTile` as no-return and leaves these as raw bytes.
```
023231e8  ldrh  r1,[r0,#0x0]    ; tile terrain flags
023231ec  tst   r1,#0x3
023231f0  beq   0x0232321c      ; wall → stop
023231f4  ldr   r1,[r0,#0xc]    ; tile->monster
023231f8  cmp   r1,#0x0
023231fc  beq   0x02323210      ; empty → next tile
02323200  ldr   r0,[r1,#0x0]    ; entity->type
02323204  cmp   r0,#0x1         ; ENTITY_MONSTER
02323208  moveq r5,r1           ; remember hit monster
0232320c  beq   0x0232321c      ; monster → stop
02323210  add   r4,r4,#0x1
```

`tile` becomes the destination passed to `FUN_02322f78`. `T` drives amplitude and phase.

### Spawn (`FUN_02322f78`)

No effect is spawned (returns −1) when:
- `dungeon + 0x1A23E` is set
- `GetBodySize(species) ≥ 4` and R == 1
- the move's layer 3 effect id is 0

**Body size:** `monster.md` `body_size` (u8, entry offset 0x13). Values are only 1, 2 or 4 (1080 / 35 / 40 entries). Size 4: Onix, Gyarados, Steelix, Lugia, Ho-Oh, Wailord, Milotic, Castform's forms, Kyogre, Groudon, Rayquaza. The offset comes from this data correlation and pmdsky-debug's `monster_data_table_entry`; `GetBodySize` itself is not decompiled.

```c
idx = FUN_022bf01c(species, anim_id);               // move.attachment_point_idx, species % 600 override
if (idx == -1) launch = *(s16x2 *)0x02352A54;       // (0, 0)
else           FUN_0201cf90(&launch, &attacker->anim_ctrl, idx);
dest = (tile.x * 24 + 12, tile.y * 24 + 16);
FUN_022be9e8(&{anim_id, species, px >> 8, py >> 8, launch.x, launch.y, dir, 0}, &dest, 0, dir);
```

The attachment index is the **move's** `attachment_point_idx` (0x11), not the effect's `field_0x19`. `FUN_022be9e8` stores the same index (via the identical `FUN_022bf088`) at `ctx + 0x28`.

**Launch pose:** `o₀` is read from the attacker's current frame when `FUN_02322f78` runs. `FUN_023250d4` ends its attack loop on the first frame with WAN flag `0x1` (return point), when entity byte `+0x21` is set, or after 120 frames. Nothing resets the pose between that and `FUN_023230fc`. Types 98/99 end on `walk` (anim 0) in the original direction, 2 frames in.

**Spawn param mapping** (stack in `FUN_02322f78` → `param_1` in `FUN_022be9e8` → context):

| Stack | `param_1[i]` | Context |
|-------|--------------|---------|
| sp+4 | [0] anim move id | — |
| sp+6 | [1] species | 0x130 |
| sp+8/A | [2],[3] attacker pixel >> 8 | 0x20/0x22, 0x128/0x12A |
| sp+C/E | [4],[5] launch offset | **0x24/0x26**, 0x132/0x134 |
| sp+10 | [6..7] direction | 0x1C |
| sp+14 | [8..9] 0 | 0x18 (delay) |

**Evidence:** `FUN_02322f78` asm
```
02323074  add  r0,sp,#0xc        ; FUN_0201cf90 writes launch offset to sp+0xC
023230a4  add  r0,sp,#0x4        ; param_1 = sp+4
023230a8  add  r1,sp,#0x0        ; param_2 = sp+0 (dest)
023230d8  bl   FUN_022be9e8
```

## Speed System

### Raw to Mapped Speed

The `projectile_speed` field in move_animation_info is mapped to actual values:

| Raw Value | Mapped Value | Frame Count | Description |
|-----------|--------------|-------------|-------------|
| 1 | 2 | 12 frames | Slow projectile |
| 2 | 3 | 8 frames | Medium projectile |
| Other (0, 3+) | 6 | 4 frames | Fast projectile |

**Evidence:** `FUN_023230fc`
```c
iVar7 = GetMoveAnimationSpeed((uint)(ushort)param_2[2]);
if (iVar7 == 1) {
    iVar7 = 2;
}
else if (iVar7 == 2) {
    iVar7 = 3;
}
else {
    iVar7 = 6;
}
```

### Frame Count Calculation
```c
frame_count = 24 / mapped_speed;  // 0x18 / iVar7
// Results: 24/2=12, 24/3=8, 24/6=4
```

**Evidence:** `FUN_023230fc`
```c
uVar20 = _s32_div_f(0x18, iVar7);
```

### Velocity Calculation

Velocity is speed multiplied by fixed point scale:
```c
velocity = mapped_speed * 256;  // iVar7 * 0x100
// Results: 512, 768, or 1536 in fixed point
```

**Evidence:** `FUN_023230fc`
```c
iVar7 = iVar7 * 0x100;
```

## Direction System

### Direction Table

8-direction vectors stored in `DIRECTIONS_XY`:

| Index | Direction | X Delta | Y Delta |
|-------|-----------|---------|---------|
| 0 | Down | 0 | +1 |
| 1 | Down-Right | +1 | +1 |
| 2 | Right | +1 | 0 |
| 3 | Up-Right | +1 | -1 |
| 4 | Up | 0 | -1 |
| 5 | Up-Left | -1 | -1 |
| 6 | Left | -1 | 0 |
| 7 | Down-Left | -1 | +1 |

**Evidence:** `pmd-sky/src/dungeon_util.c`
```c
const struct position DIRECTIONS_XY[] = {
    {0, 1},    // 0: Down
    {1, 1},    // 1: Down-Right
    {1, 0},    // 2: Right
    {1, -1},   // 3: Up-Right
    {0, -1},   // 4: Up
    {-1, -1},  // 5: Up-Left
    {-1, 0},   // 6: Left
    {-1, 1}    // 7: Down-Left
};
```

### Direction Lookup

Direction is read from monster_info and used as table index:

**Evidence:** `FUN_023230fc`
```c
uVar18 = (uint)*(byte *)(uVar18 + 0x4c);  // Read direction from monster_info
iVar16 = *(int *)(DAT_02323900 + uVar18 * 4);
sVar3 = *(short *)(DAT_023238f8 + uVar18 * 4);  // X delta
sVar4 = *(short *)(DAT_023238fc + uVar18 * 4);  // Y delta
```

### Memory Addresses

| Symbol | Address (NA) | Contents |
|--------|--------------|----------|
| DAT_023238f4 | 0x02352A54 | Default launch offset (0, 0), then two −1 handle inits |
| DAT_023238f8 | 0x0235171C | DIRECTIONS_XY x (stride 4) |
| DAT_023238fc | 0x0235171E | DIRECTIONS_XY y |
| DAT_02323900 | 0x0235175C | DIRECTION_ANGLE_4096, int32[8] |
| DAT_02323904 | — | Angle mask 0xFFF |
| DAT_02323908 | 0x02352A6C | PROJECTILE_DRAW_ORDER_BIAS, int32[8] |
| DAT_0232390c | 0x02353538 | DUNGEON_PTR |
| DAT_02323910 | — | 0x1A226, camera Y in dungeon struct |
| DIRECTIONS_XY | 0x0235171C | Direction vector table |

### Reverse Direction

For boomerang/return effects, direction is rotated 180°:
```c
reversed_direction = (original_direction + 4) & 7;
```

**Evidence:** `FUN_0232393c`
```c
iVar4 = (*(byte *)(iVar2 + 0x4c) + 4 & 7) * 4;
iVar11 = (int)*(short *)(DAT_02323c30 + iVar4);  // Reversed X delta
sVar1 = *(short *)(DAT_02323c34 + iVar4);        // Reversed Y delta
```

## Flight Loop (`FUN_023230fc`)

### Setup

```c
fc         = 24 / speed;
amp        = (R < 2) ? 32 : min(T * fc + 8, 64);
phase_step = 0x80000 / (T * fc);          // phase >> 8 spans 0..0x800 = half a turn
z_bias     = PROJECTILE_DRAW_ORDER_BIAS[dir];
side_angle = (DIRECTION_ANGLE_4096[dir] + 0xC00) & 0xFFF;
```

**Evidence:** asm 0x02323334-0x023233f4. `0x02323340 mul r1,r2,r0` (T × fc, the divisor); amplitude at 0x02323344; phase at 0x02323360; side angle at 0x023233bc/0x023233cc; z bias at 0x023233e4.

### Per Tile (outer loop, up to R iterations)

```c
prev = tile;  tile += DIRECTIONS_XY[dir];
if (FUN_022e2ca0(tile) && !dungeon[0x1A23E]) {
    ground_x = (prev.x * 24 + 12) << 8;
    ground_y = (prev.y * 24 + 16) << 8;
    for (f = 0; f < fc; f++) { /* per frame */ }
} else {
    phase += phase_step * fc;            // frames skipped, phase kept in step
}
t = GetTile(tile);
if (wall) break;
if (monster) { /* hit checks, add to target list */ break; }
```

`FUN_022e2ca0` is a visibility check (within ±6 x / ±5 y of the camera tile at `dungeon + 0x1A21C`, and in sight), not collision.

The post-tile block (0x02323758-0x02323814, hand-decoded) tests wall then monster, makes four checks (calls at 0x02323788 with 0x2E, 0x0232379C with 0x60, 0x023237B0, 0x023237C4; unidentified), builds the target list at `sp + 0xA8`, stores the count in `local_60` and leaves the loop.

### Per Frame

```c
if (wave == 1)      { wx = 0;  wy = amp * Sin(phase >> 8); }
else if (wave == 2) { r  = ((amp >> 1) * Sin(phase >> 8)) >> 8;
                      wx = r * Cos(side_angle);  wy = r * Sin(side_angle); }
else                { wx = wy = 0; }

base = ((ground_x + wx) >> 8, (ground_y - wy) >> 8);
z    = z_bias + ((ground_y >> 8) - cam_y) / 2;   // cam_y = *(s16 *)(dungeon + 0x1A226)
FUN_022beb2c(handle, &base, z);                  // sets 0x20/0x22, decays 0x24/0x26, sets 0x2C
AdvanceFrame('0');                               // tick draws here
ground_x += dx * speed * 256;
ground_y += dy * speed * 256;
phase    += phase_step;
```

- Y wave is subtracted: positive = up on screen
- Pattern 1 ignores direction (screen-vertical only)
- Pattern 2 is **not a spiral**. The angle is fixed for the whole flight; only the radius changes. It is a sideways bulge of `amp / 2` to the right of travel; in y-down screen coordinates the offset direction is `(−dy, dx)` of the travel vector
- `z` uses the ground track only (no wave, no launch offset)
- The last drawn ground position is one step short of the end tile centre

### Wave Pattern Source

`flags & 7` (`FUN_022bfd58`) → `FUN_02324e78` → returned by `FUN_02322ddc` → passed by `FUN_02322374` as `param_4`. `FUN_02322ddc` returns 0 when neither the attacker nor any target is displayed.

The only moves with pattern 2 and a nonzero layer 3 effect are Skill Swap, Guard Swap, Power Swap, Heart Swap and Switcheroo. All five also set flag bit 3, so the bulge pairs with the return projectile on the opposite side.

### Launch Offset Decay (`FUN_022beb2c`)

```c
n  = max(|dest_x - src_x|, |dest_y - src_y|) / 4;   // = 6T for a standing attacker
oy = (s16)(oy + 9);
ox = (s16)(ox * (n - 1));
oy = (s16)(oy * (n - 1));
ox = ox / n;                                        // truncate toward zero
oy = oy / n;
oy -= 9;
```

**Evidence:** asm 0x022beb9c-0x022bebf8. Every intermediate is written back with `strh`; the division is `_s32_div_f` (truncating).

- Runs before `AdvanceFrame`, so the undecayed launch offset is never drawn
- Converges to (0, −9)
- Fraction of `(o + 9)` left on arrival ≈ `(1 − 1/6T)^(T·fc)`: about 0.5 / 0.25 / 0.13 at speed 6 / 3 / 2, nearly independent of T
- Example, T = 1, speed 6, launch offset (−10, −40): drawn offsets (−8, −34), (−6, −29), (−5, −25), (−4, −22)
- Launch offset (0, 0), n = 6: y = −2, −4, −5, −6, −7, −8, −9 (x stays 0)
- n = 0: `_s32_div_f(x, 0)` returns x, so `ox = −ox` and `oy = −oy − 18` every frame (flips). See "Return Projectiles"

The offset is only visible when `ctx + 0x28 != −1`; with `attachment_point_idx == −1` the projectile follows ground track + wave exactly. See `effect_context.md` → "Render Tick".

## Effect Cleanup

After projectile completes, effect is stopped:
```c
if (effect_handle >= 0) {
    FUN_022bde50((int)(short)effect_handle);
}
```

**Evidence:** `FUN_023230fc`
```c
if (-1 < local_38) {
    FUN_022bde50((int)(short)local_38);
}
```

## Return Projectiles

### Second Projectile (`param_7`)

`param_7` = move flag bit 3 (`FUN_022bfd6c`). If the pre-walk hit a monster, a second effect is spawned from it (`FUN_02322f78(hit, &hit->pos, ...)`) and moves along `dir + 4`, sharing the phase. Pattern 1 amplitude is negated (`local_b4 = −amp`); pattern 2 uses `+0x1400` (= +0x400, the other side).

### `FUN_0232393c`

Travels `param_4` tiles from `param_2`'s tile along the user's `dir + 4`. No pre-walk and no hit check; amplitude and phase use `param_4`; only pattern 1 (`param_5`). Ends with `AnimationDelayOrSomething`, then `PlayMoveAnimation` at the end tile, and clears `info + 0x170` if the move is `DAT_02323c44`. Caller unidentified.

### n = 0

Both pass the projectile's own tile as `dest`, so source equals destination and the decay flips the offset each frame. Visible only when the attachment index is not −1. Probable ROM bug.

## Post-Flight

```c
FUN_022bde50(handle);  FUN_022bde50(second);
FUN_0234b4cc(0);
if (move == 0x1E5) AnimationDelayOrSomething(1);
if (target_count > 0)                       ExecuteMoveEffect(&targets, attacker, move, ...);
else if (R == 1 && FUN_022e2ca0(end_tile)) { FUN_022ea370(1, 0x4A); PlayMoveAnimation(attacker, NULL, move, &end_tile); }
```

With R == 1 and no target, the primary still plays on the empty end tile.

Back in `FUN_02322374`: `FUN_02304b14`, then up to 100 frames (`'J'`) waiting while `FUN_0201d1b0` reports the attacker's animation still running. The attack animation keeps playing during the flight; the attacker does not return to idle before launch.

## Trigonometry

- 4096 units per turn, maths convention: 0 = right, 0x400 = up, 0xC00 = down
- `SinAbs4096`: quarter-wave table at `DAT_02001978` (index `x & 0x3FF`, mirrored for 0x400-0x7FF, negated for 0x800-0xFFF)
- Output scale (`table[0x3FF]`) unconfirmed. Clients assuming 0x100 = 1.0 get `amp` in pixels, which matches observed arcs

**DIRECTION_ANGLE_4096** (`0x0235175C`):

| dir | 0 D | 1 DR | 2 R | 3 UR | 4 U | 5 UL | 6 L | 7 DL |
|-----|-----|------|-----|------|-----|------|-----|------|
| angle | 0xC00 | 0xE00 | 0x000 | 0x200 | 0x400 | 0x600 | 0x800 | 0xA00 |

**PROJECTILE_DRAW_ORDER_BIAS** (`0x02352A6C`): `{1, 1, 1, 0, 0, 0, 1, 1}`. It ties with the entity row when travelling toward the top of the screen and sits one bucket in front otherwise. This is neither the unbound ±3 nor the bound ±1 table.

## Dive/Dig Terrain Check (`FUN_02325d20`)

Unrelated to projectile waves. Returns 1 for Dive on a ground tile or Dig on a non-water tile. Gates the animation paths in `PlayMoveAnimation`, `FUN_023250d4`, `FUN_023258ec` and `FUN_02324e78`.

## Implementation Summary

| Parameter | ROM |
|-----------|-----|
| Flight runs | `R = GetMoveRangeDistance(user, move, 1) > 0` |
| Effect spawned | layer 3 ≠ 0 and not (`BodySize ≥ 4` and R == 1) |
| Tiles flown T | pre-walk to first wall/monster, ≤ R |
| Ground start | attacker tile × 24 + (12, 16) |
| Ground step | `DIRECTIONS_XY[dir] × speed` px/frame |
| Speed / fc | raw 1 → 2 / 12, raw 2 → 3 / 8, else 6 / 4 |
| Total frames | T × fc |
| Amplitude | R < 2 ? 32 : min(T·fc + 8, 64) |
| Phase | 0 → 0x800 over T·fc frames |
| Pattern 1 | y −= amp · sin |
| Pattern 2 | += r · (cos θ, −sin θ); r = (amp/2) · sin; θ = angle[dir] + 0xC00 |
| Launch offset | attacker attachment[move.attachment_point_idx, species override] at launch |
| Decay | n = 6T; applied before every draw |
| Offset ignored | attachment index == −1 |
| Draw order | (ground_y − cam_y)/2 + bias[dir] |
| 99 component | frame not drawn |

All units are DS screen pixels, 1:1 with sprite pixels.

## Cross-References

> See `Data Structures/effect_context.md` for trajectory field storage (offsets 0x128-0x134) and offset field dual-purpose (0x24-0x26)

> See `Systems/move_effect_pipeline.md` for projectile spawn call chain

> See `Systems/entity_positioning.md` for coordinate system details and entity-binding system (FUN_022bfb6c)

> See `move_target_and_range.md` for how `GetMoveRangeDistance` derives param_3 from `waza_p.bin`

> See `Systems/effect_lifecycle.md` for FUN_022bf4f0 tick function details

> See `Items/thrown_item_visuals.md` for **thrown item** motion, which is a separate
> system. Line-thrown items travel flat at 6 frames/tile; arc-thrown items use a
> half-sine parabola. Neither uses the `FUN_022beb2c` gravity arc documented here.

## Resolved Questions (cont.)

### `FUN_0234b4cc`
**Resolved:** writes a single global byte at `*(DAT_0234b4dc + 4) + 0xC8A`, set to 1 for
the duration of an animation sequence and 0 after. An animation/input lock. It brackets
both thrown-item flight handlers identically. See `Items/thrown_item_visuals.md`.

## Resolved Questions

- **"Gravity arc":** the attacker's launch offset decaying toward (0, −9), not a fixed arc from 0
- **Arc divisor:** n = 6T, not constant 6. The old claim misread the pre-walk as a single step
- **Destination:** the first wall/monster tile within R, not attacker + 1
- **Attachment index:** the move's `attachment_point_idx` (0x11). The old "all projectiles have 0-3" checked the effect's `field_0x19`
- **Decompiler artifacts:** the amplitude/phase divisor is `T × fc` (`local_64 × frame_count`), not `R × fc`
- **Pattern 2:** a fixed-direction sideways bulge; angle mask is 0xFFF
- **Draw order:** explicit bias table, not the unbound sentinel
- **Launch pose:** the attack loop's break frame (first return frame), not idle and not the animation's last frame
- **Body size field:** `monster.md` 0x13, values 1/2/4

## Open Questions

- `SinAbs4096` output scale (`DAT_02001978[0x3FF]`)
- The four hit-check calls in the post-tile block (needs the `GetTile` flow fix)
- Caller of `FUN_0232393c` and the move at `DAT_02323c44`
- Whether any flag-bit-3 move has R > 0 (if so, the second projectile jitters)
- `dungeon + 0x1A23E`: skips spawn and frames; probably "animations off"

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_02322374` | `0x02322374` | Strike loop; calls flight when R > 0 |
| `FUN_023230fc` | `0x023230fc` | Pre-walk, flight loop, per-tile hit checks |
| `FUN_0232393c` | `0x0232393c` | Reverse-direction flight |
| `FUN_02322f78` | `0x02322f78` | Spawn: body-size gate, launch offset, dest |
| `FUN_022be9e8` | `0x022be9e8` | Layer 3 spawn; writes 0x128-0x134 |
| `FUN_022beb2c` | `0x022beb2c` | Per-frame base position, offset decay, draw order |
| `FUN_022bf4f0` | `0x022bf4f0` | Render tick |
| `FUN_022bf01c` / `FUN_022bf088` | `0x022bf01c` / `0x022bf088` | Move attachment index with species override |
| `FUN_0201cf90` | `0x0201cf90` | Attachment offset from current WAN frame |
| `FUN_022e2ca0` | `0x022e2ca0` | Tile visibility |
| `FUN_022bde50` | `0x022bde50` | Stop effect |
| `FUN_0234b4cc` | `0x0234b4cc` | Animation/input lock |
| `FUN_02324e78` | `0x02324e78` | Final wave pattern |
| `FUN_022bfd58` | `0x022bfd58` | flags & 7 |
| `FUN_022bfd6c` | `0x022bfd6c` | Flag bit 3 (second projectile) |
| `FUN_02325d20` | `0x02325d20` | Dive/Dig terrain gate |
| `GetMoveAnimationSpeed` | — | Raw speed |
| `GetMoveRangeDistance` | — | R |
| `GetBodySize` | — | Large-body gate |
| `SinAbs4096` / `CosAbs4096` | — | 4096-unit trig |
| `_s32_div_f` | — | Truncating divide; x / 0 = x |
