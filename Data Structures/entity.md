# Entity Structure

## Summary

- Entities are the base object for all dungeon objects (monsters, items, traps)
- Entity struct contains type, position, pixel position, and pointer to type-specific info
- Monster info is a separate struct pointed to by `entity->info`
- Direction stored at monster_info offset 0x4C

## Entity Structure
```c
struct entity {
    entity_type type;              // 0x00: ENTITY_NOTHING, ENTITY_MONSTER, ENTITY_ITEM, etc.
    void *info;                    // 0x04: Pointer to type-specific data
    position pos;                  // 0x08: Tile coordinates
    pixel_position pixel_pos;      // 0x0C: Pixel position (8.8 fixed point)
    // ...
    uint16_t spawn_genid;          // Generation ID
    damage_visual_8 damage_visual_effect;
    int16_t elevation;             // Z offset for rendering
    char is_visible;               // Visibility flag
    // ...
    uint8_t animation_id;          // Current animation state
};
```

## Entity Types
```c
enum entity_type {
    ENTITY_NOTHING = 0,
    ENTITY_MONSTER = 1,
    ENTITY_ITEM = 2,
    // ... other types
};
```

**Evidence:** `EntityIsValidMoveEffects`
```c
bool EntityIsValidMoveEffects(entity *entity)
{
    if (entity != NULL) {
        entity_type type = entity->type;
        if (type != ENTITY_NOTHING) {
            return type == ENTITY_MONSTER;
        }
    }
    return false;
}
```

## Position Structure
```c
struct position {
    int16_t x;    // Tile X coordinate
    int16_t y;    // Tile Y coordinate
};
```

## Pixel Position Structure
```c
struct pixel_position {
    int32_t x;    // 8.8 fixed point (upper 8 bits = screen pixel)
    int32_t y;    // 8.8 fixed point
};
```

**Usage:**
```c
// Get screen pixel from fixed point
int screen_x = entity->pixel_pos.x >> 8;
int screen_y = entity->pixel_pos.y >> 8;
```

## Monster Info Structure

When `entity->type == ENTITY_MONSTER`, the `entity->info` pointer points to monster-specific data.

### Known Offsets

| Offset | Type | Field | Description |
|--------|------|-------|-------------|
| 0x04 | int16_t | species_id | Monster species (used in FUN_023258ec) |
| 0x10 | int16_t | current_hp | Current HP |
| 0x4C | uint8_t | direction | Facing direction (0-7) |
| 0xAC | uint16_t | held_move_id | Currently selected move |
| 0xB0 | uint32_t | some_id | Used in snatch check |
| 0xD2 | uint8_t | charging_status | Two-turn move charge state |
| 0x10B | uint8_t | some_flag | Various status checks |
| 0x146 | uint32_t | some_value | Fixed point value (ceiling applied) |
| 0x164 | uint8_t | flag | Cleared at start of move effect |
| 0x166 | uint8_t | flag | Set when move misses |
| 0x167 | uint8_t | flag | Related to move execution |
| 0x170 | uint8_t | flag | Cleared for specific moves |
| 0x23F | uint8_t | flag | Checked in move targeting |

### Direction Values

Direction is stored at `monster_info + 0x4C` as a single byte (0-7).
```c
enum direction_id {
    DIR_DOWN = 0,
    DIR_DOWN_RIGHT = 1,
    DIR_RIGHT = 2,
    DIR_UP_RIGHT = 3,
    DIR_UP = 4,
    DIR_UP_LEFT = 5,
    DIR_LEFT = 6,
    DIR_DOWN_LEFT = 7,
};
```

**Evidence:** `FUN_023230fc` (projectile motion)
```c
uVar18 = (uint)*(byte *)(uVar18 + 0x4c);  // Read direction from monster_info
iVar16 = *(int *)(DAT_023238f8 + uVar18 * 4);  // Look up in direction table
```

**Evidence:** `ExecuteMoveEffect`
```c
bVar4 = *(byte *)(pvVar20 + 0x4c);  // Read direction
uVar11 = bVar4 + 1 & 7;  // Rotate direction by 1
```

## Animation Control

Entity contains animation control data at offset 0x0B (relative to some base):

**Evidence:** `FUN_023258ec`
```c
FUN_0201cf90((short *)local_30, (ushort *)(param_2 + 0xb), uVar12);  // target + 0xb
FUN_0201cf90((short *)local_44, (ushort *)(param_1 + 0xb), uVar12);  // attacker + 0xb
```

**Evidence:** `PlayMoveAnimation`
```c
FUN_0201cf90((short *)&local_2c, &(target->anim_ctrl).some_bitfield, uVar12);
```

This suggests `entity->anim_ctrl` or similar is at a fixed offset and contains the animation state used for attachment point lookups.

## Entity Validation

**Evidence:** `EntityIsValid` pattern
```c
bool EntityIsValid(entity *entity)
{
    if (entity == NULL) return false;
    return entity->type != ENTITY_NOTHING;
}
```

Used throughout the codebase before accessing entity data:
```c
bVar8 = EntityIsValid(attacker);
if (bVar8 != '\0') {
    // Safe to access attacker->info, etc.
}
```

## Entity Tables

Entities are stored in pools within dungeon memory:

**Evidence:** `SpawnItemEntity`
```c
// Item entity pool at dungeon_struct + 0x12BC8
// Up to 0x40 (64) items
for (iVar6 = 0; iVar6 < 0x3f; iVar6++) {
    entity = *(entity **)(*piVar2 + iVar6 * 4 + 0x12bc8);
    if (!EntityIsValid(entity)) break;  // Found empty slot
}
```

## Open Questions

- Complete monster_info structure (many offsets unknown)
- Exact entity struct size
- Animation control structure layout at entity + 0xB
- How entity pools are organized for different entity types

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `EntityIsValid` | - | Checks if entity pointer is valid and not NOTHING |
| `EntityIsValidMoveEffects` | - | Checks if entity is valid monster for move effects |
| `SpawnItemEntity` | - | Creates item entity in pool |
| `SpawnMonster` | `0x022fd084` | Creates monster entity |
| `UpdateEntityPixelPos` | - | Updates pixel position from tile position |
| `SetEntityPixelPosXY` | - | Sets pixel position directly |
