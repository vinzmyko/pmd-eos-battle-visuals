# Fog of War Rendering (Dungeon Mode)

## Summary

- Fog of war is rendered entirely via BG layer 1 tile replacement, not 3D polygons or alpha blending
- Each dungeon tile maps to a 3×3 grid of NDS BG tiles (24×24 pixels at 8×8 per BG tile)
- `FUN_02336db0` is the core function: reads tile terrain bits and visibility flags, returns a pointer to the appropriate tilemap entries
- Five visual states: full dark, fully lit, gray (revealed-not-visited), edge transition, secondary variant
- Soft edges at dark/lit boundaries use a pre-computed lookup table indexed by neighbor visibility pattern
- Gradient artwork is baked into the tileset tile graphics, not computed at runtime
- The system is gated by `display_data.darkness`; Luminous Orb and natural lighting override to fully lit

---

## Display Data Fields

The `display_data` struct (within the dungeon struct, accessed at `base + 0x1a21c`) contains all visibility control state.

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 (0x1a21c) | 2×int16 | camera_pos | Leader tile position (x, y) |
| +0x08 (0x1a224) | 2×int16 | camera_pixel_pos | Camera pixel position (pixel_x >> 8 - 0x80, pixel_y >> 8 - 0x6C) |
| +0x10 (0x1a22c) | ptr | camera_target | Leader entity pointer |
| +0x21 (0x1a23d) | uint8 | visibility_range | 0 = no darkness (default 2), 1-3 = dark floor |
| +0x22 (0x1a23e) | uint8 | blinded | Camera entity has Blinker status |
| +0x23 (0x1a23f) | uint8 | luminous | Luminous Orb used / darkness force-disabled |
| +0x24 (0x1a240) | uint8 | natural_lighting | Floor has no darkness (visibility_range == 0) |
| +0x26 (0x1a242) | uint8 | can_see_enemies | X-Ray Specs / Power Ears active |
| +0x27 (0x1a243) | uint8 | can_see_items | X-Ray Specs active |
| +0x28 (0x1a244) | uint8 | can_see_all | See-all flag (monsters + traps) |
| +0x2D (0x1a249) | uint8 | field_0x2D | Controls darkness display |
| +0x34 (0x1a250) | uint8 | dropeye_status | Dropeye status on camera entity |

**Evidence:** `display_data` struct definition in pmdsky-debug headers, confirmed by `FUN_022e2b68` initialization and `UpdateCamera` field writes.

## Visibility Range Initialization

`FUN_022e2b68` initializes display_data from floor properties at dungeon start.

**Evidence:** `FUN_022e2b68`
```c
bVar1 = *(byte *)(*DAT_022e2c5c + 0x286c8);  // floor_properties.visibility_range
*(byte *)(iVar3 + 0x1a23d) = bVar1 & 3;       // Store masked to 2 bits (0-3)
if ((bVar1 & 3) == 0) {
    *(undefined *)(iVar3 + 0x1a240) = 1;       // natural_lighting = true (no darkness)
}
```

`GetVisibilityRange` reads the stored value, defaulting to 2 if 0:

**Evidence:** `GetVisibilityRange`
```c
int GetVisibilityRange(void) {
    uint uVar1 = (uint)*(byte *)(*DAT_022e3358 + 0x1a23d);
    if (uVar1 == 0) {
        uVar1 = 2;
    }
    return uVar1;
}
```

## Illumination Check

`FUN_022e3534` determines if the floor is fully illuminated (no darkness rendering needed).

**Evidence:** `FUN_022e3534`
```c
bool IsFloorIlluminated(display_data *dd) {
    if (dd->field_0x34 != 0)       // Force-darkness override (cutscenes?)
        return false;
    if (dd->luminous)              // Luminous Orb active
        return true;
    if (dd->natural_lighting)      // Floor has no darkness
        return true;
    return false;                  // Dark floor, no luminous
}
```

Called by `FUN_022e2ca0` before `IsPositionActuallyInSight` — if illuminated, all positions are considered visible without range checks.

**Evidence:** `FUN_022e2ca0`
```c
iVar2 = FUN_022e3534((uint)(iVar3 + 0x1a21c));
if (iVar2 != 0) {
    return 1;  // Floor illuminated, position always visible
}
// ... fall through to IsPositionActuallyInSight range check
```

Per-floor illumination is also set from fixed room properties:

**Evidence:** `RunDungeon`
```c
bVar7 = IsRoomIlluminated((uint)*(byte *)(*DAT_022dff40 + 0x40da));
if (bVar7 != '\0') {
    (dungeon->display_data).luminous = '\x01';
    (dungeon->display_data).natural_lighting = '\x01';
}
```

---

## Tile Visibility State

Each tile's visibility is encoded in two fields of the tile struct.

### Terrain Field (offset 0x00)

Terrain type with embedded visibility control bits.

| Bit | Mask | Purpose |
|-----|------|---------|
| 5 | 0x0020 | Revealed but not visited (Luminous Orb / map reveal) |
| 9 | 0x0200 | Wall or void terrain → always render as full dark |
| 12 | 0x1000 | Currently visible / in sight |
| 13 | 0x2000 | Previously visited |

### Visibility Flags (offset 0x04)

`spawn_or_visibility_flags` union. During dungeon play, contains:

```c
struct visibility_flags {
    bool f_revealed : 1;  // Bit 0: Revealed on the map (gray on minimap)
    bool f_visited : 1;   // Bit 1: Visited by the player
    uint16_t visibility_flags_unk2 : 14;
};
```

**Evidence:** `RevealWholeFloor` sets bit 0 (f_revealed):
```c
do {
    ptVar1 = GetTileSafe(x, y);
    ptVar1->spawn_or_visibility_flags =
        (spawn_or_visibility_flags)(ptVar1->spawn_or_visibility_flags | 1);
} while (...)
```

### Neighbor Visibility Encoding

For edge/transition tiles, the `visibility_flags` field also encodes which of the 8 neighboring tiles are visible. This is used as an index into the transition tile lookup table.

---

## Core Tilemap Selection Function (FUN_02336db0)

The central function that determines what tilemap entries to use for each tile position. Called by `UpdateTrapsVisibility`, `FUN_0233711c`, and `FUN_023372a4`.

### Signature

```c
uint16_t* FUN_02336db0(int tilemap_base, int tile_x, int tile_y, int sub_index);
```

| Parameter | Description |
|-----------|-------------|
| tilemap_base | Base of tilemap buffer (`dungeon + 0x40C4`) |
| tile_x | Dungeon tile X coordinate |
| tile_y | Dungeon tile Y coordinate |
| sub_index | Sub-tile index within the 3×3 grid (0-8) |

### Return Value

Pointer to a `uint16_t` BG map entry. The callers read values from the returned pointer and pass them to `FUN_02051d8c` for BG layer writes.

### Decision Tree

```
GetTile(tile_x, tile_y)
    │
    ├─► terrain & 0x200 (bit 9 = wall/void)
    │       → return DARK_TILE_CONSTANT (0x02352F14)
    │
    ├─► terrain & 0x1000 (bit 12 = currently visible)
    │       → return tilemap_base + 0xE032 + sub_index * 2  [FULLY LIT]
    │
    ├─► terrain & 0x2000 (bit 13 = previously visited)
    │       → return tilemap_base + 0xE032 + sub_index * 2  [FULLY LIT]
    │
    ├─► terrain & 0x0020 (bit 5 = revealed not visited)
    │       → return tilemap_base + 0xE020 + sub_index * 2  [GRAY]
    │
    ├─► Check visibility_flags (tile + 0x04)
    │       → return tilemap_base + 0xC1E4 + vis * 18 + sub_index * 2  [EDGE TRANSITION]
    │
    ├─► terrain & 0x04 (bit 2 = secondary terrain?)
    │       → return tilemap_base + 0xE044 + sub_index * 2  [LIT VARIANT]
    │
    ├─► Room pointer check (tile + 0x10)
    │   ├─► Room exists: room-based visibility checks
    │   │       → tilemap_base + 0xC1E4 transition table  [EDGE TRANSITION]
    │   │       OR tilemap_base + 0xE032  [FULLY LIT]
    │   │
    │   └─► No room (null ptr)
    │           → tilemap_base + 0xDE04 transition table  [EDGE TRANSITION 2]
    │           OR tilemap_base + 0xE032  [FULLY LIT]
    │
    └─► Default fallback
            → return DARK_TILE_CONSTANT  [FULL DARK]
```

**Evidence:** Manual decode of ARM assembly at `0x02336DC8`–`0x02336F48`.

### Tilemap Buffer Offsets

| Offset from tilemap_base | Size | Purpose |
|--------------------------|------|---------|
| +0xC1E4 | Variable | Edge/transition tiles (indexed by visibility_flags × 18) |
| +0xDE04 | Variable | Edge/transition tiles for hallway tiles (null room pointer) |
| +0xE020 | 18 bytes | Gray/revealed-not-visited tiles (3×3 sub-tiles) |
| +0xE032 | 18 bytes | Fully lit tiles (3×3 sub-tiles) |
| +0xE044 | 18 bytes | Lit variant for secondary terrain (water/lava/void) |

### Edge Transition Table

The table at `tilemap_base + 0xC1E4` is the key to soft fog edges.

- Each entry is **18 bytes** = 9 × uint16 BG map entries
- 9 entries = **3×3 sub-tile grid** for one dungeon tile
- Indexed by `visibility_flags` value, which encodes neighbor visibility as a bitmask
- Each combination of lit/dark neighbors maps to a unique set of edge/corner/gradient sub-tiles
- The gradient artwork is baked into the tileset graphics data

```
 ┌───┬───┬───┐
 │ 0 │ 1 │ 2 │  Each cell = one 8×8 BG tile
 ├───┼───┼───┤  Edge/corner tiles contain gradient
 │ 3 │ 4 │ 5 │  artwork transitioning from dark to lit
 ├───┼───┼───┤
 │ 6 │ 7 │ 8 │
 └───┴───┴───┘
```

---

## BG Layer Architecture

### Layer Assignment

| Layer | `FUN_02051d8c` params (layer, screen) | Purpose |
|-------|---------------------------------------|---------|
| 0 | `(x, y, val, 0, 0)` | Y-button grid overlay |
| 1 | `(x, y, val, 1, 0)` | Dungeon terrain + darkness tilemap |
| 2-3 | Not written via tilemap | Background / weather layers, disabled during blinker |

**Evidence:** All `FUN_02051d8c` calls in dungeon rendering use layer 0 or 1 only. No tilemap writes target layers 2 or 3.

### Tilemap Writer

```c
void FUN_02051d8c(int x, int y, uint16_t tile_value, int layer, int screen) {
    FUN_0200b3fc(
        *(layer * 0x18 + screen * 0x30 + *DAT_02051dcc + 8),
        &{x, y, layer},
        tile_value
    );
}
```

### Tilemap Flush

`FUN_02051e60(layer, screen)` commits pending tilemap writes to VRAM.

**Evidence:** Called after every batch of `FUN_02051d8c` writes:
- `FUN_02051e60(0, 0)` — flush grid overlay
- `FUN_02051e60(1, 0)` — flush terrain/darkness layer

### Blinker Blackout

When the camera entity has Blinker status, BG layers 2 and 3 are hardware-disabled rather than using tilemap darkening.

**Evidence:** `UpdateCamera`
```c
bVar10 = IsBlinded(peVar18, '\x01');
if (bVar10 == '\0') {
    uVar16 = 0;
    *(undefined *)(iVar13 + 0x1a23e) = 0;   // blinded = false
} else {
    uVar16 = 0xe;
    *(undefined *)(iVar13 + 0x1a23e) = 1;   // blinded = true
}
if (*puVar8 != uVar16) {
    *puVar8 = uVar16;
    if (uVar16 == 0xe) {
        FUN_02009194(2, 0);  // Disable BG layer 2
        FUN_02009194(3, 0);  // Disable BG layer 3
    } else {
        FUN_020091b0(2, 0);  // Enable BG layer 2
        FUN_020091b0(3, 0);  // Enable BG layer 3
    }
}
```

When blinded, `FUN_022e1854` (entity renderer) only renders the camera entity itself — all other entities are hidden.

---

## Per-Frame Rendering Pipeline

The main per-frame function is `FUN_022ea008`.

```
FUN_022ea008 (per-frame render)
    │
    ├─► UpdateCamera('\x01')
    │       ├─► Update camera_pos from leader entity
    │       ├─► Update camera_pixel_pos (pixel_x >> 8 - 0x80, pixel_y >> 8 - 0x6C)
    │       ├─► Check X-Ray Specs, Power Ears → set can_see_enemies/items
    │       ├─► Check Y-Ray Specs → set can_see_traps
    │       ├─► Check Blinker → toggle BG layers 2/3
    │       ├─► Check Dropeye → update visibility range
    │       ├─► If position changed: redraw minimap tiles (DrawMinimapTile per entity)
    │       └─► FUN_022e34c8() — screen shake handler
    │
    ├─► FUN_02051e20() — BG layer scroll positioning from camera pixel pos
    │
    ├─► FUN_0230473c() — Monster idle turn animation
    │
    ├─► FUN_022e1854() — Entity renderer
    │       ├─► Normal: render team (FUN_022e8270), allies, items (FUN_023457c8)
    │       └─► Blinded: render camera entity only, hide all others
    │
    ├─► FUN_022e335c() — Touch screen number update (HP, level, floor)
    │
    ├─► DisplayUi() — HUD rendering (HP bar, floor number, level)
    │
    └─► Minimap rendering
```

### Tile Redraw Triggers

The terrain/darkness tilemap (BG layer 1) is NOT fully redrawn every frame. Instead it is updated incrementally when:

1. **Camera scrolls** — `FUN_0233711c` and `FUN_023372a4` update single rows/columns at the screen edge as the camera moves, calling `FUN_02336db0` for each new tile entering the viewport
2. **Trap visibility changes** — `UpdateTrapsVisibility` redraws the full visible area
3. **Status changes** — Blinker infliction/cure, Cross-Eyed, explosions, etc.
4. **Luminous Orb / reveal** — `RevealWholeFloor` marks all tiles revealed, triggers full redraw via `UpdateCamera` → `UpdateTrapsVisibility`

**Evidence:** `FUN_0233711c` (horizontal scroll) and `FUN_023372a4` (vertical scroll) are called from `UpdateCamera` when the camera pixel position changes:
```c
if (iVar20 < iVar19) {
    FUN_0233711c(0xff, 0);   // Scroll right: draw new column
} else if (iVar19 < iVar20) {
    FUN_0233711c(0, 0);      // Scroll left: draw new column
}
if (*(short *)(...) < *(short *)(...)) {
    FUN_023372a4(0, 0xc0);   // Scroll down: draw new row
} else if (...) {
    FUN_023372a4(0, 0);      // Scroll up: draw new row
}
```

---

## Visibility Logic

### Position In Sight Check

`IsPositionActuallyInSight` determines if a target position is visible from the camera.

**Rules:**
- If camera is in a **room**: target must be within the room's boundaries
- If camera is in a **hallway** (or has Dropeye): target must be within `visibility_range` tiles (Chebyshev distance)
- Default `visibility_range` is 2 if stored as 0

### Entity Display Gating

`ShouldDisplayEntity` applies a two-stage visibility gate:

1. **Screen bounds check**: entity must be within ~64 pixels of camera (items/traps use ~32)
2. **Logical visibility check** via `FUN_022e2ca0`:
   - Hard distance cutoff: 6 tiles X, 5 tiles Y (matches NDS screen aspect ratio)
   - If floor illuminated (`FUN_022e3534`): always visible
   - Otherwise: calls `IsPositionActuallyInSight` for range/room check

### See-All Overrides

| Flag | Offset | Set By | Effect |
|------|--------|--------|--------|
| can_see_enemies | +0x1a242 | X-Ray Specs, Power Ears (info + 0xF9) | Show all monsters |
| can_see_items | +0x1a243 | X-Ray Specs (info + 0xFA) | Show all items |
| can_see_all | +0x1a244 | `CanSeeInvisibleMonsters` | Show invisible + traps |

---

## Tilemap Buffer Population

The tilemap buffer at `dungeon + 0x40C4` is populated during floor initialization. The buffer contains pre-computed BG map entries for all visibility states and edge configurations.

`FUN_02052060` loads tileset graphics from a pack file into BG tile memory and writes tilemap entries:

```c
void FUN_02052060(char *filename, uint vram_offset, int start_row, int palette_idx,
                  int bg_layer, int screen);
```

This function:
1. Loads a tileset file via `FUN_02051ff0` (allocate and load)
2. Copies tile graphics to VRAM via `FUN_02051804`
3. Iterates 32×(32-start_row) BG tiles, writing map entries to the specified BG layer
4. Each map entry encodes: tile index + palette index + flip flags
5. Loads palette data via `FUN_0200a590`

The gray/dim tiles reuse the same tile graphics as lit tiles but with a **different palette index** in the BG map entry, pointing to a darkened version of the palette.

---

## Implementation Guide (Modern Engine)

### Core Algorithm

```
Per frame:
1. Determine camera position (leader tile)
2. Check display_data.darkness — if false, render all tiles lit
3. Check display_data.luminous / natural_lighting — if true, render all tiles lit
4. Check display_data.blinded — if true, render black screen

5. For each tile on screen:
   a. If wall/void → render black
   b. If currently visible (in room, or within visibility_range) → render fully lit
   c. If revealed but not visited → render with dark tint
   d. If unvisited and unseen → render black
   e. If at boundary between lit and dark:
      → Check 8 neighbors for their visibility state
      → Build bitmask of neighbor visibility
      → Select edge/corner overlay from lookup table (or use shader gradient)
```

### Visibility Marking

```
Per leader move:
1. Get leader position and visibility_range
2. Get tile at leader position
3. If tile is in a room:
   → Mark all tiles within room boundaries as visible
4. If tile is in a hallway:
   → Mark all tiles within visibility_range (Chebyshev distance) as visible
5. Mark visited tiles permanently (they stay gray when out of range)
```

### Edge Transition Approach

The original ROM uses a lookup table with pre-baked gradient tiles. For a modern engine:

**Option A (Faithful):** Create a 256-entry lookup table (8 neighbors = 8-bit mask) mapping each neighbor configuration to a set of edge/corner overlay sprites. This exactly replicates the NDS behavior.

**Option B (Modern):** Use a shader-based approach:
- Render visibility as a low-resolution mask texture (1 pixel per tile)
- Apply Gaussian blur or bilinear interpolation to create smooth gradients
- Use the blurred mask to modulate darkness overlay opacity

---

## Cross-References

> See `Data Structures/effect_animation_info.md` for type 5 screen effects (separate system from fog of war)

> See `status_visual_pipeline.md` for Blinker status visual effects

> See `tileset-properties.md` for `weather_effect` foggy levels (ambient weather, not fog of war)

## Open Questions

- Complete bit layout of the terrain field at tile offset 0x00 (bits 0-4, 6-8, 10-11, 14-15 unknown)
- What populates the transition lookup table at `tilemap_base + 0xC1E4` during floor init
- Exact meaning of the second transition table at `tilemap_base + 0xDE04` (hallway-specific?)
- Whether `field_0x34` in display_data is used for cutscene blackout or another purpose
- How `visibility_flags` neighbor encoding is computed (which bits map to which neighbors)
- Purpose of BG layers 2/3 beyond weather/background (confirmed not used for darkness tilemap)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_02336db0` | `0x02336db0` | Core tilemap selection: returns dark/gray/lit/edge tile entries |
| `UpdateTrapsVisibility` | `0x02336f4c` | Redraws full visible area on BG layer 1 |
| `FUN_0233711c` | `0x0233711c` | Incremental horizontal scroll tilemap update |
| `FUN_023372a4` | `0x023372a4` | Incremental vertical scroll tilemap update |
| `DrawTileGrid` | - | Y-button walkable tile grid overlay (BG layer 0) |
| `HideTileGrid` | - | Clear Y-button grid overlay |
| `FUN_02051d8c` | `0x02051d8c` | Write single BG tilemap entry |
| `FUN_02051e60` | `0x02051e60` | Flush/commit tilemap writes to VRAM |
| `FUN_02052060` | `0x02052060` | Load tileset and populate BG tilemap + tile VRAM |
| `FUN_020091b0` | `0x020091b0` | Enable BG layer |
| `FUN_02009194` | `0x02009194` | Disable BG layer |
| `GetTile` | `0x02336150` | Get tile struct at position (returns default for out-of-bounds) |
| `GetVisibilityRange` | - | Read visibility_range, default to 2 if 0 |
| `IsPositionActuallyInSight` | - | Range/room-based visibility check |
| `IsPositionInSight` | - | Simplified visibility check (AI targeting) |
| `ShouldDisplayEntity` | - | Two-stage entity display gate (screen bounds + visibility) |
| `ShouldDisplayEntityAdvanced` | - | Display gate with blinker check |
| `FUN_022e3534` | `0x022e3534` | Check if floor is illuminated (luminous / natural_lighting) |
| `FUN_022e2b68` | `0x022e2b68` | Initialize display_data from floor properties |
| `FUN_022e2ca0` | `0x022e2ca0` | Entity visibility check with distance cutoff |
| `FUN_022e2dfc` | `0x022e2dfc` | Set camera target entity |
| `UpdateCamera` | - | Per-frame camera update, minimap, blinker toggle |
| `FUN_022ea008` | `0x022ea008` | Main per-frame dungeon render function |
| `FUN_022e1854` | `0x022e1854` | Entity renderer (sprites, items) |
| `FUN_022e335c` | `0x022e335c` | Touch screen number update |
| `FUN_022e34c8` | `0x022e34c8` | Screen shake handler |
| `RevealWholeFloor` | - | Luminous Orb: set f_revealed on all tiles |
| `IsRoomIlluminated` | - | Check fixed room illumination property |
| `CanSeeTarget` | - | Full visibility check (entity → entity) |
