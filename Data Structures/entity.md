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
| 0xBD | uint8_t | sleep_class_status | Sleep class (0=none, 1=sleep, 2=sleepless, 3=nightmare, 4=yawning, 5=napping) |
| 0xBE | uint8_t | sleep_counter | Duration counter for sleep class |
| 0xBF | uint8_t | burn_class_status | Burn class (0=none, 1=burn, 2=poison, 3=badly_poisoned, 4=paralysis) |
| 0xC0 | uint8_t | burn_duration_counter | Duration counter — ticked by TickStatusAndHealthRegen |
| 0xC1 | uint8_t | burn_damage_countdown | Damage tick countdown — ticked by FUN_0230fc24 (burn/poison only) |
| 0xC2 | uint8_t | toxic_tick_count | Badly poisoned escalation index (caps at 29) |
| 0xC4 | uint8_t | freeze_class_status | Freeze class (0=none, 1=frozen, 3=shadow_hold, 4=wrap, 5=ingrain, 6=petrified, 7=constriction) |
| 0xCC | uint8_t | freeze_counter | Duration counter for freeze class |
| 0xCD | uint8_t | freeze_damage_countdown | Damage tick countdown (constriction/wrap) |
| 0xD0 | uint8_t | cringe_class_status | Cringe class (0=none, 1=cringe, 2=confused, 3=paused, 4=cowering, 5=taunted, 6=encore, 7=infatuated) |
| 0xD1 | uint8_t | cringe_counter | Duration counter for cringe class |
| 0xD2 | uint8_t | bide_class_status | Bide/two-turn status |
| 0xD3 | uint8_t | bide_counter | Duration counter for bide |
| 0xD5 | uint8_t | reflect_class_status | Reflect class (see status_icon_system.md for enum) |
| 0xD6 | uint8_t | reflect_counter | Duration counter for reflect class |
| 0xD7 | uint8_t | aqua_ring_counter | Aqua Ring heal tick counter (reflect class value 0x10) |
| 0xD8 | uint8_t | curse_class_status | Curse class (0=none, 1=cursed, 2=decoy, 3=snatch, 4=gastro_acid, 5=heal_block, 6=embargo) |
| 0xDB | uint8_t | curse_counter | Duration counter for curse class |
| 0xDC | uint8_t | curse_damage_countdown | Damage tick countdown (curse) |
| 0xE0 | uint8_t | leech_seed_class_status | Leech seed class |
| 0xE4 | int32_t | leech_seed_source_unique_id | Unique ID of the entity that applied leech seed |
| 0xE8 | uint8_t | leech_seed_source_index | Entity pool index of leech seed source |
| 0xE9 | uint8_t | leech_seed_counter | Duration counter for leech seed |
| 0xEA | uint8_t | leech_seed_damage_countdown | Damage tick countdown (leech seed) |
| 0xEC | uint8_t | sure_shot_class_status | Sure shot class |
| 0xED | uint8_t | sure_shot_counter | Duration counter |
| 0xEF | uint8_t | invisible_class_status | Invisible class |
| 0xF0 | uint8_t | invisible_counter | Duration counter |
| 0xF1 | uint8_t | blinker_class_status | Blinker class |
| 0xF2 | uint8_t | blinker_counter | Duration counter |
| 0xF3 | uint8_t | muzzled_status | Boolean |
| 0xF4 | uint8_t | muzzled_counter | Duration counter |
| 0xF5 | uint8_t | miracle_eye_status | Boolean |
| 0xF6 | uint8_t | miracle_eye_counter | Duration counter |
| 0xF7 | uint8_t | magnet_rise_status | Boolean |
| 0xF8 | uint8_t | magnet_rise_counter | Duration counter |
| 0x104 | uint8_t | run_away_status | Flee state (1=type A, 2=type B, logged differently) |
| 0x105 | uint8_t | run_away_counter | Duration counter for flee |
| 0x106 | uint8_t | perish_song_counter | Perish Song countdown |
| 0x110 | int32_t | cached_speed_stage | Cached speed stage value, compared after counter expiry |
| 0x114-0x118 | uint8_t[5] | stat_stage_counters | Speed/stat stage duration counters (set A) |
| 0x119-0x11D | uint8_t[5] | stat_stage_counters_2 | Speed/stat stage duration counters (set B) |
| 0x150 | uint8_t | hunger_damage_flag | Set to 1 when hunger damage dealt this turn |
| 0x210 | int16_t | hp_regen_accumulator | HP regen fractional accumulator |

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
